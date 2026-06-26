# Manual OCP Install (without the Ansible scripts)

Step-by-step agent-based OpenShift install on a single Fedora KVM host, using the
raw `virsh` / `openshift-install` / `oc` commands that the Ansible roles automate.
Values below match this lab (`group_vars/all.yml`) — adjust if you change them.

> Run everything as a user with sudo. Commands that touch libvirt use the
> **system** connection (`qemu:///system`).

## Reference values

| Item | Value |
|------|-------|
| Cluster / domain | `ocp` / `lab.local` |
| OCP version | `4.18` (stream; actual build may be newer) |
| Node network | `labnet`, bridge `virbr4`, `192.168.200.0/24`, gw `192.168.200.1` |
| API VIP | `192.168.200.10` → `api.ocp.lab.local` |
| Ingress VIP | `192.168.200.11` → `*.apps.ocp.lab.local` |
| Storage pool | `ocp-images` at `/opt/images` |
| Install dir | `/opt/ocp/cluster` |
| Pull secret | `/opt/ocp/pull-secret.json` |
| Redfish (sushy) | `http://192.168.200.1:8000` |

| Node | Role | IP | MAC (NIC1) |
|------|------|----|-----------|
| ocp-control-1 | master | 192.168.200.21 | 52:54:00:c0:01:01 |
| ocp-control-2 | master | 192.168.200.22 | 52:54:00:c0:01:02 |
| ocp-control-3 | master | 192.168.200.23 | 52:54:00:c0:01:03 |
| ocp-worker-1  | worker | 192.168.200.31 | 52:54:00:c0:02:01 |
| ocp-worker-2  | worker | 192.168.200.32 | 52:54:00:c0:02:02 |

---

## 0. Prerequisites

- Host: ≥48 GB RAM, ≥500 GB free on `/opt`, CPU with `vmx`/`svm`.
- Red Hat pull secret saved to `/opt/ocp/pull-secret.json`
  (from <https://console.redhat.com/openshift/install/pull-secret>).
- An SSH keypair: `ssh-keygen -t rsa -b 4096 -N '' -f ~/.ssh/id_rsa` (if absent).

```bash
egrep -c '(vmx|svm)' /proc/cpuinfo      # must be > 0
sudo mkdir -p /opt/ocp /opt/images
```

---

## 1. Packages + OCP binaries

```bash
sudo dnf install -y \
  qemu-kvm qemu-img libvirt libvirt-client \
  libvirt-daemon-driver-qemu libvirt-daemon-driver-network \
  libvirt-daemon-driver-storage-core libvirt-daemon-driver-storage-logical \
  virt-install virt-viewer python3-libvirt python3-lxml \
  dnsmasq bind bind-utils nmstate haproxy \
  python3-virtualenv jq wget curl tmux

VER=4.18
cd /tmp
wget -q https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable-$VER/openshift-install-linux.tar.gz
wget -q https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable-$VER/openshift-client-linux.tar.gz
sudo tar -xzf openshift-install-linux.tar.gz -C /usr/local/bin openshift-install
sudo tar -xzf openshift-client-linux.tar.gz  -C /usr/local/bin oc kubectl
sudo chmod 755 /usr/local/bin/{openshift-install,oc,kubectl}
```

---

## 2. Host / libvirt prep

```bash
# Modular libvirt daemons (Fedora uses these, not monolithic libvirtd)
for s in virtqemud virtnetworkd virtstoraged virtnodedevd virtsecretd virtlogd; do
  sudo systemctl enable --now ${s}.socket
done

# qemu.conf — permissive for a lab
sudo sed -i 's/^#\?\s*security_driver\s*=.*/security_driver = "none"/'  /etc/libvirt/qemu.conf
sudo sed -i 's/^#\?\s*user\s*=.*/user = "root"/'                        /etc/libvirt/qemu.conf
sudo sed -i 's/^#\?\s*group\s*=.*/group = "kvm"/'                       /etc/libvirt/qemu.conf
sudo sed -i 's/^#\?\s*dynamic_ownership\s*=.*/dynamic_ownership = 1/'   /etc/libvirt/qemu.conf
sudo systemctl restart virtqemud

# Groups
sudo usermod -aG libvirt,kvm "$USER"   # log out/in for this to take effect

# Nested virtualization + IP forwarding
sudo modprobe kvm_intel nested=1
echo 'options kvm_intel nested=1' | sudo tee /etc/modprobe.d/kvm-nested.conf
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-kvm-ip-forward.conf
sudo sysctl -w net.ipv4.ip_forward=1

# Keep NetworkManager off libvirt bridges/taps (else it grabs :53 on virbr4)
sudo tee /etc/NetworkManager/conf.d/99-libvirt.conf >/dev/null <<'EOF'
[keyfile]
unmanaged-devices=interface-name:virbr*;interface-name:vnet*
EOF
sudo nmcli general reload conf

# Firewall: trust libvirt zone
sudo firewall-cmd --zone=libvirt --set-target=ACCEPT --permanent
sudo firewall-cmd --reload
```

Optional 32 GB swap on `/opt` (helps a RAM-tight host):
```bash
sudo dd if=/dev/zero of=/opt/swapfile bs=1M count=32768
sudo chmod 600 /opt/swapfile && sudo mkswap /opt/swapfile
echo '/opt/swapfile swap swap defaults 0 0' | sudo tee -a /etc/fstab
sudo swapon /opt/swapfile
```

---

## 3. Node network (`labnet` / virbr4)

`/tmp/labnet.xml`:
```xml
<network xmlns:dnsmasq='http://libvirt.org/schemas/network/dnsmasq/1.0'>
  <name>labnet</name>
  <forward mode='nat'><nat><port start='1024' end='65535'/></nat></forward>
  <bridge name='virbr4' stp='on' delay='0'/>
  <dns enable='no'/>                     <!-- BIND on the host owns DNS -->
  <ip address='192.168.200.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.200.100' end='192.168.200.200'/>
      <host mac='52:54:00:c0:01:01' name='ocp-control-1' ip='192.168.200.21'/>
      <host mac='52:54:00:c0:01:02' name='ocp-control-2' ip='192.168.200.22'/>
      <host mac='52:54:00:c0:01:03' name='ocp-control-3' ip='192.168.200.23'/>
      <host mac='52:54:00:c0:02:01' name='ocp-worker-1' ip='192.168.200.31'/>
      <host mac='52:54:00:c0:02:02' name='ocp-worker-2' ip='192.168.200.32'/>
    </dhcp>
  </ip>
  <dnsmasq:options>
    <dnsmasq:option value='dhcp-option=option:dns-server,192.168.200.1'/>
  </dnsmasq:options>
</network>
```
```bash
sudo virsh -c qemu:///system net-define /tmp/labnet.xml
sudo virsh -c qemu:///system net-autostart labnet
sudo virsh -c qemu:///system net-start labnet
```

---

## 4. DNS (BIND)

BIND owns `192.168.200.1:53` (VM queries) and `127.0.0.1:5353` (host forward).
NetworkManager's dnsmasq keeps `127.0.0.1:53` and forwards the cluster domain to BIND.

`/etc/named.conf`:
```
options {
    listen-on port 53 { 192.168.200.1; };
    listen-on port 5353 { 127.0.0.1; };
    listen-on-v6 { none; };
    directory "/var/named";
    allow-query     { 192.168.200.0/24; 127.0.0.0/8; };
    allow-recursion { 192.168.200.0/24; 127.0.0.0/8; };
    recursion yes;
    forwarders { 8.8.8.8; 8.8.4.4; };
    forward only;
    dnssec-validation no;
};
zone "ocp.lab.local" IN {
    type primary;
    file "/var/named/ocp.lab.local.zone";
    allow-update { none; };
};
```

`/var/named/ocp.lab.local.zone`:
```
$TTL 3600
@   IN SOA  ns1.ocp.lab.local. admin.ocp.lab.local. ( 2026052301 3600 900 604800 300 )
@           IN NS   ns1
ns1         IN A    192.168.200.1
api         IN A    192.168.200.10
api-int     IN A    192.168.200.10
*.apps      IN A    192.168.200.11
ocp-control-1 IN A  192.168.200.21
ocp-control-2 IN A  192.168.200.22
ocp-control-3 IN A  192.168.200.23
ocp-worker-1  IN A  192.168.200.31
ocp-worker-2  IN A  192.168.200.32
```

Host resolver forwarding — `/etc/NetworkManager/conf.d/01-dnsmasq.conf`:
```
[main]
dns=dnsmasq
```
`/etc/NetworkManager/dnsmasq.d/ocp.conf`:
```
server=/ocp.lab.local/127.0.0.1#5353
```
```bash
sudo chown root:named /etc/named.conf /var/named/ocp.lab.local.zone
sudo firewall-cmd --zone=libvirt --add-service=dns --permanent && sudo firewall-cmd --reload
sudo systemctl enable --now named
sudo systemctl reload NetworkManager
# Verify
dig +short api.ocp.lab.local        # → 192.168.200.10
dig +short test.apps.ocp.lab.local  # → 192.168.200.11
```

> BIND can only bind `192.168.200.1:53` **after** virbr4 exists. The VIP hook in
> step 5 restarts named when labnet starts, which handles reboot ordering.

---

## 5. VIPs + libvirt network hook

The hook binds both VIPs to virbr4 whenever labnet starts (and restarts BIND).

`/etc/libvirt/hooks/network` (chmod 755):
```bash
#!/bin/bash
NETWORK=$1; ACTION=$2
[[ "$NETWORK" != "labnet" ]] && exit 0
case "$ACTION" in
  started)
    ip addr add 192.168.200.10/32 dev virbr4 2>/dev/null || true
    ip addr add 192.168.200.11/32 dev virbr4 2>/dev/null || true
    sleep 1; systemctl restart named 2>/dev/null || true ;;
  stopped)
    ip addr del 192.168.200.10/32 dev virbr4 2>/dev/null || true
    ip addr del 192.168.200.11/32 dev virbr4 2>/dev/null || true ;;
esac
exit 0
```
```bash
# Modular virtnetworkd needs this SELinux boolean to run hook scripts
sudo setsebool -P virt_hooks_unconfined on
sudo chmod 755 /etc/libvirt/hooks/network
# Bind now (labnet already running)
sudo ip addr add 192.168.200.10/32 dev virbr4
sudo ip addr add 192.168.200.11/32 dev virbr4
```

---

## 6. HAProxy (API + ingress load balancer)

`/etc/haproxy/haproxy.cfg`:
```
global
    log stdout format raw local0
    maxconn 20000
defaults
    log global
    mode tcp
    option tcplog
    timeout connect 10s
    timeout client 1m
    timeout server 1m

frontend api
    bind 192.168.200.10:6443
    default_backend api-backend
backend api-backend
    balance roundrobin
    server ocp-control-1 192.168.200.21:6443 check
    server ocp-control-2 192.168.200.22:6443 check
    server ocp-control-3 192.168.200.23:6443 check

frontend machine-config
    bind 192.168.200.10:22623
    default_backend mc-backend
backend mc-backend
    balance roundrobin
    server ocp-control-1 192.168.200.21:22623 check
    server ocp-control-2 192.168.200.22:22623 check
    server ocp-control-3 192.168.200.23:22623 check

frontend ingress-http
    bind 192.168.200.11:80
    default_backend ingress-http-backend
backend ingress-http-backend
    balance roundrobin
    server ocp-worker-1 192.168.200.31:80 check
    server ocp-worker-2 192.168.200.32:80 check

frontend ingress-https
    bind 192.168.200.11:443
    default_backend ingress-https-backend
backend ingress-https-backend
    balance roundrobin
    server ocp-worker-1 192.168.200.31:443 check
    server ocp-worker-2 192.168.200.32:443 check
```
```bash
for p in 6443 22623 80 443; do
  sudo firewall-cmd --zone=libvirt --add-port=${p}/tcp --permanent
done
sudo firewall-cmd --reload
sudo systemctl enable --now haproxy
```

---

## 7. sushy-emulator (Redfish for virtual media)

`openshift-install` talks Redfish to insert/eject the agent ISO on each VM.

```bash
sudo python3 -m venv --system-site-packages /opt/sushy-venv
sudo /opt/sushy-venv/bin/pip install sushy-tools
sudo mkdir -p /etc/sushy
sudo tee /etc/sushy/sushy.conf >/dev/null <<'EOF'
SUSHY_EMULATOR_LISTEN_IP = '0.0.0.0'
SUSHY_EMULATOR_LISTEN_PORT = 8000
SUSHY_EMULATOR_SSL_CERT = None
SUSHY_EMULATOR_SSL_KEY = None
SUSHY_EMULATOR_AUTH_FILE = None
SUSHY_EMULATOR_LIBVIRT_URI = 'qemu:///system'
SUSHY_EMULATOR_IGNORE_BOOT_DEVICE = True
SUSHY_EMULATOR_BOOT_LOADER_MAP = {}
EOF
sudo tee /etc/systemd/system/sushy-emulator.service >/dev/null <<'EOF'
[Unit]
Description=Sushy Redfish Emulator (libvirt backend)
After=virtqemud.socket network.target
Wants=virtqemud.socket
[Service]
Type=simple
User=root
ExecStart=/opt/sushy-venv/bin/sushy-emulator --config /etc/sushy/sushy.conf
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
EOF
sudo firewall-cmd --zone=libvirt --add-port=8000/tcp --permanent && sudo firewall-cmd --reload
sudo systemctl daemon-reload && sudo systemctl enable --now sushy-emulator
# Verify (VM names appear once VMs are defined in step 9)
curl -s http://192.168.200.1:8000/redfish/v1/Systems/ | jq .
```

---

## 8. Storage pool

```bash
sudo virsh -c qemu:///system pool-define-as ocp-images dir --target /opt/images
sudo virsh -c qemu:///system pool-build ocp-images
sudo virsh -c qemu:///system pool-autostart ocp-images
sudo virsh -c qemu:///system pool-start ocp-images
```

---

## 9. VM disks + domain definitions

Create a qcow2 and define a domain for each node. The domain XML is a Q35/UEFI
machine with **boot disk = order 1, ISO = order 2** (so it boots the ISO only
while the disk is empty — no manual eject needed). Loop over all five nodes:

```bash
# name ram_mb vcpus disk_gb mac
NODES=(
  "ocp-control-1 14336 4 120 52:54:00:c0:01:01"
  "ocp-control-2 14336 4 120 52:54:00:c0:01:02"
  "ocp-control-3 14336 4 120 52:54:00:c0:01:03"
  "ocp-worker-1  12288 4 120 52:54:00:c0:02:01"
  "ocp-worker-2  12288 4 120 52:54:00:c0:02:02"
)
for n in "${NODES[@]}"; do
  set -- $n; NAME=$1; RAM=$2; CPU=$3; DISK=$4; MAC=$5
  sudo qemu-img create -f qcow2 /opt/images/${NAME}.qcow2 ${DISK}G
  sudo chown root:kvm /opt/images/${NAME}.qcow2 && sudo chmod 600 /opt/images/${NAME}.qcow2
  cat >/tmp/${NAME}.xml <<EOF
<domain type='kvm'>
  <name>${NAME}</name>
  <memory unit='MiB'>${RAM}</memory>
  <vcpu placement='static'>${CPU}</vcpu>
  <os firmware='efi'>
    <type arch='x86_64' machine='q35'>hvm</type>
    <firmware><feature enabled='no' name='secure-boot'/></firmware>
    <bootmenu enable='yes' timeout='3000'/>
  </os>
  <features><acpi/><apic/></features>
  <cpu mode='host-passthrough' check='none' migratable='on'>
    <topology sockets='1' dies='1' cores='${CPU}' threads='1'/>
  </cpu>
  <clock offset='utc'/>
  <on_poweroff>destroy</on_poweroff><on_reboot>restart</on_reboot><on_crash>destroy</on_crash>
  <devices>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2' discard='unmap' cache='unsafe' io='threads'/>
      <source file='/opt/images/${NAME}.qcow2'/>
      <target dev='vda' bus='virtio'/><boot order='1'/>
    </disk>
    <disk type='file' device='cdrom'>
      <driver name='qemu' type='raw'/>
      <source file='/opt/ocp/cluster/agent.x86_64.iso'/>
      <target dev='sda' bus='sata'/><readonly/><boot order='2'/>
    </disk>
    <interface type='network'>
      <mac address='${MAC}'/><source network='labnet'/><model type='virtio'/>
    </interface>
    <serial type='pty'><target type='isa-serial' port='0'/></serial>
    <console type='pty'><target type='serial' port='0'/></console>
    <graphics type='vnc' port='-1' autoport='yes' listen='127.0.0.1'/>
    <video><model type='virtio' heads='1' primary='yes'/></video>
    <rng model='virtio'><backend model='random'>/dev/urandom</backend></rng>
    <memballoon model='virtio'/>
  </devices>
</domain>
EOF
  sudo virsh -c qemu:///system define /tmp/${NAME}.xml
done
```

> Do **not** start the VMs yet — the agent ISO referenced above doesn't exist
> until step 11. (RHOSO needs two extra NICs per VM; see `roles/vms/templates/vm.xml.j2`.)

---

## 10. install-config.yaml + agent-config.yaml

```bash
mkdir -p /opt/ocp/cluster
```

`/opt/ocp/cluster/install-config.yaml` (insert your pull secret + SSH key):
```yaml
apiVersion: v1
baseDomain: lab.local
metadata:
  name: ocp
networking:
  networkType: OVNKubernetes
  clusterNetwork:
    - cidr: 10.128.0.0/14
      hostPrefix: 23
  machineNetwork:
    - cidr: 192.168.200.0/24
  serviceNetwork:
    - 172.30.0.0/16
compute:
  - name: worker
    replicas: 2
controlPlane:
  name: master
  replicas: 3
  hyperthreading: Enabled
platform:
  none: {}
pullSecret: '<contents of /opt/ocp/pull-secret.json on one line>'
sshKey: '<contents of ~/.ssh/id_rsa.pub>'
```

`/opt/ocp/cluster/agent-config.yaml` — one `hosts:` block per node. The Redfish
`bmc.address` points at sushy; `networkConfig` pins each node's static IP:
```yaml
apiVersion: v1alpha1
kind: AgentConfig
metadata:
  name: ocp
rendezvousIP: 192.168.200.21
hosts:
  - hostname: ocp-control-1.ocp.lab.local
    role: master
    bmc:
      address: redfish-virtualmedia://192.168.200.1:8000/redfish/v1/Systems/ocp-control-1
      username: admin
      password: admin
      disableCertificateVerification: true
    interfaces:
      - name: enp1s0
        macAddress: 52:54:00:c0:01:01
    networkConfig:
      interfaces:
        - name: enp1s0
          type: ethernet
          state: up
          mac-address: 52:54:00:c0:01:01
          ipv4:
            enabled: true
            address: [{ ip: 192.168.200.21, prefix-length: 24 }]
            dhcp: false
      dns-resolver:
        config:
          server: [192.168.200.1]
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 192.168.200.1
            next-hop-interface: enp1s0
# ... repeat for control-2/3 (.22/.23) and worker-1/2 (.31/.32, role: worker) ...
```

> `openshift-install` **consumes** (deletes) these two files when it builds the
> ISO. Keep copies outside the install dir if you want to rerun.

---

## 11. Generate the agent ISO

```bash
openshift-install agent create image --dir /opt/ocp/cluster --log-level info
ls -lh /opt/ocp/cluster/agent.x86_64.iso     # ~1 GB
```

---

## 12. Boot the cluster

```bash
for vm in ocp-control-1 ocp-control-2 ocp-control-3 ocp-worker-1 ocp-worker-2; do
  sudo virsh -c qemu:///system start $vm
done
```
Each VM boots the ISO (disk empty → falls through to order 2), writes RHCOS to
`vda`, and reboots into the disk. No ISO eject needed.

---

## 13. Wait for completion

```bash
# Phase 1: control plane forms (~30–45 min)
openshift-install agent wait-for bootstrap-complete --dir /opt/ocp/cluster --log-level info

# Phase 2: all operators converge (up to ~3 h on slow hardware)
openshift-install agent wait-for install-complete --dir /opt/ocp/cluster --log-level info
```
If `install-complete` times out, the cluster is usually still converging — check
with `oc get co` rather than assuming failure.

---

## 14. Post-install

```bash
sudo chown -R "$USER":"$USER" /opt/ocp/cluster
echo "export KUBECONFIG=/opt/ocp/cluster/auth/kubeconfig" >> ~/.bashrc
export KUBECONFIG=/opt/ocp/cluster/auth/kubeconfig

# Compact 3C/W: make control plane schedulable (see ARCHITECTURE.md)
oc patch schedulers.config.openshift.io cluster --type merge \
  -p '{"spec":{"mastersSchedulable":true}}'

oc get nodes
oc get clusteroperators
cat /opt/ocp/cluster/auth/kubeadmin-password
# Console: https://console-openshift-console.apps.ocp.lab.local
```

---

## Reboot ordering (cold start)

On host reboot: `labnet` autostarts → the network hook rebinds the VIPs and
restarts BIND → HAProxy and sushy come up via systemd. After a **long** shutdown,
expect expired certs / pending CSRs — see the cold-start recovery notes in
`README.md` troubleshooting. Approve all pending CSRs:
```bash
oc get csr -o name | xargs oc adm certificate approve
```
