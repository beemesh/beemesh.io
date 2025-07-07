## Vision

As the digital landscape continues to evolve, the limitations of traditional system architecture and security models are becoming increasingly apparent. The growing demand for simpler, more flexible, resilient, and scalable systems necessitates a complete rethinking of the concepts and methodologies that underpin today’s computing and hyperscalers.

Current hyperscaler models are fundamentally flawed, relying on outdated principles that no longer align with the needs of a dynamic, distributed digital world. The future lies in a model built on loosely connected, meshed resources where different versions coexist seamlessly, much like the decentralized approach seen in technologies like Bittorrent, which effectively scales to millions of nodes.

In this new era, stateless workloads are becoming the standard, diminishing the need for rigid, stateful clustering of resources. Consistency should no longer be a one-size-fits-all solution but rather be tailored to the specific requirements of transient workloads. This adaptable consistency can be achieved through algorithms that operate alongside the workloads, providing the flexibility to manage various levels of consistency. For stateless workloads, these algorithms promote eventual consistency, fostering a more distributed and resilient approach to data management. For stateful workloads, they ensure strong consistency, synchronizing all data replicas to maintain integrity and reliability. Crucially, this adaptive model removes the scalability barriers inherent in current systems, enabling seamless scaling without compromising consistency, and ensuring that both integrity and performance are maintained while accommodating the diverse needs of different workloads.

Security, too, requires a fresh perspective. By embedding identities into userland communications using spiffe.io, security is enhanced through mutual authentication and continuous authorization, fully embracing zero-trust principles and providing a level of security and traceability that traditional models cannot achieve.

Moreover, this architecture is designed to be self-layering, with APIs and scheduling inherently decentralized. A key innovation would be a Kubernetes-compatible API, ensuring that existing ecosystems can transition smoothly to this new resource management model. These changes are essential for moving beyond current architectures.

While this vision is ambitious, it is not without challenges. The complexity of building systems that can proactively adapt to change is considerable. However, these obstacles are surmountable, and the potential benefits could be transformative for the industry. Though this vision may seem radical today, it is precisely such disruptive ideas that drive true innovation. Progress happens when we challenge established norms. As we move forward, it is crucial to engage in discussions about the future of system architecture and security, pushing the boundaries of what we currently believe is possible.

## Table of Contents

1.  Introduction
2.  Core Philosophy & Design Principles
3.  Architectural Overview
      * Component Breakdown
      * Communication Flow
      * Planes of Operation
4.  Core Concepts
      * Beemesh Binary (The Peer)
      * Beemesh Nodes (The Workers)
      * Google Cloud Pub/Sub
      * The `Task` Definition
      * `BeemeshDeployment` (Stateless Workloads)
      * `BeemeshStatefulSet` (Stateful Workloads)
      * Stateless Watchdog Sidecar
      * Raft Sidecar
      * The "Clone Function" (Sidecar-Driven Self-Healing)
      * Separation of Concerns
      * Engagement with the CAP Theorem
5.  Conceptual Implementation Details
      * Project Structure
      * Beemesh Binary Logic (`main.go`, `api.go`, `node.go`)
      * Workload Definitions (`types.go`)
      * Sidecar Logic (Conceptual)
      * Pub/Sub Setup Considerations
6.  Conceptual Usage Guide
      * Prerequisites
      * GCP Pub/Sub Setup
      * Running Beemesh Peers & Nodes
      * Deploying a Stateless Application
      * Deploying a Stateful Application
      * Observing Behavior (Limitations)
7.  Limitations, Trade-offs & Future Work
      * Current Limitations
      * Design Trade-offs
      * Potential Future Enhancements

-----

## 1\. Introduction

Beemesh is a Proof-of-Concept (POC) for a novel, decentralized "global mesh computing" platform. Unlike traditional container orchestrators that rely on a centralized control plane (even if distributed), Beemesh pushes orchestration intelligence to the very edge of the network, striving for a highly autonomous and self-healing environment.

The primary goal of this POC is to demonstrate that containerized workloads can be effectively managed and healed across a distributed network of machines using only:

  * **Podman** for local container execution.
  * **Google Cloud Pub/Sub** for all inter-component communication.
  * **Intelligent sidecar containers** responsible for application-level health and triggering self-healing.

This approach prioritizes extreme decentralization and simplicity of the core "orchestrator" binary over maintaining a globally consistent, real-time view of infrastructure state.

## 2\. Core Philosophy & Design Principles

  * **Extreme Decentralization:** No single point of failure at the control plane level. Every Beemesh binary instance can potentially act as an API endpoint, a task publisher, and a worker node.
  * **"Fire and Forget" Core Orchestrator:** The central Beemesh binary (peer) publishes tasks and immediately moves on. It does not actively track the runtime status or health of individual Pods.
  * **Application-Driven Self-Healing:** The responsibility for ensuring replica counts, application health, and triggering re-launches lies entirely within specialized sidecar containers running alongside the main application.
  * **Strict Separation of Concerns:**
      * The **Machine-Plane** (Beemesh Node/Podman) focuses solely on executing tasks. It does not care about application health or distributed consensus.
      * The **Workload-Plane** (Sidecars) focuses solely on application-level health, consistency, and desired state. It does not care about underlying machine details.
  * **Asynchronous Communication:** All communication leverages the Pub/Sub model, ensuring high scalability, decoupling, and resilience to transient network issues.
  * **Engagement with CAP Theorem:** Beemesh explicitly makes trade-offs:
      * The Beemesh orchestrator layer prioritizes **Availability (A)** and **Partition Tolerance (P)** over strong **Consistency (C)** of the global infrastructure state.
      * The Workload-Plane (sidecars) makes its own CAP choices: stateless Deployments lean towards A & P (with eventual consistency), while StatefulSets via Raft explicitly prioritize C & P for application data.

## 3\. Architectural Overview

### Component Breakdown

1.  **Beemesh Binary (The Peer):**

      * A single Go executable.
      * Exposes an HTTP API for user interaction (submitting workloads).
      * Publishes initial `Task` messages to Pub/Sub.
      * Subscribes to the same `scheduler-tasks` Pub/Sub topic to act as a worker node.
      * Does *not* maintain any long-term internal state about deployed workloads after initial dispatch.

2.  **Beemesh Nodes (The Workers):**

      * Any machine running the Beemesh binary.
      * Must have **Podman** installed and configured to run Kubernetes manifests.
      * Subscribes to `scheduler-tasks` to receive and execute `Task` messages.
      * Upon successful `podman play kube` initiation, acknowledges the Pub/Sub message.

3.  **Google Cloud Pub/Sub:**

      * The sole communication backbone.
      * `scheduler-tasks` Topic: Used by Beemesh peers (for initial deployments) and sidecars (for self-healing re-launches) to publish `Task` messages. Beemesh Nodes subscribe here.
      * Sidecar Coordination Topics (Conceptual): May be used by sidecars for their internal coordination (e.g., watchdogs exchanging heartbeats).

4.  **BeemeshDeployment / BeemeshStatefulSet Definitions:**

      * User-facing YAML manifests defining desired workloads, embedding standard Kubernetes Pod specifications.

5.  **Stateless Watchdog Sidecar:**

      * A container injected into `BeemeshDeployment` Pods.
      * Monitors its local application instance's health.
      * Coordinates with *other* watchdog instances of the same Deployment (via Pub/Sub) to determine the global count of running replicas.
      * If the count is low, it executes a "clone function" to publish a new `Task` for a replacement replica.

6.  **Raft Sidecar:**

      * A container injected into `BeemeshStatefulSet` Pods.
      * Implements the Raft consensus protocol for distributed state management and leader election within its application cluster.
      * Monitors its Raft cluster's health and membership.
      * If a Raft member is lost and the cluster is undersized, it executes a "clone function" to publish a new `Task` for the specific missing replica identity.

### Communication Flow

1.  **User Deployment:** User sends `BeemeshDeployment`/`StatefulSet` manifest (via HTTP POST) to *any* running Beemesh peer.
2.  **Initial Task Dispatch:** The receiving Beemesh peer parses the manifest, generates `N` initial `Task` messages (one for each replica), and publishes them to `scheduler-tasks`.
3.  **Node Execution:** Any Beemesh Node subscribed to `scheduler-tasks` receives a `Task` message, initiates `podman play kube`, and acknowledges the message.
4.  **Application Runs:** The Podman container(s) start, including the main application and its sidecar(s).
5.  **Sidecar Monitoring & Coordination:**
      * **Watchdog Sidecars:** Monitor local app health. Coordinate with peer watchdogs of the same Deployment via Pub/Sub to ascertain global active replica count.
      * **Raft Sidecars:** Participate in Raft consensus; monitor cluster membership.
6.  **Self-Healing Trigger:** If a sidecar detects a problem (e.g., local app failure, global replica count low, Raft member lost), it decides a new replica is needed.
7.  **"Clone Function" (Task Re-publishing):** The sidecar generates a new `Task` message (containing the Pod definition, potentially updated for the specific re-launch scenario, e.g., stable StatefulSet ID) and publishes it back to `scheduler-tasks`.
8.  **Re-launch:** Any available Beemesh Node picks up this new `Task` and launches the replacement Pod.

### Planes of Operation

  * **Control Plane (Distributed):**
      * **Initial Dispatch:** Handled by any Beemesh Peer instance.
      * **Reconciliation/Self-Healing:** Handled primarily by the **sidecars** themselves, which act as distributed controllers for their respective workloads.
      * **Communication Backbone:** Pub/Sub.
  * **Data Plane (Local):**
      * **Execution:** Handled by Beemesh Nodes via Podman.
      * **Application Data:** Managed directly by the application (e.g., Raft for StatefulSets).

## 4\. Core Concepts

### Beemesh Binary (The Peer)

The central executable that plays multiple roles. When started, it initializes its HTTP server for API requests and its Pub/Sub client for both publishing and subscribing.

### Beemesh Nodes (The Workers)

Any machine running the Beemesh binary, configured to connect to Pub/Sub and having Podman installed. They are the execution engine.

### Google Cloud Pub/Sub

The single, vital piece of infrastructure for all inter-component communication. Topics are ephemeral unless configured persistently in GCP.

  * `scheduler-tasks`: Main channel for Pod launch requests.
  * `beemesh-watchdog-events` (conceptual): Used by watchdog sidecars to broadcast presence/heartbeats for coordination.

### The `Task` Definition

A fundamental data structure representing a request to run a Pod. It encapsulates a standard Kubernetes `corev1.Pod` along with Beemesh-specific metadata.

```go
// In types.go
package types

import (
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"time"
)

type Task struct {
	corev1.Pod `json:",inline"` // Embeds the K8s Pod object

	TaskID string `json:"task_id"` // Unique ID for this specific task dispatch
	ExecuteAt time.Time `json:"execute_at"` // Timestamp for when task was created/should be executed

	// Beemesh-specific metadata for reconciliation/self-healing logic
	ParentKind string `json:"beemesh_parent_kind,omitempty"`   // e.g., "Deployment", "StatefulSet"
	ParentName string `json:"beemesh_parent_name,omitempty"`   // Name of the parent Beemesh object
	ReplicaIndex *int32 `json:"beemesh_replica_index,omitempty"` // For StatefulSets: the stable index (e.g., 0, 1, 2)
}

// BeemeshDeployment defines the desired state for a stateless application.
type BeemeshDeployment struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`
	Spec              BeemeshDeploymentSpec `json:"spec"`
}

type BeemeshDeploymentSpec struct {
	Replicas *int32                 `json:"replicas,omitempty"`
	Template PodTemplateSpec        `json:"template"` // Pod definition for the replicas
	// Might include configuration for the stateless watchdog sidecar here
	WatchdogSidecar WatchdogSidecarSpec `json:"watchdogSidecar,omitempty"`
}

type WatchdogSidecarSpec struct {
    Image string `json:"image"`
    // Other config for watchdog, e.g., health check path, coordination topic
}


// BeemeshStatefulSet defines the desired state for a stateful application.
type BeemeshStatefulSet struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`
	Spec              BeemeshStatefulSetSpec `json:"spec"`
}

type BeemeshStatefulSetSpec struct {
	Replicas    *int32          `json:"replicas,omitempty"`
	ServiceName string          `json:"serviceName"` // For network identity, similar to K8s headless service
	Template    PodTemplateSpec `json:"template"`    // Pod definition for the replicas
	RaftSidecar RaftSidecarSpec `json:"raftSidecar,omitempty"`
}

type RaftSidecarSpec struct {
    Image string `json:"image"`
    Port int32 `json:"port"`
    // Other Raft specific configuration, e.g., data path
}

type PodTemplateSpec struct {
	metav1.ObjectMeta `json:"metadata,omitempty"`
	corev1.PodSpec    `json:"spec"`
}
```

### `BeemeshDeployment` (Stateless Workloads)

A Beemesh-specific Custom Resource Definition (CRD) that defines a desired number of identical, stateless Pods.

  * **Key Behavior:** The Beemesh peer initially launches `N` tasks. The stateless watchdog sidecars then take over to ensure the actual running count stays at `N` across the mesh.

### `BeemeshStatefulSet` (Stateful Workloads)

A Beemesh-specific CRD defining a desired number of Pods with stable network identities, intended for stateful applications using a consensus protocol like Raft.

  * **Key Behavior:** The Beemesh peer initially launches `N` *identically named* tasks (e.g., `my-app-0`, `my-app-1`). The Raft sidecars then manage cluster membership and trigger re-launches of specific missing identities.

### Stateless Watchdog Sidecar

This conceptual sidecar is injected into every `BeemeshDeployment` Pod.

  * **Responsibilities:**
      * **Local Health Check:** Periodically checks the health of the main application container within its Pod.
      * **Global Replica Count Coordination:** Communicates with other watchdog sidecars of the *same Deployment* (e.g., by exchanging heartbeats or presence messages over a shared Pub/Sub topic dedicated to watchdog coordination). They collectively maintain a view of how many healthy replicas are currently active across the entire mesh for their Deployment.
      * **Deficit Detection:** If the coordinated global count falls below the `spec.replicas` for their Deployment, one watchdog (potentially after a lightweight coordination/election) determines a new replica is needed.
      * **"Clone Function" (Publish Task):** Generates a new `Task` message identical to the original Pod definition for a generic new replica and publishes it to `scheduler-tasks`.

### Raft Sidecar

This conceptual sidecar is injected into every `BeemeshStatefulSet` Pod.

  * **Responsibilities:**
      * **Raft Consensus:** Runs the Raft protocol, maintaining the application's distributed state, leader election, and cluster membership.
      * **Peer Discovery:** Must establish communication with other Raft sidecars (e.g., via `RAFT_PEERS` environment variable passed during Pod creation, possibly requiring a dynamic IP resolution mechanism or an external service discovery solution).
      * **Membership Monitoring:** Raft's internal mechanisms allow it to detect when a peer has failed or become unreachable.
      * **Missing Member Detection:** If a Raft member is lost and the cluster size drops below the desired `spec.replicas`, the Raft leader (or a designated cluster member) recognizes the need for a replacement.
      * **"Clone Function" (Publish Task):** Generates a new `Task` message *specifically for the missing replica identity* (e.g., `my-stateful-app-1`). This task contains the full Pod manifest with the Raft sidecar, ensuring consistent naming and configuration, and publishes it to `scheduler-tasks`.

### The "Clone Function" (Sidecar-Driven Self-Healing)

This is a conceptual method or logic within the sidecar binaries. When invoked, it means the sidecar constructs a `Task` message based on its own Pod's template and relevant parent information (Deployment/StatefulSet name, replica index), assigns a new `TaskID`, and publishes it to the `scheduler-tasks` Pub/Sub topic. This effectively tells the Beemesh system: "I need another instance of myself (or my specific replica)."

### Separation of Concerns & CAP Engagement

Beemesh strictly separates concerns:

  * **Machine-Plane (Node/Podman):** Cares only about executing tasks. It is "dumb" to application logic or global state.
  * **Workload-Plane (Sidecars):** Cares only about application-specific health and maintaining desired counts/consistency. This allows for highly specialized, application-aware healing.

This design makes explicit CAP trade-offs:

  * The Beemesh core orchestrator prioritizes **Availability (A)** and **Partition Tolerance (P)** by having no central state and using "fire and forget."
  * Stateless Deployments' sidecars favor **A & P** for the global replica count, accepting potential transient over-provisioning.
  * StatefulSet Raft sidecars prioritize **Consistency (C)** and **P** for application data, potentially sacrificing **A** during quorum loss, but still use the "clone function" for eventual **A** of the desired *number* of members.

## 5\. Conceptual Implementation Details

### Project Structure

```
beemesh/
├── main.go                     // Beemesh binary entry point (API server, node, publisher, subscriber setup)
├── api/                        // HTTP API handler logic
│   └── handlers.go             // HTTP handlers for /v1/deployments, /v1/statefulsets
├── pkg/
│   ├── types/                  // Struct definitions for Task, BeemeshDeployment, BeemeshStatefulSet
│   │   └── types.go
│   ├── pubsub/                 // Pub/Sub client wrapper
│   │   └── client.go
│   └── podman/                 // Podman execution wrapper
│       └── podman.go           // Functions to call `podman play kube`
├── cmd/
│   ├── sidecars/
│   │   ├── watchdog-sidecar/   // Go application for the stateless watchdog
│   │   │   └── main.go
│   │   └── raft-sidecar/       // Go application for the Raft sidecar
│   │       └── main.go
│   └── beemeshctl/             // Future CLI client (optional for POC, direct curl/httpie used initially)
├── manifests/                  // Example BeemeshDeployment/StatefulSet YAMLs
│   ├── my-stateless-app.yaml
│   └── my-stateful-app.yaml
└── go.mod
└── go.sum
```

### Beemesh Binary Logic (`main.go`, `api.go`, `node.go`)

**`main.go`:**

```go
package main

import (
	"context"
	"log"
	"net/http"
	"os"
	"time"

	"beemesh/api"
	"beemesh/pkg/pubsub"
	"beemesh/pkg/podman"
	"beemesh/pkg/types"

	"github.com/google/uuid"
	// k8s client-go for API types, if needed, but here using local types
	corev1 "k8s.io/api/core/v1"
)

const (
	projectID       = "your-gcp-project-id"
	schedulerTopicID = "beemesh-scheduler-tasks"
	nodeSubscriptionID = "beemesh-node-listener"
)

func main() {
	ctx := context.Background()

	// Initialize Pub/Sub client
	psClient, err := pubsub.NewClient(ctx, projectID)
	if err != nil {
		log.Fatalf("Failed to create Pub/Sub client: %v", err)
	}
	defer psClient.Close()

	// Ensure topics/subscriptions exist
	schedulerTopic, err := psClient.EnsureTopic(ctx, schedulerTopicID)
	if err != nil {
		log.Fatalf("Failed to ensure scheduler topic: %v", err)
	}
	nodeSub, err := psClient.EnsureSubscription(ctx, schedulerTopic, nodeSubscriptionID)
	if err != nil {
		log.Fatalf("Failed to ensure node subscription: %v", err)
	}

	// Beemesh Peer (API Server) setup
	publisher := func(task types.Task) error {
		return psClient.PublishTask(ctx, schedulerTopic, task)
	}
	apiServer := api.NewApiServer(publisher) // API server needs a way to publish tasks

	// Start HTTP API server
	go func() {
		log.Println("Beemesh API Server listening on :8080")
		if err := http.ListenAndServe(":8080", apiServer.Router()); err != nil && err != http.ErrServerClosed {
			log.Fatalf("API Server failed: %v", err)
		}
	}()

	// Beemesh Node (Worker) setup - integrated into the same binary
	log.Println("Beemesh Node starting task consumer...")
	err = nodeSub.Receive(ctx, func(ctx context.Context, msg *pubsub.Message) {
		var task types.Task
		if err := msg.DataTo(&task); err != nil {
			log.Printf("Node: Error unmarshaling task: %v", err)
			msg.Nack() // Nack if unable to unmarshal
			return
		}

		log.Printf("Node: Received Task %s for Pod %s (Parent: %s/%s, Replica: %v)",
			task.TaskID, task.Name, task.ParentKind, task.ParentName, task.ReplicaIndex)

		// Execute Pod via Podman
		if err := podman.PlayKube(task.Pod); err != nil {
			log.Printf("Node: Failed to run Pod %s: %v", task.Name, err)
			msg.Nack() // Nack if Podman execution fails
		} else {
			log.Printf("Node: Successfully initiated Pod %s. Acknowledging task.", task.Name)
			msg.Ack() // Ack if Podman execution initiated successfully (fire and forget)
		}
	})
	if err != nil {
		log.Fatalf("Node: Pub/Sub subscription receive error: %v", err)
	}

	log.Println("Beemesh system shutdown.")
}
```

**`api/handlers.go`:**

```go
package api

import (
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"time"

	"github.com/google/uuid"
	"github.com/gorilla/mux"

	"beemesh/pkg/types"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

type TaskPublisher func(task types.Task) error

type ApiServer struct {
	publisher TaskPublisher
	router    *mux.Router
}

func NewApiServer(publisher TaskPublisher) *ApiServer {
	s := &ApiServer{
		publisher: publisher,
		router:    mux.NewRouter(),
	}
	s.setupRoutes()
	return s
}

func (s *ApiServer) Router() *mux.Router {
	return s.router
}

func (s *ApiServer) setupRoutes() {
	s.router.HandleFunc("/v1/deployments", s.createDeployment).Methods("POST")
	s.router.HandleFunc("/v1/statefulsets", s.createStatefulSet).Methods("POST")
	// Future: Get/Delete endpoints, but actual state is distributed in sidecars
}

func (s *ApiServer) createDeployment(w http.ResponseWriter, r *http.Request) {
	var deploy types.BeemeshDeployment
	if err := json.NewDecoder(r.Body).Decode(&deploy); err != nil {
		http.Error(w, err.Error(), http.StatusBadRequest)
		return
	}

	log.Printf("API: Received request to create Deployment: %s (Replicas: %d)",
		deploy.Name, *deploy.Spec.Replicas)

	// Publish initial tasks for each replica
	for i := int32(0); i < *deploy.Spec.Replicas; i++ {
		pod := deploy.Spec.Template.PodSpec // Use the embedded PodSpec
		// Assign a unique name for the pod
		podName := fmt.Sprintf("%s-%s", deploy.Name, uuid.New().String()[:8]) // Unique name
		if pod.Containers == nil || len(pod.Containers) == 0 {
            log.Printf("Warning: Deployment %s Pod has no containers specified. Adding dummy.", deploy.Name)
            pod.Containers = []corev1.Container{{Name: "dummy", Image: "alpine/git", Command: []string{"sleep", "3600"}}}
        }

		// Inject stateless watchdog sidecar if specified in manifest
		if deploy.Spec.WatchdogSidecar.Image != "" {
            watchdogContainer := corev1.Container{
                Name:  "beemesh-watchdog",
                Image: deploy.Spec.WatchdogSidecar.Image,
                // Pass needed info for watchdog coordination and clone function
                Env: []corev1.EnvVar{
                    {Name: "BEEMESH_PARENT_KIND", Value: "Deployment"},
                    {Name: "BEEMESH_PARENT_NAME", Value: deploy.Name},
                    {Name: "BEEMESH_DESIRED_REPLICAS", Value: fmt.Sprintf("%d", *deploy.Spec.Replicas)},
                    {Name: "BEEMESH_SCHEDULER_TOPIC", Value: schedulerTopicID}, // Pass Pub/Sub topic for clone
                    {Name: "BEEMESH_WATCHDOG_COORD_TOPIC", Value: "beemesh-watchdog-events"}, // For internal watchdog coordination
                },
            }
            pod.Containers = append(pod.Containers, watchdogContainer)
        }
        // Ensure restartPolicy for resilience
        if pod.RestartPolicy == "" {
            pod.RestartPolicy = corev1.RestartPolicyAlways
        }


		task := types.Task{
			Pod: corev1.Pod{
				ObjectMeta: metav1.ObjectMeta{
					Name:   podName,
					Labels: map[string]string{
						"beemesh.deployment": deploy.Name,
						"beemesh.parent-kind": "Deployment",
						"beemesh.parent-name": deploy.Name,
					},
				},
				Spec: pod,
			},
			TaskID:      uuid.New().String(),
			ExecuteAt:   time.Now(),
			ParentKind:  "Deployment",
			ParentName:  deploy.Name,
		}

		if err := s.publisher(task); err != nil {
			log.Printf("API: Failed to publish task for Deployment %s replica %d: %v", deploy.Name, i, err)
			http.Error(w, "Failed to publish some tasks", http.StatusInternalServerError)
			return
		}
		log.Printf("API: Published initial task %s for Deployment %s", task.TaskID, deploy.Name)
	}

	w.WriteHeader(http.StatusAccepted)
	json.NewEncoder(w).Encode(map[string]string{"status": "Deployment accepted", "name": deploy.Name})
}

func (s *ApiServer) createStatefulSet(w http.ResponseWriter, r *http.Request) {
	var ss types.BeemeshStatefulSet
	if err := json.NewDecoder(r.Body).Decode(&ss); err != nil {
		http.Error(w, err.Error(), http.StatusBadRequest)
		return
	}

	log.Printf("API: Received request to create StatefulSet: %s (Replicas: %d)",
		ss.Name, *ss.Spec.Replicas)

	// Generate initial RAFT_PEERS list for sidecar (e.g., my-app-0,my-app-1,my-app-2)
	raftPeers := ""
	for i := int32(0); i < *ss.Spec.Replicas; i++ {
		raftPeers += fmt.Sprintf("%s-%d", ss.Name, i)
		if i < *ss.Spec.Replicas-1 {
			raftPeers += ","
		}
	}

	for i := int32(0); i < *ss.Spec.Replicas; i++ {
		pod := ss.Spec.Template.PodSpec
		replicaPodName := fmt.Sprintf("%s-%d", ss.Name, i) // Stable name for StatefulSet replica

		if pod.Containers == nil || len(pod.Containers) == 0 {
            log.Printf("Warning: StatefulSet %s Pod has no containers specified. Adding dummy.", ss.Name)
            pod.Containers = []corev1.Container{{Name: "dummy", Image: "alpine/git", Command: []string{"sleep", "3600"}}}
        }

		// Inject Raft sidecar
		if ss.Spec.RaftSidecar.Image != "" {
			raftContainer := corev1.Container{
				Name:  "beemesh-raft-sidecar",
				Image: ss.Spec.RaftSidecar.Image,
				Env: []corev1.EnvVar{
					{Name: "RAFT_NODE_ID", Value: replicaPodName},
					{Name: "RAFT_PEERS", Value: raftPeers}, // All expected peers
                    {Name: "BEEMESH_PARENT_KIND", Value: "StatefulSet"},
                    {Name: "BEEMESH_PARENT_NAME", Value: ss.Name},
                    {Name: "BEEMESH_REPLICA_INDEX", Value: fmt.Sprintf("%d", i)},
                    {Name: "BEEMESH_DESIRED_REPLICAS", Value: fmt.Sprintf("%d", *ss.Spec.Replicas)},
                    {Name: "BEEMESH_SCHEDULER_TOPIC", Value: schedulerTopicID}, // Pass Pub/Sub topic for clone
				},
				Ports: []corev1.ContainerPort{{ContainerPort: ss.Spec.RaftSidecar.Port}},
				// Mount data volume for Raft state (for POC, emptyDir or hostPath)
				VolumeMounts: []corev1.VolumeMount{{
					Name:      "raft-data",
					MountPath: "/var/lib/raft",
				}},
			}
			pod.Containers = append(pod.Containers, raftContainer)
            pod.Volumes = append(pod.Volumes, corev1.Volume{
                Name: "raft-data",
                VolumeSource: corev1.VolumeSource{
                    EmptyDir: &corev1.EmptyDirVolumeSource{}, // For POC, ephemeral
                },
            })
		}
        // Ensure stable network identity
        pod.Hostname = replicaPodName
        pod.Subdomain = ss.Spec.ServiceName
        if pod.RestartPolicy == "" {
            pod.RestartPolicy = corev1.RestartPolicyAlways
        }


		task := types.Task{
			Pod: corev1.Pod{
				ObjectMeta: metav1.ObjectMeta{
					Name:   replicaPodName,
					Labels: map[string]string{
						"beemesh.statefulset": ss.Name,
						"beemesh.replica-index": fmt.Sprintf("%d", i),
						"beemesh.parent-kind": "StatefulSet",
						"beemesh.parent-name": ss.Name,
					},
				},
				Spec: pod,
			},
			TaskID:       uuid.New().String(),
			ExecuteAt:    time.Now(),
			ParentKind:   "StatefulSet",
			ParentName:   ss.Name,
			ReplicaIndex: &i,
		}

		if err := s.publisher(task); err != nil {
			log.Printf("API: Failed to publish task for StatefulSet %s replica %d: %v", ss.Name, i, err)
			http.Error(w, "Failed to publish some tasks", http.StatusInternalServerError)
			return
		}
		log.Printf("API: Published initial task %s for StatefulSet %s replica %d", task.TaskID, ss.Name, i)
	}

	w.WriteHeader(http.StatusAccepted)
	json.NewEncoder(w).Encode(map[string]string{"status": "StatefulSet accepted", "name": ss.Name})
}
```

### Sidecar Logic (Conceptual)

Both sidecar types would be separate Go binaries (or other languages). They'd receive configuration via environment variables passed from the Pod manifest.

**`cmd/sidecars/watchdog-sidecar/main.go` (Conceptual):**

```go
package main

import (
	"context"
	"encoding/json"
	"log"
	"net/http"
	"os"
	"time"

	"beemesh/pkg/pubsub" // Re-use Beemesh's Pub/Sub client wrapper
	"beemesh/pkg/types"
	"github.com/google/uuid"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// Simplified. Real-world would need more robust coordination.
type WatchdogCoordMessage struct {
	PodName    string `json:"pod_name"`
	ParentName string `json:"parent_name"`
	NodeID     string `json:"node_id"` // Node where this watchdog is running
	Healthy    bool   `json:"healthy"`
}

func main() {
	// Get config from environment variables
	parentKind := os.Getenv("BEEMESH_PARENT_KIND")
	parentName := os.Getenv("BEEMESH_PARENT_NAME")
	desiredReplicasStr := os.Getenv("BEEMESH_DESIRED_REPLICAS")
	schedulerTopicID := os.Getenv("BEEMESH_SCHEDULER_TOPIC")
	coordTopicID := os.Getenv("BEEMESH_WATCHDOG_COORD_TOPIC")
	podName := os.Getenv("HOSTNAME") // Pod name is usually hostname in K8s, assume for Podman too

	// Parse desiredReplicas
	// ... (error handling)

	ctx := context.Background()
	psClient, err := pubsub.NewClient(ctx, "your-gcp-project-id") // Needs GCP project ID
	if err != nil {
		log.Fatalf("Watchdog: Failed to create Pub/Sub client: %v", err)
	}
	defer psClient.Close()

	schedulerTopic, err := psClient.EnsureTopic(ctx, schedulerTopicID)
	if err != nil {
		log.Fatalf("Watchdog: Failed to ensure scheduler topic: %v", err)
	}
	coordTopic, err := psClient.EnsureTopic(ctx, coordTopicID)
	if err != nil {
		log.Fatalf("Watchdog: Failed to ensure coordination topic: %v", err)
	}

	// --- Global Coordination & Replica Count ---
	// This is a highly simplified model for POC.
	// In reality, it would need a distributed consensus/gossip for accurate count.
	// We assume a simple broadcast/listen for now.
	activeReplicas := make(map[string]time.Time) // podName -> last_seen_healthy
	mu := sync.Mutex{}

	// Subscriber for coordination messages
	coordSub, err := psClient.EnsureSubscription(ctx, coordTopic, "watchdog-listener-"+podName)
	if err != nil {
		log.Fatalf("Watchdog: Failed to ensure coord subscription: %v", err)
	}
	go func() {
		err := coordSub.Receive(ctx, func(ctx context.Context, msg *pubsub.Message) {
			var coordMsg WatchdogCoordMessage
			if err := json.Unmarshal(msg.Data, &coordMsg); err != nil {
				log.Printf("Watchdog: Error unmarshaling coord message: %v", err)
				msg.Nack()
				return
			}
			mu.Lock()
			if coordMsg.Healthy {
				activeReplicas[coordMsg.PodName] = time.Now()
			} else {
				delete(activeReplicas, coordMsg.PodName)
			}
			mu.Unlock()
			msg.Ack()
		})
		if err != nil {
			log.Fatalf("Watchdog: Coord sub receive error: %v", err)
		}
	}()

	// Health check and Coordination Message Publisher Loop
	ticker := time.NewTicker(10 * time.Second)
	defer ticker.Stop()

	for range ticker.C {
		// 1. Local Health Check (conceptual, e.g., HTTP GET to app container)
		appHealthy := checkApplicationHealth() // Placeholder function

		// 2. Publish Coordination Message
		coordMsg := WatchdogCoordMessage{
			PodName:    podName,
			ParentName: parentName,
			NodeID:     os.Getenv("NODE_ID"), // Assume node exports its ID
			Healthy:    appHealthy,
		}
		data, _ := json.Marshal(coordMsg) // error handling omitted
		coordTopic.Publish(ctx, &pubsub.Message{Data: data})

		// 3. Evaluate Global Count & Trigger Clone
		mu.Lock()
		currentActive := 0
		for p, lastSeen := range activeReplicas {
			if time.Since(lastSeen) < 30*time.Second { // Consider alive if recent heartbeat
				currentActive++
			} else {
				delete(activeReplicas, p) // Remove stale
			}
		}
		mu.Unlock()

		desiredReplicas := 3 // Hardcoded for POC, should parse desiredReplicasStr

		if currentActive < desiredReplicas {
			log.Printf("Watchdog: Global replica count for %s is %d, desired %d. Triggering clone.", parentName, currentActive, desiredReplicas)
			// Call clone function
			cloneTask(ctx, schedulerTopic, parentKind, parentName, podName, psClient)
		} else {
			log.Printf("Watchdog: Global replica count for %s is %d, desired %d. All good.", parentName, currentActive, desiredReplicas)
		}
	}
}

// checkApplicationHealth is a placeholder for actual application health check logic
func checkApplicationHealth() bool {
	// In a real scenario, this would perform an HTTP GET, check process status, etc.
	// For POC, just return true for simplicity or add some random failure for testing.
	return true
}

// cloneTask is the "clone function" for stateless deployments
func cloneTask(ctx context.Context, schedulerTopic *pubsub.Topic, parentKind, parentName, originalPodName string, psClient *pubsub.Client) {
	log.Printf("Watchdog: Executing clone function for %s %s...", parentKind, parentName)

	// This is highly simplified. A real sidecar would need access to its *original* PodSpec
	// or have a stored template to recreate the exact Pod for the clone.
	// For POC, we assume it can reconstruct the original Pod's details needed for a new Task.
	// The new Pod needs a fresh name to avoid collisions on nodes.
	clonedPodName := fmt.Sprintf("%s-%s-clone-%s", parentName, originalPodName, uuid.New().String()[:6])

	// --- IMPORTANT: This Pod template should come from the original Deployment definition ---
	// For POC, we might have a hardcoded base template or assume sidecar can reconstruct.
	basePodSpec := corev1.PodSpec{
		Containers: []corev1.Container{{
			Name: "cloned-app",
			Image: "nginx:latest", // Needs to be configurable or derived from original
			// ... other app container details
		}, {
			Name: "beemesh-watchdog",
			Image: os.Getenv("BEEMESH_WATCHDOG_IMAGE"), // Sidecar image
			Env: []corev1.EnvVar{ // Pass essential env vars back to cloned sidecar
				{Name: "BEEMESH_PARENT_KIND", Value: parentKind},
				{Name: "BEEMESH_PARENT_NAME", Value: parentName},
				{Name: "BEEMESH_DESIRED_REPLICAS", Value: os.Getenv("BEEMESH_DESIRED_REPLICAS")},
				{Name: "BEEMESH_SCHEDULER_TOPIC", Value: schedulerTopic.ID()},
				{Name: "BEEMESH_WATCHDOG_COORD_TOPIC", Value: os.Getenv("BEEMESH_WATCHDOG_COORD_TOPIC")},
			},
		}},
        RestartPolicy: corev1.RestartPolicyAlways,
	}

	newTask := types.Task{
		Pod: corev1.Pod{
			ObjectMeta: metav1.ObjectMeta{
				Name: clonedPodName,
				Labels: map[string]string{
					"beemesh.deployment": parentName,
					"beemesh.parent-kind": parentKind,
					"beemesh.parent-name": parentName,
				},
			},
			Spec: basePodSpec,
		},
		TaskID:    uuid.New().String(),
		ExecuteAt: time.Now(),
		ParentKind: parentKind,
		ParentName: parentName,
	}

	if err := psClient.PublishTask(ctx, schedulerTopic, newTask); err != nil {
		log.Printf("Watchdog: Failed to publish clone task for %s: %v", parentName, err)
	} else {
		log.Printf("Watchdog: Successfully published clone task %s for %s.", newTask.TaskID, parentName)
	}
}
```

**`cmd/sidecars/raft-sidecar/main.go` (Conceptual):**

```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"time"

	"beemesh/pkg/pubsub" // Re-use Beemesh's Pub/Sub client wrapper
	"beemesh/pkg/types"
	"github.com/google/uuid"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// --- Raft implementation omitted for brevity ---
// Assume a Raft library is used (e.g., hashicorp/raft)
// Raft logic would handle peer discovery (using RAFT_PEERS),
// leader election, state replication, and membership changes.

func main() {
	// Get config from environment variables
	nodeID := os.Getenv("RAFT_NODE_ID") // e.g., my-stateful-app-0
	peersStr := os.Getenv("RAFT_PEERS") // e.g., my-stateful-app-0,my-stateful-app-1,my-stateful-app-2
	parentKind := os.Getenv("BEEMESH_PARENT_KIND")
	parentName := os.Getenv("BEEMESH_PARENT_NAME")
	replicaIndexStr := os.Getenv("BEEMESH_REPLICA_INDEX")
	desiredReplicasStr := os.Getenv("BEEMESH_DESIRED_REPLICAS")
	schedulerTopicID := os.Getenv("BEEMESH_SCHEDULER_TOPIC")
	raftPortStr := os.Getenv("RAFT_PORT") // Port Raft listens on

	// ... (Parse env vars, error handling)
	replicaIndex := int32(0) // Default or parse
	desiredReplicas := int32(3) // Default or parse

	ctx := context.Background()
	psClient, err := pubsub.NewClient(ctx, "your-gcp-project-id") // Needs GCP project ID
	if err != nil {
		log.Fatalf("Raft Sidecar: Failed to create Pub/Sub client: %v", err)
	}
	defer psClient.Close()

	schedulerTopic, err := psClient.EnsureTopic(ctx, schedulerTopicID)
	if err != nil {
		log.Fatalf("Raft Sidecar: Failed to ensure scheduler topic: %v", err)
	}

	// --- Raft Core Logic (Conceptual) ---
	// Initialize Raft node here using nodeID and peersStr.
	// This would involve setting up network listeners, storage, etc.
	// A placeholder to represent the Raft cluster's view of members:
	raftClusterMembers := make(map[string]bool) // nodeID -> is_active

	// Simulate Raft cluster health check loop
	ticker := time.NewTicker(10 * time.Second)
	defer ticker.Stop()

	for range ticker.C {
		// In a real Raft implementation, this check would be internal to the Raft library
		// e.g., check Raft's peer list, monitor last heartbeats/commit indices.
		// For POC, simulate a check.
		currentRaftMembers := simulateRaftMemberCheck(peersStr) // Placeholder

		// Update our conceptual view of active Raft members
		newlyMissing := []string{}
		for _, peer := range parsePeers(peersStr) { // Iterate all expected peers
			if _, isActive := currentRaftMembers[peer]; !isActive {
				if _, wasActive := raftClusterMembers[peer]; wasActive { // Was active, now missing
					log.Printf("Raft Sidecar %s: Detected missing peer %s", nodeID, peer)
					newlyMissing = append(newlyMissing, peer)
				}
				raftClusterMembers[peer] = false // Mark as inactive
			} else {
				raftClusterMembers[peer] = true // Mark as active
			}
		}

		// Count currently active Raft members
		activeCount := 0
		for _, active := range raftClusterMembers {
			if active {
				activeCount++
			}
		}

		// Trigger clone if cluster size is too low
		// Assuming 'desiredReplicas' is the target size for the Raft cluster
		if activeCount < int(desiredReplicas) {
			log.Printf("Raft Sidecar %s: Raft cluster size is %d, desired %d. Triggering clone.", nodeID, activeCount, desiredReplicas)
			// For simplicity, we trigger a clone for *any* missing peer
			// In a real system, you might trigger for specific, known missing replicaIndex.
			// The original logic of cloning a specific replica index is better here.
			// Assuming we iterate through 0 to desiredReplicas and check if it's active.
			for i := int32(0); i < desiredReplicas; i++ {
				expectedReplicaName := fmt.Sprintf("%s-%d", parentName, i)
				if _, isActive := currentRaftMembers[expectedReplicaName]; !isActive {
					log.Printf("Raft Sidecar %s: Requesting clone for missing replica %s", nodeID, expectedReplicaName)
					cloneRaftTask(ctx, schedulerTopic, parentKind, parentName, i, psClient)
					break // Only request one missing replica at a time (basic example)
				}
			}
		} else {
			log.Printf("Raft Sidecar %s: Raft cluster size is %d, desired %d. All good.", nodeID, activeCount, desiredReplicas)
		}
	}
}

// simulateRaftMemberCheck is a placeholder. Real Raft would have this internal.
func simulateRaftMemberCheck(peers string) map[string]bool {
	// In reality, this would involve Raft's internal state.
	// For POC, simulate some active members.
	active := make(map[string]bool)
	allPeers := parsePeers(peers)
	for i, peer := range allPeers {
		if i != 1 { // Simulate my-stateful-app-1 as potentially missing
			active[peer] = true
		}
	}
	return active
}

func parsePeers(peersStr string) []string {
	// Simple comma split
	return strings.Split(peersStr, ",")
}

// cloneRaftTask is the "clone function" for stateful workloads
func cloneRaftTask(ctx context.Context, schedulerTopic *pubsub.Topic, parentKind, parentName string, replicaIndex int32, psClient *pubsub.Client) {
	log.Printf("Raft Sidecar: Executing clone function for %s %s replica %d...", parentKind, parentName, replicaIndex)

	clonedPodName := fmt.Sprintf("%s-%d", parentName, replicaIndex) // Stable name
	raftPort := int32(7000) // Assumed, should come from env var

	// --- IMPORTANT: This Pod template should come from the original StatefulSet definition ---
	// For POC, we reconstruct or assume access to the original template.
	basePodSpec := corev1.PodSpec{
		Containers: []corev1.Container{{
			Name: "cloned-app",
			Image: "my-app-image:latest", // Needs to be configurable or derived
			// ... other app container details
		}, {
			Name:  "beemesh-raft-sidecar",
			Image: os.Getenv("BEEMESH_RAFT_SIDECAR_IMAGE"), // Sidecar image
			Env: []corev1.EnvVar{
				{Name: "RAFT_NODE_ID", Value: clonedPodName},
				{Name: "RAFT_PEERS", Value: os.Getenv("RAFT_PEERS")}, // Full original peers list
				{Name: "BEEMESH_PARENT_KIND", Value: parentKind},
				{Name: "BEEMESH_PARENT_NAME", Value: parentName},
				{Name: "BEEMESH_REPLICA_INDEX", Value: fmt.Sprintf("%d", replicaIndex)},
				{Name: "BEEMESH_DESIRED_REPLICAS", Value: os.Getenv("BEEMESH_DESIRED_REPLICAS")},
				{Name: "BEEMESH_SCHEDULER_TOPIC", Value: schedulerTopic.ID()},
			},
			Ports: []corev1.ContainerPort{{ContainerPort: raftPort}},
			VolumeMounts: []corev1.VolumeMount{{Name: "raft-data", MountPath: "/var/lib/raft"}},
		}},
        Hostname: clonedPodName,
        Subdomain: os.Getenv("BEEMESH_SERVICE_NAME"), // Needs to be passed
        Volumes: []corev1.Volume{{
            Name: "raft-data",
            VolumeSource: corev1.VolumeSource{EmptyDir: &corev1.EmptyDirVolumeSource{}},
        }},
        RestartPolicy: corev1.RestartPolicyAlways,
	}

	newTask := types.Task{
		Pod: corev1.Pod{
			ObjectMeta: metav1.ObjectMeta{
				Name: clonedPodName, // Stable name for stateful set
				Labels: map[string]string{
					"beemesh.statefulset": parentName,
					"beemesh.replica-index": fmt.Sprintf("%d", replicaIndex),
					"beemesh.parent-kind": parentKind,
					"beemesh.parent-name": parentName,
				},
			},
			Spec: basePodSpec,
		},
		TaskID:    uuid.New().String(),
		ExecuteAt: time.Now(),
		ParentKind: parentKind,
		ParentName: parentName,
		ReplicaIndex: &replicaIndex, // Crucial for stateful
	}

	if err := psClient.PublishTask(ctx, schedulerTopic, newTask); err != nil {
		log.Printf("Raft Sidecar: Failed to publish clone task for %s replica %d: %v", parentName, replicaIndex, err)
	} else {
		log.Printf("Raft Sidecar: Successfully published clone task %s for %s replica %d.", newTask.TaskID, parentName, replicaIndex)
	}
}
```

### Pub/Sub Setup Considerations

  * **GCP Project:** A Google Cloud Project is required.
  * **Service Account:** A GCP service account with `Pub/Sub Editor` role (or more granular `Publisher` and `Subscriber` roles) is needed. The JSON key file for this service account must be available to every Beemesh binary instance (peer and sidecars). Set `GOOGLE_APPLICATION_CREDENTIALS` environment variable.
  * **Topic/Subscription Creation:** The Beemesh binary itself *can* attempt to create topics and subscriptions if they don't exist (as shown in `pubsub.NewClient` conceptual code).

## 6\. Conceptual Usage Guide

### Prerequisites

1.  **Go Language:** Version 1.20+
2.  **Podman:** Installed and configured on all machines intended to be Beemesh Nodes.
3.  **Google Cloud Platform (GCP) Project:**
      * Enable Pub/Sub API.
      * Create a service account with necessary Pub/Sub permissions.
      * Download the JSON key file.
4.  **Container Images:**
      * `nginx:latest` (or your application image)
      * Pre-built images for `beemesh-watchdog-sidecar` and `beemesh-raft-sidecar` (you'd build these from `cmd/sidecars`). These images need to include `beemesh/pkg/pubsub` or equivalent client logic.

### GCP Pub/Sub Setup

Set `GOOGLE_APPLICATION_CREDENTIALS` environment variable to the path of your GCP service account JSON key file on every machine running a Beemesh binary or a sidecar.

```bash
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/your/gcp-service-account-key.json"
```

### Running Beemesh Peers & Nodes

You would compile the `main.go` into a single binary, then run it on multiple machines.

```bash
# On Machine 1 (e.g., your local dev machine, acts as API endpoint and a node)
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/key.json"
export NODE_ID="beemesh-node-1" # Unique ID for this machine
go run main.go

# On Machine 2 (e.g., a remote server, acts as a node)
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/key.json"
export NODE_ID="beemesh-node-2" # Unique ID for this machine
go run main.go
# Repeat for N machines
```

*Note: The `NODE_ID` is used by sidecars for coordination, not directly by the Beemesh node itself for now.*

### Deploying a Stateless Application

Create a YAML file (e.g., `my-stateless-app.yaml`):

```yaml
# manifests/my-stateless-app.yaml
apiVersion: beemesh.io/v1alpha1
kind: BeemeshDeployment
metadata:
  name: my-web-app
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: my-web-app
    spec:
      containers:
      - name: main-app
        image: nginx:latest
        ports:
        - containerPort: 80
      restartPolicy: Always
  watchdogSidecar:
    image: your-gcp-project-id/beemesh-watchdog-sidecar:latest # Path to your built image
```

Deploy using `curl` or `httpie` to one of your running Beemesh peers (e.g., on `Machine 1`):

```bash
curl -X POST -H "Content-Type: application/json" --data @manifests/my-stateless-app.yaml http://localhost:8080/v1/deployments
```

You would see logs on the Beemesh peer indicating task publication, and on Beemesh nodes indicating task reception and Podman execution.

### Deploying a Stateful Application

Create a YAML file (e.g., `my-stateful-app.yaml`):

```yaml
# manifests/my-stateful-app.yaml
apiVersion: beemesh.io/v1alpha1
kind: BeemeshStatefulSet
metadata:
  name: my-raft-cluster
spec:
  replicas: 3
  serviceName: my-raft-service # For stable hostnames
  template:
    metadata:
      labels:
        app: my-raft-node
    spec:
      containers:
      - name: main-app
        image: alpine/git # Placeholder for your stateful app, e.g., a database
        command: ["sleep", "3600"]
      restartPolicy: Always
  raftSidecar:
    image: your-gcp-project-id/beemesh-raft-sidecar:latest # Path to your built image
    port: 7000
```

Deploy using `curl`:

```bash
curl -X POST -H "Content-Type: application/json" --data @manifests/my-stateful-app.yaml http://localhost:8080/v1/statefulsets
```

You would see similar logs. The Raft sidecars would attempt to form a cluster. If you manually shut down a node, the remaining Raft sidecars (if configured correctly to re-publish tasks) would signal for a new replica.

### Observing Behavior (Limitations)

  * **No Central Dashboard:** There is no `kubectl get pods` equivalent.
  * **Logs are Key:** You must observe the logs of each running Beemesh peer/node and the sidecars (which you'd typically retrieve from Podman logs on each machine) to understand the system's behavior and healing actions.
  * **Pub/Sub Monitoring:** Use GCP's Pub/Sub console to see message flow on `scheduler-tasks` and `beemesh-watchdog-events` (if implemented).

## 7\. Limitations, Trade-offs & Future Work

### Current Limitations (as a POC)

1.  **Network Identity for StatefulSets:** Stable hostname resolution (`my-raft-cluster-0` to its IP) across different nodes is **not provided by Beemesh**. This is a critical blocker for robust StatefulSets. Solutions like mDNS, a custom DNS service managed by Beemesh, or an external service discovery tool (like Consul) would be required.
2.  **Persistent Storage:** The current POC doesn't address persistent storage for StatefulSets (uses `emptyDir` or `HostPath`). True persistence across node failures requires a distributed storage solution (e.g., NFS, GlusterFS, cloud-native volumes).
3.  **Observability & Debugging:** The lack of a central state view makes debugging complex at scale. There's no easy way to get a global picture of what's actually running or why something isn't healing.
4.  **Resource Management:** No awareness of node CPU, memory, disk, or network usage. Tasks are dispatched to any available node, potentially leading to overloaded machines.
5.  **Security:** Sidecars require Pub/Sub publishing credentials. Secure management of these credentials within a Pod is crucial (e.g., using GCP Workload Identity).
6.  **"Clone Function" Robustness:** While defined, the internal logic of watchdog/Raft sidecars to correctly reconstruct and publish new `Task` messages with accurate metadata (including the original Pod's full spec) is complex and needs careful implementation.

### Design Trade-offs

  * **Simplicity of Core vs. Complexity in Sidecars:** The Beemesh binary is simple, but the sidecars become highly complex distributed systems themselves.
  * **Availability over Consistency (Orchestrator):** The core orchestrator doesn't guarantee a consistent global view, potentially leading to transient over-provisioning if multiple sidecars detect a shortfall simultaneously before a replacement is fully launched and observed.
  * **No Centralized Control:** While a strength for resilience, it makes centralized management, monitoring, and debugging significantly harder.

### Potential Future Enhancements

1.  **Network Overlay/Service Discovery:** Implement a lightweight cross-node networking solution or integrate with an existing service mesh/discovery tool (e.g., Consul) for robust StatefulSet communication.
2.  **Persistent Volume Claims:** Define a mechanism for requesting and provisioning distributed persistent storage.
3.  **Basic Node Resource Reporting:** Nodes could periodically send simple resource reports (CPU/Mem usage) via Pub/Sub to allow Beemesh peers to make rudimentary scheduling hints.
4.  **CLI Tool (`beemeshctl`):** A command-line client to simplify user interaction.
5.  **Telemetry & Metrics:** Sidecars could publish metrics to a distributed monitoring system (e.g., Prometheus via Pushgateway or agents) to gain observability.
6.  **Authentication & Authorization:** Secure the HTTP API and Pub/Sub interactions more robustly.
7.  **Dynamic RAFT\_PEERS Generation:** More sophisticated mechanisms for Raft sidecars to dynamically discover and update their peer list, rather than relying on a static initial list.
8.  **Graceful Shutdown:** Implement logic for sidecars and Beemesh nodes to gracefully shut down containers and acknowledge messages.

This documentation provides a comprehensive overview of the Beemesh POC's innovative, decentralized architecture. Its successful implementation would demonstrate a compelling alternative in the distributed computing landscape.
