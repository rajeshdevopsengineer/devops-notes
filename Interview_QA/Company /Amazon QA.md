# DevOps and Cloud Interview Questions

> Interview-ready answers covering AWS, Kubernetes, CI/CD, container images, networking, monitoring, and Terraform.

## 1. Sending log files from EC2 to S3

1. Create an S3 bucket and enable encryption, versioning, and lifecycle rules as required.
2. Create an IAM role trusted by EC2.
3. Grant the role least-privilege permissions such as `s3:PutObject` for the required bucket prefix.
4. Attach the IAM role to the EC2 instance. Do not store long-lived AWS access keys on the server.
5. Install the AWS CLI if it is not already available.
6. Compress and upload logs using `aws s3 cp` or `aws s3 sync`.
7. Schedule the operation with `cron`, a `systemd` timer, or AWS Systems Manager.
8. Monitor exit codes, upload failures, S3 events, and CloudTrail records.

Example:

```bash
#!/usr/bin/env bash
set -euo pipefail

BUCKET="company-application-logs"
DATE="$(date -u +%Y-%m-%d)"
ARCHIVE="/tmp/application-${DATE}.tar.gz"

tar -czf "${ARCHIVE}" /var/log/my-application
aws s3 cp "${ARCHIVE}" "s3://${BUCKET}/ec2-logs/${HOSTNAME}/${DATE}/" --sse AES256
rm -f "${ARCHIVE}"
```

Example cron entry:

```cron
0 * * * * /usr/local/bin/upload-logs.sh >> /var/log/log-upload.log 2>&1
```

For near-real-time log centralization, send logs to CloudWatch Logs with the CloudWatch agent and archive them to S3 separately.

---

## 2. Limiting Kubernetes resource usage through a namespace

Use two namespace-scoped objects:

- `ResourceQuota` limits aggregate resource consumption in a namespace.
- `LimitRange` defines default, minimum, and maximum resources for individual containers, Pods, or PVCs.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: development-quota
  namespace: development
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "50"
    persistentvolumeclaims: "10"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: development-limits
  namespace: development
spec:
  limits:
    - type: Container
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      default:
        cpu: 500m
        memory: 512Mi
      min:
        cpu: 50m
        memory: 64Mi
      max:
        cpu: "2"
        memory: 4Gi
```

Apply and verify:

```bash
kubectl apply -f namespace-resources.yaml
kubectl describe resourcequota -n development
kubectl describe limitrange -n development
```

---

## 3. Three-tier architecture

A three-tier architecture separates an application into independently managed layers.

### Presentation tier

Handles the user interface and incoming requests. Examples include Route 53, CloudFront, AWS WAF, an Application Load Balancer, a website, or frontend servers.

### Application tier

Executes business logic, validation, authentication, and APIs. Examples include EC2 Auto Scaling groups, EKS, ECS, or Lambda. It usually runs in private subnets.

### Data tier

Stores persistent data and cache. Examples include RDS, Aurora, DynamoDB, ElastiCache, and S3. Databases should normally be deployed in private subnets.

```text
User -> Route 53 -> CloudFront/WAF -> Public ALB
     -> Application servers in private subnets
     -> Database in isolated/private subnets
```

Production recommendations include Multi-AZ deployment, Auto Scaling, TLS, least-privilege security groups, backups, encryption, monitoring, and secrets management.

---

## 4. Updating Kubernetes worker nodes

Recommended rolling-replacement process:

1. Review Kubernetes version skew, deprecated APIs, add-on compatibility, and Pod Disruption Budgets.
2. Create or update a node group with the target Kubernetes version, AMI, and configuration.
3. Confirm that new nodes join and become `Ready`.
4. Cordon an old node to prevent new Pods from being scheduled.
5. Drain the node so workloads move safely.
6. Verify application health.
7. remove the old node and repeat gradually.

```bash
kubectl get nodes -o wide
kubectl cordon worker-node-01
kubectl drain worker-node-01 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=60 \
  --timeout=10m
```

After an in-place upgrade, use:

```bash
kubectl uncordon worker-node-01
```

For EKS managed node groups, initiate a managed node-group update and configure the allowed unavailable-node count. Validate StatefulSets, local storage, singleton workloads, PDBs, and available spare capacity before draining.

---

## 5. I am an administrator but cannot access an S3 bucket

Administrator-style permissions do not override every AWS policy layer. Check:

1. IAM identity policies
2. Permissions boundary
3. AWS Organizations Service Control Policy
4. S3 bucket policy
5. VPC endpoint policy
6. Assumed-role session policy
7. KMS key policy for SSE-KMS objects
8. Cross-account permissions
9. Explicit deny conditions

A permissions boundary establishes the maximum permissions an identity policy can grant. It does not grant access by itself.

```text
Effective permissions = identity-policy allow intersected with boundary allow
```

An explicit deny in an applicable policy takes precedence over an allow.

Useful checks:

```bash
aws sts get-caller-identity
aws s3api get-bucket-policy --bucket BUCKET_NAME
aws s3api get-public-access-block --bucket BUCKET_NAME
```

Also inspect CloudTrail, IAM Policy Simulator, IAM Access Analyzer, SCPs, endpoint policies, and the KMS key policy.

---

## 6. Various stages of CI/CD

A mature pipeline commonly includes:

1. **Source:** Commit, branch, pull request, and peer review.
2. **Validation:** Formatting, linting, secret detection, and IaC validation.
3. **Build:** Compile the code and build packages or container images.
4. **Unit test:** Validate individual functions and produce code coverage.
5. **Security:** SAST, SCA, image scanning, IaC scanning, license checks, and SBOM creation.
6. **Publish:** Push immutable artifacts to a trusted registry or repository.
7. **Deploy to lower environments:** Development, QA, UAT, and staging.
8. **Integration testing:** API, contract, UI, migration, and performance tests.
9. **Approval:** Apply automated policy gates or manual production approval.
10. **Production deployment:** Rolling, blue-green, or canary release.
11. **Verification:** Smoke tests, health checks, metrics, alerts, and rollback.

```text
Commit -> Validate -> Build -> Test -> Scan -> Publish
       -> Deploy -> Verify -> Promote or Roll Back
```

---

## 7. Building and managing container images during CI

1. Keep the `Dockerfile` with the source code.
2. Use a reproducible, multi-stage build.
3. Authenticate to the registry with short-lived credentials.
4. Build from a specific commit.
5. Scan the filesystem, dependencies, and final image.
6. Generate an SBOM.
7. Tag the image with a semantic version and Git SHA.
8. Push it to ECR, ACR, GHCR, Artifactory, or another trusted registry.
9. Sign the image.
10. Deploy by immutable digest.

```bash
IMAGE="123456789012.dkr.ecr.ap-south-1.amazonaws.com/payment-api"
GIT_SHA="$(git rev-parse --short=12 HEAD)"
VERSION="1.5.0"

docker build --pull \
  --tag "${IMAGE}:${VERSION}" \
  --tag "${IMAGE}:${GIT_SHA}" .

docker push "${IMAGE}:${VERSION}"
docker push "${IMAGE}:${GIT_SHA}"
```

Production practices:

- Avoid the `latest` tag.
- Make published tags immutable.
- Promote the same digest from development to production instead of rebuilding.
- Pin trusted base images and dependencies.
- Run as a non-root user.
- Keep secrets out of build arguments and image layers.
- Enable vulnerability scanning and registry lifecycle policies.

---

## 8. Accessing an S3 bucket from another AWS Region

Yes. A workload in one AWS Region can access an S3 bucket in another Region when IAM, bucket-policy, KMS, and network requirements permit it.

`us-south-1` is not a standard AWS commercial Region identifier. The intended Region may be `ap-south-1`, `us-east-1`, `us-east-2`, `us-west-1`, or `us-west-2`.

```bash
aws s3 cp report.log s3://company-log-bucket/logs/report.log \
  --region ap-south-1
```

Cross-Region access may increase latency and data-transfer cost. For a multi-Region application, consider S3 Cross-Region Replication and S3 Multi-Region Access Points. Replication requires separate buckets and explicit replication rules.

---

## 9. Terraform: VPC, subnet, EC2, and S3

This example creates a VPC, public subnet, internet gateway, route table, security group, EC2 instance, and private encrypted S3 bucket.

```hcl
terraform {
  required_version = ">= 1.6.0"

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

variable "admin_cidr" {
  description = "Trusted address allowed to use SSH"
  type        = string
  default     = "203.0.113.10/32"
}

variable "bucket_name" {
  description = "Globally unique S3 bucket name"
  type        = string
}

data "aws_availability_zones" "available" {
  state = "available"
}

data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-2023.*-x86_64"]
  }

  filter {
    name   = "architecture"
    values = ["x86_64"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

resource "aws_vpc" "main" {
  cidr_block           = "10.10.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = { Name = "interview-vpc" }
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.10.1.0/24"
  availability_zone       = data.aws_availability_zones.available.names[0]
  map_public_ip_on_launch = true

  tags = { Name = "interview-public-subnet" }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "interview-igw" }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

resource "aws_security_group" "ec2" {
  name   = "interview-ec2-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    description = "SSH from trusted administrator address"
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

resource "aws_instance" "server" {
  ami                         = data.aws_ami.amazon_linux.id
  instance_type               = "t3.micro"
  subnet_id                   = aws_subnet.public.id
  vpc_security_group_ids      = [aws_security_group.ec2.id]
  associate_public_ip_address = true

  metadata_options {
    http_endpoint = "enabled"
    http_tokens   = "required"
  }

  root_block_device {
    encrypted   = true
    volume_type = "gp3"
    volume_size = 10
  }

  tags = { Name = "interview-ec2" }
}

resource "aws_s3_bucket" "logs" {
  bucket = var.bucket_name
}

resource "aws_s3_bucket_public_access_block" "logs" {
  bucket = aws_s3_bucket.logs.id

  block_public_acls       = true
  ignore_public_acls      = true
  block_public_policy     = true
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

output "vpc_id" {
  value = aws_vpc.main.id
}

output "ec2_public_ip" {
  value = aws_instance.server.public_ip
}

output "s3_bucket_name" {
  value = aws_s3_bucket.logs.bucket
}
```

Run:

```bash
terraform fmt
terraform init
terraform validate
terraform plan -var="bucket_name=globally-unique-bucket-name"
terraform apply -var="bucket_name=globally-unique-bucket-name"
```

---

## 10. Purpose of CNI in Kubernetes

CNI means **Container Network Interface**. Kubernetes uses a compatible CNI plugin to implement Pod networking.

Typical responsibilities include:

- Creating or attaching a network interface for a Pod
- Assigning the Pod IP address
- Connecting the Pod network namespace to the host network
- Configuring routes
- Cleaning up network resources when the Pod is deleted
- Supporting Pod-to-Pod communication
- Enforcing NetworkPolicy when supported by the chosen plugin

Examples include Amazon VPC CNI, Calico, Cilium, Flannel, Weave Net, and Antrea.

```text
Pod scheduled -> Runtime creates sandbox -> Runtime invokes CNI
              -> Interface and IP configured -> Pod becomes network-ready
```

---

## 11. Automating log uploads, CPU monitoring, and alarms

Treat log archival and CPU alarms as separate automation paths:

```text
Application logs -> Scheduled archive script -> S3
EC2 CPU metric -> CloudWatch alarm -> SNS -> Email/notification
```

### Log upload

Use the script from Question 1 and schedule it through `systemd`, cron, or Systems Manager. Grant the instance role access only to its S3 log prefix.

### SNS notification

```bash
TOPIC_ARN="$(aws sns create-topic \
  --name high-cpu-alerts \
  --query TopicArn \
  --output text)"

aws sns subscribe \
  --topic-arn "${TOPIC_ARN}" \
  --protocol email \
  --notification-endpoint devops-team@example.com
```

The email recipient must confirm the subscription.

### CPU alarm

```bash
INSTANCE_ID="i-0123456789abcdef0"

aws cloudwatch put-metric-alarm \
  --alarm-name "EC2-High-CPU-${INSTANCE_ID}" \
  --alarm-description "CPU above 80 percent for 10 minutes" \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions "Name=InstanceId,Value=${INSTANCE_ID}" \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --datapoints-to-alarm 2 \
  --threshold 80 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --treat-missing-data missing \
  --alarm-actions "${TOPIC_ARN}" \
  --ok-actions "${TOPIC_ARN}"
```

Use a separate alarm for failed log uploads. The script can publish a custom `LogUploadFailure` metric or write to CloudWatch Logs and use a metric filter.

---

## 12. Can a NAT gateway be created in a private subnet?

### Public NAT gateway

A public NAT gateway should be created in a **public subnet**, associated with an Elastic IP address, and have a route to an internet gateway. Private subnets route outbound internet traffic to it.

```text
Private subnet: 0.0.0.0/0 -> NAT Gateway
Public subnet:  0.0.0.0/0 -> Internet Gateway
```

Putting a public NAT gateway in a subnet without an internet-gateway route will not provide outbound internet connectivity.

### Private NAT gateway

AWS also supports a private NAT gateway for translated traffic to other VPCs or on-premises networks through a Transit Gateway or virtual private gateway. It has no Elastic IP and is not used to provide internet access.

For production resiliency, deploy one NAT gateway per Availability Zone and route private workloads to the NAT gateway in their own Availability Zone.

---

## 13. Configuring Kubernetes Cluster Autoscaler

Cluster Autoscaler adjusts the number of worker nodes:

- It scales out when Pods cannot be scheduled because the cluster lacks capacity.
- It scales in when nodes are underutilized and workloads can be moved safely.

### Prerequisites

1. Scalable node groups or Auto Scaling groups
2. Minimum, desired, and maximum capacity
3. Correct cloud-provider IAM permissions
4. CPU and memory requests on application Pods
5. Auto-discovery tags or explicit node-group configuration
6. Pod Disruption Budgets that permit safe movement
7. A Cluster Autoscaler version compatible with the Kubernetes version

EKS Auto Scaling group discovery tags:

```text
k8s.io/cluster-autoscaler/enabled = true
k8s.io/cluster-autoscaler/CLUSTER_NAME = owned
```

Use EKS Pod Identity or IAM Roles for Service Accounts instead of static AWS credentials.

### Helm installation

```bash
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm repo update

helm upgrade --install cluster-autoscaler \
  autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName=my-eks-cluster \
  --set awsRegion=ap-south-1 \
  --set rbac.serviceAccount.create=false \
  --set rbac.serviceAccount.name=cluster-autoscaler \
  --set extraArgs.balance-similar-node-groups=true \
  --set extraArgs.expander=least-waste
```

### Verification

```bash
kubectl get deployment cluster-autoscaler -n kube-system
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-cluster-autoscaler
kubectl logs -n kube-system deployment/cluster-autoscaler --follow
```

### Common causes of failure

- Pods have no resource requests.
- The node group has reached maximum capacity.
- IAM permissions or discovery tags are missing.
- Taints, tolerations, affinity, or labels cannot be satisfied.
- A volume is restricted to another Availability Zone.
- A PDB prevents scale-in.
- Adding a node cannot resolve the scheduling constraint.

### Cluster Autoscaler and HPA

```text
Horizontal Pod Autoscaler -> changes the number of Pods
Cluster Autoscaler        -> changes the number of worker nodes
```

They are often used together: HPA creates Pods, pending Pods trigger Cluster Autoscaler, and the new nodes provide scheduling capacity.

---

## Quick Interview Summary

- Use an EC2 IAM role instead of static keys for S3 uploads.
- Use `ResourceQuota` and `LimitRange` for namespace-level controls.
- Three-tier architecture separates presentation, application, and data layers.
- Upgrade nodes through cordon, drain, replacement, and validation.
- Administrator access can still be limited by boundaries, SCPs, resource policies, endpoint policies, KMS policies, and explicit denies.
- CI/CD covers validation, build, test, security, artifact publishing, deployment, verification, and rollback.
- Build an image once and promote the same immutable digest.
- S3 supports authorized cross-Region access.
- CNI provides Pod networking.
- A public NAT gateway belongs in a public subnet.
- HPA scales Pods, while Cluster Autoscaler scales worker nodes.


# DevOps Consultant Interview Questions and Answers

> **Experience level:** 7 years, DevOps Consultant  
> These answers are designed for senior-level interviews. They include architectural decisions, trade-offs, operational controls, and practical examples.

## 1. Migrating a monolithic application with a local file system to AWS

The correct storage depends on the application's access pattern. I would first identify whether the existing data is shared, block-based, object-based, temporary, or latency-sensitive.

- **Amazon EFS:** Preferred when multiple EC2 instances, containers, or Kubernetes Pods must concurrently mount the same POSIX-compatible shared file system. It supports `ReadWriteMany`-style shared access and avoids tying data to one server.
- **Amazon EBS:** Use when one EC2 instance or one availability-zone-bound workload needs low-latency block storage. Multi-Attach is specialized and is not a general shared-file-system replacement.
- **Amazon FSx:** Use when the application requires Windows SMB, NetApp ONTAP features, Lustre, or OpenZFS compatibility.
- **Amazon S3:** Use for objects such as reports, documents, images, archives, and backups after modifying the application to use object APIs. S3 is not a drop-in POSIX file system.
- **Instance store:** Use only for disposable cache, scratch, or temporary processing because data does not provide durable application storage.

For the initial lift-and-shift, I would commonly use EFS for a shared Linux file system, migrate the files with AWS DataSync, validate permissions and throughput, and later refactor suitable content into S3. In Kubernetes, persistent data should be represented by a `PersistentVolume` and consumed through a `PersistentVolumeClaim`; the volume lifecycle is independent of an individual Pod. citeturn4search7turn4search8

## 2. Storing monolithic application configuration in AWS

I separate configuration by sensitivity and lifecycle:

- **AWS Systems Manager Parameter Store:** Non-secret runtime configuration, feature settings, URLs, and environment values.
- **AWS Secrets Manager:** Database passwords, API credentials, tokens, and certificates requiring controlled access and rotation.
- **AWS AppConfig:** Validated configuration rollout, feature flags, gradual deployment, and automatic rollback.
- **S3:** Larger versioned configuration documents, with encryption and a restrictive bucket policy.
- **Kubernetes ConfigMaps and Secrets:** Delivery into Pods, preferably synchronized from an external secret manager rather than treating Kubernetes Secrets as the primary source of truth.
- **Infrastructure as Code:** Resource configuration belongs in Terraform or CloudFormation and source control, not in manually maintained console settings.

Controls should include KMS encryption, least-privilege IAM, versioning, audit logs, secret rotation, environment separation, schema validation, and no secrets in Git, container images, user data, or CI logs. Secrets Manager supports encrypted credential storage and automatic rotation; AWS DMS can also consume Secrets Manager credentials for endpoints. citeturn4search1turn4search4

## 3. Observability required for an application

Observability should cover **metrics, logs, traces, events, alerting, response, and automated remediation**.

### Monitoring

Track the four golden signals: latency, traffic, errors, and saturation. Add business KPIs, dependency health, JVM or runtime metrics, CPU, memory, disk, queue depth, database connections, replication lag, and Kubernetes health. Define SLIs, SLOs, and an error budget.

### Logging

Use structured JSON logs with timestamp, severity, service, environment, correlation ID, trace ID, request ID, and safe error details. Centralize logs in CloudWatch Logs, OpenSearch, or another platform. Apply retention, encryption, access control, and redaction of credentials and personal data. Cluster-level logging needs storage whose lifecycle is independent of Pods and nodes. citeturn4search10

### Distributed tracing

Instrument calls with OpenTelemetry and export traces to AWS X-Ray or another backend. Traces reveal latency across the load balancer, application, cache, queue, and database.

### Alerting

Alert on symptoms and SLO impact rather than every raw metric. Use severity levels, deduplication, inhibition, actionable descriptions, runbook links, ownership, and escalation through SNS, PagerDuty, or an equivalent system.

### Incident response and remediation

Create runbooks for diagnosis, rollback, failover, capacity expansion, credential rotation, and dependency isolation. Automate safe actions with EventBridge, Lambda, Systems Manager Automation, Kubernetes operators, or Auto Scaling. Every remediation must have guardrails, idempotency, auditability, limits, and a manual override.

## 4. Logging without installing Filebeat on worker nodes

Possible options are:

1. **Sidecar collector:** Run Fluent Bit, Fluentd, or an OpenTelemetry Collector beside the application container and share a log volume.
2. **DaemonSet collector:** Deploy Fluent Bit or the OpenTelemetry Collector as a Kubernetes DaemonSet. This is not a host-installed package, although it still runs once per eligible node and may mount container log directories.
3. **Write to stdout and stderr:** Let the container runtime capture logs, then use a managed or platform-provided cluster logging integration.
4. **Direct application export:** Send structured logs or telemetry from the application through an SDK or OpenTelemetry protocol endpoint.
5. **Serverless logging router:** On AWS Fargate or ECS, use supported log drivers or FireLens where applicable.

Preferred answer: applications write to stdout/stderr, and a centrally managed Fluent Bit or OpenTelemetry DaemonSet forwards logs. If node-level access is prohibited, use a sidecar, application-level OTLP export, or the managed platform integration. Kubernetes identifies stdout/stderr as the simplest widely adopted container logging approach, but a complete cluster logging solution needs an independent backend. citeturn4search10

## 5. Security controls for a three-tier architecture

Apply defence in depth:

- Put only the internet-facing load balancer in public subnets. Keep application and database tiers in private or isolated subnets across multiple Availability Zones.
- Allow internet traffic only through CloudFront, AWS WAF, Shield, and an ALB. Do not assign public IPs to application or database instances.
- Use security-group references: internet or CloudFront to ALB on `443`, ALB to application port, and application security group to database port.
- Use TLS 1.2 or later externally and internally where supported, ACM-managed certificates, secure cookies, HSTS, and modern cipher policies.
- Encrypt EBS, EFS, S3, RDS, backups, logs, queues, and secrets with appropriate KMS keys.
- Use IAM roles and temporary credentials, least privilege, SCPs, permission boundaries, MFA, and separate production accounts.
- Store secrets in Secrets Manager and rotate them.
- Enable CloudTrail, AWS Config, VPC Flow Logs, WAF logs, load-balancer logs, GuardDuty, Security Hub, and centralized alerting.
- Protect the application against OWASP risks through validation, authentication, authorization, rate limiting, dependency scanning, and patching.
- Use RDS Multi-AZ, backups, tested recovery, immutable deployments, and restricted administrative access through Systems Manager.

## 6. Database migration and data synchronization

I use a staged process:

1. Discover the source engine, size, extensions, stored procedures, character sets, dependencies, RPO/RTO, and acceptable downtime.
2. Select the target engine. For a homogeneous migration, preserve the engine where practical. For a heterogeneous migration, assess and convert the schema first.
3. Establish encrypted private connectivity using Direct Connect, VPN, or controlled VPC connectivity.
4. Take an initial full load using native backup/restore or AWS DMS.
5. Enable ongoing change replication using DMS CDC, native logical replication, binlog replication, or database-specific tooling.
6. Monitor latency, failed tables, validation errors, target capacity, and replication lag.
7. Rehearse the cutover, freeze writes or use a short write drain, wait for replication lag to reach zero, validate row counts and checksums, and switch the application endpoint.
8. Keep a rollback window and do not destroy the source until business validation is complete.

AWS DMS supports migrations between same or different database engines and ongoing change replication. Its source and target endpoints define the connection and data-store information. citeturn4search2turn4search3

## 7. If a database Pod goes down, is the data affected?

It depends on where the database writes data:

- With only the container writable layer or `emptyDir`, the data is ephemeral and can be lost when the Pod is replaced or moved.
- With a correctly configured PVC backed by EBS, EFS, or another durable CSI volume, a replacement Pod can reattach or remount the persistent storage.
- A PVC alone does not guarantee database consistency, high availability, or disaster recovery. Use a StatefulSet, stable identity, replication, quorum, anti-affinity, topology controls, regular snapshots, transaction logs, and tested restores.
- If an EBS volume is zonal, the replacement must normally run in the compatible Availability Zone unless the recovery design creates or restores storage elsewhere.
- Abrupt failure can lose acknowledged data if the engine, storage, or replication configuration does not meet the required durability guarantees.

Persistent volumes have a lifecycle independent of an individual Pod, whereas ephemeral volumes are tied to the Pod lifecycle. citeturn4search7turn4search11

For critical production databases, I prefer a managed database such as RDS or Aurora unless there is a compelling requirement to operate the database in Kubernetes.

## 8. Sticky-session data when a Pod goes down

If session state exists only in a Pod's memory or local file system, that session can be lost when the Pod terminates. Even if load-balancer stickiness directs subsequent requests to the same target, it cannot recover in-memory state from a failed target.

The correct long-term design is a stateless application with session state externalized to Redis, DynamoDB, or a database. Use a short TTL, secure random session IDs, encryption in transit, access restrictions, and explicit invalidation. Sticky sessions may still be used temporarily for legacy compatibility, but they should not be the durability mechanism. AWS defines sticky sessions as repeatedly routing a client to one destination, often because that destination maintains local session state. citeturn4search19turn4search24

## 9. Alternative to sticky sessions

Use a distributed session store, commonly **Amazon ElastiCache for Valkey or Redis OSS**. Every application replica reads and writes session state through the same logical cache, so the load balancer can distribute requests normally.

Design considerations:

- Multi-AZ replication and automatic failover
- Encryption in transit and at rest
- Authentication and security groups
- Session TTL and eviction policy
- Capacity and connection limits
- Graceful behavior if the cache is unavailable
- Avoid storing highly durable business records only in the cache

ElastiCache provides managed distributed in-memory caches using Valkey, Redis OSS, or Memcached. citeturn4search20turn4search23

## 10. Load-balancer causes of sticky-session problems

Common causes include:

- Stickiness is disabled or enabled on the wrong listener rule or target group.
- Cookie duration is too short, or the cookie expires.
- The application cookie name, path, domain, `Secure`, or `SameSite` attributes are wrong.
- A proxy, CDN, browser policy, or API client removes or does not return cookies.
- The target becomes unhealthy, is deregistered, scales in, or is replaced. The load balancer must then select another healthy target.
- Different load balancers or target groups do not share the same stickiness context.
- Long deregistration delay, poor readiness checks, or deployment termination causes requests to reach an unready target.
- WebSocket or long-lived connection behavior is being confused with cookie-based stickiness.
- Cross-zone or target-group routing configuration does not match the intended design.

Troubleshoot with ALB access logs, browser headers, target health, cookie inspection, deployment events, and correlation IDs. AWS supports load-balancer-generated and application-generated sticky-session cookies, plus target-group stickiness for cases such as blue-green deployments. citeturn4search19turn4search24

## 11. Kubernetes clusters in multiple Regions

Yes. A single conventional Kubernetes control plane is normally regional, so I create **one independent cluster per Region**, not one stretched cluster across distant Regions.

Management approach:

- Provision clusters with reusable Terraform modules and GitOps.
- Use a multi-account landing zone and consistent IAM, network, policy, add-on, logging, and tagging baselines.
- Use Argo CD, Flux, or a fleet-management platform to deploy desired state to all clusters.
- Use Route 53, Global Accelerator, or CloudFront for health-based, latency-based, or failover routing.
- Replicate container images and required data across Regions.
- Select active-active or active-passive based on consistency, cost, and operational complexity.
- Make applications stateless where possible and explicitly design database replication and conflict handling.
- Centralize observability and security findings while preserving regional independence.
- Test regional failover, DNS TTLs, capacity, secrets, certificates, and recovery runbooks.

Clusters should remain operationally isolated so a control-plane or regional problem does not propagate fleet-wide.

## 12. Onboarding a trading application to AWS

A trading platform requires explicit latency, availability, consistency, security, audit, and regulatory requirements.

### Availability

- Multi-AZ for every critical regional component
- Redundant load balancers and stateless services
- RDS/Aurora Multi-AZ or a database architecture aligned with transaction guarantees
- Durable messaging and idempotent order processing
- Cross-Region disaster recovery with defined RTO/RPO
- Health-based routing, tested failover, graceful degradation, and dependency circuit breakers

### Scalability and performance

- Horizontal application scaling from CPU, latency, queue depth, or business metrics
- Predictable baseline capacity for market-open spikes
- Load testing with realistic order flow
- Redis caching only where consistency rules permit
- Back-pressure, rate limits, bounded queues, connection pooling, and performance budgets
- Measure tail latency such as p95, p99, and p99.9, not only averages

### Security

- Private subnets, WAF, DDoS protection, network segmentation, and restricted egress
- TLS, encryption at rest, KMS-based key controls, and secret rotation
- Strong customer and operator authentication, least privilege, MFA, and separation of duties
- Immutable artifacts, signed images, SBOMs, vulnerability management, and policy gates
- Tamper-evident audit trails, synchronized time, data retention, fraud monitoring, and incident response

### Operational governance

Define SLOs, change controls, reconciliation, duplicate-order prevention, disaster-recovery drills, capacity tests, chaos tests, and compliance evidence before production approval.

## 13. Regular backup of an entire Kubernetes cluster

An entire-cluster backup has multiple independent parts:

1. **Kubernetes objects:** Back up namespaced and cluster-scoped resources with Velero or an equivalent tool.
2. **Persistent data:** Use CSI snapshots and application-consistent database backup procedures. A manifest backup alone does not back up volume contents.
3. **Managed control plane:** In EKS, AWS operates the control-plane data store, but customers still need workload-resource and application-data recovery procedures.
4. **Infrastructure:** Keep VPCs, EKS configuration, node groups, IAM, add-ons, and supporting services in Terraform or CloudFormation.
5. **External dependencies:** Back up RDS, DynamoDB, S3 configuration, secrets, certificates, DNS, registries, and GitOps repositories according to their service-specific mechanisms.
6. **Off-site protection:** Encrypt backups, use immutable retention where required, and copy critical backups to another account or Region.
7. **Restore tests:** Regularly restore into an isolated cluster and measure RPO, RTO, integrity, and application startup order.

A sample Velero schedule is:

```bash
velero schedule create daily-cluster-backup \
  --schedule="0 1 * * *" \
  --include-namespaces='*' \
  --ttl 720h0m0s
```

PersistentVolumes are separate storage resources with a lifecycle independent of Pods, so the backup design must include both Kubernetes metadata and underlying persistent data. citeturn4search7turn4search11

## 14. Python script to list running EC2 instances tagged PROD

The script uses Boto3's default credential chain. Run it with an IAM role or federated session that allows `ec2:DescribeInstances`. Avoid static credentials in the source code.

```python
#!/usr/bin/env python3

import argparse
import sys
from typing import Iterator

import boto3
from botocore.exceptions import BotoCoreError, ClientError


def running_prod_instances(region: str, tag_key: str, tag_value: str) -> Iterator[dict]:
    ec2 = boto3.client("ec2", region_name=region)
    paginator = ec2.get_paginator("describe_instances")

    filters = [
        {"Name": "instance-state-name", "Values": ["running"]},
        {"Name": f"tag:{tag_key}", "Values": [tag_value]},
    ]

    for page in paginator.paginate(Filters=filters):
        for reservation in page.get("Reservations", []):
            for instance in reservation.get("Instances", []):
                tags = {tag["Key"]: tag["Value"] for tag in instance.get("Tags", [])}
                yield {
                    "InstanceId": instance["InstanceId"],
                    "Name": tags.get("Name", ""),
                    "InstanceType": instance["InstanceType"],
                    "PrivateIp": instance.get("PrivateIpAddress", ""),
                    "AvailabilityZone": instance["Placement"]["AvailabilityZone"],
                }


def main() -> int:
    parser = argparse.ArgumentParser(description="List running PROD EC2 instances")
    parser.add_argument("--region", required=True, help="AWS Region, for example ap-south-1")
    parser.add_argument("--tag-key", default="Environment")
    parser.add_argument("--tag-value", default="PROD")
    args = parser.parse_args()

    try:
        instances = list(running_prod_instances(args.region, args.tag_key, args.tag_value))
    except (BotoCoreError, ClientError) as exc:
        print(f"AWS API error: {exc}", file=sys.stderr)
        return 1

    if not instances:
        print("No matching running instances found.")
        return 0

    print(f"{'InstanceId':<22} {'Name':<25} {'Type':<12} {'PrivateIp':<16} AZ")
    for item in instances:
        print(
            f"{item['InstanceId']:<22} {item['Name']:<25} "
            f"{item['InstanceType']:<12} {item['PrivateIp']:<16} "
            f"{item['AvailabilityZone']}"
        )
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

Usage:

```bash
python3 list_prod_instances.py --region ap-south-1
```

If the actual convention is `Stage=prod` or `Environment=Production`, pass the matching key and value.

## 15. Creating and connecting 200 EC2 instances across 10 AWS accounts

I would not create or connect them manually.

### Provisioning

- Use AWS Organizations with a dedicated deployment role in each account.
- Use Terraform modules with one provider alias or workspace per account, or use CloudFormation StackSets.
- Run account deployments through a controlled CI/CD pipeline with concurrency limits, approvals, remote state isolation, tagging, and policy checks.
- Prefer Auto Scaling groups when the 20 instances in each account form a homogeneous fleet.

### Network connectivity

Use **AWS Transit Gateway** as the hub for private connectivity among account VPCs. Share it through AWS Resource Access Manager, attach each VPC, configure route propagation or controlled static routes, and make sure all VPC CIDRs are non-overlapping. For a very large global design, evaluate AWS Cloud WAN. VPC peering becomes difficult to operate at this scale because it is non-transitive and creates a mesh.

### Administrative connectivity

Use **AWS Systems Manager Session Manager**, Run Command, and State Manager for administration and automation. This removes the need to expose SSH port `22`, distribute SSH keys, or operate bastion hosts. Network reachability and management access are separate concerns: Transit Gateway connects networks, while Systems Manager provides managed administrative sessions.

## 16. Connecting to a database in a private subnet without NAT or a bastion

Options include:

- **Systems Manager Session Manager port forwarding:** Connect through an SSM-managed EC2 instance that has private reachability to the database. It needs SSM connectivity through interface VPC endpoints when there is no NAT.
- **AWS Client VPN:** Provide authorized users with routed private access to the VPC.
- **Site-to-Site VPN or Direct Connect:** Connect the corporate network to AWS privately.
- **Transit Gateway or VPC peering:** Connect an application or tooling VPC to the database VPC.
- **PrivateLink:** Expose a supported service privately across VPCs or accounts, often through an endpoint service design. Direct database applicability depends on the architecture.
- **RDS Data API:** For supported database configurations, call the database through the managed API rather than a normal client TCP connection.
- **In-VPC tooling:** Run migrations through CodeBuild, ECS tasks, Lambda, or Kubernetes jobs in subnets with database reachability.

A VPC endpoint is not a generic direct endpoint for every relational database. The selected solution must also permit DNS resolution, security-group traffic, NACL traffic, authentication, TLS, and database authorization.

## 17. OSI model

The OSI model has seven conceptual layers:

1. **Physical:** Transmits bits through cables, radio, optics, NIC signalling, and physical interfaces.
2. **Data Link:** Frames, MAC addresses, switching, VLANs, ARP-related local delivery, and link-level error detection.
3. **Network:** IP addressing, routing, subnets, routers, ICMP, and packet forwarding.
4. **Transport:** End-to-end TCP or UDP communication, ports, reliability, flow control, and retransmission.
5. **Session:** Establishes, manages, and terminates logical communication sessions.
6. **Presentation:** Data representation, encoding, serialization, compression, and encryption concepts.
7. **Application:** User-facing protocols such as HTTP, DNS, SMTP, and SSH.

Troubleshooting should move systematically through layers. For example:

```text
Physical/link -> IP and route -> TCP port -> TLS/session -> HTTP/application
```

In practical cloud troubleshooting, security groups, NACLs, routes, DNS, load balancers, TLS certificates, application listeners, and process health map across multiple OSI layers.

## 18. Difference between a directory and a mount

A **directory** is a file-system entry used to organize files, for example `/var/log/app`.

A **mount** attaches a filesystem or storage source to a directory called the mount point. After mounting, accessing the directory shows the mounted filesystem's contents. The directory existed before the mount, but its previous contents are hidden until the filesystem is unmounted.

```bash
sudo mkdir -p /data
sudo mount /dev/nvme1n1p1 /data
findmnt /data
df -hT /data
```

Examples of mount sources include an EBS filesystem, EFS through NFS, a network share, `tmpfs`, or a bind mount. A mount exists in the active mount namespace and may be made persistent through `/etc/fstab` or a systemd mount unit.

## 19. Terraform local values versus input variables

### Input variables

- Form the configurable input interface of a module.
- Can be supplied by callers, environment-specific variable files, the CLI, environment variables, or HCP Terraform.
- Support types, defaults, descriptions, sensitivity, and validation.
- Referenced as `var.name`.

```hcl
variable "environment" {
  type        = string
  description = "Deployment environment"

  validation {
    condition     = contains(["dev", "test", "prod"], var.environment)
    error_message = "Environment must be dev, test, or prod."
  }
}
```

### Local values

- Are internal computed or reusable expressions within a module.
- Cannot be directly overridden by a module caller.
- Can combine variables, resource attributes, functions, and other locals.
- Referenced as `local.name`.

```hcl
locals {
  name_prefix = "payments-${var.environment}"
  common_tags = {
    Application = "payments"
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}
```

Use variables for external customization and locals for internal derivation, naming, transformations, and avoiding repetition. HashiCorp documents variables as module inputs and locals as reusable module-scoped expressions. citeturn4search13turn4search14turn4search18

## 20. Preventing console modification and automating drift checks

Terraform cannot by itself stop an authorized user from changing a resource in the AWS console. Prevention and detection require governance controls.

### Prevent unauthorized manual changes

- Use AWS IAM and SCPs to deny mutating production APIs except through approved pipeline roles.
- Grant people read-only production access and use time-bound, audited break-glass roles for emergencies.
- Apply tag-based controls carefully and protect the tags used for authorization.
- Use AWS Config rules and Security Hub controls for policy compliance.
- Enable CloudTrail and alert on sensitive changes made outside approved roles.
- Require all intended changes through pull requests and CI/CD.

### Detect Terraform drift

Run a scheduled pipeline:

```bash
set +e
terraform init -input=false
terraform plan -detailed-exitcode -input=false -lock-timeout=5m -out=tfplan
status=$?
set -e

case "$status" in
  0) echo "No drift or pending changes" ;;
  1) echo "Terraform plan failed"; exit 1 ;;
  2) echo "Drift or pending changes detected"; exit 2 ;;
esac
```

Schedule this daily or hourly using the CI system, use read-only planning credentials where practical, archive the plan output securely, and notify the resource owner. A normal plan refreshes provider data in memory and compares configuration, state, and actual infrastructure. Refresh-only mode can update state to match infrastructure, but it should not be run automatically as a fix because it can accept unauthorized changes into state. citeturn4search16turn4search17

### Remediation policy

Do not automatically run `terraform apply` for every detected drift. Classify the change first:

1. Unauthorized or unsafe change: revert through the controlled pipeline.
2. Approved emergency change: update the code, review it, and reconcile state.
3. Benign provider-managed field: model it correctly or narrowly use `ignore_changes` only after understanding ownership.
4. Security-critical drift: isolate, alert, and follow incident-management procedures.

`prevent_destroy` protects against Terraform-driven destruction of a resource, but it does not prevent console edits or deletion outside Terraform.

---

## Senior-level closing summary

A strong DevOps Consultant answer should explain not just the service name, but also the decision criteria, failure modes, security controls, automation method, observability, and recovery process. The recurring principles are: externalize state, use managed services where appropriate, remove static credentials, design across failure domains, codify infrastructure, detect drift, and test restoration and failover regularly.
