

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

