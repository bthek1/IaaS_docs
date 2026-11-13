# LXC and LXD — Complete Guide

**LXC (Linux Containers)** is a low-level containerization technology that uses Linux kernel features (namespaces + cgroups) to create isolated environments. Think of it as a lightweight VM — it runs a full OS userspace, not just a single process.

Author

Benedict Thekkel

## LXC vs LXD vs Docker

| Feature               | LXC       | LXD        | Docker      |
|-----------------------|-----------|------------|-------------|
| Level                 | Low-level | High-level | High-level  |
| Container type        | System    | System     | Application |
| Runs full OS          | Yes       | Yes        | No          |
| Init system (systemd) | Yes       | Yes        | No          |
| Persistent by default | Yes       | Yes        | No          |
| Image registry        | No        | Yes        | Yes         |
| Clustering            | No        | Yes        | Yes (Swarm) |
| Ease of use           | Hard      | Easy       | Easiest     |

**System containers** (LXC/LXD) behave like VMs — they boot an OS, run systemd, have multiple processes. **Application containers** (Docker) run a single process/app.

------------------------------------------------------------------------

## Installation

**LXD via snap (recommended):**

``` bash
sudo snap install lxd
sudo usermod -aG lxd $USER
newgrp lxd        # apply group without logout
lxd init          # interactive setup wizard
```

**LXD via apt (older, Ubuntu 20.04 and below):**

``` bash
sudo apt install lxd lxd-client
```

**LXC only (without LXD):**

``` bash
sudo apt install lxc
```

------------------------------------------------------------------------

## lxd init — What Each Option Means

    Would you like to use LXD clustering? → No (unless multi-node)
    Do you want to configure a new storage pool? → Yes
    Name of the new storage pool: default
    Name of the storage backend (dir, btrfs, lvm, zfs): dir  ← simplest
    Would you like to connect to a MAAS server? → No
    Would you like to create a new local network bridge? → Yes
    What should the new bridge be called? → lxdbr0
    What IPv4 address should be used? → auto
    Would you like LXD to be available over the network? → No (yes for remote)
    Would you like stale cached images to be updated automatically? → Yes

------------------------------------------------------------------------

## Core Concepts

### Instances

LXD manages two types: - **Containers** — LXC-based, shared kernel, very lightweight - **Virtual Machines** — full hardware virtualization, own kernel

### Images

Pre-built OS rootfs snapshots. LXD pulls from image servers. Key servers: - `ubuntu:` — Official Ubuntu images - `ubuntu-daily:` — Daily Ubuntu builds - `images:` — Community images (Alpine, Fedora, Debian, Arch, etc.)

### Profiles

Reusable config templates applied to containers (networking, devices, limits).

### Storage Pools

Where container filesystems live. Backends: `dir`, `btrfs`, `zfs`, `lvm`, `ceph`.

### Networks

LXD creates a bridge (`lxdbr0`) with NAT by default. Containers get IPs in `10.x.x.x`.

------------------------------------------------------------------------

## Essential Commands

### Container Lifecycle

``` bash
lxc launch ubuntu:22.04 mycontainer        # create + start
lxc start mycontainer
lxc stop mycontainer
lxc stop mycontainer --force               # kill immediately
lxc restart mycontainer
lxc delete mycontainer                     # must be stopped first
lxc delete mycontainer --force             # stop + delete
```

### Listing & Info

``` bash
lxc list                                   # all containers
lxc list --format=json                     # JSON output
lxc info mycontainer                       # detailed info
lxc config show mycontainer                # full config YAML
```

### Shell Access

``` bash
lxc exec mycontainer -- bash               # root shell
lxc exec mycontainer -- su - ubuntu        # as specific user
lxc exec mycontainer -- /bin/sh            # for minimal images
lxc console mycontainer                    # attach to console (for VMs)
```

### Running Commands

``` bash
lxc exec mycontainer -- apt update
lxc exec mycontainer -- systemctl status nginx
lxc exec mycontainer -- env VAR=value command
```

### File Transfer

``` bash
lxc file push localfile.txt mycontainer/root/
lxc file pull mycontainer/etc/nginx/nginx.conf ./
lxc file edit mycontainer/etc/hosts         # edit in place
lxc file push -r ./mydir mycontainer/root/  # recursive
```

------------------------------------------------------------------------

## Images

``` bash
lxc image list                             # locally cached images
lxc image list ubuntu:                     # browse Ubuntu images
lxc image list images: alpine              # find Alpine images
lxc image list images: | grep -i fedora
lxc image info ubuntu:22.04
lxc image delete <fingerprint>             # remove cached image
```

**Common image aliases:**

``` bash
ubuntu:22.04       # Ubuntu Jammy
ubuntu:20.04       # Ubuntu Focal
ubuntu:24.04       # Ubuntu Noble
images:alpine/3.19
images:debian/12
images:fedora/39
images:archlinux
images:centos/9-Stream
```

------------------------------------------------------------------------

## Configuration & Limits

### Resource Limits

``` bash
# CPU
lxc config set mycontainer limits.cpu 2
lxc config set mycontainer limits.cpu.allowance 50%

# Memory
lxc config set mycontainer limits.memory 512MB
lxc config set mycontainer limits.memory.swap false

# Disk
lxc config device set mycontainer root size 20GB
```

### Environment Variables

``` bash
lxc config set mycontainer environment.MY_VAR=hello
```

### View / Edit Full Config

``` bash
lxc config show mycontainer
lxc config edit mycontainer               # opens in $EDITOR
```

------------------------------------------------------------------------

## Networking

### Default Setup

- Bridge: `lxdbr0` on host
- Containers get DHCP IPs like `10.178.x.x`
- NAT out to internet by default
- Containers can reach each other on the bridge

### Get Container IP

``` bash
lxc list                                  # shows IP column
lxc info mycontainer | grep -i inet
```

### Port Forwarding (proxy device)

``` bash
# Forward host port 8080 → container port 80
lxc config device add mycontainer myport proxy \
  listen=tcp:0.0.0.0:8080 \
  connect=tcp:127.0.0.1:80
```

### Static IP

``` bash
lxc network attach lxdbr0 mycontainer eth0
lxc config device set mycontainer eth0 ipv4.address 10.178.0.100
```

### Direct bridge (macvlan — gets LAN IP)

``` bash
lxc config device add mycontainer eth0 nic \
  nictype=macvlan \
  parent=eth0
```

------------------------------------------------------------------------

## Storage

### Storage Pools

``` bash
lxc storage list
lxc storage info default
lxc storage create mypool btrfs           # create btrfs pool
lxc storage create zfspool zfs            # create zfs pool
```

### Volumes

``` bash
lxc storage volume list default
lxc storage volume create default myvolume
lxc storage volume attach default myvolume mycontainer /mnt/data
```

### Disk Devices

``` bash
# Mount a host directory into container
lxc config device add mycontainer shareddir disk \
  source=/home/user/data \
  path=/mnt/data
```

------------------------------------------------------------------------

## Snapshots & Backups

``` bash
# Snapshots
lxc snapshot mycontainer snap1
lxc snapshot mycontainer snap1 --stateful  # include RAM state
lxc restore mycontainer snap1
lxc delete mycontainer/snap1
lxc info mycontainer                       # lists snapshots

# Export / Import
lxc export mycontainer mycontainer.tar.gz
lxc import mycontainer.tar.gz
lxc publish mycontainer --alias myimage    # make image from container
```

------------------------------------------------------------------------

## Profiles

Profiles let you define reusable config applied to many containers.

``` bash
lxc profile list
lxc profile show default
lxc profile create webserver
lxc profile edit webserver                 # edit YAML
lxc profile apply mycontainer default,webserver   # apply multiple
lxc launch ubuntu:22.04 mycontainer --profile webserver
```

**Example profile YAML:**

``` yaml
config:
  limits.cpu: "2"
  limits.memory: 1GB
description: Web server profile
devices:
  eth0:
    name: eth0
    nictype: bridged
    parent: lxdbr0
    type: nic
  root:
    path: /
    pool: default
    type: disk
  http:
    type: proxy
    listen: tcp:0.0.0.0:80
    connect: tcp:127.0.0.1:80
name: webserver
```

------------------------------------------------------------------------

## Virtual Machines

LXD can also manage full VMs (requires `qemu`):

``` bash
lxc launch ubuntu:22.04 myvm --vm
lxc launch ubuntu:22.04 myvm --vm --config limits.memory=2GB
lxc console myvm                          # serial console
lxc console myvm --type=vga              # graphical console
```

Same `lxc` commands work — `exec`, `file`, `snapshot`, `config`, etc.

------------------------------------------------------------------------

## Remote & Clustering

### Remote Hosts

``` bash
lxc remote add myserver https://192.168.1.10:8443
lxc list myserver:
lxc launch ubuntu:22.04 myserver:mycontainer
lxc copy mycontainer myserver:mycontainer  # migrate
```

### Clustering

``` bash
# On first node
lxd init   # enable clustering, set name + address

# On additional nodes
lxd init   # join existing cluster
lxc cluster list
lxc cluster show node1
```

------------------------------------------------------------------------

## Security

### Privileged vs Unprivileged

- **Unprivileged** (default) — UIDs mapped to high host UIDs, safer
- **Privileged** — container root = host root, needed for some workloads

``` bash
lxc config set mycontainer security.privileged true
```

### AppArmor & Seccomp

LXD applies AppArmor profiles and seccomp filters by default. To disable:

``` bash
lxc config set mycontainer raw.lxc "lxc.apparmor.profile=unconfined"
```

### Nesting (containers inside containers)

``` bash
lxc config set mycontainer security.nesting true
```

------------------------------------------------------------------------

## Useful Patterns

### Run a service and expose it

``` bash
lxc launch ubuntu:22.04 webserver
lxc exec webserver -- apt install -y nginx
lxc exec webserver -- systemctl enable --now nginx
lxc config device add webserver http proxy \
  listen=tcp:0.0.0.0:80 connect=tcp:127.0.0.1:80
```

### Autostart on boot

``` bash
lxc config set mycontainer boot.autostart true
lxc config set mycontainer boot.autostart.priority 10
```

### Limit a container completely

``` bash
lxc config set mycontainer limits.cpu 1
lxc config set mycontainer limits.memory 256MB
lxc config set mycontainer limits.processes 100
```

------------------------------------------------------------------------

## Troubleshooting

``` bash
lxc list                                  # check state
lxc info mycontainer                      # events + config
lxc monitor                               # live event stream
journalctl -u snap.lxd.daemon            # LXD daemon logs
lxc query /1.0/containers                 # raw REST API
```

------------------------------------------------------------------------

The key mental model: **LXD containers are like fast VMs**, not like Docker containers. They boot full OS, run systemd, persist by default, and are designed to be long-lived environments rather than ephemeral single-process runners.

------------------------------------------------------------------------

Back to top
