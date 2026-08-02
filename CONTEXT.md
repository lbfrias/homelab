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

Architecture Decision Records are in [`.scratch/adr/`](.scratch/adr/):

| ADR | Title |
|-----|-------|
| [0001](.scratch/adr/0001-mixed-os.md) | Mixed OS (RPi OS + Rocky Linux) |
| [0002](.scratch/adr/0002-all-nodes-control-plane.md) | All Nodes as K3s Control Plane |
| [0003](.scratch/adr/0003-containers-not-vms.md) | Containers Instead of VMs for HA/Omada |
| [0004](.scratch/adr/0004-multus-macvlan.md) | Multus + Macvlan for LAN Access |
| [0005](.scratch/adr/0005-flux-gitops.md) | Flux for GitOps |
| [0006](.scratch/adr/0006-bitwarden-secrets.md) | Bitwarden Secrets Manager |
| [0007](.scratch/adr/0007-physical-media-provisioning.md) | Physical Media Provisioning (Cloud-init + Kickstart) |
| [0008](.scratch/adr/0008-hybrid-upgrade-strategy.md) | Hybrid Upgrade Strategy |
| [0009](.scratch/adr/0009-dns-architecture.md) | DNS Architecture (dnsdist + Technitium) |
| [0010](.scratch/adr/0010-separate-macvlan-nads.md) | Separate Macvlan NADs per Node Architecture |
| [0011](.scratch/adr/0011-ip-plan-as-source-of-truth.md) | IP Plan as Source of Truth |

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

### Macvlan Parent Interfaces

| Node | LAN Interface | IoT VLAN Interface |
|------|---------------|-------------------|
| peggy (RPi) | eth0 | eth0.300 |
| yelena (RPi) | eth0 | eth0.300 |
| xialing (x86) | eno1 | eno1.300 |

**NAD selection:** Use `macvlan-lan` / `macvlan-iot` for RPi nodes, `macvlan-lan-x86` / `macvlan-iot-x86` for xialing.

**IoT VLAN interface:** Created via Ansible bootstrap (Step 2) — `eno1.300` on xialing for Home Assistant macvlan.

### Macvlan IP Ranges (Whereabouts IPAM)

| Network | Range | Notes |
|---------|-------|-------|
| LAN | 10.0.0.200 - 10.0.0.239 | Excludes node IPs (10.0.0.11-15), gateway |
| IoT VLAN | 10.0.30.200 - 10.0.30.239 | Excludes gateway |

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
- **Flux structure:** `apps/` is NOT in `manifests/kustomization.yaml`. It's managed by a Flux Kustomization CR in `infrastructure/kustomizations.yaml`. This is because:
  - Variable substitution requires `cluster-vars` ConfigMap to exist first
  - `cluster-vars` is created by `infrastructure-controllers`
  - The Flux Kustomization CRs enforce ordering: controllers → configs → apps
  - Each CR can have `postBuild.substituteFrom` to replace `${VAR}` placeholders
