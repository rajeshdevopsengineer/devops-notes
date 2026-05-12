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

Would you like me to continue with the next set of questions, or dive deeper into any specific topic above?