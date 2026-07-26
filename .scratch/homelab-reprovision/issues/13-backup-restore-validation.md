# 13 — Backup restore validation

**What to build:** A tested, documented restore procedure that proves backups actually work.

**Blocked by:** 05 — Longhorn, 12 — Media stack (need a volume to test with)

**Status:** ready-for-agent

- [ ] Verify Longhorn backup target shows existing backups (or create new ones)
- [ ] Test restore: Delete prowlarr PVC (non-critical)
- [ ] Restore prowlarr volume from Longhorn backup
- [ ] Verify prowlarr starts with restored data
- [ ] Update `docs/restore-guide.md` with any lessons learned
- [ ] Document restore test results and date
