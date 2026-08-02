# Containers Instead of VMs for HA/Omada

Home Assistant and Omada Controller run as K8s containers with Multus + macvlan networking instead of VMs. This fits the GitOps model and reduces storage overhead.

**Context:** Home Assistant and Omada Controller were previously VMs.

**Rationale:**
- VMs have storage overhead (expanding disk images)
- xialing moving to smaller SSD (120GB)
- Containers are lighter, fit K8s GitOps model
- macvlan solves mDNS/multicast requirements
