# Step 5.5 — DNS (dnsdist + Technitium)

**What to build:** Two-tier DNS with dnsdist load balancing to Technitium backends, all on macvlan for per-client visibility.

**Blocked by:** Step 5.4 (Multus/Macvlan)

**Status:** in-progress

## Architecture

```
Clients (configure dnsdist VIPs: 10.0.0.98, 10.0.0.99)
    │
    ▼
dnsdist x2 (macvlan: .98, .99) ─── VIPs for clients
    │
    ▼ round-robin
Technitium x3 (macvlan: .95-.97 via Whereabouts) ─── one per node
```

## Tasks

### Technitium (DNS Server)

- [x] Create Technitium StatefulSet in `manifests/apps/network/technitium/`
- [x] Configure macvlan with Whereabouts IPAM for IPs (.95, .96, .97)
- [x] Create volumeClaimTemplate for Technitium data (zone configs)
- [ ] Configure Prometheus metrics exporter
- [ ] Set up authoritative zones for owned domains (frias.app, levinf.com) via API automation
- [x] Configure upstream resolvers (Cloudflare, Quad9)
- [x] Configure ad-blocking lists

### dnsdist (Load Balancer)

- [x] Create dnsdist Deployment in `manifests/apps/network/dnsdist/`
- [x] Configure macvlan annotations for static IPs (.98, .99)
- [x] Create ConfigMap with Lua config (3 backends, round-robin)
- [x] Configure health checks (DNS query-based)

### Infrastructure

- [x] Enable Whereabouts IPAM in Multus helm release
- [x] Create macvlan-dns NAD with Whereabouts IPAM
- [x] Unified interface naming (eth0 → eno1) for all nodes

### Verification

- [ ] Verify: DNS queries resolve correctly via both VIPs
- [ ] Verify: Queries are distributed across all 3 Technitium instances
- [ ] Verify: Technitium logs show real client IPs (not dnsdist IPs)
- [ ] Verify: Ad-blocking works
- [ ] Verify: Authoritative zones respond correctly
- [ ] Verify: Failover works (stop one Technitium, queries still succeed)

## IPs

| Component | IP |
|-----------|-----|
| dnsdist VIP 1 | 10.0.0.98 |
| dnsdist VIP 2 | 10.0.0.99 |
| Technitium (Whereabouts range) | 10.0.0.95-97 |

## Notes

- dnsdist preserves client source IP natively — no special config needed
- Technitium API enables GitOps for zone management (no web UI dependency)
- See ADR-009 in CONTEXT.md for full rationale
- Interface unified to 'eno1' across all nodes (RPi renamed from eth0)
