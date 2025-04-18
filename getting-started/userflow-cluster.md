# Cluster user

Functionalities of cluster user:

1. Create and join a cluster to the existing network.

2. Implement governance policies for the cluster.

3. Update governance policies for the cluster.

4. Writing stability checker policy

5. Executing stability checker management commands

6. Update cluster data.

7. Deploy addons like container registry, assets registry, inference server etc.


## 1. Create and join a cluster to the existing network.

To join an existing network, you need to know the network's public API URL and make sure you satisfy the prerequisites as per the [instructions provided here](../onboarding-notes/onboarding-cluster.md#prerequisites).

Once the prerequisites are satisfied, follow the steps below:

### Run the following script if one or more nodes in your cluster has GPUs:

Run the following script on all the nodes:

```
cd k8s/onboarding/cluster

sh ./gpu_info.sh
```

This script will add an annotation `gpu.aios/info` to the respective node which will be used by the onboarding script to obtain the GPU info.

### Prepare the config file:

The onboarding directory provides a sample configuration file which can be edited as per the requirements, here is the location of that file, to understand more about the config file, [refer to this documentation.](../onboarding-notes/onboarding-cluster.md)

```sh
cd k8s/onboarding/cluster
cat config.json
```

### Run the cluster onboarding script:

Once you have configured the cluster's `config.json`, run the onboarding script.

Onboarding script validates the `config.json`, obtains the resources of the nodes in the cluster using kubernetes API and uses the parser API of the target network to onboard the cluster. 

Run the onboarding script:

```sh
cd k8s/onboarding/cluster

sh onboard.sh 

./onboard.sh cluster.json --onboard --api-url <parser-server-url>

```

If onboarding is successful, the script should create the cluster entry and also initiate the creation of cluster services infra on the target cluster.

If you want to run the script to inspect the final cluster config data without onboarding:

```
./onboard.sh cluster.json
```

This script will produce another JSON file called `<cluster_id>_populated.json` which contains the processed cluster config file with all the resource info substituted.

---

## 2. Implement governance policies for the cluster

**Pre-check policies** or governance policies are customizable rule sets, authored in Python, that evaluate and authorize actions prior to their execution. These policies serve as a governance mechanism, enabling cluster administrators and developers to enforce cluster-specific constraints and compliance rules.

By implementing pre-check policies, the system ensures that only authorized operations are performed.

The **Cluster Controller** currently supports the following action-level policy hooks:


| **Action**               | **Description**                                                                 |
|--------------------------|---------------------------------------------------------------------------------|
| `add_node`               | Validates whether a new node can be added to the cluster.                      |
| `remove_node`            | Determines if a node is eligible for safe removal from the cluster.            |
| `update_cluster`         | Governs modifications to cluster-wide configurations or metadata.              |
| `scale_block`            | Checks authorization and constraints before scaling block instances.           |
| `add_block`              | Validates if a new block can be provisioned based on policy rules.             |
| `remove_block`           | Ensures safe and policy-compliant decommissioning of a block.                  |
| `add_vdag_controller`    | Authorizes creation of vDAG controller infrastructure.                         |
| `remove_vdag_controller` | Validates and controls teardown of vDAG controller components.                 |
| `block_mgmt`             | Enforces policy rules for administrative actions executed on blocks.           |
| `cluster_mgmt`           | Applies constraints on high-level cluster management operations and commands.  |

To implement pre-check policies for the cluster, [refer to this documentation](../cluster-controller/cluster-controller.md#pre-check-policies).

Once the policy is written, it must be onboarded to the policies system to make it usable. [Refer to this documentation to understand how to onboard a policy.](../policies-system/policies-system.md#preparing-policy-for-onboarding)

Governance policies can be updated using the Cluster Controller Gateway's policy update API. Here is the `curl` command to perform the update:

```sh
curl -X POST "http://<cluster-controller-url>/pre-check-policies/update" \
    -H "Content-Type: application/json" \
    -d '{
        "add_node": "aiosv1.policies.cluster-controller-gateway.add-node:v0.0.12-stable",
        "remove_node": "aiosv1.policies.cluster-controller-gateway.remove-node:v0.0.1-stable",
        "add_cluster": "aiosv1.policies.cluster-controller-gateway.add-cluster:v0.0.2-stable",
        "remove_block": "",
        "add_vdag_controller": ""
    }'
```

In the above example, policy rules for `add_node`, `remove_node`, and `add_cluster` will be updated, and the pre-check policy rules for `remove_block` and `add_vdag_controller` will be removed. Actions not specified in the update request will continue to use the existing policy rules, if any.

---

### Switching between local and remote execution modes of governance policies

> This guide assumes you have `kubectl` installed and access to the cluster's `kubeconfig` file.

The Cluster Controller can execute governance policies either locally or via a remote executor. By default, governance policies execute locally. Based on compute availability, the network admin can switch between local and remote execution modes.

**Switch to remote:**

To switch to remote execution, update the environment variables:  
`POLICY_EXECUTION_MODE=remote` and `POLICY_SYSTEM_EXECUTOR_ID=<executor_id>`  
The `executor_id` must be the remote policy executor ID to which policy execution will be offloaded.

```sh
kubectl patch deployment <cluster_id>-controller \
  -n controllers \
  --type='json' \
  -p='[
    {
      "op": "add",
      "path": "/spec/template/spec/containers/1/env",
      "value": []
    },
    {
      "op": "add",
      "path": "/spec/template/spec/containers/1/env/-",
      "value": {
        "name": "POLICY_EXECUTION_MODE",
        "value": "remote"
      }
    },
    {
      "op": "add",
      "path": "/spec/template/spec/containers/1/env/-",
      "value": {
        "name": "POLICY_SYSTEM_EXECUTOR_ID",
        "value": "<executor_id>"
      }
    }
  ]'
```

To verify:

```sh
kubectl -n controllers get deployment <cluster_id>-controller -o jsonpath="{.spec.template.spec.containers[1].env}"
```

**Switch to local:**

To switch back to local execution, set:  
`POLICY_EXECUTION_MODE=local` and `POLICY_SYSTEM_EXECUTOR_ID=""`

```sh
kubectl patch deployment <cluster_id>-controller \
  -n controllers \
  --type='json' \
  -p='[
    {
      "op": "replace",
      "path": "/spec/template/spec/containers/1/env",
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
kubectl -n controllers get deployment <cluster_id>-controller -o jsonpath="{.spec.template.spec.containers[1].env}"
```
--- 

## Writing stability checker policy:

Stability checker policy is used to check whether the cluster is stable or not by analyzing the data of nodes collected from the cluster periodically.

For writing stability checker policy, [refer to this document](../cluster-controller/cluster-controller.md#stability-checker-policy)

Once the policy is written, it must be onboarded to the policies system to make it usable. [Refer to this documentation to understand how to onboard a policy.](../policies-system/policies-system.md#preparing-policy-for-onboarding)

---

## Executing stability checker management commands

Management command executor provides the following API:

**Endpoint:** `/mgmt`  
**Method:** `POST`  
**Description:** 

This API can be used to execute management command/s against block's services or the cluster controller gateway's stability checker. In this documentation, we will refer to only the stability checker policy management execution.

`block_id` will be 

```bash
curl -X POST http://cluster-controller-service-url/mgmt \
  -H "Content-Type: application/json" \
  -d '{
    "block_id": "",
    "service": "stability_checker",
    "mgmt_action": "<action>",
    "mgmt_data": {}
  }'
```

---

## Update cluster data:

Cluster fields can be updated via cluster controller gateway. Multiple cluster fields can be updated using the update API, the updated data need to pass the pre-check policy - `update_cluster` if specified by the network admin.

```sh
 curl -X PATCH http://<server-url>/clusters/update/<cluster-id> \
     -H "Content-Type: application/json" \
     -d '<updated-fields-json>'
```

Example:

```sh
 curl -X PATCH http://<server-url>/clusters/update/cluster-123 \
     -H "Content-Type: application/json" \
     -d '{
           "clusterMetadata.creator": "prasanna@opevision.ai",
           "regionId": "us-west-2",
           "tags": ["ai-cluster", "llm", "datacenter-grade-1", ]
        }'
```

---

## Deploy addons like container registry, assets registry, inference server etc

For deploying addons, [refer to this documentation](../onboarding-notes/onboarding-cluster.md#deploying-add-on-services)


