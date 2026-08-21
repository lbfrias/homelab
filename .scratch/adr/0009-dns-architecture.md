# DNS Architecture (dnsdist + Technitium)

Two-tier DNS with dnsdist (load balancer) + Technitium (DNS server). Provides ad-blocking, authoritative DNS for owned domains, HA across all nodes, and per-client visibility.

## Architecture

```
Clients (2 DNS servers configured)
    │
    ▼
dnsdist x2 (macvlan .98, .99) ─── VIPs for external clients
    │
    │  k8s network + PROXY protocol (preserves client IP)
    ▼
Technitium x3 (StatefulSet) ─── one per node via anti-affinity
    │
    └── macvlan .95-.97 for web UI access
```

**Key design:** dnsdist talks to Technitium via k8s network (pod IPs), not macvlan. This avoids the macvlan hairpin problem where pods on the same node can't communicate via macvlan. Client IPs are preserved via PROXY protocol.

## Why not PiHole

- Stateful (SQLite DB) — harder to manage in K8s
- No authoritative DNS support for owned domains

## Why Technitium over Blocky

- Blocky is forwarding-only; can't serve as authoritative for owned domains
- Technitium handles both recursive resolution + authoritative zones
- Full HTTP API enables GitOps (no dependency on web UI)
- Prometheus metrics via exporter

## Why dnsdist over MetalLB

- MetalLB L2 mode is failover-only (1 active node), not true load balancing
- Most clients only accept 2 DNS servers, but we have 3 Technitium instances
- dnsdist provides real round-robin across all 3 backends per VIP
- dnsdist preserves client source IP via PROXY protocol
- Purpose-built for DNS — health checks via actual DNS queries

## Why macvlan for client-facing only

- Per-client visibility requires real client IPs — dnsdist on macvlan receives them
- PROXY protocol passes client IP to Technitium over k8s network
- Avoids macvlan hairpin problem (same-node pods can't talk via macvlan in bridge mode)
- Technitium still has macvlan for direct web UI access

## Why StatefulSet for Technitium

- Stable DNS names: `technitium-{0,1,2}.technitium.dns.svc.cluster.local`
- dnsdist resolves these names to pod IPs at startup
- Anti-affinity ensures one pod per node (like DaemonSet, but with stable names)

## Dynamic backend discovery

dnsdist doesn't support hostnames in backend config — only IPs. We solve this with:

1. **Init container** resolves StatefulSet DNS names → pod IPs at startup
2. **Sidecar watcher** monitors for IP changes every 15 seconds
3. On change: regenerates config, kills dnsdist → container restarts with new IPs

This ensures dnsdist automatically adapts when Technitium pods restart and get new IPs.

## Why 2 dnsdist + 3 Technitium

- 2 VIPs match typical client DNS server limit
- 3 backends for HA (all nodes participate)
- Each VIP load balances to all 3 backends

## Trade-offs accepted

- Extra component (dnsdist) vs simpler MetalLB
- Init container + sidecar complexity for dynamic IP discovery
- dnsdist config is Lua (less familiar than YAML)
