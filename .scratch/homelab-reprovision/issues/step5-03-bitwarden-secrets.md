# Step 5.3 — Bitwarden Secrets Manager Operator

**What to build:** Native Bitwarden SM Kubernetes Operator for syncing secrets.

**Blocked by:** Step 4 (Flux)

**Status:** in-progress

## Tasks

- [x] Add Bitwarden SM Operator HelmRelease to `manifests/infrastructure/controllers/`
- [x] Pin chart version (2.0.3) for Renovate tracking
- [x] Create test `BitwardenSecret` CR template
- [x] Update `docs/secrets.md` with setup instructions
- [x] Set up Renovate for automated version updates
- [ ] Create Machine Account in Bitwarden SM
- [ ] Create auth token secret on cluster (`bw-auth-token`)
- [ ] Fill in test secret IDs and deploy
- [ ] Verify: Secret syncs to K8s Secret

## Secret Flow

1. **Bitwarden SM (web UI):** Create secret with actual value → get secret ID (UUID)
2. **Git repo:** Commit `BitwardenSecret` CR with the ID (safe — no values exposed)
3. **Cluster:** Operator auto-creates K8s Secret with actual value

## Manual Setup Required (one-time per namespace)

```bash
kubectl create secret generic bw-auth-token \
  -n <NAMESPACE> \
  --from-literal=token="<MACHINE_ACCOUNT_TOKEN>"
```

## Files Created

- `manifests/infrastructure/controllers/bitwarden-sm-operator/` — Operator HelmRelease
- `manifests/infrastructure/configs/bitwarden-sm/` — BitwardenSecret resources
- `renovate.json` — Automated dependency updates

## Notes

Using native Bitwarden operator instead of ESO — simpler setup, fewer dependencies, direct integration with Bitwarden SM.
