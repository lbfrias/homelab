# Step 4.1 — Flux Operator (Test Branch Workflow)

**What to build:** Flux Operator with ResourceSet/ResourceSetInputProvider for automatic `test/*` branch deployments.

**Blocked by:** Step 4 (Flux)

**Status:** done

## Overview

Enable dynamic branch testing: push to `test/*` → auto-deploy to `test` namespace → delete branch → auto-cleanup.

## Known Limitations

### GitHub API vs SSH Keys
The ResourceSetInputProvider uses the **GitHub REST API** to list branches, not git protocol. This means:
- It cannot reuse Flux's SSH deploy keys (those are for git clone/pull)
- For **public repos**: No authentication needed (60 requests/hour rate limit)
- For **private repos**: Requires a GitHub PAT with `repo` scope

At 5-minute polling intervals, a public repo uses ~288 requests/day, well under the 1,440/day limit.

### Manual Reconcile for Instant Feedback
During active development, the 5-minute polling interval can feel slow. Use manual reconcile for instant feedback:
```bash
kubectl annotate resourcesetinputprovider test-branches -n flux-system reconcile.fluxcd.io/requestedAt=$(date +%s) --overwrite
```

## Tasks

- [x] Create `manifests/infrastructure/controllers/flux-operator/` directory
- [x] Add Flux Operator HelmRepository + HelmRelease
- [x] Create `test` namespace manifest
- [x] Create ResourceSetInputProvider for `test/*` branches
- [x] Create ResourceSet templating GitRepository + Kustomization per branch
- [x] Create Kustomization to deploy infrastructure/controllers
- [x] Verify: Push `test/hello` branch with simple ConfigMap
- [x] Verify: ConfigMap appears in `test` namespace
- [x] Verify: Delete branch → ConfigMap is pruned
