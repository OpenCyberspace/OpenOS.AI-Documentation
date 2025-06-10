# Cluster Controller gateway:

Cluster controller gateway acts a gateway/interface between the clusters present in the network and the users. Users can invoke/execute multiple cluster level functionalities through this gateway. Management services like Parser can also interact with cluster controller gateway to perform several functionalities.

## Functionalities of cluster controller gateway:

**User-level functionalities:**

1. Query cluster metrics by specifying the cluster ID.

2. Query the metrics of a node by specifying the cluster ID and the node ID.

3. Update the details of the cluster.

4. Delete the cluster entry by specifying the cluster ID.

5. Allocate the cluster infrastructure components by providing the cluster Kubernetes config file ID.

6. De-allocate the cluster infrastructure components by providing the cluster Kubernetes config file and cluster ID.

7. Add/Remove nodes to the cluster.

8. Create/Delete vDAG controller infrastructure components.

9. Execute block level management commands (like parameter updates, updating policy parameters or SDK level parameters) by specifying the block ID.

10. Execute cluster controller level management commands by specifying the cluster ID.

11. Manually scale the block’s instances by specifying the block ID and the cluster ID.

**Parser level functionalities (Parser also interacts with cluster controller to perform several functionalities):**

1. Allocate block.
2. De-allocate block.
3. Dry-run block allocation.
4. Create new cluster entry

---

## Cluster controller gateway components:

Following are the components of the cluster controller gateway:

**1. Internal and external APIs server:**

The cluster controller gateway runs an internal API server that hosts multiple APIs, some APIs are used internally by other management plane services like parser and search server, some APIs are exposed via the management ingress to the external world to be used by the users.

**2. Pre-check policy rule executor:**

All the write and update API calls pass through a pre-check policy rule executor, the network admin who manages the management cluster can implement custom checks that should be executed when a call is made to perform a specific action, each of such pre-check is a policy-rule that will determine whether a corresponding action can be allowed or not. The service maintains a mapping of actions to the policy rule executor, this mapping can be updated anytime using an API.

**3. Block allocator, dry run modules:**

This module will be used internally from the parser flow when allocating a block or dry run. This module loads the block’s resource allocation policy rule, uses the cluster DB API client to filter the clusters matching the block’s cluster allocation criteria and then invokes the policy rule with the matched results, this module also uses the cluster controller client APIs to interact with the cluster controllers of the selected clusters to perform block allocation. This whole process runs asynchronously, so a task will be registered in the global task registry and the task ID is returned to the user, user can query the task DB registry to infer the status of the task. Similarly, this module can also initiate block deletion when the user wants to delete the block.

**4. Cluster infra management module:**

This module is responsible for creating / removing the cluster controller infrastructure, on the target cluster. Here is the flow of infra creation:

a. Initializes the metrics DBs on the target nodes and waits for them to become available.

b.  Creates the metrics reader and writer services that are responsible for writing the collected metrics to the local DBs.

c. Creates metrics collector daemon-set – this service runs on all the nodes by default and is responsible for collecting the hardware metrics.

d. Creates the cluster controller – with the configuration options provided in the cluster entry.

e. Configures the services, ingresses and the ports mapping so the services become available for the external world.

f. Updates the cluster entry in the DB with the URL details of the cluster controller and the metrics services, so it can be used by other services.
              
These steps happen in the background; thus, the module registers a task in the tasks DB registry and returns the task ID, users can query the task ID for the status of this task. The removal process is the exact opposite of the creation process.

**7. Cluster nodes add/remove module:**
Using this module, users can add/remove nodes to the existing cluster, these APIs are used internally by the node onboard script that users run when they want to onboard, deboard the nodes from the cluster. This module proxies the request to the cluster controller of the specified cluster, this is where the actual functionality of the node onboarding/de-boarding happens. 

**8. vDAG controller module:**
This module is responsible for creation, removal and scaling of the vDAG controller on the target cluster, this module simply forwards the request to the respective cluster controller on the cluster on which the given vDAG controller needs to be created, removed or scaled.


**9. Cluster DB, Cluster metrics DB, Cluster controller client modules**
These are the internal client modules that are used by other modules for interacting with cluster controller service, cluster metrics service of the target cluster and the Global cluster DB.

## Setting up the cluster controller gateway:

Cluster controller gateway runs as a part of AIOS management services by default, so the management of the cluster controller gateway falls under the network owner, the network owner can use Kubernetes YAML file to remove the cluster controller gateway.

Cluster level gateway can also be used to implement pre-check for the actions executed through cluster controller gateway by specifying the policies. The policies for these actions need to be specified using the env variable `CLUSTER_CONTROLLER_GATEWAY_ACTIONS_MAP`.

This variable is a JSON which contains “action” as the key and the policy rule URI to be executed as a pre-check for that action as the value. Here are the pre-check options supported (sample “CLUSTER_CONTROLLER_GATEWAY_ACTIONS_MAP” JSON):

```json
{
    "add_node": "aiosv1.policies.cluster-controller-gateway.add-node:v0.0.1-stable",
    "remove_node": "aiosv1.policies.cluster-controller-gateway.remove-node:v0.0.1-stable",
    "add_cluster": "aiosv1.policies.cluster-controller-gateway.add-cluster:v0.0.1-stable",
    "remove_cluster": "aiosv1.policies.cluster-controller-gateway.remove-cluster:v0.0.1-stable",
    "add_block": "aiosv1.policies.cluster-controller-gateway.add-block:v0.0.1-stable",
    "remove_block": "aiosv1.policies.cluster-controller-gateway.remove-block:v0.0.1-stable",
    "update_cluster": "aiosv1.policies.cluster-controller-gateway.update-cluster:v0.0.1-stable",
    "scale_block": "aiosv1.policies.cluster-controller-gateway.scale-block:v0.0.1-stable",
    "block_mgmt": "aiosv1.policies.cluster-controller-gateway.block-mgmt:v0.0.1-stable",
    "cluster_mgmt": "aiosv1.policies.cluster-controller-gateway.cluster-mgmt:v0.0.1-stable"
}
```

The specified policy will be picked up, loaded and executed for the corresponding action, if you want to skip the pre-checks for that action, you can skip the corresponding action entry in the JSON. 

Here’s what each key represents:

1.	`add_node` – The pre-check policy rule to be executed when there is a request to add a new node to a cluster.
2.	`remove_node` – The pre-check policy rule to be executed when there is a request to remove the node from a cluster.
3.	`add_cluster` – The pre-check policy rule to be executed when there is a request to add the new cluster.
4.	`remove_cluster` – The pre-check policy rule to be executed when there is a request to remove the cluster. 
5.	`add_block` – The pre-check policy rule to be executed when the block needs to be created on the cluster.
6.	`remove_block` – The pre-check policy rule to be executed when the block needs to be removed from the cluster.
7.	`update_cluster` – The pre-check policy rule to be executed when the cluster details need to be updated by the user.
8.	`scale_block` – The pre-check policy rule to be executed when the block scale/downscale command needs to be executed on the cluster.
9.	`block_mgmt` – The pre-check policy rule to be executed when the block management command needs to be executed.
10.	`cluster_mgmt` – The pre-check policy rule to be executed when the cluster management command needs to be executed.

### Cluster Controller gateway pre-check policy rule:

The pre-check policy rule should return a dict containing following fields:

```json
{
    "allowed": True,
    "input_data": input_data
}
```

or if the action cannot be allowed:

```json
{
    "allowed": False,
    "input_data": <message or dict containing the reason data of why the action was not allowed>
}
```

The Boolean key “allowed” tells whether the execution of the given action should proceed or not, also the input_data that is passed to the policy rule can be tweaked by the pre-check policy rule, thus the input_data field should contain the updated version of the input dictionary passed to the policy rule, if no modifications are made, return the input_data as it is in this field. Here is the structure of the policy rule that can be used as a pre-check:

```python

class AIOSv1PolicyRule:

 def __init__(self, rule_id, settings, parameters):
   """
     Initializes an AIOSv1PolicyRule instance.

     Args:
      rule_id (str): Unique identifier for the rule.
      settings (dict): Configuration settings for the rule.
      parameters (dict): Parameters defining the rule's behavior.
   """
   self.rule_id = rule_id
   self.settings = settings
   self.parameters = parameters

 def eval(self, parameters, input_data, context):
   """
     Evaluates the policy rule.

     This method should be implemented by subclasses to define the   rule's logic. 
     It takes parameters, input data, and a context object to perform  evaluation.

     Args:
     parameters (dict): The current parameters.
     input_data (any): The input data to be evaluated.
     context (dict): Context (external cache), this can be used for  storing and accessing the state across multiple runs.
   """

   # the input_data dict can be modified by the policy
   # make input_data dict modifications here

   return {
     "allowed": True,
     "input_data": input_data
   }
```

---

## Pre-check policy inputs:

### add_node

```python
{
    "node_data": node_data,
    "cluster_data": cluster,
    "cluster_metrics": metrics
}
```

1. `node_data` - data of the node being added
2. `cluster_data` - data of the cluster to which the node is being added.
3. `cluster_metrics` - metrics of the cluster to which the node is being added.

**Sample node_data**:

```json
{
  "clusterId": "cluster-west-vision-001",
  "id": "node-1",
  "gpus": {
    "count": 2,
    "memory": 32768,
    "gpus": [
      {
        "modelName": "NVIDIA A100",
        "memory": 16384
      },
      {
        "modelName": "NVIDIA A100",
        "memory": 16384
      }
    ],
    "modelNames": [
      "NVIDIA A100"
    ],
    "features": [
      "tensor_cores"
    ]
  },
  "vcpus": {
    "count": 32
  },
  "memory": 131072,
  "swap": 8192,
  "storage": {
    "disks": 2,
    "size": 1048576
  },
  "network": {
    "interfaces": 2,
    "txBandwidth": 0,
    "rxBandwidth": 0
  },
  "tags": [
    "gpu",
    "fp16",
    "production"
  ],
  "nodeMetadata": {
    "vendor": "Supermicro",
    "location": "Rack 2 - DC1",
    "notes": "Installed 2024-12"
  }
}
```

### remove_node

```python
{
    "node_id": node_id,
    "cluster_data": cluster,
    "cluster_metrics": metrics,
    "block_instances": []
}
```

1. `node_id` - id of the node being removed
2. `cluster_data` - data of the cluster to which the node is being removed.
3. `cluster_metrics` - metrics of the cluster to which the node is being removed.
3. `node_instances` - List of block_id that have instances on the current node being removed.

### add_cluster

```python
{
  "cluster_data": cluster
}
```

1. `cluster_data`: Data of the cluster being added

**Sample cluster data**:

```python
{
  "id": "cluster-west-vision-001",
  "regionId": "us-west-2",
  "status": "live",
  "nodes": {
    "count": 2,
    "nodeData": [
      {
        "id": "node-1",
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
        "id": "node-2",
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
  "clusterMetadata": {
    "name": "Sample cluster",
    "vendor": "dma-bangalore",
    "description": "Dedicated to serving large-scale computer vision models in production.",
    "owner": "AI Infrastructure Team",
    "email": "ai-infra@company.net",
    "countries": ["USA", "Canada"],
    "miscContactInfo": {
      "pagerDuty": "https://sample-website/ai-clusters",
      "slack": "#ml-infra"
    },
    "additionalInfo": {
      
    }
  },
  "reputation": 94
}
```

### remove_cluster

```json
{
  "cluster_data": cluster,
  "cluster_metrics": cluster_metrics,
  "cluster_blocks": []
}
```

1. `cluster_data`: Data of the cluster being removed
2. `cluster_metrics`: Metrics of the cluster being removed
3. `cluster_blocks`: List of block ids scheduled on the current cluster

### add_block

```
{
  "block_data": block,
  "cluster_data": cluster,
  "cluster_metrics": cluster_metrics
}
```

### remove_block

```
{
  "block_data": block,
  "cluster_data": cluster,
  "cluster_metrics": cluster_metrics,
  "block_metrics": block_metrics,
  "block_instance_nodes": []
}
```

**Sample block data:**

```json
{
  "blockId": "detector-yolo-88",
  "blockComponentURI": "model.object-detector:2.0.0-stable",
  "component": {
    "componentId": {
      "name": "vllm-chat",
      "version": "1.0.0",
      "releaseTag": "stable"
    },
    "componentType": "model",
    "containerRegistryInfo": {
      "containerImage": "registry.ai-platform.com/models/vllm-external-chat:2.0.0",
      "containerRegistryId": "ai-platform-registry",
      "containerImageMetadata": {
        "author": "llm-team",
        "description": "LLM team"
      },
      "componentMode": "third-party",
      "initContainer": {
        "containerImage": "registry.ai-platform.com/init-containers/vLLM-deployer:2.0.0",
        "containerRegistryId": "ai-platform-registry",
        "containerImageMetadata": {
          "author": "vision-team",
          "description": "Init container to deploy vLLM system"
        }
      }
    },
    "componentMetadata": {
      "usecase": "chat-completion",
      "framework": "external-api",
      "hardware": "external"
    },
    "componentInitData": {
      "vllm_server_url": "http://vllm-service:8080",
      "model_name": "mistralai/Mistral-7B-Instruct-v0.1"
    },
    "componentInitParametersProtocol": {
      "temperature": {
        "type": "number",
        "description": "Sampling temperature",
        "min": 0,
        "max": 2
      }
    },
    "componentInitSettingsProtocol": {
      "max_new_tokens": {
        "type": "number",
        "description": "Maximum number of tokens to generate",
        "min": 1,
        "max": 2048
      }
    },
    "componentInputProtocol": {
      "message": {
        "type": "string",
        "description": "Input message from the user"
      },
      "session_id": {
        "type": "string",
        "description": "Unique identifier for the chat session"
      }
    },
    "componentOutputProtocol": {
      "reply": {
        "type": "string",
        "description": "Chat reply from the vLLM model"
      }
    },
    "componentParameters": {
      "temperature": 0.7
    },
    "componentInitSettings": {
      "max_new_tokens": 256
    },
    "componentManagementCommandsTemplate": {
      "reset": {
        "description": "Clears all stored chat sessions",
        "args": {}
      }
    },
    "tags": [
      "vllm",
      "chat",
      "openai-compatible",
      "mistral",
      "llm"
    ]
  },
  "minInstances": 1,
  "maxInstances": 5,
  "blockInitData": {
    "weights_path": "/models/yolov5s.pt",
    "device": "cuda"
  },
  "initSettings": {
    "batch_size": 4,
    "timeout_ms": 1000
  },
  "parameters": {
    "confidence_threshold": 0.5,
    "nms_threshold": 0.4
  },
  "policyRulesSpec": [
    {
      "values": {
        "name": "clusterAllocator",
        "policyRuleURI": "policy.clusterAllocator.yolo-cluster-selector:v1",
        "parameters": {
          "filter": {
            "clusterMetricsQuery": {
              "logicalOperator": "AND",
              "conditions": [
                {
                  "variable": "cluster.vcpu.load_15m",
                  "operator": "<",
                  "value": "10"
                },
                {
                  "variable": "cluster.gpu.totalFreeMem",
                  "operator": ">",
                  "value": "9000"
                }
              ]
            },
            "clusterQuery": {
              "logicalOperator": "AND",
              "conditions": [
                {
                  "variable": "gpus.count",
                  "operator": ">",
                  "value": "5"
                },
                {
                  "variable": "clusterMetadata.vendor",
                  "operator": "==",
                  "value": "dma-bangalore"
                }
              ]
            }
          }
        },
        "settings": {
          "max_candidates": 3
        }
      }
    },
    {
      "values": {
        "name": "resourceAllocator",
        "policyRuleURI": "policy.resourceAllocator.yolo-gpu-scheduler:v0.0.1-beta",
        "parameters": {
          "gpus": 1,
          "max_gpu_utilization": 0.8,
          "min_gpu_free_memory": 5000
        },
        "settings": {
          "selection_mode": "greedy"
        }
      }
    },
    {
      "values": {
        "name": "loadBalancer",
        "policyRuleURI": "policy.loadBalancer.roundrobin:v0.0.1",
        "parameters": {
          "cache_sessions": true
        },
        "settings": {
          "session_cache_size": 1000
        }
      }
    },
    {
      "values": {
        "name": "stabilityChecker",
        "policyRuleURI": "policy.stabilityChecker.unhealthy-retry:v0.0.1",
        "parameters": {
          "unhealthy_threshold": 3
        },
        "settings": {
          "check_interval_sec": 15
        }
      }
    },
    {
      "values": {
        "name": "autoscaler",
        "policyRuleURI": "policy.autoscaler.gpu-based-autoscaler:v0.0.01",
        "parameters": {
          "target_cpu_utilization": 0.7
        },
        "settings": {
          "scale_up_cooldown": 60,
          "scale_down_cooldown": 120
        }
      }
    }
  ]
}
```

### update_cluster

```python
{
  "cluster_data": cluster
}
```

### scale_block

```python
{
  "block_data": block_data,
  "input": scale_payload
}
```

**Sample scale payload**

```json
{
           "operation": "scale",
           "block_id": <block-id>,
           "instances_count": 2,
           "allocation_data": [
               {
                   "node_id": <node_id>,
                   "gpu_ids": [<gpu-ids>]
               },
               {
                   "node_id": <node_id>,
                   "gpu_ids": [<gpu-ids>]
               }
           ]
}
```

### block_mgmt

```python
{
  "block_data": block,
  "input": {
      "action": "set_iou",
      "payload": {
          "value": 0.9
      }
  } 
}
```

For management command information, refer to this [documentation](https://github.com/OpenCyberspace/OpenOS.AI-Documentation/blob/main/cluster-controller/cluster-controller.md#management-command-executor-apis)

### cluster_mgmt

```python
{
  "cluster_data": cluster,
  "input": {
      "action": action_name,
      "payload": {}
  }
}
```

For management command information, refer to this [documentation](https://github.com/OpenCyberspace/OpenOS.AI-Documentation/blob/main/cluster-controller/cluster-controller.md#management-command-executor-apis)


## API documentation:

### 1. Cluster entry read, update and delete APIs:


#### 1.1 Cluster read API: Get the cluster entry by specifying the cluster ID:


```sh
curl -X GET http://<server-url>/clusters/read/<cluster-id>
```

Example:

 curl -X GET http://<server-url>/clusters/read/cluster-123

---

#### 1.2 Update cluster API: Update one or more fields of the cluster :

Multiple cluster fields can be updated using the update API, the updated data need to pass the pre-check policy if specified by the network admin.
 curl -X PATCH http://<server-url>/clusters/update/<cluster-id> \
     -H "Content-Type: application/json" \
     -d '<updated-fields-json>'

Example:
 curl -X PATCH http://<server-url>/clusters/update/cluster-123 \
     -H "Content-Type: application/json" \
     -d '{
           "clusterMetadata.creator": "prasanna@opevision.ai",
           "regionId": "us-west-2",
           "tags": ["ai-cluster", "llm", "datacenter-grade-1", ]
        }'


#### 1.3 Delete cluster entry API: Delete the cluster by specifying it’s ID

Cluster entry can be removed using the cluster delete API, the delete action needs to be approved by the pre-check policy before it can be deleted.


```sh
 curl -X DELETE http://localhost:5000/clusters/delete/<cluster-id>
```

**Example:**

```sh
curl -X DELETE http://localhost:5000/clusters/delete/cluster-123
```

### 2. Cluster metrics query APIs: Obtain the metrics of clusters and its nodes:

#### 2.1 Get metrics of cluster by specifying the cluster ID: 


```sh
curl -X GET "http://<server-url>/cluster-metrics/<cluster_id>" -H "Content-Type: application/json"
```

**Example:**

```sh
 curl -X GET "http://<server-url>/cluster-metrics/cluster-123" -H "Content-Type: application/json"
```

#### V2.2 Get metrics of a node in the cluster by specifying cluster ID and the node ID:

```sh
curl -X GET " http://<server-url>/cluster-metrics/node/<cluster-id>/<node-id>" -H "Content-Type: application/json"
```

**Example:**

```sh
curl -X GET "http://<server-url>/cluster-metrics/node/cluster-123/node-2" -H "Content-Type: application/json"
```

---

### 3. Scale/Downscale instances of a running block by specifying one or more block IDs:

This API acts as a proxy for the corresponding cluster on which the given block is running, the API can be used to upscale, remove specific instances of the block. 

#### 3.1  Upscale – provide the number of new instances to spin up and the manual resource allocation details (optional)

```sh

  curl -X POST "http://<server-url>/controller/block-scaling/<cluster-id>" \
     -H "Content-Type: application/json" \
     -d '{
           "operation": "scale",
           "block_id": <block-id>,
           "instances_count": 2,
           "allocation_data": [
               {
                   "node_id": <node_id>,
                   "gpu_ids": [<gpu-ids>]
               },
               {
                   "node_id": <node_id>,
                   "gpu_ids": [<gpu-ids>]
               }
           ]
         }'
```

The number of new instances to spin up for the block “block_id” needs to be specified in the “instances_count” field as an integer, the “allocation_data” is optional and allocation data for each instance can be specified as an array, each instance should have the node_id which specifies which node in the cluster the corresponding instance should be scheduled on and the “gpu_ids” list – which will specify what GPUs the instances should be able to use.

This action should be approved by the pre-check policy rule before it is allowed to execute.

**Note:** if allocation_data is not specified, then the block’s resource allocator policy rule will be used to decide these parameters.

**Example:**

```sh
 curl -X POST "http://<server-url>/controller/block-scaling/cluster-123" \
     -H "Content-Type: application/json" \
     -d '{
           "operation": "scale",
           "block_id": "block-787sj",
           "instances_count": 2,
           "allocation_data": [
               {
                   "node_id": "node-1",
                   "gpu_ids": [0, 2]
               },
               {
                   "node_id": "node-5",
                   "gpu_ids": [1, 2]
               }
           ]
         }'
```

---

### 3.2 Downscale:

```sh
 curl -X POST "http://<server-url>/controller/block-scaling/<cluster-id>" \
     -H "Content-Type: application/json" \
     -d '{
           "operation": "downscale",
           "block_id": "<block-id>",
           "instances_list": [<instance-ids>]
         }'
```


One or more instances of the block can be removed using the downscale operation by specifying the block_id and the list of instace_ids that should be removed.
Example:

```sh
curl -X POST "http://<server-url>/controller/block-scaling/<cluster-id>" \
     -H "Content-Type: application/json" \
     -d '{
           "operation": "downscale",
           "block_id": "block-787sj",
           "instances_list": ["instance-7347a"]
         }'
```

This action should be approved by the pre-check policy rule before it is allowed to execute.

---

### 4. Cluster Infra APIs:

Cluster infrastructure components can be created and removed using the cluster infra management APIs, the system does not store the Kubernetes cluster config file, thus the API expects the users to provide the Kubernetes config file data which is used to create and remove the infrastructure components. 

These APIs allows users (cluster managers) to create and remove the cluster whenever needed without having to re-enter or delete the actual cluster entry in the DB.

#### 4.1 Create cluster infrastructure API:

```sh
curl -X POST "http://<server-url>/create-cluster-infra" \
    -H "Content-Type: application/json" \
    -d '{
    "cluster_id": "<cluster-id>",
    "kube_config_data": {}
    }'
```

**Example:**

```sh
curl -X POST "http://<server-url>/create-cluster-infra" \
    -H "Content-Type: application/json" \
    -d '{
    "cluster_id": "cluster-123",
    "kube_config_data": "c3BlY2lhbC1lbmNvZGVkLWt1YmUtY29uZmlnLWRhdGE="
    }'
```

---

#### 4.2 Remove cluster infrastructure API:

```sh
curl -X POST "http://<server-url>/remove-cluster-infra" \
    -H "Content-Type: application/json" \
    -d '{
        "cluster_id": "<cluster-id>",
        "kube_config_data": {}
    }'
```

**Example:**

```sh
curl -X POST "http://<server-url>/remove-cluster-infra" \
-H "Content-Type: application/json" \
-d '{
        "cluster_id": "cluster-123",
        "kube_config_data": {}
    }'
```

---

#### 4.3 Sample python scripts to create/remove infra with Kubernetes config:

The following script below reads the Kubernetes config file from $HOME/.kube/config file by default or takes the user provided config file path, parses the config as a python dictionary and then calls the API with the provided cluster ID.
Script to create cluster infra:

```python

import os
import json
import argparse
import requests

# Constants
DEFAULT_KUBE_CONFIG_PATH = os.path.join(os.path.expanduser("~"), ".kube", "config")
API_URL = "http://<server-url>/create-cluster-infra" # Replace with actual server URL

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
  parser.add_argument("--config", default=DEFAULT_KUBE_CONFIG_PATH,       help="Path to Kubernetes config file")

  args = parser.parse_args()
  kube_config_data = read_kube_config(args.config)
  create_cluster_infra(args.cluster_id, kube_config_data)

if __name__ == "__main__":
  main()

```

To run:

```sh
python create_cluster_infra.py cluster-123 --config /path/to/custom/config
```

Script to remove cluster infra:

```python
import os
import json
import argparse
import requests

# Constants
DEFAULT_KUBE_CONFIG_PATH = os.path.join(os.path.expanduser("~"), ".kube", "config")
API_URL = "http://<server-url>/remove-cluster-infra" # Replace with actual server URL

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
  parser.add_argument("--config", default=DEFAULT_KUBE_CONFIG_PATH,   help="Path to Kubernetes config file")

  args = parser.parse_args()
  kube_config_data = read_kube_config(args.config)
  remove_cluster_infra(args.cluster_id, kube_config_data)

if __name__ == "__main__":
  main()

```

To run:
```python
python remove_cluster_infra.py cluster-123 --config /path/to/custom/config
```

---

### 5. vDAG controller actions:
Cluster controller gateway acts like a proxy for vDAG controller management actions like creating/removing a new vDAG controller, scaling, executing vDAG controller related management actions. The proxy is API provided here, for specific action types and payload structures refer to the vDAG controller documentation.

```sh
curl -X POST "http://<server-url>/vdag-controller/<cluster_id>" \
-H "Content-Type: application/json" \
-d '{
"action": "<action_value>",
"payload": <payload_data>
}'
```

---

### 6. Block management actions:

Cluster controller gateway acts like a proxy for block related management actions, these actions can be used to execute management functionalities on various policies used by the block and its infra components, custom management functionalities supported by the custom instance code built on top of the AIOSv1 instance SDK. The proxy API is provided here, for specific action types and payload structures refer to the Block documentation.

```sh
curl -X POST "http://<server-url>/blocks/mgmt" \
-H "Content-Type: application/json" \
-d '{
    "block_id": "<block_id_value>",
    "service": "<service_name>",
    "action": "<mgmt_command>",
    "payload": <payload_data>
}'
```

---

### 7. Cluster management actions:

Cluster controller gateway acts like a proxy for cluster related management actions, these actions can be used to execute management functionalities on various policies used by the cluster controller and its infra components. The proxy API is provided here, for specific action types and payload structures refer to the Block documentation.

```sh
curl -X POST "http://<server-url>/cluster/mgmt" \
-H "Content-Type: application/json" \
-d '{
    "cluster_id": "<cluster_id_value>",
    "service": "<service_name>",
    "action": "<mgmt_command>",
    "payload": <payload_data>
}'
```

---

### 8. Adding/Removing nodes to/from the cluster:
These API will be called internally by the python script that adds and removes nodes from the cluster, and this is not supposed to be called directly, however, here are the API definitions:

#### 8.1 Add node:

```sh
curl -X POST http://<server-url>/nodes/add-node-to-cluster/<cluster_id> \
    -H "Content-Type: application/json" \
    -d '{}'
```

---

#### 8.2 Remove node:
curl -X GET http://<server-url>/nodes/remove-node-from-cluster/<cluster_id>/<node_id>

---

### 9. Updating the cluster controller gateway pre-check policy rules:

The policy rules used to perform pre-check for various actions can be updated using this API, the network admin can set new policy rules or remove the pre-check constraints for specific actions.

Example:

```sh
curl -X POST "http://<server-url>/pre-check-policies/update" \
    -H "Content-Type: application/json" \
    -d '{
        "add_node": "aiosv1.policies.cluster-controller-gateway.add-node:v0.0.12-stable",
        "remove_node": "aiosv1.policies.cluster-controller-gateway.remove-node:v0.0.1-stable",
        "add_cluster": "aiosv1.policies.cluster-controller-gateway.add-cluster:v0.0.2-stable",
        "remove_cluster": "",
        "add_block": ""
    }'
```

In the above example, policy rules for add_node, remove_node and add_cluster will be updated and the pre-check policy rules for remove_cluster and add_block will be removed. Those actions that are not specified in the update API will continue to use the existing policy rules if any.

