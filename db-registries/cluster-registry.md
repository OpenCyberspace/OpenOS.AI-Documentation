# Clusters registry:
Clusters registry contains the information about the on-boarded clusters in the network.

Here is the schema of the cluster entry in clusters registry:

```js
const clusterSchema = new Schema({
    // Unique ID for the cluster
    id: { type: String, required: true, unique: true },

    // Optional region or network ID where the cluster is deployed
    regionId: { type: String, required: false },

    status: { type: String, required: true },

    // Aggregated and per-node information for all nodes in the cluster
    nodes: {
        // Total number of nodes in the cluster
        count: { type: Number, required: true },
        // Detailed info for each individual node
        nodeData: [{
            // Unique ID for the node
            id: { type: String, required: true },
            // GPU details for the node
            gpus: {
                // Number of GPUs in the node
                count: { type: Number, required: true },
                // Total GPU memory in MB
                memory: { type: Number, required: true },
                // List of GPU models with individual memory sizes
                gpus: [{
                    modelName: { type: String, required: true }, // GPU model name
                    memory: { type: Number, required: true }     // Memory per GPU in MB
                }],
                // Optional GPU features (e.g., CUDA versions)
                features: [String],
                // List of distinct GPU model names
                modelNames: [String]
            },
            // Virtual CPU details
            vcpus: {
                count: { type: Number, required: true } // Number of vCPUs in the node
            },
            // Total memory in MB
            memory: { type: Number, required: true },
            // Total swap space in MB
            swap: { type: Number, required: true },
            // Storage info per node
            storage: {
                disks: { type: Number, required: true }, // Number of disks
                size: { type: Number, required: true }   // Total storage size in MB
            },
            // Network interface stats per node
            network: {
                interfaces: { type: Number, required: true },   // Number of network interfaces
                txBandwidth: { type: Number, required: true },  // Transmit bandwidth (MBps)
                rxBandwidth: { type: Number, required: true }   // Receive bandwidth (MBps)
            }
        }]
    },

    // Total GPU stats across all nodes
    gpus: {
        count: { type: Number, required: true },   // Total number of GPUs in the cluster
        memory: { type: Number, required: true }   // Total GPU memory in MB
    },

    // Total vCPU count across the cluster
    vcpus: {
        count: { type: Number, required: true }
    },

    // Total memory across the cluster in MB
    memory: { type: Number, required: true },

    // Total swap space across the cluster in MB
    swap: { type: Number, required: true },

    // Aggregated storage details for the cluster
    storage: {
        disks: { type: Number, required: true }, // Total number of disks
        size: { type: Number, required: true }   // Total storage size in MB
    },

    // Aggregated network configuration
    network: {
        interfaces: { type: Number, required: true },  // Total number of interfaces
        txBandwidth: { type: Number, required: true }, // Total TX bandwidth
        rxBandwidth: { type: Number, required: true }  // Total RX bandwidth
    },

    // Configuration used by the cluster controller
    config: {
        type: new Schema({
            policyExecutorId: { type: String, required: false, default: "" },      // Optional custom policy executor ID
            policyExecutionMode: { type: String, required: false, default: "local" }, // Execution mode for policies
            customPolicySystem: { type: Schema.Types.Mixed, required: false },     // Any custom policy logic/plugin
            publicHostname: { type: String, required: true },                      // Public hostname for the cluster
            useGateway: { type: Boolean, required: false, default: true },         // Whether to use a gateway for access
            actionsPolicyMap: { type: Schema.Types.Mixed, required: false },       // Mapping for policy actions
            // URLs to internal/external services in the cluster
            urlMap: {
                controllerService: { type: String, required: true },     // URL for controller service
                metricsService: { type: String, required: true },        // URL for metrics collection
                blocksQuery: { type: String, required: true },           // URL for querying blockchain blocks
                publicGateway: { type: String, required: true },         // Public-facing gateway URL
                parameterUpdater: { type: String, required: true }       // URL for model/cluster parameter updates
            }
        }),
        required: true
    },

    // List of user-defined tags or labels
    tags: { type: [String], required: true },

    // Human-readable metadata about the cluster
    clusterMetadata: { 
        type: new Schema({
            name: { type: String, required: true },                   // Friendly name of the cluster
            description: { type: String, required: true },            // Purpose or use-case of the cluster
            owner: { type: String, required: true },                  // Who owns or manages the cluster
            email: { type: String, required: false },                 // Optional contact email
            countries: { type: [String], required: false },           // Countries associated with this cluster
            miscContactInfo: { type: Schema.Types.Mixed, required: false },  // Additional contact or support info
            additionalInfo: { type: Schema.Types.Mixed, required: false }    // Any extra metadata as needed
        }),
        required: true
    },

    // Reputation score or reliability indicator for the cluster (not yet used anywhere in the system)
    reputation: { type: Number, required: false }
});

```
Example:

```json
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
    },

    // these fields are populated by the system:
    "urlMap": {
      "controllerService": "http://cluster-west-vision-001.company.net:32000/controller",
      "metricsService": "http://cluster-west-vision-001.company.net:32000/metrics",
      "blocksQuery": "http://cluster-west-vision-001.company.net:32000/blocks",
      "publicGateway": "http://cluster-west-vision-001.company.net:32000",
      "parameterUpdater": "http://cluster-west-vision-001.company.net:32000/mgmt"
    }
  },
  "tags": ["gpu", "production", "ml", "vision", "us-west"],
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
      
    }
  },
  "reputation": 94
}
```

### Creating a cluster:
For creating the cluster, refer to the documentation of Parser.

### Cluster registry APIs:

**Endpoint:** `/clusters/:id`  
**Method:** `GET`  
**Description:**  
Fetches a single cluster document by its unique `id`.

**Example curl Command:**

```bash
curl -X GET http://<server-url>/clusters/cluster-west-vision-001
```

---

**Endpoint:** `/clusters/:id`  
**Method:** `PUT`  
**Description:**  
Updates a cluster document by its `id` using the payload provided in the request body. The body should use MongoDB-style update syntax.

**Example curl Command:**

```bash
curl -X PUT http://<server-url>/clusters/cluster-west-vision-001 \
  -H "Content-Type: application/json" \
  -d '{
    "$set": {
      "tags": ["gpu", "updated"],
      "reputation": 97
    }
  }'
```

---

**Endpoint:** `/clusters/:id`  
**Method:** `DELETE`  
**Description:**  
Deletes the cluster document with the specified `id`.

**Example curl Command:**

```bash
curl -X DELETE http://<server-url>/clusters/cluster-west-vision-001
```

---

**Endpoint:** `/clusters/query`  
**Method:** `POST`  
**Description:**  
Queries cluster documents using a MongoDB-style filter provided in the request body. Supports standard MongoDB operators such as `$eq`, `$gt`, `$in`, etc.

**Example curl Command:**

```bash
curl -X POST http://<server-url>/clusters/query \
  -H "Content-Type: application/json" \
  -d '{
    "gpus.count": { "$gte": 2 },
    "clusterMetadata.countries": { "$in": ["USA"] }
  }'
```
---