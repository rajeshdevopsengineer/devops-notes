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
