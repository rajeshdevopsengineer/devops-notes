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

## 2.

Multiple engineers are modifying the same infrastructure. How do you prevent concurrent state conflicts?

## 3.

A developer accidentally deleted the remote S3 state bucket. What steps would you take?

## 4.

Terraform state contains resources that no longer exist in AWS. How do you reconcile it?

## 5.

You need to split one large monolithic state file into smaller states. How would you do it?

## 6.

How would you migrate Terraform local state to remote backend with zero downtime?

## 7.

How do you securely store and protect Terraform state containing secrets?

## 8.

You lost access to DynamoDB lock table used for Terraform locking. What happens and how do you fix it?

## 9.

How do you rotate backend credentials without breaking Terraform pipelines?

## 10.

How do you recover if someone manually edited the state file incorrectly?

---

# Drift / Change Control Scenarios

## 11.

Production resources were modified manually from AWS Console. How do you detect and fix drift?

## 12.

Terraform plan shows no changes, but users report infrastructure mismatch. What would you investigate?

## 13.

How do you implement automated drift detection across 20 AWS accounts?

## 14.

How do you stop teams from making manual cloud changes outside Terraform?

## 15.

A security group rule was manually removed in production. How do you restore it safely?

---

# Multi-Environment Scenarios

## 16.

How do you design Terraform for dev, stage, prod using same codebase?

## 17.

How do you ensure prod changes require approval but dev changes are automatic?

## 18.

How do you isolate state files between environments?

## 19.

How do you handle environment-specific variables without duplicating code?

## 20.

How do you promote the same Terraform module version from dev to prod?

---

# Modular Design Scenarios

## 21.

Your Terraform codebase has thousands of lines duplicated. How would you refactor it?

## 22.

How do you design reusable modules for VPC, EC2, RDS, IAM?

## 23.

A shared module change broke multiple teams. How do you prevent this in future?

## 24.

How do you version Terraform modules in enterprise environments?

## 25.

How do you test modules before publishing for production use?

---

# CI/CD Integration Scenarios

## 26.

How would you integrate Terraform into Jenkins pipeline for production deployment?

## 27.

How do you implement Terraform plan approval workflow in GitHub Actions?

## 28.

How do you run Terraform automatically when code changes in a repo?

## 29.

How do you manage secrets for Terraform in CI/CD pipelines?

## 30.

How do you rollback a failed Terraform deployment in pipeline?

---

# Security / Compliance Scenarios

## 31.

How do you prevent secrets from being hardcoded in Terraform code?

## 32.

How do you scan Terraform code for security misconfigurations before apply?

## 33.

How do you enforce encryption on all S3 buckets created by Terraform?

## 34.

How do you restrict who can run terraform apply in production?

## 35.

How do you implement least privilege IAM roles for Terraform execution?

---

# AWS Advanced Scenarios

## 36.

Terraform apply fails because AWS API rate limits are hit during large deployment. How do you solve it?

## 37.

Terraform deployment fails because VPC quota is exceeded. What is your response plan?

## 38.

How would you provision infrastructure across multiple AWS accounts using AssumeRole?

## 39.

How do you deploy same infrastructure in multiple AWS regions using Terraform?

## 40.

How do you manage hundreds of S3 buckets or IAM users efficiently?

---

# Troubleshooting Scenarios

## 41.

Terraform apply partially succeeded and failed halfway. What next?

## 42.

Terraform wants to recreate a production database unexpectedly. How do you investigate?

## 43.

Circular dependency error appears in modules. How do you fix it?

## 44.

terraform init fails after provider upgrade. What troubleshooting steps do you take?

## 45.

Provider plugin download fails in restricted corporate network. How do you solve it?

---

# Enterprise / Architecture Scenarios

## 46.

How would you structure Terraform repositories for 100+ microservices?

## 47.

How do you manage Terraform across platform teams and application teams?

## 48.

How do you implement policy-as-code for Terraform deployments?

## 49.

How do you standardize tagging, naming conventions, and governance using Terraform?

## 50.

How would you scale Terraform usage globally across multiple teams and cloud accounts?

---

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
