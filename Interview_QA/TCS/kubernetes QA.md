As of September 9, 2026, **Kubernetes 1.37.0 is the latest stable minor release**, released August 26, 2026. The Kubernetes project currently maintains the 1.37, 1.36, and 1.35 branches, so the guide below is aligned with modern Kubernetes while keeping the core answers applicable across recent versions. ([Kubernetes][1])

# Top 100 Kubernetes Interview Questions and Answers

## Senior DevOps Engineer — Architecture, Deployment, Networking, Security, Performance, Observability & Troubleshooting

---

# Section 1 — Kubernetes Architecture and Core Concepts

## 1. What is Kubernetes, and why do organizations use it?

### Answer

Kubernetes is an open-source container orchestration platform used to deploy, manage, scale, and recover containerized applications.

It provides capabilities including:

```text
Container scheduling
Service discovery
Load balancing
Self-healing
Horizontal scaling
Rolling deployments
Secret/configuration management
Persistent storage orchestration
Resource management
```

Without Kubernetes:

```text
Application
   ↓
Manually choose server
   ↓
Start container
   ↓
Configure networking
   ↓
Monitor
   ↓
Restart if it fails
```

With Kubernetes:

```text
Desired State
     ↓
Kubernetes Control Plane
     ↓
Schedule workloads
     ↓
Monitor state
     ↓
Reconcile actual state
     ↓
Maintain desired state
```

For example:

```yaml
replicas: 5
```

means Kubernetes continuously tries to maintain five replicas.

If one Pod disappears:

```text
Desired = 5
Actual = 4
     ↓
Controller detects difference
     ↓
Creates replacement
```

### Senior-level point

Kubernetes is fundamentally a **distributed reconciliation system**, not merely a container launcher.

---

## 2. Explain Kubernetes architecture.

### Answer

A Kubernetes cluster normally consists of:

```text
                Control Plane
                ─────────────
              kube-apiserver
                    |
        +-----------+-----------+
        |           |           |
       etcd     Scheduler   Controller
                             Manager

                    |
               Worker Nodes
        +-----------+-----------+
        |                       |
      Node A                   Node B
        |                       |
     kubelet                 kubelet
     runtime                 runtime
        |                       |
      Pods                    Pods
```

### Control plane components

```text
kube-apiserver
etcd
kube-scheduler
kube-controller-manager
cloud-controller-manager
```

### Worker-node components

```text
kubelet
container runtime
networking/CNI
kube-proxy or alternative service networking implementation
```

The API server acts as the central interface through which Kubernetes components interact.

---

## 3. What is kube-apiserver?

### Answer

`kube-apiserver` exposes the Kubernetes API.

Almost every cluster operation passes through it.

For example:

```bash
kubectl get pods
```

flow:

```text
kubectl
   ↓ HTTPS
kube-apiserver
   ↓
Authentication
   ↓
Authorization
   ↓
Admission
   ↓
Read data
   ↓
Response
```

For writes:

```text
kubectl apply
     ↓
API server
     ↓
Authentication
     ↓
Authorization
     ↓
Admission controllers
     ↓
Validation
     ↓
Persist to etcd
```

The API server is designed to be horizontally scalable.

In HA clusters:

```text
           Load Balancer
                |
      +---------+---------+
      |         |         |
   API #1    API #2    API #3
```

### Senior troubleshooting

High API latency may come from:

```text
etcd latency
admission webhooks
API server CPU
large LIST operations
network latency
excessive controllers
client request storms
```

---

## 4. What is etcd and why is it critical?

### Answer

etcd is a distributed key-value database used to store Kubernetes cluster state.

It contains information such as:

```text
Pods
Deployments
Services
Secrets
ConfigMaps
RBAC
Nodes
Custom resources
```

Conceptually:

```text
Kubernetes API objects
        ↓
   kube-apiserver
        ↓
       etcd
```

If etcd is permanently lost without backup, the Kubernetes control-plane state is effectively lost.

### Important characteristics

etcd uses the **Raft consensus algorithm**.

For HA, an odd number of members is common:

```text
3 members → tolerate 1 failure
5 members → tolerate 2 failures
```

### Senior principle

etcd performance depends strongly on:

```text
disk fsync latency
network latency
CPU
database size
```

Putting etcd on slow or highly variable storage can cause control-plane instability.

---

## 5. What does the kube-scheduler do?

### Answer

The scheduler decides which node should run an unscheduled Pod.

A new Pod initially looks conceptually like:

```text
Pod
nodeName = empty
```

The scheduler evaluates nodes.

Typical factors include:

```text
CPU requests
memory requests
node selectors
node affinity
pod affinity
pod anti-affinity
taints/tolerations
topology constraints
persistent-volume constraints
resource availability
priority
```

Conceptually:

```text
Pending Pod
    ↓
Filter nodes
    ↓
Node A ✓
Node B ✗ insufficient memory
Node C ✗ taint not tolerated
    ↓
Score remaining nodes
    ↓
Select Node A
```

The scheduler does not normally start the container.

It chooses the node; the node's kubelet handles execution.

---

## 6. What is kube-controller-manager?

### Answer

The controller manager runs controllers that continuously reconcile Kubernetes objects.

Examples include:

```text
Deployment/ReplicaSet-related controllers
Node controller
Job controller
EndpointSlice controller
ServiceAccount controller
Namespace controller
```

A controller follows a pattern:

```text
Observe desired state
        ↓
Observe actual state
        ↓
Compare
        ↓
Take corrective action
```

For example:

```text
Deployment replicas = 3

Current Pods = 2
      ↓
Controller creates another Pod
```

This reconciliation loop is one of Kubernetes' most fundamental concepts.

---

## 7. What does kubelet do?

### Answer

The kubelet is the primary Kubernetes agent running on each node.

Responsibilities include:

```text
Register node
Watch assigned Pods
Interact with container runtime
Mount volumes
Run probes
Report Pod/node status
Manage container lifecycle
```

Architecture:

```text
API Server
    |
    ↓
 kubelet
    |
    ↓ CRI
Container Runtime
    |
    ↓
 Containers
```

Useful troubleshooting commands include:

```bash
systemctl status kubelet
journalctl -u kubelet
```

If kubelet stops functioning:

```text
existing containers may continue running

but

Pod lifecycle/status management becomes unhealthy
```

Eventually the node may be marked unavailable by the control plane.

---

## 8. What is kube-proxy?

### Answer

kube-proxy traditionally implements Kubernetes Service networking on nodes.

It watches objects such as:

```text
Services
EndpointSlices
```

and configures networking rules so a Service virtual IP can direct traffic to backend Pods.

Conceptually:

```text
Client
  ↓
Service ClusterIP
  ↓
Service networking rules
  ↓
Pod A
Pod B
Pod C
```

Implementations can use Linux networking mechanisms.

Modern clusters may also use CNI/eBPF implementations that replace or reduce reliance on traditional kube-proxy behavior.

### Senior interview point

A Service is not usually a proxy process listening on its ClusterIP.

Its behavior is normally implemented through node networking rules or data-plane programming.

---

## 9. Explain CRI, CNI, and CSI.

### Answer

These three interfaces allow Kubernetes to integrate external runtime, network, and storage implementations.

### CRI — Container Runtime Interface

Used between:

```text
kubelet
   ↓
container runtime
```

Examples:

```text
containerd
CRI-O
```

---

### CNI — Container Network Interface

Responsible for Pod networking.

Examples include:

```text
Cilium
Calico
Flannel
Antrea
cloud-provider CNI plugins
```

---

### CSI — Container Storage Interface

Allows storage systems to integrate with Kubernetes.

Examples:

```text
AWS EBS
Azure Disk
Google Persistent Disk
Ceph
NetApp
VMware storage
```

Memorize:

```text
CRI → Runtime
CNI → Network
CSI → Storage
```

---

## 10. What does "desired state" mean in Kubernetes?

### Answer

Kubernetes is declarative.

You normally specify what you want, rather than every command needed to achieve it.

Example:

```yaml
spec:
  replicas: 3
```

You declare:

```text
I want three Pods.
```

Controllers determine how to achieve and maintain that state.

If:

```text
Desired = 3
Actual = 2
```

Kubernetes creates one.

If:

```text
Desired = 3
Actual = 4
```

Kubernetes removes one.

This is the Kubernetes reconciliation model.

### Senior point

This is why manually modifying objects controlled by other controllers can produce surprising results: reconciliation may overwrite or reverse the manual change.

---

# Section 2 — Pods and Workload Controllers

## 11. What is a Pod?

### Answer

A Pod is Kubernetes' smallest schedulable workload unit.

A Pod may contain one or more containers.

Containers inside one Pod share certain resources, including:

```text
network namespace
Pod IP
localhost networking
mounted volumes
lifecycle
```

Example:

```text
Pod
├── Application container
└── Sidecar container
```

Both containers can communicate through:

```text
localhost
```

A Pod should generally be considered ephemeral.

Do not design around a Pod retaining:

```text
same IP
same node
same filesystem
```

forever.

---

## 12. What is a Deployment?

### Answer

A Deployment manages stateless workloads using ReplicaSets.

Example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
```

Architecture:

```text
Deployment
    ↓
ReplicaSet
    ↓
+------+------+
|      |      |
Pod   Pod    Pod
```

Deployments provide features such as:

```text
rolling updates
rollback
replica management
declarative application updates
```

Useful commands:

```bash
kubectl get deployment
kubectl rollout status deployment/api
kubectl rollout history deployment/api
kubectl rollout undo deployment/api
```

---

## 13. What is a ReplicaSet?

### Answer

A ReplicaSet ensures that a specified number of Pod replicas exist.

Example:

```text
ReplicaSet desired = 4

Current = 3
     ↓
Create 1 Pod
```

You normally do not manage ReplicaSets directly for application deployments.

Instead:

```text
Deployment
   ↓
ReplicaSet
   ↓
Pods
```

During a rolling update:

```text
Deployment
├── Old ReplicaSet
└── New ReplicaSet
```

The Deployment gradually scales one down and the other up.

---

## 14. What is a StatefulSet, and when would you use one?

### Answer

StatefulSet manages workloads requiring stable identity or storage relationships.

Typical examples:

```text
databases
Kafka
ZooKeeper-like systems
distributed storage
stateful clustered systems
```

StatefulSet Pods receive predictable identities:

```text
db-0
db-1
db-2
```

Unlike Deployment Pods:

```text
api-6b4887c8-x12aa
api-6b4887c8-q91jk
```

StatefulSets can provide:

```text
stable Pod naming
ordered creation/deletion
stable volume association
predictable DNS identities
```

### Senior point

StatefulSet does **not** automatically make an application highly available.

The application must still understand:

```text
replication
leader election
quorum
data consistency
failover
```

---

## 15. What is a DaemonSet?

### Answer

DaemonSet ensures a Pod runs on applicable nodes.

Typical uses:

```text
logging agents
monitoring agents
CNI components
security agents
storage agents
```

Architecture:

```text
Node 1 → logging-agent Pod
Node 2 → logging-agent Pod
Node 3 → logging-agent Pod
```

When another node joins:

```text
Node 4 joins
   ↓
DaemonSet controller
   ↓
logging-agent Pod created
```

Examples include node-level collectors and system components.

---

## 16. What is the difference between Job and CronJob?

### Answer

A **Job** represents finite work that should complete.

Example:

```yaml
kind: Job
```

Use cases:

```text
database migration
batch calculation
one-time data transformation
```

A **CronJob** creates Jobs according to a schedule.

Example:

```yaml
schedule: "0 2 * * *"
```

Use cases:

```text
nightly reports
scheduled cleanup
periodic processing
```

Important Job controls include:

```text
completions
parallelism
backoffLimit
activeDeadlineSeconds
```

For CronJobs, consider:

```text
concurrencyPolicy
startingDeadlineSeconds
successfulJobsHistoryLimit
failedJobsHistoryLimit
```

---

## 17. What are init containers?

### Answer

Init containers execute before normal application containers.

Example:

```text
Pod starts
   ↓
Init Container 1
   ↓ completes
Init Container 2
   ↓ completes
Application Container
```

Typical use cases:

```text
generate configuration
wait for dependency
perform initialization
download startup assets
set filesystem permissions
```

Example:

```yaml
initContainers:
- name: init-config
  image: busybox
```

All required init containers must successfully complete before normal application containers start.

### Senior advice

Do not build indefinite dependency waiting into init containers without appropriate timeout/failure behavior.

---

## 18. What is the sidecar pattern?

### Answer

A sidecar is a secondary container that supports the primary application container.

Example:

```text
Pod
├── Application
└── Proxy/Agent
```

Examples:

```text
service-mesh proxy
log processor
configuration synchronizer
security agent
```

Containers share Pod networking, so:

```text
Application
   ↓ localhost
Sidecar proxy
```

is possible.

### Trade-off

Sidecars increase:

```text
CPU consumption
memory consumption
operational complexity
startup/shutdown dependencies
debugging complexity
```

Use them when they provide clear benefits.

---

## 19. Explain graceful Pod termination.

### Answer

When Kubernetes terminates a Pod, the sequence broadly includes:

```text
Pod marked terminating
      ↓
Endpoint removal begins
      ↓
preStop hook if configured
      ↓
SIGTERM sent to container
      ↓
terminationGracePeriodSeconds
      ↓
SIGKILL if process remains
```

Applications should handle `SIGTERM`.

Example:

```text
SIGTERM
  ↓
stop accepting new traffic
  ↓
finish active requests
  ↓
close connections
  ↓
exit
```

A poorly designed application may drop customer requests during every rolling deployment.

### Senior design

Coordinate:

```text
readiness
preStop
load balancer behavior
termination grace period
application shutdown duration
```

---

## 20. Explain liveness, readiness, and startup probes.

### Answer

### Readiness probe

Answers:

> Should this Pod receive traffic?

Failure generally removes the Pod from eligible Service endpoints.

### Liveness probe

Answers:

> Is the application stuck or unhealthy enough that the container should restart?

### Startup probe

Answers:

> Has the application completed startup?

A startup probe can protect slow-starting applications from premature liveness failures. Current Kubernetes documentation distinguishes these three probe types and notes that startup probes delay liveness/readiness execution until startup succeeds. ([Kubernetes][2])

Example:

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080

livenessProbe:
  httpGet:
    path: /live
    port: 8080
```

### Common production mistake

Using the same dependency-heavy endpoint for liveness and readiness.

If the database goes down:

```text
DB unavailable
     ↓
Liveness fails
     ↓
Every application container restarts
     ↓
Additional load/outage
```

The application may not actually need a restart.

---

# Section 3 — Kubernetes Networking

## 21. Explain Kubernetes Service types.

### Answer

The major Service types are:

```text
ClusterIP
NodePort
LoadBalancer
ExternalName
```

### ClusterIP

Default.

Accessible inside the cluster.

```text
Pod
 ↓
ClusterIP Service
 ↓
Backend Pods
```

### NodePort

Exposes a port through cluster nodes.

```text
NodeIP:NodePort
```

### LoadBalancer

Requests an external load balancer where supported.

```text
External LB
    ↓
Service
    ↓
Pods
```

### ExternalName

Provides DNS indirection to an external hostname.

---

## 22. How does Kubernetes DNS work?

### Answer

Kubernetes commonly runs CoreDNS for cluster DNS.

A Service such as:

```text
payments
namespace = production
```

can be addressed as:

```text
payments.production.svc.cluster.local
```

Pods in the same namespace can commonly use:

```text
payments
```

Flow:

```text
Application
    ↓
DNS query
    ↓
CoreDNS
    ↓
Kubernetes Service records
```

Useful troubleshooting:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system <coredns-pod>

kubectl exec -it <pod> -- nslookup kubernetes.default
```

DNS failures can affect virtually every microservice simultaneously.

---

## 23. What is the difference between Ingress and Gateway API?

### Answer

Ingress provides HTTP/HTTPS routing into a cluster through an Ingress controller.

Example:

```text
Internet
   ↓
Ingress Controller
   ↓
Ingress rules
   ↓
Service
   ↓
Pods
```

Gateway API provides a newer and more expressive set of APIs intended to support more sophisticated traffic-management and infrastructure/application separation.

Conceptually:

```text
GatewayClass
    ↓
Gateway
    ↓
HTTPRoute / GRPCRoute / other routes
    ↓
Services
```

Senior engineers should understand that neither an Ingress resource nor Gateway configuration itself moves packets.

A supporting controller/data plane must implement it.

---

## 24. What is CNI?

### Answer

CNI is the Container Network Interface standard used for configuring Pod networking.

When a Pod starts:

```text
kubelet/runtime
      ↓
CNI plugin
      ↓
Create network interface
Assign Pod IP
Configure routing
Apply networking policies
```

Popular implementations include:

```text
Cilium
Calico
Flannel
Antrea
cloud-provider networking plugins
```

CNI solutions differ in areas such as:

```text
routing model
overlay vs native routing
NetworkPolicy
eBPF support
IP address management
encryption
observability
```

Choosing CNI is an architectural decision.

---

## 25. How does a Kubernetes Service load-balance to Pods?

### Answer

Suppose:

```text
Service: api
ClusterIP: 10.96.10.20
```

Backends:

```text
10.244.1.8
10.244.2.5
10.244.3.7
```

The Service's EndpointSlices identify eligible endpoints.

Traffic:

```text
Client
  ↓
10.96.10.20
  ↓
Service data plane
  ↓
one backend Pod
```

Historically kube-proxy commonly programs kernel networking rules for this.

Modern eBPF networking implementations may provide the Service data plane differently.

### Important

If a readiness probe fails:

```text
Pod running
but
not ready
```

it is generally removed from eligible Service endpoints.

---

## 26. What is NetworkPolicy?

### Answer

NetworkPolicy controls permitted Pod network traffic when the CNI supports policy enforcement.

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api
spec:
  podSelector:
    matchLabels:
      app: database
```

You can control:

```text
ingress
egress
source/destination Pods
namespaces
CIDRs
ports
```

A strong security model commonly starts with:

```text
default deny
```

then explicitly permits required flows.

Conceptually:

```text
Frontend → API ✓
API → Database ✓
Frontend → Database ✗
Unknown namespace → Database ✗
```

### Important

Creating NetworkPolicy objects has no effect unless the networking implementation supports and enforces them.

---

## 27. How would you troubleshoot Pod-to-Pod communication failure?

### Answer

Work layer by layer.

### 1. Check Pod IPs

```bash
kubectl get pods -o wide
```

### 2. Test direct connectivity

```bash
kubectl exec <pod> -- curl <pod-ip>:8080
```

### 3. Check routes/interfaces

Inside troubleshooting container:

```bash
ip addr
ip route
```

### 4. Inspect CNI

Check plugin Pods and logs.

```bash
kubectl get pods -n kube-system
```

### 5. Check NetworkPolicies

```bash
kubectl get networkpolicy -A
```

### 6. Check nodes

```bash
kubectl get nodes
```

### 7. Compare same-node vs cross-node

If:

```text
same-node traffic works
cross-node traffic fails
```

suspect:

```text
CNI routing
tunnel networking
firewall
security groups
MTU
host routes
```

---

## 28. A Service exists but is unreachable. What do you check?

### Answer

Start with:

```bash
kubectl get svc
kubectl describe svc <service>
```

Then check endpoints:

```bash
kubectl get endpointslices
```

If no endpoints exist:

```text
Service selector
     ↓
does not match Pod labels

or

Pods not Ready
```

Check:

```bash
kubectl get pods --show-labels
```

Verify:

```text
Service port
targetPort
container listening port
```

Then test:

```bash
curl <Pod-IP>:port
curl <Service-IP>:port
```

Interpretation:

```text
Pod works, Service fails
→ Service/data-plane problem

Pod itself fails
→ application or Pod networking problem
```

---

## 29. How would you troubleshoot CoreDNS problems?

### Answer

Check CoreDNS Pods:

```bash
kubectl get pods -n kube-system
```

Inspect logs:

```bash
kubectl logs -n kube-system <coredns-pod>
```

Test from application Pod:

```bash
cat /etc/resolv.conf

nslookup kubernetes.default

nslookup myservice.mynamespace
```

Check:

```text
CoreDNS Service
EndpointSlices
network connectivity
NetworkPolicy
Corefile configuration
upstream DNS
node DNS configuration
```

Useful separation:

```text
Cluster Service DNS fails
→ Kubernetes/CoreDNS issue

Cluster DNS works but internet DNS fails
→ upstream DNS/forwarding issue
```

---

## 30. How do you expose Kubernetes workloads to external traffic?

### Answer

Common methods:

```text
LoadBalancer Service
Ingress
Gateway API
NodePort
external reverse proxy
service mesh gateway
```

A typical production architecture:

```text
Internet
   ↓
Cloud Load Balancer
   ↓
Ingress/Gateway Data Plane
   ↓
Service
   ↓
Pods
```

Senior concerns include:

```text
TLS termination
WAF
health checks
source IP preservation
timeouts
connection draining
rate limiting
certificate automation
multi-zone availability
DNS
```

Exposure should be designed deliberately rather than simply setting every Service to `LoadBalancer`.

---

# Section 4 — Storage

## 31. What types of storage can Pods use?

### Answer

Pod storage includes:

```text
container writable filesystem
emptyDir
ConfigMap
Secret
PersistentVolume-backed storage
CSI volumes
hostPath
projected volumes
```

Container filesystem data is generally ephemeral.

If Pod is recreated:

```text
old container filesystem
     ↓
gone
```

Persistent application data should normally use appropriate persistent storage.

---

## 32. Explain PV and PVC.

### Answer

A **PersistentVolume (PV)** represents storage available to Kubernetes.

A **PersistentVolumeClaim (PVC)** represents a workload's request for storage.

Architecture:

```text
Pod
 ↓
PVC
 ↓
PV
 ↓
Storage backend
```

Example PVC:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```

The application should request storage through the PVC abstraction instead of needing to understand backend implementation details.

---

## 33. What is a StorageClass?

### Answer

StorageClass describes how dynamic storage should be provisioned.

Example:

```text
PVC requests 100Gi
    ↓
StorageClass
    ↓
CSI provisioner
    ↓
Cloud disk created
    ↓
PV created
    ↓
PVC bound
```

Storage classes may describe properties such as:

```text
storage backend
parameters
reclaim policy
volume binding behavior
expansion support
```

Dynamic provisioning eliminates the need to manually create a PV for every application.

---

## 34. Explain PersistentVolume access modes.

### Answer

Common access modes include:

```text
ReadWriteOnce
ReadOnlyMany
ReadWriteMany
ReadWriteOncePod
```

Conceptually:

### ReadWriteOnce

Volume writable from a single node at a time.

### ReadOnlyMany

Can be mounted read-only by multiple nodes when the backend supports it.

### ReadWriteMany

Can be mounted read/write by multiple nodes where supported.

### ReadWriteOncePod

Restricts writable mounting to one Pod where supported.

### Important

Access modes describe scheduling/mounting semantics and storage capabilities; they are not equivalent to filesystem-level application locking.

---

## 35. How does StatefulSet persistent storage work?

### Answer

StatefulSet often uses:

```yaml
volumeClaimTemplates:
```

For:

```text
db-0
db-1
db-2
```

Kubernetes can create:

```text
data-db-0
data-db-1
data-db-2
```

This association survives ordinary Pod replacement.

If `db-1` is recreated:

```text
new db-1
    ↓
same associated PVC
```

This provides stable storage identity.

### Senior point

Deletion behavior and retention policy should be explicitly understood before deleting StatefulSets or claims in production.

---

## 36. What is CSI?

### Answer

CSI stands for Container Storage Interface.

It standardizes integration between Kubernetes and storage systems.

Architecture:

```text
Kubernetes
    ↓
CSI controller/node components
    ↓
Storage provider
```

Typical capabilities include:

```text
provision volume
attach volume
mount volume
expand volume
snapshot volume
```

CSI enables storage vendors to implement integrations without embedding each provider directly inside core Kubernetes.

---

## 37. How do you expand a Kubernetes persistent volume?

### Answer

When supported by the StorageClass and CSI driver:

```text
PVC = 100Gi
   ↓
Edit request to 200Gi
   ↓
CSI/storage backend expands volume
   ↓
Filesystem expansion if required
```

Example:

```bash
kubectl edit pvc database-data
```

Change:

```yaml
resources:
  requests:
    storage: 200Gi
```

Before performing production expansion, verify:

```text
StorageClass allowVolumeExpansion
CSI support
cloud/backend limits
filesystem support
application behavior
```

---

## 38. What are VolumeSnapshots?

### Answer

Kubernetes CSI snapshot APIs can coordinate snapshots with compatible storage drivers.

Conceptually:

```text
PVC
 ↓
VolumeSnapshot
 ↓
Storage backend snapshot
```

Snapshots can be useful for:

```text
backup workflows
cloning
testing
pre-upgrade protection
```

### Senior warning

A storage snapshot is not automatically an application-consistent database backup.

For databases, consider:

```text
database quiescing
transaction consistency
WAL/binlog
database-native backup tools
```

---

## 39. When would you use local persistent storage?

### Answer

Local disks can provide high performance for workloads that can tolerate node affinity and handle replication themselves.

Examples:

```text
distributed databases
Kafka-like systems
high-performance local storage
```

Trade-off:

```text
Volume tied to node
      ↓
Node fails
      ↓
Storage may be unavailable
```

Therefore the application often needs:

```text
replication
quorum
failure recovery
```

Local storage is not equivalent to automatically highly available storage.

---

## 40. A PVC remains Pending. How do you troubleshoot it?

### Answer

Start with:

```bash
kubectl describe pvc <pvc>
```

Look at Events.

Check:

```bash
kubectl get storageclass
kubectl get pv
```

Possible causes:

```text
StorageClass does not exist
wrong storageClassName
CSI controller failure
cloud quota exceeded
unsupported access mode
no matching static PV
zone/topology constraints
provisioning error
```

If using delayed binding:

```text
WaitForFirstConsumer
```

the PVC may intentionally remain unbound until a Pod is scheduled.

Always inspect Events before guessing.

---

# Section 5 — Scheduling, Resources and Autoscaling

## 41. Explain resource requests and limits.

### Answer

Example:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "1"
    memory: "1Gi"
```

### Requests

Used primarily for scheduling and resource guarantees.

```text
Scheduler asks:
Does node have requested capacity?
```

### Limits

Constrain resource consumption.

CPU limit typically results in throttling when exceeded.

Memory behaves differently:

```text
Container memory exceeds effective limit
     ↓
OOM risk
     ↓
process may be killed
```

### Senior point

Bad requests affect cluster scheduling and utilization.

Bad limits can affect application performance.

Resource configuration should come from measurements.

---

## 42. Explain Kubernetes QoS classes.

### Answer

Kubernetes classifies Pods broadly into:

```text
Guaranteed
Burstable
BestEffort
```

### Guaranteed

Containers meet requirements where CPU/memory requests and limits align appropriately.

### Burstable

At least some requests/limits exist but the Pod doesn't meet Guaranteed criteria.

### BestEffort

No CPU/memory requests or limits.

Under node memory pressure, lower-priority resource guarantees influence eviction behavior.

### Senior advice

Critical production workloads should rarely be deployed with no resource requests at all.

---

## 43. How does Kubernetes scheduling work?

### Answer

A simplified scheduler cycle:

```text
New Pending Pod
     ↓
Scheduling queue
     ↓
Filtering
     ↓
Scoring
     ↓
Select node
     ↓
Bind Pod
```

Filtering checks hard requirements such as:

```text
resources
taints
node selector
affinity
volume topology
```

Scoring compares feasible nodes.

A Pod that cannot find any suitable node remains:

```text
Pending
```

with Events explaining why.

Example:

```bash
kubectl describe pod <pod>
```

may show:

```text
Insufficient cpu
node(s) had untolerated taint
node(s) didn't match Pod's node affinity
```

---

## 44. Explain taints and tolerations.

### Answer

Taints are applied to nodes.

Example:

```bash
kubectl taint nodes gpu-node workload=gpu:NoSchedule
```

This says:

```text
Do not schedule ordinary Pods here.
```

A Pod needs a matching toleration.

```yaml
tolerations:
- key: workload
  value: gpu
  effect: NoSchedule
```

Common effects:

```text
NoSchedule
PreferNoSchedule
NoExecute
```

### Key distinction

Taints repel Pods from nodes.

Affinity attracts/selects Pods toward appropriate nodes.

Toleration does not necessarily force a Pod onto the tainted node.

---

## 45. Explain node affinity and pod affinity/anti-affinity.

### Answer

### Node affinity

Controls which nodes a Pod prefers or requires.

Example:

```text
Run only on nodes:
disk=ssd
```

### Pod affinity

Places Pods near specified other Pods.

Example:

```text
API Pods near cache Pods
```

### Pod anti-affinity

Separates workloads.

Example:

```text
Do not place two replicas
on the same node
```

For HA workloads:

```text
Replica A → Node 1
Replica B → Node 2
Replica C → Node 3
```

This reduces correlated failure.

### Trade-off

Complex hard affinity rules can create unschedulable Pods.

---

## 46. What are topology spread constraints?

### Answer

Topology spread constraints distribute Pods across failure domains.

Examples:

```text
zones
nodes
regions/custom topology
```

Example objective:

```text
3 replicas

Zone A → 1
Zone B → 1
Zone C → 1
```

rather than:

```text
Zone A → 3
Zone B → 0
Zone C → 0
```

This is important for availability.

Use cases:

```text
multi-zone application replicas
avoid node concentration
controlled skew
```

For many HA applications, topology spread provides clearer distribution semantics than large sets of anti-affinity rules.

---

## 47. Explain Pod priority and preemption.

### Answer

Pods can use a PriorityClass.

Higher priority workloads may receive scheduling preference.

If necessary, Kubernetes can potentially preempt lower-priority Pods to make room.

Example:

```text
Critical Pod needs 4 CPU
No capacity
    ↓
Lower-priority workloads identified
    ↓
Some evicted/preempted
    ↓
Critical Pod scheduled
```

Use this carefully.

Overuse of high priority results in:

```text
everything is critical
→ priority becomes meaningless
```

Typical use:

```text
cluster-critical workloads
core platform services
critical business workloads
```

---

## 48. What is a PodDisruptionBudget?

### Answer

A PodDisruptionBudget limits disruption from **voluntary** operations.

Example:

```yaml
minAvailable: 2
```

For a three-replica application:

```text
At least two should remain available
during voluntary disruption.
```

This matters during:

```text
node drain
cluster maintenance
node upgrades
autoscaler eviction
```

Alternatively:

```yaml
maxUnavailable: 1
```

### Important

A PDB does not protect against involuntary failure such as:

```text
hardware crash
power loss
kernel panic
```

Also, overly restrictive PDBs can prevent node maintenance.

---

## 49. What is Horizontal Pod Autoscaler?

### Answer

HPA changes workload replica count based on metrics.

Example:

```text
CPU rises
   ↓
HPA
   ↓
Deployment replicas:
3 → 6
```

It can use metrics from APIs including:

```text
resource metrics
custom metrics
external metrics
```

Resource metrics are commonly supplied using Metrics Server through `metrics.k8s.io`; custom and external metrics use their respective APIs/adapters. ([Kubernetes][3])

Example:

```bash
kubectl autoscale deployment api \
  --cpu-percent=70 \
  --min=3 \
  --max=20
```

### Senior considerations

HPA effectiveness depends on:

```text
accurate resource requests
metric quality
application startup time
traffic pattern
stabilization behavior
downstream capacity
```

Scaling application Pods does not help if the database is already saturated.

---

## 50. Explain HPA, VPA, and node autoscaling.

### Answer

These operate at different layers.

### HPA

Changes number of Pods.

```text
Pods 3 → 10
```

### VPA

Adjusts Pod resource recommendations/requests and, depending on configuration/tooling, can trigger workload resizing/replacement behavior.

```text
CPU request 500m → 1 CPU
```

### Node autoscaling

Changes number/capacity of cluster nodes.

```text
Nodes 5 → 8
```

A common chain:

```text
Traffic grows
    ↓
HPA creates Pods
    ↓
Pods become Pending
    ↓
Node autoscaler adds capacity
    ↓
Pods schedule
```

Senior engineers tune the system holistically rather than independently.

---

## 51. How does Cluster Autoscaler work?

### Answer

Cluster Autoscaler watches for situations such as:

```text
Pods cannot schedule
because cluster lacks resources
```

Then it may increase node-group capacity.

Conceptually:

```text
Pending Pod
   ↓
Scheduler: insufficient CPU
   ↓
Cluster Autoscaler
   ↓
Increase node group
   ↓
Node joins
   ↓
Pod schedules
```

It may also scale down nodes judged unnecessary, respecting constraints.

Consider:

```text
PodDisruptionBudgets
local storage
node affinity
taints
daemonsets
system Pods
cloud quotas
```

---

## 52. How do you troubleshoot node resource pressure?

### Answer

Check:

```bash
kubectl describe node <node>
kubectl top node
kubectl top pods -A
```

Look for conditions:

```text
MemoryPressure
DiskPressure
PIDPressure
```

Check node OS:

```bash
free -m
df -h
df -i
top
iostat -xz 1
```

Check:

```text
container logs
image filesystem
container writable layers
ephemeral storage
runaway processes
```

Kubernetes may begin evicting Pods under resource pressure.

A senior response should identify the cause instead of simply adding nodes.

---

## 53. A Pod is OOMKilled. What does that mean?

### Answer

The container process was killed because of memory pressure/limits.

Check:

```bash
kubectl describe pod <pod>
```

or:

```bash
kubectl get pod <pod> \
  -o jsonpath='{.status.containerStatuses[*].lastState}'
```

Potential reasons:

```text
memory limit too low
memory leak
runtime heap misconfiguration
unexpected workload spike
too many worker threads
large in-memory cache
```

Example:

```yaml
limits:
  memory: 512Mi
```

but application requires:

```text
700Mi
```

results in repeated failure.

### Senior response

Do not automatically raise the limit.

First determine:

```text
expected baseline
peak memory
leak trend
heap/native memory
node pressure
```

---

## 54. What is CPU throttling in Kubernetes?

### Answer

If a container has:

```yaml
limits:
  cpu: "500m"
```

then it is limited to approximately half a CPU's capacity over scheduling periods.

If application demand exceeds this limit:

```text
needs 1.5 CPU
limit = 0.5
    ↓
CPU throttling
    ↓
latency
```

The container may appear:

```text
healthy
not OOM
not crashed
```

yet performance is poor.

Investigate:

```text
CPU usage
CPU throttled seconds
application latency
resource limits
```

This is particularly important for latency-sensitive services.

---

## 55. A Pod stays Pending. How would you troubleshoot?

### Answer

Start:

```bash
kubectl describe pod <pod>
```

Look at Events.

Common causes:

```text
Insufficient CPU
Insufficient memory
unbound PVC
node affinity mismatch
untolerated taint
topology constraint
host-port conflict
no suitable node
```

Check:

```bash
kubectl get nodes
kubectl describe nodes
kubectl get pvc
```

Example Event:

```text
0/10 nodes are available:
5 Insufficient cpu,
5 had untolerated taint
```

The scheduler usually tells you why.

---

# Section 6 — Kubernetes Security

## 56. Explain Kubernetes RBAC.

### Answer

RBAC controls authorization.

Main objects:

```text
Role
ClusterRole
RoleBinding
ClusterRoleBinding
```

### Role

Namespace-scoped permissions.

### ClusterRole

Cluster-wide or reusable permissions.

### Binding

Assigns roles to:

```text
users
groups
ServiceAccounts
```

Example concept:

```text
Role:
can get/list Pods

RoleBinding:
assign Role to developer
```

Test access:

```bash
kubectl auth can-i delete pods \
  --as=user@example.com
```

### Senior principle

Use least privilege.

Avoid casually granting:

```text
cluster-admin
```

to applications or automation.

---

## 57. What is a ServiceAccount?

### Answer

A ServiceAccount provides an identity for workloads interacting with the Kubernetes API.

Example:

```text
Pod
 ↓
ServiceAccount
 ↓
RBAC
 ↓
API Server
```

A Pod may use:

```yaml
serviceAccountName: payments-api
```

Then RBAC determines which API operations it can perform.

### Security practice

If a workload does not need Kubernetes API access, consider:

```yaml
automountServiceAccountToken: false
```

where appropriate.

Do not use one highly privileged ServiceAccount across unrelated applications.

---

## 58. How should Kubernetes Secrets be managed?

### Answer

A Secret object stores sensitive values.

Example:

```yaml
kind: Secret
```

However, base64 representation is **not encryption**.

```text
base64
≠
secure encryption
```

Production practices include:

```text
encrypt Secrets at rest in etcd
strict RBAC
avoid committing plaintext Secrets to Git
use external secret-management systems
rotate credentials
restrict secret exposure
audit access
```

Secret-management systems might integrate with:

```text
Vault
cloud secret managers
external-secrets solutions
CSI secret stores
```

---

## 59. What is the difference between ConfigMap and Secret?

### Answer

### ConfigMap

Used for non-sensitive configuration.

Examples:

```text
application flags
URLs
feature settings
configuration files
```

### Secret

Used for sensitive data.

Examples:

```text
passwords
tokens
certificates
keys
```

Both can be exposed through:

```text
environment variables
mounted files
API
```

### Senior point

Environment variables can be operationally convenient but may leak into:

```text
process dumps
debug output
diagnostic tooling
```

Mounted secret files or external secret integration may be preferable depending on the application.

---

## 60. What are Pod Security Standards?

### Answer

Kubernetes defines Pod Security Standards with profiles:

```text
Privileged
Baseline
Restricted
```

`Privileged` allows broad permissions.

`Baseline` blocks common privilege-escalation risks while preserving compatibility.

`Restricted` represents a stronger hardened posture.

Pod Security Admission can enforce, audit, or warn based on namespace policy configuration. ([Kubernetes][4])

### Senior approach

For production application namespaces, aim toward:

```text
Restricted
```

where application compatibility permits.

Exceptions should be tightly controlled and documented.

---

## 61. What is `securityContext`?

### Answer

`securityContext` configures Pod/container security attributes.

Example:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10001
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
```

Other controls include:

```text
capabilities
seccompProfile
fsGroup
runAsGroup
```

Example:

```yaml
capabilities:
  drop:
  - ALL
```

A hardened application often combines:

```text
non-root
read-only filesystem
no privilege escalation
minimal capabilities
seccomp
```

---

## 62. What are Linux capabilities and seccomp in Kubernetes?

### Answer

Linux capabilities divide root privileges into smaller permissions.

Examples:

```text
NET_ADMIN
SYS_ADMIN
NET_BIND_SERVICE
CHOWN
```

Rather than giving the application full privilege:

```yaml
privileged: true
```

prefer:

```yaml
capabilities:
  drop:
  - ALL
```

and add only required capabilities.

Seccomp restricts allowed system calls.

Example:

```yaml
seccompProfile:
  type: RuntimeDefault
```

### Senior principle

Avoid:

```text
privileged containers
hostPID
hostNetwork
hostPath
SYS_ADMIN
```

unless explicitly justified.

---

## 63. What are admission controllers and admission webhooks?

### Answer

Admission occurs after authentication and authorization but before an API object is persisted.

Flow:

```text
API request
    ↓
Authentication
    ↓
Authorization
    ↓
Admission
    ↓
Persist
```

Two webhook categories:

```text
MutatingAdmissionWebhook
ValidatingAdmissionWebhook
```

### Mutating

Can modify objects.

Example:

```text
inject labels
inject sidecar
apply defaults
```

### Validating

Accepts or rejects requests.

Examples:

```text
disallow privileged containers
require approved registries
enforce naming standards
```

### Senior warning

A broken admission webhook can become a cluster-wide deployment outage if failure policies and availability are poorly designed.

---

## 64. How do you secure the container image supply chain?

### Answer

A mature pipeline might perform:

```text
Source commit
   ↓
Build
   ↓
Dependency scan
   ↓
Image vulnerability scan
   ↓
SBOM
   ↓
Sign/attest
   ↓
Registry
   ↓
Admission verification
   ↓
Deployment
```

Best practices:

```text
trusted base images
immutable tags/digests
image scanning
signed artifacts
SBOM generation
minimal runtime images
dependency updates
private registry/RBAC
```

Avoid:

```yaml
image: myapp:latest
```

for production deployments when deterministic artifacts matter.

Prefer immutable identification such as a digest.

---

## 65. How would you implement network security in Kubernetes?

### Answer

Use defense in depth.

Example:

```text
External firewall/security groups
        ↓
Load balancer controls
        ↓
Ingress/Gateway policy
        ↓
NetworkPolicy
        ↓
Application authentication
```

Recommended principles:

```text
default-deny NetworkPolicy
explicit ingress permissions
explicit egress permissions
namespace segmentation
TLS/mTLS where appropriate
minimal externally exposed services
```

Never assume:

```text
It's inside the cluster, therefore it's trusted.
```

A compromised Pod is an internal attacker.

---

## 66. How do you approach Kubernetes multi-tenancy?

### Answer

Possible isolation mechanisms include:

```text
Namespaces
RBAC
ResourceQuota
LimitRange
NetworkPolicy
Pod Security
node isolation
separate clusters
```

Soft multi-tenancy might use namespaces.

Harder security boundaries may require:

```text
dedicated nodes
sandboxing
virtual clusters
separate Kubernetes clusters
```

### Senior decision

The answer depends on threat model.

Do not claim namespaces are automatically strong security isolation equivalent to separate physical systems.

---

## 67. How do you encrypt Kubernetes Secrets at rest?

### Answer

Secrets are persisted through the API server into etcd.

Production clusters should consider API-server encryption-at-rest configuration so sensitive resources are encrypted before persistence.

Conceptually:

```text
Secret
  ↓
API server
  ↓
Encryption provider
  ↓
encrypted data
  ↓
etcd
```

Potential providers may involve:

```text
AES-based providers
external KMS
cloud KMS integration
```

Also secure:

```text
etcd backups
control-plane disks
KMS access
RBAC
```

An encrypted etcd datastore is not useful if the attacker can freely ask the API server to decrypt every Secret.

---

## 68. Explain Kubernetes authentication and authorization.

### Answer

Authentication asks:

> Who are you?

Examples may include:

```text
client certificates
OIDC
ServiceAccount tokens
cloud IAM integrations
```

Authorization asks:

> What are you allowed to do?

Usually:

```text
RBAC
```

Flow:

```text
Request
 ↓
Authentication
 ↓
Identity established
 ↓
Authorization
 ↓
Allowed?
 ↓
Admission
```

Senior interviewers expect candidates to clearly distinguish:

```text
authentication
authorization
admission
```

---

## 69. What are Kubernetes audit logs?

### Answer

Audit logs record API-server activity.

They can answer:

```text
Who made the request?
What resource?
What action?
When?
Was it allowed?
What was the result?
```

Useful for:

```text
security investigation
compliance
incident response
change tracking
```

A production audit policy should balance:

```text
visibility
storage volume
sensitive-data exposure
performance
```

Audit data should normally be shipped to durable centralized storage/SIEM.

---

## 70. A Kubernetes Secret was accidentally committed to Git. What do you do?

### Answer

Assume the secret is compromised.

Do not only delete the Git line.

Process:

```text
1. Revoke/rotate credential.
2. Update secret-management source.
3. Update Kubernetes workloads.
4. Restart/reload affected applications safely.
5. Verify old credential no longer works.
6. Remove secret from repository history where needed.
7. Investigate access/logs.
8. Add secret scanning/prevention.
```

The most important action is:

```text
Rotate the credential.
```

Git history cleanup alone cannot prove nobody copied it.

---

# Section 7 — Deployments, Helm, GitOps and Upgrades

## 71. What is Helm?

### Answer

Helm is a package manager/template system for Kubernetes applications.

A Helm chart typically contains:

```text
Chart.yaml
values.yaml
templates/
```

Example:

```bash
helm install payments ./chart
```

Upgrade:

```bash
helm upgrade payments ./chart
```

Rollback:

```bash
helm rollback payments <revision>
```

Useful:

```bash
helm history payments
```

### Senior concern

Helm can simplify packaging but can also produce complex templates.

Keep charts:

```text
testable
predictable
version controlled
easy to render/debug
```

Use:

```bash
helm template
```

to inspect generated manifests.

---

## 72. Helm vs Kustomize — when would you use each?

### Answer

### Helm

Best when:

```text
packaging reusable applications
parameterized charts
versioned releases
third-party software distribution
```

### Kustomize

Applies declarative overlays/patches to YAML.

Example:

```text
base/
overlays/
  dev/
  staging/
  prod/
```

Useful when:

```text
base manifests stay close to native Kubernetes YAML
environment differences are patches
```

Both can coexist.

Example:

```text
Helm for third-party chart packaging
+
Kustomize/GitOps overlays for environment composition
```

---

## 73. What is GitOps?

### Answer

GitOps uses Git as the declarative source of truth for desired cluster/application state.

Architecture:

```text
Engineer changes Git
       ↓
Pull Request
       ↓
Review
       ↓
Merge
       ↓
GitOps controller
       ↓
Compare Git vs cluster
       ↓
Reconcile
```

Common tools include:

```text
Argo CD
Flux
```

Benefits:

```text
auditability
repeatable deployments
drift detection
pull-based deployment
rollback through Git
```

### Senior principle

Avoid mixing uncontrolled manual `kubectl edit` changes with GitOps-controlled objects because the controller may revert them.

Emergency changes should be reconciled back into Git.

---

## 74. How does a rolling update work?

### Answer

For a Deployment:

```text
Old ReplicaSet
    ↓ scale down gradually

New ReplicaSet
    ↑ scale up gradually
```

Configuration includes:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

`maxSurge` controls extra temporary Pods.

`maxUnavailable` controls how many may be unavailable.

A safe rolling deployment also depends on:

```text
readiness probes
graceful termination
capacity
PDBs
application compatibility
```

A rolling update alone does not guarantee zero downtime.

---

## 75. How would you implement zero-downtime application deployment?

### Answer

Combine:

```text
multiple replicas
readiness probes
rolling updates
graceful SIGTERM handling
connection draining
appropriate termination grace
compatible schema changes
sufficient surge capacity
```

Sequence:

```text
New Pod starts
    ↓
Startup complete
    ↓
Readiness succeeds
    ↓
Pod receives traffic
    ↓
Old Pod becomes terminating
    ↓
traffic drained
    ↓
active requests finish
    ↓
old process exits
```

Database changes should often use:

```text
expand
migrate
contract
```

rather than breaking schema changes.

---

## 76. Blue-green vs canary deployment?

### Answer

### Blue-green

Two complete environments:

```text
Blue = current
Green = new
```

After validation:

```text
Traffic
  ↓
Blue → Green
```

Fast rollback:

```text
Green → Blue
```

Trade-off:

```text
higher temporary capacity
```

### Canary

Send small traffic percentage to the new version:

```text
95% → old
5% → new
```

Observe:

```text
errors
latency
business metrics
```

Then gradually increase.

Canary provides better risk control but needs traffic-management and observability capabilities.

---

## 77. How would you upgrade a production Kubernetes cluster?

### Answer

A safe process generally includes:

```text
1. Review release notes/deprecations.
2. Verify supported version path.
3. Test upgrade in staging.
4. Back up critical control-plane state.
5. Check add-on compatibility.
6. Upgrade control plane.
7. Validate control plane.
8. Drain worker nodes gradually.
9. Upgrade node components/runtime.
10. Validate workloads.
11. Upgrade cluster add-ons.
12. Monitor.
```

Check compatibility for:

```text
CNI
CSI
Ingress/Gateway controller
service mesh
metrics stack
admission webhooks
operators
CRDs
```

Do not treat a cluster upgrade as only a kubelet package update.

---

## 78. Explain Kubernetes version skew.

### Answer

Kubernetes components do not have unlimited version compatibility.

For current Kubernetes policy:

* kubelets must not be newer than the kube-apiserver and can generally be up to three minor versions older.
* `kubectl` is supported within one minor version older or newer than kube-apiserver.
* HA API-server instances must remain within one minor version of each other.
* minor-version upgrades must not skip API-server minor releases. ([Kubernetes][5])

Example with an API server at 1.37:

```text
kubelet:
1.37 ✓
1.36 ✓
1.35 ✓
1.34 ✓
1.38 ✗
```

The exact version-skew policy should always be checked before an upgrade.

---

## 79. How do you safely drain a Kubernetes node?

### Answer

First:

```bash
kubectl cordon node-1
```

prevents new normal workloads from scheduling there.

Then:

```bash
kubectl drain node-1 \
  --ignore-daemonsets \
  --delete-emptydir-data
```

depending on workload requirements.

Drain attempts to evict workloads while respecting relevant disruption policies.

After maintenance:

```bash
kubectl uncordon node-1
```

### Before draining

Check:

```text
PDBs
StatefulSets
local storage
DaemonSets
single-replica workloads
critical Pods
capacity on remaining nodes
```

A blind drain can cause an outage.

---

## 80. Managed Kubernetes vs self-managed Kubernetes?

### Answer

Managed platforms commonly manage much of the control-plane responsibility.

Examples include:

```text
Amazon EKS
Azure Kubernetes Service
Google Kubernetes Engine
```

Advantages:

```text
managed control plane
integrated cloud IAM
managed upgrade capabilities
cloud networking/storage integration
reduced etcd/control-plane operations
```

Self-managed Kubernetes provides more control but requires responsibility for:

```text
etcd
control-plane HA
certificates
patching
upgrades
backups
security
```

### Senior decision

Choose based on:

```text
business requirements
regulatory needs
team expertise
cost
operational burden
platform integrations
```

Not simply because self-managed appears cheaper on raw VM cost.

---

# Section 8 — Observability and Monitoring

## 81. What should you monitor in Kubernetes?

### Answer

Monitor multiple levels.

### Cluster

```text
API latency/errors
etcd performance
scheduler
controller manager
```

### Nodes

```text
CPU
memory
disk
inode
network
pressure conditions
```

### Pods

```text
CPU
memory
restart count
OOM
status
```

### Workloads

```text
desired vs available replicas
deployment failures
HPA
Jobs
```

### Applications

```text
request rate
latency
errors
saturation
business metrics
```

### Dependencies

```text
databases
queues
external APIs
```

Monitoring only Pod status is insufficient.

---

## 82. Metrics Server vs Prometheus — what is the difference?

### Answer

### Metrics Server

Provides lightweight resource metrics used by functions such as:

```text
kubectl top
HPA resource metrics
```

Example:

```bash
kubectl top pods
kubectl top nodes
```

It is not intended to be a complete long-term monitoring system.

### Prometheus

Typically provides:

```text
time-series retention
application metrics
alerting integration
PromQL
historical analysis
dashboards through Grafana
```

A production environment may use both:

```text
Metrics Server → HPA/basic resource metrics
Prometheus → observability
```

---

## 83. What is kube-state-metrics?

### Answer

kube-state-metrics exposes metrics based on Kubernetes API object state.

Examples:

```text
Deployment desired replicas
Deployment available replicas
Pod status
PVC phase
Job completion state
Node conditions
```

It differs from node/container runtime usage metrics.

Think:

```text
Prometheus/node metrics:
How much CPU is Pod using?

kube-state-metrics:
Is Deployment configured for 5 replicas,
and how many are available?
```

Both types are useful.

---

## 84. How important are Kubernetes Events?

### Answer

Extremely important for troubleshooting.

Use:

```bash
kubectl get events \
  --sort-by=.lastTimestamp
```

Or:

```bash
kubectl describe pod <pod>
```

Events reveal issues such as:

```text
FailedScheduling
FailedMount
BackOff
Unhealthy
ImagePullBackOff
FailedCreate
FailedAttachVolume
Evicted
```

Before diving into node internals, inspect the resource's Events.

### Important

Events are not intended as a permanent long-term observability store.

Export important information externally if longer retention is needed.

---

## 85. Which `kubectl` commands should a Senior DevOps Engineer know?

### Answer

At minimum:

```bash
kubectl get pods -A
kubectl get pods -o wide
kubectl describe pod <pod>
kubectl logs <pod>
kubectl logs <pod> --previous
kubectl exec -it <pod> -- sh
kubectl top pod
kubectl top node
kubectl get events
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl rollout undo deployment/<name>
kubectl get endpointslices
kubectl get networkpolicy -A
kubectl get pvc
kubectl describe node <node>
kubectl auth can-i
kubectl diff -f manifest.yaml
kubectl apply --dry-run=server
```

For temporary debugging:

```bash
kubectl debug
```

is particularly useful when the application image contains no troubleshooting tools.

---

# Section 9 — Common Production Troubleshooting

## 86. How do you troubleshoot CrashLoopBackOff?

### Answer

CrashLoopBackOff means a container repeatedly starts and exits, and Kubernetes increases restart backoff.

Check:

```bash
kubectl describe pod <pod>
kubectl logs <pod>
kubectl logs <pod> --previous
```

Possible causes:

```text
application crash
bad configuration
missing Secret
dependency unavailable
bad command
permissions
OOMKilled
liveness failure
port conflict
migration failure
```

Inspect exit information:

```bash
kubectl get pod <pod> -o yaml
```

### Senior sequence

```text
Exit code
  ↓
Previous logs
  ↓
Events
  ↓
Configuration
  ↓
Resources
  ↓
Dependencies
```

Do not repeatedly delete the Pod hoping the problem disappears.

---

## 87. How do you troubleshoot ImagePullBackOff?

### Answer

Inspect:

```bash
kubectl describe pod <pod>
```

Common reasons:

```text
wrong image name
wrong tag
registry unavailable
authentication failure
missing imagePullSecret
rate limiting
DNS/network issue
certificate failure
architecture mismatch
```

Typical Event:

```text
Failed to pull image
```

Verify:

```text
registry hostname
repository
tag/digest
credentials
node connectivity
```

If private registry:

```bash
kubectl get secret
```

and confirm the workload references the correct pull credentials.

---

## 88. How do you troubleshoot Node NotReady?

### Answer

Start:

```bash
kubectl describe node <node>
```

Check conditions.

Then inspect node:

```bash
systemctl status kubelet
journalctl -u kubelet
```

Check:

```text
container runtime
network
CNI
disk
memory
PID pressure
API server connectivity
certificates
clock
```

Commands:

```bash
df -h
df -i
free -m
systemctl status containerd
```

Potential causes:

```text
kubelet stopped
API connectivity lost
runtime failure
CNI failure
disk full
certificate problem
kernel problem
node crash
```

---

## 89. How do you troubleshoot Kubernetes control-plane failure?

### Answer

Determine which component is failing.

Check:

```text
kube-apiserver
etcd
scheduler
controller manager
load balancer
DNS/network
certificates
```

If kubeadm static Pods are used:

```bash
crictl ps
crictl logs <container>
```

Check:

```bash
journalctl -u kubelet
```

For API issues:

```bash
kubectl get --raw='/readyz?verbose'
```

if API access is possible.

Investigate:

```text
etcd latency
disk full
certificate expiry
API-server flags
admission webhook failure
control-plane networking
```

Do not restart every control-plane component simultaneously on a quorum-sensitive cluster.

---

## 90. How would you back up and recover etcd?

### Answer

For self-managed clusters, etcd snapshot backups are critical.

Conceptually:

```text
etcd
 ↓
Snapshot
 ↓
Encrypted external backup storage
```

Back up:

```text
etcd data
plus
critical cluster configuration/certificates
```

Recovery generally involves:

```text
stop/replace failed control plane as needed
restore snapshot to new etcd data location
configure etcd members
start etcd
start API server
validate resources
```

### Senior requirement

A backup strategy must include:

```text
regular snapshots
off-node copies
encryption
retention
restore testing
documented RTO/RPO
```

Backup success without restore testing is incomplete.

---

# Section 10 — Advanced Incident Scenarios

## 91. Cluster-wide DNS resolution suddenly fails. What is your response?

### Answer

First determine scope:

```text
all namespaces?
all nodes?
internal DNS only?
external DNS only?
```

Check CoreDNS:

```bash
kubectl get pods -n kube-system
kubectl logs -n kube-system <coredns>
```

Check Service:

```bash
kubectl get svc -n kube-system
kubectl get endpointslices -n kube-system
```

Test:

```bash
kubectl exec <pod> -- \
  nslookup kubernetes.default
```

Inspect:

```text
Corefile
NetworkPolicy
CNI
Service networking
node resolv.conf
upstream DNS
```

If CoreDNS Pods are healthy but inaccessible:

```text
Service/CNI/network issue
```

If internal names work but external names fail:

```text
upstream forwarding/resolver issue
```

---

## 92. kube-apiserver latency suddenly increases. What do you investigate?

### Answer

Check:

```text
API request latency
request rate
etcd latency
API CPU/memory
admission webhook latency
LIST/WATCH behavior
controller activity
network
```

Potential cause chain:

```text
slow admission webhook
    ↓
API request waits
    ↓
client retries
    ↓
API request load increases
    ↓
cluster-wide degradation
```

Also inspect etcd:

```text
disk fsync
leader changes
network RTT
database size
compaction/defragmentation needs
```

Correlate with:

```text
new controller/operator deployment
GitOps storm
mass resource update
cluster upgrade
network event
```

---

## 93. A Kubernetes node filesystem is full. How do you troubleshoot?

### Answer

Check:

```bash
df -h
df -i
```

Investigate:

```text
container images
container writable layers
logs
emptyDir
orphaned files
runtime storage
journald
```

Check:

```bash
du -xhd1 /var/lib/containerd
du -xhd1 /var/log
```

Kubernetes may report:

```text
DiskPressure
```

and evict workloads.

### Do not

Blindly delete files from container-runtime storage.

Use runtime/container lifecycle tools and understand ownership.

Prevent recurrence using:

```text
log rotation
ephemeral-storage requests/limits
image garbage collection
disk alerts
capacity planning
```

---

## 94. Kubernetes certificates expire. What symptoms might appear?

### Answer

Depending on certificate:

```text
kubectl failures
kubelet unable to authenticate
control-plane components unable to communicate
etcd TLS failure
node registration issues
```

Errors may include:

```text
x509: certificate has expired
```

For kubeadm-managed environments:

```bash
kubeadm certs check-expiration
```

may help inspect certificates.

Production operations should monitor certificate expiry well in advance.

Also consider certificates used by:

```text
Ingress
webhooks
service meshes
private registries
internal applications
```

not just Kubernetes' core PKI.

---

## 95. A Pod or namespace is stuck in Terminating. What do you check?

### Answer

Check:

```bash
kubectl describe pod <pod>
kubectl get pod <pod> -o yaml
```

Potential causes:

```text
finalizers
volume detach
kubelet/node unreachable
preStop hook
grace period
API finalization controller
```

For namespaces:

```bash
kubectl get namespace <ns> -o yaml
```

look at:

```text
spec.finalizers
status.conditions
```

### Senior warning

Do not remove finalizers blindly.

Finalizers exist because cleanup is expected.

First identify:

```text
Which controller owns this finalizer?
Why isn't cleanup completing?
What resource would be leaked?
```

Force removal is a recovery technique, not the first troubleshooting step.

---

## 96. An admission webhook goes down and deployments stop. What do you do?

### Answer

Symptoms may include:

```text
kubectl apply timing out
Pod creation fails
deployment controllers cannot create replicas
```

Check API-server errors and webhook Service/endpoints.

```bash
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations
```

Investigate:

```text
webhook Pods
Service
TLS certificate
DNS
NetworkPolicy
timeout
failurePolicy
```

Potential mitigation depends on risk:

```text
restore webhook
scale webhook
correct certificate/networking
temporarily change policy only if security impact is understood
```

### Design lesson

Critical admission webhooks should have:

```text
multiple replicas
topology distribution
appropriate PDB
short timeouts
monitoring
carefully chosen failurePolicy
```

---

## 97. A Deployment rollout is stuck. How do you troubleshoot?

### Answer

Start:

```bash
kubectl rollout status deployment/api
kubectl describe deployment api
kubectl get rs
kubectl get pods
```

Inspect new Pods:

```bash
kubectl describe pod <new-pod>
kubectl logs <new-pod>
```

Common causes:

```text
image pull failure
readiness failure
CrashLoopBackOff
insufficient resources
PDB restrictions
quota
PVC issues
bad configuration
```

If customer impact is increasing:

```bash
kubectl rollout undo deployment/api
```

may restore the previous revision when appropriate.

### Senior principle

Rollback first when necessary to restore service; investigate the failed version after impact is mitigated.

---

## 98. How do you perform node maintenance safely?

### Answer

Before maintenance:

```text
Check node workloads
Check spare cluster capacity
Check PDBs
Check stateful/local storage
Check application replicas
```

Then:

```bash
kubectl cordon node-1
kubectl drain node-1 --ignore-daemonsets
```

Perform:

```text
OS patch
kernel update
runtime update
hardware maintenance
```

Validate:

```text
kubelet
runtime
CNI
CSI
node conditions
```

Then:

```bash
kubectl uncordon node-1
```

For large clusters, use controlled batches instead of draining many nodes simultaneously.

---

## 99. What should a Kubernetes disaster-recovery strategy contain?

### Answer

Start from business requirements:

```text
RPO
RTO
```

Protect:

```text
etcd/control-plane state
application databases
persistent volumes
GitOps repositories
Helm configuration
Secrets/KMS
certificates
container registry
DNS
cloud infrastructure definitions
```

Architecture might use:

```text
Infrastructure as Code
     +
GitOps manifests
     +
etcd backup
     +
application-aware data backups
     +
replicated registry
```

Test:

```text
cluster recreation
application restoration
database restoration
DNS cutover
secret recovery
```

### Senior principle

A Kubernetes cluster should ideally be replaceable.

The hardest data to replace should live in controlled durable systems—not in undocumented manual cluster state.

---

## 100. Production application latency increases from 50 ms to 3 seconds. Walk through your Kubernetes incident response.

### Answer

This is an excellent Senior DevOps interview question.

The key is to troubleshoot systematically.

---

### Step 1 — Establish scope

Determine:

```text
Which application?
Which region?
Which namespace?
All endpoints?
One version?
When did it start?
Recent deployment?
```

---

### Step 2 — Check workload state

```bash
kubectl get pods -n production -o wide
kubectl get deployment -n production
```

Look for:

```text
restarts
Pending
CrashLoopBackOff
NotReady
changed replica count
```

---

### Step 3 — Check application latency/errors

Use:

```text
Prometheus
Grafana
APM/tracing
application logs
```

Compare:

```text
request rate
error rate
latency
dependency latency
```

---

### Step 4 — Check container resources

```bash
kubectl top pods -n production
```

Inspect:

```text
CPU
memory
OOM
CPU throttling
```

---

### Step 5 — Check nodes

```bash
kubectl top nodes
kubectl describe node <node>
```

Look for:

```text
CPU saturation
MemoryPressure
DiskPressure
network problems
```

---

### Step 6 — Check networking

Determine whether:

```text
Pod-to-Pod latency increased
Service networking broken
DNS slow
Ingress/Gateway slow
external LB unhealthy
```

Test from inside the cluster:

```bash
kubectl exec <debug-pod> -- \
  curl http://service:8080/health
```

---

### Step 7 — Check dependencies

Many apparent Kubernetes problems are dependency problems.

Check:

```text
database
cache
Kafka/message queue
external API
DNS
storage
```

For example:

```text
DB latency:
10 ms → 2 seconds

Application latency:
50 ms → 3 seconds
```

Scaling Pods will not fix database saturation.

---

### Step 8 — Check recent changes

Correlate against:

```text
application deployment
ConfigMap/Secret change
Ingress change
NetworkPolicy
HPA update
node rollout
cluster upgrade
CNI change
service mesh change
database deployment
cloud infrastructure event
```

---

### Step 9 — Check autoscaling

```bash
kubectl get hpa
kubectl describe hpa <name>
```

Maybe:

```text
traffic rises
    ↓
HPA tries scaling
    ↓
new Pods Pending
    ↓
cluster capacity unavailable
```

Now inspect node autoscaling.

---

### Step 10 — Mitigate based on evidence

Possible actions:

```text
rollback bad deployment
scale application
restore dependency
increase node capacity
fix HPA
correct CPU limits
remove bad NetworkPolicy
repair DNS
move workload from unhealthy node
```

Do not make ten changes simultaneously.

---

### Step 11 — Verify recovery

Confirm:

```text
latency back to baseline
error rate normal
Pods Ready
queue depth falling
dependency healthy
customer traffic recovered
```

---

### Step 12 — Root-cause analysis

Document:

```text
timeline
impact
trigger
root cause
contributing factors
why monitoring did/didn't detect it
mitigation
permanent correction
preventive action
```

### Strong Senior DevOps interview response

> "I treat Kubernetes incidents as distributed-systems incidents. I start with customer symptoms and recent changes, then trace the request path through ingress, Service networking, Pods, nodes, storage and downstream dependencies. I correlate resource saturation, scheduling, autoscaling and application telemetry before changing configuration. I mitigate customer impact first, preserve evidence, and then implement the permanent fix."

---

# Senior Kubernetes Interview Revision Checklist

Before a Senior DevOps interview, be able to explain these topics without notes.

## Architecture

```text
kube-apiserver
etcd
scheduler
controller manager
kubelet
container runtime
kube-proxy
CRI
CNI
CSI
reconciliation
```

## Workloads

```text
Pod
Deployment
ReplicaSet
StatefulSet
DaemonSet
Job
CronJob
init container
sidecar
Pod lifecycle
```

## Networking

```text
ClusterIP
NodePort
LoadBalancer
Ingress
Gateway API
CoreDNS
CNI
NetworkPolicy
EndpointSlice
service discovery
```

## Storage

```text
PV
PVC
StorageClass
CSI
dynamic provisioning
access modes
StatefulSet storage
snapshots
volume expansion
```

## Scheduling

```text
requests
limits
QoS
taints
tolerations
node affinity
pod affinity
anti-affinity
topology spread
priority
preemption
PDB
```

## Autoscaling

```text
HPA
VPA concepts
Cluster Autoscaler
resource metrics
custom metrics
external metrics
```

## Security

```text
RBAC
ServiceAccounts
Secrets
Pod Security Standards
securityContext
capabilities
seccomp
NetworkPolicy
admission webhooks
audit logs
image security
```

## Deployment

```text
Helm
Kustomize
GitOps
rolling deployment
blue-green
canary
zero-downtime deployment
```

## Operations

```text
cluster upgrades
node draining
version skew
certificate management
etcd backup
disaster recovery
```

## Troubleshooting

```text
Pending
CrashLoopBackOff
OOMKilled
ImagePullBackOff
Node NotReady
DNS failure
Service failure
PVC Pending
DiskPressure
control-plane failure
stuck Terminating
admission webhook outage
```

---

# 40 Kubernetes Commands Worth Memorizing

```bash
# Cluster
kubectl cluster-info
kubectl get nodes
kubectl describe node <node>

# Pods
kubectl get pods -A
kubectl get pods -o wide
kubectl describe pod <pod>
kubectl delete pod <pod>

# Logs
kubectl logs <pod>
kubectl logs -f <pod>
kubectl logs <pod> --previous
kubectl logs <pod> -c <container>

# Exec/debug
kubectl exec -it <pod> -- sh
kubectl debug <pod> -it --image=busybox

# Resources
kubectl top pods
kubectl top nodes

# Deployments
kubectl get deployment
kubectl describe deployment <name>
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl rollout undo deployment/<name>

# Services
kubectl get svc
kubectl describe svc <name>
kubectl get endpointslices

# Networking
kubectl get ingress
kubectl get networkpolicy -A

# Storage
kubectl get pvc
kubectl get pv
kubectl describe pvc <pvc>
kubectl get storageclass

# Events
kubectl get events --sort-by=.lastTimestamp

# Scheduling
kubectl get pods --field-selector=status.phase=Pending
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets
kubectl uncordon <node>

# Security
kubectl auth can-i get pods
kubectl auth can-i '*' '*'
kubectl get role,rolebinding
kubectl get clusterrole,clusterrolebinding

# Autoscaling
kubectl get hpa
kubectl describe hpa <name>

# Configuration
kubectl get configmap
kubectl get secret

# Validation
kubectl diff -f deployment.yaml
kubectl apply --dry-run=server -f deployment.yaml

# API debugging
kubectl get --raw='/readyz?verbose'
```

---

# 20 Production Kubernetes Best Practices

```text
1. Set CPU/memory requests.
2. Set sensible limits based on measurements.
3. Configure readiness probes.
4. Configure liveness probes appropriately.
5. Use startup probes for slow applications.
6. Run multiple replicas for critical stateless applications.
7. Spread replicas across nodes/zones.
8. Use PodDisruptionBudgets appropriately.
9. Run containers as non-root.
10. Drop unnecessary Linux capabilities.
11. Apply Pod Security controls.
12. Use default-deny NetworkPolicies where feasible.
13. Follow least-privilege RBAC.
14. Encrypt sensitive data and protect Secrets.
15. Avoid mutable production image tags.
16. Use GitOps/IaC for reproducibility.
17. Back up etcd/application state.
18. Test disaster recovery.
19. Monitor application and infrastructure layers.
20. Test upgrades before production.
```

---

# 15 Critical Kubernetes Troubleshooting Patterns

## Pod Pending

```text
Scheduler
resources
taints
affinity
PVC
```

## CrashLoopBackOff

```text
logs
previous logs
exit code
config
OOM
probes
```

## ImagePullBackOff

```text
image
tag
registry
credentials
network
```

## OOMKilled

```text
memory limit
actual usage
memory leak
heap configuration
```

## Pod not receiving traffic

```text
readiness
Service selector
EndpointSlice
port/targetPort
```

## Service unreachable

```text
Pod
→ endpoint
→ Service
→ networking
```

## DNS failure

```text
Pod resolv.conf
→ CoreDNS Service
→ CoreDNS Pod
→ upstream DNS
```

## PVC Pending

```text
StorageClass
CSI
quota
topology
access mode
```

## Node NotReady

```text
kubelet
runtime
CNI
disk
memory
network
certificate
```

## DiskPressure

```text
images
logs
ephemeral storage
inode usage
runtime files
```

## Failed rollout

```text
new ReplicaSet
new Pods
readiness
images
resources
configuration
```

## API latency

```text
API server
etcd
webhooks
controllers
request storm
```

## Namespace terminating

```text
finalizers
API resources
controller cleanup
```

## Cluster upgrade failure

```text
version skew
CNI
CSI
webhooks
CRDs
deprecated APIs
```

## Application latency

```text
Application
    +
Pod
    +
Node
    +
Network
    +
Storage
    +
Dependencies
```

---

# 10 Answers That Make You Sound Senior-Level

## 1. Kubernetes is a reconciliation system

Do not describe Kubernetes merely as:

```text
"a tool that runs containers."
```

Explain:

```text
desired state
→ controllers
→ reconciliation
```

---

## 2. A running Pod is not necessarily a healthy Pod

Distinguish:

```text
Running
Ready
Healthy
Serving customer traffic
```

---

## 3. StatefulSet does not make an application highly available

Application HA still requires:

```text
replication
quorum
leader election
data consistency
failover
```

---

## 4. A Service is not necessarily a real process

ClusterIP behavior is generally implemented in the networking/data plane.

---

## 5. Requests and limits solve different problems

```text
Requests → scheduling/capacity
Limits → consumption control
```

---

## 6. Liveness and readiness are not interchangeable

```text
Readiness:
Should I receive traffic?

Liveness:
Should Kubernetes restart me?
```

---

## 7. Kubernetes autoscaling cannot fix every bottleneck

```text
More Pods
≠
more database capacity
```

Always find the constrained resource.

---

## 8. Replicas are not backups

Three database Pods can all reproduce the same accidental deletion.

You still need:

```text
backup
PITR
restore testing
```

---

## 9. High availability depends on failure-domain separation

Three replicas on one node provide poor node-level HA.

Prefer appropriate distribution across:

```text
nodes
zones
regions where required
```

---

## 10. Kubernetes incidents are distributed-systems incidents

The correct investigation path is usually:

```text
User request
      ↓
DNS
      ↓
Load balancer
      ↓
Ingress/Gateway
      ↓
Service
      ↓
Pod
      ↓
Node
      ↓
Storage/Network
      ↓
Downstream dependencies
```

A Senior DevOps Engineer should understand and troubleshoot the complete chain rather than repeatedly deleting Pods.

For the interview itself, prioritize **questions 2–10, 20–30, 41–55, 56–70, 77–78, and 86–100**. Those areas are particularly effective at separating someone who can operate Kubernetes production systems from someone who mainly knows YAML and `kubectl`.

[1]: https://kubernetes.io/releases/?utm_source=chatgpt.com "Releases | Kubernetes"
[2]: https://kubernetes.io/docs/concepts/workloads/pods/probes/?utm_source=chatgpt.com "Liveness, Readiness, and Startup Probes | Kubernetes"
[3]: https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/?utm_source=chatgpt.com "Horizontal Pod Autoscaling | Kubernetes"
[4]: https://kubernetes.io/docs/tutorials/security/cluster-level-pss/?utm_source=chatgpt.com "Apply Pod Security Standards at the Cluster Level | Kubernetes"
[5]: https://kubernetes.io/releases/version-skew-policy/?utm_source=chatgpt.com "Version Skew Policy | Kubernetes"
