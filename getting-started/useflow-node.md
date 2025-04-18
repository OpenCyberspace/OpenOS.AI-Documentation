# Node user

A node user can onboard a node either as part of the network or as a remote node.

## Joining a node to an already existing cluster

To join any cluster as a node, the node must satisfy certain prerequisites. [Refer to this documentation for the prerequisites](../onboarding-notes/onboarding-node.md).

Once the prerequisites are satisfied, run the following script to onboard the node:

```sh
./join_node.sh \
  --node-id node-001 \             # ID of the node
  --cluster-id cluster-abc \       # ID of the target cluster
  --api-url <api-url> \           # (Provide the appropriate API URL)
  --kubeadm-join-cmd "join command provided in clusterMetadata.additionalInfo.joinCommand field"
```