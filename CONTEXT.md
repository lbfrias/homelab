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

### ADR-006: External Secrets with Bitwarden

**Status:** Accepted

**Context:** Repo is public. Secrets cannot be committed, even encrypted.

**Decision:** External Secrets Operator pulling from Bitwarden. README documents required secrets for others.

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

## Provisioning Stack

| Layer | Tool |
|-------|------|
| OS install (RPi) | SD card + cloud-init |
| OS install (x86) | USB installer + Kickstart |
| Node config | Ansible |
| K8s workloads | Flux (GitOps) |
| Secrets | External Secrets Operator → Bitwarden |

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
- **MetalLB Pool:** 10.0.0.30-10.0.0.99

### Static IPs (Infrastructure)

| Service | IP |
|---------|-----|
| Home Assistant | TBD (IoT VLAN) |
| Omada Controller | TBD (LAN) |
| PiHole | 10.0.0.98, 10.0.0.99 |

## Services

### Current (migrating from homelab-ansible)

- **Media:** Jellyfin, Radarr, Sonarr, Bazarr, Prowlarr, Transmission, Kavita, Mylar3
- **Network:** PiHole, Tailscale, Omada Controller
- **Home Automation:** Home Assistant
- **Storage:** Longhorn, NFS from xialing
- **Monitoring:** Prometheus + Grafana (deferred)

## Repository Structure

```
homelab/
├── ansible/              # Node provisioning
│   ├── inventory/
│   ├── playbooks/
│   └── roles/
├── manifests/            # K8s workloads (Flux watches)
│   ├── flux-system/
│   ├── infrastructure/
│   └── apps/
├── docs/                 # Architecture, guides
│   ├── networking.md
│   ├── restore-guide.md
│   └── secrets.md
├── CONTEXT.md            # This file
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
