# IP Plan as Source of Truth

All IP allocations are defined in `docs/ip-plan.yaml`, which serves as the single source of truth for network configuration. A generator script produces derived configs (`ansible/vars.yaml`, `cluster-vars.yaml`, cloud-init templates), and a K8s Job syncs DHCP settings to the Omada Controller via API. This eliminates manual synchronization between multiple config files and enables full network reproducibility from Git.

## Context

IP addresses were previously defined in multiple places that had to be manually kept in sync:
- `ansible/vars.yaml` — node IPs for Ansible
- `manifests/infrastructure/controllers/cluster-vars.yaml` — node IPs for Flux substitution
- Omada Controller UI — DHCP pools and reservations (not in Git)

This created drift risk and made full reprovisioning error-prone.

## Decision

1. **`docs/ip-plan.yaml` is the source of truth** for all IP assignments: VLANs, DHCP pools, node IPs, macvlan ranges, static device reservations.

2. **K8s nodes use static IPs** configured via cloud-init (RPi) or kickstart (x86), not DHCP reservations. This removes the chicken-and-egg dependency on Omada during initial cluster bootstrap.

3. **A generator script** reads `ip-plan.yaml` and produces:
   - `ansible/vars.yaml`
   - `manifests/infrastructure/controllers/cluster-vars.yaml`
   - Cloud-init/kickstart templates with static IPs
   - ConfigMap for Omada sync Job

4. **A K8s Job** runs after Omada Controller deploys and syncs DHCP configuration via the Omada Open API:
   - DHCP pool ranges
   - IP-MAC bindings (reservations) for static devices
   - VLANs / LAN networks

5. **Router bootstrap** requires a temporary Docker Omada instance on the dev machine for initial adoption and config push (since K8s doesn't exist yet). After the router is configured with the correct LAN IP (10.0.0.1/24), K8s nodes can boot and the cluster takes over.

## Workflow

**IP change:**
```
1. Edit docs/ip-plan.yaml
2. Run ./scripts/generate-configs.py
3. git add -A && git commit && git push
4. Flux reconciles (K8s Job syncs to Omada if needed)
5. Reflash node media if node IP changed
```

**Full reprovision:**
```
1. Run Docker Omada on dev machine
2. Script pushes router config via API (LAN IP, DHCP)
3. Router reprovisions to 10.0.0.x
4. Boot K8s nodes (static IPs from cloud-init)
5. Ansible bootstraps cluster
6. Flux deploys Omada + sync Job
7. Job adopts switch/AP, pushes remaining config
```

## Consequences

- All IP config is versioned in Git
- Reprovisioning is deterministic (no manual Omada UI steps except device adoption)
- Generator script must be run before committing IP changes
- Router is special: requires Docker Omada bootstrap on fresh install
