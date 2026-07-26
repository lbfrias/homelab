# 06 — External Secrets Operator + Bitwarden

**What to build:** ESO pulling secrets from Bitwarden Secrets Manager. Apps can reference ExternalSecrets instead of hardcoding credentials.

**Blocked by:** 04 — Flux bootstrap

**Status:** ready-for-agent

- [ ] Add ESO HelmRelease to `manifests/infrastructure/controllers/`
- [ ] Create ClusterSecretStore pointing to Bitwarden Secrets Manager
- [ ] Create a test ExternalSecret that pulls a secret from Bitwarden
- [ ] Verify: Test secret appears as a K8s Secret in the namespace
- [ ] Update `docs/secrets.md` with Bitwarden Secrets Manager setup instructions
