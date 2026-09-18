# Cisco DevOps Interview Answers

> **Experience:** 6+ years  
> **Focus:** SRE, monitoring, Ansible, Terraform, Jenkins, Kubernetes, scripting, API security, and performance.

## First Round

## 1. Improving reliability of a critical system

Use STAR and your real project data.

**Sample:** A customer-facing API suffered latency spikes and 5xx errors at peak load. I correlated load-balancer latency, application traces, Pod saturation, and database connection metrics. The application scaled horizontally, but each new Pod created an oversized connection pool, exhausting the database. I set a per-Pod connection budget, corrected HPA thresholds, spread replicas across zones, added Pod Disruption Budgets and startup/readiness probes, and introduced SLO burn-rate alerts, load tests, canary release, and automatic rollback. After validation, p99 latency and availability returned to target and the failure did not recur during later traffic peaks.

Explain impact, diagnosis, mitigation, root cause, permanent correction, and measurable result.

## 2. Proactive monitoring solutions

I implement:

- Golden signals: latency, traffic, errors, and saturation
- CPU, memory, disk, inode, network, and capacity monitoring
- Kubernetes node readiness, Pod restarts, pending Pods, unavailable replicas, HPA saturation, and scheduling failures
- Application p95/p99 latency, throughput, exceptions, GC, threads, and connection pools
- Queue depth, oldest-message age, database connections, locks, and dependency latency
- Synthetic transactions, certificate expiry, and endpoint availability
- OpenTelemetry traces and structured logs with correlation IDs
- SLOs, error budgets, and multi-window burn-rate alerts

Typical tools include Prometheus, Grafana, Alertmanager, CloudWatch, OpenTelemetry, Splunk/OpenSearch, AppDynamics, and PagerDuty. Alerts must be actionable, deduplicated, routed to an owner, and linked to a runbook.

## 3. Nginx Ansible playbook and secrets

```yaml
---
- name: Install and configure Nginx
  hosts: web
  become: true

  tasks:
    - name: Install Nginx
      ansible.builtin.package:
        name: nginx
        state: present

    - name: Deploy configuration
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        owner: root
        group: root
        mode: "0644"
        validate: "nginx -t -c %s"
      notify: Reload Nginx

    - name: Enable and start Nginx
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true

  handlers:
    - name: Reload Nginx
      ansible.builtin.service:
        name: nginx
        state: reloaded
```

Use Ansible Vault, Automation Controller credentials, or an external secret manager. Vault encrypts secrets at rest; use `no_log: true` where decrypted values could appear because Vault does not protect data while in use. citeturn9search141turn9search142

```bash
ansible-vault encrypt_string --vault-id prod@prompt 'SECRET' --name db_password
ansible-playbook site.yml --vault-id prod@prompt
```

## 4. Migrate Terraform local state to S3 with locking

Current Terraform supports S3 native lockfiles with `use_lockfile = true`. DynamoDB locking is deprecated, although it remains relevant to older installations and migrations. Enable S3 versioning for recovery. citeturn9search123

1. Stop concurrent Terraform runs.
2. Back up `terraform.tfstate`.
3. Create a versioned, encrypted, private S3 bucket separately.
4. Grant least-privilege access to the pipeline role.
5. Configure the backend.
6. Run `terraform init -migrate-state`.
7. Validate with `state pull`, `state list`, and `plan`.

```hcl
terraform {
  backend "s3" {
    bucket       = "company-prod-terraform-state"
    key          = "platform/prod/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    kms_key_id   = "alias/terraform-state"
    use_lockfile = true

    # Legacy only:
    # dynamodb_table = "terraform-state-lock"
  }
}
```

```bash
cp terraform.tfstate terraform.tfstate.backup
terraform init -migrate-state
terraform state pull > remote-state-check.json
terraform plan
```

Remote backends support collaboration and can provide locking; do not place credentials in the backend configuration. citeturn9search124turn9search128

## 5. Corrupted or lost Terraform state

State maps Terraform addresses to real resources. Corruption can make a plan propose duplicate creation, replacement, or deletion. Do not apply.

1. Freeze all Terraform runs.
2. Verify backend bucket, key, account, region, and workspace.
3. Preserve the damaged state.
4. Restore a known-good S3 object version.
5. Run `terraform init -reconfigure`, `terraform state pull`, and `terraform plan -refresh-only`.
6. Run a normal plan and review every action.
7. If no backup exists, reconstruct bindings using import blocks or `terraform import`.
8. Use `terraform state push` only as a last resort. It overwrites remote state and must be preceded by a backup and peer review. citeturn9search125

Terraform recommends secure remote state rather than version control because state needs locking, controlled access, and may contain sensitive values. citeturn9search126

## 6. Terraform EC2 with SSH-only inbound access

```hcl
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 6.0" }
  }
}

provider "aws" { region = "ap-south-1" }

variable "vpc_id" { type = string }
variable "subnet_id" { type = string }
variable "admin_cidr" {
  type = string
  validation {
    condition     = can(cidrhost(var.admin_cidr, 0)) && var.admin_cidr != "0.0.0.0/0"
    error_message = "Use a restricted administrator CIDR."
  }
}

data "aws_ami" "al2023" {
  most_recent = true
  owners      = ["amazon"]
  filter { name = "name", values = ["al2023-ami-2023.*-x86_64"] }
}

resource "aws_security_group" "ssh" {
  name   = "restricted-ssh"
  vpc_id = var.vpc_id

  ingress {
    protocol    = "tcp"
    from_port   = 22
    to_port     = 22
    cidr_blocks = [var.admin_cidr]
  }

  egress {
    protocol    = "-1"
    from_port   = 0
    to_port     = 0
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "server" {
  ami                    = data.aws_ami.al2023.id
  instance_type          = "t3.micro"
  subnet_id              = var.subnet_id
  vpc_security_group_ids = [aws_security_group.ssh.id]

  metadata_options { http_endpoint = "enabled", http_tokens = "required" }
  root_block_device { encrypted = true, volume_type = "gp3" }
  tags = { Name = "restricted-ssh-server" }
}
```

Prefer Systems Manager Session Manager and no inbound SSH for production.

## 7. GitHub multibranch Jenkins pipeline

1. Install and pin Pipeline, Git, and GitHub Branch Source plugins.
2. Store a GitHub App credential or scoped token in Jenkins Credentials.
3. Commit a reviewed `Jenkinsfile`.
4. Create a Multibranch Pipeline and add the GitHub repository as Branch Source.
5. Configure branch and pull-request discovery and retention.
6. Configure the GitHub webhook.
7. Scan the repository and verify generated branch and PR jobs.
8. Prevent fork PRs from accessing production credentials.

Jenkins automatically discovers branches containing a `Jenkinsfile` and exposes variables such as `BRANCH_NAME` and `CHANGE_ID`. citeturn9search135turn9search136

```groovy
pipeline {
  agent any
  stages {
    stage('Build') { steps { checkout scm; sh './mvnw clean verify' } }
    stage('Publish') {
      when { branch 'main' }
      steps { sh './ci/publish.sh' }
    }
    stage('PR Checks') {
      when { changeRequest() }
      steps { sh './ci/security-checks.sh' }
    }
  }
}
```

## 8. Dynamic Jenkins stages

Prefer Declarative `when` for known stages:

```groovy
pipeline {
  agent any
  parameters { choice(name: 'TARGET_ENV', choices: ['dev', 'stage', 'prod']) }
  stages {
    stage('Build') { steps { sh './build.sh' } }
    stage('Integration Test') {
      when { expression { params.TARGET_ENV in ['stage', 'prod'] } }
      steps { sh './integration-test.sh' }
    }
    stage('Production Approval') {
      when { expression { params.TARGET_ENV == 'prod' } }
      steps {
        timeout(time: 20, unit: 'MINUTES') {
          input message: 'Approve production?', submitter: 'prod-approvers'
        }
      }
    }
  }
}
```

For data-driven parallel stages, validate environment input against an allow-list before constructing stages inside `script`.

## 9. Kubernetes zero-downtime upgrade

Zero downtime is an application property, not a command guarantee.

### Before

- Review release notes, deprecations, version skew, and supported upgrade path.
- Verify CNI, CSI, CoreDNS, ingress, service mesh, operators, CRDs, and webhooks.
- Back up etcd for self-managed clusters and back up workload data.
- Test in non-production.
- Ensure multiple replicas, topology spreading, PDBs, correct probes, and spare capacity.
- Freeze unrelated changes and define rollback criteria.

### Execution

1. Upgrade the control plane.
2. Upgrade compatible add-ons.
3. Upgrade or blue-green replace workers gradually.
4. Cordon and drain one node or controlled batch.
5. Validate workloads before continuing.

```bash
kubectl cordon NODE
kubectl drain NODE --ignore-daemonsets --delete-emptydir-data   --grace-period=60 --timeout=15m
# Replace or upgrade node
kubectl uncordon NODE
```

The upstream order is control plane, nodes, clients, and manifest/API adjustments. citeturn9search133 Drain marks a node unschedulable and evicts workloads, while Pod Disruption Budgets help control voluntary disruption. citeturn9search131turn9search134

## 10. Post-upgrade verification

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.metadata.creationTimestamp
kubectl get apiservices
kubectl get --raw='/readyz?verbose'
kubectl top nodes
kubectl top pods -A
```

Verify node/control-plane versions, DNS, CNI, CSI, ingress, metrics, autoscaling, webhooks, CRDs, operators, PVC attach/mount, desired versus ready replicas, restart counts, NetworkPolicies, certificates, backups, monitoring, synthetic transactions, SLOs, and node rescheduling. Keep enhanced monitoring during an observation window.

## 11. Monitor a directory and SCP new files

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

WATCH_DIR="/nobackup"
REMOTE_USER="transfer"
REMOTE_HOST="10.20.30.40"
REMOTE_DIR="/nobackup/incoming"
SSH_KEY="/home/ubuntu/.ssh/transfer_key"
LOG_FILE="/var/log/file-transfer.log"

ssh_opts=(-i "$SSH_KEY" -o BatchMode=yes -o StrictHostKeyChecking=yes -o ConnectTimeout=10)
ssh "${ssh_opts[@]}" "$REMOTE_USER@$REMOTE_HOST" "mkdir -p '$REMOTE_DIR'"

inotifywait -m -e close_write -e moved_to --format '%w%f' "$WATCH_DIR" |
while IFS= read -r file; do
  [[ -f "$file" ]] || continue
  name=$(basename "$file")
  temp=".${name}.part.$$"
  if scp "${ssh_opts[@]}" -- "$file" "$REMOTE_USER@$REMOTE_HOST:$REMOTE_DIR/$temp" &&
     ssh "${ssh_opts[@]}" "$REMOTE_USER@$REMOTE_HOST"        "mv -- '$REMOTE_DIR/$temp' '$REMOTE_DIR/$name'"; then
    printf '%s uploaded %s
' "$(date -Is)" "$file" >> "$LOG_FILE"
  else
    printf '%s failed %s
' "$(date -Is)" "$file" >> "$LOG_FILE"
  fi
done
```

Pin the host key, protect the private key with `0600`, run under systemd, add retries/alerts, and use `rsync` when restartability/checksums are required.

## Second Round

## 12. Install Apache with Ansible

```yaml
---
- name: Install Apache
  hosts: web
  become: true
  vars:
    apache_name: "{{ 'apache2' if ansible_facts.os_family == 'Debian' else 'httpd' }}"
  tasks:
    - name: Install package
      ansible.builtin.package:
        name: "{{ apache_name }}"
        state: present

    - name: Enable and start service
      ansible.builtin.service:
        name: "{{ apache_name }}"
        state: started
        enabled: true

    - name: Verify Apache
      ansible.builtin.uri:
        url: http://127.0.0.1/
        status_code: 200
```

## 13. Local state to S3 and recovery if lost

Use the controlled migration in Question 4. Protect it with bucket versioning, KMS, public-access blocking, locking, audit logs, and least privilege. S3 backend documentation specifically recommends versioning for recovery from accidental deletion and human error. citeturn9search123

If lost, freeze operations, verify backend/workspace, restore the correct S3 object version, reinitialize, pull state, run refresh-only and normal plans, and import resources only when no backup exists.

## 14. Terraform scripts for AWS services

Use reusable modules and small environment root modules:

```text
infra/
├── modules/{vpc,eks,iam,database,observability}/
└── environments/{dev,stage,prod}/
```

```hcl
module "vpc" {
  source             = "../../modules/vpc"
  name               = "payments-prod"
  cidr               = "10.40.0.0/16"
  availability_zones = ["ap-south-1a", "ap-south-1b", "ap-south-1c"]
}

module "eks" {
  source             = "../../modules/eks"
  cluster_name       = "payments-prod"
  kubernetes_version = var.kubernetes_version
  vpc_id             = module.vpc.vpc_id
  private_subnet_ids = module.vpc.private_subnet_ids
}
```

CI runs formatting, validation, linting, security/policy scans, saved plan, approval, and protected apply using short-lived AWS credentials.

## 15. Jenkins pipeline on Kubernetes

Use ephemeral Kubernetes agents, least-privilege service accounts, immutable tool images, resource limits, and credentials scoped to deployment steps.

```groovy
pipeline {
  agent { label 'kubernetes-agent' }
  stages {
    stage('Validate') {
      steps { sh 'helm lint ./chart && kubectl apply --dry-run=server -f k8s/' }
    }
    stage('Deploy') {
      when { branch 'main' }
      steps {
        sh 'helm upgrade --install app ./chart --namespace app --atomic --timeout 10m'
      }
    }
  }
}
```

## 16. EKS and on-premises Kubernetes upgrades

### EKS

1. Review EKS version/add-on compatibility and deprecated APIs.
2. Test in non-production.
3. Upgrade control plane.
4. Upgrade VPC CNI, CoreDNS, kube-proxy, CSI, and other compatible add-ons.
5. Upgrade or replace managed node groups.
6. Drain old nodes gradually.
7. Update clients and run post-upgrade validation.

### On-premises/kubeadm

1. Verify HA control plane and take a tested etcd snapshot.
2. Back up manifests, certificates, configuration, and persistent data.
3. Upgrade one supported minor version at a time.
4. Upgrade control-plane nodes one by one using the exact version procedure.
5. Cordon, drain, upgrade, and uncordon each worker.
6. Upgrade CNI, CSI, DNS, ingress, and other add-ons compatibly.
7. Validate SLOs and recovery behavior.

Upstream guidance uses control plane, nodes, clients, then resource/API adjustments, with separate checks for third-party extensions. citeturn9search133

## 17. Copy `/nobackup` to another VM using an SSH key

`rsync` over SSH is better for directory synchronization because it resumes partial transfers and copies changes.

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
SOURCE_DIR="/nobackup/"
REMOTE_USER="ubuntu"
REMOTE_HOST="ubuntu2.example.internal"
REMOTE_DIR="/nobackup/"
PRIVATE_KEY="/home/ubuntu/.ssh/ubuntu2_key"

ssh_opts="-i $PRIVATE_KEY -o BatchMode=yes -o StrictHostKeyChecking=yes -o ConnectTimeout=10"
ssh $ssh_opts "$REMOTE_USER@$REMOTE_HOST" "mkdir -p '$REMOTE_DIR'"
rsync -aHAX --numeric-ids --partial --delay-updates   -e "ssh $ssh_opts" "$SOURCE_DIR" "$REMOTE_USER@$REMOTE_HOST:$REMOTE_DIR"
```

Test with `--dry-run`; verify free space, ownership, firewall, permissions, and host-key pinning.

## 18. Pod constantly restarting

```bash
kubectl get pod POD -n NS -o wide
kubectl describe pod POD -n NS
kubectl logs POD -n NS --all-containers --since=30m
kubectl logs POD -n NS --all-containers --previous
kubectl get events -n NS --sort-by=.metadata.creationTimestamp
kubectl top pod POD -n NS --containers
```

Check exit code, `OOMKilled`, bad entrypoint, missing Secret/ConfigMap, probe failures, resource limits, volume permissions, DNS/dependency failures, node pressure, image regression, init/sidecar failure, and security context. Compare the previous ReplicaSet and roll back a bad deployment. Cluster troubleshooting begins with expected nodes and `Ready` state, followed by detailed node information and events. citeturn9search130

## 19. Deployment timeout and API gateway

Clarify whether timeout is in Jenkins, Kubernetes rollout, load balancer, gateway, application, or dependency.

- REST API gateway: rich API management and transformations
- HTTP API gateway: simpler HTTP services
- WebSocket gateway: bidirectional WebSocket connections
- On-premises: Kong, NGINX, Envoy, HAProxy, Apigee, or another gateway based on requirements

Check DNS, TLS, authentication, gateway integration timeout, load-balancer idle timeout, Pod readiness, application pools, retries, database latency, and downstream timeouts. Do not blindly increase all timeouts. Use queue-based asynchronous processing for legitimately long operations.

```bash
kubectl rollout status deployment/APP -n NS --timeout=10m
kubectl describe deployment APP -n NS
kubectl get rs,pods -n NS
kubectl get events -n NS --sort-by=.metadata.creationTimestamp
```

## 20. Application-level security

- OIDC/OAuth or mTLS authentication where appropriate
- Server-side authorization and least privilege
- TLS and managed certificate rotation
- Input validation, output encoding, parameterized queries, and safe file handling
- Secret manager integration
- Secure cookies, headers, CSRF protection, and controlled CORS
- Rate limits, quotas, payload limits, WAF, and DDoS controls
- SAST, SCA, DAST, IaC/image scanning, SBOM, and artifact signing
- Restricted egress and network segmentation
- Safe audit logging and data redaction
- Threat modeling, penetration testing, patching, and incident response

## 21. Secure a public on-premises API

```text
Internet -> DDoS protection -> firewall -> WAF/API gateway in DMZ
         -> internal load balancer -> app network -> database network
```

Use TLS 1.2+, OIDC/OAuth or mTLS, scopes and tenant authorization, WAF, schema validation, rate limits, payload limits, deny-by-default network flows, VPN/ZTNA plus MFA for administration, redundant gateways, SIEM logging, certificate rotation, patching, and penetration testing. Gateway controls do not replace application-side authorization.

## 22. Where and how to check performance metrics

### Platforms

Prometheus/Grafana, CloudWatch, OpenTelemetry, AppDynamics, Dynatrace, Datadog, New Relic, Elastic APM, gateway/load-balancer dashboards, Kubernetes metrics, log analytics, database performance tools, and synthetic monitoring.

### Metrics

- Traffic, success rate, 4xx/5xx, p50/p95/p99 latency
- CPU, memory, GC, threads, file descriptors, event-loop delay
- Database query time, connections, locks, and pool saturation
- Cache hit ratio, queue depth, and oldest-message age
- Dependency latency/errors
- Pod restarts, throttling, OOM, node pressure, and desired/ready replicas
- Business transaction completion and SLO burn rate

Start with the affected transaction, correlate gateway/service/database telemetry through trace IDs, inspect slow spans, correlate logs and saturation, compare recent changes, and validate recovery through canary and synthetic tests. Avoid high-cardinality metric labels such as raw user IDs.

## Quick Revision

- Use STAR for reliability incidents.
- Monitor customer outcomes and golden signals, then infrastructure for diagnosis.
- Vault protects Ansible secrets at rest; prevent output disclosure too.
- Prefer S3 lockfiles for new Terraform backends because DynamoDB locking is deprecated.
- Stop all applies before state recovery.
- Multibranch Jenkins discovers branches with a `Jenkinsfile`.
- Zero-downtime upgrades require redundant workloads, PDBs, probes, capacity, and controlled drains.
- Use `rsync` for durable directory synchronization.
- Investigate CrashLoopBackOff through previous logs, exit reason, probes, resources, configuration, and dependencies.
- Diagnose each timeout layer rather than only increasing timeouts.
- Secure public APIs across edge, identity, network, application, data, and observability layers.
