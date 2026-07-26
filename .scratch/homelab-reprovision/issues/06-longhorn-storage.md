# 05 — Longhorn storage + NAS backup target

**What to build:** Longhorn as the default StorageClass with backups to the NAS. PVCs provision automatically, and critical data is backed up.

**Blocked by:** 04 — Flux bootstrap

**Status:** ready-for-agent

- [ ] Add Longhorn HelmRelease to `manifests/infrastructure/controllers/`
- [ ] Set as default StorageClass
- [ ] Configure replica count (2 for HA)
- [ ] Set backup target to NFS on xialing (4TB USB HDD)
- [ ] Verify: Create a test PVC, confirm volume provisions
- [ ] Verify: Longhorn UI accessible
- [ ] Verify: Backup target connected and showing previous backups (if any)
