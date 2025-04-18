# Management cluster

Management cluster is a dedicated cluster in a network responsible for hosting the management services responsible for orchestrating, managing the clusters in a network, it also hosts multiple registries used for storing information about blocks, clusters, vDAGs, metrics, policies and others.

## Prerequisites:

1. A working kubernetes cluster installed using `kubeadm` or others.

### Cluster requirements:

1. One master node, you can optionally setup multi-master cluster for high availability.

2. At-least 3 worker nodes with 40GB HDD and 8GB RAM. 

---

2. Install a CNI layer like `weave`, `flannel`,  `calico`, `cilium` for inter container communication.

3. `kubectl` to deploy the services.

4. Make sure that these network ports are open:

#### 1. Control Plane Node (Master)

| **Port**     | **Protocol** | **Component**           | **Description**                                                                 |
|--------------|--------------|--------------------------|---------------------------------------------------------------------------------|
| 6443         | TCP          | kube-apiserver           | Primary API server port. Required for `kubeadm join` and all cluster operations |
| 2379-2380    | TCP          | etcd                     | Used for etcd server and peer communication (required if running etcd yourself) |
| 10250        | TCP          | kubelet                  | Used by control plane to communicate with kubelet on nodes                      |
| 10257        | TCP          | kube-controller-manager  | Used internally for metrics (usually not needed externally)                     |
| 10259        | TCP          | kube-scheduler           | Used internally for metrics (usually not needed externally)                     |

#### 2. Worker Node/s

| **Port**     | **Protocol** | **Component**           | **Description**                                                               |
|--------------|--------------|--------------------------|-------------------------------------------------------------------------------|
| 10250        | TCP          | kubelet                  | Required for communication with the control plane                             |
| 30000–32767  | TCP/UDP      | NodePort Services        | Port range for services exposed via NodePort type                             |

---

## Database:

### Installing the regitries data-base:

For minimal setup, a single database system can be used as the data backend for all the registries, here is how you can setup the database:

**Label the nodes on which you want to deploy registry database on:**

```sh
kubectl label node <node-name> registry=yes
```

```sh
cd k8s/

# ./installer/db/install.sh <replicas> <size-of-each-replica>

# creates a DB with 3 replicas, 10GB each
./installer/db/install.sh 3 10Gi

```

If the script executes properly, you should see the pods running in `registries` namespace:

```sh
NAME                          READY   STATUS      RESTARTS   AGE
registry-0-647977bfd7-78v2f   1/1     Running     0          47s
registry-1-f7cb968f5-2bn4x    1/1     Running     0          41s
registry-2-5766f9b467-p896c   1/1     Running     0          34s
registry-init-client          0/1     Completed   0          29s
```

Internally the registry can be accessible at: `registry-0.registry.svc.cluster.local` 

To uninstall:

```
cd k8s
./installer/db/uninstall.sh
```

---

## Deploying registry services:

Once the registry is ready, registry services can be deployed:

```
cd k8s
kubectl create -f ./installer/registries/install.sh
```

The script should create following services

| **Service Name**                   | **Internal URL**                                                                 | **External URL**                        | **Description**                                                   |
|-----------------------------------|----------------------------------------------------------------------------------|-----------------------------------------|------------------------------------------------------------------|
| `blocks-db-svc`                   | `http://blocks-db-svc.registries.svc.cluster.local:3001`                        | `http://<NODE_PUBLIC_IP>:30100`         | Stores the blocks running in the network     |
| `clusters-db-service`             | `http://clusters-db-service.registries.svc.cluster.local:3000`                  | `http://<NODE_PUBLIC_IP>:30101`         | Stores the clusters onboarded in the networks             |
| `policies-system`                 | `http://policies-system.registries.svc.cluster.local:10000`                     | `http://<NODE_PUBLIC_IP>:30102`         | Stores the definitions of policies    |
| `vdag-db-svc`                     | `http://vdag-db-svc.registries.svc.cluster.local:10501`                         | `http://<NODE_PUBLIC_IP>:30103`         | Stores vDAGs running in the network         |
| `assets-db-registry-svc`         | `http://assets-db-registry-svc.registries.svc.cluster.local:4000`              | `http://<NODE_PUBLIC_IP>:30104`         | Stores the list of assets DB available in the network      |
| `adhoc-server-registry-svc`    | `http://adhoc-serviers-registry-svc.registries.svc.cluster.local:6000`         | `http://<NODE_PUBLIC_IP>:30105`         | Stores the list of adhoc inference servers running in the network                |
| `templates-registry-svc`         | `http://templates-registry-svc.registries.svc.cluster.local:9000`              | `http://<NODE_PUBLIC_IP>:30106`         | Stores the templates used for building custom parsers                             |
| `spec-registry-svc`              | `http://spec-registry-svc.registries.svc.cluster.local:8000`                   | `http://<NODE_PUBLIC_IP>:30107`         | Stores the list of re-usable parser spec             |
| `tasks-db-svc`                   | `http://tasks-db-svc.registries.svc.cluster.local:8000`                        | `http://<NODE_PUBLIC_IP>:30108`         | Stores the tasks executed in the system  (tasks that need async processing)                 |
| `containers-registry-svc`        | `http://containers-registry-svc.registries.svc.cluster.local:8000`             | `http://<NODE_PUBLIC_IP>:30109`         |        Stores the list of container registries available            |
| `layers-registry-svc`            | `http://layers-registry-svc.registries.svc.cluster.local:8002`                 | `http://<NODE_PUBLIC_IP>:30110`         | Stores the list of LLM model layers which can be reused       |
| `splits-runners-registry-svc`    | `http://splits-runners-registry-svc.registries.svc.cluster.local:8001`         | `http://<NODE_PUBLIC_IP>:30111`         | Stores the list of model splitter servers available in the network |
| `components-registry-svc`        | `http://components-registry-svc.registries.svc.cluster.local:4000`             | `http://<NODE_PUBLIC_IP>:30112`         | Stores the list of all components on-boarded            |

---

## Deploying Utility services:

Utility services act as supporting services for the core the system services, to deploy these services:

```sh

cd k8s/

# install
./installer/utils/install.sh
```

Here is the structured table for your `utils` services, including a filled-in **Description** column:

| **Service Name**          | **Internal URL**                                                            | **External URL**                    | **Description**                                                |
|---------------------------|-----------------------------------------------------------------------------|-------------------------------------|----------------------------------------------------------------|
| `search-server`           | `http://search-server.utils.svc.cluster.local:12000`                        | `http://<NODE_PUBLIC_IP>:30150`     | Handles search queries which is used for filtering and similarity search       |
| `resource-allocator`      | `http://resource-allocator.utils.svc.cluster.local:8765`                    | `http://<NODE_PUBLIC_IP>:30151`     | Used for resource allocation tasks        |
| `resource-allocator`      | `http://resource-allocator.utils.svc.cluster.local:7777`                    | Not exposed                         | Internal-only resource management and control interface         |
| `vdag-processor`          | `http://vdag-processor.utils.svc.cluster.local:10500`                       | Not exposed                         | Parses the vDAG structure and populates the necessary registries          |
| `failure-notifier`        | `http://failure-notifier.utils.svc.cluster.local:5000`                      | Not exposed                         | Used for notifying the failures to external services with policy execution   |

---

## Deploying metrics system:

To deploy the metrics system, first you need to deploy the metrics DB, to deploy the metrics DB you need to mark the nodes where you want to place the replicas of metrics DB as `metrics-db=yes`

```sh
kubectl label node <node-name> metrics-db=yes
```

### Deploy the metrics DB:

```
cd k8s/

# ./installer/metrics/install_db.sh <replicas> <size of each replica>
./installer/metrics/install_db.sh 3 10Gi
```

If all installation is successful, you will see logs like:

```
NAME                            READY   STATUS    RESTARTS   AGE
metrics-db-0-896c86859-t4d5b    1/1     Running   0          99s
metrics-db-1-7df48db4d7-8c4hb   1/1     Running   0          92s
metrics-db-2-77c6fd8b87-9kp5f   1/1     Running   0          85s
metrics-db-init-client          1/1     Running   0          80s
```

### Deploy the metrics services:

To deploy the metrics services:

```sh
cd k8s/installer/metrics

./install.sh

```

This will install the following services:

| **Service Name**              | **Internal URL**                                                                      | **External URL**                    | **Description**                                                        |
|-------------------------------|----------------------------------------------------------------------------------------|-------------------------------------|------------------------------------------------------------------------|
| `metrics-redis`               | `http://metrics-redis.metrics-system.svc.cluster.local:6379`                          | `http://<NODE_PUBLIC_IP>:30200`     | Redis instance through which all the metrics are pushed from the clusters                    |
| `global-block-metrics`        | `http://global-block-metrics.metrics-system.svc.cluster.local:8889`                   | `http://<NODE_PUBLIC_IP>:30201`     | Service that stores all the block metrics to the DB and provided query APIs           |
| `global-cluster-metrics`      | `http://global-cluster-metrics.metrics-system.svc.cluster.local:8888`                 | `http://<NODE_PUBLIC_IP>:30202`     | Service that stores all the cluster metrics to the DB and provided query APIs          |
| `global-vdag-metrics`         | `http://global-vdag-metrics.metrics-system.svc.cluster.local:8890`                    | `http://<NODE_PUBLIC_IP>:30203`     | Service that stores all the vdag metrics to the DB and provided query APIs     |

---

## Installing core services:

Once db, metrics, utils and registries are deployed, core services like parser and cluster controller gateway can be deployed.

```sh
cd k8s/installer/services

./install.sh
```

This will install following services:

Here is the table for the `services` namespace entries with appropriate descriptions:

| **Service Name**                   | **Internal URL**                                                                   | **External URL**                    | **Description**                                                   |
|------------------------------------|-------------------------------------------------------------------------------------|-------------------------------------|--------------------------------------------------------------------|
| `parser`                           | `http://parser.services.svc.cluster.local:8000`                                    | `http://<NODE_PUBLIC_IP>:30500`     | Parser              |
| `cluster-controller-gateway`       | `http://cluster-controller-gateway.services.svc.cluster.local:5000`                | `http://<NODE_PUBLIC_IP>:30600`     | Cluster controller gateway              |


---

## List of services and their ports:

| **Service Name**                   | **Internal URL**                                                                 | **External URL**                        | **Description**                                                   |
|-----------------------------------|----------------------------------------------------------------------------------|-----------------------------------------|------------------------------------------------------------------|
| `blocks-db-svc`                   | `http://blocks-db-svc.registries.svc.cluster.local:3001`                        | `http://<NODE_PUBLIC_IP>:30100`         | Stores the blocks running in the network                         |
| `clusters-db-service`             | `http://clusters-db-service.registries.svc.cluster.local:3000`                  | `http://<NODE_PUBLIC_IP>:30101`         | Stores the clusters onboarded in the networks                    |
| `policies-system`                 | `http://policies-system.registries.svc.cluster.local:10000`                     | `http://<NODE_PUBLIC_IP>:30102`         | Stores the definitions of policies                               |
| `vdag-db-svc`                     | `http://vdag-db-svc.registries.svc.cluster.local:10501`                         | `http://<NODE_PUBLIC_IP>:30103`         | Stores vDAGs running in the network                              |
| `assets-db-registry-svc`         | `http://assets-db-registry-svc.registries.svc.cluster.local:4000`              | `http://<NODE_PUBLIC_IP>:30104`         | Stores the list of assets DB available in the network            |
| `adhoc-server-registry-svc`      | `http://adhoc-serviers-registry-svc.registries.svc.cluster.local:6000`         | `http://<NODE_PUBLIC_IP>:30105`         | Stores the list of adhoc inference servers running in the network|
| `templates-registry-svc`         | `http://templates-registry-svc.registries.svc.cluster.local:9000`              | `http://<NODE_PUBLIC_IP>:30106`         | Stores the templates used for building custom parsers            |
| `spec-registry-svc`              | `http://spec-registry-svc.registries.svc.cluster.local:8000`                   | `http://<NODE_PUBLIC_IP>:30107`         | Stores the list of re-usable parser spec                         |
| `tasks-db-svc`                   | `http://tasks-db-svc.registries.svc.cluster.local:8000`                        | `http://<NODE_PUBLIC_IP>:30108`         | Stores the tasks executed in the system (async tasks)            |
| `containers-registry-svc`        | `http://containers-registry-svc.registries.svc.cluster.local:8000`             | `http://<NODE_PUBLIC_IP>:30109`         | Stores the list of container registries available                |
| `layers-registry-svc`            | `http://layers-registry-svc.registries.svc.cluster.local:8002`                 | `http://<NODE_PUBLIC_IP>:30110`         | Stores the list of LLM model layers which can be reused          |
| `splits-runners-registry-svc`    | `http://splits-runners-registry-svc.registries.svc.cluster.local:8001`         | `http://<NODE_PUBLIC_IP>:30111`         | Stores the list of model splitter servers available in the network |
| `components-registry-svc`        | `http://components-registry-svc.registries.svc.cluster.local:4000`             | `http://<NODE_PUBLIC_IP>:30112`         | Stores the list of all components on-boarded                     |
| `search-server`                  | `http://search-server.utils.svc.cluster.local:12000`                            | `http://<NODE_PUBLIC_IP>:30150`         | Handles search queries for filtering and similarity search       |
| `resource-allocator`             | `http://resource-allocator.utils.svc.cluster.local:8765`                        | `http://<NODE_PUBLIC_IP>:30151`         | Used for resource allocation tasks                                |
| `resource-allocator`             | `http://resource-allocator.utils.svc.cluster.local:7777`                        | Not exposed                             | Internal-only resource management and control interface          |
| `vdag-processor`                 | `http://vdag-processor.utils.svc.cluster.local:10500`                           | Not exposed                             | Parses the vDAG structure and populates the necessary registries |
| `failure-notifier`               | `http://failure-notifier.utils.svc.cluster.local:5000`                          | Not exposed                             | Notifies failures to external services with policy execution     |
| `metrics-redis`                  | `http://metrics-redis.metrics-system.svc.cluster.local:6379`                    | `http://<NODE_PUBLIC_IP>:30200`         | Redis instance used for metrics aggregation                      |
| `global-block-metrics`           | `http://global-block-metrics.metrics-system.svc.cluster.local:8889`             | `http://<NODE_PUBLIC_IP>:30201`         | Stores all block metrics and provides query APIs                 |
| `global-cluster-metrics`         | `http://global-cluster-metrics.metrics-system.svc.cluster.local:8888`           | `http://<NODE_PUBLIC_IP>:30202`         | Stores all cluster metrics and provides query APIs               |
| `global-vdag-metrics`            | `http://global-vdag-metrics.metrics-system.svc.cluster.local:8890`              | `http://<NODE_PUBLIC_IP>:30203`         | Stores all vDAG metrics and provides query APIs                  |
| `parser`                         | `http://parser.services.svc.cluster.local:8000`                                  | `http://<NODE_PUBLIC_IP>:30500`         | Parser                                                           |
| `cluster-controller-gateway`     | `http://cluster-controller-gateway.services.svc.cluster.local:5000`             | `http://<NODE_PUBLIC_IP>:30600`         | Cluster controller gateway                                       |
