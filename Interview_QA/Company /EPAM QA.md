Below are detailed answers for the EPAM interview set. I’ve numbered the questions consecutively. For the Linux question, I’m assuming **“vat” means `/var`**.

**1. How do you create and use custom resources in Kubernetes?**

A **custom resource** extends the Kubernetes API with an application-specific object. For example, you could introduce `WebApp`, `Database`, or `Backup` alongside built-in objects such as `Deployment` and `Service`.

There are three related concepts:

| Component                        | Purpose                                                                             |
| -------------------------------- | ----------------------------------------------------------------------------------- |
| CustomResourceDefinition, or CRD | Defines a new resource type, its scope, versions, and schema                        |
| Custom resource, or CR           | An instance of that resource type                                                   |
| Custom controller                | Watches resources and reconciles the actual system with their desired configuration |

**Step 1: Define the resource type.**

Example `webapp-crd.yaml`:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: webapps.platform.example.com
spec:
  group: platform.example.com
  scope: Namespaced
  names:
    plural: webapps
    singular: webapp
    kind: WebApp
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          required:
            - spec
          properties:
            spec:
              type: object
              required:
                - image
              properties:
                image:
                  type: string
                replicas:
                  type: integer
                  minimum: 1
                  default: 2
```

Register it:

```bash
kubectl apply -f webapp-crd.yaml

kubectl wait \
  --for=condition=Established \
  crd/webapps.platform.example.com \
  --timeout=60s
```

**Step 2: Create an instance.**

Example `webapp.yaml`:

```yaml
apiVersion: platform.example.com/v1
kind: WebApp
metadata:
  name: customer-portal
  namespace: default
spec:
  image: nginx:stable
  replicas: 3
```

```bash
kubectl apply -f webapp.yaml
kubectl get webapps
kubectl describe webapp customer-portal
```

Kubernetes now validates and stores these objects using the CRD’s schema. [Kubernetes CRD guide](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)

**Step 3: Implement the behavior.**

A controller could watch `WebApp` objects and create or update the associated Deployment and Service.

**Creating a CRD alone does not deploy the application.** The controller supplies the operational behavior. An operator builds on this pattern to encode application-specific knowledge, such as database backup, recovery, or upgrades. [Kubernetes custom resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)

**2. What are namespaces in Kubernetes?**

A namespace provides a scope for naming and organizing resources within a cluster.

For example, teams might use:

* `development`
* `testing`
* `production`
* `monitoring`

You can have a Service named `api` in both `development` and `production` because each belongs to a different namespace.

```bash
kubectl create namespace development

kubectl get pods -n development

kubectl config set-context \
  --current \
  --namespace=development
```

Namespaces help apply:

* **RBAC:** control who can access resources.
* **ResourceQuota:** limit aggregate resource consumption.
* **LimitRange:** set default or permitted resource requests and limits.
* **NetworkPolicy:** control permitted network communication, with a supporting network implementation.

Namespaces do not automatically isolate network traffic or provide complete tenant security.

Some resources are namespaced, including Pods, Services, Deployments, Secrets, and PVCs. Others are cluster-scoped, including Nodes, PersistentVolumes, StorageClasses, and CRDs. [Kubernetes namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)

**3. What is the difference between a Deployment and a StatefulSet?**

Both manage replicated pods, but StatefulSets preserve identity across pod replacement.

| Aspect       | Deployment                                           | StatefulSet                                                             |
| ------------ | ---------------------------------------------------- | ----------------------------------------------------------------------- |
| Typical use  | Interchangeable application replicas                 | Workloads requiring stable identity or storage                          |
| Pod identity | Replacement pods generally receive new names         | Stable ordinal names, such as `db-0`, `db-1`                            |
| Storage      | Can use PVCs; storage relationships must be designed | Can create a separate PVC for each replica using `volumeClaimTemplates` |
| Ordering     | Does not provide StatefulSet-style ordered startup   | Ordered creation and termination by default                             |
| Networking   | Usually accessed through a common Service            | Supports stable per-pod DNS identity, commonly with a headless Service  |
| Example      | REST API or frontend                                 | Database or clustered messaging system                                  |

A StatefulSet’s stable identity does **not** mean that a pod keeps the same IP address.

Also:

* Deployments can use persistent storage.
* StatefulSets do not automatically implement database replication, backups, or leader election.
* StatefulSet PVC retention depends on the configured retention policy; the default retains them.

Choose based on identity, ordering, and storage requirements. [Kubernetes StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)

**4. What is role-based access control in Kubernetes?**

**RBAC—Role-Based Access Control—determines which actions an authenticated identity can perform on Kubernetes resources.**

A permission might allow an identity to list pods while preventing it from deleting them.

| Object             | Purpose                                                                          |
| ------------------ | -------------------------------------------------------------------------------- |
| Role               | Defines permissions within a namespace                                           |
| ClusterRole        | Defines reusable permissions, including permissions for cluster-scoped resources |
| RoleBinding        | Grants a Role or ClusterRole’s applicable permissions within one namespace       |
| ClusterRoleBinding | Grants a ClusterRole’s permissions across the cluster                            |

Example: grant a service account permission to read pods in `development`.

```bash
kubectl create serviceaccount reader -n development

kubectl create role pod-reader \
  -n development \
  --verb=get,list,watch \
  --resource=pods

kubectl create rolebinding pod-reader-binding \
  -n development \
  --role=pod-reader \
  --serviceaccount=development:reader
```

Verify using an identity allowed to impersonate that service account:

```bash
kubectl auth can-i list pods \
  -n development \
  --as=system:serviceaccount:development:reader
```

Permissions are additive: Kubernetes RBAC has no explicit deny rule. Apply least privilege and distinguish authentication—identifying the caller—from authorization—deciding what that caller may do. [Kubernetes RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

**5. What are Cluster Autoscaler and Horizontal Pod Autoscaler?**

They scale different parts of the system.

| Aspect           | Horizontal Pod Autoscaler—HPA             | Cluster Autoscaler—CA                                                            |
| ---------------- | ----------------------------------------- | -------------------------------------------------------------------------------- |
| Scales           | Workload replica count                    | Nodes in configured node groups                                                  |
| Main input       | CPU, memory, custom, or external metrics  | Pods that cannot be scheduled and opportunities to remove unnecessary nodes      |
| Example          | Increase API replicas from 3 to 8         | Increase a worker node group from 3 to 5 nodes                                   |
| Main constraints | Replica bounds, metrics, scaling behavior | Node-group bounds, scheduling constraints, capacity, and disruption restrictions |

Example HPA:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payments
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payments
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
```

Here, CPU utilization is measured relative to the containers’ **CPU requests**. If a container requests `500m`, then `60%` corresponds to `300m`. Resource-based HPA needs a working resource metrics API, typically provided by Metrics Server. [Kubernetes HPA](https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/)

The two can work together:

1. Traffic increases, and HPA increases the desired replicas.
2. Some new pods cannot fit on existing nodes.
3. Cluster Autoscaler identifies a suitable node group and increases its size.
4. Once nodes become Ready, the scheduler places the pending pods.

Cluster Autoscaler primarily reasons about scheduling requirements and resource requests, rather than simply scaling whenever node CPU is high. It cannot resolve every Pending condition—for example, adding nodes may not fix an incompatible affinity rule. [Kubernetes node autoscaling](https://kubernetes.io/docs/concepts/cluster-administration/node-autoscaling/)

**6. What is a Terraform provider?**

A provider is a plugin that enables Terraform to interact with an external API.

Examples include:

* AWS
* Azure
* Google Cloud
* Kubernetes
* GitHub

Providers implement resource types and data sources. The AWS provider, for example, supplies `aws_instance`, `aws_vpc`, and AWS lookup data sources.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = "ap-south-1"
}
```

The distinction is:

* `required_providers` declares the provider source and permitted versions.
* `provider` configures a provider instance, including its Region and authentication settings.

`terraform init` installs the required providers. Provider aliases allow multiple configurations, such as different AWS Regions or accounts. Use IAM roles or the standard credential chain rather than embedding access keys in configuration. [Terraform providers](https://developer.hashicorp.com/terraform/language/providers)

**7. How do you manage state in Terraform?**

Terraform state maps configuration addresses to actual infrastructure objects.

For example:

```text
aws_instance.application → i-0123456789abcdef0
```

State also records resource attributes and metadata needed for planning and managing dependencies.

For team environments, I would:

1. Use a protected remote backend.
2. Enable state locking to coordinate concurrent operations.
3. Enable versioning or equivalent recovery capabilities.
4. Restrict access because state can contain sensitive values.
5. Separate state according to environment, ownership, and lifecycle.
6. Use Terraform’s supported commands and migration mechanisms for changes.

Useful inspection commands:

```bash
terraform state list
terraform state show aws_instance.application
```

When reorganizing resource addresses, use `moved` blocks where appropriate. For existing unmanaged resources, use an import workflow.

Two interview distinctions matter:

* Marking a value `sensitive` hides it in some output; it does not encrypt the state.
* `.terraform.lock.hcl` records provider dependency selections. It is different from a state lock.

Terraform also reads remote infrastructure during normal planning; state is not a continuously updated inventory. [Terraform state](https://developer.hashicorp.com/terraform/language/state)

**8. Do you store state locally or remotely? Which block configures remote state?**

For a shared production environment, I would use remote state. Local state can be adequate for an isolated learning exercise.

An S3 backend example:

```hcl
terraform {
  required_version = ">= 1.10, < 2.0"

  backend "s3" {
    bucket       = "example-company-terraform-state"
    key          = "production/network/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

The `backend "s3"` block configures state storage.

The bucket must already exist when Terraform initializes the backend. Enable bucket versioning and configure appropriate access permissions.

```bash
terraform init
```

To migrate existing local state after adding the backend:

```bash
terraform init -migrate-state
```

Current Terraform supports **native S3 state locking** through `use_lockfile = true`. DynamoDB-based locking is deprecated. The execution identity needs access to both the state object and its lock object. [Terraform S3 backend](https://developer.hashicorp.com/terraform/language/backend/s3)

Keep backend credentials out of committed configuration. Backend settings also cannot directly reference ordinary Terraform input variables; environment-specific backend configuration can be supplied during initialization.

**9. What is a Terraform module?**

A module is a collection of Terraform configuration files managed together.

* The configuration you run directly is the **root module**.
* A module called from another module is a **child module**.

For example, a VPC module could manage the VPC, subnets, route tables, gateways, and related outputs.

A caller might use:

```hcl
module "network" {
  source = "./modules/vpc"

  cidr_block  = "10.40.0.0/16"
  environment = "development"
}

output "vpc_id" {
  value = module.network.vpc_id
}
```

This assumes the child module defines those inputs and exposes `vpc_id`.

Typical module files include:

* `main.tf`: resources and module calls.
* `variables.tf`: input definitions and validation.
* `outputs.tf`: values exposed to callers.
* `versions.tf`: Terraform and provider requirements.
* `README.md`: usage and assumptions.

Modules help reuse tested infrastructure patterns and keep environment-specific differences in inputs.

**A child module does not automatically get its own state file.** Its resources normally belong to the calling root module’s state. [Terraform modules](https://developer.hashicorp.com/terraform/language/modules)

**10. How do you manage multiple environments in Terraform?**

I would reuse modules while separating environment configuration, state, and access.

For example:

| Environment | Root configuration   | State key                         |
| ----------- | -------------------- | --------------------------------- |
| Development | `environments/dev/`  | `dev/platform/terraform.tfstate`  |
| QA          | `environments/qa/`   | `qa/platform/terraform.tfstate`   |
| Production  | `environments/prod/` | `prod/platform/terraform.tfstate` |

Each environment can have its own:

* AWS account and execution role.
* Backend configuration.
* Input values.
* Capacity settings.
* Change-review and deployment requirements.

Example:

```bash
terraform -chdir=environments/prod init

terraform -chdir=environments/prod plan \
  -var-file=prod.tfvars
```

The pipeline must select the account, backend, and variables consistently.

CLI workspaces provide separate state instances within the same working directory and backend. They can be useful for temporary copies of infrastructure, but they do not independently provide strong credential or access isolation.

HCP Terraform workspaces are different: each has its own configuration, variables, state, and run settings. [Terraform workspace guidance](https://developer.hashicorp.com/terraform/cli/workspaces)

**11. What are the use cases of Amazon CloudWatch?**

CloudWatch supports operational monitoring and investigation.

| Capability         | Example use                                                      |
| ------------------ | ---------------------------------------------------------------- |
| Metrics            | Observe EC2 CPU, ALB errors, or Lambda duration                  |
| Logs               | Centralize application and infrastructure logs                   |
| Logs Insights      | Search logs and summarize error patterns                         |
| Alarms             | Notify or trigger supported actions when thresholds are breached |
| Dashboards         | Display application and infrastructure health together           |
| Container Insights | Monitor container workloads                                      |
| Synthetics         | Exercise application endpoints or user journeys                  |

For example, an application could have alarms for elevated server errors, excessive response time, and insufficient healthy capacity. Logs then help identify the failing requests or dependencies. [Amazon CloudWatch overview](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html)

A common interview detail: **EC2 guest memory utilization and filesystem-used percentage are not standard EC2 metrics collected automatically.** Publish them using the CloudWatch agent or another suitable integration. [CloudWatch agent metrics](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/metrics-collected-by-CloudWatch-agent.html)

**12. What are ECS and EKS?**

Both orchestrate containers, but they expose different operational models.

| Aspect                     | Amazon ECS                                       | Amazon EKS                                                         |
| -------------------------- | ------------------------------------------------ | ------------------------------------------------------------------ |
| Orchestration platform     | AWS-native container orchestration               | Managed Kubernetes                                                 |
| Workload definition        | Task definitions, tasks, and services            | Pods, Deployments, StatefulSets, and other Kubernetes objects      |
| Main interfaces            | AWS APIs, CLI, console, and infrastructure tools | Kubernetes API, `kubectl`, Helm, and infrastructure tools          |
| Common capacity choices    | EC2 and Fargate                                  | EC2-based nodes and Fargate                                        |
| Operational considerations | ECS-specific configuration and integrations      | Kubernetes APIs, controllers, add-ons, and ecosystem compatibility |

In ECS:

* A **task definition** describes containers, resources, networking, and related settings.
* A **task** is a running instance of that definition.
* A **service** maintains the desired tasks and supports deployment and load-balancing behavior. [Amazon ECS overview](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html)

In EKS, AWS operates the managed Kubernetes control plane. You remain responsible for workload configuration and the parts of networking, compute, access, and add-on management associated with your chosen operating model. [Amazon EKS overview](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html)

**13. What is AWS Fargate?**

Fargate provides managed compute capacity for containers used with ECS or EKS.

You specify the workload’s container images and supported CPU, memory, networking, and access configuration. AWS supplies and operates the underlying compute infrastructure.

With:

* **ECS:** Fargate runs tasks.
* **EKS:** Fargate runs eligible pods.

It reduces work such as provisioning container hosts and maintaining their operating systems.

You still manage the application image, application security, IAM permissions, network access, observability, and scaling configuration.

Fargate has platform-specific constraints around host access and workload capabilities. For example, workloads that require privileged host-level operations need careful evaluation. It also should not be assumed to be the cheapest option for every workload; compare actual resource utilization and operating requirements. [AWS Fargate for ECS](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/AWS_Fargate.html)

**14. What are the limitations of AWS Lambda?**

For standard Lambda function invocations, important current limits include:

| Item                            | Limit or behavior                                                       |
| ------------------------------- | ----------------------------------------------------------------------- |
| Execution timeout               | Up to 900 seconds—15 minutes                                            |
| Memory                          | 128 MB to 10,240 MB                                                     |
| Writable temporary storage      | `/tmp`, configurable from 512 MB to 10,240 MB                           |
| Container image size            | Up to 10 GB uncompressed, including layers                              |
| ZIP deployment contents         | Up to 250 MB uncompressed, including layers                             |
| Direct ZIP upload               | Up to 50 MB zipped through the Lambda API/SDK or console                |
| Buffered synchronous payload    | Up to 6 MB for request and response                                     |
| Streamed synchronous response   | Up to 200 MB where supported                                            |
| Asynchronous invocation payload | Up to 1 MB                                                              |
| Regional concurrent executions  | Common default of 1,000; adjustable, with lower initial quotas possible |

Other design considerations include cold starts, dependency connection limits, invocation retries, and the absence of guaranteed durable local state.

Use external storage for persistent data and make event processing tolerate retries where applicable. Long-running workflows can coordinate multiple bounded executions. Check the account’s actual quotas and any limits imposed by invoking services. [AWS Lambda quotas](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html)

**15. How does Lambda work with containers?**

A Lambda function can be packaged as a container image. The image must support the **Lambda Runtime API**, which allows the runtime to receive invocation events and return results.

AWS-provided Lambda base images include the runtime integration.

Example `app.py`:

```python
def handler(event, context):
    return {
        "statusCode": 200,
        "body": "Hello from a Lambda container image"
    }
```

Example Dockerfile:

```dockerfile
FROM public.ecr.aws/lambda/python:3.13

COPY app.py ${LAMBDA_TASK_ROOT}/app.py

CMD ["app.handler"]
```

The deployment process is:

1. Build the image for the function’s architecture.
2. Push it to Amazon ECR in the same Region as the function.
3. Create a Lambda function using the image and an execution role.
4. Configure memory, timeout, networking, and event triggers.
5. Invoke it through a supported event source or API.

Lambda initializes an execution environment and calls the handler for an invocation. Environments may be reused, so code must not depend on a fresh environment every time.

Images must be Linux-based and target one architecture. The filesystem must work as read-only apart from supported writable storage such as `/tmp`. Packaging a standard Lambda function as an image does not remove its invocation limits. [Lambda container images](https://docs.aws.amazon.com/lambda/latest/dg/images-create.html)

**16. What are EC2 instances?**

An EC2 instance is a virtual server in AWS.

When launching one, you choose or configure:

* **AMI:** operating system and initial software.
* **Instance type:** CPU, memory, networking, and other capabilities.
* **Storage:** commonly EBS volumes.
* **Networking:** VPC, subnet, and addresses.
* **Security groups:** permitted network traffic.
* **IAM role:** application access to AWS APIs.
* **Startup configuration:** such as user data.

Common instance-family purposes include:

| Family category       | Typical workload                           |
| --------------------- | ------------------------------------------ |
| General purpose       | Application servers and balanced workloads |
| Compute optimized     | CPU-intensive processing                   |
| Memory optimized      | Memory-intensive databases and analytics   |
| Storage optimized     | High local-storage throughput or IOPS      |
| Accelerated computing | GPU or other accelerator workloads         |

EC2 provides operating-system control, which also makes the customer responsible for guest OS maintenance, application configuration, and associated access management. AWS manages the underlying infrastructure. [Amazon EC2 overview](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/concepts.html)

**17. What is AWS Direct Connect?**

Direct Connect establishes dedicated network connectivity between an organization’s network and AWS through a Direct Connect location.

Typical uses include:

* Hybrid applications communicating with on-premises systems.
* Large or sustained data transfers.
* Workloads requiring more consistent network behavior than an internet-based path.

Connectivity uses virtual interfaces and BGP routing:

| Virtual interface | Purpose                                                  |
| ----------------- | -------------------------------------------------------- |
| Private VIF       | Access VPC resources using private addresses             |
| Public VIF        | Access AWS public services using public addresses        |
| Transit VIF       | Access Transit Gateways through a Direct Connect gateway |

[Direct Connect virtual interfaces](https://docs.aws.amazon.com/directconnect/latest/UserGuide/WorkingWithVirtualInterfaces.html)

**Dedicated connectivity does not automatically mean encrypted connectivity.** Direct Connect does not encrypt traffic by default. Use appropriate encryption, such as an IPsec VPN over the connection, when required. [Direct Connect encryption](https://docs.aws.amazon.com/directconnect/latest/UserGuide/encryption-in-transit.html)

For production reliability, design redundant paths and test failover instead of depending on one physical connection.

**18. What is AWS Storage Gateway?**

Storage Gateway connects existing application storage interfaces with AWS storage.

A gateway deployed near the applications exposes familiar protocols while transferring data to AWS and providing local caching.

Common patterns include:

| Gateway         | Application interface      | Typical use                                                  |
| --------------- | -------------------------- | ------------------------------------------------------------ |
| S3 File Gateway | NFS or SMB                 | Store files as S3 objects while applications use file shares |
| Volume Gateway  | iSCSI block storage        | Cloud-backed volumes and snapshot-based recovery             |
| Tape Gateway    | iSCSI virtual tape library | Move compatible tape backup workflows to AWS                 |

These allow applications to adopt cloud storage without necessarily being rewritten to call S3 APIs. [Storage Gateway features](https://aws.amazon.com/storagegateway/features/)

Volume Gateway has two important configurations:

* **Cached volumes:** primary data resides in AWS, with frequently accessed data cached locally.
* **Stored volumes:** primary data remains locally available, with backups stored in AWS.

The choice depends on local storage needs, access patterns, bandwidth, and recovery requirements. [Volume Gateway overview](https://docs.aws.amazon.com/storagegateway/latest/vgw/WhatIsStorageGateway.html)

**19. Explain VPC, NAT Gateway, S3, Route 53, VPC peering, Transit Gateway, and Auto Scaling groups.**

| Service                                                                                              | Purpose                                                                                                                 | Example                                                   |
| ---------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------- |
| VPC                                                                                                  | A logically isolated network with subnets, routing, and network controls                                                | Host application and database tiers                       |
| [NAT Gateway](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html)                 | Managed address translation; a public NAT gateway commonly provides outbound internet access for private IPv4 workloads | Private instances download software updates               |
| [S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html)                             | Object storage organized into buckets and objects                                                                       | Backups, artifacts, logs, and static content              |
| [Route 53](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)                   | DNS, domain registration, and health-check-related capabilities                                                         | Resolve `app.example.com` to an application endpoint      |
| [VPC peering](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-peering.html)                     | Private connectivity between two VPCs                                                                                   | An application accesses a service in another VPC          |
| [Transit Gateway](https://docs.aws.amazon.com/vpc/latest/tgw/what-is-transit-gateway.html)           | A routing hub connecting multiple VPCs and on-premises networks                                                         | Connect many accounts’ VPCs through managed routing       |
| [Auto Scaling group](https://docs.aws.amazon.com/autoscaling/ec2/userguide/auto-scaling-groups.html) | Maintains and adjusts a fleet of EC2 instances                                                                          | Keep a minimum application capacity and scale with demand |

Important distinctions:

* A public NAT gateway supports connections initiated from the private side; it does not provide unsolicited inbound access to private instances.
* VPC peering requires appropriate routes and security rules. It is non-transitive: peering A–B and B–C does not automatically connect A–C.
* Transit Gateway supports hub-based routing, but attachments, associations, propagation, and VPC routes must still be configured.
* An Auto Scaling group has minimum, desired, and maximum capacity. Scaling policies change desired capacity within applicable bounds.

In a typical web application, Route 53 resolves the application name to a load balancer, which distributes requests to EC2 instances maintained by an Auto Scaling group.

**20. What is the difference between a security group and a network ACL?**

| Aspect              | Security group                                          | Network ACL                                |
| ------------------- | ------------------------------------------------------- | ------------------------------------------ |
| Scope               | Associated resources/network interfaces                 | Subnet boundary                            |
| Connection tracking | Stateful                                                | Stateless                                  |
| Rules               | Allow rules                                             | Allow and deny rules                       |
| Evaluation          | Applicable allow rules are combined                     | Lowest numbered matching rule takes effect |
| Return traffic      | Automatically permitted for tracked allowed connections | Must be permitted by rules                 |
| Associations        | An interface can have multiple security groups          | A subnet has one associated NACL           |
| Common role         | Workload-specific access control                        | Additional subnet-level filtering          |

A newly created security group normally has no inbound allowances and allows outbound traffic. A custom NACL initially denies traffic; the default VPC NACL initially allows traffic. [Security groups](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html), [network ACLs](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html)

For example, when a client connects to an application on TCP 443:

* The security group permits the inbound connection and tracks its return traffic.
* The NACL must permit traffic in both directions, including the relevant client-side ephemeral port range.

Security groups and NACLs must both permit the traffic along its path.

**21. What is the difference between Docker `COPY` and `ADD`?**

Both add content to an image during its build.

| Feature                                             | `COPY`               | `ADD`                  |
| --------------------------------------------------- | -------------------- | ---------------------- |
| Copy files from build context                       | Yes                  | Yes                    |
| Copy from another build stage                       | Yes, with `--from`   | No equivalent `--from` |
| Fetch supported remote URLs or Git sources          | No direct equivalent | Yes                    |
| Automatically extract recognized local tar archives | No                   | Yes                    |

Examples:

```dockerfile
COPY application.jar /app/application.jar
```

```dockerfile
ADD application.tar.gz /opt/application/
```

In the second example, a recognized local tar archive is extracted.

For a multi-stage build:

```dockerfile
COPY --from=builder /workspace/app /usr/local/bin/app
```

Use `COPY` for straightforward copying. Use `ADD` when its downloading or archive-handling features are required, and validate remote artifacts appropriately. [Docker build guidance](https://docs.docker.com/build/building/best-practices/)

**22. What is the difference between `CMD` and `ENTRYPOINT`?**

Both affect container startup.

* **`ENTRYPOINT`** defines the executable and any fixed arguments.
* **`CMD`** supplies the default command or default arguments to the entrypoint.

Example:

```dockerfile
FROM python:3.13-slim

WORKDIR /app
COPY app.py .

ENTRYPOINT ["python", "/app/app.py"]
CMD ["--port", "8080"]
```

Assuming the application accepts those arguments:

```bash
docker run myapp
```

Runs:

```text
python /app/app.py --port 8080
```

Passing arguments after the image replaces `CMD`:

```bash
docker run myapp --port 9090
```

Runs:

```text
python /app/app.py --port 9090
```

The entrypoint can also be overridden:

```bash
docker run --entrypoint sh -it myapp
```

The JSON-array form is called **exec form**. It avoids an implicit shell, which helps the application receive signals directly. Shell expansion does not happen automatically in this form. [Dockerfile reference](https://docs.docker.com/reference/dockerfile/)

**23. What are `run` and `exec` commands?**

In Docker, distinguish three operations:

| Operation        | When it runs                 | Purpose                                              |
| ---------------- | ---------------------------- | ---------------------------------------------------- |
| Dockerfile `RUN` | During image build           | Execute build steps and record filesystem changes    |
| `docker run`     | At container startup         | Create and start a new container from an image       |
| `docker exec`    | While a container is running | Start another process inside that existing container |

Dockerfile example:

```dockerfile
RUN mkdir -p /app/data
```

Create a new container:

```bash
docker run -d --name web -p 8080:80 nginx:stable
```

Execute a command inside it:

```bash
docker exec web nginx -t
```

Open an interactive shell, if the image contains one:

```bash
docker exec -it web sh
```

`docker exec` requires a running container. Changes made interactively are not automatically incorporated into the source image or Dockerfile. [Docker run](https://docs.docker.com/reference/cli/docker/container/run/), [Docker exec](https://docs.docker.com/reference/cli/docker/container/exec/)

Also, **exec form** in a Dockerfile means JSON-array syntax such as:

```dockerfile
RUN ["mkdir", "-p", "/app/data"]
```

It is different from the `docker exec` command.

**24. What is stored in `/var` and `/opt` in Linux?**

`/var` contains data expected to change while the system operates.

| Directory    | Typical contents                            |
| ------------ | ------------------------------------------- |
| `/var/log`   | Logs                                        |
| `/var/lib`   | Persistent application and service state    |
| `/var/cache` | Regenerable cached data                     |
| `/var/spool` | Queued work, such as mail or print jobs     |
| `/var/tmp`   | Temporary files intended to survive reboots |

Exact locations depend on the application and distribution. For example, databases commonly keep their data under a directory within `/var/lib`. [Filesystem Hierarchy Standard: `/var`](https://refspecs.linuxfoundation.org/FHS_3.0/fhs/ch05.html)

`/opt` contains add-on application software, often organized by application or vendor:

```text
/opt/company/application
/opt/vendor/product
```

Under the filesystem standard, configuration for these packages belongs under `/etc/opt`, and their changing data belongs under `/var/opt`. [Filesystem Hierarchy Standard: `/opt`](https://refspecs.linuxfoundation.org/FHS_3.0/fhs/ch03s13.html)

**25. Write a script that accepts `test.log` as an argument and separates error and warning lines.**

This script performs case-insensitive substring matching and creates two files beside the input file:

* `test.log.errors`
* `test.log.warnings`

Save as `split-log.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

if [[ $# -ne 1 ]]; then
    printf 'Usage: %s LOG_FILE\n' "$0" >&2
    exit 2
fi

log_file=$1

if [[ ! -f "$log_file" || ! -r "$log_file" ]]; then
    printf 'Cannot read log file: %s\n' "$log_file" >&2
    exit 2
fi

grep -iF -- 'error' "$log_file" > "${log_file}.errors" || {
    rc=$?
    [[ "$rc" -eq 1 ]] || exit "$rc"
}

grep -iF -- 'warning' "$log_file" > "${log_file}.warnings" || {
    rc=$?
    [[ "$rc" -eq 1 ]] || exit "$rc"
}

printf 'Created: %s\n' "${log_file}.errors" "${log_file}.warnings"
```

Run it:

```bash
chmod +x split-log.sh
./split-log.sh test.log
```

For a filename containing spaces:

```bash
./split-log.sh "application test.log"
```

How it works:

* `$1` supplies the input filename.
* `-i` ignores case, matching `error`, `ERROR`, and similar variations.
* `-F` treats the search pattern as literal text.
* `>` creates or overwrites each output file.
* A line containing both patterns appears in both files.
* No matches produce an empty output file.

`grep` returns `1` when it finds no matches. The script treats that as normal while preserving actual command failures.

I verified mixed-case and overlapping matches, filenames containing spaces, empty input, no matches, missing files, and missing arguments.


Below are detailed answers for the **EPAM AWS DevOps Engineer interview set**. Commands use illustrative resource names and IDs.

**1. What is the difference between ALB and NLB?**

| Aspect            | Application Load Balancer—ALB                          | Network Load Balancer—NLB                                                                     |
| ----------------- | ------------------------------------------------------ | --------------------------------------------------------------------------------------------- |
| Primary layer     | Layer 7: application                                   | Layer 4: transport                                                                            |
| Common protocols  | HTTP and HTTPS, including WebSocket and gRPC support   | TCP, TLS, UDP, and other supported transport protocols                                        |
| Routing decisions | Hostname, URL path, headers, and other HTTP conditions | Transport connections and flows                                                               |
| Addressing        | DNS name; underlying IP addresses can change           | Static addresses per enabled Availability Zone; optional Elastic IPs for internet-facing IPv4 |
| TLS               | HTTPS termination                                      | TLS termination or TCP forwarding for TLS passthrough                                         |
| Typical use       | Websites, APIs, microservices                          | TCP/UDP applications, static-IP requirements, transport-level workloads                       |
| AWS WAF           | Direct integration                                     | No direct WAF association                                                                     |

Use an **ALB** when requests to `/orders` and `/payments` must reach different application target groups. Use an **NLB** when exposing a TCP application or when clients require stable IP addresses. [ALB overview](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html), [NLB overview](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html)

Both support health checks. NLB also supports security groups; an older interview answer claiming that it never supports them is incorrect. The creation-time restriction is discussed in question 14.

**2. What is the purpose of a VPC endpoint? Give a use case.**

A VPC endpoint provides private connectivity to supported services or resources without requiring that traffic to use an internet gateway or NAT gateway.

Two common endpoint types are:

| Type               | How it works                                                                                  | Common example                                                      |
| ------------------ | --------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| Gateway endpoint   | Adds service-prefix routes to selected VPC route tables                                       | S3 or DynamoDB                                                      |
| Interface endpoint | Provides endpoint network interfaces with private addresses; applications connect through DNS | Secrets Manager, Systems Manager, and many other supported services |

Gateway endpoints do not use AWS PrivateLink. Interface endpoints do. Additional endpoint types support other connectivity patterns. [VPC endpoint concepts](https://docs.aws.amazon.com/vpc/latest/privatelink/concepts.html)

**Example:** An EC2 application in a private subnet needs to retrieve files from S3.

I would:

1. Create an S3 gateway endpoint.
2. Associate the application subnet’s route table.
3. Configure the endpoint policy for the required buckets.
4. Give the application’s IAM role the required S3 permissions.
5. Apply any required bucket-policy restrictions.
6. Test access from the application.

This can avoid NAT processing for that S3 traffic. An endpoint provides connectivity; it does not replace IAM authorization. [Gateway endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/gateway-endpoints.html)

**3. Can you obtain AMI details from a snapshot?**

**Sometimes, by finding an accessible AMI that references the snapshot.** A snapshot itself is not a complete AMI metadata record.

For an EBS-backed AMI, its block-device mappings contain snapshot IDs. Search those mappings in the snapshot’s Region:

```bash
aws ec2 describe-images \
  --region ap-south-1 \
  --owners self \
  --filters \
    "Name=block-device-mapping.snapshot-id,Values=snap-0123456789abcdef0" \
  --query 'Images[].{AMI:ImageId,Name:Name,State:State}'
```

`--owners self` limits this search to your account’s AMIs. Results depend on Region, permissions, and image visibility. [DescribeImages filters](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-images.html)

Limitations:

* A snapshot can exist without an AMI referencing it.
* An AMI can be deregistered while its snapshots remain.
* A copied snapshot might not preserve a discoverable relationship to the original AMI.
* A snapshot’s description can provide clues, but should not be treated as authoritative inventory.

A suitable snapshot containing a bootable root volume can also be used to register a **new** AMI with the required boot and platform settings. An arbitrary data-volume snapshot does not become bootable merely by registering it.

**4. How do you monitor load-balancer health using AWS services?**

Use **CloudWatch** for ongoing monitoring and **target health information** for diagnosis.

For an ALB, useful metrics include:

| Metric                      | What it helps detect                  |
| --------------------------- | ------------------------------------- |
| `HealthyHostCount`          | Available backend capacity            |
| `UnHealthyHostCount`        | Failing health checks                 |
| `TargetResponseTime`        | Backend latency                       |
| `HTTPCode_ELB_5XX_Count`    | Errors generated by the load balancer |
| `HTTPCode_Target_5XX_Count` | Errors returned by applications       |
| `RequestCount`              | Traffic volume                        |

Build dashboards and alarms with appropriate dimensions, statistics, and evaluation periods. For example, alert when healthy capacity falls below the application’s availability requirement. [ALB CloudWatch metrics](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-cloudwatch-metrics.html)

Inspect individual targets:

```bash
aws elbv2 describe-target-health \
  --target-group-arn "<target-group-arn>"
```

Check reason codes such as failed connections, timeouts, and response-code mismatches.

For NLBs, monitor protocol-appropriate metrics such as active flows, healthy targets, and TCP reset counts. [NLB CloudWatch metrics](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-cloudwatch-metrics.html)

Finally, use an external or synthetic application check. A healthy target does not prove that a complete customer transaction succeeds.

**5. What are the different types of instance profiles?**

The term needs clarification: in IAM, an **instance profile is a container for an IAM role that EC2 uses**.

AWS does not define standard profile categories such as “compute profile” or “memory profile.” Those usually refer to instance families.

You create profiles according to workload permissions, for example:

| Example profile        | Associated role’s purpose                                   |
| ---------------------- | ----------------------------------------------------------- |
| Application profile    | Read particular S3 objects and retrieve application secrets |
| Operations profile     | Support Systems Manager management                          |
| Image-building profile | Permit required image-build operations and logging          |

An instance profile contains **one IAM role**. That role can have multiple policies. An EC2 instance can have one associated instance profile, while the same profile can be used by multiple instances.

Applications obtain temporary role credentials through the supported AWS credential mechanisms, avoiding embedded long-lived access keys. [IAM instance profiles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2_instance-profiles.html)

**6. What is the difference between an AMI and a snapshot?**

| Aspect                 | AMI                                                                                   | EBS snapshot                                              |
| ---------------------- | ------------------------------------------------------------------------------------- | --------------------------------------------------------- |
| Purpose                | Describe an image from which EC2 instances can launch                                 | Preserve the contents of an EBS volume at a point in time |
| Contains or references | Boot metadata, architecture, block-device mappings, and—for EBS-backed AMIs—snapshots | Volume data                                               |
| Typical operation      | Launch an instance                                                                    | Restore an EBS volume                                     |
| Common use             | Standardized application or operating-system images                                   | Backup and recovery                                       |
| Relationship           | Can reference multiple snapshots                                                      | Can be referenced by an AMI                               |

An EBS snapshot is incremental in storage, but it contains the information needed to restore its point-in-time volume state. [EBS snapshots](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-snapshots.html)

An AMI does not reproduce an entire deployment’s VPC, security groups, IAM configuration, load balancer, and scaling policies. Those should be captured separately, usually in infrastructure code. [Amazon Machine Images](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html)

**7. The team changes the AMI in an ASG launch template. How do you ensure the new version is deployed correctly?**

**Creating a launch-template version does not replace existing instances.**

I would use a controlled rolling **Instance Refresh**:

1. Validate the AMI in a test instance or environment.
2. Create an explicit, numbered launch-template version referencing the approved AMI.
3. Start Instance Refresh with that version as its desired configuration.
4. Configure healthy-capacity limits, warmup, and sufficient temporary capacity.
5. Use checkpoints or a bake period to validate the rollout.
6. Check application health, error rates, latency, and actual user transactions.
7. Confirm that the active fleet uses the intended AMI and launch-template version. [Instance Refresh behavior](https://docs.aws.amazon.com/autoscaling/ec2/userguide/instance-refresh-overview.html)

For example, inspect the instances’ AMIs:

```bash
aws ec2 describe-instances \
  --filters \
    "Name=tag:aws:autoscaling:groupName,Values=web-asg" \
    "Name=instance-state-name,Values=running" \
  --query \
    'Reservations[].Instances[].{Instance:InstanceId,AMI:ImageId}'
```

Configure automatic rollback with suitable CloudWatch alarms where supported. Rollback requires an explicit desired configuration, and previous launch-template references must use numbered versions rather than `$Latest` or `$Default`.

If the ASG was already changed before the refresh began, verify the rollback target: the pre-refresh configuration may already reference the new AMI. [Instance Refresh rollback](https://docs.aws.amazon.com/autoscaling/ec2/userguide/instance-refresh-rollback.html)

**8. Is `170.90.00.9/0` public or private?**

First, normalize the notation carefully. The spaces and leading-zero octet in the original example can cause strict parsers to reject it. Assuming the intended address is **`170.90.0.9/0`**:

* The address `170.90.0.9` is in public address space, outside RFC 1918 private ranges.
* The prefix `/0` represents **all IPv4 addresses**.
* Its canonical network is `0.0.0.0/0`.

Therefore, distinguish the **individual address** from the **network prefix**:

| Item         | Meaning                                                    |
| ------------ | ---------------------------------------------------------- |
| `170.90.0.9` | Individual address outside private address space           |
| `0.0.0.0/0`  | All IPv4 destinations, including public and private ranges |

A `/0` route is commonly used as a default route. It does not make an individual resource publicly reachable.

**9. How do you determine whether an IPv4 address is public or private?**

First check whether it belongs to an RFC 1918 private range:

| CIDR             | Address range                   |
| ---------------- | ------------------------------- |
| `10.0.0.0/8`     | `10.0.0.0`–`10.255.255.255`     |
| `172.16.0.0/12`  | `172.16.0.0`–`172.31.255.255`   |
| `192.168.0.0/16` | `192.168.0.0`–`192.168.255.255` |

Not every `172.x.x.x` or `192.x.x.x` address is private. [RFC 1918](https://www.rfc-editor.org/rfc/rfc1918.html)

Then check special-purpose ranges. For example:

* `127.0.0.0/8`: loopback.
* `169.254.0.0/16`: link-local.
* `100.64.0.0/10`: shared address space.
* Other ranges are reserved for documentation, multicast, or special uses.

**Outside RFC 1918 does not automatically mean ordinary public unicast space.** Consult the IANA registry when classification is unclear. Reachability is a separate issue controlled by routing and security. [IANA IPv4 special-purpose registry](https://www.iana.org/assignments/iana-ipv4-special-registry/)

**10. Is `192.90.90.88/12` private or a host address?**

These describe different properties: a host address can be public or private.

For this example:

| Property                  | Value            |
| ------------------------- | ---------------- |
| Address                   | `192.90.90.88`   |
| Private RFC 1918 address? | No               |
| Network                   | `192.80.0.0/12`  |
| Subnet mask               | `255.240.0.0`    |
| Broadcast address         | `192.95.255.255` |

`192.90.90.88` has host bits set within that `/12` network. It is not the network address.

The private `192` range is specifically `192.168.0.0/16`; the entire `192.0.0.0/8` range is not private. I verified the subnet calculation programmatically.

**11. What is Transit Gateway in AWS?**

Transit Gateway—TGW—is a regional routing hub that connects VPCs and other networks, including VPN and Direct Connect connectivity.

A typical setup requires:

1. Create the TGW.
2. Create attachments for the participating networks.
3. Associate attachments with TGW route tables.
4. Configure route propagation or static routes.
5. Add routes in VPC subnet route tables for remote networks through the TGW.
6. Configure return routing and security controls.

Two concepts matter:

* **Association:** selects the TGW route table used for traffic arriving from an attachment.
* **Propagation:** adds an attachment’s advertised routes to selected TGW route tables.

An attachment can associate with one TGW route table and propagate to multiple tables. This enables network segmentation instead of automatically allowing every attached VPC to communicate. [How Transit Gateway works](https://docs.aws.amazon.com/vpc/latest/tgw/how-transit-gateways-work.html)

**12. After connecting VPCs through TGW, how do you block A-to-B and B-to-C traffic?**

Use **TGW route-table segmentation**, optionally with explicit blackhole routes.

Assume:

* A: `10.10.0.0/16`
* B: `10.20.0.0/16`
* C: `10.30.0.0/16`

For complete isolation between A–B and B–C, while preserving A–C communication:

| Incoming attachment’s table | Destination A | Destination B | Destination C |
| --------------------------- | ------------- | ------------- | ------------- |
| RT-A, associated with A     | —             | Blackhole     | Attachment C  |
| RT-B, associated with B     | Blackhole     | —             | Blackhole     |
| RT-C, associated with C     | Attachment A  | Blackhole     | —             |

Implementation steps:

1. Create and associate the route tables.
2. Disable unwanted automatic propagation into them.
3. Add only permitted routes.
4. Add blackhole routes where explicit drops are useful.
5. Check VPC routes, return paths, and alternate connectivity.

AWS supports TGW blackhole routes that discard matching traffic. [TGW blackhole routes](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-create-static-route.html)

Cover every relevant CIDR, and check for more-specific routes that could override a broader routing decision.

TGW routing is destination-based and is not a stateful firewall. If the requirement concerns ports, applications, or connection initiation while permitting responses, combine routing with security groups or appropriate firewall inspection.

**13. How can an EC2 instance in a private subnet receive inbound traffic?**

For a public web application, place an **internet-facing load balancer in public subnets** and register the private EC2 instance as a target.

The request path is:

1. The client reaches the public ALB.
2. The ALB connects to the instance’s private address.
3. The application returns its response through that connection.

Configure:

* Public subnet routing for the ALB.
* An HTTPS listener and target group.
* ALB security-group rules for permitted clients.
* Instance security-group rules allowing the application port **from the ALB security group**.
* Working health checks.

The EC2 instance does not require a public IP. [ALB security-group configuration](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-update-security-groups.html)

Other requirements use different paths:

* Corporate access: VPN or Direct Connect, potentially through an internal load balancer.
* Administration: Systems Manager Session Manager can provide access without opening inbound SSH. [Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html)

A NAT gateway is not the mechanism for unsolicited inbound access.

**14. How would you apply tight security to a load balancer?**

I would configure security at the network, TLS, application, and operational layers:

* Expose only required listeners and ports.
* Use HTTPS with ACM certificates and an appropriate TLS policy.
* Restrict source networks where the application’s audience allows it.
* Permit backend traffic only from the load balancer’s security group.
* Protect HTTP applications with suitable WAF rules.
* Use application authentication or supported ALB authentication features.
* Enable logs and alarms for failures and suspicious activity.
* Restrict IAM permissions for modifying listeners, certificates, and rules.
* If CloudFront fronts the application, prevent clients from bypassing its origin protections.

For NLBs, associate security groups **when creating the load balancer**. An NLB created without security groups cannot have them added later. [NLB security groups](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-security-groups.html)

AWS WAF can protect supported HTTP entry points such as ALBs and CloudFront distributions; it cannot attach directly to an NLB. [WAF-supported resources](https://docs.aws.amazon.com/waf/latest/developerguide/how-aws-waf-works-resources.html)

**15. Can different subpages use multiple load balancers?**

Yes, but the design depends on whether you mean **subdomains** or **URL paths**.

| Requirement                                     | Suitable design                                                     |
| ----------------------------------------------- | ------------------------------------------------------------------- |
| `shop.example.com` and `admin.example.com`      | Separate DNS records can point to different load balancers          |
| `example.com/shop` and `example.com/admin`      | One ALB can route paths to different target groups                  |
| One hostname with paths served by separate ALBs | CloudFront can route different cache behaviors to different origins |

For a single ALB:

| Rule       | Destination                 |
| ---------- | --------------------------- |
| `/api/*`   | API target group            |
| `/admin/*` | Administration target group |
| Default    | Frontend target group       |

Listener rules support HTTP-aware routing. [ALB listener rules](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-rules.html)

DNS does not see the URL path, so Route 53 cannot independently route `/api` and `/admin` under the same hostname. Use an HTTP-aware routing layer for that requirement.

**16. What is EC2 user data?**

User data supplies initialization instructions when an instance launches.

Typical tasks include:

* Installing software.
* Writing configuration.
* Starting services.
* Registering the instance with management systems.
* Retrieving application configuration.

Example for Amazon Linux 2023, assuming package-repository access:

```bash
#!/bin/bash
set -euo pipefail

dnf install -y nginx

printf '%s\n' 'Application is healthy' \
  > /usr/share/nginx/html/index.html

systemctl enable --now nginx
```

For typical Linux cloud-init configurations, user-data scripts run as root during the initial boot, rather than automatically on every reboot.

Useful troubleshooting commands:

```bash
sudo cloud-init status --long
sudo tail -n 100 /var/log/cloud-init-output.log
sudo journalctl -u cloud-final
```

Keep scripts repeatable where practical, and retrieve secrets using the instance role instead of embedding passwords in user data.

For fast ASG launches, preinstall substantial dependencies into the AMI and keep boot-time configuration small. [EC2 user data](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html)

**17. How do you extract critical information from VPC Flow Logs?**

Start with an operational question, such as:

* Which traffic is being rejected?
* Which hosts send the most data?
* Are unexpected systems accessing sensitive ports?
* Is the expected source reaching the intended interface?
* Are log-delivery gaps present?

Useful fields include source and destination addresses, ports, protocol, action, bytes, packets, interface ID, timestamps, and log status. Packet-level address fields help investigate traffic passing through intermediate devices. [Flow-log fields](https://docs.aws.amazon.com/vpc/latest/userguide/flow-log-records.html)

With CloudWatch Logs Insights and discovered VPC Flow Log fields:

```sql
filter action = "REJECT"
| stats count(*) as flowRecords,
        sum(bytes) as totalBytes
  by srcAddr, dstAddr, dstPort
| sort flowRecords desc
| limit 20
```

For a particular IP:

```sql
fields @timestamp, srcAddr, dstAddr, dstPort, action, bytes
| filter srcAddr = "10.20.1.10" or dstAddr = "10.20.1.10"
| sort @timestamp desc
| limit 100
```

Custom formats may require explicit parsing. [CloudWatch query examples](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax-examples.html)

Flow records aggregate traffic; their count is not necessarily a connection or request count. Correlate results with security rules, application logs, and the expected network design.

**18. What is the difference between Fargate and EKS worker nodes?**

Within EKS, the comparison is between **Fargate-backed pods** and **EC2-backed worker nodes**. Fargate is a compute option, not an alternative to the Kubernetes API.

| Aspect                | EKS on Fargate                                | EKS on EC2 workers                                     |
| --------------------- | --------------------------------------------- | ------------------------------------------------------ |
| Host management       | AWS manages underlying compute                | Responsibilities depend on the node-management model   |
| Capacity              | Allocated for eligible pods                   | Shared across pods on instances                        |
| Scheduling selection  | Fargate profiles select namespaces and labels | Kubernetes scheduling onto eligible nodes              |
| DaemonSets            | Unsupported                                   | Supported                                              |
| Privileged containers | Unsupported                                   | Possible when permitted                                |
| GPU workloads         | Unsupported                                   | Supported with appropriate instances and software      |
| EBS volumes for pods  | Unsupported                                   | Supported through appropriate CSI configuration        |
| Control               | Less host-level control                       | More flexibility over instances and node configuration |

Fargate can suit workloads that fit its capabilities and benefit from reduced host administration. EC2 workers suit workloads requiring specialized hardware, host agents, particular storage, or greater compute control.

Evaluate workload compatibility, utilization, and operational requirements. EKS Fargate can use supported EFS configurations, but that does not make it equivalent to EC2 storage support. [EKS Fargate considerations](https://docs.aws.amazon.com/eks/latest/userguide/fargate.html)

**19. How do you update an EKS cluster?**

I would use a staged upgrade process:

1. Review the target version, upgrade insights, and deprecated APIs.
2. Check controllers, admission webhooks, CRDs, and add-on compatibility.
3. Test in a representative non-production environment.
4. Confirm backups, recovery procedures, subnet IP capacity, and spare compute.
5. Prepare required add-ons and worker-version prerequisites.
6. Upgrade the control plane **one minor version at a time**.
7. Update worker nodes and add-ons in the tested, compatible sequence.
8. Drain old workers gradually while respecting disruption budgets.
9. Verify application transactions, DNS, networking, storage, and observability.

Use sufficient replicas, topology distribution, readiness checks, graceful shutdown, and appropriate PDBs to preserve application availability during worker replacement. Do not assume a control-plane upgrade automatically updates every node and add-on. [EKS upgrade process](https://docs.aws.amazon.com/eks/latest/userguide/update-cluster.html)

Current AWS documentation supports rolling eligible control planes back one minor version within **seven days** of an in-place upgrade. Compatibility and eligibility conditions apply; worker nodes and add-ons need separate consideration. [EKS rollback requirements](https://docs.aws.amazon.com/eks/latest/userguide/rollback-cluster.html)

**20. ASG launches instances, but terminates them during their two-to-three-minute initialization. How do you prevent this?**

First inspect **ASG activity history** and target-health reasons. Determine whether the termination is caused by failed health checks, scale-in, Spot interruption, or another event.

If healthy instances are being replaced because the application is still starting, configure these controls appropriately:

| Control                   | Purpose                                                                                               |
| ------------------------- | ----------------------------------------------------------------------------------------------------- |
| Health-check grace period | Gives a newly InService instance time before initialization-related health failures cause replacement |
| Default instance warmup   | Stabilizes scaling decisions while new capacity starts contributing useful metrics                    |
| Launch lifecycle hook     | Holds an instance in `Pending:Wait` until bootstrap completes                                         |
| Warm pool or prebuilt AMI | Reduces time needed to provide usable capacity                                                        |

Example:

```bash
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name web-asg \
  --health-check-grace-period 300 \
  --default-instance-warmup 300
```

The values are illustrative; use measured startup time plus health-check stabilization.

**Warmup does not replace the health-check grace period.** They control different behavior. Also, grace periods do not suppress every failure—for example, an instance leaving the EC2 running state can still be replaced. [Health-check grace period](https://docs.aws.amazon.com/autoscaling/ec2/userguide/health-check-grace-period.html), [instance warmup](https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling-default-instance-warmup.html)

For complex initialization, complete the launch lifecycle action only after bootstrap succeeds. [ASG lifecycle hooks](https://docs.aws.amazon.com/autoscaling/ec2/userguide/lifecycle-hooks.html)

**21. Traffic is high every day from 5 PM to 8 PM. How would you configure ASG?**

Use **scheduled scaling** to prepare capacity before the peak, and retain dynamic scaling for variations.

Example using `Asia/Kolkata`:

```bash
aws autoscaling put-scheduled-update-group-action \
  --auto-scaling-group-name web-asg \
  --scheduled-action-name prepare-evening-peak \
  --recurrence "55 16 * * *" \
  --time-zone "Asia/Kolkata" \
  --min-size 10 \
  --desired-capacity 10 \
  --max-size 30
```

Restore the lower baseline after the peak:

```bash
aws autoscaling put-scheduled-update-group-action \
  --auto-scaling-group-name web-asg \
  --scheduled-action-name restore-normal-baseline \
  --recurrence "5 20 * * *" \
  --time-zone "Asia/Kolkata" \
  --min-size 2 \
  --desired-capacity 2 \
  --max-size 30
```

These capacities are illustrative. Determine them through load testing, allow sufficient startup time, and account for active requests or queued work before reducing capacity.

ASG recurring schedules use five-field cron expressions and support named time zones. Use the business’s actual time zone rather than relying on the default UTC behavior. [Scheduled scaling](https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling-scheduled-scaling.html)

**22. Can one VPC have CIDR blocks from both the `172` and `192` series?**

**The complete CIDRs matter.**

A VPC supports multiple IPv4 CIDR blocks, but AWS imposes association restrictions.

If you mean combining:

* `172.20.0.0/16`, from the private `172.16.0.0/12` range, and
* `192.168.0.0/16`, from the other private range,

**AWS’s documented association restrictions prohibit that combination in one VPC.**

A permitted example, assuming no other conflicts, is:

```text
Primary:   172.20.0.0/16
Secondary: 172.21.0.0/16
```

Both belong to the same permitted private range and do not overlap.

Also check:

* Existing routes and connected-network overlap.
* CIDR association quotas.
* Allowed IPv4 block sizes.
* Whether additional subnets need to be created.

Adding a secondary CIDR does not enlarge an existing subnet. AWS adds a corresponding local route when the CIDR is associated. [VPC CIDR association restrictions](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-cidr-blocks.html)

**23. How do you customize AWS WAF?**

Configure a web ACL with managed rules and application-specific rules.

Typical customizations include:

| Requirement                                    | Rule approach                                            |
| ---------------------------------------------- | -------------------------------------------------------- |
| Block known unwanted sources                   | IP sets                                                  |
| Protect login endpoints from excessive traffic | Rate-based rule scoped to login paths                    |
| Detect common injection patterns               | Suitable managed rules or custom inspection              |
| Restrict administrative URLs                   | Path conditions combined with approved-source conditions |
| Inspect application-specific headers           | Header or regex matching                                 |
| Apply different behavior to selected traffic   | Rule scope and logical conditions                        |

A practical deployment process is:

1. Define the protected application and expected traffic.
2. Add suitable managed rules.
3. Add narrowly scoped custom rules.
4. Set priorities deliberately.
5. Evaluate new rules in **Count** mode.
6. Review false positives and tune.
7. Enable blocking or other appropriate actions.
8. Monitor logs and metrics.

An early terminating Allow rule can bypass later inspection, so exceptions must be carefully scoped. [WAF rules](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rules.html)

Use the correct regional or CloudFront scope and test changes before enforcement. [WAF configuration guidance](https://docs.aws.amazon.com/waf/latest/developerguide/web-acl.html)

**24. How would you configure CloudFront?**

CloudFront distributes content through edge locations and can cache responses to reduce origin traffic and latency.

For a website with static assets and an API:

| Path        | Origin              | Typical caching approach                                                    |
| ----------- | ------------------- | --------------------------------------------------------------------------- |
| `/static/*` | S3                  | Cache versioned assets                                                      |
| `/api/*`    | ALB or API endpoint | Disable caching initially, or design it carefully around response semantics |

Configuration steps:

1. Create the distribution and origins.
2. Configure ordered cache behaviors and the default behavior.
3. Define cache keys, TTLs, and origin-request forwarding.
4. Require or redirect viewers to HTTPS.
5. Configure the custom domain and certificate.
6. Add Route 53 alias records.
7. Configure origin access restrictions, WAF, logging, and monitoring.
8. Test cache hits, misses, authentication, and error handling. [CloudFront cache behaviors](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DownloadDistValuesCacheBehavior.html)

For a private S3 origin, use **Origin Access Control—OAC** and an appropriately restricted bucket policy. OAC applies to supported S3 bucket origins, not S3 website endpoints. [Restricting S3 origin access](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html)

For viewer HTTPS with an ACM certificate, request or import the certificate in **`us-east-1`**. [CloudFront certificate requirements](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cnames-and-https-requirements.html)

Avoid sharing cached authenticated responses between users through an incorrectly designed cache policy.

**25. What is EC2 Image Builder?**

EC2 Image Builder automates the creation, testing, and distribution of AMIs and container images.

Its main building blocks include:

| Component                    | Purpose                                                          |
| ---------------------------- | ---------------------------------------------------------------- |
| Recipe                       | Base image and selected build components                         |
| Build components             | Install, patch, and configure software                           |
| Test components              | Validate the resulting image                                     |
| Infrastructure configuration | Build-instance settings, IAM, networking, and logging            |
| Distribution configuration   | Destination Regions, accounts, and related distribution settings |
| Pipeline/workflow            | Coordinate and schedule the process                              |

Example use case:

A pipeline starts from an approved Linux image, installs the application runtime and monitoring agent, applies configuration, tests startup, and distributes the approved AMI.

The deployment pipeline then references that AMI in a new launch-template version and performs a controlled rollout.

This reduces boot-time work and produces repeatable images. Image Builder can validate images before distribution; application deployment still needs its own rollout controls. [EC2 Image Builder](https://docs.aws.amazon.com/imagebuilder/latest/userguide/what-is-image-builder.html)

**26. What are the different types of EC2 instances?**

Instance families are designed around different resource requirements.

| Category                   | Example family prefixes | Typical use                                              |
| -------------------------- | ----------------------- | -------------------------------------------------------- |
| General purpose            | M, T                    | Application servers and balanced workloads               |
| Compute optimized          | C                       | CPU-intensive processing                                 |
| Memory optimized           | R, X                    | Memory-intensive databases and analytics                 |
| Storage optimized          | I, D                    | High local-storage throughput or IOPS                    |
| Accelerated computing      | G, P, Inf, Trn          | Graphics, machine learning, and specialized acceleration |
| High-performance computing | Hpc                     | Suitable scientific and engineering workloads            |

T-family instances are burstable, so CPU-credit behavior matters.

Choose using measured requirements: memory, CPU, architecture, EBS bandwidth, network performance, local storage, and workload compatibility—not CPU count alone. [EC2 instance types](https://docs.aws.amazon.com/ec2/latest/instancetypes/instance-types.html)

Instance families, IAM instance profiles, and purchasing options such as Spot are separate concepts.

**27. If VPC Flow Logs are stored in S3, how do you inspect them?**

For occasional inspection, retrieve an individual log object and open it using a tool compatible with its format and compression.

For regular analysis, use **Athena**:

1. Define a table matching the flow-log schema and S3 layout.
2. Register it in the catalog.
3. Configure partitions or partition projection.
4. Grant the required S3, catalog, query-output, and applicable KMS access.
5. Run SQL queries filtered to relevant accounts, Regions, and dates.

Example, assuming these columns and a `yyyy/MM/dd` day partition:

```sql
SELECT
    srcaddr,
    dstaddr,
    dstport,
    action,
    COUNT(*) AS flow_records,
    SUM(bytes) AS total_bytes
FROM vpc_flow_logs
WHERE day = '2026/09/21'
  AND (
      srcaddr = '10.20.1.10'
      OR dstaddr = '10.20.1.10'
  )
GROUP BY srcaddr, dstaddr, dstport, action
ORDER BY total_bytes DESC
LIMIT 50;
```

Match the schema to the actual configured fields. Use partition filters to limit scans; suitable Parquet delivery can also improve analytical efficiency. [Athena VPC Flow Log queries](https://docs.aws.amazon.com/athena/latest/ug/vpc-flow-logs-partition-projection.html)

**28. How would you configure API Gateway?**

First select the API product based on requirements:

* **HTTP API:** commonly suitable for straightforward HTTP or Lambda integrations.
* **REST API:** supports additional capabilities such as usage plans, request validation, direct WAF integration, and private API endpoints.
* **WebSocket API:** supports persistent bidirectional messaging. [HTTP versus REST APIs](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-vs-rest.html)

For an HTTP API backed by Lambda:

1. Create the API.
2. Define routes, such as `GET /orders` and `POST /orders`.
3. Create and attach the Lambda integration.
4. Grant API Gateway permission to invoke the function.
5. Configure appropriate authorization.
6. Configure required CORS behavior.
7. Deploy to a stage.
8. Configure a custom domain, certificate, and DNS.
9. Enable access logs, metrics, and appropriate throttling.
10. Test successful requests, invalid inputs, unauthorized requests, and backend failures.

For private backends, use a supported VPC Link integration—for example, to an internal ALB. A private backend integration does not by itself make the API’s client-facing endpoint private. [HTTP API private integrations](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-develop-integrations-private.html)

API keys support identification and usage-management functions; they should not replace proper user authentication and authorization.

**29. What is the difference between private and public IP addresses?**

For conventional IPv4 networking:

| Aspect                  | Private address                                 | Public address                                  |
| ----------------------- | ----------------------------------------------- | ----------------------------------------------- |
| Typical address space   | RFC 1918 ranges                                 | Publicly allocated address space                |
| Uniqueness              | Can be reused in separate networks              | Must be globally coordinated for public routing |
| Public internet routing | Not directly routed as private space            | Can participate in public routing               |
| Typical use             | Internal application and database communication | Public-facing network endpoints                 |

A public IP does not automatically make a service accessible. Routing, security groups, NACLs, listeners, and application behavior still matter.

In AWS, direct EC2 IPv4 internet connectivity ordinarily requires the appropriate public-address mapping and internet-gateway route, with security controls permitting the traffic.

AWS also uses “private IP” for an interface’s VPC-side address even when the VPC uses a non-RFC-1918 CIDR. Keep address classification separate from AWS interface terminology and actual reachability. [VPC addressing considerations](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-cidr-blocks.html)

**30. What is the difference between Spot and Reserved Instances?**

| Aspect             | Spot Instances                                           | Reserved Instances—RIs                                           |
| ------------------ | -------------------------------------------------------- | ---------------------------------------------------------------- |
| Basic model        | Use available spare EC2 capacity                         | Receive a billing discount for matching usage under a commitment |
| Commitment         | No RI-style one- or three-year commitment                | Typically one or three years                                     |
| Availability       | Depends on available Spot capacity                       | Capacity reservation depends on RI scope                         |
| Interruption       | AWS can reclaim capacity                                 | No Spot-reclamation behavior                                     |
| Typical workload   | Retryable jobs, batch processing, fault-tolerant workers | Predictable baseline usage                                       |
| Main consideration | Tolerate interruption and capacity variation             | Avoid paying for a commitment that is underused                  |

An RI is primarily a **billing construct**, not a different hardware instance family. A **regional RI does not reserve capacity**; a qualifying **zonal RI reserves capacity in its specified Availability Zone**. [Reserved Instances overview](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-reserved-instances.html), [regional and zonal scope](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/reserved-instances-scope.html)

Spot workloads should tolerate replacement through retries, checkpointing, and suitable instance diversification. Stop/terminate interruption notices generally provide two minutes; hibernation starts immediately rather than providing that two-minute interval. [Spot interruption notices](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-instance-termination-notices.html)

A common design maintains dependable baseline capacity with applicable commitment discounts and uses Spot for additional work that can tolerate interruption.

