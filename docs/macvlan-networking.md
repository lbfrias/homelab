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

NetworkAttachmentDefinitions (NADs) define the macvlan configuration. Different NADs exist because the parent interface name differs between RPi and x86 nodes.

### LAN (10.0.0.0/24)

| NAD Name | Parent Interface | Use On |
|----------|------------------|--------|
| `macvlan-lan` | eth0 | peggy, yelena (RPi) |
| `macvlan-lan-x86` | eno1 | xialing (x86) |

IP range: `10.0.0.200` - `10.0.0.239` (reserved for pods, managed manually via pod annotations)

### IoT VLAN (10.0.30.0/24)

| NAD Name | Parent Interface | Use On |
|----------|------------------|--------|
| `macvlan-iot` | eth0.300 | peggy, yelena (RPi) |
| `macvlan-iot-x86` | eno1.300 | xialing (x86) |

IP range: `10.0.30.200` - `10.0.30.239` (reserved for pods, managed manually via pod annotations)

### VLAN Interface Prerequisite

**VLAN-tagged NADs require the host to have the VLAN interface pre-created.** The IoT NADs reference `eth0.300` / `eno1.300`, which must exist before pods can use them.

If the interface doesn't exist, pods fail with:
```
master "eth0.300" not found
```

The `eno1.300` VLAN interface on xialing is created via the `vlan_interface` Ansible role in bootstrap (Step 2). Run `ansible-playbook -e @vars.yaml playbooks/bootstrap.yaml --tags vlan` to create it.

### Scheduling Limitation

**A pod with a NAD annotation cannot dynamically schedule to all nodes.** Each NAD is tied to a specific parent interface. If a pod lands on an incompatible node, it fails:

```
Failed to create pod sandbox: ... master "eth0" not found
```

The Kubernetes scheduler is unaware of NAD compatibility — you must enforce it yourself.

**Options:**

| Approach | When to Use |
|----------|-------------|
| Node affinity | Single replica on a known node family |
| Separate Deployments | One per node family, each with matching NAD |
| DaemonSet | Run on all nodes, each pod uses local NAD |
| Skip macvlan | Use Service/Ingress if L2 access not required |

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

### Node Affinity (Important!)

Since NADs are tied to specific parent interfaces, you must ensure pods land on the correct node type:

**For RPi nodes (peggy, yelena):**

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: kubernetes.io/arch
                operator: In
                values: ["arm64"]
  containers:
    # ...
```

Then use `macvlan-lan` or `macvlan-iot`.

**For x86 node (xialing):**

```yaml
spec:
  nodeSelector:
    kubernetes.io/hostname: xialing
  containers:
    # ...
```

Then use `macvlan-lan-x86` or `macvlan-iot-x86`.

### Complete Deployment Example

Home Assistant on IoT VLAN with a specific IP:

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
            "ips": ["10.0.30.210/24"]
          }]
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: kubernetes.io/arch
                    operator: In
                    values: ["arm64"]
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
