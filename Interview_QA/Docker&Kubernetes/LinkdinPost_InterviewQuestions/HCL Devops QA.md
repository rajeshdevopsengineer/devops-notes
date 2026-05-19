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

