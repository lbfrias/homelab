# Step 0 — Network Prep (Pre-Work)

**What to build:** Simplify network to flat 10.0.0.0/24 before tearing down cluster.

**Blocked by:** None

**Status:** done

## Why

- Omada controller runs on K8s — tearing down cluster kills network management
- Flat network is simpler to troubleshoot during bootstrap
- Can re-add VLANs and Omada controller after cluster is stable

## Tasks

- [x] Set Omada router (TL-ER605) to standalone mode
- [x] Set Omada switch (TL-SG2008P) to standalone mode
- [x] Set Omada AP (TL-EAP225) to standalone mode
- [x] Configure flat network: 10.0.0.0/24, gateway 10.0.0.1
- [x] Set static IPs/DHCP reservations for nodes
- [x] Verify: All devices reachable, internet works
