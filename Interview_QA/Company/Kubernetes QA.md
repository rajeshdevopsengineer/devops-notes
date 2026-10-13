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


# Kubernetes Interview Questions and Answers (26–50)

Topics: Custom Resources, Labels, Annotations, Namespaces, Services, DNS, CNI, EKS Networking, CSI, Persistent Volumes, EBS, EFS, and StatefulSets.

These answers are designed for DevOps, AWS EKS, SRE, and Kubernetes Administrator interviews, with practical explanations, YAML examples, and troubleshooting commands.

## Section 1: Custom Resources, Labels, Annotations, and Namespaces

### 1. How do you create and use Custom Resources in Kubernetes?

Answer:

A Custom Resource (CR) allows us to extend Kubernetes with our own resource types beyond built-in resources such as Pods, Deployments, and Services.

We use a CustomResourceDefinition (CRD) to define the new resource type, then create Custom Resources from that definition.

For example, suppose we want to manage a custom resource called `Database`.

Step 1: Create a CRD

```
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.example.com
spec:
  group: example.com
  scope: Namespaced
  names:
    plural: databases
    singular: database
    kind: Database
    shortNames:
      - db
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                replicas:
                  type: integer
                  minimum: 1
```

Step 2: Apply the CRD

```
kubectl apply -f database-crd.yaml
```

Step 3: Create a Custom Resource

```
apiVersion: example.com/v1
kind: Database
metadata:
  name: production-db
spec:
  replicas: 3
```

```
kubectl apply -f database.yaml
kubectl get databases
kubectl describe database production-db
```

Interview point: A CRD extends the Kubernetes API. Creating a CRD alone does not automatically create application Pods or databases. A custom controller or Operator is needed to implement that behavior.

### 2. What are labels and annotations in Kubernetes?

Answer:

Labels and annotations are key-value pairs added to Kubernetes objects as metadata.

| Labels                                | Annotations                                 |
| ------------------------------------- | ------------------------------------------- |
| Used to identify and select resources | Used to store additional metadata           |
| Support label selectors               | Not used by standard label selectors        |
| Typically short, meaningful values    | Can contain longer descriptive information  |
| Used by Services and controllers      | Used by tools, automation, and integrations |

Example:

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    environment: production
  annotations:
    description: "Production Nginx web server"
    owner: "platform-team"
spec:
  containers:
    - name: nginx
      image: nginx:stable
```

Real-world example:

A Service selects Pods using labels such as `app: nginx`. An annotation may contain a deployment description, build reference, or configuration hint for another tool.

Interview point: Labels are used for identification and selection; annotations provide supporting metadata.

### 3. How do you add labels and annotations to an existing Pod?

Answer:

Use `kubectl label` and `kubectl annotate`.

Add a label:

```
kubectl label pod nginx-pod environment=production
```

Add an annotation:

```
kubectl annotate pod nginx-pod owner=devops-team
```

View the labels:

```
kubectl get pod nginx-pod --show-labels
```

View annotations and other metadata:

```
kubectl describe pod nginx-pod
```

Update an existing label:

```
kubectl label pod nginx-pod \
  environment=staging --overwrite
```

Remove a label:

```
kubectl label pod nginx-pod environment-
```

Interview point: If the Pod belongs to a Deployment, update the Deployment's Pod template to make the change persist when Pods are recreated.

### 4. What is a namespace in Kubernetes?

Answer:

A namespace is a logical grouping mechanism used to organize and isolate Kubernetes resources within a cluster.

Namespaces are useful for separating applications, projects, teams, or environments.

Common namespaces include:

| Namespace       | Purpose                                                 |
| --------------- | ------------------------------------------------------- |
| default         | Default namespace for namespaced resources              |
| kube-system     | Kubernetes system components                            |
| kube-public     | Publicly readable resources, subject to access controls |
| kube-node-lease | Node heartbeat Lease objects                            |

Create a namespace:

```
kubectl create namespace dev
```

Create a Deployment in that namespace:

```
kubectl create deployment nginx \
  --image=nginx:stable \
  -n dev
```

List Pods in the namespace:

```
kubectl get pods -n dev
```

List all namespaces:

```
kubectl get namespaces
```

Interview point: Namespaces provide logical separation, but they do not automatically enforce network isolation or resource limits. Use NetworkPolicies, RBAC, ResourceQuotas, and LimitRanges as needed.

## Section 2: Kubernetes Services and Networking

### 5. What are the different types of Services in Kubernetes?

Answer:

Kubernetes has four main Service types.

| Service type | Purpose                                                                |
| ------------ | ---------------------------------------------------------------------- |
| ClusterIP    | Exposes an application internally within the cluster                   |
| NodePort     | Exposes an application through a port on nodes                         |
| LoadBalancer | Exposes an application using an external load balancer, when supported |
| ExternalName | Maps a Service DNS name to another DNS hostname                        |

How Kubernetes Services expose applications

ClusterIP

Pod → Service virtual IP → Backend Pod

NodePort

Client → Node IP:NodePort → Backend Pod

LoadBalancer

Client → Load Balancer → Service Backend

ExternalName

Service DNS name → External DNS hostname

Conceptual traffic paths; actual forwarding depends on the Kubernetes networking and load-balancer implementation.

Example:

```
kubectl expose deployment nginx \
  --type=ClusterIP \
  --port=80
```

Interview point: ClusterIP is the default Service type.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://kubernetes.io\&sz=32)

Kubernetes

+1



### 6. What is the difference between ClusterIP and NodePort?

| ClusterIP                              | NodePort                                     |
| -------------------------------------- | -------------------------------------------- |
| Accessible through the cluster network | Exposes a port on each applicable node       |
| Default Service type                   | Must be specified explicitly                 |
| No node-level external port            | Uses a configured node port                  |
| Good for internal APIs                 | Useful for simple external access or testing |

Example:

```
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

Typical NodePort range:

```
30000–32767
```

This range is configurable by cluster administrators.

Access using:

```
http://<node-ip>:30080
```

Interview point: NodePort builds on ClusterIP behavior. External accessibility still depends on node reachability, firewall rules, and cloud security groups.

### 7. What is the difference between NodePort and LoadBalancer Services?

Answer:

| NodePort                                 | LoadBalancer                                           |
| ---------------------------------------- | ------------------------------------------------------ |
| Exposes a port on nodes                  | Requests a load balancer from an integration           |
| Requires clients to reach a node IP      | Typically provides a load balancer address or hostname |
| Does not provision a cloud load balancer | Can automatically provision cloud infrastructure       |
| Common for development and custom setups | Common for production external access                  |
| Uses a node port                         | May use NodePort or direct Pod IP targets              |

AWS EKS example:

A `LoadBalancer` Service can be used with a supported AWS integration to provision a Network Load Balancer.

```
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

```
kubectl get svc nginx-lb
```

Interview point: LoadBalancer Services are convenient for production traffic, but the exact AWS implementation depends on the installed controller and Service configuration.

### 8. When would you recommend NodePort instead of LoadBalancer?

Answer:

I would recommend NodePort in cases such as:

- Development or lab clusters.
- On-premises environments without load-balancer integration.
- Temporary testing of applications.
- Environments with an existing external load balancer that forwards to NodePorts.

Real-world example:

If I am testing an Nginx application in a small Kubernetes lab, I can expose it using NodePort rather than provisioning a cloud load balancer.

Interview point: For production AWS EKS applications, LoadBalancer or an appropriately configured Ingress/Gateway is usually a better option for managed public traffic.

### 9. What Kubernetes Service types are available apart from ClusterIP, NodePort, and LoadBalancer?

Answer:

The fourth official Service type is ExternalName.

ExternalName maps a Kubernetes Service name to another DNS hostname.

Example:

```
apiVersion: v1
kind: Service
metadata:
  name: external-database
spec:
  type: ExternalName
  externalName: db.example.com
```

Applications can resolve:

```
external-database.default.svc.cluster.local
```

The DNS response aliases the Service name to `db.example.com`.

Another important concept is a Headless Service:

```
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
    - port: 3306
```

Headless Services expose individual endpoint addresses through DNS rather than a single virtual ClusterIP. They are commonly used with StatefulSets.

Interview point: ExternalName is a Service type; Headless is a Service configuration, not a fifth official Service type.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://kubernetes.io\&sz=32)

Kubernetes



### 10. How does Kubernetes service discovery work?

Answer:

Kubernetes primarily provides service discovery through DNS.

When a Service is created, the cluster DNS system makes its name resolvable.

Example:

Suppose we have:

- Service: `backend`
- Namespace: `production`
- Cluster domain: `cluster.local`

Its fully qualified DNS name is:

```
backend.production.svc.cluster.local
```

Another Pod can access the application using that name and the Service port.

Frontend Pod

Requests backend.production.svc.cluster.local

CoreDNS

Resolves backend Service to its ClusterIP

Service networking

Forwards application traffic to an eligible backend endpoint

Backend Pod 1

Backend Pod 2

Backend Pod 3

Interview point: Kubernetes service discovery separates logical service names from changing individual Pod IP addresses.

### 11. What is CoreDNS in Kubernetes?

Answer:

CoreDNS is a DNS server commonly used in Kubernetes clusters for internal Service and Pod name resolution.

Its primary responsibilities include:

- Resolving Kubernetes Service DNS names.
- Supporting DNS discovery of cluster resources.
- Forwarding external DNS queries to upstream DNS resolvers.
- Supporting DNS-based service discovery across namespaces.

Check CoreDNS:

```
kubectl get pods -n kube-system \
  -l k8s-app=kube-dns
```

The selector above is common but not universal.

Check DNS Service:

```
kubectl get svc -n kube-system kube-dns
```

Troubleshoot DNS:

```
kubectl run dns-test \
  --image=busybox:stable \
  --restart=Never \
  --command -- sleep 3600
```

```
kubectl exec dns-test -- \
  nslookup kubernetes.default.svc.cluster.local
```

Interview point: CoreDNS resolves names. It does not itself perform the usual ClusterIP Service traffic forwarding.

## Section 3: CNI, Pod Networking, and AWS EKS

### 12. What is the purpose of CNI in Kubernetes?

Answer:

CNI stands for Container Network Interface. It is a specification and plugin system used to configure networking for containerized workloads.

A Kubernetes CNI plugin typically handles:

- Assigning IP addresses to Pods.
- Configuring Pod network interfaces.
- Enabling connectivity between Pods across nodes.
- Configuring routes or networking datapaths.
- Supporting network policies where implemented.

Common CNI implementations include Calico, Cilium, Flannel, and Amazon VPC CNI.

Real-world example: When a new Pod starts in EKS, the AWS VPC CNI configures its networking and assigns an IP address from an appropriate VPC subnet.

Interview point: CNI handles Pod networking. CoreDNS handles name resolution, while Kubernetes Service networking is commonly implemented using kube-proxy or an alternative datapath.

### 13. What is the difference between Calico and the AWS VPC CNI plugin?

Answer:

| Calico                                                                       | AWS VPC CNI                                                      |
| ---------------------------------------------------------------------------- | ---------------------------------------------------------------- |
| Kubernetes networking and network security solution                          | AWS-native Kubernetes networking implementation                  |
| Supports different networking modes, including overlay and routed networking | Integrates with AWS VPC networking and ENIs                      |
| Can use separate Pod CIDR ranges                                             | Typically allocates Pod IPs from VPC subnets                     |
| Strong network policy capabilities                                           | Supports Kubernetes network policies in supported configurations |
| Can run on multiple cloud and on-premises platforms                          | Primarily designed for AWS environments                          |

Real-world example:

Calico can provide Pod networking using an overlay network such as VXLAN.

AWS VPC CNI attaches or manages EC2 network interfaces and assigns VPC-routable addresses to Pods.

Important: Calico networking and AWS VPC CNI are not always mutually exclusive. Calico can also be used to enforce network policies in EKS environments that use AWS VPC CNI for networking.

AWS VPC CNI also supports network policies in supported configurations.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.aws.amazon.com\&sz=32)

Amazon EKS

+1



### 14. How is an IP address assigned to a Pod?

Answer:

A Pod IP is normally assigned as part of Pod network setup by the configured CNI implementation.

Typical process:

1. The scheduler assigns the Pod to a worker node.
2. The kubelet asks the container runtime to create the Pod sandbox.
3. The runtime invokes CNI network configuration.
4. The CNI implementation allocates or obtains an available IP.
5. The Pod network interface and routing are configured.
6. The Pod can communicate according to the cluster network configuration.

In EKS, the AWS VPC CNI commonly uses `ipamd` to manage IP addresses and ENIs on EC2 nodes.

Check Pod IP addresses:

```
kubectl get pods -o wide
```

```
kubectl get pod nginx-pod \
  -o jsonpath='{.status.podIP}'
```

Interview point: The scheduler decides which node runs a Pod; the network implementation is responsible for its Pod IP configuration.

### 15. Does the CNI plugin use the VPC CIDR range or a different CIDR range?

Answer:

It depends on the CNI implementation and network configuration.

In a standard IPv4 EKS cluster using AWS VPC CNI, Pods generally receive IPs from subnets associated with the cluster VPC.

For example:

Example: Default AWS VPC CNI

VPC CIDR: 10.0.0.0/16

Subnet A

10.0.1.0/24

Node 10.0.1.10

Pod 10.0.1.20

Subnet B

10.0.2.0/24

Node 10.0.2.10

Pod 10.0.2.20

Custom networking option

Secondary VPC CIDR: 100.64.0.0/16

Pod subnet example: 100.64.1.0/24

With configured custom networking, Pods can receive addresses from secondary VPC CIDR subnets.

AWS VPC CNI supports custom networking through `ENIConfig` resources, allowing Pod addresses to come from alternate VPC subnets.

Other CNIs, such as Calico, can use separate Pod networks that are not allocated directly from the underlying VPC subnets.

Interview point: Do not confuse the Pod CIDR with the Kubernetes Service CIDR. They serve different purposes.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.aws.amazon.com\&sz=32)

Amazon EKS

+1



## Section 4: CSI, StorageClass, PV, and PVC

### 16. What is the purpose of CSI drivers in Kubernetes?

Answer:

CSI stands for Container Storage Interface.

CSI is a standard interface that allows Kubernetes to integrate with external storage systems without requiring storage-specific logic inside Kubernetes core.

CSI drivers enable functionality such as:

- Provisioning persistent volumes.
- Attaching storage to worker nodes.
- Mounting storage inside Pods.
- Resizing volumes, when supported.
- Creating volume snapshots, when supported.
- Cleaning up storage resources.

Common CSI drivers:

| CSI Driver              | Storage System               |
| ----------------------- | ---------------------------- |
| AWS EBS CSI             | Amazon Elastic Block Store   |
| AWS EFS CSI             | Amazon Elastic File System   |
| Azure Disk CSI          | Azure Managed Disks          |
| Azure File CSI          | Azure Files                  |
| GCE Persistent Disk CSI | Google Cloud Persistent Disk |

Real-world example: In EKS, when a PVC requests 20 GiB of EBS storage using a suitable StorageClass, the AWS EBS CSI driver can provision an EBS volume automatically.

Interview point: CNI handles container networking, while CSI handles container storage.

### 17. What is a Kubernetes StorageClass?

Answer:

A StorageClass defines how Kubernetes should provision storage volumes.

It specifies properties such as the storage provisioner, driver-specific parameters, volume binding mode, and reclaim policy for dynamically provisioned volumes.

Example using AWS EBS:

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-storage
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

Apply:

```
kubectl apply -f storageclass.yaml
```

Check:

```
kubectl get storageclass
```

Important parameters:

- `provisioner`: Specifies the CSI driver.
- `parameters`: Configures storage characteristics.
- `reclaimPolicy`: Determines what happens to dynamically provisioned PVs after claim release.
- `volumeBindingMode`: Controls when storage provisioning and binding occur.
- `allowVolumeExpansion`: Enables supported volume expansion.

Interview point: `WaitForFirstConsumer` is useful for availability-zone-specific storage such as EBS because it allows scheduling requirements to influence volume provisioning.

On EKS Auto Mode, the EBS provisioner differs from the standard EBS CSI provisioner.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.aws.amazon.com\&sz=32)

Amazon EKS



### 18. What are PersistentVolumes and PersistentVolumeClaims?

Answer:

A PersistentVolume (PV) represents storage available to workloads in a Kubernetes cluster.

A PersistentVolumeClaim (PVC) is a request for storage made by a user or workload.

A Pod uses a PVC to access a compatible PV.

Kubernetes Persistent Storage Flow

Application Pod

Mounts storage at /app/data

PersistentVolumeClaim (PVC)

Requests 20 GiB, ReadWriteOnce

PersistentVolume (PV)

Represents the bound storage volume

Underlying Storage

For example, an AWS EBS volume

Example PVC:

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3-storage
  resources:
    requests:
      storage: 20Gi
```

Check storage resources:

```
kubectl get pv
kubectl get pvc
```

Interview point: Pod replacement does not automatically delete the PVC. Persistent storage can survive Pod restarts and replacements, subject to the storage configuration and lifecycle policies.

### 19. What is the difference between PV and PVC?

| PersistentVolume (PV)                            | PersistentVolumeClaim (PVC)                         |
| ------------------------------------------------ | --------------------------------------------------- |
| Represents cluster storage                       | Requests storage                                    |
| Cluster-scoped resource                          | Namespace-scoped resource                           |
| Created manually or by a provisioner             | Usually created by application users or controllers |
| Defines storage capacity and access capabilities | Defines requested storage and access modes          |
| Binds to a compatible PVC                        | Binds to a compatible PV                            |

Example:

A PV represents an existing 100 GiB storage volume.

A PVC requests 20 GiB. If compatibility and binding requirements are met, Kubernetes can bind the claim to the PV.

Interview point: A PVC does not necessarily create storage directly. A StorageClass and provisioner can dynamically create the PV and underlying storage when needed.

### 20. What are the different PVC access modes?

Answer:

Kubernetes supports four main PersistentVolume access modes.

| Mode             | Short name | Meaning                                 |
| ---------------- | ---------- | --------------------------------------- |
| ReadWriteOnce    | RWO        | Read-write mount by one node            |
| ReadOnlyMany     | ROX        | Read-only mount by multiple nodes       |
| ReadWriteMany    | RWX        | Read-write mount by multiple nodes      |
| ReadWriteOncePod | RWOP       | Read-write mount exclusively by one Pod |

Important distinction:

`ReadWriteOnce` means a single node, not necessarily a single Pod. Multiple Pods on the same node may be able to mount the volume.

`ReadWriteOncePod` provides Pod-level exclusivity for supported CSI storage.

Real-world examples:

- AWS EBS: Commonly RWO.
- AWS EFS: Commonly RWX.
- Database requiring exclusive Pod-level access: RWOP, if supported.

Interview point: Supported access modes depend on the underlying storage driver. Listing a mode in a PVC does not make an incompatible storage backend support it.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://kubernetes.io\&sz=32)

Kubernetes



### 21. What is the difference between static and dynamic volume provisioning?

Answer:

| Static Provisioning                       | Dynamic Provisioning                           |
| ----------------------------------------- | ---------------------------------------------- |
| Administrator prepares storage in advance | Storage is provisioned on demand               |
| PV is created manually                    | PV is created by a provisioner                 |
| Suitable for existing storage volumes     | Suitable for automated application deployments |
| More manual lifecycle management          | Reduced manual provisioning work               |
| PVC binds to an existing suitable PV      | PVC can trigger volume creation                |

Static provisioning example:

An administrator creates an EBS volume and a corresponding PV. An application creates a PVC that binds to it.

Dynamic provisioning example:

An application creates a PVC referencing an EBS StorageClass. The CSI driver provisions a new EBS volume and Kubernetes binds the newly created PV to the PVC.

Interview point: Dynamic provisioning generally requires a suitable storage provisioner, commonly a CSI driver, and a StorageClass.

### 22. How do you mount a PVC into a Deployment?

Answer:

First, create a PVC, then reference it in the Deployment using `volumes` and `volumeMounts`.

Step 1: Create the PVC

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3-storage
  resources:
    requests:
      storage: 5Gi
```

Step 2: Mount it into the Deployment

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
spec:
  replicas: 1
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
          volumeMounts:
            - name: app-storage
              mountPath: /usr/share/nginx/html
      volumes:
        - name: app-storage
          persistentVolumeClaim:
            claimName: nginx-pvc
```

Apply both files:

```
kubectl apply -f pvc.yaml
kubectl apply -f deployment.yaml
```

Verify:

```
kubectl get pvc
kubectl get pods
kubectl describe pod <pod-name>
```

This example assumes the StorageClass and EBS CSI driver are installed and configured.

Interview point: For EBS, avoid scaling a Deployment that shares one RWO PVC across different nodes. Use a StatefulSet with separate PVCs for each replica, or choose suitable shared storage where appropriate.

## Section 5: AWS EBS, EFS, and StatefulSets

### 23. What is the difference between EBS and EFS storage in Kubernetes?

Answer:

Amazon EBS provides block storage, while Amazon EFS provides a managed shared file system.

| Amazon EBS                                   | Amazon EFS                                                |
| -------------------------------------------- | --------------------------------------------------------- |
| Block storage                                | Network file storage                                      |
| Generally tied to a single Availability Zone | Regional file system with multi-AZ access when configured |
| Commonly supports RWO                        | Commonly supports RWX                                     |
| Well suited for databases                    | Well suited for shared files                              |
| Uses EBS CSI driver                          | Uses EFS CSI driver                                       |
| Volume capacity is provisioned               | File system capacity grows and shrinks with stored data   |

EBS

Dedicated block volume

Database Pod

EBS Volume

Typical use: PostgreSQL data storage

EFS

Shared network file system

Pod A · Pod B · Pod C

Shared EFS

Typical use: shared uploads and documents

Real-world example:

For a PostgreSQL database requiring its own filesystem, EBS is often a suitable choice.

For multiple application Pods accessing the same shared documents, EFS is often more suitable.

Interview point: Standard EBS volumes have AZ and attachment constraints. EFS supports shared file access through NFS, making it a common choice for RWX workloads.

### 24. What is the purpose of the EFS CSI driver?

Answer:

The Amazon EFS CSI driver allows Kubernetes Pods to mount Amazon EFS file systems through Kubernetes persistent storage resources.

It provides integration for:

- Mounting EFS file systems into Pods.
- Static PV provisioning.
- Dynamic provisioning through EFS access points in supported configurations.
- Shared file access across multiple eligible Pods and nodes.

Example EFS StorageClass:

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-0123456789abcdef0
  directoryPerms: "700"
```

The file system ID must be replaced with an actual EFS file system ID.

Example PVC:

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
```

Check the EFS CSI driver:

```
kubectl get csidrivers
```

```
kubectl get pods -n kube-system
```

Important: The EFS file system must already exist and have suitable mount targets, networking, security groups, and driver permissions. The storage size requested in an EFS PVC is not a hard EFS filesystem capacity quota.

The EFS CSI driver supports dynamic provisioning on supported EC2-based EKS configurations. Fargate has different restrictions.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.aws.amazon.com\&sz=32)

Amazon EKS



### 25. What is the difference between a StatefulSet and a Deployment when using persistent storage?

Answer:

Both Deployments and StatefulSets can mount persistent storage, but they differ in how storage identities are managed across replicas.

| Deployment                                        | StatefulSet                                        |
| ------------------------------------------------- | -------------------------------------------------- |
| Can mount a PVC                                   | Can mount PVCs                                     |
| Typically uses manually defined claim references  | Supports `volumeClaimTemplates`                    |
| Pods are interchangeable                          | Pods have stable ordinal identities                |
| Replicas can reference the same PVC               | Each replica can automatically receive its own PVC |
| Common for stateless workloads and shared storage | Common for databases and other stateful workloads  |

Example: StatefulSet with a PVC per replica

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql-headless
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
          image: mysql:8.4
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: root-password
          ports:
            - containerPort: 3306
          volumeMounts:
            - name: mysql-storage
              mountPath: /var/lib/mysql
  volumeClaimTemplates:
    - metadata:
        name: mysql-storage
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: gp3-storage
        resources:
          requests:
            storage: 10Gi
```

Assume the headless Service, Secret, StorageClass, and CSI driver already exist.

With three replicas, Kubernetes creates storage claims resembling:

```
mysql-storage-mysql-0
mysql-storage-mysql-1
mysql-storage-mysql-2
```

Each Pod gets its own persistent claim.

Interview point: StatefulSets provide stable storage identity, but they do not automatically configure MySQL replication. That requires additional database configuration or an Operator.

## Quick Revision Cheat Sheet

| Topic            | Remember this                                  |
| ---------------- | ---------------------------------------------- |
| CRD              | Extends Kubernetes API with new resource types |
| Label            | Resource identification and selection          |
| Annotation       | Additional resource metadata                   |
| Namespace        | Logical resource organization                  |
| ClusterIP        | Internal Service access                        |
| NodePort         | Service exposed through node ports             |
| LoadBalancer     | External load-balancer integration             |
| ExternalName     | DNS alias to another hostname                  |
| Headless Service | Direct endpoint discovery without a ClusterIP  |
| CoreDNS          | DNS resolution                                 |
| CNI              | Pod networking                                 |
| AWS VPC CNI      | VPC-integrated Pod networking                  |
| Calico           | Networking and network policy enforcement      |
| CSI              | Storage integration                            |
| StorageClass     | Defines storage provisioning                   |
| PV               | Actual storage resource abstraction            |
| PVC              | Storage request                                |
| RWO              | Read-write access on one node                  |
| RWX              | Read-write access on multiple nodes            |
| RWOP             | Exclusive read-write access by one Pod         |
| EBS              | Block storage                                  |
| EFS              | Shared file storage                            |
| StatefulSet      | Stable per-replica identities and storage      |

## AWS EKS Troubleshooting Commands

```
# Check all Services
kubectl get svc -A

# Check Service endpoints
kubectl get endpointslices -A

# Check DNS
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check AWS VPC CNI DaemonSet
kubectl get ds aws-node -n kube-system

# Check Pod IP assignments
kubectl get pods -A -o wide

# Check CSI drivers
kubectl get csidrivers

# Check StorageClasses
kubectl get storageclass

# Check all PVCs
kubectl get pvc -A

# Check PVs
kubectl get pv

# Troubleshoot a pending PVC
kubectl describe pvc <pvc-name>

# Investigate Pod volume-mount failures
kubectl describe pod <pod-name>

# Check StatefulSets
kubectl get statefulsets -A
```

## Practice Interview Quiz

Knowledge check

Question 1 of 6

What is required for a Custom Resource to automatically manage an application?

Only a CRD

A custom controller or Operator

Only a Namespace

CoreDNS

PreviousNext&#x20;

Interview preparation tip: For AWS EKS roles, give extra attention to questions 13–15 and 17–25. Be ready to explain AWS VPC CNI IP allocation, the difference between EBS and EFS, why an EBS PVC might remain Pending, and how StatefulSets provide dedicated persistent storage to each database replica.
