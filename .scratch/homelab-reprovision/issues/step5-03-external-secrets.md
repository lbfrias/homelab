# Step 5.3 — External Secrets

**What to build:** ESO pulling secrets from Bitwarden Secrets Manager.

**Blocked by:** Step 4 (Flux)

**Status:** ready

## Tasks

- [ ] Add ESO HelmRelease to `manifests/infrastructure/controllers/`
- [ ] Create ClusterSecretStore pointing to Bitwarden
- [ ] Create test ExternalSecret
- [ ] Verify: Test secret appears as K8s Secret
- [ ] Update `docs/secrets.md` with setup instructions
