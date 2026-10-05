Below are detailed answers for the **Optum DevOps interview — 5 years’ experience**. Terraform examples are excerpts; assume the referenced providers and input variables are configured.

**1. What are Terraform lifecycle policies?**

Terraform’s `lifecycle` block controls how Terraform updates, replaces, and destroys a resource.

| Lifecycle setting | What it does | Typical use |
|---|---|---|
| `create_before_destroy` | Creates the replacement before deleting the existing resource. | Replacing infrastructure while maintaining capacity. |
| `prevent_destroy` | Rejects a Terraform plan that would destroy the resource while this setting remains configured. | Protecting important databases or storage resources. |
| `ignore_changes` | Excludes selected attributes from update comparisons. | An attribute is intentionally managed by another system. |
| `replace_triggered_by` | Replaces a resource when a referenced managed resource or attribute changes. | Replacing dependent infrastructure when its underlying component changes. |

These settings customize Terraform’s normal resource lifecycle. [HashiCorp Developer](https://developer.hashicorp.com/terraform/tutorials/state/resource-lifecycle?utm_source=chatgpt.com)

For example:

```hcl
resource "aws_instance" "application" {
  ami           = var.approved_ami_id
  instance_type = "t3.micro"

  tags = {
    Name  = "production-app"
    Owner = "platform-team"
  }

  lifecycle {
    prevent_destroy = true
    ignore_changes  = [tags["Owner"]]
  }
}
```

Here:

- Terraform blocks operations requiring this instance to be destroyed.
- Terraform sets `Owner` during creation but ignores subsequent changes to that tag.

**Important limitations:**

- `prevent_destroy` does not prevent someone deleting the resource through AWS.
- Removing the resource block also removes its configured destruction protection.
- `create_before_destroy` requires enough capacity and compatible naming constraints. Application availability still depends on health checks, traffic switching, and deployment design.
- `replace_triggered_by` requires managed-resource references. An ordinary variable cannot directly serve as a replacement trigger. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/meta-arguments/lifecycle?utm_source=chatgpt.com)

Terraform also supports conditions inside `lifecycle`:

```hcl
lifecycle {
  precondition {
    condition     = data.aws_ami.approved.architecture == "x86_64"
    error_message = "The application requires an x86_64 AMI."
  }
}
```

A **precondition** validates an assumption before the relevant operation. A **postcondition** validates a resulting resource or retrieved data. A failed postcondition can stop dependent operations, but it does not undo changes already completed. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/expressions/custom-conditions?utm_source=chatgpt.com)

---

**2. Why do we use workspaces in Terraform?**

A Terraform CLI workspace provides a **separate state for the same configuration and backend**.

For example, the same configuration can represent separate development and testing deployments. Each workspace tracks its own resource instances.

```bash
terraform init

terraform workspace new dev
terraform plan -var-file=dev.tfvars

terraform workspace new test
terraform plan -var-file=test.tfvars

terraform workspace list
terraform workspace select dev
terraform workspace show
```

You can reference the selected workspace:

```hcl
locals {
  environment = terraform.workspace
}

resource "aws_instance" "application" {
  ami           = var.approved_ami_id
  instance_type = var.instance_type

  tags = {
    Name        = "application-${local.environment}"
    Environment = local.environment
  }
}
```

**When workspaces help:**

- Creating similar deployments from one configuration.
- Isolating temporary feature or test environments.
- Keeping those deployments’ resource tracking separate.

**Their limitation:** CLI workspaces share the configured backend and are not an access-control boundary. They do not automatically select different AWS accounts, credentials, or variable files.

For production environments requiring separate permissions and ownership, I would generally use separate root configurations, backend locations, and deployment roles, while sharing reusable modules. [HashiCorp Developer](https://developer.hashicorp.com/terraform/cli/workspaces?utm_source=chatgpt.com)

---

**3. How do you transfer payloads between Lambda functions in two different AWS accounts?**

One Lambda can invoke another through the **Lambda Invoke API**, using the target function’s ARN.

Assume:

- Account A: `111111111111`
- Account A execution role: `ProducerLambdaRole`
- Account B: `222222222222`
- Target function: `ProcessPayload`
- Existing target alias: `live`
- Target Region: `us-east-1`

**Step 1: Give the caller permission in Account A.**

Attach an identity policy to `ProducerLambdaRole`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "lambda:InvokeFunction",
      "Resource": "arn:aws:lambda:us-east-1:222222222222:function:ProcessPayload:live"
    }
  ]
}
```

**Step 2: Allow that caller on the target in Account B.**

Add this statement to the target alias’s resource-based policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAccountAProducer",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111111111111:role/ProducerLambdaRole"
      },
      "Action": "lambda:InvokeFunction",
      "Resource": "arn:aws:lambda:us-east-1:222222222222:function:ProcessPayload:live"
    }
  ]
}
```

Both permissions are necessary for this direct cross-account invocation. Scoping access to a specific role and alias limits who can invoke which target. Preserve any existing resource-policy statements when updating the policy. [AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/permissions-function-cross-account.html?utm_source=chatgpt.com)

**Step 3: Invoke the target from Account A.**

Example Python handler:

```python
import json
import boto3

lambda_client = boto3.client("lambda", region_name="us-east-1")

def lambda_handler(event, context):
    response = lambda_client.invoke(
        FunctionName=(
            "arn:aws:lambda:us-east-1:"
            "222222222222:function:ProcessPayload:live"
        ),
        InvocationType="RequestResponse",
        Payload=json.dumps(event).encode("utf-8"),
    )

    result = json.loads(response["Payload"].read())

    if response.get("FunctionError"):
        raise RuntimeError("Target Lambda execution failed")

    return result
```

The SDK uses the caller Lambda’s execution-role credentials. Static access keys are unnecessary.

`RequestResponse` waits for the target execution and returns its response. **A successful invocation API response does not automatically mean the function succeeded**; inspect `FunctionError` and the application response. [AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/invocation-sync.html?utm_source=chatgpt.com)

**Step 4: Choose synchronous or asynchronous processing.**

| Invocation type | Behaviour | Use case |
|---|---|---|
| `RequestResponse` | Caller waits for the result. | The caller immediately needs the processed response. |
| `Event` | Lambda accepts the event for asynchronous execution. | Background processing. |

For asynchronous calls, configure failure handling and make processing idempotent because retries or duplicate delivery can occur. An acceptance response does not confirm successful processing. [AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/invocation-async.html?utm_source=chatgpt.com)

**Step 5: Check network connectivity and payload size.**

The caller contacts the Lambda service API; VPC peering between the functions is not inherently required. A caller in a private VPC needs a route to that API, such as a Lambda interface VPC endpoint or suitable internet egress. [AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/configuration-vpc-endpoints.html?utm_source=chatgpt.com)

Current invocation payload limits are **6 MB for a synchronous request** and **1 MB for an asynchronous request**. For larger payloads, I would store the data in S3, send an object reference, and grant the receiving role permission to read that object. [AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html?utm_source=chatgpt.com)

---

**4. How do you ensure least-privilege access for IAM users?**

Least privilege means granting only the **actions, resources, and conditions needed for a person’s job**.

My approach would be:

1. **Identify the required tasks.**  
   For example, an operations engineer may need to start and stop designated instances without creating or terminating them.

2. **Prefer federated access with temporary credentials.**  
   For workforce access, use IAM Identity Center or an identity provider with roles. Require MFA.

3. **If IAM users are necessary, organize access through groups.**  
   Attach task-specific policies to groups, while checking for additional direct user permissions.

4. **Scope policies carefully.**  
   Specify required actions and resource ARNs. Use supported conditions for restrictions such as environment tags, Region, or access context.

5. **Review actual usage.**  
   Use CloudTrail activity, IAM Access Analyzer, and last-accessed information to refine policies and remove unused access. [AWS Identity and Access Management](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html?utm_source=chatgpt.com)

For example, this policy grants start and stop access to one instance:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:StartInstances",
        "ec2:StopInstances"
      ],
      "Resource": "arn:aws:ec2:us-east-1:111111111111:instance/i-0123456789abcdef0"
    }
  ]
}
```

The resource ARN restricts the scope of this grant. Review the user’s combined permissions because another attached policy might grant broader access. [AWS Identity and Access Management](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_resource.html?utm_source=chatgpt.com)

Additional controls include:

- **Permissions boundaries:** limit permissions that an identity’s policies can grant; boundaries do not grant access themselves. [AWS Identity and Access Management](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_boundaries.html?utm_source=chatgpt.com)
- **Policy validation and testing:** use Access Analyzer and the IAM policy simulator, then verify required and prohibited operations in a controlled environment. Simulator results can differ from live behaviour in advanced configurations. [AWS Identity and Access Management](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_testing-policies.html?utm_source=chatgpt.com)

A strong interview answer explains both **how permissions are initially scoped** and **how they are reviewed over time**.

---

**5. What is the Terraform external command, and when should it be used?**

There is no standard Terraform CLI command named `terraform external`.

The interviewer probably means either:

| Feature | Purpose |
|---|---|
| `external` data source | Execute a program that returns information Terraform can read. |
| `local-exec` provisioner | Execute a local command as part of a resource operation. |

The **external data source** comes from the `hashicorp/external` provider. It is useful for an exceptional lookup that has no suitable native provider data source—for example, querying a company’s custom IP address management system. [GitHub](https://github.com/hashicorp/terraform-provider-external/blob/main/docs/data-sources/external.md?utm_source=chatgpt.com)

Example:

```hcl
terraform {
  required_providers {
    external = {
      source  = "hashicorp/external"
      version = "~> 2.3"
    }
  }
}

data "external" "ipam" {
  program = [
    "python3",
    "${path.module}/scripts/ipam_lookup.py"
  ]

  query = {
    environment = "dev"
    region      = "us-east-1"
  }
}

output "allocated_cidr" {
  value = data.external.ipam.result.cidr
}
```

The script must follow this protocol:

- Read a JSON object from standard input.
- Return a JSON object on standard output.
- Use **strings as the values** in both objects.
- Write diagnostic messages to standard error.
- Exit with a nonzero status on failure.

An example result is:

```json
{
  "cidr": "10.40.0.0/16"
}
```

The program should have no observable side effects because Terraform can run it again during refresh. Its runtime and dependencies must exist on the Terraform runner. [GitHub](https://github.com/hashicorp/terraform-provider-external/blob/main/docs/data-sources/external.md?utm_source=chatgpt.com)

**Production considerations:**

- Prefer a native provider data source when available.
- Reads usually happen during planning but may be deferred to apply when inputs are unknown. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/data-sources?utm_source=chatgpt.com)
- Ordinary data-source results can appear in state. Marking a value `sensitive` hides display output; it does not remove the value from state or encrypt it. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/state/sensitive-data?utm_source=chatgpt.com)

---

**6. How do you ensure a particular AMI is present in an AWS account using Terraform?**

First distinguish two requirements:

1. **Verify that an existing AMI is available.**
2. **Create a copy in the destination account or Region.**

**A. Verify an existing AMI**

Use the `aws_ami` data source with an exact image ID:

```hcl
data "aws_ami" "approved" {
  owners = ["self"]

  filter {
    name   = "image-id"
    values = [var.approved_ami_id]
  }

  filter {
    name   = "state"
    values = ["available"]
  }
}

resource "aws_instance" "application" {
  ami           = data.aws_ami.approved.id
  instance_type = "t3.micro"
}
```

Here:

- `owners = ["self"]` requires ownership by the account used for the lookup.
- `image-id` selects the specific AMI.
- `state = available` checks that the image is ready.
- The data source fails if it cannot resolve a single matching image. [Terraform Registry](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/ami.html?utm_source=chatgpt.com)

If the AMI is **shared from another account**, use that trusted owner’s account ID rather than `self`. The destination also needs the appropriate launch permissions. Sharing makes the AMI usable; it does not transfer ownership. [Amazon Elastic Compute Cloud](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/sharingamis-explicit.html?utm_source=chatgpt.com)

For controlled releases, I would pin an approved AMI ID. A broad lookup using `most_recent = true` can select a newly published image on a later plan.

**B. Copy the AMI into the destination**

Assuming `aws.destination` is configured for the target account and Region:

```hcl
resource "aws_ami_copy" "approved" {
  provider = aws.destination

  name              = var.destination_ami_name
  source_ami_id     = var.source_ami_id
  source_ami_region = var.source_region

  encrypted  = true
  kms_key_id = var.destination_kms_key_arn
}
```

`aws_ami_copy` duplicates the AMI and its associated EBS snapshots, and waits for the copied image to become available. Subsequent resources can use `aws_ami_copy.approved.id`. [Terraform Registry](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/ami_copy.html?utm_source=chatgpt.com)

For cross-account copying, configure source-image sharing, access to backing storage, and applicable KMS permissions. AMIs are regional resources, so the image must also be available in the Region where instances will launch. [Amazon Elastic Compute Cloud](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/CopyingAMIs.html?utm_source=chatgpt.com)

**Important:** an AMI data source performs a lookup. It does not automatically copy an image when the lookup fails.

---

**7. What are Terraform provisioners?**

Provisioners execute commands or transfer files during a resource’s lifecycle.

| Provisioner | Where it operates | Example |
|---|---|---|
| `local-exec` | On the machine executing Terraform. | Running a local integration script. |
| `remote-exec` | On a remote machine through SSH or WinRM. | Running a bootstrap command. |
| `file` | Transfers files to a remote machine. | Uploading a configuration file. |

In a CI/CD pipeline, **local means the Terraform runner**, not the engineer’s laptop or the newly created EC2 instance.

Provisioners are generally a last resort because Terraform cannot fully model the changes their scripts make. By default, they run when the parent resource is created. A creation-time failure normally fails apply and taints the parent resource for replacement. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/provisioners?utm_source=chatgpt.com)

Example: rerun a local integration script when an instance is replaced:

```hcl
resource "terraform_data" "inventory_registration" {
  triggers_replace = [
    aws_instance.application.id
  ]

  provisioner "local-exec" {
    command = "python3 scripts/register_instance.py"

    environment = {
      INSTANCE_ID = aws_instance.application.id
    }
  }
}
```

`terraform_data` gives the operation a managed lifecycle. When the instance ID changes, Terraform replaces this helper resource and runs its provisioner again. The script must exist on the runner and handle repeated execution appropriately. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/resources/terraform-data?utm_source=chatgpt.com)

Useful alternatives include:

- `user_data` or cloud-init for initial machine bootstrap.
- Packer or EC2 Image Builder for preconfigured images.
- Ansible for ongoing configuration management.
- Native provider resources for operations supported by an API.

Other points to remember:

- Provisioners do not normally run on every apply.
- `when = destroy` configures execution **before** resource destruction.
- `on_failure = continue` can hide incomplete configuration, so use it deliberately.
- Remote execution requires connectivity, authentication, and a ready operating system. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/provisioners?utm_source=chatgpt.com)

---

**8. What are S3 bucket lifecycle policies?**

An S3 lifecycle configuration automates management of **objects and object versions** according to rules.

Rules can select objects by prefix, tags, size, or combinations of filters.

Common actions include:

| Action | Purpose |
|---|---|
| Transition current objects | Move data to another storage class as it ages. |
| Expire current objects | Apply the configured expiration behaviour. |
| Transition noncurrent versions | Move older versions to another storage class. |
| Expire noncurrent versions | Permanently remove old versions after the retention period. |
| Abort incomplete multipart uploads | Remove abandoned upload parts. |
| Remove expired delete markers | Clean up eligible delete markers. |

These rules help implement retention requirements and control storage usage. [Amazon Simple Storage Service](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html?utm_source=chatgpt.com)

For example, suppose a log-retention policy specifies:

- Move eligible logs to Standard-IA after 30 days.
- Move them to Glacier Flexible Retrieval after 90 days.
- Expire current objects after 365 days.
- Abort unfinished multipart uploads after 7 days.

Terraform configuration:

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "logs" {
  bucket = var.log_bucket_id

  rule {
    id     = "log-retention"
    status = "Enabled"

    filter {
      prefix = "logs/"
    }

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 90
      storage_class = "GLACIER"
    }

    expiration {
      days = 365
    }

    abort_incomplete_multipart_upload {
      days_after_initiation = 7
    }
  }
}
```

Manage a bucket’s rules through one lifecycle configuration resource; multiple competing configuration resources for the same bucket cause persistent differences. [Terraform Registry](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket_lifecycle_configuration?utm_source=chatgpt.com)

**Important production details:**

- In a versioning-enabled bucket, current-version expiration generally creates a delete marker. Older versions need a separate `noncurrent_version_expiration` rule for permanent removal. [Amazon Simple Storage Service](https://docs.aws.amazon.com/AmazonS3/latest/userguide/intro-lifecycle-rules.html?utm_source=chatgpt.com)
- Transitions have storage-class constraints and possible minimum-duration charges.
- New or modified lifecycle configurations normally do not transition objects smaller than **128 KB** unless their configuration overrides that behaviour.
- Glacier Flexible Retrieval objects require restoration before ordinary access. [Amazon Simple Storage Service](https://docs.aws.amazon.com/AmazonS3/latest/userguide/lifecycle-transition-general-considerations.html?utm_source=chatgpt.com)

S3 lifecycle operates on stored objects. Terraform’s `lifecycle` block controls Terraform’s operations on infrastructure resources.

---

**9. What are meta-arguments in Terraform?**

Meta-arguments are **built into Terraform’s language** and control resource instances, dependencies, provider selection, or lifecycle behaviour.

They differ from provider-specific arguments such as `ami`, `instance_type`, and `bucket`.

| Meta-argument | Purpose | Example |
|---|---|---|
| `count` | Creates instances addressed by numeric indexes. | `count = 3` |
| `for_each` | Creates instances addressed by map or set keys. | `for_each = var.servers` |
| `depends_on` | Declares an explicit dependency. | `depends_on = [aws_iam_role_policy.application]` |
| `provider` | Selects a provider configuration for a resource. | `provider = aws.secondary` |
| `lifecycle` | Customizes resource lifecycle behaviour. | `prevent_destroy = true` inside the block. |
| `providers` | Passes provider configurations into a child module. | `providers = { aws = aws.secondary }` |

Support varies by block type. In particular, singular `provider` selects a configuration, while plural `providers` maps configurations into modules. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/meta-arguments?utm_source=chatgpt.com)

Example using `for_each`:

```hcl
locals {
  servers = {
    api    = "t3.small"
    worker = "t3.medium"
  }
}

resource "aws_instance" "server" {
  for_each = local.servers

  ami           = var.approved_ami_id
  instance_type = each.value

  tags = {
    Name = each.key
  }
}
```

Terraform addresses these instances as:

```text
aws_instance.server["api"]
aws_instance.server["worker"]
```

The keys provide stable identities. Removing `worker` targets that instance for removal.

**Rules worth explaining in the interview:**

- A block cannot use both `count` and `for_each`.
- `for_each` accepts a map or a set of strings.
- Its instance keys must be known before remote operations.
- Sensitive values cannot be used as instance keys.
- Resource references normally establish implicit dependencies; use `depends_on` for dependencies that those references do not express. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/meta-arguments/for_each?utm_source=chatgpt.com)

For newer Terraform versions, also recognize **`action_trigger` inside `lifecycle`**. With a compatible provider, it invokes provider-defined actions at configured resource lifecycle events. Terraform actions require version 1.14 or newer. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/invoke-actions?utm_source=chatgpt.com)
