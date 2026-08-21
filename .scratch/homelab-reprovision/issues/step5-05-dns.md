# Step 5.5 — DNS (dnsdist + Technitium)

**What to build:** Two-tier DNS with dnsdist load balancing to Technitium backends. dnsdist on macvlan for client access, communicates with Technitium via k8s network using PROXY protocol to preserve client IPs.

**Blocked by:** Step 5.4 (Multus/Macvlan)

**Status:** done

## Architecture

```
Clients (configure dnsdist VIPs: 10.0.0.98, 10.0.0.99)
    │
    ▼
dnsdist x2 (macvlan: .98, .99) ─── VIPs for clients
    │
    │  k8s network + PROXY protocol
    ▼
Technitium x3 (StatefulSet, pod IPs) ─── one per node via anti-affinity
    │
    └── macvlan: .95-.97 for web UI access
```

**Why k8s network between dnsdist and Technitium?**
Macvlan bridge mode has a "hairpin problem" — pods on the same node can't communicate via macvlan. Using k8s network avoids this while PROXY protocol preserves client IPs.

## Tasks

### Technitium (DNS Server)

- [x] Create Technitium StatefulSet in `manifests/apps/network/dns/technitium/`
- [x] Configure anti-affinity to spread across nodes (one per node)
- [x] Configure macvlan with Whereabouts IPAM for IPs (.95, .96, .97) — web UI access
- [x] Use emptyDir for Technitium config (zone/records recreated via API on start)
- [x] Configure PROXY protocol ACL for k8s pod network (10.42.0.0/16)
- [ ] Configure Prometheus metrics exporter
- [ ] Set up authoritative zones for owned domains (frias.app, levinf.com) via API automation
- [x] Configure upstream resolvers (Cloudflare, Quad9)
- [x] Configure ad-blocking lists

### dnsdist (Load Balancer)

- [x] Create dnsdist Deployment in `manifests/apps/network/dns/dnsdist/`
- [x] Configure macvlan with Whereabouts IPAM for IPs (.98, .99)
- [x] Init container to resolve StatefulSet DNS names to pod IPs
- [x] Sidecar watcher to detect Technitium IP changes and reload dnsdist
- [x] Create ConfigMap with Lua config template (3 backends, round-robin)
- [x] Configure health checks (DNS query-based)
- [x] Enable shareProcessNamespace for sidecar to signal dnsdist

### Infrastructure

- [x] Enable Whereabouts IPAM in Multus helm release
- [x] Create macvlan-dns-technitium NAD with Whereabouts IPAM
- [x] Create macvlan-dns-dnsdist NAD with Whereabouts IPAM
- [x] Unified interface naming (eth0 → eno1) for all nodes

### Verification

- [x] Verify: DNS queries resolve correctly via both VIPs
- [x] Verify: Queries are distributed across all 3 Technitium instances
- [x] Verify: Ad-blocking works (ads.google.com blocked)
- [x] Verify: Technitium logs show real client IPs (via PROXY protocol)
- [x] Verify: dnsdist auto-reloads when Technitium pod IPs change
- [ ] Verify: Authoritative zones respond correctly
- [x] Verify: Failover works (stop one Technitium, queries still succeed)

## IPs

| Component | IP | Purpose |
|-----------|-----|---------|
| dnsdist VIP 1 | 10.0.0.98 | Client DNS server |
| dnsdist VIP 2 | 10.0.0.99 | Client DNS server |
| Technitium (Whereabouts) | 10.0.0.95-97 | Web UI access |
| Technitium (k8s) | 10.42.x.x (dynamic) | dnsdist backend traffic |

## Access

- **Technitium Web UI:** http://10.0.0.95:5380 (or .96, .97)
- **Username:** admin
- **Password:** stored in `technitium-admin-password` secret in `dns` namespace

## Notes

- dnsdist uses PROXY protocol (port 538) to pass real client IPs to Technitium
- Technitium PROXY ACL accepts connections from k8s pod network (10.42.0.0/16)
- StatefulSet provides stable DNS names: `technitium-{0,1,2}.technitium.dns.svc.cluster.local`
- dnsdist init container resolves these names to pod IPs for config generation
- dnsdist sidecar watches for IP changes every 15 seconds, regenerates config and restarts dnsdist
- See ADR-009 for full architecture rationale
- dnsdist deployment uses maxSurge: 0 to prevent IP exhaustion during rolling updates
