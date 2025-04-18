# Network user

Network user is responsible for setting up and managing the network by creating the management network.

Here are the functionalities of a network user:

1. Setting up the management cluster  
2. Implementing governance policies  
3. Updating the governance policies  

## 1. Setting up the management cluster

To create a new network, you need to install the management services responsible for managing various life-cycle activities of a network on the management cluster. The management cluster serves as the centralized Kubernetes cluster on which all management services and databases will be deployed. The network's core APIs will be exposed from this cluster.

For a detailed, step-by-step guide to set up your own network, [refer to this documentation.](../installation/installation.md)

The script provided below automates the entire process of deploying a network. Before running the script, ensure that the prerequisites are met. To understand the prerequisites, [refer to this documentation](../installation/installation.md#prerequisites).

Once the prerequisites are met, execute the following steps:

**1. Label the nodes in the management cluster where you want to deploy the registry databases:**

```sh
kubectl label node <registry-node-name> registry=yes --overwrite
```

**2. Label the nodes in the management cluster where you want to deploy the metrics databases:**

```sh
kubectl label node <metrics-node-name> metrics=yes --overwrite
```

**3. Edit the deployment configurations if needed** (optional)  
The deployment files are provided in the `k8s/installer` directory. Feel free to modify any of the deployment files as per your requirements.

```sh
cd k8s/installer
```

**4. Configure the replication parameters for the databases:**

The configuration parameters are provided in the `k8s/install_mgmt_services.sh` file. Modify the following parameters:

```sh
# Number of replicas to be created for registries
REGISTRY_REPLICAS=3

# Size of each storage replica
REGISTRY_SIZE="10Gi"

# Number of replicas to be created for the metrics DB
METRICS_REPLICAS=3

# Size of each metrics DB replica
METRICS_SIZE="10Gi"
```

**5. Run the deployment script:**

To deploy all services automatically, run the following script:

```sh
cd k8s/install_mgmt_services.sh

sh ./install_mgmt_services.sh
```

Here's the corrected section with the **pre-check policies** rewritten as a table while keeping the rest of the formatting intact:

---

## 2. Implementing the governance policies (pre-check policies)

Governance policies (pre-check policies) are executed for every action performed on a cluster or block via the cluster controller gateway of the network. Using this mechanism, the network owner can implement policies that govern what types of actions are allowed or disallowed.

**Supported Pre-check Policies:**

| **Policy Name**     | **Description**                                                                 |
|---------------------|---------------------------------------------------------------------------------|
| `add_node`          | Executed when there is a request to add a new node to a cluster.                |
| `remove_node`       | Executed when there is a request to remove a node from a cluster.              |
| `add_cluster`       | Executed when there is a request to add a new cluster.                          |
| `remove_cluster`    | Executed when there is a request to remove a cluster.                           |
| `add_block`         | Executed when a block needs to be created on a cluster.                         |
| `remove_block`      | Executed when a block needs to be removed from a cluster.                       |
| `update_cluster`    | Executed when cluster details are updated by the user.                          |
| `scale_block`       | Executed when a block scale/downscale command is issued on a cluster.          |
| `block_mgmt`        | Executed when a block management command is executed.                           |
| `cluster_mgmt`      | Executed when a cluster management command is executed.                         |

To write a pre-check policy, [refer to this documentation](../cluster-controller-gateway/cluster-cotroller-gateway.md#setting-up-the-cluster-controller-gateway).

Once the policy is written, it must be onboarded to the policies system to make it usable. [Refer to this documentation to understand how to onboard a policy.](../policies-system/policies-system.md#preparing-policy-for-onboarding)

--- 

## 3. Updating the governance policies

Governance policies can be updated using the Cluster Controller Gateway's policy update API. Here is the `curl` command to perform the update:

```sh
curl -X POST "http://<cluster-controller-gateway-url>/pre-check-policies/update" \
    -H "Content-Type: application/json" \
    -d '{
        "add_node": "aiosv1.policies.cluster-controller-gateway.add-node:v0.0.12-stable",
        "remove_node": "aiosv1.policies.cluster-controller-gateway.remove-node:v0.0.1-stable",
        "add_cluster": "aiosv1.policies.cluster-controller-gateway.add-cluster:v0.0.2-stable",
        "remove_cluster": "",
        "add_block": ""
    }'
```

In the above example, policy rules for `add_node`, `remove_node`, and `add_cluster` will be updated, and the pre-check policy rules for `remove_cluster` and `add_block` will be removed. Actions not specified in the update request will continue to use the existing policy rules, if any.

### Switching between local and remote execution modes of governance policies

> This guide assumes you have `kubectl` installed and you have cluster's `kubeconfig` file.

The Cluster Controller Gateway can execute governance policies either locally or via a remote executor. By default, governance policies execute locally. Based on compute availability, the network admin can switch between local and remote execution modes.

**Switch to remote:**

To switch to remote execution, update the environment variables:  
`POLICY_EXECUTION_MODE=remote` and `POLICY_SYSTEM_EXECUTOR_ID=<executor_id>`  
The `executor_id` must be the remote policy executor ID to which policy execution will be offloaded.

```sh
kubectl patch deployment cluster-controller-gateway \
  -n services \
  --type='json' \
  -p='[
    {
      "op": "add",
      "path": "/spec/template/spec/containers/0/env",
      "value": []
    },
    {
      "op": "add",
      "path": "/spec/template/spec/containers/0/env/-",
      "value": {
        "name": "POLICY_EXECUTION_MODE",
        "value": "remote"
      }
    },
    {
      "op": "add",
      "path": "/spec/template/spec/containers/0/env/-",
      "value": {
        "name": "POLICY_SYSTEM_EXECUTOR_ID",
        "value": "<executor_id>"
      }
    }
  ]'
```

To verify:

```sh
kubectl -n services get deployment cluster-controller-gateway -o jsonpath="{.spec.template.spec.containers[*].env}"
```

**Switch to local:**

To switch back to local execution, set:  
`POLICY_EXECUTION_MODE=local` and `POLICY_SYSTEM_EXECUTOR_ID=""`

```sh
kubectl patch deployment cluster-controller-gateway \
  -n services \
  --type='json' \
  -p='[
    {
      "op": "replace",
      "path": "/spec/template/spec/containers/0/env",
      "value": [
        {
          "name": "POLICY_EXECUTION_MODE",
          "value": "local"
        },
        {
          "name": "POLICY_SYSTEM_EXECUTOR_ID",
          "value": ""
        }
      ]
    }
  ]'
```

To verify:

```sh
kubectl -n services get deployment cluster-controller-gateway -o jsonpath="{.spec.template.spec.containers[*].env}"
```