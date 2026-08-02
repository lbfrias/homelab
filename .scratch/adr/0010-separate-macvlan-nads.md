# Separate Macvlan NADs per Node Architecture

Create separate NetworkAttachmentDefinitions per interface family (`macvlan-lan` for eth0, `macvlan-lan-x86` for eno1). Pods use node affinity to land on compatible nodes.

**Context:** Macvlan CNI requires a hardcoded parent interface name. Our nodes have different interface names:
- RPi (peggy, yelena): `eth0`
- x86 (xialing): `eno1`

## Alternatives considered

- *Single NAD with dynamic interface detection* — Macvlan CNI doesn't support runtime interface discovery
- *Rename eno1 → eth0 via udev* — Fragile, breaks predictable naming convention
- *Label-based NAD selection* — Multus doesn't support selecting NADs based on node labels

**Rationale:**
- Explicit and reliable
- Matches community recommendations for heterogeneous clusters
- Node affinity ensures pods only schedule where their NAD works
