# Step 5.2 — Longhorn

**What to build:** Longhorn as the default StorageClass with backups to NAS.

**Blocked by:** Step 4 (Flux)

**Status:** ready

## Tasks

- [ ] Add Longhorn HelmRelease to `manifests/infrastructure/controllers/`
- [ ] Set as default StorageClass
- [ ] Configure replica count (2 for HA)
- [ ] Set backup target to NFS on xialing
- [ ] Verify: Test PVC provisions correctly
- [ ] Verify: Longhorn UI accessible
- [ ] Verify: Backup target connected
