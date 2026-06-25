# Lab Architecture & Roadmap

## Final design: compact **3C/W** cluster

The cluster runs as a **compact 3-node-schedulable** topology: the 3 control-plane
nodes are *also* schedulable for workloads (`mastersSchedulable: true`), plus a
worker. This is the permanent design.

### Why 3C/W instead of 3C + 1W

With the default **tainted** control plane, control-node RAM headroom is unusable —
workloads can't land there. Everything piles onto the single worker until it fills
(seen: worker at 99% memory, a 256Mi CirrOS VM couldn't schedule), while the 3
control nodes sat at 55–69% with stranded capacity. Making masters schedulable
**pools** the headroom across all nodes, so VMs draw from wherever there's room.

This is set cluster-wide:
```bash
oc patch schedulers.config.openshift.io cluster --type merge \
  -p '{"spec":{"mastersSchedulable":true}}'
```
It is applied automatically as a post-install step in `eject-iso.yml` so redeploys
keep the 3C/W design (see `make_masters_schedulable` / `mastersSchedulable` task).

> RHOSO and OCP-Virt are **mutually exclusive** — only one runs at a time.

---

## Current state (Host A only — temporary)

**Host A:** 62 GiB RAM, i7-10710U (12 threads), VM disks on `/opt` (`/dev/sda1`, 733 GB).

| Role | Count | RAM (live) | Notes |
|------|-------|-----------|-------|
| control + worker (schedulable) | 3 | ~12 GiB | ballooned down from 14 to fit CNV |
| worker | 1 | 10 GiB | |

This is at the host's limit. CNV installs and a small VM runs, but the cluster
churns under controller load. **Config drift:** live RAM (controls 12 GiB) differs
from `group_vars/all.yml` (controls 14, workers 10); a full redeploy resets to the
committed values.

---

## Roadmap: add Host B (≈ 2026-07-02)

**Goal:** keep 3C/W on Host A, move the dedicated worker to **Host B** with bigger
RAM + storage for VM/CDI disks. This frees Host A to size its 3 C/W nodes up.

**Host B:** 32 GiB RAM (+ larger disk for VM storage).

### Target layout

| Host | Nodes | Per-node | Total | Headroom |
|------|-------|----------|-------|----------|
| A (62 GiB) | 3 × control+worker | ~18 GiB | 54 GiB | ~8 GiB |
| B (32 GiB) | 1 × worker | ~24 GiB | 24 GiB | ~6 GiB |

Host A's current `ocp-worker-1` is **decommissioned**; the worker role lives on B.

### Steps (do NOT run until Host B is ready)

1. **Networking — span both hosts (the main new work).**
   Today `labnet` (virbr4 / 192.168.200.0/24) is a host-local NAT bridge and can't
   reach Host B. Switch to shared L2:
   - Create a Linux bridge (`br0`) on **each** host enslaving its physical NIC.
   - Attach OCP node VMs to `br0` instead of the NAT network.
   - Keep one DNS (the BIND on Host A) reachable from both hosts' VMs; it serves
     `api`, `api-int`, `*.apps`.
   - API + Ingress VIPs must live in the shared subnet.
   - This is the one piece needing new Ansible (a "bridged" network mode in the
     `networks` role + moving the VIP/DNS subnet onto the LAN).

2. **Resize Host A control nodes** 12/14 → ~18 GiB once worker-1 is removed
   (update `group_vars/all.yml`: `control_vm_ram_mb: 18432`).

3. **Add the Host B worker as a day-2 node.**
   - Define the worker VM on Host B (larger `ram_mb` and a bigger `disk_gb`, e.g.
     200+ GB for VM/CDI images).
   - Generate an add-node ISO and boot it; approve the join CSRs.
   - Label it (e.g. `node-role.kubernetes.io/worker`) and optionally taint it as a
     dedicated virtualization host so CNV VMs prefer it.

4. **Storage for VMs on Host B.**
   - Use the larger Host B disk for a `hostpath-provisioner` (HPP) storage class or
     a dedicated CDI scratch/storage location, so VM disks land on B, not A.

5. **Decommission Host A `ocp-worker-1`** (`oc delete node`, stop/undefine the VM),
   then apply step 2's resize.

### Operating model (temporary / on-demand)

- **Stop/start VMs** for short idle gaps — but after long downtime expect the
  cert-expiry + CSR-approval cold-start recovery (see the cold-start runbook in
  README troubleshooting).
- **Destroy + reinstall** (cleanup role + agent-based install, ~2–3 h) is cleaner if
  it sits idle for weeks. Preferred for a genuinely temporary lab.

### CNV install/teardown

See `ocp-virt.yaml` / `hyperconverged.yaml`. Always **free RAM before installing**
(stop a worker / balloon controls) — installing onto a full cluster thrashes the
host into an unreachable API. Validated teardown order: subscription → CSV →
webhookconfigurations → CR finalizers → stale APIServices → namespace →
force-delete stuck pods → CRDs.
