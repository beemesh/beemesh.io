## Vision
Imagine a world where everyone understands everyone else. Workload would be continuously organized based on ability, capacity, trustworthiness and loosely distributed to meet the actual demand. When statefulness is required, ephemeral consensus just lives alongside the work. Would such a computational model scale globally?

## Problem Statement
When thinking about scalability in distributed systems, the first thing that comes to mind is consensus algorithms. Well, this is not a necessary requirement. Scalability for stateless workloads can be achieved without consensus and is limited only by available resources. Therefore, consensus is only required for stateful workloads. Consensus requires an underlying protocol that can uniquely identify all participants and counter partitioning. Thus, to have a robust and consistent consensus, we first need robust messaging.

So what is the underlying messaging used in Kubernetes? Actually, none. The reality is that trustworthiness is predefined manually and Kubernetes doesn't care about messaging. Instead, Kubernetes implements a monolithic consensus that overall manages cluster state, regardless of whether the workload requires it. If consensus is compromised, then all work is also affected.

Kubernetes networking is extensible by additional virtualization. Several clusters are disjoint and therefore require further extensions such as service meshes with replicated control planes or cluster federation as needed. Traditional services are excluded and must be considered separately. Despite the overall intention, this infrastructure-focused connectivity does not meet today's requirements for a fine-grained, resilient software design as an underlying messaging for scalable consensus.
