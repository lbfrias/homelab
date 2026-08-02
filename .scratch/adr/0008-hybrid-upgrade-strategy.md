# Hybrid Upgrade Strategy

In-place patches via Ansible for routine updates. Full reprovision for major OS version upgrades to ensure clean state.

**Rationale:**
- Patches are safe to apply in-place
- Major upgrades risk drift — reprovision ensures clean state
- If automation is solid, reprovision is fast
