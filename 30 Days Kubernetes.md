# 🚀 Day 1: Kubernetes Introduction & Architecture

Welcome to Day 1 of your 30-day Kubernetes journey, ! As a Senior DevOps Engineer, you likely have exposure to container orchestration already, so we'll go deep into the *why* and *how* — not just the *what*. Let's build a rock-solid foundation today.

***

## 📚 Part 1: Theory — Kubernetes Architecture Deep Dive

### What is Kubernetes?

Kubernetes (K8s) is an **open-source container orchestration platform** originally developed by Google (based on their internal "Borg" system) and now maintained by the **CNCF (Cloud Native Computing Foundation)**. It automates deployment, scaling, networking, and management of containerized applications.

### High-Level Architecture

A Kubernetes cluster consists of **two main planes**:

```
┌─────────────────────────────────────────────────────────────┐
│                     KUBERNETES CLUSTER                       │
│                                                              │
│  ┌──────────────────────┐      ┌──────────────────────────┐ │
│  │   CONTROL PLANE      │      │      WORKER NODES        │ │
│  │   (Master Node)      │◄────►│   (where pods run)       │ │
│  │                      │      │                          │ │
│  │  • API Server        │      │  • kubelet               │ │
│  │  • etcd              │      │  • kube-proxy            │ │
│  │  • Scheduler         │      │  • Container Runtime     │ │
│  │  • Controller Mgr    │      │  • Pods                  │ │
│  │  • Cloud Controller  │      │                          │ │
│  └──────────────────────┘      └──────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

***

### 🧠 Control Plane Components (The "Brain")

The control plane makes **global decisions** about the cluster (scheduling, detecting/responding to events).

#### 1. **kube-apiserver** 🚪

* **Role**: The **front door** to the cluster. Exposes the Kubernetes REST API.
* **Function**: Every component (kubectl, kubelet, controllers) talks **only** through the API server. It validates and processes REST requests, then updates the corresponding objects in etcd.
* **Key fact**: It's **stateless** and horizontally scalable.
* **Real-world analogy**: Like an Azure Resource Manager (ARM) endpoint — all requests funnel through it.

#### 2. **etcd** 🗄️

* **Role**: **Distributed key-value store** — the single source of truth for cluster state.
* **Function**: Stores all configuration data, cluster state, secrets, service discovery info.
* **Key fact**: Uses **Raft consensus algorithm** for consistency; backups are CRITICAL.
* **Production tip**: Always run etcd in an **odd number of nodes** (3, 5, 7) for quorum. Back it up regularly using `etcdctl snapshot save`.

#### 3. **kube-scheduler** 📅

* **Role**: Decides **which node** a new pod should run on.
* **Function**: Watches for unscheduled pods and assigns them to nodes based on:
  * Resource requirements (CPU/memory)
  * Hardware/software constraints (node selectors, affinity/anti-affinity)
  * Taints and tolerations
  * Data locality, inter-workload interference
* **Process**: **Filtering** (eligible nodes) → **Scoring** (best fit) → **Binding**.

#### 4. **kube-controller-manager** 🎛️

* **Role**: Runs multiple **controller loops** that regulate cluster state.
* **Key controllers bundled inside**:
  * **Node Controller** — notices when nodes go down.
  * **Replication Controller** — maintains desired pod count.
  * **Endpoints Controller** — joins Services & Pods.
  * **Service Account & Token Controllers** — create default accounts and API tokens.
* **Pattern**: Each runs a **reconciliation loop** — *"desired state vs current state, fix the delta."*

#### 5. **cloud-controller-manager** ☁️ *(optional, cloud-specific)*

* **Role**: Embeds cloud-specific control logic (Azure, AWS, GCP).
* **Function**: Manages load balancers, persistent volumes, and route tables specific to the cloud provider.
* **For Azure**: Integrates with Azure Load Balancer, Azure Disk, Azure Route Tables.

***

### 🔧 Worker Node Components (The "Muscle")

These run on every worker node and are responsible for **running pods**.

#### 1. **kubelet** 👷

* **Role**: The **primary node agent**.
* **Function**: Communicates with the API server, ensures containers described in **PodSpecs** are running and healthy.
* **Key fact**: It does NOT manage containers not created by Kubernetes.

#### 2. **kube-proxy** 🔀

* **Role**: **Network proxy** running on each node.
* **Function**: Maintains network rules (iptables / IPVS) that enable communication to pods from inside or outside the cluster — essentially implements the **Service** concept.

#### 3. **Container Runtime** 🐳

* **Role**: The software that runs containers.
* **Options**: `containerd` (default since K8s 1.24), `CRI-O`, Docker (deprecated as direct runtime).
* **Interface**: K8s talks to runtimes via **CRI (Container Runtime Interface)**.

***

## 🛠️ Part 2: Hands-On — Setup on Windows

### Prerequisites

* Windows 10/11 (64-bit)
* **Hyper-V** OR **Docker Desktop** enabled
* Minimum **4 GB RAM**, **2 CPUs**, **20 GB free disk**
* **Admin** PowerShell access

***

### Step 1: Install `kubectl` (Kubernetes CLI)

Open **PowerShell as Administrator**:

```powershell
# Option A: Using Chocolatey (recommended)
choco install kubernetes-cli -y

# Option B: Manual download
curl.exe -LO "https://dl.k8s.io/release/v1.30.0/bin/windows/amd64/kubectl.exe"
# Move to a folder in PATH, e.g., C:\kube
mkdir C:\kube
Move-Item .\kubectl.exe C:\kube\
$env:Path += ";C:\kube"
:SetEnvironmentVariable("Path", $env:Path, "Machine")

# Verify
kubectl version --client
```

***

### Step 2: Install **Minikube**

```powershell
# Using Chocolatey
choco install minikube -y

# OR manual install
curl.exe -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-windows-amd64.exe
Move-Item .\minikube-windows-amd64.exe C:\kube\minikube.exe

# Verify
minikube version
```

**Start your first Minikube cluster:**

```powershell
# Using Docker driver (recommended if Docker Desktop is installed)
minikube start --driver=docker --cpus=2 --memory=4096

# Using Hyper-V driver
minikube start --driver=hyperv --cpus=2 --memory=4096
```

***

### Step 3: Install **Kind** (Kubernetes IN Docker)

```powershell
# Using Chocolatey
choco install kind -y

# OR manual install
curl.exe -Lo kind.exe https://kind.sigs.k8s.io/dl/v0.23.0/kind-windows-amd64
Move-Item .\kind.exe C:\kube\kind.exe

# Verify
kind version
```

**Create a multi-node Kind cluster** (great for simulating real environments):

```powershell
# Create a config file
@"
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
"@ | Out-File -Encoding ASCII kind-config.yaml

# Create the cluster
kind create cluster --name devops-lab --config kind-config.yaml
```

***

## 💻 Part 3: Essential Day-1 Commands

```powershell
# 1. Check kubectl & cluster version
kubectl version
kubectl version --short

# 2. View cluster endpoints (API server, DNS, etc.)
kubectl cluster-info
kubectl cluster-info dump   # detailed dump

# 3. List all nodes
kubectl get nodes
kubectl get nodes -o wide   # IPs, OS, kernel, container runtime

# 4. Describe a node (deep dive into capacity, conditions, taints)
kubectl describe node <node-name>

# 5. Check what's running in the control plane
kubectl get pods -n kube-system

# 6. Check current context (which cluster you're talking to)
kubectl config current-context
kubectl config get-contexts

# 7. Switch context (e.g., between Minikube and Kind)
kubectl config use-context kind-devops-lab
kubectl config use-context minikube
```

### 🎯 Try This Mini-Exercise

```powershell
# Run a quick smoke test
kubectl run hello-nginx --image=nginx --port=80
kubectl get pods
kubectl describe pod hello-nginx
kubectl delete pod hello-nginx
```

***

## 🎤 Part 4: Interview Prep

### ⭐ Star Question: *"Explain the role of each control plane component."*

**Model answer for a Senior DevOps interview:**

> "The Kubernetes control plane is the **brain of the cluster** and consists of four core components plus an optional cloud controller manager.
>
> The **kube-apiserver** is the central gateway — every interaction, whether from kubectl, internal controllers, or kubelets on worker nodes, goes through it. It's stateless and horizontally scalable, which makes it the ideal entry point for high-availability deployments.
>
> Behind the API server sits **etcd**, a distributed key-value store using the Raft consensus algorithm. It holds the entire cluster state — every Pod, Service, Secret, and ConfigMap definition. In production, we run it on an odd-numbered cluster (typically 3 or 5 nodes) and take regular snapshots, because losing etcd means losing the cluster.
>
> The **kube-scheduler** watches for newly created pods without an assigned node and decides where to place them. It uses a two-phase process — filtering nodes that meet the pod's requirements (resources, taints, affinity rules), then scoring them to pick the optimal fit.
>
> The **kube-controller-manager** runs multiple reconciliation loops — Node Controller, ReplicaSet Controller, Endpoints Controller, and others. Each loop continuously compares the desired state in etcd against the actual cluster state and takes corrective action.
>
> Finally, in cloud environments like Azure, the **cloud-controller-manager** abstracts cloud-specific logic — provisioning Azure Load Balancers when you create a Service of type LoadBalancer, attaching Azure Disks for PersistentVolumes, and managing route tables."

### 🔥 Likely Follow-Up Questions

| # | Question                                                     | Quick Pointer                                                                                                                           |
| - | ------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------- |
| 1 | What happens if etcd goes down?                              | Cluster becomes read-only at best; no new scheduling. Restore from snapshot.                                                            |
| 2 | Can you run the control plane on the same node as workloads? | Yes (Minikube/Kind do this), but in prod, **isolate** with taints.                                                                      |
| 3 | How does kubelet authenticate to the API server?             | Via **TLS certificates** + ServiceAccount tokens; bootstrap via TLS bootstrapping.                                                      |
| 4 | What's the difference between Minikube and Kind?             | Minikube spins up a **VM** with single/multi-node K8s; Kind runs K8s nodes as **Docker containers** — faster and great for CI.          |
| 5 | What replaced Docker as the default runtime, and why?        | **containerd** (since v1.24). Docker shim was removed because it added unnecessary overhead — K8s talks directly to containerd via CRI. |

***

# 🎯 Day 2: Pods & Basic Workloads

Welcome to Day 2, ! Yesterday you built the foundation — today we get hands-on with the **smallest deployable unit in Kubernetes: the Pod**. By the end of today, you'll understand why Kubernetes doesn't deploy containers directly, master multi-container patterns used in production, and confidently answer one of the most common K8s interview questions.

***

## 📚 Part 1: Theory — Pods Deep Dive

### What Exactly is a Pod?

A **Pod** is the **smallest, atomic deployable unit** in Kubernetes. It's a **logical wrapper** around one or more containers that:

* Share the **same network namespace** (same IP, same port space)
* Share **storage volumes**
* Share the **same lifecycle** (created, scheduled, and terminated together)
* Are **always co-located** on the same node

> 💡 **Key insight**: Kubernetes doesn't manage containers directly — it manages **Pods**. Think of a Pod as a "logical host" for one or more tightly-coupled containers.

```
┌─────────────────────────────────────────┐
│              POD (10.244.1.5)           │
│  ┌────────────┐  ┌────────────┐        │
│  │ Container1 │  │ Container2 │        │
│  │ (app)      │  │ (sidecar)  │        │
│  └────────────┘  └────────────┘        │
│  Shared: Network (IP, Ports) + Volumes  │
└─────────────────────────────────────────┘
```

***

### 🔄 Pod Lifecycle — The 5 Phases

| Phase         | Meaning                                                                             |
| ------------- | ----------------------------------------------------------------------------------- |
| **Pending**   | Pod accepted by cluster, but containers not yet running (image pulling, scheduling) |
| **Running**   | Pod bound to a node, all containers created, at least one running                   |
| **Succeeded** | All containers terminated **successfully** (exit 0) and won't restart               |
| **Failed**    | All containers terminated, at least one failed (non-zero exit)                      |
| **Unknown**   | Pod state can't be determined (usually node communication failure)                  |

### Container States Within a Pod

* **Waiting** — pulling image, applying secrets, etc.
* **Running** — executing without issues
* **Terminated** — finished execution (with success or failure)

### Pod Conditions (status flags)

* `PodScheduled` — pod assigned to a node
* `Initialized` — all init containers completed
* `ContainersReady` — all containers ready
* `Ready` — pod can serve traffic

***

### 🧩 Multi-Container Pods

While most pods have **one container**, Kubernetes supports multiple containers in a pod for **tightly-coupled helper processes**. They share network + storage but run as separate processes.

**When to use multi-container pods?**

* Containers must run on the **same host** for performance
* They need to share a **filesystem or localhost network**
* They have **shared lifecycle** requirements

***

### 🚀 Init Containers

**Init containers** are specialized containers that run **before** application containers start. They:

* Run to **completion sequentially** (one after another)
* Must **all succeed** before app containers start
* Are perfect for **setup tasks**

**Common use cases:**

* Waiting for a database or service to be ready
* Cloning a Git repo before app starts
* Setting file permissions, downloading configs
* Running DB migrations or schema setup
* Generating dynamic config files

```yaml
initContainers:
- name: wait-for-db
  image: busybox
  command: ['sh', '-c', 'until nslookup mydb; do echo waiting for db; sleep 2; done']
```

***

### 🤝 Sidecar Pattern

The **sidecar pattern** is the most popular multi-container design. A sidecar container **augments or extends** the main container — without modifying it.

**Real-world sidecar examples:**

| Sidecar Type    | Purpose                             | Example                       |
| --------------- | ----------------------------------- | ----------------------------- |
| **Logging**     | Ship app logs to centralized system | Fluentd, Filebeat, Promtail   |
| **Monitoring**  | Expose metrics                      | Prometheus exporters          |
| **Proxy**       | Service mesh data plane             | Envoy in Istio, Linkerd-proxy |
| **Config Sync** | Pull config from external store     | git-sync, consul-template     |
| **Security**    | Secret rotation, certs              | Vault Agent                   |

> 🆕 **Kubernetes 1.28+** introduced **native sidecar support** via `restartPolicy: Always` on init containers — they now run alongside main containers but with proper lifecycle handling.

### Other Multi-Container Patterns

* **Ambassador** — proxies outbound connections (e.g., a local redis proxy that connects to a sharded cluster)
* **Adapter** — standardizes/normalizes output from the main container (e.g., reformats logs)

***

## 🛠️ Part 2: Hands-On — Deploy Your First Pod

### Step 1: Create a Simple Pod with YAML

Create a file named `nginx-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: web
    env: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
    resources:
      requests:
        memory: "64Mi"
        cpu: "100m"
      limits:
        memory: "128Mi"
        cpu: "250m"
```

**Deploy and verify:**

```powershell
# Apply the YAML
kubectl apply -f nginx-pod.yaml

# Watch it come up
kubectl get pods -w

# Detailed status
kubectl describe pod nginx-pod
```

***

### Step 2: Multi-Container Pod (App + Sidecar)

Create `multi-container-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-with-logger
spec:
  volumes:
  - name: shared-logs
    emptyDir: {}
  containers:
  # Main app container
  - name: nginx
    image: nginx:1.25
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx
  # Sidecar logging container
  - name: log-shipper
    image: busybox
    command: ['sh', '-c', 'tail -f /var/log/nginx/access.log']
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx
```

**Deploy and test:**

```powershell
kubectl apply -f multi-container-pod.yaml

# Generate traffic to nginx
kubectl exec web-with-logger -c nginx -- curl -s localhost

# View sidecar logs (it tails nginx access logs)
kubectl logs web-with-logger -c log-shipper
```

***

### Step 3: Init Container Example

Create `init-container-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-with-init
spec:
  initContainers:
  - name: init-service-check
    image: busybox:1.36
    command: ['sh', '-c', 'echo "Initializing..."; sleep 10; echo "Init complete!"']
  - name: init-config-download
    image: busybox:1.36
    command: ['sh', '-c', 'echo "Downloading config..."; sleep 5']
  containers:
  - name: myapp
    image: nginx:1.25
    ports:
    - containerPort: 80
```

**Observe init sequence:**

```powershell
kubectl apply -f init-container-pod.yaml

# Watch the pod progress through init phases
kubectl get pod myapp-with-init -w

# Check logs of init containers specifically
kubectl logs myapp-with-init -c init-service-check
kubectl logs myapp-with-init -c init-config-download
```

You'll see status progress: `Init:0/2` → `Init:1/2` → `PodInitializing` → `Running`.

***

## 💻 Part 3: Essential Pod Commands

### Creating Pods

```powershell
# Imperative — quick test pod
kubectl run nginx --image=nginx --port=80

# With labels and restart policy
kubectl run debug-pod --image=busybox --restart=Never -- sleep 3600

# Generate YAML without creating (great for learning!)
kubectl run nginx --image=nginx --dry-run=client -o yaml > nginx.yaml

# Declarative (recommended for production)
kubectl apply -f nginx-pod.yaml
```

### Inspecting Pods

```powershell
# List pods
kubectl get pods
kubectl get pods -o wide              # IPs, nodes
kubectl get pods --show-labels         # show all labels
kubectl get pods -l app=web            # filter by label
kubectl get pods -A                    # all namespaces

# Detailed information
kubectl describe pod nginx-pod

# Get pod as YAML/JSON
kubectl get pod nginx-pod -o yaml
kubectl get pod nginx-pod -o jsonpath='{.status.podIP}'
```

### Viewing Logs

```powershell
# Basic logs
kubectl logs nginx-pod

# Specific container in a multi-container pod
kubectl logs web-with-logger -c log-shipper

# Stream logs (tail -f)
kubectl logs -f nginx-pod

# Last 50 lines
kubectl logs --tail=50 nginx-pod

# Logs from previous container instance (after crash)
kubectl logs nginx-pod --previous

# Logs from last 1 hour
kubectl logs --since=1h nginx-pod
```

### Executing Inside Pods

```powershell
# Open interactive shell
kubectl exec -it nginx-pod -- /bin/bash
kubectl exec -it nginx-pod -- sh         # for minimal images

# Run a one-off command
kubectl exec nginx-pod -- ls /etc/nginx
kubectl exec nginx-pod -- curl localhost

# Specific container
kubectl exec -it web-with-logger -c nginx -- /bin/bash
```

### Port Forwarding (Test Without Service)

```powershell
# Forward local port to pod
kubectl port-forward pod/nginx-pod 8080:80

# Now open http://localhost:8080 in browser
```

### Copying Files

```powershell
# Copy file from local to pod
kubectl cp ./config.txt nginx-pod:/tmp/config.txt

# Copy from pod to local
kubectl cp nginx-pod:/var/log/nginx/access.log ./access.log
```

### Deleting Pods

```powershell
kubectl delete pod nginx-pod
kubectl delete -f nginx-pod.yaml
kubectl delete pods --all                # nuclear option in current ns
kubectl delete pod nginx-pod --grace-period=0 --force   # force delete stuck pod
```

***

### 🎯 Mini-Exercise: Debug a Crashing Pod

```powershell
# Deploy a pod that intentionally fails
kubectl run crash-pod --image=busybox --restart=Never -- sh -c "exit 1"

# Investigate
kubectl get pod crash-pod
kubectl describe pod crash-pod        # look at 'Events' section
kubectl logs crash-pod                # see why it crashed
kubectl delete pod crash-pod
```

***

## 🎤 Part 4: Interview Prep

### ⭐ Star Question: *"What's the difference between a Pod and a container?"*

**Model answer for a Senior DevOps interview:**

> "A **container** is a runtime instance of a container image — it's an OS-level virtualization unit with its own filesystem, process tree, and network stack, typically managed by a runtime like containerd or Docker.
>
> A **Pod**, on the other hand, is a **Kubernetes abstraction** — the smallest deployable unit in K8s. It's a wrapper around one or more containers that share the **same network namespace** (same IP and port space, communicate via localhost), **shared storage volumes**, and a **common lifecycle** — they're scheduled, started, and terminated together on the same node.
>
> Kubernetes doesn't schedule containers directly; it schedules Pods. This abstraction exists because some workloads need **tightly-coupled helper processes** — for example, a main application container with a logging sidecar, or an init container that prepares config files before the main app starts.
>
> In summary:
>
> * **Container** = runtime instance of an image (managed by container runtime)
> * **Pod** = Kubernetes scheduling unit, can contain 1+ containers sharing network/storage
>
> In practice, most pods run a single container, but multi-container pods are essential for patterns like sidecars (Istio's Envoy proxy), ambassadors, and adapters."

***

### 🔥 Likely Follow-Up Questions

| #  | Question                                                             | Quick Pointer                                                                                                                                                                                         |
| -- | -------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1  | **Why are most Pods single-container?**                              | Single-container = single responsibility, easier scaling, independent updates. Multi-container only when processes are tightly coupled.                                                               |
| 2  | **How do containers in the same Pod communicate?**                   | Via **localhost** (shared network namespace) — no service discovery needed. Also via **shared volumes** for file-based IPC.                                                                           |
| 3  | **What's the difference between init containers and sidecars?**      | Init containers run to **completion sequentially** before app containers start. Sidecars run **alongside** main containers for their entire lifecycle.                                                |
| 4  | **Can a Pod span multiple nodes?**                                   | **No.** All containers in a Pod are always scheduled on the **same node**.                                                                                                                            |
| 5  | **What happens when a Pod dies?**                                    | A Pod is **ephemeral** — it's not recreated automatically. That's why we use **Deployments/ReplicaSets** to maintain desired replicas.                                                                |
| 6  | **How does Kubernetes know a container is healthy?**                 | Via **liveness, readiness, and startup probes** (we'll cover these in detail on Day 8).                                                                                                               |
| 7  | **What's the difference between `kubectl run` and `kubectl apply`?** | `run` is **imperative** (quick), `apply` is **declarative** (uses YAML, tracks state, idempotent — production standard).                                                                              |
| 8  | **What's the restartPolicy for Pods?**                               | `Always` (default), `OnFailure`, `Never`. Note: ReplicaSets only support `Always`.                                                                                                                    |
| 9  | **How would you debug a Pod stuck in `Pending`?**                    | Check `kubectl describe pod` → Events. Usually: insufficient resources, unschedulable node, image pull errors, PVC not bound.                                                                         |
| 10 | **What's the difference between a static pod and a regular pod?**    | **Static pods** are managed directly by kubelet on a specific node (defined in `/etc/kubernetes/manifests/`), not by the API server. Control plane components in kubeadm clusters run as static pods. |

***

### 💡 Senior-Level Scenario Question

**Q: "Your app needs to read config from a Git repo on startup, expose metrics to Prometheus, and ship logs to Elasticsearch. Design the Pod."**

**Answer structure:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: production-app
spec:
  initContainers:
  - name: git-config-loader      # Init: clone config repo
    image: alpine/git
    command: ['git', 'clone', 'https://repo.git', '/config']
    volumeMounts:
    - {name: config, mountPath: /config}
  containers:
  - name: app                     # Main app
    image: myapp:1.0
    volumeMounts:
    - {name: config, mountPath: /etc/app/config}
    - {name: logs, mountPath: /var/log/app}
  - name: prometheus-exporter     # Sidecar: metrics
    image: prom/node-exporter
    ports:
    - {containerPort: 9100}
  - name: log-shipper             # Sidecar: logs
    image: fluent/fluent-bit
    volumeMounts:
    - {name: logs, mountPath: /var/log/app}
  volumes:
  - name: config
    emptyDir: {}
  - name: logs
    emptyDir: {}
```

**Explain**: *"I used an **init container** for one-time setup (Git clone), then two **sidecars** for cross-cutting concerns — metrics exposure and log shipping — sharing volumes with the main app via `emptyDir`."*

***

# 🚀 Day 3: ReplicaSets & Deployments

Welcome to Day 3, ! Yesterday you learned that Pods are **ephemeral** — they die and don't come back. Today we fix that. **Deployments** are the workhorse of production Kubernetes — every senior DevOps engineer must master them. By the end of today, you'll confidently perform zero-downtime updates, instant rollbacks, and design deployment strategies for real-world scenarios.

***

## 📚 Part 1: Theory — The Controller Hierarchy

### Why Not Deploy Pods Directly?

In production, deploying bare Pods is a **cardinal sin** because:

* ❌ No self-healing (pod dies → gone forever)
* ❌ No scaling (can't run N replicas)
* ❌ No rolling updates
* ❌ No rollback capability
* ❌ No version history

That's why Kubernetes provides **higher-level controllers**: ReplicaSets and Deployments.

***

### 🏗️ The Hierarchy: Deployment → ReplicaSet → Pod

```
┌──────────────────────────────────────────────────────┐
│                   DEPLOYMENT                          │
│  (Manages versions, rollouts, rollbacks)             │
│                                                       │
│   ┌──────────────────┐    ┌──────────────────┐      │
│   │  ReplicaSet v1   │    │  ReplicaSet v2   │      │
│   │  (old version)   │    │  (new version)   │      │
│   │                  │    │                  │      │
│   │  Pod  Pod  Pod   │    │  Pod  Pod  Pod   │      │
│   └──────────────────┘    └──────────────────┘      │
└──────────────────────────────────────────────────────┘
```

| Object         | Role                                                              |
| -------------- | ----------------------------------------------------------------- |
| **Pod**        | Smallest deployable unit (Day 2)                                  |
| **ReplicaSet** | Ensures **N identical Pods** are always running                   |
| **Deployment** | Manages **ReplicaSets** — handles versioning, rollouts, rollbacks |

> 💡 **Golden rule**: You manage **Deployments**, Kubernetes manages ReplicaSets, ReplicaSets manage Pods. **Never** create ReplicaSets directly in production.

***

### 🧩 ReplicaSet Deep Dive

A **ReplicaSet (RS)** ensures a specified number of pod replicas are running at any time.

**Key features:**

* Uses a **label selector** to identify which pods it owns
* Continuously **reconciles** desired vs actual replica count
* If a pod dies → RS spins up a replacement automatically
* If excess pods exist → RS terminates them

**Reconciliation loop:**

```
Desired = 3, Actual = 2 → Create 1 pod
Desired = 3, Actual = 4 → Delete 1 pod
Desired = 3, Actual = 3 → Do nothing ✅
```

> 🔑 **ReplicaSet vs ReplicationController**: RS is the newer version with **set-based selectors** (supports `In`, `NotIn`, `Exists`). RC uses only equality-based selectors and is deprecated.

***

### 🎯 Deployment Deep Dive

A **Deployment** is a higher-level abstraction that manages ReplicaSets. It provides:

* ✅ **Declarative updates** for pods and ReplicaSets
* ✅ **Rolling updates** with configurable strategy
* ✅ **Rollback** to any previous revision
* ✅ **Pause & resume** rollouts
* ✅ **Scale up/down** with one command
* ✅ **Revision history** tracking

***

### 🔄 Deployment Strategies

Kubernetes supports **two built-in strategies**:

#### 1. **RollingUpdate** (Default) — Zero Downtime

Gradually replaces old pods with new ones. Controlled by two knobs:

| Parameter        | Default | Meaning                                              |
| ---------------- | ------- | ---------------------------------------------------- |
| `maxUnavailable` | 25%     | Max pods that can be **unavailable** during update   |
| `maxSurge`       | 25%     | Max pods that can be created **above desired count** |

**Example timeline** (3 replicas, maxSurge=1, maxUnavailable=1):

```
Start:    [v1] [v1] [v1]                  (3 old)
Step 1:   [v1] [v1] [v1] [v2]             (3 old + 1 new = surge)
Step 2:   [v1] [v1] [v2] [v2]             (kill 1 old, 2 new)
Step 3:   [v1] [v2] [v2] [v2]             (kill 1 old, 3 new)
Step 4:   [v2] [v2] [v2]                  (kill last old) ✅
```

**Pros**: Zero downtime, gradual rollout, easy rollback
**Cons**: Old + new versions run simultaneously (need backward compatibility)

#### 2. **Recreate** — Downtime Allowed

Kills ALL old pods first, **then** creates new ones.

```
Start:    [v1] [v1] [v1]
Step 1:   [  ] [  ] [  ]    ← downtime begins ⚠️
Step 2:   [v2] [v2] [v2]    ← downtime ends ✅
```

**Pros**: Simpler, no version mixing, useful when versions are incompatible (e.g., DB schema changes)
**Cons**: Causes downtime

***

### 🎨 Advanced Strategies (Not built-in, but interview-worthy)

| Strategy        | How                                                     | Tooling                                |
| --------------- | ------------------------------------------------------- | -------------------------------------- |
| **Blue-Green**  | Run two full environments, switch traffic instantly     | Service selector switch, Argo Rollouts |
| **Canary**      | Send small % traffic to new version, gradually increase | Argo Rollouts, Flagger, Istio          |
| **A/B Testing** | Route based on headers/users (not just %)               | Istio, service mesh                    |
| **Shadow**      | Mirror prod traffic to new version (no user impact)     | Istio mirroring                        |

***

### ⏮️ Rollback Mechanism

Kubernetes maintains a **revision history** (default: 10 revisions, configurable via `revisionHistoryLimit`).

Each rollout creates a **new ReplicaSet** but keeps the old ones (scaled to 0). To rollback, K8s simply scales up the old RS and scales down the current one.

```
Revision 1: ReplicaSet-abc123 (scaled to 0, retained)
Revision 2: ReplicaSet-def456 (scaled to 0, retained)
Revision 3: ReplicaSet-ghi789 (CURRENT — scaled to 3) ✅
```

Rollback to revision 2 = scale up `def456`, scale down `ghi789`.

***

## 🛠️ Part 2: Hands-On — Build, Update, Rollback

### Step 1: Create a Deployment

Create `nginx-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.24
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "250m"
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 3
          periodSeconds: 5
```

**Deploy:**

```powershell
# Apply the deployment
kubectl apply -f nginx-deployment.yaml

# Watch the rollout
kubectl get deployment nginx-deployment -w

# See the hierarchy: Deployment → ReplicaSet → Pods
kubectl get deploy,rs,pods -l app=nginx
```

***

### Step 2: Inspect the Deployment

```powershell
# Detailed view
kubectl describe deployment nginx-deployment

# Check ReplicaSet created by deployment
kubectl get rs -l app=nginx

# Pods created by ReplicaSet
kubectl get pods -l app=nginx -o wide

# View the deployment YAML (cluster's current state)
kubectl get deployment nginx-deployment -o yaml
```

***

### Step 3: Perform a Rolling Update

```powershell
# Method 1: Edit image directly
kubectl set image deployment/nginx-deployment nginx=nginx:1.25 --record

# Method 2: Edit YAML and re-apply
# Change image to nginx:1.25 in YAML, then:
kubectl apply -f nginx-deployment.yaml

# Method 3: Interactive edit
kubectl edit deployment nginx-deployment
```

**Watch the rollout happen in real time:**

```powershell
# Check rollout progress
kubectl rollout status deployment/nginx-deployment

# In another terminal, watch pods being replaced
kubectl get pods -l app=nginx -w

# See OLD and NEW ReplicaSets coexisting
kubectl get rs -l app=nginx
# Notice: old RS scales down, new RS scales up gradually
```

***

### Step 4: Check Rollout History

```powershell
# View revision history
kubectl rollout history deployment/nginx-deployment

# Output:
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         kubectl set image deployment/nginx-deployment nginx=nginx:1.25

# Inspect a specific revision
kubectl rollout history deployment/nginx-deployment --revision=2
```

***

### Step 5: Simulate a Failed Rollout

```powershell
# Push a broken image
kubectl set image deployment/nginx-deployment nginx=nginx:non-existent-tag --record

# Watch it fail
kubectl rollout status deployment/nginx-deployment
# Output: Waiting for rollout to finish... (stuck)

# Check what's happening
kubectl get pods -l app=nginx
# You'll see: ImagePullBackOff or ErrImagePull on new pods

kubectl describe pod <broken-pod-name>
# Events will show: Failed to pull image
```

> 🛡️ **Key behavior**: Because `maxUnavailable=1`, **old pods are NOT killed** until new ones become Ready. So your app stays up! This is K8s self-protecting you.

***

### Step 6: Rollback to Previous Version

```powershell
# Roll back to the previous working revision
kubectl rollout undo deployment/nginx-deployment

# Roll back to a SPECIFIC revision
kubectl rollout undo deployment/nginx-deployment --to-revision=1

# Verify rollback worked
kubectl rollout status deployment/nginx-deployment
kubectl get deployment nginx-deployment -o jsonpath='{.spec.template.spec.containers[0].image}'
```

***

### Step 7: Scaling

```powershell
# Scale up
kubectl scale deployment/nginx-deployment --replicas=5

# Scale down
kubectl scale deployment/nginx-deployment --replicas=2

# Conditional scaling (only if current replicas = 5)
kubectl scale deployment/nginx-deployment --current-replicas=5 --replicas=10

# Autoscale (Horizontal Pod Autoscaler — Day 12 topic)
kubectl autoscale deployment/nginx-deployment --min=2 --max=10 --cpu-percent=80
```

***

### Step 8: Pause & Resume Rollouts

Useful when you want to make **multiple changes** without triggering separate rollouts:

```powershell
# Pause the rollout
kubectl rollout pause deployment/nginx-deployment

# Make multiple changes (won't trigger rollout)
kubectl set image deployment/nginx-deployment nginx=nginx:1.26
kubectl set resources deployment/nginx-deployment -c=nginx --limits=memory=256Mi

# Resume — all changes roll out as ONE update
kubectl rollout resume deployment/nginx-deployment
```

***

### Step 9: Recreate Strategy Example

Create `recreate-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-recreate
spec:
  replicas: 3
  strategy:
    type: Recreate    # ← all pods killed before new ones start
  selector:
    matchLabels:
      app: legacy-app
  template:
    metadata:
      labels:
        app: legacy-app
    spec:
      containers:
      - name: app
        image: nginx:1.24
        ports:
        - containerPort: 80
```

```powershell
kubectl apply -f recreate-deployment.yaml
kubectl set image deployment/app-recreate app=nginx:1.25
# Watch: all 3 old pods terminate FIRST, then 3 new pods start
kubectl get pods -l app=legacy-app -w
```

***

## 💻 Part 3: Essential Commands Cheat Sheet

### Deployment Lifecycle

```powershell
# Create
kubectl create deployment nginx --image=nginx --replicas=3
kubectl apply -f deployment.yaml

# Read
kubectl get deployments
kubectl get deploy -o wide
kubectl describe deployment nginx-deployment

# Update
kubectl set image deployment/nginx-deployment nginx=nginx:1.25
kubectl edit deployment nginx-deployment
kubectl apply -f deployment.yaml

# Delete
kubectl delete deployment nginx-deployment
```

### Rollout Management

```powershell
# Status
kubectl rollout status deployment/nginx-deployment
kubectl rollout status deployment/nginx-deployment --timeout=2m

# History
kubectl rollout history deployment/nginx-deployment
kubectl rollout history deployment/nginx-deployment --revision=3

# Rollback
kubectl rollout undo deployment/nginx-deployment
kubectl rollout undo deployment/nginx-deployment --to-revision=2

# Pause/Resume
kubectl rollout pause deployment/nginx-deployment
kubectl rollout resume deployment/nginx-deployment

# Restart (rolling restart — re-creates all pods)
kubectl rollout restart deployment/nginx-deployment
```

### Scaling

```powershell
# Manual
kubectl scale deployment/nginx-deployment --replicas=5

# Autoscale
kubectl autoscale deployment/nginx-deployment --min=2 --max=10 --cpu-percent=70
```

***

### 🎯 Real-World Exercise: Blue-Green Style with Two Deployments

```powershell
# Deploy "blue" version
kubectl create deployment app-blue --image=nginx:1.24 --replicas=3
kubectl expose deployment app-blue --port=80 --name=app-svc --selector=app=app-blue

# Deploy "green" version (new)
kubectl create deployment app-green --image=nginx:1.25 --replicas=3

# Verify green is healthy
kubectl get pods -l app=app-green

# Switch traffic by changing the Service selector
kubectl patch service app-svc -p '{"spec":{"selector":{"app":"app-green"}}}'

# Verify
kubectl get svc app-svc -o jsonpath='{.spec.selector}'

# If issues, switch back instantly
kubectl patch service app-svc -p '{"spec":{"selector":{"app":"app-blue"}}}'
```

***

## 🎤 Part 4: Interview Prep

### ⭐ Star Question: *"How does Kubernetes handle a failed Deployment rollout?"*

**Model answer for a Senior DevOps interview:**

> "Kubernetes handles failed rollouts through a combination of **rollout strategy controls, readiness probes, and progress deadlines** — all designed to **fail safely** without taking down the application.
>
> When I trigger a rollout — say, updating an image — the Deployment controller creates a **new ReplicaSet** and starts scaling it up while scaling the old one down, respecting `maxSurge` and `maxUnavailable`. Crucially, K8s only marks a new pod 'available' when its **readiness probe passes**. If the new pod never becomes ready — say, due to an `ImagePullBackOff`, `CrashLoopBackOff`, or failed health check — the old ReplicaSet **never scales down** beyond `maxUnavailable`. So the old version keeps serving traffic, and there's **no outage**.
>
> The Deployment also has a `progressDeadlineSeconds` (default 600s). If the rollout doesn't progress within that window, the Deployment is marked with `Progressing=False` and reason `ProgressDeadlineExceeded`. This is observable via `kubectl rollout status`, which exits non-zero — perfect for CI/CD pipelines to detect failure.
>
> For recovery, I use `kubectl rollout undo` to revert to the previous revision. K8s simply scales up the old ReplicaSet and scales down the broken one — instant rollback because the old RS is still there, just at zero replicas.
>
> In production, I integrate this into CI/CD: after `kubectl apply`, I run `kubectl rollout status --timeout=5m`. If it fails, the pipeline triggers `kubectl rollout undo` automatically. For more advanced scenarios — automated canary analysis, traffic shifting — I use **Argo Rollouts** or **Flagger**, which extend Deployments with metric-based promotion and rollback."

***

### 🔥 Likely Follow-Up Questions

| #  | Question                                                                       | Quick Pointer                                                                                                                                                                     |
| -- | ------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1  | **What's the difference between `kubectl rollout restart` and deleting pods?** | `rollout restart` does a **controlled rolling restart** respecting `maxUnavailable`/`maxSurge`. Deleting pods causes all to restart simultaneously, risking downtime.             |
| 2  | **Why use a Deployment instead of a ReplicaSet?**                              | Deployments provide **version control, rollouts, rollbacks**. ReplicaSets only ensure replica count — no update strategy.                                                         |
| 3  | **What happens to old ReplicaSets after an update?**                           | They're **retained** (scaled to 0) per `revisionHistoryLimit` (default 10) for rollback.                                                                                          |
| 4  | **What's `maxSurge=0, maxUnavailable=0`?**                                     | **Invalid** — at least one must be non-zero, otherwise the rollout can't progress.                                                                                                |
| 5  | **How do you do a zero-downtime update with stateful apps?**                   | Use **StatefulSet** (not Deployment) — preserves order, stable identity. We'll cover this on Day 7.                                                                               |
| 6  | **What is `revisionHistoryLimit`?**                                            | Number of old ReplicaSets to keep. Default 10. Set lower (e.g., 3) for storage savings; set higher for audit trails.                                                              |
| 7  | **Can you rollback to a deleted revision?**                                    | **No.** If `revisionHistoryLimit` purged it, it's gone. Use GitOps (Argo CD / Flux) for true version history.                                                                     |
| 8  | **What if my new pods are stuck `Pending`?**                                   | Check `describe pod` for resource shortage, taints, PVC issues. The Deployment will wait indefinitely until `progressDeadlineSeconds` triggers.                                   |
| 9  | **How does Deployment differ from DaemonSet?**                                 | Deployment runs **N replicas** (anywhere); **DaemonSet** runs **one pod per node** (e.g., log shippers, monitoring agents).                                                       |
| 10 | **What's the difference between RollingUpdate and Canary?**                    | RollingUpdate replaces ALL pods gradually with the new version. Canary keeps the old version running and only shifts a **percentage of traffic** to the new one based on metrics. |
| 11 | **How do you achieve a true blue-green deployment in K8s?**                    | Two separate Deployments + one Service. Switch the Service's `selector` to flip traffic instantly. Or use Argo Rollouts.                                                          |
| 12 | **What is `change-cause` in rollout history?**                                 | Annotation `kubernetes.io/change-cause`. Set via `--record` flag or manually for audit trail.                                                                                     |

***

### 💡 Senior-Level Scenario Questions

**Q1: "Your team deployed v2 of an API. After 10 minutes, error rates spiked. How do you handle this in production?"**

> "First, I'd run `kubectl rollout undo deployment/api` for an immediate rollback to v1 — this is faster than redeploying because the old ReplicaSet still exists at zero replicas. Then I'd verify via `kubectl rollout status` and monitor error rates returning to baseline. Post-incident, I'd investigate via `kubectl logs --previous` on a failed pod, check Application Insights/Prometheus metrics, and do a root cause analysis. Going forward, I'd add **canary deployments via Argo Rollouts** with automated rollback based on Prometheus metrics, plus stricter readiness probes."

**Q2: "You have a database schema change that's not backward compatible. Which strategy do you use?"**

> "Definitely **Recreate strategy** (or blue-green with maintenance window). RollingUpdate would cause v1 pods (expecting old schema) to coexist with v2 pods (expecting new schema), causing runtime errors. With Recreate, I scale all old pods to zero, run the migration, then bring up the new version. For zero-downtime alternatives, I'd use the **expand-contract pattern**: deploy a backward-compatible schema first, then deploy app v2, then clean up old schema fields — this allows RollingUpdate."

**Q3: "Your Deployment is stuck in `Progressing` for 30 minutes. How do you debug?"**

> "Step-by-step:
>
> 1. `kubectl rollout status deployment/<name>` — confirm it's stuck.
> 2. `kubectl get rs` — check if new RS is created and how many pods are ready.
> 3. `kubectl get pods -l <selector>` — identify failing pods.
> 4. `kubectl describe pod <pod-name>` — read Events (ImagePullBackOff? Insufficient resources? Failed readiness?).
> 5. `kubectl logs <pod-name>` — application-level errors.
> 6. `kubectl describe deployment <name>` — check `progressDeadlineSeconds` and conditions.
> 7. If broken, `kubectl rollout undo`. If config error, fix YAML and re-apply."

***

# 🚀 Day 3: ReplicaSets & Deployments

Welcome to Day 3,  you learned that Pods are **ephemeral** — they die and don't come back. Today we fix that. **Deployments** are the workhorse of production Kubernetes — every senior DevOps engineer must master them. By the end of today, you'll confidently perform zero-downtime updates, instant rollbacks, and design deployment strategies for real-world scenarios.

***

## 📚 Part 1: Theory — The Controller Hierarchy

### Why Not Deploy Pods Directly?

In production, deploying bare Pods is a **cardinal sin** because:

* ❌ No self-healing (pod dies → gone forever)
* ❌ No scaling (can't run N replicas)
* ❌ No rolling updates
* ❌ No rollback capability
* ❌ No version history

That's why Kubernetes provides **higher-level controllers**: ReplicaSets and Deployments.

***

### 🏗️ The Hierarchy: Deployment → ReplicaSet → Pod

```
┌──────────────────────────────────────────────────────┐
│                   DEPLOYMENT                          │
│  (Manages versions, rollouts, rollbacks)             │
│                                                       │
│   ┌──────────────────┐    ┌──────────────────┐      │
│   │  ReplicaSet v1   │    │  ReplicaSet v2   │      │
│   │  (old version)   │    │  (new version)   │      │
│   │                  │    │                  │      │
│   │  Pod  Pod  Pod   │    │  Pod  Pod  Pod   │      │
│   └──────────────────┘    └──────────────────┘      │
└──────────────────────────────────────────────────────┘
```

| Object         | Role                                                              |
| -------------- | ----------------------------------------------------------------- |
| **Pod**        | Smallest deployable unit (Day 2)                                  |
| **ReplicaSet** | Ensures **N identical Pods** are always running                   |
| **Deployment** | Manages **ReplicaSets** — handles versioning, rollouts, rollbacks |

> 💡 **Golden rule**: You manage **Deployments**, Kubernetes manages ReplicaSets, ReplicaSets manage Pods. **Never** create ReplicaSets directly in production.

***

### 🧩 ReplicaSet Deep Dive

A **ReplicaSet (RS)** ensures a specified number of pod replicas are running at any time.

**Key features:**

* Uses a **label selector** to identify which pods it owns
* Continuously **reconciles** desired vs actual replica count
* If a pod dies → RS spins up a replacement automatically
* If excess pods exist → RS terminates them

**Reconciliation loop:**

```
Desired = 3, Actual = 2 → Create 1 pod
Desired = 3, Actual = 4 → Delete 1 pod
Desired = 3, Actual = 3 → Do nothing ✅
```

> 🔑 **ReplicaSet vs ReplicationController**: RS is the newer version with **set-based selectors** (supports `In`, `NotIn`, `Exists`). RC uses only equality-based selectors and is deprecated.

***

### 🎯 Deployment Deep Dive

A **Deployment** is a higher-level abstraction that manages ReplicaSets. It provides:

* ✅ **Declarative updates** for pods and ReplicaSets
* ✅ **Rolling updates** with configurable strategy
* ✅ **Rollback** to any previous revision
* ✅ **Pause & resume** rollouts
* ✅ **Scale up/down** with one command
* ✅ **Revision history** tracking

***

### 🔄 Deployment Strategies

Kubernetes supports **two built-in strategies**:

#### 1. **RollingUpdate** (Default) — Zero Downtime

Gradually replaces old pods with new ones. Controlled by two knobs:

| Parameter        | Default | Meaning                                              |
| ---------------- | ------- | ---------------------------------------------------- |
| `maxUnavailable` | 25%     | Max pods that can be **unavailable** during update   |
| `maxSurge`       | 25%     | Max pods that can be created **above desired count** |

**Example timeline** (3 replicas, maxSurge=1, maxUnavailable=1):

```
Start:    [v1] [v1] [v1]                  (3 old)
Step 1:   [v1] [v1] [v1] [v2]             (3 old + 1 new = surge)
Step 2:   [v1] [v1] [v2] [v2]             (kill 1 old, 2 new)
Step 3:   [v1] [v2] [v2] [v2]             (kill 1 old, 3 new)
Step 4:   [v2] [v2] [v2]                  (kill last old) ✅
```

**Pros**: Zero downtime, gradual rollout, easy rollback
**Cons**: Old + new versions run simultaneously (need backward compatibility)

#### 2. **Recreate** — Downtime Allowed

Kills ALL old pods first, **then** creates new ones.

```
Start:    [v1] [v1] [v1]
Step 1:   [  ] [  ] [  ]    ← downtime begins ⚠️
Step 2:   [v2] [v2] [v2]    ← downtime ends ✅
```

**Pros**: Simpler, no version mixing, useful when versions are incompatible (e.g., DB schema changes)
**Cons**: Causes downtime

***

### 🎨 Advanced Strategies (Not built-in, but interview-worthy)

| Strategy        | How                                                     | Tooling                                |
| --------------- | ------------------------------------------------------- | -------------------------------------- |
| **Blue-Green**  | Run two full environments, switch traffic instantly     | Service selector switch, Argo Rollouts |
| **Canary**      | Send small % traffic to new version, gradually increase | Argo Rollouts, Flagger, Istio          |
| **A/B Testing** | Route based on headers/users (not just %)               | Istio, service mesh                    |
| **Shadow**      | Mirror prod traffic to new version (no user impact)     | Istio mirroring                        |

***

### ⏮️ Rollback Mechanism

Kubernetes maintains a **revision history** (default: 10 revisions, configurable via `revisionHistoryLimit`).

Each rollout creates a **new ReplicaSet** but keeps the old ones (scaled to 0). To rollback, K8s simply scales up the old RS and scales down the current one.

```
Revision 1: ReplicaSet-abc123 (scaled to 0, retained)
Revision 2: ReplicaSet-def456 (scaled to 0, retained)
Revision 3: ReplicaSet-ghi789 (CURRENT — scaled to 3) ✅
```

Rollback to revision 2 = scale up `def456`, scale down `ghi789`.

***

## 🛠️ Part 2: Hands-On — Build, Update, Rollback

### Step 1: Create a Deployment

Create `nginx-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.24
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "250m"
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 3
          periodSeconds: 5
```

**Deploy:**

```powershell
# Apply the deployment
kubectl apply -f nginx-deployment.yaml

# Watch the rollout
kubectl get deployment nginx-deployment -w

# See the hierarchy: Deployment → ReplicaSet → Pods
kubectl get deploy,rs,pods -l app=nginx
```

***

### Step 2: Inspect the Deployment

```powershell
# Detailed view
kubectl describe deployment nginx-deployment

# Check ReplicaSet created by deployment
kubectl get rs -l app=nginx

# Pods created by ReplicaSet
kubectl get pods -l app=nginx -o wide

# View the deployment YAML (cluster's current state)
kubectl get deployment nginx-deployment -o yaml
```

***

### Step 3: Perform a Rolling Update

```powershell
# Method 1: Edit image directly
kubectl set image deployment/nginx-deployment nginx=nginx:1.25 --record

# Method 2: Edit YAML and re-apply
# Change image to nginx:1.25 in YAML, then:
kubectl apply -f nginx-deployment.yaml

# Method 3: Interactive edit
kubectl edit deployment nginx-deployment
```

**Watch the rollout happen in real time:**

```powershell
# Check rollout progress
kubectl rollout status deployment/nginx-deployment

# In another terminal, watch pods being replaced
kubectl get pods -l app=nginx -w

# See OLD and NEW ReplicaSets coexisting
kubectl get rs -l app=nginx
# Notice: old RS scales down, new RS scales up gradually
```

***

### Step 4: Check Rollout History

```powershell
# View revision history
kubectl rollout history deployment/nginx-deployment

# Output:
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         kubectl set image deployment/nginx-deployment nginx=nginx:1.25

# Inspect a specific revision
kubectl rollout history deployment/nginx-deployment --revision=2
```

***

### Step 5: Simulate a Failed Rollout

```powershell
# Push a broken image
kubectl set image deployment/nginx-deployment nginx=nginx:non-existent-tag --record

# Watch it fail
kubectl rollout status deployment/nginx-deployment
# Output: Waiting for rollout to finish... (stuck)

# Check what's happening
kubectl get pods -l app=nginx
# You'll see: ImagePullBackOff or ErrImagePull on new pods

kubectl describe pod <broken-pod-name>
# Events will show: Failed to pull image
```

> 🛡️ **Key behavior**: Because `maxUnavailable=1`, **old pods are NOT killed** until new ones become Ready. So your app stays up! This is K8s self-protecting you.

***

### Step 6: Rollback to Previous Version

```powershell
# Roll back to the previous working revision
kubectl rollout undo deployment/nginx-deployment

# Roll back to a SPECIFIC revision
kubectl rollout undo deployment/nginx-deployment --to-revision=1

# Verify rollback worked
kubectl rollout status deployment/nginx-deployment
kubectl get deployment nginx-deployment -o jsonpath='{.spec.template.spec.containers[0].image}'
```

***

### Step 7: Scaling

```powershell
# Scale up
kubectl scale deployment/nginx-deployment --replicas=5

# Scale down
kubectl scale deployment/nginx-deployment --replicas=2

# Conditional scaling (only if current replicas = 5)
kubectl scale deployment/nginx-deployment --current-replicas=5 --replicas=10

# Autoscale (Horizontal Pod Autoscaler — Day 12 topic)
kubectl autoscale deployment/nginx-deployment --min=2 --max=10 --cpu-percent=80
```

***

### Step 8: Pause & Resume Rollouts

Useful when you want to make **multiple changes** without triggering separate rollouts:

```powershell
# Pause the rollout
kubectl rollout pause deployment/nginx-deployment

# Make multiple changes (won't trigger rollout)
kubectl set image deployment/nginx-deployment nginx=nginx:1.26
kubectl set resources deployment/nginx-deployment -c=nginx --limits=memory=256Mi

# Resume — all changes roll out as ONE update
kubectl rollout resume deployment/nginx-deployment
```

***

### Step 9: Recreate Strategy Example

Create `recreate-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-recreate
spec:
  replicas: 3
  strategy:
    type: Recreate    # ← all pods killed before new ones start
  selector:
    matchLabels:
      app: legacy-app
  template:
    metadata:
      labels:
        app: legacy-app
    spec:
      containers:
      - name: app
        image: nginx:1.24
        ports:
        - containerPort: 80
```

```powershell
kubectl apply -f recreate-deployment.yaml
kubectl set image deployment/app-recreate app=nginx:1.25
# Watch: all 3 old pods terminate FIRST, then 3 new pods start
kubectl get pods -l app=legacy-app -w
```

***

## 💻 Part 3: Essential Commands Cheat Sheet

### Deployment Lifecycle

```powershell
# Create
kubectl create deployment nginx --image=nginx --replicas=3
kubectl apply -f deployment.yaml

# Read
kubectl get deployments
kubectl get deploy -o wide
kubectl describe deployment nginx-deployment

# Update
kubectl set image deployment/nginx-deployment nginx=nginx:1.25
kubectl edit deployment nginx-deployment
kubectl apply -f deployment.yaml

# Delete
kubectl delete deployment nginx-deployment
```

### Rollout Management

```powershell
# Status
kubectl rollout status deployment/nginx-deployment
kubectl rollout status deployment/nginx-deployment --timeout=2m

# History
kubectl rollout history deployment/nginx-deployment
kubectl rollout history deployment/nginx-deployment --revision=3

# Rollback
kubectl rollout undo deployment/nginx-deployment
kubectl rollout undo deployment/nginx-deployment --to-revision=2

# Pause/Resume
kubectl rollout pause deployment/nginx-deployment
kubectl rollout resume deployment/nginx-deployment

# Restart (rolling restart — re-creates all pods)
kubectl rollout restart deployment/nginx-deployment
```

### Scaling

```powershell
# Manual
kubectl scale deployment/nginx-deployment --replicas=5

# Autoscale
kubectl autoscale deployment/nginx-deployment --min=2 --max=10 --cpu-percent=70
```

***

### 🎯 Real-World Exercise: Blue-Green Style with Two Deployments

```powershell
# Deploy "blue" version
kubectl create deployment app-blue --image=nginx:1.24 --replicas=3
kubectl expose deployment app-blue --port=80 --name=app-svc --selector=app=app-blue

# Deploy "green" version (new)
kubectl create deployment app-green --image=nginx:1.25 --replicas=3

# Verify green is healthy
kubectl get pods -l app=app-green

# Switch traffic by changing the Service selector
kubectl patch service app-svc -p '{"spec":{"selector":{"app":"app-green"}}}'

# Verify
kubectl get svc app-svc -o jsonpath='{.spec.selector}'

# If issues, switch back instantly
kubectl patch service app-svc -p '{"spec":{"selector":{"app":"app-blue"}}}'
```

***

## 🎤 Part 4: Interview Prep

### ⭐ Star Question: *"How does Kubernetes handle a failed Deployment rollout?"*

**Model answer for a Senior DevOps interview:**

> "Kubernetes handles failed rollouts through a combination of **rollout strategy controls, readiness probes, and progress deadlines** — all designed to **fail safely** without taking down the application.
>
> When I trigger a rollout — say, updating an image — the Deployment controller creates a **new ReplicaSet** and starts scaling it up while scaling the old one down, respecting `maxSurge` and `maxUnavailable`. Crucially, K8s only marks a new pod 'available' when its **readiness probe passes**. If the new pod never becomes ready — say, due to an `ImagePullBackOff`, `CrashLoopBackOff`, or failed health check — the old ReplicaSet **never scales down** beyond `maxUnavailable`. So the old version keeps serving traffic, and there's **no outage**.
>
> The Deployment also has a `progressDeadlineSeconds` (default 600s). If the rollout doesn't progress within that window, the Deployment is marked with `Progressing=False` and reason `ProgressDeadlineExceeded`. This is observable via `kubectl rollout status`, which exits non-zero — perfect for CI/CD pipelines to detect failure.
>
> For recovery, I use `kubectl rollout undo` to revert to the previous revision. K8s simply scales up the old ReplicaSet and scales down the broken one — instant rollback because the old RS is still there, just at zero replicas.
>
> In production, I integrate this into CI/CD: after `kubectl apply`, I run `kubectl rollout status --timeout=5m`. If it fails, the pipeline triggers `kubectl rollout undo` automatically. For more advanced scenarios — automated canary analysis, traffic shifting — I use **Argo Rollouts** or **Flagger**, which extend Deployments with metric-based promotion and rollback."

***

### 🔥 Likely Follow-Up Questions

| #  | Question                                                                       | Quick Pointer                                                                                                                                                                     |
| -- | ------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1  | **What's the difference between `kubectl rollout restart` and deleting pods?** | `rollout restart` does a **controlled rolling restart** respecting `maxUnavailable`/`maxSurge`. Deleting pods causes all to restart simultaneously, risking downtime.             |
| 2  | **Why use a Deployment instead of a ReplicaSet?**                              | Deployments provide **version control, rollouts, rollbacks**. ReplicaSets only ensure replica count — no update strategy.                                                         |
| 3  | **What happens to old ReplicaSets after an update?**                           | They're **retained** (scaled to 0) per `revisionHistoryLimit` (default 10) for rollback.                                                                                          |
| 4  | **What's `maxSurge=0, maxUnavailable=0`?**                                     | **Invalid** — at least one must be non-zero, otherwise the rollout can't progress.                                                                                                |
| 5  | **How do you do a zero-downtime update with stateful apps?**                   | Use **StatefulSet** (not Deployment) — preserves order, stable identity. We'll cover this on Day 7.                                                                               |
| 6  | **What is `revisionHistoryLimit`?**                                            | Number of old ReplicaSets to keep. Default 10. Set lower (e.g., 3) for storage savings; set higher for audit trails.                                                              |
| 7  | **Can you rollback to a deleted revision?**                                    | **No.** If `revisionHistoryLimit` purged it, it's gone. Use GitOps (Argo CD / Flux) for true version history.                                                                     |
| 8  | **What if my new pods are stuck `Pending`?**                                   | Check `describe pod` for resource shortage, taints, PVC issues. The Deployment will wait indefinitely until `progressDeadlineSeconds` triggers.                                   |
| 9  | **How does Deployment differ from DaemonSet?**                                 | Deployment runs **N replicas** (anywhere); **DaemonSet** runs **one pod per node** (e.g., log shippers, monitoring agents).                                                       |
| 10 | **What's the difference between RollingUpdate and Canary?**                    | RollingUpdate replaces ALL pods gradually with the new version. Canary keeps the old version running and only shifts a **percentage of traffic** to the new one based on metrics. |
| 11 | **How do you achieve a true blue-green deployment in K8s?**                    | Two separate Deployments + one Service. Switch the Service's `selector` to flip traffic instantly. Or use Argo Rollouts.                                                          |
| 12 | **What is `change-cause` in rollout history?**                                 | Annotation `kubernetes.io/change-cause`. Set via `--record` flag or manually for audit trail.                                                                                     |

***

### 💡 Senior-Level Scenario Questions

**Q1: "Your team deployed v2 of an API. After 10 minutes, error rates spiked. How do you handle this in production?"**

> "First, I'd run `kubectl rollout undo deployment/api` for an immediate rollback to v1 — this is faster than redeploying because the old ReplicaSet still exists at zero replicas. Then I'd verify via `kubectl rollout status` and monitor error rates returning to baseline. Post-incident, I'd investigate via `kubectl logs --previous` on a failed pod, check Application Insights/Prometheus metrics, and do a root cause analysis. Going forward, I'd add **canary deployments via Argo Rollouts** with automated rollback based on Prometheus metrics, plus stricter readiness probes."

**Q2: "You have a database schema change that's not backward compatible. Which strategy do you use?"**

> "Definitely **Recreate strategy** (or blue-green with maintenance window). RollingUpdate would cause v1 pods (expecting old schema) to coexist with v2 pods (expecting new schema), causing runtime errors. With Recreate, I scale all old pods to zero, run the migration, then bring up the new version. For zero-downtime alternatives, I'd use the **expand-contract pattern**: deploy a backward-compatible schema first, then deploy app v2, then clean up old schema fields — this allows RollingUpdate."

**Q3: "Your Deployment is stuck in `Progressing` for 30 minutes. How do you debug?"**

> "Step-by-step:
>
> 1. `kubectl rollout status deployment/<name>` — confirm it's stuck.
> 2. `kubectl get rs` — check if new RS is created and how many pods are ready.
> 3. `kubectl get pods -l <selector>` — identify failing pods.
> 4. `kubectl describe pod <pod-name>` — read Events (ImagePullBackOff? Insufficient resources? Failed readiness?).
> 5. `kubectl logs <pod-name>` — application-level errors.
> 6. `kubectl describe deployment <name>` — check `progressDeadlineSeconds` and conditions.
> 7. If broken, `kubectl rollout undo`. If config error, fix YAML and re-apply."

***


# 🔐 Day 5: ConfigMaps & Secrets

Welcome to Day 5, Rajesh! You've built workloads, networked them, and exposed them — but here's a hard truth: **hardcoding config in your container images is one of the biggest anti-patterns in DevOps**. Today we'll fix that with **ConfigMaps** (non-sensitive config) and **Secrets** (sensitive data), and you'll learn how to securely manage credentials in production — a topic that comes up in *every* Senior DevOps interview.

***

## 📚 Part 1: Theory — Externalizing Configuration

### The Problem: Config Baked into Images

```
❌ ANTI-PATTERN:
Docker Image
├── /app
├── /config
│   ├── prod-db-password.txt   ← embedded in image!
│   └── app-settings.json       ← changes need rebuild!
```

**Consequences:**

* 🔥 Rebuild image for every config change
* 🔥 Same image can't run in dev/test/prod
* 🔥 Secrets leaked into registries
* 🔥 Violates **12-Factor App** principle (Factor III: Config in environment)

### The Solution: External Configuration

```
✅ BEST PRACTICE:
Docker Image (immutable, env-agnostic)
       │
       ├──> ConfigMap (dev/test/prod)   ← non-sensitive
       └──> Secret (dev/test/prod)      ← sensitive
```

**One image** → many environments. Config injected at runtime via **ConfigMaps** and **Secrets**.

***

### 📋 ConfigMap Deep Dive

A **ConfigMap** stores **non-sensitive configuration data** as **key-value pairs** or **files**.

**Use cases:**

* App settings (log level, feature flags)
* Environment-specific URLs (API endpoints)
* Configuration files (nginx.conf, application.properties)
* Command-line arguments

**Limits:**

* Max size: **1 MiB** per ConfigMap (etcd constraint)
* Not encrypted at rest by default
* Namespace-scoped

**Data types stored:**

* `data` — UTF-8 string values
* `binaryData` — base64-encoded binary content

***

### 🔒 Secret Deep Dive

A **Secret** stores **sensitive data** like passwords, tokens, certificates, SSH keys.

**Key fact**: Secrets are **base64-encoded, NOT encrypted**! 🚨

```
"password123" → base64 → "cGFzc3dvcmQxMjM="
```

> ⚠️ **Critical interview point**: Base64 is **encoding**, not **encryption**. Anyone with `kubectl get secret -o yaml` access can decode it instantly: `echo "cGFzc3dvcmQxMjM=" | base64 -d`.

### Built-in Secret Types

| Type                                  | Use Case                            |
| ------------------------------------- | ----------------------------------- |
| `Opaque`                              | Generic user-defined data (default) |
| `kubernetes.io/service-account-token` | Auto-created for service accounts   |
| `kubernetes.io/dockerconfigjson`      | Docker registry credentials         |
| `kubernetes.io/tls`                   | TLS cert + private key              |
| `kubernetes.io/basic-auth`            | Username + password                 |
| `kubernetes.io/ssh-auth`              | SSH private key                     |

***

### 🎯 Mounting: Env Vars vs Volumes

You can consume ConfigMaps/Secrets in **three ways**:

#### Option 1: Environment Variables

```yaml
env:
- name: LOG_LEVEL
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: log_level
```

**Pros**: Simple, familiar pattern
**Cons**: ⚠️ Not updated automatically when ConfigMap changes — pod must restart!

#### Option 2: Volume Mounts (Files)

```yaml
volumeMounts:
- name: config-volume
  mountPath: /etc/app/config
volumes:
- name: config-volume
  configMap:
    name: app-config
```

**Pros**:

* ✅ **Auto-updates** (within \~60s by default) without pod restart!
* ✅ Best for config **files** (nginx.conf, application.yml)
* ✅ Better for large configs

**Cons**: App must re-read files (or use file watchers)

#### Option 3: envFrom (Bulk Import)

```yaml
envFrom:
- configMapRef:
    name: app-config
- secretRef:
    name: db-credentials
```

Injects **all keys** as environment variables. Convenient but less explicit.

***

### 📊 Env Var vs Volume — When to Use What

| Aspect                 | Env Variables                    | Volume Mounts           |
| ---------------------- | -------------------------------- | ----------------------- |
| **Auto-update**        | ❌ Pod restart needed             | ✅ \~60s, no restart     |
| **Best for**           | Simple key-values                | Config files, certs     |
| **Security (Secrets)** | ⚠️ Visible in `kubectl describe` | ✅ Less exposure         |
| **Multi-line content** | ❌ Awkward                        | ✅ Natural               |
| **Atomicity**          | ❌ One-by-one                     | ✅ Symlink swap (atomic) |
| **App reload**         | Pod restart                      | App must watch files    |

> 💡 **Rule of thumb**: Use **volumes for files & certs**, **env vars for simple flags & small values**, and prefer **volumes for production secrets**.

***

### 🛡️ Secret Storage Reality

By default in Kubernetes:

* Secrets are stored **base64-encoded** in **etcd**
* If etcd disk is compromised → secrets exposed
* **Encryption at rest** must be **enabled explicitly** (EncryptionConfiguration)
* **In-transit** is protected via TLS (between API server & etcd)

### Secret Encryption at Rest

Cluster admins should configure `EncryptionConfiguration`:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources: ["secrets"]
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-32-byte-key>
  - identity: {}     # fallback (no encryption)
```

**Better**: Use **KMS provider** (Azure Key Vault, AWS KMS, HashiCorp Vault) for envelope encryption — keys never touch etcd disk.

***

## 🛠️ Part 2: Hands-On — Real-World App with Config + Secrets

### Scenario

Deploy a Node.js-style web app that needs:

* `LOG_LEVEL`, `APP_ENV`, `API_BASE_URL` → ConfigMap
* `DB_PASSWORD`, `JWT_SECRET` → Secret
* `application.properties` file → ConfigMap as volume

***

### Step 1: Create a ConfigMap (Imperative)

```powershell
# From literal values
kubectl create configmap app-config `
  --from-literal=LOG_LEVEL=info `
  --from-literal=APP_ENV=production `
  --from-literal=API_BASE_URL=https://api.example.com

# From a file
echo "log.level=info`napp.env=prod`napp.name=myapp" > application.properties
kubectl create configmap app-properties --from-file=application.properties

# From a directory (all files become keys)
kubectl create configmap app-configs --from-file=./config-dir/

# Verify
kubectl get configmap
kubectl describe configmap app-config
kubectl get configmap app-config -o yaml
```

***

### Step 2: Create a ConfigMap (Declarative — Recommended)

Create `app-configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  # Simple key-value pairs
  LOG_LEVEL: "info"
  APP_ENV: "production"
  API_BASE_URL: "https://api.example.com"
  MAX_CONNECTIONS: "100"
  
  # Multi-line file content
  application.properties: |
    log.level=info
    app.env=production
    app.name=myapp
    db.pool.size=20
  
  nginx.conf: |
    server {
      listen 80;
      server_name myapp.local;
      location / {
        proxy_pass http://backend:3000;
      }
    }
```

```powershell
kubectl apply -f app-configmap.yaml
kubectl get cm app-config -o yaml
```

***

### Step 3: Create a Secret (Imperative)

```powershell
# Generic secret from literals
kubectl create secret generic db-credentials `
  --from-literal=DB_USER=admin `
  --from-literal=DB_PASSWORD='S3cur3P@ssw0rd!' `
  --from-literal=JWT_SECRET='my-super-secret-jwt-key'

# From file
echo "S3cur3P@ssw0rd!" > db_password.txt
kubectl create secret generic db-pwd --from-file=db_password.txt

# Docker registry credentials
kubectl create secret docker-registry regcred `
  --docker-server=myregistry.azurecr.io `
  --docker-username=myuser `
  --docker-password=mypassword `
  --docker-email=me@example.com

# TLS certificate
kubectl create secret tls tls-secret `
  --cert=path/to/tls.crt `
  --key=path/to/tls.key

# Verify (only metadata shown — values masked)
kubectl get secret
kubectl describe secret db-credentials
```

***

### Step 4: Create a Secret (Declarative)

Create `db-secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  # Base64-encoded values
  DB_USER: YWRtaW4=                                    # admin
  DB_PASSWORD: UzNjdXIzUEBzc3cwcmQh                   # S3cur3P@ssw0rd!
  JWT_SECRET: bXktc3VwZXItc2VjcmV0LWp3dC1rZXk=        # my-super-secret-jwt-key

# Alternative: use stringData (auto-encoded, easier to read)
# stringData:
#   DB_USER: admin
#   DB_PASSWORD: S3cur3P@ssw0rd!
```

**Encode values for Secrets:**

```powershell
# PowerShell
:ToBase64String([Text.Encoding]::UTF8.GetBytes("S3cur3P@ssw0rd!"))

# Bash / Linux
echo -n "S3cur3P@ssw0rd!" | base64

# Decode
echo "UzNjdXIzUEBzc3cwcmQh" | base64 -d
```

```powershell
kubectl apply -f db-secret.yaml
```

> 💡 **`stringData` vs `data`**: Use `stringData` in YAML for plain text — K8s auto-encodes it. Cleaner for humans, perfect for templates.

***

### Step 5: Deploy App Consuming Both

Create `app-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: app
        image: nginx:1.25
        ports:
        - containerPort: 80
        
        # 1️⃣ Env vars from ConfigMap (single key)
        env:
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: LOG_LEVEL
        - name: APP_ENV
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_ENV
        
        # 2️⃣ Env vars from Secret (single key)
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: DB_PASSWORD
        
        # 3️⃣ Bulk import — all keys as env vars
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: db-credentials
        
        # 4️⃣ Mount config files as volumes
        volumeMounts:
        - name: app-config-vol
          mountPath: /etc/app/config
          readOnly: true
        - name: nginx-config
          mountPath: /etc/nginx/conf.d/default.conf
          subPath: nginx.conf
          readOnly: true
        - name: db-creds
          mountPath: /etc/app/secrets
          readOnly: true
      
      volumes:
      # Mount specific keys from ConfigMap
      - name: app-config-vol
        configMap:
          name: app-config
          items:
          - key: application.properties
            path: application.properties
      
      # Mount nginx.conf as a single file
      - name: nginx-config
        configMap:
          name: app-config
          items:
          - key: nginx.conf
            path: nginx.conf
      
      # Mount entire Secret as files
      - name: db-creds
        secret:
          secretName: db-credentials
          defaultMode: 0400    # read-only for owner only
```

```powershell
kubectl apply -f app-deployment.yaml
kubectl get pods -l app=webapp
```

***

### Step 6: Verify Inside the Pod

```powershell
# Get pod name
$pod = kubectl get pod -l app=webapp -o jsonpath='{.items[0].metadata.name}'

# 1. Check env vars
kubectl exec $pod -- env | Select-String "LOG_LEVEL|APP_ENV|DB_PASSWORD"

# 2. Check mounted config files
kubectl exec $pod -- ls /etc/app/config
kubectl exec $pod -- cat /etc/app/config/application.properties

# 3. Check mounted secret files
kubectl exec $pod -- ls /etc/app/secrets
kubectl exec $pod -- cat /etc/app/secrets/DB_PASSWORD

# 4. Check nginx.conf injection
kubectl exec $pod -- cat /etc/nginx/conf.d/default.conf
```

***

### Step 7: Test Hot Reload (Volume Mounts)

```powershell
# Edit the ConfigMap
kubectl edit configmap app-config
# Change LOG_LEVEL: "info" → "debug" and save

# Within ~60 seconds, the mounted file updates automatically!
kubectl exec $pod -- cat /etc/app/config/application.properties

# But env vars DO NOT update — verify:
kubectl exec $pod -- env | Select-String "LOG_LEVEL"
# Still shows old value!

# To force env var refresh, restart the deployment
kubectl rollout restart deployment webapp
```

***

### Step 8: Immutable ConfigMaps & Secrets (Performance)

For configs that **never change** (use new ones for updates):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v1
immutable: true     # ← can't be modified or rolled back to mutable
data:
  LOG_LEVEL: "info"
```

**Benefit**: Reduces kube-apiserver load (no watch needed). Ideal for large clusters.

***

### 🎯 Mini-Exercise: Multi-Environment with Same Image

```powershell
# Same image, different ConfigMaps per environment
kubectl create namespace dev
kubectl create namespace prod

# Dev config
kubectl create configmap app-config -n dev `
  --from-literal=LOG_LEVEL=debug `
  --from-literal=API_BASE_URL=https://dev-api.example.com

# Prod config
kubectl create configmap app-config -n prod `
  --from-literal=LOG_LEVEL=warn `
  --from-literal=API_BASE_URL=https://api.example.com

# Same deployment YAML, different namespace — different behavior!
kubectl apply -f app-deployment.yaml -n dev
kubectl apply -f app-deployment.yaml -n prod
```

***

## 💻 Part 3: Essential Commands Cheat Sheet

### ConfigMap

```powershell
# Create
kubectl create configmap NAME --from-literal=key=value
kubectl create configmap NAME --from-file=path/to/file
kubectl create configmap NAME --from-file=path/to/dir/
kubectl create configmap NAME --from-env-file=.env

# Read
kubectl get configmap
kubectl get cm
kubectl describe cm app-config
kubectl get cm app-config -o yaml
kubectl get cm app-config -o jsonpath='{.data.LOG_LEVEL}'

# Update
kubectl edit cm app-config
kubectl apply -f cm.yaml
kubectl create cm app-config --from-literal=key=newval -o yaml --dry-run=client | kubectl apply -f -

# Delete
kubectl delete cm app-config
```

### Secret

```powershell
# Create
kubectl create secret generic NAME --from-literal=key=value
kubectl create secret generic NAME --from-file=path/to/file
kubectl create secret docker-registry NAME --docker-server=... --docker-username=...
kubectl create secret tls NAME --cert=tls.crt --key=tls.key

# Read
kubectl get secret
kubectl describe secret db-credentials
kubectl get secret db-credentials -o yaml

# Decode a secret value
kubectl get secret db-credentials -o jsonpath='{.data.DB_PASSWORD}' | base64 -d

# Update
kubectl edit secret db-credentials
kubectl apply -f secret.yaml

# Delete
kubectl delete secret db-credentials
```

### Useful Tricks

```powershell
# Create ConfigMap/Secret from .env file
kubectl create cm app-env --from-env-file=.env

# Compare two ConfigMaps across namespaces
diff <(kubectl get cm app-config -n dev -o yaml) <(kubectl get cm app-config -n prod -o yaml)

# Bulk decode secret
kubectl get secret db-credentials -o json | jq '.data | map_values(@base64d)'
```

***

## 🎤 Part 4: Interview Prep

### ⭐ Star Question: *"How do you securely manage secrets in Kubernetes?"*

**Model answer for a Senior DevOps interview:**

> "Managing secrets securely in Kubernetes is a layered challenge — it's not just about creating a Secret object, because **Secrets in K8s are base64-encoded, not encrypted by default**. So my strategy in production combines **multiple controls**.
>
> **First, encryption at rest**: I enable `EncryptionConfiguration` at the API server level so etcd doesn't store secrets in plaintext. In managed services like **AKS**, I configure the **KMS provider integration with Azure Key Vault** — this gives envelope encryption where the data encryption key is itself encrypted by a KMS key that never leaves Key Vault.
>
> **Second, externalize secrets** — I avoid storing production secrets in K8s entirely when possible. I use **CSI Secret Store Driver** with the **Azure Key Vault provider** to mount secrets directly from Key Vault as a volume. The secret never persists in etcd; it's pulled at pod startup. This is the gold standard for Azure-based clusters.
>
> Alternatives include **HashiCorp Vault** with the Vault Agent Injector (mutating webhook injects a sidecar that fetches secrets dynamically) or **External Secrets Operator** (ESO), which syncs from external stores like Key Vault, AWS Secrets Manager, or 1Password into native K8s Secrets — useful for legacy apps that expect env-var-based secrets.
>
> **Third, RBAC and least privilege**: I lock down `get`, `list`, `watch` on Secrets via Role/RoleBinding. Service accounts should only access secrets they need. I audit access via the K8s audit log.
>
> **Fourth, no secrets in Git**. Never commit raw YAML with secrets. For GitOps workflows, I use **Sealed Secrets** (Bitnami) — the controller decrypts them in-cluster — or **SOPS with Mozilla SOPS + age/PGP**, encrypting only the secret values in YAML. This makes secrets safe in Git and decryptable only by the cluster.
>
> **Fifth, rotation and audit**: I enforce regular rotation (every 30/60/90 days) via automation — Azure Key Vault has built-in rotation policies. The CSI driver auto-syncs new versions. I also enable Kubernetes audit logs to detect unauthorized secret access.
>
> **Sixth, runtime hardening**: I avoid env vars for sensitive secrets — they show up in `kubectl describe`, process listings, crash dumps. Volume mounts with `defaultMode: 0400` reduce surface area. I also use **Pod Security Standards (restricted)** to prevent privileged containers from accessing host paths.
>
> So in short: **Encrypt at rest → externalize via CSI / Vault → enforce RBAC → never store in Git → rotate continuously → mount as files, not env vars**."

***

### 🔥 Likely Follow-Up Questions

| #  | Question                                                              | Quick Pointer                                                                                                                                                        |
| -- | --------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1  | **Are Secrets encrypted in Kubernetes?**                              | **No, only base64-encoded by default**. Must enable `EncryptionConfiguration` for at-rest encryption.                                                                |
| 2  | **What's the difference between `data` and `stringData` in Secrets?** | `data` requires base64 values; `stringData` accepts plain text and K8s encodes it. `stringData` overrides `data` if both present.                                    |
| 3  | **What happens when you update a ConfigMap mounted as a volume?**     | Files auto-update within \~60s (kubelet sync period). Atomic via symlink swap.                                                                                       |
| 4  | **What happens when you update a ConfigMap consumed as env var?**     | **Nothing** — pod must be restarted. Use `kubectl rollout restart` to trigger.                                                                                       |
| 5  | **How do you trigger a pod restart when a ConfigMap changes?**        | Add a hash annotation to deployment template, or use **Reloader** (Stakater) operator, or use **immutable ConfigMaps** with new names per version.                   |
| 6  | **Max size of a ConfigMap/Secret?**                                   | **1 MiB** (etcd limit). For larger data, use PersistentVolume or external storage.                                                                                   |
| 7  | **How does the CSI Secret Store driver work?**                        | CSI volume plugin that mounts secrets from external stores (Azure Key Vault, Vault, AWS SM) as a volume. Optionally syncs to a K8s Secret.                           |
| 8  | **What is Sealed Secrets?**                                           | Bitnami tool. You encrypt a Secret YAML with a public key; only the in-cluster controller (with private key) can decrypt. Safe to commit to Git.                     |
| 9  | **Difference between Sealed Secrets and External Secrets?**           | Sealed Secrets = encrypted YAML in Git, decrypted in-cluster. External Secrets = synced from external secret manager (Key Vault, AWS SM) into K8s Secrets.           |
| 10 | **What's a ServiceAccount token Secret?**                             | Auto-created Secret of type `kubernetes.io/service-account-token` for pods to authenticate to API server. K8s 1.24+ moved to projected volumes (short-lived tokens). |
| 11 | **How do you mount only specific keys from a ConfigMap?**             | Use `items` in volume spec with `key` and `path` mapping.                                                                                                            |
| 12 | **What is `subPath` in volume mount?**                                | Mount a single file from ConfigMap/Secret without overwriting the whole directory. Drawback: no auto-update for subPath.                                             |
| 13 | **What's `defaultMode` for Secret volumes?**                          | File permissions (e.g., `0400` = owner read-only). Critical for sensitive files.                                                                                     |
| 14 | **Can ConfigMaps be shared across namespaces?**                       | No — they're namespace-scoped. Replicate via tools like `kubed` (Bitnami) or copy in CI/CD.                                                                          |
| 15 | **What's an immutable ConfigMap/Secret?**                             | `immutable: true` — can't be modified, only deleted/recreated. Reduces API watches, useful for large clusters.                                                       |

***

### 💡 Senior-Level Scenario Questions

**Q1: "Your dev team accidentally committed a Secret YAML with a real database password to Git. What's your response?"**

> "Immediate steps:
>
> 1. **Rotate the password immediately** in the database — the compromised one is now public.
> 2. Update the secret in the cluster: `kubectl create secret ... --dry-run=client -o yaml | kubectl apply -f -`.
> 3. Trigger a `kubectl rollout restart` on affected deployments to pick up new credentials.
> 4. **Scrub Git history** using `git filter-repo` or BFG Repo-Cleaner, then force-push (coordinate with the team).
> 5. **Notify security & rotate any downstream tokens** that might have been derived.
>
> **Preventive measures going forward**:
>
> * Install **pre-commit hooks** with `gitleaks` or `detect-secrets`.
> * Adopt **Sealed Secrets** or **SOPS** so committed YAML is encrypted.
> * Set up **External Secrets Operator** so secrets come from Key Vault, not Git.
> * Enable GitHub Advanced Security secret scanning.
> * Conduct a team workshop on secret hygiene."

**Q2: "How would you implement zero-downtime secret rotation for a database password?"**

> "Two strategies:
>
> **Approach A (App supports dual-secret reload):**
>
> 1. Generate new DB password while keeping the old one valid.
> 2. Update the K8s Secret with the new password.
> 3. App watches the mounted file (or use Reloader to auto-restart pods).
> 4. After all pods are on the new password, revoke the old one.
>
> **Approach B (CSI + Key Vault — recommended for AKS):**
>
> 1. Update password in Azure Key Vault — version increments automatically.
> 2. CSI driver syncs the new version to the pod volume within polling interval.
> 3. App reloads from the file watcher OR a sidecar signals SIGHUP.
> 4. No K8s API interaction needed — secret never touches etcd.
>
> For databases that don't support multi-credential overlap, use a **rolling restart** during low-traffic windows. For PostgreSQL/MySQL, use **dual credentials** with `pg_hba.conf` allowing both for the transition window."

**Q3: "How do you prevent a developer from reading production Secrets via kubectl?"**

> "Layered RBAC:
>
> 1. **Namespace isolation** — devs only have access to `dev` namespace, prod is locked.
> 2. **RBAC Role** explicitly excluding `secrets`:
>    ```yaml
>    rules:
>    - apiGroups: [\"\"]
>      resources: [\"pods\", \"deployments\"]
>      verbs: [\"get\", \"list\"]
>    # secrets NOT included
>    ```
> 3. Use **Azure AD integration** with AKS — map AD groups to RoleBindings.
> 4. **Audit logging** to detect any unauthorized attempts.
> 5. For break-glass scenarios, use **just-in-time access** via PIM (Privileged Identity Management).
> 6. Migrate secrets to **Key Vault via CSI driver** — even if someone gets pod access, the secret isn't in etcd; access is governed by Key Vault RBAC + Azure AD."

**Q4: "ConfigMap changes are not reflected in the running pod. What's happening?"**

> "Depends on how it's mounted:
>
> 1. **If env var** — env vars never refresh; pod restart required. Use `kubectl rollout restart deployment/<name>`.
> 2. **If volume mount** — should auto-update in \~60s. If not, check:
>    * `subPath` is used (subPath mounts don't auto-update).
>    * Pod is not using `readOnly: true` filesystem incorrectly.
>    * kubelet sync interval (`--sync-frequency`).
> 3. **App-level cache** — many apps cache config in memory; restart or trigger reload (SIGHUP).
> 4. **Solution for env vars**: deploy **Reloader operator** which annotates deployments and auto-restarts pods on ConfigMap/Secret changes."

***

## 🔐 Bonus: Secret Management Tools Comparison

| Tool                          | Best For               | Pros                              | Cons                                 |
| ----------------------------- | ---------------------- | --------------------------------- | ------------------------------------ |
| **Native K8s Secrets**        | Simple cases           | Built-in, no extra setup          | Base64 only, no rotation             |
| **Sealed Secrets**            | GitOps                 | Safe in Git, simple               | Manual encryption per env            |
| **External Secrets Operator** | Multi-cloud sync       | Pulls from Key Vault/AWS SM/Vault | Adds operator complexity             |
| **CSI Secret Store Driver**   | Cloud-native (AKS/EKS) | Secrets never in etcd             | Slightly higher latency on pod start |
| **HashiCorp Vault**           | Enterprise             | Dynamic secrets, leasing, audit   | Complex setup, ops overhead          |
| **SOPS**                      | Lightweight GitOps     | Multi-key (PGP, age, KMS)         | CLI-based workflow                   |

***

## ✅ Day 5 Checklist

* [ ] Understood why externalizing config matters (12-Factor App)
* [ ] Created ConfigMaps imperatively and declaratively
* [ ] Created Secrets imperatively, declaratively, and with `stringData`
* [ ] Mounted ConfigMaps/Secrets as **env vars** and **volumes**
* [ ] Differentiated when to use env vars vs volumes
* [ ] Tested auto-reload behavior of volume-mounted ConfigMaps
* [ ] Understood that Secrets are **base64, not encrypted** by default
* [ ] Reviewed encryption-at-rest, KMS, Sealed Secrets, External Secrets Operator
* [ ] Practiced multi-environment configs using namespace separation
* [ ] Articulated production-grade secret management strategy

***



