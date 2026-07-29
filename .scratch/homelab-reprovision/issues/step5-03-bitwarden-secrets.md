# Step 5.3 — Bitwarden Secrets Manager Operator

**What to build:** Native Bitwarden SM Kubernetes Operator for syncing secrets.

**Blocked by:** Step 4 (Flux)

**Status:** ready

## Tasks

- [ ] Add Bitwarden SM Operator HelmRelease to `manifests/infrastructure/controllers/`
- [ ] Create machine account token secret for operator auth
- [ ] Create test `BitwardenSecret` CR referencing SM project
- [ ] Verify: Secret syncs to K8s Secret
- [ ] Update `docs/secrets.md` with setup instructions

## Notes

Using native Bitwarden operator instead of ESO — simpler setup, fewer dependencies, direct integration with Bitwarden SM.
