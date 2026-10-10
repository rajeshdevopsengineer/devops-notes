**1. A database fails over from A to B. How do you handle writes during that interval?**

The application must reconnect to the new primary and handle interrupted transactions safely. **A lost connection does not prove that a write failed—the database might have committed it before the response was lost.**

For an AWS RDS Multi-AZ deployment, failover updates the database endpoint’s DNS to the new primary. Existing connections may break, so applications must reconnect and avoid retaining stale DNS results or broken pooled connections. [Amazon Relational Database Service](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZ.Failover.html?utm_source=chatgpt.com)

I would implement the following:

1. **Use the writer endpoint.**  
   Avoid hardcoding individual database IP addresses. Let the database’s HA mechanism control promotion and prevent the old primary from accepting conflicting writes.

2. **Recover connections.**  
   Remove broken connections from the pool and establish new ones against the writer endpoint. RDS Proxy can help manage connections during failover, but application-level recovery is still necessary. [Amazon Relational Database Service](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy.html?utm_source=chatgpt.com)

3. **Retry transactions with bounded backoff.**  
   For retryable failures, retry the complete transaction with exponential backoff and jitter. Do not repeatedly retry indefinitely.

4. **Make business operations idempotent.**  
   Assign a persistent operation ID, such as an order-request ID, and enforce uniqueness. Retrying the same operation must not create another order or payment. Idempotency is essential when the first request’s outcome is uncertain. [aws.amazon.com](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/?utm_source=chatgpt.com)

| Situation | Handling |
|---|---|
| Connection could not be established | Reconnect and retry using the same operation ID |
| Transaction definitely rolled back | Retry the complete transaction |
| Commit might have succeeded, but the response was lost | Check the operation’s recorded status before repeating it |

If the failover exceeds the request deadline, either return a clear retryable error or durably queue the operation and report it as **pending**. Do not report completion before confirming it.

Durability also depends on replication. RDS Multi-AZ DB instances use synchronous standby replication; an asynchronous disaster-recovery replica can have a nonzero recovery point because of replication lag. [Amazon Relational Database Service](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZSingleStandby.html?utm_source=chatgpt.com)

**2. What is a Lambda cold start?**

A cold start occurs when Lambda must initialize a new execution environment before running your handler.

Initialization can include:

- Starting the runtime.
- Loading application code and dependencies.
- Running initialization code outside the handler.
- Initializing extensions and application clients.

A subsequent invocation can reuse an existing environment and avoid this initialization overhead. New environments can be required during initial invocation, scaling, or environment replacement. [AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtime-environment.html?utm_source=chatgpt.com)

To reduce cold-start impact:

- Keep dependencies and initialization work small.
- Initialize reusable SDK or HTTP clients outside the handler.
- Avoid unnecessary network calls during initialization.
- Tune memory allocation based on measured performance; additional CPU can reduce initialization time for some workloads.
- Use **provisioned concurrency** when predictable startup latency is required.

Provisioned concurrency prepares environments in advance. It applies to a published version or alias, not `$LATEST`. Requests exceeding the provisioned capacity can still use on-demand environments and experience cold starts. **Reserved concurrency limits/reserves execution capacity; it does not preinitialize environments.** [AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/provisioned-concurrency.html?utm_source=chatgpt.com)

Another option is **SnapStart**, where supported. It restores execution environments from snapshots of initialized state. [AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/snapstart.html?utm_source=chatgpt.com)

For troubleshooting, inspect initialization information such as `Init Duration` in CloudWatch logs and compare cold and warm invocation latency.

**3. What are the commonly used AWS CDK commands?**

AWS CDK lets you define infrastructure using programming languages and synthesize CloudFormation templates.

| Command | Purpose |
|---|---|
| `cdk init app --language typescript` | Initialize a TypeScript CDK application in a suitable new directory |
| `cdk bootstrap aws://111122223333/ap-south-1` | Prepare an account and Region for CDK deployments |
| `cdk list` | List stacks in the application |
| `cdk synth` | Generate CloudFormation templates and deployment assets |
| `cdk diff MyStack` | Compare the proposed stack with its deployed configuration |
| `cdk deploy MyStack` | Deploy a stack |
| `cdk deploy --all` | Deploy all applicable stacks |
| `cdk destroy MyStack` | Delete a deployed stack |
| `cdk context` | Inspect cached context information |

These commands form the main CDK development and deployment workflow. [docs.aws.amazon.com](https://docs.aws.amazon.com/cdk/v2/guide/cli.html?utm_source=chatgpt.com)

Bootstrapping creates supporting resources for deployment, such as asset storage and IAM roles, in the target account and Region. [docs.aws.amazon.com](https://docs.aws.amazon.com/cdk/v2/guide/bootstrapping.html?utm_source=chatgpt.com)

Before running deployment commands, configure the required language toolchain and AWS credentials for the intended environment.

**4. How would you implement Terraform in a CD pipeline?**

I would separate **validation, planning, approval, and execution**, and apply the exact plan that was reviewed.

A typical workflow is:

1. Developer raises a pull request.
2. Pipeline checks formatting and validates configuration.
3. Infrastructure-security and policy checks run.
4. A plan is generated for review.
5. Approved code merges into a protected branch.
6. The deployment pipeline generates the environment’s final plan.
7. Production approval is obtained.
8. The saved plan is applied.
9. Infrastructure and application smoke checks run.

HashiCorp documents this plan-and-apply approach for automated workflows. [HashiCorp Developer](https://developer.hashicorp.com/terraform/tutorials/automation/automate-terraform?utm_source=chatgpt.com)

Core commands:

```bash
terraform fmt -check -recursive
terraform init -input=false
terraform validate

terraform plan \
  -input=false \
  -lock-timeout=5m \
  -out=tfplan

terraform show -no-color tfplan
```

After approval:

```bash
terraform apply \
  -input=false \
  -lock-timeout=5m \
  tfplan
```

Passing the saved plan prevents the apply stage from silently generating a different plan. If the saved plan becomes stale, generate a new plan and repeat the review. [HashiCorp Developer](https://developer.hashicorp.com/terraform/cli/commands/apply?utm_source=chatgpt.com)

Important implementation details:

- Use short-lived deployment credentials and narrowly scoped roles.
- Use remote state with locking and recoverable versions.
- Separate state appropriately between environments.
- Serialize applies against the same state.
- Pin Terraform and provider versions, and commit `.terraform.lock.hcl`.
- Protect saved plans because they can contain sensitive values.
- Keep plan and apply execution environments compatible.

Terraform expects compatible platform, configuration, and provider versions when applying a saved plan. [HashiCorp Developer](https://developer.hashicorp.com/terraform/tutorials/automation/automate-terraform?utm_source=chatgpt.com)

**5. Ten developers checked in code. How do you remove developer 10’s changes?**

For a commit already shared with other developers, use **`git revert`**. It creates a new commit that reverses the selected change while preserving history. [git-revert Documentation](https://git-scm.com/docs/git-revert?utm_source=chatgpt.com)

First identify the exact commit, then create a revert branch:

```bash
git fetch origin
git switch -c revert-developer-10 origin/main

git log --oneline
git show abc1234

git revert abc1234
git push -u origin revert-developer-10
```

Here, `abc1234` represents the commit to remove. Raise a pull request, run the checks, and merge the revert.

If conflicts occur:

```bash
# Resolve the conflicting files
git add resolved-file
git revert --continue
```

To cancel the operation:

```bash
git revert --abort
```

If the changes entered through a merge commit, you may need:

```bash
git revert -m 1 def5678
```

Verify which parent represents the main branch before selecting `-m 1`; the option chooses the mainline parent. [git-revert Documentation](https://git-scm.com/docs/git-revert?utm_source=chatgpt.com)

Do not assume that removing the tenth developer’s commit leaves everything working. Other changes may depend on it, so review the resulting code and run tests.

**6. Someone manually changed an EC2 configuration created through Terraform. How do you fix it?**

This is **Terraform drift**: the actual infrastructure differs from the managed configuration.

First inspect the change:

```bash
terraform plan -out=drift.tfplan
terraform show drift.tfplan
```

Terraform refreshes provider information and proposes changes based on the configuration. [HashiCorp Developer](https://developer.hashicorp.com/terraform/tutorials/state/resource-drift?utm_source=chatgpt.com)

Then decide which state is intended:

- **The manual change was unauthorized:** Review and apply the plan that restores the configured value.
- **The manual change was approved:** Update the Terraform configuration, review it in Git, and verify the resulting plan.

To restore the reviewed configuration:

```bash
terraform apply drift.tfplan
```

For example, if code specifies `t3.medium` and someone changes the instance to `t3.large`, the plan may propose restoring the configured instance type.

Before applying, review potential interruption or replacement. Check CloudTrail when you need to establish who made the change.

Do not delete state to fix drift. Also remember that Terraform only reconciles the attributes it manages; intentional `ignore_changes` settings can suppress corresponding updates.

**7. A payment application in Lambda intermittently cannot connect to an external API. How do you investigate and fix it?**

I would investigate connectivity while protecting the payment’s transaction outcome.

**First, classify the failure**

| Failure | Areas to investigate |
|---|---|
| DNS error | Resolver configuration, hostname, DNS behavior |
| Connection timeout | Routing, NAT, firewall rules, provider availability |
| TLS failure | Certificate trust, hostname, protocol configuration |
| HTTP 401/403 | Credentials, access restrictions, provider policies |
| HTTP 429 | Rate limits and excessive concurrency |
| HTTP 5xx | Provider failure or overload |
| Timeout after sending the payment | An uncertain transaction outcome requiring reconciliation |

Correlate CloudWatch logs and traces using the order ID, application request ID, and provider request ID. Record useful timing and status information without logging card data or credentials.

**Check Lambda networking**

For a VPC-connected function, inspect:

- Subnet routes and the outbound internet path.
- NAT availability for the intended IPv4 internet access.
- Security-group outbound rules.
- NACL rules, including return traffic.
- DNS configuration.
- NAT connection or port exhaustion during high concurrency.

Attaching Lambda to a public subnet does not itself provide internet access. AWS documents the appropriate private-subnet/NAT configuration. [AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/configuration-vpc-internet.html?utm_source=chatgpt.com)

Also check Lambda-specific networking errors and limits when failures correlate with concurrency or DNS behavior. [AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/troubleshooting-networking.html?utm_source=chatgpt.com)

**Bound the dependency call**

Configure connection and response timeouts within the remaining Lambda and caller deadline. Reuse HTTP clients where appropriate, and handle stale connections.

Use bounded retries for eligible transient failures. Honor the provider’s retry guidance and `Retry-After` response where applicable.

**Protect against duplicate payments**

Persist one operation ID and reuse the same provider idempotency key and payload for retries, where the provider supports this behavior. Provider-specific key retention and retry semantics matter; Stripe, for example, documents how repeated requests with the same key are handled. [docs.stripe.com](https://docs.stripe.com/api/idempotent_requests?utm_source=chatgpt.com)

If a request times out after submission:

1. Mark the payment as pending or requiring reconciliation.
2. Query the provider using its payment reference.
3. Process verified webhook updates.
4. Retry only according to the provider’s documented semantics.
5. Reconcile local and provider records.

A timeout must not automatically become either “payment failed” or a new charge attempt with a new key.

For prolonged outages, use a circuit breaker and, if the business flow permits it, durable asynchronous processing.

**8. What is the purpose of blue-green deployment, and how do you switch back?**

Blue-green deployment maintains two application versions or environments:

- **Blue:** The currently serving version.
- **Green:** The new version being prepared and validated.

Its purpose is to validate the new release before switching traffic and retain a working release for rapid rollback.

The process is:

1. Deploy green.
2. Run readiness, integration, and smoke checks.
3. Switch traffic to green.
4. Monitor technical and business outcomes.
5. Keep blue available during the rollback window.
6. Switch traffic back if the release fails.

For Lambda, published versions and an alias can provide this mechanism. An alias points clients to a selected version and can support weighted routing. [AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/configuration-aliases.html?utm_source=chatgpt.com)

Suppose blue is version `41` and green is `42`. Outside an active CodeDeploy-managed rollout, a direct promotion could be:

```bash
aws lambda update-alias \
  --function-name payment-api \
  --name prod \
  --function-version 42 \
  --routing-config '{"AdditionalVersionWeights":{}}'
```

Rollback:

```bash
aws lambda update-alias \
  --function-name payment-api \
  --name prod \
  --function-version 41 \
  --routing-config '{"AdditionalVersionWeights":{}}'
```

Clearing additional weights ensures traffic is directed to the selected version. [AWS CLI 2.37.5 Command Reference](https://docs.aws.amazon.com/cli/latest/reference/lambda/update-alias.html?utm_source=chatgpt.com)

CodeDeploy can manage Lambda traffic shifting and configured automatic rollback. If it owns the rollout, use its deployment controls rather than changing the alias independently mid-deployment. [AWS CodeDeploy](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-steps-lambda.html?utm_source=chatgpt.com)

For EC2 applications, traffic can be switched between appropriately configured load-balancer target groups.

Keep database changes compatible with both releases. Switching application traffic does not reverse database migrations or undo completed payments.

**9. Write a script to check whether an external API is reachable before starting a request.**

Use a **read-only health endpoint**. This script requires a successful HTTP response before allowing the caller to proceed.

Download: check_external_api.sh[check_external_api.sh](sandbox:/workspace/scratch/cb08965e1c64/check_external_api.sh)

```bash
#!/usr/bin/env bash
set -euo pipefail

# Exit codes: 0 = healthy, 1 = failed check, 2 = invalid usage/setup.
if [[ $# -ne 1 ]]; then
  printf 'Usage: %s <http-or-https-health-url>\n' "$0" >&2
  exit 2
fi

health_url=$1
case "$health_url" in
  http://*|https://*) ;;
  *) printf 'Provide an HTTP or HTTPS health endpoint.\n' >&2; exit 2 ;;
esac

if ! command -v curl >/dev/null 2>&1; then
  printf 'curl is required.\n' >&2
  exit 2
fi

if http_code=$(curl \
  --silent --show-error \
  --proto '=http,https' \
  --connect-timeout 3 \
  --max-time 8 \
  --output /dev/null \
  --write-out '%{http_code}' \
  -- "$health_url"); then

  case "$http_code" in
    2[0-9][0-9])
      printf 'Health check passed: HTTP %s\n' "$http_code"
      exit 0
      ;;
    *)
      printf 'Endpoint responded with HTTP %s; expected 2xx.\n' "$http_code" >&2
      exit 1
      ;;
  esac
else
  curl_exit=$?
  printf 'Health request failed: curl exit %s (DNS, TLS, connection, or timeout).\n' "$curl_exit" >&2
  exit 1
fi
```

The script sets connection and overall time limits, discards the response body, and leaves HTTPS certificate validation enabled. [How To Use](https://curl.se/docs/manpage.html?utm_source=chatgpt.com)

Example usage:

```bash
chmod +x check_external_api.sh

./check_external_api.sh "https://api.example.com/health" &&
  python3 payment_client.py --order-id ORDER-123
```

Here, `payment_client.py` represents your existing application client. It runs only if the check exits successfully.

Important distinctions:

- HTTP 401, 403, or 503 proves the endpoint responded, but fails this health check.
- Use an authorized health endpoint if authentication is required.
- A successful precheck does not guarantee the next request succeeds. The payment client still needs timeouts, idempotency, and reconciliation.

The script passed syntax validation and ten behavior checks covering successful responses, HTTP errors, redirects, connection refusal, timeout, argument validation, and request gating.
