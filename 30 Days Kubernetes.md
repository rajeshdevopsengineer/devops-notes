# 🚀 Day 1: Kubernetes Introduction & Architecture

Welcome to Day 1 of your 30-day Kubernetes journey, Rajesh! As a Senior DevOps Engineer, you likely have exposure to container orchestration already, so we'll go deep into the *why* and *how* — not just the *what*. Let's build a rock-solid foundation today.

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

## ✅ Day 1 Checklist

* [ ] Understood control plane vs worker node components
* [ ] Installed `kubectl`, `minikube`, and `kind` on Windows
* [ ] Started a local cluster (Minikube or Kind)
* [ ] Ran `kubectl version`, `cluster-info`, `get nodes` successfully
* [ ] Practiced the interview answer out loud
* [ ] Deployed and deleted a test `nginx` pod

***


