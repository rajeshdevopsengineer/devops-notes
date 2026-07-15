

### **1. What is Kubernetes and why is it used?**

**Kubernetes** (often abbreviated as *K8s*) is an **open-source container orchestration platform** designed to automate the deployment, scaling, management, and operation of containerized applications.

It is used because it:

* Manages containerized applications efficiently
* Automatically handles scaling and load balancing
* Provides high availability and self-healing
* Simplifies application deployment in cloud and hybrid environments

In essence, Kubernetes allows developers and organizations to run distributed systems reliably at scale.

---

### **2. What problem does Kubernetes solve?**

Kubernetes solves the problem of **managing containers in production environments**. While containers (e.g., Docker) package applications, running many containers across multiple machines introduces challenges such as:

* Service discovery
* Load balancing
* Scaling applications up or down
* Handling failures and restarts
* Rolling updates without downtime

Kubernetes abstracts these complexities and provides a **declarative, automated system** to manage them.

---

### **3. Explain the architecture of Kubernetes.**

Kubernetes follows a **master–worker (control plane–node) architecture** consisting of:

1. **Control Plane (Master Components)** – Manages the cluster.
2. **Worker Nodes** – Run the actual application workloads.

Key architectural components include:

* API Server
* Scheduler
* Controller Manager
* etcd (distributed key-value store)
* Kubelet and container runtime on worker nodes

This architecture ensures scalability, fault tolerance, and centralized control.

---

### **4. What is a Kubernetes cluster?**

A **Kubernetes cluster** is a set of machines (physical or virtual) that run containerized applications managed by Kubernetes.

It consists of:

* **One or more control plane (master) nodes**
* **Multiple worker nodes**

The cluster provides a unified environment where applications can be deployed, scaled, and maintained consistently.

---

### **5. What are nodes in Kubernetes?**

A **node** is a worker machine in a Kubernetes cluster where containers are executed.

Each node:

* Runs application pods
* Is managed by the Kubernetes control plane
* Contains services required to run containers

Nodes can be physical servers or virtual machines.

---

### **6. What are master and worker nodes?**

* **Master (Control Plane) Nodes**

  * Manage the cluster
  * Make global decisions (scheduling, scaling)
  * Expose the Kubernetes API

* **Worker Nodes**

  * Run application workloads
  * Host pods and containers
  * Communicate with the control plane

This separation ensures **centralized management** and **distributed execution**.

---

### **7. What are pods in Kubernetes?**

A **Pod** is the **smallest deployable unit** in Kubernetes.

Characteristics:

* Contains one or more containers
* Containers in a pod share:

  * Network (IP address and ports)
  * Storage volumes
* Represents a single application instance

Pods are ephemeral and are usually managed by higher-level objects like Deployments.

---

### **8. What is a ReplicaSet?**

A **ReplicaSet** ensures that a specified number of **identical pod replicas** are running at any given time.

Functions:

* Maintains desired pod count
* Automatically creates or deletes pods
* Improves availability and fault tolerance

ReplicaSets are rarely used directly and are typically managed by Deployments.

---

### **9. What is a Deployment?**

A **Deployment** is a higher-level Kubernetes object that manages ReplicaSets and Pods.

It provides:

* Declarative updates to applications
* Rolling updates and rollbacks
* Version control of application states

Deployments are the **recommended way** to manage stateless applications in Kubernetes.

---

### **10. What is a Service in Kubernetes?**

A **Service** is an abstraction that exposes a set of pods as a **stable network endpoint**.

Key purposes:

* Enables communication between pods
* Provides load balancing
* Handles dynamic pod IP changes

Types of Services include:

* **ClusterIP** (internal access)
* **NodePort** (external access via node)
* **LoadBalancer** (cloud-based external access)

Services ensure reliable connectivity in a dynamic container environment.
Below are **concise, exam-oriented explanations** of Kubernetes core components and concepts (Questions **11–20**), written in a clear academic style.

---

### **11. What is the kube-apiserver?**

The **kube-apiserver** is the **central management component** of the Kubernetes control plane.
It exposes the Kubernetes API and acts as the **front end** for all administrative operations.

Key functions:

* Receives REST requests from users and components
* Validates and processes API objects
* Communicates with `etcd` to store and retrieve cluster state
* Serves as the only component that directly accesses `etcd`

---

### **12. What is etcd used for?**

**etcd** is a **distributed, consistent key-value store** used by Kubernetes to store **all cluster data**.

It stores:

* Cluster configuration
* Node and pod information
* Secrets and ConfigMaps
* Desired and current state of the cluster

etcd ensures **high availability, consistency, and fault tolerance** of critical Kubernetes data.

---

### **13. What is kube-scheduler?**

The **kube-scheduler** is responsible for **assigning pods to nodes**.

Its tasks include:

* Monitoring newly created pods with no assigned node
* Evaluating nodes based on resource availability and constraints
* Selecting the most suitable node for pod placement

Scheduling decisions consider CPU, memory, affinity rules, and policies.

---

### **14. What is kube-controller-manager?**

The **kube-controller-manager** runs various **controller processes** that regulate the cluster’s state.

Controllers include:

* Node Controller
* ReplicaSet Controller
* Deployment Controller
* Endpoint Controller

It continuously compares the **desired state** with the **actual state** and takes corrective actions to maintain consistency.

---

### **15. What is kubelet?**

The **kubelet** is an agent that runs on **each worker node**.

Responsibilities:

* Communicates with the kube-apiserver
* Ensures containers described in pod specifications are running
* Monitors pod and container health
* Reports node status back to the control plane

It directly interacts with the container runtime.

---

### **16. What is kube-proxy?**

**kube-proxy** is a network component that manages **network rules** on each node.

Functions:

* Enables communication between pods and services
* Implements service load balancing
* Maintains network forwarding rules using iptables or IPVS

It ensures that traffic reaches the correct pod even as pod IPs change.

---

### **17. What is Container Runtime Interface (CRI)?**

The **Container Runtime Interface (CRI)** is a standardized API that allows Kubernetes to interact with different container runtimes.

Purpose:

* Decouples Kubernetes from specific container runtimes
* Enables pluggable runtime implementations
* Standardizes container lifecycle management

CRI allows Kubernetes to remain runtime-agnostic.

---

### **18. What runtimes can Kubernetes use besides Docker?**

Besides Docker, Kubernetes supports several **CRI-compatible runtimes**, including:

* **containerd**
* **CRI-O**
* **rkt** (deprecated)
* **gVisor** (sandboxed runtime)

Modern Kubernetes versions primarily use **containerd** or **CRI-O** for improved performance and security.

---

### **19. What are namespaces in Kubernetes?**

**Namespaces** are a Kubernetes feature used to **logically divide cluster resources**.

They provide:

* Resource isolation
* Name scoping
* Multi-tenancy support

Common namespaces include:

* `default`
* `kube-system`
* `kube-public`

---

### **20. Why do we use namespaces?**

Namespaces are used to:

* Separate environments (dev, test, prod)
* Isolate teams or applications
* Apply resource quotas and access control
* Improve organization and security

They are especially useful in **large or shared Kubernetes clusters**.

---


Below are **clear, exam-ready explanations** for Kubernetes Questions **21–30**, written in a concise academic style with practical command examples where relevant.

---

### **21. What are multi-container pods?**

**Multi-container pods** are Kubernetes pods that run **more than one container** within the same pod.

Key characteristics:

* Containers share the **same network namespace** (IP and ports)
* Containers share **storage volumes**
* Containers are scheduled and scaled **together**

They are used when containers are **tightly coupled** and must work collaboratively.

---

### **22. What are sidecar containers?**

A **sidecar container** is a helper container that runs alongside the main application container in the same pod.

Common use cases:

* Log collection
* Monitoring and metrics
* Proxying or service mesh integration
* Configuration synchronization

The sidecar pattern enhances functionality **without modifying the main application**.

---

### **23. How do you create a pod?**

A pod can be created using a **YAML manifest** and the `kubectl apply` command.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: nginx
    image: nginx
```

Command:

```bash
kubectl apply -f pod.yaml
```

Pods can also be created imperatively, but declarative YAML is preferred.

---

### **24. What is the difference between Pod and Deployment?**

| Feature      | Pod               | Deployment      |
| ------------ | ----------------- | --------------- |
| Purpose      | Runs containers   | Manages pods    |
| Scalability  | No                | Yes             |
| Self-healing | No                | Yes             |
| Updates      | Manual            | Rolling updates |
| Usage        | Rarely used alone | Recommended     |

A **Deployment manages pods indirectly** using ReplicaSets.

---

### **25. How do you scale a Deployment?**

A Deployment can be scaled **manually** using `kubectl scale`.

Example:

```bash
kubectl scale deployment my-deployment --replicas=5
```

Alternatively, scaling can be done by editing the Deployment YAML or using **Horizontal Pod Autoscaler (HPA)**.

---

### **26. How do you perform a rolling update?**

Rolling updates occur automatically when you update a Deployment.

Example:

```bash
kubectl set image deployment my-deployment nginx=nginx:1.25
```

Kubernetes:

* Gradually replaces old pods
* Ensures minimal or zero downtime
* Maintains availability during updates

---

### **27. How do you roll back a deployment?**

You can roll back to a previous revision using:

```bash
kubectl rollout undo deployment my-deployment
```

To view revision history:

```bash
kubectl rollout history deployment my-deployment
```

Rollback restores the application to a **previous stable state**.

---

### **28. How do you check pod status?**

To check pod status:

```bash
kubectl get pods
```

For detailed information:

```bash
kubectl describe pod my-pod
```

To view logs:

```bash
kubectl logs my-pod
```

Pod states include `Pending`, `Running`, `Succeeded`, `Failed`, and `CrashLoopBackOff`.

---

### **29. What’s the difference between desired, current, and available replicas?**

* **Desired replicas**: Number of pods requested by the user
* **Current replicas**: Number of pods currently created
* **Available replicas**: Number of pods ready to serve traffic

These values help monitor **deployment health and availability**.

---

### **30. How do you delete a pod manually?**

A pod can be deleted using:

```bash
kubectl delete pod my-pod
```

If the pod is managed by a Deployment or ReplicaSet, Kubernetes will **automatically recreate it** to maintain the desired state.

---


Below are **clear, exam-ready explanations** for Kubernetes networking and service discovery concepts (**Questions 31–40**), written in a concise academic style.

---

### **31. What is a ClusterIP service?**

A **ClusterIP service** is the **default Kubernetes Service type** that exposes an application **internally within the cluster**.

Key points:

* Accessible only inside the cluster
* Provides a stable virtual IP (VIP)
* Used for internal microservice communication

It is ideal for backend services not meant to be exposed externally.

---

### **32. What is a NodePort service?**

A **NodePort service** exposes an application by opening a **static port** on each worker node.

Characteristics:

* Accessible using `<NodeIP>:<NodePort>`
* NodePort range is typically `30000–32767`
* Suitable for development or simple external access

NodePort builds on ClusterIP and adds external reachability.

---

### **33. What is a LoadBalancer service?**

A **LoadBalancer service** exposes applications to the internet using an **external load balancer**, usually provided by a cloud provider.

Features:

* Automatically provisions a cloud load balancer
* Routes external traffic to cluster nodes
* Common in production environments

It combines NodePort and ClusterIP with managed external access.

---

### **34. What is an ExternalName service?**

An **ExternalName service** maps a Kubernetes service to an **external DNS name**.

Key aspects:

* No selector or endpoints
* Returns a CNAME record
* Used to access external services as if they were internal

Example use case: connecting to a managed database outside the cluster.

---

### **35. What is a headless service?**

A **headless service** is a service created with `clusterIP: None`.

Purpose:

* Does not provide load balancing
* Returns individual pod IPs via DNS
* Used for stateful applications (e.g., databases)

It allows clients to directly discover and connect to specific pods.

---

### **36. What’s the default DNS name for a service?**

The default DNS format for a Kubernetes service is:

```
<service-name>.<namespace>.svc.cluster.local
```

Example:

```
myservice.default.svc.cluster.local
```

This DNS name is automatically created by Kubernetes DNS.

---

### **37. How do pods communicate internally?**

Pods communicate internally through:

* **Pod IPs** (flat, routable network)
* **Services** (stable access and load balancing)
* **DNS-based service discovery**

Kubernetes networking ensures that **all pods can communicate without NAT**, regardless of the node they run on.

---

### **38. What is kube-dns / CoreDNS?**

**kube-dns / CoreDNS** is the **DNS service** in Kubernetes clusters.

Functions:

* Resolves service names to IP addresses
* Enables service discovery
* Maintains DNS records for pods and services

**CoreDNS** has replaced kube-dns in modern Kubernetes versions due to better performance and extensibility.

---

### **39. What happens when a pod restarts — does IP change?**

Yes, **a pod’s IP address usually changes** when it restarts or is recreated.

Important implication:

* Pods are ephemeral
* Applications should not rely on pod IPs
* Services provide stable endpoints instead

This design encourages loose coupling and resilience.

---

### **40. How does Service Discovery work in Kubernetes?**

Service discovery in Kubernetes works through:

1. **Services** that group pods using labels
2. **DNS records** automatically created for services
3. **kube-proxy** routing traffic to healthy pods

Applications discover services using **DNS names**, not IP addresses, enabling scalability and fault tolerance.

---

Below are **concise, exam-ready explanations** for Kubernetes configuration, security, and namespace management (**Questions 41–50**), written in a clear academic style with practical examples.

---

### **41. What are ConfigMaps?**

**ConfigMaps** are Kubernetes objects used to store **non-confidential configuration data** as key–value pairs.

They are used to:

* Externalize application configuration
* Avoid hardcoding configuration in container images
* Improve portability and maintainability

ConfigMaps can store environment variables, command-line arguments, or configuration files.

---

### **42. What are Secrets?**

**Secrets** are Kubernetes objects used to store **sensitive information**, such as:

* Passwords
* API tokens
* TLS certificates

Secrets are base64-encoded and can be mounted into pods or injected as environment variables, with access controlled by RBAC.

---

### **43. What is the difference between ConfigMap and Secret?**

| Feature   | ConfigMap     | Secret            |
| --------- | ------------- | ----------------- |
| Data type | Non-sensitive | Sensitive         |
| Encoding  | Plain text    | Base64-encoded    |
| Security  | Minimal       | Access-controlled |
| Examples  | URLs, configs | Passwords, keys   |

**Secrets** provide better security handling, while **ConfigMaps** focus on configuration flexibility.

---

### **44. How do you mount ConfigMaps into pods?**

A ConfigMap can be mounted as a **volume** inside a pod.

Example:

```yaml
volumes:
- name: config-volume
  configMap:
    name: app-config
containers:
- name: app
  image: nginx
  volumeMounts:
  - name: config-volume
    mountPath: /etc/config
```

Each key in the ConfigMap becomes a file in the mounted directory.

---

### **45. How do you inject environment variables into containers?**

Environment variables can be injected using:

* ConfigMaps
* Secrets
* Direct values

Example using ConfigMap:

```yaml
env:
- name: APP_MODE
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: mode
```

This allows dynamic configuration without rebuilding images.

---

### **46. What is a downward API?**

The **downward API** allows containers to access **metadata about themselves** at runtime.

It can expose:

* Pod name
* Namespace
* Labels
* Resource limits

This information is injected via environment variables or mounted files, enabling context-aware applications.

---

### **47. What is a ServiceAccount?**

A **ServiceAccount** provides an **identity for pods** to authenticate with the Kubernetes API.

Key points:

* Used by applications running inside pods
* Supports role-based access control (RBAC)
* Each namespace has a default ServiceAccount

ServiceAccounts enable secure, fine-grained API access.

---

### **48. How do you create a namespace?**

A namespace can be created using:

```bash
kubectl create namespace my-namespace
```

Or declaratively using YAML:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-namespace
```

Namespaces help logically isolate cluster resources.

---

### **49. How do you list all pods in a namespace?**

To list pods in a specific namespace:

```bash
kubectl get pods -n my-namespace
```

To list pods across all namespaces:

```bash
kubectl get pods --all-namespaces
```

---

### **50. How to switch between namespaces using kubectl?**

You can set a default namespace in the current context:

```bash
kubectl config set-context --current --namespace=my-namespace
```

After this, all `kubectl` commands will apply to that namespace unless overridden.

---

Here's a detailed breakdown of these Kubernetes fundamentals, structured for interview prep with practical context. 🚀

## 1. What is Kubernetes?

Kubernetes (often abbreviated **K8s**) is an **open-source container orchestration platform** originally developed by Google and now maintained by the Cloud Native Computing Foundation (CNCF). It automates the **deployment, scaling, healing, and management** of containerized applications across a cluster of machines.

**Key value propositions:**

* **Self-healing** – restarts failed containers, reschedules them on healthy nodes, and kills unresponsive ones.
* **Horizontal scaling** – scale apps up/down manually or automatically based on CPU/memory or custom metrics.
* **Service discovery & load balancing** – exposes containers via DNS names and distributes traffic.
* **Automated rollouts & rollbacks** – progressive deployments with the ability to revert on failure.
* **Declarative configuration** – you describe the *desired state* in YAML, and Kubernetes continuously reconciles the *actual state* to match it.

> **Interview tip:** Emphasize the **declarative + reconciliation loop** concept — it's the heart of how Kubernetes works.

***

## 2. Architecture of Kubernetes

Kubernetes follows a **master–worker (control plane + data plane)** architecture.

```
┌─────────────────────────────────────────────────────────┐
│                     CONTROL PLANE                         │
│  ┌────────────┐  ┌──────┐  ┌────────────┐  ┌──────────┐  │
│  │kube-apiserver│ │ etcd │  │  scheduler │  │controller│  │
│  └────────────┘  └──────┘  └────────────┘  │ manager  │  │
│                                            └──────────┘  │
└─────────────────────────────────────────────────────────┘
                          │  (API)
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  WORKER NODE │  │  WORKER NODE │  │  WORKER NODE │
│ ┌─────────┐  │  │ ┌─────────┐  │  │ ┌─────────┐  │
│ │ kubelet │  │  │ │ kubelet │  │  │ │ kubelet │  │
│ │kube-proxy│ │  │ │kube-proxy│ │  │ │kube-proxy│ │
│ │ runtime │  │  │ │ runtime │  │  │ │ runtime │  │
│ │  Pods   │  │  │ │  Pods   │  │  │ │  Pods   │  │
│ └─────────┘  │  │ └─────────┘  │  │ └─────────┘  │
└──────────────┘  └──────────────┘  └──────────────┘
```

* **Control Plane** – the "brain" that makes global decisions (scheduling, detecting/responding to events).
* **Worker Nodes** – the "muscle" where your actual application Pods run.

***

## 3. What is a Kubernetes Cluster?

A **Kubernetes cluster** is a set of machines (physical or virtual) that work together to run containerized workloads. It consists of:

| Component                           | Role                                              |
| ----------------------------------- | ------------------------------------------------- |
| **At least one Control Plane node** | Manages cluster state and orchestration decisions |
| **One or more Worker nodes**        | Run the application Pods                          |

The cluster is the **fundamental unit of Kubernetes**. When you `kubectl apply` a manifest, you are telling the cluster your desired state, and the cluster collectively works to achieve it. In production, you typically run **multiple control-plane nodes** (HA setup) to avoid a single point of failure.

***

## 4. Components of the Kubernetes Control Plane

The control plane comprises these core components:

1. **kube-apiserver** – the front door / API gateway to the cluster.
2. **etcd** – the distributed key-value store (source of truth).
3. **kube-scheduler** – assigns Pods to nodes.
4. **kube-controller-manager** – runs controller loops that reconcile state.
5. **cloud-controller-manager** – integrates with the underlying cloud provider.

> Together these ensure the **desired state** stored in etcd is continuously enforced across the cluster.

***

## 5. Role of the kube-apiserver

The **kube-apiserver** is the **central management hub** and the **only component that talks directly to etcd**.

**Key functions:**

* Exposes the **Kubernetes REST API** (what `kubectl`, controllers, and kubelets talk to).
* **Authentication, authorization (RBAC), and admission control** for every request.
* **Validates** and processes API objects (Pods, Services, Deployments).
* Acts as the **communication hub** — all components interact *through* the API server, never directly with each other.

```bash
# Every kubectl command hits the apiserver
kubectl get pods        # → GET request to kube-apiserver
kubectl apply -f app.yaml  # → POST/PATCH to kube-apiserver
```

> **Interview tip:** It's **stateless and horizontally scalable** — you can run multiple replicas behind a load balancer for HA.

***

## 6. Function of etcd in Kubernetes

**etcd** is a **consistent, distributed, highly-available key-value store** that serves as the **single source of truth** for the entire cluster.

**What it stores:**

* All cluster state and configuration (Pods, Services, ConfigMaps, Secrets, node info, etc.).
* The **desired state** and **current state** of every object.

**Key characteristics:**

* Uses the **Raft consensus algorithm** for consistency across replicas.
* Should be deployed in an **odd number** (3, 5) for quorum.
* **Critical to back up** — losing etcd means losing your cluster state.

```bash
# Back up etcd (essential for disaster recovery)
ETCDCTL_API=3 etcdctl snapshot save snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

> **Interview tip:** A common real-world question is *"How do you back up and restore a cluster?"* → answer with etcd snapshots.

***

## 7. Role of the kube-scheduler

The **kube-scheduler** watches for **newly created Pods that have no node assigned** and selects the best node for them to run on.

**Two-phase decision process:**

1. **Filtering (Predicates)** – eliminates nodes that can't run the Pod (insufficient CPU/memory, taints, node selectors not matching).
2. **Scoring (Priorities)** – ranks the remaining feasible nodes and picks the highest-scoring one.

**Factors it considers:**

* Resource requests/limits, node affinity/anti-affinity, taints & tolerations, data locality, and inter-Pod affinity.

> Note: The scheduler only **decides** placement — the **kubelet** on the chosen node actually starts the Pod.

***

## 8. What is the kube-controller-manager?

The **kube-controller-manager** is a single binary that runs multiple **controller loops**, each continuously watching the cluster state and working to move the *current state* toward the *desired state*.

**Key built-in controllers:**

| Controller                    | Responsibility                             |
| ----------------------------- | ------------------------------------------ |
| **Node Controller**           | Monitors node health; marks nodes NotReady |
| **ReplicaSet Controller**     | Ensures the correct number of Pod replicas |
| **Deployment Controller**     | Manages rollouts/rollbacks                 |
| **Job Controller**            | Runs Pods for batch jobs to completion     |
| **Endpoints Controller**      | Populates Service endpoints                |
| **ServiceAccount Controller** | Creates default accounts/tokens            |

> **Interview tip:** Explain the **reconciliation loop**: *observe → compare desired vs actual → act → repeat*.

***

## 9. Purpose of the cloud-controller-manager

The **cloud-controller-manager (CCM)** embeds **cloud-provider-specific control logic**, decoupling it from the core Kubernetes code. This lets cloud vendors (Azure, AWS, GCP) evolve their integrations independently.

**Controllers it runs:**

* **Node Controller** – checks the cloud provider to determine if a deleted node still exists.
* **Route Controller** – configures network routes in the cloud infrastructure.
* **Service Controller** – creates/manages cloud **load balancers** when you deploy a `Service` of type `LoadBalancer`.

> **Real-world example:** On **Azure AKS**, when you create a `LoadBalancer` service, the CCM provisions an **Azure Load Balancer** and assigns a public IP automatically. In an on-prem cluster, this component isn't used.

***

## 10. Role of the kubelet

The **kubelet** is the **primary node agent** that runs on **every worker node**. It's the bridge between the control plane and the container runtime.

**Key responsibilities:**

* Registers the node with the API server.
* Receives **PodSpecs** and ensures the described containers are **running and healthy**.
* Runs **liveness, readiness, and startup probes**.
* Reports node and Pod status back to the API server.
* Mounts volumes, pulls secrets, and interacts with the container runtime via **CRI (Container Runtime Interface)**.

> **Interview tip:** The kubelet only manages containers created by Kubernetes — it does **not** manage arbitrary containers on the host.

***

## 11. Function of kube-proxy

**kube-proxy** is a network component running on **each node** that maintains network rules to enable **Service communication** and load balancing.

**How it works:**

* Watches the API server for **Service** and **Endpoint** changes.
* Programs the node's networking (via **iptables** or **IPVS** mode) to route traffic destined for a Service's virtual IP (ClusterIP) to one of the backing Pods.
* Provides **basic L4 (TCP/UDP) load balancing** across Pod replicas.

```bash
# A Service's ClusterIP is virtual — kube-proxy makes it work
kubectl get svc my-app
# NAME      TYPE        CLUSTER-IP      PORT(S)
# my-app    ClusterIP   10.96.145.22    80/TCP
# Traffic to 10.96.145.22:80 → distributed to healthy Pods
```

> **Interview tip:** **IPVS mode** scales better than iptables for clusters with thousands of Services.

***

## 12. Role of a Pod in Kubernetes

A **Pod** is the **smallest deployable unit** in Kubernetes. It's a wrapper around **one or more containers** that share:

* The same **network namespace** (same IP address & port space — containers communicate via `localhost`).
* The same **storage volumes**.
* The same **lifecycle** (created and destroyed together).

**Key points:**

* Most Pods run a **single container**; multi-container Pods use the **sidecar pattern** (e.g., a logging agent alongside the main app).
* Pods are **ephemeral** — they get a new IP when recreated, which is why you use **Services** for stable networking.
* You rarely create Pods directly; you use controllers like **Deployments** that manage Pods for you.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
```

***

## 13. How do Namespaces work in Kubernetes?

**Namespaces** provide a mechanism to **logically partition a single cluster** into multiple virtual clusters. They enable **multi-tenancy** and resource isolation.

**Use cases:**

* Separate **environments** (dev, staging, prod) in one cluster.
* Isolate **teams or projects**.
* Apply **ResourceQuotas** and **LimitRanges** per namespace.
* Scope **RBAC permissions** to specific namespaces.

**Default namespaces:**

| Namespace         | Purpose                            |
| ----------------- | ---------------------------------- |
| `default`         | Where objects go if none specified |
| `kube-system`     | Kubernetes system components       |
| `kube-public`     | Publicly readable resources        |
| `kube-node-lease` | Node heartbeat lease objects       |

```bash
kubectl create namespace dev
kubectl get pods -n dev
kubectl apply -f app.yaml -n dev
```

> **Important:** Not all objects are namespaced — cluster-scoped objects like **Nodes**, **PersistentVolumes**, and **ClusterRoles** live outside namespaces.

***

## 14. Role of the Container Runtime in Kubernetes

The **container runtime** is the software responsible for **actually running the containers** on each node. Kubernetes doesn't run containers itself — it delegates this to a runtime via the **CRI (Container Runtime Interface)**.

**Responsibilities:**

* Pulling container images from registries.
* Creating, starting, stopping, and deleting containers.
* Managing container isolation, resources, and lifecycle.

**Common runtimes:**

| Runtime                 | Notes                                                                                          |
| ----------------------- | ---------------------------------------------------------------------------------------------- |
| **containerd**          | Most widely used; default in most clusters (incl. AKS)                                         |
| **CRI-O**               | Lightweight, purpose-built for Kubernetes                                                      |
| **Docker (dockershim)** | **Removed in Kubernetes v1.24+** — Docker images still work, but the runtime is now containerd |

> **Interview tip:** A very common question — *"What happened to Docker in Kubernetes?"* → Explain the **dockershim deprecation** (removed in v1.24). Docker-built images (OCI-compliant) still run fine; only the runtime shim was removed.

***



