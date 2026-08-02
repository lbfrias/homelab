# Flux for GitOps

Hybrid approach: Ansible for node provisioning (Steps 1-4), Flux for K8s workloads (Step 5). Git is the source of truth with auto-reconciliation.

**Rationale:**
- Separation of concerns (infra vs apps)
- Git as source of truth
- Auto-reconciliation corrects drift
- `kubectl apply` as emergency fallback
