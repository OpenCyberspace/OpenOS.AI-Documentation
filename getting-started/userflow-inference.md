# Inference flows

Users can submit inference tasks to the available blocks and vDAGs through inference servers and vDAG controllers, respectively.

### Here are the inference flows:

**1. Submitting inference requests to a block:**

- Discover available inference servers

- Submit inference task to a block by specifying the `block_id`

- Inference with `block_id` and files attached as `FileInfo` elements

- Inference with `query_parameters` using similarity search

- Inference using FrameDB as storage by specifying `frame_ptr`

- Inference using Graphs

**2. vDAG inference**

- Discover vDAG controllers

- Submit inference task 

## 1. Block inference:

### 1.1 Discovering inference servers:

To submit an inference task, first select an inference server from the registry.

The Inference Server Registry lists all inference servers for discovery purposes. Users can add their inference servers to this global registry if they want them to be publicly accessible.

**Schema:**

Here is the data class used to represent an inference server in the registry:

```python
@dataclass
class InferenceServer:
    inference_server_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    inference_server_name: str = ''
    inference_server_metadata: Dict[str, str] = field(default_factory=dict)
    inference_server_tags: List[str] = field(default_factory=list)
    inference_server_public_url: str = ''
```

Sample entry:

```json
{
    "inference_server_name": "inference-server-us-east-1",
    "inference_server_metadata": {
        "region": "us-east-1",
        "availability_zone": "us-east-1a",
        "provider": "AWS",
        "cluster_id": "cluster-123",
        "quota_management_data": {
            "requests_per_second_total": 100,
            "requests_per_second_per_session": 10,
            "requests_per_second_per_block_id": 10,
            "requests_per_session_id": 1000,
            "requests_per_block_id": 100
        }
    },
    "inference_server_tags": ["NLP", "Transformer", "BERT"],
    "inference_server_public_url": "https://us-east-1.inference.example.com"
}
```

For more details about the inference server registry, [refer to this documentation](../adhoc-inference-server/adhoc-inference-server.md#schema).

**Query endpoint:**

**Endpoint:**  
`POST /inference_servers`  

**Description:**  
Retrieves a list of inference servers matching specific criteria using MongoDB-style queries.  

**Example Query: Find all servers in cluster-123**
```sh
curl -X POST <inference-servers-registry>/inference_servers \
     -H "Content-Type: application/json" \
     -d '{
           "inference_server_metadata.cluster_id": "cluster-123"
         }'
```

For more information about the inference server registry, [refer to this documentation](../adhoc-inference-server/adhoc-inference-server.md#inference-server-registry)

Once you find the required inference server, use `inference_server_public_url` as the gRPC endpoint for submitting inference tasks.

## 1.2 Submit inference task to a block using an inference server by specifying the `block_id`

The simplest and most direct way to submit an inference task is by specifying the `block_id`. The task will be submitted directly to the block, and the output will be returned.

For example Python client code, [refer to this documentation](../adhoc-inference-server/adhoc-inference-server.md#1-inference-with-specified-blockid).

For the schema of the inference task, [refer to this documentation](../adhoc-inference-server/adhoc-inference-server.md#grpc-inference-guide)

### 1.3 Inference with `block_id` and files attached as `FileInfo` elements

Users can also attach files along with the inference task request. These files will be accessible to the block and can be used for processing.

For example Python client code, [refer to this documentation](../adhoc-inference-server/adhoc-inference-server.md#2-inference-with-blockid-and-files-attached-as-fileinfo-elements).

For the schema of the inference task, [refer to this documentation](../adhoc-inference-server/adhoc-inference-server.md#grpc-inference-guide)

### 1.4 Inference with `query_parameters` using similarity search

If the `block_id` is not known, users can specify `query_parameters` to perform a similarity search across all available blocks. A matching block will then be selected as the target block.

For example Python client code, [refer to this documentation](../adhoc-inference-server/adhoc-inference-server.md#3-inference-with-queryparameters-using-similarity-search).

For the schema of the inference task, [refer to this documentation](../adhoc-inference-server/adhoc-inference-server.md#grpc-inference-guide)

For writing similarity search queries, [refer to this documentation](../parser/search-server.md)

### 1.5 Inference using FrameDB as storage by specifying `frame_ptr`

If very large files need to be submitted with the task, it is recommended to insert the file into FrameDB and submit the FrameDB pointer along with the task data.

For example Python client code, [refer to this documentation](../adhoc-inference-server/adhoc-inference-server.md#4-inference-with-framedb-by-specifying-frameptr).

For the schema of the inference task, [refer to this documentation](../adhoc-inference-server/adhoc-inference-server.md#grpc-inference-guide)

### 1.6 Inference using Graphs

The inference server allows users to define a dynamic graph across multiple known blocks to execute an inference workflow. These dynamic graphs are not vDAGs—they do not support pre/post-processing policies and assume interoperability of inputs and outputs across blocks.

For example Python client code, [refer to this documentation](../adhoc-inference-server/adhoc-inference-server.md#inference-using-graphs).

For the schema of the inference task, [refer to this documentation](../adhoc-inference-server/adhoc-inference-server.md#grpc-inference-guide)

---

## vDAG inference

### 2.1 Discovering vDAG controllers

The vDAG controller for the provided `vdagURI` must be discovered first in order to connect and submit an inference task.

Users can query the vDAG registry to obtain the corresponding vDAG controller, if available, for the given vDAG. The vDAG controller registry stores all vDAG controllers created to serve vDAG inference requests.

Here is the schema of a vDAG controller:

```python
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

For more details about the vDAG controller registry, [refer to this documentation](../vdag-controller/vdag-controller.md).

**Endpoint:** `/vdag-controllers/by-vdag-uri/:vdag_uri`  
**Method:** `GET`  
**Description:**  
Fetches all vDAG controller documents associated with the given `vdag_uri`.

**Example curl Command:**

```bash
curl -X GET http://<vdag-registry>/vdag-controllers/by-vdag-uri/sample-vdag:1.0-stable
```

The `public_url` field of the vDAG controller can be used as the gRPC server URL for submitting the task.

### 2.2 Submit inference task

Inference tasks for a vDAG can be submitted using the vDAG controller as the gateway. The vDAG controller provides a gRPC API through which tasks can be submitted.

For example Python client code, [refer to this documentation](../vdag-controller/vdag-controller.md#adhoc-inference-grpc-api).

For the schema of the inference task, [refer to this documentation](../vdag-controller/vdag-controller.md#adhoc-inference-grpc-api)