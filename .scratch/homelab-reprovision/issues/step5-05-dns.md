# Step 5.5 — DNS (dnsdist + Technitium)

**What to build:** Two-tier DNS with dnsdist load balancing to Technitium backends, all on macvlan for per-client visibility.

**Blocked by:** Step 5.4 (Multus/Macvlan)

**Status:** ready

## Architecture

```
Clients (configure dnsdist VIPs)
    │
    ▼
dnsdist x2 (macvlan: TBD) ─── VIPs for clients
    │
    ▼ round-robin
Technitium x3 (macvlan: TBD) ─── one per node
```

## Tasks

### Technitium (DNS Server)

- [ ] Create Technitium StatefulSet in `manifests/apps/network/technitium/`
- [ ] Configure macvlan annotations for static IPs (.61, .62, .63)
- [ ] Create PVC for Technitium data (zone configs)
- [ ] Configure Prometheus metrics exporter
- [ ] Set up authoritative zones for owned domains via API
- [ ] Configure upstream resolvers (Cloudflare, Quad9)
- [ ] Configure ad-blocking lists

### dnsdist (Load Balancer)

- [ ] Create dnsdist Deployment in `manifests/apps/network/dnsdist/`
- [ ] Configure macvlan annotations for static IPs (.50, .51)
- [ ] Create ConfigMap with Lua config (3 backends, round-robin)
- [ ] Configure health checks (DNS query-based)

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
| dnsdist VIP 1 | TBD |
| dnsdist VIP 2 | TBD |
| Technitium (peggy) | TBD |
| Technitium (yelena) | TBD |
| Technitium (xialing) | TBD |

## Notes

- dnsdist preserves client source IP natively — no special config needed
- Technitium API enables GitOps for zone management (no web UI dependency)
- See ADR-009 in CONTEXT.md for full rationale
