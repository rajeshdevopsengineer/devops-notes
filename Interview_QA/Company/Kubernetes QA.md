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


# Kubernetes Interview Questions and Answers (51–100)

Advanced DevOps and AWS EKS Interview Preparation

These 50 questions cover real-world Kubernetes scenarios involving persistent storage, application deployments, networking, Ingress, security, and service mesh.

The answers are written so you can explain each concept confidently in an interview, with practical commands and YAML examples.

## Section 1: StatefulSets, PVCs, EBS, EFS, and Application Data

### 1. Why do we need StatefulSets when a Deployment can also use PVCs?

Answer:

Although both Deployments and StatefulSets can use PersistentVolumeClaims, StatefulSets provide stable identity and persistent storage management for each replica.

| Deployment                                 | StatefulSet                                   |
| ------------------------------------------ | --------------------------------------------- |
| Pods are interchangeable                   | Each Pod has a stable identity                |
| Replicas may reference the same PVC        | Can create dedicated PVCs per replica         |
| Pods have generated names                  | Pods have predictable names like `mysql-0`    |
| Designed primarily for stateless workloads | Designed for stateful workloads               |
| No built-in ordered startup requirement    | Supports ordered Pod creation and termination |

Real-world example:

If we deploy MongoDB with three replicas using a StatefulSet:

```
mongo-0 → pvc-mongo-0 → EBS Volume 1
mongo-1 → pvc-mongo-1 → EBS Volume 2
mongo-2 → pvc-mongo-2 → EBS Volume 3
```

Each database Pod has its own persistent storage.

Interview point: StatefulSets are recommended for databases because they provide stable identities and per-replica storage, but database replication and failover require additional configuration or an Operator.

### 2. What happens when a Deployment sharing one PVC is scaled from one replica to three replicas?

Answer:

It depends on the PVC's access mode, storage backend, and where Kubernetes schedules the three Pods.

Consider a Deployment using a single AWS EBS PVC with `ReadWriteOnce`.

One EBS PVC shared by three replicas

Pod 1

Pod 2

Pod 3

Single PVC (ReadWriteOnce)

AWS EBS Volume

Normally attached to one EC2 node

Scenario A: All three Pods run on the same node

They may be able to mount the same RWO volume. However, simultaneous writes could corrupt application data if the application isn't designed for shared filesystem access.

Scenario B: Pods run on different nodes

With a standard EBS volume, only the node holding the volume attachment can mount it. Other Pods may remain in `ContainerCreating` or experience `Multi-Attach` errors.

Solution: Use a StatefulSet with individual PVCs for database replicas, or use EFS when multiple Pods need shared filesystem storage.

Interview point: `ReadWriteOnce` means a volume can be mounted read-write by one node, not necessarily one Pod.

### 3. Can multiple Pods on the same node share an EBS volume?

Answer:

Yes. Multiple Pods running on the same node can generally mount the same EBS-backed `ReadWriteOnce` PVC, provided the filesystem and application usage are compatible.

Example:

```
EC2 Worker Node
  ├── Pod A ──┐
  ├── Pod B ──┼── PVC → EBS
  └── Pod C ──┘
```

However, this does not guarantee safe concurrent file writes.

For exclusive mounting by a single Pod, use `ReadWriteOncePod` with a compatible CSI driver.

Advanced interview point: Supported EBS `io2` Multi-Attach configurations are a specialized exception for multi-node attachment, requiring block mode and application-level coordination. They are not a general replacement for EFS.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub

+1



### 4. How would you implement shared storage across multiple Pods running on different EKS nodes?

Answer:

For general shared filesystem access, I would use Amazon EFS with the EFS CSI driver.

Implementation steps:

1. Create an Amazon EFS filesystem.
2. Configure mount targets in the appropriate VPC subnets/AZs.
3. Allow NFS port 2049 through the required security groups.
4. Install and configure the EFS CSI driver.
5. Create a StorageClass using `efs.csi.aws.com`.
6. Create a PVC with `ReadWriteMany`.
7. Mount the same PVC into multiple Pods.

Example PVC:

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-data
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
```

Interview point: EFS supports shared filesystem access across nodes. Ensure application-level concurrency safety, EFS permissions, and appropriate access-point configuration.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.aws.amazon.com\&sz=32)

Amazon EKS



### 5. What happens to data when a database Pod goes down?

Answer:

The result depends on where the database stores its data.

| Storage type                | What happens when the Pod fails?                                     |
| --------------------------- | -------------------------------------------------------------------- |
| Container writable layer    | Data in the failed container can be lost when it is replaced         |
| `emptyDir`                  | Survives container restarts within the same Pod, but not Pod removal |
| EBS-backed PVC              | Data remains on the persistent volume                                |
| EFS-backed PVC              | Data remains in the shared file system                               |
| Externally managed database | Data remains in the external database                                |

With an EBS-backed PVC, a replacement Pod can reattach and mount the volume on a compatible node in the correct Availability Zone.

Important: Storage persistence does not guarantee database availability. Database recovery may still require crash recovery, transaction log replay, replication, or failover.

### 6. What happens to sticky session data when a Pod goes down?

Answer:

Sticky sessions direct requests from a particular user to the same backend Pod.

If that Pod fails, any session state stored only in that Pod's memory may be lost.

The traffic router can direct subsequent requests to another healthy Pod, but that Pod may not have the previous session data.

Recommended solution:

Store session data in an external shared store such as Redis, rather than relying on Pod-local memory.

```
User → Ingress → Pod A
                   ↓
                Redis
                   ↑
User → Ingress → Pod B
```

If Pod A fails, Pod B can retrieve the user's session state from Redis.

Interview point: Session affinity does not provide session persistence. Externalized session storage is more resilient.

### 7. What is the difference between stateful and stateless applications?

| Stateful Application                | Stateless Application                                  |
| ----------------------------------- | ------------------------------------------------------ |
| Maintains durable application state | Does not rely on local instance state between requests |
| Often needs persistent storage      | Can often use ephemeral containers                     |
| Replica identity may matter         | Replicas are generally interchangeable                 |
| Recovery can involve stored data    | Failed Pods can usually be replaced easily             |
| Example: MongoDB                    | Example: REST API                                      |

Interview point: A stateless application can still read and write data. The important distinction is that the individual application instance does not need to retain that data locally between requests.

### 8. How do you manage data for stateless applications?

Answer:

Stateless applications typically store durable data outside their own Pods.

Common patterns include:

- Amazon RDS or DynamoDB for persistent application data.
- Redis for sessions or caching.
- Amazon S3 for uploaded files and objects.
- EFS where a shared filesystem is necessary.
- Secrets Manager or Kubernetes Secrets for credentials.
- ConfigMaps for non-sensitive application configuration.

Real-world example: A Spring Boot API runs as three interchangeable Pods, stores business data in RDS, and stores user sessions in Redis.

If one Pod fails, another Pod can serve the user without depending on the failed Pod's local data.

### 9. How do you deploy MongoDB in an EKS cluster?

Answer:

For a production MongoDB deployment, I would generally use a supported MongoDB Operator, a validated Helm-based deployment, or a managed database service such as MongoDB Atlas, depending on requirements.

For self-managed MongoDB on EKS:

1. Create an EKS cluster with suitable EC2 worker nodes.
2. Install the EBS CSI driver with appropriate IAM permissions.
3. Create a `gp3` StorageClass using `WaitForFirstConsumer`.
4. Install the MongoDB Operator.
5. Configure a MongoDB replica set with three members.
6. Provision separate PVCs for each member.
7. Enable authentication, TLS, backups, monitoring, and suitable Pod distribution.
8. Configure internal Services for MongoDB member discovery.

Example conceptual architecture:

```
MongoDB Replica Set
├── mongo-0 → EBS PVC 1
├── mongo-1 → EBS PVC 2
└── mongo-2 → EBS PVC 3
```

With proper replica set configuration, the database can elect a new primary if the current primary fails and a voting majority remains available.

Interview point: A StatefulSet alone does not configure a MongoDB replica set. Use an Operator or implement replication, authentication, lifecycle, and recovery configuration separately.

### 10. What storage limitations exist when using EBS volumes in EKS?

Answer:

Important limitations include:

1. Availability Zone restriction: An EBS volume must normally be attached to an EC2 node in its own AZ.
2. Attachment restrictions: Standard filesystem volumes cannot be mounted read-write across multiple nodes simultaneously.
3. Volume attachment limits: EC2 instance types have limits on attached EBS volumes.
4. Scheduling constraints: Pods can remain Pending if no suitable node can use the required volume.
5. Fargate restrictions: EBS volumes cannot be mounted to EKS Fargate Pods.
6. Performance limits: IOPS and throughput depend on volume configuration and underlying EC2 limits.

Recommended practice is to use `WaitForFirstConsumer`, appropriate topology constraints, and separate PVCs for independent stateful replicas.

Interview point: EKS Auto Mode uses the `ebs.csi.eks.amazonaws.com` provisioner, while the standard EBS CSI driver uses `ebs.csi.aws.com`.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.aws.amazon.com\&sz=32)

Amazon EKS



## Section 2: Deployments, YAML, and kubectl

### 11. How do you deploy an application in Kubernetes?

Answer:

A typical application deployment process is:

1. Build the application's container image.
2. Push the image to Amazon ECR, Docker Hub, or another registry.
3. Create a Kubernetes Deployment manifest.
4. Configure a Service to expose the application.
5. Apply the manifests using `kubectl`.
6. Verify the rollout, Pods, logs, and connectivity.

Example:

```
kubectl create deployment nginx-app \
  --image=nginx:stable \
  --replicas=3

kubectl expose deployment nginx-app \
  --port=80 \
  --type=ClusterIP

kubectl rollout status deployment/nginx-app
```

Interview point: In production, use declarative YAML, Helm, Kustomize, or GitOps to make deployments reproducible.

### 12. How do you write Kubernetes Deployment YAML files?

Answer:

A Deployment YAML file defines the application name, desired replica count, Pod selector, Pod template, and container configuration.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: nginx
          image: nginx:stable
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              memory: "256Mi"
```

Deploy:

```
kubectl apply -f deployment.yaml
kubectl get deployments
kubectl get pods
```

Interview point: The labels in `spec.selector.matchLabels` must match those in `spec.template.metadata.labels`.

### 13. Explain the different fields in a Kubernetes Deployment YAML file.

| Field                           | Purpose                                   |
| ------------------------------- | ----------------------------------------- |
| `apiVersion`                    | API version, normally `apps/v1`           |
| `kind`                          | Resource type, `Deployment`               |
| `metadata.name`                 | Deployment name                           |
| `metadata.namespace`            | Target namespace                          |
| `spec.replicas`                 | Desired number of Pods                    |
| `spec.selector`                 | Identifies Pods managed by the Deployment |
| `spec.template`                 | Defines the Pod template                  |
| `spec.template.metadata.labels` | Labels assigned to Pods                   |
| `spec.template.spec.containers` | Application container definitions         |
| `image`                         | Container image                           |
| `ports.containerPort`           | Declares an application container port    |
| `resources.requests`            | CPU/memory used for scheduling            |
| `resources.limits`              | Configured resource limits                |
| `env`                           | Environment variables                     |
| `volumeMounts`                  | Container filesystem mounts               |
| `volumes`                       | Pod-level volume definitions              |
| `strategy`                      | Deployment update strategy                |

Interview point: `containerPort` is metadata about the container's intended port; it does not automatically expose the application outside the Pod.

### 14. How do you deploy an Apache HTTPD container with three replicas?

Answer:

Create a Deployment:

```
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

```
kubectl apply -f apache.yaml
```

Expose internally:

```
kubectl expose deployment apache \
  --name=apache-service \
  --port=80 \
  --target-port=80
```

Verify:

```
kubectl get deployment apache
kubectl get pods -l app=apache
kubectl get svc apache-service
```

Interview point: The Deployment maintains three replicas and replaces failed Pods to restore the desired count.

### 15. How do you deploy a Tomcat application on every Kubernetes worker node?

Answer:

Use a DaemonSet, because it ensures one Pod runs on each eligible node.

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: tomcat-daemon
spec:
  selector:
    matchLabels:
      app: tomcat
  template:
    metadata:
      labels:
        app: tomcat
    spec:
      nodeSelector:
        kubernetes.io/os: linux
      containers:
        - name: tomcat
          image: tomcat:10.1
          ports:
            - containerPort: 8080
```

This targets eligible Linux nodes. To target only worker nodes, apply a dedicated worker label and add it to `nodeSelector` or node affinity, rather than assuming every Linux node is a worker.

Interview point: A DaemonSet is technically appropriate for one Pod per node, although ordinary Tomcat web applications are normally better deployed using Deployments.

### 16. How do you deploy multiple Pods across all worker nodes?

Answer:

There are two different requirements:

One Pod on every worker node: Use a DaemonSet.

Multiple application replicas distributed among worker nodes: Use a Deployment with topology spread constraints or Pod anti-affinity.

Example Deployment distribution constraint:

```
spec:
  replicas: 6
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: web
      containers:
        - name: nginx
          image: nginx:stable
```

This encourages even distribution across eligible nodes. It does not guarantee coverage of every worker node.

### 17. What happens internally when you run `kubectl apply`?

Answer:

When we execute:

```
kubectl apply -f deployment.yaml
```

Kubernetes performs the following general sequence:

kubectl

Reads manifest and sends API request

kube-apiserver

Authenticates, authorizes, validates and admits request

etcd

Persists the updated cluster state

Deployment Controller

Reconciles Deployment and ReplicaSet

ReplicaSet Controller

Creates the required Pod objects

Scheduler

Assigns unscheduled Pods to nodes

kubelet

Starts containers through the runtime

CNI / CSI

Configures required networking and storage

The exact API operation depends on client-side versus server-side apply, existing objects, and field ownership.

Interview point: `kubectl apply` submits the desired configuration; the API server and Kubernetes controllers then reconcile the cluster toward that state. The command does not directly start containers.

### 18. What are the commonly used kubectl commands?

| Command                                 | Purpose                    |
| --------------------------------------- | -------------------------- |
| `kubectl get pods -A`                   | List all Pods              |
| `kubectl get nodes -o wide`             | List nodes                 |
| `kubectl describe pod POD`              | Inspect Pod details/events |
| `kubectl logs POD`                      | View container logs        |
| `kubectl exec -it POD -- sh`            | Execute shell in container |
| `kubectl apply -f file.yaml`            | Apply manifest             |
| `kubectl delete -f file.yaml`           | Delete manifest resources  |
| `kubectl get svc -A`                    | List Services              |
| `kubectl get ingress -A`                | List Ingress resources     |
| `kubectl rollout status deploy/APP`     | Check rollout              |
| `kubectl rollout undo deploy/APP`       | Roll back Deployment       |
| `kubectl scale deploy APP --replicas=3` | Change replicas            |
| `kubectl top pods`                      | Display Pod resource usage |
| `kubectl get events -A`                 | List events                |
| `kubectl config get-contexts`           | List contexts              |
| `kubectl auth can-i create pods`        | Check API permission       |

### 19. How do you list Pods running on specific nodes?

Answer:

List Pods and their nodes:

```
kubectl get pods -A -o wide
```

List Pods on a particular node:

```
kubectl get pods -A \
  --field-selector spec.nodeName=worker-node-1
```

Find a Pod's assigned node:

```
kubectl get pod nginx-pod \
  -o jsonpath='{.spec.nodeName}'
```

Interview point: The `spec.nodeName` field identifies the node where a Pod is assigned.

### 20. How do you log in to a Pod using kubectl?

Answer:

Use `kubectl exec` to execute a shell inside a running container:

```
kubectl exec -it nginx-pod -- /bin/sh
```

If Bash is available:

```
kubectl exec -it nginx-pod -- /bin/bash
```

For a multi-container Pod:

```
kubectl exec -it nginx-pod \
  -c nginx -- /bin/sh
```

For containers without shell utilities, `kubectl debug` can be useful when permissions and cluster features allow it.

Interview point: `kubectl exec` does not perform an SSH login. It starts a process inside a container.

### 21. How do you switch between Kubernetes clusters using kubectl?

Answer:

Use Kubernetes contexts configured in your kubeconfig file.

List contexts:

```
kubectl config get-contexts
```

Display the current context:

```
kubectl config current-context
```

Switch contexts:

```
kubectl config use-context production-cluster
```

For EKS, create or update a kubeconfig context:

```
aws eks update-kubeconfig \
  --region ap-south-1 \
  --name production-eks
```

Interview point: Always verify your current context and namespace before modifying a production cluster.

### 22. What is a Kubernetes context?

Answer:

A Kubernetes context is a named combination of:

- A cluster connection configuration.
- A user or authentication configuration.
- An optional default namespace.

Kubernetes contexts are normally stored in kubeconfig.

Example:

```
kubectl config get-contexts
```

Set the namespace for the current context:

```
kubectl config set-context --current \
  --namespace=production
```

Interview point: A context helps switch environments, but it does not itself grant permissions. Permissions are enforced by the target cluster.

### 23. How do you access a private Kubernetes cluster using kubectl?

Answer:

For a private EKS API endpoint, the machine running `kubectl` must have network connectivity to the private API endpoint and valid Kubernetes authentication/authorization.

Common solutions include:

- Connecting through a VPN.
- Using AWS Direct Connect.
- Running `kubectl` on an authorized EC2 administration host.
- Using an SSM-managed host inside the connected VPC.
- Using a secured network path through peering or a transit gateway.

Example:

```
aws eks update-kubeconfig \
  --region ap-south-1 \
  --name private-eks

kubectl get nodes
```

Interview point: Updating kubeconfig is insufficient without network routing, DNS resolution, security rules, AWS identity permissions, and Kubernetes authorization.

## Section 3: Services, Ingress, Gateway API, and Traffic Flow

### 24. How do you configure Services and Ingress for microservices?

Answer:

A typical microservices design uses one ClusterIP Service per microservice and an Ingress or Gateway to route external HTTP requests.

Example:

```
api.example.com/users  → user-service → User Pods
api.example.com/orders → order-service → Order Pods
```

First define the Services:

```
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  selector:
    app: users
  ports:
    - port: 80
      targetPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: orders
  ports:
    - port: 80
      targetPort: 8080
```

Then create the Ingress:

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: microservices-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /users
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 80
          - path: /orders
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 80
```

This assumes a compatible NGINX-based Ingress Controller is installed and owns the `nginx` IngressClass. The backends must also exist with matching labels and ports.

Interview point: An Ingress defines routing rules. An Ingress Controller implements those rules.

### 25. What is an Ingress in Kubernetes?

Answer:

Ingress is a Kubernetes API resource used to describe HTTP and HTTPS routing from outside a cluster to Services inside the cluster.

It supports features such as:

- Host-based routing.
- Path-based routing.
- TLS termination configuration.
- Routing multiple applications through a shared entry point.

Example:

```
shop.example.com/products → product-service
shop.example.com/orders   → order-service
```

Interview point: Creating an Ingress resource alone does not expose traffic. You need a compatible Ingress Controller.

### 26. What is an Ingress Controller?

Answer:

An Ingress Controller is a component that watches Kubernetes Ingress resources and configures the actual routing infrastructure.

For example, an NGINX-based controller may configure NGINX proxies to forward HTTP requests to application backends.

An AWS Load Balancer Controller can translate supported Ingress resources into AWS Application Load Balancer configuration.

Difference:

| Ingress                                | Ingress Controller                   |
| -------------------------------------- | ------------------------------------ |
| Kubernetes configuration resource      | Software that implements routing     |
| Defines desired routing rules          | Configures proxies or load balancers |
| Does not independently process traffic | Manages the actual traffic path      |

### 27. How do you configure an Ingress Controller?

Answer:

The process depends on the controller.

For AWS EKS using AWS Load Balancer Controller, the broad setup is:

1. Configure the controller's required IAM permissions.
2. Configure EKS Pod Identity or IRSA as supported.
3. Install the AWS Load Balancer Controller using its supported Helm chart.
4. Configure VPC subnet discovery or explicit subnet selection.
5. Create an Ingress resource using the appropriate IngressClass.
6. Configure AWS ALB listener, scheme, certificates, and target settings.
7. Point DNS to the resulting load balancer.

Example ALB Ingress:

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-alb
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
```

Interview point: AWS ALB operates at Layer 7 and provides HTTP/HTTPS features. An AWS NLB is a Layer 4 load balancer and is commonly used with `LoadBalancer` Services.

### 28. What is the difference between Ingress and Egress?

| Ingress                                                 | Egress                                                                   |
| ------------------------------------------------------- | ------------------------------------------------------------------------ |
| Traffic entering a workload or network boundary         | Traffic leaving a workload or network boundary                           |
| Example: Browser accessing a web application            | Example: Application calling an external API                             |
| Can be controlled by Ingress routing and security rules | Can be restricted through NetworkPolicies, firewalls, or egress gateways |
| Often called north-south inbound traffic                | May be external or internal outbound traffic                             |

Important: Kubernetes Ingress is also the name of a specific HTTP routing API. The term ingress in NetworkPolicies simply means incoming network traffic.

### 29. What is the difference between Kubernetes Ingress and Gateway API?

Answer:

Gateway API is a newer, more expressive Kubernetes networking API designed to support routing and traffic management across different protocols.

| Ingress                                            | Gateway API                                                 |
| -------------------------------------------------- | ----------------------------------------------------------- |
| Primarily HTTP/HTTPS routing                       | Supports HTTP, gRPC, and additional protocol routing APIs   |
| Uses the `Ingress` resource                        | Uses resources such as GatewayClass, Gateway, and HTTPRoute |
| Limited native routing model                       | More expressive and extensible routing model                |
| Advanced features often use controller annotations | More features have portable API definitions                 |
| Less granular role separation                      | Explicit infrastructure and routing ownership               |

Gateway API commonly separates responsibilities:

- GatewayClass: Defines the controller/implementation.
- Gateway: Defines listeners and traffic entry points.
- HTTPRoute: Defines HTTP routing behavior.

Interview point: Gateway API is positioned as the next generation of Kubernetes traffic routing APIs, but a supported Gateway API implementation/controller is still required.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://gateway-api.sigs.k8s.io\&sz=32)

Gateway API



### 30. Explain the complete request flow from a user accessing a URL until the request reaches an application Pod.

Answer:

Consider a public application deployed on AWS EKS with an Application Load Balancer and AWS Load Balancer Controller using IP targets.

External request flow — AWS EKS ALB

User's Browser

Requests https\://app.example.com

Public DNS

Resolves hostname to the ALB

AWS Application Load Balancer

Accepts HTTPS request and evaluates listener rules

ALB Target Group

Selects a healthy registered Pod IP target

EKS Pod Networking

Routes traffic to the target Pod IP

Application Pod

Application handles the HTTP request

Example using ALB IP target mode. Other controllers and NodePort/instance-target implementations have different forwarding paths.

Detailed explanation:

1. A user enters `https://app.example.com`.
2. DNS resolves the domain to the external load balancer.
3. The ALB receives the HTTPS connection.
4. Its listener evaluates the configured routing rules.
5. The matching target group selects a healthy registered Pod IP.
6. VPC networking delivers traffic to the Pod.
7. The application container processes the request and returns a response.

The AWS Load Balancer Controller reconciles Kubernetes Ingress and Service/Endpoint information into AWS ALB resources. It is not generally in the request datapath.

Interview point: With ALB IP targets, traffic can go directly to Pod IPs. It does not necessarily pass through kube-proxy or a ClusterIP Service.

### 31. How does traffic flow inside a Kubernetes cluster?

Answer:

There are two common cases.

Pod-to-Pod communication

```
Pod A → CNI-configured network → Pod B
```

Pods can generally communicate using their IP addresses without requiring a Service, assuming routing and security policies allow it.

Pod-to-Service communication

```
Pod A
  ↓
ClusterIP Service
  ↓
Service networking implementation
  ↓
Backend Pod B
```

The Service provides a stable virtual address while backend Pods may be created, removed, or replaced.

Interview point: DNS resolves Service names; kube-proxy or an alternative datapath commonly implements ClusterIP forwarding.

### 32. How do you expose a Kubernetes application to external users?

Answer:

Common approaches:

| Approach                | Typical use                                    |
| ----------------------- | ---------------------------------------------- |
| NodePort                | Testing or external load-balancer integrations |
| LoadBalancer Service    | TCP/UDP applications or direct load balancing  |
| Ingress                 | HTTP/HTTPS applications                        |
| Gateway API             | Advanced and structured traffic routing        |
| API Gateway integration | Centralized external API management            |

For an EKS web application, I would often use an ALB-backed Ingress or Gateway implementation.

For a Layer 4 application, I might use an NLB-backed LoadBalancer Service.

### 33. How do you expose an application internally without using LoadBalancer or NodePort?

Answer:

Use a ClusterIP Service.

```
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
```

Other Pods can access:

```
http://backend-service:80
```

Or across namespaces:

```
http://backend-service.production.svc.cluster.local
```

Interview point: ClusterIP is intended for internal cluster networking, although private connectivity from outside the cluster may also be possible depending on routing.

### 34. How do two Pods communicate without using a Service?

Answer:

Two Pods can communicate directly using their Pod IP addresses.

```
Pod A (10.0.1.20)
       ↓
Pod B (10.0.2.30)
```

Example:

```
kubectl get pods -o wide
```

From a Pod that has curl installed:

```
curl http://10.0.2.30:8080
```

However, Pod IP addresses can change when Pods are recreated.

Interview point: Direct Pod-to-Pod communication is possible, but Services provide stable discovery and routing.

### 35. Can two Pods on the same worker node communicate?

Answer:

Yes. Two Pods on the same worker node can communicate using their Pod IPs.

The CNI configures the required virtual interfaces, networking, and routing.

NetworkPolicy rules may still restrict their communication.

Interview point: Pods on the same node normally have different Pod IP addresses. They do not share `localhost` unless they are containers within the same Pod or special networking is configured.

### 36. Can Pods running on different worker nodes communicate?

Answer:

Yes. Kubernetes networking is designed to enable Pod-to-Pod connectivity across nodes.

The exact implementation depends on the CNI.

For example:

- AWS VPC CNI uses VPC-integrated Pod networking.
- Calico can use routed or overlay networking.
- Cilium supports various networking modes.

Interview point: Cross-node Pod communication depends on correct routes, CNI configuration, security groups, firewalls, and network policies.

### 37. How do Pods communicate across different namespaces?

Answer:

Pods can communicate across namespaces using Pod IPs or Services.

The recommended approach is Service DNS.

Example:

A Pod in namespace `frontend` needs to access the `payment-service` in namespace `backend`.

```
payment-service.backend.svc.cluster.local
```

Application request:

```
curl http://payment-service.backend.svc.cluster.local:8080
```

Interview point: Kubernetes namespaces do not automatically block Pod-to-Pod communication. NetworkPolicies are needed for network-level isolation.

### 38. How do you securely allow two Pods in different namespaces to communicate?

Answer:

Use namespace labels, Pod labels, and NetworkPolicies to restrict traffic to authorized workloads.

For stronger security, combine these controls with authenticated TLS or mTLS.

Example scenario:

- `frontend` namespace contains client Pods labeled `app: frontend`.
- `backend` namespace contains server Pods labeled `app: payment`.
- Only frontend Pods should connect to payment Pods on TCP port 8080.

Label namespaces:

```
kubectl label namespace frontend \
  kubernetes.io/metadata.name=frontend --overwrite
```

The standard namespace-name label is normally managed by Kubernetes, so explicitly changing it is unnecessary. You can use that existing label in NetworkPolicy selectors.

Example policy in the backend namespace:

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: payment
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: frontend
          podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```

This allows matching frontend Pods into payment Pods on port 8080. For an allowlist-only result, ensure payment Pods have no other ingress policies granting unwanted access and that frontend egress policies also permit the flow.

Interview point: NetworkPolicies filter network traffic. They do not authenticate the identity of the calling application.

## Section 4: Kubernetes NetworkPolicies and Zero-Trust Security

### 39. How do you restrict communication between Pods using NetworkPolicies?

Answer:

Use Kubernetes NetworkPolicy resources to control permitted ingress and egress traffic for selected Pods.

A common approach is to deny all traffic by default and explicitly allow required communication.

Example default-deny policy:

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

This selects all Pods in the production namespace and isolates their ingress and egress traffic.

Then create targeted allow rules for required application traffic, DNS, monitoring, and external dependencies.

Important: Policies are additive. Multiple policies can allow traffic; a policy cannot override another policy's allowed traffic with an explicit deny rule.

Interview point: NetworkPolicies only work when the cluster's networking solution supports and enforces them.

### 40. How do you check whether NetworkPolicy enforcement is enabled?

Answer:

Kubernetes does not provide a single universal command proving that NetworkPolicies are enforced.

Check the installed CNI:

```
kubectl get daemonsets -n kube-system
```

Check existing policies:

```
kubectl get networkpolicy -A
```

Inspect the cluster's network implementation.

For EKS, check the AWS VPC CNI configuration and whether its network policy functionality is enabled. For Calico or Cilium, inspect their relevant components and configuration.

Best verification method:

1. Deploy two test Pods.
2. Confirm traffic between them initially succeeds.
3. Apply a deny policy selecting the destination.
4. Repeat the connectivity test.
5. Confirm the traffic is blocked.
6. Apply an explicit allow rule.
7. Confirm the authorized traffic succeeds again.

Interview point: Creating a NetworkPolicy resource does not prove enforcement. Test the actual network behavior.

### 41. What components do you configure in a NetworkPolicy YAML file?

Answer:

The main fields are:

| Field                | Purpose                             |
| -------------------- | ----------------------------------- |
| `apiVersion`         | `networking.k8s.io/v1`              |
| `kind`               | `NetworkPolicy`                     |
| `metadata.name`      | Policy name                         |
| `metadata.namespace` | Namespace containing selected Pods  |
| `spec.podSelector`   | Selects Pods affected by the policy |
| `spec.policyTypes`   | Ingress, Egress, or both            |
| `spec.ingress`       | Allowed incoming traffic            |
| `spec.egress`        | Allowed outgoing traffic            |
| `namespaceSelector`  | Selects namespaces                  |
| `ipBlock`            | Selects IP CIDR ranges              |
| `ports`              | Restricts protocol and port         |

Example allowing only matching application traffic:

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```

This example allows `frontend` Pods in the same namespace to reach `backend` Pods on port 8080.

### 42. How do you implement zero-trust networking inside Kubernetes without a service mesh?

Answer:

Zero-trust networking means not automatically trusting traffic simply because it originates inside the cluster.

I would implement it using several layers:

1. Apply default-deny ingress and egress NetworkPolicies.
2. Allow only the required service-to-service communication.
3. Isolate workloads using namespaces and labels.
4. Use dedicated ServiceAccounts with least-privilege RBAC.
5. Encrypt sensitive application traffic using application-level TLS or mTLS.
6. Use certificate automation for identity and rotation.
7. Restrict access to external APIs and databases.
8. Implement workload security controls and monitoring.
9. Regularly test network reachability and authorization.

Interview point: NetworkPolicies alone do not deliver complete zero trust. They generally provide Layer 3/4 filtering, while workload authentication, encryption, and application authorization require additional controls.

### 43. How do you implement and manage NetworkPolicies for strict inter-service communication?

Answer:

For a production microservices environment, I would define a service communication matrix.

Example:

| Source            | Destination               | Port | Allow? |
| ----------------- | ------------------------- | ---- | ------ |
| Frontend          | API                       | 8080 | Yes    |
| API               | Database                  | 5432 | Yes    |
| Frontend          | Database                  | 5432 | No     |
| API               | External payment provider | 443  | Yes    |
| Unknown workloads | Database                  | 5432 | No     |

Implementation process:

- Create default-deny policies per namespace.
- Add explicit ingress and egress allow policies.
- Use narrowly scoped Pod and namespace selectors.
- Allow DNS only where needed.
- Manage policy YAML through GitOps.
- Validate policies using automated connectivity tests.
- Audit changes and monitor rejected connections.

Interview point: Both the source's egress policy and destination's ingress policy must permit a connection when both directions are isolated.

## Section 5: Service Mesh, Istio, and Production Security

### 44. What is a service mesh?

Answer:

A service mesh is infrastructure that manages communication between services, especially in distributed microservices environments.

Common capabilities include:

- Mutual TLS (mTLS) encryption.
- Service identity and authorization.
- Traffic management.
- Retries and timeouts.
- Traffic observability.
- Canary and progressive deployments.
- Service-to-service access controls.

Popular service mesh technologies include Istio, Linkerd, and Consul-based service mesh solutions.

Interview point: A service mesh primarily manages service-to-service communication. It does not replace Kubernetes Services, storage, RBAC, or application security.

### 45. How do you design an Istio service mesh for Kubernetes?

Answer:

For a production EKS cluster, I would design Istio based on workload requirements, operational complexity, and whether I need Layer 4 or Layer 7 functionality.

Two common deployment models are:

| Sidecar mode                               | Ambient mode                                    |
| ------------------------------------------ | ----------------------------------------------- |
| Envoy proxy runs alongside application Pod | Node-level ztunnel provides secure transport    |
| Per-Pod proxy resource overhead            | No sidecar required for basic L4 mesh           |
| Supports L7 features through sidecars      | Optional waypoint proxies provide L7 features   |
| Requires sidecar lifecycle management      | Applications can join without sidecar injection |

Conceptual Istio Architecture

Istio Control Plane — istiod

Service discovery, configuration, identity and certificates

Sidecar data plane

App + Envoy

App + Envoy

Per-Pod proxies

Ambient data plane

App Pods

ztunnel + Waypoint\*

Node L4 and optional L7 proxy

\*Waypoint proxies are optional and can be shared by workloads; they are not necessarily deployed per node.

Implementation approach:

1. Define which namespaces and services require mesh capabilities.
2. Install Istio using a supported installation method.
3. Choose sidecar or ambient data plane mode.
4. Enroll selected workloads.
5. Configure ingress and egress gateways where required.
6. Enable mTLS and workload identity-based authorization.
7. Configure traffic routing, resilience, and telemetry.
8. Integrate monitoring and tracing.
9. Validate failures, upgrades, resource usage, and security policies.

Ambient enrollment example, after installing ambient mode:

```
kubectl label namespace production \
  istio.io/dataplane-mode=ambient
```

Interview point: Ambient mesh can supply L4 security without sidecars, with optional waypoint proxies for L7 functionality.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://istio.io\&sz=32)

Istio

+1



### 46. What security policies would you implement in a service mesh?

Answer:

I would implement several security layers.

1\. Mutual TLS

Require encrypted, authenticated service-to-service connections.

Example Istio policy:

```
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT
```

2\. Workload authorization

Restrict which services are allowed to communicate using verified workload identities.

3\. Request authentication

Validate JWTs where appropriate for application-facing APIs.

4\. Egress controls

Restrict and observe calls to external services.

5\. Ingress security

Configure HTTPS, gateway authorization, certificate management, and appropriate rate limiting.

6\. Audit and observability

Collect metrics, logs, and traces for authorized and denied traffic.

Interview point: Istio authorization should use verified workload identities and narrowly scoped policies. In ambient mode, L7 authorization generally requires waypoint proxies.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://istio.io\&sz=32)

Istio

+1



### 47. Is a service mesh always necessary, or can alternative tools provide similar capabilities?

Answer:

No. A service mesh is not always necessary.

For smaller or simpler applications, adding a service mesh may introduce unnecessary resource overhead and operational complexity.

| Requirement                | Possible alternative                        |
| -------------------------- | ------------------------------------------- |
| Restrict Pod communication | NetworkPolicies                             |
| TLS between applications   | Application TLS with certificate automation |
| HTTP ingress routing       | Ingress or Gateway API                      |
| Traffic routing            | Gateway controller or application proxy     |
| Observability              | OpenTelemetry and Prometheus                |
| Retries and timeouts       | Application libraries or gateway            |
| Rate limiting              | API gateway, reverse proxy, or application  |
| External access control    | Firewall, security groups, or egress proxy  |

Interview point: Choose a service mesh when centralized service identity, mTLS, policy enforcement, and advanced traffic management provide enough value to justify the operational cost.

### 48. What is the difference between service mesh control planes and data planes?

Answer:

| Control Plane                                        | Data Plane                               |
| ---------------------------------------------------- | ---------------------------------------- |
| Distributes configuration                            | Processes application traffic            |
| Handles service and workload discovery               | Routes or forwards traffic               |
| Manages identity/certificate-related configuration   | Enforces mTLS and traffic policy         |
| Does not normally handle application request traffic | Lies in the application traffic path     |
| Istio example: `istiod`                              | Istio example: Envoy, ztunnel, waypoints |

Interview point: If the control plane becomes temporarily unavailable, existing proxies may continue handling traffic using their current configuration, but configuration changes, certificate operations, and new workloads may be affected.

### 49. What are alternative Ingress Controllers available for Kubernetes?

Answer:

Common Kubernetes ingress implementations include:

| Controller                            | Typical use                       |
| ------------------------------------- | --------------------------------- |
| AWS Load Balancer Controller          | AWS ALB ingress for EKS           |
| F5 NGINX Ingress Controller           | NGINX-based ingress and routing   |
| Traefik                               | Dynamic reverse proxy and ingress |
| HAProxy Kubernetes Ingress Controller | HAProxy-based ingress             |
| Contour                               | Envoy-based Kubernetes ingress    |
| Kong Ingress Controller               | API gateway and ingress           |
| Istio ingress gateway                 | Istio-based north-south traffic   |

The historical community ingress-nginx project should be distinguished from the separately maintained F5 NGINX Ingress Controller. Check current support and maintenance status before selecting a controller for a new production deployment.

Interview point: Select an ingress implementation based on cloud integration, routing capabilities, operational support, security features, and Gateway API compatibility.

### 50. How do you restrict public access to Kubernetes LoadBalancers?

Answer:

In AWS EKS, the strongest common approach is to provision an internal load balancer when public access is not required.

Other measures include restricting inbound source CIDRs, security group rules, private DNS, and authentication.

Option 1: Internal AWS ALB

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: internal-api
  annotations:
    alb.ingress.kubernetes.io/scheme: internal
spec:
  ingressClassName: alb
  rules:
    - host: api.internal.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
```

Option 2: Internal LoadBalancer Service

Using a supported AWS NLB controller configuration:

```
apiVersion: v1
kind: Service
metadata:
  name: internal-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-scheme: internal
spec:
  type: LoadBalancer
  selector:
    app: internal-api
  ports:
    - port: 80
      targetPort: 8080
```

The annotation is AWS-controller-specific, so its interpretation depends on the configured load-balancer integration.

Option 3: Restrict allowed source CIDRs

For an appropriately supported LoadBalancer Service:

```
spec:
  type: LoadBalancer
  loadBalancerSourceRanges:
    - 203.0.113.0/24
```

Replace the example CIDR with the authorized client network. Verify the cloud controller enforces the setting.

Additional production controls include:

- Restricting security groups.
- Using private subnets and private network connectivity.
- Authenticating users through an application or identity-aware gateway.
- Using TLS.
- Applying AWS WAF where compatible.
- Auditing load-balancer listeners and target security.

Interview point: An internal AWS load balancer is not publicly internet-facing, but it can still be reachable from permitted connected private networks. Security groups and application authentication remain important.

## Final Interview Revision: 10 Critical Concepts

| Topic                         | Key answer to remember                                                    |
| ----------------------------- | ------------------------------------------------------------------------- |
| StatefulSet vs Deployment     | Stable identities and individual PVCs vs interchangeable replicas         |
| EBS vs EFS                    | AZ-bound block storage vs shared network filesystem                       |
| PVC and multiple Pods         | Sharing depends on access mode, node placement, and storage driver        |
| `kubectl apply`               | Sends desired configuration through API server; controllers reconcile     |
| Ingress vs Controller         | Defines routing vs implements routing                                     |
| ALB IP target mode            | Load balancer can send traffic directly to Pod IPs                        |
| Cross-namespace communication | Possible by default unless network controls restrict it                   |
| NetworkPolicy                 | Enforces traffic restrictions through a compatible network implementation |
| Zero trust                    | Requires least privilege, network isolation, identity, and encryption     |
| Istio                         | Manages service identity, security, traffic, and observability            |

Important current update: The community `ingress-nginx` project was officially retired on March 24, 2026. It no longer receives security fixes. For new production deployments, consider a maintained Ingress Controller or Gateway API implementation. This retirement does not apply to all NGINX-based ingress products.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://kubernetes.io\&sz=32)

Kubernetes

+1



## Production Troubleshooting Commands

```
# Check cluster and current context
kubectl config current-context
kubectl get nodes -o wide

# Check Pod scheduling
kubectl get pods -A -o wide
kubectl describe pod <pod-name>

# Check deployment rollout
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>

# Inspect Services and EndpointSlices
kubectl get svc -A
kubectl get endpointslices -A

# Check Ingress and Gateway resources
kubectl get ingress -A
kubectl get gateway -A
kubectl get httproute -A

# Check persistent storage
kubectl get pv
kubectl get pvc -A
kubectl get storageclass

# Check policies
kubectl get networkpolicy -A

# Check Kubernetes events
kubectl get events -A --sort-by=.lastTimestamp

# Test Service access from an application Pod
kubectl exec <pod-name> -- \
  curl -v http://backend-service:8080

# Check configured CSI drivers
kubectl get csidrivers
```

## Interview Practice: Scenario-Based Questions

Advanced Kubernetes Quiz

1 / 5

You scale an EBS-backed Deployment to three Pods on different nodes. Two fail with Multi-Attach errors. What is the likely issue?

CoreDNS is unavailable

The Pods share a standard EBS RWO volume across nodes

The Deployment selector is missing

The API server cannot schedule Deployments

PreviousNext&#x20;

For an experienced DevOps or EKS interview, prioritize being able to explain questions 1–5, 17, 23–24, 29–30, 38–43, and 45–50 with practical examples. These test how you reason about real production environments rather than simply recall definitions.
