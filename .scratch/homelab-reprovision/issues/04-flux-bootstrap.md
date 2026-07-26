# 03 — Flux bootstrap + repo structure

**What to build:** Flux installed and reconciling from this repo. The GitOps foundation for all K8s workloads.

**Blocked by:** 03 — K3s HA cluster bootstrap

**Status:** ready-for-agent

- [ ] Create Flux directory structure under `manifests/`
- [ ] Bootstrap Flux pointing to this repo's `manifests/clusters/homelab` path
- [ ] Create placeholder Kustomizations for `infrastructure/` and `apps/`
- [ ] Verify: `flux get all` shows healthy reconciliation
- [ ] Verify: Changes pushed to repo are automatically applied to cluster
