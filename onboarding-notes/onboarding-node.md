# Onboarding Nodes

Nodes can be added to the cluster at any time. You can add nodes from the same network or from an external network, provided the necessary network configurations are in place.

---

## Network Prerequisites (for External Nodes Joining over the Public Internet)

The cluster will allow external nodes to join only if `config.allowExternalNodes=true`.

| **Port**     | **Protocol** | **Component**        | **Description**                                                              |
|--------------|--------------|----------------------|------------------------------------------------------------------------------|
| 10250        | TCP          | kubelet              | Required for communication with the control plane                            |
| 30000–32767  | TCP/UDP      | NodePort Services    | Port range for services exposed using NodePort type                          |

**Note:** A static IP is mandatory.

**Note:** If the cluster uses a VPN, the node must first join the cluster’s VPN. This is **not** managed by AIOS.

---

## Network Prerequisites (for Internal Network Nodes)

If the node resides in the same internal network as the master node and is governed by a common firewall configuration, expose:

- **30000–32767** (TCP/UDP)

---

## Prerequisites

1. `kubeadm` must be installed on the node, and its version should match the target cluster. Refer to `clusterMetadata.additionalInfo.kubeadmVersion`.

2. The network must be configured correctly as per the guidelines above.

---

## Obtain the Cluster Data

Fetch the target cluster's metadata from the cluster registry.

**Endpoint:** `/clusters/:id`  
**Method:** `GET`  
**Description:** Fetches the cluster document using its unique `id`.

**Example:**

```bash
curl -X GET http://<cluster-registry-url>/clusters/cluster-west-vision-001
```

---

## Onboarding the Node

1. Extract the `joinCommand` from the `clusterMetadata.additionalInfo.joinCommand` field of the cluster document.

2. Use the onboarding script provided in the `installer/` directory:

```sh
cd installer/

python join_node.py \
  --node-id=<node-id> \
  --cluster-id=<cluster-id> \
  --api-url=<parser-api-url> \
  --kubeadm-join-cmd="<join command>"
```

If the cluster's and network's `add_node` pre-check policies pass, the node will be successfully onboarded.

> **Note:** The Cluster Controller will automatically label the node with the correct `nodeID` once it successfully joins the cluster.