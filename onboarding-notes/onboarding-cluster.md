# Onboarding a Cluster to the AIOS Network

In AIOS, a **network** represents a collection of clusters managed under a unified **cluster controller gateway** and a centralized set of **management services**. All clusters within the network operate under a single administrative domain for orchestration, monitoring, and policy enforcement.

This document outlines the procedure for onboarding a new cluster into an existing AIOS network.

## Network Prerequisites

### Required Ports to Expose for External Node Joining (if the cluster allows external nodes over the internet to join)

When a node joins a Kubernetes cluster using `kubeadm join` or needs to interact with the control plane over the internet, certain ports must be exposed on the control plane node. These include ports used by the API server, kubelet, and other control plane components.

#### 1. Control Plane Node (Master)

| **Port**     | **Protocol** | **Component**           | **Description**                                                                 |
|--------------|--------------|--------------------------|---------------------------------------------------------------------------------|
| 6443         | TCP          | kube-apiserver           | Primary API server port. Required for `kubeadm join` and all cluster operations |
| 2379–2380    | TCP          | etcd                     | Used for etcd server and peer communication (required if running etcd yourself) |
| 10250        | TCP          | kubelet                  | Used by the control plane to communicate with kubelet on nodes                  |
| 10257        | TCP          | kube-controller-manager  | Used internally for metrics (usually not needed externally)                     |
| 10259        | TCP          | kube-scheduler           | Used internally for metrics (usually not needed externally)                     |

#### 2. Worker Node

| **Port**     | **Protocol** | **Component**           | **Description**                                                               |
|--------------|--------------|--------------------------|-------------------------------------------------------------------------------|
| 10250        | TCP          | kubelet                  | Required for communication with the control plane                             |
| 30000–32767  | TCP/UDP      | NodePort Services        | Port range for services exposed via NodePort type                             |

---

#### NodePort Range (Services Exposed Externally)

If you are exposing Kubernetes services using `type: NodePort`, the default port range is:

- **30000–32767** (TCP/UDP)

You must ensure this range is allowed in your firewall settings if external clients need access to these services.

---

#### Minimum Required Ports for External Node Join

If your goal is only to allow worker nodes from outside the internal network to **join the cluster**, you must at minimum expose the following on the **control plane node**:

- **6443/TCP** – Kubernetes API Server (required for all cluster communications)
- **10250/TCP** – Kubelet API (used for node-to-control plane communication)

Optional based on your setup:

- **2379–2380/TCP** – etcd (if you're managing etcd directly and not using managed Kubernetes)
- **10257/TCP**, **10259/TCP** – for metrics or debugging (optional)

---

#### Security Considerations

Exposing these ports to the public internet is **high risk**. It is highly recommended to:

- Use a **VPN** (e.g., WireGuard, Tailscale) to tunnel control plane access.
- Restrict access using **firewall rules** (allow only trusted IPs).

#### VPNs

For security, it is always recommended to set up a VPN if the cluster allows external nodes to join. This ensures that only nodes with valid VPN credentials can join the network, preventing malicious access.

VPN setup and management must be handled outside of AIOS. For guidance, refer to the [WireGuard](https://www.wireguard.com/) or [Tailscale](https://tailscale.com/) documentation.

---

### Required Ports to Expose for Self-Hosted Clusters (When External Nodes Are Not Allowed to Join)

NodePorts: **30000–32767**

----

## Prerequisites

Before onboarding a cluster to the AIOS network, ensure the following prerequisites are met:

### 1. Install Kubernetes

AIOS is built to operate **exclusively on Kubernetes**, but it is **agnostic to the Kubernetes version**.

#### Requirements

- A fully functional Kubernetes cluster with `kubectl` access (must be set up using `kubeadm`)
- Helm v3.x or later installed on the administrator's machine
- Network access to NodePorts (required for gateway exposure)

> Note: AIOS does not impose constraints on the Kubernetes version, but it requires the cluster to be API-compliant.

---

### 2. Install Ambassador Gateway

The Ambassador Gateway serves as the ingress point for communication between the AIOS controller and the onboarded cluster. It must be installed within the cluster and exposed over a static NodePort (`32000`).

#### Step-by-Step Installation Using Helm

**Add the Helm Repository:**

```bash
helm repo add datawire https://app.getambassador.io
helm repo update
```

**Create a Namespace:**

```bash
kubectl create namespace ambassador
```

**Install Ambassador with NodePort Exposure:**

```bash
helm install ambassador datawire/ambassador \
  --namespace ambassador \
  --set service.type=NodePort \
  --set service.nodePorts.http=32000 \
  --set service.nodePorts.https=32000
```

This installs Ambassador into the `ambassador` namespace and exposes it via NodePort on port `32000`.

**Verify Installation:**

```bash
kubectl get svc -n ambassador
kubectl get pods -n ambassador
```

**Ensure the following:**

- The `ambassador` service is of type `NodePort`
- Both HTTP and HTTPS ports are mapped to `32000`
- All Ambassador pods are in a `Running` or `Ready` state

---

### 3. Label Kubernetes Nodes with `nodeID`

Each node in the cluster must be labeled with a unique `nodeID` to identify it within the cluster.

#### Label Format

```bash
nodeID="<unique-id>"
```

#### Steps to Label a Node

1. List nodes:

```bash
kubectl get nodes
```

2. Label a node:

```bash
kubectl label node <node-name> nodeID=<unique-id>
```

3. Verify:

```bash
kubectl get nodes --show-labels
```

To update an existing label:

```bash
kubectl label node <node-name> nodeID=<new-id> --overwrite
```

---

## Onboarding Cluster Entry to the Network

Once the cluster is ready to be onboarded, prepare the cluster onboarding JSON document. All sizes must be specified in `MB`.

(Assuming the cluster initially has two nodes with the following configuration):

```json
{
  "id": "cluster-west-vision-001",
  "regionId": "us-west-2",
  "status": "live",
  "nodes": {
    "count": 2,
    "nodeData": [
      {
        "id": "node-1", // should match nodeID
        "gpus": {
          "count": 2,
          "memory": 32768,
          "gpus": [
            { "modelName": "NVIDIA A100", "memory": 16384 },
            { "modelName": "NVIDIA A100", "memory": 16384 }
          ],
          "features": ["fp16", "tensor_cores"],
          "modelNames": ["NVIDIA A100"]
        },
        "vcpus": { "count": 32 },
        "memory": 131072,
        "swap": 8192,
        "storage": {
          "disks": 2,
          "size": 1048576
        },
        "network": {
          "interfaces": 2,
          "txBandwidth": 10000,
          "rxBandwidth": 10000
        }
      },
      {
        "id": "node-2", // should match nodeID
        "gpus": {
          "count": 1,
          "memory": 16384,
          "gpus": [
            { "modelName": "NVIDIA V100", "memory": 16384 }
          ],
          "features": ["fp16"],
          "modelNames": ["NVIDIA V100"]
        },
        "vcpus": { "count": 16 },
        "memory": 65536,
        "swap": 4096,
        "storage": {
          "disks": 1,
          "size": 524288
        },
        "network": {
          "interfaces": 1,
          "txBandwidth": 5000,
          "rxBandwidth": 5000
        }
      }
    ]
  },
  "gpus": {
    "count": 3,
    "memory": 49152
  },
  "vcpus": {
    "count": 48
  },
  "memory": 196608,
  "swap": 12288,
  "storage": {
    "disks": 3,
    "size": 1572864
  },
  "network": {
    "interfaces": 3,
    "txBandwidth": 15000,
    "rxBandwidth": 15000
  },
  "config": {
    "allowExternalNodes": true,
    "policyExecutorId": "policy-exec-007",
    "policyExecutionMode": "local",
    "customPolicySystem": {
      "name": "AdvancedPolicyRunner",
      "version": "2.1.0"
    },
    "publicHostname": "cluster-west-vision-001.company.net",
    "useGateway": true,
    "actionsPolicyMap": {
      "onScaleUp": "evaluate-gpu-availability",
      "onFailure": "notify-admin"
    }
  },
  "tags": ["gpu", "production", "ml", "vision", "us-west"],

  // this is just an example, feel free to add more fields
  "clusterMetadata": {
    "name": "Sample cluster",
    "description": "Dedicated to serving large-scale computer vision models in production.",
    "owner": "AI Infrastructure Team",
    "email": "ai-infra@company.net",
    "countries": ["USA", "Canada"],
    "miscContactInfo": {
      "pagerDuty": "https://sample-website/ai-clusters",
      "slack": "#ml-infra"
    },
    "additionalInfo": {
      "kubeadmVersion": "v28.0.1",
      "kubectlVersion": "v28.0.01",
      // kubeadm join command
      "joinCommand": ""
    }
  },
  "reputation": 0
}
```

---

### Call the `createCluster` Parser API

The `createCluster` API from the Parser should be invoked to register the cluster in the network. This step **does not** create the AIOS infrastructure on the cluster; it only registers the cluster in the database. (For more details, refer to the Parser documentation.)

```sh
curl -X POST http://<parser-host>:<port>/api/createCluster \
  -H "Content-Type: application/json" \
  -d @cluster.json
```

> **Note:** The cluster will be registered only if it passes the network’s cluster creation pre-check policy.

---

## Create Cluster Infrastructure

Once the cluster is set up, its infrastructure can be created using the Cluster Controller Gateway API:

```sh
curl -X POST "http://<cluster-controller-gateway-url>/create-cluster-infra" \
  -H "Content-Type: application/json" \
  -d '{ 
    "cluster_id": "cluster-west-vision-001", 
    "kube_config_data": {} 
}'
```

The `kube_config_data` must contain the JSON representation of the `kubeconfig` file, usually located at `$HOME/.kube/config`. This file contains credentials to access the Kubernetes API.

---

### Sample Python Script

```python
import os
import json
import argparse
import requests

# Constants
DEFAULT_KUBE_CONFIG_PATH = os.path.join(os.path.expanduser("~"), ".kube", "config")
API_URL = "http://<server-url>/create-cluster-infra"  # Replace with actual server URL

def read_kube_config(file_path):
    try:
        with open(file_path, "r") as f:
            return f.read()
    except Exception as e:
        print(f"Error reading Kubernetes config file: {e}")
        exit(1)

def create_cluster_infra(cluster_id, kube_config_data):
    payload = {
        "cluster_id": cluster_id,
        "kube_config_data": kube_config_data
    }

    response = requests.post(API_URL, json=payload)

    if response.status_code == 200:
        print("Cluster creation scheduled successfully:", response.json())
    else:
        print("Error creating cluster:", response.json())

def main():
    parser = argparse.ArgumentParser(description="Create Cluster Infrastructure via API")
    parser.add_argument("cluster_id", help="Cluster ID")
    parser.add_argument("--config", default=DEFAULT_KUBE_CONFIG_PATH,
                        help="Path to Kubernetes config file")

    args = parser.parse_args()
    kube_config_data = read_kube_config(args.config)
    create_cluster_infra(args.cluster_id, kube_config_data)

if __name__ == "__main__":
    main()
```

---

## Services

Below is a list of NodePorts that will be exposed once all services are deployed:

### Services Exposed via NodePort

| **Service Name**                 | **Node Port** | **Description**                                                    |
|----------------------------------|---------------|--------------------------------------------------------------------|
| Cluster Service Endpoint         | `32300`       | Cluster Controller Service URL                                     |
| Cluster Metrics Endpoint         | `32301`       | Metrics server for the cluster                                     |
| Block Transactions Service       | `32302`       | API to query block-related data                                    |
| Public Controller Gateway        | `32000`       | Public ingress gateway                                             |
| Management Commands Endpoint     | `32303`       | Endpoint for sending management commands to blocks and the cluster |

### Services Exposed via Gateway Suffix

| **Service Name**                 | **Gateway Suffix**       | **Description**                                                    |
|----------------------------------|---------------------------|--------------------------------------------------------------------|
| Cluster Service Endpoint         | `/controller`            | Cluster controller service URL                                     |
| Cluster Metrics Endpoint         | `/metrics`               | Metrics server URL for the cluster                                 |
| Block Transactions Service       | `/blocks`                | API to query block-related data                                    |
| Public Controller Gateway        | `/`                      | Public ingress gateway                                             |
| Management Commands Endpoint     | `/mgmt`                  | Endpoint for sending management commands to blocks and the cluster |

---

### Optional Endpoints

#### NodePort-Based (Optional, if these services are deployed)

| **Service Name**                 | **Node Port** | **Description**                                                    |
|----------------------------------|---------------|--------------------------------------------------------------------|
| Policy Executor Server           | `32380`       | Executes policy functions and jobs                                 |
| Assets Registry Server           | `32390`       | Stores and queries model or policy assets                          |
| Model Split Runner Server        | `32286`       | Executes model splitting operations                                |

#### Gateway-Based (Dynamically registered based on IDs)

| **Service Name**                 | **Gateway Suffix**            | **Description**                                                    |
|----------------------------------|--------------------------------|--------------------------------------------------------------------|
| Adhoc Inference Server           | `/tasks/<server-id>`          | API server for submitting inference tasks                          |
| Block Endpoint                   | `/block/<block-id>`           | Block services exposed via this endpoint                           |
| vDAG Controller                  | `/<vdag-controller-id>`       | vDAG controller service endpoint                                   |

---

## Uninstalling the Cluster Infrastructure

Cluster infrastructure can be removed using the following script:

```python
import os
import json
import argparse
import requests

DEFAULT_KUBE_CONFIG_PATH = os.path.join(os.path.expanduser("~"), ".kube", "config")
API_URL = "http://<server-url>/remove-cluster-infra"  # Replace with actual server URL

def read_kube_config(file_path):
    try:
        with open(file_path, "r") as f:
            return f.read()
    except Exception as e:
        print(f"Error reading Kubernetes config file: {e}")
        exit(1)

def remove_cluster_infra(cluster_id, kube_config_data):
    payload = {
        "cluster_id": cluster_id,
        "kube_config_data": kube_config_data
    }
    response = requests.post(API_URL, json=payload)
    if response.status_code == 200:
        print("Cluster removal scheduled successfully:", response.json())
    else:
        print("Error removing cluster:", response.json())

def main():
    parser = argparse.ArgumentParser(description="Remove Cluster Infrastructure via API")
    parser.add_argument("cluster_id", help="Cluster ID")
    parser.add_argument("--config", default=DEFAULT_KUBE_CONFIG_PATH,
                        help="Path to Kubernetes config file")
    args = parser.parse_args()
    kube_config_data = read_kube_config(args.config)
    remove_cluster_infra(args.cluster_id, kube_config_data)

if __name__ == "__main__":
    main()
```

---

## Deploying Add-on Services

Add-on services are optional. You can choose to install any of the following:

1. **Container Registry** – Stores container images. You can contribute space to the community.
2. **Assets Registry** – Stores models, policy code, and general assets.
3. **Adhoc Inference Server** – Accepts and routes inference tasks to blocks and vDAGs.
4. **Policy Executor** – Executes and schedules policy jobs and functions.
5. **Model Splitter Service** – Performs model splitting operations.

---

### Deploying Container Registry

Set up an OCI-compliant container image store (e.g., `registry:2`) and register it in the container registries metadata registry.

```sh
cd v1_deploy/container-registry

kubectl create ns registry
kubectl create -f pv.yaml
kubectl create -f pvc.yaml
kubectl create -f deployment.yaml
kubectl create -f svc.yaml
```

Register the registry:

```sh
curl -X POST http://registry-db-url/container_registry \
     -H "Content-Type: application/json" \
     -d @entry.json
```

---

### Deploying Assets Registry

To install Ceph (for large clusters):

```sh
cd k8s/installer/addons/assets-registry
./deploy_ceph_operator.sh
./deploy_ceph_obs.sh
./create_ceph_user.sh
```

To install MinIO (for simpler setups):

```sh
cd k8s/installer/addons/assets-registry
./install_minio.sh
```

Install and deploy the DB and registry:

```sh
./install_db.sh 3 10Gi
./install_registry.sh
```

Register the registry:

```sh
curl -X POST "http://<assets-network-registry>/asset_registry" \
     -H "Content-Type: application/json" \
     -d '{
           "asset_registry_name": "ResNet Models registry",
           "asset_registry_metadata": { "framework": "PyTorch" },
           "asset_registry_tags": ["image-classification"],
           "asset_registry_public_url": "http://192.168.0.106:31700"
         }'
```

---

### Deploying Policy Executor

1. Create executor document (`executor.json`)
2. Register it:

```sh
curl -X POST http://<policy-system-url>/executor \
     -H "Content-Type: application/json" \
     -d @executor.json
```

3. Create/remove infra:

```sh
# Create
curl -X POST http://<policy-system-url>/executor/executor-001/create-infra \
  -H "Content-Type: application/json" \
  -d '{ "cluster_config": <kubeconfig JSON> }'

# Remove
curl -X DELETE http://<policy-system-url>/executor/executor-001/remove-infra \
  -H "Content-Type: application/json" \
  -d '{ "cluster_config": { "provider": "kubernetes" } }'
```

Service will be available at:
- External: `http://<public-ip>:30900`
- Internal: `http://executor-001.policies-system.svc.cluster.local:10250`

---

### Deploying Model Splitter

```sh
curl -X POST http://<server-url>:8001/split-runner \
  -H "Content-Type: application/json" \
  -d '{
    "cluster_k8s_config": <kubeconfig JSON>,
    "split_runner_public_host": "192.168.0.106",
    "split_runner_metadata": {"key": "value"},
    "split_runner_tags": ["tag1", "tag2"]
  }'
```

If successful, NodePort `31701` will be exposed.

---

### Deploying Adhoc Inference Server

**Optional: install logging DB**

```sh
cd k8s/installer/addons/inference-server
./install_db.sh 5Gi 3 logging username password
```

**Install the inference server**

```sh
./install_inference_server.sh server-pr-home-123 192.168.0.101 logging-primary.inference-server.svc.cluster.local 5432 username password
```

**Register the inference server**

```sh
curl -X POST <server-url>/inference_server \
     -H "Content-Type: application/json" \
     -d '{
           "inference_server_name": "server-pr-home-123",
           "inference_server_metadata": {
               "region": "us-east-1",
               "gpu_available": true
           },
           "inference_server_tags": ["NLP", "Transformer"],
           "inference_server_public_url": "http://192.168.0.101:31500"
         }'
```

NodePorts exposed:
- `31500`: gRPC
- `31501`: REST