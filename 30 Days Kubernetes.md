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

Welcome to Day 5, ! You've built workloads, networked them, and exposed them — but here's a hard truth: **hardcoding config in your container images is one of the biggest anti-patterns in DevOps**. Today we'll fix that with **ConfigMaps** (non-sensitive config) and **Secrets** (sensitive data), and you'll learn how to securely manage credentials in production — a topic that comes up in *every* Senior DevOps interview.

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

# 🏢 Day 6: Namespaces, Labels & Selectors

Welcome to Day 6, ! So far you've built workloads and exposed them — but in real enterprises, you're not running **one app on one cluster**. You're running **dozens of teams, hundreds of microservices, multiple environments** on shared infrastructure. Today you'll master the **organizational backbone** of Kubernetes: Namespaces (isolation), Labels (grouping), and Selectors (querying). These are the tools that turn chaos into a multi-tenant, governable platform.

***

## 📚 Part 1: Theory — The Organizational Trinity

### Why Do These Three Concepts Exist Together?

```
┌─────────────────────────────────────────────────────────┐
│                  KUBERNETES CLUSTER                       │
│                                                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  NAMESPACE   │  │  NAMESPACE   │  │  NAMESPACE   │  │
│  │   "team-a"   │  │   "team-b"   │  │   "shared"   │  │
│  │              │  │              │  │              │  │
│  │  Pods, SVCs  │  │  Pods, SVCs  │  │  Monitoring  │  │
│  │  ConfigMaps  │  │  ConfigMaps  │  │  Logging     │  │
│  │  Secrets     │  │  Secrets     │  │  Ingress     │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│                                                           │
│  Each resource has LABELS → queried via SELECTORS        │
└──────────────────────────────────────────────────────────┘
```

| Concept       | Purpose                                                                   |
| ------------- | ------------------------------------------------------------------------- |
| **Namespace** | **Isolation boundary** — scope for resources, quotas, RBAC                |
| **Label**     | **Identifying metadata** attached to objects (`app=frontend`, `env=prod`) |
| **Selector**  | **Query language** to filter/group objects by labels                      |

***

### 🏠 Namespaces Deep Dive

A **Namespace** is a virtual cluster within a physical cluster — a way to partition resources.

**What's namespaced vs not?**

| ✅ Namespaced                | ❌ Cluster-Scoped                  |
| --------------------------- | --------------------------------- |
| Pods, Services, Deployments | Nodes                             |
| ConfigMaps, Secrets         | PersistentVolumes                 |
| ReplicaSets, StatefulSets   | StorageClasses                    |
| Roles, RoleBindings         | ClusterRoles, ClusterRoleBindings |
| Ingress, NetworkPolicies    | Namespaces themselves             |
| ServiceAccounts             | CustomResourceDefinitions (CRDs)  |

**Default namespaces in every cluster:**

| Namespace         | Purpose                                                  |
| ----------------- | -------------------------------------------------------- |
| `default`         | Where resources go if you don't specify (avoid in prod!) |
| `kube-system`     | Control plane components (CoreDNS, kube-proxy)           |
| `kube-public`     | Publicly readable, for cluster discovery                 |
| `kube-node-lease` | Node heartbeat lease objects (perf optimization)         |

***

### 🎯 What Namespaces DO Provide

✅ **Logical separation** — same name resources in different namespaces  
✅ **RBAC boundary** — grant access per namespace  
✅ **Resource quotas** — limit CPU, memory, object counts per namespace  
✅ **LimitRanges** — default/min/max per container  
✅ **DNS scoping** — services get `<svc>.<ns>.svc.cluster.local`  
✅ **NetworkPolicy boundary** — easy to write "allow only within ns" policies  
✅ **Cost allocation** — chargeback per team/namespace

### ⚠️ What Namespaces DO NOT Provide

❌ **Strong security isolation** — pods in different namespaces can still talk by default!  
❌ **Network isolation** — requires explicit **NetworkPolicies** (Day 14)  
❌ **Hardware isolation** — pods still share nodes (use **node selectors** + **taints**)  
❌ **Kernel isolation** — same kernel; one bad pod can still affect node

> 🔑 **Senior interview insight**: Namespaces are a **soft boundary**, not a security boundary. For **hard multi-tenancy**, you need separate clusters or **virtual clusters** (vCluster, Capsule, Kiosk).

***

### 🏷️ Labels Deep Dive

A **Label** is a `key=value` pair attached to objects via metadata. It's how Kubernetes **identifies, groups, and connects** objects.

```yaml
metadata:
  labels:
    app: frontend
    tier: web
    env: production
    version: v2.1.0
    team: payments
    owner: 
```

**Label rules:**

* Keys: alphanumeric, `-`, `_`, `.` (max 63 chars)
* Optional prefix: `domain.com/key` (max 253 chars)
* Values: alphanumeric, `-`, `_`, `.` (max 63 chars)
* Multiple labels per object — design them for queries

***

### 📋 Recommended Standard Labels (Kubernetes Convention)

| Label                          | Example      | Meaning              |
| ------------------------------ | ------------ | -------------------- |
| `app.kubernetes.io/name`       | `mysql`      | Application name     |
| `app.kubernetes.io/instance`   | `mysql-abcd` | Unique instance ID   |
| `app.kubernetes.io/version`    | `5.7.21`     | App version          |
| `app.kubernetes.io/component`  | `database`   | Functional component |
| `app.kubernetes.io/part-of`    | `wordpress`  | Higher-level app     |
| `app.kubernetes.io/managed-by` | `helm`       | Tool that manages it |

These are **convention**, not enforced — but adopting them makes tooling (Helm, ArgoCD, monitoring) work seamlessly.

***

### 🏷️ Labels vs Annotations

| Aspect                 | Labels                  | Annotations                                           |
| ---------------------- | ----------------------- | ----------------------------------------------------- |
| **Purpose**            | Identify, group, select | Attach non-identifying metadata                       |
| **Used by selectors?** | ✅ Yes                   | ❌ No                                                  |
| **Size limit**         | 63 chars per value      | Up to 256 KB total                                    |
| **Examples**           | `env=prod`, `tier=web`  | Build info, commit SHA, contact email, ingress config |

```yaml
metadata:
  labels:
    app: web         # ← used to select
  annotations:
    git.commit: "abc123"               # ← descriptive
    contact: "@example.com"       # ← descriptive
    nginx.ingress.kubernetes.io/rewrite-target: "/"   # ← controller config
```

***

### 🔍 Selectors Deep Dive

A **Selector** filters objects based on their labels. Used everywhere — Services, Deployments, NetworkPolicies, kubectl queries.

#### Two Selector Types

**1. Equality-based** (simple `=`, `==`, `!=`):

```bash
kubectl get pods -l env=production
kubectl get pods -l env!=dev
kubectl get pods -l env=prod,tier=web    # AND logic
```

**2. Set-based** (richer — `In`, `NotIn`, `Exists`, `DoesNotExist`):

```bash
kubectl get pods -l 'env in (prod, staging)'
kubectl get pods -l 'env notin (dev, test)'
kubectl get pods -l 'tier'                # has 'tier' label (any value)
kubectl get pods -l '!tier'               # does NOT have 'tier' label
```

**In YAML** (used by Deployments, NetworkPolicies):

```yaml
selector:
  matchLabels:
    app: web                    # equality
  matchExpressions:
    - key: env
      operator: In              # In, NotIn, Exists, DoesNotExist
      values: [prod, staging]
```

***

### 💰 Resource Quotas & LimitRanges (Multi-Tenancy)

Namespaces become **truly useful for multi-tenancy** when paired with **ResourceQuotas** and **LimitRanges**.

#### ResourceQuota — "How much can the namespace consume?"

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "10"            # Total CPU requests
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"                    # Max pods
    services: "20"
    persistentvolumeclaims: "10"
    secrets: "30"
    configmaps: "30"
```

#### LimitRange — "What's the default/max per pod or container?"

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: team-a-limits
  namespace: team-a
spec:
  limits:
  - type: Container
    default:                    # Default LIMITS if not specified
      cpu: 500m
      memory: 512Mi
    defaultRequest:             # Default REQUESTS if not specified
      cpu: 100m
      memory: 128Mi
    max:                        # Max allowed per container
      cpu: "2"
      memory: 2Gi
    min:                        # Min allowed
      cpu: 50m
      memory: 64Mi
```

> 💡 Together, **ResourceQuota + LimitRange = enforceable team boundaries** without overcommitting the cluster.

***

## 🛠️ Part 2: Hands-On — Multi-Team Cluster Setup

### Scenario

You're the platform engineer for a company with two teams — **payments** and **catalog** — plus shared **monitoring**. Each team has dev + prod environments.

***

### Step 1: Create Namespaces

**Imperative:**

```powershell
# Create namespaces
kubectl create namespace payments-dev
kubectl create namespace payments-prod
kubectl create namespace catalog-dev
kubectl create namespace catalog-prod
kubectl create namespace shared-monitoring

# List
kubectl get namespaces
kubectl get ns
```

**Declarative** (`namespaces.yaml`):

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: payments-dev
  labels:
    team: payments
    env: dev
    cost-center: "1001"
---
apiVersion: v1
kind: Namespace
metadata:
  name: payments-prod
  labels:
    team: payments
    env: prod
    cost-center: "1001"
---
apiVersion: v1
kind: Namespace
metadata:
  name: catalog-dev
  labels:
    team: catalog
    env: dev
    cost-center: "1002"
---
apiVersion: v1
kind: Namespace
metadata:
  name: catalog-prod
  labels:
    team: catalog
    env: prod
    cost-center: "1002"
```

```powershell
kubectl apply -f namespaces.yaml

# Label-based namespace queries
kubectl get ns -l team=payments
kubectl get ns -l env=prod
kubectl get ns -l 'env in (dev,test)'
```

***

### Step 2: Set Default Namespace (Avoid Typing `-n` Every Time)

```powershell
# Switch default namespace for current context
kubectl config set-context --current --namespace=payments-dev

# Verify
kubectl config view --minify | Select-String namespace

# Now all commands target payments-dev
kubectl get pods
```

> 💡 **Pro tool**: Install **`kubens`** (part of `kubectx`) for super-fast namespace switching:
>
> ```powershell
> choco install kubens
> kubens payments-prod    # instant switch
> kubens                  # interactive picker
> ```

***

### Step 3: Apply Resource Quotas

`quotas.yaml`:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: payments-prod-quota
  namespace: payments-prod
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "100"
    services: "30"
    services.loadbalancers: "2"
    persistentvolumeclaims: "20"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: payments-prod-limits
  namespace: payments-prod
spec:
  limits:
  - type: Container
    default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    max:
      cpu: "4"
      memory: 4Gi
```

```powershell
kubectl apply -f quotas.yaml

# Verify
kubectl describe quota -n payments-prod
kubectl describe limitrange -n payments-prod
```

***

### Step 4: Deploy Labeled Workloads

`payments-app.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payments-api
  namespace: payments-prod
  labels:
    app.kubernetes.io/name: payments-api
    app.kubernetes.io/instance: payments-api-v2
    app.kubernetes.io/version: "2.1.0"
    app.kubernetes.io/component: backend
    app.kubernetes.io/part-of: payments
    app.kubernetes.io/managed-by: kubectl
    team: payments
    env: prod
    tier: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payments-api
      env: prod
  template:
    metadata:
      labels:
        app: payments-api
        env: prod
        tier: backend
        team: payments
        version: v2.1.0
    spec:
      containers:
      - name: api
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payments-frontend
  namespace: payments-prod
  labels:
    team: payments
    env: prod
    tier: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: payments-frontend
  template:
    metadata:
      labels:
        app: payments-frontend
        team: payments
        env: prod
        tier: frontend
    spec:
      containers:
      - name: web
        image: nginx:1.25
```

```powershell
kubectl apply -f payments-app.yaml
kubectl get pods -n payments-prod --show-labels
```

***

### Step 5: Query with Selectors

```powershell
# By single label
kubectl get pods -n payments-prod -l app=payments-api

# AND logic — multiple labels
kubectl get pods -n payments-prod -l team=payments,tier=backend

# Set-based queries
kubectl get pods -n payments-prod -l 'tier in (backend, frontend)'
kubectl get pods -n payments-prod -l 'env!=dev'
kubectl get pods -n payments-prod -l 'team,!deprecated'

# Across all namespaces
kubectl get pods -A -l team=payments

# Combined with output formatting
kubectl get pods -A -l env=prod -o wide
kubectl get pods -A -l env=prod -o custom-columns=NS:.metadata.namespace,POD:.metadata.name,STATUS:.status.phase

# Show all labels
kubectl get pods -n payments-prod --show-labels

# Filter by label and apply action
kubectl delete pods -l app=payments-api -n payments-prod
```

***

### Step 6: Add / Modify / Remove Labels

```powershell
# Add a label
kubectl label pod <pod-name> -n payments-prod owner=

# Overwrite existing label
kubectl label pod <pod-name> -n payments-prod env=staging --overwrite

# Remove a label (note the trailing minus)
kubectl label pod <pod-name> -n payments-prod owner-

# Bulk label all pods matching a selector
kubectl label pods -n payments-prod -l tier=backend reviewed=true

# Label a namespace
kubectl label namespace payments-prod environment=production
```

***

### Step 7: Annotations Example

```powershell
# Add annotation
kubectl annotate deployment payments-api -n payments-prod `
  contact="@example.com" `
  git.commit="abc123def" `
  build.number="456"

# View
kubectl get deployment payments-api -n payments-prod -o jsonpath='{.metadata.annotations}'

# Remove
kubectl annotate deployment payments-api -n payments-prod contact-
```

***

### Step 8: Service Selector Demo

Watch how the Service selector "magically" connects to pods via labels:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payments-api-svc
  namespace: payments-prod
spec:
  selector:
    app: payments-api        # ← selects ALL pods with this label
    env: prod
  ports:
  - port: 80
    targetPort: 80
```

```powershell
kubectl apply -f payments-svc.yaml

# Check which pods are selected
kubectl get endpoints payments-api-svc -n payments-prod

# Add a new pod with matching labels — auto-added!
kubectl run extra-pod --image=nginx -n payments-prod -l app=payments-api,env=prod
kubectl get endpoints payments-api-svc -n payments-prod
# New pod IP appears in endpoints!
```

***

### Step 9: Cross-Namespace Service Discovery

```powershell
# From a pod in catalog-prod, reach payments-prod service:
kubectl run client --image=busybox:1.36 -n catalog-prod -it --rm --restart=Never -- sh

# Inside the pod:
nslookup payments-api-svc.payments-prod.svc.cluster.local
wget -qO- http://payments-api-svc.payments-prod
```

> 💡 **FQDN pattern**: `<service>.<namespace>.svc.cluster.local`

***

### 🎯 Mini-Exercise: Test Quota Enforcement

```powershell
# Try to exceed the quota
$yaml = @"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: greedy-app
  namespace: payments-prod
spec:
  replicas: 200
  selector:
    matchLabels:
      app: greedy
  template:
    metadata:
      labels:
        app: greedy
    spec:
      containers:
      - name: app
        image: nginx
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
"@
$yaml | kubectl apply -f -

# Watch what happens
kubectl get deployment greedy-app -n payments-prod
kubectl describe replicaset -n payments-prod | Select-String "exceeded quota"
# Quota blocks further pods, but doesn't fail the deployment — RS retries
```

***

## 💻 Part 3: Essential Commands Cheat Sheet

### Namespaces

```powershell
kubectl create namespace <name>
kubectl get namespaces
kubectl get ns
kubectl describe ns <name>
kubectl delete ns <name>                        # ⚠️ deletes EVERYTHING inside

# Switch default namespace
kubectl config set-context --current --namespace=<name>

# Run commands in a specific namespace
kubectl get pods -n <namespace>
kubectl get all -A                              # all namespaces, all resources
```

### Labels

```powershell
# Add
kubectl label <resource> <name> key=value

# Overwrite
kubectl label <resource> <name> key=value --overwrite

# Remove
kubectl label <resource> <name> key-

# Bulk
kubectl label pods -l app=web reviewed=true

# Show
kubectl get pods --show-labels
kubectl get pods -L app,env,tier               # specific labels as columns
```

### Selectors

```powershell
# Equality
kubectl get pods -l env=prod
kubectl get pods -l env=prod,tier=web

# Inequality
kubectl get pods -l env!=dev

# Set-based
kubectl get pods -l 'env in (prod,staging)'
kubectl get pods -l 'env notin (dev)'
kubectl get pods -l 'tier'                     # has label
kubectl get pods -l '!tier'                    # missing label

# Combined with field selectors
kubectl get pods --field-selector status.phase=Running -l env=prod
```

### Quotas & LimitRanges

```powershell
kubectl get resourcequota -A
kubectl describe quota <name> -n <ns>

kubectl get limitrange -A
kubectl describe limitrange <name> -n <ns>
```

### Annotations

```powershell
kubectl annotate <resource> <name> key=value
kubectl annotate <resource> <name> key- 
kubectl get <resource> <name> -o jsonpath='{.metadata.annotations}'
```

***

## 🎤 Part 4: Interview Prep

### ⭐ Star Question: *"How would you isolate workloads for different teams?"*

**Model answer for a Senior DevOps interview:**

> "Workload isolation in Kubernetes is a **multi-layered design** — namespaces alone are not enough for real production isolation, so my approach combines several mechanisms.
>
> **Layer 1 — Namespaces for logical separation**: I create per-team, per-environment namespaces — `team-payments-dev`, `team-payments-prod`, etc. Each namespace gets labels like `team`, `env`, `cost-center` for governance and billing chargeback.
>
> **Layer 2 — RBAC for access control**: I create `Role` and `RoleBinding` objects per namespace. Teams get edit rights only in their own namespace. I integrate with **Azure AD** on AKS so RBAC is mapped to AD groups — `payments-devs` group gets edit on `payments-*` namespaces. Cluster admins use `ClusterRole` sparingly.
>
> **Layer 3 — ResourceQuotas + LimitRanges**: To prevent one team from starving others, I apply ResourceQuotas (`requests.cpu`, `requests.memory`, `pods`, `services.loadbalancers`) and LimitRanges (defaults and maximums per container). This enforces fair share and predictable costs.
>
> **Layer 4 — NetworkPolicies for traffic isolation**: By default, all pods in K8s can talk to each other — even across namespaces. I deploy a default **deny-all** NetworkPolicy per namespace, then explicitly allow only required ingress/egress. This is critical because namespaces alone don't isolate networking.
>
> **Layer 5 — Node-level isolation (when required)**: For sensitive workloads, I use **taints, tolerations, and node selectors** so payments workloads run on dedicated node pools — separate from catalog. On AKS, this maps to multiple node pools with different VM SKUs or security profiles. For compliance use cases (PCI-DSS, HIPAA), I'd go further with **separate clusters or virtual clusters via vCluster**.
>
> **Layer 6 — Pod Security Standards**: I enforce the `restricted` profile per namespace using Pod Security Admission so no privileged containers, no hostPath mounts, no root users.
>
> **Layer 7 — GitOps boundaries**: Each team has their own Git repo / folder, and ArgoCD or Flux is configured so team A's pipeline can't deploy to team B's namespace. This shifts isolation left into the platform layer.
>
> **Layer 8 — Cost visibility and audit**: Labels like `team`, `cost-center`, `env` feed into tools like **Kubecost** or **OpenCost** for chargeback. Audit logs help detect cross-team violations.
>
> So in short: **Namespaces (logical) → RBAC (who) → Quotas (how much) → NetworkPolicies (traffic) → Node pools (compute) → PSS (security) → GitOps (deploy) → Cost/audit (governance)**. Namespaces alone are a soft boundary; real isolation comes from layering all of these."

***

### 🔥 Likely Follow-Up Questions

| #  | Question                                                                                   | Quick Pointer                                                                                                                                    |
| -- | ------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1  | **Are namespaces a security boundary?**                                                    | **No** — they're a logical boundary. Pods can talk across namespaces by default. Need NetworkPolicies + RBAC.                                    |
| 2  | **What resources are cluster-scoped vs namespaced?**                                       | Cluster: Nodes, PVs, StorageClass, ClusterRole. Namespaced: Pods, Services, ConfigMaps, Roles.                                                   |
| 3  | **What happens when you delete a namespace?**                                              | All namespaced resources inside get deleted. Cluster-scoped resources (like PVs) are unaffected.                                                 |
| 4  | **Can you have the same resource name across namespaces?**                                 | Yes — names are unique only within a namespace.                                                                                                  |
| 5  | **What's the difference between labels and annotations?**                                  | Labels are for **identification & selection**. Annotations are non-identifying metadata (build info, ingress config, contacts).                  |
| 6  | **How do Services use labels?**                                                            | Services use a **selector** to find pods. The Service's `spec.selector` must match pod labels exactly.                                           |
| 7  | **What are equality-based vs set-based selectors?**                                        | Equality: `=`, `!=`. Set-based: `In`, `NotIn`, `Exists`, `DoesNotExist`. Set-based is more powerful.                                             |
| 8  | **How do you enforce naming/labeling standards?**                                          | Use **OPA Gatekeeper** or **Kyverno** policies. Reject resources without required labels.                                                        |
| 9  | **What if a pod has multiple replicas and you want only one selected?**                    | Add a unique label like `pod-hash` and use that in selector. But typically Services select ALL replicas — that's the point.                      |
| 10 | **What is `kubectl explain`?**                                                             | Built-in docs for any resource and field: `kubectl explain pod.metadata.labels`.                                                                 |
| 11 | **How do ResourceQuotas interact with LimitRanges?**                                       | LimitRanges set defaults per container; ResourceQuota enforces totals at the namespace level. Together they enforce both micro and macro limits. |
| 12 | **What happens if a pod doesn't specify `requests`/`limits` in a namespace with a quota?** | The pod is **rejected** unless a LimitRange provides defaults.                                                                                   |
| 13 | **Can NetworkPolicies span namespaces?**                                                   | Yes — selectors can use `namespaceSelector`.                                                                                                     |
| 14 | **Why might you use multiple clusters instead of namespaces?**                             | Stronger isolation, different K8s versions, regulatory compliance, blast radius reduction, independent upgrade cycles.                           |
| 15 | **What is vCluster / Capsule?**                                                            | Virtual clusters running inside a host cluster. Stronger multi-tenancy than namespaces — each tenant gets their own API server.                  |

***

### 💡 Senior-Level Scenario Questions

**Q1: "Two teams are sharing a cluster. Team A keeps deploying buggy apps that consume all resources, starving Team B. How do you solve this?"**

> "Multi-pronged response:
>
> 1. **Apply ResourceQuota** to Team A's namespace — cap their `requests.cpu`, `requests.memory`, and pod counts. This enforces a hard upper bound.
> 2. **Apply LimitRange** — set sensible defaults and maximums per container, so individual pods can't be massive.
> 3. **PriorityClasses** — assign higher priority to Team B's critical workloads so the scheduler preempts Team A's low-priority pods under contention.
> 4. **Separate node pools** with taints and tolerations — Team A runs on `nodepool-team-a`, Team B on `nodepool-team-b`. They no longer compete for the same compute.
> 5. **NetworkPolicies** ensure no cross-team interference at the network layer.
> 6. **Monitoring & alerts** via Prometheus on per-namespace usage. Alert when a team approaches quota.
> 7. Long-term, if conflicts persist — consider **separate clusters** or **vCluster** for stronger isolation."

**Q2: "A developer accidentally ran `kubectl delete namespace production`. What's your incident response?"**

> "Immediate steps:
>
> 1. **Check status** — `kubectl get ns production` (it's likely in `Terminating` phase with finalizers).
> 2. If still terminating, **STOP if possible** — depending on finalizers, you may be able to abort by editing them out. But this is risky.
> 3. **Restore from backup** — ideally we have **Velero** or **AKS Backup** taking scheduled snapshots. Restore the namespace: `velero restore create --from-backup <backup-name> --include-namespaces production`.
> 4. **Restore PVs/data** — PersistentVolumes are cluster-scoped, so if the **ReclaimPolicy was `Retain`**, the underlying storage survives. Re-create PVCs to bind back.
> 5. **GitOps recovery** — if managed by ArgoCD/Flux, re-applying the Git state recreates resources.
> 6. **Post-incident**:
>    * Enable **RBAC** so devs can't delete namespaces (`delete` verb on `namespaces` removed).
>    * Add **OPA Gatekeeper/Kyverno** policy: protected namespaces require an annotation override to delete.
>    * Implement **just-in-time admin access** via Azure PIM.
>    * Daily Velero backups + tested DR runbook."

**Q3: "How do you organize labels for a 200-microservice platform?"**

> "I'd standardize on **Kubernetes recommended labels** plus organizational labels:
>
> **Identity labels (required, enforced by Gatekeeper):**
>
> * `app.kubernetes.io/name` — service name
> * `app.kubernetes.io/version` — semver
> * `app.kubernetes.io/component` — frontend/backend/db
> * `app.kubernetes.io/part-of` — bounded context (e.g., `checkout`)
>
> **Operational labels:**
>
> * `team` — owning team
> * `env` — dev/staging/prod
> * `tier` — frontend/backend/data
> * `criticality` — tier-1/tier-2/tier-3
>
> **Governance labels:**
>
> * `cost-center` — for chargeback
> * `compliance` — pci/hipaa/none
> * `data-classification` — public/internal/confidential
>
> **Release labels:**
>
> * `version` — semver
> * `release` — canary/stable
>
> I'd enforce these via **Kyverno policies** that reject resources missing required labels. I'd document the schema in an internal developer portal (Backstage). Tools like **Kubecost** consume these for cost allocation; **Prometheus** uses them for alerting rules; **ArgoCD** uses them for app grouping."

**Q4: "Your monitoring tool needs to read pods across ALL namespaces. How do you grant that without giving full cluster admin?"**

> "Use a **ClusterRole** with read-only verbs on the needed resources, bound to a ServiceAccount via **ClusterRoleBinding**.
>
> ```yaml
> apiVersion: rbac.authorization.k8s.io/v1
> kind: ClusterRole
> metadata:
>   name: monitoring-reader
> rules:
> - apiGroups: [\"\"]
>   resources: [\"pods\", \"services\", \"nodes\", \"endpoints\"]
>   verbs: [\"get\", \"list\", \"watch\"]
> - apiGroups: [\"metrics.k8s.io\"]
>   resources: [\"pods\", \"nodes\"]
>   verbs: [\"get\", \"list\"]
> ```
>
> Then a ClusterRoleBinding ties this to the Prometheus ServiceAccount in `shared-monitoring` namespace. Principle of least privilege — read-only, only required resources. We'd also enable **audit logs** to track what monitoring reads."

***

## 🏆 Bonus: Multi-Tenancy Models Comparison

| Model                                                | Isolation   | Complexity | Cost   | When to Use                          |
| ---------------------------------------------------- | ----------- | ---------- | ------ | ------------------------------------ |
| **Shared cluster + namespaces**                      | Soft        | Low        | Low    | Small/medium teams, internal trust   |
| **Shared cluster + NetworkPolicies + Quotas + RBAC** | Medium      | Medium     | Low    | Most enterprise platforms            |
| **Shared cluster + node pools per team**             | Medium-High | Medium     | Medium | Performance/compliance needs         |
| **vCluster (virtual clusters)**                      | High        | Medium     | Low    | Many tenants, dev/test isolation     |
| **Separate clusters per team**                       | Highest     | High       | High   | Strong compliance, isolated upgrades |

***

# 🎯 Day 7: Week 1 Review + Mini Project

Welcome to Day 7, We've covered massive ground this week — from architecture to networking to configuration management. Today we consolidate everything into a **real, working 2-tier application** (Node.js + PostgreSQL) that exercises every concept from Days 1–6. Then we'll close with **20 fundamental interview questions** to lock in your knowledge. Let's build something real! 🚀

***

## 📋 Part 1: Week 1 Recap

### What You've Mastered

| Day       | Topic                | Key Takeaway                                                                                                  |
| --------- | -------------------- | ------------------------------------------------------------------------------------------------------------- |
| **Day 1** | Architecture         | Control plane (API server, etcd, scheduler, controller manager) + Worker nodes (kubelet, kube-proxy, runtime) |
| **Day 2** | Pods                 | Smallest deployable unit — shared network/storage; init containers + sidecars                                 |
| **Day 3** | Deployments          | Manage ReplicaSets → Pods; rolling updates, rollbacks, scaling                                                |
| **Day 4** | Services             | Stable network endpoint — ClusterIP, NodePort, LoadBalancer, ExternalName                                     |
| **Day 5** | ConfigMaps & Secrets | Externalize config; env vars vs volume mounts; Secrets are **base64, not encrypted**                          |
| **Day 6** | Namespaces & Labels  | Logical isolation, multi-tenancy, label-based queries, quotas                                                 |

### The Big Picture

```
┌────────────────────────────────────────────────────────────┐
│                    KUBERNETES CLUSTER                        │
│                                                              │
│  ┌──────────────────── NAMESPACE: petstore ────────────────┐│
│  │                                                          ││
│  │  ┌─────────────┐         ┌─────────────────┐           ││
│  │  │ ConfigMap   │         │   Secret        │           ││
│  │  │ app-config  │         │ db-credentials  │           ││
│  │  └──────┬──────┘         └────────┬────────┘           ││
│  │         │                          │                    ││
│  │         ▼                          ▼                    ││
│  │  ┌─────────────────────────────────────┐               ││
│  │  │     Deployment: nodejs-app          │               ││
│  │  │  ┌──────┐ ┌──────┐ ┌──────┐         │               ││
│  │  │  │ Pod  │ │ Pod  │ │ Pod  │         │               ││
│  │  │  └──────┘ └──────┘ └──────┘         │               ││
│  │  └────────────────┬────────────────────┘               ││
│  │                   │                                     ││
│  │  ┌────────────────▼────────────────┐                   ││
│  │  │ Service: nodejs-svc (ClusterIP) │                   ││
│  │  └────────────────┬────────────────┘                   ││
│  │                   │                                     ││
│  │  ┌────────────────▼────────────────────┐               ││
│  │  │   Deployment: postgres              │               ││
│  │  │        ┌──────┐                     │               ││
│  │  │        │ Pod  │ ◄── PersistentVolume│               ││
│  │  │        └──────┘                     │               ││
│  │  └─────────────────────────────────────┘               ││
│  │                   ▲                                     ││
│  │  ┌────────────────┴───────────────┐                    ││
│  │  │ Service: postgres-svc (ClusterIP)│                  ││
│  │  └──────────────────────────────────┘                  ││
│  └────────────────────────────────────────────────────────┘│
│                                                              │
│  ┌──────────── Service: nodejs-public (NodePort) ─────────┐│
│  │      External access on <NodeIP>:30080                  ││
│  └─────────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────┘
```

***

## 🛠️ Part 2: Mini Project — 2-Tier App (Node.js + PostgreSQL)

### Project Overview

We'll deploy a **simple Pet Store API** — a Node.js app that talks to PostgreSQL. This mini-project will use **every concept** from Week 1:

✅ Dedicated **Namespace** with labels & quota  
✅ **PostgreSQL Deployment** with **PVC** (basic storage — full StatefulSet on Day 7 of Week 2)  
✅ **Node.js Deployment** with **ConfigMap** + **Secret**  
✅ Internal **ClusterIP Service** for DB  
✅ External **NodePort Service** for app  
✅ Init container to wait for DB readiness  
✅ Resource requests/limits  
✅ Readiness & liveness probes

***

### File Structure

Create a project folder:

```powershell
mkdir petstore-k8s
cd petstore-k8s
```

You'll create **6 files**:

1. `00-namespace.yaml`
2. `01-configmap.yaml`
3. `02-secret.yaml`
4. `03-postgres.yaml`
5. `04-nodejs.yaml`
6. `05-services.yaml`

***

### Step 1: Namespace + Quota

**`00-namespace.yaml`:**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: petstore
  labels:
    app: petstore
    env: dev
    team: backend
    cost-center: "2001"
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: petstore-quota
  namespace: petstore
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 4Gi
    limits.cpu: "8"
    limits.memory: 8Gi
    pods: "20"
    services: "10"
    persistentvolumeclaims: "5"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: petstore-limits
  namespace: petstore
spec:
  limits:
  - type: Container
    default:
      cpu: 250m
      memory: 256Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    max:
      cpu: "1"
      memory: 1Gi
```

***

### Step 2: ConfigMap (App Settings)

**`01-configmap.yaml`:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: petstore
  labels:
    app: petstore
data:
  # Non-sensitive app configuration
  NODE_ENV: "production"
  LOG_LEVEL: "info"
  APP_PORT: "3000"
  DB_HOST: "postgres-svc"        # ← uses K8s DNS
  DB_PORT: "5432"
  DB_NAME: "petstore"
  POOL_SIZE: "10"
  REQUEST_TIMEOUT: "30000"
  
  # App config file (mounted as volume)
  app.properties: |
    feature.newCheckout=true
    feature.recommendations=false
    cache.ttl=3600
    metrics.enabled=true
```

***

### Step 3: Secret (DB Credentials)

**`02-secret.yaml`:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: petstore
  labels:
    app: petstore
type: Opaque
stringData:
  # stringData = plain text, K8s base64-encodes automatically
  POSTGRES_USER: "petadmin"
  POSTGRES_PASSWORD: "S3cur3P@ss2026!"
  POSTGRES_DB: "petstore"
  DB_USER: "petadmin"
  DB_PASSWORD: "S3cur3P@ss2026!"
```

> 🔒 **Production note**: In real life, this Secret would come from **Azure Key Vault via CSI Secret Store Driver**, never committed to Git. For learning, this works.

***

### Step 4: PostgreSQL Deployment + PVC

**`03-postgres.yaml`:**

```yaml
# Persistent Volume Claim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: petstore
  labels:
    app: postgres
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
# PostgreSQL Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: petstore
  labels:
    app: postgres
    tier: database
spec:
  replicas: 1
  strategy:
    type: Recreate            # DB shouldn't have multiple writers
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
        tier: database
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
          name: postgres
        env:
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: POSTGRES_USER
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: POSTGRES_PASSWORD
        - name: POSTGRES_DB
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: POSTGRES_DB
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        readinessProbe:
          exec:
            command:
              - sh
              - -c
              - "pg_isready -U $POSTGRES_USER -d $POSTGRES_DB"
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
        livenessProbe:
          exec:
            command:
              - sh
              - -c
              - "pg_isready -U $POSTGRES_USER"
          initialDelaySeconds: 30
          periodSeconds: 10
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
```

***

### Step 5: Node.js Application Deployment

**`04-nodejs.yaml`:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-app
  namespace: petstore
  labels:
    app: nodejs-app
    tier: backend
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: nodejs-app
  template:
    metadata:
      labels:
        app: nodejs-app
        tier: backend
        version: v1.0.0
    spec:
      # 🚦 Init container: wait until DB is ready
      initContainers:
      - name: wait-for-postgres
        image: busybox:1.36
        command:
        - sh
        - -c
        - |
          echo "Waiting for PostgreSQL...";
          until nc -z postgres-svc 5432; do
            echo "DB not ready, sleeping 2s...";
            sleep 2;
          done;
          echo "PostgreSQL is ready!"
      
      containers:
      - name: app
        # Using nginx as a placeholder; in real life this is your Node.js image
        # Example: /petstore-api:1.0.0
        image: nginx:1.25-alpine
        ports:
        - containerPort: 80
          name: http
        
        # Inject ALL ConfigMap & Secret keys as env vars
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: db-credentials
        
        # Mount config file as volume (auto-reloads on change)
        volumeMounts:
        - name: app-config-vol
          mountPath: /etc/app/config
          readOnly: true
        
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
        
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 20
      
      volumes:
      - name: app-config-vol
        configMap:
          name: app-config
          items:
          - key: app.properties
            path: app.properties
```

***

### Step 6: Services (Internal + External)

**`05-services.yaml`:**

```yaml
# Internal Service for PostgreSQL (ClusterIP)
apiVersion: v1
kind: Service
metadata:
  name: postgres-svc
  namespace: petstore
  labels:
    app: postgres
spec:
  type: ClusterIP
  selector:
    app: postgres
  ports:
  - name: postgres
    port: 5432
    targetPort: 5432
    protocol: TCP
---
# Internal Service for Node.js (ClusterIP — service-to-service)
apiVersion: v1
kind: Service
metadata:
  name: nodejs-svc
  namespace: petstore
  labels:
    app: nodejs-app
spec:
  type: ClusterIP
  selector:
    app: nodejs-app
  ports:
  - name: http
    port: 80
    targetPort: 80
---
# External Service for Node.js (NodePort — for testing access)
apiVersion: v1
kind: Service
metadata:
  name: nodejs-public
  namespace: petstore
  labels:
    app: nodejs-app
spec:
  type: NodePort
  selector:
    app: nodejs-app
  ports:
  - name: http
    port: 80
    targetPort: 80
    nodePort: 30080
```

***

### 🚀 Deploy It All!

```powershell
# Apply in order (or all at once — K8s resolves dependencies eventually)
kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-configmap.yaml
kubectl apply -f 02-secret.yaml
kubectl apply -f 03-postgres.yaml
kubectl apply -f 04-nodejs.yaml
kubectl apply -f 05-services.yaml

# OR deploy everything in one shot
kubectl apply -f .
```

***

### 🔍 Verify the Deployment

```powershell
# Switch context to petstore namespace
kubectl config set-context --current --namespace=petstore

# 1. Check all resources
kubectl get all
kubectl get configmap,secret,pvc

# 2. Watch pods come up
kubectl get pods -w

# 3. Verify init container ran for nodejs-app
kubectl describe pod -l app=nodejs-app | Select-String "init|Init"

# 4. Check rollout status
kubectl rollout status deployment/postgres
kubectl rollout status deployment/nodejs-app

# 5. Inspect Service endpoints
kubectl get endpoints
kubectl describe svc postgres-svc

# 6. Verify quota usage
kubectl describe quota petstore-quota
```

***

### 🧪 Test Connectivity

```powershell
# Test 1: External access via NodePort
# On Minikube:
minikube service nodejs-public -n petstore --url
# Or get node IP and curl:
$nodeIP = kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}'
curl http://$nodeIP:30080

# Test 2: From inside the cluster — service discovery
kubectl run debug --image=busybox:1.36 -it --rm --restart=Never -- sh
# Inside the pod:
nslookup nodejs-svc
nslookup postgres-svc.petstore.svc.cluster.local
wget -qO- http://nodejs-svc

# Test 3: Connect to Postgres from a temp pod
kubectl run pg-client --image=postgres:15-alpine -it --rm --restart=Never `
  --env="PGPASSWORD=S3cur3P@ss2026!" `
  -- psql -h postgres-svc -U petadmin -d petstore -c "SELECT version();"

# Test 4: Verify env vars and config file inside the app pod
$pod = kubectl get pod -l app=nodejs-app -o jsonpath='{.items[0].metadata.name}'
kubectl exec $pod -- env | Select-String "NODE_ENV|DB_HOST|DB_USER"
kubectl exec $pod -- cat /etc/app/config/app.properties
```

***

### 🎬 Bonus Demos to Run

```powershell
# Demo 1: Scale the Node.js app
kubectl scale deployment/nodejs-app --replicas=5
kubectl get pods -l app=nodejs-app

# Demo 2: Rolling update (change image tag)
kubectl set image deployment/nodejs-app app=nginx:1.26-alpine --record
kubectl rollout status deployment/nodejs-app
kubectl rollout history deployment/nodejs-app

# Demo 3: Rollback
kubectl rollout undo deployment/nodejs-app

# Demo 4: Update ConfigMap → see file auto-update
kubectl edit configmap app-config
# Change LOG_LEVEL: "info" → "debug"
# Wait ~60s, then:
kubectl exec $pod -- cat /etc/app/config/app.properties

# Demo 5: Check that postgres data survives pod restart
kubectl exec -it deploy/postgres -- psql -U petadmin -d petstore -c "CREATE TABLE pets (id SERIAL, name VARCHAR(50)); INSERT INTO pets (name) VALUES ('Buddy');"
kubectl delete pod -l app=postgres
# Wait for new pod
kubectl get pods -l app=postgres -w
# Verify data survived
kubectl exec -it deploy/postgres -- psql -U petadmin -d petstore -c "SELECT * FROM pets;"
```

***

### 🧹 Cleanup

```powershell
# Delete everything by deleting the namespace
kubectl delete namespace petstore

# OR delete one by one
kubectl delete -f .
```

***

## 🎤 Part 3: 20 Fundamental Interview Questions

### Section A: Architecture & Basics (Q1–Q5)

**Q1. What is Kubernetes, and what problems does it solve?**

> Kubernetes is an open-source container orchestration platform that automates **deployment, scaling, networking, and management** of containerized applications. It solves problems like manual container scheduling, self-healing (auto-restart failed containers), scaling under load, rolling updates without downtime, service discovery, and infrastructure abstraction across cloud/on-prem.

***

**Q2. Explain the role of each control plane component.**

> * **kube-apiserver**: REST API gateway; all components talk to it
> * **etcd**: distributed key-value store; the single source of truth for cluster state
> * **kube-scheduler**: decides which node a new Pod runs on (filter + score)
> * **kube-controller-manager**: runs reconciliation loops (node, replicaset, endpoint controllers)
> * **cloud-controller-manager**: cloud-specific logic (LBs, disks, routes for Azure/AWS/GCP)

***

**Q3. What runs on a Kubernetes worker node?**

> * **kubelet**: node agent that ensures containers in PodSpecs are running
> * **kube-proxy**: implements Service networking via iptables/IPVS rules
> * **Container runtime**: actually runs containers (containerd, CRI-O); communicates via CRI

***

**Q4. What's the difference between a Pod and a container?**

> A **container** is a runtime instance of an image — OS-level virtualization unit. A **Pod** is the smallest Kubernetes scheduling unit — a logical wrapper around one or more containers that share the same network namespace (same IP, localhost communication), shared volumes, and lifecycle. Kubernetes schedules Pods, not containers directly.

***

**Q5. Why use a Deployment instead of bare Pods?**

> Bare Pods don't self-heal, scale, or update. A Deployment provides:
>
> * **Self-healing** via ReplicaSet (replaces dead Pods)
> * **Declarative updates** with rolling strategy
> * **Rollback** to any previous revision
> * **Scaling** with one command
> * **Version history** for audit

***

### Section B: Workloads & Updates (Q6–Q10)

**Q6. Explain RollingUpdate vs Recreate strategies.**

> * **RollingUpdate** (default): gradually replaces old Pods with new ones; controlled by `maxSurge` and `maxUnavailable`. Zero downtime. Requires backward-compatible versions.
> * **Recreate**: kills ALL old Pods first, then creates new ones. Causes downtime. Used when versions can't coexist (e.g., DB schema changes).

***

**Q7. How does Kubernetes handle a failed Deployment rollout?**

> The new ReplicaSet scales up only when new Pods become **Ready** (via probes). If new Pods never become Ready, old Pods aren't killed (governed by `maxUnavailable`) → no downtime. After `progressDeadlineSeconds` (default 600s), the Deployment is marked failed. `kubectl rollout undo` instantly reverts by scaling up the old ReplicaSet (still retained at zero replicas).

***

**Q8. What is the Pod lifecycle?**

> Phases: **Pending** → **Running** → **Succeeded** or **Failed** (or **Unknown** for communication issues). Container states inside: **Waiting**, **Running**, **Terminated**. Conditions track readiness: `PodScheduled`, `Initialized`, `ContainersReady`, `Ready`.

***

**Q9. What's the difference between init containers and sidecars?**

> * **Init containers**: run **to completion sequentially BEFORE** the main containers start. Used for one-time setup (DB readiness check, Git clone, schema migration).
> * **Sidecars**: run **alongside** the main container for the pod's entire lifetime. Used for cross-cutting concerns (logging, proxy, metrics export).

***

**Q10. What's the difference between `kubectl apply` and `kubectl create`?**

> * **`create`** is imperative — fails if the object already exists.
> * **`apply`** is declarative — idempotent, creates or updates based on the YAML; tracks last-applied-configuration for proper merging. **`apply` is the production standard.**

***

### Section C: Services & Networking (Q11–Q14)

**Q11. Explain the four Service types and when to use each.**

> * **ClusterIP** (default): internal-only; microservice-to-microservice
> * **NodePort**: exposes on every node IP at a high port (30000–32767); dev/test or on-prem
> * **LoadBalancer**: provisions a cloud LB with public IP; production on AKS/EKS/GKE
> * **ExternalName**: DNS CNAME alias to an external host; migrations and external service abstraction

***

**Q12. How does service discovery work in Kubernetes?**

> **CoreDNS** auto-creates DNS records for every Service: `<svc>.<ns>.svc.cluster.local`. Pods resolve service names via DNS to the ClusterIP (a stable virtual IP). kube-proxy programs iptables/IPVS rules to load-balance traffic from the ClusterIP to backing Pod IPs.

***

**Q13. What's the difference between `port`, `targetPort`, and `nodePort`?**

> * **`port`**: the Service's port (used by clients via ClusterIP)
> * **`targetPort`**: the container port the traffic is forwarded to
> * **`nodePort`**: the port exposed on every node (NodePort/LoadBalancer types only)

***

**Q14. How do pods in different namespaces communicate?**

> Via the **fully qualified DNS name**: `<service>.<namespace>.svc.cluster.local`. Pods can reach services in other namespaces by default (namespaces don't isolate network) — you need **NetworkPolicies** to restrict.

***

### Section D: Config, Secrets & Storage (Q15–Q17)

**Q15. Are Secrets encrypted in Kubernetes?**

> **No** — by default Secrets are only **base64-encoded** (which is NOT encryption — easily reversible). Encryption at rest in etcd must be **explicitly enabled** via `EncryptionConfiguration`. Production should use **KMS providers** (Azure Key Vault) or external systems like **CSI Secret Store Driver** or **HashiCorp Vault**.

***

**Q16. What's the difference between mounting a ConfigMap as env var vs volume?**

> * **Env var**: simpler, but does **NOT auto-update** — pod restart required.
> * **Volume mount**: auto-updates within \~60 seconds (no restart); better for files and large configs. Atomic via symlink swap.

***

**Q17. How do you securely manage secrets in Kubernetes?**

> Layered approach:
>
> 1. **Enable encryption at rest** (KMS via Azure Key Vault)
> 2. **Externalize via CSI Secret Store Driver** or HashiCorp Vault
> 3. **RBAC** — restrict `get/list` on secrets
> 4. **No secrets in Git** — use Sealed Secrets, SOPS, or External Secrets Operator
> 5. **Rotate regularly** with automation
> 6. **Mount as files, not env vars** (env vars leak in `describe`, process listings)

***

### Section E: Organization & Multi-Tenancy (Q18–Q20)

**Q18. Are namespaces a security boundary?**

> **No** — they're a logical/organizational boundary. By default, pods in different namespaces can talk to each other. Real isolation requires **NetworkPolicies**, **RBAC**, **ResourceQuotas**, and possibly **separate node pools** or **separate clusters/vClusters** for hard multi-tenancy.

***

**Q19. What's the difference between labels and annotations?**

> * **Labels**: identifying metadata used by **selectors** to group and query objects (`app=web`, `env=prod`). Limited to 63 chars per value.
> * **Annotations**: non-identifying metadata for tools/humans (build info, commit SHA, ingress controller config). Up to 256 KB. **Cannot be queried with selectors.**

***

**Q20. How would you isolate workloads for different teams?**

> Multi-layer strategy:
>
> 1. **Namespace per team/env** for logical separation
> 2. **RBAC** with Role/RoleBinding scoped to namespace
> 3. **ResourceQuotas + LimitRanges** for fair resource share
> 4. **NetworkPolicies** for network isolation (default deny + explicit allow)
> 5. **Taints/tolerations + node selectors** for compute isolation
> 6. **Pod Security Standards** (restricted profile)
> 7. **GitOps boundaries** so teams can only deploy to their namespaces
> 8. **Labels** for cost allocation (Kubecost/OpenCost) and audit
> 9. **vCluster or separate clusters** for hard multi-tenancy / compliance

***


# 💾 Day 8: Volumes & Persistent Storage — Production Deep Dive

Welcome to Day 8, ! Today we tackle the **single hardest topic** in Kubernetes for most engineers: **storage**. This is where the rubber meets the road — databases, message queues, ML datasets, user uploads all depend on getting this right. By end of day, you'll have a **production-grade mental model** of every volume type, the PVC binding lifecycle (a guaranteed interview question), and AKS-specific patterns you'll use day one on the job.

***

## 📚 Part 1: Theory — The Storage Stack

### The Fundamental Problem

Containers are **stateless by design**. When a container dies:

* Its filesystem is **gone**
* Any data written inside is **lost**
* A replacement container starts with a fresh image

```
Time T0:                Time T1 (pod restart):
┌──────────────┐        ┌──────────────┐
│ Pod          │        │ Pod (new)    │
│ /data/       │   →    │ /data/       │
│   users.db ✓ │        │   (empty) ✗  │
└──────────────┘        └──────────────┘
```

Kubernetes solves this with **Volumes** — a layer of abstraction that decouples storage from container lifecycle.

***

### 🎯 Volume Lifecycle Spectrum

```
SHORTER LIFE ────────────────────────────────────── LONGER LIFE

emptyDir          configMap/secret      hostPath        PV/PVC
(pod-bound)       (pod-bound)           (node-bound)    (cluster-bound)
   │                   │                    │              │
   └─ Dies w/ pod      └─ Dies w/ pod       └─ Tied to     └─ Survives
                                              specific       everything
                                              node
```

***

### 1️⃣ emptyDir — The Scratch Pad

Created when a Pod is **assigned to a node**. Empty initially. **Lives only as long as the Pod**.

```yaml
volumes:
- name: scratch
  emptyDir:
    sizeLimit: 500Mi
    medium: Memory      # Optional: RAM-backed (tmpfs) for speed
```

**When to use:**

* ✅ Inter-container file sharing in same Pod
* ✅ Scratch space for sorting/processing
* ✅ Cache that can be regenerated

**When NOT to use:**

* ❌ Any data you care about
* ❌ Database storage
* ❌ Anything that should outlive the Pod

***

### 2️⃣ hostPath — Mount the Node's Disk

Mounts a file/directory from the **host node** into the Pod.

```yaml
volumes:
- name: docker-sock
  hostPath:
    path: /var/run/docker.sock
    type: Socket
```

**Legitimate uses (all DaemonSets):**

* Node monitoring (Prometheus node-exporter reading `/proc`, `/sys`)
* Log shippers reading `/var/log`
* CNI/CSI drivers needing host access

**🚨 Senior interview red flag**: Using hostPath in regular workloads is a **security anti-pattern**. It:

* Breaks portability (Pod tied to specific node's files)
* Can read/write **anything** on the node (including kubelet credentials!)
* Bypasses RBAC and Pod Security Standards
* Should be blocked by Pod Security Admission `restricted` profile

***

### 3️⃣ Persistent Volumes — The Production Pattern

For real, durable storage, use the **PV/PVC/StorageClass** triangle.

```
┌────────────────────────────────────────────────────────────────┐
│                                                                 │
│   USER SPACE (Application Team)                                │
│   ┌─────────┐         ┌──────────────────────┐                │
│   │  Pod    │ ─uses─→ │ PersistentVolumeClaim│  ← Namespaced  │
│   └─────────┘         │ (PVC)                │                 │
│                       │ "I need 10Gi RWO"    │                 │
│                       └──────────┬───────────┘                 │
│                                  │ binds to                    │
│   ═══════════════════════════════│════════════════════════════ │
│   PLATFORM SPACE (Infra Team)    ▼                            │
│                       ┌──────────────────────┐                 │
│                       │ PersistentVolume (PV)│  ← Cluster      │
│                       │ "10Gi Azure Disk"    │     scoped     │
│                       └──────────┬───────────┘                 │
│                                  │ created by                  │
│                                  ▼                             │
│                       ┌──────────────────────┐                 │
│                       │ StorageClass         │  ← Template     │
│                       │ "managed-csi-premium"│                 │
│                       └──────────┬───────────┘                 │
│                                  │ calls                       │
│                                  ▼                             │
│                       ┌──────────────────────┐                 │
│                       │ CSI Driver           │                 │
│                       │ disk.csi.azure.com   │                 │
│                       └──────────┬───────────┘                 │
│                                  │ provisions                  │
│                                  ▼                             │
│                       ┌──────────────────────┐                 │
│                       │ Azure Managed Disk   │  ← Real cloud   │
│                       └──────────────────────┘     resource    │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

This separation lets **developers ask for storage** without knowing **how it's provisioned**.

***

### 🔐 Access Modes — Critical to Get Right

| Mode             | Abbrev   | Meaning                          | Storage Type                     |
| ---------------- | -------- | -------------------------------- | -------------------------------- |
| ReadWriteOnce    | **RWO**  | Single **node** can mount RW     | Block (Azure Disk, EBS)          |
| ReadOnlyMany     | **ROX**  | Many nodes mount RO              | Files, NFS                       |
| ReadWriteMany    | **RWX**  | Many nodes mount RW              | Files (Azure Files, NFS, CephFS) |
| ReadWriteOncePod | **RWOP** | Single **Pod** mount (K8s 1.27+) | Stricter than RWO                |

> 🎯 **Senior gotcha #1**: RWO = one **node**, not one **pod**. Multiple pods on the same node can share an RWO volume.
>
> 🎯 **Senior gotcha #2**: **Azure Disk is RWO only**. Need RWX? Use **Azure Files** or **Azure NetApp Files**.

***

### ♻️ Reclaim Policies — Data Safety Switch

| Policy      | What Happens on PVC Delete                          | When to Use                   |
| ----------- | --------------------------------------------------- | ----------------------------- |
| **Retain**  | PV stays as `Released`, data intact, manual cleanup | 🛡️ **Production databases**  |
| **Delete**  | PV + underlying storage **wiped**                   | Dev/test, ephemeral workloads |
| **Recycle** | ⚠️ Deprecated — basic scrub                         | Don't use                     |

**Default for dynamic provisioning is `Delete`** — this has caused countless production incidents. **Always override to `Retain` for critical data**.

***

### ⚙️ Volume Binding Modes

| Mode                   | When PV Binds                      |
| ---------------------- | ---------------------------------- |
| `Immediate`            | As soon as PVC is created          |
| `WaitForFirstConsumer` | When a Pod actually mounts the PVC |

**Why `WaitForFirstConsumer` matters in multi-zone AKS:**

```
Without it (Immediate):                With it (WaitForFirstConsumer):
1. PVC created                         1. PVC created (Pending)
2. PV in zone-1 provisioned            2. Pod scheduled to zone-2
3. Pod scheduled to zone-2             3. PV NOW provisioned in zone-2
4. ❌ Mount fails — wrong zone         4. ✅ Mount succeeds
```

All modern Azure StorageClasses use `WaitForFirstConsumer` by default. **Never change this for zonal storage.**

***

### 🌀 The PVC Lifecycle (Interview Star Topic)

```
        ┌─────────────────┐
        │  1. Provisioning │
        │  Static OR       │
        │  Dynamic (CSI)   │
        └────────┬─────────┘
                 ▼
        ┌─────────────────┐
        │  2. Binding      │ ← Match: storageClass, size, accessMode
        │  PVC: Pending →  │
        │  Bound           │
        │  PV: Available → │
        │  Bound           │
        └────────┬─────────┘
                 ▼
        ┌─────────────────┐
        │  3. Using        │ ← Pod mounts → CSI attaches to node
        │  Finalizers add  │
        │  protection      │
        └────────┬─────────┘
                 ▼
        ┌─────────────────┐
        │  4. Releasing    │ ← Pod deleted → volume unmounted
        │  PVC deleted →   │
        │  PV: Released    │
        └────────┬─────────┘
                 ▼
        ┌─────────────────┐
        │  5. Reclaiming   │ ← Based on reclaimPolicy
        │  Retain → manual │
        │  Delete → wiped  │
        └──────────────────┘
```

**PV phases**: `Available → Bound → Released → (Available OR Failed OR deleted)`

***

### 📦 Azure StorageClasses on AKS (Memorize These!)

| StorageClass            | Backend                 | Access | Use Case                           |
| ----------------------- | ----------------------- | ------ | ---------------------------------- |
| `managed-csi`           | Azure Disk Standard SSD | RWO    | General workloads, default         |
| `managed-csi-premium`   | Azure Disk Premium SSD  | RWO    | Databases, high IOPS               |
| `azurefile-csi`         | Azure Files Standard    | RWX    | Shared content, multi-pod writes   |
| `azurefile-csi-premium` | Azure Files Premium     | RWX    | High-perf shared storage           |
| `azureblob-fuse`        | Azure Blob Storage      | RWX    | Massive object data (ML, archives) |

```powershell
kubectl get storageclass
# View provisioners
kubectl get sc -o custom-columns=NAME:.metadata.name,PROVISIONER:.provisioner,RECLAIM:.reclaimPolicy
```

***

## 🛠️ Part 2: Hands-On — From emptyDir to Production PVC

### Lab 1: Prove emptyDir Dies with the Pod

```yaml
# 01-emptydir.yaml
apiVersion: v1
kind: Pod
metadata:
  name: scratch-pod
spec:
  containers:
  - name: writer
    image: busybox:1.36
    command: ['sh', '-c', 'echo "Created at $(date)" > /tmp/data/note.txt && sleep 3600']
    volumeMounts:
    - name: scratch
      mountPath: /tmp/data
  volumes:
  - name: scratch
    emptyDir: {}
```

```powershell
kubectl apply -f 01-emptydir.yaml
kubectl exec scratch-pod -- cat /tmp/data/note.txt

# Delete and recreate
kubectl delete pod scratch-pod
kubectl apply -f 01-emptydir.yaml
kubectl exec scratch-pod -- cat /tmp/data/note.txt
# ✅ NEW timestamp — data was lost and recreated
```

***

### Lab 2: Multi-Container Pod Sharing emptyDir

A classic producer-consumer pattern:

```yaml
# 02-shared-emptydir.yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-pod
spec:
  containers:
  - name: producer
    image: busybox:1.36
    command: ['sh', '-c', 'while true; do date >> /shared/log.txt; sleep 5; done']
    volumeMounts:
    - {name: shared, mountPath: /shared}
  - name: consumer
    image: busybox:1.36
    command: ['sh', '-c', 'tail -f /shared/log.txt']
    volumeMounts:
    - {name: shared, mountPath: /shared}
  volumes:
  - name: shared
    emptyDir: {}
```

```powershell
kubectl apply -f 02-shared-emptydir.yaml
kubectl logs shared-pod -c consumer -f
```

***

### Lab 3: Dynamic PVC (The Production Pattern)

```yaml
# 03-dynamic-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  # storageClassName omitted → uses cluster default
```

```powershell
kubectl apply -f 03-dynamic-pvc.yaml

# Watch the binding
kubectl get pvc app-data -w
# STATUS: Pending → Bound (within seconds on AKS)

# Inspect the auto-created PV
kubectl get pv
kubectl describe pv $(kubectl get pvc app-data -o jsonpath='{.spec.volumeName}')
```

***

### Lab 4: Persistence Across Pod Restarts (The Core Test)

```yaml
# 04-persistent-app.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: notes-pvc
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: notes-app
spec:
  replicas: 1
  strategy:
    type: Recreate           # ← Critical for RWO!
  selector:
    matchLabels:
      app: notes
  template:
    metadata:
      labels:
        app: notes
    spec:
      containers:
      - name: app
        image: busybox:1.36
        command:
        - sh
        - -c
        - |
          echo "===== Pod started at $(date) =====" >> /data/notes.log
          echo "----- Full history -----"
          cat /data/notes.log
          sleep 3600
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: notes-pvc
```

```powershell
kubectl apply -f 04-persistent-app.yaml

# First run
$pod1 = kubectl get pod -l app=notes -o jsonpath='{.items[0].metadata.name}'
kubectl logs $pod1

# Delete pod — Deployment recreates
kubectl delete pod $pod1
Start-Sleep -Seconds 10

# Second run
$pod2 = kubectl get pod -l app=notes -o jsonpath='{.items[0].metadata.name}'
kubectl logs $pod2
# ✅ Logs from BOTH runs visible — data persisted!
```

***

### Lab 5: Premium Storage on AKS

```yaml
# 05-premium-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-data
spec:
  storageClassName: managed-csi-premium
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 50Gi
```

```powershell
kubectl apply -f 05-premium-pvc.yaml
kubectl describe pvc db-data
# Watch Events: ExternalProvisioning → Provisioning → ProvisioningSucceeded
```

***

### Lab 6: Volume Expansion (Online Resize)

```powershell
# Check current size
kubectl get pvc app-data

# Expand from 2Gi to 5Gi
kubectl patch pvc app-data -p '{"spec":{"resources":{"requests":{"storage":"5Gi"}}}}'

# Watch expansion happen ONLINE (no pod restart needed on modern CSI)
kubectl get pvc app-data -w
kubectl describe pvc app-data | Select-String -Pattern "Capacity|Conditions"
```

> ⚠️ **Cannot shrink** PVCs. Plan growth carefully.

***

### Lab 7: Custom StorageClass with `Retain` Policy

```yaml
# 06-retain-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: critical-data-retain
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
  kind: Managed
reclaimPolicy: Retain                    # ← Data safety!
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

```powershell
kubectl apply -f 06-retain-sc.yaml
kubectl get sc critical-data-retain
```

***

### Lab 8: Snapshot & Restore (Production Backup Pattern)

```yaml
# 07-snapshot.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-azuredisk-vsc
driver: disk.csi.azure.com
deletionPolicy: Retain
---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: notes-snap-001
spec:
  volumeSnapshotClassName: csi-azuredisk-vsc
  source:
    persistentVolumeClaimName: notes-pvc
```

```powershell
kubectl apply -f 07-snapshot.yaml
kubectl get volumesnapshot

# Restore from snapshot into a new PVC
$restoreYaml = @"
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: notes-restored
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
  dataSource:
    name: notes-snap-001
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
"@
$restoreYaml | kubectl apply -f -
```

***

### Lab 9: Debug a Stuck PVC

```powershell
# Create a PVC referencing a non-existent StorageClass
$brokenYaml = @"
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: broken-pvc
spec:
  storageClassName: does-not-exist
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
"@
$brokenYaml | kubectl apply -f -

# Debug
kubectl describe pvc broken-pvc
# Events: storageclass.storage.k8s.io "does-not-exist" not found

# Fix and clean up
kubectl delete pvc broken-pvc
```

***

## 💻 Part 3: Command Reference

### Inspecting Storage

```powershell
# Quick overview
kubectl get pv,pvc,sc

# All PVCs cluster-wide
kubectl get pvc -A

# Custom columns for PVs
kubectl get pv -o custom-columns=`
NAME:.metadata.name,`
CAPACITY:.spec.capacity.storage,`
ACCESS:.spec.accessModes,`
RECLAIM:.spec.persistentVolumeReclaimPolicy,`
STATUS:.status.phase,`
CLAIM:.spec.claimRef.name,`
SC:.spec.storageClassName

# StorageClass details
kubectl describe sc managed-csi-premium

# What CSI drivers are installed?
kubectl get csidrivers
```

### Working with Default StorageClass

```powershell
# Find the default
kubectl get sc | Select-String "(default)"

# Set a new default
kubectl patch storageclass managed-csi-premium `
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Unset old default
kubectl patch storageclass managed-csi `
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

### PVC Operations

```powershell
# Expand (online)
kubectl patch pvc <name> -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'

# Force-delete a stuck PVC
kubectl patch pvc <name> -p '{"metadata":{"finalizers":null}}'
kubectl delete pvc <name>

# Reclaim a Released PV
kubectl patch pv <name> -p '{"spec":{"claimRef":null}}'
```

### Snapshots

```powershell
kubectl get volumesnapshot
kubectl get volumesnapshotclass
kubectl get volumesnapshotcontent
```

***

## 🎤 Part 4: Interview Prep

### ⭐ Star Question: *"Explain the binding lifecycle of a PVC."*

**Senior DevOps model answer:**

> "A PVC goes through **five phases** in Kubernetes: provisioning, binding, using, releasing, and reclaiming.
>
> **Provisioning** can be **static** — where a cluster admin manually creates PV objects backed by pre-existing storage like an NFS export or an Azure Disk — or **dynamic**, which is the modern norm. In dynamic provisioning, the PVC references a `StorageClass`, and when the PVC is created, the StorageClass's provisioner (a CSI driver like `disk.csi.azure.com` on AKS) calls the cloud API to create the actual storage and the PV object representing it.
>
> **Binding** is the matchmaking step. Kubernetes pairs a PVC with a suitable PV based on four criteria: `storageClassName` matches, the PV's `accessModes` satisfy what the PVC requests, the PV's capacity is at least what the PVC asks for, and any label selectors match. Binding is **1:1** — once a PV is bound to a PVC, no other PVC can claim it. The PVC moves from `Pending` to `Bound`; the PV from `Available` to `Bound`.
>
> Timing here depends on `volumeBindingMode`. **Immediate** binds at PVC creation, but in multi-zone AKS that's dangerous — the PV might be created in `eastus-1`, and the Pod scheduled to `eastus-2`, causing mount failure. So Azure StorageClasses default to **WaitForFirstConsumer**, which delays provisioning until a Pod actually consumes the PVC. The scheduler then picks a node, and the CSI driver provisions the disk in that node's zone.
>
> **Using** is when the Pod is scheduled and the CSI driver attaches the volume to the node, then mounts it into the container. Kubernetes adds finalizers — `kubernetes.io/pvc-protection` and `kubernetes.io/pv-protection` — so the PVC and PV can't be deleted while in use.
>
> **Releasing** happens when the Pod terminates. The volume is unmounted from the node. If the user deletes the PVC, the PV transitions to `Released`, but the actual storage is still intact at this point.
>
> Finally, **reclaiming** is governed by `persistentVolumeReclaimPolicy`. **Retain** keeps the PV and underlying storage — perfect for production databases where accidental deletion shouldn't lose data; the admin must manually clean up. **Delete** removes both the PV object and the cloud storage — fine for dev/test. **Recycle** is deprecated. For dynamic provisioning, `Delete` is the default — which I always override to `Retain` for critical workloads via custom StorageClasses.
>
> In production I always combine `Retain` + `WaitForFirstConsumer` + `allowVolumeExpansion: true` + Velero scheduled backups + tested restore runbooks. That's the full picture of PVC lifecycle for a senior role."

***

### 🔥 15 Likely Follow-Ups

| #  | Question                                                 | Quick Answer                                                                                                                 |
| -- | -------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| 1  | **PV vs PVC?**                                           | PV = cluster-scoped storage resource (supply). PVC = namespaced request (demand).                                            |
| 2  | **Why does dynamic provisioning need a StorageClass?**   | StorageClass is the template that tells the CSI driver *what kind* of storage to provision.                                  |
| 3  | **Why does Azure Disk only support RWO?**                | Block storage; Azure Disk can only attach to one VM at a time. For RWX use Azure Files (SMB/NFS).                            |
| 4  | **What's WaitForFirstConsumer?**                         | Delays PV creation until a Pod schedules — ensures PV is in same zone as Pod.                                                |
| 5  | **Can you shrink a PVC?**                                | No — only expand. Plan ahead.                                                                                                |
| 6  | **What if I delete a PVC with `Retain` policy?**         | PV becomes `Released`, data intact. Reclaim by clearing `claimRef` and recreating PVC with `volumeName`.                     |
| 7  | **What if I delete with `Delete` policy?**               | PV deleted + underlying storage **wiped**. Recovery only via snapshot/backup.                                                |
| 8  | **Difference between hostPath and PVC?**                 | hostPath = node-local, breaks on reschedule, security risk. PVC = cluster-managed, survives anywhere.                        |
| 9  | **How do snapshots work?**                               | `VolumeSnapshot` + `VolumeSnapshotClass` (CSI feature). Restore via `dataSource` in new PVC spec.                            |
| 10 | **What's PVC protection?**                               | Finalizer that prevents PVC deletion while a Pod uses it.                                                                    |
| 11 | **Can two pods share a PVC?**                            | If accessMode is RWX, yes. If RWO, only if pods are on the same node (not recommended).                                      |
| 12 | **What is CSI?**                                         | Container Storage Interface — vendor-neutral plugin model for storage. Replaces in-tree drivers.                             |
| 13 | **What's a CSI inline ephemeral volume?**                | Volume defined inline in pod spec (no PVC), provisioned per-pod. Used for secrets/configs from CSI drivers.                  |
| 14 | **How do you back up volumes?**                          | Velero (cluster + PVCs), CSI snapshots (storage layer), or app-level (pg\_dump, mongodump). Best practice: layer all three.  |
| 15 | **Difference between Azure Files & Azure NetApp Files?** | Azure Files = SMB/NFS, general-purpose, RWX. Azure NetApp Files = high-perf NFS for SAP HANA, EDA workloads, sub-ms latency. |

***

### 💡 Senior Scenarios

**Scenario 1: "PVC stuck Pending for 15 minutes on AKS"**

> Step-by-step:
>
> 1. `kubectl describe pvc <name>` — read Events carefully.
> 2. Check StorageClass exists: `kubectl get sc`.
> 3. CSI driver healthy: `kubectl get pods -n kube-system | grep csi-azuredisk`.
> 4. Check AKS managed identity has `Contributor` on node resource group.
> 5. Azure subscription disk quota (`az quota show`).
> 6. If `WaitForFirstConsumer`, verify a Pod is actually trying to mount.
> 7. CSI provisioner logs: `kubectl logs -n kube-system csi-azuredisk-controller-* -c csi-provisioner`.

**Scenario 2: "Need RWX for shared content across 10 pods"**

> Use `azurefile-csi-premium` StorageClass. Azure Files supports NFS 4.1 or SMB 3.x with concurrent writers. Set up:
>
> * StorageClass with `accountTier: Premium_LRS`, `protocol: nfs`
> * PVC with `accessModes: [ReadWriteMany]`
> * Mount the same PVC in all 10 pods
> * For latency-sensitive workloads, consider **Azure NetApp Files** instead.

**Scenario 3: "Accidentally deleted prod database PVC. Reclaim was set to Delete."**

> 1. **Stop the bleeding** — pause the Deployment to prevent new pod from starting (which would create empty PVC).
> 2. Check Azure Backup Vault for recent snapshots of the disk.
> 3. Check if any VolumeSnapshots survived: `kubectl get volumesnapshot`.
> 4. Check Azure Soft Delete on the resource group (if enabled, 14-day window).
> 5. Restore from Velero scheduled backup.
> 6. **Post-mortem actions**:
>    * Change all prod StorageClasses to `reclaimPolicy: Retain`.
>    * RBAC: revoke `delete` on PVCs for non-admin users.
>    * Add Kyverno policy blocking deletion of PVCs labeled `criticality=tier-1`.
>    * Enable Velero with 4-hour backup cadence + tested DR runbook.

**Scenario 4: "Database pod won't start — Multi-Attach error"**

> Symptom: `Multi-Attach error for volume "pvc-xxx" Volume is already used by pod(s) ...`
>
> Cause: Old pod is still holding the RWO volume on a different node. Common after node failure or strategy=RollingUpdate on a Deployment with PVC.
>
> Fix:
>
> 1. Check old pod status: `kubectl get pods -o wide`.
> 2. If old pod is `Terminating` indefinitely — node likely unhealthy. Force-delete: `kubectl delete pod <name> --grace-period=0 --force`.
> 3. Long-term: switch to `strategy: Recreate` for single-replica DBs, or use **StatefulSet** (Day 9) which handles this properly.

**Scenario 5: "We're moving from EKS to AKS. How do we migrate 50 PVCs of data?"**

> Multi-step strategy:
>
> 1. **App-level replication** where possible — PostgreSQL streaming, MongoDB replica set, Kafka MirrorMaker. Lowest downtime.
> 2. **Velero with restic** — back up EKS PVCs to Azure Blob, restore to AKS. Storage-class translation via `--change-storage-class` map.
> 3. **AzCopy / rsync** for file-based — mount source PVC in EKS pod with AzCopy, push to Azure Files, mount in AKS.
> 4. **Cloud-to-cloud snapshot** — not native; requires intermediate object storage.
> 5. Always do a **dry run on non-prod first**, document RTO/RPO, test restore.

***

## 🧠 Production Patterns Cheat Sheet

| Pattern                  | Recipe                                                                                        |
| ------------------------ | --------------------------------------------------------------------------------------------- |
| **Production DB on AKS** | StatefulSet + managed-csi-premium + Retain policy + Velero hourly backups + Premium snapshots |
| **Shared file storage**  | azurefile-csi-premium + RWX + mount in multiple pods                                          |
| **High IOPS**            | Premium SSD v2 or Ultra Disk via custom StorageClass                                          |
| **Multi-zone HA**        | WaitForFirstConsumer + zone-redundant LRS → ZRS                                               |
| **Cost-optimized dev**   | managed-csi (Standard SSD) + Delete policy                                                    |
| **Critical data safety** | Retain policy + Snapshot CronJob + offsite Velero replication                                 |

***

# 🗃️ Day 9: StatefulSets & DaemonSets

Welcome to Day 9, ! Yesterday you mastered storage. Today you'll learn the workloads that **actually use** that storage in production — **StatefulSets** for databases and stateful apps, and **DaemonSets** for cluster-wide agents. These are the workloads that separate junior K8s users from senior engineers, because they require deep understanding of identity, ordering, and node-level operations. Let's go! 🚀

***

## 📚 Part 1: Theory — Beyond Deployments

### Why Deployments Aren't Enough

Deployments are perfect for **stateless** workloads (web servers, APIs). But they have **critical limitations** for stateful systems:

| Aspect               | Deployment                | Reality of Databases                             |
| -------------------- | ------------------------- | ------------------------------------------------ |
| **Pod names**        | Random (`web-7d4f8-x9k2`) | Replica needs stable name (`mysql-0`, `mysql-1`) |
| **Network identity** | Pod IPs change            | Replication requires stable DNS                  |
| **Storage**          | All pods share PVC        | Each replica needs **its own** disk              |
| **Startup order**    | All at once               | Master must start before replicas                |
| **Scaling**          | Random pod terminated     | Must remove highest-ordinal pod last             |

Enter **StatefulSets** and **DaemonSets** — purpose-built for these scenarios.

***

## 🏛️ StatefulSets Deep Dive

A **StatefulSet** is a workload controller that manages stateful applications, providing guarantees that Deployments don't.

### 🎯 The Five Guarantees

```
1️⃣  STABLE, UNIQUE NETWORK IDENTITY
    Pod names: mysql-0, mysql-1, mysql-2 (predictable)
    DNS: mysql-0.mysql-headless.default.svc.cluster.local

2️⃣  STABLE, PERSISTENT STORAGE
    Each replica gets its OWN PVC
    data-mysql-0, data-mysql-1, data-mysql-2
    PVCs survive pod restarts AND StatefulSet deletion

3️⃣  ORDERED, GRACEFUL DEPLOYMENT
    Pods created sequentially: 0 → 1 → 2
    Each must be Running + Ready before next starts

4️⃣  ORDERED, GRACEFUL SCALING
    Scale down in REVERSE: 2 → 1 → 0
    Each terminated before next

5️⃣  ORDERED, AUTOMATED ROLLING UPDATES
    Update from highest ordinal down: 2 → 1 → 0
```

***

### 🏗️ Architecture Anatomy

```
┌───────────────────────────────────────────────────────────────┐
│                       StatefulSet: mysql                       │
│                                                                │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐       │
│  │  mysql-0    │    │  mysql-1    │    │  mysql-2    │       │
│  │  (master)   │    │  (replica)  │    │  (replica)  │       │
│  │             │    │             │    │             │       │
│  │  PVC:       │    │  PVC:       │    │  PVC:       │       │
│  │  data-      │    │  data-      │    │  data-      │       │
│  │  mysql-0    │    │  mysql-1    │    │  mysql-2    │       │
│  └─────────────┘    └─────────────┘    └─────────────┘       │
│         ▲                  ▲                  ▲                │
│         │                  │                  │                │
│         └──────────┬───────┴──────────────────┘                │
│                    │                                            │
│         ┌──────────▼──────────┐                                │
│         │ Headless Service:    │ ← clusterIP: None             │
│         │ mysql-headless       │ ← DNS returns pod IPs         │
│         └──────────────────────┘                                │
└───────────────────────────────────────────────────────────────┘

DNS Records (auto-created):
  mysql-0.mysql-headless.default.svc.cluster.local → Pod IP of mysql-0
  mysql-1.mysql-headless.default.svc.cluster.local → Pod IP of mysql-1
  mysql-2.mysql-headless.default.svc.cluster.local → Pod IP of mysql-2
```

***

### 🌐 Headless Service — The Magic Behind Stable DNS

A **headless Service** has `clusterIP: None`. Instead of load-balancing, it returns **individual pod DNS records**.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
spec:
  clusterIP: None              # ← headless!
  selector:
    app: mysql
  ports:
  - port: 3306
```

This is **required** for StatefulSets — it's how `mysql-0.mysql-headless` works.

***

### 💾 volumeClaimTemplates — Per-Pod Storage

The biggest StatefulSet trick: instead of a shared PVC, you declare a **template** that K8s instantiates per replica.

```yaml
volumeClaimTemplates:
- metadata:
    name: data
  spec:
    accessModes: [ReadWriteOnce]
    resources:
      requests:
        storage: 10Gi
```

Kubernetes auto-creates:

* `data-mysql-0` (10Gi)
* `data-mysql-1` (10Gi)
* `data-mysql-2` (10Gi)

> 🎯 **Senior gotcha**: PVCs created by `volumeClaimTemplates` are **NOT** automatically deleted when you delete the StatefulSet. This is **intentional** — data safety. You must delete PVCs manually.

***

### 🔄 Pod Management Policies

| Policy                   | Behavior                                                                   |
| ------------------------ | -------------------------------------------------------------------------- |
| `OrderedReady` (default) | Pods created/deleted **strictly in order** (0 → 1 → 2)                     |
| `Parallel`               | Pods created/deleted **all at once** (faster, but unsafe for ordered apps) |

```yaml
spec:
  podManagementPolicy: Parallel    # only when order doesn't matter
```

***

### 🔁 Update Strategies

| Strategy                       | Behavior                                                           |
| ------------------------------ | ------------------------------------------------------------------ |
| `RollingUpdate` (default)      | Update one pod at a time, **highest ordinal first**                |
| `OnDelete`                     | Manual — update template, then delete pods one-by-one for new spec |
| `RollingUpdate + partition: N` | Only update pods with ordinal ≥ N (canary on `mysql-2` only)       |

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 2     # only update mysql-2 and higher
```

> 💡 **Canary deploy in StatefulSet**: set partition to N-1, only the last pod updates. Validate, then drop partition to 0 for full rollout.

***

### ⚠️ StatefulSet Limitations to Know

* ❌ **No automatic data backup** — you must handle this
* ❌ **No replication logic** — StatefulSet provides identity, not data sync
* ❌ **PVC deletion is manual** after StatefulSet delete
* ❌ **Scaling down doesn't migrate data** — losing pods can lose data
* ❌ **Storage class must support dynamic provisioning** (or pre-provision PVs)

***

## 🛡️ DaemonSets Deep Dive

A **DaemonSet** ensures **one pod runs on every (or selected) node** in the cluster. Perfect for **node-level agents**.

```
┌────────────────────────────────────────────────────────┐
│                  CLUSTER                                │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │   Node 1     │  │   Node 2     │  │   Node 3     │ │
│  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │ │
│  │ │ Fluentd  │ │  │ │ Fluentd  │ │  │ │ Fluentd  │ │ │
│  │ │ Pod      │ │  │ │ Pod      │ │  │ │ Pod      │ │ │
│  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
│                                                         │
│  Add Node 4 → DaemonSet automatically deploys here     │
└────────────────────────────────────────────────────────┘
```

### 🎯 Classic DaemonSet Use Cases

| Use Case            | Example                                 |
| ------------------- | --------------------------------------- |
| **Log shipping**    | Fluentd, Fluent Bit, Filebeat, Promtail |
| **Node monitoring** | Prometheus node-exporter, Datadog agent |
| **Network (CNI)**   | Calico, Cilium, Azure CNI               |
| **Storage drivers** | CSI plugins (Azure Disk, EBS CSI)       |
| **Security**        | Falco, Aqua, Trivy operator             |
| **Service mesh**    | Linkerd-proxy, Cilium                   |

### 🎛️ Targeting Specific Nodes

By default, DaemonSets run on **all schedulable nodes**. Use `nodeSelector` or `affinity` to target subsets:

```yaml
spec:
  template:
    spec:
      nodeSelector:
        gpu: "true"           # only nodes labeled gpu=true
      
      # OR use affinity for advanced rules
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: workload-type
                operator: In
                values: [logging, monitoring]
      
      # Tolerate taints so DaemonSet runs even on tainted nodes
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      - operator: Exists      # tolerate ALL taints
```

> 🎯 **Senior tip**: Almost all DaemonSets need `tolerations: operator: Exists` so they run on master nodes, GPU nodes, and other tainted nodes too.

### 🔄 DaemonSet Update Strategies

| Strategy                  | Behavior                               |
| ------------------------- | -------------------------------------- |
| `RollingUpdate` (default) | Update one node at a time              |
| `OnDelete`                | Manual — delete pods to trigger update |

***

## 🛠️ Part 2: Hands-On — Real Workloads

### Lab 1: Simple StatefulSet (NGINX with stable identity)

```yaml
# 01-nginx-statefulset.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-headless
  labels:
    app: nginx-sts
spec:
  clusterIP: None
  selector:
    app: nginx-sts
  ports:
  - port: 80
    name: web
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: nginx-headless    # must match headless service name
  replicas: 3
  selector:
    matchLabels:
      app: nginx-sts
  template:
    metadata:
      labels:
        app: nginx-sts
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ReadWriteOnce]
      resources:
        requests:
          storage: 1Gi
```

```powershell
kubectl apply -f 01-nginx-statefulset.yaml

# Watch ordered creation: web-0 → web-1 → web-2
kubectl get pods -l app=nginx-sts -w

# Verify per-pod PVCs created automatically
kubectl get pvc
# www-web-0, www-web-1, www-web-2

# Test stable DNS from another pod
kubectl run debug --image=busybox:1.36 -it --rm --restart=Never -- sh
# Inside:
nslookup web-0.nginx-headless
nslookup web-1.nginx-headless
nslookup nginx-headless          # returns ALL pod IPs (headless)
exit

# Write unique data to each pod
for ($i = 0; $i -lt 3; $i++) {
    kubectl exec web-$i -- sh -c "echo 'Hello from web-$i' > /usr/share/nginx/html/index.html"
}

# Delete a pod and verify data survives
kubectl delete pod web-1
Start-Sleep -Seconds 15

# Same pod NAME comes back with SAME data
kubectl exec web-1 -- cat /usr/share/nginx/html/index.html
# ✅ "Hello from web-1" — stable identity + persistent data!
```

***

### Lab 2: PostgreSQL StatefulSet (The Real Deal)

This is your **assignment** — a 3-replica Postgres with persistent storage.

#### Step 1: Namespace + Secret

```yaml
# 02-postgres-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: db
  labels:
    purpose: database
---
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: db
type: Opaque
stringData:
  POSTGRES_USER: pgadmin
  POSTGRES_PASSWORD: SuperSecure2026!
  POSTGRES_DB: appdb
  POSTGRES_REPLICATION_PASSWORD: ReplicaPass2026!
```

#### Step 2: ConfigMap

```yaml
# 03-postgres-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: db
data:
  POSTGRES_INITDB_ARGS: "--encoding=UTF8 --locale=C"
  PGDATA: /var/lib/postgresql/data/pgdata
  postgresql.conf: |
    listen_addresses = '*'
    max_connections = 100
    shared_buffers = 256MB
    effective_cache_size = 1GB
    work_mem = 4MB
    maintenance_work_mem = 64MB
    wal_level = replica
    max_wal_senders = 10
    max_replication_slots = 10
```

#### Step 3: Headless Service

```yaml
# 04-postgres-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: db
  labels:
    app: postgres
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
  - port: 5432
    name: postgres
---
# Regular ClusterIP for app connections (load-balanced)
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: db
spec:
  type: ClusterIP
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
```

#### Step 4: StatefulSet

```yaml
# 05-postgres-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: db
spec:
  serviceName: postgres-headless
  replicas: 3
  podManagementPolicy: OrderedReady
  updateStrategy:
    type: RollingUpdate
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      terminationGracePeriodSeconds: 30
      containers:
      - name: postgres
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
          name: postgres
        envFrom:
        - secretRef:
            name: postgres-secret
        - configMapRef:
            name: postgres-config
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            cpu: 250m
            memory: 512Mi
          limits:
            cpu: 1
            memory: 1Gi
        readinessProbe:
          exec:
            command:
            - sh
            - -c
            - pg_isready -U $POSTGRES_USER -d $POSTGRES_DB
          initialDelaySeconds: 15
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        livenessProbe:
          exec:
            command:
            - sh
            - -c
            - pg_isready -U $POSTGRES_USER
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 6
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ReadWriteOnce]
      resources:
        requests:
          storage: 5Gi
      # storageClassName: managed-csi-premium   # on AKS for prod
```

#### Deploy and Verify

```powershell
kubectl apply -f 02-postgres-namespace.yaml
kubectl apply -f 03-postgres-config.yaml
kubectl apply -f 04-postgres-service.yaml
kubectl apply -f 05-postgres-statefulset.yaml

# Watch ordered startup
kubectl get pods -n db -w
# postgres-0 → Running+Ready
# postgres-1 → starts AFTER 0 is ready
# postgres-2 → starts AFTER 1 is ready

# Verify per-pod PVCs
kubectl get pvc -n db
# data-postgres-0 (5Gi, Bound)
# data-postgres-1 (5Gi, Bound)
# data-postgres-2 (5Gi, Bound)

# Verify stable DNS
kubectl run dbg --image=busybox:1.36 -it --rm --restart=Never -n db -- sh
# Inside:
nslookup postgres-0.postgres-headless
nslookup postgres-1.postgres-headless
nslookup postgres-2.postgres-headless
exit
```

#### Test Data Persistence

```powershell
# Connect to postgres-0 and create data
kubectl exec -it postgres-0 -n db -- psql -U pgadmin -d appdb -c `
  "CREATE TABLE customers (id SERIAL PRIMARY KEY, name VARCHAR(50)); INSERT INTO customers (name) VALUES (''), ('Nikhil');"

# Verify data
kubectl exec -it postgres-0 -n db -- psql -U pgadmin -d appdb -c "SELECT * FROM customers;"

# Delete postgres-0 and watch it recreate
kubectl delete pod postgres-0 -n db
kubectl get pods -n db -w
# Same name comes back: postgres-0 (with same PVC)

# Verify data PERSISTED
kubectl exec -it postgres-0 -n db -- psql -U pgadmin -d appdb -c "SELECT * FROM customers;"
# ✅ Customers still there!
```

#### Test Scaling

```powershell
# Scale up to 4
kubectl scale statefulset postgres -n db --replicas=4
kubectl get pods -n db -w
# postgres-3 created LAST (ordered)

# Scale down to 2
kubectl scale statefulset postgres -n db --replicas=2
kubectl get pods -n db -w
# postgres-3 deleted FIRST, then postgres-2 (REVERSE order)

# ⚠️ NOTE: PVCs of scaled-down pods are NOT deleted (data safety!)
kubectl get pvc -n db
# All 4 PVCs still exist
```

#### Cleanup

```powershell
# Deleting StatefulSet leaves PVCs (intentional!)
kubectl delete statefulset postgres -n db

# To delete PVCs too (data goes!)
kubectl delete pvc -l app=postgres -n db

# Or nuke the whole namespace
kubectl delete namespace db
```

***

### Lab 3: DaemonSet — Node Logger (Fluent Bit Style)

```yaml
# 06-fluentbit-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: kube-system
  labels:
    app: fluent-bit
spec:
  selector:
    matchLabels:
      app: fluent-bit
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: fluent-bit
    spec:
      # Run on every node — including masters and tainted nodes
      tolerations:
      - operator: Exists
        effect: NoSchedule
      - operator: Exists
        effect: NoExecute
      
      # Don't use cluster DNS for itself
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:2.2
        resources:
          requests:
            cpu: 50m
            memory: 100Mi
          limits:
            cpu: 200m
            memory: 256Mi
        volumeMounts:
        # Read container logs from the node
        - name: varlog
          mountPath: /var/log
          readOnly: true
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

```powershell
kubectl apply -f 06-fluentbit-daemonset.yaml

# Verify ONE pod per node
kubectl get pods -n kube-system -l app=fluent-bit -o wide
kubectl get nodes
# Pod count should == Node count
```

***

### Lab 4: Targeted DaemonSet (GPU Nodes Only)

```yaml
# 07-gpu-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nvidia-device-plugin
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: nvidia-plugin
  template:
    metadata:
      labels:
        app: nvidia-plugin
    spec:
      # Only schedule on GPU nodes
      nodeSelector:
        accelerator: nvidia-gpu
      
      tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
      
      containers:
      - name: plugin
        image: nvcr.io/nvidia/k8s-device-plugin:v0.14.0
        securityContext:
          privileged: true
```

```powershell
# Label a node as GPU (for testing)
kubectl label node <node-name> accelerator=nvidia-gpu

# Apply
kubectl apply -f 07-gpu-daemonset.yaml

# Verify it only runs on labeled nodes
kubectl get pods -n kube-system -l app=nvidia-plugin -o wide
```

***

### Lab 5: DaemonSet Rolling Update

```powershell
# Update the image of an existing DaemonSet
kubectl set image daemonset/fluent-bit -n kube-system fluent-bit=fluent/fluent-bit:2.3

# Watch rolling update (one node at a time)
kubectl rollout status daemonset/fluent-bit -n kube-system

# History & rollback
kubectl rollout history daemonset/fluent-bit -n kube-system
kubectl rollout undo daemonset/fluent-bit -n kube-system
```

***

## 💻 Part 3: Commands Cheat Sheet

### StatefulSet

```powershell
# Create / Update
kubectl apply -f sts.yaml

# Inspect
kubectl get statefulset
kubectl get sts
kubectl describe sts postgres -n db
kubectl get pods -l app=postgres -n db

# Scale
kubectl scale sts postgres -n db --replicas=5

# Rollout
kubectl rollout status sts/postgres -n db
kubectl rollout history sts/postgres -n db
kubectl rollout undo sts/postgres -n db

# Partition update (canary)
kubectl patch sts postgres -n db -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":2}}}}'

# Force restart all pods
kubectl rollout restart sts/postgres -n db

# Delete (PVCs NOT deleted — data safe)
kubectl delete sts postgres -n db

# Delete PVCs too (DESTROYS DATA)
kubectl delete pvc -l app=postgres -n db
```

### DaemonSet

```powershell
# Create / Update
kubectl apply -f ds.yaml

# Inspect
kubectl get daemonset -A
kubectl get ds -n kube-system
kubectl describe ds fluent-bit -n kube-system

# Pods per node
kubectl get pods -l app=fluent-bit -o wide

# Rollout
kubectl rollout status ds/fluent-bit -n kube-system
kubectl rollout undo ds/fluent-bit -n kube-system

# Delete
kubectl delete ds fluent-bit -n kube-system
```

### Debugging

```powershell
# StatefulSet stuck pod
kubectl describe pod postgres-1 -n db
kubectl logs postgres-1 -n db --previous

# Check PVC binding for StatefulSet pods
kubectl get pvc -l app=postgres -n db

# Why isn't DaemonSet pod running on a node?
kubectl describe node <node-name>     # check taints
kubectl get pods -A --field-selector spec.nodeName=<node-name>
```

***

## 🎤 Part 4: Interview Prep

### ⭐ Star Question: *"Why use StatefulSet over Deployment for databases?"*

**Senior DevOps model answer:**

> "Databases have requirements that **Deployments fundamentally can't satisfy**, so we use StatefulSets for five key reasons.
>
> **First, stable network identity**. Deployment pods get random suffixes — `postgres-7d4f8-x9k2` — and these change on every restart. A database replica needs a **predictable hostname** because replication is configured with peer addresses. With StatefulSet, pods get ordinal names: `postgres-0`, `postgres-1`, `postgres-2`. Combined with a headless Service, you get stable DNS like `postgres-0.postgres-headless.db.svc.cluster.local`. The replica configuration file can hardcode this — it survives any pod restart.
>
> **Second, dedicated persistent storage per replica**. With a Deployment, all pods share the same PVC reference, but a PVC of type RWO can only be mounted on one node — so you can't actually scale a Deployment with a PVC. Even with RWX, a database can't have two writers to the same files. StatefulSets solve this with `volumeClaimTemplates` — Kubernetes auto-creates a **dedicated PVC per replica**: `data-postgres-0`, `data-postgres-1`, `data-postgres-2`. Each replica has its own disk for its own data.
>
> **Third, ordered startup**. Database clusters typically need a leader/primary to start first, then replicas connect to it. Deployments start all pods in parallel — there's no concept of order. StatefulSets default to `OrderedReady` — `postgres-0` must be Running AND Ready before `postgres-1` starts, and so on. This matches the bootstrap requirements of Postgres replication, MongoDB replica sets, Kafka brokers, and ZooKeeper ensembles.
>
> **Fourth, ordered scaling and updates in reverse**. When you scale down from 5 to 3, StatefulSet terminates `postgres-4` first, then `postgres-3`. This protects the leader (typically the lowest ordinal) and lets the application drain replicas gracefully. Updates work the same way — `postgres-2` is updated first; if it crashes, you stop the rollout before touching the leader.
>
> **Fifth, data safety on deletion**. When you delete a StatefulSet, **PVCs are intentionally retained**. Compare this to Deployments where there's no PVC association at all. This prevents accidental data loss — even if someone runs `kubectl delete sts postgres`, the data on `data-postgres-0` etc. survives.
>
> Now, an important caveat I always mention: **StatefulSet provides identity and storage; it does NOT provide replication or data consistency**. You still need application-level setup — Postgres streaming replication, MongoDB replica config, etcd raft. For production, I typically use **operators** like Zalando's postgres-operator, CrunchyData PGO, or Percona for MySQL — they wrap StatefulSets with replication, failover, backups, and PITR. StatefulSet is the building block; the operator is the brain."

***

### 🔥 Common Follow-Ups

| #  | Question                                                                      | Quick Answer                                                                                                                                                                           |
| -- | ----------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1  | **What's a headless Service?**                                                | Service with `clusterIP: None`. DNS returns individual pod IPs, not a load-balanced VIP. Required for StatefulSet stable DNS.                                                          |
| 2  | **Difference between StatefulSet and Deployment update strategy?**            | Deployment: parallel rolling update with surge. StatefulSet: one-at-a-time, **highest ordinal first**.                                                                                 |
| 3  | **What's `podManagementPolicy: Parallel`?**                                   | Skips ordered startup — all pods created at once. Use only when order doesn't matter (like cache layers).                                                                              |
| 4  | **Are PVCs deleted when StatefulSet is deleted?**                             | **No** — intentionally retained for data safety. Delete manually with `kubectl delete pvc -l <selector>`. K8s 1.27+ adds `persistentVolumeClaimRetentionPolicy` for automated cleanup. |
| 5  | **How do you do a canary update on StatefulSet?**                             | Use `updateStrategy.rollingUpdate.partition: N`. Only pods with ordinal ≥ N get the new spec. Validate, then drop partition to 0.                                                      |
| 6  | **Can you have HPA on a StatefulSet?**                                        | Yes, but rarely useful — most stateful apps scale by data, not load. Manual scaling is preferred.                                                                                      |
| 7  | **What happens if a StatefulSet pod's PVC gets corrupted?**                   | Delete the pod AND its PVC, then it'll recreate fresh. Restore from backup. PVC deletion is the only "reset" path.                                                                     |
| 8  | **Why does my StatefulSet pod stay in Pending?**                              | Usually PVC binding issues. Check `kubectl describe pod`, `kubectl get pvc`. Often: no StorageClass, no default SC, RWO zone mismatch.                                                 |
| 9  | **What's the difference between DaemonSet and Deployment with nodeSelector?** | DaemonSet automatically reacts to **node add/remove**. Deployment doesn't — you'd manually scale to match nodes.                                                                       |
| 10 | **How does a DaemonSet handle node taints?**                                  | Pods need matching `tolerations`. Common pattern: `tolerations: [{operator: Exists}]` to run on all nodes including masters.                                                           |
| 11 | **Can you update a DaemonSet without affecting all nodes at once?**           | Yes — `updateStrategy: RollingUpdate` with `maxUnavailable: 1` means one node at a time.                                                                                               |
| 12 | **Why is `hostNetwork: true` common in DaemonSets?**                          | Network plugins (CNI), node-level monitoring, kube-proxy — they need direct access to the node's network stack.                                                                        |
| 13 | **What does kubelet run as — a DaemonSet?**                                   | **No** — kubelet is a systemd service on each node, not a K8s resource. kube-proxy IS a DaemonSet though.                                                                              |
| 14 | **Why might you choose `Parallel` over `OrderedReady` for a StatefulSet?**    | When pods don't depend on each other — like a Redis cluster with gossip discovery, or stateless shards. Faster scaling.                                                                |
| 15 | **How do you back up a StatefulSet?**                                         | Layered: (1) app-level dumps (`pg_dump`) on a sidecar/CronJob, (2) CSI VolumeSnapshots of PVCs, (3) Velero for cluster-level backups with PVC snapshot integration.                    |

***

### 💡 Senior Scenarios

**Scenario 1: "Your Postgres StatefulSet pod `postgres-1` is stuck in Pending. How do you debug?"**

> Methodical approach:
>
> 1. `kubectl describe pod postgres-1 -n db` — read Events.
> 2. Common cause #1: **PVC binding stuck**. `kubectl get pvc data-postgres-1 -n db` — is it Pending?
> 3. Common cause #2: **Insufficient resources** — check node capacity vs requested CPU/memory.
> 4. Common cause #3: **OrderedReady waiting** — `postgres-0` must be Ready first. Is it?
> 5. Common cause #4: **Zone affinity** — PVC provisioned in a zone with no available nodes. Solution: WaitForFirstConsumer SC.
> 6. Common cause #5: **Taints with no toleration** — `kubectl describe node` and check.
> 7. Common cause #6: **Headless Service missing** — StatefulSet's `serviceName` must point to an existing headless Service.
> 8. Fix the underlying issue and the pod auto-recovers.

***

**Scenario 2: "You scale down your StatefulSet from 5 to 3, then back to 5. What happens to data on `postgres-4`?"**

> When scaling down, `postgres-4` and `postgres-3` are deleted, but their PVCs (`data-postgres-4`, `data-postgres-3`) **persist**.
>
> When scaling back up, K8s creates `postgres-3` and `postgres-4` again — and crucially, they **reattach to the existing PVCs** with the original data intact.
>
> This is a key data safety feature. If you actually want to wipe the data, you must explicitly `kubectl delete pvc data-postgres-3 data-postgres-4`.
>
> In K8s 1.27+, you can configure `persistentVolumeClaimRetentionPolicy.whenScaled: Delete` if you want automatic PVC cleanup on scale-down — but I always leave this as `Retain` for production databases.

***

**Scenario 3: "Your DaemonSet logging agent is killing nodes by consuming too much memory. What's your fix?"**

> Multi-step response:
>
> 1. **Immediate**: Add `resources.limits.memory` to cap usage. Set `requests` so scheduler accounts for it.
> 2. **Prevent OOM cascades**: Set a `priorityClass` lower than critical workloads — DaemonSet gets evicted first under pressure.
> 3. **QoS class**: Setting `requests == limits` makes it `Guaranteed` QoS (not first to evict). Setting `requests < limits` makes it `Burstable`. For agents, Burstable with reasonable limits is usually right.
> 4. **Configure the agent**: tune Fluent Bit's `Mem_Buf_Limit`, flush intervals, retry buffers.
> 5. **Persistent buffer**: have it spill to local disk instead of RAM under backpressure.
> 6. **Reduce noisy logs**: filter at source — most log floods are from chatty containers.
> 7. **Monitor**: alert when DaemonSet pods restart frequently (OOMKilled count).

***

**Scenario 4: "How do you do a zero-downtime major Postgres version upgrade on a StatefulSet?"**

> StatefulSet rolling update **alone won't work** — Postgres 14 → 15 needs `pg_upgrade` on data files; just changing the image causes the new pod to fail to start.
>
> Production approach:
>
> 1. **Backup first** — `pg_dump` of all DBs, plus CSI VolumeSnapshot of every PVC.
> 2. **Build a new StatefulSet** with v15 image and **new PVCs** (different name).
> 3. **Use logical replication** (built into Postgres 10+) — set up the new cluster as a subscriber to the old.
> 4. **Sync data** until both are caught up.
> 5. **Cut over** — update the Service selector to point at new StatefulSet pods.
> 6. **Monitor**, then decommission the old StatefulSet.
>
> Alternative: use a Postgres operator like Zalando's, which handles version upgrades via planned maintenance windows with built-in `pg_upgrade` automation.

***

**Scenario 5: "Why might you NOT use a StatefulSet for a database?"**

> Several legitimate reasons:
>
> 1. **Managed cloud DBs are usually better** — Azure Database for PostgreSQL, Azure Cosmos DB, AWS RDS. Less operational burden, built-in HA, backups, patching.
> 2. **External database** — if your DB lives outside K8s entirely, use an `ExternalName` Service to abstract it.
> 3. **K8s operators** wrap StatefulSets with better automation — using the operator (Zalando PGO, Strimzi for Kafka, MongoDB Community Operator) is preferred over raw StatefulSet.
> 4. **Workloads with replication-aware logic** like Kafka, Elasticsearch, Cassandra — use their dedicated operators (Strimzi, ECK, K8ssandra) — they understand the application semantics.
>
> Rule of thumb: **prefer managed service → operator → raw StatefulSet, in that order**.

***

### 🌟 Bonus DaemonSet Scenario

**"You added a new node pool to AKS, but the new nodes don't have your logging agent running. Why?"**

> Likely causes:
>
> 1. **Taints on new node pool** that the DaemonSet doesn't tolerate. Spot pools, GPU pools, system pools often have taints. Check `kubectl describe node <new-node>` → Taints section.
> 2. **NodeSelector mismatch** — DaemonSet has `nodeSelector` that the new nodes don't satisfy.
> 3. **Affinity rule** rejecting them.
> 4. **DaemonSet doesn't reconcile fast enough** — usually within seconds.
>
> Fix: add tolerations for the new pool's taints, or label the nodes to match the nodeSelector. Verify with `kubectl get ds <name> -o wide` → `DESIRED` count should match new node count.

***

## 🧠 Decision Matrix

| Workload                                   | Use                                                     |
| ------------------------------------------ | ------------------------------------------------------- |
| Stateless web app, API                     | **Deployment**                                          |
| Database, message queue, distributed cache | **StatefulSet** (or operator)                           |
| Logging agent on every node                | **DaemonSet**                                           |
| One-off batch task                         | **Job** (Day 10!)                                       |
| Scheduled task                             | **CronJob** (Day 10!)                                   |
| Per-tenant DB instance                     | **StatefulSet per tenant** OR operator                  |
| Cluster-wide CSI driver                    | **DaemonSet**                                           |
| Static-content NGINX cluster               | **Deployment** (not StatefulSet — no need for identity) |

***

# ⏰ Day 10: Jobs & CronJobs

Welcome to Day 10, Rajesh! You've mastered long-running workloads (Deployments, StatefulSets, DaemonSets). But what about **one-off tasks** that should **run, finish, and exit** — database migrations, backups, batch processing, ML training jobs, periodic cleanups? Today you'll master **Jobs** (batch tasks) and **CronJobs** (scheduled tasks) — workloads that power every production cluster's operational pipelines. This is **interview gold** for Senior DevOps roles. 🚀

***

## 📚 Part 1: Theory — Tasks That End

### The Problem with Deployments for Tasks

Deployments are designed for **always-running** workloads. They have `restartPolicy: Always` baked in. So if you run a DB migration as a Deployment:

```
1. Migration container runs migration → succeeds → exits with code 0
2. Kubernetes sees pod exited → restarts it
3. Migration runs AGAIN
4. ❌ Infinite migration loop → corrupts DB → wakes up Rajesh at 3 AM
```

**Jobs** were created exactly for this: run to **completion**, then stop.

***

### 🧱 The Job Family Tree

```
┌────────────────────────────────────────────────────────┐
│             KUBERNETES BATCH WORKLOADS                  │
├────────────────────────────────────────────────────────┤
│                                                         │
│  ┌────────────┐         ┌────────────┐                 │
│  │   JOB      │         │  CRONJOB   │                 │
│  │            │         │            │                 │
│  │ Run once   │ ◄────── │ Schedule   │                 │
│  │ until      │ creates │ Jobs based │                 │
│  │ completion │   Jobs  │ on cron    │                 │
│  └────────────┘         └────────────┘                 │
│                                                         │
└────────────────────────────────────────────────────────┘
```

| Resource    | Purpose                  | Lifecycle                              |
| ----------- | ------------------------ | -------------------------------------- |
| **Job**     | Run a task to completion | One-shot; pod terminates after success |
| **CronJob** | Schedule recurring Jobs  | Creates Jobs on cron schedule          |

***

### 🎯 Job Deep Dive

A **Job** ensures that **a specified number of pods successfully terminate**.

#### Pod RestartPolicy in Jobs

| Policy      | Behavior                                                                   |
| ----------- | -------------------------------------------------------------------------- |
| `OnFailure` | Container restarts in same pod on failure (counted against `backoffLimit`) |
| `Never`     | Pod fails → new pod created on retry (more debuggable, cleaner logs)       |

⚠️ `Always` is **NOT allowed** in Jobs — defeats the purpose.

***

### 🔢 Three Job Execution Patterns

#### Pattern 1: Single Job (Non-Parallel)

```
completions: 1      ← need 1 successful pod
parallelism: 1      ← run 1 at a time
```

**Use case**: DB migration, single-shot data import.

#### Pattern 2: Fixed Completion Count (Parallel)

```
completions: 5      ← need 5 successful pods
parallelism: 2      ← run 2 at a time
```

**Use case**: Process 5 input files in parallel, 2 workers at once.

```
Time T0:   [pod-1 ▶] [pod-2 ▶]
Time T1:   pod-1 ✓ → [pod-3 ▶] [pod-2 ▶]
Time T2:   pod-2 ✓ → [pod-3 ▶] [pod-4 ▶]
Time T3:   ...all done ✓
```

#### Pattern 3: Work Queue (Parallel, No Fixed Completion)

```
completions: not set
parallelism: 5      ← 5 workers all pulling from a queue
```

**Use case**: Queue-based workers (RabbitMQ consumers). Workers exit when queue is empty.

***

### 🆕 Indexed Jobs (K8s 1.24+)

A modern pattern — each pod gets a **unique index** via env var `JOB_COMPLETION_INDEX`.

```yaml
spec:
  completions: 10
  parallelism: 3
  completionMode: Indexed
```

Pods are named `myjob-0`, `myjob-1`, ..., `myjob-9`. Each can process a specific shard of work.

**Use case**: Parallel ML training, sharded batch processing, distributed sorting.

***

### 🛡️ Key Job Parameters (Memorize!)

| Field                     | Purpose                                          | Common Value               |
| ------------------------- | ------------------------------------------------ | -------------------------- |
| `completions`             | Number of successful pods required               | 1 (default)                |
| `parallelism`             | Max pods running concurrently                    | 1 (default)                |
| `backoffLimit`            | Max retries before marking Job Failed            | 6 (default)                |
| `activeDeadlineSeconds`   | Hard timeout for entire Job (kills running pods) | None                       |
| `ttlSecondsAfterFinished` | Auto-cleanup Job after completion                | 100 (1m40s)                |
| `completionMode`          | `NonIndexed` (default) or `Indexed`              | `Indexed` for sharded work |
| `suspend`                 | Pause Job creation                               | `false`                    |
| `podFailurePolicy`        | Granular failure handling (K8s 1.26+)            | See below                  |

***

### 🎯 `podFailurePolicy` — The Modern Failure Control

Beyond simple retries, K8s 1.26+ lets you decide what to do based on **exit code** or **pod condition**:

```yaml
spec:
  podFailurePolicy:
    rules:
    - action: FailJob              # immediate failure, no retry
      onExitCodes:
        operator: In
        values: [42]                # exit 42 = unrecoverable
    - action: Ignore               # don't count against backoffLimit
      onPodConditions:
      - type: DisruptionTarget      # node eviction, not app's fault
```

Actions: `FailJob`, `Ignore`, `Count` (default).

***

### 📅 CronJob Deep Dive

A **CronJob** creates Jobs on a cron schedule.

```yaml
spec:
  schedule: "0 2 * * *"           # 2 AM daily
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: my-backup:1.0
```

### Cron Syntax Refresher

```
┌───────────── minute (0–59)
│ ┌───────────── hour (0–23)
│ │ ┌───────────── day of month (1–31)
│ │ │ ┌───────────── month (1–12)
│ │ │ │ ┌───────────── day of week (0–6, Sun=0)
│ │ │ │ │
* * * * *
```

**Common examples:**

| Schedule      | Meaning                           |
| ------------- | --------------------------------- |
| `*/5 * * * *` | Every 5 minutes                   |
| `0 * * * *`   | Top of every hour                 |
| `0 2 * * *`   | 2:00 AM daily                     |
| `0 0 * * 0`   | Midnight every Sunday             |
| `0 9 * * 1-5` | 9 AM weekdays                     |
| `0 0 1 * *`   | Midnight on the 1st of each month |
| `@hourly`     | `0 * * * *` (shortcut)            |
| `@daily`      | `0 0 * * *`                       |
| `@weekly`     | `0 0 * * 0`                       |

> 💡 **Test cron expressions** at <https://crontab.guru> before deploying!

***

### ⚠️ The Five Critical CronJob Settings

#### 1. `concurrencyPolicy` — What if previous run is still going?

| Policy            | Behavior                          |
| ----------------- | --------------------------------- |
| `Allow` (default) | Start new Job alongside old       |
| `Forbid`          | Skip new Job if old still running |
| `Replace`         | Kill old, start new               |

**Production rule**: Use `Forbid` for backups/migrations to prevent overlap.

#### 2. `startingDeadlineSeconds`

If the cluster was down or controller paused, this is how late a missed Job can start. Without it, missed Jobs may never run.

#### 3. `successfulJobsHistoryLimit` / `failedJobsHistoryLimit`

How many old Jobs to retain for inspection (default: 3 successful, 1 failed). Critical — without limits, your cluster accumulates thousands of completed Jobs.

#### 4. `suspend`

Pause the CronJob without deleting (`true`/`false`). Perfect for maintenance windows.

#### 5. `timeZone` (K8s 1.27+)

```yaml
spec:
  schedule: "0 9 * * *"
  timeZone: "Asia/Kolkata"     # finally! No more UTC confusion
```

> 🎯 **Senior gotcha**: Without `timeZone`, CronJobs run in **UTC**. A "9 AM IST" schedule needs `30 3 * * *` UTC, OR `0 9 * * *` with `timeZone: Asia/Kolkata`.

***

### 🔄 CronJob → Job → Pod Hierarchy

```
CronJob: nightly-backup
   │
   ├─ Job: nightly-backup-28456789  (created 2026-06-24 02:00)
   │  └─ Pod: nightly-backup-28456789-xyz12 ✓
   │
   ├─ Job: nightly-backup-28456790  (created 2026-06-25 02:00)
   │  └─ Pod: nightly-backup-28456790-abc34 ✓
   │
   └─ Job: nightly-backup-28456791  (created 2026-06-26 02:00)
      └─ Pod: nightly-backup-28456791-def56 ⟳ running
```

***

## 🛠️ Part 2: Hands-On — From Hello World to Production

### Lab 1: Hello-World Job

```yaml
# 01-hello-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job
spec:
  ttlSecondsAfterFinished: 100      # auto-cleanup after 100s
  backoffLimit: 3
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: hello
        image: busybox:1.36
        command: ['sh', '-c', 'echo "Hello from $HOSTNAME at $(date)"; sleep 5; echo "Done!"']
```

```powershell
kubectl apply -f 01-hello-job.yaml

# Watch the Job complete
kubectl get jobs -w

# View the pod and its logs
kubectl get pods -l job-name=hello-job
$pod = kubectl get pods -l job-name=hello-job -o jsonpath='{.items[0].metadata.name}'
kubectl logs $pod

# Inspect the Job
kubectl describe job hello-job

# Watch auto-cleanup after 100s (TTL controller)
kubectl get jobs -w
```

***

### Lab 2: Parallel Job (Fixed Completions)

Process 6 "work items" with 3 parallel workers:

```yaml
# 02-parallel-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-processor
spec:
  completions: 6                     # need 6 successful pods
  parallelism: 3                     # run 3 at a time
  backoffLimit: 6
  ttlSecondsAfterFinished: 300
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: worker
        image: busybox:1.36
        command:
        - sh
        - -c
        - |
          echo "Worker $HOSTNAME starting at $(date)"
          # Simulate work: 5-15 seconds
          DURATION=$((RANDOM % 10 + 5))
          sleep $DURATION
          echo "Worker $HOSTNAME finished after $DURATION seconds"
```

```powershell
kubectl apply -f 02-parallel-job.yaml

# Watch parallelism in action
kubectl get pods -l job-name=parallel-processor -w

# Check final Job state
kubectl get job parallel-processor
# COMPLETIONS: 6/6
```

***

### Lab 3: Indexed Job (K8s 1.24+)

Each pod processes a specific shard:

```yaml
# 03-indexed-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: indexed-shards
spec:
  completions: 5
  parallelism: 3
  completionMode: Indexed
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: worker
        image: busybox:1.36
        command:
        - sh
        - -c
        - |
          echo "Processing shard $JOB_COMPLETION_INDEX of 5"
          # In real life: process file shard-${JOB_COMPLETION_INDEX}.csv
          sleep 5
          echo "Shard $JOB_COMPLETION_INDEX done"
```

```powershell
kubectl apply -f 03-indexed-job.yaml

# Pods named indexed-shards-0, -1, -2, ...
kubectl get pods -l job-name=indexed-shards

# Verify each got a unique index
for ($i = 0; $i -lt 5; $i++) {
  kubectl logs indexed-shards-$i 2>$null
}
```

***

### Lab 4: Job with Failure Retry

A Job that fails the first 2 attempts then succeeds:

```yaml
# 04-flaky-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: flaky-job
spec:
  backoffLimit: 4                    # allow up to 4 retries
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: flaky
        image: busybox:1.36
        command:
        - sh
        - -c
        - |
          # Fail randomly 50% of the time
          if [ $((RANDOM % 2)) -eq 0 ]; then
            echo "FAIL at $(date)"
            exit 1
          else
            echo "SUCCESS at $(date)"
            exit 0
          fi
```

```powershell
kubectl apply -f 04-flaky-job.yaml

# Watch retry attempts
kubectl get pods -l job-name=flaky-job -w
# You'll see pods with status Error, then new pods spawn

# Inspect Job conditions
kubectl describe job flaky-job
```

***

### Lab 5: Job with Hard Timeout

```yaml
# 05-timeout-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: timeout-job
spec:
  activeDeadlineSeconds: 30          # entire Job killed after 30s
  backoffLimit: 0
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: long
        image: busybox:1.36
        command: ['sh', '-c', 'echo "Started"; sleep 600; echo "Never reached"']
```

```powershell
kubectl apply -f 05-timeout-job.yaml

# Job will be killed at 30s
kubectl get job timeout-job -w

# Check the reason
kubectl describe job timeout-job | Select-String "Reason|Message"
# Reason: DeadlineExceeded
```

***

### Lab 6: PostgreSQL Backup CronJob (PRODUCTION-READY)

This is your **assignment** — a periodic backup of the Postgres DB from Day 9.

```yaml
# 06-pg-backup-cronjob.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: db
---
# PVC for storing backups
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pg-backup-storage
  namespace: db
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 10Gi
---
# Secret with DB credentials (assumes Day 9 secret exists)
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: db
type: Opaque
stringData:
  POSTGRES_USER: pgadmin
  POSTGRES_PASSWORD: SuperSecure2026!
  POSTGRES_DB: appdb
---
# The CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: db
  labels:
    app: postgres-backup
    criticality: tier-1
spec:
  schedule: "0 2 * * *"              # 2 AM every day
  timeZone: "Asia/Kolkata"           # IST
  concurrencyPolicy: Forbid          # never run two backups simultaneously
  startingDeadlineSeconds: 600       # if missed by >10min, skip
  successfulJobsHistoryLimit: 7      # keep last 7 successful (1 week)
  failedJobsHistoryLimit: 3          # keep last 3 failed (debugging)
  
  jobTemplate:
    spec:
      backoffLimit: 2                # 2 retries max per scheduled run
      activeDeadlineSeconds: 1800    # 30-minute timeout per backup
      ttlSecondsAfterFinished: 604800  # cleanup after 7 days
      
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: postgres:15-alpine
            envFrom:
            - secretRef:
                name: postgres-secret
            env:
            - name: PGHOST
              value: "postgres.db.svc.cluster.local"
            - name: PGPORT
              value: "5432"
            command:
            - sh
            - -c
            - |
              set -e
              TIMESTAMP=$(date +%Y%m%d-%H%M%S)
              BACKUP_FILE="/backups/${POSTGRES_DB}-${TIMESTAMP}.sql.gz"
              
              echo "===== Starting backup at $(date) ====="
              echo "Database: $POSTGRES_DB"
              echo "Host: $PGHOST"
              echo "Backup file: $BACKUP_FILE"
              
              # Run pg_dump and compress
              PGPASSWORD=$POSTGRES_PASSWORD pg_dump \
                -h $PGHOST \
                -U $POSTGRES_USER \
                -d $POSTGRES_DB \
                --no-owner \
                --no-acl \
                --format=plain \
                | gzip > $BACKUP_FILE
              
              # Verify backup file
              if [ -s "$BACKUP_FILE" ]; then
                SIZE=$(du -h $BACKUP_FILE | cut -f1)
                echo "✅ Backup successful: $SIZE"
              else
                echo "❌ Backup file is empty!"
                exit 1
              fi
              
              # Clean up backups older than 30 days
              echo "----- Pruning old backups -----"
              find /backups -name "*.sql.gz" -mtime +30 -delete -print
              
              # List remaining backups
              echo "----- Current backups -----"
              ls -lh /backups
              
              echo "===== Backup completed at $(date) ====="
            
            resources:
              requests:
                cpu: 100m
                memory: 256Mi
              limits:
                cpu: 500m
                memory: 512Mi
            
            volumeMounts:
            - name: backup-storage
              mountPath: /backups
          
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: pg-backup-storage
```

```powershell
kubectl apply -f 06-pg-backup-cronjob.yaml

# Verify the CronJob is scheduled
kubectl get cronjobs -n db
kubectl describe cronjob postgres-backup -n db

# Manually trigger a run (don't wait until 2 AM!)
kubectl create job -n db --from=cronjob/postgres-backup manual-backup-$(date +%s)

# Watch the Job and Pod
kubectl get jobs -n db
kubectl get pods -n db -l app=postgres-backup

# Tail the backup logs
$pod = kubectl get pods -n db -l job-name=manual-backup-* -o jsonpath='{.items[0].metadata.name}'
kubectl logs $pod -n db -f
```

#### Verify Backup Files

```powershell
# Spin up a temporary pod with access to the backup PVC
kubectl run -n db backup-inspector --image=busybox:1.36 -it --rm --restart=Never `
  --overrides='{"spec":{"containers":[{"name":"backup-inspector","image":"busybox:1.36","stdin":true,"tty":true,"volumeMounts":[{"name":"data","mountPath":"/backups"}]}],"volumes":[{"name":"data","persistentVolumeClaim":{"claimName":"pg-backup-storage"}}]}}' `
  -- sh

# Inside:
ls -lh /backups
exit
```

***

### Lab 7: CronJob — Cleanup Task (Cluster Hygiene)

A real-world example — clean up old completed pods every hour:

```yaml
# 07-cleanup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: pod-cleanup
  namespace: kube-system
spec:
  schedule: "0 * * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 1
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 1
      ttlSecondsAfterFinished: 86400
      template:
        spec:
          restartPolicy: OnFailure
          serviceAccountName: pod-cleaner   # needs RBAC!
          containers:
          - name: cleanup
            image: bitnami/kubectl:1.30
            command:
            - sh
            - -c
            - |
              echo "Cleaning up Succeeded/Failed pods older than 1 day..."
              kubectl get pods --all-namespaces \
                --field-selector=status.phase=Succeeded \
                -o json | \
                jq -r '.items[] | select(.metadata.creationTimestamp | fromdateiso8601 < (now - 86400)) | "\(.metadata.namespace) \(.metadata.name)"' | \
                while read ns name; do
                  echo "Deleting pod $name in $ns"
                  kubectl delete pod $name -n $ns
                done
              echo "Cleanup done at $(date)"
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-cleaner
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-cleaner
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: pod-cleaner
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: pod-cleaner
subjects:
- kind: ServiceAccount
  name: pod-cleaner
  namespace: kube-system
```

***

### Lab 8: Suspending and Resuming a CronJob

```powershell
# Pause a CronJob (e.g., during maintenance window)
kubectl patch cronjob postgres-backup -n db -p '{"spec":{"suspend":true}}'
kubectl get cronjob postgres-backup -n db
# SUSPEND: True

# Resume
kubectl patch cronjob postgres-backup -n db -p '{"spec":{"suspend":false}}'
```

***

## 💻 Part 3: Commands Cheat Sheet

### Job Operations

```powershell
# Create
kubectl apply -f job.yaml
kubectl create job hello --image=busybox -- echo "hi"

# Inspect
kubectl get jobs
kubectl get jobs -A
kubectl describe job <name>
kubectl get pods -l job-name=<jobname>

# Logs
$pod = kubectl get pod -l job-name=<jobname> -o jsonpath='{.items[0].metadata.name}'
kubectl logs $pod

# Delete (cascades to pods)
kubectl delete job <name>

# Delete keeping pods
kubectl delete job <name> --cascade=orphan
```

### CronJob Operations

```powershell
# Read
kubectl get cronjobs
kubectl get cj
kubectl describe cronjob <name>

# Manual run (testing)
kubectl create job manual-run-001 --from=cronjob/<name>

# Suspend / Resume
kubectl patch cronjob <name> -p '{"spec":{"suspend":true}}'
kubectl patch cronjob <name> -p '{"spec":{"suspend":false}}'

# Edit schedule
kubectl patch cronjob <name> -p '{"spec":{"schedule":"*/15 * * * *"}}'

# List Jobs created by a CronJob
kubectl get jobs --selector=job-name -o wide | findstr <cronjob-name>

# Delete
kubectl delete cronjob <name>
```

### Debugging

```powershell
# Why did my Job fail?
kubectl describe job <name>
# Look at Conditions section: "BackoffLimitExceeded" or "DeadlineExceeded"

# View pod failure reasons
kubectl get pods -l job-name=<name> -o yaml | Select-String "reason|message"

# Why isn't my CronJob triggering?
kubectl describe cronjob <name>
# Check LastScheduleTime, Active jobs, suspended state

# Get CronJob's Job history
kubectl get jobs --sort-by=.metadata.creationTimestamp
```

***

## 🎤 Part 4: Interview Prep

### ⭐ Star Question: *"How do you handle failed Jobs in Kubernetes?"*

**Senior DevOps model answer:**

> "Job failure handling in Kubernetes is a **layered concern** involving retries, timeouts, alerting, and graceful recovery. Here's my full strategy.
>
> **First, distinguish between pod failure and Job failure.** A pod can fail (exit non-zero, OOM, image pull error), but the Job itself only fails after exhausting retries. The key parameters are:
>
> * **`backoffLimit`** (default 6) — total number of retries before the Job is marked Failed.
> * **`restartPolicy`** — `OnFailure` restarts the container in the same pod; `Never` creates a new pod for each retry. I prefer `Never` in production because each retry gets its own pod with separate logs and clear failure events — much easier to debug.
> * **`backoffLimit`** uses **exponential backoff** — 10s, 20s, 40s, up to 6 minutes. This prevents thundering-herd retries when a downstream system is overloaded.
>
> **Second, set hard timeouts.** I always set `activeDeadlineSeconds` — this is the maximum time the entire Job can take, including retries. Without it, a stuck Job could hang forever, holding resources. For a DB migration I might use 600s; for a backup, 1800s. When exceeded, the Job is marked Failed with reason `DeadlineExceeded`.
>
> **Third, in K8s 1.26+, use `podFailurePolicy` for granular control.** This is a senior-level feature most people don't know. You can say:
>
> * 'If exit code 42 (unrecoverable error), fail the Job immediately without retries.'
> * 'If pod was terminated due to node disruption (`DisruptionTarget` condition), don't count it against `backoffLimit`.'
>
> This separates **infrastructure failures** (retry-worthy) from **application bugs** (fail fast).
>
> **Fourth, idempotency at the application layer is non-negotiable.** Jobs WILL be retried. So the script must be safe to run twice. For a DB migration, that means checking `schema_migrations` table first. For a backup, use timestamped filenames so retries don't overwrite. For an API call, use idempotency keys.
>
> **Fifth, observability.** I configure:
>
> * `ttlSecondsAfterFinished` to auto-cleanup, but only AFTER metrics are scraped.
> * Prometheus monitoring of `kube_job_status_failed`, `kube_job_status_succeeded`.
> * Alerts via Alertmanager when a Job fails — paging on-call for tier-1 Jobs like backups.
> * Logs shipped to centralized logging (Azure Monitor, ELK, Loki) before the pod is garbage-collected.
>
> **Sixth, for CronJobs specifically**, I use:
>
> * `concurrencyPolicy: Forbid` for backups so a slow run doesn't overlap with the next scheduled one.
> * `startingDeadlineSeconds` to handle missed schedules — without it, a brief controller outage can cause Jobs to never run.
> * `successfulJobsHistoryLimit` and `failedJobsHistoryLimit` — failed Jobs kept longer for debugging, successful ones cleaned aggressively.
>
> **Seventh, recovery**. For a permanently failed Job, my runbook is:
>
> 1. `kubectl describe job <name>` → check Conditions and Events.
> 2. `kubectl logs <pod>` and `kubectl logs <pod> --previous` for the actual error.
> 3. If it's an infrastructure issue, recreate the Job manually with `kubectl create job <new-name> --from=cronjob/<name>`.
> 4. If it's a code bug, fix in Git, redeploy via GitOps, and trigger a fresh Job.
> 5. For backups specifically, I have a Velero or PG dump restore runbook — never silently accept a missed backup.
>
> So in short: **retries + timeouts + idempotency + podFailurePolicy + monitoring + automatic cleanup + runbooks**. Failure handling is a system, not a single setting."

***

### 🔥 Common Follow-Ups

| #  | Question                                                               | Quick Answer                                                                                                                        |
| -- | ---------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| 1  | **Difference between Job and Deployment?**                             | Job = runs to completion (`restartPolicy: Never` or `OnFailure`). Deployment = always running (`restartPolicy: Always`).            |
| 2  | **What is `backoffLimit`?**                                            | Max number of failed pod retries before the Job is marked Failed. Default: 6. Uses exponential backoff.                             |
| 3  | **Difference between `restartPolicy: OnFailure` and `Never` in Jobs?** | `OnFailure` restarts container in same pod (faster, simpler). `Never` creates new pod (separate logs, better for debugging).        |
| 4  | **What's `completions` vs `parallelism`?**                             | `completions` = total successful pods needed. `parallelism` = how many run concurrently.                                            |
| 5  | **What is an Indexed Job?**                                            | K8s 1.24+. Each pod gets a unique `JOB_COMPLETION_INDEX` env var. Used for sharded work like ML training, batch processing.         |
| 6  | **How do you ensure a Job doesn't run forever?**                       | Set `activeDeadlineSeconds`. Job is killed after this duration regardless of retries.                                               |
| 7  | **How do you clean up completed Jobs?**                                | Set `ttlSecondsAfterFinished` — TTL controller deletes the Job (and its pods) after the specified seconds.                          |
| 8  | **What's `concurrencyPolicy` in CronJob?**                             | `Allow` (default, overlap), `Forbid` (skip if previous still running), `Replace` (kill old, start new).                             |
| 9  | **Why might a CronJob miss a scheduled run?**                          | Controller outage, cluster down, `startingDeadlineSeconds` exceeded, or `concurrencyPolicy: Forbid` with previous run still active. |
| 10 | **What's `startingDeadlineSeconds`?**                                  | Max delay (in seconds) before a missed Job is considered too late to run. Without it, the Job may never start.                      |
| 11 | **Are CronJob times in UTC or cluster timezone?**                      | **UTC by default**. K8s 1.27+ supports `timeZone: "Asia/Kolkata"` to specify locally.                                               |
| 12 | **How do you manually trigger a CronJob?**                             | `kubectl create job manual-001 --from=cronjob/<name>`.                                                                              |
| 13 | **What's `podFailurePolicy`?**                                         | K8s 1.26+ feature for granular failure handling — fail immediately on specific exit codes, ignore failures from node disruptions.   |
| 14 | **How do you pause a CronJob without deleting it?**                    | `kubectl patch cronjob <name> -p '{"spec":{"suspend":true}}'`.                                                                      |
| 15 | **What happens if a Job's pod is OOMKilled?**                          | Pod fails, counted against `backoffLimit`, retried. Fix: increase `resources.limits.memory` or check for memory leaks.              |

***

### 💡 Senior Scenario Questions

**Scenario 1: "Your nightly backup CronJob hasn't run for 3 days. How do you investigate?"**

> Methodical debugging:
>
> 1. `kubectl describe cronjob postgres-backup -n db` — check Status section for `LastScheduleTime`, `Active`, and any error events.
> 2. Check if **suspended**: `kubectl get cronjob -n db -o jsonpath='{.spec.suspend}'`.
> 3. Check **CronJob controller logs**: `kubectl logs -n kube-system -l component=kube-controller-manager`.
> 4. Look for `MissedSchedule` events — happens when controller wasn't running and `startingDeadlineSeconds` expired.
> 5. Are old Jobs accumulating beyond `successfulJobsHistoryLimit`? Indicates broken cleanup.
> 6. Verify schedule with `crontab.guru` — typos like `0 25 * * *` (invalid hour) cause silent failures.
> 7. Check if `concurrencyPolicy: Forbid` is blocking — is there a stuck Job from 3 days ago?
> 8. **Recovery**: manually trigger via `kubectl create job --from=cronjob/postgres-backup recovery-$(date +%s) -n db`.
> 9. **Long-term fix**: add Prometheus alert on `time() - kube_cronjob_status_last_schedule_time > 90000` (no schedule in 25 hours).

***

**Scenario 2: "Your DB migration Job ran 3 times and corrupted the database. What went wrong?"**

> Root cause: **the migration script was not idempotent**, and Kubernetes retried it on transient failures.
>
> Forensics:
>
> 1. `kubectl describe job migrate` → check retry count and failure reasons.
> 2. Look at `kubectl logs` for each pod (use `--previous` for crashed containers).
>
> Common causes:
>
> * Migration script doesn't check schema version before applying.
> * Container exits 0 prematurely (before migration commits), then K8s thinks it succeeded but data wasn't fully applied.
> * Migration tool (Flyway, Liquibase) wasn't properly initialized, so it re-ran completed migrations.
>
> **Fix going forward**:
>
> 1. **Use idempotent migration tools** — Flyway/Liquibase track applied versions in a metadata table; safe to retry.
> 2. **Set `backoffLimit: 0`** for migrations — fail fast rather than retry corruption.
> 3. **Always backup before migration** — chain a backup Job + migration Job with init container check.
> 4. **Test in staging first** with identical Job spec.
> 5. **Migration in transactions** — if your DB supports it, wrap the whole thing in BEGIN/COMMIT.
> 6. **Recovery**: restore from last good backup, fix the migration, re-run.

***

**Scenario 3: "How would you run a batch ML training task across 100 GPU pods?"**

> Indexed Job pattern:
>
> ```yaml
> spec:
>   completions: 100
>   parallelism: 100              # all at once
>   completionMode: Indexed
>   template:
>     spec:
>       nodeSelector:
>         accelerator: nvidia-gpu
>       containers:
>       - name: trainer
>         image: ml-trainer:1.0
>         command:
>         - python
>         - train.py
>         - --shard-id=$(JOB_COMPLETION_INDEX)
>         - --total-shards=100
>         resources:
>           limits:
>             nvidia.com/gpu: 1
> ```
>
> Each pod processes 1/100th of the dataset. Combined with:
>
> * **Persistent volume** for shared model artifacts (RWX).
> * **PriorityClass** so training preempts lower-priority workloads.
> * **`activeDeadlineSeconds`** so a stuck training run doesn't burn GPU costs forever.
> * **podFailurePolicy** to ignore node disruption failures (spot node preemption shouldn't fail the whole Job).
> * **podDisruptionBudget** to prevent voluntary disruptions during training.

***

**Scenario 4: "Two backup CronJobs are running on the same DB and corrupting each other. How do you fix?"**

> Root cause: **`concurrencyPolicy: Allow`** (default), or two separate CronJobs targeting the same DB.
>
> Fix:
>
> 1. **Immediately** set `concurrencyPolicy: Forbid` on the CronJob.
> 2. If two CronJobs exist, **consolidate to one** and schedule appropriately.
> 3. **Application-level locking** — use a row lock in a `backup_locks` table; the script checks before starting.
> 4. **Verify schedule** doesn't overlap with maintenance windows or other heavy operations.
> 5. **Monitoring**: alert when two backup Jobs are simultaneously Active.

***

**Scenario 5: "Your team wants to backup 50 databases. Would you create 50 CronJobs?"**

> Options ranked by maintainability:
>
> 1. **❌ 50 separate CronJobs** — high YAML duplication, hard to update.
>
> 2. **✅ Helm chart with a loop** (Day 11 topic):
>    ```yaml
>    {{- range .Values.databases }}
>    apiVersion: batch/v1
>    kind: CronJob
>    metadata:
>      name: backup-{{ .name }}
>    spec:
>      # ...
>    {{- end }}
>    ```
>    One template, one values.yaml, 50 CronJobs generated.
>
> 3. **✅ Single CronJob with Indexed Job pattern** — one CronJob spawns 50 indexed pods, each reads its own DB config from a ConfigMap by index.
>
> 4. **✅✅ Best**: an **operator** like Stash or Velero — declarative `BackupConfiguration` per DB, operator handles scheduling, retention, validation, off-site replication.
>
> For an enterprise-scale answer, I'd combine Helm for templating + Velero/Stash for backup orchestration + Prometheus for monitoring + Azure Blob lifecycle policies for retention/archival.

***

## 📊 Decision Matrix

| Need                             | Use                                         |
| -------------------------------- | ------------------------------------------- |
| One-off DB migration             | Job (`backoffLimit: 0`)                     |
| Batch processing 100 files       | Job (`completions: 100`, `parallelism: 10`) |
| Sharded ML training              | Indexed Job                                 |
| Queue worker (RabbitMQ consumer) | Job (parallelism only, no completions)      |
| Periodic DB backup               | CronJob (`concurrencyPolicy: Forbid`)       |
| Hourly log archival              | CronJob (`@hourly`)                         |
| Continuous data sync             | **Deployment**, not Job/CronJob             |
| Need replication & failover      | **StatefulSet**, not Job                    |
| Always-running daemon            | **Deployment** or **DaemonSet**, not Job    |

***

# ⏰ Day 10: Jobs & CronJobs

Welcome to Day 10, Rajesh! You've mastered long-running workloads (Deployments, StatefulSets, DaemonSets). But what about **one-off tasks** that should **run, finish, and exit** — database migrations, backups, batch processing, ML training jobs, periodic cleanups? Today you'll master **Jobs** (batch tasks) and **CronJobs** (scheduled tasks) — workloads that power every production cluster's operational pipelines. This is **interview gold** for Senior DevOps roles. 🚀

***

## 📚 Part 1: Theory — Tasks That End

### The Problem with Deployments for Tasks

Deployments are designed for **always-running** workloads. They have `restartPolicy: Always` baked in. So if you run a DB migration as a Deployment:

```
1. Migration container runs migration → succeeds → exits with code 0
2. Kubernetes sees pod exited → restarts it
3. Migration runs AGAIN
4. ❌ Infinite migration loop → corrupts DB → wakes up Rajesh at 3 AM
```

**Jobs** were created exactly for this: run to **completion**, then stop.

***

### 🧱 The Job Family Tree

```
┌────────────────────────────────────────────────────────┐
│             KUBERNETES BATCH WORKLOADS                  │
├────────────────────────────────────────────────────────┤
│                                                         │
│  ┌────────────┐         ┌────────────┐                 │
│  │   JOB      │         │  CRONJOB   │                 │
│  │            │         │            │                 │
│  │ Run once   │ ◄────── │ Schedule   │                 │
│  │ until      │ creates │ Jobs based │                 │
│  │ completion │   Jobs  │ on cron    │                 │
│  └────────────┘         └────────────┘                 │
│                                                         │
└────────────────────────────────────────────────────────┘
```

| Resource    | Purpose                  | Lifecycle                              |
| ----------- | ------------------------ | -------------------------------------- |
| **Job**     | Run a task to completion | One-shot; pod terminates after success |
| **CronJob** | Schedule recurring Jobs  | Creates Jobs on cron schedule          |

***

### 🎯 Job Deep Dive

A **Job** ensures that **a specified number of pods successfully terminate**.

#### Pod RestartPolicy in Jobs

| Policy      | Behavior                                                                   |
| ----------- | -------------------------------------------------------------------------- |
| `OnFailure` | Container restarts in same pod on failure (counted against `backoffLimit`) |
| `Never`     | Pod fails → new pod created on retry (more debuggable, cleaner logs)       |

⚠️ `Always` is **NOT allowed** in Jobs — defeats the purpose.

***

### 🔢 Three Job Execution Patterns

#### Pattern 1: Single Job (Non-Parallel)

```
completions: 1      ← need 1 successful pod
parallelism: 1      ← run 1 at a time
```

**Use case**: DB migration, single-shot data import.

#### Pattern 2: Fixed Completion Count (Parallel)

```
completions: 5      ← need 5 successful pods
parallelism: 2      ← run 2 at a time
```

**Use case**: Process 5 input files in parallel, 2 workers at once.

```
Time T0:   [pod-1 ▶] [pod-2 ▶]
Time T1:   pod-1 ✓ → [pod-3 ▶] [pod-2 ▶]
Time T2:   pod-2 ✓ → [pod-3 ▶] [pod-4 ▶]
Time T3:   ...all done ✓
```

#### Pattern 3: Work Queue (Parallel, No Fixed Completion)

```
completions: not set
parallelism: 5      ← 5 workers all pulling from a queue
```

**Use case**: Queue-based workers (RabbitMQ consumers). Workers exit when queue is empty.

***

### 🆕 Indexed Jobs (K8s 1.24+)

A modern pattern — each pod gets a **unique index** via env var `JOB_COMPLETION_INDEX`.

```yaml
spec:
  completions: 10
  parallelism: 3
  completionMode: Indexed
```

Pods are named `myjob-0`, `myjob-1`, ..., `myjob-9`. Each can process a specific shard of work.

**Use case**: Parallel ML training, sharded batch processing, distributed sorting.

***

### 🛡️ Key Job Parameters (Memorize!)

| Field                     | Purpose                                          | Common Value               |
| ------------------------- | ------------------------------------------------ | -------------------------- |
| `completions`             | Number of successful pods required               | 1 (default)                |
| `parallelism`             | Max pods running concurrently                    | 1 (default)                |
| `backoffLimit`            | Max retries before marking Job Failed            | 6 (default)                |
| `activeDeadlineSeconds`   | Hard timeout for entire Job (kills running pods) | None                       |
| `ttlSecondsAfterFinished` | Auto-cleanup Job after completion                | 100 (1m40s)                |
| `completionMode`          | `NonIndexed` (default) or `Indexed`              | `Indexed` for sharded work |
| `suspend`                 | Pause Job creation                               | `false`                    |
| `podFailurePolicy`        | Granular failure handling (K8s 1.26+)            | See below                  |

***

### 🎯 `podFailurePolicy` — The Modern Failure Control

Beyond simple retries, K8s 1.26+ lets you decide what to do based on **exit code** or **pod condition**:

```yaml
spec:
  podFailurePolicy:
    rules:
    - action: FailJob              # immediate failure, no retry
      onExitCodes:
        operator: In
        values: [42]                # exit 42 = unrecoverable
    - action: Ignore               # don't count against backoffLimit
      onPodConditions:
      - type: DisruptionTarget      # node eviction, not app's fault
```

Actions: `FailJob`, `Ignore`, `Count` (default).

***

### 📅 CronJob Deep Dive

A **CronJob** creates Jobs on a cron schedule.

```yaml
spec:
  schedule: "0 2 * * *"           # 2 AM daily
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: my-backup:1.0
```

### Cron Syntax Refresher

```
┌───────────── minute (0–59)
│ ┌───────────── hour (0–23)
│ │ ┌───────────── day of month (1–31)
│ │ │ ┌───────────── month (1–12)
│ │ │ │ ┌───────────── day of week (0–6, Sun=0)
│ │ │ │ │
* * * * *
```

**Common examples:**

| Schedule      | Meaning                           |
| ------------- | --------------------------------- |
| `*/5 * * * *` | Every 5 minutes                   |
| `0 * * * *`   | Top of every hour                 |
| `0 2 * * *`   | 2:00 AM daily                     |
| `0 0 * * 0`   | Midnight every Sunday             |
| `0 9 * * 1-5` | 9 AM weekdays                     |
| `0 0 1 * *`   | Midnight on the 1st of each month |
| `@hourly`     | `0 * * * *` (shortcut)            |
| `@daily`      | `0 0 * * *`                       |
| `@weekly`     | `0 0 * * 0`                       |

> 💡 **Test cron expressions** at <https://crontab.guru> before deploying!

***

### ⚠️ The Five Critical CronJob Settings

#### 1. `concurrencyPolicy` — What if previous run is still going?

| Policy            | Behavior                          |
| ----------------- | --------------------------------- |
| `Allow` (default) | Start new Job alongside old       |
| `Forbid`          | Skip new Job if old still running |
| `Replace`         | Kill old, start new               |

**Production rule**: Use `Forbid` for backups/migrations to prevent overlap.

#### 2. `startingDeadlineSeconds`

If the cluster was down or controller paused, this is how late a missed Job can start. Without it, missed Jobs may never run.

#### 3. `successfulJobsHistoryLimit` / `failedJobsHistoryLimit`

How many old Jobs to retain for inspection (default: 3 successful, 1 failed). Critical — without limits, your cluster accumulates thousands of completed Jobs.

#### 4. `suspend`

Pause the CronJob without deleting (`true`/`false`). Perfect for maintenance windows.

#### 5. `timeZone` (K8s 1.27+)

```yaml
spec:
  schedule: "0 9 * * *"
  timeZone: "Asia/Kolkata"     # finally! No more UTC confusion
```

> 🎯 **Senior gotcha**: Without `timeZone`, CronJobs run in **UTC**. A "9 AM IST" schedule needs `30 3 * * *` UTC, OR `0 9 * * *` with `timeZone: Asia/Kolkata`.

***

### 🔄 CronJob → Job → Pod Hierarchy

```
CronJob: nightly-backup
   │
   ├─ Job: nightly-backup-28456789  (created 2026-06-24 02:00)
   │  └─ Pod: nightly-backup-28456789-xyz12 ✓
   │
   ├─ Job: nightly-backup-28456790  (created 2026-06-25 02:00)
   │  └─ Pod: nightly-backup-28456790-abc34 ✓
   │
   └─ Job: nightly-backup-28456791  (created 2026-06-26 02:00)
      └─ Pod: nightly-backup-28456791-def56 ⟳ running
```

***

## 🛠️ Part 2: Hands-On — From Hello World to Production

### Lab 1: Hello-World Job

```yaml
# 01-hello-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job
spec:
  ttlSecondsAfterFinished: 100      # auto-cleanup after 100s
  backoffLimit: 3
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: hello
        image: busybox:1.36
        command: ['sh', '-c', 'echo "Hello from $HOSTNAME at $(date)"; sleep 5; echo "Done!"']
```

```powershell
kubectl apply -f 01-hello-job.yaml

# Watch the Job complete
kubectl get jobs -w

# View the pod and its logs
kubectl get pods -l job-name=hello-job
$pod = kubectl get pods -l job-name=hello-job -o jsonpath='{.items[0].metadata.name}'
kubectl logs $pod

# Inspect the Job
kubectl describe job hello-job

# Watch auto-cleanup after 100s (TTL controller)
kubectl get jobs -w
```

***

### Lab 2: Parallel Job (Fixed Completions)

Process 6 "work items" with 3 parallel workers:

```yaml
# 02-parallel-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-processor
spec:
  completions: 6                     # need 6 successful pods
  parallelism: 3                     # run 3 at a time
  backoffLimit: 6
  ttlSecondsAfterFinished: 300
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: worker
        image: busybox:1.36
        command:
        - sh
        - -c
        - |
          echo "Worker $HOSTNAME starting at $(date)"
          # Simulate work: 5-15 seconds
          DURATION=$((RANDOM % 10 + 5))
          sleep $DURATION
          echo "Worker $HOSTNAME finished after $DURATION seconds"
```

```powershell
kubectl apply -f 02-parallel-job.yaml

# Watch parallelism in action
kubectl get pods -l job-name=parallel-processor -w

# Check final Job state
kubectl get job parallel-processor
# COMPLETIONS: 6/6
```

***

### Lab 3: Indexed Job (K8s 1.24+)

Each pod processes a specific shard:

```yaml
# 03-indexed-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: indexed-shards
spec:
  completions: 5
  parallelism: 3
  completionMode: Indexed
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: worker
        image: busybox:1.36
        command:
        - sh
        - -c
        - |
          echo "Processing shard $JOB_COMPLETION_INDEX of 5"
          # In real life: process file shard-${JOB_COMPLETION_INDEX}.csv
          sleep 5
          echo "Shard $JOB_COMPLETION_INDEX done"
```

```powershell
kubectl apply -f 03-indexed-job.yaml

# Pods named indexed-shards-0, -1, -2, ...
kubectl get pods -l job-name=indexed-shards

# Verify each got a unique index
for ($i = 0; $i -lt 5; $i++) {
  kubectl logs indexed-shards-$i 2>$null
}
```

***

### Lab 4: Job with Failure Retry

A Job that fails the first 2 attempts then succeeds:

```yaml
# 04-flaky-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: flaky-job
spec:
  backoffLimit: 4                    # allow up to 4 retries
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: flaky
        image: busybox:1.36
        command:
        - sh
        - -c
        - |
          # Fail randomly 50% of the time
          if [ $((RANDOM % 2)) -eq 0 ]; then
            echo "FAIL at $(date)"
            exit 1
          else
            echo "SUCCESS at $(date)"
            exit 0
          fi
```

```powershell
kubectl apply -f 04-flaky-job.yaml

# Watch retry attempts
kubectl get pods -l job-name=flaky-job -w
# You'll see pods with status Error, then new pods spawn

# Inspect Job conditions
kubectl describe job flaky-job
```

***

### Lab 5: Job with Hard Timeout

```yaml
# 05-timeout-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: timeout-job
spec:
  activeDeadlineSeconds: 30          # entire Job killed after 30s
  backoffLimit: 0
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: long
        image: busybox:1.36
        command: ['sh', '-c', 'echo "Started"; sleep 600; echo "Never reached"']
```

```powershell
kubectl apply -f 05-timeout-job.yaml

# Job will be killed at 30s
kubectl get job timeout-job -w

# Check the reason
kubectl describe job timeout-job | Select-String "Reason|Message"
# Reason: DeadlineExceeded
```

***

### Lab 6: PostgreSQL Backup CronJob (PRODUCTION-READY)

This is your **assignment** — a periodic backup of the Postgres DB from Day 9.

```yaml
# 06-pg-backup-cronjob.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: db
---
# PVC for storing backups
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pg-backup-storage
  namespace: db
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 10Gi
---
# Secret with DB credentials (assumes Day 9 secret exists)
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: db
type: Opaque
stringData:
  POSTGRES_USER: pgadmin
  POSTGRES_PASSWORD: SuperSecure2026!
  POSTGRES_DB: appdb
---
# The CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: db
  labels:
    app: postgres-backup
    criticality: tier-1
spec:
  schedule: "0 2 * * *"              # 2 AM every day
  timeZone: "Asia/Kolkata"           # IST
  concurrencyPolicy: Forbid          # never run two backups simultaneously
  startingDeadlineSeconds: 600       # if missed by >10min, skip
  successfulJobsHistoryLimit: 7      # keep last 7 successful (1 week)
  failedJobsHistoryLimit: 3          # keep last 3 failed (debugging)
  
  jobTemplate:
    spec:
      backoffLimit: 2                # 2 retries max per scheduled run
      activeDeadlineSeconds: 1800    # 30-minute timeout per backup
      ttlSecondsAfterFinished: 604800  # cleanup after 7 days
      
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: postgres:15-alpine
            envFrom:
            - secretRef:
                name: postgres-secret
            env:
            - name: PGHOST
              value: "postgres.db.svc.cluster.local"
            - name: PGPORT
              value: "5432"
            command:
            - sh
            - -c
            - |
              set -e
              TIMESTAMP=$(date +%Y%m%d-%H%M%S)
              BACKUP_FILE="/backups/${POSTGRES_DB}-${TIMESTAMP}.sql.gz"
              
              echo "===== Starting backup at $(date) ====="
              echo "Database: $POSTGRES_DB"
              echo "Host: $PGHOST"
              echo "Backup file: $BACKUP_FILE"
              
              # Run pg_dump and compress
              PGPASSWORD=$POSTGRES_PASSWORD pg_dump \
                -h $PGHOST \
                -U $POSTGRES_USER \
                -d $POSTGRES_DB \
                --no-owner \
                --no-acl \
                --format=plain \
                | gzip > $BACKUP_FILE
              
              # Verify backup file
              if [ -s "$BACKUP_FILE" ]; then
                SIZE=$(du -h $BACKUP_FILE | cut -f1)
                echo "✅ Backup successful: $SIZE"
              else
                echo "❌ Backup file is empty!"
                exit 1
              fi
              
              # Clean up backups older than 30 days
              echo "----- Pruning old backups -----"
              find /backups -name "*.sql.gz" -mtime +30 -delete -print
              
              # List remaining backups
              echo "----- Current backups -----"
              ls -lh /backups
              
              echo "===== Backup completed at $(date) ====="
            
            resources:
              requests:
                cpu: 100m
                memory: 256Mi
              limits:
                cpu: 500m
                memory: 512Mi
            
            volumeMounts:
            - name: backup-storage
              mountPath: /backups
          
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: pg-backup-storage
```

```powershell
kubectl apply -f 06-pg-backup-cronjob.yaml

# Verify the CronJob is scheduled
kubectl get cronjobs -n db
kubectl describe cronjob postgres-backup -n db

# Manually trigger a run (don't wait until 2 AM!)
kubectl create job -n db --from=cronjob/postgres-backup manual-backup-$(date +%s)

# Watch the Job and Pod
kubectl get jobs -n db
kubectl get pods -n db -l app=postgres-backup

# Tail the backup logs
$pod = kubectl get pods -n db -l job-name=manual-backup-* -o jsonpath='{.items[0].metadata.name}'
kubectl logs $pod -n db -f
```

#### Verify Backup Files

```powershell
# Spin up a temporary pod with access to the backup PVC
kubectl run -n db backup-inspector --image=busybox:1.36 -it --rm --restart=Never `
  --overrides='{"spec":{"containers":[{"name":"backup-inspector","image":"busybox:1.36","stdin":true,"tty":true,"volumeMounts":[{"name":"data","mountPath":"/backups"}]}],"volumes":[{"name":"data","persistentVolumeClaim":{"claimName":"pg-backup-storage"}}]}}' `
  -- sh

# Inside:
ls -lh /backups
exit
```

***

### Lab 7: CronJob — Cleanup Task (Cluster Hygiene)

A real-world example — clean up old completed pods every hour:

```yaml
# 07-cleanup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: pod-cleanup
  namespace: kube-system
spec:
  schedule: "0 * * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 1
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 1
      ttlSecondsAfterFinished: 86400
      template:
        spec:
          restartPolicy: OnFailure
          serviceAccountName: pod-cleaner   # needs RBAC!
          containers:
          - name: cleanup
            image: bitnami/kubectl:1.30
            command:
            - sh
            - -c
            - |
              echo "Cleaning up Succeeded/Failed pods older than 1 day..."
              kubectl get pods --all-namespaces \
                --field-selector=status.phase=Succeeded \
                -o json | \
                jq -r '.items[] | select(.metadata.creationTimestamp | fromdateiso8601 < (now - 86400)) | "\(.metadata.namespace) \(.metadata.name)"' | \
                while read ns name; do
                  echo "Deleting pod $name in $ns"
                  kubectl delete pod $name -n $ns
                done
              echo "Cleanup done at $(date)"
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-cleaner
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-cleaner
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: pod-cleaner
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: pod-cleaner
subjects:
- kind: ServiceAccount
  name: pod-cleaner
  namespace: kube-system
```

***

### Lab 8: Suspending and Resuming a CronJob

```powershell
# Pause a CronJob (e.g., during maintenance window)
kubectl patch cronjob postgres-backup -n db -p '{"spec":{"suspend":true}}'
kubectl get cronjob postgres-backup -n db
# SUSPEND: True

# Resume
kubectl patch cronjob postgres-backup -n db -p '{"spec":{"suspend":false}}'
```

***

## 💻 Part 3: Commands Cheat Sheet

### Job Operations

```powershell
# Create
kubectl apply -f job.yaml
kubectl create job hello --image=busybox -- echo "hi"

# Inspect
kubectl get jobs
kubectl get jobs -A
kubectl describe job <name>
kubectl get pods -l job-name=<jobname>

# Logs
$pod = kubectl get pod -l job-name=<jobname> -o jsonpath='{.items[0].metadata.name}'
kubectl logs $pod

# Delete (cascades to pods)
kubectl delete job <name>

# Delete keeping pods
kubectl delete job <name> --cascade=orphan
```

### CronJob Operations

```powershell
# Read
kubectl get cronjobs
kubectl get cj
kubectl describe cronjob <name>

# Manual run (testing)
kubectl create job manual-run-001 --from=cronjob/<name>

# Suspend / Resume
kubectl patch cronjob <name> -p '{"spec":{"suspend":true}}'
kubectl patch cronjob <name> -p '{"spec":{"suspend":false}}'

# Edit schedule
kubectl patch cronjob <name> -p '{"spec":{"schedule":"*/15 * * * *"}}'

# List Jobs created by a CronJob
kubectl get jobs --selector=job-name -o wide | findstr <cronjob-name>

# Delete
kubectl delete cronjob <name>
```

### Debugging

```powershell
# Why did my Job fail?
kubectl describe job <name>
# Look at Conditions section: "BackoffLimitExceeded" or "DeadlineExceeded"

# View pod failure reasons
kubectl get pods -l job-name=<name> -o yaml | Select-String "reason|message"

# Why isn't my CronJob triggering?
kubectl describe cronjob <name>
# Check LastScheduleTime, Active jobs, suspended state

# Get CronJob's Job history
kubectl get jobs --sort-by=.metadata.creationTimestamp
```

***

## 🎤 Part 4: Interview Prep

### ⭐ Star Question: *"How do you handle failed Jobs in Kubernetes?"*

**Senior DevOps model answer:**

> "Job failure handling in Kubernetes is a **layered concern** involving retries, timeouts, alerting, and graceful recovery. Here's my full strategy.
>
> **First, distinguish between pod failure and Job failure.** A pod can fail (exit non-zero, OOM, image pull error), but the Job itself only fails after exhausting retries. The key parameters are:
>
> * **`backoffLimit`** (default 6) — total number of retries before the Job is marked Failed.
> * **`restartPolicy`** — `OnFailure` restarts the container in the same pod; `Never` creates a new pod for each retry. I prefer `Never` in production because each retry gets its own pod with separate logs and clear failure events — much easier to debug.
> * **`backoffLimit`** uses **exponential backoff** — 10s, 20s, 40s, up to 6 minutes. This prevents thundering-herd retries when a downstream system is overloaded.
>
> **Second, set hard timeouts.** I always set `activeDeadlineSeconds` — this is the maximum time the entire Job can take, including retries. Without it, a stuck Job could hang forever, holding resources. For a DB migration I might use 600s; for a backup, 1800s. When exceeded, the Job is marked Failed with reason `DeadlineExceeded`.
>
> **Third, in K8s 1.26+, use `podFailurePolicy` for granular control.** This is a senior-level feature most people don't know. You can say:
>
> * 'If exit code 42 (unrecoverable error), fail the Job immediately without retries.'
> * 'If pod was terminated due to node disruption (`DisruptionTarget` condition), don't count it against `backoffLimit`.'
>
> This separates **infrastructure failures** (retry-worthy) from **application bugs** (fail fast).
>
> **Fourth, idempotency at the application layer is non-negotiable.** Jobs WILL be retried. So the script must be safe to run twice. For a DB migration, that means checking `schema_migrations` table first. For a backup, use timestamped filenames so retries don't overwrite. For an API call, use idempotency keys.
>
> **Fifth, observability.** I configure:
>
> * `ttlSecondsAfterFinished` to auto-cleanup, but only AFTER metrics are scraped.
> * Prometheus monitoring of `kube_job_status_failed`, `kube_job_status_succeeded`.
> * Alerts via Alertmanager when a Job fails — paging on-call for tier-1 Jobs like backups.
> * Logs shipped to centralized logging (Azure Monitor, ELK, Loki) before the pod is garbage-collected.
>
> **Sixth, for CronJobs specifically**, I use:
>
> * `concurrencyPolicy: Forbid` for backups so a slow run doesn't overlap with the next scheduled one.
> * `startingDeadlineSeconds` to handle missed schedules — without it, a brief controller outage can cause Jobs to never run.
> * `successfulJobsHistoryLimit` and `failedJobsHistoryLimit` — failed Jobs kept longer for debugging, successful ones cleaned aggressively.
>
> **Seventh, recovery**. For a permanently failed Job, my runbook is:
>
> 1. `kubectl describe job <name>` → check Conditions and Events.
> 2. `kubectl logs <pod>` and `kubectl logs <pod> --previous` for the actual error.
> 3. If it's an infrastructure issue, recreate the Job manually with `kubectl create job <new-name> --from=cronjob/<name>`.
> 4. If it's a code bug, fix in Git, redeploy via GitOps, and trigger a fresh Job.
> 5. For backups specifically, I have a Velero or PG dump restore runbook — never silently accept a missed backup.
>
> So in short: **retries + timeouts + idempotency + podFailurePolicy + monitoring + automatic cleanup + runbooks**. Failure handling is a system, not a single setting."

***

### 🔥 Common Follow-Ups

| #  | Question                                                               | Quick Answer                                                                                                                        |
| -- | ---------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| 1  | **Difference between Job and Deployment?**                             | Job = runs to completion (`restartPolicy: Never` or `OnFailure`). Deployment = always running (`restartPolicy: Always`).            |
| 2  | **What is `backoffLimit`?**                                            | Max number of failed pod retries before the Job is marked Failed. Default: 6. Uses exponential backoff.                             |
| 3  | **Difference between `restartPolicy: OnFailure` and `Never` in Jobs?** | `OnFailure` restarts container in same pod (faster, simpler). `Never` creates new pod (separate logs, better for debugging).        |
| 4  | **What's `completions` vs `parallelism`?**                             | `completions` = total successful pods needed. `parallelism` = how many run concurrently.                                            |
| 5  | **What is an Indexed Job?**                                            | K8s 1.24+. Each pod gets a unique `JOB_COMPLETION_INDEX` env var. Used for sharded work like ML training, batch processing.         |
| 6  | **How do you ensure a Job doesn't run forever?**                       | Set `activeDeadlineSeconds`. Job is killed after this duration regardless of retries.                                               |
| 7  | **How do you clean up completed Jobs?**                                | Set `ttlSecondsAfterFinished` — TTL controller deletes the Job (and its pods) after the specified seconds.                          |
| 8  | **What's `concurrencyPolicy` in CronJob?**                             | `Allow` (default, overlap), `Forbid` (skip if previous still running), `Replace` (kill old, start new).                             |
| 9  | **Why might a CronJob miss a scheduled run?**                          | Controller outage, cluster down, `startingDeadlineSeconds` exceeded, or `concurrencyPolicy: Forbid` with previous run still active. |
| 10 | **What's `startingDeadlineSeconds`?**                                  | Max delay (in seconds) before a missed Job is considered too late to run. Without it, the Job may never start.                      |
| 11 | **Are CronJob times in UTC or cluster timezone?**                      | **UTC by default**. K8s 1.27+ supports `timeZone: "Asia/Kolkata"` to specify locally.                                               |
| 12 | **How do you manually trigger a CronJob?**                             | `kubectl create job manual-001 --from=cronjob/<name>`.                                                                              |
| 13 | **What's `podFailurePolicy`?**                                         | K8s 1.26+ feature for granular failure handling — fail immediately on specific exit codes, ignore failures from node disruptions.   |
| 14 | **How do you pause a CronJob without deleting it?**                    | `kubectl patch cronjob <name> -p '{"spec":{"suspend":true}}'`.                                                                      |
| 15 | **What happens if a Job's pod is OOMKilled?**                          | Pod fails, counted against `backoffLimit`, retried. Fix: increase `resources.limits.memory` or check for memory leaks.              |

***

### 💡 Senior Scenario Questions

**Scenario 1: "Your nightly backup CronJob hasn't run for 3 days. How do you investigate?"**

> Methodical debugging:
>
> 1. `kubectl describe cronjob postgres-backup -n db` — check Status section for `LastScheduleTime`, `Active`, and any error events.
> 2. Check if **suspended**: `kubectl get cronjob -n db -o jsonpath='{.spec.suspend}'`.
> 3. Check **CronJob controller logs**: `kubectl logs -n kube-system -l component=kube-controller-manager`.
> 4. Look for `MissedSchedule` events — happens when controller wasn't running and `startingDeadlineSeconds` expired.
> 5. Are old Jobs accumulating beyond `successfulJobsHistoryLimit`? Indicates broken cleanup.
> 6. Verify schedule with `crontab.guru` — typos like `0 25 * * *` (invalid hour) cause silent failures.
> 7. Check if `concurrencyPolicy: Forbid` is blocking — is there a stuck Job from 3 days ago?
> 8. **Recovery**: manually trigger via `kubectl create job --from=cronjob/postgres-backup recovery-$(date +%s) -n db`.
> 9. **Long-term fix**: add Prometheus alert on `time() - kube_cronjob_status_last_schedule_time > 90000` (no schedule in 25 hours).

***

**Scenario 2: "Your DB migration Job ran 3 times and corrupted the database. What went wrong?"**

> Root cause: **the migration script was not idempotent**, and Kubernetes retried it on transient failures.
>
> Forensics:
>
> 1. `kubectl describe job migrate` → check retry count and failure reasons.
> 2. Look at `kubectl logs` for each pod (use `--previous` for crashed containers).
>
> Common causes:
>
> * Migration script doesn't check schema version before applying.
> * Container exits 0 prematurely (before migration commits), then K8s thinks it succeeded but data wasn't fully applied.
> * Migration tool (Flyway, Liquibase) wasn't properly initialized, so it re-ran completed migrations.
>
> **Fix going forward**:
>
> 1. **Use idempotent migration tools** — Flyway/Liquibase track applied versions in a metadata table; safe to retry.
> 2. **Set `backoffLimit: 0`** for migrations — fail fast rather than retry corruption.
> 3. **Always backup before migration** — chain a backup Job + migration Job with init container check.
> 4. **Test in staging first** with identical Job spec.
> 5. **Migration in transactions** — if your DB supports it, wrap the whole thing in BEGIN/COMMIT.
> 6. **Recovery**: restore from last good backup, fix the migration, re-run.

***

**Scenario 3: "How would you run a batch ML training task across 100 GPU pods?"**

> Indexed Job pattern:
>
> ```yaml
> spec:
>   completions: 100
>   parallelism: 100              # all at once
>   completionMode: Indexed
>   template:
>     spec:
>       nodeSelector:
>         accelerator: nvidia-gpu
>       containers:
>       - name: trainer
>         image: ml-trainer:1.0
>         command:
>         - python
>         - train.py
>         - --shard-id=$(JOB_COMPLETION_INDEX)
>         - --total-shards=100
>         resources:
>           limits:
>             nvidia.com/gpu: 1
> ```
>
> Each pod processes 1/100th of the dataset. Combined with:
>
> * **Persistent volume** for shared model artifacts (RWX).
> * **PriorityClass** so training preempts lower-priority workloads.
> * **`activeDeadlineSeconds`** so a stuck training run doesn't burn GPU costs forever.
> * **podFailurePolicy** to ignore node disruption failures (spot node preemption shouldn't fail the whole Job).
> * **podDisruptionBudget** to prevent voluntary disruptions during training.

***

**Scenario 4: "Two backup CronJobs are running on the same DB and corrupting each other. How do you fix?"**

> Root cause: **`concurrencyPolicy: Allow`** (default), or two separate CronJobs targeting the same DB.
>
> Fix:
>
> 1. **Immediately** set `concurrencyPolicy: Forbid` on the CronJob.
> 2. If two CronJobs exist, **consolidate to one** and schedule appropriately.
> 3. **Application-level locking** — use a row lock in a `backup_locks` table; the script checks before starting.
> 4. **Verify schedule** doesn't overlap with maintenance windows or other heavy operations.
> 5. **Monitoring**: alert when two backup Jobs are simultaneously Active.

***

**Scenario 5: "Your team wants to backup 50 databases. Would you create 50 CronJobs?"**

> Options ranked by maintainability:
>
> 1. **❌ 50 separate CronJobs** — high YAML duplication, hard to update.
>
> 2. **✅ Helm chart with a loop** (Day 11 topic):
>    ```yaml
>    {{- range .Values.databases }}
>    apiVersion: batch/v1
>    kind: CronJob
>    metadata:
>      name: backup-{{ .name }}
>    spec:
>      # ...
>    {{- end }}
>    ```
>    One template, one values.yaml, 50 CronJobs generated.
>
> 3. **✅ Single CronJob with Indexed Job pattern** — one CronJob spawns 50 indexed pods, each reads its own DB config from a ConfigMap by index.
>
> 4. **✅✅ Best**: an **operator** like Stash or Velero — declarative `BackupConfiguration` per DB, operator handles scheduling, retention, validation, off-site replication.
>
> For an enterprise-scale answer, I'd combine Helm for templating + Velero/Stash for backup orchestration + Prometheus for monitoring + Azure Blob lifecycle policies for retention/archival.

***

## 📊 Decision Matrix

| Need                             | Use                                         |
| -------------------------------- | ------------------------------------------- |
| One-off DB migration             | Job (`backoffLimit: 0`)                     |
| Batch processing 100 files       | Job (`completions: 100`, `parallelism: 10`) |
| Sharded ML training              | Indexed Job                                 |
| Queue worker (RabbitMQ consumer) | Job (parallelism only, no completions)      |
| Periodic DB backup               | CronJob (`concurrencyPolicy: Forbid`)       |
| Hourly log archival              | CronJob (`@hourly`)                         |
| Continuous data sync             | **Deployment**, not Job/CronJob             |
| Need replication & failover      | **StatefulSet**, not Job                    |
| Always-running daemon            | **Deployment** or **DaemonSet**, not Job    |

***

# 🌐 Day 11: Ingress & Ingress Controllers

Welcome to Day 11, Rajesh! On Day 4 you learned that LoadBalancer Services give you external access — but if you have 20 microservices, that's **20 expensive cloud LBs**. Today you'll master **Ingress** — the L7 routing layer that solves this elegantly. You'll route by hostname, path, terminate TLS, and stand up production-grade external access with a single LoadBalancer. This is one of the **most common Senior DevOps interview topics**. Let's go! 🚀

***

## 📚 Part 1: Theory — Why Ingress Exists

### The Problem Without Ingress

Imagine 5 microservices needing external access:

```
❌ EXPENSIVE & MESSY:
┌─────────┐    ┌────────────┐    ┌─────────────┐
│ Client  │ →  │ Azure LB 1 │ →  │ api-svc     │  $$$
├─────────┤    ├────────────┤    ├─────────────┤
│ Client  │ →  │ Azure LB 2 │ →  │ web-svc     │  $$$
├─────────┤    ├────────────┤    ├─────────────┤
│ Client  │ →  │ Azure LB 3 │ →  │ admin-svc   │  $$$
├─────────┤    ├────────────┤    ├─────────────┤
│ Client  │ →  │ Azure LB 4 │ →  │ payments-svc│  $$$
├─────────┤    ├────────────┤    ├─────────────┤
│ Client  │ →  │ Azure LB 5 │ →  │ catalog-svc │  $$$
└─────────┘    └────────────┘    └─────────────┘

5 Load Balancers × ~$25/month each = $125/month + 5 Public IPs
+ No SSL termination at LB
+ No path-based routing
+ No WAF
```

### The Ingress Solution

```
✅ ELEGANT & COST-EFFECTIVE:

┌─────────┐
│ Client  │ ──HTTPS──┐
└─────────┘          │
                     ▼
              ┌──────────────┐
              │  1 LoadBalancer │ ← single public IP
              │  (Azure LB)     │   single cost
              └──────┬──────────┘
                     │
                     ▼
              ┌──────────────────┐
              │ Ingress Controller│ ← NGINX/AGIC pod(s)
              │  (deciphers L7)   │   running in cluster
              └──────┬───────────┘
                     │ routes by hostname/path
        ┌────────────┼─────────────┬────────────┐
        ▼            ▼             ▼            ▼
   ┌─────────┐ ┌─────────┐  ┌──────────┐  ┌──────────┐
   │ api-svc │ │ web-svc │  │admin-svc │  │payments  │
   │ (Cluster│ │ (Cluster│  │ (Cluster │  │   svc    │
   │   IP)   │ │   IP)   │  │   IP)    │  │          │
   └─────────┘ └─────────┘  └──────────┘  └──────────┘

1 LB + 1 Public IP + SSL termination + Path routing + WAF
```

***

### 🎯 Ingress vs Service — The Layer Difference

| Aspect                  | Service (LoadBalancer)             | Ingress                                   |
| ----------------------- | ---------------------------------- | ----------------------------------------- |
| **OSI Layer**           | L4 (TCP/UDP)                       | L7 (HTTP/HTTPS)                           |
| **Routes by**           | Port + Protocol                    | Hostname + Path + Headers                 |
| **TLS termination**     | Pass-through only                  | ✅ Native                                  |
| **One external IP for** | 1 service                          | Many services                             |
| **Cost**                | 1 LB per service                   | 1 LB for all                              |
| **Cloud independence**  | Cloud-specific                     | Standard K8s API                          |
| **L7 features**         | None                               | Rewrite, redirect, auth, rate limit, CORS |
| **Use case**            | TCP databases, gRPC, raw protocols | HTTP/HTTPS web traffic                    |

***

### 🧩 Two Components You Need

#### 1. **Ingress Resource** (declarative routing rules)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 80
```

This is **just configuration** — it does nothing on its own.

#### 2. **Ingress Controller** (the actual proxy)

A pod (or pods) running in your cluster that:

* Watches Ingress resources
* Translates them into proxy configuration
* Receives external traffic and routes it

> 🎯 **Critical Senior insight**: Kubernetes ships **no built-in Ingress Controller**. You **MUST install one**. Common options:

| Controller                                        | Notes                                                        |
| ------------------------------------------------- | ------------------------------------------------------------ |
| **NGINX Ingress Controller**                      | Most popular OSS option (community-maintained)               |
| **NGINX Plus Ingress**                            | Enterprise version with advanced features                    |
| **Application Gateway Ingress Controller (AGIC)** | Azure-native, integrates with App Gateway WAF                |
| **Traefik**                                       | Modern, dynamic config, great for K8s-native workflows       |
| **HAProxy Ingress**                               | High-performance, mature                                     |
| **Istio Gateway**                                 | Part of service mesh (different API: Gateway/VirtualService) |
| **Contour**                                       | Envoy-based, by VMware                                       |
| **Kong Ingress**                                  | API management features built-in                             |

***

### 🛣️ Routing Patterns

#### Path-Based Routing

```yaml
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend: {service: {name: api-svc, port: {number: 80}}}
      - path: /web
        pathType: Prefix
        backend: {service: {name: web-svc, port: {number: 80}}}
```

```
example.com/api/*    → api-svc
example.com/web/*    → web-svc
```

#### Host-Based (Subdomain) Routing

```yaml
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - {path: /, pathType: Prefix, backend: {service: {name: api-svc, port: {number: 80}}}}
  - host: web.example.com
    http:
      paths:
      - {path: /, pathType: Prefix, backend: {service: {name: web-svc, port: {number: 80}}}}
```

```
api.example.com/*    → api-svc
web.example.com/*    → web-svc
```

#### Combined (Most Common)

```yaml
spec:
  rules:
  - host: shop.example.com
    http:
      paths:
      - {path: /api/v1, pathType: Prefix, backend: ...}
      - {path: /api/v2, pathType: Prefix, backend: ...}
      - {path: /, pathType: Prefix, backend: ...}
```

***

### 🎚️ Path Types — Subtle but Important

| Type                       | Behavior                  | Example                                               |
| -------------------------- | ------------------------- | ----------------------------------------------------- |
| **Exact**                  | URL must match exactly    | `/api` matches only `/api`, NOT `/api/users`          |
| **Prefix**                 | Element-wise prefix match | `/api` matches `/api`, `/api/users`, `/api/v1/orders` |
| **ImplementationSpecific** | Up to the controller      | Often regex/glob with NGINX                           |

> 💡 **Use `Prefix`** for most cases. `Exact` is rarely what you want.

***

### 🔐 TLS Termination

TLS termination at the Ingress means:

* Client → Ingress: encrypted (HTTPS)
* Ingress → backend Service: usually plain HTTP (inside cluster)

```yaml
spec:
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls-cert      # K8s Secret of type kubernetes.io/tls
  rules:
  - host: api.example.com
    http:
      paths: [...]
```

The `Secret` contains:

```yaml
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-private-key>
```

For production: use **cert-manager** to auto-provision certs from Let's Encrypt or Azure Key Vault.

***

### 🏷️ IngressClass — Multi-Controller Support

What if you have **NGINX for public traffic** and **AGIC for internal**? Use `IngressClass`:

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
spec:
  controller: k8s.io/ingress-nginx
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  ingressClassName: nginx        # ← uses NGINX, not other controllers
  rules: [...]
```

> 🎯 **Senior tip**: Always set `ingressClassName` explicitly. The "default" IngressClass behavior can be confusing in multi-controller setups.

***

### 🔄 Common Ingress Annotations (Controller-Specific)

NGINX-specific annotations modify behavior:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/8"
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://app.example.com"
```

> ⚠️ **Caveat**: Annotations are **controller-specific** — what works for NGINX won't work for AGIC. Read the controller's docs.

***

### 🆕 Gateway API — The Future

Kubernetes is moving toward the **Gateway API** as the next-generation replacement for Ingress (more expressive, role-oriented). Key resources: `GatewayClass`, `Gateway`, `HTTPRoute`, `TCPRoute`. Still GA only since 2023, but worth knowing.

***

## 🛠️ Part 2: Hands-On — Production Ingress Setup

### Step 1: Install NGINX Ingress Controller

**On Minikube:**

```powershell
minikube addons enable ingress
kubectl get pods -n ingress-nginx
```

**On AKS / general clusters (via Helm — preview of Day 12):**

```powershell
# Add the Helm repo
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install in a dedicated namespace
helm install ingress-nginx ingress-nginx/ingress-nginx `
  --namespace ingress-nginx `
  --create-namespace `
  --set controller.replicaCount=2 `
  --set controller.service.type=LoadBalancer

# Wait for external IP
kubectl get svc -n ingress-nginx -w
# EXTERNAL-IP transitions from <pending> to actual IP
```

**On Kind (local):**

```powershell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Wait until ready
kubectl wait --namespace ingress-nginx `
  --for=condition=ready pod `
  --selector=app.kubernetes.io/component=controller `
  --timeout=120s
```

#### Verify

```powershell
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
kubectl get ingressclass
# IngressClass "nginx" should exist
```

***

### Step 2: Deploy 3 Demo Services

```yaml
# 01-demo-apps.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo
---
# App 1: API
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-app
  namespace: demo
spec:
  replicas: 2
  selector:
    matchLabels: {app: api-app}
  template:
    metadata:
      labels: {app: api-app}
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo:1.0
        args:
        - -text=Hello from API service! v1.0
        - -listen=:5678
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: api-svc
  namespace: demo
spec:
  selector: {app: api-app}
  ports:
  - port: 80
    targetPort: 5678
---
# App 2: WEB
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: demo
spec:
  replicas: 2
  selector:
    matchLabels: {app: web-app}
  template:
    metadata:
      labels: {app: web-app}
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo:1.0
        args:
        - -text=Hello from WEB frontend!
        - -listen=:5678
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: web-svc
  namespace: demo
spec:
  selector: {app: web-app}
  ports:
  - port: 80
    targetPort: 5678
---
# App 3: ADMIN
apiVersion: apps/v1
kind: Deployment
metadata:
  name: admin-app
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels: {app: admin-app}
  template:
    metadata:
      labels: {app: admin-app}
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo:1.0
        args:
        - -text=Admin Portal - Restricted Access!
        - -listen=:5678
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: admin-svc
  namespace: demo
spec:
  selector: {app: admin-app}
  ports:
  - port: 80
    targetPort: 5678
```

```powershell
kubectl apply -f 01-demo-apps.yaml
kubectl get pods,svc -n demo
```

***

### Step 3: Path-Based Routing Ingress

```yaml
# 02-path-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-ingress
  namespace: demo
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: demo.local                  # use hostnames you control (add to /etc/hosts)
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 80
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-svc
            port:
              number: 80
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: admin-svc
            port:
              number: 80
```

```powershell
kubectl apply -f 02-path-ingress.yaml
kubectl get ingress -n demo
```

**Find the Ingress IP:**

```powershell
# On Minikube
minikube ip
# Returns something like 192.168.49.2

# On AKS / cloud
kubectl get svc -n ingress-nginx
# Look for EXTERNAL-IP of ingress-nginx-controller
```

**Map the hostname locally** by editing `C:\Windows\System32\drivers\etc\hosts`:

```
192.168.49.2  demo.local
```

**Test:**

```powershell
curl http://demo.local/api
# → Hello from API service! v1.0

curl http://demo.local/web
# → Hello from WEB frontend!

curl http://demo.local/admin
# → Admin Portal - Restricted Access!
```

***

### Step 4: Host-Based Routing

```yaml
# 03-host-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-ingress
  namespace: demo
spec:
  ingressClassName: nginx
  rules:
  - host: api.demo.local
    http:
      paths:
      - {path: /, pathType: Prefix, backend: {service: {name: api-svc, port: {number: 80}}}}
  - host: web.demo.local
    http:
      paths:
      - {path: /, pathType: Prefix, backend: {service: {name: web-svc, port: {number: 80}}}}
  - host: admin.demo.local
    http:
      paths:
      - {path: /, pathType: Prefix, backend: {service: {name: admin-svc, port: {number: 80}}}}
```

Add to `hosts` file:

```
192.168.49.2  api.demo.local web.demo.local admin.demo.local
```

```powershell
kubectl apply -f 03-host-ingress.yaml

curl http://api.demo.local
curl http://web.demo.local
curl http://admin.demo.local
```

***

### Step 5: TLS Termination with Self-Signed Cert

#### Generate a self-signed certificate:

```powershell
# Using OpenSSL (install via Chocolatey if missing: choco install openssl)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 `
  -keyout tls.key -out tls.crt `
  -subj "/CN=secure.demo.local/O=demo" `
  -addext "subjectAltName=DNS:secure.demo.local"

# Create K8s TLS Secret
kubectl create secret tls demo-tls `
  --cert=tls.crt `
  --key=tls.key `
  -n demo
```

#### Ingress with TLS:

```yaml
# 04-tls-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  namespace: demo
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - secure.demo.local
    secretName: demo-tls
  rules:
  - host: secure.demo.local
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port: {number: 80}
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-svc
            port: {number: 80}
```

Add to hosts file:

```
192.168.49.2  secure.demo.local
```

```powershell
kubectl apply -f 04-tls-ingress.yaml

# Test (use -k to skip cert validation for self-signed)
curl -k https://secure.demo.local/api
curl -k https://secure.demo.local/web

# HTTP requests auto-redirect to HTTPS
curl -kIL http://secure.demo.local/api
# Should show 308 redirect → HTTPS
```

***

### Step 6: Production-Grade Annotations

```yaml
# 05-prod-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: production-ingress
  namespace: demo
  annotations:
    # SSL
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    
    # Body size (e.g., file uploads)
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    
    # Timeouts
    nginx.ingress.kubernetes.io/proxy-read-timeout: "120"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "120"
    
    # Rate limiting (requests per second per IP)
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-connections: "20"
    
    # IP whitelisting (restrict to corporate network)
    # nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/8,192.168.0.0/16"
    
    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://app.demo.local"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS"
    
    # Custom headers
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: SAMEORIGIN";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "X-XSS-Protection: 1; mode=block";
spec:
  ingressClassName: nginx
  tls:
  - hosts: [secure.demo.local]
    secretName: demo-tls
  rules:
  - host: secure.demo.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-svc
            port: {number: 80}
```

***

### Step 7: Sticky Sessions (Session Affinity)

```yaml
# 06-sticky-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sticky-ingress
  namespace: demo
  annotations:
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "INGRESSCOOKIE"
    nginx.ingress.kubernetes.io/session-cookie-expires: "172800"
    nginx.ingress.kubernetes.io/session-cookie-max-age: "172800"
spec:
  ingressClassName: nginx
  rules:
  - host: sticky.demo.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-svc
            port: {number: 80}
```

Now a client gets pinned to one pod for the cookie's lifetime — useful for stateful apps that haven't fully externalized session state.

***

### Step 8: Multi-Tenant Ingress with cert-manager (Production Pattern)

```yaml
# 07-cert-manager-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: rajesh@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prod-ingress
  namespace: demo
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.realdomain.com
    secretName: api-tls-cert       # cert-manager AUTO-CREATES this!
  rules:
  - host: api.realdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: {name: api-svc, port: {number: 80}}
```

> 🚀 **In production this is the gold standard**: cert-manager + Let's Encrypt = **fully automated, free, auto-renewing TLS certificates**. Set it once, forget it.

***

### Step 9: Debugging an Ingress

```powershell
# View ingress
kubectl describe ingress path-ingress -n demo

# Check NGINX controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller -f

# Get the actual NGINX configuration generated by the controller
$ctrl = kubectl get pods -n ingress-nginx -l app.kubernetes.io/component=controller -o jsonpath='{.items[0].metadata.name}'
kubectl exec -n ingress-nginx $ctrl -- cat /etc/nginx/nginx.conf | findstr "demo.local"

# Test from inside the cluster
kubectl run debug --image=curlimages/curl:8.5.0 -it --rm --restart=Never -- sh
# Inside:
curl -H "Host: demo.local" http://ingress-nginx-controller.ingress-nginx.svc.cluster.local/api
```

***

## 💻 Part 3: Commands Cheat Sheet

### Ingress Operations

```powershell
# CRUD
kubectl apply -f ingress.yaml
kubectl get ingress -A
kubectl get ing
kubectl describe ingress <name> -n <ns>
kubectl delete ingress <name>

# IngressClass
kubectl get ingressclass
kubectl describe ingressclass nginx

# TLS Secret
kubectl create secret tls <name> --cert=tls.crt --key=tls.key
kubectl get secret -n demo
```

### Controller Operations

```powershell
# Pods
kubectl get pods -n ingress-nginx
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller

# Service (find external IP)
kubectl get svc -n ingress-nginx

# Reload config (rarely needed — controller auto-reloads)
kubectl rollout restart deployment ingress-nginx-controller -n ingress-nginx
```

### Quick Test

```powershell
# Get external IP
$ingressIP = kubectl get svc ingress-nginx-controller -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

# Send request with host header
curl -H "Host: demo.local" http://$ingressIP/api
```

***

## 🎤 Part 4: Interview Prep

### ⭐ Star Question: *"How does Ingress differ from a Service?"*

**Senior DevOps model answer:**

> "The difference comes down to **OSI layer, capabilities, and cost model**.
>
> A **Service** operates at **Layer 4** — TCP/UDP. It exposes pods using a port and protocol. Specifically, a Service of type `LoadBalancer` provisions a cloud load balancer that forwards raw TCP traffic to your pods. It's a great fit for non-HTTP workloads — databases, gRPC streams, raw TCP. But it has no awareness of HTTP semantics. If you want path-based routing or TLS termination, the Service can't do that.
>
> An **Ingress** operates at **Layer 7** — HTTP/HTTPS. It's an **API object** that defines routing rules: 'requests for `api.example.com/v1` go to api-svc, `api.example.com/v2` go to api-v2-svc, and so on.' The actual proxy work is done by an **Ingress Controller** — NGINX, AGIC, Traefik — which is a pod (or set of pods) running in the cluster. The Ingress is just config; the Controller is the engine.
>
> Here's the **practical impact**: imagine 20 microservices needing external HTTPS. With LoadBalancer Services, you'd provision **20 cloud LBs and 20 public IPs** — costly, hard to manage, no SSL termination at the LB layer. With Ingress, you provision **one LoadBalancer Service** that points to the Ingress Controller; the Controller routes by hostname/path to the right ClusterIP Service. Single LB, single IP, single cert management workflow — far cheaper and more elegant.
>
> **Ingress also adds L7 features Services can't provide**: SSL termination, host-based routing, path-based routing, redirects, rewrites, basic auth, rate limiting, CORS, WAF integration, sticky sessions, header manipulation. All of this is impossible at L4.
>
> **However**, Ingress has its own caveats. The Kubernetes Ingress API is **only HTTP/HTTPS** — for TCP/UDP traffic, you still need LoadBalancer Services (or NGINX's TCP/UDP ConfigMap extension). Annotations are **controller-specific** — what works for NGINX won't work for AGIC. Also, the Ingress API itself is being superseded by the new **Gateway API**, which is more expressive and role-oriented.
>
> So in my decision matrix: **Use Service-type LoadBalancer for raw TCP/UDP traffic or single-service exposure. Use Ingress for HTTP/HTTPS with multiple services — which is most production microservice traffic.** In a typical AKS setup, I deploy NGINX Ingress (or AGIC on Azure) backed by a single Standard SKU LoadBalancer, with cert-manager for automated TLS, plus annotations for rate limiting, WAF, and CORS. That's the **standard production pattern** for any team with more than 2–3 web services."

***

### 🔥 Common Follow-Ups

| #  | Question                                                                           | Quick Answer                                                                                                                                                                                              |
| -- | ---------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1  | **What's an Ingress Controller?**                                                  | A pod that watches Ingress resources and translates them into proxy config (NGINX, Traefik, AGIC). Kubernetes ships none — you must install one.                                                          |
| 2  | **Does Ingress replace Services?**                                                 | No — Ingress routes TO Services. You still need ClusterIP Services for each backend.                                                                                                                      |
| 3  | **What's the difference between `Prefix` and `Exact` pathType?**                   | `Prefix` matches `/api` and `/api/anything`. `Exact` matches only `/api`.                                                                                                                                 |
| 4  | **How do you enable TLS on Ingress?**                                              | Create a Secret of type `kubernetes.io/tls` with cert + key, then reference it in `spec.tls`. Or use cert-manager for automation.                                                                         |
| 5  | **What is `IngressClass`?**                                                        | Identifies which Ingress Controller handles an Ingress when you have multiple. Set via `spec.ingressClassName`.                                                                                           |
| 6  | **Can Ingress handle TCP/UDP traffic?**                                            | The standard API: no — HTTP/HTTPS only. NGINX supports TCP/UDP via a ConfigMap extension. For native L4, use LoadBalancer Services.                                                                       |
| 7  | **What's the difference between NGINX OSS and AGIC?**                              | NGINX = generic, runs in-cluster, vendor-neutral. AGIC = Azure-native, leverages Azure Application Gateway (WAF, autoscaling) — Ingress traffic doesn't enter the cluster network until backend services. |
| 8  | **How does cert-manager work?**                                                    | It's a K8s operator that watches Ingress resources, requests certs from issuers (Let's Encrypt, Azure Key Vault, internal CA), stores them as Secrets, auto-renews before expiry.                         |
| 9  | **What are common Ingress annotations?**                                           | `rewrite-target`, `ssl-redirect`, `proxy-body-size`, `rate-limit`, `whitelist-source-range`, `enable-cors`. All controller-specific.                                                                      |
| 10 | **How do you do canary deployments with NGINX Ingress?**                           | Add `nginx.ingress.kubernetes.io/canary: "true"` + `canary-weight: "10"` to a second Ingress pointing to the new version. Or use Argo Rollouts/Flagger.                                                   |
| 11 | **What's `externalTrafficPolicy: Local` for the Ingress Controller's LB Service?** | Preserves client source IP for the Ingress Controller (important for rate limiting and IP allowlisting). Trade-off: pods on nodes without controller pods don't get traffic.                              |
| 12 | **Why might Ingress traffic bypass NetworkPolicies?**                              | If the Ingress Controller is in a different namespace, NetworkPolicies in your app namespace must explicitly allow ingress from the controller's namespace.                                               |
| 13 | **What's the difference between Ingress and a Service Mesh?**                      | Ingress = north-south traffic (external → cluster). Service Mesh (Istio, Linkerd) = east-west traffic (pod to pod) plus advanced policy, mTLS, observability.                                             |
| 14 | **What's the Gateway API?**                                                        | The next-gen replacement for Ingress. More expressive, role-oriented (`GatewayClass`, `Gateway`, `HTTPRoute`). GA since 2023; still gaining adoption.                                                     |
| 15 | **How do you do sticky sessions?**                                                 | NGINX annotations: `affinity: cookie`, `session-cookie-name`. Pins client to one backend pod via cookie.                                                                                                  |

***

### 💡 Senior Scenario Questions

**Scenario 1: "Your Ingress returns 502 Bad Gateway. How do you debug?"**

> Systematic approach:
>
> 1. **Verify backend Service**: `kubectl get svc -n <ns>`. Does the Service exist with the name referenced in Ingress?
> 2. **Check Endpoints**: `kubectl get endpoints <svc-name>`. If empty → Service selector doesn't match any healthy pods. Pods may be unhealthy (readiness failing) or label mismatch.
> 3. **Pod readiness**: `kubectl get pods -l <selector>`. Are they `Ready 1/1`?
> 4. **Controller logs**: `kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller`. Look for upstream errors.
> 5. **Test from inside cluster**: `kubectl run curl --image=curlimages/curl -it --rm -- curl http://api-svc.demo.svc.cluster.local`. If this also fails, problem is backend, not Ingress.
> 6. **Check Ingress definition**: `kubectl describe ingress <name>`. Port number correct? Service name correct? Namespace match (Ingress and Service must be in same NS)?
> 7. **NetworkPolicy blocking**: ingress controller might be denied from reaching the pods.
> 8. **Resource limits**: backend pods OOMKilled? Check events.

***

**Scenario 2: "How would you implement zero-downtime deployment with Ingress for a blue-green release?"**

> Multiple approaches:
>
> **Approach A — Service selector swap**:
>
> 1. Deploy `app-v2` Deployment alongside `app-v1`.
> 2. Both have ClusterIP Services: `app-v1-svc` and `app-v2-svc`.
> 3. Ingress points to a "production" Service whose `selector` decides who serves.
> 4. To cut over: patch the Service selector from `version: v1` to `version: v2`. Instant cutover.
>
> **Approach B — NGINX canary with two Ingresses**:
>
> ```yaml
> # Primary Ingress → v1
> # Second Ingress with canary annotations → v2
> annotations:
>   nginx.ingress.kubernetes.io/canary: "true"
>   nginx.ingress.kubernetes.io/canary-weight: "10"  # 10% to v2
> ```
>
> Gradually shift from 10% → 50% → 100%.
>
> **Approach C — Argo Rollouts** (production gold standard):
> Replace Deployment with Rollout CR. It manipulates Service selectors, runs analysis steps with Prometheus, auto-promotes or rolls back based on SLO metrics. Combine with Ingress for traffic shifting.

***

**Scenario 3: "Your team wants to expose 50 microservices to the internet on AKS. What's your architecture?"**

> **Single Ingress Controller pattern**:
>
> 1. Deploy **NGINX Ingress** (or AGIC) backed by **one Standard SKU Azure Load Balancer**.
> 2. All 50 microservices have ClusterIP Services.
> 3. Each service has an Ingress resource (or one Ingress with 50 rules — split based on team ownership for RBAC).
> 4. **cert-manager** + **Let's Encrypt DNS-01 challenge** for wildcard cert `*.api.example.com` or per-service certs.
> 5. **External DNS operator** auto-syncs hostnames to Azure DNS.
> 6. **Annotations** for rate limiting, WAF rules, IP allowlisting (per service).
> 7. **Multi-zone HA**: 3+ Ingress Controller replicas across zones, with PodAntiAffinity.
> 8. **Cost optimization**: 1 LB (~~$25/month) vs 50 (~~$1,250/month) — \~98% savings.
> 9. **Observability**: NGINX metrics → Prometheus → Grafana for per-service traffic, error rates, latencies.
> 10. **WAF**: either use AGIC + Application Gateway WAF, or ModSecurity in NGINX.

***

**Scenario 4: "How do you handle multi-tenancy with Ingress — different teams shouldn't see each other's Ingress configs?"**

> Multi-layer:
>
> 1. **Namespace per team** — Ingress objects are namespaced. Team A's Ingress is in `team-a` namespace; can only reach Services in same namespace by default.
> 2. **RBAC** — Roles allow `get/list/create/update/delete` on `ingresses` in their own namespace only.
> 3. **OPA Gatekeeper / Kyverno policies** — enforce that Team A can only use hostnames matching `*.team-a.example.com`. Prevents Team A from claiming `team-b.example.com`.
> 4. **IngressClass restriction** — Team A can only use `ingressClassName: team-a-nginx` (separate controller instance for sensitive workloads).
> 5. **Shared controller, separate certs** — cert-manager `Certificate` resources are namespaced; each team manages their own.
> 6. **Audit logging** for any Ingress create/update operations.

***

**Scenario 5: "Your Ingress was working yesterday, today suddenly all requests are 404. What changed?"**

> Investigation:
>
> 1. **Recent changes?** Check Git history, ArgoCD/Flux sync logs.
> 2. **`kubectl get ingress -A`** — does the Ingress still exist?
> 3. **`kubectl describe ingress <name>`** — check Rules, are hostnames correct?
> 4. **IngressClass missing or changed?** A common bug — if `ingressClassName` is removed or `IngressClass` deleted, the controller stops handling it.
> 5. **Controller upgrade** — did NGINX upgrade overnight? Check version, breaking changes in annotations.
> 6. **DNS** — did the hostname resolve elsewhere? `nslookup api.example.com`.
> 7. **Certificate expiry** — for HTTPS, expired cert triggers connection failure (but usually shows in browser, not 404).
> 8. **NGINX config errors** — `kubectl logs -n ingress-nginx ...` for syntax errors, often shown as `nginx: [emerg] ...`.
> 9. **Service / Endpoints** — backend Service was deleted/renamed?
> 10. **Common culprit**: annotations changed format between NGINX versions — `kubernetes.io/ingress.class` was deprecated in favor of `spec.ingressClassName`.

***

## 📊 Decision Matrix: Service vs Ingress vs Gateway API

| Need                                                 | Use                                          |
| ---------------------------------------------------- | -------------------------------------------- |
| Expose one HTTP service externally, quickly          | LoadBalancer Service (dev) or Ingress (prod) |
| Expose many HTTP services on one IP                  | **Ingress**                                  |
| TLS termination                                      | **Ingress** + cert-manager                   |
| TCP/UDP (databases, gRPC streaming)                  | **LoadBalancer Service**                     |
| Path/host-based HTTP routing                         | **Ingress**                                  |
| Advanced traffic management (mTLS, canary by header) | **Service Mesh** (Istio/Linkerd)             |
| Future-proof, role-oriented routing                  | **Gateway API**                              |
| Internal-only east-west routing                      | ClusterIP Services + DNS                     |

***

## 🧠 Production Best Practices

✅ **Use cert-manager** + Let's Encrypt for automated TLS  
✅ **Single Ingress Controller** with multiple Ingress resources — not multiple LBs  
✅ **`force-ssl-redirect: true`** on all production Ingresses  
✅ **Set `proxy-body-size`** appropriately (default 1m blocks uploads!)  
✅ **Rate limiting** per Ingress to prevent DoS  
✅ **WAF**: AGIC + Azure Application Gateway, or ModSecurity for NGINX  
✅ **Multi-replica Ingress Controller** with PodAntiAffinity across nodes/zones  
✅ **Monitor** controller metrics: `nginx_ingress_controller_requests`, error rates, p99 latency  
✅ **HorizontalPodAutoscaler** on Ingress Controller for traffic spikes  
✅ **`externalTrafficPolicy: Local`** on the controller Service to preserve client IPs  
✅ **PodDisruptionBudget** so cluster ops don't kill all controller pods at once

***

# 🌐 Day 12: Kubernetes Networking Deep Dive

Welcome to Day 12, Rajesh! Today is **the day** that separates Kubernetes users from Kubernetes engineers. Networking is the most-asked, least-understood topic in K8s interviews. By the end of today, you'll trace a packet from one pod to another across nodes, explain why every pod gets its own IP, and articulate the Kubernetes networking model better than 90% of candidates. This is **deep stuff** — let's go! 🚀

***

## 📚 Part 1: Theory — The K8s Networking Model

### The 4 Fundamental Requirements

Kubernetes mandates that any networking solution must satisfy these **4 rules** — no NAT, full reachability:

```
1️⃣  Pod-to-Pod (same node)        ← every pod gets a unique IP
2️⃣  Pod-to-Pod (across nodes)     ← no NAT, direct reachability  
3️⃣  Pod-to-Service                ← virtual IP, load-balanced
4️⃣  External-to-Service           ← LoadBalancer, NodePort, Ingress
```

This is enforced by the **CNI (Container Network Interface)** plugin.

***

### 🎯 The 5 Networks in Every Cluster

```
┌────────────────────────────────────────────────────────┐
│                  KUBERNETES CLUSTER                     │
│                                                         │
│  1. NODE NETWORK    (e.g., 10.0.0.0/16)                │
│     The VNet your AKS/EKS nodes live in                │
│                                                         │
│  2. POD NETWORK     (e.g., 10.244.0.0/16)              │
│     CIDR allocated to pods by the CNI                   │
│                                                         │
│  3. SERVICE NETWORK (e.g., 10.96.0.0/12)               │
│     Virtual IPs for Services (ClusterIPs)              │
│                                                         │
│  4. CLUSTER DNS     (e.g., 10.96.0.10)                 │
│     CoreDNS Service IP                                  │
│                                                         │
│  5. EXTERNAL        Public IPs, LoadBalancer endpoints │
└────────────────────────────────────────────────────────┘
```

***

### 🔌 CNI — The Container Network Interface

**CNI** is a **specification** (not a tool) that defines how container runtimes set up networking. When kubelet creates a pod, it calls the CNI plugin which:

1. Assigns the pod an **IP** from the cluster's pod CIDR
2. Creates a **veth pair** (virtual ethernet) — one end in the pod's network namespace, the other on the host
3. Configures **routing rules** so the pod can reach other pods, services, and the internet

```
┌──────── Node ─────────┐
│                        │
│  ┌──────┐  veth pair   │
│  │ Pod  │═════════════════ cbr0/cni0 ────── eth0 ──→ network
│  │ IP   │  (10.244.1.5)│   (bridge)         (10.0.0.5)
│  └──────┘              │
│                        │
└────────────────────────┘
```

***

### 🎭 Popular CNI Plugins Comparison

| CNI                   | Approach                      | Pros                                         | Cons                               | Best For                     |
| --------------------- | ----------------------------- | -------------------------------------------- | ---------------------------------- | ---------------------------- |
| **Flannel**           | Overlay (VXLAN)               | Simplest, just works                         | No NetworkPolicy in basic mode     | Learning, small clusters     |
| **Calico**            | Routing (BGP) + iptables/eBPF | Fast, mature, NetworkPolicy, eBPF dataplane  | Steeper learning curve             | Production, security-focused |
| **Cilium**            | eBPF-native                   | Best perf, observability (Hubble), L7 policy | Newer, requires modern kernel      | Modern cloud-native          |
| **Weave Net**         | Overlay + mesh routing        | Easy multi-cloud                             | Slower than Calico/Cilium          | Multi-cloud                  |
| **Azure CNI**         | Native VNet                   | Pods get real VNet IPs (no overlay)          | Limited pods per node by VNet size | AKS production               |
| **Azure CNI Overlay** | Overlay on AKS                | Scales to massive clusters                   | Pod IPs not VNet-routable          | AKS at scale                 |
| **AWS VPC CNI**       | Native VPC                    | Pods get real VPC IPs                        | ENI limits per node                | EKS                          |

***

### 🏗️ Overlay vs Underlay — The Key Decision

#### Overlay (Flannel, Calico VXLAN, Azure CNI Overlay)

* Pod IPs live on a **virtual network** layered on top of node network
* Traffic between nodes is **encapsulated** (VXLAN, IP-in-IP)
* ✅ Easy to scale; doesn't require infrastructure changes
* ❌ \~5–10% performance overhead from encapsulation

#### Underlay (Calico BGP, Azure CNI native, AWS VPC CNI)

* Pod IPs are **routable on the actual network**
* ✅ Best performance, no encapsulation overhead
* ✅ Direct visibility from outside (security tools, monitoring)
* ❌ Requires IP space planning; harder to scale

***

### 🔄 kube-proxy Deep Dive

`kube-proxy` runs as a **DaemonSet** on every node. Its job: implement the **Service abstraction** by programming the kernel's network rules.

When you create a Service:

1. K8s assigns a virtual IP (ClusterIP) from the service CIDR
2. EndpointSlices track healthy backing pod IPs
3. kube-proxy programs kernel rules so traffic to ClusterIP gets DNAT'd to a pod IP

### kube-proxy Modes

| Mode                          | How it Works             | Pros                                             | Cons                                         |
| ----------------------------- | ------------------------ | ------------------------------------------------ | -------------------------------------------- |
| **iptables** (default)        | Adds iptables NAT rules  | Stable, works everywhere                         | O(n) match — slow with thousands of services |
| **IPVS**                      | Linux IPVS kernel module | O(1) lookup, more LB algorithms (rr, lc, dh, sh) | Requires kernel modules                      |
| **nftables** (new, K8s 1.29+) | Replaces iptables        | Faster, modern                                   | Still maturing                               |
| **userspace** (deprecated)    | Proxy in userspace       | —                                                | Slow, gone                                   |

```powershell
# Check current mode
kubectl get configmap kube-proxy -n kube-system -o yaml | findstr mode
```

> 🎯 **Senior gotcha**: For clusters with **1000+ Services**, switch from iptables to IPVS. iptables matches rules **linearly**; with 5000 services, every packet walks through thousands of rules.

***

### 🌍 CoreDNS — Cluster DNS

**CoreDNS** is the default DNS server in K8s (replaced kube-dns since 1.13). Runs as a Deployment in `kube-system`.

Every pod gets `/etc/resolv.conf` pointing to CoreDNS's ClusterIP:

```
# Inside any pod:
$ cat /etc/resolv.conf
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
```

### DNS Records Created

For a Service `web-svc` in namespace `prod`:

| Query                                       | Resolves To              |
| ------------------------------------------- | ------------------------ |
| `web-svc` (same NS)                         | ClusterIP                |
| `web-svc.prod`                              | ClusterIP                |
| `web-svc.prod.svc.cluster.local` (FQDN)     | ClusterIP                |
| `_http._tcp.web-svc.prod.svc.cluster.local` | SRV record (port + host) |

For **headless Services** (Day 9 — StatefulSets):

| Query                                  | Resolves To             |
| -------------------------------------- | ----------------------- |
| `web-svc.prod.svc.cluster.local`       | All pod IPs (A records) |
| `web-0.web-svc.prod.svc.cluster.local` | Specific pod IP         |

***

### 🔍 The `ndots:5` Gotcha

```
options ndots:5
```

This means: **if the hostname has fewer than 5 dots, try search domains first**.

```
Query: api               → tries: api.default.svc.cluster.local → api.svc.cluster.local → api.cluster.local → api  (slow!)
Query: api.prod          → tries: api.prod.default.svc... → api.prod.svc... → api.prod.cluster... → api.prod
Query: api.example.com.  → tries: api.example.com. directly (trailing dot = FQDN, no search)
```

> 💡 **Production tip**: For external lookups, append a **trailing dot** or set `ndots:1` to avoid unnecessary lookups. Saves milliseconds and reduces CoreDNS load by 4x.

***

### 📡 The Pod-to-Pod DNS Resolution Flow

This is **THE interview question**. Memorize this flow:

```
Pod-A (10.244.1.5) needs to talk to "web-svc" in another namespace
│
▼
1. Pod-A's app calls: getaddrinfo("web-svc.prod")
│
▼
2. resolv.conf says: ask 10.96.0.10 (CoreDNS ClusterIP)
│
▼
3. DNS query packet: dst=10.96.0.10:53, src=10.244.1.5
│
▼
4. Packet hits node's kernel — kube-proxy iptables rules DNAT it
│  to one of CoreDNS pods (e.g., 10.244.0.20)
│
▼
5. CoreDNS pod receives query, checks if "web-svc.prod" is a 
│  K8s service → looks up EndpointSlices in etcd cache
│
▼
6. Returns A record: "web-svc.prod" → 10.96.45.123 (ClusterIP)
│
▼
7. Pod-A now opens TCP connection: dst=10.96.45.123:80
│
▼
8. Packet hits kernel — kube-proxy's iptables rules DNAT
│  10.96.45.123 → one of the backing pod IPs (e.g., 10.244.3.7)
│
▼
9. Packet routed across CNI overlay/underlay to target node
│
▼
10. Target node delivers packet to Pod-B at 10.244.3.7:80
```

***

### 🛡️ NetworkPolicies — Firewall in Kubernetes

By default, **all pods can talk to all pods** in K8s. This is dangerous! `NetworkPolicy` adds firewall rules:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-default
  namespace: prod
spec:
  podSelector: {}              # applies to ALL pods in namespace
  policyTypes:
  - Ingress
  - Egress
  # No rules = deny all
```

> 🎯 **Senior insight**: NetworkPolicies need a CNI that supports them. **Flannel doesn't** (out of the box). Calico, Cilium, Azure CNI, Weave all do.

We'll cover NetworkPolicies in depth on Day 13.

***

### 🌐 Container Network Anatomy

When a pod starts, this happens on the node:

```
┌─────────────── Node ────────────────────────┐
│                                              │
│  ┌────── Pod (network namespace) ────┐      │
│  │   ┌─────────┐                      │      │
│  │   │ Container│  eth0 (10.244.1.5)  │      │
│  │   └─────────┘   │                  │      │
│  │                 │ veth pair         │      │
│  │                 ├─ pause container  │      │
│  └─────────────────┼───────────────────┘     │
│                    │                          │
│              veth-abc123 (host side)         │
│                    │                          │
│              ┌─────▼─────┐                   │
│              │ cni0 bridge│  (or routes)    │
│              └─────┬──────┘                  │
│                    │                          │
│                  eth0 (10.0.0.5)              │
│                    │                          │
└────────────────────┼──────────────────────────┘
                     │
                     ▼
              ┌────────────┐
              │  Network   │
              └────────────┘
```

### The Pause Container (Hidden Gem)

Every pod has a **hidden "pause" container** (`registry.k8s.io/pause:3.x`). It:

* Holds the network namespace
* Lets app containers join the namespace
* Survives container restarts → pod IP doesn't change

```powershell
# See pause containers (on Minikube or any node access)
docker ps | findstr pause
```

***

### 🎯 Where K8s Networking Fits the OSI Model

```
┌──────────────────────────────────────────┐
│ L7 (App)      │ Ingress, HTTP, gRPC      │
├──────────────────────────────────────────┤
│ L4 (Transport)│ Service, kube-proxy,     │
│               │ TCP/UDP                  │
├──────────────────────────────────────────┤
│ L3 (Network)  │ CNI (Calico BGP, IP-IP)  │
├──────────────────────────────────────────┤
│ L2 (Data Link)│ CNI overlay (VXLAN),     │
│               │ veth pairs               │
└──────────────────────────────────────────┘
```

***

## 🛠️ Part 2: Hands-On — Trace Packets in Your Cluster

### Setup: Deploy Two Pods to Trace

```yaml
# 01-trace-setup.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: trace-demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: server
  namespace: trace-demo
spec:
  replicas: 3
  selector:
    matchLabels: {app: server}
  template:
    metadata:
      labels: {app: server}
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: server-svc
  namespace: trace-demo
spec:
  selector: {app: server}
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: v1
kind: Pod
metadata:
  name: client
  namespace: trace-demo
spec:
  containers:
  - name: tools
    image: nicolaka/netshoot:latest
    command: ['sleep', '3600']
```

```powershell
kubectl apply -f 01-trace-setup.yaml
kubectl get pods,svc -n trace-demo -o wide
```

***

### Lab 1: Inspect Cluster Network Configuration

```powershell
# Pod CIDR (where pods get IPs)
kubectl cluster-info dump | findstr -i "cluster-cidr"

# Service CIDR (where Services get virtual IPs)
kubectl cluster-info dump | findstr -i "service-cluster-ip-range"

# Per-node Pod CIDR allocation
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podCIDR}{"\n"}{end}'

# CoreDNS info
kubectl get svc -n kube-system kube-dns
kubectl get pods -n kube-system -l k8s-app=kube-dns

# kube-proxy info
kubectl get pods -n kube-system -l k8s-app=kube-proxy
kubectl get cm -n kube-system kube-proxy -o yaml | findstr mode
```

***

### Lab 2: Identify the CNI Plugin

```powershell
# Method 1: Check CNI config on a node (works on Minikube)
minikube ssh
sudo ls /etc/cni/net.d/
sudo cat /etc/cni/net.d/*.conflist

# Method 2: Look for CNI pods
kubectl get pods -n kube-system -o wide | findstr -i "calico flannel cilium azure"

# Method 3: Check for CNI-specific CRDs
kubectl api-resources | findstr -i "calico cilium"
```

***

### Lab 3: Trace DNS Resolution

```powershell
# Get into the netshoot client pod
kubectl exec -n trace-demo -it client -- bash

# Inside:
cat /etc/resolv.conf
# search trace-demo.svc.cluster.local svc.cluster.local cluster.local
# nameserver 10.96.0.10
# options ndots:5

# Query CoreDNS directly
nslookup server-svc
nslookup server-svc.trace-demo.svc.cluster.local
nslookup kubernetes.default

# See ALL queries to CoreDNS
nslookup -type=SRV _http._tcp.server-svc.trace-demo.svc.cluster.local

# Use dig for detailed view
dig server-svc.trace-demo.svc.cluster.local +trace

# Time how long DNS takes
time nslookup server-svc
time nslookup google.com

exit
```

***

### Lab 4: Trace a Real Packet

```powershell
kubectl exec -n trace-demo -it client -- bash

# Inside the client pod:
# Get our IP
ip addr show eth0
# 10.244.x.x

# Get the target service
nslookup server-svc
# server-svc.trace-demo.svc.cluster.local → 10.96.x.x (ClusterIP)

# Get backing pod IPs
nslookup server-svc      # returns ClusterIP
kubectl exec -n trace-demo client -- nslookup -type=A server-svc

# Watch a packet trip
# Open 2 terminals — in one, run tcpdump on host or use ksniff plugin
# In the other:
curl -v http://server-svc/

# See the path
traceroute server-svc       # may not show much in overlay networks
mtr server-svc

exit
```

***

### Lab 5: Inspect kube-proxy iptables Rules

```powershell
# SSH into a node (Minikube)
minikube ssh

# Look at iptables rules for the Service
sudo iptables-save | grep -A 5 "server-svc"
# You'll see:
# -A KUBE-SERVICES -d 10.96.x.x/32 ... -j KUBE-SVC-XXXXX
# -A KUBE-SVC-XXXXX ... -m statistic --mode random --probability 0.33 -j KUBE-SEP-AAA
# -A KUBE-SVC-XXXXX ... -m statistic --mode random --probability 0.50 -j KUBE-SEP-BBB
# -A KUBE-SVC-XXXXX ... -j KUBE-SEP-CCC

# Each KUBE-SEP-XXX is a DNAT to a pod IP
sudo iptables-save | grep "KUBE-SEP-AAA"
# -A KUBE-SEP-AAA -p tcp -m tcp -j DNAT --to-destination 10.244.x.x:80

# View the chain hierarchy
sudo iptables -t nat -L KUBE-SERVICES -n | head -20

exit
```

> 🎯 **Insight**: With 3 backend pods, you'll see 3 KUBE-SEP rules with random probabilities (0.33, 0.50, 1.0). That's how iptables does load balancing — sequential probability checks. This is why iptables mode doesn't scale to thousands of services.

***

### Lab 6: Compare iptables vs IPVS Modes

```powershell
# Check current mode
kubectl get cm -n kube-system kube-proxy -o yaml | findstr mode

# In IPVS mode, use ipvsadm instead
minikube ssh
sudo ipvsadm -L -n
# Lists virtual servers (ClusterIPs) and real servers (pod IPs) with weight, conn count
```

***

### Lab 7: CoreDNS Configuration & Performance

```powershell
# View CoreDNS config
kubectl get cm coredns -n kube-system -o yaml

# Look for the Corefile data section
# .:53 {
#     errors
#     health
#     ready
#     kubernetes cluster.local in-addr.arpa ip6.arpa {
#         pods insecure
#         fallthrough in-addr.arpa ip6.arpa
#     }
#     prometheus :9153
#     forward . /etc/resolv.conf
#     cache 30
#     loop
#     reload
#     loadbalance
# }

# Watch CoreDNS metrics
kubectl get svc -n kube-system kube-dns
kubectl port-forward -n kube-system svc/kube-dns 9153:9153
# Browse to http://localhost:9153/metrics

# DNS query latency from a pod
kubectl exec -n trace-demo client -- bash -c "for i in 1 2 3; do time nslookup server-svc; done"
```

***

### Lab 8: NodeLocal DNSCache (Production Optimization)

NodeLocal DNSCache runs as a DaemonSet — pods talk to local cache instead of CoreDNS. Reduces DNS latency and load by \~80%.

```powershell
# Check if installed
kubectl get pods -n kube-system | findstr node-local-dns

# If on AKS, can be enabled with:
# az aks enable-addons --addons azure-keyvault-secrets-provider,confcom -g <rg> -n <aks>
# OR via cluster creation flag

# View resolv.conf on a pod with NodeLocal DNSCache enabled
kubectl exec -n trace-demo client -- cat /etc/resolv.conf
# nameserver 169.254.20.10  ← node-local cache (instead of CoreDNS ClusterIP)
```

***

### Lab 9: Trace a Cross-Node Packet (Advanced)

```powershell
# Find two pods on different nodes
kubectl get pods -n trace-demo -o wide

# Get the client pod's node and server pod's node
$clientNode = kubectl get pod client -n trace-demo -o jsonpath='{.spec.nodeName}'
$serverPod = kubectl get pods -n trace-demo -l app=server -o jsonpath='{.items[0].metadata.name}'
$serverNode = kubectl get pod $serverPod -n trace-demo -o jsonpath='{.spec.nodeName}'

Write-Host "Client on $clientNode → Server on $serverNode"

# From client, hit the specific pod IP
$serverPodIP = kubectl get pod $serverPod -n trace-demo -o jsonpath='{.status.podIP}'
kubectl exec -n trace-demo client -- curl -s http://$serverPodIP

# On the server node (Minikube)
minikube ssh
sudo tcpdump -i any -nn host $serverPodIP and port 80 -c 5
# (re-run the curl from the client pod)
exit
```

***

### Lab 10: Network Performance Test

```powershell
# Deploy iperf3 server
$iperfYaml = @"
apiVersion: v1
kind: Pod
metadata:
  name: iperf-server
  namespace: trace-demo
  labels: {app: iperf-server}
spec:
  containers:
  - name: iperf
    image: networkstatic/iperf3
    args: ['-s']
    ports:
    - containerPort: 5201
---
apiVersion: v1
kind: Service
metadata:
  name: iperf-svc
  namespace: trace-demo
spec:
  selector: {app: iperf-server}
  ports:
  - port: 5201
    targetPort: 5201
"@
$iperfYaml | kubectl apply -f -

# Run iperf3 client
kubectl run -n trace-demo iperf-client --rm -it --image=networkstatic/iperf3 -- -c iperf-svc -t 10

# Compare: same node vs different node (use nodeSelector to force placement)
```

***

## 💻 Part 3: Commands Cheat Sheet

### Inspect Networking

```powershell
# Cluster CIDRs
kubectl cluster-info dump | findstr -i "cidr"

# Per-node Pod CIDR
kubectl get nodes -o custom-columns=NAME:.metadata.name,POD_CIDR:.spec.podCIDR

# Pod IPs
kubectl get pods -A -o wide

# Service IPs
kubectl get svc -A -o wide

# Endpoints
kubectl get endpoints -A
kubectl get endpointslices -A

# CNI info
kubectl get pods -n kube-system | findstr -i "calico cilium flannel azure"
kubectl get crds | findstr -i "calico cilium"

# CoreDNS
kubectl get svc -n kube-system kube-dns
kubectl get cm coredns -n kube-system
kubectl logs -n kube-system -l k8s-app=kube-dns

# kube-proxy
kubectl get ds -n kube-system kube-proxy
kubectl get cm kube-proxy -n kube-system -o yaml | findstr mode
kubectl logs -n kube-system -l k8s-app=kube-proxy
```

### Debug from Inside a Pod

```powershell
# Quick netshoot pod
kubectl run debug --image=nicolaka/netshoot -it --rm --restart=Never -- bash

# Inside:
# ─── DNS ───
nslookup <service>
dig <service>.<namespace>.svc.cluster.local
host <service>

# ─── Connectivity ───
curl -v http://<service>
nc -zv <service> 80
telnet <service> 80
ping <pod-ip>           # may not work — CNI dependent

# ─── Network namespace ───
ip addr
ip route
ip neigh
ss -tnp                 # listening sockets
ss -ant                 # active connections

# ─── Trace ───
traceroute <target>
mtr <target>
tcpdump -i any -nn
```

### Debug from a Node

```powershell
# Iptables (kube-proxy iptables mode)
iptables-save | grep KUBE-SERVICES
iptables -t nat -L KUBE-SERVICES -n
iptables -t nat -L KUBE-SVC-XXXXX -n

# IPVS (kube-proxy IPVS mode)
ipvsadm -L -n
ipvsadm -L -n -t <ClusterIP>:80

# CNI
ls /etc/cni/net.d/
cat /etc/cni/net.d/*.conflist

# Interfaces
ip addr | grep -E "cni|veth|calico|flannel"
```

***

## 🎤 Part 4: Interview Prep

### ⭐ Star Question: *"Walk through how a Pod resolves another Pod's DNS name."*

**Senior DevOps model answer:**

> "Let me walk through the complete journey of a DNS query — say `Pod-A` (in namespace `default`) calling `web-svc.prod` — touching every component along the way.
>
> **Step 1 — The application calls a DNS function.** Code like `connect('web-svc.prod', 80)` triggers the libc resolver, which reads `/etc/resolv.conf`. In every K8s pod, that file is injected by kubelet with three things: a `nameserver` pointing to CoreDNS's ClusterIP (typically `10.96.0.10`), a `search` list of suffixes (`default.svc.cluster.local`, `svc.cluster.local`, `cluster.local`), and an `options ndots:5` setting.
>
> **Step 2 — `ndots` logic kicks in.** Because `web-svc.prod` has only 1 dot (less than 5), the resolver tries the **search domains first**. So it first attempts `web-svc.prod.default.svc.cluster.local` (NXDOMAIN — wrong namespace), then `web-svc.prod.svc.cluster.local` (HIT!). This is why I often advise developers to use FQDNs with a trailing dot for external lookups — it skips this expensive search.
>
> **Step 3 — The DNS query packet leaves the pod.** Source: `Pod-A`'s IP, destination: `10.96.0.10:53` (CoreDNS ClusterIP). The packet enters the node's kernel network stack.
>
> **Step 4 — kube-proxy's iptables/IPVS rules DNAT the destination.** `10.96.0.10` isn't a real IP — it's a virtual IP backed by EndpointSlices. The kernel rewrites the destination to one of the actual CoreDNS pod IPs, say `10.244.0.20`. Load balancing happens here — random in iptables mode, weighted in IPVS mode.
>
> **Step 5 — Packet routes via CNI to the CoreDNS pod.** Depending on the CNI plugin, this might be a VXLAN-encapsulated overlay packet (Flannel), a directly routed packet (Calico BGP), or a native VNet packet (Azure CNI). On the destination node, the CNI delivers the packet to CoreDNS through its veth pair.
>
> **Step 6 — CoreDNS resolves the query.** It checks if `web-svc.prod.svc.cluster.local` matches any K8s resource via the `kubernetes` plugin. It queries its in-memory cache (synced from etcd via the API server). Finding the Service, it returns an A record with the ClusterIP, say `10.96.45.123`. Importantly, this is a **virtual IP**, not a real one.
>
> **Step 7 — Pod-A now opens a TCP connection** to `10.96.45.123:80`. Same flow as before: packet enters the kernel, hits kube-proxy's iptables/IPVS rules. The rules see `10.96.45.123` is the ClusterIP for `web-svc`, look up the EndpointSlice, pick one of the backing pod IPs (e.g., `10.244.3.7`), and DNAT the destination.
>
> **Step 8 — Cross-node delivery via CNI.** The packet now has destination `10.244.3.7`. The CNI plugin's routing tables know which node owns that CIDR range, and the packet is forwarded — encapsulated or directly routed — to the right node.
>
> **Step 9 — Delivery to the target pod.** The destination node's CNI delivers the packet through the veth pair into the pod's network namespace. The app's listener accepts the connection.
>
> **A few production gotchas I always mention**:
>
> * **DNS is a hidden performance bottleneck**. With ndots:5 and 4 search domains, every external lookup does 5 queries. NodeLocal DNSCache solves this by running a per-node DNS cache as a DaemonSet, reducing CoreDNS load by 80% and DNS latency to sub-millisecond.
> * **`headless Services` skip steps 4 and 7** — DNS returns pod IPs directly instead of a ClusterIP. Used heavily by StatefulSets for stable per-pod DNS.
> * **`ExternalName` Services** are pure DNS CNAMEs — no proxying, no load balancing, just CoreDNS returning the external name.
> * **The CNI plugin matters massively**. Calico in BGP mode performs better than Flannel VXLAN because there's no encapsulation. Cilium with eBPF can bypass kube-proxy entirely, replacing iptables with eBPF programs for 10x performance.
> * **NetworkPolicies are enforced at the CNI layer** — Flannel can't enforce them by itself; you need Calico, Cilium, or Azure CNI.
>
> That's the full path — DNS resolution + Service load balancing + CNI routing, all working together to make `web-svc.prod` reachable from any pod in the cluster."

***

### 🔥 Common Follow-Ups

| #  | Question                                                  | Quick Answer                                                                                                                                                               |
| -- | --------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1  | **What are the 4 K8s networking requirements?**           | Pod-to-Pod (same node), Pod-to-Pod (across nodes), Pod-to-Service, External-to-Service. All without NAT inside the cluster.                                                |
| 2  | **What is CNI?**                                          | Container Network Interface — a spec (not a tool) for how runtimes set up pod networking. Plugins: Calico, Flannel, Cilium, Azure CNI.                                     |
| 3  | **Difference between overlay and underlay networks?**     | Overlay encapsulates pod traffic (VXLAN, IP-in-IP) — easy, \~5–10% overhead. Underlay routes pod IPs natively (BGP, Azure CNI) — faster, requires IP planning.             |
| 4  | **What does kube-proxy actually do?**                     | Programs kernel networking rules (iptables/IPVS/nftables) to implement Service virtual IPs. Doesn't proxy traffic itself — kernel does.                                    |
| 5  | **iptables vs IPVS — when to use IPVS?**                  | IPVS: better perf with thousands of services (O(1) vs O(n)), more LB algorithms. Switch when you have 1000+ services.                                                      |
| 6  | **What's the pause container?**                           | Tiny container in every pod that holds the network namespace. Lets pod IPs survive container restarts.                                                                     |
| 7  | **Why does every pod have its own IP?**                   | K8s networking model mandates flat networking — no NAT, no port mapping headaches. Makes microservice communication identical to traditional VMs.                          |
| 8  | **What's CoreDNS?**                                       | Default cluster DNS (replaced kube-dns in 1.13). Resolves Service names, headless pod names, external lookups via forward plugin.                                          |
| 9  | **What is ndots:5?**                                      | resolv.conf option — hostnames with fewer than 5 dots try search domains first. Causes extra DNS queries for external lookups.                                             |
| 10 | **What's NodeLocal DNSCache?**                            | DaemonSet that caches DNS per node — pods talk to `169.254.20.10`. Reduces CoreDNS load and DNS latency by \~80%.                                                          |
| 11 | **How does a headless Service differ from regular?**      | `clusterIP: None`. DNS returns all pod IPs directly (A records), not a load-balanced virtual IP. Used by StatefulSets.                                                     |
| 12 | **What happens if CoreDNS crashes?**                      | New DNS lookups fail; pods using cached entries keep working briefly. Existing TCP connections survive (they use IPs, not names). Critical reason for HA CoreDNS replicas. |
| 13 | **Can pods on different nodes ping each other directly?** | Yes — that's a K8s networking requirement. The CNI plugin ensures pod IPs are routable cluster-wide.                                                                       |
| 14 | **What's a NetworkPolicy?**                               | Firewall rules for pod traffic. Requires a CNI that supports them (Calico, Cilium, Azure CNI — NOT basic Flannel).                                                         |
| 15 | **How does Cilium with eBPF differ from kube-proxy?**     | Cilium can fully replace kube-proxy. Instead of iptables, it loads eBPF programs into the kernel for L3-L7 filtering and load balancing — 10x faster at scale.             |
| 16 | **What is the service CIDR?**                             | The IP range used by ClusterIPs (default `10.96.0.0/12`). Disjoint from pod CIDR. Configured at cluster creation.                                                          |
| 17 | **How do pods reach the internet?**                       | Outbound traffic from pod → SNAT'd at the node (using node's IP) → routed to internet. In cloud, often via NAT Gateway.                                                    |
| 18 | **What's MTU and why does it matter in K8s?**             | Maximum Transmission Unit. Overlay networks reduce effective MTU (VXLAN headers eat \~50 bytes). Miscounted MTU causes silent packet drops or fragmentation.               |

***

### 💡 Senior Scenario Questions

**Scenario 1: "Your pod can't reach a Service. Walk through how you debug."**

> Methodical approach:
>
> 1. **Verify Service exists**: `kubectl get svc -n <ns>`. Right name, right namespace?
> 2. **Check Endpoints**: `kubectl get endpoints <svc-name>`. If empty → selector doesn't match any pod, OR pods are not Ready.
> 3. **Check pod labels**: `kubectl get pods -l <selector>`. Mismatched labels are the #1 cause.
> 4. **DNS test from inside the cluster**: `kubectl run debug --image=nicolaka/netshoot -it --rm -- nslookup <svc>`. NXDOMAIN? FQDN vs short name issue.
> 5. **CoreDNS health**: `kubectl get pods -n kube-system -l k8s-app=kube-dns` — all Ready? `kubectl logs -n kube-system -l k8s-app=kube-dns` for errors.
> 6. **Direct connection test**: `curl http://<ClusterIP>:<port>` — bypasses DNS.
> 7. **Pod IP direct**: `curl http://<pod-ip>:<port>` — bypasses Service entirely. If this works but ClusterIP doesn't, kube-proxy issue.
> 8. **kube-proxy logs**: `kubectl logs -n kube-system -l k8s-app=kube-proxy`.
> 9. **NetworkPolicy?** `kubectl get netpol -A` — accidentally blocking traffic?
> 10. **MTU mismatch** — check `ip link show` on nodes. If overlay MTU > underlay MTU, packets silently dropped.

***

**Scenario 2: "DNS lookups in your AKS cluster are taking 2–5 seconds. How do you fix?"**

> Likely causes & fixes:
>
> 1. **ndots:5 amplification** — every external lookup is 5 queries (4 search domains + actual). Solution: use FQDN with trailing dot, or set `dnsConfig` in pods to override ndots.
> 2. **CoreDNS overloaded** — `kubectl top pods -n kube-system | grep dns`. Scale up replicas, increase CPU limits, enable autoscaling.
> 3. **No NodeLocal DNSCache** — install it. Pods then query `169.254.20.10` (per-node cache) instead of CoreDNS Service IP. Cuts DNS load 80%.
> 4. **CoreDNS upstream slow** — check `forward . /etc/resolv.conf` in Corefile. Replace with explicit `forward . 168.63.129.16` (Azure DNS).
> 5. **Negative caching too aggressive** — tune `cache 30` in Corefile (TTL for negative responses).
> 6. **DNS spoofing/loop** — check for `loop` plugin detecting loops.
> 7. **conntrack table full** — `dmesg | grep conntrack`. Increase `net.netfilter.nf_conntrack_max`.
> 8. **Monitor**: Prometheus metrics `coredns_dns_request_duration_seconds`, `coredns_cache_hits_total`.

***

**Scenario 3: "Pods in `team-a` namespace can't reach pods in `team-b`. We just installed Calico. What did we miss?"**

> Likely cause: **NetworkPolicy applied somewhere**. Calico installs the enforcement engine — if NetworkPolicies exist, they're now enforced.
>
> Debug:
>
> 1. `kubectl get netpol -A` — list all policies.
> 2. Look for `deny-all` defaults in `team-a` or `team-b`.
> 3. If `team-b` has a default-deny on Ingress, `team-a` pods are blocked.
> 4. Add explicit allow rule:
>    ```yaml
>    apiVersion: networking.k8s.io/v1
>    kind: NetworkPolicy
>    metadata:
>      name: allow-team-a
>      namespace: team-b
>    spec:
>      podSelector: {}
>      ingress:
>      - from:
>        - namespaceSelector:
>            matchLabels: {team: a}
>    ```
> 5. Test with `kubectl exec` + `curl` from team-a pod.
> 6. Use Calico's `calicoctl get policies` for detailed view.

***

**Scenario 4: "Your AKS cluster's pods can reach external internet but not Azure VNet peer."**

> Investigation:
>
> 1. **Azure CNI vs Kubenet/Overlay** — if using Kubenet, pod IPs are NAT'd to the node's IP for outbound. Peer VNet sees node IPs, not pod IPs. Some peer firewalls reject this.
> 2. **Effective routes**: in AKS Kubenet, an Azure route table is auto-managed. Confirm it's attached to the VNet.
> 3. **UDR (User-Defined Route)** — does a custom route table override system routes?
> 4. **NSG rules** on the AKS subnet or peer VNet subnet. Check both inbound and outbound rules.
> 5. **VNet peering settings** — is `Forwarded traffic` allowed? Is `Use remote gateways` set correctly?
> 6. **Firewall in between** — Azure Firewall or NVA in transit VNet?
> 7. Fix: switch to Azure CNI for direct pod IPs in VNet; or whitelist node CIDR in firewall.

***

**Scenario 5: "Your cluster has 5000 Services and pod-to-Service latency is high."**

> Root cause: **iptables mode kube-proxy** doesn't scale. Each Service adds \~5 rules; 5000 services = 25,000+ rules walked linearly per packet.
>
> Solutions:
>
> 1. **Switch to IPVS mode**:
>    ```powershell
>    kubectl edit cm kube-proxy -n kube-system
>    # mode: "ipvs"
>    kubectl rollout restart ds kube-proxy -n kube-system
>    ```
>    IPVS uses hash tables — O(1) lookups.
> 2. **Switch to nftables mode** (K8s 1.29+, GA in newer releases).
> 3. **Move to Cilium** with kube-proxy replacement — uses eBPF, even faster.
> 4. **Reduce service count** — do you really need 5000? Maybe consolidate.
> 5. **Tune conntrack**: `sysctl -w net.netfilter.nf_conntrack_max=2097152` on nodes.

***

## 📊 CNI Decision Matrix

| Need                              | Pick                                   |
| --------------------------------- | -------------------------------------- |
| Learning K8s, simplest setup      | **Flannel**                            |
| Production, NetworkPolicy, mature | **Calico**                             |
| Best perf, observability, eBPF    | **Cilium**                             |
| AKS with VNet integration         | **Azure CNI (native)**                 |
| AKS at massive scale              | **Azure CNI Overlay**                  |
| EKS, AWS-native                   | **AWS VPC CNI**                        |
| GKE                               | **GKE-native (Dataplane V2 = Cilium)** |
| Multi-cloud / hybrid              | **Calico** or **Cilium**               |
| On-prem, BGP-friendly             | **Calico BGP**                         |

***

