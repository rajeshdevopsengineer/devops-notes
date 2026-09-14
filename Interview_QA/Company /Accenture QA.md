These are strong AWS/EKS/Terraform interview questions. For EKS specifically, I’d mention the **current EKS Access Entries approach** first, and then the older `aws-auth` ConfigMap method, because interviewers may ask about either.

### 1. You created an IAM user in AWS and configured role-based access in EKS. How do you bind the IAM user to the EKS role?

AWS IAM authentication and Kubernetes RBAC authorization are two different layers.

The flow is:

```text
IAM User / IAM Role
        |
        | AWS IAM authentication
        v
      EKS API
        |
        | mapped to Kubernetes identity/group
        v
 Kubernetes RBAC
        |
        +--> Role / ClusterRole
        |
        +--> RoleBinding / ClusterRoleBinding
```

With modern EKS, the recommended method is **EKS Access Entries**.

For example, create an access entry for an IAM principal:

```bash
aws eks create-access-entry \
  --cluster-name prod-cluster \
  --principal-arn arn:aws:iam::123456789012:user/dev-user \
  --kubernetes-groups developers
```

Then create Kubernetes RBAC:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: developer-role
rules:
- apiGroups: [""]
  resources:
    - pods
    - services
  verbs:
    - get
    - list
    - watch
```

And bind the Kubernetes group:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: development
  name: developer-binding
subjects:
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
```

So the key interview statement is:

> **We don't directly bind an IAM user to a Kubernetes Role. We authenticate the IAM principal to EKS, map it to a Kubernetes identity/group, and RBAC RoleBinding or ClusterRoleBinding provides authorization.**

Another modern option is associating an EKS access policy directly:

```bash
aws eks associate-access-policy \
  --cluster-name prod-cluster \
  --principal-arn arn:aws:iam::123456789012/role/DeveloperRole \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy \
  --access-scope type=namespace,namespaces=development
```

Historically, EKS used the `aws-auth` ConfigMap:

```yaml
mapRoles:
- rolearn: arn:aws:iam::123456789012:role/DeveloperRole
  username: developer
  groups:
    - developers
```

You should know `aws-auth` because many existing clusters still use it, but **EKS Access Entries are the preferred newer approach**.

---

### 2. You have 10 AWS accounts. How will you securely log in without access keys?

For multiple AWS accounts, my preferred approach is **AWS Organizations + AWS IAM Identity Center**.

Architecture:

```text
                     AWS Organizations
                            |
                  IAM Identity Center
                            |
                 Corporate Identity Provider
                 Okta / Entra ID / AD
                            |
          +---------+-------+---------+
          |         |       |         |
       Dev Acct   QA Acct  Prod    Security
          |         |       |
       Roles      Roles    Roles
```

The user logs in once through SSO and then assumes appropriate roles in each AWS account.

For example:

```text
User
 |
 v
IAM Identity Center
 |
 +--> Account 1 -> Developer Role
 +--> Account 2 -> Developer Role
 +--> Account 3 -> ReadOnly Role
 +--> Account 4 -> Admin Role
```

For CLI access:

```bash
aws configure sso
```

Then:

```bash
aws sso login --profile production
```

And:

```bash
aws sts get-caller-identity --profile production
```

Advantages include:

* No long-lived access keys
* Centralized user lifecycle management
* MFA
* Short-lived credentials
* Account-level role separation
* Easier auditing
* Least-privilege permission sets

For ten or more accounts, I would generally avoid creating separate IAM users in every account.

---

### 3. What are the ways to log in to an AWS account?

There are several authentication mechanisms.

| Method                      | Typical use                          |
| --------------------------- | ------------------------------------ |
| AWS account root user       | Emergency/account-level tasks only   |
| IAM user                    | Legacy/direct console authentication |
| IAM Identity Center / SSO   | Recommended enterprise access        |
| Federated identity          | Okta, Entra ID, AD, SAML/OIDC        |
| AssumeRole                  | Cross-account/temporary access       |
| AWS CLI with SSO            | Engineer CLI access                  |
| IAM Role for EC2/ECS/Lambda | Workload authentication              |
| EKS Pod Identity / IRSA     | Kubernetes workload authentication   |

In an enterprise environment, I would normally use:

```text
Human users
    |
IAM Identity Center
    |
Assume IAM Role
    |
Temporary STS credentials
```

Applications should also avoid static IAM user credentials.

For example, an EC2 instance should use an **instance profile**, Lambda an **execution role**, and EKS workloads **EKS Pod Identity or IRSA**.

---

### 4. Does Amazon S3 require a VPC?

**No.**

Amazon S3 is a regional AWS managed service and is not deployed inside your VPC.

For example:

```text
VPC
 |
 +-- EC2
 |
 +-- EKS
 |
 +-- Lambda/VPC
 |
 +-------------> Amazon S3
```

An EC2 instance can access S3 through the internet using NAT/Internet Gateway depending on its network design.

However, for private access, I would normally use an **S3 Gateway VPC Endpoint**:

```text
Private Subnet
     |
     v
EC2 / EKS
     |
     v
S3 Gateway Endpoint
     |
     v
Amazon S3
```

This allows traffic to S3 without going through:

```text
NAT Gateway
Internet Gateway
Public Internet
```

Example Terraform:

```hcl
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.us-east-1.s3"

  vpc_endpoint_type = "Gateway"
}
```

An important interview distinction is:

> S3 isn't inside the VPC, but workloads inside a VPC can privately reach S3 using a VPC endpoint.

---

### 5. What happens when we run `terraform init`?

`terraform init` initializes a Terraform working directory.

When we run:

```bash
terraform init
```

Terraform performs several activities.

First, it initializes the **backend**.

For example:

```hcl
terraform {
  backend "s3" {
    bucket = "company-terraform-state"
    key    = "production/app.tfstate"
    region = "us-east-1"
  }
}
```

Terraform connects/configures the backend used for state storage.

Second, it downloads required **providers**:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}
```

Terraform downloads the AWS provider into:

```text
.terraform/
```

Third, it downloads referenced **modules**.

For example:

```hcl
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
}
```

Fourth, it creates/updates the provider dependency lock file:

```text
.terraform.lock.hcl
```

So:

```text
terraform init
     |
     +--> Initialize backend
     |
     +--> Download providers
     |
     +--> Download modules
     |
     +--> Create/update dependency lock
     |
     +--> Prepare working directory
```

It **does not create infrastructure**.

That happens during:

```bash
terraform apply
```

---

### 6. Write Terraform to create EC2 instances in multiple regions

For multiple AWS regions, use **provider aliases**.

Example for `us-east-1`, `us-west-2`, and `eu-west-1`:

```hcl
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
    }
  }
}

provider "aws" {
  alias  = "useast"
  region = "us-east-1"
}

provider "aws" {
  alias  = "uswest"
  region = "us-west-2"
}

provider "aws" {
  alias  = "euwest"
  region = "eu-west-1"
}

data "aws_ami" "amazon_linux_useast" {
  provider    = aws.useast
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

data "aws_ami" "amazon_linux_uswest" {
  provider    = aws.uswest
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

data "aws_ami" "amazon_linux_euwest" {
  provider    = aws.euwest
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

resource "aws_instance" "useast_server" {
  provider      = aws.useast
  ami           = data.aws_ami.amazon_linux_useast.id
  instance_type = "t3.micro"

  tags = {
    Name = "us-east-server"
  }
}

resource "aws_instance" "uswest_server" {
  provider      = aws.uswest
  ami           = data.aws_ami.amazon_linux_uswest.id
  instance_type = "t3.micro"

  tags = {
    Name = "us-west-server"
  }
}

resource "aws_instance" "euwest_server" {
  provider      = aws.euwest
  ami           = data.aws_ami.amazon_linux_euwest.id
  instance_type = "t3.micro"

  tags = {
    Name = "eu-west-server"
  }
}
```

One important detail: **AMIs are regional**, so I should not blindly use the same AMI ID across all regions.

---

### 7. You have region1, region2 and region3 providers. If you create an EC2 instance, where is it created?

It depends on which **provider configuration the resource uses**.

Suppose:

```hcl
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

provider "aws" {
  alias  = "europe"
  region = "eu-west-1"
}
```

If I write:

```hcl
resource "aws_instance" "server" {
  ami           = "ami-xxxx"
  instance_type = "t3.micro"
}
```

it uses the **default unaliased AWS provider**, so it is created in:

```text
us-east-1
```

If I write:

```hcl
resource "aws_instance" "server" {
  provider = aws.west

  ami           = "ami-xxxx"
  instance_type = "t3.micro"
}
```

then it is created in:

```text
us-west-2
```

So:

> **Declaring three regions doesn't mean Terraform automatically creates the resource in all three. The resource runs against the provider configuration assigned to it.**

---

### 8. Frontend, backend and database are all in private subnets. How can an end user access the application?

A private subnet does not mean the application can't be exposed to users.

I would put a **public-facing load balancer** in public subnets and keep the application workloads private.

Architecture:

```text
                       Internet
                          |
                        DNS
                     Route 53
                          |
                         WAF
                          |
                    Public ALB
                   /          \
             Public AZ-1   Public AZ-2
                   |
                   v
        -------------------------
        |     Private Subnets   |
        |                       |
        |    Frontend Pods      |
        |         |             |
        |         v             |
        |    Backend Pods       |
        |         |             |
        |         v             |
        |       RDS             |
        -------------------------
```

For EKS:

```text
User
 |
HTTPS :443
 |
Route 53
 |
AWS WAF
 |
ALB
 |
Ingress
 |
Kubernetes Service
 |
Frontend Pods
 |
Backend Service
 |
Backend Pods
 |
RDS/Aurora
```

Security groups would typically allow:

```text
Internet
   |
   | 443
   v
ALB Security Group
   |
   | Application port
   v
Frontend Security Group
   |
   v
Backend Security Group
   |
   | 3306/5432
   v
Database Security Group
```

The database should **never need direct public access**.

Also note that a private frontend isn't a contradiction: the **load balancer is public**, while the frontend workload remains private.

---

### 9. Secrets exist in AWS Secrets Manager. How can EKS access them?

There are several approaches, but for modern EKS I would prefer **EKS Pod Identity**, or use **IRSA — IAM Roles for Service Accounts** where appropriate.

The key principle is:

> Don't store AWS access keys in Kubernetes Secrets.

Architecture:

```text
Application Pod
      |
      v
Kubernetes Service Account
      |
      v
EKS Pod Identity / IRSA
      |
      v
IAM Role
      |
      | secretsmanager:GetSecretValue
      v
AWS Secrets Manager
```

The IAM policy could be:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/database-*"
    }
  ]
}
```

Then associate that permission with the workload identity.

The application can retrieve the secret through the AWS SDK:

```python
client = boto3.client("secretsmanager")

response = client.get_secret_value(
    SecretId="prod/database"
)
```

Another very common solution is the **Secrets Store CSI Driver with AWS Secrets and Configuration Provider**.

Flow:

```text
AWS Secrets Manager
       |
       v
AWS Secrets Store Provider
       |
       v
Secrets Store CSI Driver
       |
       v
Kubernetes Pod
       |
       v
Mounted secret file
```

Example conceptually:

```yaml
volumeMounts:
- name: secrets-store
  mountPath: "/mnt/secrets-store"
  readOnly: true
```

The pod can then consume the secret as a mounted file.

For interview purposes, I'd say:

**"I'd give the pod a dedicated IAM identity using EKS Pod Identity or IRSA and grant only `secretsmanager:GetSecretValue` for the required secret. If the application needs filesystem-mounted secrets, I'd use the Secrets Store CSI Driver."**

---

### 10. How do you set up RBAC in Amazon EKS?

I explain this as **two layers**:

```text
AWS Authentication
        +
Kubernetes Authorization
```

First, define a Kubernetes `Role` or `ClusterRole`.

A namespace-specific Role:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: developer
rules:
- apiGroups: [""]
  resources:
    - pods
    - services
  verbs:
    - get
    - list
    - watch
    - create
    - update
```

Then create a RoleBinding:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: development
  name: developer-binding
subjects:
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

Then map the AWS IAM principal to that Kubernetes group using EKS Access Entries:

```bash
aws eks create-access-entry \
  --cluster-name production \
  --principal-arn arn:aws:iam::123456789012:role/DeveloperRole \
  --kubernetes-groups developers
```

The complete flow becomes:

```text
Engineer
   |
   v
IAM Identity Center
   |
   v
IAM DeveloperRole
   |
   v
EKS Access Entry
   |
   v
Kubernetes Group: developers
   |
   v
RoleBinding
   |
   v
Role
   |
   +--> get pods
   +--> list pods
   +--> create pods
```

For administrators, I would generally use a separate role and tightly control its access.

I can verify permissions using:

```bash
kubectl auth can-i get pods -n development
```

or:

```bash
kubectl auth can-i delete deployments -n production
```

For another user's effective permissions:

```bash
kubectl auth can-i --list
```

The senior-level point to make is that I follow **least privilege**. Developers might have write access in `development`, read-only access in staging, and little or no direct production access.

A good overall EKS access design looks like:

```text
                 Corporate User
                       |
                       v
              IAM Identity Center
                       |
             +---------+---------+
             |                   |
        DeveloperRole        SRERole
             |                   |
             v                   v
       EKS Access Entry    EKS Access Entry
             |                   |
             v                   v
        developers             sre
             |                   |
       RoleBinding       ClusterRoleBinding
             |                   |
             v                   v
       Namespace Role       ClusterRole
             |                   |
         Limited Access       Broader Access
```

For interviews, one especially useful distinction to remember is:

**IAM answers "who can authenticate to the EKS cluster?" Kubernetes RBAC answers "what can that authenticated identity do inside Kubernetes?"**
