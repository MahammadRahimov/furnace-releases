# Furnace 0.11.0 on Amazon Linux 2023: the .rpm package

[All guides](../README.md) · Same system, other ways: [Docker, keepalived in the container](../docker/amazon-2023.md) · [Docker, keepalived on the host](../docker-host-keepalived/amazon-2023.md)

Furnace is installed as a normal **.rpm** package on two Amazon Linux 2023 servers. `sudo furnace setup` asks a few questions and sets up everything: the API, nginx, keepalived (the shared IP), the host firewall, replication, the daily backup and the nightly cleanup.

**Use this guide when** both servers run only Furnace. Setup writes the whole `/etc/keepalived/keepalived.conf`. If the servers already use keepalived for other services, use [Docker, keepalived on the host](../docker-host-keepalived/amazon-2023.md) instead.

> **Inside AWS, a shared IP that moves between two machines does not work** (AWS does not allow an IP to move by ARP). Use Amazon Linux 2023 on your own VMware, KVM or Hyper-V, where the network allows it.

## The plan

You need three IP addresses in **one network segment**. This guide uses these examples. Write your own next to them
before you start, and use yours everywhere the guide says the example:

| | Example | Yours |
|---|---|---|
| Host A: the leader server | `192.0.2.11` | |
| Host B: the follower server | `192.0.2.12` | |
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

- Two servers (real or VMs) with **Amazon Linux 2023**, the same system on both. 1 CPU, 1 GB memory and 5 GB free disk
  are enough; 2 GB memory is better with large lists.
- A user with `sudo` on both.
- The three IPs from the plan, all in one network segment.
- Both servers can reach the Amazon Linux 2023 package mirror (directly, through a proxy or a local mirror) and
  github.com (to download Furnace; or copy the package there yourself).
- Port 443 free on both (no other web server).
- keepalived on these servers is **for Furnace only** (setup replaces its settings file; an old one is saved as
  `keepalived.conf.before-furnace.*`).

### Network: what must be open

Ask your network team to allow these **before** you start. "Each host" means both directions.

| From | To | Port | Why |
|---|---|---|---|
| the firewall that downloads the lists (its source IP) | shared IP `192.0.2.10` | TCP 443 | the lists |
| the SOAR or script that calls the API (and its engines) | shared IP `192.0.2.10` | TCP 443 | the API |
| each host | the other host | TCP 443 | replication every 20 seconds, the join code |
| each host | the other host | IP protocol 112 (VRRP) | keepalived: who owns the shared IP |
| each host | the other host | TCP 22 (SSH) | only if you turn on the optional SSH backup copy |
| your admin PC | both hosts | TCP 22 (SSH) | your own login |
| both hosts | their package mirror and github.com | TCP 443 | install the packages and download Furnace |

**Virtual machines:** the shared IP moves by ARP. On **VMware**, the port group of both VMs must allow
"MAC address changes" and "Forged transmits" (or use a distributed switch with these set). On **Hyper-V**, turn on
"Enable MAC address spoofing" on both VMs' network adapters. Most **clouds** (AWS, Azure, GCP) do not allow a moving
IP at all: use real servers or your own virtualization.

**VRRP is the part that most often goes wrong.** If protocol 112 is blocked between the hosts, both hosts take the
shared IP at the same time ("split brain"). Ask for "IP protocol 112", not a port: VRRP has no port.

### What is special on Amazon Linux 2023

- **Package:** `furnace-0.11.0-1.noarch.rpm`, installed with `dnf`. It pulls in nginx, keepalived, Python, OpenSSL, curl, SQLite and
  firewalld and `policycoreutils-python-utils` (for SELinux) from the Amazon Linux 2023 mirror.
  Flask and gunicorn are inside the Furnace package.
- **Host firewall:** `firewalld` (off by default).
- **SELinux:** permissive by default; setup sets the labels.
- **nginx site file:** `/etc/nginx/conf.d/furnace.conf`.
- **Services:** `furnace-api`, `nginx`, `keepalived`; timers `furnace-replicate`, `furnace-backup`, `furnace-cleanup`.

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

### A5. Check the host firewall

Amazon Linux 2023 has **no host firewall** by default. The Furnace package pulls in `firewalld` (a weak dependency), and it starts **off**. Look at it:

```bash
systemctl is-active firewalld; systemctl is-enabled firewalld
sudo firewall-cmd --get-default-zone 2>/dev/null
```

What you see, and what setup will do:

| You see | Setup |
|---|---|
| `active` and `enabled` | asks *Let Furnace set up this server's own firewall (firewalld)? [yes]*. It adds rich rules to the default zone (usually `public`). |
| `inactive` (or *could not be found*) | asks *Turn on and set up this server's own firewall (firewalld)? [yes]*. SSH stays open. |

### A6. Check SELinux

```bash
getenforce
```

SELinux is **Permissive** by default on Amazon Linux 2023 (it logs, but blocks nothing). Keep it as it is. Setup handles it for you: it labels the list folders for nginx, labels the two keepalived scripts, and turns on `httpd_can_network_connect` so nginx may pass API calls to Furnace. It needs the `semanage` command, which the package pulls in (`policycoreutils-python-utils`).

### A7. Download the package

```bash
cd ~
curl -fLO https://github.com/MahammadRahimov/furnace-releases/releases/download/v0.11.0/furnace-0.11.0-1.noarch.rpm
ls -l furnace-0.11.0-1.noarch.rpm
```

No direct internet on the server? Download it on your PC from <https://github.com/MahammadRahimov/furnace-releases/releases>
(release v0.11.0) and copy it over: `scp furnace-0.11.0-1.noarch.rpm you@192.0.2.11:~/`.

### A8. Install the package

```bash
sudo dnf install ./furnace-0.11.0-1.noarch.rpm
```

dnf shows the extra packages it will install and asks *Is this ok* / *Do you want to continue*: answer `y`.
**Nothing starts yet**: Furnace waits for setup. The package ends with the line *After the install, run: sudo furnace setup*.

### A9. Run setup and answer the questions

```bash
sudo furnace setup
```

Press **Enter** to take the answer in [brackets]. A wrong answer is explained and asked again. Here is the whole
dialog on host A, with the example answers:

```text
Furnace setup. Press Enter to take the answer in [brackets].

  Leader = the first server. Follower = the second (standby) server. Set up the leader first.
  Is this server the leader or the follower? [leader]:

  The network card that carries this server's IP and the shared IP: its name, not its IP. See: ip -br addr
  Network card name [ens192]:
  This server's IP (the leader) [192.0.2.11]:
  The follower server's IP: 192.0.2.12

  The shared IP (VIP) is a free third address. Everything that uses Furnace connects to it.
  Shared IP: 192.0.2.10
  Network prefix of the shared IP (24 = 255.255.255.0) [24]:

  Who may connect. 'any' = every address: the API still needs a token and the lists need a login.
    Or limit it to IPs and ranges, separated by commas.
  Who may call the API [any]:
  Who may download the lists [any]:

  The firewall's EDL limit. Adds beyond it are refused. 0 = no limit. Used for every list (IPs, domains, URLs).
  Most entries per list [50000]:
  Days an IP stays blocked (permanent IPs never expire) [90]:

  VRRP router ID: the same on both servers, unique in this network. Keep 51 if you are not sure.
  VRRP router ID (1-255) [51]:

  Who may log in with SSH (people, for example admins): 'any', or IPs separated by commas.
  SSH allowed from [any]:

  The leader copies the data to the follower over HTTPS. SSH can be a backup for when HTTPS does not work.
    Most servers do not need it. It adds a user (furnace-sync) and SSH between the two servers.
  Use SSH as a backup for the copy to the other server? [no]:

  The host firewall firewalld is installed but off. Furnace allows SSH before it turns it on.
  Turn on and set up this server's own firewall (firewalld)? [yes]:

    The first API token, for the system that calls the API (for example a SOAR or a script).
  Token name [api]: soar

  What may this token do? Type a box number to switch it (or several: 2 4).

                   Read       Write
    Blocklist     1 [x]    2 [x]
    Whitelist     3 [x]    4 [x]

  Box number, or Enter when done:

    Each token has its own expiry. Furnace warns 14 days before. 0 = never.
  Token expires after how many days: 365
```

What each answer means:

| Question | Example answer | What to type |
|---|---|---|
| Leader or follower | Enter (`leader`) | Host A is always the leader. |
| Network card name | Enter (`ens192`) | The name from step A3. Setup suggests the card of the default route. |
| This server's IP | Enter (`192.0.2.11`) | Host A's own IP. It must be on that card. |
| The follower server's IP | `192.0.2.12` | Host B's IP. |
| Shared IP | `192.0.2.10` | The free third address. |
| Network prefix | Enter (`24`) | The same as in step A3. |
| Who may call the API | Enter (`any`) | `any`, or the SOAR's IPs (and its engines), for example `198.51.100.20,198.51.100.21`. A token is always needed. |
| Who may download the lists | Enter (`any`) | `any`, or the firewall's source IPs. For a firewall HA pair, list both. The login is always needed. |
| Most entries per list | Enter (`50000`) | Your firewall's limit per list (Palo Alto: check your model). |
| Days an IP stays blocked | Enter (`90`) | Temporary IPs leave the list after this many days. Permanent IPs and domains stay. |
| VRRP router ID | Enter (`51`) | 1 to 255. Must not be used by any other keepalived or VRRP device in this network segment. Ask your network team. |
| SSH allowed from | Enter (`any`) | Who may log in with SSH: `any`, or your admin PCs, for example `198.51.100.5`. **Include your own PC**, or you lock yourself out. |
| SSH as a backup copy | Enter (`no`) | `no` for almost everyone. `yes` adds the user `furnace-sync` and SSH between the servers, used only when HTTPS fails. |
| Host firewall | Enter (`yes`) | `yes`: setup opens SSH, 443 and VRRP from host B in `firewalld`. `no`: you open them yourself. |
| Token name | `soar` | A name for the program that calls the API. 1 to 40 letters, numbers, `.`, `-`, `_`. |
| The permission grid | Enter | All four boxes are on. Type `4` to switch *whitelist write* off, for example. Write always includes read. |
| Token expires after | `365` | Days, 0 to 3650. 0 = never. There is no default: type a number. |

Then setup shows all answers once more:

```text
  Settings:
    Role                   leader
    Network card           ens192
    This server            192.0.2.11
    Other server           192.0.2.12
    Shared IP              192.0.2.10/24
    API from               any
    Lists from             any
    Most entries per list  50000
    IPs expire after       90 days
    VRRP router ID         51
    SSH allowed from       any
    Host firewall          firewalld
    HTTPS                  yes

  Set up Furnace with these settings? [yes]:
```

Check them, then press **Enter**. (`no` stops; nothing is changed.)

### A10. Read what setup did, and store the two secrets

Setup works through these blocks, each with `[ok]` lines: *Creating users and folders*, *Preparing secrets*,
*Writing configuration files*, *Configuring the host firewall (firewalld)*, *Starting services*, *Self-test*,
*Summary*. It stops at the first error and says what went wrong.

At the end it shows the **new secrets once**:

```text
  ============ NEW SECRETS (shown only once, not in the log) ============
  Store them now. Furnace keeps only a hash and cannot show them again.
  Lost one? Make a new one: sudo furnace token rotate NAME / edl-user rotate NAME

  API token 'soar'. Permissions: blocklist read+write, whitelist read+write. Expires: ... (in 365 days). Token:
      fur_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
  List login (for the device that downloads the lists): user 'lists', password:
      xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
  =======================================================================
```

**Copy both into your password safe now.** The token goes to the SOAR later, the list login (user `lists`) to the
firewall. They are not in the log file. Lost one? `sudo furnace token rotate soar` or `sudo furnace edl-user rotate lists`.

The last block, *Still to do by hand*, lists what setup cannot do. Part 3 of this guide covers it.
The full log is in `/var/log/furnace-setup.log` (no secrets in it).

### A11. Check host A

```bash
sudo furnace status
```

```text
Furnace 0.11.0
Role:     leader (this server owns the shared IP 192.0.2.10)
Server:   192.0.2.11 on ens192, other server 192.0.2.12

Services:
  furnace-api.service          active
  nginx.service                active
  keepalived.service           active
  furnace-replicate.timer      active
  furnace-backup.timer         active
  furnace-cleanup.timer        active

Replication: failed over HTTPS: ... (5 seconds ago)
             <-- the other server does not get the changes
```

- **Role: leader** and *this server owns the shared IP*: right.
- Every service says `active`.
- **Replication failed** is normal now: host B does not exist yet. It turns to `ok` after Part 2.

Check that the shared IP is on host A's card, and that the firewall rules are in:

```bash
ip -br addr show ens192       # now shows 192.0.2.11/24 AND 192.0.2.10/24
```

```bash
sudo firewall-cmd --list-rich-rules
```
```text
rule port port="22" protocol="tcp" accept
rule port port="443" protocol="tcp" accept
rule family="ipv4" source address="192.0.2.12" protocol value="vrrp" accept
```
The order may differ. Setup remembers these rules in `/etc/furnace/firewalld.rules`, so it can remove them again later.

### A12. Make a join code for host B

```bash
sudo furnace join-code
```

It shows a code like `FJ1-ABCD-EFGH-IJKL-MNOP-QRST-UVWX-YZ23-4567`. Copy it.

- It works **once**, for **1 hour**, and only from host B's IP (`192.0.2.12`).
- 5 wrong tries delete it. A new `furnace join-code` replaces the old code.
- Took longer than an hour? Run `sudo furnace join-code` again.

Host A is done. Leave it running.

## Part 2. Host B: the follower (192.0.2.12)

Do every step of this part on **host B**. The first steps are the same checks as on host A.

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

### B5. Check the host firewall

Amazon Linux 2023 has **no host firewall** by default. The Furnace package pulls in `firewalld` (a weak dependency), and it starts **off**. Look at it:

```bash
systemctl is-active firewalld; systemctl is-enabled firewalld
sudo firewall-cmd --get-default-zone 2>/dev/null
```

What you see, and what setup will do:

| You see | Setup |
|---|---|
| `active` and `enabled` | asks *Let Furnace set up this server's own firewall (firewalld)? [yes]*. It adds rich rules to the default zone (usually `public`). |
| `inactive` (or *could not be found*) | asks *Turn on and set up this server's own firewall (firewalld)? [yes]*. SSH stays open. |

### B6. Check SELinux

```bash
getenforce
```

SELinux is **Permissive** by default on Amazon Linux 2023 (it logs, but blocks nothing). Keep it as it is. Setup handles it for you: it labels the list folders for nginx, labels the two keepalived scripts, and turns on `httpd_can_network_connect` so nginx may pass API calls to Furnace. It needs the `semanage` command, which the package pulls in (`policycoreutils-python-utils`).

### B7. Download the package

```bash
cd ~
curl -fLO https://github.com/MahammadRahimov/furnace-releases/releases/download/v0.11.0/furnace-0.11.0-1.noarch.rpm
ls -l furnace-0.11.0-1.noarch.rpm
```

### B8. Install the package

```bash
sudo dnf install ./furnace-0.11.0-1.noarch.rpm
```

Answer `y`. Nothing starts yet.

### B9. Run setup with the join code

```bash
sudo furnace setup
```

Host B asks only a few questions. All shared settings (shared IP, limits, VRRP router ID, the keepalived password,
the certificate) come from host A:

```text
Furnace setup. Press Enter to take the answer in [brackets].

  Leader = the first server. Follower = the second (standby) server. Set up the leader first.
  Is this server the leader or the follower? [leader]: follower
  Join code from the leader (Enter = use the file): FJ1-ABCD-EFGH-IJKL-MNOP-QRST-UVWX-YZ23-4567

  Network card for 192.0.2.12 and the shared IP (192.0.2.10): its name, not its IP.
  Network card name [ens192]:

  The host firewall firewalld is installed but off. Furnace allows SSH before it turns it on.
  Turn on and set up this server's own firewall (firewalld)? [yes]:
```

- **follower**: type it. Host B is always the follower.
- **Join code**: paste the code from step A12. Host B connects to host A on port 443, checks host A's certificate
  against the code, then fetches the certificate and the secrets. A fake server gets nothing.
- **Network card**: the name from step B3.
- The firewall question comes because the firewall is off on host B: press Enter (`yes`).

Then the settings summary (role `follower`, this server `192.0.2.12`, other server `192.0.2.11`) and *Set up Furnace with these
settings? [yes]*: press **Enter**. Setup runs the same blocks as on host A. Its self-test ends with
*the follower refuses changes while it is not the leader (503)*: right. Host B shows **no** new secrets: tokens and
logins come from host A with the replication.

**The join code did not work?**

| Message | Fix |
|---|---|
| *Cannot reach the leader at https://192.0.2.11:443 (...)* | Port 443 from host B to host A is closed: the network or host A's firewall (step A5). Or Furnace on host A is not running. |
| *No join code is waiting on the leader* | The code was used already, or never made. Make a new one on host A, run `sudo furnace setup` on host B again. |
| *The join code is older than 1 hour* | Make a new one on host A, run `sudo furnace setup` on host B again. |
| *The join code is wrong / is not complete or has a typo* | Copy it again, all of it, from host A. |
| *5 wrong join codes: the code is gone* | Make a new one on host A, run `sudo furnace setup` on host B again. |
| *The join code was made by ..., but the leader here is ...* | Host B has another leader IP than host A. Use the same settings on both. |
| *The server at 192.0.2.11 is not the leader that made this code* | Another machine answers on host A's IP. Nothing was sent. Check the IP. |

**No join code possible** (for example port 443 between the hosts opens only later)? Copy the bundle file instead:

```bash
# on host A: copy the bundle to host B (it is secret)
sudo scp /root/furnace-peer-bundle.tar.gz you@192.0.2.12:~/

# on host B: run setup, answer follower, and press Enter at the join code question
sudo furnace setup
```

Setup finds the bundle in your home folder, in `/root` or in `/tmp`, uses it and deletes it.

### B10. Check host B

```bash
sudo furnace status
```

```text
Furnace 0.11.0
Role:     follower (the shared IP 192.0.2.10 is on the other server, 192.0.2.11)
Server:   192.0.2.12 on ens192, other server 192.0.2.11

Services:
  furnace-api.service          active
  nginx.service                active
  keepalived.service           active
  furnace-replicate.timer      active
  furnace-backup.timer         active
  furnace-cleanup.timer        inactive

Replication: this server receives the snapshots from the leader
```

- **Role: follower**, the shared IP is on host A: right.
- `furnace-cleanup.timer inactive` is right: the nightly cleanup runs on the leader only.

```bash
ip -br addr show ens192       # only 192.0.2.12/24: the shared IP stays on host A
```

```bash
sudo firewall-cmd --list-rich-rules
```
```text
rule port port="22" protocol="tcp" accept
rule port port="443" protocol="tcp" accept
rule family="ipv4" source address="192.0.2.11" protocol value="vrrp" accept
```
The order may differ. Setup remembers these rules in `/etc/furnace/firewalld.rules`, so it can remove them again later.

## Part 3. Both hosts together

### 3.1 Replication works

On **host A**, wait 20 seconds, then:

```bash
sudo furnace status | grep Replication
```

```text
Replication: ok over HTTPS: ... (8 seconds ago)
```

On **host B**, the token from host A is there:

```bash
sudo furnace token list        # shows 'soar' (never the token itself)
```

If host A still says *failed*: port 443 from host A to host B is closed, or nginx on host B is not running.

### 3.2 Failover test

On **host A** (the server that owns the shared IP):

```bash
sudo furnace failover-test
```

It stops keepalived on host A (like a dead machine), waits until host B owns the shared IP and answers as the leader,
then starts keepalived again. About 15 seconds. Afterwards **host B stays the leader**: a host that comes back does
not take the shared IP back. Check on both:

```bash
sudo furnace status | head -3
```

Host B now says `Role: leader`, host A `Role: follower`. That is fine. To move it back, run
`sudo furnace failover-test` on host B. `sudo furnace failover-test --web` tests a broken nginx instead.

### 3.3 Test from another machine

From your PC, or from the firewall's network:

```bash
curl -k -u lists:PASSWORD https://192.0.2.10/edl/blocklist.txt     # an empty answer (the list is empty), no error
curl -k -u lists:WRONG https://192.0.2.10/edl/blocklist.txt        # 401: a wrong password is refused
```

`-k` skips the certificate check for this test only.

### 3.4 Get the certificate

Both hosts use the same self-signed certificate. It is **public**, not secret. Copy it to your PC:

```bash
sudo cp /etc/furnace/tls/furnace.crt ~/ && sudo chown $USER ~/furnace.crt
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
- **Client Authentication:** user `lists` and its password.
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
with only the boxes it needs: `sudo furnace token add NAME`.

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
| `429` `locked` | 10 wrong tokens in 15 minutes locked this address. Fix the token, then on the leader: `sudo furnace unlock CALLER_IP`. |
| `503`, `429`, network error | Retry a few times with a pause. `503 not_leader` means it reached the standby host: use the shared IP. |

Send the analyst's name in `added_by` and a short reason in `reason` (an incident number fits there too).


### 3.7 Clean up

On host A, the bundle `/root/furnace-peer-bundle.tar.gz` holds the certificate key and the secrets. Keep it only if
you want to set up host B again without a join code. Otherwise:

```bash
sudo rm /root/furnace-peer-bundle.tar.gz
```

Delete the package files too, if you like: `rm ~/furnace-0.11.0-1.noarch.rpm` (on both).

## Part 4. The same without questions (automation)

For Ansible or other scripts. **One settings file for both hosts**:

```bash
cp /usr/share/doc/furnace/config.env.example config.env
nano config.env
```

```text
LEADER_IP=192.0.2.11
FOLLOWER_IP=192.0.2.12
SHARED_IP=192.0.2.10
SHARED_IP_PREFIX=24
INTERFACE=ens192
TOKEN_NAME=soar
TOKEN_ALLOW=all
TOKEN_EXPIRES_DAYS=365
HOST_FIREWALL=firewalld       # needed only to let setup turn on a firewall that is off
```

```bash
# host A
sudo dnf install -y ./furnace-0.11.0-1.noarch.rpm
sudo furnace setup --check --config config.env    # shows what it would do, changes nothing
sudo furnace setup --config config.env
sudo furnace join-code                            # copy the code

# host B: the same file, plus the code (used once, never saved)
sudo dnf install -y ./furnace-0.11.0-1.noarch.rpm
echo 'JOIN_CODE=FJ1-...' >> config.env
sudo furnace setup --config config.env
```

Without a person at the terminal, setup writes the new secrets to `/root/furnace-new-secrets.txt` on host A. Read
it, store the secrets, then delete the file.

## Part 5. Later

**Daily work** (on the server that owns the shared IP; changes go to the other server within 20 seconds):

```bash
sudo furnace status
sudo furnace token add NAME           # one token per program
sudo furnace edl-user add NAME        # one list login per firewall
journalctl -u furnace-api --since today
```

**Change a setting:** run `sudo furnace setup` again. The brackets show the current answers. Change a shared setting
(shared IP, VRRP router ID, limits) on **both** hosts with the same answer.

### Update Furnace

Nothing updates by itself, and only Furnace changes: the other packages on the server stay as they are.

**Once, on both hosts:** add the signed Furnace package repository (the key comes with the package):

```bash
sudo furnace repo add          # "It works. Newest Furnace there: ..."
```

Once a day each host checks whether a newer version is out (it reads one small public file and sends nothing
about the server). `sudo furnace status` then shows a line `Updates:  A new version is available: ...`.

**When there is a new version:** first the host **without** the shared IP (`sudo furnace status` shows the role;
after a failover that can be host A), then the other one:

```bash
sudo furnace upgrade --check   # is there a new version? Changes nothing.
sudo furnace upgrade           # asks, then: a database copy, the new package, a self-check
```

On the host with the shared IP it stops with a message until the other host has the new version. Each release has
notes on the GitHub release page: read them before you update.

**Going back** to the version before: `sudo furnace upgrade --rollback`, on both hosts.
**A host without internet:** download the new package on another computer, copy it to the host, then
`sudo furnace upgrade --file furnace-NEW-1.noarch.rpm`.
**No daily check:** `UPDATE_CHECK=no` in `/etc/furnace/setup.conf`, then `sudo furnace setup --upgrade`.

If the setup step of an update reports a problem, the package stays installed: fix it and run
`sudo furnace setup --upgrade` (details in `/var/log/furnace-setup.log`).

### Coming from 0.9 or 0.10

0.9 and 0.10 have no `furnace upgrade`, so the update to 0.11.0 is the last one by hand. First the host without
the shared IP, then the other one:

```bash
curl -fLO https://github.com/MahammadRahimov/furnace-releases/releases/download/v0.11.0/furnace-0.11.0-1.noarch.rpm
sudo dnf install ./furnace-0.11.0-1.noarch.rpm
sudo furnace repo add          # from now on: sudo furnace upgrade
```

The package runs `furnace setup --upgrade` by itself: no questions, data and secrets kept, the shared IP does not
move. `dnf` shows the version as `1:0.11.0` (the `1:` keeps Furnace newer than other programs called
`furnace`); `furnace version` shows `0.11.0`.
- **From 0.9:** the servers copy over HTTPS only, and the update removes the old SSH copy login (`furnace-sync`).
  Your own SSH rule stays. `setup.conf` gets the new IP names (`LEADER_IP`, `FOLLOWER_IP`).
- **From 0.7 or older:** the update keeps the SSH backup copy on, because a 0.7 server can copy only over SSH.
  When both run 0.11.0, turn it off: `sudo furnace setup` on both, answer `no` to the SSH backup question.

### Uninstall

On both hosts:

```bash
sudo dnf remove furnace     # program gone; data, tokens and logins kept

# RPM has no purge. To delete all Furnace data afterwards:
sudo rm -rf /var/lib/furnace /var/lib/furnace-edl /var/lib/furnace-auth /var/lib/furnace-incoming \
    /var/lib/furnace-sync /var/lib/furnace-upgrade /etc/furnace /root/furnace-peer-bundle.tar.gz /var/log/furnace-setup.log
sudo userdel furnace-sync 2>/dev/null; sudo userdel furnace
```

Removing the package also removes the Furnace nginx site, its keepalived settings (the shared IP is released) and its firewall rules. The firewall itself stays on. `dnf remove` also removes packages that only Furnace needed (nginx, keepalived, maybe `openssh-server`): read dnf's list before you answer y, or keep one first with `sudo dnf mark install openssh-server`. SELinux: `httpd_can_network_connect` stays on; turn it off with `sudo setsebool -P httpd_can_network_connect off` if nothing else needs it.

## Troubleshooting on Amazon Linux 2023

| Problem | What to check |
|---|---|
| Both hosts say leader / both have the shared IP | VRRP (protocol 112) is blocked between the hosts. Check the network and both host firewalls. `sudo journalctl -u keepalived --since '10 min ago'` on both hosts: *Receive advertisement timeout* and *Entering MASTER STATE* on both = no VRRP arrives. |
| The shared IP never comes up | On VMs: MAC address changes / forged transmits / MAC spoofing (Part 0). Is the VRRP router ID used by another device? |
| Replication: failed | Port 443 between the hosts, nginx on the other host, the same certificate on both. |
| Permission denied in the logs (SELinux) | `sudo ausearch -m avc -ts recent` shows what SELinux blocked. Run `sudo furnace setup` again: it sets the labels again. |
| Port 443 works locally but not from outside | `sudo firewall-cmd --list-all`: is the rule in the **default** zone? `firewall-cmd --get-active-zones` shows which zone the card uses. |

Logs: `/var/log/furnace-setup.log`, `journalctl -u furnace-api`, `journalctl -u keepalived`, `/var/log/nginx/furnace-access.log`.
