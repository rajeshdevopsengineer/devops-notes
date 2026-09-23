**1. How do you find the second-largest integer in an array?**

First, clarify whether the interviewer means the **second-largest distinct value**. For example, in `[9, 9, 5]`, the second-largest distinct value is `5`; the second element after descending sorting is `9`.

Assuming distinct values, use a single pass:

```python
def second_largest(numbers):
    largest = None
    second = None

    for number in numbers:
        if largest is None or number > largest:
            second, largest = largest, number

        elif number != largest and (
            second is None or number > second
        ):
            second = number

    if second is None:
        raise ValueError(
            "At least two distinct integers are required"
        )

    return second


print(second_largest([12, 35, 1, 10, 34, 35]))  # 34
print(second_largest([-8, -2, -5]))             # -5
```

**How it works:**

* When a new largest value appears, move the previous largest into `second`.
* Otherwise, update `second` if the value is smaller than `largest` but larger than the current `second`.
* Ignore duplicates of the largest value.

| Input              | Result                              |
| ------------------ | ----------------------------------- |
| `[12, 35, 34, 35]` | `34`                                |
| `[-8, -2, -5]`     | `-5`                                |
| `[2, 1]`           | `1`                                 |
| `[7, 7]`           | Error: insufficient distinct values |
| `[]`               | Error: insufficient distinct values |

**Complexity:** `O(n)` time and `O(1)` additional space.

Using `None` avoids the common mistake of initializing the values to zero, which fails for arrays containing only negative numbers.

---

**2. How do you set CPU and memory limits on a Linux machine?**

Usually, this means restricting a **process, service, or group of processes**. On a modern systemd-based Linux machine, use **cgroups**, typically managed through systemd.

For an existing service:

```bash
sudo systemctl set-property myapp.service \
  CPUQuota=50% \
  MemoryHigh=768M \
  MemoryMax=1G \
  MemorySwapMax=0
```

| Setting           | Meaning                                                                |
| ----------------- | ---------------------------------------------------------------------- |
| `CPUQuota=50%`    | Limits aggregate CPU time to approximately half of one logical CPU.    |
| `MemoryHigh=768M` | Applies memory pressure and reclaim when usage exceeds this threshold. |
| `MemoryMax=1G`    | Sets a hard memory boundary of 1 GiB.                                  |
| `MemorySwapMax=0` | Prevents the service’s cgroup from using swap.                         |

These settings apply to processes in the service’s cgroup, including its children. Systemd provides application-level resource management through the kernel’s cgroup controls. ([Red Hat Documentation][1])

**CPU quota interpretation matters:** `100%` means one logical CPU’s worth of execution time, not 100% of a multicore machine. `200%` allows two CPUs’ worth of aggregate execution time. A quota does not reserve that capacity.

Inspect settings and usage:

```bash
systemctl show myapp.service \
  -p CPUQuotaPerSecUSec \
  -p MemoryHigh \
  -p MemoryMax

systemd-cgtop
```

If memory cannot be reclaimed below the hard limit, the cgroup can experience an out-of-memory kill.

For shell-launched processes, `ulimit` provides other controls:

```bash
(
  ulimit -t 60
  ulimit -v 1048576
  exec ./my-program
)
```

Here:

* `-t 60` limits accumulated CPU time to 60 seconds; it does **not** impose a CPU percentage.
* `-v 1048576` limits virtual address space to 1 GiB; it does **not** mean 1 GiB of resident RAM.

For reliable aggregate CPU and memory limits, cgroups are the better fit. Also, `nice` changes scheduling priority rather than imposing a hard CPU limit. ([Linux manual page][2])

---

**3. Explain a TLS handshake.**

A TLS handshake authenticates the communicating endpoint and establishes keys for encrypted communication.

For a typical **full, certificate-based TLS 1.3 connection**:

1. **ClientHello:** The client sends supported TLS versions, cipher suites, and key-share information.
2. **ServerHello:** The server selects compatible parameters and returns its key share.
3. **Key derivation:** Both sides derive shared handshake keys.
4. **Server authentication:** The server sends its certificate chain, a signature proving possession of its private key, and a Finished message.
5. **Client verification:** The client validates the certificate chain, hostname, validity, signature, and handshake integrity.
6. **Client Finished:** The client confirms the handshake.
7. **Application traffic:** HTTP data travels using symmetric encryption.

The server does not transmit its private key. With ephemeral Diffie–Hellman, the shared secret is derived independently on both sides.

In **mutual TLS**, the server also authenticates the client using a client certificate. Resumed sessions can use a different handshake flow. ([rfc-editor.org][3])

For troubleshooting:

```bash
openssl s_client \
  -connect api.example.com:443 \
  -servername api.example.com \
  -verify_hostname api.example.com \
  -verify_return_error </dev/null
```

This helps investigate certificate-chain problems, hostname mismatches, expiration, and protocol negotiation. `-servername` supplies SNI so a server hosting multiple domains can select the appropriate certificate. ([OpenSSL Documentation][4])

---

**4. Apart from storing Terraform log files in S3, what other options are available?**

First distinguish **execution logs** from **Terraform state**.

For execution logs:

| Destination                           | Suitable use                                                            |
| ------------------------------------- | ----------------------------------------------------------------------- |
| CI/CD job logs and artifacts          | Associate output with a pipeline run, commit, and environment.          |
| HCP Terraform or Terraform Enterprise | View centrally managed plan and apply runs.                             |
| CloudWatch Logs                       | Centralize logs, search them, configure retention, and generate alerts. |
| Elasticsearch/Elastic Stack           | Search and correlate Terraform logs with other operational logs.        |
| Local files                           | Short-term troubleshooting on a developer machine or runner.            |

HCP Terraform provides run views, while CloudWatch Logs provides centralized log storage and analysis. ([HashiCorp Developer][5])

To create a Terraform debug log:

```bash
TF_LOG=DEBUG \
TF_LOG_PATH=./terraform-debug.log \
terraform plan -no-color
```

Terraform supports verbosity levels such as `ERROR`, `WARN`, `INFO`, `DEBUG`, and `TRACE`. `TF_LOG_PATH` selects the file, but logging must also be enabled through `TF_LOG`. ([HashiCorp Developer][6])

In a pipeline, I would retain normal execution output, enable detailed debugging when investigating a problem, and attach metadata such as environment, workspace, commit, and run ID. Debug logs can expose sensitive information, so access and retention should be controlled.

**If the interviewer meant state storage:** alternatives to S3 include HCP Terraform, Azure Blob Storage through the `azurerm` backend, and Google Cloud Storage through the `gcs` backend. A logging platform does not replace a Terraform state backend. ([HashiCorp Developer][7])

---

**5. What are the different Terraform provisioners?**

The main built-in provisioners are:

| Provisioner   | Function                                                                   |
| ------------- | -------------------------------------------------------------------------- |
| `local-exec`  | Executes a command where Terraform is running.                             |
| `remote-exec` | Executes commands on a remote machine through a configured connection.     |
| `file`        | Copies files or directories from the Terraform runner to a remote machine. |

Example:

```hcl
resource "terraform_data" "runner_info" {
  provisioner "local-exec" {
    command = "uname -a"
  }
}
```

This prints information about the Linux machine executing Terraform.

Provisioners normally run during resource creation. They do not automatically rerun on every apply. Some operations can be configured for destruction using `when = destroy`.

A failed creation-time provisioner normally fails the apply and marks its resource as tainted, potentially leading to replacement on a later apply.

Provisioners should be used sparingly because Terraform cannot fully model their script effects. Prefer provider-managed resources, machine images, cloud-init, or configuration-management tools when appropriate. ([HashiCorp Developer][8])

---

**6. What is the difference between local and remote provisioners in Terraform runners?**

The distinction is **where the command executes**.

| Aspect                     | `local-exec`                           | `remote-exec`                               |
| -------------------------- | -------------------------------------- | ------------------------------------------- |
| Execution location         | Terraform runner                       | Target machine                              |
| In Jenkins                 | Jenkins agent running Terraform        | Remote instance                             |
| In a hosted Terraform run  | Hosted execution environment           | Remote instance                             |
| Tools required             | Installed on the runner                | Installed on the target                     |
| SSH/WinRM connection block | Not required by the provisioner        | Required                                    |
| Typical use                | Invoke an existing runner-side utility | Execute a command inside a provisioned host |

For example:

```hcl
provisioner "local-exec" {
  command = "hostname"
}
```

This prints the **runner’s hostname**.

```hcl
provisioner "remote-exec" {
  inline = ["hostname"]
}
```

With an appropriate connection block, this prints the **target machine’s hostname**. ([HashiCorp Developer][9])

An important interview point: **“local” does not necessarily mean your laptop.** If Terraform runs on a CI agent, local execution happens on that agent.

For remote execution, verify connectivity, credentials, host identity, operating-system username, and target readiness. A private instance requires a reachable private network path or a suitable intermediary.

Files created by local execution on an ephemeral runner also need explicit artifact collection if they must survive the run.

---

**7. What is the difference between EC2 user data and a remote provisioner?**

Both can configure a machine, but their execution models differ.

| Aspect                        | EC2 user data                                                  | Terraform `remote-exec`                                      |
| ----------------------------- | -------------------------------------------------------------- | ------------------------------------------------------------ |
| Execution mechanism           | Instance initialization software, commonly cloud-init on Linux | Terraform connects and executes commands                     |
| Typical timing                | First boot by default                                          | Resource creation during Terraform apply                     |
| Runner-to-instance connection | Not required                                                   | Required                                                     |
| SSH credentials               | Not required for user-data execution                           | Usually required for Linux SSH execution                     |
| Auto Scaling                  | New instances can bootstrap from launch-template user data     | Does not automatically execute for each ASG-created instance |
| Completion visibility         | Instance creation does not prove initialization succeeded      | Terraform observes the remote command’s exit status          |

For Amazon Linux 2023, a simple user-data script could be:

```bash
#!/bin/bash
set -euo pipefail

dnf install -y nginx
systemctl enable --now nginx
```

The instance needs access to the package repositories.

To investigate initialization:

```bash
sudo cloud-init status --long
sudo less /var/log/cloud-init-output.log
```

User-data scripts normally run on the first boot, although this behavior can be configured. Updating user data later does not automatically guarantee that cloud-init reruns the initialization. ([Amazon Elastic Compute Cloud][10])

For a scalable fleet, I would generally use a prepared AMI plus minimal user data, and verify application readiness through health checks. Retrieve secrets using the instance’s role instead of embedding long-lived credentials in the script.

---

**8. What are A, AAAA, and CNAME records in DNS?**

“AAA” most likely means **AAAA**, which contains four A characters.

| Record | Maps                               | Example                             |
| ------ | ---------------------------------- | ----------------------------------- |
| A      | Hostname to IPv4 address           | `app.example.com → 192.0.2.10`      |
| AAAA   | Hostname to IPv6 address           | `app.example.com → 2001:db8::10`    |
| CNAME  | Alias hostname to another hostname | `www.example.com → app.example.com` |

Example zone entries:

```dns
app.example.com.  300 IN A     192.0.2.10
app.example.com.  300 IN AAAA  2001:db8::10
www.example.com.  300 IN CNAME app.example.com.
```

A CNAME contains a **name**, not an IP address. Resolution follows that name to obtain the necessary address records.

A hostname with a CNAME cannot also hold ordinary records such as A or MX records at that same name. Consequently, a standard CNAME cannot be placed at the zone apex, where SOA and NS records are required.

Route 53 **alias records** support certain apex use cases, but they are an AWS feature distinct from a standard CNAME. ([Amazon Route 53][11])

Useful checks:

```bash
dig app.example.com A
dig app.example.com AAAA
dig www.example.com CNAME
```

---

**9. How does DNS failover work if one IP is unreachable?**

Publishing multiple IP addresses alone does not provide reliable, health-aware failover. You need **health checks connected to the DNS routing policy**.

For Route 53 active-passive failover, configure two records with the same name and record type:

| Record            | Destination                  | Role      |
| ----------------- | ---------------------------- | --------- |
| `app.example.com` | Primary application endpoint | Primary   |
| `app.example.com` | Standby application endpoint | Secondary |

The process is:

1. Monitor a meaningful endpoint, such as an application health URL.
2. Associate health checks with the records, or use supported alias target-health evaluation.
3. Return the primary endpoint while it is healthy.
4. Return the secondary endpoint when the primary is unhealthy and the secondary is healthy.
5. Test both failover and recovery behavior. ([Amazon Route 53][12])

**Operational limitations:**

* Failover includes health-detection time and DNS-cache expiration.
* Existing connections do not move automatically.
* Clients may retain addresses or connection pools longer than expected.
* The standby needs sufficient capacity and appropriate data replication.

For non-alias records associated with health checks, AWS recommends a TTL of 60 seconds or less to improve responsiveness. This still does not guarantee a precise failover time. ([Amazon Route 53][13])

A useful edge case: with health checks on both records, **Route 53 returns the primary if both primary and secondary are unhealthy**. Do not assume it will return no answer. ([Amazon Route 53][14])

---

**10. How are logs segregated in ELK?**

Logs are separated using **metadata, processing rules, storage organization, and access permissions**.

The components have different responsibilities:

* **Collectors**, such as Elastic Agent or Filebeat, gather logs.
* **Logstash or ingest pipelines** parse and enrich events.
* **Elasticsearch** stores and indexes them.
* **Kibana** provides search, visualization, and dashboards.

First, add consistent fields:

```json
{
  "service": {
    "name": "orders",
    "environment": "prod"
  },
  "log": {
    "level": "error"
  },
  "message": "Database connection timed out"
}
```

Service and environment metadata make it possible to distinguish production orders logs from development logs or other applications. ([Elastic Common Schema (ECS)][15])

Modern Elastic deployments commonly use data streams with this naming structure:

```text
<type>-<dataset>-<namespace>
```

Examples:

```text
logs-orders.application-prod
logs-orders.application-dev
logs-payments.application-prod
```

Here, `logs` identifies the data type, the dataset describes the log source, and the namespace separates environments or another chosen grouping. ([Elastic Common Schema (ECS)][16])

Logstash can route events using conditions. For example, this filter assigns orders production logs to the corresponding data stream:

```ruby
filter {
  if [service][name] == "orders" and
     [service][environment] == "prod" {
    mutate {
      replace => {
        "[data_stream][type]"      => "logs"
        "[data_stream][dataset]"   => "orders.application"
        "[data_stream][namespace]" => "prod"
      }
    }
  }
}
```

The Elasticsearch output must also be configured for compatible data-stream handling and automatic routing. ([Logstash][17])

In Kibana, you can filter:

```text
service.name : "orders"
and service.environment : "prod"
and log.level : "error"
```

For production design, I would also:

* Separate datasets when retention, ownership, or access requirements differ.
* Enforce Elasticsearch permissions on the appropriate data streams.
* Redact sensitive fields before storage.
* Avoid creating a separate index for every Pod or request.
* Use fields and filters for distinctions that do not require separate storage.

A dashboard filter helps users find data; it does not itself enforce a security boundary.

---

**11. Apart from a password, how can you log in to EC2?**

For Linux EC2 instances, the main options are **SSH keys, EC2 Instance Connect, and Systems Manager Session Manager**.

**SSH key authentication**

The instance trusts a public key, and the client proves possession of the corresponding private key:

```bash
chmod 400 team-key.pem

ssh -i team-key.pem ec2-user@203.0.113.10
```

The username depends on the AMI—for example, `ec2-user` for Amazon Linux or commonly `ubuntu` for Ubuntu.

SSH requires network reachability, suitable security rules, and a running SSH server. ([Amazon Elastic Compute Cloud][18])

**EC2 Instance Connect**

Instance Connect uses IAM authorization to publish an SSH public key temporarily. The key remains available for connection establishment for **60 seconds**; this does not mean the established SSH session expires after 60 seconds. ([Amazon Elastic Compute Cloud][19])

For private instances, an **EC2 Instance Connect Endpoint** can provide connectivity without assigning the instance a public IP. SSH authentication and applicable network controls still matter. ([Amazon Elastic Compute Cloud][20])

**Systems Manager Session Manager**

Start an interactive session using:

```bash
aws ssm start-session \
  --target i-0123456789abcdef0
```

Typical prerequisites include:

* SSM Agent installed and running.
* Appropriate instance permissions.
* IAM permission for the operator to start the session.
* Outbound HTTPS connectivity to required Systems Manager endpoints, directly or through VPC endpoints.
* The Session Manager plugin for CLI access.

A normal Session Manager shell does not require an inbound SSH port, a public IP, or an SSH key pair. ([Amazon Elastic Compute Cloud][21])

For routine administration, Session Manager is often useful because access can be centrally authorized and native shell sessions can be logged. However, Session Manager does **not** capture session contents for SSH tunneling or port-forwarding sessions. ([AWS Systems Manager][22])

[1]: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/monitoring_and_managing_system_status_and_performance/assembly_using-systemd-to-manage-resources-used-by-applications_monitoring-and-managing-system-status-and-performance?utm_source=chatgpt.com "Chapter 35. Using systemd to manage resources used by applications | Monitoring and managing system status and performance | Red Hat Enterprise Linux | 9"
[2]: https://man7.org/linux/man-pages/man2/getrlimit.2.html?utm_source=chatgpt.com "getrlimit(2)"
[3]: https://www.rfc-editor.org/rfc/rfc8446.html?utm_source=chatgpt.com "www.rfc-editor.org"
[4]: https://docs.openssl.org/3.5/man1/openssl-s_client/?utm_source=chatgpt.com "openssl-s_client"
[5]: https://developer.hashicorp.com/terraform/cloud-docs/run?utm_source=chatgpt.com "Manage and view runs in HCP Terraform | Terraform"
[6]: https://developer.hashicorp.com/terraform/internals/debugging?utm_source=chatgpt.com "Enable logs to debug Terraform | Terraform"
[7]: https://developer.hashicorp.com/terraform/language/backend?utm_source=chatgpt.com "Backend block configuration overview | Terraform"
[8]: https://developer.hashicorp.com/terraform/language/resources/provisioners/syntax?utm_source=chatgpt.com "Perform post-apply operations using provisioners | Terraform"
[9]: https://developer.hashicorp.com/terraform/language/block/resource?utm_source=chatgpt.com "resource block reference | Terraform"
[10]: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html?utm_source=chatgpt.com "Run commands when you launch an EC2 instance with user data input"
[11]: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/ResourceRecordTypes.html?utm_source=chatgpt.com "Supported DNS record types"
[12]: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover-types.html?utm_source=chatgpt.com "Active-active and active-passive failover"
[13]: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-values-failover.html?utm_source=chatgpt.com "Values specific for failover records"
[14]: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/health-checks-how-route-53-chooses-records.html?utm_source=chatgpt.com "How Amazon Route 53 chooses records when health checking is configured"
[15]: https://www.elastic.co/docs/reference/ecs/ecs-service?utm_source=chatgpt.com "Service fields"
[16]: https://www.elastic.co/docs/reference/ecs/ecs-data_stream?utm_source=chatgpt.com "Data Stream fields"
[17]: https://www.elastic.co/docs/reference/logstash/event-dependent-configuration?utm_source=chatgpt.com "Accessing event data and fields"
[18]: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connect-linux-inst-ssh.html?utm_source=chatgpt.com "Connect to your Linux instance using an SSH client"
[19]: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-connect-methods.html?utm_source=chatgpt.com "Connect to a Linux instance using EC2 Instance Connect"
[20]: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connect-with-ec2-instance-connect-endpoint.html?utm_source=chatgpt.com "Connect to your instances using a private IP address and EC2 Instance Connect Endpoint"
[21]: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connect-with-systems-manager-session-manager.html?utm_source=chatgpt.com "Connect to your Amazon EC2 instance using Session Manager"
[22]: https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-logging.html?utm_source=chatgpt.com "Enabling and disabling session logging"
