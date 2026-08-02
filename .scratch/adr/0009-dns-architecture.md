# DNS Architecture (dnsdist + Technitium)

Two-tier DNS with dnsdist (load balancer) + Technitium (DNS server), all on macvlan. Provides ad-blocking, authoritative DNS for owned domains, HA across all nodes, and per-client visibility.

## Architecture

```
Clients (2 DNS servers configured)
    │
    ▼
dnsdist x2 (macvlan IPs) ─── VIPs for clients
    │
    ▼ round-robin
Technitium x3 (macvlan IPs) ─── one per node
```

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
- dnsdist preserves client source IP natively (no SNAT)
- Purpose-built for DNS — health checks via actual DNS queries

## Why macvlan for everything

- Per-client visibility requires no SNAT — macvlan delivers this
- Direct L2 path: client → dnsdist → Technitium (no kube-proxy)
- Stable IPs (manually assigned) vs ephemeral pod IPs
- Health checks validate the actual serving interface

## Why 2 dnsdist + 3 Technitium

- 2 VIPs match typical client DNS server limit
- 3 backends for prod simulation (all nodes participate)
- Each VIP load balances to all 3 backends

## Trade-offs accepted

- Extra component (dnsdist) vs simpler MetalLB
- Manual IP management for macvlan interfaces
- dnsdist config is Lua (less familiar than YAML)
