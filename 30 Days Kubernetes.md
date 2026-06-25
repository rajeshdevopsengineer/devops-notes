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

# 🏢 Day 6: Namespaces, Labels & Selectors

Welcome to Day 6, Rajesh! So far you've built workloads and exposed them — but in real enterprises, you're not running **one app on one cluster**. You're running **dozens of teams, hundreds of microservices, multiple environments** on shared infrastructure. Today you'll master the **organizational backbone** of Kubernetes: Namespaces (isolation), Labels (grouping), and Selectors (querying). These are the tools that turn chaos into a multi-tenant, governable platform.

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
    owner: rajesh
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
    contact: "rajesh@example.com"       # ← descriptive
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
kubectl label pod <pod-name> -n payments-prod owner=rajesh

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
  contact="rajesh@example.com" `
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
        # Example: rajesh/petstore-api:1.0.0
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








