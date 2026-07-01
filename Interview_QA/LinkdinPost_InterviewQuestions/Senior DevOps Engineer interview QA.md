# Senior DevOps/Cloud/SRE Interview Answers

Here's how I'd answer these in an interview setting, drawing from real production experience.

---

## 🛠 Git & Version Control

**What does `git add` do and why is it important before committing?**

`git add` moves changes from the working directory to the staging area (index). It's important because it lets you craft logical, atomic commits — you decide *what* goes into a commit, not just *that* a commit happens. Without staging, you'd be forced to commit every modified file together, which makes history noisy and bisecting bugs harder.

**How do you stage only specific changes in Git?**

A few approaches depending on granularity:
- `git add <file>` — specific files
- `git add -p` (patch mode) — stage individual hunks within a file interactively; this is what I use most often when I've made unrelated changes in one file
- `git add -i` — full interactive mode
- `git add '*.tf'` — glob patterns

`git add -p` is the senior-engineer move. It forces you to review your diff line by line before it goes anywhere.

**If you have to update 2 YAML files in a remote repo as a new joiner, how will you do it?**

Standard workflow:

```bash
git clone <repo-url>
cd <repo>
git checkout -b feature/update-yaml-configs
# edit the two YAML files
yamllint file1.yaml file2.yaml      # validate syntax first
git add file1.yaml file2.yaml
git commit -m "Update <thing> in config files"
git push origin feature/update-yaml-configs
```

Then raise a Pull/Merge Request for review. As a new joiner I'd never push directly to `main`/`master` — I'd follow the team's branching convention (GitFlow, trunk-based, whatever they use), tag the right reviewers, and link the ticket. I'd also pull latest from `main` and rebase before pushing to avoid stale-branch surprises.

**What is `git stash` & when do you use it?**

`git stash` shelves uncommitted changes so your working directory is clean, without losing the work. Common scenarios:
- A hotfix lands and I need to switch branches without committing half-done work
- I'm on the wrong branch and need to move changes (`git stash`, checkout correct branch, `git stash pop`)
- I want to pull latest but have local changes that would conflict

Useful commands: `git stash push -m "wip: refactoring auth"`, `git stash list`, `git stash pop`, `git stash apply stash@{2}`.

**Difference between `git fetch` & `git pull`**

- `git fetch` downloads commits, refs, and objects from the remote but does **not** modify your working branch. Safe — it just updates your local view of the remote.
- `git pull` = `git fetch` + `git merge` (or `git rebase` if configured). It modifies your current branch.

In production work I prefer `git fetch` then inspect (`git log origin/main..HEAD`) before merging or rebasing. `git pull` is fine for trivial cases but can produce surprising merge commits.

**What is `git cherry-pick` & when would you use it?**

`git cherry-pick <commit-sha>` applies a specific commit from one branch onto another. Real use cases:
- Backporting a hotfix from `main` to a release branch (e.g., `release/v2.3`)
- Pulling in a single bug fix without merging an entire feature branch
- Recovering a commit from a branch that was reset/rebased

Caveat: cherry-picking creates a new commit with a different SHA, which can cause duplicate-commit confusion if the source branch is later merged.

---

## ⚙ Ansible

**What does an Ansible playbook look like & its main components?**

A playbook is a YAML file describing automation. Main components: hosts (target inventory), become (privilege escalation), vars, tasks (actions using modules), handlers (triggered on change notifications), roles (reusable bundles), and tags.

**Sample playbook — installing Docker on Ubuntu:**

```yaml
---
- name: Install Docker on Ubuntu hosts
  hosts: docker_nodes
  become: yes
  vars:
    docker_users:
      - ubuntu

  tasks:
    - name: Install prerequisite packages
      apt:
        name:
          - ca-certificates
          - curl
          - gnupg
          - lsb-release
        state: present
        update_cache: yes

    - name: Add Docker GPG key
      apt_key:
        url: https://download.docker.com/linux/ubuntu/gpg
        state: present

    - name: Add Docker apt repository
      apt_repository:
        repo: "deb [arch=amd64] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
        state: present

    - name: Install Docker Engine
      apt:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
          - docker-compose-plugin
        state: present
        update_cache: yes

    - name: Ensure Docker service is started and enabled
      systemd:
        name: docker
        state: started
        enabled: yes

    - name: Add users to docker group
      user:
        name: "{{ item }}"
        groups: docker
        append: yes
      loop: "{{ docker_users }}"

  handlers:
    - name: restart docker
      systemd:
        name: docker
        state: restarted
```

Run with `ansible-playbook -i inventory.ini docker-install.yml`. For Nexus, the pattern is similar — install Java prereqs, download the Nexus tarball, extract to `/opt/nexus`, create a systemd unit file, and start the service.

---

## 🐳 Docker & Containers

**Dockerfile for a Maven project (multi-stage):**

```dockerfile
# ---------- Build stage ----------
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /build
COPY pom.xml .
RUN mvn dependency:go-offline -B
COPY src ./src
RUN mvn clean package -DskipTests -B

# ---------- Runtime stage ----------
FROM eclipse-temurin:17-jre-alpine
RUN addgroup -S app && adduser -S app -G app
WORKDIR /app
COPY --from=build /build/target/*.jar app.jar
RUN chown -R app:app /app
USER app
EXPOSE 8080
ENTRYPOINT ["java","-XX:+UseContainerSupport","-XX:MaxRAMPercentage=75.0","-jar","app.jar"]
```

**What is a multi-stage build & why use it?**

Multi-stage builds use multiple `FROM` statements in one Dockerfile. The build stage has all the compilers, JDKs, and build tools you need; the final stage copies only the built artifact into a slim runtime image. Benefits: dramatically smaller images (a JDK + Maven image is ~800MB; a JRE-alpine image is ~180MB), reduced attack surface, no build secrets or source code in the final image, and faster pulls in CI/CD.

**How can two containers (app & DB) talk to each other?**

Several patterns:
1. **Docker Compose / user-defined bridge network** — containers on the same network resolve each other by service name (`jdbc:postgresql://db:5432/mydb`). This is the standard for local dev.
2. **Docker network manually** — `docker network create app-net`, then run both containers with `--network app-net`.
3. **Kubernetes** — Service objects provide DNS (`db-service.namespace.svc.cluster.local`).
4. **External managed DB** — in production I almost always use RDS/CloudSQL with the connection string injected via env vars/secrets, not a DB container.

**Best practices for production Dockerfiles:**
- Use specific, pinned base image tags (never `latest`)
- Multi-stage builds to keep final images lean
- Run as a non-root user
- Use `.dockerignore` to exclude `.git`, `node_modules`, secrets, etc.
- Order instructions by change frequency (deps before code) to maximize layer caching
- Use `COPY` over `ADD` unless you specifically need ADD's URL/tarball features
- Add `HEALTHCHECK` instructions
- Don't store secrets in image layers — use BuildKit secrets or runtime env
- Scan images with Trivy/Snyk in CI

**Why use a non-root user in Docker images?**

Defense in depth. If an attacker exploits a vulnerability in your app (RCE, container escape primitive), running as root inside the container gives them root inside the namespace, which is the foundation many escapes start from. Running as a non-root UID limits what they can read/write inside the container and reduces blast radius. Kubernetes admission policies (PSA `restricted`) often outright reject root containers.

**How to optimize Docker layers & reduce image size:**
- Multi-stage builds (biggest single win)
- Alpine or distroless base images
- Combine related `RUN` commands with `&&` to avoid intermediate layers, and clean apt caches in the same layer: `RUN apt-get update && apt-get install -y foo && rm -rf /var/lib/apt/lists/*`
- Copy only what you need; respect `.dockerignore`
- Order Dockerfile commands so frequently-changing layers are at the bottom
- Use BuildKit cache mounts for package managers

**Deploy a Dockerized app on EC2 using Terraform — minimal sketch:**

```hcl
resource "aws_security_group" "app_sg" {
  name        = "app-sg"
  description = "Allow HTTP and SSH"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.admin_cidr]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "app" {
  ami                    = data.aws_ami.amazon_linux_2.id
  instance_type          = "t3.small"
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.app_sg.id]
  key_name               = var.key_name
  iam_instance_profile   = aws_iam_instance_profile.ec2_profile.name

  user_data = <<-EOF
    #!/bin/bash
    yum update -y
    amazon-linux-extras install docker -y
    systemctl enable --now docker
    usermod -aG docker ec2-user
    docker run -d --restart=always -p 80:8080 \
      --name app ${var.ecr_repo}:${var.image_tag}
  EOF

  tags = { Name = "app-server" }
}
```

In a real setup I'd front this with an ALB, put the EC2 in an Auto Scaling Group, store the image in ECR, and authenticate with an instance profile rather than baking credentials.

**CMD vs ENTRYPOINT vs RUN:**

- **RUN** — executes at *build time* and commits the result as a new layer. Used to install packages, compile code, etc.
- **ENTRYPOINT** — defines the executable that runs when the container starts. Hard to override (requires `--entrypoint`).
- **CMD** — provides default arguments. Easily overridden by passing args to `docker run`.

The idiomatic pattern: `ENTRYPOINT ["java","-jar","app.jar"]` + `CMD ["--spring.profiles.active=prod"]`. ENTRYPOINT defines what runs; CMD provides defaults you can change.

---

## ☸ Kubernetes

**Kubernetes architecture:**

Two planes — control plane and data plane (worker nodes).

Control plane components: API server, etcd, scheduler, controller manager, cloud controller manager. Worker nodes run: kubelet, kube-proxy, container runtime (containerd/CRI-O).

**Component roles:**

- **kube-apiserver** — front door for everything. All reads/writes (kubectl, controllers, kubelet) go through it. Validates and persists to etcd.
- **etcd** — distributed key-value store holding all cluster state. Source of truth. Needs backups (`etcdctl snapshot save`).
- **kube-scheduler** — watches for unscheduled Pods, picks a node based on resource requests, affinity, taints/tolerations, topology constraints. Writes the binding back through the API server.
- **kube-controller-manager** — runs control loops: Deployment controller, ReplicaSet controller, Node controller, endpoints controller, etc. Each reconciles desired state vs actual state.
- **kubelet** — node agent. Receives PodSpecs, talks to the container runtime to start/stop containers, reports node and pod status.
- **kube-proxy** — programs iptables/IPVS rules on each node so Service IPs route to backing pods.

**Service types:**

- **ClusterIP** (default) — internal-only virtual IP; pods in the cluster can reach it
- **NodePort** — exposes the service on a static port on every node's IP (30000-32767)
- **LoadBalancer** — provisions a cloud LB (ELB/NLB on AWS) pointing at the service
- **ExternalName** — DNS CNAME to an external hostname; no proxying
- **Headless** (`clusterIP: None`) — no virtual IP; DNS returns pod IPs directly. Used for StatefulSets and service discovery patterns.

**kubectl essentials:**

```bash
# Exec into a container in a pod
kubectl exec -it <pod-name> -- /bin/bash
kubectl exec -it <pod-name> -c <container-name> -- sh   # multi-container

# Pod info
kubectl get pod <pod-name> -o wide
kubectl describe pod <pod-name>
kubectl get pod <pod-name> -o yaml

# All pod IPs across the cluster
kubectl get pods -A -o wide
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\n"}{end}'

# IP of a specific pod
kubectl get pod <pod-name> -o jsonpath='{.status.podIP}'
```

---

## 📜 Terraform & AWS

**Script vs module:**

A "script" (root module / config) is the Terraform you `apply` directly — it stitches together resources for a specific environment. A **module** is a reusable, parameterized bundle of resources you call from a root config. Modules promote DRY: write a VPC module once, call it from dev/staging/prod with different inputs.

**Module structure I use:**

```
modules/
  vpc/
    main.tf         # resources
    variables.tf    # inputs
    outputs.tf      # exposed values
    versions.tf     # provider/TF version constraints
    README.md       # how to use it
environments/
  dev/
    main.tf         # calls modules with dev values
    backend.tf      # remote state config
    terraform.tfvars
  prod/
    main.tf
    backend.tf
    terraform.tfvars
```

**Secure VPC with subnets & route tables — sketch:**

```hcl
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = { Name = "main-vpc" }
}

resource "aws_subnet" "public" {
  count                   = length(var.azs)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index)
  availability_zone       = var.azs[count.index]
  map_public_ip_on_launch = true
  tags = { Name = "public-${var.azs[count.index]}" }
}

resource "aws_subnet" "private" {
  count             = length(var.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index + 10)
  availability_zone = var.azs[count.index]
  tags = { Name = "private-${var.azs[count.index]}" }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
}

resource "aws_eip" "nat" {
  count  = length(var.azs)
  domain = "vpc"
}

resource "aws_nat_gateway" "nat" {
  count         = length(var.azs)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
}

resource "aws_route_table" "private" {
  count  = length(var.azs)
  vpc_id = aws_vpc.main.id
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.nat[count.index].id
  }
}

resource "aws_route_table_association" "public" {
  count          = length(var.azs)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count          = length(var.azs)
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}
```

Security touches: private subnets for app/DB tiers, NACLs at the subnet boundary, flow logs to CloudWatch/S3, no `0.0.0.0/0` ingress on app/db SGs, only the ALB SG can reach app SG on the app port.

**EC2 in multiple subnets** — use `count` or `for_each` over the subnet IDs and place instances accordingly, or use an Auto Scaling Group with `vpc_zone_identifier = aws_subnet.private[*].id`.

**Best practices for Terraform infra provisioning:**
- Remote state (S3 + DynamoDB lock, or Terraform Cloud) — never local state in team setups
- State per environment (separate state files for dev/staging/prod)
- Pin provider versions in `required_providers`
- Use modules for repeated patterns
- `terraform fmt`, `terraform validate`, `tflint`, `tfsec`/`checkov` in CI
- Plan in PR, apply on merge (Atlantis or similar)
- Tag everything (env, owner, cost-center)
- Never commit `.tfstate` or `.tfvars` containing secrets

**user-data scripts:**

```hcl
user_data = <<-EOF
  #!/bin/bash
  yum update -y
  yum install -y nginx
  systemctl enable --now nginx
EOF
```

Or `user_data = file("scripts/bootstrap.sh")`, or `templatefile()` for variable interpolation. Use `user_data_replace_on_change = true` if you want changes to force a new instance.

**Corrupted state files — recovery:**

1. **Pull the latest known-good version** from S3 versioning (always enable versioning on the state bucket) or from your Terraform Cloud state history.
2. If you have a local backup: `terraform state push backup.tfstate` to restore.
3. If specific resources are out of sync: `terraform import <resource> <id>` to re-attach existing AWS resources to state.
4. For partial corruption: `terraform state rm` and re-import problematic resources.
5. As a last resort, `terraform refresh` against the cloud to reconcile.
6. Always run `terraform plan` first — if it shows wild destroy/create, do not apply.

Prevention is more important than cure: state locking via DynamoDB, versioning on the S3 bucket, restricted IAM on the state bucket.

**Integrating Terraform with Kubernetes:**

Two layers: provision the cluster (EKS module), then manage in-cluster resources with the `kubernetes` and `helm` providers.

```hcl
provider "kubernetes" {
  host                   = module.eks.cluster_endpoint
  cluster_ca_certificate = base64decode(module.eks.cluster_ca)
  exec {
    api_version = "client.authentication.k8s.io/v1beta1"
    command     = "aws"
    args        = ["eks","get-token","--cluster-name", module.eks.cluster_name]
  }
}

resource "helm_release" "argocd" {
  name             = "argocd"
  repository       = "https://argoproj.github.io/argo-helm"
  chart            = "argo-cd"
  namespace        = "argocd"
  create_namespace = true
  version          = "7.6.12"
}
```

In practice I keep cluster provisioning in Terraform but push application-layer manifests through ArgoCD/Flux — Terraform shouldn't be in the hot path for every app deploy.

---

## 🚀 Deployment Strategies

**Zero-downtime strategies:** Blue-Green, Canary, and Rolling Update can all achieve zero downtime if configured with proper readiness probes and connection draining. Blue-Green is the cleanest in concept; Rolling is the Kubernetes default.

**How Rolling Update works:**

The Deployment controller gradually replaces old pods with new ones, governed by `maxSurge` and `maxUnavailable`. With `maxSurge: 25%, maxUnavailable: 25%` on a 4-pod deployment, K8s spins up one new pod, waits for readiness probe to pass, then terminates one old pod, repeating until all are replaced. Failed readiness probes pause the rollout. `kubectl rollout undo` reverts. Critical: tune `terminationGracePeriodSeconds` and a `preStop` hook so in-flight requests drain.

**Blue-Green:**

Two identical environments (blue = current prod, green = new version). Deploy to green, run smoke tests, then flip the load balancer / service selector / Route53 weight to green. Rollback is just flipping back. Pro: instant rollback, full validation before traffic. Con: 2× infrastructure cost during cutover, DB schema changes need backwards-compatible migrations.

**Canary:**

Route a small slice of traffic (5%) to the new version, monitor SLOs (latency, error rate), gradually shift more (25%, 50%, 100%). Tools: Argo Rollouts, Flagger, Istio's VirtualService weight splitting, or ALB weighted target groups. Pro: catches problems with real production traffic at low blast radius. Con: more orchestration complexity, needs strong observability to make automated promote/rollback decisions.

---

## 🔍 Code Quality & CI/CD

**Quality Profile vs Quality Gate in SonarQube:**

- **Quality Profile** — the *ruleset* used to analyze code (which rules are active, severities, language-specific). "Sonar Way" is the default; teams often clone and tune it.
- **Quality Gate** — the *pass/fail criteria* for a build: e.g., "0 new bugs, 0 new vulnerabilities, ≥80% coverage on new code, ≤3% duplication on new code." Builds fail if the gate isn't met.

Profile defines *what* is checked; gate defines *whether the build passes*.

**Code coverage in Maven with SonarQube** — add JaCoCo:

```xml
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <version>0.8.11</version>
  <executions>
    <execution>
      <goals><goal>prepare-agent</goal></goals>
    </execution>
    <execution>
      <id>report</id>
      <phase>test</phase>
      <goals><goal>report</goal></goals>
    </execution>
  </executions>
</plugin>
```

Then run:

```bash
mvn clean verify sonar:sonar \
  -Dsonar.host.url=https://sonar.company.com \
  -Dsonar.login=$SONAR_TOKEN \
  -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
```

**SonarQube server setup (high level):**

1. Provision a VM (or run on EKS via official Helm chart)
2. Install PostgreSQL (required for production; embedded H2 is dev-only)
3. Set `vm.max_map_count=524288` and `fs.file-max=131072` (Elasticsearch reqs)
4. Run SonarQube as a non-root user, point it at the Postgres DSN in `sonar.properties`
5. Front with Nginx + TLS, set up LDAP/SSO
6. Configure webhooks back to Jenkins so quality gate results post to pipelines
7. Back up the DB and the `data/` directory

**Jenkins Shared Library:**

A versioned Git repo containing reusable Groovy code (steps, variables, classes) that pipelines can import. Avoids copy-pasting the same `Jenkinsfile` logic across 50 repos.

Standard structure:

```
(shared-lib repo)
├── vars/                # global variables / "steps" callable from pipelines
│   ├── buildMaven.groovy
│   ├── deployToK8s.groovy
│   └── notifySlack.groovy
├── src/                 # standard Java/Groovy classes
│   └── com/company/ci/
│       └── PipelineUtils.groovy
├── resources/           # static files loadable via libraryResource()
│   └── templates/
└── README.md
```

Configure once globally in Jenkins (*Manage Jenkins → System → Global Pipeline Libraries*), then in a Jenkinsfile:

```groovy
@Library('company-shared-lib@v1.4.0') _

pipeline {
  agent any
  stages {
    stage('Build') { steps { buildMaven() } }
    stage('Deploy') { steps { deployToK8s(env: 'dev', chart: 'my-app') } }
  }
  post {
    failure { notifySlack(channel: '#alerts') }
  }
}
```

Pin to tags/versions, not `main` — pipeline reproducibility matters.

---

## 📈 Monitoring & Observability

**Setting up Prometheus & Grafana from scratch (on Kubernetes):**

The standard play is the `kube-prometheus-stack` Helm chart, which bundles Prometheus, Alertmanager, Grafana, node-exporter, kube-state-metrics, and a starter set of dashboards and alerts.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create namespace monitoring
helm install kps prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f values.yaml
```

Key things in `values.yaml`: storage class & PV size for Prometheus, retention (`--storage.tsdb.retention.time=30d`), Grafana admin password from a secret, Ingress with TLS, Alertmanager receivers (Slack, PagerDuty, email).

For VMs (non-K8s): install Prometheus binary, node_exporter on each target, configure `scrape_configs` in `prometheus.yml`, install Grafana, add Prometheus as a data source.

**Integrating metrics & logs into Grafana:**

- Metrics: Prometheus data source. Apps expose `/metrics` (Spring Boot Actuator + Micrometer, prom-client for Node, etc.). For K8s, ServiceMonitor CRDs tell Prometheus what to scrape.
- Logs: Loki (lightweight, log-aggregation pairs with Prometheus labels) + Promtail/Fluent Bit shipping logs. Or Elastic/OpenSearch as a data source if you have an existing ELK stack.
- Traces: Tempo or Jaeger as a Grafana data source. OpenTelemetry SDK in apps.

Build dashboards layered: infra (node CPU/mem/disk/network), platform (K8s deployments/pods, replica health), application (RED metrics — Rate, Errors, Duration), business (orders/sec, signups). Use folders and variables (`$namespace`, `$pod`) for reusable panels.

**Monitoring & logging tools I've used:**

- Metrics: Prometheus, CloudWatch Metrics, Datadog
- Logs: Loki, ELK (Elasticsearch/Logstash/Kibana), CloudWatch Logs, Datadog Logs, Splunk
- Tracing: Jaeger, Tempo, AWS X-Ray
- Synthetic / uptime: Blackbox exporter, Pingdom, CloudWatch Synthetics
- APM: New Relic, Datadog APM

**Alerts — examples for infra & apps:**

Infra:
- Node CPU > 85% for 10m
- Node disk > 80%
- Node memory pressure
- Pod CrashLoopBackOff
- PV usage > 85%
- etcd leader changes spiking

Application (SLO-based):
- Error rate > 1% over 5m (page) / > 0.5% over 30m (ticket)
- p99 latency > SLO threshold
- Request rate drop > 50% vs same-time-last-week (could mean upstream is broken)
- Queue depth growing without bound
- Cert expiry < 14 days

Route by severity: critical → PagerDuty + Slack; warning → Slack only. Always include a runbook link in the alert annotation.

---

## 📆 SDLC & ITIL

**SDLC basics & models:**

SDLC is the structured process of building software: requirements → design → implementation → testing → deployment → maintenance.

Models:
- **Waterfall** — sequential, phase-gated. Works for fixed-scope, regulated domains; brittle when requirements change.
- **Agile** — iterative sprints, working software each iteration, embrace change. Scrum/Kanban being the dominant flavors.
- **Spiral** — risk-driven iterations; each loop = plan/analyze risk/engineer/evaluate. Used in large, high-risk programs.
- **DevOps** — extends Agile across dev *and* ops: CI/CD, IaC, automated testing, monitoring, blameless postmortems. Emphasis on flow, feedback, continuous learning (the "Three Ways" from *The Phoenix Project*).

**ITIL basics:**

ITIL is a framework for IT Service Management. The current version (ITIL 4) organizes work around the Service Value System and 34 practices, but the classic stages people still ask about are: Service Strategy → Service Design → Service Transition → Service Operation → Continual Service Improvement.

Key practices for an SRE/DevOps role:
- **Incident Management** — restore service ASAP. Triage, severity, comms, MTTR is the metric. Maps to on-call/SRE incident response.
- **Problem Management** — find the *root cause* so the incident doesn't recur. Maps to blameless postmortems and action items.
- **Change Management / Change Enablement** — assess risk and authorize changes to production. In modern DevOps, this becomes lightweight standard/normal/emergency change tracking, ideally automated through the CI/CD pipeline rather than manual CAB meetings.

Relation to DevOps: ITIL provides governance and process structure; DevOps provides speed and automation. Mature orgs blend the two — auto-classify low-risk changes as "standard" so they flow through CI/CD without manual approval, while still requiring CAB review for high-risk changes.

---

## 💻 CI/CD Flow in Project

**Bitbucket → Jenkins → JFrog → EKS → ArgoCD → Helm — end-to-end:**

1. **Bitbucket** — developer pushes to a feature branch, opens a PR to `develop`/`main`. PR triggers a Jenkins webhook.
2. **Jenkins (CI)** — multibranch pipeline (using the shared library) runs:
   - Checkout
   - Build (`mvn clean package` or `npm build`)
   - Unit tests + JaCoCo coverage
   - SonarQube scan → quality gate
   - Build Docker image, tag with `${git-short-sha}-${build-number}`
   - Trivy/Snyk image scan
   - Push image to **JFrog Artifactory** (Docker repo)
   - Update the **Helm values repo** with the new image tag (commit + push)
3. **JFrog Artifactory** — the artifact store. Holds Docker images, Helm charts, and (for Java) Maven artifacts. Promotion between repos (dev → staging → prod) signals readiness for each environment.
4. **GitOps repo** — separate Git repo (or directory) holding Helm values for each environment: `values-dev.yaml`, `values-staging.yaml`, `values-prod.yaml`. Jenkins commits the new image tag to `values-dev.yaml` on a merge to `develop`; promotion to staging/prod is a separate PR (approved by release manager).
5. **ArgoCD** — watches the GitOps repo. When `values-dev.yaml` changes, ArgoCD detects drift and reconciles the cluster by applying the new Helm chart with the updated values.
6. **EKS** — runs the workload. ArgoCD applies via Helm; pods rolling-update; readiness probes gate traffic.
7. **Post-deploy** — Prometheus/Grafana watches SLOs, Alertmanager pages on regression, Slack notification on success/failure.

**Role of the Helm repository:**

Charts are versioned packages (templates + default values). The Helm repo (hosted in JFrog or ChartMuseum or as OCI artifacts in ECR/Artifactory) is the *source of truth for chart versions*. ArgoCD Applications reference `chart: my-app, targetRevision: 1.4.2, helm.values: ...`. Versioning charts independently of the app image lets you change deployment shape (resource limits, init containers) without rebuilding the app, and roll back the chart independently.

**How dev/prod values repo syncs with the chart repo:**

Two-repo GitOps pattern:
- **Chart repo** — owned by platform team. Holds chart templates (deployment, service, ingress, HPA). Versioned via SemVer. Published to the Helm repo on release.
- **Env/values repo** — owned by app team. Holds `values-<env>.yaml` per environment. References the chart version + image tag.

ArgoCD Application points at the env repo's path; the Application manifest declares `helm.chart` (or pulls from the Helm repo) and `helm.valueFiles: [values-prod.yaml]`. Promoting a release = PR in the env repo bumping `imageTag` (and optionally chart version). ArgoCD syncs.

This separation means:
- Chart changes (platform-level) go through their own review and SemVer
- Environment changes (config, image tag) are isolated and audit-logged via Git
- Rollback in any environment = revert the PR in the env repo

---

That covers the full set. Happy to drill into any of these deeper — common follow-ups are around state-file recovery scenarios, canary automation with Argo Rollouts, or SLO-based alerting design.

