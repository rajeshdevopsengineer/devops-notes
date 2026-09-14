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
