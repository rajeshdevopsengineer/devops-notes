**1. What is desired state and “in-desired state”?**

“In-desired state” is not a standard Kubernetes term. The interviewer probably means **actual/current state** or an **undesired state**.

| Term | Meaning | Example |
|---|---|---|
| Desired state | What you declare should exist | Three application replicas running version `v2` |
| Actual state | What currently exists | Only two replicas are running |
| Undesired state | Actual state differs from the intended configuration | A missing replica, failed container, or incorrect image version |

In Kubernetes, the desired configuration is generally represented by an object’s **`spec`**, while **`status`** reports its observed state. Controllers continuously work to bring the actual state toward the desired state. This process is called **reconciliation**. :chatgpt-content-reference{index="0"}

For example:

```yaml
spec:
  replicas: 3
```

If a Deployment-managed Pod is deleted, the relevant controllers create a replacement to maintain three replicas. However, if the cluster lacks capacity, the replacement can remain `Pending`; reconciliation cannot overcome missing resources automatically.

**Interview answer:**

> “Desired state is the configuration we declare, such as three application replicas. Actual state is what currently exists. Kubernetes controllers continuously compare the two and take corrective action when they differ.”

---

**2. How do you deploy an application in Kubernetes?**

A typical process is:

1. Build and test the application.
2. Build a container image and scan it.
3. Push the image to an approved registry.
4. Prepare application configuration and secrets.
5. Deploy the workload and create a Service.
6. Verify rollout status, application health, and connectivity.

Use a **Deployment** for a typical stateless application. It manages ReplicaSets and supports controlled rolling updates. :chatgpt-content-reference{index="1"}

For example, save this as `application.yaml`. Replace the image reference with your image; the application must listen on port `8080` and implement the specified health endpoints.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders
spec:
  replicas: 2
  selector:
    matchLabels:
      app: orders
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    metadata:
      labels:
        app: orders
    spec:
      containers:
        - name: orders
          image: registry.example.com/team/orders:1.4.2
          ports:
            - name: http
              containerPort: 8080
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "1"
              memory: "512Mi"
          readinessProbe:
            httpGet:
              path: /readyz
              port: http
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            initialDelaySeconds: 30
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: orders
spec:
  type: ClusterIP
  selector:
    app: orders
  ports:
    - name: http
      port: 80
      targetPort: http
```

Deploy and verify:

```bash
kubectl create namespace demo

kubectl apply -n demo \
  --dry-run=server -f application.yaml

kubectl apply -n demo -f application.yaml

kubectl rollout status deployment/orders \
  -n demo --timeout=180s

kubectl get deployments,pods,services -n demo

kubectl logs -n demo deployment/orders --tail=100
```

The Service selects Pods using `app: orders` and provides a stable endpoint. For external HTTP access, configure an Ingress or Gateway with a suitable controller.

For a private registry, configure an appropriate image-pull identity or `imagePullSecrets` in the workload’s namespace. :chatgpt-content-reference{index="2"}

Additional production considerations:

- Use an immutable image digest for release reproducibility.
- Set resource values from measurements.
- Add a startup probe for slow-starting applications.
- Configure TLS, access controls, and monitoring.
- Ensure sufficient capacity for the additional Pod during a rolling update.

For an imperative rollback:

```bash
kubectl rollout undo deployment/orders -n demo
```

In a GitOps setup, also revert the desired configuration in Git so the controller preserves the rollback.

---

**3. Have you faced memory issues in a Jenkins pipeline? How would you troubleshoot?**

For the experience portion, describe an incident you actually handled. The troubleshooting approach should begin by identifying **which process is running out of memory**.

| Location | Typical symptoms |
|---|---|
| Jenkins controller | UI becomes slow, widespread pipeline problems, controller JVM errors |
| Jenkins agent JVM | Agent disconnects or reports Java heap errors |
| Build process | Maven, Gradle, Node.js, or tests fail while Jenkins remains healthy |
| Container or host | `OOMKilled`, kernel OOM messages, or a process is forcibly terminated |

**First, collect evidence.**

Read the Jenkins console output and correlate the failure with the build stage. Look for:

```text
java.lang.OutOfMemoryError: Java heap space
GC overhead limit exceeded
JavaScript heap out of memory
OOMKilled
```

Exit code `137` means the process received `SIGKILL`; it is **not sufficient evidence by itself** to conclude that the process ran out of memory. Check container termination details and kernel logs. :chatgpt-content-reference{index="3"}

For a Kubernetes agent:

```bash
kubectl describe pod <agent-pod> -n ci

kubectl top pod <agent-pod> -n ci --containers

kubectl get pod <agent-pod> -n ci \
  -o jsonpath='{range .status.containerStatuses[*]}{.name}{" current="}{.state.terminated.reason}{" previous="}{.lastState.terminated.reason}{"\n"}{end}'
```

`kubectl top` requires a metrics provider and only shows current usage. Use historical monitoring to investigate peaks before termination.

For a Linux agent:

```bash
free -h

ps -eo pid,comm,rss --sort=-rss | head

journalctl -k --since "1 hour ago" |
  rg -i 'oom|out of memory|killed process'
```

**Then fix the identified cause.**

- **Build concurrency:** Reduce parallel jobs, Maven test forks, or Gradle workers when they exceed available memory.
- **Build heap:** Tune the affected process. Increasing the Jenkins controller heap does not increase Maven’s heap.
- **Container sizing:** Adjust requests and limits based on observed usage.
- **Controller workload:** Avoid loading large files or huge command outputs into Pipeline Groovy variables. Process them on agents and return a small result. Jenkins specifically documents controller-memory problems caused by large `readFile`/JSON-processing operations. :chatgpt-content-reference{index="4"}
- **Suspected leak:** Capture JVM diagnostics, examine retained objects, and investigate relevant plugins or application code.

For Java processes, useful diagnostic options include:

```text
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/diagnostics
```

The directory must be writable and have sufficient space; protect heap dumps because they can contain credentials and application data.

**Example scenario:**

> “A build container has a 2 GiB memory limit, but Maven is configured with a 2 GiB heap and also launches test JVMs. Total memory can exceed the limit. I would measure the processes, reduce heap or test concurrency, and increase container resources only if the workload justifies it.”

---

**4. Two VPCs, A and B, must communicate. What options are available? What if only A should communicate with B?**

First clarify whether the requirement is **general network connectivity** or access to **one particular service**.

| Option | Suitable use case | Main consideration |
|---|---|---|
| VPC Peering | Connecting a small number of VPCs | Non-overlapping CIDRs; peering is not transitive |
| Transit Gateway | Connecting many VPCs and hybrid networks | Central routing, attachment management, and segmentation |
| AWS PrivateLink | Exposing a particular service across VPCs/accounts | Provides service access without general VPC connectivity |
| IPsec VPN through suitable network appliances | Special routing, migration, or encryption requirements | Additional configuration and operational responsibility |

For **VPC peering**, configure:

1. The peering connection and acceptance.
2. Routes from the relevant A subnets to B.
3. Routes from the relevant B subnets to A.
4. Security groups and network ACLs.
5. DNS resolution where required.

An active peering connection alone does not establish usable application connectivity. Routes and security rules must also permit it. :chatgpt-content-reference{index="5"}

For **Transit Gateway**, attach both VPCs, configure their subnet route tables, and configure the relevant Transit Gateway route-table associations and routes. :chatgpt-content-reference{index="6"}

**When only A should initiate connections to B**

Usually, this means:

> A can initiate a connection to B, and B can reply, but B cannot initiate a new connection to A.

With peering or Transit Gateway:

- Allow A’s outbound traffic to the required B service and port.
- Allow B’s inbound traffic from the approved A clients.
- Ensure A has no inbound security-group rule permitting new connections from B.
- Check all attached security groups, because their allow rules combine.
- Keep routes in both directions for request and response traffic.

Security groups are stateful, so replies to permitted connections are automatically allowed. Removing B’s return route would break A’s connections too. Network ACLs are stateless and must allow the necessary return traffic. :chatgpt-content-reference{index="7"}

**PrivateLink is particularly useful for service-specific access.**

For a typical endpoint-service design:

1. B exposes its application through a Network Load Balancer.
2. B creates an endpoint service and permits approved consumers.
3. A creates an interface endpoint.
4. A connects through that endpoint.

This connection allows A to consume B’s service without giving B general access to A’s network. B can return application responses through the established connection. :chatgpt-content-reference{index="8"}

---

**5. Terraform created two instances. State exists locally and in S3. Someone deletes one instance. What happens?**

**First clarify which state backend is active.** A Terraform configuration uses one backend. A leftover local state file and an S3 state object are not automatically synchronized copies. With the S3 backend correctly initialized, Terraform uses the remote state for that working directory and workspace. :chatgpt-content-reference{index="9"}

Assume:

- Both instances are standalone `aws_instance` resources.
- Both remain declared in the configuration.
- One instance is manually terminated.

The next normal `terraform plan` refreshes resource information from AWS, detects the missing instance, and proposes recreating it. For a simple configuration, the result will resemble:

```text
Plan: 1 to add, 0 to change, 0 to destroy.
```

Related resources or outputs may also need changes. Terraform does not continuously monitor and repair resources by itself; a run must occur. :chatgpt-content-reference{index="10"}

**If the deletion was accidental:**

```bash
terraform init

terraform workspace show

terraform plan -out=recovery.tfplan

# Review the plan before executing it.
terraform apply recovery.tfplan
```

The replacement receives a new instance ID. Terraform recreates infrastructure configuration; it does not recover lost application data. Restore deleted data from backups where necessary.

**If the deletion was intentional:**

Update the configuration to describe the intended remaining infrastructure, then review and apply a fresh plan.

A refresh-only operation can reconcile state with observed changes, but it does not change the configuration. If the configuration still declares the missing instance, a later normal plan proposes recreating it. :chatgpt-content-reference{index="11"}

**For team usage, configure a remote backend with locking:**

```hcl
terraform {
  backend "s3" {
    bucket       = "company-terraform-state"
    key          = "prod/compute/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

The bucket must already exist. Enable bucket versioning, restrict access, and use appropriate encryption settings. Current Terraform supports native S3 locking through `use_lockfile`; DynamoDB-based locking is deprecated. :chatgpt-content-reference{index="12"}

When deliberately migrating an existing local backend to S3:

```bash
terraform init -migrate-state
```

Verify the correct source state before migration. State locking coordinates Terraform operations; it does not prevent someone from deleting an instance through the AWS console. :chatgpt-content-reference{index="13"}

---

**6. A Docker image is too large. How would you reduce its size?**

Start by finding what contributes to its size:

```bash
docker image ls myapp

docker image history --no-trunc myapp:latest
```

Then apply the relevant improvements:

| Action | Why it helps |
|---|---|
| Use a multi-stage build | Keeps compilers and build tools out of the runtime image |
| Use a suitable smaller runtime base | Avoids shipping an entire development environment |
| Install only runtime dependencies | Removes unnecessary development and debugging packages |
| Use `.dockerignore` | Excludes unnecessary build-context files and prevents accidental copying |
| Copy specific application artifacts | Avoids copying source repositories, caches, and unrelated files |
| Clean temporary files in the same build step | Prevents temporary content remaining in earlier layers |

Docker recommends separating build and runtime stages and choosing an appropriate minimal base image. :chatgpt-content-reference{index="14"}

**Example: Java multi-stage build**

Assume the Maven project produces an executable `target/app.jar`.

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build

WORKDIR /src

COPY pom.xml .
COPY src ./src

RUN mvn -B package

FROM eclipse-temurin:21-jre

WORKDIR /app

COPY --from=build --chown=10001:10001 \
    /src/target/app.jar /app/app.jar

USER 10001:10001

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

The final image contains the JRE and application artifact. Maven, the JDK build environment, and source files remain in the build stage. Pin approved base-image digests in production. :chatgpt-content-reference{index="15"}

Example `.dockerignore`:

```text
.git
target
*.log
.env
.env.*
secrets/
```

**Important details:**

- Deleting a large file in a later layer does not remove its bytes from an earlier image layer.
- Reducing the number of `RUN` instructions alone does not guarantee a smaller image.
- Alpine may require compatibility changes because of its different C library.
- `docker system prune` frees local storage; it does not shrink an existing image. :chatgpt-content-reference{index="16"}

After optimization, verify application functionality, image size, vulnerabilities, and startup behavior.

---

**7. What tools have you used for CI/CD?**

Answer this using your actual experience. An illustrative stack is:

| Purpose | Example tools | Responsibility |
|---|---|---|
| Source control | GitHub, GitLab, Azure Repos | Code reviews, branch policies, version history |
| CI orchestration | Jenkins | Build, test, and automate release stages |
| Build and testing | Maven, Gradle, npm, pytest | Compile and validate application code |
| Code quality | SonarQube | Quality gates and static analysis |
| Security scanning | Trivy, Gitleaks | Image/dependency scanning and secret detection |
| Container packaging | Docker/BuildKit | Produce deployable container images |
| Artifact storage | ECR, ACR, Artifactory | Store images and build artifacts |
| Deployment | Helm, Argo CD | Package Kubernetes configuration and reconcile deployments |
| Infrastructure | Terraform, Ansible | Provision infrastructure and configure systems |
| Secrets | Secrets Manager, Key Vault, Vault | Manage credentials and application secrets |

Explain how the tools work together rather than only listing names.

**Sample answer to adapt:**

> “Our source code and Jenkinsfiles are stored in Git. Pull requests trigger Jenkins builds, tests, quality checks, and security scans. After merging, the pipeline publishes a versioned image to the registry. We deploy that tested image through our deployment workflow, verify health, and promote the same artifact across environments.”

Keeping the Jenkinsfile in source control makes pipeline changes reviewable and versioned alongside application changes. :chatgpt-content-reference{index="17"}

---

**8. Can we use a Pod as a Jenkins agent? What are the drawbacks?**

**Yes.** The Jenkins Kubernetes plugin can provision temporary Pods as build agents.

Typically:

1. Jenkins receives a build.
2. The Kubernetes plugin creates an agent Pod.
3. An agent container connects to the Jenkins controller.
4. Build steps run in the configured containers.
5. The Pod is removed after the job, according to its retention configuration.

A Pod can contain an agent container and additional containers for tools such as Maven or Node.js. :chatgpt-content-reference{index="18"}

For example, assume Jenkins has a configured Pod template named `maven-agent` containing a container named `maven`:

```groovy
pipeline {
    agent {
        kubernetes {
            inheritFrom 'maven-agent'
        }
    }

    stages {
        stage('Build and Test') {
            steps {
                container('maven') {
                    sh 'mvn -B verify'
                }
            }
        }
    }
}
```

Configure the template with approved images, resource requests and limits, a suitable namespace, and restricted permissions.

**Advantages:**

- Agents are created on demand.
- Builds get clean execution environments.
- Different applications can use different tool versions.
- Idle agent capacity can be reduced.

**Drawbacks and mitigations:**

| Drawback | Mitigation |
|---|---|
| Pod startup and image-pull delays | Small agent images, image caching, sufficient cluster capacity |
| Ephemeral workspace and dependency caches | External artifact storage and controlled dependency caches |
| Pod eviction or node failure interrupts builds | Retry suitable stages and make deployment operations safe to retry |
| Large builds exceed resource limits | Measure usage and size each build container appropriately |
| Agents compete with production workloads | Dedicated node pools or a separate build cluster |
| Container-image builds need a suitable builder | Use an approved isolated builder; avoid exposing the host Docker socket |
| Jenkins or cluster connectivity failures block jobs | Monitor controller availability, API access, DNS, and agent connections |

The Jenkins controller still coordinates Pipeline execution. Moving builds into Pods does not eliminate controller availability or memory requirements. Jenkins agents execute delegated work; they do not replace the controller. :chatgpt-content-reference{index="19"}

---

**9. What are the Kubernetes Service types and their use cases?**

Kubernetes defines four Service types:

| Type | Behavior | Example use case |
|---|---|---|
| **ClusterIP** | Provides a cluster-internal virtual IP; default type | Frontend communicates with an internal API |
| **NodePort** | Exposes a port on node addresses; default range is `30000–32767` | An external load balancer targets node ports |
| **LoadBalancer** | Requests a load balancer through a supported implementation | Exposing an application through a cloud load balancer |
| **ExternalName** | Returns a DNS CNAME for another hostname; no proxying | Giving an external database a service-style DNS alias |

A `LoadBalancer` Service can be internal or internet-facing, depending on its implementation and configuration. :chatgpt-content-reference{index="20"}

**What about a headless Service?**

A headless Service normally uses:

```yaml
spec:
  type: ClusterIP
  clusterIP: None
```

It provides endpoint discovery without allocating a Service virtual IP. A common use case is discovering individual StatefulSet members. It is a Service configuration, not a fifth `type` value. :chatgpt-content-reference{index="21"}

**Ingress and Gateway are separate resources**, commonly used for HTTP routing to Services.

---

**10. What are the different types of Jenkins pipelines?**

There are **two Pipeline syntaxes**:

| Aspect | Declarative Pipeline | Scripted Pipeline |
|---|---|---|
| Structure | Defined structure using `pipeline`, `stages`, and `steps` | Groovy-based programmatic flow, commonly inside `node` |
| Typical use | Standardized team pipelines | Workflows requiring more custom programmatic control |
| Readability | Usually easier to review consistently | Depends more heavily on implementation |
| Example | `pipeline { ... }` | `node('linux') { ... }` |

Both use Jenkins Pipeline steps and can be stored in a `Jenkinsfile`. :chatgpt-content-reference{index="22"}

The Kubernetes-agent example above is Declarative. A simple Scripted example is:

```groovy
node('linux-maven') {
    stage('Checkout') {
        checkout scm
    }

    stage('Build') {
        sh 'mvn -B package'
    }

    stage('Test') {
        sh 'mvn -B verify'
    }
}
```

This assumes an agent with the `linux-maven` label and an SCM-backed job. In a real pipeline, structure Maven goals to avoid unnecessary repeated work.

Also distinguish syntax from **job organization**:

- **Pipeline job:** Runs a configured Pipeline definition.
- **Multibranch Pipeline:** Discovers and manages branch-specific Pipeline jobs.
- **Organization Folder:** Discovers repositories and creates multibranch projects.

A Freestyle job is another Jenkins job type, not another Pipeline syntax.

---

**11. What are the advantages of a multibranch pipeline?**

A Multibranch Pipeline discovers branches containing a `Jenkinsfile` and creates corresponding jobs. Pull-request discovery is available with suitable branch-source plugins. :chatgpt-content-reference{index="23"}

Its main advantages are:

1. **Less manual administration:** New branches do not require manually created jobs.
2. **Versioned pipeline behavior:** Each branch uses its corresponding Jenkinsfile.
3. **Pull-request validation:** Changes can be tested before merging.
4. **Branch-specific behavior:** Different branches can run different deployment stages.
5. **Separate build histories:** Easier investigation of branch-specific failures.
6. **Job cleanup:** Orphaned branch jobs can be handled through configured retention policies.

An example policy is:

| Branch | Pipeline behavior |
|---|---|
| Feature branch / PR | Build, test, scan, optional preview environment |
| `develop` | Deploy to development |
| Release branch | Deploy to staging or UAT |
| `main` or approved release tag | Production promotion with required approval |

Configure webhooks and/or periodic indexing so Jenkins discovers changes.

**Production considerations:** Branch separation is not a security boundary. Restrict production credentials, protect release branches, and prevent untrusted PR builds from accessing privileged credentials. Promote the same tested artifact across environments.

---

**12. If Docker Hub denies access, where else can you push Docker images?**

You can use another container registry that your organization authorizes:

| Registry | Typical fit |
|---|---|
| **Amazon ECR** | AWS workloads and IAM-based access |
| **Azure Container Registry** | Azure-based application delivery |
| **Google Artifact Registry** | Google Cloud workloads and artifact management |
| **GitHub Container Registry — `ghcr.io`** | Images associated with GitHub projects |
| **GitLab Container Registry** | Images associated with GitLab projects |
| **Harbor** | A self-managed registry |

ACR and Artifact Registry provide managed container-image storage; GitHub and GitLab also provide registries integrated with their development platforms. :chatgpt-content-reference{index="24"} Harbor is an option when the organization wants to operate its own registry. :chatgpt-content-reference{index="25"}

**Example: push to Amazon ECR**

Assume an ECR repository named `myapp` already exists in the authenticated AWS account:

```bash
AWS_REGION=ap-south-1

AWS_ACCOUNT_ID=$(
  aws sts get-caller-identity \
    --query Account \
    --output text
)

REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

aws ecr get-login-password --region "$AWS_REGION" |
  docker login \
    --username AWS \
    --password-stdin "$REGISTRY"

docker tag myapp:1.0 "$REGISTRY/myapp:1.0"

docker push "$REGISTRY/myapp:1.0"
```

The caller needs ECR authentication and repository push permissions. ECR authorization tokens are valid for 12 hours. :chatgpt-content-reference{index="26"}

Before changing registries, investigate the Docker Hub failure:

- Is the image tagged with the correct user or organization namespace?
- Does the account have repository write access?
- Is the token expired or missing write permission?
- Is CLI authentication configured correctly? Docker Hub requires a personal access token for CLI access when two-factor authentication is enabled. :chatgpt-content-reference{index="27"}

After moving the image, update Kubernetes image references and configure pull access to the new registry.
