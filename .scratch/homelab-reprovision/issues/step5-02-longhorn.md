# Step 5.2 — Longhorn

**What to build:** Longhorn as the default StorageClass with backups to NAS.

**Blocked by:** Step 4 (Flux)

**Status:** done

## Tasks

- [x] Add Longhorn HelmRelease to `manifests/infrastructure/controllers/`
- [x] Set as default StorageClass
- [x] Configure replica count (2 for HA)
- [x] Set backup target to NFS on xialing
- [x] Verify: Test PVC provisions correctly
- [x] Verify: Longhorn UI accessible
- [x] Verify: Backup target connected

## Notes

- Backup target uses Flux variable substitution: `nfs://${XIALING_IP}:/mnt/data/library/longhorn_backup`
- Variables defined in `manifests/flux-system/cluster-vars.yaml`
