# Directory Structure and Renovate Grouping

Infrastructure manifests are organized by function, not deployment mechanism. "Controllers" contains operators that reconcile Kubernetes resources (cert-manager, Longhorn, Multus). "Observability" is a sibling directory for metrics/logs (Prometheus, Loki, Alloy) — these are workloads, not controllers. Apps with related components share a parent directory (e.g., `apps/network/dns/` for dnsdist + Technitium). Renovate uses leaf-level grouping for debuggability: each component gets its own PR, making it easy to identify which update broke something.

## Considered Options

1. **Group all infrastructure under "controllers"** — simpler structure, but semantically incorrect (observability isn't a controller)
2. **Group Renovate PRs by parent directory** — fewer PRs, but harder to debug when updates break things
3. **Separate directories + leaf-level grouping** — more PRs, but each is isolated and easy to revert

We chose option 3 because debuggability matters more than PR volume for infrastructure that runs 24/7.

## Consequences

- Flux has separate Kustomizations: `infrastructure-controllers`, `infrastructure-observability`, `infrastructure-configs`, `apps`
- Renovate creates individual PRs for each infrastructure component (alloy, loki, kube-prometheus-stack, cert-manager, etc.)
- Apps still group at the leaf directory level (e.g., all containers in `apps/media/jellyfin/` update together)
