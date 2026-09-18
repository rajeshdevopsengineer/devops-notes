# CMT SRE Interview Questions and Answers

> **Experience:** 4 to 5 years  
> **Role:** Site Reliability Engineer  
> The answers are framed for an intermediate SRE interview and emphasize reliability, troubleshooting, observability, automation, and measurable operational outcomes.

## 1. Which AWS services have you used?

Answer only with services you have genuinely used, then explain your responsibility and one practical use case.

### Sample answer

I have worked with the following AWS services:

- **Compute:** EC2, Auto Scaling, Lambda, ECS, Fargate, and EKS
- **Networking:** VPC, public and private subnets, route tables, Internet Gateway, NAT Gateway, security groups, NACLs, Route 53, ALB, NLB, VPC endpoints, Transit Gateway, and VPN
- **Storage:** S3, EBS, EFS, and AWS Backup
- **Databases and messaging:** RDS, Aurora, ElastiCache, SQS, and SNS
- **Security:** IAM, KMS, Secrets Manager, Systems Manager Parameter Store, WAF, GuardDuty, Security Hub, and ACM
- **Observability:** CloudWatch metrics, logs, alarms, dashboards, EventBridge, CloudTrail, and AWS Config
- **DevOps and IaC:** ECR, CodeBuild, CodePipeline, CloudFormation, and Terraform with AWS

I used these services to build private, Multi-AZ platforms, operate container workloads, automate infrastructure, centralize logs and alarms, manage secrets, and implement recovery controls. CloudWatch provides real-time monitoring for AWS resources and applications. citeturn6search53

## 2. Design a highly available and fault-tolerant system in AWS

A production web platform could use this design:

```text
Users
  |
Route 53 health-based DNS
  |
CloudFront + WAF
  |
Application Load Balancer across multiple AZs
  |
ECS/EKS/EC2 application replicas in private subnets
  |
ElastiCache        SQS        RDS/Aurora Multi-AZ
  |                  |                 |
cache/session     async work      transactional data
```

### Design decisions

1. Use at least two Availability Zones with independent public, application, and database subnets.
2. Place only the public load balancer in public subnets. Keep workloads and databases private.
3. Run multiple stateless application replicas across AZs using Auto Scaling.
4. Use health checks, readiness checks, graceful termination, and deployment rollback.
5. Use RDS or Aurora Multi-AZ, automated backups, point-in-time recovery, and tested restores.
6. Externalize sessions to Redis and use SQS to decouple asynchronous processing.
7. Store objects in versioned, encrypted S3 and use lifecycle rules.
8. Use Route 53 or Global Accelerator for regional failover when the business requires multi-Region DR.
9. Apply least-privilege IAM, KMS encryption, WAF, Secrets Manager, security-group segmentation, CloudTrail, and Config.
10. Define SLOs, monitor golden signals, conduct load and failover tests, and maintain runbooks.

High availability minimizes interruption through redundancy and automatic recovery. Fault tolerance goes further by continuing operation despite component failure, often at greater cost and complexity. EKS reliability guidance emphasizes detection, self-healing, scaling, and resilient control and data planes. citeturn6search55

## 3. What is DNS?

DNS, or Domain Name System, translates names such as `api.example.com` into IP addresses. It is hierarchical and distributed.

Typical lookup path:

```text
Application -> OS cache -> Recursive resolver -> Root
            -> TLD server -> Authoritative server -> Answer
```

Common records:

- `A`: Name to IPv4 address
- `AAAA`: Name to IPv6 address
- `CNAME`: Alias to another name
- `MX`: Mail server
- `TXT`: Verification and policy text
- `NS`: Authoritative name servers
- `PTR`: Reverse lookup
- `SRV`: Service location

TTL controls how long resolvers cache a record. For troubleshooting, use:

```bash
dig api.example.com
dig +trace api.example.com
nslookup api.example.com
resolvectl status
cat /etc/resolv.conf
```

DNS failover is not instantaneous because recursive resolvers and clients may retain answers until TTL expiry.

## 4. What are TCP and UDP?

### TCP

TCP is connection-oriented and provides ordered, reliable byte delivery using a handshake, acknowledgements, retransmission, flow control, and congestion control.

Use cases include HTTPS, SSH, database connections, and most transactional APIs.

### UDP

UDP is connectionless and sends independent datagrams without guaranteeing delivery, ordering, or duplicate prevention. It has lower protocol overhead and allows the application to control recovery behavior.

Use cases include DNS queries, real-time voice/video, telemetry, and protocols such as QUIC, which add reliability and security above UDP.

Troubleshooting commands:

```bash
ss -lntp       # Listening TCP sockets
ss -lnup       # Listening UDP sockets
ss -s          # Socket summary
nc -vz host 443
curl -v --connect-timeout 5 https://host/
tcpdump -nn -i any host 10.0.1.20
```

## 5. What are IPv4 and IPv6?

- **IPv4** uses 32-bit addresses, such as `10.0.1.25`. Private ranges and NAT are widely used because the address space is limited.
- **IPv6** uses 128-bit addresses, such as `2001:db8::10`, providing a much larger address space. IPv6 does not use broadcast and relies on mechanisms such as Neighbor Discovery and multicast.

Important differences include address size, notation, header design, fragmentation behavior, and address configuration. IPv6 does not mean traffic is automatically secure. Security groups, routing, firewalls, observability, and patching are still required.

Useful commands:

```bash
ip -4 address
ip -6 address
ip route
ip -6 route
ping -4 example.com
ping -6 example.com
```

Kubernetes can be IPv4-only, IPv6-only, or dual-stack, and Pod, Service, and node addressing components must agree on the configured IP families. citeturn6search61

## 6. What is a PID?

A PID is a Process Identifier assigned by the operating system to a running process. PID 1 is the userspace process started first and is responsible for important process-supervision and orphan-reaping behavior.

Related concepts:

- PPID: Parent process ID
- TID: Thread ID
- PGID: Process group ID
- SID: Session ID

```bash
ps -ef
ps -p 1234 -o pid,ppid,user,stat,lstart,etime,cmd
pstree -p
cat /proc/1234/status
lsof -p 1234
kill -TERM 1234
```

Use `SIGTERM` for graceful termination before considering `SIGKILL`. The `ps` utility reports active process information and can display PID, state, CPU time, command, and other fields. citeturn6search71turn6search72

## 7. How do you troubleshoot a Linux system that is down?

First determine whether the host is unreachable, the OS is unhealthy, or only the application is unavailable.

### From another host

```bash
ping -c 4 HOST
traceroute HOST
nc -vz HOST 22
curl -vk --connect-timeout 5 https://HOST/health
nslookup HOST
```

### After obtaining console or Systems Manager access

```bash
uptime
who -b
last -x | head
systemctl --failed
systemctl status application.service
journalctl -b -p warning
journalctl -u application.service --since "30 minutes ago"
dmesg -T | tail -100
```

`journalctl` reads structured records from the systemd journal and supports filtering by service and other fields. citeturn6search68turn6search70

### Resource checks

```bash
top
ps aux --sort=-%cpu | head
ps aux --sort=-%mem | head
free -m
vmstat 1 5
iostat -xz 1 5
df -hT
df -i
findmnt
lsof +L1
```

### Network checks

```bash
ip address
ip route
ss -lntup
ss -s
resolvectl status
curl -v http://127.0.0.1:PORT/health
sudo nft list ruleset
sudo tcpdump -nn -i any port PORT
```

### Kernel and hardware checks

```bash
dmesg -T | grep -Ei 'oom|error|fail|timeout|reset'
journalctl -k -b
cat /proc/loadavg
cat /proc/meminfo
```

Check recent deployments, configuration changes, certificates, dependencies, disk/inodes, OOM kills, file descriptors, connection limits, and load-balancer health. Mitigate first through rollback, failover, or replacement, then complete root-cause analysis. Avoid immediately rebooting because it may destroy evidence.

## 8. Production incidents attended

Use a real incident in STAR format. Do not invent experience.

### Sample incident: latency caused by database connection exhaustion

- **Situation:** API p99 latency and 5xx errors increased during peak traffic.
- **Task:** Restore service within the SLO and identify the cause.
- **Action:** I checked ALB latency and target errors, compared application traces, and found threads waiting for database connections. A recent scaling change increased Pod count, but each Pod retained a large connection pool, exhausting the database connection limit. We reduced pool size, temporarily scaled database capacity, restarted affected replicas gradually, and verified recovery. We then added pool-utilization and database-connection alarms, set a connection budget per replica, load-tested the correction, and updated the runbook.
- **Result:** Error rate returned to normal, p99 latency recovered, and the same failure mode was prevented by capacity validation and alerting.

Other credible examples include full disk, expired certificate, bad deployment, SQS backlog, DNS failure, node pressure, memory leak, or an incorrect security rule. Include detection, impact, timeline, mitigation, root cause, prevention, and measurable result.

## 9. Kubernetes architecture

A Kubernetes cluster contains a control plane and worker nodes. citeturn6search56turn6search57

### Control plane

- **kube-apiserver:** Exposes the Kubernetes API and is the control-plane front end.
- **etcd:** Consistent key-value store for cluster state.
- **kube-scheduler:** Assigns unscheduled Pods to suitable nodes.
- **kube-controller-manager:** Runs reconciliation controllers.
- **cloud-controller-manager:** Integrates with cloud-provider APIs where applicable.

### Worker node

- **kubelet:** Ensures assigned Pods and containers are running.
- **Container runtime:** Runs containers, commonly through a CRI-compatible runtime.
- **kube-proxy or equivalent dataplane:** Implements Service connectivity where used.
- **CNI plugin:** Implements Pod networking.
- **CSI driver:** Integrates persistent storage.

```text
kubectl/client -> API server -> etcd
                         |-> scheduler
                         |-> controllers
                              |
                         kubelet on nodes
                              |
                      runtime -> Pods
```

Controllers continuously reconcile actual state toward desired state. Production clusters use multiple nodes, failure-domain spreading, resource requests, disruption budgets, autoscaling, probes, persistent storage, and observability.

## 10. Terraform structure

A maintainable repository separates reusable modules from environment-specific root modules:

```text
terraform/
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── versions.tf
│   └── eks/
└── environments/
    ├── dev/
    │   ├── backend.tf
    │   ├── main.tf
    │   ├── providers.tf
    │   ├── variables.tf
    │   ├── outputs.tf
    │   └── dev.tfvars
    └── prod/
```

- `terraform` block: Required versions and providers
- `provider` block: Provider configuration
- `resource`: Managed infrastructure objects
- `data`: Read existing information
- `variable`: Module inputs
- `locals`: Internal calculated values
- `output`: Exposed values
- `module`: Reusable child module call
- `backend`: Remote state configuration

Terraform evaluates `.tf` files in one directory as one module, irrespective of filenames. `main.tf`, `variables.tf`, and `outputs.tf` are recommended conventions for a minimal reusable module. citeturn6search62turn6search63

Use remote encrypted state with locking, pin versions, commit `.terraform.lock.hcl`, keep secrets out of state where possible, use CI for `fmt`, `validate`, lint, security scan, plan, approval, and apply, and avoid one oversized state for unrelated systems. Modules improve reuse and standardization. citeturn6search64turn6search65

## 11. Maintaining high availability in ECS/Fargate or EKS

### ECS with Fargate

- Run an ECS service with multiple tasks in at least two AZs.
- Use ALB health checks and ECS deployment circuit breaker with rollback.
- Configure target-tracking or step scaling for CPU, memory, request count, or queue backlog.
- Use appropriate minimum healthy percent and maximum percent during deployment.
- Define CPU/memory correctly and use Fargate On-Demand for the critical baseline.
- Use CloudWatch Container Insights, logs, alarms, and service-quota alerts.
- Externalize sessions and state.

Fargate services expose CPU and memory utilization, while CloudWatch usage metrics can alert as vCPU usage approaches quotas. citeturn6search51turn6search52

### EKS

- Spread replicas across AZs and nodes using topology constraints and anti-affinity.
- Use readiness, liveness, and startup probes.
- Set resource requests/limits and multiple replicas.
- Configure Pod Disruption Budgets and graceful termination.
- Use HPA for Pods and Karpenter, Cluster Autoscaler, or EKS Auto Mode for compute.
- Maintain multiple node groups and sufficient spare capacity.
- Perform controlled updates and test disruption behavior.
- Use managed persistent services where possible and design storage by failure domain.

EKS runs its managed control plane across three AZs, while customers share responsibility for data-plane reliability. citeturn6search55

## 12. ALB versus NLB

### Application Load Balancer

- Layer 7, primarily HTTP and HTTPS
- Host, path, header, method, query, and source-IP-aware routing
- Redirects, fixed responses, WebSockets, authentication integrations, and WAF support
- Best for web applications, REST APIs, microservices, and Kubernetes Ingress

### Network Load Balancer

- Layer 4, supporting TCP, UDP, and TLS listeners
- Very high-throughput, low-latency connection handling
- Preserves source IP in supported target configurations
- Supports static IP behavior and Elastic IPs for internet-facing mappings
- Best for non-HTTP protocols, very high connection rates, allow-listed fixed addresses, and pass-through-style network workloads

Use ALB when routing requires application awareness. Use NLB when the workload needs transport-level balancing, UDP, static entry addresses, or extreme connection performance. Monitor target health for both; ALB metrics include HTTP target responses and latency, while NLB metrics focus on flows, resets, TLS, throughput, and target health. citeturn6search50turn6search54

## 13. Web and application server HA metrics

Monitor service outcomes instead of relying only on CPU:

- Availability and successful request percentage
- Healthy and unhealthy target count
- Request rate and concurrent connections
- 4xx and 5xx rates, especially target-generated 5xx
- Latency percentiles: p50, p95, p99
- Timeouts, resets, rejected connections, and TLS errors
- CPU, memory, load, garbage collection, threads, file descriptors, and sockets
- Queue depth, thread-pool saturation, and connection-pool usage
- Pod/task desired versus running count and restart rate
- Dependency latency and error rate
- Synthetic health-check success

For ALB, CloudWatch publishes target health and load-balancer metrics at 60-second intervals while requests are flowing. citeturn6search50

Define an availability SLI such as:

```text
successful valid requests / total valid requests
```

Alert based on SLO burn rate and customer impact, then use infrastructure metrics for diagnosis.

## 14. Daily-work automation

### Sample answer

I automated repetitive, error-prone, and auditable activities, including:

- Terraform provisioning and drift detection
- Scheduled patching through Systems Manager
- Jenkins/GitHub CI/CD flows
- AMI and container-image builds
- Kubernetes deployment, health validation, and rollback
- Log-retention and backup verification
- CloudWatch dashboard and alarm provisioning
- Expired-certificate and unused-resource reporting
- Incident diagnostics and ChatOps runbooks
- Access-review and tagging-compliance reports

A useful example is an automated daily Terraform drift pipeline that executes `terraform plan -detailed-exitcode`, stores the report, and notifies the owner only when drift is present. I make automation idempotent, least privileged, observable, tested in non-production, and protected by approvals for destructive actions. I measure time saved, error reduction, recovery time, and failure rate.

## 15. Metrics for EC2 CPU and memory

### Default EC2 metrics

- `CPUUtilization`
- `NetworkIn` and `NetworkOut`
- `NetworkPacketsIn` and `NetworkPacketsOut`
- Disk read/write operations and bytes for relevant instance-volume behavior
- `StatusCheckFailed`, `StatusCheckFailed_Instance`, and `StatusCheckFailed_System`

### Memory and filesystem metrics

EC2 does not publish guest operating-system memory and filesystem utilization as standard EC2 metrics. Install and configure the CloudWatch agent or another observability agent to collect metrics such as:

- `mem_used_percent`
- `swap_used_percent`
- `disk_used_percent`
- `disk_inodes_free`
- Process, socket, and application metrics as required

Create alarms using baselines and workload behavior. High CPU can be healthy under load; combine it with latency, queueing, errors, throttling, and saturation. CloudWatch is AWS's managed real-time monitoring platform. citeturn6search53

## 16. Handling slowness in a decoupled SQS system

Determine where the waiting time occurs:

```text
producer -> SQS backlog -> consumer receive -> processing -> dependency
```

Monitor:

- `ApproximateNumberOfMessagesVisible`
- `ApproximateAgeOfOldestMessage`
- Messages sent, received, and deleted
- Not-visible/in-flight messages
- Empty receives
- DLQ message count
- Consumer count, CPU/memory, processing latency, errors, and dependency latency

Troubleshooting and remediation:

1. Verify whether producers created an abnormal traffic spike.
2. Check whether consumers are running, polling with long polling, and deleting successfully.
3. Scale consumers using backlog per worker and message age, not only CPU.
4. Ensure visibility timeout exceeds normal processing time and extend it for long tasks.
5. Make consumer processing idempotent because delivery may repeat.
6. Use batching and long polling to improve efficiency.
7. Check downstream database/API throttling before adding consumers.
8. Confirm in-flight limits, Lambda concurrency, ECS capacity, Fargate quotas, and API quotas.
9. Configure DLQ and redrive policy; inspect poison messages instead of endlessly retrying them.
10. For FIFO queues, inspect message-group imbalance because one blocked group can serialize work.
11. Apply backpressure, producer rate limits, and load shedding if downstream systems are saturated.

Estimate required concurrency:

```text
required consumers ≈ arrival rate × average processing time
```

Add safety margin and validate through load testing. Increasing consumers without checking the downstream capacity can move the outage to the database or external API.

## 17. Best practices for highly available systems

1. Define business SLO, RTO, RPO, consistency, and data-loss requirements first.
2. Remove single points of failure across compute, network, storage, identity, DNS, and deployment systems.
3. Distribute workloads across multiple AZs and use multiple Regions only when justified.
4. Use stateless replicas and externalize durable state and sessions.
5. Use health checks, self-healing, graceful shutdown, retries with backoff and jitter, timeouts, circuit breakers, and idempotency.
6. Scale from demand and saturation signals, with tested quotas and spare capacity.
7. Use durable queues to decouple workloads and enforce backpressure.
8. Use managed Multi-AZ databases, backups, point-in-time recovery, and regular restore testing.
9. Apply infrastructure as code, immutable artifacts, progressive delivery, and automated rollback.
10. Monitor metrics, logs, traces, events, synthetic checks, and SLO burn rates.
11. Apply least privilege, secret rotation, patching, encryption, segmentation, and continuous security controls.
12. Test failures with game days, load tests, recovery exercises, and region/AZ evacuation runbooks.
13. Manage dependencies with explicit timeouts, fallback behavior, and capacity contracts.
14. Review incidents blamelessly and track corrective actions to completion.
15. Balance availability against cost and complexity. More components do not automatically create a more reliable system.

A reliable system must detect failures, recover automatically, and scale as demand changes. citeturn6search55

---

## Quick interview revision

- Explain AWS services through real use cases, not a memorized list.
- HA uses redundancy and recovery; fault tolerance aims to continue through failure.
- DNS resolves names, TCP provides reliable streams, and UDP provides datagrams.
- A PID identifies a process.
- Troubleshoot Linux from reachability through logs, resources, network, kernel, and recent changes.
- Explain incidents using impact, diagnosis, mitigation, root cause, prevention, and result.
- Kubernetes consists of a control plane and worker nodes.
- Terraform should use remote state, small root modules, reusable modules, version pinning, and CI controls.
- For ECS/EKS availability, spread replicas, use health checks, autoscaling, disruption controls, and external state.
- ALB is application-aware Layer 7; NLB is transport-oriented Layer 4.
- Availability monitoring must include success rate, latency, errors, and saturation.
- EC2 provides CPU by default; guest memory needs an agent.
- For SQS slowness, monitor backlog age and consumer throughput, then protect downstream dependencies.
