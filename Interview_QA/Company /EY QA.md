Below are interview-ready answers for the **E&Y interview topics**. Replace the bracketed details in the introduction and project explanation with your actual experience.

**1. Introduction**

Keep your introduction to approximately 60–90 seconds. Cover your experience, strongest technical skills, current responsibilities, and one meaningful contribution.

You can adapt this answer:

> “My name is [Name]. I have [X years] of IT experience, including [Y years] working in DevOps and cloud infrastructure.
>
> My main experience is with [AWS/Azure], Terraform, Docker, Kubernetes, and [Jenkins/Azure DevOps/GitHub Actions]. I work on infrastructure provisioning, CI/CD automation, application deployments, and production monitoring.
>
> In my current project, I support [application or business domain]. My responsibilities include [your actual responsibilities], troubleshooting deployment issues, and improving reliability.
>
> One improvement I contributed was [specific automation or reliability improvement], which resulted in [measured or clearly observable outcome].
>
> My strongest areas are [two relevant areas], particularly troubleshooting and automating repeatable operational work.”

Choose technologies you can discuss confidently. If you mention a result such as reducing deployment time, be ready to explain the original problem, your changes, and how you measured the improvement.

**2. Explain your current project**

Start with the business purpose, then explain the architecture and your responsibilities. Listing tools alone does not explain a project.

A useful structure is:

| Area              | What to explain                                                          |
| ----------------- | ------------------------------------------------------------------------ |
| Business purpose  | What the application does and who uses it                                |
| Architecture      | Frontend, backend services, databases, queues, and external integrations |
| Infrastructure    | Cloud platform, networking, Kubernetes or other compute, and storage     |
| Delivery process  | How a code change reaches production                                     |
| Operations        | Monitoring, incident response, security, backups, and recovery           |
| Your contribution | What you personally implemented or operated                              |

For example, **if this matches your experience**, you could say:

> “My project supports a [business application] used by [type of users]. It consists of a frontend, backend services, and a database.
>
> We run the containerized applications on [EKS/AKS]. Incoming requests pass through [the actual load balancer or ingress setup] and reach the appropriate application service.
>
> Terraform provisions our infrastructure. When developers raise a pull request, the pipeline runs tests and security checks. After approval, it builds a versioned container image and publishes it to our registry. We deploy the same image through QA and production using [Helm/Argo CD/your actual deployment tool].
>
> We use Prometheus and Grafana for metrics and [logging platform] for application logs.
>
> My responsibilities include [actual ownership]. A significant issue I worked on was [problem]. I investigated [evidence], implemented [change], and verified the result through [metrics or tests].”

Be prepared for follow-up questions about replica counts, environment separation, database availability, deployment rollback, and an incident you personally handled.

**3. Explain Kubernetes deployments, services, and configurations**

These components solve different problems:

| Component  | Purpose                                                        |
| ---------- | -------------------------------------------------------------- |
| Deployment | Maintains the desired application replicas and manages updates |
| Service    | Provides a stable network endpoint for a group of Pods         |
| ConfigMap  | Stores non-sensitive application configuration                 |
| Secret     | Stores sensitive values such as credentials and tokens         |

**Deployment**

A Deployment manages ReplicaSets, which maintain the required number of Pods. For example, if you request three replicas and a Pod fails, Kubernetes attempts to create a replacement.

Updating the Deployment’s Pod template, such as changing its image, triggers a rollout. With rolling updates:

* `maxSurge` controls how many additional Pods may exist during the update.
* `maxUnavailable` controls how many desired replicas may be unavailable.

A Deployment commonly runs applications whose Pods can be replaced interchangeably. Its selector must match the labels in its Pod template. [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

**Service**

Pod IP addresses can change when Pods are recreated. A Service provides a stable way to reach the application and normally routes traffic to ready endpoints selected by labels.

| Service type   | Typical use                                                               |
| -------------- | ------------------------------------------------------------------------- |
| `ClusterIP`    | Access within the cluster; the default                                    |
| `NodePort`     | Exposes a port on cluster nodes                                           |
| `LoadBalancer` | Requests a load balancer through the available infrastructure integration |
| `ExternalName` | Returns a DNS alias for another hostname                                  |

A headless Service uses `clusterIP: None`; it is not a separate Service type. [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)

**Configuration example**

The following example connects a ConfigMap, Deployment, and Service. Replace the image with your actual image; the application must listen on port `8080` and implement `/ready`.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: orders-config
data:
  APP_ENV: "qa"
  LOG_LEVEL: "info"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders
spec:
  replicas: 3
  selector:
    matchLabels:
      app: orders
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: orders
    spec:
      containers:
        - name: orders
          image: registry.example.com/orders:1.0.0
          ports:
            - name: http
              containerPort: 8080
          envFrom:
            - configMapRef:
                name: orders-config
          readinessProbe:
            httpGet:
              path: /ready
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
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
    - port: 80
      targetPort: http
```

Apply and check it:

```bash
kubectl apply -f orders.yaml
kubectl get deployments,replicasets,pods,services
kubectl rollout status deployment/orders
```

Within the same namespace, another application can call `http://orders`.

ConfigMaps can supply environment variables or mounted files. **Changing a ConfigMap does not automatically trigger a Deployment rollout.** Environment variables require replacement Pods to pick up new values. Mounted configuration generally updates eventually, but the application may still need to reload it. [Kubernetes ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)

For credentials, use Secrets or an external secret management integration. Base64 encoding in a Secret is not encryption; access controls and encryption protection must be configured appropriately. [Kubernetes Secret practices](https://kubernetes.io/docs/concepts/security/secrets-good-practices/)

**4. How do you integrate Grafana with Prometheus?**

**Prometheus collects and stores metrics; Grafana queries those metrics and displays dashboards.**

The integration process is:

1. **Verify Prometheus collection.**
   Confirm that the expected targets appear in Prometheus and are being scraped successfully.

2. **Add a Prometheus data source in Grafana.**
   Open **Connections → Data sources → Add data source → Prometheus**.

3. **Configure the Prometheus URL.**
   Use an address reachable from the Grafana server. For example:

   ```text
   http://prometheus.monitoring.svc.cluster.local:9090
   ```

   This example assumes a Service named `prometheus` in the `monitoring` namespace. Use your actual Service name.

4. **Configure authentication and TLS where required.**

5. **Select Save & test**, then query `up` in Grafana Explore.

6. Create or import dashboards and configure alerts.

When Grafana and Prometheus run in different containers, `localhost` inside Grafana refers to Grafana’s own container network environment. It does not automatically refer to Prometheus. [Grafana Prometheus configuration](https://grafana.com/docs/grafana/latest/datasources/prometheus/configure/)

For repeatable deployments, provision the data source through configuration:

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus.monitoring.svc.cluster.local:9090
    isDefault: true
    editable: false
```

Place this in Grafana’s configured provisioning `datasources` directory or provide the equivalent through your deployment chart. [Grafana provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/)

For dashboards, I would include request rate, error rate, latency, CPU and memory utilization, Pod restarts, and application-specific business metrics.

If dashboards are empty, check data source connectivity, the query’s labels and time range, and whether Prometheus actually contains the expected metrics. Remember that `up = 1` confirms a successful scrape, not complete application health.

**5. Explain Terraform**

Terraform is an infrastructure-as-code tool. You describe the desired infrastructure declaratively, and Terraform calculates the actions needed to move the managed infrastructure toward that configuration.

For example, a configuration can define a VPC, subnets, security groups, a Kubernetes cluster, and a database.

Its main concepts are:

| Concept     | Explanation                                                         |
| ----------- | ------------------------------------------------------------------- |
| Provider    | Integrates Terraform with an API, such as AWS, Azure, or Kubernetes |
| Resource    | Describes an infrastructure object Terraform manages                |
| Data source | Reads information about existing objects                            |
| Variable    | Makes configuration reusable and configurable                       |
| Output      | Exposes selected values from a configuration or module              |
| Module      | Groups related configuration for reuse                              |
| State       | Records the mapping between Terraform addresses and managed objects |

Terraform uses references between resources to infer dependencies. For example, referencing a subnet ID when creating an instance establishes a dependency on that subnet. [Terraform overview](https://developer.hashicorp.com/terraform/intro)

A typical workflow is:

```bash
terraform init
terraform fmt -check -recursive
terraform validate
terraform plan -out=tfplan
```

Review the plan, then apply the saved plan:

```bash
terraform apply tfplan
```

For production, explain the operational controls as well:

* Run changes through a reviewed pipeline.
* Separate state according to environment and ownership boundaries.
* Use encrypted remote state with access controls, versioning, and supported locking.
* Restrict access because state can contain sensitive information.
* Pin dependencies and commit `.terraform.lock.hcl`.

For an S3 backend, current Terraform supports native locking with `use_lockfile = true`; DynamoDB-based locking is deprecated. [Terraform S3 backend](https://developer.hashicorp.com/terraform/language/backend/s3)

A concise interview answer is:

> “I use Terraform to make infrastructure changes repeatable and reviewable. Shared modules provide standard building blocks, while environment configurations supply different inputs. Changes go through a plan review and controlled apply, with remote state and locking.”

Use that wording only for practices you have actually followed.

**6. What is a service mesh?**

A service mesh provides common controls for communication between services, including security, traffic management, and observability.

Consider a checkout application calling payment, inventory, and delivery services. A mesh can apply communication policies consistently across those calls.

Common capabilities include:

* **Mutual TLS:** Encrypt communication and authenticate workload identities.
* **Authorization:** Control which workloads may call other workloads.
* **Traffic routing:** Send selected traffic to a canary version.
* **Resilience controls:** Apply timeouts, bounded retries, and outlier detection.
* **Telemetry:** Collect request latency, error rates, and traffic information.

The **control plane** distributes configuration and policies. The **data plane** handles application traffic according to those policies. [Istio service mesh overview](https://istio.io/latest/about/service-mesh/)

A service mesh does not always require a sidecar in every Pod. In Istio:

| Mode    | How traffic is handled                                                                                              |
| ------- | ------------------------------------------------------------------------------------------------------------------- |
| Sidecar | A proxy runs alongside the application                                                                              |
| Ambient | Node-level `ztunnel` provides the foundational secure transport; optional waypoint proxies provide Layer 7 features |

The choice affects deployment, resource use, and operational behavior. [Istio data plane modes](https://istio.io/latest/docs/overview/dataplane-modes/)

An interview example:

> “For a payment service upgrade, I could route a small percentage of requests to the new version, compare latency and errors, and increase traffic after validation. I could also enforce authenticated, encrypted communication between checkout and payment workloads.”

A mesh adds infrastructure and operational complexity, so I would introduce it when the communication requirements justify it.

Also, workload authentication does not replace application-level user authorization. Retries require care: automatically retrying a non-idempotent payment operation can create duplicate actions unless the application handles them safely.

**7. What is a PodDisruptionBudget?**

A PodDisruptionBudget, or PDB, limits **voluntary disruptions handled through Kubernetes eviction**, such as draining a node for maintenance.

Suppose `orders` has three replicas:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: orders-pdb
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: orders
```

When all three Pods are healthy, this budget permits one to become unavailable through eviction. If one is already unavailable, another healthy Pod’s eviction may be blocked.

You can specify either:

* `minAvailable`: The minimum number or percentage that should remain available.
* `maxUnavailable`: The maximum number or percentage allowed to be unavailable.

Do not specify both in the same PDB.

**A PDB does not guarantee uninterrupted service.** It cannot prevent node failures or application crashes, and direct Pod deletion bypasses its eviction protection. It also does not constrain the Deployment controller’s own rolling update; that uses the Deployment’s rollout settings. [Kubernetes disruptions](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)

Check a budget with:

```bash
kubectl get pdb
kubectl describe pdb orders-pdb
```

In practice, combine it with sufficient replicas, readiness probes, spare capacity, and distribution across failure domains. An overly restrictive budget can block node maintenance.

**8. What is Git squash?**

Squashing combines several commits into one logical commit.

For example, a feature branch might contain commits for implementation, a small correction, and tests. Squashing can present them as one complete feature change.

For the last three commits on your own unpublished branch:

```bash
git rebase -i HEAD~3
```

In the editor:

```text
pick a1b2c3d Add login handler
squash b2c3d4e Fix validation
squash c3d4e5f Add tests
```

Keep the first commit as `pick`; mark subsequent commits as `squash`.

* `squash` combines changes and lets you combine commit messages.
* `fixup` combines changes while discarding the later commit’s message.

This rewrites commit history, so coordinate before rewriting commits other people are using. [Git history rewriting](https://git-scm.com/book/en/v2/Git-Tools-Rewriting-History)

Another method is a squash merge, run from the target branch:

```bash
git merge --squash feature/login
git commit -m "Add login feature"
```

`git merge --squash` prepares the combined changes but does not create the commit itself. The resulting commit is an ordinary single-parent commit; it does not record the feature branch as a merge parent. [Git merge documentation](https://git-scm.com/docs/git-merge)

**9. What is Git rebase?**

Rebase replays commits onto a different base commit.

For example, your feature branch started from an older version of `main`. Rebasing it onto the latest `origin/main` places your feature changes after the newer main-branch commits.

```bash
git fetch origin
git switch feature/login
git rebase origin/main
```

If conflicts occur:

```bash
# Edit the conflicted files, then stage the resolutions.
git add path/to/resolved-file
git rebase --continue
```

To abandon the operation:

```bash
git rebase --abort
```

| Git merge                                             | Git rebase                                        |
| ----------------------------------------------------- | ------------------------------------------------- |
| Combines histories without rewriting existing commits | Replays commits, normally creating new commit IDs |
| May create a merge commit, or fast-forward            | Commonly produces a linear feature history        |
| Suitable for integrating shared branches              | Useful for updating and cleaning a feature branch |

Rebase and squash are different: rebase changes where commits are based, while squash combines commits. Interactive rebase can perform both.

Avoid rewriting shared `main` or release history. If a published feature branch must be rewritten, coordinate with collaborators; `--force-with-lease` is safer than an unconditional force push, but still requires care. [Git rebase documentation](https://git-scm.com/docs/git-rebase)

**10. What is the purpose of Docker?**

Docker packages an application and its dependencies into an image and runs that image as a container.

This helps produce consistent deployments: the same image can be tested in QA and promoted to production, while environment-specific configuration is supplied separately.

| Term       | Meaning                                                   |
| ---------- | --------------------------------------------------------- |
| Dockerfile | Instructions for building an image                        |
| Image      | Packaged application, dependencies, and filesystem layers |
| Container  | A running instance of an image                            |
| Registry   | A service for storing and distributing images             |
| Volume     | Storage whose lifecycle can be independent of a container |

For an application that listens on `0.0.0.0:8080` inside its container:

```bash
docker build -t orders:1.0 .
docker run --rm -p 127.0.0.1:8080:8080 orders:1.0
```

The port mapping makes it accessible at `http://localhost:8080` on the host.

Linux containers share the host’s kernel, while virtual machines run their own guest operating systems. Containers therefore generally have lower overhead, but images still require compatible operating-system and CPU architecture support.

Persistent application data should use volumes or external storage. Data stored only in a container’s writable layer is lost when that container is removed. [Docker overview](https://docs.docker.com/get-started/docker-overview/)

Docker and Kubernetes have complementary purposes: Docker provides image-building and container tooling, while Kubernetes orchestrates workloads across machines. Kubernetes can run Docker-built images through runtimes such as containerd; Docker Engine is not required on every Kubernetes node. [Kubernetes container runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
