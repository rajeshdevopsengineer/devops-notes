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

