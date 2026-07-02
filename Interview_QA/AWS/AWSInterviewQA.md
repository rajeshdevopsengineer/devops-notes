# AWS Fundamentals – Detailed Interview-Ready Answers

Here's a comprehensive walkthrough of your 10 AWS questions, structured for a **Senior DevOps interview** with practical examples, CLI snippets, and real-world context.

***

## 1. What is AWS and what are its primary value propositions?

**Amazon Web Services (AWS)** is the world's largest public cloud platform, launched by Amazon in 2006. It offers **200+ on-demand services** across compute, storage, networking, databases, analytics, ML, IoT, and security — delivered via a pay-as-you-go model.

### Core Value Propositions

| Value Proposition                       | Explanation                                                                     |
| --------------------------------------- | ------------------------------------------------------------------------------- |
| **Elasticity & Scalability**            | Scale up/down within minutes using Auto Scaling Groups, Lambda, ECS/EKS.        |
| **Pay-as-you-go pricing**               | No upfront CapEx; pay only for what you consume (per second/hour/GB).           |
| **Global Reach**                        | 30+ Regions, 100+ Availability Zones, 400+ Edge locations (CloudFront).         |
| **High Availability & Fault Tolerance** | Multi-AZ deployments, cross-region replication, Route 53 health checks.         |
| **Security & Compliance**               | Shared Responsibility Model, IAM, KMS, GuardDuty, ISO/SOC/PCI/HIPAA certified.  |
| **Broad Service Catalog**               | Compute (EC2, Lambda), Storage (S3, EBS), DB (RDS, DynamoDB), AI/ML, IoT.       |
| **DevOps Ecosystem**                    | CodePipeline, CodeBuild, CodeDeploy, CloudFormation, native Terraform provider. |
| **Speed of Innovation**                 | \~3,000+ new features/services released per year.                               |

### Real-world example

A startup can launch a global SaaS product on AWS in days — using EC2 for compute, RDS for the DB, S3 for static assets, CloudFront for CDN, and IAM for access control — without buying a single server.

***

## 2. What is an EC2 Instance?

**Amazon EC2 (Elastic Compute Cloud)** provides **resizable virtual servers** in the cloud. An **EC2 instance** is a single VM running on AWS's hypervisor (Nitro/Xen), launched from an **AMI** (image) with a chosen **instance type** (CPU/RAM/network config).

### Key characteristics

* Runs Linux, Windows, macOS, or custom OSes.
* Attached to a **VPC subnet** with a private IP (optionally public IP or Elastic IP).
* Storage via **EBS** (persistent) or **Instance Store** (ephemeral).
* Controlled by **Security Groups** (stateful firewall) and **IAM roles** for AWS API access.
* Billed per second (Linux) or per hour (Windows) — Spot, On-Demand, Reserved, Savings Plans.

### Launch example (AWS CLI)

```bash
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t3.medium \
  --key-name my-keypair \
  --security-group-ids sg-0123456789abcdef0 \
  --subnet-id subnet-0123456789abcdef0 \
  --iam-instance-profile Name=EC2-SSM-Role \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=WebServer-Prod}]'
```

***

## 3. EC2 Instance Types and Families

AWS groups instances into **families** optimized for different workloads. Naming convention: **`[family][generation][additional-caps].[size]`** — e.g., `m5a.xlarge` = general purpose, gen 5, AMD CPU, xlarge size.

### Major Families

| Family                    | Prefix         | Use Case                                                                 | Example Instances                                 |
| ------------------------- | -------------- | ------------------------------------------------------------------------ | ------------------------------------------------- |
| **General Purpose**       | T, M, A        | Balanced CPU/Memory/Network — web servers, small DBs, dev/test           | `t3.medium`, `m6i.large`, `m7g.xlarge` (Graviton) |
| **Compute Optimized**     | C              | High-CPU workloads — batch processing, gaming servers, HPC, ML inference | `c6i.large`, `c7g.xlarge`                         |
| **Memory Optimized**      | R, X, z        | In-memory DBs (Redis, SAP HANA), real-time analytics                     | `r6i.large`, `x2iedn.xlarge`                      |
| **Storage Optimized**     | I, D, H        | High IOPS or throughput — NoSQL DBs, data warehouses, Hadoop             | `i4i.large`, `d3.xlarge`                          |
| **Accelerated Computing** | P, G, Inf, Trn | GPU/FPGA workloads — ML training, graphics rendering                     | `p4d.24xlarge`, `g5.xlarge`                       |
| **HPC Optimized**         | Hpc            | Tightly coupled HPC                                                      | `hpc7g.16xlarge`                                  |

### General Purpose vs Compute Optimized — Key Difference

| Aspect              | General Purpose (M-series)              | Compute Optimized (C-series)                         |
| ------------------- | --------------------------------------- | ---------------------------------------------------- |
| **vCPU:RAM Ratio**  | 1:4 (e.g., `m5.large` = 2 vCPU / 8 GiB) | 1:2 (e.g., `c5.large` = 2 vCPU / 4 GiB)              |
| **CPU Performance** | Balanced                                | Higher clock speed, latest Intel/AMD/Graviton        |
| **Use Case**        | Web apps, microservices, small DBs      | CPU-bound: encoding, batch jobs, scientific modeling |
| **Cost per vCPU**   | Higher (you pay for extra RAM)          | Lower per vCPU                                       |

**Rule of thumb:** If your app is **CPU-bound and doesn't need much RAM**, choose C-series to save cost. If it's **balanced or memory-leaning**, choose M-series.

***

## 4. What is an AMI?

**AMI (Amazon Machine Image)** is a pre-configured template used to launch EC2 instances. It contains:

* **Root volume snapshot** (OS + apps + config).
* **Launch permissions** (who can use it).
* **Block device mapping** (attached volumes).
* **Architecture** (x86\_64, ARM64).

### Types of AMIs

1. **AWS-provided** — Amazon Linux 2/2023, Ubuntu, Windows Server, RHEL.
2. **AWS Marketplace** — vendor images (e.g., Palo Alto, Bitnami).
3. **Community AMIs** — public, shared by users (⚠️ vet carefully).
4. **Custom (private) AMIs** — your own golden images built via **Packer** or `create-image`.

### Creating a Golden AMI (DevOps best practice)

```bash
# From a running, patched, hardened instance
aws ec2 create-image \
  --instance-id i-0abcdef1234567890 \
  --name "webserver-golden-v1.2" \
  --description "Nginx + Node20 + CloudWatch agent, CIS-hardened" \
  --no-reboot
```

Use **EC2 Image Builder** or **HashiCorp Packer** to automate golden AMI pipelines — critical for immutable infrastructure and auto-scaling groups.

***

## 5. How to Launch an EC2 Instance from the Console

### Step-by-Step

1. **Sign in** → AWS Console → **EC2 service** → **Launch Instance**.
2. **Name & Tags** — e.g., `WebServer-Prod`, add cost-allocation tags.
3. **Choose AMI** — Amazon Linux 2023, Ubuntu 22.04, Windows Server 2022, or your custom AMI.
4. **Choose Instance Type** — e.g., `t3.medium` for a web server.
5. **Key Pair** — create/select an SSH key pair (`.pem` for Linux, `.ppk` for Windows RDP password decryption).
6. **Network Settings**:
   * Select **VPC** and **Subnet** (AZ matters for HA).
   * Enable/disable **Auto-assign Public IP**.
   * Configure **Security Group** (e.g., allow port 22 from your IP, 80/443 from `0.0.0.0/0`).
7. **Configure Storage** — root EBS volume (default 8 GB gp3); add extra volumes if needed.
8. **Advanced Details**:
   * **IAM Instance Profile** (e.g., SSM role for Session Manager).
   * **User Data** script for bootstrap (install packages, configure services).
   * Enable **detailed monitoring**, **termination protection**, **IMDSv2**.
9. **Review & Launch** → download key pair → click **Launch Instance**.
10. **Connect**:
    ```bash
    ssh -i my-keypair.pem ec2-user@<public-ip>
    ```
    Or use **Session Manager** (no SSH port needed — best-practice for prod).

### Example User Data (bootstrap Nginx)

```bash
#!/bin/bash
dnf update -y
dnf install -y nginx
systemctl enable --now nginx
echo "<h1>Hello from $(hostname)</h1>" > /usr/share/nginx/html/index.html
```

***

## 6. What is an EBS Volume and How Is It Attached?

**Amazon EBS (Elastic Block Store)** provides **persistent block storage** for EC2 instances — analogous to a virtual hard disk. Data survives instance stop/reboot/termination (if `DeleteOnTermination=false`).

### EBS Volume Types

| Type                          | Use Case                                | Max IOPS | Max Throughput |
| ----------------------------- | --------------------------------------- | -------- | -------------- |
| **gp3** (General Purpose SSD) | Default, cost-effective, boot volumes   | 16,000   | 1,000 MB/s     |
| **gp2** (Legacy SSD)          | Older workloads                         | 16,000   | 250 MB/s       |
| **io2 Block Express**         | Mission-critical DBs (SAP HANA, Oracle) | 256,000  | 4,000 MB/s     |
| **st1** (Throughput HDD)      | Big data, log processing                | 500      | 500 MB/s       |
| **sc1** (Cold HDD)            | Infrequent access, archival             | 250      | 250 MB/s       |

### How EBS is Attached to EC2

1. **Same AZ requirement** — an EBS volume must be in the same AZ as the EC2 instance.
2. **Attach** — either at launch (via block device mapping) or later:
   ```bash
   aws ec2 create-volume --availability-zone us-east-1a --size 100 --volume-type gp3
   aws ec2 attach-volume --volume-id vol-0abc --instance-id i-0xyz --device /dev/sdf
   ```
3. **Format & Mount** (Linux):
   ```bash
   sudo mkfs -t xfs /dev/nvme1n1
   sudo mkdir /data
   sudo mount /dev/nvme1n1 /data
   echo '/dev/nvme1n1 /data xfs defaults,nofail 0 2' | sudo tee -a /etc/fstab
   ```
4. **Snapshots** — Point-in-time backups stored in S3 (incremental). Automate via **Data Lifecycle Manager (DLM)** or **AWS Backup**.
5. **Multi-Attach** — `io1`/`io2` volumes can attach to up to 16 Nitro instances in the same AZ (for clustered apps).

**Key point:** EBS is **AZ-scoped**, unlike S3 which is regional/global.

***

## 7. What is S3 and When to Use It?

**Amazon S3 (Simple Storage Service)** is a **highly durable, scalable object storage** service (not block/file). Data is stored as **objects** (up to 5 TB each) inside **buckets** (globally unique namespace).

### Core Features

* **11 nines of durability** (99.999999999%) — replicated across ≥3 AZs.
* **Virtually unlimited capacity** — trillions of objects.
* **Strong read-after-write consistency** (since Dec 2020).
* **REST API + SDKs + CLI** access.
* **Versioning, replication, lifecycle, event notifications, encryption**.

### When to Use S3

| Use Case                   | Example                                                  |
| -------------------------- | -------------------------------------------------------- |
| **Static website hosting** | React/Angular SPA served via S3 + CloudFront             |
| **Backup & DR**            | RDS snapshots, EBS snapshots, on-prem backups            |
| **Data lake**              | Petabyte-scale analytics with Athena/Redshift Spectrum   |
| **Application assets**     | Images, videos, PDFs for a web/mobile app                |
| **Log storage**            | ALB/CloudFront/VPC Flow Logs, application logs           |
| **Software distribution**  | Installers, container images (indirect), Terraform state |
| **Big data / ML training** | Input datasets for SageMaker, EMR, Glue                  |

### Example (upload with CLI)

```bash
aws s3 cp ./build/ s3://my-webapp-prod/ --recursive --acl public-read
aws s3 sync ./logs s3://company-logs/2026/07/ --storage-class STANDARD_IA
```

***

## 8. S3 Storage Classes

S3 offers multiple tiers optimized for **access frequency vs cost**. All classes offer the same 11 nines of durability (except One Zone-IA).

| Storage Class                     | Use Case                                 | Retrieval Time   | Min Storage Duration | Cost (approx / GB / month) |
| --------------------------------- | ---------------------------------------- | ---------------- | -------------------- | -------------------------- |
| **S3 Standard**                   | Frequently accessed data                 | Milliseconds     | None                 | $0.023                     |
| **S3 Intelligent-Tiering**        | Unknown/changing access patterns         | Milliseconds     | None                 | Auto-tiers                 |
| **S3 Standard-IA**                | Infrequent access, needs quick retrieval | Milliseconds     | 30 days              | $0.0125                    |
| **S3 One Zone-IA**                | Re-creatable, infrequent (single AZ)     | Milliseconds     | 30 days              | $0.01                      |
| **S3 Glacier Instant Retrieval**  | Archive, occasional access               | Milliseconds     | 90 days              | $0.004                     |
| **S3 Glacier Flexible Retrieval** | Archive, occasional restore              | Minutes to hours | 90 days              | $0.0036                    |
| **S3 Glacier Deep Archive**       | Long-term archival (7-10 yrs)            | 12–48 hours      | 180 days             | $0.00099                   |

### Lifecycle Policy Example

Automatically transition logs: Standard → IA (30 days) → Glacier (90 days) → Delete (7 years).

```json
{
  "Rules": [{
    "ID": "log-lifecycle",
    "Status": "Enabled",
    "Filter": { "Prefix": "logs/" },
    "Transitions": [
      { "Days": 30,  "StorageClass": "STANDARD_IA" },
      { "Days": 90,  "StorageClass": "GLACIER" }
    ],
    "Expiration": { "Days": 2555 }
  }]
}
```

***

## 9. What is an S3 Bucket Policy?

An **S3 Bucket Policy** is a **JSON-based resource policy** attached directly to a bucket that defines **who (Principal)** can perform **which actions (Action)** on **which resources (Resource)** under **what conditions (Condition)**.

### Bucket Policy vs IAM Policy vs ACL

| Feature       | Bucket Policy     | IAM Policy           | ACL              |
| ------------- | ----------------- | -------------------- | ---------------- |
| Attached to   | Bucket (resource) | User/Role (identity) | Bucket/Object    |
| Format        | JSON              | JSON                 | XML (legacy)     |
| Cross-account | ✅ Yes             | With trust           | ✅ Yes            |
| Recommended   | ✅ Primary         | ✅ Primary            | ❌ Legacy — avoid |

### Example 1 — Allow cross-account read access

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowCrossAccountRead",
    "Effect": "Allow",
    "Principal": { "AWS": "arn:aws:iam::123456789012:role/AnalyticsRole" },
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::my-data-bucket",
      "arn:aws:s3:::my-data-bucket/*"
    ]
  }]
}
```

### Example 2 — Enforce HTTPS only (security best practice)

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyInsecureTransport",
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:*",
    "Resource": [
      "arn:aws:s3:::my-bucket",
      "arn:aws:s3:::my-bucket/*"
    ],
    "Condition": { "Bool": { "aws:SecureTransport": "false" } }
  }]
}
```

### Example 3 — Enforce encryption at upload

```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:PutObject",
  "Resource": "arn:aws:s3:::my-bucket/*",
  "Condition": {
    "StringNotEquals": { "s3:x-amz-server-side-encryption": "AES256" }
  }
}
```

***

## 10. How Do You Make an S3 Object Public — and Why is That Risky?

### Ways to Make an Object Public

1. **Object ACL** (legacy):
   ```bash
   aws s3api put-object-acl --bucket my-bucket --key file.jpg --acl public-read
   ```
2. **Bucket Policy** allowing anonymous read:
   ```json
   {
     "Effect": "Allow",
     "Principal": "*",
     "Action": "s3:GetObject",
     "Resource": "arn:aws:s3:::my-bucket/*"
   }
   ```
3. **Disable "Block Public Access"** setting at bucket/account level (required for any of the above to take effect since 2018).
4. **Presigned URLs** — the *safe* alternative: generate a temporary signed URL for anonymous access without making the object public.
   ```bash
   aws s3 presign s3://my-bucket/file.pdf --expires-in 3600
   ```

### Why It's Risky ⚠️

| Risk                           | Impact                                                                                                                                                                     |
| ------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Data leakage**               | Sensitive PII, credentials, source code exposed to the internet — one of the most common cloud breaches (Capital One, Accenture, Verizon, US DoD all had public S3 leaks). |
| **Bill shock**                 | Public objects can be scraped/downloaded en masse — you pay egress fees ($0.09/GB). Attackers have used public buckets for **denial-of-wallet** attacks.                   |
| **Compliance violations**      | GDPR, HIPAA, PCI-DSS, SOC2 all prohibit unencrypted/unauthenticated public data. Fines can run into millions.                                                              |
| **Malware distribution**       | Attackers upload malware to writable public buckets and use your domain as a trusted CDN.                                                                                  |
| **Search engine indexing**     | Google indexes public buckets; `site:s3.amazonaws.com` searches expose them.                                                                                               |
| **Bucket enumeration attacks** | Tools like `s3scanner`, `Grayhat Warfare` scan for public buckets 24/7.                                                                                                    |

### Best Practices to Avoid the Risk

1. ✅ **Enable "Block Public Access"** at account level — the safest default.
2. ✅ Use **CloudFront + OAC (Origin Access Control)** to serve public content — bucket stays private.
3. ✅ Use **Presigned URLs** for temporary sharing.
4. ✅ Enable **S3 Access Analyzer** in IAM to detect unintended public access.
5. ✅ Enable **AWS Config rule** `s3-bucket-public-read-prohibited`.
6. ✅ Turn on **CloudTrail data events** + **GuardDuty S3 Protection** for anomaly detection.
7. ✅ Enforce **encryption (SSE-S3/KMS)** and **HTTPS-only** via bucket policy.

***

## Summary Cheat Sheet

| Concept               | One-liner                                               |
| --------------------- | ------------------------------------------------------- |
| **AWS**               | Global pay-as-you-go cloud with 200+ services           |
| **EC2**               | Virtual servers in the cloud                            |
| **Instance Families** | M=balanced, C=CPU, R=RAM, I=storage, P/G=GPU            |
| **AMI**               | Blueprint (OS + apps) used to launch EC2                |
| **EBS**               | Persistent block storage, AZ-scoped                     |
| **S3**                | Object storage, 11-nines durability, regional           |
| **Storage Classes**   | Standard → IA → Glacier → Deep Archive                  |
| **Bucket Policy**     | JSON resource policy for cross-account/anonymous access |
| **Public S3**         | Convenient but historically the #1 cloud breach source  |

***

# AWS IAM & VPC Fundamentals – Detailed Interview-Ready Answers (Q11–Q20)

Here's a comprehensive walkthrough of your next 10 questions covering **AWS Identity & Access Management (IAM)** and **Virtual Private Cloud (VPC)** — structured for a **Senior DevOps interview** with practical examples, JSON policies, CLI snippets, and real-world scenarios.

***

## 11. What is IAM and What Are Its Core Components?

**AWS IAM (Identity and Access Management)** is a **free, global service** that controls **who is authenticated (signed in)** and **authorized (has permissions)** to access AWS resources. It's the foundation of AWS security and typically the **first service you configure** in any account.

### Core Components

| Component                           | Description                                                                    | Example                       |
| ----------------------------------- | ------------------------------------------------------------------------------ | ----------------------------- |
| **Users**                           | Long-term identity for a human or app with credentials (password/access keys). | `rajesh.singh`                |
| **Groups**                          | Collection of users sharing the same policies.                                 | `DevOpsEngineers` group       |
| **Roles**                           | Temporary identity assumed by users, services, or external accounts via STS.   | `EC2-S3-ReadRole`             |
| **Policies**                        | JSON documents defining permissions (Allow/Deny).                              | `AmazonS3ReadOnlyAccess`      |
| **Identity Providers (IdP)**        | Federate external identities via SAML/OIDC.                                    | Azure AD, Okta, Google        |
| **Access Keys**                     | Programmatic credentials (Access Key ID + Secret) for CLI/SDK.                 | `AKIA…`                       |
| **MFA Devices**                     | Extra security layer (virtual, hardware U2F).                                  | Google Authenticator, YubiKey |
| **Root User**                       | Account owner — unrestricted access; **should never be used** daily.           | Account email login           |
| **Permission Boundaries**           | Guardrail limiting max permissions a role/user can have.                       | Cap developer permissions     |
| **Service Control Policies (SCPs)** | Org-level policies restricting entire accounts (AWS Organizations).            | Deny region access org-wide   |
| **Instance Profiles**               | Container that passes a role to an EC2 instance.                               | Attach role to EC2            |
| **STS (Security Token Service)**    | Issues temporary credentials for role assumption.                              | `sts:AssumeRole`              |

### Key Characteristics

* **Global service** (not region-scoped) — one IAM per account.
* **Free to use** — no cost for users, roles, or policies.
* **Deny-by-default** — no permissions unless explicitly granted.
* **Explicit Deny always wins** over Allow.
* Supports **JSON policy language** with `Effect`, `Action`, `Resource`, `Condition`, `Principal`.

### IAM Policy Anatomy

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowS3Read",
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::my-bucket",
      "arn:aws:s3:::my-bucket/*"
    ],
    "Condition": {
      "IpAddress": { "aws:SourceIp": "203.0.113.0/24" }
    }
  }]
}
```

### Best Practices Snapshot

1. ✅ Enable **MFA** on root and all users.
2. ✅ Use **roles**, not access keys, for EC2/Lambda/ECS.
3. ✅ Follow **least privilege** — start minimal, expand as needed.
4. ✅ Use **AWS IAM Identity Center (SSO)** for human access, not IAM users.
5. ✅ Rotate access keys every 90 days; audit via **IAM Credential Report**.
6. ✅ Enable **CloudTrail** to log all IAM activity.
7. ✅ Use **IAM Access Analyzer** to detect unintended external access.

***

## 12. IAM User vs IAM Role

Both are **identities**, but fundamentally different in **how credentials work** and **when to use them**.

### Comparison Table

| Feature                   | IAM User                                      | IAM Role                                            |
| ------------------------- | --------------------------------------------- | --------------------------------------------------- |
| **Purpose**               | Long-term identity for a human or application | Temporary identity assumed on-demand                |
| **Credentials**           | Permanent (username/password + access keys)   | Temporary (STS-issued, auto-rotated, 15 min–12 hrs) |
| **Who uses it**           | Person, on-prem app, legacy tool              | EC2, Lambda, ECS, cross-account, federated users    |
| **Assumed via**           | Direct login / access key                     | `sts:AssumeRole` API call                           |
| **Trust Policy Required** | ❌ No                                          | ✅ Yes (defines *who can assume* the role)           |
| **Password/MFA**          | Yes                                           | No (temporary tokens)                               |
| **Security Risk**         | Higher — keys can leak in Git/logs            | Lower — short-lived credentials                     |
| **Best for**              | Break-glass admin, legacy sync                | Modern workloads, cross-account, federation         |

### Example — IAM Role for EC2 (recommended pattern)

**Trust Policy** (who can assume):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "ec2.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
```

**Permission Policy** (what the role can do):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::my-app-config",
      "arn:aws:s3:::my-app-config/*"
    ]
  }]
}
```

Attach the role to EC2 via **Instance Profile** → the SDK/CLI on the instance automatically retrieves temp credentials from **IMDSv2** (`http://169.254.169.254/latest/meta-data/iam/security-credentials/`).

### When to Use Which

| Scenario                            | Use                                             |
| ----------------------------------- | ----------------------------------------------- |
| Human logs into Console daily       | IAM Identity Center (SSO) — not plain IAM users |
| EC2 needs to read from S3           | **IAM Role** (instance profile)                 |
| Lambda writes to DynamoDB           | **IAM Role** (execution role)                   |
| GitHub Actions deploys to AWS       | **IAM Role** via OIDC federation                |
| On-prem legacy script uploads to S3 | IAM User with rotated access keys               |
| Cross-account access                | **IAM Role** with trust policy                  |
| Kubernetes pod needs AWS access     | **IAM Role for Service Accounts (IRSA)** on EKS |

### Real-World DevOps Example

For your **GitHub → AWS deployment pipeline**, use **OIDC federation** instead of storing long-lived AWS keys in GitHub secrets:

```json
{
  "Effect": "Allow",
  "Principal": {
    "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
  },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:sub": "repo:my-org/my-repo:ref:refs/heads/main"
    }
  }
}
```

***

## 13. IAM Policies vs Permission Boundaries

Both are JSON documents, but serve **different purposes** in the permission evaluation flow.

### IAM Policies (Standard)

**Grants** permissions to identities or resources.

| Type                                | Attached To                              | Purpose                             |
| ----------------------------------- | ---------------------------------------- | ----------------------------------- |
| **Identity-based (Managed/Inline)** | User, Group, Role                        | Says *what the identity CAN do*     |
| **Resource-based**                  | S3 bucket, SQS, KMS key, Lambda          | Says *who CAN access this resource* |
| **AWS Managed**                     | Pre-built by AWS (`AdministratorAccess`) | Common permission sets              |
| **Customer Managed**                | Your own reusable policies               | Company-specific access patterns    |
| **Inline**                          | Embedded directly in one user/role       | One-off, non-reusable               |

### Permission Boundaries

A **guardrail** — defines the **maximum permissions** an IAM user or role can have, regardless of what identity policies grant. **Boundaries don't grant permissions** — they only limit.

### Key Differences

| Aspect                   | IAM Policy                      | Permission Boundary                  |
| ------------------------ | ------------------------------- | ------------------------------------ |
| **Purpose**              | Grants permissions              | Sets maximum permissions ceiling     |
| **Grants access?**       | ✅ Yes                           | ❌ No — only restricts                |
| **Attached to**          | Users, groups, roles, resources | Users, roles (only)                  |
| **Effect**               | Additive                        | Subtractive (upper limit)            |
| **Common use**           | Day-to-day access               | Delegated admin, developer sandboxes |
| **Multiple attachments** | Many policies per identity      | Only ONE boundary per identity       |

### Effective Permissions = **Identity Policy ∩ Permission Boundary**

If a user's identity policy grants `s3:*` but the boundary only allows `s3:GetObject`, the effective permission is **only `s3:GetObject`**.

### Real-World Example — Developer Self-Service

**Scenario:** Let junior devs create IAM roles for Lambda functions, but prevent them from creating admin roles.

**Permission Boundary** (attached to any role dev creates):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "s3:GetObject",
      "dynamodb:*",
      "logs:*"
    ],
    "Resource": "*"
  }]
}
```

**Dev's IAM Policy** — allows creating roles ONLY if boundary is attached:

```json
{
  "Effect": "Allow",
  "Action": "iam:CreateRole",
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "iam:PermissionsBoundary": "arn:aws:iam::123456789012:policy/DevBoundary"
    }
  }
}
```

Even if a dev tries to attach `AdministratorAccess` to their new role, the **boundary caps** it to S3/DynamoDB/Logs only.

### Policy Evaluation Order

```
1. Explicit Deny anywhere (SCP / Policy / Boundary / Session)  → DENY (final)
2. Organizations SCPs                                           → Must Allow
3. Resource-based policies                                      → Can grant
4. Identity-based policies                                      → Must Allow
5. Permission Boundaries                                        → Must Allow
6. Session policies (for assumed roles)                         → Must Allow
7. Default                                                      → DENY
```

**Rule of thumb:** For a request to succeed, it must be **explicitly allowed at every applicable layer** and **denied at none**.

***

## 14. What is AWS VPC?

**Amazon VPC (Virtual Private Cloud)** is a **logically isolated virtual network** in AWS where you launch resources (EC2, RDS, Lambda, EKS nodes). It gives you **full control over networking** — IP ranges, subnets, routing, gateways, firewalls — just like an on-prem data center network.

### Core Features

* **Isolation** — your VPC is invisible to other AWS customers.
* **Customizable IP range** — define via **CIDR block** (e.g., `10.0.0.0/16` allows \~65k IPs).
* **Region-scoped** — spans all AZs in one region.
* **Free** to create (you pay for gateways, endpoints, data transfer).
* **Default VPC** created in each region on account creation; can also create **custom VPCs**.

### Key VPC Components

| Component                  | Purpose                                                         |
| -------------------------- | --------------------------------------------------------------- |
| **CIDR Block**             | IP address range (primary IPv4, optional IPv6, secondary CIDRs) |
| **Subnets**                | Segments of the CIDR, tied to a single AZ                       |
| **Route Tables**           | Direct traffic between subnets and gateways                     |
| **Internet Gateway (IGW)** | Public internet access                                          |
| **NAT Gateway**            | Outbound-only internet from private subnets                     |
| **Security Groups**        | Instance-level stateful firewall                                |
| **NACLs**                  | Subnet-level stateless firewall                                 |
| **VPC Endpoints**          | Private access to AWS services (S3, DynamoDB) without internet  |
| **VPC Peering**            | Connect two VPCs directly                                       |
| **Transit Gateway**        | Hub for many VPCs / on-prem VPN                                 |
| **DHCP Option Sets**       | Custom DNS, NTP settings                                        |
| **Flow Logs**              | Capture network traffic metadata                                |

### Typical 3-Tier VPC Design

```
VPC: 10.0.0.0/16
├── AZ us-east-1a
│   ├── Public Subnet    10.0.1.0/24  (ALB, NAT GW)
│   ├── App Subnet       10.0.11.0/24 (EC2/EKS)
│   └── DB Subnet        10.0.21.0/24 (RDS)
├── AZ us-east-1b
│   ├── Public Subnet    10.0.2.0/24
│   ├── App Subnet       10.0.12.0/24
│   └── DB Subnet        10.0.22.0/24
└── AZ us-east-1c
    ├── Public Subnet    10.0.3.0/24
    ├── App Subnet       10.0.13.0/24
    └── DB Subnet        10.0.23.0/24
```

### Create VPC via CLI

```bash
aws ec2 create-vpc --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=Prod-VPC}]'
```

### Best Practices

* ✅ Use **/16** for prod VPCs to allow room for growth.
* ✅ Avoid overlapping CIDRs across VPCs/on-prem (needed for peering, TGW).
* ✅ Deploy across **≥2 AZs** for HA.
* ✅ Enable **VPC Flow Logs** for security/audit.
* ✅ Use **Transit Gateway** for multi-VPC/hybrid architectures.

***

## 15. What are Subnets and How are They Used Inside a VPC?

A **subnet** is a **subdivision of a VPC's CIDR block**, tied to a **single Availability Zone**. Subnets are where you place resources like EC2, RDS, and Lambda ENIs.

### Types of Subnets

| Type                | Definition                     | Route to IGW? | Public IP assigned?               | Use Case                                   |
| ------------------- | ------------------------------ | ------------- | --------------------------------- | ------------------------------------------ |
| **Public Subnet**   | Has route `0.0.0.0/0 → IGW`    | ✅ Yes         | ✅ Yes (auto-assign or Elastic IP) | ALB, NAT GW, bastion hosts                 |
| **Private Subnet**  | No IGW route; outbound via NAT | ❌ No          | ❌ No                              | App servers, DBs, internal microservices   |
| **Isolated Subnet** | No IGW, no NAT                 | ❌ No          | ❌ No                              | Highly sensitive DBs, air-gapped workloads |

### Key Facts

* **AZ-scoped** — a subnet lives in exactly one AZ.
* **CIDR must be within the VPC CIDR** (e.g., VPC `10.0.0.0/16` → subnets like `10.0.1.0/24`).
* **AWS reserves 5 IPs** per subnet:
  * `.0` = network
  * `.1` = VPC router
  * `.2` = DNS
  * `.3` = future use
  * `.255` = broadcast
  * So a `/24` (256 IPs) gives **251 usable**.
* Each subnet is associated with **one route table** (main table by default).
* Each subnet has **one NACL** (default allows all).

### Create a Subnet

```bash
aws ec2 create-subnet \
  --vpc-id vpc-0abcdef1234567890 \
  --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Public-1a}]'

# Enable auto-assign public IP for public subnets
aws ec2 modify-subnet-attribute --subnet-id subnet-0abc --map-public-ip-on-launch
```

### Subnet Sizing Guidance

| Subnet Size | Usable IPs | Use For                         |
| ----------- | ---------- | ------------------------------- |
| /28         | 11         | Very small (transit, endpoints) |
| /26         | 59         | Small services                  |
| /24         | 251        | Standard tier                   |
| /22         | 1,019      | EKS/large workloads             |
| /20         | 4,091      | Very large clusters             |

### DevOps Best Practices

* ✅ **One subnet per AZ per tier** (public, app, DB) → minimum 6 subnets across 2 AZs.
* ✅ Size EKS subnets generously — each pod consumes an IP.
* ✅ Never put DBs in public subnets.
* ✅ Tag subnets clearly (`kubernetes.io/role/elb`, `kubernetes.io/role/internal-elb` for EKS).

***

## 16. What is an Internet Gateway (IGW)?

An **Internet Gateway (IGW)** is a **horizontally scaled, redundant, highly available VPC component** that enables **bi-directional communication between resources in the VPC and the public internet**.

### Key Characteristics

* **One IGW per VPC** (1:1 mapping).
* **Free** to create; no bandwidth cost for IGW itself (you pay for data transfer out).
* **Highly available** across all AZs — no HA config needed.
* Performs **NAT for public IPs** — translates public IP ↔ private IP for instances.
* Supports **both IPv4 and IPv6**.

### How It Works

For an EC2 to reach the internet via IGW, three things must be true:

1. ✅ VPC has an IGW attached.
2. ✅ Instance's subnet route table has `0.0.0.0/0 → igw-xxx`.
3. ✅ Instance has a **public IP or Elastic IP**.
4. ✅ Security Group + NACL allow the traffic.

### Create & Attach IGW

```bash
# Create IGW
aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=Prod-IGW}]'

# Attach to VPC
aws ec2 attach-internet-gateway \
  --internet-gateway-id igw-0abc \
  --vpc-id vpc-0xyz

# Add default route in public subnet's route table
aws ec2 create-route \
  --route-table-id rtb-0public \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id igw-0abc
```

### IGW vs Other Gateways

| Gateway                           | Direction                  | Use Case                         |
| --------------------------------- | -------------------------- | -------------------------------- |
| **IGW**                           | Bi-directional to internet | Public web servers, ALBs, NAT GW |
| **Egress-only IGW**               | Outbound-only (IPv6)       | IPv6 private subnets             |
| **NAT Gateway**                   | Outbound-only (IPv4)       | Private subnet internet access   |
| **Virtual Private Gateway (VGW)** | To on-prem via VPN         | Site-to-site VPN                 |
| **Transit Gateway**               | Between VPCs / on-prem     | Hub-and-spoke networking         |

***

## 17. What is a NAT Gateway and When Do You Use It?

A **NAT (Network Address Translation) Gateway** is a **managed AWS service** that enables **instances in private subnets** to initiate **outbound connections to the internet** (or other AWS services) while **preventing inbound connections from the internet**.

### Why You Need It

Private EC2 instances still need to:

* Download OS patches (`yum update`, `apt-get update`).
* Pull Docker images from Docker Hub / ECR public.
* Call third-party APIs (payment gateways, SaaS integrations).
* Send logs/metrics to external monitoring (Datadog, New Relic).

...but you **don't want them reachable from the internet**. NAT GW solves this.

### How It Works

1. Instance in **private subnet** sends packet to `0.0.0.0/0`.
2. Private subnet route table sends it to **NAT Gateway** (in a **public subnet**).
3. NAT GW replaces the source private IP with its **Elastic IP** and forwards to IGW.
4. Return traffic comes back through NAT GW to the private instance.
5. **Inbound-initiated traffic is blocked** (stateful).

### Deploy NAT Gateway

```bash
# Allocate Elastic IP
aws ec2 allocate-address --domain vpc

# Create NAT GW in public subnet
aws ec2 create-nat-gateway \
  --subnet-id subnet-0public1a \
  --allocation-id eipalloc-0abc \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=NAT-1a}]'

# Update private subnet route table
aws ec2 create-route \
  --route-table-id rtb-0private \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id nat-0abc
```

### NAT Gateway vs NAT Instance

| Feature             | NAT Gateway (managed)             | NAT Instance (EC2-based, legacy) |
| ------------------- | --------------------------------- | -------------------------------- |
| **Management**      | Fully managed by AWS              | You manage the EC2               |
| **Bandwidth**       | Up to 100 Gbps (auto-scales)      | Limited by instance type         |
| **HA**              | Highly available in one AZ        | Manual HA config                 |
| **Cost**            | \~$0.045/hr + $0.045/GB processed | EC2 cost only                    |
| **Security Group**  | ❌ Not supported                   | ✅ Supported                      |
| **Port Forwarding** | ❌ No                              | ✅ Yes                            |
| **Recommendation**  | ✅ **Use this**                    | Legacy — avoid                   |

### High Availability Design

* Deploy **one NAT GW per AZ** (not one shared across AZs).
* Each private subnet routes to the NAT GW in **its own AZ**.
* If an AZ fails, only that AZ's outbound traffic is affected.

### Cost Optimization Tips ⚠️

NAT Gateway can be **expensive** ($0.045/GB processed adds up quickly):

1. ✅ Use **VPC Endpoints** (Gateway) for S3 and DynamoDB — **free**, bypasses NAT GW.
2. ✅ Use **Interface Endpoints** for ECR, CloudWatch, SSM, Secrets Manager (\~$0.01/hr + $0.01/GB).
3. ✅ For dev/test, use a **single shared NAT GW** to save cost (sacrificing HA).
4. ✅ Monitor NAT GW `BytesOutToDestination` metric — high traffic often indicates unoptimized data flows.

***

## 18. What is a Security Group?

A **Security Group (SG)** is a **stateful virtual firewall** attached to the **Elastic Network Interface (ENI)** of AWS resources — EC2, RDS, Lambda (in VPC), ELB, EKS pods, etc. It controls **inbound and outbound traffic** at the **instance level**.

### Key Characteristics

* **Stateful** — return traffic is automatically allowed (no need to add outbound rule for allowed inbound).
* **Allow rules only** — you cannot create explicit Deny rules.
* **Deny-by-default** — everything blocked unless explicitly allowed.
* **Multiple SGs per ENI** — up to 5 (soft limit 16).
* **Referenced by ID** — you can reference another SG as source (`sg-abc123` allows all instances with that SG).
* Applied to **the instance/ENI**, not the subnet.
* Changes take effect **immediately**.

### Default Behavior

* **Default SG**: Allows all inbound from same SG, all outbound to anywhere.
* **Custom SG**: No inbound rules, all outbound allowed.

### Rule Anatomy

| Field                  | Example                                          |
| ---------------------- | ------------------------------------------------ |
| **Type**               | HTTP, SSH, MySQL, Custom TCP                     |
| **Protocol**           | TCP, UDP, ICMP                                   |
| **Port Range**         | 80, 443, 22, 3306, 1024-65535                    |
| **Source/Destination** | CIDR (`10.0.0.0/8`), SG (`sg-0abc`), Prefix List |

### Example — Three-tier App SGs

**ALB Security Group:**

```
Inbound:  443 from 0.0.0.0/0
Outbound: All to app-sg
```

**App Server Security Group:**

```
Inbound:  8080 from alb-sg
          22 from bastion-sg
Outbound: 3306 to db-sg
          443 to 0.0.0.0/0 (patches)
```

**Database Security Group:**

```
Inbound:  3306 from app-sg
Outbound: (none needed — stateful)
```

### CLI Example

```bash
# Create SG
aws ec2 create-security-group \
  --group-name web-sg \
  --description "Web tier SG" \
  --vpc-id vpc-0abc

# Add ingress rule
aws ec2 authorize-security-group-ingress \
  --group-id sg-0web \
  --protocol tcp --port 443 --cidr 0.0.0.0/0

# Reference another SG
aws ec2 authorize-security-group-ingress \
  --group-id sg-0db \
  --protocol tcp --port 3306 \
  --source-group sg-0app
```

### Best Practices

* ✅ Reference **SG IDs** instead of CIDRs where possible — auto-scales with membership.
* ✅ Never use `0.0.0.0/0` on port 22 or 3389 — use a bastion or **SSM Session Manager**.
* ✅ One SG per tier/role for clarity (`web-sg`, `app-sg`, `db-sg`).
* ✅ Use **descriptive tags** and rule descriptions.
* ✅ Audit unused SGs regularly.

***

## 19. What is a Network ACL (NACL) and How Does It Differ from Security Groups?

A **Network ACL (NACL)** is a **stateless firewall** at the **subnet level** that controls inbound and outbound traffic **entering or leaving a subnet**. It's an optional additional layer of defense.

### Key Characteristics

* **Stateless** — return traffic must be explicitly allowed by a separate rule.
* **Supports Allow AND Deny rules** — unlike SGs.
* **Rules processed in numbered order** (lowest first); first match wins.
* **Default NACL**: Allows all inbound + outbound.
* **Custom NACL**: Denies all inbound + outbound until you add rules.
* **One NACL per subnet** (but a NACL can be attached to multiple subnets).
* **Rule number range**: 1 – 32,766 (best practice: increments of 10 or 100 for insertion room).

### Rule Anatomy

| Field       | Example   |
| ----------- | --------- |
| Rule #      | 100       |
| Type        | HTTP      |
| Protocol    | TCP (6)   |
| Port        | 80        |
| Source/Dest | 0.0.0.0/0 |
| Allow/Deny  | ALLOW     |

### Security Group vs NACL — Full Comparison

| Feature              | Security Group                       | Network ACL                                        |
| -------------------- | ------------------------------------ | -------------------------------------------------- |
| **Scope**            | Instance/ENI level                   | Subnet level                                       |
| **State**            | Stateful                             | Stateless                                          |
| **Rules Supported**  | Allow only                           | Allow + Deny                                       |
| **Rule Evaluation**  | All rules evaluated                  | Rules evaluated in order (lowest first)            |
| **Default Behavior** | Deny all inbound, allow all outbound | Default NACL allows all; custom denies all         |
| **Return Traffic**   | Auto-allowed                         | Must explicitly allow ephemeral ports (1024-65535) |
| **Apply To**         | Instances (ENIs)                     | Subnets                                            |
| **Reference SGs**    | ✅ Yes                                | ❌ No — CIDRs only                                  |
| **Max Rules**        | 60 inbound + 60 outbound             | 20 per direction (soft limit 40)                   |
| **Use Case**         | Primary firewall — always used       | Extra defense, block bad IPs subnet-wide           |

### Ephemeral Ports Gotcha (Stateless)

If your NACL allows inbound HTTP (port 80), you **must also allow outbound ephemeral ports** (1024-65535) for the reply to leave. This is a common troubleshooting issue.

Example web-server NACL:

```
INBOUND
  100  Allow  TCP  80        0.0.0.0/0   ALLOW
  110  Allow  TCP  443       0.0.0.0/0   ALLOW
  120  Allow  TCP  1024-65535 0.0.0.0/0  ALLOW  # Ephemeral for responses to outbound

OUTBOUND
  100  Allow  TCP  80        0.0.0.0/0   ALLOW  # Patch downloads
  110  Allow  TCP  443       0.0.0.0/0   ALLOW
  120  Allow  TCP  1024-65535 0.0.0.0/0  ALLOW  # Ephemeral for inbound responses
```

### When to Use NACLs

* ✅ **Block malicious IP ranges** at the subnet edge (SGs can't Deny).
* ✅ **Compliance requirements** for defense-in-depth.
* ✅ **Protect DB subnets** — deny all except from app subnet CIDR.
* ✅ **DDoS mitigation** for known bad actors.

### Best Practices

* ✅ Use **SGs as primary** firewall; NACLs as **defense-in-depth**.
* ✅ Number rules in increments of 100 to allow insertion.
* ✅ Don't forget **ephemeral ports** for stateless return traffic.
* ✅ Document rules clearly — NACL misconfigs are hard to debug.

***

## 20. What is a Route Table in VPC Networking?

A **Route Table** contains a set of **rules (routes)** that determine **where network traffic from a subnet or gateway is directed**. Every subnet must be associated with a route table.

### Key Characteristics

* **VPC-scoped** — belongs to one VPC.
* **Main route table** — created automatically with the VPC; used by any subnet not explicitly associated.
* **Custom route tables** — you create for public/private/DB subnets.
* **One route table per subnet**, but one route table can serve **many subnets**.
* **Longest prefix match wins** — more specific routes take precedence.
* **Local route** (`10.0.0.0/16 → local`) is always present and **cannot be deleted** — enables intra-VPC communication.

### Route Anatomy

| Destination      | Target   | Meaning                                |
| ---------------- | -------- | -------------------------------------- |
| `10.0.0.0/16`    | local    | Intra-VPC traffic (default, immutable) |
| `0.0.0.0/0`      | igw-xxx  | Default route to internet              |
| `0.0.0.0/0`      | nat-xxx  | Outbound via NAT                       |
| `172.16.0.0/16`  | pcx-xxx  | To peered VPC                          |
| `192.168.0.0/16` | vgw-xxx  | To on-prem via VPN                     |
| `10.100.0.0/16`  | tgw-xxx  | Via Transit Gateway                    |
| `pl-xxx` (S3)    | vpce-xxx | VPC Gateway Endpoint for S3            |

### Public vs Private Subnet Route Tables

**Public Route Table:**

```
10.0.0.0/16   → local
0.0.0.0/0     → igw-0abc      ← THIS makes the subnet "public"
```

**Private Route Table:**

```
10.0.0.0/16   → local
0.0.0.0/0     → nat-0abc      ← Outbound-only via NAT
pl-63a5400a   → vpce-0abc     ← S3 Gateway Endpoint (free!)
```

**Isolated Route Table:**

```
10.0.0.0/16   → local          ← ONLY intra-VPC, no internet
```

### CLI Example

```bash
# Create custom route table
aws ec2 create-route-table --vpc-id vpc-0abc \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=Public-RT}]'

# Add internet route
aws ec2 create-route \
  --route-table-id rtb-0public \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id igw-0abc

# Associate with subnet
aws ec2 associate-route-table \
  --route-table-id rtb-0public \
  --subnet-id subnet-0abc
```

### Advanced Concepts

**Route Priority (longest prefix match):**

```
0.0.0.0/0        → nat-0abc
10.20.0.0/16     → pcx-0xyz    ← Wins for traffic to 10.20.x.x (more specific)
```

**Route Propagation** — VGWs and TGWs can auto-propagate on-prem routes into the table.

**Edge Association** — Route tables can be associated with an **IGW** (edge routing) to direct inbound traffic to a firewall/appliance before it reaches subnets.

### Real-World Design Tip

For **hub-and-spoke networks with Transit Gateway**:

```
Spoke VPC Private RT:
  10.0.0.0/16    → local
  10.0.0.0/8     → tgw-xxx    ← All internal → TGW
  0.0.0.0/0      → tgw-xxx    ← Internet via central egress VPC
```

This centralizes egress inspection through a firewall in the hub VPC.

***

## Summary Cheat Sheet (Q11–Q20)

| #  | Concept                | One-Liner                                                        |
| -- | ---------------------- | ---------------------------------------------------------------- |
| 11 | **IAM**                | Global service for authN + authZ; users, groups, roles, policies |
| 12 | **User vs Role**       | User = permanent creds; Role = temporary, assumed via STS        |
| 13 | **Policy vs Boundary** | Policy grants; Boundary caps max permissions                     |
| 14 | **VPC**                | Isolated virtual network with your own IP range                  |
| 15 | **Subnets**            | AZ-scoped CIDR partitions; public (IGW route) vs private (NAT)   |
| 16 | **Internet Gateway**   | Bi-directional public internet access for VPC                    |
| 17 | **NAT Gateway**        | Managed outbound-only internet for private subnets               |
| 18 | **Security Group**     | Stateful, instance-level firewall, Allow-only                    |
| 19 | **NACL**               | Stateless, subnet-level firewall, Allow + Deny                   |
| 20 | **Route Table**        | Rules directing traffic from subnets to gateways                 |

***

## Bonus: Common Interview Scenarios

**Q:** *"You launched an EC2 in a public subnet with a public IP and SG allows port 80, but you can't reach it. Why?"*
**A:** Check → (1) Route table has `0.0.0.0/0 → IGW`, (2) NACL allows port 80 + ephemeral outbound, (3) OS-level firewall (iptables/Windows firewall), (4) Web server is actually listening (`netstat`).

**Q:** *"Your private EC2 can't `yum update`. Fix?"*
**A:** (1) NAT GW exists in public subnet with EIP, (2) Private subnet route `0.0.0.0/0 → nat-xxx`, (3) SG allows outbound 443, (4) NACL allows outbound 443 + inbound ephemeral, (5) NAT GW's subnet has route to IGW.

**Q:** *"Explain how to give an EC2 read access to S3 without embedding keys."*
**A:** Create IAM Role with `s3:GetObject` policy + `ec2.amazonaws.com` trust policy → attach via Instance Profile → SDK auto-retrieves temp creds from IMDSv2.

***

Would you like me to continue with the **next batch (Q21–Q30)** covering **RDS, DynamoDB, Lambda, CloudFront, Route 53**, or should I dive deeper into any topic above with **hands-on Terraform code, EKS-specific networking, or scenario-based troubleshooting drills**?

Given your Senior DevOps interview prep, I can also compile a **500+ categorized AWS interview question bank** (similar to your Azure/Kubernetes/Terraform sets) as a downloadable document. 🚀

# AWS Managed Services & Serverless – Detailed Interview-Ready Answers (Q21–Q30)

Here's a comprehensive walkthrough of your next 10 questions covering **RDS, Lambda, API Gateway, CloudFormation, and CloudWatch** — structured for a **Senior DevOps interview** with practical examples, YAML/JSON templates, CLI snippets, and real-world scenarios.

***

## 21. What is AWS RDS and Which Database Engines Does It Support?

**Amazon RDS (Relational Database Service)** is a **fully managed relational database service** that handles provisioning, patching, backups, replication, failover, and monitoring — so you focus on schema design and queries, not DB admin tasks.

### What AWS Manages vs What You Manage

| AWS Manages                       | You Manage                      |
| --------------------------------- | ------------------------------- |
| OS patching, DB engine updates    | Schema design, queries, indexes |
| Automated backups (up to 35 days) | Application connection strings  |
| Multi-AZ failover                 | User accounts, grants           |
| Read replicas provisioning        | Parameter groups tuning         |
| Storage autoscaling               | Performance tuning              |
| Encryption at rest (KMS)          | Security groups, IAM auth       |

### Supported Database Engines

| Engine                                          | Notes                                                                  | Use Case                        |
| ----------------------------------------------- | ---------------------------------------------------------------------- | ------------------------------- |
| **Amazon Aurora** (MySQL/PostgreSQL compatible) | AWS-native, 5× MySQL perf, 3× Postgres, auto-scaling storage to 128 TB | Cloud-native prod workloads     |
| **MySQL**                                       | Community edition, versions 5.7 / 8.0+                                 | Web apps, WordPress, LAMP stack |
| **PostgreSQL**                                  | Versions 12–16, with extensions (PostGIS, pgvector)                    | Complex apps, GIS, analytics    |
| **MariaDB**                                     | MySQL fork, versions 10.x                                              | Drop-in MySQL replacement       |
| **Oracle**                                      | SE2, EE — BYOL or license-included                                     | Enterprise migrations           |
| **Microsoft SQL Server**                        | Web, Standard, Enterprise editions                                     | .NET / Windows apps             |
| **IBM Db2**                                     | LUW edition (newer addition)                                           | Legacy IBM migrations           |

### RDS Deployment Options

| Option                   | Purpose                                           |
| ------------------------ | ------------------------------------------------- |
| **Single-AZ**            | Dev/test — one instance, one AZ                   |
| **Multi-AZ**             | Prod HA — sync standby in another AZ              |
| **Multi-AZ Cluster**     | 1 writer + 2 readable standbys (MySQL/PostgreSQL) |
| **Read Replicas**        | Scale reads; async replication; up to 15          |
| **Aurora Serverless v2** | Auto-scaling capacity (0.5–256 ACUs)              |
| **RDS Custom**           | For Oracle/SQL Server needing OS/DB access        |
| **RDS on Outposts**      | On-prem RDS                                       |

### Create RDS via CLI

```bash
aws rds create-db-instance \
  --db-instance-identifier prod-mysql \
  --db-instance-class db.r6g.large \
  --engine mysql --engine-version 8.0.35 \
  --master-username admin --master-user-password 'SecurePass123!' \
  --allocated-storage 100 --storage-type gp3 \
  --multi-az --storage-encrypted \
  --backup-retention-period 7 \
  --vpc-security-group-ids sg-0db \
  --db-subnet-group-name prod-db-subnet-group
```

### RDS vs Aurora vs Self-Managed

| Feature      | RDS       | Aurora              | EC2 self-managed  |
| ------------ | --------- | ------------------- | ----------------- |
| **Ease**     | High      | Highest             | Low               |
| **Cost**     | $         | \$$                 | $ (but ops-heavy) |
| **Storage**  | 64 TB max | 128 TB auto-scaling | Anything          |
| **Perf**     | Good      | 5× MySQL            | Depends on tuning |
| **Failover** | 60–120s   | \~30s               | Manual            |

### Best Practices

* ✅ Enable **Multi-AZ** for prod.
* ✅ Use **Aurora** for new workloads (better perf + features).
* ✅ Store credentials in **Secrets Manager** with rotation.
* ✅ Use **IAM database authentication** where possible.
* ✅ Enable **Performance Insights** and **Enhanced Monitoring**.
* ✅ Never expose RDS to `0.0.0.0/0` — put in private subnets only.

***

## 22. What is a Read Replica in RDS?

A **Read Replica** is a **read-only copy** of your primary RDS database, kept in sync via **asynchronous replication**. Used to **scale read-heavy workloads** and offload traffic from the primary.

### Key Characteristics

* **Asynchronous** replication → some lag (usually milliseconds to seconds).
* **Read-only** — apps can query but not write.
* Up to **15 read replicas** per source (Aurora), **5** for MySQL/PostgreSQL/MariaDB.
* Can be in **same AZ, different AZ, or different Region** (Cross-Region Read Replicas).
* Each replica has its **own endpoint**.
* Can be **promoted to a standalone writable DB** (used for migrations, splits, DR).
* Available for MySQL, PostgreSQL, MariaDB, Oracle (12.1+), SQL Server (2016+ EE), Aurora.

### Use Cases

| Use Case                         | How It Helps                                              |
| -------------------------------- | --------------------------------------------------------- |
| **Read scaling**                 | Route SELECT queries to replicas                          |
| **Analytics/reporting**          | Heavy queries don't impact prod OLTP                      |
| **Disaster Recovery**            | Cross-region replica → promote in DR event                |
| **Zero-downtime major upgrades** | Upgrade replica → promote → cutover                       |
| **Data locality**                | Cross-region replica for global users (low latency reads) |
| **Migration**                    | Promote replica to new region/account                     |

### Create a Read Replica

```bash
aws rds create-db-instance-read-replica \
  --db-instance-identifier prod-mysql-read-1 \
  --source-db-instance-identifier prod-mysql \
  --db-instance-class db.r6g.large \
  --availability-zone us-east-1b
```

### Application Routing Pattern

```python
# Django example
DATABASES = {
    'default': { 'HOST': 'prod-mysql.abc.rds.amazonaws.com' },      # Writes
    'replica': { 'HOST': 'prod-mysql-read-1.abc.rds.amazonaws.com' } # Reads
}
```

### Read Replica vs Multi-AZ (critical interview distinction!)

| Aspect                  | Read Replica             | Multi-AZ Standby               |
| ----------------------- | ------------------------ | ------------------------------ |
| **Purpose**             | Read scaling             | High Availability              |
| **Replication**         | Asynchronous             | Synchronous                    |
| **Readable?**           | ✅ Yes                    | ❌ No (except Multi-AZ Cluster) |
| **Automatic failover?** | ❌ Manual promotion       | ✅ Automatic (60–120s)          |
| **Same Region?**        | Same or cross-region     | Same region only               |
| **Data loss risk**      | Possible (async lag)     | Zero (sync)                    |
| **Cost**                | Additional full instance | Standby costs same as primary  |

### Best Practices

* ✅ Monitor `ReplicaLag` in CloudWatch — alarm if > 30s.
* ✅ Don't rely on replicas for **strong consistency** reads (async lag).
* ✅ Use **Aurora Reader Endpoint** (auto load-balances across replicas) instead of picking one.
* ✅ Use Cross-Region Read Replicas for DR + global apps.

***

## 23. What is a Multi-AZ RDS Deployment?

**Multi-AZ** is RDS's **High Availability feature** where AWS maintains a **synchronous standby replica** in a **different Availability Zone**, with **automatic failover** on primary failure.

### How It Works

1. **Primary** DB in AZ-1a handles all reads + writes.
2. **Standby** in AZ-1b receives **synchronous writes** (data guaranteed identical).
3. Standby is **not readable** — pure HA (not scale-out).
4. Uses a **single DNS endpoint** — apps don't change connection strings on failover.
5. On failure, AWS flips DNS to standby in **60–120 seconds** (typically \~35s for Aurora).

### What Triggers Failover?

* Primary instance failure or hardware issue.
* AZ outage.
* DB engine crash.
* Storage failure.
* Manual failover (`reboot --force-failover`) — for testing / OS patching.
* Instance type modification.

### Multi-AZ Instance vs Multi-AZ Cluster

| Feature               | Multi-AZ Instance (classic)                    | Multi-AZ Cluster (newer)  |
| --------------------- | ---------------------------------------------- | ------------------------- |
| **Standbys**          | 1 (not readable)                               | 2 (both readable)         |
| **Replication**       | Synchronous block-level                        | Semi-synchronous          |
| **Failover**          | 60–120s                                        | \~35s                     |
| **Reads on standby?** | ❌ No                                           | ✅ Yes                     |
| **Engines**           | MySQL, PostgreSQL, MariaDB, Oracle, SQL Server | MySQL, PostgreSQL only    |
| **Use Case**          | Traditional HA                                 | HA + limited read scaling |

### Enable Multi-AZ

```bash
# On creation
aws rds create-db-instance ... --multi-az

# Convert existing
aws rds modify-db-instance \
  --db-instance-identifier prod-mysql \
  --multi-az \
  --apply-immediately
```

### Failover Behavior

* **Endpoint DNS TTL** = 5s → apps re-resolve quickly.
* Sessions get disconnected — apps need **connection retry logic** and **connection pooling** (RDS Proxy helps).
* No data loss (synchronous).
* No manual intervention required.

### Cost

* **Doubles compute cost** (you pay for standby too).
* Storage is not duplicated in billing.

### Real-World DevOps Example

For a critical **e-commerce order DB**:

* ✅ Multi-AZ deployment (HA).
* ✅ 2 Read Replicas (read scaling for product catalog).
* ✅ Cross-Region Read Replica (DR to us-west-2).
* ✅ RDS Proxy in front for connection pooling.
* ✅ Automated backups (35 days) + manual snapshots before releases.

***

## 24. What is Amazon Lambda?

**AWS Lambda** is a **serverless compute service** that runs your code in response to **events** without provisioning or managing servers. You upload code → Lambda handles everything: compute, scaling, patching, HA.

### Key Characteristics

* **Serverless** — no servers to manage.
* **Event-driven** — triggered by 200+ AWS services or HTTP requests.
* **Auto-scaling** — from 0 to thousands of concurrent executions in seconds.
* **Pay-per-use** — billed per ms of execution (no idle cost).
* **Stateless** — no persistent local state (use S3/DynamoDB for state).
* **Ephemeral** — max 15 min execution timeout.
* **Runtime support** — Python, Node.js, Java, .NET, Go, Ruby, custom runtimes (via container image).

### Lambda Function Anatomy

| Component                 | Description                                  |
| ------------------------- | -------------------------------------------- |
| **Handler**               | Entry point (e.g., `index.handler`)          |
| **Runtime**               | `python3.12`, `nodejs20.x`, `java21`, etc.   |
| **Memory**                | 128 MB – 10,240 MB (CPU scales with memory)  |
| **Timeout**               | 1 sec – 15 min (900s max)                    |
| **Ephemeral storage**     | `/tmp` — 512 MB to 10 GB                     |
| **Environment variables** | Config (encrypted via KMS)                   |
| **Execution role**        | IAM role for AWS API access                  |
| **VPC config**            | Optional — attach to VPC subnets             |
| **Layers**                | Reusable code/libraries (up to 5)            |
| **Trigger**               | Event source (S3, API GW, SQS, EventBridge…) |

### Sample Python Lambda

```python
import json
import boto3

s3 = boto3.client('s3')

def lambda_handler(event, context):
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']
    
    # Process the S3 object
    response = s3.get_object(Bucket=bucket, Key=key)
    content = response['Body'].read().decode('utf-8')
    
    return {
        'statusCode': 200,
        'body': json.dumps(f'Processed {key} ({len(content)} bytes)')
    }
```

### Deployment Packaging Options

1. **ZIP file** (up to 50 MB zipped, 250 MB unzipped) — direct upload or from S3.
2. **Container image** (up to 10 GB) — from ECR — great for large ML/data-processing workloads.
3. **Lambda Layers** — shared libraries across functions.

### Concurrency Concepts

* **Unreserved concurrency** — shared account pool (default 1,000 per region).
* **Reserved concurrency** — dedicated slice for critical functions.
* **Provisioned concurrency** — pre-warmed instances (eliminates cold start).

### Cold Start

* First invocation (or after idle) → Lambda spins up a new execution environment → **cold start latency** (100 ms – few seconds).
* Mitigate with: **Provisioned Concurrency**, smaller packages, SnapStart (Java), avoid VPC unless needed.

***

## 25. Typical Use Cases for Lambda

Lambda is best for **short-lived, event-driven, stateless workloads**. It's NOT ideal for long-running batch jobs (>15 min) or workloads needing persistent connections.

### Top 12 Use Cases

| #  | Use Case                        | Example                                                        |
| -- | ------------------------------- | -------------------------------------------------------------- |
| 1  | **REST/HTTP APIs**              | API Gateway + Lambda → serverless backend                      |
| 2  | **S3 event processing**         | Auto-resize images uploaded to S3, virus scan, thumbnails      |
| 3  | **Real-time stream processing** | Kinesis/DynamoDB Streams → transform + load                    |
| 4  | **Scheduled jobs (cron)**       | EventBridge cron → daily reports, cleanup, snapshot management |
| 5  | **Chatbots/voice**              | Alexa Skills, Slack/Teams bots                                 |
| 6  | **Data ETL**                    | Trigger on S3 → transform → load to Redshift/Snowflake         |
| 7  | **IoT backend**                 | AWS IoT → Lambda → DynamoDB                                    |
| 8  | **Auth/authorization**          | Cognito triggers, custom API GW authorizers                    |
| 9  | **Log processing**              | CloudWatch Logs → Lambda → Elasticsearch/Splunk                |
| 10 | **CI/CD automation**            | Auto-tag EC2, notify Slack on deploys, remediate config drift  |
| 11 | **Async task queues**           | SQS → Lambda for background jobs (emails, notifications)       |
| 12 | **Edge computing**              | Lambda\@Edge / CloudFront Functions for request rewrites       |

### Real-World DevOps Examples

**Auto-Tag EC2 Instances on Launch:**

```python
def lambda_handler(event, context):
    ec2 = boto3.client('ec2')
    instance_id = event['detail']['instance-id']
    user = event['detail']['userIdentity']['arn']
    ec2.create_tags(
        Resources=[instance_id],
        Tags=[
            {'Key': 'Owner', 'Value': user},
            {'Key': 'LaunchedBy', 'Value': 'Lambda-AutoTag'}
        ]
    )
```

**Nightly EBS Snapshot Cleanup:**

* EventBridge cron → Lambda → deletes snapshots older than 30 days.

**Slack Notifications for CloudWatch Alarms:**

* SNS → Lambda → posts to Slack webhook on CPU spike.

### When NOT to Use Lambda

* ❌ Long-running jobs (> 15 min) → use ECS/Batch/Step Functions.
* ❌ Steady 24/7 heavy traffic → EC2/ECS is cheaper at scale.
* ❌ Stateful WebSocket servers (short-lived per invocation).
* ❌ Very low-latency (< 10 ms) → cold starts hurt.
* ❌ Applications requiring GPU (limited support).

***

## 26. How is Lambda Billed?

Lambda pricing has **three main components**: number of requests, execution duration, and provisioned features.

### Pricing Model

| Component                    | Cost                                                     |
| ---------------------------- | -------------------------------------------------------- |
| **Requests**                 | $0.20 per 1 million requests                             |
| **Duration (x86)**           | $0.0000166667 per GB-second                              |
| **Duration (ARM/Graviton2)** | $0.0000133334 per GB-second (\~20% cheaper)              |
| **Provisioned Concurrency**  | $0.0000041667 per GB-second (always billed even if idle) |
| **Ephemeral Storage**        | Free up to 512 MB; $0.0000000309 per GB-s beyond         |
| **Data Transfer**            | Standard AWS egress ($0.09/GB out)                       |

### Free Tier (always-free)

* **1 million requests/month**
* **400,000 GB-seconds/month** (\~3.2M seconds at 128 MB)

### Formula

```
Cost = (Requests × $0.20 / 1,000,000)
     + (Invocations × Memory_GB × Duration_seconds × $0.0000166667)
```

### Example Calculation

Function: 512 MB memory, 200 ms average, 5M invocations/month.

* **Requests cost**: 5M × ($0.20 / 1M) = **$1.00**
* **GB-seconds**: 5M × 0.5 GB × 0.2 s = 500,000 GB-s
* **Duration cost**: 500,000 × $0.0000166667 = **$8.33**
* **Total**: \~**$9.33/month**

### Key Billing Insights

1. **Memory drives cost linearly**, but more memory = more vCPU → shorter duration → sometimes **cheaper overall**.
2. **Right-size memory** using **AWS Lambda Power Tuning** tool.
3. **Duration billed in 1 ms increments** (previously 100 ms).
4. **Cold starts are billed** (initialization time counts).
5. **Failed invocations still bill** (except throttles / permission errors).
6. **ARM/Graviton2 is \~20% cheaper** — use unless you need x86-specific libs.

### Cost Optimization Tips

* ✅ Use **ARM64** (Graviton2) architecture.
* ✅ **Right-size memory** — sometimes bumping from 128→512 MB reduces duration enough to save money.
* ✅ Use **provisioned concurrency** only for latency-critical, predictable traffic.
* ✅ **Reduce cold starts** — smaller deployment packages, avoid VPC unless required.
* ✅ Use **Lambda SnapStart** (Java) for near-zero cold start.
* ✅ Batch events from SQS/Kinesis to reduce invocation count.
* ✅ Set appropriate **timeout** — prevent runaway costs on hung functions.

***

## 27. What is API Gateway and How Does It Integrate with Lambda?

**Amazon API Gateway** is a **fully managed service** for creating, publishing, securing, and monitoring APIs at scale. It's the **front door** for serverless applications.

### API Gateway Types

| Type                 | Protocol  | Use Case                                                               | Cost                      |
| -------------------- | --------- | ---------------------------------------------------------------------- | ------------------------- |
| **REST API**         | HTTP/REST | Full features (throttling, caching, API keys, WAF, request validation) | \$$$                      |
| **HTTP API**         | HTTP      | Modern, cheaper, faster, JWT auth built-in                             | $ (70% cheaper than REST) |
| **WebSocket API**    | WebSocket | Real-time bi-directional (chat, live dashboards, gaming)               | \$$                       |
| **Private REST API** | HTTP      | Internal APIs only accessible from within VPC                          | \$$$                      |

### Key Features

* **Traffic management** — throttling, quotas, burst limits.
* **Authorization** — IAM, Cognito, Lambda authorizers, JWT (HTTP APIs).
* **Request/response transformation** — mapping templates (VTL).
* **Caching** — response cache (0.5 GB – 237 GB).
* **API versioning & stages** — dev, staging, prod.
* **Custom domain names** with ACM SSL certs.
* **CORS** support built-in.
* **Usage plans + API keys** for third-party consumers.
* **CloudWatch + X-Ray integration**.
* **WAF integration** for L7 protection.

### Lambda Integration Types

| Integration                    | How                                                                                    |
| ------------------------------ | -------------------------------------------------------------------------------------- |
| **Lambda Proxy** (most common) | API GW passes entire request as JSON to Lambda; Lambda returns full response structure |
| **Lambda Custom**              | Use mapping templates to transform request/response — more control                     |

### Lambda Proxy Event Structure

```json
{
  "resource": "/users/{id}",
  "path": "/users/42",
  "httpMethod": "GET",
  "headers": { "Authorization": "Bearer xyz" },
  "queryStringParameters": { "verbose": "true" },
  "pathParameters": { "id": "42" },
  "body": "{\"name\":\"Rajesh\"}",
  "requestContext": { "requestId": "abc-123" }
}
```

### Lambda Response Structure (Proxy Mode)

```python
def lambda_handler(event, context):
    user_id = event['pathParameters']['id']
    return {
        'statusCode': 200,
        'headers': {
            'Content-Type': 'application/json',
            'Access-Control-Allow-Origin': '*'
        },
        'body': json.dumps({'id': user_id, 'name': 'Rajesh'})
    }
```

### Common Architecture Pattern

```
Client → CloudFront → WAF → API Gateway → Lambda → DynamoDB
                              ↓
                          Cognito (auth)
                              ↓
                          CloudWatch (logs/metrics)
```

### Setup via CLI (HTTP API + Lambda)

```bash
# Create Lambda function
aws lambda create-function --function-name myApi \
  --runtime python3.12 --handler index.handler \
  --role arn:aws:iam::123456789012:role/lambda-role \
  --zip-file fileb://function.zip

# Create HTTP API
aws apigatewayv2 create-api --name my-http-api --protocol-type HTTP \
  --target arn:aws:lambda:us-east-1:123456789012:function:myApi

# Grant API GW permission to invoke Lambda
aws lambda add-permission --function-name myApi \
  --statement-id apigw-invoke --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com
```

### REST API vs HTTP API — Quick Guide

| Feature                    | REST API                  | HTTP API            |
| -------------------------- | ------------------------- | ------------------- |
| **Pricing**                | $3.50 per M               | $1.00 per M         |
| **Latency**                | Higher                    | Lower (\~50% less)  |
| **JWT Auth built-in**      | ❌ (via Lambda authorizer) | ✅                   |
| **Request Validation**     | ✅                         | ❌                   |
| **Caching**                | ✅                         | ❌                   |
| **API Keys / Usage Plans** | ✅                         | ❌                   |
| **WAF**                    | ✅                         | Via CloudFront      |
| **Best For**               | Legacy, complex APIs      | New serverless APIs |

### Best Practices

* ✅ Use **HTTP API** for new projects unless you need REST-specific features.
* ✅ Enable **CloudWatch logs** + **X-Ray tracing**.
* ✅ Use **stages** (dev/staging/prod) with **stage variables**.
* ✅ Apply **throttling** to prevent Lambda overload / cost blowup.
* ✅ Use **Cognito or JWT authorizers** — never expose unauthenticated write endpoints.

***

## 28. What is CloudFormation?

**AWS CloudFormation** is AWS's native **Infrastructure as Code (IaC)** service that provisions and manages AWS resources through **declarative templates**. You define desired state → CloudFormation figures out the API calls, order, and rollback.

### Why Use CloudFormation

* **Version-controlled infrastructure** (Git-based).
* **Repeatable environments** (dev / staging / prod identical).
* **Automated rollback** on failure.
* **Drift detection** — detect manual changes.
* **Change sets** — preview changes before applying.
* **Free** — you only pay for the resources created.

### Core Concepts

| Concept                 | Description                                                 |
| ----------------------- | ----------------------------------------------------------- |
| **Template**            | YAML/JSON file describing desired resources                 |
| **Stack**               | Deployed instance of a template (all resources as one unit) |
| **Change Set**          | Preview of changes before executing                         |
| **Stack Set**           | Deploy the same stack across multiple accounts/regions      |
| **Nested Stack**        | Stack referenced within another (modularization)            |
| **Parameters**          | Runtime inputs                                              |
| **Mappings**            | Static lookups (region → AMI ID)                            |
| **Conditions**          | Conditional resource creation                               |
| **Outputs**             | Export values for other stacks                              |
| **Intrinsic Functions** | `!Ref`, `!GetAtt`, `!Sub`, `!Join`, `!ImportValue`          |
| **Drift Detection**     | Compare live state vs template                              |

### Deployment Workflow

```
1. Author template (YAML/JSON)
2. Validate: aws cloudformation validate-template
3. Create Change Set (preview)
4. Execute Change Set
5. Monitor via Events tab / CloudWatch
6. Rollback automatically on failure (or manual)
```

### CloudFormation vs Terraform (interview classic!)

| Aspect        | CloudFormation | Terraform                |
| ------------- | -------------- | ------------------------ |
| **Vendor**    | AWS-only       | Multi-cloud              |
| **Language**  | YAML/JSON      | HCL                      |
| **State**     | AWS-managed    | You manage (S3+DynamoDB) |
| **Rollback**  | Automatic      | Manual                   |
| **Modules**   | Nested stacks  | First-class modules      |
| **Preview**   | Change Sets    | `terraform plan`         |
| **Community** | AWS-blessed    | Massive registry         |
| **Best For**  | Pure AWS shops | Multi-cloud / hybrid     |

### CloudFormation Alternatives from AWS

* **AWS CDK** — write CloudFormation in TypeScript, Python, Java, Go, C#.
* **SAM (Serverless Application Model)** — CFN extension for Lambda/API GW.
* **AWS Copilot** — CLI for ECS/App Runner.
* **AWS Amplify** — full-stack app IaC.

### Best Practices

* ✅ Use **YAML** (more readable than JSON).
* ✅ Modularize with **nested stacks** or **CDK constructs**.
* ✅ Store templates in **Git**, deploy via **CodePipeline** or GitHub Actions.
* ✅ Use **StackSets** for multi-account deployments (AWS Organizations).
* ✅ Enable **termination protection** on prod stacks.
* ✅ Tag every resource for cost allocation.
* ✅ Use **CDK** for complex/logic-heavy infrastructure.

***

## 29. What Are CloudFormation Templates Written In?

CloudFormation templates are written in **YAML** or **JSON**. Both are functionally equivalent, but **YAML is preferred** for readability and support of comments.

### Template Structure (Standard Sections)

```yaml
AWSTemplateFormatVersion: '2010-09-09'   # Version
Description: 'My web app stack'          # Human description

Metadata:                                # Extra info (UI hints, etc.)
  AWS::CloudFormation::Interface: {}

Parameters:                              # Runtime inputs
  InstanceType:
    Type: String
    Default: t3.micro
    AllowedValues: [t3.micro, t3.small, t3.medium]
    Description: EC2 instance type

Mappings:                                # Static lookups
  RegionMap:
    us-east-1: { AMI: ami-0abc123 }
    us-west-2: { AMI: ami-0def456 }

Conditions:                              # Conditional creation
  IsProd: !Equals [!Ref Environment, 'prod']

Resources:                               # REQUIRED — actual AWS resources
  MyEC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceType
      ImageId: !FindInMap [RegionMap, !Ref 'AWS::Region', AMI]
      Tags:
        - Key: Name
          Value: !Sub '${AWS::StackName}-web'

Outputs:                                 # Export values
  InstanceId:
    Value: !Ref MyEC2Instance
    Export:
      Name: !Sub '${AWS::StackName}-InstanceId'
```

### Only `Resources` Section is Required — Everything Else is Optional

### Common Intrinsic Functions

| Function       | Purpose                            | Example                                  |
| -------------- | ---------------------------------- | ---------------------------------------- |
| `!Ref`         | Reference param or resource        | `!Ref MyBucket`                          |
| `!GetAtt`      | Get attribute of a resource        | `!GetAtt MyEC2.PublicIp`                 |
| `!Sub`         | String substitution                | `!Sub 'arn:aws:s3:::${BucketName}/*'`    |
| `!Join`        | Concatenate strings                | `!Join ['-', [!Ref Env, 'bucket']]`      |
| `!Split`       | Split string into list             | `!Split [',', 'a,b,c']`                  |
| `!FindInMap`   | Lookup in Mappings                 | `!FindInMap [RegionMap, us-east-1, AMI]` |
| `!ImportValue` | Import from another stack's output | `!ImportValue VPC-ID`                    |
| `!If`          | Conditional value                  | `!If [IsProd, m5.large, t3.small]`       |
| `!Equals`      | Compare values                     | `!Equals [!Ref Env, prod]`               |
| `!GetAZs`      | List of AZs in region              | `!GetAZs us-east-1`                      |

### Pseudo Parameters (built-in)

* `AWS::Region` — current region.
* `AWS::AccountId` — current account.
* `AWS::StackName` — this stack's name.
* `AWS::Partition` — `aws`, `aws-cn`, `aws-us-gov`.
* `AWS::NoValue` — remove a property conditionally.

### Complete Working Example (Lambda + API GW)

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Serverless Hello World

Resources:
  MyFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: hello-world
      Runtime: python3.12
      Handler: index.handler
      Role: !GetAtt LambdaRole.Arn
      Code:
        ZipFile: |
          def handler(event, context):
              return {'statusCode': 200, 'body': 'Hello from CFN!'}

  LambdaRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal: { Service: lambda.amazonaws.com }
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

Outputs:
  FunctionArn:
    Value: !GetAtt MyFunction.Arn
```

### Deploy the Stack

```bash
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name hello-world-stack \
  --capabilities CAPABILITY_IAM
```

### YAML vs JSON

| Feature                             | YAML       | JSON                       |
| ----------------------------------- | ---------- | -------------------------- |
| Readability                         | ✅ Superior | ❌ Verbose                  |
| Comments (`#`)                      | ✅ Yes      | ❌ No                       |
| Short-form intrinsic funcs (`!Ref`) | ✅ Yes      | ❌ (must use `{"Ref":...}`) |
| Preferred?                          | ✅ **Yes**  | Only for auto-generated    |

***

## 30. What is AWS CloudWatch?

**Amazon CloudWatch** is AWS's **unified observability service** for monitoring, logging, tracing, and alarming across AWS resources, applications, and on-prem systems. It's the DevOps engineer's daily driver.

### Core Components

| Component                      | Purpose                                              |
| ------------------------------ | ---------------------------------------------------- |
| **Metrics**                    | Numeric time-series data (CPU, memory, requests/sec) |
| **Logs**                       | Centralized log storage & search (CloudWatch Logs)   |
| **Alarms**                     | Trigger notifications/actions on thresholds          |
| **Events / EventBridge**       | Event-driven automation (cron, state changes)        |
| **Dashboards**                 | Custom visualizations                                |
| **Log Insights**               | SQL-like query language for logs                     |
| **Synthetics**                 | Canary scripts to monitor endpoints                  |
| **RUM (Real User Monitoring)** | Client-side browser telemetry                        |
| **ServiceLens**                | Distributed tracing (with X-Ray)                     |
| **Container Insights**         | ECS/EKS monitoring                                   |
| **Application Insights**       | Auto-monitor .NET/SQL/Java apps                      |
| **Contributor Insights**       | Top-N analytics on logs                              |

### CloudWatch Metrics Deep Dive

* **Namespaces**: `AWS/EC2`, `AWS/RDS`, `AWS/Lambda`, custom (`MyApp/Prod`).
* **Dimensions**: Filters (e.g., `InstanceId=i-abc`).
* **Standard metrics**: Free, 5-minute granularity (1 min for detailed monitoring).
* **Custom metrics**: $0.30/metric/month; publish via CLI/SDK.
* **High-resolution metrics**: 1-second granularity for custom metrics.
* **Retention**: Metrics retained 15 months (rolled up).

### Publishing Custom Metrics

```bash
aws cloudwatch put-metric-data \
  --namespace "MyApp/Prod" \
  --metric-name OrdersPlaced \
  --value 42 \
  --unit Count \
  --dimensions Environment=Prod,Region=us-east-1
```

### CloudWatch Logs Concepts

* **Log Groups** — logical containers (e.g., `/aws/lambda/myFunc`, `/var/log/nginx`).
* **Log Streams** — sequences within a group (per instance/session).
* **Retention policy** — 1 day to 10 years (or Never Expire).
* **Metric Filters** — extract numbers from logs into metrics.
* **Subscription Filters** — stream logs to Kinesis/Lambda/OpenSearch.

### CloudWatch Logs Insights Query Example

```sql
fields @timestamp, @message
| filter @message like /ERROR/
| stats count() by bin(5m)
| sort @timestamp desc
| limit 100
```

### CloudWatch Alarms

**States**: `OK`, `ALARM`, `INSUFFICIENT_DATA`.

**Actions on alarm**:

* Send SNS notification (email/SMS/Slack via Lambda).
* Trigger **Auto Scaling** action.
* Reboot / stop / terminate EC2.
* Trigger **Systems Manager** automation.

**Example — CPU High Alarm:**

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "EC2-HighCPU-prod" \
  --metric-name CPUUtilization --namespace AWS/EC2 \
  --statistic Average --period 300 \
  --threshold 80 --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=InstanceId,Value=i-0abc \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:ops-alerts
```

### Composite Alarms

Combine multiple alarms with AND/OR/NOT logic → reduce alert fatigue.

```
ALARM("HighCPU") AND ALARM("HighMemory") → PageOnCall
```

### Anomaly Detection

ML-based dynamic thresholds — good for spiky/seasonal metrics (e.g., web traffic).

### CloudWatch Agent

Unified agent for EC2/on-prem servers to collect:

* OS-level metrics (memory, disk — not available by default).
* Application logs.
* StatsD/collectd metrics.

Install:

```bash
sudo yum install amazon-cloudwatch-agent -y
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 -c file:/opt/aws/config.json -s
```

### EventBridge (formerly CloudWatch Events)

Event bus for AWS service events + SaaS integrations.

```
Every day at 2 AM UTC → Lambda → Delete old snapshots
EC2 state change → Lambda → Update CMDB
CodeBuild failure → SNS → Slack
```

### Real-World DevOps CloudWatch Stack

```
EC2 / EKS / RDS / Lambda
        │
        ▼
CloudWatch Metrics + Logs
        │
        ├── Alarms → SNS → Slack / PagerDuty / OpsGenie
        ├── Dashboards (business + tech KPIs)
        ├── Log Insights (troubleshooting)
        ├── Container Insights (EKS pods)
        ├── EventBridge → Lambda (auto-remediation)
        └── Export → S3 → Athena (long-term analytics)
```

### Cost Optimization

* ✅ Set **log retention** (default is Never Expire — \$$$).
* ✅ Use **metric filters** instead of storing all logs long-term.
* ✅ Export old logs to **S3 + Glacier**.
* ✅ Aggregate custom metrics (batch `PutMetricData` calls).
* ✅ Delete unused dashboards & alarms.

### Best Practices

* ✅ Enable **detailed monitoring** on prod EC2 (1-min resolution).
* ✅ Install **CloudWatch Agent** on all instances for memory/disk.
* ✅ Use **structured JSON logging** — enables powerful Insights queries.
* ✅ Create **service-level dashboards** grouped by app/team.
* ✅ Use **composite alarms** to reduce alert noise.
* ✅ Integrate with **X-Ray** for distributed tracing.
* ✅ Alert on **error rates and latency**, not just CPU.

***

## Summary Cheat Sheet (Q21–Q30)

| #  | Concept                   | One-Liner                                                                           |
| -- | ------------------------- | ----------------------------------------------------------------------------------- |
| 21 | **RDS**                   | Managed relational DB — MySQL, PostgreSQL, Aurora, Oracle, SQL Server, MariaDB, Db2 |
| 22 | **Read Replica**          | Async read-only copy for scaling reads                                              |
| 23 | **Multi-AZ**              | Sync standby in another AZ for automatic failover HA                                |
| 24 | **Lambda**                | Serverless event-driven compute, pay-per-ms                                         |
| 25 | **Lambda Use Cases**      | APIs, S3 processing, cron jobs, stream ETL, automation                              |
| 26 | **Lambda Billing**        | Requests + GB-seconds; free tier 1M req + 400K GB-s                                 |
| 27 | **API Gateway**           | Managed API front door; REST/HTTP/WebSocket; integrates natively with Lambda        |
| 28 | **CloudFormation**        | Native AWS IaC service — declarative, auto-rollback                                 |
| 29 | **CFN Template Language** | YAML (preferred) or JSON                                                            |
| 30 | **CloudWatch**            | Unified monitoring: metrics, logs, alarms, events, dashboards                       |

***

## Bonus: Interview Scenarios

**Q:** *"How would you design HA for a critical production RDS?"*
**A:** Multi-AZ + automated backups (35 days) + Read Replicas for read scaling + Cross-Region Read Replica for DR + RDS Proxy for connection pooling + Performance Insights for troubleshooting + Secrets Manager for credential rotation.

**Q:** *"Your Lambda is timing out at 15 min processing a large file. Fix?"*
**A:** Break into smaller chunks (Step Functions), use **ECS Fargate** or **AWS Batch** for long jobs, use **S3 event-driven fan-out** to parallel Lambda invocations, or process via **Kinesis Data Streams**.

**Q:** *"A CloudFormation stack failed and rolled back. How do you debug?"*
**A:** Check the **Events tab** for the first failure reason → CloudTrail for API errors → CloudWatch logs (if custom resources) → temporarily disable rollback for debug (`--disable-rollback`) → use `describe-stack-events`.

**Q:** *"How do you monitor a distributed microservices app on EKS?"*
**A:** **Container Insights** for cluster/pod metrics, **CloudWatch Logs** with FluentBit sidecar, **X-Ray** for distributed tracing, custom **application metrics** via CloudWatch Embedded Metric Format (EMF), **Composite alarms** for SLO breach, **EventBridge → Lambda** for auto-remediation.

***

