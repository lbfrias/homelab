# Step 5.12 — Observability (Prometheus, Grafana, Loki, Alloy)

**What to build:** Metrics collection, log aggregation, and visualization.

**Blocked by:** Step 5.11 (NGINX Ingress) — for Grafana web UI access

**Status:** done

## Overview

Deploy an observability stack in `manifests/infrastructure/observability/`:

1. **kube-prometheus-stack** — Prometheus, Grafana, Alertmanager, node-exporter
2. **Loki** — Log aggregation
3. **Alloy** — Unified collector (metrics and logs)

## Tasks

### kube-prometheus-stack (Helm)

- [x] Create `manifests/infrastructure/observability/` directory
- [x] Create namespace.yaml for `monitoring` namespace
- [x] Deploy kube-prometheus-stack via HelmRelease
- [x] Configure ServiceMonitors for K8s components
- [x] Configure retention and storage (use Longhorn PVC via storageclass.yaml)

### Loki

- [x] Deploy Loki via HelmRelease
- [x] Configure storage backend

#### Loki Canary (optional)

Loki Canary is a DaemonSet that writes synthetic log entries to Loki and reads them back to measure write/read latency and detect missing logs. Useful for production reliability monitoring.

**To enable:**
1. Set `lokiCanary.enabled: true` in `loki.yaml` (top-level, not under `monitoring`)
2. Add ServiceMonitor scraping for canary metrics
3. Import Loki Canary Grafana dashboard

### Alloy

- [x] Deploy Alloy via HelmRelease
- [x] Configure to collect metrics and logs
- [x] Ship to Prometheus and Loki

### Grafana

- [x] Deployed as part of kube-prometheus-stack
- [x] Configure Prometheus as data source (automatic)
- [x] Configure Loki as data source
- [x] Configure Ingress for `monitoring.frias.app` (ingress.yaml)
- [x] Add BitwardenSecret for Grafana admin password

## IP/Ingress

Grafana accessed via NGINX Ingress: `https://monitoring.frias.app`

No macvlan IP needed — standard ClusterIP service behind Ingress.

## Directory Structure

```
manifests/infrastructure/observability/
├── namespace.yaml
├── storageclass.yaml
├── bitwardensecret.yaml
├── kube-prometheus-stack.yaml   # HelmRelease
├── loki.yaml                    # HelmRelease
├── alloy.yaml                   # HelmRelease
├── ingress.yaml                 # Grafana ingress
└── kustomization.yaml
```

## Alerting (Future)

- [ ] Configure Alertmanager rules
- [ ] Set up alerts for: node down, disk low, crash loops, volume degraded

## References

- [Prometheus Operator](https://prometheus-operator.dev/)
- [kube-prometheus-stack Helm chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [Grafana Loki](https://grafana.com/oss/loki/)
- [Grafana Alloy](https://grafana.com/oss/alloy/)
