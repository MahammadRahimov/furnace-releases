# Furnace 0.10.0 on Amazon Linux 2023: Docker, keepalived on the host

[All guides](../README.md) · Same system, other ways: [Packages (.deb / .rpm)](../packages/amazon-2023.md) · [Docker, keepalived in the container](../docker/amazon-2023.md)

Each of the two hosts runs one Furnace container, but the **host's own keepalived** moves the shared IP. Use this guide when the hosts already run keepalived for other services (or you want to manage keepalived yourself). Furnace gets **its own** `vrrp_instance` (its own shared IP and router ID) next to your other blocks, so a Furnace problem moves only Furnace's shared IP.

How it works: your keepalived runs a small **health check** (is Furnace alive here?) and a **notify** script that writes `leader` or `follower` into `/run/furnace/role` on the host. The container reads that file. **Furnace never changes your host**: it prints the scripts and the block, and you add them.

> **Inside AWS, a shared IP that moves between two machines does not work** (AWS does not allow an IP to move by ARP). Use Amazon Linux 2023 on your own VMware, KVM or Hyper-V, where the network allows it.

## The plan

You need three IP addresses in **one network segment**. This guide uses these examples. Write your own next to them
before you start, and use yours everywhere the guide says the example:

| | Example | Yours |
|---|---|---|
| Host A: the leader host | `192.0.2.11` | |
| Host B: the follower host | `192.0.2.12` | |
| Shared IP (VIP): a **free** address, nobody owns it, keepalived moves it | `192.0.2.10` | |
| Network prefix (24 = 255.255.255.0) | `24` | |
| The network card name on both hosts (see step A3) | `ens192` | |

- **Host A** is set up first. It makes the certificate and the secrets.
- **Host B** is set up second. It fetches the certificate and the secrets from host A with a **join code**.
- The firewall that downloads the lists and the SOAR that calls the API **always use the shared IP**, never a host IP.
- The settings are **the same on both hosts**. Each host finds its role from its own IP: the host that has the leader's
  IP is the leader. Never swap the two IPs on host B.

## Part 0. Before you start

### What you need

- Two hosts (real or VMs) with **Amazon Linux 2023**. 1 CPU, 1 to 2 GB memory and 5 GB free disk each.
- A user with `sudo` on both. This guide writes `sudo docker ...` everywhere. (You can add your user to the `docker`
  group instead: `sudo usermod -aG docker $USER`, then log in again. Members of that group are root in practice.)
- The three IPs from the plan, all in one network segment.
- Both hosts can reach the Amazon Linux 2023 package mirror, download.docker.com and github.com.
- Port 443 free on both.
- keepalived on both hosts (Part 1 installs it if it is missing).

### Network: what must be open

Ask your network team to allow these **before** you start. "Each host" means both directions.

| From | To | Port | Why |
|---|---|---|---|
| the firewall that downloads the lists (its source IP) | shared IP `192.0.2.10` | TCP 443 | the lists |
| the SOAR or script that calls the API (and its engines) | shared IP `192.0.2.10` | TCP 443 | the API |
| each host | the other host | TCP 443 | replication every 20 seconds, the join code |
| each host | the other host | IP protocol 112 (VRRP) | keepalived: who owns the shared IP |
| your admin PC | both hosts | TCP 22 (SSH) | your own login |
| both hosts | their package mirror and github.com | TCP 443 | install the packages and download Furnace |

**Virtual machines:** the shared IP moves by ARP. On **VMware**, the port group of both VMs must allow
"MAC address changes" and "Forged transmits" (or use a distributed switch with these set). On **Hyper-V**, turn on
"Enable MAC address spoofing" on both VMs' network adapters. Most **clouds** (AWS, Azure, GCP) do not allow a moving
IP at all: use real servers or your own virtualization.

**VRRP is the part that most often goes wrong.** If protocol 112 is blocked between the hosts, both hosts take the
shared IP at the same time ("split brain"). Ask for "IP protocol 112", not a port: VRRP has no port.

### How the container runs

- `network_mode: host`: the container uses the host's network card directly.
- No extra rights are needed in this mode (you may delete the `cap_add` line, optional).
- Two folders next to `docker-compose.yml`: `config` (certificate and secrets: **private**) and `data` (database,
  backups). They survive a new image.
- **The container never changes the host firewall.** You open the ports yourself (step A6).
- `restart: unless-stopped`: it starts again after a reboot.

## Part 1. Host A: the leader (192.0.2.11)

Do every step of this part on **host A**.

### A1. Log in and check the system

Log in to host A (192.0.2.11) with SSH as your normal user (not root). Then:

```bash
grep PRETTY_NAME /etc/os-release    # which system is this?
sudo -v                              # can you use sudo? (asks your password)
```

You should see `PRETTY_NAME="Amazon Linux 2023"` (the last numbers may be newer). Another system? Use the guide
for that system instead: [all guides](../README.md).

### A2. Bring the system up to date

```bash
sudo dnf upgrade -y
```

If it installed a new kernel, restart now (`sudo reboot`) and log in again. A restart later would move the shared IP.

### A3. Find the network card and check the IP

```bash
ip -br addr
```

You see one line per card. Example on host A:

```text
lo               UNKNOWN        127.0.0.1/8 ::1/128
ens192           UP             192.0.2.11/24 fe80::250:56ff:fe8a:1b2c/64
```

- The card with `192.0.2.11` is the one Furnace uses. Its name here is `ens192`. Yours may be `eth0`, `ens160`,
  `ens33` or `enp1s0`: write down **your** name.
- The number after the slash is the prefix (`/24`). The shared IP uses the same one.
- The shared IP `192.0.2.10` must **not** be in this list yet, on either host.

Check that nobody uses the shared IP yet:

```bash
ping -c 3 192.0.2.10            # should FAIL: 100% packet loss
ping -c 3 192.0.2.12            # the other host: should answer (if ICMP is allowed)
```

If `192.0.2.10` answers, another machine has it. Pick another free address for the shared IP.

### A4. Check that port 443 is free

```bash
sudo ss -ltnp 'sport = :443'
```

You should see only the header line (`State  Recv-Q  Send-Q ...`). If a program is listed (for example `apache2`,
`httpd` or `nginx` with another site), port 443 is taken. Furnace needs it: stop and remove that program first,
or use another server.

### A5. Install Docker and the Compose plugin

```bash
# 1. Docker comes from Amazon's own repository
sudo dnf install -y docker
sudo systemctl enable --now docker

# 2. The Compose plugin is not packaged on Amazon Linux 2023: put Docker's own file in place
sudo mkdir -p /usr/local/lib/docker/cli-plugins
sudo curl -fL -o /usr/local/lib/docker/cli-plugins/docker-compose \
    "https://github.com/docker/compose/releases/latest/download/docker-compose-linux-$(uname -m)"
sudo chmod 755 /usr/local/lib/docker/cli-plugins/docker-compose
```

Check it:

```bash
sudo docker version --format '{{.Server.Version}}'   # a version number, for example 28.5.1
sudo docker compose version                           # Docker Compose version v2.x
sudo docker run --rm hello-world                      # 'Hello from Docker!' (needs Docker Hub; skip if blocked)
```

### A6. Open the host firewall

The container does not touch the host firewall. On host A, allow SSH, HTTPS (443) from everyone, and VRRP from
host B (`192.0.2.12`):

```bash
sudo dnf install -y firewalld          # Amazon Linux 2023: not there by default
sudo systemctl enable --now firewalld
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.0.2.12" protocol value="vrrp" accept'
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```
You should see `services: ... https ssh` and the rich rule with `192.0.2.12` and `vrrp`.

Already limit 443 to some addresses? Then allow at least: the firewall, the SOAR, and host B.

### A7. Check SELinux

```bash
getenforce
```

SELinux is **Permissive** by default on Amazon Linux 2023 (it logs, but blocks nothing). Keep it as it is. The container needs nothing (Docker CE runs it without SELinux labels by default). The host's keepalived needs a label on the two Furnace scripts: the keepalived step below does it.

### A8. Install keepalived on the host (if it is not there yet)

**a) Is keepalived already in use on this host?**

```bash
systemctl is-active keepalived
```

- `active`: keepalived already runs here for other services. Go to **d)**, case 2.
- `inactive`, `unknown` or `failed`: keepalived is new on this host. Do **b)**, **c)** and **d)** case 1.

**b) Install keepalived and curl:**

```bash
sudo dnf install -y keepalived curl policycoreutils-python-utils
```

**c) Back up the old file** (if there is one):

```bash
sudo cp -a /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.before-furnace 2>/dev/null; ls /etc/keepalived/
```

**d) Write the top of `/etc/keepalived/keepalived.conf`.**

*Case 1: keepalived is new on this host.* On Amazon Linux 2023, a fresh keepalived package comes with an **example** `/etc/keepalived/keepalived.conf` (with `vrrp_instance VI_1`, `virtual_router_id 51` and example IPs `192.168.200.x`). Its router ID 51 would clash with Furnace's. The command below **replaces** that example file.

```bash
sudo tee /etc/keepalived/keepalived.conf >/dev/null <<'EOF'
global_defs {
    script_user root
    enable_script_security
}
EOF
```

*Case 2: keepalived already runs here for other services.* Keep your file. Open it:

```bash
sudo nano /etc/keepalived/keepalived.conf
```

- If it has a `global_defs { ... }` block, add these two lines inside it:
  `script_user root` and `enable_script_security`.
- If it has no `global_defs` block, add this at the very top of the file:

   ```text
   global_defs {
       script_user root
       enable_script_security
   }
   ```

  Save with Ctrl+O, Enter, Ctrl+X. **Do not** restart keepalived yet.

**e) Check it:**

```bash
sudo head -5 /etc/keepalived/keepalived.conf
```

```text
global_defs {
    script_user root
    enable_script_security
}
```

(In case 2 you may see your own lines too. The two lines must be inside `global_defs`.)

What the two lines do: `script_user root` lets keepalived run the Furnace scripts as root. `enable_script_security`
makes it refuse scripts that others could change (the Furnace scripts are owned by root, mode 755).

### A9. Download Furnace and load the image

```bash
mkdir ~/furnace && cd ~/furnace
curl -fLO https://github.com/MahammadRahimov/furnace-releases/releases/download/v0.10.0/furnace-image-0.10.0.tar.gz
curl -fLO https://github.com/MahammadRahimov/furnace-releases/releases/download/v0.10.0/docker-compose.yml
curl -fLO https://github.com/MahammadRahimov/furnace-releases/releases/download/v0.10.0/env.example
sudo docker load -i furnace-image-0.10.0.tar.gz     # "Loaded image: furnace:0.10.0"
mv env.example .env
ls -la                                                  # .env  docker-compose.yml  furnace-image-0.10.0.tar.gz
```

`docker-compose.yml` is the same on both hosts: **never edit it**. Your settings go into `.env`.

### A10. Write the settings file `.env`

```bash
nano .env
```

Delete what is in it and write (with your own IPs):

```text
# The three IPs. This whole file is THE SAME on both hosts.
FURNACE_LEADER_IP=192.0.2.11
FURNACE_FOLLOWER_IP=192.0.2.12
FURNACE_SHARED_IP=192.0.2.10
FURNACE_SHARED_IP_PREFIX=24

# keepalived: the HOST's own (this guide)
FURNACE_KEEPALIVED=external

# Optional (the same on both hosts):
#FURNACE_INTERFACE=ens192
#FURNACE_API_ALLOW_FROM=any
#FURNACE_LISTS_ALLOW_FROM=any
#FURNACE_VRRP_ROUTER_ID=51
#FURNACE_ARCHIVE_AFTER_DAYS=90
#FURNACE_MAX_IPS=50000

TZ=UTC
```

| Setting | Meaning |
|---|---|
| `FURNACE_LEADER_IP` | host A's own IP |
| `FURNACE_FOLLOWER_IP` | host B's own IP |
| `FURNACE_SHARED_IP` | the free shared IP |
| `FURNACE_SHARED_IP_PREFIX` | the prefix from step A3 |
| `FURNACE_KEEPALIVED=external` | the container runs no keepalived; the host's keepalived decides who is the leader |
| `FURNACE_INTERFACE` | only if the card is not found by itself (the card that has the host's IP) |
| `FURNACE_API_ALLOW_FROM` | `any`, or the SOAR's IPs and ranges, comma-separated (a token is always needed) |
| `FURNACE_LISTS_ALLOW_FROM` | `any`, or the firewall's source IPs (the login is always needed) |
| `FURNACE_VRRP_ROUTER_ID` | 1 to 255, unused by every other keepalived/VRRP in this segment (also your own blocks on this host) |
| `FURNACE_ARCHIVE_AFTER_DAYS` | days a temporary IP stays in the list |
| `FURNACE_MAX_IPS` (and `_DOMAINS`, `_URLS`) | the firewall's limit per list, 0 = no limit |
| `TZ` | time zone of the daily backup (01:30) and the nightly cleanup (02:30), for example `Europe/Berlin` |

Save with Ctrl+O, Enter, Ctrl+X. Keep `.env` private: `chmod 600 .env`.

### A11. Check the settings and start the container

```bash
sudo docker compose run --rm furnace check
```

```text
Settings are fine: this host is the leader (192.0.2.11 on ens192), the other host is 192.0.2.12, shared IP 192.0.2.10/24.
```

A mistake? It names every problem (for example *FURNACE_SHARED_IP is not an IPv4 address*). Fix `.env` and check again.

```bash
sudo docker compose up -d
sudo docker logs furnace
```

```text
furnace: Starting Furnace 0.10.0 (Docker).
furnace: This host is the leader (192.0.2.11); the other host is 192.0.2.12.
furnace: Self-signed certificate made (10 years) for 192.0.2.10, 192.0.2.11, 192.0.2.12.
furnace: keepalived password made.
furnace: Replication secret made.
furnace: Follower bundle: /etc/furnace/furnace-peer-bundle.tar.gz (...)
furnace: Database ready and lists written (existing data kept).
...
furnace: Ready. This host is the leader by setting; the HOST's keepalived decides who owns 192.0.2.10 (FURNACE_KEEPALIVED=external).
furnace: No role from the host's keepalived yet: this host works as the follower until it writes /run/furnace/role. ...
```

The certificate and the secrets are now in `~/furnace/config`. `ls -l ~/furnace` may show odd owner names for
`config` and `data` (for example `systemd-journal`): normal, the container's users have numbers that mean other names
on the host.

### A12. Add Furnace to the host's keepalived

Until now, host A's container works as the follower (no role from the host yet). This step makes host A's keepalived
take the Furnace shared IP. `sudo docker exec furnace furnace host-keepalived` shows all of it for this host; here it is
step by step.

**a) The two scripts:**

```bash
sudo docker exec furnace furnace host-keepalived notify       | sudo tee /usr/local/sbin/furnace-notify >/dev/null
sudo docker exec furnace furnace host-keepalived health-check | sudo tee /usr/local/sbin/furnace-health-check >/dev/null
sudo chmod 755 /usr/local/sbin/furnace-notify /usr/local/sbin/furnace-health-check
sudo chown root:root /usr/local/sbin/furnace-notify /usr/local/sbin/furnace-health-check
```

- `furnace-health-check` asks `https://127.0.0.1/health` on this host (with `curl`). 3 failures in a row (15 seconds)
  and keepalived gives the shared IP away.
- `furnace-notify` writes `leader` or `follower` into `/run/furnace/role`. The container reads it.

**SELinux** (permissive on Amazon Linux 2023): let keepalived run the two scripts:

```bash
sudo semanage fcontext -a -t keepalived_unconfined_script_exec_t '/usr/local/sbin/furnace-(notify|health-check)'
sudo restorecon -v /usr/local/sbin/furnace-notify /usr/local/sbin/furnace-health-check
```


**b) The Furnace block.** It is filled in **for this host** (card, both IPs, router ID, priority, password). Always
print it on the host it is for:

```bash
sudo docker exec furnace furnace host-keepalived config
```

```text
vrrp_script chk_furnace {
    script "/usr/local/sbin/furnace-health-check"
    interval 5
    timeout 4
    fall 3
    rise 2
}

vrrp_instance FURNACE {
    state BACKUP
    nopreempt
    interface ens192
    virtual_router_id 51
    priority 150
    advert_int 1

    unicast_src_ip 192.0.2.11
    unicast_peer {
        192.0.2.12
    }

    authentication {
        auth_type PASS
        auth_pass ********
    }

    virtual_ipaddress {
        192.0.2.10/24 dev ens192
    }

    track_script {
        chk_furnace
    }

    notify /usr/local/sbin/furnace-notify
}
```

- `priority 150` on host A, `100` on host B: host A wins at the very first start.
- `state BACKUP` + `nopreempt` on both: a host that comes back does not take the shared IP back.
- `unicast_peer`: VRRP goes straight to the other host, not by multicast.
- `auth_pass`: Furnace's keepalived password, the same on both hosts (host B gets it with the join code).

Add it **below** your other blocks (the backup of the old file was made in the keepalived install step). Do not
change your own blocks:

```bash
sudo docker exec furnace furnace host-keepalived config | sudo tee -a /etc/keepalived/keepalived.conf >/dev/null
```

**c) Check the file and restart keepalived:**

```bash
sudo keepalived -t -f /etc/keepalived/keepalived.conf && echo "file OK"
sudo systemctl enable keepalived
sudo systemctl restart keepalived
sleep 5
cat /run/furnace/role                       # leader
ip -br addr show ens192                     # 192.0.2.11/24 and 192.0.2.10/24
```

Restarting keepalived can move **your other** shared IPs for a moment. Do it in a quiet time.

### A13. Check host A

```bash
sudo docker exec furnace furnace status
```

```text
Furnace 0.10.0 (Docker)
Role:     leader (this host owns the shared IP 192.0.2.10)
keepalived: the host's own (FURNACE_KEEPALIVED=external)
Host:     192.0.2.11 (the leader by setting), other host 192.0.2.12

Processes:
  api          running
  nginx        running

Replication: failed ...
```

**Replication failed** is normal now: host B is not there yet.

### A14. Make the first API token and the list login

```bash
sudo docker exec -it furnace furnace token add soar          # the permission grid, then the days
sudo docker exec -it furnace furnace edl-user add paloalto   # the list login for the firewall
```

`token add` asks:

```text
  What may this token do? Type a box number to switch it (or several: 2 4).

                   Read       Write
    Blocklist     1 [x]    2 [x]
    Whitelist     3 [x]    4 [x]

  Box number, or Enter when done:

  Expires after how many days? (0 = never): 365
```

Each command shows its secret **once** (the token `fur_...`, the password). **Store both now** in your password safe.
They are never in the container log. Lost one? `sudo docker exec -it furnace furnace token rotate soar` or
`... edl-user rotate paloalto`. (`-it` is needed: they ask questions.)

### A15. Make a join code for host B

```bash
sudo docker exec furnace furnace join-code
```

```text
Join code for the follower (works once, for 60 minutes, only from 192.0.2.12):

    FJ1-ABCD-EFGH-IJKL-MNOP-QRST-UVWX-YZ23-4567

On the follower host, add this line to .env, then start the follower:
    FURNACE_JOIN_CODE=FJ1-ABCD-EFGH-IJKL-MNOP-QRST-UVWX-YZ23-4567
```

Copy the code. It works **once**, for **1 hour**, only from host B. 5 wrong tries delete it. A new code replaces the old one.

Host A is done. Leave it running.

## Part 2. Host B: the follower (192.0.2.12)

Do every step of this part on **host B**. The first steps are the same as on host A, with host A's IP where host A had host B's.

### B1. Log in and check the system

Log in to host B (192.0.2.12) with SSH as your normal user (not root). Then:

```bash
grep PRETTY_NAME /etc/os-release    # which system is this?
sudo -v                              # can you use sudo? (asks your password)
```

You should see `PRETTY_NAME="Amazon Linux 2023"` (the last numbers may be newer). Another system? Use the guide
for that system instead: [all guides](../README.md).

### B2. Bring the system up to date

```bash
sudo dnf upgrade -y
```

If it installed a new kernel, restart now (`sudo reboot`) and log in again. A restart later would move the shared IP.

### B3. Find the network card and check the IP

```bash
ip -br addr
```

You see one line per card. Example on host B:

```text
lo               UNKNOWN        127.0.0.1/8 ::1/128
ens192           UP             192.0.2.12/24 fe80::250:56ff:fe8a:1b2c/64
```

- The card with `192.0.2.12` is the one Furnace uses. Its name here is `ens192`. Yours may be `eth0`, `ens160`,
  `ens33` or `enp1s0`: write down **your** name.
- The number after the slash is the prefix (`/24`). The shared IP uses the same one.
- The shared IP `192.0.2.10` must **not** be in this list yet, on either host.

Check that nobody uses the shared IP yet:

```bash
ping -c 3 192.0.2.10            # should FAIL: 100% packet loss
ping -c 3 192.0.2.11            # the other host: should answer (if ICMP is allowed)
```

If `192.0.2.10` answers, another machine has it. Pick another free address for the shared IP.

### B4. Check that port 443 is free

```bash
sudo ss -ltnp 'sport = :443'
```

You should see only the header line (`State  Recv-Q  Send-Q ...`). If a program is listed (for example `apache2`,
`httpd` or `nginx` with another site), port 443 is taken. Furnace needs it: stop and remove that program first,
or use another server.

### B5. Install Docker and the Compose plugin

```bash
# 1. Docker comes from Amazon's own repository
sudo dnf install -y docker
sudo systemctl enable --now docker

# 2. The Compose plugin is not packaged on Amazon Linux 2023: put Docker's own file in place
sudo mkdir -p /usr/local/lib/docker/cli-plugins
sudo curl -fL -o /usr/local/lib/docker/cli-plugins/docker-compose \
    "https://github.com/docker/compose/releases/latest/download/docker-compose-linux-$(uname -m)"
sudo chmod 755 /usr/local/lib/docker/cli-plugins/docker-compose
```

```bash
sudo docker compose version
```

### B6. Open the host firewall

On host B, VRRP comes from **host A** (`192.0.2.11`):

```bash
sudo dnf install -y firewalld          # Amazon Linux 2023: not there by default
sudo systemctl enable --now firewalld
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.0.2.11" protocol value="vrrp" accept'
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```
You should see `services: ... https ssh` and the rich rule with `192.0.2.11` and `vrrp`.

### B7. Check SELinux

```bash
getenforce
```

SELinux is **Permissive** by default on Amazon Linux 2023 (it logs, but blocks nothing). Keep it as it is. The container needs nothing (Docker CE runs it without SELinux labels by default). The host's keepalived needs a label on the two Furnace scripts: the keepalived step below does it.

### B8. Install keepalived on the host (if it is not there yet)

**a) Is keepalived already in use on this host?**

```bash
systemctl is-active keepalived
```

- `active`: keepalived already runs here for other services. Go to **d)**, case 2.
- `inactive`, `unknown` or `failed`: keepalived is new on this host. Do **b)**, **c)** and **d)** case 1.

**b) Install keepalived and curl:**

```bash
sudo dnf install -y keepalived curl policycoreutils-python-utils
```

**c) Back up the old file** (if there is one):

```bash
sudo cp -a /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.before-furnace 2>/dev/null; ls /etc/keepalived/
```

**d) Write the top of `/etc/keepalived/keepalived.conf`.**

*Case 1: keepalived is new on this host.* On Amazon Linux 2023, a fresh keepalived package comes with an **example** `/etc/keepalived/keepalived.conf` (with `vrrp_instance VI_1`, `virtual_router_id 51` and example IPs `192.168.200.x`). Its router ID 51 would clash with Furnace's. The command below **replaces** that example file.

```bash
sudo tee /etc/keepalived/keepalived.conf >/dev/null <<'EOF'
global_defs {
    script_user root
    enable_script_security
}
EOF
```

*Case 2: keepalived already runs here for other services.* Keep your file. Open it:

```bash
sudo nano /etc/keepalived/keepalived.conf
```

- If it has a `global_defs { ... }` block, add these two lines inside it:
  `script_user root` and `enable_script_security`.
- If it has no `global_defs` block, add this at the very top of the file:

   ```text
   global_defs {
       script_user root
       enable_script_security
   }
   ```

  Save with Ctrl+O, Enter, Ctrl+X. **Do not** restart keepalived yet.

**e) Check it:**

```bash
sudo head -5 /etc/keepalived/keepalived.conf
```

```text
global_defs {
    script_user root
    enable_script_security
}
```

(In case 2 you may see your own lines too. The two lines must be inside `global_defs`.)

What the two lines do: `script_user root` lets keepalived run the Furnace scripts as root. `enable_script_security`
makes it refuse scripts that others could change (the Furnace scripts are owned by root, mode 755).

### B9. Download Furnace and load the image

```bash
mkdir ~/furnace && cd ~/furnace
curl -fLO https://github.com/MahammadRahimov/furnace-releases/releases/download/v0.10.0/furnace-image-0.10.0.tar.gz
curl -fLO https://github.com/MahammadRahimov/furnace-releases/releases/download/v0.10.0/docker-compose.yml
sudo docker load -i furnace-image-0.10.0.tar.gz     # "Loaded image: furnace:0.10.0"
```

### B10. Copy `.env` from host A and add the join code

Copy host A's `.env` **as it is**. Nothing is swapped: host B finds its role from its own IP.

```bash
scp you@192.0.2.11:furnace/.env ~/furnace/.env
chmod 600 ~/furnace/.env
echo 'FURNACE_JOIN_CODE=FJ1-ABCD-EFGH-IJKL-MNOP-QRST-UVWX-YZ23-4567' >> ~/furnace/.env
cat ~/furnace/.env | grep -v '^#'          # the same IPs as on host A, plus the code
```

No SSH between the hosts? Open `.env` on host A, copy the text, and paste it into `nano ~/furnace/.env` on host B.

### B11. Check and start host B

```bash
cd ~/furnace
sudo docker compose run --rm furnace check
```

```text
Settings are fine: this host is the follower (192.0.2.12 on ens192), the other host is 192.0.2.11, shared IP 192.0.2.10/24.
```

```bash
sudo docker compose up -d
sudo docker logs furnace
```

```text
furnace: Starting Furnace 0.10.0 (Docker).
furnace: This host is the follower (192.0.2.12); the other host is 192.0.2.11.
furnace: Certificate and secrets fetched from the leader 192.0.2.11 with the join code. You can remove FURNACE_JOIN_CODE from .env now (it is used up).
...
furnace: Ready. This host is the follower by setting; the HOST's keepalived decides who owns 192.0.2.10 (FURNACE_KEEPALIVED=external).
```

The code is used up. Remove the line (optional, it is not used again):

```bash
sed -i '/^FURNACE_JOIN_CODE=/d' ~/furnace/.env
```

**The join code did not work?** The log says why, and the container tries again by itself:

| Message | Fix |
|---|---|
| *Cannot reach the leader at https://192.0.2.11:443 (...)* | Port 443 from host B to host A is closed: the network or host A's firewall (step A6). Or Furnace on host A is not running. |
| *No join code is waiting on the leader* | The code was used already, or never made. Make a new one on host A, put it in `.env` on host B (replace the old line), then `sudo docker compose up -d`. |
| *The join code is older than 1 hour* | Make a new one on host A, put it in `.env` on host B (replace the old line), then `sudo docker compose up -d`. |
| *The join code is wrong / is not complete or has a typo* | Copy it again, all of it, from host A. |
| *5 wrong join codes: the code is gone* | Make a new one on host A, put it in `.env` on host B (replace the old line), then `sudo docker compose up -d`. |
| *The join code was made by ..., but the leader here is ...* | Host B has another leader IP than host A. Use the same settings on both. |
| *The server at 192.0.2.11 is not the leader that made this code* | Another machine answers on host A's IP. Nothing was sent. Check the IP. |

**No join code possible?** Copy the bundle file instead:

```bash
# host A: take the bundle out of the container (it is secret)
sudo docker cp furnace:/etc/furnace/furnace-peer-bundle.tar.gz ~/ && sudo chown $USER ~/furnace-peer-bundle.tar.gz
chmod 600 ~/furnace-peer-bundle.tar.gz

# host B: put it into the config folder, then start
mkdir -p ~/furnace/config
scp you@192.0.2.11:furnace-peer-bundle.tar.gz ~/furnace/config/
cd ~/furnace && sudo docker compose up -d       # it uses the bundle and deletes it

# host A again: delete your copy
rm ~/furnace-peer-bundle.tar.gz
```

### B12. Add Furnace to host B's keepalived

This step puts three things on host B, all **printed by host B's own container**:

- the two scripts: `/usr/local/sbin/furnace-notify` writes `leader` or `follower` into `/run/furnace/role`, and
  `/usr/local/sbin/furnace-health-check` asks `https://127.0.0.1/health` (3 failures in a row and keepalived gives
  the shared IP away),
- the Furnace `vrrp_instance` block, added **below** the other blocks in `/etc/keepalived/keepalived.conf` (the
  backup of the old file was made in the keepalived install step),
- then a file check and a keepalived restart.

Run on **host B**:

```bash
sudo docker exec furnace furnace host-keepalived notify       | sudo tee /usr/local/sbin/furnace-notify >/dev/null
sudo docker exec furnace furnace host-keepalived health-check | sudo tee /usr/local/sbin/furnace-health-check >/dev/null
sudo chmod 755 /usr/local/sbin/furnace-notify /usr/local/sbin/furnace-health-check
sudo chown root:root /usr/local/sbin/furnace-notify /usr/local/sbin/furnace-health-check
sudo semanage fcontext -a -t keepalived_unconfined_script_exec_t '/usr/local/sbin/furnace-(notify|health-check)'
sudo restorecon -v /usr/local/sbin/furnace-notify /usr/local/sbin/furnace-health-check

sudo docker exec furnace furnace host-keepalived config | sudo tee -a /etc/keepalived/keepalived.conf >/dev/null
sudo keepalived -t -f /etc/keepalived/keepalived.conf && echo "file OK"
sudo systemctl enable keepalived
sudo systemctl restart keepalived
sleep 5
cat /run/furnace/role                       # follower
```

Host B's block says `priority 100`, `unicast_src_ip 192.0.2.12` and `unicast_peer 192.0.2.11`: the other way round from host A.
That is right. **Never copy host A's block to host B.**

### B13. Check host B

```bash
sudo docker exec furnace furnace status
```

```text
Furnace 0.10.0 (Docker)
Role:     follower (the other host owns the shared IP 192.0.2.10)
keepalived: the host's own (FURNACE_KEEPALIVED=external)
Host:     192.0.2.12 (the follower by setting), other host 192.0.2.11

Processes:
  api          running
  nginx        running

Replication: not needed (follower)
```

```bash
ip -br addr show ens192     # only 192.0.2.12/24: the shared IP stays on host A
```

## Part 3. Both hosts together

### 3.1 Replication works

On **host A**, wait 20 seconds:

```bash
sudo docker exec furnace furnace status | grep Replication     # Replication: ok ...
```

On **host B**, the token and the login from host A are there:

```bash
sudo docker exec furnace furnace token list        # soar
sudo docker exec furnace furnace edl-user list     # paloalto
```

### 3.2 Failover test

On **host A**:

```bash
sudo docker stop furnace
```

Host A's health check fails 3 times; after about **15 seconds** host A's keepalived gives the Furnace shared IP
to host B. **Your other shared IPs do not move.**

On **host B**:

```bash
ip -br addr show ens192                              # now has 192.0.2.10
sudo docker exec furnace furnace status | head -2      # Role: leader
```

Then on **host A**:

```bash
sudo docker start furnace
```

Host A comes back as the **follower** and keeps it: a host that comes back does not take the shared IP back. It gets
the new data within 20 seconds. To move the shared IP back, stop and start the container on host B the same way.

### 3.3 Test from another machine

```bash
curl -k -u paloalto:PASSWORD https://192.0.2.10/edl/blocklist.txt    # empty answer, no error
curl -k -u paloalto:WRONG https://192.0.2.10/edl/blocklist.txt       # 401
```

### 3.4 Get the certificate

Both hosts use the same self-signed certificate. It is **public**, not secret. Copy it to your PC:

```bash
sudo docker exec furnace cat /etc/furnace/tls/furnace.crt > ~/furnace.crt
# on your PC:
scp you@192.0.2.11:furnace.crt .
```

The firewall and the SOAR must trust it. Then they check that they really talk to Furnace.

### 3.5 Connect the firewall (Palo Alto)

Palo Alto calls the lists **External Dynamic Lists (EDLs)**. Other firewalls have the same idea under other names
(threat feed, external block list): use the same addresses and login.

**a) Trust the certificate**

1. *Device > Certificate Management > Certificates > Import*. Import `furnace.crt`. Tick **Trusted Root CA**.
2. *Device > Certificate Management > Certificate Profile > Add*. Name it `Furnace`, add `furnace.crt` as the CA.
3. Commit.

**b) The three lists.** *Objects > External Dynamic Lists > Add*, once per list:

| Name (example) | Type | Source | Use it in |
|---|---|---|---|
| `Furnace-IPs` | IP List | `https://192.0.2.10/edl/blocklist.txt` | security rules, action deny (source and/or destination) |
| `Furnace-Domains` | Domain List | `https://192.0.2.10/edl/domains.txt` | an Anti-Spyware profile, DNS policies: blocks the DNS lookup |
| `Furnace-URLs` | URL List | `https://192.0.2.10/edl/urls.txt` | a custom URL category, in a deny rule or a URL Filtering profile (only if you block URLs) |

For every list:
- **Certificate Profile:** `Furnace`.
- **Client Authentication:** user `paloalto` and its password.
- **Check for updates:** every five minutes.
- **The type must match the list:** IP List for `blocklist.txt`, Domain List for `domains.txt`, URL List for
  `urls.txt`. All three are **block** lists: use them only in deny/block rules, never in an allow rule. Furnace never
  publishes the whitelist, so no list here can hold a whitelisted entry.

**c) Commit and check.** In the EDL object, click *Test Source URL*. Or on the firewall CLI:

```text
request system external-list show type ip name Furnace-IPs
```

An empty list is fine at first. For URLs: deny the `quic` application too, so browsers use HTTPS the firewall can
read. Paths in URLs work only with SSL decryption.

**d) Good to know**
- The firewall downloads EDLs from its **management interface**, unless you set a *service route* for External
  Dynamic Lists. That source IP must reach `192.0.2.10` on TCP 443.
- Check your firewall model's EDL limit per list. Furnace refuses new entries beyond `50000` by default
  (answer `limit_reached`).
- If both Furnace hosts are down, the firewall keeps the last list it downloaded and keeps blocking.

### 3.6 Connect the SOAR (the API)

Furnace needs no plugin: the SOAR (for example Cortex XSOAR) sends plain HTTPS calls.

**a) Store the token** (`fur_...`, from Part 1) in the SOAR's credentials store. Give every program its own token
with only the boxes it needs: `sudo docker exec -it furnace furnace token add NAME`.

**b) Trust the certificate:** add `furnace.crt` as a trusted custom certificate on the SOAR server **and its
engines** (the calls come from the engines' IPs).

**c) The calls.** Always the shared IP. JSON in and out. Header `Authorization: Bearer TOKEN`.

```bash
FURNACE=https://192.0.2.10
TOKEN='fur_...'
CA=furnace.crt

# block an IP (add "permanent": true to keep it for ever)
curl --cacert "$CA" -X POST "$FURNACE/api/v1/ips" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"ip":"203.0.113.45","reason":"Phishing C2 server","added_by":"analyst.one"}'

# look it up, unblock it
curl --cacert "$CA" "$FURNACE/api/v1/ips/203.0.113.45" -H "Authorization: Bearer $TOKEN"
curl --cacert "$CA" -X DELETE "$FURNACE/api/v1/ips/203.0.113.45?removed_by=analyst.one" -H "Authorization: Bearer $TOKEN"

# block a domain, a URL
curl --cacert "$CA" -X POST "$FURNACE/api/v1/domains" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"domain":"bad.example.com","reason":"Malware site","added_by":"analyst.one"}'
curl --cacert "$CA" -X POST "$FURNACE/api/v1/urls" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"url":"https://bad.example.com/login/","reason":"Phishing page","added_by":"analyst.one"}'

# whitelist an IP range (needs a token with whitelist write)
curl --cacert "$CA" -X POST "$FURNACE/api/v1/whitelist/ips" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"entry":"198.51.100.0/24","reason":"Partner network","added_by":"admin.one"}'
```

Production refuses the example ranges (`203.0.113.x`, `198.51.100.x`, `192.0.2.x`) with `documentation_range`. That
refusal is a good first test. Then use a real IP.

**d) Read the answer.** Every answer has `status` and a plain `message`: show `message` to the analyst.

| Answer | What the SOAR should do |
|---|---|
| `201` `added` | Tell the analyst: blocked. |
| `200` `already_exists` | Tell the analyst: already blocked since the date in `message`. |
| `422` `rejected` | Show `message` (`reason_code` says why: `whitelisted`, `private_ip`, `limit_reached`, ...). Do not retry. |
| `401` | The token is wrong, expired or lacks a box. Alert the admin. **Do not retry**: wrong tokens lock the caller. |
| `429` `locked` | 10 wrong tokens in 15 minutes locked this address. Fix the token, then on the leader: `sudo docker exec furnace furnace unlock CALLER_IP`. |
| `503`, `429`, network error | Retry a few times with a pause. `503 not_leader` means it reached the standby host: use the shared IP. |

Send the analyst's name in `added_by` and a short reason in `reason` (an incident number fits there too).


## Part 4. Later

**Daily work** (on the host that owns the shared IP):

```bash
sudo docker exec furnace furnace status
sudo docker exec -it furnace furnace token add NAME
sudo docker cp ips.txt furnace:/tmp/ips.txt && sudo docker exec furnace furnace import-ips /tmp/ips.txt
sudo docker logs --since 1h furnace
```

**Change a setting:** edit `.env` on **both** hosts the same way, then `sudo docker compose up -d` on both (it starts
the container again). A new VRRP router ID or shared IP also needs a new Furnace block in keepalived.conf: print it again, replace the old block, restart keepalived (both hosts).

**Backup:** the leader makes a daily backup in `~/furnace/data/backups`. To be safe from a lost host, copy
`~/furnace/config` and `~/furnace/data` somewhere else from time to time (`config` is secret).

**Upgrade** to a newer version, on both hosts (the leader first, then the follower; `furnace status` shows which is
the leader now):

```bash
cd ~/furnace
curl -fLO https://github.com/MahammadRahimov/furnace-releases/releases/download/vNEW/furnace-image-NEW.tar.gz
curl -fLO https://github.com/MahammadRahimov/furnace-releases/releases/download/vNEW/docker-compose.yml        # names the new image; .env stays
sudo docker load -i furnace-image-NEW.tar.gz
sudo docker compose up -d
```

The database is updated at start. Data and secrets stay in `config` and `data`. Each release has notes on the
GitHub release page: read them before you upgrade. Coming from 0.9: an old `.env` with `FURNACE_THIS_IP` /
`FURNACE_PEER_IP` keeps working; the log suggests the new names (`FURNACE_LEADER_IP`, `FURNACE_FOLLOWER_IP`).

**Remove Furnace** (on both):

```bash
cd ~/furnace && sudo docker compose down
sudo docker image rm furnace:0.10.0
# the data stays in ~/furnace/config and ~/furnace/data until you delete the folder
```

Then remove the Furnace block from `/etc/keepalived/keepalived.conf`, restart keepalived, and delete `/usr/local/sbin/furnace-notify` and `furnace-health-check`.

## Troubleshooting on Amazon Linux 2023

| Problem | What to check |
|---|---|
| Both hosts say leader / both have the shared IP | VRRP (protocol 112) is blocked between the hosts. Check the network and both host firewalls. `sudo journalctl -u keepalived --since '10 min ago'` on both hosts: *Receive advertisement timeout* and *Entering MASTER STATE* on both = no VRRP arrives. |
| The shared IP never comes up | On VMs: MAC address changes / forged transmits / MAC spoofing (Part 0). Is the VRRP router ID used by another device? |
| Replication: failed | Port 443 between the hosts, nginx on the other host, the same certificate on both. |
| Permission denied in the logs (SELinux) | `sudo ausearch -m avc -ts recent` shows what SELinux blocked. For the host keepalived scripts, repeat the `semanage` and `restorecon` lines. |
| Port 443 works locally but not from outside | `sudo firewall-cmd --list-all`: is the rule in the **default** zone? `firewall-cmd --get-active-zones` shows which zone the card uses. |

Logs: `sudo docker logs --since 1h furnace`, `sudo journalctl -u keepalived`.
