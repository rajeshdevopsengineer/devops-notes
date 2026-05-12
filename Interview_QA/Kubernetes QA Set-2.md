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

# Advanced Kubernetes Interview Questions - Part 3

## Scheduling, Workload Management & Resource Control

---

### 21. Explain taints, tolerations, and node affinity.

These are Kubernetes' primary mechanisms for **influencing Pod-to-Node placement**. They work together but serve different purposes — taints repel, tolerations accept, and affinity attracts.

**Taints (Applied to Nodes)**

A taint is a key-value pair with an effect attached to a node. It tells the scheduler: "Don't put Pods here unless they explicitly tolerate this taint." Taints are **repulsion** mechanisms.

```bash
kubectl taint nodes node1 key=value:NoSchedule
```

**Three Taint Effects:**

- **NoSchedule** — the scheduler won't place new Pods on this node unless they tolerate the taint. Existing Pods are unaffected.
- **PreferNoSchedule** — soft version; scheduler tries to avoid placing Pods here but will if necessary. No guarantees.
- **NoExecute** — the strictest. New Pods won't be scheduled, and existing Pods that don't tolerate the taint are **evicted immediately** (or after `tolerationSeconds`).

**Tolerations (Applied to Pods)**

A toleration allows a Pod to be scheduled onto nodes with matching taints. It doesn't force placement — it just removes the repulsion.

```yaml
tolerations:
- key: "dedicated"
  operator: "Equal"
  value: "gpu"
  effect: "NoSchedule"
```

The `operator` can be `Equal` (key, value, and effect must all match) or `Exists` (just the key needs to exist, regardless of value).

**Real-World Use Cases for Taints:**

- **Dedicated nodes** — taint GPU nodes with `dedicated=gpu:NoSchedule` so only GPU workloads (with matching tolerations) land there
- **Master/Control plane nodes** — historically tainted `node-role.kubernetes.io/control-plane:NoSchedule` to prevent regular workloads
- **Specialized hardware** — high-memory nodes, SSD-backed nodes, ARM nodes
- **Maintenance** — `kubectl drain` adds `node.kubernetes.io/unschedulable:NoSchedule` automatically
- **Failure conditions** — Node Controller auto-adds taints like `node.kubernetes.io/not-ready:NoExecute`, `node.kubernetes.io/unreachable:NoExecute`, `node.kubernetes.io/memory-pressure:NoSchedule`, `node.kubernetes.io/disk-pressure:NoSchedule`

**Node Affinity (Applied to Pods)**

Node affinity is the **attraction** mechanism — Pods declare preferences for the nodes they want. It's expressed in the Pod spec and matches node labels.

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: disktype
          operator: In
          values: ["ssd"]
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 80
      preference:
        matchExpressions:
        - key: zone
          operator: In
          values: ["us-east-1a"]
```

Node affinity replaces the older `nodeSelector` (which is just a simple key=value match) with richer expressions: `In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`.

**Key Differences from Taints/Tolerations:**

| Feature | Taints/Tolerations | Node Affinity |
|---------|-------------------|---------------|
| Defined on | Nodes (taint) + Pods (toleration) | Pods only |
| Direction | Repel (node says "go away") | Attract (Pod says "I want") |
| Purpose | Reserve nodes | Choose nodes |
| Without rule | Can be scheduled anywhere | Can be scheduled anywhere |

**Combined Pattern: Truly Dedicated Nodes**

To dedicate nodes for specific workloads, use both:

1. **Taint the node** so general workloads avoid it: `dedicated=gpu:NoSchedule`
2. **Tolerate the taint** on GPU workloads so they're allowed
3. **Add node affinity** on GPU workloads so they're attracted

Without taints, GPU workloads might run elsewhere. Without tolerations, GPU workloads can't run on GPU nodes. Without node affinity, you depend on the scheduler not picking other nodes (which it might).

**Pod Affinity / Anti-Affinity (Bonus):**

Beyond node affinity, there's also Pod affinity (schedule with these other Pods) and Pod anti-affinity (avoid these other Pods). These work on Pod labels rather than node labels.

A common pattern: anti-affinity to spread replicas across nodes or zones for high availability.

```yaml
podAntiAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
  - labelSelector:
      matchExpressions:
      - key: app
        operator: In
        values: ["nginx"]
    topologyKey: kubernetes.io/hostname
```

This ensures no two nginx Pods land on the same node.

---

### 22. Difference between hard and soft affinity rules.

Affinity rules come in two strengths — **required** (hard) and **preferred** (soft) — with the naming reflecting whether the scheduler treats them as constraints or preferences.

**Hard Affinity (Required)**

`requiredDuringSchedulingIgnoredDuringExecution` — the scheduler **must** find a node satisfying these rules. If no such node exists, the Pod stays in Pending state indefinitely (until matching nodes become available).

Key characteristics:
- Acts as a **filter** during the scheduling filtering phase
- No node passes filtering → Pod is unscheduled
- Functions like a strict requirement, not a preference
- Used when you absolutely need certain characteristics (e.g., GPU workloads need GPU nodes)

**Soft Affinity (Preferred)**

`preferredDuringSchedulingIgnoredDuringExecution` — the scheduler **tries** to satisfy these rules but will schedule the Pod elsewhere if no preferred node is available.

Key characteristics:
- Acts as a **scoring** function during the scheduler's priority phase
- Each rule has a `weight` (1-100) that contributes to node scores
- Pod gets scheduled even if no preferred match exists — just on lower-scoring nodes
- Used for optimization rather than constraint

**Naming Decoded:**

The full name `requiredDuringSchedulingIgnoredDuringExecution` reveals two dimensions:

- **DuringScheduling**: Required vs Preferred — affects placement decisions
- **DuringExecution**: Ignored vs (planned: Required) — what happens if node labels change after scheduling

Currently, only `IgnoredDuringExecution` is implemented for affinity. This means: once a Pod is scheduled, changes to node labels don't cause re-evaluation. The Pod stays put even if the node no longer matches.

The planned `RequiredDuringExecution` would evict Pods when their nodes no longer match — but it hasn't been implemented yet.

**Weight in Soft Affinity:**

Soft affinity rules each have a weight (1-100). The scheduler sums weights of matching rules per node, contributing to that node's overall score. Higher weights = stronger preference.

```yaml
preferredDuringSchedulingIgnoredDuringExecution:
- weight: 80
  preference:
    matchExpressions:
    - key: zone
      operator: In
      values: ["us-east-1a"]
- weight: 20
  preference:
    matchExpressions:
    - key: instance-type
      operator: In
      values: ["r5.large"]
```

Here, being in zone us-east-1a contributes 80 points, while the instance type contributes 20. A node matching both gets 100 added; only the first gets 80; only the second gets 20.

**When to Use Each:**

**Use Hard (required) when:**
- Workload literally cannot function without specific hardware (GPUs, specific CPU architectures)
- Compliance/data-residency requirements (workload must run in EU region)
- Security isolation (sensitive workloads must run on dedicated nodes)
- Storage locality requirements (must be in same zone as the PV)

**Use Soft (preferred) when:**
- Performance optimization (prefer faster nodes but okay if unavailable)
- Topology spreading (prefer to distribute across zones for HA)
- Cost optimization (prefer spot instances)
- Co-location preferences (prefer to be near related services for latency)

**Common Pitfall:**

Over-using hard affinity creates **unschedulable Pods**. If you require 100GB of memory AND a specific zone AND a specific instance type AND a specific OS version, you might end up with no matching nodes. Pods sit in Pending forever, and the cluster autoscaler might struggle to scale up.

Best practice: use hard affinity for true requirements, soft affinity for preferences. Reserve hard rules for things that would actually break the workload if violated.

**Pod Affinity/Anti-Affinity Considerations:**

Pod affinity rules can be expensive to evaluate, especially when using `topologyKey: kubernetes.io/hostname` and large numbers of Pods. The scheduler must compare every Pod's labels against the affinity rules for every candidate node. Soft Pod anti-affinity is generally fine, but hard Pod anti-affinity at scale can slow scheduling significantly. This is why topology spread constraints (next question) were introduced as a more efficient alternative.

---

### 23. Explain topology spread constraints.

Topology spread constraints are Kubernetes' modern, scalable way to **distribute Pods evenly across failure domains** — zones, nodes, racks, regions. They were introduced in v1.16 (GA in v1.19) as a more efficient alternative to Pod anti-affinity for spreading workloads.

**The Problem They Solve:**

Imagine running 6 replicas of a service across 3 availability zones. You want 2 replicas per zone for HA. With Pod anti-affinity, expressing this is awkward and the scheduling becomes O(n²). Topology spread constraints provide a declarative, efficient mechanism.

**Anatomy of a Constraint:**

```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: nginx
  matchLabelKeys:    # newer feature
  - pod-template-hash
  minDomains: 3      # newer feature
  nodeAffinityPolicy: Honor
  nodeTaintsPolicy: Honor
```

**Key Fields Explained:**

**maxSkew** — the maximum allowed difference between the most populated and least populated topology domains. If maxSkew=1, the difference between zones can never exceed 1 Pod. So with 6 replicas across 3 zones: 2-2-2 is fine (skew 0), 3-2-1 is fine (skew 2 — wait, that's actually 3-1=2, which exceeds maxSkew=1, so this would be rejected).

**topologyKey** — the node label that defines the topology domain. Common values:
- `kubernetes.io/hostname` — spread across nodes
- `topology.kubernetes.io/zone` — spread across AZs
- `topology.kubernetes.io/region` — spread across regions
- Custom labels like `rack` or `datacenter`

**whenUnsatisfiable** — what to do if the constraint can't be satisfied:
- `DoNotSchedule` (hard) — Pod stays Pending
- `ScheduleAnyway` (soft) — schedule on the least-loaded domain even if it violates maxSkew

**labelSelector** — identifies which Pods are counted when computing skew. Usually matches the Deployment/StatefulSet's labels.

**How Scheduling Works:**

When scheduling a new Pod, the scheduler:
1. Counts existing matching Pods in each topology domain (based on topologyKey)
2. For each candidate node, simulates adding the new Pod
3. Calculates the new skew (max - min count across domains)
4. If skew exceeds maxSkew, the node is rejected (DoNotSchedule) or penalized (ScheduleAnyway)
5. Among valid nodes, normal scoring continues

**Real-World Example: Multi-AZ Deployment**

A web service with 9 replicas across 3 zones:

```yaml
spec:
  replicas: 9
  template:
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: web
```

This guarantees 3-3-3 distribution. If one zone fails, you lose 3 Pods but 6 remain. Without this, you might end up with 5-2-2 or even worse distributions.

**Combining Multiple Constraints:**

You can have multiple constraints — each acts as an AND condition.

```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: web
- maxSkew: 2
  topologyKey: kubernetes.io/hostname
  whenUnsatisfiable: ScheduleAnyway
  labelSelector:
    matchLabels:
      app: web
```

This enforces strict zone balance AND prefers to spread across nodes (but allows clustering on nodes if necessary).

**Default Cluster-Wide Constraints:**

You can configure cluster-wide defaults in the scheduler's KubeSchedulerConfiguration, applied to all Pods that don't specify their own constraints. Kubernetes ships with built-in defaults that lightly spread Pods across nodes and zones.

**Advanced Fields:**

**minDomains** (v1.27 GA) — ensures the constraint considers at least N domains. If you have minDomains=3 but only 2 zones currently exist, the constraint treats the missing zones as having 0 Pods, forcing better spreading. Useful when zones might temporarily disappear (autoscaling).

**matchLabelKeys** (v1.27 GA) — automatically distinguish Pods of different revisions. By including `pod-template-hash`, the constraint only counts Pods of the same rollout, allowing rolling updates to proceed without violating spread.

**nodeAffinityPolicy** and **nodeTaintsPolicy** — control whether nodes excluded by affinity/taints are still counted as topology domains.

**Comparison to Pod Anti-Affinity:**

| Feature | Topology Spread Constraints | Pod Anti-Affinity |
|---------|----------------------------|---------------------|
| Granularity | maxSkew controls distribution | Yes/no co-location |
| Performance | O(n) | O(n²) for required |
| Use Case | Even distribution | Strict separation |
| Multiple domains | Yes, native | Awkward |
| Default support | Cluster-wide defaults possible | No |

Generally, **prefer topology spread constraints for HA distribution**. Use Pod anti-affinity only when you need strict "no two Pods on same node" (e.g., HostPort conflicts).

---

### 24. How does Kubernetes handle node failures?

Node failure handling involves multiple components working together to detect failure, mark the node, evict workloads, and ensure replacements are scheduled. The process is intentionally slow to avoid spurious actions during transient issues.

**Detection Phase:**

The kubelet on each node sends regular updates to the API server via two mechanisms:

1. **Node Lease** (every 10 seconds by default) — lightweight heartbeat in `kube-node-lease` namespace
2. **Node Status updates** — full status PATCH every few minutes or on state changes

The **Node Controller** in kube-controller-manager watches these. If the Node Lease isn't updated within `--node-monitor-grace-period` (default 40 seconds), the controller:
- Marks the Node's Ready condition as `Unknown`
- Updates lastHeartbeatTime to indicate when contact was lost

**Tainting Phase (Automatic):**

Once the node is unhealthy, the Node Controller adds taints:
- `node.kubernetes.io/not-ready:NoExecute` (Ready condition False)
- `node.kubernetes.io/unreachable:NoExecute` (Ready condition Unknown)

These NoExecute taints would normally evict Pods immediately, except Pods have default tolerations with `tolerationSeconds: 300` (5 minutes) for these specific taints. This grace period prevents premature eviction during brief network blips.

**Eviction Phase:**

After 5 minutes of node unreachability, the TaintBasedEvictionController begins deleting Pods on the failed node. But here's a critical nuance: **deletion from the API server doesn't actually stop the running containers**. The kubelet is dead and can't process the deletion. The Pod objects are removed from etcd, freeing controllers (ReplicaSet, StatefulSet) to create replacement Pods elsewhere.

**For Stateless Workloads (Deployments):**

ReplicaSet controller observes that desired replicas (e.g., 3) exceeds actual (now 2). It creates a new Pod, scheduler places it on a healthy node, kubelet starts it. Total downtime for the affected replica: ~5-6 minutes by default.

**For Stateful Workloads (StatefulSets):**

This is where things get tricky. StatefulSet controller won't create a replacement Pod with the same ordinal (mysql-0) until the old one is **confirmed gone**. Since the old Pod might still be running on the unreachable node (with its persistent volume still mounted), creating a replacement could cause:
- Two Pods with identical identity
- Concurrent access to the same persistent volume
- Data corruption

So by default, StatefulSet Pods on unreachable nodes stay in **Terminating state forever**. The cluster operator must manually verify the old Pod is truly gone and force-delete it:

```bash
kubectl delete pod mysql-0 --force --grace-period=0
```

This is intentional — better extended downtime than data corruption. For automated handling, you need **fencing** at the storage layer (e.g., I/O fencing via STONITH-like mechanisms) or use storage that handles split-brain (distributed databases with proper consensus protocols).

**Rate Limiting Evictions:**

The Node Controller rate-limits evictions to avoid cascading failures. Key parameters:

- `--node-eviction-rate` (default 0.1 nodes/second) — how fast to evict from failed nodes
- `--secondary-node-eviction-rate` (default 0.01) — slower rate when many nodes are unhealthy
- `--unhealthy-zone-threshold` (default 55%) — if more than 55% of nodes in a zone are unhealthy, that zone is marked unhealthy and uses secondary eviction rate
- `--large-cluster-size-threshold` (default 50) — different logic for small vs large clusters

This prevents scenarios like "AZ outage takes down half the cluster, controllers evict everything simultaneously, healthy nodes get overwhelmed by rescheduling."

**Pod Disruption Budgets (PDBs):**

PDBs protect workloads during **voluntary** disruptions (drain, eviction API). They don't apply to involuntary disruptions like node failure — that's a common misconception. If a node dies, PDBs don't stop replacement Pod creation.

**Node Auto-Recovery:**

If a failed node comes back online:
- kubelet re-establishes contact with the API server
- Node Lease is refreshed, taints removed
- Pods running on the node are reconciled against API state
- Pods deleted during the outage are cleaned up by kubelet
- The node is again schedulable

**Cluster Autoscaler Integration:**

In cloud environments, the cluster autoscaler can detect unhealthy nodes and replace them with new instances. AWS, GCP, Azure cluster autoscalers all support this. Karpenter (AWS) is more aggressive — it can rapidly cycle out unhealthy nodes.

**Tuning for Faster Detection:**

For latency-sensitive workloads, you can tune:
- Reduce `--node-monitor-grace-period` (40s → 10s)
- Reduce default toleration seconds via `default-not-ready-toleration-seconds` and `default-unreachable-toleration-seconds` API server flags
- Use shorter heartbeat intervals

Trade-off: aggressive timing causes more false positives during network blips. The 5-minute default is conservative because Kubernetes assumes flaky networks are temporary.

---

### 25. Explain Pod eviction mechanisms.

Pod eviction is the process of **terminating Pods to free resources or accommodate cluster operations**. There are several distinct eviction mechanisms, each triggered by different conditions and following different rules.

**1. API-Initiated Eviction**

This is the "polite" eviction triggered via the Eviction API (used by `kubectl drain`).

How it works:
- A request is sent to `/api/v1/namespaces/<ns>/pods/<name>/eviction`
- The API server checks **Pod Disruption Budgets (PDBs)**
- If eviction would violate a PDB (drop available replicas below minAvailable), the request is rejected
- Otherwise, the Pod gets a normal deletion with grace period

This is the only eviction type that respects PDBs. It's used by:
- `kubectl drain` for planned node maintenance
- Cluster Autoscaler when scaling down nodes
- Karpenter when consolidating workloads

**2. kubelet Node-Pressure Eviction**

The kubelet monitors node resource pressure and proactively evicts Pods to keep the node healthy.

**Monitored Signals:**
- `memory.available` — free memory on the node
- `nodefs.available` — free space on node filesystem (rootfs)
- `nodefs.inodesFree` — free inodes on node filesystem
- `imagefs.available` — free space on filesystem holding container images
- `imagefs.inodesFree` — free inodes on imagefs
- `pid.available` — free PIDs

**Threshold Types:**

- **Soft thresholds** — trigger eviction after a grace period (`--eviction-soft-grace-period`)
- **Hard thresholds** — trigger immediate eviction (`--eviction-hard`)

Example kubelet config:
```yaml
evictionHard:
  memory.available: "100Mi"
  nodefs.available: "10%"
  nodefs.inodesFree: "5%"
evictionSoft:
  memory.available: "300Mi"
evictionSoftGracePeriod:
  memory.available: "1m30s"
```

**Eviction Order — Pod Selection:**

When kubelet needs to evict, it sorts Pods by:
1. **QoS class** — BestEffort first, then Burstable, then Guaranteed
2. **Priority** — lower PriorityClass evicted first
3. **Resource usage** — Pods exceeding their requests are prioritized for eviction (using more than they asked for)

**QoS Classes:**

- **Guaranteed** — every container has equal requests and limits for memory and CPU; evicted last
- **Burstable** — at least one container has requests but they don't equal limits; middle priority
- **BestEffort** — no requests or limits set; evicted first

This is why **setting proper requests is so important** — Guaranteed Pods survive node pressure scenarios.

**3. Node OOM Killer (Linux-Level)**

Independent of kubelet, the Linux kernel's OOM killer can kill processes when memory is critically low. This bypasses Kubernetes entirely.

OOM scores are adjusted based on QoS:
- BestEffort containers get the highest OOM score (most likely to be killed)
- Guaranteed containers get adjusted low scores (least likely)

If kubelet's eviction doesn't act fast enough, OOM killer takes over. The killed container shows reason `OOMKilled` in its state.

**4. Preemption-Based Eviction**

When a high-priority Pod can't be scheduled, the scheduler can preempt (evict) lower-priority Pods to make room. This is **scheduler-driven, not kubelet-driven**.

Process:
1. Scheduler can't place a Pod with PriorityClass A (priority 1000)
2. It looks for nodes where evicting some lower-priority Pods would enable scheduling
3. It deletes those lower-priority Pods (respecting PDBs only as a courtesy, not strictly)
4. After grace period, the high-priority Pod is scheduled

**5. Taint-Based Eviction**

NoExecute taints cause eviction of non-tolerating Pods. Either added manually:
```bash
kubectl taint nodes node1 maintenance=true:NoExecute
```

Or automatically by the Node Controller for unhealthy nodes.

Pods with `tolerationSeconds` get a grace period; Pods without tolerations are evicted immediately.

**Eviction vs Deletion:**

- **Deletion** — standard graceful termination via DELETE API; respects terminationGracePeriodSeconds
- **Eviction** — usually goes through deletion but may bypass for emergency cases

**Pod Disruption Budgets (PDBs):**

PDBs only apply to **voluntary** disruptions (API-initiated evictions). For node-pressure or node-failure scenarios, PDBs are ignored.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 2  # or maxUnavailable: 1
  selector:
    matchLabels:
      app: web
```

This protects against drains that would leave fewer than 2 replicas available. But during a node crash, the PDB doesn't prevent the loss.

**Monitoring Evictions:**

```bash
# Check why a Pod was evicted
kubectl describe pod evicted-pod
# Look at Status.Reason: "Evicted"
# Status.Message often explains the trigger

# View node conditions
kubectl describe node <node-name>
# Look for MemoryPressure, DiskPressure, PIDPressure

# Events at namespace level
kubectl get events --field-selector reason=Evicted
```

**Best Practices:**

- **Always set resource requests** to get out of BestEffort QoS
- **Set requests = limits** for critical workloads (Guaranteed QoS)
- **Configure PDBs** for production workloads
- **Set priorityClass** for critical pods to avoid preemption
- **Monitor for OOMKilled events** — they indicate memory limits are too low or memory leaks exist

---

### 26. What happens during a rolling update internally?

A rolling update is the default Deployment strategy that **gradually replaces old Pods with new ones**, maintaining availability throughout. The internal mechanics involve multiple controllers, ReplicaSets, and careful sequencing.

**The Setup:**

You have a Deployment with 10 replicas of nginx:1.20. You update it to nginx:1.21. Internally, this triggers a sophisticated dance.

**Step 1: Deployment Update**

You apply the new spec via `kubectl apply` or `kubectl set image`. The Deployment controller in kube-controller-manager observes the change. It computes a hash of the new PodTemplate (pod-template-hash) — if this hash matches an existing ReplicaSet, that one is reused; otherwise, a new ReplicaSet is needed.

**Step 2: New ReplicaSet Creation**

The Deployment controller creates a new ReplicaSet (say `nginx-7d8f9c`) with:
- Replicas: 0 initially
- PodTemplate matching the new spec
- Labels including `pod-template-hash: 7d8f9c`
- OwnerReferences pointing to the Deployment

The old ReplicaSet (say `nginx-6b2c1a`) still has 10 replicas.

**Step 3: Rolling Update Begins**

The Deployment controller now choreographs the rolling update based on:
- **maxSurge** (default 25%) — how many extra Pods can exist above replicas during the rollout. With 10 replicas, maxSurge=25% allows 12 total.
- **maxUnavailable** (default 25%) — how many Pods can be unavailable during rollout. With 10 replicas, maxUnavailable=25% allows 7 minimum available.

Iteration loop:

1. **Scale up new ReplicaSet** — if total Pods < replicas + maxSurge, scale up new RS by some amount. So nginx-7d8f9c goes from 0 → 2 (now we have 12 Pods: 10 old + 2 new).

2. **Wait for new Pods to become Ready** — the controller watches Pod conditions. Once readiness probes pass, the Pods are Ready.

3. **Scale down old ReplicaSet** — once we have enough new Ready Pods, scale down old RS. Old RS goes from 10 → 8. Total Pods: 10 (8 old + 2 new).

4. **Repeat** — scale new up, wait for ready, scale old down, until old RS is at 0 and new RS is at 10.

The exact sequence varies based on maxSurge and maxUnavailable. With maxSurge=25%, maxUnavailable=25%, replicas=10:
- Can add up to 2 Pods (replicas + 25%)
- Must keep at least 8 Ready (replicas - 25%)

So the rollout proceeds in batches: scale up 2 new, wait for ready, scale down 2 old, scale up 2 more new, etc.

**Step 4: Pod-Level Events**

For each new Pod:
1. ReplicaSet controller creates Pod object via API
2. Scheduler places it on a node
3. Kubelet pulls image (if not cached)
4. Kubelet creates containers via CRI
5. Container starts, readiness probe begins
6. Once ready, Pod is added to Service endpoints
7. EndpointSlice is updated, kube-proxy reconfigures routing

For each old Pod being terminated:
1. ReplicaSet controller deletes Pod object
2. Endpoint controller removes Pod from EndpointSlices
3. kube-proxy stops routing new connections to this Pod
4. Existing connections continue (but no new ones)
5. PreStop hook runs (if defined)
6. SIGTERM sent to container
7. Grace period (default 30s)
8. SIGKILL if still running
9. Pod object removed from API

**Step 5: Completion**

When new RS reaches replicas (10) and old RS is at 0, the rollout completes. The old ReplicaSet object remains for rollback purposes (controlled by `revisionHistoryLimit`, default 10).

**Tracking and Status:**

The Deployment status tracks:
- `replicas` — total Pods (across all RSs)
- `updatedReplicas` — Pods of the new RS
- `readyReplicas` — Pods that are Ready
- `availableReplicas` — Ready Pods that have been Ready for `minReadySeconds`
- `unavailableReplicas` — desired - available

You can watch progress: `kubectl rollout status deployment/nginx`

**Progress Deadlines:**

`progressDeadlineSeconds` (default 600s = 10 min) — if the rollout doesn't make progress in this time, the Deployment is marked failed. "Progress" means at least one new Pod becoming Ready in that window.

Failure doesn't auto-rollback — that requires `kubectl rollout undo`. But it does set the Progressing condition to False with reason ProgressDeadlineExceeded.

**Rollback Mechanism:**

```bash
kubectl rollout undo deployment/nginx
# or specific revision
kubectl rollout undo deployment/nginx --to-revision=3
```

This swaps which ReplicaSet is "current" — scaling the previous RS back up and the new one back down, following the same rolling logic in reverse.

**Edge Cases:**

- **Failed image pull** — new Pods stuck in ImagePullBackOff. Old Pods remain serving traffic. Rollout doesn't progress.
- **Failing readiness probes** — new Pods never become Ready. Rollout stalls.
- **Resource limits exceeded** — new Pods may be Pending due to insufficient cluster capacity.
- **PDBs preventing scaling down** — if scaling down old RS would violate PDB, eviction blocks.

**Pause and Resume:**

```bash
kubectl rollout pause deployment/nginx
# Make multiple changes without triggering rollouts
kubectl set image deployment/nginx nginx=nginx:1.22
kubectl set env deployment/nginx KEY=value
kubectl rollout resume deployment/nginx
```

When paused, the Deployment controller still reconciles existing state but doesn't initiate new rollouts. Useful for batching changes.

---

### 27. Difference between recreate and rolling deployment strategy.

Deployments support two strategies, configured under `spec.strategy.type`. Each has different trade-offs around availability, resource usage, and complexity.

**Rolling Update (Default)**

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
```

Already covered in detail above. Key characteristics:
- Old and new versions coexist during the update
- No downtime if properly configured with health probes
- Gradual rollout with controlled rate
- Can be paused, monitored, and rolled back smoothly
- Resource overhead during rollout (maxSurge adds Pods)

**Recreate**

```yaml
spec:
  strategy:
    type: Recreate
```

The Recreate strategy is straightforward and brutal: kill all old Pods first, then create new ones.

Process:
1. Deployment controller scales old ReplicaSet to 0
2. Waits for all old Pods to terminate completely
3. Scales new ReplicaSet to the desired replica count
4. New Pods come up over time as scheduler places them and they start

Key characteristics:
- **Downtime** between old Pods terminating and new Pods becoming Ready
- No version coexistence — at any point, all running Pods are the same version
- No extra resource usage during rollout (no surge)
- Simpler — no complex sequencing

**Detailed Comparison:**

| Aspect | RollingUpdate | Recreate |
|--------|---------------|----------|
| Downtime | Zero (with proper health checks) | Yes — during transition |
| Version coexistence | Old and new run simultaneously | Never |
| Resource overhead | maxSurge adds Pods temporarily | None |
| Speed of rollout | Slower, controlled | Faster total time |
| Rollback complexity | Same rolling logic in reverse | Same recreate logic |
| Database migrations | Risky if schema changes | Safer — clean version cutover |
| Singleton apps | Inappropriate (two instances) | Appropriate |

**When to Use Recreate:**

1. **Application can't run two versions simultaneously** — e.g., a singleton that holds a global lock, or two versions can't share the same database schema safely.

2. **Database schema migrations** — if v2 requires schema changes incompatible with v1, running both simultaneously would corrupt data. Recreate ensures clean cutover after migration runs.

3. **License limitations** — software licensed per running instance, can't temporarily exceed count.

4. **Resource-constrained clusters** — no headroom for maxSurge Pods.

5. **PVC ReadWriteOnce constraints** — if a Pod uses an RWO PersistentVolume, the new Pod can't mount it until the old Pod releases it. Rolling update would deadlock; Recreate works.

6. **Stateful workloads with side effects** — applications that, when starting, do incompatible initialization.

**When to Use RollingUpdate:**

- Any stateless workload — web servers, APIs, microservices
- Applications designed to be horizontally scaled
- Need for zero-downtime deployments
- Workloads with backward-compatible changes
- Most production scenarios

**Tuning RollingUpdate:**

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1          # absolute number or percentage
    maxUnavailable: 0    # ensure full capacity during rollout
```

Setting `maxUnavailable: 0` and `maxSurge: 1` gives the safest rollout — always full capacity, add one new before removing one old. Slower but most conservative.

For faster rollouts:
```yaml
maxSurge: 50%
maxUnavailable: 25%
```

For aggressive rollouts (with sufficient capacity):
```yaml
maxSurge: 100%   # double the Pods temporarily
maxUnavailable: 0
```

This creates a parallel deployment, swaps traffic, then kills old. Similar to blue-green in effect.

**Beyond Built-in Strategies:**

For more advanced patterns, Kubernetes-native tools fill the gap:
- **Argo Rollouts** — supports canary, blue-green, automated analysis
- **Flagger** — canary deployments with metric-based progression
- **Service Mesh (Istio, Linkerd)** — traffic splitting for canary/blue-green

These are NOT built into Deployments — you need extra tooling. Deployments only support RollingUpdate and Recreate natively.

---

### 28. Explain ReplicaSet internals.

ReplicaSets are the **next-generation Replication Controllers** that ensure a specified number of Pod replicas are running at any time. They're rarely created directly — usually a Deployment manages them — but understanding their internals is essential.

**Core Responsibility:**

A ReplicaSet maintains a stable set of replica Pods running at any given time. It's a **reconciliation loop**: compare desired replicas to actual matching Pods, create or delete to converge.

**Anatomy:**

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-7d8f9c
  labels:
    app: nginx
    pod-template-hash: 7d8f9c
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
      pod-template-hash: 7d8f9c
  template:
    metadata:
      labels:
        app: nginx
        pod-template-hash: 7d8f9c
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
status:
  replicas: 3
  fullyLabeledReplicas: 3
  readyReplicas: 3
  availableReplicas: 3
  observedGeneration: 1
```

**The ReplicaSet Controller:**

A goroutine in kube-controller-manager. Its reconciliation logic:

1. **Watch Pods and ReplicaSets** via informers
2. **For each ReplicaSet**, list Pods matching its selector
3. **Filter to "active" Pods** (not being deleted, not failed permanently)
4. **Compare counts**:
   - actual < desired → create (desired - actual) Pods
   - actual > desired → delete (actual - desired) Pods
   - actual == desired → no-op

**Pod Creation:**

When the controller needs to create Pods:
1. Generates a unique name (ReplicaSet name + random suffix)
2. Applies the PodTemplate from `spec.template`
3. Adds ownerReferences pointing to the ReplicaSet (with `controller: true`)
4. Creates Pod via API server

The controller creates multiple Pods in parallel (with concurrency limits) but uses a slow-start mechanism for large scale-ups to avoid overwhelming the cluster.

**Pod Selection (Adoption and Orphaning):**

The selector is the key relationship. The ReplicaSet uses its `spec.selector` to identify "its" Pods. Important subtleties:

- **Adoption** — if a Pod's labels match the selector AND it doesn't have an ownerReference, the ReplicaSet adopts it (sets ownerReference). This is rare but possible.

- **Orphaning** — if a Pod's labels are changed to no longer match the selector, the ReplicaSet's ownerReference is removed. The Pod becomes "orphaned" — it keeps running but is no longer managed.

This means **you can extract a Pod from a ReplicaSet by changing its labels** — useful for debugging:
```bash
kubectl label pod nginx-xyz pod-template-hash- # remove the hash label
# Now this Pod is orphaned; ReplicaSet creates a replacement
# You can debug the orphaned Pod without affecting service
```

**Deletion Logic:**

When the ReplicaSet needs to delete Pods (scaling down or replacing), it sorts candidates by:

1. **Not assigned** to a node (Pending Pods deleted first — they haven't started)
2. **Phase** — Pending > Unknown > Running
3. **Ready state** — not Ready before Ready
4. **Creation time** — newer before older (newer is presumed less "established")
5. **Restart count** — higher restart count first
6. **Pod name** — alphabetical as tiebreaker

This sorting tries to minimize impact — kill the least valuable Pods first.

**Status Maintenance:**

The controller updates the ReplicaSet status:
- `replicas` — count of Pods matching selector
- `fullyLabeledReplicas` — Pods with all template labels
- `readyReplicas` — Pods with Ready condition True
- `availableReplicas` — Ready Pods that have been Ready for minReadySeconds (used by Deployments)
- `observedGeneration` — the spec generation observed; used to detect controller staleness

**Relationship with Deployments:**

Deployments don't manage Pods directly. They:
1. Compute a hash of the PodTemplate (`pod-template-hash`)
2. Create a ReplicaSet with that hash in its labels
3. Update only the ReplicaSet's `replicas` field
4. The ReplicaSet does the actual Pod management

When you update a Deployment's template:
1. Hash changes
2. Existing ReplicaSet doesn't match the new hash
3. Deployment creates a new ReplicaSet
4. Deployment scales new RS up and old RS down
5. Old RS sticks around at replicas=0 for rollback history

This separation means **ReplicaSets are stable but Deployments are dynamic**. A ReplicaSet, once created, only changes its `replicas` count.

**Why Not Use ReplicaSet Directly?**

You could, but you lose:
- Rolling updates (you'd have to manage the swap manually)
- Rollback history (no revision tracking)
- Declarative version management

Always use Deployments for stateless workloads. Direct ReplicaSets are for very specialized cases or as internal building blocks.

**ReplicaSet vs ReplicationController (Legacy):**

The old ReplicationController used equality-based selectors only. ReplicaSet supports set-based selectors (`In`, `NotIn`, `Exists`, `DoesNotExist`), making it more flexible. ReplicationController is deprecated; always use ReplicaSet (or Deployment).

**Common Operations:**

```bash
# List ReplicaSets
kubectl get rs

# Describe a ReplicaSet
kubectl describe rs nginx-7d8f9c

# Scale directly (rare — usually scale the Deployment)
kubectl scale rs nginx-7d8f9c --replicas=5

# Delete a ReplicaSet
kubectl delete rs nginx-7d8f9c
# Note: this also deletes its Pods due to ownerReferences
```

---

### 29. What is the difference between requests and limits?

Requests and limits are the **resource management** primitives in Kubernetes. They serve completely different purposes — requests influence scheduling, limits enforce runtime constraints.

**Requests:**

A request is the **minimum** amount of a resource the container needs to run. It's used by the scheduler to find a node with sufficient capacity.

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
```

This says: "Schedule me on a node that has at least 500 millicores of CPU and 512 MiB of memory available."

**What requests affect:**

1. **Scheduling decisions** — the scheduler sums up requests of all Pods on a node and ensures new Pods fit within node capacity. A node with 4 cores can host Pods whose CPU requests sum to no more than 4 cores.

2. **QoS class assignment** — requests (and how they compare to limits) determine the Pod's QoS class.

3. **Eviction priority** — Pods using more than their requests are evicted first under node pressure.

4. **Horizontal Pod Autoscaler** — HPA uses CPU usage as a percentage of requests to make scaling decisions.

5. **Resource quota accounting** — namespace ResourceQuotas count against requested resources.

**Limits:**

A limit is the **maximum** amount of a resource the container can use. It's enforced at runtime by the kernel.

```yaml
resources:
  limits:
    cpu: "1000m"
    memory: "1Gi"
```

**What limits affect:**

1. **Runtime enforcement** — the container runtime configures cgroups to enforce these limits.

2. **CPU throttling** — if the container exceeds its CPU limit, the kernel CFS (Completely Fair Scheduler) throttles it. The process doesn't get killed, just slowed down.

3. **OOM kill on memory** — if the container exceeds its memory limit, the kernel's OOM killer terminates it. The container shows `OOMKilled` in status.

**Critical Differences:**

| Aspect | Requests | Limits |
|--------|----------|---------|
| Purpose | Minimum guaranteed | Maximum allowed |
| Used by | Scheduler | Runtime (kernel) |
| Enforcement | Scheduling time | Runtime |
| CPU exceeded | No effect | Throttled |
| Memory exceeded | No effect | OOMKilled |
| Required? | Optional but recommended | Optional |

**CPU Specifics:**

CPU is **compressible** — a container can be throttled without breaking. The unit is "cores" where 1000m (millicores) = 1 core.

- Request `500m` means: 0.5 cores reserved during scheduling
- Limit `1000m` means: kernel won't let this container use more than 1 core, even if available

CPU requests also affect **CPU shares** in cgroups — relative weighting when multiple containers compete for CPU. A container with request 1000m gets twice the shares of one with 500m.

**Memory Specifics:**

Memory is **incompressible** — you can't just "slow down" memory usage. The only enforcement is killing the process.

- Request `512Mi` means: 512 MiB reserved during scheduling
- Limit `1Gi` means: if the container uses more than 1 GiB, it's killed

Memory above request but below limit is "tolerated" — it works but the Pod can be evicted if the node experiences memory pressure.

**QoS Classes (Determined by Requests/Limits):**

**Guaranteed** — every container has equal requests and limits for both CPU and memory:
```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```
These Pods get the strongest guarantees — last to be evicted, lowest OOM score.

**Burstable** — at least one container has requests, but requests ≠ limits, or some resource is missing:
```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    memory: "512Mi"
```
Can burst above requests if resources are available. Middle eviction priority.

**BestEffort** — no requests or limits set on any container:
```yaml
# no resources block
```
First to be evicted. Highest OOM score. Generally avoid in production.

**Practical Recommendations:**

1. **Always set requests** — without them, Pods are BestEffort and unpredictably evicted.

2. **Set memory limits** — without them, a memory leak can take down the entire node.

3. **Be careful with CPU limits** — overly aggressive CPU limits cause throttling, which is often worse than just letting the container use available CPU. Many production environments don't set CPU limits at all.

4. **Set requests = limits for critical workloads** — get Guaranteed QoS for the strongest scheduling and survival guarantees.

5. **Right-size based on metrics** — use Prometheus or Vertical Pod Autoscaler (in recommend mode) to determine actual usage and tune accordingly.

**Common Anti-Patterns:**

- **No resources specified** — BestEffort, unpredictable
- **Limits without requests** — Kubernetes sets request=limit automatically (not always desired)
- **Way-too-high requests** — wastes cluster capacity, low utilization
- **Way-too-low limits** — frequent OOMKills, throttling, poor performance
- **Memory limit = memory request** — no headroom for spikes, frequent OOMKills

**Overcommitment:**

A cluster is "overcommitted" when the sum of limits exceeds node capacity, but the sum of requests doesn't. This is common and intended — most workloads don't use their full limits simultaneously. The risk is that if many workloads spike at once, evictions begin.

---

### 30. How does Kubernetes enforce resource quotas?

ResourceQuotas constrain **aggregate resource consumption per namespace**. They're a critical multi-tenancy mechanism — preventing one team from consuming all cluster resources.

**Basic Structure:**

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    persistentvolumeclaims: "10"
    pods: "50"
    services: "10"
    services.loadbalancers: "2"
    count/deployments.apps: "20"
    count/jobs.batch: "30"
```

**What Can Be Quota'd:**

**Compute resources:**
- `requests.cpu`, `requests.memory` — sum of all Pod CPU/memory requests
- `limits.cpu`, `limits.memory` — sum of all Pod CPU/memory limits
- `requests.ephemeral-storage`, `limits.ephemeral-storage`
- Extended resources: `requests.nvidia.com/gpu` etc.

**Object counts:**
- `pods` — number of Pods (excluding terminal phases)
- `services`, `services.loadbalancers`, `services.nodeports`
- `persistentvolumeclaims`
- `configmaps`, `secrets`
- `count/<resource>.<group>` for any resource type — `count/deployments.apps`, `count/jobs.batch`, etc.

**Storage:**
- `requests.storage` — sum of PVC storage requests
- `<storage-class>.storageclass.storage.k8s.io/requests.storage` — per-storage-class

**Enforcement Mechanism:**

The **ResourceQuota admission controller** intercepts every API request that creates or modifies relevant objects in a namespace with quotas.

For each create/update:
1. Calculate what resources the operation would consume
2. Compare to current usage + remaining quota
3. If it would exceed quota, **reject the request** with a 403 Forbidden

The quota controller in kube-controller-manager maintains the **status.used** field:
- Watches all relevant resources
- Recomputes usage on changes
- Updates the ResourceQuota status

**Compute Quota Specifics:**

When you have a quota on `requests.cpu`, **every Pod created in the namespace must specify a CPU request** — otherwise it's not countable. This is why ResourceQuota typically requires using LimitRange (which sets defaults) together.

A common pattern:
```yaml
# LimitRange sets defaults
apiVersion: v1
kind: LimitRange
metadata:
  name: defaults
  namespace: team-a
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

```yaml
# ResourceQuota caps total
apiVersion: v1
kind: ResourceQuota
metadata:
  name: caps
  namespace: team-a
spec:
  hard:
    requests.cpu: "20"
    limits.memory: "40Gi"
```

Now Pods without resource specs get defaults from LimitRange, satisfying the requirement that quotas can count them.

**Quota Scopes:**

Scopes allow quotas to apply to subsets of resources:

```yaml
spec:
  hard:
    pods: "10"
    requests.cpu: "5"
  scopes:
  - BestEffort  # only counts BestEffort Pods
```

Available scopes:
- **Terminating / NotTerminating** — Pods with/without activeDeadlineSeconds
- **BestEffort / NotBestEffort** — Pods by QoS class
- **PriorityClass** (via scopeSelector) — Pods of specific priority classes

This enables policies like "BestEffort Pods limited to 10, but Guaranteed Pods can have more."

**ScopeSelector for Fine-Grained Control:**

```yaml
spec:
  hard:
    pods: "1000"
  scopeSelector:
    matchExpressions:
    - scopeName: PriorityClass
      operator: In
      values: ["high", "critical"]
```

This quota only applies to high-priority Pods.

**Multiple Quotas in One Namespace:**

You can have multiple ResourceQuota objects in one namespace. They're all evaluated — an operation must pass all of them. This allows layered policies:

- Quota A: total compute caps
- Quota B: object count caps
- Quota C: BestEffort-specific limits
- Quota D: high-priority-class caps

**Quota Update Behavior:**

If you lower a quota below current usage, **existing resources are not deleted**. The quota is just exceeded until users delete things or until controllers can't create new ones. This is intentional — quotas are admission-time controls, not retroactive enforcement.

**Common Operations:**

```bash
# View all quotas
kubectl get resourcequota -A

# Detailed quota info
kubectl describe resourcequota team-a-quota -n team-a

# Output shows:
# Resource         Used   Hard
# requests.cpu     15.2   20
# requests.memory  30Gi   40Gi
# pods             32     50
```

**Common Failure Modes:**

When a Pod creation fails due to quota:
```
Error from server (Forbidden): error when creating "pod.yaml":
pods "nginx" is forbidden: exceeded quota: team-a-quota,
requested: requests.cpu=2, used: requests.cpu=19, limited: requests.cpu=20
```

This is admission-time. The user sees the error immediately.

**Interaction with Deployments and Controllers:**

When a Deployment tries to scale up but hits quota, the ReplicaSet controller can't create new Pods. The ReplicaSet status shows the error, and Pods stay at the lower count. Users see the issue in `kubectl describe rs`.

This can cause **rollout stalls** — if a rolling update needs maxSurge Pods temporarily but quota doesn't allow it, the rollout deadlocks.

**Best Practices:**

1. **Combine LimitRange with ResourceQuota** — defaults ensure Pods always have requests/limits
2. **Set quotas on both requests and limits** — controls scheduling reserve and overcommit
3. **Quota object counts** to prevent runaway controllers (a buggy operator creating millions of CMs)
4. **Use scopes** to differentiate workload classes
5. **Monitor quota usage** — alert when usage approaches limits to give teams time to react
6. **Account for rolling update overhead** — quotas need headroom for maxSurge during deployments

**Hierarchical Quotas:**

Vanilla Kubernetes doesn't support hierarchical quotas (parent namespace caps that constrain children). For this, you need:
- **Hierarchical Namespace Controller** (HNC) — adds parent-child relationships with inheritable quotas
- **Capsule** (open source) — multi-tenancy with tenant-level quotas
- **vCluster** — virtual clusters with their own quotas

---

# Advanced Kubernetes Interview Questions - Part 4

## Runtime, Networking, Secrets, Extensibility & API Management

---

### 31. Explain cgroups usage in Kubernetes.

Control groups (cgroups) are a Linux kernel feature that **limits, accounts for, and isolates resource usage** (CPU, memory, disk I/O, network) of process groups. Kubernetes leverages cgroups extensively for resource management — without them, requests and limits would be meaningless.

**Hierarchical Structure:**

Kubernetes organizes cgroups in a hierarchy on each node:

```
/sys/fs/cgroup/
└── kubepods.slice/                    # root cgroup for Kubernetes
    ├── kubepods-besteffort.slice/     # all BestEffort Pods
    │   └── kubepods-besteffort-pod<UID>.slice/
    │       └── cri-containerd-<container-id>.scope/
    ├── kubepods-burstable.slice/      # all Burstable Pods
    │   └── ...
    └── kubepods-pod<UID>.slice/       # Guaranteed Pods (top-level)
        └── ...
```

This is the **systemd cgroup driver** layout. There's also a `cgroupfs` driver with slightly different paths, but systemd is now recommended.

**Three-Tier Hierarchy:**

1. **Root cgroup (`kubepods`)** — caps total resources Kubernetes can use on the node. This protects the kubelet, container runtime, and system daemons from being starved.

2. **QoS class cgroups** — separate cgroups for BestEffort and Burstable Pods. Guaranteed Pods are placed at the same level as QoS cgroups (no parent cgroup grouping them). This hierarchy enables QoS-aware resource allocation.

3. **Pod cgroups** — each Pod gets its own cgroup containing all its containers. This enables Pod-level resource accounting and limits.

4. **Container cgroups** — individual containers within the Pod each get a sub-cgroup. This is where per-container limits are actually enforced.

**How Requests and Limits Map to cgroups:**

**CPU Requests → `cpu.shares` (cgroup v1) or `cpu.weight` (v2):**
- Used for proportional CPU allocation when contention exists
- A container with request 500m gets 512 shares (millicores × 1024 / 1000)
- A container with request 1000m gets 1024 shares
- Under contention, they get CPU proportional to their share ratio
- When no contention exists, containers can use all available CPU

**CPU Limits → `cpu.cfs_quota_us` and `cpu.cfs_period_us` (v1) or `cpu.max` (v2):**
- The Completely Fair Scheduler (CFS) enforces hard CPU caps
- Period is typically 100ms; quota is (limit × period)
- A container with limit 500m gets `cpu.cfs_quota_us=50000` (50ms per 100ms period)
- Once the container uses its quota in a period, it's **throttled** until the next period

**Memory Requests → No direct cgroup setting:**
- Used only for scheduling decisions
- Not enforced at runtime — the kernel doesn't reserve memory

**Memory Limits → `memory.limit_in_bytes` (v1) or `memory.max` (v2):**
- Hard memory cap; the kernel enforces this
- Exceeding triggers OOM kill (memory is incompressible)
- Includes RSS, page cache, and other accounted memory

**cgroup v1 vs v2:**

**cgroup v1** (legacy) uses separate hierarchies per controller (cpu, memory, blkio, etc.). A process can be in different cgroups for different controllers.

**cgroup v2** (unified) uses a single unified hierarchy where all controllers operate together. Benefits:
- Cleaner model — single hierarchy
- Better memory accounting (no double-counting of shared memory)
- Improved IO controller (cooperates with memory pressure)
- **Required for newer features** like MemoryQoS (memory.min, memory.high)

Most modern distros default to cgroup v2. Kubernetes fully supports v2 since v1.25.

**MemoryQoS (cgroup v2 feature):**
- `memory.min` — guaranteed memory the cgroup won't have reclaimed under pressure
- `memory.high` — soft memory limit; reclaim is triggered above this but doesn't OOM kill
- This enables more nuanced memory management than v1's binary limit

**CPU Throttling Issues:**

CPU limits frequently cause **unintended throttling**. A container running a burst (e.g., garbage collection pause) might be throttled even if the average usage is well below the limit, because throttling is per-100ms-window.

For example, a Java app with limit 1 core that runs a 50ms GC pause that uses 4 cores effectively needs 200ms of CPU compressed into 50ms. With CFS period 100ms and quota 100ms, this gets throttled severely.

Many production environments deliberately **don't set CPU limits** — they only set CPU requests. This avoids throttling while still providing scheduling guarantees. Memory limits should still be set since memory exhaustion is more dangerous than throttling.

**Kubelet cgroup Configuration:**

```yaml
# kubelet config
cgroupDriver: systemd                  # systemd or cgroupfs
cgroupsPerQOS: true                    # create QoS cgroups
cgroupRoot: /                          # base path for cgroups
enforceNodeAllocatable:                # what to enforce
- pods                                 # cap Pod resources at allocatable
- kube-reserved                        # reserve for kubelet, runtime
- system-reserved                      # reserve for OS
```

**Node Allocatable Calculation:**

```
Total node capacity
- kube-reserved (for kubelet, container runtime)
- system-reserved (for OS, kernel)
- eviction-hard threshold (buffer)
= Allocatable (available to Pods)
```

This is enforced via the root `kubepods` cgroup limit. Without it, Pods could starve the kubelet itself, causing cascading failures.

---

### 32. What is QoS class in Kubernetes?

Quality of Service (QoS) class is a **classification Kubernetes assigns to Pods based on their resource requests and limits**. It influences how the Pod is treated during resource contention — primarily for eviction priority and OOM kill order.

QoS class is **derived automatically** from the Pod's resource specification. You don't set it directly; you set requests and limits, and Kubernetes computes the class.

**The Three Classes:**

**Guaranteed:**
- Every container in the Pod has both CPU and memory requests AND limits specified
- requests must equal limits for both CPU and memory
- All containers (including init containers) must meet this

**Burstable:**
- At least one container has a request OR limit specified for CPU or memory
- But doesn't meet the Guaranteed criteria
- Most common class in practice

**BestEffort:**
- No requests or limits specified anywhere in the Pod
- All containers have no resource specifications

**Determining Order:**

Kubernetes evaluates Pods bottom-up:
1. Are any requests or limits set? No → BestEffort
2. Are CPU and memory requests and limits equal across all containers? Yes → Guaranteed
3. Otherwise → Burstable

**Where QoS Matters:**

QoS class directly affects:

1. **Eviction order during node pressure**:
   - BestEffort first (lowest priority)
   - Burstable next (those exceeding requests prioritized)
   - Guaranteed last (highest priority)

2. **OOM kill score adjustment** — Linux kernel's OOM killer uses oom_score_adj:
   - BestEffort: 1000 (most likely to be killed)
   - Burstable: 1000 - (1000 × memory_request / node_memory_capacity), bounded
   - Guaranteed: -997 (least likely to be killed)

3. **Static CPU manager pinning** — only Guaranteed Pods with integer CPU requests can get **exclusive CPU pinning** via the static CPU manager. Burstable and BestEffort always share the CPU pool.

4. **Memory manager features** — similarly, Guaranteed Pods can leverage NUMA-aware memory allocation.

5. **Topology manager** — coordinates with CPU and memory managers; Guaranteed Pods get the best hardware locality.

**Practical Implications:**

For production-critical workloads, **always use Guaranteed QoS**. Set requests = limits. The Pod is harder to evict, less likely to be OOM killed, and gets best-effort hardware placement.

For development or batch workloads, Burstable is fine — set requests but allow limits higher for bursting.

Avoid BestEffort in production entirely. They're first in line for eviction during any node issue.

---

### 33. Explain BestEffort, Burstable, and Guaranteed Pods.

Covered conceptually above, but here's a deeper dive into each class with practical implications.

**Guaranteed Pods:**

The strongest resource guarantees Kubernetes provides. Requirements:
- Every container has CPU request = CPU limit
- Every container has memory request = memory limit
- Both CPU and memory are specified (you can't be Guaranteed for just one)
- Applies to all init containers too

Example:
```yaml
resources:
  requests:
    cpu: "1000m"
    memory: "1Gi"
  limits:
    cpu: "1000m"
    memory: "1Gi"
```

**Behavior:**
- The Pod always gets exactly what it requested — no more, no less
- CPU is reserved on the node; no other Pod can use it (when CPU pinning enabled)
- Memory is hard-capped at 1Gi; exceeding triggers OOM kill
- Last to be evicted under node pressure
- Lowest OOM score (around -997)
- Can use exclusive CPU cores via static CPU manager (with integer CPU requests)

**Use Cases:**
- Production databases, caches, mission-critical services
- Latency-sensitive workloads (no resource competition)
- Workloads requiring predictable performance
- Anything you absolutely don't want killed

**Burstable Pods:**

The most common QoS class. Requirements:
- At least one container has a request or limit set
- But not all containers have matching request = limit

Example:
```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

**Behavior:**
- Guaranteed at least the request amount
- Can burst up to the limit if resources are available
- Above request, may be throttled (CPU) or competing for memory
- Middle eviction priority — Pods using above their request evicted before those at/below request
- OOM score scales based on memory request relative to node capacity
- Can't use exclusive CPU cores

**Use Cases:**
- Most application workloads
- Web services with variable load
- Development environments
- Stateless apps where occasional throttling/eviction is acceptable

**BestEffort Pods:**

The most flexible but least protected. Requirements:
- No requests or limits set anywhere

Example:
```yaml
spec:
  containers:
  - name: app
    image: nginx
    # no resources block at all
```

**Behavior:**
- No resource guarantees whatsoever
- Can use any available resources on the node
- First evicted under any node pressure
- Highest OOM score (1000 — kernel kills these first)
- No reservation — scheduler treats them as having 0 requests
- Many can fit on a node, but they're the most fragile

**Use Cases:**
- Best avoided in production
- Batch jobs that can tolerate restart
- Experimentation in development clusters
- Workloads where killing/restart is cheap

**Practical Decision Matrix:**

| Workload Type | Recommended QoS |
|---------------|-----------------|
| Production database | Guaranteed |
| Production API (critical) | Guaranteed |
| Production API (standard) | Burstable |
| Background workers | Burstable |
| Batch jobs | Burstable |
| CI/CD jobs | Burstable |
| Experimental workloads | Burstable |
| Anything in production | Not BestEffort |

**Real-World Tip:**

A common mistake is setting memory limits without CPU limits, or vice versa. This puts you in Burstable when you might have intended Guaranteed. Always specify both if you want Guaranteed.

Another mistake: setting overly large requests "just in case." This wastes cluster resources because the scheduler reserves them whether used or not. Right-size based on actual measurements.

---

### 34. How does Kubernetes DNS resolution work?

Kubernetes DNS is the service discovery foundation. It provides stable DNS names for Services and Pods, allowing applications to find each other without hardcoded IPs.

**Architecture Overview:**

CoreDNS runs as a Deployment (typically 2+ replicas) in `kube-system`, fronted by a Service named `kube-dns` with a stable ClusterIP (usually `10.96.0.10`). Every Pod is configured to use this IP as its DNS resolver.

**Pod DNS Configuration:**

When a Pod is created, kubelet writes `/etc/resolv.conf` based on the cluster DNS configuration:

```
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

The search list and ndots are critical to understand for performance optimization.

**The dnsPolicy Field:**

Pods can have different `dnsPolicy` values:
- **ClusterFirst** (default) — uses cluster DNS, with the search domains above
- **ClusterFirstWithHostNet** — same as ClusterFirst but for Pods using hostNetwork
- **Default** — uses the node's DNS (whatever's in the node's resolv.conf)
- **None** — no DNS configuration; you must specify `dnsConfig` manually

**DNS Records Created:**

For a Service `web` in namespace `production`:

**A/AAAA Records (for ClusterIP Services):**
- `web.production.svc.cluster.local` → ClusterIP
- Also accessible as `web.production` and `web` (via search domains)

**A/AAAA Records (for Headless Services):**
Headless Services (clusterIP: None) return multiple A records, one per backing Pod:
- `web.production.svc.cluster.local` → Pod1-IP, Pod2-IP, Pod3-IP
- Plus per-Pod records: `pod-name.web.production.svc.cluster.local`

**SRV Records:**
- `_<port-name>._<protocol>.<service>.<namespace>.svc.cluster.local` → priority, weight, port, target
- Example: `_http._tcp.web.production.svc.cluster.local`

**StatefulSet Pod DNS:**
StatefulSets get stable per-Pod DNS:
- `pod-ordinal.headless-service.namespace.svc.cluster.local`
- e.g., `mysql-0.mysql.default.svc.cluster.local`

This stability is what allows things like database clusters to know their peers.

**The Resolution Flow:**

When a Pod queries `api.production`:

1. **Pod's resolver** consults `/etc/resolv.conf`
2. With `ndots:5` and only 1 dot in `api.production`, search expansion happens
3. Tries `api.production.default.svc.cluster.local` (first search domain) → CoreDNS returns NXDOMAIN
4. Tries `api.production.svc.cluster.local` → CoreDNS finds the Service, returns ClusterIP
5. Resolver returns the IP to the application

For `api` (no dots, less than 5):
1. Tries `api.default.svc.cluster.local` → found, return ClusterIP

For `example.com` (1 dot, less than 5):
1. Tries `example.com.default.svc.cluster.local` → NXDOMAIN
2. Tries `example.com.svc.cluster.local` → NXDOMAIN
3. Tries `example.com.cluster.local` → NXDOMAIN
4. Tries `example.com` (absolute) → CoreDNS forwards to upstream → returns real IP

This is why **external queries can be slow** — multiple negative lookups before reaching the external resolver.

**CoreDNS Plugin Pipeline:**

When CoreDNS receives a query, it processes through configured plugins:

```
.:53 {
    errors                              # log errors
    health                              # health endpoint
    ready                               # readiness endpoint
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure                   # Pod records
        fallthrough in-addr.arpa ip6.arpa
        ttl 30
    }
    prometheus :9153                    # metrics
    forward . /etc/resolv.conf {        # external queries
        max_concurrent 1000
    }
    cache 30                            # 30-second cache
    loop                                # loop detection
    reload                              # auto-reload Corefile
    loadbalance                         # randomize A records
}
```

The `kubernetes` plugin watches the API server for Services, Endpoints, and Pods. Queries matching `cluster.local` are answered from this cache. Other queries pass through `forward` to upstream DNS.

**Performance Optimizations:**

**NodeLocal DNSCache:**
A DaemonSet running CoreDNS on every node, listening on a link-local address (typically `169.254.20.10`). Pods are configured to use this local cache instead of the cluster DNS Service.

Benefits:
- No iptables/IPVS hop through kube-proxy
- Cache hits served locally
- Reduced latency
- Reduces load on central CoreDNS
- Better handling of conntrack table exhaustion

**ndots Tuning:**
The default `ndots:5` causes many spurious lookups for external names. For Pods that primarily talk to external services, set lower ndots:
```yaml
dnsConfig:
  options:
  - name: ndots
    value: "2"
```

**Use FQDN for External:**
End external hostnames with a trailing dot to skip search expansion:
- `example.com.` instead of `example.com`

**Caching:**
Increase CoreDNS cache TTL for stable lookups. Note that this might delay propagation of Service IP changes.

**DNS Issues — Common Problems:**

1. **DNS lookups returning NXDOMAIN intermittently** — often CoreDNS replicas can't reach the API server, or watch is stale. Restart CoreDNS pods.

2. **Slow DNS** — usually ndots or search domain issues. Use NodeLocal DNSCache, tune ndots.

3. **DNS lookups timing out** — could be conntrack table full (large clusters), kube-proxy issues, or CoreDNS overloaded. Scale CoreDNS, add NodeLocal DNSCache.

4. **Cross-namespace queries failing** — make sure you use the FQDN: `service.namespace` or `service.namespace.svc.cluster.local`.

5. **External DNS queries slow** — search domain expansion. Use absolute names with trailing dot, or lower ndots.

---

### 35. Explain the pause container concept.

The pause container is one of the most elegant and underappreciated pieces of Kubernetes. It's the **invisible foundation** of every Pod.

**The Problem It Solves:**

A Pod is a "group of containers that share resources" — but how do you actually implement that in Linux? Containers are processes with namespaces (PID, network, IPC, mount, UTS, user). To share network and IPC between containers, you need to share namespaces.

In Linux, namespaces are owned by processes. If process A creates a network namespace and dies, the namespace is destroyed. This is problematic for Pods where individual containers might restart.

**The Solution: Pause Container**

Each Pod has a hidden "pause" container (also called the "infra container" or "sandbox container") that:
1. Starts first when the Pod is created
2. Creates and holds the network, IPC, and PID namespaces
3. Does literally nothing — just sleeps forever
4. Other Pod containers join its namespaces instead of creating their own

The image is tiny — typically ~700KB total. The source code is famously simple — fewer than 50 lines of C that essentially calls `pause()` syscall.

**Source code essence (simplified):**
```c
int main() {
    // ignore signals so we don't get killed easily
    signal(SIGINT, sigdown);
    signal(SIGTERM, sigdown);
    signal(SIGCHLD, sigreap);

    // wait forever
    for (;;) pause();
    return 0;
}
```

**What It Does:**

1. **Holds the network namespace** — when application containers in the Pod start, they join this network namespace. That's why all Pod containers share localhost and the same IP.

2. **Holds the IPC namespace** — containers can communicate via System V IPC, POSIX shared memory.

3. **PID 1 in the Pod (sort of)** — when shareProcessNamespace is enabled, pause is PID 1 and reaps zombie processes from other containers.

4. **Survives container restarts** — if your nginx container crashes, the pause container is still there holding the namespace, so the new nginx joins the same namespace and keeps the same Pod IP.

**Why "Pause"?**

It calls the Linux `pause()` syscall, which suspends the process until a signal is received. The process effectively sleeps forever using zero CPU.

**Lifecycle:**

When you create a Pod:
1. kubelet receives the Pod spec
2. kubelet asks the CRI runtime to create a "PodSandbox" — this creates the pause container
3. The pause container starts, creating namespaces
4. kubelet asks the CRI runtime to start each application container, joining the sandbox's namespaces
5. Application containers run; pause container sleeps

When the Pod is deleted:
1. Application containers are stopped (SIGTERM, then SIGKILL after grace period)
2. The pause container is stopped last
3. Once pause exits, the namespaces are destroyed
4. Network plugins are called to tear down the Pod's network config

**Why You Don't See It:**

`kubectl get pods` doesn't show pause containers because they're CRI-level constructs, not user containers. But you can see them with low-level tools:

```bash
# On the node
crictl pods
crictl ps -a   # shows all containers including pause
docker ps -a | grep pause   # if using Docker runtime
```

**Resource Accounting:**

The pause container is included in resource accounting. It uses negligible CPU and memory (a few hundred KB), so it doesn't matter in practice. But for very high Pod density nodes, the cumulative overhead is real (e.g., 100 Pods = 100 pause containers).

**The Image:**

The standard pause image is `registry.k8s.io/pause:3.X`. Each Kubernetes version has its own. Custom container runtimes can use their own pause image (e.g., containerd has its own).

You can configure kubelet's pause image:
```yaml
# kubelet config
podInfraContainerImage: registry.k8s.io/pause:3.9
```

**Why Not Use One of the App Containers as PID 1?**

Could you skip the pause container by having one app container hold the namespaces? Technically yes, but problems arise:
- If that container restarts, the Pod loses its IP and namespaces
- You'd need to designate which container is "special"
- Container lifecycle becomes complex

The pause container is dedicated, minimal, and predictable. It's the cleanest design.

---

### 36. What are init containers?

Init containers are **specialized containers that run to completion before app containers start**. They're the way Kubernetes handles initialization tasks that don't fit cleanly into the application container itself.

**Key Characteristics:**

- Run **sequentially**, in order, before any app containers start
- Each must complete successfully before the next runs
- If any init container fails, the Pod is restarted (according to restartPolicy)
- Have their own resource requests/limits, image, command, etc.
- Share volumes with app containers
- Don't have liveness/readiness probes (they're transient)
- Don't support lifecycle hooks (preStop, postStart)

**Example:**

```yaml
spec:
  initContainers:
  - name: wait-for-db
    image: busybox:1.36
    command: ['sh', '-c', 'until nc -z db 5432; do sleep 1; done']
  - name: run-migrations
    image: myapp/migrations:v1
    command: ['./migrate', '--config', '/etc/migrations.yaml']
    volumeMounts:
    - name: config
      mountPath: /etc/migrations.yaml
  containers:
  - name: app
    image: myapp:v1
    # main app container
  volumes:
  - name: config
    configMap:
      name: app-config
```

**Common Use Cases:**

**1. Waiting for Dependencies:**
Wait for a service or database to be ready before starting the app.

```yaml
initContainers:
- name: wait-for-service
  image: busybox
  command: ['sh', '-c', 'until nslookup myservice; do sleep 2; done']
```

**2. Performing Setup:**
Run database migrations, generate configuration, prepare data.

```yaml
initContainers:
- name: setup-db
  image: myapp/setup:v1
  command: ['./initialize-database']
```

**3. Cloning Repositories:**
Pull configuration or content from Git before app starts.

```yaml
initContainers:
- name: git-clone
  image: alpine/git
  command: ['git', 'clone', 'https://repo.git', '/data']
  volumeMounts:
  - name: shared-data
    mountPath: /data
```

**4. Setting File Permissions:**
Adjust ownership of volumes (especially for non-root containers).

```yaml
initContainers:
- name: chown
  image: busybox
  command: ['sh', '-c', 'chown -R 1000:1000 /data']
  volumeMounts:
  - name: data
    mountPath: /data
```

**5. Generating Secrets/Certificates:**
Fetch credentials from secret managers, generate TLS certs.

**6. Tool Installation:**
Install tools or download artifacts that the main container needs but shouldn't be in its image.

**Why Use Init Containers Instead of Embedding Logic?**

1. **Separation of concerns** — init logic doesn't pollute the app image
2. **Different tooling** — init might need tools (curl, git, kubectl) that you don't want in production app images
3. **Different security context** — init might need root for chown, app runs as non-root
4. **Better failure visibility** — if init fails, you know it's the init step, not the app
5. **Reusability** — same init container can be used across different Pods

**Restart Behavior:**

If an init container fails:
- The Pod restartPolicy determines what happens
- `Always` (default): the Pod is restarted, including all init containers from the beginning
- `OnFailure`: same as Always for init failures
- `Never`: the Pod fails

If init containers succeed but main containers fail, init containers don't re-run on restart — only the main containers restart.

**Resource Implications:**

Init containers can request more resources than app containers — useful for resource-intensive setup. The Pod's effective request is the max of:
- The sum of any single init container's resources (since they run one at a time)
- The sum of all app containers' resources (they run in parallel)

So if init container needs 4GB but app containers total 2GB, the Pod's effective memory request is 4GB.

**Init Containers vs Sidecar Containers:**

Traditionally, init containers run before app containers and terminate. **Sidecar containers** (made native in v1.29) are a special type that:
- Run alongside the main container
- Have lifecycle independent of main container
- Are defined as init containers with `restartPolicy: Always`

This bridges the gap — useful for log shippers, proxies, etc. that need to start before the main app but keep running.

---

### 37. Explain ephemeral containers and debugging use cases.

Ephemeral containers are a special type of container designed for **debugging running Pods** without modifying the Pod spec or restarting containers. Introduced in v1.16 (GA in v1.25).

**The Problem They Solve:**

Imagine your production Pod is misbehaving. You want to:
- Run `netstat`, `ps`, `tcpdump`, or other diagnostic tools
- But your production image is minimal (FROM scratch, distroless) — no shell, no tools
- You can't add tools to the image without rebuilding and redeploying
- You can't modify a running Pod's containers field

Ephemeral containers solve this — you attach a debug container to a running Pod, sharing namespaces with the target container.

**Key Characteristics:**

- Added to a Pod **after creation** via the ephemeralcontainers subresource
- Can't be removed once added (Pod must be deleted/recreated to remove)
- No resource requests/limits (they're scavenging resources)
- No probes (liveness, readiness, startup)
- No ports
- No restart policy (they don't restart)
- Can share PID namespace with target containers (via `targetContainerName`)
- Useful image: `busybox`, `nicolaka/netshoot` (loaded with networking tools)

**Adding an Ephemeral Container:**

The easiest way is `kubectl debug`:

```bash
# Attach a debug container to a running Pod
kubectl debug -it my-pod --image=nicolaka/netshoot

# Share PID namespace with a specific container
kubectl debug -it my-pod --image=nicolaka/netshoot --target=my-app

# Debug a node (creates a Pod on the node)
kubectl debug node/my-node -it --image=ubuntu
```

You can also do it via raw API:

```yaml
# PATCH /api/v1/namespaces/default/pods/my-pod/ephemeralcontainers
spec:
  ephemeralContainers:
  - name: debug
    image: nicolaka/netshoot
    stdin: true
    tty: true
    targetContainerName: my-app
```

**Debugging Use Cases:**

**1. Inspecting a Running Container's Filesystem:**

With process namespace sharing, you can see the target container's filesystem via `/proc`:

```bash
kubectl debug -it my-pod --image=ubuntu --target=my-app
# Inside the debug container:
ls /proc/1/root/  # see the target's filesystem
cat /proc/1/environ  # see environment variables
```

**2. Network Debugging:**

```bash
kubectl debug -it my-pod --image=nicolaka/netshoot
# Inside:
tcpdump -i eth0
ss -tulpn
nslookup other-service
curl -v http://other-service:8080
```

**3. Process Debugging:**

```bash
kubectl debug -it my-pod --image=ubuntu --target=my-app
# With shared PID namespace:
ps auxf
strace -p 1  # attach to PID 1 of target
```

**4. Debugging Distroless Images:**

Distroless images (no shell, no package manager) are great for production but terrible for debugging. Ephemeral containers let you bring debugging tools without modifying the image:

```bash
kubectl debug -it my-distroless-pod --image=busybox --target=app
```

**5. Copying a Pod for Safer Debugging:**

`kubectl debug` can also create a copy of a Pod with debugging modifications:

```bash
# Create a copy with debug tools, leaving original intact
kubectl debug my-pod -it --image=busybox --copy-to=my-pod-debug

# Copy and change the command
kubectl debug my-pod -it --copy-to=my-debug --container=app -- sh
```

**Node Debugging:**

`kubectl debug node/<node-name>` creates a privileged Pod on the node with the host filesystem mounted at /host:

```bash
kubectl debug node/worker-1 -it --image=ubuntu
# Inside:
chroot /host  # now you're effectively on the node
journalctl -u kubelet
```

**Limitations:**

- Once added, ephemeral containers can't be removed (you must delete the Pod)
- They use the Pod's resources — be careful with very tight limits
- They can't bind to ports the main containers already use
- Some clusters disable them via admission policies

**Security Considerations:**

Ephemeral containers can be powerful and dangerous. They:
- Can access the same volumes
- Can see secrets mounted in the Pod
- Can be privileged
- Have access to the same network

Restrict via RBAC who can use the `pods/ephemeralcontainers` subresource. In multi-tenant environments, you usually want only cluster admins to have this permission.

**Comparison with Alternatives:**

**Before ephemeral containers:**
- `kubectl exec` — only works if there's a shell in the image
- Rebuild image with tools — slow, may not match production
- Sidecar with debug tools — pollutes production Pods

**Ephemeral containers**:
- Work on minimal images
- No Pod modification needed
- Can be added on demand

---

### 38. Difference between liveness, readiness, and startup probes.

These three probe types serve **distinct purposes** in container lifecycle management. Misunderstanding them is one of the most common sources of production issues.

**Liveness Probe:**

**Purpose:** Detects when a container is **alive but broken** — running but unable to recover (deadlock, infinite loop, hung state).

**Action on Failure:** kubelet **kills and restarts** the container.

**Behavior:**
- After `failureThreshold` consecutive failures, the container is killed
- Container restarts according to restartPolicy
- Pod-level identity (IP, etc.) is preserved
- Restart counter increments

**Example:**
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
```

This probes every 10s, gives 5s to respond, restarts after 3 consecutive failures.

**Common Pitfall:**

Liveness probes that depend on external services (databases, caches) cause cascading failures. If the database is slow, every Pod fails its liveness probe and restarts simultaneously, making things worse. Liveness probes should check **only the local container's health**.

**Use sparingly:**
- A well-designed app rarely needs liveness probes
- If your app crashes on errors, the container restarts naturally
- Liveness probes are for cases where the process is alive but stuck

**Readiness Probe:**

**Purpose:** Detects when a container is **ready to serve traffic**.

**Action on Failure:** Pod is **removed from Service endpoints**. Traffic stops, but the container keeps running.

**Behavior:**
- After `failureThreshold` failures, the Pod is marked NotReady
- The Endpoint controller removes it from EndpointSlices
- kube-proxy on all nodes updates rules to skip this Pod
- When the probe succeeds again, the Pod is added back

**Example:**
```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
```

**Use Cases:**

- **Startup time** — app takes time to warm up before serving
- **Temporary unavailability** — app is processing a batch job and shouldn't accept new requests
- **Dependency failures** — app can't function without database; mark itself unready until database is back
- **Graceful shutdown** — app is finishing in-flight requests, no longer accepting new ones

**Key Difference from Liveness:**

Liveness restarts the container (assumes broken). Readiness just removes from load balancer (assumes temporarily not ready). For dependency checks, **use readiness, not liveness**.

**Startup Probe:**

**Purpose:** Determines when **the application has finished starting up**, so that liveness and readiness probes can begin.

**Action on Failure:** Container is killed (like liveness) but only during startup phase.

**Behavior:**
- Runs only **once during startup**
- Liveness and readiness probes are paused until startup probe succeeds
- After success, switches to liveness/readiness probes
- If startup fails for `failureThreshold` × `periodSeconds`, the container is killed

**Example:**
```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

This gives the app up to 300 seconds (30 × 10s) to start. After success, regular probes begin.

**Why Needed:**

Before startup probes existed, slow-starting apps had a problem:
- Liveness probe with short `initialDelaySeconds` would fail during startup → restart loop
- Liveness probe with long `initialDelaySeconds` would mask real issues post-startup

Startup probes decouple "is starting" from "is healthy" — long startup tolerance separately from normal operation.

**Use Cases:**

- Java apps with slow JVM warmup
- Apps that load large amounts of data on startup
- Apps with extensive initialization (caches, ML models)
- Legacy apps with unpredictable startup times

**Probe Types (All Three Probes Support):**

**httpGet:**
```yaml
httpGet:
  path: /healthz
  port: 8080
  scheme: HTTP
  httpHeaders:
  - name: Custom-Header
    value: Awesome
```
Considered successful if HTTP response code is 200-399.

**tcpSocket:**
```yaml
tcpSocket:
  port: 5432
```
Successful if TCP connection can be established.

**exec:**
```yaml
exec:
  command:
  - cat
  - /tmp/healthy
```
Successful if command exits with code 0.

**grpc:**
```yaml
grpc:
  port: 9000
  service: "my-service"
```
Uses the standard gRPC health checking protocol.

**Probe Configuration Parameters:**

- `initialDelaySeconds` — wait this long after container start before first probe
- `periodSeconds` — interval between probes (default 10s)
- `timeoutSeconds` — how long to wait for response (default 1s)
- `successThreshold` — consecutive successes needed (default 1; min for non-liveness is 1)
- `failureThreshold` — consecutive failures before action (default 3)
- `terminationGracePeriodSeconds` — override Pod's grace period when probe-killed

**Probe Endpoint Design:**

**Liveness endpoint should:**
- Check only local state (is the process responsive?)
- NOT check dependencies (database, external services)
- Be fast and lightweight
- Examples: "/healthz" returning OK if event loop is responsive

**Readiness endpoint should:**
- Check whether ready to serve traffic
- May check dependencies if absence would cause failures
- Should be fast
- Examples: "/ready" returning OK if database connection pool is healthy

**Startup endpoint** can be the same as liveness, just with longer tolerance.

**Decision Matrix:**

| Scenario | Use |
|----------|-----|
| App may deadlock | Liveness |
| App takes long to start | Startup |
| App temporarily can't serve | Readiness |
| Dependency unavailable | Readiness (not Liveness!) |
| Memory leak forcing restart | Liveness (but fix the leak) |
| Slow JVM startup | Startup |

---

### 39. Explain graceful shutdown handling in Kubernetes.

Graceful shutdown is the **coordinated process** of terminating a Pod while minimizing impact on users — completing in-flight requests, deregistering from load balancers, cleaning up resources.

**The Termination Sequence:**

When a Pod is deleted (via kubectl, scaling down, eviction, etc.):

**Step 1: Deletion API Call**
- API server receives delete request
- Sets `deletionTimestamp` on the Pod
- Pod enters "Terminating" state
- Default `terminationGracePeriodSeconds` is 30s

**Step 2: Parallel Actions Begin**

Simultaneously, two things happen:

**A) Endpoint Removal:**
- Endpoint Controller observes the Terminating state
- Removes the Pod from all Service EndpointSlices
- All kube-proxy instances across the cluster update their iptables/IPVS rules
- New traffic stops being routed to this Pod
- **But this takes time to propagate** — there's a window where traffic still arrives

**B) Kubelet Begins Termination:**
- kubelet on the node sees the deletion event
- Begins the container shutdown sequence

**Step 3: PreStop Hook (If Defined)**

The preStop hook runs **before** SIGTERM is sent. Useful for explicit cleanup actions.

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 10"]
```

Or HTTP:
```yaml
lifecycle:
  preStop:
    httpGet:
      path: /shutdown
      port: 8080
```

The preStop hook is synchronous — kubelet waits for it to complete before SIGTERM. But it runs **within the grace period**, not in addition to it.

**Step 4: SIGTERM Sent**
- Kubelet sends SIGTERM to PID 1 of each container
- The container should begin graceful shutdown — finish current requests, close connections, flush buffers
- The grace period countdown is active

**Step 5: Wait for Containers to Exit**
- Kubelet waits for containers to exit cleanly
- If they exit before grace period ends, cleanup proceeds immediately

**Step 6: SIGKILL (If Necessary)**
- If grace period expires and containers are still running
- Kubelet sends SIGKILL — immediate termination
- No more cleanup happens in the container

**Step 7: Cleanup**
- Volumes are unmounted
- Network is torn down (CNI delete)
- Pause container is stopped
- Pod object is removed from etcd

**The Endpoint Removal Race Condition:**

There's a fundamental race: SIGTERM is sent to your container immediately, but endpoint removal takes time to propagate across all nodes (typically 0.5-5 seconds, depending on cluster size). During this window, **traffic still arrives at the terminating Pod**.

If your app closes its listening socket as soon as it receives SIGTERM, those in-flight requests fail.

**The Solution: PreStop Sleep**

A common pattern is a preStop sleep before SIGTERM:

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "sleep 15"]
terminationGracePeriodSeconds: 60
```

The sleep gives kube-proxy time to update endpoints across the cluster before your app starts shutting down. The grace period must accommodate the sleep plus actual shutdown time.

**Application-Level Graceful Shutdown:**

Your application should handle SIGTERM properly:

```python
import signal
import sys

def handle_shutdown(signum, frame):
    print("Received SIGTERM, shutting down...")
    server.stop_accepting_connections()
    server.wait_for_requests_to_complete(timeout=25)
    sys.exit(0)

signal.signal(signal.SIGTERM, handle_shutdown)
```

**Common Issues:**

**1. Application doesn't trap SIGTERM:**
By default, many apps just exit on SIGTERM without cleanup. Bash scripts especially — `command: ["/bin/sh", "script.sh"]` traps SIGTERM at sh, not the actual app. Use `exec` to replace the shell:

```bash
#!/bin/sh
exec my-app "$@"
```

**2. PID 1 doesn't forward signals:**
If your app runs as a child of a wrapper (init system, supervisord), SIGTERM may not reach the actual process. Use `tini` or `dumb-init`:

```dockerfile
ENTRYPOINT ["tini", "--", "my-app"]
```

**3. Grace period too short:**
Default 30s is often too short for apps with long-running requests. Increase it:

```yaml
terminationGracePeriodSeconds: 60
```

**4. Database connections not closed properly:**
Apps must close DB connections gracefully to avoid leaving them in a half-open state.

**5. In-flight HTTP requests dropped:**
HTTP servers must stop accepting new connections, then wait for current ones to finish. Most modern frameworks support this (Go's `Server.Shutdown()`, Node's `server.close()`, etc.).

**Forceful Deletion:**

```bash
kubectl delete pod my-pod --force --grace-period=0
```

This skips graceful shutdown entirely — the Pod object is removed from etcd immediately, and kubelet sends SIGKILL. Use only when:
- The node is unreachable (Pod won't terminate normally)
- The Pod is truly stuck and won't shut down
- You accept the risk of data loss / corrupt state

**StatefulSet Considerations:**

For StatefulSets, the controller waits for each Pod to fully terminate before scaling the next one down (during ordered shutdown). This means graceful shutdown is critical — stuck Pods block the entire scale-down.

**Final Steps — Pod Cleanup:**

After containers exit:
1. Volumes unmounted (CSI NodeUnpublish + NodeUnstage)
2. Persistent volumes detached if no other Pods use them (CSI ControllerUnpublish)
3. Network resources released (CNI DEL)
4. Pause container stopped
5. Pod's cgroup removed
6. Pod object removed from etcd (after all finalizers complete)

---

### 40. What is PodDisruptionBudget?

PodDisruptionBudget (PDB) is a policy object that **limits voluntary disruptions** to a set of Pods. It ensures that critical workloads maintain availability during cluster operations like node drains, upgrades, and rebalancing.

**Voluntary vs Involuntary Disruptions:**

**Voluntary disruptions** — initiated by humans or controllers:
- `kubectl drain` for node maintenance
- Cluster autoscaler scaling down
- Karpenter consolidating workloads
- Manual Pod evictions via the Eviction API

**Involuntary disruptions** — outside Kubernetes' control:
- Node hardware failure
- Kernel panic
- Network partition
- Out-of-memory kills

**PDBs only apply to voluntary disruptions.** During a node failure, PDBs don't prevent Pod loss.

**Basic Structure:**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
  namespace: production
spec:
  minAvailable: 2
  # OR
  maxUnavailable: 1
  selector:
    matchLabels:
      app: web
```

**Two Modes:**

**minAvailable** — the minimum number (or percentage) of Pods that must remain available.

```yaml
spec:
  minAvailable: 3
  # or
  minAvailable: 50%
```

With 10 replicas and minAvailable: 3, voluntary evictions can take up to 7 Pods at once.

**maxUnavailable** — the maximum number (or percentage) of Pods that can be unavailable.

```yaml
spec:
  maxUnavailable: 1
  # or
  maxUnavailable: 25%
```

With 10 replicas and maxUnavailable: 1, only 1 Pod can be voluntarily disrupted at a time.

**How It Works:**

When something tries to voluntarily disrupt a Pod (e.g., `kubectl drain` sends eviction requests):

1. The Eviction API receives the request
2. Checks all matching PDBs
3. Calculates currently available Pods vs PDB requirements
4. If eviction would violate the PDB → request rejected with 429 Too Many Requests
5. If allowed → standard graceful deletion proceeds

**Realistic Scenario: Node Drain**

You have 10 web Pods spread across 5 nodes. You need to drain a node hosting 2 of them for maintenance.

Without PDB:
```bash
kubectl drain node-3
# Both Pods on node-3 are evicted simultaneously
# Service drops 20% capacity instantly
```

With PDB (`maxUnavailable: 1`):
```bash
kubectl drain node-3
# First Pod is evicted; drain waits
# When the first Pod's replacement is Ready, second Pod is evicted
# Service maintains at least 9/10 availability throughout
```

This **serializes evictions** — much safer for production.

**Critical: PDB Doesn't Trigger Pod Creation**

PDBs are gatekeepers, not creators. They block evictions but don't create new Pods. The replacement Pods come from whatever controller owns the disrupted Pods (Deployment, StatefulSet).

This means if your Deployment scales itself to 0 manually, PDB doesn't prevent that — it only affects eviction requests.

**PDB and Cluster Autoscaler:**

Cluster autoscalers respect PDBs when scaling down:
- Identify candidate nodes to remove
- Check that removing them wouldn't violate any PDB
- If PDB would be violated → wait, or pick another node

This can **block scale-down** indefinitely if a PDB is too strict. Common issue: PDB with `minAvailable: 100%` blocks ALL evictions, preventing node removal.

**Common PDB Configurations:**

**Always have at least 50% available:**
```yaml
spec:
  minAvailable: 50%
```

**Allow only one disruption at a time:**
```yaml
spec:
  maxUnavailable: 1
```

**Critical workload — never drop below 3:**
```yaml
spec:
  minAvailable: 3
```

**Single-replica edge case:**
```yaml
spec:
  minAvailable: 1
  # With replicas: 1, this means no evictions can happen at all
  # Use maxUnavailable: 0 for same effect
```

For single-replica workloads, PDBs are often counterproductive — you literally can't evict ever.

**Selector and Templates:**

PDBs use label selectors, just like Services. The selector typically matches the Deployment's template labels:

```yaml
spec:
  selector:
    matchLabels:
      app: web
      tier: frontend
```

**Counting Logic:**

PDBs count "healthy" Pods — those passing readiness probes. A Pod that's not Ready isn't counted as available even if it's not technically unavailable.

This can cause issues: if some Pods are unhealthy and you also have a strict PDB, evictions block until those Pods become healthy or are removed.

**Status Fields:**

```yaml
status:
  currentHealthy: 8
  desiredHealthy: 8
  disruptionsAllowed: 0
  expectedPods: 10
```

`disruptionsAllowed: 0` means no more disruptions allowed right now. Useful for debugging stuck drains.

**Common Pitfalls:**

1. **PDB with single-replica workload** — effectively blocks all evictions
2. **PDB minAvailable equals replicas** — same problem; no headroom for disruption
3. **PDB on non-replicated objects** — pointless
4. **Forgetting PDB when using Cluster Autoscaler** — autoscaler can't downsize because evictions blocked
5. **PDB labels not matching** — silent failure; PDB doesn't apply

**Best Practices:**

1. **Set PDBs for production workloads** with at least 2 replicas
2. **Use percentages for autoscaling workloads** — adapt to replica count changes
3. **Don't set minAvailable = replicas** — leaves no room for disruption
4. **Test PDBs by draining nodes** in staging
5. **Monitor for blocked drains** — alert if `disruptionsAllowed` stays at 0

---

### 41. Explain RuntimeClass in Kubernetes.

RuntimeClass is a feature that allows **selecting different container runtime configurations** for different workloads in the same cluster. It enables running specialized runtimes (Kata Containers, gVisor, Firecracker) alongside the default runtime.

**The Problem It Solves:**

In a cluster, you might want different isolation levels for different workloads:
- Untrusted code (multi-tenant SaaS): strong isolation via Kata or gVisor
- Production services: standard runc for performance
- GPU workloads: NVIDIA runtime
- Windows containers: Windows runtime

Before RuntimeClass, this required separate clusters or complex node-level configuration. RuntimeClass makes runtime selection a Pod-level decision.

**How It Works:**

A RuntimeClass is a cluster-scoped object that maps a friendly name to a container runtime handler. The kubelet has been configured to know about runtime handlers via the CRI.

**Defining a RuntimeClass:**

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc       # the handler name configured in containerd/CRI-O
```

The `handler` must match what the runtime is registered as in the CRI runtime configuration.

**Using a RuntimeClass:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: untrusted-pod
spec:
  runtimeClassName: gvisor
  containers:
  - name: app
    image: my-untrusted-app:v1
```

The kubelet passes the runtime handler to the CRI when creating the Pod sandbox. The CRI uses the appropriate runtime to run the containers.

**Configuration at the CRI Layer:**

For containerd, you configure runtimes in `/etc/containerd/config.toml`:

```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  runtime_type = "io.containerd.runc.v2"

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runsc]
  runtime_type = "io.containerd.runsc.v1"

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.kata]
  runtime_type = "io.containerd.kata.v2"
```

Each Pod can then specify which runtime to use via its RuntimeClass.

**Common Runtimes Used with RuntimeClass:**

**runc (default):**
- Standard OCI runtime
- Direct kernel access via namespaces
- Best performance, weakest isolation

**gVisor (runsc):**
- User-space kernel implementation
- Intercepts syscalls and runs them in user space
- Better isolation, ~5-30% performance overhead
- Good for: untrusted code, multi-tenant scenarios

**Kata Containers:**
- Each container runs in a lightweight VM
- True hardware isolation
- Higher overhead but strong isolation
- Good for: high-security workloads, compliance requirements

**Firecracker:**
- AWS's microVM technology
- Designed for serverless (Lambda uses it)
- Fast boot, strong isolation
- Used by Fargate

**Windows Containers:**
- Process Isolation or Hyper-V Isolation
- Different RuntimeClasses for each

**NVIDIA Container Runtime:**
- Enables GPU access
- Configured for GPU workloads

**Scheduling Constraints:**

A RuntimeClass can specify node selection requirements:

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
scheduling:
  nodeSelector:
    runtime: gvisor
  tolerations:
  - key: runtime
    operator: Equal
    value: gvisor
    effect: NoSchedule
```

This ensures Pods using this RuntimeClass only land on nodes labeled appropriately. The scheduler injects these into Pods at scheduling time.

**Pod Overhead:**

Some runtimes (Kata, Firecracker) have significant overhead per Pod — the VM itself uses memory and CPU. RuntimeClass supports declaring this overhead:

```yaml
overhead:
  podFixed:
    cpu: "250m"
    memory: "120Mi"
```

The scheduler accounts for this when placing Pods, ensuring nodes have capacity not just for containers but also runtime overhead.

**Use Cases:**

**1. Multi-tenant SaaS:**
Untrusted user code runs with gVisor for syscall-level isolation, preventing container escape.

**2. PCI Compliance:**
Payment processing workloads run with Kata Containers for hardware-level isolation, meeting compliance requirements.

**3. Mixed Architecture Clusters:**
ARM and x86 nodes with different runtimes, RuntimeClass routes workloads appropriately.

**4. GPU Workloads:**
NVIDIA runtime via RuntimeClass for ML/AI workloads.

**5. Heterogeneous Workloads:**
Run Windows containers on Windows nodes, Linux on Linux nodes, all in one cluster.

**Practical Caveat:**

Most production clusters use one runtime (containerd or CRI-O with runc) and don't need RuntimeClass. It's a specialized feature. But for security-sensitive multi-tenant scenarios or specialized workloads, it's invaluable.

---

### 42. How does Kubernetes manage secrets internally?

Secrets in Kubernetes are objects designed to **hold sensitive data** — passwords, tokens, certificates, SSH keys. Understanding how they're stored, distributed, and consumed is critical for security.

**Internal Storage:**

Secrets are stored in **etcd**, the same as any other Kubernetes object. By default, **they're stored base64-encoded but not encrypted** — anyone with access to etcd can read them.

This is a critical security consideration:

1. **etcd encryption at rest** — must be explicitly enabled to actually encrypt secrets in etcd
2. **etcd backup files** contain decrypted (base64) secrets if encryption not enabled
3. **API server caches** secrets in memory

**Enabling Encryption at Rest:**

Configure the API server with an EncryptionConfiguration:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-32-byte-key>
  - identity: {}    # fallback - plain
```

With this, secrets are encrypted with AES-CBC before being written to etcd. The `identity` provider is the fallback for reading already-written objects, eventually replaced as objects are rewritten.

**Better: KMS Encryption:**

For production, use a KMS plugin (AWS KMS, Azure Key Vault, GCP KMS, HashiCorp Vault):

```yaml
providers:
- kms:
    name: my-kms
    endpoint: unix:///tmp/socketfile.sock
    cachesize: 100
    timeout: 3s
```

This:
- Generates a Data Encryption Key (DEK) per secret, encrypted by the KMS Key Encryption Key (KEK)
- Encrypted DEK stored with the secret in etcd
- Decryption requires KMS access
- Keys can be rotated externally
- Audit logs in the KMS show every decryption

**Secret Types:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  username: YWRtaW4=         # base64
  password: cGFzc3dvcmQ=     # base64
```

Built-in types:
- `Opaque` — arbitrary user-defined data (default)
- `kubernetes.io/service-account-token` — service account tokens
- `kubernetes.io/dockerconfigjson` — Docker registry credentials
- `kubernetes.io/tls` — TLS certificate and key
- `kubernetes.io/ssh-auth` — SSH credentials
- `kubernetes.io/basic-auth` — username/password
- `bootstrap.kubernetes.io/token` — bootstrap tokens

**How Secrets Are Distributed to Pods:**

When a Pod uses a Secret (via volume or env var):

1. Pod is scheduled to a node
2. kubelet on the node fetches the Secret from the API server
3. Secret is stored in **tmpfs** (RAM-backed filesystem) on the node — never written to disk
4. Mounted into the container at the specified path or injected as env vars
5. When the Pod is deleted, the tmpfs is unmounted, the data disappears

**Volume Mount:**

```yaml
volumes:
- name: secret-volume
  secret:
    secretName: my-secret
    defaultMode: 0400
containers:
- name: app
  volumeMounts:
  - name: secret-volume
    mountPath: /etc/secrets
    readOnly: true
```

Each key becomes a file: `/etc/secrets/username`, `/etc/secrets/password`. Updates to the Secret propagate to mounted volumes (with some delay, typically up to a minute).

**Environment Variables:**

```yaml
containers:
- name: app
  env:
  - name: USERNAME
    valueFrom:
      secretKeyRef:
        name: my-secret
        key: username
```

Caveat: env vars are **NOT updated** when the Secret changes. The container must restart to pick up new values.

**Why Volume Mounts Are Preferred:**

- Auto-update on Secret changes (no restart needed)
- Permissions can be tightly controlled (file modes)
- Secrets don't appear in `env` (less risk of accidental logging)
- Better integration with file-watching applications

**Security Concerns:**

1. **Anyone with read access to a Pod's namespace can read its Secrets** (via mounted volume or env). Use RBAC carefully.

2. **etcd encryption at rest is NOT enabled by default.** Always enable it for production.

3. **Secrets are NOT encrypted in transit** between API server and kubelet by default — secured by TLS but readable to API server admins.

4. **kubectl describe pod shows env var names but not values** — but you can `exec` into the Pod and read them.

5. **Memory dumps could expose Secrets** since they're in tmpfs.

**Service Account Tokens:**

ServiceAccounts have automatically-generated tokens that are projected into Pods. In modern Kubernetes (1.24+), these are **bound, time-limited, audience-aware** tokens:

```yaml
spec:
  serviceAccountName: my-sa
  # Token auto-mounted at /var/run/secrets/kubernetes.io/serviceaccount/token
```

The token is short-lived and refreshed automatically by kubelet — much more secure than the old model where tokens were stored as Secrets indefinitely.

**External Secret Management:**

For better secret management, many use external systems:

**HashiCorp Vault:**
- Pods authenticate to Vault using Kubernetes Service Account tokens
- Vault Agent (sidecar or init container) fetches secrets, writes to shared volume
- Secrets never stored in Kubernetes etcd

**External Secrets Operator:**
- Syncs secrets from external sources (Vault, AWS Secrets Manager, Azure Key Vault) into Kubernetes Secrets
- ExternalSecret CR specifies what to sync
- Operator maintains the sync continuously

**Sealed Secrets:**
- Encrypts Secrets so they can be stored in Git
- Sealed Secret operator decrypts to regular Secret in-cluster
- Solves "Secrets in version control" problem

**CSI Secret Store Drivers:**
- AWS Secrets and Configuration Provider, GCP Secret Manager Provider, Azure Key Vault Provider
- Secrets mounted as volumes from cloud secret managers directly
- Optionally synced to Kubernetes Secret for env var use

**Audit Logging:**

Enable audit logging to track Secret access:

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
  resources:
  - group: ""
    resources: ["secrets"]
```

This logs every Secret access. Critical for compliance and breach detection.

---

### 43. Explain projected volumes.

Projected volumes are a special volume type that **combines multiple volume sources into a single mount point**. They're particularly useful for service account tokens, ConfigMaps, Secrets, and downward API data.

**Why They Exist:**

Without projected volumes, mounting multiple types of data required multiple volumes:

```yaml
volumes:
- name: config
  configMap:
    name: app-config
- name: secrets
  secret:
    secretName: app-secrets
- name: pod-info
  downwardAPI:
    items:
    - path: pod-name
      fieldRef:
        fieldPath: metadata.name
```

This creates three separate mount points. Projected volumes consolidate them.

**Anatomy:**

```yaml
volumes:
- name: combined
  projected:
    sources:
    - configMap:
        name: app-config
        items:
        - key: config.yaml
          path: config.yaml
    - secret:
        name: app-secrets
        items:
        - key: db-password
          path: secrets/db-password
    - downwardAPI:
        items:
        - path: pod-name
          fieldRef:
            fieldPath: metadata.name
    - serviceAccountToken:
        path: token
        expirationSeconds: 3600
        audience: vault
```

All four sources are mounted at the same mountPath. Files are organized by their `path` field, allowing subdirectories.

**Supported Sources:**

- **configMap** — files from a ConfigMap
- **secret** — files from a Secret
- **downwardAPI** — Pod metadata (name, namespace, labels, annotations, resource limits)
- **serviceAccountToken** — projected service account token (modern bound tokens)
- **clusterTrustBundle** (alpha) — cluster trust bundles for CA certificates

**The Killer Feature: Projected Service Account Tokens**

This is the most important use of projected volumes — it's how modern Kubernetes (1.24+) handles service account tokens.

**Old Token Model (deprecated):**
- ServiceAccount auto-generated a long-lived Secret
- Secret was mounted into Pods
- Token never expired, never bound to a specific Pod or audience
- Security nightmare — token leaks were catastrophic

**New Bound Token Model:**

```yaml
volumes:
- name: token-volume
  projected:
    sources:
    - serviceAccountToken:
        path: token
        expirationSeconds: 3600
        audience: my-service
```

Properties:
- **Time-bound** — expires after `expirationSeconds` (default 1 hour, max 24 hours)
- **Audience-bound** — token includes `aud` claim, validated by the consumer
- **Pod-bound** — token is invalidated if the Pod is deleted
- **Auto-rotated** — kubelet refreshes the token before expiration

This is dramatically more secure. A leaked token is only valid for a short time and only for one specific audience.

**Use Cases for Projected Tokens:**

**1. Authenticating to Vault:**
Pod gets a projected token with audience `vault`. Vault validates the token and exchanges it for Vault credentials. Token can't be used for anything except Vault.

**2. AWS IRSA (IAM Roles for Service Accounts):**
Pod gets a projected token with audience `sts.amazonaws.com`. AWS STS exchanges it for AWS credentials. Used for EKS workloads to access AWS resources.

**3. GCP Workload Identity:**
Similar pattern — projected token with GCP audience, exchanged for GCP credentials.

**4. Internal Service Authentication:**
Microservices can mutually authenticate using projected tokens with each service's audience.

**Configuration in Practice:**

EKS IRSA configures Pods automatically when you annotate the ServiceAccount:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-sa
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/MyRole
```

EKS's webhook adds the projected token volume to Pods automatically. The Pod gets the token, presents it to AWS STS, and assumes the role. No long-lived AWS credentials stored anywhere.

**Default Behavior in Modern Clusters:**

Since v1.21, the default service account token mount is a projected volume:

```yaml
# Auto-injected into every Pod
volumeMounts:
- name: kube-api-access-xxxxx
  readOnly: true
  mountPath: /var/run/secrets/kubernetes.io/serviceaccount
volumes:
- name: kube-api-access-xxxxx
  projected:
    sources:
    - serviceAccountToken:
        expirationSeconds: 3607
        path: token
    - configMap:
        items:
        - key: ca.crt
          path: ca.crt
        name: kube-root-ca.crt
    - downwardAPI:
        items:
        - path: namespace
          fieldRef:
            fieldPath: metadata.namespace
```

This single projected volume includes:
- The bound token (refreshed every hour)
- The cluster CA certificate (from ConfigMap)
- The Pod's namespace (from downward API)

All mounted at `/var/run/secrets/kubernetes.io/serviceaccount/`.

**Benefits Over Separate Volumes:**

1. **Atomic updates** — when any source updates, the projected volume is updated as a whole
2. **Single mount point** — cleaner Pod spec, easier to manage
3. **Path control** — fine-grained control over where each file appears
4. **Performance** — kubelet manages one volume instead of multiple

---

### 44. Difference between ConfigMap and Secret.

ConfigMaps and Secrets are both objects that **inject configuration into Pods**, but they serve different purposes and have different security characteristics.

**Surface Similarities:**

Both:
- Store key-value data
- Can be mounted as volumes or injected as environment variables
- Are namespace-scoped
- Support updates without Pod restart (for volume mounts)
- Are referenced by Pods via name

**Key Differences:**

| Aspect | ConfigMap | Secret |
|--------|-----------|--------|
| Purpose | Non-sensitive config | Sensitive data |
| Storage encoding | Plain text | Base64 (NOT encrypted) |
| etcd encryption support | No (just plain) | Yes (when enabled) |
| Volume backing | Regular fs | tmpfs (RAM-backed) |
| Size limit | 1MB | 1MB |
| Mount permissions | Default | Configurable (defaultMode 0644) |
| Use cases | App config, env settings | Passwords, tokens, certs |

**ConfigMap Example:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database_url: "postgres://db:5432/myapp"
  log_level: "info"
  config.yaml: |
    server:
      port: 8080
    cache:
      ttl: 3600
binaryData:
  certificate.der: <base64-binary>
```

`data` holds string values; `binaryData` holds binary (base64-encoded). The `config.yaml` example uses YAML's literal block scalar (`|`) to embed multi-line content as a single key.

**Secret Example:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  username: YWRtaW4=                    # base64 of "admin"
  password: c3VwZXJzZWNyZXQ=            # base64 of "supersecret"
stringData:                             # convenience: provide plain text, k8s base64 encodes
  api-key: my-plain-api-key
```

`data` requires base64; `stringData` is a convenience field that takes plain text and Kubernetes encodes it.

**Mounting as Volumes:**

Both support similar volume syntax:

```yaml
# ConfigMap
volumes:
- name: config
  configMap:
    name: app-config
    items:                              # selectively project keys
    - key: config.yaml
      path: app/config.yaml
      mode: 0644

# Secret
volumes:
- name: secrets
  secret:
    secretName: db-secret
    defaultMode: 0400                   # restrict permissions
    items:
    - key: password
      path: db/password
```

**Environment Variables:**

```yaml
# ConfigMap
env:
- name: LOG_LEVEL
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: log_level

# Bulk import from ConfigMap
envFrom:
- configMapRef:
    name: app-config

# Secret
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: password

# Bulk import from Secret
envFrom:
- secretRef:
    name: db-secret
```

**Update Behavior:**

When a ConfigMap or Secret is updated:
- **Volume mounts** — files are updated automatically (with delay, typically up to a minute, due to kubelet sync period)
- **Environment variables** — NOT updated; Pod must restart

This is why volume mounts are preferred for config that might change.

**Immutability:**

Since v1.21, both ConfigMaps and Secrets support immutability:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
immutable: true
data:
  config: value
```

Immutable objects:
- Can't be updated (only deleted/recreated)
- Reduce kubelet load (no watches needed)
- Provide guarantee against accidental config drift
- Useful for versioned configs (config-v1, config-v2)

**Size Limits:**

Both are limited to **1 MB**. For larger data, use:
- **Volumes from PersistentVolumes**
- **Init containers** that download larger files
- **External configuration services** (etcd directly, Consul, etc.)

**When to Use Which:**

**Use ConfigMap for:**
- Application settings (log levels, feature flags)
- Connection strings (without credentials)
- Config files (yaml, json, ini, properties)
- Public certificates (CA bundles)
- Command-line arguments

**Use Secret for:**
- Passwords, API keys, tokens
- Private keys (TLS, SSH)
- Database credentials
- Anything you'd be uncomfortable seeing in logs/diffs/git

**Real-World Pattern: Splitting Config**

A common pattern is to have both:

```yaml
# Non-sensitive config in ConfigMap
data:
  database_host: "db.example.com"
  database_port: "5432"
  database_name: "myapp"
  pool_size: "20"

# Sensitive in Secret
data:
  database_username: <base64>
  database_password: <base64>
```

Inject both into the Pod. The Pod assembles the connection string. This way:
- ConfigMap can be in version control (no secrets)
- Secret can be from Vault or other secret manager

**Why Secrets Aren't Truly Secret:**

The "Secret" name is somewhat misleading because:

1. **Base64 is encoding, not encryption** — anyone with read access can decode
2. **etcd is unencrypted by default** — admins can read all Secrets
3. **kubectl get secret -o yaml** shows the base64-encoded value (trivially decoded)
4. **Anyone with read access to a namespace** can read its Secrets
5. **API server logs and audit logs** might capture Secret content

Real secrets management requires:
- etcd encryption at rest (or KMS)
- Strict RBAC on Secret resources
- Use of external secret managers (Vault, AWS Secrets Manager) for the most sensitive data
- Audit logging
- Network policies preventing exfiltration

---

### 45. Explain namespace isolation.

Namespaces are Kubernetes' primary mechanism for **logical separation of resources** within a cluster. But "isolation" is a nuanced concept — namespaces provide name isolation but not security or resource isolation by default.

**What Namespaces Actually Do:**

1. **Name scoping** — `nginx` in namespace A and `nginx` in namespace B are different objects
2. **Default resource visibility** — `kubectl get pods` shows only the current namespace
3. **DNS scoping** — Services are addressable as `svc.namespace.svc.cluster.local`
4. **RBAC boundary** — permissions can be granted per-namespace
5. **Quota enforcement** — ResourceQuotas apply per-namespace
6. **NetworkPolicy boundary** — policies can target namespaces
7. **Default ServiceAccount** — each namespace has its own

**What Namespaces DO NOT Do (Out of the Box):**

1. **Network isolation** — Pods in different namespaces can talk to each other by default
2. **Resource isolation** — without quotas, one namespace can consume all cluster resources
3. **Storage isolation** — PVs are cluster-scoped (only PVCs are namespaced)
4. **Node isolation** — Pods from any namespace can land on any node
5. **Strong security boundary** — a compromised Pod can still potentially access other namespaces if RBAC allows

**Default Namespaces:**

- **default** — where objects go if no namespace specified
- **kube-system** — system components (CoreDNS, kube-proxy if as DaemonSet, etc.)
- **kube-public** — readable by all users (including unauthenticated); used for cluster info
- **kube-node-lease** — node lease objects for heartbeats

**Cluster-Scoped vs Namespace-Scoped Resources:**

**Namespaced** (most things):
- Pods, Services, Deployments, ConfigMaps, Secrets, PVCs, Ingresses, NetworkPolicies, Roles, RoleBindings...

**Cluster-scoped**:
- Nodes, PersistentVolumes, StorageClasses, ClusterRoles, ClusterRoleBindings, Namespaces themselves, CustomResourceDefinitions, IngressClasses

Cluster-scoped resources can't be "isolated" by namespace — they exist outside any namespace.

**Achieving True Isolation:**

To make namespaces a real security boundary, layer multiple mechanisms:

**1. RBAC:**

Restrict permissions per namespace:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team-access
  namespace: team-a
subjects:
- kind: User
  name: alice
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
```

Alice can only edit resources in `team-a` namespace. This is the most fundamental isolation.

**2. NetworkPolicies:**

Block cross-namespace traffic by default:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-from-other-namespaces
  namespace: team-a
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}  # only from same namespace
```

This blocks ingress from other namespaces unless explicitly allowed.

**3. ResourceQuotas:**

Cap resource consumption per namespace:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "40Gi"
    pods: "100"
```

**4. LimitRanges:**

Force per-container resource boundaries within the namespace:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: defaults
  namespace: team-a
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "512Mi"
    max:
      cpu: "4"
      memory: "8Gi"
    min:
      cpu: "10m"
      memory: "10Mi"
```

**5. Pod Security Standards:**

Enforce security policies per namespace:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: team-a
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

This forces all Pods in `team-a` to comply with the restricted Pod Security Standard — no privileged Pods, no hostNetwork, runs as non-root, etc.

**6. Node Isolation (via taints/affinity):**

Reserve nodes for specific namespaces:

```bash
# Taint nodes for team-a
kubectl taint nodes team-a-node-1 dedicated=team-a:NoSchedule
kubectl label nodes team-a-node-1 dedicated=team-a
```

Then add admission controllers or webhooks that inject tolerations and node affinity based on the Pod's namespace.

**Hard Multi-Tenancy:**

For true multi-tenant isolation (each tenant gets a cluster-like experience), even all of the above isn't enough. Real multi-tenancy requires either:

**Virtual Clusters (vCluster):**
- Each tenant gets a virtual control plane inside the host cluster
- Tenants get their own API server, their own resources
- Strong isolation, looks like separate clusters
- Some performance overhead

**Hierarchical Namespaces (HNC):**
- Parent-child namespace relationships
- Inherited policies, quotas
- Better delegation model

**Separate Clusters:**
- Ultimate isolation
- Higher operational cost
- Often used for prod/staging/dev separation, sensitive workloads

**Common Use Cases for Namespaces:**

1. **Environment separation** — dev, staging, prod
2. **Team separation** — team-a, team-b, team-c
3. **Application separation** — app1, app2, app3
4. **System vs user workloads** — kube-system, monitoring, logging vs app namespaces

**Anti-Patterns:**

- **One namespace for everything** — doesn't scale; everyone has access to everything
- **Namespace per Pod** — overkill; namespaces are for groups
- **Treating namespaces as security boundaries without additional controls** — they're not, by default
- **Cross-namespace dependencies without explicit policies** — fragile

**Best Practices:**

- Use namespaces for **logical grouping** (teams, environments, apps)
- Layer **RBAC**, **NetworkPolicies**, **ResourceQuotas**, and **Pod Security Standards** for real isolation
- Don't put untrusted workloads in shared clusters; use separate clusters or vCluster
- Document the namespace structure and ownership
- Use **labels on namespaces** for organization (owner, cost-center, environment)

---

### 46. What are custom resources and CRDs?

Custom Resources (CRs) and Custom Resource Definitions (CRDs) are how Kubernetes is **extended with domain-specific objects**. They turn the Kubernetes API into a platform — you can define your own object types and the API server handles them just like built-in resources.

**The Distinction:**

- **CRD (CustomResourceDefinition)** — defines a new resource type, including its schema. Cluster-scoped.
- **CR (Custom Resource)** — an instance of a CRD-defined type. Behaves like any other Kubernetes object.

Think of it like classes vs objects: CRD is the class definition, CR is an instance.

**Why They Exist:**

Kubernetes ships with built-in resources (Pod, Service, Deployment, etc.). But applications and operators need to model their own domain concepts:
- Database operators want a `PostgresCluster` resource
- Cert-manager wants `Certificate` and `Issuer` resources
- Istio wants `VirtualService` and `DestinationRule`
- Argo wants `Application` and `Workflow`

Rather than each tool having its own API, CRDs let them register custom types directly with the Kubernetes API server. Users interact with them just like native resources.

**Anatomy of a CRD:**

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: backups.example.com    # must be plural.group
spec:
  group: example.com
  versions:
  - name: v1
    served: true               # accessible via API
    storage: true              # stored in etcd (only one version can be storage)
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required: ["source", "destination"]
            properties:
              source:
                type: string
              destination:
                type: string
              schedule:
                type: string
                pattern: '^([0-9*/,-]+\s+){4}[0-9*/,-]+$'
              retention:
                type: integer
                minimum: 1
                maximum: 365
                default: 30
          status:
            type: object
            properties:
              phase:
                type: string
                enum: ["Pending", "Running", "Succeeded", "Failed"]
              lastBackupTime:
                type: string
                format: date-time
    subresources:
      status: {}              # enable /status subresource
      scale:                  # enable /scale subresource
        specReplicasPath: .spec.replicas
        statusReplicasPath: .status.replicas
    additionalPrinterColumns:
    - name: Source
      type: string
      jsonPath: .spec.source
    - name: Phase
      type: string
      jsonPath: .status.phase
    - name: Age
      type: date
      jsonPath: .metadata.creationTimestamp
  scope: Namespaced            # or Cluster
  names:
    plural: backups
    singular: backup
    kind: Backup
    shortNames:
    - bk
    listKind: BackupList
    categories:
    - all                     # included in `kubectl get all`
```

**Using the Custom Resource:**

Once the CRD is created:

```yaml
apiVersion: example.com/v1
kind: Backup
metadata:
  name: db-backup-daily
  namespace: production
spec:
  source: postgres://prod-db
  destination: s3://backups/prod
  schedule: "0 2 * * *"
  retention: 90
```

```bash
kubectl apply -f backup.yaml
kubectl get backups
kubectl get bk                 # using short name
kubectl describe backup db-backup-daily
```

**Schema Validation:**

The OpenAPI v3 schema enables:
- **Type checking** — wrong types rejected
- **Required fields** — missing required fields rejected
- **Patterns** — regex validation
- **Enums** — only allowed values
- **Min/max** — value bounds
- **Defaults** — values filled in if not provided

This validation happens at the API server, like for built-in resources.

**Subresources:**

**status subresource** — separates spec (user-controlled) from status (controller-controlled). Updating one doesn't affect the other. Critical for operators.

**scale subresource** — enables `kubectl scale` and HPA to work with the CR. Maps spec.replicas and status.replicas to specific JSON paths.

**Additional Printer Columns:**

Customize what `kubectl get` shows for the resource. Without these, only Name and Age are shown.

**Versioning:**

CRDs can have multiple versions:

```yaml
versions:
- name: v1alpha1
  served: true
  storage: false
- name: v1beta1
  served: true
  storage: false
- name: v1
  served: true
  storage: true                # only one storage version
```

The cluster can serve multiple versions, but only one is stored in etcd. **Conversion webhooks** translate between versions on read/write.

This enables evolution — start with v1alpha1, graduate to v1beta1, finally v1. Old clients still work via conversion.

**Scope:**

- **Namespaced** — CRs belong to a namespace, like Pods or Services
- **Cluster** — CRs are cluster-wide, like Nodes or ClusterRoles

**Categories:**

Categories let you include CRs in convenience groupings:

```yaml
categories:
- all
- monitoring
```

Then `kubectl get all` and `kubectl get monitoring` include this resource.

**Limitations of Vanilla CRDs:**

CRDs alone are just **declarative API objects**. By themselves, they don't DO anything — they're stored in etcd and you can CRUD them. To make them functional, you need a **controller** (operator) that watches the CRs and takes action.

This is why CRDs and operators almost always go together. CRDs without operators are just structured data; operators without CRDs can only manage built-in resources.

**Other Extension Mechanisms:**

CRDs aren't the only way to extend Kubernetes:

**Aggregated API Server:**
A separate API server that the main API server proxies to. More powerful (custom storage, custom logic) but much more complex. Used for things like metrics-server.

**Admission Webhooks:**
Don't add resources but modify/validate existing ones. Often used alongside CRDs.

**Custom Schedulers:**
Replace or supplement the default scheduler with custom logic.

For most use cases, CRDs are the right answer.

---

### 47. Explain Operator pattern.

The Operator pattern is a Kubernetes-specific approach to **encoding human operational knowledge into software**. It combines CRDs (to define domain objects) with controllers (to implement domain logic), creating a system that manages complex applications the way a skilled human operator would.

**The Origin:**

Coined by CoreOS in 2016, the term comes from the idea: "What does a database operator do? They take backups, do failovers, scale, upgrade, troubleshoot. Can we put that knowledge in software?"

The answer was: yes, by combining:
1. **CRDs** to declaratively define what we want (a PostgresCluster with 3 replicas, daily backups, etc.)
2. **Custom controllers** to continuously make it so (reconcile loops that handle all the operational details)

**Anatomy of an Operator:**

An operator consists of:

1. **CRDs** — define the resources the operator manages (e.g., PostgresCluster, PostgresBackup, PostgresUser)

2. **Controller(s)** — reconciliation loops that watch the CRs and take action

3. **Operational logic** — knowledge of how to operate the application:
   - How to deploy
   - How to upgrade safely
   - How to back up and restore
   - How to scale
   - How to handle failures
   - How to monitor
   - How to expose metrics

4. **Operator deployment manifests** — Deployment, RBAC, ServiceAccount for the operator itself

**Example: A Hypothetical Postgres Operator**

User creates:
```yaml
apiVersion: postgres.example.com/v1
kind: PostgresCluster
metadata:
  name: production-db
spec:
  version: "15"
  replicas: 3
  storage:
    size: 100Gi
    storageClass: fast-ssd
  backup:
    schedule: "0 2 * * *"
    retention: 30
    destination: s3://backups/prod-db
  monitoring:
    enabled: true
```

The operator's reconcile loop:
1. Sees the new PostgresCluster
2. Creates a StatefulSet with 3 PostgreSQL replicas
3. Configures streaming replication
4. Creates Services (read-write to primary, read-only to replicas)
5. Sets up automatic backups (CronJob to S3)
6. Deploys monitoring sidecars (postgres_exporter)
7. Creates Secrets for database credentials
8. Sets up cert-manager Certificate for TLS
9. Configures PodDisruptionBudget
10. Watches for primary failure → automatic failover
11. Handles version upgrades — minor versions via rolling, major via dump/restore
12. Continuously monitors all of this

The user just declared "I want a Postgres cluster." The operator handles every detail.

**The Operator Maturity Model:**

CoreOS defined 5 levels:

**Level 1 — Basic Install:**
The operator deploys the application. CRD spec includes basic config; CRD becomes a Deployment.

**Level 2 — Seamless Upgrades:**
The operator handles updates. Change the version in spec; operator orchestrates the upgrade.

**Level 3 — Full Lifecycle:**
Backup, restore, failure recovery. The operator handles operational events.

**Level 4 — Deep Insights:**
Metrics, alerts, log analysis. The operator exposes detailed monitoring.

**Level 5 — Auto Pilot:**
Auto-scaling, auto-healing, auto-tuning, capacity planning. The operator continuously optimizes.

Most production operators are Level 2-3. Level 5 is aspirational.

**Building Operators:**

Several frameworks make operator development tractable:

**Operator SDK:**
- Multiple language bindings (Go, Ansible, Helm)
- Scaffolds the project, generates CRDs from Go structs
- Includes testing utilities

**Kubebuilder:**
- Go-specific framework
- Maintained by Kubernetes SIG API Machinery
- Generates code from Go type definitions
- Strong patterns for controller logic

**Kopf (Python):**
- Python-native framework
- Decorator-based
- Good for quick prototypes or Python-heavy environments

**KubeBuilder vs Operator SDK:**
- Operator SDK supports more languages
- Kubebuilder is more focused on Go
- They share underlying libraries (controller-runtime)
- Most new Go operators use Kubebuilder directly

**Anatomy of a Controller (Go):**

```go
func (r *PostgresClusterReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 1. Fetch the PostgresCluster
    var cluster postgresv1.PostgresCluster
    if err := r.Get(ctx, req.NamespacedName, &cluster); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 2. Handle finalizer for cleanup
    if cluster.DeletionTimestamp != nil {
        return r.handleDeletion(ctx, &cluster)
    }

    // 3. Ensure desired state
    if err := r.reconcileStatefulSet(ctx, &cluster); err != nil {
        return ctrl.Result{}, err
    }
    if err := r.reconcileServices(ctx, &cluster); err != nil {
        return ctrl.Result{}, err
    }
    if err := r.reconcileBackupSchedule(ctx, &cluster); err != nil {
        return ctrl.Result{}, err
    }

    // 4. Update status
    cluster.Status.Phase = computePhase(&cluster)
    if err := r.Status().Update(ctx, &cluster); err != nil {
        return ctrl.Result{}, err
    }

    return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
}
```

The reconcile function is called whenever any watched resource changes (the PostgresCluster, or its child resources like StatefulSets).

**Popular Production Operators:**

- **Prometheus Operator** — manages Prometheus, Alertmanager, ServiceMonitor, etc.
- **cert-manager** — manages TLS certificates
- **Istio operator** — manages service mesh
- **Strimzi** — Kafka operator
- **Crunchy Data PostgreSQL Operator** — production PostgreSQL
- **Argo CD** — GitOps controller (sort of an operator)
- **External Secrets Operator** — syncs from external secret stores
- **OpenEBS, Rook, Longhorn** — storage operators
- **Cluster API** — operator for managing Kubernetes clusters themselves

**OperatorHub:**

OperatorHub.io is a community catalog of operators, with the Operator Lifecycle Manager (OLM) providing installation and lifecycle management.

**Best Practices:**

1. **Single Responsibility** — each operator manages one application well
2. **Idempotency** — reconcile multiple times produces the same result
3. **Watch what you create** — set ownerReferences for garbage collection
4. **Status conditions** — use standard condition pattern (Ready, Progressing, Degraded)
5. **Events** — emit Kubernetes events for observability
6. **Metrics** — expose Prometheus metrics
7. **RBAC minimum** — operator should have only the permissions it needs
8. **Handle deletion** — finalizers for cleanup
9. **Version your CRDs** — plan for evolution
10. **Test reconciliation** — unit tests for the controller logic

---

### 48. Difference between Helm and Operators.

Both Helm and Operators are ways to package and deploy applications in Kubernetes, but they take fundamentally different approaches. Understanding when to use which is crucial.

**Helm: Package Manager Approach**

Helm treats Kubernetes applications like **software packages**. You have:
- **Charts** — templated YAML manifests parameterized with values
- **Helm CLI** — installs, upgrades, rollbacks charts
- **Releases** — installed instances of charts

A Helm chart contains:
```
mychart/
├── Chart.yaml              # metadata
├── values.yaml             # default config values
├── templates/              # YAML templates (using Go templating)
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
└── charts/                 # dependencies
```

You install with: `helm install my-app ./mychart --values myconfig.yaml`

Helm renders the templates with your values and applies the resulting YAML to Kubernetes.

**Operator: Application-Aware Approach**

An operator is a **running program in the cluster** that continuously manages an application using a CR as its API.

Components:
- **CRD** — defines the API
- **Controller** — runs as a Deployment, watches CRs, manages app
- **CR** — user creates this to declare desired state

You install with: `kubectl apply -f cr.yaml` (after the operator itself is installed)

**Core Philosophical Difference:**

**Helm is push-based and templating-focused:**
- Render templates → apply → done
- Day 1 operations (initial install)
- Templating limits — complex logic is hard
- Once applied, Helm doesn't manage runtime; standard controllers (Deployment, etc.) take over

**Operators are pull-based and continuously reconciling:**
- Define desired state → operator continuously enforces it
- Day 2 operations (backups, upgrades, healing) are first-class
- Logic is real code (Go, Python, etc.)
- Operator runs forever, watching and acting

**Comparison Table:**

| Aspect | Helm | Operator |
|--------|------|----------|
| Mental model | Package manager | Application-aware controller |
| When it runs | Only during install/upgrade | Continuously |
| Logic complexity | Limited (templates) | Unlimited (real code) |
| Day 1 (install) | Excellent | Good |
| Day 2 (operate) | Limited | Excellent |
| Versioning | Chart versions | CRD versions + operator versions |
| State tracking | Helm releases | CR status |
| Customization | Values.yaml | CR spec |
| Learning curve | Easy | Steep (need to write controllers) |
| Cluster overhead | Zero (Helm runs client-side) | Operator Pod runs always |
| Best for | Stateless apps, simple deployments | Stateful, complex apps |

**When to Use Helm:**

- Stateless web apps with standard deployment patterns
- Tools and dependencies (databases like Postgres in dev, monitoring stacks, etc.)
- Multi-environment deployments where you want value-based customization
- One-shot installation/upgrade scenarios
- When community charts exist for what you need
- Internal apps where Day 2 ops are handled by humans/CI

Examples: deploying nginx-ingress, prometheus (basic), redis (basic), your own microservices

**When to Use Operators:**

- Complex stateful applications (databases, message queues, distributed storage)
- Apps requiring sophisticated lifecycle management (failover, backup, recovery)
- Apps where domain knowledge matters (how to safely upgrade a database)
- When you need a custom API for users to declare app intent
- Cross-cutting concerns (cert management, secrets sync, GitOps)
- Multi-cluster orchestration

Examples: Kafka, Postgres production, Cassandra, Elasticsearch, complex ML pipelines

**The Reality: They Can Coexist**

Many operators are **distributed via Helm**:
```bash
helm install prometheus-operator prometheus-community/kube-prometheus-stack
```

The Helm chart installs the operator (Deployment, RBAC, CRDs). Then users create CRs that the operator manages. Best of both worlds.

**Hybrid Patterns:**

**Helm + Custom Resources:**
Use Helm to deploy your operator and initial CRs. Operator takes over for Day 2.

**Helm Operator (older approach):**
Some operators are themselves Helm charts wrapped in an operator framework. The operator just calls `helm install` on changes. Limited but easy.

**Helm Hooks for Lifecycle:**
Helm hooks (pre-install, pre-upgrade, post-install, etc.) can run Jobs that handle some operational tasks. But these are one-shot, not continuous.

**Strengths and Weaknesses:**

**Helm Strengths:**
- Mature ecosystem, huge community
- Easy to write
- No runtime overhead
- Well-understood by ops teams
- Excellent for templating across environments

**Helm Weaknesses:**
- Day 2 ops are not its concern
- Template complexity gets ugly fast
- No reconciliation — once applied, drift isn't corrected
- Limited error handling
- No knowledge of app internals

**Operator Strengths:**
- True application awareness
- Continuous reconciliation
- Real code, real logic
- Day 2 ops automated
- API-driven (cluster admins extend Kubernetes)

**Operator Weaknesses:**
- Complex to develop and maintain
- Runtime overhead (operator Pod always running)
- Higher learning curve
- Many bad operators exist; quality varies
- Coupling between cluster and app lifecycle

**Decision Heuristic:**

Ask yourself:
1. Does this app need Day 2 ops automation? → Operator
2. Is this a stateless or simple stateful app? → Helm
3. Do I want a custom API (CR) for users? → Operator
4. Is this a tool/dependency I just need installed? → Helm
5. Is there a community operator I can use? → Use the operator
6. Is there only a Helm chart available? → Use Helm

**Modern Approaches:**

- **Kustomize** — YAML overlays, no templating, simpler than Helm for some cases
- **GitOps tools (Argo CD, Flux)** — continuously sync git → cluster, work with both Helm and raw YAML
- **Crossplane** — manages cloud infrastructure via Kubernetes CRDs and operators

---

### 49. Explain API aggregation layer.

The API aggregation layer is a mechanism for **extending the Kubernetes API server** by registering additional API servers that handle specific resources. Unlike CRDs (which add resource types to the main API server), aggregated APIs run as separate processes that the main API server proxies to.

**The Architecture:**

```
Client (kubectl, controllers)
       ↓
   kube-apiserver  ←—— authenticates, authorizes, then routes
       ↓
   ┌────┴────┬─────────┬──────────┐
   ↓         ↓         ↓          ↓
 core API  CRDs   metrics-server  custom aggregated API
                  (extension)     (extension)
```

The main API server doesn't handle all requests itself. For paths registered as aggregated APIs, it transparently proxies to the extension API server.

**Why It Exists:**

CRDs are great but have limitations:
- All data stored in main etcd
- Limited validation (just OpenAPI schemas)
- No custom storage backends
- No custom business logic at the storage layer
- Can't implement subresources beyond status/scale

For some use cases, you need a real API server. Examples:
- **metrics-server** — stores metrics in memory (not etcd), high turnover, doesn't need durability
- **Custom authentication backends**
- **Complex validation logic**
- **Integration with non-etcd storage**

**How It Works:**

You deploy an extension API server (just a regular Kubernetes Service running a binary that speaks the Kubernetes API).

You register it via APIService:

```yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1beta1.metrics.k8s.io
spec:
  service:
    namespace: kube-system
    name: metrics-server
    port: 443
  group: metrics.k8s.io
  version: v1beta1
  insecureSkipTLSVerify: false
  caBundle: <base64-encoded-CA>
  groupPriorityMinimum: 100
  versionPriority: 100
```

This tells the main API server: "When you receive requests for paths in the `metrics.k8s.io/v1beta1` group, proxy them to the `metrics-server` Service."

**Request Flow:**

A request like `kubectl top nodes`:

1. kubectl issues GET to `/apis/metrics.k8s.io/v1beta1/nodes`
2. Main API server receives request, performs authentication
3. Sees the path matches an APIService registration
4. Performs authorization (using main API server's RBAC) — checks if user has access to `nodes.metrics.k8s.io`
5. Proxies the (authenticated and authorized) request to metrics-server
6. metrics-server returns response
7. Main API server forwards response to client

The client doesn't know it's talking to a different server — the proxy is transparent.

**Key Components for Extension API Servers:**

To build an aggregated API server, your code must:

1. **Speak the Kubernetes API protocol** — JSON/Protobuf serialization, resource paths, watch streams, etc.
2. **Validate requests** — own validation logic
3. **Store data somehow** — your own etcd, an external database, in-memory, whatever
4. **Implement watch** — clients use long-lived watches for changes
5. **Handle authentication** — trust the main API server's authentication (verify with cluster CA)
6. **Handle authorization** — typically delegate to main API server via SubjectAccessReview

The Kubernetes project provides the **kube-aggregator** library to make this easier.

**Real Examples:**

**metrics-server:**
- Stores resource usage metrics (CPU, memory) in memory
- Used by `kubectl top` and HPA
- Doesn't persist data — recomputes on restart
- Registered as `metrics.k8s.io/v1beta1`

**custom-metrics adapter:**
- Exposes custom metrics for HPA
- Often backed by Prometheus
- Registered as `custom.metrics.k8s.io`

**external-metrics adapter:**
- Exposes external metrics for HPA
- Backed by cloud provider monitoring (CloudWatch, Stackdriver)
- Registered as `external.metrics.k8s.io`

**OpenShift's various aggregated APIs:**
- ImageStream, BuildConfig, Project — Red Hat OpenShift uses aggregation extensively

**CRDs vs Aggregated APIs — When to Use Which:**

| Aspect | CRDs | Aggregated APIs |
|--------|------|-----------------|
| Storage | etcd (with main cluster) | Anywhere (your choice) |
| Schema | OpenAPI v3 | Code-defined |
| Logic | Operator (separate) | In API server |
| Subresources | status, scale | Anything |
| Versioning | Via conversion webhooks | Code-handled |
| Performance | etcd-bound | Independent |
| Complexity | Lower | Much higher |
| Operational overhead | Just the operator | Whole API server |
| Use case | Most extensions | Special needs |

**When to Use CRDs:**
- Almost always — they're simpler
- Operator-managed resources
- Standard CRUD patterns
- When etcd storage is fine

**When to Use Aggregated APIs:**
- Need non-etcd storage (metrics in memory, external DB)
- Need complex API logic that doesn't fit operator pattern
- Need custom validation beyond OpenAPI
- High-throughput resources where etcd would be a bottleneck
- Need custom subresources

**Complexity Cost:**

Building an aggregated API server is **much more work** than a CRD + operator. You need:
- Full API server scaffolding
- Schema generation
- Watch implementation
- Strategic merge patch
- Garbage collection integration
- Audit logging integration

This is why CRDs dominate — they're "good enough" for 95% of use cases.

**Discovery and Aggregation:**

The main API server periodically fetches OpenAPI specs from aggregated API servers, merging them into its own discovery document. This makes the extension transparent to clients — `kubectl get` knows about the extended resources.

**Authentication Flow Detail:**

A subtle but important detail: when the main API server proxies a request, it adds headers identifying the original requester:
- `X-Remote-User`
- `X-Remote-Group`
- `X-Remote-Extra-*`

The extension API server trusts these headers because the request comes from the main API server (verified via mTLS). It uses them for authorization decisions.

This is configured via the `--requestheader-*` flags on the main API server and matching configuration on extensions.

---

### 50. What are API deprecations and version skew policies?

Kubernetes evolves rapidly, and managing API changes safely is critical. The project has formal policies for **deprecations** (announcing future removals) and **version skew** (allowing controlled differences between component versions).

**API Versioning Levels:**

Kubernetes APIs have stability levels indicated by the version name:

**Alpha (v1alpha1, v1alpha2):**
- May be buggy
- May change incompatibly without notice
- May be disabled by default (require feature gate)
- Not recommended for production
- Disabled in most managed Kubernetes services

**Beta (v1beta1, v1beta2):**
- Well-tested, generally stable
- May change in incompatible ways but with notice
- Enabled by default
- Used widely in production despite the name
- New beta APIs since v1.20 are disabled by default (to discourage long-lived beta dependence)

**Stable (v1, v2):**
- Production-ready
- Backward compatible within the major version
- Long-term support guaranteed

**Deprecation Policy:**

Kubernetes has formal rules for how long features must be supported after deprecation:

**For Stable (GA) APIs:**
- **12 months or 3 releases** (whichever is longer) of deprecation notice before removal
- This gives a long migration window

**For Beta APIs:**
- **9 months or 3 releases** of deprecation
- Shorter than GA but still substantial

**For Alpha APIs:**
- Can be removed in any release with notice in the release notes

**The Lifecycle:**

A typical API journey:

1. **Alpha (v1alpha1)** — introduced, experimental, may change
2. **Beta (v1beta1)** — stabilized, widely tested
3. **Stable (v1)** — GA, long-term commitment
4. **Deprecated** — replaced by newer version
5. **Removed** — after deprecation period

Example: PodSecurityPolicy
- Introduced v1.8 (beta)
- Deprecated v1.21
- Removed v1.25
- Replaced by Pod Security Standards / Pod Security Admission

**Recent Major Deprecations:**

**Ingress (extensions/v1beta1, networking.k8s.io/v1beta1):**
- Deprecated v1.14
- Final stable v1 in 1.19
- v1beta1 removed in v1.22

**PodSecurityPolicy:**
- Deprecated v1.21
- Removed v1.25

**RBAC (rbac.authorization.k8s.io/v1beta1):**
- Deprecated v1.17
- Removed v1.22

**FlowSchema and PriorityLevelConfiguration:**
- Multiple beta versions
- v1 GA in v1.29

**Dockershim:**
- The CRI shim for Docker (not an API but worth noting)
- Deprecated v1.20
- Removed v1.24

**Version Skew Policy:**

Different Kubernetes components can run at different versions, within limits:

**kube-apiserver:**
- The newest component in a cluster
- In HA clusters, multiple apiservers must be within **one minor version** of each other

**kubelet:**
- Can be **up to 3 minor versions older** than kube-apiserver (since v1.28; was 2 before)
- e.g., apiserver v1.29 supports kubelets v1.26, v1.27, v1.28, v1.29
- Can never be **newer** than the apiserver

**kube-proxy:**
- Same skew as kubelet (3 minor versions)
- Should match kubelet on the same node

**kube-controller-manager, kube-scheduler, cloud-controller-manager:**
- Must be **equal to or one minor version older** than apiserver
- Tighter coupling because they're control plane components

**kubectl:**
- Supports one minor version older or newer than apiserver
- e.g., kubectl 1.28 works with apiservers 1.27, 1.28, 1.29

**Why This Matters for Upgrades:**

The skew policy dictates a safe upgrade order:

1. **Upgrade etcd** (if needed)
2. **Upgrade kube-apiserver** (one minor version at a time)
3. **Upgrade other control plane** (controller-manager, scheduler, cloud-controller-manager)
4. **Upgrade kubelets** (one minor version at a time)
5. **Upgrade kubectl** on operator machines

You can never:
- Upgrade kubelet ahead of apiserver
- Skip minor versions on apiserver (1.27 → 1.29 directly)

For multi-version skew (kubelets behind apiservers), you have flexibility — upgrade the control plane first, then upgrade kubelets later when convenient.

**Practical Implications:**

**Cluster Upgrades:**
You generally upgrade one minor version at a time:
- 1.26 → 1.27 → 1.28 → 1.29

Not directly 1.26 → 1.29. Each minor version has its own deprecations and changes.

**Managed Kubernetes (EKS, GKE, AKS):**
Cloud providers handle the control plane upgrade for you. Your responsibility:
- Upgrade node groups (which run kubelet)
- Update your manifests to remove deprecated APIs
- Test in staging before production

**API Deprecation in Practice:**

Kubernetes provides ways to detect deprecated API usage:

```bash
# Audit logs can show deprecated API usage
kubectl get --raw /apis | jq

# Check for objects using deprecated APIs
kubectl get deployments.extensions  # would show deprecated API usage
```

Tools like:
- **kubectl deprecations plugin** — scans manifests
- **Pluto** — detects deprecated APIs in manifests
- **kube-no-trouble (kubent)** — scans live cluster for deprecated API usage

**The Removal Test:**

Before removing an API:
- It must have been deprecated for the policy-required time
- It must have a stable successor
- The successor must have been available throughout the deprecation period
- Release notes must clearly document the change

**Storage Version Handling:**

When an API has multiple versions (v1beta1, v1), Kubernetes:
- Serves all enabled versions (read/write)
- Stores in **one storage version** (the latest stable)
- Auto-converts between versions

For CRDs, you can use conversion webhooks to handle complex transformations. For built-in APIs, the conversion logic is in the API server itself.

**What This Means for Operators:**

- **Test against new Kubernetes versions** before they become default
- **Monitor deprecation announcements** in release notes
- **Update manifests proactively** when APIs are deprecated
- **Use CI tooling** (Pluto, kubent) to catch deprecated APIs
- **Plan upgrades** in stages with version skew in mind
- **Don't skip minor versions** — upgrade incrementally

**Common Pitfalls:**

1. **Skipping minor versions** — not supported; you'll hit incompatibilities
2. **Upgrading kubelets ahead of apiservers** — violates skew policy
3. **Not updating manifests** when APIs are removed — apply will fail
4. **Long delay between minor version upgrades** — accumulates technical debt
5. **Forgetting CRDs and operators** — they may not support the new Kubernetes version

---
# Advanced Kubernetes Interview Questions - Part 5

## etcd, Scheduling Internals, Runtime, Node Management & Security

---

### 51. Explain etcd architecture and quorum.

etcd is a **distributed, consistent key-value store** that serves as the single source of truth for all Kubernetes cluster state. Its architecture is fundamental to Kubernetes' reliability — if etcd fails or becomes inconsistent, the entire cluster is affected.

**Core Architecture:**

etcd is typically deployed as a cluster of an **odd number of nodes** (3, 5, or 7). The odd number is critical for quorum-based consensus — with even numbers, you can have split-brain scenarios where no majority exists.

Each etcd node runs the same software and contains a full copy of the data. They communicate with each other to maintain consistency using the Raft consensus algorithm.

**Components Within Each etcd Node:**

**Storage Layer:**
- **WAL (Write-Ahead Log)** — every write is first appended to the WAL on disk before being applied. This provides durability — if etcd crashes mid-write, it can replay the WAL on restart.
- **Snapshot** — periodically, etcd compacts the WAL by creating a snapshot of current state. Reduces recovery time.
- **bbolt (formerly BoltDB)** — the embedded key-value database used for actual storage. Memory-mapped B+ tree.
- **MVCC (Multi-Version Concurrency Control)** — every modification creates a new revision. Old revisions are kept until compacted, enabling watch streams and historical reads.

**Network Layer:**
- **Peer communication** (port 2380) — between etcd members for Raft consensus
- **Client communication** (port 2379) — from clients (API server) to etcd
- All communication is over **gRPC with mutual TLS**

**API Layer:**
- **etcd v3 API** — gRPC-based, the modern API used by Kubernetes
- Supports get, put, delete, txn (transactions), watch, lease, lock

**Quorum and the Raft Protocol:**

**Quorum** is the minimum number of nodes required to make a decision. For an N-node cluster, quorum is `(N/2) + 1`:

| Cluster Size | Quorum | Failure Tolerance |
|--------------|--------|-------------------|
| 1 | 1 | 0 (no HA) |
| 3 | 2 | 1 node |
| 5 | 3 | 2 nodes |
| 7 | 4 | 3 nodes |
| 9 | 5 | 4 nodes |

**Why Odd Numbers:**

With 4 nodes, quorum is 3 — you can lose 1 node. With 3 nodes, quorum is 2 — you can also lose 1 node. So 4 nodes give you no more fault tolerance than 3 but cost more and have more replication traffic. Going from 3 to 5 is meaningful (tolerate 2 failures); going from 4 to 5 also gains nothing significant.

Production clusters typically use 3 or 5 etcd members. Beyond 5, the latency cost of Raft consensus (every write must be confirmed by majority) typically outweighs the failure tolerance benefit.

**How Writes Work:**

1. **Client sends write to any etcd node**
2. **If the receiving node is not the leader, it forwards to the leader**
3. **Leader appends to its WAL** and sends AppendEntries RPC to followers
4. **Followers append to their WALs** and respond
5. **Once a quorum (majority) has acknowledged**, the leader commits the entry
6. **Leader applies to its state machine** (bbolt) and responds to client
7. **Followers learn of the commit** on next heartbeat and apply to their state machines

This means writes have latency proportional to network round-trip × log of cluster size, plus disk write time. **Disk performance is critical** — etcd recommends NVMe SSDs.

**How Reads Work:**

etcd supports two read modes:

**Linearizable reads (default, strongest consistency):**
- The receiving node confirms it's still the leader (via heartbeat)
- Returns the most recently committed value
- Slower due to the leader confirmation
- Always returns the latest value

**Serializable reads:**
- Returns the value from the local node's state machine
- Faster (no network round-trip)
- May return stale data (a value committed but not yet applied locally)
- Used for some performance-critical paths

The Kubernetes API server uses linearizable reads for most operations to ensure consistency.

**Failure Tolerance:**

If a minority of nodes fail, the cluster continues operating with the majority. If you lose quorum (e.g., 2 of 3 nodes fail), the cluster becomes **read-only or completely unavailable** — no new writes can be committed because there's no majority to confirm them.

**Network Partitions:**

If the network partitions the cluster (e.g., 3 nodes split into 2+1):
- The side with 2 nodes (majority) continues operating
- The side with 1 node can't elect a leader and can't accept writes
- When the partition heals, the isolated node catches up by following the new leader

**The "Split-Brain" Prevention:**

Raft prevents split-brain by requiring **majority votes** for both:
1. **Leader election** — a candidate needs votes from majority to become leader
2. **Write commitment** — a write needs majority acknowledgment to be committed

This means only one leader can exist at a time, and only its writes are committed. Even if a partitioned old leader still thinks it's the leader, it can't commit writes (no majority).

**Storage Sizing:**

etcd has a default storage quota of **2 GiB**. For large Kubernetes clusters, increase this:

```bash
# etcd configuration
quota-backend-bytes: 8589934592  # 8 GiB
```

If etcd hits the quota, it enters **NO SPACE** alarm mode — no more writes allowed. You must compact and defragment to free space.

**Compaction and Defragmentation:**

**Compaction** removes old revisions of keys (MVCC history). Kubernetes automatically triggers compaction every 5 minutes by default.

**Defragmentation** reclaims disk space after compaction:
```bash
etcdctl defrag --endpoints=https://etcd-1:2379
```

Defrag must be run on each member individually and locks the database during operation. Do it one node at a time during maintenance windows.

**Performance Considerations:**

- **Disk latency** is the biggest factor — etcd writes to WAL synchronously. NVMe SSD recommended; HDD is too slow.
- **Network latency** matters for inter-node communication. etcd members should be in the same network/AZ for low latency, but spread across AZs in a region for fault tolerance.
- **Memory** — etcd memory-maps the entire database. For 8 GiB DB, you want at least 16 GiB RAM.
- **CPU** — moderate; mostly for crypto (TLS) and database operations.

**Monitoring etcd Health:**

Critical metrics:
- `etcd_server_has_leader` — is there a leader?
- `etcd_server_leader_changes_seen_total` — leader churn (should be near zero)
- `etcd_disk_wal_fsync_duration_seconds` — WAL write latency (P99 < 10ms)
- `etcd_disk_backend_commit_duration_seconds` — backend commit latency
- `etcd_network_peer_round_trip_time_seconds` — inter-member latency
- `etcd_mvcc_db_total_size_in_bytes` — database size

---

### 52. How do you recover corrupted etcd?

etcd corruption or data loss is one of the most serious failure scenarios in Kubernetes. Recovery procedures depend on the failure mode and what backups exist.

**Types of etcd Failures:**

1. **Single member failure** — one node down, cluster still has quorum
2. **Loss of quorum** — majority of nodes failed, cluster can't accept writes
3. **Data corruption** — files on disk corrupted, member can't start
4. **Complete data loss** — all etcd data lost (extremely bad)
5. **Logical corruption** — bad data in etcd from buggy clients

**Single Member Failure Recovery:**

If one member of a 3-node cluster fails:

```bash
# 1. Remove the failed member from the cluster
etcdctl member remove <member-id> --endpoints=https://healthy-etcd:2379

# 2. Clean up the failed node's data directory
rm -rf /var/lib/etcd/

# 3. Re-add the member to the cluster
etcdctl member add etcd-3 --peer-urls=https://etcd-3:2380 \
  --endpoints=https://healthy-etcd:2379

# 4. Start etcd on the failed node with --initial-cluster-state=existing
etcd --name etcd-3 \
  --initial-cluster etcd-1=https://etcd-1:2380,etcd-2=https://etcd-2:2380,etcd-3=https://etcd-3:2380 \
  --initial-cluster-state=existing \
  ...
```

The new member catches up by replicating the WAL from existing members.

**Loss of Quorum Recovery:**

This is more serious. If 2 of 3 etcd members are down, you have no quorum. The remaining member is read-only and can't accept writes.

**Option 1: Restore the failed members**
Just restart them — if the data is intact, they'll rejoin and quorum is restored.

**Option 2: Force new cluster from surviving member**

If you can't recover the failed members, you can force the survivor to be a new single-node cluster:

```bash
# On the surviving etcd node
etcd --force-new-cluster ...
```

This rewrites cluster membership to contain only this node. Then add new members back one by one.

**Critical warning:** This effectively discards any writes the failed members had but the survivor didn't. Used only when you have no other option.

**Restoring from Backup:**

The standard recovery procedure for serious corruption or total data loss.

**Step 1: Stop the etcd cluster**
Stop all etcd processes on all nodes. The cluster must be completely down to avoid split-brain.

```bash
systemctl stop etcd  # on each node
```

**Step 2: Restore snapshot on each node**

```bash
ETCDCTL_API=3 etcdctl snapshot restore /backups/etcd-snapshot.db \
  --name etcd-1 \
  --initial-cluster etcd-1=https://etcd-1:2380,etcd-2=https://etcd-2:2380,etcd-3=https://etcd-3:2380 \
  --initial-cluster-token etcd-cluster-restore \
  --initial-advertise-peer-urls https://etcd-1:2380 \
  --data-dir /var/lib/etcd-restored
```

Note the new `--initial-cluster-token` — this prevents the restored cluster from communicating with any remnants of the old one.

**Step 3: Update etcd configuration**

Point each etcd member to the restored data directory.

**Step 4: Start the cluster**
Start all members simultaneously. They'll form a new cluster from the restored data.

```bash
systemctl start etcd
```

**Step 5: Verify**

```bash
etcdctl endpoint health --cluster
etcdctl member list
```

**Step 6: Restart kube-apiservers**

The API servers had stale data in their caches; restart them so they reconcile with the restored etcd.

**Step 7: Verify cluster state**

```bash
kubectl get nodes
kubectl get pods -A
```

You may have orphaned resources (Pods that existed in old etcd but not new). Controllers will reconcile and clean these up.

**Critical Considerations:**

**Workloads continue running during etcd outage:**
Containers running on nodes don't stop when etcd dies — kubelet has them cached. But:
- No new Pods can be scheduled
- No deployments/scaling/updates work
- kubectl is broken
- Existing Pod failures don't trigger rescheduling
- Service endpoints don't update

**Data loss between snapshot and restore:**
Anything written to etcd between the last snapshot and the failure is lost. Snapshot frequency is critical. Recommended: every 30 minutes.

**Snapshot Creation:**

```bash
ETCDCTL_API=3 etcdctl snapshot save /backups/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db \
  --endpoints=https://etcd-1:2379 \
  --cacert=/etc/etcd/ca.crt \
  --cert=/etc/etcd/server.crt \
  --key=/etc/etcd/server.key
```

In production, automate this with a CronJob, cron, or backup operator. Store snapshots **off-cluster** (S3, GCS, etc.) — otherwise the backup goes down with the cluster.

**Logical Corruption (Bad Data):**

If etcd is healthy but contains bad data (e.g., a runaway controller created millions of orphaned objects), you can:

1. **Identify and remove bad data** via kubectl
2. **Find specific keys** with etcdctl:
```bash
etcdctl get /registry/pods/ --prefix --keys-only | grep <pattern>
etcdctl del /registry/pods/namespace/pod-name
```
3. **Direct etcd edits** are last resort — bypassing the API server bypasses validation and finalizers

**Disaster Recovery Best Practices:**

1. **Automated snapshots** — every 15-30 minutes
2. **Off-cluster snapshot storage** — S3, GCS, etc.
3. **Test restore procedure regularly** — backups are useless if untested
4. **Documented runbooks** for various failure scenarios
5. **Monitoring** — alert on etcd health degradation early
6. **Multi-region for critical clusters** — etcd across regions has latency cost but extreme HA
7. **Velero or similar** for application-level backups (PVs, ConfigMaps, etc.)

**Recovery Time Objectives:**

For production:
- Single member failure: minutes to hours (depending on data size)
- Restore from snapshot: 10-30 minutes for typical clusters
- Larger snapshot databases (10GB+) can take hours to restore

---

### 53. Explain Raft consensus in etcd.

Raft is the consensus algorithm etcd uses to maintain consistency across distributed nodes. Compared to Paxos, Raft was designed to be more understandable while providing the same guarantees.

**The Problem Raft Solves:**

Distributed systems need to agree on:
- The order of operations
- What data is committed
- Who is in charge (leader)

With network partitions, message delays, and node failures, achieving this agreement is hard. Raft provides a structured way to do it.

**Core Concepts:**

**Server States:**

Each etcd node is always in one of three states:
- **Leader** — handles all client writes, replicates to followers
- **Follower** — passive; receives entries from leader, votes in elections
- **Candidate** — temporary state during election

**Terms:**

Time is divided into **terms** — monotonically increasing integers. Each term has at most one leader. Terms are how Raft handles ordering across leadership changes — a higher term always wins.

**Log Entries:**

The fundamental unit of replication. Each entry has:
- **Term** — the term when created
- **Index** — position in the log
- **Command** — the operation (put, delete, etc.)
- **State** — uncommitted or committed

Logs must be identical across all nodes once committed.

**The Raft Protocol Phases:**

**1. Leader Election**

When a follower hasn't heard from a leader for a while (election timeout, randomized 150-300ms), it transitions to candidate:

1. Increments its current term
2. Votes for itself
3. Sends RequestVote RPCs to all other nodes
4. Other nodes vote yes if:
   - They haven't voted yet this term
   - The candidate's log is at least as up-to-date as theirs

If the candidate gets votes from a majority, it becomes leader. If a higher-term leader is detected, it reverts to follower.

The randomized timeouts prevent perpetual split votes.

**2. Log Replication**

Once a leader is elected:

1. **Client sends a write to the leader**
2. **Leader appends the entry to its log** (uncommitted)
3. **Leader sends AppendEntries RPC** to all followers
4. **Followers append to their logs** and respond
5. **When a majority has acknowledged**, the leader **commits** the entry
6. **Leader applies the entry** to its state machine and responds to client
7. **In subsequent AppendEntries**, leader notifies followers of the new commit index
8. **Followers apply** the committed entries to their state machines

**3. Heartbeats**

The leader sends **empty AppendEntries** as heartbeats (every 100ms in etcd) to:
- Prevent followers from starting new elections
- Indicate the leader is still alive

If a follower misses heartbeats for the election timeout, it becomes a candidate.

**Safety Properties:**

Raft guarantees several critical properties:

**Election Safety:**
At most one leader can be elected in a given term.

**Leader Append-Only:**
A leader never overwrites or deletes entries in its log; only appends.

**Log Matching:**
If two logs have an entry with the same index and term, they're identical from that point back. Achieved by AppendEntries doing consistency checks.

**Leader Completeness:**
Any committed entry in a term is present in all higher-term leaders. Guaranteed by the voting rules — candidates must have up-to-date logs.

**State Machine Safety:**
If a node has applied an entry at a given index, no other node will apply a different entry at the same index.

**Practical Implications for etcd:**

**Write Latency:**
Every write requires:
- Disk write on leader (WAL)
- Network round-trip to majority of followers
- Disk write on those followers (WAL)
- Network round-trip back to leader

This is why etcd needs fast disks and low-latency networks.

**Read Consistency:**
For linearizable reads, the leader must confirm it's still the leader by getting heartbeat acknowledgments from a majority. This adds latency to reads.

**Why Quorum Matters:**
The need for majority acknowledgment is what prevents split-brain. Even if the network partitions, only the side with a majority can commit writes.

**Log Compaction:**
Raft logs grow indefinitely. Periodically, etcd creates a snapshot of the state machine and discards old log entries. When a new follower joins, the leader sends the snapshot rather than replaying every entry.

**Handling Leader Failures:**

When a leader fails:
1. Followers stop receiving heartbeats
2. After election timeout, one becomes a candidate
3. New election held (votes based on log up-to-dateness)
4. Winner becomes new leader
5. New leader's term is higher than old
6. Operations resume

Failover typically takes 1-2 seconds in well-configured etcd clusters.

**Edge Cases Raft Handles:**

**Network Partition:**
If a leader is partitioned from majority:
- It can't commit new writes (no majority acknowledgment)
- Followers on the majority side eventually time out and elect new leader
- Old leader, when partition heals, sees higher term and reverts to follower

**Slow Followers:**
Leaders track each follower's nextIndex (the next log entry to send). If a follower is behind, the leader sends entries from its nextIndex forward. This handles slow or temporarily-disconnected followers automatically.

**Configuration Changes:**

Adding/removing etcd members is a special operation. Raft handles this via "joint consensus" — a transitional period where both old and new configurations agree, preventing split-brain during reconfiguration.

In practice, you change membership one node at a time and Raft handles the transitions.

**Comparison with Paxos:**

Both Raft and Paxos provide the same guarantees, but Raft is designed for clarity:
- Strong leader (Paxos has more peer-to-peer feel)
- Clear separation of leader election, log replication, safety
- Easier to implement correctly

**Why It Matters for Kubernetes:**

Every Kubernetes API write goes through etcd, which uses Raft. This means:
- Writes are consistent across the cluster
- The cluster has a single source of truth
- The cluster can tolerate failures up to (N-1)/2 etcd nodes
- Disk and network performance directly affect API latency

---

### 54. What happens during etcd leader election?

Leader election is the process by which etcd nodes agree on who serves as the cluster leader. It happens at cluster startup and whenever the existing leader fails or is partitioned.

**When Elections Are Triggered:**

1. **Cluster startup** — no leader exists yet
2. **Leader failure** — the existing leader crashes or becomes unreachable
3. **Network partition** — the leader is on the minority side of a partition
4. **Leader voluntary stepdown** — rare, but a leader can step down for graceful handover

**The Trigger Mechanism:**

Each follower has an **election timeout** — typically 1000ms in etcd, with random jitter (1000-1500ms). When a follower doesn't receive any AppendEntries RPC (heartbeat or log replication) from the leader within this timeout, it assumes the leader has failed.

The randomization is critical — without it, all followers would time out simultaneously and split votes endlessly.

**Election Phases:**

**Phase 1: Become a Candidate**

When the election timeout fires:
1. The follower increments its **currentTerm**
2. Transitions state to **Candidate**
3. **Votes for itself**
4. Resets its election timer
5. Sends **RequestVote RPCs** to all other nodes

The RequestVote RPC contains:
- The candidate's term
- The candidate's ID
- The index and term of the candidate's last log entry

**Phase 2: Voting Rules**

When a node receives a RequestVote RPC, it grants its vote if **all** of these are true:

1. **The candidate's term is at least as high as the receiver's term**
   - If the receiver's term is higher, vote is denied (and receiver tells candidate the higher term)

2. **The receiver hasn't voted in this term** (or has voted for the same candidate)
   - Each node votes for at most one candidate per term

3. **The candidate's log is at least as up-to-date as the receiver's**
   - "Up-to-date" means: last log entry has a higher term, OR same term but higher index
   - This prevents candidates with stale data from becoming leader

This is the **safety property** — only candidates with current data can become leader.

**Phase 3: Election Outcomes**

Three outcomes are possible:

**Outcome A: Candidate wins**
The candidate receives votes from a **majority** of nodes (including itself).

When this happens:
1. The candidate transitions to **Leader** state
2. Immediately starts sending **heartbeat AppendEntries** to all followers
3. Heartbeats prevent other elections from starting
4. The new leader begins serving client requests

**Outcome B: Another candidate becomes leader**
The candidate receives an AppendEntries from another node with term ≥ its current term.

When this happens:
1. The candidate recognizes a legitimate leader exists
2. Transitions back to **Follower** state
3. Updates its currentTerm if necessary
4. Begins replicating from the new leader

**Outcome C: Split vote (no winner)**
Multiple candidates split the vote, no one gets a majority.

When this happens:
1. The election timer eventually expires again
2. Each candidate increments its term again
3. New election begins
4. Randomized timeouts ensure they don't all time out at the same moment

Split votes are rare due to randomized timeouts but can happen.

**Election Timeline (Typical):**

```
T=0ms:    Leader fails (crash, partition, etc.)
T=0-1500ms: Followers wait for heartbeat (election timeout, randomized)
T=1500ms: First follower times out, becomes candidate
T=1500ms: Candidate sends RequestVote RPCs
T=1510ms: Other nodes receive RequestVote, send votes back
T=1520ms: Candidate has majority votes
T=1520ms: Candidate becomes leader, sends first heartbeat
T=1530ms: Followers receive heartbeat, recognize new leader
```

Total failover time: typically **1-2 seconds** for a well-configured cluster.

**Configuration Parameters:**

In etcd:
- `--heartbeat-interval` (default 100ms) — how often leader sends heartbeats
- `--election-timeout` (default 1000ms) — how long followers wait before starting election

The election-timeout should be much larger than heartbeat-interval, typically 10x. This prevents spurious elections during temporary network jitter.

For high-latency networks (e.g., cross-region), increase both:
```
--heartbeat-interval=300
--election-timeout=3000
```

**The Term Number's Role:**

Terms are the **logical clock** of Raft. They serve several purposes:

1. **Detecting stale leaders** — if a node sees a higher term, it knows there's a newer leader and should step down
2. **Ordering events** — operations with higher terms are always more recent
3. **Preventing duplicate writes** — committed entries from older terms can't be uncommitted

Every RPC includes the sender's current term. Receivers always check this first:
- If sender's term is lower, reject the RPC
- If sender's term is higher, update own term and revert to follower

**What Happens to In-Flight Writes:**

When a leader fails mid-write:

- **Committed writes** (acknowledged by majority): preserved. The new leader will have them.
- **Uncommitted writes** (only on the old leader): may be lost. The client sees an error and should retry.

This is why etcd clients must be designed for retry logic.

**Recovering from Persistent Leader Churn:**

If you see frequent leader elections in `etcd_server_leader_changes_seen_total`, possible causes:

1. **Network instability** between etcd nodes
2. **Slow disks** causing heartbeat timeouts (leader can't send heartbeats fast enough)
3. **CPU contention** on etcd nodes
4. **Election timeout too short** for the network latency
5. **Bugs in the deployment** (firewalls, MTU issues)

Frequent elections degrade Kubernetes API performance significantly — every election causes a brief write outage.

**Handling Partitioned Clusters:**

If a 3-node cluster partitions into 2+1:

**Majority side (2 nodes):**
- If they had the old leader, fine — continues operating
- If they didn't have the leader, one times out, becomes candidate, wins election (has 2/2 votes from majority side)
- Cluster operates normally on this side

**Minority side (1 node):**
- If it was the leader, it can't get majority for new writes
- If a partition heals, it discovers the higher term, steps down, syncs with new leader
- Until the partition heals, this side is unable to commit writes

**Voluntary Leadership Transfer:**

etcd supports explicit leader transfer:
```bash
etcdctl move-leader <new-leader-id>
```

The current leader sends a TimeoutNow message to the target, which immediately starts an election with the current log. Useful for maintenance or planned migrations.

---

### 55. How does scheduler scoring work?

The Kubernetes scheduler uses a **two-phase process** to assign Pods to nodes: filtering eliminates impossible nodes, then scoring ranks the remaining candidates. Understanding scoring is essential for tuning workload placement.

**The Two Phases:**

**Phase 1: Filtering (Predicates)**
Eliminate nodes that cannot run the Pod. This is binary — node either qualifies or doesn't. If zero nodes pass filtering, the Pod stays Pending.

**Phase 2: Scoring (Priorities)**
Among qualifying nodes, compute a score from 0 to 100. The highest-scoring node wins. Ties are broken randomly.

This document focuses on scoring.

**The Scoring Framework:**

The scheduler is built on a **plugin-based framework** with several extension points. Scoring happens in three steps:

1. **Score** — each scoring plugin computes a raw score for each candidate node
2. **NormalizeScore** — scores are normalized to the 0-100 range across all nodes
3. **Weighted Sum** — final score is the weighted sum of all plugin scores

**Built-in Scoring Plugins:**

**NodeResourcesFit (default weight: 1)**

The most fundamental — scores nodes based on resource utilization. Supports multiple strategies:

- **LeastAllocated** (default): prefers nodes with most free resources. Spreads load.
- **MostAllocated**: prefers nodes that are most full. Bin packing — useful for cost optimization with autoscaling.
- **RequestedToCapacityRatio**: configurable function. Most flexible.

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- pluginConfig:
  - name: NodeResourcesFit
    args:
      scoringStrategy:
        type: MostAllocated
        resources:
        - name: cpu
          weight: 1
        - name: memory
          weight: 1
```

**ImageLocality (default weight: 1)**
Prefers nodes that already have the container images cached locally. Faster Pod start because no image pull needed.

The bigger the image and more recent the use, the bigger the score boost.

**InterPodAffinity (default weight: 2)**
Scores based on Pod affinity and anti-affinity preferences (the soft `preferredDuringSchedulingIgnoredDuringExecution` rules).

Higher score for nodes that match the desired affinity, lower for matching anti-affinity.

**NodeAffinity (default weight: 2)**
Scores based on `preferredDuringSchedulingIgnoredDuringExecution` node affinity rules. The weight from your affinity expression contributes to the score.

**PodTopologySpread (default weight: 2)**
Scores based on Pod topology spread constraints. Prefers nodes in topology domains with fewer matching Pods, encouraging spread.

**TaintToleration (default weight: 3)**
Scores nodes based on taint matching. Nodes with more taints the Pod tolerates score lower (assumed less desirable), nodes with fewer "unmatched" taints score higher.

**NodeResourcesBalancedAllocation (default weight: 1)**
Prefers nodes where CPU and memory utilization are balanced. Avoids nodes where one resource is heavily used and the other is barely touched.

For example, a node with 80% CPU and 20% memory used is less balanced than one with 50%/50%.

**The Scoring Process in Detail:**

For each candidate node (after filtering):

1. **Each scoring plugin runs** and produces a raw score (typically 0-100)
2. **NormalizeScore phase** — scores are adjusted to a consistent range
3. **Plugin weights are applied** — each score multiplied by its plugin weight
4. **Sum of weighted scores** = final node score

```
final_score(node) = sum over all plugins of (weight_p * score_p(node))
```

The node with the highest final score wins.

**Example Calculation:**

Imagine a Pod that:
- Requires 2 cores, 4 GiB memory
- Has soft anti-affinity for other replicas
- The cluster has 3 candidate nodes

| Plugin | Weight | Node A score | Node B score | Node C score |
|--------|--------|--------------|--------------|--------------|
| NodeResourcesFit (LeastAllocated) | 1 | 80 (lots of free) | 50 (medium) | 30 (full-ish) |
| ImageLocality | 1 | 100 (has image) | 0 (no image) | 0 (no image) |
| InterPodAffinity | 2 | 50 (has 1 replica) | 100 (no replicas) | 50 (has 1 replica) |
| NodeAffinity | 2 | 100 (preferred zone) | 100 (preferred zone) | 0 (wrong zone) |
| TaintToleration | 3 | 100 | 100 | 100 |

Computed:
- Node A: 1×80 + 1×100 + 2×50 + 2×100 + 3×100 = 780
- Node B: 1×50 + 1×0 + 2×100 + 2×100 + 3×100 = 750
- Node C: 1×30 + 1×0 + 2×50 + 2×0 + 3×100 = 430

Node A wins.

**Strategy Trade-offs:**

**LeastAllocated (spread workload):**
- Better fault tolerance — losing one node loses less of any workload
- More resource fragmentation
- Higher cluster autoscaler bills (more nodes used)
- Better performance under bursty load

**MostAllocated (bin packing):**
- Better resource utilization
- Cluster autoscaler can scale down more aggressively
- Lower costs
- Performance risk under bursty load (less spare capacity)
- More impact if a node fails

For autoscaling-aware clusters, MostAllocated is often better — it enables tight packing and node consolidation.

**Customizing the Scheduler:**

You can:

1. **Adjust weights** of existing plugins
2. **Disable plugins** you don't need
3. **Configure plugins** with different strategies
4. **Use multiple profiles** — different scoring for different Pod types

```yaml
profiles:
- schedulerName: default-scheduler
  plugins:
    score:
      enabled:
      - name: NodeResourcesFit
      - name: NodeAffinity
      disabled:
      - name: ImageLocality   # disable this one
```

Then Pods using `spec.schedulerName: default-scheduler` use this profile.

**Multiple Schedulers:**

You can run multiple scheduler instances with different configurations. Pods specify which scheduler to use:

```yaml
spec:
  schedulerName: my-batch-scheduler
```

Common patterns:
- Default scheduler for normal workloads
- Custom scheduler with MostAllocated for batch jobs
- Custom scheduler with topology awareness for stateful workloads

**Performance Considerations:**

Scoring runs on every candidate node. For large clusters (thousands of nodes), scoring can become expensive. Optimizations:

- **PercentageOfNodesToScore** — by default, the scheduler only scores 50% of nodes (capped). For small clusters, all nodes are scored.
- **Disable expensive plugins** if not needed
- **Use node labels and affinity** to reduce candidate set

```yaml
percentageOfNodesToScore: 50  # only score 50% of nodes
```

**Pod Preemption (Beyond Scoring):**

If no node passes filtering, the scheduler can preempt lower-priority Pods to make room:

1. Pod with high priority can't be scheduled
2. Scheduler looks for nodes where evicting some Pods would enable scheduling
3. Identifies victim Pods (lowest priority, respecting PDBs as a courtesy)
4. Deletes victim Pods
5. New Pod is scheduled on the freed node

This is configured via PriorityClass objects.

---

### 56. Explain scheduler extender use cases.

Scheduler extenders are the **older mechanism** for extending the Kubernetes scheduler with custom logic. They've largely been superseded by the **Scheduling Framework** with plugins, but extenders are still in use and worth understanding.

**The Concept:**

A scheduler extender is an **external webhook** that the scheduler calls during the scheduling process. The webhook can filter nodes, score nodes, or be involved in binding.

The scheduler sends HTTP requests to the extender with the Pod and candidate nodes. The extender responds with its decisions.

**Extender Configuration:**

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
extenders:
- urlPrefix: "http://custom-scheduler-extender:8080/scheduler"
  filterVerb: "filter"
  prioritizeVerb: "prioritize"
  preemptVerb: "preempt"
  bindVerb: "bind"
  weight: 5
  enableHTTPS: false
  managedResources:
  - name: "example.com/gpu"
    ignoredByScheduler: true
```

This tells the scheduler:
- Send filter requests to `/scheduler/filter`
- Send prioritize requests to `/scheduler/prioritize`
- The extender manages `example.com/gpu` resources
- For these resources, the built-in scheduler ignores them (extender handles fitting)

**Extender Phases:**

**1. Filter Phase**

After the built-in scheduler runs its filters, it sends the surviving nodes to the extender:

```json
{
  "Pod": {...},
  "Nodes": {"items": [...]},
  "NodeNames": ["node-1", "node-2", "node-3"]
}
```

The extender responds with the subset that pass its filter:

```json
{
  "Nodes": {"items": [...]},
  "NodeNames": ["node-1", "node-3"],
  "FailedNodes": {
    "node-2": "insufficient custom resource"
  },
  "Error": ""
}
```

**2. Prioritize Phase**

The extender scores the filtered nodes:

```json
{
  "Pod": {...},
  "Nodes": {"items": [...]}
}
```

The extender responds with scores:

```json
[
  {"Host": "node-1", "Score": 80},
  {"Host": "node-3", "Score": 95}
]
```

The scheduler combines these with built-in scores using the configured weight.

**3. Preempt Phase**

If preemption is needed, the scheduler asks the extender which victims it would allow. Useful for protecting workloads beyond just PriorityClass.

**4. Bind Phase**

The extender can take over the actual binding — useful when special handling is needed (e.g., reserving resources in an external system before binding).

**Common Use Cases:**

**1. GPU and Specialized Hardware Scheduling**

Before the Device Plugin framework, GPU scheduling often used extenders. Even now, complex GPU scheduling (NVLink topology awareness, MIG slicing) sometimes uses extenders.

**2. Resource-Aware Scheduling**

External systems track resources Kubernetes doesn't natively understand:
- Network bandwidth allocations
- Storage IOPS reservations
- Custom hardware (FPGAs, TPUs)
- License-based scheduling

The extender consults the external system to filter/score.

**3. Cross-Cluster Scheduling**

For multi-cluster setups, an extender could consult cluster-level metrics and decide which cluster (or which nodes in this cluster) should run a Pod.

**4. Cost-Aware Scheduling**

An extender could consider spot instance pricing, on-demand vs reserved pricing, and steer workloads toward cheaper options.

**5. Performance-Aware Scheduling**

Use historical performance data (latency to dependencies, throughput) to score nodes.

**6. Capacity Reservation**

External systems that manage capacity reservations (in HPC environments) use extenders to filter nodes based on reservations.

**7. Volume Scheduling (Older)**

Before in-tree volume scheduling, extenders helped place Pods near their storage. CSI now handles this natively.

**Limitations of Extenders:**

1. **HTTP overhead** — every scheduling decision involves HTTP calls, adding latency
2. **Single point of failure** — if the extender is down, scheduling stalls or falls back to built-in only
3. **Limited integration points** — extenders only hook at a few phases (filter, prioritize, preempt, bind)
4. **Coarser-grained** — can't override individual scheduler decisions like an in-process plugin
5. **State management** — extenders typically need their own state, separate from the scheduler

**Scheduling Framework vs Extenders:**

The Scheduling Framework (introduced in v1.15, GA in v1.19) provides **in-process plugins** that can hook into many more points:
- QueueSort
- PreFilter
- Filter
- PostFilter
- PreScore
- Score
- NormalizeScore
- Reserve
- Permit
- PreBind
- Bind
- PostBind

These plugins run inside the scheduler — no HTTP overhead, fine-grained control.

**Migration Path:**

The Kubernetes community recommends:
- **New custom logic**: write as a Scheduling Framework plugin
- **Existing extenders**: migrate when feasible
- **Simple use cases**: extenders may still be appropriate if you don't want to compile into the scheduler

**Building a Scheduling Framework Plugin:**

You write Go code that implements plugin interfaces:

```go
type ExamplePlugin struct {
    handle framework.Handle
}

func (pl *ExamplePlugin) Name() string {
    return "ExamplePlugin"
}

func (pl *ExamplePlugin) Filter(ctx context.Context, state *framework.CycleState,
    pod *v1.Pod, nodeInfo *framework.NodeInfo) *framework.Status {

    // Custom filtering logic
    if !someCondition(pod, nodeInfo) {
        return framework.NewStatus(framework.Unschedulable, "doesn't satisfy custom requirement")
    }
    return framework.NewStatus(framework.Success)
}
```

The plugin is then registered and compiled into a custom scheduler binary.

**Multiple Schedulers Approach:**

Instead of extending the default scheduler, you can deploy a **separate scheduler binary** with custom logic. Pods specify which scheduler:

```yaml
spec:
  schedulerName: my-custom-scheduler
```

This gives full control but operationally is more complex.

**Real-World Examples:**

- **Volcano** — batch scheduling framework, originally used extenders
- **YuniKorn** — Apache project for big data workloads on Kubernetes
- **Tencent Cloud GPU Manager** — used extenders for GPU sharing

Most modern implementations are migrating to or using the Scheduling Framework.

---

### 57. How do admission controllers modify objects?

Admission controllers can modify objects through **mutating admission webhooks** that intercept API requests and return JSON patches. This is one of Kubernetes' most powerful extensibility mechanisms.

**The Admission Flow:**

When you create or update a resource via the API:

1. **Authentication** — who are you?
2. **Authorization** — are you allowed?
3. **Mutating admission** — modify the object (this is where mutating webhooks run)
4. **Schema validation** — does it match the OpenAPI schema?
5. **Validating admission** — is it acceptable per policies?
6. **Persistence** — write to etcd

Mutating admission runs **after auth/authz** but **before validation** — allowing webhooks to fix up objects before they're checked for correctness.

**Built-in Mutating Admission Controllers:**

Several are compiled into the API server:

**MutatingAdmissionWebhook** — invokes external webhooks (we'll cover this in detail)

**LimitRanger** — applies LimitRange defaults to Pods. If a container doesn't specify resources, LimitRanger fills in the namespace's defaults.

**ServiceAccount** — automatically:
- Adds `serviceAccountName: default` if not specified
- Mounts the service account token
- Sets image pull secrets from the service account

**DefaultStorageClass** — assigns the default StorageClass to PVCs without one specified

**DefaultTolerationSeconds** — adds default tolerations for `node.kubernetes.io/not-ready` and `unreachable` taints

**PodNodeSelector** — adds nodeSelector based on namespace annotations

**PodTopologySpread** — adds default topology spread constraints

**External Mutating Webhooks:**

The MutatingAdmissionWebhook plugin calls **external webhooks** registered via MutatingWebhookConfiguration objects.

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: sidecar-injector
webhooks:
- name: inject.sidecar.example.com
  clientConfig:
    service:
      name: sidecar-injector
      namespace: kube-system
      path: /mutate
    caBundle: <base64-CA>
  rules:
  - apiGroups: [""]
    apiVersions: ["v1"]
    operations: ["CREATE"]
    resources: ["pods"]
  admissionReviewVersions: ["v1"]
  sideEffects: None
  failurePolicy: Fail
  namespaceSelector:
    matchLabels:
      sidecar-injection: enabled
  reinvocationPolicy: Never
```

**The AdmissionReview Protocol:**

When a Pod is created in a labeled namespace, the API server sends an `AdmissionReview` to the webhook:

```json
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "request": {
    "uid": "abc-123",
    "kind": {"group": "", "version": "v1", "kind": "Pod"},
    "resource": {"group": "", "version": "v1", "resource": "pods"},
    "namespace": "production",
    "operation": "CREATE",
    "userInfo": {"username": "alice", "groups": ["dev"]},
    "object": {
      "metadata": {"name": "my-app"},
      "spec": {
        "containers": [{
          "name": "app",
          "image": "myapp:v1"
        }]
      }
    }
  }
}
```

The webhook processes this and returns:

```json
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "response": {
    "uid": "abc-123",
    "allowed": true,
    "patchType": "JSONPatch",
    "patch": "<base64-encoded JSON Patch>"
  }
}
```

The patch (base64-decoded) is a JSON Patch (RFC 6902) describing the modifications:

```json
[
  {
    "op": "add",
    "path": "/spec/containers/-",
    "value": {
      "name": "envoy-sidecar",
      "image": "envoyproxy/envoy:v1.28",
      "ports": [{"containerPort": 15000}]
    }
  },
  {
    "op": "add",
    "path": "/spec/volumes",
    "value": [{
      "name": "envoy-config",
      "configMap": {"name": "envoy-config"}
    }]
  },
  {
    "op": "add",
    "path": "/metadata/annotations/sidecar.example.com~1injected",
    "value": "true"
  }
]
```

**JSON Patch Operations:**

The patch uses standard JSON Patch operations:
- **add** — add a value at a path
- **remove** — remove a value
- **replace** — replace a value
- **move** — move a value from one path to another
- **copy** — copy a value
- **test** — verify a value (no modification)

Paths use JSON Pointer syntax with `/` as separator. Special characters in keys are escaped (`/` becomes `~1`, `~` becomes `~0`).

**The Sidecar Injection Example:**

Istio's sidecar injector is the canonical example. When a Pod is created in an Istio-enabled namespace:

1. Webhook receives the Pod's AdmissionReview
2. Checks if the Pod should have a sidecar (labels, annotations)
3. If yes, constructs a JSON Patch that:
   - Adds an init container (`istio-init` for iptables setup)
   - Adds the Envoy sidecar container
   - Adds volumes for certs and config
   - Adds environment variables
   - Modifies the Pod spec to route traffic through Envoy

The patch is sent back, API server applies it, the Pod now has Istio sidecar — all transparent to the user.

**Common Mutations:**

**1. Adding default labels and annotations:**

```json
{"op": "add", "path": "/metadata/labels/managed-by", "value": "platform-team"}
```

**2. Injecting environment variables:**

```json
{
  "op": "add",
  "path": "/spec/containers/0/env/-",
  "value": {"name": "TEAM", "value": "platform"}
}
```

**3. Setting resource defaults:**

```json
{
  "op": "add",
  "path": "/spec/containers/0/resources",
  "value": {
    "requests": {"cpu": "100m", "memory": "128Mi"},
    "limits": {"cpu": "500m", "memory": "256Mi"}
  }
}
```

**4. Adding security contexts:**

```json
{
  "op": "add",
  "path": "/spec/securityContext",
  "value": {
    "runAsNonRoot": true,
    "runAsUser": 1000,
    "fsGroup": 2000
  }
}
```

**5. Rewriting image references:**

```json
{
  "op": "replace",
  "path": "/spec/containers/0/image",
  "value": "registry.company.com/library/nginx:1.21"
}
```

This rewrites public image references to use an internal mirror.

**Reinvocation Policy:**

```yaml
reinvocationPolicy: IfNeeded
```

When multiple mutating webhooks modify the same object, order matters. If webhook A adds a sidecar, then webhook B modifies that sidecar, webhook A should run again to see the modified result.

- `Never` (default) — webhook runs once
- `IfNeeded` — webhook runs again if later webhooks modified the object

**Ordering:**

Mutating webhooks run in alphabetical order of their names. To control order, name them carefully (e.g., `01-sidecar-injector`, `02-policy-injector`).

**Failure Modes:**

`failurePolicy` determines what happens if the webhook fails:

- `Fail` (recommended) — reject the request. Safer but can break the cluster if webhook is down.
- `Ignore` — proceed without the webhook. More resilient but bypasses your mutations.

**Critical: Don't Brick Your Cluster:**

If your webhook is in a Pod that the webhook itself mutates, you have a circular dependency. Workarounds:

1. Run webhook in `kube-system` namespace
2. Use `namespaceSelector` to exclude system namespaces
3. Use `failurePolicy: Ignore` for non-critical webhooks
4. Carefully tune timeouts

**Security Considerations:**

Mutating webhooks have **enormous power** — they can modify any aspect of the object. Be very careful with permissions:

- Restrict who can create MutatingWebhookConfigurations (cluster-admin only)
- Audit existing webhooks regularly
- Don't trust user-provided webhooks
- Use strict RBAC

A malicious mutating webhook could inject backdoors into every Pod.

**Validating Webhooks:**

Validating webhooks **cannot modify objects** — they only accept or reject. They run after mutating webhooks and schema validation. The same protocol but the response has no patch:

```json
{
  "response": {
    "uid": "abc-123",
    "allowed": false,
    "status": {
      "message": "Image must come from approved registry"
    }
  }
}
```

---

### 58. Explain kubelet Pod sync loop.

The kubelet Pod sync loop is the **core event loop** that keeps Pods in their desired state. Understanding it is essential for debugging Pod lifecycle issues.

**Overview:**

The kubelet continuously synchronizes the **desired state** (from the API server) with the **actual state** (containers running on the node). The sync loop runs frequently and is the heart of kubelet's operation.

**Sources of Pod Configuration:**

The kubelet receives Pod specs from three sources:

1. **API Server** — most common; via watch on Pods with `spec.nodeName == this-node`
2. **HTTP endpoint** — kubelet can poll an HTTP URL for Pod specs (rare)
3. **File** — Pod manifests in a directory (static Pods)

All three sources feed into a unified channel processed by the sync loop.

**The Main Loop:**

The kubelet runs a goroutine that processes events:

```
while running:
    select event from channels:
    - API server update (new/modified/deleted Pod)
    - Static Pod file change
    - HTTP poll result
    - Internal sync timer (every 1 minute)
    - PLEG event (Pod Lifecycle Event Generator)

    for each Pod event:
        call syncPod(pod)
```

**The syncPod Function:**

This is where the actual work happens. For each Pod:

**Step 1: Determine the action**

```
if Pod is being created:
    - validate
    - allocate resources
    - create network sandbox
    - start init containers
    - start containers

elif Pod is being updated:
    - identify what changed (image, config, etc.)
    - kill changed containers
    - create new containers with updated spec

elif Pod is being deleted:
    - run preStop hooks
    - send SIGTERM
    - wait for grace period
    - send SIGKILL if needed
    - clean up sandbox, network, volumes

elif Pod is running normally:
    - check container states
    - run probes
    - update status
```

**Step 2: Pod admission**

Before doing anything, the kubelet checks if the Pod can actually run on this node:
- Resource availability (CPU, memory)
- Node selector and affinity match
- Tolerations for taints
- Other admission checks

If admission fails, the Pod stays Pending and an event is generated.

**Step 3: Volume management**

For each volume in the Pod:
- For PVCs, wait for the volume to be attached (CSI ControllerPublish)
- Mount the volume on the node (CSI NodeStage + NodePublish)
- Make it available to containers

This happens asynchronously — the kubelet polls volume state.

**Step 4: Pull container images**

If images aren't cached:
- Authenticate to registries (using imagePullSecrets, IRSA, etc.)
- Pull images via the CRI ImageService
- Track image pull progress

Image pulling can take a long time, blocking Pod start.

**Step 5: Create Pod sandbox**

The pause container is created first. This:
- Creates network namespace
- Calls CNI to set up Pod networking (assign IP, configure routes)
- Sets up IPC, UTS namespaces
- Configures cgroups

**Step 6: Start init containers**

Init containers run sequentially:
- Start the init container
- Wait for it to exit
- Check exit code
- If success, move to next
- If failure, restart Pod (per restartPolicy)

**Step 7: Start application containers**

After all init containers succeed:
- Start each main container (in parallel)
- Run postStart hooks if defined
- Begin probes (startup, then liveness/readiness)

**Step 8: Update status**

The Pod's status is updated via PATCH to the API server:
- Container states (Waiting, Running, Terminated)
- Pod conditions (Ready, ContainersReady, Initialized, PodScheduled)
- Pod IP
- Restart counts

This status flows back through the API server to all watchers (Endpoint controller, Service routing, etc.).

**The Pod Lifecycle Event Generator (PLEG):**

PLEG is a separate goroutine that **observes container state changes** by polling the container runtime. When it detects a change (container crashed, exited, etc.), it generates an event that triggers syncPod.

PLEG was introduced because purely event-driven runtime communication was unreliable. By periodically reconciling, the kubelet catches state changes even if events were missed.

PLEG polls every 1 second by default. The `relist_period` setting controls this.

**Common PLEG Issues:**

`PLEG is not healthy` — the dreaded error meaning the runtime is slow to respond. Causes:
- Runtime under heavy load
- Disk I/O bottleneck
- Too many containers on the node
- Runtime bug

When PLEG is unhealthy, the kubelet reports the node as NotReady.

**Probe Workers:**

Each container with probes gets dedicated worker goroutines:
- One for the liveness probe
- One for the readiness probe
- One for the startup probe (until it succeeds)

These run independently of the main sync loop, ensuring probes execute on their configured periods.

**Workers and Concurrency:**

The kubelet uses workers to process Pods in parallel:

```
podWorkers map[types.UID]*podWorker
```

Each Pod has its own worker. Multiple Pods can be synced concurrently, but each Pod is processed serially.

**Sync Frequency:**

The kubelet syncs each Pod:
1. **Immediately** when an event arrives (API change, runtime event)
2. **Periodically** every `--sync-frequency` (default 1 minute)

Periodic syncs catch missed events and ensure eventual consistency.

**Status Updates:**

Pod status is updated:
1. **On state changes** — immediately when something changes
2. **Periodically** — every `--node-status-update-frequency` (default 10s)

This makes Pod state visible to controllers and other watchers.

**Pod Manager:**

Internally, the kubelet has a `PodManager` that tracks:
- The "current" desired state for each Pod
- The "secret" data (Secrets, ConfigMaps) referenced
- Mirror Pods (for static Pods)

The PodManager is the cached source of truth within kubelet.

**Static Pods:**

For static Pods (read from disk), the kubelet:
1. Watches the manifest directory
2. On change, parses the YAML
3. Creates a Pod object internally
4. Treats it like any other Pod for sync
5. Creates a mirror Pod in the API server for visibility

If the API server is unreachable, static Pods still run (this is how kubeadm bootstraps the control plane).

**Eviction Manager:**

A separate component watches node-level resource pressure (memory, disk, PIDs). When thresholds are exceeded, it triggers Pod evictions, calling into the sync loop to terminate Pods.

**Container Manager:**

Manages cgroups for QoS classes:
- Creates `kubepods.slice`, `kubepods-besteffort.slice`, etc.
- Updates cgroup parameters when Pods change
- Coordinates with CPU manager, memory manager, topology manager

**Common Sync Loop Issues:**

1. **Stuck in ContainerCreating** — usually CNI failure or image pull issue
2. **Stuck in Pending** — admission failure (no node fits, resource pressure)
3. **CrashLoopBackOff** — container fails to start, kubelet backs off restart attempts
4. **PLEG is not healthy** — runtime slow
5. **Failed mount** — volume issue, often CSI driver

**Debugging:**

```bash
# Kubelet logs are the primary source
journalctl -u kubelet -f

# Pod events show kubelet decisions
kubectl describe pod <pod>

# Container runtime status
crictl ps -a
crictl inspect <container-id>
```

---

### 59. How does container runtime interact with kubelet?

The kubelet communicates with the container runtime via the **Container Runtime Interface (CRI)** — a gRPC-based protocol that decouples kubelet from specific runtime implementations.

**Historical Context:**

Originally, kubelet had hardcoded support for Docker. As more runtimes emerged (rkt, etc.), this became unsustainable. CRI was introduced in v1.5 as an abstraction layer. The "dockershim" implemented CRI for Docker, but was removed in v1.24 — Docker users migrated to containerd or CRI-O.

**The CRI Architecture:**

```
kubelet (gRPC client)
    ↓
CRI gRPC over Unix socket
    ↓
Container Runtime (containerd, CRI-O, etc.)
    ↓
OCI Runtime (runc, crun, etc.)
    ↓
Linux kernel (namespaces, cgroups, ...)
```

**The CRI Has Two Services:**

**1. RuntimeService**

Manages Pod sandboxes and containers:

```protobuf
service RuntimeService {
    rpc Version(VersionRequest) returns (VersionResponse) {}

    // Sandbox (Pod) lifecycle
    rpc RunPodSandbox(...) returns (...) {}
    rpc StopPodSandbox(...) returns (...) {}
    rpc RemovePodSandbox(...) returns (...) {}
    rpc PodSandboxStatus(...) returns (...) {}
    rpc ListPodSandbox(...) returns (...) {}

    // Container lifecycle
    rpc CreateContainer(...) returns (...) {}
    rpc StartContainer(...) returns (...) {}
    rpc StopContainer(...) returns (...) {}
    rpc RemoveContainer(...) returns (...) {}
    rpc ContainerStatus(...) returns (...) {}
    rpc ListContainers(...) returns (...) {}

    // Container operations
    rpc UpdateContainerResources(...) returns (...) {}
    rpc ContainerStats(...) returns (...) {}
    rpc ExecSync(...) returns (...) {}
    rpc Exec(...) returns (...) {}
    rpc Attach(...) returns (...) {}
    rpc PortForward(...) returns (...) {}

    // Runtime info
    rpc Status(...) returns (...) {}
}
```

**2. ImageService**

Manages container images:

```protobuf
service ImageService {
    rpc ListImages(...) returns (...) {}
    rpc ImageStatus(...) returns (...) {}
    rpc PullImage(...) returns (...) {}
    rpc RemoveImage(...) returns (...) {}
    rpc ImageFsInfo(...) returns (...) {}
}
```

**Socket Location:**

The CRI runtime listens on a Unix domain socket. Kubelet's `--container-runtime-endpoint` flag specifies the path:

- containerd: `unix:///run/containerd/containerd.sock`
- CRI-O: `unix:///var/run/crio/crio.sock`

**Pod Creation Sequence:**

When a Pod is scheduled to a node, the kubelet executes:

**Step 1: Pull images (if needed)**

```
kubelet → ImageService.ImageStatus(nginx:1.21) → response
if not found:
    kubelet → ImageService.PullImage(nginx:1.21) → response with image ref
```

**Step 2: Create Pod sandbox**

```
kubelet → RuntimeService.RunPodSandbox({
    metadata: {name: "my-pod", uid: "...", namespace: "default"},
    hostname: "my-pod",
    log_directory: "/var/log/pods/...",
    dns_config: {...},
    port_mappings: [...],
    labels: {...},
    annotations: {...},
    linux: {
        cgroup_parent: "/kubepods.slice/...",
        security_context: {...}
    }
})

Runtime:
    - Creates a network namespace
    - Calls CNI plugins (ADD command)
    - Starts pause container
    - Returns sandbox ID

kubelet receives sandbox_id
```

**Step 3: Create each container**

For each init container, then each app container:

```
kubelet → RuntimeService.CreateContainer({
    pod_sandbox_id: "<sandbox-id>",
    config: {
        metadata: {name: "app"},
        image: {image: "nginx:1.21"},
        command: [...],
        args: [...],
        envs: [...],
        mounts: [...],
        labels: {...},
        log_path: "...",
        linux: {
            resources: {cpu_quota: ..., memory_limit_in_bytes: ...},
            security_context: {...}
        }
    },
    sandbox_config: {...}
})

Runtime creates container, returns container_id
```

**Step 4: Start the container**

```
kubelet → RuntimeService.StartContainer({container_id: "..."})

Runtime:
    - Joins sandbox's network and IPC namespaces
    - Sets up the container's own PID and mount namespaces
    - Configures cgroups (resource limits)
    - Calls runc or equivalent OCI runtime
    - Returns success

kubelet sees container running
```

**Status Updates:**

Periodically and on events, kubelet polls status:

```
kubelet → RuntimeService.ListContainers(...) → all containers
kubelet → RuntimeService.ContainerStatus(container_id) → detailed state
kubelet → RuntimeService.PodSandboxStatus(sandbox_id) → sandbox details
```

This information feeds into Pod status updates sent to the API server.

**Pod Lifecycle Event Generator (PLEG):**

Kubelet's PLEG component periodically calls ListContainers to detect state changes:

```
every 1 second:
    containers = RuntimeService.ListContainers(...)
    diff with previous list
    for each change:
        generate event → trigger syncPod
```

This is a polling-based reconciliation. Event-driven CRI exists (CRI Events RPC) but is less universally adopted.

**Stop and Remove:**

When a Pod is deleted:

```
1. kubelet sends preStop hook (via ExecSync if defined)
2. kubelet → RuntimeService.StopContainer({
       container_id: "...",
       timeout: grace_period_seconds
   })
   Runtime sends SIGTERM, waits, then SIGKILL

3. kubelet → RuntimeService.RemoveContainer({container_id: "..."})

After all containers removed:
4. kubelet → RuntimeService.StopPodSandbox({sandbox_id: "..."})
   Runtime:
       - Calls CNI to tear down network
       - Stops pause container

5. kubelet → RuntimeService.RemovePodSandbox({sandbox_id: "..."})
   Runtime cleans up sandbox resources
```

**Exec, Attach, PortForward:**

For `kubectl exec`, `kubectl attach`, `kubectl port-forward`:

```
kubectl → API server → kubelet → RuntimeService.Exec({...})
Runtime returns URL of an exec endpoint
Streaming session established between kubelet and runtime
kubelet proxies between API server and runtime
API server proxies between kubectl and kubelet
```

**Image Authentication:**

When pulling images:
```
kubelet → ImageService.PullImage({
    image: {image: "private.registry.com/app:v1"},
    auth: {
        username: "...",
        password: "...",
        // OR
        identity_token: "...",
        // OR
        registry_token: "..."
    }
})
```

Credentials come from:
- `imagePullSecrets` on the Pod or ServiceAccount
- Node-level credential providers (kubelet `--image-credential-provider-config`)
- IRSA / Workload Identity (cloud-specific)

**Resource Management:**

The kubelet specifies cgroup parameters when creating containers:

```
linux: {
    resources: {
        cpu_period: 100000,
        cpu_quota: 50000,           // 0.5 cores
        cpu_shares: 512,             // proportional weight
        memory_limit_in_bytes: 1073741824,  // 1 GiB
        oom_score_adj: 1000,         // QoS-based
        cpu_set_cpus: "0-3",         // CPU pinning if enabled
        cpu_set_mems: "0",           // NUMA pinning
        hugepage_limits: [...]
    }
}
```

The runtime translates these into cgroup writes.

**Updating Resources:**

For Vertical Pod Autoscaler in-place updates:

```
kubelet → RuntimeService.UpdateContainerResources({
    container_id: "...",
    linux: {
        cpu_period: ...,
        cpu_quota: ...,
        memory_limit_in_bytes: ...
    }
})
```

Runtime updates cgroups without restarting the container.

**CRI Statistics:**

For monitoring:
```
kubelet → RuntimeService.ContainerStats(container_id)
Response: CPU usage, memory usage, network stats, etc.

kubelet → RuntimeService.ListContainerStats(filter)
Bulk stats for many containers.

kubelet → ImageService.ImageFsInfo(...)
Disk usage by container images
```

This data flows to:
- Pod status (CPU/memory usage)
- Metrics-server (for `kubectl top` and HPA)
- Node monitoring

---

### 60. Explain CRI-O vs containerd.

Both CRI-O and containerd are container runtimes that implement the Container Runtime Interface (CRI) for Kubernetes. They're the two dominant choices since dockershim's removal.

**containerd:**

**Origin and Design:**

containerd started as a component of Docker, extracted and donated to CNCF in 2017. It's a **general-purpose container runtime** — used not only by Kubernetes but also by Docker itself, Rancher, and other tools.

**Architecture:**

containerd is intentionally minimal. It provides:
- Image management (pull, store, push)
- Container lifecycle (create, start, stop, delete)
- Snapshot management (for layered filesystems)
- Network namespace coordination (delegates to CNI)
- Plugin system for extensions

For Kubernetes, the **CRI plugin** (built into containerd) translates between CRI gRPC calls and containerd's native API.

```
kubelet → CRI gRPC → containerd's CRI plugin → containerd core → runc → kernel
```

**Key Features:**

- **Single binary** — one daemon (containerd) handles everything
- **Plugin architecture** — extensible via plugins (CRI plugin, snapshot drivers, runtime plugins)
- **Snapshotter system** — supports overlayfs, btrfs, zfs, native, others
- **Multi-tenant** — supports namespaces (different from Kubernetes namespaces)
- **OCI compliant** — uses OCI runtime (runc by default)
- **Wide ecosystem** — used by EKS, GKE, AKS as default

**Configuration:**

`/etc/containerd/config.toml`:

```toml
version = 2

[plugins."io.containerd.grpc.v1.cri"]
  sandbox_image = "registry.k8s.io/pause:3.9"

[plugins."io.containerd.grpc.v1.cri".containerd]
  default_runtime_name = "runc"

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  runtime_type = "io.containerd.runc.v2"

[plugins."io.containerd.grpc.v1.cri".registry.configs."docker.io".auth]
  username = "..."
  password = "..."
```

**CLI Tools:**

- `ctr` — low-level containerd CLI (debugging, advanced operations)
- `nerdctl` — Docker-compatible CLI for containerd (drop-in replacement)
- `crictl` — Kubernetes-focused CRI CLI (works with any CRI runtime)

**CRI-O:**

**Origin and Design:**

CRI-O was created by Red Hat specifically as a **Kubernetes-only container runtime**. Started in 2016, originally as "OCID." The philosophy: a minimal runtime designed exclusively for Kubernetes' needs, nothing more.

**Architecture:**

CRI-O is more single-purpose than containerd:

```
kubelet → CRI gRPC → CRI-O daemon → conmon → runc → kernel
```

- **No general-purpose CLI for users** — CRI-O is meant only for kubelet
- **conmon** — a separate process that monitors each container (handles I/O, exit codes)
- **OCI compliant** — uses runc by default, supports others

**Key Features:**

- **Designed for Kubernetes** — every feature exists because Kubernetes needs it
- **Minimal footprint** — fewer features means smaller attack surface
- **Tight CRI alignment** — closely tracks Kubernetes CRI versions
- **Used by OpenShift** — Red Hat's enterprise Kubernetes default
- **Uses libpod libraries** — shared with Podman
- **conmon design** — one conmon per container; survives daemon restart

**Configuration:**

`/etc/crio/crio.conf`:

```toml
[crio]
storage_driver = "overlay"

[crio.image]
pause_image = "registry.k8s.io/pause:3.9"

[crio.runtime]
default_runtime = "runc"
log_level = "info"

[crio.runtime.runtimes.runc]
runtime_path = "/usr/bin/runc"
runtime_type = "oci"

[crio.network]
cni_default_network = ""
plugin_dirs = ["/opt/cni/bin/"]
```

**CLI Tools:**

- `crictl` — Kubernetes CRI CLI (same as containerd)
- No equivalent of `ctr` or `nerdctl` — CRI-O is meant for kubelet only
- `podman` — separate tool but uses related libraries

**Comparison Table:**

| Aspect | containerd | CRI-O |
|--------|-----------|-------|
| Primary purpose | General-purpose | Kubernetes-only |
| Maintainer | CNCF (originally Docker) | Red Hat / Kubernetes SIG |
| First release | 2016 | 2016 |
| Default in | EKS, GKE, AKS, k3s | OpenShift |
| User CLI | nerdctl, ctr | crictl only |
| Plugin architecture | Yes, extensive | Limited |
| Image format | OCI, Docker | OCI, Docker |
| OCI runtime | runc (default), customizable | runc (default), customizable |
| Memory footprint | Slightly larger | Slightly smaller |
| Performance | Comparable | Comparable |
| Daemon model | Single daemon | Daemon + conmon per container |
| Use outside K8s | Yes (Docker, others) | No |
| Maturity | Very mature | Mature |

**Performance:**

Benchmarks generally show comparable performance:
- Pod startup time: similar
- Image pull speed: similar (limited by network)
- Container start latency: similar
- Steady-state CPU/memory: containerd slightly less overhead in some tests

Differences are usually negligible in practice.

**Operational Differences:**

**containerd:**
- More tooling for non-Kubernetes use
- Larger community and ecosystem
- Better for developers who use containers outside K8s
- More configuration options

**CRI-O:**
- Cleaner separation from Kubernetes
- conmon survives daemon restart — containers keep running if CRI-O crashes
- Tighter alignment with Kubernetes versions
- Better for Kubernetes-only deployments

**Choosing Between Them:**

**Choose containerd if:**
- Using EKS, GKE, AKS, or most managed Kubernetes (it's the default)
- You want broader tooling support
- You need containerd's plugin ecosystem
- You also use containers outside Kubernetes

**Choose CRI-O if:**
- Using OpenShift (it's the default)
- You want strict Kubernetes alignment
- You prefer a minimal runtime
- You have strict separation between K8s and other container use

**For most users:** containerd is the safer choice due to broader ecosystem support and being the default in most managed services.

**Other CRI Runtimes:**

While containerd and CRI-O dominate, others exist:

- **Mirantis Container Runtime (formerly Docker EE)** — fork of dockershim
- **gVisor (runsc)** — userspace kernel for sandbox isolation
- **Kata Containers** — VM-based containers (used via containerd or CRI-O as a runtime handler)
- **Firecracker (via firecracker-containerd)** — microVMs, used by AWS Fargate

These are typically used **alongside** containerd or CRI-O via RuntimeClass.

**Migration Considerations:**

When migrating between runtimes:
- Image format is OCI-standard — works across runtimes
- CRI is standardized — kubelet doesn't care which runtime
- Configuration differs — `/etc/containerd/config.toml` vs `/etc/crio/crio.conf`
- CLI tools differ — adjust scripts using `docker`, `ctr`, etc.

Most migrations are smooth because of CRI's standardization.

---

### 61. What happens during container creation?

Container creation in Kubernetes is a **multi-step process** involving the kubelet, container runtime, OCI runtime, and Linux kernel. Understanding the full sequence helps debug startup issues.

**The Full Sequence:**

Let's trace what happens when a Pod with one container starts on a node.

**Step 1: kubelet Receives the Pod Spec**

The Pod is assigned to this node (spec.nodeName set). The kubelet's watch sees this and triggers syncPod for the new Pod.

**Step 2: Pod Admission**

The kubelet evaluates whether the Pod can run:
- Sufficient resources (CPU, memory) available?
- Node selector matches?
- Tolerations cover all taints?
- Custom admission handlers (some are local to kubelet)

If admission fails, the Pod stays Pending with an event explaining why.

**Step 3: Volume Setup**

For each volume:
- **ConfigMaps/Secrets** — kubelet fetches from API, prepares in tmpfs
- **EmptyDir** — creates a directory on the node
- **PersistentVolumeClaim** — coordinates with CSI driver:
  1. Wait for volume to be attached to node (ControllerPublishVolume)
  2. Stage the volume (NodeStageVolume) — mount to global path
  3. Publish the volume (NodePublishVolume) — bind mount into Pod's path
- **hostPath** — directly accessible from node

Volume setup can block container start. Common failure modes: PVC pending, CSI driver issues, file mode mismatches.

**Step 4: Image Pull**

For each container in the Pod:

```
1. Check if image is already cached:
   ImageService.ImageStatus({image: "nginx:1.21"})

2. If not cached, pull:
   ImageService.PullImage({
       image: {image: "nginx:1.21"},
       auth: <credentials from imagePullSecrets>,
       sandbox_config: <pod info>
   })

3. Runtime pulls layers, verifies signatures (if enabled),
   extracts to snapshotter
```

Image pull policy:
- **Always** — always pull
- **IfNotPresent** (default) — pull if not cached
- **Never** — fail if not cached (offline mode)

**Step 5: Create Pod Sandbox**

The kubelet creates the Pod sandbox (the pause container plus shared namespaces):

```
RuntimeService.RunPodSandbox({
    metadata: {name, uid, namespace, attempt},
    hostname,
    log_directory: "/var/log/pods/<ns>_<name>_<uid>/",
    dns_config,
    port_mappings,
    labels,
    annotations,
    linux: {
        cgroup_parent: "/kubepods.slice/kubepods-<qos>.slice/kubepods-<qos>-pod<uid>.slice",
        security_context: {namespace_options, seccomp, selinux, ...},
        sysctls
    }
})
```

The runtime executes:

1. **Create cgroups** — pod-level cgroup hierarchy
2. **Pull pause image** if not cached
3. **Create the pause container**:
   - Mount the rootfs (pause image, very small)
   - Create namespaces:
     - Network namespace (new)
     - IPC namespace (new)
     - UTS namespace (new)
     - Mount namespace (new)
     - PID namespace (new, or shared with host based on config)
     - User namespace (if enabled)
   - Apply security context (seccomp, capabilities, SELinux)
   - Start the pause process (just calls `pause()` syscall)

4. **Network setup (CNI)**:
   - For each network in `/etc/cni/net.d/`:
     - Execute the CNI plugin binary with command `ADD`
     - Pass environment variables (CNI_COMMAND, CNI_CONTAINERID, CNI_NETNS, etc.)
     - Pass JSON config on stdin
     - CNI plugin creates veth pair, configures IPAM, sets up routes
   - For port mappings, the portmap plugin adds iptables NAT rules

5. **Return sandbox ID** to kubelet

**Step 6: Run Init Containers (Sequentially)**

For each init container, in order:

```
1. RuntimeService.CreateContainer({
       pod_sandbox_id,
       config: {
           metadata: {name},
           image: {image: "init-image"},
           command,
           args,
           envs,
           mounts,
           working_dir,
           labels,
           annotations,
           log_path,
           linux: {resources, security_context}
       },
       sandbox_config
   })

2. RuntimeService.StartContainer({container_id})

3. Wait for container to exit
   - Poll via ContainerStatus
   - Or wait for PLEG event

4. Check exit code:
   - 0: success, proceed to next init container
   - non-zero: failure, restart Pod per restartPolicy
```

Init containers can have postStart hooks but typically don't (they're short-lived).

**Step 7: Create Application Containers**

After all init containers succeed, app containers are created (in parallel):

```
For each container in spec.containers:
    RuntimeService.CreateContainer({...similar to init...})
```

The CreateContainer step in the runtime:

1. **Generate OCI runtime spec**:
   - Translate CRI config to OCI runtime spec JSON
   - Set rootfs (from image)
   - Specify namespaces to join (from sandbox)
   - Set resources (cgroups)
   - Set mounts
   - Set environment variables
   - Set capabilities, seccomp profile

2. **Prepare rootfs**:
   - Get image layers from snapshotter
   - Create overlay filesystem (typical with overlayfs):
     - Lower dirs: image layers (read-only)
     - Upper dir: container's writable layer
     - Merged: what the container sees as `/`

3. **Bind mounts**:
   - ConfigMaps and Secrets (from volumes)
   - EmptyDir
   - Persistent volumes
   - Service account token (projected volume)

4. **Resolve image command and args**:
   - If Pod doesn't override, use image's CMD/ENTRYPOINT
   - Apply args from Pod spec

5. **Return container_id** (container exists but not running yet)

**Step 8: Start Application Containers**

```
For each created container:
    RuntimeService.StartContainer({container_id})
```

The StartContainer step:

1. **Call OCI runtime** (typically runc):
   - `runc create` then `runc start`
   - Or `runc run` for combined

2. **runc executes**:
   - Sets up cgroups (CPU, memory, blkio, etc.)
   - Joins the sandbox's network, IPC, UTS namespaces
   - Creates new PID and mount namespaces (or joins sandbox's PID if shared)
   - Applies user namespace mapping (if configured)
   - Sets up capabilities, seccomp filter
   - Sets file system mounts (rootfs, bind mounts)
   - Applies AppArmor or SELinux profile
   - Sets environment variables
   - Executes the container's command via `execve()`

3. **Container process starts** — this is the actual application

4. **conmon (if CRI-O) or shim (containerd)**:
   - Separate process that owns the container's stdio
   - Captures logs to log files
   - Waits for container exit
   - Reports exit status back

**Step 9: postStart Hook (If Defined)**

If the container has a postStart hook:

```yaml
lifecycle:
  postStart:
    exec:
      command: ["sh", "-c", "echo started > /tmp/marker"]
```

The kubelet executes the hook **synchronously after start**. The container is technically running, but the kubelet may consider it not fully ready until postStart completes.

Hooks can be exec, httpGet, or tcpSocket. Hook failure restarts the container.

**Step 10: Probes Begin**

The kubelet starts probe workers:

1. **Startup probe** (if defined) — runs first, until success or failure
2. **Liveness probe** — begins after startup probe (or immediately if no startup)
3. **Readiness probe** — begins same time as liveness

Each probe runs in its own goroutine on its own period.

**Step 11: Status Reporting**

The kubelet continuously updates Pod status:

```yaml
status:
  phase: Running
  conditions:
  - type: PodScheduled
    status: "True"
  - type: Initialized
    status: "True"
  - type: ContainersReady
    status: "True"
  - type: Ready
    status: "True"
  containerStatuses:
  - name: app
    image: nginx:1.21
    imageID: docker.io/library/nginx@sha256:...
    containerID: containerd://abc123...
    ready: true
    restartCount: 0
    state:
      running:
        startedAt: "2024-01-15T10:00:00Z"
```

The status flows to the API server, then to:
- Endpoint controller (adds Pod to Service endpoints)
- Anyone watching Pods

**Step 12: Traffic Begins Flowing**

Once the readiness probe passes:
- EndpointSlice is updated
- kube-proxy updates iptables/IPVS on all nodes
- Traffic to the Service starts being routed to this Pod

**Total Time:**

For a simple Pod:
- Already-cached image: 1-3 seconds
- New image (small): 5-15 seconds
- New image (large, ~1 GiB): 30-120 seconds
- Plus init container time
- Plus startup probe time

**Common Failures During Creation:**

1. **ErrImagePull / ImagePullBackOff** — can't pull image (auth, network, not found)
2. **CreateContainerError** — runtime can't create container (invalid spec, resource issues)
3. **RunContainerError** — container fails to start (binary not found, permission denied)
4. **CrashLoopBackOff** — container starts but exits immediately
5. **Init:Error** — init container failed
6. **PodInitializing** — stuck in init container that's slow
7. **ContainerCreating** stuck — usually CNI or volume mount issue

---

### 62. Explain image pull workflow.

The image pull workflow is more complex than it appears, involving authentication, manifest fetching, layer downloading, deduplication, and security checks.

**High-Level Flow:**

```
kubelet → CRI ImageService → containerd/CRI-O → Registry → Local storage
```

**Step 1: Determine If Pull Needed**

Based on `imagePullPolicy`:
- **Always** — always check for newer image
- **IfNotPresent** (default for tagged images) — only pull if not cached
- **Never** — fail if not present

For `imagePullPolicy: Always`, kubelet asks the runtime if the local image has the same digest as the registry's version. If different, pull is required.

For tags like `latest` or no tag, default is `Always`. For specific tags (e.g., `nginx:1.21`), default is `IfNotPresent`. For digests (e.g., `nginx@sha256:...`), pull is only needed if not present.

**Step 2: Resolve Authentication**

The kubelet collects credentials from multiple sources:

**Priority Order:**

1. **Node's docker config** — `/root/.docker/config.json` (if exists)
2. **Cloud provider credential helpers** — kubelet's credential provider plugins:
   - For ECR (AWS): IAM role credentials
   - For GCR (GCP): service account
   - For ACR (Azure): managed identity
3. **Pod's `imagePullSecrets`**:
   ```yaml
   spec:
     imagePullSecrets:
     - name: my-registry-secret
   ```
4. **ServiceAccount's `imagePullSecrets`** — auto-applied to Pods using that SA

The runtime uses the first matching credential.

**Credential Provider Plugins (modern approach):**

```yaml
# kubelet config
imageCredentialProviderConfig: /etc/kubernetes/credential-providers.yaml
imageCredentialProviderBinDir: /usr/local/bin/credential-providers
```

```yaml
# /etc/kubernetes/credential-providers.yaml
apiVersion: kubelet.config.k8s.io/v1
kind: CredentialProviderConfig
providers:
- name: ecr-credential-provider
  matchImages:
  - "*.dkr.ecr.*.amazonaws.com"
  defaultCacheDuration: "12h"
  apiVersion: credentialprovider.kubelet.k8s.io/v1
```

This avoids storing static credentials anywhere — the binary fetches credentials dynamically using the node's IAM role.

**Step 3: Parse Image Reference**

```
nginx:1.21
↓
registry-1.docker.io/library/nginx:1.21
```

Default registry: `docker.io`. If no registry in the reference, prepended. If no organization, `library/` is prepended.

Other examples:
- `gcr.io/google-containers/nginx:1.21` → as-is
- `nginx@sha256:abc...` → fetched by digest
- `my-registry.com:5000/team/app:v1` → as-is

**Step 4: Authenticate to Registry**

The runtime sends an initial request:

```
GET https://registry-1.docker.io/v2/
```

The registry responds with a 401 and a `Www-Authenticate` header:

```
Www-Authenticate: Bearer realm="https://auth.docker.io/token",service="registry.docker.io",scope="repository:library/nginx:pull"
```

The runtime then requests a token:

```
GET https://auth.docker.io/token?service=registry.docker.io&scope=repository:library/nginx:pull
Authorization: Basic <base64-credentials>
```

Response includes a bearer token used for subsequent requests.

**Step 5: Fetch Manifest**

```
GET https://registry-1.docker.io/v2/library/nginx/manifests/1.21
Accept: application/vnd.oci.image.manifest.v1+json
Authorization: Bearer <token>
```

For multi-architecture images, this returns a **manifest list** (also called an index):

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.index.v1+json",
  "manifests": [
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "size": 1234,
      "digest": "sha256:abc...",
      "platform": {"architecture": "amd64", "os": "linux"}
    },
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "size": 1234,
      "digest": "sha256:def...",
      "platform": {"architecture": "arm64", "os": "linux"}
    }
  ]
}
```

The runtime selects the manifest matching the node's architecture.

**Step 6: Fetch Single Manifest**

```
GET https://registry-1.docker.io/v2/library/nginx/manifests/sha256:abc...
```

Returns:

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "size": 7023,
    "digest": "sha256:config..."
  },
  "layers": [
    {"mediaType": "...", "size": 28000000, "digest": "sha256:layer1..."},
    {"mediaType": "...", "size": 5000000, "digest": "sha256:layer2..."},
    {"mediaType": "...", "size": 1000000, "digest": "sha256:layer3..."}
  ]
}
```

**Step 7: Download Config**

```
GET https://registry-1.docker.io/v2/library/nginx/blobs/sha256:config...
```

The config contains:
- Entrypoint, CMD
- Environment variables
- Working directory
- Exposed ports
- Volumes
- User
- Labels
- Image history

**Step 8: Download Layers in Parallel**

The runtime checks which layers are already in local storage (by digest). Missing layers are downloaded in parallel:

```
GET https://registry-1.docker.io/v2/library/nginx/blobs/sha256:layer1...
GET https://registry-1.docker.io/v2/library/nginx/blobs/sha256:layer2...
GET https://registry-1.docker.io/v2/library/nginx/blobs/sha256:layer3...
```

Layers are content-addressable (by SHA256 digest). If multiple images share a layer, it's only stored once on disk.

**Layer Deduplication:**

If 100 Pods use images derived from `nginx:base`, the base layers are stored once on the node. Each Pod's container gets an overlay on top.

**Step 9: Verify Layer Integrity**

Each downloaded layer's SHA256 is computed and compared against the digest in the manifest. Mismatch means corruption — retry or fail.

**Step 10: Extract Layers**

Layers are typically gzipped tar archives. The runtime extracts each layer to the snapshotter's storage:

- **overlay snapshotter** (default for containerd) — uses overlayfs
- **btrfs snapshotter** — uses btrfs subvolumes
- **zfs snapshotter** — uses ZFS datasets

Each extracted layer becomes a snapshot. When creating a container, snapshots are stacked to form the container's rootfs.

**Step 11: Image Status Recorded**

The image is now available locally:
```
$ crictl images
IMAGE                    TAG     IMAGE ID            SIZE
docker.io/library/nginx  1.21    sha256:abc...       142MB
```

**Image Pull Optimization Techniques:**

**1. Pre-pulling:**

Tools like **kraken** (Uber), **dragonfly** (Alibaba), or **spegel** distribute images peer-to-peer across nodes, dramatically speeding up large-scale Pod rollouts.

**2. Image Streaming:**

containerd can stream images — start the container before fully downloading, fetching data on demand. Reduces startup latency for large images.

**3. Image Caching:**

Container registries can be cached locally:
- **AWS ECR pull-through cache**
- **GCP Artifact Registry remote repositories**
- **Local Docker registry as mirror**

**4. Smaller Images:**

Multi-stage builds, distroless base images, and Alpine reduce pull time.

**5. ImagePullPolicy: IfNotPresent for stable tags:**

For immutable tags (`v1.2.3`), IfNotPresent means the pull only happens once per node.

**Common Issues:**

**ErrImagePull:**
- Authentication failure
- Image doesn't exist
- Network issue reaching registry

**ImagePullBackOff:**
- After repeated ErrImagePull, kubelet backs off (exponential)
- Initial backoff is short, grows to 5 minutes max

**Manifest issues:**
- "manifest for X not found" — wrong tag or registry
- "no matching manifest for linux/amd64" — image doesn't support this arch

**Disk pressure:**
- Node out of space
- Image too large to fit
- Garbage collection hasn't freed enough space

**Rate limiting:**
- Docker Hub rate limits anonymous pulls
- Affects clusters pulling many images without authentication

**Best Practices:**

1. **Use specific tags or digests** — never `latest` in production
2. **Use a private mirror** for public images (Docker Hub rate limits)
3. **Pull through cache** for cloud registries
4. **Pre-pull frequently used images** on nodes
5. **Configure imagePullSecrets correctly** at ServiceAccount level
6. **Monitor image pull duration** — long pulls slow Pod startup

---

### 63. What is image garbage collection?

Image garbage collection is the kubelet's mechanism for **automatically removing unused container images** from a node to prevent disk space exhaustion.

**The Problem:**

Container images can be large (hundreds of MB to several GB). As Pods come and go on a node, the local image cache grows. Without cleanup, nodes eventually run out of disk space — causing image pull failures and eventually node-level issues.

**The Solution:**

The kubelet periodically scans local images and deletes those:
1. Not currently used by any running container
2. Older than the minimum age threshold
3. Required to bring disk usage below the high threshold

**Configuration:**

```yaml
# kubelet config (kubelet-config.yaml)
imageMinimumGCAge: 2m                    # don't GC images younger than this
imageGCHighThresholdPercent: 85          # start GC at this disk usage
imageGCLowThresholdPercent: 80           # stop GC at this disk usage
```

**The Algorithm:**

The kubelet runs the image GC routine every minute (controlled by `imageMinimumGCAge`). When triggered:

1. **Check disk usage** — calculate current usage of the image filesystem
2. **If above high threshold** (e.g., 85%), start GC
3. **List all images** on the node
4. **Sort by age** — oldest unused images first
5. **For each image** (oldest first):
   - Skip if used by any container (running, stopped, or paused)
   - Skip if younger than `imageMinimumGCAge`
   - Otherwise, delete it
6. **Continue until** disk usage drops below low threshold (e.g., 80%)

**Why Two Thresholds?**

The hysteresis (gap between high and low) prevents GC from running constantly. Without it, deleting one image would put you below the threshold, but new image pulls would push you back above immediately. The 5% gap lets you operate in a stable range.

**What Gets Considered "In Use":**

An image is considered in use if:
- Any running container uses it
- Any stopped container (in the kubelet's tracking) uses it
- Recently removed containers — there's a brief window

This conservative behavior prevents GC from removing an image that's about to be used by a restarting container.

**Pinned Images:**

The kubelet has a list of images it should never delete:
- The pause/sandbox image (pulls it once, uses for every Pod)
- Images that match `pin` configuration

**Different Filesystems:**

Modern containerd setups can split image storage from container writable layers. Two filesystems are tracked:

- **imagefs** — where image layers are stored
- **nodefs** — where container writable layers (and everything else) are stored

GC thresholds apply to imagefs primarily. Eviction (which is separate) considers both.

**Triggers Beyond Time-Based GC:**

The kubelet also runs image GC in response to:

1. **Disk pressure** — when nodefs or imagefs is approaching eviction thresholds, GC runs aggressively
2. **Manual trigger** — restarting kubelet causes a check

**What If GC Can't Free Enough?**

If even after deleting all unused images, the disk is still full:
- The kubelet reports `DiskPressure` condition on the node
- The Node Controller adds a `node.kubernetes.io/disk-pressure:NoSchedule` taint
- New Pods aren't scheduled here
- Existing Pods using lots of disk may be evicted

**Practical Considerations:**

**1. Reserved Space:**

The disk threshold percentages are relative to **total disk size**, not free space. So 85% high threshold on a 100 GiB disk means GC starts when 85 GiB are used — leaving 15 GiB for other operations.

For nodes with very large disks, you might want different thresholds:

```yaml
imageGCHighThresholdPercent: 90    # only worry above 90%
imageGCLowThresholdPercent: 85     # stop at 85%
```

**2. Image Churn:**

If your workload pulls many different images frequently (e.g., CI runners pulling different builder images), you may experience:
- Higher GC activity
- Slower Pod starts (frequent re-pulls)
- More registry traffic

Consider larger node disks or pre-pulling commonly-used images.

**3. Slow GC on Large Image Stores:**

GC can be slow with thousands of cached images. Deleting an image involves:
- Removing all of its layers
- Updating snapshotter state
- Sometimes coalescing free space

This can take seconds per image. During heavy GC, kubelet may seem slow.

**4. Snapshotter Implications:**

Different snapshotters have different GC characteristics:
- **overlayfs** — straightforward, file removal
- **btrfs / zfs** — subvolume operations, can be slower but more efficient

**5. Image References in CronJobs:**

Be careful with images used by Jobs/CronJobs that haven't run recently. They might be GC'd, then need re-pulling when the Job runs. For frequently-run Jobs, ensure adequate disk to avoid this.

**Monitoring:**

Key metrics:
- `kubelet_image_pull_duration_seconds` — image pull times (longer = more GC churn likely)
- Node disk usage (`node_filesystem_usage_bytes` from node-exporter)
- `DiskPressure` condition on nodes
- kubelet logs showing GC actions

**Logs:**

```
I0115 10:00:00.123  kubelet: Image GC: freed 5.2 GB
I0115 10:00:00.124  kubelet: Image GC: removed image nginx:1.20 (250 MB)
```

**Manual Cleanup:**

If you need to free space immediately:

```bash
# On the node
crictl rmi --prune    # remove unused images
```

This is essentially what kubelet's GC does.

**Other Cleanup Tasks:**

The kubelet also garbage collects:

**1. Containers (separately from images):**

```yaml
maxContainerCount: -1                # per-Pod (deprecated)
maxPerPodContainerCount: 1           # max stopped containers per Pod
```

Stopped containers eventually removed.

**2. Logs:**

Container logs (`/var/log/pods/*/`) are managed but actual log rotation depends on the runtime configuration.

**3. Pod sandboxes:**

When a Pod is deleted, the sandbox container is removed.

**Best Practices:**

1. **Right-size node disks** — too small means frequent GC churn; too large is wasteful
2. **Pre-pull common base images** so they don't get GC'd repeatedly
3. **Monitor disk usage** with alerts before hitting thresholds
4. **Use a registry cache** to reduce pull traffic when GC removes images
5. **Tune thresholds** based on workload patterns
6. **Consider image streaming or P2P distribution** for large clusters with many images

---

### 64. Explain node heartbeats.

Node heartbeats are how the kubelet **signals to the control plane that a node is alive and healthy**. The mechanism has evolved significantly over Kubernetes versions for scalability.

**Why Heartbeats Exist:**

The control plane needs to know:
- Which nodes are available for scheduling
- When a node has died (so Pods can be rescheduled)
- Current state of each node (resources, conditions)

The kubelet must constantly tell the control plane "I'm still here." If it stops, the control plane assumes the node has failed.

**Evolution of Heartbeats:**

**Original Approach (Pre-1.14):**

The kubelet updated the **Node object's status** every 10 seconds. This included:
- Capacity (CPU, memory, etc.)
- Allocatable
- Conditions (Ready, MemoryPressure, etc.)
- Node info (kernel version, OS, etc.)
- Addresses
- Timestamps

Every update was a full PATCH on the Node object. Even small updates wrote significant data to etcd.

**Scaling Problem:**

In large clusters (1000+ nodes), this approach caused:
- Heavy etcd write load (each node × every 10s = significant write rate)
- Increased API server CPU usage
- etcd database growth (every update is a new revision)
- Watch event storm (every Node update notifies all watchers)

**The Solution: Node Lease Objects**

In Kubernetes 1.14, the **Lease API** was introduced for lightweight heartbeats. A Lease object is a minimal resource designed for this purpose:

```yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: node-1
  namespace: kube-node-lease
spec:
  holderIdentity: node-1
  leaseDurationSeconds: 40
  renewTime: "2024-01-15T10:00:00Z"
```

**The Dual Heartbeat Approach:**

In modern Kubernetes, there are **two types of heartbeats**:

**1. Node Lease (lightweight, frequent):**
- Updated every 10 seconds by default
- Stored in `kube-node-lease` namespace
- Just a renewTime update — minimal data
- Cheap to write, cheap to watch

**2. Node Status Updates (heavyweight, infrequent):**
- Updated every 5 minutes by default (or on significant changes)
- Stored as the Node object's status
- Contains full state (resources, conditions, info)
- Expensive to write

**How They Work Together:**

The Node Controller monitors both:

- **Lease checking** — primary mechanism for fast failure detection
- **Status checking** — secondary, for detailed health information

If the Lease isn't renewed within `--node-monitor-grace-period` (default 40s), the node is considered unhealthy. The actual Node status update can be much slower without triggering failure detection.

**Configuration:**

kubelet config:
```yaml
nodeStatusUpdateFrequency: "5m"     # how often to update Node status (full)
nodeLeaseDurationSeconds: 40        # lease duration
nodeStatusReportFrequency: "5m"     # when to report status anyway
```

Controller manager:
```bash
--node-monitor-period=5s            # how often to check Node state
--node-monitor-grace-period=40s     # how long until considered unhealthy
--pod-eviction-timeout=5m           # how long until evicting Pods
```

**The Lease Update Process:**

Every 10 seconds:
```go
lease := getCurrentLease()
lease.Spec.RenewTime = now
updateLease(lease)
```

This is a simple PATCH — etcd writes are minimal, watches don't fire heavily (Lease changes only matter to the Node Controller).

**Status Updates:**

Every 5 minutes, OR when conditions change significantly:
```go
status := computeNodeStatus()       // gather all the data
patchNode(status)
```

Significant changes that trigger immediate updates:
- Node becomes Ready or NotReady
- Pressure conditions change
- Capacity changes (rare)
- IP address changes

**Failure Detection:**

The Node Controller's logic:

```
every --node-monitor-period (5s):
    for each Node:
        lease = getLease(node.name)
        if lease == nil or (now - lease.RenewTime) > --node-monitor-grace-period:
            mark node as Ready=Unknown
            taint node with node.kubernetes.io/unreachable:NoExecute
            schedule Pod eviction (after toleration period)
```

**Why Lease Made Things Better:**

Before Lease (Status-only heartbeats):
- 1000 nodes × 6 updates/min = 6000 writes/min to Node status
- Each write: significant data, full Node object
- etcd write load: substantial
- Watch events: every update notifies every watcher of Nodes

After Lease:
- 1000 nodes × 6 lease updates/min = 6000 lightweight writes to Lease
- Plus 1000 × 0.2 Node status updates/min = 200 heavier writes
- Lease writes are tiny — less etcd load
- Lease watches are typically only by the Node Controller — less fan-out

The improvement was approximately **10-100x reduction** in heartbeat-related load.

**Components Involved:**

**Kubelet:**
- Updates Lease every 10s (default)
- Updates Node status every 5min (default) or on changes

**API Server:**
- Validates and persists both Lease and Node updates
- Manages watches

**etcd:**
- Stores Lease and Node objects
- Lease changes generate compactable history (kept briefly)

**Node Controller:**
- Watches Leases for fast detection
- Watches Nodes for detailed status
- Takes action (taints, eviction) based on Lease expiration

**Practical Implications:**

**1. Heartbeat Tuning:**

For latency-sensitive scenarios (fast failure detection):
```yaml
nodeLeaseDurationSeconds: 20
```

```bash
--node-monitor-grace-period=20s
```

This detects failures in ~20 seconds instead of 40, at the cost of more false positives during network blips.

**2. Heartbeat-Related Failures:**

If you see nodes flapping between Ready and NotReady:
- Network instability between node and API server
- API server overload (lease updates rejected)
- etcd performance issues
- kubelet bugs

**3. Disconnected Nodes:**

If a node loses connectivity to the API server but is otherwise fine:
- Lease isn't updated
- Node Controller marks it Unknown after 40s
- Pods are evicted after 5min
- But containers keep running locally (kubelet still operates)
- This causes the "split-brain" scenario for StatefulSets

When connectivity returns, the node re-establishes communication. Pods that were "evicted" in API state still run locally — kubelet reconciles and cleans them up.

**4. HA Considerations:**

Multiple API servers behind a load balancer:
- Lease updates go to whichever API server the LB picks
- All API servers see the update (via etcd watch)
- No coordination needed

**Monitoring Heartbeats:**

```bash
# View all Leases
kubectl get leases -n kube-node-lease

# Specific node
kubectl describe lease node-1 -n kube-node-lease

# Watch for updates
kubectl get leases -n kube-node-lease -w
```

Key metrics:
- `node_collector_evictions_total` — Pod evictions by Node Controller
- `node_collector_unhealthy_nodes_in_zone` — nodes considered unhealthy
- `apiserver_request_duration_seconds{resource="leases"}` — Lease API latency

---

### 65. What is NodeLease object?

NodeLease (typically just called "Lease") is the **modern, lightweight heartbeat mechanism** for Kubernetes nodes. Built on the Lease API which was generalized for various uses, NodeLease is a specific application: a Lease per node, used as a heartbeat.

**The Lease Resource:**

Leases are part of the `coordination.k8s.io/v1` API group:

```yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: node-1
  namespace: kube-node-lease
  ownerReferences:
  - apiVersion: v1
    kind: Node
    name: node-1
    uid: <node-uid>
spec:
  holderIdentity: node-1
  leaseDurationSeconds: 40
  renewTime: "2024-01-15T10:00:00.000000Z"
```

**Fields:**

- **holderIdentity** — who holds the lease (the node name)
- **leaseDurationSeconds** — how long the lease is valid (40 seconds default)
- **renewTime** — when the lease was last renewed (this is the "heartbeat" timestamp)
- **acquireTime** — when first acquired (less relevant for node leases)
- **leaseTransitions** — how many times the lease has changed holders (rare for node leases)

**The kube-node-lease Namespace:**

NodeLeases live in a dedicated namespace: `kube-node-lease`. This namespace:
- Is created automatically by Kubernetes
- Contains only Lease objects (one per node)
- Has special access controls
- Should not contain user resources

**Why a Separate Namespace?**

- **Performance** — watches on this namespace only fire on heartbeat updates
- **Permissions** — node authorizer specifically grants kubelet access to its own lease
- **Cleanup** — when a node is deleted, its Lease can be GC'd

**Owner References:**

Each Lease has an ownerReference to its corresponding Node:

```yaml
ownerReferences:
- apiVersion: v1
  kind: Node
  name: node-1
  uid: <node-uid>
```

This means when a Node is deleted, its Lease is **automatically garbage collected**. No orphaned leases.

**Lease Renewal Flow:**

Every `nodeLeaseDurationSeconds / 4` (default 10s), the kubelet:

1. Computes the new renewTime (current time)
2. Sends a PATCH request to the API server updating just `spec.renewTime`
3. API server validates (NodeAuthorizer ensures only this node's kubelet can update its own lease)
4. Persists to etcd
5. Returns success

The PATCH is minimal — just a few bytes for the timestamp. This is why it's so much cheaper than full Node status updates.

**Failure Detection:**

The Node Controller (in kube-controller-manager) watches Leases. Its detection logic:

```
every node_monitor_period (5s):
    for each Node:
        lease = getLease(node.name)
        last_observed = lease.renewTime
        if (now - last_observed) > node_monitor_grace_period:
            this node is unhealthy:
                set node.Conditions[Ready] = Unknown
                add taint: node.kubernetes.io/unreachable:NoExecute
                event: "Node lease not renewed"
```

**Comparing Lease vs Old-Style Heartbeats:**

| Aspect | Old (Status Updates) | New (NodeLease) |
|--------|----------------------|-----------------|
| Frequency | Every 10s | Every 10s |
| Size | Full Node object (~10KB) | Tiny PATCH (~100B) |
| Watches affected | Anyone watching Nodes | Only Node Controller |
| etcd writes | Heavy | Light |
| API server load | High | Low |
| Scalability | Limits at ~5000 nodes | Scales to 15000+ |

**The Lease API (Generalized):**

The Lease resource isn't only used for nodes. It's a general-purpose primitive for distributed coordination:

**Other Uses of Leases:**

**Leader Election:**

The kube-controller-manager, kube-scheduler, and many operators use Leases for leader election:

```yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: kube-controller-manager
  namespace: kube-system
spec:
  holderIdentity: kube-controller-manager-abc
  leaseDurationSeconds: 15
  renewTime: "2024-01-15T10:00:00Z"
```

Multiple replicas race to acquire the Lease. Only one succeeds, becomes the leader. It renews periodically. If the leader dies, the Lease expires and another replica acquires it.

**Custom Operators:**

Operators built with controller-runtime use Leases for leader election automatically. No manual code needed.

**Distributed Locks:**

Applications running in Kubernetes can use Leases as distributed locks — the holderIdentity field tracks who has the lock.

**Why Lease for Coordination:**

Leases are:
- Lightweight (minimal data)
- Atomic (updates via optimistic locking with resourceVersion)
- Watchable (real-time notifications)
- TTL-based (auto-expire if not renewed)

These properties make them ideal primitives for coordination patterns.

**Authorization for NodeLease:**

The NodeAuthorizer (a special authorizer for kubelet requests) has specific rules:
- A node's kubelet can read/write its own Lease in `kube-node-lease`
- A node's kubelet **cannot** access other nodes' Leases
- Combined with NodeRestriction admission plugin

This prevents a compromised kubelet from impersonating other nodes by manipulating their Leases.

**Common Operations:**

```bash
# List all node leases
kubectl get leases -n kube-node-lease

# Check a specific node's lease
kubectl describe lease node-1 -n kube-node-lease

# Watch for lease updates (you'll see them every 10s)
kubectl get leases -n kube-node-lease -w
```

**Debugging Lease Issues:**

If a node is reported NotReady but you think it should be:

```bash
# Check the lease
kubectl get lease <node-name> -n kube-node-lease -o yaml

# Look at renewTime — is it recent?
# If old, kubelet isn't updating the lease

# Check kubelet logs on the node
journalctl -u kubelet -f
```

Common issues:
- **Network problem** between node and API server
- **Kubelet bug** or hang
- **API server rejecting updates** (RBAC issue, certificate problem)
- **Clock skew** — if node's clock is wrong, renewTime might appear stale

**Tuning Lease Behavior:**

For faster failure detection in critical environments:

kubelet config:
```yaml
nodeLeaseDurationSeconds: 20      # half default
```

Controller manager:
```bash
--node-monitor-grace-period=20s   # match
```

This detects node failures in ~20 seconds. Trade-off: more sensitivity to transient issues (network blips can cause false-positive evictions).

**Lease Eviction Behavior:**

When a node's Lease expires:
1. Node Controller marks node Ready=Unknown
2. Adds `node.kubernetes.io/unreachable:NoExecute` taint
3. Pods with default tolerations get 300 seconds (5 min) before eviction
4. After 5 minutes, Pods are deleted from the API (not actually terminated locally)
5. Controllers create replacement Pods elsewhere

This is the classic "node failure" response — Lease is what drives it.

---

### 66. Explain API priority and fairness.

API Priority and Fairness (APF) is a Kubernetes feature that **protects the API server from overload** by managing concurrent requests in a fair, prioritized manner. It replaced the older max-in-flight request limits with a more sophisticated system.

**The Problem:**

The API server has limited capacity. Without management:
- A misbehaving client could flood it with requests
- A buggy controller could starve out other workloads
- Priority requests (control plane components) could be blocked by lower-priority traffic
- Total chaos under load

The old approach used simple counters: max concurrent reads and max concurrent writes (`--max-requests-inflight=400`, `--max-mutating-requests-inflight=200`). Above these, all requests were rejected with 429. Crude and unfair.

**The APF Solution:**

APF introduces:
- **FlowSchemas** — define which requests get what priority
- **PriorityLevelConfigurations** — define how much of the API server each priority level gets
- **Queues** — requests in excess are queued before being rejected
- **Fair queuing** — within a priority level, ensures fair distribution among different users/sources

**Core Concepts:**

**FlowSchema:**
A FlowSchema is a **routing rule**. It matches requests (by user, resource, verb, etc.) and assigns them to a priority level.

```yaml
apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: FlowSchema
metadata:
  name: kube-system-service-accounts
spec:
  priorityLevelConfiguration:
    name: workload-high
  matchingPrecedence: 1000
  distinguisherMethod:
    type: ByUser
  rules:
  - subjects:
    - kind: ServiceAccount
      serviceAccount:
        name: kube-controller-manager
        namespace: kube-system
    resourceRules:
    - verbs: ["*"]
      apiGroups: ["*"]
      resources: ["*"]
```

This says: requests from kube-controller-manager's ServiceAccount get routed to the `workload-high` priority level.

**PriorityLevelConfiguration:**
Defines **how much capacity** is allocated to this priority level, and how requests are queued.

```yaml
apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: PriorityLevelConfiguration
metadata:
  name: workload-high
spec:
  type: Limited
  limited:
    nominalConcurrencyShares: 200
    limitResponse:
      type: Queue
      queuing:
        queues: 128
        handSize: 6
        queueLengthLimit: 50
```

- **nominalConcurrencyShares: 200** — relative allocation. The API server's total concurrent capacity is divided proportionally based on shares across all priority levels.
- **Queue** — requests above the limit are queued, not rejected
- **128 queues, handSize 6** — fair queuing parameters
- **queueLengthLimit: 50** — max queue length before rejecting

**Built-in Priority Levels:**

Kubernetes ships with several:

- **system** — for the API server's own internal requests
- **node-high** — for kubelet status updates
- **leader-election** — for leader election traffic
- **workload-high** — for high-priority workloads (controller-manager, scheduler)
- **workload-low** — for general workloads
- **global-default** — catch-all
- **exempt** — bypasses all APF limits
- **catch-all** — fallback for unmatched requests

**Concurrency Shares:**

The API server has a total concurrency budget (calculated from `--max-requests-inflight` + `--max-mutating-requests-inflight`). This is divided among priority levels proportionally to their nominalConcurrencyShares.

If two levels have shares 100 and 200, the second gets 2x the capacity.

**Queueing:**

When a priority level is at capacity, additional requests are queued. The queue uses **fair queuing** to prevent one user from monopolizing the queue:

- Each "flow" (distinguished by user, namespace, or other) gets its own queue
- Round-robin between queues
- "handSize" controls how many queues participate in each round

This ensures that even if 10,000 requests come in from User A, requests from User B still get processed promptly.

**Request Flow:**

When a request arrives at the API server:

1. **Match FlowSchema** — find the first matching FlowSchema (by precedence)
2. **Get Priority Level** — look up the FlowSchema's priorityLevelConfiguration
3. **Determine Flow** — compute a flow identifier (e.g., the user, the namespace)
4. **Check Concurrency** — is the priority level below its limit?
   - **Yes**: process the request
   - **No**: queue or reject (based on configuration)
5. **Process** — actually run the request

If queued:
- Wait for a slot to free up
- Or timeout (~30s) and return 429

**Why It's Important:**

**1. Prevents Cascading Failures:**

Without APF, a misbehaving client could overload the API server, causing critical components (controller-manager, scheduler) to fail their requests. With APF, control plane components are protected via dedicated priority levels.

**2. Fair Multi-Tenancy:**

In a multi-tenant cluster, one team's heavy usage doesn't impact others. Each gets its share.

**3. Graceful Degradation:**

Under load, low-priority requests are queued or rejected first. Critical operations continue.

**4. Observability:**

APF provides detailed metrics about which flows are heavy, which are being queued, which are being rejected.

**Example: Customizing for Your Cluster:**

Suppose you have a critical controller that needs guaranteed API capacity:

```yaml
apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: PriorityLevelConfiguration
metadata:
  name: critical-controller
spec:
  type: Limited
  limited:
    nominalConcurrencyShares: 50
    limitResponse:
      type: Queue
      queuing:
        queues: 16
        handSize: 4
        queueLengthLimit: 25

---

apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: FlowSchema
metadata:
  name: critical-controller-flow
spec:
  matchingPrecedence: 500
  priorityLevelConfiguration:
    name: critical-controller
  rules:
  - subjects:
    - kind: ServiceAccount
      serviceAccount:
        name: critical-controller
        namespace: kube-system
    resourceRules:
    - verbs: ["*"]
      apiGroups: ["*"]
      resources: ["*"]
```

Now this controller gets dedicated capacity, isolated from other workloads.

**Monitoring APF:**

Key metrics:

- `apiserver_flowcontrol_priority_level_seat_count_samples` — concurrency usage per priority level
- `apiserver_flowcontrol_current_inqueue_requests` — queued requests
- `apiserver_flowcontrol_rejected_requests_total` — rejected requests
- `apiserver_flowcontrol_dispatched_requests_total` — completed requests
- `apiserver_flowcontrol_request_wait_duration_seconds` — queue wait time

If you see high `rejected_requests_total` or `current_inqueue_requests`, you have a load problem.

**Identifying Problem Flows:**

```bash
# Find which flows are heavy
kubectl get --raw /metrics | grep apiserver_flowcontrol_dispatched_requests_total | sort -k2 -n -r | head
```

This shows which flows are using the most API server capacity.

**Disabling APF:**

If needed (rare):
```bash
--enable-priority-and-fairness=false
```

Returns to the old max-in-flight model. Generally don't do this — APF is strictly better.

**Common Issues:**

1. **Misconfigured FlowSchemas** — wrong precedence, wrong matching
2. **Insufficient capacity for a priority level** — many rejected requests
3. **Long queue waits** — capacity is too low; increase shares
4. **Custom controllers without their own priority level** — falls into global-default, may be starved

**Best Practices:**

1. **Don't disable APF** — it's protecting you
2. **Create dedicated priority levels** for critical custom controllers
3. **Monitor metrics** to identify bottlenecks
4. **Use lower-precedence FlowSchemas** for catch-all cases
5. **Don't go too aggressive on queueing** — queues use memory

---

### 67. How does API Server authentication work?

The Kubernetes API server has a **multi-step authentication pipeline** that supports many credential types. Understanding it is essential for cluster security and access management.

**The Authentication Pipeline:**

When a request arrives at the API server:

1. **TLS handshake** — establishes encrypted connection
2. **Identify authentication method** from request headers/data
3. **Try each enabled authenticator** in order
4. **First successful authenticator wins** — sets user, groups, extra info
5. **Pass to authorization** (next phase)

Multiple authenticators can be enabled simultaneously. The first one that successfully identifies the user is used.

**Authentication Methods:**

**1. X.509 Client Certificates:**

The most common method for users and components.

When a client connects, it presents an X.509 certificate signed by the cluster's CA. The API server validates:
- Certificate is signed by a trusted CA
- Certificate is not expired
- Certificate is not revoked

Then extracts identity from the cert:
- **Common Name (CN)** → username
- **Organization (O)** → groups

```
Subject: CN=alice, O=developers, O=team-a
```

Maps to:
- User: `alice`
- Groups: `developers`, `team-a`

This is how kubelet authenticates (cert CN: `system:node:node-1`, O: `system:nodes`).

Configuration:
```bash
--client-ca-file=/etc/kubernetes/pki/ca.crt
```

**2. Bearer Tokens (Static Token Files):**

A simple text file maps tokens to users:

```csv
abc123token,alice,1000,"system:authenticated,developers"
xyz789token,bob,1001,"system:authenticated,admins"
```

Clients send:
```
Authorization: Bearer abc123token
```

Used in development or simple setups. Not recommended for production (tokens don't expire, no rotation).

Configuration:
```bash
--token-auth-file=/etc/kubernetes/tokens.csv
```

**3. Bootstrap Tokens:**

Short-lived tokens for new nodes joining the cluster. Used during kubeadm join.

Stored as Secrets in `kube-system`:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: bootstrap-token-abcdef
  namespace: kube-system
type: bootstrap.kubernetes.io/token
data:
  token-id: YWJjZGVm
  token-secret: ...
  expiration: ...
  usage-bootstrap-authentication: "true"
```

The kubelet uses this briefly to authenticate and request a permanent X.509 certificate via TLS bootstrapping.

**4. Service Account Tokens:**

For Pods talking to the API server.

**Legacy tokens (deprecated):**
- Long-lived JWT tokens stored as Secrets
- Never expired
- Bound to a ServiceAccount

**Bound tokens (modern, default since 1.22):**
- Short-lived JWT tokens generated on-demand
- Time-bound (default 1 hour, max 24 hours)
- Audience-bound (specific intended recipient)
- Optionally bound to a Pod
- Auto-rotated by kubelet

Pod uses projected volume to mount the token:
```yaml
volumes:
- name: kube-api-access-xxxxx
  projected:
    sources:
    - serviceAccountToken:
        expirationSeconds: 3607
        path: token
```

The token is a JWT signed by the API server. The API server validates it on every request (verifying signature, expiration, audience).

**5. OpenID Connect (OIDC):**

Federated authentication via external identity providers (Google, Okta, Azure AD, Dex, Keycloak).

Configuration:
```bash
--oidc-issuer-url=https://accounts.google.com
--oidc-client-id=kubernetes
--oidc-username-claim=email
--oidc-groups-claim=groups
```

Flow:
1. User authenticates with the OIDC provider externally
2. Gets an ID token (JWT)
3. Sends it to the API server:
   ```
   Authorization: Bearer <id-token>
   ```
4. API server verifies the JWT signature against the OIDC provider's public keys
5. Extracts claims for username and groups

Used widely with kubectl plugins like `kubectl oidc-login` or kubeconfig exec plugins.

**6. Webhook Token Authentication:**

The API server delegates token verification to an external webhook.

```bash
--authentication-token-webhook-config-file=/etc/kubernetes/webhook-config.yaml
```

For any bearer token it doesn't recognize, the API server sends it to the webhook:

```json
{
  "apiVersion": "authentication.k8s.io/v1",
  "kind": "TokenReview",
  "spec": {
    "token": "<token-value>"
  }
}
```

Webhook responds:
```json
{
  "apiVersion": "authentication.k8s.io/v1",
  "kind": "TokenReview",
  "status": {
    "authenticated": true,
    "user": {
      "username": "alice",
      "groups": ["developers"]
    }
  }
}
```

This integrates with any custom or external identity system.

**7. Authenticating Proxy:**

Trust headers from a front proxy.

```bash
--requestheader-username-headers=X-Remote-User
--requestheader-group-headers=X-Remote-Group
--requestheader-client-ca-file=/etc/kubernetes/proxy-ca.crt
```

A proxy in front of the API server (e.g., Dex, oauth2-proxy) handles authentication, then forwards requests with identity headers. The API server trusts these headers if the proxy presents a valid client cert.

**8. Anonymous Authentication:**

Requests without any credentials are authenticated as `system:anonymous` in the group `system:unauthenticated`.

```bash
--anonymous-auth=true   # default
```

By default, `system:anonymous` has minimal permissions. Disable if you want to require auth for all requests:

```bash
--anonymous-auth=false
```

**The Authentication Order:**

The API server tries authenticators in this order (roughly):
1. Authenticating Proxy (if headers present)
2. Client certificate
3. Bearer token authenticators (file, OIDC, webhook, service account)
4. Bootstrap tokens
5. Anonymous (if enabled)

The first one to successfully authenticate wins. If none succeed and anonymous is disabled, request is rejected with 401.

**User Identity Structure:**

After authentication, the user is represented as:

```go
type UserInfo struct {
    Username string             // e.g., "alice"
    UID      string             // e.g., "12345"
    Groups   []string           // e.g., ["developers", "team-a"]
    Extra    map[string][]string // additional info
}
```

This is then passed to authorization (RBAC, etc.) for permission checks.

**Special Users and Groups:**

Some users/groups have special meaning:
- `system:anonymous` — unauthenticated requests
- `system:unauthenticated` — group for unauthenticated users
- `system:authenticated` — group automatically added to all authenticated users
- `system:masters` — built-in cluster-admin group (special)
- `system:nodes` — kubelets
- `system:serviceaccount:<ns>:<name>` — service accounts

**TokenRequest API:**

Modern Kubernetes provides the **TokenRequest API** for generating tokens dynamically:

```bash
kubectl create token my-sa --duration=3600s --audience=my-audience
```

This generates a short-lived, audience-bound token. Useful for:
- CI/CD systems needing temporary access
- External services authenticating to Kubernetes
- Avoiding long-lived stored credentials

**Common Authentication Patterns:**

**For users:**
- Use OIDC with corporate identity provider (Google, Okta, Azure AD)
- Avoid long-lived static tokens
- Use kubectl plugins for token refresh

**For Pods:**
- Use bound service account tokens (default since 1.22)
- For cloud APIs, use IRSA (AWS) or Workload Identity (GCP)

**For cluster components:**
- Use X.509 certificates with the cluster CA
- Use kubelet TLS bootstrapping for new nodes

**For external systems:**
- ServiceAccount tokens with TokenRequest for short-lived access
- Or webhook authentication integrating with your identity system

**Security Best Practices:**

1. **Never use static token files** in production
2. **Use OIDC** for human users
3. **Use bound tokens** for ServiceAccounts
4. **Rotate certificates regularly** (cert-manager helps)
5. **Audit authentication events** via audit logs
6. **Disable anonymous auth** if not needed
7. **Restrict who can access cluster admin credentials**
8. **Monitor for unusual auth patterns** (failed auths, new users, etc.)

---

### 68. Explain authorization modes in Kubernetes.

After authentication identifies the user, **authorization** determines what they're allowed to do. Kubernetes supports multiple authorization modes that can be chained together.

**Authorization Modes:**

The API server is configured with a list of authorization modes via `--authorization-mode`:

```bash
--authorization-mode=Node,RBAC,Webhook
```

Each mode is consulted in order. If any mode **allows** the request, it proceeds. If all modes are exhausted without explicit allow, the request is denied.

**Built-in Authorization Modes:**

**1. AlwaysAllow:**

Allows every request. Used only for development clusters or very trusted environments.

```bash
--authorization-mode=AlwaysAllow
```

**Never use in production.**

**2. AlwaysDeny:**

Denies every request. Useful for testing.

```bash
--authorization-mode=AlwaysDeny
```

**3. ABAC (Attribute-Based Access Control):**

Authorization based on attributes specified in a policy file.

Policy file format (JSON Lines):
```json
{"apiVersion":"abac.authorization.kubernetes.io/v1beta1","kind":"Policy","spec":{"user":"alice","namespace":"production","resource":"pods","verb":"get"}}
{"apiVersion":"abac.authorization.kubernetes.io/v1beta1","kind":"Policy","spec":{"group":"system:authenticated","resource":"*","apiGroup":"*","verb":"get"}}
```

Configuration:
```bash
--authorization-mode=ABAC
--authorization-policy-file=/etc/kubernetes/abac-policy.json
```

**Drawbacks:**
- Static policy file requires API server restart to change
- Hard to manage at scale
- Not dynamic

**Largely deprecated in favor of RBAC.**

**4. RBAC (Role-Based Access Control):**

The dominant authorization mode in modern Kubernetes. Permissions are defined as resources within Kubernetes itself.

Configuration:
```bash
--authorization-mode=RBAC
```

Uses four resource types:
- **Role** — namespace-scoped set of permissions
- **ClusterRole** — cluster-scoped set of permissions
- **RoleBinding** — grants a Role's permissions to subjects in a namespace
- **ClusterRoleBinding** — grants a ClusterRole's permissions cluster-wide

Example:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-pod-reader
  namespace: production
subjects:
- kind: User
  name: alice
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

This grants Alice read-only access to Pods in the production namespace.

We'll cover RBAC in detail in the next question.

**5. Node:**

Special authorizer that grants kubelets just the permissions they need.

Configuration:
```bash
--authorization-mode=Node,RBAC
```

The Node authorizer:
- Identifies kubelet requests by the user format (`system:node:<nodename>` in group `system:nodes`)
- Grants access only to resources relevant to that node:
  - Pods scheduled to the node
  - Secrets/ConfigMaps used by those Pods
  - PVCs used by those Pods
  - Nodes (only the node's own object)
  - Leases in `kube-node-lease` (only its own)

Combined with the **NodeRestriction** admission plugin, this ensures kubelets can only:
- Modify their own Node object
- Modify Pods scheduled to them
- Cannot impersonate other nodes
- Cannot read Secrets unrelated to their Pods

This is critical security — without it, a compromised kubelet could read all Secrets in the cluster.

**6. Webhook:**

Delegates authorization decisions to an external webhook.

Configuration:
```bash
--authorization-mode=RBAC,Webhook
--authorization-webhook-config-file=/etc/kubernetes/webhook-auth.yaml
```

For each request, the API server (after the previous authorizers don't deny) sends a SubjectAccessReview to the webhook:

```json
{
  "apiVersion": "authorization.k8s.io/v1",
  "kind": "SubjectAccessReview",
  "spec": {
    "resourceAttributes": {
      "namespace": "production",
      "verb": "get",
      "resource": "pods",
      "name": "my-pod"
    },
    "user": "alice",
    "groups": ["developers"]
  }
}
```

Webhook responds:
```json
{
  "status": {
    "allowed": true,
    "reason": "User is in allowed group"
  }
}
```

Used to integrate with external policy systems:
- **OPA (Open Policy Agent)** — policy as code
- **Custom authorization servers** — company-specific policies

**Mode Chaining:**

Modes are evaluated in order. The first allow grants access; the first deny stops evaluation only for that mode. The actual logic:

```
for each mode in authorization modes:
    result = mode.authorize(request)
    if result == ALLOWED:
        return ALLOWED
    if result == DENIED:
        continue   # next mode might allow
    # if no_opinion, continue
return DENIED  # default if none allowed
```

**Common Configurations:**

**Standard production setup:**
```bash
--authorization-mode=Node,RBAC
```

- Node for kubelet requests
- RBAC for everything else

**With external policy:**
```bash
--authorization-mode=Node,RBAC,Webhook
```

- Node and RBAC handle most cases
- Webhook for additional policy enforcement (OPA, custom)

**Development:**
```bash
--authorization-mode=AlwaysAllow
```

- Don't use in production!

**Authorization Attributes:**

Every authorization check considers:
- **User** — who is asking
- **Groups** — what groups they belong to
- **Verb** — what action (get, list, watch, create, update, patch, delete, etc.)
- **Resource** — what kind of object (pods, services, etc.)
- **API Group** — which API (core, apps, batch, etc.)
- **Namespace** — for namespaced resources
- **Name** — specific resource name (optional)
- **Subresource** — specific subresource like `status`, `scale`, `exec`

Authorization decisions can use any combination of these.

**Non-Resource URL Authorization:**

Some endpoints aren't resources (like `/healthz`, `/metrics`). These are authorized by URL path:

```yaml
rules:
- nonResourceURLs: ["/healthz", "/version"]
  verbs: ["get"]
```

**SubjectAccessReview API:**

You can query what a user can do via the SubjectAccessReview API:

```bash
kubectl auth can-i create pods --namespace=production
kubectl auth can-i create pods --namespace=production --as=alice
kubectl auth can-i --list --as=alice
```

This is invaluable for debugging authorization.

**SelfSubjectAccessReview:**

Users can check their own permissions:
```bash
kubectl auth can-i list pods
```

**SelfSubjectRulesReview:**

List all rules applying to the current user:
```bash
kubectl auth can-i --list
```

**Authorization Caching:**

The API server caches authorization decisions briefly to reduce overhead. Cache TTL is short — changes to RBAC take effect quickly.

**Performance Considerations:**

- RBAC is fast (in-memory)
- Node authorizer is fast
- Webhook authorization adds network latency to every request
- Excessive webhook usage can become a bottleneck

For high-throughput scenarios:
- Keep RBAC as primary
- Use webhook only for specific use cases
- Cache webhook decisions when possible

**Best Practices:**

1. **Use RBAC** as your primary authorization mode
2. **Add Node authorizer** for kubelet protection
3. **Avoid AlwaysAllow** in production
4. **Audit authorization decisions** via audit logs
5. **Use OPA or Kyverno** for complex policy needs (often as admission webhooks rather than authorization webhooks)
6. **Minimize cluster admin** access
7. **Use namespaces** to scope access

---

### 69. Difference between RBAC, ABAC, and Webhook auth.

These three are alternative authorization mechanisms in Kubernetes, each with different strengths and weaknesses.

**RBAC (Role-Based Access Control):**

**Concept:** Permissions defined by roles. Users are assigned roles, roles have permissions.

**Resources:**
- Role — namespace-scoped permissions
- ClusterRole — cluster-scoped permissions
- RoleBinding — grants Role to subjects in a namespace
- ClusterRoleBinding — grants ClusterRole cluster-wide

**Example:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: deployment-manager
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: alice-deployments
subjects:
- kind: User
  name: alice
roleRef:
  kind: ClusterRole
  name: deployment-manager
  apiGroup: rbac.authorization.k8s.io
```

**Strengths:**
- **Native Kubernetes resources** — manage via kubectl, GitOps friendly
- **Dynamic** — changes take effect immediately
- **Well-understood** — extensive documentation, tools, examples
- **Granular** — per-resource, per-verb, per-namespace controls
- **Aggregation** — ClusterRoles can be combined via labels
- **Default** — used by virtually all production clusters

**Weaknesses:**
- **Limited expressiveness** — can't express "if X then Y" rules
- **No attribute matching beyond user/groups/resource** — can't say "only on weekdays"
- **Permissions are explicit** — must list everything
- **No deny rules** — only positive permissions (deny via not granting)

**When to use:** Almost always. RBAC is the default and sufficient for ~95% of cases.

**ABAC (Attribute-Based Access Control):**

**Concept:** Authorization based on attributes specified in policy rules.

**Format:** JSON Lines policy file:

```json
{"apiVersion":"abac.authorization.kubernetes.io/v1beta1","kind":"Policy","spec":{"user":"alice","namespace":"production","resource":"pods","verb":"*"}}
{"apiVersion":"abac.authorization.kubernetes.io/v1beta1","kind":"Policy","spec":{"group":"system:authenticated","nonResourcePath":"/version","readonly":true}}
{"apiVersion":"abac.authorization.kubernetes.io/v1beta1","kind":"Policy","spec":{"user":"*","apiGroup":"*","namespace":"public","resource":"*","verb":"get"}}
```

**Strengths:**
- **More flexible than RBAC in some ways** — wildcards across multiple dimensions
- **Single file** — all policies in one place

**Weaknesses:**
- **Static** — requires API server restart to apply changes (major drawback)
- **No native resource** — can't use kubectl to manage
- **Limited tooling** — much less ecosystem support than RBAC
- **No real flexibility advantages** in modern Kubernetes
- **Hard to audit** at scale
- **Deprecated path** — community focuses on RBAC

**When to use:** Almost never anymore. ABAC was an early authorization mode that's been superseded by RBAC. Some very old clusters may still use it, but new deployments should use RBAC.

**Webhook Authorization:**

**Concept:** Delegate authorization decisions to an external service via HTTP webhook.

**Configuration:**
```bash
--authorization-mode=RBAC,Webhook
--authorization-webhook-config-file=/etc/kubernetes/webhook-auth.yaml
```

```yaml
# webhook-auth.yaml
clusters:
- name: my-authz-webhook
  cluster:
    certificate-authority: /etc/kubernetes/webhook-ca.crt
    server: https://authz-webhook.example.com/authorize
users:
- name: my-authz-webhook
contexts:
- context:
    cluster: my-authz-webhook
    user: my-authz-webhook
  name: my-authz-webhook
current-context: my-authz-webhook
```

**Flow:**

For each request, the API server (after previous authorizers don't decide) sends a SubjectAccessReview:

```json
{
  "apiVersion": "authorization.k8s.io/v1",
  "kind": "SubjectAccessReview",
  "spec": {
    "resourceAttributes": {
      "namespace": "production",
      "verb": "create",
      "resource": "deployments",
      "group": "apps"
    },
    "user": "alice",
    "groups": ["developers"]
  }
}
```

Webhook responds:
```json
{
  "status": {
    "allowed": true,
    "reason": "User is approved deployer for production"
  }
}
```

**Strengths:**
- **Unlimited flexibility** — any logic you can code
- **Integration with external systems** — LDAP, custom policy engines, billing systems
- **Dynamic** — webhook can use real-time data
- **Complex policies** — time-based, attribute-based, ML-based, whatever

**Weaknesses:**
- **Latency** — webhook call on every request adds latency
- **Reliability** — if webhook is down, depends on failure policy
- **Operational complexity** — must operate the webhook
- **Performance bottleneck** — high request rates can overwhelm webhook
- **Debugging hard** — black box from Kubernetes' perspective

**When to use:**
- You have a complex authorization system already (e.g., enterprise policy engine)
- You need attribute-based rules beyond what RBAC supports
- You need dynamic decisions based on real-time data
- You need to integrate with external compliance systems

**Comparison Table:**

| Aspect | RBAC | ABAC | Webhook |
|--------|------|------|---------|
| Management | Kubernetes resources | Static file | External service |
| Dynamic updates | Yes | No (requires restart) | Yes |
| Granularity | Resource/verb level | Multiple attributes | Unlimited |
| Custom logic | No | Limited | Yes |
| Performance | Fast (in-memory) | Fast | Slower (network call) |
| Complexity | Medium | Low | High |
| Audit | Standard tools | Limited | Custom |
| GitOps friendly | Yes | Partial | No (config) |
| External integration | No | No | Yes |
| Multi-cluster | Per cluster | Per cluster | Centralized possible |
| Default in modern K8s | Yes | No | No |

**Common Real-World Patterns:**

**Pattern 1: RBAC Only (Most Common)**

For 95% of cases:
```bash
--authorization-mode=Node,RBAC
```

All authorization via RBAC. Simple, well-tested, sufficient.

**Pattern 2: RBAC + Webhook (Advanced)**

For complex policies:
```bash
--authorization-mode=Node,RBAC,Webhook
```

RBAC handles the standard cases (~95% of requests). Webhook handles the exceptional cases requiring complex logic. Performance is generally OK because most requests are handled by RBAC.

**Pattern 3: OPA Gatekeeper as Admission Controller**

Many policy needs are addressed via admission webhooks rather than authorization webhooks:

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: must-have-owner
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Namespace"]
  parameters:
    labels: ["owner"]
```

This is technically admission, not authorization. The difference:
- **Authorization** — can the user perform this action?
- **Admission** — even if they can, should this specific request be allowed?

Both are policy-like, but RBAC + admission webhooks (OPA, Kyverno) is generally simpler than authorization webhooks.

**Cross-Cluster Considerations:**

For multi-cluster setups:
- **Per-cluster RBAC** — each cluster has its own roles. Hard to keep consistent.
- **GitOps for RBAC** — sync RBAC manifests via Argo CD or Flux. Consistent across clusters.
- **Centralized webhook** — single source of truth, but adds latency and SPOF.

**Migration Path:**

If you're currently using ABAC:
1. Map ABAC policies to RBAC ClusterRoles/Roles
2. Test in non-prod
3. Switch to RBAC
4. Remove ABAC config

ABAC will continue to work for now but is essentially legacy.

**The Modern Stack:**

Most production clusters use:
1. **Node + RBAC** for authorization
2. **OPA Gatekeeper or Kyverno** for admission policies
3. **Audit logging** for visibility
4. **OIDC for user authentication**
5. **Bound tokens for service accounts**

This combination handles virtually all real-world authorization needs without webhook authorization complexity.

---

### 70. Explain service account token projection.

Service account token projection is the **modern mechanism for providing time-bound, audience-bound tokens to Pods**. It replaced the old long-lived static tokens with a much more secure model.

**The Old Way (Pre-1.22 default, now deprecated):**

When a ServiceAccount was created, Kubernetes auto-generated a Secret containing a JWT token:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: default-token-xyz
  annotations:
    kubernetes.io/service-account.name: default
type: kubernetes.io/service-account-token
data:
  token: <very-long-base64-JWT>
  ca.crt: <base64-CA>
  namespace: ZGVmYXVsdA==
```

This token had **major security issues:**
- **Never expired** — valid until the Secret was deleted
- **Not audience-bound** — could be used to authenticate to any service that trusted the cluster's CA
- **Pod-independent** — the same token continued working even if the Pod that "owned" it was deleted
- **Stored in etcd** — readable by anyone with access
- **Persisted forever** — even after Pod deletion, the token remained valid

If leaked, it was catastrophic. There was no automatic rotation.

**The New Way: Projected Service Account Tokens (Default Since 1.22):**

Tokens are now **generated on demand** by the API server and projected into Pods via the TokenRequest API. They're **bound, expiring tokens** with specific properties.

**Properties of Projected Tokens:**

1. **Time-bound** — expires after a configurable duration (default 1 hour)
2. **Audience-bound** — includes an `aud` claim specifying who should accept it
3. **Pod-bound** — tied to a specific Pod's lifecycle (token invalidated when Pod deleted)
4. **Automatically rotated** — kubelet refreshes before expiration
5. **Not persisted** — generated on demand, never stored in Secrets

**How It Works:**

When a Pod is created with a ServiceAccount (default behavior):

1. **kubelet receives the Pod spec** with `automountServiceAccountToken: true` (default)
2. **API server's TokenRequest API** is called by kubelet to generate a token
3. **Token is mounted** into the Pod via a projected volume
4. **Before expiration**, kubelet calls TokenRequest again, replaces the file
5. **Application reads the file** to get the current token

The Pod spec implicitly includes a projected volume:

```yaml
spec:
  serviceAccountName: my-sa
  containers:
  - name: app
    volumeMounts:
    - name: kube-api-access-xxxxx
      readOnly: true
      mountPath: /var/run/secrets/kubernetes.io/serviceaccount
  volumes:
  - name: kube-api-access-xxxxx
    projected:
      defaultMode: 0644
      sources:
      - serviceAccountToken:
          expirationSeconds: 3607      # ~1 hour
          path: token
      - configMap:
          items:
          - key: ca.crt
            path: ca.crt
          name: kube-root-ca.crt
      - downwardAPI:
          items:
          - path: namespace
            fieldRef:
              fieldPath: metadata.namespace
```

This single projected volume contains:
- **token** — the bound service account token (JWT)
- **ca.crt** — the cluster CA certificate
- **namespace** — the Pod's namespace

All mounted at `/var/run/secrets/kubernetes.io/serviceaccount/`.

**Token Format (JWT Claims):**

The projected token is a JWT with claims like:

```json
{
  "aud": ["https://kubernetes.default.svc.cluster.local"],
  "exp": 1700000000,
  "iat": 1699996393,
  "iss": "https://kubernetes.default.svc.cluster.local",
  "kubernetes.io": {
    "namespace": "production",
    "pod": {
      "name": "my-pod",
      "uid": "abc-123"
    },
    "serviceaccount": {
      "name": "my-sa",
      "uid": "def-456"
    }
  },
  "nbf": 1699996393,
  "sub": "system:serviceaccount:production:my-sa"
}
```

Key claims:
- **aud** — audience; who should accept this token
- **exp** — expiration timestamp
- **kubernetes.io.pod** — the specific Pod this token is for
- **sub** — the ServiceAccount identifier

**Audience Binding:**

For accessing the Kubernetes API, the default audience is the cluster's API URL. For external services, you specify a different audience:

```yaml
volumes:
- name: vault-token
  projected:
    sources:
    - serviceAccountToken:
        path: token
        expirationSeconds: 3600
        audience: vault                 # <-- specific to Vault
```

Now this token has `aud: ["vault"]`. The Kubernetes API would **reject** this token (wrong audience). Only Vault would accept it.

This is critical for security — a leaked token can only be used for one specific purpose.

**Pod Binding:**

When a token is generated via TokenRequest API, it can be bound to a specific Pod:

```json
{
  "apiVersion": "authentication.k8s.io/v1",
  "kind": "TokenRequest",
  "spec": {
    "audiences": ["vault"],
    "expirationSeconds": 3600,
    "boundObjectRef": {
      "kind": "Pod",
      "apiVersion": "v1",
      "name": "my-pod",
      "uid": "abc-123"
    }
  }
}
```

The token is invalid once the Pod is deleted (UID mismatch). This prevents tokens from being used after their Pod's life ended.

**Automatic Rotation:**

The kubelet manages token rotation:
- When the token reaches 80% of its lifetime (or configurable threshold)
- Calls TokenRequest API for a new token
- Replaces the file in the projected volume
- Application reads the new value on next access

Applications need to **re-read the token file** periodically. Most Kubernetes client libraries do this automatically.

**Real-World Use Cases:**

**1. AWS IRSA (IAM Roles for Service Accounts):**

EKS uses projected tokens to enable Pods to assume AWS IAM roles:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/S3Reader
```

EKS's mutating webhook injects a projected token volume with audience `sts.amazonaws.com`. The Pod:

1. Reads the token from `/var/run/secrets/eks.amazonaws.com/serviceaccount/token`
2. Calls AWS STS AssumeRoleWithWebIdentity using the token
3. STS validates the token against EKS's OIDC issuer
4. Returns temporary AWS credentials
5. Pod uses these for S3 access

No long-lived AWS credentials anywhere.

**2. GCP Workload Identity:**

Similar pattern for GCP — projected tokens with audience for GCP exchange to GCP credentials.

**3. HashiCorp Vault Integration:**

Vault uses Kubernetes authentication:
1. Pod gets projected token with audience `vault`
2. Vault Agent (sidecar) uses the token to authenticate to Vault
3. Vault validates the token via Kubernetes TokenReview API
4. If valid, Vault returns secrets to the agent
5. Agent writes secrets to a shared volume for the application

**4. Cross-Service Authentication:**

Microservices can mutually authenticate:
- Service A has projected token with audience "service-b"
- Calls Service B with the token
- Service B validates via TokenReview API
- Confirmed authentic, allowed to proceed

**Configuration Options:**

```yaml
volumes:
- name: token
  projected:
    sources:
    - serviceAccountToken:
        path: token
        expirationSeconds: 3600     # 1 hour (default 1 hour, max 24 hours)
        audience: my-audience       # required for non-default audiences
```

**Enabling/Disabling Token Mounting:**

By default, tokens are mounted into all Pods. To disable:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-sa
automountServiceAccountToken: false
```

Or per Pod:
```yaml
spec:
  serviceAccountName: my-sa
  automountServiceAccountToken: false
```

This avoids mounting tokens into Pods that don't need them — good security hygiene.

**Validating Tokens:**

External services can validate tokens via the TokenReview API:

```bash
kubectl create -f - <<EOF
apiVersion: authentication.k8s.io/v1
kind: TokenReview
spec:
  token: <token>
  audiences: ["vault"]
EOF
```

Response:
```yaml
status:
  authenticated: true
  user:
    username: system:serviceaccount:default:my-sa
    uid: ...
    groups: [...]
    extra:
      authentication.kubernetes.io/pod-name: ["my-pod"]
      authentication.kubernetes.io/pod-uid: ["abc-123"]
  audiences: ["vault"]
```

**Security Improvements vs Legacy Tokens:**

| Aspect | Legacy Token | Projected Token |
|--------|--------------|-----------------|
| Expiration | Never | Configurable (default 1h) |
| Audience binding | None | Specific audience |
| Pod binding | None | Bound to Pod UID |
| Storage | etcd (long-term) | Memory only |
| Rotation | None | Automatic |
| Revocation | Delete Secret | Token expires |
| Leak impact | Catastrophic | Limited (short-lived, specific audience) |

**Migration:**

If you have legacy code that reads tokens from Secrets:
- Modern Kubernetes still creates `kubernetes.io/service-account-token` Secrets if explicitly requested
- But projected tokens are mounted automatically into Pods
- Update applications to read tokens from the projected volume path
- Watch for token file updates (token rotates automatically)

**Best Practices:**

1. **Disable token mounting for Pods that don't need API access:**
   ```yaml
   automountServiceAccountToken: false
   ```

2. **Use specific audiences** for non-Kubernetes-API tokens
3. **Don't extract tokens to log them** — they're sensitive
4. **Use IRSA / Workload Identity** for cloud APIs (don't store cloud creds)
5. **Use cert-manager** for app-level certificates rather than relying on cluster CA
6. **Monitor TokenReview API usage** to detect unauthorized token validation
7. **Audit ServiceAccount usage** — RBAC permissions on SA = potential blast radius

---

# Kubernetes Advanced Interview Questions (71-80)

## 71. How does encryption at rest work?

Encryption at rest in Kubernetes protects sensitive data stored in etcd, the cluster's key-value store. Without encryption, anyone with access to etcd's data files can read secrets, configmaps, and other resources in plaintext.

**How it works:**

The kube-apiserver intercepts write operations to etcd and encrypts the data before persisting it. When reading, it decrypts on the fly. This is configured via the `--encryption-provider-config` flag pointing to an `EncryptionConfiguration` file.

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
      - configmaps
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-32-byte-key>
      - identity: {}
```

**Provider hierarchy matters:**

The order of providers is critical. The first provider listed is used for *encryption*, while *decryption* tries each provider in order. So you can rotate keys by adding a new key first, re-encrypting all secrets (`kubectl get secrets -A -o json | kubectl replace -f -`), then removing the old key.

**Provider types:**
- `identity`: No encryption (plaintext, the default)
- `aesgcm`: AES-GCM with random nonce (faster but requires key rotation every ~200k writes)
- `aescbc`: AES-CBC with PKCS#7 padding (deprecated in newer versions)
- `secretbox`: XSalsa20 and Poly1305
- `kms`: Envelope encryption using an external KMS like AWS KMS, Azure Key Vault, or HashiCorp Vault

**KMS provider (recommended for production):**

With KMS, Kubernetes uses envelope encryption. A Data Encryption Key (DEK) encrypts the actual data, and the DEK itself is encrypted by a Key Encryption Key (KEK) stored in the external KMS. This means the master key never leaves the KMS, and key rotation doesn't require re-encrypting all data, just the DEKs.

**Important caveat:** Encryption at rest only protects data in etcd. Data in memory (in the apiserver) and in transit (between components) requires separate TLS configuration. Also, existing secrets aren't automatically encrypted when you enable encryption; you must update each one.

---

## 72. Explain kube-apiserver request lifecycle

The apiserver is the gateway to the cluster. Every kubectl command, controller action, and kubelet update flows through it. Understanding the request lifecycle is critical for debugging permissions, admission, and performance issues.

**The full pipeline (in order):**

**1. Authentication:** The request arrives over HTTPS. The apiserver tries each configured authenticator in sequence: client certificates, bearer tokens (service account tokens, OIDC), webhook authenticators, or static token files. The first one that succeeds returns a user identity (username, UID, groups). If all fail, the request is rejected with 401.

**2. Authorization:** Given the authenticated user, can they perform this action? The apiserver consults authorization modules in order (typically Node, RBAC, Webhook). Any module returning "allow" permits the request; "deny" or "no opinion" passes it to the next. If none allow, the request gets 403.

**3. Mutating admission controllers:** These can modify the request. Built-in ones include `DefaultStorageClass`, `ServiceAccount` (injects SA token volumes), and `NamespaceLifecycle`. External `MutatingAdmissionWebhooks` are called here, used by tools like Istio sidecar injection.

**4. Schema validation:** The request body is validated against the OpenAPI schema for the resource. Required fields, types, and structure are checked.

**5. Validating admission controllers:** These can only accept or reject, not modify. Built-in examples include `ResourceQuota` and `PodSecurity`. `ValidatingAdmissionWebhooks` run here for custom policies (OPA Gatekeeper, Kyverno).

**6. etcd write:** If everything passes, the apiserver writes to etcd. For updates, it uses optimistic concurrency via `resourceVersion`, ensuring no concurrent write conflicts.

**7. Response:** The apiserver returns the persisted object (with `resourceVersion`, `uid`, `creationTimestamp` populated) to the client.

**For read requests:**

The flow is simpler: authentication → authorization → fetch from etcd (or watch cache, which keeps recent objects in memory to reduce etcd load) → return.

**Key insight on watch cache:** The apiserver maintains an in-memory cache of recent resource versions for each resource type. List and watch requests are typically served from this cache, not etcd directly. This is why a watch from `resourceVersion=0` is fast but may return slightly stale data.

---

## 73. How does watch API work internally?

The watch API is the foundation of Kubernetes's reactive architecture. Controllers, kubelets, and operators all use watches to react to state changes without polling.

**The protocol:**

A watch is an HTTP long-lived connection. The client sends `GET /api/v1/pods?watch=true&resourceVersion=12345`, and the server keeps the connection open, streaming JSON events as they occur:

```json
{"type":"ADDED","object":{...}}
{"type":"MODIFIED","object":{...}}
{"type":"DELETED","object":{...}}
{"type":"BOOKMARK","object":{"metadata":{"resourceVersion":"12999"}}}
```

**Event types:**
- `ADDED`: A new object matching the watch criteria
- `MODIFIED`: An object was updated
- `DELETED`: An object was removed
- `BOOKMARK`: A periodic checkpoint with the current `resourceVersion`, helpful for resuming watches after disconnection without missing events
- `ERROR`: Usually means the resourceVersion is too old (event history was pruned), and the client must relist

**Internal flow inside the apiserver:**

When you write to etcd, etcd itself emits a watch event back to the apiserver. The apiserver has a component called the `watch cache` (or `cacher`) that subscribes to etcd watches. The cacher maintains a ring buffer of recent events (typically the last ~1000 events per resource type).

When a client establishes a watch, the apiserver:
1. Finds the requested `resourceVersion` in the ring buffer
2. Replays events from that point
3. Then streams new events as they arrive from etcd

This means many clients watching the same resource don't each create separate etcd watches; they share the apiserver's single etcd watch. This is essential for scalability.

**The "too old resource version" problem:**

If the client's `resourceVersion` is older than the oldest event in the ring buffer (because too much time passed or too many changes occurred), the apiserver returns a 410 Gone error. The client must then perform a fresh LIST to get the current state and a new `resourceVersion`, then restart the watch. This LIST-then-WATCH pattern is fundamental.

**Bookmarks** were added to avoid unnecessary relists. They let the client advance its known `resourceVersion` even when no relevant objects change, so reconnecting after a network blip doesn't require a full relist.

---

## 74. Explain informer and shared informer

Building a controller by directly using the watch API is painful. You'd need to handle reconnections, maintain a local cache, deduplicate events, and so on. The **informer** abstracts all of this, and the **shared informer** makes it efficient across multiple consumers in the same process.

**What an informer does:**

An informer encapsulates the LIST-then-WATCH pattern. Internally, it:

1. **Lists** all objects of a type to get the initial state, storing them in a local in-memory store called the `Indexer` (a thread-safe cache with indexed lookups)
2. **Watches** for changes, updating the cache and triggering event handlers
3. **Resyncs periodically** (optional), re-emitting cached objects to trigger reconciliation even without changes, useful as a safety net
4. **Reconnects automatically** on failures, handling "too old resource version" by relisting

The local Indexer means your controller code reads from memory, not the apiserver, dramatically reducing apiserver load.

**Components of an informer:**

- **Reflector:** Performs the LIST/WATCH against the apiserver and pushes events into a DeltaFIFO queue
- **DeltaFIFO:** A FIFO queue that deduplicates and batches deltas for the same object
- **Indexer/Store:** The local cache, indexable by namespace, labels, or custom indexers
- **Controller loop:** Pops from DeltaFIFO, updates the Indexer, and invokes registered event handlers

**Shared informer:**

In a controller manager running many controllers, each watching pods (say), you'd waste resources running independent informers. The **SharedInformerFactory** ensures only one informer per resource type exists per process. Multiple event handlers can register against it, all reading from the same underlying cache and watch.

```go
factory := informers.NewSharedInformerFactory(clientset, 30*time.Second)
podInformer := factory.Core().V1().Pods()

podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc:    func(obj interface{}) { /* enqueue */ },
    UpdateFunc: func(old, new interface{}) { /* enqueue */ },
    DeleteFunc: func(obj interface{}) { /* enqueue */ },
})

factory.Start(stopCh)
factory.WaitForCacheSync(stopCh)
```

**Critical pattern:** Event handlers should *not* perform the actual reconciliation work. They should enqueue the object key (namespace/name) into a workqueue, and a separate set of worker goroutines pulls keys and reconciles. This decouples event speed from reconciliation speed and provides natural retry semantics via the workqueue's rate-limiting.

---

## 75. What is controller-runtime framework?

`controller-runtime` is a Go library maintained by the Kubernetes SIG API Machinery, built on top of client-go primitives (informers, workqueues). It's the foundation for **Kubebuilder** and **Operator SDK**, the two dominant frameworks for building Kubernetes operators and CRD-based controllers.

**Why it exists:**

Writing a controller directly with client-go requires significant boilerplate: setting up informers, workqueues, leader election, metrics, scheme registration, and webhook servers. `controller-runtime` provides idiomatic abstractions for all of these.

**Core abstractions:**

- **Manager:** The top-level orchestrator. It holds shared dependencies (cache, client, scheme, leader election lock) and starts all controllers, webhooks, and metrics endpoints.
- **Cache:** A shared informer-backed read cache. All controllers in the manager share it, so watching the same resource doesn't duplicate watches.
- **Client:** A high-level client that reads from the cache (fast, eventually consistent) and writes directly to the apiserver. The split-client pattern.
- **Controller:** A loop that watches resources and invokes a Reconciler for each event.
- **Reconciler:** Your business logic. It implements `Reconcile(ctx, req) (Result, error)`, where `req` contains the namespace/name of the object to reconcile.
- **Webhook server:** Built-in HTTP server for admission and conversion webhooks, with TLS handling.

**The reconciliation contract:**

The reconciler is called with just the *key* of an object, not the object itself. The reconciler must fetch the current state from the cache, compare with the desired state (often from a CR's spec), and take action to converge them. This is **level-triggered**, not edge-triggered: if the same object changes twice in quick succession, the reconciler may only see the latest state, and that's fine.

The return value controls what happens next:
- `Result{}, nil`: Done, don't requeue
- `Result{Requeue: true}, nil`: Requeue immediately (with rate limiting)
- `Result{RequeueAfter: 5*time.Minute}, nil`: Requeue after a delay
- `Result{}, err`: Requeue with exponential backoff

**Example skeleton:**

```go
func (r *MyReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    var obj myv1.MyResource
    if err := r.Get(ctx, req.NamespacedName, &obj); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    // ... reconcile logic ...
    return ctrl.Result{}, nil
}

func (r *MyReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&myv1.MyResource{}).
        Owns(&appsv1.Deployment{}).
        Complete(r)
}
```

The `Owns` declaration sets up automatic event-mapping: changes to owned Deployments trigger reconciliation of the parent MyResource, using the ownerReference chain.

---

## 76. Explain leader election in controllers

When you run multiple replicas of a controller for high availability, you almost never want all replicas actively reconciling. Two controllers trying to manage the same resource will race, double-create, and conflict. Leader election ensures only one replica is active at a time, with automatic failover.

**The mechanism:**

Leader election uses a Kubernetes resource (typically a `Lease` in `coordination.k8s.io/v1`, formerly ConfigMaps or Endpoints) as a distributed lock. The leader periodically updates the Lease's `renewTime` to prove liveness. If the leader dies or stalls, the Lease ages out, and standby replicas race to take ownership.

**Key parameters:**

- **LeaseDuration** (default 15s): How long a replica can be leader without renewing before its lease is considered invalid
- **RenewDeadline** (default 10s): How long the current leader retries renewing before giving up and stepping down
- **RetryPeriod** (default 2s): How frequently the leader renews and how often non-leaders attempt acquisition

The invariant `LeaseDuration > RenewDeadline > RetryPeriod` must hold. If RenewDeadline ≥ LeaseDuration, two leaders could simultaneously believe they hold the lock.

**How a replica becomes leader:**

1. On startup, the replica reads the Lease object
2. If unowned or expired, it attempts an atomic update setting itself as `holderIdentity` (using resourceVersion-based optimistic concurrency)
3. If the update succeeds, it's now the leader and starts the controller's main loop
4. It begins a renewal goroutine that updates `renewTime` every `RetryPeriod`
5. Non-leaders sit in a loop checking the Lease, ready to attempt acquisition when they see it expire

**Step-down behavior:**

The fact that the previous leader has stopped doing work *before* its lease expires is crucial. controller-runtime achieves this by canceling the manager's context as soon as renewal fails. Your reconcilers should respect context cancellation and exit promptly to avoid split-brain.

**A subtle pitfall:** Leader election only prevents two controllers in the same Kubernetes-aware system from acting simultaneously. It does not prevent an old leader from completing an in-flight external API call (writing to a cloud provider, for instance) after losing leadership. For operations on external systems, additional idempotency and ownership checks may be needed.

**In controller-runtime:**

```go
mgr, err := ctrl.NewManager(cfg, ctrl.Options{
    LeaderElection:          true,
    LeaderElectionID:        "mycontroller.example.com",
    LeaderElectionNamespace: "mycontroller-system",
})
```

---

## 77. How does kube-proxy program iptables?

`kube-proxy` in iptables mode is the default implementation of Kubernetes Services. It watches Services and EndpointSlices from the apiserver and translates them into iptables rules that redirect traffic from ClusterIPs to actual Pod IPs.

**The rule structure:**

kube-proxy uses several custom chains in iptables's `nat` table:

- `KUBE-SERVICES`: Entry point for all Service traffic, matched on destination IP and port
- `KUBE-SVC-XXXXXX`: Per-service chain that load-balances across endpoints using probabilistic matching
- `KUBE-SEP-XXXXXX`: Per-endpoint chain that performs the actual DNAT to a Pod IP
- `KUBE-MARK-MASQ`: Marks packets for source NAT (for traffic that needs SNAT to avoid asymmetric routing)
- `KUBE-POSTROUTING`: Applies SNAT to marked packets

**Flow for a packet to a ClusterIP:**

1. Packet enters PREROUTING (or OUTPUT for locally generated traffic)
2. Jumps to `KUBE-SERVICES`
3. Matches the rule for the destination ClusterIP and port, jumps to that service's `KUBE-SVC-XXX` chain
4. The SVC chain has multiple rules with `statistic --mode random --probability` matchers:
   - For 3 endpoints: first rule has 33% probability, second has 50% (of remaining), third has 100%
   - This produces uniform 1/3 distribution
5. The selected rule jumps to a `KUBE-SEP-XXX` chain
6. The SEP chain performs `DNAT --to-destination <pod-ip>:<pod-port>`
7. The packet is then routed normally to the destination Pod

**Why this approach is problematic at scale:**

iptables rules are evaluated linearly. With 5,000 Services and 10 endpoints each, you have roughly 50,000+ rules. Every connection traversal walks through this list. Both packet processing latency and rule update time degrade significantly. Updating a single Service may require regenerating and atomically swapping a large iptables ruleset.

**This is why IPVS mode exists:**

In IPVS mode, kube-proxy programs the Linux kernel's IP Virtual Server, which uses hash tables for O(1) lookup regardless of the number of Services. iPVS also provides real load-balancing algorithms (round-robin, least-connections, source-hash) instead of just random selection.

**eBPF alternatives:**

Projects like Cilium replace kube-proxy entirely with eBPF programs attached to socket and packet hooks. Service resolution happens in eBPF maps with native hash lookup, bypassing iptables/netfilter overhead entirely.

---

## 78. Explain conntrack issues in Kubernetes

Conntrack (connection tracking) is the Linux kernel subsystem that remembers the state of network connections to enable stateful firewalling and NAT. Since kube-proxy uses DNAT extensively, every connection through a Service creates a conntrack entry. This makes conntrack a frequent source of subtle networking problems in Kubernetes.

**The capacity problem:**

The conntrack table has a fixed maximum size (`nf_conntrack_max`, typically 262144 or 524288 on modern systems). When it fills up, new connections fail with `nf_conntrack: table full, dropping packet` errors in dmesg. Symptoms include random connection failures, intermittent timeouts, and DNS resolution failures.

In high-throughput environments (web servers handling many short-lived connections, especially with sidecar proxies that effectively double the connection count), exhaustion happens fast. Mitigation:
- Increase `net.netfilter.nf_conntrack_max`
- Reduce `net.netfilter.nf_conntrack_tcp_timeout_established` (default 5 days is wildly long)
- Reduce `nf_conntrack_tcp_timeout_time_wait`
- Monitor `nf_conntrack_count` vs `nf_conntrack_max` and alert at 80%

**The "race condition" problem:**

A well-known issue: DNS queries from glibc-based images often send A and AAAA queries in parallel from the same source port. Both queries get DNAT'd by kube-proxy to (potentially different) CoreDNS pods. Under specific timing, conntrack can create entries pointing to different destinations for what it considers "the same flow," and one of the responses gets dropped. This manifests as 5-second DNS delays (the resolver retry timeout).

Mitigations include using `single-request-reopen` or `single-request` options in resolv.conf, setting `dnsConfig` options at the pod level, or using NodeLocal DNSCache, which avoids the DNAT entirely by routing DNS through a local cache.

**The asymmetric routing problem:**

If a packet returns via a different path than it left, conntrack won't find a matching entry and may drop the packet as INVALID. This commonly happens with:
- Multi-interface nodes with misconfigured routing
- Source-IP-preserving load balancers
- Pods on different subnets routed asymmetrically

Setting `net.ipv4.conf.all.rp_filter=2` (loose mode) instead of strict mode (1) often helps.

**TIME_WAIT exhaustion:**

For high-rate outbound connections from a Pod, you can exhaust ephemeral ports because each closed connection sits in TIME_WAIT for 2*MSL (60 seconds typically). Each TIME_WAIT entry is a conntrack entry too. Workarounds: connection pooling at the application level, or `net.ipv4.tcp_tw_reuse=1`.

---

## 79. What causes DNS latency in Kubernetes?

DNS is one of the most common sources of latency and reliability issues in Kubernetes. The default setup routes every DNS query through CoreDNS pods via Service VIPs, introducing several failure modes.

**Cause 1: The conntrack race (5-second timeout)**

As described above, parallel A and AAAA queries from glibc resolvers can race in conntrack, causing one response to be dropped. The application waits the default 5-second timeout and retries. Users see this as occasional 5-second latency spikes that look completely random.

**Cause 2: ndots:5 search domain explosion**

Kubernetes injects this into Pod resolv.conf:

```
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

`ndots:5` means any name with fewer than 5 dots is first tried as a relative name against each search domain. So `api.example.com` (2 dots) is tried as:
1. `api.example.com.default.svc.cluster.local`
2. `api.example.com.svc.cluster.local`
3. `api.example.com.cluster.local`
4. `api.example.com` (finally)

That's 4 round-trips for every external lookup, doubled for A+AAAA. Fix: use FQDN with trailing dot (`api.example.com.`) or set `dnsConfig.options` with a lower `ndots` value.

**Cause 3: CoreDNS resource limits**

CoreDNS pods have CPU limits by default in many distributions. Under load, CPU throttling causes query queueing and timeouts. Common fix: remove CPU limits (leave requests), increase replica count, or use autoscaling.

**Cause 4: Single CoreDNS replica, cross-AZ hops**

Default CoreDNS deployments have 2 replicas. A pod in zone A might query a CoreDNS pod in zone C, adding cross-AZ latency to every lookup. Fix: NodeLocal DNSCache (a DaemonSet that runs a DNS cache on every node, listening on a link-local IP) so queries are answered locally without crossing the network.

**Cause 5: Upstream DNS issues**

CoreDNS forwards external queries to the node's resolv.conf nameservers. If those are slow or rate-limited, every external lookup is slow. CoreDNS caching helps, but TTLs from upstream control cache effectiveness. Configure CoreDNS's `cache` plugin with `success` and `denial` TTLs.

**Cause 6: Negative caching gaps**

NXDOMAIN responses are cached, but if your search-domain expansion produces many NXDOMAINs (per cause 2), and the cache isn't sized appropriately, you get repeated misses.

**The recommended stack:**

NodeLocal DNSCache + lowered `ndots` value + FQDN usage + adequate CoreDNS replicas + monitoring of CoreDNS metrics (`coredns_dns_request_duration_seconds`, cache hit rates).

---

## 80. Explain EndpointSlices

EndpointSlices were introduced in Kubernetes 1.16 (GA in 1.21) to replace the legacy `Endpoints` API. They solve a fundamental scalability problem: a single Endpoints object stored all endpoints for a Service, leading to severe issues at scale.

**The problem with Endpoints:**

A Service with 1,000 backing Pods had a single Endpoints object containing all 1,000 endpoints. Whenever any one Pod's IP changed (rolling update, scale event, pod restart), the *entire* Endpoints object was rewritten and sent to every kube-proxy in the cluster. With 1,000 nodes and 1,000 endpoints, that's 1 MB × 1,000 nodes = 1 GB of traffic for a single Pod IP change. At larger scales, this overwhelmed the apiserver.

Additionally, etcd has a 1.5 MB object size limit, capping Services at roughly 5,000 endpoints.

**How EndpointSlices solve it:**

A Service's endpoints are split across multiple EndpointSlice objects, each holding up to 100 endpoints (configurable). The Service's endpoints are derived by aggregating all EndpointSlices labeled with `kubernetes.io/service-name=<service-name>`.

When a Pod's IP changes, only the EndpointSlice containing it needs updating. So updating 1 of 1,000 endpoints means rewriting and distributing only 1 small slice (~100 endpoints), not the full set.

**Structure of an EndpointSlice:**

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-service-abc123
  labels:
    kubernetes.io/service-name: my-service
addressType: IPv4
ports:
  - name: http
    port: 8080
    protocol: TCP
endpoints:
  - addresses: ["10.244.1.5"]
    conditions:
      ready: true
      serving: true
      terminating: false
    nodeName: node-1
    zone: us-east-1a
    hints:
      forZones:
        - name: us-east-1a
```

**New capabilities EndpointSlices unlock:**

- **Dual-stack support:** `addressType` can be IPv4, IPv6, or FQDN. Separate slices per address family avoid mixing.
- **Topology-aware routing:** The `hints.forZones` field lets kube-proxy prefer endpoints in the same zone, reducing cross-AZ traffic and costs. Enabled via the `service.kubernetes.io/topology-mode` annotation.
- **Distinct ready/serving/terminating states:** Traditional Endpoints had only "ready." EndpointSlices separate these so traffic can continue going to terminating Pods during graceful shutdown (`serving: true, terminating: true`), preventing connection drops during rolling updates.
- **Better scalability ceiling:** Services can now have tens of thousands of endpoints without hitting etcd or watch-distribution limits.

**Backward compatibility:**

The legacy `Endpoints` API still exists. The endpoints controller maintains both Endpoints and EndpointSlice objects, and kube-proxy in modern versions reads EndpointSlices. Older clients can still use the Endpoints API, but they miss new features like topology hints.

**EndpointSliceMirroring:**

If you manually create an Endpoints object (for headless services pointing to external resources, for instance), the EndpointSliceMirroring controller automatically generates corresponding EndpointSlices so kube-proxy continues to work uniformly.

# Kubernetes Advanced Interview Questions (81-100)

## 81. What is the role of cAdvisor?

**cAdvisor** (Container Advisor) is an open-source agent originally developed by Google that collects, aggregates, and exports resource usage and performance metrics for running containers. In Kubernetes, it's not a separate component you deploy; it's **embedded directly inside the kubelet** on every node.

**What it collects:**

For each container running on the node, cAdvisor gathers:
- CPU usage (per-core, throttling stats, user/system time)
- Memory usage (RSS, cache, working set, page faults)
- Network statistics (bytes/packets sent/received per interface)
- Filesystem usage (reads, writes, IOPS)
- Container start/stop events

**How it works:**

cAdvisor reads from the Linux kernel's cgroup filesystem (`/sys/fs/cgroup`) to extract resource usage. It also queries the container runtime to learn about container identities and metadata. It does not require any per-container instrumentation; everything comes from kernel accounting that cgroups already maintains.

**How metrics flow:**

```
Container → cgroups → cAdvisor (in kubelet) → kubelet metrics endpoint
                                            ↓
                                      /metrics/cadvisor (Prometheus format)
                                      /stats/summary (Summary API for metrics-server)
```

- **metrics-server** scrapes the kubelet's Summary API and exposes the `metrics.k8s.io` API used by `kubectl top` and the Horizontal Pod Autoscaler
- **Prometheus** scrapes `/metrics/cadvisor` directly for detailed historical metrics

**Why this design matters:**

Because cAdvisor lives inside kubelet, every Kubernetes node automatically exposes container metrics without additional deployment. The HPA, VPA, and any monitoring stack can rely on this being present.

**Important note:** cAdvisor does not store metrics historically. It exposes a point-in-time view. Long-term storage requires scraping its endpoint into Prometheus, Datadog, or similar.

---

## 82. Explain kube-state-metrics architecture

**kube-state-metrics (KSM)** is a service that listens to the Kubernetes API and generates metrics about the *state of Kubernetes objects*, not about resource consumption. It's the counterpart to cAdvisor: cAdvisor tells you "how much CPU is this container using," while KSM tells you "how many replicas does this Deployment have, and is it available."

**The architectural distinction:**

- **cAdvisor / metrics-server:** Resource utilization (CPU, memory, network bytes)
- **kube-state-metrics:** Object state (Deployment replicas, Pod phase, Node conditions, PVC status)

**How it works internally:**

1. KSM uses **shared informers** (the same client-go pattern controllers use) to watch all relevant Kubernetes resources: Pods, Deployments, ReplicaSets, StatefulSets, Nodes, PersistentVolumes, ConfigMaps, etc.
2. The informers maintain an in-memory cache of every object.
3. On each Prometheus scrape, KSM walks its cache and generates fresh metrics from the cached state.
4. It exposes these on `/metrics` in Prometheus format.

**Key metrics it produces:**

```
kube_deployment_status_replicas{deployment="nginx", namespace="default"} 3
kube_deployment_status_replicas_available{deployment="nginx"} 2
kube_pod_status_phase{pod="x", phase="Running"} 1
kube_node_status_condition{node="n1", condition="Ready", status="true"} 1
kube_pod_container_status_restarts_total{...} 5
```

**Why it's stateless:**

KSM doesn't store anything itself. The Kubernetes API is the source of truth; KSM is just a metrics adapter. If KSM dies and restarts, it rebuilds its cache via LIST+WATCH and continues. This means it can run as a single replica, but multiple replicas are supported via **sharding** for very large clusters.

**Sharding:**

For clusters with hundreds of thousands of objects, a single KSM instance becomes memory-heavy (gigabytes of cache). KSM supports horizontal sharding via the `--shard` and `--total-shards` flags, where each instance handles a subset of objects based on a hash. The DaemonSet deployment mode runs one shard per node.

**Performance consideration:**

KSM is essentially a snapshot generator. The metric values it emits reflect the cache state at scrape time, not a streaming view. For events that change rapidly between scrapes (like Pod restarts), you'll see only sampled state, not every transition.

---

## 83. What is node pressure eviction?

**Node pressure eviction** is the kubelet's mechanism for protecting node stability by proactively evicting Pods when the node experiences resource pressure. Unlike API-initiated eviction (which goes through the eviction API), node-pressure eviction is a local kubelet decision made in real time when monitored resources cross configured thresholds.

**What resources are monitored:**

The kubelet monitors:
- `memory.available`: Available memory on the node
- `nodefs.available`: Available space on the kubelet's root filesystem
- `nodefs.inodesFree`: Available inodes on the root filesystem
- `imagefs.available`: Available space on the container image filesystem (if separate)
- `imagefs.inodesFree`: Available inodes on the image filesystem
- `pid.available`: Available PIDs on the node

**Soft vs. hard eviction thresholds:**

- **Hard eviction:** Pods are killed immediately when crossed. No grace period.
  - Example: `--eviction-hard=memory.available<100Mi`
- **Soft eviction:** A grace period applies. If pressure persists beyond the grace period, eviction proceeds.
  - Example: `--eviction-soft=memory.available<300Mi --eviction-soft-grace-period=memory.available=30s`

**The eviction decision process:**

When a threshold is crossed:

1. The kubelet sets a node condition (`MemoryPressure`, `DiskPressure`, `PIDPressure`)
2. The scheduler stops placing new Pods on the node (most conditions taint the node)
3. The kubelet ranks all running Pods for eviction
4. It evicts Pods one at a time, waiting briefly between evictions to see if pressure subsides

**Pod ranking for eviction:**

Pods are sorted by:

1. **Whether they exceed requests:** Pods using more than they requested are evicted first.
2. **Pod Priority:** Lower priority Pods are evicted first.
3. **Usage relative to requests:** Among pods with the same priority, the one using the most beyond its request is evicted first.

**QoS classes and eviction order:**

- `BestEffort` Pods (no requests/limits) are typically evicted first
- `Burstable` Pods (some requests, no limits or limits ≠ requests) are next
- `Guaranteed` Pods (requests == limits for all resources) are evicted last, and only if they're somehow exceeding their guarantee

**Critical pods:**

System-critical pods (those with `priorityClassName: system-cluster-critical` or `system-node-critical`) are exempt unless absolutely necessary. The kubelet itself runs outside this mechanism, but DaemonSet pods are not automatically exempt; they need priority classes.

**Why this exists:**

Without node pressure eviction, the Linux OOM killer would handle memory exhaustion, but OOM kills are abrupt and unpredictable. The kubelet's eviction is graceful: it sends SIGTERM, allows termination grace period, and ensures the cluster control plane is informed so the Pod can be rescheduled elsewhere.

---

## 84. Explain memory pressure handling

Memory pressure is the most common and most dangerous resource pressure in Kubernetes. Misconfigurations here cause production outages.

**The layered defense:**

Kubernetes handles memory pressure at multiple levels:

**Layer 1: Container memory limits and cgroup OOM**

Each container with a memory limit runs inside a cgroup with that limit set. If the process inside exceeds it, the kernel's OOM killer terminates a process *within that cgroup*. This typically kills the main container process, resulting in OOMKilled status. This is fast and localized.

**Layer 2: Node-level memory pressure detection**

Kubelet polls memory stats every 10 seconds (configurable via `--housekeeping-interval`). When `memory.available` crosses the eviction threshold, kubelet:
1. Sets `MemoryPressure=true` on the node
2. Taints the node with `node.kubernetes.io/memory-pressure:NoSchedule`
3. Begins evicting Pods ranked by QoS and usage

**Layer 3: Kernel OOM killer (last resort)**

If memory pressure escalates faster than kubelet can react (e.g., a sudden allocation spike), the kernel OOM killer fires. It selects victims using `oom_score`, which Kubernetes manipulates so BestEffort containers have higher scores (more likely to be killed) than Guaranteed ones.

**OOM score adjustments by QoS:**

- **Guaranteed:** `oom_score_adj = -997` (very unlikely to be killed)
- **Burstable:** `oom_score_adj = 1000 - (1000 * memoryRequest / memoryCapacity)` (scaled)
- **BestEffort:** `oom_score_adj = 1000` (most likely to be killed)

**Memory working set vs. RSS:**

Kubelet evicts based on **working set**, not RSS. Working set ≈ `total_memory - inactive_file_cache`. This excludes reclaimable page cache, which is important: a container with high file cache usage isn't actually under memory pressure because that cache can be evicted by the kernel.

**Allocatable memory:**

Not all node memory is available to Pods. The kubelet reserves memory for:
- `--system-reserved`: For system daemons (sshd, systemd)
- `--kube-reserved`: For kubelet, container runtime, kube-proxy
- `--eviction-hard`: Buffer for eviction threshold

**Allocatable = Capacity - SystemReserved - KubeReserved - HardEvictionThreshold**

This is what Pods can actually request. Misconfiguring reservations leads to nodes that look "free" to the scheduler but are actually under memory pressure.

**Common production issue:**

A pod with `requests.memory=100Mi, limits.memory=2Gi` (Burstable) suddenly grows from 100Mi to 1.5Gi. Other Pods on the same node, also Burstable, are now competing for limited memory. The kubelet sees memory pressure, starts evicting Pods (perhaps the wrong ones), and a cascade follows. Solution: avoid wildly disparate requests and limits; treat the limit as the actual ceiling.

---

## 85. What is OOMKilled and how do you debug it?

**OOMKilled** means a container was terminated by the Linux Out-Of-Memory killer because it (or its cgroup) exceeded its memory limit, or the node ran out of memory entirely. The container's exit code is 137 (128 + SIGKILL signal 9).

**Identifying OOMKilled:**

```bash
kubectl describe pod <pod-name>
# Look for:
# Last State:     Terminated
#   Reason:       OOMKilled
#   Exit Code:    137
```

Also check:

```bash
kubectl get events --sort-by='.lastTimestamp'
# Look for: "Memory cgroup out of memory"
```

**Two distinct scenarios:**

**Scenario A: Container exceeded its own memory limit**

The container hit `resources.limits.memory`. The kernel killed it via the cgroup OOM killer. Only this container died; the node is healthy. This is the more common case and usually means:
- Memory limit set too low for actual workload
- Memory leak in application
- Sudden traffic spike or large data processing operation

**Scenario B: Node-level OOM**

The node ran out of memory and the global OOM killer selected this container. The node's other Pods may also be affected. Check:

```bash
# On the node:
dmesg -T | grep -i "out of memory"
journalctl -k | grep -i "out of memory"
```

You'll see lines like `oom-kill:constraint=CONSTRAINT_MEMCG,...,task=java,pid=12345`. The constraint tells you whether it was a cgroup (memcg) or system-wide OOM.

**Debugging workflow:**

**Step 1: Check the limit vs. actual usage history**

```bash
kubectl top pod <pod-name> --containers
```

Then check Prometheus/Grafana for the container's `container_memory_working_set_bytes` over time, compared to the limit. Look for the pattern leading up to the kill.

**Step 2: Inspect application memory profile**

Get a heap dump or memory profile if the language supports it (Java's jmap, Go's pprof, Python's tracemalloc). Look for memory leaks, unbounded caches, or large object allocations.

**Step 3: Check for off-heap memory**

For JVM applications, the heap size (`-Xmx`) doesn't include direct buffers, metaspace, thread stacks, or native libraries. A common pitfall: `-Xmx2g` with `limits.memory=2Gi` will OOMKill because total JVM memory exceeds 2Gi. Rule of thumb: set heap to 60-70% of the container limit.

**Step 4: Examine init containers and sidecars**

The Pod's total memory is the sum of all containers. A sidecar with a memory leak can OOMKill the entire Pod's most memory-consuming container.

**Step 5: Check kernel messages**

```bash
# On the affected node
dmesg | grep -A 20 "killed process"
```

This shows exactly which process was killed and the memory state at that moment.

**Common fixes:**

- Increase memory limit (if usage is legitimate)
- Set memory request closer to typical usage (helps scheduling and avoids overcommit)
- Fix application memory leaks
- For JVM: tune heap size relative to container limit, use container-aware flags (`-XX:MaxRAMPercentage=70`)
- Add a Vertical Pod Autoscaler in recommendation mode to suggest right-sizing

---

## 86. Explain CPU throttling in containers

CPU throttling is when the Linux kernel deliberately stops a container's processes from running because they've exceeded their CFS (Completely Fair Scheduler) quota. Unlike memory limits, which cause kills, CPU limits cause *delays*. This makes throttling much harder to diagnose because applications keep running but slowly.

**How CFS quota works:**

The kernel divides time into **periods** (default 100ms) and assigns each cgroup a **quota** per period. A container with `limits.cpu=500m` gets 50ms of CPU time per 100ms period (across all cores combined). Once the quota is exhausted, the cgroup is throttled until the next period begins.

**Critical insight:** The quota is summed across all CPUs. A container with 0.5 CPU limit on a 4-core machine can use all 4 cores for 12.5ms, then be throttled for 87.5ms. From the application's perspective, it gets blocked for almost the entire period despite the machine being idle.

**Detecting throttling:**

```bash
# Inside the container (cgroup v1):
cat /sys/fs/cgroup/cpu/cpu.stat
# nr_periods 1000
# nr_throttled 234        # Number of periods where throttling occurred
# throttled_time 12345678  # Total time throttled (ns)
```

Prometheus metric: `container_cpu_cfs_throttled_periods_total / container_cpu_cfs_periods_total` gives the throttling ratio.

**Why this is more problematic than it seems:**

Multi-threaded applications often hit throttling well before reaching their average CPU limit. A Java application with 10 threads briefly using full CPU each for 5ms (total: 50ms across threads) exhausts a 500m CPU quota in just 10ms of wall-clock time, then gets throttled for 90ms. P99 latencies spike dramatically while average CPU usage looks fine.

**The "no limits" debate:**

A controversial but widespread practice: **don't set CPU limits at all** (only requests). Rationale:
- Requests ensure scheduling fairness
- Without limits, applications can use spare CPU on the node
- Throttling causes latency that's worse than the alternative (occasional contention)

The counter-argument: without limits, a misbehaving Pod can starve neighbors. The right answer depends on workload SLOs and cluster operation philosophy.

**Common kernel fixes:**

A historical Linux kernel bug caused premature throttling even when quota was available. Fixed in 5.4+. Earlier kernels could be helped by:
- Increasing `cpu.cfs_period_us` to a larger value (less granular but fewer throttle events)
- Setting `kernel.sched_min_granularity_ns` and related tunables

**For latency-sensitive workloads:**

Consider the **static CPU Manager policy** (described later), which gives exclusive CPU cores to Guaranteed-class pods, completely avoiding CFS quota and its throttling.

---

## 87. What is PID pressure?

**PID pressure** is a node condition triggered when the node is running out of available Process IDs (PIDs). Linux has a finite PID space (default 32768, expandable to 4194304 on 64-bit systems), and a node hitting this limit can't fork new processes; even `kubectl exec` or running new containers will fail.

**Why PID exhaustion happens:**

- **Fork bombs:** Buggy or malicious applications spawning processes uncontrollably
- **Zombie processes:** Containers without proper init systems leave zombies that accumulate
- **Thread-heavy workloads:** Linux threads share PIDs with processes (each thread is a "task" with a PID)
- **High pod density:** Many pods × many processes per pod = aggregate PID usage

**The defense layers:**

**Layer 1: Per-pod PID limits**

The kubelet supports `--pod-max-pids` (default unlimited). Setting this caps each pod's PID usage via the cgroup PID controller (`pids.max`). A pod that hits its limit can't fork more processes within itself, but doesn't affect other pods.

**Layer 2: Node-level reservations**

`--system-reserved=pid=1000` and `--kube-reserved=pid=1000` reserve PIDs for system and Kubernetes daemons.

**Layer 3: PID pressure eviction**

When available PIDs drop below the `pid.available` eviction threshold (e.g., `--eviction-hard=pid.available<100`), kubelet:
1. Sets `PIDPressure=true` on the node
2. Taints the node `node.kubernetes.io/pid-pressure:NoSchedule`
3. Begins evicting pods using the same QoS-based ranking

**Detecting the issue:**

```bash
kubectl describe node <node-name>
# Look for: PIDPressure   True

# On the node:
cat /proc/sys/kernel/pid_max
ps -eLf | wc -l   # Count all threads/processes
```

**Why it's often missed:**

PID pressure is rarer than memory or disk pressure, so monitoring stacks often lack alerts for it. But when it hits, it's catastrophic: even debugging tools can't run on the node. Always include `kube_node_status_condition{condition="PIDPressure"}` in node alerting.

---

## 88. Explain seccomp profiles

**seccomp** (secure computing mode) is a Linux kernel feature that restricts which system calls a process can make. seccomp profiles in Kubernetes define which syscalls a container is allowed to invoke, dramatically reducing the kernel attack surface available to a compromised container.

**Why it matters:**

A typical container uses maybe 50-100 distinct syscalls. The Linux kernel exposes over 400. The unused 300+ are potential attack vectors: kernel exploits often rely on rarely-used syscalls. seccomp blocks them by default.

**Modes:**

- **Unconfined:** All syscalls allowed (default historically, still default in some setups)
- **RuntimeDefault:** Uses the container runtime's default profile, a moderate-restriction profile maintained by runtime developers (containerd, CRI-O)
- **Localhost:** A custom profile file located on the node at `/var/lib/kubelet/seccomp/<profile>.json`

**Configuration:**

```yaml
apiVersion: v1
kind: Pod
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: nginx
      securityContext:
        seccompProfile:
          type: Localhost
          localhostProfile: profiles/nginx-strict.json
```

**Profile format (BPF-based filter):**

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "syscalls": [
    {
      "names": ["read", "write", "open", "close", "stat", "fstat"],
      "action": "SCMP_ACT_ALLOW"
    },
    {
      "names": ["socket"],
      "action": "SCMP_ACT_ALLOW",
      "args": [
        {"index": 0, "value": 2, "op": "SCMP_CMP_EQ"}
      ]
    }
  ]
}
```

The `defaultAction` is what happens for any syscall not explicitly listed. Common actions:
- `SCMP_ACT_ALLOW`: Permit
- `SCMP_ACT_ERRNO`: Return an error (typically EPERM)
- `SCMP_ACT_KILL`: Terminate the process
- `SCMP_ACT_LOG`: Log but allow (useful for profile development)

**Building custom profiles:**

The **Security Profiles Operator (SPO)** can record syscalls a workload uses (via `SeccompProfile` and recording resources), then generate a minimal profile. This avoids manually figuring out which 80 syscalls your application needs.

**The RuntimeDefault default:**

From Kubernetes 1.27+, you can enable `SeccompDefault=true` feature gate to make `RuntimeDefault` the default for all Pods, not Unconfined. This is a significant security improvement and recommended for most clusters. Some workloads (those using exotic syscalls) may break and need explicit Unconfined or custom profiles.

**Performance impact:**

seccomp adds a small overhead per syscall (BPF filter evaluation). Typically <1% for normal workloads, more for syscall-heavy workloads (databases, networking-intensive apps). Generally worth the security trade-off.

---

## 89. What are Linux namespaces in containers?

**Linux namespaces** are the kernel feature that provides containers with the illusion of isolation. Each namespace type isolates a particular aspect of the system. A container is fundamentally a process running with its own set of namespaces.

**The seven namespace types:**

**1. PID namespace:** Isolates the process ID number space. The container's first process sees itself as PID 1, and can't see processes outside its namespace. This is why `ps aux` inside a container shows only its own processes.

**2. Network namespace (netns):** Isolates network interfaces, routing tables, iptables rules, and socket port numbers. Each Pod gets its own netns (shared across containers in the Pod, which is why they communicate via localhost). The CNI plugin creates this namespace and connects it to the host network.

**3. Mount namespace (mnt):** Isolates the filesystem mount points. The container sees only its own root filesystem mounted at /, even though it's actually a subdirectory on the host.

**4. UTS namespace:** Isolates hostname and domain name. Each container can have its own hostname without affecting the host.

**5. IPC namespace:** Isolates System V IPC objects (shared memory, semaphores, message queues) and POSIX message queues. Containers can't see each other's IPC objects.

**6. User namespace:** Isolates user and group IDs. UID 0 (root) inside the container can map to an unprivileged UID outside. This is the foundation of **rootless containers**: the container's "root" has no real root privileges on the host.

**7. Cgroup namespace:** Isolates the cgroup root directory. The container sees its own cgroup as the root, hiding the host's cgroup hierarchy.

**How a container is created:**

When you start a container, the runtime (containerd/runc) calls `clone()` or `unshare()` with flags like `CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWNS | ...`. The new process exists in fresh namespaces. The runtime then sets up the network interface, mounts the rootfs, etc., inside those namespaces.

**Pod-level sharing:**

In Kubernetes, all containers in a Pod share the **network and IPC namespaces** (via the pause container, which holds them open). Each container has its own **PID, mount, UTS, and user** namespaces. This is why containers in a Pod communicate via localhost but each has its own filesystem.

**With `shareProcessNamespace: true`,** containers in a Pod also share the PID namespace, allowing them to see and signal each other's processes. Useful for debug sidecars.

**Inspecting namespaces:**

```bash
# On the node, find the container's PID:
crictl inspect <container-id> | grep pid

# View its namespaces:
ls -l /proc/<pid>/ns/
# net -> net:[4026532008]
# pid -> pid:[4026532010]
# ...

# Enter a namespace:
nsenter -t <pid> -n ip addr   # Enter the network namespace
```

---

## 90. Explain AppArmor and SELinux integration

Both **AppArmor** and **SELinux** are Linux Mandatory Access Control (MAC) systems that enforce security policies the kernel can't be tricked into bypassing, even by root. Kubernetes integrates with both, though their philosophies differ significantly.

**AppArmor: path-based access control**

AppArmor confines programs based on **file paths**. A profile specifies which paths a process can read, write, or execute. Profiles are loaded into the kernel and attached to processes.

Configuration:

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    container.apparmor.security.beta.kubernetes.io/<container-name>: localhost/k8s-apparmor-profile
spec:
  containers:
    - name: app
      image: nginx
```

(In newer Kubernetes versions, this moves to `securityContext.appArmorProfile`.)

Example profile:

```
#include <tunables/global>
profile k8s-app flags=(attach_disconnected) {
  #include <abstractions/base>
  /usr/sbin/nginx ix,
  /etc/nginx/** r,
  /var/log/nginx/* w,
  deny /etc/shadow r,
}
```

Profiles must be present on every node (typically as part of the host OS provisioning). Common in Ubuntu-based clusters; Ubuntu's default AppArmor profiles cover Docker.

**SELinux: label-based access control**

SELinux assigns **labels** to every process, file, port, and network endpoint, then uses policy rules to govern interactions between labels. It's more granular but more complex than AppArmor.

Each container gets a unique SELinux **context** (e.g., `system_u:system_r:container_t:s0:c123,c456`). The `c123,c456` is the MCS (Multi-Category Security) label, randomly assigned per container. Two containers with different MCS labels can't access each other's files even if they're both `container_t`.

Configuration:

```yaml
spec:
  securityContext:
    seLinuxOptions:
      level: "s0:c123,c456"
      type: "container_t"
      user: "system_u"
      role: "system_r"
```

Common in Red Hat-based distributions (RHEL, CentOS, Fedora) where SELinux is enforcing by default.

**Practical differences:**

- **AppArmor** is easier to write profiles for; path-based syntax is intuitive
- **SELinux** is harder to write but more powerful; label-based system is fundamentally more expressive
- **AppArmor** is the default on Ubuntu/Debian-derived distros
- **SELinux** is the default on RHEL/CentOS/Fedora
- You typically pick one based on your host OS, not both

**Performance:**

Both are kernel-side and very fast. AppArmor has slightly less overhead in benchmarks but the difference rarely matters.

**Integration with PodSecurity:**

The PodSecurity admission controller (replacing PodSecurityPolicy) can enforce that pods declare AppArmor/SELinux configurations matching the namespace's security level (privileged/baseline/restricted).

---

## 91. Difference between rootless and privileged containers

These represent opposite ends of the container privilege spectrum.

**Privileged containers:**

A privileged container runs with effectively all kernel capabilities, full device access, and disabled security restrictions. From the kernel's perspective, the container is essentially equivalent to running directly on the host.

```yaml
securityContext:
  privileged: true
```

What this means in practice:
- All Linux capabilities granted (CAP_SYS_ADMIN, CAP_NET_ADMIN, etc.)
- AppArmor/SELinux often disabled or set to unconfined
- All devices accessible via `/dev`
- seccomp filter disabled
- Can mount filesystems, modify kernel parameters, load kernel modules
- Can break out of the container with relative ease

**When privileged is legitimately needed:**
- CNI plugins (configuring host network)
- CSI drivers (mounting storage)
- Monitoring agents accessing `/proc` and `/sys` deeply
- GPU drivers and device plugins
- Pods that need to manage other containers

In all these cases, the privileged Pod is itself part of the infrastructure, not user workload.

**Rootless containers:**

A rootless container runs as a non-root user *both inside and outside the container*. This typically requires **user namespace mapping**, where UID 0 inside the container maps to a high-numbered unprivileged UID on the host (e.g., 100000).

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
  containers:
    - name: app
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop: ["ALL"]
        readOnlyRootFilesystem: true
```

**Why rootless is safer:**

Even if an attacker compromises the application, they don't have root inside the container. Even if they escape the container (e.g., via a kernel exploit), they don't have root on the host. They're an unprivileged user with limited damage potential.

**The user namespace feature:**

Kubernetes 1.30+ supports the `userNamespaces` feature gate, allowing pods to use Linux user namespaces. With `hostUsers: false`, the pod runs in its own user namespace, providing strong UID isolation.

```yaml
spec:
  hostUsers: false   # Enable user namespace
```

This is qualitatively different from just `runAsUser: 1000`, which still uses the host's UID 1000. With user namespaces, the container's UID 0 maps to a per-pod range like 65536-131072 on the host.

**The middle ground:**

Most production workloads should be:
- Not privileged
- Run as non-root
- Drop all capabilities, add only what's needed
- Use read-only root filesystem
- Use RuntimeDefault seccomp profile
- Use AppArmor/SELinux confinement

This is what the PodSecurity admission controller's "restricted" profile enforces.

---

## 92. Explain gVisor and Kata Containers

Both are **alternative container runtimes** that provide stronger isolation than standard runc-based containers, at the cost of some performance and compatibility. They exist because traditional containers share the host kernel, which is a single point of failure for security.

**The fundamental problem they solve:**

A standard container is just a process group on the host. A kernel vulnerability (and Linux has many CVEs per year) can let a container escape into the host. For multi-tenant clusters (running untrusted code, public CI services, function-as-a-service platforms), this risk is unacceptable.

**gVisor: a user-space kernel**

gVisor (specifically its runsc runtime) implements a **user-space kernel** in Go that intercepts container syscalls and handles them itself, instead of passing them directly to the host kernel. It implements the Linux syscall ABI in a separate process.

Architecture:
```
Container app → syscall → gVisor Sentry (user-space kernel) → minimal host syscalls
```

The host kernel sees gVisor making only a small whitelist of syscalls, dramatically shrinking the attack surface. If an attacker exploits a bug in the container, they're trapped in the Sentry, which is itself sandboxed.

Tradeoffs:
- **Performance:** Syscall-heavy workloads (databases, network proxies) can see 10-30% slowdown
- **Compatibility:** Some syscalls aren't implemented; some applications break (especially those using exotic features like ptrace, certain mmap flags)
- **Memory overhead:** Sentry uses extra memory per container

**Kata Containers: lightweight VMs**

Kata runs each container (or Pod) inside a **lightweight virtual machine** with its own kernel. Each Pod gets a real VM (typically with QEMU or Firecracker as hypervisor), and the containers run inside that VM's kernel.

Architecture:
```
Pod → Lightweight VM (own kernel) → containerd-shim-kata → kata-agent inside VM
```

The isolation is hardware-enforced: even a kernel exploit only compromises the VM's kernel, not the host. This is the same isolation model as cloud providers use between tenants.

Tradeoffs:
- **Performance:** VM startup adds latency (~100-500ms), networking has extra layer
- **Memory overhead:** Each VM consumes additional memory (kernel, agent, etc., typically 50-100MB)
- **Compatibility:** Very high, since it's a real Linux kernel
- **Hardware requirements:** Needs nested virtualization or bare metal

**Using them in Kubernetes:**

Both are integrated via the **RuntimeClass** feature:

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
---
apiVersion: v1
kind: Pod
spec:
  runtimeClassName: gvisor
  containers:
    - name: app
      image: nginx
```

The node's container runtime (containerd/CRI-O) is configured with multiple handlers, and the RuntimeClass selects which to use per Pod.

**When to use which:**

- **gVisor:** When you need stronger isolation but want lighter overhead than VMs, and your workloads use standard syscalls. Google uses it for App Engine and Cloud Run.
- **Kata:** When you need true hardware-enforced isolation (multi-tenant SaaS, regulatory compliance), and can afford the overhead.
- **runc (default):** For trusted workloads within your organization.

---

## 93. How does Kubernetes checkpointing work?

**Container checkpointing** is the ability to save the complete state of a running container to disk (a "checkpoint"), and later restore it to running state, potentially on a different node. This is fundamentally different from a snapshot of the filesystem; it captures memory, open files, sockets, and process state.

**The underlying technology: CRIU**

Checkpointing in Linux is provided by **CRIU** (Checkpoint/Restore In Userspace), a project that uses the kernel's ptrace and other interfaces to freeze processes, dump their state, and later restore them.

When CRIU checkpoints a process tree:
- All memory pages are dumped to files
- Open file descriptors are recorded (with paths)
- Network sockets are recorded (TCP state, queued data)
- Signal masks, namespaces, cgroups are captured
- The process is either left running or killed

**Kubernetes integration:**

The `ContainerCheckpoint` feature, alpha in 1.25 and beta in 1.30+, exposes checkpointing through the kubelet API. It's a forensic/debugging feature, not a general-purpose migration tool yet.

Trigger via kubelet API:

```bash
curl -X POST "https://<node>:10250/checkpoint/<namespace>/<pod>/<container>" \
  --cert ... --key ...
```

This produces a checkpoint archive (a tar file containing CRIU images) on the node, typically at `/var/lib/kubelet/checkpoints/`. The archive can be:

1. **Analyzed forensically:** Inspect memory contents of a compromised container without disturbing it further
2. **Restored as a container image:** Built into a new image with `buildah` or similar, then run elsewhere
3. **Used for migration:** Move running state to another node (experimental)

**Limitations and challenges:**

- **Stateful connections break:** TCP connections checkpointed and restored elsewhere usually can't resume because peer state, IP addresses, and routing differ
- **External dependencies:** Open files on hostPath volumes, GPU device handles, and similar external state may not restore correctly
- **Security implications:** A checkpoint contains all memory contents, including secrets. Treat checkpoints as highly sensitive
- **Image compatibility:** Restoration must occur on a kernel compatible with the original

**Future direction:**

Live pod migration via checkpointing is an active research area. Use cases include:
- Draining a node for maintenance without restart-based downtime
- Moving a long-running workload (model training, simulation) to a different node
- Rapid forensic capture of suspicious containers

For now, checkpoint/restore is mainly a powerful debugging tool, not a production HA mechanism.

---

## 94. Explain ephemeral storage management

**Ephemeral storage** is local disk space that a Pod uses for its lifetime but loses when it's deleted or rescheduled. Unlike persistent volumes, it has no durability guarantees. Kubernetes manages and limits it to prevent runaway disk usage from destabilizing nodes.

**What counts as ephemeral storage:**

For a Pod, ephemeral storage includes:
- **Container writable layer:** Anything written to the container's root filesystem
- **emptyDir volumes:** Including `emptyDir.medium: ""` (disk-backed) and the default
- **Container logs:** Stdout/stderr captured by the kubelet

Notably **not** included:
- emptyDir with `medium: Memory` (counts against memory)
- PersistentVolumes (these are durable)
- hostPath volumes (these are host disk, separately managed)

**Requests and limits:**

```yaml
resources:
  requests:
    ephemeral-storage: "1Gi"
  limits:
    ephemeral-storage: "2Gi"
```

The scheduler uses requests to ensure the node has enough free space. Limits cause eviction if exceeded.

**Where it's stored:**

By default, all ephemeral storage lives on the kubelet's root filesystem (`/var/lib/kubelet`). With a split image filesystem configuration, container images go to a separate disk (`imagefs`), and writable layers and emptyDirs stay on `nodefs`. The kubelet monitors both separately.

**How enforcement works:**

Kubelet periodically (default every 10s) calculates each Pod's ephemeral storage usage by summing:
- Writable layer disk usage (queried from the container runtime)
- emptyDir directory sizes (du-like scan)
- Container log file sizes

If a Pod exceeds its `ephemeral-storage` limit, kubelet evicts it. Note: this is a polling check, not instantaneous; a Pod can briefly exceed its limit between polls.

**Node-level disk pressure:**

Separately, the kubelet monitors total node disk usage. When `nodefs.available` or `imagefs.available` crosses eviction thresholds, kubelet begins evicting Pods (regardless of their individual ephemeral-storage limits) and also performs **garbage collection** on unused images and containers.

**Garbage collection:**

The kubelet has two GC mechanisms:
- **Container GC:** Removes stopped containers older than the configured threshold
- **Image GC:** Removes unused images when disk usage crosses `--image-gc-high-threshold` (default 85%), down to `--image-gc-low-threshold` (default 80%)

If image GC and container GC can't free enough space, eviction begins.

**Common issues:**

1. **Log files filling disk:** An application logging gigabytes/hour to stdout with no log rotation eats nodefs space. Kubernetes uses container runtime log rotation (containerd's `max_size` config), but it must be configured.
2. **emptyDir leaks:** A Pod writing temp files to emptyDir without cleanup can fill the node. Use ephemeral-storage limits.
3. **Image bloat:** Pulling many large images (especially with `imagePullPolicy: Always`) without GC tuning can fill the imagefs.

**Generic Ephemeral Volumes:**

A newer feature lets Pods request ephemeral volumes from any storage class (including CSI drivers), getting per-pod isolation and storage class features (encryption, IOPS limits) without the durability of a normal PVC.

---

## 95. What is container runtime shim?

A **container runtime shim** is a small process that sits between the container manager (containerd or CRI-O) and the low-level runtime (runc) that actually starts containers. It's a critical architectural component that decouples container manager upgrades from running containers.

**Why shims exist:**

In the original Docker architecture, the Docker daemon was the parent process of every container. Restarting Docker killed all containers, which is unacceptable. The shim was introduced to solve this: each container has a small parent (the shim) that survives container manager restarts.

**Architecture in modern Kubernetes:**

```
kubelet 
   ↓ (CRI gRPC)
containerd 
   ↓ (creates shim per container/pod)
containerd-shim-runc-v2 (one per pod, persists across containerd restarts)
   ↓ (manages)
runc 
   ↓ (briefly runs, then exits)
container processes
```

**What the shim does:**

1. **Holds the container's stdio:** The shim is the parent of the container process, so it owns the stdin/stdout/stderr file descriptors. It pipes them to log files or fifos.
2. **Reports exit codes:** When the container exits, the shim captures the exit code and reports it to containerd.
3. **Survives container manager restarts:** If containerd is restarted (upgrade, crash), the shim keeps running. When containerd comes back, it reconnects to existing shims via Unix sockets.
4. **Handles signals:** Forwards termination signals appropriately.
5. **Manages container lifecycle:** Pause, resume, exec into container all go through the shim.

**Why containerd-shim-runc-v2 is one per pod, not per container:**

In current versions, the v2 shim manages all containers in a pod (multiple `runc` invocations from the same shim). This reduces memory overhead (one shim per pod instead of per container) and simplifies log streaming.

**Shim API:**

The shim implements a gRPC service (the **Runtime v2 API**) with methods like `Create`, `Start`, `Delete`, `Exec`, `Kill`, `Wait`, `Pause`, `Resume`. Different shim binaries can be plugged in to support different runtimes:

- `containerd-shim-runc-v2`: Standard runc (default)
- `containerd-shim-kata-v2`: Kata Containers
- `containerd-shim-runsc-v1`: gVisor

This is what makes RuntimeClass work: selecting a runtimeClassName routes to a different shim binary.

**Inspecting shims on a node:**

```bash
ps aux | grep containerd-shim
# Each line is one shim process, with --id <container-id>

# Find the shim for a specific pod:
ps -ef | grep <pod-id>
```

**Why this matters operationally:**

- Upgrading containerd doesn't restart your pods (because shims persist)
- Each shim consumes a small amount of memory (~5-10MB); thousands of pods = noticeable overhead
- Shim crashes can leave orphaned containers; usually handled by containerd's reconciliation logic on restart

---

## 96. Explain cgroup v2 support

**cgroups (control groups)** are the Linux kernel feature that limits, accounts for, and isolates resource usage (CPU, memory, I/O, etc.) of process groups. They underpin all container resource limits in Kubernetes. **cgroup v2** is the major rewrite of the original cgroup v1, addressing fundamental design flaws.

**cgroup v1 problems:**

- **Separate hierarchies per controller:** Each resource type (CPU, memory, etc.) had its own filesystem hierarchy. A process could be in different positions in different hierarchies, leading to confusing semantics.
- **Inconsistent interfaces:** Each controller had its own quirky API
- **Weak memory and I/O integration:** Memory and I/O accounting didn't cooperate well, leading to issues with writeback throttling
- **No unified pressure metrics:** PSI (Pressure Stall Information) wasn't available

**cgroup v2 improvements:**

- **Unified hierarchy:** All controllers operate on the same hierarchy, located at `/sys/fs/cgroup/`. Each cgroup has all relevant controllers, simplifying reasoning.
- **Consistent interface:** Standardized file naming (`<controller>.<metric>`)
- **PSI metrics:** Pressure Stall Information at every cgroup level, exposing memory, CPU, and I/O pressure
- **Better OOM handling:** More predictable OOM killer behavior
- **Improved I/O control:** Real cgroup-aware I/O bandwidth limits, not just blkio's broken weights

**Kubernetes support:**

Kubernetes supports cgroup v2 since 1.19 (alpha), GA in 1.25+. To use it:

1. The node OS must boot with cgroup v2 (`systemd.unified_cgroup_hierarchy=1` kernel parameter, default in many modern distros)
2. The container runtime must support cgroup v2 (containerd 1.4+, CRI-O 1.20+)
3. runc must be 1.0+

You can check:

```bash
# On the node:
stat -fc %T /sys/fs/cgroup/
# Returns "cgroup2fs" for v2, "tmpfs" for v1
```

**Distro defaults:**

- Fedora 31+: v2 by default
- Ubuntu 21.10+: v2 by default
- RHEL 9+: v2 by default
- Older distros: usually v1 (hybrid mode possible but not recommended)

**Important changes for Kubernetes:**

**Memory QoS:** cgroup v2 enables a feature called **MemoryQoS** in kubelet, which uses cgroup v2's `memory.min` and `memory.high` to provide soft memory guarantees and gentler throttling before hard limits trigger OOM. With v1, only hard limits (`memory.limit_in_bytes`) existed.

**CPU bandwidth:** Same CFS quota mechanism, but with cleaner controls (`cpu.max` instead of separate `cpu.cfs_quota_us` and `cpu.cfs_period_us`).

**PSI metrics:** Kubelet can expose Pressure Stall Information per pod, giving much better insight into actual resource contention than just utilization. A pod showing low CPU usage but high CPU PSI is being throttled.

**Container compatibility:**

Some older container images and applications query `/sys/fs/cgroup` directly and expect v1 layout. These can break on v2 nodes. Examples:
- Old JVMs (pre-15) reading memory limits from `/sys/fs/cgroup/memory/memory.limit_in_bytes` (v1 path)
- Some monitoring agents

Solutions: upgrade the application, or use distros offering hybrid mode (not recommended long-term).

---

## 97. What are device plugins?

**Device plugins** are a Kubernetes extension mechanism that allows nodes to advertise hardware devices (GPUs, FPGAs, NICs, specialized accelerators) to the scheduler and to mount them into containers. They're how Kubernetes supports hardware that isn't a "standard" resource like CPU or memory.

**The problem they solve:**

Kubernetes has a fixed set of compute resources it understands natively: cpu, memory, ephemeral-storage, hugepages. But clusters increasingly need to schedule based on hardware like NVIDIA GPUs, AMD GPUs, Habana accelerators, SR-IOV NICs, RDMA devices, or FPGAs. Hard-coding support for each in Kubernetes wouldn't scale.

**The device plugin architecture:**

A device plugin is a vendor-provided **DaemonSet** that:

1. Discovers devices on the node (e.g., scans PCI for NVIDIA GPUs)
2. Registers with the kubelet via a Unix socket at `/var/lib/kubelet/device-plugins/`
3. Advertises devices as **extended resources** (e.g., `nvidia.com/gpu: 4`)
4. Handles allocation requests when a Pod requesting the resource is scheduled
5. Performs container preparation (mounting device files, setting environment variables, mounting libraries)

**Lifecycle of a GPU pod:**

1. Cluster admin installs the NVIDIA device plugin DaemonSet
2. Plugin starts on each GPU node, detects 4 GPUs, reports `nvidia.com/gpu: 4` to kubelet
3. Kubelet updates the Node object: `status.allocatable.nvidia.com/gpu: 4`
4. User submits a Pod with `resources.limits.nvidia.com/gpu: 1`
5. Scheduler finds a node with available GPU resource and binds the Pod
6. Kubelet, before starting the container, calls the device plugin's `Allocate()` gRPC method, asking it to prepare 1 GPU
7. Plugin responds with which device files to mount (`/dev/nvidia0`), environment variables (`NVIDIA_VISIBLE_DEVICES=0`), and any mounts (CUDA libraries)
8. Kubelet passes these to the container runtime, which sets them up when creating the container
9. Container starts with access to the GPU

**The gRPC API:**

The device plugin must implement:

- `ListAndWatch`: Stream available devices and their health to kubelet (called continuously)
- `Allocate`: Prepare a container's device access (called per-container at startup)
- `GetDevicePluginOptions`: Capabilities advertisement
- `PreStartContainer` (optional): Called before container starts
- `GetPreferredAllocation` (optional): Hint preferred device selection for topology

**Resource isolation gotcha:**

Extended resources are **integer-only** and **not overcommittable**. You can't request 0.5 GPUs through this mechanism. For GPU sharing, vendors like NVIDIA provide separate mechanisms (MIG, time-slicing, MPS) configured through the plugin.

**Health management:**

The plugin continuously monitors device health via `ListAndWatch`. If a GPU fails, the plugin reports it as unhealthy, and kubelet removes it from allocatable resources. Pods using that specific GPU may continue running but won't be rescheduled there.

**Examples of device plugins:**

- `nvidia-device-plugin`: NVIDIA GPUs
- `intel-device-plugins`: Intel QAT, GPU, FPGA
- `amdgpu-device-plugin`: AMD GPUs
- `sriov-network-device-plugin`: SR-IOV virtual functions
- `kubevirt-device-plugin`: KVM devices for virtualization

---

## 98. Explain GPU scheduling in Kubernetes

GPU scheduling builds on device plugins but involves additional complexity due to GPU's unique characteristics: high cost, indivisibility (traditionally), topology sensitivity, and software stack requirements.

**The basic model:**

A Pod requests GPUs as an extended resource:

```yaml
spec:
  containers:
    - name: trainer
      image: pytorch/pytorch
      resources:
        limits:
          nvidia.com/gpu: 2
```

The scheduler finds a node with 2 available GPUs and binds. The device plugin allocates 2 specific GPUs and provides them to the container.

**Whole-GPU allocation (default):**

By default, GPUs are allocated whole. A node with 4 GPUs can run at most 4 pods each requesting `nvidia.com/gpu: 1`. This is wasteful for inference workloads that only need part of a GPU.

**GPU sharing strategies:**

**1. Time-slicing (NVIDIA):**

The NVIDIA device plugin can be configured to advertise each physical GPU as multiple "logical" GPUs:

```yaml
# Plugin config:
sharing:
  timeSlicing:
    replicas: 4   # Each physical GPU appears as 4
```

Now `nvidia.com/gpu: 1` requests get 1/4 of a GPU's time. There's no memory or compute isolation; all sharing pods see the full GPU and contend at the GPU scheduler level.

**2. MIG (Multi-Instance GPU):**

NVIDIA A100, H100, and later support hardware partitioning. A single A100 can be split into up to 7 MIG instances with dedicated memory and compute. Each MIG instance appears as a separate device:

```yaml
resources:
  limits:
    nvidia.com/mig-1g.5gb: 1   # 1 compute slice, 5GB memory
```

This provides true isolation between pods.

**3. MPS (Multi-Process Service):**

CUDA MPS allows multiple processes to share a GPU with reduced context-switch overhead. Less isolated than MIG but more efficient than naive time-slicing.

**Topology and NUMA:**

In multi-GPU nodes, GPUs are connected via NVLink, NVSwitch, or PCIe. A training job requesting 4 GPUs performs much better when those 4 GPUs are NVLink-connected. The default scheduler doesn't know this. Solutions:

- **NVIDIA GPU Operator** with topology-aware scheduling
- **Topology Manager** (in kubelet) coordinates GPU allocation with NUMA placement
- **Volcano scheduler:** Alternative scheduler with gang scheduling and GPU topology awareness

**Drivers and runtime:**

Beyond just scheduling, GPU pods need:
- NVIDIA drivers on the host (installed via DaemonSet or pre-installed)
- NVIDIA Container Toolkit configured with containerd/CRI-O
- The CUDA libraries (typically baked into the container image)

The **NVIDIA GPU Operator** automates this stack: drivers, container runtime config, device plugin, monitoring, MIG configuration, all managed declaratively.

**Common production patterns:**

- **Dedicated GPU node pools:** Taint GPU nodes with `nvidia.com/gpu=true:NoSchedule`, require tolerations from GPU workloads
- **Priority classes:** GPU jobs use high-priority classes to preempt less important workloads when GPUs are scarce
- **Gang scheduling:** For distributed training (PyTorch DDP, Horovod), use a scheduler like Volcano that won't start any of the pods until all can start, avoiding deadlock on partial allocation
- **Resource monitoring:** Scrape `dcgm-exporter` for GPU utilization, memory, temperature, errors

---

## 99. How does Topology Manager work?

**Topology Manager** is a kubelet component that coordinates resource allocation across multiple resource managers (CPU Manager, Memory Manager, Device Manager) to ensure topology-aligned placement. Without it, a Pod might get CPUs on NUMA node 0 but a GPU on NUMA node 1, causing severe performance degradation for latency-sensitive or high-bandwidth workloads.

**The problem:**

Modern servers have **NUMA** (Non-Uniform Memory Access) topology. A typical 2-socket server has two NUMA nodes; each socket has its own memory controller, attached PCIe devices (NICs, GPUs), and local memory. Accessing memory or devices on the *other* NUMA node goes through inter-socket interconnect (Intel UPI, AMD Infinity Fabric), adding latency and reducing bandwidth.

For:
- High-frequency trading apps: cross-NUMA access doubles latency
- ML training: GPU-CPU communication across NUMA halves throughput
- DPDK networking: NIC-CPU misalignment kills performance

**Topology Manager's role:**

It acts as a coordinator. Before kubelet starts a container, each resource manager (CPU, Memory, Device) provides Topology Manager with a list of allocation possibilities (`TopologyHints`), each tagged with NUMA affinity. Topology Manager merges these hints and selects an aligned allocation, then instructs each manager to allocate accordingly.

**Policies:**

Configured via kubelet flag `--topology-manager-policy`:

- **none** (default): No topology awareness; each manager allocates independently
- **best-effort:** Topology Manager tries to find an aligned allocation. If impossible, allocates anyway
- **restricted:** If no aligned allocation is found, the Pod fails admission with `Topology Affinity error`
- **single-numa-node:** Requires all resources to come from a single NUMA node; fails admission otherwise

**The hint generation process:**

For a Pod requesting 4 CPUs and 1 GPU on a 2-NUMA-node machine:

1. CPU Manager identifies that 4 CPUs are available on NUMA 0 and NUMA 1 separately, generates hints: `{NUMA: 0, preferred: true}, {NUMA: 1, preferred: true}`
2. Device Manager identifies that the GPU is on NUMA 1: `{NUMA: 1, preferred: true}`
3. Topology Manager intersects these: only NUMA 1 satisfies both. Selects NUMA 1.
4. Instructs CPU Manager to allocate 4 CPUs from NUMA 1's CPU pool
5. Instructs Device Manager to allocate the NUMA 1 GPU

**Scope: pod vs. container:**

- `--topology-manager-scope=container` (default): Each container independently aligned. A Pod with multiple containers might have container A on NUMA 0 and container B on NUMA 1.
- `--topology-manager-scope=pod`: All containers in the pod aligned to the same NUMA node, important for inter-container communication via shared memory or localhost.

**Limitations:**

- Only QoS class **Guaranteed** with integer CPU requests benefits fully (the static CPU policy only handles these)
- Topology Manager works on a single node; cross-node optimization requires a scheduler-level approach
- Requires Memory Manager and CPU Manager configured correctly to be effective

**Operational considerations:**

Enabling `single-numa-node` policy on a cluster can cause unexpected scheduling failures: pods that fit on a node by resource count but don't fit on a single NUMA node will fail admission. Test thoroughly before enabling.

---

## 100. Explain NUMA-aware scheduling

NUMA-aware scheduling extends Topology Manager's node-level coordination to the **cluster-level scheduling decisions**. While Topology Manager prevents misaligned allocation *on a node*, NUMA-aware scheduling ensures the scheduler picks nodes where alignment is *possible* in the first place.

**The fundamental gap:**

Consider a 2-NUMA-node cluster node with:
- 32 total CPUs (16 per NUMA node)
- 4 GPUs (2 per NUMA node)
- Currently using 14 CPUs on NUMA 0 and 0 CPUs on NUMA 1

A Pod requesting 8 CPUs and 1 GPU with `single-numa-node` policy:
- The default scheduler sees: "Node has 18 free CPUs and 4 GPUs, fits!" and binds the pod
- The kubelet's Topology Manager can't find 8 CPUs + 1 GPU on a single NUMA node (NUMA 0 has 2 free CPUs, NUMA 1 has 16 free but you also need the right GPU)
- The pod fails admission, gets stuck in `TopologyAffinityError`

The scheduler doesn't know about NUMA topology, so it makes incorrect placement decisions for topology-sensitive workloads.

**NUMA-aware scheduler plugins:**

The Kubernetes scheduler is extensible via the scheduler framework. NUMA-aware plugins include:

**1. NodeResourceTopology plugin (sigs.k8s.io/scheduler-plugins):**

Works with a companion DaemonSet (Resource Topology Exporter) that publishes each node's NUMA topology as a `NodeResourceTopology` Custom Resource:

```yaml
apiVersion: topology.node.k8s.io/v1alpha1
kind: NodeResourceTopology
metadata:
  name: node-1
topologyPolicies: ["SingleNUMANodeContainerLevel"]
zones:
  - name: node-0   # NUMA node 0
    type: Node
    resources:
      - name: cpu
        capacity: "16"
        allocatable: "16"
        available: "2"
      - name: nvidia.com/gpu
        capacity: "2"
        available: "1"
  - name: node-1   # NUMA node 1
    ...
```

The scheduler plugin reads these CRs during scheduling and only binds the pod to nodes where alignment is feasible.

**2. Volcano scheduler:**

A batch-oriented scheduler with built-in NUMA awareness, gang scheduling, and queue-based fair sharing. Common in HPC and ML workloads.

**3. Cloud-provider-specific schedulers:**

Some cloud providers offer NUMA-aware scheduling extensions tied to their hardware (e.g., AWS for instance-specific topology).

**How it works end-to-end:**

1. Resource Topology Exporter runs on each node, queries kubelet's `/podresources` API and `/sys/devices/system/node/` to determine current per-NUMA resource availability
2. Publishes NodeResourceTopology CR (updated every few seconds)
3. NodeResourceTopology scheduler plugin filters nodes during scheduling:
   - For each candidate node, simulates the allocation across NUMA zones
   - Only passes nodes where the pod's resources can fit on a single NUMA zone (for SingleNUMANodeContainerLevel policy)
4. Scheduler binds pod to a node where Topology Manager will succeed
5. Kubelet's Topology Manager performs the actual aligned allocation

**Performance impact when working correctly:**

For NUMA-sensitive workloads, the difference between aligned and misaligned placement can be:
- 20-40% throughput for memory-bandwidth-bound applications
- 2-5x for cross-NUMA inter-process communication
- Significant tail latency reduction for trading and real-time systems

**Operational tradeoffs:**

- **Bin-packing efficiency drops:** NUMA-aware scheduling can leave nodes "fragmented" where resources are free but unalignable. Over time, this reduces effective cluster capacity by 5-15%.
- **Complexity:** More moving parts (topology exporter, custom scheduler, careful policy configuration)
- **Scheduling latency:** Considering NUMA topology adds work per scheduling decision

**When to invest in NUMA-awareness:**

- Latency-critical applications (financial trading, real-time bidding)
- High-bandwidth networking (DPDK, SR-IOV, RDMA workloads)
- ML training with multi-GPU + CPU coordination
- HPC workloads

For general web services and microservices, NUMA-awareness adds complexity without meaningful benefit; stick with default scheduling.

# Kubernetes Troubleshooting & Production Scenarios (101-110)

## 101. A Pod is stuck in Pending state. How do you troubleshoot?

A Pending pod means the API server has accepted it but it hasn't been scheduled to a node yet, or the kubelet hasn't been able to start it. The diagnosis flow follows the pod's lifecycle.

**Step 1: Get the pod's events**

```bash
kubectl describe pod <pod-name> -n <namespace>
```

The `Events` section at the bottom is almost always where the answer lies. Common messages:

- `0/10 nodes are available: 5 Insufficient cpu, 5 Insufficient memory` — Resource shortage
- `0/10 nodes are available: 10 node(s) had untolerated taint` — Taint/toleration mismatch
- `0/10 nodes are available: 10 node(s) didn't match node selector` — nodeSelector or affinity rules
- `pod has unbound immediate PersistentVolumeClaims` — PVC isn't bound
- `FailedScheduling: 0/10 nodes are available: 10 node(s) had volume node affinity conflict` — PV is on a different zone than the schedulable nodes

**Step 2: Categorize the failure**

**Category A: Scheduling failure**

The scheduler can't find a fitting node. Check:

```bash
# What resources is the pod requesting?
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[*].resources}'

# What's available on nodes?
kubectl describe nodes | grep -A 5 "Allocated resources"

# Are there taints?
kubectl get nodes -o json | jq '.items[].spec.taints'

# Node selector or affinity issues?
kubectl get pod <pod-name> -o yaml | grep -A 10 -E "nodeSelector|affinity"
```

**Category B: Volume binding failure**

```bash
kubectl get pvc -n <namespace>
# Look for Pending PVCs

kubectl describe pvc <pvc-name>
# Check events; common issues: no matching PV, storage class doesn't exist,
# provisioner failing, zone mismatch with WaitForFirstConsumer

kubectl get storageclass
# Verify the storage class exists and has correct provisioner
```

**Category C: Image pull issues (pod scheduled but stuck)**

If the pod has a node assigned but is still Pending or in ImagePullBackOff:

```bash
kubectl describe pod <pod-name> | grep -A 5 Events
# Look for: Failed to pull image, ErrImagePull, ImagePullBackOff
```

Check:
- Image name typo or wrong tag
- Private registry credentials missing (`imagePullSecrets`)
- Network access to registry from nodes
- Image too large and timing out

**Category D: Admission rejection (rarer)**

If the pod was created but a quota or admission webhook is blocking creation, the pod won't even exist. But sometimes pods pass creation and then fail admission at kubelet level (e.g., PodSecurity violations on pre-existing pods).

**Step 3: Check scheduler logs if the cause is unclear**

```bash
kubectl logs -n kube-system <kube-scheduler-pod> | grep <pod-name>
```

Useful when standard events are vague (custom schedulers, scheduler extenders, complex affinity).

**Common production root causes:**

- A new node pool taint wasn't matched by existing workloads
- ResourceQuota exhausted in the namespace
- PodDisruptionBudget preventing eviction-driven rescheduling
- HPA scaled up but cluster autoscaler hasn't provisioned nodes yet (look for `cluster-autoscaler` events)
- StorageClass `volumeBindingMode: WaitForFirstConsumer` combined with topology constraints

---

## 102. A Deployment rollout is stuck. What will you check?

A stuck rollout means `kubectl rollout status deployment/<name>` doesn't complete, or `kubectl get deployment` shows `READY 2/3` indefinitely. The Deployment controller wants to progress but something is blocking it.

**Step 1: Check rollout status and history**

```bash
kubectl rollout status deployment/<name> -n <namespace>
# Often hangs at: "Waiting for deployment 'X' rollout to finish: 2 of 3 updated replicas are available..."

kubectl describe deployment <name>
# Look at Conditions, especially: Progressing, Available
# Reason: ProgressDeadlineExceeded means the rollout failed
```

**Step 2: Inspect the ReplicaSets**

A rolling update creates a new ReplicaSet while scaling down the old one. The new RS not reaching desired count is the visible symptom.

```bash
kubectl get rs -n <namespace> -l app=<app-label>
# Old RS:     DESIRED 1   CURRENT 1   READY 1
# New RS:     DESIRED 3   CURRENT 3   READY 2   ← One pod not Ready

kubectl describe rs <new-rs-name>
# Events show why new pods aren't coming up
```

**Step 3: Examine the unhealthy pod**

```bash
kubectl get pods -l app=<app> -o wide
# Find the pod that's not Ready

kubectl describe pod <unhealthy-pod>
kubectl logs <unhealthy-pod> --previous   # Logs from the crashed instance
```

**Common stuck rollout causes:**

**Cause 1: New pod CrashLoopBackOff**

The new image has a bug, missing env var, or config issue. Old pods stay alive (deployment respects `maxUnavailable`), new pods never become Ready. The rollout pauses indefinitely until you `kubectl rollout undo` or fix the image.

**Cause 2: Readiness probe failing**

The pod is running but never passes its readiness probe. Common reasons:
- Probe endpoint not implemented in the new version
- Probe timing too aggressive (initialDelaySeconds too low)
- Application takes longer to start than the failureThreshold * periodSeconds allows

**Cause 3: PodDisruptionBudget blocking old pod termination**

If `maxUnavailable: 0` and you have a PDB requiring `minAvailable`, the deployment can't terminate old pods because doing so would violate the PDB.

```bash
kubectl get pdb -n <namespace>
kubectl describe pdb <pdb-name>
# Allowed disruptions: 0   ← This is blocking
```

**Cause 4: Resource constraints**

The new pod requests resources that no node can satisfy. New pods stuck Pending, old pods remain (because we can't terminate them until new ones are Ready).

**Cause 5: Image pull failure**

```bash
kubectl describe pod <new-pod> | grep -A 3 "Failed"
# ImagePullBackOff or ErrImagePull
```

**Cause 6: Quota exhausted**

The namespace ResourceQuota is full; new pods can't be created.

```bash
kubectl describe quota -n <namespace>
```

**Cause 7: Progress deadline exceeded**

By default, after 600 seconds without progress, the Deployment marks `Progressing=False` with reason `ProgressDeadlineExceeded`. This doesn't undo the rollout; it just flags it as failed. You need to investigate and fix or rollback.

**Step 4: Rollback if needed**

```bash
kubectl rollout undo deployment/<name>
# Or to a specific revision:
kubectl rollout history deployment/<name>
kubectl rollout undo deployment/<name> --to-revision=3
```

**Prevention:**

- Always set readiness probes correctly
- Use canary or blue-green deployments via Argo Rollouts or Flagger for risky changes
- Set `progressDeadlineSeconds` to a realistic value
- Monitor `kube_deployment_status_condition` metrics for failed rollouts

---

## 103. Pods are continuously restarting. How do you investigate?

A pod with high RESTARTS count (`kubectl get pod` showing `5`, `12`, climbing) is in a CrashLoopBackOff or repeated kill/restart pattern. The kubelet restarts containers per the pod's `restartPolicy` (default Always for Deployments).

**Step 1: Identify the restart pattern**

```bash
kubectl get pod <pod-name>
# NAME    READY   STATUS             RESTARTS   AGE
# app-x   0/1     CrashLoopBackOff   7          12m
```

`CrashLoopBackOff` is kubelet's exponential backoff state between restart attempts (10s, 20s, 40s, 80s, up to 5 min).

**Step 2: Get the exit reason**

```bash
kubectl describe pod <pod-name>
```

Look at `Last State`:

```
Last State:     Terminated
  Reason:       Error          # Or OOMKilled, Completed
  Exit Code:    1              # 0 = clean exit, 137 = SIGKILL/OOM, 143 = SIGTERM
  Started:      ...
  Finished:     ...
```

**Common exit codes:**

- **0**: Clean exit (but for long-running processes, this means the main process exited unexpectedly)
- **1**: Generic application error
- **137**: SIGKILL (usually OOMKilled or a deliberate kill)
- **139**: SIGSEGV (segmentation fault, application crash)
- **143**: SIGTERM (graceful termination signal received)
- **125**: Container runtime error (often misconfigured command)
- **126**: Command in container can't be executed (permission issue)
- **127**: Command not found

**Step 3: Read logs from the previous instance**

```bash
kubectl logs <pod-name> --previous
# This shows logs from the LAST crashed container, which is what you need
```

If `--previous` returns nothing, the container exited before producing logs (very fast crash, usually a config issue at startup).

**Step 4: Categorize and dig deeper**

**Category A: Application crashes (exit 1, 139)**

Look at logs for stack traces, panics, fatal errors. Common:
- Missing environment variables or ConfigMap values
- Cannot connect to dependent service (database, cache)
- Configuration parse error
- Missing required file in image

**Category B: OOMKilled (exit 137)**

```bash
kubectl describe pod <pod-name> | grep -i OOMKilled
```

The pod exceeded its memory limit. Solutions covered in Q85 (debug OOMKilled).

**Category C: Liveness probe failure**

```bash
kubectl describe pod <pod-name> | grep -i "liveness probe failed"
```

The liveness probe failed `failureThreshold` consecutive times, so kubelet killed the container. Causes:
- Probe endpoint slow/broken under load
- Probe timeout too aggressive
- Application becomes briefly unresponsive (e.g., GC pause) and gets killed

**Category D: Init container failure**

If init containers fail, the main container never starts but the pod restarts.

```bash
kubectl describe pod <pod-name> | grep -A 5 "Init Containers"
kubectl logs <pod-name> -c <init-container-name>
```

**Category E: Permission issues**

```bash
kubectl logs <pod-name> --previous
# Common errors: "Permission denied", "Cannot create file"
```

Usually:
- runAsNonRoot but image expects root
- Volume mounted without correct fsGroup
- securityContext blocking required syscalls

**Step 5: Check node-level issues**

If multiple pods on the same node restart, check the node:

```bash
kubectl describe node <node-name>
# Conditions: MemoryPressure, DiskPressure?

# On the node:
journalctl -u kubelet --since "10 minutes ago"
dmesg -T | tail -100
```

**Step 6: Look at metrics**

Container restart rate in Prometheus:

```
rate(kube_pod_container_status_restarts_total[5m]) > 0
```

Memory usage approaching limit:

```
container_memory_working_set_bytes / container_spec_memory_limit_bytes > 0.9
```

**Sneaky causes I've seen in production:**

- A liveness probe that hits `/health`, which queries a database. Database hiccup → all pods crash simultaneously
- A pod's main process being a shell script that exits when its child process exits normally (exit 0, but pod restarts because Deployment expects continuous running)
- A sidecar container's main process is the actual workload, not the labeled "main" container, leading to confusing restart attribution

---

## 104. Cluster DNS suddenly stops working. How do you debug?

DNS failures cascade quickly: services can't reach each other, external API calls fail, even pulling images may break. Debugging DNS requires understanding the resolution path.

**Step 1: Confirm DNS is actually the problem**

From inside a pod:

```bash
kubectl run -it --rm debug --image=nicolaka/netshoot --restart=Never -- bash

# Inside pod:
nslookup kubernetes.default
# Expected: server 10.96.0.10, result with cluster IP

nslookup google.com
# Tests external resolution

# Check resolv.conf
cat /etc/resolv.conf
# nameserver 10.96.0.10
# search default.svc.cluster.local svc.cluster.local cluster.local
# options ndots:5
```

If both fail, DNS is broken. If only external fails, upstream forwarding is broken.

**Step 2: Check CoreDNS is running**

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
# Should show 2+ running pods (depending on cluster size)

kubectl get svc -n kube-system kube-dns
# CLUSTER-IP should match what pods see in /etc/resolv.conf
```

If CoreDNS pods aren't running:

```bash
kubectl describe pod -n kube-system <coredns-pod>
kubectl logs -n kube-system <coredns-pod>
```

Common CoreDNS issues:
- OOMKilled (memory limit too low for cache size)
- CrashLoopBackOff due to Corefile syntax error
- Node where CoreDNS runs is NotReady

**Step 3: Test DNS directly to CoreDNS pods**

Bypass the Service to isolate the issue:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
# Get pod IPs

# From a debug pod:
dig @<coredns-pod-ip> kubernetes.default.svc.cluster.local
```

If direct pod query works but Service IP doesn't, the issue is with the Service / kube-proxy.

**Step 4: Check kube-proxy**

```bash
kubectl get pods -n kube-system -l k8s-app=kube-proxy
kubectl logs -n kube-system <kube-proxy-pod>

# On a node where DNS fails:
iptables-save | grep KUBE-SVC | grep kube-dns
# Should show rules for the kube-dns service
```

If kube-proxy isn't programming iptables correctly, DNS Service IP won't route to CoreDNS pods.

**Step 5: Inspect CoreDNS Corefile**

```bash
kubectl get configmap -n kube-system coredns -o yaml
```

A typical Corefile:

```
.:53 {
    errors
    health
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
    }
    prometheus :9153
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
}
```

Common issues:
- The `forward` plugin points to a broken upstream
- Custom server block has syntax errors
- `health` plugin port conflict
- `loop` plugin detected a forwarding loop and CoreDNS won't start

**Step 6: Check upstream DNS**

CoreDNS forwards external queries to the node's `/etc/resolv.conf`:

```bash
# On a node:
cat /etc/resolv.conf
# Test the upstream:
dig @<upstream-dns> google.com
```

If the upstream is dead (e.g., cloud-provider VPC resolver having issues, or a custom forwarder DNS down), all external lookups fail.

**Step 7: Check NodeLocal DNSCache (if deployed)**

If you have NodeLocal DNSCache:

```bash
kubectl get pods -n kube-system -l k8s-app=node-local-dns
# Should be running on every node (DaemonSet)

kubectl logs -n kube-system <node-local-dns-pod>
```

When NodeLocal DNSCache fails, pods using `169.254.20.10` (the link-local IP) get errors immediately, not 5-second timeouts.

**Production scenarios I've seen:**

1. **CoreDNS pods scheduled on a failing node**: All replicas happened to land on the same NotReady node. Fix: anti-affinity rules ensuring distribution.

2. **Corefile change with typo applied via ConfigMap**: CoreDNS detected reload, failed to parse new config, kept running old config. Then a CoreDNS pod restarted (deployment update, eviction) and CrashLoopBackOff. Lesson: validate Corefile changes before applying.

3. **VPC DNS rate limiting**: Cloud provider DNS being rate-limited because CoreDNS cache TTL was set to 0. Lookups for external services worked initially then started failing as rate limit was hit.

4. **conntrack exhaustion**: Not strictly a DNS bug but manifests as DNS failures. UDP DNS queries dropped randomly.

5. **Loop detected**: CoreDNS forwarding to a DNS server that forwards back to CoreDNS (common in self-hosted clusters with misconfigured `kubelet --cluster-dns`). The `loop` plugin kills CoreDNS on startup.

---

## 105. Nodes become NotReady intermittently. What are possible reasons?

A node flapping between Ready and NotReady is one of the most disruptive failure modes: pods get evicted, rescheduled, and possibly re-evicted as the node recovers and fails again.

**Step 1: Understand what NotReady means**

The kubelet on each node posts a `NodeStatus` heartbeat every 10 seconds (Lease-based since 1.17). If the control plane doesn't see updates for `--node-monitor-grace-period` (default 40s), it marks the node `NotReady`. After 5 more minutes (`--pod-eviction-timeout`, deprecated but conceptually still applies via the TaintBasedEvictions controller), pods are evicted.

**Step 2: Check node conditions**

```bash
kubectl describe node <node-name>
# Conditions section:
#   Ready                False    "kubelet stopped posting node status"
#   MemoryPressure       False
#   DiskPressure         False
#   PIDPressure          False
#   NetworkUnavailable   False
```

The Ready condition's `Message` and `Reason` are crucial. Common ones:

- "kubelet stopped posting node status" → kubelet not heartbeating
- "container runtime is down" → containerd/CRI-O failure
- "PLEG is not healthy" → Pod Lifecycle Event Generator issue (often relates to runtime)
- "Kubelet has insufficient memory available" → MemoryPressure

**Step 3: Check kubelet health**

```bash
# On the node:
systemctl status kubelet
journalctl -u kubelet --since "1 hour ago" --no-pager | tail -200
```

Look for:
- `failed to update node lease` → kubelet can't reach API server
- `PLEG is not healthy: pleg was last seen active...` → container runtime slow
- OOM kills of kubelet itself (rare but devastating)
- Disk/inode exhaustion errors

**Step 4: Check container runtime**

```bash
# Containerd:
systemctl status containerd
journalctl -u containerd --since "1 hour ago" | tail -100
crictl ps   # Should respond quickly
```

If `crictl ps` hangs, the runtime is stuck. Common causes:
- Disk pressure preventing image layer reads
- A single problematic container exhausting runtime resources
- Storage driver issues (especially with overlayfs on certain filesystems)

**Step 5: Check network connectivity**

```bash
# From the node to API server:
curl -k https://<api-server>:6443/healthz
nc -zv <api-server> 6443
```

If the node can't reach the API server, kubelet can't post heartbeats. Causes:
- Security group / firewall rule change
- Cloud provider network blip
- DNS resolution failure for API server endpoint
- TLS certificate issues (especially expired client certs)

**Step 6: Check resource pressure**

```bash
# On the node:
free -h                  # Memory
df -h                    # Disk
df -i                    # Inodes
top -bn1 | head -20      # CPU load
uptime                   # Load average
```

A node thrashing (high load average, swapping if enabled, OOM events) often manifests as kubelet failing to heartbeat in time.

**Common intermittent NotReady causes:**

**Cause 1: Memory pressure causing kubelet OOM**

The kubelet itself is a process. If the node is so memory-constrained that kubelet gets killed (or paged out heavily), it can't heartbeat. Fix: increase `--system-reserved` and `--kube-reserved`.

**Cause 2: PLEG (Pod Lifecycle Event Generator) timeout**

Kubelet's PLEG relists all containers periodically. With many pods (200+) and a slow runtime, this can take longer than 3 minutes, triggering "PLEG is not healthy". Fixes: reduce pod density, upgrade runtime, ensure fast disk.

**Cause 3: Cloud provider API throttling**

Cloud-controller-manager calls cloud provider APIs to verify node status. AWS API rate limits, for instance, can cause nodes to appear NotReady. Particularly bad in large clusters or during mass scaling events.

**Cause 4: containerd / runc bugs**

Specific versions have had bugs where a hung container blocks the entire runtime. `crictl ps` hangs, kubelet PLEG fails, node goes NotReady. Often fixed by killing the stuck shim process.

**Cause 5: Disk I/O saturation**

A noisy neighbor pod (or a runaway log writer) saturates the node's disk. Kubelet's writes to etcd-backed status updates time out. Common on EBS-backed instances with bursting credits exhausted.

**Cause 6: kubelet GC sweeps blocking**

When images or containers need GC and the operation takes long (slow disk, many layers), kubelet can become unresponsive. Tune `--image-gc-high-threshold` lower so GC happens incrementally rather than under pressure.

**Cause 7: Certificate rotation issues**

If kubelet client certs expire and rotation fails, kubelet loses API server access. Watch for cert rotation errors in journald.

**Mitigation patterns:**

- Set up alerting on `kube_node_status_condition{condition="Ready",status="false"}` immediately
- Monitor PLEG duration: `kubelet_pleg_relist_duration_seconds`
- Ensure cluster autoscaler can remove problematic nodes (sometimes cordon+drain+terminate is the fix)
- Use node problem detector (`node-problem-detector`) to catch hardware/kernel issues

---

## 106. Applications face intermittent 502 errors behind Ingress

502 Bad Gateway from an Ingress means the Ingress controller (nginx, HAProxy, Traefik, etc.) couldn't get a valid response from the upstream pod. "Intermittent" is the key word: connectivity exists, but some requests fail.

**Step 1: Identify which component returns 502**

The 502 could come from:
- Cloud load balancer in front of the Ingress controller
- Ingress controller itself (most common)
- Service mesh sidecar (Istio, Linkerd)

Check the response headers:

```bash
curl -v https://your-app.example.com
# Server: nginx → 502 from nginx ingress
# Via: ... ELB ... → could be from AWS ELB
```

**Step 2: Examine Ingress controller logs**

For nginx-ingress:

```bash
kubectl logs -n ingress-nginx <controller-pod> | grep 502
# Look for:
# "upstream prematurely closed connection while reading response header"
# "connect() failed (111: Connection refused)"
# "upstream timed out"
# "no live upstreams"
```

These specific messages tell you the failure mode.

**Step 3: Match failure modes to root causes**

**Cause 1: Pod terminated mid-request ("upstream prematurely closed connection")**

The most common cause. Sequence of events:

1. A pod receives `SIGTERM` (rolling update, eviction, HPA scale-down)
2. Pod is removed from Service Endpoints, but iptables rules update isn't instantaneous
3. New connections still hit this pod
4. Pod's main process exits, closing connections mid-response
5. Ingress controller sees a half-closed connection, returns 502

**Fix:**
- Implement proper `preStop` hooks: `sleep 10` to allow endpoint propagation
- Ensure application handles SIGTERM gracefully (drain in-flight requests, then exit)
- Set `terminationGracePeriodSeconds` longer than the longest expected request
- Configure Ingress controller upstream retry on idempotent methods

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 15"]
terminationGracePeriodSeconds: 60
```

**Cause 2: Application timeout shorter than Ingress timeout**

If the app closes connection after 30s but Ingress waits 60s, brief stalls cause 502s. Configure consistent timeouts.

**Cause 3: Backend connection limits exhausted**

If the app has too few worker threads/processes, requests queue and time out:

```
upstream timed out (110: Connection timed out)
```

Scale the app, increase worker count, or add connection pooling.

**Cause 4: HTTP/1.1 keepalive timing mismatch**

Ingress reuses upstream connections via keepalive. If the backend's keepalive timeout is shorter than the Ingress's, the backend closes a connection just as the Ingress tries to reuse it:

```
upstream prematurely closed connection
```

Backend keepalive should be longer than Ingress keepalive. For Go apps, set `IdleTimeout` > Ingress's `keepalive-timeout`.

**Cause 5: Application crashes/restarts during rollouts**

Rolling update kills pods faster than readiness probes can keep up. Briefly, the service has fewer (or zero) ready endpoints.

Fix: tune `maxUnavailable` lower, ensure readiness probes are accurate.

**Cause 6: Service endpoint stale**

EndpointSlices/Endpoints lag behind reality. A pod's IP is in the endpoints list but the pod is gone. Symptom: "Connection refused".

This is fundamental to Kubernetes networking — there's always some propagation delay. Reduce its impact with retries and preStop hooks.

**Step 4: Look at application logs**

```bash
kubectl logs <app-pod> --tail=100
# Are there crashes? Slow requests? GC pauses?
```

**Step 5: Check pod resource state**

```bash
kubectl top pod -l app=<app>
kubectl describe pod <app-pod> | grep -A 5 -i memory
```

Pods under memory pressure or being CPU-throttled can briefly become unresponsive, causing upstream timeouts.

**Step 6: Distributed tracing**

If you have tracing (Jaeger, Tempo, Datadog APM), look for traces ending in errors. The trace tells you exactly which span failed and how long it took.

**Production patterns to prevent 502s:**

- **Always set preStop sleep** for HTTP services
- **Readiness probes must be strict**: a pod that doesn't pass readiness shouldn't get traffic
- **Use PodDisruptionBudgets** to prevent simultaneous termination of too many pods
- **Configure ingress retries** for safe methods (GET, HEAD)
- **Set generous terminationGracePeriod** for long-request services
- **Monitor**: `nginx_ingress_controller_requests{status=~"5.."}` rate

---

## 107. A StatefulSet Pod cannot attach storage after rescheduling

StatefulSet pods have stable identity tied to persistent storage. When a pod moves to a new node, the volume must reattach. Failures here often manifest as the pod stuck in `ContainerCreating` with mount errors.

**Step 1: Get the pod's state**

```bash
kubectl get pod <statefulset-pod>
# NAME      READY   STATUS              RESTARTS   AGE
# mysql-0   0/1     ContainerCreating   0          5m

kubectl describe pod <statefulset-pod>
# Events:
#   Warning  FailedMount       2m    kubelet  MountVolume.MountDevice failed for volume "pvc-abc"
#                                             : rpc error: code = Internal desc = ...
```

**Step 2: Categorize the volume failure**

The error message usually points to one of these:

**Category A: Volume still attached to old node**

```
Multi-Attach error for volume "pvc-abc"
Volume is already exclusively attached to one node and can't be attached to another
```

This is the classic problem. The volume is `ReadWriteOnce` (RWO), can only attach to one node, and the old node hasn't released it yet.

**Why it happens:**
- Old node became NotReady (network issue), kubelet didn't gracefully detach
- The volume controller waits for the old node to confirm detach before allowing reattach
- If the old node is permanently dead, manual intervention is needed

**Resolution:**

```bash
# Check VolumeAttachment objects:
kubectl get volumeattachment | grep <pv-name>

# If old node is dead and confirmed not coming back:
kubectl delete volumeattachment <attachment-name>

# Or force-delete the pod on the old node:
kubectl delete pod <old-pod> --grace-period=0 --force
# (Use with extreme caution: data corruption risk if old pod is actually still writing)
```

**Category B: CSI driver issues**

```
MountVolume.MountDevice failed for volume "pvc-xyz" :
rpc error: code = DeadlineExceeded
```

The CSI driver (storage plugin) is timing out or erroring. Check:

```bash
# Find the CSI driver pods:
kubectl get pods -n kube-system -l app=ebs-csi-controller   # AWS example
kubectl get pods -n kube-system -l app=ebs-csi-node

kubectl logs -n kube-system <csi-controller-pod> -c csi-attacher
kubectl logs -n kube-system <csi-controller-pod> -c csi-provisioner
kubectl logs -n kube-system <csi-node-pod-on-target-node>
```

Common CSI issues:
- Cloud provider API throttling (too many attach/detach operations)
- IAM permissions changed; CSI can't call cloud API
- CSI driver pod crashed on target node

**Category C: Topology mismatch**

```
0/10 nodes are available: 10 node(s) had volume node affinity conflict
```

The PV is in availability zone A, but no schedulable nodes exist in zone A. Pod can't be placed where the volume is.

This commonly happens with:
- `volumeBindingMode: Immediate` (legacy default) creating PVs in one zone before the pod was scheduled
- A zonal outage making the volume's AZ unschedulable

**Resolution:**

For new workloads, always use `volumeBindingMode: WaitForFirstConsumer`:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
volumeBindingMode: WaitForFirstConsumer
```

This delays volume creation until the pod is scheduled, ensuring the volume is provisioned in a zone that has capacity.

For existing stuck pods: scale the cluster to have capacity in the volume's zone, or migrate data to a new PV.

**Category D: Filesystem corruption**

```
mount: /var/lib/kubelet/.../mount: wrong fs type, bad option, bad superblock
```

The filesystem on the volume is corrupted. Causes:
- Previous unclean shutdown
- Underlying storage issue

Resolution requires fsck on the volume, ideally by attaching it to a debug pod. This is dangerous and may require restoring from backup.

**Category E: Wrong filesystem permissions / fsGroup issues**

The volume mounts but the application can't write to it:

```
permission denied
```

Especially common after migrating to non-root containers. Fix:

```yaml
spec:
  securityContext:
    fsGroup: 1000
```

This causes Kubernetes to chown the volume contents to GID 1000 on mount.

**Step 3: Check the underlying PV/PVC**

```bash
kubectl get pvc -n <namespace>
kubectl describe pvc <pvc-name>

kubectl get pv
kubectl describe pv <pv-name>
# Look at: Status (Bound/Released/Available), NodeAffinity
```

**Step 4: Check the cloud volume directly (if applicable)**

For AWS EBS:

```bash
aws ec2 describe-volumes --volume-ids vol-xxx
# Look at: State (in-use/available), Attachments
```

The cloud's view of attachment may differ from Kubernetes's view. Reconciling them sometimes requires manual action.

**Production patterns for StatefulSet reliability:**

- Use `WaitForFirstConsumer` storage classes always
- Set pod anti-affinity across zones for HA across AZ failures
- Use `podManagementPolicy: Parallel` only if your workload tolerates it; otherwise `OrderedReady` (default)
- Monitor `kubelet_volume_stats_*` metrics for capacity and health
- For databases, prefer operators (Patroni, MySQL Operator) that handle failover including volume management
- Have a documented runbook for stuck VolumeAttachment scenarios

---

## 108. API Server latency suddenly increases. How do you analyze?

API server latency affects everything: scheduling, controller reconciliation, kubectl responsiveness. Sudden spikes are usually traceable to either etcd issues, request floods, or webhooks.

**Step 1: Confirm the symptom**

```bash
# From a client:
time kubectl get pods --all-namespaces
# If this takes 30+ seconds vs. normal sub-second, latency is real

# Check API server metrics:
curl -sk https://<api-server>:6443/metrics --header "Authorization: Bearer <token>" | grep apiserver_request_duration
```

Key metrics:
- `apiserver_request_duration_seconds`: Histogram of request durations by verb, resource, code
- `apiserver_request_total`: Request counts (helps identify a flood)
- `apiserver_current_inflight_requests`: Currently-running requests
- `etcd_request_duration_seconds`: etcd latency (often the root cause)

**Step 2: Identify the slow request types**

```promql
histogram_quantile(0.99,
  rate(apiserver_request_duration_seconds_bucket[5m])
) by (verb, resource)
```

This tells you which operations are slow. Patterns:

- **All requests slow** → etcd issue, or API server resource starvation
- **Only writes slow** → etcd write performance, admission webhooks
- **Only LIST slow** → unbounded list queries, watch cache rebuild
- **Specific resource slow** → admission webhook on that resource, or that resource type has many objects

**Step 3: Check etcd**

etcd is the most common culprit for API server latency:

```bash
# Get etcd member info:
kubectl exec -n kube-system etcd-<master> -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status --write-out=table
```

Metrics to check:
- `etcd_disk_wal_fsync_duration_seconds` (P99 should be <10ms; >100ms is bad)
- `etcd_disk_backend_commit_duration_seconds`
- `etcd_server_leader_changes_seen_total` (leader elections are bad)
- `etcd_server_has_leader` (should be 1)

**Common etcd issues:**

- **Slow disk**: EBS gp2 throttled, network-attached storage saturated. Solution: faster disks (NVMe, gp3 with provisioned IOPS, local SSDs)
- **Database too large**: etcd performs poorly above ~8GB. Defragment regularly: `etcdctl defrag`
- **Quota exceeded**: If etcd hits its space quota (default 2GB, configurable to 8GB), all writes fail. Alarm: `etcdserver: mvcc: database space exceeded`
- **Compaction lag**: Old revisions not compacted. Configure auto-compaction

**Step 4: Check for request floods**

```promql
sum(rate(apiserver_request_total[1m])) by (verb, resource, subresource)
```

Sudden spike in:
- LISTs on Pods/Events → poorly written controller or operator listing entire cluster repeatedly
- WATCHes opened → reconnecting clients (often after API server restart)
- CREATE/UPDATE on events → noisy operators generating events

A single misbehaving controller doing `kubectl get pods --all-namespaces` in a loop can saturate the API server. Find it:

```promql
sum(rate(apiserver_request_total[5m])) by (user_agent)
```

Look for unfamiliar user-agents with high rates.

**Step 5: Check admission webhooks**

Slow webhooks add latency to every relevant request:

```promql
histogram_quantile(0.99,
  rate(apiserver_admission_webhook_admission_duration_seconds_bucket[5m])
) by (name)
```

A webhook taking 5s adds 5s to every mutating/validating request that triggers it.

**Common webhook issues:**
- Webhook pod down or scaled to zero → API server waits for timeout (default 10s), then fails open or closed
- Webhook hits external services that are slow
- Webhook running on the only master node and competing for resources

Find and fix:

```bash
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations

# For each, check the targeted service
kubectl get svc -n <webhook-namespace>
kubectl get pods -n <webhook-namespace>
```

If a critical webhook's failurePolicy is `Fail`, all matching requests will block. If `Ignore`, requests succeed but webhook isn't applied (security implications).

**Step 6: Check API server resources**

```bash
kubectl top pods -n kube-system | grep apiserver
kubectl describe pod -n kube-system <kube-apiserver-pod>
```

API server CPU pegged at limit? Memory at limit? It needs more resources, or there's a memory leak (rare but happens with old versions).

**Step 7: Check API server logs**

```bash
kubectl logs -n kube-system <kube-apiserver-pod> | grep -E "slow|timeout|error"
```

Look for:
- "Slow request" warnings (logged for requests >1s)
- "etcdserver: request timed out"
- "OpenAPI spec did not validate"

**Production scenarios I've seen:**

1. **A new operator listing all pods every 10 seconds**: With 50,000 pods, this overwhelmed the API server. Fix: use informers (shared, watch-based) instead of polling LIST.

2. **etcd defrag never run**: After 6 months, etcd was 7GB, mostly fragmented. Defragmenting brought it to 800MB and latency dropped 10x.

3. **A validating webhook calling an external SaaS for policy checks**: SaaS had a brief outage. Every API request timed out for 10s. Fix: cache decisions, set tight webhook timeouts, careful failurePolicy choice.

4. **Stuck watch connections**: A buggy controller opened watches but never read them. API server buffered events until OOM. Fix in client-go versions, also rate-limit per-client watches at the API server.

5. **Audit log writing to slow disk**: API server audit logs were going to a slow disk. Writing audit events synchronously blocked the request. Fix: async audit, or fast audit disk.

---

## 109. etcd disk usage is full. What happens?

When etcd's disk fills up or it hits its space quota, the cluster doesn't just degrade — it becomes effectively read-only and many operations fail catastrophically.

**What "full" means in etcd context:**

etcd has two notions of "full":

1. **Disk full**: The physical filesystem etcd writes to is full. WAL writes fail. etcd crashes or refuses to start.

2. **Space quota exceeded**: etcd has an internal quota (default 2GB, often configured to 8GB). When the database file (`member/snap/db`) exceeds this, etcd enters maintenance mode.

**What happens when space quota is exceeded:**

```
etcdserver: mvcc: database space exceeded
```

- **All write requests fail** with `NOSPACE` error
- **Reads continue working**
- **No new pods can be created, scheduled, or updated**
- **Controllers can't update statuses**
- **kubectl get works, kubectl apply fails**

The cluster essentially freezes in its current state. Existing pods keep running (kubelet doesn't need etcd writes), but anything requiring control plane action stops.

**What happens when actual disk is full:**

Worse: etcd can't write to its WAL (Write-Ahead Log). This causes:

- **etcd member crashes**
- **Loss of quorum if multiple members fill up simultaneously**
- **API server can't connect to etcd, returns 500 errors**
- **Full control plane outage**

**Why etcd disk fills:**

1. **Object growth**: Lots of Pods, ConfigMaps, Secrets, Events accumulating over time
2. **No compaction**: etcd keeps every revision (every change) of every object by default. With many updates, history grows unbounded
3. **No defragmentation**: After compaction, freed space remains as fragmentation; the file doesn't shrink
4. **Excessive events**: Events are stored in etcd. Cluster generating thousands per minute fills etcd fast
5. **Large objects**: A ConfigMap or Secret approaching the 1.5MB limit, multiplied across many copies
6. **Audit logs not rotated**: Sometimes confused with etcd issues but separate

**Detection:**

Monitor:

```promql
etcd_mvcc_db_total_size_in_bytes
etcd_mvcc_db_total_size_in_use_in_bytes
etcd_server_quota_backend_bytes
```

The ratio of `in_use` to `total_size` shows fragmentation. If `total_size` approaches quota, you're in trouble.

Alert at 70% of quota.

**Recovery from "database space exceeded":**

**Step 1: Compact old revisions**

```bash
# Get current revision:
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=... --cert=... --key=... \
  endpoint status --write-out=json | jq '.[].Status.header.revision'

# Compact up to (current - N):
ETCDCTL_API=3 etcdctl --endpoints=... compact <revision>
```

This removes old revisions from etcd's history.

**Step 2: Defragment**

Compaction frees space logically; defrag reclaims it physically:

```bash
# Per-member; do one at a time to maintain quorum:
ETCDCTL_API=3 etcdctl --endpoints=https://etcd-1:2379 defrag
ETCDCTL_API=3 etcdctl --endpoints=https://etcd-2:2379 defrag
ETCDCTL_API=3 etcdctl --endpoints=https://etcd-3:2379 defrag
```

Defrag is blocking; the member is unavailable during it. Do one member at a time.

**Step 3: Clear the alarm**

After defrag:

```bash
ETCDCTL_API=3 etcdctl --endpoints=... alarm list
# Shows: alarm:NOSPACE

ETCDCTL_API=3 etcdctl --endpoints=... alarm disarm
```

Writes can resume.

**Prevention:**

**1. Configure auto-compaction**

```
--auto-compaction-mode=periodic
--auto-compaction-retention=8h
```

This automatically compacts revisions older than 8 hours. Set conservatively for your change rate.

**2. Schedule regular defragmentation**

A CronJob or external scheduler that defrags each etcd member weekly during low-traffic windows. Tools like `etcd-defrag` automate this safely.

**3. Increase quota if needed**

```
--quota-backend-bytes=8589934592   # 8GB
```

8GB is generally the practical max; beyond this, etcd performance suffers.

**4. Limit event retention**

Reduce kube-apiserver flag:

```
--event-ttl=1h
```

Events older than 1h are removed from etcd, drastically reducing growth.

**5. Set up disk and quota alerts**

Alert at:
- 70% of `--quota-backend-bytes`
- 80% of actual disk
- WAL fsync P99 > 100ms (predicts performance degradation)

**6. Separate etcd from other workloads**

etcd should run on dedicated nodes with dedicated, fast disks. Sharing disks with kubelet or container images is a common production mistake.

**Disaster recovery:**

If etcd is completely corrupted:

1. Stop all etcd members
2. Restore from snapshot:

```bash
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --name etcd-1 \
  --initial-cluster etcd-1=https://10.0.1.1:2380,etcd-2=https://10.0.1.2:2380,etcd-3=https://10.0.1.3:2380 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-advertise-peer-urls https://10.0.1.1:2380 \
  --data-dir /var/lib/etcd-restored
```

Repeat for each member with appropriate values. Start the cluster.

**Lesson:** Regular etcd snapshots are non-negotiable. Schedule them every 30 minutes, retain 24+ hours.

---

## 110. Worker node CPU usage reaches 100%. How do you handle?

A node at 100% CPU isn't immediately catastrophic (unlike memory exhaustion), but it causes latency spikes, missed liveness probes triggering kills, and kubelet itself becoming slow. The response depends on whether it's a single pod or systemic.

**Step 1: Confirm and identify**

```bash
kubectl top nodes
# NAME      CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# node-1    7800m        97%    14Gi            60%

# Which pods on that node?
kubectl top pods --all-namespaces --field-selector spec.nodeName=node-1 --sort-by=cpu
```

This usually reveals the culprit pod immediately.

**Step 2: Investigate the top consumer**

If a single pod is consuming most of the CPU:

```bash
kubectl describe pod <culprit-pod>
# Check: limits, requests, restart count

kubectl logs <culprit-pod> --tail=200
# Looking for: infinite loops, GC thrashing, retry storms
```

**Common single-pod causes:**

- **Application bug**: infinite loop, runaway recursion, busy-wait
- **Garbage collection storm**: JVM/Go runtime in continuous GC because heap is too small or there's a memory leak
- **Retry storm**: app retrying a failed downstream call without backoff
- **Tight polling**: app polling something every millisecond instead of every second
- **Crypto/compression**: legitimate but expensive operations under load

**Step 3: Profile the application (if possible)**

For Go apps with pprof enabled:

```bash
kubectl port-forward <pod> 6060:6060
go tool pprof http://localhost:6060/debug/pprof/profile
```

For Java apps:

```bash
kubectl exec <pod> -- jstack 1
```

Look for hot stack traces.

**Step 4: Immediate mitigation**

If the pod has no CPU limits, it's consuming everything:

**Option A: Add/lower CPU limits**

```bash
kubectl set resources deployment/<name> --limits=cpu=2,memory=2Gi
```

Caveat: This causes throttling (Q86) but stops the noisy neighbor problem.

**Option B: Delete the pod**

```bash
kubectl delete pod <culprit-pod>
```

If it's a Deployment, a new pod starts. If the issue persists, it's not a transient bug.

**Option C: Scale down or pause**

```bash
kubectl scale deployment/<name> --replicas=0
```

Buys time to investigate without ongoing impact.

**Step 5: Examine system-level pressure**

If no single pod dominates, the load is distributed:

```bash
# On the node:
top -bn1 | head -20
uptime
cat /proc/loadavg
```

Load average way above CPU count (e.g., 30+ on a 8-core node) means severe contention.

```bash
# What's competing?
ps aux --sort=-%cpu | head -20
```

Look for non-container processes consuming CPU: kubelet, container runtime, kernel processes.

**Step 6: Check kubelet and runtime health**

Under heavy CPU pressure, kubelet itself slows down, which has cascading effects:

```bash
journalctl -u kubelet --since "5 minutes ago" | grep -E "slow|timeout"
```

Look for:
- "PLEG is not healthy"
- "Sync ... took longer than expected"
- Liveness probe timeouts

If kubelet is starving, it can't kill misbehaving pods or report status, making everything worse.

**Step 7: Node-level mitigation**

If the node is unrecoverable in real-time:

**Step A: Cordon the node**

```bash
kubectl cordon <node-name>
```

Prevents new pods from being scheduled there.

**Step B: Drain (gradually)**

```bash
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data --grace-period=60
```

Evicts pods (respecting PodDisruptionBudgets). They'll be rescheduled to other nodes. The CPU-hungry pod might land on another node and cause the same issue there — fix it first.

**Step C: Reboot or replace the node**

For a stuck node, replacement is often faster than recovery:

```bash
# Cloud-managed:
# Terminate the instance; autoscaling group creates a replacement

# Bare metal: 
# Reboot via IPMI
```

**Step 8: Long-term fixes**

**Pattern 1: Always set CPU requests**

Without requests, the scheduler doesn't know how much CPU your pod really needs. It packs pods onto nodes without accounting for actual usage, leading to contention. Set requests based on observed P95 usage.

**Pattern 2: Use VPA in recommendation mode**

Vertical Pod Autoscaler can suggest right-sized requests/limits based on observed usage. Even if you don't enable auto-update, the recommendations are valuable.

**Pattern 3: HPA with appropriate metrics**

CPU-bound services should have HPA scaling on CPU. Set target utilization around 60-70% to leave headroom.

**Pattern 4: Pod anti-affinity for CPU-intensive workloads**

Spread CPU-heavy pods across nodes:

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values: ["cpu-heavy-app"]
          topologyKey: kubernetes.io/hostname
```

**Pattern 5: Reserved capacity for system processes**

```
--system-reserved=cpu=500m
--kube-reserved=cpu=500m
```

Ensures kubelet and system daemons aren't starved even when pods consume everything available.

**Pattern 6: Priority classes**

System-critical pods (CoreDNS, kube-proxy, CNI) should have high priority:

```yaml
priorityClassName: system-node-critical
```

Under pressure, low-priority pods are preempted first.

**Pattern 7: Monitor and alert before 100%**

Alert at 80% sustained CPU:

```promql
avg_over_time(
  (sum(rate(container_cpu_usage_seconds_total{}[5m])) by (node)
  / on(node) sum(kube_node_status_allocatable{resource="cpu"}) by (node)
  )[15m:]
) > 0.8
```

You want to scale or investigate before it becomes critical.

**Production scenarios I've seen:**

1. **GC death spiral**: Java app with `-Xmx` too small for the workload. GC ran continuously, consuming all CPU. Memory wasn't full, just garbage-heavy. Fix: increase heap, tune GC.

2. **Retry storm cascade**: Service A's downstream became slow. Service A retried aggressively, consuming CPU. Other pods on the same node got starved, became slow themselves, started retrying their downstreams. Whole node cascaded to 100%. Fix: circuit breakers, exponential backoff with jitter.

3. **A cronjob that overlapped itself**: A CronJob with `concurrencyPolicy: Allow` (default) and a long-running job. Each minute, new instances started while previous ones still ran. After an hour, 60 instances on one node. Fix: `concurrencyPolicy: Forbid`.

4. **Crypto miner**: A compromised pod running a crypto miner. Took down nodes one by one. Fix: NetworkPolicies, image scanning, runtime security (Falco).

5. **Logging library spinning**: Application's log shipper had a bug causing infinite CPU on log file rotation. Fix: upgrade the library, monitor sidecar CPU separately.

