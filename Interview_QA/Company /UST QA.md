**1. If a resource exists in the cloud but is not present in Terraform, how do you manage it?**

Use **Terraform import** to bring the existing resource under Terraform management.

Terraform needs both:

- **Configuration:** The desired settings written in `.tf` files.
- **State:** The mapping between Terraform resource addresses and actual cloud resource IDs.

My approach would be:

1. Identify the resource, its AWS account, Region, dependencies, and current configuration.
2. Confirm another Terraform state does not already manage it.
3. Write a matching Terraform resource block.
4. Import the resource into the correct backend and workspace.
5. Run `terraform plan`.
6. Reconcile differences until the plan contains only intended changes.

For example, after defining an existing EC2 instance as `aws_instance.existing`:

```bash
terraform init
terraform workspace show

terraform import aws_instance.existing i-0123456789abcdef0

terraform plan
```

The CLI import command updates state; it does **not** automatically write the resource configuration. An imported resource should map to one Terraform resource address. [HashiCorp Developer](https://developer.hashicorp.com/terraform/cli/import?utm_source=chatgpt.com)

You can also declare the import in configuration:

```hcl
import {
  to = aws_instance.existing
  id = "i-0123456789abcdef0"
}
```

This allows the import to participate in the normal plan and apply workflow. Terraform also supports generating initial configuration:

```bash
terraform plan -generate-config-out=generated.tf
```

Review generated configuration carefully before applying it. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/import?utm_source=chatgpt.com)

**Important distinction:** If Terraform only needs to look up an existing resource—for example, find an existing VPC ID—use a **data source**. Import is appropriate when Terraform should manage that resource’s lifecycle.

---

**2. Terraform apply is creating all the resources again. What could be wrong?**

First, **stop and inspect the plan before approving it**.

Terraform uses state to connect configuration to existing infrastructure. If it cannot find the correct state, it may propose creating resources that already exist. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/state?utm_source=chatgpt.com)

Common causes include:

| Possible cause | What I would check |
|---|---|
| State was deleted or is empty | State file, remote object, backups, and object versions |
| Wrong backend configuration | Bucket, state key, storage account, or workspace |
| CI runner starts with local state each time | Whether a persistent remote backend is configured |
| Wrong directory or workspace | Current root module and selected workspace |
| Resource or module addresses changed | Renamed resources or resources moved into modules |
| Changed `count` ordering or `for_each` keys | Whether resource identities changed |
| Replacement is required | Attributes marked as forcing replacement, or an explicit `-replace` |

Useful commands:

```bash
terraform workspace show
terraform state list
terraform plan
```

Understand the plan symbols:

- `+`: Create.
- `~`: Update in place.
- `-`: Destroy.
- `-/+` or `+/-`: Replace.

A replacement is different from Terraform proposing a new resource because its state mapping is missing.

If a resource was renamed, preserve its identity with a `moved` block:

```hcl
moved {
  from = aws_instance.old_name
  to   = aws_instance.new_name
}
```

Terraform can then update the address without unnecessarily destroying and recreating the resource. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/modules/develop/refactoring?utm_source=chatgpt.com)

**Fix the underlying issue:** Restore missing state, correct the backend/workspace, or declare address moves. Repeatedly running apply will not repair a missing mapping.

---

**3. How can AI assist in cloud infrastructure monitoring?**

AI can help engineers interpret large amounts of telemetry and identify unusual behavior faster.

| Use case | Practical benefit |
|---|---|
| Anomaly detection | Detect changes from normal CPU, latency, traffic, or error patterns |
| Alert correlation | Group related alerts into a likely incident |
| Log and trace analysis | Summarize failures and identify recurring patterns |
| Capacity forecasting | Predict when resources may approach capacity |
| Investigation assistance | Suggest likely causes and relevant runbooks |

For example, AWS CloudWatch anomaly detection learns expected metric behavior, including recurring hourly, daily, and weekly patterns. An alarm can trigger when a metric moves outside the expected range. [Amazon CloudWatch](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Anomaly_Detection.html?utm_source=chatgpt.com)

Suppose CPU regularly reaches 70% during an evening traffic peak. A static threshold may repeatedly alert. An anomaly model can recognize that pattern while highlighting an unexpected CPU increase at another time.

For a latency incident, an AI assistant could summarize:

- When latency started.
- Which services are affected.
- Whether errors increased.
- Which deployments or configuration changes occurred nearby.
- Which dependencies show abnormal behavior.

These are **investigation hypotheses** that engineers must verify using metrics, traces, logs, and change history.

I would retain explicit alerts for customer impact, such as failed payments or an SLO breach. AI output should complement those alerts, and any automated remediation should have restricted permissions and clear execution limits.

---

**4. How can AI improve developer productivity?**

AI can reduce time spent on repetitive development and investigation tasks.

Examples include:

- Generating initial code, Terraform, Dockerfiles, and pipeline definitions.
- Explaining unfamiliar code or error messages.
- Drafting unit tests and identifying missing test cases.
- Suggesting refactoring opportunities.
- Reviewing changes for potential defects.
- Creating documentation and release notes.

GitHub Copilot, for example, supports code suggestions, codebase questions, change reviews, and assistance with development tasks. Developers remain responsible for reviewing its output. [GitHub Docs](https://docs.github.com/en/copilot/get-started/what-is-github-copilot?utm_source=chatgpt.com)

A practical DevOps example would be asking AI to draft a Jenkins pipeline containing build, test, image scanning, and deployment stages. The engineer then checks:

1. Whether the commands match the application.
2. Whether credentials are handled securely.
3. Whether dependencies and plugin syntax are correct.
4. Whether failures correctly stop deployment.
5. Whether the pipeline passes validation and testing.

I would measure the benefit through shorter feedback cycles, faster reviews, and fewer escaped defects. Generated code still needs normal engineering checks, and sensitive information should only be shared with approved tools.

---

**5. A Python program is failing because of memory issues. What could cause it?**

First, determine **how the process failed**.

- A Python `MemoryError` indicates an allocation failed and Python could raise an exception. [Python 3.15.0 documentation](https://docs.python.org/3/library/exceptions.html?utm_source=chatgpt.com)
- An operating-system or container memory kill may terminate the process without a Python traceback.
- Exit code `137` indicates termination by `SIGKILL`; it does not, by itself, prove an out-of-memory event.

Common causes include:

| Cause | Example |
|---|---|
| Loading too much data | Reading a huge file or dataset entirely into memory |
| Unbounded collections | A list, dictionary, queue, or cache grows indefinitely |
| Retained references | Objects remain reachable after their useful lifetime |
| Excessive concurrency | Many workers each hold large datasets |
| Data copying | Repeated DataFrame or array copies |
| Native allocations | A library allocates memory outside normal Python tracking |
| Insufficient memory allocation | The workload exceeds the VM or container limit |

Start with system evidence:

```bash
free -h

ps -eo pid,comm,rss,%mem --sort=-rss | head

journalctl -k | grep -Ei 'oom|out of memory|killed process'
```

For Kubernetes:

```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous
kubectl top pod <pod-name> -n <namespace>
```

Look for `OOMKilled`, memory limits, and memory growth before termination.

For Python allocation analysis, use `tracemalloc`, starting it before the workload and comparing snapshots to identify growing allocation sites. It tracks Python allocations; it does not provide a complete picture of all native memory or process RSS. [Python 3.15.0 documentation](https://docs.python.org/3/library/tracemalloc.html?utm_source=chatgpt.com)

Fixes could include streaming files, processing data in chunks, bounding caches and queues, reducing workers, avoiding unnecessary copies, or increasing memory when the workload legitimately requires it.

---

**6. A CI pipeline takes 45 minutes. How would you optimize it?**

I would first measure where those 45 minutes are spent. Separate **queue time** from actual execution time, then identify the stages controlling total completion time.

| Bottleneck | Improvement |
|---|---|
| Dependency downloads | Cache dependencies using lockfile, runtime, and OS information |
| Repeated Docker builds | Use BuildKit caching and an external cache |
| Independent tests run sequentially | Run them in parallel with sufficient agent capacity |
| Everything builds after every change | Build affected services and their dependencies |
| Slow agents | Right-size agents and investigate CPU, memory, disk, or network saturation |
| Rebuilding for every environment | Build once and promote the same immutable artifact |
| Large Docker build context | Add `.dockerignore` and exclude unnecessary files |

For Docker, copy dependency manifests and install dependencies before copying frequently changing application files. This preserves reusable layers when only application code changes. Cache mounts and external caches can also reduce repeated downloads and builds. [Docker Docs](https://docs.docker.com/build/cache/optimize/?utm_source=chatgpt.com)

Dependency cache keys should change when relevant inputs change. Avoid storing credentials in caches, and distinguish a cache from a release artifact. [GitHub Docs](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows?utm_source=chatgpt.com)

For example, three independent ten-minute test suites might contribute approximately ten minutes when run in parallel, provided the agents have enough capacity.

I would preserve necessary security and quality gates, then compare stage timings, reliability, and runner cost before and after the changes.

---

**7. Write Terraform code to create EC2 with variables for instance type and Region.**

The following example creates **one instance by default**. It also includes an instance-count variable for the next question.

It assumes an existing subnet and security group. Supply an AMI compatible with the instance type and available in the selected Region. Terraform’s AWS provider uses the configured Region when managing resources. [HashiCorp Developer](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/aws-build?utm_source=chatgpt.com)

**`main.tf`:**

```hcl
terraform {
  required_version = ">= 1.5, < 2.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = var.region
}

variable "region" {
  description = "AWS Region"
  type        = string
  default     = "ap-south-1"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
}

variable "ami_id" {
  description = "Compatible AMI ID in the selected Region"
  type        = string
}

variable "subnet_id" {
  description = "Existing subnet ID"
  type        = string
}

variable "security_group_ids" {
  description = "Security groups belonging to the subnet's VPC"
  type        = set(string)
}

variable "instance_count" {
  description = "Number of EC2 instances"
  type        = number
  default     = 1

  validation {
    condition = (
      var.instance_count >= 1 &&
      floor(var.instance_count) == var.instance_count
    )
    error_message = "instance_count must be a positive integer."
  }
}

resource "aws_instance" "web" {
  count = var.instance_count

  ami                         = var.ami_id
  instance_type               = var.instance_type
  subnet_id                   = var.subnet_id
  vpc_security_group_ids      = var.security_group_ids
  associate_public_ip_address = false

  root_block_device {
    volume_type = "gp3"
    encrypted   = true
  }

  metadata_options {
    http_tokens = "required"
  }

  tags = {
    Name      = format("ust-web-%02d", count.index + 1)
    ManagedBy = "Terraform"
  }
}

output "instance_ids" {
  value = aws_instance.web[*].id
}

output "private_ips" {
  value = aws_instance.web[*].private_ip
}
```

**`terraform.tfvars`:** Replace these illustrative IDs with real values.

```hcl
region             = "ap-south-1"
instance_type      = "t3.micro"
ami_id             = "ami-0123456789abcdef0"
subnet_id          = "subnet-0123456789abcdef0"
security_group_ids = ["sg-0123456789abcdef0"]
instance_count     = 1
```

Commands:

```bash
terraform init
terraform fmt
terraform validate
terraform plan -out=tfplan

# Execute after reviewing the saved plan:
terraform apply tfplan
```

Authenticate through an appropriate AWS role or approved credential mechanism; keep access keys out of the configuration. The example disables automatic public-IP assignment and requires IMDSv2 tokens. [Amazon Elastic Compute Cloud](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html?utm_source=chatgpt.com)

---

**8. How would you create 50 instances in one go?**

For 50 similar instances, use the `count` meta-argument. The previous configuration already supports this:

```bash
terraform plan -var="instance_count=50" -out=tfplan
terraform apply tfplan
```

Terraform gives each instance a separate address:

```text
aws_instance.web[0]
aws_instance.web[1]
...
aws_instance.web[49]
```

`count.index` provides the zero-based index used to generate each instance’s name. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/meta-arguments/count?utm_source=chatgpt.com)

If these instances require stable business identities, use `for_each` with explicit keys instead:

```hcl
locals {
  instance_names = toset([
    for n in range(1, 51) : format("web-%02d", n)
  ])
}

# In the resource, use this instead of count:
# for_each = local.instance_names
#
# In its tags, use:
# Name = each.key
```

`for_each` identifies instances by their map keys or set members. It is useful when removing one named instance should not change other instances’ identities. Do not use `count` and `for_each` on the same resource block. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/meta-arguments/for_each?utm_source=chatgpt.com)

Before applying, check EC2 quotas, subnet IP availability, capacity, and cost. One apply requests the desired fleet; Terraform and AWS still limit concurrent operations. For an application fleet requiring automatic scaling and replacement, consider an Auto Scaling group.

---

**9. Someone deleted the local Terraform state file. What can be done?**

**Do not immediately run apply.** First determine whether local state was authoritative.

**If you use a remote backend:** Verify that its state object still exists. Losing local initialization files does not necessarily mean the remotely stored infrastructure state is lost. Reinitialize the working directory against the correct backend and verify:

```bash
terraform init
terraform state list
terraform plan
```

**If you genuinely use local state:** Look for `terraform.tfstate.backup` or another known-good backup. Terraform normally maintains a backup of the previous local state. Restore it, then check for changes made after that snapshot. [ibm.com](https://www.ibm.com/support/pages/terraform-state-restoration-overview?utm_source=chatgpt.com)

**If there is no usable backup:**

1. Inventory the existing cloud resources.
2. Recover their IDs and configurations.
3. Import them into matching resource addresses.
4. Review the plan carefully.
5. Apply only after the mappings are correct.

A refresh cannot automatically reconstruct all missing resource mappings from an empty state.

For prevention, use a secured remote backend with locking and versioned backups. With the S3 backend, bucket versioning supports recovery, and native locking can be enabled through `use_lockfile = true` on supported Terraform versions. DynamoDB-based locking is deprecated. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/backend/s3?utm_source=chatgpt.com)

---

**10. A production instance is failing. What could be the causes?**

I would first establish the scope: **Is the VM unhealthy, the application unhealthy, or the application unreachable?**

For EC2, examine system, instance, and attached-EBS status checks. Also inspect application-level checks and load-balancer target health. Passing infrastructure checks does not prove that the application works. AWS also supports opt-in EC2 application status checks. [Amazon Elastic Compute Cloud](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/monitoring-system-instance-status-check.html?utm_source=chatgpt.com)

Likely causes include:

| Layer | Possible problems |
|---|---|
| AWS infrastructure | Host or storage impairment |
| Operating system | Memory exhaustion, full disk, filesystem or startup problems |
| Application | Crashed process, bad deployment, exhausted thread pool |
| Network | Security group, NACL, route, DNS, or load-balancer configuration |
| Dependencies | Database outage, connection exhaustion, API timeout |
| Configuration | Expired certificate, rotated secret, invalid environment variable |

Useful Linux checks:

```bash
systemctl status myapp
journalctl -u myapp --since "30 minutes ago"

free -h
df -h
df -i
ss -lntp
```

Then check application logs, recent changes, resource graphs, and dependency health.

My recovery priority would be restoring service through healthy instances, replacement capacity, or a justified rollback. I would retain relevant evidence for root-cause analysis and verify recovery with a real application transaction.

---

**11. During disaster recovery, users cannot access the application. What would you do?**

I would follow the DR runbook and check the entire request path, including dependencies.

The recovery objective is to restore service within the agreed **RTO**, using data that satisfies the agreed **RPO**. Those targets should come from business requirements. [Reliability Pillar](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/plan-for-disaster-recovery-dr.html?utm_source=chatgpt.com)

My sequence would be:

1. **Confirm the incident:** Determine whether the primary Region, application, network, or a dependency failed.
2. **Check the recovery environment:** Validate compute capacity, images, configuration, secrets, certificates, and IAM permissions.
3. **Recover the database:** Check replication lag or the backup recovery point; promote or restore as required.
4. **Prevent conflicting writes:** Ensure an old primary cannot continue accepting writes after failover.
5. **Validate application health:** Confirm that it can connect to the recovered database and other dependencies.
6. **Check traffic redirection:** Verify DNS, load-balancer targets, routing, firewall rules, and WAF policies.
7. **Test externally:** Validate access from outside the recovery network.
8. **Verify business transactions:** Test login, reads, writes, and relevant critical workflows.

A healthy DR application can still be inaccessible because DNS points to the old endpoint, resolver caches have not expired, certificates are incorrect, or ingress rules differ.

Backups alone do not provide an immediately available recovery environment. Recovery procedures need regular testing, and asynchronous replication can leave a data-loss window. Failback should be controlled after the primary environment and data consistency have been verified.

---

**12. There is a sudden traffic spike. How would you troubleshoot it?**

I would identify **what generated the traffic** and **which component is becoming saturated**.

First, examine:

- Requests per second and concurrent requests.
- Source patterns and affected URLs.
- HTTP status codes and latency percentiles.
- CPU, memory, network, and disk utilization.
- Database connections, query latency, and queue depth.
- Recent releases, campaigns, scheduled jobs, or retry behavior.

Then distinguish legitimate demand from bot traffic, an attack, or a retry storm.

| Finding | Appropriate response |
|---|---|
| Legitimate demand overloads application servers | Scale healthy application capacity |
| Repeated requests for cacheable content | Improve CDN or application caching |
| Bot or abusive traffic | Apply targeted WAF and rate-limiting rules |
| Database connection exhaustion | Bound connection pools and protect database capacity |
| Retry amplification | Use bounded retries, backoff, jitter, and circuit breakers |
| Work can be asynchronous | Queue it and apply backpressure |

For an Auto Scaling group, verify that the scaling metric represents the bottleneck and that new instances become healthy before serving traffic. Configure instance warmup appropriately so startup behavior does not distort scaling decisions. [Amazon EC2 Auto Scaling](https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling-default-instance-warmup.html?utm_source=chatgpt.com)

After mitigation, verify that customer-facing latency and error rates recover. Scaling application servers alone may worsen an already overloaded database, so the response must follow the evidence.

---

**13. Credentials are visible in CI/CD logs. What would you do?**

Treat the credentials as **compromised**, even if the logs have limited access.

My response would be:

1. **Contain exposure:** Stop further printing and restrict access to affected logs and artifacts.
2. **Revoke or rotate:** Invalidate exposed credentials promptly and update dependent systems securely.
3. **Investigate usage:** Review access logs and audit trails for unauthorized activity.
4. **Remove exposed copies:** Clean affected logs, artifacts, caches, or repositories according to incident-handling requirements.
5. **Fix the source:** Identify why the pipeline printed the secret.
6. **Verify recovery:** Confirm that workloads use the replacement credential and the old credential no longer works.

GitHub explicitly recommends rotating a secret and deleting affected logs when an unredacted secret reaches workflow logs. [GitHub Docs](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions?utm_source=chatgpt.com)

Common exposure sources include:

- Shell tracing with `set -x`.
- Printing environment variables.
- Verbose commands that expose authorization headers.
- Debug logs containing connection strings.
- Secrets embedded in build arguments or artifacts.

For AWS credentials, use CloudTrail to investigate usage. Prefer temporary credentials through IAM roles rather than long-lived access keys. [AWS Identity and Access Management](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html?utm_source=chatgpt.com)

Prevent recurrence with managed secret stores, short-lived workload identity, least privilege, restricted production credentials, and log masking. Masking is an additional protection; it does not undo an earlier exposure.

---

**14. Explain Blue-Green deployment.**

Blue-Green deployment maintains two application environments:

- **Blue:** The version currently serving production traffic.
- **Green:** The replacement version being prepared and validated.

The application is deployed to Green while Blue continues serving users. Once Green passes validation, traffic switches to it. Keeping Blue available provides a fast route back if the new version fails. AWS CodeDeploy supports this approach for EC2 deployments by directing traffic to replacement instances. [AWS CodeDeploy](https://docs.aws.amazon.com/codedeploy/latest/userguide/welcome.html?utm_source=chatgpt.com)

A practical workflow is:

| Step | Action |
|---|---|
| 1 | Build and scan an immutable application artifact |
| 2 | Deploy it to Green |
| 3 | Run readiness, integration, and smoke tests |
| 4 | Switch the load-balancer routing to Green |
| 5 | Monitor errors, latency, and business transactions |
| 6 | Route traffic back to Blue if validation fails |
| 7 | Retire Blue after the agreed observation period |

For AWS, this could involve separate Blue and Green target groups behind an ALB. In Kubernetes, separate Deployments can run both versions, with a Service or ingress routing configuration selecting the active version.

**Database compatibility is critical.** Both versions may use the same database, so schema changes should support both during the transition—for example, add a new column before removing the old one.

Switching traffic back does not automatically reverse database writes or destructive migrations. Also account for connection draining, sessions, and background workers.

Blue-Green provides controlled cutover and fast application rollback, with the tradeoff of temporarily maintaining additional capacity.
