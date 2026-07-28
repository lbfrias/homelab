# Step 4.1 — Flux Operator (Test Branch Workflow)

**What to build:** Flux Operator with ResourceSet/ResourceSetInputProvider for automatic `test/*` branch deployments.

**Blocked by:** Step 4 (Flux)

**Status:** in-progress

## Overview

Enable dynamic branch testing: push to `test/*` → auto-deploy to `test` namespace → delete branch → auto-cleanup.

## Known Limitations

### GitHub API vs SSH Keys
The ResourceSetInputProvider uses the **GitHub REST API** to list branches, not git protocol. This means:
- It cannot reuse Flux's SSH deploy keys (those are for git clone/pull)
- For **public repos**: No authentication needed (60 requests/hour rate limit)
- For **private repos**: Requires a GitHub PAT with `repo` scope

At 5-minute polling intervals, a public repo uses ~288 requests/day, well under the 1,440/day limit.

## Tasks

- [ ] Create `manifests/infrastructure/controllers/flux-operator/` directory
- [ ] Add Flux Operator HelmRepository + HelmRelease
- [ ] Create `test` namespace manifest
- [ ] Create ResourceSetInputProvider for `test/*` branches
- [ ] Create ResourceSet templating GitRepository + Kustomization per branch
- [ ] Create Kustomization to deploy infrastructure/controllers
- [ ] Verify: Push `test/hello` branch with simple ConfigMap
- [ ] Verify: ConfigMap appears in `test` namespace
- [ ] Verify: Delete branch → ConfigMap is pruned
