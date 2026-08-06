# Macvlan Networking with Multus

This guide explains how macvlan networking works in this homelab and how to use it in your deployments.

## Overview

By default, Kubernetes pods only have access to the cluster network (Flannel, 10.42.0.0/16). They can reach external services via NAT, but external devices can't initiate connections to pods, and pods can't use L2 protocols like mDNS.

**Multus** solves this by allowing pods to have multiple network interfaces. Combined with **macvlan**, pods can get a "real" IP address on your LAN or VLAN, appearing as a separate device on the network.

```
┌─────────────────────────────────────────────────────────┐
│ Pod                                                     │
│                                                         │
│  eth0 (10.42.x.x)  ←── Flannel (cluster-internal)       │
│  net1 (10.0.0.210) ←── Macvlan (LAN-routable)           │
│                                                         │
└─────────────────────────────────────────────────────────┘
         │                        │
         ▼                        ▼
   K8s Services            Direct L2 access
   Other pods              mDNS, ARP, DHCP
   Egress to internet      Reachable by LAN clients
```

## When to Use Macvlan

Use macvlan when your pod needs:

| Requirement | Example |
|-------------|---------|
| **L2 protocols** | Home Assistant discovering IoT devices via mDNS |
| **Stable LAN IP** | Omada Controller needs a known IP for AP adoption |
| **No SNAT** | DNS server seeing real client IPs for per-client policies |
| **Direct client access** | Services that clients connect to directly (not via ingress) |

**Don't use macvlan** for typical web services — use Ingress or LoadBalancer instead.

## Available NetworkAttachmentDefinitions

NetworkAttachmentDefinitions (NADs) define the macvlan configuration. All NADs use the unified `eno1` interface, configured via the `unified_lan_interface` Ansible role.

### LAN (10.0.0.0/24)

| NAD Name | Parent Interface |
|----------|------------------|
| `macvlan-lan` | eno1 |

IP range: `10.0.0.200` - `10.0.0.239` (reserved for pods, managed manually via pod annotations)

### IoT VLAN (10.0.30.0/24)

| NAD Name | Parent Interface |
|----------|------------------|
| `macvlan-iot` | eno1.300 |

IP range: `10.0.30.200` - `10.0.30.239` (reserved for pods, managed manually via pod annotations)

**Note:** This NAD uses the `sbr` (source-based routing) plugin to handle routing correctly when pods have multiple macvlan interfaces. See [Source-Based Routing](#source-based-routing-sbr-for-multi-interface-pods) for details.

### DNS Infrastructure (10.0.0.0/24)

| NAD Name | Parent Interface |
|----------|------------------|
| `macvlan-dns-technitium` | eno1 |
| `macvlan-dns-dnsdist` | eno1 |

These use Whereabouts IPAM for automatic IP assignment within dedicated ranges.

### VLAN Interface Prerequisite

**VLAN-tagged NADs require the host to have the VLAN interface pre-created.** The IoT NAD references `eno1.300`, which must exist before pods can use it.

If the interface doesn't exist, pods fail with:
```
master "eno1.300" not found
```

The `eno1.300` VLAN interface on xialing is created via the `unified_lan_interface` Ansible role in bootstrap (Step 2). Run `ansible-playbook -e @vars.yaml playbooks/bootstrap.yaml --tags vlan` to create it.

## Usage Examples

### Basic: Static IP (Required)

Add the annotation to request a macvlan interface. You must specify the IP address.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  annotations:
    k8s.v1.cni.cncf.io/networks: |
      [{
        "name": "macvlan-lan",
        "ips": ["10.0.0.210/24"]
      }]
spec:
  containers:
    - name: app
      image: nginx
```

The pod will have two interfaces:
- `eth0` — Flannel (10.42.x.x)
- `net1` — Macvlan (your specified IP)

### Specific IP

Request a specific IP address (include the CIDR suffix for the subnet mask):

```yaml
annotations:
  k8s.v1.cni.cncf.io/networks: |
    [{
      "name": "macvlan-lan",
      "ips": ["10.0.0.210/24"]
    }]
```

### Multiple Networks

Attach to both LAN and IoT VLAN:

```yaml
annotations:
  k8s.v1.cni.cncf.io/networks: |
    [{
      "name": "macvlan-lan",
      "ips": ["10.0.0.210/24"]
    },
    {
      "name": "macvlan-iot",
      "ips": ["10.0.30.210/24"]
    }]
```

### Source-Based Routing (sbr) for Multi-Interface Pods

When a pod has multiple macvlan interfaces on different subnets, routing conflicts can occur. For example, if a pod has:
- `net1`: 10.0.30.150 (IoT VLAN)
- `net2`: 10.0.0.25 (LAN)

The LAN interface adds a route `10.0.0.0/24 dev net2`. When a LAN client (10.0.0.x) connects to the IoT IP (10.0.30.150), replies incorrectly go out `net2` instead of `net1`, breaking the connection.

**Solution:** The `sbr` (source-based routing) CNI plugin creates policy routing rules so traffic *from* a specific IP always uses the correct interface.

The IoT NAD (`macvlan-iot`) includes sbr in its plugin chain:

```json
{
  "plugins": [
    {
      "type": "macvlan",
      "master": "eno1.300",
      "mode": "bridge",
      "ipam": {
        "type": "static",
        "routes": [
          { "dst": "10.0.0.0/16", "gw": "10.0.30.1" }
        ]
      }
    },
    {
      "type": "sbr"
    }
  ]
}
```

**What sbr does:**
1. Creates a dedicated routing table (e.g., table 100)
2. Copies the interface's routes to that table
3. Adds an `ip rule`: `from 10.0.30.150 lookup 100`

**Verify sbr is working:**

```bash
# Check policy rules
kubectl exec -n home deploy/home-assistant -- ip rule
# Output: 32765: from 10.0.30.150 lookup 100

# Check the dedicated routing table
kubectl exec -n home deploy/home-assistant -- ip route show table 100
# Output: 10.0.0.0/16 via 10.0.30.1 dev net1
#         10.0.30.0/24 dev net1 scope link src 10.0.30.150
```

Now replies from 10.0.30.150 to LAN clients correctly route via the IoT gateway instead of directly via the LAN interface.

### Complete Deployment Example

Home Assistant on IoT VLAN (for mDNS discovery) and LAN (for Wake-on-LAN):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: home-assistant
  namespace: home
spec:
  replicas: 1
  selector:
    matchLabels:
      app: home-assistant
  template:
    metadata:
      labels:
        app: home-assistant
      annotations:
        k8s.v1.cni.cncf.io/networks: |
          [{
            "name": "macvlan-iot",
            "ips": ["10.0.30.150/24"]
          },
          {
            "name": "macvlan-lan",
            "ips": ["10.0.0.25/24"]
          }]
    spec:
      containers:
        - name: home-assistant
          image: homeassistant/home-assistant:latest
          # ...
```

## Verifying Macvlan

### Check pod interfaces

```bash
kubectl exec -it <pod> -- ip addr
```

Look for `net1` with your specified IP.

### Test connectivity

From the pod:
```bash
kubectl exec -it <pod> -- ping 10.0.0.1  # Gateway
```

From a LAN device:
```bash
ping 10.0.0.210  # Pod's macvlan IP
```

## Troubleshooting

### Pod stuck in ContainerCreating

Check Multus logs:
```bash
kubectl logs -n kube-system -l app=multus --tail=50
```

Common causes:
- Wrong NAD name in annotation
- Pod scheduled on wrong node (interface mismatch)
- Missing `ips` field in annotation (required for static IPAM)

### Can't reach pod from LAN

- Verify the pod got the macvlan interface: `kubectl exec <pod> -- ip addr`
- Check the IP is in the expected range
- Ensure no firewall blocking on the pod or network

### Multi-interface pod unreachable from some networks

If a pod has multiple macvlan interfaces and is unreachable from certain networks, check for routing conflicts:

```bash
# Check main routing table
kubectl exec <pod> -- ip route

# Check if sbr rules exist
kubectl exec <pod> -- ip rule
```

If a more-specific route (e.g., `10.0.0.0/24 dev net2`) overrides a gateway route, replies may go out the wrong interface. Solution: use the `sbr` plugin in the NAD to enable source-based routing. See [Source-Based Routing](#source-based-routing-sbr-for-multi-interface-pods).

### IP conflicts

With static IPAM, you're responsible for avoiding IP conflicts. Track allocations in `docs/ip-plan.yaml`.

The IP ranges (200-239) are reserved for Kubernetes — don't assign these IPs to other devices on your network.

## Architecture Details

### Components

| Component | Purpose | Location |
|-----------|---------|----------|
| Multus | CNI multiplexer | DaemonSet in kube-system |
| NADs | Network definitions | default namespace |

### How It Works

1. Pod created with `k8s.v1.cni.cncf.io/networks` annotation
2. Kubelet calls Multus (configured as primary CNI)
3. Multus calls Flannel for `eth0`, then the NAD's CNI (macvlan) for `net1`
4. Macvlan plugin creates interface attached to host's physical NIC
5. Static IPAM assigns the IP specified in the pod annotation
6. Pod starts with both interfaces configured

### File Locations (K3s)

- CNI binaries: `/var/lib/rancher/k3s/data/cni/`
- CNI config: `/var/lib/rancher/k3s/agent/etc/cni/net.d/`
