# Step 5.12 — Monitoring (Prometheus + Grafana)

**What to build:** Prometheus for metrics collection, Grafana for visualization.

**Blocked by:** Step 5.11 (NGINX Ingress) — for Grafana web UI access

**Status:** backlog

## Overview

Deploy a monitoring stack to observe cluster and application health:

1. **Prometheus** — scrape metrics from nodes, K8s components, and apps
2. **Grafana** — dashboards for visualization
3. **node-exporter** — host metrics (CPU, memory, disk, network)

## Tasks

### Prometheus

- [ ] Create `manifests/apps/monitoring/` directory
- [ ] Create namespace.yaml for `monitoring` namespace
- [ ] Deploy Prometheus via Helm chart or manifests
- [ ] Configure ServiceMonitors for:
  - [ ] Node metrics (node-exporter)
  - [ ] K3s components (apiserver, etcd, kubelet)
  - [ ] Longhorn metrics
  - [ ] Application metrics (if exposed)
- [ ] Configure retention and storage (use Longhorn PVC)

### Grafana

- [ ] Deploy Grafana
- [ ] Configure Prometheus as data source
- [ ] Import community dashboards:
  - [ ] Node Exporter Full (ID: 1860)
  - [ ] Kubernetes Cluster (ID: 315)
  - [ ] Longhorn (ID: 16888)
- [ ] Configure Ingress for `grafana.frias.app` (via NGINX Ingress)
- [ ] Add BitwardenSecret for Grafana admin password

### Alerting (Optional, Phase 2)

- [ ] Configure Alertmanager
- [ ] Set up alerts for:
  - [ ] Node down
  - [ ] Disk space low
  - [ ] Pod crash loops
  - [ ] Longhorn volume degraded

## Secrets Required

| Secret Name | Keys | Used By |
|-------------|------|---------|
| `grafana-admin` | `password` | Grafana admin login |

## IP/Ingress

Grafana accessed via NGINX Ingress: `https://grafana.frias.app`

No macvlan IP needed — standard ClusterIP service behind Ingress.

## References

- [Prometheus Operator](https://prometheus-operator.dev/)
- [kube-prometheus-stack Helm chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
