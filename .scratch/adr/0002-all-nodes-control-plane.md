# All Nodes as K3s Control Plane

All 3 nodes run as control plane + workers with HA embedded etcd. This maximizes resource utilization while maintaining high availability.

**Rationale:**
- 3 nodes = quorum of 2, survives 1 node failure
- No wasted resources on dedicated control plane
- etcd on HDDs (/var) reduces SD card wear
