# BMW TechWorks DevOps Interview Questions and Answers

**Company:** BMW TechWorks  
**Experience:** 3 to 4 years  
**Profile:** DevOps / Cloud / Platform Engineer

> **How to use this document:** The answers below are interview-ready examples. Replace placeholders and sample project details with your real experience. Do not claim hands-on ownership of a tool if you only have theoretical or lab exposure.

---

## 1. Introduce yourself and explain your DevOps experience in the current organization

### Sample answer

Hello, I am **[Your Name]**, and I have approximately **3.5 years of experience in DevOps and cloud engineering**. In my current organization, I support the build, deployment, automation, monitoring, and reliability of applications running primarily on AWS.

My main responsibilities include:

- Building and maintaining CI/CD pipelines using **[Jenkins/GitHub Actions/GitLab CI/Azure DevOps]**.
- Managing AWS resources such as EC2, IAM, VPC, ALB, Auto Scaling, S3, CloudWatch, ECR, and EKS.
- Provisioning infrastructure with Terraform and maintaining reusable modules and remote state.
- Containerizing applications with Docker and deploying them to Kubernetes.
- Managing Kubernetes workloads, ConfigMaps, Secrets, Services, Ingress, autoscaling, and rolling deployments.
- Automating operational tasks with Shell or Python.
- Monitoring services through **[CloudWatch/Prometheus/Grafana/ELK/Splunk]** and responding to incidents.
- Implementing security controls such as least-privilege IAM, vulnerability scanning, patching, secret management, and encrypted storage.
- Working with developers, QA, security, and infrastructure teams to resolve deployment and production issues.

One significant project I worked on was the migration of **[application name/type]** from **[legacy environment]** to **[AWS/EKS/ECS]**. We containerized the services, provisioned the target infrastructure through Terraform, built CI/CD pipelines, moved secrets to **[Vault/Secrets Manager]**, and introduced centralized monitoring. This reduced manual deployment effort from **[X minutes]** to **[Y minutes]** and improved rollback and release reliability.

### Follow-up points to prepare

- Team size and your exact ownership
- Number of environments and applications
- Deployment frequency
- A production incident you resolved
- One measurable automation or cost improvement
- A difficult migration or pipeline problem

---

## 2. If an EC2 instance has a vulnerability, how would you identify and fix it?

I would use a risk-based and traceable remediation process.

### Identification

1. Use **Amazon Inspector** to identify operating-system and application-package CVEs and network exposure.
2. Aggregate findings in **AWS Security Hub** if it is part of the security architecture.
3. Review the CVE, severity, affected package, exploitability, internet exposure, asset criticality, and available fix.
4. Verify the finding directly on the instance where necessary:

```bash
# Amazon Linux/RHEL-family examples
sudo dnf updateinfo list security
rpm -q <package-name>

# Ubuntu/Debian-family examples
apt list --upgradable
apt-cache policy <package-name>

# Check listening ports and active services
sudo ss -lntup
sudo systemctl --type=service --state=running
```

Amazon Inspector can assess EC2 software inventory through agent-based scanning or agentless scanning based on EBS snapshots. It produces findings for package vulnerabilities and network reachability exposure.

### Containment

For a critical, exploitable vulnerability:

- Remove the instance from the load balancer or isolate it through a quarantine security group.
- Restrict unnecessary inbound and outbound access.
- Preserve required logs and evidence.
- Take an EBS snapshot if rollback or investigation is required.
- Avoid direct internet SSH and use Systems Manager Session Manager where possible.

### Remediation

1. Test the patch in a non-production environment.
2. Back up or snapshot the affected storage.
3. Apply the approved operating-system or application patch using **AWS Systems Manager Patch Manager**, a package manager, or an immutable AMI replacement.
4. Reboot if the patched kernel or library requires it.
5. Run application, health, and regression tests.
6. Re-enable traffic gradually.
7. Re-scan the instance and verify that the Inspector finding is closed or suppressed only with documented risk acceptance.

### Preferred long-term solution

For Auto Scaling workloads, I prefer an **immutable approach**:

```text
Patch base image -> Scan image -> Create new AMI
-> Update launch template -> Rolling instance replacement
-> Validate -> Retire old AMI and instances
```

This avoids configuration drift and makes rollback repeatable.

---

## 3. How do you patch EC2 instances?

The preferred managed approach is **AWS Systems Manager Patch Manager**.

### Prerequisites

- The instance is a Systems Manager managed node.
- SSM Agent is installed and running.
- The EC2 instance profile contains the required Systems Manager permissions.
- The instance can reach Systems Manager endpoints through internet/NAT access or VPC interface endpoints.
- Patch baselines, maintenance windows, tags, reboot behavior, and approval rules are defined.

### Patching workflow

1. Group instances using tags such as `Environment`, `Application`, and `PatchGroup`.
2. Create or select a patch baseline.
3. First run a **scan** operation to identify missing patches.
4. Test installation in development and staging.
5. Snapshot important EBS volumes or validate application backups.
6. Patch production in controlled batches during a maintenance window.
7. Drain or deregister each instance from the ALB where required.
8. Install the patches and reboot according to policy.
9. Run health checks and restore traffic.
10. Review Systems Manager compliance and Amazon Inspector findings.

Useful checks:

```bash
sudo systemctl status amazon-ssm-agent
sudo dnf check-update
uname -r
uptime
```

### Production considerations

- Use one Availability Zone or a limited percentage at a time.
- Respect application quorum and minimum healthy capacity.
- Define rollback and AMI replacement plans.
- Monitor application errors, latency, CPU, memory, disk, and health checks.
- Patch Manager can automate operating-system and supported application patching, but AWS does not test every patch against the customer's application. Application testing remains the customer's responsibility.

---

## 4. Can we separate disk space in an EC2 instance and run the application on one partition and observability on another?

Yes. There are two approaches:

### Option 1: Multiple partitions on one EBS volume

Create separate partitions and mount them at paths such as:

```text
/dev/nvme1n1p1 -> /opt/application
/dev/nvme1n1p2 -> /var/lib/observability
```

This gives logical separation, but both partitions share the same volume's throughput, IOPS, failure boundary, snapshot lifecycle, and capacity management. If one workload saturates the volume, the other can be affected.

### Option 2: Separate EBS volumes, recommended

```text
Root EBS volume          -> /
Application EBS volume   -> /opt/application or /var/lib/app
Observability EBS volume -> /var/lib/prometheus, /var/lib/elasticsearch, or /var/log
```

Separate EBS volumes provide better isolation for:

- Capacity
- Performance and IOPS
- Snapshot and retention policies
- Encryption keys
- Expansion
- Backup and restore
- Monitoring and alerting

For high-write observability tools, select the EBS type based on workload. `gp3` is a common general-purpose option; provisioned IOPS volumes may be appropriate for strict latency or IOPS requirements. Validate the EC2 instance's aggregate EBS bandwidth limits as well.

### Important caution

Running the application and a heavy observability stack on the same EC2 instance can still cause CPU, memory, network, and instance-level EBS contention. For production, consider dedicated observability infrastructure or managed services such as Amazon Managed Service for Prometheus, Amazon Managed Grafana, CloudWatch, or Amazon OpenSearch Service, depending on the requirements.

---

## 5. How do you perform disk separation, and which AWS services or tools do you use?

### AWS services and tools

- **Amazon EBS:** Durable block storage volumes
- **AWS KMS:** Customer-managed encryption keys where required
- **EBS snapshots / AWS Backup:** Backup and recovery
- **CloudWatch Agent:** Filesystem usage and custom metrics
- **AWS Systems Manager:** Remote command execution and automation
- **Terraform or CloudFormation:** Repeatable volume and attachment provisioning
- Linux tools: `lsblk`, `blkid`, `fdisk`/`parted`, `mkfs`, `mount`, `findmnt`, and `/etc/fstab`

An EBS volume must be in the same Availability Zone as the EC2 instance to which it is attached.

### Recommended method: separate EBS volumes

#### 1. Create and attach EBS volumes

Example Terraform:

```hcl
resource "aws_ebs_volume" "application" {
  availability_zone = aws_instance.server.availability_zone
  size              = 100
  type              = "gp3"
  encrypted         = true

  tags = {
    Name = "application-data"
  }
}

resource "aws_volume_attachment" "application" {
  device_name = "/dev/sdf"
  volume_id   = aws_ebs_volume.application.id
  instance_id = aws_instance.server.id
}

resource "aws_ebs_volume" "observability" {
  availability_zone = aws_instance.server.availability_zone
  size              = 200
  type              = "gp3"
  encrypted         = true

  tags = {
    Name = "observability-data"
  }
}

resource "aws_volume_attachment" "observability" {
  device_name = "/dev/sdg"
  volume_id   = aws_ebs_volume.observability.id
  instance_id = aws_instance.server.id
}
```

#### 2. Identify the actual Linux devices

On Nitro-based instances, requested device names may appear as NVMe names.

```bash
lsblk -f
sudo nvme list
sudo file -s /dev/nvme1n1
sudo file -s /dev/nvme2n1
```

#### 3. Format only new, empty volumes

```bash
sudo mkfs.xfs /dev/nvme1n1
sudo mkfs.xfs /dev/nvme2n1
```

> Never run `mkfs` on a device containing required data.

#### 4. Create mount points and mount

```bash
sudo mkdir -p /opt/application
sudo mkdir -p /var/lib/observability

sudo mount /dev/nvme1n1 /opt/application
sudo mount /dev/nvme2n1 /var/lib/observability
```

#### 5. Configure persistent mounts with UUIDs

```bash
sudo blkid
```

Example `/etc/fstab` entries:

```fstab
UUID=<application-volume-uuid> /opt/application xfs defaults,nofail 0 2
UUID=<observability-volume-uuid> /var/lib/observability xfs defaults,nofail 0 2
```

Validate before rebooting:

```bash
sudo mount -a
findmnt
lsblk -f
df -hT
```

#### 6. Set ownership and move data safely

```bash
sudo chown -R appuser:appgroup /opt/application
sudo chown -R prometheus:prometheus /var/lib/observability
```

Stop the service before moving an existing database or active data directory. Copy with preserved ownership, change the service configuration, start the service, and verify data integrity.

#### 7. Monitor and protect

- Publish disk-space metrics with CloudWatch Agent.
- Alert at warning and critical thresholds.
- Schedule snapshots or AWS Backup policies.
- Test restoration.
- Encrypt volumes and control KMS access.

---

## 6. Explain the project that you migrated

Use the following **STAR-style** sample and replace it with your real project.

### Sample answer

**Situation:** Our application was running on manually managed virtual machines with manual deployments. Releases were slow, configuration differed between environments, scaling was difficult, and rollback depended on manual steps.

**Task:** I was part of the team responsible for migrating the application to AWS and introducing repeatable infrastructure, containerization, CI/CD, centralized monitoring, and improved security.

**Action:**

1. Collected the application inventory, dependencies, ports, storage, integrations, certificates, DNS records, and availability requirements.
2. Assessed migration risks, downtime requirements, database compatibility, and rollback options.
3. Provisioned VPC, subnets, security groups, load balancer, compute, IAM, ECR, storage, and monitoring through Terraform.
4. Containerized the application using a multi-stage Dockerfile and ran it as a non-root user.
5. Created CI stages for build, unit tests, dependency scanning, image scanning, ECR publishing, and deployment.
6. Stored environment configuration outside the image and moved secrets to **[Vault/AWS Secrets Manager]**.
7. Deployed to **[EKS/ECS/EC2 Auto Scaling]** and configured health checks, autoscaling, log collection, dashboards, and alarms.
8. Performed data migration and reconciliation using **[DMS/native tools/custom scripts]**.
9. Conducted performance, security, failover, and rollback testing.
10. Used a blue-green or controlled cutover strategy, updated DNS, closely monitored the production environment, and retained a rollback window.

**Result:** The migration reduced manual deployment work, improved deployment frequency and recovery, standardized environments, and provided better monitoring. Use genuine metrics, for example: deployment time reduced from **45 minutes to 10 minutes**, rollback reduced from **30 minutes to under 10 minutes**, and deployment frequency increased from weekly to several times per week.

### Follow-up questions to prepare

- Why did you choose EKS, ECS, or EC2?
- How was the database migrated?
- What was the rollback plan?
- How was downtime minimized?
- What failed during migration and how did you fix it?
- What did you personally own?

---

## 7. Have you containerized any application?

### If you have hands-on experience

Yes. I containerized **[Java/Spring Boot/Node.js/Python]** applications by analyzing runtime dependencies, creating a multi-stage Dockerfile, externalizing configuration, adding health checks, scanning the final image, and publishing it to Amazon ECR.

Example for a Java application:

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /workspace
COPY pom.xml .
RUN mvn -B dependency:go-offline
COPY src ./src
RUN mvn -B clean package -DskipTests

FROM eclipse-temurin:21-jre
RUN useradd --system --uid 10001 appuser
WORKDIR /app
COPY --from=build /workspace/target/*.jar app.jar
USER 10001
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

My process includes:

- Pinning a trusted minimal base image
- Using multi-stage builds
- Excluding secrets and unnecessary files with `.dockerignore`
- Running as a non-root user
- Sending logs to stdout and stderr
- Configuring CPU and memory limits at runtime
- Adding application health endpoints
- Scanning dependencies and images
- Tagging with a version and Git commit SHA
- Deploying the same immutable image digest across environments

### If you only have lab experience

I have containerized applications in projects and labs and understand the complete workflow, but I have not independently owned a production containerization migration. I can explain how I would productionize it, including image hardening, scanning, configuration, secrets, observability, and rollback.

---

## 8. Have you set up any CI/CD pipelines on your own?

### Sample answer

Yes. I created a pipeline for **[application]** using **[Jenkins/GitHub Actions/GitLab CI/Azure DevOps]**.

```text
Commit/PR
  -> Lint and unit tests
  -> SAST and dependency scan
  -> Build artifact and container image
  -> Image scan and SBOM
  -> Push immutable image to ECR
  -> Deploy to development
  -> Integration and smoke tests
  -> Approval/policy gate
  -> Deploy to production
  -> Health verification and rollback
```

I configured:

- Webhook or pull-request triggers
- Branch protection and mandatory checks
- Reusable pipeline templates
- Artifact and Docker-image versioning
- Short-lived AWS authentication through OIDC or an assumed role
- Terraform plan and controlled apply stages
- Kubernetes or ECS deployments
- Environment approvals
- Notifications and audit logs
- Rollback to the previous known-good image digest

Example deployment verification:

```bash
kubectl set image deployment/payment-api \
  payment-api="$ECR_REPOSITORY@$IMAGE_DIGEST" \
  -n production

kubectl rollout status deployment/payment-api -n production --timeout=5m
```

If verification fails:

```bash
kubectl rollout undo deployment/payment-api -n production
```

I also monitor pipeline duration, failure rate, deployment frequency, change failure rate, and recovery time.

---

## 9. Have you used HashiCorp Vault?

### If you have hands-on experience

Yes. I used HashiCorp Vault to centrally manage application secrets instead of storing them in Git repositories, container images, or pipeline variables.

My experience includes:

- KV v2 for static secrets
- Policies for least-privilege access
- Kubernetes, AWS, AppRole, or JWT authentication
- Dynamic or leased credentials where applicable
- Secret rotation and revocation
- Vault Agent, CSI integration, or Vault Secrets Operator for Kubernetes
- TLS, audit devices, token TTLs, and operational monitoring

Example workflow in Kubernetes:

```text
Pod service account -> Kubernetes/JWT authentication -> Vault role
-> Vault policy -> short-lived secret -> application
```

The Vault Secrets Operator can authenticate using Kubernetes, JWT, AppRole, AWS, or other supported methods and synchronize managed secrets into Kubernetes. Vault's AWS secrets engine can generate time-bound AWS credentials and revoke them when the lease expires.

Security principles:

- Use short-lived identities rather than static root tokens.
- Never print secrets in pipeline logs.
- Restrict policies by path and operation.
- Protect unseal/recovery material.
- Enable audit logging.
- Test backup, restore, high availability, and secret rotation.

### If you have not used Vault in production

I have not used Vault in production, but I understand its architecture and have practiced KV secrets, policies, authentication, and Kubernetes integration in a lab. In my current environment, I use **[AWS Secrets Manager/SSM Parameter Store]**. The concepts are similar in terms of centralized storage, least-privilege access, encryption, rotation, and auditability, although the implementation differs.

---

## 10. Have you used Java for coding?

### If you have basic Java exposure

I am not primarily a Java application developer, but I work with Java services from a DevOps perspective. I can read basic Java and troubleshoot builds, runtime configuration, and deployments.

My Java-related work includes:

- Building applications with Maven or Gradle
- Troubleshooting dependency and test failures
- Packaging and running JAR/WAR files
- Configuring JVM options, heap size, garbage collection, and environment variables
- Containerizing Spring Boot applications
- Investigating stack traces and application logs
- Monitoring JVM heap, non-heap memory, threads, GC pauses, and latency
- Managing JDK versions and patching base images

Useful commands:

```bash
java -version
mvn -version
mvn clean test package
java -Xms512m -Xmx1024m -jar application.jar
jps -l
jcmd <PID> VM.flags
jcmd <PID> GC.heap_info
jstack <PID>
```

A strong answer is: “My core coding languages are Shell and Python, while my Java experience is focused on building, packaging, containerizing, deploying, and troubleshooting Java applications.”

### If you have no Java experience

I have not developed production features in Java. My coding work is mainly in Shell and Python. However, I understand the Java build and runtime workflow and have supported Java applications through Maven/Gradle pipelines, Docker, JVM configuration, logs, and monitoring. I am comfortable learning the application-level Java needed for automation and troubleshooting.

---

## Rapid Revision

- Use Amazon Inspector to discover EC2 package vulnerabilities and network exposure.
- Use Systems Manager Patch Manager for controlled fleet patching and compliance reporting.
- Prefer immutable AMI replacement for repeatable Auto Scaling workloads.
- Use separate EBS volumes when application and observability data need capacity, performance, snapshot, or lifecycle isolation.
- A separate volume does not isolate CPU, memory, or network contention on the same instance.
- Explain migrations with Situation, Task, Action, Result, measurable outcomes, and your exact ownership.
- Containerize with multi-stage builds, non-root execution, external configuration, scanning, and immutable tags or digests.
- CI/CD should include quality, security, deployment, verification, and rollback stages.
- Vault supports centralized static and dynamic secrets, short-lived authentication, policy control, rotation, and auditability.
- Be honest about Java depth and emphasize build, JVM, containerization, deployment, and troubleshooting experience.

---

## Official References

- Amazon Inspector EC2 scanning: https://docs.aws.amazon.com/inspector/latest/user/scanning-ec2.html
- Amazon Inspector scan types: https://docs.aws.amazon.com/inspector/latest/user/scanning-resources.html
- AWS Systems Manager Patch Manager: https://docs.aws.amazon.com/systems-manager/latest/userguide/patch-manager.html
- Attach an EBS volume: https://docs.aws.amazon.com/ebs/latest/userguide/ebs-attaching-volume.html
- Make an EBS volume available for use: https://docs.aws.amazon.com/ebs/latest/userguide/ebs-using-volumes.html
- Amazon EBS volumes: https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volumes.html
- HashiCorp Vault Secrets Operator: https://developer.hashicorp.com/vault/docs/deploy/kubernetes/vso/sources/vault
- HashiCorp Vault AWS secrets engine: https://developer.hashicorp.com/vault/docs/secrets/aws
