Below are senior-level, interview-ready answers. I’ve framed them in a practical DevOps/SRE style, with enough depth to handle follow-up questions.

### 1) How do you ensure high availability in Kubernetes?

I look at HA across several layers: control plane, worker nodes, application replicas, networking, and data.

For a production cluster, I would use a highly available control plane, distribute worker nodes across multiple Availability Zones, run multiple pod replicas, use PodDisruptionBudgets, topology spread constraints or anti-affinity, and configure readiness/liveness probes.

For example:

```text
                 Load Balancer
                      |
          +-----------+-----------+
          |                       |
       AZ-1                     AZ-2
          |                       |
      Worker Node             Worker Node
          |                       |
       App Pod                 App Pod
          \                       /
           +------ Service ------+
```

For applications:

```yaml
replicas: 3
```

I also use:

* Horizontal Pod Autoscaler
* Cluster Autoscaler or Karpenter
* PodDisruptionBudget
* Multi-AZ node groups
* Readiness and liveness probes
* Rolling or canary deployments
* Multiple replicas of critical services
* Highly available databases
* Backup and disaster recovery
* Monitoring and alerting

For stateful workloads, application HA alone is not enough. The database and persistent storage must also be redundant.

---

### 2) What are SLO, SLI, and SLA, and why are they important?

**SLI — Service Level Indicator**

This is an actual measurement of system behavior.

Examples:

```text
Availability = 99.95%
P95 latency = 250 ms
Error rate = 0.2%
```

**SLO — Service Level Objective**

This is the reliability target.

Example:

```text
99.9% availability per month
```

**SLA — Service Level Agreement**

This is a contractual commitment to the customer.

For example:

```text
Customer SLA = 99.9% availability
```

If the SLA is violated, there may be service credits or other business consequences.

The relationship is:

```text
SLI = What we measure
SLO = What we want to achieve
SLA = What we promise externally
```

They are important because SRE should not define reliability as "the system should never fail." We need measurable objectives.

They also allow us to calculate an **error budget**:

```text
SLO = 99.9%

Error budget = 0.1%
```

If we are consuming the error budget too quickly, we may slow feature releases and focus on reliability improvements.

---

### 3) Describe a recent major incident you handled and how you resolved it.

A strong example:

We had a production incident where users started seeing increased HTTP 500 errors shortly after a deployment.

Our monitoring showed:

```text
5xx errors ↑
Latency ↑
Pod restarts ↑
```

I first checked application health and Kubernetes:

```bash
kubectl get pods -n production
kubectl describe pod <pod> -n production
kubectl logs <pod> -n production --previous
```

Several pods were restarting and the container termination reason was:

```text
OOMKilled
Exit Code: 137
```

Grafana metrics showed memory usage increasing rapidly after the new application version was deployed.

Since customer impact was ongoing, our priority was restoration rather than debugging the new code in production.

We rolled back:

```bash
kubectl rollout undo deployment/payment-service
```

Once the previous version was restored:

```text
5xx errors -> normal
Latency -> normal
Restart count -> stable
```

The root cause was a memory regression introduced by the latest application release.

After the incident we implemented:

* Memory alerts
* Better resource requests and limits
* Load testing in CI/CD
* Canary deployments
* Deployment health gates
* Improved dashboards
* RCA/postmortem documentation

My approach in a major incident is always:

```text
Detect
  ↓
Assess customer impact
  ↓
Mitigate
  ↓
Restore service
  ↓
Find root cause
  ↓
Permanent fix
  ↓
Prevent recurrence
```

---

### 4) What is the deployment setup in your organization?

A good production example would be a GitOps-based Kubernetes pipeline:

```text
Developer
   |
   v
GitHub / GitLab
   |
   v
Pull Request
   |
   v
CI Pipeline
   |
   +-- Unit tests
   +-- Code quality
   +-- SAST scan
   +-- Dependency scan
   +-- Docker build
   +-- Container scan
   |
   v
Container Registry
   |
   v
Helm / Kustomize
   |
   v
Argo CD
   |
   v
EKS / Kubernetes
   |
   +-- Dev
   +-- QA
   +-- Stage
   +-- Production
```

For production, I prefer:

```text
Rolling deployment
or
Canary deployment
```

with automated health checks.

We validate:

```text
Error rate
Latency
CPU/memory
Pod health
Business transaction success
```

If those metrics degrade, the deployment can be automatically or manually rolled back.

Infrastructure itself is managed using Terraform and application configuration through Helm or Kustomize.

---

### 5) What is the maximum time taken for a node to start after failure or restart?

There is no single fixed Kubernetes maximum.

It depends on:

* Cloud provider
* Node provisioning mechanism
* VM/image boot time
* Bootstrap scripts
* Container image download
* Cluster Autoscaler/Karpenter
* Network initialization
* CNI initialization

For an existing EC2 node reboot, it may become usable within a few minutes.

For replacement:

```text
Node failure detected
       |
       v
Node becomes NotReady
       |
       v
Pods rescheduled
       |
       v
Autoscaler/Karpenter creates replacement
       |
       v
EC2 boots
       |
       v
Node joins cluster
       |
       v
Pods scheduled
```

In a well-designed EKS environment, I would expect the replacement path to be on the order of minutes, not seconds.

But the more important senior-level answer is:

> We should not depend on waiting for a failed node to recover. Applications should already have replicas running across multiple nodes and Availability Zones.

If one node fails:

```text
Node 1             Node 2
Pod A               Pod A
Pod B        -->    Pod B replica already running
```

Customer traffic should therefore continue while Kubernetes replaces capacity in the background.

---

### 6) How have you resolved high-performance issues or critical incidents?

I follow a metric-driven approach.

First, I determine whether the bottleneck is:

```text
CPU
Memory
Disk I/O
Network
Database
Application
External dependency
```

For Kubernetes:

```bash
kubectl top pods
kubectl top nodes
kubectl describe pod <pod>
```

At Linux level:

```bash
top
htop
vmstat 1
iostat -xz 1
sar
free -m
df -h
ss -tulpn
```

For application performance, I correlate:

```text
Metrics
+
Logs
+
Distributed traces
```

For example:

```text
API request = 5 seconds
      |
      +-- API Gateway = 20 ms
      |
      +-- Backend = 100 ms
      |
      +-- Database = 4.8 seconds
```

Now I know the bottleneck is likely database-related rather than Kubernetes.

Typical fixes I've used include:

* Scaling pods horizontally
* Increasing worker capacity
* Optimizing SQL queries
* Creating database indexes
* Connection pooling
* Caching
* Raising appropriate CPU/memory resources
* Fixing memory leaks
* CDN caching
* Rate limiting
* Fixing external dependency timeouts

I avoid immediately scaling infrastructure until I've identified the actual bottleneck.

---

### 7) What is the difference between observability and monitoring?

Monitoring answers:

> "Is something wrong?"

Observability helps answer:

> "Why is it wrong?"

Monitoring usually relies on predefined metrics and alerts.

For example:

```text
CPU > 90%
5xx > 5%
Pod unavailable
Disk > 85%
```

Observability includes:

```text
Metrics
Logs
Traces
Events
Profiles
```

Example:

Monitoring tells me:

```text
Checkout latency increased to 5 seconds.
```

Observability lets me trace:

```text
Frontend
  |
  v
Checkout API
  |
  v
Payment Service
  |
  v
Database
          <-- slow SQL query
```

Monitoring is therefore a subset of a broader observability strategy.

---

### 8) What Linux command is used for mounting a file system?

The command is:

```bash
mount
```

Example:

```bash
mount /dev/xvdf1 /data
```

To check mounted filesystems:

```bash
mount
```

or:

```bash
df -h
```

For persistent mounting after reboot, configure `/etc/fstab`.

Example:

```text
/dev/xvdf1 /data ext4 defaults,nofail 0 2
```

Then test it:

```bash
mount -a
```

A better production practice is to use UUID instead of device names:

```bash
blkid
```

Then `/etc/fstab`:

```text
UUID=<uuid> /data ext4 defaults,nofail 0 2
```

---

### 9) How do you detect the root cause when an application goes down in the cloud?

I start from customer impact and work down the stack.

```text
User
 |
DNS
 |
CDN/WAF
 |
Load Balancer
 |
Ingress
 |
Service
 |
Pod
 |
Application
 |
Database
 |
External services
```

I first determine:

```text
Is DNS resolving?
Is the LB healthy?
Are targets healthy?
Are pods running?
Is the application responding?
Is the database reachable?
Did anything change recently?
```

For Kubernetes:

```bash
kubectl get pods
kubectl get events --sort-by=.lastTimestamp
kubectl describe pod <pod>
kubectl logs <pod>
kubectl logs <pod> --previous
```

Then I check observability:

```text
Metrics
Logs
Traces
Deployment history
Cloud events
```

I always check recent changes early:

```text
Deployment?
Configuration?
Secret rotation?
Infrastructure change?
Database migration?
Certificate expiry?
DNS change?
```

A very useful operational principle is:

> "What changed immediately before the outage?"

---

### 10a) What happens if the master node goes down?

In modern terminology, it is the **control plane** rather than master node.

In a production Kubernetes cluster, the control plane should be highly available.

For self-managed Kubernetes:

```text
Control Plane 1
Control Plane 2
Control Plane 3
```

If one fails, the remaining control-plane nodes continue operating, assuming etcd still has quorum.

The existing workloads on worker nodes generally continue running even if the Kubernetes API temporarily becomes unavailable.

What may be affected:

```text
New deployments
Scheduling new pods
Scaling
kubectl/API operations
Controller reconciliation
```

If using **Amazon EKS**, AWS manages the Kubernetes control plane and its high availability, so I don't manually recover an individual control-plane node.

---

### 10b) What happens if a worker node goes down?

When a worker becomes unavailable:

```text
Worker Node
    X
    |
 Pods become unavailable
    |
 Kubernetes detects failure
    |
 Pods recreated on healthy nodes
```

I would check:

```bash
kubectl get nodes
kubectl describe node <node>
kubectl get pods -A -o wide
```

If the cluster doesn't have enough capacity, Cluster Autoscaler or Karpenter can provision additional nodes.

A highly available application should already have pods distributed:

```text
AZ-1                AZ-2
Node A              Node B
App Pod 1           App Pod 2
```

So loss of one worker should not cause a user-visible outage.

---

### 11) How do you configure a VPC for high availability?

I use at least two Availability Zones for production.

Example:

```text
                    VPC
                     |
        +------------+------------+
        |                         |
       AZ-A                      AZ-B
        |                         |
 Public Subnet               Public Subnet
        |                         |
      NAT-A                     NAT-B
        |                         |
 Private App               Private App
 Subnet                    Subnet
        |                         |
      EKS/EC2                   EKS/EC2
        |                         |
 Private DB                Private DB
 Subnet                    Subnet
         \                       /
          +------ Multi-AZ -----+
                   RDS
```

Important considerations:

* Multiple AZs
* Separate public/private subnets
* Load balancers across AZs
* NAT Gateway per AZ when appropriate
* Multi-AZ databases
* Redundant routes
* Auto Scaling groups across AZs
* EKS node groups across AZs
* VPC endpoints for AWS services
* Route 53 health checks/failover where needed

For production, I avoid making one NAT Gateway or one AZ a single point of failure.

---

### 12) Have you worked on scripting? Which tool and what did you implement?

Yes. I commonly use **Bash and Python**.

With Bash, I've automated operational tasks such as:

```text
Server health checks
Log cleanup
Filesystem monitoring
Application restart validation
Backup verification
Docker cleanup
Kubernetes health checks
```

Example:

```bash
#!/bin/bash

usage=$(df / | awk 'NR==2 {print $5}' | tr -d '%')

if [ "$usage" -gt 80 ]; then
    echo "Disk usage critical: ${usage}%"
fi
```

For more complex automation, I prefer Python.

Examples include:

* AWS resource inventory
* Finding unused EBS volumes
* Checking expiring certificates
* Automating snapshots
* Parsing CloudWatch logs
* Kubernetes health reports
* Calling REST APIs
* Generating operational reports

For AWS automation I use `boto3`.

A good interview point is:

> I use shell scripting for lightweight OS/CI tasks and Python when I need structured data processing, APIs, error handling or AWS SDK integration.

---

### 13) Have you written Terraform code for deployments?

Yes. I've used Terraform to provision cloud infrastructure such as:

```text
VPC
Subnets
Route tables
Security Groups
IAM
EC2
ALB
EKS
RDS
S3
CloudWatch
Route 53
```

I normally structure Terraform using reusable modules:

```text
terraform/
 |
 +-- modules/
 |    +-- vpc/
 |    +-- eks/
 |    +-- rds/
 |    +-- alb/
 |
 +-- environments/
      +-- dev/
      +-- stage/
      +-- prod/
```

Remote state is stored centrally, for example in S3:

```hcl
terraform {
  backend "s3" {
    bucket = "company-terraform-state"
    key    = "prod/eks/terraform.tfstate"
    region = "us-east-1"
  }
}
```

CI/CD typically performs:

```text
terraform fmt
      ↓
terraform validate
      ↓
terraform plan
      ↓
Approval
      ↓
terraform apply
```

I avoid hardcoding values and use:

```text
Variables
Modules
Outputs
Data sources
Remote state
Provider aliases
```

For production changes, `terraform plan` is reviewed before apply.

---

### 14) Explain your hands-on experience with Docker.

I've worked with Docker for building, packaging, troubleshooting and optimizing application containers.

A typical Dockerfile:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

USER 10001

EXPOSE 8080

CMD ["python", "app.py"]
```

I use:

```bash
docker build -t app:v1 .
docker run -p 8080:8080 app:v1
docker ps
docker logs <container>
docker exec -it <container> sh
docker inspect <container>
```

In production, important practices include:

* Multi-stage builds
* Minimal base images
* Non-root user
* Image vulnerability scanning
* `.dockerignore`
* No secrets inside images
* Versioned immutable tags
* Resource limits
* Health checks

For example:

```text
Source Code
   |
Docker Build
   |
Image Scan
   |
ECR
   |
Kubernetes
```

---

### 15) Why do we use workspaces in Terraform?

Terraform workspaces allow multiple state instances for the same configuration.

For example:

```bash
terraform workspace new dev
terraform workspace new qa
terraform workspace new prod
```

Check:

```bash
terraform workspace list
```

Switch:

```bash
terraform workspace select dev
```

Then Terraform maintains different state for each workspace.

You can reference it:

```hcl
tags = {
  Environment = terraform.workspace
}
```

However, for large production environments I generally prefer separate state/configurations or folders for strong environment isolation:

```text
environments/
  dev/
  qa/
  prod/
```

Why?

Because workspaces share the same configuration and backend setup, and production isolation can become harder to reason about.

So my answer is:

> Workspaces are useful for multiple similar instances of the same configuration, especially temporary or lightweight environments, but for strongly isolated production environments I usually prefer separate state boundaries.

---

### 16) Tell me about Ansible. Have you worked on it? In what context?

Ansible is an agentless configuration-management and automation tool.

It commonly connects to Linux machines over SSH.

Architecture:

```text
Ansible Control Node
        |
        | SSH
        |
   +----+----+
   |         |
Server 1   Server 2
```

An inventory might look like:

```ini
[web]
10.0.1.10
10.0.1.11

[db]
10.0.2.10
```

A simple playbook:

```yaml
- name: Configure web servers
  hosts: web
  become: yes

  tasks:
    - name: Install nginx
      package:
        name: nginx
        state: present

    - name: Start nginx
      service:
        name: nginx
        state: started
        enabled: yes
```

I've used Ansible-style automation for tasks such as:

```text
Package installation
OS configuration
User creation
SSH configuration
Application deployment
Certificate installation
Service restart
Configuration file deployment
Security hardening
Patching
```

For larger implementations, I organize playbooks using roles:

```text
roles/
 |
 +-- nginx/
 |    +-- tasks/
 |    +-- templates/
 |    +-- handlers/
 |
 +-- java/
 |
 +-- monitoring/
```

An important interview distinction is:

```text
Terraform
    |
    +--> Infrastructure provisioning

Ansible
    |
    +--> Configuration management
```

For example:

```text
Terraform
   |
Create EC2
   |
Create VPC
   |
Create security groups
   |
   v
Ansible
   |
Install packages
   |
Configure application
   |
Start services
```

They complement each other rather than replace one another.

For senior interviews, try to answer these questions around **availability, customer impact, automation, observability, recovery, and prevention**, rather than only describing commands. Interviewers usually care more about how you operate production systems than whether you remember every command exactly.
