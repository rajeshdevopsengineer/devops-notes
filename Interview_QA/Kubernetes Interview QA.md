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

## 31. Check if the pod’s ServiceAccount has permission to list services in that namespace

When a pod accesses the Kubernetes API, it uses its attached **ServiceAccount** identity. If RBAC permissions are missing, DNS sidecars, operators, controllers, or app code may fail when querying Services.

---

## Step 1: Identify Pod ServiceAccount

```bash id="o1n5v6"
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.serviceAccountName}'
```

Default is usually:

```text id="u2q5ca"
default
```

---

## Step 2: Test Permission Using kubectl auth can-i

```bash id="y8m0rn"
kubectl auth can-i list services \
--as=system:serviceaccount:<namespace>:<serviceaccount> \
-n <namespace>
```

Example:

```bash id="w7j4v9"
kubectl auth can-i list services \
--as=system:serviceaccount:dev:web-sa \
-n dev
```

---

## Output Meaning

```text id="i9g3pd"
yes  -> permission granted
no   -> denied by RBAC
```

---

## Fix

Create Role + RoleBinding.

```yaml id="u4w9sj"
kind: Role
rules:
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get","list","watch"]
```

Bind it to ServiceAccount.

---

## Real Use Case

Apps using Kubernetes API for service discovery or operators needing to watch services.

---

# 32. Inspect RBAC roles — is there a RoleBinding for discovery?

If ServiceAccount has no permission, check whether RoleBinding exists.

---

## List RoleBindings in Namespace

```bash id="x1j6qn"
kubectl get rolebinding -n <namespace>
```

---

## Describe Specific RoleBinding

```bash id="u6f4mt"
kubectl describe rolebinding <rolebinding-name> -n <namespace>
```

Look for:

```text id="u9k1ca"
Subjects:
  Kind: ServiceAccount
  Name: web-sa

RoleRef:
  Kind: Role
  Name: discovery-role
```

---

## Check Role Permissions

```bash id="x0f3lu"
kubectl describe role discovery-role -n <namespace>
```

Need:

```text id="t7k0nb"
resources: services
verbs: get,list,watch
```

---

## If Missing, Create Binding

```yaml id="n6v3mx"
kind: RoleBinding
subjects:
- kind: ServiceAccount
  name: web-sa
roleRef:
  kind: Role
  name: discovery-role
```

---

## Interview Answer

> I verify both Role permissions and whether RoleBinding correctly maps the ServiceAccount.

---

# 33. Review NetworkPolicies – are they blocking intra-namespace traffic?

Kubernetes NetworkPolicy controls pod-to-pod traffic. If deny rules exist, DNS or service communication may fail.

---

## Step 1: List Policies

```bash id="v0r7yt"
kubectl get networkpolicy -n <namespace>
```

---

## Step 2: Describe Policies

```bash id="c8d2la"
kubectl describe networkpolicy <policy-name> -n <namespace>
```

---

## What to Check

### Ingress Rules

Allows incoming traffic.

### Egress Rules

Allows outgoing traffic.

### Pod Selectors

Which pods are affected.

---

## Common DNS Blocking Issue

Pod cannot reach CoreDNS because UDP/TCP 53 blocked.

Need egress allow:

```yaml id="x8s4rz"
ports:
- protocol: UDP
  port: 53
- protocol: TCP
  port: 53
```

---

## Intra-namespace Block Example

If policy only allows labeled pods:

```text id="v7m0lw"
app=frontend can talk
others blocked
```

---

## Test Connectivity

```bash id="v2u1ox"
kubectl exec -it <pod> -- wget -qO- http://service-name
```

---

## Interview Answer

> When NetworkPolicy exists, I always validate DNS egress and namespace pod selectors first.

---

# 34. Examine CoreDNS ConfigMap – look for caching misconfigurations or recursive loops

CoreDNS provides cluster DNS.

---

## View ConfigMap

```bash id="j4h8ke"
kubectl get configmap coredns -n kube-system -o yaml
```

or

```bash id="n2z5dr"
kubectl edit configmap coredns -n kube-system
```

---

## Typical Corefile Example

```text id="s1u0gn"
.:53 {
    errors
    health
    kubernetes cluster.local
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
}
```

---

# What to Check

## 1. Cache Misconfiguration

Too high cache TTL may return stale records.

```text id="k5q2cx"
cache 3600
```

May delay updates.

Better:

```text id="n3j4sz"
cache 30
```

---

## 2. Recursive Forwarding Loop

If forward points back to itself:

```text id="m4r7qp"
forward . 10.x.x.x
```

Where IP is CoreDNS service IP.

This creates loop.

Use valid upstream DNS.

---

## 3. Missing kubernetes Plugin

Without it, cluster service DNS fails.

---

## 4. Syntax Errors

After changes restart pods:

```bash id="q2e7wy"
kubectl rollout restart deployment coredns -n kube-system
```

---

## Logs

```bash id="d7m3ko"
kubectl logs -n kube-system deploy/coredns
```

---

# 35. Use tcpdump inside the pod to confirm UDP port 53 DNS traffic

Used when DNS issue is unclear.

---

## Exec Into Pod

```bash id="u8n2lr"
kubectl exec -it <pod-name> -- sh
```

---

## Run tcpdump

If installed:

```bash id="e4j6pn"
tcpdump -i any udp port 53
```

---

## Filter for Specific DNS Server

```bash id="n5q9sv"
tcpdump -i any host <dns-ip> and port 53
```

---

## What You Confirm

### Successful Query

```text id="j1z6wa"
Pod -> DNS server UDP/53
Response returned
```

### No Packets

Likely:

* NetworkPolicy block
* CNI issue
* Pod routing issue

### Query Sent, No Reply

Likely:

* DNS server unreachable
* Firewall block
* CoreDNS down

---

## If tcpdump Not Installed

Use debug pod:

```bash id="v4t0cm"
kubectl run debug --rm -it --image=nicolaka/netshoot -- bash
```

Then:

```bash id="p1m7dr"
tcpdump -i any udp port 53
```

---

## Interview Tip (Strong Answer)

> For DNS failures I validate in sequence: serviceaccount RBAC, rolebindings, network policies, CoreDNS config, then packet capture on UDP 53 to isolate whether the issue is auth, policy, config, or network.

---
## 36. Confirm EndpointSlices and actual backend pod readiness

When a Service is not routing traffic, check whether backend pods are actually registered and Ready.

---

# What is EndpointSlice?

EndpointSlice stores pod endpoints behind a Service (modern replacement for Endpoints).

It tells Kubernetes which pod IPs should receive traffic.

---

## Step 1: Check Service

```bash id="a1f2g3"
kubectl get svc -n <namespace>
```

---

## Step 2: Check EndpointSlices

```bash id="b2g3h4"
kubectl get endpointslice -n <namespace>
```

Detailed:

```bash id="c3h4i5"
kubectl describe endpointslice <name> -n <namespace>
```

Look for:

```text id="d4i5j6"
Addresses: 10.244.1.8
Conditions:
  Ready: true
```

---

## Step 3: Check Pod Readiness

```bash id="e5j6k7"
kubectl get pods -n <namespace> -o wide
```

If pod is:

```text id="f6k7l8"
1/1 Running   -> Ready
0/1 Running   -> Not Ready
```

---

## Why Endpoints Missing?

* Wrong Service selector
* Readiness probe failing
* Pod not Running
* Namespace mismatch

---

## Fix Example

```bash id="g7l8m9"
kubectl get svc mysvc -o yaml
kubectl get pods --show-labels
```

Ensure labels match selector.

---

# 37. What are the core components of Kubernetes, and which are the most important?

Kubernetes has **Control Plane** + **Worker Node** components.

---

# Control Plane Components

## 1. API Server (Most Important)

Main entry point for all requests.

```bash id="h8m9n0"
kubectl apply -f app.yaml
```

goes to API Server.

---

## 2. ETCD (Critical)

Stores cluster state:

* Pods
* Nodes
* Secrets
* Configs

If ETCD fails badly, cluster state is impacted.

---

## 3. Scheduler

Chooses which node runs pod.

---

## 4. Controller Manager

Ensures desired state.

Example:

```text id="i9n0o1"
3 replicas desired -> recreate missing pod
```

---

# Worker Components

## 5. Kubelet

Node agent. Starts pods.

## 6. Container Runtime

Runs containers (containerd / CRI-O)

## 7. kube-proxy

Service networking.

---

# Most Important in Interview Terms

1. API Server
2. ETCD
3. Kubelet
4. Scheduler

---

# 38. A pod is in CrashLoopBackOff but logs show no error — how do you debug it?

Very common production question.

---

# Step-by-Step Debugging

## 1. Check Previous Logs

```bash id="j0o1p2"
kubectl logs <pod> --previous
```

Current logs may be empty if crash is immediate.

---

## 2. Describe Pod

```bash id="k1p2q3"
kubectl describe pod <pod>
```

Check:

* Exit code
* OOMKilled
* Probe failures
* Events

---

## 3. Check Command / Entrypoint

Wrong startup command often exits silently.

```bash id="l2q3r4"
kubectl get pod <pod> -o yaml
```

---

## 4. Check Resources

OOM kills may show no app logs.

```bash id="m3r4s5"
kubectl top pod
```

---

## 5. Run Interactive Shell

Override command:

```bash id="n4s5t6"
kubectl run debug --rm -it --image=<image> -- sh
```

Inspect filesystem/env.

---

## 6. Check Dependencies

* DB unreachable
* Secret missing
* ConfigMap wrong
* DNS failure

---

# Strong Answer

> If logs are empty, I check previous logs, describe events, exit code, OOMKilled, startup command, probes, and dependency failures.

---

# 39. Explain how HPA and VPA work together and when to use each.

# HPA = Horizontal Pod Autoscaler

Scales pod count.

```text id="o5t6u7"
3 pods -> 10 pods
```

Based on CPU/memory/custom metrics.

---

# VPA = Vertical Pod Autoscaler

Adjusts pod resources.

```text id="p6u7v8"
CPU 200m -> 500m
RAM 512Mi -> 1Gi
```

---

# Can They Work Together?

Yes, but carefully.

Best practice:

* HPA uses CPU/custom metrics
* VPA in recommend mode or memory-focused

Avoid conflict when both change CPU aggressively.

---

# Use Cases

## Use HPA For:

* Stateless web apps
* APIs
* Traffic spikes

## Use VPA For:

* Memory-heavy apps
* Batch jobs
* Apps hard to scale horizontally

---

# Combined Strategy

* HPA for scale-out
* VPA for right-sizing

---

# 40. What are readiness and liveness probes and what mistakes can cause them to fail?

# Readiness Probe

Checks if pod is ready to receive traffic.

If fails:

* Pod stays running
* Removed from Service endpoints

---

# Liveness Probe

Checks if pod is alive.

If fails:

* Container restarted

---

# Example

```yaml id="q7v8w9"
livenessProbe:
  httpGet:
    path: /health
    port: 8080

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
```

---

# Common Mistakes

## 1. Wrong Port

App on 8080, probe checks 80.

## 2. Wrong Path

```text id="r8w9x0"
/healthz vs /health
```

## 3. Startup Delay Too Low

App needs 60 sec, probe starts in 5 sec.

Use:

```yaml id="s9x0y1"
initialDelaySeconds: 60
```

## 4. Timeout Too Short

## 5. Dependency-Based Readiness Too Strict

If DB slow, pod never ready.

---

# 41. How do you create and manage Kubernetes clusters (using tools like Terraform), and what are the master and worker nodes?

# Create Cluster with Terraform

Usually managed services:

* Microsoft AKS
* Amazon Web Services EKS
* Google GKE

Terraform example:

```bash id="t0y1z2"
terraform init
terraform plan
terraform apply
```

---

# Master Node (Control Plane)

Runs:

* API Server
* ETCD
* Scheduler
* Controller Manager

Manages cluster.

---

# Worker Nodes

Run applications.

Components:

* kubelet
* container runtime
* kube-proxy

---

# Management Tasks

* Upgrades
* Autoscaling
* Node pools
* RBAC
* Monitoring
* Backups

---

# 42. What are common Kubernetes errors you’ve faced and how did you resolve them?

# CrashLoopBackOff

Cause:

* App crash
* Wrong config
* OOM

Fix:

```bash id="u1z2a3"
kubectl logs --previous
kubectl describe pod
```

---

# ImagePullBackOff

Cause:

* Wrong image tag
* Registry auth issue

Fix:

```bash id="v2a3b4"
kubectl create secret docker-registry
```

---

# Pending Pod

Cause:

* No resources
* PVC issue
* Affinity mismatch

Fix:

```bash id="w3b4c5"
kubectl describe pod
```

---

# DNS Failure

Cause:

CoreDNS / NetworkPolicy.

---

# Probe Failure

Cause:

Wrong health check config.

---

# 43. What is the command to access a pod and how can you define or create a Kubernetes class or object?

# Access Pod

```bash id="x4c5d6"
kubectl exec -it <pod-name> -- /bin/sh
```

or

```bash id="y5d6e7"
kubectl exec -it <pod-name> -- /bin/bash
```

---

# Create Kubernetes Object

Objects = Pod, Deployment, Service, ConfigMap, Secret, etc.

Create via YAML:

```yaml id="z6e7f8"
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
```

Apply:

```bash id="a7f8g9"
kubectl apply -f pod.yaml
```

---

# 44. How would you set up an automated rollback strategy in Kubernetes for failed deployments?

# Native Kubernetes Rolling Update

Deployment tracks ReplicaSets.

Rollback:

```bash id="b8g9h0"
kubectl rollout undo deployment/myapp
```

---

# Automated Strategy

## 1. Readiness/Liveness Probes

If new pods fail readiness, traffic not routed.

## 2. Progress Deadline

```yaml id="c9h0i1"
progressDeadlineSeconds: 600
```

Deployment marked failed.

## 3. CI/CD Automation

Pipeline checks rollout:

```bash id="d0i1j2"
kubectl rollout status deployment/myapp
```

If failed:

```bash id="e1j2k3"
kubectl rollout undo deployment/myapp
```

---

# Better Enterprise Tools

* Argo Rollouts
* Spinnaker
* Jenkins pipelines

---

# 45. Your pod keeps getting stuck in CrashLoopBackOff, but logs show no errors. How would you approach debugging and resolution?

# Structured Approach

## Step 1

```bash id="f2k3l4"
kubectl describe pod <pod>
```

Check events / restart reason.

---

## Step 2

```bash id="g3l4m5"
kubectl logs <pod> --previous
```

---

## Step 3

Check Exit Code:

```text id="h4m5n6"
137 = OOMKilled
143 = graceful stop
1   = app error
```

---

## Step 4

Check Probes

Wrong liveness probe may restart healthy app.

---

## Step 5

Check Config

* Secret mounted?
* ConfigMap valid?
* Env vars set?

---

## Step 6

Check Startup Time

Use:

```yaml id="i5n6o7"
startupProbe:
```

for slow apps.

---

## Step 7

Run image manually for debugging.

---

# Final Interview Answer

> Even with empty logs, I use describe output, previous logs, exit codes, probe config, resources, and dependency checks. Most silent CrashLoopBackOff cases are probe failures, OOM kills, or wrong startup commands.

---

## 46. You have a StatefulSet deployed with persistent volumes, and one of the pods is not recreating properly after deletion. What could be the reasons, and how do you fix it without data loss?

StatefulSets are different from Deployments because pods have:

* Stable names (`db-0`, `db-1`)
* Persistent storage
* Ordered startup/shutdown
* Stable network identity

If pod is deleted and not recreated properly, investigate carefully to avoid data loss.

---

# Possible Reasons

## 1. PVC / PV Binding Issue

Pod waits because volume not attached or PVC pending.

```bash id="a1b2c3"
kubectl get pvc -n <ns>
kubectl describe pvc <pvc-name>
```

Look for:

```text id="b2c3d4"
Pending
Lost
FailedAttachVolume
```

---

## 2. Storage Backend Issue

Cloud disk / SAN / NFS unavailable.

Examples:

* Microsoft Azure Disk detach issue
* Amazon Web Services EBS still attached
* NFS mount failure

Check events:

```bash id="c3d4e5"
kubectl describe pod <pod-name>
```

---

## 3. Ordered Startup Blocking

If `db-0` unhealthy, `db-1` or later pods may wait depending on policy.

---

## 4. Node Affinity / Scheduling Issue

Volume attached to zone A but pod scheduled to zone B.

---

## 5. Init Container / Probe Failure

Pod created but never becomes Ready.

---

# Safe Recovery Without Data Loss

## Step 1: Never delete PVC blindly

Do **not** run:

```bash id="d4e5f6"
kubectl delete pvc ...
```

unless intended.

---

## Step 2: Check Pod + PVC + Events

```bash id="e5f6g7"
kubectl get pod,pvc,pv -n <ns>
kubectl describe pod <pod>
```

---

## Step 3: Verify Volume Attachment

Cloud console / CSI driver logs.

---

## Step 4: Force Reattach if Stuck

If stale mount:

* cordon/drain bad node
* reschedule pod

```bash id="f6g7h8"
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets
```

---

## Step 5: Preserve Data with Snapshot

Before risky actions, snapshot disk/storage.

---

## Step 6: If Pod Spec Corrupt

Restart StatefulSet pod only:

```bash id="g7h8i9"
kubectl delete pod db-0
```

StatefulSet recreates same pod with same PVC.

---

# Strong Interview Answer

> I first protect the PVC, inspect binding/events, validate CSI/storage health, zone scheduling, and probes. StatefulSet pod recreation should reuse the same PVC, so I avoid deleting storage resources unless backed up.

---

# 47. You see high CPU usage in one pod, but logs look clean. What next?

Logs being clean does not mean app is healthy.

---

# Step-by-Step Investigation

## 1. Confirm Usage

```bash id="h8i9j0"
kubectl top pod <pod> -n <ns>
```

---

## 2. Compare Requests/Limits

```bash id="i9j0k1"
kubectl describe pod <pod>
```

Maybe CPU limit too low causing throttling.

---

## 3. Exec into Pod

```bash id="j0k1l2"
kubectl exec -it <pod> -- sh
```

Run:

```bash id="k1l2m3"
top
ps aux
```

Find hot process/thread.

---

## 4. Check Traffic Spike

Maybe real load increased.

Inspect:

* request rate
* queue depth
* latency

Use Prometheus / Grafana Labs dashboards.

---

## 5. Garbage Collection / Runtime Issues

Java, Node.js, Python may spike CPU without logs.

---

## 6. Tight Loop / Retry Storm

App repeatedly retries failed dependency.

---

## 7. DNS / Network Retries

Silent loops can consume CPU.

---

# Fix Options

* Increase CPU requests/limits
* Scale replicas (HPA)
* Patch code inefficiency
* Reduce retry loops
* Add profiling

---

# 48. Your cluster autoscaler is not scaling up even though pods are in Pending state. What would you investigate?

Pending pods do not always mean autoscaler should scale.

---

# Step 1: Check Why Pod is Pending

```bash id="l2m3n4"
kubectl describe pod <pod>
```

Look for:

```text id="m3n4o5"
Insufficient cpu
Insufficient memory
node affinity mismatch
PVC pending
taint mismatch
```

Only resource shortage usually triggers scale-up.

---

# Step 2: Check Cluster Autoscaler Logs

```bash id="n4o5p6"
kubectl logs -n kube-system deploy/cluster-autoscaler
```

---

# Step 3: Node Group Limits

Max nodes may already be reached.

Cloud examples:

* Amazon Web Services ASG max size
* Microsoft VMSS max count

---

# Step 4: Unschedulable Constraints

If pod requires impossible node type:

* GPU node absent
* wrong zone
* huge memory request

Autoscaler may not help.

---

# Step 5: Taints / Labels

Pending pod needs node labels not present in scalable node group.

---

# Step 6: Cloud Capacity / Quotas

vCPU quota exhausted.

---

# Strong Interview Answer

> I verify whether Pending is due to resources or constraints, then inspect autoscaler logs, node group max size, taints, affinity, and cloud quota limits.

---

# 49. A network policy is blocking traffic between services in different namespaces. How would you design and debug the policy to allow only specific communication paths?

Use least-privilege access.

---

# Example Requirement

Allow:

```text id="o5p6q7"
frontend namespace -> api namespace on TCP 8080 only
```

Block everything else.

---

# Design Policy in Target Namespace

```yaml id="p6q7r8"
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: api
spec:
  podSelector:
    matchLabels:
      app: api
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend
    ports:
    - protocol: TCP
      port: 8080
  policyTypes:
  - Ingress
```

---

# Debug Steps

## 1. List Policies

```bash id="q7r8s9"
kubectl get networkpolicy -A
```

## 2. Describe Policy

```bash id="r8s9t0"
kubectl describe networkpolicy -n api allow-frontend
```

## 3. Verify Namespace Labels

```bash id="s9t0u1"
kubectl get ns --show-labels
```

## 4. Test Connectivity

```bash id="t0u1v2"
kubectl exec -it <frontend-pod> -- curl api-service.api:8080
```

## 5. Check Egress Policies Too

Ingress may allow, but source namespace egress may block.

---

# Best Practice

Need both:

* target ingress allow
* source egress allow (if egress restricted)

---

# 50. One of your microservices has to connect to an external database via a VPN inside the cluster. How would you architect this in Kubernetes with HA and security in mind?

This is a real enterprise hybrid architecture question.

---

# Recommended Architecture

```text id="u1v2w3"
App Pods
   |
Kubernetes Service
   |
Egress Layer / Firewall
   |
HA VPN Tunnel
   |
External Database
```

---

# Design Components

## 1. Highly Available VPN

Use two tunnels / redundant gateways.

Examples:

* Cisco VPN
* Palo Alto Networks firewall
* Cloud VPN gateways

---

## 2. Private Routing

Route DB subnet via VPN only.

No public internet path.

---

## 3. Kubernetes Network Security

Use NetworkPolicy:

Only microservice namespace/pods can reach DB IP:port.

---

## 4. Secrets Management

Store DB creds in:

* Kubernetes Secret (minimum)
* Better: HashiCorp Vault

---

## 5. HA Application Layer

Run multiple replicas:

```text id="v2w3x4"
replicas: 3
```

Use readiness probes.

---

## 6. Connection Pooling

Use sidecar/proxy like:

* PgBouncer for PostgreSQL
* JDBC pools

---

## 7. Monitoring

Track:

* VPN tunnel state
* DB latency
* Packet loss
* App retries

---

## 8. DNS Strategy

Use internal DNS / private hostname for DB.

---

# Failure Planning

If one tunnel fails:

* route to second tunnel
* pods continue serving

---

# Strong Interview Answer

> I’d design redundant VPN tunnels, private routing, restricted egress NetworkPolicies, secret-managed credentials, multi-replica services, connection pooling, and monitoring. This gives secure hybrid connectivity with high availability.

---

## 51. You notice the kubelet is constantly restarting on a particular node. What steps would you take to isolate the issue and ensure node stability?

kubelet restarting repeatedly usually indicates node-level instability, config corruption, resource exhaustion, or runtime/network issues.

---

# Step 1: Check Node Status

```bash id="a11b22"
kubectl get nodes
kubectl describe node <node-name>
```

Look for:

* `NotReady`
* MemoryPressure
* DiskPressure
* PIDPressure

---

# Step 2: Inspect kubelet Service Logs

```bash id="b22c33"
systemctl status kubelet
journalctl -u kubelet -f
```

Common errors:

```text id="c33d44"
certificate expired
container runtime unavailable
cni plugin not initialized
out of memory
config file invalid
```

---

# Step 3: Verify Container Runtime

Example runtimes:

* containerd
* CRI-O

```bash id="d44e55"
systemctl status containerd
crictl ps
```

If runtime is unhealthy, kubelet often restarts.

---

# Step 4: Check Disk / Memory

```bash id="e55f66"
df -h
free -m
top
```

Full disk can crash kubelet due to log/image pressure.

---

# Step 5: Check Certificates

```bash id="f66g77"
ls -l /var/lib/kubelet/pki
```

Expired node certs can cause repeated reconnect failures.

---

# Step 6: Check CNI Networking

```bash id="g77h88"
ls /etc/cni/net.d
journalctl -u kubelet | grep cni
```

Broken CNI config can block kubelet readiness.

---

# Stabilization Actions

## Safely isolate node:

```bash id="h88i99"
kubectl cordon <node-name>
kubectl drain <node-name> --ignore-daemonsets
```

## Then repair:

* restart runtime
* fix config
* free disk
* renew certs
* reboot if needed

---

# Strong Interview Answer

> I cordon/drain the node first to protect workloads, then inspect kubelet logs, runtime health, disk/memory pressure, certificates, and CNI configuration before returning node to service.

---

# 52. A critical pod in production gets evicted due to node pressure. How would you prevent this from happening again, and how do QoS classes play a role?

Evictions happen when node lacks:

* Memory
* Disk
* PID resources

Kubernetes removes pods to protect node stability.

---

# Immediate Recovery

```bash id="i99j00"
kubectl describe pod <pod>
kubectl describe node <node>
```

Look for:

```text id="j00k11"
Evicted: The node had condition MemoryPressure
```

---

# Prevention Strategy

## 1. Set Requests and Limits Properly

```yaml id="k11l22"
resources:
  requests:
    cpu: "500m"
    memory: "1Gi"
  limits:
    cpu: "1"
    memory: "2Gi"
```

---

## 2. Use QoS Classes

Kubernetes QoS decides eviction priority.

### Guaranteed (Best for critical pods)

Requests = Limits for CPU & Memory.

Least likely to be evicted.

### Burstable

Has requests, limits differ.

### BestEffort

No requests/limits. Evicted first.

---

# Example Guaranteed Pod

```yaml id="l22m33"
requests:
  cpu: "1"
  memory: "2Gi"
limits:
  cpu: "1"
  memory: "2Gi"
```

---

## 3. Use PriorityClass

Critical workloads scheduled/retained first.

```yaml id="m33n44"
priorityClassName: high-priority
```

---

## 4. Reserve Node Capacity

Use kubelet reservations:

* system-reserved
* kube-reserved

---

## 5. Autoscaling / Larger Nodes

Prevent pressure before eviction.

---

# Strong Interview Answer

> I’d move critical pods to Guaranteed QoS, assign PriorityClass, tune requests/limits, and increase node capacity or autoscaling so pressure never reaches eviction thresholds.

---

# 53. You need to deploy a service that requires TCP and UDP on the same port. How would you configure this in Kubernetes using Services and Ingress?

Example protocols:

* DNS uses TCP/UDP 53
* Game servers
* Media streaming

---

# Kubernetes Service Supports This

Create multiple ports with same port number but different protocol.

```yaml id="n44o55"
apiVersion: v1
kind: Service
metadata:
  name: dual-protocol
spec:
  selector:
    app: app1
  ports:
  - name: tcp-port
    port: 53
    targetPort: 53
    protocol: TCP
  - name: udp-port
    port: 53
    targetPort: 53
    protocol: UDP
  type: LoadBalancer
```

---

# Important Note on Ingress

Standard Kubernetes Ingress is mainly for:

* HTTP
* HTTPS

Not general UDP/TCP.

---

# For TCP/UDP Use

## Option 1: LoadBalancer Service

Best in cloud.

## Option 2: NodePort

Expose node ports.

## Option 3: Ingress Controller Extensions

Some controllers support TCP/UDP config maps:

* NGINX Ingress TCP/UDP streams
* Traefik

---

# Strong Interview Answer

> Native Ingress is HTTP-focused, so for same-port TCP/UDP I’d use a Service (LoadBalancer/NodePort), or controller-specific stream support.

---

# 54. An application upgrade caused downtime even though you had rolling updates configured. What advanced strategies would you apply to ensure zero-downtime deployments next time?

Rolling updates alone do not guarantee zero downtime.

Downtime often comes from:

* Bad readiness probes
* Too few replicas
* DB migrations
* Slow startup
* Connection draining issues

---

# Better Strategies

## 1. Blue-Green Deployment

Two environments:

```text id="o55p66"
Blue = current
Green = new version
```

Switch traffic only after validation.

---

## 2. Canary Deployment

Send small traffic first:

```text id="p66q77"
5% -> 25% -> 50% -> 100%
```

Use metrics gates.

Tools:

* Istio
* Argo Rollouts

---

## 3. Proper Readiness Probes

Only send traffic when truly ready.

---

## 4. Increase Replicas + Surge

```yaml id="q77r88"
maxUnavailable: 0
maxSurge: 1
```

Ensures no pod loss during rollout.

---

## 5. Graceful Shutdown

Use:

```yaml id="r88s99"
terminationGracePeriodSeconds: 60
```

and app SIGTERM handling.

---

## 6. Separate DB Migrations

Do schema-compatible migrations first.

---

## 7. Auto Rollback

```bash id="s99t00"
kubectl rollout undo deployment/myapp
```

Pipeline rollback on failed health checks.

---

# Strong Interview Answer

> I’d combine canary or blue-green deployments, strict readiness probes, maxUnavailable=0, graceful draining, backward-compatible DB migrations, and automated rollback gates.

---

# 55. Your service mesh sidecar (e.g., Istio Envoy) is consuming more resources than the app itself. How do you analyze and optimize this setup?

Istio sidecars like Envoy add networking, mTLS, retries, telemetry—but can be expensive.

---

# Step 1: Measure Consumption

```bash id="t00u11"
kubectl top pod <pod-name> --containers
```

Compare:

```text id="u11v22"
app-container
istio-proxy
```

---

# Step 2: Check Traffic Pattern

High RPS, retries, TLS handshakes, verbose telemetry can drive CPU.

Use metrics from:

* Prometheus
* Grafana Labs

---

# Step 3: Tune Proxy Resources

```yaml id="v22w33"
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

---

# Step 4: Reduce Unneeded Features

Disable where unnecessary:

* access logs
* excessive tracing
* redundant telemetry
* retries loops

---

# Step 5: Scope Sidecar Injection

Do not inject mesh into every namespace blindly.

Disable for low-value workloads:

```text id="w33x44"
batch jobs
cron jobs
internal utilities
```

---

# Step 6: Optimize Connections

Use keepalive / connection pooling to reduce TLS churn.

---

# Step 7: Consider Ambient / Sidecarless Modes

Where supported, newer mesh models reduce per-pod overhead.

---

# Strong Interview Answer

> I’d profile per-container usage, inspect traffic/retries/telemetry overhead, tune Envoy resources, reduce unnecessary sidecar injection, and optimize mesh policies. Many environments over-instrument proxies unnecessarily.

---

## 51. You notice the kubelet is constantly restarting on a particular node. What steps would you take to isolate the issue and ensure node stability?

kubelet restarting repeatedly usually indicates node-level instability, config corruption, resource exhaustion, or runtime/network issues.

---

# Step 1: Check Node Status

```bash id="a11b22"
kubectl get nodes
kubectl describe node <node-name>
```

Look for:

* `NotReady`
* MemoryPressure
* DiskPressure
* PIDPressure

---

# Step 2: Inspect kubelet Service Logs

```bash id="b22c33"
systemctl status kubelet
journalctl -u kubelet -f
```

Common errors:

```text id="c33d44"
certificate expired
container runtime unavailable
cni plugin not initialized
out of memory
config file invalid
```

---

# Step 3: Verify Container Runtime

Example runtimes:

* containerd
* CRI-O

```bash id="d44e55"
systemctl status containerd
crictl ps
```

If runtime is unhealthy, kubelet often restarts.

---

# Step 4: Check Disk / Memory

```bash id="e55f66"
df -h
free -m
top
```

Full disk can crash kubelet due to log/image pressure.

---

# Step 5: Check Certificates

```bash id="f66g77"
ls -l /var/lib/kubelet/pki
```

Expired node certs can cause repeated reconnect failures.

---

# Step 6: Check CNI Networking

```bash id="g77h88"
ls /etc/cni/net.d
journalctl -u kubelet | grep cni
```

Broken CNI config can block kubelet readiness.

---

# Stabilization Actions

## Safely isolate node:

```bash id="h88i99"
kubectl cordon <node-name>
kubectl drain <node-name> --ignore-daemonsets
```

## Then repair:

* restart runtime
* fix config
* free disk
* renew certs
* reboot if needed

---

# Strong Interview Answer

> I cordon/drain the node first to protect workloads, then inspect kubelet logs, runtime health, disk/memory pressure, certificates, and CNI configuration before returning node to service.

---

# 52. A critical pod in production gets evicted due to node pressure. How would you prevent this from happening again, and how do QoS classes play a role?

Evictions happen when node lacks:

* Memory
* Disk
* PID resources

Kubernetes removes pods to protect node stability.

---

# Immediate Recovery

```bash id="i99j00"
kubectl describe pod <pod>
kubectl describe node <node>
```

Look for:

```text id="j00k11"
Evicted: The node had condition MemoryPressure
```

---

# Prevention Strategy

## 1. Set Requests and Limits Properly

```yaml id="k11l22"
resources:
  requests:
    cpu: "500m"
    memory: "1Gi"
  limits:
    cpu: "1"
    memory: "2Gi"
```

---

## 2. Use QoS Classes

Kubernetes QoS decides eviction priority.

### Guaranteed (Best for critical pods)

Requests = Limits for CPU & Memory.

Least likely to be evicted.

### Burstable

Has requests, limits differ.

### BestEffort

No requests/limits. Evicted first.

---

# Example Guaranteed Pod

```yaml id="l22m33"
requests:
  cpu: "1"
  memory: "2Gi"
limits:
  cpu: "1"
  memory: "2Gi"
```

---

## 3. Use PriorityClass

Critical workloads scheduled/retained first.

```yaml id="m33n44"
priorityClassName: high-priority
```

---

## 4. Reserve Node Capacity

Use kubelet reservations:

* system-reserved
* kube-reserved

---

## 5. Autoscaling / Larger Nodes

Prevent pressure before eviction.

---

# Strong Interview Answer

> I’d move critical pods to Guaranteed QoS, assign PriorityClass, tune requests/limits, and increase node capacity or autoscaling so pressure never reaches eviction thresholds.

---

# 53. You need to deploy a service that requires TCP and UDP on the same port. How would you configure this in Kubernetes using Services and Ingress?

Example protocols:

* DNS uses TCP/UDP 53
* Game servers
* Media streaming

---

# Kubernetes Service Supports This

Create multiple ports with same port number but different protocol.

```yaml id="n44o55"
apiVersion: v1
kind: Service
metadata:
  name: dual-protocol
spec:
  selector:
    app: app1
  ports:
  - name: tcp-port
    port: 53
    targetPort: 53
    protocol: TCP
  - name: udp-port
    port: 53
    targetPort: 53
    protocol: UDP
  type: LoadBalancer
```

---

# Important Note on Ingress

Standard Kubernetes Ingress is mainly for:

* HTTP
* HTTPS

Not general UDP/TCP.

---

# For TCP/UDP Use

## Option 1: LoadBalancer Service

Best in cloud.

## Option 2: NodePort

Expose node ports.

## Option 3: Ingress Controller Extensions

Some controllers support TCP/UDP config maps:

* NGINX Ingress TCP/UDP streams
* Traefik

---

# Strong Interview Answer

> Native Ingress is HTTP-focused, so for same-port TCP/UDP I’d use a Service (LoadBalancer/NodePort), or controller-specific stream support.

---

# 54. An application upgrade caused downtime even though you had rolling updates configured. What advanced strategies would you apply to ensure zero-downtime deployments next time?

Rolling updates alone do not guarantee zero downtime.

Downtime often comes from:

* Bad readiness probes
* Too few replicas
* DB migrations
* Slow startup
* Connection draining issues

---

# Better Strategies

## 1. Blue-Green Deployment

Two environments:

```text id="o55p66"
Blue = current
Green = new version
```

Switch traffic only after validation.

---

## 2. Canary Deployment

Send small traffic first:

```text id="p66q77"
5% -> 25% -> 50% -> 100%
```

Use metrics gates.

Tools:

* Istio
* Argo Rollouts

---

## 3. Proper Readiness Probes

Only send traffic when truly ready.

---

## 4. Increase Replicas + Surge

```yaml id="q77r88"
maxUnavailable: 0
maxSurge: 1
```

Ensures no pod loss during rollout.

---

## 5. Graceful Shutdown

Use:

```yaml id="r88s99"
terminationGracePeriodSeconds: 60
```

and app SIGTERM handling.

---

## 6. Separate DB Migrations

Do schema-compatible migrations first.

---

## 7. Auto Rollback

```bash id="s99t00"
kubectl rollout undo deployment/myapp
```

Pipeline rollback on failed health checks.

---

# Strong Interview Answer

> I’d combine canary or blue-green deployments, strict readiness probes, maxUnavailable=0, graceful draining, backward-compatible DB migrations, and automated rollback gates.

---

# 55. Your service mesh sidecar (e.g., Istio Envoy) is consuming more resources than the app itself. How do you analyze and optimize this setup?

Istio sidecars like Envoy add networking, mTLS, retries, telemetry—but can be expensive.

---

# Step 1: Measure Consumption

```bash id="t00u11"
kubectl top pod <pod-name> --containers
```

Compare:

```text id="u11v22"
app-container
istio-proxy
```

---

# Step 2: Check Traffic Pattern

High RPS, retries, TLS handshakes, verbose telemetry can drive CPU.

Use metrics from:

* Prometheus
* Grafana Labs

---

# Step 3: Tune Proxy Resources

```yaml id="v22w33"
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

---

# Step 4: Reduce Unneeded Features

Disable where unnecessary:

* access logs
* excessive tracing
* redundant telemetry
* retries loops

---

# Step 5: Scope Sidecar Injection

Do not inject mesh into every namespace blindly.

Disable for low-value workloads:

```text id="w33x44"
batch jobs
cron jobs
internal utilities
```

---

# Step 6: Optimize Connections

Use keepalive / connection pooling to reduce TLS churn.

---

# Step 7: Consider Ambient / Sidecarless Modes

Where supported, newer mesh models reduce per-pod overhead.

---

# Strong Interview Answer

> I’d profile per-container usage, inspect traffic/retries/telemetry overhead, tune Envoy resources, reduce unnecessary sidecar injection, and optimize mesh policies. Many environments over-instrument proxies unnecessarily.

---

## 56. You need to create a Kubernetes operator to automate complex application lifecycle events. How do you design the CRD and controller loop logic?

A Kubernetes Operator extends Kubernetes using:

* **CRD** = Custom Resource Definition
* **Controller** = Reconciliation loop

Used for databases, middleware, backup systems, SaaS platforms, etc.

---

# Example Use Case

Automate lifecycle of:

* App install
* Upgrade
* Backup
* Scaling
* Failover
* Certificate rotation

---

# Step 1: Design the CRD

Create a custom object representing desired state.

Example:

```yaml id="a101b202"
apiVersion: apps.company.com/v1
kind: MyApp
metadata:
  name: prod-app
spec:
  version: "2.1.0"
  replicas: 3
  storageSize: 100Gi
  backupEnabled: true
  upgradeStrategy: rolling
status:
  phase: Running
  readyReplicas: 3
```

---

# CRD Design Principles

## spec = desired state

What user wants.

## status = observed state

What operator reports.

## Validation Schema

Use OpenAPI validations:

* enum values
* required fields
* ranges

---

# Step 2: Controller Reconcile Loop

Watch:

* CR changes
* Pods
* StatefulSets
* PVCs
* Secrets

Logic:

```text id="b202c303"
Read CR
Compare desired vs actual
Take action
Update status
Requeue if needed
```

---

# Example Actions

If replicas mismatch:

* scale StatefulSet

If version mismatch:

* rolling upgrade

If cert expiring:

* rotate secret

If backup time reached:

* run Job

---

# Step 3: Idempotency

Controller must safely rerun multiple times.

Never assume one-time execution.

---

# Step 4: Finalizers

Before delete:

* backup data
* detach resources
* clean DNS

---

# Tools Commonly Used

* Red Hat Operator SDK
* Kubebuilder
* controller-runtime

---

# Strong Interview Answer

> I’d model desired state in the CRD spec, observed state in status, then implement an idempotent reconcile loop that continuously compares actual vs desired state and performs safe lifecycle automation.

---

# 57. Multiple nodes are showing high disk IO usage due to container logs. What Kubernetes features or practices can you apply to avoid this scenario?

Container stdout/stderr logs can overwhelm node disks.

---

# Root Causes

* Verbose debug logs
* Large JSON logs
* No rotation
* Chatty sidecars
* Many crashing pods

---

# Best Practices

## 1. Enable Log Rotation

For kubelet:

```text id="c303d404"
containerLogMaxSize
containerLogMaxFiles
```

Example:

```text id="d404e505"
10Mi x 5 files
```

---

## 2. Centralized Logging

Ship logs off-node using DaemonSet agents:

* Elastic Filebeat
* Fluent Bit
* Datadog Agent

Then keep local logs minimal.

---

## 3. Reduce Log Verbosity

Set production log level:

```text id="e505f606"
INFO instead of DEBUG
```

---

## 4. Fix CrashLooping Pods

Repeated restarts generate huge logs.

---

## 5. Separate Application Logs

Write structured logs efficiently.

Avoid excessive stack traces.

---

## 6. Use Ephemeral Storage Limits

```yaml id="f606g707"
resources:
  limits:
    ephemeral-storage: "2Gi"
```

---

## 7. Monitor Disk IO

Use:

* Prometheus
* node exporter

---

# Strong Interview Answer

> I’d combine log rotation, centralized logging, lower verbosity, ephemeral storage controls, and remediation of noisy pods to reduce node disk IO.

---

# 58. Your Kubernetes cluster's etcd performance is degrading. What are the root causes and how do you ensure etcd high availability and tuning?

etcd is the cluster source of truth. Poor performance impacts API responsiveness.

---

# Root Causes

## 1. Slow Disk Latency

Most common issue.

ETCD needs fast SSD / low-latency storage.

---

## 2. Large Database Size

Too many objects:

* events
* secrets
* CRDs
* stale resources

---

## 3. Frequent Writes

Controllers or operators spamming updates.

---

## 4. Network Latency Between Members

Especially across regions.

---

## 5. Resource Starvation

CPU / RAM too low.

---

# Checks

```bash id="g707h808"
etcdctl endpoint health
etcdctl endpoint status --write-out=table
```

---

# Tuning / HA Design

## 1. Odd Number Cluster

Use:

```text id="h808i909"
3 members (standard)
5 members (large)
```

---

## 2. Same Low-Latency Region

Avoid cross-region etcd members.

---

## 3. SSD Storage

Dedicated disks.

---

## 4. Defragment Regularly

```bash id="i909j010"
etcdctl defrag
```

---

## 5. Compaction

Remove old revisions.

---

## 6. Backup Snapshots

```bash id="j010k111"
etcdctl snapshot save backup.db
```

---

## 7. Resource Reservations

Guarantee CPU/memory.

---

# Strong Interview Answer

> etcd issues are usually storage latency, object bloat, or network latency. I keep etcd regional, SSD-backed, odd-membered, compacted, defragmented, and regularly backed up.

---

# 59. You want to enforce that all images used in the cluster must come from a trusted internal registry. How do you implement this at the policy level?

Goal:

Only allow images from:

```text id="k111l212"
registry.company.local/*
```

---

# Best Methods

## 1. Admission Policy (Recommended)

Use:

* Kyverno
* Open Policy Agent Gatekeeper
* ValidatingAdmissionPolicy (native newer Kubernetes)

---

# Example Kyverno Policy

```yaml id="l212m313"
match:
  resources:
    kinds:
    - Pod
validate:
  message: "Images must come from internal registry"
  pattern:
    spec:
      containers:
      - image: "registry.company.local/*"
```

---

# 2. Restrict Pull Secrets

Only internal registry credentials.

---

# 3. Image Signing

Use:

* Cosign
* Notary

Only signed trusted images.

---

# 4. CI/CD Controls

Mirror approved external images internally.

---

# 5. Audit Existing Violations

```bash id="m313n414"
kubectl get pods -A -o jsonpath=...
```

---

# Strong Interview Answer

> I’d enforce registry allowlists using admission policies like Kyverno or Gatekeeper, combined with signed images and CI pipelines that mirror approved images internally.

---

# 60. You're managing multi-region deployments using a single Kubernetes control plane. What architectural considerations must you address to avoid cross-region latency and single points of failure?

Single control plane across regions is risky.

---

# Major Concerns

## 1. API Server Latency

Remote nodes constantly talk to control plane.

High latency affects:

* kubelet heartbeats
* scheduling
* controllers

---

## 2. ETCD Sensitivity

etcd performs poorly across high-latency WAN links.

---

## 3. Control Plane SPOF

If region hosting control plane fails, all regions impacted.

---

# Better Architecture

## Prefer Multi-Cluster Per Region

```text id="n414o515"
Region A = cluster A
Region B = cluster B
Region C = cluster C
```

Then federate apps / GitOps.

---

# If Single Control Plane Must Exist

## 1. Highly Available Control Plane

3+ control plane nodes in resilient zone set.

## 2. Keep etcd Localized

Do not stretch etcd globally.

## 3. Regional Worker Pools Carefully Sized

Expect slower reconciliation remotely.

## 4. Use Global Traffic Manager

Examples:

* Cloudflare
* DNS failover
* GSLB

---

## 5. Disaster Recovery Plan

Backups + rebuild cluster in second region.

---

## 6. Workload Placement

Latency-sensitive apps run close to users.

---

# Strong Interview Answer

> A single cross-region control plane introduces latency and blast-radius risk. Best practice is separate regional clusters with GitOps federation and global traffic routing rather than one stretched control plane.

---

## 61. During peak traffic, your ingress controller fails to route requests efficiently. How would you diagnose and scale ingress resources effectively under heavy load?

Ingress bottlenecks under load are usually caused by CPU saturation, connection limits, insufficient replicas, backend latency, or misconfiguration.

---

# Step 1: Identify Where the Bottleneck Is

Check:

```bash id="a611b722"
kubectl top pod -n ingress-nginx
kubectl get pods -n ingress-nginx
```

Look for:

* High CPU
* High memory
* Restarts
* OOMKilled

---

# Step 2: Inspect Ingress Controller Metrics

For NGINX / Traefik / HAProxy check:

* Requests/sec
* 4xx / 5xx rates
* Active connections
* Latency
* Upstream errors

Use:

* Prometheus
* Grafana Labs

---

# Step 3: Check Backend Health

Ingress may be healthy while backend services fail.

```bash id="b722c833"
kubectl get endpoints -A
kubectl get pods -A
```

If no healthy endpoints → 503s.

---

# Step 4: Scale Ingress Horizontally

Increase replicas:

```bash id="c833d944"
kubectl scale deployment ingress-nginx-controller \
-n ingress-nginx --replicas=5
```

---

# Step 5: Enable HPA

```yaml id="d944e055"
minReplicas: 3
maxReplicas: 10
targetCPUUtilizationPercentage: 70
```

---

# Step 6: Tune Ingress Settings

Examples:

* worker processes
* keepalive
* max connections
* proxy timeouts
* buffering

---

# Step 7: Multi-Zone / HA Design

Spread replicas across nodes/zones.

---

# Strong Interview Answer

> I first isolate whether ingress or backend is failing, then scale ingress replicas, enable HPA, tune connection settings, and distribute controllers across zones.

---

# 62. How do you implement network policies to restrict pod-to-pod communication in Kubernetes?

NetworkPolicy controls which pods can talk to each other.

Default behavior in many clusters is allow-all until policies are applied.

---

# Step 1: Default Deny

Start by denying traffic.

```yaml id="e055f166"
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

---

# Step 2: Allow Specific App Traffic

Example:

frontend pods can call api pods on 8080.

```yaml id="f166g277"
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: api
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

---

# Step 3: Restrict Egress Too

Allow only DNS / DB / APIs.

---

# Requirements

Your CNI must support policies:

* Calico
* Cilium
* Weave Net (legacy)

---

# Best Practices

* Default deny first
* Allow least privilege
* Separate namespaces
* Test connectivity after change

---

# 63. A critical production Kubernetes cluster is experiencing multiple issues. Pods are stuck in ImagePullBackOff, some pods are being evicted, and users are reporting 503 errors from the application. What troubleshooting process will you follow, and how can to avoid this in the future?

This is a multi-symptom incident. Prioritize user impact first.

---

# Immediate Priority Order

```text id="g277h388"
1. Restore service
2. Identify common root causes
3. Stabilize cluster
4. Prevent recurrence
```

---

# Step 1: Investigate 503 Errors

503 often means no healthy backend endpoints.

```bash id="h388i499"
kubectl get svc -A
kubectl get endpoints -A
kubectl get pods -A
```

Check if pods are Ready.

---

# Step 2: Investigate ImagePullBackOff

```bash id="i499j500"
kubectl describe pod <pod>
```

Look for:

* Wrong image tag
* Registry outage
* Auth failure
* DNS issue

Fix:

* Correct image
* Restore registry creds
* Retry rollout

---

# Step 3: Investigate Evictions

```bash id="j500k611"
kubectl describe node <node>
kubectl top nodes
```

Look for:

* MemoryPressure
* DiskPressure

---

# Step 4: Stabilize Capacity

* Scale node pool
* Drain unhealthy nodes
* Reduce noisy workloads

---

# Step 5: Check Ingress / Load Balancer

If endpoints empty, ingress returns 503.

---

# Prevent Future Issues

## ImagePullBackOff

* Use internal registry mirror
* Image pre-pull
* CI tag validation

## Evictions

* Proper requests/limits
* HPA + cluster autoscaler
* Guaranteed QoS for critical apps

## 503 Errors

* Readiness probes
* PDB
* Multi-replica deployments
* Alerting

---

# Strong Interview Answer

> I’d triage 503 first by checking endpoints, then fix image pull failures, then resolve node pressure causing evictions. Most likely these symptoms are connected through capacity or registry failures.

---

# 64. What is Pod Disruption Budget (PDB) in K8s?

Pod Disruption Budget protects applications during **voluntary disruptions** such as:

* Node drain
* Upgrades
* Maintenance
* Cluster autoscaler scale-down

---

# Example

Deployment has 3 replicas.

PDB:

```yaml id="k611l722"
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web
```

Meaning:

At least 2 pods must remain available.

---

# Options

## minAvailable

Minimum running pods.

## maxUnavailable

Maximum pods allowed down.

---

# Important Note

PDB does **not** stop involuntary failures like node crash.

---

# Use Cases

* APIs
* Databases
* Critical microservices

---

# 65. How does DNS resolution work inside a pod?

Kubernetes pods use cluster DNS, usually CoreDNS.

---

# Flow

Inside pod:

```text id="l722m833"
App requests: myservice
```

Resolver checks:

```text id="m833n944"
myservice.<namespace>.svc.cluster.local
```

Then CoreDNS returns Service ClusterIP.

---

# Example

Pod in namespace `dev` requests:

```text id="n944o055"
api
```

Resolved as:

```text id="o055p166"
api.dev.svc.cluster.local
```

---

# Components Involved

## 1. /etc/resolv.conf inside Pod

Contains DNS server IP (CoreDNS service IP) and search domains.

Check:

```bash id="p166q277"
kubectl exec -it <pod> -- cat /etc/resolv.conf
```

---

## 2. CoreDNS

Maps:

* Services → ClusterIP
* Headless Services → Pod IPs

---

## 3. kubelet

Injects DNS config into pod.

---

# Troubleshooting Commands

```bash id="q277r388"
kubectl exec -it <pod> -- nslookup kubernetes.default
kubectl get pods -n kube-system
kubectl logs -n kube-system deploy/coredns
```

---

# Common Failures

* CoreDNS down
* NetworkPolicy blocking UDP/TCP 53
* Wrong resolv.conf
* High DNS latency

---

# Strong Interview Answer

> Pods use CoreDNS through resolv.conf. Service names resolve to ClusterIP, and FQDNs follow service.namespace.svc.cluster.local.

---

## 66. What does the Kubernetes controller manager do during a Deployment?

kube-controller-manager runs multiple controllers that continuously reconcile desired state with actual state.

During a Deployment rollout, it plays a central role.

---

# What Happens During Deployment

You apply:

```bash id="a661b772"
kubectl apply -f deploy.yaml
```

Desired state:

```text id="b772c883"
replicas: 5
image: nginx:v2
```

---

# Controller Manager Actions

## 1. Deployment Controller

Detects new Deployment spec.

Creates / updates a ReplicaSet.

---

## 2. ReplicaSet Controller

Ensures required number of pods exist.

```text id="c883d994"
desired = 5
actual = 3
=> create 2 more pods
```

---

## 3. Rolling Update Logic

Old ReplicaSet scaled down gradually, new ReplicaSet scaled up according to:

```yaml id="d994e105"
maxSurge
maxUnavailable
```

---

## 4. Node Controller

Monitors node health during rollout.

If node unhealthy, pods recreated elsewhere.

---

## 5. Endpoint / Service Controllers

Ready pods added to service endpoints.

---

# In Short

> Controller manager ensures Deployments continuously converge to the desired state.

---

# 67. What happens when a node with local emptyDir volumes gets scaled down?

emptyDir is **node-local ephemeral storage** created when pod starts.

If node is scaled down / terminated:

```text id="e105f216"
Node removed -> Pod removed -> emptyDir data lost
```

---

# Important Behavior

## emptyDir lifecycle = Pod lifecycle

Not persistent.

Data survives container restart inside same pod, but not pod/node deletion.

---

# During Scale Down

If cluster autoscaler removes node:

1. Pod evicted/rescheduled
2. New pod starts on another node
3. New emptyDir is empty

---

# Risks

* Temp uploads lost
* Cache lost
* Session files lost
* Processing jobs interrupted

---

# Best Practices

Use emptyDir only for:

* temp files
* cache
* scratch space

Use PVC for persistent data.

---

# Interview Answer

> emptyDir is ephemeral. If node scales down, pod is rescheduled and data is lost.

---

# 68. After a deployment, 30% of users report slow response times. No errors. No logs. No spikes in CPU/memory. → How do you triage? Where do you even start?

This is a senior-level troubleshooting question.

No obvious errors means likely partial-path issue.

---

# First Principle: Scope the 30%

Ask:

```text id="f216g327"
Which users?
Which region?
Which browser?
Which endpoint?
Only new pods?
Only one AZ?
```

30% often means subset of traffic path affected.

---

# Triage Flow

## 1. Compare Old vs New Pods

Maybe some pods unhealthy but still Ready.

```bash id="g327h438"
kubectl get pods -o wide
```

Check latency per pod.

---

## 2. Check Load Balancer Distribution

Maybe one node / zone is slow.

30% may equal one of three nodes.

---

## 3. Check Readiness Probe Quality

Pod marked Ready too early.

App accepts traffic before warm-up.

---

## 4. Check DB / External Dependency Latency

No CPU spike in app, but waiting on DB/network.

Use tracing:

* Jaeger
* Datadog APM

---

## 5. DNS / Connection Reuse Issues

Slow DNS, socket exhaustion, TLS renegotiation.

---

## 6. Recent Config Change?

Feature flag, sidecar policy, new timeout.

---

## 7. Compare Region / Node / Pod Labels

```bash id="h438i549"
kubectl get pods -L topology.kubernetes.io/zone
```

---

# Immediate Mitigation

* Roll back deployment
* Drain suspicious node
* Remove slow pods from service

---

# Strong Interview Answer

> I’d segment the affected 30% by pod, node, zone, endpoint, and dependency path. Partial slowness usually indicates one bad subset of traffic rather than global resource exhaustion.

---

# 69. How do you enforce runtime security in Kubernetes? PSP? OPA? Kyverno? AppArmor? Or are you still hoping RBAC will protect your pods?

RBAC only controls **API permissions**, not runtime pod behavior.

Need layered security.

---

# Modern Runtime Security Stack

## 1. Pod Security Standards (PSP Replaced)

Pod Security Admission replaced deprecated PSP.

Modes:

* privileged
* baseline
* restricted

Apply per namespace.

---

## 2. Admission Policies

Use:

* Open Policy Agent Gatekeeper
* Kyverno

Enforce:

* no privileged containers
* no latest tags
* readOnlyRootFilesystem
* trusted registries

---

## 3. Linux Runtime Controls

### AppArmor

Restrict syscalls/files.

### seccomp

Reduce syscall surface.

### SELinux (where supported)

---

## 4. Image Security

* signed images
* vulnerability scanning

---

## 5. Network Security

Use NetworkPolicy

---

## 6. Secrets Security

Use:

* KMS encryption
* HashiCorp Vault

---

# Strong Interview Answer

> RBAC protects the API, not the container runtime. Real security combines Pod Security Admission, Kyverno/OPA, seccomp/AppArmor, image trust, and network policies.

---

# 70. HPA vs VPA vs Karpenter — when do you NOT use each?

## HPA = Horizontal Pod Autoscaler

Scales pods.

### Do NOT use when:

* app is stateful and cannot scale out
* startup takes too long
* traffic bursts faster than pod startup
* bottleneck is DB, not app tier

---

## VPA = Vertical Pod Autoscaler

Adjusts CPU/RAM.

### Do NOT use when:

* low-latency apps can’t tolerate pod restarts
* large fleets needing rapid scale-out
* conflicts with aggressive HPA on same CPU metric

---

## Karpenter

Amazon Web Services node autoscaler/provisioner for Kubernetes.

Adds right-sized nodes dynamically.

### Do NOT use when:

* not running in compatible environment
* strict manually-managed nodes required
* governance forbids dynamic instance classes
* workload sizing data is poor

---

# When to Use Together

```text id="i549j650"
HPA = scale pods
VPA = right-size requests
Karpenter = add nodes
```

---

# Bonus: How would you simulate HPA behavior in staging without real traffic?

## Option 1: CPU Load Test

```bash id="j650k761"
kubectl exec -it <pod> -- stress --cpu 2
```

---

## Option 2: Load Generator

Use:

* hey
* wrk
* k6

Generate HTTP traffic.

---

## Option 3: Custom Metrics Replay

Replay staging metrics to Prometheus adapter.

---

## Option 4: Lower Thresholds Temporarily

Example:

```yaml id="k761l872"
targetCPUUtilizationPercentage: 20
```

So small load triggers scale.

---

# Strong Interview Answer

> HPA scales pods, VPA resizes pods, Karpenter adds nodes. I avoid each when they don’t match workload behavior. In staging, I simulate HPA using synthetic CPU or HTTP load generators.

---

## 71. Tell me about the last Kubernetes outage you debugged.

This is a behavioral + technical interview question. They want:

* Real incident ownership
* Troubleshooting method
* Communication under pressure
* Preventive actions

If you don’t have a direct example, use a realistic production scenario answer.

---

# Strong Sample Answer (Senior Level)

> The last Kubernetes outage I debugged involved users receiving intermittent 503 errors after a deployment. Around 30% of requests were failing while the application pods looked Running.
>
> I started by checking ingress metrics and service endpoints. I found several newly deployed pods marked Running but failing readiness checks intermittently, so they were repeatedly entering and leaving endpoints.
>
> Then I reviewed pod events and saw slow startup due to delayed database connection initialization. Readiness probes were too aggressive and started checking before the app was fully ready.
>
> Immediate mitigation was to pause rollout and scale stable previous replicas up. Then we rolled back the deployment.
>
> Permanent fix included:
>
> * Increased `initialDelaySeconds`
> * Added proper startup probe
> * Improved DB connection retry logic
> * Added canary deployment stage
> * Added alerting on endpoint flapping
>
> Total impact was around 12 minutes, and after changes we avoided recurrence.

---

# What Interviewers Like

* Structured triage
* Root cause, not guessing
* Fast mitigation
* Long-term prevention

---

# 72. How do you expose a Kubernetes application to external traffic?

Several production-grade methods exist depending on use case.

---

# 1. Service Type LoadBalancer (Best in Cloud)

Creates cloud load balancer automatically.

```yaml id="a722b833"
spec:
  type: LoadBalancer
```

Used in:

* Microsoft AKS
* Amazon Web Services EKS
* Google GKE

---

# 2. Ingress (Best for HTTP/HTTPS)

Single IP with host/path routing.

```text id="b833c944"
app.company.com
api.company.com
```

Needs controller:

* NGINX Ingress
* Traefik

---

# 3. NodePort

Exposes on node IP + port.

```text id="c944d055"
<NodeIP>:30080
```

Good for labs/on-prem.

---

# 4. Port Forward

Temporary admin/testing access.

```bash id="d055e166"
kubectl port-forward svc/myapp 8080:80
```

---

# 5. On-Prem Production Pattern

```text id="e166f277"
Internet -> Firewall -> Reverse Proxy / LB -> Ingress -> Service -> Pods
```

---

# Strong Interview Answer

> For production web apps I prefer Ingress or LoadBalancer. NodePort is more common for testing or on-prem edge integration.

---

# 73. A pod is in CrashLoopBackOff. How do you troubleshoot it?

CrashLoopBackOff means container starts, crashes, restarts repeatedly.

---

# Step-by-Step Troubleshooting

## 1. Describe Pod

```bash id="f277g388"
kubectl describe pod <pod-name>
```

Check:

* Events
* Exit code
* OOMKilled
* Probe failures

---

## 2. View Logs

```bash id="g388h499"
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
```

`--previous` is critical if container restarts quickly.

---

## 3. Check Common Causes

### Wrong Command / Entrypoint

Container exits immediately.

### Missing Config

Bad env vars, ConfigMap, Secret.

### Dependency Failure

DB/API unreachable.

### OOMKilled

Memory too low.

### Probe Failure

Bad liveness probe causing restarts.

---

## 4. Debug Interactively

```bash id="h499i500"
kubectl run debug --rm -it --image=<image> -- sh
```

---

# Strong Interview Answer

> I check describe output, previous logs, exit codes, probes, resources, and dependencies. Most CrashLoopBackOff cases are config errors, OOM kills, or bad health checks.

---

# 74. What are the types of services in Kubernetes?

Service gives stable access to pods.

---

# 1. ClusterIP (Default)

Internal-only service.

```yaml id="i500j611"
type: ClusterIP
```

Use:

* app to DB
* microservice communication

---

# 2. NodePort

Exposes service on every node.

```text id="j611k722"
NodeIP:30000-32767
```

---

# 3. LoadBalancer

Creates external load balancer in cloud.

```yaml id="k722l833"
type: LoadBalancer
```

---

# 4. ExternalName

Maps to external DNS name.

```text id="l833m944"
db.company.com
```

---

# 5. Headless Service

No ClusterIP, direct pod DNS.

(covered below)

---

# Strong Interview Answer

> Main service types are ClusterIP, NodePort, LoadBalancer, and ExternalName. Headless services are used for direct pod discovery.

---

# 75. What is a headless service in Kubernetes?

Headless Service is a Service with:

```yaml id="m944n055"
clusterIP: None
```

No virtual IP is assigned.

---

# What It Does

Instead of one service IP, DNS returns pod IPs directly.

```text id="n055o166"
db-0.db
db-1.db
db-2.db
```

---

# Common Use Cases

## StatefulSets

Databases needing stable identities:

* MongoDB
* Apache Kafka
* Redis clusters

---

## Custom Client Load Balancing

App receives all pod IPs.

---

# Example

```yaml id="o166p277"
apiVersion: v1
kind: Service
metadata:
  name: db
spec:
  clusterIP: None
  selector:
    app: db
```

---

# DNS Result

```text id="p277q388"
db.default.svc.cluster.local
-> returns pod IPs
```

---

# Strong Interview Answer

> Headless service removes ClusterIP and returns pod DNS records directly. It is commonly used with StatefulSets for stable pod identity.

---

## 76. In a Kubernetes cluster, one ReplicaSet is not working. How do you debug it?

ReplicaSet ensures a specified number of identical pod replicas are running. If it is “not working,” it usually means pods are not being created, not becoming Ready, or replica count is mismatched.

---

# Step-by-Step Debugging

## 1. Check ReplicaSet Status

```bash id="a761b872"
kubectl get rs -A
kubectl describe rs <replicaset-name> -n <namespace>
```

Look for:

* Desired vs Current vs Ready replicas
* Events

Example:

```text id="b872c983"
Desired: 3
Current: 1
Ready: 0
```

---

## 2. Check Pods Created by ReplicaSet

```bash id="c983d094"
kubectl get pods -n <namespace> -l app=<label>
```

If no pods exist:

* selector mismatch
* admission policy block
* quota issue

---

## 3. Describe Pod

```bash id="d094e105"
kubectl describe pod <pod-name> -n <namespace>
```

Look for:

* Pending
* CrashLoopBackOff
* ImagePullBackOff
* FailedMount
* Probe failures

---

## 4. Verify Labels / Selectors

```bash id="e105f216"
kubectl get rs <name> -o yaml
kubectl get pods --show-labels
```

ReplicaSet selector must match pod template labels.

---

## 5. Check Resources / Scheduling

```bash id="f216g327"
kubectl describe pod <pod>
kubectl get nodes
```

Possible causes:

* Insufficient CPU/RAM
* Taints
* Affinity mismatch

---

## 6. Check Namespace Quotas / Limits

```bash id="g327h438"
kubectl get quota -n <namespace>
kubectl get limitrange -n <namespace>
```

---

# Strong Interview Answer

> I verify desired/current replicas, inspect events, confirm selectors, then check pod states like Pending or CrashLoopBackOff. Most ReplicaSet issues are scheduling, image, or label mismatches.

---

# 77. Difference between Deployment and StatefulSet?

Both manage pods, but for different workload types.

---

# Deployment

Used for **stateless applications**.

Examples:

* Web apps
* APIs
* Frontend services

Features:

* Easy rolling updates
* Random pod names
* Pods interchangeable

Example:

```text id="h438i549"
web-6f7d8c9f4-x2abc
```

---

# StatefulSet

Used for **stateful applications** needing identity/storage.

Examples:

* MongoDB
* Apache Kafka
* Redis cluster

Features:

* Stable pod names
* Ordered deployment
* Persistent per-pod storage

Example:

```text id="i549j650"
db-0
db-1
db-2
```

---

# Comparison Table

| Feature       | Deployment       | StatefulSet          |
| ------------- | ---------------- | -------------------- |
| Workload Type | Stateless        | Stateful             |
| Pod Identity  | Dynamic          | Stable               |
| Storage       | Shared/Ephemeral | Dedicated Persistent |
| Scaling Order | Any              | Ordered              |
| Use Case      | APIs/Web         | DB/Queues            |

---

# Strong Interview Answer

> Deployment is for stateless scalable apps; StatefulSet is for apps needing stable identity and persistent storage.

---

# 78. What is a DaemonSet used for?

DaemonSet ensures **one pod runs on every node** (or selected nodes).

---

# Typical Use Cases

## 1. Logging Agents

Collect node/container logs.

Examples:

* Elastic Filebeat
* Fluent Bit

---

## 2. Monitoring Agents

Per-node metrics.

Examples:

* Prometheus Node Exporter
* Datadog Agent

---

## 3. Security Agents

Runtime scanning / compliance.

---

## 4. Networking Components

CNI plugins often use DaemonSets.

---

# Behavior

If new node joins cluster:

```text id="j650k761"
DaemonSet automatically schedules pod there
```

If node removed:

Pod removed with node.

---

# Strong Interview Answer

> DaemonSet is used for node-level agents such as logging, monitoring, networking, or security—one pod per node.

---

# 79. How does a Service in Kubernetes work?

Service provides stable access to pods whose IPs change.

Pods are ephemeral, so Service gives:

* Stable virtual IP
* DNS name
* Load balancing across pods

---

# How It Works

Suppose 3 pods:

```text id="k761l872"
app-1 10.1.1.2
app-2 10.1.1.3
app-3 10.1.1.4
```

Service selects them using labels:

```yaml id="l872m983"
selector:
  app: myapp
```

Then creates:

```text id="m983n094"
myapp-service
ClusterIP: 10.96.0.20
```

Traffic to service is distributed to pods.

---

# DNS Access

Inside cluster:

```text id="n094o105"
http://myapp-service
```

---

# Components Involved

* kube-proxy (iptables/ipvs rules)
* EndpointSlices
* CoreDNS

---

# Strong Interview Answer

> A Service provides stable networking and load balances traffic to matching pods using labels.

---

# 80. What is a ConfigMap vs Secret?

Both externalize configuration from container images.

---

# ConfigMap

Stores **non-sensitive** configuration.

Examples:

* App mode
* URLs
* Feature flags
* Port numbers

```yaml id="o105p216"
data:
  APP_ENV: production
  PORT: "8080"
```

---

# Secret

Stores **sensitive** values.

Examples:

* Passwords
* API keys
* Tokens
* Certificates

```yaml id="p216q327"
data:
  password: cGFzc3dvcmQ=
```

(Base64 encoded, not encryption by itself)

---

# Usage

Both can be mounted as:

* Environment variables
* Files/volumes

---

# Comparison Table

| Feature                | ConfigMap  | Secret      |
| ---------------------- | ---------- | ----------- |
| Sensitive Data         | No         | Yes         |
| Typical Use            | App config | Credentials |
| Encoding               | Plain text | Base64      |
| Should Encrypt at Rest | Optional   | Recommended |

---

# Best Practice

Use:

* Secrets + KMS encryption
* External secret managers like HashiCorp Vault

---

# Strong Interview Answer

> ConfigMap stores non-sensitive config, while Secret stores credentials or confidential data. Both decouple config from images.

---

## 81. What are taints and tolerations?

Taints and Tolerations are used to control **which pods can run on which nodes**.

Think of it as:

* **Taint on node** = “Do not schedule pods here unless allowed”
* **Toleration on pod** = “This pod is allowed to run on that tainted node”

---

# Why Used?

Used for:

* Dedicated GPU nodes
* Production-only nodes
* Isolating workloads
* Spot/preemptible nodes
* Preventing normal pods on master/control-plane nodes

---

# Example Taint on Node

```bash id="a811b922"
kubectl taint nodes worker1 env=prod:NoSchedule
```

Meaning:

```text id="b922c033"
Pods cannot schedule on worker1 unless they tolerate env=prod
```

---

# Pod Toleration Example

```yaml id="c033d144"
tolerations:
- key: "env"
  operator: "Equal"
  value: "prod"
  effect: "NoSchedule"
```

---

# Taint Effects

## 1. NoSchedule

New pods won't schedule.

## 2. PreferNoSchedule

Soft preference to avoid scheduling.

## 3. NoExecute

Existing non-tolerating pods are evicted.

---

# Strong Interview Answer

> Taints repel pods from nodes, tolerations allow selected pods to run there. Useful for dedicated or restricted nodes.

---

# 82. How do liveness and readiness probes work?

Liveness Probe and Readiness Probe are health checks for containers.

---

# Readiness Probe

Checks if app is ready to receive traffic.

If probe fails:

* Pod keeps running
* Removed from Service endpoints
* No traffic sent

---

# Liveness Probe

Checks if app is alive / not hung.

If probe fails:

* Container restarted automatically

---

# Example

```yaml id="d144e255"
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 10

livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
```

---

# Real Use Case

```text id="e255f366"
App starts in 20 sec
Readiness waits until app ready
Liveness restarts if app hangs later
```

---

# Common Mistakes

* Wrong path
* Wrong port
* Delay too short
* Timeout too aggressive
* Dependency checks too strict

---

# Strong Interview Answer

> Readiness controls traffic flow, liveness controls restarts. Readiness failure removes pod from service; liveness failure restarts container.

---

# 83. How to troubleshoot a pod stuck in CrashLoopBackOff?

CrashLoopBackOff means container starts, crashes, and Kubernetes retries with backoff.

---

# Step-by-Step

## 1. Describe Pod

```bash id="f366g477"
kubectl describe pod <pod-name> -n <namespace>
```

Check:

* Events
* Exit codes
* OOMKilled
* Probe failures

---

## 2. Check Logs

```bash id="g477h588"
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
```

`--previous` is very important.

---

## 3. Common Causes

### App Crash

Bad code/config.

### Wrong Command

Container exits immediately.

### Missing Secret / ConfigMap

App fails on startup.

### OOMKilled

Memory too low.

### Probe Failure

Liveness restarting pod.

---

## 4. Inspect YAML

```bash id="h588i699"
kubectl get pod <pod-name> -o yaml
```

Check command, env, mounts.

---

## 5. Debug Shell

```bash id="i699j700"
kubectl run debug --rm -it --image=<image> -- sh
```

---

# Strong Interview Answer

> I inspect events, previous logs, exit code, probes, resources, and dependencies. Most CrashLoopBackOff issues are config, memory, or startup command related.

---

# 84. What is Builder POD?

“Builder Pod” is not a native Kubernetes object. It is commonly used in CI/CD platforms to mean a pod used to **build application artifacts or container images**.

---

# Where It Appears

## 1. Jenkins on Kubernetes

Dynamic agent pod used for builds.

## 2. OpenShift BuildConfig

Build pods compile code and create images.

## 3. GitLab Runner on Kubernetes

Ephemeral build pods.

---

# What It Does

* Pull source code
* Run build commands
* Run tests
* Build Docker/OCI image
* Push image to registry

---

# Example Flow

```text id="j700k811"
Git commit -> pipeline -> builder pod starts -> image built -> pod removed
```

---

# Strong Interview Answer

> Builder pod usually refers to an ephemeral CI/CD pod used to compile code, run tests, and build/push container images.

---

# 85. What is deployer POD?

Like builder pod, “Deployer Pod” is commonly platform/tool terminology, not a core Kubernetes resource.

---

# Meaning

A pod responsible for deploying an application into Kubernetes.

Used by CI/CD tools or platforms.

---

# Common Uses

## 1. Pipeline Deployment Agent

Runs:

```bash id="k811l922"
kubectl apply -f manifests/
helm upgrade --install app
```

## 2. OpenShift Older DeploymentConfig

Historically used deployer pods during rollout process.

---

# What It Does

* Pull manifests / Helm charts
* Apply deployments/services
* Run rollout checks
* Roll back if failed

---

# Example Flow

```text id="l922m033"
Image built -> deployer pod runs -> updates deployment -> verifies rollout
```

---

# Strong Interview Answer

> Deployer pod generally means a CI/CD or platform-managed pod that performs rollout actions such as applying manifests or Helm upgrades.

---

# Interview Tip

If asked Builder/Deployer pod in a generic Kubernetes interview, clarify:

> “Those aren’t native Kubernetes workload types; they’re usually CI/CD platform terms.”

That answer shows maturity.

---

## 86. How to create a New application POD?

A **Pod** is the smallest deployable unit in Kubernetes. It runs one or more containers.

For production, we usually create a **Deployment**, but if interviewer asks Pod creation, show both imperative and YAML methods.

---

# Method 1: Imperative Command

Create an NGINX pod:

```bash id="a861b972"
kubectl run nginx-pod --image=nginx
```

Check:

```bash id="b972c083"
kubectl get pods
```

---

# Method 2: YAML Manifest (Recommended)

```yaml id="c083d194"
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: nginx:latest
    ports:
    - containerPort: 80
```

Apply:

```bash id="d194e205"
kubectl apply -f pod.yaml
```

---

# Production Best Practice

Use Deployment instead of standalone Pod:

```bash id="e205f316"
kubectl create deployment myapp --image=nginx
```

---

# Strong Interview Answer

> We can create a pod using `kubectl run` or YAML manifest, but in production we generally use Deployments because pods are ephemeral.

---

# 87. How to check logs of POD?

Use `kubectl logs`.

---

# Basic Command

```bash id="f316g427"
kubectl logs <pod-name>
```

Example:

```bash id="g427h538"
kubectl logs nginx-pod
```

---

# Follow Live Logs

```bash id="h538i649"
kubectl logs -f nginx-pod
```

---

# Multi-Container Pod

```bash id="i649j750"
kubectl logs <pod-name> -c <container-name>
```

---

# Previous Crash Logs

Useful for CrashLoopBackOff:

```bash id="j750k861"
kubectl logs <pod-name> --previous
```

---

# Logs in Namespace

```bash id="k861l972"
kubectl logs nginx-pod -n dev
```

---

# Strong Interview Answer

> I use `kubectl logs`, `-f` for live logs, `--previous` for crashed containers, and `-c` for multi-container pods.

---

# 88. What is Deployment Configuration & why we need DC?

This term often refers to **DeploymentConfig (DC)** in OpenShift, not standard Kubernetes Deployment.

So answer depends on platform.

---

# In Standard Kubernetes

Use **Deployment** object.

It manages:

* ReplicaSets
* Rolling updates
* Rollbacks
* Scaling

---

# In OpenShift

OpenShift uses **DeploymentConfig (DC)**.

It supports:

* Image change triggers
* Config change triggers
* Rollbacks
* Deployment strategies

---

# Why Need DC?

To automate application rollout when:

* New image pushed
* Config changed
* Need controlled deployments

---

# Example

```bash id="l972m083"
oc get dc
```

---

# Interview Safe Answer

> In Kubernetes we use Deployment. In OpenShift, DeploymentConfig is an older/native object that manages application rollouts with triggers and rollback controls.

---

# 89. What is SVC & why we need SVC?

SVC means **Service** in Kubernetes.

Service gives stable network access to pods.

---

# Why Need Service?

Pods are temporary:

```text id="m083n194"
Pod IP changes after restart
```

Service provides:

* Stable IP
* DNS name
* Load balancing
* Access to multiple pod replicas

---

# Example

Pods:

```text id="n194o205"
web-1
web-2
web-3
```

Service:

```text id="o205p316"
web-service
```

Traffic distributed across pods.

---

# Create Service

```bash id="p316q427"
kubectl expose deployment myapp --port=80 --type=ClusterIP
```

---

# Types of Services

* ClusterIP
* NodePort
* LoadBalancer
* ExternalName
* Headless

---

# Strong Interview Answer

> Service gives stable connectivity and load balancing to pods whose IPs may change.

---

# 90. What is RC (Replication Controller)?

ReplicationController is an older Kubernetes controller that ensures a specified number of pod replicas are running.

---

# Example

If desired replicas = 3

```text id="q427r538"
One pod fails -> RC creates new pod
```

---

# Why Used?

Earlier Kubernetes versions used RC for:

* Scaling pods
* Self-healing
* High availability

---

# Example YAML

```yaml id="r538s649"
apiVersion: v1
kind: ReplicationController
metadata:
  name: web-rc
spec:
  replicas: 3
  selector:
    app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx
```

---

# Modern Replacement

Today we use:

* ReplicaSet
* Deployment

ReplicaSet is more advanced than RC.

---

# Strong Interview Answer

> ReplicationController is a legacy controller that maintains pod replica count. It has mostly been replaced by ReplicaSets managed through Deployments.

---

# Interview Tip

If interviewer asks RC, say:

> RC is older; today I use Deployments and ReplicaSets in production.

That sounds experienced.

---

## 91. How to check DC of POD & how to edit DC?

This is **OpenShift** terminology. DC = OpenShift **DeploymentConfig**.

A pod is usually created by a DeploymentConfig, Deployment, StatefulSet, etc.

---

# Check Which DC Created the Pod

```bash id="a911b022"
oc describe pod <pod-name>
```

Look for:

```text id="b022c133"
Controlled By: DeploymentConfig/myapp
```

---

# List DeploymentConfigs

```bash id="c133d244"
oc get dc
```

---

# View Specific DC

```bash id="d244e355"
oc describe dc <dc-name>
oc get dc <dc-name> -o yaml
```

---

# Edit DC

```bash id="e355f466"
oc edit dc <dc-name>
```

This opens editor to modify:

* image
* replicas
* env vars
* resources

---

# Example Scale in Edit

```yaml id="f466g577"
spec:
  replicas: 5
```

---

# Strong Interview Answer

> I first identify if pod is controlled by a DeploymentConfig using `oc describe pod`, then use `oc get dc`, `oc describe dc`, or `oc edit dc` to manage it.

---

# 92. How to create route?

OpenShift Route exposes a service externally using hostname.

---

# Basic Route

```bash id="g577h688"
oc expose svc myapp
```

Creates route automatically.

---

# Specify Hostname

```bash id="h688i799"
oc expose svc myapp --hostname=myapp.apps.company.com
```

---

# Check Routes

```bash id="i799j800"
oc get route
```

---

# Why Route?

Equivalent to Ingress in standard Kubernetes.

---

# 93. How to expose svc?

Expose Service means make app reachable.

---

# In OpenShift Route

```bash id="j800k911"
oc expose svc myapp
```

---

# In Kubernetes Service Creation

```bash id="k911l022"
kubectl expose deployment myapp --port=80 --type=ClusterIP
```

---

# External Exposure Options

* Route (OpenShift)
* NodePort
* LoadBalancer
* Ingress

---

# Check Service

```bash id="l022m133"
oc get svc
kubectl get svc
```

---

# Strong Interview Answer

> In OpenShift I usually expose the service using a Route. In Kubernetes I use Service types like ClusterIP, NodePort, LoadBalancer, or Ingress.

---

# 94. How to do rollout?

Rollout means deploying a new version/configuration.

---

# OpenShift DC Rollout

```bash id="m133n244"
oc rollout latest dc/myapp
```

Check status:

```bash id="n244o355"
oc rollout status dc/myapp
```

Undo:

```bash id="o355p466"
oc rollout undo dc/myapp
```

---

# Kubernetes Deployment Rollout

```bash id="p466q577"
kubectl rollout restart deployment/myapp
kubectl rollout status deployment/myapp
kubectl rollout undo deployment/myapp
```

---

# Strong Interview Answer

> Rollout updates an application version gradually and supports status checks and rollback.

---

# 95. How to increase replica?

Scaling means increasing pod count.

---

# OpenShift DC

```bash id="q577r688"
oc scale dc myapp --replicas=5
```

---

# Kubernetes Deployment

```bash id="r688s799"
kubectl scale deployment myapp --replicas=5
```

---

# Verify

```bash id="s799t800"
kubectl get pods
oc get pods
```

---

# Strong Interview Answer

> I use `scale` command or edit replicas in manifest.

---

# 96. What is Source to Image in OpenShift?

OpenShift **Source-to-Image (S2I)** builds container images directly from source code.

---

# How It Works

```text id="t800u911"
Source code + Builder image -> Application image
```

Example:

* Java source + Java builder image
* Node.js code + Node builder image

---

# Benefits

* No Dockerfile required
* Standardized builds
* Faster developer workflow
* Secure enterprise builds

---

# Example

```bash id="u911v022"
oc new-app nodejs~https://github.com/app/repo.git
```

---

# Strong Interview Answer

> S2I takes application source code and combines it with a builder image to create a runnable container image automatically.

---

# 97. What is builder image?

Builder image is a preconfigured container image used to build source code.

---

# Contains

* Runtime tools
* Build tools
* Language dependencies

Examples:

* Red Hat Node.js builder image
* Java Maven builder image
* Python builder image

---

# Example

```text id="v022w133"
nodejs builder image contains node + npm
```

Used by S2I.

---

# Strong Interview Answer

> Builder image contains compilers, runtimes, and scripts used to convert source code into a deployable application image.

---

# 98. What are the process to create source to image?

# S2I Workflow

---

## Step 1: Provide Source Code Repo

Git repository.

---

## Step 2: Choose Builder Image

Example:

* Node.js
* Java
* Python

---

## Step 3: Create App

```bash id="w133x244"
oc new-app nodejs~https://github.com/user/app.git
```

---

## Step 4: Build Starts

Builder pod:

* pulls code
* installs dependencies
* builds artifact
* creates final image

---

## Step 5: Image Stored in Internal Registry

---

## Step 6: Deployment Created

Application pods run new image.

---

## Check Build

```bash id="x244y355"
oc get builds
oc logs -f build/<build-name>
```

---

# Strong Interview Answer

> S2I process is source repo + builder image + build pod + generated runtime image + deployment.

---

# 99. How to give the Cluster role/permission to the user?

Use RBAC.

---

# OpenShift Add Cluster Role

```bash id="y355z466"
oc adm policy add-cluster-role-to-user cluster-admin rajesh
```

---

# Kubernetes ClusterRoleBinding

```bash id="z466a577"
kubectl create clusterrolebinding rajesh-admin \
--clusterrole=cluster-admin \
--user=rajesh
```

---

# View Bindings

```bash id="a577b688"
kubectl get clusterrolebinding
```

---

# Best Practice

Avoid cluster-admin unless necessary.

Use least privilege roles.

---

# Strong Interview Answer

> I grant access using ClusterRole + ClusterRoleBinding. For admin tasks, cluster-admin can be assigned carefully.

---

# 100. How to create secure route?

Secure Route in OpenShift means HTTPS/TLS-enabled route.

---

# Edge Termination

TLS ends at router.

```bash id="b688c799"
oc create route edge myapp-secure \
--service=myapp \
--hostname=myapp.apps.company.com
```

---

# Re-encrypt Route

TLS at router + TLS to backend.

```bash id="c799d800"
oc create route reencrypt myapp-secure \
--service=myapp
```

---

# Passthrough Route

TLS passed directly to backend.

```bash id="d800e911"
oc create route passthrough myapp-secure \
--service=myapp
```

---

# Check Route

```bash id="e911f022"
oc get route
```

---

# Best Practice

Use:

* trusted certificate
* redirect HTTP to HTTPS
* modern TLS versions
* secure headers

---

# Strong Interview Answer

> Secure route exposes service over HTTPS using edge, re-encrypt, or passthrough TLS modes depending where encryption should terminate.

---

# Interview Tip

Questions 91–100 are heavily **OpenShift-focused**. If interviewer asks these, mention:

> OpenShift uses DeploymentConfig, Routes, and S2I in addition to standard Kubernetes objects.

That makes you stand out.

---

## 101. What is PV & PVC?

PersistentVolume (PV) and PersistentVolumeClaim (PVC) are Kubernetes storage objects used for persistent data.

---

# PV = PersistentVolume

A storage resource created in the cluster by admin or dynamically provisioned.

Examples:

* NFS
* Amazon Web Services EBS
* Microsoft Azure Disk
* Ceph

Example:

```text id="a1011b122"
100Gi storage available
```

---

# PVC = PersistentVolumeClaim

A request for storage by application/user.

Example:

```text id="b122c233"
Need 20Gi ReadWriteOnce
```

---

# Flow

```text id="c233d344"
PVC requests storage -> binds to PV -> Pod mounts PVC
```

---

# Why Needed?

Separates app teams from storage backend details.

---

# Strong Interview Answer

> PV is actual storage resource; PVC is request/claim for storage used by pods.

---

# 102. What are access modes in PV?

Access modes define how a volume can be mounted.

---

# Main Modes

## 1. ReadWriteOnce (RWO)

Mounted read-write by one node.

Common with cloud block storage.

```text id="d344e455"
One node only
```

---

## 2. ReadOnlyMany (ROX)

Many nodes can mount read-only.

---

## 3. ReadWriteMany (RWX)

Many nodes can mount read-write simultaneously.

Used with NFS / shared storage.

---

## 4. ReadWriteOncePod (RWOP)

Mounted read-write by only one pod cluster-wide.

(Newer Kubernetes mode)

---

# Examples

* EBS / Azure Disk = mostly RWO
* NFS = RWX

---

# Strong Interview Answer

> Access modes define whether one or many nodes/pods can mount the volume and with read-only or read-write access.

---

# 103. What is node selector?

nodeSelector is the simplest way to schedule a pod onto nodes with matching labels.

---

# Example

Label node:

```bash id="e455f566"
kubectl label node worker1 disktype=ssd
```

Pod YAML:

```yaml id="f566g677"
spec:
  nodeSelector:
    disktype: ssd
```

---

# Result

Pod runs only on nodes labeled:

```text id="g677h788"
disktype=ssd
```

---

# Use Cases

* GPU nodes
* SSD nodes
* Dedicated environments
* Region-specific workloads

---

# Strong Interview Answer

> nodeSelector pins pods to nodes using simple key-value labels.

---

# 104. What are the two regions in projects?

This question is commonly asked in OpenShift context and often means **user/project roles or project visibility zones** rather than geographic regions. It may also be a typo for “roles”.

Most common interview interpretation:

# Two Types of Project Access in OpenShift

## 1. User Project / Namespace

Application team deploys workloads.

## 2. System / Infrastructure Project

Reserved for platform components.

Examples:

```text id="h788i899"
openshift-*
kube-system
```

---

# If They Mean Geographic Regions

Then answer:

```text id="i899j900"
Primary region
Disaster Recovery region
```

---

# Best Interview Response

> This question can mean project access areas or cloud regions. In OpenShift, we usually separate user namespaces and system namespaces. In DR design, we use primary and secondary regions.

---

# 105. Difference between template & Deployment Configuration?

This is OpenShift terminology.

---

# Template

Reusable parameterized blueprint containing multiple objects.

Can generate:

* Service
* Route
* DeploymentConfig
* Secret

Example:

```text id="j900k011"
APP_NAME=myapp
IMAGE=nodejs
```

---

# DeploymentConfig (DC)

Actual deployment controller managing rollout of app pods.

Handles:

* replicas
* image changes
* rollout strategy

---

# Difference

| Feature           | Template           | DeploymentConfig |
| ----------------- | ------------------ | ---------------- |
| Purpose           | Reusable blueprint | Deploy app       |
| Creates Resources | Yes                | No               |
| Manages Rollout   | No                 | Yes              |
| Parameters        | Yes                | Limited          |

---

# Strong Interview Answer

> Template is used to generate resources; DeploymentConfig manages deployment lifecycle.

---

# 106. How to migrate whole cluster to another?

Cluster migration means workloads, configs, storage, and access moved to a new cluster.

---

# High-Level Process

## 1. Build Target Cluster

Same / newer version.

## 2. Export Resources

```bash id="k011l122"
kubectl get all -A -o yaml
```

Also export:

* ConfigMaps
* Secrets
* CRDs
* RBAC
* Ingress

---

## 3. Migrate Storage/Data

* DB replication
* Snapshot restore
* PVC migration tools

---

## 4. Restore to New Cluster

Use GitOps:

* ArgoCD
* Helm
* Terraform

---

## 5. Validate

* Pods healthy
* Services reachable
* Monitoring active

---

## 6. Cutover Traffic

DNS / Load balancer switch.

---

# Strong Interview Answer

> I recreate infrastructure as code, migrate data separately, restore workloads, validate, then cut traffic gradually.

---

# 107. How to manually migrate container?

If they mean move workload manually from one node/cluster to another:

---

# Within Same Cluster

Drain node:

```bash id="l122m233"
kubectl drain <node> --ignore-daemonsets
```

Pods reschedule elsewhere.

---

# To Another Cluster

1. Push image to registry
2. Export manifests
3. Apply to target cluster

```bash id="m233n344"
kubectl apply -f app.yaml
```

---

# If Need Data

Copy persistent data separately.

---

# If Plain Docker Container

Commit/push image then run elsewhere.

---

# Strong Interview Answer

> Containers themselves are ephemeral; we migrate images + manifests + data, not “move” running containers directly.

---

# 108. What are the types of health probes in Kubernetes? How does K8s determine if a pod is ready?

Kubernetes probes

---

# Probe Types

## 1. Liveness Probe

Checks if app is alive.

Fail = restart container.

---

## 2. Readiness Probe

Checks if app can receive traffic.

Fail = removed from service endpoints.

---

## 3. Startup Probe

For slow-starting apps.

Disables liveness until startup completes.

---

# How K8s Determines Readiness

If readiness probe succeeds:

```text id="n344o455"
Pod Ready = True
Added to Service endpoints
```

If fails:

```text id="o455p566"
Ready = False
No traffic routed
```

---

# Methods

* HTTP GET
* TCP socket
* Exec command

---

# 109. Difference between stateful and stateless applications?

# Stateless Applications

Do not store session/data locally.

Examples:

* Web frontend
* REST APIs

Pods interchangeable.

Easy horizontal scaling.

---

# Stateful Applications

Need persistent data / identity.

Examples:

* MongoDB
* Apache Kafka
* Redis cluster

Need:

* Persistent storage
* Stable hostname
* Ordered startup

---

# Comparison

| Feature      | Stateless  | Stateful     |
| ------------ | ---------- | ------------ |
| Local Data   | No         | Yes          |
| Easy Replace | Yes        | More Complex |
| Controller   | Deployment | StatefulSet  |

---

# Strong Interview Answer

> Stateless apps can be replaced freely; stateful apps depend on data persistence and stable identity.

---

# 110. Difference between ConfigMap and Secret in Kubernetes?

ConfigMap stores non-sensitive configuration.
Secret stores sensitive data.

---

# ConfigMap Examples

* URLs
* Feature flags
* App mode

```yaml id="p566q677"
data:
  APP_ENV: prod
```

---

# Secret Examples

* Passwords
* Tokens
* TLS certs

```yaml id="q677r788"
data:
  password: cGFzcw==
```

(Base64 encoded)

---

# Differences

| Feature   | ConfigMap | Secret                               |
| --------- | --------- | ------------------------------------ |
| Sensitive | No        | Yes                                  |
| Use Case  | Config    | Credentials                          |
| Security  | Basic     | Better (encrypt at rest recommended) |

---

# Best Practice

Use external secret stores:

* HashiCorp Vault
* Cloud key vaults

---

# Strong Interview Answer

> ConfigMap is for normal configuration; Secret is for credentials and confidential values.

---
## 111. How can we restrict which pod can access the other pod in Kubernetes?

The standard way is using NetworkPolicy.

By default, many clusters allow pod-to-pod communication. NetworkPolicy lets you enforce **zero trust** communication.

---

# Example Requirement

Only frontend pods can access backend pods on port 8080.

---

# Example Policy

```yaml id="a111b222"
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
  policyTypes:
  - Ingress
```

---

# Result

```text id="b222c333"
frontend -> backend allowed
other pods -> blocked
```

---

# Also Restrict Cross-Namespace Access

Use:

* `namespaceSelector`
* `podSelector`

---

# Important Requirement

Need CNI supporting policies:

* Calico
* Cilium

---

# Strong Interview Answer

> I use NetworkPolicies with pod labels and namespace selectors to allow only required pod communication.

---

# 112. Suppose I want the pod to be scheduled on a specific node only. How can I achieve this?

Use one of these methods:

1. nodeSelector
2. Node Affinity
3. Taints + Tolerations (advanced isolation)

---

# Option 1: nodeSelector (Simple)

Label node:

```bash id="c333d444"
kubectl label node worker1 env=prod
```

Pod YAML:

```yaml id="d444e555"
spec:
  nodeSelector:
    env: prod
```

---

# Option 2: Node Affinity (Preferred)

```yaml id="e555f666"
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: env
          operator: In
          values:
          - prod
```

---

# Option 3: Dedicated Node

Taint node:

```bash id="f666g777"
kubectl taint node worker1 dedicated=app:NoSchedule
```

Add toleration to pod.

---

# Strong Interview Answer

> For specific node scheduling, I use nodeSelector for simple cases and node affinity for production-grade flexible scheduling.

---

# 113. I am getting a 503 error when I hit a load balancer URL, which routes traffic to the application deployed in a Kubernetes cluster. How will you troubleshoot this?

503 usually means:

```text id="g777h888"
Load balancer reachable, but backend unavailable
```

This is commonly due to no healthy pods/endpoints.

---

# Step-by-Step Troubleshooting

## 1. Check LoadBalancer Service

```bash id="h888i999"
kubectl get svc -A
kubectl describe svc <service-name>
```

Verify external IP exists.

---

## 2. Check Endpoints

```bash id="i999j000"
kubectl get endpoints <service-name> -n <namespace>
```

If empty:

```text id="j000k111"
No healthy backend pods
```

---

## 3. Check Pods

```bash id="k111l222"
kubectl get pods -n <namespace>
```

Look for:

* CrashLoopBackOff
* Pending
* Not Ready

---

## 4. Check Readiness Probe

Pods running but not Ready = not added to service endpoints.

```bash id="l222m333"
kubectl describe pod <pod>
```

---

## 5. Verify Labels / Selectors

```bash id="m333n444"
kubectl get svc <svc> -o yaml
kubectl get pods --show-labels
```

Service selector must match pod labels.

---

## 6. Check Ingress / LB Health Checks

Cloud LB may fail health probe path/port.

---

# Strong Interview Answer

> I first check if service has healthy endpoints. Most 503 issues happen when pods are not Ready, labels mismatch, or readiness probes fail.

---

# 114. I am getting CrashLoopBackOff error for one of the pods in a namespace. What should be the reason?

CrashLoopBackOff means container starts, crashes, and restarts repeatedly.

---

# Common Reasons

## 1. Application Crash

Bug or startup failure.

---

## 2. Wrong Command / Entrypoint

Container exits immediately.

---

## 3. Missing ConfigMap / Secret

App fails during startup.

---

## 4. OOMKilled

Memory exceeded.

```bash id="n444o555"
kubectl describe pod <pod>
```

Look for:

```text id="o555p666"
Reason: OOMKilled
```

---

## 5. Liveness Probe Failure

Health check restarts container continuously.

---

## 6. Dependency Unreachable

DB/API not reachable.

---

# Troubleshooting Commands

```bash id="p666q777"
kubectl logs <pod>
kubectl logs <pod> --previous
kubectl describe pod <pod>
```

---

# Strong Interview Answer

> CrashLoopBackOff is usually due to app crash, wrong startup command, probe failure, missing config, or memory kill.

---

# 115. Pod in Pending state after deployment is done? What can be the reason behind this?

Pending Pod means pod is accepted but not yet scheduled/running.

---

# Common Reasons

## 1. Insufficient Resources

No node has enough CPU/RAM.

```text id="q777r888"
0/3 nodes available: insufficient memory
```

---

## 2. Node Selector / Affinity Mismatch

Pod requests labels no node has.

---

## 3. Taints Without Tolerations

Node repels pod.

---

## 4. PVC Pending

Storage volume not available.

```bash id="r888s999"
kubectl get pvc
```

---

## 5. Image Pull Waiting (sometimes after scheduling)

Though usually moves to ContainerCreating/ImagePullBackOff later.

---

## 6. Quota / LimitRange Restrictions

Namespace resource quota exceeded.

---

# Troubleshooting Commands

```bash id="s999t000"
kubectl describe pod <pod>
kubectl get nodes
kubectl top nodes
kubectl get pvc
```

---

# Strong Interview Answer

> Pending usually means scheduler cannot place the pod due to resources, taints, affinity mismatch, or storage issues.

---

# Interview Tip

For questions 113–115, always say:

> I start with `kubectl describe pod/service` because Events usually reveal the root cause quickly.

That sounds practical and experienced.

---
## 116. How can you ensure high availability for your application deployed in K8s cluster?

High Availability (HA) means application stays available even if pods, nodes, or zones fail.

---

# Application-Level HA

## 1. Multiple Replicas

Run more than one pod.

```yaml id="a116b227"
replicas: 3
```

If one pod fails, others serve traffic.

---

## 2. Use Deployment / StatefulSet

* Stateless apps → Deployment
* Stateful apps → StatefulSet

---

## 3. Readiness + Liveness Probes

Only healthy pods receive traffic.

---

## 4. Pod Disruption Budget

Pod Disruption Budget prevents too many pods being down during maintenance.

```yaml id="b227c338"
minAvailable: 2
```

---

## 5. Pod Anti-Affinity

Spread replicas across nodes.

```text id="c338d449"
Do not place all pods on same node
```

---

## 6. Multi-Zone Node Pools

Distribute workloads across availability zones.

---

## 7. Autoscaling

* HPA for pods
* Cluster autoscaler for nodes

---

## 8. Ingress / Load Balancer HA

Multiple ingress replicas.

---

# Strong Interview Answer

> I ensure HA with multiple replicas, anti-affinity, probes, PDBs, multi-zone nodes, and autoscaling so failures don’t affect availability.

---

# 117. I want to run one one-time database migration task before my application starts?

Best Kubernetes-native options:

1. Init Container
2. Job
3. Helm Hook / Pipeline Migration Step

---

# Option 1: Init Container (Before App Container Starts)

Runs migration before app starts.

```yaml id="d449e550"
initContainers:
- name: migrate
  image: myapp:latest
  command: ["sh","-c","python migrate.py"]
```

Main container starts only after success.

---

# Best For

Simple migration tied to pod startup.

---

# Option 2: Kubernetes Job (Recommended Production)

Job runs once and exits.

```yaml id="e550f661"
kind: Job
```

Run migration separately, then deploy app.

---

# Option 3: CI/CD Step

Pipeline runs:

```text id="f661g772"
DB migration -> deploy app
```

Safest for production schema changes.

---

# Strong Interview Answer

> For controlled production releases, I prefer a Job or pipeline migration step. For simpler startup migrations, initContainers work well.

---

# 118. I want to only give read-only permissions to check the logs for users. How can you set up RBAC for this?

Use RBAC with Role + RoleBinding.

---

# Need Permissions

To view logs:

* get pods
* list pods
* get pods/log

---

# Role Example

```yaml id="g772h883"
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: log-reader
  namespace: prod
rules:
- apiGroups: [""]
  resources: ["pods","pods/log"]
  verbs: ["get","list","watch"]
```

---

# Bind User

```yaml id="h883i994"
kind: RoleBinding
metadata:
  name: log-reader-binding
  namespace: prod
subjects:
- kind: User
  name: rajesh
roleRef:
  kind: Role
  name: log-reader
  apiGroup: rbac.authorization.k8s.io
```

---

# Verify

```bash id="i994j005"
kubectl auth can-i get pods/log --as=rajesh -n prod
```

---

# Strong Interview Answer

> I create a namespace Role with read access to pods and pods/log, then bind it using RoleBinding.

---

# 119. What happens if the K8s master node and worker node firewall gets broken?

Firewall misconfiguration can severely impact cluster communication.

---

# If Control Plane Firewall Breaks

## API Server Blocked

Symptoms:

```text id="j005k116"
kubectl timeout
nodes NotReady
controllers fail
```

## ETCD Ports Blocked

Cluster instability.

## kubelet → API Server Blocked

Nodes become NotReady.

---

# If Worker Node Firewall Breaks

## Pod Networking Breaks

Pods cannot talk.

## Service Traffic Fails

kube-proxy rules ineffective if traffic blocked.

## DNS Fails

CoreDNS unreachable.

---

# Common Required Ports

* 6443 API Server
* 10250 kubelet
* etcd ports
* CNI overlay ports
* NodePort range if used

---

# Recovery Steps

1. Identify changed rules
2. Restore baseline firewall config
3. Test control plane connectivity
4. Validate node Ready status

---

# Strong Interview Answer

> Firewall issues can partition the cluster. If master communication breaks, nodes go NotReady. If worker ports break, pod networking and services fail.

---

# 120. What are the steps to be performed while updating a Kubernetes cluster?

Cluster upgrades must be planned carefully.

---

# Step 1: Review Compatibility

Check supported version upgrade path.

---

# Step 2: Backup ETCD / Cluster State

etcd snapshot.

```bash id="k116l227"
etcdctl snapshot save backup.db
```

---

# Step 3: Check Deprecated APIs

Ensure manifests compatible.

---

# Step 4: Upgrade Non-Prod First

Test staging cluster.

---

# Step 5: Upgrade Control Plane

Managed service or kubeadm process.

---

# Step 6: Upgrade Worker Nodes One by One

```bash id="l227m338"
kubectl drain <node> --ignore-daemonsets
```

Upgrade node, then uncordon.

```bash id="m338n449"
kubectl uncordon <node>
```

---

# Step 7: Validate Workloads

* Pods healthy
* Services healthy
* Monitoring normal

---

# Step 8: Upgrade Add-ons

* CNI
* CoreDNS
* CSI
* Ingress

---

# Strong Interview Answer

> I back up etcd, validate compatibility, upgrade control plane first, then rolling-upgrade workers node by node with drain/uncordon.

---

# 121. How can you ensure that only pods with a specific label can talk to your backend service?

Use NetworkPolicy.

---

# Example Requirement

Only pods labeled:

```text id="n449o550"
role=frontend
```

can access backend pods on port 8080.

---

# Policy Example

```yaml id="o550p661"
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 8080
  policyTypes:
  - Ingress
```

---

# Result

```text id="p661q772"
role=frontend -> allowed
all others -> denied
```

---

# Important

Need policy-capable CNI:

* Calico
* Cilium

---

# Strong Interview Answer

> I apply a NetworkPolicy selecting backend pods and allow ingress only from pods with the approved label.

---
## 122. All pods on the StatefulSet are trying to connect to the same storage volume. What's wrong, and how do you fix it?

This usually means the StatefulSet storage is configured incorrectly.

StatefulSet should give **each pod its own persistent volume**, not one shared volume (unless intentionally using RWX storage).

---

# What’s Wrong?

Most common mistakes:

## 1. Same PVC Mounted by All Pods

Using one shared PVC like:

```yaml id="a122b233"
claimName: data-pvc
```

for every replica.

That causes all pods to mount same storage.

---

## 2. Using Deployment Instead of StatefulSet

Deployment does not create unique per-pod storage identities.

---

## 3. Shared RWX Volume Used for Database Workload

Multiple DB replicas writing same filesystem can corrupt data.

---

# Correct Fix

Use `volumeClaimTemplates`.

```yaml id="b233c344"
volumeClaimTemplates:
- metadata:
    name: data
  spec:
    accessModes: ["ReadWriteOnce"]
    resources:
      requests:
        storage: 20Gi
```

This creates:

```text id="c344d455"
data-app-0
data-app-1
data-app-2
```

Unique PVC per pod.

---

# Why Important?

Stateful apps need:

* Separate disks
* Stable identity
* Data isolation

---

# Strong Interview Answer

> A StatefulSet should usually create one PVC per pod using volumeClaimTemplates. If all pods share one volume, storage was configured incorrectly.

---

# 123. Have you worked with Kubernetes? What deployment strategy do you prefer and why?

Yes—strong answer should combine practical use cases.

---

# Preferred Strategy Depends on Application Risk

## My Preferred Default: Rolling Update

Best for most stateless apps.

Why:

* Zero/minimal downtime
* Native Kubernetes support
* Easy rollback
* Efficient resource use

```yaml id="d455e566"
strategy:
  type: RollingUpdate
```

---

# For Critical Releases: Canary

Preferred when change risk is high.

```text id="e566f677"
5% traffic -> observe -> 25% -> 100%
```

Best for:

* APIs
* Revenue apps
* High user impact systems

---

# For Major Version Changes: Blue-Green

Safer for schema/app rewrites.

---

# Strong Interview Answer

> My default is RollingUpdate for standard stateless apps. For high-risk production changes I prefer Canary, and for major releases I use Blue-Green.

---

# 124. How have you implemented blue-green deployment in a Kubernetes environment?

Blue-Green means two identical environments:

```text id="f677g788"
Blue = current production
Green = new version
```

Traffic switches only after validation.

---

# Example Implementation

## Step 1: Existing Deployment

```text id="g788h899"
myapp-blue
```

## Step 2: New Deployment

```text id="h899i900"
myapp-green
```

---

## Step 3: Service Initially Points to Blue

```yaml id="i900j011"
selector:
  app: myapp-blue
```

---

## Step 4: Validate Green

* health checks
* smoke tests
* synthetic traffic

---

## Step 5: Switch Service Selector

```yaml id="j011k122"
selector:
  app: myapp-green
```

Traffic instantly moves.

---

## Step 6: Keep Blue for Rollback

Rollback = point service back to blue.

---

# Tools Used

* Argo Rollouts
* Istio
* NGINX Ingress

---

# Strong Interview Answer

> I deploy green alongside blue, validate it, then switch service routing. It gives near-zero downtime and instant rollback.

---

# 125. What is HPA in Kubernetes, and when would you use it?

Horizontal Pod Autoscaler automatically scales pod replicas based on metrics.

---

# Example

```text id="k122l233"
CPU high -> 3 pods to 8 pods
CPU low -> scale back to 3
```

---

# Based On

* CPU
* Memory
* Custom metrics
* External metrics

---

# Use Cases

## Use HPA For:

* Web apps
* APIs
* Traffic spikes
* Queue consumers

---

# Example

```bash id="l233m344"
kubectl autoscale deployment web \
--cpu-percent=70 --min=3 --max=10
```

---

# Strong Interview Answer

> HPA scales pods horizontally based on demand. I use it for stateless workloads with variable traffic.

---

# 126. Suppose a new deployment introduces issues — how would you roll back to the previous stable version using Kubernetes?

Kubernetes Deployments maintain rollout history.

---

# Check Rollout History

```bash id="m344n455"
kubectl rollout history deployment/myapp
```

---

# Roll Back to Previous Version

```bash id="n455o566"
kubectl rollout undo deployment/myapp
```

---

# Roll Back to Specific Revision

```bash id="o566p677"
kubectl rollout undo deployment/myapp --to-revision=2
```

---

# Verify

```bash id="p677q788"
kubectl rollout status deployment/myapp
```

---

# Production Best Practice

Trigger rollback automatically if:

* readiness fails
* error rate spikes
* latency rises

---

# Strong Interview Answer

> I use rollout history and `kubectl rollout undo`. In production I prefer automated rollback gates in CI/CD.

---

# 127. What is the purpose of StatefulSets in Kubernetes, and how are they different from Deployments?

StatefulSet is for applications needing stable identity and persistent storage.

---

# Purpose

Used for:

* Databases
* Queues
* Consensus systems

Examples:

* MongoDB
* Apache Kafka
* Redis cluster

---

# Difference from Deployment

| Feature       | Deployment       | StatefulSet        |
| ------------- | ---------------- | ------------------ |
| Workload      | Stateless        | Stateful           |
| Pod Names     | Random           | Stable             |
| Storage       | Shared/Ephemeral | Per-pod persistent |
| Startup Order | Any              | Ordered            |

---

# Strong Interview Answer

> Deployments are for interchangeable stateless pods. StatefulSets are for apps needing stable identity and dedicated storage.

---

# 128. How do you implement Pod Disruption Budgets in Kubernetes for high availability?

Pod Disruption Budget limits voluntary disruptions.

---

# Example

3 replicas app:

```yaml id="q788r899"
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: api
```

---

# Meaning

Only one pod can be disrupted at a time.

---

# Used During

* Node drain
* Upgrades
* Autoscaler scale-down

---

# Best Practice

Combine with:

* replicas >= 3
* anti-affinity
* probes

---

# Strong Interview Answer

> I define PDBs so maintenance events never take down too many replicas at once.

---

# 129. What are Kubernetes Ingress Controllers, and how are they configured in AKS?

Ingress Controller watches Ingress resources and configures routing.

Examples:

* NGINX Ingress
* Traefik
* Microsoft Application Gateway Ingress Controller (AGIC)

---

# In AKS Common Options

## 1. NGINX Ingress

Install via Helm.

## 2. Application Gateway Ingress Controller

Integrates with Azure Application Gateway.

---

# Basic Flow in AKS

```text id="r899s900"
Internet -> Public IP/App Gateway -> Ingress Controller -> Service -> Pods
```

---

# Example Ingress

```yaml id="s900t011"
rules:
- host: app.company.com
  http:
    paths:
    - path: /
```

---

# Strong Interview Answer

> Ingress controllers translate Ingress rules into real routing. In AKS I commonly use NGINX or Application Gateway Ingress Controller.

---

# 130. Explain how Kubernetes Horizontal Pod Autoscaler (HPA) works based on custom metrics.

By default HPA uses CPU/memory, but it can also scale using custom metrics.

---

# Examples of Custom Metrics

* Requests per second
* Queue length
* Active sessions
* Response latency
* Kafka lag

---

# Architecture

```text id="t011u122"
App metrics -> Prometheus -> Adapter -> HPA
```

Common stack:

* Prometheus
* Prometheus Adapter

---

# Example Use Case

Scale workers by queue depth:

```text id="u122v233"
Queue > 100 messages -> scale 2 to 10 pods
```

---

# Benefits

Better than CPU when CPU isn’t bottleneck.

---

# Example HPA YAML

```yaml id="v233w344"
metrics:
- type: Pods
  pods:
    metric:
      name: requests_per_second
```

---

# Strong Interview Answer

> Custom-metric HPA scales based on business signals like queue depth or RPS, which is often more accurate than CPU-based scaling.

---


