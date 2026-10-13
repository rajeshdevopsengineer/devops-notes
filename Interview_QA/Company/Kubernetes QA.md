# Kubernetes Interview Questions and Answers (1–25)

These are interview-ready answers for DevOps, Cloud Engineer, SRE, and Kubernetes Administrator roles. Each answer includes the main concept, practical examples, and important points interviewers expect.

## Section 1: Kubernetes Basics and Architecture

### 1. What is Kubernetes, and how is it different from Docker?

Answer:

Kubernetes (K8s) is an open-source container orchestration platform used to deploy, manage, scale, and monitor containerized applications automatically.

Docker is a container technology and tooling ecosystem commonly used to build and run container images.

| Docker                                             | Kubernetes                                      |
| -------------------------------------------------- | ----------------------------------------------- |
| Builds and runs containers                         | Orchestrates containers across nodes            |
| Primarily handles container-level operations       | Manages applications across clusters            |
| Provides image-building tools                      | Uses container images                           |
| No built-in Kubernetes-style cluster orchestration | Provides scheduling and self-healing            |
| Can run containers using Docker Engine             | Uses CRI-compatible runtimes such as containerd |

Real-time example: You build a Java application image using Docker. Kubernetes deploys it across multiple nodes, maintains the required replicas, and replaces failed Pods automatically.

Interview tip: Kubernetes does not require Docker Engine. Modern Kubernetes commonly uses containerd or CRI-O.

### 2. Explain Kubernetes architecture and its components.

Answer:

Kubernetes follows a control plane and worker node architecture.

Kubernetes Cluster Architecture

Control Plane

kube-apiserver

etcd

Scheduler

Controller Manager

Kubernetes API communication

Worker Node 1

kubelet

kube-proxy\*

Container Runtime

Pod A

Pod B

Worker Node 2

kubelet

kube-proxy\*

Container Runtime

Pod A

Pod B

\*kube-proxy is optional when another implementation provides Service networking.

Control plane components:

- API Server — entry point for Kubernetes API requests.
- etcd — stores cluster configuration and state.
- Scheduler — selects nodes for unscheduled Pods.
- Controller Manager — runs controllers that reconcile desired and actual states.
- Cloud Controller Manager — integrates with cloud infrastructure when applicable.

Worker node components:

- kubelet — ensures Pod containers are running.
- kube-proxy — commonly implements Service networking.
- Container Runtime — runs containers.
- Pods — execute application workloads.

### 3. What are the control plane components in Kubernetes?

Answer:

The main Kubernetes control plane components are:

| Component                | Responsibility                                     |
| ------------------------ | -------------------------------------------------- |
| kube-apiserver           | Handles API requests and cluster access            |
| etcd                     | Stores Kubernetes cluster state                    |
| kube-scheduler           | Assigns Pods to nodes                              |
| kube-controller-manager  | Runs reconciliation controllers                    |
| cloud-controller-manager | Integrates with cloud-provider services, when used |

Interview point: The API server is the central interface through which Kubernetes components read and update cluster state.

### 4. What is the role of the Kubernetes Controller Manager?

Answer:

The Kubernetes Controller Manager runs multiple controllers to maintain the desired state of the cluster.

For example, if a Deployment requires five replicas but only three are running, the relevant controllers work to create two additional Pods.

Common controllers include:

- Node Controller — monitors node health.
- ReplicaSet Controller — maintains the desired number of Pod replicas.
- Deployment Controller — manages Deployment rollouts.
- Job Controller — manages task completion.
- ServiceAccount Controller — manages ServiceAccount resources.

Interview point: Controllers continuously compare actual state with desired state and take corrective action. This is called the reconciliation loop.

### 5. What is the purpose of the Kubernetes Scheduler?

Answer:

The kube-scheduler selects the most suitable node for newly created Pods that have not yet been assigned a node.

It considers factors such as CPU and memory requests, node affinity, taints and tolerations, topology constraints, and other scheduling policies.

Example:

Suppose a Pod requests 2 CPUs and 4 GiB memory.

- Node A has insufficient allocatable CPU remaining.
- Node B has sufficient CPU and memory.
- Node C has sufficient resources but an untolerated taint.

The scheduler can select Node B.

Interview point: The scheduler assigns Pods to nodes, but kubelet actually starts the containers.

### 6. What is the difference between a master node and a worker node?

Answer:

A master node is traditionally called a control plane node in modern Kubernetes terminology.

| Control Plane Node                                  | Worker Node                         |
| --------------------------------------------------- | ----------------------------------- |
| Manages cluster operations                          | Runs application workloads          |
| Hosts API server and other control plane components | Hosts kubelet and container runtime |
| Maintains or coordinates cluster state              | Executes Pods                       |
| Makes scheduling and management decisions           | Follows assigned Pod specifications |
| Often protected by scheduling taints                | Commonly accepts application Pods   |

Interview point: A control plane node can also run application Pods when configured to do so.

### 7. How do master and worker nodes communicate?

Answer:

Control plane and worker nodes primarily communicate through the Kubernetes API server.

Typical process:

1. A user submits a Deployment using `kubectl`.
2. The API server accepts the request and stores the relevant state in etcd.
3. The scheduler assigns Pods to worker nodes.
4. The kubelet on each selected worker watches the API server for its assigned Pods.
5. kubelet instructs the container runtime to start containers.
6. kubelet reports Pod and node status through the API server.

Communication is secured using TLS and Kubernetes authentication and authorization mechanisms.

Important: Worker nodes do not normally access etcd directly. They communicate through the API server.

### 8. What is the purpose of kubelet and kube-proxy?

Answer:

| kubelet                         | kube-proxy                               |
| ------------------------------- | ---------------------------------------- |
| Node agent                      | Network service proxy                    |
| Manages Pod container lifecycle | Implements Service traffic forwarding    |
| Works with container runtime    | Configures network forwarding rules      |
| Reports Pod and node status     | Helps route Service traffic to endpoints |
| Runs on Kubernetes nodes        | Commonly runs on Kubernetes nodes        |

Example:

- kubelet starts an Nginx application container.
- kube-proxy configures forwarding so traffic addressed to an Nginx Service reaches a suitable backend Pod.

Interview point: kube-proxy does not normally forward every packet through its own process; it programs kernel networking rules. Some clusters replace it with eBPF-based networking implementations.

### 9. What is etcd, and can we manually write data into it?

Answer:

etcd is a distributed, strongly consistent key-value store used by Kubernetes to store cluster state, configuration, and resource metadata.

It stores information about Pods, Deployments, Services, ConfigMaps, Secrets, and other API resources.

Can we manually write data into etcd?

Technically yes, if authorized and given direct etcd access, using tools such as `etcdctl`.

However, direct modification of Kubernetes-managed data in etcd is strongly discouraged because it bypasses API validation, admission controls, and normal cluster management.

The recommended approach is to use Kubernetes API operations through `kubectl` or API clients.

Example:

```
kubectl create configmap app-config \
  --from-literal=ENV=production
```

This updates cluster state through the Kubernetes API.

Interview point: etcd backups are essential for recovering the state of self-managed Kubernetes control planes.

## Section 2: Pods and Scheduling

### 10. What is a Pod in Kubernetes?

Answer:

A Pod is the smallest deployable unit in Kubernetes. It contains one or more containers that share network resources and can share storage volumes.

Key characteristics:

- Containers in a Pod share the same network namespace and IP address.
- Containers communicate using `localhost`.
- Pods can contain multiple closely related containers.
- Pods are generally ephemeral and can be replaced when they fail.

Example:

A Pod might contain an Nginx application container and a log-processing sidecar container.

```
kubectl get pods
kubectl describe pod nginx-pod
kubectl logs nginx-pod
```

Interview point: Kubernetes schedules Pods, not individual containers.

### 11. What is the difference between a Pod and a container?

| Pod                                          | Container                                  |
| -------------------------------------------- | ------------------------------------------ |
| Kubernetes deployment unit                   | Isolated application execution environment |
| Contains one or more containers              | Runs an application process                |
| Has a Pod-level IP address                   | Usually shares the Pod network namespace   |
| Scheduled onto a node                        | Started by the container runtime           |
| Can provide shared storage to its containers | Uses mounted volumes for shared data       |

Example:

A Pod can have an application container and a sidecar container that collects application logs.

Both containers run inside the same Pod and can share storage and networking.

### 12. How many containers can run inside a single Pod?

Answer:

A Pod can contain one or more containers. Kubernetes does not define a small fixed maximum such as 2, 5, or 10 containers per Pod.

The practical limit depends on available CPU, memory, runtime capabilities, and cluster configuration.

Example:

```
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  containers:
    - name: nginx
      image: nginx:stable
    - name: busybox
      image: busybox:stable
      command: ["sh", "-c", "sleep 3600"]
```

This Pod contains two application containers.

Interview point: Prefer keeping containers together only when they are tightly coupled and need to share resources or lifecycle.

### 13. Can one container run inside two different Pods?

Answer:

No. A running container instance belongs to one Pod at a time.

However, you can use the same container image to create separate container instances in multiple Pods.

Example:

- Pod A runs an Nginx container created from `nginx:stable`.
- Pod B runs another Nginx container created from `nginx:stable`.

The image is the same, but the running container instances are different.

Interview point: A container image is reusable; an individual running container cannot be shared across Pods.

### 14. Can we schedule Pods on a master node?

Answer:

Yes. Pods can run on control plane nodes when scheduling rules permit it.

In kubeadm-based clusters, control plane nodes are normally tainted to prevent ordinary application workloads from being scheduled.

Check node taints:

```
kubectl describe node <node-name>
```

To remove the standard control plane scheduling taint:

```
kubectl taint nodes <node-name> \
  node-role.kubernetes.io/control-plane:NoSchedule-
```

Alternatively, add an appropriate toleration to a Pod rather than removing the taint.

Interview point: Running ordinary applications on control plane nodes is possible, but separating workloads is generally preferred in production.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://kubernetes.io\&sz=32)

Kubernetes

+1



### 15. Why are application Pods not scheduled on master nodes by default?

Answer:

In common kubeadm configurations, control plane nodes have a `NoSchedule` taint to protect critical cluster components.

Reasons include:

- Reserving CPU and memory for the API server, scheduler, etcd, and controllers.
- Preventing application resource spikes from affecting control plane availability.
- Improving workload isolation and cluster reliability.
- Reducing unnecessary security exposure.

Typical taint:

```
node-role.kubernetes.io/control-plane:NoSchedule
```

Interview point: This is a default of certain cluster provisioning methods, not a universal rule enforced by every Kubernetes cluster.

## Section 3: Kubernetes Workload Controllers

### 16. What is the difference between a Deployment and a ReplicaSet?

Answer:

A ReplicaSet ensures that a specified number of matching Pods are running.

A Deployment is a higher-level controller that manages ReplicaSets and supports declarative application updates.

| Deployment                              | ReplicaSet                                               |
| --------------------------------------- | -------------------------------------------------------- |
| Manages ReplicaSets                     | Manages Pods                                             |
| Supports rolling updates                | Does not independently provide Deployment-style rollouts |
| Supports rollback to previous revisions | No built-in revision rollback                            |
| Suitable for application deployment     | Primarily maintains replica count                        |
| Commonly created directly               | Usually created through a Deployment                     |

Example:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
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
          image: nginx:stable
```

Apply:

```
kubectl apply -f deployment.yaml
```

Check:

```
kubectl get deployments
kubectl get replicasets
kubectl get pods
```

Interview point: A Deployment creates and manages ReplicaSets, and ReplicaSets maintain the required Pods.

### 17. What is the difference between a Deployment and a StatefulSet?

Answer:

Deployments are generally used for stateless applications, while StatefulSets are designed for applications requiring stable identities and persistent storage.

| Deployment                                        | StatefulSet                                         |
| ------------------------------------------------- | --------------------------------------------------- |
| Typically used for stateless applications         | Typically used for stateful applications            |
| Pods are interchangeable                          | Pods have stable identities                         |
| Pod names can change on replacement               | Predictable ordinal Pod names                       |
| No default per-replica persistent volume identity | Supports per-replica volume claim templates         |
| Flexible rollout behavior                         | Supports ordered creation and termination           |
| Common for APIs and web apps                      | Common for databases and clustered stateful systems |

Example:

Deployment Pod names:

```
nginx-7b9c6d8f5-xa12b
nginx-7b9c6d8f5-xy34c
```

StatefulSet Pod names:

```
mysql-0
mysql-1
mysql-2
```

StatefulSets preserve stable Pod identities across replacements. With suitable persistent volume configuration, replacement Pods can reattach their corresponding storage.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://kubernetes.io\&sz=32)

Kubernetes



Interview point: A StatefulSet helps manage stateful applications but does not automatically implement database replication, backups, or failover.

### 18. What is the difference between a ReplicaSet and a DaemonSet?

Answer:

| ReplicaSet                           | DaemonSet                                |
| ------------------------------------ | ---------------------------------------- |
| Maintains a desired number of Pods   | Maintains a Pod on every eligible node   |
| Replica count is specified           | Pod count depends on eligible nodes      |
| Pods may run on various nodes        | Pods are targeted to particular nodes    |
| Useful for application replicas      | Useful for node-level agents             |
| Commonly managed through Deployments | Commonly deployed directly or by add-ons |

Example:

Suppose a Kubernetes cluster has five worker nodes.

- ReplicaSet with `replicas: 3` maintains three Pods in the cluster.
- DaemonSet targeting all five worker nodes maintains five Pods, one per node.

Interview point: ReplicaSet controls replica count; DaemonSet controls node coverage.

### 19. What is the difference between a DaemonSet and a StatefulSet?

Answer:

| DaemonSet                              | StatefulSet                                  |
| -------------------------------------- | -------------------------------------------- |
| Runs Pods on eligible nodes            | Runs a desired number of stateful Pods       |
| Typically one Pod per eligible node    | Multiple Pods can run on the same node       |
| Focuses on node-level services         | Focuses on stable workload identity          |
| Node coverage determines Pod placement | Scheduler places Pods subject to constraints |
| Used for logging and monitoring agents | Used for databases and stateful services     |

Real-world example:

DaemonSet: Running Fluent Bit on every worker node to collect logs.

StatefulSet: Running a three-member database cluster with stable identities and dedicated persistent storage.

### 20. What are the different Kubernetes workload resources?

Answer:

The main built-in Kubernetes workload resources are:

| Resource              | Purpose                                  | Example                     |
| --------------------- | ---------------------------------------- | --------------------------- |
| Pod                   | Runs one or more containers              | Single application instance |
| Deployment            | Manages application replicas and updates | Nginx web application       |
| ReplicaSet            | Maintains a desired number of Pods       | Three frontend replicas     |
| StatefulSet           | Manages stateful application instances   | PostgreSQL, ZooKeeper       |
| DaemonSet             | Runs Pods across eligible nodes          | Log collection agent        |
| Job                   | Runs tasks to completion                 | Database migration          |
| CronJob               | Executes Jobs on a schedule              | Nightly database backup     |
| ReplicationController | Legacy replication controller            | Older Kubernetes workloads  |

Interview point: Deployments, StatefulSets, DaemonSets, Jobs, and CronJobs are the main higher-level workload APIs to understand.

### 21. What is the purpose of a DaemonSet?

Answer:

A DaemonSet ensures that all or selected eligible nodes run a copy of a particular Pod.

When an eligible node joins the cluster, the DaemonSet automatically arranges for a Pod to run there.

Common uses:

- Log collection: Fluent Bit, Fluentd.
- Monitoring: Prometheus Node Exporter.
- Networking: CNI agents such as Cilium.
- Storage: node-level storage agents.

Check DaemonSets:

```
kubectl get daemonsets -A
```

Example DaemonSet:

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-logger
spec:
  selector:
    matchLabels:
      app: node-logger
  template:
    metadata:
      labels:
        app: node-logger
    spec:
      containers:
        - name: logger
          image: busybox:stable
          command:
            - sh
            - -c
            - while true; do echo "Node logging"; sleep 60; done
```

Interview point: Unlike a Deployment, a DaemonSet automatically adjusts its Pod count when the number of eligible nodes changes.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://kubernetes.io\&sz=32)

Kubernetes



### 22. Which default DaemonSets are present in Kubernetes?

Answer:

There is no single list of DaemonSets guaranteed to exist in every Kubernetes cluster.

The installed DaemonSets depend on the cluster distribution, CNI plugin, cloud provider, and installed add-ons.

Common examples are:

| DaemonSet       | Purpose                                           |
| --------------- | ------------------------------------------------- |
| kube-proxy      | Kubernetes Service networking, where used         |
| aws-node        | AWS VPC CNI networking on applicable EKS clusters |
| calico-node     | Calico node networking                            |
| cilium          | Cilium networking                                 |
| kube-flannel-ds | Flannel networking                                |
| node-exporter   | Node metrics, if installed                        |
| fluent-bit      | Log collection, if installed                      |

Check DaemonSets in the system namespace:

```
kubectl get daemonsets -n kube-system
```

Check all namespaces:

```
kubectl get ds -A
```

Interview point: In many kubeadm clusters, kube-proxy is deployed as a DaemonSet. However, some clusters do not use kube-proxy, and add-on DaemonSets are installation-dependent.

## Section 4: Init Containers and Kubernetes Operators

### 23. What is the purpose of an init container?

Answer:

An init container runs initialization tasks before normal application containers start.

Regular init containers run sequentially and must complete successfully before the main application containers can start.

Common uses:

- Waiting for a database or dependent service.
- Downloading configuration files.
- Preparing shared directories.
- Performing required startup checks.
- Running setup scripts.

Example:

```
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  initContainers:
    - name: init-setup
      image: busybox:stable
      command:
        - sh
        - -c
        - echo "Initialization completed"
  containers:
    - name: nginx
      image: nginx:stable
```

Execution sequence:

Init Container 1

Execute setup tasks

Init Container Completes

Exit code 0

Application Container Starts

Nginx starts running

Interview point: A regular init container terminates after successful completion, unlike a sidecar container that continues running alongside the application.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://kubernetes.io\&sz=32)

Kubernetes



### 24. How do you execute a shell script before an application container starts?

Answer:

The recommended Kubernetes approach is to use an init container.

Example:

```
apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
spec:
  volumes:
    - name: scripts
      configMap:
        name: startup-scripts

  initContainers:
    - name: run-script
      image: busybox:stable
      command:
        - /bin/sh
        - /scripts/init.sh
      volumeMounts:
        - name: scripts
          mountPath: /scripts

  containers:
    - name: app
      image: nginx:stable
```

Create the initialization script:

```
cat > init.sh <<'EOF'
#!/bin/sh
echo "Running initialization tasks"
echo "Initialization completed"
EOF
```

Create the ConfigMap:

```
kubectl create configmap startup-scripts \
  --from-file=init.sh
```

Apply the Pod manifest:

```
kubectl apply -f pod.yaml
```

Check logs of the init container:

```
kubectl logs app-with-init -c run-script
```

How it works: Kubernetes mounts the shell script through the ConfigMap, executes it through `/bin/sh` in the init container, and only starts the application container after successful completion.

Interview point: A `postStart` lifecycle hook is not a substitute for an init container when strict completion-before-application-start ordering is required.

### 25. What is a Kubernetes Operator?

Answer:

A Kubernetes Operator is a software extension that automates the deployment, configuration, management, and lifecycle of an application using Kubernetes APIs.

Operators typically combine:

- Custom Resource Definition (CRD): Introduces a new resource type to Kubernetes.
- Custom Resource (CR): Defines an instance and its desired configuration.
- Custom Controller: Watches resources and reconciles their actual state with the desired state.

Real-world example:

A PostgreSQL Operator might automatically:

1. Create a PostgreSQL database cluster.
2. Configure persistent storage.
3. Manage replication and backups.
4. Handle upgrades.
5. Detect failures and automate supported recovery operations.

A simplified custom resource might look like:

```
apiVersion: database.example.com/v1
kind: PostgreSQLCluster
metadata:
  name: production-db
spec:
  replicas: 3
  storage: 20Gi
```

This is an illustrative API; actual fields depend on the installed Operator and CRD.

Interview point: An Operator extends Kubernetes with application-specific operational knowledge, implementing tasks that standard workload controllers do not automatically handle.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://kubernetes.io\&sz=32)

Kubernetes

+1



## Quick Revision: Most Important Kubernetes Differences

| Comparison                | Key difference to remember                                |
| ------------------------- | --------------------------------------------------------- |
| Docker vs Kubernetes      | Container tooling vs orchestration                        |
| Control Plane vs Worker   | Manages cluster vs runs workloads                         |
| kubelet vs kube-proxy     | Container lifecycle vs Service networking                 |
| Pod vs Container          | Kubernetes scheduling unit vs application execution unit  |
| Deployment vs ReplicaSet  | Rollout management vs replica count                       |
| Deployment vs StatefulSet | Interchangeable Pods vs stable Pod identities             |
| ReplicaSet vs DaemonSet   | Fixed desired replicas vs node coverage                   |
| DaemonSet vs StatefulSet  | Node-level agents vs stateful workloads                   |
| Init Container vs Sidecar | Completes before app startup vs continues running         |
| Controller vs Operator    | Generic reconciliation vs application-specific automation |

## Practical Commands to Prepare for Interviews

```
# Cluster information
kubectl cluster-info

# List nodes
kubectl get nodes -o wide

# List all Pods
kubectl get pods -A

# View Deployments
kubectl get deployments -A

# View ReplicaSets
kubectl get rs -A

# View StatefulSets
kubectl get statefulsets -A

# View DaemonSets
kubectl get ds -A

# View node details and taints
kubectl describe node <node-name>

# Check a Pod's events and state
kubectl describe pod <pod-name>

# View application logs
kubectl logs <pod-name>

# Check init container logs
kubectl logs <pod-name> -c <init-container-name>

# Scale a Deployment
kubectl scale deployment nginx --replicas=5

# Check rollout history
kubectl rollout history deployment/nginx

# Roll back a Deployment
kubectl rollout undo deployment/nginx

# List CRDs used by Operators
kubectl get crds
```

## Test Your Interview Knowledge

# 4/5

Good progress!

You've completed the Kubernetes fundamentals quiz.

&#x20;Try again

Preparation advice: For a Kubernetes or DevOps interview, be especially confident explaining the architecture, Pod scheduling flow, Deployment versus StatefulSet, DaemonSets, init containers, and how controllers reconcile desired state. These concepts are foundational to troubleshooting and production Kubernetes operations.
