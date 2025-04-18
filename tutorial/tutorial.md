# Quick start tutorial

The quick start tutorial provides an example-based approach for performing different flows.

**Note: This documentation is in progress and is being updated incrementally**

## User flows:

The quick start tutorial contains handpicked user flows that are regularly used by most users:

**1. Network, cluster, and node onboarding flows**: Includes flows to create a new network, onboard a new cluster to an existing network, and join a node to an existing cluster.

**2. Block scheduling and inference flows**: Serve LLMs, deep learning and non-AI models, and computation logic on one or more clusters across the network and submit inference requests.

---

### Network, cluster, and node onboarding flows:

1. To create a new network from scratch, [refer to this section](tutorial.md#creating-a-new-network)

2. To onboard a new cluster to an existing network, [refer to this section](tutorial.md#joining-a-cluster-to-an-existing-network)

3. To onboard a remote node to an existing cluster, [refer to this section](tutorial.md#joining-a-node-to-an-already-existing-cluster)

---

### Block scheduling and inference flows:

1. **Method-1: Basic instance blocks**: Deploy AI/non-AI workloads as computational instances that can be placed on multiple GPUs and scaled across multiple nodes within the cluster on demand:

    1.1 To schedule and serve a simple object model built on top of the AIOS SDK, [refer to this section](tutorial.md#simple-block-deployment-on-a-single-gpu-sample-model-considered-yolov5)

    1.2 To schedule and serve an LLM model that spans across 3 GPUs within a node, built using the AIOS SDK and AIOS LLM Utils SDK, [refer to this section](tutorial.md#simple-block-deployment-across-multiple-gpus-reference-model-considered-mistral7b-llm)

2. **Method-2: Specifying a graph across the blocks for composable AI inference**: Deploy a directed acyclic graph across the existing blocks to compose a workflow that uses different AI models, [refer to this section](#deploying-a-vdag-and-submitting-inference-tasks-to-the-vdag)

3. **Method-3: Using a third-party system**: Use an external system either deployed remotely or internally within the cluster along with the block.

    3.1: To use a remotely deployed system that is manually deployed externally, [refer to this section](#linking-an-externally-deployed-vllm-system-to-the-block-for-serving)

    3.2: To use a third-party system internally within the cluster by deploying it alongside the block using init containers, [refer to this section](tutorial.md#deploying-external-system-along-with-the-block-using-init-containers)

4. **Method-4: Serving large models by splitting the model and distributing it across the network**: A large language model, like an LLM that can't fit into the resources of a single node, can be spawned across a cluster or across a network of clusters by splitting the model into smaller chunks using a custom splitting logic and creating a graph of the splits, [refer to this section](tutorial.md#splitting-llms-and-deploying-them-across-the-network-as-a-vdag)

---

## Creating a new network:

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

---

## Joining a cluster to an existing network:

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

## Joining a node to an already existing cluster:

To join any cluster as a node, the node has to satisfy some prerequisites, [refer to this documentation for the prerequisites](../onboarding-notes/onboarding-node.md).

Once the prerequsites are satisfied, run the following script to onboard the node:

```sh
./join_node.sh \
  --node-id node-001 \             # ID of the node
  --cluster-id cluster-abc \       # cluster ID of the target cluster
  --api-url  \
  --kubeadm-join-cmd "join command provided in clusterMetadata.additionalInfo.joinCommand field"

```
---

## Deployment and inference guide:


### Simple block deployment across multiple GPUs (Reference model considered: Mistral7B LLM):

The Mistral 7B example is provided in the `services/applications/examples/mistral7b` directory.

**Note**:

1. The policies referenced in the sample specification are assumed to exist. These sample policies will be provided during the testnet release. To write custom policies, [refer to this documentation](../policies-system/policies-system.md).

2. The registry `registry.ai-platform.com` is assumed to exist. However, a registry will be provided for use during the testnet release. To onboard your own registry, [refer to this documentation](../container-registry/container-registry.md).

---

#### Steps to Deploy the Mistral 7B Chat Block

**1. Onboard the Component**

```sh
cd services/applications/examples/mistral7b

# Step 1: Build and push the component image
sh ./build_and_push_image.sh

# Step 2: Create the component
sh ./create_component.sh
```

For more details on how to onboard a component, [refer to this documentation](../parser/component.md).

---

**2. Create the Block**

To schedule the component as a block for serving, use the following command:

```sh
cd services/applications/examples/mistral7b

# Step 3: Create a block
sh ./create_block.sh
```

---

####  Deployment Logic

The specification selects a cluster with at least **3 total GPUs**, provided by the vendor `dma-bangalore`. The cluster must also have an **average 15-minute CPU load of less than 10** and at least **48 GB of free GPU memory**.

Within the selected cluster, nodes are chosen based on the availability of GPUs with:
- **Less than 90% utilization**
- **At least 16 GB of free memory**

### Scaling Parameters:
- **Minimum instances**: 1  
- **Maximum instances**: Up to 3  

If the deployment is successful, it should create a block with ID: `mistral-chat-88`.

#### Submit tasks:

```python
import grpc
import time
import inference_pb2
import inference_pb2_grpc
import json

# Replace with a valid AIOS inference server address
SERVER_ADDRESS = "<server-url>"

def create_chat_request(message, session_id, seq_no):
    """
    Creates a Mistral 7B chat inference request.
    """
    data_dict = {
        "message": message,
        "session_id": session_id
    }

    request = inference_pb2.BlockInferencePacket(
        block_id="mistral-chat-88",  # Target block ID
        session_id=session_id,
        seq_no=seq_no,
        data=json.dumps(data_dict),
        ts=time.time(),
        query_parameters="",
        output_ptr="",
        files=[]
    )

    return request

def main():
    # Sample user inputs
    messages = [
        "Hello! What can you do?",
        "Can you explain the concept of quantum entanglement?",
        "What’s the capital of Japan?"
    ]

    # Create gRPC channel and stub
    channel = grpc.insecure_channel(SERVER_ADDRESS)
    stub = inference_pb2_grpc.BlockInferenceServiceStub(channel)

    session_id = "sess-chat-001"

    for i, msg in enumerate(messages):
        request = create_chat_request(
            message=msg,
            session_id=session_id,
            seq_no=i + 1
        )

        response = stub.infer(request)

        print(f"\nResponse for message {i + 1}:")
        print("Session ID:", response.session_id)
        print("Sequence No:", response.seq_no)
        print("Reply:", json.loads(response.data).get("reply", ""))
        print("Timestamp:", response.ts)

if __name__ == "__main__":
    main()
```

---

### Simple block deployment on a single GPU (Sample model considered: YOLOv5)

The YOLOv5 example is provided in the `services/applications/examples/yolov5-block` directory.

**Note**:

1. The policies referenced in the sample specification are assumed to exist. These sample policies will be provided during the testnet release. To write custom policies, [refer to this documentation](../policies-system/policies-system.md).

2. The registry `registry.ai-platform.com` is assumed to exist. However, a registry will be provided for use during the testnet release. To onboard your own registry, [refer to this documentation](../container-registry/container-registry.md).

---

#### Steps to Deploy a Block

**1. Onboard the Component**

```sh
cd services/applications/examples/yolov5-block

# Step 1: Build and push the component image
sh ./build_and_push_image.sh

# Step 2: Create the component
sh ./create_component.sh
```

For more details on how to onboard a component, [refer to this documentation](../parser/component.md).

---

**2. Create the Block**

To schedule the component as a block for serving, use the following command:

```sh
cd services/applications/examples/yolov5-block

# Step 3: Create a block
sh ./create_block.sh
```

---

The specification selects a cluster with at least **5 total GPUs**, provided by the vendor `dma-bangalore`. The cluster must also have an **average 15-minute CPU load of less than 10** and at least **9 GB of free GPU memory**.

Within the selected cluster, nodes are chosen based on the availability of GPUs with **less than 80% utilization** and **at least 5GB of free memory**.

The specification also defines scaling parameters:
- **Minimum instances**: 1  
- **Maximum instances**: Up to 5  

If the deployment is successful, it should create a block with ID: `detector-yolo-88`.

--- 

#### Submit tasks:

Here is the sample python script to submit the inference task via inference server and obtain the results:

```py

import grpc
import time
import inference_pb2
import inference_pb2_grpc
import json

SERVER_ADDRESS = "<server-url>" # use any of the available inference server gateways

def load_image(file_path):
    """Loads an image as binary data."""
    with open(file_path, "rb") as f:
        return f.read()

def create_yolo_request(image_bytes, bbox, file_name, session_id, seq_no):
    """
    Creates a YOLOv5 inference request with region of interest and one image file.
    """
    x1 = bbox["x"]
    y1 = bbox["y"]
    x2 = x1 + bbox["width"]
    y2 = y1 + bbox["height"]

    data_dict = {
        "roi": {
            "x1": x1,
            "y1": y1,
            "x2": x2,
            "y2": y2
        }
    }

    request = inference_pb2.BlockInferencePacket(
        block_id="detector-yolo-88",  # Target block ID
        session_id=session_id,
        seq_no=seq_no,
        data=json.dumps(data_dict),
        ts=time.time(),
        query_parameters="",
        output_ptr="",
        files=[
            inference_pb2.FileInfo(
                metadata=json.dumps({"file_name": file_name}),
                file_data=image_bytes
            )
        ]
    )

    return request

def main():
    # Sample bounding boxes for each image
    bounding_boxes = [
        {"file_path": "image1.jpg", "file_name": "image1.jpg", "x": 50, "y": 100, "width": 200, "height": 150},
        {"file_path": "image2.jpg", "file_name": "image2.jpg", "x": 30, "y": 60, "width": 180, "height": 140}
    ]

    # Create gRPC channel and stub
    channel = grpc.insecure_channel(SERVER_ADDRESS)
    stub = inference_pb2_grpc.BlockInferenceServiceStub(channel)

    session_id = "sess-67890"
    
    for i, bbox in enumerate(bounding_boxes):
        image_bytes = load_image(bbox["file_path"])
        request = create_yolo_request(
            image_bytes=image_bytes,
            bbox=bbox,
            file_name=bbox["file_name"],
            session_id=session_id,
            seq_no=i + 1
        )

        response = stub.infer(request)

        print(f"\nInference Response for image {i + 1}:")
        print("Session ID:", response.session_id)
        print("Sequence No:", response.seq_no)
        print("Data:", response.data)
        print("Timestamp:", response.ts)

if __name__ == "__main__":
    main()
```

---


### Linking an externally deployed vLLM system to the block for serving:

The vLLM Chat block example is provided in the `services/applications/examples/vllm-chat` directory.

**Notes:**

1. The policies referenced in the sample block specification are assumed to exist. These will be available during the testnet release. To write custom policies, [refer to this documentation](../policies-system/policies-system.md).

2. The registry `registry.ai-platform.com` is assumed to be available. A default registry will be provided for the testnet. To onboard your own registry, [refer to this documentation](../container-registry/container-registry.md).

3. The block communicates with an external vLLM server (running in OpenAI-compatible mode). This server is assumed to be reachable from inside the cluster. If not, expose it using an internal service or tunnel.

---

#### Steps to Deploy the vLLM Chat Block

**1. Onboard the Component**

```bash
cd services/applications/examples/vllm-chat

# Step 1: Build and push the component image
sh ./build_and_push_image.sh

# Step 2: Create the component
sh ./create_component.sh
```

For more details on how to onboard a component, [see component documentation](../parser/component.md).

---

**2. Create the Block**

To deploy the component as a running block instance:

```bash
cd services/applications/examples/vllm-chat

# Step 3: Create the block
sh ./create_block.sh
```

---

**Deployment Overview**

This block connects to an external Mistral 7B vLLM server exposed at:

```json
"vllm_server_url": "http://vllm-service:8080"
```

The server must support the OpenAI-compatible API (`/v1/chat/completions`) and load the model:

```
mistralai/Mistral-7B-Instruct-v0.1
```

---

**Resource Requirements:**

This block performs no local inference and runs entirely on CPU. Therefore:

- No GPU affinity is needed.
- The block can be deployed to any node with:
  - At least 2 CPUs
  - Approximately 2 GB of RAM

---

**Scaling and Policies:**

The specification defines:

- Minimum instances: 1  
- Maximum instances: 2  
- CPU-based autoscaling policies  
- Round-robin load balancing with session caching

---

#### Submit tasks:

Here is the sample python script to submit the inference task via inference server and obtain the results:

```python
import grpc
import time
import json
import inference_pb2
import inference_pb2_grpc

# Replace with the actual gateway host:port
SERVER_ADDRESS = "<inference-gateway-host>:<port>"

BLOCK_ID = "vllm-chat-01"  # Block ID from block.json
SESSION_ID = "chat-session-001"

def create_chat_request(message: str, session_id: str, seq_no: int):
    """
    Creates a chat inference request for the vLLMChatBlock.
    """
    data_dict = {
        "message": message,
        "session_id": session_id
    }

    request = inference_pb2.BlockInferencePacket(
        block_id=BLOCK_ID,
        session_id=session_id,
        seq_no=seq_no,
        data=json.dumps(data_dict),
        ts=time.time(),
        query_parameters="",
        output_ptr="",
        files=[]
    )

    return request

def main():
    # Example user messages
    messages = [
        "Hello, what is your name?",
        "Can you explain quantum physics in simple terms?",
        "What's the capital of France?"
    ]

    # Create gRPC channel and stub
    channel = grpc.insecure_channel(SERVER_ADDRESS)
    stub = inference_pb2_grpc.BlockInferenceServiceStub(channel)

    for i, msg in enumerate(messages):
        request = create_chat_request(msg, SESSION_ID, seq_no=i + 1)

        try:
            response = stub.infer(request)
            print(f"\nResponse {i + 1}:")
            print("Session ID:", response.session_id)
            print("Sequence No:", response.seq_no)
            print("Reply:", json.loads(response.data).get("reply", "<no reply>"))
            print("Timestamp:", response.ts)
        except grpc.RpcError as e:
            print(f"\nError during inference: {e}")

if __name__ == "__main__":
    main()

```
---

### Deploying a vDAG and submitting inference tasks to the vDAG

For detailed explanation of vDAG controller - [refer to this documentation](../vdag-controller/vdag-controller.md) and for the detailed explanation of vDAG creation process using the parser API, [refer to this documentation](../parser/vdag.md).

The following section shows the steps to create a vDAG using a predefined example and makes following assumptions:

1. Blocks `object-det-1` and `tracker-122` already exist.

2. The policies are assumed to exist, the actual policies will be provided during the testnet release.

To create the vDAG:

```sh
cd services/applications/examples/vdag-pose-estimation
sh ./create_vdag.sh
```

This should create a vDAG with name: `pose-estimator:v0.0.1-stable`.

Once this vDAG is created, deploy a controller on the cluster of your choice:

```sh
cd services/applications/examples/vdag-pose-estimation
sh ./create_controller.sh
```

The above script should create a vDAG controller with ID `estimator-001` and on cluster `cluster-123`, now the vDAG is ready to accept inference requests.

Perform the inference with the script below:

```python
import grpc
import json
import time

# Import the generated gRPC classes
import vdag_pb2
import vdag_pb2_grpc

# Create a gRPC channel to the vDAG inference service
channel = grpc.insecure_channel("http://<cluster-public-ip>:32000/vdag/estimator-001")
stub = vdag_pb2_grpc.vDAGInferenceServiceStub(channel)

# Example session and sequence information
session_id = "session-5678"
seq_no = 1

# Define the area of interest (bounding box coordinates)
# Format: [x_min, y_min, x_max, y_max]
area_of_interest = {
    "region_of_interest": {
        "x_min": 120,
        "y_min": 200,
        "x_max": 450,
        "y_max": 650
    }
}

# Load the image file to send as part of inference
with open("example_image.jpg", "rb") as img_file:
    image_bytes = img_file.read()

# Create vDAGFileInfo message with image
image_file = vdag_pb2.vDAGFileInfo(
    metadata=json.dumps({
        "filename": "example_image.jpg",
        "content_type": "image/jpeg"
    }),
    file_data=image_bytes
)

# Construct the vDAGInferencePacket
packet = vdag_pb2.vDAGInferencePacket(
    session_id=session_id,              # Unique session ID
    seq_no=seq_no,                      # Sequence number for packet ordering
    data=json.dumps(area_of_interest),  # Area of interest in JSON
    ts=time.time(),                     # Timestamp
    files=[image_file]                  # List of attached files
)

# Send inference request to the server
response = stub.infer(packet)

# Handle the response
print("Response received.")
print("Data:", response.data)

# Print details of any returned files
for f in response.files:
    metadata = json.loads(f.metadata)
    print(f"Returned file: {metadata.get('filename')} (type: {metadata.get('content_type')})")
    # Save returned file if needed
    with open(f"output_{metadata.get('filename')}", "wb") as out_file:
        out_file.write(f.file_data)

```

---

### Deploying external system along with the block using init containers:

In the above example, the vLLM system was deployed by the user manually and was linked to the block, however the deployment of vLLM or any external system can be automated in general along with the block creation and removal, this can be achieved by binding an init container along with the component which will be used to setup the third party system. The init containers are written using python SDK and it should contain the automation logic of the third party system deployment.

**The example for this approach will be provided during the testnet release**.

Here are the steps to achieve this:

1. Create the init container using python SDK, [refer this documentation](../llm-docs/third-party-blocks.md).

2. Create the component with init container specified, [refer this documentation](../parser/component.md#component-spec-example-with-init-container)

3. Create the block normally from the component just like the examples specified above, along with the block the third party system will also be deployed and the URL of the third party system will be passed into the block under `blockInitData.initContainer` field, [refer this documentation for more details](../llm-docs/third-party-blocks.md#accessing-the-third-party-system-endpoints-inside-block-services)

---

### Splitting LLMs and deploying them across the network as a vDAG:

The LLM model that can't be fit into a single node or a single cluster (in some cases) can be split into smaller chunks and can be deployed like a vDAG. Different frameworks and different model architectures might use different approaches to split the model, thus the system supports generic way to split the model split SDK, users are provided with the flexibility to write their own model splitting logic and generate the model plan file - which includes the block specification for each split of the model and a vDAG specification which specifies the connections.

**The example for this approach will be provided during the testnet release**.

Here are the steps to achieve this:

1. To create model splitting code using model splitting SDK [refer to this documentation](../llm-docs/llm-model-splits.md#model-splitter-sdk)

2. Create model splitting task to deploy the model splitting code as a container to perform model splitting, [refer to this documentation](../llm-docs/llm-model-splits.md#model-split-registry-apis), the result will be model plan layout which contains blocks and vDAGs.

3. To deploy these blocks [refer to this documentation](../parser/block.md) and to deploy the vDAG [refer to this documentation](../parser/vdag.md)
