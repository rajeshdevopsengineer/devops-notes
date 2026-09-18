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
