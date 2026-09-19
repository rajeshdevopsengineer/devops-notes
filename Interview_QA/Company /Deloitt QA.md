Below are detailed answers to all 21 questions. For questions about your experience, use the example designs to explain systems you actually worked on.

**1. What types of nodes did you deploy in AWS?**

“Node” can mean an EC2 server, an ECS container instance, or an EKS worker node. Explain the platform first, then describe the instance choice and its purpose.

| Workload requirement         | Example EC2 family | Reason                                                 |
| ---------------------------- | ------------------ | ------------------------------------------------------ |
| General application services | M family           | Balanced CPU and memory                                |
| CPU-intensive processing     | C family           | More compute capacity relative to memory               |
| Memory-intensive workloads   | R family           | Higher memory capacity                                 |
| GPU workloads                | G or P families    | Hardware acceleration for graphics or machine learning |

Instance size also depends on network throughput, storage performance, architecture compatibility, and measured utilization. [AWS EC2 instance types](https://docs.aws.amazon.com/ec2/latest/instancetypes/instance-types.html)

Then explain the operating model:

* **On-Demand capacity:** Suitable for baseline workloads that should not depend on spare-capacity availability.
* **Spot capacity:** Suitable for workloads designed to tolerate interruptions.
* **Private subnets:** Common placement for application and worker nodes.
* **Multiple Availability Zones:** Reduce dependence on one AZ.
* **Auto Scaling:** Adjust capacity to workload demand.

For EKS, distinguish worker nodes from the AWS-managed control plane. A strong answer explains **why you chose the capacity**, how it scaled, and which operational responsibilities you owned.

---

**2. What is the difference between an Interface Endpoint and a Gateway Endpoint?**

Both provide private access to supported services, but their networking mechanisms differ.

| Aspect            | Interface endpoint                                              | Gateway endpoint                                          |
| ----------------- | --------------------------------------------------------------- | --------------------------------------------------------- |
| Mechanism         | AWS PrivateLink                                                 | Route-table integration                                   |
| Network resources | Creates endpoint network interfaces in selected subnets         | Does not create endpoint ENIs                             |
| Service support   | Many AWS services and supported endpoint services               | Amazon S3 and DynamoDB                                    |
| Traffic routing   | Requests reach endpoint IP addresses, often through private DNS | Service prefix-list routes direct traffic to the endpoint |
| Security groups   | Applied to endpoint ENIs                                        | No security group attached to the endpoint                |
| Endpoint policies | Available where supported                                       | Supported                                                 |
| Endpoint charges  | Generally hourly and data-processing charges                    | No additional endpoint charge                             |

Current AWS documentation lists **both gateway and interface endpoint support for S3 and DynamoDB**. [Gateway endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/gateway-endpoints.html), [Interface endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/create-interface-endpoint.html)

For example:

* EC2 instances accessing S3 within their VPC commonly use an S3 gateway endpoint.
* Private access from an on-premises network can use an interface endpoint, with the necessary connectivity and DNS configuration.

Gateway endpoints cannot be consumed through a Transit Gateway. [S3 endpoint considerations](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-s3.html)

**An endpoint supplies a network path; IAM and resource policies still determine access.**

---

**3. How did you set up ECS using EC2 instances?**

A complete answer should cover networking, host capacity, application deployment, permissions, and scaling.

1. **Prepare networking.** Create a VPC with application subnets across multiple AZs. Private EC2 hosts need outbound connectivity or the appropriate endpoints for image pulls, ECS communication, logging, and other dependencies.

2. **Create an ECS cluster.** This is the logical grouping for container capacity and workloads.

3. **Create a launch template.** Use an ECS-optimized AMI, an instance profile, security groups, storage configuration, and bootstrap settings.

For an ECS-optimized Amazon Linux host, the cluster registration setting can be supplied through user data:

```bash
#!/bin/bash
echo 'ECS_CLUSTER=production-cluster' >> /etc/ecs/ecs.config
```

The ECS agent uses this setting when registering the instance. [ECS instance bootstrapping](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/bootstrap_container_instance.html)

4. **Create an Auto Scaling group and capacity provider.** Associate the capacity provider with the cluster and configure managed scaling and draining as appropriate. [ECS Auto Scaling group capacity providers](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/asg-capacity-providers.html)

5. **Register a task definition.** Specify the container image, CPU, memory, ports, logging, networking mode, secrets, and IAM roles.

Keep the permissions separate:

| Role                        | Purpose                                                                                 |
| --------------------------- | --------------------------------------------------------------------------------------- |
| EC2 container instance role | Permissions used by the host’s ECS agent                                                |
| Task execution role         | Permissions for configured task startup features, such as retrieving referenced secrets |
| Task role                   | AWS permissions used by application code inside the container                           |

The exact execution-role requirements depend on the launch type and enabled features. [ECS instance role](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/instance_IAM_role.html), [Task execution role](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html)

6. **Create an ECS service and load balancer integration.** Configure desired task count, placement, health checks, and deployment behavior. For tasks using `awsvpc` networking, the ALB target group must use target type **`ip`**, including when those tasks run on EC2. [ECS ALB integration](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/alb.html)

7. **Configure both scaling layers.** Service Auto Scaling changes the number of tasks; capacity-provider scaling changes the EC2 capacity available to run them.

---

**4. Can’t we configure Route 53?**

Yes. In the ECS example, Route 53 provides a custom domain for the application.

For `app.example.com`:

1. Manage the domain’s DNS in a Route 53 hosted zone.
2. Create an alias record pointing to the ALB.
3. Configure an HTTPS listener and a certificate covering the application domain.
4. Let the ALB forward requests to healthy ECS tasks.

Route 53 answers the DNS query. The client then connects to the load balancer; application requests do not pass through Route 53.

An alias record avoids maintaining the load balancer’s changing IP addresses manually. It can also be used at the zone apex, such as `example.com`. [Route 53 routing to an ELB load balancer](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-to-elb-load-balancer.html)

For internal applications, a private hosted zone can provide names resolvable within the associated VPC environment.

---

**5. How do you configure a GoDaddy domain in Route 53?**

Domain registration and DNS hosting are separate. The domain can remain registered with GoDaddy while Route 53 hosts its DNS.

The migration process is:

1. **Inventory the existing DNS records.** Include website records, MX records, email verification records, SPF, DKIM, and DMARC.
2. **Create a public hosted zone** in Route 53 with the same domain name.
3. **Re-create the required records** and validate them against the Route 53 authoritative name servers.
4. **Plan the DNS transition.** Lower relevant TTLs in advance where possible and allow existing cached values to expire.
5. **Update the domain’s nameservers at GoDaddy** to the four nameservers assigned to the Route 53 hosted zone.
6. **Verify DNS and application behavior**, keeping the old DNS configuration available while caches expire.

Example checks:

```bash
dig NS example.com
dig A app.example.com
dig MX example.com
```

If DNSSEC is enabled, coordinate the existing DS record removal and subsequent Route 53 DNSSEC setup; stale delegation information can break validation.

**Transferring the registration to AWS is optional.** [AWS DNS migration procedure](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/migrate-dns-domain-in-use.html)

---

**6. What is the difference between AWS Config and AWS CloudTrail?**

| Aspect           | AWS Config                                                        | AWS CloudTrail                                            |
| ---------------- | ----------------------------------------------------------------- | --------------------------------------------------------- |
| Main question    | “How is this resource configured, and is it compliant?”           | “Who performed this action, when, and through which API?” |
| Main information | Resource configurations, relationships, and configuration history | User, role, and service activity                          |
| Typical use      | Detect configuration drift and evaluate rules                     | Investigate changes and audit API activity                |
| Example          | A security group permits SSH from everywhere                      | A particular role called `AuthorizeSecurityGroupIngress`  |

For a security group opened to `0.0.0.0/0`:

* **Config** can record the configuration change and flag a rule violation.
* **CloudTrail** can identify the principal and API request responsible.

Config rules can support remediation workflows, but recording a violation does not automatically prevent the change. [AWS Config](https://docs.aws.amazon.com/config/latest/developerguide/WhatIsConfig.html), [AWS CloudTrail](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-user-guide.html)

CloudTrail’s default Event history covers 90 days of management events. S3 object operations such as `GetObject` require appropriate **data-event logging**; they are not all captured by default. [CloudTrail data events](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/logging-data-events-with-cloudtrail.html)

---

**7. What node groups have you used in AWS EKS?**

Describe the grouping, its purpose, and the scheduling decisions. An illustrative design is:

| Node group        | Capacity                         | Purpose                                                        |
| ----------------- | -------------------------------- | -------------------------------------------------------------- |
| Baseline group    | On-Demand                        | Critical platform components and baseline application capacity |
| Application group | On-Demand                        | Services needing predictable capacity                          |
| Batch group       | Spot                             | Retryable jobs and interruption-tolerant processing            |
| Specialized group | Suitable memory or GPU instances | Workloads with specific hardware requirements                  |

Explain how you used:

* Labels and node affinity to select suitable nodes.
* Taints and tolerations to reserve specialized capacity.
* Multiple AZs for availability.
* Pod disruption budgets and controlled updates.
* Autoscaling to add or remove capacity.

An EKS managed node group uses **either On-Demand or Spot capacity**. Separate groups can provide both within one cluster. [EKS managed node groups](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html)

Avoid simply saying “we used managed nodes.” Explain what you configured and why that arrangement suited the workloads.

---

**8. What are the types of node groups in AWS EKS?**

For conventional EC2 node groups, the main distinction is:

| Type                         | Operational responsibility                                                                                                        |
| ---------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| **Managed node groups**      | EKS manages substantial parts of EC2 provisioning and node lifecycle; you configure capacity and initiate/manage relevant updates |
| **Self-managed node groups** | You manage the EC2 instances, bootstrap process, scaling arrangements, patching, and replacement workflow                         |

The broader EKS compute options also include:

* **AWS Fargate:** Runs selected Pods without customer-managed EC2 hosts; selection uses Fargate profiles.
* **EKS Auto Mode:** Automates more of the compute and supporting infrastructure lifecycle.
* **EKS Hybrid Nodes:** Connects supported customer-managed infrastructure outside AWS to an EKS control plane.

Fargate profiles and Auto Mode compute should not simply be described as conventional EC2 managed node groups. [EKS compute options](https://docs.aws.amazon.com/eks/latest/userguide/eks-compute.html)

Also distinguish **Pod scaling** from **node scaling**: increasing replicas does not guarantee sufficient compute capacity exists.

---

**9. If a user wants access to an S3 bucket, what is the process?**

First establish the required access: which bucket or prefix, which operations, and whether access is permanent or temporary.

A typical process is:

1. **Authenticate through an appropriate identity.** For employees, federation or IAM Identity Center commonly supplies temporary role credentials.
2. **Grant least-privilege permissions.**
3. **Check resource policies and other restrictions.**
4. **Provide a reachable S3 endpoint.**
5. **Test the intended operation and audit access where required.**

The permissions differ by operation:

| Requirement    | IAM action        | Resource                       |
| -------------- | ----------------- | ------------------------------ |
| List objects   | `s3:ListBucket`   | Bucket ARN                     |
| Read objects   | `s3:GetObject`    | Object ARN or permitted prefix |
| Upload objects | `s3:PutObject`    | Object ARN or permitted prefix |
| Delete objects | `s3:DeleteObject` | Object ARN or permitted prefix |

For example, reading reports can be restricted to:

```text
arn:aws:s3:::company-reports/reports/*
```

Listing can separately be restricted with an `s3:prefix` condition. Keep Block Public Access enabled for private data. SSE-KMS objects may additionally require KMS authorization. [S3 access management](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-management.html)

For **cross-account direct access**, configure the required identity permissions and bucket policy, or let the user assume an appropriately trusted role in the bucket’s account. [Cross-account S3 permissions](https://docs.aws.amazon.com/AmazonS3/latest/userguide/example-walkthroughs-managing-access-example2.html)

For **temporary access to a specific object**, a presigned URL may be appropriate. Its effective lifetime is also limited by the signing credentials. [S3 presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html)

---

**10. How does VPC Peering work?**

VPC Peering provides private connectivity between two VPCs. The VPCs can belong to different accounts and can be in different Regions. [VPC Peering overview](https://docs.aws.amazon.com/vpc/latest/peering/what-is-vpc-peering.html)

For VPC A `10.10.0.0/16` and VPC B `10.20.0.0/16`:

1. Create the peering request.
2. Accept it from the peer account when required.
3. Add a route in A for `10.20.0.0/16` targeting the peering connection.
4. Add the reverse route in B.
5. Allow the required traffic through security groups and network ACLs.
6. Configure DNS resolution behavior if needed.

Two limitations matter:

* **CIDRs must not overlap.**
* **Peering is not transitive.** If A peers with B and B peers with C, A does not automatically reach C through B.

A peered VPC also cannot simply use the other VPC’s NAT gateway, internet gateway, or S3 gateway endpoint as a shared exit. [VPC Peering behavior and limitations](https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-basics.html)

---

**11. How does Transit Gateway work, and how would you configure it?**

AWS Transit Gateway acts as a regional routing hub connecting VPCs and supported network attachments. It is useful when connectivity needs extend beyond a small number of VPC pairs.

A typical configuration is:

1. Create the Transit Gateway.
2. Share it through AWS Resource Access Manager if other accounts need to attach their VPCs.
3. Create VPC attachments using suitable subnets in the required AZs.
4. Associate attachments with the appropriate Transit Gateway route tables.
5. Configure route propagation or static routes.
6. Update the workload subnet route tables.
7. Verify return routes, security groups, network ACLs, and DNS.

Two concepts are essential:

* **Association:** Determines which Transit Gateway route table is consulted for traffic arriving from an attachment.
* **Propagation:** Adds an attachment’s reachable prefixes to selected Transit Gateway route tables.

Separate route tables can support different connectivity policies—for example, allowing production and development to reach shared services without providing general connectivity between them. [Transit Gateway route tables](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-route-tables.html)

Attaching a VPC alone does not complete the routing configuration.

---

**12. When VPCs connect to Transit Gateway, what changes are needed in the VPC route tables?**

Add routes for the remote networks, with the **Transit Gateway ID as the target**.

For this example:

* VPC A: `10.10.0.0/16`
* VPC B: `10.20.0.0/16`

The relevant subnet route tables need:

| Route table | Destination    | Target  |
| ----------- | -------------- | ------- |
| VPC A       | `10.10.0.0/16` | `local` |
| VPC A       | `10.20.0.0/16` | `tgw-…` |
| VPC B       | `10.20.0.0/16` | `local` |
| VPC B       | `10.10.0.0/16` | `tgw-…` |

The Transit Gateway route tables also need routes directing:

* `10.10.0.0/16` to attachment A.
* `10.20.0.0/16` to attachment B.

**VPC routes target the Transit Gateway; Transit Gateway routes target attachments.**

Update the route tables associated with the participating workload subnets. The attachment subnets must also have appropriate routes to destinations inside their VPC.

Do not assume enabling Transit Gateway route propagation automatically updates VPC subnet route tables. [VPC attachment routing requirements](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-vpc-attachments.html)

---

**13. Do you configure each Transit Gateway attachment with a CIDR range?**

An attachment is created using:

* Transit Gateway ID.
* VPC ID.
* Subnet IDs.

You select at most one attachment subnet per AZ. The attachment connects the VPC to the Transit Gateway; **CIDR ranges belong in the routing configuration**.

For example:

```bash
aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id tgw-0123456789abcdef0 \
  --vpc-id vpc-0123456789abcdef0 \
  --subnet-ids subnet-11111111111111111 subnet-22222222222222222
```

The example subnets must belong to the specified VPC and different AZs. [Create a Transit Gateway VPC attachment](https://docs.aws.amazon.com/cli/latest/reference/ec2/create-transit-gateway-vpc-attachment.html)

Selecting attachment subnets does not mean that only resources in those subnets may communicate. Other workload subnets need the correct routing and attachment availability in their AZ.

Overlapping VPC CIDRs remain a routing problem; Transit Gateway does not automatically translate overlapping addresses.

---

**14. What is Route 53?**

Amazon Route 53 is AWS’s managed DNS service. Its capabilities include domain registration, DNS hosting, and health checking. [Route 53 overview](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)

Its major components include:

* **Public hosted zones:** DNS records intended for public resolution.
* **Private hosted zones:** DNS records available through associated private AWS environments.
* **DNS records:** Such as A, AAAA, CNAME, MX, TXT, and aliases.
* **Routing policies:** Control which DNS answers Route 53 returns.
* **Health checks:** Can influence suitable routing configurations.

Examples of routing policies:

| Policy      | Example use                                               |
| ----------- | --------------------------------------------------------- |
| Simple      | One straightforward destination                           |
| Weighted    | Distribute DNS responses between deployments              |
| Latency     | Prefer a configured endpoint based on latency             |
| Failover    | Select a secondary endpoint when the primary is unhealthy |
| Geolocation | Choose answers according to the requester’s location      |

DNS routing is affected by caching. A weighted policy does not guarantee an exact percentage of individual HTTP requests, and a failover change does not instantly replace every cached DNS answer.

For an application behind an ALB, Route 53 supplies the DNS answer; the ALB distributes application requests.

---

**15. What are WAF and AAF?**

**AWS WAF** is a web application firewall that inspects HTTP and HTTPS requests reaching supported resources, including CloudFront, Application Load Balancers, and API Gateway REST APIs.

Typical protections include:

* SQL injection and cross-site scripting rules.
* IP allow/block rules.
* Request-rate controls.
* Rules inspecting paths, headers, and query parameters.
* Managed rule groups and bot-related controls.

You associate a **web ACL** with the protected resource. New rules can first run in **Count** mode to assess their effect before enforcing blocks. [AWS WAF documentation](https://docs.aws.amazon.com/waf/latest/developerguide/what-is-aws-waf.html)

**“AAF — Application Access Firewall” is ambiguous here.** It is not a standard AWS counterpart to WAF; establish which product or capability the interviewer means.

Related AWS services have different purposes:

| Service              | Purpose                                                      |
| -------------------- | ------------------------------------------------------------ |
| AWS WAF              | Filter web requests                                          |
| AWS Network Firewall | Inspect and filter routed VPC network traffic                |
| AWS Shield           | DDoS protection                                              |
| AWS Firewall Manager | Centrally manage supported security policies across accounts |

AWS Network Firewall supports stateful and stateless inspection; it requires traffic to be routed through its firewall endpoints. [AWS Network Firewall](https://docs.aws.amazon.com/network-firewall/latest/developerguide/what-is-aws-network-firewall.html)

---

**16. What are VPC Flow Logs, and how do you track IPs hitting a VPC?**

VPC Flow Logs record metadata about IP traffic associated with network interfaces. You can configure collection at the VPC, subnet, or network-interface level, with delivery to CloudWatch Logs, S3, or Amazon Data Firehose. [VPC Flow Logs](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html)

Useful fields include:

* Source and destination IP addresses.
* Source and destination ports.
* Protocol.
* Network interface ID.
* Packet and byte counts.
* Start and end times.
* `ACCEPT` or `REJECT`.

To investigate an IP:

1. Identify the affected resource and its interfaces.
2. Select the relevant time window.
3. Filter by source or destination IP.
4. Examine ports, actions, volume, and traffic direction.
5. Correlate with load balancer or application logs.

Flow Logs contain traffic metadata, not HTTP URLs or packet payloads. Delivery is best effort, so they should not be treated as a complete packet capture. Custom fields such as `pkt-srcaddr` can help distinguish packet-level addresses from intermediary addresses. [Flow log fields](https://docs.aws.amazon.com/vpc/latest/userguide/flow-log-records.html)

For applications behind proxies or load balancers, use the corresponding access logs to investigate the client request. An `ACCEPT` flow record does not prove that the application request succeeded.

---

**17. How do you filter a particular IP in a CloudWatch log group?**

Use **CloudWatch Logs Insights**:

1. Select the relevant log group.
2. Choose the investigation time range.
3. Run a query appropriate to the log format.

For VPC Flow Logs with automatically discovered fields:

```sql
fields @timestamp, interfaceId, srcAddr, dstAddr,
       srcPort, dstPort, action, bytes
| filter srcAddr = "198.51.100.25"
      or dstAddr = "198.51.100.25"
| sort @timestamp desc
| limit 100
```

This returns records where the address appears on either side of the flow.

To investigate rejected traffic originating from that address:

```sql
fields @timestamp, interfaceId, srcAddr, dstAddr, dstPort, action
| filter srcAddr = "198.51.100.25" and action = "REJECT"
| sort @timestamp desc
| limit 100
```

To summarize traffic volume:

```sql
filter srcAddr = "198.51.100.25"
| stats sum(bytes) as totalBytes, count(*) as flowRecords
  by dstAddr, dstPort, action
| sort totalBytes desc
```

Field names depend on the log format. Custom text logs may need a `parse` expression; JSON application logs might expose a field such as `clientIp` instead. Prefer exact field comparisons over matching an IP anywhere in a message. [CloudWatch Logs Insights examples](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax-examples.html)

---

**18. If the logs are stored in S3, how do you track a particular IP?**

Use **Amazon Athena** to query the log objects.

The process is:

1. Identify the S3 prefix and log format.
2. Create a table in the AWS Glue Data Catalog whose schema matches that format.
3. Configure partitions or partition projection.
4. Configure the Athena query-results location.
5. Query the relevant dates and IP fields.

For a VPC Flow Logs table following AWS’s example schema, including a `date` partition:

```sql
SELECT
    from_unixtime("start") AS start_time,
    interface_id,
    srcaddr,
    dstaddr,
    dstport,
    action,
    bytes
FROM vpc_flow_logs
WHERE "date" = DATE '2026-09-19'
  AND (
      srcaddr = '198.51.100.25'
      OR dstaddr = '198.51.100.25'
  )
ORDER BY start_time DESC
LIMIT 100;
```

The table’s column order and types must match the delivered logs. Filtering a registered date partition reduces the data scanned. [Athena VPC Flow Logs table and queries](https://docs.aws.amazon.com/athena/latest/ug/vpc-flow-logs-create-table-statement.html)

Use a different schema for ALB, CloudFront, WAF, or CloudTrail logs. The relevant field might represent a client IP, a request source address, or an intermediary address.

**S3 stores the logs; Athena provides the SQL query capability.**

---

**19. How do you back up AWS services?**

Start with business recovery requirements:

* **RPO:** How much recent data loss is acceptable?
* **RTO:** How quickly must the service be restored?
* **Retention:** How long must recovery points remain available?
* **Failure scope:** Must recovery survive an account compromise or Regional outage?

Then choose the appropriate mechanism:

| Resource   | Typical backup approach                                                                      |
| ---------- | -------------------------------------------------------------------------------------------- |
| EC2/EBS    | EBS snapshots or AWS Backup protection for supported EC2 instances                           |
| RDS/Aurora | Automated backups, point-in-time recovery, and snapshots as supported                        |
| DynamoDB   | Point-in-time recovery and on-demand backups                                                 |
| EFS        | AWS Backup                                                                                   |
| S3         | Versioning and suitable AWS Backup protection; replication can provide additional protection |
| EKS        | Supported cluster-resource and persistent-storage backups                                    |

Current AWS Backup support includes EKS, but backup, copy, and restore capabilities vary by resource type and Region. [AWS Backup supported resources](https://docs.aws.amazon.com/aws-backup/latest/devguide/whatisbackup.html), [Feature availability](https://docs.aws.amazon.com/aws-backup/latest/devguide/backup-feature-availability.html)

A practical implementation includes scheduled backup plans, resource assignments, retention rules, encryption, monitoring, and appropriate account or Regional copies.

For databases running on EC2, consider application consistency: a storage snapshot alone may not provide the recovery behavior the application requires.

**Regular restore testing is essential.** Restore into an isolated environment, validate the data, start the application, and measure recovery time. A completed backup job does not establish that the full business service can recover within its target.

---

**20. Can we create AWS backups using shell scripting?**

Yes. A shell script can call the AWS CLI to start backups and inspect their status.

For example, this script submits an AWS Backup job for an EBS volume and retrieves its initial status:

```bash
#!/usr/bin/env bash
set -euo pipefail

region="ap-south-1"
vault="production-backups"
resource_arn="arn:aws:ec2:ap-south-1:123456789012:volume/vol-0123456789abcdef0"
backup_role_arn="arn:aws:iam::123456789012:role/AWSBackupServiceRole"

# Supply a stable identifier for this logical backup request.
request_id="${1:?Usage: backup.sh unique-request-id}"

job_id="$(
  aws backup start-backup-job \
    --region "$region" \
    --backup-vault-name "$vault" \
    --resource-arn "$resource_arn" \
    --iam-role-arn "$backup_role_arn" \
    --idempotency-token "$request_id" \
    --query BackupJobId \
    --output text
)"

printf 'Submitted backup job: %s\n' "$job_id"

aws backup describe-backup-job \
  --region "$region" \
  --backup-job-id "$job_id" \
  --query '{State:State,RecoveryPoint:RecoveryPointArn,Message:StatusMessage}' \
  --output json
```

Replace the example values with existing resources. The runner needs the appropriate backup permissions and `iam:PassRole`; the backup service role needs access to the resource and relevant encryption keys.

The idempotency token helps prevent duplicate submissions when retrying the same logical request. [Start backup job](https://docs.aws.amazon.com/cli/latest/reference/backup/start-backup-job.html)

**Submitting a job is asynchronous.** The script above does not wait for completion. A production workflow should use EventBridge or bounded polling to track the job, handle failure states, and record the recovery point. Inspect status messages too: a completed job can have reported issues. [Describe backup job](https://docs.aws.amazon.com/cli/latest/reference/backup/describe-backup-job.html), [Backup events](https://docs.aws.amazon.com/aws-backup/latest/devguide/eventbridge.html)

For routine scheduled protection, AWS Backup plans provide scheduling and retention management without requiring a custom scheduler.

---

**21. Once the backup is created, where do you store the log files?**

Keep backup data, operational logs, and audit records distinct:

| Information              | Suitable location                                                                          |
| ------------------------ | ------------------------------------------------------------------------------------------ |
| Backup recovery data     | AWS Backup vault or the service’s native backup storage                                    |
| Script output and errors | CloudWatch Logs through the configured execution environment or log agent                  |
| Backup job status        | AWS Backup job records, with EventBridge events sent to configured monitoring destinations |
| API audit activity       | CloudTrail                                                                                 |
| Long-term log archive    | A protected S3 logging bucket                                                              |

AWS Backup integrates with CloudWatch for metrics and EventBridge for job events. Configure failure notifications and log delivery explicitly; do not assume every job automatically produces a detailed CloudWatch log stream. [AWS Backup monitoring integrations](https://docs.aws.amazon.com/aws-backup/latest/devguide/whatisbackup.html)

Useful log fields include:

* Resource ARN and backup job ID.
* Start time, completion time, and final state.
* Recovery-point ARN.
* Failure or warning messages.
* Copy-job results where applicable.
* Restore-test results.

For archived logs, use a predictable account/Region/date prefix, appropriate access controls, and a defined retention policy.

**An EBS snapshot is not an ordinary object in a customer-managed S3 bucket.** The recovery data stays in its managed backup storage; execution and audit logs can be stored separately in CloudWatch Logs and S3.
