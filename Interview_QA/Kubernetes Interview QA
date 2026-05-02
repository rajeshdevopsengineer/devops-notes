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

