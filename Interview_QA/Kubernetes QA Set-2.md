# Advanced Kubernetes Interview Questions - Detailed Answers

## Kubernetes Architecture & Core Concepts

---

### 1. Explain the complete Kubernetes control plane architecture.

The Kubernetes control plane is the brain of the cluster, responsible for making global decisions about the cluster and detecting/responding to cluster events. It consists of several tightly integrated components that work together to maintain the desired state.

**Core Control Plane Components:**

**kube-apiserver** is the front-end of the control plane and the only component that directly communicates with etcd. It exposes the Kubernetes REST API, validates and processes API requests, performs authentication, authorization (RBAC), and admission control. It's designed to scale horizontally — you can run multiple instances behind a load balancer. Every component (kubelet, controllers, scheduler, kubectl) talks through the API server.

**etcd** is a distributed, consistent key-value store that serves as the single source of truth for all cluster data — configuration, state, secrets, and metadata. It uses the Raft consensus algorithm to maintain consistency across nodes. Typically deployed as a 3 or 5-node cluster for HA. Loss of etcd means loss of cluster state, so backups are critical.

**kube-scheduler** watches for newly created Pods with no assigned node and selects an optimal node for them based on resource requirements, affinity/anti-affinity rules, taints/tolerations, and other policies. It only makes the decision — the kubelet actually runs the Pod.

**kube-controller-manager** runs multiple controller loops bundled into a single binary: Node Controller, Replication Controller, Endpoint Controller, Service Account Controller, etc. Each controller watches the cluster state via the API server and works toward the desired state.

**cloud-controller-manager** handles cloud-provider-specific logic — managing load balancers, persistent volumes, node lifecycle in cloud environments (AWS, GCP, Azure). Separating it from kube-controller-manager allows cloud providers to evolve their integrations independently.

**Worker Node Components (technically not control plane, but essential):**

The kubelet runs on every node, registers the node with the API server, and ensures containers described in PodSpecs are running and healthy. kube-proxy maintains network rules for Service abstraction using iptables, IPVS, or eBPF. The container runtime (containerd, CRI-O) actually pulls images and runs containers.

**Communication Flow:** All components communicate exclusively through the API server. This hub-and-spoke design means no component talks directly to etcd except the API server, which provides a clean abstraction and security boundary.

---

### 2. How does the API Server communicate with etcd?

The API server is the **sole component** that interacts directly with etcd — this is a deliberate architectural decision that centralizes data access, enforces consistency, and simplifies security.

**Communication Mechanism:**

The API server uses the **etcd v3 gRPC API** over a secure TLS connection. The connection details (endpoints, certificates, keys) are configured via flags like `--etcd-servers`, `--etcd-cafile`, `--etcd-certfile`, and `--etcd-keyfile`.

**Storage Model:**

Kubernetes objects are serialized (by default in Protocol Buffers for efficiency, though JSON is also supported) and stored as key-value pairs in etcd. Keys follow a hierarchical pattern like `/registry/<resource-type>/<namespace>/<name>`. For example, a Pod named `nginx` in the `default` namespace is stored at `/registry/pods/default/nginx`.

**Watch Mechanism:**

This is critical. The API server uses etcd's **watch API** to stream changes in real time. When any object changes in etcd, etcd notifies the API server, which in turn fans out notifications to all clients (controllers, schedulers, kubelets) that have established watches. This is how the entire reconciliation model works — components don't poll; they watch.

**Resource Versioning:**

Every object has a `resourceVersion` field, which corresponds to etcd's modification index. This enables **optimistic concurrency control** — when you update an object, you send the resourceVersion you read; if it has changed in etcd, etcd rejects the write and the API server returns a 409 Conflict. This prevents lost updates without requiring locks.

**Caching Layer:**

The API server maintains a **watch cache** in memory to reduce direct etcd load. Most read requests (LIST, GET, WATCH) are served from this cache rather than hitting etcd directly. Only writes and consistent reads (with `resourceVersion=0` semantics) bypass the cache.

**Compaction & Defragmentation:**

etcd keeps a history of all changes (MVCC — Multi-Version Concurrency Control), which would grow unbounded. The API server periodically triggers compaction to remove old revisions, and defragmentation reclaims disk space.

**Security:** Mutual TLS (mTLS) is used between the API server and etcd, with separate certificate authorities recommended for the etcd cluster.

---

### 3. What happens internally when you run kubectl apply?

`kubectl apply` is far more sophisticated than a simple "send YAML to the server." Here's the complete internal flow:

**Step 1: Client-Side Processing**

kubectl reads your kubeconfig (typically `~/.kube/config`) to determine the cluster, user, and authentication credentials. It parses the YAML/JSON manifest and validates it locally (basic schema validation). It then determines the resource type and constructs the appropriate API endpoint URL.

**Step 2: Three-Way Merge Preparation**

Unlike `kubectl create` or `kubectl replace`, `apply` performs a **three-way merge**:
- **Last-applied configuration** (stored in the `kubectl.kubernetes.io/last-applied-configuration` annotation on the live object)
- **Current live object** (fetched from the API server)
- **New configuration** (your YAML file)

This allows apply to detect and remove fields you previously set but have now omitted, while preserving fields managed by other actors.

**Step 3: Authentication**

The request is sent over HTTPS to the API server. The API server authenticates using one of the configured methods — client certificates, bearer tokens, OIDC, webhook tokens, or service account tokens.

**Step 4: Authorization**

The API server checks if the authenticated user has permission to perform the operation on the resource. RBAC (Role-Based Access Control) is the most common authorizer, but ABAC, Node, and Webhook authorizers also exist.

**Step 5: Admission Control**

The request passes through two phases of admission controllers:
- **Mutating admission webhooks** (e.g., adding sidecars via Istio, injecting default values, setting resource limits via LimitRanger)
- **Validating admission webhooks** (e.g., OPA Gatekeeper, Kyverno policies, PodSecurity standards)

If any validating webhook rejects, the request fails.

**Step 6: Schema Validation**

The API server validates the object against the OpenAPI schema for that resource. CRDs use their own schemas defined in the CustomResourceDefinition.

**Step 7: Persistence to etcd**

The API server serializes the object and writes it to etcd. etcd assigns a new resourceVersion and replicates the write via Raft to its peers. Only after a quorum acknowledgment does the write succeed.

**Step 8: Watch Notifications**

etcd notifies the API server of the change, which then notifies all watchers. This triggers the cascade:
- The relevant controller (e.g., Deployment controller for a Deployment) sees the change and creates/updates child resources (ReplicaSet).
- The ReplicaSet controller creates Pods.
- The scheduler sees unscheduled Pods and assigns them to nodes.
- The kubelet on the target node sees Pods assigned to it and instructs the container runtime to pull images and start containers.

**Step 9: Response to Client**

kubectl receives the response and prints something like `deployment.apps/nginx configured` or `created`.

**Server-Side Apply (modern approach):** Since v1.22, Kubernetes supports server-side apply, where the merge logic runs on the API server instead of the client. It uses **field ownership** tracking — each field is owned by a specific manager (kubectl, a controller, etc.), preventing conflicts when multiple actors modify the same object.

---

### 4. Explain the lifecycle of a Pod from creation to termination.

A Pod's lifecycle is governed by its **phase** and the underlying container states. Understanding this is critical for debugging and designing resilient applications.

**Phase 1: Pending**

The Pod has been accepted by the API server (object persisted in etcd) but one or more containers are not yet running. Sub-stages include:
- **Scheduling** — kube-scheduler is selecting a node
- **Image pulling** — kubelet on the assigned node is pulling container images
- **Volume mounting** — Persistent Volumes are being attached and mounted

**Phase 2: Running**

The Pod has been bound to a node, all containers have been created, and at least one container is running, starting, or restarting.

Within Running, init containers execute **sequentially** before the main containers start. Each init container must complete successfully before the next runs. If any init container fails, the Pod restarts according to its restartPolicy.

Then main containers start, potentially in parallel. **Startup probes** run first (if defined) to determine when the app has started. Once startup succeeds, **liveness** and **readiness probes** take over:
- Liveness probe failure → container is killed and restarted
- Readiness probe failure → Pod is removed from Service endpoints (no traffic) but not killed

**Phase 3: Succeeded or Failed**

- **Succeeded**: All containers terminated with exit code 0 and won't restart (typical for Jobs)
- **Failed**: At least one container terminated with non-zero exit code or was killed by the system

**Phase 4: Unknown**

The Pod state cannot be determined, usually due to communication failure with the node hosting the Pod.

**Termination Sequence (Graceful Shutdown):**

When a Pod is deleted (via kubectl, scaling down, eviction, etc.):

1. The Pod is marked for deletion in etcd — its `deletionTimestamp` is set.
2. The Pod's state changes to **Terminating** and it's removed from Service endpoints immediately (so no new traffic arrives).
3. The kubelet receives the deletion event and begins the shutdown sequence.
4. **PreStop hook** executes if defined — useful for connection draining, deregistering from external systems, or flushing logs.
5. **SIGTERM** is sent to PID 1 in each container.
6. The grace period clock starts (default 30 seconds, configurable via `terminationGracePeriodSeconds`).
7. If containers exit cleanly within the grace period, the Pod is removed.
8. If the grace period expires, **SIGKILL** is sent and containers are forcibly terminated.
9. The kubelet cleans up volumes, network namespaces, and reports completion.
10. The API server removes the Pod object from etcd (after all finalizers have been processed).

**Restart Policy** governs container-level restarts within a Pod:
- `Always` (default for Deployments) — restart regardless of exit code
- `OnFailure` — restart only on non-zero exit
- `Never` — don't restart

**Finalizers** can delay actual deletion — the Pod stays in Terminating until all finalizers are removed by their respective controllers.

---

### 5. How does kube-scheduler make scheduling decisions?

The scheduler is one of the most sophisticated control plane components. It runs a continuous loop, watching for unscheduled Pods (those with `spec.nodeName == ""`) and assigning them to nodes through a two-phase process.

**Phase 1: Filtering (Predicates)**

The scheduler eliminates nodes that cannot run the Pod. Key filter plugins include:

- **NodeResourcesFit** — checks if the node has enough CPU, memory, ephemeral storage, and other resources to satisfy the Pod's requests.
- **NodeAffinity** — evaluates `nodeAffinity` rules (requiredDuringSchedulingIgnoredDuringExecution).
- **NodeName** — if Pod specifies a node name, only that node passes.
- **NodeUnschedulable** — excludes nodes marked unschedulable (cordoned).
- **TaintToleration** — excludes nodes with taints the Pod doesn't tolerate.
- **VolumeBinding** — verifies the Pod's volume requirements can be satisfied (PV availability, zone constraints).
- **PodTopologySpread** — enforces topology spread constraints.
- **InterPodAffinity** — evaluates pod affinity/anti-affinity rules.

If zero nodes pass filtering, the Pod stays Pending and the scheduler logs `FailedScheduling` events.

**Phase 2: Scoring (Priorities)**

Among the surviving nodes, the scheduler ranks them. Each scoring plugin returns a score from 0 to 100, and the scheduler computes a weighted sum:

- **NodeResourcesFit (scoring)** — supports strategies like LeastAllocated (spread workload), MostAllocated (bin packing), or RequestedToCapacityRatio.
- **InterPodAffinity** — prefers nodes that satisfy soft affinity rules.
- **NodeAffinity (preferred)** — bonuses for matching preferredDuringSchedulingIgnoredDuringExecution.
- **ImageLocality** — prefers nodes that already have the container images cached (faster start).
- **TaintToleration (scoring)** — prefers nodes where the Pod tolerates fewer taints.
- **PodTopologySpread (scoring)** — prefers balanced distribution across topology domains.

The node with the highest total score wins. If there's a tie, the scheduler picks randomly among the top scorers.

**Phase 3: Binding**

The scheduler creates a **Binding** object via the API server, which sets the Pod's `spec.nodeName`. This is what triggers the kubelet on that node to start the Pod.

**Scheduling Framework:**

Modern Kubernetes uses a plugin-based **Scheduling Framework** with extension points: QueueSort, PreFilter, Filter, PostFilter, PreScore, Score, NormalizeScore, Reserve, Permit, PreBind, Bind, PostBind. You can write custom plugins or even deploy multiple schedulers (`spec.schedulerName`).

**Advanced Features:**

- **Pod Preemption**: If a high-priority Pod can't be scheduled, the scheduler may evict lower-priority Pods to make room (PriorityClass).
- **Pod Overhead**: For runtimes with overhead (Kata Containers), this is factored into resource calculations.
- **Node Scoring with Caching**: The scheduler maintains an internal cache of node state to avoid hitting the API server constantly.

---

### 6. Explain kube-controller-manager responsibilities.

The kube-controller-manager is a single binary that runs **many controllers** as goroutines. Each controller is an independent reconciliation loop watching specific resources and driving them toward their desired state. Bundling them into one process reduces operational overhead.

**Major Controllers:**

**Node Controller** monitors node health by watching for Node objects. If a node stops sending heartbeats (NodeStatus updates), it marks the node `NotReady` after `--node-monitor-grace-period` (default 40s). After `--pod-eviction-timeout` (default 5 min), it evicts Pods from the unreachable node so they can be rescheduled.

**Replication Controller / ReplicaSet Controller** ensures the specified number of Pod replicas are running. It watches ReplicaSet objects and creates/deletes Pods to match `spec.replicas`. It uses label selectors to identify which Pods belong to it.

**Deployment Controller** manages Deployments and their owned ReplicaSets. It handles rolling updates by creating a new ReplicaSet, gradually scaling it up while scaling the old one down according to `maxSurge` and `maxUnavailable`. It also manages rollback history.

**StatefulSet Controller** manages stateful workloads with stable identities. It creates Pods sequentially (ordinal 0, 1, 2...) and waits for each to be Ready before creating the next. It maintains stable network identities and persistent storage per replica.

**DaemonSet Controller** ensures a Pod runs on every (or a selected subset of) node. When new nodes join, it creates Pods on them; when nodes leave, it cleans up.

**Job Controller** manages batch Jobs — creates Pods, tracks completions, and retries failed Pods up to `backoffLimit`. The CronJob controller schedules Jobs based on cron expressions.

**Endpoint Controller / EndpointSlice Controller** populates Endpoints (or EndpointSlices) objects by watching Services and matching Pods, providing the data plane for Service routing.

**Service Account Controller** creates default ServiceAccounts in every namespace and manages their associated tokens (in older versions; modern clusters use TokenRequest API for projected tokens).

**Namespace Controller** handles namespace lifecycle, especially deletion — it cascades deletion to all resources in the namespace.

**Persistent Volume Controller (PV/PVC binding)** matches PersistentVolumeClaims to available PersistentVolumes based on size, access modes, and storage class. It also handles dynamic provisioning by communicating with storage classes.

**Horizontal Pod Autoscaler (HPA) Controller** queries metrics (CPU, memory, custom metrics) and scales Deployments/ReplicaSets accordingly.

**Garbage Collector** handles cascading deletion via owner references. When an owner is deleted, dependents are deleted too.

**TTL Controller** automatically deletes finished Jobs after `ttlSecondsAfterFinished`.

**Common Pattern:** Every controller follows the same skeleton — use an informer to watch a resource type, enqueue events into a work queue, dequeue and reconcile by comparing desired vs actual state, and patch via the API server.

---

### 7. What are reconciliation loops in Kubernetes?

Reconciliation loops are the **fundamental design pattern** that makes Kubernetes declarative and self-healing. Every controller in Kubernetes implements this pattern.

**Core Concept:**

A reconciliation loop continuously:
1. Observes the **current state** of the cluster
2. Compares it to the **desired state** (what's declared in resource specs)
3. Takes action to **converge** current state toward desired state
4. Repeats forever

This is fundamentally different from imperative orchestration. You don't tell Kubernetes "create 3 pods" — you declare "I want 3 pods to exist," and a controller continuously ensures this is true.

**The Reconcile Function:**

In controller-runtime (the standard library for building operators), every controller implements:

```go
func (r *Reconciler) Reconcile(ctx context.Context, req Request) (Result, error)
```

This function is called whenever a watched object changes. It's expected to be:
- **Idempotent** — running it multiple times produces the same result
- **Level-triggered** — it acts on current state, not on the event that triggered it
- **Eventually consistent** — may take multiple iterations to converge

**Level-Triggered vs Edge-Triggered:**

Kubernetes is **level-triggered**, not edge-triggered. This is crucial. An edge-triggered system reacts to events (a Pod was deleted!), and if you miss the event, you're in trouble. A level-triggered system looks at the current state (there should be 3 pods, but there are 2) and acts. If the controller restarts or misses events, it still converges because it always re-evaluates the world.

**Example: ReplicaSet Reconciliation**

1. ReplicaSet says `replicas: 3`
2. Controller lists Pods matching the ReplicaSet's selector → finds 2
3. Desired - Actual = 1 Pod needs creation
4. Controller creates 1 Pod via API server
5. Eventually the new Pod is observed via watch
6. Controller re-reconciles → finds 3 Pods → no action needed
7. Later a node fails, 1 Pod disappears
8. Controller observes 2 Pods, creates 1 more — self-healing

**Work Queues and Backoff:**

Controllers use rate-limited work queues. If reconciliation fails, the item is requeued with exponential backoff (5ms, 10ms, 20ms... up to a cap). This prevents tight loops on persistent errors.

**Informers and Caches:**

Controllers don't poll the API server. They use **SharedInformers** that maintain a local cache populated via watch streams. Reads go to the cache (fast, no API server load), while writes go to the API server.

**Custom Controllers / Operators:**

The reconciliation pattern is so powerful it's been extended to custom resources via CRDs. Operators (built with Operator SDK, Kubebuilder) implement domain-specific reconciliation — e.g., "if PostgresCluster says 3 replicas, ensure 3 Postgres pods with proper replication setup, automated failover, backups, etc."

---

### 8. How does Kubernetes achieve desired state management?

Desired state management is Kubernetes' core value proposition. It's the combination of declarative APIs, persistent state storage, reconciliation loops, and watch-based event distribution working together.

**The Declarative Model:**

You describe **what** you want (3 replicas of nginx, exposed on port 80, with these resource limits), not **how** to achieve it. Compare to imperative orchestration where you'd script "if instance count < 3, launch a new instance."

**Component Breakdown:**

1. **Resource Specs as Desired State** — Every Kubernetes object has a `spec` (desired) and a `status` (observed). Users and controllers write to spec; controllers update status.

2. **etcd as Source of Truth** — All specs are persisted in etcd. Even if every other component crashes, the desired state survives.

3. **Controllers as the Convergence Engine** — Each controller is responsible for one resource type and works tirelessly to make `status` match `spec`. The cumulative effect of dozens of controllers running in parallel produces the cluster behavior you observe.

4. **Watch API for Efficiency** — Components don't poll. They watch for changes and react. This makes the system responsive (low latency to converge) and efficient (no constant polling load).

5. **Idempotency** — Operations are designed so retries are safe. Creating a Pod that already exists with the same spec is a no-op. This is essential for level-triggered reconciliation.

6. **Owner References & Garbage Collection** — Higher-level resources own lower-level ones (Deployment → ReplicaSet → Pod). When an owner is deleted, dependents are garbage-collected. This makes complex hierarchies manageable.

7. **Finalizers** — Allow controllers to perform cleanup before an object is actually deleted (e.g., releasing cloud resources, draining traffic). The object stays in `Terminating` state until all finalizers complete.

**Self-Healing Behaviors:**

- Pod dies → ReplicaSet controller creates a replacement
- Node dies → Node controller marks it NotReady, Pods are evicted and rescheduled
- Deployment update fails → can roll back via stored ReplicaSet history
- Service endpoints change → kube-proxy updates iptables/IPVS rules automatically
- Persistent storage fails → some storage drivers can remount or failover

**Failure Modes & Trade-offs:**

This model is **eventually consistent**, not immediately consistent. There's always a lag between declaring desired state and achieving it. For most workloads this is fine, but it means you can't assume instantaneous changes.

The system is also robust to component failures — if the scheduler is down, new Pods stay Pending but existing Pods keep running. If a controller is down, the corresponding resource type stops reconciling but other resources continue working.

---

### 9. Difference between Deployment, StatefulSet, and DaemonSet.

These three workload controllers serve fundamentally different purposes. Choosing the right one is a common architectural decision.

**Deployment**

Deployments are for **stateless** applications where Pods are interchangeable.

Key characteristics: Pods get random names (nginx-7d8f9c-xyz12). Pods are created in parallel and have no ordering guarantees. Any Pod can be replaced by any other Pod without consequence. Rolling updates are managed via underlying ReplicaSets, with configurable maxSurge and maxUnavailable. Rollback to previous versions is supported via revision history. Scaling is trivial — just change replicas.

Use cases include web servers, API backends, stateless microservices, batch processors that don't maintain state between requests.

**StatefulSet**

StatefulSets are for **stateful** applications requiring stable identity, ordered deployment, and persistent per-Pod storage.

Key characteristics: Pods get predictable names with ordinal indices (mysql-0, mysql-1, mysql-2). Pods are created and deleted in strict order (0 before 1 before 2 for creation; reverse for deletion). Each Pod gets a stable DNS hostname via a Headless Service (mysql-0.mysql.namespace.svc.cluster.local). Each Pod gets its own PersistentVolumeClaim from a volumeClaimTemplate — when a Pod is rescheduled, it reattaches to the same PVC. Rolling updates happen in reverse ordinal order, one Pod at a time. Pod identity persists across rescheduling and restarts.

Use cases include databases (MySQL, PostgreSQL, MongoDB), distributed systems with stable membership (Kafka, Elasticsearch, Cassandra, ZooKeeper, etcd), and any application where Pod identity matters for clustering or sharding.

**DaemonSet**

DaemonSets ensure a copy of a Pod runs on **every node** (or every node matching a selector).

Key characteristics: One Pod per node, automatically created when new nodes join the cluster and removed when nodes leave. The scheduler doesn't pick nodes — the DaemonSet controller bypasses normal scheduling. Supports rolling updates with configurable maxUnavailable. Often used with hostNetwork, hostPath volumes, or privileged access. Can be restricted to specific nodes via nodeSelector or nodeAffinity.

Use cases include log collectors (Fluentd, Filebeat, Vector), monitoring agents (Prometheus node-exporter, Datadog agent), network plugins (Calico, Cilium, Flannel), storage plugins (CSI drivers, Ceph), and security agents.

**Summary Comparison Table:**

| Aspect | Deployment | StatefulSet | DaemonSet |
|--------|------------|-------------|-----------|
| Pod Identity | Ephemeral, random | Stable, ordinal | Per-node |
| Storage | Shared or none | Per-Pod PVC via template | Usually hostPath |
| Scheduling | Scheduler decides | Scheduler decides | One per node, controller decides |
| Scaling | Arbitrary, parallel | Arbitrary, but ordered | Tied to node count |
| Update Order | Configurable (maxSurge/maxUnavailable) | Reverse ordinal, one at a time | Configurable |
| Use Case | Stateless apps | Databases, clustered systems | Node-level agents |

---

### 10. Explain static Pods and their use cases.

Static Pods are a special category of Pods managed directly by the **kubelet** on a specific node, rather than by the API server and control plane.

**How They Work:**

The kubelet watches a configured directory (typically `/etc/kubernetes/manifests/`) for Pod manifest YAML files. When it finds one, it creates and runs the Pod directly via the container runtime. The kubelet also creates a **mirror Pod** in the API server so the Pod is visible via `kubectl get pods`, but you cannot modify or delete the Pod through the API — you must edit or remove the manifest file on the node's filesystem.

**Configuration:**

The kubelet's static Pod path is configured via the `--pod-manifest-path` flag or `staticPodPath` in its config file. It can also watch an HTTP URL via `--manifest-url`. The kubelet polls this location periodically and reacts to file changes by creating, updating, or deleting Pods.

**Key Characteristics:**

Static Pods don't depend on the API server, scheduler, or any controller. The kubelet runs them autonomously. The Pod's name is automatically suffixed with the node's hostname (e.g., `kube-apiserver-master01`). Mirror Pods cannot be modified via API; changes only stick if you edit the manifest file on disk. They have a self-healing behavior local to the node — if a container crashes, the kubelet restarts it according to restartPolicy. They cannot be scheduled — they always run on the specific node where their manifest lives. ReplicaSet/Deployment semantics don't apply.

**Primary Use Case: Control Plane Bootstrapping**

This is the killer use case. The Kubernetes control plane has a chicken-and-egg problem — the API server, etcd, scheduler, and controller-manager need to be running, but normal Pods require the API server to schedule them.

Static Pods break this cycle. Tools like **kubeadm** install the control plane components as static Pods. When the kubelet starts on a control plane node, it sees manifests for kube-apiserver, kube-controller-manager, kube-scheduler, and etcd in `/etc/kubernetes/manifests/` and brings them up directly. Once the API server is running, it can manage everything else normally. If a control plane component crashes, the kubelet restarts it — no external orchestration needed.

**Other Use Cases:**

Node-level system components that must run before the cluster is fully functional. Air-gapped or edge environments where you need a Pod to run regardless of control plane availability. Bootstrap agents that configure the node before joining the cluster. Recovery scenarios — if the API server is broken, you can still run troubleshooting Pods via static manifests.

**Limitations:**

You can't use Services, ConfigMaps, or Secrets as easily — though you can mount them via hostPath. They don't participate in normal scheduling, autoscaling, or affinity rules. Managing them at scale across many nodes is awkward — usually you'd use DaemonSets instead unless control-plane bootstrapping is involved. No rolling updates — you have to manually update manifests on each node.

**Inspecting Static Pods:**

```bash
# View manifest files on the node
ls /etc/kubernetes/manifests/

# See mirror Pods (note the node name suffix)
kubectl get pods -n kube-system
# Output includes: kube-apiserver-master01, etcd-master01, etc.

# Check kubelet config for path
ps aux | grep kubelet | grep pod-manifest-path
```

If you delete a mirror Pod with `kubectl delete`, the kubelet will recreate it from the manifest file within seconds — only removing the manifest file truly deletes a static Pod.

---

# Advanced Kubernetes Interview Questions - Part 2

## Node Components & Cluster Internals

---

### 1. How does kubelet communicate with the control plane?

The kubelet is the primary node agent that runs on every node in the cluster. Its communication with the control plane is exclusively through the **kube-apiserver** — it never talks directly to etcd, the scheduler, or other components.

**Connection Establishment:**

When kubelet starts, it reads its kubeconfig file (typically `/etc/kubernetes/kubelet.conf`) which contains the API server endpoint, the cluster CA certificate, and the kubelet's own client credentials. The connection is established over **HTTPS with mutual TLS** — both sides authenticate each other.

**Authentication Methods:**

The kubelet authenticates to the API server using one of:
- **X.509 client certificates** — most common, with the certificate's CN identifying the node (e.g., `system:node:worker01`) and the Organization (O) field being `system:nodes`
- **Bootstrap tokens** — used during initial node registration before the kubelet has a proper certificate
- **TLS Bootstrapping** — the kubelet uses a bootstrap token to request a real client certificate via the CertificateSigningRequest API, which is approved (manually or automatically) and then used for subsequent communication

**Authorization:**

The API server uses the **Node Authorizer** specifically designed for kubelet requests. It grants kubelets only the permissions they need — typically read access to Pods, Services, Endpoints, Nodes, and Secrets/ConfigMaps that their Pods actually reference. Combined with the **NodeRestriction admission plugin**, this prevents one node's kubelet from reading or modifying resources belonging to other nodes.

**Watch-Based Communication:**

The kubelet establishes long-lived **watch connections** to the API server for:
- Pods assigned to its node (filtered by `spec.nodeName`)
- Secrets and ConfigMaps referenced by those Pods
- Node objects (its own state)

When the API server detects a change (e.g., a new Pod is scheduled to this node), it pushes the event through the watch stream. The kubelet acts on it — pulling images, starting containers, etc.

**Status Updates (Node Heartbeats):**

The kubelet reports node status back to the API server through two mechanisms:

1. **Node Status Updates** — periodically (default every 5 minutes, or whenever state changes significantly), kubelet PATCHes the Node object with current status: capacity, allocatable resources, conditions (Ready, MemoryPressure, DiskPressure, PIDPressure), addresses, and so on.

2. **Node Lease** — a lightweight heartbeat in the `kube-node-lease` namespace. The kubelet updates a Lease object every 10 seconds (default `--node-status-update-frequency` divided). This is much cheaper than full status updates and is what the Node Controller primarily uses to detect node failures. Introduced in v1.14 to scale to thousands of nodes.

**Pod Status Updates:**

The kubelet continuously reports Pod status back via PATCH operations — container states, restart counts, IPs, conditions. These updates flow through the API server to anyone watching, including the Endpoints controller (which then updates Service endpoints).

**Reverse Direction (API Server → Kubelet):**

Sometimes the API server needs to call **into** the kubelet — for `kubectl exec`, `kubectl logs`, `kubectl port-forward`, and `kubectl attach`. The API server connects to the kubelet's HTTPS endpoint (port 10250) and the kubelet proxies the request to the container runtime. This is why the API server needs to validate the kubelet's serving certificate (`--kubelet-certificate-authority` flag).

**Container Runtime Interaction:**

While not strictly control plane communication, the kubelet talks to the container runtime (containerd, CRI-O) via the **CRI (Container Runtime Interface)** over a Unix domain socket — typically `/run/containerd/containerd.sock` or `/var/run/crio/crio.sock`.

---

### 2. Explain CRI, CNI, and CSI architecture.

These three interfaces are the **plugin boundaries** that make Kubernetes pluggable for containers, networking, and storage. They're gRPC or executable-based specifications that decouple Kubernetes from specific implementations.

**CRI — Container Runtime Interface**

CRI is the gRPC-based API that the kubelet uses to talk to container runtimes. Before CRI existed, kubelet had hardcoded support for Docker. CRI was introduced in v1.5 to allow any compliant runtime to plug in.

The kubelet acts as the **gRPC client**, and the runtime exposes a **gRPC server** on a Unix socket. The interface has two main services:

- **RuntimeService** — manages the Pod and container lifecycle: RunPodSandbox, CreateContainer, StartContainer, StopContainer, RemoveContainer, ListContainers, exec, attach, port-forward
- **ImageService** — manages container images: PullImage, ListImages, RemoveImage, ImageStatus

**Architecture in practice:**

```
kubelet → CRI gRPC → containerd → runc → Linux namespaces/cgroups → container process
```

The "Pod sandbox" concept is central — the runtime first creates a sandbox (the pause container that holds network namespace, IPC, etc.), then creates application containers that share that sandbox. This is how Pod semantics are implemented.

Major implementations: **containerd** (most common, used by EKS, GKE, AKS defaults), **CRI-O** (purpose-built for Kubernetes, used by OpenShift), and historically **dockershim** (removed in v1.24).

**CNI — Container Network Interface**

CNI is a CNCF specification for configuring network interfaces in Linux containers. Unlike CRI, CNI is **not gRPC** — it's a much simpler model based on executables and JSON.

When a Pod is created, the kubelet (via the runtime) invokes CNI plugins by:
1. Finding plugin configurations in `/etc/cni/net.d/` (JSON files defining the network)
2. Executing plugin binaries from `/opt/cni/bin/`
3. Passing parameters via environment variables (CNI_COMMAND, CNI_CONTAINERID, CNI_NETNS, CNI_IFNAME) and JSON config via stdin
4. Receiving result JSON via stdout

**Commands:** ADD (set up network for new Pod), DEL (tear down), CHECK (verify), VERSION.

**Plugin Chaining:**

CNI configs often **chain** multiple plugins. A typical chain might be: a main plugin to create the interface and assign an IP, then a portmap plugin to set up iptables NAT rules for hostPort, then a bandwidth plugin for traffic shaping.

**Major Implementations:**

- **Calico** — uses BGP for routing, popular for network policy
- **Cilium** — eBPF-based, high performance, advanced observability and security
- **Flannel** — simple overlay (VXLAN), often used in small clusters
- **Weave Net** — mesh networking with encryption
- **AWS VPC CNI** — assigns real VPC ENIs and IPs to Pods
- **Azure CNI, GCP's networking plugin** — cloud-native implementations

**CSI — Container Storage Interface**

CSI is a gRPC-based interface for storage providers, designed to decouple storage drivers from Kubernetes core. Before CSI, storage drivers were "in-tree" (compiled into kubelet/controller-manager). CSI moved them out-of-tree, allowing independent development and updates.

**CSI Architecture has three services:**

- **Identity Service** — returns plugin info, capabilities
- **Controller Service** — runs once per cluster (typically as a Deployment), handles cluster-wide operations: CreateVolume, DeleteVolume, ControllerPublishVolume (attach to node), ControllerUnpublishVolume (detach), CreateSnapshot, DeleteSnapshot, ExpandVolume
- **Node Service** — runs on every node (as a DaemonSet), handles node-local operations: NodeStageVolume (mount to a global path), NodePublishVolume (bind mount into Pod), NodeUnpublishVolume, NodeUnstageVolume, NodeGetVolumeStats

**Sidecar Containers:**

CSI drivers don't run alone — they work with sidecars provided by the Kubernetes community:
- **external-provisioner** — watches PVCs and calls CreateVolume
- **external-attacher** — handles attach/detach by watching VolumeAttachment objects
- **external-resizer** — handles volume expansion
- **external-snapshotter** — manages snapshots
- **node-driver-registrar** — registers the CSI driver with kubelet

**Major Implementations:** AWS EBS CSI Driver, Azure Disk/File CSI, GCE PD CSI, Ceph CSI, Portworx, Longhorn, OpenEBS, Rook.

**Common Pattern Across CRI/CNI/CSI:**

All three follow the same philosophy: define a stable interface, push implementation details out of Kubernetes core, allow third parties to innovate independently. This is why Kubernetes has such a rich ecosystem.

---

### 3. What happens if kubelet stops working on a node?

When kubelet fails, the impact unfolds in stages over several minutes, and the consequences depend on what fails and for how long.

**Immediate Impact (0-10 seconds):**

The kubelet stops updating its Node Lease object (in `kube-node-lease` namespace) and stops PATCHing Node status. However, **containers continue running** — they're managed by the container runtime (containerd, CRI-O) directly, not by kubelet. The kubelet only orchestrates; it doesn't host the containers.

This is an important distinction: kubelet death ≠ container death. Workloads continue serving traffic as long as the network and the container runtime are healthy.

**Detection Phase (~40 seconds):**

The Node Controller in kube-controller-manager watches Node Lease updates. When the lease hasn't been refreshed within `--node-monitor-grace-period` (default 40 seconds), the controller marks the Node's Ready condition as `Unknown`.

**Tainting Phase:**

Once the node is marked NotReady or Unknown, the Node Controller adds taints to the node:
- `node.kubernetes.io/not-ready:NoExecute` (if Ready=False)
- `node.kubernetes.io/unreachable:NoExecute` (if Ready=Unknown)

**Eviction Phase (~5 minutes default):**

Pods on the unreachable node have default tolerations for these taints with `tolerationSeconds: 300`. After 300 seconds, the **TaintBasedEviction** mechanism deletes the Pods from the API server.

Crucially, deletion from the API server **does not actually stop the containers** on the unreachable node — kubelet is dead and can't process the deletion. The Pod objects are removed from API state, allowing their controllers (ReplicaSet, StatefulSet) to create replacement Pods on healthy nodes.

**Special Case: StatefulSets**

For StatefulSets, this creates a problem. The controller won't create a replacement Pod with the same identity (mysql-0) until the old one is **fully gone**. Since the old Pod might still be running on the unreachable node (kubelet just can't report it), there's a risk of two Pods with the same identity accessing the same persistent volume.

By default, StatefulSet Pods on unreachable nodes **stuck in Terminating state forever** — they need manual intervention (`kubectl delete pod --force --grace-period=0`) to confirm the old Pod is really gone before the replacement is created. This is the "fencing" problem and is intentional — better to have downtime than data corruption.

**Network Implications:**

- Service endpoints pointing to Pods on the failed node remain until kubelet reports the Pod as not Ready or the Pod is deleted. kube-proxy on other nodes still routes traffic there.
- Readiness probes can't run, but they were last reported as healthy. Traffic continues flowing until eviction.
- This is why **TCP-level health checks at load balancers** are important — they catch failures faster than Kubernetes' eviction process.

**Recovery:**

When kubelet restarts:
- It re-establishes the watch with the API server
- It reconciles its local state with what the API server expects
- It may find Pods that have been "deleted" from API but still running locally — it cleans these up
- New Pods scheduled during the outage now get processed

**Mitigation Strategies:**

- **Node Problem Detector** — DaemonSet that detects various node issues and reports them as Node conditions
- **kubelet should run as a systemd service with `Restart=always`** so it auto-restarts on crash
- **Pod Disruption Budgets** ensure that even during evictions, minimum replicas stay available on other nodes
- **Topology spread constraints / pod anti-affinity** distribute replicas across nodes so one failure doesn't take everything down
- **Tune `--node-monitor-grace-period` and toleration seconds** for faster failure response if your workload tolerates aggressive evictions

---

### 4. Explain the role of kube-proxy in Kubernetes.

kube-proxy is a network component that runs on every node (typically as a DaemonSet) and implements the **Service abstraction**. Without kube-proxy, Service VIPs would be meaningless — they're just virtual IPs that need someone to actually route traffic to backing Pods.

**Core Responsibility:**

When you create a Service like `ClusterIP: 10.96.50.20`, that IP doesn't exist on any network interface. It's purely virtual. kube-proxy programs the node's kernel networking (iptables/IPVS/nftables) so that when any process on the node sends traffic to 10.96.50.20:80, it gets transparently redirected to one of the actual Pod IPs backing that Service.

**What kube-proxy Does:**

1. **Watches the API server** for Services, Endpoints (or EndpointSlices), and other relevant resources
2. **Translates these objects into kernel-level rules** on the local node
3. **Maintains the rules** as Services and Pods change — adding rules when Pods become ready, removing when they're deleted

**Service Types and kube-proxy:**

- **ClusterIP** — kube-proxy creates rules that DNAT traffic destined to the cluster IP to one of the backing Pod IPs (with load balancing)
- **NodePort** — additionally opens a port on every node, traffic to that port gets DNAT'd to the Service's endpoints
- **LoadBalancer** — the cloud provider provisions an external LB that routes to NodePort; kube-proxy handles the rest
- **Headless Services (`clusterIP: None`)** — kube-proxy does NOT process these. They have no VIP. Clients use DNS to get all backing Pod IPs directly.

**What kube-proxy Does NOT Do:**

- It's not a proxy in the L7 (HTTP) sense — it's purely L3/L4 (IP and port-level routing)
- It doesn't terminate TLS, route based on hostnames or paths (that's Ingress)
- It doesn't handle Pod-to-Pod traffic directly (that's the CNI plugin)
- It doesn't do DNS resolution (that's CoreDNS)

**Load Balancing:**

kube-proxy implements simple load balancing — typically random selection (in iptables mode) or round-robin (in IPVS mode). It's connection-level, not request-level — once a connection is established to a Pod, it stays with that Pod until the connection closes.

**Operating Modes:**

- **iptables mode** (default) — uses Linux iptables for rule programming
- **IPVS mode** — uses IPVS (Linux kernel L4 load balancer) for better performance
- **nftables mode** — newer alternative to iptables, becoming stable in recent versions
- **kernelspace mode** — Windows-specific implementation

**Externally:**

kube-proxy also handles **session affinity** (sticky sessions via client IP when `service.spec.sessionAffinity: ClientIP`), **source IP preservation** in certain configurations (`externalTrafficPolicy: Local`), and integration with `topologyKeys` / topology-aware routing.

**Modern Alternatives:**

Some CNI plugins replace kube-proxy entirely. **Cilium**, in kube-proxy replacement mode, uses eBPF to implement Service routing at a lower level with better performance and observability. This is increasingly popular in performance-sensitive environments.

---

### 5. Difference between iptables and IPVS kube-proxy modes.

Both modes achieve the same goal — routing Service traffic to Pods — but with significantly different performance characteristics and operational behavior.

**iptables Mode:**

kube-proxy programs iptables rules in the NAT table. For each Service, it creates a chain of rules that probabilistically select a backing Pod and DNAT the packet to that Pod's IP.

The structure looks like: `KUBE-SERVICES` chain matches packets destined to Service VIPs, jumps to `KUBE-SVC-*` chains (one per Service), which use the `statistic` module with `mode random probability X` to distribute packets to `KUBE-SEP-*` chains (one per endpoint), which finally do the DNAT.

**Performance Characteristics:**

iptables rules are evaluated **sequentially** in a linear fashion through chains. For 100 Services with 10 endpoints each, that's potentially thousands of rules being evaluated per packet. As cluster size grows, this becomes a bottleneck:

- Rule update time scales O(n) with number of Services — adding one Service requires rewriting many rules
- Packet processing latency increases with rule count
- During large bulk updates (e.g., rolling restart of a large Deployment), there can be noticeable CPU spikes and brief packet drops

Loads balancing is **random selection** based on probability — no awareness of backend health beyond what Endpoints tell it.

**IPVS Mode:**

IPVS (IP Virtual Server) is a Layer 4 load balancer built into the Linux kernel. It uses **hash tables** for rule lookup, providing O(1) performance regardless of cluster size.

When kube-proxy is in IPVS mode, it programs IPVS virtual servers and real servers, while still using a small number of iptables rules for things like SNAT, NodePort handling, and Service IP filtering.

**Performance Characteristics:**

- Rule lookup is O(1) via hash tables — performance doesn't degrade with cluster size
- Programmed natively in kernel space with high throughput
- Significantly lower latency for large clusters (typically >1000 Services)
- Updates are faster — adding/removing endpoints is a single IPVS operation

IPVS supports multiple **scheduling algorithms**: rr (round-robin), wrr (weighted round-robin), lc (least connection), wlc (weighted least connection), sh (source hashing), dh (destination hashing), sed (shortest expected delay), nq (never queue). Kubernetes defaults to rr but you can configure others via the `--ipvs-scheduler` flag.

**Trade-offs and Comparison:**

| Aspect | iptables | IPVS |
|--------|----------|------|
| Performance at scale | Degrades after ~1000 Services | Constant time |
| Rule update time | O(n), can spike | Fast, incremental |
| LB algorithms | Random only | rr, wrr, lc, wlc, sh, dh, sed, nq |
| Kernel module | iptables (mature) | ipvs, ip_vs_* (must be loaded) |
| Debugging | `iptables-save` shows everything | `ipvsadm -ln` for IPVS rules + some iptables |
| Default | Yes (historically) | No, must enable |
| Memory usage | Lower for small clusters | Slightly higher baseline |
| Connection tracking | Uses conntrack | Uses conntrack |

**When to Use Which:**

- **Small/medium clusters (<500 Services)**: iptables is simpler, well-tested, fine for performance
- **Large clusters (1000+ Services)**: IPVS is significantly better. AWS EKS, GKE recommendations often suggest IPVS for large clusters
- **Need advanced LB algorithms**: IPVS only
- **Going to eBPF**: Consider Cilium kube-proxy replacement instead of either

**Practical Considerations:**

To use IPVS mode, the kernel modules must be loaded: `ip_vs`, `ip_vs_rr`, `ip_vs_wrr`, `ip_vs_sh`, `nf_conntrack`. Configure kube-proxy with `--proxy-mode=ipvs` or set it in its ConfigMap.

Even in IPVS mode, you'll still see some iptables rules — for masquerading, filtering, NodePort handling, etc. This is normal.

---

### 6. How does CoreDNS work internally?

CoreDNS is the default DNS server for Kubernetes since v1.13, replacing kube-dns. It's a flexible, plugin-based DNS server written in Go that provides service discovery for the cluster.

**Deployment Model:**

CoreDNS runs as a Deployment in the `kube-system` namespace (typically 2 replicas for HA) with a Service called `kube-dns` (the name is retained for backward compatibility with kube-dns). This Service has a stable ClusterIP, usually `10.96.0.10` or `.10` of whatever Service CIDR is in use.

Every Pod's `/etc/resolv.conf` is configured to point to this Service IP as the nameserver — kubelet does this automatically when creating Pods (unless `dnsPolicy` is set otherwise).

**Configuration via Corefile:**

CoreDNS is configured through a **Corefile** stored in a ConfigMap (`coredns` in `kube-system`). A typical Corefile looks like:

```
.:53 {
    errors
    health {
       lameduck 5s
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    prometheus :9153
    forward . /etc/resolv.conf {
       max_concurrent 1000
    }
    cache 30
    loop
    reload
    loadbalance
}
```

Each line is a **plugin**, and they're processed in a chain.

**Key Plugins:**

- **kubernetes** — the most important. Watches the API server for Services, Endpoints, and Pods. Resolves DNS queries for cluster names by looking up these cached objects. Returns A/AAAA/SRV records for Services and Pods.
- **forward** — forwards queries it can't answer to upstream DNS servers (typically the node's `/etc/resolv.conf` for external resolution)
- **cache** — caches responses for a TTL to reduce load and latency
- **errors / log** — logging
- **health / ready** — health endpoints for kubelet probes
- **prometheus** — exposes metrics for monitoring
- **loop** — detects DNS loops
- **reload** — automatically reloads Corefile on changes
- **loadbalance** — randomizes A/AAAA record order for round-robin load balancing
- **rewrite** — rewrites queries (useful for custom domains)

**DNS Record Structure:**

For a Service `nginx` in namespace `default`, the records are:

- `nginx.default.svc.cluster.local` → ClusterIP (A record)
- `nginx.default` and `nginx` (short names) work because of search domains in resolv.conf

For headless Services, CoreDNS returns multiple A records (one per backing Pod):
- `nginx.default.svc.cluster.local` → IP1, IP2, IP3...
- Each Pod also gets `pod-hostname.nginx.default.svc.cluster.local`

For Pods (when `pods insecure` is set):
- `10-244-1-5.default.pod.cluster.local` (IP-based naming)

**SRV Records for StatefulSets:**

StatefulSets with headless Services get rich DNS records:
- `_http._tcp.nginx.default.svc.cluster.local` → SRV record listing all endpoints
- `nginx-0.nginx.default.svc.cluster.local` → stable per-Pod DNS

**Query Resolution Flow:**

1. Pod issues DNS query for `api.production.svc.cluster.local`
2. Query goes to CoreDNS via the cluster's DNS Service IP
3. CoreDNS receives query, checks its cache plugin first
4. If not cached, the kubernetes plugin matches the cluster.local suffix
5. It looks up the Service in its informer cache (populated by watching API server)
6. Constructs response with ClusterIP
7. Response goes back through plugins (cache stores it, loadbalance may shuffle, etc.)
8. Pod receives the IP

For external queries (`google.com`):
1. CoreDNS receives query
2. kubernetes plugin doesn't match (not cluster.local)
3. Falls through to forward plugin
4. forward sends to upstream DNS (node's resolv.conf nameservers)
5. Response cached and returned

**Pod DNS Configuration:**

By default, a Pod's resolv.conf looks like:
```
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

The `ndots:5` setting is important — any name with fewer than 5 dots triggers search domain expansion. This is why `nginx` resolves correctly inside the cluster (tries `nginx.default.svc.cluster.local` first via search).

**This is also a common performance issue** — external lookups can be slow because each search domain is tried first. Setting `ndots:1` or 2 helps but breaks short-name resolution. NodeLocal DNSCache (DaemonSet running CoreDNS on each node) helps mitigate this.

**Scaling Considerations:**

CoreDNS scales horizontally — more replicas handle more queries. It's also commonly deployed with **autoscaling** based on cluster size (cluster-proportional-autoscaler). For very large or DNS-heavy clusters, **NodeLocal DNSCache** is recommended — each node runs a local CoreDNS instance to cache queries, reducing latency and load on the central CoreDNS pods.

---

### 7. Explain admission controllers with real examples.

Admission controllers are **plugins that intercept API requests** after authentication and authorization, but before persistence to etcd. They can modify requests (mutating) or reject them (validating). They're the enforcement layer for cluster policies.

**Where They Fit in the Request Flow:**

```
Authentication → Authorization → Mutating Admission → Schema Validation → Validating Admission → etcd
```

Only **create, update, delete, and connect** requests go through admission. Read requests (get, list, watch) skip admission entirely.

**Built-in Admission Controllers:**

The API server ships with many compiled-in admission plugins, enabled via `--enable-admission-plugins`. Some of the most important:

**NamespaceLifecycle** prevents creation of objects in non-existent or terminating namespaces. Without this, you could create a Pod in a namespace being deleted, creating zombie resources.

**LimitRanger** enforces resource limits defined by LimitRange objects in a namespace. If a Pod doesn't specify resource requests/limits, LimitRanger fills in defaults. If a Pod requests more than the namespace allows, it's rejected.

**ResourceQuota** enforces namespace-level resource quotas. If creating a Pod would exceed the quota (CPU, memory, object counts), the request is rejected.

**ServiceAccount** auto-mounts service account tokens, sets the default service account if none specified, and ensures the service account exists.

**PodSecurity** (replaced PodSecurityPolicy in v1.25) enforces Pod Security Standards (privileged, baseline, restricted) based on namespace labels. For example, labeling a namespace with `pod-security.kubernetes.io/enforce=restricted` blocks privileged Pods.

**NodeRestriction** limits what each kubelet can do — can only modify its own Node object and Pods scheduled to its node. Critical security control.

**AlwaysPullImages** forces every Pod to pull images rather than use cached versions — useful in multi-tenant clusters to prevent unauthorized access to images other users have pulled.

**DefaultStorageClass** assigns the default StorageClass to PVCs that don't specify one.

**MutatingAdmissionWebhook and ValidatingAdmissionWebhook** delegate to external webhooks (covered in next question).

**Real-World Examples:**

*Example 1: Enforcing resource limits*

A LimitRange in the namespace:
```yaml
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
```

When a developer creates a Pod without specifying resources, LimitRanger mutates the spec to include these values. The developer doesn't have to remember; the admission controller enforces it.

*Example 2: Preventing latest tag*

You can deploy a ValidatingAdmissionWebhook (or use OPA Gatekeeper / Kyverno) that rejects any Pod whose container images use the `:latest` tag or no tag. This prevents non-reproducible deployments.

*Example 3: Auto-injecting sidecars*

Istio uses a MutatingAdmissionWebhook on the `sidecar.istio.io/inject=true` label. When a Pod is created in a labeled namespace, the webhook adds the Envoy sidecar container, init container, and volumes — transparent to the developer.

*Example 4: Enforcing image registry*

Many enterprises use OPA Gatekeeper or Kyverno to reject Pods whose images don't come from approved registries (e.g., only `registry.company.com/*` allowed). This prevents shadow IT and untrusted images.

*Example 5: Namespace must have specific labels*

A common policy: every namespace must have an `owner` label and a `cost-center` label. Without these, the namespace creation is rejected. This enables billing and accountability.

**Execution Order:**

Mutating admission controllers run **before** validating ones — this allows defaults to be set first, then validated. Within each category, controllers run in a defined order based on their configuration.

---

### 8. What is the role of mutating and validating webhooks?

Webhooks are the **extensibility mechanism** for admission control. While built-in admission controllers are compiled into the API server, webhooks let you write custom admission logic running in your own Pods.

**Architecture:**

You register a webhook by creating a **MutatingWebhookConfiguration** or **ValidatingWebhookConfiguration** object. This tells the API server: "When you see operations matching these rules (resource, verb, namespace, etc.), send an AdmissionReview to my webhook at this URL."

The API server sends HTTPS POSTs to your webhook with an `AdmissionReview` JSON payload containing the operation, the object being created/modified, the user, and other context. The webhook responds with another AdmissionReview indicating allow/reject and (for mutating) a JSON patch.

**Mutating Webhooks:**

Run first (before validating) and can **modify** the object being submitted.

The webhook returns a JSON patch like:
```json
{
  "op": "add",
  "path": "/spec/containers/0/env/-",
  "value": {"name": "INJECTED_VAR", "value": "hello"}
}
```

The API server applies this patch to the object before continuing.

**Common Use Cases:**

- **Sidecar injection** — Istio, Linkerd, Vault Agent injector add containers to Pods
- **Default value injection** — set default labels, annotations, resource limits, image pull policies, node selectors
- **Image tag rewriting** — rewrite `nginx:latest` to `nginx@sha256:...` for immutability
- **Secret injection** — Vault webhook injects secrets as files or environment variables
- **Pod scheduling hints** — auto-add nodeSelector or tolerations based on labels

**Validating Webhooks:**

Run **after** mutating webhooks and schema validation. They can only **accept or reject** — no modifications. If multiple validating webhooks are configured, they run in parallel; if any rejects, the request fails.

**Common Use Cases:**

- **Policy enforcement** — OPA Gatekeeper, Kyverno enforce custom policies (no `:latest`, required labels, allowed registries, security context constraints)
- **Compliance checks** — verify Pods meet security baselines
- **Reference validation** — ensure referenced ConfigMaps/Secrets/PVCs exist
- **Naming conventions** — enforce resource naming patterns
- **Quota enforcement beyond ResourceQuota** — custom resource-aware quotas

**Configuration Example:**

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: image-policy
webhooks:
- name: validate.images.company.com
  rules:
  - apiGroups: [""]
    apiVersions: ["v1"]
    operations: ["CREATE", "UPDATE"]
    resources: ["pods"]
  clientConfig:
    service:
      name: image-validator
      namespace: security
      path: /validate
    caBundle: <base64-ca-cert>
  admissionReviewVersions: ["v1"]
  sideEffects: None
  failurePolicy: Fail
  timeoutSeconds: 5
```

**Critical Configuration Options:**

**failurePolicy** — what to do if the webhook is unreachable or errors:
- `Fail` — reject the request (safer, ensures policies are always applied, but can break clusters if webhook is down)
- `Ignore` — proceed without the webhook (less safe but more resilient)

**timeoutSeconds** — how long to wait for the webhook (max 30s). If exceeded, failurePolicy kicks in.

**sideEffects** — declares whether the webhook has side effects. Mostly cosmetic, used during dry-run.

**namespaceSelector / objectSelector** — limit which requests trigger the webhook. Critical for performance and to avoid breaking system namespaces.

**reinvocationPolicy** (mutating only) — `Never` (default) or `IfNeeded` (re-invoke if a later mutating webhook modifies the object).

**Pitfalls and Best Practices:**

- **Don't put webhooks in the kube-system namespace's path** — if your webhook is in kube-system and fails, you can't restart it because new Pods need the webhook to succeed
- **Exclude critical namespaces** with namespaceSelector
- **Set short timeouts** — slow webhooks delay every API request
- **Make webhooks highly available** — multiple replicas, careful resource limits, PDBs
- **Use failurePolicy: Ignore for non-critical webhooks** to avoid bricking the cluster
- **Monitor webhook latency and error rates** — webhooks are on the critical path
- **Use cert-manager or similar** to rotate webhook serving certificates

**Frameworks:** Writing webhooks from scratch is tedious. Most teams use **OPA Gatekeeper**, **Kyverno**, or **Kubewarden** for policy-style webhooks, and **Operator SDK / controller-runtime** for custom logic.

---

### 9. Explain finalizers in Kubernetes.

Finalizers are **pre-deletion hooks** that allow controllers to perform cleanup before an object is actually removed from etcd. They're how Kubernetes handles graceful resource cleanup for external dependencies.

**The Problem They Solve:**

When you delete a PersistentVolume, the underlying cloud disk (EBS, GCE PD) needs to be detached, unmounted, and potentially deleted. If Kubernetes immediately removed the PV object, the controller would lose its reference and might leak cloud resources. Finalizers solve this by saying: "Don't actually delete this object until I've finished cleanup."

**How They Work:**

A finalizer is just a string in the `metadata.finalizers` array of an object. When you delete an object that has finalizers:

1. The API server sets the `deletionTimestamp` on the object
2. The object enters the "deleting" state — it's marked for deletion, but **not actually deleted**
3. Controllers see the deletionTimestamp and the finalizers they own, then perform cleanup
4. When done, each controller removes its finalizer from the list
5. Once the finalizers array is empty, the API server actually removes the object from etcd

**Example: PersistentVolume**

A PV typically has the finalizer `kubernetes.io/pv-protection`. When deleted:
1. deletionTimestamp set
2. PV controller waits until the PV is no longer bound to any PVC
3. Once unbound, the controller removes the finalizer
4. PV is deleted

**Example: Namespace**

When you delete a namespace, the `kubernetes` finalizer is added. The namespace controller iterates through every resource type in that namespace and deletes them. Only when all resources are gone does it remove the finalizer, allowing the namespace itself to be deleted.

This is why deleting a namespace can sometimes get **stuck** — if any resource has a stuck finalizer (e.g., a PVC waiting for an unreachable storage backend), the namespace will hang in Terminating forever. The classic debugging command:

```bash
kubectl get namespace stuck-namespace -o json | jq '.spec.finalizers = []' | kubectl replace --raw "/api/v1/namespaces/stuck-namespace/finalize" -f -
```

This force-removes finalizers. Use with caution — you might leak resources.

**Common Built-in Finalizers:**

- `kubernetes.io/pv-protection` — on PVs, prevents deletion while bound
- `kubernetes.io/pvc-protection` — on PVCs, prevents deletion while in use by Pods
- `foregroundDeletion` — for cascading deletion in foreground mode
- `orphan` — for orphan deletion mode (don't delete dependents)
- `kubernetes` — on Namespaces

**Custom Finalizers:**

Operators commonly add custom finalizers to their CRs. For example, a database operator managing PostgreSQL clusters might add `db.example.com/backup-finalizer`. When the cluster is deleted, the operator:
1. Takes a final backup
2. Deregisters from monitoring
3. Removes the finalizer
4. Lets the cluster object be deleted

This ensures cleanup actions can't be skipped.

**Patterns for Implementing Finalizers in Controllers:**

In a reconciler:
```go
if object.DeletionTimestamp.IsZero() {
    // Not being deleted — ensure finalizer is present
    if !containsString(object.Finalizers, myFinalizer) {
        object.Finalizers = append(object.Finalizers, myFinalizer)
        update(object)
    }
} else {
    // Being deleted — run cleanup
    if containsString(object.Finalizers, myFinalizer) {
        if err := cleanupExternalResources(object); err != nil {
            return err  // retry next reconcile
        }
        object.Finalizers = removeString(object.Finalizers, myFinalizer)
        update(object)
    }
}
```

**Stuck Finalizers — Troubleshooting:**

Objects stuck in Terminating usually mean a controller can't complete cleanup:
- The controller is down or crashing
- The controller can't reach an external dependency (cloud API, database)
- A bug in the controller's cleanup logic

Solutions:
1. Fix the underlying issue (restart controller, restore connectivity)
2. As a last resort, manually remove finalizers with `kubectl patch ... --type=merge -p '{"metadata":{"finalizers":[]}}'`

---

### 10. What are owner references and garbage collection?

Owner references and garbage collection are how Kubernetes manages **resource hierarchies** and handles **cascading deletion**. They enable patterns like "delete the Deployment, and all its ReplicaSets and Pods go away automatically."

**Owner References:**

Every object can have one or more entries in `metadata.ownerReferences`, each pointing to a parent object:

```yaml
metadata:
  name: nginx-7d8f9c-xyz12
  ownerReferences:
  - apiVersion: apps/v1
    kind: ReplicaSet
    name: nginx-7d8f9c
    uid: a1b2c3d4-...
    controller: true
    blockOwnerDeletion: true
```

Key fields:
- **uid** — the UID of the owner (not just name, to handle recreation cases)
- **controller** — true if this is the "managing" controller (only one can be true per object)
- **blockOwnerDeletion** — if true, the owner can't be deleted in foreground until this dependent is gone

**Hierarchy Example:**

```
Deployment (you create this)
    └── ReplicaSet (owned by Deployment, created by Deployment controller)
            └── Pod (owned by ReplicaSet, created by ReplicaSet controller)
                    └── Pod's hidden ownerless resources (e.g., its EndpointSlice entry)
```

When you delete the Deployment, garbage collection cascades down: ReplicaSet is deleted, which causes Pods to be deleted.

**The Garbage Collector:**

The Garbage Collector is a controller in kube-controller-manager that runs continuously. It maintains a graph of owner-dependent relationships across all objects in the cluster.

When an object is deleted, the GC:
1. Finds all objects whose ownerReferences point to the deleted object
2. Deletes those objects (based on propagation policy)
3. Recurses — those deletions trigger more cascading deletions

**Deletion Propagation Policies:**

When you delete an object, you can specify how cascading should happen:

- **Background (default)** — Kubernetes deletes the object immediately. GC then asynchronously deletes the dependents. Fast but the dependents may briefly outlive the owner.

- **Foreground** — The object stays in "deleting" state (with `foregroundDeletion` finalizer) until all dependents with `blockOwnerDeletion=true` are deleted first. This gives a strict bottom-up deletion order. Used when ordering matters.

- **Orphan** — Dependents are not deleted; their ownerReferences are just removed. They become orphaned and continue to run. Useful when you want to replace a controller but keep its Pods.

Specifying via kubectl:
```bash
kubectl delete deployment nginx --cascade=foreground
kubectl delete deployment nginx --cascade=orphan
kubectl delete deployment nginx --cascade=background  # default
```

**Why This Matters:**

1. **No manual cleanup** — delete one object, and all derived objects go too
2. **No orphaned resources** — by default, nothing is left behind
3. **Hierarchical management** — you only manage top-level resources; controllers handle the rest

**Built-in Hierarchies:**

- Deployment owns ReplicaSets (one per revision)
- ReplicaSet owns Pods
- StatefulSet owns Pods and PersistentVolumeClaims (if `whenDeleted: Delete` on PVC retention policy)
- DaemonSet owns Pods
- Job owns Pods
- CronJob owns Jobs
- Service owns EndpointSlices

**Custom Resources and Owner References:**

When building operators, you should set owner references on resources your operator creates. This way, when the CR is deleted, the dependent resources are automatically cleaned up.

Example: A `WordPressSite` CR owns a Deployment, Service, PVC. Set ownerReferences on each of these pointing to the WordPressSite. Deleting the WordPressSite deletes everything.

The controller-runtime library provides `controllerutil.SetControllerReference()` to do this easily.

**Cross-Namespace Limitations:**

Owner references **cannot cross namespaces**. A namespaced object can only have an owner in the same namespace (or a cluster-scoped owner). This is a security and consistency constraint.

**Pitfalls:**

- **Cluster-scoped objects can't be owned by namespaced objects** — only the reverse works
- **Wrong UID = no link** — if you set ownerReferences with just a name, it doesn't work. UIDs are required
- **Orphaned resources from manual changes** — if you remove ownerReferences manually, the object survives parent deletion
- **Custom controllers must respect ownerReferences** — when building operators, set them correctly or you'll leak resources

**Inspecting the GC:**

```bash
# See what owns a Pod
kubectl get pod nginx-xxx -o jsonpath='{.metadata.ownerReferences}'

# See what a Deployment owns (no direct command, but you can list ReplicaSets/Pods with matching labels)
kubectl get replicasets,pods -l app=nginx

# Check for stuck deletions (objects with deletionTimestamp set)
kubectl get all -A -o json | jq '.items[] | select(.metadata.deletionTimestamp != null) | .metadata.name'
```

---
