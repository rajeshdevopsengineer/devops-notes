Below is a **5-year DevOps/SRE interview-ready answer set** for the Alphadyne round. The key here is to answer like someone who has actually operated production systems: explain the design, why you chose it, failure cases, and tradeoffs.

## 1. Introduce yourself

A strong 60–90 second introduction:

> I have around 5 years of experience in DevOps, cloud, and SRE, mainly working on AWS and Kubernetes-based environments. My core experience includes AWS services such as EC2, VPC, IAM, ALB, Route 53, S3, RDS, EKS, CloudWatch, and Secrets Manager.
>
> On the DevOps side, I have worked with Jenkins, Git, Docker, Kubernetes, Helm, Terraform, Ansible, and monitoring/logging tools such as Prometheus, Grafana, ELK/OpenSearch, and CloudWatch.
>
> My day-to-day responsibilities include building CI/CD pipelines, managing infrastructure using Terraform, deploying applications to Kubernetes, handling production incidents, improving observability, implementing security best practices, and automating repetitive operational tasks using Bash and Python.
>
> I have also worked on improving application availability by moving workloads to multi-AZ architectures, implementing rolling or canary deployments, configuring autoscaling, and improving monitoring and alerting.
>
> Apart from project delivery, I regularly handle production troubleshooting and RCA, so I am comfortable working across application, Kubernetes, Linux, networking, and AWS layers.

The important point: **don't list 25 technologies without connecting them to actual work.**

---

# 2. Application is hosted on a public EC2 instance. How do you migrate it to a private subnet?

Start with the current architecture:

```text
Internet
   |
Public IP
   |
EC2
Application
```

This isn't ideal because the EC2 instance itself is directly exposed.

I would redesign it like this:

```text
                     Internet
                        |
                    Route 53
                        |
                      HTTPS
                        |
                       ALB
                 Public Subnets
               /                \
            AZ-1                AZ-2
              |                  |
              +--------+---------+
                       |
                Private Subnets
               /                \
            EC2-1              EC2-2
            App                 App
```

### Migration approach

First, create or verify a VPC with at least two Availability Zones:

```text
VPC
 |
 +-- AZ-A
 |    +-- Public subnet
 |    +-- Private application subnet
 |
 +-- AZ-B
      +-- Public subnet
      +-- Private application subnet
```

The **ALB goes in public subnets**.

The **EC2 application servers go in private subnets**.

The EC2 instances should:

* Have no public IP
* Accept application traffic only from the ALB security group
* Use NAT Gateway if outbound internet access is required
* Prefer VPC endpoints for AWS services such as S3, ECR, SSM, Secrets Manager, etc.
* Use IAM roles instead of access keys

Security groups:

```text
Internet
   |
443
   |
ALB Security Group
   |
Application port, e.g. 8080
   |
EC2 Security Group
```

For example:

```text
ALB-SG:
Inbound:
443 from 0.0.0.0/0

App-SG:
Inbound:
8080 only from ALB-SG
```

Not:

```text
8080 from 0.0.0.0/0
```

For HA, I would use:

```text
ALB
 |
Auto Scaling Group
 |
+-- EC2 in AZ-1
+-- EC2 in AZ-2
```

During migration:

```text
Build private infrastructure
        ↓
Deploy application
        ↓
Test using internal endpoint
        ↓
Register targets with ALB
        ↓
Validate health checks
        ↓
Update Route 53
        ↓
Monitor
        ↓
Remove old public exposure
```

If possible, I would use weighted DNS or blue/green cutover rather than an immediate irreversible switch.

---

# 3. How do you provide HTTPS access to an application in a private subnet?

The application server itself does not need to be public.

Architecture:

```text
User
 |
HTTPS :443
 |
Route 53
 |
Public ALB
 |
TLS termination using ACM certificate
 |
HTTP/HTTPS
 |
Private EC2
```

I would request or import an ACM certificate:

```text
*.example.com
or
app.example.com
```

Attach it to an ALB HTTPS listener:

```text
Listener:
HTTPS :443
Certificate:
ACM certificate
```

Then forward traffic:

```text
443
 |
ALB
 |
Target Group
 |
Private EC2 :8080
```

If end-to-end encryption is required:

```text
Client
 HTTPS
   |
 ALB
 HTTPS
   |
 EC2
```

Otherwise the ALB can terminate TLS:

```text
Client
 HTTPS
   |
 ALB
 HTTP
   |
 EC2
```

For most applications, terminating TLS at the ALB is acceptable unless security requirements demand encryption all the way to the backend.

---

# 4. ALB vs NLB

A common interview question.

### ALB

Application Load Balancer works at **Layer 7**.

Use it for:

```text
HTTP
HTTPS
Host-based routing
Path-based routing
Header-based routing
```

Example:

```text
api.example.com
      |
      +-- /orders   -> Order Service
      |
      +-- /payments -> Payment Service
```

It also integrates well with:

```text
ACM
WAF
Cognito
OIDC
EKS Ingress
```

### NLB

Network Load Balancer works primarily at **Layer 4**.

Use it for:

```text
TCP
UDP
TLS
Very high connection volumes
Static IP requirement
Preserving source IP scenarios
```

A useful summary:

| ALB             | NLB                     |
| --------------- | ----------------------- |
| Layer 7         | Layer 4                 |
| HTTP/HTTPS      | TCP/UDP/TLS             |
| Path routing    | No HTTP path routing    |
| Host routing    | No HTTP host routing    |
| WAF integration | Different feature set   |
| Web apps / APIs | Network-level workloads |

For a normal web application:

> I would generally choose ALB because I need HTTPS termination, health checks, and Layer 7 routing.

---

# 5. Infrastructure changes you worked on

Don't answer this as:

> "I created EC2, VPC, S3."

Give an actual architecture improvement.

A strong example:

> One significant infrastructure improvement I worked on was converting a single-AZ application architecture into a multi-AZ highly available design.

Before:

```text
Internet
 |
EC2
 |
Database
```

Problems:

```text
Single EC2
Single AZ
Manual scaling
Public exposure
Limited monitoring
```

After:

```text
                     Route 53
                        |
                       ALB
                +-------+-------+
                |               |
              AZ-1             AZ-2
                |               |
               EC2             EC2
                \               /
                 \             /
                  Multi-AZ RDS
```

We added:

```text
Auto Scaling Group
ALB
Private subnets
Multi-AZ RDS
CloudWatch alarms
Centralized logging
Secrets Manager
Terraform
CI/CD
```

Challenges included:

* Session persistence
* Hardcoded application configuration
* Database connection handling
* Security-group redesign
* DNS cutover
* Backward compatibility
* Rollback planning

This answer is much stronger because it demonstrates architecture thinking.

---

# 6. One major Kubernetes production issue

Use a memorable incident with a clear RCA.

### Situation

After a production release, users experienced intermittent HTTP 500 errors.

Monitoring showed:

```text
5xx errors ↑
Latency ↑
Pod restarts ↑
```

First:

```bash
kubectl get pods -n production
```

I saw:

```text
payment-service-xxx   CrashLoopBackOff
payment-service-yyy   Running
payment-service-zzz   CrashLoopBackOff
```

Then:

```bash
kubectl describe pod payment-service-xxx -n production
```

I found:

```text
Reason: OOMKilled
Exit Code: 137
```

Then:

```bash
kubectl logs payment-service-xxx \
  -n production \
  --previous
```

Grafana showed memory usage increasing significantly after the new deployment.

### Immediate mitigation

Since customers were impacted, we first restored service:

```bash
kubectl rollout undo deployment/payment-service \
  -n production
```

After rollback:

```text
5xx ↓
Latency ↓
Restarts stopped
```

### Root cause

The latest release introduced increased memory consumption.

The container memory limit was:

```text
512Mi
```

but application usage crossed the limit.

Linux cgroups terminated the process, causing:

```text
OOMKilled
```

Kubernetes restarted it, causing intermittent failures.

### Permanent fix

We:

* Fixed the application's memory issue
* Updated requests/limits based on measured usage
* Added memory alerts
* Added performance testing
* Implemented canary deployment
* Added better deployment monitoring
* Defined rollback criteria
* Documented an RCA

The strongest closing statement:

> During an incident my first priority is restoring customer service. Root-cause analysis comes immediately afterward; I don't continue experimenting in production while users are impacted.

---

# 7. Troubleshooting approach for production issues

I follow:

```text
Detect
  ↓
Determine blast radius
  ↓
Check recent changes
  ↓
Metrics
  ↓
Logs
  ↓
Traces
  ↓
Infrastructure
  ↓
Hypothesis
  ↓
Mitigate
  ↓
RCA
  ↓
Prevent recurrence
```

For Kubernetes:

```bash
kubectl get pods
kubectl get nodes
kubectl get events --sort-by=.lastTimestamp
kubectl describe pod <pod>
kubectl logs <pod>
kubectl logs <pod> --previous
kubectl top pods
kubectl top nodes
```

I also immediately ask:

```text
What changed?
```

Deployment?

ConfigMap?

Secret?

Database migration?

Certificate?

Network rule?

Infrastructure change?

This often reduces MTTR significantly.

---

# 8. Branching strategy

For an organization supporting DEV, QA, PROD, releases, and hotfixes, a common approach is a simplified GitFlow-style model:

```text
feature/*
    |
    v
develop
    |
    v
release/*
    |
    v
main
```

Hotfix:

```text
main
 |
hotfix/*
 |
main
 |
merge back to develop
```

Example:

```text
feature/payment-api
       |
       v
    develop
       |
       v
release/2.5
       |
       v
      main
       |
      PROD
```

Environment mapping might be:

```text
feature branch -> temporary/dev testing

develop
   |
   v
DEV

release/*
   |
   v
QA / Stage

main
   |
   v
PROD
```

---

# 9. Why choose this branching strategy?

Because it provides separation between:

```text
Ongoing development
Release stabilization
Production
Hotfixes
```

Suppose version `2.0` is being tested while developers are already working on `2.1`.

Without release branches:

```text
Development for 2.1
       +
Bug fixes for 2.0
```

can interfere.

With release branches:

```text
develop
 |
 |---- New 2.1 development
 |
release/2.0
 |
 |---- QA fixes only
 |
main
 |
Production
```

This lets development continue without blocking the current release.

However, I would also mention:

> For teams practicing true continuous delivery, trunk-based development can be simpler and faster. The branching strategy should match release frequency and organizational requirements rather than being selected blindly.

That is a strong senior-level answer.

---

# 10. How do hotfixes work?

If production has a critical bug:

```text
main
 |
 v
hotfix/payment-timeout
```

Fix and test.

Then:

```text
hotfix
 |
 +--> main
 |
 +--> develop
```

Why merge it back into development?

Otherwise:

```text
Production contains fix
Development does not
```

and the next release can reintroduce the same bug.

---

# 11. Developer has only written source code. How do you design the CI/CD pipeline?

I would divide the problem into:

```text
Source
Build
Test
Security
Package
Publish
Deploy
Validate
Promote
```

Architecture:

```text
Developer
   |
   v
Git Repository
   |
   v
Jenkins
   |
   +--> Compile/build
   |
   +--> Unit tests
   |
   +--> SonarQube
   |
   +--> SAST
   |
   +--> Dependency scan
   |
   +--> Docker build
   |
   +--> Image vulnerability scan
   |
   v
ECR
   |
   v
Helm / Kustomize
   |
   v
Kubernetes
```

---

# 12. DEV → QA → PROD pipeline

A production flow:

```text
Developer Commit
      |
      v
Git
      |
      v
Jenkins CI
      |
      +--> Build
      +--> Unit Test
      +--> SonarQube
      +--> Security Scan
      +--> Docker Build
      +--> Trivy Scan
      |
      v
     ECR
      |
      v
     DEV
      |
Integration Test
      |
      v
     QA
      |
QA/UAT
      |
Approval
      |
      v
    PROD
```

Important principle:

> Build the artifact once and promote the same immutable artifact between environments.

Don't do:

```text
Build DEV image
Build QA image
Build PROD image
```

Instead:

```text
app:1.8.4
   |
   +--> DEV
   |
   +--> QA
   |
   +--> PROD
```

This ensures what was tested in QA is exactly what runs in production.

---

# 13. What would you create if developer only gives source code?

As DevOps engineer, I may create:

```text
Dockerfile
Jenkinsfile
Helm chart / Kubernetes manifests
Terraform
Monitoring configuration
Logging configuration
Secrets integration
Deployment strategy
```

Repository:

```text
application/
 |
 +-- src/
 |
 +-- Dockerfile
 |
 +-- Jenkinsfile
 |
 +-- helm/
 |    +-- Chart.yaml
 |    +-- values.yaml
 |    +-- templates/
 |
 +-- terraform/
```

I would also discuss application requirements with developers:

```text
Runtime
Port
Health endpoint
CPU/memory
Environment variables
Database
Secrets
External APIs
Persistent storage
Startup command
```

A DevOps engineer should not assume these details.

---

# 14. Example CI pipeline stages

For Jenkins:

```groovy
pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Unit Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t app:${BUILD_NUMBER} .'
            }
        }

        stage('Image Scan') {
            steps {
                sh 'trivy image app:${BUILD_NUMBER}'
            }
        }

        stage('Push Image') {
            steps {
                sh 'docker push <ecr>/app:${BUILD_NUMBER}'
            }
        }

        stage('Deploy Dev') {
            steps {
                sh '''
                helm upgrade --install app ./helm \
                  --namespace dev \
                  --set image.tag=${BUILD_NUMBER}
                '''
            }
        }
    }
}
```

In a mature environment, credentials should come from Jenkins credentials integration, IAM roles, OIDC, Vault, Secrets Manager, etc., not hardcoded in the Jenkinsfile.

---

# 15. Production deployment best practices

For production I would implement:

```text
Approval
        ↓
Canary / Rolling / Blue-Green
        ↓
Readiness checks
        ↓
Observe metrics
        ↓
Promote or rollback
```

For Kubernetes rolling deployment:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

Also:

```text
Readiness probe
Liveness probe
PodDisruptionBudget
Multiple replicas
Multi-AZ
HPA
Rollback capability
```

---

# 16. Jenkins Multibranch Pipeline

A Jenkins Multibranch Pipeline automatically discovers repository branches containing a `Jenkinsfile`.

Suppose repository contains:

```text
main
develop
feature/login
feature/payment
release/2.0
```

Jenkins automatically creates separate jobs:

```text
Application
 |
 +-- main
 |
 +-- develop
 |
 +-- feature/login
 |
 +-- feature/payment
 |
 +-- release/2.0
```

Each branch runs its own `Jenkinsfile`.

---

# 17. How do you configure a Multibranch Pipeline?

In Jenkins:

```text
New Item
   |
Multibranch Pipeline
   |
Branch Sources
   |
GitHub / GitLab / Bitbucket
   |
Repository URL
   |
Credentials
   |
Branch discovery rules
   |
Save
```

Jenkins scans the repository and identifies branches containing:

```text
Jenkinsfile
```

Ideally configure a webhook:

```text
Git Push
   |
GitHub webhook
   |
Jenkins
   |
Branch pipeline starts
```

Rather than constant SCM polling.

---

# 18. What problem does a Multibranch Pipeline solve?

With a traditional pipeline, you may have:

```text
job-feature1
job-feature2
job-develop
job-release
job-main
```

and someone manually configures each one.

With Multibranch:

```text
Repository
   |
   +--> Branch discovered automatically
   |
   +--> Jenkinsfile detected
   |
   +--> Pipeline created automatically
```

If a developer creates:

```text
feature/new-payment
```

Jenkins automatically discovers it.

When the branch is deleted, Jenkins can clean up the corresponding job.

This reduces:

```text
Manual job creation
Configuration drift
Maintenance overhead
```

---

# 19. Why Multibranch Pipeline instead of normal SCM pipeline?

A normal pipeline generally tracks one configured branch.

Example:

```text
Jenkins Job
   |
main
```

A Multibranch Pipeline handles many:

```text
Jenkins
 |
 +-- main
 +-- develop
 +-- feature/*
 +-- release/*
 +-- hotfix/*
```

Comparison:

| Normal Pipeline                    | Multibranch                    |
| ---------------------------------- | ------------------------------ |
| Usually one branch/job             | Automatic branch discovery     |
| Manual setup                       | Automatic job creation         |
| Poor fit for many feature branches | Excellent for feature branches |
| One pipeline context               | Branch-specific pipelines      |
| More maintenance                   | Lower maintenance              |

This is especially useful for pull-request validation.

For example:

```text
Developer opens PR
       |
       v
Jenkins
       |
Build
Test
Security scan
       |
       v
PR status check
```

Developers cannot merge if required checks fail.

---

# 20. Strong end-to-end answer for Alphadyne

If they ask, **"Design the whole deployment platform,"** connect everything:

```text
                        Developer
                            |
                            v
                        GitHub
                            |
                     Pull Request
                            |
                            v
                 Jenkins Multibranch
                            |
             +--------------+--------------+
             |              |              |
           Build           Test          Scan
             |              |              |
             +--------------+--------------+
                            |
                      Docker Image
                            |
                            v
                           ECR
                            |
                            v
                       DEV EKS
                            |
                     Integration Test
                            |
                            v
                       QA EKS
                            |
                         UAT
                            |
                     Manual Approval
                            |
                            v
                     Production EKS
                            |
                  Canary / Rolling Deploy
                            |
                      ALB + Route53
                            |
                           Users
```

Infrastructure:

```text
Terraform
   |
   +--> VPC
   +--> Subnets
   +--> EKS
   +--> IAM
   +--> ALB
   +--> RDS
   +--> Route 53
```

Security:

```text
IAM roles
Private subnets
Security groups
Secrets Manager
ACM
WAF
Image scanning
Least privilege
```

Observability:

```text
Prometheus
Grafana
CloudWatch
Centralized logs
Alerts
Tracing
```

The recurring theme in your answers should be:

> **I don't only deploy an application. I design for security, availability, observability, rollback, automation, and operational support.**

That is the level interviewers typically expect from a **5-year DevOps/SRE candidate**.

For this Alphadyne assessment, the MCQs are mostly straightforward. The Azure Terraform task is the part where they will check whether you understand **networking relationships**, not just Terraform syntax.

## 1. Likely MCQ answers

| Question                                                        | Expected answer                                     | What to say if asked                                                                                |
| --------------------------------------------------------------- | --------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| Tool to automate deployment/configuration of VMs and containers | **Ansible**                                         | Ansible is configuration management/automation. Terraform is primarily infrastructure provisioning. |
| Module to install nginx using Ansible                           | **`ansible.builtin.package`**                       | Or `apt` for Ubuntu/Debian and `dnf`/`yum` for RHEL-family systems.                                 |
| Module to ensure a package is installed                         | **`ansible.builtin.package` with `state: present`** | Generic cross-platform package module.                                                              |
| Kubernetes object that ensures pods maintain desired state      | **Deployment**                                      | Deployment manages ReplicaSets, which maintain the requested replicas.                              |
| Monitor disk reads/writes                                       | **`iostat`**                                        | `iostat -xz 1` is a strong troubleshooting command. `iotop` gives per-process I/O.                  |
| Where to put a developer's SSH public key                       | **`~/.ssh/authorized_keys`**                        | Under the target user's home directory.                                                             |
| Where user passwords are stored on modern Linux                 | **`/etc/shadow`**                                   | `/etc/passwd` stores account metadata; password hashes are normally in `/etc/shadow`.               |

For nginx, this is valid Ansible:

```yaml
- name: Install nginx
  ansible.builtin.package:
    name: nginx
    state: present
```

On Ubuntu you could specifically use:

```yaml
- name: Install nginx
  ansible.builtin.apt:
    name: nginx
    state: present
    update_cache: yes
```

`ansible.builtin.apt` manages apt packages, while `package` is the generic abstraction. ([Ansible Documentation][1])

### Kubernetes question — important distinction

If the exact MCQ says:

> Which Kubernetes object ensures that a required number of Pods are always running?

Answer:

**Deployment/ReplicaSet**.

If it says:

> How do you guarantee CPU and memory for a Pod?

Answer:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "1"
    memory: "1Gi"
```

If it asks about namespace-wide resource restrictions, that's **ResourceQuota/LimitRange**.

---

# Azure Terraform Coding Task

I would first explain the architecture:

```text
                         Internet
                            |
                            |
                    Public IP - LB
                            |
                    Azure Load Balancer
                            |
                  +---------+---------+
                  |                   |
              App VM 1             App VM 2
                  |                   |
                  +--------+----------+
                           |
                    App Subnet
                     10.0.1.0/24
                           |
                           |
                       DB VM
                           |
                     DB Subnet
                     10.0.2.0/24


Administrator
     |
 Internet
     |
 Bastion Public IP
     |
 Azure Bastion
     |
 AzureBastionSubnet
  10.0.0.0/26
     |
     +------ SSH -----> App VMs / DB VM
```

The important security point is:

**The application and database VMs do not need public IP addresses.**

Only:

```text
Azure Load Balancer
Azure Bastion
```

receive public IPs.

Azure Bastion requires a dedicated subnet named exactly `AzureBastionSubnet`; for current non-Developer deployments, it should be at least `/26`. A public Bastion deployment uses a Standard, static public IP. ([Terraform Registry][2])

---

## `providers.tf`

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 5.0"
    }
  }
}

provider "azurerm" {
  features {}

  subscription_id = var.subscription_id
}
```

---

## `variables.tf`

```hcl
variable "subscription_id" {
  type      = string
  sensitive = true
}

variable "location" {
  type    = string
  default = "East US"
}

variable "admin_username" {
  type    = string
  default = "azureadmin"
}

variable "admin_password" {
  type      = string
  sensitive = true
}
```

For the assessment, password authentication is used because it was specifically requested.

In production, I would generally prefer SSH keys/identity-based access. Also, AzureRM stores VM password arguments in Terraform state, so state security becomes important. ([Terraform Registry][3])

---

# `main.tf`

## Resource group

```hcl
resource "azurerm_resource_group" "rg" {
  name     = "rg-alphadyne-demo"
  location = var.location
}
```

---

## VNet

```hcl
resource "azurerm_virtual_network" "vnet" {
  name                = "alphadyne-vnet"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  address_space = [
    "10.0.0.0/16"
  ]
}
```

Architecture:

```text
VNet
10.0.0.0/16
```

---

## Bastion subnet

The subnet name must be exactly:

```text
AzureBastionSubnet
```

```hcl
resource "azurerm_subnet" "bastion" {
  name                 = "AzureBastionSubnet"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name

  address_prefixes = [
    "10.0.0.0/26"
  ]
}
```

---

## Application subnet

```hcl
resource "azurerm_subnet" "application" {
  name                 = "application-subnet"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name

  address_prefixes = [
    "10.0.1.0/24"
  ]
}
```

---

## Database subnet

```hcl
resource "azurerm_subnet" "database" {
  name                 = "database-subnet"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name

  address_prefixes = [
    "10.0.2.0/24"
  ]
}
```

So we now have:

```text
10.0.0.0/16
     |
     +-- AzureBastionSubnet
     |      10.0.0.0/26
     |
     +-- application-subnet
     |      10.0.1.0/24
     |
     +-- database-subnet
            10.0.2.0/24
```

---

# Application NSG

Allow web traffic through the Load Balancer and SSH from Bastion.

```hcl
resource "azurerm_network_security_group" "app" {
  name                = "app-nsg"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  security_rule {
    name                       = "Allow-HTTP"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "80"
    source_address_prefix      = "Internet"
    destination_address_prefix = "*"
  }

  security_rule {
    name                       = "Allow-Bastion-SSH"
    priority                   = 110
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "22"
    source_address_prefix      = "10.0.0.0/26"
    destination_address_prefix = "*"
  }
}
```

Associate it:

```hcl
resource "azurerm_subnet_network_security_group_association" "app" {
  subnet_id                 = azurerm_subnet.application.id
  network_security_group_id = azurerm_network_security_group.app.id
}
```

---

# Database NSG

Only application servers should access the database.

Assuming PostgreSQL:

```hcl
resource "azurerm_network_security_group" "db" {
  name                = "db-nsg"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  security_rule {
    name                       = "Allow-App-Postgres"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "5432"
    source_address_prefix      = "10.0.1.0/24"
    destination_address_prefix = "*"
  }

  security_rule {
    name                       = "Allow-Bastion-SSH"
    priority                   = 110
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "22"
    source_address_prefix      = "10.0.0.0/26"
    destination_address_prefix = "*"
  }
}
```

Associate:

```hcl
resource "azurerm_subnet_network_security_group_association" "db" {
  subnet_id                 = azurerm_subnet.database.id
  network_security_group_id = azurerm_network_security_group.db.id
}
```

---

# Application NICs

Let's create **two application VMs** so that the Load Balancer has multiple backends.

```hcl
resource "azurerm_network_interface" "app" {
  count = 2

  name                = "app-nic-${count.index + 1}"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  ip_configuration {
    name = "app-ipconfig-${count.index + 1}"

    subnet_id = azurerm_subnet.application.id

    private_ip_address_allocation = "Dynamic"
  }
}
```

Notice there is **no `public_ip_address_id`**.

Therefore the VMs stay private.

---

# Database NIC

```hcl
resource "azurerm_network_interface" "db" {
  name                = "db-nic"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  ip_configuration {
    name = "db-ipconfig"

    subnet_id = azurerm_subnet.database.id

    private_ip_address_allocation = "Dynamic"
  }
}
```

---

# Application VMs

```hcl
resource "azurerm_linux_virtual_machine" "app" {
  count = 2

  name                = "app-vm-${count.index + 1}"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  size                = "Standard_B2s"

  admin_username = var.admin_username
  admin_password = var.admin_password

  disable_password_authentication = false

  network_interface_ids = [
    azurerm_network_interface.app[count.index].id
  ]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }

  custom_data = base64encode(<<-EOF
    #!/bin/bash
    apt-get update
    apt-get install -y nginx

    echo "Hello from app-vm-${count.index + 1}" \
      > /var/www/html/index.html

    systemctl enable nginx
    systemctl restart nginx
  EOF
  )
}
```

This creates:

```text
app-vm-1
app-vm-2
```

Both have:

```text
Private IP only
Ubuntu
Nginx
Port 80
```

A Linux VM is attached to Azure networking through its `network_interface_ids`. ([Terraform Registry][3])

---

# Database VM

```hcl
resource "azurerm_linux_virtual_machine" "db" {
  name                = "db-vm"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  size                = "Standard_B2s"

  admin_username = var.admin_username
  admin_password = var.admin_password

  disable_password_authentication = false

  network_interface_ids = [
    azurerm_network_interface.db.id
  ]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }
}
```

Again:

```text
No public IP
```

---

# Azure Bastion public IP

```hcl
resource "azurerm_public_ip" "bastion" {
  name                = "bastion-pip"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  allocation_method = "Static"
  sku               = "Standard"
}
```

---

# Azure Bastion Host

```hcl
resource "azurerm_bastion_host" "bastion" {
  name                = "alphadyne-bastion"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  sku = "Standard"

  ip_configuration {
    name = "bastion-ipconfig"

    subnet_id = azurerm_subnet.bastion.id

    public_ip_address_id = azurerm_public_ip.bastion.id
  }
}
```

The flow becomes:

```text
Administrator
     |
     v
Azure Bastion
     |
     v
Private App/DB VMs
```

Therefore there is no requirement to expose SSH:

```text
VM Public IP -> None
```

Microsoft's current Terraform example follows the same pattern: a dedicated Bastion subnet, Standard public IP, and Bastion IP configuration referencing both. ([Microsoft Learn][4])

---

# Load Balancer public IP

```hcl
resource "azurerm_public_ip" "lb" {
  name                = "lb-public-ip"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  allocation_method = "Static"
  sku               = "Standard"
}
```

---

# Azure Load Balancer

```hcl
resource "azurerm_lb" "app" {
  name                = "app-lb"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  sku = "Standard"

  frontend_ip_configuration {
    name = "public-frontend"

    public_ip_address_id = azurerm_public_ip.lb.id
  }
}
```

---

# Backend Pool

```hcl
resource "azurerm_lb_backend_address_pool" "app" {
  name            = "app-backend-pool"
  loadbalancer_id = azurerm_lb.app.id
}
```

The backend pool represents:

```text
app-vm-1
app-vm-2
```

Current AzureRM supports a dedicated load-balancer backend-address-pool resource. ([Terraform Registry][5])

---

# Associate both application NICs with the Load Balancer

```hcl
resource "azurerm_network_interface_backend_address_pool_association" "app" {
  count = 2

  network_interface_id = azurerm_network_interface.app[count.index].id

  ip_configuration_name = "app-ipconfig-${count.index + 1}"

  backend_address_pool_id = azurerm_lb_backend_address_pool.app.id
}
```

This relationship is:

```text
Load Balancer
      |
Backend Pool
      |
 +----+----+
 |         |
NIC 1     NIC 2
 |         |
VM 1      VM 2
```

Azure provides a dedicated NIC-to-backend-pool association resource for this purpose. ([Terraform Registry][6])

---

# Health Probe

The Load Balancer must know whether each application VM is healthy.

```hcl
resource "azurerm_lb_probe" "http" {
  name            = "http-health-probe"
  loadbalancer_id = azurerm_lb.app.id

  protocol     = "Http"
  port         = 80
  request_path = "/"
}
```

Conceptually:

```text
Load Balancer
     |
GET /
     |
app-vm-1 -> Healthy
app-vm-2 -> Healthy
```

If one server stops responding, traffic is sent to the healthy backend.

---

# Load Balancer Rule

```hcl
resource "azurerm_lb_rule" "http" {
  name = "http-rule"

  loadbalancer_id = azurerm_lb.app.id

  protocol = "Tcp"

  frontend_port = 80
  backend_port  = 80

  frontend_ip_configuration_name = "public-frontend"

  backend_address_pool_ids = [
    azurerm_lb_backend_address_pool.app.id
  ]

  probe_id = azurerm_lb_probe.http.id
}
```

Traffic flow:

```text
User
 |
Public IP :80
 |
Azure Load Balancer
 |
Backend Pool
 |
+----------+----------+
|                     |
App VM 1 :80       App VM 2 :80
```

This backend-pool + health-probe + rule structure is also the pattern Microsoft documents for Terraform load-balancer deployments. ([Microsoft Learn][4])

---

# Outputs

```hcl
output "load_balancer_public_ip" {
  value = azurerm_public_ip.lb.ip_address
}

output "bastion_dns_name" {
  value = azurerm_bastion_host.bastion.dns_name
}

output "application_private_ips" {
  value = [
    for nic in azurerm_network_interface.app :
    nic.private_ip_address
  ]
}

output "database_private_ip" {
  value = azurerm_network_interface.db.private_ip_address
}
```

After:

```bash
terraform apply
```

you could get conceptually:

```text
load_balancer_public_ip = "20.x.x.x"

application_private_ips = [
  "10.0.1.4",
  "10.0.1.5"
]

database_private_ip = "10.0.2.4"
```

Then:

```text
http://20.x.x.x
```

would return something like:

```text
Hello from app-vm-1
```

or:

```text
Hello from app-vm-2
```

depending on which backend receives the request.

---

# Commands to execute it

```bash
az login
```

Set your subscription:

```bash
az account set --subscription "<subscription-id>"
```

Then:

```bash
terraform init
```

```bash
terraform fmt
```

```bash
terraform validate
```

```bash
terraform plan
```

```bash
terraform apply
```

Microsoft's Terraform guidance follows the same `init → plan → apply` workflow. ([Microsoft Learn][4])

## How to explain the entire task in 30 seconds

> I created one VNet with separate Bastion, application, and database subnets. The application and database VMs have only private NICs, so they're not directly exposed to the internet. Azure Bastion provides secure administrative access to those private VMs. I created a public Standard Load Balancer with two application VMs in its backend pool, along with a health probe and load-balancing rule. The database subnet only allows database traffic from the application subnet. Public IPs are only assigned where required—to the Load Balancer and Azure Bastion.

That explanation demonstrates **Terraform + Azure networking + security + availability**, which is what the coding exercise is really testing.

[1]: https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/apt_module.html?utm_source=chatgpt.com "ansible.builtin.apt module – Manages apt-packages — Ansible Community Documentation"
[2]: https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/bastion_host.html?utm_source=chatgpt.com "azurerm_bastion_host | Resources | hashicorp/azurerm | Terraform | Terraform Registry"
[3]: https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/linux_virtual_machine?utm_source=chatgpt.com "azurerm_linux_virtual_machine | Resources | hashicorp/azurerm | Terraform | Terraform Registry"
[4]: https://learn.microsoft.com/zh-cn/azure/load-balancer/quickstart-load-balancer-standard-internal-terraform "快速入门：创建内部负载均衡器 - Terraform - Azure Load Balancer | Microsoft Learn"
[5]: https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/lb_backend_address_pool?utm_source=chatgpt.com "azurerm_lb_backend_address_pool | Resources | hashicorp/azurerm | Terraform | Terraform Registry"
[6]: https://registry.terraform.io/providers/hashicorp/Azurerm/4.28.0/docs/resources/network_interface_backend_address_pool_association?utm_source=chatgpt.com "azurerm_network_interface_backend_address_pool_association | Resources | hashicorp/azurerm | Terraform | Terraform Registry"

