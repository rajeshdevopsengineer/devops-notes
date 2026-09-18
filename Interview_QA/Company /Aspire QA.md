# Aspire DevOps Interview Questions and Answers

**Company:** Aspire  
**Experience:** 3.4 years  
**Focus:** Kubernetes, Git, AWS, Terraform, HTTP troubleshooting, and operations

> Interview tip: Start with a one-line definition, explain the difference or workflow, and finish with a practical example or troubleshooting command.

---

## 1. What is the difference between ReplicaSet and DaemonSet?

### ReplicaSet

A ReplicaSet maintains a specified number of identical Pod replicas across the cluster. If a Pod fails, it creates a replacement. In practice, a Deployment normally manages ReplicaSets and adds rolling updates and rollback support.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
```

### DaemonSet

A DaemonSet runs one Pod on every eligible node, or on selected nodes using labels, affinity, and tolerations. When a matching node joins, the DaemonSet creates a Pod on it; when the node is removed, that Pod is cleaned up.

Common uses:

- Log collectors such as Fluent Bit
- Node-monitoring agents
- CNI node agents
- Storage or security agents

### Key difference

- ReplicaSet controls the **desired total replica count**.
- DaemonSet controls **node coverage**, normally one Pod per eligible node.
- Use a Deployment for stateless applications and a DaemonSet for node-level services.

References: Kubernetes ReplicaSet and DaemonSet documentation. citeturn5search84turn5search109

---

## 2. What is the difference between PV and PVC in Kubernetes?

### PersistentVolume

A PersistentVolume, or PV, represents storage available to the cluster. It can be created statically by an administrator or dynamically through a StorageClass and CSI provisioner. A PV is cluster-scoped and has a lifecycle independent of a Pod.

### PersistentVolumeClaim

A PersistentVolumeClaim, or PVC, is a namespace-scoped request for storage made by an application. It specifies requirements such as capacity, access mode, volume mode, and StorageClass.

```text
Pod -> PVC -> matching or dynamically created PV -> physical storage
```

Example PVC:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: application-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3
  resources:
    requests:
      storage: 20Gi
```

Important points:

- PV is the **storage resource**; PVC is the **request or claim**.
- Binding is normally one-to-one.
- A Pod mounts a PVC, not usually a PV directly.
- The PV reclaim policy, `Delete` or `Retain`, controls what happens to the backing storage after claim deletion.
- Access modes describe mounting capabilities, not a complete authorization mechanism.

References: Kubernetes PersistentVolume documentation. citeturn5search89turn5search88

---

## 3. What is the difference between `git pull` and `git fetch`?

### `git fetch`

Downloads remote commits, branches, tags, and objects and updates remote-tracking references such as `origin/main`. It does not automatically integrate them into the checked-out branch or modify working files.

```bash
git fetch origin
git log --oneline HEAD..origin/main
git diff HEAD..origin/main
```

### `git pull`

Fetches remote changes and then integrates the selected upstream branch into the current branch using the configured strategy, such as fast-forward, rebase, or merge.

```bash
git pull --ff-only origin main
git pull --rebase origin main
```

### Recommended answer

Use `fetch` when you want to inspect remote changes safely before integration. Use `pull` when you want a combined fetch-and-integrate operation. In automation, explicitly select `--ff-only` or `--rebase` so the behavior is predictable.

References: Official Git fetch and pull documentation. citeturn5search91turn5search93

---

## 4. What do `git stash` and `git stash pop` mean?

`git stash` temporarily records uncommitted working-tree and index changes and restores a clean working state. It is useful before changing branches or pulling updates when unfinished work is not ready to commit.

```bash
git stash push -m "WIP: payment validation"
git stash list
git stash show -p stash@{0}
```

`git stash pop` reapplies a stash and removes it from the stash list when application succeeds.

```bash
git stash pop
git stash pop stash@{1}
```

Related distinction:

- `git stash apply` restores changes but keeps the stash entry.
- `git stash pop` restores changes and then drops the entry if successful.
- Use `-u` to include untracked files: `git stash push -u`.
- Conflicts can occur when applying a stash to changed code and must be resolved manually.

References: Official Git stash documentation. citeturn5search90turn5search95

---

## 5. What is the difference between a NACL and a Security Group?

### Security Group

- Applied to supported resources or network interfaces.
- Stateful: response traffic for an allowed connection is automatically permitted.
- Supports allow rules, not explicit deny rules.
- All rules are evaluated together.
- Can reference another security group in supported scenarios.
- Primary control for workload-level network access.

### Network ACL

- Applied at subnet boundaries.
- Stateless: return traffic must be explicitly allowed.
- Supports both allow and deny rules.
- Rules are evaluated by rule number from lowest to highest, and the first match wins.
- Uses CIDR, protocol, and port rules rather than security-group references.
- Useful as coarse-grained subnet guardrails and defense in depth.

Example: For an internet-facing HTTPS server, the security group can allow inbound TCP 443. A restrictive NACL must also allow TCP 443 inbound and the relevant ephemeral return-port range outbound.

AWS recommends security groups as the primary mechanism and NACLs where subnet-level, stateless guardrails are needed. citeturn5search98turn5search115turn5search117

---

## 6. What is the use of `terraform fmt`?

`terraform fmt` rewrites Terraform configuration into the canonical HashiCorp style. It improves consistency and readability but does not validate provider credentials, resource semantics, or deploy infrastructure.

```bash
terraform fmt
terraform fmt -recursive
terraform fmt -check -recursive
terraform fmt -check -diff -recursive
```

In CI, `-check` detects formatting differences without rewriting files and returns a non-zero exit status when formatting is required. Use `terraform validate` separately for syntax and internal configuration validation.

References: HashiCorp command reference and CLI workflow guidance. citeturn5search102turn5search106

---

## 7. What is the use of `terraform import`?

`terraform import` associates an existing real infrastructure object with a Terraform resource address in state, allowing Terraform to begin managing it.

Example:

```hcl
resource "aws_s3_bucket" "logs" {
  bucket = "existing-company-logs"
}
```

```bash
terraform import aws_s3_bucket.logs existing-company-logs
terraform plan
```

Important points:

- The traditional CLI import updates state but does not automatically create the full configuration.
- Write the destination resource block before a CLI import.
- Modern import blocks can be reviewed through the plan/apply workflow and can support configuration-generation workflows.
- Run `terraform plan` after import and align configuration until Terraform proposes no unintended changes.
- Back up state and ensure one remote object is mapped to only one Terraform resource address.

References: HashiCorp import documentation. citeturn5search103turn5search105

---

## 8. What are providers and provisioners in Terraform?

### Provider

A provider is a plugin through which Terraform interacts with an external API, such as AWS, Azure, Kubernetes, GitHub, or Datadog. It exposes resources and data sources and handles API operations.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = "ap-south-1"
}
```

### Provisioner

A provisioner performs an imperative action associated with resource creation or destruction.

- `local-exec`: Runs a command on the machine executing Terraform.
- `remote-exec`: Runs commands on a remote resource.
- `file`: Copies files to a remote resource.

```hcl
provisioner "local-exec" {
  command = "echo ${self.id} >> instance-ids.txt"
}
```

Provisioners should be a last resort because their side effects are difficult for Terraform to model reliably. Prefer cloud-init or `user_data`, immutable images, configuration-management tools, or provider-native resources. citeturn5search104turn5search107

---

## 9. What does CRR mean in Amazon S3?

CRR means **Cross-Region Replication**. It asynchronously copies selected S3 objects from a source bucket to one or more destination buckets in different AWS Regions.

Use cases:

- Disaster recovery and secondary copies
- Compliance and data residency
- Lower-latency regional access
- Account and Region isolation

Requirements and behavior:

- Enable versioning on source and destination buckets.
- Configure a replication rule and an IAM role that S3 can assume.
- Add destination bucket permissions for cross-account replication.
- Grant required KMS permissions for SSE-KMS objects.
- Live replication applies to new or updated objects after configuration; use S3 Batch Replication for existing objects.
- Monitor replication status and failures rather than assuming immediate consistency.

References: Amazon S3 replication overview and requirements. citeturn5search119turn5search120turn5search124

---

## 10. In which cases do 503 errors occur in DevOps?

HTTP `503 Service Unavailable` means the server or intermediary is temporarily unable to handle the request. The exact source matters, so first identify whether the response came from the application, ingress, load balancer, API gateway, CDN, proxy, or service mesh.

Common causes:

- No healthy targets or endpoints behind a load balancer, ingress, or Kubernetes Service
- Application overload, exhausted worker threads, connection pools, file descriptors, or memory
- Deployment rollout temporarily leaving insufficient ready replicas
- Incorrect readiness probes causing all Pods to be removed from endpoints
- Dependency failure, such as database or downstream API outage
- Autoscaling lag or exhausted cluster capacity
- Rate limiting, throttling, concurrency exhaustion, or maintenance mode
- Proxy or upstream connection failures
- DNS, routing, security-group, NACL, or NAT connectivity problems
- Lambda concurrency limits or slow downstream connections

Troubleshooting order:

```bash
curl -vk https://service.example.com/health
kubectl get pods,endpoints,endpointslices -A
kubectl describe ingress <name>
kubectl get events -A --sort-by=.lastTimestamp
kubectl logs <pod> --previous
```

Then correlate load-balancer status codes, target status codes, target health, application logs, latency, saturation, recent deployments, and dependency health. AWS documents that VPC routes, security groups, NACLs, NAT, IP exhaustion, and ENI availability can cause Lambda timeouts or connectivity failure. citeturn5search96turn5search97

---

## 11. What are the different types of AWS Lambda triggers?

There are two main event-driven patterns:

### Direct invocation, or push

A service directly invokes the function when an event occurs. Examples include:

- API Gateway or Lambda Function URL for HTTP requests
- S3 object events
- SNS notifications
- EventBridge events and schedules
- Application Load Balancer requests
- Cognito and selected service events

The invocation can be synchronous or asynchronous depending on the service.

### Event source mapping, or pull/poll

Lambda pollers read batches from a queue or stream and invoke the function. Examples include:

- SQS
- Kinesis Data Streams
- DynamoDB Streams
- Amazon MSK
- Self-managed Apache Kafka
- Amazon MQ
- DocumentDB change streams

Make handlers idempotent because queue and stream processing can deliver a record more than once. Configure retries, partial batch responses where supported, dead-letter handling, and destinations based on the event source.

References: AWS Lambda invocation and event-source documentation. citeturn5search125turn5search127turn5search128turn5search129

---

## 12. What are a NAT gateway and NAT instance?

Both provide IPv4 network address translation so private resources can initiate outbound connections while not accepting unsolicited inbound internet connections through that path.

### NAT gateway

- AWS-managed and highly available within its Availability Zone
- Automatically scales within service limits
- No operating-system patching by the customer
- No security group attached directly to it
- Charged per gateway-hour and processed data
- A public NAT gateway belongs in a public subnet and uses an Elastic IP

### NAT instance

- An EC2 instance configured as a NAT router
- Customer manages AMI, patching, scaling, recovery, routes, and monitoring
- Source/destination checks must be disabled
- Security groups can be attached
- Capacity is limited by the selected instance and architecture
- Can support customized firewall or forwarding behavior

### Selection

Use a NAT gateway for most standard production outbound-internet requirements. Consider an appliance or NAT instance only when specialized control justifies its operational burden. For high availability, deploy a NAT gateway in each AZ and route each private subnet to the local-AZ gateway.

AWS networking documentation describes NAT-related connectivity failures and required Internet Gateway, Elastic IP, subnet-address, and quota conditions. citeturn5search100turn5search96

---

## 13. How do you find and remove orphaned Kubernetes resources?

An orphan resource is generally an object no longer required by or correctly attached to its intended owner. Kubernetes garbage collection uses `metadata.ownerReferences`, but custom resources, finalizers, `Retain` policies, and manual operations can leave resources behind. citeturn5search108turn5search113

### Audit before deleting

```bash
kubectl get all -A
kubectl get pvc,pv -A
kubectl get jobs -A
kubectl get events -A --sort-by=.lastTimestamp
kubectl get events -A --field-selector=reason=OwnerRefInvalidNamespace
```

Find failed, evicted, succeeded, or terminating Pods:

```bash
kubectl get pods -A --field-selector=status.phase=Failed
kubectl get pods -A --field-selector=status.phase=Succeeded
```

Inspect ownership and finalizers:

```bash
kubectl get pod <pod> -n <namespace> -o jsonpath='{.metadata.ownerReferences}'
kubectl get <kind> <name> -n <namespace> -o jsonpath='{.metadata.finalizers}'
```

Check resources with no controller owner using `jq`:

```bash
kubectl get pods -A -o json | jq -r '
  .items[]
  | select((.metadata.ownerReferences // []) | length == 0)
  | [.metadata.namespace, .metadata.name] | @tsv'
```

### Safe removal examples

```bash
kubectl delete pod <pod> -n <namespace>
kubectl delete job <job> -n <namespace>
kubectl delete pvc <claim> -n <namespace>
kubectl delete namespace <namespace>
```

Before deleting a PVC or PV, verify application ownership, snapshots/backups, StorageClass, reclaim policy, and cloud disk. A `Retain` PV may require deliberate data cleanup and PV handling. Do not blindly remove finalizers because they often protect required cleanup. Remove a finalizer manually only after understanding why its controller cannot complete and what external resource could be leaked.

Prevent recurrence with Job TTLs, CronJob history limits, correct owner references, Helm/GitOps uninstall workflows, labels, quotas, and periodic audits. Kubernetes automatically garbage-collects many owned dependents and selected unused resources, but not every operationally unused object. citeturn5search108turn5search113

---

## 14. What is a sticky session in an ALB?

Sticky sessions, also called session affinity, make an Application Load Balancer send subsequent requests from the same client to the same target for a configured duration.

Flow:

```text
First request -> ALB selects target B -> response sets/uses cookie
Next request with cookie -> ALB routes to target B
```

ALB supports:

- **Load-balancer-generated cookie stickiness**, using an ALB-managed cookie and configured duration.
- **Application-cookie stickiness**, using a named application-generated cookie together with ALB behavior.

Use it when an application stores temporary session state locally, such as a legacy shopping cart. However, the preferred cloud-native design is usually stateless application instances with session state in Redis, a database, or another shared store.

Limitations and risks:

- Can distribute load unevenly.
- Does not rescue a request when the bound target is unhealthy; ALB routes according to target health and stickiness behavior.
- Makes scaling and deployments more sensitive to session duration.
- Cookie behavior must be tested with browsers, expiry settings, and cross-origin requirements.

When stickiness is enabled, the routing algorithm chooses the initial target and later requests from that client bypass normal target selection while affinity applies. citeturn5search131turn5search132turn5search134

---

## Rapid Revision

- **ReplicaSet:** Keeps a desired number of Pods. **DaemonSet:** Runs a Pod on every eligible node.
- **PV:** Storage resource. **PVC:** Namespace-scoped request for storage.
- **Fetch:** Downloads remote history. **Pull:** Fetches and integrates it.
- **Stash:** Temporarily saves uncommitted changes. **Pop:** Restores and removes the stash on success.
- **Security Group:** Stateful, resource-level allow rules. **NACL:** Stateless, subnet-level allow and deny rules.
- **`terraform fmt`:** Canonical formatting. **`terraform import`:** Links existing infrastructure to Terraform state.
- **Provider:** Talks to an external API. **Provisioner:** Runs imperative post-create or destroy actions and should be a last resort.
- **CRR:** Asynchronous S3 replication across Regions.
- **503:** Service temporarily unavailable, often because of no healthy backends, overload, dependency failure, or connectivity issues.
- **Lambda triggers:** Direct push invocation or event source mappings that poll queues and streams.
- **NAT gateway:** Managed service. **NAT instance:** Customer-managed EC2-based NAT.
- **Orphan cleanup:** Inspect ownership, finalizers, storage policy, and external dependencies before deletion.
- **ALB stickiness:** Cookie-based affinity to the same target for a configured duration.
