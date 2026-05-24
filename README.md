# KVM Host Setup for OCP Agent-Based Install

Ansible playbooks to configure a bare-metal KVM host and deploy OpenShift
Container Platform using the agent-based installation method.

---

## Table of Contents

1. [Quick Start](#quick-start)
2. [Host Specifications](#host-specifications)
3. [Cluster Design](#cluster-design)
4. [Architecture Overview](#architecture-overview)
5. [Infrastructure Components](#infrastructure-components)
   - [KVM / libvirt Stack](#kvm--libvirt-stack)
   - [Swap (32 GB)](#swap-32-gb)
   - [DNS — BIND9](#dns--bind9)
   - [Load Balancer — HAProxy + VIPs](#load-balancer--haproxy--vips)
   - [NetworkManager DNS Forwarding](#networkmanager-dns-forwarding)
   - [Libvirt Network](#libvirt-network)
   - [Redfish Emulator — sushy](#redfish-emulator--sushy)
   - [System Sleep Prevention](#system-sleep-prevention)
6. [OCP Deployment](#ocp-deployment)
   - [Unattended (deploy.yml)](#unattended-deployyml)
   - [Manual Step-by-Step](#manual-step-by-step)
7. [Post-Install Verification](#post-install-verification)
8. [Customisation](#customisation)
9. [Project Structure](#project-structure)
10. [Troubleshooting](#troubleshooting)

---

## Quick Start

**Prerequisites (one-time):**

```bash
# 1. Passwordless sudo
echo "$USER ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/$USER-nopasswd

# 2. Pull secret from https://console.redhat.com/openshift/install/pull-secret
cp ~/pull-secret.json /opt/ocp/pull-secret.json
chmod 600 /opt/ocp/pull-secret.json
```

**Full unattended install:**

```bash
cd /opt/kvm-host-setup
ansible-playbook deploy.yml
```

`deploy.yml` chains all phases automatically:
1. Host setup (packages, KVM, networking, DNS, HAProxy, sushy, storage)
2. Agent ISO generation
3. VM boot
4. Wait for bootstrap-complete (~20 min)
5. Wait for install-complete (~90 min)

**To wipe and reinstall:**

```bash
ansible-playbook cleanup.yml -e "unattended=true"
ansible-playbook deploy.yml
```

---

## Host Specifications

| Property | Value |
|---|---|
| OS | Fedora Linux 42 |
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

**RAM overcommit:** 3×16 + 2×8 = 64 GB on a 62 GB host. The 32 GB swapfile on `/opt`
and kernel zram provide enough headroom. VMs don't all actively use their full
allocation simultaneously during stable operation.

### Network

| Purpose | Value |
|---|---|
| Network name | `labnet` |
| Bridge | `virbr4` |
| Mode | NAT |
| Subnet | 192.168.200.0/24 |
| Gateway | 192.168.200.1 |
| DHCP range | 192.168.200.100–200 (nodes use static IPs via NMState) |
| API VIP | 192.168.200.10 → `api.ocp.lab.local`, `api-int.ocp.lab.local` |
| Ingress VIP | 192.168.200.11 → `*.apps.ocp.lab.local` |
| Cluster domain | `ocp.lab.local` |
| Rendezvous node | ocp-control-1 (192.168.200.21) |

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
│  │  BIND9        :53   (VMs)  :5353 (host NM forwarding)  │   │
│  │  HAProxy      :6443 :22623 :80 :443  (bound to VIPs)   │   │
│  │  sushy-emul.  :8000  (Redfish for assisted-service)     │   │
│  │  dnsmasq      :67   DHCP only — DNS disabled            │   │
│  └─────────────────────────────────────────────────────────┘   │
│          │ vnet1   │ vnet3   │ vnet4   │ vnet5   │ vnet6        │
│          ▼         ▼         ▼         ▼         ▼              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────┐ ┌──────┐     │
│  │control-1 │ │control-2 │ │control-3 │ │wkr-1 │ │wkr-2 │     │
│  │.21       │ │.22       │ │.23       │ │.31   │ │.32   │     │
│  └──────────┘ └──────────┘ └──────────┘ └──────┘ └──────┘     │
│                                                                 │
│  Host DNS:  app → NM dnsmasq (127.0.0.1:53)                    │
│                → ocp.lab.local? → BIND9 (127.0.0.1:5353)       │
│                → other?         → internet                      │
│  VM DNS:    VM → BIND9 (192.168.200.1:53) — direct             │
└─────────────────────────────────────────────────────────────────┘
```

---

## Infrastructure Components

### KVM / libvirt Stack

**Roles:** `packages`, `libvirt_setup`

**Packages installed:**
```
qemu-kvm  qemu-img  libvirt  libvirt-client
libvirt-daemon-driver-{qemu,network,storage-core,storage-logical}
virt-install  python3-libvirt  python3-lxml  python3-virtualenv
bind  bind-utils  dnsmasq  haproxy  nmstate
openshift-install  oc  kubectl
```

**Fedora 42 modular libvirt daemons.** The sockets that must be enabled:

| Socket | Purpose |
|---|---|
| `virtqemud.socket` | QEMU hypervisor driver |
| `virtnetworkd.socket` | Network (bridge, NAT, DHCP) |
| `virtstoraged.socket` | Storage pool management |
| `virtnodedevd.socket` | Host device enumeration |
| `virtsecretd.socket` | Secrets |
| `virtlogd.socket` | Console log forwarding |

**QEMU configuration** (`/etc/libvirt/qemu.conf`):

| Setting | Value | Reason |
|---|---|---|
| `security_driver` | `none` | Disables sVirt/SELinux labelling |
| `user` | `root` | Read qcow2 images in `/opt/images` |
| `group` | `kvm` | Access to KVM device nodes |
| `dynamic_ownership` | `1` | QEMU can chown disk images at runtime |

**VM disk cache:** The VM template uses `cache='unsafe' io='threads'` on the boot disk. This makes guest `fdatasync` a no-op on the host, which is required for the RHCOS `installation-disk-speed-check` to pass. See [T7](#t7-installation-disk-speed-check-timeout).

**UEFI boot order:** VM XML sets the boot disk (`vda`) as boot order 1 and the agent ISO (`sda`) as boot order 2. On first boot the empty disk falls through to the ISO; after coreos-installer writes RHCOS with an EFI bootloader, every subsequent boot uses the disk automatically. **The ISO never needs to be ejected.**

---

### Swap (32 GB)

**Role:** `swap`

The 5 VMs allocate 64 GB on a 62 GB host. Without sufficient swap, anonymous mmaps
that the kernel cannot back with physical pages cause `SIGSEGV` (exit 139) inside
guest Go processes — observed as `openshift-apiserver` crashing on a control node.

The role creates `/opt/swapfile` (32 GB), formats and activates it, and adds it
to `/etc/fstab`. Combined with the kernel's 8 GB zram device (priority 100),
the host has ~40 GB total swap.

---

### DNS — BIND9

**Role:** `dns`, `nm_dns`

BIND9 is authoritative for `ocp.lab.local`. It returns `NXDOMAIN` for unknown
names and `NOERROR` with empty answer for AAAA queries — no forwarding, no loops.

**Why not libvirt's dnsmasq?** libvirt dnsmasq + OCP's `ndots:5` resolv.conf
creates a AAAA query loop that causes 5-second timeouts for every OCP operator
HTTP call. BIND9 eliminates this entirely.

**Zone records:**

| Name | Type | Value |
|---|---|---|
| `api.ocp.lab.local` | A | 192.168.200.10 |
| `api-int.ocp.lab.local` | A | 192.168.200.10 |
| `*.apps.ocp.lab.local` | A | 192.168.200.11 |
| `ocp-control-{1,2,3}.ocp.lab.local` | A | 192.168.200.{21,22,23} |
| `ocp-worker-{1,2}.ocp.lab.local` | A | 192.168.200.{31,32} |

**`api-int` is critical.** RHCOS first-boot fetches ignition config from
`https://api-int.ocp.lab.local:22623/config/master`. If this name does not
resolve, the node boots without SSH keys or kubelet and can never join the cluster.

**Listen addresses:**

| Address:Port | Purpose |
|---|---|
| `192.168.200.1:53` | Direct DNS for all VMs |
| `127.0.0.1:5353` | For NM dnsmasq to forward `*.ocp.lab.local` from the host |

BIND9 has a systemd drop-in (`After=libvirtd.service`) so it starts after `virbr4`
exists. The libvirt network hook also restarts `named` after binding VIPs.

**Host NM forwarding** (`/etc/NetworkManager/dnsmasq.d/ocp.conf`):
```
server=/ocp.lab.local/127.0.0.1#5353
```

---

### Load Balancer — HAProxy + VIPs

**Roles:** `haproxy`, `vip`

HAProxy load-balances four services:

| Frontend | VIP:Port | Backend |
|---|---|---|
| API server | 192.168.200.10:6443 | control-{1,2,3}:6443 |
| Machine Config | 192.168.200.10:22623 | control-{1,2,3}:22623 |
| Ingress HTTP | 192.168.200.11:80 | worker-{1,2}:80 |
| Ingress HTTPS | 192.168.200.11:443 | worker-{1,2}:443 |

**VIP persistence via libvirt network hook** (`/etc/libvirt/hooks/network`):

VIPs must be secondary addresses on `virbr4` for ARP to work. NetworkManager
ignores `virbr*` interfaces, so a libvirt network hook binds and removes VIPs
as `labnet` starts and stops. This runs automatically on every boot.

```bash
case "$NETWORK/$ACTION" in
    labnet/started)
        ip addr add 192.168.200.10/32 dev virbr4
        ip addr add 192.168.200.11/32 dev virbr4
        sleep 1 && systemctl restart named
        ;;
    labnet/stopped)
        ip addr del 192.168.200.10/32 dev virbr4
        ip addr del 192.168.200.11/32 dev virbr4
        ;;
esac
```

HAProxy has a systemd drop-in (`After=libvirtd.service, Restart=on-failure`)
so it retries if it starts before the VIPs are bound.

---

### Libvirt Network

**Role:** `networks`

The `labnet` network has DNS **disabled** (`<dns enable='no'/>`), which prevents
libvirt's dnsmasq from binding to `192.168.200.1:53` ahead of BIND9. Dnsmasq
only handles DHCP. VMs are told to use BIND9 via a DHCP option:

```xml
<dnsmasq:option value='dhcp-option=option:dns-server,192.168.200.1'/>
```

Node IPs are static (configured via NMState in `agent-config.yaml`), but the
DHCP option is still delivered during DISCOVER/OFFER and NMState uses it
when writing the node's `resolv.conf`.

---

### RHOSO Networks

**Role:** `networks` (added to main network setup)

Two additional libvirt bridges are created for RHOSO (Red Hat OpenStack Services on OpenShift):

| Network | Bridge | Subnet | Mode | Purpose |
|---|---|---|---|---|
| `osp-internalapi` | `virbr5` | isolated L2 | isolated | OpenStack internal API traffic between services |
| `osp-provider` | `virbr6` | 10.0.100.0/24 | NAT | OpenStack external network / floating IP egress |

**osp-internalapi** is a pure L2 isolated bridge with no host IP and no DHCP. RHOSO configures node IP addresses on this interface via NMState `NodeNetworkConfigurationPolicy` (NNCP). No routing to or from outside.

**osp-provider** is a NAT bridge with host IP `10.0.100.1/24`. OpenStack's external network maps to this bridge. Floating IPs are allocated from `10.0.100.100–200` (configured in RHOSO). The host masquerades traffic for internet egress.

Each OCP node has all 3 NICs defined in the VM XML from creation time:

| NIC | Network | MAC prefix |
|---|---|---|
| eth0 / NIC 1 | `labnet` (OCP) | `52:54:00:c0:01:xx` / `52:54:00:c0:02:xx` |
| NIC 2 | `osp-internalapi` | `52:54:00:c0:03:xx` |
| NIC 3 | `osp-provider` | `52:54:00:c0:04:xx` |

**Note on PCIe slots:** Q35 VMs allocate PCIe root ports at QEMU startup. All 3 NICs must be defined in the VM XML at creation time — live hotplug of a 3rd NIC fails with "No more available PCI slots". `deploy.yml` handles this correctly. The `attach-osp-nics.yml` playbook handles migration of existing clusters (adds NICs to persistent XML, then cold-restarts VMs).

---

### Redfish Emulator — sushy

**Role:** `sushy`

[sushy-tools](https://opendev.org/openstack/sushy-tools) is an OpenStack Redfish
API emulator backed by libvirt. It exposes each KVM VM as a Redfish System at:

```
http://192.168.200.1:8000/redfish/v1/Systems/<vm-name>
```

This allows `openshift-install` to communicate with the assisted-service using
a `redfish-virtualmedia://` BMC address in `agent-config.yaml`, which is the
standard interface for assisted installs.

**Critical venv setup:** sushy requires `python3-libvirt` (system package).
The virtualenv must be created with `--system-site-packages` so it can see the
system libvirt bindings:

```bash
python3 -m venv --system-site-packages /opt/sushy-venv
```

Without `--system-site-packages`, sushy logs `libvirt driver not loaded` and
cannot enumerate VMs.

**Config** (`/etc/sushy/sushy.conf`):

```python
SUSHY_EMULATOR_LISTEN_IP = '0.0.0.0'
SUSHY_EMULATOR_LISTEN_PORT = 8000
SUSHY_EMULATOR_LIBVIRT_URI = 'qemu:///system'
SUSHY_EMULATOR_IGNORE_BOOT_DEVICE = True   # VM XML controls boot order
```

`SUSHY_EMULATOR_IGNORE_BOOT_DEVICE = True` lets the VM XML control boot order
rather than having sushy manipulate EFI NVRAM, which keeps the boot order fix
(disk=1, ISO=2) intact.

Verify sushy is working:
```bash
curl -s http://192.168.200.1:8000/redfish/v1/Systems/ | python3 -m json.tool
```

---

### WireGuard VPN

**Playbook:** `wireguard.yml` (standalone — not part of `site.yml`)

WireGuard gives remote access to the KVM host and all VM networks from a Mac.

| Host | WireGuard IP | Routes via tunnel |
|---|---|---|
| KVM host | 10.10.0.1 | — |
| Mac client | 10.10.0.2 | 192.168.200.0/24, 10.0.100.0/24 |

**Setup:**

```bash
# 1. Run the playbook (generates keys, deploys config, starts wg-quick@wg0)
ansible-playbook wireguard.yml

# 2. Copy client config to Mac
scp <kvm-host>:/etc/wireguard/client.conf ~/wg-ocp-lab.conf

# 3. Import into WireGuard app on macOS (File → Import Tunnel)
```

Install the [WireGuard macOS app](https://apps.apple.com/us/app/wireguard/id1451685025) from the App Store.

**For internet (outside LAN) access:** Set `wireguard_server_endpoint` in [group_vars/all.yml](group_vars/all.yml) to your router's public IP and forward UDP port 51820 → `192.168.1.154` on your router.

**Re-running** `wireguard.yml` is safe and idempotent — keys are only generated once (`creates:` guard). To force new keys, delete `/etc/wireguard/server.key` and `/etc/wireguard/client.key` on the host first, then re-run.

**DNS over VPN:** The client config sets DNS to `192.168.200.1` (BIND9), so `*.ocp.lab.local` resolves correctly when the tunnel is active.

---

### System Sleep Prevention

The KVM host must not suspend during a long OCP install. GNOME's default
15-minute suspend timeout has caused SSH disconnections and interrupted deploys.

**One-time host configuration:**

```bash
# GNOME power settings
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-timeout 0
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'
gsettings set org.gnome.desktop.session idle-delay 0

# Mask systemd sleep targets
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target

# Prevent logind from suspending on lid close / key press
sudo tee /etc/systemd/logind.conf.d/nosleep.conf <<'EOF'
[Login]
HandleSuspendKey=ignore
HandleHibernateKey=ignore
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
IdleAction=ignore
EOF
sudo systemctl restart systemd-logind
```

Verify:
```bash
systemctl is-active sleep.target      # → inactive (masked)
systemctl is-active suspend.target    # → inactive (masked)
```

---

## OCP Deployment

### Unattended (deploy.yml)

The recommended path for a complete install from scratch:

```bash
# Prerequisites verified — then:
ansible-playbook deploy.yml
```

`deploy.yml` runs four phases in sequence:
1. `site.yml` — full host setup (packages through storage pool; VMs deferred)
2. `generate-ocp-config.yml` — render configs, generate agent ISO
3. Start all 5 VMs
4. `eject-iso.yml` — wait for bootstrap-complete, then install-complete

Total time: approximately 2–2.5 hours from a clean host.

---

### Manual Step-by-Step

Use this when you need to run phases independently or troubleshoot.

#### Step 1 — Wipe Everything (if reinstalling)

```bash
ansible-playbook cleanup.yml
# or unattended:
ansible-playbook cleanup.yml -e "unattended=true"
```

Destroys all VMs, networks, pools, and wipes `/opt/ocp/cluster`. Does not
remove `/opt/ocp/pull-secret.json`.

#### Step 2 — Host Setup

```bash
ansible-playbook site.yml --skip-tags vms
```

Configures in order: packages → swap → libvirt → networks → DNS → NM DNS →
VIPs → HAProxy → sushy → storage pool.

Individual roles can be re-run with tags:
```bash
ansible-playbook site.yml --tags dns       # re-deploy BIND config only
ansible-playbook site.yml --tags haproxy   # re-deploy HAProxy config only
ansible-playbook site.yml --tags sushy     # re-deploy sushy emulator
```

**Verify before continuing:**
```bash
# DNS
dig api.ocp.lab.local A +short            # → 192.168.200.10
dig api-int.ocp.lab.local A +short        # → 192.168.200.10
dig anything.apps.ocp.lab.local A +short  # → 192.168.200.11
dig ocp-control-1.ocp.lab.local A +short  # → 192.168.200.21
dig api.ocp.lab.local AAAA +time=2        # → NOERROR, empty (no timeout)

# VIPs
ip addr show virbr4 | grep '192.168.200.1[01]'

# HAProxy
haproxy -c -f /etc/haproxy/haproxy.cfg && echo "config OK"
systemctl is-active haproxy

# sushy
curl -s http://192.168.200.1:8000/redfish/v1/Systems/ | python3 -m json.tool

# Swap
swapon --show
grep swapfile /etc/fstab
```

#### Step 3 — Pull Secret

```bash
cp ~/pull-secret.json /opt/ocp/pull-secret.json
chmod 600 /opt/ocp/pull-secret.json

# Verify
python3 -c "
import json; ps = json.load(open('/opt/ocp/pull-secret.json'))
print('Keys:', list(ps['auths'].keys()))
"
```

#### Step 4 — Generate Agent ISO

```bash
ansible-playbook generate-ocp-config.yml
```

Renders `install-config.yaml` and `agent-config.yaml` from Jinja2 templates,
then calls `openshift-install agent create image`. The ISO is written to
`/opt/ocp/cluster/agent.x86_64.iso` (~1.4 GB).

All 5 VMs boot the **same** ISO. The agent on each node matches its MAC address
to determine its role, static IP, and whether it is the rendezvous node
(control-1, 192.168.200.21, which hosts the bootstrap cluster).

#### Step 5 — Start VMs

```bash
ansible-playbook site.yml --tags vms
```

Creates 120 GB thin-provisioned qcow2 images in `/opt/images/`, defines VMs
with the agent ISO attached, and starts all 5 nodes. UEFI boot order: disk=1
(empty, falls through), ISO=2 (boots the agent installer).

#### Step 6 — Wait for Bootstrap + Install Complete

```bash
ansible-playbook eject-iso.yml
```

Runs `openshift-install agent wait-for bootstrap-complete` (up to 90 min),
then `wait-for install-complete` (up to 120 min). No ISO ejection is needed —
after coreos-installer writes RHCOS, the disk's EFI bootloader takes over on
every subsequent reboot automatically.

While waiting, monitor operator status in another terminal:
```bash
export KUBECONFIG=/opt/ocp/cluster/auth/kubeconfig
watch -n15 "oc get co | grep -v 'True.*False.*False'"
```

---

## Post-Install Verification

Run these checks after `eject-iso.yml` completes:

```bash
export KUBECONFIG=/opt/ocp/cluster/auth/kubeconfig

# All 5 nodes Ready
oc get nodes

# All operators Available=True, Progressing=False, Degraded=False
oc get co

# No stuck pods
oc get pods -A | grep -v -E 'Running|Completed'

# Cluster version
oc get clusterversion

# kubeadmin credentials
cat /opt/ocp/cluster/auth/kubeadmin-password

# Web console
echo "https://console-openshift-console.apps.ocp.lab.local"
```

**Access from outside the KVM host** — add to workstation `/etc/hosts`:
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
| `ocp_version` | `4.18` | Version for binary downloads |
| `api_vip` | `192.168.200.10` | API load-balancer VIP |
| `ingress_vip` | `192.168.200.11` | Ingress/router VIP |
| `ocp_network_cidr` | `192.168.200.0/24` | Machine network |
| `ocp_network_gateway` | `192.168.200.1` | Bridge IP, DHCP server, BIND9 |
| `control_vm_vcpus` | `4` | vCPUs per control node |
| `control_vm_ram_mb` | `16384` | RAM per control node (OCP minimum) |
| `control_vm_disk_gb` | `120` | Disk per control node |
| `worker_vm_vcpus` | `4` | vCPUs per worker |
| `worker_vm_ram_mb` | `8192` | RAM per worker |
| `worker_vm_disk_gb` | `120` | Disk per worker |
| `libvirt_storage_pool_path` | `/opt/images` | qcow2 image directory |
| `ocp_install_dir` | `/opt/ocp/cluster` | openshift-install working directory |
| `sushy_port` | `8000` | sushy-emulator listen port |

---

## Project Structure

```
kvm-host-setup/
├── ansible.cfg                          # Inventory, become, callbacks
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
│   │   └── templates/ocp.lab.local.zone.j2
│   ├── nm_dns/
│   │   ├── tasks/main.yml               # NM dnsmasq forwarding config
│   │   └── templates/ocp.conf.j2        # server=/ocp.lab.local/127.0.0.1#5353
│   ├── vip/
│   │   ├── tasks/main.yml               # Deploy libvirt hook, bind VIPs now
│   │   └── templates/network.hook.j2    # Hook: bind/unbind VIPs + restart named
│   ├── haproxy/
│   │   ├── tasks/main.yml               # HAProxy systemd override + config
│   │   └── templates/haproxy.cfg.j2     # VIP-bound frontends for API + ingress
│   ├── sushy/
│   │   ├── tasks/main.yml               # Install sushy-tools venv, service
│   │   ├── handlers/main.yml            # Reload systemd, restart sushy-emulator
│   │   ├── templates/sushy.conf.j2      # Redfish emulator config
│   │   └── templates/sushy-emulator.service.j2
│   ├── wireguard/
│   │   ├── tasks/main.yml               # Install, generate keys, deploy config
│   │   ├── handlers/main.yml            # Restart wg-quick@wg0
│   │   ├── templates/wg0.conf.j2        # Server config (PostUp/PostDown iptables)
│   │   └── templates/wg-client.conf.j2  # Client config for macOS
│   ├── storage_pool/tasks/main.yml      # /opt/images pool in libvirt
│   └── vms/
│       ├── tasks/main.yml               # Disk images + VM XML + start
│       └── templates/vm.xml.j2          # VM: q35, UEFI, 3 NICs, disk=1/ISO=2
│
├── ocp-config/
│   ├── install-config.yaml.j2           # Cluster config (network, replicas)
│   └── agent-config.yaml.j2             # Per-node NMState static IP + BMC
│
├── site.yml                             # Full host setup
├── deploy.yml                           # Unattended end-to-end deploy
├── cleanup.yml                          # Wipe VMs, networks, install dirs
├── generate-ocp-config.yml              # Render configs + openshift-install create image
├── eject-iso.yml                        # Wait for bootstrap-complete + install-complete
├── wireguard.yml                        # Install/configure WireGuard VPN (standalone)
└── attach-osp-nics.yml                  # Add RHOSO NICs to existing cluster (migration)
```

---

## Troubleshooting

### T1 — `Invalid callback for stdout specified: yaml`

```
ERROR! Invalid callback for stdout specified: yaml
```

`ansible-core` on Fedora 42 does not ship the `yaml` stdout callback.

**Fix:** Use `ansible.builtin.default` in `ansible.cfg`:
```ini
[defaults]
stdout_callback   = ansible.builtin.default
```

---

### T2 — `sudo: a password is required` during fact gathering

**Fix:** Configure passwordless sudo or pass `-K`:
```bash
echo "$USER ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/$USER-nopasswd
# or:
ansible-playbook site.yml -K
```

---

### T3 — DNS resolution broken — `Could not resolve host`

**Symptom:** DNF or host DNS fails after cleanup.

**Cause:** Stale NM connection profile with a dead `ipv4.dns` entry.

**Fix:**
```bash
sudo nmcli connection show          # look for stale virbr/provisioning entries
sudo nmcli connection delete <stale-name>
sudo systemctl restart NetworkManager
```

---

### T4 — `dnsmasq: Address already in use` on 192.168.200.1:53

**Cause 1:** Stale PID files from old monolithic libvirtd:
```bash
sudo kill $(cat /var/run/libvirt/network/*.pid 2>/dev/null) 2>/dev/null
sudo rm -rf /var/run/libvirt/network/* /var/lib/libvirt/dnsmasq/virbr*.status
```

**Cause 2:** BIND9 started before `virbr4` and grabbed `0.0.0.0:53`:
```bash
sudo systemctl restart named
```

---

### T5 — `exec: "nmstatectl": executable file not found`

```
failed to validate network yaml ... exec: "nmstatectl": executable file not found
```

**Fix:**
```bash
sudo dnf install -y nmstate
ansible-playbook generate-ocp-config.yml
```

---

### T6 — `unknown field 'macAddress'` in NMState validation

**Cause:** NMState 2.x requires `mac-address` (hyphenated) inside
`networkConfig.interfaces`. Note the distinction in `agent-config.yaml`:
- `hosts[].interfaces[].macAddress` — OCP AgentConfig API field (camelCase, correct)
- `hosts[].networkConfig.interfaces[].mac-address` — NMState 2.x field (hyphenated, correct)

The template uses the correct form for both.

---

### T7 — `installation-disk-speed-check` timeout (exit 124)

**Symptom:** All nodes stuck at `preparing-for-installation`.

**Cause:** OCP runs `fio --rw=write --ioengine=sync --fdatasync=1`. With
`cache='none'` every `fdatasync` flushes to physical disk, giving <10 IOPS on
a new qcow2.

**Fix:** The VM template uses `cache='unsafe' io='threads'` — guest `fdatasync`
becomes a no-op on the host. Acceptable for a lab.

---

### T8 — DNS validation warning on nodes

**Symptom:** Nodes stuck at agent validation with DNS warnings.

**Check:**
```bash
# Must return 192.168.200.10 immediately
dig api-int.ocp.lab.local @192.168.200.1 +short

# Must return in <2s (no timeout = no loop)
timeout 2 dig api.ocp.lab.local AAAA @192.168.200.1 | grep status
```

**Fix:** If `api-int` is missing:
```bash
ansible-playbook site.yml --tags dns
```

---

### T9 — Node booted back into installer after image write

**Symptom:**
```
Expected the host to boot from disk, but it booted the installation image
```
Or node shows `installing-pending-user-action` in the assisted-service UI.

**Why it happens:** The ISO is still boot order 2 in the UEFI. This should not
occur with the current VM template (disk=1 in UEFI means the disk boots first
once RHCOS writes its EFI bootloader to vda). If it does occur, verify the VM
XML has the correct boot order:

```bash
sudo virsh -c qemu:///system dumpxml ocp-control-1 | grep -A2 'boot order'
```

The disk (`vda`) must have `<boot order='1'/>` and the cdrom (`sda`) must have
`<boot order='2'/>`. If VMs were created before this fix, destroy and recreate:

```bash
ansible-playbook cleanup.yml -e "unattended=true"
ansible-playbook deploy.yml
```

---

### T10 — Bootstrap-complete timeout — nodes never joined cluster

**Root cause:** `api-int.ocp.lab.local` did not resolve when a node's RHCOS
first-boot ran ignition. The node booted without SSH keys or kubelet. DNS
fixes do not help — ignition runs once in the initramfs and does not retry.

**Recovery:** Full reinstall.
```bash
ansible-playbook cleanup.yml -e "unattended=true"
ansible-playbook deploy.yml
```

**Prevention:** Verify before starting VMs:
```bash
dig api-int.ocp.lab.local @192.168.200.1 +short   # MUST return 192.168.200.10
```

---

### T11 — `openshift-apiserver` pod SIGSEGV (exit 139)

**Cause:** Memory overcommit exhausted. 64 GB allocated on 62 GB host; swap
not active.

**Fix:**
```bash
swapon --show
grep swapfile /etc/fstab

# If missing:
echo '/opt/swapfile swap swap defaults 0 0' | sudo tee -a /etc/fstab
sudo swapon /opt/swapfile
```

---

### T12 — Squashfs read errors on first RHCOS boot

**Symptom:** VM console shows `SQUASHFS error: Unable to read page` and node
reboots in a loop.

**Cause:** Transient I/O contention when 3–5 VMs simultaneously boot. The RHCOS
image write is complete — this is a timing issue, not disk damage.

**Fix:**
```bash
sudo virsh reset ocp-worker-2
```

The next boot has no I/O contention and RHCOS comes up cleanly. If errors
persist after 2–3 resets:
```bash
sudo virsh screenshot ocp-worker-2 /tmp/screen.ppm   # view in an image viewer
qemu-img check /opt/images/ocp-worker-2.qcow2
```

---

### T13 — Worker stuck: `machine-config-daemon-pull` skipped, node never Ready

**Symptom:** Node boots to login prompt but never appears Ready. `kubelet` and
`crio` not running.

**Cause:** Node crashed mid-ignition (e.g., squashfs error from T12). Ignition
marked firstboot done but never wrote
`/etc/ignition-machine-config-encapsulated.json`. MCD's `ConditionPathExists`
check fails silently.

**Fix:**
```bash
ssh core@<node-ip>
sudo cp /etc/ignition-machine-config-encapsulated.json.bak \
        /etc/ignition-machine-config-encapsulated.json
sudo systemctl start machine-config-daemon-pull
sudo systemctl start machine-config-daemon-firstboot
```

The node will apply its MachineConfig and reboot automatically — this is normal.
After the reboot it joins the cluster.

---

### T14 — `wait-for install-complete` times out: operators not Available

**Symptom:**
```
ERROR failed to initialize the cluster: Cluster operators authentication,
console, ingress, monitoring are not available
```

**Cause:** Operators that require workers (`ingress`, `monitoring`, `console`,
`authentication`) are degraded if a worker is missing. The default 120-minute
timeout may not be enough if workers join very late.

**Diagnose:**
```bash
export KUBECONFIG=/opt/ocp/cluster/auth/kubeconfig
oc get nodes
oc get co
oc get csr | grep Pending          # approve if any are waiting
oc adm certificate approve $(oc get csr -o name | grep Pending)
```

Fix worker issues (T12/T13) then restart the monitor (there is no harm running
it multiple times):
```bash
nohup /usr/local/bin/openshift-install --dir /opt/ocp/cluster \
    wait-for install-complete --log-level=info \
    >> /tmp/install-complete.log 2>&1 &
tail -f /tmp/install-complete.log
```

---

### T15 — `openshift-install` monitor exits before install completes

The `wait-for` commands are monitors only — killing them does not affect the
cluster. Restart any time:

```bash
pgrep -a openshift-install

# Restart bootstrap-complete monitor
nohup /usr/local/bin/openshift-install --dir /opt/ocp/cluster \
    wait-for bootstrap-complete --log-level=info \
    >> /tmp/bootstrap-complete.log 2>&1 &

# Restart install-complete monitor
nohup /usr/local/bin/openshift-install --dir /opt/ocp/cluster \
    wait-for install-complete --log-level=info \
    >> /tmp/install-complete.log 2>&1 &
```

Install log (ground truth): `/opt/ocp/cluster/.openshift_install.log`

---

### T16 — sushy-emulator: `libvirt driver not loaded`

**Symptom:** `systemctl status sushy-emulator` shows `libvirt driver not loaded`.
`curl http://192.168.200.1:8000/redfish/v1/Systems/` returns an error or empty list.

**Cause:** The sushy venv was created without `--system-site-packages` and cannot
see the system `python3-libvirt` package.

**Fix:**
```bash
sudo systemctl stop sushy-emulator
sudo rm -rf /opt/sushy-venv
sudo python3 -m venv --system-site-packages /opt/sushy-venv
sudo /opt/sushy-venv/bin/pip install sushy-tools
sudo systemctl start sushy-emulator

# Verify
curl -s http://192.168.200.1:8000/redfish/v1/Systems/ | python3 -m json.tool
```

Or re-run the Ansible role:
```bash
ansible-playbook site.yml --tags sushy
```

---

### T17 — Host goes to sleep during install (SSH disconnects)

**Symptom:** SSH session drops repeatedly during long install. VMs keep running
but openshift-install monitor (running in the foreground) is killed.

**Cause:** GNOME default: `sleep-inactive-ac-timeout = 900` (15 min).

**Fix:** See [System Sleep Prevention](#system-sleep-prevention) above.

Quick check:
```bash
gsettings get org.gnome.settings-daemon.plugins.power sleep-inactive-ac-timeout
# should be 0
systemctl is-active suspend.target   # should be inactive (masked)
```

---

### Diagnostic Commands

```bash
# VM console (Ctrl+] to exit)
virsh console ocp-control-1

# Screenshot VM console
sudo virsh screenshot ocp-worker-2 /tmp/screen.ppm

# SSH into a node
ssh core@192.168.200.21

# Clear stale host key after reinstall
ssh-keygen -R 192.168.200.31

# Watch agent bootstrap logs on rendezvous node
ssh core@192.168.200.21 journalctl -u agent.service -f

# Install log on KVM host
tail -f /opt/ocp/cluster/.openshift_install.log

# Cluster status
export KUBECONFIG=/opt/ocp/cluster/auth/kubeconfig
oc get nodes
oc get co
oc get pods -A | grep -v -E 'Running|Completed'

# Approve pending CSRs
oc get csr | grep Pending
oc get csr -o name | xargs oc adm certificate approve

# sushy health
curl -s http://192.168.200.1:8000/redfish/v1/Systems/ | python3 -m json.tool
journalctl -u sushy-emulator -n 50

# BIND9
dig api.ocp.lab.local @192.168.200.1 +short
dig api-int.ocp.lab.local @192.168.200.1 +short
sudo rndc status

# HAProxy
haproxy -c -f /etc/haproxy/haproxy.cfg
echo "show stat" | sudo socat stdio /run/haproxy/admin.sock | cut -d',' -f1,2,18

# VIP check
ip addr show virbr4 | grep 192.168.200.1

# VM list and state
virsh -c qemu:///system list --all
```
