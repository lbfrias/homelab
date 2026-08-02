# IP Plan Automation — Generator Script & Omada Sync

**What to build:** Automation to make `docs/ip-plan.yaml` the true source of truth for all IP configuration.

**Blocked by:** Step 5.8 (Omada Controller)

**Status:** backlog

## Overview

Two components:

1. **Generator script** — reads `ip-plan.yaml`, outputs derived configs
2. **Omada sync Job** — K8s Job that pushes DHCP config to Omada via API

See ADR: `.scratch/adr/0011-ip-plan-as-source-of-truth.md`

## Tasks

### Phase 1: Generator Script

- [ ] Create `scripts/generate-configs.py` (or shell/node)
- [ ] Parse `docs/ip-plan.yaml`
- [ ] Generate `ansible/vars.yaml` (node_ips, nfs_export_networks, etc.)
- [ ] Generate `manifests/infrastructure/controllers/cluster-vars.yaml`
- [ ] Generate cloud-init templates with static IPs for RPi nodes
- [ ] Generate kickstart template with static IP for xialing
- [ ] Generate `manifests/apps/network/omada/omada-config.yaml` (ConfigMap for sync Job)
- [ ] Add validation: check for IP conflicts, range overlaps
- [ ] Add `--check` mode for pre-commit hook
- [ ] Document usage in README

### Phase 2: Omada Sync Job

- [ ] Research Omada Open API authentication (OAuth2 client credentials)
- [ ] Create K8s Secret for Omada API credentials
- [ ] Create sync script (Python recommended — good HTTP/JSON support)
- [ ] Implement: Create/update LAN networks (VLANs)
- [ ] Implement: Set DHCP pool ranges
- [ ] Implement: Create IP-MAC bindings (reservations)
- [ ] Create K8s Job manifest (runs on ConfigMap change)
- [ ] Add Flux Kustomization to trigger Job after Omada healthy
- [ ] Test: Change DHCP pool in ip-plan.yaml → verify Omada updates

### Phase 3: Full Network GitOps (Future)

- [ ] Add `docs/network-config.yaml` for non-IP Omada settings
- [ ] Implement: SSIDs and wireless security
- [ ] Implement: ACLs / firewall rules
- [ ] Implement: NAT / port forwarding
- [ ] Implement: mDNS / Bonjour profiles
- [ ] Implement: Switch port profiles
- [ ] Implement: PoE settings
- [ ] Implement: User/role management

### Phase 4: Router Bootstrap (Future)

- [ ] Document Docker Omada bootstrap procedure
- [ ] Create bootstrap script for dev machine
- [ ] Script: Adopt router via API
- [ ] Script: Push initial router config (LAN IP, basic DHCP)
- [ ] Test: Full reprovision from factory-reset router

## API Endpoints (Reference)

From Omada Open API validation:

| Function | Endpoint |
|----------|----------|
| LAN networks | `GET/POST/PATCH /openapi/v1/{omadacId}/sites/{siteId}/lan-networks` |
| DHCP settings | `PATCH /lan-networks/{networkId}` → `dhcpSettingsVO` |
| IP-MAC bindings | `GET/POST/DELETE /ip-mac-binds` |
| Batch adopt | `POST /cmd/devices/batch-adopt` |
| Users | `GET/POST/PUT/DELETE /users` |
| Roles | `GET/POST /roles` |

## Acceptance Criteria

**Phase 1 complete when:**
- Editing `ip-plan.yaml` and running generator produces correct derived files
- Pre-commit hook catches uncommitted generator output
- CI validates generated files match source

**Phase 2 complete when:**
- Changing DHCP pool in ip-plan.yaml → push → Omada reflects change
- Adding device to ip-plan.yaml → push → DHCP reservation created
- No manual Omada UI interaction required for IP management

## Notes

- Omada API docs: https://use1-omada-northbound.tplinkcloud.com/doc.html
- Existing Python client: https://github.com/MarkGodwin/tplink-omada-api
- Consider using existing client vs raw API calls
