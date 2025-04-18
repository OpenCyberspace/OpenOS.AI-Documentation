# LLM Model Splitting

Very large language models can be served by splitting them into smaller independent models (chunks), with each chunk deployed as a separate block. This approach allows each chunk to be managed and scaled independently based on demand. A **vDAG** (virtual Directed Acyclic Graph) is constructed over the model splits to establish data flow between them. Using the vDAG controller infrastructure, inference can be executed across these splits as if the model were a single unit.

**Note**: Model split runner system is a generic framework system, it only provides the basic bare-bones for the splitting and deployment of model splitting.

## Advantages of This Approach:

1. A large language model can be split and deployed across different clusters over a network or across multiple nodes within the same cluster—enabling distributed inference using pipeline parallelism.

2. Models that cannot fit within a single machine or cluster can still be deployed efficiently.

3. Since each model split is deployed as an independent block, it can be individually managed and scaled, enabling granular scaling and avoiding unnecessary resource usage.

4. Collective resource sharing is achieved, allowing developers to leverage GPU and compute resources across clusters for inference.

5. The model splits can be re-used across multiple vDAGs.

## Disadvantages:

1. Not all model architectures are suitable for splitting—only a subset of models can effectively run using this approach.

2. Latency can increase due to communication bottlenecks, especially if splits are deployed across clusters and intermediate data must travel over the internet.

3. Data transfer latencies can be significant, particularly when large intermediate tensors are exchanged between splits.

4. Implementing model splitting logic can be complex and may require iterative experimentation and fine-tuning.

5. Onboarding a model split and runner system end to end can be complex and time-consuming.

---

## Model Splitting Task Runners

Clusters can optionally deploy a **model split runner service**, which is responsible for receiving model splitting tasks from users and executing them on the target cluster using a Kubernetes job. This job runs a user-defined custom container image (built using the Model Split SDK, as explained in later sections).

The model split runner generates a unique task ID for each submitted task and uses the global task registry to track its progress and update the status once the model split task is completed.

Absolutely! Here's the **API documentation** for the `start_model_split_job` endpoint, tailored to include your `task_data` schema with fields like `model_name`, `model_download_url`, `world_size`, and `model_split_config`.

Sure! Here's the cleaned-up, professional version of your API documentation without emojis and using consistent `###` sub-headings throughout:

---

### API Documentation

#### 1. Start Model Split Job

**Description:**  
Creates a new task of type `model_splits` in the global task database and launches a Kubernetes job to perform the model splitting.

**Endpoint:**  
`POST /start_model_split_job`

**Request Content-Type:**  
`application/json`

---

#### Request Body

| Field             | Type     | Required | Description                                              |
|------------------|----------|----------|----------------------------------------------------------|
| `task_data`       | `object` | Yes      | Data required to split the model                         |
| `container_image` | `string` | Yes      | Docker image to use for the job                          |
| `storage_size`    | `string` | No       | Volume size for persistent storage (default: `"1Gi"`)    |

**`task_data` Fields**

| Field                | Type     | Required | Description                                     |
|---------------------|----------|----------|-------------------------------------------------|
| `model_name`         | `string` | Yes      | Name of the model to split                      |
| `model_download_url` | `string` | Yes      | URL to download the model weights               |
| `world_size`         | `int`    | Yes      | Number of splits/parallel jobs required         |
| `model_split_config` | `object` | Yes      | Configuration for how the model should be split |

---

#### Example Request

```json
POST /start_model_split_job
Content-Type: application/json

{
  "task_data": {
    "model_name": "gpt2",
    "model_download_url": "https://example.com/models/gpt2.tar.gz",
    "world_size": 4,
    "model_split_config": {
      "split_strategy": "tensor_parallel",
      "tp_degree": 2
    }
  },
  "container_image": "myregistry/model-split-job:latest",
  "storage_size": "2Gi"
}
```

---

#### Successful Response

```json
{
  "success": true,
  "task_id": "abc123-task-id"
}
```

---

#### Error Response

```json
{
  "success": false,
  "error": "Missing required fields"
}
```

---

## Split Runners Registry

### 1. Usage

The **Split Runners Registry** is designed to manage and deploy split runner servers. It allows the registration, querying, updating, and deletion of split runner instances, which run model split APIs on Kubernetes clusters.

#### Steps to Use the Registry:
1. **Create a Split Runner**  
   Use the `POST /split-runner` endpoint to create a new Split Runner instance. This will:
   - Deploy a new split runner server to the Kubernetes cluster using the provided `cluster_k8s_config`.
   - Register the split runner instance in the registry.
   
2. **Get a Split Runner**  
   Use the `GET /split-runner/{runner_id}` endpoint to retrieve details of an existing split runner instance by its ID.

3. **Update a Split Runner**  
   Use the `PUT /split-runner/{runner_id}` endpoint to update an existing split runner's details.

4. **Delete a Split Runner**  
   Use the `DELETE /split-runner/{runner_id}` endpoint to delete a split runner instance from the registry and Kubernetes.

5. **Query Split Runners**  
   Use the `POST /split-runners` endpoint to query and filter split runners based on specific criteria.

---

### 2. API Documentation

#### 2.1 `POST /split-runner`

**Description:**  
Creates a new Split Runner instance and deploys it to the Kubernetes cluster.

**Request Body:**

```json
{
  "cluster_k8s_config": { /* Kubernetes config dict */ },
  "split_runner_public_host": "<server-url>",
  "split_runner_metadata": { "key": "value" },
  "split_runner_tags": ["tag1", "tag2"]
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "SplitRunner created",
    "id": "runner-id"
  }
}
```

**cURL Command:**

```bash
curl -X POST http://<server-url>:8001/split-runner \
  -H "Content-Type: application/json" \
  -d '{
    "cluster_k8s_config": { /* Kubernetes config dict */ },
    "split_runner_public_host": "192.168.0.106",
    "split_runner_metadata": {"key": "value"},
    "split_runner_tags": ["tag1", "tag2"]
  }'
```

---

#### 2.2 `GET /split-runner/{runner_id}`

**Description:**  
Retrieves a Split Runner instance by its ID.

**Response:**

```json
{
  "success": true,
  "data": {
    "split_runner_id": "runner-id",
    "split_runner_public_url": "http://split-runner-url",
    "split_runner_metadata": { "key": "value" },
    "split_runner_public_host": "<server-url>",
    "split_runner_tags": ["tag1", "tag2"]
  }
}
```

**cURL Command:**

```bash
curl -X GET http://<server-url>:8001/split-runner/runner-id
```

---

#### 2.3 `PUT /split-runner/{runner_id}`

**Description:**  
Updates the details of a Split Runner instance.

**Request Body:**

```json
{
  "$set": {
    "split_runner_metadata.version": "2.0"
  },
  "$addToSet": {
    "split_runner_tags": "auto-scaled"
  }
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "SplitRunner updated"
  }
}
```

**cURL Command:**

```bash
curl -X PUT http://<server-url>:8001/split-runner/runner-id \
  -H "Content-Type: application/json" \
  -d '{
    "$set": {
      "split_runner_metadata.version": "2.0"
    },
    "$addToSet": {
      "split_runner_tags": "auto-scaled"
    }
  }'
```

---

#### 2.4 `DELETE /split-runner/{runner_id}`

**Description:**  
Deletes a Split Runner instance by its ID.

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "SplitRunner deleted"
  }
}
```

**cURL Command:**

```bash
curl -X DELETE http://<server-url>:8001/split-runner/runner-id
```

---

#### 2.5 `POST /split-runners`

**Description:**  
Queries split runners based on a filter.

**Request Body:**

```json
{
  "split_runner_metadata.framework": "transformers",
  "split_runner_tags": { "$in": ["llm"] }
}
```

**Response:**

```json
{
  "success": true,
  "data": [
    {
      "split_runner_id": "runner-id",
      "split_runner_public_url": "http://host:32286",
      "split_runner_metadata": { "framework": "transformers" },
      "split_runner_public_host": "<server-url>",
      "split_runner_tags": ["llm", "split"]
    }
  ]
}

```

**cURL Command:**

```bash
curl -X POST http://<server-url>:8001/split-runners \
  -H "Content-Type: application/json" \
  -d '{
    "split_runner_metadata.framework": "transformers",
    "split_runner_tags": { "$in": ["llm"] }
  }'

```

## Model layers registry

The Model Layers Registry is used to store information about individual model layers obtained as a result of model splitting. This registry enables reusability by tracking layer hashes that can be referenced as metadata within blocks. By doing so, the system can detect whether a block is already running with an existing split, allowing for intelligent sharing and avoiding redundant layer creation.

To ensure uniqueness and consistency, the MD5 hash of each model layer must be computed and stored as the `model_layer_hash`. This hash serves as the primary key in the model layers registry and is the basis for identifying and matching reusable layers across different model instantiations.

Certainly! Here's the **technical documentation** of the **Model Layer Registry schema**, presented in a structured table format:

---

### Model Layer Registry schema:

```python
@dataclass
class ModelLayerObject:
    model_layer_hash: str = ''
    model_asset_id: str = ''
    model_component_registry_uri: str = ''
    model_layer_public_url: str = ''
    model_layer_metadata: List[Dict[str, Any]] = field(default_factory=list)
    model_layer_rank: int = 0
    model_world_size: int = 0
```


| Field                         | Type              | Required | Description                                                                 |
|------------------------------|-------------------|----------|-----------------------------------------------------------------------------|
| `model_layer_hash`           | `string`          | yes    | **Primary Key.** MD5 hash of the serialized model layer. Used for uniqueness. |
| `model_asset_id`             | `string`          | yes   | ID of the original model asset from which this layer was generated.         |
| `model_component_registry_uri` | `string`        | yes    | URI pointing to the component spec or metadata used for this model layer.   |
| `model_layer_public_url`     | `string`          | yes   | Public URL where the layer artifact is hosted (e.g., S3, HTTP).             |
| `model_layer_metadata`       | `array[dict]`     | no      | Arbitrary metadata attached to this layer (e.g., shape, precision, config). |
| `model_layer_rank`           | `integer`         | yes    | Rank/index of this layer in the full model pipeline (used in splits).       |
| `model_world_size`           | `integer`         | yes   | Total number of model splits (parallel components) this layer belongs to.   |

---

### Model split registry APIs:

#### **1. Create a Model Layer (used by the model split job to register the new layer)**
**POST** `/model-layer`  
**Description:** Inserts a new model layer entry into the model layer registry.  
- Expects the full model layer payload in JSON format.

```bash
curl -X POST http://<server-url>:8002/model-layer \
  -H "Content-Type: application/json" \
  -d '{
    "model_layer_hash": "abc123...",
    "model_asset_id": "gpt2-base",
    "model_component_registry_uri": "registry://component/decoder",
    "model_layer_public_url": "https://example.com/layers/abc123",
    "model_layer_metadata": [{"precision": "fp16"}],
    "model_layer_rank": 0,
    "model_world_size": 4
  }'
```

---

#### **2. Get a Model Layer by Hash**
**GET** `/model-layer/<model_layer_hash>`  
**Description:** Fetches the details of a specific model layer using its hash.

```bash
curl -X GET http://<server-url>:8002/model-layer/abc123...
```

---

#### **3. Update a Model Layer**
**PUT** `/model-layer/<model_layer_hash>`  
**Description:** Updates an existing model layer.  
- Accepts Mongo-style update operations (e.g. `$set`).

```bash
curl -X PUT http://<server-url>:8002/model-layer/abc123... \
  -H "Content-Type: application/json" \
  -d '{
    "$set": {
      "model_layer_public_url": "https://example.com/layers/abc123-v2",
      "model_layer_metadata": [{"precision": "fp32"}]
    }
  }'
```

---

#### **4. Delete a Model Layer**
**DELETE** `/model-layer/<model_layer_hash>`  
**Description:** Deletes a model layer from the registry by its hash.

```bash
curl -X DELETE http://<server-url>:8002/model-layer/abc123...
```

---

#### **5. Query Model Layers**
**POST** `/model-layers`  
**Description:** Performs a Mongo-style filter query across all model layers.

```bash
curl -X POST http://<server-url>:8002/model-layers \
  -H "Content-Type: application/json" \
  -d '{
    "model_asset_id": "gpt2-base",
    "model_layer_rank": 0
  }'
```

---

Certainly. Below is a clean, professional, and detailed technical documentation for the `aios_model_splitter` library and Docker build process:

---

## Model splitter SDK

The `aios_model_splitter` library provides a structured and reusable framework for defining containerized model splitting tasks in AIOS. It defines a consistent lifecycle (`begin`, `main`, `finish`), handles environment parsing via a `Context` object, and integrates with a global task tracking system for task state management.

---

### 1. Context Object

The `Context` object abstracts environment variables into a structured Python interface. It is automatically instantiated at runtime inside the `execute_model_splitter` execution wrapper and passed to the user-defined container class.

#### Environment Variables

| Environment Variable          | Field in `Context`             | Type   | Description                                                  |
|------------------------------|-------------------------------|--------|--------------------------------------------------------------|
| `TASK_ID`                    | `task_id`                      | `str`  | Unique task identifier used to track lifecycle status        |
| `TASK_DATA`                  | `task_data`                    | `dict` | JSON-formatted data used to control the behavior of the task |
| `TASK_STATUS_UPDATE_URL`     | `task_status_update_url`       | `str`  | Endpoint to update task status in the global task DB         |
| `MODEL_LAYERS_REGISTRY_URL`  | `model_layers_registry_url`    | `str`  | URL of the model layer registry for tracking model splits    |
| `BLOCK_DB_URL`               | `block_db_url`                 | `str`  | URL of the block-level metadata registry                     |

#### Sample `task_data` Structure

```json
{
  "model_name": "gpt2",
  "model_download_url": "https://example.com/models/gpt2.tar.gz",
  "world_size": 4,
  "model_split_config": {
    "split_strategy": "tensor_parallel",
    "tp_degree": 2
  }
}
```

The `model_split_config` field is expected to be interpreted by the custom user code. It can be used to pass model-specific splitting logic.

---

### 2. Sample Container Class

The user must implement a class that defines three methods: `begin`, `main`, and `finish`. Each method accepts and returns a consistent structure.

#### Method Signature

| Method    | Input Parameter | Output                                  |
|-----------|------------------|-----------------------------------------|
| `begin()` | None             | `(True, extra_data)` or `(False, error)`|
| `main()`  | `extra_data`     | `(True, extra_data)` or `(False, error)`|
| `finish()`| `extra_data`     | `(True, result_dict)` or `(False, error)`|

#### Sample Implementation

```python
class SampleContainerClass:
    def __init__(self, context: Context):
        self.context = context

    def begin(self):
        # Perform setup, download model, etc.
        return True, {"download_path": "/tmp/gpt2.tar.gz"}

    def main(self, extra_data):
        # Perform model split based on task_data['model_split_config']
        split_info = {"splits": ["split1", "split2"]}
        return True, {**extra_data, **split_info}

    def finish(self, extra_data):
        # Final logging, upload metadata, or generate registry payload
        return True, {"model_plan_data": ""}
```

---

### 3. `finish()` Method in Detail

The `finish()` method is responsible for wrapping up the task. It should:

- Persist final outputs (e.g., split metadata, upload results)
- Clean up temporary storage (optional)
- Return a dictionary summarizing the final state of the task

#### Expected Return

| Return Type | Structure |
|-------------|-----------|
| Success     | `(True, final_status_dict)` |
| Failure     | `(False, error_message)`    |

#### Example Output

```python
return True, {
    "status": "done",
    "model_name": "gpt2",
    "split_strategy": "tensor_parallel",
    "tp_degree": 2,
    "output_dir": "/mnt/splits/gpt2/",
    "artifact": "/mnt/splits/gpt2/registry.json"
}
```

The contents of this dictionary are sent to the task database using the `GlobalTasksDB.update_task(..., "complete", result_dict)` API.

---

### 4. Building the Base Docker Image

The base Docker image provides a pre-installed, editable version of `aios_model_splitter`. This image can be used as a foundation for custom model splitting implementations.

#### Dockerfile

```Dockerfile
FROM python:3.10-slim

WORKDIR /app

COPY . /app

RUN pip3 install --upgrade pip && pip3 install -e .

CMD ["python3"]
```

#### Build Instructions

```bash
cd services/llm_splits_system/splits_sdk/

docker build -t aios_model_splitter:base .
```

---

### 5. Building on Top of the Base Docker Image

To implement a custom model splitting task, create a new Dockerfile that uses the base image and overrides the entrypoint logic.

#### Example Custom Dockerfile

```Dockerfile
FROM aios_model_splitter:base

COPY my_splitter_code/ /splitter

WORKDIR /splitter

CMD ["python3", "my_splitter_entrypoint.py"]
```

#### Example Entrypoint Script (`my_splitter_entrypoint.py`)

```python
from aios_model_splitter.executor import execute_model_splitter
from my_splitter_code.splitter_container import MyCustomModelSplitter

if __name__ == "__main__":
    execute_model_splitter(MyCustomModelSplitter)
```

---

## Utility Modules – aios_model_splitter

This section documents the utility modules included in the `aios_model_splitter` library. These utilities support common operations such as file transfer, model artifact download, and hash computation. They are designed to be reusable components for building containerized model splitters.

---

### 1. `S3Manager`

The `S3Manager` class provides an interface to upload and download files from Amazon S3 using the `boto3` SDK.

#### Constructor

```python
S3Manager(aws_access_key_id: str, aws_secret_access_key: str, region_name: str = 'us-east-1')
```

| Parameter              | Type   | Description                                 |
|------------------------|--------|---------------------------------------------|
| `aws_access_key_id`    | str    | AWS access key for authentication           |
| `aws_secret_access_key`| str    | AWS secret key for authentication           |
| `region_name`          | str    | AWS region name (default: `"us-east-1"`)    |

#### Methods

**`upload_file`**

```python
upload_file(file_path: str, bucket_name: str, object_key: str) -> bool
```

Uploads a local file to the specified S3 bucket and object key.

| Parameter     | Type   | Description                                  |
|---------------|--------|----------------------------------------------|
| `file_path`   | str    | Local path of the file to upload             |
| `bucket_name` | str    | Name of the target S3 bucket                 |
| `object_key`  | str    | Target S3 key (i.e., object path in the bucket) |

Returns `True` on success, `False` on failure.

#### `download_file`

```python
download_file(bucket_name: str, object_key: str, destination_path: str) -> bool
```

Downloads a file from S3 and stores it locally at the given path.

| Parameter         | Type   | Description                                  |
|-------------------|--------|----------------------------------------------|
| `bucket_name`     | str    | Name of the source S3 bucket                 |
| `object_key`      | str    | S3 object key to download                    |
| `destination_path`| str    | Local path to store the downloaded file      |

Returns `True` on success, `False` on failure.

#### Usage Example

```python
s3 = S3Manager("ACCESS_KEY", "SECRET_KEY")
s3.upload_file("/tmp/model.bin", "my-models", "gpt2/model.bin")
s3.download_file("my-models", "gpt2/model.bin", "/tmp/model.bin")
```

---

### 2. `FileDownloader`

The `FileDownloader` class provides functionality to download a file from a remote URL and save it locally. This is useful for downloading pretrained models, datasets, or any required resources during the `begin()` phase.

#### Methods

**`download`**

```python
download(url: str, target_path: str) -> bool
```

Downloads a file from the specified URL and saves it to the given path.

| Parameter     | Type   | Description                                     |
|---------------|--------|-------------------------------------------------|
| `url`         | str    | URL of the file to download                     |
| `target_path` | str    | Full local path to store the downloaded file    |

Returns `True` on success, `False` on failure.

#### Behavior

- Automatically creates directories in the target path.
- Downloads in 4 MB chunks to optimize memory usage.
- Logs success or failure with appropriate context.

#### Usage Example

```python
downloader = FileDownloader()
downloader.download("https://example.com/model.tar.gz", "/tmp/model.tar.gz")
```

---

### 3. `LayerHash`

The `LayerHash` class provides a method to generate a deterministic hash of a model file. This is typically used to track layer reuse, deduplication, and caching in model layer registries.

#### Methods

**`get_hash`**

```python
get_hash(file_path: str) -> Optional[str]
```

Computes an MD5 hash of the file at the specified path.

| Parameter     | Type   | Description                           |
|---------------|--------|---------------------------------------|
| `file_path`   | str    | Absolute path to the file             |

Returns the MD5 hex digest of the file contents on success, or `None` on failure.

#### Usage Example

```python
hasher = LayerHash()
md5 = hasher.get_hash("/mnt/models/split_1.pt")
```

#### Behavior

- Returns `None` and logs an error if the file does not exist.
- Reads the file in 4 KB chunks to conserve memory.

---

### Summary

| Utility      | Purpose                                         | Key Methods             |
|--------------|--------------------------------------------------|--------------------------|
| `S3Manager`  | Upload/download files to/from Amazon S3         | `upload_file`, `download_file` |
| `FileDownloader` | Download remote files via HTTP/HTTPS        | `download`               |
| `LayerHash`  | Compute MD5 hash for model or layer files       | `get_hash`               |

## Registry Clients – aios_model_splitter

This section documents the client utilities for interacting with external registries within the `aios_model_splitter` library. These registries include the **Model Layer Registry** and the **Block Metadata Registry**, which are crucial for tracking reusable model components and active block deployments.

---

### 1. `ModelLayerClient`

The `ModelLayerClient` class is used to interact with the **Model Layers Registry**. It allows you to create, retrieve, and query stored model layer metadata.

#### Constructor

```python
ModelLayerClient()
```

Reads the registry endpoint from the environment variable `MODEL_LAYERS_REGISTRY_URL`.

#### Environment Variable

| Variable                     | Description                                 |
|-----------------------------|---------------------------------------------|
| `MODEL_LAYERS_REGISTRY_URL` | Base URL of the model layers registry API   |

#### Methods

**`create_model_layer`**

```python
create_model_layer(payload: dict) -> dict
```

Registers a new model layer with the registry.

| Parameter | Type | Description                          |
|-----------|------|--------------------------------------|
| `payload` | dict | Metadata to register the model layer |

Returns the created layer metadata as a dictionary on success. Raises an exception on failure.

**Sample Payload**

```python
{
    "layer_hash": "abc123",
    "layer_metadata": {
        "model_name": "gpt2",
        "split_index": 0,
        "size_mb": 45
    }
}
```

**`get_model_layer`**

```python
get_model_layer(layer_hash: str) -> dict
```

Retrieves metadata of a specific model layer using its hash.

| Parameter     | Type | Description                          |
|---------------|------|--------------------------------------|
| `layer_hash`  | str  | MD5 hash of the model layer file     |

Returns the layer metadata as a dictionary on success. Raises an exception if the layer is not found.

**`query_model_layers`**

```python
query_model_layers(query_filter: dict) -> list
```

Queries multiple model layers based on a Mongo-style filter.

| Parameter       | Type | Description                                 |
|-----------------|------|---------------------------------------------|
| `query_filter`  | dict | Filter criteria for matching layer metadata |

Returns a list of matching model layer entries. Raises an exception on failure.

**Example Query Filter**

```python
{
    "layer_metadata.model_name": "gpt2"
}
```

---

### 2. `Blocks`

The `Blocks` class allows querying the **Block Metadata Registry** to find blocks that are running a particular model layer, identified by a layer hash and component URI.

### Constructor

```python
Blocks()
```

Internally initializes a `BlocksClient()` which is assumed to be a wrapper for block-level querying.

#### Methods

**`query`**

```python
query(component_uri: str, hash: str) -> list
```

Queries for active blocks that are running a specific model layer identified by both the component URI and its layer hash.

| Parameter        | Type | Description                                     |
|------------------|------|-------------------------------------------------|
| `component_uri`  | str  | Logical identifier for the component or block   |
| `hash`           | str  | Hash of the model layer used by the block       |

Returns a list of matching block records. Raises an exception on failure.

**Example**

```python
blocks = Blocks()
running_blocks = blocks.query("split-runner.torch-split-runner:v0.0.1-beta", "abc123")
```

This would return all running blocks using layer hash `"abc123"` for the component `"split-runner.torch-split-runner:v0.0.1-beta"`.

---

### Summary

| Client            | Purpose                                             | Key Methods                    |
|-------------------|-----------------------------------------------------|---------------------------------|
| `ModelLayerClient`| Create, retrieve, and query registered model layers | `create_model_layer`, `get_model_layer`, `query_model_layers` |
| `Blocks`          | Query running blocks by component URI and layer hash| `query`                         |

---



