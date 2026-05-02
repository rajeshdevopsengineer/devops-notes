## 1. In Kubernetes, what are the errors you are getting, why they come, and how do you resolve them?

In real projects, Kubernetes issues usually fall into **Pod, Networking, Storage, Scheduling, and Cluster-level errors**.

---

### A. Pod Pending Error

### Why it comes:

* No available node resources (CPU/RAM)
* Taints/Tolerations mismatch
* PVC not bound
* Node selector mismatch

### Resolve:

```bash
kubectl describe pod <pod-name>
kubectl get nodes
kubectl get pvc
```

Check events section.

---

### B. CrashLoopBackOff

### Why:

Application inside container keeps crashing repeatedly.

### Resolve:

```bash
kubectl logs <pod-name>
kubectl describe pod <pod-name>
```

Fix application config, env vars, DB connectivity, memory issue.

---

### C. ImagePullBackOff / ErrImagePull

### Why:

* Wrong image name
* Wrong tag
* Private registry auth missing
* Network issue

### Resolve:

```bash
kubectl describe pod <pod-name>
kubectl get secret
```

Correct image / add registry secret.

---

### D. OOMKilled

### Why:

Container exceeded memory limit.

### Resolve:

Increase limits:

```yaml
resources:
  requests:
    memory: "512Mi"
  limits:
    memory: "1Gi"
```

---

### E. Node Not Ready

### Why:

* Kubelet stopped
* Disk full
* Network issue

### Resolve:

```bash
systemctl status kubelet
df -h
journalctl -u kubelet
```

---

### F. PVC Pending

### Why:

StorageClass missing / no storage provisioner.

### Resolve:

```bash
kubectl get sc
kubectl describe pvc
```

---

### G. DNS Resolution Error

### Why:

CoreDNS issue.

### Resolve:

```bash
kubectl get pods -n kube-system
kubectl logs -n kube-system <coredns-pod>
```

---

## 2. Explain CrashLoopBackOff and Image Pull Error

---

# CrashLoopBackOff

Means container starts, crashes, Kubernetes restarts it repeatedly.

### Causes:

* App code failure
* Wrong startup command
* Missing config/env variables
* Port conflict
* DB not reachable
* Memory issue

### Troubleshooting:

```bash
kubectl logs pod-name
kubectl logs pod-name --previous
kubectl describe pod pod-name
```

---

# ImagePullBackOff / ErrImagePull

Means Kubernetes cannot download image.

### Causes:

* Wrong image name/tag
* Registry unavailable
* Private registry auth missing

### Fix:

```bash
kubectl create secret docker-registry regcred ...
```

Then use:

```yaml
imagePullSecrets:
- name: regcred
```

---

## 3. Command to go inside a Pod

```bash
kubectl exec -it <pod-name> -- /bin/bash
```

If bash not available:

```bash
kubectl exec -it <pod-name> -- /bin/sh
```

For multi-container pod:

```bash
kubectl exec -it <pod-name> -c <container-name> -- /bin/bash
```

---

## 4. What are prerequisites to upgrade K8s Cluster?

Cluster upgrade is critical.

---

### Checklist:

### A. Backup ETCD

```bash
etcdctl snapshot save backup.db
```

### B. Read Version Compatibility

Check Kubernetes version skew policy.

### C. Check Workloads

Ensure apps support target version.

### D. Drain Nodes Plan

Need maintenance window.

### E. Check Add-ons

* CNI plugin
* CoreDNS
* Ingress controller
* CSI driver

### F. Check API Deprecation

Old APIs removed in new versions.

### G. Backup manifests

```bash
kubectl get all -A -o yaml
```

### H. Health Check

```bash
kubectl get nodes
kubectl get pods -A
```

---

## 5. What is PDB in K8s?

PDB = **Pod Disruption Budget**

Used to ensure minimum pods remain available during:

* Node drain
* Cluster upgrade
* Voluntary maintenance

---

### Example:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: nginx
```

Means minimum 2 pods must run.

---

### Use Case:

If deployment has 3 replicas, only 1 can be disrupted.

---

## 6. How do we make a K8s Cluster Highly Available?

HA means no single point of failure.

---

## Control Plane HA

Need multiple master nodes.

* 3 Control Plane nodes
* Load Balancer in front of API server

Architecture:

```text
LB
 |--- Master1
 |--- Master2
 |--- Master3
```

---

## ETCD HA

Use odd number nodes:

* 3 node ETCD cluster
* 5 node for large environments

---

## Worker Node HA

Multiple worker nodes across zones.

---

## Application HA

Use:

* ReplicaSets
* Deployments
* HPA
* PDB

---

## Storage HA

Use replicated storage:

* Ceph
* Portworx
* Cloud managed disks

---

## Multi AZ Setup

Spread nodes across zones.

---

## 7. How can you create the Kubernetes cluster?

Depends on environment.

---

## A. kubeadm (On-prem / VM)

```bash
kubeadm init
kubeadm join
```

---

## B. Managed Services

* Amazon Web Services EKS
* Microsoft AKS
* Google GKE

---

## C. Local Testing

* Minikube
* Kind

---

## 8. What are the steps to create the cluster?

Using kubeadm:

---

### Step 1 Install Runtime

Containerd / Docker

### Step 2 Install packages

```bash
apt install kubeadm kubelet kubectl
```

### Step 3 Disable swap

```bash
swapoff -a
```

### Step 4 Initialize master

```bash
kubeadm init --pod-network-cidr=10.244.0.0/16
```

### Step 5 Configure kubectl

```bash
mkdir -p $HOME/.kube
cp /etc/kubernetes/admin.conf $HOME/.kube/config
```

### Step 6 Install CNI

Example Flannel / Calico

```bash
kubectl apply -f calico.yaml
```

### Step 7 Join worker nodes

```bash
kubeadm join <master-ip>:6443 --token xxxx
```

---

## 9. What is Master Node and Other Node?

---

## Master Node (Control Plane)

Manages cluster.

Components:

* API Server
* Scheduler
* Controller Manager
* ETCD

Responsibilities:

* Accept API requests
* Schedule pods
* Maintain desired state

---

## Worker Node

Runs application containers.

Components:

* kubelet
* kube-proxy
* container runtime

Responsibilities:

* Run Pods
* Report node status

---

## 10. How would you implement Blue-Green Deployment in Kubernetes?

Blue-Green = Two environments:

* Blue = Current production
* Green = New version

Traffic switches after testing.

---

## Example:

### Existing deployment

```yaml
app: myapp-blue
```

### New deployment

```yaml
app: myapp-green
```

### Service initially points to blue:

```yaml
selector:
  app: myapp-blue
```

### After testing switch service:

```yaml
selector:
  app: myapp-green
```

---

## Steps:

1. Deploy Green version
2. Test Green internally
3. Change Service selector
4. Traffic moves instantly
5. Keep Blue as rollback

---

## Benefits:

* Zero downtime
* Easy rollback
* Safer release

---

## Real Enterprise Tools:

* Argo Rollouts
* Istio
* NGINX Ingress
* Service Mesh

---

# Interview Tip (Strong Answer)

Say:

> In production I troubleshoot using kubectl logs, describe, events, metrics, and node checks. For upgrades I backup ETCD, validate add-ons, and drain nodes sequentially. For zero downtime I prefer blue-green or canary deployments.

---

If you'd like, I can also give **Top 50 Kubernetes Production Scenario Interview Questions with Answers (Senior DevOps Level)** next.

## 11. Why is Kubernetes considered over Docker? Mention the advantages of it.

First, clarify in interviews:

**Docker** and **Kubernetes** are not direct competitors.

* **Docker** = Container runtime / image platform used to build and run containers.
* **Kubernetes** = Container orchestration platform used to manage containers at scale.

So Kubernetes is chosen **over plain Docker-only environments** when running many containers in production.

---

## Why Kubernetes is Preferred

### 1. Auto Scaling

Automatically increases/decreases pods based on load.

Example:

```text
Traffic increases → Pods scale from 3 to 10
```

### 2. Self Healing

If container crashes:

* Restart pod
* Recreate failed pod
* Replace unhealthy pod

### 3. Load Balancing

Distributes traffic across multiple pods.

### 4. Rolling Updates / Rollbacks

Deploy new version without downtime.

### 5. Service Discovery

Pods communicate using service names instead of IPs.

### 6. High Availability

Pods run across multiple nodes.

### 7. Secret / Config Management

Use ConfigMaps and Secrets.

### 8. Multi-cloud Support

Runs on:

* Amazon Web Services EKS
* Microsoft AKS
* Google GKE
* On-premises

---

## Interview Answer

> Docker runs containers. Kubernetes manages containers in production with scaling, healing, upgrades, networking, and HA. That’s why enterprises use Kubernetes.

---

# 12. Explain the architecture of Kubernetes. Mention each component’s role.

Kubernetes architecture has two parts:

1. Control Plane (Master Node)
2. Worker Nodes

---

# Architecture Diagram

```text
Users / CI-CD
     |
 API Server
     |
 -----------------------
 | Scheduler           |
 | Controller Manager  |
 | ETCD                |
 -----------------------
      |
 ------------------------
 | Worker Node 1        |
 | Worker Node 2        |
 ------------------------
```

---

# A. Control Plane Components

## 1. API Server

Main entry point.

```bash
kubectl apply -f app.yaml
```

Request goes to API Server.

### Role:

* Accept REST requests
* Validate requests
* Update cluster state

---

## 2. ETCD

Distributed key-value database.

Stores:

* Pods
* Nodes
* Secrets
* Configurations

### Role:

Cluster brain/database.

---

## 3. Scheduler

Chooses node for pod placement.

Checks:

* CPU
* Memory
* Taints
* Affinity

---

## 4. Controller Manager

Maintains desired state.

Examples:

* Replica controller
* Node controller
* Endpoint controller

---

# B. Worker Node Components

## 1. Kubelet

Agent running on node.

### Role:

* Talks to API server
* Starts pods
* Reports health

---

## 2. Container Runtime

Examples:

* containerd
* CRI-O

Runs containers.

---

## 3. kube-proxy

Handles service networking.

Routes traffic to pods.

---

# 13. What are Services in Kubernetes? Explain.

Pods are temporary and IP changes often.

**Service** gives stable network access to pods.

---

## Why Needed?

Instead of pod IP:

```text
10.244.1.5 (changes)
```

Use:

```text
myapp-service
```

---

# Types of Services

## 1. ClusterIP (Default)

Internal communication only.

Use case:

Backend DB service.

```yaml
type: ClusterIP
```

---

## 2. NodePort

Exposes service on node port.

```yaml
type: NodePort
```

Access:

```text
NodeIP:30080
```

---

## 3. LoadBalancer

Creates cloud load balancer.

Used in Microsoft AKS / Amazon Web Services EKS.

---

## 4. ExternalName

Maps service to DNS.

---

# Interview Answer

> Service provides stable IP/DNS for pods and load balances traffic.

---

# 14. What is a Namespace? What is its role?

Namespace logically separates resources inside same cluster.

---

## Example

```text
dev
test
prod
monitoring
```

Each team uses same cluster safely.

---

## Benefits

### 1. Isolation

Same pod names allowed in different namespaces.

### 2. Resource Quotas

Limit CPU/RAM.

### 3. RBAC Access Control

Grant user access to specific namespace only.

### 4. Organization

Separate environments.

---

## Commands

```bash
kubectl get ns
kubectl create ns dev
kubectl get pods -n dev
```

---

# 15. What is difference between Deployment and StatefulSet?

| Feature      | Deployment       | StatefulSet              |
| ------------ | ---------------- | ------------------------ |
| Pod Identity | Random           | Fixed                    |
| Pod Names    | Dynamic          | Ordered                  |
| Storage      | Shared/ephemeral | Persistent unique volume |
| Scaling      | Easy             | Ordered                  |
| Use Case     | Stateless apps   | Stateful apps            |

---

## Deployment Use Cases

* Nginx
* Frontend apps
* APIs

---

## StatefulSet Use Cases

* MongoDB MongoDB
* Apache Kafka Kafka
* Redis Redis Cluster

---

# 16. What is a ConfigMap, and how is it different from a Secret?

Both store configuration data externally.

---

# ConfigMap

Stores non-sensitive config.

Examples:

* App URL
* Port
* Environment name

```yaml
data:
  APP_ENV: prod
```

---

# Secret

Stores sensitive data.

Examples:

* Passwords
* Tokens
* Certificates

```yaml
data:
  password: cGFzc3dvcmQ=
```

(Base64 encoded)

---

# Differences

| Feature        | ConfigMap | Secret    |
| -------------- | --------- | --------- |
| Sensitive Data | No        | Yes       |
| Encoding       | Plain     | Base64    |
| Use Case       | Config    | Passwords |

---

# 17. What is Autoscaling and its types? When can we use vertical scaling?

Autoscaling adjusts resources automatically.

---

# Types

## 1. Horizontal Pod Autoscaler (HPA)

Adds/removes pods.

```text
3 pods → 10 pods
```

Based on:

* CPU
* Memory
* Custom metrics

---

## 2. Vertical Pod Autoscaler (VPA)

Increases pod resources.

```text
512Mi → 2Gi RAM
```

---

## 3. Cluster Autoscaler

Adds/removes nodes.

```text
3 nodes → 5 nodes
```

---

# When to Use Vertical Scaling?

Use when application cannot scale horizontally.

Examples:

* Legacy apps
* Single-threaded DB tools
* License-based software
* Stateful apps needing more RAM

---

# 18. What is difference between StatefulSet and Deployment?

(Important repeated question)

## Deployment

Used for stateless apps.

* Pods replace freely
* No identity required

## StatefulSet

Used for apps needing:

* Stable hostname
* Ordered startup/shutdown
* Persistent storage

Example:

```text
mysql-0
mysql-1
mysql-2
```

---

# 19. Difference between StatefulSet, DaemonSet, and ReplicaSet. Use cases.

---

# StatefulSet

For stateful apps.

### Features:

* Stable names
* Ordered deployment
* Persistent storage

### Example:

MongoDB replica set cluster.

---

# DaemonSet

Runs one pod on every node.

### Use Cases:

* Logging agent
* Monitoring agent
* Security scanner

Examples:

* Datadog Agent
* Elastic Filebeat
* Prometheus Node Exporter

---

# ReplicaSet

Ensures fixed number of pods running.

```text
replicas: 3
```

Usually managed by Deployment.

---

# Comparison

| Type        | Purpose           | Example       |
| ----------- | ----------------- | ------------- |
| StatefulSet | Stateful apps     | DB cluster    |
| DaemonSet   | One pod per node  | Log collector |
| ReplicaSet  | Maintain replicas | Web pods      |

---

# 20. Write YAML file for simple nginx pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx-container
    image: nginx:latest
    ports:
    - containerPort: 80
```

---

## Apply Command

```bash
kubectl apply -f nginx-pod.yaml
```

---

## Verify

```bash
kubectl get pods
kubectl describe pod nginx-pod
```

---

# Interview Bonus Tip

If interviewer asks YAML for production pod, add:

* resource limits
* readiness probe
* liveness probe
* securityContext

---

## 21. Write an imperative command to create a deployment with image nginx and replica count of 3

Create deployment:

```bash id="8b70jh"
kubectl create deployment nginx-deployment --image=nginx
```

Scale to 3 replicas:

```bash id="b5gzyu"
kubectl scale deployment nginx-deployment --replicas=3
```

---

## One-liner:

```bash id="0h68m7"
kubectl create deployment nginx-deployment --image=nginx --replicas=3
```

*(Works in newer kubectl versions depending on release)*

---

## Verify:

```bash id="18o0c6"
kubectl get deploy
kubectl get pods
```

---

# 22. What does Node Affinity do? Mention its rules and what it does.

## What is Node Affinity?

Node Affinity controls **which node a pod should run on** based on node labels.

Used for:

* SSD nodes only
* GPU nodes only
* Region-specific nodes
* Dedicated nodes

---

# Example Labels

```bash id="9g0uv3"
kubectl label node worker1 disktype=ssd
kubectl label node worker2 env=prod
```

---

# Types / Rules

## 1. RequiredDuringSchedulingIgnoredDuringExecution

Mandatory rule.

If no matching node exists → pod stays Pending.

```yaml id="0x1d3u"
requiredDuringSchedulingIgnoredDuringExecution
```

Use when pod must run only on SSD node.

---

## 2. PreferredDuringSchedulingIgnoredDuringExecution

Soft rule.

Scheduler tries preferred node, but can place elsewhere.

```yaml id="v8c8iu"
preferredDuringSchedulingIgnoredDuringExecution
```

---

# Example YAML

```yaml id="0m69q2"
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: disktype
          operator: In
          values:
          - ssd
```

---

# Real Use Cases

* Database pods on SSD nodes
* AI workloads on GPU nodes
* Production workloads on dedicated nodes

---

# 23. Code to create a cluster using Terraform?

Most common real-world example = create managed Kubernetes cluster like Microsoft AKS, Amazon Web Services EKS, or Google GKE.

---

# Example: AKS using Terraform

```hcl id="1lw4vf"
provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "rg" {
  name     = "aks-rg"
  location = "Central India"
}

resource "azurerm_kubernetes_cluster" "aks" {
  name                = "myakscluster"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  dns_prefix          = "myaks"

  default_node_pool {
    name       = "default"
    node_count = 2
    vm_size    = "Standard_DS2_v2"
  }

  identity {
    type = "SystemAssigned"
  }
}
```

---

# Commands

```bash id="2u7fsq"
terraform init
terraform plan
terraform apply
```

---

# 24. What is Ingress in Kubernetes, and how can we access applications deployed on-premises?

# What is Ingress?

Ingress manages **HTTP/HTTPS external access** to services.

Features:

* URL-based routing
* Hostname routing
* SSL termination
* Load balancing

---

# Example

```text id="5m3x7m"
app.company.com -> frontend service
api.company.com -> backend service
```

---

# Ingress Needs Controller

Examples:

* NGINX Ingress Controller
* HAProxy
* Traefik

---

# On-Prem Access Methods

## Option 1: External Load Balancer

Use F5 / HAProxy / NGINX reverse proxy in front of Ingress.

## Option 2: NodePort + Reverse Proxy

Traffic:

```text id="mxn9mz"
Internet -> Firewall -> Reverse Proxy -> NodePort -> Service -> Pod
```

## Option 3: MetalLB

For bare-metal Kubernetes to provide LoadBalancer IP.

---

# 25. What is the port range for NodePort mode?

Default Kubernetes NodePort range:

```text id="rcqg1h"
30000 - 32767
```

---

# Example

```yaml id="h8a4f5"
type: NodePort
ports:
- port: 80
  nodePort: 30080
```

Access:

```text id="o8j2sj"
http://NodeIP:30080
```

---

# 26. How do you expose a Kubernetes application to external traffic?

Several methods:

---

# 1. NodePort

```bash id="49m12d"
kubectl expose deployment nginx --type=NodePort --port=80
```

---

# 2. LoadBalancer

Cloud environments:

```bash id="n6j6uv"
kubectl expose deployment nginx --type=LoadBalancer --port=80
```

Used in Microsoft AKS / Amazon Web Services EKS

---

# 3. Ingress (Best for HTTP/HTTPS)

Multiple apps with one public IP.

---

# 4. Port Forward (Temporary)

```bash id="ljlwmk"
kubectl port-forward pod/nginx-pod 8080:80
```

---

# 27. What is PV and PVC in Kubernetes? Which one comes first, and why?

# PV = Persistent Volume

Cluster storage resource.

Examples:

* NFS
* Azure Disk
* EBS
* Ceph

---

# PVC = Persistent Volume Claim

Storage request by user/app.

Example:

```text id="ezr3gd"
Need 10Gi ReadWriteOnce
```

---

# Which Comes First?

## Traditional Static Provisioning:

PV first, then PVC binds to matching PV.

## Dynamic Provisioning (Modern)

PVC first → StorageClass automatically creates PV.

---

# Why?

PVC separates developers from storage admins.

Developers request storage without knowing backend.

---

# Example Flow

```text id="h4ud9q"
PVC request -> StorageClass -> PV created -> Pod mounts PVC
```

---

# 28. Explain RBAC – what is it, its uses, and how is it configured?

# RBAC = Role Based Access Control

Controls who can do what in Kubernetes.

---

# Uses

* Restrict namespace access
* Read-only users
* Dev team can deploy only in dev namespace
* Security compliance

---

# Components

## Role

Namespace permissions.

## ClusterRole

Cluster-wide permissions.

## RoleBinding

Assign Role to user/service account.

## ClusterRoleBinding

Assign ClusterRole cluster-wide.

---

# Example

```yaml id="v2s2lq"
kind: Role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get","list","watch"]
```

Bind to user:

```yaml id="s5mbuk"
kind: RoleBinding
```

---

# Commands

```bash id="k6cfpq"
kubectl auth can-i create pods --as=user1
```

---

# 29. How can we expose on-premise applications?

On-prem apps can be exposed using:

---

# Option 1: Reverse Proxy

NGINX / HAProxy in DMZ.

---

# Option 2: Firewall NAT

Public IP forwards to internal app.

---

# Option 3: Kubernetes Ingress

If app inside K8s cluster:

```text id="3v4h7n"
Public DNS -> Firewall -> Ingress -> Service -> Pods
```

---

# Option 4: VPN

Private secure access only.

---

# Option 5: NodePort

Direct node exposure (less preferred production).

---

# Best Practice

Use:

* Reverse proxy
* SSL certificate
* WAF
* Authentication
* Restricted ports

---

# 30. Use kubectl exec to run nslookup with full FQDN: svc.namespace.svc.cluster.local

Example command:

```bash id="l32gaj"
kubectl exec -it test-pod -- nslookup myservice.mynamespace.svc.cluster.local
```

---

# Example

```bash id="n8hl3j"
kubectl exec -it busybox -- nslookup nginx.default.svc.cluster.local
```

---

# Why Used?

To test:

* DNS resolution
* Service discovery
* CoreDNS health
* Cross-namespace connectivity

---

# If Pod Has No nslookup

Use temporary pod:

```bash id="c7umra"
kubectl run dns-test --rm -it --image=busybox -- sh
```

Then:

```bash id="6hb7ra"
nslookup nginx.default.svc.cluster.local
```

---

# Interview Tip

Say:

> I use kubectl exec with nslookup or curl from inside pods to troubleshoot service DNS, networking, and namespace communication issues.

---



