## 1. Terraform code to launch an EC2 instance

Below is a basic example to launch an Amazon Web Services EC2 instance using HashiCorp Terraform.

```hcl
provider "aws" {
  region = "ap-south-1"
}

resource "aws_instance" "webserver" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t2.micro"
  key_name      = "my-keypair"

  tags = {
    Name = "Terraform-EC2"
    Env  = "Dev"
  }
}
```

### Explanation:

* **provider "aws"** → Terraform connects to AWS.
* **region** → Mumbai region (`ap-south-1`)
* **resource "aws_instance"** → Creates an EC2 VM.
* **ami** → OS image ID.
* **instance_type** → CPU/RAM size.
* **key_name** → SSH key for login.
* **tags** → Labels for management.

### Commands:

```bash
terraform init
terraform plan
terraform apply
```

### Best Practice Version:

```hcl
resource "aws_instance" "web" {
  ami                    = var.ami_id
  instance_type          = var.instance_type
  subnet_id              = var.subnet_id
  vpc_security_group_ids = [aws_security_group.web.id]

  tags = {
    Name = "Prod-Web"
  }
}
```

---

## 2. What happens if your state file is accidentally deleted?

Terraform state file (`terraform.tfstate`) tracks real infrastructure.

If deleted:

### What Happens:

* Terraform loses mapping of resources.
* Existing infrastructure still runs in AWS.
* Terraform thinks resources do not exist.
* Next `terraform apply` may try to recreate them.

### Risks:

* Duplicate resources
* Downtime
* Conflicts
* Cost increase

### Recovery Options:

### Option 1: Restore Backup

Terraform usually keeps:

```bash
terraform.tfstate.backup
```

### Option 2: Import Existing Resources

```bash
terraform import aws_instance.web i-1234567890
```

### Option 3: Remote Backend Restore

Use Amazon Web Services S3 + DynamoDB locking.

### Best Practice:

Use remote backend:

```hcl
terraform {
 backend "s3" {
   bucket = "tf-state-prod"
   key    = "prod/terraform.tfstate"
   region = "ap-south-1"
 }
}
```

---

## 3. What happens if multiple team members run terraform apply simultaneously?

### What Happens Without Locking:

Two users modify same infrastructure at same time.

Result:

* State corruption
* Conflicting changes
* Partial deployments
* Lost updates

### Example:

Engineer A creates EC2
Engineer B deletes SG simultaneously

Infrastructure becomes inconsistent.

### Solution: State Locking

Use backend locking:

* S3 + DynamoDB
* Terraform Cloud
* Consul

### Example:

```hcl
backend "s3" {
 bucket         = "tf-state"
 key            = "prod.tfstate"
 region         = "ap-south-1"
 dynamodb_table = "terraform-lock"
}
```

### Behavior:

If one user runs apply:

```bash
Acquiring state lock...
```

Second user waits or gets lock error.

### Best Practice:

* CI/CD only apply
* PR approval process
* One workspace per env

---

## 4. What happens if a resource fails halfway through a terraform apply?

Terraform applies resources in dependency order.

### Example:

10 resources planned:

* VPC created ✅
* Subnet created ✅
* EC2 creation fails ❌

### Result:

Already created resources remain.

Terraform stops execution.

### Next Apply:

Terraform checks state and resumes remaining resources.

```bash
terraform apply
```

Will continue from failed point.

### Important:

Terraform is **not fully transactional** like a database rollback.

### Cleanup If Needed:

```bash
terraform destroy
```

### Best Practice:

* Smaller modules
* Retry logic
* Use `depends_on`
* Validate quotas before apply

---

## 5. What happens when AWS API rate limits are hit during a large terraform apply?

Amazon Web Services APIs throttle excessive requests.

### Example Errors:

```bash
ThrottlingException
Rate exceeded
RequestLimitExceeded
```

### What Terraform Does:

Providers retry automatically with exponential backoff.

### If Severe:

Apply slows down or fails.

### Common During:

* Creating hundreds of EC2
* Many IAM resources
* Massive tagging operations

### Fixes:

### Reduce Parallelism

```bash
terraform apply -parallelism=5
```

(Default often 10)

### Split Deployments

Separate networking / compute / database.

### Use Retry Settings

AWS provider supports retries.

### Best Practice:

Use modules and staggered deployments.

---

## 6. What happens if terraform plan shows no changes but infrastructure was modified outside Terraform?

This is called **configuration drift**.

### Example:

Someone manually changes EC2 size in AWS Console.

Terraform config still says `t2.micro`.

### If Terraform Plan Shows No Changes:

Usually state file outdated or refresh disabled.

### Run:

```bash
terraform plan -refresh-only
```

or

```bash
terraform refresh
```

Terraform re-reads actual cloud resources.

### Then Plan May Show Drift:

```bash
~ instance_type = t3.medium -> t2.micro
```

### Resolution Options:

1. Accept manual change → update code
2. Revert drift → apply Terraform

### Best Practice:

Never change manually in console.

---

## 7. What happens if you delete a resource definition from your configuration?

Example remove:

```hcl
resource "aws_instance" "web" {
}
```

### What Happens:

Terraform sees resource exists in state but no longer in code.

### Next Plan:

```bash
- destroy aws_instance.web
```

### Apply:

Terraform deletes real resource from AWS.

### If You Want Keep Resource But Stop Managing It:

```bash
terraform state rm aws_instance.web
```

This removes from state only.

### Use Case:

Hand over DB to another team.

---

## 8. What happens if a provider API changes between Terraform versions?

Terraform providers evolve.

Example: Amazon Web Services provider changes deprecated field.

### What Happens:

* `terraform init` downloads newer provider
* Old code may fail
* Resource schema may change
* Plan may force replacement

### Example:

```bash
Error: Unsupported argument
```

### Prevention:

### Pin Versions

```hcl
terraform {
 required_providers {
   aws = {
     source  = "hashicorp/aws"
     version = "~> 5.0"
   }
 }
}
```

### Upgrade Safely:

```bash
terraform init -upgrade
terraform plan
```

### Best Practice:

* Read changelog
* Test in dev first
* Use lock file `.terraform.lock.hcl`

---

## 9. What happens if you have circular dependencies in your Terraform modules?

Terraform builds dependency graph.

Example:

* Security Group depends on EC2
* EC2 depends on Security Group

### Result:

```bash
Error: Cycle detected
```

Terraform cannot determine creation order.

### Example Bad Design:

```hcl
resource A depends_on B
resource B depends_on A
```

### Fixes:

### Break Dependency

Create SG first, then attach EC2.

### Use Data Sources

Reference existing resource.

### Modularize Properly

Separate network module and compute module.

### Best Practice:

Keep one-way dependencies only.

---

## 10. What happens if you exceed AWS service quotas during deployment?

AWS has quotas like:

* VPCs per region
* Elastic IPs
* EC2 instances
* IAM roles
* Security groups

### Example Error:

```bash
LimitExceeded
QuotaExceeded
Max number of VPCs reached
```

### What Terraform Does:

Resource creation fails.

Already created resources remain.

### Impact:

Partial deployment.

### Fixes:

### Check Quotas Before Apply

Use AWS Service Quotas console.

### Request Increase

Submit quota request in AWS.

### Cleanup Unused Resources

```bash
terraform destroy
```

### Add Validation in Pipeline

Example:

* Check available IPs
* Check instance limits

### Best Practice:

Capacity planning before scaling.

---

# Interview Summary Answer

Terraform itself is declarative, but success depends on:

* Proper remote state management
* Locking
* Version pinning
* Drift detection
* Quota planning
* Modular architecture
* CI/CD controlled applies

---

# Senior-Level Interview Tip

If asked these questions in interview, answer in format:

**Problem → Impact → Terraform Behavior → Fix → Best Practice**

That sounds highly professional.

---

## 10. What happens if you exceed AWS service quotas during deployment?

Amazon Web Services enforces quotas (limits) on many services.

### Examples:

* EC2 instances per region
* VPCs per region
* Elastic IPs
* Security groups
* IAM roles
* EBS volumes

### What Happens During Terraform Apply:

Terraform sends create request, AWS rejects it.

Example errors:

```bash id="x1"
LimitExceeded
QuotaExceeded
InstanceLimitExceeded
```

### Result:

* Resource creation fails
* Terraform stops at failed resource
* Previously created resources remain
* Partial deployment state possible

### Fix:

* Request quota increase
* Clean unused resources
* Pre-check quotas in pipeline
* Use smaller staged deployments

### Best Practice:

Always validate quotas before scaling production workloads.

---

## 11. What happens if you lose access to the remote backend storing your state?

Remote backends commonly use:

* Amazon Web Services S3 + DynamoDB
* HashiCorp Terraform Cloud
* Consul backend

### If Access Is Lost:

Examples:

* IAM permission removed
* Bucket deleted
* Network issue
* Backend credentials expired

### What Happens:

```bash id="x2"
terraform plan
terraform apply
```

Fails because Terraform cannot read/write state.

### Impact:

* No new deployments
* No state lock operations
* Cannot detect changes accurately

### Recovery:

1. Restore backend access
2. Restore permissions
3. Recover deleted bucket backup/versioning
4. Use backend migration if needed

### Example:

```bash id="x3"
terraform init -migrate-state
```

### Best Practice:

* Enable S3 versioning
* Least privilege IAM
* Backend backup strategy
* Monitor backend health

---

## 12. Write a Terraform script for VPC architecture for production

## Example Production VPC (2 Public + 2 Private Subnets)

```hcl id="x4"
provider "aws" {
  region = "ap-south-1"
}

resource "aws_vpc" "prod" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "prod-vpc"
  }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.prod.id
}

resource "aws_subnet" "public_1" {
  vpc_id                  = aws_vpc.prod.id
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = true
  availability_zone       = "ap-south-1a"
}

resource "aws_subnet" "public_2" {
  vpc_id            = aws_vpc.prod.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = "ap-south-1b"
}

resource "aws_subnet" "private_1" {
  vpc_id     = aws_vpc.prod.id
  cidr_block = "10.0.11.0/24"
}

resource "aws_subnet" "private_2" {
  vpc_id     = aws_vpc.prod.id
  cidr_block = "10.0.12.0/24"
}
```

## Production Best Practice Includes:

* NAT Gateway
* Multi-AZ subnets
* Route tables
* NACLs
* Security Groups
* Flow logs
* Private DB subnet group

### Interview Tip:

Mention using modules, separate state, tagging, and HA design.

---

## 13. What’s the purpose of terraform validate, plan, and taint commands in daily workflow?

## terraform validate

Checks syntax and internal consistency.

```bash id="x5"
terraform validate
```

Use before commit or pipeline.

### Detects:

* Invalid references
* Wrong block syntax
* Missing arguments

---

## terraform plan

Shows execution preview.

```bash id="x6"
terraform plan
```

### Purpose:

* What will be created
* What will change
* What will be destroyed

Used in PR approval process.

---

## terraform taint (legacy, now replace preferred)

Marks resource for recreation.

```bash id="x7"
terraform taint aws_instance.web
```

Next apply destroys and recreates it.

Modern preferred method:

```bash id="x8"
terraform apply -replace="aws_instance.web"
```

### Daily Workflow:

```bash id="x9"
terraform fmt
terraform validate
terraform plan
terraform apply
```

---

## 14. How do you handle Terraform resource drift in production environments?

Drift = actual infrastructure changed outside Terraform.

### Example:

Engineer changes EC2 type manually in AWS console.

### Detection:

```bash id="x10"
terraform plan
```

or

```bash id="x11"
terraform plan -refresh-only
```

### Handling Process:

### Option 1: Revert Manual Change

Run apply to restore desired state.

### Option 2: Accept Change

Update Terraform code to match real infra.

### Production Best Practices:

* Restrict console access
* Use change management
* Scheduled drift detection pipeline
* Alerts on manual modifications

### Enterprise Method:

Daily CI job runs:

```bash id="x12"
terraform plan -detailed-exitcode
```

---

## 15. How do you implement condition-based resource provisioning using Terraform?

Use:

* `count`
* `for_each`
* conditional expressions

---

## Example 1: Create resource only in prod

```hcl id="x13"
resource "aws_instance" "web" {
  count = var.environment == "prod" ? 1 : 0
}
```

If prod = create 1 instance, else none.

---

## Example 2: Conditional instance size

```hcl id="x14"
instance_type = var.environment == "prod" ? "t3.large" : "t3.micro"
```

---

## Example 3: Multiple users

```hcl id="x15"
resource "aws_iam_user" "users" {
  for_each = toset(var.user_list)
  name     = each.value
}
```

### Best Practice:

Keep logic simple and variable-driven.

---

## 16. What's your approach to safely rotating secrets or keys in an IaC workflow?

### Never Hardcode Secrets

Bad:

```hcl id="x16"
password = "Admin123"
```

---

## Secure Approach:

### Use Secret Stores:

* Amazon Web Services Secrets Manager
* AWS SSM Parameter Store
* HashiCorp Vault
* CI/CD secret variables

---

## Rotation Workflow:

1. Create new secret version
2. Update consuming apps
3. Validate connectivity
4. Decommission old secret

---

## Terraform Example:

```hcl id="x17"
data "aws_secretsmanager_secret_version" "db" {
  secret_id = "prod/db/password"
}
```

### Best Practices:

* Use short-lived credentials
* Use IAM roles instead of keys
* Rotate periodically
* Audit secret access

---

## 17. Write Terraform code to create any resource and modularize it

## Root Module Structure

```bash id="x18"
modules/
  ec2/
main.tf
variables.tf
```

---

## modules/ec2/main.tf

```hcl id="x19"
resource "aws_instance" "this" {
  ami           = var.ami
  instance_type = var.instance_type

  tags = {
    Name = var.name
  }
}
```

## modules/ec2/variables.tf

```hcl id="x20"
variable "ami" {}
variable "instance_type" {}
variable "name" {}
```

## Root main.tf

```hcl id="x21"
module "webserver" {
  source        = "./modules/ec2"
  ami           = "ami-123456"
  instance_type = "t2.micro"
  name          = "prod-web"
}
```

### Benefits:

* Reusable
* Standardized
* Easy scaling
* Cleaner codebase

---

## 18. How do you handle Terraform state file corruption?

State corruption examples:

* Manual editing
* Interrupted write
* Backend issue
* Merge conflicts

### Recovery Steps:

## 1. Restore Backup

```bash id="x22"
terraform.tfstate.backup
```

## 2. Pull Remote State

```bash id="x23"
terraform state pull
```

## 3. Validate Resources

```bash id="x24"
terraform plan
```

## 4. Re-import Missing Resources

```bash id="x25"
terraform import aws_instance.web i-12345
```

### Prevention:

* Remote backend
* Locking
* No manual editing
* Versioning enabled

---

## 19. You need to import an existing AWS VPC into Terraform. What are the steps?

Suppose VPC already exists manually.

VPC ID:

```bash id="x26"
vpc-0abc1234
```

---

## Step 1: Write Resource Block

```hcl id="x27"
resource "aws_vpc" "prod" {
}
```

---

## Step 2: Import Resource

```bash id="x28"
terraform import aws_vpc.prod vpc-0abc1234
```

---

## Step 3: Verify

```bash id="x29"
terraform state list
```

---

## Step 4: Add Full Configuration

```hcl id="x30"
resource "aws_vpc" "prod" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "prod-vpc"
  }
}
```

---

## Step 5: Run Plan

```bash id="x31"
terraform plan
```

Adjust code until no unnecessary changes.

### Best Practice:

Import subnet, route tables, IGW too.

---

## 20. How do you manage secrets in Terraform without hardcoding them?

## Preferred Methods:

### 1. Environment Variables

```bash id="x32"
export TF_VAR_db_password=secret123
```

Terraform:

```hcl id="x33"
variable "db_password" {
  sensitive = true
}
```

---

### 2. Secret Managers

Use:

* Amazon Web Services Secrets Manager
* SSM Parameter Store
* HashiCorp Vault

---

### 3. CI/CD Secret Store

Use secrets in:

* GitHub Actions
* GitLab CI
* Jenkins credentials store

---

### 4. Mark Variables Sensitive

```hcl id="x34"
variable "password" {
  sensitive = true
}
```

### Best Practice:

* Never commit `.tfvars`
* Encrypt backend
* Rotate secrets
* Use IAM roles instead of passwords

---

# Senior Interview Tip Answer Format

For Terraform questions answer like:

**Problem → Terraform Behavior → Business Impact → Recovery → Best Practice**

That makes you sound like a real production engineer.

---

## 21. How would you implement cross-account resource provisioning using Terraform?

Cross-account provisioning means Terraform manages resources in multiple Amazon Web Services accounts (for example Dev, Shared Services, Prod).

## Common Use Case:

* Account A = Terraform execution account
* Account B = Target AWS account

Terraform uses **AssumeRole**.

---

## Architecture Flow:

Terraform user/CI pipeline in Account A assumes IAM role in Account B.

```text
CI/CD -> IAM User/Role -> AssumeRole -> Target AWS Account
```

---

## Terraform Example:

```hcl id="t21a"
provider "aws" {
  region = "ap-south-1"

  assume_role {
    role_arn = "arn:aws:iam::123456789012:role/TerraformRole"
  }
}
```

---

## Multiple Accounts Example:

```hcl id="t21b"
provider "aws" {
  alias  = "dev"
  region = "ap-south-1"

  assume_role {
    role_arn = "arn:aws:iam::111111111111:role/TerraformRole"
  }
}

provider "aws" {
  alias  = "prod"
  region = "ap-south-1"

  assume_role {
    role_arn = "arn:aws:iam::222222222222:role/TerraformRole"
  }
}
```

Use:

```hcl id="t21c"
resource "aws_s3_bucket" "prod_logs" {
  provider = aws.prod
  bucket   = "prod-log-bucket"
}
```

---

## Best Practices:

* Least privilege IAM roles
* Separate state per account
* MFA for admin access
* CI/CD controlled applies
* Audit with CloudTrail

---

## 22. Write Terraform code to create 5 EC2 instances with different names and sizes

Use `for_each`.

```hcl id="t22a"
provider "aws" {
  region = "ap-south-1"
}

locals {
  servers = {
    web1 = "t2.micro"
    web2 = "t2.small"
    app1 = "t3.micro"
    app2 = "t3.small"
    db1  = "t3.medium"
  }
}

resource "aws_instance" "servers" {
  for_each      = local.servers
  ami           = "ami-0abcdef1234567890"
  instance_type = each.value

  tags = {
    Name = each.key
  }
}
```

---

## Output:

Creates:

* web1 → t2.micro
* web2 → t2.small
* app1 → t3.micro
* app2 → t3.small
* db1 → t3.medium

---

## Best Practice:

Use modules + autoscaling for production.

---

## 23. How does Terraform maintain the state of resources?

Terraform tracks infrastructure using a **state file**.

Default:

```bash id="t23a"
terraform.tfstate
```

---

## What State Contains:

* Resource IDs
* Current attributes
* Dependencies
* Metadata
* Outputs

Example:

```json id="t23b"
aws_instance.web = i-123456
```

Terraform uses state to compare:

```text
Desired Config vs Current State
```

Then decides create/update/delete.

---

## State Backends:

* Local file
* S3 backend
* Terraform Cloud
* Consul

---

## Why Important:

Without state Terraform cannot map config to real resources.

---

## Best Practice:

Use remote backend + locking.

---

## 24. What are Terraform modules?

Modules are reusable Terraform code packages.

Think of them like functions in programming.

---

## Example:

Instead of repeating EC2 code many times:

```hcl id="t24a"
module "webserver" {
  source = "./modules/ec2"
}
```

---

## Structure:

```text
modules/
  ec2/
    main.tf
    variables.tf
    outputs.tf
```

---

## Benefits:

* Reusability
* Standardization
* Cleaner code
* Easier maintenance
* Team collaboration

---

## Types:

### Root Module

Current working directory.

### Child Module

Called from root.

### Remote Module

```hcl id="t24b"
source = "terraform-aws-modules/vpc/aws"
```

---

## 25. How to manage sensitive variables in Terraform?

Never hardcode passwords or keys.

---

## Use Sensitive Variables:

```hcl id="t25a"
variable "db_password" {
  type      = string
  sensitive = true
}
```

---

## Pass via Environment Variable:

```bash id="t25b"
export TF_VAR_db_password=MySecret123
```

---

## Use Secret Managers:

* Amazon Web Services Secrets Manager
* SSM Parameter Store
* HashiCorp Vault

---

## Example:

```hcl id="t25c"
data "aws_secretsmanager_secret_version" "db" {
  secret_id = "prod/db/password"
}
```

---

## Best Practice:

* Encrypt remote state
* Restrict access
* Rotate secrets regularly

---

## 26. What is the purpose of terraform validate and terraform fmt?

## terraform validate

Checks syntax and configuration correctness.

```bash id="t26a"
terraform validate
```

Checks:

* Invalid references
* Missing arguments
* Bad syntax

---

## terraform fmt

Formats Terraform files automatically.

```bash id="t26b"
terraform fmt
```

Fixes:

* Alignment
* Indentation
* Readability

---

## Daily Workflow:

```bash id="t26c"
terraform fmt
terraform validate
terraform plan
terraform apply
```

---

## CI/CD Best Practice:

Run both in pull request pipeline.

---

## 27. How do you handle provisioning in different environments (dev/stage/prod)?

Use one reusable codebase with environment separation.

---

## Method 1: tfvars Files

```text
dev.tfvars
stage.tfvars
prod.tfvars
```

Example:

```hcl id="t27a"
instance_type = "t2.micro"
```

Prod:

```hcl id="t27b"
instance_type = "t3.large"
```

Run:

```bash id="t27c"
terraform apply -var-file=prod.tfvars
```

---

## Method 2: Workspaces

```bash id="t27d"
terraform workspace new dev
terraform workspace new prod
```

---

## Method 3: Separate State Backends

Recommended for enterprise.

```text
dev/state.tfstate
prod/state.tfstate
```

---

## Best Practice:

* Separate accounts/subscriptions
* Separate pipelines
* Approval gates for prod

---

## 28. What is drift detection in Terraform and how is it handled?

Drift = infrastructure changed outside Terraform.

Example:
Someone changes EC2 type manually.

---

## Detection:

```bash id="t28a"
terraform plan
```

or

```bash id="t28b"
terraform plan -refresh-only
```

---

## Result:

```text
~ instance_type = t3.medium -> t2.micro
```

---

## Handling:

### Option 1:

Reapply Terraform to restore desired config.

### Option 2:

Update code to accept new change.

---

## Prevention:

* Restrict manual console changes
* Scheduled drift scans
* IaC-only policy

---

## 29. What are taint and untaint in Terraform?

## terraform taint

Marks resource for forced recreation.

```bash id="t29a"
terraform taint aws_instance.web
```

Next apply:

* Destroy old instance
* Recreate new instance

Used when resource is unhealthy.

---

## terraform untaint

Removes tainted status.

```bash id="t29b"
terraform untaint aws_instance.web
```

---

## Modern Preferred Command:

```bash id="t29c"
terraform apply -replace="aws_instance.web"
```

---

## Example Use Cases:

* Corrupted VM
* Failed userdata
* Broken boot disk

---

## 30. What happens if your Terraform state file is accidentally deleted?

State file maps Terraform config to real infrastructure.

If deleted:

## What Happens:

* Terraform forgets existing resources
* Infra still exists in AWS
* Next plan may show resources need creation

This can create duplicates.

---

## Recovery Options:

### 1. Restore Backup

```bash id="t30a"
terraform.tfstate.backup
```

### 2. Remote Backend Recovery

Restore from S3 versioning.

### 3. Import Existing Resources

```bash id="t30b"
terraform import aws_instance.web i-12345
```

---

## Prevention:

* Use remote backend
* Enable versioning
* State locking
* Backups
* Restricted access

---

## Senior Interview Tip

Answer Terraform production questions in this structure:

**Scenario → Terraform Behavior → Risk → Recovery → Best Practice**

That sounds like real-world experience.

---

## 31. How do you manage Terraform state files when working in a team?

When multiple engineers work on the same infrastructure, Terraform state management becomes critical. The state file is the **single source of truth** that maps Terraform configuration to actual resources in Amazon Web Services or another cloud.

---

## Problems with Local State in Teams

If everyone keeps `terraform.tfstate` locally:

* Different team members have different copies
* State becomes outdated
* Simultaneous applies overwrite changes
* Easy to lose state file
* No audit trail

---

## Best Practice: Remote Backend

Use centralized remote storage.

### Common Options:

* Amazon Web Services S3 + DynamoDB locking
* HashiCorp Terraform Cloud
* Azure Storage backend
* GCS backend

---

## Example: S3 Backend

```hcl id="q31a"
terraform {
  backend "s3" {
    bucket         = "company-terraform-state"
    key            = "prod/network/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

---

## Why This Helps

### Centralized State

Everyone reads same state file.

### Locking

Only one `terraform apply` at a time.

### Security

Encrypted storage + IAM access control.

### Backup

S3 versioning restores deleted/corrupted state.

---

## Team Workflow

```bash id="q31b"
terraform init
terraform plan
terraform apply
```

All users operate against same backend.

---

## Enterprise Best Practices

### Separate State Per Environment

```text id="twm9w1"
dev/app.tfstate
stage/app.tfstate
prod/app.tfstate
```

### CI/CD Controlled Apply

Only pipelines run `apply`, engineers run `plan`.

### Access Control

* Read-only for developers
* Apply permissions for DevOps admins

### Naming Convention

```text id="4e2j4d"
project/environment/component.tfstate
```

---

## Interview Answer Summary

“In teams, I never use local state. I store state remotely, enable locking, separate environments, encrypt state, and restrict apply actions through CI/CD pipelines.”

---

# 32. What approach do you take to avoid or resolve state file conflicts in Terraform?

State conflicts happen when multiple users or pipelines update state simultaneously.

---

## Example Conflict Scenario

Engineer A:

```bash id="q32a"
terraform apply
```

Engineer B runs apply same time.

Possible issues:

* State lock errors
* Overwritten updates
* Partial state corruption

---

## How I Avoid Conflicts

## 1. Enable State Locking

Use backend locking.

### Example with S3 + DynamoDB:

```hcl id="q32b"
backend "s3" {
  bucket         = "tf-state"
  key            = "prod.tfstate"
  region         = "ap-south-1"
  dynamodb_table = "terraform-locks"
}
```

When one user runs apply:

```text id="q32c"
Acquiring state lock...
```

Second user waits or gets blocked.

---

## 2. CI/CD Only Apply Model

Developers run:

```bash id="q32d"
terraform plan
```

Only pipeline runs:

```bash id="q32e"
terraform apply
```

This eliminates human collision.

---

## 3. Split Large State Files

Avoid one giant state.

Instead use:

```text id="55q7gh"
network.tfstate
security.tfstate
app.tfstate
database.tfstate
```

Reduces contention.

---

## 4. Use Separate Workspaces / Environments

```bash id="q32f"
terraform workspace select dev
terraform workspace select prod
```

---

## How I Resolve Conflicts

## If Lock Stuck:

```bash id="q32g"
terraform force-unlock LOCK_ID
```

Use only after confirming no active apply.

---

## If State Diverged:

```bash id="q32h"
terraform state pull
terraform plan
```

Then reconcile resources.

---

## If Corrupted:

Restore previous backend version or backup.

---

## Interview Summary

“My strategy is remote backend + locking + CI/CD applies + smaller state boundaries. If conflicts occur, I inspect locks, reconcile state safely, and restore backups if needed.”

---

# 33. Can you write a Terraform configuration to provision multiple S3 buckets?

Yes — best way is using `for_each`.

---

## Example: Create Multiple S3 Buckets

```hcl id="q33a"
provider "aws" {
  region = "ap-south-1"
}

locals {
  bucket_names = [
    "company-dev-logs-bucket",
    "company-stage-backup-bucket",
    "company-prod-assets-bucket"
  ]
}

resource "aws_s3_bucket" "buckets" {
  for_each = toset(local.bucket_names)

  bucket = each.value

  tags = {
    Environment = split("-", each.value)[1]
    ManagedBy   = "Terraform"
  }
}
```

---

## What This Creates

* company-dev-logs-bucket
* company-stage-backup-bucket
* company-prod-assets-bucket

---

## Better Production Version with Versioning

```hcl id="q33b"
resource "aws_s3_bucket_versioning" "versioning" {
  for_each = aws_s3_bucket.buckets

  bucket = each.value.id

  versioning_configuration {
    status = "Enabled"
  }
}
```

---

## Better Production Version with Encryption

```hcl id="q33c"
resource "aws_s3_bucket_server_side_encryption_configuration" "enc" {
  for_each = aws_s3_bucket.buckets

  bucket = each.value.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
```

---

## Enterprise Best Practices

* Enable versioning
* Enable encryption
* Block public access
* Lifecycle policies
* Bucket naming standards
* Separate logging bucket

---

## Interview Tip

Say:

“I’d use `for_each` instead of repeating bucket resources manually. It keeps the code scalable, cleaner, and easier to maintain.”

---

## Top 40 Terraform Coding Interview Questions with Hands-on Answers

---

# 1. Write Terraform code to create an EC2 instance

```hcl id="tf1"
provider "aws" {
  region = "ap-south-1"
}

resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"

  tags = {
    Name = "web-server"
  }
}
```

---

# 2. Create multiple EC2 instances using count

```hcl id="tf2"
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-12345678"
  instance_type = "t2.micro"

  tags = {
    Name = "server-${count.index}"
  }
}
```

---

# 3. Create multiple EC2 instances using for_each

```hcl id="tf3"
resource "aws_instance" "web" {
  for_each = {
    app1 = "t2.micro"
    app2 = "t2.small"
  }

  ami           = "ami-12345678"
  instance_type = each.value

  tags = {
    Name = each.key
  }
}
```

---

# 4. Create an S3 bucket

```hcl id="tf4"
resource "aws_s3_bucket" "bucket" {
  bucket = "my-prod-bucket-demo"
}
```

---

# 5. Create multiple S3 buckets

```hcl id="tf5"
resource "aws_s3_bucket" "buckets" {
  for_each = toset(["dev-bucket", "stage-bucket", "prod-bucket"])
  bucket   = each.value
}
```

---

# 6. Create a VPC

```hcl id="tf6"
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}
```

---

# 7. Create subnet inside VPC

```hcl id="tf7"
resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
}
```

---

# 8. Create Internet Gateway

```hcl id="tf8"
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
}
```

---

# 9. Create Route Table

```hcl id="tf9"
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
}
```

---

# 10. Create Security Group

```hcl id="tf10"
resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---

# 11. Output EC2 Public IP

```hcl id="tf11"
output "public_ip" {
  value = aws_instance.web.public_ip
}
```

---

# 12. Create variables

```hcl id="tf12"
variable "instance_type" {
  default = "t2.micro"
}
```

---

# 13. Use variables

```hcl id="tf13"
resource "aws_instance" "web" {
  ami           = "ami-123"
  instance_type = var.instance_type
}
```

---

# 14. Use locals

```hcl id="tf14"
locals {
  env = "prod"
}
```

---

# 15. Conditional resource creation

```hcl id="tf15"
resource "aws_instance" "web" {
  count = var.env == "prod" ? 1 : 0
}
```

---

# 16. Conditional instance size

```hcl id="tf16"
instance_type = var.env == "prod" ? "t3.large" : "t2.micro"
```

---

# 17. Create IAM user

```hcl id="tf17"
resource "aws_iam_user" "user" {
  name = "devops-user"
}
```

---

# 18. Create multiple IAM users

```hcl id="tf18"
resource "aws_iam_user" "users" {
  for_each = toset(["raj", "john", "admin"])
  name     = each.value
}
```

---

# 19. Create EBS Volume

```hcl id="tf19"
resource "aws_ebs_volume" "disk" {
  availability_zone = "ap-south-1a"
  size              = 20
}
```

---

# 20. Attach EBS to EC2

```hcl id="tf20"
resource "aws_volume_attachment" "attach" {
  device_name = "/dev/sdf"
  volume_id   = aws_ebs_volume.disk.id
  instance_id = aws_instance.web.id
}
```

---

# 21. Create Key Pair

```hcl id="tf21"
resource "aws_key_pair" "key" {
  key_name   = "mykey"
  public_key = file("id_rsa.pub")
}
```

---

# 22. Read file in Terraform

```hcl id="tf22"
file("userdata.sh")
```

---

# 23. Use User Data Script

```hcl id="tf23"
resource "aws_instance" "web" {
  user_data = file("userdata.sh")
}
```

---

# 24. Create module call

```hcl id="tf24"
module "ec2" {
  source = "./modules/ec2"
}
```

---

# 25. Create backend config

```hcl id="tf25"
terraform {
  backend "s3" {
    bucket = "tfstate-bucket"
    key    = "prod.tfstate"
    region = "ap-south-1"
  }
}
```

---

# 26. Sensitive variable

```hcl id="tf26"
variable "password" {
  sensitive = true
}
```

---

# 27. Create workspace

```bash id="tf27"
terraform workspace new dev
terraform workspace new prod
```

---

# 28. Dynamic block example

```hcl id="tf28"
dynamic "ingress" {
  for_each = [22, 80]

  content {
    from_port = ingress.value
    to_port   = ingress.value
    protocol  = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---

# 29. Create data source AMI lookup

```hcl id="tf29"
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
}
```

---

# 30. Use data source in EC2

```hcl id="tf30"
ami = data.aws_ami.amazon_linux.id
```

---

# 31. Create NAT Gateway

```hcl id="tf31"
resource "aws_nat_gateway" "nat" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public.id
}
```

---

# 32. Create Elastic IP

```hcl id="tf32"
resource "aws_eip" "nat" {
  domain = "vpc"
}
```

---

# 33. Import existing resource command

```bash id="tf33"
terraform import aws_instance.web i-123456
```

---

# 34. Replace damaged resource

```bash id="tf34"
terraform apply -replace="aws_instance.web"
```

---

# 35. Target specific resource

```bash id="tf35"
terraform apply -target=aws_instance.web
```

---

# 36. Create RDS instance

```hcl id="tf36"
resource "aws_db_instance" "db" {
  allocated_storage = 20
  engine            = "mysql"
  instance_class    = "db.t3.micro"
  username          = "admin"
  password          = "Password123"
}
```

---

# 37. Create tags using map

```hcl id="tf37"
tags = {
  Env   = "Prod"
  Owner = "DevOps"
}
```

---

# 38. Use depends_on

```hcl id="tf38"
resource "aws_instance" "web" {
  depends_on = [aws_internet_gateway.igw]
}
```

---

# 39. Terraform loop with list

```hcl id="tf39"
locals {
  names = ["web1", "web2", "web3"]
}
```

---

# 40. Null resource example

```hcl id="tf40"
resource "null_resource" "run" {
  provisioner "local-exec" {
    command = "echo Hello"
  }
}
```

---

# Real Interview Commands You Must Know

```bash id="tfcmd"
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply
terraform destroy
terraform state list
terraform output
terraform import
```

---

# Senior Interview Tip

If interviewer gives coding challenge:

### Use This Structure:

1. Provider
2. Variables
3. Resource
4. Tags
5. Output
6. Reusability

---

# Most Asked Live Coding Questions

* Create EC2 + SG
* Create VPC + Public/Private subnet
* Create 5 instances with loop
* Create reusable module
* Remote backend config
* Import existing infra
* Multi-environment setup
* IAM users with for_each

---

## Top 50 Advanced Terraform Scenario Questions for 7+ Years DevOps Interviews

---

# State Management Scenarios

## 1.

# Your Terraform state file is corrupted during a production deployment. How do you recover safely?

If a Terraform state file gets corrupted during a production deployment, the recovery process must prioritize:

* Preventing further infrastructure damage
* Restoring state consistency
* Avoiding accidental resource recreation/deletion
* Preserving production uptime

Here’s the step-by-step recovery approach used in real enterprise environments.

---

# 1. Immediately Stop All Terraform Operations

First action:

```bash
Ctrl + C
```

Then ensure nobody else runs Terraform.

If using remote state:

* Lock the backend
* Disable CI/CD pipeline temporarily
* Restrict concurrent access

For example in:

* HashiCorp Terraform Cloud
* Azure Storage backend
* AWS S3 + DynamoDB locking

Preventing parallel operations is critical.

---

# 2. Identify the Type of Corruption

Common scenarios:

| Problem                 | Symptoms                          |
| ----------------------- | --------------------------------- |
| Partial write           | JSON parse error                  |
| State lock issue        | “Error acquiring state lock”      |
| Missing resources       | Terraform wants to recreate infra |
| Backend corruption      | Cannot download state             |
| Manual edits broke JSON | Invalid syntax                    |
| Interrupted apply       | Drift between infra and state     |

Run:

```bash
terraform state list
```

and

```bash
terraform plan
```

Carefully inspect errors.

---

# 3. Restore From State Backup

Terraform automatically creates backups locally:

```bash
terraform.tfstate.backup
```

Check for:

* Latest valid backup
* Remote backend version history
* CI/CD artifacts

---

## Example Recovery

### Replace corrupted state

```bash
cp terraform.tfstate.backup terraform.tfstate
```

Then validate:

```bash
terraform state list
```

---

# 4. Recover Remote Backend State

## AWS S3 Backend

If using versioning:

* Open S3 bucket
* Restore previous object version

Also verify DynamoDB lock table.

---

## Azure Storage Backend

If using:

* Blob versioning
* Soft delete
* Snapshots

Restore previous blob version.

This is one reason enterprise Azure environments enable:

* Blob versioning
* Immutable storage
* Soft delete

---

# 5. Validate Infrastructure Before Apply

NEVER immediately run:

```bash
terraform apply
```

Instead:

```bash
terraform refresh
```

or in modern versions:

```bash
terraform plan -refresh-only
```

This checks real infrastructure vs state.

---

# 6. Compare Real Resources vs State

Validate cloud resources manually.

For example in:

* Microsoft Azure
* Amazon AWS

Check:

* VMs
* Networking
* Load balancers
* Databases
* Kubernetes clusters
* Public IPs

Ensure production resources still exist.

---

# 7. Import Missing Resources (If Needed)

If resources exist in cloud but not in state:

```bash
terraform import azurerm_resource_group.rg /subscriptions/xxx/resourceGroups/prod-rg
```

or AWS:

```bash
terraform import aws_instance.web i-123456
```

This avoids destructive recreation.

---

# 8. Use State Commands Carefully

Useful recovery commands:

## Remove broken entries

```bash
terraform state rm <resource>
```

## Move state resources

```bash
terraform state mv
```

## Pull remote state

```bash
terraform state pull
```

## Push recovered state

```bash
terraform state push terraform.tfstate
```

Use `state push` cautiously in production.

---

# 9. Run a Safe Plan

After recovery:

```bash
terraform plan
```

Expected result:

```text
No changes. Infrastructure is up-to-date.
```

If Terraform wants to recreate critical resources:

* STOP
* Investigate drift/state mismatch

Never blindly apply.

---

# 10. Re-enable CI/CD Carefully

Once validated:

* Re-enable pipelines
* Unlock backend
* Inform team
* Document incident

---

# Enterprise Best Practices to Prevent Future Corruption

## Use Remote Backend

Never rely on local state in production.

Use:

* Azure Storage Account backend
* S3 backend
* Terraform Cloud

---

## Enable State Locking

Examples:

* DynamoDB locking for S3
* Native locking in Terraform Cloud

Prevents concurrent writes.

---

## Enable Versioning

Critical for recovery.

### Azure

* Blob versioning
* Soft delete

### AWS

* S3 versioning

---

## Store State Securely

State contains sensitive data:

* Passwords
* Keys
* Secrets
* Connection strings

Use:

* Encryption at rest
* RBAC/IAM
* Key Vault/KMS integration

---

# Real Production Recovery Strategy

In enterprise environments, the safest sequence is:

```text
1. Stop all deployments
2. Backup corrupted state
3. Restore previous version
4. Validate infrastructure
5. Refresh/import missing resources
6. Run plan
7. Apply only if absolutely safe
```

---

# Interview-Ready Answer

> If Terraform state gets corrupted during production deployment, I first stop all Terraform operations and lock the backend to avoid further corruption. Then I identify whether the issue is due to partial writes, backend issues, or state drift. I restore the latest valid backup or previous backend version, validate the infrastructure manually, and use terraform refresh or plan -refresh-only to reconcile state with actual resources. If resources exist but are missing from state, I use terraform import instead of recreating them. Finally, I run a safe terraform plan and ensure no destructive changes are proposed before resuming deployments. In production, I always use remote backends with versioning, encryption, and state locking to minimize such risks.

# 2. Multiple engineers are modifying the same infrastructure. How do you prevent concurrent state conflicts?

The best way to prevent concurrent Terraform state conflicts is to use:

* Remote backend
* State locking
* CI/CD controlled deployments
* RBAC/IAM restrictions

---

## Recommended Architecture

### AWS

* S3 bucket → stores state
* DynamoDB table → provides state locking

Example:

```hcl id="r1"
terraform {
  backend "s3" {
    bucket         = "prod-terraform-state"
    key            = "network/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

---

## How Locking Works

When Engineer A runs:

```bash id="r2"
terraform apply
```

Terraform creates a lock entry in DynamoDB.

If Engineer B tries another apply:

```text id="r3"
Error acquiring the state lock
```

This prevents simultaneous writes.

---

## Additional Enterprise Controls

### CI/CD Only Deployment

Prevent local applies.

Use:

* GitHub Actions
* Microsoft Azure DevOps
* HashiCorp Terraform Cloud

---

## Branch Protection

Require:

* Pull requests
* Code review
* Plan approval

---

## Interview Answer

> I prevent concurrent Terraform conflicts by using a remote backend with state locking enabled. In AWS, I use S3 for state storage and DynamoDB for locking. I also enforce CI/CD-driven deployments instead of local applies, implement RBAC permissions, and use pull-request approvals to ensure serialized infrastructure changes.

---

# 3. A developer accidentally deleted the remote S3 state bucket. What steps would you take?

This is a critical production incident.

---

# Recovery Steps

## Step 1 — Stop All Terraform Operations

Immediately stop:

* Pipelines
* Applies
* Automation jobs

---

## Step 2 — Check S3 Versioning

If versioning was enabled:

Restore deleted objects.

AWS keeps previous versions recoverable.

---

## Step 3 — Restore Bucket

Recreate:

* Bucket
* Policies
* Encryption
* Lifecycle rules

---

## Step 4 — Recover State File

Possible recovery sources:

| Source          | Recovery Option          |
| --------------- | ------------------------ |
| S3 versioning   | Restore previous version |
| CI/CD artifacts | Download backup          |
| Local machine   | terraform.tfstate.backup |
| Terraform Cloud | Workspace history        |

---

## Step 5 — Reconfigure Backend

```bash id="r4"
terraform init
```

---

## Step 6 — Validate Infrastructure

Run:

```bash id="r5"
terraform plan -refresh-only
```

Ensure Terraform does not want to recreate production infra.

---

## Prevention

Always enable:

* S3 versioning
* MFA delete
* Backup policies
* Restricted delete permissions

---

## Interview Answer

> I would immediately stop all Terraform operations, restore the S3 bucket using versioning or backup recovery, recover the latest valid state file, and reinitialize the backend. Then I would validate infrastructure consistency using terraform plan -refresh-only before allowing deployments again. In production, I always enable versioning, encryption, and restricted IAM policies on Terraform state buckets.

---

# 4. Terraform state contains resources that no longer exist in AWS. How do you reconcile it?

This is state drift.

---

# Solution

## Detect Drift

```bash id="r6"
terraform plan
```

Terraform may show:

```text id="r7"
resource will be created
```

because it exists in state but not in AWS.

---

# Decide Desired Outcome

## Case 1 — Resource SHOULD Exist

Recreate it:

```bash id="r8"
terraform apply
```

---

## Case 2 — Resource Was Intentionally Deleted

Remove from state:

```bash id="r9"
terraform state rm aws_instance.web
```

Then update code accordingly.

---

# Refresh State

```bash id="r10"
terraform refresh
```

or

```bash id="r11"
terraform plan -refresh-only
```

---

## Interview Answer

> I first identify whether the deleted resource should still exist. If it is required, I allow Terraform to recreate it. If deletion was intentional, I remove the resource from Terraform state using terraform state rm and update the codebase accordingly. I then refresh the state and validate with terraform plan to ensure consistency.

---

# 5. You need to split one large monolithic state file into smaller states. How would you do it?

This is common in large enterprises.

---

# Why Split State?

Benefits:

* Faster plans
* Reduced blast radius
* Team isolation
* Better security
* Independent deployments

---

# Typical Split

| State          | Resources    |
| -------------- | ------------ |
| network-state  | VPC, subnets |
| security-state | IAM, SG      |
| app-state      | EC2, ECS     |
| database-state | RDS          |

---

# Migration Process

## Step 1 — Create New Terraform Projects

Separate folders/modules.

---

## Step 2 — Move Resources

Use:

```bash id="r12"
terraform state mv
```

Example:

```bash id="r13"
terraform state mv aws_vpc.main module.network.aws_vpc.main
```

---

## Step 3 — Create Separate Backends

Each state gets:

* Independent S3 key
* Independent locking

---

## Step 4 — Validate

Run:

```bash id="r14"
terraform plan
```

for each state.

---

# Important

Never:

* Copy-paste state manually
* Move resources without backup

---

## Interview Answer

> I split monolithic Terraform states by grouping infrastructure into logical domains such as networking, security, applications, and databases. I create separate backends and use terraform state mv to safely migrate resources into new states without recreating infrastructure. This reduces blast radius, improves performance, and enables team-level ownership.

---

# 6. How would you migrate Terraform local state to remote backend with zero downtime?

---

# Steps

## Step 1 — Configure Backend

```hcl id="r15"
terraform {
  backend "s3" {
    bucket = "terraform-prod-state"
    key    = "prod/terraform.tfstate"
    region = "ap-south-1"
  }
}
```

---

## Step 2 — Initialize Migration

```bash id="r16"
terraform init
```

Terraform asks:

```text id="r17"
Do you want to copy existing state?
```

Choose:

```text id="r18"
yes
```

---

## Step 3 — Verify

```bash id="r19"
terraform state list
```

---

## Step 4 — Run Plan

```bash id="r20"
terraform plan
```

Expected:

```text id="r21"
No changes
```

---

# Zero Downtime Reason

Only state location changes.
Infrastructure is untouched.

---

## Interview Answer

> To migrate local Terraform state to a remote backend safely, I configure the backend and run terraform init to migrate the existing state automatically. Since only the state storage changes and no infrastructure changes occur, there is no downtime. I then validate with terraform plan to ensure the migration succeeded.

---

# 7. How do you securely store and protect Terraform state containing secrets?

Terraform state may contain:

* Passwords
* Keys
* Tokens
* DB connection strings

This is a major security concern.

---

# Security Best Practices

## Encrypt State

### AWS

Enable:

* S3 SSE-KMS encryption

### Azure

Use:

* Storage Account encryption

---

## Restrict IAM Access

Least privilege only.

Only:

* Terraform pipeline
* Infra admins

should access state.

---

## Enable Versioning

Protect against:

* Accidental deletion
* Corruption

---

## Use Remote Backend

Never commit state to Git.

---

## Store Secrets Outside State

Use:

* Amazon Secrets Manager
* Microsoft Key Vault
* HashiCorp Vault

Avoid plaintext variables.

---

## Sensitive Variables

```hcl id="r22"
sensitive = true
```

---

## Interview Answer

> I secure Terraform state using encrypted remote backends such as S3 with KMS encryption or Azure Storage with encryption enabled. I restrict IAM access using least privilege, enable versioning and backups, and avoid storing secrets directly in Terraform variables whenever possible by integrating with Vault, Secrets Manager, or Key Vault.

---

# 8. You lost access to DynamoDB lock table used for Terraform locking. What happens and how do you fix it?

---

# What Happens?

Terraform cannot acquire lock:

```text id="r23"
Error acquiring the state lock
```

Applies fail.

This protects against concurrent modification.

---

# Fix Steps

## Step 1 — Verify IAM Permissions

Check:

* dynamodb:GetItem
* PutItem
* DeleteItem

---

## Step 2 — Restore Table Access

Possible issues:

* Deleted table
* Wrong IAM role
* Region mismatch

---

## Step 3 — Recreate Table If Needed

```hcl id="r24"
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

---

# Emergency Only

You can bypass locking:

```bash id="r25"
terraform apply -lock=false
```

But this is risky in production.

---

## Interview Answer

> If DynamoDB locking becomes unavailable, Terraform cannot safely acquire state locks and applies fail. I would first verify IAM permissions and table availability, then restore or recreate the lock table. I avoid using -lock=false in production unless absolutely necessary because it risks state corruption from concurrent writes.

---

# 9. How do you rotate backend credentials without breaking Terraform pipelines?

---

# Best Practice

Never hardcode credentials.

Use:

* IAM roles
* OIDC federation
* Managed identities

---

# Rotation Strategy

## Step 1 — Create New Credentials

Do not remove old credentials yet.

---

## Step 2 — Update CI/CD Secrets

Examples:

* GitHub Secrets
* Microsoft Variable Groups

---

## Step 3 — Test Pipeline

Run:

* terraform init
* terraform plan

---

## Step 4 — Remove Old Credentials

After validation.

---

# Enterprise Recommendation

Prefer short-lived credentials:

* AWS AssumeRole
* OIDC
* Azure Managed Identity

---

## Interview Answer

> I rotate Terraform backend credentials by introducing new credentials alongside existing ones, updating CI/CD secret stores, validating pipeline execution, and then removing old credentials after successful testing. In enterprise environments, I prefer IAM roles, OIDC, or managed identities over static credentials.

---

# 10. How do you recover if someone manually edited the state file incorrectly?

---

# Recovery Steps

## Step 1 — Stop Terraform Operations

Prevent additional corruption.

---

## Step 2 — Restore Backup

Use:

* terraform.tfstate.backup
* S3 version history
* Blob snapshots

---

## Step 3 — Validate State

```bash id="r26"
terraform state list
```

---

## Step 4 — Refresh Infrastructure

```bash id="r27"
terraform plan -refresh-only
```

---

## Step 5 — Import Missing Resources

If necessary:

```bash id="r28"
terraform import
```

---

# Prevention

Never manually edit state unless absolutely required.

Use:

* terraform state mv
* terraform state rm
* terraform import

instead.

---

## Interview Answer

> If someone manually edits Terraform state incorrectly, I stop deployments immediately, restore the latest valid backup, validate state integrity, and reconcile infrastructure using refresh-only plans and imports if necessary. Manual state editing should be avoided unless no safer Terraform state commands are available.

---

# 21. Your Terraform codebase has thousands of lines duplicated. How would you refactor it?

Large duplicated Terraform codebases become difficult to:

* Maintain
* Scale
* Audit
* Secure

The solution is modularization.

---

# Refactoring Strategy

## Step 1 — Identify Repeated Patterns

Typical duplicates:

* VPC creation
* EC2 instances
* IAM roles
* Security groups
* RDS databases

---

## Step 2 — Create Reusable Modules

Structure:

```text id="m1"
modules/
 ├── vpc/
 ├── ec2/
 ├── rds/
 ├── iam/
```

---

## Step 3 — Parameterize Modules

Example:

```hcl id="m2"
module "web" {
  source        = "./modules/ec2"
  instance_type = "t3.medium"
  subnet_id     = var.subnet_id
}
```

---

## Step 4 — Use Locals & for_each

Reduce repeated blocks:

```hcl id="m3"
for_each = var.environments
```

---

## Step 5 — Standardize Naming & Tags

Centralize:

* Naming conventions
* Tags
* Security policies

---

# Enterprise Benefits

* DRY principle
* Easier upgrades
* Consistency
* Reduced errors
* Faster onboarding

---

## Interview Answer

> I refactor duplicated Terraform code by identifying repeated infrastructure patterns and converting them into reusable modules. I parameterize variables, use locals and for_each to eliminate repetitive blocks, and separate infrastructure logically into modules such as VPC, EC2, IAM, and RDS. This improves maintainability, consistency, scalability, and reduces operational risk.

---

# 22. How do you design reusable modules for VPC, EC2, RDS, IAM?

---

# Module Design Principles

Good modules should be:

* Reusable
* Minimal
* Flexible
* Secure
* Versioned

---

# Example Structure

```text id="m4"
modules/
 ├── vpc/
 │    ├── main.tf
 │    ├── variables.tf
 │    ├── outputs.tf
```

---

# VPC Module Example

Inputs:

```hcl id="m5"
variable "cidr_block" {}
variable "public_subnets" {}
variable "private_subnets" {}
```

Outputs:

```hcl id="m6"
output "vpc_id" {}
output "private_subnet_ids" {}
```

---

# EC2 Module

Should support:

* AMI
* Instance type
* Security groups
* User data
* IAM profile
* Tags

---

# RDS Module

Should enforce:

* Encryption
* Backup retention
* Multi-AZ
* Parameter groups

---

# IAM Module

Reusable policies/roles.

Avoid overly permissive policies.

---

# Enterprise Best Practices

* Keep modules focused
* Avoid hardcoding
* Document inputs/outputs
* Include examples
* Add validations

---

## Interview Answer

> I design Terraform modules as small reusable building blocks with clear inputs, outputs, validations, and minimal assumptions. For example, VPC modules expose subnet and VPC IDs, EC2 modules support configurable instance parameters, and RDS modules enforce security defaults like encryption and backups. I keep modules loosely coupled and version-controlled for enterprise reuse.

---

# 23. A shared module change broke multiple teams. How do you prevent this in future?

This is a common enterprise issue.

---

# Prevention Strategy

## Use Semantic Versioning

Example:

```text id="m7"
v1.2.0
```

Never auto-consume latest modules.

---

# Pin Module Versions

```hcl id="m8"
module "vpc" {
  source  = "git::ssh://repo/modules/vpc"
  version = "1.2.0"
}
```

---

# Backward Compatibility

Avoid breaking:

* Variables
* Outputs
* Naming conventions

---

# CI/CD Testing

Every module change should trigger:

* terraform validate
* terraform plan
* Security scan
* Integration tests

---

# Release Process

Use:

* Pull requests
* Approvals
* Changelog
* Release notes

---

# Separate Environments

Test in:

* Dev
* Sandbox
  before production rollout.

---

## Interview Answer

> To prevent shared module failures, I use semantic versioning, pin module versions explicitly, and avoid auto-upgrading modules. All module updates go through CI/CD validation, integration testing, and approval workflows before release. Breaking changes are introduced only through major versions with proper documentation and migration guidance.

---

# 24. How do you version Terraform modules in enterprise environments?

---

# Recommended Approach

Use:

* Git tags
* Semantic versioning
* Private module registry

---

# Semantic Versioning

```text id="m9"
MAJOR.MINOR.PATCH
```

Example:

| Version | Meaning         |
| ------- | --------------- |
| 1.0.1   | Bug fix         |
| 1.1.0   | New feature     |
| 2.0.0   | Breaking change |

---

# Module Source Example

```hcl id="m10"
module "network" {
  source = "git::ssh://git.company.com/modules/vpc.git?ref=v1.4.0"
}
```

---

# Enterprise Module Registry

Common choices:

* HashiCorp Terraform Cloud Registry
* GitHub Enterprise
* Artifactory

---

# Governance

Maintain:

* Documentation
* Release notes
* Approval process
* Security validation

---

## Interview Answer

> In enterprise environments, I version Terraform modules using semantic versioning with Git tags and maintain them in a private module registry or Git repository. Teams pin specific module versions to avoid unexpected breaking changes, and all releases go through testing, approvals, and documented release management processes.

---

# 25. How do you test modules before publishing for production use?

---

# Testing Layers

## 1. Syntax Validation

```bash id="m11"
terraform validate
```

---

## 2. Formatting

```bash id="m12"
terraform fmt
```

---

## 3. Security Scanning

Tools:

* Checkov
* TFSec
* Terrascan

---

## 4. Plan Validation

```bash id="m13"
terraform plan
```

Review:

* Resource changes
* Naming
* Security settings

---

## 5. Integration Testing

Deploy into:

* Sandbox
* Dev AWS account

Validate:

* Networking
* IAM
* Connectivity

---

## 6. Automated Testing

Tools:

* Terratest
* Kitchen-Terraform

---

# Enterprise Pipeline

```text id="m14"
Commit → Validate → Scan → Test → Approve → Publish
```

---

## Interview Answer

> Before publishing Terraform modules, I validate syntax and formatting, run security scans using tools like Checkov or TFSec, execute Terraform plans, and deploy the module into sandbox environments for integration testing. In enterprise environments, I automate this process in CI/CD pipelines and publish modules only after approval and successful validation.

---

# 26. How would you integrate Terraform into Jenkins pipeline for production deployment?

---

# Jenkins Pipeline Flow

```text id="m15"
Git Commit
   ↓
Jenkins Trigger
   ↓
Terraform Init
   ↓
Terraform Validate
   ↓
Terraform Plan
   ↓
Manual Approval
   ↓
Terraform Apply
```

---

# Example Jenkinsfile

```groovy id="m16"
pipeline {
  agent any

  stages {
    stage('Init') {
      steps {
        sh 'terraform init'
      }
    }

    stage('Validate') {
      steps {
        sh 'terraform validate'
      }
    }

    stage('Plan') {
      steps {
        sh 'terraform plan -out=tfplan'
      }
    }

    stage('Approval') {
      steps {
        input 'Approve Production Deployment?'
      }
    }

    stage('Apply') {
      steps {
        sh 'terraform apply tfplan'
      }
    }
  }
}
```

---

# Enterprise Enhancements

Include:

* Remote backend
* Secrets manager integration
* Role assumption
* Security scanning
* Drift detection

---

## Interview Answer

> I integrate Terraform into Jenkins using a staged pipeline that performs terraform init, validate, plan, approval, and apply steps. Production deployments require manual approval, and the pipeline uses remote state backends, IAM role assumption, and secrets management to ensure secure automated infrastructure delivery.

---

# 27. How do you implement Terraform plan approval workflow in GitHub Actions?

---

# Workflow Design

## Pull Request Flow

```text id="m17"
PR Created
   ↓
Terraform Plan
   ↓
Plan Output Posted
   ↓
Reviewer Approval
   ↓
Merge
   ↓
Terraform Apply
```

---

# GitHub Actions Example

```yaml id="m18"
name: Terraform

on:
  pull_request:

jobs:
  terraform:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Terraform Init
        run: terraform init

      - name: Terraform Plan
        run: terraform plan
```

---

# Production Protection

Use:

* Protected branches
* Required reviewers
* Environment approvals

---

## Interview Answer

> I implement Terraform approvals in GitHub Actions by generating Terraform plans during pull requests and requiring reviewer approval before merge. The apply step executes only after approved changes are merged into protected branches. This ensures visibility, governance, and controlled production deployments.

---

# 28. How do you run Terraform automatically when code changes in a repo?

---

# CI/CD Trigger

Examples:

* GitHub Actions
* Jenkins
* Azure DevOps

---

# Workflow

```text id="m19"
Code Commit
   ↓
Pipeline Trigger
   ↓
Terraform Validate
   ↓
Terraform Plan
   ↓
Optional Apply
```

---

# GitHub Example

```yaml id="m20"
on:
  push:
    branches:
      - main
```

---

# Best Practice

Automatic apply:

* Dev only

Manual approval:

* Stage/Prod

---

## Interview Answer

> I configure CI/CD pipelines to trigger automatically on Git commits or pull requests. The pipeline runs Terraform validation and planning automatically, while production applies require approval. This enables fast infrastructure automation while maintaining governance controls.

---

# 29. How do you manage secrets for Terraform in CI/CD pipelines?

---

# Never Store Secrets In

* Git repos
* tfvars files
* Jenkinsfiles

---

# Recommended Solutions

Use:

* Amazon Secrets Manager
* Microsoft Key Vault
* HashiCorp Vault

---

# CI/CD Secret Injection

Examples:

* GitHub Secrets
* Jenkins Credentials
* Azure DevOps Library

---

# Runtime Retrieval

Example:

```hcl id="m21"
data "aws_secretsmanager_secret_version" "db" {
  secret_id = "prod-db-password"
}
```

---

# Additional Security

* Short-lived credentials
* OIDC
* Masked logs
* RBAC

---

## Interview Answer

> I manage Terraform secrets by integrating CI/CD pipelines with secure secret stores such as AWS Secrets Manager, Azure Key Vault, or HashiCorp Vault. Secrets are injected dynamically during runtime rather than stored in code or tfvars files. I also use short-lived credentials, masked logs, and least privilege access controls.

---

# 30. How do you rollback a failed Terraform deployment in pipeline?

Terraform does not provide traditional rollback.

Recovery depends on infrastructure state.

---

# Safe Recovery Strategy

## Step 1 — Stop Further Applies

Pause pipelines immediately.

---

## Step 2 — Analyze Failure

Check:

* Partial resource creation
* State consistency
* Cloud console status

---

## Step 3 — Restore Previous Code Version

Git revert:

```bash id="m22"
git revert <commit>
```

---

## Step 4 — Re-run Terraform

```bash id="m23"
terraform apply
```

Terraform reconciles infrastructure.

---

# Important

Never manually delete resources blindly.

---

# Enterprise Best Practice

Use:

* Blue/Green deployments
* Immutable infrastructure
* Canary releases

---

## Interview Answer

> Terraform rollback is typically achieved by reverting the infrastructure code to the last stable version and reapplying it. I first stop pipelines, analyze partial changes, restore the previous Git commit, and run Terraform again to reconcile infrastructure safely. For critical environments, I prefer blue-green or immutable deployment strategies to minimize rollback risk.

---

# 31. How do you prevent secrets from being hardcoded in Terraform code?

---

# Best Practices

## Use Secret Managers

* Amazon Secrets Manager
* HashiCorp Vault
* Microsoft Key Vault

---

## Use Environment Variables

```bash id="m24"
export TF_VAR_db_password=*****
```

---

## Sensitive Variables

```hcl id="m25"
sensitive = true
```

---

## Prevent Git Exposure

Use:

* .gitignore
* Secret scanning
* Pre-commit hooks

---

## Interview Answer

> I prevent hardcoded secrets by integrating Terraform with secure secret management systems like AWS Secrets Manager, Vault, or Azure Key Vault. Secrets are injected dynamically through environment variables or data sources, marked sensitive, and excluded from Git repositories using proper controls and scanning tools.

# 41. Terraform apply partially succeeded and failed halfway. What next?

This is a very common production scenario.

Terraform may have:

* Created some resources
* Failed on others
* Updated state partially

The goal is to safely reconcile infrastructure and state.

---

# Recovery Process

## Step 1 — Stop Additional Changes

Do NOT rerun apply immediately.

Pause:

* CI/CD pipelines
* Manual deployments

---

## Step 2 — Review Error Carefully

Check:

* AWS quota issues
* Permission failures
* API throttling
* Dependency failures
* Timeout errors

---

## Step 3 — Inspect Current State

Run:

```bash id="t1"
terraform state list
```

Check actual AWS resources in:
Amazon console.

---

## Step 4 — Run Safe Plan

```bash id="t2"
terraform plan
```

Terraform usually detects partially created resources and continues reconciliation.

---

## Step 5 — Import Missing Resources (If Needed)

Sometimes resources exist in AWS but not in state:

```bash id="t3"
terraform import aws_instance.web i-123456
```

---

## Step 6 — Re-Apply Carefully

After validation:

```bash id="t4"
terraform apply
```

---

# Enterprise Prevention

Use:

* Smaller deployments
* Modular states
* Retry mechanisms
* CI/CD approvals
* Blue/green deployment

---

## Interview Answer

> If Terraform partially succeeds, I first stop further deployments and analyze the root cause. Then I validate both the Terraform state and actual cloud resources to identify inconsistencies. I run terraform plan to reconcile drift, import any missing resources if necessary, and only then rerun terraform apply after ensuring no destructive changes are proposed.

---

# 42. Terraform wants to recreate a production database unexpectedly. How do you investigate?

This is a high-risk production issue.

Never approve blindly.

---

# Investigation Process

## Step 1 — Review Terraform Plan Carefully

Look for:

```text id="t5"
-/+ destroy and recreate
```

on:

* RDS
* DynamoDB
* Production storage

---

# Common Causes

| Cause                       | Example                  |
| --------------------------- | ------------------------ |
| Immutable parameter changed | Engine version           |
| Resource renamed            | Identifier changed       |
| State drift                 | Manual AWS changes       |
| Module change               | Different resource block |
| Wrong backend/workspace     | Using incorrect state    |

---

## Step 2 — Compare Current Infrastructure

Check actual DB settings in:
Amazon RDS Console.

---

## Step 3 — Review Git Changes

Investigate:

* Variable changes
* Module updates
* Renamed resources
* Provider upgrades

---

## Step 4 — Validate State

```bash id="t6"
terraform state show aws_db_instance.prod
```

---

## Step 5 — Prevent Destruction

Temporarily use:

```hcl id="t7"
lifecycle {
  prevent_destroy = true
}
```

---

# Important

Never run apply until root cause is fully understood.

---

## Interview Answer

> If Terraform unexpectedly wants to recreate a production database, I immediately stop the deployment and investigate the Terraform plan, state file, recent code changes, provider upgrades, and actual cloud resource configuration. I verify whether immutable attributes changed or drift occurred. I often use prevent_destroy on critical databases to avoid accidental deletion while troubleshooting.

---

# 43. Circular dependency error appears in modules. How do you fix it?

---

# Cause

Terraform dependency cycle occurs when:

```text id="t8"
Module A depends on Module B
Module B depends on Module A
```

Terraform cannot determine execution order.

---

# Example

VPC module outputs SG ID.

EC2 module depends on VPC.

But VPC module also references EC2 output.

---

# Fix Strategies

## 1. Break Dependency Chain

Move shared resources into separate module.

---

## 2. Use Data Sources

Instead of direct references:

```hcl id="t9"
data "aws_vpc" "existing" {}
```

---

## 3. Separate Deployment Stages

Example:

```text id="t10"
Network → Security → Compute
```

---

## 4. Avoid Over-Coupled Modules

Modules should expose outputs only.

Avoid cross-module resource creation.

---

# Enterprise Best Practice

Design modules:

* Loosely coupled
* Single responsibility
* Independent lifecycle

---

## Interview Answer

> Circular dependency errors occur when Terraform modules reference each other in a way that creates an execution loop. I fix this by decoupling modules, introducing separate shared modules, using data sources where appropriate, and restructuring deployments into independent layers such as networking, security, and compute.

---

# 44. terraform init fails after provider upgrade. What troubleshooting steps do you take?

---

# Investigation Steps

## Step 1 — Read Exact Error

Common issues:

* Version incompatibility
* Provider source changes
* Lock file mismatch
* Corrupted cache

---

## Step 2 — Check Provider Version

Example:

```hcl id="t11"
required_providers {
  aws = {
    source  = "hashicorp/aws"
    version = "~> 5.0"
  }
}
```

---

## Step 3 — Remove Cache

```bash id="t12"
rm -rf .terraform
```

---

## Step 4 — Reinitialize

```bash id="t13"
terraform init -upgrade
```

---

## Step 5 — Validate Terraform Version

```bash id="t14"
terraform version
```

Ensure compatibility.

---

## Step 6 — Review Provider Changelog

Breaking changes may require:

* Resource updates
* Syntax changes
* State migration

---

# Enterprise Best Practice

Pin:

* Terraform version
* Provider versions

Avoid uncontrolled upgrades.

---

## Interview Answer

> When terraform init fails after a provider upgrade, I first analyze the exact error, verify Terraform and provider version compatibility, clear the local .terraform cache, and reinitialize using terraform init -upgrade. I also review provider changelogs for breaking changes and ensure versions are pinned to avoid unexpected upgrade issues.

---

# 45. Provider plugin download fails in restricted corporate network. How do you solve it?

Very common in enterprise environments.

---

# Solutions

## 1. Configure Proxy

```bash id="t15"
export HTTPS_PROXY=http://proxy.company.com:8080
```

---

## 2. Use Internal Provider Mirror

Terraform supports filesystem/provider mirrors.

Example:

```hcl id="t16"
provider_installation {
  filesystem_mirror {
    path = "/terraform/providers"
  }
}
```

---

## 3. Artifactory/Nexus Mirror

Use:

* JFrog Artifactory
* Sonatype Nexus

---

## 4. Pre-download Providers

```bash id="t17"
terraform providers mirror
```

---

## 5. Firewall Allowlisting

Allow:

* registry.terraform.io

---

# Enterprise Recommendation

Maintain centralized provider mirror for:

* Security
* Compliance
* Faster builds

---

## Interview Answer

> In restricted enterprise networks, I solve Terraform provider download issues using proxy configuration, internal provider mirrors, or artifact repositories such as Artifactory or Nexus. I also pre-download providers and maintain centralized approved provider repositories to improve security and reliability.

---

# 46. How would you structure Terraform repositories for 100+ microservices?

Large-scale environments require strong standardization.

---

# Recommended Structure

## Option 1 — Separate Infra & App Repos

```text id="t18"
infra-live/
infra-modules/
service-repos/
```

---

# Example Structure

```text id="t19"
terraform-live/
 ├── dev/
 ├── stage/
 ├── prod/

terraform-modules/
 ├── eks/
 ├── vpc/
 ├── rds/
```

---

# Per Microservice

Each service:

* Own module usage
* Own state
* Own pipeline

---

# Key Principles

## Isolate States

Avoid giant shared state.

---

## Reusable Modules

Platform team maintains modules.

---

## Environment Segregation

Separate:

* Accounts
* States
* Pipelines

---

## CI/CD Standardization

Common reusable pipeline templates.

---

## Interview Answer

> For large microservice environments, I separate reusable Terraform modules from live environment configurations. Each microservice has isolated state and deployment pipelines, while platform teams maintain shared modules such as networking and Kubernetes infrastructure. This enables scalability, ownership separation, and safer deployments across environments.

---

# 47. How do you manage Terraform across platform teams and application teams?

---

# Recommended Operating Model

## Platform Team Responsibilities

Own:

* Shared modules
* Networking
* IAM
* Security baselines
* CI/CD standards

---

## Application Team Responsibilities

Own:

* Service-specific infrastructure
* Variables
* Deployments

---

# Governance Model

Platform team provides:

* Approved modules
* Policies
* Guardrails

App teams consume standardized infrastructure.

---

# Example

```text id="t20"
Platform Team:
  VPC Module
  EKS Module

Application Team:
  Service Deployment
  Autoscaling
  DNS
```

---

# Enterprise Benefits

* Central governance
* Team autonomy
* Standardization
* Faster onboarding

---

## Interview Answer

> I separate responsibilities between platform and application teams. Platform teams manage shared infrastructure modules, governance, security, and CI/CD standards, while application teams consume approved modules for service deployment. This balances centralized control with team-level agility.

---

# 48. How do you implement policy-as-code for Terraform deployments?

---

# Common Tools

* HashiCorp Sentinel
* Open Policy Agent (OPA)
* Checkov
* TFSec

---

# Example Policies

Enforce:

* Encryption enabled
* Approved instance types
* Mandatory tags
* Restricted public access

---

# CI/CD Enforcement

Pipeline flow:

```text id="t21"
Terraform Plan
   ↓
Policy Validation
   ↓
Approval
   ↓
Apply
```

---

# Example OPA Policy

```rego id="t22"
deny[msg] {
  input.resource.aws_s3_bucket.public == true
  msg = "Public S3 buckets are not allowed"
}
```

---

# Enterprise Benefits

* Compliance automation
* Prevent insecure deployments
* Standardized governance

---

## Interview Answer

> I implement policy-as-code using tools such as Sentinel, OPA, Checkov, or TFSec integrated into CI/CD pipelines. Policies enforce security, compliance, encryption, tagging, and governance rules before Terraform apply is allowed, ensuring consistent enterprise controls across environments.

---

# 49. How do you standardize tagging, naming conventions, and governance using Terraform?

---

# Centralized Standards

Use:

* Locals
* Shared modules
* Policy-as-code

---

# Example Tags

```hcl id="t23"
locals {
  common_tags = {
    Environment = var.environment
    Owner       = "platform-team"
    CostCenter  = "IT"
  }
}
```

---

# Naming Convention

```hcl id="t24"
name = "${var.env}-${var.app}-${var.region}"
```

---

# Governance Enforcement

Use:

* Sentinel
* OPA
* AWS SCPs

---

# Enterprise Benefits

* Cost tracking
* Compliance
* Easier operations
* Audit readiness

---

## Interview Answer

> I standardize governance in Terraform using centralized tagging and naming conventions implemented through reusable modules and locals. Mandatory tags, naming standards, and compliance policies are enforced through policy-as-code tools and CI/CD validation to ensure consistency across all environments.

---

# 50. How would you scale Terraform usage globally across multiple teams and cloud accounts?

---

# Enterprise Scaling Model

## Multi-Account Strategy

Separate accounts for:

* Dev
* Stage
* Prod
* Business units

---

# Shared Module Registry

Platform teams maintain:

* Approved reusable modules
* Versioned releases

---

# Central Governance

Implement:

* RBAC
* Policy-as-code
* Security scanning
* Approval workflows

---

# Standardized CI/CD

Provide reusable pipelines for all teams.

---

# State Isolation

Each environment/account:

* Independent backend
* Independent locking

---

# Cross-Account Access

Use:

* AWS AssumeRole
* Federated identity

---

# Observability

Centralize:

* Drift detection
* Audit logs
* Cost monitoring
* Compliance reports

---

# Recommended Enterprise Architecture

```text id="t25"
Platform Team
   ↓
Shared Modules + Policies
   ↓
CI/CD Templates
   ↓
Application Teams
   ↓
Multi-Account Deployments
```

---

## Interview Answer

> To scale Terraform globally, I implement a multi-account architecture with isolated state management, centralized reusable modules, policy-as-code governance, and standardized CI/CD pipelines. Platform teams provide approved infrastructure building blocks, while application teams deploy independently using controlled access and automated compliance validation.


# 1. How does Terraform work in real-time projects?

In real-time enterprise projects, Terraform is used as an Infrastructure as Code (IaC) tool to automate provisioning and management of cloud infrastructure.

Instead of manually creating resources from cloud consoles, engineers define infrastructure in `.tf` files.

Terraform workflow typically looks like this:

```text id="rt1"
Developer Writes Code
        ↓
Git Commit / Pull Request
        ↓
CI/CD Pipeline Trigger
        ↓
terraform init
        ↓
terraform validate
        ↓
terraform plan
        ↓
Approval Process
        ↓
terraform apply
        ↓
Infrastructure Created/Updated
```

---

# Real-Time Example

Suppose an application team needs:

* VPC
* EC2 instances
* Load balancer
* RDS database
* IAM roles

Using Terraform:

* Infrastructure becomes reusable
* Deployments become automated
* Changes are version controlled
* Rollbacks become easier
* Multi-environment consistency improves

---

# Typical Enterprise Setup

| Component      | Example                 |
| -------------- | ----------------------- |
| Source Control | GitHub                  |
| CI/CD          | Jenkins / Microsoft     |
| Cloud          | Amazon / Microsoft      |
| State Backend  | S3 / Azure Storage      |
| Locking        | DynamoDB                |
| Secrets        | Vault / Secrets Manager |

---

# Key Benefits

* Automation
* Consistency
* Reusability
* Version control
* Faster deployments
* Reduced manual errors

---

# Interview Answer

> In real-time projects, Terraform is used to automate cloud infrastructure provisioning through Infrastructure as Code. Infrastructure definitions are stored in Git repositories, executed through CI/CD pipelines, and deployed consistently across environments like dev, stage, and production. Teams use reusable modules, remote state management, approval workflows, and policy enforcement to manage infrastructure safely at scale.

---

# 2. How do you manage Terraform state files in production?

Terraform state is critical because it maps Terraform configuration to actual cloud resources.

In production, state management must be:

* Secure
* Shared
* Locked
* Recoverable

---

# Best Practices

## Use Remote Backend

Never use local state in production.

Common backends:

* AWS S3
* Azure Storage Account
* Terraform Cloud

---

# Enable State Locking

In AWS:

```text id="rt2"
S3 → State Storage
DynamoDB → State Locking
```

Prevents concurrent modifications.

---

# Enable Versioning

Critical for recovery from:

* Corruption
* Accidental deletion
* Bad deployments

---

# Encrypt State

Because state may contain:

* Passwords
* Tokens
* Connection strings

Use:

* SSE-KMS
* Storage encryption

---

# Restrict Access

Only:

* CI/CD pipelines
* Infra admins

should access state.

---

# Backup Strategy

Enable:

* S3 versioning
* Blob snapshots
* Automated backups

---

# Interview Answer

> In production, I store Terraform state remotely using secure backends such as S3 or Azure Storage. I enable state locking using DynamoDB, enforce encryption and versioning, restrict access using IAM/RBAC, and avoid storing local state files. This ensures safe collaboration, recovery capability, and protection against corruption or concurrent changes.

---

# 3. Difference between local state and remote state?

| Feature          | Local State     | Remote State   |
| ---------------- | --------------- | -------------- |
| Storage Location | Local machine   | Shared backend |
| Collaboration    | Poor            | Excellent      |
| Locking          | No              | Yes            |
| Security         | Weak            | Strong         |
| Backup           | Manual          | Automated      |
| Production Use   | Not recommended | Recommended    |

---

# Local State

Default behavior:

```text id="rt3"
terraform.tfstate
```

stored locally.

Good for:

* Learning
* Small testing

Not good for:

* Teams
* Production

---

# Remote State

Stored in:

* S3
* Azure Blob
* Terraform Cloud

Supports:

* Locking
* Shared access
* Encryption
* Versioning

---

# Interview Answer

> Local Terraform state is stored on an engineer’s machine and is mainly suitable for testing or learning environments. Remote state is stored in centralized backends such as S3 or Azure Storage and supports collaboration, locking, encryption, versioning, and secure access control. Remote state is the recommended approach for production environments.

---

# 4. How do you use S3 + DynamoDB in Terraform?

This is the most common AWS production backend design.

---

# Architecture

```text id="rt4"
S3 Bucket       → Stores Terraform State
DynamoDB Table  → Provides State Locking
```

---

# Backend Configuration

```hcl id="rt5"
terraform {
  backend "s3" {
    bucket         = "prod-terraform-state"
    key            = "network/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

---

# Benefits

## S3

Provides:

* Centralized storage
* Versioning
* Encryption
* Backup

---

## DynamoDB

Provides:

* State locking
* Prevents concurrent apply

---

# Enterprise Recommendations

Enable:

* S3 versioning
* SSE-KMS encryption
* Least privilege IAM
* MFA delete

---

# Interview Answer

> In AWS production environments, Terraform state is typically stored in an S3 bucket while DynamoDB is used for state locking. S3 provides centralized encrypted state storage with versioning, and DynamoDB prevents concurrent Terraform operations from corrupting the state file.

---

# 5. What are Terraform modules and why are they important?

Terraform modules are reusable infrastructure components.

They help avoid duplication and standardize deployments.

---

# Example

Instead of writing EC2 configuration repeatedly:

```text id="rt6"
modules/
 ├── vpc/
 ├── ec2/
 ├── rds/
```

---

# Benefits

| Benefit         | Description                  |
| --------------- | ---------------------------- |
| Reusability     | Write once, reuse many times |
| Consistency     | Standardized deployments     |
| Maintainability | Easier updates               |
| Scalability     | Supports enterprise growth   |
| Governance      | Enforce security standards   |

---

# Module Example

```hcl id="rt7"
module "web" {
  source        = "./modules/ec2"
  instance_type = "t3.medium"
}
```

---

# Interview Answer

> Terraform modules are reusable infrastructure components that help standardize deployments and reduce duplicated code. In enterprise environments, modules are used for common resources such as VPCs, EC2 instances, IAM roles, and databases. They improve maintainability, scalability, governance, and deployment consistency.

---

# 6. How do you manage multiple environments in Terraform?

Common environments:

* Dev
* QA
* Stage
* Production

---

# Recommended Structure

```text id="rt8"
environments/
 ├── dev/
 ├── stage/
 ├── prod/
```

---

# Key Practices

## Separate State Files

Each environment gets:

* Independent backend
* Independent locking

---

## Use Same Modules

```text id="rt9"
Shared Modules
    ↓
Different Variables
```

---

# Environment-Specific Variables

Example:

```hcl id="rt10"
instance_type = "t3.micro"   # Dev
instance_type = "m5.large"   # Prod
```

---

# CI/CD Governance

| Environment | Deployment      |
| ----------- | --------------- |
| Dev         | Automatic       |
| Prod        | Manual approval |

---

# Interview Answer

> I manage multiple Terraform environments using the same reusable modules with environment-specific variable files and separate remote state backends. Each environment has isolated state, CI/CD pipelines, and approval workflows to ensure safe and consistent deployments across dev, stage, and production.

---

# 7. Difference between terraform plan and terraform apply?

| Command         | Purpose         |
| --------------- | --------------- |
| terraform plan  | Preview changes |
| terraform apply | Execute changes |

---

# terraform plan

Shows:

* What Terraform will create
* Modify
* Destroy

Safe operation.

---

# Example

```bash id="rt11"
terraform plan
```

---

# terraform apply

Actually provisions infrastructure.

```bash id="rt12"
terraform apply
```

---

# Enterprise Workflow

```text id="rt13"
Plan → Review → Approval → Apply
```

---

# Interview Answer

> terraform plan generates an execution preview showing what infrastructure changes Terraform intends to make, while terraform apply actually executes those changes in the cloud environment. In production environments, plans are reviewed and approved before apply is executed.

---

# 8. How do you handle secrets securely in Terraform?

Terraform state can expose secrets, so security is critical.

---

# Best Practices

## Use Secret Managers

* Amazon Secrets Manager
* HashiCorp Vault
* Microsoft Key Vault

---

# Avoid Hardcoding

Never store secrets in:

* `.tf` files
* Git repos
* Plain tfvars

---

# Use Environment Variables

```bash id="rt14"
export TF_VAR_db_password=*****
```

---

# Sensitive Variables

```hcl id="rt15"
sensitive = true
```

---

# Secure State Backend

Encrypt state using:

* KMS
* Storage encryption

---

# Interview Answer

> I handle secrets securely in Terraform by integrating with secret management solutions such as AWS Secrets Manager, Azure Key Vault, or Vault. I avoid hardcoding secrets, use environment variables or dynamic retrieval, mark variables as sensitive, and ensure Terraform state is encrypted and access-controlled.

---

# 9. How do you debug Terraform deployment failures?

---

# Troubleshooting Process

## Step 1 — Read Error Carefully

Common causes:

* IAM permission issues
* Quota limits
* Invalid variables
* API failures
* Dependency issues

---

## Step 2 — Enable Debug Logs

```bash id="rt16"
export TF_LOG=DEBUG
```

---

## Step 3 — Validate Code

```bash id="rt17"
terraform validate
```

---

## Step 4 — Check Plan

```bash id="rt18"
terraform plan
```

---

## Step 5 — Verify Cloud Resources

Check actual infrastructure in:

* AWS Console
* Azure Portal

---

## Step 6 — Review State

```bash id="rt19"
terraform state list
```

---

# Common Enterprise Issues

| Issue          | Example             |
| -------------- | ------------------- |
| Quota exceeded | VPC limits          |
| IAM denied     | Missing permissions |
| Drift          | Manual changes      |
| API throttling | Large deployments   |

---

# Interview Answer

> I debug Terraform failures by analyzing the exact error message, validating Terraform configuration, reviewing plans, checking Terraform state, and verifying cloud resources directly in the provider console. I also enable Terraform debug logs and investigate IAM permissions, quotas, provider issues, or dependency conflicts.

---

# 10. How do you integrate Terraform with AWS?

Terraform integrates with AWS using the AWS provider.

---

# Steps

## Configure Provider

```hcl id="rt20"
provider "aws" {
  region = "ap-south-1"
}
```

---

# Authentication Methods

Recommended:

* IAM roles
* AssumeRole
* OIDC federation

Avoid static keys.

---

# Typical AWS Resources

Terraform provisions:

* VPC
* EC2
* S3
* IAM
* RDS
* EKS
* Lambda

---

# CI/CD Integration

Used with:

* GitHub Actions
* Jenkins
* Microsoft

---

# Interview Answer

> Terraform integrates with AWS using the AWS provider and IAM-based authentication. Infrastructure such as VPCs, EC2, RDS, IAM, EKS, and S3 can be provisioned declaratively through Terraform code. In enterprise environments, Terraform is integrated with CI/CD pipelines and uses remote state management with S3 and DynamoDB.

---

# 11. How do you integrate Terraform with Azure or GCP?

Terraform supports multi-cloud through providers.

---

# Azure Integration

Use:

* Microsoft AzureRM provider

Example:

```hcl id="rt21"
provider "azurerm" {
  features {}
}
```

Authentication:

* Service Principal
* Managed Identity

Resources:

* VNet
* VM
* AKS
* Storage
* SQL Database

---

# GCP Integration

Use:

* Google provider

Example:

```hcl id="rt22"
provider "google" {
  project = "prod-project"
  region  = "asia-south1"
}
```

Authentication:

* Service Accounts

Resources:

* GKE
* Compute Engine
* VPC
* Cloud SQL

---

# Multi-Cloud Benefits

* Standardized IaC
* Reusable workflows
* Unified governance

---

# Interview Answer

> Terraform integrates with Azure using the AzureRM provider and with GCP using the Google provider. Authentication is typically handled through service principals, managed identities, or service accounts. Terraform can provision cloud-native services such as AKS, GKE, virtual networks, compute instances, databases, and storage using the same Infrastructure as Code principles across clouds.

# 1. What is Terraform drift and how do you fix it?

Terraform drift occurs when real infrastructure changes outside Terraform, causing Terraform state and actual cloud resources to become inconsistent.

---

# Example of Drift

Suppose Terraform created:

* EC2 instance
* Security group
* S3 bucket

Then someone manually changes:

* Security group rule
* Instance size
* Bucket policy

from the:
Amazon console.

Terraform state still thinks old configuration exists.

This mismatch is called drift.

---

# How to Detect Drift

## Run Terraform Plan

```bash id="d1"
terraform plan
```

Terraform compares:

* State file
* Actual infrastructure
* Current code

---

# Example Output

```text id="d2"
~ security_group_rule will be updated
```

---

# How to Fix Drift

## Option 1 — Revert Manual Changes

Run:

```bash id="d3"
terraform apply
```

Terraform restores desired state.

---

## Option 2 — Accept Manual Changes

Update Terraform code to match actual infrastructure.

Then run:

```bash id="d4"
terraform apply
```

---

## Option 3 — Import Missing Resources

If resource exists but not in state:

```bash id="d5"
terraform import
```

---

# Enterprise Prevention

Use:

* RBAC restrictions
* Policy-as-code
* Drift detection pipelines
* Infrastructure governance

---

# Interview Answer

> Terraform drift occurs when infrastructure is modified outside Terraform, causing mismatch between Terraform state and actual resources. Drift is detected using terraform plan or refresh-only operations. To fix drift, I either reapply Terraform to restore the desired configuration or update Terraform code to match approved manual changes. In enterprise environments, drift prevention is enforced through governance and restricted manual access.

---

# 2. How do you destroy specific resources in Terraform?

Terraform allows targeted resource destruction.

---

# Command

```bash id="d6"
terraform destroy -target=aws_instance.web
```

---

# Example

Destroy only one EC2 instance:

```bash id="d7"
terraform destroy -target=aws_instance.app
```

---

# Important Notes

Terraform destroys:

* Selected resource
* Dependent resources if required

---

# Production Caution

Avoid frequent use of:

```bash id="d8"
-target
```

because it may:

* Skip dependencies
* Create inconsistent state

---

# Preferred Enterprise Approach

Usually:

* Remove resource from code
* Run normal plan/apply

This ensures dependency integrity.

---

# Interview Answer

> Specific Terraform resources can be destroyed using terraform destroy -target. However, in production environments, targeted operations should be used cautiously because they may bypass dependency management. The preferred approach is usually to modify the Terraform code and allow Terraform to reconcile infrastructure safely.

---

# 3. What are lifecycle rules in Terraform?

Lifecycle rules control how Terraform handles resource changes.

They are used to:

* Prevent accidental deletion
* Ignore changes
* Force replacement behavior

---

# Common Lifecycle Rules

| Rule                  | Purpose                    |
| --------------------- | -------------------------- |
| prevent_destroy       | Prevent deletion           |
| ignore_changes        | Ignore selected attributes |
| create_before_destroy | Reduce downtime            |

---

# Example — Prevent Deletion

```hcl id="d9"
lifecycle {
  prevent_destroy = true
}
```

Used for:

* Production databases
* Critical storage

---

# Example — Ignore Changes

```hcl id="d10"
lifecycle {
  ignore_changes = [tags]
}
```

Useful when tags are managed externally.

---

# Example — Create Before Destroy

```hcl id="d11"
lifecycle {
  create_before_destroy = true
}
```

Helps minimize downtime.

---

# Interview Answer

> Lifecycle rules in Terraform control how resources are created, updated, or destroyed. Common lifecycle settings include prevent_destroy to protect critical resources, ignore_changes to avoid unnecessary updates, and create_before_destroy to minimize downtime during replacement operations.

---

# 4. How do you manage dependencies between resources?

Terraform automatically handles dependencies using resource references.

---

# Example

```hcl id="d12"
resource "aws_instance" "web" {
  subnet_id = aws_subnet.public.id
}
```

Terraform understands:

* Subnet must exist first

---

# Explicit Dependencies

If needed:

```hcl id="d13"
depends_on = [aws_iam_role.app]
```

---

# Best Practice

Prefer:

* Implicit dependencies

Avoid excessive:

* depends_on

---

# Enterprise Example

```text id="d14"
VPC
 ↓
Subnets
 ↓
Security Groups
 ↓
EC2/EKS/RDS
```

---

# Interview Answer

> Terraform manages dependencies automatically through resource references. When one resource references another, Terraform determines the correct creation order. Explicit depends_on statements are used only when implicit dependencies are insufficient, ensuring proper orchestration and infrastructure consistency.

---

# 5. How do you use variables and output values in Terraform?

Variables make Terraform reusable and configurable.

Outputs expose important resource information.

---

# Variable Example

```hcl id="d15"
variable "instance_type" {
  default = "t3.micro"
}
```

Usage:

```hcl id="d16"
instance_type = var.instance_type
```

---

# Output Example

```hcl id="d17"
output "instance_ip" {
  value = aws_instance.web.public_ip
}
```

---

# Benefits

## Variables

* Environment customization
* Reusability
* Reduced duplication

---

## Outputs

Expose:

* IP addresses
* Resource IDs
* Endpoints
* Load balancer DNS

---

# Enterprise Usage

Different environments use:

* Separate tfvars files
* CI/CD injected variables

---

# Interview Answer

> Variables make Terraform configurations reusable and environment-specific, while outputs expose useful infrastructure information such as IP addresses, IDs, or DNS names. Variables are commonly supplied through tfvars files or CI/CD pipelines, and outputs are often consumed by other modules or deployment processes.

---

# 6. How do you automate Terraform using CI/CD pipelines?

Terraform automation is a core enterprise practice.

---

# Typical Workflow

```text id="d18"
Git Commit
   ↓
Pipeline Trigger
   ↓
terraform init
   ↓
terraform validate
   ↓
terraform plan
   ↓
Approval
   ↓
terraform apply
```

---

# Common CI/CD Platforms

* Jenkins
* GitHub Actions
* Microsoft
* GitLab CI

---

# Enterprise Enhancements

Include:

* Security scanning
* Drift detection
* Policy-as-code
* Secret management
* Environment approvals

---

# Example GitHub Actions Flow

```yaml id="d19"
- terraform init
- terraform validate
- terraform plan
- terraform apply
```

---

# Interview Answer

> Terraform is automated through CI/CD pipelines that execute initialization, validation, planning, approval, and apply stages. Pipelines are integrated with Git repositories, remote state backends, approval workflows, and security scanning tools to provide safe and repeatable infrastructure deployments.

---

# 7. What are common Terraform issues in production?

---

# Common Problems

| Issue                   | Example                   |
| ----------------------- | ------------------------- |
| State corruption        | Interrupted apply         |
| Drift                   | Manual cloud changes      |
| Concurrent updates      | Missing locking           |
| Quota limits            | VPC/Elastic IP exhaustion |
| IAM permission failures | Access denied             |
| API throttling          | Large deployments         |
| Provider bugs           | Upgrade incompatibility   |
| Secret exposure         | Sensitive state data      |
| Monolithic state        | Slow deployments          |

---

# Example

```text id="d20"
Error acquiring state lock
```

Usually due to:

* DynamoDB locking issue

---

# Enterprise Mitigation

Use:

* Remote state
* Versioning
* CI/CD governance
* Modular design
* Monitoring
* Drift detection

---

# Interview Answer

> Common Terraform production issues include state corruption, infrastructure drift, provider incompatibilities, API throttling, permission errors, and concurrent state conflicts. These are mitigated using remote state backends, locking, modular architecture, CI/CD governance, monitoring, and controlled provider versioning.

---

# 8. How do you optimize Terraform code for large projects?

Large projects require:

* Scalability
* Faster execution
* Easier maintenance

---

# Optimization Strategies

## Use Modules

Reduce duplication.

---

## Split Monolithic State

Separate:

* Networking
* Security
* Applications
* Databases

---

## Use for_each and count

Example:

```hcl id="d21"
for_each = var.instances
```

---

## Avoid Hardcoding

Use:

* Variables
* Locals

---

## Pin Provider Versions

Avoid unexpected behavior.

---

## Standardize Structure

```text id="d22"
modules/
environments/
shared/
```

---

## Reduce Unnecessary Dependencies

Too many dependencies slow plans.

---

# Enterprise Best Practice

Use:

* CI/CD templates
* Shared modules
* Policy enforcement
* Automated testing

---

# Interview Answer

> I optimize Terraform for large projects by using reusable modules, splitting large state files into smaller logical domains, minimizing unnecessary dependencies, standardizing repository structures, and parameterizing infrastructure using variables and locals. I also implement CI/CD automation, provider version pinning, and policy enforcement for scalability and maintainability.

---

# 9. What are Terraform best practices used in companies?

---

# Infrastructure Best Practices

## Use Remote State

Examples:

* S3 + DynamoDB
* Azure Storage

---

## Enable Locking

Prevent concurrent modifications.

---

## Use Modular Design

Reusable:

* VPC
* IAM
* EKS
* RDS modules

---

## Version Control Everything

Store in:

* GitHub
* GitLab
* Bitbucket

---

## Use CI/CD Pipelines

Automate:

* Validation
* Planning
* Deployment

---

## Separate Environments

Independent:

* State
* Pipelines
* Variables

---

## Secure Secrets

Use:

* HashiCorp Vault
* Amazon Secrets Manager

---

## Pin Versions

Pin:

* Terraform version
* Provider versions
* Module versions

---

## Implement Policy-as-Code

Use:

* Sentinel
* OPA
* Checkov

---

## Enable Monitoring & Drift Detection

Continuously monitor infrastructure consistency.

---

# Enterprise Terraform Workflow

```text id="d23"
Developer
   ↓
Pull Request
   ↓
Terraform Plan
   ↓
Security Scan
   ↓
Approval
   ↓
Terraform Apply
```

---

# Interview Answer

> Enterprise Terraform best practices include using remote encrypted state backends with locking, modular infrastructure design, CI/CD automation, provider and module version pinning, policy-as-code enforcement, secure secret management, environment isolation, and governance through approvals and automated security scanning.



# Most Asked Interview Questions (High Priority)

If preparing for senior interviews, focus first on:

* State locking
* Remote backend
* Drift detection
* Modules
* CI/CD integration
* Multi-account AWS
* Secrets handling
* Failed apply recovery
* Provider upgrades
* Governance

---

# Senior-Level Answer Structure

When interviewer asks scenario questions, answer like this:

### 1. Problem Identification

### 2. Risk / Impact

### 3. Root Cause Analysis

### 4. Safe Fix Implementation

### 5. Prevention / Best Practice

---

# Example Professional Answer

**Q: Terraform state corrupted in production?**

Answer:

* Stop further applies
* Restore last good backend version
* Compare resources with actual AWS infra
* Run terraform plan carefully
* Re-import missing resources if needed
* Enable versioning + backups

---

# Interview Tip for 7+ Years DevOps

Speak in terms of:

* Automation
* Governance
* Scale
* Recovery
* Security
* Team workflows
* Cost optimization

That separates senior engineers from beginners.

---

If you'd like, I can also give you **Top 50 Terraform Answers for These Scenario Questions (Senior-Level Sample Responses)** next.
