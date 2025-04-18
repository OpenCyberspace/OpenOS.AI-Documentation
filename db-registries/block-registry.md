# Blocks registry:
Blocks registry stores the information of all the blocks that are currently running across all the clusters in the network.

Here is the schema of the block:

```js
const BlockSchema = new mongoose.Schema({
    // Unique identifier for the block
    id: { type: String, required: true, unique: true },

    // the component URI the block is running - taken from component registry
    componentUri: { type: String },

    // The component data of the block - copied from component registry
    component: { type: mongoose.Schema.Types.Mixed },

    // same as componentUri + block-id
    blockUri: { type: String },

    // Human-readable or structured metadata about the block - copied from component
    blockMetadata: { type: mongoose.Schema.Types.Mixed },

    // Policy configuration or rules tied to the block
    policies: { type: mongoose.Schema.Types.Mixed },

    // The cluster data of the block, copied as it is from the cluster registry
    cluster: { type: mongoose.Schema.Types.Mixed },

    // Data used to initialize the block during deployment/startup
    blockInitData: { type: mongoose.Schema.Types.Mixed },

    // Initialization settings (env vars, args, flags, etc.)
    initSettings: { type: mongoose.Schema.Types.Mixed },

    // Parameters required to configure the block's runtime behavior
    parameters: { type: mongoose.Schema.Types.Mixed },

    // Minimum number of instances this block should maintain
    minInstances: { type: Number, required: false },

    // Maximum number of instances allowed for scaling
    maxInstances: { type: Number, required: false },

    // Input interface specification (protocol - follows a template - copied from component)
    inputProtocol: { type: mongoose.Schema.Types.Mixed },

    // Output interface specification (protocol - copied from component)
    outputProtocol: { type: mongoose.Schema.Types.Mixed }
});

```

Example:

```json
{
  "id": "block-object-detector-001",
  "componentUri": "",
  "component": {},
  "blockUri": "",
  "blockMetadata": {},
  "policies": {
        "resourceAllocator": {
            "policyRuleURI": "policies.resource_allocator.standard:latest",
            "parameters": {},
            "settings": {}
        },
        "loadBalancer": {
            "policyRuleURI": "policies.load_balancer.gateway.load_balancer_sep2:v0.0.1-beta",
            "parameters": {},
            "settings": {}
        },
        "loadBalancerMapper": {
            "policyRuleURI": "policies.load_balancer.mapper.loadbalancer_mapper_oct1:v1.2.0",
            "parameters": {},
            "settings": {}
        },
        "assignment": {
            "policyRuleURI": "policies.assignment.default_strategy:v1.0.3",
            "parameters": {},
            "settings": {}
        },
        "stabilityChecker": {
            "policyRuleURI": "policies.health.stability_checker:v0.1.0",
            "parameters": {},
            "settings": {}
        },
        "autoscaler": {
            "policyRuleURI": "policies.autoscaler.basic_auto_scaler:v0.3.5",
            "parameters": {},
            "settings": {}
        },
        "accessRulesPolicy": {
            "policyRuleURI": "policies.access.control.access_rules_policy:v2.0.0",
            "parameters": {},
            "settings": {}
        }
    },
  "cluster": {},
  "blockInitData": {},
  "initSettings": {},
  "parameters": {},
  "minInstances": 1,
  "maxInstances": 5,
  "inputProtocol": {},
  "outputProtocol": {}
}

```

### Creating a block:
For creating the block, refer to the documentation of Parser.

### Block registry APIs:

**Endpoint:** `/blocks`  
**Method:** `GET`  
**Description:**  
Fetches all block documents in the database.

**Example curl Command:**

```bash
curl -X GET http://<server-url>/blocks
```

---

**Endpoint:** `/blocks/:id`  
**Method:** `GET`  
**Description:**  
Fetches a single block document by its unique `id`.

**Example curl Command:**

```bash
curl -X GET http://<server-url>/blocks/block-object-detector-001
```

---

**Endpoint:** `/blocks/:id`  
**Method:** `PUT`  
**Description:**  
Updates a block document by its `id` using MongoDB-style update syntax in the request body.

**Example curl Command:**

```bash
curl -X PUT http://<server-url>/blocks/block-object-detector-001 \
  -H "Content-Type: application/json" \
  -d '{
    "$set": {
      "blockMetadata.description": "Updated description for object detection block",
      "minInstances": 2
    }
  }'
```

---

**Endpoint:** `/blocks/:id`  
**Method:** `DELETE`  
**Description:**  
Deletes the block document with the specified `id`.

**Example curl Command:**

```bash
curl -X DELETE http://<server-url>/blocks/block-object-detector-001
```

---

**Endpoint:** `/blocks/query`  
**Method:** `POST`  
**Description:**  
Queries block documents using a MongoDB-style filter provided in the JSON body. Supports standard MongoDB operators such as `$eq`, `$gt`, `$in`, etc. Optional `options` can be passed for sorting, pagination, etc.

**Example curl Command:**

```bash
curl -X POST http://<server-url>/blocks/query \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "cluster.reputation": { "$gt": 90 },
      "policies.autoscaler.policyRuleURI": { "$ne": "" }
    },
    "options": {
      "sort": { "id": 1 },
      "limit": 10
    }
  }'
```