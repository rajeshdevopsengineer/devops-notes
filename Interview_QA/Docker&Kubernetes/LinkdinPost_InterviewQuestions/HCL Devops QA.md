# HCL Technologies — DevOps Interview Answers

Covering both rounds: the general technical round and the Terraform-focused deep dive.

---

# 1️⃣ Technical Round — DevOps Fundamentals

## 1. Explain your CI/CD pipeline design. Which tools did you use and why?

A representative pipeline I'd describe in an interview:

**Flow:** Developer pushes to Bitbucket → Jenkins multibranch pipeline triggers → build + test + SonarQube → Docker image to JFrog Artifactory → image tag commit to GitOps repo → ArgoCD reconciles to EKS → Prometheus/Grafana monitor.

**Tool choices and the *why*:**
- **Bitbucket** — enterprise SSO, branch protection, PR workflow. Could swap for GitHub Enterprise.
- **Jenkins** — mature, extensible via shared libraries, strong fit for regulated enterprises. Self-hosted means data residency control.
- **SonarQube** — code quality + security hotspots; quality gates fail the build before bad code progresses.
- **JFrog Artifactory** — single artifact store (Docker, Maven, Helm, npm), promotion semantics between repos (dev → staging → prod).
- **Helm** — templating + versioning for K8s manifests; rollback is one command.
- **ArgoCD (GitOps)** — Git as source of truth, drift detection, auditability, cluster pulls instead of pipelines pushing prod creds around.
- **Prometheus + Grafana + Loki** — cloud-native observability; integrates natively with K8s; cheap to run.
- **Trivy** — image vulnerability scan in the pipeline; gates on critical/high CVEs.

The principle: **build once, promote the same artifact**. Never rebuild for prod.

## 2. Jenkins pipeline for multi-environment deployment (dev/stage/prod)

The mental model: one Jenkinsfile, parameterized by environment, with progressively stricter gates.

```groovy
@Library('platform-shared-lib@v2.1.0') _

pipeline {
  agent { label 'docker-builder' }

  options {
    timestamps()
    timeout(time: 60, unit: 'MINUTES')
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '50'))
  }

  environment {
    IMAGE   = "artifactory.company.com/myapp"
    TAG     = "${env.BRANCH_NAME}-${env.GIT_COMMIT.take(7)}-${env.BUILD_NUMBER}"
  }

  stages {
    stage('Build & Test') {
      steps {
        sh 'mvn -B clean verify'
        junit 'target/surefire-reports/*.xml'
      }
    }

    stage('SonarQube') {
      steps {
        withSonarQubeEnv('sonar-prod') { sh 'mvn sonar:sonar' }
      }
    }

    stage('Quality Gate') {
      steps { timeout(time: 5, unit: 'MINUTES') { waitForQualityGate abortPipeline: true } }
    }

    stage('Build & Push Image') {
      steps {
        sh "docker build -t ${IMAGE}:${TAG} ."
        sh "trivy image --exit-code 1 --severity CRITICAL,HIGH ${IMAGE}:${TAG}"
        withCredentials([usernamePassword(credentialsId: 'jfrog', usernameVariable: 'U', passwordVariable: 'P')]) {
          sh "echo \$P | docker login artifactory.company.com -u \$U --password-stdin"
          sh "docker push ${IMAGE}:${TAG}"
        }
      }
    }

    stage('Deploy Dev') {
      when { branch 'develop' }
      steps { deployToEnv(env: 'dev', tag: env.TAG) }
    }

    stage('Deploy Staging') {
      when { branch 'main' }
      steps { deployToEnv(env: 'staging', tag: env.TAG) }
    }

    stage('Approval — Prod') {
      when { branch 'main' }
      steps {
        timeout(time: 24, unit: 'HOURS') {
          input message: 'Promote to production?', submitter: 'release-managers'
        }
      }
    }

    stage('Deploy Prod') {
      when { branch 'main' }
      steps { deployToEnv(env: 'prod', tag: env.TAG) }
    }
  }

  post {
    failure { notifySlack(channel: '#deploy-alerts', status: 'FAILED') }
    success { notifySlack(channel: '#deploy-alerts', status: 'OK') }
  }
}
```

Key points: separate credentials per environment, separate Kubernetes namespaces (or clusters), manual approval gate for prod, the same image promoted across environments. In a GitOps model, the "deploy" stage just bumps the image tag in the env-specific values file and ArgoCD does the rest.

## 3. Freestyle vs Pipeline jobs in Jenkins

- **Freestyle** — UI-configured, single-job, limited flow control. Useful for simple one-shot tasks. Not version-controlled. Hard to maintain at scale.
- **Pipeline** — code-defined (Jenkinsfile), supports stages, parallel execution, conditional logic, shared libraries, restartable from a stage, full version control. Two flavors: **Declarative** (structured, easier) and **Scripted** (full Groovy, more powerful).

Rule for any serious team: **Pipeline only**. Freestyle is fine for a personal experimental Jenkins, not production.

## 4. Handling Jenkins pipeline failures — real example

**Scenario I handled:** Pipeline started failing intermittently at the `mvn deploy` stage with `Connection reset by peer` to Artifactory. Failures were random — same code, same commit, sometimes pass, sometimes fail.

**How I diagnosed:**
1. Pulled the last 50 build logs, noticed failures clustered on certain agents.
2. Checked agent network: those agents were in a subnet with a stale NAT rule, dropping long-lived TCP connections to Artifactory after ~60s.
3. Maven uploads of large artifacts (~200MB) exceeded the timeout.

**Fix layers:**
- Immediate: tagged the affected agents offline, restarted pipelines on healthy agents.
- Short-term: added retry logic in the pipeline: `retry(3) { sh 'mvn deploy' }`.
- Permanent: networking team fixed the NAT timeout; we also moved Artifactory access to a dedicated VPC peering with no NAT in the path.

**General playbook for Jenkins failures:**
- Look at the *console output* and identify the failing stage.
- Reproduce locally if possible (Docker agent, same image).
- Differentiate transient (network, agent capacity, flaky test) vs deterministic (code regression, dep bump).
- For transient → retry with backoff + alert if retry exhausted. For deterministic → fix in code, never `retry` over a real bug.
- Pin everything: agent images, tool versions, dependency versions, plugin versions. "Works on my machine" usually traces back to an unpinned version.

## 5. Integrating SonarQube

```groovy
stage('SonarQube Analysis') {
  steps {
    withSonarQubeEnv('sonar-prod') {
      sh '''
        mvn sonar:sonar \
          -Dsonar.projectKey=myapp \
          -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml \
          -Dsonar.qualitygate.wait=true
      '''
    }
  }
}
stage('Quality Gate') {
  steps {
    timeout(time: 5, unit: 'MINUTES') {
      waitForQualityGate abortPipeline: true
    }
  }
}
```

Setup: install SonarQube Scanner plugin in Jenkins, configure the SonarQube server in *Manage Jenkins → System*, store the SonarQube auth token in Jenkins credentials. On the SonarQube side, configure a webhook back to Jenkins so quality gate results post asynchronously.

I always pair this with **JaCoCo** (coverage) and a **Quality Gate** that fails on: new bugs > 0, new vulnerabilities > 0, new code coverage < 80%, duplication on new code > 3%.

## 6. Production-ready Dockerfile — best practices

```dockerfile
# Build stage
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /build
COPY pom.xml .
RUN mvn -B dependency:go-offline
COPY src ./src
RUN mvn -B clean package -DskipTests

# Runtime stage
FROM eclipse-temurin:17-jre-alpine
RUN addgroup -S app && adduser -S -G app app
WORKDIR /app
COPY --from=build --chown=app:app /build/target/*.jar app.jar
USER app
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java","-XX:+UseContainerSupport","-XX:MaxRAMPercentage=75","-jar","/app/app.jar"]
```

Checklist:
- Multi-stage build (smaller final image, no build tools in runtime)
- Pinned base image tag (never `latest`)
- Non-root user
- `.dockerignore` excluding `.git`, secrets, `target/`, `node_modules`
- Layer ordering: deps before source (maximizes cache hits)
- Single concern per image
- `HEALTHCHECK` defined
- No secrets in layers — BuildKit `--mount=type=secret` for build-time secrets
- Scanned by Trivy in CI; signed with Cosign for prod

## 7. CMD vs ENTRYPOINT

- **ENTRYPOINT** — the executable that always runs when the container starts. Hard to override (needs `--entrypoint`).
- **CMD** — default arguments to ENTRYPOINT. Easily overridden by passing args to `docker run`.

Idiomatic pattern: `ENTRYPOINT ["java","-jar","app.jar"]` + `CMD ["--spring.profiles.active=prod"]`. The container always runs `java -jar app.jar`; the profile can be changed at runtime.

If you only use `CMD ["java","-jar","app.jar"]`, then `docker run image bash` replaces the whole command — convenient for debugging but risky in production (people might accidentally override). ENTRYPOINT + CMD is the production pattern.

## 8. What is Docker Compose and where I've used it

Docker Compose is a tool for defining and running multi-container apps on a single host via a `docker-compose.yml` file. One command spins up the full stack: app + DB + cache + reverse proxy on a shared network.

```yaml
services:
  app:
    build: .
    ports: ["8080:8080"]
    environment:
      - DB_URL=jdbc:postgresql://db:5432/myapp
    depends_on: { db: { condition: service_healthy } }
  db:
    image: postgres:16
    environment: { POSTGRES_DB: myapp, POSTGRES_PASSWORD: dev }
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
    volumes: [pgdata:/var/lib/postgresql/data]
volumes:
  pgdata:
```

Where I've used it: **local development** (one command spins up the whole stack), **integration tests in CI** (test containers ephemeral), **demo environments** for small internal tools. Not for production-scale — that's Kubernetes territory.

## 9. Container orchestration — what and why

Orchestration manages the lifecycle of containers across a fleet of machines: scheduling, scaling, networking, service discovery, self-healing, rolling updates, secret distribution.

**Why it matters:**
- **Scheduling** — Kubernetes places containers on nodes based on resources, affinity, taints, topology, instead of you running `docker run` per host.
- **Self-healing** — crashed containers auto-restart, unhealthy ones get replaced, dead nodes have their pods rescheduled.
- **Scaling** — HPA scales pods on metrics; cluster autoscaler scales nodes.
- **Service discovery** — Services + DNS so pods find each other without hardcoded IPs.
- **Rolling updates** — replace pods gradually with rollback on failure.
- **Declarative config** — describe desired state; the orchestrator reconciles.

Without orchestration at scale, you're writing custom scripts for everything K8s gives you out of the box, and reliability suffers.

## 10. Pods, Deployments, Services in Kubernetes

- **Pod** — smallest deployable unit; one or more co-located containers sharing network and storage. Ephemeral; if a pod dies, it's gone (replaced, not restarted in place).
- **Deployment** — declarative manager for stateless apps. Defines desired pod template + replica count, manages ReplicaSets, supports rolling updates and rollback.
- **Service** — stable virtual IP + DNS name for a set of pods. Decouples consumers from pod IPs (which change). Types: ClusterIP (internal), NodePort, LoadBalancer, ExternalName, Headless.

Layered relationship: Deployment → ReplicaSet → Pods → containers; Service routes traffic to Pods via label selectors.

## 11. Rolling update with a YAML file

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1            # at most 1 extra pod during rollout
      maxUnavailable: 0      # never go below desired count
  selector:
    matchLabels: { app: myapp }
  template:
    metadata:
      labels: { app: myapp }
    spec:
      containers:
      - name: myapp
        image: myacr.azurecr.io/myapp:1.4.2
        readinessProbe:
          httpGet: { path: /health/ready, port: 8080 }
          periodSeconds: 5
        livenessProbe:
          httpGet: { path: /health/live, port: 8080 }
          periodSeconds: 10
      terminationGracePeriodSeconds: 30
```

Trigger an update by changing the image tag and `kubectl apply -f deploy.yaml`. K8s spins up one new pod (maxSurge: 1), waits for readiness, then terminates one old pod, repeating. Rollback: `kubectl rollout undo deployment/myapp`. Status: `kubectl rollout status deployment/myapp`. The two critical knobs are `maxSurge`/`maxUnavailable` and proper readiness probes — without readiness, K8s sends traffic to pods that aren't ready, breaking the rollout's zero-downtime guarantee.

## 12. ConfigMap vs Secret

- **ConfigMap** — non-sensitive configuration (feature flags, log levels, URLs). Stored plain in etcd.
- **Secret** — sensitive data (passwords, tokens, certs). Base64-encoded (which is *not* encryption); encryption requires `--encryption-provider-config` on the API server or an external KMS.

Usage:

```yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: app-config }
data:
  LOG_LEVEL: "info"
  FEATURE_NEW_CHECKOUT: "true"
---
apiVersion: v1
kind: Secret
metadata: { name: app-secret }
type: Opaque
stringData:
  DB_PASSWORD: "rotate-me"
---
spec:
  containers:
  - name: app
    image: myapp:1.0
    envFrom:
    - configMapRef: { name: app-config }
    - secretRef: { name: app-secret }
```

For production secrets I avoid native K8s secrets in favor of **External Secrets Operator** or **CSI Secrets Store driver** pulling from Azure Key Vault / AWS Secrets Manager / HashiCorp Vault — secrets stay in the dedicated KMS, are rotated automatically, and never live in Git or unencrypted etcd.

## 13. Resource limits and requests

- **requests** — guaranteed reservation; the scheduler uses this for placement.
- **limits** — hard cap; CPU is throttled when exceeded, memory triggers OOMKill.

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "512Mi"
  limits:
    cpu: "1000m"
    memory: "1Gi"
```

Principles I apply:
- **Always set both**. Unbounded pods cause noisy-neighbor problems.
- **Memory limit = request** (memory is not compressible — set them equal so pods get OOMKilled deterministically instead of randomly slowed down).
- **CPU request realistic, CPU limit optional / generous**. Hard CPU limits can cause weird throttling; many SRE teams (Netflix and others) recommend not setting CPU limits at all for latency-sensitive services and relying on `requests` for scheduling.
- Use **LimitRanges** per namespace to set defaults and prevent unbounded pods.
- Use **ResourceQuotas** at the namespace level to cap team consumption.
- Tune based on real telemetry (Goldilocks, VPA recommendations, or Grafana panels showing actual usage vs requests).

## 14. Cloud provider and DevOps services

I've worked across AWS and Azure. Quick map:

| Capability | AWS | Azure |
|---|---|---|
| Compute | EC2, ECS, EKS, Lambda | VM, AKS, App Service, Functions |
| IaC | CloudFormation, CDK + Terraform | ARM, Bicep + Terraform |
| CI/CD | CodePipeline, CodeBuild, CodeDeploy | Azure DevOps Pipelines, GitHub Actions |
| Container Registry | ECR | ACR |
| Secrets | Secrets Manager, Parameter Store | Key Vault |
| Monitoring | CloudWatch, X-Ray | Azure Monitor, Application Insights, Log Analytics |
| Identity | IAM, IAM Identity Center | Entra ID, Managed Identities |
| Networking | VPC, ALB, NLB | VNet, App Gateway, Front Door |

In real projects I lean on **Terraform** as the IaC layer regardless of provider (multi-cloud-portable skill, single tool), **EKS/AKS** for orchestration, **CloudWatch + Prometheus** or **Azure Monitor + Prometheus** as a hybrid observability stack.

## 15. Managing infrastructure with Terraform

Standard structure I use:

```
repo/
├── modules/
│   ├── network/      # VPC/VNet, subnets, NSGs
│   ├── eks-aks/      # cluster
│   ├── rds-sql/      # database
│   └── monitoring/   # log workspace, alerts
└── environments/
    ├── dev/
    │   ├── main.tf       # calls modules
    │   ├── backend.tf    # remote state
    │   └── terraform.tfvars
    ├── staging/
    └── prod/
```

Workflow:
1. PR with Terraform changes → CI runs `fmt`, `validate`, `tflint`, `tfsec`, `plan` → plan posted to the PR.
2. Reviewer approves → merge.
3. Pipeline applies on merge with the saved plan: `terraform plan -out=plan.tfplan && terraform apply plan.tfplan`.
4. Separate state per environment; state in S3/Azure Storage with locking via DynamoDB/blob lease.
5. Provider versions and Terraform version pinned in `versions.tf`.

For prod, manual approval gate before apply; never `-auto-approve` directly to prod.

## 16. Terraform backend + remote state with locking

The **backend** defines where state is stored. Local backend = `terraform.tfstate` on disk (fine for solo learning, terrible for teams). Remote backend stores state in shared, durable storage.

```hcl
# AWS
terraform {
  backend "s3" {
    bucket         = "company-tfstate"
    key            = "prod/network/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "tfstate-locks"   # state locking
    encrypt        = true
    kms_key_id     = "alias/tfstate"
  }
}

# Azure
terraform {
  backend "azurerm" {
    resource_group_name  = "tfstate-rg"
    storage_account_name = "companytfstate"
    container_name       = "tfstate"
    key                  = "prod/network.tfstate"
    use_azuread_auth     = true
  }
}
```

Yes, I always use remote state with locking. Locking prevents two engineers (or pipelines) from running `apply` concurrently — without it, you get state corruption. S3 + DynamoDB is the canonical AWS pattern; Azure Storage uses blob leases natively. Also enable **versioning** on the bucket/container so state can be recovered if corrupted.

## 17. Storing secrets in cloud pipelines

Patterns by maturity:

- **Worst:** plain text in pipeline YAML or repo.
- **Better:** pipeline-native secret stores (Jenkins Credentials, GitHub Encrypted Secrets, Azure DevOps Variable Groups). Good for low-value secrets.
- **Best (production):** dedicated KMS — **Azure Key Vault**, **AWS Secrets Manager**, **HashiCorp Vault**. Pipeline authenticates via **OIDC federation** (no long-lived credentials anywhere) and pulls secrets at runtime.

Concrete example — GitHub Actions to Azure Key Vault using OIDC:

```yaml
permissions:
  id-token: write
  contents: read
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: azure/login@v2
      with:
        client-id:       ${{ secrets.AZURE_CLIENT_ID }}
        tenant-id:       ${{ secrets.AZURE_TENANT_ID }}
        subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
    - uses: azure/get-keyvault-secrets@v1
      with:
        keyvault: "kv-prod-pipeline"
        secrets: "db-password, api-key"
      id: kv
    - run: ./deploy.sh
      env:
        DB_PASSWORD: ${{ steps.kv.outputs.db-password }}
```

Key principles: never log secrets, scope service principals to least privilege, rotate regularly (or use short-lived credentials), audit secret access (Key Vault diagnostic logs to Log Analytics).

## 18. Auto-scaling group with Terraform

AWS example:

```hcl
resource "aws_launch_template" "app" {
  name_prefix   = "app-"
  image_id      = data.aws_ami.amazon_linux_2.id
  instance_type = "t3.medium"
  vpc_security_group_ids = [aws_security_group.app.id]
  iam_instance_profile { name = aws_iam_instance_profile.app.name }
  user_data = base64encode(file("bootstrap.sh"))
  lifecycle { create_before_destroy = true }
}

resource "aws_autoscaling_group" "app" {
  name                = "app-asg"
  vpc_zone_identifier = aws_subnet.private[*].id
  min_size            = 3
  max_size            = 20
  desired_capacity    = 3
  health_check_type   = "ELB"
  health_check_grace_period = 120
  target_group_arns   = [aws_lb_target_group.app.arn]

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  tag {
    key                 = "Name"
    value               = "app-asg"
    propagate_at_launch = true
  }

  lifecycle { create_before_destroy = true }
}

resource "aws_autoscaling_policy" "scale_out" {
  name                   = "scale-out-on-cpu"
  autoscaling_group_name = aws_autoscaling_group.app.name
  policy_type            = "TargetTrackingScaling"

  target_tracking_configuration {
    predefined_metric_specification { predefined_metric_type = "ASGAverageCPUUtilization" }
    target_value = 60.0
  }
}
```

Azure VMSS equivalent uses `azurerm_linux_virtual_machine_scale_set` + `azurerm_monitor_autoscale_setting`. For Kubernetes workloads I'd use **HPA + Cluster Autoscaler** (or **Karpenter** on EKS) instead of raw ASGs.

## 19. RBAC in Jenkins and Kubernetes

**Jenkins:**
- Install **Role-based Authorization Strategy** plugin.
- Define global roles (admin, developer, read-only) and project roles (per-team folder permissions).
- Integrate with **LDAP / SSO** (SAML, OIDC) so groups in your IdP map to Jenkins roles — no per-user assignment.
- Use **Folders** to group jobs and apply role bindings at folder level.
- Audit plugin to log who triggered what.

**Kubernetes RBAC:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: team-payments
  name: developer
rules:
- apiGroups: ["", "apps"]
  resources: ["pods", "pods/log", "deployments", "services"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developers-binding
  namespace: team-payments
subjects:
- kind: Group
  name: oidc:payments-developers     # from OIDC provider
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

Patterns I follow:
- **OIDC integration** (Azure AD, Okta, AWS IAM Identity Center) — never per-user kubeconfigs.
- **Groups, not users** — bindings on AD/IdP groups; user lifecycle handled by HR/IdM.
- **Namespace as the unit of isolation** — one team or app per namespace, Roles scoped to namespace.
- **ClusterRoles sparingly** — only for cluster-wide things (storage classes, namespaces).
- **Audit logs** to a SIEM; alert on suspicious patterns (escalation, cluster-admin bindings).

## 20. Rollback a faulty deployment with Git + CI

Multiple layers depending on your model:

**Imperative (kubectl-based):**
```bash
kubectl rollout undo deployment/myapp        # revert to previous revision
kubectl rollout undo deployment/myapp --to-revision=3
```

**Helm:**
```bash
helm rollback myapp 0      # previous release
helm history myapp
```

**GitOps (preferred):**
```bash
git revert <bad-commit>
git push
# ArgoCD reconciles to previous state automatically
```

**Blue-green:** flip the service selector back to blue.

**Argo Rollouts canary:** if SLO metrics regress during canary analysis, the rollout aborts and routes 100% back to stable automatically — no human in the loop.

The principle: **the safest rollback is the one you don't have to think about**. Git revert + GitOps reconciliation is my default because it leaves a perfect audit trail and works the same way every time.

## 21. Gitflow and branching strategies

**Gitflow** has long-lived branches: `main` (production), `develop` (integration), `feature/*`, `release/*`, `hotfix/*`. Heavyweight; suits versioned products with scheduled releases.

In a DevOps/CD context I usually push back against Gitflow because it works against continuous deployment. Alternatives:

- **Trunk-based development** — short-lived feature branches (hours-to-days), merged to `main` behind feature flags. `main` is always deployable. Pairs naturally with CI/CD.
- **GitHub Flow** — `main` + short feature branches + PRs. Simple.
- **Release branches** — only when needing to support multiple production versions in parallel (rare in SaaS).

In one team I worked with: trunk-based + feature flags + ephemeral PR preview environments per branch. Deploys to production multiple times per day, zero merge conflicts, fast feedback. The strategy matched the cadence; you should never adopt Gitflow because "it's the standard" — pick what matches your release cadence and team size.

## 22. Logging and monitoring setup

**Architecture I deploy:**

| Layer | Tool |
|---|---|
| Metrics collection | Prometheus (or managed Prometheus / Azure Monitor) |
| Metric storage | Prometheus TSDB or Thanos/Mimir for long-term |
| Log collection | Fluent Bit DaemonSet |
| Log storage | Loki (cheap, label-based) or ELK (full-text search) |
| Traces | Tempo or Application Insights via OpenTelemetry |
| Dashboards | Grafana |
| Alerts | Alertmanager → PagerDuty + Slack |
| Synthetics | Blackbox exporter, CloudWatch Synthetics |
| APM | Application Insights / Datadog APM |

**Setup pattern on K8s:** install `kube-prometheus-stack` via Helm (Prometheus + Alertmanager + Grafana + node-exporter + kube-state-metrics + default dashboards), add Loki + Promtail, then app-side instrumentation (Micrometer, OpenTelemetry SDK).

**What I instrument:**
- Infra: node CPU/mem/disk/network, K8s pod restarts, PV usage, etcd latency.
- Platform: Deployment rollout status, HPA activity, ingress request rate/error rate/latency.
- Application: RED metrics (Rate, Errors, Duration), business KPIs (orders/sec, etc.), p95/p99 latency.
- SLOs and burn-rate alerts (Google SRE workbook style).

**Alerting hygiene:** every alert links to a runbook, is actionable, and has a clear owner. No "informational" alerts paging on-call. Page on user-visible symptoms (SLO burn), ticket on causes (high CPU). Test alerts via game days.

## 23. DevSecOps practices — example

DevSecOps = security as a shared responsibility, automated and shifted left into the pipeline.

**A specific example I implemented:** for a regulated workload, we built security into every pipeline stage:

1. **Pre-commit** — `gitleaks` and `pre-commit` hooks catching secrets before commit.
2. **PR build** — SAST (SonarQube + CodeQL), dependency scan (Snyk / OWASP Dependency-Check), IaC scan (`tfsec`, `checkov`), license scan (FOSSA).
3. **Container build** — Trivy CVE scan (block CRITICAL/HIGH), Dockerfile linter (hadolint), Cosign signing.
4. **Pre-deploy** — Helm chart scan (`kubeaudit`, `kubesec`), policy check against Open Policy Agent / Kyverno policies.
5. **Admission** — Kyverno verifies image signatures and blocks non-compliant manifests at the cluster boundary.
6. **Runtime** — Falco for runtime anomaly detection, Defender for Containers for cluster posture.
7. **Continuous** — DAST (OWASP ZAP) against staging weekly, registry rescans for newly-disclosed CVEs.

**Cultural piece:** security findings appear in the developer's PR (where they can fix them), not in a quarterly report. Security team writes policies and approves exceptions but doesn't gatekeep day-to-day work. Threat modeling happens in design reviews. Postmortems include a security-impact section.

---

# 2️⃣ HCL Terraform Deep Dive

## 1. Terraform state — what and why critical

State is Terraform's record of the resources it manages — a JSON file mapping resource addresses in your code (`aws_instance.web`) to real-world resource IDs (`i-0abc123…`) along with attributes and metadata.

**Why it's critical:**
- **Mapping** — without state, Terraform can't know which real resource corresponds to which code block.
- **Performance** — caches attributes so plans don't have to read every resource from the cloud API every time.
- **Dependency ordering** — state holds the DAG that determines apply/destroy order.
- **Drift detection** — `terraform plan` compares state vs cloud actuals to surface drift.
- **Concurrency safety** — locking on state prevents concurrent applies from corrupting infra.
- **Sensitive data** — state contains raw resource attributes including passwords, keys, certs. This is *the* reason state must be encrypted and access-controlled.

Losing state = losing the ability to manage that infrastructure with Terraform without rebuilding via imports.

## 2. Terraform vs ARM/CloudFormation — enterprise perspective

| | Terraform | ARM / CloudFormation |
|---|---|---|
| Cloud support | Multi-cloud + SaaS providers | Single cloud (Azure / AWS) |
| Language | HCL (purpose-built) | JSON/YAML (ARM/CFN) or Bicep/CDK |
| State | Explicit, managed by Terraform | Managed by the cloud provider |
| Maturity of ecosystem | Massive (Registry, modules) | Solid but provider-locked |
| Drift detection | First-class (`plan`) | Limited (CFN Drift Detection; Bicep what-if) |
| Modularity | Modules, easy reuse | Nested templates / linked templates / Bicep modules |
| Day-2 operations | Strong (import, refresh, state mv) | Provider-dependent |
| Vendor lock-in | Low (HashiCorp; can replace with OpenTofu) | High |
| Learning curve | Moderate | Provider-specific |

**Enterprise considerations:**
- Multi-cloud or hybrid? Terraform is the only realistic choice.
- Azure-only and strategically Microsoft? Bicep is increasingly compelling — better Azure feature parity at launch.
- AWS-only? CDK is gaining traction (TypeScript/Python). CloudFormation YAML is mature but verbose.
- For HCL Technologies-type system integrators serving multiple cloud clients, Terraform skill is the universal currency.

Note: HashiCorp's license change in 2023 spawned **OpenTofu**, an MIT-licensed fork, which now matters for some enterprises. Worth mentioning in interview to show you're current.

## 3. What happens during init, plan, apply

**`terraform init`:**
- Reads the configuration to find `terraform {}` block (required version, backend, providers).
- Initializes the backend — connects to S3/Azure Storage/etc., creates the state file if absent, locks it briefly.
- Downloads providers (e.g., `hashicorp/aws`) into `.terraform/providers`.
- Downloads modules referenced in code.
- Writes `.terraform.lock.hcl` (provider checksums for reproducibility).

**`terraform plan`:**
- Acquires state lock.
- Reads current state from backend.
- *Refreshes* state by querying the cloud API for current attributes of tracked resources (this is what surfaces drift).
- Parses configuration, builds desired-state DAG.
- Computes the diff (current state vs desired config).
- Outputs the plan: + add, ~ change, - destroy, with attribute-level detail.
- Optionally writes a binary plan file (`-out=plan.tfplan`) that can be applied later.

**`terraform apply`:**
- Acquires state lock (if not already held from a passed plan file).
- If no plan file passed, generates a plan and prompts for approval.
- Walks the DAG in dependency order, calling provider CRUD operations.
- Updates state after each resource operation (so a partial apply still leaves an accurate state).
- Releases lock.

A subtle but important detail: state is updated *incrementally* during apply, so a crash or network blip leaves state reflecting what actually got created. This is why state corruption is rare in practice — it's robust to interruptions.

## 4. Resource dependency handling

Two kinds:

**Implicit** — when one resource references another's attribute, Terraform infers the dependency automatically and orders the DAG accordingly:

```hcl
resource "aws_vpc" "main" { cidr_block = "10.0.0.0/16" }

resource "aws_subnet" "a" {
  vpc_id     = aws_vpc.main.id    # implicit dependency on aws_vpc.main
  cidr_block = "10.0.1.0/24"
}
```

**Explicit** — when there's no attribute reference but ordering matters, use `depends_on`:

```hcl
resource "aws_iam_role_policy" "example" { ... }

resource "aws_instance" "web" {
  depends_on = [aws_iam_role_policy.example]
  ...
}
```

The DAG is built once per plan; Terraform parallelizes operations on independent nodes (controlled by `-parallelism=N`, default 10). On destroy, the DAG is reversed.

`depends_on` should be a last resort — prefer attribute references because they're declarative and the dependency travels with the code. Overusing `depends_on` makes the DAG unnecessarily serialized and slow.

## 5. Providers and version locking

A **provider** is the plugin that translates Terraform resource declarations into cloud API calls. `hashicorp/aws`, `hashicorp/azurerm`, `hashicorp/kubernetes`, `cloudflare/cloudflare`, etc.

```hcl
terraform {
  required_version = ">= 1.6.0, < 2.0.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.40"      # >= 5.40, < 6.0
    }
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.100"
    }
  }
}
```

`terraform init` resolves these constraints, picks the highest compatible version, downloads it, and records the *exact* version + checksum in `.terraform.lock.hcl`. The lockfile must be committed to Git — it pins providers across team members and CI for reproducibility.

To upgrade: `terraform init -upgrade`, review the lock file diff in a PR, run plan, verify no surprise changes.

`~>` (pessimistic) is the standard pattern — allows patch updates automatically, blocks unexpected major/minor jumps.

---

## 🧱 State & Backend

## 6. Local vs remote backend

| | Local | Remote |
|---|---|---|
| Storage | `terraform.tfstate` on disk | S3, Azure Storage, GCS, TF Cloud, Consul |
| Locking | None (or file-based) | Yes (DynamoDB, blob lease, native) |
| Team support | No (state lives on one machine) | Yes |
| Versioning | No | Yes (with versioning on the bucket) |
| Encryption | No (plain file) | Yes (KMS/SSE) |
| Audit | No | Yes (bucket access logs) |
| Use case | Local learning, throwaway demos | Anything real |

For any team larger than one, remote backend is mandatory — local state is unsafe (no concurrency control, secrets in plain file on a laptop, loss when the laptop dies).

## 7. State locking in team environments

**S3 + DynamoDB (AWS):**
```hcl
backend "s3" {
  bucket         = "company-tfstate"
  key            = "prod/network.tfstate"
  region         = "us-east-1"
  dynamodb_table = "tfstate-locks"
  encrypt        = true
}
```
DynamoDB table needs a primary key `LockID` (string). When `apply` starts, Terraform writes a lock record; concurrent applies see it and wait or fail.

**Azure Storage:** locking is built into blob leases — no extra resources required, just configure the `azurerm` backend.

**Terraform Cloud / Enterprise:** native locking + UI + run queue.

If a lock gets stuck (process killed mid-apply), `terraform force-unlock <lock-id>` releases it — but only after confirming no one is actually mid-apply. Operational discipline: don't run apply from laptops in shared environments; use a single CI runner as the only thing that touches state.

## 8. State drift — detection and fix

**Drift** = the real cloud state diverges from Terraform's state. Causes:
- Someone clicked in the console (`ClickOps`).
- Another tool (Ansible, CloudFormation, manual API call) modified the resource.
- Auto-scaling changed instance count.
- A cloud-provider-side change (forced upgrade).

**Detection:**
- `terraform plan` will show `Note: Objects have changed outside of Terraform` and list the drift.
- Scheduled CI job: `terraform plan -detailed-exitcode` (exit code 2 = changes detected). Alert if drift found.
- **driftctl** (open source) for continuous drift scanning across providers.
- Terraform Cloud has built-in drift detection.

**Fix options:**
1. **Adopt the drift** — update the Terraform code to match reality if the manual change was intentional and correct. Then `terraform apply` makes the plan a no-op.
2. **Revert the drift** — `terraform apply` to push the original code's state back onto the resource, undoing the manual change.
3. **Re-import** if the resource was recreated outside Terraform: `terraform import` to re-link.

Cultural fix: lock down `*Action` permissions in IAM so engineers can't modify Terraform-managed resources outside the pipeline. Make Terraform the only path for changes.

## 9. `terraform import` — when and how

Use cases:
- Adopting existing infrastructure into Terraform (greenfield team takes over brownfield infra).
- Recovering from accidental `terraform state rm` or state corruption.
- Resources created out-of-band that you now want to manage.

How:
1. Write the resource block in Terraform first — empty if needed.
2. Run `terraform import <resource_address> <real_world_id>`:
   ```bash
   terraform import aws_instance.web i-0abc123def456
   terraform import azurerm_resource_group.rg /subscriptions/<sub>/resourceGroups/myrg
   ```
3. Run `terraform plan` — usually shows a non-trivial diff because your config doesn't match all the actual attributes.
4. Update the Terraform code until `plan` shows no changes. Iterate.

Modern Terraform (1.5+) supports **`import` blocks** in HCL for declarative import:
```hcl
import {
  to = aws_instance.web
  id = "i-0abc123def456"
}
```
This is reviewable in PRs and runs as part of `apply`, much cleaner than the imperative CLI command.

For large existing estates, **terraformer** (open source from GoogleCloudPlatform) can scan a cloud and generate Terraform code + state for thousands of resources in one shot — useful starting point, but always review the output.

## 10. Risks of manually editing `.tfstate`

State is a JSON file; theoretically you can edit it. **Don't.** Risks:
- **Corruption** — syntax error, wrong type, broken reference; subsequent operations fail.
- **Data loss** — if you delete an entry, Terraform forgets the resource exists; next plan tries to recreate it (potentially destroying live infrastructure).
- **Lock bypass** — editing the file directly bypasses backend locking; concurrent edits conflict.
- **Provider schema mismatch** — state schema is provider-version-specific; hand-edits often violate invariants the provider enforces.
- **No audit trail** — there's no record of what changed and why.

The right tools:
- `terraform state mv` — rename/move resources (e.g., when refactoring code into modules).
- `terraform state rm` — remove a resource from state without destroying it.
- `terraform state replace-provider` — swap providers.
- `terraform import` — add a resource to state.
- `terraform state pull` / `push` — for backend migration, with extreme care.

If you absolutely must hand-edit: `pull` the state, edit a copy, validate the JSON, take a backup, `push` it back. Run a `plan` immediately to verify. But seriously — try the supported `terraform state` subcommands first.

---

## 🧩 Modules & Reusability

## 11. Terraform modules — what and why

A module is a packaged, parameterized collection of Terraform resources with a defined input/output interface. Modules let you:
- **DRY** — write a VPC once, reuse across 20 environments.
- **Encapsulate** — module exposes a clean interface; internals are hidden.
- **Version** — pin to a specific module version; upgrade deliberately.
- **Compose** — build environments by combining modules.
- **Govern** — central platform team owns hardened modules with policies baked in (encryption on, tagging required, etc.); app teams consume them.

Every Terraform configuration is technically a module — the "root module" is what you run `terraform apply` against. **Child modules** are called from the root via `module` blocks.

Layout of a well-built module:
```
modules/network/
├── README.md
├── main.tf
├── variables.tf
├── outputs.tf
├── versions.tf
└── examples/
    └── basic/
        └── main.tf
```

## 12. Passing variables between root and child modules

Inputs flow root → child via `module` block arguments. Outputs flow child → root via `output` blocks consumed as `module.<name>.<output>`.

```hcl
# modules/network/variables.tf
variable "cidr_block"  { type = string }
variable "azs"         { type = list(string) }
variable "tags"        { type = map(string)  default = {} }

# modules/network/outputs.tf
output "vpc_id"             { value = aws_vpc.main.id }
output "private_subnet_ids" { value = aws_subnet.private[*].id }

# environments/prod/main.tf
module "network" {
  source     = "../../modules/network"
  cidr_block = "10.0.0.0/16"
  azs        = ["us-east-1a", "us-east-1b", "us-east-1c"]
  tags       = { env = "prod", owner = "platform" }
}

module "eks" {
  source     = "../../modules/eks"
  vpc_id     = module.network.vpc_id
  subnet_ids = module.network.private_subnet_ids
}
```

Tips:
- Validate inputs: `validation {}` blocks on variables.
- Mark sensitive outputs: `sensitive = true`.
- Don't expose every internal — outputs are the public API of the module; keep it minimal.

## 13. Module versioning and reuse across environments

**Versioning:** modules live in their own Git repo (or Terraform Registry / private registry) and are tagged with SemVer:

```hcl
module "network" {
  source = "git::https://github.com/company/tf-modules.git//network?ref=v2.4.0"
  # or with Terraform Registry:
  # source  = "company/network/aws"
  # version = "~> 2.4"
  ...
}
```

`ref` pins to a tag. Never use `ref=main` in prod — that means the module can change under your feet between plans.

**Reuse across environments:** same module, different inputs per environment.
```hcl
# environments/dev/main.tf
module "network" {
  source     = "git::.../tf-modules.git//network?ref=v2.4.0"
  cidr_block = "10.10.0.0/16"
  instance_type = "t3.small"
}

# environments/prod/main.tf
module "network" {
  source     = "git::.../tf-modules.git//network?ref=v2.4.0"
  cidr_block = "10.0.0.0/16"
  instance_type = "m5.large"
}
```

**Upgrade flow:** module v2.5.0 released → bump in dev, run plan, verify, apply → bump in staging, repeat → finally bump in prod. Each bump is a PR with the plan reviewed.

For module development: `examples/` directory with at least one runnable example, automated tests with **Terratest** or **`terraform test`** (1.6+).

---

## 🔁 Advanced Terraform Logic

## 14. `count` vs `for_each`

Both create multiple instances of a resource, but with different semantics:

**`count`** — uses an integer; resources indexed by position (`[0]`, `[1]`, ...).
```hcl
resource "aws_subnet" "public" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]
}
```

**`for_each`** — uses a map or set of strings; resources indexed by key.
```hcl
resource "aws_iam_user" "users" {
  for_each = toset(["alice", "bob", "carol"])
  name     = each.key
}

resource "aws_subnet" "private" {
  for_each          = { for idx, az in var.azs : az => idx }
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 8, each.value + 10)
  availability_zone = each.key
}
```

**The killer difference:** if you remove `count = 5` → `count = 4`, Terraform recreates resources because indices shift — the resource at `[2]` may now be a completely different thing. With `for_each`, removing one key only affects that one resource; others are unaffected.

**Rule of thumb:**
- Use `for_each` when you have a stable, named set of things (users, subnets-per-AZ, environments).
- Use `count` for simple replication where ordering doesn't matter and the count is stable, or for boolean toggles: `count = var.enabled ? 1 : 0`.

In practice, `for_each` is the right choice 90% of the time in production code.

## 15. Dynamic blocks — real use cases

`dynamic` blocks generate nested config blocks based on a variable, replacing repetitive copy-paste.

Real example — security group rules driven by a variable:

```hcl
variable "ingress_rules" {
  type = list(object({
    description = string
    from_port   = number
    to_port     = number
    protocol    = string
    cidr_blocks = list(string)
  }))
  default = []
}

resource "aws_security_group" "app" {
  name   = "app-sg"
  vpc_id = var.vpc_id

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      description = ingress.value.description
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }
}
```

Other real use cases:
- IAM policy statements driven by a list.
- ALB listener rules for variable routing paths.
- VPC subnet associations across an arbitrary AZ list.
- Lifecycle rules on S3 buckets configured per-environment.

Tip: don't *over*use dynamic blocks. When the structure is fixed (always 3 ingress rules), static blocks are more readable. Use dynamic when the count is genuinely variable.

## 16. `lifecycle` meta-argument for production safety

The `lifecycle` block changes how Terraform manages a resource:

```hcl
resource "aws_db_instance" "prod" {
  identifier = "prod-db"
  # ...

  lifecycle {
    prevent_destroy       = true                     # apply fails if anything would destroy this
    ignore_changes        = [password, tags["LastModified"]]   # ignore drift on these
    create_before_destroy = true                     # for replace-required changes
    replace_triggered_by  = [aws_kms_key.db.arn]    # force replace when this changes
  }
}
```

Production safety patterns:
- **`prevent_destroy = true`** on critical resources (prod DBs, KMS keys, route 53 zones). Forces explicit removal of the lifecycle rule before destruction — a guardrail against accidental `apply` deletions.
- **`ignore_changes`** on attributes that drift legitimately (auto-scaling group `desired_capacity`, tags set by external automation, secrets rotated outside Terraform).
- **`create_before_destroy = true`** for resources where replacement should be zero-downtime (launch templates, certificates). Without it, Terraform deletes the old one first, leaving a gap.
- **`replace_triggered_by`** when a downstream resource needs replacement when an upstream changes.

Caveat: `prevent_destroy` blocks `terraform destroy` for that resource — useful for prod, painful for ephemeral environments. Use only where it matters.

## 17. Managing secrets securely in Terraform

The fundamental challenge: state contains every resource attribute including secrets in plain JSON.

**Patterns:**
1. **Never hardcode secrets in `.tf` files.** Obvious but worth saying.
2. **Reference from a KMS** at runtime:
   ```hcl
   data "azurerm_key_vault_secret" "db_password" {
     name         = "db-password"
     key_vault_id = data.azurerm_key_vault.this.id
   }
   resource "azurerm_postgresql_flexible_server" "this" {
     administrator_password = data.azurerm_key_vault_secret.db_password.value
   }
   ```
3. **Generate secrets in Terraform and store in KMS:** `random_password` resource → Key Vault secret. The password value still ends up in state but never in code.
4. **Encrypt state at rest** — KMS-encrypted S3 backend, customer-managed key on Azure Storage.
5. **Restrict state access** — only the pipeline service principal and a small SRE group should read state.
6. **Use `sensitive = true`** on variables and outputs containing secrets so they're not displayed in logs.
7. **Ephemeral providers/data sources** instead of resources where possible (Vault provider's `vault_generic_secret` data source).
8. **Avoid state-stored secrets entirely:** for DB passwords, use IAM authentication (RDS, Azure AD auth for Postgres/SQL); for app creds, use workload identity. The best secret is no secret.

For CI: pipeline pulls cloud credentials via OIDC federation (no long-lived secrets); Terraform itself runs with managed identity / instance profile and reads Key Vault as needed.

## 18. Terraform workspaces — enterprise recommendation

**Workspaces** allow multiple state files within a single backend, switchable via `terraform workspace select <name>`. The same config produces different infrastructure per workspace.

**Use cases:**
- Quick ephemeral environments (per-developer testing).
- Identical infra topology with different scales (small dev vs large prod).

**Why I generally don't recommend workspaces for enterprise environment separation:**
- Single backend = single set of credentials = blast radius problem (prod and dev share auth).
- Easy to run a command in the wrong workspace (`terraform apply` in `prod` when you meant `dev`).
- Workspaces are not visible in code — you can't tell from the `.tf` file which env you're in.
- HashiCorp's own docs now recommend separate directories/backends for environment separation.

**My preferred pattern:**
```
environments/
  dev/   { main.tf, backend.tf  ← backend "key" = "dev/network.tfstate" }
  prod/  { main.tf, backend.tf  ← backend "key" = "prod/network.tfstate" }
```

Each environment has its own state file, own backend config (often a different account/subscription), own credentials, own pipeline approval gates. Workspaces are useful for **feature branches** within an environment ("preview environments per PR"), not for dev-vs-prod isolation.

## 19. Terraform with CI/CD pipelines — safely

Reference pipeline:

```yaml
# .github/workflows/terraform.yml
on:
  pull_request:
    paths: ['terraform/**']
  push:
    branches: [main]
    paths: ['terraform/**']

jobs:
  validate:
    runs-on: ubuntu-latest
    permissions: { id-token: write, contents: read, pull-requests: write }
    steps:
      - uses: actions/checkout@v4
      - uses: azure/login@v2
        with:
          client-id:       ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id:       ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - uses: hashicorp/setup-terraform@v3
        with: { terraform_version: 1.7.5 }

      - run: terraform fmt -check -recursive
      - run: terraform init
      - run: terraform validate
      - uses: terraform-linters/setup-tflint@v4
      - run: tflint --recursive
      - uses: aquasecurity/tfsec-action@v1
      - uses: bridgecrewio/checkov-action@master

      - name: Plan
        if: github.event_name == 'pull_request'
        run: terraform plan -out=tfplan -no-color | tee plan.txt
      - name: Comment plan on PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const body = '```\n' + fs.readFileSync('plan.txt','utf8').slice(0, 60000) + '\n```';
            github.rest.issues.createComment({
              ...context.repo, issue_number: context.issue.number, body
            });

  apply:
    if: github.ref == 'refs/heads/main'
    needs: validate
    runs-on: ubuntu-latest
    environment: production         # required approval here
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init
      - run: terraform plan -out=tfplan
      - run: terraform apply -auto-approve tfplan
```

Safety practices:
- **OIDC federation**, no long-lived cloud credentials.
- **`fmt`, `validate`, `tflint`, `tfsec`/`checkov`** mandatory on every PR.
- **Plan posted to PR** for reviewer visibility.
- **Plan file applied directly** (`terraform apply tfplan`) so what got reviewed is what gets applied.
- **Environment-based approvals** for prod.
- **Separate state per environment**, isolated credentials.
- **Pin Terraform version** (`.terraform-version` file or `required_version`).
- **Atlantis or Terraform Cloud** for richer PR workflows (mention these as scale-up options).
- **Drift detection cron** running `plan -detailed-exitcode` nightly; alert on drift.

## 20. Rolling back Terraform changes

Terraform doesn't have a native "rollback" command. Options ordered by safety:

**1. Git revert + re-apply** (the GitOps way):
```bash
git revert <bad-commit>
git push
# Pipeline runs plan → apply, reverting infrastructure
```
This is the cleanest and produces a clean audit trail. It only works if the revert is itself a valid Terraform plan — destructive changes (a deleted resource being re-created) require the resource to be re-importable or recreatable.

**2. Roll forward** — write a new commit that fixes the broken state. Often safer than reverting because you may not want to recreate destroyed resources.

**3. State surgery** — for cases where state got into a bad shape:
- `terraform state rm` to remove an entry and re-import.
- `terraform state pull > backup.tfstate` for a safety net before any operation.
- Restore an earlier state version from S3 versioning / Azure Storage soft-delete:
  ```bash
  aws s3api list-object-versions --bucket company-tfstate --prefix prod/network.tfstate
  aws s3api get-object --bucket company-tfstate --key prod/network.tfstate --version-id <id> backup.tfstate
  terraform state push backup.tfstate
  ```

**4. Recover via cloud-native snapshots** — for stateful resources (DBs especially), Terraform rollback is meaningless if data was lost. Database point-in-time restore from RDS/Azure SQL snapshots is the real recovery path; Terraform just reconfigures around the restored resource.

**Prevention is the actual answer:**
- `prevent_destroy` lifecycle on critical resources.
- Plan reviewed in PRs — destruction lines visible in plan output should fail review.
- `terraform plan -detailed-exitcode` gating.
- Run `apply` only against a saved plan file from CI — never `apply` interactively against prod from a laptop.
- Snapshots and backups outside Terraform's control for any stateful resource.
- Sentinel / OPA policies blocking destroy operations against `env=prod` tagged resources unless an explicit override variable is set.

The honest interview answer: **don't rely on Terraform rollback as a recovery strategy.** Build the pipeline so bad changes don't reach prod, and rely on cloud-native data protection for the stateful pieces.

---

That covers both rounds. Likely HCL follow-ups:
- "Walk me through a Terraform issue you debugged in production" — have a specific story ready (state lock stuck, provider version bump breaking plan, accidental destroy averted by `prevent_destroy`).
- "How would you migrate from CloudFormation to Terraform at scale?" — terraformer for bulk import, run in parallel for a quarter, cut over by service, decommission CFN stacks last.
- "Design a Terraform structure for a 50-team org" — central platform modules repo, environment-per-folder, account-per-environment isolation, OPA policies, internal Terraform registry, Backstage portal for self-service.


# DevOps Interview — Complete Answers

Going through each with the depth a senior engineer should bring.

---

## 🐳 Docker

### 1. Main Docker network types

Docker ships with several network drivers. The main ones:

**Bridge** — the default for standalone containers. Creates a private internal network on the host using a Linux bridge (`docker0`). Containers get IPs in 172.17.0.0/16, communicate via the bridge, and reach the internet via host NAT. On user-defined bridges, containers can resolve each other by name via Docker's embedded DNS.

```bash
docker network create app-net
docker run -d --network app-net --name api myapi
docker run -d --network app-net --name db postgres
# api can reach db via hostname "db"
```

**Host** — container shares the host's network namespace. No isolation, no NAT. Binding to port 80 binds to the host's port 80. Best raw performance.

```bash
docker run --network host nginx
```

Trade-off: port conflicts, no namespace isolation. Used for performance-sensitive workloads or system-level tools.

**None** — no network at all, only loopback. Used for batch jobs or maximum isolation.

```bash
docker run --network none my-job
```

**Overlay** — multi-host network for Docker Swarm (or external orchestrators). Uses VXLAN encapsulation to span containers across multiple hosts. In Kubernetes, the equivalent role is filled by CNI plugins (Calico, Cilium, AWS VPC CNI).

**Macvlan** — container gets its own MAC address and appears as a physical device on the parent network. Container has a real IP on the LAN. Used for legacy apps that need direct LAN addressability.

**ipvlan** — similar to macvlan but containers share the parent's MAC, differentiated by IP. Better for environments with switch-level MAC limits.

**Practical view:** in production on Kubernetes, the relevant question is CNI choice, not Docker network drivers. For local dev and CI, user-defined bridge networks via Docker Compose serve most needs. Host network is occasionally useful for diagnostic tools.

### 2. Use of Dockerfile

A Dockerfile is a **declarative text file with instructions for building a Docker image**. Each instruction creates a layer; together they define how to assemble a reproducible application environment.

It serves as:
- **The recipe** — exact, version-controlled definition of what's in the image.
- **The reproducibility contract** — anyone with the Dockerfile and source can rebuild the same image.
- **The interface to CI/CD** — pipelines run `docker build` against it.
- **The documentation** — anyone reading it sees what the runtime depends on, what user it runs as, what port it exposes.

### 3. Why we use Dockerfile

Several converging reasons:

**Reproducibility** — building "the same image" on different machines means literally the same bytes, not "approximately the same." A Dockerfile pinned to specific base images and dependency versions builds the same image today, six months from now, or on any developer's laptop.

**Version control of infrastructure** — the runtime environment lives in Git alongside the code. Changes to dependencies, OS packages, or build steps are diffable, reviewable, and revertable.

**Automation** — pipelines can build images deterministically without anyone manually setting up a server. The Dockerfile is the source of truth that automation consumes.

**Immutability** — the resulting image is immutable. Once built and tagged, it doesn't change. Promoting from dev to prod is "deploy the same image" rather than "build again in prod, hope it matches."

**Separation of concerns** — app code is one thing; how to package it is another. Developers can change one without disturbing the other, and platform teams can provide standard base images that app teams inherit from.

**Audit and security** — image scanners (Trivy, Snyk, Defender) read the Dockerfile-built image and produce CVE reports tied to specific layers. Without a Dockerfile, there's no defensible audit trail of what's in production.

### 4. Dockerfile instructions

The instructions, grouped by purpose:

**Setup / metadata:**
- `FROM <image>:<tag>` — base image. Every Dockerfile starts here.
- `ARG <name>` — build-time variable, available only during build.
- `LABEL <key>=<value>` — metadata baked into the image (maintainer, version, vendor).
- `ENV <key>=<value>` — environment variable available at build time AND runtime.
- `WORKDIR <path>` — sets the working directory for subsequent instructions.
- `USER <username>` — sets which user subsequent instructions and the running container run as.

**Filesystem:**
- `COPY <src> <dest>` — copies files from the build context into the image. Simple and predictable.
- `ADD <src> <dest>` — like COPY, but can fetch URLs and auto-extract tarballs. Prefer COPY unless you specifically need ADD's tar extraction.
- `VOLUME <path>` — declares a volume mount point (advisory; actual volume managed by the runtime).

**Build-time execution:**
- `RUN <command>` — runs a command at build time and commits the result as a layer. Used to install packages, compile code.

**Runtime configuration:**
- `EXPOSE <port>` — documents the port the container listens on. Doesn't actually publish the port (the runtime does).
- `CMD <command>` — default command/arguments executed when the container starts. Easily overridden.
- `ENTRYPOINT <command>` — fixed executable that runs when the container starts. Hard to override.
- `HEALTHCHECK <command>` — periodic check the runtime uses to mark the container healthy/unhealthy.
- `STOPSIGNAL <signal>` — signal sent on container stop (default SIGTERM).
- `SHELL ["/bin/bash", "-c"]` — change the shell used for `RUN` commands.

**Multi-stage build:**
- `FROM <image> AS <stage>` — names a build stage so later stages can `COPY --from=<stage>`.

**Build-time secret handling (BuildKit):**
- `RUN --mount=type=secret,id=<id> <cmd>` — mount a secret at build time without baking it into a layer.
- `RUN --mount=type=cache,target=<path> <cmd>` — cache mount for package managers, persisted across builds without inflating the image.

**Complete example showing most instructions:**

```dockerfile
# syntax=docker/dockerfile:1.6
FROM node:20-alpine AS build
ARG BUILD_VERSION=1.0.0
LABEL maintainer="platform@company.com" \
      version="${BUILD_VERSION}"

WORKDIR /build
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm npm ci
COPY . .
RUN npm run build

FROM node:20-alpine
RUN addgroup -S app && adduser -S -G app app
WORKDIR /app
ENV NODE_ENV=production PORT=3000

COPY --from=build --chown=app:app /build/dist ./dist
COPY --from=build --chown=app:app /build/node_modules ./node_modules
COPY --chown=app:app package.json ./

USER app
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget -qO- http://localhost:3000/health || exit 1

ENTRYPOINT ["node"]
CMD ["dist/server.js"]
```

---

## ⚙️ Terraform

### 5. Execute a script even if the resource is not created

This is asking about **unconditional or independent script execution** in Terraform. Several approaches:

**Approach 1: `null_resource` with `local-exec`**

The classic pattern. `null_resource` is a "resource" that doesn't manage any infrastructure but provides hooks for provisioners.

```hcl
resource "null_resource" "run_script" {
  triggers = {
    always_run = timestamp()    # forces re-run on every apply
  }

  provisioner "local-exec" {
    command = "./scripts/notify-deploy.sh"
  }
}
```

With `triggers = { always_run = timestamp() }`, the resource is "tainted" on every apply and the provisioner runs again.

**Approach 2: `terraform_data` resource (Terraform 1.4+)**

`terraform_data` is the modern replacement for `null_resource` — same behavior, built-in, no provider needed.

```hcl
resource "terraform_data" "run_script" {
  triggers_replace = [timestamp()]

  provisioner "local-exec" {
    command = "./scripts/setup.sh"
  }
}
```

**Approach 3: External data source**

If the goal is to *run a script and capture its output* (not just side effects), use `external`:

```hcl
data "external" "lookup" {
  program = ["bash", "${path.module}/scripts/lookup.sh"]
}

output "result" {
  value = data.external.lookup.result["some_key"]
}
```

Data sources are read during plan, so this runs every plan/apply.

**Approach 4: `local_file` data sources or scripts run during `terraform plan`**

`data` blocks run during plan. Useful when you need values from a script as inputs to other resources.

**Critical caveat I'd state in the interview:** running scripts in Terraform is generally a smell. Terraform is declarative — describing *what* should exist, not *how* to make it happen. If you find yourself reaching for `local-exec` and `null_resource` frequently, that's a sign the work should move outside Terraform — into Ansible, a CI/CD pipeline, or actual cloud-native resources (Lambda, AWS Systems Manager). I use `null_resource` / `terraform_data` for narrow gaps that the Terraform ecosystem doesn't cover, and treat anything more complex as an architectural smell.

### 6. Terraform state file — what and why

**What:** Terraform's state file (`terraform.tfstate`) is a JSON file containing a mapping between resource declarations in your code and the real-world resources they correspond to, plus metadata.

**Why it's used — five reasons:**

1. **Mapping** — `aws_instance.web` in code maps to `i-0abc123def456` in AWS. Without state, Terraform can't know which real resource any code block refers to.

2. **Performance** — caches resource attributes so plans don't have to API-query every resource every time.

3. **Dependency tracking** — holds the DAG that determines apply/destroy order.

4. **Drift detection** — `terraform plan` compares state vs cloud reality and surfaces what changed externally.

5. **Concurrency safety** — when paired with a locking backend (DynamoDB on S3, blob lease on Azure Storage), state-level locking prevents two simultaneous applies from corrupting infrastructure.

**Critical operational facts:**

- **State contains secrets** — passwords, KMS keys, certificates are stored as raw resource attributes. This is *the* reason state must be encrypted at rest, access-controlled, and never committed to Git.
- **State must be remote in any team setup** — local state on a laptop is a single point of failure.
- **Versioning matters** — enable bucket versioning on S3/Azure Storage so corruption is recoverable.
- **State surgery is supported** — `terraform state mv/rm/import/replace-provider` — but hand-editing the JSON is never the right move.

### 7. `terraform` block and `required_providers`

The `terraform` block configures Terraform itself — version constraints, backends, and required providers.

```hcl
terraform {
  required_version = ">= 1.6.0, < 2.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.40"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.27"
    }
  }

  backend "s3" {
    bucket         = "company-tfstate"
    key            = "prod/network.tfstate"
    region         = "us-east-1"
    dynamodb_table = "tfstate-locks"
    encrypt        = true
  }
}
```

**`required_version`** — pins Terraform CLI version. Different Terraform versions have different state schemas; mismatches between team members cause state corruption.

**`required_providers`** — declares which providers the configuration uses, where to source them from (Terraform Registry path), and version constraints.

- **`source`** is the Registry address. Without it, Terraform 0.13+ doesn't know where to download the provider. Custom or non-HashiCorp providers must specify their source explicitly.
- **`version`** uses operators: `~> 5.40` is "pessimistic" (>= 5.40, < 6.0); `>= 5.0, < 6.0` is explicit range; pinning `= 5.42.0` is most strict.
- **`.terraform.lock.hcl`** — generated by `terraform init`, records the exact version + SHA256 checksum of each provider. Commit to Git for reproducibility.

**Why this matters:** before Terraform 0.13, providers were inferred from resource names — `aws_instance` implied the AWS provider. Modern Terraform requires explicit declaration, which enables third-party providers from any registry and prevents ambiguity. The `required_providers` block is mandatory for any modern config.

### 8. `remote-exec` in Terraform

`remote-exec` is a **provisioner** that runs commands on the remote resource after it's created — typically over SSH for Linux or WinRM for Windows.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0abc123"
  instance_type = "t3.micro"
  key_name      = "my-key"

  connection {
    type        = "ssh"
    user        = "ec2-user"
    host        = self.public_ip
    private_key = file("~/.ssh/id_rsa")
  }

  provisioner "remote-exec" {
    inline = [
      "sudo yum update -y",
      "sudo yum install -y nginx",
      "sudo systemctl enable --now nginx"
    ]
  }
}
```

You can also use `script` to run a local script remotely, or `scripts` for multiple:

```hcl
provisioner "remote-exec" {
  script = "scripts/bootstrap.sh"
}
```

**Critical caveats I'd raise in the interview:**

HashiCorp's official guidance is to **avoid provisioners as a last resort**. Reasons:

- **Anti-pattern for cloud configuration** — modern cloud-native approaches don't need SSH. Use `user_data` (EC2), Cloud-init, Ignition (CoreOS), or baked AMIs (Packer).
- **Brittle** — requires direct network connectivity to the instance during apply; firewalls, NATs, bastion hosts complicate this.
- **Hard to make idempotent** — Terraform reruns on changes can re-run the provisioner unpredictably.
- **Not part of state** — if the provisioner fails, the resource is marked "tainted" and gets destroyed/recreated on next apply, even if the resource itself is fine.
- **Order issues** — provisioners run only at create time (or destroy time with `when = destroy`), not on updates.

**Better alternatives:**
- **EC2 user_data** for initial bootstrap.
- **Packer** to bake configured AMIs.
- **Ansible / Chef / Puppet** for ongoing config management.
- **AWS Systems Manager Run Command / State Manager** for post-deploy operations.
- **Cloud-init** for declarative cloud-instance configuration.

I'd say in the interview: "I use `remote-exec` sparingly — typically only for one-off bootstrap actions I can't accomplish via user_data or a baked image, and only when the cost of a more proper solution outweighs the technical-debt cost of a provisioner."

### 9. `null_resource` in Terraform

`null_resource` is a resource that **manages no actual infrastructure** but participates in Terraform's lifecycle so it can host provisioners, depend on other resources, and respond to changes via `triggers`.

```hcl
resource "null_resource" "post_deploy" {
  triggers = {
    instance_id = aws_instance.web.id      # re-runs when instance changes
    config_hash = sha256(file("config.yaml"))   # re-runs when config changes
  }

  provisioner "local-exec" {
    command = "./scripts/notify-deploy.sh ${aws_instance.web.public_ip}"
  }

  depends_on = [aws_db_instance.database]
}
```

**Common uses:**

- **Run a script after a resource is created** — notifications, registration with external systems.
- **Force re-execution on triggers** — re-run provisioner when a watched value changes.
- **Compose multiple provisioners** — group related actions under one resource.
- **Workaround for missing resources** — when Terraform doesn't have a provider for some action, fall back to `null_resource` + `local-exec` calling the CLI.

**Modern alternative — `terraform_data` (Terraform 1.4+):**

```hcl
resource "terraform_data" "post_deploy" {
  triggers_replace = [aws_instance.web.id, sha256(file("config.yaml"))]

  provisioner "local-exec" {
    command = "./scripts/notify.sh"
  }
}
```

`terraform_data` is built into Terraform (no `null` provider needed) and uses cleaner syntax. New code should prefer it; legacy code still uses `null_resource`.

**The same caveat applies** — both are escape hatches. Heavy use signals that work should move outside Terraform.

---

## ☸️ Kubernetes

### 10. Worker node keeps restarting — diagnosis

Systematic triage:

**1. Check the node and why it's restarting**

```bash
# Node status and conditions
kubectl describe node <node>
# Look at:
#   - Conditions: MemoryPressure, DiskPressure, PIDPressure, NetworkUnavailable
#   - Events at the bottom — recent state changes

# Node uptime — how long since last restart
kubectl get nodes
ssh <node>
uptime
```

**2. SSH to the node and check OS-level**

```bash
# Recent reboots
last reboot
last -x | head

# Kernel logs around the restart
dmesg -T | tail -100
journalctl -k --since "30 minutes ago"
journalctl -u kubelet --since "30 minutes ago"

# Look for: kernel panics, OOM events, disk failures
```

**3. Common root causes ranked by likelihood**

**A. OOM kill cascading to node restart**
A pod or system process consumes all node memory; OOM killer kills critical processes (kubelet, containerd); node becomes unresponsive; instance health check fails; ASG terminates and replaces.

```bash
dmesg | grep -i "killed process"
journalctl -k | grep -i oom
```

Fix: set memory requests/limits on all pods, configure `--system-reserved` and `--kube-reserved` on kubelet, add LimitRanges per namespace.

**B. Disk full**
`/var/lib/containerd` or `/var/log` fills up. Kubelet starts evicting; eventually node goes NotReady; cluster autoscaler / ASG terminates.

```bash
df -h
du -sh /var/lib/containerd/* /var/log/*
```

Fix: configure kubelet image GC (`--image-gc-high-threshold=85, --image-gc-low-threshold=70`), larger root volume, separate volume for `/var/log`.

**C. Kubelet or container runtime crashing**
Check journal for kubelet and containerd:
```bash
systemctl status kubelet containerd
journalctl -u kubelet -n 200
```

Common causes: kubelet config bug, runtime version mismatch, certificate expiry, network issues to API server.

**D. Spot instance interruption (cloud-specific)**
On EKS/AKS with spot nodes, the cloud provider reclaims capacity. The "restart" is actually termination + replacement.

```bash
# EKS — check Node Termination Handler events
kubectl get events --field-selector reason=NodeTerminationStarted
```

Fix: install AWS Node Termination Handler, configure PodDisruptionBudgets, diversify spot instance types.

**E. ASG health check failing**
ASG configured with ELB health check; if the node doesn't respond on the health-check path, ASG terminates.

```bash
# AWS: check ASG events
aws autoscaling describe-scaling-activities --auto-scaling-group-name <name>
```

**F. Underlying hypervisor issue**
Rare on managed K8s, more common on self-managed. Cloud provider hardware failure → instance auto-recovery.

**G. Kernel panic**
```bash
dmesg | grep -i panic
# Or check serial console output via cloud provider console
```

**4. Mitigation steps**

- **Cordon affected nodes** to prevent new pods being scheduled.
- **Drain pods** if the issue is per-node and you can move workloads elsewhere.
- **Increase observability** — install Node Problem Detector if not already running.
- **Replace the node** rather than fight to fix it in place (cattle, not pets).
- **Scale up node pool** if capacity is the actual constraint.

**5. Long-term fixes**

- Set resource requests/limits on every workload.
- Use PriorityClasses so critical system workloads aren't evicted first.
- Run Node Problem Detector with alerting.
- Monitor node disk, memory, file descriptors as standard SRE practice.
- Consider Karpenter (or equivalent) for better node lifecycle management.

### 11. Kubernetes resource for tokens, secrets, passwords

**Secret** — designed for sensitive data: passwords, OAuth tokens, SSH keys, certificates, registry credentials.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: app
type: Opaque
stringData:
  username: admin
  password: "rotate-me"
```

**Types:**
- `Opaque` — generic (default).
- `kubernetes.io/tls` — TLS cert + key.
- `kubernetes.io/dockerconfigjson` — image registry credentials.
- `kubernetes.io/service-account-token` — service account tokens.
- `kubernetes.io/basic-auth`, `kubernetes.io/ssh-auth` — auth types.

**Important caveat:** Secrets are base64-encoded, **not encrypted by default**. They sit in etcd as base64. For real encryption:
- Enable **encryption at rest** via `--encryption-provider-config` on the API server, backed by KMS.
- Or — better — use **External Secrets Operator** or **CSI Secrets Store driver** to sync from AWS Secrets Manager / Azure Key Vault / HashiCorp Vault. Secrets stay in the dedicated KMS; the in-cluster Secret is just a sync target.

**Consuming Secrets in pods:**

```yaml
# As environment variables
envFrom:
- secretRef: { name: db-credentials }

# Individual env var
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-credentials
      key: password

# Mounted as files
volumes:
- name: db-creds
  secret: { secretName: db-credentials }
volumeMounts:
- name: db-creds
  mountPath: /etc/db
  readOnly: true
```

Prefer file mounts over env vars where possible — env vars are visible to any process in the container and can leak via crash dumps.

### 12. Readiness probe

A **readiness probe** determines whether a pod is ready to **receive traffic from a Service**.

- When readiness fails: kubelet removes the pod's IP from the Service's Endpoints — no traffic is routed to it. The pod itself is not killed.
- When readiness passes: the pod is added back to Endpoints.

Use cases:
- **Warm-up period** — JIT compilation, cache pre-loading, connection pool establishment.
- **Dependency readiness** — DB connectivity, cache available, downstream services reachable.
- **Temporary back-pressure** — under heavy load, a pod can fail readiness to shed traffic until it recovers.
- **Graceful shutdown** — during `preStop`, fail readiness so traffic stops being routed before the process exits.

```yaml
readinessProbe:
  httpGet:
    path: /health/ready
    port: http
  initialDelaySeconds: 0      # rely on startupProbe instead
  periodSeconds: 5
  failureThreshold: 2
  successThreshold: 1
  timeoutSeconds: 3
```

Three probe types: `httpGet`, `tcpSocket`, `exec`. Plus newer `grpc` (v1.24+).

### 13. Liveness vs Readiness (vs Startup)

Three probe types, three distinct purposes — easy to confuse, and getting them wrong causes real outages.

| | Liveness | Readiness | Startup |
|---|---|---|---|
| Question it answers | "Is the container alive?" | "Should this pod receive traffic?" | "Has the container finished starting?" |
| Failure consequence | Kubelet kills the container; restart per restart policy | Pod removed from Service endpoints; no traffic | Disables liveness/readiness until passing |
| Use case | Deadlock detection | Traffic gating, dependency readiness | Slow-starting apps (Spring Boot, JVMs) |
| Should check | The process itself | Dependencies + readiness for traffic | Application initialization complete |
| Frequency typical | Every 10s | Every 5s | Every 5-10s during startup |

**The critical mistake people make:** putting dependency checks in the liveness probe.

If liveness probe checks "can I reach the database," and the DB has a 30-second blip, every pod fails liveness, kubelet kills every pod, all pods restart simultaneously, none of them can reach the DB during restart either, and you have a cascading outage from a transient DB issue.

**Correct separation:**
- **Liveness**: process is alive and not deadlocked. Cheap, reliable, doesn't depend on external services.
- **Readiness**: ready to serve traffic. Can include dependency checks because failure just means "stop routing," not "kill me."
- **Startup**: gives slow-starting apps a longer initial grace period.

```yaml
# Java Spring Boot example
startupProbe:
  httpGet: { path: /actuator/health, port: http }
  failureThreshold: 30
  periodSeconds: 10       # → 5 minutes of boot grace

livenessProbe:
  httpGet: { path: /actuator/health/liveness, port: http }
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet: { path: /actuator/health/readiness, port: http }
  periodSeconds: 5
  failureThreshold: 2
```

Spring Boot 2.3+ separates `/health/liveness` and `/health/readiness` for exactly this reason.

### 14. Storage resources in a Pod

Several storage primitives, varying in lifetime, scope, and use case:

**Volume types within a pod:**

- **`emptyDir`** — temporary scratch space created when the pod starts, deleted when pod is removed. Lives on node disk or in tmpfs (`medium: Memory`).
- **`hostPath`** — mounts a path from the node's filesystem. Use sparingly — breaks pod portability and is a security concern.
- **`configMap` / `secret`** — projects ConfigMap or Secret data as files into the pod.
- **`downwardAPI`** — exposes pod/container metadata (labels, annotations, resource limits) as files.
- **`projected`** — combines multiple volume sources (configMap + secret + serviceAccountToken) into a single mount.
- **`csi`** — generic interface to any CSI driver (EBS, Azure Disk, GCE PD, Ceph, etc.).
- **`persistentVolumeClaim` (PVC)** — claims a `PersistentVolume` for durable storage.

**PersistentVolume model:**

- **PersistentVolume (PV)** — a cluster-level storage resource (EBS volume, Azure Disk, NFS share). Created statically by an admin or dynamically by a StorageClass.
- **PersistentVolumeClaim (PVC)** — a namespace-scoped request for storage. Pod references the PVC.
- **StorageClass** — defines a class of storage (gp3 SSD, premium SSD, etc.) and a provisioner that creates PVs on demand.

**Access modes:**
- `ReadWriteOnce` (RWO) — mounted read-write by one node. Most block storage (EBS).
- `ReadOnlyMany` (ROX) — multiple nodes can mount read-only.
- `ReadWriteMany` (RWX) — multiple nodes can mount read-write. NFS, EFS, Azure Files.
- `ReadWriteOncePod` (RWOP) — one pod can use it (v1.22+).

**Example:**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: data, namespace: app }
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: gp3
  resources:
    requests:
      storage: 50Gi
---
apiVersion: v1
kind: Pod
metadata: { name: app }
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: data
      mountPath: /data
    - name: config
      mountPath: /etc/config
      readOnly: true
    - name: scratch
      mountPath: /tmp
  volumes:
  - name: data
    persistentVolumeClaim: { claimName: data }
  - name: config
    configMap: { name: app-config }
  - name: scratch
    emptyDir: { sizeLimit: 1Gi }
```

**For StatefulSets:** `volumeClaimTemplates` creates a PVC per pod, preserving identity across restarts.

### 15. ReplicaSet manifest

A ReplicaSet ensures a specified number of pod replicas are running. In practice, you almost never write a ReplicaSet directly — you write a Deployment, which manages ReplicaSets for you.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
  namespace: web
  labels:
    app: nginx
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
        image: nginx:1.27-alpine
        ports:
        - containerPort: 80
        resources:
          requests: { cpu: 100m, memory: 128Mi }
          limits:   { cpu: 500m, memory: 256Mi }
```

**Why you use Deployment instead:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels: { app: nginx }
  template:
    # ... same as above
```

The Deployment adds rolling-update strategy, revision history, and rollback support on top of ReplicaSet's basic "keep N pods alive" guarantee.

**When you'd see a bare ReplicaSet:** rarely. Maybe a teaching example, or an unusual use case where you specifically don't want a Deployment's strategy logic. In production, always Deployment.

### 16. NodePort range

**Default range: 30000–32767** (configurable on the API server via `--service-node-port-range`).

That's 2,768 ports available. The range is intentionally outside the standard system port range (1-1023) and ephemeral range (32768+) to avoid conflicts.

```yaml
apiVersion: v1
kind: Service
metadata: { name: my-svc }
spec:
  type: NodePort
  selector: { app: myapp }
  ports:
  - port: 80           # Service port
    targetPort: 8080   # Container port
    nodePort: 30080    # Optional — specify explicitly, otherwise auto-assigned
```

Without `nodePort` specified, Kubernetes auto-assigns from the range. If you specify one, it must be in-range and not already used.

### 17. ClusterIP vs NodePort

| | ClusterIP | NodePort |
|---|---|---|
| Default type | Yes | No |
| Accessibility | Internal to cluster only | Exposed on every node's IP at a specific port |
| IP/port | Virtual IP from cluster service CIDR | Node IP + port from NodePort range |
| Use case | Service-to-service communication | Direct external access without a cloud LB |
| Built on top of | iptables/IPVS rules | ClusterIP + node-level port forwarding |

**ClusterIP (default):**
```yaml
spec:
  type: ClusterIP
  selector: { app: backend }
  ports:
  - port: 80
    targetPort: 8080
```
DNS: `backend.namespace.svc.cluster.local` resolves to the ClusterIP, which kube-proxy load-balances to backing pods.

**NodePort:**
```yaml
spec:
  type: NodePort
  selector: { app: api }
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080
```
Reachable at `http://<any-node-ip>:30080` from outside the cluster.

**Why you'd choose each:**
- **ClusterIP** for internal services — frontend talks to backend, backend talks to database.
- **NodePort** for dev/test exposure, or when you don't have a cloud load balancer (bare-metal clusters).
- **Neither for production external traffic** — use `LoadBalancer` (provisions cloud LB) or, better, `Ingress` (single LB fronting many services with L7 routing).

**Architect's framing:** in production on EKS/AKS, ClusterIP is what's used 95% of the time for internal traffic; an Ingress controller (also a single LoadBalancer or NLB) handles external traffic for all services with hostname/path routing. NodePort exists mostly as a primitive others build on.

### 18. Default namespaces in Kubernetes

Four namespaces ship with every cluster:

**`default`** — where resources go if no namespace is specified. Best practice: don't put your workloads here. Create proper namespaces.

**`kube-system`** — Kubernetes control-plane components: CoreDNS, kube-proxy, CNI DaemonSet, metrics-server, cluster-autoscaler. Generally hands-off for users; system reserves it.

**`kube-public`** — readable by all users including unauthenticated. Mostly used for cluster-info (`kubectl cluster-info`).

**`kube-node-lease`** — holds Lease objects for node heartbeats. Replaced the old "every node updates its status" pattern with lightweight leases for scalability. Hidden from typical use.

In managed Kubernetes you'll also see provider-specific namespaces:
- EKS: `aws-node`, sometimes `karpenter`, `amazon-cloudwatch`, `external-dns`
- AKS: `gatekeeper-system`, `azure-arc`, `kured`
- GKE: `gmp-system`, `gke-gmp-system`, `gke-managed-*`

These aren't "default" but appear by virtue of cluster add-ons.

---

## 🔐 Networking & Security

### 19. Basis for using SSL termination

You'd choose SSL termination based on several factors:

**1. Centralized cert management**
Managing certificates on 50 backend services is operationally painful. Terminate TLS at one layer (LB, ingress, mesh) and let backends speak plain HTTP within a trusted network. One renewal touches one place.

**2. Performance offload**
TLS handshakes are CPU-expensive. Termination devices (hardware LBs, cloud LBs like ALB/Application Gateway) have optimized crypto paths. Offloading lets backends use that CPU for business logic.

**3. L7 features that require visibility into the request**
A LB can do path-based routing, header inspection, WAF inspection, request rewriting only if it can see the unencrypted request. TLS termination is the precondition.

**4. Consistent policy enforcement**
WAF rules, rate limiting, bot detection, authentication checks all run at the termination point. Without termination, these features have to live in every backend.

**5. Internal trust boundary**
For traffic on a private network within a trusted boundary (VPC, mesh-managed), backend services don't need to verify TLS again — the perimeter already authenticated. Avoiding re-encryption simplifies operations.

**Counter-considerations** (when NOT to terminate at the edge):

- **End-to-end encryption requirements** — regulated workloads (PCI, HIPAA, banking) may require encryption all the way to the database/application. Use **passthrough** or **re-encryption** instead.
- **Zero-trust networks** — service mesh patterns (Istio, Linkerd) terminate TLS at the sidecar, not the LB. mTLS is the source-of-truth.
- **End-user-encrypted data** — if the request is meant only for the final service to read, terminating earlier is a leak.

**Common pattern in production:**
- TLS terminates at CloudFront / ALB (public edge).
- ALB re-encrypts to the EKS ingress (private LB-to-pod traffic).
- Service mesh adds mTLS between services inside the cluster.
- This is **TLS termination at multiple layers**, sometimes called "TLS at every hop."

### 20. SSL termination

**SSL/TLS termination** is the process of decrypting incoming TLS-encrypted traffic at an intermediate point (typically a load balancer, reverse proxy, or ingress controller) so the request can be inspected and forwarded to backends — usually as plain HTTP within a trusted network.

**The flow:**

```
[Client]  ←─ TLS encrypted (HTTPS) ─→  [Termination point: ALB / NGINX / API Gateway]  ─ HTTP plaintext (or re-encrypted TLS) ─→  [Backend service]
```

**Three patterns:**

1. **Pure termination** — TLS terminated at the LB; plaintext to backends.
   - Trade-off: traffic inside the trust boundary is unencrypted.
   - Use case: simple architecture, single VPC, modest security posture.

2. **Termination and re-encryption (TLS bridging)** — TLS terminated at edge, new TLS connection to backend.
   - Trade-off: double crypto cost but full end-to-end protection.
   - Use case: regulated workloads requiring encryption in transit everywhere.

3. **Passthrough (no termination)** — TLS terminates at the backend; the LB is just a TCP proxy.
   - Trade-off: no L7 visibility at the LB, no WAF, no path-based routing.
   - Use case: backend needs the actual client cert (mTLS), or backend handles its own crypto.

**Required components at the termination point:**

- **TLS certificate + private key** — stored on the device. In cloud, typically managed via ACM (AWS), Key Vault (Azure), Certificate Manager (GCP). cert-manager in Kubernetes for in-cluster.
- **Cipher suite policy** — which algorithms are allowed. Modern policies disable old TLS 1.0/1.1 and weak ciphers.
- **SNI handling** — Server Name Indication routes connections to the right cert when multiple certs share an IP.

**On AWS specifically:**
- **ALB / NLB / CLB** can all terminate TLS, using certs from ACM.
- **CloudFront** terminates at the edge globally.
- **API Gateway** terminates TLS for HTTP APIs.

**In Kubernetes:**
- Ingress controllers (NGINX, AGIC, Traefik) terminate based on Ingress `tls:` config.
- cert-manager auto-renews certs via Let's Encrypt or internal CAs.

**Architect's framing:**
> "SSL termination is the foundation for everything you'd want a modern L7 LB to do — WAF, path-based routing, header inspection, rate limiting. It also centralizes cert lifecycle, which is a huge operational win. The decision isn't whether to terminate — it's where to terminate, and whether to re-encrypt downstream. For most production architectures: terminate at the edge for L7 features, re-encrypt to the application tier for defense in depth, and let a service mesh enforce mTLS between services for zero-trust within the cluster."

---

## ☁️ AWS

### 21. ALB vs NLB

| | Application Load Balancer | Network Load Balancer |
|---|---|---|
| OSI Layer | L7 (HTTP/HTTPS) | L4 (TCP/UDP/TLS) |
| Protocols | HTTP, HTTPS, gRPC, WebSocket | TCP, UDP, TLS |
| Routing | Path-based, host-based, header-based, query-string | IP/port-based |
| TLS termination | Yes | Yes (or passthrough) |
| WAF integration | Yes, native | No (use Network Firewall) |
| Static IP | No (uses DNS) | Yes (Elastic IP per AZ) |
| Performance | Lower throughput, higher latency | Very high throughput, very low latency (millions of req/s, ~100µs) |
| Connection draining | Yes | Yes |
| Sticky sessions | Yes (cookie-based) | Yes (source IP-based) |
| Health checks | HTTP/HTTPS at L7 | TCP / HTTP/HTTPS |
| Targets | EC2, IP, Lambda, ECS, EKS | EC2, IP, ALB (NLB → ALB chaining) |
| Cross-zone LB | Yes (free) | Yes (charged) |
| Source IP preservation | Headers (X-Forwarded-For) | Yes (preserved natively) |
| Pricing | Per LCU (request-based) | Per NLCU (connection/throughput-based) |
| Use case | HTTP/web apps, microservice routing | TCP/UDP services, ultra-high-performance, static IP need, non-HTTP protocols |

**When to use ALB:**
- Web applications, REST APIs, gRPC services.
- Need path-based or host-based routing.
- Want WAF protection.
- Need request-level routing logic.
- HTTPS workloads.

**When to use NLB:**
- TCP/UDP protocols (databases, gaming, custom protocols).
- Need static IP addresses (for firewall whitelisting by downstream consumers).
- Ultra-high throughput / ultra-low latency.
- Source IP preservation required (e.g., for IP-based authentication).
- TLS passthrough to backends.
- Backend handles its own L7 logic.

**Common pattern:** for EKS, **AWS Load Balancer Controller** can provision either ALB (for Ingress) or NLB (for Service of type LoadBalancer). Most workloads use ALB; specific cases (e.g., gRPC at very high QPS, MQTT, custom TCP) use NLB.

### 22. Amazon Route 53

**Amazon Route 53** is AWS's managed DNS service. It also provides domain registration and health checking. The "53" refers to DNS port 53.

**What it does:**

**1. Authoritative DNS**
- Hosts zones for your domains.
- Resolves DNS queries globally via a network of edge locations.
- 100% SLA on DNS resolution.

**2. Domain registration**
- Register and manage domains directly.
- TLD support including most gTLDs and many ccTLDs.

**3. Health checks**
- Monitors endpoints (HTTP, HTTPS, TCP) from multiple AWS regions.
- Combines with routing policies for DNS-based failover.
- CloudWatch metrics on check status.

**4. Routing policies — the powerful part:**

- **Simple** — basic A/AAAA record.
- **Weighted** — split traffic by percentage (canary deployments at the DNS layer).
- **Latency-based** — route to the AWS region with lowest measured latency to the user.
- **Geolocation** — route based on user's geographic location (continent, country, US state).
- **Geoproximity** — route based on distance, with adjustable bias to expand/shrink a region's footprint.
- **Failover** — primary/secondary; on primary health-check failure, traffic shifts to secondary.
- **Multivalue answer** — returns up to 8 healthy records (poor man's load balancing at DNS).
- **IP-based** — newer policy, routes based on user's source IP range.

**5. Route 53 Resolver**
- DNS forwarder for hybrid VPC/on-prem setups.
- Inbound endpoints — on-prem can resolve AWS-internal names.
- Outbound endpoints — AWS workloads can resolve on-prem names.

**6. Private hosted zones**
- DNS zones accessible only within associated VPCs.
- Common pattern: `internal.company.com` resolving to private endpoints.

**Common use cases:**
- **DNS failover** for DR — Route 53 health checks detect primary region failure and shift DNS to secondary.
- **Geo-routing for data residency** — EU users hit EU region, US users hit US region.
- **Latency-based routing for performance** — users get the nearest region automatically.
- **Weighted routing for blue-green or canary** — shift 5% of traffic to the new version, then 25%, then 100%.
- **Hybrid DNS** — corporate domain forwarding to AWS Route 53 private zones via Resolver.

### 23. Is Elastic IP single per AWS account?

**No.** By default, AWS gives you a **soft limit of 5 Elastic IPs per region per account**. This limit can be raised via service quota request.

Important nuances:

- The limit is **per region**, not per account globally. 5 EIPs in us-east-1 + 5 EIPs in us-west-2 = 10 EIPs total.
- The limit is a **soft limit** — request increases via the Service Quotas console; AWS routinely approves to dozens or hundreds for legitimate needs.
- EIPs **incur charges when not associated** with a running instance. This is intentional to discourage hoarding.
- EIPs come from a finite pool of public IPv4 addresses; AWS started charging $0.005/hour (~$3.65/month) for all public IPv4 in 2024, including allocated EIPs.

**Modern best practice — avoid EIPs where possible:**

- **Use a NAT Gateway** for outbound egress from private subnets (1 NAT per AZ, shared across many instances).
- **ALB / NLB** for inbound traffic (NLB can have an EIP per AZ, ALB uses DNS-only).
- **CloudFront** for global edge.
- **PrivateLink + VPC Endpoints** to avoid traversing the public internet entirely.
- **AWS Global Accelerator** for static anycast IPs with intelligent routing.

EIPs are most useful when an external system needs to whitelist a specific IP address (third-party API integrations with strict allowlists), or for legacy systems that expect a stable public IP.

### 24. Security Groups

**Security Groups (SGs)** are virtual firewalls operating at the **instance/ENI level** in AWS, controlling inbound and outbound traffic.

**Key properties:**

- **Stateful** — if a request goes out, the response is automatically allowed back. You don't need a return rule.
- **Allow-only** — rules are allow rules; no deny rules. Default is deny-all; you allow specific traffic.
- **Attached to ENIs** — instances, ALBs, NLBs, RDS instances, EFS mount targets, EKS nodes/pods all use security groups via their ENIs.
- **Evaluated together** — multiple SGs attached to one ENI; rules are combined (union of allows).
- **Reference other SGs** — instead of an IP CIDR, you can allow `from security group sg-abc123`. Highly recommended pattern.

**Example:**

```hcl
resource "aws_security_group" "app" {
  name        = "app-sg"
  description = "Application tier"
  vpc_id      = aws_vpc.main.id

  ingress {
    description     = "HTTP from ALB"
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]   # reference, not CIDR
  }

  egress {
    description = "All outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "db" {
  name   = "db-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]   # only app SG can reach DB
  }
}
```

**Best practices:**
- **Reference SGs, not CIDRs**, when both sides are within the VPC. Self-documenting and resilient to IP changes.
- **One SG per tier** — `web-sg`, `app-sg`, `db-sg`. Don't pile everything into one giant SG.
- **Restrict outbound** in regulated environments. Default `0.0.0.0/0` egress is convenient but excessive.
- **Use AWS Config rules** to detect SGs with `0.0.0.0/0` on dangerous ports (22, 3389).
- **Tag SGs** with environment, owner, and purpose for audit and cost allocation.

**Quotas:**
- 60 inbound + 60 outbound rules per SG (soft).
- 5 SGs per ENI (soft, up to 16 with quota increase).

### 25. NACL (Network Access Control List)

**NACL** is a **stateless** firewall operating at the **subnet level** in a VPC.

| | Security Group | NACL |
|---|---|---|
| Level | ENI (instance) | Subnet |
| State | Stateful | Stateless |
| Rule types | Allow only | Allow AND deny |
| Default | Deny all (until rules added) | Allow all (default NACL); custom NACLs deny all |
| Rule evaluation | All rules evaluated together | Numbered, evaluated in order (lowest first); first match wins |
| Applies to | Specific resources | All resources in associated subnets |

**Statelessness implications:**

Since NACLs are stateless, if you allow inbound on port 443, you must *also* explicitly allow outbound responses on ephemeral ports (typically 1024-65535).

```
NACL Inbound rules:
  100  Allow  TCP 443   from 0.0.0.0/0   ← inbound HTTPS
  200  Allow  TCP 80    from 0.0.0.0/0   ← inbound HTTP
  *    Deny   ALL

NACL Outbound rules:
  100  Allow  TCP 1024-65535  to 0.0.0.0/0   ← response traffic
  *    Deny   ALL
```

**When to use NACLs (in addition to SGs):**

- **Subnet-level allow/deny by IP CIDR** — block a specific malicious IP range from reaching any resource in a subnet.
- **Defense in depth** — SGs + NACLs are independent layers. If an SG is misconfigured permissively, NACL still enforces subnet boundary.
- **Compliance requirements** — some regulatory regimes mandate subnet-level controls separate from instance-level.
- **Deny-by-IP** capabilities SGs don't provide (SGs are allow-only).

**Practical advice:**

For most workloads, **security groups are sufficient**. NACLs add complexity (the stateless ephemeral-port allowance trips up everyone the first time) and most teams leave the default NACL as-is. Use custom NACLs when you specifically need:
- IP-based denies.
- Subnet-level enforcement decoupled from instance configuration.
- An additional layer of defense for compliance.

**Architect's framing:**
> "NACL is the network-layer firewall at the subnet boundary; SG is the host firewall at the ENI. I lean on SGs for the day-to-day allow rules because stateful + reference-by-SG is cleaner. NACLs come in for two specific cases: blocking known-bad IPs at the subnet boundary, and providing the compliance auditors with a defense-in-depth answer at the subnet layer."

### 26. EC2 User Data

**User data** is a script or cloud-init configuration passed to an EC2 instance at launch time that runs **once on first boot** (by default).

**What it's for:**

- **Bootstrap configuration** — install packages, set hostnames, configure services on a fresh instance.
- **Pull configuration** from S3, Secrets Manager, or Parameter Store.
- **Join the instance to a cluster** — register with EKS, join an Auto Scaling group's app, register with a service discovery system.
- **Apply patches and updates** on first boot.

**Common forms:**

**1. Shell script (most common):**

```bash
#!/bin/bash
yum update -y
yum install -y nginx
systemctl enable --now nginx
echo "<h1>Hello from $(hostname)</h1>" > /usr/share/nginx/html/index.html
```

**2. Cloud-init (declarative, multi-distribution):**

```yaml
#cloud-config
package_update: true
packages:
  - nginx
  - curl
write_files:
  - path: /etc/nginx/conf.d/app.conf
    content: |
      server {
        listen 80;
        location / { root /var/www; }
      }
runcmd:
  - systemctl enable --now nginx
```

**Configuring user data in Terraform:**

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"

  user_data = <<-EOF
    #!/bin/bash
    set -e
    yum update -y
    yum install -y docker
    systemctl enable --now docker
    usermod -aG docker ec2-user

    # Pull image and start
    aws ecr get-login-password --region us-east-1 | \
      docker login --username AWS --password-stdin ${var.ecr_registry}
    docker run -d --restart=always -p 80:8080 \
      ${var.ecr_registry}/${var.image}:${var.tag}
  EOF

  user_data_replace_on_change = true   # force replacement on user_data change
}
```

**Important properties:**

- **Runs as root** by default.
- **Limited to 16 KB** in raw form; use S3-fetched scripts for larger payloads.
- **Runs once by default**, but `cloud-init-per` or `cloud-init` modules can run scripts every boot.
- **Logs to `/var/log/cloud-init-output.log`** and `/var/log/user-data.log` — essential for debugging.
- **Available via metadata service** at `http://169.254.169.254/latest/user-data` (use IMDSv2 with token).

**Production best practices:**

- **Bake AMIs with Packer** for stable, repeatable images. Don't rely on user_data for heavy lifting — the more user_data does, the longer your instance launch takes and the more failure modes you have.
- **Keep user_data minimal** — just enough to "wake up" a pre-configured AMI (pull config, register with cluster, signal readiness).
- **Use Systems Manager** for ongoing configuration management after launch. Don't use user_data as a config-management tool.
- **Use Auto Scaling Group Launch Templates** to manage user_data centrally.
- **Sign your user_data** or use Parameter Store / Secrets Manager for sensitive content — user_data is visible to anyone with `ec2:DescribeInstanceAttribute`.
- **Use `user_data_replace_on_change = true` in Terraform** so changes force new instances rather than silently being ignored.

**Architect's framing:**
> "User data is the bootstrap mechanism, not the configuration system. The pattern I follow: bake immutable AMIs with Packer that contain the application and its dependencies, then use user_data only for last-mile configuration — fetching environment-specific secrets from Parameter Store, registering with a service mesh, joining a cluster. Heavy user_data scripts are slow, brittle, and hard to debug; they're a sign your AMI strategy is wrong."

---

That covers everything. Likely follow-ups to be ready for:

- **"Show me a Service that's both ClusterIP and exposed via an Ingress."** — basic pattern, useful to demonstrate.
- **"What's the difference between User Data and Instance Metadata?"** — user data is what you pass in; metadata is the IMDS service the instance can query (including its user data and dynamic info).
- **"When would you choose Ingress over Service of type LoadBalancer?"** — Ingress for multi-service shared LB with L7 routing; LoadBalancer per-service for non-HTTP or for total isolation.

