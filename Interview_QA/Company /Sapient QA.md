# DevOps and AWS Interview Questions

## Company: Sapient

## Experience: 6 Years

This guide covers AWS, Route 53, VPC networking, EKS, storage, Terraform, Kubernetes administration, and workload access to S3.

> **Important:** Replace placeholders such as `example.com`, subnet counts, AWS Regions, and service names with details from your actual project. Do not claim services or architectures you have not worked with.

---

# AWS Experience and DNS

## 1. Which AWS services have you used?

### Interview-ready answer template

> “I have worked with AWS services across compute, networking, containers, storage, security, monitoring, and infrastructure automation.
>
> For compute and containers, I have worked with services such as EC2, Auto Scaling, ECR, and EKS.
>
> For networking, I have worked with VPC, public and private subnets, route tables, Internet Gateway, NAT Gateway, security groups, Network ACLs, Route 53, and Elastic Load Balancing.
>
> For storage and databases, I have used services such as S3, EBS, EFS, RDS, and AWS Backup, based on project requirements.
>
> For identity and security, I have worked with IAM roles and policies, KMS, Secrets Manager, and ACM.
>
> For monitoring and auditing, I have used CloudWatch and CloudTrail.
>
> I have provisioned AWS resources through Terraform and managed containerized application deployments through Kubernetes manifests and CI/CD pipelines.”

### Explain services by responsibility

| Area | Services you can mention if actually used |
|---|---|
| Compute | EC2, Auto Scaling, Lambda |
| Containers | EKS, ECS, ECR |
| Networking | VPC, Route 53, ELB, NAT Gateway, Internet Gateway |
| Storage | S3, EBS, EFS, FSx |
| Databases | RDS, Aurora, DynamoDB, ElastiCache |
| Security | IAM, KMS, Secrets Manager, ACM, WAF |
| Monitoring | CloudWatch, CloudTrail, AWS Config |
| Messaging | SNS, SQS, EventBridge |
| DevOps | CodeBuild, CodeDeploy, CodePipeline |
| Infrastructure as Code | Terraform, CloudFormation |

### Strong answer pattern

For every important service, describe:

1. Why it was used.
2. What you personally configured.
3. How it was secured.
4. How it was monitored.
5. One problem you solved.

### Example

> “We used EKS to run containerized applications. I worked on node groups, Kubernetes deployments, ingress configuration, EBS-backed volumes, IAM access for workloads, cluster monitoring, and deployment troubleshooting.”

> **Interview tip:** Do not answer only with a long list of AWS service names. Explain your responsibility and one practical use case.

---

## 2. What is the difference between an A record and a CNAME record in Route 53?

### A record

An A record maps a DNS name to one or more IPv4 addresses.

Example:

```text
api.example.com -> 192.0.2.10
```

### CNAME record

A CNAME, or Canonical Name, maps one DNS name to another DNS name.

Example:

```text
www.example.com -> application.example.net
```

The client or DNS resolver must then resolve the target name.

### Comparison

| Feature | A record | CNAME record |
|---|---|---|
| Target | IPv4 address | Another DNS name |
| Zone apex | Supported | Not supported |
| Common use | Map a hostname to an IP address | Provide an alias for another hostname |
| Example | `api.example.com -> 192.0.2.10` | `www.example.com -> app.example.net` |

### Route 53 alias records

Route 53 provides alias records as an AWS-specific DNS extension.

An alias record can route traffic to supported AWS resources such as:

- Application Load Balancer.
- Network Load Balancer.
- CloudFront.
- API Gateway.
- S3 static website.
- Another Route 53 record.

Unlike a CNAME, an alias record can be created at the zone apex.

Example:

```text
example.com -> Application Load Balancer
```

### Interview-ready answer

> “An A record returns an IPv4 address, while a CNAME points one DNS name to another DNS name. A CNAME cannot be used at the zone apex. In Route 53, I normally use an A-type alias record for supported AWS resources such as an Application Load Balancer.”

**Reference:** [Route 53 alias and non-alias records](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-choosing-alias-non-alias.html)

---

## 3. What is the purpose of DNS in your project?

DNS provides a human-readable and stable name for an application or service.

Without DNS, users and applications would need to use infrastructure addresses directly.

### Typical project uses

- Map a public application name to a load balancer.
- Provide private names for internal applications.
- Separate application names by environment.
- Support controlled traffic changes.
- Validate domain ownership for certificates.
- Route requests based on failover, latency, weight, or geography where required.
- Allow applications to use stable service names while infrastructure changes behind them.

### Example

```text
api.example.com
        |
        v
Route 53 alias record
        |
        v
Application Load Balancer
        |
        v
EKS Ingress or Kubernetes Service
        |
        v
Application Pods
```

### Public and private DNS

| DNS type | Use case |
|---|---|
| Public hosted zone | Names that must resolve from the internet. |
| Private hosted zone | Internal names resolved from associated VPCs. |

### Interview-ready answer

> “We used DNS to provide stable and readable endpoints for applications. Route 53 mapped application hostnames to load balancers, while Kubernetes handled service discovery inside the cluster.”

---

## 4. What was your domain name, and how did you connect it to the service?

### Safe interview template

Do not disclose a confidential production domain unless you are permitted to do so.

> “I cannot disclose the customer’s actual domain, but the structure was similar to `api.example.com`. The domain was managed through a Route 53 hosted zone. We created an alias record pointing the application hostname to an AWS load balancer. The load balancer forwarded traffic to the Kubernetes ingress or service, which routed requests to the application Pods.”

### Example request flow

```text
User
  |
  | HTTPS request to api.example.com
  v
Route 53 hosted zone
  |
  | Alias record
  v
AWS Application Load Balancer
  |
  | Host/path routing
  v
EKS Ingress or Kubernetes Service
  |
  v
Application Pods
```

### Required components

1. Registered domain.
2. Route 53 hosted zone or another authoritative DNS provider.
3. DNS record for the application hostname.
4. AWS load balancer.
5. ACM certificate for HTTPS.
6. Listener and routing rules.
7. Healthy backend targets.

### Interview-ready answer

> “The public hostname pointed to an ALB through a Route 53 alias record. ACM provided the TLS certificate. The ALB listener routed requests to the EKS workload through the configured ingress or target group.”

---

## 5. Have you created a DNS record?

### Interview-ready answer

> “Yes. I have created Route 53 records for application endpoints and validation requirements. For AWS load balancers, I used A-type alias records. For validation or integration scenarios, I used CNAME or TXT records as required. Before creating a record, I confirmed the hosted zone, record name, routing policy, target, and whether the record was public or private.”

### Terraform example: Alias record for an ALB

```hcl
data "aws_route53_zone" "main" {
  name         = "example.com."
  private_zone = false
}

resource "aws_route53_record" "application" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "app.example.com"
  type    = "A"

  alias {
    name                   = aws_lb.application.dns_name
    zone_id                = aws_lb.application.zone_id
    evaluate_target_health = true
  }
}
```

### Validation commands

```bash
dig app.example.com

nslookup app.example.com
```

### Important checks

- Correct hosted zone.
- Correct public or private zone.
- Correct record type.
- Correct target.
- Certificate includes the hostname.
- Load balancer is reachable from the intended network.
- Backend targets are healthy.

---

# VPC and EKS Networking

## 6. Explain the VPC and networking architecture used in your project.

### Example production architecture

Use this as an architecture template and replace it with your actual design.

```text
                         Internet
                            |
                     Internet Gateway
                            |
                  Application Load Balancer
                   /                       \
          Public Subnet AZ-A         Public Subnet AZ-B
                  |                         |
             NAT Gateway               NAT Gateway
                  |                         |
          Private Subnet AZ-A        Private Subnet AZ-B
            EKS Worker Nodes           EKS Worker Nodes
                  |                         |
          Private Data Subnet A      Private Data Subnet B
                    Database / Cache
```

### Example design

- One VPC in the selected AWS Region.
- Multiple Availability Zones for resilience.
- Public subnets for internet-facing load balancers and NAT Gateways.
- Private application subnets for EKS worker nodes.
- Private data subnets for databases, when separate data-tier subnets are required.
- Internet Gateway attached to the VPC.
- NAT Gateways for controlled outbound IPv4 connectivity from private subnets.
- Security groups for resource-level traffic control.
- Network ACLs for optional subnet-level controls.
- Route 53 for DNS.
- VPC endpoints for private access to selected AWS services.
- VPC Flow Logs for network visibility.

### Route-table pattern

| Subnet type | Default IPv4 route |
|---|---|
| Public subnet | `0.0.0.0/0` to Internet Gateway |
| Private subnet with outbound internet | `0.0.0.0/0` to NAT Gateway |
| Isolated data subnet | No direct internet route |

### Interview-ready answer

> “We used a multi-AZ VPC with separate public, private application, and private data subnets. Public subnets hosted internet-facing entry components and NAT Gateways. EKS nodes and application workloads ran in private subnets. Databases were isolated further, with traffic controlled through security groups and route tables.”

---

## 7. How many subnets did your project have?

The correct answer must match your actual project.

### Example answer

> “We used two Availability Zones. In each Availability Zone, we had one public subnet, one private application subnet, and one private data subnet. That resulted in six subnets.”

### Example layout

| Availability Zone | Public | Private application | Private data |
|---|---:|---:|---:|
| AZ-A | 1 | 1 | 1 |
| AZ-B | 1 | 1 | 1 |
| Total | 2 | 2 | 2 |

Total:

```text
2 + 2 + 2 = 6 subnets
```

### Alternative example

If your project had only public and private application subnets across three Availability Zones:

```text
3 public subnets + 3 private subnets = 6 subnets
```

### Interview-ready answer pattern

> “We had `[actual number]` subnets across `[actual number]` Availability Zones. They were divided into `[actual layout]`. We used separate route tables because public, application, and data tiers had different connectivity requirements.”

> **Interview tip:** Do not choose a subnet count from an example. State the actual count and explain the purpose of each subnet category.

---

## 8. In which subnet do you place the EKS cluster, and which networking components do you use?

### Important distinction

Amazon EKS has:

- An AWS-managed Kubernetes control plane.
- A data plane consisting of worker nodes or other supported compute.

Do not describe the managed control plane as an EC2 instance placed directly inside one of your subnets.

### Recommended application architecture

EKS worker nodes generally run in private subnets.

Internet-facing entry components can use public subnets, while internal load balancers use private subnets.

### Networking components

- VPC.
- Public and private subnets.
- Route tables.
- Internet Gateway.
- NAT Gateway, when required.
- Security groups.
- Network ACLs.
- EKS cluster private or restricted public API endpoint.
- AWS VPC CNI.
- Elastic Load Balancing.
- Route 53.
- VPC endpoints.
- Network policies, where supported and required.

### Example traffic flow

```text
Internet
   |
Internet-facing ALB in public subnets
   |
Ingress or Kubernetes Service
   |
EKS Pods on nodes in private subnets
   |
Database in private data subnets
```

### Private cluster without normal internet egress

Depending on workload requirements, private connectivity to AWS services can include endpoints for:

- ECR API.
- ECR Docker registry.
- S3.
- STS.
- EC2.
- CloudWatch Logs.
- Other required AWS services.

### Interview-ready answer

> “The EKS worker nodes were placed in private subnets across multiple Availability Zones. Internet-facing load balancers used public subnets. We used route tables, NAT Gateways or VPC endpoints for controlled outbound access, security groups for traffic restriction, and the AWS VPC CNI for Pod networking.”

**References:**

- [EKS cluster endpoint access](https://docs.aws.amazon.com/eks/latest/userguide/cluster-endpoint.html)
- [Private EKS clusters](https://docs.aws.amazon.com/eks/latest/userguide/private-clusters.html)

---

## 9. Why are you keeping the web application in a public subnet?

### Correct the premise carefully

In a secure design, the application servers or EKS worker nodes usually do not need to be in public subnets.

An internet-facing load balancer can be placed in public subnets, while application workloads remain in private subnets.

### Recommended architecture

```text
Internet
   |
Internet Gateway
   |
Public ALB
   |
Private application workloads
   |
Private database
```

### Why keep the workload private?

- No direct public IP is required.
- The load balancer becomes the controlled entry point.
- Backend security groups can allow traffic only from the load balancer.
- Administrative access can use approved private connectivity or management services.
- The architecture reduces direct exposure.

### When might a workload have public connectivity?

A workload might be publicly reachable when there is a documented requirement for direct internet communication.

That should be an intentional exception with:

- Restricted security groups.
- Minimal exposed ports.
- Strong authentication.
- Logging and monitoring.
- Documented approval.

### Interview-ready answer

> “I normally would not keep EKS worker nodes or application servers in public subnets. I would place the internet-facing load balancer in public subnets and keep the application in private subnets. The backend would accept application traffic only from the load balancer.”

---

## 10. Where will the load balancer be located?

It depends on whether the application is public or private.

### Internet-facing load balancer

An internet-facing load balancer uses public subnets.

Those subnets must have the appropriate route through an Internet Gateway.

```text
Internet
   |
Internet-facing ALB
   |
Private EKS workloads
```

### Internal load balancer

An internal load balancer uses private addressing and is suitable for access from:

- The same VPC.
- Connected VPCs.
- VPN.
- Direct Connect.
- Other approved private networks.

```text
Internal client
   |
Internal ALB or NLB
   |
Private EKS workloads
```

### Security-group design for an ALB

- ALB security group allows the intended client traffic.
- Backend security group allows the application and health-check ports from the ALB security group.
- Backend workloads are not broadly exposed.

### Interview-ready answer

> “For a public web application, the internet-facing load balancer uses public subnets across multiple Availability Zones while the application remains private. For an internal application, I use an internal load balancer with private connectivity.”

---

# AWS Storage and EBS Security

## 11. How many AWS storage services are available?

Avoid giving a fixed number because the answer depends on whether the interviewer means:

- Core storage types.
- Services in the AWS Storage category.
- Database services.
- Backup and data-transfer services.
- Specialized storage products.

### Core storage categories

| Storage model | Common AWS service |
|---|---|
| Object storage | Amazon S3 |
| Block storage | Amazon EBS |
| Shared file storage | Amazon EFS |
| Managed specialized file systems | Amazon FSx |
| Instance-local storage | EC2 instance store |
| Hybrid storage | AWS Storage Gateway |
| Centralized data protection | AWS Backup |
| Data movement | AWS DataSync |

### Interview-ready answer

> “AWS provides multiple storage services rather than one fixed storage option. The major models are object storage with S3, block storage with EBS, shared file storage with EFS, and specialized managed file systems through FSx. AWS also provides instance store, Storage Gateway, Backup, and DataSync. I select one based on access pattern, sharing, durability, performance, and recovery requirements.”

**Reference:** [Choosing an AWS storage service](https://docs.aws.amazon.com/decision-guides/latest/decision-guides/choosing-aws-storage-service.html)

---

## 12. How do you secure sensitive data stored on EBS?

### A. Encryption at rest

Use EBS encryption with an appropriate KMS key.

EBS encryption protects:

- Data at rest in the volume.
- Data moving between supported EC2 instances and attached EBS storage.
- Snapshots created from an encrypted volume.
- Volumes created from encrypted snapshots.

### B. KMS access control

For sensitive workloads:

- Use a customer-managed key when additional key-policy control is required.
- Limit who can administer the key.
- Limit which workloads can use the key.
- Audit key usage.
- Protect against accidental key disablement or deletion.

### C. IAM access control

Restrict permissions for:

- Creating and attaching volumes.
- Creating and sharing snapshots.
- Copying snapshots.
- Modifying snapshot permissions.
- Using the KMS key.

### D. Operating-system controls

EBS encryption does not replace host security.

Use:

- Filesystem permissions.
- Application authentication.
- Secure mount settings.
- Patch management.
- Restricted administrative access.
- Encryption in transit at the application layer where appropriate.

### E. Backup and recovery

- Create encrypted snapshots.
- Use approved backup policies.
- Restrict snapshot sharing.
- Test restoration.
- Apply retention requirements.
- Consider cross-account or cross-Region recovery only when required.

### F. Monitoring

Monitor:

- CloudTrail events.
- Volume attachment changes.
- Snapshot sharing.
- KMS events.
- Unexpected IAM activity.
- Configuration drift.

### Example Terraform configuration

```hcl
resource "aws_kms_key" "ebs" {
  description         = "KMS key for sensitive EBS volumes"
  enable_key_rotation = true

  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_ebs_volume" "sensitive" {
  availability_zone = "ap-south-1a"
  size              = 100
  type              = "gp3"

  encrypted  = true
  kms_key_id = aws_kms_key.ebs.arn

  tags = {
    Name        = "sensitive-application-data"
    Environment = "prod"
  }

  lifecycle {
    prevent_destroy = true
  }
}
```

### Interview-ready answer

> “I enable EBS encryption with KMS, restrict volume and snapshot operations through IAM, protect the operating system and application, monitor administrative actions, and maintain encrypted, tested backups. KMS encryption alone does not replace file permissions or application authorization.”

---

# Terraform

## 13. What is an iteration limit in Terraform?

The phrase **iteration limit** is ambiguous.

Terraform commonly supports repeated resource or module instances through:

- `count`.
- `for_each`.
- `for` expressions.
- `dynamic` blocks.

There is no single general interview-ready number called the Terraform iteration limit.

The practical limit is affected by:

- Cloud-service quotas.
- Memory and execution time.
- Provider and API behavior.
- State size.
- Plan complexity.
- CI/CD timeout limits.

### count example

```hcl
resource "aws_subnet" "private" {
  count = length(var.private_subnet_cidrs)

  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]
}
```

### for_each example

```hcl
resource "aws_s3_bucket" "application" {
  for_each = toset([
    "logs",
    "artifacts",
    "reports"
  ])

  bucket = "${var.project}-${var.environment}-${each.key}"
}
```

### count versus for_each

| Feature | count | for_each |
|---|---|---|
| Instance identity | Numeric index | Map key or set value |
| Best suited for | Nearly identical indexed instances | Objects with stable logical names |
| Resource address | `resource.name[0]` | `resource.name["logs"]` |
| List reorder impact | Can shift indexes | Stable keys reduce identity changes |

### Interview-ready answer

> “Terraform supports iteration through count, for_each, for expressions, and dynamic blocks. There is no single global iteration-limit value I would quote. In practice, limits come from cloud quotas, provider APIs, state size, pipeline constraints, and configuration complexity.”

---

## 14. What is a data block in Terraform?

A data block reads information from a provider or another supported source without creating the associated infrastructure object.

### Example: Read an existing VPC

```hcl
data "aws_vpc" "selected" {
  filter {
    name   = "tag:Name"
    values = ["shared-production-vpc"]
  }
}
```

Reference the result:

```hcl
resource "aws_security_group" "application" {
  name   = "application"
  vpc_id = data.aws_vpc.selected.id
}
```

### Example: Read an existing Route 53 zone

```hcl
data "aws_route53_zone" "main" {
  name         = "example.com."
  private_zone = false
}
```

### Resource versus data block

| Block | Purpose |
|---|---|
| `resource` | Creates or manages infrastructure. |
| `data` | Reads information without creating the associated infrastructure object. |

### Important behavior

Terraform generally attempts to read data sources during planning.

If data-source arguments depend on values that are not known until apply, the read can be deferred.

### Interview-ready answer

> “A data block queries information that already exists, such as a VPC, AMI, subnet, or hosted zone. I then reference the returned attributes instead of hardcoding IDs.”

**Reference:** [Terraform data block](https://developer.hashicorp.com/terraform/language/block/data)

---

## 15. What are Terraform modules?

A module is a collection of Terraform configuration files managed as a unit.

Every Terraform configuration has a root module. A root module can call reusable child modules.

### Typical module structure

```text
terraform/
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── versions.tf
│   └── eks/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── versions.tf
└── environments/
    ├── dev/
    │   └── main.tf
    └── prod/
        └── main.tf
```

### Benefits

- Reusability.
- Standardization.
- Reduced duplication.
- Consistent security controls.
- Easier testing and reviews.
- Clear input and output contracts.

### Good module design

A module should:

- Have a focused responsibility.
- Declare input variables.
- Expose useful outputs.
- Include provider and Terraform version constraints.
- Avoid unnecessary hardcoding.
- Be versioned when consumed remotely.

### Interview-ready answer

> “A Terraform module is a reusable collection of configuration files. I use modules to standardize infrastructure patterns such as VPCs, EKS clusters, IAM roles, and storage across environments.”

---

## 16. How do you call a Terraform module?

Use a `module` block.

### Call a local module

```hcl
module "network" {
  source = "../../modules/vpc"

  name               = "production"
  vpc_cidr           = "10.20.0.0/16"
  availability_zones = ["ap-south-1a", "ap-south-1b"]

  public_subnet_cidrs = [
    "10.20.0.0/24",
    "10.20.1.0/24"
  ]

  private_subnet_cidrs = [
    "10.20.10.0/24",
    "10.20.11.0/24"
  ]
}
```

### Use a module output

```hcl
module "eks" {
  source = "../../modules/eks"

  cluster_name       = "production-eks"
  vpc_id             = module.network.vpc_id
  private_subnet_ids = module.network.private_subnet_ids
}
```

### Remote module example

```hcl
module "network" {
  source = "git::https://example.com/infrastructure/modules.git//vpc?ref=v1.4.0"

  name     = "production"
  vpc_cidr = "10.20.0.0/16"
}
```

The example URL is a placeholder. Use the approved source and an immutable version reference.

### Module output syntax

```text
module.<module_name>.<output_name>
```

Example:

```hcl
module.network.vpc_id
```

### Interview-ready answer

> “I call a child module using a module block, configure its source, and pass values through input variables. I consume values exposed by the module through module-name and output-name references.”

**Reference:** [Terraform module outputs](https://developer.hashicorp.com/terraform/language/values/outputs)

---

## 17. Have you worked with null_resource?

### Explanation

Historically, `null_resource` was used when Terraform needed to run provisioners or conditionally trigger local or remote actions without managing a normal infrastructure object.

### Historical example

```hcl
resource "null_resource" "application_check" {
  triggers = {
    application_version = var.application_version
  }

  provisioner "local-exec" {
    command = "./scripts/check-release.sh"
  }
}
```

### Modern alternative

Terraform provides the built-in `terraform_data` resource for values that need a resource lifecycle and for triggering provisioners when no other logical managed resource is appropriate.

```hcl
resource "terraform_data" "application_check" {
  triggers_replace = [
    var.application_version
  ]

  provisioner "local-exec" {
    command = "./scripts/check-release.sh"
  }
}
```

### Caution about provisioners

Provisioners should be a last resort.

Prefer:

- Image-building tools.
- Cloud-init or user data.
- Configuration-management tools.
- CI/CD deployment stages.
- Provider-native Terraform resources.

Provisioners can make operations harder to make idempotent and harder to recover after partial failure.

### Interview-ready answer

> “I understand null_resource and have used it only where a Terraform-managed trigger was required for an external action. For newer configurations, I evaluate terraform_data. I avoid provisioners when a provider resource or deployment tool can perform the operation more reliably.”

**Reference:** [Terraform data resource](https://developer.hashicorp.com/terraform/language/resources/terraform-data)

---

## 18. Explain the Terraform state file.

Terraform state records the relationship between Terraform resource addresses and real infrastructure objects.

By default, Terraform uses a local state file named:

```text
terraform.tfstate
```

### Why state is required

Terraform uses state to:

- Track managed infrastructure.
- Map configuration addresses to real resources.
- Store resource attributes.
- Determine the changes proposed by a plan.
- Track dependencies and outputs.

### State may contain sensitive information

Even when an output is marked sensitive, the underlying value may still exist in state.

Therefore:

- Do not commit state to Git.
- Restrict access.
- Encrypt remote storage.
- Enable version recovery where appropriate.
- Avoid manual state-file editing.
- Use state locking where supported.
- Separate state by environment and ownership boundary.

### Useful state commands

```bash
terraform state list

terraform state show aws_vpc.main

terraform show

terraform plan
```

### Avoid manual editing

Use Terraform state commands and declarative configuration rather than opening and modifying JSON manually.

### Interview-ready answer

> “Terraform state is the mapping between configuration and real infrastructure. Because it can contain sensitive values, I store it in a restricted remote backend, use locking where supported, and never commit it to source control.”

---

## 19. In which file do you define where Terraform state is stored?

Terraform backend configuration is defined in a nested `backend` block inside the top-level `terraform` block.

The filename is not mandatory.

Teams commonly use:

```text
backend.tf
```

Terraform loads `.tf` files in the working directory as one configuration.

### Example: S3 backend

```hcl
terraform {
  backend "s3" {
    bucket       = "example-terraform-state"
    key          = "production/network/terraform.tfstate"
    region       = "ap-south-1"
    use_lockfile = true
    encrypt      = true
  }
}
```

### Initialize the backend

```bash
terraform init
```

To migrate existing state after changing the backend:

```bash
terraform init -migrate-state
```

### Important backend restrictions

Backend configuration cannot refer to normal input variables, local values, or data-source attributes.

Backend credentials should not be written into Terraform configuration.

Use approved credential mechanisms such as:

- Environment-based credentials.
- Workload identity.
- CI/CD role assumption.
- Approved AWS credential configuration.

### Interview-ready answer

> “I define the backend in the terraform block, commonly in a file named backend.tf. The filename is only an organizational convention. The backend block decides where state is stored.”

**Reference:** [Terraform backend configuration](https://developer.hashicorp.com/terraform/language/backend)

---

# Kubernetes and EKS Administration

## 20. How do you manage your Kubernetes cluster?

### Interview-ready answer

> “I use kubectl for operational inspection and troubleshooting, but I do not manage production primarily through ad hoc commands. Cluster infrastructure is managed through Terraform or another approved infrastructure-as-code workflow, while application manifests are managed through Helm, Kustomize, or Git-based deployment pipelines.”

### Tooling split

| Area | Typical management method |
|---|---|
| AWS infrastructure | Terraform |
| EKS configuration | Terraform, eksctl, or approved automation |
| Kubernetes resources | Helm, Kustomize, manifests |
| Production deployment | CI/CD or GitOps |
| Troubleshooting | kubectl |
| Autoscaling | HPA, Cluster Autoscaler, Karpenter, where applicable |
| Policies | Admission policies and infrastructure controls |
| Monitoring | CloudWatch, Prometheus, Grafana, or project tooling |

### Common kubectl commands

```bash
kubectl get nodes

kubectl get pods -A

kubectl get deployments -A

kubectl describe pod POD_NAME -n NAMESPACE

kubectl logs POD_NAME -n NAMESPACE

kubectl get events \
  -n NAMESPACE \
  --sort-by=.metadata.creationTimestamp

kubectl rollout status \
  deployment/APP_NAME \
  -n NAMESPACE
```

### Important practice

Avoid production changes such as the following without recording them in the intended source of truth:

```bash
kubectl edit
kubectl patch
kubectl scale
```

Emergency changes may be necessary, but they should be reconciled afterward.

### Interview-ready answer

> “I use declarative configuration through Terraform, Helm, and pipeline automation. kubectl is mainly for inspection, controlled operations, and troubleshooting. The repository remains the source of truth.”

---

## 21. If the EKS cluster endpoint is private, how do you access it with kubectl?

If public access is disabled, the Kubernetes API server can receive requests only through private network connectivity to the cluster VPC.

Running `kubectl` from an arbitrary internet-connected laptop will not work.

### Access patterns

Run `kubectl` from:

- A controlled administration host inside the VPC.
- A connected corporate network through VPN.
- A network connected through Direct Connect.
- A connected VPC.
- An approved CI/CD runner with private connectivity.
- A controlled management host accessed through an approved administrative channel.

### Access flow

```text
Administrator
     |
VPN, Direct Connect, or approved administrative channel
     |
Management environment inside connected network
     |
Private EKS API endpoint
```

### Configure kubeconfig

From a machine that can reach the private endpoint:

```bash
aws eks update-kubeconfig \
  --region ap-south-1 \
  --name production-eks
```

Then verify:

```bash
kubectl get nodes
```

### Two separate requirements

The administrator needs:

1. **Network reachability** to the private endpoint.
2. **Authorization** to perform Kubernetes actions.

Network connectivity alone does not grant Kubernetes permissions.

Authentication and authorization can involve:

- AWS IAM identity.
- EKS access entries or the cluster's configured access mechanism.
- Kubernetes RBAC.

### Security recommendations

- Prefer temporary credentials.
- Apply least privilege.
- Log administrative activity.
- Avoid assigning public IPs only for cluster management.
- Restrict access by network and identity.
- Separate production administrative access from normal developer access.

### Interview-ready answer

> “For a private EKS endpoint, I run kubectl from a network that can reach the cluster VPC, such as through VPN, Direct Connect, a controlled management host, or a private CI/CD runner. I also configure the required IAM and Kubernetes RBAC permissions.”

**Reference:** [EKS API endpoint configuration](https://docs.aws.amazon.com/eks/latest/userguide/config-cluster-endpoint.html)

---

## 22. What are EBS and EFS in Kubernetes?

EBS and EFS are AWS storage services that can provide persistent storage to Kubernetes applications through the appropriate CSI driver.

### EBS in Kubernetes

Amazon EBS is block storage.

The EBS CSI driver manages EBS volumes for Kubernetes persistent volumes.

Common characteristics:

- Block storage.
- Volume exists in one Availability Zone.
- Commonly used for a workload needing its own persistent disk.
- Suitable for databases and other disk-oriented applications when the application architecture supports it.
- Pod scheduling must be compatible with the volume's Availability Zone.

### EFS in Kubernetes

Amazon EFS is shared file storage.

The EFS CSI driver allows Kubernetes applications to mount EFS as persistent volumes.

Common characteristics:

- Shared file storage.
- Multiple Pods can access the same filesystem where the access mode and application design allow it.
- Useful for shared content or common filesystem data.
- Requires the appropriate network connectivity and security-group rules.

### Comparison

| Feature | EBS | EFS |
|---|---|---|
| Storage type | Block | Shared file |
| Scope | Availability Zone | Regional service with mount targets |
| Typical access | One workload or limited attachment model | Multiple clients |
| CSI driver | EBS CSI driver | EFS CSI driver |
| Typical use | Database disk, single-workload persistence | Shared content or common filesystem |

### EBS StorageClass example

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: encrypted-gp3
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
  encrypted: "true"
```

### PVC example

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: application-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: encrypted-gp3
  resources:
    requests:
      storage: 20Gi
```

### Important considerations

- The CSI driver needs appropriate AWS permissions.
- Storage encryption and backup must be designed explicitly.
- PVC deletion behavior depends on the StorageClass reclaim policy.
- Persistent storage does not replace application-level backup and recovery.

### Interview-ready answer

> “EBS provides block storage and is typically used for per-workload persistent disks. EFS provides shared file storage and can support multiple clients. In EKS, both are integrated using their CSI drivers.”

**References:**

- [EBS CSI driver](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html)
- [EFS CSI driver](https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html)

---

## 23. A Pod needs to access a file in S3. How does it access the bucket?

### Recommended approach: AWS SDK or API

The application should normally use an AWS SDK or the S3 API.

The Pod receives an IAM identity with only the required S3 permissions.

For EKS, workload identity can be provided through:

- EKS Pod Identity.
- IAM Roles for Service Accounts, where used in the environment.

Avoid storing long-lived AWS access keys in:

- Container images.
- Kubernetes manifests.
- ConfigMaps.
- Source code.
- Plain environment configuration.

### Access flow

```text
Application Pod
      |
EKS workload identity
      |
Temporary AWS credentials
      |
S3 API through HTTPS
      |
S3 bucket and object
```

### Example IAM policy

Replace the bucket and prefix with the approved values.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListApprovedPrefix",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::example-application-data",
      "Condition": {
        "StringLike": {
          "s3:prefix": [
            "reports/*"
          ]
        }
      }
    },
    {
      "Sid": "ReadApprovedObjects",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::example-application-data/reports/*"
    }
  ]
}
```

If the Pod only needs one known object and does not list the bucket, `s3:ListBucket` might not be required.

### Example service account

The exact configuration depends on whether the cluster uses EKS Pod Identity or IRSA.

Illustrative IRSA-style service account:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: reports-reader
  namespace: reporting
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/reports-s3-reader
```

### Reference the service account from a Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: reports-api
  namespace: reporting
spec:
  replicas: 2

  selector:
    matchLabels:
      app: reports-api

  template:
    metadata:
      labels:
        app: reports-api

    spec:
      serviceAccountName: reports-reader

      containers:
        - name: application
          image: example-registry/reports-api:1.0.0
```

### Example application logic

Python SDK example:

```python
import boto3

s3 = boto3.client("s3")

response = s3.get_object(
    Bucket="example-application-data",
    Key="reports/daily-report.json"
)

content = response["Body"].read().decode("utf-8")
print(content)
```

Do not print sensitive object content in production logs.

### Network connectivity

The Pod also needs an appropriate network path to S3.

Possible options include:

- Controlled outbound connectivity.
- An S3 VPC endpoint where suitable.

For a private cluster, using an S3 gateway endpoint can keep supported S3 access on private AWS networking.

### Additional access controls

Depending on the security requirements, evaluate:

- Bucket policy.
- IAM identity policy.
- S3 Block Public Access.
- KMS key permissions for SSE-KMS objects.
- VPC endpoint policy.
- TLS.
- CloudTrail data-event logging where required.

### If the application requires a filesystem interface

The Mountpoint for Amazon S3 CSI driver can present an existing S3 bucket as a Kubernetes volume for supported use cases.

However:

- S3 is object storage.
- The mounted interface does not provide every POSIX filesystem behavior.
- The driver supports static provisioning for existing buckets.
- Confirm application compatibility before selecting this method.

### Interview-ready answer

> “The preferred approach is for the Pod to access S3 through the AWS SDK or S3 API using a workload-specific IAM role. I grant only the required bucket and object permissions and provide private connectivity where appropriate. If the application specifically requires a mounted interface, I evaluate the Mountpoint for S3 CSI driver and its filesystem limitations.”

**Reference:** [Mountpoint for Amazon S3 CSI driver](https://docs.aws.amazon.com/eks/latest/userguide/s3-csi.html)

---

# Quick Revision

1. Explain how you used AWS services, not only their names.
2. An A record returns an IPv4 address; a CNAME points to another DNS name.
3. A Route 53 alias can point the zone apex to supported AWS resources.
4. DNS usually points the application hostname to a load balancer.
5. Avoid disclosing confidential customer domains.
6. Public and private hosted zones serve different audiences.
7. State the actual subnet count from your project.
8. EKS worker nodes normally run in private subnets.
9. The managed EKS control plane is not an EC2 instance in your subnet.
10. A public load balancer does not require public application nodes.
11. Internet-facing and internal load balancers have different exposure.
12. AWS storage is categorized as object, block, file, local, and specialized storage.
13. Protect EBS with encryption, IAM, operating-system controls, monitoring, and tested backups.
14. Terraform has no single general value called an iteration limit.
15. A data block reads existing information; a resource block manages infrastructure.
16. Modules provide reusable infrastructure patterns.
17. Use module outputs to pass values between modules.
18. Evaluate `terraform_data` for newer use cases previously handled by `null_resource`.
19. Terraform state can contain sensitive values.
20. Backend configuration belongs in the top-level `terraform` block.
21. `backend.tf` is a naming convention, not a requirement.
22. Use declarative tools as the production source of truth.
23. Private EKS access requires both network reachability and authorization.
24. EBS is block storage; EFS is shared file storage.
25. Pods should access S3 using workload identity rather than long-lived access keys.
26. S3 object access and filesystem mounting are different application patterns.
