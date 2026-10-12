Your pasted list contains **well over 150 individual questions, with many repeats and near-duplicates** across Kubernetes/EKS, AWS, Terraform, Docker, Git/CI/CD, Linux, Ansible, monitoring, Azure, scripting, networking, SRE, and behavioral rounds. I consolidated repeated questions rather than giving contradictory copies of the same answer. The source list is aimed at roughly **6 YOE / senior cloud-administrator–DevOps–SRE level**, so the answers below are framed accordingly. Pasted text

# Master DevOps / Cloud / SRE Interview Answer Bank

## 1. Kubernetes architecture in depth

A Kubernetes cluster has two logical layers:

```text
                         +--------------------------+
                         |      CONTROL PLANE       |
                         |                          |
kubectl/API clients ---> | kube-apiserver           |
                         |       |                  |
                         |       +----> etcd         |
                         |       |                  |
                         |       +----> scheduler    |
                         |       |                  |
                         |       +----> controller   |
                         |            manager        |
                         +-------------+------------+
                                       |
                       Watches/updates desired state
                                       |
             +-------------------------+-------------------------+
             |                                                   |
      +------v-------+                                    +------v-------+
      | Worker Node1 |                                    | Worker Node2 |
      | kubelet      |                                    | kubelet      |
      | kube-proxy   |                                    | kube-proxy   |
      | containerd   |                                    | containerd   |
      | Pods         |                                    | Pods         |
      +--------------+                                    +--------------+
```

The **API server** is the front door. `kubectl`, controllers, scheduler and kubelets communicate through it. **etcd** stores Kubernetes API state. The **scheduler** watches unscheduled Pods and chooses suitable nodes according to resource requests, affinities, taints/tolerations and other constraints. The **controller manager** runs reconciliation loops such as Deployment, ReplicaSet and Node controllers. On worker nodes, **kubelet** ensures the PodSpec assigned to its node is actually running, while the container runtime such as containerd runs containers. `kube-proxy` implements Service forwarding unless the chosen CNI provides equivalent functionality. [Kubernetes](https://kubernetes.io/docs/concepts/architecture/?utm_source=chatgpt.com)

A useful workflow explanation is:

```text
kubectl apply deployment.yaml
        |
        v
API Server -> authentication -> authorization -> admission
        |
        v
      etcd
        |
        v
Deployment Controller notices desired replicas
        |
        v
ReplicaSet creates Pod objects
        |
        v
Scheduler assigns each Pod -> Node
        |
        v
Node kubelet sees PodSpec
        |
        v
CRI -> containerd -> image pull -> container starts
        |
        v
CNI configures Pod networking
        |
        v
Readiness succeeds -> Service Endpoint/EndpointSlice -> traffic
```

### Is there something like kubelet on the control-plane side?

There is **not one equivalent “control-plane agent”** doing everything. Control-plane functionality is split between `kube-apiserver`, `kube-scheduler` and `kube-controller-manager`.

However, in a kubeadm-based cluster the control-plane machine normally also runs **kubelet**. kubeadm writes the core control-plane components as static Pod manifests, and the kubelet watches those manifests and starts/restarts them. In EKS, AWS manages the Kubernetes control plane for you.

### How do you join a new node?

For a worker:

```bash
kubeadm token create --print-join-command
```

Then execute the generated command on the new machine:

```bash
sudo kubeadm join api.example.com:6443 \
  --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:<hash>
```

`kubeadm join` performs discovery, validates the cluster CA, starts kubelet TLS bootstrapping, requests a certificate and registers the Node. For another control-plane node use `--control-plane`, normally together with the uploaded certificate key. [Kubernetes](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/?utm_source=chatgpt.com)

---

# 2. What happens if the Kubernetes master/control plane fails?

In a single-control-plane cluster:

```text
Existing worker node:
Pod ---- continues running
kubelet ---- continues supervising containers

But:
kubectl apply       X
new scheduling      X
Deployment reconcile X
HPA decisions       X
new API operations  X
```

Existing Pods **usually continue running** because the container runtime and kubelet are on the workers. However, the cluster cannot properly reconcile desired state while the API/control plane is unavailable.

For production, use multiple control-plane instances and an API load balancer:

```text
             API Load Balancer
              /     |     \
             /      |      \
           CP1     CP2     CP3
             \      |      /
              \   etcd quorum
```

Loss of one member should not stop the cluster as long as quorum and enough control-plane capacity remain.

### What if etcd stops?

The API server loses its persistent source of cluster state. Existing containers do not instantly disappear, but you cannot reliably create/update/delete Kubernetes objects.

For HA etcd, use an odd number such as 3 or 5 members and maintain tested backups. Kubernetes explicitly recommends an etcd backup plan. [Kubernetes](https://kubernetes.io/docs/concepts/architecture/?utm_source=chatgpt.com)

### Can we manually write to etcd?

Technically `etcdctl` can access etcd, but **you should not manually manipulate Kubernetes registry entries**. Kubernetes object encoding, resource versions and consistency guarantees are managed through the API server.

For ordinary changes:

```bash
kubectl apply
kubectl patch
kubectl edit
```

Use direct etcd operations primarily for administrative backup/restore and carefully documented recovery procedures.

---

# 3. Pod is Pending — troubleshooting

Start with:

```bash
kubectl get pod <pod> -o wide
kubectl describe pod <pod>
kubectl get events --sort-by=.lastTimestamp
```

The key information is usually under `Events`.

Common causes include:

```text
Pending Pod
   |
   +-- Insufficient CPU/memory?
   |       -> requests too high / cluster full
   |
   +-- Node selector/affinity mismatch?
   |
   +-- Taint without toleration?
   |
   +-- PVC Pending / volume zone mismatch?
   |
   +-- max Pods / IP exhaustion?
   |
   +-- Pod topology/PDB constraints?
   |
   +-- Scheduler problem?
   |
   +-- Cluster autoscaler/Karpenter unable to provision?
```

For:

```text
0/5 nodes are available: insufficient memory
```

check:

```bash
kubectl top nodes
kubectl describe nodes
kubectl get pod <pod> -o yaml
```

The scheduler uses **requests**, not current `top` utilization, to determine schedulability.

Solutions can include reducing incorrect requests, adding nodes, enabling node autoscaling, moving workloads or using larger instances.

Scheduling is explicitly based on resource and policy constraints. [Kubernetes](https://kubernetes.io/docs/concepts/scheduling-eviction/?utm_source=chatgpt.com)

### Pending because of disk

Check both disk and inode exhaustion:

```bash
df -h
df -ih
du -xhd1 /var | sort -h
journalctl --disk-usage
docker system df
```

On Kubernetes/containerd:

```bash
sudo du -sh /var/lib/containerd/*
sudo du -sh /var/lib/kubelet/*
```

Also inspect node pressure:

```bash
kubectl describe node <node> | grep -A10 Conditions
```

Look for:

```text
DiskPressure=True
```

---

# 4. Worker node is not joining the cluster

I troubleshoot from bottom up.

```bash
systemctl status kubelet
journalctl -u kubelet -xe
```

Then validate:

```bash
swapoff -a
lsmod | grep br_netfilter
sysctl net.ipv4.ip_forward
crictl info
```

Check network access to API server:

```bash
nc -vz <api-server> 6443
curl -k https://<api-server>:6443/healthz
```

Then verify:

- DNS resolution
- firewall/NACL/security-group rules
- Kubernetes version compatibility
- kubeadm join token expiry
- CA hash
- container runtime
- CNI requirements
- system clock/NTP
- certificates
- kubelet bootstrap configuration

If the machine partially joined:

```bash
sudo kubeadm reset
```

correct the issue and re-run the appropriate `kubeadm join`.

---

# 5. StatefulSet, PVC and HPA

A StatefulSet is appropriate when an application needs one or more of:

- stable Pod identity
- ordered deployment/termination
- stable network identity
- persistent per-replica storage

Examples include Cassandra, ZooKeeper, Kafka and replicated databases.

Kubernetes creates one PVC for each StatefulSet ordinal when you use `volumeClaimTemplates`; that PVC remains associated with that ordinal across Pod recreation. [Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/?utm_source=chatgpt.com)

Example:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 100Gi
```

### If HPA creates a new StatefulSet Pod, won't the PVC be empty?

Yes: **a newly added StatefulSet ordinal gets a new PVC**, so the volume starts without the application's existing data unless your storage/application design populates it.

That is why stateful scaling normally works like this:

```text
HPA adds mysql-3
       |
       v
StatefulSet creates data-mysql-3 PVC
       |
       v
Storage provisioner creates volume
       |
       v
mysql-3 starts
       |
       v
DB joins existing cluster
       |
       v
DB replication/bootstrap copies data
       |
       v
Replica becomes Ready
```

The important part is **application-level replication or restore/bootstrap**.

You do not solve this by attaching the same ReadWriteOnce database volume to all replicas.

For something such as Cassandra:

1. new Pod gets fresh storage;
2. Pod discovers seeds;
3. joins the ring;
4. streams relevant partitions;
5. health/readiness succeeds;
6. it begins serving traffic.

If the application does not support safe horizontal stateful scaling, do not blindly apply HPA.

HPA can target StatefulSets; it changes replica count based on resource or custom metrics. [Kubernetes](https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/?utm_source=chatgpt.com)

---

# 6. HPA vs VPA

**HPA** changes the number of Pods.

```text
2 pods -> load rises -> 6 pods
```

Use it for horizontally scalable services such as REST APIs.

Example:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

Kubernetes HPA supports CPU, memory and custom/external metrics depending on the metrics APIs installed. [Kubernetes](https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/?utm_source=chatgpt.com)

**VPA** changes Pod resource requests/limits:

```text
500m CPU / 512Mi
        |
        v
1 CPU / 1Gi
```

Use it when the number of replicas should stay approximately stable but resource sizing is difficult.

For example:

```text
Java service continuously memory constrained
       |
       v
VPA observes actual resource usage
       |
       v
recommends/increases memory request
```

### HPA and VPA together?

Possible, but avoid having both modify the same CPU/memory signal in conflicting ways. A common approach is HPA based on custom/business metrics while VPA adjusts resource sizing.

Kubernetes describes horizontal scaling as increasing instances and vertical scaling as resizing CPU/memory assigned to containers. [Kubernetes](https://kubernetes.io/docs/concepts/workloads/autoscaling/?utm_source=chatgpt.com)

---

# 7. Scaling vs autoscaling

Manual scaling:

```bash
kubectl scale deployment api --replicas=10
```

Autoscaling:

```text
Metrics / pending workload
          |
          +--> HPA -> Pod replicas
          |
          +--> VPA -> Pod resources
          |
          +--> Karpenter/Cluster Autoscaler -> Nodes
```

---

# 8. NodeSelector, affinity, taints and tolerations

Label nodes:

```bash
kubectl label node node1 size=large
kubectl label node node2 size=medium
kubectl label node node3 size=small
```

To schedule **only large**:

```yaml
nodeSelector:
  size: large
```

To allow large OR medium but exclude small:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: size
          operator: In
          values:
          - large
          - medium
```

Node affinity is more expressive than `nodeSelector`; Kubernetes supports both hard `required...` and soft `preferred...` affinity. [Kubernetes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/?utm_source=chatgpt.com)

Taints repel workloads:

```bash
kubectl taint nodes gpu-node dedicated=gpu:NoSchedule
```

A Pod must explicitly tolerate it:

```yaml
tolerations:
- key: dedicated
  operator: Equal
  value: gpu
  effect: NoSchedule
```

Remember:

```text
Affinity / nodeSelector = Pod attracts/selects a node
Taint                  = Node repels Pods
Toleration             = Pod is allowed through the taint
```

---

# 9. Kubernetes Services

Main types:

```text
ClusterIP
NodePort
LoadBalancer
ExternalName
```

**ClusterIP** is default and exposes the application only inside the cluster.

**NodePort** opens a port on the nodes and forwards it to the Service.

**LoadBalancer** asks an integrated cloud provider/load-balancer controller to expose the Service externally.

### Service vs Deployment

Deployment:

```text
manages Pods and application lifecycle
```

Service:

```text
stable networking + load balancing to matching Pods
```

Typical flow:

```text
Client
  |
Service ClusterIP
  |
EndpointSlice
  |
Pod A / Pod B / Pod C
```

### What if targetPort is missing?

If `targetPort` is omitted, Kubernetes defaults it to the same value as `port`.

For example:

```yaml
ports:
- port: 80
```

is effectively:

```yaml
ports:
- port: 80
  targetPort: 80
```

If the application actually listens on `8080`, use:

```yaml
port: 80
targetPort: 8080
```

---

# 10. Why isn't Service mapping to Deployment?

Services do not directly map to Deployment objects.

They select **Pods by labels**.

Check:

```bash
kubectl get svc myservice -o yaml
kubectl get pods --show-labels
kubectl get endpointslices
```

Example:

```yaml
selector:
  app: web
```

requires Pods containing:

```yaml
labels:
  app: web
```

Typical causes:

- selector mismatch
- wrong namespace
- Pods not Ready
- wrong `targetPort`
- application isn't listening
- NetworkPolicy/firewall problem

---

# 11. Kubernetes probes

Three main probe purposes:

**Startup probe:** “Has the application finished starting?”

**Readiness probe:** “Should it receive Service traffic?”

**Liveness probe:** “Is it dead and should Kubernetes restart it?”

Kubernetes supports HTTP, TCP, command/exec and gRPC probe mechanisms where appropriate. [Kubernetes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/?utm_source=chatgpt.com)

Example:

```yaml
startupProbe:
  httpGet:
    path: /startup
    port: 8080
  failureThreshold: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  periodSeconds: 5

livenessProbe:
  httpGet:
    path: /health
    port: 8080
  periodSeconds: 10
```

Do not make liveness checks unnecessarily dependent on external databases; otherwise a database outage can trigger a restart storm.

---

# 12. Kubernetes Secrets

Built-in Secret types include `Opaque`, TLS, Docker registry credentials, SSH/basic-auth and others. [Kubernetes](https://kubernetes.io/docs/concepts/configuration/secret/?utm_source=chatgpt.com)

Example:

```bash
kubectl create secret generic db-secret \
  --from-literal=username=app \
  --from-literal=password='StrongPassword'
```

Use in a Pod:

```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: password
```

Security point for an interview: **base64 is encoding, not encryption**.

Kubernetes Secrets can be stored unencrypted in etcd unless encryption-at-rest is configured. Kubernetes supports encryption providers and KMS envelope encryption. [Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/?utm_source=chatgpt.com)

In production I would combine:

```text
RBAC least privilege
+
Encryption at rest/KMS
+
short-lived ServiceAccount tokens
+
external secret manager where required
+
audit logs
+
network policies
```

---

# 13. Kubernetes security — container and infrastructure side

A strong senior-level answer covers multiple layers.

```text
                   Kubernetes Security
                         |
          +--------------+--------------+
          |                             |
     Workload                        Cluster/Infra
          |                             |
 non-root containers               private API endpoint
 read-only root FS                 RBAC least privilege
 seccomp                           etcd encryption
 drop capabilities                 audit logs
 no privileged Pods                patched worker AMIs
 resource limits                   network segmentation
 Pod Security                      KMS
 NetworkPolicies                   node hardening
 image scanning                    restricted IAM
 signed images                     backups
```

Use the `restricted` Pod Security Standard where possible. Kubernetes defines Privileged, Baseline and Restricted profiles. [Kubernetes](https://kubernetes.io/docs/concepts/security/pod-security-standards/?utm_source=chatgpt.com)

NetworkPolicy can enforce L3/L4 Pod ingress/egress when supported by the CNI. [Kubernetes](https://kubernetes.io/docs/concepts/services-networking/network-policies/?utm_source=chatgpt.com)

RBAC objects are:

```text
Role
ClusterRole
RoleBinding
ClusterRoleBinding
``` :chatgpt-content-reference{index="15"}


Example hardened container:

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
    - ALL
```

---

# 14. What is Karpenter? What metric does it scale on?

This question is often answered incorrectly.

Karpenter is a Kubernetes node-lifecycle/provisioning project. It watches Pods the Kubernetes scheduler considers **unschedulable**, evaluates their resource requests and scheduling requirements, and provisions matching compute capacity. [Karpenter](https://karpenter.sh/docs/?utm_source=chatgpt.com)

Therefore it is **not simply a “CPU above 70%” node autoscaler**.

```text
Pod created
   |
scheduler cannot place it
   |
Pod becomes unschedulable
   |
Karpenter evaluates:
 CPU/memory requests
 architecture
 instance family
 AZ
 capacity type
 taints
 affinity
 topology
   |
launches appropriate node
```

For scale-down Karpenter uses disruption/consolidation logic: empty or under-utilized nodes can be removed/replaced when Pods can safely run elsewhere and policy/PDB constraints allow it. [Karpenter](https://karpenter.sh/v1.15/concepts/disruption/?utm_source=chatgpt.com)

---

# 15. Controller for self-managed EKS workers

Self-managed EKS nodes are normally EC2 instances in an **Auto Scaling Group** that you manage. AWS documentation explicitly describes self-managed nodes as managed directly through EC2 APIs, whereas managed node groups have EKS lifecycle APIs. [AWS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/ml-node-groups.html?utm_source=chatgpt.com)

Node count can be driven by:

```text
Cluster Autoscaler -> EC2 ASG
```

or replaced with a Karpenter-oriented architecture.

Do not say “EKS automatically manages self-managed nodes”—that contradicts the name.

---

# 16. EKS cluster upgrade process

I use a staged process:

```text
Inventory/dependency validation
       |
       v
Deprecated API scan
       |
       v
Backups / rollback plan
       |
       v
Upgrade lower environment
       |
       v
Upgrade EKS control plane
       |
       v
Upgrade cluster add-ons
 VPC CNI / CoreDNS / kube-proxy / CSI
       |
       v
Upgrade node groups
       |
       v
Cordon/drain old nodes
       |
       v
Workload validation
       |
       v
Production rollout
```

Important points:

1. review Kubernetes/EKS version compatibility;
2. search manifests and CRDs for deprecated APIs;
3. validate PDBs and HA;
4. update the control plane;
5. update EKS add-ons/controllers;
6. roll worker nodes to compatible AMIs/Kubernetes versions;
7. perform smoke, integration, networking, DNS, storage and autoscaling checks;
8. observe metrics/errors before finishing.

AWS manages the EKS control-plane upgrade, while the data-plane/node version remains your responsibility. [AWS Documentation](https://docs.aws.amazon.com/eks/latest/best-practices/cluster-upgrades.html?utm_source=chatgpt.com)

---

# 17. Upgrade fails because storage plugin is incompatible

Do not force the entire production upgrade forward.

I would:

```text
1. Freeze further upgrade actions
2. Identify CSI/plugin compatibility matrix
3. Check controller + node plugin logs
4. Validate CRDs and Kubernetes API compatibility
5. Keep existing working nodes available
6. Upgrade/patch CSI driver
7. Test PVC attach/detach/mount
8. Canary one upgraded node
9. Resume node rollout
```

Typical commands:

```bash
kubectl get csidrivers
kubectl get pods -n kube-system
kubectl logs -n kube-system <csi-controller-pod>
kubectl describe pvc <pvc>
kubectl describe pv <pv>
kubectl get volumeattachments
```

For production, prove storage operations before aggressively draining stateful workloads.

---

# 18. Unable to evict Pods while draining a node

Start with:

```bash
kubectl drain node1 --ignore-daemonsets
```

Look for:

- blocking PDB
- unmanaged Pods
- DaemonSets
- local storage
- stuck finalizers
- terminating Pods
- StatefulSets with unavailable replicas
- admission webhook failures

Check:

```bash
kubectl get pdb -A
kubectl describe pdb <name>
kubectl get pod <pod> -o yaml
```

Do **not** immediately use `--force` in production. Determine why the workload says disruption is unsafe.

---

# 19. PDB

A PodDisruptionBudget limits **voluntary disruptions**.

Example:

```yaml
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

With 3 replicas, voluntary operations such as draining nodes should preserve at least two available replicas.

PDB does not stop every failure—for example, hardware failure can still kill a Pod.

---

# 20. PV vs PVC

```text
PersistentVolume
    = actual cluster storage resource

PersistentVolumeClaim
    = application's request for storage

StorageClass
    = provisioning policy/template
```

Dynamic flow:

```text
Pod -> PVC -> StorageClass -> CSI provisioner -> EBS/Azure Disk/etc.
```

---

# 21. Init container

An init container runs **before application containers** and must finish successfully.

Example:

```yaml
initContainers:
- name: wait-for-db
  image: busybox
  command:
  - sh
  - -c
  - until nc -z database 5432; do sleep 2; done
```

Use cases:

- wait for dependency
- retrieve configuration
- perform one-time initialization
- prepare filesystem
- schema checks

If the interview asks “run a shell script before the container starts,” an **init container** is usually the simpler answer—not a custom Kubernetes Operator.

---

# 22. Kubernetes Operator

An Operator combines:

```text
Custom Resource Definition
        +
Custom Controller
        =
domain-specific automation
```

Example:

```yaml
apiVersion: database.example.com/v1
kind: DatabaseCluster
spec:
  replicas: 3
```

The controller continually reconciles this desired state into StatefulSets, Services, backups, upgrades, failover, etc.

Use an Operator when operational knowledge itself needs automation—not merely for a one-time shell command.

---

# 23. Ingress isn't routing — troubleshooting

Flow:

```text
DNS
 |
Load Balancer
 |
Ingress/Gateway controller
 |
Ingress/Gateway rules
 |
Service
 |
EndpointSlice
 |
Pod
```

Debug every hop:

```bash
nslookup app.example.com
curl -vk https://app.example.com

kubectl get ingress
kubectl describe ingress
kubectl get ingressclass

kubectl get svc
kubectl get endpointslices
kubectl get pods -o wide

kubectl logs -n <controller-ns> <controller-pod>
```

Typical problems:

- wrong DNS
- LoadBalancer SG/firewall
- wrong ingress class
- controller missing
- hostname/path mismatch
- TLS Secret problem
- Service port mismatch
- empty EndpointSlice
- readiness failures
- NetworkPolicy
- application isn't listening

### Important 2026 update: ingress-nginx

The Kubernetes-community **Ingress NGINX project was retired in March 2026** and no longer receives security fixes. Kubernetes recommends migration toward Gateway API or another maintained implementation. This is **not the same product as F5 NGINX Ingress Controller**, which remains maintained. [Kubernetes](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/?utm_source=chatgpt.com)

As of 2026, alternatives include maintained Gateway API implementations such as NGINX Gateway Fabric, Traefik, HAProxy-based implementations, cloud-native gateways and others. Kubernetes describes Gateway API as its modern service-networking direction. [Kubernetes](https://kubernetes.io/blog/2026/04/21/gateway-api-v1-5/?utm_source=chatgpt.com)

---

# 24. Design Kubernetes with Ingress/Gateway

```text
                         Internet
                            |
                         Route53
                            |
                           ALB
                            |
                 Gateway / Ingress Controller
                    /             \
             /api                   /web
              |                       |
        api ClusterIP            web ClusterIP
          /      \                /       \
       Pod      Pod             Pod       Pod
        |                         |
      DB/cache                backend APIs

Metrics -> Prometheus -> Grafana
Logs    -> Fluent Bit -> OpenSearch/Loki
Secrets -> Secrets Manager / CSI
```

For HA, distribute replicas across AZs/nodes using topology constraints and Pod anti-affinity.

---

# 25. Istio design

```text
                         Internet
                            |
                     Istio Gateway
                            |
                       VirtualService
                       /            \
                 Service A         Service B
                /        \         /        \
            Pod+proxy Pod+proxy Pod+proxy Pod+proxy
                 \          Service Mesh       /
                            |
                     Istio control plane
```

Typical responsibilities:

- traffic management
- mTLS
- retries/timeouts
- circuit breaking
- canary percentages
- telemetry
- authorization policies

Do not deploy a service mesh simply because it is fashionable; factor in operational complexity.

---

# 26. Deployment with 3 Apache replicas

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apache
spec:
  replicas: 3
  selector:
    matchLabels:
      app: apache
  template:
    metadata:
      labels:
        app: apache
    spec:
      containers:
      - name: httpd
        image: httpd:2.4
        ports:
        - containerPort: 80
```

Apply:

```bash
kubectl apply -f deployment.yaml
kubectl rollout status deployment/apache
kubectl get pods -l app=apache
```

---

# 27. Add labels and annotations to an existing Pod

```bash
kubectl label pod mypod environment=prod
kubectl annotate pod mypod owner=platform-team
```

Overwrite:

```bash
kubectl label pod mypod environment=stage --overwrite
kubectl annotate pod mypod owner=sre --overwrite
```

---

# 28. Roll back Kubernetes Deployment

Check history:

```bash
kubectl rollout history deployment/myapp
```

Rollback one revision:

```bash
kubectl rollout undo deployment/myapp
```

Specific revision:

```bash
kubectl rollout undo deployment/myapp --to-revision=3
```

Verify:

```bash
kubectl rollout status deployment/myapp
```

---

# 29. Blue-green vs canary

### Blue-green

```text
              Load Balancer
                   |
             currently BLUE
              /          \
         BLUE v1       GREEN v2
         100%           0%

After validation:

              Load Balancer
                   |
                GREEN
              /       \
         BLUE v1    GREEN v2
           0%         100%
```

Advantages:

- extremely fast switch/rollback
- clean isolation

Disadvantage:

- duplicate capacity.

### Canary

```text
Load Balancer
     |
     +---- 95% ---> v1
     |
     +---- 5% ----> v2
```

Then:

```text
5% -> 20% -> 50% -> 100%
```

Good for evaluating error rate, latency and business KPIs before full exposure.

---

# 30. Helm deployment and CI/CD

Typical chart:

```text
myapp/
  Chart.yaml
  values.yaml
  templates/
    deployment.yaml
    service.yaml
    hpa.yaml
```

Important commands:

```bash
helm create myapp
helm lint ./myapp
helm template myapp ./myapp
helm install myapp ./myapp -n production --create-namespace
helm list -A
helm status myapp
helm history myapp
helm upgrade myapp ./myapp -f values-prod.yaml
helm rollback myapp 3
helm uninstall myapp
```

Helm officially supports linting, installation, upgrades, history and rollbacks. [Helm](https://docs.helm.sh/docs/intro/cheatsheet/?utm_source=chatgpt.com)

CI/CD:

```text
Git push
   |
Unit tests
   |
Build image
   |
Security scan
   |
Push image -> ECR
   |
helm lint/template
   |
Deploy dev
   |
Integration tests
   |
Approval
   |
helm upgrade --install production
   |
Smoke test
   |
Failure? -> rollback
```

A production-style deployment can use:

```bash
helm upgrade --install myapp ./chart \
  --namespace production \
  --values values-prod.yaml \
  --set image.tag="${GIT_SHA}" \
  --wait \
  --rollback-on-failure
```

Helm can wait for Pods/PVCs/Services and roll back a failed upgrade. [Helm](https://docs.helm.sh/docs/helm/helm_upgrade/?utm_source=chatgpt.com)

---

# 31. On-prem Kubernetes vs EKS

Prefer **on-prem Kubernetes** when:

- strict data-sovereignty requirements
- specialized hardware already owned
- predictable capacity makes owned infrastructure economical
- very low latency to on-prem equipment
- cloud connectivity is unsuitable
- organization requires complete control of networking/control plane

Prefer **EKS** when:

- you want managed Kubernetes control plane
- rapid environment creation matters
- AWS integration is valuable
- workloads need elastic capacity
- multi-AZ resiliency is easier in AWS
- operational overhead should be reduced

The decision is not “cloud always wins.” Discuss security, cost, latency, regulatory requirements, skills and operational ownership.

---

# 32. EKS vs ECS vs Fargate

**EKS:** managed Kubernetes control plane.

**ECS:** AWS-native container orchestrator.

**Fargate:** serverless compute option that can run containers without you managing EC2 worker hosts; it can be used by ECS and EKS in supported configurations.

Interview scenario:

```text
Need Kubernetes ecosystem / portability -> EKS

AWS-only simple container platform ->
ECS

Don't want to manage worker EC2 instances ->
Fargate
```

---

# 33. ECR

Amazon ECR is AWS's managed container registry.

Typical flow:

```bash
aws ecr get-login-password --region ap-south-1 |
docker login --username AWS --password-stdin \
123456789012.dkr.ecr.ap-south-1.amazonaws.com

docker build -t myapp:1.0 .

docker tag myapp:1.0 \
123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp:1.0

docker push \
123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp:1.0
```

ECR requires authentication before push; the auth token is time-limited. [AWS Documentation](https://docs.aws.amazon.com/AmazonECR/latest/userguide/docker-push-ecr-image.html?utm_source=chatgpt.com)

For production also discuss:

- immutable image tags where appropriate
- vulnerability scanning
- lifecycle policies
- encryption
- least-privilege repository policies
- cross-account access
- replication if required

---

# 34. AWS three-tier HA architecture

```text
                       Internet
                          |
                       Route53
                          |
                     CloudFront/WAF
                          |
                    Public ALB
               AZ-A              AZ-B
                 |                 |
            private app        private app
            ASG/EKS Pods       ASG/EKS Pods
                 \                /
                  \              /
                   RDS Multi-AZ
                       |
                    backups

Outbound:
private subnet -> NAT GW per AZ -> Internet GW

Observability:
CloudWatch + CloudTrail + alarms

Secrets:
Secrets Manager / Parameter Store

Encryption:
KMS
```

Security principles:

- no public DB
- least privilege IAM
- SG-to-SG access
- WAF
- TLS
- encrypted EBS/RDS/S3
- Secrets Manager
- Multi-AZ
- backups/PITR
- monitoring
- patching
- VPC Flow Logs/CloudTrail
- GuardDuty/Inspector/Security Hub where appropriate

---

# 35. Security Group vs NACL

| Security Group | Network ACL |
|---|---|
| Attached to ENI/resource | Attached to subnet |
| Stateful | Stateless |
| Allow rules only | Allow and deny rules |
| All rules evaluated | Ordered rules |
| Return traffic automatically allowed | Return path must also be allowed |

### Can SG explicitly deny a port?

No explicit `DENY` rule exists.

You “block” port 22 by **not allowing it**.

For explicit subnet-level deny rules, use NACL or an appropriate firewall.

---

# 36. NAT Gateway — two public and two private subnets

There is no technical rule saying the number of NAT Gateways must equal the number of private subnets.

Minimum:

```text
1 NAT Gateway
```

Both private subnets can route through it.

For production multi-AZ HA, normally:

```text
AZ-A:
 public subnet -> NAT GW A
 private subnet -> NAT GW A

AZ-B:
 public subnet -> NAT GW B
 private subnet -> NAT GW B
```

That avoids a single-AZ dependency and avoids unnecessary cross-AZ traffic.

So a good interview response is:

> Minimum one. For a resilient two-AZ architecture, normally two—one NAT Gateway per AZ. There isn't a useful architectural “maximum = two”; you could create more as the design requires.

For internet egress the NAT gateway belongs in a public subnet whose route reaches an Internet Gateway, while private subnets route internet-bound traffic to the NAT gateway. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-troubleshooting.html?utm_source=chatgpt.com)

---

# 37. VPC peering vs Transit Gateway

VPC Peering:

```text
VPC A <------> VPC B
```

Point-to-point.

With many VPCs:

```text
A-B
A-C
A-D
B-C
B-D
C-D
```

becomes operationally complex.

Transit Gateway:

```text
                 VPC A
                   |
VPC B ----- Transit Gateway ----- VPC C
                   |
                 VPC D
                   |
                 VPN/DX
```

Use TGW for hub-and-spoke connectivity across multiple VPCs/accounts and hybrid networks.

---

# 38. Connect AWS to on-premises

Common options:

```text
On-Prem
   |
   +-- Site-to-Site VPN ------ AWS VPC/TGW
   |
   +-- Direct Connect -------- AWS
   |
   +-- DX + VPN -------------- encrypted hybrid design
```

Use Transit Gateway when many VPCs need centralized connectivity.

---

# 39. Access S3 securely from VPC

For workloads inside a VPC, avoid unnecessarily sending S3 traffic through public internet/NAT.

Use:

```text
EC2/private subnet
      |
  route table
      |
S3 Gateway Endpoint
      |
      S3
```

or an S3 interface endpoint/PrivateLink where the use case requires private IP access, including certain hybrid requirements. AWS documents both gateway and interface endpoint options. [AWS Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/privatelink-interface-endpoints.html?utm_source=chatgpt.com)

Then restrict bucket policy using endpoint/VPC conditions where appropriate.

---

# 40. S3 static website without public bucket access

Do **not** use the S3 website endpoint if you require a private S3 origin.

Architecture:

```text
Browser
  |
HTTPS
  |
CloudFront
  |
Origin Access Control
  |
Private S3 bucket
```

Enable S3 Block Public Access and allow CloudFront's origin identity/control path through the bucket policy.

AWS recommends S3 Block Public Access unless public access is intentionally required. [AWS Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html?utm_source=chatgpt.com)

---

# 41. Bucket policy vs ACL

Bucket policy:

- JSON resource policy
- supports rich IAM conditions
- recommended for most bucket authorization patterns.

ACL:

- older access-control mechanism
- limited set of grants.

For modern S3 configurations, prefer IAM/bucket policy and Object Ownership/Bucket owner enforced where possible rather than ACL-heavy designs.

---

# 42. S3 lifecycle

Example:

```text
Day 0       S3 Standard
Day 30      Standard-IA
Day 90      Glacier
Day 365     Expire
```

Lifecycle policies can:

- transition storage class
- expire current objects
- remove old versions
- abort incomplete multipart uploads

Important distinction:

```text
Terraform lifecycle block
!=
S3 object lifecycle policy
```

---

# 43. Secrets Manager vs Parameter Store

Use **Secrets Manager** when you need secret-specific capabilities such as secret lifecycle/rotation integrations and centralized secret management.

Use **Systems Manager Parameter Store** for configuration values and secrets where its feature set is adequate.

Typical answer:

```text
DB password with automatic rotation -> Secrets Manager
Application configuration parameter -> Parameter Store
```

---

# 44. IAM User vs Role

IAM User:

```text
long-lived identity
often used for a person/legacy workload
may have password/access keys
```

IAM Role:

```text
assumed identity
temporary STS credentials
preferred for EC2/Lambda/EKS/cross-account access
```

For workloads, prefer temporary role credentials over embedded long-lived access keys.

---

# 45. Cross-account token/secret access

Requirement:

```text
Account A: EC2
Account B: secret/resource
```

Architecture:

```text
EC2 Instance Role (A)
        |
    sts:AssumeRole
        |
CrossAccountRole (B)
        |
GetSecretValue/resource permission
        |
Secret/resource
```

In Account B, the target role's trust policy allows Account A's workload role.

Account A's workload role is allowed to call:

```text
sts:AssumeRole
```

Then the application uses short-lived STS credentials.

---

# 46. AWS security practices

A senior answer should describe controls by category.

```text
Identity:
  IAM Identity Center
  roles
  MFA
  least privilege
  SCPs

Network:
  private subnets
  SG/NACL
  WAF/Shield
  VPC endpoints
  segmentation

Data:
  KMS encryption
  TLS
  backups
  S3 Block Public Access

Detection:
  CloudTrail
  GuardDuty
  Security Hub
  Config
  VPC Flow Logs

Vulnerability:
  Inspector
  image scanning
  patch management

Recovery:
  backups
  immutable recovery
  DR testing
```

---

# 47. How do you scan AWS EC2 vulnerabilities?

A typical native solution is Amazon Inspector for supported EC2 workloads/container images, combined with Systems Manager/Patch Manager where appropriate.

Then operationalize results:

```text
Finding
  |
severity + exploitability
  |
ticket/remediation SLA
  |
patch/rebuild AMI
  |
redeploy
  |
rescan
```

Prefer immutable replacement over repeatedly hand-patching long-lived production servers where your architecture permits it.

---

# 48. EC2 production security parameters

Before production:

- private subnet unless public exposure is necessary
- least-privilege instance role
- IMDSv2
- encrypted EBS
- restricted SG
- patch baseline
- hardened AMI
- Systems Manager instead of exposing SSH where possible
- monitoring
- logs
- EDR/Inspector as required
- backups
- termination protection for appropriate critical systems
- tags
- no secrets in user data
- disable unused services

---

# 49. Private EC2 inaccessible and SSH isn't working

Use AWS Systems Manager Session Manager if already configured:

```text
Operator
   |
SSM API
   |
SSM Agent on EC2
```

No inbound port 22 is needed.

If the instance itself is broken, troubleshoot:

- instance status checks
- route tables
- NACL
- SSM Agent
- disk full
- filesystem corruption
- OS firewall
- bad sshd config
- CPU/memory exhaustion

Recovery methods may include EC2 Serial Console or attaching the root EBS volume to a recovery instance, depending on OS and configuration.

---

# 50. EC2 vs Lambda

Use EC2 when:

- long-running server/application
- custom OS/kernel requirements
- persistent processes
- specialized software/networking
- workload doesn't fit serverless execution model.

Use Lambda when:

- event-driven
- short-lived execution
- bursty workload
- little/no infrastructure management desired.

Example:

```text
Nightly resize uploaded images
S3 event -> Lambda -> processed S3 object
```

versus:

```text
legacy commercial application requiring OS daemon
-> EC2
```

---

# 51. Lambda example

Python:

```python
import json

def lambda_handler(event, context):
    name = event.get("name", "World")

    return {
        "statusCode": 200,
        "body": json.dumps({
            "message": f"Hello, {name}"
        })
    }
```

---

# 52. Lambda logs aren't coming to CloudWatch

Check:

1. Was the function actually invoked?
2. Is logging code executed?
3. Does execution role have:
   - `logs:CreateLogGroup`
   - `logs:CreateLogStream`
   - `logs:PutLogEvents`
4. Is the expected log group correct?
5. Any CloudWatch quota/throttling issue?
6. Is custom logging destination configured?

Default Lambda logs normally go to:

```text
/aws/lambda/<function-name>
```

provided the execution role has the required permissions. [AWS Documentation](https://docs.aws.amazon.com/lambda/latest/dg/monitoring-cloudwatchlogs.html?utm_source=chatgpt.com)

---

# 53. CloudWatch Log Group vs CloudTrail

This interview question is sometimes phrased incorrectly as “log group and log trail.”

**CloudWatch Logs / Log Group** stores application/system/service log streams.

**CloudTrail** records AWS API activity such as:

```text
Who called DeleteBucket?
From which identity?
When?
From which IP?
What API action?
```

A CloudTrail **trail** can deliver events to S3 and CloudWatch Logs.

---

# 54. Route 53 TTL

TTL means **Time To Live**.

```text
Client
  |
Recursive DNS resolver
  |
Route53 authoritative DNS
```

Suppose:

```text
app.example.com -> 10.0.0.5
TTL = 300
```

A resolver may cache that answer for about 300 seconds.

Low TTL:

- quicker DNS changes/failover
- more DNS queries.

High TTL:

- more caching
- slower propagation of record changes.

Before a planned migration, teams sometimes reduce TTL in advance.

---

# 55. Weighted routing

If referring to **Route 53**:

```text
Record A -> endpoint v1 weight 90
Record B -> endpoint v2 weight 10
```

DNS answers are distributed approximately according to weights.

Useful for:

- canary releases
- traffic migration
- multi-region distribution.

If the interviewer means **ALB weighted target groups**, the forwarding rule can distribute traffic among multiple target groups with configured weights.

Clarify which layer they mean.

---

# 56. SSL/TLS flow

Simplified TLS handshake:

```text
Client ---------------------- Server
  | --- ClientHello --------> |
  |                           |
  | <-- ServerHello + cert -- |
  |                           |
  | verify CA + hostname      |
  | key agreement             |
  |                           |
  | ===== encrypted data ==== |
```

A certificate establishes the server's authenticated identity through a certificate authority trust chain.

### Let's Encrypt / Certbot

Typical ACME flow:

```text
Certbot
 |
register/order certificate
 |
Let's Encrypt asks for challenge
 |
HTTP-01 or DNS-01 proof
 |
CA validates domain control
 |
certificate issued
 |
webserver installs cert
 |
scheduled renewal
```

### AWS

Common AWS-managed pattern:

```text
Client -> HTTPS -> ALB/CloudFront/API Gateway
                    |
                 ACM cert
```

AWS Certificate Manager can manage certificate lifecycle for integrated AWS services.

---

# 57. GSLB

Global Server Load Balancing generally distributes users across geographically separate sites/regions.

```text
                  Global DNS/GSLB
                  /             \
             Region A         Region B
               LB               LB
              / \              / \
```

Selection may be based on:

- health
- geography
- latency
- policy
- weights.

DNS-based implementations typically return the preferred regional endpoint rather than forwarding the request themselves.

---

# 58. OSI model

```text
7 Application   HTTP, DNS, SMTP
6 Presentation  encoding/encryption concepts
5 Session       session management
4 Transport     TCP, UDP
3 Network       IP, routing
2 Data Link     Ethernet, MAC
1 Physical      cable/radio/signals
```

Troubleshooting example:

```text
L1: interface/link up?
L2: VLAN/ARP?
L3: IP/route?
L4: port listening/TCP?
L7: HTTP response/application?
```

Senior interviewers value how you use the model to troubleshoot more than memorizing seven names.

---

# 59. Packet flow from user to Kubernetes application

Example EKS flow:

```text
User browser
   |
DNS / Route53
   |
Internet
   |
AWS ALB
   |
Ingress/Gateway rule
   |
Kubernetes Service
   |
EndpointSlice
   |
Pod IP
   |
Application container
   |
Database/cache
```

Return traffic traverses the corresponding networking stack.

For debugging, test each boundary instead of saying “Kubernetes networking problem.”

---

# 60. AWS cost optimization

I divide it into measurement, elimination, rightsizing, commitment and architecture.

```text
Cost Explorer / CUR
      |
identify biggest spend
      |
idle resources?
      |
rightsizing?
      |
architecture change?
      |
commitment discounts?
      |
measure again
```

Examples:

- delete unattached EBS/snapshots where policy permits
- shut down unused non-prod resources
- right-size EC2/RDS
- Auto Scaling
- Spot for fault-tolerant workloads
- Savings Plans/Reserved capacity for predictable baselines
- S3 lifecycle policies
- reduce NAT/data-transfer inefficiencies
- log-retention policies
- Karpenter/bin-packing
- Graviton where compatible
- CloudFront caching
- budget/anomaly alerts.

In interviews quantify a real result if you genuinely have one:

> “We reduced compute spend by approximately X% by rightsizing, scheduling non-production environments and moving fault-tolerant workers to Spot.”

Use your actual number.

---

# 61. AWS migration approach

A senior answer:

```text
Discovery
 |
Dependency mapping
 |
6R/7R decision
 |
Landing zone
 |
Network + IAM
 |
Data replication
 |
Pilot migration
 |
Performance/security testing
 |
Cutover
 |
DNS switch
 |
Hypercare
 |
Decommission old environment
```

For low downtime:

```text
Initial bulk replication
       |
continuous delta sync
       |
freeze/change window
       |
final sync
       |
validation
       |
DNS/traffic switch
```

Discuss rollback criteria **before** migration.

---

# 62. RDS design

Consider:

- engine/version
- CPU/memory
- storage size and IOPS
- expected connections
- read/write ratio
- Multi-AZ requirement
- backup/PITR
- encryption
- parameter groups
- subnet group
- SG
- monitoring
- maintenance
- read replicas
- connection pooling.

### “How many masters and slaves in RDS?”

Do not use generic “1 master + N slaves” for every RDS architecture.

Answer according to the actual engine/deployment:

> “Our standard RDS MySQL Multi-AZ architecture has a primary and standby for HA; read replicas are separately created for read scaling.”

Use AWS terminology such as **primary, standby and read replica**.

### “How much data is in your RDS?”

Use the actual value:

> “The production database is approximately `<actual GB/TB>`, with `<actual growth>` monthly, and we provision `<storage>` with appropriate IOPS.”

Don't fabricate it.

---

# 63. RDS migration with minimal downtime

One approach:

```text
Source DB
  |
initial load
  |
AWS DMS/replication
  |
continuous CDC
  |
Target RDS
  |
validate
  |
short write freeze
  |
replication lag -> ~0
  |
application cutover
```

For supported RDS topology changes, AWS also documents read-replica based reduced-downtime migration patterns where writes continue during replication before promotion. [AWS Documentation](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/multi-az-db-clusters-migrating-to-with-read-replica.html?utm_source=chatgpt.com)

---

# 64. AWS Elastic Disaster Recovery

Simplified architecture:

```text
Source servers
     |
DRS replication agent
     |
continuous block replication
     |
AWS staging area
     |
Recovery point/state
     |
on disaster
     v
Launch EC2 recovery instances
     |
Route53/traffic cutover
```

AWS distinguishes:

- **recovery:** DRS launches recovery instances
- **failover:** you redirect production traffic
- **failback:** DRS helps replicate back to the source environment. [AWS Documentation](https://docs.aws.amazon.com/drs/latest/userguide/failback.html?utm_source=chatgpt.com)


Always conduct recovery drills rather than assuming DR works.

---

# 65. Terraform init, plan, apply

Core workflow:

```bash
terraform init
terraform fmt -check
terraform validate
terraform plan -out=tfplan
terraform apply tfplan
```

`init`:

- initializes backend
- downloads providers/modules
- creates/updates provider dependency lock file.

`plan`:

- compares configuration/state/real infrastructure
- creates proposed execution plan.

`apply`:

- executes approved changes.

HashiCorp describes init → plan → apply as the core workflow. [HashiCorp Developer](https://developer.hashicorp.com/terraform/tutorials/cli/init?utm_source=chatgpt.com)

---

# 66. Terraform state

Terraform state maps configuration addresses to real infrastructure objects.

Example:

```text
aws_instance.web
      |
      v
i-0abc123...
```

By default it uses `terraform.tfstate`, but for a team use a supported remote backend/HCP Terraform with appropriate access controls and locking. HashiCorp recommends remote state for collaboration. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/state?utm_source=chatgpt.com)

Do not commit `terraform.tfstate` to Git; it can contain sensitive values. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/style?utm_source=chatgpt.com)

---

# 67. State locking and conflicts

When the backend supports locking:

```text
Pipeline A -> acquires lock -> terraform apply
Pipeline B -> lock unavailable -> waits/fails
```

This prevents concurrent writers from corrupting state.

Terraform automatically locks state for operations that can write it when supported by the backend. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/state/locking?utm_source=chatgpt.com)

Only use:

```bash
terraform force-unlock <LOCK_ID>
```

after proving the original writer is no longer active.

Never casually run:

```bash
-lock=false
```

on a shared production state.

---

# 68. What if tfstate is lost?

### With backup/versioned remote backend

Restore the correct state version.

Then:

```bash
terraform plan
```

and verify that Terraform sees the expected resources.

### No backup anywhere

Do **not** immediately run `terraform apply`; Terraform may think existing infrastructure needs to be created again.

Rebuild the state by importing resources:

```hcl
import {
  to = aws_instance.app
  id = "i-0123456789abcdef0"
}
```

Then:

```bash
terraform plan
terraform apply
```

Terraform supports configuration-driven import, and HashiCorp recommends planning before applying the import. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/import/single-resource?utm_source=chatgpt.com)

For a large estate:

1. inventory cloud resources;
2. recover matching Terraform configuration;
3. import resources systematically;
4. reconcile drift;
5. only then resume normal deployments.

---

# 69. Create EC2 B without deleting EC2 A

If A currently exists in state, simply add a second address rather than repurposing A's address.

```hcl
resource "aws_instance" "a" {
  ami           = var.ami
  instance_type = "t3.micro"
}

resource "aws_instance" "b" {
  ami           = var.ami
  instance_type = "t3.micro"
}
```

Or use `for_each`:

```hcl
resource "aws_instance" "server" {
  for_each = {
    a = "t3.micro"
    b = "t3.small"
  }

  ami           = var.ami
  instance_type = each.value
}
```

`for_each` creates separate resource instances keyed by map/set entries. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/meta-arguments/for_each?utm_source=chatgpt.com)

Always inspect:

```bash
terraform plan
```

If plan says destroy A unexpectedly, stop and understand the resource-address change.

---

# 70. Terraform map of objects

```hcl
variable "servers" {
  type = map(object({
    instance_type = string
    subnet_id     = string
  }))

  default = {
    api = {
      instance_type = "t3.small"
      subnet_id     = "subnet-111"
    }
    worker = {
      instance_type = "t3.medium"
      subnet_id     = "subnet-222"
    }
  }
}

resource "aws_instance" "server" {
  for_each = var.servers

  ami           = var.ami
  instance_type = each.value.instance_type
  subnet_id     = each.value.subnet_id

  tags = {
    Name = each.key
  }
}
```

---

# 71. Terraform provisioners

Common built-in patterns:

```text
local-exec
remote-exec
file
```

Example:

```hcl
provisioner "local-exec" {
  command = "echo ${self.private_ip}"
}
```

But the correct interview point is:

> Provisioners are a last resort. Prefer cloud-init/user data, configuration management, image baking or provider-native resources.

HashiCorp explicitly documents provisioners as post-apply operations and provides `terraform_data` for provisioners not tied to a real resource. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/provisioners?utm_source=chatgpt.com)

---

# 72. Run something every Terraform apply

Historically candidates show:

```hcl
resource "null_resource" "always" {
  triggers = {
    always_run = timestamp()
  }

  provisioner "local-exec" {
    command = "./script.sh"
  }
}
```

Because `timestamp()` changes, the resource is replaced each run.

Modern Terraform also provides `terraform_data` and `triggers_replace` for lifecycle-driven helper operations. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/provisioners?utm_source=chatgpt.com)

Be cautious: making side-effect commands run every apply hurts idempotency.

---

# 73. Terraform `.terraform.lock.hcl`

This is **not the state lock**.

`.terraform.lock.hcl` is the dependency lock file recording selected provider versions/checksums so subsequent initialization is reproducible. HashiCorp recommends committing it to version control. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/files/dependency-lock?utm_source=chatgpt.com)

Distinguish:

```text
terraform.tfstate     -> infrastructure state
.terraform.lock.hcl  -> dependency/provider selections
state lock            -> prevents concurrent state writes
```

---

# 74. `terraform fmt` vs `validate`

`terraform fmt`:

```text
normalizes Terraform code formatting
```

`terraform validate`:

```text
checks whether configuration is syntactically and internally valid
```

Typical CI:

```bash
terraform fmt -check -recursive
terraform init -backend=false
terraform validate
```

HashiCorp recommends using `validate` in addition to formatting. [HashiCorp Developer](https://developer.hashicorp.com/terraform/tutorials/configuration-language/troubleshooting-workflow?utm_source=chatgpt.com)

---

# 75. Terraform multiple S3 buckets

```hcl
variable "buckets" {
  default = [
    "company-app-dev-example",
    "company-app-qa-example",
    "company-app-prod-example"
  ]
}

resource "aws_s3_bucket" "bucket" {
  for_each = toset(var.buckets)

  bucket = each.value
}
```

---

# 76. Terraform EC2 in public subnet

Skeleton:

```hcl
provider "aws" {
  region = "ap-south-1"
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = true
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

resource "aws_security_group" "web" {
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "web" {
  ami                         = var.ami
  instance_type               = "t3.micro"
  subnet_id                   = aws_subnet.public.id
  vpc_security_group_ids      = [aws_security_group.web.id]
  associate_public_ip_address = true

  user_data = <<-EOF
              #!/bin/bash
              dnf install -y httpd
              systemctl enable --now httpd
              EOF
}
```

For production I normally avoid public application EC2 instances and place them behind a load balancer.

---

# 77. Terraform module output

Network module:

```hcl
output "subnet_id" {
  value = azurerm_subnet.app.id
}
```

Root module:

```hcl
module "network" {
  source = "./modules/network"
}

module "vm" {
  source    = "./modules/vm"
  subnet_id = module.network.subnet_id
}
```

Terraform exposes child module outputs as:

```text
module.<module_name>.<output_name>
``` :chatgpt-content-reference{index="41"}


---

# 78. Docker architecture

```text
Docker CLI
   |
Docker API
   |
Docker daemon
   |
containerd
   |
OCI runtime
   |
Linux namespaces/cgroups
   |
Containers
```

Docker images are layered immutable templates; containers add a writable layer and runtime isolation.

---

# 79. Dockerfile

Example:

```dockerfile
FROM python:3.13-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

USER 10001

EXPOSE 8080

CMD ["python", "app.py"]
```

A Dockerfile contains instructions Docker uses to build an image. Common instructions include `FROM`, `RUN`, `WORKDIR`, `COPY`, `ENV`, `EXPOSE`, `USER`, `CMD` and `ENTRYPOINT`. [Docker Documentation](https://docs.docker.com/reference/dockerfile?utm_source=chatgpt.com)

---

# 80. COPY vs ADD

Use `COPY` for ordinary local build-context files.

```dockerfile
COPY app.jar /app/
```

`ADD` has extra functionality, including sources such as remote locations and additional handling behavior depending on source.

Default interview recommendation:

> Prefer `COPY` unless you specifically need an `ADD` feature, because intent is clearer.

Docker documents both instructions and their differing capabilities. [Docker Documentation](https://docs.docker.com/reference/dockerfile?utm_source=chatgpt.com)

---

# 81. CMD vs ENTRYPOINT

```dockerfile
ENTRYPOINT ["python", "app.py"]
CMD ["--port", "8080"]
```

`ENTRYPOINT` defines the executable.

`CMD` commonly provides default arguments.

Running:

```bash
docker run myimage --port 9090
```

overrides the CMD arguments while retaining the exec-form entrypoint.

Docker documents `ENTRYPOINT` as the configured executable and `CMD` as default command/arguments that users can override. [Docker Documentation](https://docs.docker.com/reference/dockerfile?utm_source=chatgpt.com)

---

# 82. Can you make a Docker image without a base image?

Yes, using:

```dockerfile
FROM scratch
```

This is practical for static binaries or specially constructed root filesystems.

Example:

```dockerfile
FROM scratch
COPY app /app
ENTRYPOINT ["/app"]
```

---

# 83. Reduce Docker image size

Use:

- smaller suitable base image
- multi-stage builds
- `.dockerignore`
- only runtime dependencies
- remove package-manager caches
- avoid copying build tools
- combine temporary install/remove operations appropriately
- do not store logs/artifacts in image
- use language-specific dependency pruning.

Example:

```dockerfile
FROM golang:1.25 AS build
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o /app

FROM scratch
COPY --from=build /app /app
ENTRYPOINT ["/app"]
```

---

# 84. Container port mismatch

`EXPOSE 8080` does not magically reconfigure your application.

If application listens on 9090 while Dockerfile declares:

```dockerfile
EXPOSE 8080
```

the app still listens on 9090.

Correct mapping might be:

```bash
docker run -p 8080:9090 myimage
```

Meaning:

```text
host 8080 -> container 9090
```

In Kubernetes ensure `targetPort` reaches the application's real listening port.

---

# 85. Docker consuming too much disk

Inspect:

```bash
docker system df
docker ps -a
docker images
docker volume ls
```

Clean only unused objects after understanding what is safe:

```bash
docker image prune
docker container prune
docker builder prune
```

Avoid blindly running aggressive prune commands on production hosts.

Also check application logs and configure rotation.

---

# 86. How many containers can run in a Pod?

Kubernetes does not define a normal design rule such as “maximum 2.”

A Pod can contain multiple containers, constrained by node/Pod resources and platform limits.

But use multiple containers only when they are tightly coupled.

4-5 container example:

```text
Pod
 |
 +-- main application
 +-- service-mesh proxy
 +-- log/telemetry sidecar
 +-- configuration/reload helper
 +-- security agent/helper
```

Modern Kubernetes also supports patterns such as sidecar containers, but avoid turning a Pod into a “mini VM” containing unrelated applications.

---

# 87. Git push vs pull

```bash
git push
```

sends local refs/commits to remote.

```bash
git pull
```

fetches remote changes and integrates them into the current branch, typically via merge or configured rebase behavior.

Conceptually:

```text
git pull ≈ git fetch + integration
```

---

# 88. `git fetch` vs `git pull`

`fetch`:

```text
download remote refs/objects
do not automatically change current branch
```

`pull`:

```text
fetch + merge/rebase into current branch
```

For careful production/release work I often prefer:

```bash
git fetch origin
git log HEAD..origin/main
```

before deciding how to integrate.

---

# 89. Merge vs rebase

Merge:

```text
A---B---C main
     \   \
      D---E---M
```

preserves both histories and creates a merge commit where required.

Rebase:

```text
A---B---C main
         \
          D'---E'
```

rewrites your branch commits onto a new base.

Avoid rebasing shared published history unless your team's process intentionally supports it.

---

# 90. Tags

Tags identify important commits, commonly releases:

```bash
git tag -a v2.3.0 -m "Release 2.3.0"
git push origin v2.3.0
```

Annotated/signed tags are useful for releases and traceability.

---

# 91. Git branching strategy

Example:

```text
main ---------------- production
  \
   feature/* -------- short-lived features

release tags:
v1.2.0
v1.3.0
```

For high-frequency DevOps teams I generally prefer trunk-based/short-lived feature branches over very long-lived branching unless release governance requires something else.

The important interview answer is your governance:

- PR required
- review
- status checks
- protected main
- automated tests
- signed/reviewed releases
- tags.

---

# 92. `--force` vs `--force-with-lease`

```bash
git push --force
```

overwrites the remote ref regardless of whether someone else pushed changes you haven't incorporated.

```bash
git push --force-with-lease
```

only updates if the remote ref still matches what you believe it to be.

Therefore prefer `--force-with-lease` after an intentional rebase when force-pushing cannot be avoided.

---

# 93. Delete last two Git commits

If they are **local/unpublished**:

```bash
git reset --hard HEAD~2
```

If already shared and you should preserve public history, prefer reverting:

```bash
git revert <commit2>
git revert <commit1>
```

Do not casually rewrite shared production history.

---

# 94. Jenkins Declarative vs Scripted

Declarative:

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh 'mvn package'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
    }
}
```

Scripted:

```groovy
node {
    stage('Build') {
        sh 'mvn package'
    }

    stage('Test') {
        sh 'mvn test'
    }
}
```

Declarative is more structured/opinionated. Scripted exposes more Groovy programming flexibility. Jenkins officially supports both syntaxes. [Jenkins](https://www.jenkins.io/doc/book/pipeline/syntax/?utm_source=chatgpt.com)

---

# 95. CI/CD pipeline design

A mature pipeline:

```text
Checkout
  |
Compile/build
  |
Unit test
  |
SAST
  |
dependency scan
  |
build container
  |
image scan
  |
sign/provenance
  |
push ECR
  |
deploy dev
  |
integration test
  |
approval
  |
deploy prod
  |
smoke test
  |
metrics gate
  |
success / rollback
```

Security measures include:

- no long-lived cloud credentials where OIDC is available
- secret manager
- least-privilege runners
- dependency scanning
- SAST
- container scanning
- IaC scanning
- signed artifacts
- protected branches
- approvals
- audit logs
- immutable artifact promotion.

---

# 96. Manual approval in Jenkins

Declarative:

```groovy
stage('Approve Production') {
    input {
        message "Deploy to production?"
        ok "Deploy"
        submitter "release-managers"
    }

    steps {
        sh './deploy-prod.sh'
    }
}
```

Jenkins provides an `input` directive/step to pause a pipeline until authorized input is provided. [Jenkins](https://www.jenkins.io/doc/book/pipeline/syntax/?utm_source=chatgpt.com)

---

# 97. Jenkins credentials

Store the secret in Jenkins Credentials—not in Jenkinsfile.

Example:

```groovy
environment {
    REGISTRY_CREDS = credentials('registry-creds')
}
```

Jenkins stores supported credentials encrypted on the controller and exposes them to jobs through credential IDs/bindings. [Jenkins](https://www.jenkins.io/doc/book/pipeline/jenkinsfile/?utm_source=chatgpt.com)

For AWS, modern pipelines can instead use workload/OIDC-based short-lived credentials where supported.

---

# 98. SonarQube integration

Pipeline concept:

```text
Checkout
  |
Build
  |
Unit tests + coverage
  |
SonarQube analysis
  |
Quality Gate
  |
pass? -> build/publish/deploy
fail? -> stop
```

A sensible policy is to run quality checks on PRs to stop bad code before merge, with branch/release analysis according to your project's SonarQube model.

---

# 99. GitHub Actions workflow

```yaml
name: ci

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
  workflow_dispatch:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@<pinned-sha>

      - name: Test
        run: ./run-tests.sh

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@<pinned-sha>
      - run: docker build -t app:${{ github.sha }} .
```

Key concepts:

```text
on       -> triggers
jobs     -> major execution units
runs-on  -> runner selection
steps    -> actions/commands
needs    -> dependencies
matrix   -> generate job combinations
```

For third-party actions, GitHub recommends pinning to a full-length commit SHA for immutable execution. [GitHub Docs](https://docs.github.com/en/actions/reference/security/secure-use?learn=getting_started\&learnProduct=actions\&utm_source=chatgpt.com)

---

# 100. Matrix and `needs`

Matrix:

```yaml
strategy:
  matrix:
    python: ["3.12", "3.13"]
```

creates multiple variants.

`needs`:

```yaml
deploy:
  needs:
    - build
    - security
```

makes `deploy` wait for required jobs.

---

# 101. Self-hosted vs GitHub-hosted runners

Use GitHub-hosted where:

- standard environment is enough
- ephemeral clean VMs are desirable
- no private-only environment requirements.

Use self-hosted where:

- private network connectivity required
- specialized hardware
- custom tooling
- larger compute
- regulatory constraints.

With self-hosted runners **you own the OS/software/security lifecycle**. GitHub explicitly notes this responsibility. [GitHub Docs](https://docs.github.com/en/actions/concepts/runners/self-hosted-runners?ref=localhost\&utm_source=chatgpt.com)

For private workloads:

```yaml
runs-on:
  - self-hosted
  - linux
  - production
```

GitHub routes jobs according to runner labels/groups. [GitHub Docs](https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/use-in-a-workflow?apiVersion=2022-11-28\&utm_source=chatgpt.com)

---

# 102. GitHub OIDC vs IAM user credentials

Preferred:

```text
GitHub Actions
   |
GitHub OIDC JWT
   |
AWS STS
   |
short-lived role credentials
```

rather than:

```text
GitHub Secret
   |
permanent AWS_ACCESS_KEY_ID
```

GitHub officially documents AWS OIDC specifically to avoid storing long-lived AWS credentials in GitHub secrets. [GitHub Docs](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws?apiVersion=2022-11-28\&utm_source=chatgpt.com)

---

# 103. Ansible playbook

Example:

```yaml
---
- name: Configure web servers
  hosts: web
  become: true

  vars:
    package_name: nginx

  tasks:
    - name: Install nginx
      ansible.builtin.package:
        name: "{{ package_name }}"
        state: present

    - name: Start nginx
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true
```

Run:

```bash
ansible-playbook -i inventory.ini web.yml
```

Ansible playbooks define tasks executed against inventory hosts. [Ansible Documentation](https://docs.ansible.com/projects/ansible/latest/cli/ansible-playbook.html?utm_source=chatgpt.com)

### Client prerequisites

Typically:

- reachable SSH/WinRM
- authentication
- supported Python where modules require it
- privilege escalation permissions
- inventory information
- network/firewall access.

---

# 104. Inventory

```ini
[web]
web01 ansible_host=10.0.1.10
web02 ansible_host=10.0.1.11

[database]
db01 ansible_host=10.0.2.10

[production:children]
web
database
```

Use variables rather than duplicating environment-specific content throughout playbooks.

---

# 105. Ansible roles

Typical role:

```text
roles/nginx/
  defaults/
  vars/
  tasks/
  handlers/
  templates/
  files/
  meta/
```

Purpose: reusable, structured automation.

Play:

```yaml
- hosts: web
  roles:
    - nginx
```

---

# 106. Import vs include in Ansible

General distinction:

```text
import_*  -> static processing
include_* -> dynamic at runtime
```

Dynamic includes are useful when runtime conditions determine what content is included; static imports provide a more preprocessed structure.

---

# 107. One Ansible host times out

Do not increase the global timeout first.

Test:

```bash
ansible host01 -i inventory -m ping -vvvv
ssh -vvv host01
```

Check:

- DNS
- route
- firewall
- SSH port
- credentials
- host key
- Python
- load
- disk/inodes
- sudo
- connection count.

Ansible supports `-T/--timeout` but timeout tuning should follow root-cause analysis. [Ansible Documentation](https://docs.ansible.com/projects/ansible/latest/cli/ansible-playbook.html?utm_source=chatgpt.com)

---

# 108. Linux load average

`uptime`:

```text
load average: 1.52, 1.34, 0.91
```

These represent approximately the **1-, 5- and 15-minute load averages**.

On Linux, load reflects tasks that are runnable plus certain uninterruptible waits, so high load is not necessarily only “high CPU.”

Interpret relative to CPU capacity:

```text
4 CPU system
load ~4 -> roughly full runnable capacity
load >>4 -> contention/waiting likely
```

Then inspect:

```bash
top
vmstat 1
iostat -xz 1
pidstat 1
mpstat -P ALL 1
```

---

# 109. Server performance troubleshooting

Use the USE-like mindset:

```text
CPU:
  utilization
  saturation
  errors

Memory:
  available memory
  swap
  OOM

Disk:
  latency
  queue
  utilization
  capacity/inodes

Network:
  throughput
  drops
  retransmissions
```

Commands:

```bash
uptime
top
vmstat 1
free -m
mpstat -P ALL 1
iostat -xz 1
df -h
df -ih
ss -s
sar -n DEV 1
dmesg
journalctl
```

Correlate with when the problem started and deployment/config changes.

---

# 110. `/var` is 90% full

Immediate response:

```bash
df -h /var
df -ih /var
du -xhd1 /var | sort -h
```

Identify:

- logs
- container layers
- package cache
- core dumps
- deleted-open files.

Check deleted files still held open:

```bash
lsof +L1
```

Then:

- rotate/compress safely
- increase storage if required
- fix retention/logging root cause
- set alerting.

Do not start with:

```bash
rm -rf /var/log/*
```

---

# 111. Filesystem 50% full but can't write; `df -ih` full

That means inode exhaustion.

Disk blocks:

```bash
df -h
```

Inodes:

```bash
df -ih
```

Millions of tiny files can consume all inodes even when capacity remains.

Find:

```bash
find /path -xdev -type f | wc -l
```

Then locate the workload creating excessive files and fix its retention/design.

---

# 112. LVM add 50 GB to `/opt` online

Example sequence:

```bash
lsblk
pvs
vgs
lvs
df -Th /opt
```

If new device `/dev/xvdb`:

```bash
pvcreate /dev/xvdb
vgextend vgdata /dev/xvdb
lvextend -L +50G /dev/vgdata/lvopt
```

For XFS:

```bash
xfs_growfs /opt
```

For ext4:

```bash
resize2fs /dev/vgdata/lvopt
```

Often `lvextend -r` can coordinate filesystem resizing where supported:

```bash
lvextend -r -L +50G /dev/vgdata/lvopt
```

Validate first; device names vary.

---

# 113. Attach/detach filesystem

Attach/mount:

```bash
mount /dev/mapper/vg-lv /opt
```

Persistent configuration:

```text
/etc/fstab
```

Detach:

```bash
umount /opt
```

If busy:

```bash
lsof +D /opt
fuser -vm /opt
```

Stop/relocate the process before unmounting.

---

# 114. Last 15 lines

```bash
tail -n 15 file.log
```

Total lines:

```bash
wc -l file.txt
```

Count `ERR` lines:

```bash
grep -c 'ERR' app.log
```

Count every occurrence including multiple per line:

```bash
grep -o 'ERR' app.log | wc -l
```

---

# 115. Passwordless authentication between Linux servers

Create key:

```bash
ssh-keygen -t ed25519
```

Copy public key:

```bash
ssh-copy-id user@server2
```

Then:

```bash
ssh user@server2
```

For automation use dedicated accounts/keys, proper permissions, rotation and restricted privileges.

---

# 116. Create user without SSH login

Depending on the requirement:

```bash
useradd -s /usr/sbin/nologin appuser
```

or:

```bash
usermod -s /usr/sbin/nologin appuser
```

That is useful for service accounts that should not receive interactive shell access.

---

# 117. Zombie process

A zombie has exited but its parent has not yet collected the exit status.

Identify:

```bash
ps aux | awk '$8 ~ /Z/'
ps -eo pid,ppid,state,cmd | grep ' Z '
```

You cannot “kill” an already dead zombie. Address the parent process so it calls `wait()`, restart/fix the parent if appropriate.

---

# 118. Prometheus and Grafana architecture

```text
Application / Node Exporter
        |
     /metrics
        |
    Prometheus
       /   \
   TSDB     Rules
             |
        Alertmanager
             |
        Slack/email/etc.

Prometheus
    |
Grafana data source
    |
Dashboards
```

Grafana treats Prometheus as a data source and uses PromQL-backed panels/alerts. [Grafana Labs](https://grafana.com/docs/grafana/latest/fundamentals/getting-started/first-dashboards/get-started-grafana-prometheus/?utm_source=chatgpt.com)

For Kubernetes I commonly deploy via a maintained Helm stack and monitor:

- node CPU/memory/disk/network
- Pod CPU/memory
- restart count
- readiness
- API-server metrics
- scheduler/controller metrics where available
- application RED metrics
- business KPIs.

---

# 119. Grafana dashboard

Typical steps:

1. configure data source;
2. create dashboard;
3. add panel;
4. select Prometheus;
5. write PromQL;
6. choose visualization;
7. add variables;
8. set thresholds/units;
9. save;
10. provision as code where appropriate.

Grafana's documented workflow is dashboard → visualization → data source → query → save. [Grafana Labs](https://grafana.com/docs/grafana/latest/visualizations/dashboards/build-dashboards/create-dashboard/?utm_source=chatgpt.com)

CPU example:

```promql
100 -
(
  avg by(instance)
  (
    rate(node_cpu_seconds_total{mode="idle"}[5m])
  ) * 100
)
```

Pod memory example:

```promql
sum by(namespace, pod) (
  container_memory_working_set_bytes{container!=""}
)
```

---

# 120. SLI, SLO, SLA

**SLI** = what you measure.

Example:

```text
successful HTTP requests / total requests
```

**SLO** = internal reliability objective.

```text
99.9% successful requests over 30 days
```

**SLA** = external contractual commitment, potentially with consequences.

If SLO = 99.9%, allowed unavailability is roughly 0.1% of the measurement period; the engineering concept tied to this is an **error budget**.

---

# 121. ELK architecture

```text
Application logs
     |
Filebeat / Fluent Bit / agent
     |
optional Logstash
     |
Elasticsearch
     |
Kibana
```

Use structured JSON where possible:

```json
{
  "timestamp": "2026-10-12T10:00:00Z",
  "level": "ERROR",
  "service": "payments",
  "request_id": "abc123",
  "message": "payment timeout"
}
```

Then map fields so you can filter by `service`, `level`, `request_id`, etc.

---

# 122. Airflow job failed

Start with the task instance rather than the whole DAG.

```text
Failed task
  |
task logs
  |
exception/root cause
  |
scheduler/worker state
  |
dependencies/upstream
  |
external service
  |
retry/remediate
```

Check:

- task logs
- scheduler logs
- worker/executor logs
- DAG parsing
- credentials
- connection
- timeout
- memory/OOM
- external dependency
- retries
- task heartbeat
- queue state.

Airflow's official troubleshooting guide specifically calls out scheduler timeout, heartbeat and externally changed task state as causes of failures. [Apache Airflow](https://airflow.apache.org/docs/apache-airflow/stable/troubleshooting.html?utm_source=chatgpt.com)

---

# 123. SRE Sev-1 handling

My sequence:

```text
Detect
 |
declare severity
 |
assign Incident Commander
 |
stabilize/mitigate
 |
parallel diagnosis
 |
stakeholder communication
 |
restore service
 |
validate
 |
post-incident RCA
 |
corrective actions
```

During Sev-1:

- stop unrelated risky changes
- prioritize restoration over perfect diagnosis
- maintain one source of truth
- time-stamp major events
- communicate at a fixed cadence
- make reversible changes
- preserve evidence.

Afterward:

```text
Timeline
Impact
Trigger
Contributing factors
Why controls failed
Corrective actions
Owners
Due dates
```

Avoid “human error” as the root cause; ask why the system allowed one mistake to produce the outage.

---

# 124. Sudden traffic surge makes application unresponsive

I would first distinguish legitimate traffic from an attack.

```text
Client surge
   |
CDN/WAF?
   |
Load Balancer
   |
App capacity
   |
DB/cache/downstream capacity
```

Immediate checks:

- request rate
- 4xx/5xx
- latency
- ALB target response time
- CPU/memory
- HPA
- pending Pods
- DB connections/CPU
- cache hit ratio
- downstream errors.

Mitigations:

- scale replicas/nodes
- enable/ramp cache
- rate limiting
- shed optional workload
- protect DB with pooling
- queue asynchronous jobs
- WAF/block abusive patterns
- degrade non-essential features.

Then find why automatic capacity controls failed.

---

# 125. Example Linux RCA answer

Use STAR.

> “We received latency alerts while CPU remained normal. `iostat` showed high device await and disk utilization, while `df -h` showed adequate capacity. I correlated the start time with a new backup job that was saturating the volume. We paused the job, latency recovered, then moved backups to a separate volume/time window and added disk-latency alerts. The RCA was storage saturation caused by competing I/O, not CPU.”

This demonstrates a diagnostic chain rather than saying “I restarted the server.”

---

# 126. Shell: delete files older than 30 days

Safer preview:

```bash
find /var/app/archive -type f -mtime +30 -print
```

Then delete:

```bash
find /var/app/archive -type f -mtime +30 -delete
```

Always confirm the directory and matching criteria before destructive execution.

---

# 127. Shell: monitor disk >80%

```bash
#!/usr/bin/env bash

THRESHOLD=80
LOG=/var/log/disk-monitor.log

usage=$(df -P / | awk 'NR==2 {gsub("%","",$5); print $5}')

if (( usage >= THRESHOLD )); then
    msg="$(date '+%F %T') Disk usage is ${usage}% on $(hostname)"
    echo "$msg" >> "$LOG"

    printf '%s\n' "$msg" |
      mail -s "Disk alert: $(hostname)" ops@example.com
fi
```

In production I would usually send this through the organization's monitoring/alerting platform rather than local mail alone.

---

# 128. Rename `.txt` files with date

```bash
#!/usr/bin/env bash

today=$(date +%Y%m%d)

for file in *.txt; do
    [[ -e "$file" ]] || continue

    base="${file%.txt}"
    mv -- "$file" "${base}_${today}.txt"
done
```

---

# 129. Service health script

```bash
#!/usr/bin/env bash

SERVICE=nginx
LOG=/var/log/service-watch.log

if ! systemctl is-active --quiet "$SERVICE"; then
    echo "$(date '+%F %T') $SERVICE down; restarting" >> "$LOG"

    systemctl restart "$SERVICE"

    if systemctl is-active --quiet "$SERVICE"; then
        echo "$(date '+%F %T') restart succeeded" >> "$LOG"
    else
        echo "$(date '+%F %T') restart FAILED" >> "$LOG"
        exit 1
    fi
fi
```

---

# 130. Compress logs older than 30 days, delete after 90 days

```bash
#!/usr/bin/env bash
set -euo pipefail

LOGDIR=/var/log/myapp

find "$LOGDIR" \
  -type f \
  -name '*.log' \
  -mtime +30 \
  -mtime -90 \
  -exec gzip -- {} \;

find "$LOGDIR" \
  -type f \
  -name '*.gz' \
  -mtime +90 \
  -delete
```

Cron:

```cron
0 2 * * * /usr/local/bin/log-cleanup.sh >> /var/log/log-cleanup.log 2>&1
```

---

# 131. Track newly created file names

Linux with `inotifywait`:

```bash
#!/usr/bin/env bash

DIR=/data/incoming
OUTPUT=/var/log/new-files.log

inotifywait -m -e create --format '%w%f' "$DIR" |
while read -r file; do
    printf '%s %s\n' "$(date '+%F %T')" "$file" >> "$OUTPUT"
done
```

---

# 132. Python list vs tuple

List:

```python
x = [1, 2, 3]
x.append(4)
```

Mutable.

Tuple:

```python
x = (1, 2, 3)
```

Immutable.

Use tuple when fixed structure/immutability is desirable; list for a changing sequence.

---

# 133. Reverse string

```python
text = "DevOps"
print(text[::-1])
```

Output:

```text
spOveD
```

Slice 0 to 4:

```python
text[0:4]
```

includes indexes `0,1,2,3`; end index is excluded.

---

# 134. Python own module

`greetings.py`:

```python
def hello():
    return "Hello, World"
```

`main.py`:

```python
from greetings import hello

print(hello())
```

---

# 135. Python inheritance

```python
class Server:
    def start(self):
        print("Starting server")


class WebServer(Server):
    def deploy(self):
        print("Deploying website")


web = WebServer()
web.start()
web.deploy()
```

`WebServer` inherits behavior from `Server` and adds its own behavior.

---

# 136. Python decorators

A decorator wraps or modifies function/class behavior.

```python
def log_call(func):
    def wrapper(*args, **kwargs):
        print(f"Calling {func.__name__}")
        return func(*args, **kwargs)
    return wrapper


@log_call
def deploy():
    print("Deploying")


deploy()
```

Useful for:

- logging
- authentication
- timing
- retries
- caching.

---

# 137. Reverse words in a list

```python
words = ["aws", "docker", "kubernetes"]

result = [word[::-1] for word in words]

print(result)
```

---

# 138. Remove first duplicate

If the requirement means “remove the first occurrence that causes a duplicate”:

```python
items = [1, 2, 3, 2, 4, 3]

seen = set()
result = []
removed = False

for item in items:
    if item in seen and not removed:
        removed = True
        continue

    seen.add(item)
    result.append(item)

print(result)
```

Result:

```text
[1, 2, 3, 4, 3]
```

In an interview clarify what “first duplicate element” means.

---

# 139. MongoDB and high availability

MongoDB replica set:

```text
             Application
                 |
              Primary
             /       \
      Secondary     Secondary
```

Writes normally go to the primary. Secondaries replicate the primary's oplog. If the primary becomes unavailable, eligible voting members conduct an election and a secondary can become the new primary. [MongoDB](https://www.mongodb.com/docs/manual/core/replica-set-primary/?utm_source=chatgpt.com)

MongoDB recommends three data-bearing members as a common minimum replica-set configuration. [MongoDB](https://www.mongodb.com/docs/v8.0/core/replica-set-members/?utm_source=chatgpt.com)

---

# 140. Maven lifecycle

Common lifecycle:

```text
validate
compile
test
package
verify
install
deploy
```

Example:

```bash
mvn clean verify
```

CI usually runs tests/verification before artifact publication.

---

# 141. Artifact management

An artifact is a versioned immutable output of a build:

```text
Source
 |
Build
 |
Artifact
 |
Repository
 |
Dev -> QA -> Prod
```

Examples:

- JAR/WAR
- Docker/OCI image
- Helm chart
- ZIP/package.

Tools might include:

- Artifactory
- Nexus
- ECR
- GitHub Packages
- Azure Artifacts.

Important interview concept:

> Build once, promote the same immutable artifact across environments rather than rebuilding different binaries for dev and production.

---

# 142. Azure NSG vs Azure Firewall

Conceptually:

```text
NSG:
resource/subnet network filtering
L3/L4 rules

Azure Firewall:
centralized managed firewall
richer network/application inspection and policy
```

Use NSGs for distributed workload-level segmentation and Azure Firewall for centralized north-south/east-west policy where the architecture requires it.

---

# 143. Connect VMs in different VNets

Options include:

```text
VNet A <-- VNet peering --> VNet B
```

or hub-and-spoke architectures using VPN/Virtual WAN/firewall depending on complexity.

Ensure:

- non-overlapping address spaces
- correct peering/routing
- NSG allowance
- DNS resolution.

---

# 144. Azure Private Endpoint

Conceptually:

```text
VM/AKS
  |
private IP in VNet
  |
Private Endpoint
  |
Azure PaaS service
```

This lets supported PaaS services be reached over a private network path rather than requiring public exposure.

Private DNS configuration is a major part of a correct implementation.

---

# 145. ExpressRoute

ExpressRoute provides private connectivity between on-premises infrastructure and Microsoft's network through a connectivity provider.

Typical architecture:

```text
On-Prem
   |
Provider circuit
   |
ExpressRoute
   |
Azure VNet
```

Use when predictable private hybrid connectivity is required beyond ordinary internet VPN capabilities.

---

# 146. Managed identity vs service principal

A **service principal** is an application/workload identity in Microsoft Entra ID whose credential lifecycle you may need to manage depending on authentication method.

A **managed identity** is an Azure-managed identity abstraction designed to eliminate you having to manage credentials for supported Azure resources.

Microsoft describes a service principal as the local application identity within a tenant. [Microsoft Learn](https://learn.microsoft.com/en-us/azure/active-directory/fundamentals/service-accounts-principal?utm_source=chatgpt.com)

Interview framing:

```text
Azure VM/app needs Azure API access
          |
Supported managed identity?
          |
        yes
          |
   prefer managed identity
```

For external systems such as CI/CD, use workload federation where available instead of long-lived client secrets.

---

# 147. Only one AKS application should access Storage Account

Use workload identity rather than giving the node identity broad access.

```text
Pod
 |
Kubernetes ServiceAccount
 |
OIDC projected token
 |
Microsoft Entra Workload ID
 |
Azure managed identity/app identity
 |
RBAC scoped to Storage Account
```

Other Pods use different ServiceAccounts and have no corresponding federated identity/role.

Microsoft Entra Workload ID specifically enables AKS Pods to federate Kubernetes identities to Azure resource identities. [Microsoft Learn](https://learn.microsoft.com/en-us/azure/aks/workload-identity-deploy-cluster?utm_source=chatgpt.com)

Combine this with network/private-endpoint restrictions when required.

---

# 148. Build image and push to ACR

```bash
az acr login --name myregistry

docker build -t myapp:1.0 .

docker tag myapp:1.0 \
myregistry.azurecr.io/myapp:1.0

docker push \
myregistry.azurecr.io/myapp:1.0
```

Kubernetes deployment:

```yaml
containers:
- name: app
  image: myregistry.azurecr.io/myapp:1.0
```

Ensure AKS/Pod identity has registry pull permission rather than using broadly shared credentials where possible.

---

# 149. Azure Application Gateway SSL/TLS certificate

For HTTPS termination, configure a certificate on the Application Gateway listener.

Microsoft states that the listener certificate is used to terminate the client TLS connection. The certificate can be uploaded or referenced from Key Vault, depending on configuration. [Microsoft Learn](https://learn.microsoft.com/en-us/azure/application-gateway/ssl-certificate-management?utm_source=chatgpt.com)

```text
Client
 |
HTTPS
 |
App Gateway HTTPS listener + certificate
 |
HTTP or HTTPS backend
```

For end-to-end TLS:

```text
Client --TLS--> App Gateway --TLS--> Backend
```

---

# 150. Azure Build Artifacts vs Pipeline Artifacts

For Azure DevOps Services, Pipeline Artifacts are the newer optimized mechanism and Microsoft recommends Pipeline Artifacts over older build-artifact tasks for better performance. [Microsoft Learn](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/download-build-artifacts-v1?utm_source=chatgpt.com)

Conceptually both transfer build outputs, but Pipeline Artifacts are the default choice for modern Azure Pipelines unless compatibility constraints dictate otherwise.

---

# 151. Terraform dynamic block

Useful when a provider resource contains a repeatable nested block.

Example:

```hcl
variable "security_rules" {
  default = [
    {
      name = "https"
      port = 443
    },
    {
      name = "http"
      port = 80
    }
  ]
}

resource "azurerm_network_security_group" "example" {
  name                = "app-nsg"
  location            = var.location
  resource_group_name = var.resource_group_name

  dynamic "security_rule" {
    for_each = var.security_rules

    content {
      name                       = security_rule.value.name
      priority                   = 100 + security_rule.key
      direction                  = "Inbound"
      access                     = "Allow"
      protocol                   = "Tcp"
      source_port_range          = "*"
      destination_port_range     = tostring(security_rule.value.port)
      source_address_prefix      = "*"
      destination_address_prefix = "*"
    }
  }
}
```

Dynamic blocks reduce repetitive nested HCL; don't use them if ordinary explicit blocks are clearer.

---

# 152. Azure DevOps pipeline output variable

Typical pattern:

Producer:

```yaml
- bash: |
    echo "##vso[task.setvariable variable=imageTag;isOutput=true]$(Build.BuildId)"
  name: setvars
```

Downstream stage/job references the output through the appropriate `dependencies` / `stageDependencies` expression according to where it was set.

The key interview point is that ordinary variables do not automatically become cross-stage output variables; you must mark and reference them appropriately.

---

# 153. Pipeline deleted accidentally

Recovery:

1. check repository: pipeline YAML should be version controlled;
2. recreate pipeline pointing at the same YAML;
3. restore variables/service connections securely;
4. validate environment permissions/approvals;
5. run in non-production first.

Prevention:

```text
Pipeline-as-code
+
protected repo
+
RBAC
+
auditing
+
limited delete permission
+
backup/export for non-code configuration
```

This is why storing the pipeline definition only inside a UI is weaker than pipeline-as-code.

---

# 154. Reusable pipelines for 50 applications

Do not copy 50 full pipelines.

Use:

```text
central template
   |
parameters:
 language
 build command
 image name
 helm chart
 environment
   |
50 tiny application pipelines
```

Example:

```yaml
extends:
  template: templates/microservice.yml
  parameters:
    serviceName: payments
    dockerfile: Dockerfile
    helmPath: deploy/chart
```

Version templates and introduce breaking changes intentionally.

---

# 155. Private data/authentication between Azure DevOps and Key Vault

Preferred design:

```text
Azure DevOps pipeline
      |
service connection / workload identity federation
      |
Microsoft Entra
      |
temporary token
      |
Key Vault
```

Then authorize only required secret operations and control network access through Key Vault firewall/private endpoint if required.

Avoid storing a permanent client secret in pipeline variables when workload identity federation is available.

---

# 156. AKS 30 GiB used out of 32 GiB; Pod request 500 Mi, limit 4 Gi

Scheduling is primarily based on **requests**.

If a suitable node has at least the Pod's requested schedulable memory (plus other requirements), it *might* schedule even though the limit is 4 GiB.

But the question needs nuance:

```text
Node allocatable memory
-
sum(existing Pod requests)
>= 500Mi ?
```

That is the scheduling question—not:

```text
32Gi total - 30Gi current usage
```

If the container later approaches 4 GiB but the node has no physical memory available, memory pressure/OOM risks arise.

HPA does not create node memory. VPA might increase resource requests, potentially making the Pod unschedulable.

Karpenter/Cluster Autoscaler can provision node capacity when scheduling requires it.

---

# 157. Top infrastructure attacks and mitigation

A good top-five framing:

**Credential compromise:** MFA, federation, short-lived credentials, least privilege, secret rotation.

**Public service exploitation:** patching, hardened images, minimal exposure, WAF, vulnerability management.

**DDoS:** CDN/WAF/Shield/rate limits/autoscaling.

**Supply-chain compromise:** dependency scanning, signed artifacts, immutable action references, SBOM/provenance, trusted registries.

**Data exfiltration/ransomware:** segmentation, encryption, immutable backups, least privilege, anomaly detection and tested recovery.

Tie security controls to threat models rather than listing tools randomly.

---

# 158. CloudFormation vs Terraform

CloudFormation:

```text
AWS-native IaC service
AWS resource model
stack-oriented
```

Terraform:

```text
provider-based multi-platform IaC
state-managed
large ecosystem
HCL
```

Choose based on organization architecture, governance, multi-cloud requirements and skills.

Neither tool makes bad architecture safe automatically.

---

# 159. Declarative pipeline vs continuous delivery/deployment

Continuous **Delivery**:

```text
Build/test automatically
 |
production-ready artifact
 |
manual approval
 |
deploy
```

Continuous **Deployment**:

```text
Build/test
 |
quality gates
 |
automatically deploy production
```

A Jenkins `input` approval can turn an otherwise automated pipeline into a Continuous Delivery workflow rather than unconditional deployment.

---

# 160. Build and pipeline artifact integrity

Use:

- cryptographic checksums
- signed packages/images
- immutable tags/digests
- trusted repositories
- provenance/SBOM
- restricted publish access.

For container deployment prefer immutable digest references where high assurance matters:

```text
image@sha256:...
```

instead of relying only on mutable tags such as `latest`.

---

# 161. Kubernetes ImagePullBackOff

Check:

```bash
kubectl describe pod <pod>
```

Typical causes:

- wrong image name/tag
- private registry authentication
- missing `imagePullSecrets`
- registry/network/DNS failure
- registry rate limit
- architecture mismatch.

Then:

```bash
kubectl get secret
kubectl get serviceaccount
kubectl describe secret <pull-secret>
```

Test from an appropriate environment/node if needed.

`ImagePullBackOff` means Kubernetes attempted image pull, failed, and is backing off before another attempt.

---

# 162. OpenShift pull secret

A pull secret contains registry credentials used to authenticate image pulls, commonly Docker config JSON-style credentials.

Keep it restricted like any other credential and rotate/update it according to the registry/security model.

---

# 163. Password security practices

For humans:

- SSO/federation
- MFA
- strong password/passphrase policy
- password manager
- compromised-password detection
- no shared accounts.

For machines:

- avoid human passwords
- workload identities
- temporary credentials
- service roles
- automated rotation.

A mature answer is “remove passwords where stronger federation/workload identity exists,” not merely “make passwords 20 characters.”

---

# 164. Terraform provisioning through CI/CD

Typical production flow:

```text
PR
 |
terraform fmt/validate
 |
security/policy checks
 |
terraform plan
 |
plan reviewed
 |
merge/approval
 |
terraform apply using same reviewed commit
 |
remote state + locking
```

Authentication:

```text
CI workload identity/OIDC
        |
temporary cloud role
```

not static access keys.

Use separate state/workspaces/accounts according to your environment isolation model.

---

# 165. New AWS environment using Terraform

Example hierarchy:

```text
AWS Organization/account
        |
VPC
 + public subnets
 + private app subnets
 + private DB subnets
        |
NAT / endpoints
        |
EKS/EC2
        |
RDS/cache
        |
observability/security
```

Terraform repo:

```text
terraform/
  modules/
    vpc/
    eks/
    rds/
    iam/
  environments/
    dev/
    qa/
    prod/
```

Pipeline applies each environment with separate state and credentials.

---

# 166. “How many clusters/add-ons/data/etc. are you managing?”

These are **credibility questions**, not knowledge questions.

Use actual figures:

> “I currently help operate `<N>` production and `<N>` non-production EKS clusters. Our standard platform add-ons include `<actual components>` such as VPC CNI, CoreDNS, kube-proxy, EBS CSI, metrics stack, ingress/gateway, autoscaler, logging and security integrations.”

Then expect follow-ups on each named component.

Likewise for:

- RDS data volume
- number of accounts
- Pods
- nodes
- deployments
- pipeline count.

Never inflate them.

---

# 167. “Rate your AWS skill out of 5”

A credible senior answer:

> “I would rate myself around 4/5 for the services I operate regularly—networking, IAM, EC2, EKS, S3, load balancing, monitoring and Terraform. I wouldn't call myself 5/5 because AWS is too broad and I still verify service-specific edge cases in the documentation.”

Adjust the domain list to what you genuinely know.

---

# 168. Self-introduction for 6 YOE

A good two-minute structure:

```text
Current role
   |
years + core specialization
   |
current platform scale
   |
primary tools
   |
2-3 achievements
   |
what role you want next
```

Sample:

> “I have around six years of experience in cloud and DevOps engineering, with my recent work focused on AWS infrastructure, Kubernetes/EKS, Terraform and CI/CD. In my current role I’m responsible for provisioning and operating cloud environments, maintaining Kubernetes workloads, automating infrastructure through Terraform, supporting release pipelines and production troubleshooting. I’ve worked on areas such as cluster upgrades, autoscaling, observability, security hardening and cost optimization. I’m now looking for a role where I can take more ownership of platform reliability, automation and architecture rather than only operational support.”

Replace technologies that are not genuinely yours.

---

# 169. “What motivates you for this role?”

A useful answer should connect your next step to the job:

> “The role matches the areas where I want deeper ownership: cloud architecture, Kubernetes/platform engineering, automation and reliability. I enjoy eliminating manual operations, designing systems that remain stable during failures and improving the developer deployment experience. At this stage I’m looking for a role where I can combine hands-on troubleshooting with architecture and engineering ownership.”

---

# 170. Toughest situation / failed production deployment

Use STAR, but include technical depth:

```text
Situation:
Production release caused elevated 5xx.

Task:
Restore service safely.

Action:
- froze deployments
- reviewed dashboards/logs
- correlated errors with release
- rolled back
- validated DB compatibility
- kept stakeholders updated

Result:
Service restored.

Prevention:
- automated smoke tests
- deployment metrics gate
- backward-compatible DB migration
- documented rollback
```

Do not make the story end at “we rolled back.” Explain the prevention.

---

# 171. Communicating with non-technical stakeholders

Use:

```text
Impact
What we know
What we don't yet know
What we're doing
Risk
Next update
```

Example:

> “Checkout is unavailable for approximately 30% of users. We isolated the issue to the application tier and have stopped the faulty release. The team is restoring the previous version now. Customer data integrity has not shown an issue. I’ll provide the next update after service validation.”

Avoid dumping Kubernetes stack traces into executive updates.

---

# 172. Conflict inside DevOps/cross-functional team

Structure:

```text
Shared outcome first
Evidence second
Trade-offs explicitly
Small experiment if uncertain
Decision documented
Follow-up
```

For example, if developers want unrestricted production access and SRE wants zero access, frame the actual objective:

```text
fast incident response
+
auditability
+
least privilege
```

Then implement temporary elevated/JIT access rather than arguing in absolutes.

---

# 173. Reducing deployment time

Strong examples include:

- parallel tests
- dependency caching
- smaller Docker contexts
- multi-stage builds
- reusable CI workers
- incremental tests
- eliminating manual environment steps
- GitOps/automated deployment
- immutable artifact promotion.

Measure:

```text
Before: X minutes
After: Y minutes
Change failure rate?
Reliability impact?
```

Do not optimize pipeline duration by removing meaningful quality gates.

---

# 174. CI/CD pipeline vulnerability incident

Example response:

```text
Detection
 |
stop affected pipeline/revoke credential
 |
identify impacted repositories/runs
 |
rotate secrets
 |
preserve logs/evidence
 |
patch/remove malicious dependency/action
 |
rebuild clean artifacts
 |
verify downstream systems
 |
strengthen controls
```

For GitHub Actions specifically, pin third-party actions to immutable commit SHAs and prefer OIDC to persistent cloud credentials. GitHub explicitly recommends full-length SHA pinning. [GitHub Docs](https://docs.github.com/en/actions/reference/security/secure-use?learn=getting_started\&learnProduct=actions\&utm_source=chatgpt.com)

---

# 175. High-pressure collaboration

Operationally:

```text
Incident Commander
Technical leads
Scribe
Stakeholder communicator
```

Keep troubleshooting channels focused; move speculation away from the main incident timeline.

One owner per action prevents:

```text
"Someone is checking the DB."
```

from becoming nobody checking the DB.

---

# 176. S3 made public accidentally

Immediate response:

```text
1. Enable/verify S3 Block Public Access
2. Remove public bucket policy/ACL
3. Preserve CloudTrail/access evidence
4. Determine exposure interval
5. Determine accessed data
6. Rotate exposed secrets if any
7. Security incident workflow
8. Add preventive controls
```

AWS recommends S3 Block Public Access and avoiding wildcard public principals except when explicitly intended. [AWS Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html?utm_source=chatgpt.com)

Prevent recurrence with:

- Organizations/SCP controls
- AWS Config/Security Hub findings
- IaC review/policy-as-code
- least privilege.

---

# 177. EC2 compromised

Do not immediately destroy evidence unless containment requirements dictate it.

Better incident sequence:

```text
Contain
 |
quarantine SG / isolate network
 |
preserve evidence/snapshots
 |
revoke/rotate credentials
 |
investigate lateral movement
 |
rebuild from trusted image
 |
restore validated data
 |
root cause + controls
```

Terminating instantly can destroy forensic evidence.

---

# 178. Runners and private environments

Self-hosted runners should be segmented.

```text
GitHub
   |
outbound HTTPS
   |
Runner subnet
   |
restricted network route
   |
deployment environment
```

GitHub warns that self-hosted runners can execute repository-controlled code and recommends careful treatment, particularly around untrusted public PRs. [GitHub Docs](https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/add-runners?utm_source=chatgpt.com)

Use ephemeral runner strategies where feasible.

---

# 179. Azure monitoring alert — CPU/memory

Architecture:

```text
VM
 |
Azure Monitor metrics / agent
 |
Alert rule
 |
Condition
 |
Action Group
 |
Email/Teams/webhook/automation
```

For CPU, use platform metrics.

For guest memory metrics/log queries, ensure the required monitoring agent/data collection is configured.

Set:

- metric/query
- aggregation
- threshold
- evaluation window
- severity
- action group
- suppression where required.

Then test alert delivery rather than assuming configuration means it works.

---

# 180. “What if firewall between Kubernetes control plane and workers breaks?”

Existing containers may continue running, but important control-plane/node communication is lost.

Possible effects:

- Node status becomes Unknown/NotReady
- kubelet cannot report status
- Secrets/Config updates may not propagate
- new workloads cannot be properly scheduled/started on affected nodes
- exec/log/port-forward operations fail
- Service/networking behaviors can degrade depending on components.

First:

```text
Declare incident
Identify impacted clusters/nodes
Freeze risky deployments
Restore connectivity
Verify node state
Reconcile workloads
Communicate customer impact
```

Do not answer “everything immediately crashes.”

---

# 181. Four or five containers in one Pod — project use case

A defensible example:

```text
Pod
 |
 + main application
 + Envoy/Istio sidecar
 + log collector
 + config reloader
 + local authentication/helper proxy
```

But immediately add:

> “I wouldn't put unrelated microservices into one Pod. Containers in a Pod share lifecycle/network context and should form one logical application unit.”

That distinction demonstrates Kubernetes design maturity.

---

# 182. `restartPolicy: Never`

If a container terminates in a Pod configured:

```yaml
restartPolicy: Never
```

the kubelet does not restart that terminated container under the normal Pod restart-policy behavior.

This is common for one-shot Jobs/tasks rather than long-running application Deployments.

---

# 183. DaemonSet vs Deployment/ReplicaSet

DaemonSet:

```text
one matching Pod per eligible node
```

Common:

- node monitoring
- log collection
- CNI agents.

Deployment:

```text
desired number of replicas independent of node count
```

ReplicaSet maintains replica count; Deployment manages ReplicaSets and rollout history/strategy.

---

# 184. Kubernetes namespace

Namespaces provide logical scoping for namespaced resources and are commonly used for:

- environment/team separation
- RBAC
- quotas
- policies.

Example:

```bash
kubectl create namespace payments
kubectl get pods -n payments
```

A namespace is **not by itself a strong security boundary**. Combine it with RBAC, NetworkPolicy, quota and admission policies.

---

# 185. Kubernetes ConfigMap

Use ConfigMap for non-secret configuration:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: info
  API_URL: https://api.internal
```

Consume through:

```yaml
envFrom:
- configMapRef:
    name: app-config
```

Use Secret—not ConfigMap—for credentials.

---

# 186. EBS-backed vs instance-store-backed

EBS-backed storage:

- persistent EBS volume
- can persist independently according to volume/delete settings
- supports snapshots
- flexible attachment/storage features.

Instance store:

- physically attached temporary block storage associated with the host
- data is ephemeral across certain instance lifecycle events.

Do not store irreplaceable persistent application data solely on instance store.

---

# 187. Why Kubernetes instead of Docker Swarm?

A fair answer:

Kubernetes offers a much larger orchestration ecosystem and richer capabilities around:

- extensible APIs/controllers
- scheduling
- autoscaling
- network/storage integration
- RBAC/policy
- managed offerings
- operators
- ecosystem/tooling.

Docker Swarm can be simpler, but Kubernetes is generally a better match when your organization needs the broader ecosystem and platform capabilities.

Avoid claiming Swarm is “bad”; architecture is contextual.

---

# 188. Prometheus/Grafana install answer

For Kubernetes, I would normally use a maintained Helm stack rather than manually constructing every component:

```text
Helm release
 |
 + Prometheus
 + Alertmanager
 + Grafana
 + kube-state-metrics
 + node exporter
```

Then:

1. configure persistent storage if required;
2. scrape ServiceMonitors/PodMonitors;
3. configure retention;
4. add alert rules;
5. configure Grafana data source;
6. import/create dashboards;
7. configure notification receivers;
8. test alerts.

Grafana officially supports Prometheus as a native data source. [Grafana Labs](https://grafana.com/docs/grafana/latest/datasources/?utm_source=chatgpt.com)

---

# 189. Azure/Application Gateway “Nginx is deprecated” interview correction

Phrase it carefully:

> “The Kubernetes-community **ingress-nginx** controller has been retired as of March 2026. The Kubernetes Ingress API itself still exists but is frozen; Kubernetes recommends Gateway API for new evolution. F5's NGINX Ingress Controller is a separate project and remains maintained.”

This distinction matters in 2026 interviews. [Kubernetes](https://kubernetes.io/blog/2026/02/27/ingress-nginx-before-you-migrate/?utm_source=chatgpt.com)

---

# 190. Interview questions about organization architecture

For:

- “What is your current CI/CD?”
- “How many clusters?”
- “Which Jenkinsfile?”
- “What add-ons?”
- “What security tools?”
- “How are you using CyberArk?”
- “How many containers?”

Use this framework:

```text
Business/application context
        |
Source control
        |
CI
        |
security gates
        |
artifact repository
        |
CD
        |
runtime platform
        |
observability
        |
security/secrets
```

Example:

> “Developers raise PRs in GitHub. PR pipelines run unit tests, static analysis and container/IaC scans. After merge we build one immutable image, push it to ECR and deploy to dev using Helm. Integration tests run there. Production requires environment approval, after which the same image digest is promoted. AWS authentication uses short-lived workload credentials, secrets come from our secret-management system, and Kubernetes/Prometheus/Grafana plus centralized logs handle runtime visibility.”

Then substitute **your actual toolchain**.

---

## Five senior-level corrections worth memorizing

Several questions in your source are deliberately worded to see whether you challenge the premise.

1. **Karpenter does not simply scale because CPU reaches a threshold.** It provisions nodes for unschedulable Pods and their constraints; consolidation manages excess/inefficient capacity. [Karpenter](https://karpenter.sh/docs/?utm_source=chatgpt.com)

2. **A new StatefulSet replica's PVC can indeed begin empty.** The application must bootstrap/replicate state when horizontal expansion requires it. Kubernetes gives each ordinal its own PVC. [Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/?utm_source=chatgpt.com)

3. **Security Groups do not have explicit deny rules.** Missing allow rules blocks traffic; use NACL/firewall controls for explicit deny patterns where appropriate.

4. **A Pod can schedule based on a 500 Mi request even with a 4 Gi limit** if request-based scheduling constraints are satisfied; that does not mean 4 Gi of actual runtime memory is guaranteed to exist.

5. **Ingress NGINX and F5 NGINX Ingress Controller are different projects.** The former was retired in March 2026; the latter is separately maintained. [Kubernetes](https://kubernetes.io/blog/2026/02/27/ingress-nginx-before-you-migrate/?utm_source=chatgpt.com)

This consolidated bank covers the substantive topics and repeated variants throughout the supplied interview list, including its Kubernetes/EKS, AWS, Terraform, CI/CD, Docker, Linux/SRE, security, monitoring, Azure and behavioral sections. Pasted text
