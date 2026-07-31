# Homelab Context

This document captures the shared understanding and key decisions for this homelab infrastructure.

## Goals

- **Primary drivers:** Disaster Recovery + Reproducibility
- **Vision:** Tear down and reprovision everything easily, anytime
- **Repo purpose:** Public, reproducible IaC that doubles as portfolio/resume

## Hardware

| Name | Hardware | OS | Role |
|------|----------|-----|------|
| **peggy** | RPi 4 4GB, 64GB SD, 1TB HDD on /var | Raspberry Pi OS Lite (64-bit) | K3s control plane + worker |
| **yelena** | RPi 4 8GB, 64GB SD, 1TB HDD on /var | Raspberry Pi OS Lite (64-bit) | K3s control plane + worker |
| **xialing** | HP Prodesk i5-8500, 16GB, 120GB SSD, 4TB USB HDD | Rocky Linux 10 | K3s control plane + worker, NAS, hardware transcoding |

### Network Equipment

- Omada TL-ER605 v1 (Router)
- Omada TL-SG2008P (PoE Switch)
- Omada TL-EAP225 (Access Point)
- Omada Software Controller (runs as K8s workload)

## Architecture Decisions

### ADR-001: Mixed OS (RPi OS + Rocky Linux)

**Status:** Accepted

**Context:** Need to choose OS for cluster nodes. Options were single OS everywhere or mixed.

**Decision:** Raspberry Pi OS Lite (64-bit) on ARM nodes (peggy, yelena), Rocky Linux 10 on x86 node (xialing).

**Rationale:**
- RPi OS has best hardware support for Raspberry Pi
- Rocky Linux provides RHEL practice (used at work)
- ARM64 support for Rocky on RPi is limited
- Ansible roles practice with multi-OS management

### ADR-002: All Nodes as K3s Control Plane

**Status:** Accepted

**Context:** K3s cluster topology — dedicated control plane vs shared.

**Decision:** All 3 nodes run as control plane + workers (HA embedded etcd).

**Rationale:**
- 3 nodes = quorum of 2, survives 1 node failure
- No wasted resources on dedicated control plane
- etcd on HDDs (/var) reduces SD card wear

### ADR-003: Containers Instead of VMs for HA/Omada

**Status:** Accepted

**Context:** Home Assistant and Omada Controller were previously VMs. Options: keep as VMs, move to K8s containers.

**Decision:** Run as K8s containers with Multus + macvlan networking.

**Rationale:**
- VMs have storage overhead (expanding disk images)
- xialing moving to smaller SSD (120GB)
- Containers are lighter, fit K8s GitOps model
- macvlan solves mDNS/multicast requirements

### ADR-004: Multus + Macvlan for LAN Access

**Status:** Accepted

**Context:** Home Assistant needs mDNS for device discovery. Omada needs L2 access for AP adoption.

**Decision:** Use Multus CNI with macvlan secondary interface. Pods get dual interfaces: eth0 (cluster) + net1 (LAN).

**Rationale:**
- mDNS/multicast requires direct LAN access
- Dual-interface preserves cluster connectivity
- Static IPs for infrastructure pods (HA, Omada)
- Macvlan-shim DaemonSet solves host-to-pod communication

### ADR-005: Flux for GitOps

**Status:** Accepted

**Context:** How to manage K8s workloads — Ansible, Flux, ArgoCD, or manual.

**Decision:** Hybrid — Ansible for node provisioning, Flux for K8s workloads.

**Rationale:**
- Separation of concerns (infra vs apps)
- Git as source of truth
- Auto-reconciliation corrects drift
- `kubectl apply` as emergency fallback

### ADR-006: Bitwarden Secrets Manager

**Status:** Accepted

**Context:** Repo is public. Secrets cannot be committed, even encrypted.

**Decision:** Bitwarden Secrets Manager Operator syncing secrets from Bitwarden SM. README documents required secrets for others.

**Rationale:**
- Public repo = no secrets in Git
- Already using Bitwarden
- Others can use their own secret store or create manually

### ADR-007: Physical Media Provisioning (Cloud-init + Kickstart)

**Status:** Accepted

**Context:** How to install OS on nodes for full automation.

**Decision:** Physical media with automated first-boot config. Different mechanisms per hardware:
- **RPi (peggy, yelena):** SD card with Raspberry Pi OS + cloud-init
- **x86 (xialing):** USB installer with Rocky Linux + Kickstart (internal SSD)

**Rationale:**
- Simpler than PXE (no DHCP proxy, no boot server)
- RPi: cloud-init embedded in boot partition works well
- x86: internal SSD can't be flashed from desktop; USB installer + Kickstart automates install
- Both achieve same outcome: boot → Ansible-ready

### ADR-008: Hybrid Upgrade Strategy

**Status:** Accepted

**Context:** How to handle OS upgrades (patches vs major versions).

**Decision:** In-place patches via Ansible. Reprovision for major version upgrades.

**Rationale:**
- Patches are safe to apply in-place
- Major upgrades risk drift — reprovision ensures clean state
- If automation is solid, reprovision is fast

### ADR-009: DNS Architecture (dnsdist + Technitium)

**Status:** Accepted

**Context:** Need a DNS solution for the homelab that provides:
- Ad-blocking / DNS filtering
- Authoritative DNS for 2 owned domains
- High availability across all 3 nodes
- Per-client visibility (no SNAT)
- GitOps-friendly configuration
- Metrics export to Prometheus/Grafana

Previous setup used PiHole with MetalLB load balancing. Evaluating alternatives for the new macvlan-based architecture.

**Decision:** Two-tier DNS with dnsdist (load balancer) + Technitium (DNS server), all on macvlan.

Architecture:
```
Clients (2 DNS servers configured)
    │
    ▼
dnsdist x2 (macvlan IPs: TBD) ─── VIPs for clients
    │
    ▼ round-robin
Technitium x3 (macvlan IPs: TBD) ─── one per node
```

**Rationale:**

*Why not PiHole:*
- Stateful (SQLite DB) — harder to manage in K8s
- No authoritative DNS support for owned domains

*Why Technitium over Blocky:*
- Blocky is forwarding-only; can't serve as authoritative for owned domains
- Technitium handles both recursive resolution + authoritative zones
- Full HTTP API enables GitOps (no dependency on web UI)
- Prometheus metrics via exporter

*Why dnsdist over MetalLB:*
- MetalLB L2 mode is failover-only (1 active node), not true load balancing
- Most clients only accept 2 DNS servers, but we have 3 Technitium instances
- dnsdist provides real round-robin across all 3 backends per VIP
- dnsdist preserves client source IP natively (no SNAT)
- Purpose-built for DNS — health checks via actual DNS queries

*Why macvlan for everything:*
- Per-client visibility requires no SNAT — macvlan delivers this
- Direct L2 path: client → dnsdist → Technitium (no kube-proxy)
- Stable IPs (manually assigned) vs ephemeral pod IPs
- Health checks validate the actual serving interface

*Why 2 dnsdist + 3 Technitium:*
- 2 VIPs match typical client DNS server limit
- 3 backends for prod simulation (all nodes participate)
- Each VIP load balances to all 3 backends

**Trade-offs accepted:**
- Extra component (dnsdist) vs simpler MetalLB
- Manual IP management for macvlan interfaces
- dnsdist config is Lua (less familiar than YAML)

## Five-Step Provisioning Model

| Step | Name | Tool | Command |
|------|------|------|---------|
| 1 | **Provision** | cloud-init / Kickstart | Flash media, boot node |
| 2 | **Bootstrap** | Ansible | `ansible-playbook playbooks/bootstrap.yaml` |
| 3 | **K3s** | Ansible | `ansible-playbook playbooks/k3s.yaml` |
| 4 | **Flux** | Ansible | `ansible-playbook playbooks/flux.yaml` |
| 5 | **Services** | Flux | Automatic (GitOps reconciles `manifests/`) |

### Step Details

**Step 1 — Provision:** OS installation via physical media
- RPi: SD card with Raspberry Pi OS + cloud-init
- x86: USB installer with Rocky Linux + Kickstart
- Output: Nodes boot with SSH access, ready for Ansible

**Step 2 — Bootstrap:** Package installation and OS prerequisites
- Updates system packages
- Installs common tools (htop, vim, etc.)
- Configures kernel modules and sysctl for K8s
- Configures NFS/SMB on xialing
- Output: Nodes ready for K3s installation

**Step 3 — K3s:** Kubernetes cluster bootstrap
- Installs K3s in HA mode (embedded etcd)
- xialing: `--cluster-init` (first server)
- peggy, yelena: join via xialing
- Output: 3-node K3s cluster with kubeconfig

**Step 4 — Flux:** GitOps bootstrap
- Installs Flux on the cluster
- Configures GitHub source and kustomization
- Output: Flux watching `manifests/` directory

**Step 5 — Services:** Application deployment (automatic)
- Flux reconciles infrastructure (Longhorn, Bitwarden SM Operator, Multus)
- Flux reconciles apps (media stack, networking, home automation)
- Output: All workloads running

## K3s Prerequisites

Platform-specific requirements discovered during cluster bootstrap:

### Rocky Linux (xialing)

- **firewalld:** Disabled (conflicts with K3s iptables rules)
- **kernel-modules-extra:** Required for `br_netfilter` module. Must match running kernel version — if repos don't have the matching version, boot into an older kernel that has it.
- **Kernel modules:** `br_netfilter`, `overlay`
- **Sysctl:** `net.bridge.bridge-nf-call-iptables=1`, `net.bridge.bridge-nf-call-ip6tables=1`, `net.ipv4.ip_forward=1`

### Raspberry Pi OS (peggy, yelena)

- **cgroup memory:** Add `cgroup_memory=1 cgroup_enable=memory` to `/boot/firmware/cmdline.txt` (requires reboot)
- **iptables-persistent:** Required for iptables-save/restore tools
- **Kernel modules:** `br_netfilter`, `overlay`
- **Sysctl:** Same as Rocky Linux

## Backup & Restore

| Data | Backup Method | Location |
|------|---------------|----------|
| App configs (PVCs) | Longhorn backups | NAS (4TB HDD via NFS) |
| App-native backups | radarr/sonarr/HA exports | NAS |
| Media files | Already on NAS | 4TB USB HDD |

### Restore Process

1. Install Longhorn on fresh cluster
2. Configure backup target (same NFS path)
3. Longhorn auto-discovers backups
4. Restore volumes from backup
5. Create PVCs pointing to restored volumes

## Networking

### Subnets

- **LAN:** 10.0.0.0/24
- **IoT VLAN:** 10.0.30.0/24 (for Home Assistant device discovery)
- **K8s Pod Network:** 10.42.0.0/16 (Flannel)
- **K8s Service Network:** 10.43.0.0/16

### Static IPs (Infrastructure)

| Service | IP |
|---------|-----|
| Home Assistant | TBD (IoT VLAN) |
| Omada Controller | TBD (LAN) |
| dnsdist (VIP 1) | TBD |
| dnsdist (VIP 2) | TBD |
| Technitium (peggy) | TBD |
| Technitium (yelena) | TBD |
| Technitium (xialing) | TBD |

## Services

### Current (migrating from homelab-ansible)

- **Media:** Jellyfin, Radarr, Sonarr, Bazarr, Prowlarr, Transmission, Kavita, Mylar3
- **Network:** dnsdist, Technitium, Tailscale, Omada Controller
- **Home Automation:** Home Assistant
- **Storage:** Longhorn, NFS from xialing
- **Monitoring:** Prometheus + Grafana (deferred)

## Repository Structure

```
homelab/
├── ansible/
│   ├── inventory/
│   │   └── hosts.yaml           # Static inventory (rpi, x86, k3s_cluster groups)
│   ├── playbooks/
│   │   ├── bootstrap.yaml       # Step 2: packages + prerequisites
│   │   ├── k3s.yaml             # Step 3: K3s cluster setup
│   │   └── flux.yaml            # Step 4: Flux GitOps bootstrap
│   ├── provisioning/            # Step 1: OS install configs
│   │   └── templates/           # cloud-init and kickstart templates
│   └── roles/
│       ├── common/              # Shared tasks (packages, sysctl, modules)
│       ├── nfs_smb/             # NFS/SMB server (xialing only)
│       └── prereqs/             # OS-specific K8s prerequisites
├── manifests/                   # Step 5: K8s workloads (Flux watches)
│   ├── flux/                    # Flux bootstrap files (GitRepository, Kustomization)
│   ├── infrastructure/
│   │   ├── controllers/         # Longhorn, Bitwarden SM, Multus
│   │   └── configs/             # NADs, ClusterSecretStore
│   └── apps/
│       ├── media/               # Jellyfin, *arr stack
│       ├── network/             # dnsdist, Technitium, Tailscale, Omada
│       └── home/                # Home Assistant
├── docs/
│   ├── restore-guide.md
│   └── secrets.md
├── CONTEXT.md                   # This file
└── README.md
```

## Glossary

| Term | Definition |
|------|------------|
| **peggy** | Raspberry Pi 4 4GB node |
| **yelena** | Raspberry Pi 4 8GB node |
| **xialing** | HP Prodesk 600 G4 node |
| **macvlan-shim** | Host interface enabling node-to-macvlan-pod communication |
| **LAN-attached pod** | A pod with a macvlan secondary interface for direct LAN access (e.g., Home Assistant, Omada Controller) |

## Conventions

- **YAML files:** Use `.yaml` extension (not `.yml`)
- **Configuration split:**
  - `ansible/vars.yaml` — committed, contains IPs, usernames, public config
  - `ansible/secrets.local.yaml` — gitignored, contains tokens only (copy from `secrets.local.example.yaml`)
  - Pass both to playbooks: `-e @vars.yaml -e @secrets.local.yaml`
- **DRY config:** Node IPs are defined in two places that must stay in sync:
  - `ansible/vars.yaml` — source of truth for Ansible
  - `manifests/infrastructure/controllers/cluster-vars.yaml` — ConfigMap for Flux substitution
  - K8s manifests use `${XIALING_IP}` etc., substituted by Flux `postBuild`
