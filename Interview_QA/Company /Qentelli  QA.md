# DevOps, Cloud, and SRE Interview Questions

## Qentelli Solutions | Experience: 5 Years

This guide includes explanatory answers, Terraform examples, Linux commands, operational checklists, and production considerations.

> **Important:** Examples are illustrative, not executed against a live environment. Review permissions, costs, retention requirements, and change approvals before using them.

---

## 1. Create an S3 bucket using Terraform

### Explanation

Define the bucket and its security settings as infrastructure as code.

For this example, the intended configuration is:

- Private access.
- Public access blocked.
- Versioning enabled.
- Explicit server-side encryption.
- Protection against accidental destruction while the resource remains configured.

The AWS provider exposes separate resources for public-access settings, encryption, and versioning.

### Example: main.tf

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

variable "aws_region" {
  type    = string
  default = "ap-south-1"
}

variable "bucket_name" {
  description = "Choose an available S3 bucket name."
  type        = string
}

resource "aws_s3_bucket" "logs" {
  bucket        = var.bucket_name
  force_destroy = false

  tags = {
    Name        = var.bucket_name
    Environment = "dev"
    ManagedBy   = "Terraform"
  }

  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_s3_bucket_public_access_block" "logs" {
  bucket = aws_s3_bucket.logs.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_versioning" "logs" {
  bucket = aws_s3_bucket.logs.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "logs" {
  bucket = aws_s3_bucket.logs.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

output "bucket_arn" {
  value = aws_s3_bucket.logs.arn
}
```

### Example: terraform.tfvars

```hcl
aws_region  = "ap-south-1"
bucket_name = "replace-with-an-available-bucket-name"
```

### Suggested execution workflow

Authenticate using an approved AWS identity, then:

```bash
terraform fmt
terraform init
terraform validate
terraform plan -out=tfplan

# Run only after reviewing and approving the saved plan.
terraform apply tfplan
```

### Production considerations

- Do not hardcode AWS access keys in Terraform.
- Use approved temporary credentials or workload identity.
- Store state securely and restrict access.
- Review version-retention costs.
- Add lifecycle policies only after confirming retention requirements.
- Use SSE-KMS when customer-controlled key policies and key auditing are required.

> **Interview trap:** `prevent_destroy` does not protect a resource after its entire configuration block is removed.

### Interview-ready answer

> “I create the bucket with separate resources for public-access blocking, versioning, and encryption. I review the Terraform plan before applying it and use secure credentials and remote state.”

**References:** [Public access block resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket_public_access_block.html), [Encryption resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket_server_side_encryption_configuration), [Versioning resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket_versioning).

---

## 2. If a developer changes a private subnet to public, what should you do?

### First clarify what actually changed

In AWS, a subnet is public when its associated route table has a route to an Internet Gateway.

Changing a subnet name or tag does not change its network behavior.

For direct IPv4 internet communication, an EC2 instance also needs a public IPv4 address, along with appropriate routing and traffic permissions.

### Suggested response workflow

#### Step 1: Establish the change and affected scope

Inspect:

- Route table entries.
- Route table associations.
- Public-IP assignment settings.
- Existing public IP addresses.
- Security groups.
- Network ACLs.
- IPv6 routes and addresses.
- Workloads sharing the affected route table.

A shared route table can make the impact larger than one subnet.

#### Step 2: Determine actual exposure

Ask:

- Which workloads have publicly routable addresses?
- Which ports are permitted?
- Are databases or administrative interfaces reachable?
- Is there evidence of unexpected access?

A public route does not automatically mean every instance is reachable.

#### Step 3: Restore the approved configuration

My recommended approach is to restore the intended route or route-table association through the approved incident/change process.

Avoid blindly removing routes if that would interrupt legitimate workloads.

If an emergency manual correction is necessary, reconcile Terraform afterward.

#### Step 4: Investigate and prevent recurrence

Recommended controls:

- Review audit records for the change.
- Review available network and application logs.
- Restrict production network changes to approved deployment roles.
- Require reviewed infrastructure plans.
- Add policy checks for unintended Internet Gateway routes.
- Monitor configuration drift.

Treat this as a configuration incident, not an assumption about an individual's competence or intent.

### Interview-ready answer

> “I verify the route-table change and actual exposure, restore the approved configuration without unnecessary disruption, review audit evidence, and add guardrails to prevent recurrence.”

**Reference:** [Internet Gateway and public subnet behavior](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html).

---

## 3. What is AWS KMS?

### Explanation

AWS Key Management Service is a managed service for creating and controlling cryptographic keys.

It integrates with AWS services and applications to protect data.

It supports separation between:

- Who can administer a key.
- Who can use a key for cryptographic operations.

KMS usage can be audited through CloudTrail.

### Envelope encryption

AWS services commonly use envelope encryption:

1. A data key encrypts the actual data.
2. A KMS key protects the data key.
3. The encrypted data key is stored with the encrypted data.
4. Authorized decryption recovers the data key so the data can be decrypted.

The KMS key is not used to directly encrypt every byte of a large object.

### Key-management choices

| Choice | Typical reason to use it |
|---|---|
| AWS-managed key | Service-managed key lifecycle with less administrative control. |
| Customer-managed key | Explicit control over policies, lifecycle, and administration. |

### Example: Customer-managed key

```hcl
resource "aws_kms_key" "application" {
  description         = "Application encryption key"
  enable_key_rotation = true

  tags = {
    Environment = "prod"
    ManagedBy   = "Terraform"
  }

  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_kms_alias" "application" {
  name          = "alias/application-data"
  target_key_id = aws_kms_key.application.key_id
}
```

This is an illustrative starting point. Production key-policy design must explicitly account for administrators, application roles, and integrated services.

### Production considerations

My recommendations:

- Separate key administrators from application users.
- Grant only required cryptographic permissions.
- Review key policies and IAM policies together.
- Audit key usage.
- Treat disabling or deleting keys as a high-impact change.
- Never assume key rotation automatically re-encrypts existing application data.

### Interview-ready answer

> “KMS centrally manages encryption keys and access to cryptographic operations. Integrated services use envelope encryption, and I separate key administration from key usage while auditing access.”

**Reference:** [AWS KMS overview and envelope encryption](https://aws.amazon.com/documentation-overview/kms/).

---

## 4. Explain ELB inflow and outflow

### Interpretation

“Inflow and outflow” usually refers to:

- Client traffic entering the load balancer.
- Load-balancer traffic reaching targets.
- Responses returning to clients.
- Security-group rules for these connections.

ELB is a service family. The following example uses an Application Load Balancer.

### Example architecture

A suggested web-application design is:

- Internet-facing ALB as the public entry point.
- HTTPS listener on port `443`.
- Application targets in private subnets.
- Target group forwarding to application port `8080`.
- Health checks on the configured application health endpoint.

### Request flow

1. The client connects to the ALB listener.
2. Listener rules select a target group.
3. The ALB forwards the request to a target.
4. The target sends its response to the ALB.
5. The ALB returns the response to the client.

ALB is a proxy, so client-to-ALB and ALB-to-target are separate connections.

### Example security-group design

| Security group | Direction | Rule |
|---|---|---|
| ALB SG | Inbound | Allow TCP 443 from approved clients, or the internet for an intentionally public application. |
| ALB SG | Outbound | Allow application and health-check traffic to the target SG. |
| Target SG | Inbound | Allow application and health-check ports from the ALB SG. |

Security groups are stateful, so response traffic for allowed connections is automatically permitted.

### Health-check consideration

If health checks use a different port from application traffic, allow both ports.

### Troubleshooting checklist

My suggested checks:

- DNS resolution.
- Listener configuration.
- Listener rules.
- Target-group registration.
- Target health.
- ALB and target security groups.
- Network ACLs.
- Application listening port.
- Access logs and error metrics.

> **Interview trap:** Do not apply every ALB behavior to an NLB. Protocol handling and client-IP behavior differ by load-balancer type and configuration.

### Interview-ready answer

> “For ALB, I validate two paths: client to listener, and ALB to target. I allow target traffic only from the ALB security group and include health-check ports.”

**References:** [Application Load Balancers](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/application-load-balancers.html), [ALB security groups](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-update-security-groups.html).

---

## 5. How do you manage resources in Terraform?

### Explanation

Terraform configuration describes desired infrastructure.

Terraform state maps resource addresses in configuration to real infrastructure objects.

A plan compares configuration, state, and refreshed infrastructure information to propose changes.

### Recommended management approach

#### 1. Reusable modules

Use modules for repeated infrastructure patterns.

Example:

```hcl
module "application_network" {
  source = "../../modules/network"

  environment = "prod"
  vpc_cidr    = "10.20.0.0/16"
}
```

The referenced module must declare these inputs.

#### 2. Environment isolation

My recommendation is to separate production and non-production:

- State.
- Credentials.
- Root configurations.
- Approval controls.

#### 3. Secure state

State can contain sensitive information.

Use a secure remote backend with appropriate access controls and locking support.

Do not commit state files to Git.

#### 4. Reviewed changes

Suggested workflow:

```bash
terraform fmt
terraform init
terraform validate
terraform plan -out=tfplan

# Apply the reviewed plan through the approved workflow.
terraform apply tfplan
```

#### 5. Existing resources

Import existing resources deliberately rather than attempting to create duplicates.

Example:

```hcl
import {
  to = aws_s3_bucket.existing
  id = "existing-bucket-name"
}

resource "aws_s3_bucket" "existing" {
  bucket = "existing-bucket-name"
}
```

Review the plan carefully because the resource configuration must match the intended existing settings.

#### 6. Drift management

My recommended practice:

- Run scheduled plans.
- Investigate unexpected changes.
- Reconcile emergency manual changes.
- Avoid direct state-file editing.

#### 7. Lifecycle controls

Use lifecycle settings only for a specific requirement.

For example:

```hcl
lifecycle {
  prevent_destroy = true
}
```

Do not broadly use `ignore_changes` to hide unmanaged drift.

### Interview-ready answer

> “I manage resources through reusable modules, isolated state, secure credentials, reviewed plans, and controlled applies. I import existing resources deliberately and investigate drift rather than editing state manually.”

**Reference:** [Terraform state](https://developer.hashicorp.com/terraform/language/state).

---

## 6. Write a shell script to back up logs from the last seven days and remove older logs

### Define the requirement precisely

For this example:

- “Last seven days” means file modification time newer than a rolling seven-day cutoff.
- Only rotated, immutable logs are processed.
- Active application logs are not processed.
- Recent logs are archived.
- Older logs are also archived before their source files are removed.
- Dry-run is the default.
- Deletion happens only after archive creation and readability checks succeed.

> File modification time does not identify the dates of individual log records. A file modified today can contain older records.

### Prerequisites

This example assumes Linux with:

- Bash.
- GNU `find`.
- GNU `tar`.
- GNU `date`.
- `flock`.

Use a dedicated, trusted directory containing only rotated logs.

### Script: backup_logs.sh

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
umask 077

# Dedicated directory containing immutable, rotated logs only.
LOG_DIR="${LOG_DIR:-/var/log/myapp/rotated}"
BACKUP_DIR="${BACKUP_DIR:-/var/backups/myapp}"
MODE="${1:---dry-run}"

case "$MODE" in
  --dry-run|--execute) ;;
  *)
    echo*"*sage: $0 [--dry-run|--execute]" >&*
    exit 2
    ;;
esac*
[[ -d "$LOG_DIR" ]] || {
 *echo*"Log directory does not exist: $LO*_DIR" >&2
  exit 1
}

mkdir -* -- "$BACKUP_DIR"

LOG_DIR*"$(realpath -* -- "$LOG_DIR")"
BACKUP*DIR="$(realpath -e -- "$BACKUP_DIR*)"

# Prevent*arch*ving backups recursively*
case*"$*ACKUP_DIR/" in
  "$LOG_DIR/"*)
    echo "Backup directory must be outside the log directory." >&2
    exit 1
    ;;
esac

# Prevent overlapping executions of this script.
exec 9>"$BACKUP_DIR/.backup.lock"
flock -n 9 || {
  echo "Another backup is running." >&2
  exit 1
}

WORK="$(mktemp -d "$BACKUP_DIR/.work.XXXXXX")"
trap 'rm -rf -- "$WORK"' EXIT

CUTOFF="$(date -u -d '7 days ago' '+%Y-%m-%d %H:%M:%S UTC')"
STAMP="$(date -u '+%Y%m%dT%H%M%SZ')-$$"

# Do not follow symlinks. Match regular rotated-log files only.
(
  cd -- "$LOG_DIR"

  find . -type f \
    \( -name '*.log' -o -name '*.log.*' \) \
*  *-newermt "$CUTOFF" \
*  *-print0 > "$WORK/recent.list"

  f*nd . -type f \
    \( -name '*.log' -o -name '*.log.*' \) \
    ! -newermt "$CUTOFF" \
    -print0 > "$WORK/older.list"
)

echo "Recent logs selected:"
while IFS= read -r -d '' file; do
  printf '  %q\n' "$file"
done < "$WORK/recent.list"

echo "Older logs selected for archive-before-removal:"
while IFS= read -r -d '' file; do
  printf '  %q\n' "$file"
done < "$WORK/older.list"

if [[ "$MODE" == "--dry-run" ]]; then
  echo "Dry-run complete. No logs were archived or removed."
  exit 0
fi

archive_list() {
  local list="$1"
  local label="$2"
  local final="$BACKUP_DIR/${label}-${STAMP}.tar.gz"
  local partial="$WORK/${label}.tar.gz"

  [[ -s "$list" ]] || return 0

  tar -C "$LOG_DIR" \
    --create --gzip \
    --file="$partial" \
    --null --verbatim-files-from \
    --files-from="$list"

  # Check that the archive can be read.
  tar --list --gzip --file="$partial" > /dev/null

  mv -- "$partial" "$final"
  echo "Archive created: $final"
}

archive_list "$WORK/recent.list" "recent"
archive_list "$WORK/older.list" "older"

# Older logs have been archived successfully before this point.
while IFS= read -r -d '' file; do
  rm -- "$LOG_DIR/$file"
done < "$WORK/older.list"

echo "Backup complete. Selected older source logs were removed."
```

### Usage

```bash
chmod 750 backup_logs.sh

# Inspect the selection first.
./backup_logs.sh --dry-run

# Execute only after validating paths and retention requirements.
./backup_logs.sh --execute
```

### Why this approach?

- Quoted paths avoid common shell expansion problems.
- NUL-separated filenames support spaces and newlines.
- Dry-run makes the selection visible.
- Older logs are archived before removal.
- Failed archive creation stops deletion.
- The backup directory is outside the source directory.

### Limitations and production recommendations

- Archive listing is not a full restore test.
- The script lock does not stop another process from modifying logs.
- Run against immutable rotated logs or a consistent snapshot.
- Do not allow untrusted users to modify the source or backup directories.
- Backups require a separate retention policy; this script does not delete backup archives.
- Test restoration and consider off-host backup storage.
- Use the application's supported rotation mechanism for active logs.

> **Interview trap:** `find -mtime +7` uses rounded age intervals and does not mean an exact rolling seven-day cutoff. This example uses an explicit timestamp comparison.

**References:** [GNU Findutils manual](https://www.gnu.org/software/findutils/manual/find.html), [GNU tar NUL-separated filenames](https://www.gnu.org/software/tar/manual/html_node/nul.html).

---

## 7. What is the difference between Security Groups and NACLs?

### Comparison

| Feature | Security Group | Network ACL |
|---|---|---|
| Scope | Associated resource/network interface. | Subnet boundary. |
| State | Stateful. | Stateless. |
| Rules | Allow rules. | Allow and deny rules. |
| Evaluation | Applicable allow rules are combined. | Rules evaluated in ascending rule-number order. |
| Return traffic | Automatically permitted for allowed connections. | Must be permitted explicitly. |
| Typical purpose | Fine-grained workload access. | Additional subnet-level traffic control. |

### Example: HTTPS server

For a security group:

- Allow inbound TCP `443` from approved clients.
- Response traffic for that allowed connection is automatically permitted.

For a NACL:

- Allow inbound TCP `443`.
- Allow outbound traffic to the client's applicable ephemeral port range.

### Example: Application-to-database access

My recommended security-group pattern:

- Application SG allows required outbound database traffic.
- Database SG allows its database port from the application SG.
- Do not expose the database port broadly to the internet.

### Default behavior

- The default NACL permits inbound and outbound traffic.
- A newly created custom NACL initially blocks inbound and outbound traffic until rules are added.

### Interview-ready answer

> “Security groups are stateful resource-level allow controls. NACLs are stateless subnet-level controls with allow and deny rules. With NACLs, I explicitly account for return traffic and rule ordering.”

**References:** [Security groups](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html), [Network ACLs](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html).

---

## 8. How do you secure environments in AWS?

### Suggested defense-in-depth approach

This is an operational design recommendation, not a single mandatory architecture.

### A. Account and environment isolation

- Separate production and non-production accounts.
- Restrict cross-account access.
- Centralize governance and audit visibility.
- Keep emergency access controlled and auditable.

### B. Identity

- Prefer federation and temporary credentials.
- Require MFA for human access.
- Apply least privilege.
- Use workload roles instead of embedded access keys.
- Review unused permissions and credentials.

### C. Network

- Keep application and database workloads private where appropriate.
- Expose only approved entry points.
- Restrict security groups.
- Review IPv4 and IPv6 paths.
- Control outbound connectivity.

### D. Data

- Encrypt sensitive data at rest and in transit.
- Restrict access to encryption keys.
- Block unintended public storage access.
- Manage secrets outside source code.
- Validate backup and restore procedures.

### E. Compute and delivery

- Patch operating systems and dependencies.
- Scan deployment artifacts.
- Review infrastructure plans.
- Restrict production deployment identities.
- Prefer repeatable, immutable deployment patterns.

### F. Detection and response

- Collect audit and operational logs.
- Monitor configuration changes.
- Investigate security findings.
- Maintain tested incident-response procedures.

### Example architecture description

For a typical web application, I would propose:

- A controlled public load-balancer entry point.
- Private application workloads.
- A private database.
- Restricted workload-to-workload security groups.
- Role-based service access.
- Centralized logs and security monitoring.

### Interview-ready answer

> “I secure AWS through account isolation, least-privilege identity, controlled network exposure, encryption, secure delivery, and continuous detection. I also test incident response and recovery.”

**Reference:** [AWS Well-Architected Security Pillar](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/welcome.html).

---

## 9. What is the difference between a soft link and a hard link?

### Explanation

A hard link is another name for the same file inode.

A symbolic link, also called a soft link, is a separate file containing a path to another file or directory.

### Comparison

| Feature | Hard link | Symbolic link |
|---|---|---|
| Refers to | Same underlying inode. | Target pathname. |
| Inode | Shared with the linked file. | Separate inode. |
| Cross-filesystem use | Not supported on normal Linux filesystems. | Supported. |
| Directory links | Normally prohibited. | Supported. |
| Target pathname removed | Other hard-link names remain usable. | Link can become dangling. |
| Creation | `ln target link` | `ln -s target link` |

### Example

```bash
printf 'application configuration\n' > original.txt

ln original.txt hard-link.txt
ln -s original.txt soft-link.txt

ls -li original.txt hard-link.txt soft-link.txt
```

### Expected behavior

- `original.txt` and `hard-link.txt` share an inode.
- `soft-link.txt` has its own inode and refers to `original.txt`.
- Editing through either hard-link name changes the same underlying file.
- Removing one hard-link name does not remove the other name.
- A symbolic link becomes dangling if its target path no longer resolves.

### DevOps use cases

Suggested examples:

- Symbolic link: a `current` deployment path pointing to a release directory.
- Hard link: multiple names for the same file on one filesystem.

> **Interview trap:** A hard link is not an independent backup. Both names refer to the same underlying data.

### Interview-ready answer

> “A hard link shares the same inode and cannot normally cross filesystems. A soft link stores a pathname, can cross filesystems, and can become dangling when the target path disappears.”

**Reference:** [GNU ln documentation](https://www.gnu.org/software/coreutils/manual/html_node/ln-invocation.html).

---

## 10. A user cannot SSH to an instance. What troubleshooting steps do you perform?

### Start with the exact error

| Error | Initial investigation |
|---|---|
| Connection timed out | Network path, routes, security groups, NACLs, instance health. |
| Connection refused | SSH listener, service state, host firewall, configured port. |
| Permission denied (publickey) | Username, key, authorized keys, ownership and permissions. |
| Host key verification failed | Verify host identity and whether the host key legitimately changed. |

These are investigation starting points, not definitive diagnoses.

### Suggested troubleshooting sequence

#### Step 1: Check instance health

- Is the instance running?
- Are status checks passing?
- Is the address current?
- Are you connecting to the intended instance?

#### Step 2: Verify the access path

For direct public IPv4 access:

- Public IPv4 address.
- Route through an Internet Gateway.
- Security group allowing the approved source on the SSH port.
- NACL rules allowing request and return traffic.

For a private instance:

- Use an approved private connectivity path.
- Alternatively, use an appropriately configured administrative access service.

#### Step 3: Check username and key

Common default usernames include:

- Amazon Linux: `ec2-user`.
- Ubuntu: `ubuntu`.

Confirm the username for the actual AMI.

```bash
chmod 400 my-key.pem

ssh -vvv \
  -i my-key.pem \
  ec2-user@INSTANCE_ADDRESS
```

Verbose output helps identify the stage at which the connection fails.

#### Step 4: Inspect the operating system through an approved alternate path

If available, use Session Manager or an approved recovery method.

Illustrative diagnostic commands:

```bash
sudo ss -lntp

# Service names vary by distribution.
sudo systemctl status sshd
sudo systemctl status ssh

sudo journalctl -u sshd --since "30 minutes ago"
sudo journalctl -u ssh --since "30 minutes ago"

df -h
df -i

sudo sshd -t
```

My suggested checks:

- SSH daemon listening on the expected port.
- Valid SSH configuration.
- Correct authorized-key ownership and permissions.
- Host firewall rules.
- Disk and inode availability.
- Authentication logs.

### Safe remediation

- Correct only the identified problem.
- Do not open SSH to `0.0.0.0/0` as a default troubleshooting step.
- Do not disable host-key checking to bypass an unexplained identity change.
- Validate SSH configuration before restarting the service.
- Keep an existing administrative session open during changes when possible.

### Interview-ready answer

> “I classify the error first, then check instance health, network reachability, username and key, and finally the SSH service and OS logs. I avoid broad firewall changes and use approved alternate access for recovery.”

**Reference:** [Troubleshoot EC2 Linux connection issues](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/TroubleshootingInstancesConnecting.html).

---

## 11. Which CloudWatch metrics should you focus on?

### Start with application health

My recommendation is to prioritize:

- Request success.
- Latency.
- Traffic.
- Capacity saturation.

Infrastructure metrics explain problems, but CPU alone does not prove the application is healthy.

### A. EC2 metrics

| Metric | What it helps investigate |
|---|---|
| `CPUUtilization` | CPU demand. |
| `NetworkIn` / `NetworkOut` | Network traffic volume. |
| `StatusCheckFailed` | Instance status-check failures. |
| `StatusCheckFailed_Instance` | Instance-level status failure. |
| `StatusCheckFailed_System` | Underlying system status failure. |
| `CPUCreditBalance` | Available credits on applicable burstable instances. |
| `CPUSurplusCreditsCharged` | Additional charged credits on applicable unlimited instances. |

### B. Operating-system metrics

Memory and filesystem usage are not standard EC2 metrics.

Use the CloudWatch agent or another approved custom-metric publisher.

Important examples:

- `mem_used_percent`.
- `disk_used_percent`.
- Filesystem inode availability.
- Swap usage where relevant.

### Example agent configuration

```json
{
  "metrics": {
    "namespace": "CWAgent",
    "append_dimensions": {
      "InstanceId": "${aws:InstanceId}"
    },
    "metrics_collected": {
      "mem": {
        "measurement": [
          "mem_used_percent"
        ]
      },
      "disk": {
        "measurement": [
          "used_percent"
        ],
        "resources": [
          "/",
          "/var/log"
        ]
      }
    }
  }
}
```

Adapt the filesystem paths to the host.

### C. Application Load Balancer metrics

| Metric | What it helps investigate |
|---|---|
| `RequestCount` | Requests for which a target was selected. |
| `TargetResponseTime` | Time until the target starts sending response headers. |
| `HTTPCode_ELB_5XX_Count` | Errors generated by the load balancer. |
| `HTTPCode_Target_5XX_Count` | Errors generated by targets. |
| `HealthyHostCount` | Healthy target count. |
| `UnHealthyHostCount` | Unhealthy target count. |
| `TargetConnectionErrorCount` | Failed ALB-to-target connections. |

### Alarm-design recommendations

- Use latency percentiles where appropriate.
- Use error rates, not only raw error counts.
- Define thresholds from workload baselines and SLOs.
- Configure missing-data behavior deliberately.
- Attach actionable runbooks.
- Avoid universal thresholds for every application.

> **Interview trap:** EC2 `DiskReadOps` and `DiskWriteOps` describe instance-store activity. Review EBS-specific metrics for EBS volumes.

### Interview-ready answer

> “I monitor application success and latency first, then CPU, memory, filesystem usage, network traffic, and status checks. For ALB, I distinguish load-balancer errors from target errors and monitor target health.”

**References:** [EC2 metrics](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/viewing_metrics_with_cloudwatch.html), [ALB metrics](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-cloudwatch-metrics.html), [CloudWatch agent](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Install-CloudWatch-Agent.html).

---

## 12. Your AWS bill spikes. What should you check?

### First distinguish actual cost from forecast

Check whether the increase is:

- Actual billed or accrued usage.
- A forecast.
- A different cost view.
- A change in credits or discounts.

Cost Explorer is not a real-time operational telemetry source.

### Suggested investigation workflow

#### Step 1: Identify the cost driver

Use Cost Explorer to compare the affected period with an appropriate baseline.

Investigate by:

- Service.
- Account.
- Region.
- Usage type.
- Available cost-allocation tags.
- Resource-level data where available.

#### Step 2: Investigate the dominant service

| Area | Suggested checks |
|---|---|
| EC2 | Instance count, instance size, runtime, scaling changes, CPU-credit charges. |
| EBS | Unattached volumes, provisioned capacity, snapshots. |
| S3 | Stored data, versions, requests, retrieval, transfer. |
| Networking | NAT usage, data transfer, public IPv4 resources. |
| Load balancing | Load-balancer usage and capacity consumption. |
| CloudWatch | Log ingestion, retention, custom metrics, query activity. |
| Managed services | Capacity changes, new deployments, increased request volume. |

This is a troubleshooting checklist, not a claim that each item caused the spike.

#### Step 3: Correlate with operational changes

My recommended checks:

- Deployment history.
- Infrastructure changes.
- Scaling events.
- Traffic increases.
- Logging changes.
- New resources in unexpected regions.
- Security findings or suspicious activity.

Do not assume compromise merely because costs increased.

#### Step 4: Contain safely

- Stop runaway automation through the approved process.
- Right-size only after checking workload requirements.
- Remove unused resources only after confirming ownership and dependencies.
- Preserve evidence if suspicious activity is identified.

#### Step 5: Prevent recurrence

Recommended controls:

- Cost Anomaly Detection.
- Budget notifications.
- Cost-allocation tags.
- Resource ownership.
- Approved scaling limits.
- Regular unused-resource reviews.

### Important caveat

A budget notification is not inherently a hard spending cap. Automated budget actions require explicit configuration and review.

### Interview-ready answer

> “I identify the service, account, region, and usage type driving the increase, then correlate it with resource and deployment changes. I contain the cause safely and add cost alerts and ownership controls.”

**References:** [AWS Cost Explorer](https://docs.aws.amazon.com/cost-management/latest/userguide/ce-what-is.html), [Unexpected charges](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/checklistforunwantedcharges.html), [Cost Anomaly Detection](https://aws.amazon.com/aws-cost-management/aws-cost-anomaly-detection/).

---

## 13. What security options are available in AWS?

### How this differs from Question 8

Question 8 asks for an overall security strategy.

This question asks which controls or services you would select for specific requirements.

### Suggested control-selection checklist

| Requirement | Options to evaluate |
|---|---|
| Identity and permissions | IAM roles and policies, federation, MFA, IAM Identity Center. |
| Organization guardrails | AWS Organizations policies and account isolation. |
| Resource-level network access | Security groups. |
| Subnet-level traffic control | Network ACLs. |
| Web request filtering | AWS WAF. |
| DDoS protection | AWS Shield options. |
| Managed network inspection | AWS Network Firewall. |
| Encryption-key management | AWS KMS. |
| TLS certificates | AWS Certificate Manager. |
| Secret storage | AWS Secrets Manager or appropriately configured Parameter Store. |
| API activity auditing | AWS CloudTrail. |
| Configuration tracking | AWS Config. |
| Threat detection | Amazon GuardDuty. |
| Security posture and findings | AWS Security Hub. |
| Vulnerability assessment | Amazon Inspector. |
| Sensitive-data discovery in S3 | Amazon Macie. |
| Network visibility | VPC Flow Logs. |
| Recovery | AWS Backup and tested restore procedures. |

This table is a selection aid. Validate service scope, supported resources, regional availability, pricing, and configuration before adoption.

### Important distinctions

- IAM controls permissions; encryption does not replace authorization.
- Security groups and NACLs are network controls, not application authentication.
- WAF is not a replacement for secure application code.
- Audit logging is not the same as threat detection.
- Organization guardrails do not grant permissions by themselves.
- Backup creation is not proof that recovery works.

### Example interview scenario

**Requirement:** Secure a production web application.

My proposed control set:

1. Federated administrative access with least privilege.
2. Controlled public application entry point.
3. Private application and database workloads.
4. Restricted security groups.
5. TLS and encryption at rest.
6. Managed secrets.
7. Audit logging and configuration monitoring.
8. Threat and vulnerability monitoring.
9. Tested backup and incident-response procedures.

### Interview-ready answer

> “I select controls by requirement: IAM for authorization, network controls for reachability, KMS for keys, managed secrets for credentials, audit and detection services for visibility, and tested backups for recovery. No single service secures the whole environment.”

**Reference:** [AWS Well-Architected Security Pillar](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/welcome.html).

---

# Quick Revision

1. A public subnet is defined by routing, not its name.
2. A public route alone does not make every instance reachable.
3. KMS commonly protects data keys through envelope encryption.
4. ALB client connections and target connections are separate.
5. Include health-check ports in load-balancer access rules.
6. Terraform state maps configuration to real infrastructure.
7. Never commit Terraform state or credentials to Git.
8. Back up and validate before deleting logs.
9. File modification time is not the timestamp of every log record.
10. Security groups are stateful; NACLs are stateless.
11. A hard link is not an independent backup.
12. Classify SSH errors before changing network rules.
13. EC2 memory and filesystem usage require additional collection.
14. Distinguish ALB-generated errors from target-generated errors.
15. Investigate cost by service and usage type before removing resources.
16. Security requires layered controls, not a single AWS service.
