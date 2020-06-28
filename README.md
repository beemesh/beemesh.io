## Overview
BeeMesh combines zero trust data centric security with peer to peer concepts. Decentralized processing, service based meshing and ad-hoc storage are current and foreseeable requirements of towards evolution. Data-centric security is cryptographically and policy-based fundamentally considered.

## Problem Statement
Kubernetes aggregates infrastructure as a single uniform computer. Unity is achieved by a cluster algorithm. The continuous allocation of workload is done according to this set of rules and repeated as needed. Such uniform computers aka clusters, usually follow the rules of perimeter security.

Kubernetes follows a scale-out approach when it comes to increase the available resources for workloads. The infrastructure can also be requested larger eg. scaled-up for a more advantageous utilization rate.

Kubernetes connectivity can be extended with basic network virtualization in different forms. Several clusters are disjoint and therefore require further extensions such as service meshes with replicated control planes or cluster federation as needed. Traditional services are excluded and must be considered separately. Despite the overall intention, this infrastructure-focused connectivity does not meet today's requirements for a fine-grained, resilient software design.

The overall design decisions and ranking made for a) clustering and b) connectivity, although individually exemplary and modularly implemented, lead to limitations in terms of scaling and connectivity.


## Architecture
![BeeMesh Binary](assets/img/prototype.png)

BeeMesh prioritises connectivity and dissolves clustering in its present form. Removing the infrastructure clustering eliminates the scaling limits. This favours today's service and data-centric security concepts, life cycle and reduces the administrative efforts.

A Kademlia based DHT peer to peer mesh has been alive since 2005. Measurements from the year 2013 show a volume of 10 to 25 million subscribers with a daily volatility of around 10 million. The peer to peer mesh naturally prefers participiants beeing alive for longer over newer entrants. 

Upranking the peer to peer mesh over infrastructure state management enables a massive scale-out of stateless based long-lasting processing and functions. State management is solely required by stateful workload. As such, the problem context shrinks to a transient state machine exactly matching the workload lifecycle.

## Policies
Peer to peer mesh policies allows you to make long-lasting processing or functions act as a resilient system through controlling how they communicate with each other as well as with external services.


## API
A Kubernetes compliant API is encouraged so that workloads can be shifted smoothly.


## Building Blocks
* Peer-to-peer Networking: [libp2p](https://libp2p.io/)
* Workload Clustering: [libp2p-raft](https://github.com/libp2p/go-libp2p-raft)
* Standalone pods: [Podman](https://github.com/containers/libpod)
* Lightweight Kubernetes: [k3s.io](https://k3s.io/)
* Example P2P Database: [OrbitDB](https://github.com/orbitdb)


## Longterm
A reconsideration of Cri-O and Kubernetes instead of Podman while retaining the new innovative design decisions is possible. This will be evaluated on a second stage.

![BeeMesh Binary](assets/img/beemesh.png)
