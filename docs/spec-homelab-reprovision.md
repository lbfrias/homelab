# Spec: Homelab Full Reprovision

**Status:** ready-for-agent  
**Created:** 2026-07-27

---

## Problem Statement

The homelab currently exists as a running K3s cluster with workloads, but there's no automated way to tear it down and rebuild it from scratch. If a disaster strikes (disk failures, corruption, or even just wanting a clean slate), recovery requires manual intervention, tribal knowledge, and hours of work.

The user needs full Infrastructure as Code that enables:
- One-command reprovisioning of all nodes
- GitOps-managed workloads that self-heal
- Documented, tested disaster recovery procedures

## Solution

Build a complete IaC stack that provisions the 3-node K3s cluster from bare metal to running workloads:

1. **Physical media** installs the OS (cloud-init for RPi, Kickstart for x86)
2. **Ansible** configures nodes and bootstraps K3s
3. **Flux** deploys and manages all K8s workloads via GitOps
4. **Longhorn** provides persistent storage with NAS-backed backups
5. **Bitwarden Secrets Manager Operator** syncs secrets from Bitwarden

The entire stack is version-controlled in a public repo, serving as both disaster recovery documentation and a portfolio piece.

## User Stories

1. As a homelab operator, I want to flash OS images with cloud-init configs, so that nodes are ready for Ansible on first boot without manual setup.

2. As a homelab operator, I want a single Ansible command to configure all nodes, so that I can reprovision without remembering manual steps.

3. As a homelab operator, I want K3s installed in HA mode across all 3 nodes, so that the cluster survives single-node failures.

4. As a homelab operator, I want Flux to manage all K8s workloads, so that the cluster self-heals from drift.

5. As a homelab operator, I want to test new workloads on `test/*` branches before merging to main, so that I can validate changes without affecting production.

6. As a homelab operator, I want Longhorn for persistent storage, so that app data survives pod rescheduling.

7. As a homelab operator, I want Longhorn backups to my NAS, so that I can recover data after disk failures.

8. As a homelab operator, I want Home Assistant running with a macvlan interface, so that it can discover devices via mDNS.

9. As a homelab operator, I want Omada Controller running with a macvlan interface, so that it can adopt APs via L2 discovery.

10. As a homelab operator, I want Technitium running on all 3 nodes with dnsdist load balancing, so that I have highly available DNS with per-client visibility.

11. As a homelab operator, I want Tailscale for remote access, so that I can manage the cluster from anywhere.

12. As a homelab operator, I want my media stack (Jellyfin, Radarr, Sonarr, etc.) deployed via GitOps, so that I don't lose my configuration.

13. As a homelab operator, I want Jellyfin to use hardware transcoding on xialing, so that 4K streams don't buffer.

14. As a homelab operator, I want secrets pulled from Bitwarden, so that I can keep the repo public.

15. As a homelab operator, I want Bitwarden Secrets Manager Operator configured, so that K8s secrets are automatically populated.

16. As a homelab operator, I want documented manual secret creation, so that others can reproduce the setup without Bitwarden.

17. As a homelab operator, I want a tested restore procedure, so that I know backups actually work.

18. As a homelab operator, I want in-place OS patching via Ansible, so that I stay secure without full reprovisions.

19. As a homelab operator, I want the macvlan-shim DaemonSet deployed, so that nodes can communicate with macvlan pods.

20. As a homelab operator, I want Whereabouts IPAM for macvlan, so that infrastructure pods get predictable IPs.

21. As a homelab operator, I want cgroups enabled on RPi nodes, so that K3s runs correctly.

22. As a homelab operator, I want etcd data stored on the HDD (/var), so that SD cards aren't worn out.

23. As a homelab operator, I want Multus CNI installed via K3s-specific paths, so that macvlan secondary interfaces work.

24. As a homelab operator, I want a public repo with clear documentation, so that it serves as a portfolio piece.

25. As a homelab operator, I want NetworkAttachmentDefinitions for both LAN and IoT VLANs, so that pods can join different networks.

## Implementation Decisions

### Provisioning Stack Layers

- **OS install (RPi):** SD card with cloud-init (peggy, yelena)
- **OS install (x86):** USB installer with Kickstart (xialing — internal SSD)
- **Node config:** Ansible with roles for common, k3s, and OS-specific tasks
- **K8s workloads:** Flux GitOps watching `manifests/` directory
- **Secrets:** Bitwarden Secrets Manager Operator

### Ansible Structure (Five-Step Aligned)

- **Inventory:** Static YAML with groups for `rpi` (peggy, yelena) and `x86` (xialing)
- **Playbooks:**
  - `ping.yaml` — Verify connectivity
  - `bootstrap.yaml` — Step 2: packages, kernel modules, sysctl, NFS/SMB
  - `k3s.yaml` — Step 3: K3s HA cluster installation
  - `flux.yaml` — Step 4: GitOps bootstrap
  - `site.yaml` — Runs Steps 2-4 in sequence
- **Roles:**
  - `common` — Shared packages and K8s prerequisites
  - `nfs_smb` — NFS/SMB server (xialing only)
- **Philosophy:** Start minimal, debug issues as they arise, document fixes in CONTEXT.md as real ADRs
- **Idempotency:** All tasks must be safe to re-run

### Package Management

Two-layer approach for idempotent package installation:

1. **Kickstart `%packages`** — Minimal base only (openssh, sudo, vim-minimal, rsync). These are installed during OS provisioning.

2. **Ansible role** — Runtime packages declared in a `packages` variable. When you need a new package on a live cluster:
   - Add to `ansible/roles/common/defaults/main.yaml` (or group/host vars)
   - Run: `ansible-playbook site.yaml -l xialing --tags packages`
   - On next full reprovision, Ansible installs it idempotently

This keeps Kickstart lean and makes package management a normal Ansible operation.

### Disk Partitioning (xialing)

Optimized for k3s workloads on a 120GB SSD:

| Partition | Size | Purpose |
|-----------|------|---------|
| /boot/efi | 600MB | UEFI boot |
| /boot | 1GB | Kernel images |
| / (root) | 30GB | OS, binaries |
| /var | ~75GB | K3s images, containers, etcd, logs |
| /home | 5GB | User dotfiles (minimal, no user data) |
| swap | 4GB | Swap space |

User data lives on the 4TB USB HDD mounted at `/mnt/data`.

### K3s Configuration

- All 3 nodes as control plane (embedded etcd, quorum of 2)
- Disabled components: `servicelb`, `traefik` (services use macvlan or NodePort)
- First node uses `--cluster-init`, others join via `--server https://xialing:6443`
- K3s data directory symlinked to HDD on nodes with external storage

### Flux Directory Structure

```
manifests/
├── flux-system/             # Flux bootstrap (auto-generated)
├── infrastructure/
│   ├── controllers/         # Longhorn, Bitwarden SM, Multus, Flux Operator
│   └── configs/             # NADs, ResourceSet
└── apps/
    ├── media/               # Jellyfin, *arr stack
    ├── network/             # dnsdist, Technitium, Tailscale, Omada
    └── home/                # Home Assistant
```

- Kustomizations with dependencies ensure correct ordering
- Infrastructure deploys before apps

### Flux Operator + Dynamic Test Branches

The Flux Operator extends base Flux with ResourceSet and ResourceSetInputProvider CRDs, enabling automatic deployment from `test/*` branches:

**Workflow:**
1. Push to `test/my-feature` branch
2. ResourceSetInputProvider detects the branch (via GitHub API)
3. ResourceSet templates a GitRepository + Kustomization for that branch
4. Flux deploys the branch's manifests to the `test` namespace
5. Delete the branch → resources are pruned automatically

**Components:**
- **Flux Operator:** Installed via HelmRelease in `infrastructure/controllers/`
- **ResourceSetInputProvider:** Watches `test/*` branches in `lbfrias/homelab`
- **ResourceSet:** Templates GitRepository + Kustomization per branch, targets `test` namespace

**Benefits:**
- Test new workloads in isolation before merging to main
- No manual resource creation/deletion
- Branch deletion auto-cleans test resources

### Multus + Macvlan Networking

- Multus installed via thick plugin (K3s-specific path: `/var/lib/rancher/k3s/agent/etc/cni/net.d/`)
- Whereabouts IPAM for dynamic IP assignment within defined ranges
- Static IPs for infrastructure pods (HA, Omada, DNS) via pod annotations
- macvlan-shim DaemonSet creates host interface for node-to-pod communication
- NetworkAttachmentDefinitions:
  - `lan-macvlan` — 10.0.0.0/24, parent `eth0`
  - `iot-macvlan` — 10.0.30.0/24, parent `eth0.300` (VLAN 300 tagged)

### DNS Architecture (dnsdist + Technitium)

Two-tier DNS on macvlan for HA with per-client visibility:

**Architecture:**
```
Clients (configure dnsdist VIPs as DNS servers)
    │
    ▼
dnsdist x2 (macvlan: TBD) ─── load balancer VIPs
    │
    ▼ round-robin
Technitium x3 (macvlan: TBD) ─── one per node
```

**Why this design:**
- **Technitium over PiHole:** Stateless, authoritative DNS for owned domains, full API for GitOps
- **dnsdist over MetalLB:** Real round-robin (MetalLB L2 is failover-only), preserves client source IP
- **Macvlan for both tiers:** No SNAT, per-client visibility in Technitium logs/metrics
- **2 VIPs + 3 backends:** Matches typical client 2-DNS-server limit while using all nodes

**Configuration:**
- dnsdist: Lua config, forwards to all 3 Technitium backends
- Technitium: HTTP API for zone management, Prometheus metrics via exporter
- All pods get static macvlan IPs via annotations

### Longhorn Configuration

- Default StorageClass
- Backup target: NFS to xialing (4TB USB HDD)
- Replica count: 2 (survives 1 node failure)
- Scheduled backups for critical volumes (HA, Omada, *arr configs)

### Bitwarden Secrets Manager

- Bitwarden SM Operator with machine account auth
- BitwardenSecret resources per app namespace
- Secrets documented in `docs/secrets.md` for manual creation

### Home Assistant

- Deployment with dual interfaces (eth0 cluster, net1 macvlan)
- Static IP on IoT VLAN for mDNS device discovery
- PVC for `/config` with Longhorn backup
- Host network access NOT required (macvlan sufficient)

### Omada Controller

- Deployment with macvlan interface on LAN
- Static IP for AP adoption consistency
- PVC for controller data
- Inform URL configured to macvlan IP

### Media Stack

- All apps in `media` namespace
- Shared NFS mount for media files (from xialing 4TB HDD)
- Individual PVCs for app configs
- Jellyfin with GPU passthrough for Intel Quick Sync (xialing only, node selector)

## Testing Decisions

### What Makes a Good Test

- Tests validate **external behavior**, not implementation details
- Tests should be runnable without real hardware where possible
- Integration tests document expected system state
- Failed tests should provide actionable diagnostic information

### Ansible Playbook Tests

- Minimal testing: verify playbooks are idempotent (second run = no changes)
- No Molecule initially — add if roles grow complex enough to warrant it
- Integration testing against real hardware preferred over containerized mocks

### Manifest Validation (Pre-commit)

- `kubeconform` validates all YAML against K8s schemas
- Kustomize build succeeds for each overlay
- Flux dry-run catches dependency issues
- Runs in CI and pre-commit hook

### Integration Test: Cluster Bootstrap

Documented smoke test procedure:

1. Ansible completes without errors
2. All nodes show `Ready` in `kubectl get nodes`
3. Flux reconciliation succeeds (`flux get all` shows ready)
4. Longhorn UI accessible, all volumes healthy
5. Bitwarden SM Operator creates expected secrets

### Integration Test: Multus/Macvlan

1. Deploy test pod with macvlan annotation
2. Verify pod gets IP from Whereabouts range
3. Verify pod can ping LAN gateway (10.0.0.1)
4. Verify node can ping pod (via macvlan-shim)
5. Clean up test pod

### Restore Test

Periodic (monthly) test:

1. Delete prowlarr PVC (non-critical, quick to restore)
2. Restore volume from Longhorn backup
3. Verify prowlarr starts with restored data
4. Document results

## Out of Scope

- **Monitoring stack (Prometheus/Grafana):** Deferred to future spec
- **Ingress controller:** Not needed initially; services accessed via LoadBalancer IPs
- **CI/CD pipeline:** Manual flux reconciliation or `flux reconcile` sufficient
- **Multi-cluster:** Single homelab cluster only
- **Cloud backup:** NAS-only backup; offsite backup is future work
- **Cert-manager/TLS:** Internal services, no HTTPS required initially
- **Node auto-scaling:** Fixed 3-node cluster

## Further Notes

### Open Questions to Resolve During Implementation

- **Static IPs for HA and Omada:** Need to finalize addresses (currently TBD in CONTEXT.md)
- **Parent interface name:** Assumed `eth0`, needs verification on each node
- **Whereabouts vs static IPAM:** May use static annotations for infrastructure pods, Whereabouts for others

### Migration from Old Repo

The `homelab-ansible` repo contains working manifests to migrate:
- `manifests/media-manager/` — All *arr apps, Jellyfin, Transmission
- `manifests/pihole/` — PiHole deployment and custom DNS
- `manifests/tailscale/` — Tailscale operator
- `k3s-setup/` — K3s installation playbook (adapt, don't copy verbatim)

### PXE Boot (Removed)

Originally considered PXE boot for full automation, but decided physical media is simpler for a 3-node cluster. PXE adds complexity (DHCP proxy, boot server) without significant benefit at this scale.

### Kickstart for x86 Nodes

xialing has an internal SSD that can't be flashed from the desktop. Instead, boot from USB installer with Kickstart config for automated install:
- Partitions internal SSD
- Creates user with SSH key
- Sets hostname
- Reboots ready for Ansible

Kickstart config lives in `provisioning/templates/ks.cfg.j2` alongside cloud-init templates.

### Phase Ordering (Five-Step Model)

Implementation follows the five-step provisioning model:

| Step | Name | Tool | What It Does |
|------|------|------|--------------|
| 1 | **Provision** | cloud-init / Kickstart | OS install, SSH access |
| 2 | **Bootstrap** | `ansible-playbook playbooks/bootstrap.yaml` | Packages, kernel modules, sysctl, NFS/SMB |
| 3 | **K3s** | `ansible-playbook playbooks/k3s.yaml` | HA cluster with embedded etcd |
| 4 | **Flux** | `ansible-playbook playbooks/flux.yaml` | GitOps bootstrap |
| 4.1 | **Flux Operator** | Flux reconciles `manifests/infrastructure/controllers/` | ResourceSet for test branches |
| 5 | **Services** | Automatic (Flux reconciles `manifests/`) | Infrastructure and apps deploy |

Document every fix needed in CONTEXT.md — these become real ADRs based on actual problems, not cargo-culted configs from old playbooks.
