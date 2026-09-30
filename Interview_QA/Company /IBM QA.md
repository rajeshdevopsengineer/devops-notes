Below are interview-ready answers for all **30 IBM questions**. Personal-experience answers are templates—replace the project details with work you have actually done.

**1. Tell me about yourself.**

Keep your introduction focused on your experience, responsibilities, technical strengths, and one relevant example.

**Sample answer:**

> “I’m [Name], a DevOps engineer with [your experience] working on [application or project type]. My responsibilities include maintaining CI/CD pipelines, provisioning infrastructure with Terraform, automating server configuration with Ansible, and supporting application deployments on AWS.
>
> I work with Git, Linux, Docker, and Kubernetes, and I help troubleshoot pipeline and production issues using logs and monitoring tools.
>
> One example of my work is [a genuine automation or incident example], where I [explain your contribution]. I’m looking for a role where I can deepen my skills in reliable deployments, infrastructure automation, and cloud operations.”

Be ready to explain every technology and example you mention.

---

**2. What are your day-to-day activities?**

A practical answer should explain your routine and how you prioritize work.

Typical activities include:

- Reviewing alerts, application health, and overnight pipeline results.
- Investigating failed builds and deployments.
- Processing infrastructure and access-change requests.
- Reviewing Terraform plans and infrastructure pull requests.
- Maintaining Ansible playbooks and deployment scripts.
- Supporting dev, QA, and production releases.
- Reviewing vulnerabilities and coordinating remediation.
- Updating dashboards, alerts, and runbooks.
- Working with developers and operations teams on incidents.

**Sample answer:**

> “I start by checking service health, alerts, and failed pipelines. I then prioritize production issues and planned releases. During the day, I work on infrastructure changes, automation, deployment support, and troubleshooting. Changes go through review and the appropriate validation before production.”

Distinguish your routine support work from longer-term improvement projects.

---

**3. What is Terraform, and how does it work?**

Terraform is an **infrastructure-as-code tool**. You describe the infrastructure you want, and Terraform uses providers to interact with cloud or platform APIs.

For example, you can describe an EC2 instance, subnet, security group, or EKS cluster in configuration.

Its normal workflow is:

1. **Write configuration:** Define resources, variables, and provider requirements.
2. **Initialize:** Download providers/modules and configure the backend.
3. **Plan:** Compare the desired configuration with Terraform’s refreshed view of managed infrastructure.
4. **Apply:** Execute the proposed operations.
5. **Record state:** Maintain the relationship between configuration and real resources.

```bash
terraform init
terraform fmt -check
terraform validate
terraform plan
terraform apply
```

Terraform builds a dependency graph. For example, an instance referencing a subnet cannot be created until that subnet exists. Terraform can execute independent operations concurrently. :chatgpt-content-reference{index="0"}

It is declarative: you specify the desired result rather than manually listing every API operation.

---

**4. Where do you run Terraform—locally or on a server?**

Terraform can run wherever the CLI has the required configuration, credentials, network connectivity, and backend access.

Common execution locations include:

- A developer’s laptop.
- A Jenkins agent.
- An Azure DevOps or GitHub Actions runner.
- An ephemeral container or VM.
- A managed remote execution platform.

**Typical organizational approach:**

> “I use my local environment for development, formatting, validation, and permitted testing. Shared and production infrastructure changes run through CI/CD, where we have reviews, controlled credentials, state locking, and an execution history.”

A pipeline commonly performs:

```text
Pull request → Validation → Plan → Review/approval → Apply
```

**Important distinction:** The execution location and state-storage location are separate. Terraform can execute on a Jenkins agent while storing its state in S3.

---

**5. What is a `tfstate` file?**

A Terraform state file records information about resources Terraform manages.

It commonly contains:

- Resource addresses.
- Cloud resource IDs.
- Provider information.
- Resource attributes.
- Dependency-related metadata.
- Output values.

For example, state associates:

```text
aws_instance.web
```

with a particular AWS instance:

```text
i-0123456789abcdef0
```

Terraform uses that association to identify the existing object during future operations. :chatgpt-content-reference{index="1"}

State can contain sensitive information. Marking a variable or output `sensitive` does not necessarily prevent its value from being stored in state. :chatgpt-content-reference{index="2"}

A state file is not a backup of your application database or the contents of an EC2 disk.

---

**6. Where do you store the state file in your organization?**

A common AWS design stores state in a restricted S3 bucket using a remote backend.

**Example:**

```hcl
terraform {
  backend "s3" {
    bucket       = "example-org-terraform-state"
    key          = "orders/prod/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

The bucket must already exist.

For team use, I would configure:

- Bucket versioning for recovery.
- Encryption.
- Restricted IAM permissions.
- State locking.
- Separate state and access boundaries for environments.
- Audit logging where required.

Current Terraform supports native S3 locking through `use_lockfile = true`. DynamoDB-based locking is deprecated. :chatgpt-content-reference{index="3"}

**Sample interview answer:**

> “Our shared state is stored remotely, with encryption, versioning, and access controls. State locking prevents concurrent operations from writing conflicting changes.”

Use your actual backend in the answer—S3, Azure Blob Storage, or another supported backend.

---

**7. What does the state file actually do?**

Its main purpose is to connect Terraform configuration to existing infrastructure.

| Without the mapping | With the mapping |
|---|---|
| A resource declaration is just desired configuration | Terraform knows which real object it represents |
| Removed configuration would lose its connection to the object | Terraform can identify a managed object that may need removal |
| Resource relationships would be harder to reconstruct | Terraform retains relevant management metadata |

For example:

```hcl
resource "aws_instance" "web" {
  # Configuration
}
```

State tells Terraform which instance belongs to that address.

Terraform still queries providers during normal planning to discover current resource attributes. State is not a continuously updated monitoring database. :chatgpt-content-reference{index="4"}

---

**8. Another engineer changes an instance through the UI. What happens when you run `terraform plan`?**

That can create **configuration drift**.

By default, Terraform reads the current state of managed resources through the provider, then compares it with your configuration.

The result depends on the changed attribute.

**Example: a managed tag changed manually**

Your configuration requires:

```hcl
tags = {
  Environment = "prod"
}
```

Someone changes the tag to `manual-prod` in the AWS console.

An illustrative plan excerpt is:

```text
# aws_instance.app will be updated in-place
~ resource "aws_instance" "app" {
    id = "i-0123456789abcdef0"

  ~ tags = {
      ~ "Environment" = "manual-prod" -> "prod"
    }
}

Plan: 0 to add, 1 to change, 0 to destroy.
```

Terraform proposes restoring the configured value.

Other possibilities include:

| Situation | Possible plan |
|---|---|
| Managed attribute changed | Update it to match configuration |
| Change requires replacement | Destroy/create or create/destroy |
| Managed instance deleted manually | Create a replacement |
| Attribute is ignored or computed-only | No corrective resource action may be proposed |

If the manual change is intentional, review and update the code to represent it. Otherwise, applying the plan may reverse it. :chatgpt-content-reference{index="5"}

`terraform plan` itself does not perform those infrastructure changes.

---

**9. What do `+` and `−` mean in Terraform plan output?**

| Symbol | Meaning |
|---|---|
| `+` | Create a resource |
| `-` | Destroy a resource |
| `~` | Update a resource in place |
| `-/+` | Destroy and then create a replacement |
| `+/-` | Create a replacement before destroying the old object |
| `<=` | Read a data source, commonly when the read is deferred until apply |

Example:

```text
Plan: 2 to add, 1 to change, 1 to destroy.
```

This proposes two creations, one in-place update, and one destruction.

A replacement normally contributes to both the addition and destruction counts.

Review the individual attribute changes, not just the totals. An in-place update can still interrupt service depending on the resource and attribute. The plan describes proposed operations, not a guarantee of zero downtime. :chatgpt-content-reference{index="6"}

---

**10. What if you run `terraform apply` without running `terraform plan` first?**

A separate `terraform plan` command is not mandatory.

When you run:

```bash
terraform apply
```

Terraform normally:

1. Reads the configuration and managed infrastructure.
2. Generates an execution plan.
3. Displays it.
4. Requests confirmation.
5. Executes it after approval.

Therefore, it does not fail merely because you skipped the separate planning command.

If someone changed an instance through the UI, the automatically generated plan can include actions to reconcile that drift.

With:

```bash
terraform apply -auto-approve
```

Terraform skips the interactive confirmation.

With:

```bash
terraform apply tfplan
```

Terraform applies a previously saved plan rather than generating a new plan for approval. :chatgpt-content-reference{index="7"}

Errors can still occur because of permissions, stale plans, invalid configuration, resource constraints, or provider failures.

---

**11. An application has a vulnerability. How would you handle it?**

I would follow a repeatable process:

1. **Validate:** Confirm the affected component, version, and finding.
2. **Assess exposure:** Determine whether the vulnerable code is reachable and exploitable in our deployment.
3. **Prioritize:** Consider severity, business impact, exposure, and known exploitation.
4. **Contain if necessary:** Reduce access to the vulnerable functionality.
5. **Remediate:** Patch, upgrade, change configuration, or replace the affected component.
6. **Verify:** Retest functionality and confirm the vulnerability is addressed.
7. **Document:** Record evidence and close the finding through the agreed process.

CISA’s Known Exploited Vulnerabilities catalog is a useful prioritization input because it identifies vulnerabilities with evidence of exploitation. :chatgpt-content-reference{index="8"}

A vulnerability finding alone does not prove that the application has already been compromised.

---

**12. What would you do if the vulnerability is on a production server?**

Production requires both security remediation and controlled service recovery.

I would:

- Establish the affected systems and customer impact.
- Determine whether there is evidence of exploitation.
- Involve the security, application, and service owners.
- Apply suitable temporary containment.
- Test the fix and prepare a recovery plan.
- Deploy through an approved emergency or normal change process.
- Monitor application health and rescan afterward.

Where supported, use a canary or rolling deployment to validate the patched version before wider rollout.

**If compromise is suspected:** Follow the incident-response process, preserve evidence, contain the affected systems, and investigate exposed credentials or lateral movement. Patching alone may not remove an attacker’s access. NIST’s incident-response guidance treats detection, response, and recovery as coordinated activities. :chatgpt-content-reference{index="9"}

Avoid treating every security finding as a reason to shut down production without assessing its actual risk.

---

**13. A Python production vulnerability needs six months to fix. What is your approach?**

I would first challenge the assumption that all remediation must wait six months.

Determine what is vulnerable:

- Python itself.
- A direct dependency.
- A transitive dependency.
- Application code.
- Configuration or deployment behavior.

Then investigate faster options: a supported upgrade, backported patch, dependency replacement, or disabling the affected functionality.

If the permanent solution genuinely requires months:

1. **Remove or reduce the attack path.** Disable the vulnerable feature, restrict access, or isolate the affected component.
2. **Apply relevant compensating controls.** Reduce privileges, restrict unnecessary egress, validate inputs, and control access.
3. **Test those controls.** Confirm they address the actual exploit conditions.
4. **Increase relevant detection.** Monitor indicators associated with the vulnerability.
5. **Create a time-bound exception.** Record the owner, residual risk, controls, remediation milestones, and review date.
6. **Reassess regularly.** New exploitation evidence may require stronger action immediately.

A WAF can help with certain HTTP attack patterns, but it cannot address every Python runtime or library vulnerability.

If the remaining risk is unacceptable, remove the affected functionality or exposure until a safe implementation is available. A six-month development estimate is not, by itself, a justification to leave an exploitable production path open. Vulnerability management should include assessment, remediation, verification, and ongoing reporting. :chatgpt-content-reference{index="10"}

---

**14. Are you following any standards for handling these issues?**

Answer according to the standards and policies your project actually uses.

Common references include:

| Reference | Purpose |
|---|---|
| NIST SP 800-40 Rev. 4 | Enterprise patch-management planning |
| NIST SP 800-61 Rev. 3 | Incident-response guidance |
| OWASP ASVS | Application-security verification requirements |
| CIS Benchmarks | Security configuration and hardening guidance |
| Organizational policy | Remediation deadlines, ownership, exceptions, and approvals |

These provide different kinds of guidance; they are not interchangeable certifications. :chatgpt-content-reference{index="11"}

**Sample answer:**

> “I follow the organization’s vulnerability-management and change-management policies. Findings are prioritized by severity, exposure, exploitability, and business impact. Exceptions require an owner, justification, compensating controls, and an expiry date.”

Do not invent a universal deadline such as “every critical issue must be fixed within exactly 24 hours.” The applicable deadline depends on organizational and contractual requirements.

---

**15. Have you worked with Ansible?**

If it matches your experience, an answer could be:

> “I have used Ansible for package installation, service configuration, user management, and repeatable application setup. I maintain inventories and playbooks in Git, use variables for environment differences, and protect sensitive values with Vault or a secret-management integration.”

Then give one genuine example:

> “For example, I automated web-server configuration across several hosts, replacing a manual checklist with a reviewed playbook.”

Be prepared to explain the inventory, modules, privilege escalation, error handling, and how you verified the result.

---

**16. What is Ansible?**

Ansible is an automation tool commonly used for:

- Configuration management.
- Software installation.
- Application deployment.
- Server administration.
- Coordinating operational tasks.

Its main concepts are:

| Concept | Meaning |
|---|---|
| Inventory | Hosts and groups to manage |
| Playbook | YAML automation workflow |
| Play | Applies tasks to selected hosts |
| Task | Invokes an action or module |
| Module | Performs a specific operation |
| Role | Organizes reusable automation |

Ansible commonly connects to Linux hosts through SSH and does not require a persistent Ansible agent on those hosts. Most normal Linux modules require Python on the managed host.

Playbooks describe tasks and their desired results. Many modules are idempotent: they change a system only when its current state differs from what was requested. Arbitrary shell commands are not automatically idempotent. :chatgpt-content-reference{index="12"}

---

**17. Can you write an Ansible playbook?**

This example installs Nginx, publishes a page, and ensures the service is running on **Ubuntu hosts**.

`inventory.ini`:

```ini
[web]
web01 ansible_host=10.0.1.10
web02 ansible_host=10.0.2.10

[web:vars]
ansible_user=ubuntu
```

`web.yml`:

```yaml
---
- name: Configure web servers
  hosts: web
  become: true

  vars:
    welcome_message: "Application deployed using Ansible"

  tasks:
    - name: Install Nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true
        cache_valid_time: 3600

    - name: Publish the application page
      ansible.builtin.copy:
        dest: /var/www/html/index.html
        content: "<h1>{{ welcome_message }}</h1>\n"
        owner: root
        group: root
        mode: "0644"

    - name: Start and enable Nginx
      ansible.builtin.systemd_service:
        name: nginx
        state: started
        enabled: true
```

Run:

```bash
ansible-playbook -i inventory.ini web.yml
```

This assumes SSH access and appropriate sudo permissions are configured. Use `-K` if Ansible must prompt for a sudo password.

The `apt` task manages package installation, while `systemd_service` manages service state and startup behavior. :chatgpt-content-reference{index="13"}

For service-configuration changes, a handler can reload the service only when the relevant configuration task reports a change.

---

**18. How do you store credentials in Ansible?**

For encrypted credentials stored with an Ansible project, use **Ansible Vault**.

Example:

```bash
mkdir -p group_vars/web

ansible-vault create group_vars/web/vault.yml
```

Inside the editor, define the required secret variables. Ansible saves the file encrypted.

To encrypt an existing file:

```bash
ansible-vault encrypt secrets.yml
```

To edit an encrypted file:

```bash
ansible-vault edit group_vars/web/vault.yml
```

Run a playbook that loads encrypted variables:

```bash
ansible-playbook \
  -i inventory.ini \
  web.yml \
  --ask-vault-pass
```

Vault supports encrypting files or individual variable values, and Ansible can decrypt the required data during execution. :chatgpt-content-reference{index="14"}

**Operational controls:**

- Store the Vault password separately from the encrypted content.
- Prefer SSH keys or approved temporary access for host authentication.
- Use `no_log: true` on secret-consuming tasks.
- Disable diff output for sensitive files.
- Never print secret variables with debugging tasks.

Vault protects stored content; it does not protect a secret that your task explicitly exposes in logs. :chatgpt-content-reference{index="15"}

---

**19. What other approaches can securely store secret credentials?**

Use a central secret manager, such as:

- AWS Secrets Manager.
- Azure Key Vault.
- HashiCorp Vault.
- An approved CI/CD credential store.

Give the automation runner an identity with narrowly scoped access, then retrieve secrets at execution time.

**Example using AWS Secrets Manager:**

```yaml
vars:
  database_password: >-
    {{
      lookup(
        'amazon.aws.secretsmanager_secret',
        'prod/orders/database-password',
        region='ap-south-1'
      )
    }}
```

This assumes the secret contains a plain string. A JSON secret would require extracting the appropriate field.

The lookup requires the `amazon.aws` collection and its dependencies. Lookup plugins execute in the **Ansible controller context**, so the controller needs the credentials and permissions to read the secret. :chatgpt-content-reference{index="16"}

Use `no_log` and appropriate file permissions when consuming the value.

**Distinction:** Ansible Vault encrypts automation content. HashiCorp Vault is a separate centralized secrets-management product.

---

**20. How do you store variables in Ansible?**

Variables can be defined in several places:

| Location | Typical purpose |
|---|---|
| Playbook `vars` | Values specific to a play |
| `vars_files` | External variable files |
| `group_vars/` | Values shared by a host group |
| `host_vars/` | Values specific to one host |
| Role defaults | Easily overridden role settings |
| Registered variables | Results returned by tasks |
| Extra variables, `-e` | Values supplied for a particular execution |

Example:

```yaml
vars:
  app_port: 8080
  environment_name: dev

vars_files:
  - vars/common.yml
```

Typical project paths include:

```text
group_vars/all.yml
group_vars/web/main.yml
group_vars/web/vault.yml
host_vars/web01.yml
```

Pass an override:

```bash
ansible-playbook -i inventory.ini web.yml \
  -e '{"welcome_message":"Release 1.2 deployed"}'
```

Or load an extra-variable file:

```bash
ansible-playbook -i inventory.ini web.yml \
  -e @vars/release.yml
```

Extra variables have the highest variable precedence. Avoid putting plaintext passwords directly on the command line. :chatgpt-content-reference{index="17"}

---

**21. How do you debug Ansible errors? What is the command?**

Start with increased verbosity:

```bash
ansible-playbook -i inventory.ini web.yml -vvv
```

For connection troubleshooting:

```bash
ansible-playbook -i inventory.ini web.yml -vvvv
```

Ansible documents `-vvv` as a useful starting level and `-vvvv` for connection debugging. :chatgpt-content-reference{index="18"}

Other useful checks:

```bash
# Validate syntax.
ansible-playbook -i inventory.ini web.yml --syntax-check

# Investigate a single host.
ansible-playbook -i inventory.ini web.yml \
  --limit web01 -vvv

# Preview supported changes.
ansible-playbook -i inventory.ini web.yml --check
```

For non-sensitive files, add `--diff`. Check mode depends on module support and cannot guarantee that a real execution will succeed. :chatgpt-content-reference{index="19"}

| Error | What to inspect |
|---|---|
| `UNREACHABLE` | Address, DNS, routing, SSH access, key, username |
| Permission denied | File permissions and privilege escalation |
| Undefined variable | Variable name, scope, and loading |
| Module failure | Parameters, dependencies, target environment |
| Service failure | Application configuration and service logs |

Keep secret values out of debugging output.

---

**22. How do you execute one command without writing a playbook?**

Use an **Ansible ad-hoc command**.

General structure:

```bash
ansible <host-pattern> \
  -i <inventory> \
  -m <module> \
  -a "<module-arguments>"
```

Run `uptime` on the web servers:

```bash
ansible web \
  -i inventory.ini \
  -m ansible.builtin.command \
  -a "uptime"
```

Check Ansible connectivity:

```bash
ansible web \
  -i inventory.ini \
  -m ansible.builtin.ping
```

Ansible’s `ping` module checks Ansible connectivity and Python execution; it is not an ICMP ping.

For a privileged operation, add `-b`, and add `-K` if a become-password prompt is required.

Use `command` for ordinary commands. Use `shell` only when shell features such as pipes or redirection are actually required. :chatgpt-content-reference{index="20"}

---

**23. What Ansible modules have you worked with?**

Select the modules you have genuinely used and be ready to explain their parameters.

| Module | Purpose |
|---|---|
| `ansible.builtin.apt` | Debian/Ubuntu package management |
| `ansible.builtin.dnf` | DNF-based package management |
| `ansible.builtin.package` | Generic package-management interface |
| `ansible.builtin.systemd_service` | Manage systemd services |
| `ansible.builtin.copy` | Copy files or literal content |
| `ansible.builtin.template` | Render Jinja2 templates |
| `ansible.builtin.file` | Manage paths, permissions, and links |
| `ansible.builtin.lineinfile` | Ensure a line exists or is changed |
| `ansible.builtin.user` | Manage users |
| `ansible.builtin.group` | Manage groups |
| `ansible.builtin.git` | Check out Git repositories |
| `ansible.builtin.get_url` | Download files |
| `ansible.builtin.unarchive` | Extract archives |
| `ansible.builtin.uri` | Interact with HTTP endpoints |
| `ansible.builtin.command` | Execute a command |
| `ansible.builtin.shell` | Execute through a shell |
| `ansible.builtin.debug` | Display non-sensitive diagnostic information |
| `ansible.builtin.assert` | Validate conditions |

For SSH public keys, use `ansible.posix.authorized_key`. It belongs to the `ansible.posix` collection, rather than `ansible.builtin`. :chatgpt-content-reference{index="21"}

---

**24. How do you deploy an application on AWS? Which services do you use?**

Start by naming the application’s actual runtime: EC2, ECS, EKS, Lambda, or another platform.

**Example for a container application:**

1. Developers merge reviewed code.
2. CI builds and tests the application.
3. The pipeline scans and publishes the image to ECR.
4. CD deploys the approved image to EKS.
5. A load balancer exposes the application.
6. Monitoring verifies the release.
7. The previous release remains available for recovery.

Typical supporting services include:

| Service | Purpose |
|---|---|
| VPC and subnets | Network placement and isolation |
| ECR | Container-image storage |
| EKS or ECS | Container execution/orchestration |
| ALB | HTTP/HTTPS application traffic |
| Route 53 | DNS |
| ACM | Supported TLS certificate integration |
| RDS | Managed relational database |
| Secrets Manager | Application secrets |
| IAM | Access control |
| CloudWatch | Monitoring and logs |

For an EC2-based application, the runtime might instead be private EC2 instances in an Auto Scaling group behind an ALB.

Explain which parts you personally configured and which were maintained by another team.

---

**25. What other services do you use when deploying to EKS?**

EKS is one part of the application platform.

| Supporting component | Purpose |
|---|---|
| VPC, subnets, security groups | Cluster and workload networking |
| ECR | Application images |
| EC2 managed node groups or Fargate | Pod compute capacity |
| AWS Load Balancer Controller | Integration with ALB/NLB |
| Route 53 and ACM | DNS and TLS |
| IAM and workload identity | AWS permissions |
| Secrets Manager | Application secrets |
| EBS CSI driver | EBS-backed persistent volumes where needed |
| CloudWatch or other telemetry backends | Logs and metrics |

The AWS Load Balancer Controller can provision ALBs for supported Kubernetes Ingress configurations. :chatgpt-content-reference{index="22"}

The EBS CSI driver provides the Kubernetes integration for EBS-backed storage; storage is only needed when the workload requires it. :chatgpt-content-reference{index="23"}

**Keep the identities separate:**

- Node permissions support node operations, including the necessary image-pull access.
- Application pods use their own permitted AWS identity, such as EKS Pod Identity.
- Human and CI access to the Kubernetes API is managed separately through cluster-access configuration and authorization. :chatgpt-content-reference{index="24"}

Also ensure the deployment runner can reach the cluster API, especially when it is private.

---

**26. Have you worked with shell and Python scripting?**

A useful answer explains when you choose each language.

**Sample answer:**

> “I use shell scripting for short Linux and command-line workflows, such as health checks, log processing, and deployment wrappers. I use Python when the task needs more structured data handling, API pagination, reusable logic, or more extensive error handling.”

Examples:

| Shell | Python |
|---|---|
| Coordinate existing CLI commands | Call cloud APIs |
| Process local logs | Process nested JSON |
| Run deployment checks | Generate inventory reports |
| Perform simple scheduled tasks | Implement reusable automation |

Discuss exit codes, input validation, logging, credentials, and failure handling—not only the language syntax.

---

**27. Where have you used Python, and for what purpose?**

A concrete example is collecting an AWS inventory report through Boto3.

**Illustrative script: list running EC2 instances in one region**

```python
#!/usr/bin/env python3

import csv
import sys

import boto3


ec2 = boto3.client("ec2", region_name="ap-south-1")
paginator = ec2.get_paginator("describe_instances")

writer = csv.writer(sys.stdout)
writer.writerow([
    "InstanceId",
    "Name",
    "InstanceType",
    "AvailabilityZone",
])

pages = paginator.paginate(
    Filters=[
        {
            "Name": "instance-state-name",
            "Values": ["running"],
        }
    ]
)

for page in pages:
    for reservation in page["Reservations"]:
        for instance in reservation["Instances"]:
            tags = {
                tag["Key"]: tag["Value"]
                for tag in instance.get("Tags", [])
            }

            writer.writerow([
                instance["InstanceId"],
                tags.get("Name", ""),
                instance["InstanceType"],
                instance["Placement"]["AvailabilityZone"],
            ])
```

Run:

```bash
python3 ec2_inventory.py > running-instances.csv
```

The script requires Boto3 and appropriate AWS permissions. It uses the SDK’s credential configuration rather than hardcoded access keys.

Pagination ensures the report handles multiple API response pages. :chatgpt-content-reference{index="25"}

**How to explain the purpose:**

> “This automates inventory collection that would otherwise require manually checking the console. The same pattern can support missing-tag reports, resource ownership checks, or operational reporting.”

This example covers one account and region; broader reporting requires explicitly handling the additional accounts and regions.

---

**28. Have you worked with monitoring tools? What was your role?**

Select the tools that match your experience.

| Tool | Typical responsibility |
|---|---|
| CloudWatch | Cloud metrics, logs, dashboards, and alarms |
| Prometheus | Application and infrastructure metric collection |
| Grafana | Dashboards and investigation |
| Alertmanager | Alert grouping, routing, and silences |
| Centralized log platform | Searching and correlating application events |

**Example answer:**

> “My role includes configuring metric and log collection, creating dashboards, defining alerts, and using those signals during incidents. I also review noisy alerts and make sure actionable alerts have an owner and runbook.”

Useful metrics include:

- Request rate and error rate.
- p95/p99 latency.
- CPU, memory, and disk usage.
- Pod restarts and unavailable replicas.
- Queue backlog.
- Database connections and latency.

Prometheus provides metric collection and time-series querying. :chatgpt-content-reference{index="26"}

For EC2, guest memory and filesystem-space metrics generally need an agent or custom instrumentation. The CloudWatch agent can collect additional system metrics and logs. :chatgpt-content-reference{index="27"}

Explain one alert you configured, why its threshold or condition mattered, and what action followed.

---

**29. How do you disable root login on a particular server?**

If the requirement is to disable **direct root SSH login**, set:

```text
PermitRootLogin no
```

Follow a controlled procedure on the target server.

**1. Confirm alternative access**

Verify that a non-root administrator can log in and use `sudo`. Keep an existing administrative session open while testing.

**2. Edit the active SSH server configuration**

```bash
sudoedit /etc/ssh/sshd_config
```

Set the directive in the effective configuration, accounting for included files and any root-relevant `Match` rules.

`PermitRootLogin no` disables root SSH login. `prohibit-password` only disables certain authentication methods for root and can still permit key-based login. :chatgpt-content-reference{index="28"}

**3. Validate configuration syntax**

```bash
sudo /usr/sbin/sshd -t
```

**4. Check the effective setting**

```bash
sudo /usr/sbin/sshd -T \
  | awk '$1 == "permitrootlogin" { print }'
```

Expected output:

```text
permitrootlogin no
```

If `Match` rules are present, evaluate the relevant connection context:

```bash
sudo /usr/sbin/sshd -T \
  -C user=root,addr=198.51.100.20,host=client.example.com \
  | awk '$1 == "permitrootlogin" { print }'
```

Replace the example source address and hostname with the applicable client details. The `-C` option supplies connection information for evaluating conditional configuration. :chatgpt-content-reference{index="29"}

**5. Reload the appropriate service after validation passes**

Ubuntu/Debian:

```bash
sudo systemctl reload ssh
```

RHEL/Amazon Linux:

```bash
sudo systemctl reload sshd
```

**6. Verify with new connections**

Confirm that non-root administrative access still works and root SSH login is rejected before closing the existing session.

This changes SSH access; it does not disable the root account’s local or sudo-based administrative role.

---

**30. What is the filename and full path you change?**

The main SSH server configuration file is:

```text
/etc/ssh/sshd_config
```

Some systems also load configuration snippets from:

```text
/etc/ssh/sshd_config.d/*.conf
```

For example:

```text
/etc/ssh/sshd_config.d/00-disable-root-login.conf
```

That snippet would contain:

```text
PermitRootLogin no
```

The snippet is effective only when the main configuration includes it. OpenSSH uses the first obtained value for most settings, so file ordering matters; a later file is not automatically an override. Ubuntu commonly includes its snippet directory near the beginning of the main file. Always verify the effective result with `sshd -T`. :chatgpt-content-reference{index="30"}
