# KVM Host Setup for OCP 4.18 (Agent-based)

Ansible playbooks to clean and configure a bare-metal KVM host, then deploy
OpenShift Container Platform 4.18 using the agent-based installation method.

---

## Host Specifications

| Property | Value |
|---|---|
| OS | Fedora Linux 42 (Workstation) |
| Kernel | 6.19.8-100.fc42.x86_64 |
| CPU | Intel Core i7-10710U — 6 cores / 12 threads, VT-x |
| RAM | 62 GB (8 GB swap) |
| Root disk | NVMe 29 GB (btrfs, `/` + `/home`) |
| Data disk | `/dev/sda1` ext4 733 GB → `/opt` |
| KVM stack | qemu-kvm 9.2.4, libvirt 11.0.0 (modular daemons) |
| SELinux | permissive (config: enforcing) |

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

### Network

| Purpose | Value |
|---|---|
| Network name | `labnet` |
| Bridge | `virbr4` |
| Mode | NAT |
| Subnet | 192.168.200.0/24 |
| Gateway / DNS | 192.168.200.1 (libvirt dnsmasq) |
| DHCP range | 192.168.200.100–200 |
| API VIP | 192.168.200.10 |
| Ingress VIP | 192.168.200.11 |
| Cluster domain | `ocp.lab.local` |

DNS is handled by libvirt's built-in dnsmasq on `virbr4`, with a NetworkManager
stub zone forwarding `*.ocp.lab.local` to `192.168.200.1` on the host.

---

## Project Structure

```
kvm-host-setup/
├── ansible.cfg                        # Inventory path, become, callbacks
├── inventory/
│   └── hosts.yml                      # Single host: localhost (local connection)
├── group_vars/
│   └── all.yml                        # All variables — edit this first
│
├── roles/
│   ├── cleanup/tasks/main.yml         # Destroy VMs, networks, pools; wipe dirs
│   ├── packages/tasks/main.yml        # dnf + openshift-install + oc downloads
│   ├── libvirt_setup/tasks/main.yml   # Modular daemons, qemu.conf, groups, sysctl
│   ├── networks/tasks/main.yml        # Define labnet + DNS/hosts config
│   ├── haproxy/tasks/main.yml         # HAProxy LB config + VIP assignment to virbr4
│   ├── storage_pool/tasks/main.yml    # /opt/images pool
│   └── vms/tasks/main.yml            # Disk images + VM XML definitions
│
├── ocp-config/
│   ├── install-config.yaml.j2         # OCP install-config template
│   └── agent-config.yaml.j2          # Per-node NMState static IP config
│
├── site.yml                           # Full host setup playbook
├── cleanup.yml                        # Wipe-only playbook (with confirmation)
└── generate-ocp-config.yml           # Render configs + run openshift-install
```

---

## Prerequisites

1. **Fedora 42** with `qemu-kvm` and `libvirt` already installed
   (`ansible-core` is installed by the `packages` role).
2. **Pull secret** from [console.redhat.com](https://console.redhat.com/openshift/install/pull-secret).
3. **SSH key pair** at `~/.ssh/id_rsa` (generated automatically if missing).
4. `/opt` mounted on a large disk (≥ 500 GB free).

---

## Step-by-Step Deployment

### 1 — Wipe the existing environment

Destroys all running VMs, undefines networks and storage pools, wipes
`/opt/images`, `/opt/ocp`, `/opt/ocp-build`, `/opt/ocp-lab`, cleans the dnf
cache, vacuums the systemd journal, and prunes Docker.

```bash
cd /opt/kvm-host-setup
ansible-playbook cleanup.yml
```

### 2 — Set up the KVM host

Installs packages, configures libvirt modular daemons, defines the `labnet`
network, creates the `/opt/images` storage pool, and creates VM disk images +
XML definitions (VMs are **not** started yet — the agent ISO doesn't exist yet).

```bash
ansible-playbook site.yml
```

You can target individual roles with tags:

```bash
ansible-playbook site.yml --tags packages
ansible-playbook site.yml --tags libvirt
ansible-playbook site.yml --tags networks
ansible-playbook site.yml --tags storage
ansible-playbook site.yml --tags vms
```

### 3 — Add your pull secret

```bash
cp ~/pull-secret.json /opt/ocp/pull-secret.json
chmod 600 /opt/ocp/pull-secret.json
```

### 4 — Generate the agent ISO

Renders `install-config.yaml` and `agent-config.yaml` from templates, then
calls `openshift-install agent create image` to produce `agent.x86_64.iso`.

```bash
ansible-playbook generate-ocp-config.yml
```

To override variables inline:

```bash
ansible-playbook generate-ocp-config.yml \
  -e "cluster_name=mycluster base_domain=example.com"
```

### 5 — Start the VMs

The `vms` role detects the ISO and boots all nodes.

```bash
ansible-playbook site.yml --tags vms
```

### 6 — Monitor installation

```bash
# Wait for bootstrap to complete (~15 min)
openshift-install --dir=/opt/ocp/cluster agent wait-for bootstrap-complete \
  --log-level=info

# Wait for full install to complete (~45 min total)
openshift-install --dir=/opt/ocp/cluster agent wait-for install-complete \
  --log-level=info
```

### 7 — Access the cluster

```bash
export KUBECONFIG=/opt/ocp/cluster/auth/kubeconfig
oc get nodes
oc get co   # cluster operators
```

Web console: `https://console-openshift-console.apps.ocp.lab.local`

---

## Customisation

All tunable values live in [group_vars/all.yml](group_vars/all.yml):

| Variable | Default | Description |
|---|---|---|
| `cluster_name` | `ocp` | OCP cluster name |
| `base_domain` | `lab.local` | Base DNS domain |
| `ocp_version` | `4.18` | Used for client downloads |
| `api_vip` | `192.168.200.10` | API load-balancer VIP |
| `ingress_vip` | `192.168.200.11` | Ingress/router VIP |
| `ocp_network_cidr` | `192.168.200.0/24` | Machine network |
| `control_vm_vcpus` | `4` | vCPUs per control node |
| `control_vm_ram_mb` | `16384` | RAM per control node (MB) |
| `control_vm_disk_gb` | `120` | Disk per control node (GB) |
| `worker_vm_vcpus` | `4` | vCPUs per worker |
| `worker_vm_ram_mb` | `8192` | RAM per worker (MB) |
| `worker_vm_disk_gb` | `120` | Disk per worker (GB) |
| `libvirt_storage_pool_path` | `/opt/images` | Where qcow2 images are stored |
| `ocp_install_dir` | `/opt/ocp/cluster` | openshift-install working dir |

---

## Key Design Notes

**Fedora 42 modular libvirt daemons** — Fedora 38+ replaced the monolithic
`libvirtd` with split socket-activated daemons (`virtqemud`, `virtnetworkd`,
`virtstoraged`, etc.). The `libvirt_setup` role enables the correct sockets.

**UEFI / q35 VMs** — All VMs use the `q35` machine type with UEFI firmware
(`--nvram` must be passed to `virsh undefine` to clean up NVRAM state, which
the cleanup role does).

**Agent-based install** — A single `agent.x86_64.iso` is attached to all
nodes as a CD-ROM (boot order 1). The first node to boot becomes the
rendezvous host. After install completes the nodes reboot from their vda disk
(boot order 2).

**Static IPs via DHCP reservation** — Each VM is assigned a deterministic
MAC address. The libvirt network's dnsmasq binds that MAC to a fixed IP, so
the NMState config in `agent-config.yaml` and the DHCP lease always agree.

**No extra Ansible collections required** — Only `ansible.builtin.*` modules
are used. `community.general` and `ansible.posix` are not installed on this
host and are avoided.

---

## Troubleshooting

### [T1] `Invalid callback for stdout specified: yaml`

**Symptom** — First `ansible-playbook` run fails immediately:
```
ERROR! Invalid callback for stdout specified: yaml
```

**Cause** — `ansible-core` on Fedora 42 does not ship the `yaml` stdout
callback plugin. Only `ansible.builtin.*` callbacks are available without
extra collections.

**Fix** — Use the built-in `default` callback and the FQCN for
`profile_tasks` in `ansible.cfg`:

```ini
[defaults]
stdout_callback   = ansible.builtin.default
callbacks_enabled = ansible.posix.profile_tasks
```

To check which callbacks are available on your system:
```bash
ansible-doc -t callback -l
```

---

### [T2] `sudo: a password is required` — fact gathering fails

**Symptom** — Playbook fails at `Gathering Facts`:
```
fatal: [kvm_host]: FAILED! => {
  "msg": "MODULE FAILURE ... sudo: a password is required"
}
```

**Cause** — `become = True` is set globally in `ansible.cfg` so Ansible
tries to run even fact gathering as root, but no sudo password was supplied.

**Fix** — Add `become_ask_pass = True` to `ansible.cfg` so Ansible prompts
for the sudo password at the start of each run:

```ini
[privilege_escalation]
become          = True
become_method   = sudo
become_ask_pass = True
```

Alternatively, pass `-K` on the command line:
```bash
ansible-playbook site.yml -K
```

---

### [T3] DNS resolution broken — `Could not resolve host`

**Symptom** — `packages` role fails when DNF tries to download metadata:
```
fatal: [kvm_host]: FAILED! => {
  "msg": "Failed to download metadata ... Cannot prepare internal mirrorlist:
          Curl error (6): Could not resolve hostname for
          https://mirrors.fedoraproject.org ..."
}
```

**Cause** — Two stale configurations from a previous OCP lab deployment were
poisoning the host DNS:

1. **`/etc/systemd/resolved.conf`** was hardcoded to `DNS=127.0.0.1` — a
   dnsmasq instance that no longer exists after cleanup — and
   `DNSStubListener=no`, disabling the 127.0.0.53 stub resolver.

2. **NetworkManager `virbr1` connection profile** had `ipv4.dns=192.168.10.2`
   (the old provisioning network gateway) which NM pushes to systemd-resolved
   as a global DNS server even when the interface is down.

Together these routed all DNS queries to dead servers. Diagnosing:
```bash
resolvectl status          # shows Global DNS, Domains, resolv.conf mode
nmcli connection show virbr1 | grep dns
cat /etc/systemd/resolved.conf
```

**Fix** — Reset systemd-resolved to use public DNS, re-enable the stub
listener, and delete the dead virbr1 NM connection profile:

```bash
# 1. Fix systemd-resolved
sudo tee /etc/systemd/resolved.conf <<'EOF'
[Resolve]
DNS=8.8.8.8 1.1.1.1
FallbackDNS=8.8.4.4 9.9.9.9
DNSStubListener=yes
EOF

# 2. Delete the stale virbr1 NM connection (provisioning net is gone)
sudo nmcli connection delete virbr1

# 3. Restart both services
sudo systemctl restart systemd-resolved NetworkManager

# 4. Verify resolution is working
dig mirrors.fedoraproject.org +short | head -3
```

**Root cause note** — The `cleanup` role destroys libvirt networks but does
not remove the corresponding NetworkManager connection profiles or
`/etc/systemd/resolved.conf`. If DNS breaks again after a cleanup run, repeat
the steps above.

---

### [T4] `dnsmasq: failed to create listening socket — Address already in use`

**Symptom** — `networks` role fails when starting `labnet`:
```
error: Failed to start network labnet
dnsmasq: failed to create listening socket for 192.168.200.1: Address already in use
```

**Cause** — Stale dnsmasq processes are still running from networks that were
created by the old monolithic `libvirtd`. After switching to modular libvirt
daemons (`virtnetworkd`), those networks are invisible to `virsh net-list` so
the cleanup role's `virsh net-destroy` skips them — but their dnsmasq
processes and PID files remain in `/var/run/libvirt/network/`.

Diagnose:
```bash
ls /var/run/libvirt/network/      # stale *.pid files are the giveaway
ls /var/lib/libvirt/dnsmasq/     # virbr*.status files also indicate stale state
```

**Fix** — Kill the orphaned processes and clear the stale runtime state, then
retry:

```bash
# Kill dnsmasq processes still referenced by stale PID files
sudo kill $(cat /var/run/libvirt/network/br_labnet.pid) 2>/dev/null
sudo kill $(cat /var/run/libvirt/network/provisioning.pid) 2>/dev/null

# Remove all stale runtime state
sudo rm -rf /var/run/libvirt/network/* \
            /var/lib/libvirt/dnsmasq/virbr*.status

# Retry
ansible-playbook site.yml --tags networks
```

**If the error persists after clearing PID files** — the libvirt `default`
network may still be active and its dnsmasq may be binding to a wildcard
address. The root cause is that `virsh net-list` without root returns an
empty list (connects to the user session socket, not the system socket), so
the cleanup role was silently skipping the `default` network.

Verify with sudo:
```bash
sudo virsh -c qemu:///system net-list --all
```

Destroy the default network, then retry:
```bash
sudo virsh -c qemu:///system net-destroy default
sudo virsh -c qemu:///system net-undefine default
ansible-playbook site.yml --tags networks
```

All `virsh` calls in the playbooks now use `-c qemu:///system` explicitly so
they always connect to the system socket regardless of the user running them.
The cleanup role no longer skips the `default` network.

The `libvirt_setup` role also writes an NM config to leave `virbr*`/`vnet*`
interfaces unmanaged, preventing NM from racing for port 53:

```bash
sudo tee /etc/NetworkManager/conf.d/99-libvirt.conf <<'EOF'
[keyfile]
unmanaged-devices=interface-name:virbr*;interface-name:vnet*
EOF
sudo nmcli general reload conf
```

---

### [T5] `named` (BIND9) grabs port 53 on the bridge IP

**Symptom** — Same `Address already in use` error on `192.168.200.1:53`.
All previous fixes applied but the problem persists. Diagnostic:

```bash
sudo ss -tulnp | grep ':53'
# shows: named pid=XXXXX listening on 127.0.0.1:53 and all interface IPs
```

**Cause** — BIND9 (`named`) was previously installed and is running as a
system service. BIND listens on **all interfaces** by default, including
new ones that appear dynamically. When libvirt creates `virbr4` with
`192.168.200.1`, `named` immediately binds `192.168.200.1:53`, so libvirt's
dnsmasq cannot bind to the same address moments later.

**Fix** — Stop and disable `named`. For this KVM lab, libvirt's built-in
dnsmasq provides DNS for the OCP network; BIND9 is not needed.

```bash
sudo systemctl stop named
sudo systemctl disable named

# Redefine the network and retry
sudo virsh -c qemu:///system net-undefine labnet
ansible-playbook site.yml --tags networks
```

The `libvirt_setup` role now automatically stops and disables `named` if it
is found running, so this will not recur on future runs.

**Note** — If you need `named` for other purposes, configure it to listen
only on loopback instead:

```bash
# Add to /etc/named.conf options block:
#   listen-on { 127.0.0.1; };
#   listen-on-v6 { ::1; };
sudo systemctl restart named
```

---

### [T6] `exec: "nmstatectl": executable file not found in $PATH`

**Symptom** — `generate-ocp-config.yml` fails at the "Generate agent ISO" task:

```
level=fatal msg=failed to generate asset "NMState Config": staticNetwork
  configuration is not valid: 5 errors occurred:
level=fatal msg=  * failed to validate network yaml for host 0, install
  nmstate package, exec: "nmstatectl": executable file not found in $PATH
...
```

**Cause** — `openshift-install agent create image` shells out to `nmstatectl`
to validate every host's static network YAML before writing the ISO. The
`nmstate` package (which provides `nmstatectl`) is a runtime dependency of the
agent-based installer but was not in the KVM host package list.

**Fix** — Install `nmstate` on the machine running `openshift-install`:

```bash
sudo dnf install -y nmstate
```

Verify:
```bash
nmstatectl version
```

Then re-run the playbook:
```bash
ansible-playbook generate-ocp-config.yml
```

The `packages` role now includes `nmstate` so it will be installed
automatically on future `ansible-playbook site.yml` runs.

---

### [T7] `interfaces: unknown field 'macAddress' at line 6 column 1`

**Symptom** — After installing `nmstate`, the ISO generation still fails:

```
level=error msg=Provide file is not valid NetworkState or NetworkPolicy:
  interfaces: unknown field `macAddress` at line 6 column 1
level=fatal msg=  * failed to validate network yaml for host 0, failed to
  execute 'nmstatectl gc', error: Nmstate version: 2.2.57
```

**Cause** — `agent-config.yaml` uses `macAddress` (camelCase) inside the
`networkConfig.interfaces` block. That block is NMState YAML. NMState 2.x
requires the hyphenated form `mac-address`; camelCase was only accepted by
older nmstate 1.x.

Note the two different contexts in `agent-config.yaml`:

| Location | Field name | Why |
|---|---|---|
| `hosts[].interfaces[].macAddress` | `macAddress` | OCP `AgentConfig` API — matches the physical NIC to configure |
| `hosts[].networkConfig.interfaces[].mac-address` | `mac-address` | NMState 2.x format — sets the MAC in the network state |

**Fix** — In `ocp-config/agent-config.yaml.j2`, change `macAddress` to
`mac-address` inside every `networkConfig.interfaces` block only:

```yaml
# WRONG (nmstate 1.x / camelCase)
networkConfig:
  interfaces:
    - name: enp1s0
      macAddress: 52:54:00:c0:01:01   # ← fails with nmstate 2.x

# CORRECT (nmstate 2.x / hyphenated)
networkConfig:
  interfaces:
    - name: enp1s0
      mac-address: 52:54:00:c0:01:01  # ← accepted
```

The top-level `interfaces[].macAddress` entries (the OCP host-interface
matcher) keep the camelCase form — that is part of the OCP API, not NMState.

The template has been updated; re-run:
```bash
ansible-playbook generate-ocp-config.yml
```

---

### [T8] `installation-disk-speed-check` times out (exit code 124) on all hosts

**Symptom** — After nodes boot and reach `preparing-for-installation`, the
assisted-service repeatedly submits a disk speed check that times out after
480 seconds on every node. The service log shows:

```
exit-code <124> stderr <kill container: No such process ...>
error="unexpected end of JSON input"
```

The check runs:
```
fio --rw=write --ioengine=sync --size=22m --bs=2300 --fdatasync=1
```
This performs ~10,000 sequential writes each followed by an `fdatasync`. With
`cache='none'` (the VM default), every guest `fdatasync` flushes straight
through to the physical disk — and on a new qcow2 image, each write also
triggers a cluster allocation + L2 table update flush. The combined overhead
makes even a fast SSD appear to do <20 IOPS, far below OCP's 10 MB/s threshold.

`virsh update-device --live` cannot change cache mode on a running disk:
```
error: Operation not supported: cannot modify field 'cache' of the disk
```

**Why `writeback` is not enough** — `cache=writeback` was tried first but
also fails on a spinning HDD. With `writeback`, QEMU still calls the host
`fsync()` for every guest `fdatasync`, so the 480-second timeout is hit on
the second and subsequent test rounds (the first round passes because the
page cache is cold and fsyncs return quickly; by round 2, the HDD is busy
flushing dirty pages from round 1 and both rounds compete for the same disk).

**Fix** — Use `cache=unsafe`, which makes guest `fdatasync` a complete no-op
on the host. fio completes in seconds writing into the host page cache.

```bash
# 1. Destroy and undefine all VMs (qcow2 files are preserved)
for vm in ocp-control-1 ocp-control-2 ocp-control-3 ocp-worker-1 ocp-worker-2; do
  sudo virsh -c qemu:///system destroy $vm 2>/dev/null
  sudo virsh -c qemu:///system undefine $vm --nvram
done

# 2. Redefine with cache='unsafe' io='threads' and start
ansible-playbook site.yml --tags vms
```

The `vm.xml.j2` template now uses `cache='unsafe' io='threads'`.

**Trade-off** — Guest fsyncs are never flushed to the physical disk; a KVM
host crash during install could corrupt the qcow2 image. Acceptable for a
lab. Do NOT use for production VMs or any data you cannot afford to lose.

---

### [T9] `Error while evaluating DNS resolution on this host` during bootstrap

**Symptom** — After VMs boot and agents connect, DNS validation warnings appear
for all hosts and never clear:

```
WARNING Host ocp-control-1.ocp.lab.local validation: Error while evaluating DNS resolution on this host
```

NTP and majority-connectivity warnings that appear alongside this are **transient** —
they self-resolve within a few minutes as all nodes finish booting. The DNS
error is the real issue.

**Cause** — The libvirt dnsmasq DNS config was missing `api-int.ocp.lab.local`.
OCP nodes use `api-int` (the internal API endpoint) for intra-cluster
communication during installation. It must resolve to `api_vip`
(`192.168.200.10`) from within every node. Without it, the kubelet and cluster
operators cannot reach the API server and the install eventually stalls.

The `labnet.xml.j2` template had `api.ocp.lab.local` → 192.168.200.10 but
was missing the `api-int` alias on the same IP.

**Fix — inject into the live running network** (no VM or network restart needed):

```bash
sudo virsh -c qemu:///system net-update labnet add dns-host \
  '<host ip="192.168.200.10"><hostname>api-int.ocp.lab.local</hostname></host>' \
  --live --config
```

Verify:
```bash
dig api-int.ocp.lab.local @192.168.200.1 +short
# expected: 192.168.200.10
```

The `labnet.xml.j2` template now includes `api-int` so future runs do not
require this manual step.

---

### [T10] HAProxy pointing to wrong backend IPs; API/Ingress VIPs unreachable

**Symptom** — After bootstrap starts, `openshift-install agent wait-for
bootstrap-complete` cannot connect. The API and Ingress VIPs don't respond:

```bash
ping 192.168.200.10   # 100% packet loss
curl -k https://api.ocp.lab.local:6443   # connection refused / timeout
```

Inspecting `/etc/haproxy/haproxy.cfg` shows backends pointing to
`192.168.200.210–213` and `226–227` instead of the actual node IPs, and
the VIPs are not assigned to `virbr4`.

**Cause** — HAProxy was not managed by the Ansible project. A stale config
from a previous install used a different IP scheme. Additionally, the VIPs
(`192.168.200.10` and `192.168.200.11`) must be explicitly assigned to
`virbr4` on the KVM host so the host can respond to ARP for those addresses
and HAProxy can accept connections on them.

**Immediate fix** — Correct the backends and assign the VIPs:

```bash
# 1. Write correct HAProxy config
ansible-playbook site.yml --tags haproxy

# OR manually:
sudo systemctl restart haproxy

# 2. Assign VIPs to the bridge (if not yet present)
sudo ip addr add 192.168.200.10/32 dev virbr4
sudo ip addr add 192.168.200.11/32 dev virbr4

# 3. Verify
curl -k --connect-timeout 5 https://api.ocp.lab.local:6443/readyz
```

**Permanent fix** — The `haproxy` role now generates `/etc/haproxy/haproxy.cfg`
from a Jinja2 template using `control_nodes` and `worker_nodes` variables, and
assigns the VIPs to `virbr4` via `ip addr`. A NetworkManager dispatcher script
at `/etc/NetworkManager/dispatcher.d/99-ocp-vips` re-assigns the VIPs after
each reboot.

---

### Check VM console

```bash
virsh console ocp-control-1
# or
virt-viewer ocp-control-1
```

### Check agent bootstrap logs (from any control node)

```bash
ssh core@192.168.200.21
journalctl -u agent.service -f
```

### Root filesystem full

The root NVMe (`/dev/nvme0n1p6`, 29 GB btrfs) was at 95% before cleanup.
The `cleanup` role runs `dnf clean all`, `journalctl --vacuum-size=200M`, and
`docker system prune -af --volumes` to reclaim space.

If it fills again, check:

```bash
du -sh /var/lib/docker /var/lib/libvirt /var/log
sudo btrfs filesystem usage /
```

Consider moving Docker's data root to `/opt`:

```bash
# /etc/docker/daemon.json
{ "data-root": "/opt/docker" }
```

### libvirt network not starting

```bash
systemctl status virtnetworkd.socket
virsh net-list --all
virsh net-start labnet
```
