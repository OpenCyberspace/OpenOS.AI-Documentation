# Block creation flow

The block can be deployed in following methods:

1. **Method-1: Basic instance blocks**: Deploy AI/non-AI workloads as computational instances that can be placed on multiple GPUs and scaled across multiple nodes within the cluster on demand:

    1.1 To schedule and serve a simple object model built on top of the AIOS SDK, [refer to this section](userflow-block.md#simple-block-deployment-on-a-single-gpu-sample-model-considered-yolov5)

    1.2 To schedule and serve an LLM model that spans across 3 GPUs within a node, built using the AIOS SDK and AIOS LLM Utils SDK, [refer to this section](userflow-block.md#simple-block-deployment-across-multiple-gpus-reference-model-considered-mistral7b-llm)


2. **Method-3: Using a third-party system**: Use an external system either deployed remotely or internally within the cluster along with the block.

    3.1: To use a remotely deployed system that is manually deployed externally, [refer to this section](userflow-block.md#linking-an-externally-deployed-vllm-system-to-the-block-for-serving)

    3.2: To use a third-party system internally within the cluster by deploying it alongside the block using init containers, [refer to this section](userflow-block.md#deploying-external-system-along-with-the-block-using-init-containers)

3. **Method-4: Serving large models by splitting the model and distributing it across the network**: A large language model, like an LLM that can't fit into the resources of a single node, can be spawned across a cluster or across a network of clusters by splitting the model into smaller chunks using a custom splitting logic and creating a graph of the splits, [refer to this section](userflow-block.md#splitting-llms-and-deploying-them-across-the-network-as-a-vdag)

---

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


**2. Create the Block**

To schedule the component as a block for serving, use the following command:

```sh
cd services/applications/examples/mistral7b

# Step 3: Create a block
sh ./create_block.sh
```

####  Deployment Logic

The specification selects a cluster with at least **3 total GPUs**, provided by the vendor `dma-bangalore`. The cluster must also have an **average 15-minute CPU load of less than 10** and at least **48 GB of free GPU memory**.

Within the selected cluster, nodes are chosen based on the availability of GPUs with:
- **Less than 90% utilization**
- **At least 16 GB of free memory**

### Scaling Parameters:
- **Minimum instances**: 1  
- **Maximum instances**: Up to 3  

If the deployment is successful, it should create a block with ID: `mistral-chat-88`.

---

## Simple block deployment on a single GPU (Sample model considered: YOLOv5)

The YOLOv5 example is provided in the `services/applications/examples/yolov5-block` directory.

**Note**:

1. The policies referenced in the sample specification are assumed to exist. These sample policies will be provided during the testnet release. To write custom policies, [refer to this documentation](../policies-system/policies-system.md).

2. The registry `registry.ai-platform.com` is assumed to exist. However, a registry will be provided for use during the testnet release. To onboard your own registry, [refer to this documentation](../container-registry/container-registry.md).

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

**2. Create the Block**

To schedule the component as a block for serving, use the following command:

```sh
cd services/applications/examples/yolov5-block

# Step 3: Create a block
sh ./create_block.sh
```

The specification selects a cluster with at least **5 total GPUs**, provided by the vendor `dma-bangalore`. The cluster must also have an **average 15-minute CPU load of less than 10** and at least **9 GB of free GPU memory**.

Within the selected cluster, nodes are chosen based on the availability of GPUs with **less than 80% utilization** and **at least 5GB of free memory**.

The specification also defines scaling parameters:
- **Minimum instances**: 1  
- **Maximum instances**: Up to 5  

If the deployment is successful, it should create a block with ID: `detector-yolo-88`.

---

## Linking an externally deployed vLLM system to the block for serving:

The vLLM Chat block example is provided in the `services/applications/examples/vllm-chat` directory.

**Notes:**

1. The policies referenced in the sample block specification are assumed to exist. These will be available during the testnet release. To write custom policies, [refer to this documentation](../policies-system/policies-system.md).

2. The registry `registry.ai-platform.com` is assumed to be available. A default registry will be provided for the testnet. To onboard your own registry, [refer to this documentation](../container-registry/container-registry.md).

3. The block communicates with an external vLLM server (running in OpenAI-compatible mode). This server is assumed to be reachable from inside the cluster. If not, expose it using an internal service or tunnel.


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


**2. Create the Block**

To deploy the component as a running block instance:

```bash
cd services/applications/examples/vllm-chat

# Step 3: Create the block
sh ./create_block.sh
```


**Deployment Overview**

This block connects to an external Mistral 7B vLLM server exposed at:

```json
"vllm_server_url": "http://vllm-service:8080"
```

The server must support the OpenAI-compatible API (`/v1/chat/completions`) and load the model:

```
mistralai/Mistral-7B-Instruct-v0.1
```


**Resource Requirements:**

This block performs no local inference and runs entirely on CPU. Therefore:

- No GPU affinity is needed.
- The block can be deployed to any node with:
  - At least 2 CPUs
  - Approximately 2 GB of RAM


**Scaling and Policies:**

The specification defines:

- Minimum instances: 1  
- Maximum instances: 2  
- CPU-based autoscaling policies  
- Round-robin load balancing with session caching

---

## Deploying external system along with the block using init containers:

In the above example, the vLLM system was deployed by the user manually and was linked to the block, however the deployment of vLLM or any external system can be automated in general along with the block creation and removal, this can be achieved by binding an init container along with the component which will be used to setup the third party system. The init containers are written using python SDK and it should contain the automation logic of the third party system deployment.

**The example for this approach will be provided during the testnet release**.

Here are the steps to achieve this:

1. Create the init container using python SDK, [refer this documentation](../llm-docs/third-party-blocks.md).

2. Create the component with init container specified, [refer this documentation](../parser/component.md#component-spec-example-with-init-container)

3. Create the block normally from the component just like the examples specified above, along with the block the third party system will also be deployed and the URL of the third party system will be passed into the block under `blockInitData.initContainer` field, [refer this documentation for more details](../llm-docs/third-party-blocks.md#accessing-the-third-party-system-endpoints-inside-block-services)

---

## Splitting LLMs and deploying them across the network as a vDAG:

The LLM model that can't be fit into a single node or a single cluster (in some cases) can be split into smaller chunks and can be deployed like a vDAG. Different frameworks and different model architectures might use different approaches to split the model, thus the system supports generic way to split the model split SDK, users are provided with the flexibility to write their own model splitting logic and generate the model plan file - which includes the block specification for each split of the model and a vDAG specification which specifies the connections.

**The example for this approach will be provided during the testnet release**.

Here are the steps to achieve this:

1. To create model splitting code using model splitting SDK [refer to this documentation](../llm-docs/llm-model-splits.md#model-splitter-sdk)

2. Create model splitting task to deploy the model splitting code as a container to perform model splitting, [refer to this documentation](../llm-docs/llm-model-splits.md#model-split-registry-apis), the result will be model plan layout which contains blocks and vDAGs.

3. To deploy these blocks [refer to this documentation](../parser/block.md) and to deploy the vDAG [refer to this documentation](../parser/vdag.md)