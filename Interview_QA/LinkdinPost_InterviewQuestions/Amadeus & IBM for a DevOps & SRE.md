This is a great collection of real-world DevOps/SRE interview questions. Let me give you solid, practical answers for each section that you can use to prepare. I'll keep them at the depth an interviewer would expect — enough to show you understand the *why*, not just the *what*.

## SRE Scenario-Based Questions

**Production down at 2 AM — what steps?**
Acknowledge the alert and start a timeline. Establish the blast radius first: is it total or partial? Check your dashboards (error rate, latency, saturation, traffic) and recent changes — deploys, config pushes, infra changes — since *something changed* is the most common cause. If a recent deploy correlates, roll back immediately rather than debugging forward; restoring service beats finding root cause at 2 AM. Communicate status to stakeholders early and often. Once stable, capture evidence for the postmortem. The key principle: **mitigate first, diagnose later.**

**How do you reduce MTTR?**
MTTR breaks into detect → diagnose → mitigate → resolve, so attack each. Faster detection through good alerting on symptoms (SLO burn) not just causes. Faster diagnosis through good observability — correlated logs, metrics, traces and runbooks tied to alerts. Faster mitigation through one-click rollbacks, feature flags, and well-rehearsed runbooks. Practice with game days/chaos drills so the team isn't learning during a real incident. Automate the repetitive recovery steps.

**Users experiencing high latency — troubleshoot?**
Work the request path layer by layer. Confirm scope (all users or a subset/region/endpoint?). Check the four golden signals. Walk the stack: LB → app → cache → database → downstream dependencies. Look at p50 vs p99 — a high p99 with normal p50 points to tail latency (GC pauses, lock contention, a slow dependency). Check resource saturation (CPU, memory, connection pools, thread pools). Distributed tracing is your best friend here to find *which* hop is slow. Common culprits: slow DB queries, connection pool exhaustion, an overloaded downstream service, or a cache that started missing.

**Microservice intermittently failing — debug?**
"Intermittent" usually means it's load-, instance-, or dependency-correlated. Check whether failures map to specific pods/instances (one bad node?), specific times (cron, traffic spikes, batch jobs), or a flaky dependency. Look at logs around failure timestamps, check for resource limits being hit (OOM kills, throttling), connection pool exhaustion, and timeout/retry storms. Verify health checks aren't flapping. Correlate with deploys and downstream dependency health.

**Design a highly available and reliable system?**
Eliminate single points of failure through redundancy at every layer — multi-AZ (and multi-region for higher tiers), load balancing, replicated/failover databases. Design for graceful degradation (circuit breakers, timeouts, retries with backoff, bulkheads). Make services stateless where possible so they scale and recover easily. Use health checks and auto-replacement of unhealthy instances. Define SLOs and design to them rather than chasing 100%. Test failure paths regularly.

**Monitoring and alerting in production?**
Layer it: infrastructure metrics, application metrics, and business/SLO metrics. Use the Four Golden Signals (latency, traffic, errors, saturation) or RED/USE methods. Collect metrics (Prometheus), visualize (Grafana), centralize logs (ELK/Loki), and add distributed tracing (Jaeger/Tempo). Alert on **symptoms that affect users / SLO burn rate**, not every metric. Every alert should be actionable and have a runbook.

**Alert fatigue?**
Alert on symptoms, not causes. Tie alerts to SLO burn rates so you only page when users are actually affected. Set sane thresholds, use multi-window/multi-burn-rate alerting, group/deduplicate related alerts, and route by severity (page vs ticket vs dashboard). Regularly review and prune noisy or non-actionable alerts — if an alert never requires action, delete it. Auto-remediate what you can.

**Root cause analysis / postmortem?**
Run a **blameless** postmortem. Build a precise timeline (detection → mitigation → resolution). Use techniques like the 5 Whys to get past the proximate cause to systemic ones. Distinguish trigger from root cause. Document impact, what went well, what didn't, and produce concrete, owned, time-bound action items. Track them to completion — a postmortem with no follow-through is theater.

## Jenkins

**Build and deploy pipeline (declarative):**

```groovy
pipeline {
    agent any
    environment {
        REGISTRY = 'registry.example.com'
        IMAGE    = "${REGISTRY}/myapp:${BUILD_NUMBER}"
    }
    stages {
        stage('Checkout') {
            steps { checkout scm }
        }
        stage('Build') {
            steps { sh 'mvn clean package -DskipTests' }
        }
        stage('Test') {
            steps { sh 'mvn test' }
            post { always { junit '**/target/surefire-reports/*.xml' } }
        }
        stage('Docker Build & Push') {
            steps {
                script {
                    docker.withRegistry("https://${REGISTRY}", 'registry-creds') {
                        def img = docker.build("${IMAGE}")
                        img.push()
                        img.push('latest')
                    }
                }
            }
        }
        stage('Deploy') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig']) {
                    sh "kubectl set image deployment/myapp myapp=${IMAGE}"
                    sh "kubectl rollout status deployment/myapp"
                }
            }
        }
    }
    post {
        failure { echo 'Pipeline failed — notify team' }
        always  { cleanWs() }
    }
}
```

**Securely manage credentials?**
Use the Jenkins Credentials Store (and the Credentials Binding plugin) — never hardcode secrets. Reference them via the `credentials()` helper or `withCredentials` block, which injects them as masked environment variables. For production-grade setups, integrate an external secrets manager (HashiCorp Vault, AWS Secrets Manager) so Jenkins fetches secrets at runtime rather than storing them. Scope credentials to specific folders/jobs and avoid echoing them in logs.

**Declarative vs Scripted?**
Declarative is the newer, opinionated, structured syntax (`pipeline { }` block) — easier to read, validated upfront, with built-in `post`/`when`/`environment` directives. Scripted is full Groovy (`node { }`) — more flexible and powerful for complex logic but harder to maintain. Most teams use Declarative and drop into `script { }` blocks when they need Groovy flexibility.

**Handle failures?**
Use the `post` section (`failure`, `unstable`, `always`) for cleanup and notifications. Use `retry(n)` for flaky steps, `timeout()` to prevent hangs, and `try/catch` in scripted blocks. Use `catchError` or `unstable` to continue when appropriate. Make notifications (Slack/email) part of failure handling, and design pipelines to be idempotent so reruns are safe.

## Docker

**Dockerfile (Node example):**

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 3000
USER node
CMD ["node", "server.js"]
```

**CMD vs ENTRYPOINT?**
`ENTRYPOINT` sets the executable that always runs; `CMD` provides default arguments (or a default command) that can be *overridden* at `docker run`. A common pattern: `ENTRYPOINT ["python", "app.py"]` with `CMD ["--default-flag"]`, so the container always runs the app but args are easily swapped. If you only use `CMD`, the whole thing is overridable. Use `ENTRYPOINT` when the container should behave like a specific executable.

**Reduce image size?**
Use minimal base images (alpine, distroless, slim). Use multi-stage builds to leave build tools behind. Minimize layers and combine `RUN` commands. Use `.dockerignore`. Clean up package caches in the same layer (`apt-get clean`, `rm -rf /var/lib/apt/lists/*`). Install only production dependencies.

**Multi-stage build?**
A Dockerfile with multiple `FROM` stages where you build/compile in one stage and copy only the final artifacts into a clean, minimal runtime stage. This keeps compilers, dev dependencies, and source out of the final image — much smaller and more secure:

```dockerfile
FROM golang:1.22 AS build
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o /app

FROM gcr.io/distroless/static
COPY --from=build /app /app
ENTRYPOINT ["/app"]
```

## Kubernetes

**Deployment YAML:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels: { app: myapp }
spec:
  replicas: 3
  selector:
    matchLabels: { app: myapp }
  template:
    metadata:
      labels: { app: myapp }
    spec:
      containers:
        - name: myapp
          image: registry.example.com/myapp:1.0.0
          ports:
            - containerPort: 8080
          resources:
            requests: { cpu: "100m", memory: "128Mi" }
            limits:   { cpu: "500m", memory: "256Mi" }
          readinessProbe:
            httpGet: { path: /healthz, port: 8080 }
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet: { path: /healthz, port: 8080 }
            initialDelaySeconds: 15
            periodSeconds: 20
```

**Troubleshoot CrashLoopBackOff?**
The container starts, crashes, and Kubernetes keeps restarting it with backoff. Check `kubectl describe pod` (events) and `kubectl logs <pod> --previous` (logs from the crashed instance — this is the key one). Common causes: application error/misconfig, missing env vars or secrets, failing liveness probe killing a healthy-but-slow app, OOMKilled (check resource limits), bad command/entrypoint, or a dependency that isn't reachable on startup.

**Why Pending?**
The scheduler can't place the pod. Usual reasons: insufficient cluster resources (CPU/memory requests can't be satisfied — add nodes or lower requests), no node matches `nodeSelector`/affinity/taints-tolerations, unbound PVC (no matching PersistentVolume), or hitting quotas. `kubectl describe pod` shows the exact scheduling failure reason.

**Deployment strategies?**
*Rolling* (default): gradually replace old pods with new — zero downtime, but both versions run briefly. *Blue-Green*: stand up the new version (green) alongside old (blue), switch traffic all at once — instant rollback, but double the resources. *Canary*: route a small % of traffic to the new version, watch metrics, gradually increase — lowest risk, catches issues with real traffic, but more complex (often needs a service mesh or ingress weighting).

**Service vs Ingress?**
A Service provides stable networking/load balancing to a set of pods (ClusterIP internal, NodePort/LoadBalancer external) and operates at L4. Ingress operates at L7 (HTTP/HTTPS) and provides host/path-based routing, TLS termination, and a single entry point for many services — but it needs an Ingress Controller (nginx, Traefik). Think: Service exposes pods; Ingress routes external HTTP traffic to Services.

**Auto-scaling?**
Three types: **HPA** scales pod *replicas* based on metrics (CPU/memory or custom). **VPA** adjusts pod resource requests/limits. **Cluster Autoscaler** adds/removes *nodes* when pods can't be scheduled or nodes are underutilized. HPA is the most common — it watches metrics via the metrics-server and adjusts replica count to keep utilization near a target.

## Terraform

**Provision infrastructure (EC2 example):**

```hcl
provider "aws" {
  region = "eu-west-1"
}

resource "aws_instance" "web" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.micro"
  tags = {
    Name = "web-server"
  }
}
```

**State and why it matters?**
Terraform state (`terraform.tfstate`) maps your config to real-world resources — it's how Terraform knows what it already created. It tracks resource metadata and dependencies and enables `plan` to compute diffs. Without it, Terraform can't tell what exists. It can contain secrets, so it must be stored securely and never committed to git.

**Remote backend?**
Storing state remotely (S3, Terraform Cloud, Azure Blob, GCS) instead of locally. This enables team collaboration (shared single source of truth), state locking, encryption at rest, and versioning/backup. Example: S3 bucket for state + DynamoDB table for locking.

**State locking?**
Prevents concurrent `apply` operations from corrupting state. When one person runs apply, the state is locked so others can't simultaneously modify it. With the S3 backend you use a DynamoDB table for locks; Terraform Cloud handles it natively. If a lock gets stuck (crashed run), `terraform force-unlock` releases it.

**Infrastructure drift?**
Drift is when real infrastructure diverges from your Terraform config — usually someone made a manual change in the console. Detect it with `terraform plan` (or `terraform plan -refresh-only`), which shows the diff. Manage it by either reverting (re-apply to restore desired state) or importing/updating config to match reality. Prevent it by enforcing IaC-only changes via policy and restricting console write access.

## AWS

**Highly available 3-tier architecture?**
- **Web/presentation tier**: Route 53 → CloudFront (CDN) → ALB across multiple AZs → web servers in an Auto Scaling Group.
- **App/logic tier**: private subnets, ASG across AZs, reachable only from the web tier.
- **Data tier**: RDS Multi-AZ (with read replicas if needed) or DynamoDB; in private subnets.

Spread across at least two AZs, use private subnets for app/data, security groups for tier-to-tier isolation, NAT gateways for outbound, and ASGs + health checks for self-healing.

**ALB vs NLB vs API Gateway?**
- **ALB** — L7 (HTTP/HTTPS), content/path/host-based routing, good for web apps and microservices.
- **NLB** — L4 (TCP/UDP), ultra-high performance and static IPs, handles millions of requests with very low latency; use for non-HTTP or extreme throughput.
- **API Gateway** — fully managed API front door with auth, throttling, rate limiting, request transformation, and versioning; great for serverless/Lambda APIs. More features but higher per-request cost.

**Auto Scaling in AWS?**
An Auto Scaling Group maintains a desired number of EC2 instances across AZs, replacing unhealthy ones automatically. Scaling policies (target tracking, step, or scheduled) add/remove instances based on metrics like CPU or request count. Combined with an ALB it gives elastic, self-healing capacity.

**Cost optimization?**
Right-size instances; use Auto Scaling to avoid over-provisioning. Use Savings Plans/Reserved Instances for steady workloads and Spot for fault-tolerant/batch. Use S3 lifecycle policies and storage tiers (IA, Glacier). Delete unattached EBS volumes and idle resources. Use the right tool (serverless for spiky loads). Monitor with Cost Explorer, set budgets/alerts, and tag resources for accountability.

**S3 vs RDS vs DynamoDB?**
- **S3** — object storage for files, backups, static assets, data lakes. Not a database.
- **RDS** — managed relational DB (Postgres/MySQL/etc.) for structured data with complex queries, joins, transactions.
- **DynamoDB** — managed NoSQL key-value/document store for massive scale, single-digit-ms latency, predictable access patterns; no joins/complex queries.

## Python

**Automation script (disk usage alert):**

```python
import shutil
import sys

THRESHOLD = 80  # percent

def check_disk(path="/"):
    total, used, _ = shutil.disk_usage(path)
    percent = used / total * 100
    if percent > THRESHOLD:
        print(f"WARNING: {path} is {percent:.1f}% full")
        sys.exit(1)
    print(f"OK: {path} at {percent:.1f}%")

if __name__ == "__main__":
    check_disk()
```

**Parse logs?**
Read line by line (memory-efficient for large files), use `re` for pattern matching, and `collections.Counter`/`defaultdict` to aggregate:

```python
import re
from collections import Counter

pattern = re.compile(r'(\d+\.\d+\.\d+\.\d+).*"(?:GET|POST) (\S+).*" (\d{3})')
status_counts = Counter()

with open("access.log") as f:
    for line in f:
        m = pattern.search(line)
        if m:
            ip, path, status = m.groups()
            status_counts[status] += 1
            if status.startswith("5"):
                print(f"5xx error: {ip} {path}")

print("Status code distribution:", dict(status_counts))
```

**Call APIs?**

```python
import requests

resp = requests.get(
    "https://api.example.com/health",
    headers={"Authorization": "Bearer TOKEN"},
    timeout=10,
)
resp.raise_for_status()
data = resp.json()
print(data["status"])
```

Use `timeout`, `raise_for_status()`, retries (via `requests.adapters.HTTPAdapter` with `urllib3.Retry`), and never hardcode tokens.

**Common DevOps automation use cases?**
Log parsing/analysis, health checks and monitoring scripts, calling cloud APIs (boto3 for AWS), provisioning/configuration glue, CI/CD scripting, automated backups and cleanup, parsing/generating YAML/JSON configs, ChatOps bots, scheduled cron jobs, and report generation.

---

A few tips for the actual interview based on how these were framed:

For scenario questions, **always lead with "restore service first, root-cause later"** — that mindset is what separates SREs from generalists. For the "write X" questions, narrate your choices ("I'm adding readiness probes because…") rather than just dumping code. And for any "difference between" question, give a one-line distinction plus *when you'd choose each*.

Want me to do mock follow-up questions on any section, or go deeper on a specific area (the SLO/error-budget math, a real RCA walkthrough, or Terraform module structure tend to be where interviewers dig)?

Another round of questions — these look like a more conversational technical screen. Let me give you answers framed the way you'd actually *say* them in an interview, since several of these ("explain," "have you worked on") expect you to speak from experience. I'll mark where you should personalize with your own project details.

## 1. Self-introduction

Keep it to ~60–90 seconds with a clear arc: who you are → what you do → relevant experience → why this role. A template:

> "I'm a DevOps/SRE engineer with X years of experience, primarily working with AWS, Kubernetes, Terraform, and CI/CD pipelines. In my current role at [company], I'm responsible for [managing production K8s clusters / building CI/CD pipelines / infrastructure automation]. A few things I'm proud of: [reduced deployment time by X / improved system reliability / automated Y]. I enjoy the SRE side especially — building reliable systems and reducing toil through automation. I'm excited about this role at Amadeus because [reason tied to their scale/travel-tech domain]."

Tailor the achievements to be concrete and measurable — interviewers remember numbers.

## 2. Kubernetes architecture

Split it into **Control Plane** and **Worker Nodes**.

**Control Plane** (the brain):
- **API Server** — the front door; everything talks to it (kubectl, controllers, nodes). Validates and persists state.
- **etcd** — distributed key-value store holding the entire cluster state. The single source of truth.
- **Scheduler** — decides which node a new pod runs on based on resources, affinity, taints.
- **Controller Manager** — runs control loops (deployment, replicaset, node controllers) that drive actual state toward desired state.

**Worker Nodes** (where workloads run):
- **kubelet** — agent on each node; talks to the API server, ensures containers in pods are running and healthy.
- **kube-proxy** — handles networking/routing rules so Services work.
- **Container runtime** — containerd/CRI-O that actually runs containers.

The core idea to verbalize: *"Kubernetes is a declarative, control-loop-driven system — you declare desired state, and controllers continuously reconcile reality toward it."*

## 3. Have you worked on auto-scaling? (experience question)

Answer **yes** with a concrete story. Example structure:

> "Yes. In my project we used **HPA** to scale our [API service] based on CPU and custom metrics. During peak traffic our pods would scale from 3 to ~12 replicas. We also ran the **Cluster Autoscaler** so when HPA couldn't schedule pods due to resource shortage, new nodes were added automatically. One thing I learned was the importance of setting proper resource *requests* — HPA's calculations depend on them, and we initially had them set too low which caused premature scaling."

That last "lesson learned" detail makes it credible.

## 4. HPA vs VPA

- **HPA (Horizontal Pod Autoscaler)** — scales the *number of pod replicas* up/down based on metrics (CPU/memory/custom). Scaling **out**.
- **VPA (Vertical Pod Autoscaler)** — adjusts the *resource requests/limits* of pods (CPU/memory per pod). Scaling **up**.

One-liner to remember: **HPA adds more pods; VPA makes pods bigger.** Note that HPA and VPA on the same metric (CPU) can conflict, so they're usually not used together on the same resource. HPA is far more common in production.

## 5. What happens when a Pod crashes?

Walk through the chain:

When a container in a pod exits/crashes, the **kubelet** detects it (via the container runtime and liveness probes). Based on the pod's `restartPolicy` (default `Always`), kubelet restarts the container. If it keeps crashing, you get **CrashLoopBackOff** — kubelet backs off with exponentially increasing delays (10s, 20s, 40s… up to 5 min) before each restart attempt.

Important distinction: it's the *container* that restarts, not a new pod — the pod keeps its name and IP. If the entire *node* dies, the controller (ReplicaSet/Deployment) creates a *replacement* pod on a healthy node to maintain the desired replica count. That's the deployment controller doing reconciliation, separate from kubelet's container restarts.

## 6. How do you explore applications running on K8s clusters?

This is asking about your day-to-day debugging/observability toolkit:

> "Mostly through `kubectl` — `kubectl get pods/deployments/services` to see what's running and their status, `kubectl describe` for events and config, and `kubectl logs` for application output. For exploring inside a container I use `kubectl exec -it <pod> -- /bin/sh`. For a higher-level view we used dashboards — Prometheus + Grafana for metrics, and the Kubernetes Dashboard / Lens / k9s for visual cluster exploration. For tracing requests across microservices we used [Jaeger/distributed tracing]. Centralized logging was through [ELK/Loki]."

Mentioning `k9s` or `Lens` signals real hands-on usage. Useful commands to namedrop: `kubectl get all -n <namespace>`, `kubectl top pods` (resource usage), `kubectl get events --sort-by=.lastTimestamp`.

## 7. Explain Terraform modules in your project (experience question)

Describe the *structure and reasoning*, not just the definition:

> "We structured our infra into reusable modules so we didn't repeat code across environments. For example, we had a **VPC module** (subnets, route tables, NAT, IGW), an **EC2/compute module**, an **RDS module**, and an **EKS module**. Each module had its `variables.tf`, `main.tf`, and `outputs.tf`. Then per environment (dev/staging/prod) we had root configs that called these modules with different variable values — so the same module produced a small instance in dev and a large multi-AZ setup in prod. We kept state in an **S3 backend with DynamoDB locking**, with separate state files per environment."

The key concepts to land: **reusability, DRY, environment parity via variables, and separation of state.**

## 8. Sample Terraform module to launch an EC2 instance

A proper module has variables, resource, and outputs:

```hcl
# modules/ec2/variables.tf
variable "instance_type" {
  type    = string
  default = "t3.micro"
}
variable "ami_id" {
  type = string
}
variable "subnet_id" {
  type = string
}
variable "name" {
  type    = string
  default = "app-server"
}

# modules/ec2/main.tf
resource "aws_instance" "this" {
  ami           = var.ami_id
  instance_type = var.instance_type
  subnet_id     = var.subnet_id
  tags = {
    Name = var.name
  }
}

# modules/ec2/outputs.tf
output "instance_id" {
  value = aws_instance.this.id
}
output "private_ip" {
  value = aws_instance.this.private_ip
}
```

And how you'd **call** it (this part impresses interviewers — it shows you understand modules are *consumed*):

```hcl
# main.tf (root)
module "web_server" {
  source        = "./modules/ec2"
  ami_id        = "ami-0abcdef1234567890"
  instance_type = "t3.medium"
  subnet_id     = module.vpc.public_subnet_id
  name          = "web-prod"
}
```

## 9. Instance types you're aware of / have used

Explain the **families** then state what you used:

- **General Purpose (T, M)** — balanced CPU/memory. T-series (t3/t4g) are burstable, cheap, good for dev/web. M-series for steady general workloads.
- **Compute Optimized (C)** — high CPU; batch processing, gaming servers, HPC.
- **Memory Optimized (R, X)** — large RAM; databases, in-memory caches, analytics.
- **Storage Optimized (I, D)** — high disk I/O; data warehouses, NoSQL.
- **Accelerated/GPU (P, G)** — ML training, graphics.

> "I've mostly used **t3** instances for dev/test and general workloads because of the cost and burstable nature, and **m5** for steadier production workloads. For our [database/cache] we used **r5** memory-optimized instances."

Say what you actually used — don't claim GPU instances if you've never touched them.

## 10. Stages of your pipeline (experience question)

Describe a real flow and *why* each stage exists:

> "Our CI/CD pipeline had roughly these stages:
> 1. **Checkout/SCM** — pull code on commit/PR trigger.
> 2. **Build** — compile and package the app.
> 3. **Unit tests + code quality** — run tests, SonarQube scan, fail fast on quality gates.
> 4. **Security scan** — image/dependency scanning (Trivy/Snyk).
> 5. **Docker build & push** — build image, tag with build number/commit SHA, push to [ECR].
> 6. **Deploy to staging** — apply to the staging cluster, run smoke/integration tests.
> 7. **Manual approval** — gate before prod.
> 8. **Deploy to prod** — rolling deployment, then verify rollout health."

The "tag with commit SHA," "fail fast," and "manual approval gate before prod" details show maturity.

## 11. Simple Dockerfile + explanation

```dockerfile
FROM node:20-alpine          # small base image
WORKDIR /app                 # set working directory
COPY package*.json ./        # copy deps manifest first (layer caching)
RUN npm ci --only=production  # install only prod dependencies
COPY . .                     # copy the rest of the source
EXPOSE 3000                  # document the port
USER node                    # run as non-root for security
CMD ["node", "server.js"]    # start command
```

Things to explain while writing it:
- **Why copy `package.json` before the rest?** Layer caching — if source changes but deps don't, Docker reuses the cached `npm install` layer, speeding up builds.
- **Why alpine?** Small image size.
- **Why `USER node`?** Security — don't run as root.
- **`EXPOSE`** documents the port (doesn't actually publish it).

## 12. Docker image size is huge — how to reduce?

(They wrote "animate" — they mean *image*.) Cover the main levers:

- **Multi-stage builds** — build in a heavy stage, copy only artifacts into a clean minimal runtime stage. Biggest win for compiled languages.
- **Smaller base image** — alpine, `-slim`, or distroless instead of full OS images.
- **Fewer/combined layers** — chain `RUN` commands with `&&`, clean caches in the *same* layer (`apt-get clean && rm -rf /var/lib/apt/lists/*`).
- **`.dockerignore`** — exclude `node_modules`, `.git`, build artifacts from the build context.
- **Production-only dependencies** — don't ship dev/test tooling.
- **Order layers by change frequency** — for better caching.

Lead with **multi-stage builds + minimal base image** — those are the two that give the biggest reductions.

## 13. ALB vs NLB

- **ALB (Application Load Balancer)** — operates at **Layer 7** (HTTP/HTTPS). Does content-based routing (host/path), supports redirects, WAF integration, and is ideal for web apps and microservices. Slightly higher latency due to L7 processing.
- **NLB (Network Load Balancer)** — operates at **Layer 4** (TCP/UDP). Extremely high performance and low latency, handles millions of requests/sec, supports **static IPs** and preserves the client source IP. Use for non-HTTP protocols, extreme throughput, or when you need a static IP.

One-liner: **ALB routes on the content of HTTP requests (L7); NLB just forwards TCP/UDP packets fast (L4).** Choose ALB for smart HTTP routing, NLB for raw speed/static IPs/non-HTTP traffic.

---

**Two patterns to notice in this round:** several questions ("have you worked on," "in your project") want a *story from experience*, not a textbook definition — so always answer with a concrete example and ideally a "lesson learned." And the "write + explain" questions want you to **narrate while you code** ("I'm copying package.json first for layer caching…").

This round goes deeper into Docker security, Kubernetes internals, and Azure/AKS specifics. Let me work through each one. A few of these (#7 "PDS", #9 "K8 Version") need interpretation, which I'll flag.

## 1. Docker ARG vs ENV

Both set variables, but they differ in **scope and lifetime**:

- **ARG** — available only at **build time** (during `docker build`). Not present in the running container. Used for build-time parameters like a version or base image tag. Can be passed with `--build-arg`.
- **ENV** — available at **build time AND runtime** (persists in the running container). Used for runtime configuration the app reads.

```dockerfile
ARG NODE_VERSION=20          # build-time only
FROM node:${NODE_VERSION}-alpine
ENV APP_PORT=3000            # available in the running container
```

Key gotcha to mention: **don't put secrets in ARG or ENV** — both are visible in `docker history`/image layers. Use build secrets (`--mount=type=secret`) or runtime secret injection instead. Also note an `ARG` declared before `FROM` is only usable in the `FROM` line unless re-declared after it.

## 2. Dockerfile best practices

The ones interviewers want to hear:

- **Use minimal/official base images** (alpine, slim, distroless) and pin versions (avoid `latest`).
- **Multi-stage builds** to keep final images small.
- **Order layers by change frequency** — copy dependency manifests and install deps *before* copying source, so the dependency layer stays cached.
- **Use `.dockerignore`** to keep the build context small and avoid leaking secrets.
- **Run as non-root** (`USER`) for security.
- **One concern per container** — don't run multiple services in one image.
- **Combine `RUN` commands** and clean up in the same layer (`apt-get clean && rm -rf /var/lib/apt/lists/*`).
- **No secrets baked into the image.**
- **Use `COPY` over `ADD`** unless you specifically need ADD's URL/tar extraction.
- **Add `HEALTHCHECK`** and use explicit `EXPOSE`.

## 3. Multi-stage build

A Dockerfile with multiple `FROM` stages. You build/compile in an early "builder" stage that has all the heavy tooling, then copy only the final artifacts into a clean, minimal runtime stage. Everything from the build stage (compilers, dev dependencies, source) is discarded — giving a much smaller and more secure final image.

```dockerfile
FROM golang:1.22 AS build
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o /app

FROM gcr.io/distroless/static
COPY --from=build /app /app
ENTRYPOINT ["/app"]
```

The `COPY --from=build` is the key line — pulling just the artifact from the previous stage.

## 4. How to protect a Docker image from vulnerabilities

Layered approach:

- **Scan images** with tools like Trivy, Snyk, Grype, or Clair — integrate into CI so vulnerable images fail the build.
- **Use minimal base images** (distroless/alpine) — fewer packages means a smaller attack surface.
- **Keep base images and dependencies updated** — rebuild regularly to pull security patches.
- **Pin versions** for reproducibility, but rebuild to patch.
- **Run as non-root** and drop unnecessary Linux capabilities.
- **Don't store secrets** in the image.
- **Sign images** (Docker Content Trust / Cosign) and verify provenance.
- **Use a private registry** with scanning (ECR/ACR scan-on-push).
- **Make the filesystem read-only** at runtime where possible.

The two-part answer interviewers like: *scan continuously in CI* + *minimize the attack surface with minimal images and non-root.*

## 5. What is a Kubernetes Service?

A **Service** is an abstraction that provides a stable network endpoint (a fixed virtual IP and DNS name) for a dynamic set of pods. Since pods are ephemeral — they come and go with changing IPs — a Service gives clients one consistent address and load-balances across the healthy pods behind it (selected by labels).

Types:
- **ClusterIP** (default) — internal-only virtual IP, for pod-to-pod communication.
- **NodePort** — exposes the service on a static port on every node.
- **LoadBalancer** — provisions an external cloud load balancer.
- **ExternalName** — maps the service to an external DNS name.

One-liner: *"A Service decouples clients from ephemeral pod IPs by giving a stable address and load-balancing across pods matching its label selector."*

## 6. How to access a Kubernetes application locally

Several options depending on the need:

- **`kubectl port-forward`** — quickest for local debugging: `kubectl port-forward svc/myapp 8080:80` then hit `localhost:8080`. Tunnels directly to the service/pod.
- **`kubectl proxy`** — proxies the API server, useful for accessing the dashboard/API.
- **NodePort** — access via `<nodeIP>:<nodePort>`.
- **Ingress** with a local hosts-file entry (for ingress testing).
- For local clusters (minikube): **`minikube service <name>`** opens it directly.

`kubectl port-forward` is the answer they're usually after for "local device" access during development/debugging.

## 7. What is PDS?

This is almost certainly a typo/mishearing. Most likely they meant **PV/PVC** (Persistent Volume / Persistent Volume Claim) or **PDB** (which is your next question). Given #8 is PDB, I'll assume #7 is about **persistent storage — PV and PVC**:

- **PersistentVolume (PV)** — a piece of storage in the cluster provisioned by an admin or dynamically. It's a cluster resource independent of any pod's lifecycle.
- **PersistentVolumeClaim (PVC)** — a request for storage by a user/pod ("I need 10Gi, ReadWriteOnce"). Kubernetes binds the PVC to a matching PV.
- **StorageClass** — enables *dynamic* provisioning — when a PVC is created, the StorageClass automatically provisions a backing PV (e.g., an EBS/Azure Disk).

The flow: Pod → mounts → PVC → binds to → PV → backed by → real storage. If they actually meant something else (PDS could be a product-specific term), it's worth politely confirming in the interview.

## 8. PDB (Pod Disruption Budget)

A **PodDisruptionBudget** limits how many pods of an application can be **voluntarily** disrupted at once — protecting availability during operations like node drains, cluster upgrades, or autoscaling.

You set either `minAvailable` (e.g., "at least 2 pods must stay up") or `maxUnavailable` (e.g., "no more than 1 down at a time"). During a voluntary disruption (like `kubectl drain`), Kubernetes respects the PDB and won't evict pods if it would violate the budget.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: myapp
```

Key nuance: PDBs protect against **voluntary** disruptions (drains, upgrades), **not involuntary** ones (node hardware failure, crashes).

## 9. K8s Version

They're likely asking **what version you've worked with** and/or **what's current**. Let me check the current stable version so you have an accurate number.Now I have the current numbers. For this question, they're checking whether you stay current and know how versioning works:

The latest stable Kubernetes release is 1.36.1, released on 2026-05-13. Kubernetes follows an N-2 support policy, meaning the 3 most recent minor versions receive security and bug fixes, with a 15-week release cycle — so currently 1.36, 1.35, and 1.34 are supported. Explain the **versioning scheme** (`major.minor.patch`, e.g., 1.36.1) and then **say what you've actually run** ("In our project we were on 1.2x on EKS/AKS"). Don't claim the bleeding-edge version unless you really used it — interviewers respect "we were a version or two behind because we follow the N-2 policy and validate upgrades in staging first."

## 10. How to upgrade Kubernetes

The principle is **control plane first, then worker nodes, one minor version at a time** (you can't skip minor versions — 1.34 → 1.36 must go through 1.35).

General process:
1. **Read the release notes** for deprecated/removed APIs and breaking changes.
2. **Back up etcd** (and validate in a staging cluster first).
3. **Upgrade the control plane** (API server, controller-manager, scheduler) first.
4. **Upgrade worker nodes** — cordon and `drain` each node (respecting PDBs), upgrade kubelet/kube-proxy, then uncordon. Or use rolling node-pool replacement.
5. **Verify** workloads and add-ons (CNI, CoreDNS, ingress) are healthy.

For **managed clusters** the cloud handles most of this:
- **EKS**: upgrade the control plane via console/CLI, then upgrade managed node groups.
- **AKS**: `az aks upgrade` (can do control plane + node pools).
- **kubeadm** (self-managed): `kubeadm upgrade plan` then `kubeadm upgrade apply`.

The interviewer wants to hear: *can't skip minor versions, check deprecated APIs, drain nodes respecting PDBs, test in staging, control plane before workers.*

## 11. Deployment stuck in CrashLoopBackOff — root causes & fix

This is a classic. **CrashLoopBackOff** means the container starts, exits/crashes, and kubelet keeps restarting it with increasing backoff delays.

**Diagnosis steps:**
```bash
kubectl describe pod <pod>          # check Events + last state + exit code
kubectl logs <pod> --previous       # logs from the crashed instance — the key command
kubectl get events --sort-by=.lastTimestamp
```

**Common root causes and fixes:**

- **Application error / unhandled exception on startup** → fix the code/config; check `logs --previous`.
- **Missing or wrong config** — env vars, ConfigMap, or Secret not mounted/available → verify the ConfigMap/Secret exists and is referenced correctly.
- **OOMKilled** (exit code 137) — memory limit too low → check `describe` for "OOMKilled", raise memory limits.
- **Failing liveness probe** killing a healthy-but-slow-starting app → increase `initialDelaySeconds` or use a `startupProbe`.
- **Bad command/entrypoint** — wrong `command`/`args`, binary not found → fix the Dockerfile/manifest.
- **Dependency unavailable at startup** (DB, downstream service not reachable) → add retry/backoff logic or init containers that wait for dependencies.
- **Permission issues** — can't write to a path, running as wrong user → fix volume permissions / `securityContext`.

The structured answer: *"First `logs --previous` and `describe` to see the exit code and events, then map the exit code/error to the cause — 137 means OOM, a probe failure shows in events, a config error shows in logs — and fix accordingly."*

## 12. AKS unable to provision persistent storage AND load balancer — how to fix

This is an Azure-specific troubleshooting question. The common thread for *both* failing simultaneously is usually a **permissions/identity problem** — the AKS cluster identity can't create Azure resources.

**For both issues — check first:**
- **Cluster managed identity / service principal permissions** — the AKS identity needs the **Contributor** (or appropriate) role on the node resource group. If it can't provision Azure Disks or a Load Balancer, expired SP credentials or missing RBAC roles is the #1 cause.

**Persistent storage (PV/PVC stuck Pending):**
- `kubectl describe pvc <name>` to see the provisioning error.
- Verify the **StorageClass** exists and the provisioner is correct (`disk.csi.azure.com` / `file.csi.azure.com`).
- Check the access mode — **Azure Disk is ReadWriteOnce only**; if you need ReadWriteMany you must use **Azure Files**. A wrong access mode is a very common cause.
- Check subscription **disk quota** in the region.
- Ensure the CSI drivers are enabled on the cluster.

**Load balancer (Service stuck `<pending>` for EXTERNAL-IP):**
- `kubectl describe svc <name>` for the error.
- Check **public IP quota** in the subscription/region.
- Verify the cluster identity has rights on the node resource group to create the LB and public IP.
- Check subnet/NSG configuration and whether you're mixing internal/external LB annotations incorrectly.

**The unifying answer to lead with:** *"When both storage and LB provisioning fail together, I'd suspect the cluster's managed identity lacks permissions on the node resource group, or credentials expired — because both operations call Azure APIs to create resources. I'd check `kubectl describe` on the PVC and Service for the exact error, then verify identity roles and subscription quotas."*

## 13. How is storage connected in AKS

AKS uses **CSI (Container Storage Interface) drivers** to connect Azure storage to pods. The two main options:

- **Azure Disk** (`disk.csi.azure.com`) — block storage, **ReadWriteOnce** (mounted by one node at a time). Good for databases and single-pod stateful workloads. Maps to a managed disk.
- **Azure Files** (`file.csi.azure.com`) — SMB/NFS file share, **ReadWriteMany** (mounted by multiple pods/nodes). Good for shared storage.

The connection flow: you define a **StorageClass** (pointing at the CSI provisioner) → a **PVC** requests storage → dynamic provisioning creates the Azure Disk/File and a **PV** → the pod mounts the PVC. Azure Blob can also be mounted via the Blob CSI driver. For existing storage you can statically provision by creating the PV manually.

## 14. How to identify a secret issue in a Kubernetes application/cluster

Work from the symptom inward:

```bash
kubectl get secret <name> -n <ns>              # does it exist?
kubectl describe pod <pod>                      # events: "secret not found", mount errors
kubectl get secret <name> -o jsonpath='{.data}' # check keys exist
echo <value> | base64 -d                        # verify decoded value (secrets are base64-encoded, not encrypted)
```

**Common secret issues and how to spot them:**
- **Secret doesn't exist / wrong name** → pod events show `couldn't find key` or `secret not found`; pod may be stuck in `ContainerCreating`.
- **Wrong key referenced** in env/volume → app gets empty/missing value.
- **Namespace mismatch** — secrets are namespaced; a pod can't reference a secret in another namespace.
- **Base64 encoding mistakes** — people forget secret values must be base64-encoded in the manifest (or use `stringData`).
- **Not updated after rotation** — pods using env-var secrets don't pick up changes until restarted (mounted-volume secrets update, but with a delay).
- **RBAC** — the ServiceAccount may lack permission to read the secret.

Also worth mentioning: secrets are only **base64-encoded, not encrypted** by default, so for real security you enable **encryption at rest** (etcd encryption) or use an external secrets manager (Azure Key Vault via the Secrets Store CSI driver, Vault). Mentioning the Key Vault CSI integration scores points in an Azure interview.

## 15. Azure Load Balancer backend pool not receiving traffic

Troubleshoot the path from the LB to the backend:

- **Health probe failing** — this is the #1 cause. If the health probe fails, the LB marks the backend unhealthy and stops sending traffic. Check the probe's port, protocol, and path match what the backend actually serves, and that the app responds with a healthy status. Verify via the Azure portal's backend health view.
- **NSG / firewall rules** — a Network Security Group may be blocking the probe or traffic port. Ensure inbound rules allow the LB probe (and allow `AzureLoadBalancer` source) and the application port.
- **Backend pool misconfiguration** — VMs/instances not actually added to the pool, or pointing to the wrong NIC/IP.
- **LB rule mismatch** — frontend port, backend port, and protocol in the load-balancing rule must align with the app.
- **Application not listening** — the app isn't bound to the right interface/port (`0.0.0.0` vs `localhost`), or the service is down. SSH/exec in and check the port is listening.
- **(AKS context)** if it's a Kubernetes LoadBalancer Service, check the Service's `targetPort` matches the container port and that endpoints exist (`kubectl get endpoints <svc>` — empty endpoints means the selector matches no ready pods).

**Lead with:** *"First thing I check is the health probe — if the LB's probe fails, it pulls the backend out of rotation, so no traffic flows. Then NSG rules, then whether the app is actually listening on the expected port."*

---
