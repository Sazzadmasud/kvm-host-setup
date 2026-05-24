# KVM Host Setup for OCP 4.18 (Agent-based)

Ansible playbooks to configure a bare-metal KVM host and deploy OpenShift
Container Platform 4.18 using the agent-based installation method.

---

## Table of Contents

1. [Host Specifications](#host-specifications)
2. [Cluster Design](#cluster-design)
3. [Architecture Overview](#architecture-overview)
4. [Part 1 — Prerequisites](#part-1--prerequisites)
   - [1.1 Host Hardware Verification](#11-host-hardware-verification)
   - [1.2 KVM / libvirt Stack](#12-kvm--libvirt-stack)
   - [1.3 Swap — Why 32 GB and Where](#13-swap--why-32-gb-and-where)
   - [1.4 DNS — BIND9 (not libvirt dnsmasq)](#14-dns--bind9-not-libvirt-dnsmasq)
   - [1.5 Load Balancer — HAProxy with Persistent VIPs](#15-load-balancer--haproxy-with-persistent-vips)
   - [1.6 NetworkManager DNS Forwarding](#16-networkmanager-dns-forwarding)
   - [1.7 Libvirt Network (DHCP only, DNS off)](#17-libvirt-network-dhcp-only-dns-off)
5. [Part 2 — OCP Deployment](#part-2--ocp-deployment)
   - [2.1 Wipe Everything](#21-wipe-everything)
   - [2.2 Run Full Host Setup](#22-run-full-host-setup)
   - [2.3 Add Your Pull Secret](#23-add-your-pull-secret)
   - [2.4 Generate the Agent ISO](#24-generate-the-agent-iso)
   - [2.5 Start VMs](#25-start-vms)
   - [2.6 Monitor Bootstrap + Eject ISO](#26-monitor-bootstrap--eject-iso)
   - [2.7 Wait for Install Complete](#27-wait-for-install-complete)
   - [2.8 Access the Cluster](#28-access-the-cluster)
6. [Customisation](#customisation)
7. [Project Structure](#project-structure)
8. [Troubleshooting](#troubleshooting)

---

## Host Specifications

| Property | Value |
|---|---|
| OS | Fedora Linux 42 |
| Kernel | 6.19.8-100.fc42.x86_64 |
| CPU | Intel Core i7-10710U — 6 cores / 12 threads, VT-x |
| RAM | 62 GB |
| Root disk | NVMe 29 GB (btrfs, `/` + `/home`) |
| Data disk | `/dev/sda1` ext4 733 GB → `/opt` |
| KVM stack | qemu-kvm, libvirt (modular daemons) |

---

## Cluster Design

### Topology: 3 control plane + 2 workers

| Node | Role | vCPU | RAM | Disk | MAC | IP |
|---|---|---|---|---|---|---|
| ocp-control-1 | master | 4 | 16 GB | 120 GB | `52:54:00:c0:01:01` | 192.168.200.21 |
| ocp-control-2 | master | 4 | 16 GB | 120 GB | `52:54:00:c0:01:02` | 192.168.200.22 |
| ocp-control-3 | master | 4 | 16 GB | 120 GB | `52:54:00:c0:01:03` | 192.168.200.23 |
| ocp-worker-1  | worker | 4 |  8 GB | 120 GB | `52:54:00:c0:02:01` | 192.168.200.31 |
| ocp-worker-2  | worker | 4 |  8 GB | 120 GB | `52:54:00:c0:02:02` | 192.168.200.32 |

**Total VM allocation:** 3×16 + 2×8 = 64 GB on a 62 GB host. This is a
deliberate overcommit — Linux's memory overcommit + zram + the 32 GB swapfile
on `/opt` make it work in practice for a lab. The VMs don't all actively use
their full allocation simultaneously during stable operation.

### Network

| Purpose | Value |
|---|---|
| Network name | `labnet` |
| Bridge | `virbr4` |
| Mode | NAT |
| Subnet | 192.168.200.0/24 |
| Gateway | 192.168.200.1 |
| DHCP range | 192.168.200.100–200 (nodes use static IPs, range for ad-hoc VMs) |
| API VIP | 192.168.200.10 → `api.ocp.lab.local`, `api-int.ocp.lab.local` |
| Ingress VIP | 192.168.200.11 → `*.apps.ocp.lab.local` |
| Cluster domain | `ocp.lab.local` |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│  KVM Host (Fedora 42, 62 GB RAM)                                │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  virbr4  192.168.200.1/24                               │   │
│  │          192.168.200.10/32  (API VIP)                   │   │
│  │          192.168.200.11/32  (Ingress VIP)               │   │
│  │                                                         │   │
│  │  BIND9  :53   (for VMs)    :5353  (for host NM fwd)     │   │
│  │  HAProxy :6443 :22623 :80 :443  (bound to VIPs)         │   │
│  │  dnsmasq :67  DHCP only — DNS disabled                  │   │
│  └─────────────────────────────────────────────────────────┘   │
│          │ vnet1   │ vnet3   │ vnet4   │ vnet5   │ vnet6        │
│          ▼         ▼         ▼         ▼         ▼              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────┐ ┌──────┐     │
│  │control-1 │ │control-2 │ │control-3 │ │wkr-1 │ │wkr-2 │     │
│  │.21       │ │.22       │ │.23       │ │.31   │ │.32   │     │
│  └──────────┘ └──────────┘ └──────────┘ └──────┘ └──────┘     │
│                                                                 │
│  Host DNS query path:                                           │
│    app → NM dnsmasq (127.0.0.1:53)                              │
│         → ocp.lab.local? forward to BIND (127.0.0.1:5353)      │
│         → other? forward to internet                            │
│                                                                 │
│  VM DNS query path:                                             │
│    VM → BIND (192.168.200.1:53) — direct, no proxy             │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 1 — Prerequisites

This section explains **what the Ansible roles configure and why** before any
OCP deployment begins. Running `ansible-playbook site.yml` (excluding `--tags
vms`) sets all of this up automatically.

---

### 1.1 Host Hardware Verification

**Role:** `pre_tasks` in `site.yml`

Before anything runs, the playbook asserts:

- **KVM support**: `/proc/cpuinfo` must contain `vmx` (Intel VT-x) or `svm`
  (AMD-V). Without hardware virtualisation OCP nodes cannot boot.
- **Minimum RAM**: 48 GB. The cluster needs 64 GB allocated across 5 VMs;
  the host needs to support that with overcommit.
- **`/opt` space**: Minimum 500 GB free. VM disk images (5 × 120 GB = 600 GB
  nominal, ~10–20 GB actual on thin-provisioned qcow2), the agent ISO (~1.4 GB),
  the 32 GB swapfile, and install artifacts all live here.

---

### 1.2 KVM / libvirt Stack

**Role:** `packages`, `libvirt_setup`

**What is installed:**

```
qemu-kvm  qemu-img  libvirt  libvirt-client
libvirt-daemon-driver-{qemu,network,storage-core,storage-logical}
virt-install  python3-libvirt  python3-lxml
bind  bind-utils  dnsmasq  haproxy  nmstate
openshift-install  oc  kubectl
```

**Fedora 42 uses modular libvirt daemons.** Fedora 38+ replaced the
single `libvirtd` process with socket-activated micro-daemons. The sockets
that must be enabled are:

| Socket | Purpose |
|---|---|
| `virtqemud.socket` | QEMU hypervisor driver |
| `virtnetworkd.socket` | Network (bridge, NAT, DNS/DHCP) |
| `virtstoraged.socket` | Storage pool management |
| `virtnodedevd.socket` | Host device enumeration |
| `virtsecretd.socket` | Secrets (used by LUKS) |
| `virtlogd.socket` | Console log forwarding |

If only `libvirtd.socket` is enabled (the old monolithic path), `virsh
-c qemu:///system` may work but `virtnetworkd` won't be managing networks,
causing stale PID files and bridge conflicts on cleanup. See [T4](#t4).

**QEMU configuration changes** (`/etc/libvirt/qemu.conf`):

| Setting | Value | Reason |
|---|---|---|
| `security_driver` | `none` | Disables sVirt/SELinux labelling — simplifies lab setup |
| `user` | `root` | Needed to read qcow2 images in `/opt/images` |
| `group` | `kvm` | Shared access to KVM device nodes |
| `dynamic_ownership` | `1` | QEMU can chown disk images at runtime |

---

### 1.3 Swap — Why 32 GB and Where

**Role:** `swap`

**The problem:** The 5 VMs collectively allocate 64 GB of RAM on a 62 GB
host. Without sufficient swap, any mmap that the kernel cannot back with
physical pages causes a `SIGSEGV` (exit 139) inside the guest process — not an
OOM kill, just a silent crash. This was observed as `openshift-apiserver`
crashing on control-3 with exit code 139 during the initial install attempt.

**The solution:** A 32 GB swapfile on `/opt` (which has 529 GB free on the
data SSD). The host also has an 8 GB zram swap device (priority 100, used
first due to compression). Combined, the host has ~40 GB of swap, enough
headroom for the overcommit.

**What the role does:**

1. Creates `/opt/swapfile` (32 GB, `dd` if it doesn't exist)
2. Formats it as swap (`mkswap`)
3. Activates it (`swapon`)
4. Adds it to `/etc/fstab` for persistence across reboots:
   ```
   /opt/swapfile swap swap defaults 0 0
   ```

**Why `/opt` and not `/`?** The root NVMe partition is only 29 GB (btrfs).
`/opt` is on the 733 GB data SSD with over 500 GB free.

---

### 1.4 DNS — BIND9 (not libvirt dnsmasq)

**Role:** `dns`, `nm_dns`

This is the most important prerequisite to get right. DNS problems cause
cascading failures in OCP that are very difficult to diagnose.

#### Why not libvirt's built-in dnsmasq?

libvirt's dnsmasq is sufficient for basic A record resolution. The problems
appear when OCP is running:

**Problem: AAAA query loop causing timeouts**

OCP pods run with `ndots:5` in their `/etc/resolv.conf`. This makes the Go
HTTP client (used by almost every OCP operator) issue both A and AAAA queries
for every hostname. When AAAA queries hit libvirt's dnsmasq, the following
loop forms:

```
Pod DNS query: oauth-openshift.apps.ocp.lab.local AAAA
  → CoreDNS (on node) → forwards to node resolver (192.168.200.1)
  → libvirt dnsmasq → doesn't have AAAA, forwards upstream → 127.0.0.1
  → NM dnsmasq → server=/ocp.lab.local/192.168.200.1 → back to libvirt dnsmasq
  → loop → timeout after 5 seconds
```

The `filter-AAAA` dnsmasq option was tried but it only suppresses forwarding
to the *default* upstream resolver. It does **not** suppress forwarding to
domain-specific `server=` entries — so the loop persists for all
`*.ocp.lab.local` names.

`local=/ocp.lab.local/` (mark the domain as locally authoritative, never
forward) was also tried. This requires a full daemon restart to take effect
(SIGHUP only re-reads hosts files, not the main config), and restarting the
libvirt dnsmasq while VMs are running is disruptive.

**The fix: BIND9**

BIND9 is authoritative for `ocp.lab.local`. It returns `NXDOMAIN` instantly
for unknown names (no forwarding to any upstream), and returns `NOERROR` with
empty answer section for AAAA queries against A-only names. No loops, no
timeouts.

#### What BIND9 serves

Zone: `ocp.lab.local`

| Name | Type | Value |
|---|---|---|
| `api.ocp.lab.local` | A | 192.168.200.10 (API VIP) |
| `api-int.ocp.lab.local` | A | 192.168.200.10 (same VIP, internal alias) |
| `*.apps.ocp.lab.local` | A | 192.168.200.11 (Ingress VIP, wildcard) |
| `ocp-control-{1,2,3}.ocp.lab.local` | A | 192.168.200.{21,22,23} |
| `ocp-worker-{1,2}.ocp.lab.local` | A | 192.168.200.{31,32} |

**`api-int` is critical.** During first boot, RHCOS fetches its ignition
config from `https://api-int.ocp.lab.local:22623/config/master`. If this name
doesn't resolve, ignition fails silently — the node boots with no SSH keys,
no kubelet, and can never join the cluster. See [T11](#t11).

#### How BIND9 listens

BIND9 listens on two addresses and two ports:

| Address:Port | Purpose |
|---|---|
| `192.168.200.1:53` | Direct DNS for all VMs (DHCP tells VMs to use this) |
| `127.0.0.1:5353` | For NM dnsmasq to forward `*.ocp.lab.local` from the host |

`127.0.0.1:53` is **not** used by BIND9 — that port belongs to NM's own
dnsmasq process.

#### Startup order

BIND9 must start *after* `virbr4` exists (otherwise it cannot bind to
`192.168.200.1:53`). The `dns` role adds a systemd drop-in:

```ini
# /etc/systemd/system/named.service.d/after-libvirt.conf
[Unit]
After=libvirtd.service

[Service]
Restart=on-failure
RestartSec=5
```

The libvirt network hook (see §1.5) also calls `systemctl restart named`
after binding VIPs, as a belt-and-suspenders measure.

---

### 1.5 Load Balancer — HAProxy with Persistent VIPs

**Role:** `haproxy`, `vip`

#### What HAProxy does

HAProxy load-balances four services:

| Frontend | VIP:Port | Backend |
|---|---|---|
| API server | 192.168.200.10:6443 | control-{1,2,3}:6443 |
| Machine Config | 192.168.200.10:22623 | control-{1,2,3}:22623 |
| Ingress HTTP | 192.168.200.11:80 | worker-{1,2}:80 |
| Ingress HTTPS | 192.168.200.11:443 | worker-{1,2}:443 |

HAProxy runs on the KVM host (not in a VM). It binds to the specific VIP
addresses, not `*:port`. This prevents port conflicts with other host services
and makes the binding intent explicit.

#### Why VIPs must be explicitly assigned to virbr4

A VM sending a packet to `192.168.200.10` does a broadcast ARP request:
*"Who has 192.168.200.10?"* If no interface on the host has that address, the
ARP goes unanswered, the VM never gets a MAC address for the gateway, and the
TCP connection never establishes — even though HAProxy is listening.

The VIPs must be secondary addresses on `virbr4` so the host responds to ARP
and HAProxy receives the traffic.

#### How VIPs are made persistent (libvirt network hook)

Previous attempts used:
- Manual `ip addr add` — lost on reboot
- NetworkManager dispatcher script — does not fire for `virbr*` interfaces
  (NM explicitly ignores them via the `99-libvirt.conf` unmanaged-devices rule)

The correct mechanism is a **libvirt network hook**:
`/etc/libvirt/hooks/network`

Libvirt calls this script every time any network changes state. The hook binds
the VIPs when `labnet` starts and removes them when it stops:

```bash
case "$NETWORK/$ACTION" in
    labnet/started)
        ip addr add 192.168.200.10/32 dev virbr4
        ip addr add 192.168.200.11/32 dev virbr4
        sleep 1
        systemctl restart named   # BIND can now bind to 192.168.200.1:53
        ;;
    labnet/stopped)
        ip addr del 192.168.200.10/32 dev virbr4
        ip addr del 192.168.200.11/32 dev virbr4
        ;;
esac
```

This runs automatically on every boot when libvirtd autostart brings up
`labnet`. No manual intervention needed.

#### HAProxy startup order

HAProxy binds to the VIPs. The VIPs only exist after the libvirt hook runs.
The `haproxy` role adds a systemd drop-in:

```ini
# /etc/systemd/system/haproxy.service.d/after-libvirt.conf
[Unit]
After=libvirtd.service

[Service]
Restart=on-failure
RestartSec=5
```

If HAProxy starts before the hook finishes, it restarts automatically and
succeeds on the next attempt.

---

### 1.6 NetworkManager DNS Forwarding

**Role:** `nm_dns`

The KVM host uses NetworkManager with dnsmasq mode for its own DNS resolution
(`/etc/NetworkManager/conf.d/01-dnsmasq.conf`). NM dnsmasq runs on
`127.0.0.1:53` and handles all host DNS queries.

For OCP names, NM dnsmasq needs to forward to BIND9. The config
`/etc/NetworkManager/dnsmasq.d/ocp.conf` contains:

```
server=/ocp.lab.local/127.0.0.1#5353
```

This forwards all `*.ocp.lab.local` queries (including `*.apps.ocp.lab.local`
which is a subdomain) to BIND9 on port 5353. BIND9 answers authoritatively —
no further forwarding, no loops.

External names (internet) are forwarded by NM dnsmasq to the upstream
resolver from the active network connection (wifi in this case).

---

### 1.7 Libvirt Network (DHCP only, DNS off)

**Role:** `networks`

The `labnet` network definition has DNS **disabled**:

```xml
<dns enable='no'/>
```

Without this, libvirt's dnsmasq would bind to `192.168.200.1:53` and prevent
BIND9 from binding there. With DNS disabled, dnsmasq only handles DHCP.

VMs are told to use BIND9 for DNS via a DHCP option:

```xml
<dnsmasq:options>
  <dnsmasq:option value='dhcp-option=option:dns-server,192.168.200.1'/>
</dnsmasq:options>
```

This injects `option 6` (DNS server) into every DHCP offer, overriding the
default (which would be the gateway itself acting as DNS). VMs receive their
IP from DHCP but query BIND9 directly for all DNS.

Note: OCP node IPs are static (configured via NMState in `agent-config.yaml`),
not from DHCP. The DHCP option is still sent in the DISCOVER/OFFER exchange;
NMState picks it up and writes it to the node's `resolv.conf`.

---

## Part 2 — OCP Deployment

With all prerequisites in place (either from a fresh `site.yml` run or
verified manually), follow these steps in order.

---

### 2.1 Wipe Everything

If there is a previous install (VMs, networks, images) that needs to be
removed first:

```bash
cd /opt/kvm-host-setup
ansible-playbook cleanup.yml
```

This destroys and undefines all VMs (with `--nvram --remove-all-storage`),
destroys and undefines all libvirt networks and storage pools, kills any stale
dnsmasq processes left by the old monolithic libvirtd, removes stale bridge
and tap interfaces, wipes `/opt/images`, `/opt/ocp`, and `/opt/ocp-build`,
vacuums the systemd journal to 200 MB, and cleans the dnf cache.

**Warning:** This removes everything including all VM disk data. There is no
confirmation prompt — run it only when you mean it.

---

### 2.2 Run Full Host Setup

```bash
ansible-playbook site.yml --skip-tags vms
```

This configures the entire host in order:

1. **packages** — installs KVM stack, BIND, HAProxy, nmstate, openshift-install, oc
2. **swap** — creates and activates the 32 GB swapfile, adds to fstab
3. **libvirt_setup** — enables modular daemons, configures qemu.conf
4. **networks** — defines `labnet` (NAT, DNS off, DHCP with DNS option)
5. **dns** — deploys named.conf + zone file, opens libvirt firewalld zone for DNS
6. **nm_dns** — writes NM dnsmasq forwarding config for `ocp.lab.local`
7. **vip** — deploys the libvirt network hook, binds VIPs immediately
8. **haproxy** — deploys haproxy.cfg, opens firewalld ports, enables service
9. **storage_pool** — creates `/opt/images` storage pool in libvirt

Individual roles can be re-run with tags:

```bash
ansible-playbook site.yml --tags dns          # re-deploy BIND config only
ansible-playbook site.yml --tags haproxy      # re-deploy HAProxy config only
ansible-playbook site.yml --tags networks     # re-define labnet
```

**Verify the prerequisites before continuing:**

```bash
# BIND is running and serving all required names
dig api.ocp.lab.local A +short           # → 192.168.200.10
dig api-int.ocp.lab.local A +short       # → 192.168.200.10
dig ocp-control-1.ocp.lab.local A +short # → 192.168.200.21
dig anything.apps.ocp.lab.local A +short # → 192.168.200.11

# AAAA queries return immediately (no timeout)
dig api.ocp.lab.local AAAA +time=2       # → NOERROR, empty answer
dig foo.ocp.lab.local AAAA +time=2       # → NXDOMAIN

# VIPs are on virbr4
ip addr show virbr4 | grep '192.168.200.1[01]'

# HAProxy is running and config is valid
haproxy -c -f /etc/haproxy/haproxy.cfg && echo "config OK"
systemctl is-active haproxy

# Swap is active and in fstab
swapon --show
grep swapfile /etc/fstab
```

---

### 2.3 Add Your Pull Secret

Download your pull secret from
[console.redhat.com/openshift/install/pull-secret](https://console.redhat.com/openshift/install/pull-secret)
and copy it to the expected location:

```bash
cp ~/pull-secret.json /opt/ocp/pull-secret.json
chmod 600 /opt/ocp/pull-secret.json

# Verify it is valid JSON with the correct keys
python3 -c "
import json
ps = json.load(open('/opt/ocp/pull-secret.json'))
print('Keys:', list(ps['auths'].keys()))
"
```

Expected output:
```
Keys: ['cloud.openshift.com', 'quay.io', 'registry.connect.redhat.com', 'registry.redhat.io']
```

---

### 2.4 Generate the Agent ISO

This renders `install-config.yaml` and `agent-config.yaml` from Jinja2
templates (using variables from `group_vars/all.yml`), then calls
`openshift-install agent create image` to produce the boot ISO.

```bash
ansible-playbook generate-ocp-config.yml
```

The ISO is written to `/opt/ocp/cluster/agent.x86_64.iso` (~1.4 GB).
`openshift-install` also writes a `auth/` directory with the kubeconfig and
kubeadmin password.

To override variables without editing `all.yml`:
```bash
ansible-playbook generate-ocp-config.yml \
  -e "cluster_name=mycluster base_domain=example.com"
```

**What the ISO contains:**

- A minimal RHCOS (Red Hat CoreOS) live image
- Embedded `install-config.yaml` and `agent-config.yaml`
- NMState network configuration (static IPs, DNS, routes) for every node
- Ignition config pre-staged for first-boot

All 5 VMs boot the **same** ISO. The agent-service on each node reads the
embedded config, matches the node's MAC address to determine its role and IP,
and configures itself accordingly. The first node to boot (`ocp-control-1`
at `192.168.200.21`, the `rendezvousIP`) hosts the bootstrap cluster until
etcd quorum is established.

---

### 2.5 Start VMs

```bash
ansible-playbook site.yml --tags vms
```

The `vms` role creates 120 GB thin-provisioned qcow2 disk images in
`/opt/images/`, defines VM XML with the correct MAC addresses and ISO attached
as boot device, and starts all 5 VMs.

VM disk performance note: the XML template uses `cache='unsafe' io='threads'`
for the disk. This makes guest `fdatasync` calls a no-op on the host, which is
required for the RHCOS `installation-disk-speed-check` to pass. The fio test
that OCP runs (`fio --rw=write --ioengine=sync --fdatasync=1`) fails with
`cache='none'` (too slow) and `cache='writeback'` (passes first round, fails
subsequent rounds as HDD contention builds). See [T8](#t8).

---

### 2.6 Monitor Bootstrap + Eject ISO

Run this **immediately after starting the VMs**, in a separate terminal:

```bash
ansible-playbook eject-iso.yml
```

This playbook does two things in parallel:

1. **Starts `openshift-install agent wait-for bootstrap-complete`** — so the
   install log is streaming and the playbook exits cleanly when bootstrap is
   reached.

2. **Monitors each VM and ejects the agent ISO** the moment RHCOS finishes
   writing itself to disk. Detection uses two parallel signals (first one wins
   per node):
   - The install log matches `Host: <name>.*Writing image.*100` or
     `Host: <name>.*Rebooting`
   - The virsh domain state transitions to `shut-off` (post-write reboot)

**Why ejecting is necessary:** After RHCOS writes to disk, the node reboots.
On UEFI KVM VMs, the boot order in XML is not reliably honoured — the VM can
boot from the ISO again instead of the disk. This causes:
```
Expected the host to boot from disk, but it booted the installation image
```
…and the install stalls. Ejecting the ISO before the reboot forces disk boot.

If a node does accidentally reboot into the ISO before ejection, recover
manually:
```bash
sudo virsh -c qemu:///system change-media ocp-control-1 sda --eject --live --config
sudo virsh -c qemu:///system reboot ocp-control-1
```

Bootstrap completes in approximately 15–20 minutes after VM start.

---

### 2.7 Wait for Install Complete

After bootstrap-complete is logged, run:

```bash
openshift-install --dir=/opt/ocp/cluster agent wait-for install-complete \
  --log-level=info
```

This waits for all cluster operators to become available and the cluster
version to report `Completed`. Total time from VM start is approximately
45–60 minutes.

While waiting, you can watch operator status:
```bash
export KUBECONFIG=/opt/ocp/cluster/auth/kubeconfig
watch -n10 "oc get co | grep -v 'True.*False.*False'"
```

A healthy cluster has all operators `Available=True`, `Progressing=False`,
`Degraded=False`.

---

### 2.8 Access the Cluster

```bash
export KUBECONFIG=/opt/ocp/cluster/auth/kubeconfig

oc get nodes          # all 5 nodes, Ready
oc get co             # all operators available

# kubeadmin password
cat /opt/ocp/cluster/auth/kubeadmin-password
```

Web console: `https://console-openshift-console.apps.ocp.lab.local`

Add to your workstation's `/etc/hosts` if accessing from outside the KVM host:
```
192.168.200.10  api.ocp.lab.local api-int.ocp.lab.local
192.168.200.11  console-openshift-console.apps.ocp.lab.local oauth-openshift.apps.ocp.lab.local
```

---

## Customisation

All tunable values live in [group_vars/all.yml](group_vars/all.yml):

| Variable | Default | Description |
|---|---|---|
| `cluster_name` | `ocp` | OCP cluster name |
| `base_domain` | `lab.local` | Base DNS domain |
| `ocp_version` | `4.18` | Used for client binary downloads |
| `api_vip` | `192.168.200.10` | API load-balancer VIP |
| `ingress_vip` | `192.168.200.11` | Ingress/router VIP |
| `ocp_network_cidr` | `192.168.200.0/24` | Machine network |
| `ocp_network_gateway` | `192.168.200.1` | Bridge IP, DHCP server, BIND9 address |
| `control_vm_vcpus` | `4` | vCPUs per control node |
| `control_vm_ram_mb` | `16384` | RAM per control node (MB) — OCP minimum is 16384 |
| `control_vm_disk_gb` | `120` | Disk per control node |
| `worker_vm_vcpus` | `4` | vCPUs per worker |
| `worker_vm_ram_mb` | `8192` | RAM per worker (MB) |
| `worker_vm_disk_gb` | `120` | Disk per worker |
| `libvirt_storage_pool_path` | `/opt/images` | Where qcow2 images are stored |
| `ocp_install_dir` | `/opt/ocp/cluster` | openshift-install working directory |

---

## Project Structure

```
kvm-host-setup/
├── ansible.cfg                          # Inventory path, become, callbacks
├── inventory/hosts.yml                  # kvm_host → localhost
├── group_vars/all.yml                   # All variables — edit this file
│
├── roles/
│   ├── cleanup/tasks/main.yml           # Destroy VMs, networks, pools; wipe dirs
│   ├── packages/tasks/main.yml          # dnf + openshift-install + oc downloads
│   ├── swap/tasks/main.yml              # 32G swapfile on /opt, fstab entry
│   ├── libvirt_setup/tasks/main.yml     # Modular daemons, qemu.conf
│   ├── networks/tasks/main.yml          # labnet: NAT bridge, DNS off, DHCP
│   ├── dns/
│   │   ├── tasks/main.yml               # BIND9 deploy + named systemd override
│   │   ├── templates/named.conf.j2      # BIND options + zone declaration
│   │   └── templates/ocp.lab.local.zone.j2  # Zone file: VIPs, nodes, wildcard
│   ├── nm_dns/
│   │   ├── tasks/main.yml               # NM dnsmasq forwarding config
│   │   └── templates/ocp.conf.j2        # server=/ocp.lab.local/127.0.0.1#5353
│   ├── vip/
│   │   ├── tasks/main.yml               # Deploy libvirt hook, bind VIPs now
│   │   └── templates/network.hook.j2    # Hook: bind/unbind VIPs + restart named
│   ├── haproxy/
│   │   ├── tasks/main.yml               # HAProxy systemd override + config
│   │   └── templates/haproxy.cfg.j2     # VIP-bound frontends for API + ingress
│   ├── storage_pool/tasks/main.yml      # /opt/images pool in libvirt
│   └── vms/
│       ├── tasks/main.yml               # Disk images + VM XML + start
│       └── templates/
│           ├── vm.xml.j2                # VM definition (q35, UEFI, cache=unsafe)
│           └── eject-monitor.sh.j2      # ISO eject monitor script
│
├── ocp-config/
│   ├── install-config.yaml.j2           # Cluster config (network, replicas)
│   └── agent-config.yaml.j2             # Per-node NMState static IP + DNS
│
├── site.yml                             # Full host setup
├── cleanup.yml                          # Wipe-only playbook
├── generate-ocp-config.yml             # Render configs + run openshift-install
└── eject-iso.yml                        # Monitor image-write + eject + bootstrap-complete
```

---

## Troubleshooting

### [T1] `Invalid callback for stdout specified: yaml`

**Symptom:**
```
ERROR! Invalid callback for stdout specified: yaml
```

**Cause:** `ansible-core` on Fedora 42 does not ship the `yaml` stdout
callback. Only `ansible.builtin.*` callbacks are available without extra
collections.

**Fix:** Use the built-in `default` callback in `ansible.cfg`:
```ini
[defaults]
stdout_callback   = ansible.builtin.default
callbacks_enabled = ansible.posix.profile_tasks
```

---

### [T2] `sudo: a password is required` during fact gathering

**Symptom:**
```
fatal: [kvm_host]: FAILED! => {"msg": "MODULE FAILURE ... sudo: a password is required"}
```

**Fix:** Add `become_ask_pass = True` to `ansible.cfg` or pass `-K`:
```bash
ansible-playbook site.yml -K
```

---

### [T3] DNS resolution broken after cleanup — `Could not resolve host`

**Symptom:** DNF fails with `Could not resolve hostname mirrors.fedoraproject.org`.

**Cause:** Stale configurations from a previous lab deployment:
1. `/etc/systemd/resolved.conf` hardcoded to a dead DNS server
2. A NetworkManager connection profile for a decommissioned bridge had a
   stale `ipv4.dns` pointing to a non-existent gateway

**Fix:**
```bash
# Reset systemd-resolved
sudo tee /etc/systemd/resolved.conf <<'EOF'
[Resolve]
DNS=8.8.8.8 1.1.1.1
FallbackDNS=8.8.4.4 9.9.9.9
DNSStubListener=yes
EOF

# Remove stale NM connection profiles
sudo nmcli connection show   # look for dead virbr/provisioning connections
sudo nmcli connection delete <stale-connection>

sudo systemctl restart systemd-resolved NetworkManager
```

---

### [T4] `dnsmasq: failed to create listening socket — Address already in use`

**Symptom:** `networks` role fails starting `labnet` with address conflict on `192.168.200.1:53`.

**Cause 1 — Stale dnsmasq from old monolithic libvirtd:**
```bash
ls /var/run/libvirt/network/   # stale *.pid files
```
Fix:
```bash
sudo kill $(cat /var/run/libvirt/network/*.pid 2>/dev/null) 2>/dev/null
sudo rm -rf /var/run/libvirt/network/* /var/lib/libvirt/dnsmasq/virbr*.status
```

**Cause 2 — BIND9 running and grabbing all ports:**
```bash
sudo ss -tulnp | grep ':53'   # named listening on 192.168.200.1:53
```
Fix: BIND9 should only bind after virbr4 exists (the network hook restarts it).
If named started before virbr4, it grabbed `0.0.0.0:53`. Stop and restart:
```bash
sudo systemctl restart named
```

---

### [T5] `exec: "nmstatectl": executable file not found`

**Symptom:** `generate-ocp-config.yml` fails at ISO generation:
```
failed to validate network yaml ... exec: "nmstatectl": executable file not found
```

**Fix:** Install the `nmstate` package:
```bash
sudo dnf install -y nmstate
ansible-playbook generate-ocp-config.yml
```

The `packages` role includes `nmstate` so a full `site.yml` run installs it.

---

### [T6] `unknown field 'macAddress'` in NMState validation

**Symptom:**
```
interfaces: unknown field `macAddress` at line 6 column 1
```

**Cause:** NMState 2.x requires `mac-address` (hyphenated) inside
`networkConfig.interfaces`. The camelCase `macAddress` was only accepted by
nmstate 1.x.

Note the distinction in `agent-config.yaml`:
- `hosts[].interfaces[].macAddress` — OCP AgentConfig API field, stays camelCase
- `hosts[].networkConfig.interfaces[].mac-address` — NMState 2.x field, use hyphen

The template `ocp-config/agent-config.yaml.j2` uses the correct form.

---

### [T7] `installation-disk-speed-check` timeout (exit code 124)

**Symptom:** All nodes stuck at `preparing-for-installation` with:
```
exit-code <124>   # timeout
```

**Cause:** OCP runs `fio --rw=write --ioengine=sync --fdatasync=1` to verify
disk speed. With `cache='none'` (QEMU default), every guest `fdatasync` flushes
to the physical disk. On a new qcow2, each write also triggers cluster
allocation flushes. The combined overhead causes every fdatasync to take
100–200ms, giving <10 IOPS — far below OCP's 10 MB/s minimum.

**Fix:** The VM XML template uses `cache='unsafe' io='threads'`:
- Guest `fdatasync` becomes a no-op on the host
- fio completes in seconds into the host page cache
- Acceptable for a lab; do not use in production

---

### [T8] DNS validation warning: `Error while evaluating DNS resolution on this host`

**Symptom:** Nodes stuck at agent validation with persistent DNS warnings.

**What to check first:**
```bash
# From the host, verify api-int resolves
dig api-int.ocp.lab.local @192.168.200.1 +short   # must return 192.168.200.10

# Verify AAAA queries return immediately (no timeout)
timeout 2 dig api.ocp.lab.local AAAA @192.168.200.1 | grep status
```

**Cause:** Either `api-int.ocp.lab.local` is missing from BIND, or the AAAA
query is timing out (loop). In the current setup with BIND9, AAAA queries
return `NOERROR` (empty) or `NXDOMAIN` immediately — no timeout.

If `api-int` is missing, re-run the dns role:
```bash
ansible-playbook site.yml --tags dns
```

---

### [T9] Node booted back into installer after image write

**Symptom:**
```
Expected the host to boot from disk, but it booted the installation image
```

**Fix:** Eject the ISO and reboot the specific node:
```bash
sudo virsh -c qemu:///system change-media ocp-control-1 sda --eject --live --config
sudo virsh -c qemu:///system reboot ocp-control-1
```

To prevent this, always run `eject-iso.yml` in parallel with the install.

---

### [T10] Bootstrap-complete timeout — nodes never joined cluster

**Symptom:** `wait-for bootstrap-complete` times out. `oc get nodes` shows
only 1 node. Nodes 2–5 have no SSH access (Permission denied).

**Root cause:** If `api-int.ocp.lab.local` did not resolve at the moment
a node's RHCOS first-boot ran ignition, that node never fetched its ignition
config and booted without SSH keys, kubelet, or cluster membership. Fixing
DNS on a live cluster does not recover these nodes — ignition runs once, in
the initramfs, and does not retry.

**Recovery:** Full reinstall.
```bash
ansible-playbook cleanup.yml
ansible-playbook site.yml
ansible-playbook generate-ocp-config.yml
ansible-playbook site.yml --tags vms
ansible-playbook eject-iso.yml
```

**Prevention:** Always verify before starting VMs:
```bash
dig api-int.ocp.lab.local @192.168.200.1 +short   # MUST return 192.168.200.10
```

---

### [T11] `openshift-apiserver` pod SIGSEGV (exit code 139) on a control node

**Symptom:**
```
apiserver crashlooping container ... exit 139 (SIGSEGV)
```

**Cause:** Memory overcommit exhausted. The host has 62 GB RAM and 5 VMs
allocated 64 GB total. When zram swap is full and the 32 GB swapfile on
`/opt/swapfile` is not active (or not in fstab), the kernel cannot back
anonymous mmaps. Go runtime mmaps fail, causing SIGSEGV inside any Go binary.

**Fix:**
```bash
# Verify swap is active and correct
swapon --show
grep swapfile /etc/fstab

# If swapfile is missing from fstab
echo '/opt/swapfile swap swap defaults 0 0' | sudo tee -a /etc/fstab
sudo swapon /opt/swapfile
```

The `swap` role handles this automatically.

---

### Useful diagnostic commands

```bash
# VM console (Ctrl+] to exit)
virsh console ocp-control-1

# SSH into a node (core user, key from ~/.ssh/id_rsa)
ssh core@192.168.200.21

# Watch agent bootstrap logs on the rendezvous node
ssh core@192.168.200.21 journalctl -u agent.service -f

# Install log (on the KVM host)
tail -f /opt/ocp/cluster/.openshift_install.log

# Check all operator status
export KUBECONFIG=/opt/ocp/cluster/auth/kubeconfig
oc get co
oc get nodes
oc get pods -A | grep -v Running | grep -v Completed
```
