# Step 5.10 — Backup/Restore Validation

**What to build:** Tested, documented restore procedure proving backups work.

**Blocked by:** Step 5.2 (Longhorn), Step 5.9 (Media Stack)

**Status:** ready

## Tasks

- [ ] Verify Longhorn backup target shows backups
- [ ] Test restore: Delete prowlarr PVC (non-critical)
- [ ] Restore prowlarr volume from backup
- [ ] Verify prowlarr starts with restored data
- [ ] Update `docs/restore-guide.md`
- [ ] Document restore test results and date
