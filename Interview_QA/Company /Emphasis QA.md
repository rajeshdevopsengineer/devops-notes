Below are interview-ready explanations for the **Emphasis DevOps questions**, with practical AWS and Kubernetes examples.

**1. Explain the components of a three-tier architecture**

A three-tier architecture separates an application into **presentation, business logic, and data storage**.

| Tier                  | Responsibility                                      | Example                                   |
| --------------------- | --------------------------------------------------- | ----------------------------------------- |
| Presentation/web tier | Displays the interface and handles user interaction | Website, frontend application, web server |
| Application tier      | Processes business rules and requests               | Java, .NET, Python, or Node.js API        |
| Data tier             | Stores and retrieves persistent information         | MySQL, PostgreSQL, Amazon RDS             |

For an online shopping application:

1. The presentation tier displays products and the shopping cart.
2. The application tier calculates prices, validates inventory, and processes orders.
3. The data tier stores products, customers, and transactions.

Separating these responsibilities allows tiers to scale and change independently. ([AWS Serverless Multi-Tier Architectures with ...][1])

**An AWS implementation could use:**

* Route 53 for DNS.
* An internet-facing Application Load Balancer as the entry point.
* Web and application instances in Auto Scaling groups.
* Private subnets for application servers and databases.
* Amazon RDS for the database.
* Security groups that permit only the required communication between tiers.

For example, the database security group can allow PostgreSQL traffic on port `5432` from the application security group.

**Three tiers do not mean exactly three servers.** Each tier can contain several instances, containers, or managed services. For availability, distribute supported components across Availability Zones.

**2. Explain Kubernetes architecture**

A Kubernetes cluster consists of a **control plane** and **worker nodes**.

The control plane maintains the desired cluster state. Worker nodes run application Pods.

| Component                                     | Responsibility                                                          |
| --------------------------------------------- | ----------------------------------------------------------------------- |
| `kube-apiserver`                              | Exposes the Kubernetes API and handles requests                         |
| `etcd`                                        | Stores cluster configuration and state                                  |
| `kube-scheduler`                              | Selects suitable nodes for unscheduled Pods                             |
| `kube-controller-manager`                     | Runs controllers that reconcile desired and actual state                |
| `cloud-controller-manager`                    | Handles supported cloud-provider integration                            |
| `kubelet`                                     | Ensures assigned Pods run on a node                                     |
| Container runtime                             | Pulls images and runs containers; examples include containerd and CRI-O |
| `kube-proxy` or an alternative implementation | Implements Service traffic handling                                     |

Networking plugins provide Pod connectivity. Cluster DNS, commonly CoreDNS, supports service discovery.

**What happens when you create a Deployment?**

1. `kubectl` sends the specification to the API server.
2. The API server validates the request and persists the object.
3. Controllers create the required ReplicaSet and Pods.
4. The scheduler assigns Pods to nodes.
5. Each node’s kubelet works with the runtime to start its assigned containers.
6. Controllers continue reconciling the workload as conditions change.

The scheduler selects a node; it does not start containers itself. ([Kubernetes][2])

For troubleshooting, this separation helps: scheduling failures, image-pull failures, and application crashes occur at different stages.

**3. How does a private subnet connect to the outside world?**

First clarify what “outside” means: **the internet, another VPC, or an on-premises network**.

For outbound IPv4 internet access, a common design uses a public NAT Gateway.

For a conventional zonal NAT Gateway setup:

1. Attach an Internet Gateway to the VPC.
2. Create the public NAT Gateway in a public subnet and associate an Elastic IP.
3. Give the public subnet a default route to the Internet Gateway.
4. Give the private subnet a default route to the NAT Gateway.
5. Permit the required traffic through security groups and NACLs.

Example routing:

| Route table    | Destination | Target           |
| -------------- | ----------- | ---------------- |
| Private subnet | VPC CIDR    | Local            |
| Private subnet | `0.0.0.0/0` | NAT Gateway      |
| Public subnet  | VPC CIDR    | Local            |
| Public subnet  | `0.0.0.0/0` | Internet Gateway |

An instance can initiate an HTTPS request, and the reply can return through the NAT translation. An internet client cannot use that NAT Gateway to initiate an unsolicited connection to the instance. ([Amazon Virtual Private Cloud][3])

Other requirements need different connectivity:

* **Public users accessing a private application:** Use an appropriate public entry point, such as an internet-facing load balancer.
* **Private access to AWS services:** Consider VPC endpoints.
* **On-premises connectivity:** Use Site-to-Site VPN or Direct Connect.
* **Connectivity between VPCs:** Use suitable peering or Transit Gateway routing.

A subnet remains private even when its resources have outbound internet access through NAT.

**4. What is the difference between NACLs and security groups?**

Both filter network traffic, but they operate differently.

| Feature             | Security group                                          | Network ACL                                |
| ------------------- | ------------------------------------------------------- | ------------------------------------------ |
| Scope               | Associated resources/network interfaces                 | Subnet boundary                            |
| Connection tracking | Stateful                                                | Stateless                                  |
| Rules               | Allow rules; unmatched traffic is denied                | Explicit allow and deny rules              |
| Rule processing     | Applicable allow rules are combined                     | Lowest rule number first; first match wins |
| Return traffic      | Automatically allowed for tracked permitted connections | Must be permitted by the relevant rules    |
| Association         | An interface can have multiple security groups          | A subnet has one associated NACL           |
| Typical purpose     | Fine-grained workload access                            | Subnet-level controls                      |

AWS documents the stateful behavior of security groups and the ordered, stateless behavior of NACLs. ([Amazon Virtual Private Cloud][4])

**Example: outbound HTTPS**

An EC2 instance connects from a temporary client port, such as `55000`, to a remote server’s port `443`.

* Its security group allows the outbound connection and automatically permits the tracked reply.
* Its subnet NACL must permit the outgoing request and the returning traffic to the client’s temporary port.

Allowing only destination port `443` in both NACL directions would not correctly permit this return path.

Also distinguish defaults: the VPC’s default NACL initially permits traffic, while a newly created custom NACL initially denies traffic until rules are added.

**5. What is the purpose of a NAT Gateway?**

NAT stands for **Network Address Translation**.

A public NAT Gateway lets resources with private IPv4 addresses initiate internet connections without assigning a public IP address to every resource.

Typical uses include:

* Downloading operating-system updates.
* Downloading application dependencies.
* Calling third-party APIs.
* Pulling images from internet-accessible registries.

It tracks address translations so replies can reach the resource that initiated the connection. For internet access, a public NAT Gateway works with the VPC’s Internet Gateway and Elastic IP address. ([Amazon Virtual Private Cloud][5])

Important points:

* It does not publish a private web server to internet clients.
* It does not replace security groups or other required traffic controls.
* You cannot attach a security group directly to a NAT Gateway.
* A single zonal NAT Gateway can create an Availability Zone dependency. For a resilient zonal design, use NAT Gateways and routing appropriate to each AZ. ([Amazon Virtual Private Cloud][6])

In an interview, distinguish **public NAT for internet egress** from **private NAT for supported private-network translation scenarios**.

**6. Explain how to write a Dockerfile**

A Dockerfile contains instructions for building a container image.

The usual steps are:

1. Choose a suitable base image.
2. Set the working directory.
3. Copy dependency definitions and install dependencies.
4. Copy the application source.
5. Build or publish the application.
6. Define the runtime configuration and startup command.

For example, assume a single-project **.NET 10 API**, with `MyApi.csproj` in the project root:

```dockerfile
# syntax=docker/dockerfile:1

# Build stage
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build

WORKDIR /src

COPY MyApi.csproj ./
RUN dotnet restore MyApi.csproj

COPY . .
RUN dotnet publish MyApi.csproj \
    -c Release \
    --no-restore \
    -o /publish \
    -p:UseAppHost=false

# Runtime stage
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS runtime

WORKDIR /app

ENV ASPNETCORE_HTTP_PORTS=8080

COPY --from=build /publish ./

USER app

EXPOSE 8080

ENTRYPOINT ["dotnet", "MyApi.dll"]
```

This is a **multi-stage build**: the SDK compiles the application, while the final image contains the runtime and published output. Microsoft’s .NET Linux images include the non-root `app` user used here. ([.NET][7])

| Instruction  | Meaning                                            |
| ------------ | -------------------------------------------------- |
| `FROM`       | Selects a base image or starts a build stage       |
| `WORKDIR`    | Sets the working directory                         |
| `COPY`       | Copies files into the image                        |
| `RUN`        | Executes a command during the build                |
| `ENV`        | Sets an environment variable                       |
| `USER`       | Selects the user for subsequent operations/runtime |
| `EXPOSE`     | Documents the intended container port              |
| `ENTRYPOINT` | Defines the container’s main executable            |
| `CMD`        | Supplies a default command or default arguments    |

`RUN` executes while building an image. `ENTRYPOINT` and `CMD` determine what runs when a container starts. ([Docker Docs][8])

Add a `.dockerignore`:

```text
.git
**/bin
**/obj
.env*
secrets/
```

Build and run:

```bash
docker build -t myapi:1.0 .

docker run --rm \
  -p 127.0.0.1:9090:8080 \
  myapi:1.0
```

The host’s port `9090` maps to the container’s port `8080`. **`EXPOSE` alone does not publish a port.**

Copying the project file before the source also helps Docker reuse the dependency-restore cache when only application code changes.

**7. From where is an image pulled when you run `docker pull image`?**

If you do not specify a registry, Docker normally uses **Docker Hub**.

For example:

```bash
docker pull nginx
```

Resolves to:

```text
docker.io/library/nginx:latest
```

The components are:

| Part        | Meaning                                   |
| ----------- | ----------------------------------------- |
| `docker.io` | Registry                                  |
| `library`   | Namespace used for Docker Official Images |
| `nginx`     | Repository                                |
| `latest`    | Default tag when none is supplied         |

For another registry, include its hostname:

```bash
docker pull registry.example.com/team/myapi:1.0
```

Docker retrieves the image metadata and downloads required layers that are not already available locally. ([Docker Docs][9])

Two common interview follow-ups:

* **`latest` is a tag**, not a guarantee that Docker has selected the newest release chronologically.
* **Pulling an image does not start a container.** Use `docker run` or an orchestrator to run it.

For reproducible deployments, use controlled version tags and preferably record or deploy the exact image digest.

**8. How do you pull an image from a private repository?**

You need:

1. Network connectivity to the registry.
2. Authentication.
3. Authorization to pull the repository.
4. The correct image name and tag or digest.

For a generic registry:

```bash
docker login registry.example.com

docker pull registry.example.com/team/myapi:1.0
```

For automated use, supply a token securely through standard input:

```bash
printf '%s' "$REGISTRY_TOKEN" |
  docker login registry.example.com \
    --username "$REGISTRY_USER" \
    --password-stdin
```

Docker supports registry credentials and credential helpers. Avoid placing a literal password in the command line. ([Docker Docs][10])

**Amazon ECR example**

Assume your AWS CLI already has an authorized identity:

```bash
aws ecr get-login-password --region us-east-1 |
  docker login \
    --username AWS \
    --password-stdin \
    123456789012.dkr.ecr.us-east-1.amazonaws.com
```

Then pull:

```bash
docker pull \
  123456789012.dkr.ecr.us-east-1.amazonaws.com/myapi:1.0
```

Replace the account ID, region, repository, and tag. The ECR authorization token is valid for 12 hours, and its permission scope follows the IAM identity used to obtain it. ([Amazon ECR][11])

**Kubernetes follow-up**

Logging in on your laptop does not authenticate Kubernetes worker nodes.

For ECR images on EKS, image-pull permissions normally come from the worker-node IAM role, or the Fargate pod execution role for Fargate workloads. ([Amazon ECR][12])

If a pull fails, check the registry hostname, image/tag existence, permissions, token expiry, and network/DNS connectivity.

**9. What is Ingress in Kubernetes?**

An **Ingress** defines HTTP/HTTPS routing rules from outside the cluster to application Services.

It commonly supports:

* Host-based routing: `shop.example.com` and `api.example.com`.
* Path-based routing: `/orders` and `/payments`.
* TLS termination, depending on the controller and configuration.

**An Ingress resource requires an Ingress controller.** Creating the YAML object alone does not create a working traffic path.

On EKS, the AWS Load Balancer Controller can implement Ingress routing through an Application Load Balancer. ([Amazon EKS][13])

Example for an EKS cluster where that controller and the required AWS networking are already configured:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapi-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapi-service
                port:
                  number: 80
```

This assumes:

* A matching `alb` IngressClass/controller is available.
* `myapi-service` exists in the same namespace.
* Its backend Pods are reachable and ready.
* DNS for `api.example.com` points to the provisioned load balancer.

The example defines HTTP routing; production HTTPS needs the corresponding certificate and listener configuration.

Conceptually, Ingress selects a backend Service. The actual traffic path depends on the controller—for example, an ALB using IP targets can send traffic directly to Pod IPs.

The Ingress API remains available, but it is frozen for new development. Kubernetes recommends considering **Gateway API** for newer routing designs. ([Kubernetes][14])

**10. Explain a CI/CD pipeline and its stages**

A CI/CD pipeline automates the steps that turn a source-code change into a tested, deployable release.

* **Continuous integration:** Frequently integrate changes and automatically build and test them.
* **Continuous delivery:** Keep validated changes ready for production, with a release decision or approval.
* **Continuous deployment:** Automatically deploy changes that pass the required checks.

A typical pipeline contains:

| Stage                     | Purpose                                                         |
| ------------------------- | --------------------------------------------------------------- |
| Trigger and checkout      | Start from a repository event and retrieve the intended commit  |
| Build                     | Compile or package the application                              |
| Unit tests                | Check individual components                                     |
| Code analysis             | Check code quality and potential defects                        |
| Security checks           | Check dependencies, secrets, and relevant vulnerabilities       |
| Image/package creation    | Produce a versioned deployment artifact                         |
| Artifact publication      | Push the artifact to ECR or another repository                  |
| Development/QA deployment | Deploy and run integration or functional tests                  |
| UAT/staging validation    | Validate business requirements and production-like behavior     |
| Production release        | Apply release controls and deploy                               |
| Verification              | Check rollout status, application health, and business behavior |

In Jenkins, the pipeline can be defined in a version-controlled `Jenkinsfile`. Its stages organize the build, test, and delivery process. ([jenkins.io][15])

**Example using AWS and Kubernetes**

A developer raises a pull request. The pipeline validates the code. After the change is merged, it builds a container image, scans it, and pushes it to ECR using an identifiable version.

The deployment process then updates the Kubernetes workload through Helm, manifests, or a GitOps workflow.

A rollout check might be:

```bash
kubectl rollout status deployment/myapi \
  --namespace production \
  --timeout=180s
```

A completed rollout should be followed by application checks. Ready Pods alone do not prove that customers can successfully use every required dependency.

The main practices I would explain in an interview are:

* Promote the **same tested artifact** through environments.
* Keep environment configuration separate from the image.
* Use scoped credentials and approved secret storage.
* Make required test failures block promotion.
* Retain the previous release and a tested rollback procedure.
* Handle database compatibility separately, because rolling back application code does not undo database changes.

[1]: https://docs.aws.amazon.com/whitepapers/latest/serverless-multi-tier-architectures-api-gateway-lambda/introduction.html?utm_source=chatgpt.com "Introduction"
[2]: https://kubernetes.io/docs/concepts/overview/components/?utm_source=chatgpt.com "Kubernetes Components"
[3]: https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-scenarios.html?utm_source=chatgpt.com "NAT gateway use cases"
[4]: https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html?utm_source=chatgpt.com "Control subnet traffic with network access control lists"
[5]: https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html?utm_source=chatgpt.com "NAT gateways"
[6]: https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-basics.html?utm_source=chatgpt.com "NAT gateway basics"
[7]: https://learn.microsoft.com/en-us/dotnet/core/docker/build-container?utm_source=chatgpt.com "Containerize an app with Docker tutorial"
[8]: https://docs.docker.com/build/concepts/dockerfile/?utm_source=chatgpt.com "Dockerfile overview"
[9]: https://docs.docker.com/reference/cli/docker/image/pull/?utm_source=chatgpt.com "docker image pull"
[10]: https://docs.docker.com/reference/cli/docker/login/?utm_source=chatgpt.com "docker login"
[11]: https://docs.aws.amazon.com/AmazonECR/latest/userguide/registry_auth.html?utm_source=chatgpt.com "Private registry authentication in Amazon ECR"
[12]: https://docs.aws.amazon.com/AmazonECR/latest/userguide/ECR_on_EKS.html?utm_source=chatgpt.com "Using Amazon ECR Images with Amazon EKS"
[13]: https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html?utm_source=chatgpt.com "Route application and HTTP traffic with Application Load Balancers"
[14]: https://kubernetes.io/docs/concepts/services-networking/ingress/?utm_source=chatgpt.com "Ingress"
[15]: https://www.jenkins.io/doc/book/pipeline/jenkinsfile/?utm_source=chatgpt.com "Using a Jenkinsfile"
