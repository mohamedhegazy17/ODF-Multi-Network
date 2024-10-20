# ODF-Multi-Network 🚀

## Overview
This guide provides instructions for configuring **OpenShift Data Foundation (ODF)** using **Multus** with multiple network interfaces. The goal is to enable seamless traffic management between public and cluster networks for optimized data transfer and internal replication.

## Prerequisites ✅

Ensure the following basic requirements are met before starting the configuration:

1. **OpenShift hosts** must be able to route traffic to the Multus public network.
2. **Pods on the Multus public network** must be able to route traffic to OpenShift hosts.

### Key Considerations ⚠️
- **Host Requirements**:
  - Each OpenShift host must have an interface connected to the Multus public network (`public-network-interface`).
  - The `public-network-interface` should have a valid IP address and an appropriate routing table.
  - Ensure no IP address conflicts exist between nodes and pods on the Multus public network.

- **Pod Network Configuration**:
  - Configure the **NetworkAttachmentDefinition** to ensure that IPAM routes traffic destined for nodes through the appropriate network.
  
- **Network Technology Compatibility**:
  - Both node and pod network configurations should use the same network technology, typically **Macvlan** for consistent connectivity.

- **IP Range Planning**:
  - Plan IP ranges and routes for both nodes and pods in sync with the **NetworkAttachmentDefinition** and node configurations.

> **Note**: OpenShift recommends using the **NMState** operator’s `NodeNetworkConfigurationPolicies` for host network configuration.

---

## Requirements for Multus Configuration ⚙️

### Network Interface Consistency:
- All **OpenShift storage** and **worker nodes** should have the same interface name for the public network and be connected to the same underlying network.
- The **cluster network interface** should be uniform across storage nodes but does not need to be present on worker nodes.
- Ensure that each network interface supports at least **10Gbps** speeds.

### VLAN/Subnet Setup:
- Separate VLANs or subnets are required for each network interface used for the public or cluster network.

---

## Multus Deployment 🚧

### Step 1: Configure Node Network Policies (NNCP) 🛠️

You need to create `NodeNetworkConfigurationPolicy` (NNCP) on both **worker** and **storage** nodes.

- **Public Network (Pod-to-Storage Traffic)**: Create NNCP on **all worker nodes**.
- **Cluster Network (ODF Internal Traffic)**: Create NNCP on **all storage nodes**.

#### Sample NNCP Configuration:
```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: ceph-public-net-shim-compute-0
  namespace: openshift-storage
spec:
  nodeSelector:
    node-role.kubernetes.io/worker: ""
    kubernetes.io/hostname: compute-0
  desiredState:
    interfaces:
      - name: odf-pub-shim
        description: Shim interface used to connect host to OpenShift Data Foundation public Multus network
        type: mac-vlan
        state: up
        mac-vlan:
          base-iface: eth0
          mode: bridge
          promiscuous: true
        ipv4:
          enabled: true
          dhcp: false
          address:
            - ip: 192.168.252.1 # STATIC IP for compute-0
              prefix-length: 22
    routes:
      config:
        - destination: 192.168.0.0/16
          next-hop-interface: odf-pub-shim
```

### Step 2: Create Network Attachment Definitions (NAD) 📡
- If you're using two interfaces **(one for storage and one for the public network)**, you'll need to create two separate `NetworkAttachmentDefinitions` (NADs):

- **One for the cluster network**.
- **One for the public network**.

#### Sample NAD Configuration:
```yaml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: public-net
  namespace: openshift-storage
spec:
  config: '{
      "cniVersion": "0.3.1",
      "type": "macvlan", 
      "master": "eth0",
      "mode": "bridge",
      "ipam": {
        "type": "whereabouts",
        "range": "192.168.0.0/16",
        "exclude": [
           "192.168.252.0/22"
        ],
        "routes": [
          {"dst": "192.168.252.0/22"}
        ]
      }
    }'
```

### Conclusion🎯
- This setup ensures that OpenShift Data Foundation can efficiently route traffic over multiple networks using Multus. By carefully configuring the NodeNetworkConfigurationPolicies and NetworkAttachmentDefinitions, you can achieve high-performance, isolated networking for ODF operations.
