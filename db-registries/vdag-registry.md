# vDAG Registry
Clusters registry contains the information about the on-boarded clusters in the network.

Here is the schema of a vDAG:
```python
@dataclass
class NodeObject:
    # Label or name of the node
    nodeLabel: str = ''
    # Type/category of the node (e.g., model, preprocessor)
    nodeType: str = ''
    # Unique vDAG identifier this node belongs to
    vdagURI: str = ''
    # Policy rule used to assign this node
    assignmentPolicyRule: Dict[str, Any] = field(default_factory=dict)
    # Preprocessing logic or policies before node execution
    preprocessingPolicyRule: Dict[str, Any] = field(default_factory=dict)
    # Postprocessing logic or policies after node execution
    postprocessingPolicyRule: Dict[str, Any] = field(default_factory=dict)
    # Parameters specific to the model in the node
    modelParameters: Dict[str, Any] = field(default_factory=dict)
    # Output protocol specification for this node
    outputProtocol: Dict[str, Any] = field(default_factory=dict)
    # Input protocol specification for this node
    inputProtocol: Dict[str, Any] = field(default_factory=dict)
    # Mapping of input/output fields across connected nodes
    IOMap: List[Dict[str, Any]] = field(default_factory=list)
    # Optional manual block ID if node is explicitly placed
    manualBlockId: str = field(default_factory=str)


@dataclass
class vDAGObject:
    # Name of the vDAG
    vdag_name: str = ''
    # Version information of the vDAG (version and release tag)
    vdag_version: Dict[str, str] = field(default_factory=lambda: {'version': '', 'release-tag': ''})
    # Combined URI for the vDAG (name:version-release)
    vdagURI: str = ''
    # Tags used for discovery and categorization of the vDAG
    discoveryTags: List[str] = field(default_factory=list)
    # Controller configuration for the vDAG including policies and inputs
    controller: Dict[str, Any] = field(default_factory=dict)
    # List of all node objects that make up the vDAG
    nodes: List[NodeObject] = field(default_factory=list)
    # DAG structure defining connections between nodes
    graph: Dict[str, Any] = field(default_factory=dict)
    # Block assignment and mapping info for this vDAG
    assignment_info: Dict = field(default_factory=dict)
    # Current status of the vDAG (e.g., pending, active, completed)
    status: str = field(default="pending")
    # Additional compiled data from the graph for runtime use
    compiled_graph_data: Dict[str, Any] = field(default_factory=dict)
    # Arbitrary metadata or annotations associated with the vDAG
    metadata: Dict[str, Any] = field(default_factory=dict)
```

### Creating a vDAG:
For creating the vDAG, refer to the documentation of Parser.

### vDAG Registry APIs:

**Endpoint:** `/vdag/:vdagURI`  
**Method:** `GET`  
**Description:**  
Fetches the vDAG document identified by the given `vdagURI`.

**Example curl Command:**

```bash
curl -X GET http://<server-url>/vdag/sample-vdag:1.0-stable
```

---

**Endpoint:** `/vdag/:vdagURI`  
**Method:** `PUT`  
**Description:**  
Updates fields in the vDAG document identified by the given `vdagURI` using MongoDB-style update syntax.

**Example curl Command:**

```bash
curl -X PUT http://<server-url>/vdag/sample-vdag:1.0-stable \
     -H "Content-Type: application/json" \
     -d '{
           "$set": {
             "status": "active",
             "metadata.owner": "team-ml"
           }
         }'
```

---

**Endpoint:** `/vdag/:vdagURI`  
**Method:** `DELETE`  
**Description:**  
Deletes the vDAG document identified by the given `vdagURI`.

**Example curl Command:**

```bash
curl -X DELETE http://<server-url>/vdag/sample-vdag:1.0-stable
```

---

**Endpoint:** `/vdags`  
**Method:** `POST`  
**Description:**  
Queries multiple vDAG documents using a MongoDB-style filter object.

**Example curl Command:**

```bash
curl -X POST http://<server-url>/vdags \
     -H "Content-Type: application/json" \
     -d '{
           "status": "pending",
           "metadata.owner": "team-ml"
         }'
```
---

## vDAG controllers registry:
vDAG controller registry stores all the vDAG controllers that are created to serve vDAG inference requests. 

Here is the schema of a vDAG controller:

```python
from dataclasses import dataclass, field
from typing import Dict, List, Any

@dataclass
class vDAGController:
    # Unique identifier for the vDAG controller instance
    vdag_controller_id: str = ''
    # Associated vDAG URI this controller is managing
    vdag_uri: str = ''
    # Publicly accessible URL for interacting with the controller
    public_url: str = ''
    # Identifier of the cluster where the controller is deployed
    cluster_id: str = ''
    # Arbitrary metadata for storing additional information
    metadata: Dict[str, Any] = field(default_factory=dict)
    # Configuration parameters used by the controller
    config: Dict[str, Any] = field(default_factory=dict)
    # Tags used for search and discovery of the controller
    search_tags: List[str] = field(default_factory=list)

```
### Creating a vDAG controller:
For creating the vDAG controller, refer to the documentation of Parser.

### vDAG controllers registry APIs:

**Endpoint:** `/vdag-controller/:controller_id`  
**Method:** `GET`  
**Description:**  
Fetches the vDAG Controller document identified by the given `controller_id`.

**Example curl Command:**

```bash
curl -X GET http://<server-url>/vdag-controller/controller-123
```

---

**Endpoint:** `/vdag-controller/:controller_id`  
**Method:** `PUT`  
**Description:**  
Updates fields in the vDAG Controller document identified by the given `controller_id` using MongoDB-style update syntax.

**Example curl Command:**

```bash
curl -X PUT http://<server-url>/vdag-controller/controller-123 \
     -H "Content-Type: application/json" \
     -d '{
           "$set": {
             "metadata.owner": "team-alpha",
             "public_url": "https://controller.example.com"
           }
         }'
```

---

**Endpoint:** `/vdag-controller/:controller_id`  
**Method:** `DELETE`  
**Description:**  
Deletes the vDAG Controller document identified by the given `controller_id`.

**Example curl Command:**

```bash
curl -X DELETE http://<server-url>/vdag-controller/controller-123
```

---

**Endpoint:** `/vdag-controllers`  
**Method:** `POST`  
**Description:**  
Queries multiple vDAG Controller documents using a MongoDB-style filter object.

**Example curl Command:**

```bash
curl -X POST http://<server-url>/vdag-controllers \
     -H "Content-Type: application/json" \
     -d '{
           "cluster_id": "cluster-west-1",
           "metadata.owner": "team-alpha"
         }'
```

---

**Endpoint:** `/vdag-controllers/by-vdag-uri/:vdag_uri`  
**Method:** `GET`  
**Description:**  
Fetches all vDAG Controller documents associated with the given `vdag_uri`.

**Example curl Command:**

```bash
curl -X GET http://<server-url>/vdag-controllers/by-vdag-uri/sample-vdag:1.0-stable
```
