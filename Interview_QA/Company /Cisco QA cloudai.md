# DevOps / SRE Interview Prep — Detailed Answers

---

## Round 1

### 1. Describe a situation where you had to improve the reliability of a critical system.

**How to structure this (STAR format):**

- **Situation:** A production payment/order-processing service was experiencing intermittent 5xx errors and occasional full outages during peak traffic, causing SLA breaches.
- **Task:** Reduce downtime and error rate, and prevent recurrence.
- **Action (this is where the technical depth goes):**
  1. **Root-cause first, don't guess.** Pulled logs, metrics, and traces (via Prometheus/Grafana + distributed tracing like Jaeger) around the incident windows. Found the service was hitting connection pool exhaustion on the DB during traffic spikes, plus a single point of failure — only one replica behind the load balancer.
  2. **Fixed immediate risk:** Increased DB connection pool limits with proper queuing/backpressure, added a circuit breaker (e.g., resilience4j / Istio circuit breaking) so a slow downstream dependency didn't cascade failures upward.
  3. **Eliminated single points of failure:** Moved from 1 replica to a min of 3 across multiple AZs, added horizontal pod autoscaling (HPA) based on CPU + custom request-latency metrics.
  4. **Added health checks:** Proper liveness/readiness probes so Kubernetes stopped routing traffic to pods that were up but not actually ready (was previously just a TCP check).
  5. **Introduced retries with exponential backoff + jitter** on client calls, and idempotency keys so retries didn't create duplicate transactions.
  6. **Load tested** the fix with k6/Locust before rolling to production, simulating 3x expected peak load.
  7. **Rolled out gradually** using a canary deployment (5% → 25% → 100%) with automated rollback tied to error-rate alerts.
- **Result:** Error rate dropped from ~4% to <0.05%, P99 latency improved from 2.8s to 400ms, and the service maintained 99.95% uptime over the following two quarters. Also documented the incident as a postmortem and added the failure scenario to the runbook.

**Key point for interviewers:** always frame it as detect → diagnose → fix root cause → prevent recurrence (not just "restarted the pod"), and mention measurable before/after numbers.

---

### 2. What proactive monitoring solutions have you implemented in your projects?

Proactive monitoring means catching problems **before** users notice, not just alerting after an outage. Typical stack and approach:

**Metrics layer**
- **Prometheus** for scraping metrics (node_exporter for host metrics, kube-state-metrics for cluster state, application-level custom metrics via client libraries).
- **Grafana** dashboards for visualization — RED metrics (Rate, Errors, Duration) for services, USE metrics (Utilization, Saturation, Errors) for resources.

**Logging layer**
- Centralized logging via **ELK/EFK stack** (Elasticsearch, Fluentd/Filebeat, Kibana) or **Loki + Grafana** for lighter-weight log aggregation.
- Structured (JSON) logging so logs are queryable, with correlation IDs to trace a request across services.

**Tracing**
- **Jaeger/Zipkin** or **OpenTelemetry** for distributed tracing — critical in microservices to find which hop is slow.

**Alerting**
- **Alertmanager** (with Prometheus) or **PagerDuty/Opsgenie** integration for on-call routing.
- Alerts based on **SLO burn rate**, not just static thresholds — e.g., "error budget will be exhausted in 4 hours at this rate" rather than "CPU > 90%".
- Alert fatigue reduction: grouping, deduplication, and only paging on symptoms that affect users (not every underlying cause).

**Synthetic monitoring**
- Scheduled synthetic transactions (via Blackbox Exporter, Pingdom, or Postman/Newman) to simulate real user flows (login, checkout) from outside the cluster, catching issues before real users report them.

**Infrastructure/cost monitoring**
- CloudWatch (AWS) / Azure Monitor / Stackdriver for cloud-native metrics, tied into the same Grafana view where possible for a single pane of glass.

**Proactive practices**
- Capacity planning dashboards (trend lines, not just current state) to predict when you'll run out of disk/memory before it happens.
- Automated anomaly detection (e.g., Prometheus recording rules + `predict_linear`, or AWS Look­out for Metrics) to flag unusual patterns.
- Chaos engineering (e.g., Chaos Mesh, Gremlin) run periodically in staging to validate that monitoring/alerting actually fires when it should.

---

### 3. Nginx Ansible Playbook + Secrets Management

**Playbook to install Nginx, ensure it's started and enabled on boot:**

```yaml
---
- name: Install and configure Nginx
  hosts: webservers
  become: true
  vars:
    nginx_port: 80

  tasks:
    - name: Update apt cache (Debian/Ubuntu)
      apt:
        update_cache: yes
        cache_valid_time: 3600
      when: ansible_os_family == "Debian"

    - name: Install Nginx (Debian/Ubuntu)
      apt:
        name: nginx
        state: present
      when: ansible_os_family == "Debian"

    - name: Install Nginx (RedHat/CentOS)
      yum:
        name: nginx
        state: present
      when: ansible_os_family == "RedHat"

    - name: Ensure Nginx config directory exists
      file:
        path: /etc/nginx/conf.d
        state: directory
        mode: '0755'

    - name: Deploy custom Nginx site config (optional)
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/conf.d/app.conf
        owner: root
        group: root
        mode: '0644'
      notify: Reload nginx

    - name: Start and enable Nginx service
      service:
        name: nginx
        state: started
        enabled: true

    - name: Verify Nginx is listening
      wait_for:
        port: "{{ nginx_port }}"
        timeout: 15

  handlers:
    - name: Reload nginx
      service:
        name: nginx
        state: reloaded
```

**Why this is "correct" for an interview:**
- Uses `become: true` for privilege escalation rather than assuming root login.
- OS-family conditionals so it's portable across Debian and RHEL families.
- `state: started` + `enabled: true` covers both "run now" and "run on every boot".
- Uses a **handler** for config reload (only restarts if config actually changed — idempotent).
- `wait_for` gives a post-deploy verification step, not just "assume it worked".

**Managing secrets in Ansible:**

1. **Ansible Vault** (most common, built-in):
   ```bash
   # Create an encrypted file
   ansible-vault create group_vars/all/vault.yml

   # Edit it later
   ansible-vault edit group_vars/all/vault.yml

   # Encrypt an existing file
   ansible-vault encrypt secrets.yml

   # Run playbook with vault password
   ansible-playbook site.yml --ask-vault-pass
   # or with a password file (kept out of git, permissions 600)
   ansible-playbook site.yml --vault-password-file ~/.vault_pass.txt
   ```
   - Store the vault password itself in a secrets manager or CI secret store, never in the repo.
   - Use **vault IDs** (`--vault-id dev@prompt`, `--vault-id prod@~/.vault_prod`) to support multiple environments with different passwords.
   - Keep only the sensitive variables in the vault file (`vault.yml`) and reference them from a plaintext `vars.yml` using a naming convention (`vault_db_password` → `db_password: "{{ vault_db_password }}"`). This lets you `grep` variable names without decrypting.

2. **External secret backends** (better for larger teams / dynamic secrets):
   - **HashiCorp Vault** via the `community.hashi_vault` collection — pull secrets at runtime instead of storing encrypted blobs in git.
   - **AWS Secrets Manager / SSM Parameter Store** via `amazon.aws.aws_secret` lookup plugin.
   - These are preferable when secrets need rotation, fine-grained access control, or audit logging — Vault files are static and harder to rotate/audit.

3. **General hygiene:**
   - Never echo secrets in task output — use `no_log: true` on tasks that handle sensitive data.
   - Restrict who can decrypt (vault password distribution) via least privilege.
   - CI/CD pipelines inject the vault password as a masked secret variable, never checked into the pipeline config.

---

### 4. Migrating a Terraform Backend from Local to Remote (S3 + DynamoDB)

**Steps:**

1. **Create the backend infrastructure first** (S3 bucket + DynamoDB table), typically with a small bootstrap Terraform config that itself uses local state (chicken-and-egg problem — this one stays local or is managed separately).

```hcl
resource "aws_s3_bucket" "tf_state" {
  bucket = "my-company-terraform-state"

  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_s3_bucket_versioning" "tf_state" {
  bucket = aws_s3_bucket.tf_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "tf_state" {
  bucket = aws_s3_bucket.tf_state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "tf_state" {
  bucket                  = aws_s3_bucket.tf_state.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_dynamodb_table" "tf_lock" {
  name         = "terraform-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

2. **Add the backend block** to your existing project (the one currently using local state):

```hcl
terraform {
  backend "s3" {
    bucket         = "my-company-terraform-state"
    key            = "envs/prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-lock"
    encrypt        = true
  }
}
```

3. **Run the migration:**
   ```bash
   terraform init -migrate-state
   ```
   Terraform detects the backend change, prompts: *"Do you want to copy existing state to the new backend?"* → answer `yes`. It copies the local `terraform.tfstate` into the S3 key you specified.

4. **Verify:**
   ```bash
   terraform state list         # confirms resources are still tracked
   aws s3 ls s3://my-company-terraform-state/envs/prod/
   ```
   Run `terraform plan` — it should show **no changes** if migration was clean.

5. **Clean up:** back up and then remove the local `terraform.tfstate` / `terraform.tfstate.backup` from the repo (and make sure they're in `.gitignore` — local state should never be committed anyway).

6. **Team rollout:** everyone re-runs `terraform init` to pick up the new backend config; make sure IAM permissions for the S3 bucket/DynamoDB table are granted to all engineers/CI roles that need to run Terraform.

---

### 5. What happens if Terraform state becomes corrupted, and how would you recover?

**Symptoms of corruption:** `terraform plan`/`apply` fails with JSON parse errors, resources shown as missing/duplicated that don't match real infra, or a state file that's truncated/empty.

**Recovery approach, in order of preference:**

1. **Restore from versioned backend (best case).** If using S3 with versioning enabled (as configured above), just restore the previous object version:
   ```bash
   aws s3api list-object-versions --bucket my-company-terraform-state --prefix envs/prod/terraform.tfstate
   aws s3api get-object --bucket my-company-terraform-state --key envs/prod/terraform.tfstate \
     --version-id <good-version-id> terraform.tfstate.restored
   ```
   Then push it back as current, or use `terraform state push terraform.tfstate.restored`.

2. **Use the automatic local backup.** Terraform writes `terraform.tfstate.backup` before most state-changing operations (for local backend). For S3 backend, DynamoDB locking prevents concurrent corruption, but you still rely on S3 versioning rather than a `.backup` file.

3. **Rebuild state via `terraform import`** (if no backup exists at all). For each real resource in the cloud, re-import it into a fresh state:
   ```bash
   terraform import aws_instance.web i-0123456789abcdef0
   terraform import aws_security_group.web_sg sg-0123456789abcdef0
   ```
   This is tedious for large infra, so it's the last resort — hence why remote state with versioning is set up in the first place.

4. **Use `terraform state` subcommands to repair partial corruption:**
   ```bash
   terraform state list                     # see what's tracked
   terraform state show aws_instance.web    # inspect a resource
   terraform state rm aws_instance.web      # remove a bad entry, then re-import
   terraform state mv aws_instance.old aws_instance.new   # fix renamed resources
   ```

5. **Always work off a fresh `terraform plan`** after any recovery step and diff it carefully against the real infrastructure before applying — a botched recovery can lead to Terraform trying to "fix" things by destroying real resources.

**Prevention (the more important interview point):**
- Remote backend with **versioning enabled** (S3 versioning or Terraform Cloud's state history).
- **State locking** (DynamoDB) to prevent two people applying simultaneously, which is the most common cause of corruption.
- Never manually edit `.tfstate` by hand — use `terraform state` commands.
- Separate state files per environment/component (smaller blast radius).
- CI/CD as the only path to `apply` in production, avoiding ad-hoc local applies.

---

### 6. Terraform Code: EC2 Instance with SSH-Only Security Group

```hcl
provider "aws" {
  region = "us-east-1"
}

variable "allowed_ssh_cidr" {
  description = "CIDR block allowed to SSH into the instance"
  type        = string
  default     = "203.0.113.0/32"   # replace with your actual IP/CIDR, never 0.0.0.0/0 in prod
}

resource "aws_security_group" "ssh_only" {
  name        = "ssh-only-sg"
  description = "Allow inbound SSH only"
  vpc_id      = var.vpc_id

  ingress {
    description = "SSH from trusted CIDR"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.allowed_ssh_cidr]
  }

  egress {
    description = "Allow all outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "ssh-only-sg"
  }
}

variable "vpc_id" {
  type = string
}

variable "subnet_id" {
  type = string
}

variable "key_name" {
  type = string
}

resource "aws_instance" "app_server" {
  ami                    = "ami-0c101f26f147fa7fd"  # example Amazon Linux 2 AMI, region-specific
  instance_type          = "t3.micro"
  subnet_id              = var.subnet_id
  vpc_security_group_ids = [aws_security_group.ssh_only.id]
  key_name               = var.key_name

  tags = {
    Name = "app-server"
  }
}

output "instance_public_ip" {
  value = aws_instance.app_server.public_ip
}
```

**Interview talking points:**
- SG only opens port 22, restricted to a specific CIDR (never `0.0.0.0/0` for SSH in a real environment).
- Variables used for AMI/VPC/subnet/key so it's reusable across environments.
- In practice you'd also mention using **SSM Session Manager** instead of exposing SSH at all, which removes the need for an open inbound port entirely.

---

### 7. Setting up a Multi-Branch Jenkins Pipeline for a GitHub Repo

**Steps:**

1. **Install plugins:** GitHub Branch Source plugin (usually bundled with the Pipeline plugin suite), Pipeline, Credentials Binding.
2. **Create credentials in Jenkins:** a GitHub Personal Access Token (or GitHub App credentials, preferred for scale) stored in Jenkins Credentials Manager — scoped to read repo + webhook management.
3. **Create a new item → "Multibranch Pipeline"**, point it at the GitHub org/repo using the credential from step 2.
4. **Branch discovery strategy:** configure which branches/PRs to build — e.g., "All branches", or just branches with an open PR, to avoid building every feature branch unnecessarily.
5. **Add a `Jenkinsfile`** at the repo root — Jenkins auto-discovers it per branch, so every branch can have (or override) its own pipeline definition:
   ```groovy
   pipeline {
     agent any
     stages {
       stage('Checkout') {
         steps { checkout scm }
       }
       stage('Build') {
         steps { sh 'make build' }
       }
       stage('Test') {
         steps { sh 'make test' }
       }
       stage('Deploy') {
         when { branch 'main' }
         steps { sh './deploy.sh' }
       }
     }
   }
   ```
6. **Configure webhooks:** GitHub → repo settings → webhooks, pointing to `https://<jenkins-url>/github-webhook/`, so pushes/PRs trigger builds automatically instead of polling (polling wastes resources and adds latency).
7. **Set branch-specific behavior using `when` blocks** (e.g., only deploy from `main`, only run integration tests on PRs targeting `main`).
8. **PR build status integration:** Jenkins posts build status back to GitHub PR checks (via the GitHub plugin), blocking merge until CI passes if branch protection rules require it.
9. **Housekeeping:** configure "Orphaned Item Strategy" to auto-clean jobs for deleted branches, and set build discarders to limit retained build history.

---

### 8. Implementing Dynamic Stages in a Jenkinsfile Based on Environment Variables

Two common approaches — **conditional `when` blocks** (simpler, declarative) or **scripted pipeline with a loop** (more flexible for truly dynamic stage lists).

**Approach A — conditional stages (declarative):**
```groovy
pipeline {
  agent any
  environment {
    DEPLOY_ENV = "${params.DEPLOY_ENV ?: 'dev'}"
  }
  stages {
    stage('Build') {
      steps { sh 'make build' }
    }
    stage('Deploy to Dev') {
      when { environment name: 'DEPLOY_ENV', value: 'dev' }
      steps { sh './deploy.sh dev' }
    }
    stage('Deploy to Staging') {
      when { environment name: 'DEPLOY_ENV', value: 'staging' }
      steps { sh './deploy.sh staging' }
    }
    stage('Deploy to Prod') {
      when {
        allOf {
          environment name: 'DEPLOY_ENV', value: 'prod'
          branch 'main'
        }
      }
      steps {
        input message: 'Approve production deploy?'
        sh './deploy.sh prod'
      }
    }
  }
}
```

**Approach B — truly dynamic stage generation (scripted):**
```groovy
def environments = env.TARGET_ENVS ? env.TARGET_ENVS.split(',') : ['dev']

pipeline {
  agent any
  stages {
    stage('Build') {
      steps { sh 'make build' }
    }
    stage('Dynamic Deploys') {
      steps {
        script {
          def stageMap = [:]
          environments.each { envName ->
            stageMap["Deploy-${envName}"] = {
              sh "./deploy.sh ${envName}"
            }
          }
          parallel stageMap
        }
      }
    }
  }
}
```
This reads a comma-separated `TARGET_ENVS` variable (e.g., set as a build parameter or upstream job) and generates one parallel deploy stage per environment — useful when the number of targets isn't fixed at pipeline-authoring time (e.g., deploying to a variable number of regions).

**Key interview point:** declarative `when` is preferred for readability/auditability when the set of possible stages is known in advance; scripted pipelines with loops are used only when stage count/names are genuinely data-driven.

---

### 9. Kubernetes Cluster Upgrade with Zero Downtime

**General process (managed, e.g., EKS/GKE/AKS, or self-managed via kubeadm):**

1. **Pre-upgrade checks:**
   - Read the release notes / deprecated API list for the target version (`kubectl convert`, `pluto`, or `kubent` to detect deprecated APIs in use).
   - Confirm add-ons (CNI, CSI driver, ingress controller, cluster autoscaler) are compatible with the target version.
   - Take an etcd backup (self-managed) or confirm managed-service snapshot exists.
   - Ensure workloads have **PodDisruptionBudgets (PDBs)** defined so upgrades don't evict too many pods of the same app simultaneously.
   - Confirm applications have **readiness probes** and multiple replicas across nodes/AZs — a prerequisite for zero-downtime, not an upgrade-time fix.

2. **Upgrade control plane first:**
   - Managed: trigger via console/CLI (`eksctl upgrade cluster`, `gcloud container clusters upgrade`), which handles this with minimal risk since control plane is usually HA already.
   - Self-managed: upgrade one control-plane node at a time (`kubeadm upgrade plan` → `kubeadm upgrade apply vX.Y.Z`), keeping quorum on etcd throughout.
   - Verify: `kubectl get nodes`, `kubectl get componentstatuses` (or `kubectl get --raw='/healthz'`), confirm API server responds normally.

3. **Upgrade worker nodes with a rolling strategy — this is where zero downtime is actually achieved:**
   - **Node by node (or node group by node group):**
     1. `kubectl cordon <node>` — stop new pods from scheduling there.
     2. `kubectl drain <node> --ignore-daemonsets --delete-emptydir-data` — evicts pods gracefully, respecting PDBs; pods reschedule onto other nodes.
     3. Upgrade the node (new AMI/kubelet version, or replace with a new node in a managed node group / launch template).
     4. `kubectl uncordon <node>` (or the new node joins ready).
     5. Move to the next node — only proceed once the current node's pods are confirmed healthy elsewhere.
   - For managed node groups (EKS), this is often automated (`eksctl upgrade nodegroup` or rolling update in the ASG/launch template), but the same drain/cordon logic happens under the hood.
   - Because there are always multiple replicas spread across multiple nodes, draining one node at a time never removes all replicas of a service simultaneously — this is the actual mechanism behind "zero downtime."

4. **Upgrade add-ons/CNI/CSI drivers** to versions compatible with the new control plane, following vendor-specific order requirements (usually after control plane, before/with worker nodes).

5. **Canary the upgrade** on a non-prod cluster or a subset of node groups first, if the org has multiple clusters/environments.

---

### 10. What Key Things Should You Verify Post-Upgrade?

- **Node status:** `kubectl get nodes -o wide` — all nodes `Ready`, correct kubelet version reported.
- **Control plane health:** API server, scheduler, controller-manager all healthy; `kubectl version` shows expected server version.
- **Workload health:** `kubectl get pods -A | grep -v Running` — no unexpected `CrashLoopBackOff`/`Pending`/`Evicted` pods; check restart counts didn't spike.
- **Deprecated API usage:** confirm nothing is silently broken due to removed API versions (this is the #1 cause of post-upgrade incidents — e.g., `extensions/v1beta1 Ingress` removed in 1.22+).
- **Core add-ons:** CoreDNS resolving correctly, CNI plugin healthy (pod-to-pod and pod-to-service networking working), CSI provisioning still working (test a PVC create).
- **Ingress/Load balancer:** confirm external traffic routing still works end-to-end (synthetic check against a real endpoint, not just internal health).
- **Autoscaling:** HPA and Cluster Autoscaler functioning (trigger a small scale event to confirm).
- **Application-level smoke tests:** run your standard smoke/integration test suite against the cluster.
- **Metrics/logging pipeline:** confirm Prometheus/Grafana/logging agents are still scraping correctly post-upgrade (agents sometimes need version bumps too).
- **Security posture:** RBAC policies, PodSecurityStandards/PSA, network policies still enforced as expected (upgrades occasionally change default behavior).
- **Rollback plan validated:** confirm you *could* roll back (previous node group / AMI still available, etcd backup verified restorable) even after declaring success — don't delete rollback artifacts too early.

---

### 11. Script: Monitor a Directory and Auto-Copy New Files via SCP

**Using `inotifywait` (Linux, event-driven — efficient, no polling):**

```bash
#!/bin/bash
# monitor_and_scp.sh
# Watches a local directory and SCPs any new file to a remote server as it appears.

set -euo pipefail

WATCH_DIR="/data/outbound"
REMOTE_USER="deploy"
REMOTE_HOST="192.0.2.10"
REMOTE_DIR="/data/incoming"
SSH_KEY="/home/deploy/.ssh/id_rsa"
LOG_FILE="/var/log/monitor_scp.log"

log() {
  echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" >> "$LOG_FILE"
}

command -v inotifywait >/dev/null 2>&1 || { echo "inotify-tools not installed"; exit 1; }

log "Starting directory watch on $WATCH_DIR"

inotifywait -m -e close_write -e moved_to --format '%f' "$WATCH_DIR" | while read -r NEWFILE
do
  FULL_PATH="${WATCH_DIR}/${NEWFILE}"

  # Guard against partially-written files by re-checking size stability
  sleep 1
  if [ -f "$FULL_PATH" ]; then
    SIZE1=$(stat -c%s "$FULL_PATH")
    sleep 1
    SIZE2=$(stat -c%s "$FULL_PATH")

    if [ "$SIZE1" -eq "$SIZE2" ]; then
      if scp -i "$SSH_KEY" -o StrictHostKeyChecking=yes "$FULL_PATH" \
           "${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_DIR}/"; then
        log "SUCCESS: Copied $NEWFILE to ${REMOTE_HOST}:${REMOTE_DIR}"
      else
        log "ERROR: Failed to copy $NEWFILE"
      fi
    else
      log "SKIPPED: $NEWFILE still being written, will catch on next close_write event"
    fi
  fi
done
```

**Why this design:**
- `inotifywait -m` runs continuously (monitor mode) rather than polling in a loop — much lower overhead.
- Watches `close_write` (file finished being written) and `moved_to` (covers atomic writes via temp-file-then-rename patterns), not `create`, to avoid copying half-written files.
- Adds a size-stability check as a second safety net.
- Logs every attempt with success/failure for auditability.
- Uses key-based auth (`-i`) and `StrictHostKeyChecking=yes` rather than disabling host key checking, which would be a security anti-pattern.
- Run as a **systemd service** in production for auto-restart on failure, rather than a plain background nohup process.

---

## Round 2

### 12. Playbook to Install Apache on a VM

```yaml
---
- name: Install and start Apache
  hosts: webservers
  become: true

  tasks:
    - name: Install Apache (Debian/Ubuntu)
      apt:
        name: apache2
        state: present
        update_cache: yes
      when: ansible_os_family == "Debian"

    - name: Install Apache (RedHat/CentOS)
      yum:
        name: httpd
        state: present
      when: ansible_os_family == "RedHat"

    - name: Start and enable Apache (Debian)
      service:
        name: apache2
        state: started
        enabled: true
      when: ansible_os_family == "Debian"

    - name: Start and enable Apache (RedHat)
      service:
        name: httpd
        state: started
        enabled: true
      when: ansible_os_family == "RedHat"

    - name: Open firewall for HTTP (RedHat, firewalld)
      firewalld:
        service: http
        permanent: true
        state: enabled
        immediate: true
      when: ansible_os_family == "RedHat"

    - name: Verify Apache is serving
      uri:
        url: "http://{{ inventory_hostname }}"
        status_code: 200
      delegate_to: localhost
      become: false
```

Same principles as the Nginx playbook: handle both major distro families (package name differs — `apache2` vs `httpd`, service name too), ensure `enabled: true` for boot persistence, and validate the end state instead of assuming success.

---

### 13. Updating the State File from Local to S3, and Recovering if It's Lost

**Update local → S3:** identical process to Q4 above — add the `backend "s3" {}` block, run `terraform init -migrate-state`, confirm with `terraform plan` (expect no diff), then remove the local state file from the repo/workstation.

**If the state is lost entirely (no backup, no versioning):**
1. **Don't panic-apply.** The infrastructure still exists in the cloud; only Terraform's *record* of it is gone. Running `apply` blind risks duplicate resource creation.
2. **Inventory actual resources** via the AWS console/CLI (`aws resourcegroupstaggingapi get-resources`, filtered by project tags if resources are tagged — this is exactly why consistent tagging matters).
3. **Rebuild state with `terraform import`** for every real resource, resource by resource, matching each to its Terraform address as defined in your `.tf` files:
   ```bash
   terraform import aws_vpc.main vpc-0123456789abcdef0
   terraform import aws_subnet.public subnet-0123456789abcdef0
   terraform import aws_instance.app i-0123456789abcdef0
   ```
4. **Verify with `terraform plan`** after each batch of imports — the goal is a plan with zero changes, confirming the imported state matches the actual configuration.
5. **For complex/many resources:** tools like `terraformer` can generate both `.tf` and state from existing cloud resources automatically, which is faster than manual import for large environments — though the generated HCL usually needs cleanup afterward.
6. **Afterward, immediately fix the root cause:** move to a remote backend with versioning and locking (S3 + DynamoDB) so this can't happen again — this scenario is the strongest argument for why local state should never be used for anything beyond a personal sandbox.

---

### 14. Terraform for AWS Services + Jenkinsfile for EKS + On-Prem Kubernetes Upgrade Steps

**Terraform for common AWS services (pattern, using modules):**
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "app-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway = true
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "app-cluster"
  cluster_version = "1.30"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  eks_managed_node_groups = {
    default = {
      min_size       = 2
      max_size       = 6
      desired_size   = 3
      instance_types = ["t3.large"]
    }
  }
}

resource "aws_db_instance" "app_db" {
  identifier        = "app-db"
  engine            = "postgres"
  engine_version    = "15.4"
  instance_class    = "db.t3.medium"
  allocated_storage = 20
  multi_az          = true
  db_subnet_group_name = aws_db_subnet_group.app.name
  vpc_security_group_ids = [aws_security_group.db_sg.id]
  skip_final_snapshot = false
}
```
Using well-maintained public modules (`terraform-aws-modules`) for common components like VPC/EKS is a standard practice — it avoids reinventing well-tested wiring and encodes AWS best practices by default.

**Jenkinsfile to deploy to EKS:**
```groovy
pipeline {
  agent any
  environment {
    AWS_REGION   = 'us-east-1'
    CLUSTER_NAME = 'app-cluster'
    ECR_REPO     = '123456789012.dkr.ecr.us-east-1.amazonaws.com/app'
  }
  stages {
    stage('Checkout') {
      steps { checkout scm }
    }
    stage('Build & Push Image') {
      steps {
        sh '''
          docker build -t $ECR_REPO:$BUILD_NUMBER .
          aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REPO
          docker push $ECR_REPO:$BUILD_NUMBER
        '''
      }
    }
    stage('Configure kubeconfig') {
      steps {
        sh 'aws eks update-kubeconfig --name $CLUSTER_NAME --region $AWS_REGION'
      }
    }
    stage('Deploy') {
      steps {
        sh '''
          kubectl set image deployment/app app=$ECR_REPO:$BUILD_NUMBER -n production
          kubectl rollout status deployment/app -n production --timeout=120s
        '''
      }
    }
  }
  post {
    failure {
      sh 'kubectl rollout undo deployment/app -n production'
    }
  }
}
```
Note the `post { failure { rollout undo } }` — an automatic rollback if the deploy stage's health check fails.

**On-prem Kubernetes (kubeadm-based) upgrade steps:**
1. On the **first control-plane node**: `apt-mark unhold kubeadm && apt-get install kubeadm=1.x.y-00` (repeat pattern per package manager), then:
   ```bash
   kubeadm upgrade plan
   kubeadm upgrade apply v1.x.y
   ```
2. Upgrade `kubelet` and `kubectl` on that node, restart kubelet.
3. Repeat on remaining control-plane nodes with `kubeadm upgrade node`.
4. For each **worker node**, one at a time: cordon → drain → upgrade kubeadm/kubelet package → `kubeadm upgrade node` → restart kubelet → uncordon.
5. Confirm cluster-wide version consistency with `kubectl get nodes` before/after each step.
6. Same zero-downtime mechanics as the managed-cluster case in Q9 — the manual package management is the main difference versus EKS/GKE, where the cloud provider automates the control-plane part.

---

### 15. Shell Script: Auto-SSH Copy `/nobackup` from ubuntu1 to Another VM

```bash
#!/bin/bash
# sync_nobackup.sh
# Copies /nobackup from this VM (ubuntu1) to a remote VM using key-based SSH auth.

set -euo pipefail

SRC_DIR="/nobackup"
REMOTE_USER="ubuntu"
REMOTE_HOST="192.0.2.20"
REMOTE_DIR="/nobackup"
SSH_KEY="/home/ubuntu/.ssh/id_rsa_backup"
LOG_FILE="/var/log/sync_nobackup.log"

log() {
  echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE"
}

# Sanity checks before doing anything destructive
if [ ! -d "$SRC_DIR" ]; then
  log "ERROR: Source directory $SRC_DIR does not exist. Aborting."
  exit 1
fi

if [ ! -f "$SSH_KEY" ]; then
  log "ERROR: SSH key $SSH_KEY not found. Aborting."
  exit 1
fi

log "Starting sync of $SRC_DIR to ${REMOTE_HOST}:${REMOTE_DIR}"

# rsync over ssh is preferred over plain scp for a directory sync:
# - resumable, only transfers changed data, preserves permissions/timestamps
rsync -avz --delete \
  -e "ssh -i ${SSH_KEY} -o StrictHostKeyChecking=yes -o BatchMode=yes" \
  "${SRC_DIR}/" "${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_DIR}/" \
  >> "$LOG_FILE" 2>&1

if [ $? -eq 0 ]; then
  log "SUCCESS: Sync completed"
else
  log "ERROR: Sync failed, check log above"
  exit 1
fi
```

**Setting up passwordless (auto) SSH beforehand, for context:**
```bash
# On ubuntu1
ssh-keygen -t ed25519 -f ~/.ssh/id_rsa_backup -N ""
ssh-copy-id -i ~/.ssh/id_rsa_backup.pub ubuntu@192.0.2.20
# Test:
ssh -i ~/.ssh/id_rsa_backup ubuntu@192.0.2.20 "echo connection ok"
```

**Why `rsync` over plain `scp` here:** for a whole-directory sync you generally want incremental transfer (only changed files), preserved permissions, and the `--delete` flag to mirror deletions — `scp -r` copies everything every time with no diffing. `BatchMode=yes` ensures the script fails fast instead of hanging on a password prompt if key auth isn't set up correctly. This would typically be scheduled via `cron` or a systemd timer for periodic sync.

---

### 16. Jenkins/Kubernetes: Pod Constantly Restarting — Troubleshooting Steps

1. **Check pod status and restart count:**
   ```bash
   kubectl get pods -n <namespace>
   kubectl describe pod <pod-name> -n <namespace>
   ```
   `describe` shows the **Events** section — often reveals the immediate cause (OOMKilled, failed probe, image pull error, etc.) without digging further.

2. **Check the exit code / reason via `describe` or:**
   ```bash
   kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
   ```
   - `OOMKilled` → container exceeded its memory limit.
   - `Error` → application crashed (check logs).
   - `CrashLoopBackOff` → Kubernetes' backoff wrapper around repeated failures, not itself the root cause.

3. **Check logs, including the previous crashed instance:**
   ```bash
   kubectl logs <pod-name> -n <namespace>
   kubectl logs <pod-name> -n <namespace> --previous
   ```

4. **Common root causes to check for, in likely order:**
   - **Resource limits too low** → OOMKilled. Check `kubectl top pod` and compare against `resources.limits.memory`.
   - **Failing liveness probe** → app is up but the probe endpoint/path/port is misconfigured or the app is too slow to respond, so Kubernetes kills and restarts it repeatedly. Check probe config vs actual app startup time; consider adding a `startupProbe` for slow-starting apps.
   - **Application-level crash** (unhandled exception, missing env var/config, failed DB connection at startup) → visible in logs directly.
   - **ConfigMap/Secret missing or misconfigured** → check `describe pod` events for mount failures.
   - **Image issue** → wrong tag, corrupted image, or the entrypoint command is wrong (check `Command`/`Args` in the pod spec vs the image's actual entrypoint).
   - **Node-level issue** → node under memory/disk pressure evicting pods; check `kubectl describe node <node>`.
   - **Dependency not ready** — app crashes because a downstream service (DB, cache) isn't reachable yet; check whether an init container or retry logic is needed.

5. **Check resource requests/limits and node capacity** — sometimes a pod keeps getting rescheduled and killed because no node has enough free resources.

6. **If it's a known transient issue**, temporarily increase `initialDelaySeconds`/`failureThreshold` on probes and resource limits to confirm the hypothesis before making a permanent fix.

7. **Fix and validate**, then watch `kubectl get pods -w` to confirm restart count stabilizes.

---

### 17. Deployment Timeout Issues — API Gateway Choice

**Symptoms:** requests failing with `504 Gateway Timeout` during or after deployment.

**Common causes:**
- Backend pods not ready yet when traffic is routed (readiness probe not configured, or rollout strategy doesn't wait for readiness).
- Gateway's own timeout setting is shorter than the backend's actual response time under load right after a deploy (cold start, JIT warmup, cache empty).
- Connection draining not configured, so in-flight requests are dropped when old pods are terminated mid-deploy.

**API gateways typically used, and why:**
- **AWS API Gateway** (for serverless/Lambda-backed or general REST APIs) — has a hard 29-second timeout ceiling, so if backend calls can run longer, it isn't the right choice without careful design (e.g., async patterns with polling).
- **NGINX Ingress Controller / NGINX Plus** (for Kubernetes-hosted APIs) — configurable `proxy-read-timeout`, `proxy-connect-timeout` annotations per-ingress, widely used because of fine-grained control.
- **Kong / Envoy-based gateways (Istio, Ambassador)** — used when you need advanced traffic management: retries with backoff, circuit breaking, canary/blue-green routing, and per-route timeout policies — very relevant for handling deploy-time timeouts gracefully.
- **AWS ALB** (simpler L7 load balancer, not a full API gateway) for straightforward HTTP routing with target-group health checks.

**Fixes applied alongside gateway config:**
- Set `readinessProbe` correctly so the Service/Ingress only routes to pods that are actually ready — this is usually the actual fix, not the gateway timeout value.
- Use **rolling update with `maxUnavailable: 0`** and a proper `minReadySeconds` so old pods aren't removed until new ones are confirmed healthy.
- Increase gateway-level timeout modestly as a safety margin, but treat it as a symptom fix, not the root fix, if the real issue is readiness/warmup.
- Add **connection draining / `preStop` hooks** with a short sleep so in-flight requests complete before a pod is terminated during rollout.

---

### 18. Managing Security at the Application Level

- **Authentication & Authorization:** OAuth2/OIDC (via Keycloak, Auth0, Cognito) for user auth; JWT with short expiry + refresh tokens; RBAC/ABAC enforced at the API layer, not just the UI.
- **Secrets management:** no hardcoded credentials in code or images — pulled at runtime from Vault/AWS Secrets Manager/Kubernetes Secrets (ideally backed by an external secrets operator, not raw base64 K8s secrets which are only encoded, not encrypted, by default).
- **Input validation & output encoding:** protect against injection (SQLi, XSS, command injection) — parameterized queries, strict schema validation on API inputs (e.g., JSON schema, class-validator).
- **Dependency scanning:** SCA tools (Snyk, Dependabot, Trivy) in the CI pipeline to catch vulnerable libraries before merge.
- **Container/image security:** minimal base images (distroless/alpine), image scanning (Trivy, Grype) in the pipeline, images signed (cosign) and only signed images allowed to deploy (admission controller policy).
- **Network-level controls:** Kubernetes NetworkPolicies to restrict pod-to-pod traffic to only what's needed (default-deny, then explicit allow), mTLS between services via a service mesh (Istio/Linkerd) for zero-trust internal communication.
- **TLS everywhere:** terminate TLS at the edge and also enforce it internally where compliance requires it; automate cert rotation (cert-manager).
- **Least privilege IAM:** service accounts/roles scoped tightly per workload (IRSA on EKS, Workload Identity on GKE) instead of broad shared credentials.
- **Audit logging:** all authentication events, admin actions, and API access logged and shipped to a SIEM for review/alerting.
- **Static/dynamic analysis in CI:** SAST (e.g., Semgrep, SonarQube) on every PR, DAST scans (OWASP ZAP) against staging before major releases.
- **Rate limiting / WAF:** protect public endpoints from abuse and common attack patterns (OWASP Top 10) at the gateway/CDN layer.

---

### 19. Securing a Public API for an On-Prem Setup

- **Perimeter placement:** put a reverse proxy / API gateway (NGINX, Kong, or a hardware/virtual WAF appliance) in a DMZ in front of the on-prem API — never expose the app server directly to the internet.
- **TLS termination at the edge** with strong cipher suites, HSTS enabled, and certificates from a trusted CA (or internal CA + cert pinning for known clients).
- **Web Application Firewall (WAF)** in front (e.g., ModSecurity with OWASP CRS, or a commercial WAF appliance) to filter common attack patterns before they reach the app.
- **Strong authentication for API consumers:** API keys or OAuth2 client-credentials flow for machine-to-machine, mutual TLS (mTLS) for high-trust partner integrations.
- **Rate limiting and quota enforcement** per API key/client at the gateway, to prevent abuse and protect on-prem capacity (which doesn't autoscale the way cloud does).
- **Network segmentation:** the API servers sit in an internal segment; only the reverse proxy/gateway has a route to the public internet; firewall rules restrict east-west traffic to only what's required.
- **DDoS mitigation:** since on-prem generally lacks cloud-scale elastic absorption, front the public entry point with a CDN/scrubbing service (Cloudflare, Akamai) even for on-prem origins, or ensure upstream ISP-level DDoS protection is in place.
- **Patch and harden the OS/API server** on a strict cadence — on-prem means you own the full patching responsibility (no managed-service auto-patching).
- **VPN or bastion-only access for management interfaces** — the admin/API-management plane should never be reachable from the same path as public API traffic.
- **Logging & monitoring at the edge** — WAF/gateway logs shipped to a SIEM, alerting on anomalous traffic patterns (sudden spikes, repeated auth failures indicating credential stuffing).

---

### 20. Where and How to Check Application Performance Metrics

**Where:**
- **Grafana dashboards** backed by Prometheus (or Datadog/New Relic/Dynatrace if using a commercial APM) — the standard single pane of glass for RED metrics (Rate, Errors, Duration) per service.
- **APM tools** (New Relic, Datadog APM, Dynatrace, Elastic APM) for deep code-level tracing — flame graphs showing exactly which function/DB query is slow.
- **Cloud-native dashboards:** AWS CloudWatch (ALB target response time, Lambda duration/errors, RDS performance insights), GCP Cloud Monitoring, Azure Monitor.
- **Kubernetes-specific:** `kubectl top pod/node`, Metrics Server, or Grafana dashboards fed by kube-state-metrics + cAdvisor for per-pod CPU/memory/network.
- **Real User Monitoring (RUM)** for frontend performance — actual browser-side load times, Core Web Vitals — via tools like Datadog RUM or Google Analytics/Lighthouse CI.
- **Synthetic monitoring** dashboards (Pingdom, Blackbox Exporter results in Grafana) for uptime/latency from external vantage points.

**How (what to actually look at):**
- **Latency percentiles** (P50/P95/P99), not just averages — averages hide tail latency that affects a meaningful chunk of users.
- **Error rate** as a percentage of total requests, broken down by endpoint/status code.
- **Throughput** (requests/sec) correlated against latency — to spot when the system starts degrading under load.
- **Resource saturation:** CPU/memory utilization vs. limits, DB connection pool usage, queue depth (for async/message-driven systems).
- **Database performance:** slow query logs, query execution plans, index usage — via tools like `pg_stat_statements` (Postgres) or RDS Performance Insights.
- **Distributed traces** for any P99 outlier request — to pinpoint exactly which downstream hop added the latency, rather than guessing.
- **Deployment correlation:** overlay deploy markers on dashboards (Grafana annotations) so a latency/error spike can be immediately correlated with "did this start right after a release?"
- **SLO/error-budget tracking:** define SLOs (e.g., "99.9% of requests < 300ms") and track error-budget burn rate over time rather than reacting only to hard outages.

---

*Tip for the interview: whenever you answer a "how would you do X" question, briefly narrate the trade-offs you considered (e.g., "I chose rsync over scp because…", "I used Ansible Vault over a full Vault server because the team was small…") — interviewers weight reasoning and trade-off awareness more heavily than reciting the "correct" tool name.*
