## Overview
BeeMesh comes as a single binary for datacenters, edge and mobile computing. Just deploy to join and start deploying workloads. The underlying protocol naturally prefers nodes that are been alive for longer over newer entrants.

## Problem Statement
Kubernetes aggregates infrastructure into a single uniform computer. Unity is achieved by a cluster algorithm.The continuous allocation of the workload is done according to this set of rules and repeated as needed. Such uniform computers, also called clusters, usually follow the rules of perimeter security in the data center.

Kubernetes follows a scale-out approach when it comes to increasing the resources available for the workload. The infrastructure used can also be requested on a larger scale for a more advantageous utilization rate.

Kubernetes can be extended with network virtualization in different forms. Several clusters are disjoint and therefore require further extensions such as ISTIO and/or cluster federation. Traditional services are excluded and must be considered separately.

The design decisions and ranking for a) clustering and b) connectivity, although individually exemplary and modularly implemented, lead to limitations in terms of scaling and connectivity.

BeeMesh prioritises connectivity and dissolves the cluster in its present form. This favours today's software for long-lasting processing and functions according to today's concepts for service and data-centric security. By eliminating infrastructure clustering, the scaling limits are eliminated. This favours the life cycle and reduces the administrative effort.


## Architecture
Service Mesh is not just a pile up complexity but natively incorporated as P2P network. Clustering is a workload concern, as such the nodes do not need consensus or leader election. BeeMesh is designed for massive scale-out and improved software lifecycle.

Networking is fully replaced by transport based connectivity. As soon workload is deployed, consensus is deployed as sidecar to the pods. Stateless workload do not need state management. This encourages stateless and zero trust based microservices.

![BeeMesh Binary](assets/img/prototype.png)


## API
Must be K8s compliant so that everybody can move on. GitOps based workload provisioning is strongly advised.


## Building Blocks
* P2P: [https://libp2p.io/](https://libp2p.io/)
* Workload Clustering: [https://github.com/libp2p/go-libp2p-raft](https://github.com/libp2p/go-libp2p-raft)
* Container runtime: [https://containerd.io/](https://containerd.io/)
* Lightweight Kubernetes: [https://k3s.io/](https://k3s.io/)
* Podman: https://github.com/containers/libpod
* Example P2P Database: https://github.com/orbitdb


## Longterm

In the long term, an architecture designed for stability is to be set up while retaining the new innovative design decisions. This is to be achieved by considering Cri-O and Kubernetes instead of Podman.

### Architecture

![BeeMesh Binary](assets/img/beemesh.png)


WIP
