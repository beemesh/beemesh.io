## Overview
BeeMesh comes as a single binary for datacenters, edge and mobile computing. Service Mesh is not just a pile up complexity but natively incorporated as P2P network. Clustering is a workload concern, as such the nodes do not need consensus or leader election. BeeMesh is designed for massive scale-out and improved software lifecycle.

Just deploy to join the P2P network and start deploying your workloads. The underlying protocol naturally prefers nodes that are been alive for longer over newer entrants. Networking is fully replaced by transport based connectivity. As soon workload is deployed, consensus is deployed as sidecar to the pods. Stateless workload do not need state management. This encourages stateless and zero trust based microservices.


## Architecture

![BeeMesh Binary](assets/img/beemesh.png)


## API
Must be K8s compliant so that everybody can move on. GitOps based workload provisioning is strongly advised.


## Building Blocks
* P2P: [https://libp2p.io/](https://libp2p.io/)
* Workload Clustering: [https://github.com/libp2p/go-libp2p-raft](https://github.com/libp2p/go-libp2p-raft)
* Container runtime: [https://containerd.io/](https://containerd.io/)
* Lightweight Kubernetes: [https://k3s.io/](https://k3s.io/)
* Podman: https://github.com/containers/libpod
* Example P2P Database: https://github.com/orbitdb


## Prototype
WIP
