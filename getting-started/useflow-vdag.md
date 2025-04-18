Here is the revised version with improved grammar, clarity, and sentence structure:

---

# vDAG Deployment Flow

This documentation outlines the steps involved in creating a vDAG, deploying its controller, and configuring related policies.

## Overview of Steps

1. Prepare the policies required for vDAG creation  
2. Prepare the vDAG specification  
3. Deploy the vDAG  
4. Create the vDAG controller  

---

## 1. Prepare the Policies Required for vDAG Creation

### vDAG Controller Policies

The following policies can be optionally configured in the vDAG controller:

| **Policy Name**           | **Description**                                                                                                                                                                                                                      | **Reference**                                                                 |
|---------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------|
| **Quota Checker Policy**   | Allows the vDAG provider to limit inference API usage, either globally or based on the `session_id`.                                                                                                                               | [Quota Checker Policy](../vdag-controller/vdag-controller.md#quota-checker-policy) |
| **Quality Checker Policy** | Periodically samples requests and responses to audit vDAG performance. It can either implement internal auditing logic or forward the samples to an external QA system for manual review.                                           | [Quality Checker Policy](../vdag-controller/vdag-controller.md#quality-checker-policy) |
| **Health Checker Policy**  | Monitors the health and performance of all blocks within the vDAG using periodically collected health check data.                                                                                                                    | [Health Checker](../vdag-controller/vdag-controller.md#health-checker) |

---

### Node-Level Policies in vDAG

Each node in a vDAG can optionally define the following policies:

| **Policy Name**            | **Description**                                                                                                                                                                                                                                                                 | **Reference**                                                              |
|----------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------|
| **Pre-processing Policy**  | Defines custom logic to run **before** the main computation of a node. Triggered when a computation task is first detected. Typically used for data validation, enrichment, or transformation before passing it to the core logic.        | [Pre/Post Policies](../parser/vdag.md#pre-and-post-processing-policies)   |
| **Post-processing Policy** | Defines custom logic to run **after** the main computation of a node. Used to transform, validate, or enrich results before forwarding them to the next stage or returning them to the client.                                           | [Pre/Post Policies](../parser/vdag.md#pre-and-post-processing-policies)   |
| **Assignment Policy**      | Selects the most suitable block for a vDAG node. Works alongside the system’s filtering mechanism to evaluate candidate blocks and returns the final block ID to assign.                                                                | [Assignment Policy](../parser/vdag.md#assignment-policy)                  |

--- 

Here’s the revised version with improved grammar, clarity, and consistency in tone and formatting:

---

## 2. Prepare the vDAG Specification

To learn how to write a vDAG specification, [refer to this documentation](../parser/vdag.md).

You can view an example specification here:

```bash
cd services/applications/examples/vdag-pose-estimation

cat vdag.json
```

---

## 3. Deploy the vDAG

To deploy a vDAG, submit its specification to the Parser service of the network using the following command:

```bash
curl -X POST http://$SERVER_URL/api/createvDAG \
  -H "Content-Type: application/json" \
  -d @./vdag.json
```

---

## 4. Create a vDAG Controller

Creating a vDAG does **not** automatically create its controller. A vDAG controller can be created or removed at any time based on when the vDAG needs to accept inference tasks.

To create a controller, invoke the API provided by the Cluster Controller Gateway:

- **Endpoint:** `/vdag-controller/<cluster_id>`  
- **Method:** `POST`  
- **Description:**  
  Proxies a controller management request to the gateway of the specified cluster. The request is forwarded to the `/vdag-controller` endpoint inside the target cluster.

### Example: Creating a vDAG Controller

```bash
curl -X POST http://<server-url>/vdag-controller/<cluster-id> \
  -H "Content-Type: application/json" \
  -d '{
    "action": "create_controller",
    "payload": {
      "vdag_controller_id": "<controller-id>",
      "vdag_uri": "<vdag-uri>",
      "config": {
        "policy_execution_mode": "local",
        "replicas": 2
      }
    }
  }'
```

| Parameter                | Type     | Description                                                                 |
|--------------------------|----------|-----------------------------------------------------------------------------|
| `cluster_id`             | string   | ID of the target cluster where the controller should be created.            |
| `action`                 | string   | Controller management action. Use `"create_controller"` to create one.      |
| `vdag_controller_id`     | string   | Unique identifier for the vDAG controller.                                  |
| `vdag_uri`               | string   | URI of the vDAG to be associated with the controller.                       |
| `config`                 | object   | Controller configuration. Includes fields like `policy_execution_mode` and `replicas`. |
| `policy_execution_mode`  | string   | Mode for policy execution (e.g., `"local"` or `"remote"`).                  |
| `replicas`               | integer  | Number of replicas to create for the controller.                            |

---

### Example: Removing a vDAG Controller

To remove the vDAG controller when it's no longer needed:

```bash
curl -X POST http://<server-url>/vdag-controller/<cluster_id> \
  -H "Content-Type: application/json" \
  -d '{
    "action": "remove_controller",
    "payload": {
      "vdag_controller_id": "<controller-id>"
    }
  }'
```

| Parameter              | Type     | Description                                                                 |
|------------------------|----------|-----------------------------------------------------------------------------|
| `cluster_id`           | string   | ID of the cluster where the controller is deployed.                         |
| `action`               | string   | Management action. Use `"remove_controller"` to delete the controller.      |
| `vdag_controller_id`   | string   | Unique identifier of the controller to be removed.                          |

