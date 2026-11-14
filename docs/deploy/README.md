# Deployment guides (Furnace 0.11.0)

Furnace runs on **two hosts**: a leader (host A) and a follower (host B). A **shared IP** moves to the host that is
alive. The firewall that downloads the lists and the SOAR that calls the API always use the shared IP.

Every guide below is **complete**: host A from the first login to the join code, then host B, then the tests, with
every command, every setup answer and what you should see. Pick your system and the way to deploy, open that one
guide, and follow it from top to bottom.

## 1. Pick the way to deploy

| | Packages (.deb / .rpm) | Docker, keepalived in the container | Docker, keepalived on the host |
|---|---|---|---|
| What you install | one package per host | Docker + one container per host | Docker + one container + keepalived on each host |
| Settings | setup asks questions | one `.env` file, the same on both hosts | `.env` + a block in your `keepalived.conf` |
| Who moves the shared IP | keepalived on the host, set up by Furnace | keepalived inside the container | **your** keepalived on the host |
| Host firewall | setup opens the ports | you open the ports | you open the ports |
| Other keepalived on the host? | **no** (setup replaces `keepalived.conf`) | **no** (two keepalived fight) | **yes**: Furnace gets its own block |
| Best for | plain servers that only run Furnace | Docker hosts that only run Furnace | hosts that already run keepalived |

Not sure? Use **packages** on servers that only run Furnace.

## 2. Pick your system

| System | Packages | Docker, keepalived in the container | Docker, keepalived on the host |
|---|---|---|---|
| Ubuntu 22.04 | [guide](packages/ubuntu-22.04.md) | [guide](docker/ubuntu-22.04.md) | [guide](docker-host-keepalived/ubuntu-22.04.md) |
| Ubuntu 24.04 | [guide](packages/ubuntu-24.04.md) | [guide](docker/ubuntu-24.04.md) | [guide](docker-host-keepalived/ubuntu-24.04.md) |
| Debian 12 | [guide](packages/debian-12.md) | [guide](docker/debian-12.md) | [guide](docker-host-keepalived/debian-12.md) |
| Debian 13 | [guide](packages/debian-13.md) | [guide](docker/debian-13.md) | [guide](docker-host-keepalived/debian-13.md) |
| AlmaLinux 8 | [guide](packages/almalinux-8.md) | [guide](docker/almalinux-8.md) | [guide](docker-host-keepalived/almalinux-8.md) |
| AlmaLinux 9 | [guide](packages/almalinux-9.md) | [guide](docker/almalinux-9.md) | [guide](docker-host-keepalived/almalinux-9.md) |
| AlmaLinux 10 | [guide](packages/almalinux-10.md) | [guide](docker/almalinux-10.md) | [guide](docker-host-keepalived/almalinux-10.md) |
| Rocky Linux 9 | [guide](packages/rocky-9.md) | [guide](docker/rocky-9.md) | [guide](docker-host-keepalived/rocky-9.md) |
| Oracle Linux 9 | [guide](packages/oracle-9.md) | [guide](docker/oracle-9.md) | [guide](docker-host-keepalived/oracle-9.md) |
| Fedora 42 | [guide](packages/fedora-42.md) | [guide](docker/fedora-42.md) | [guide](docker-host-keepalived/fedora-42.md) |
| Amazon Linux 2023 | [guide](packages/amazon-2023.md) | [guide](docker/amazon-2023.md) | [guide](docker-host-keepalived/amazon-2023.md) |

Close systems use the guide of their twin:

- **RHEL 8, Rocky Linux 8, Oracle Linux 8**: the AlmaLinux 8 guides.
- **RHEL 9, CentOS Stream 9**: the AlmaLinux 9 guides.
- **RHEL 10, Rocky Linux 10, Oracle Linux 10, CentOS Stream 10**: the AlmaLinux 10 guides.
- **Fedora 41 and newer**: the Fedora 42 guides.

The packages are tested on all 11 systems above in every release. The Docker container is the same image everywhere;
the guides differ in how Docker, the host firewall, SELinux and keepalived are set up on each system.

## The example addresses

All guides use the same fake addresses from the documentation ranges. Change them to your own:

```
host A (leader)    192.0.2.11
host B (follower)  192.0.2.12
shared IP          192.0.2.10/24   (a free address in the same network)
network card       ens192
```
