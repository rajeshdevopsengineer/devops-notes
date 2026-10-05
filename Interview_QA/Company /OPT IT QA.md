These answers are suitable for a **2+ years DevOps interview**. For questions about your team, project, and experience, replace the placeholders with your actual details.

**1. If databases are in private subnets, how do you deploy applications in Kubernetes?**

**The application can run in Kubernetes while the database remains outside the cluster in a private subnet.** You need private network connectivity between the application pods and the database.

For example, with EKS and RDS:

1. Deploy the application using a Kubernetes Deployment.
2. Ensure the pods can reach the database’s VPC and subnet.
3. Allow the database port in its security group from the appropriate application node or pod security group.
4. Configure the application with the database’s DNS endpoint.
5. Provide credentials through a managed secret store or Kubernetes Secret.
6. Use TLS for database connections and test connectivity from the application environment.

RDS instances have private addresses for communication within their VPC. The database does not need public access for an application in that network to connect. [Amazon Relational Database Service](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_VPC.WorkingWithRDSInstanceinaVPC.html?utm_source=chatgpt.com)

For different VPCs, configure suitable private connectivity, routes, and DNS resolution.

**NAT Gateway is not required for communication within the same VPC.**

If the question means deploying the database itself inside Kubernetes, discuss an appropriate database operator or StatefulSet, persistent volumes, replication, backups, and recovery. A StatefulSet provides stable pod identity; it does not automatically implement database replication. [kubernetes.io](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/?utm_source=chatgpt.com)

**2. What best practices do you follow when creating EC2, RDS, and MongoDB resources?**

Start with common practices:

- Create resources through reviewed infrastructure code.
- Apply consistent names and ownership/environment tags.
- Grant only required permissions.
- Restrict network access.
- Enable encryption, monitoring, and suitable backups.
- Choose capacity based on workload requirements.
- Define maintenance and recovery procedures.

Then explain service-specific practices:

| Resource | Important practices |
|---|---|
| EC2 | Use approved images, patch the OS, attach an IAM role, restrict security groups, and protect administrative access |
| RDS | Use private connectivity, restrict database ports, enable backups, monitor capacity, and use Multi-AZ when availability requirements justify it |
| MongoDB | Enable authentication and RBAC, restrict network exposure, use TLS, and maintain tested backups and suitable replication |

For EC2, explain that AWS secures the underlying infrastructure while you remain responsible for aspects such as the guest OS and application configuration. [Amazon Elastic Compute Cloud](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-security.html?utm_source=chatgpt.com)

For RDS, configure backup and maintenance windows appropriately and verify restoration rather than only checking that backups exist. AWS recommends automatic backups and scheduling them during lower write activity. [Amazon Relational Database Service](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_BestPractices.html?utm_source=chatgpt.com)

For self-managed MongoDB, authentication, role-based permissions, and TLS are explicit security recommendations. [MongoDB Docs](https://www.mongodb.com/docs/manual/administration/security-checklist/?utm_source=chatgpt.com)

**3. How many team members are there?**

Give the actual number and explain your place in the team.

Template:

> “Our project has [total] team members, including [number] developers, [number] QA engineers, and [number] DevOps engineers. I work with developers on builds and deployments, with QA on environment readiness, and with senior engineers on infrastructure and production issues.”

Clarify whether the number refers to the **DevOps team** or the **entire project team**.

Do not memorize an invented team size; interviewers often ask follow-up questions about ownership and collaboration.

**4. Have you written Kubernetes manifests? What kinds?**

Mention the resources you have actually created or maintained.

| Kind | Purpose |
|---|---|
| Deployment | Manage application replicas and updates |
| Service | Provide stable access to application pods |
| ConfigMap | Supply non-secret configuration |
| Secret | Supply sensitive values |
| Ingress | Define HTTP routing handled by an ingress implementation |
| PersistentVolumeClaim | Request persistent storage |
| StatefulSet | Manage workloads requiring stable identities |
| DaemonSet | Run an agent on eligible nodes |
| Job/CronJob | Run one-time or scheduled tasks |

For an application Deployment, be ready to explain:

- Image and version.
- Replica count.
- Labels and selectors.
- Container ports.
- Environment variables.
- Readiness and liveness probes.
- Resource requests and limits.

A useful answer, if accurate, is:

> “I mainly worked on Deployment, Service, ConfigMap, and Ingress manifests. I updated image versions, configured environment-specific values, and verified that Service selectors matched the pod labels.”

**5. Explain Terraform workspaces and modules**

A **CLI workspace** provides a separate instance of Terraform state for the same working directory.

For example:

```bash
terraform workspace list
terraform workspace new dev
terraform workspace show
terraform plan -var-file=dev.tfvars
```

Separate workspaces can manage separate copies of infrastructure, but you still need appropriate names and input values.

**A workspace does not automatically switch AWS accounts or provide access isolation.** For environments requiring different credentials and access controls, separate root configurations and backends are usually more appropriate. [HashiCorp Developer](https://developer.hashicorp.com/terraform/cli/workspaces?utm_source=chatgpt.com)

A **module** packages reusable Terraform configuration.

For example:

```hcl
module "network" {
  source = "./modules/vpc"

  environment = var.environment
  cidr_block  = var.vpc_cidr
}
```

The module might create a VPC, subnets, and route tables. Different environments can call it with different inputs.

Terraform calls the current configuration the **root module**; modules invoked through `module` blocks are **child modules**. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/modules?utm_source=chatgpt.com)

| Concept | Main purpose |
|---|---|
| Workspace | Separate state |
| Module | Reuse configuration |

**6. What monitoring tools have you used?**

Name your actual tools and explain what you did with them.

Common examples:

| Tool | Purpose |
|---|---|
| Prometheus | Collect and query metrics |
| Grafana | Visualize metrics and other data |
| Alertmanager | Group and route Prometheus alerts |
| CloudWatch | Monitor AWS resources using metrics, logs, and alarms |
| Loki or ELK/EFK | Collect, search, and investigate logs |

Grafana can query Prometheus through its Prometheus data source. CloudWatch provides monitoring capabilities for AWS resources and applications. [Grafana documentation](https://grafana.com/docs/grafana/latest/datasources/prometheus/?utm_source=chatgpt.com)

Explain your contribution—for example:

- Creating dashboards.
- Configuring alerts.
- Checking pod restarts and resource usage.
- Investigating logs during incidents.
- Validating that applications expose the required metrics.

Avoid saying you implemented the entire monitoring platform if you mainly maintained dashboards and alerts.

**7. Why use Prometheus, and where did you deploy it?**

Prometheus collects metrics as time-series data and supports queries and alerting rules. It normally scrapes HTTP metrics endpoints using a pull model. [Prometheus](https://prometheus.io/docs/introduction/overview/?utm_source=chatgpt.com)

It is useful for monitoring:

- Request rate and error rate.
- Application latency.
- CPU and memory usage.
- Pod restarts.
- Node availability.
- Desired versus available application replicas.

A common Kubernetes deployment places monitoring components in a namespace such as `monitoring`.

An Operator-managed setup can include:

- Prometheus.
- Alertmanager.
- Exporters.
- Grafana.
- Persistent storage for metrics.

`ServiceMonitor` and `PodMonitor` resources help define scrape targets, while `PrometheusRule` resources define recording and alerting rules. [Prometheus Operator](https://prometheus-operator.dev/docs/getting-started/introduction/?utm_source=chatgpt.com)

Interview template:

> “We deployed Prometheus in [actual location]. I used its metrics through Grafana dashboards and worked on alerts for [actual conditions].”

**8. How many microservices are in your project? Name some**

Use the actual service count and business domain.

Template:

> “Our application has approximately [number] microservices. Examples include [service names]. Each service has its own deployment configuration, and services communicate using APIs or messaging.”

For an **illustrative e-commerce application**, service names could include:

- Authentication.
- Product catalog.
- Orders.
- Payments.
- Inventory.
- Notifications.

These are examples, not facts about your project. Be ready to explain what two or three of your actual services do.

**9. Which microservices did you work on?**

Explain both the service names and your responsibilities.

Template:

> “I supported deployment and operations for [service names]. My responsibilities included maintaining pipeline configuration, building container images, updating Kubernetes configuration, checking deployment health, and troubleshooting failures.”

For each service, know:

- Its purpose.
- Programming language.
- Deployment environment.
- Main dependencies.
- Health endpoint.
- Logs and important metrics.

Describe your DevOps contribution clearly. Supporting a service’s deployment does not necessarily mean developing its business logic.

**10. What did you learn in your previous company?**

Choose a few genuine lessons and connect them to practical work.

Examples include:

- Linux troubleshooting and log analysis.
- Git collaboration and pull-request reviews.
- Build and deployment automation.
- Docker image creation.
- Kubernetes deployment troubleshooting.
- Infrastructure as code.
- Monitoring and incident communication.

A template you can adapt:

> “I developed stronger skills in [actual areas]. I learned to investigate failures systematically, keep infrastructure changes in version control, and validate deployments after release. I also improved how I communicate issues and coordinate with development and QA teams.”

Support this with one real task you completed or helped resolve.

**11. How do you containerize an application?**

I would follow these steps:

1. **Understand the application:** Identify its runtime, dependencies, startup command, ports, and configuration.
2. **Build the artifact:** For Java, this may be an executable JAR.
3. **Write the Dockerfile:** Select a suitable base image and copy the required files.
4. **Externalize configuration:** Supply environment-specific settings and credentials at runtime.
5. **Build and test:** Verify startup, connectivity, logs, and shutdown.
6. **Scan the image:** Address unacceptable vulnerabilities.
7. **Publish:** Push a versioned image to the registry.
8. **Deploy:** Configure replicas, resources, probes, and secrets in the target environment.

A multi-stage build allows build tools to remain in the builder stage while the final image contains only the required runtime and application output. [Docker Docs](https://docs.docker.com/build/building/multi-stage/?utm_source=chatgpt.com)

Also keep persistent application data outside the container’s writable layer.

**12. Write a Dockerfile for a Java application**

This example assumes:

- Java 21.
- Maven with a committed Maven Wrapper.
- A Spring Boot application producing an executable `target/app.jar`.

```dockerfile
# syntax=docker/dockerfile:1

FROM eclipse-temurin:21-jdk-jammy AS build

WORKDIR /build

COPY .mvn/ .mvn/
COPY mvnw pom.xml ./

RUN chmod +x mvnw

RUN --mount=type=cache,target=/root/.m2 \
    ./mvnw -B dependency:go-offline

COPY src/ src/

RUN --mount=type=cache,target=/root/.m2 \
    ./mvnw -B clean package


FROM eclipse-temurin:21-jre-jammy AS runtime

WORKDIR /app

RUN groupadd --gid 10001 app \
    && useradd --uid 10001 --gid app --no-create-home app

COPY --from=build --chown=app:app \
    /build/target/app.jar /app/app.jar

USER app

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

The JDK and JRE image tags shown are listed in the official Eclipse Temurin image definitions. [raw.githubusercontent.com](https://raw.githubusercontent.com/docker-library/official-images/master/library/eclipse-temurin?utm_source=chatgpt.com)

Configure Maven’s output name inside the existing `<build>` section:

```xml
<finalName>app</finalName>
```

The application must already be configured to produce an executable JAR.

Example `.dockerignore`:

```text
.git
target
.idea
*.log
.env
.env.*
```

Build and run locally:

```bash
docker build -t java-app:1.0 .

docker run --rm \
  -p 127.0.0.1:8080:8080 \
  java-app:1.0
```

Important points:

- The builder uses a JDK; the final image uses a JRE.
- Copying build configuration before source helps reuse dependency layers.
- The Maven cache mount avoids repeated dependency downloads. [Docker Docs](https://docs.docker.com/build/cache/optimize/?utm_source=chatgpt.com)
- The application runs as a non-root user.
- Exec-form `ENTRYPOINT` starts Java directly.
- `EXPOSE` documents the port; `-p` publishes it.
- For controlled releases, pin approved base-image digests and update them through your patching process.

**13. Which tool did you work on for CI/CD?**

Name the tool you actually used and explain your contribution.

For Jenkins, a suitable template is:

> “I worked with Jenkins pipelines defined in Jenkinsfiles. I maintained build steps, configured repository integration, checked failed stages, and supported deployments to our environments.”

Be ready to explain:

- How the job is triggered.
- Where the Jenkinsfile is stored.
- Which agent executes the build.
- How credentials are supplied.
- Where artifacts are published.
- How deployment results are checked.

Jenkins supports pipelines stored as code in source control, with Declarative or Scripted syntax. [jenkins.io](https://www.jenkins.io/doc/book/pipeline/?utm_source=chatgpt.com)

If you used Azure Pipelines or GitHub Actions, describe that implementation instead.

**14. What type of pipeline did you work on?**

There are two useful ways to answer.

By **purpose**:

| Type | Typical work |
|---|---|
| CI pipeline | Build, test, quality checks, scanning, and publishing |
| CD pipeline | Deployment, approvals, smoke tests, and release verification |

By **Jenkins implementation**:

- **Declarative Pipeline:** Structured syntax using `pipeline`, `stages`, and `steps`.
- **Scripted Pipeline:** Groovy-based pipeline logic.
- **Multibranch Pipeline:** A job arrangement that discovers branches and their Jenkinsfiles.

Multibranch is a job model, rather than a third pipeline syntax. Jenkins documents Declarative and Scripted syntax separately. [www.jenkins.io](https://www.jenkins.io/doc/book/pipeline/syntax/?utm_source=chatgpt.com)

Example answer, if accurate:

> “I worked mainly on Declarative Jenkins pipelines. CI handled validation and image publication, while CD deployed the tested image to QA and production with the required checks and approvals.”

**15. Do you have any questions for us?**

Choose two or three questions that help you understand the role:

- “What would you expect me to achieve during my first three months?”
- “Which cloud platform and deployment tools does the team currently use?”
- “How are responsibilities divided between DevOps, development, and operations?”
- “What are the main deployment or reliability challenges the team is addressing?”
- “How does the team support learning, code reviews, and incident handling?”
