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

Both share one script (`dnsdist/scripts-configmap.yaml`, modes `once` and `watch`)
so startup and steady-state resolution cannot drift apart.

The sidecar must regenerate the config itself rather than exiting to force a
restart: container restarts do **not** re-run init containers, so an exit-only
design would come back up with the stale config.

### Resolution must be all-or-nothing

An earlier version substituted whatever `nslookup` returned. A single transient
lookup failure therefore wrote an **empty** backend address into the config and
killed dnsdist, leaving it running with no usable backends — observed as
intermittent, network-wide resolution failures.

The script now:

- validates every result against an IPv4 pattern, so a failed lookup can never
  reach the config
- resolves all three backends or none — a partial result is discarded
- requires a change to be observed twice, 5s apart, before acting, so pod churn
  doesn't trigger needless restarts
- writes via a temp file + `mv`, so dnsdist can never read a half-written config
- logs to stderr, keeping stdout as the resolved-IP channel

## Node resolvers must not point at the cluster

Cluster nodes use upstream resolvers (1.1.1.1 / 1.0.0.1), never the in-cluster
VIPs 10.0.0.98/.99. Those VIPs are served by dnsdist pods running on the nodes
themselves, so a node depending on them cannot resolve anything until K3s is up —
and K3s needs DNS to pull images. That circular dependency bricks a cold boot.

This is applied at provisioning time (Step 1) via a NetworkManager global-dns
drop-in, `/etc/NetworkManager/conf.d/10-homelab-dns.conf`, which overrides both
DHCP-supplied and per-connection DNS. A global override is used rather than
per-connection `nmcli` settings because it needs no knowledge of the connection
or device name — RPi NICs are still `eth0` at first boot and are renamed to
`eno1` in Step 3, and DHCP-created connection names differ per node.

## Why 2 dnsdist + 3 Technitium

- 2 VIPs match typical client DNS server limit
- 3 backends for HA (all nodes participate)
- Each VIP load balances to all 3 backends

## Trade-offs accepted

- Extra component (dnsdist) vs simpler MetalLB
- Init container + sidecar complexity for dynamic IP discovery
- dnsdist config is Lua (less familiar than YAML)
- `maxSurge: 0` plus a 2-address Whereabouts range means a dnsdist rollout is
  serialised and can stall if a reservation is leaked (see runbook below)

## Runbook: dnsdist pod stuck in `Init` with no macvlan IP

Symptom — the pod never gets `net1` and events show:

```
Could not allocate IP in range: ip: 10.0.0.98 / - 10.0.0.99
```

Whereabouts tracks addresses in two places, and a pod deleted while the API
server is unhealthy can leave the second one behind:

```sh
kubectl get ippools -n kube-system 10.0.0.0-24 -o yaml   # in-pool allocations
kubectl get overlappingrangeipreservations -n kube-system
```

If a reservation names a pod that no longer exists, delete it and the stuck pod:

```sh
kubectl delete overlappingrangeipreservations -n kube-system <ip>
kubectl delete pod -n dns <stuck-pod>
```

