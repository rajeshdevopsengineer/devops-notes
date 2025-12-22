

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


