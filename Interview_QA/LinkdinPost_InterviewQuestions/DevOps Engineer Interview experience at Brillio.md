# Brillio DevOps Engineer — Round 1 Interview Answers

A complete walkthrough with the depth a senior interview expects.

---

## 1. Transfer S3 data across accounts with zero downtime

The zero-downtime part is critical here — it shapes the answer. The application must keep reading/writing throughout the migration. I'd present **all** approaches and then recommend one.

**Approach 1: S3 Cross-Region/Cross-Account Replication (CRR/SRR)** — my preferred answer

Native, asynchronous, continuous replication from the source bucket to the destination bucket. Zero downtime by design because writes continue to the source while replication keeps the destination in sync.

```hcl
resource "aws_s3_bucket_replication_configuration" "this" {
  role   = aws_iam_role.replication.arn
  bucket = aws_s3_bucket.source.id

  rule {
    id     = "replicate-all"
    status = "Enabled"
    filter {}

    destination {
      bucket        = "arn:aws:s3:::dest-bucket-account-b"
      account       = "ACCOUNT_B_ID"
      storage_class = "STANDARD"

      access_control_translation { owner = "Destination" }

      encryption_configuration {
        replica_kms_key_id = "arn:aws:kms:us-east-1:ACCOUNT_B_ID:key/..."
      }
    }

    source_selection_criteria {
      sse_kms_encrypted_objects { status = "Enabled" }
    }

    delete_marker_replication { status = "Enabled" }
  }
}
```

Pre-requisites: **versioning enabled on both buckets**, IAM role for replication, destination bucket policy granting the replication role access, KMS key policy if encrypted.

Migration sequence:
1. Enable replication. New writes replicate within seconds.
2. Backfill existing objects with **S3 Batch Replication** (replication only catches new writes by default).
3. Monitor `BytesPendingReplication` and replication metrics in CloudWatch.
4. Once destination matches source completely, cut application traffic over to read from destination.
5. Keep replication running for a buffer period as safety net.

**Approach 2: S3 Batch Operations with Copy**

Best for one-shot migration of large existing datasets. Generate an inventory of the source bucket, run a Batch Operations job to copy each object to the destination.

```bash
# Generate inventory CSV via S3 Inventory or list-objects
aws s3control create-job \
  --account-id ACCOUNT_A_ID \
  --operation '{"S3PutObjectCopy":{"TargetResource":"arn:aws:s3:::dest-bucket"}}' \
  --manifest '...' \
  --priority 10 \
  --role-arn arn:aws:iam::ACCOUNT_A_ID:role/BatchOpsRole
```

Pair with replication for new writes during the batch run — same zero-downtime guarantee.

**Approach 3: AWS DataSync**

Managed agent-based service for bulk transfer. Good for one-time migrations with verification, scheduling, and bandwidth throttling. Less elegant than replication for ongoing sync but useful when source isn't S3 (NFS, SMB, on-prem).

**Approach 4: `aws s3 sync` from EC2 / Lambda**

Imperative copy. Fine for small datasets, terrible for large ones (single-threaded by default, no checkpointing). Use only for last-mile or specific subset migrations.

```bash
aws s3 sync s3://source-bucket s3://dest-bucket --source-region us-east-1
```

**Approach 5: Same bucket, change ownership** (rare but worth mentioning)

If "transfer" really means "this bucket now belongs to Account B," you can move the bucket name registration only by deleting and recreating — which isn't zero-downtime. So instead: replicate to a new bucket in Account B, cut over DNS/config, retire the old bucket.

**My recommendation for the interview:**

> "For zero downtime with continuous sync, I'd use **S3 Cross-Account Replication** combined with **S3 Batch Replication** to backfill existing objects. Replication handles all new writes asynchronously, the batch job handles the historical data. Once metrics show the destination is fully caught up and verification passes, switch the application config to point at the destination bucket — the cutover itself is just an environment variable or DNS change, no S3 downtime. I'd keep the replication running both ways for a week as a safety net before decommissioning the source."

Critical details to mention:
- **Versioning is mandatory** on both buckets for replication.
- **KMS keys** need cross-account access — key policy on Account B's CMK must allow Account A's replication role.
- **Object ownership** — use `access_control_translation` so Account B owns replicated objects; otherwise Account A still owns them and Account B may have access surprises.
- **Bucket policies and Lambda triggers** don't replicate — recreate on the destination.
- **Cost** — you pay for the replicated data transfer and storage in Account B; significant for petabyte-scale.

---

## 2. AWS DR best practices and a real architecture

Start by framing: **HA ≠ DR**. HA = surviving component/AZ failures without downtime. DR = recovering from large-scale disasters (region loss, data corruption, ransomware) within agreed RTO/RPO.

**Best practices:**

1. **Define RTO and RPO per workload explicitly**, agreed with business stakeholders. Different services have different tolerances; one DR plan doesn't fit all.
2. **Pick the right DR strategy** based on RTO/RPO and budget:

   | Strategy | RTO | RPO | Cost |
   |---|---|---|---|
   | Backup & Restore | Hours-days | Hours | $ |
   | Pilot Light | Tens of minutes | Minutes | $$ |
   | Warm Standby | Minutes | Seconds-minutes | $$$ |
   | Multi-region Active-Active | Near-zero | Near-zero | $$$$ |

3. **Treat backups as a separate trust boundary** — backups in a separate AWS account with separate IAM. A compromised production admin must not be able to delete backups. This is what saves you from ransomware.
4. **Test DR regularly** — quarterly game days that actually fail over a region. Untested DR plans are fiction. Measure actual RTO.
5. **Document runbooks** with exact commands, screenshots, and decision trees. The person executing the failover at 3am may not be the architect.
6. **Automate failover where possible** — Route 53 health checks for DNS failover, Aurora Global Database for managed cross-region DB failover, S3 replication for storage.
7. **Practice failback** — going back to primary is often harder than failing over. Bidirectional replication patterns matter.
8. **Cross-region IAM and KMS** — replicate KMS keys (multi-region keys) and ensure IAM roles exist in DR region.
9. **Infrastructure as code** so the DR region can be rebuilt deterministically from Terraform if needed.
10. **Monitor replication lag** with alerts before it becomes an RPO violation.

**Architecture I implemented (Warm Standby example):**

> "On a financial services platform we ran active in `us-east-1` and warm standby in `us-west-2`, with RTO 15 minutes and RPO 1 minute.
>
> **Compute:** EKS clusters in both regions, identical Terraform configurations. In `us-west-2`, deployments ran at 25% of `us-east-1`'s replica count — enough to serve traffic if failover happened, scaled up to 100% automatically via HPA after cutover.
>
> **Data:** Aurora Global Database with the primary in `us-east-1` and a global secondary in `us-west-2` — sub-second replication lag, RPO measured in seconds. S3 with cross-region replication (RA-GZRS would have been a managed alternative). ElastiCache Redis Global Datastore for cache-warm secondary.
>
> **Traffic:** Route 53 with health checks on the primary ALB in `us-east-1`. On failure, DNS failed over to the `us-west-2` ALB. CloudFront in front of everything with custom origin failover for static assets.
>
> **Secrets:** AWS Secrets Manager with multi-region replication; KMS multi-region keys so the DR region could decrypt without cross-region API calls.
>
> **GitOps:** ArgoCD running in both clusters, watching the same Git repo. Apps deployed identically.
>
> **Failover trigger:** Manual, executed via runbook. We considered automation but decided false-positive risk (one bad health check causing unnecessary DR cutover) was too high for a Tier-1 financial workload.
>
> **Game days:** Quarterly. We deliberately failed over for a 4-hour window, ran in DR, then failed back. Found and fixed: replication lag spikes during heavy writes, a Lambda function that was region-pinned in code, and a KMS key that wasn't multi-region. The drills are the only reason the plan actually worked."

---

## 3. AWS services used and where

I'd group rather than list-dump:

| Layer | Services | Project usage |
|---|---|---|
| Compute | EC2, EKS, ECS Fargate, Lambda | EKS for microservices, Lambda for event-driven processing (S3 triggers, scheduled jobs), Fargate for batch |
| Storage | S3, EBS, EFS, FSx | S3 for object storage and static assets, EBS for stateful pods, EFS for shared file storage |
| Database | RDS Aurora Postgres, DynamoDB, ElastiCache Redis | Aurora for transactional, DynamoDB for high-write event log, Redis for sessions and cache |
| Networking | VPC, Transit Gateway, ALB, NLB, CloudFront, Route 53, PrivateLink, VPC Endpoints | Hub-and-spoke via TGW, ALB for ingress, CloudFront + Route 53 for global routing |
| Identity & security | IAM, IAM Identity Center, KMS, Secrets Manager, ACM, WAF, GuardDuty | IRSA for pod-level IAM, Secrets Manager + External Secrets Operator, KMS for all encryption |
| Containers | ECR | Image storage with replication + scanning |
| Messaging | SQS, SNS, EventBridge, MSK | SQS for async work, EventBridge for cross-service events, MSK for streaming |
| Observability | CloudWatch, X-Ray | Metrics, logs, traces — alongside Prometheus/Grafana on EKS |
| Governance | Organizations, Control Tower, Config, CloudTrail, Cost Explorer | Multi-account landing zone, Config for compliance rules, CloudTrail to a security account |

In an interview I'd name the 8–10 I've used deepest and add "I've worked with these adjacent ones at a familiar level" for the rest. Padding the list invites a trap follow-up.

---

## 4. Terraform: RDS + credentials in Secrets Manager

```hcl
# Generate a strong random password
resource "random_password" "db" {
  length           = 32
  special          = true
  override_special = "!#$%&*()-_=+[]{}<>:?"
}

# Create the secret container
resource "aws_secretsmanager_secret" "db" {
  name                    = "${var.env}/rds/${var.db_identifier}/master"
  description             = "RDS master credentials for ${var.db_identifier}"
  kms_key_id              = aws_kms_key.secrets.arn
  recovery_window_in_days = 7

  tags = var.tags
}

# Store the credentials JSON in the secret
resource "aws_secretsmanager_secret_version" "db" {
  secret_id = aws_secretsmanager_secret.db.id
  secret_string = jsonencode({
    username = "dbadmin"
    password = random_password.db.result
    engine   = "postgres"
    host     = aws_db_instance.this.address
    port     = aws_db_instance.this.port
    dbname   = aws_db_instance.this.db_name
  })
}

# Subnet group for the DB
resource "aws_db_subnet_group" "this" {
  name       = "${var.env}-${var.db_identifier}"
  subnet_ids = var.private_subnet_ids
  tags       = var.tags
}

# Security group restricting access to app subnets only
resource "aws_security_group" "db" {
  name        = "${var.env}-${var.db_identifier}-sg"
  description = "RDS access from application tier only"
  vpc_id      = var.vpc_id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = var.app_security_group_ids
    description     = "Postgres from app tier"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = var.tags
}

# KMS key for storage encryption and Secrets Manager
resource "aws_kms_key" "secrets" {
  description             = "KMS key for ${var.env} RDS + Secrets"
  enable_key_rotation     = true
  deletion_window_in_days = 30
  tags                    = var.tags
}

# The RDS instance itself
resource "aws_db_instance" "this" {
  identifier     = "${var.env}-${var.db_identifier}"
  engine         = "postgres"
  engine_version = "16.3"
  instance_class = var.instance_class

  allocated_storage     = 100
  max_allocated_storage = 500
  storage_type          = "gp3"
  storage_encrypted     = true
  kms_key_id            = aws_kms_key.secrets.arn

  db_name  = var.db_name
  username = "dbadmin"
  password = random_password.db.result

  db_subnet_group_name   = aws_db_subnet_group.this.name
  vpc_security_group_ids = [aws_security_group.db.id]
  publicly_accessible    = false

  multi_az                = true
  backup_retention_period = 30
  backup_window           = "03:00-04:00"
  maintenance_window      = "sun:04:30-sun:05:30"

  performance_insights_enabled          = true
  performance_insights_retention_period = 7
  monitoring_interval                   = 60
  monitoring_role_arn                   = aws_iam_role.rds_monitoring.arn
  enabled_cloudwatch_logs_exports       = ["postgresql", "upgrade"]

  deletion_protection      = var.env == "prod" ? true : false
  skip_final_snapshot      = var.env == "prod" ? false : true
  final_snapshot_identifier = var.env == "prod" ? "${var.env}-${var.db_identifier}-final" : null

  apply_immediately = false

  lifecycle {
    ignore_changes = [password]   # password rotation outside Terraform shouldn't trigger drift
  }

  tags = var.tags
}

# Allow app pods to read the secret via IRSA
resource "aws_secretsmanager_secret_policy" "db" {
  secret_arn = aws_secretsmanager_secret.db.arn
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { AWS = var.app_irsa_role_arns }
      Action    = ["secretsmanager:GetSecretValue", "secretsmanager:DescribeSecret"]
      Resource  = "*"
    }]
  })
}

# Optional: automatic rotation
resource "aws_secretsmanager_secret_rotation" "db" {
  secret_id           = aws_secretsmanager_secret.db.id
  rotation_lambda_arn = aws_lambda_function.rotation.arn
  rotation_rules { automatically_after_days = 30 }
}
```

**Things I'd call out in the interview:**

- `lifecycle.ignore_changes = [password]` — once rotation kicks in, the password in Secrets Manager diverges from Terraform's; ignoring prevents constant drift.
- **Storage encryption with a customer-managed KMS key** — not the AWS-managed default, so we control key policies and rotation.
- **`deletion_protection = true` in prod** + `skip_final_snapshot = false` + `final_snapshot_identifier` — three layers preventing accidental data loss.
- **IRSA for app pod access** — pods get the secret via their service account's IAM role, no static credentials anywhere.
- **Even better: IAM database authentication for RDS** eliminates the password entirely. The app gets short-lived tokens via IAM; nothing to rotate. The best secret is no secret.

---

## 5. CI/CD pipeline I currently use

I'll show a GitHub Actions example since it's where I have the most recent work; Jenkins flow follows the same logical structure.

```yaml
# .github/workflows/ci-cd.yml
name: ci-cd

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

permissions:
  id-token: write       # for OIDC to AWS
  contents: read
  pull-requests: write

env:
  AWS_REGION: us-east-1
  ECR_REPO: 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp
  GITOPS_REPO: company/gitops-config

jobs:
  build-test-scan:
    name: Build, Test, Scan
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.meta.outputs.tag }}
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }

      - name: Set image tag
        id: meta
        run: echo "tag=$(date +%Y%m%d)-${GITHUB_SHA::7}-${GITHUB_RUN_NUMBER}" >> $GITHUB_OUTPUT

      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: '17', cache: maven }

      - name: Unit tests + coverage
        run: mvn -B clean verify

      - name: Publish test results
        uses: dorny/test-reporter@v1
        if: always()
        with:
          name: JUnit Tests
          path: target/surefire-reports/*.xml
          reporter: java-junit

      - name: SonarQube scan
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: |
          mvn sonar:sonar \
            -Dsonar.host.url=${{ secrets.SONAR_URL }} \
            -Dsonar.projectKey=myapp \
            -Dsonar.qualitygate.wait=true \
            -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml

      - name: Dependency vulnerability scan
        uses: snyk/actions/maven@master
        with: { args: --severity-threshold=high }
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

      - name: Configure AWS via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/gha-ci-role
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build & push image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ env.ECR_REPO }}:${{ steps.meta.outputs.tag }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Trivy image scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.ECR_REPO }}:${{ steps.meta.outputs.tag }}
          severity: CRITICAL,HIGH
          exit-code: '1'
          ignore-unfixed: true

      - name: Sign image with Cosign (keyless via OIDC)
        uses: sigstore/cosign-installer@v3
      - run: cosign sign --yes ${{ env.ECR_REPO }}:${{ steps.meta.outputs.tag }}

  deploy-dev:
    name: Deploy to Dev (GitOps)
    needs: build-test-scan
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    environment: dev
    steps:
      - name: Checkout GitOps repo
        uses: actions/checkout@v4
        with:
          repository: ${{ env.GITOPS_REPO }}
          token: ${{ secrets.GITOPS_PAT }}

      - name: Bump image tag in dev values
        run: |
          yq -i '.image.tag = "${{ needs.build-test-scan.outputs.image_tag }}"' \
            environments/dev/values.yaml

      - name: Commit & push
        run: |
          git config user.name "ci-bot"
          git config user.email "ci-bot@company.com"
          git add . && git commit -m "chore(dev): bump myapp to ${{ needs.build-test-scan.outputs.image_tag }}"
          git push

  deploy-prod:
    name: Deploy to Prod (GitOps)
    needs: build-test-scan
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production       # GitHub Environment with required reviewers
    steps:
      - name: Checkout GitOps repo
        uses: actions/checkout@v4
        with: { repository: ${{ env.GITOPS_REPO }}, token: ${{ secrets.GITOPS_PAT }} }

      - name: Open PR bumping prod image tag
        run: |
          git checkout -b bump-prod-${{ needs.build-test-scan.outputs.image_tag }}
          yq -i '.image.tag = "${{ needs.build-test-scan.outputs.image_tag }}"' \
            environments/prod/values.yaml
          git config user.name "ci-bot"
          git config user.email "ci-bot@company.com"
          git add . && git commit -m "chore(prod): bump myapp"
          git push origin HEAD
          gh pr create --title "chore(prod): bump myapp" --body "Promotion from staging" --base main
        env:
          GH_TOKEN: ${{ secrets.GITOPS_PAT }}
```

Then **ArgoCD** watches the GitOps repo and reconciles the cluster — dev auto-merges and auto-deploys; prod requires PR approval and a human merge before ArgoCD picks it up.

**Key principles I'd articulate:**
- **Build once, promote the artifact** across environments — never rebuild for prod.
- **Security at every stage** — SAST (SonarQube), dependency scan (Snyk), image scan (Trivy), image signing (Cosign).
- **OIDC federation to AWS** — no long-lived AWS credentials in GitHub secrets.
- **GitHub Environments** for approval gating, environment-scoped secrets, branch restrictions.
- **GitOps for the deploy half** — pipeline never has cluster credentials; only ArgoCD does, running inside the cluster.
- **Promotion via PR** for prod — auditable, reviewable, easy to revert.

---

## 6. Managing secrets in Jenkins and GitHub Actions

**GitHub Actions:**

Layered, in order of maturity:

1. **Repository secrets** — `${{ secrets.MY_TOKEN }}`. Encrypted at rest, masked in logs. Good for low-value secrets and bootstrap.
2. **Environment secrets** — scoped to a GitHub Environment (e.g., `production`); only jobs targeting that environment can access them. Pair with required reviewers so prod secrets need approval to use.
3. **Organization secrets** — shared across multiple repos in an org.
4. **OIDC federation to cloud providers** — best practice for AWS/Azure/GCP credentials.

```yaml
permissions:
  id-token: write
  contents: read
steps:
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/gha-deploy
    aws-region: us-east-1
    # no aws-access-key-id, no aws-secret-access-key
```

The IAM role's trust policy restricts to specific repo + branch:
```json
{
  "Effect": "Allow",
  "Principal": { "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com" },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com" },
    "StringLike":   { "token.actions.githubusercontent.com:sub": "repo:company/myapp:ref:refs/heads/main" }
  }
}
```

5. **External secrets manager at runtime** — pipeline authenticates via OIDC, pulls secrets from Secrets Manager / Key Vault / Vault. Secrets never stored in GitHub at all.

```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with: { role-to-assume: arn:aws:iam::...:role/gha-deploy, aws-region: us-east-1 }
- name: Fetch DB password
  run: |
    SECRET=$(aws secretsmanager get-secret-value --secret-id prod/db --query SecretString --output text)
    echo "::add-mask::$SECRET"
    echo "DB_PASS=$SECRET" >> $GITHUB_ENV
```

**Hard rules:**
- Never `echo $SECRET` to logs.
- Never write secrets to artifacts.
- Mask custom secrets explicitly with `::add-mask::`.
- Use `secrets.GITHUB_TOKEN` (auto-provided per job, short-lived) rather than personal PATs where possible.

**Jenkins:**

1. **Credentials plugin** — store secrets in Jenkins's encrypted credential store, reference by ID:

```groovy
withCredentials([
  string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN'),
  usernamePassword(credentialsId: 'ecr-creds', usernameVariable: 'U', passwordVariable: 'P'),
  file(credentialsId: 'kubeconfig-prod', variable: 'KUBECONFIG')
]) {
  sh 'mvn sonar:sonar -Dsonar.login=$SONAR_TOKEN'
  sh 'echo $P | docker login --username $U --password-stdin myregistry'
  sh 'kubectl --kubeconfig $KUBECONFIG apply -f manifests/'
}
```

Jenkins masks the values in console output automatically. Don't use `${SONAR_TOKEN}` Groovy interpolation — it can leak in stack traces; use `$SONAR_TOKEN` shell-side instead.

2. **HashiCorp Vault plugin** — fetch dynamic short-lived secrets at runtime:

```groovy
withVault([vaultSecrets: [
  [path: 'secret/prod/db', secretValues: [
    [vaultKey: 'password', envVar: 'DB_PASS']
  ]]
]]) {
  sh 'flyway migrate -password=$DB_PASS'
}
```

3. **AWS Secrets Manager / Azure Key Vault plugins** — pull secrets at job start.

4. **OIDC to AWS** (newer Jenkins setups) — Jenkins can federate to AWS without long-lived credentials. Or: run Jenkins on EC2/EKS with instance profile / IRSA so the agent has IAM identity directly.

5. **Folder-scoped credentials** — credentials available only to jobs inside a specific folder, supporting team isolation.

6. **Role-based authorization** — restrict which jobs/users can view or use which credentials.

**Hard rules for Jenkins:**
- Never `echo` secrets in shell.
- Never use `script` blocks that print env. Console output is logged.
- Disable plain-text logging plugins.
- Audit who accesses credentials (Jenkins Audit Trail plugin).
- Rotate secrets regularly; better, use a backend that does it for you.

**Common ground for both:**

The right answer to "where do I store production secrets" is **a dedicated secrets manager, accessed at runtime via short-lived federated credentials**. The CI/CD platform's native secret store is for bootstrap and low-sensitivity values only.

---

## 7. Multi-stage Dockerfile with line-by-line explanation

```dockerfile
# syntax=docker/dockerfile:1.6
# ─── Build stage ──────────────────────────────────────────────────────────────
FROM maven:3.9-eclipse-temurin-17 AS build
# Use a build-only image with Maven + JDK; this stage's output won't ship.
# AS build names this stage so we can copy from it later.

WORKDIR /build
# All subsequent paths are relative to /build. Avoids absolute-path mistakes.

COPY pom.xml .
# Copy only the pom.xml first to maximize layer cache: if pom.xml doesn't change,
# Docker reuses the dependency-resolution layer below.

RUN --mount=type=cache,target=/root/.m2 mvn -B dependency:go-offline
# Resolve and download all dependencies into a BuildKit cache mount.
# Cache mount means deps are NOT baked into the image layer — they're a sidecar
# cache reused across builds, even faster than layer caching alone.

COPY src ./src
# Now copy source. Source changes invalidate only this layer and below.

RUN --mount=type=cache,target=/root/.m2 mvn -B clean package -DskipTests
# Build the JAR. Tests run earlier in CI; the image build is just packaging.

# ─── Runtime stage ────────────────────────────────────────────────────────────
FROM eclipse-temurin:17-jre-alpine AS runtime
# Switch to a slim JRE-only Alpine image (~180 MB vs ~800 MB for the build image).
# No Maven, no JDK, no source code, no build tools — drastically smaller and
# fewer CVEs.

RUN addgroup -S app && adduser -S -G app app
# Create a non-root user. Running as root inside a container is the foundation
# many container escapes start from; non-root drops blast radius.

WORKDIR /app
# Set working directory in the runtime image.

COPY --from=build --chown=app:app /build/target/*.jar /app/app.jar
# Copy ONLY the built artifact from the build stage. Everything else (Maven,
# source, .m2) is left behind. --chown sets ownership at copy time so we don't
# need a separate RUN chown layer.

USER app
# All subsequent commands and the running container execute as 'app' user.

EXPOSE 8080
# Documents the port the app listens on. Doesn't actually open the port —
# the container runtime / orchestrator handles that — but it's a signal to
# operators and tools like docker-compose.

HEALTHCHECK --interval=30s --timeout=3s --start-period=20s --retries=3 \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1
# Docker daemon calls this periodically. Failed checks mark the container
# unhealthy. Kubernetes ignores this in favor of probes, but it's useful for
# docker-compose and ECS.

ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75", \
            "-jar", "/app/app.jar"]
# The fixed executable. UseContainerSupport makes the JVM aware of container
# memory limits (critical — without it, JVM sees the host memory and OOMs).
# MaxRAMPercentage gives the JVM 75% of the container's memory limit for heap.
# Hard to override accidentally because ENTRYPOINT uses exec form (array,
# not shell) so the process gets signals directly — clean SIGTERM handling.
```

Companion `.dockerignore`:
```
.git
.gitignore
target
*.iml
.idea
.vscode
Dockerfile
.dockerignore
*.md
.env*
```

**Build and run commands:**

```bash
# Build
docker build -t myapp:1.4.2 .

# Build with BuildKit cache mounts (faster, smaller)
DOCKER_BUILDKIT=1 docker build -t myapp:1.4.2 .

# Tag for ECR
docker tag myapp:1.4.2 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2

# Push
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2

# Run locally
docker run -d --name myapp -p 8080:8080 \
  --memory=1g --cpus=1 \
  -e SPRING_PROFILES_ACTIVE=local \
  myapp:1.4.2

# Run interactively for debugging
docker run -it --rm --entrypoint /bin/sh myapp:1.4.2

# Inspect, logs, exec
docker logs -f myapp
docker exec -it myapp sh
docker inspect myapp | jq '.[0].State'
```

**What this Dockerfile does right (to articulate in interview):**
- **Two stages** — runtime image has zero build tools.
- **Pinned base images** with explicit versions (never `latest`).
- **Layer order optimized for cache** — pom.xml → deps → source → build.
- **BuildKit cache mount** for `.m2` — significantly faster repeat builds.
- **Non-root user** as production hardening.
- **Container-aware JVM flags** — `UseContainerSupport` + `MaxRAMPercentage` prevent OOM on host vs container memory mismatch.
- **HEALTHCHECK** for runtime visibility.
- **Exec-form ENTRYPOINT** for clean signal handling and graceful shutdown.
- **No secrets, no source** in the final image.

---

## 8. Kubernetes manifest: 10 Pods with LoadBalancer Service

I'd never use a bare ReplicaSet for 10 pods — a Deployment is the right primitive. Here's the manifest with line-by-line explanation:

```yaml
apiVersion: apps/v1                          # Deployment lives in apps/v1
kind: Deployment                             # Declarative Pod manager (not a bare RS)
metadata:
  name: myapp                                # Resource name within the namespace
  namespace: production                      # Logical isolation boundary
  labels:                                    # Labels on the Deployment itself
    app: myapp
    tier: backend
spec:
  replicas: 10                               # Desired number of Pod replicas — the "10 pods"
  revisionHistoryLimit: 5                    # Keep last 5 ReplicaSets for rollback

  selector:                                  # How the Deployment finds its Pods
    matchLabels:                             # Must match the Pod template labels below
      app: myapp

  strategy:
    type: RollingUpdate                      # Default; explicit for clarity
    rollingUpdate:
      maxSurge: 2                            # Up to 2 extra Pods during a rollout
      maxUnavailable: 0                      # Never go below desired count

  template:                                  # Pod template — every replica looks like this
    metadata:
      labels:                                # MUST match selector.matchLabels above
        app: myapp
        tier: backend
      annotations:
        prometheus.io/scrape: "true"         # Tells Prometheus to scrape this pod
        prometheus.io/port: "8080"
        prometheus.io/path: "/actuator/prometheus"

    spec:
      serviceAccountName: myapp-sa           # SA bound to an IRSA role for AWS API access
      automountServiceAccountToken: true     # Mount the SA token (needed for IRSA)

      securityContext:                       # Pod-level security defaults
        runAsNonRoot: true                   # Reject any container that runs as root
        runAsUser: 1000
        fsGroup: 1000                        # Volume ownership for non-root user

      terminationGracePeriodSeconds: 30      # Time between SIGTERM and SIGKILL

      topologySpreadConstraints:             # Spread pods across AZs for resilience
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels: { app: myapp }

      affinity:
        podAntiAffinity:                     # Avoid two replicas on the same node
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                topologyKey: kubernetes.io/hostname
                labelSelector:
                  matchLabels: { app: myapp }

      containers:
      - name: myapp                          # Container name within the pod
        image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2-a1b2c3d
        imagePullPolicy: IfNotPresent        # Pull only if not cached locally

        ports:
        - name: http                         # Named port — referenced from Service
          containerPort: 8080                # App listens on 8080 inside the container
          protocol: TCP

        env:                                 # Static env from config
        - name: SPRING_PROFILES_ACTIVE
          value: production
        - name: POD_NAME                     # Useful for logging — which pod produced this
          valueFrom: { fieldRef: { fieldPath: metadata.name } }
        envFrom:                             # Bulk env from ConfigMap and Secret
        - configMapRef: { name: myapp-config }
        - secretRef:    { name: myapp-secrets }

        resources:
          requests:                          # Scheduler reserves this on a node
            cpu: 250m
            memory: 512Mi
          limits:                            # Hard cap — OOMKill on memory excess
            cpu: 1000m
            memory: 1Gi

        startupProbe:                        # Disables liveness/readiness until passing
          httpGet: { path: /health/startup, port: http }
          failureThreshold: 30
          periodSeconds: 10                  # ⇒ 5 min boot grace

        readinessProbe:                      # Failing = pod removed from Service endpoints
          httpGet: { path: /health/ready, port: http }
          periodSeconds: 5
          failureThreshold: 2

        livenessProbe:                       # Failing = kubelet kills and restarts container
          httpGet: { path: /health/live, port: http }
          periodSeconds: 10
          failureThreshold: 3

        lifecycle:
          preStop:                           # Graceful shutdown — give load balancer time
            exec:
              command: ["sh", "-c", "sleep 15"]

        securityContext:
          allowPrivilegeEscalation: false    # Block setuid-style privilege gain
          readOnlyRootFilesystem: true       # Root FS read-only; writable dirs via volumes
          capabilities:
            drop: ["ALL"]                    # Drop all Linux capabilities

        volumeMounts:
        - name: tmp                          # Writable /tmp since rootfs is read-only
          mountPath: /tmp

      volumes:
      - name: tmp
        emptyDir: {}

---
apiVersion: v1                               # Service is in the core API
kind: Service
metadata:
  name: myapp                                # In-cluster DNS: myapp.production.svc.cluster.local
  namespace: production
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "external"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
spec:
  type: LoadBalancer                         # Provisions an AWS load balancer
  externalTrafficPolicy: Local               # Preserves client IP; only routes to local pods
  selector:                                  # Routes to Pods matching these labels
    app: myapp                               # Must match the Pod template labels
  ports:
  - name: http
    port: 80                                 # Port on the LB
    targetPort: http                         # Pod's named "http" port (8080)
    protocol: TCP

---
apiVersion: policy/v1                        # PodDisruptionBudget protects availability
kind: PodDisruptionBudget                    # during voluntary disruptions
metadata:
  name: myapp
  namespace: production
spec:
  minAvailable: 8                            # Always keep 8 of 10 available
  selector:
    matchLabels: { app: myapp }
```

**Apply and verify:**

```bash
kubectl apply -f myapp.yaml
kubectl get deploy,rs,pods -n production -l app=myapp
kubectl rollout status deployment/myapp -n production
kubectl get svc myapp -n production    # EXTERNAL-IP shows the LB DNS
kubectl describe pod -n production -l app=myapp | head -50
```

**What this manifest does right (interview narrative):**
- **Resource requests AND limits** — no noisy neighbors.
- **Probes**: startup (boot grace), readiness (traffic gating), liveness (deadlock detection) — all three, distinct paths.
- **Security context**: non-root, read-only root FS, all capabilities dropped, no privilege escalation.
- **Topology spread + anti-affinity** so replicas don't all land on one node or one AZ.
- **PodDisruptionBudget** so voluntary disruption (node upgrades, scale-down) doesn't take us below 8 pods.
- **preStop sleep** for connection draining during termination.
- **IRSA-ready service account** for AWS API access without static credentials.
- **Prometheus scrape annotations** so monitoring works out of the box.
- **AWS NLB** via Load Balancer Controller annotations rather than the older in-tree provisioner.

---

## 9. Prometheus + Grafana integration

**Setup (on EKS, the Helm way):**

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

kubectl create namespace monitoring

helm install kps prometheus-community/kube-prometheus-stack \
  -n monitoring \
  --values - <<'EOF'
prometheus:
  prometheusSpec:
    retention: 30d
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          accessModes: ["ReadWriteOnce"]
          resources: { requests: { storage: 100Gi } }
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
grafana:
  adminPassword: from-secret
  persistence:
    enabled: true
    storageClassName: gp3
    size: 10Gi
  ingress:
    enabled: true
    ingressClassName: alb
    hosts: [grafana.company.com]
alertmanager:
  config:
    receivers:
    - name: pagerduty
      pagerduty_configs:
      - service_key: ${PD_SERVICE_KEY}
    route:
      receiver: pagerduty
      group_by: [alertname, cluster, severity]
EOF
```

This installs Prometheus + Alertmanager + Grafana + node-exporter + kube-state-metrics + a starter set of dashboards.

**App instrumentation:**

For Spring Boot, add Micrometer + Actuator:
```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```
The app then exposes `/actuator/prometheus`.

**Telling Prometheus to scrape the app — ServiceMonitor:**

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp
  namespace: production
  labels: { release: kps }
spec:
  selector:
    matchLabels: { app: myapp }
  endpoints:
  - port: http
    path: /actuator/prometheus
    interval: 30s
```

**Alerts — PrometheusRule:**

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: myapp-slos
  namespace: production
  labels: { release: kps }
spec:
  groups:
  - name: myapp.slo
    rules:
    - alert: HighErrorRate
      expr: |
        sum(rate(http_server_requests_seconds_count{app="myapp",status=~"5.."}[5m]))
        /
        sum(rate(http_server_requests_seconds_count{app="myapp"}[5m]))
        > 0.01
      for: 5m
      labels: { severity: page }
      annotations:
        summary: "Error rate >1% on myapp"
        runbook: "https://wiki.company.com/runbooks/myapp-errors"
    - alert: HighP99Latency
      expr: |
        histogram_quantile(0.99,
          sum by (le) (rate(http_server_requests_seconds_bucket{app="myapp"}[5m]))) > 1.0
      for: 10m
      labels: { severity: ticket }
```

**Grafana dashboard walkthrough:**

For an application like myapp I'd build a dashboard with these sections:

**Header — service summary**
- Big single stats: current request rate, error rate %, p95 latency, # healthy pods.
- Color-coded against SLO targets (green/yellow/red).

**RED metrics (Rate, Errors, Duration)** — Google's golden signals
- Time series: Request rate by HTTP method, broken down per endpoint.
- Time series: Error rate (5xx and 4xx separately), with annotations for deployments.
- Time series: Latency percentiles (p50, p95, p99) overlaid.

**USE metrics for resources (Utilization, Saturation, Errors)**
- CPU usage and CPU throttling — throttling is the killer signal that the CPU limit is too low.
- Memory usage and limit.
- Network I/O per pod.

**Kubernetes context**
- Pod count over time (with HPA min/max overlay).
- Restart count.
- Deployment rollout state.

**Dependencies**
- DB connection pool: active / idle / waiting.
- Cache hit/miss ratio.
- Downstream API latency and error rate (for upstream services we call).

**Business metrics**
- Orders/sec or transactions/sec or whatever maps to the product KPI.
- Drop here is usually the first sign of a problem the technical metrics haven't surfaced.

**Variables I'd add to the dashboard:**
- `$namespace`, `$pod`, `$instance` — so the same panels work across environments.
- Time range presets: last 1h, last 24h, last 7d.

**Annotations:**
- Overlay deployment events (from `kube_deployment_status_observed_generation` changes) so when latency spikes you can see "did we deploy at that time?"

In the interview I'd say: "The dashboard answers four questions in order — is the service up and serving traffic? Is it healthy (RED)? Is it constrained (USE)? Is the business metric we exist to serve being affected? That order matches how I triage during an incident."

---

## 10. 100/200 pods failing with insufficient IP addresses

This is the classic **VPC CNI IP exhaustion** problem on EKS, or **subnet exhaustion** on AKS. The symptom — `Failed to allocate IP address from CIDR block` or similar — points clearly.

**Diagnosis:**

```bash
# 1. Confirm the symptom from the failing pods
kubectl describe pod <failing-pod>
# Look for events like:
#   Warning  FailedCreatePodSandBox  ... InsufficientFreeAddressesInSubnet
#   Warning  FailedScheduling         ... failed to allocate IP

# 2. Where are the pods scheduled?
kubectl get pods -A -o wide | awk '{print $8}' | sort -u

# 3. Inspect subnet IP utilization (AWS)
aws ec2 describe-subnets --filters "Name=tag:kubernetes.io/cluster/$CLUSTER,Values=shared"
aws ec2 describe-network-interfaces --filters "Name=subnet-id,Values=subnet-..." \
  --query 'NetworkInterfaces[].PrivateIpAddresses[]' | jq 'length'

# 4. VPC CNI configuration
kubectl get ds aws-node -n kube-system -o yaml | grep -A5 env:
# WARM_IP_TARGET, WARM_ENI_TARGET, MINIMUM_IP_TARGET — these drive how aggressively
# the CNI pre-allocates IPs per node

# 5. Per-node IP usage
kubectl get nodes -o json | jq -r '.items[] | .metadata.name + " " + .status.capacity.pods'
kubectl describe node <node> | grep -A5 "Allocated resources"
```

**Root causes ranked by likelihood:**

1. **Subnet CIDR too small** — common in early-stage deployments where /24 subnets (256 IPs) made sense for a handful of pods but not at scale.
2. **WARM_IP_TARGET / WARM_ENI_TARGET on aws-node** consuming too many IPs preemptively without using them.
3. **Many nodes per AZ all in the same small subnet**, exhausting that subnet while other AZs have headroom.
4. **Pods stuck in Terminating** holding IPs that aren't released because the kubelet/CNI is hung.
5. **Stale ENIs** from rapidly cycled nodes — orphaned interfaces holding IPs.

**Immediate mitigation:**

- **Scale down non-essential workloads** to free IPs for the failing 100 pods.
- **Drain pods stuck in Terminating** to release IPs: `kubectl delete pod <pod> --grace-period=0 --force`.
- **Move workloads to a different node pool** with subnets in less-exhausted AZs.
- **Manually trigger ENI cleanup** if orphaned ENIs are visible: identify via `aws ec2 describe-network-interfaces --filters Name=status,Values=available` and delete.

**Long-term fixes:**

1. **Use AWS VPC CNI with prefix delegation** (the modern answer). Instead of one IP per pod on each ENI, the CNI allocates /28 prefixes (16 IPs) per ENI. Dramatically increases pod density per node.
   ```bash
   kubectl set env daemonset aws-node -n kube-system \
     ENABLE_PREFIX_DELEGATION=true \
     WARM_PREFIX_TARGET=1
   ```
   Replaces the old per-IP allocation model. Pods per node can jump from ~30 to ~110 on the same instance type.

2. **Use a secondary CIDR on the VPC** with `100.64.0.0/16` (CGN range) for pod IPs only — keeps service/node IPs on the original CIDR, gives massive pod IP space without re-IPing the VPC.

3. **Custom Networking** — Pods get IPs from a different (larger) CIDR than nodes. Standard pattern when the primary VPC CIDR can't be enlarged.

4. **Enlarge subnets** if possible — but this requires a VPC plan you got right at design time.

5. **Cilium CNI in tunnel mode** — pod IPs don't come from VPC subnets at all; they come from an overlay. Decouples pod density from VPC IP planning entirely.

**Prevention:**

- **Capacity planning**: at design time, calculate `pods per node × max nodes` and provision subnets accordingly. For a /24 (256 IPs) and 30 pods per node, you can only support ~8 nodes per AZ before exhaustion.
- **Monitoring**: CloudWatch alarm on subnet `AvailableIPAddressCount` < 20% threshold. This is the single most important alert to add after experiencing this incident.
- **Pre-stage prefix delegation** before scaling up — it's a single env var change but takes effect for new ENIs only, so retrofit it during a low-traffic window.

The interview answer: lead with diagnosis, give immediate mitigation, then the long-term fix (prefix delegation is the headline). Mention that the underlying lesson is "subnet sizing should be a Day 0 conversation with Networking, not a Day 100 incident."

---

## 11. ConfigMaps and Secrets

**ConfigMap** — non-sensitive configuration data stored as key-value pairs. Plain text in etcd.

**Secret** — sensitive data (passwords, tokens, certificates). Base64-encoded (which is *not* encryption). True encryption requires `--encryption-provider-config` on the API server using KMS, or external secret managers.

**Both consumed identically by pods:**

```yaml
# As env vars
envFrom:
- configMapRef: { name: app-config }
- secretRef: { name: app-secret }

# Or individually
env:
- name: LOG_LEVEL
  valueFrom: { configMapKeyRef: { name: app-config, key: log-level } }
- name: DB_PASSWORD
  valueFrom: { secretKeyRef: { name: app-secret, key: db-password } }

# As mounted files
volumes:
- name: config
  configMap: { name: app-config }
- name: secrets
  secret: { secretName: app-secret }
volumeMounts:
- { name: config,  mountPath: /etc/myapp }
- { name: secrets, mountPath: /etc/myapp/secrets, readOnly: true }
```

**ConfigMap use cases:**
- Application config files (`application.yaml`, `appsettings.json`).
- Feature flags.
- Log levels and verbosity.
- Public URLs and endpoints.
- Resource limits, timeouts, retry counts.
- Nginx config, fluent-bit config, sidecar config.

**Secret use cases:**
- Database passwords.
- API keys for third-party services.
- TLS certificates and private keys (`type: kubernetes.io/tls`).
- OAuth client secrets.
- Service account tokens.
- Docker registry credentials (`type: kubernetes.io/dockerconfigjson`).

**Production patterns I follow:**
- **Don't store secrets directly in etcd**. Use **External Secrets Operator** or **Secrets Store CSI Driver** to sync from AWS Secrets Manager / Azure Key Vault / HashiCorp Vault. Secrets stay in the dedicated KMS with rotation and audit.
- **Mount as files, not env vars**, when possible — env vars are visible to any process inside the container and can leak via crash dumps.
- **Enable etcd encryption at rest** with a KMS provider for defense in depth even with native Secrets.
- **RBAC tightly** — `get`/`list` on Secrets restricted to specific service accounts, not blanket namespace-admin.
- **Audit access** — Kubernetes audit logs to a SIEM with alerts on suspicious Secret reads.

**External Secrets Operator example** — secrets in AWS, synced to K8s:
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-secrets
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-store
    kind: SecretStore
  target:
    name: myapp-secrets
    creationPolicy: Owner
  data:
  - secretKey: db-password
    remoteRef:
      key: prod/myapp/db
      property: password
```

---

## 12. Deployment strategies — when to use which

Each strategy has a sweet spot. The decision factors: blast radius tolerance, infrastructure cost, traffic shape, stateful vs stateless, regulatory constraints.

**Rolling Update** — Kubernetes default. Gradually replace old pods with new ones controlled by `maxSurge` / `maxUnavailable`.

**Use when:**
- Stateless services with backwards-compatible API changes.
- Default for low-risk, frequent deploys.
- Cost-sensitive — no doubling of infrastructure.
- Most internal microservices in normal operation.

**Don't use when:**
- API has breaking changes (mixed-version state breaks consumers).
- DB schema changes that aren't backwards-compatible.
- Strict regulatory requirement to validate full version before any traffic.

**Blue-Green** — two identical environments; cut over traffic instantly via service selector flip or LB target swap.

**Use when:**
- Major version upgrade where you want full validation against production-like traffic before any users see it.
- Database migration that requires coordinated app+DB switchover.
- Tier-1 systems where rollback must be near-instant (one selector flip).
- Pre-cutover smoke tests / load tests against the green environment are valuable.
- Regulated environments needing pre-go-live validation evidence.

**Don't use when:**
- Cost-sensitive — you pay for 2× infrastructure during cutover.
- DB schema changes that can't support both versions simultaneously (use expand-contract).
- Long-running connections that don't drain quickly.

**Canary** — route a small slice of traffic to the new version, observe, gradually increase.

**Use when:**
- Risk-sensitive changes where you want production validation with limited blast radius.
- Performance-sensitive changes (algorithm, query optimization) where the only true test is real production traffic.
- Customer-facing user-experience changes — A/B style validation.
- Tier-1 systems with strong observability and clear SLOs (because automated promote/rollback depends on SLO measurement).
- Argo Rollouts / Flagger setups with analysis templates.

**Don't use when:**
- Service has very low traffic — 5% of 100 requests/hour is meaningless signal.
- Strong stickiness requirements (a user's session must hit one version or the other consistently — solvable but adds complexity).
- Lack of observability to make a confident promote/abort decision.

**Recreate** (kill all old, then start all new) — exists in Kubernetes but rarely a good choice.

**Use when:**
- Cannot run two versions simultaneously (singleton workload, exclusive resource lock).
- Cost optimization in dev where downtime is acceptable.
- Stateful migration scenarios where you explicitly want everything stopped before starting new.

**My decision framework in plain language for the interview:**

> "Rolling is the default for everyday microservice deploys — cheap, simple, safe enough when changes are backwards-compatible. Blue-green is what I reach for when I need an instant rollback or pre-cutover validation, accepting the cost of running two environments. Canary is for risk-sensitive changes where I want production traffic to validate me, paired with automated rollback on SLO breach. In practice, my pipelines run rolling by default and Argo Rollouts canary for any deploy touching tier-1 services. Blue-green is reserved for major version cuts or coordinated DB-and-app migrations."

---

## 13. Last production issue I fixed

Have one prepared and rehearsed. Structure: situation → action → decision points → outcome → lessons. Here's a realistic one:

> **Situation (3 min):** "On a Friday afternoon — yes, really — we got paged on a Tier-1 payments service in EKS. Error rate spiked from baseline 0.05% to 8% across the whole service, p99 latency went from 200ms to 12 seconds. Customer-facing impact: payment failures and timeouts. Severity-1.
>
> **Triage:** First move was to declare the incident in Slack and roll the affected service back via `kubectl rollout undo` — we had a green deploy 2 hours earlier. Rollback brought error rate down within 3 minutes. Mitigation first, investigation after.
>
> **Diagnosis:** With pressure off, we dug into what the green deploy changed. The diff: a new Spring Boot dependency had been added, which transitively bumped HikariCP (the JDBC connection pool) to a major version with a different default for `connectionTimeout`. Old default was 30s, new default was 250ms. Under load, the pool was returning failures instead of waiting for connections.
>
> **Root cause confirmation:** Reproduced in staging by enabling the same dependency and load-testing. Pool exhaustion logs were unmistakable once we knew where to look.
>
> **Fix:** Explicitly set `spring.datasource.hikari.connectionTimeout=30000` in application config, redeployed via the canary pipeline. Watched the metrics for 30 minutes; no regression.
>
> **Outcome:** Total customer impact was ~7 minutes from spike to rollback. About 4,000 failed transactions, all of which were retried successfully by the client (idempotency keys). No data loss.
>
> **Lessons and actions (the part interviewers care about):**
> 1. **Dependency-bump audit gap** — a transitive version bump changed behavior silently. We added a Maven Enforcer rule pinning known-critical libraries (HikariCP, Tomcat, Jackson) so unintended upgrades fail the build.
> 2. **Better canary signals** — the deploy that introduced this passed our 'error rate < 1%' canary check because the canary slice didn't hit pool capacity. We added 'database pool wait time' and 'pool acquisition errors' to the canary analysis template, so the rollout would have caught this before going to 100%.
> 3. **Friday deploy policy** — formalized: no deploys to Tier-1 services after 2pm Friday or before 10am Monday, unless emergency. The team was annoyed at first; agreed later it was the right move.
> 4. **Postmortem published org-wide** within 72 hours, action items tracked in Jira, all completed within two weeks."

**What makes this answer good in an interview:**
- Specific symptoms with numbers.
- Clear decision points (rollback first vs investigate).
- Real root cause, not "it was the network."
- Action items that show you turn incidents into systemic improvements.
- Honest about timing and customer impact.

---

## 14. 3-tier architecture on AWS — from scratch

A classic 3-tier: **presentation** (web/UI), **application** (business logic, API), **data** (persistence). Here's how I'd design it on AWS.

**Networking foundation:**
- **VPC** with non-overlapping CIDR (`10.0.0.0/16` — large enough to grow).
- **Three Availability Zones** for HA.
- **Per-AZ subnet layout, three tiers**:
  - Public subnets (`10.0.0.0/24`, `10.0.1.0/24`, `10.0.2.0/24`) — load balancers, NAT Gateways.
  - Private app subnets (`10.0.10.0/24`, `10.0.11.0/24`, `10.0.12.0/24`) — EKS nodes, EC2 app servers.
  - Private data subnets (`10.0.20.0/24`, `10.0.21.0/24`, `10.0.22.0/24`) — RDS, ElastiCache.
- **Internet Gateway** for public subnets.
- **NAT Gateways per AZ** for private-subnet egress (no SPOF).
- **VPC Endpoints** for S3, DynamoDB (Gateway endpoints — free), and ECR/Secrets Manager/CloudWatch (Interface endpoints) — keeps AWS API traffic off NAT/internet.
- **Route tables** per subnet tier.
- **NACLs** at subnet boundary (defense in depth alongside SGs).

**Tier 1: Presentation**
- **CloudFront** distribution as the global edge — caches static assets, terminates TLS at edge, integrates with AWS WAF for OWASP rules + bot protection.
- **Static frontend assets** (React/Angular bundles) hosted on **S3** behind CloudFront.
- **Route 53** for DNS, alias records to CloudFront, optionally with health-check-based failover.
- **ACM** for TLS certificates.

**Tier 2: Application**
- **EKS cluster** running in private app subnets (or ECS Fargate for simpler stateless workloads).
- **Application Load Balancer** in public subnets, fronting the cluster's ingress. WAF attached.
- **AWS Load Balancer Controller** managing ALBs from Kubernetes Ingress resources.
- **HPA + Cluster Autoscaler / Karpenter** for elastic scaling.
- **Microservices** as deployments behind ClusterIP Services, routed by Ingress.
- **Service Mesh (Istio or App Mesh)** optional, for inter-service mTLS and observability.

**Tier 3: Data**
- **RDS Aurora PostgreSQL Multi-AZ** in private data subnets — primary + replicas, automatic failover.
- **DynamoDB** for high-write low-latency workloads (event log, sessions).
- **ElastiCache Redis** for caching layer, cluster mode enabled, Multi-AZ.
- **S3** for object storage with lifecycle policies.
- **All encrypted at rest** with KMS CMKs.

**Cross-cutting:**
- **IAM**: roles for EC2 instances, IRSA for EKS pods. No static credentials.
- **Secrets**: AWS Secrets Manager for DB credentials, third-party API keys. External Secrets Operator in EKS to sync.
- **KMS**: CMKs for EBS, RDS, S3, Secrets Manager. Rotation enabled.
- **Observability**: CloudWatch (Logs, Metrics, Alarms), X-Ray for tracing. Plus Prometheus + Grafana + Loki running in-cluster.
- **CI/CD**: GitHub Actions or CodePipeline → ECR → ArgoCD-driven deploys to EKS.
- **DR**: depending on RTO/RPO, warm standby in second region with Aurora Global Database + cross-region S3 replication + Route 53 health-check failover.
- **Governance**: AWS Organizations with separate accounts (network, security, prod, non-prod, sandbox). Control Tower for baseline. Config rules + Security Hub + GuardDuty.
- **Cost**: tagging, Budgets, Cost Explorer, Compute Savings Plans for baseline workloads.

**Security:**
- **Security groups** at every tier — ALB allows 80/443 from internet; EKS nodes allow 8080 from ALB SG only; RDS allows 5432 from EKS node SG only. Zero-trust between tiers.
- **WAF** on ALB and CloudFront.
- **GuardDuty** for threat detection.
- **CloudTrail** to a separate logging account.
- **VPC Flow Logs** to CloudWatch / S3.
- **AWS Inspector** for EC2 / ECR vulnerability scanning.

**Diagram I'd sketch on the whiteboard (in words):**
```
User → Route 53 → CloudFront (WAF) → 
       ├── S3 (static assets)
       └── ALB (public subnets, WAF) → 
                 EKS (private app subnets) → 
                       ├── Aurora (private data subnets, Multi-AZ)
                       ├── ElastiCache Redis (private data subnets)
                       ├── DynamoDB
                       └── S3
```

In the interview I'd walk through this in 4-5 minutes and offer to go deeper on any layer they want — usually they pick networking or security to probe.

---

## 15. DevOps tool integration + OIDC

**Tool chain integration (the "how"):**

The platform I've built/operated integrates roughly like this:

```
Developer commit
    ↓
GitHub / Bitbucket  → webhook →
    ↓
GitHub Actions / Jenkins
    ├── SonarQube ← (quality gate webhook → pipeline)
    ├── Snyk / Trivy (vulnerability scans)
    ├── JFrog Artifactory / ECR (artifact storage)
    │     ↑
    └── Updates GitOps repo
              ↓
ArgoCD (in-cluster) → reconciles → EKS
              ↓
Prometheus scrapes pods → Grafana visualizes
Alertmanager → PagerDuty / Slack
Fluent Bit → Loki / CloudWatch
              ↓
Terraform Cloud / pipelines manage underlying AWS infra
```

**Glue between tools:**
- **Webhooks**: GitHub push triggers Actions; SonarQube quality gate result posts back; ArgoCD notifications post to Slack.
- **OIDC**: GitHub Actions → AWS, ArgoCD → AWS (IRSA), and so on. No long-lived credentials.
- **API tokens** for service-to-service calls where OIDC isn't supported yet, stored in the secrets manager.
- **Shared libraries / reusable workflows**: a Jenkins shared library or GitHub reusable workflows codifies the standard pipeline — teams onboard by referencing the library, not by copying YAML.
- **GitOps repo as integration substrate**: CI updates Git, ArgoCD reads Git. Git is the durable contract between build-time and run-time.

**OIDC — what it is and where to use it:**

OIDC = **OpenID Connect**, an authentication layer built on top of OAuth 2.0. It provides identity claims (who you are) via a signed token (JWT) issued by an Identity Provider (IdP).

In the DevOps world, OIDC specifically means **federated authentication** — short-lived tokens minted by a trusted IdP, exchanged for cloud credentials at runtime, eliminating the need to store long-lived static credentials in CI/CD systems.

**How GitHub Actions → AWS via OIDC works:**

1. AWS account is configured with GitHub's OIDC issuer (`token.actions.githubusercontent.com`) as a trusted identity provider.
2. An IAM role is created with a trust policy allowing assumption by tokens from a specific repo + branch:
   ```json
   {
     "Effect": "Allow",
     "Principal": { "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com" },
     "Action": "sts:AssumeRoleWithWebIdentity",
     "Condition": {
       "StringEquals": { "...aud": "sts.amazonaws.com" },
       "StringLike":   { "...sub": "repo:company/myapp:ref:refs/heads/main" }
     }
   }
   ```
3. At job runtime, GitHub mints an OIDC token signed by GitHub.
4. The Action exchanges the token for AWS credentials via `sts:AssumeRoleWithWebIdentity`.
5. AWS verifies the token's signature against GitHub's public keys (JWKS), checks the claims match the role's trust policy conditions, and issues temporary credentials (typically 1 hour).
6. The job uses those short-lived credentials.

**Why this matters:**
- **No long-lived AWS access keys in GitHub** — credentials can't leak via accidentally-pushed config or compromised secret store.
- **Scoped trust** — only `main` branch in a specific repo can assume the role.
- **Short-lived** — even if a token leaks, it's useless after the job ends.
- **Auditable** — CloudTrail shows which workflow assumed which role at what time.

**Where else OIDC is used:**
- **EKS IRSA (IAM Roles for Service Accounts)** — pods authenticate to AWS via OIDC tokens from the cluster's OIDC issuer. Same pattern, applied to in-cluster workloads.
- **Azure DevOps → Azure** — workload identity federation.
- **GitLab CI → AWS / Azure / GCP** — same pattern as GitHub.
- **Terraform Cloud → AWS** — dynamic provider credentials.
- **HashiCorp Vault** — JWT/OIDC auth method for workloads.
- **Kubernetes API authentication** — `--oidc-issuer-url` lets your IdP (Entra ID, Okta, Google) authenticate `kubectl` users.
- **End-user SSO to web applications** — the original OIDC use case (Sign in with Google, etc.).

**Similar tools / standards worth naming in the interview:**

| Standard / tool | What it does |
|---|---|
| **OAuth 2.0** | Authorization framework (delegated access). OIDC is built on it for *authentication*. |
| **SAML 2.0** | XML-based federated authentication, predates OIDC. Still common for enterprise SSO (Okta, ADFS, Azure AD apps). |
| **JWT (JSON Web Token)** | The token format used by OIDC. Not a protocol on its own. |
| **SPIFFE / SPIRE** | Identity standard for workload-to-workload authentication via X.509 certs. Service mesh / zero-trust. |
| **Kerberos** | Older enterprise authentication standard (Active Directory). Still in heavy use in some environments. |
| **mTLS** | Mutual TLS authentication — both client and server present certificates. Different from OIDC (cert-based identity vs token-based). |
| **AWS STS AssumeRole** | AWS-specific role assumption mechanism. OIDC federation uses `AssumeRoleWithWebIdentity`, a flavor of STS. |
| **HashiCorp Vault** | Not a protocol, but a tool that issues dynamic short-lived credentials — same goal as OIDC federation, broader scope. |
| **AWS IAM Identity Center / Azure AD / Okta** | Identity providers that issue OIDC/SAML tokens. |

**The principle to articulate:** OIDC federation is part of a broader move from "long-lived shared secrets" to "short-lived identity-scoped credentials." Other tools in this space — SPIFFE for workload identity, mTLS for service mesh, Vault for dynamic secrets — all share the same goal: minimize blast radius by minimizing credential lifetime and scope.

---

That's the full round. Likely follow-ups to be ready for:

- **"Walk me through the IAM trust policy for OIDC"** — they may want to see the JSON and have you explain each condition.
- **"How do you debug an ArgoCD sync stuck in OutOfSync?"** — sync status, diff, logs, manual sync with force.
- **"How would you design the same architecture for compliance with PCI-DSS / HIPAA?"** — separate accounts, dedicated CHD network, KMS CMKs, encryption everywhere, audit log immutability.
- **"What's wrong with this Terraform / Kubernetes manifest?"** — they share a snippet and expect a security/reliability review.
- **"How do you handle stateful workloads on Kubernetes?"** — StatefulSets, operators, PV reclaim policies, backup strategies (Velero), and "honestly, prefer managed services for production DBs."