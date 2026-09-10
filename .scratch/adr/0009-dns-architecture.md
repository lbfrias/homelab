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

## Forward over DNS-over-TLS, not plain UDP

Forwarders are queried over TLS (`forwarderProtocol: Tls`), not UDP.

Plain UDP has no retransmission: a single dropped packet upstream becomes a
client-visible `SERVFAIL`. This was diagnosed from intermittent failures where
Technitium exhausted *all four* forwarders at once, on names that resolved
fine when tested individually. The decisive measurement was that during a
burst, queries sent **directly to 1.1.1.1 — bypassing dnsdist and Technitium
entirely — failed 14/40, while ICMP to that same address had 0% packet loss**.
Small ICMP survived; UDP/53 was dropped. Because the loss originates upstream
of the cluster, no change to the DNS software could have fixed it; only the
transport could. TLS runs over TCP, which retransmits lost segments.

Forwarders use the `domain (ip)` form:

```
cloudflare-dns.com (1.1.1.1), dns.quad9.net (9.9.9.9)
```

The domain is used for certificate validation; the parenthesised IP avoids a
bootstrap dependency on resolving the forwarder's own name. Technitium rejects
a bare IP under TLS (`Address must be a domain name`).

`cacheMaximumEntries` is raised from the 10000 default to 100000. The default
is small for a whole-network resolver: entries evict quickly, forcing constant
upstream lookups, and every upstream lookup is another chance to hit loss.
`serveStale` is left enabled so expired answers cushion upstream failures.

## Resolver CPU must not be limited

Technitium sets a CPU *request* (200m) and a memory limit, but **no CPU limit**.

CFS throttling freezes the process for up to 100ms per period. For a
latency-sensitive resolver this lands mid-query and produces forwarder
timeouts. Under a 150m limit, technitium-1 was throttled in **57% of
scheduling periods** (`nr_throttled 1331 / nr_periods 2336`). Requests, not
limits, provide a proportional share under node contention, which is what
matters during storms. Verify with:

```sh
kubectl exec -n dns technitium-0 -c technitium -- cat /sys/fs/cgroup/cpu.stat
```

`nr_periods 0` confirms no quota is enforced.

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


## Runbook: intermittent client-visible SERVFAILs

Symptom — pages occasionally fail to resolve, but retrying works, and the
failing names resolve fine when tested by hand.

First, decide whether the fault is *inside* the stack or *upstream* of it.
Query a public resolver directly, bypassing dnsdist and Technitium, and
compare UDP against ICMP to the same address:

```sh
dig +time=2 +tries=1 @1.1.1.1 <name> A     # UDP/53
ping -c 30 1.1.1.1                          # ICMP
```

Run this from a workstation, not a node — the nodes have no `dig` (see
measurement traps).

If DNS fails while ICMP shows 0% loss, the packets are being dropped upstream
of the cluster (router/ISP) and **no change to the DNS stack will fix it**.
DoT forwarding (above) is the mitigation; the remaining suspect is the router
— its CPU, NAT/conntrack table, or DNS interception.

Check whether failures are concentrated or spread:

```sh
for p in technitium-0 technitium-1 technitium-2; do
  echo -n "$p: "
  kubectl logs -n dns $p -c technitium --since=15m | grep -c "^\[.*DNS Server failed to resolve"
done
```

Roughly equal counts mean a shared upstream cause, not a bad backend.
Simultaneous bursts across all three pods rule out per-pod causes such as CPU
throttling.

### Measurement traps

These produced false conclusions during diagnosis:

- **A node cannot query a macvlan VIP hosted on itself.** Probing 10.0.0.99
  from the node running that dnsdist pod fails 100% — this is macvlan
  host↔local-pod isolation, *not* an outage. Always test VIPs from an external
  client.
- **Random/NXDOMAIN probe names manufacture failures.** Flooding uncacheable
  names forces full recursion and triggers upstream rate-limiting, so the test
  creates the failures it measures. Measure with *real* domains; use uncommon
  ones (not `google.com`) so they are not already cached.
- **Technitium blames the last forwarder it tried.** Blame counts mirror
  position in the forwarder list, so the named server is not the culprit. A
  summary line listing all forwarders means all were exhausted.
- **`grep -c "failed to resolve the request"` double-counts**, matching both the
  summary and the exception line. Use `grep -c "^\[.*DNS Server failed to resolve"`
  for client-visible failures.
- Verify tooling exists before trusting a probe: `dig` is **not** installed on
  the nodes, so a probe loop using it reports 100% failure (exit 127).
