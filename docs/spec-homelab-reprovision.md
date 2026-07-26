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

1. **PXE boot** installs the OS on all nodes automatically
2. **Ansible** configures nodes and bootstraps K3s
3. **Flux** deploys and manages all K8s workloads via GitOps
4. **Longhorn** provides persistent storage with NAS-backed backups
5. **External Secrets Operator** pulls secrets from Bitwarden

The entire stack is version-controlled in a public repo, serving as both disaster recovery documentation and a portfolio piece.

## User Stories

1. As a homelab operator, I want to boot a fresh node from PXE, so that I don't need to manually flash SD cards or USB drives.

2. As a homelab operator, I want a single Ansible command to configure all nodes, so that I can reprovision without remembering manual steps.

3. As a homelab operator, I want K3s installed in HA mode across all 3 nodes, so that the cluster survives single-node failures.

4. As a homelab operator, I want Flux to manage all K8s workloads, so that the cluster self-heals from drift.

5. As a homelab operator, I want MetalLB providing LoadBalancer IPs, so that services are accessible on the LAN.

6. As a homelab operator, I want Longhorn for persistent storage, so that app data survives pod rescheduling.

7. As a homelab operator, I want Longhorn backups to my NAS, so that I can recover data after disk failures.

8. As a homelab operator, I want Home Assistant running with a macvlan interface, so that it can discover devices via mDNS.

9. As a homelab operator, I want Omada Controller running with a macvlan interface, so that it can adopt APs via L2 discovery.

10. As a homelab operator, I want PiHole running with a stable IP, so that I can configure it as my network's DNS.

11. As a homelab operator, I want Tailscale for remote access, so that I can manage the cluster from anywhere.

12. As a homelab operator, I want my media stack (Jellyfin, Radarr, Sonarr, etc.) deployed via GitOps, so that I don't lose my configuration.

13. As a homelab operator, I want Jellyfin to use hardware transcoding on xialing, so that 4K streams don't buffer.

14. As a homelab operator, I want secrets pulled from Bitwarden, so that I can keep the repo public.

15. As a homelab operator, I want External Secrets Operator configured, so that K8s secrets are automatically populated.

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

- **OS install:** PXE boot via Docker container on desktop (proxy DHCP mode)
- **Node config:** Ansible with roles for common, k3s, and OS-specific tasks
- **K8s workloads:** Flux GitOps watching `manifests/` directory
- **Secrets:** External Secrets Operator → Bitwarden

### Ansible Structure

- **Inventory:** Static YAML with groups for `rpi` (peggy, yelena) and `x86` (xialing)
- **Roles:**
  - `common` — SSH hardening, NTP, packages, mounts
  - `rpi` — cgroups, cmdline.txt tweaks
  - `rocky` — SELinux, firewalld
  - `k3s` — K3s install with HA flags, token handling
- **Idempotency:** All tasks must be safe to re-run

### K3s Configuration

- All 3 nodes as control plane (embedded etcd, quorum of 2)
- Disabled components: `servicelb`, `traefik` (replaced by MetalLB, no ingress controller needed initially)
- First node uses `--cluster-init`, others join via `--server https://xialing:6443`
- K3s data directory symlinked to HDD on nodes with external storage

### Flux Directory Structure

```
manifests/
├── clusters/
│   └── homelab/
│       └── flux-system/     # Flux bootstrap
├── infrastructure/
│   ├── controllers/         # MetalLB, Longhorn, ESO, Multus
│   └── configs/             # MetalLB pools, NADs, ClusterSecretStore
└── apps/
    ├── media/               # Jellyfin, *arr stack
    ├── network/             # PiHole, Tailscale, Omada
    └── home/                # Home Assistant
```

- Kustomizations with dependencies ensure correct ordering
- Infrastructure deploys before apps

### Multus + Macvlan Networking

- Multus installed via thick plugin (K3s-specific path: `/var/lib/rancher/k3s/agent/etc/cni/net.d/`)
- Whereabouts IPAM for dynamic IP assignment within defined ranges
- Static IPs for infrastructure pods (HA, Omada) via pod annotations
- macvlan-shim DaemonSet creates host interface for node-to-pod communication
- NetworkAttachmentDefinitions:
  - `lan-macvlan` — 10.0.0.0/24, parent `eth0`
  - `iot-macvlan` — 10.0.30.0/24, parent `eth0.30` (VLAN tagged)

### MetalLB Configuration

- L2 mode (no BGP, simple home network)
- IP pool: 10.0.0.30–10.0.0.99
- IPAddressPool and L2Advertisement resources

### Longhorn Configuration

- Default StorageClass
- Backup target: NFS to xialing (4TB USB HDD)
- Replica count: 2 (survives 1 node failure)
- Scheduled backups for critical volumes (HA, Omada, *arr configs)

### External Secrets

- ClusterSecretStore pointing to Bitwarden
- ExternalSecret resources per app namespace
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

### Ansible Role Tests (Molecule)

- Each role tested against containerized targets (Docker)
- Tests verify:
  - Package installation
  - Service configuration
  - File permissions
  - Idempotency (second run = no changes)
- Prior art: None in old repo, but standard Molecule patterns apply

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
4. MetalLB assigns IPs to LoadBalancer services
5. Longhorn UI accessible, all volumes healthy
6. ESO creates expected secrets

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

### PXE Boot Fallback

If PXE proves complex, fallback options:
1. Cloud-init images with autoinstall
2. Pre-configured SD cards / USB drives via Raspberry Pi Imager

### Phase Ordering

Implementation should follow dependency order:
1. **Phase 1:** PXE/Ansible (can provision nodes)
2. **Phase 2:** K3s + Flux + core infra (cluster runs)
3. **Phase 3:** Multus/macvlan (networking for HA/Omada)
4. **Phase 4:** Workloads (apps deploy)
5. **Phase 5:** Backup validation (DR tested)

Each phase should be independently testable before moving to the next.
