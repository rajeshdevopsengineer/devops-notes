# Kubernetes Interview Questions and Answers (1–25)

DevOps Engineer • Senior SRE • Cloud Architect • Platform Engineer

This guide covers all 25 questions from a production engineering and technical interview perspective, rather than only theoretical definitions.

The explanations include how Kubernetes behaves in real environments such as Amazon EKS, Azure AKS, Google GKE, and self-managed clusters.

## Section 1: Kubernetes Fundamentals and Architecture

### 1. What is Kubernetes, and how is it different from Docker?

Quick answer

Kubernetes is an open-source container orchestration platform that automates application deployment, scaling, scheduling, networking, and recovery.

Docker is primarily a platform and toolset used to build, package, and run containers.

Technical explanation

| Feature                 | Docker                            | Kubernetes                                 |
| ----------------------- | --------------------------------- | ------------------------------------------ |
| Primary role            | Container build and execution     | Container orchestration                    |
| Basic unit              | Container                         | Pod                                        |
| Scheduling across nodes | Not with standalone Docker Engine | Yes                                        |
| Automatic recovery      | Container restart policies        | Pod replacement and reconciliation         |
| Scaling                 | Manual or additional tooling      | Declarative replicas and autoscaling       |
| Networking              | Container networks                | Pod networking, Services, network policies |
| Rolling deployments     | Requires additional tooling       | Native Deployment support                  |
| High availability       | Requires orchestration            | Supports multi-node HA architecture        |

Real-world example

Suppose your e-commerce application has 10 backend replicas.

With standalone Docker, you must arrange container placement, service discovery, traffic distribution, and recovery yourself.

With Kubernetes, you can declare the desired replica count:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ecommerce-api
spec:
  replicas: 10
  selector:
    matchLabels:
      app: ecommerce-api
  template:
    metadata:
      labels:
        app: ecommerce-api
    spec:
      containers:
        - name: api
          image: nginx:1.27
          ports:
            - containerPort: 80
```

Kubernetes works to maintain 10 replicas. If a worker fails, the workload controller can create replacements on healthy nodes, provided sufficient capacity exists.

Interview perspective

Docker and Kubernetes are complementary, not direct replacements. Kubernetes typically runs containers through a Container Runtime Interface (CRI) implementation such as containerd or CRI-O. Docker-built OCI-compatible images can still be used.

### 2. Explain Kubernetes architecture and its components.

Quick answer

A Kubernetes cluster has two major parts:

1. Control plane: makes cluster-wide decisions and maintains desired state.
2. Worker nodes: execute application workloads.

Conceptual Kubernetes architecture. Networking components such as kube-proxy or an eBPF-based replacement are omitted for simplicity.

Technical explanation

The control plane typically contains:

- `kube-apiserver` — Kubernetes API entry point.
- `etcd` — persistent, consistent cluster-state storage.
- `kube-scheduler` — Pod placement.
- `kube-controller-manager` — reconciliation controllers.
- `cloud-controller-manager` — cloud infrastructure integration, when applicable.

Worker-node components include:

- `kubelet` — node agent.
- Container runtime — executes containers.
- `kube-proxy` or an alternative service networking implementation.
- CNI networking components.

Real-world deployment flow

When a developer applies a Deployment manifest:

1\. kubectl submits the Deployment to the API server

2\. API server validates and persists the requested state

3\. Deployment controller creates or updates a ReplicaSet

4\. ReplicaSet controller creates Pods

5\. Scheduler assigns unscheduled Pods to nodes

6\. Kubelet starts containers through the container runtime

7\. Controllers and kubelets continuously reconcile and report state

Production perspective

In managed Kubernetes, such as EKS, AKS, and GKE, the cloud provider manages significant portions of the control plane. The degree of node management depends on the service and operating mode.

For self-managed production clusters, control-plane high availability commonly uses three control-plane nodes, with an appropriate etcd quorum configuration.

Source: [Official Kubernetes component architecture](https://kubernetes.io/docs/concepts/overview/components/).

### 3. What are the control plane components in Kubernetes?

The primary control plane components are:

| Component                | Responsibility                                                                         |
| ------------------------ | -------------------------------------------------------------------------------------- |
| kube-apiserver           | Accepts and processes Kubernetes API requests                                          |
| etcd                     | Stores Kubernetes API data                                                             |
| kube-scheduler           | Assigns unscheduled Pods to suitable nodes                                             |
| kube-controller-manager  | Runs reconciliation controllers                                                        |
| cloud-controller-manager | Integrates cloud-specific node, route and load-balancer functionality where applicable |

How these components interact

For example, if a Deployment requests five replicas but only three exist:

- The relevant controllers detect the discrepancy.
- Two replacement Pods are created.
- The scheduler chooses suitable nodes.
- Kubelets start the containers.
- Status is reported through the API server.

Important interview distinction

The scheduler does not start containers, and the API server does not independently execute workload containers.

The API server exposes the authoritative Kubernetes API; controllers and node agents act on the desired state through that API.

Useful commands

```
kubectl get componentstatuses
```

This legacy command exists but is deprecated and is not recommended for complete control-plane health monitoring.

Prefer component health endpoints where available:

```
kubectl get --raw='/readyz?verbose'
kubectl get --raw='/livez'
```

These check the API server's reported health rather than automatically proving that every independent control-plane component is healthy.

For kubeadm-managed clusters, control-plane components commonly run as static Pods:

```
kubectl get pods -n kube-system
```

### 4. What is the role of the Kubernetes Controller Manager?

Quick answer

`kube-controller-manager` runs control loops that continuously compare actual cluster state with desired state and attempt to correct differences.

The central idea is reconciliation.

For example:

# 5

Desired Pods

# 3

Actual Pods

# +2

Pods to create

Controller detects drift and reconciles

Important controllers include:

- Node controller — monitors node state and handles relevant node failure behavior.
- ReplicaSet controller — maintains the requested number of Pods.
- Deployment controller — manages Deployment rollouts and ReplicaSets.
- Job controller — tracks Jobs and their completion.
- ServiceAccount controller — supports ServiceAccount lifecycle.

Not every Kubernetes controller runs inside `kube-controller-manager`. For example, custom Operator controllers and some cloud-specific controllers run independently.

Production troubleshooting

```
kubectl get deploy -A
kubectl get rs -A
kubectl get pods -A
kubectl describe deployment ecommerce-api
```

If actual state never converges to desired state, investigate controller health, API server connectivity, permissions, admission failures, and scheduling constraints.

Interview perspective

A strong explanation is that Kubernetes is a declarative, eventually reconciled system. The controller is not simply a process that restarts failed containers.

### 5. What is the purpose of the Kubernetes Scheduler?

Quick answer

The `kube-scheduler` selects nodes for Pods that have not yet been assigned to a node.

It makes placement decisions; the kubelet starts the containers.

Technical explanation

Scheduling generally involves two key phases:

1. Filtering: identify nodes that satisfy Pod requirements.
2. Scoring: rank feasible nodes to choose a suitable placement.

The scheduler considers factors such as CPU and memory requests, node selectors, affinity, topology spread constraints, taints, tolerations, and volume topology.

Real-world example

Suppose a Pod requests two CPUs and 4 GiB RAM.

| Node   | Available allocatable capacity relative to requests | Eligible on resources? |
| ------ | --------------------------------------------------- | ---------------------- |
| Node A | 1 CPU, 8 GiB                                        | No                     |
| Node B | 4 CPUs, 8 GiB                                       | Yes                    |
| Node C | 2 CPUs, 2 GiB                                       | No                     |

Node B is the only resource-feasible choice in this simplified example.

Resource feasibility does not guarantee successful execution: runtime memory use, image pulling, disk pressure, or networking may still cause failures.

Implementation example

```
apiVersion: v1
kind: Pod
metadata:
  name: cpu-example
spec:
  containers:
    - name: web
      image: nginx:1.27
      resources:
        requests:
          cpu: "500m"
          memory: "256Mi"
        limits:
          memory: "512Mi"
```

Troubleshooting

```
kubectl get pods -o wide
kubectl describe pod cpu-example
kubectl get nodes
kubectl describe nodes
```

If the Pod remains `Pending`, inspect Pod events for scheduling failures.

Production best practice

Define realistic resource requests and use topology spread constraints or anti-affinity when replicas need fault-domain separation.

### 6. What is the difference between a master node and a worker node?

The modern Kubernetes term is control-plane node rather than master node.

| Control-plane node                      | Worker node                             |
| --------------------------------------- | --------------------------------------- |
| Hosts management components             | Primarily runs application Pods         |
| Executes scheduling and controllers     | Executes scheduled workloads            |
| May host etcd                           | Hosts kubelet and container runtime     |
| Exposes or connects to the cluster API  | Connects to the API server              |
| Often tainted against regular workloads | Generally accepts application workloads |

Real-world example

A self-managed cluster could contain:

- Three control-plane nodes.
- Six worker nodes.
- Worker nodes distributed across three availability zones.

The control-plane nodes manage the cluster, while worker nodes host workloads.

Interview nuance

A control-plane node can also run workloads if scheduling restrictions permit. The roles describe responsibilities, not an absolute technical prohibition on running applications.

### 7. How do master and worker nodes communicate?

Quick answer

The API server is the main communication endpoint between control-plane components and node agents.

Technical explanation

Typical interactions include:

- Kubelet communicates with the API server using authenticated, encrypted HTTPS connections.
- Kubelet watches assigned Pods and reports node/Pod status.
- Scheduler binds Pods to nodes through the Kubernetes API.
- Controllers watch resources and update them through the API.
- The API server can connect to kubelet endpoints for supported operations such as logs and exec, depending on network configuration.

Common network ports:

| Port                | Purpose                                                               |
| ------------------- | --------------------------------------------------------------------- |
| TCP 6443            | Kubernetes API server, commonly                                       |
| TCP 10250           | Kubelet API                                                           |
| TCP 2379–2380       | etcd client/peer communication in typical self-managed configurations |
| UDP/TCP 30000–32767 | Default NodePort range, when used                                     |

These are common defaults, not universal requirements; actual ports and required network paths depend on distribution and configuration.

Validation

```
kubectl get nodes -o wide
kubectl cluster-info
kubectl get --raw='/readyz'
```

On a worker:

```
sudo systemctl status kubelet
sudo journalctl -u kubelet -n 100 --no-pager
```

Common failures

Expired kubelet credentials, DNS failures, routing errors, blocked API traffic, or mismatched node identity can cause nodes to become `NotReady`.

Security best practice

Do not broadly expose kubelet or etcd endpoints to the internet. Use least-privilege credentials, network segmentation, and TLS.

### 8. What is the purpose of kubelet and kube-proxy?

kubelet is the node agent that ensures containers described in Pod specifications are running.

It coordinates with the container runtime, handles health probes, mounts volumes, and reports status.

kube-proxy implements Kubernetes Service forwarding behavior using supported platform networking mechanisms, traditionally including iptables or IPVS, with nftables support in newer Kubernetes releases.

| kubelet                              | kube-proxy                                 |
| ------------------------------------ | ------------------------------------------ |
| Manages Pod execution on its node    | Implements Service traffic forwarding      |
| Works with container runtime         | Programs networking rules                  |
| Executes configured container probes | Routes Service connections to endpoints    |
| Reports node and Pod status          | Maintains Service-related networking state |

Real-world example

If a container crashes, kubelet can restart it according to the Pod's restart policy.

If a Service has three eligible backend Pods, kube-proxy rules can send connections to those endpoints.

Important nuance

Some CNI implementations, such as Cilium in kube-proxy replacement mode, can provide Service networking without running kube-proxy.

Also, kube-proxy is not generally the same as the CNI plugin: the CNI implementation provides Pod networking, while kube-proxy or its replacement implements Service traffic handling.

Troubleshooting commands

```
kubectl get pods -n kube-system -o wide
kubectl get services -A
kubectl get endpointslices -A
```

### 9. What is etcd, and can we manually write data into it?

Quick answer

etcd is a distributed, consistent key-value store used by Kubernetes to persist cluster API state.

Technically, clients with appropriate authorization can write data to etcd, but directly editing Kubernetes-managed etcd keys is strongly discouraged.

Why?

Kubernetes API writes involve validation, authorization, admission control, object lifecycle behavior, and storage transformations.

Direct writes bypass these safeguards and can corrupt or destabilize cluster state.

Production approach

Use Kubernetes APIs:

```
kubectl apply -f deployment.yaml
kubectl patch deployment ecommerce-api \
  --type=merge \
  -p '{"spec":{"replicas":5}}'
```

For self-managed etcd, a properly configured backup can be made using `etcdctl`:

```
ETCDCTL_API=3 etcdctl snapshot save snapshot.db
```

This requires a correctly configured etcd endpoint, trust configuration, and client authentication, which vary by deployment.

For example, common kubeadm environments use TLS client certificates for authenticated snapshot operations.

Production best practices

- Take tested etcd snapshots.
- Protect backup files because they may contain sensitive cluster data.
- Encrypt Kubernetes Secrets at rest where supported.
- Maintain etcd quorum.
- Monitor latency, disk space, and leader changes.
- Do not restore etcd snapshots casually into a running cluster.

Interview perspective

An important answer is that etcd is Kubernetes' backing store, but normal operators must interact with cluster state through the API server rather than directly manipulating etcd.

## Section 2: Pods and Container Lifecycle

### 10. What is a Pod in Kubernetes?

A Pod is the smallest deployable unit Kubernetes creates and manages.

A Pod contains one or more containers that share certain resources, including the Pod's network namespace and configured volumes.

Pod: payment-api-xyz

10.42.1.15

App container

Payment API

Sidecar

Log processor

Shared Pod IP • localhost networking • Shared configured volumes

How it works

The containers in a Pod:

- Can communicate over `localhost`.
- Share the Pod IP.
- Can mount the same volume if configured.
- Are scheduled together on the same node.

They generally maintain separate container filesystems and processes, except when additional namespace sharing is configured.

Production example

A payment API and a supporting proxy sidecar can be deployed in one Pod when their lifecycles are tightly coupled.

Interview nuance

A Pod is ephemeral. When a Pod is replaced, its IP may change. Applications should rely on Services and persistent storage rather than Pod IPs or local container filesystems.

### 11. What is the difference between a Pod and a container?

| Pod                                     | Container                               |
| --------------------------------------- | --------------------------------------- |
| Kubernetes workload unit                | Process execution environment           |
| Contains one or more containers         | Runs an application/process             |
| Has a Pod IP in conventional networking | Shares Pod networking                   |
| Scheduled onto a node                   | Started by a container runtime          |
| Provides shared resource context        | Has its own image and process lifecycle |

Example

A Pod might contain:

- An NGINX application container.
- A proxy sidecar.

The containers are separate execution units, but Kubernetes schedules and manages their shared Pod context.

Interview answer

A container packages and runs application processes. A Pod is the Kubernetes abstraction that groups containers that need to run together.

### 12. How many containers can run inside a single Pod?

A Pod can contain multiple application containers, init containers, and sidecar containers.

There is no simple universal fixed maximum such as five or ten containers per Pod.

Practical limits depend on node resources, runtime behavior, and the impact of Pod management overhead.

Example

```
apiVersion: v1
kind: Pod
metadata:
  name: multi-container
spec:
  containers:
    - name: nginx
      image: nginx:1.27
    - name: helper
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
```

Production perspective

One primary application container per Pod is a useful default, but multiple containers are appropriate for tightly coupled helpers.

Avoid putting unrelated microservices in the same Pod merely to reduce the number of Kubernetes objects.

### 13. Can one container run inside two different Pods?

No. One running container instance belongs to one Pod.

However, multiple Pods can independently create containers from the same image.

For example:

```
Pod A
  └── nginx container (instance A)

Pod B
  └── nginx container (instance B)
```

Both may use `nginx:1.27`, but they are separate container instances.

Interview nuance

A container image is a reusable artifact, not a running container. The same image can be instantiated thousands of times across many Pods.

### 14. Can we schedule Pods on a master node?

Yes. Kubernetes can schedule application Pods on control-plane nodes when the node is eligible and scheduling constraints permit it.

For example, a kubeadm-created control-plane node commonly has a taint similar to:

```
node-role.kubernetes.io/control-plane:NoSchedule
```

To inspect taints:

```
kubectl describe node <control-plane-node>
```

For an appropriate development or single-node environment, an administrator may remove that taint:

```
kubectl taint nodes <control-plane-node> \
  node-role.kubernetes.io/control-plane:NoSchedule-
```

This permits normal scheduling with respect to that taint, but does not guarantee Pod placement. Other taints and scheduling constraints still apply.

Production recommendation

Avoid enabling arbitrary application workloads on dedicated production control-plane nodes unless you have carefully evaluated the resulting resource isolation and security risks.

### 15. Why are application Pods not scheduled on master nodes by default?

In many self-managed distributions, control-plane nodes are protected by `NoSchedule` taints.

This is done to reduce interference from ordinary workloads.

Main reasons

- Resource isolation for critical cluster processes.
- More predictable control-plane latency.
- Reduced risk of application-induced CPU or memory exhaustion.
- Security separation between workloads and cluster management.
- Better operational fault isolation.

Important qualification

This is a common distribution/bootstrap configuration, not a universal Kubernetes rule. Managed environments also differ in whether users can access or schedule their control-plane hosts.

Troubleshooting

```
kubectl describe node <node-name>
kubectl get pod <pod-name> -o yaml
```

Inspect taints, tolerations, selectors, affinity, and resource requirements.

Interview perspective

A taint repels Pods that lack a matching toleration. A toleration permits scheduling past that taint but does not itself force placement onto the node.

## Section 3: Kubernetes Workload Controllers

### 16. What is the difference between a Deployment and a ReplicaSet?

Quick answer

A ReplicaSet maintains a specified number of matching Pods.

A Deployment manages ReplicaSets and provides higher-level application lifecycle capabilities, including declarative rollouts, rollback history, and deployment strategies.

Architecture

Deployment

web-app

ReplicaSet

Desired replicas: 3

Pod 1

Pod 2

Pod 3

| Feature                                       | Deployment             | ReplicaSet                                |
| --------------------------------------------- | ---------------------- | ----------------------------------------- |
| Maintains replica count                       | Through ReplicaSets    | Yes                                       |
| Rolling updates                               | Yes                    | No native rollout management              |
| Rollback history                              | Yes                    | No                                        |
| Manages ReplicaSets                           | Yes                    | No                                        |
| Common use                                    | Stateless applications | Underlying Deployment controller resource |
| Recommended for direct application deployment | Yes                    | Usually no                                |

Real-world example

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: nginx:1.27
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 2
            periodSeconds: 5
```

How rolling updates work

Suppose you have three replicas, `maxSurge: 1`, and `maxUnavailable: 0`.

During a rolling update:

1. The Deployment can temporarily increase the number of Pods to four.
2. Kubernetes starts a Pod using the new template.
3. Once sufficient new Pods become available, it scales down old replicas.
4. This process continues until the rollout reaches the new desired state.

This avoids intentionally reducing the count of available replicas below three during a healthy rollout, but it does not guarantee end-to-end zero downtime.

Production commands

```
kubectl get deployment frontend
kubectl get rs
kubectl rollout status deployment/frontend
kubectl rollout history deployment/frontend
kubectl rollout undo deployment/frontend
```

Common failure scenarios

- New image cannot be pulled.
- Readiness probe keeps failing.
- Insufficient capacity for surge Pods.
- Application startup exceeds expected duration.
- A PodDisruptionBudget or other availability constraint limits voluntary disruptions.

Interview perspective

A Deployment is the correct standard abstraction for a typical stateless microservice. Creating ReplicaSets directly is rarely necessary.

### 17. What is the difference between a Deployment and a StatefulSet?

Quick answer

A Deployment is best suited to workloads whose replicas are interchangeable.

A StatefulSet is designed for workloads that require stable identity, persistent storage association, or ordered operations.

| Feature                  | Deployment                 | StatefulSet                                   |
| ------------------------ | -------------------------- | --------------------------------------------- |
| Pod identities           | Generally replaceable      | Stable ordinal identities                     |
| Pod naming               | Generated names            | Predictable ordinal names                     |
| Storage                  | Can use persistent storage | Supports per-replica volume claim templates   |
| Default startup ordering | No ordinal ordering        | Ordered                                       |
| Rolling updates          | Yes                        | Yes, with StatefulSet semantics               |
| Common workloads         | Web APIs, frontends        | Databases, brokers, clustered storage systems |

Real-world example

An API server usually does not care whether requests go to `api-abc123` or `api-def456`.

A database cluster may need stable member identities:

```
postgres-0
postgres-1
postgres-2
```

A StatefulSet can associate each database Pod with its own PersistentVolumeClaim.

Example StatefulSet

```
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:17
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: password
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

This illustrates Kubernetes StatefulSet mechanics, not a complete production PostgreSQL cluster. It assumes a Secret, suitable storage provisioning, and application configuration. Three standalone PostgreSQL containers do not automatically form a replicated PostgreSQL database.

Why StatefulSets work

Each replica gets a stable ordinal and an associated persistent storage identity. A replacement Pod can reconnect to the corresponding volume rather than receiving an arbitrary new volume.

Production considerations

Use a supported database Operator or managed database for production-critical databases when possible. StatefulSets alone do not implement leader election, replication, backups, fencing, or database failover.

Validation

```
kubectl get statefulsets
kubectl get pods -l app=postgres
kubectl get pvc
kubectl describe statefulset postgres
```

Interview perspective

StatefulSet means stable identity and ordered management. It does not automatically guarantee data consistency, replication, or high availability.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://kubernetes.io\&sz=32)

Kubernetes

+1



### 18. What is the difference between a ReplicaSet and a DaemonSet?

A ReplicaSet maintains a desired number of Pods, while a DaemonSet ensures eligible nodes run a copy of a particular Pod.

| Feature              | ReplicaSet                     | DaemonSet                                |
| -------------------- | ------------------------------ | ---------------------------------------- |
| Desired count        | Explicit replica number        | Based on eligible nodes                  |
| Scheduling intention | Cluster-wide replica count     | Node-level coverage                      |
| Scale mechanism      | Change replica count           | Change eligible node set                 |
| Common usage         | Stateless application replicas | Node monitoring, logging, network agents |

Example

You have five worker nodes.

A ReplicaSet requesting three replicas normally creates three Pods across eligible nodes, subject to scheduling constraints. Kubernetes does not necessarily place them on separate nodes unless configured.

A DaemonSet targeting all five nodes normally aims to place one DaemonSet Pod on each eligible node.

ReplicaSet: 3 replicas

Node 1

Pod

Node 2

Pod

Node 3

Pod

Node 4

—

Node 5

—

DaemonSet: all 5 nodes

Node 1

Pod

Node 2

Pod

Node 3

Pod

Node 4

Pod

Node 5

Pod

Illustrative placement only. Actual eligibility and placement depend on taints, selectors, affinity, and available resources.

Production use case

You need to collect container logs from every worker node.

Instead of manually deploying a log collector to every node, you deploy a DaemonSet.

When a new eligible worker node joins the cluster, the DaemonSet controller automatically arranges a collector Pod for that node.

Troubleshooting commands

```
kubectl get ds -A
kubectl describe ds <daemonset-name> -n <namespace>
kubectl get pods -o wide -n <namespace>
```

Interview perspective

A DaemonSet follows node eligibility, not a fixed application replica count.

### 19. What is the difference between a DaemonSet and a StatefulSet?

Quick answer

DaemonSets provide node-local services. StatefulSets provide stable identity and storage behavior for application replicas.

| Feature            | DaemonSet                 | StatefulSet                           |
| ------------------ | ------------------------- | ------------------------------------- |
| Primary objective  | One Pod per eligible node | Stable application replica identities |
| Replica management | Eligibility-based         | Explicit ordinal replicas             |
| Pod identity       | Node-associated placement | Persistent ordinal identity           |
| Storage            | Depends on application    | Commonly per-Pod PVCs                 |
| Typical workloads  | Logging, monitoring, CNI  | Databases, brokers                    |
| Scaling            | New eligible nodes        | Change replica count                  |

Example

In a three-node Kubernetes cluster:

A logging DaemonSet typically has one collector on each eligible node.

A three-replica StatefulSet has Pods named, for example:

```
database-0
database-1
database-2
```

Those three StatefulSet Pods might reside on separate nodes, but Kubernetes does not guarantee this without appropriate scheduling constraints.

Production nuance

StatefulSets can benefit from Pod anti-affinity, topology spread constraints, and CSI storage topology awareness.

DaemonSets may require specific tolerations and elevated permissions for node-level functionality.

Interview perspective

Choose based on the actual requirement:

- Every eligible node needs the agent → DaemonSet.
- Every application instance needs stable identity → StatefulSet.

### 20. What are the different Kubernetes workload resources?

The most important built-in workload resources are:

| Resource              | Purpose                                | Production example          |
| --------------------- | -------------------------------------- | --------------------------- |
| Pod                   | Executes one or more containers        | Single application instance |
| Deployment            | Manages stateless application rollouts | REST API                    |
| ReplicaSet            | Maintains Pod replica count            | Deployment-managed replicas |
| StatefulSet           | Maintains stateful identities          | Database cluster members    |
| DaemonSet             | Runs workloads on eligible nodes       | Log collection agent        |
| Job                   | Runs tasks toward completion           | One-time migration          |
| CronJob               | Schedules Jobs repeatedly              | Nightly data export         |
| ReplicationController | Legacy replica management              | Older workloads             |

Real-world platform example

An enterprise Kubernetes platform might use:

- Deployment for API microservices.
- StatefulSet for clustered applications requiring stable identity.
- DaemonSet for OpenTelemetry Collector agents.
- Job for a one-time data migration.
- CronJob for a scheduled report export.

CronJob example

```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-report
spec:
  schedule: "0 2 * * *"
  timeZone: "Etc/UTC"
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: report
              image: busybox:1.36
              command:
                - sh
                - -c
                - echo "Generating daily report"
```

The CronJob attempts to run daily at 02:00 UTC.

Production consideration

CronJob execution is not an exactly-once guarantee. Jobs should be designed to tolerate retries, duplication, and missed schedules.

Validation

```
kubectl get deploy,rs,sts,ds,job,cronjob -A
```

Source: [Kubernetes workload management documentation](https://kubernetes.io/docs/concepts/workloads/controllers/).

## Section 4: DaemonSets, Init Containers, and Operators

### 21. What is the purpose of a DaemonSet?

Quick answer

A DaemonSet ensures that a Pod runs on every eligible node, or on a selected group of nodes.

This is particularly useful when the application must operate at node level.

Common production use cases

- Container log collection.
- Node-level observability.
- Container networking agents.
- Storage drivers and supporting components.
- GPU node device plugins.
- Security or runtime monitoring agents.

DaemonSet example

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-agent
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-agent
  template:
    metadata:
      labels:
        app: node-agent
    spec:
      containers:
        - name: node-agent
          image: busybox:1.36
          command:
            - sh
            - -c
            - while true; do echo "Agent running"; sleep 60; done
          resources:
            requests:
              cpu: "25m"
              memory: "32Mi"
            limits:
              memory: "64Mi"
```

This is an illustrative agent rather than a functional production monitoring collector. The `monitoring` namespace must already exist.

Implementation

```
kubectl create namespace monitoring
kubectl apply -f daemonset.yaml
kubectl get ds -n monitoring
kubectl get pods -n monitoring -o wide
```

Production best practices

Set resource requests and limits, control node targeting, and avoid granting host filesystem or privileged access unless required.

For a large cluster, remember that DaemonSet costs scale with node count. A per-node agent consuming 100 MiB of memory across 1,000 nodes has a substantial aggregate resource footprint.

Troubleshooting

If a node does not run the expected agent, check:

```
kubectl describe ds node-agent -n monitoring
kubectl get nodes --show-labels
kubectl describe node <node-name>
```

Look for selector mismatches, missing tolerations, scheduling failures, and resource pressure.

### 22. Which default DaemonSets are present in Kubernetes?

Quick answer

There is no universal fixed set of DaemonSets automatically installed by Kubernetes itself.

The DaemonSets present depend on the Kubernetes distribution, cloud provider, CNI, CSI installation, and enabled platform add-ons.

Common examples:

| DaemonSet       | Typical purpose                    | Always installed? |
| --------------- | ---------------------------------- | ----------------- |
| kube-proxy      | Service networking                 | No                |
| aws-node        | AWS VPC CNI node networking in EKS | No                |
| calico-node     | Calico networking                  | No                |
| cilium          | Cilium networking/data plane       | No                |
| CSI node plugin | Storage integration                | No                |
| Fluent Bit      | Log forwarding                     | No                |
| node-exporter   | Node metrics exporter              | No                |

Important clarification

CoreDNS is commonly deployed as a Deployment, not a DaemonSet.

Control-plane components in kubeadm clusters commonly run as static Pods, not DaemonSets.

Check your cluster

```
kubectl get daemonsets --all-namespaces
```

Example commands to inspect networking-related workloads:

```
kubectl get pods -n kube-system -o wide
kubectl get deployments -n kube-system
kubectl get daemonsets -n kube-system
```

Interview perspective

The correct answer is distribution-dependent. Saying that every Kubernetes cluster must have a kube-proxy, Calico, and Fluent Bit DaemonSet would be inaccurate.

Source: [Kubernetes DaemonSet documentation](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/).

### 23. What is the purpose of an init container?

Quick answer

An init container performs initialization work before ordinary application containers start.

Regular init containers execute sequentially and must complete successfully before the primary application containers can start.

Common use cases

- Generate configuration files.
- Wait for necessary services.
- Perform one-time Pod-level preparation.
- Download or stage application artifacts.
- Check initialization prerequisites.

Lifecycle

Pod initialization

Init container 1

Prepare configuration → Exit 0

Init container 2

Check prerequisites → Exit 0

Application containers start

Technical explanation

Regular init containers:

- Run before application containers.
- Execute in the defined order.
- Must successfully finish to allow application startup.
- Can use shared volumes to provide data to the application.
- Can use separate container images and security contexts.

A failed init container may be retried according to Pod restart behavior.

Troubleshooting

```
kubectl describe pod <pod-name>
kubectl logs <pod-name> -c <init-container-name>
```

For a previously terminated instance:

```
kubectl logs <pod-name> \
  -c <init-container-name> \
  --previous
```

The `--previous` option is only useful if a previous container instance's logs are available.

Production nuance

Do not use an init container to run a non-idempotent database migration in every application replica. Multiple replicas may execute the migration concurrently.

For cluster-wide migration tasks, a dedicated Job with appropriate concurrency and failure handling is generally better.

Also, Kubernetes supports restartable init containers used as native sidecars, whose lifecycle differs from ordinary run-to-completion init containers.

Source:&#x20;

[image](https://www.google.com/s2/favicons?domain=https://kubernetes.io\&sz=32)

Kubernetes

.



### 24. How do you execute a shell script before an application container starts?

Recommended answer: use an init container.

Suppose an application needs a generated configuration file before NGINX starts.

Real-world example

This Pod uses an init container to write `index.html` into a shared `emptyDir` volume. The NGINX container then serves the generated file.

```
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  volumes:
    - name: web-content
      emptyDir: {}

  initContainers:
    - name: prepare-content
      image: busybox:1.36
      command:
        - /bin/sh
        - -c
        - |
          set -eu
          echo "Preparing application content"
          echo "Application initialized" \
            > /work/index.html
      volumeMounts:
        - name: web-content
          mountPath: /work

  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
      volumeMounts:
        - name: web-content
          mountPath: /usr/share/nginx/html
          readOnly: true
```

Implementation steps

Save as `init-demo.yaml`, then run:

```
kubectl apply -f init-demo.yaml

kubectl get pod init-demo

kubectl describe pod init-demo

kubectl logs init-demo -c prepare-content
```

Validate that the init container generated the file:

```
kubectl exec init-demo -c nginx -- \
  cat /usr/share/nginx/html/index.html
```

Why this works

The `emptyDir` volume belongs to the Pod and is shared by the two containers.

The init container writes the file and exits successfully. Kubernetes then starts the NGINX container, which mounts the same volume.

The content persists across container restarts within the Pod's lifetime, but not across replacement of the Pod itself.

Common failure scenarios

| Symptom                  | Possible cause                      | Troubleshooting                     |
| ------------------------ | ----------------------------------- | ----------------------------------- |
| `Init:CrashLoopBackOff`  | Script exits nonzero repeatedly     | Check init container logs           |
| Pod remains initializing | Init container hangs or waits       | Inspect command and dependencies    |
| File missing             | Wrong mount path                    | Compare both volume mounts          |
| Permission denied        | Filesystem ownership or permissions | Check UID/GID and security contexts |
| `ImagePullBackOff`       | Init image unavailable              | Check image and registry access     |

Production best practices

Use timeouts and bounded retries where initialization depends on external systems. Make scripts idempotent, avoid embedding credentials in scripts, and prefer a Secret or external secret integration when credentials are required.

An init container is appropriate for per-Pod initialization; a Job is usually more appropriate for once-per-release operations.

### 25. What is a Kubernetes Operator?

Quick answer

A Kubernetes Operator extends Kubernetes by combining custom resources with a controller that automates the operational lifecycle of an application or system.

It captures application-specific operations in software and runs them through Kubernetes-style reconciliation loops.

Technical explanation

An Operator commonly involves:

1. CustomResourceDefinition (CRD): extends the Kubernetes API with a new resource type.
2. Custom Resource (CR): declares a desired state using that resource type.
3. Controller: watches the resources and reconciles the running environment toward the desired state.

Custom Resource

PostgresCluster: replicas = 3

Operator Controller

Watch → Compare → Reconcile

Managed resources and application

Pods • Services • PVCs • Backups • Replication

Observe status and reconcile again

Real-world example

A production PostgreSQL Operator might automate:

- Provisioning database instances.
- Configuring replication.
- Rotating credentials.
- Scheduling backups.
- Coordinating version upgrades.
- Replacing failed instances.
- Promoting a healthy replica when a primary fails, if supported.

Illustrative Custom Resource

```
apiVersion: database.example.com/v1
kind: PostgresCluster
metadata:
  name: production-db
spec:
  replicas: 3
  storage:
    size: 100Gi
  backup:
    enabled: true
```

This is conceptual YAML, not a resource you can apply to an arbitrary Kubernetes cluster. The API group, schema, and fields must be defined by a specific installed Operator's CRD.

How Operators differ from Helm

| Helm                                                         | Operator                                    |
| ------------------------------------------------------------ | ------------------------------------------- |
| Packages and renders Kubernetes manifests                    | Continuously reconciles application state   |
| Commonly used for installation and upgrades                  | Can automate runtime operations             |
| Does not inherently provide application-specific controllers | Implements application-specific controllers |
| Can install an Operator                                      | Can itself be installed using Helm          |

Production troubleshooting

```
kubectl get crds
kubectl api-resources
kubectl get pods -A
```

For a specific installed Operator, inspect its controller logs and corresponding custom-resource status.

Common failure scenarios

- Missing or incompatible CRDs.
- Insufficient Operator RBAC permissions.
- Invalid custom-resource specifications.
- Reconciliation errors.
- Conflicts between multiple controllers.
- Unsupported upgrade paths.
- Dependence on external systems that are unavailable.

Production best practices

Review Operator permissions and upgrade compatibility before installation. Test disaster recovery and backup restoration. Monitor reconcile errors, controller availability, and custom-resource conditions.

Interview perspective

A Kubernetes Operator is not simply a custom Deployment or a Helm chart. Its distinguishing capability is application-specific, continuous operational automation based on declarative state.

Source: [Official Kubernetes Operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/).

## Section 5: Production Troubleshooting and Interview Preparation

### Essential Kubernetes troubleshooting commands

These commands are worth practicing hands-on because they repeatedly appear in real incidents and senior engineering interviews.

| Requirement               | Command                                                       |
| ------------------------- | ------------------------------------------------------------- |
| Cluster information       | `kubectl cluster-info`                                        |
| Node health               | `kubectl get nodes -o wide`                                   |
| All Pods                  | `kubectl get pods -A -o wide`                                 |
| Pod events and scheduling | `kubectl describe pod <pod>`                                  |
| Application logs          | `kubectl logs <pod> -c <container>`                           |
| Previous container logs   | `kubectl logs <pod> --previous`                               |
| Deployment rollout        | `kubectl rollout status deploy/<name>`                        |
| Deployment history        | `kubectl rollout history deploy/<name>`                       |
| ReplicaSets               | `kubectl get rs -A`                                           |
| StatefulSets              | `kubectl get sts -A`                                          |
| DaemonSets                | `kubectl get ds -A`                                           |
| Persistent storage claims | `kubectl get pvc -A`                                          |
| Recent events             | `kubectl get events -A --sort-by=.metadata.creationTimestamp` |
| Node resource usage       | `kubectl top nodes`                                           |
| Pod resource usage        | `kubectl top pods -A`                                         |

The `kubectl top` commands require a functioning resource metrics API, commonly provided by Metrics Server.

### Production best practices across all 25 topics

For a production cluster, experienced engineering teams typically prioritize:

- Reliability: Multiple failure domains, appropriate replica distribution, PodDisruptionBudgets, tested recovery procedures, and application health probes.
- Security: Least-privilege RBAC, workload identity, network policies, hardened images, secure secret handling, and controlled admission policies.
- Scalability: Resource requests, Horizontal Pod Autoscalers, node autoscaling, and workload-specific placement constraints.
- Observability: Metrics, logs, traces, Kubernetes events, control-plane health, and actionable alerting.
- Cost: Node rightsizing, namespace resource governance, efficient DaemonSets, and a careful mix of compute capacity types.
- Change management: GitOps, reviewed configuration changes, version-tested manifests, and safe rollback procedures.

## Interview practice: five scenario-based follow-ups

Senior DevOps / SRE

Question 1 of 5

A Deployment requires five Pods, but only three are Running. What would you check?

Think through the production incident, then reveal the suggested answer.

Reveal suggested answer

&#x20;PreviousNext&#x20;

### What interviewers expect at different experience levels

| Experience        | Expected depth                                                                                                                                       |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1–3 years         | Define core objects, write basic YAML, use kubectl, troubleshoot Pods                                                                                |
| 3–5 years         | Understand controllers, scheduling, networking, deployments, probes, storage                                                                         |
| 5–8 years         | Debug node failures, production rollouts, networking, autoscaling, workload availability                                                             |
| 8–12+ years       | Explain control-plane HA, multi-AZ design, failure recovery, observability, security, and cost trade-offs                                            |
| Principal / Staff | Justify architectural decisions, evaluate failure modes, design platform guardrails, and connect technical choices to SLOs and business requirements |

Final interview advice: For each question, be prepared to explain not only what Kubernetes does, but also which component is responsible, how you verify its behavior, what happens when it fails, and how you would mitigate the problem in production. That depth is what distinguishes an experienced DevOps/SRE engineer from someone who has only memorized definitions.
