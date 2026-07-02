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
# AWS Monitoring, Messaging & Container Services – Detailed Interview-Ready Answers (Q31–Q40)

Here's a comprehensive walkthrough of your next 10 questions covering **CloudWatch, CloudTrail, SQS, SNS, SES, and EKS** — structured for a **Senior DevOps interview** with practical examples, CLI snippets, architecture patterns, and real-world scenarios.

***

## 31. What is CloudWatch Logs?

**Amazon CloudWatch Logs** is a **fully managed log aggregation, storage, search, and analysis service** for logs from AWS services, applications, and on-premises systems. It's the central log repository for AWS-native observability.

### Key Concepts

| Concept                       | Description                                                                               |
| ----------------------------- | ----------------------------------------------------------------------------------------- |
| **Log Group**                 | Logical container for logs sharing retention/access policies (e.g., `/aws/lambda/myFunc`) |
| **Log Stream**                | Sequence of log events from a single source (per instance/session/container)              |
| **Log Event**                 | Single record with timestamp + message                                                    |
| **Retention Policy**          | 1 day to 10 years, or **Never Expire** (default — watch costs!)                           |
| **Metric Filters**            | Extract numeric values from logs → CloudWatch Metrics                                     |
| **Subscription Filters**      | Real-time stream to Kinesis / Lambda / OpenSearch / Firehose                              |
| **Log Insights**              | SQL-like query language for ad-hoc analysis                                               |
| **Contributor Insights**      | Top-N analytics (top talkers, top errors)                                                 |
| **Live Tail**                 | Real-time log streaming in the console                                                    |
| **Cross-Account Log Sharing** | Central logging account pattern                                                           |

### Sources That Auto-Send to CloudWatch Logs

* Lambda functions (default).
* API Gateway (execution + access logs).
* VPC Flow Logs.
* Route 53 query logs.
* CloudTrail (optional).
* ECS/EKS containers (via awslogs/FluentBit driver).
* RDS logs (error, slow query, audit).
* ELB/ALB access logs (via S3 → subscription).
* EC2 (via **CloudWatch Agent**).
* On-prem servers (via CloudWatch Agent).

### Install CloudWatch Agent on EC2 (Linux)

```bash
# Install
sudo yum install amazon-cloudwatch-agent -y

# Sample config for /var/log/messages
cat > /opt/aws/config.json <<EOF
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [{
          "file_path": "/var/log/messages",
          "log_group_name": "prod-app-logs",
          "log_stream_name": "{instance_id}/messages",
          "retention_in_days": 30
        }]
      }
    }
  }
}
EOF

# Start agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 -c file:/opt/aws/config.json -s
```

### CloudWatch Logs Insights — Query Examples

**Top 10 error messages in the last hour:**

```sql
fields @timestamp, @message
| filter @message like /ERROR/
| stats count() as errorCount by @message
| sort errorCount desc
| limit 10
```

**Lambda cold starts:**

```sql
filter @type = "REPORT"
| stats count() as coldStarts by bin(5m)
| filter ispresent(@initDuration)
```

**API Gateway 5xx errors by endpoint:**

```sql
fields @timestamp, path, status
| filter status >= 500
| stats count() by path
| sort count desc
```

### Real-World Log Streaming Architecture

```
App Logs ──> CloudWatch Logs ──> Subscription Filter
                                        │
                        ┌───────────────┼─────────────────┐
                        ▼               ▼                 ▼
                    Lambda          Kinesis Firehose   OpenSearch
                (transform)         (to S3 archive)    (search UI)
```

### Best Practices

* ✅ **Always set retention** — default is Never Expire (huge cost trap).
* ✅ Use **structured JSON logging** — powers rich Insights queries.
* ✅ Group logs by application/environment (`/app/prod/api`, `/app/dev/api`).
* ✅ Use **metric filters** to convert log events into alertable metrics.
* ✅ Export old logs to **S3 + Glacier** for compliance.
* ✅ Centralize logs across accounts using **subscription filters** to a logging account.

***

## 32. What is CloudWatch Metrics?

**CloudWatch Metrics** is the **time-series database** at the heart of CloudWatch — storing **numeric data points** about your AWS resources and applications, queryable and alertable.

### Key Concepts

| Term           | Description                                                                                    |
| -------------- | ---------------------------------------------------------------------------------------------- |
| **Metric**     | A time-ordered set of data points (e.g., CPUUtilization)                                       |
| **Namespace**  | Container for metrics (e.g., `AWS/EC2`, `AWS/Lambda`, `MyApp/Prod`)                            |
| **Dimension**  | Name/value pair to identify a metric (e.g., `InstanceId=i-abc`)                                |
| **Data Point** | Timestamp + value + optional statistics                                                        |
| **Statistic**  | Aggregation: `Average`, `Sum`, `Minimum`, `Maximum`, `SampleCount`, percentiles (`p95`, `p99`) |
| **Period**     | Interval for aggregation (1s, 10s, 30s, 60s, 5 min…)                                           |
| **Unit**       | `Bytes`, `Percent`, `Count`, `Milliseconds`, etc.                                              |
| **Resolution** | Standard (60s) or High-resolution (1s)                                                         |
| **Retention**  | 3 hours (1s res) → 15 months (60s res, rolled up)                                              |

### Metric Types

* **Standard AWS metrics** — auto-published, free (5-min for EC2 basic, 1-min for detailed).
* **Custom metrics** — you publish via CLI/SDK ($0.30/metric/month).
* **High-resolution metrics** — 1-second granularity (critical dashboards).

### Publishing Custom Metrics (CLI)

```bash
aws cloudwatch put-metric-data \
  --namespace "MyApp/Prod" \
  --metric-name OrdersProcessed \
  --value 42 \
  --unit Count \
  --dimensions Environment=Prod,Service=Checkout
```

### Publishing via Python SDK

```python
import boto3
cw = boto3.client('cloudwatch')

cw.put_metric_data(
    Namespace='MyApp/Prod',
    MetricData=[{
        'MetricName': 'OrderLatency',
        'Dimensions': [{'Name': 'Service', 'Value': 'Checkout'}],
        'Value': 234.5,
        'Unit': 'Milliseconds',
        'StorageResolution': 1  # high-res
    }]
)
```

### EC2 Default vs Missing Metrics

| Category               | Available by Default | Requires CloudWatch Agent |
| ---------------------- | -------------------- | ------------------------- |
| CPU Utilization        | ✅                    |                           |
| Network In/Out         | ✅                    |                           |
| Disk Read/Write (EBS)  | ✅                    |                           |
| Status Checks          | ✅                    |                           |
| **Memory Utilization** | ❌                    | ✅                         |
| **Disk Space Used**    | ❌                    | ✅                         |
| **Swap Usage**         | ❌                    | ✅                         |
| **Process Metrics**    | ❌                    | ✅                         |

### Metric Math

Combine multiple metrics into derived expressions.

```
Requests per minute:   m1 = SUM(RequestCount)
Errors per minute:     m2 = SUM(5xxError)
Error Rate:            e1 = (m2 / m1) * 100
```

Use in dashboards & alarms.

### CloudWatch Embedded Metric Format (EMF)

Log structured JSON → CloudWatch auto-extracts metrics (no `PutMetricData` calls needed). Great for Lambda/containers.

```json
{
  "_aws": {
    "Timestamp": 1690000000000,
    "CloudWatchMetrics": [{
      "Namespace": "MyApp",
      "Dimensions": [["Service"]],
      "Metrics": [{"Name": "Latency", "Unit": "Milliseconds"}]
    }]
  },
  "Service": "Checkout",
  "Latency": 234
}
```

### Best Practices

* ✅ Enable **detailed monitoring** (1-min) on prod EC2.
* ✅ Install **CloudWatch Agent** for memory/disk metrics.
* ✅ Use **EMF** in Lambda/containers instead of `PutMetricData` (cheaper + faster).
* ✅ Standardize **dimensions** across services (`Environment`, `Service`, `Region`).
* ✅ Track **business metrics**, not just infra (orders/sec, signups/hr).

***

## 33. What is CloudWatch Alarms?

**CloudWatch Alarms** watch metrics and trigger actions when a defined threshold is breached — the automation backbone of AWS monitoring.

### Alarm States

| State                  | Meaning                          |
| ---------------------- | -------------------------------- |
| **OK**                 | Metric is within threshold       |
| **ALARM**              | Threshold breached               |
| **INSUFFICIENT\_DATA** | Not enough data points to decide |

### Alarm Types

| Type                        | Description                                         |
| --------------------------- | --------------------------------------------------- |
| **Metric Alarm**            | Single metric threshold (e.g., CPU > 80% for 5 min) |
| **Composite Alarm**         | Combine multiple alarms with AND/OR/NOT logic       |
| **Anomaly Detection Alarm** | ML-based dynamic threshold bands                    |

### Actions When Alarm Triggers

1. **SNS notification** → email/SMS/Slack/PagerDuty via Lambda.
2. **Auto Scaling action** → add/remove EC2 instances.
3. **EC2 action** → stop/terminate/reboot/recover instance.
4. **Systems Manager action** → run automation runbook (auto-remediation).
5. **Lambda invocation** (via EventBridge).

### Anatomy of an Alarm

```
Metric:               AWS/EC2 → CPUUtilization
Dimension:            InstanceId=i-0abc
Statistic:            Average
Period:               60 seconds
Evaluation Periods:   3 (i.e., 3 minutes)
Datapoints to Alarm:  3 out of 3
Threshold:            >= 80
Comparison:           GreaterThanOrEqualToThreshold
Treat missing data:   notBreaching
```

### Create Alarm — CLI

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "EC2-HighCPU-prod-web01" \
  --alarm-description "CPU >= 80% for 3 minutes" \
  --metric-name CPUUtilization --namespace AWS/EC2 \
  --statistic Average --period 60 \
  --threshold 80 --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 3 --datapoints-to-alarm 3 \
  --dimensions Name=InstanceId,Value=i-0abc \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:ops-critical \
  --ok-actions arn:aws:sns:us-east-1:123456789012:ops-recovery \
  --treat-missing-data notBreaching
```

### Composite Alarm Example

Only page on-call when **BOTH** CPU and memory are high (reduces false alerts).

```bash
aws cloudwatch put-composite-alarm \
  --alarm-name "PagerDuty-HighLoad" \
  --alarm-rule "ALARM('EC2-HighCPU') AND ALARM('EC2-HighMemory')" \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:pagerduty
```

### Anomaly Detection Alarm

Use for spiky/seasonal metrics (e.g., web traffic where daytime vs nighttime differ). CloudWatch learns the pattern and alerts on **deviation**, not static threshold.

### EC2 Auto-Recovery Alarm

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "EC2-AutoRecover-i-0abc" \
  --metric-name StatusCheckFailed_System --namespace AWS/EC2 \
  --statistic Maximum --period 60 --threshold 0 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=InstanceId,Value=i-0abc \
  --alarm-actions arn:aws:automate:us-east-1:ec2:recover
```

### Real-World Alarm Strategy

| Layer              | Example Alarms                                          |
| ------------------ | ------------------------------------------------------- |
| **Infrastructure** | EC2 CPU/memory, disk full, EBS burst credits low        |
| **Platform**       | RDS connections, ELB 5xx, Lambda throttles              |
| **Application**    | Error rate > 1%, p99 latency > 500ms                    |
| **Business**       | Orders/hour drops below expected, signup failure spikes |
| **Cost**           | Estimated monthly bill > $X                             |

### Best Practices

* ✅ Alert on **rates** (error %, latency percentiles) not raw counts.
* ✅ Use **composite alarms** to reduce noise.
* ✅ Set `treat-missing-data` explicitly (usually `notBreaching`).
* ✅ Use **anomaly detection** for seasonal metrics.
* ✅ Route alarms by severity (`ALARM` → PagerDuty, `WARN` → Slack).
* ✅ Test alarms regularly (chaos engineering).
* ✅ Include **runbook link** in alarm descriptions.

***

## 34. What is CloudTrail and How Is It Different from CloudWatch?

**AWS CloudTrail** is the **audit log service** that records **all API calls** made in your AWS account — who did what, when, from where. It's your compliance, security, and forensic tool.

### What CloudTrail Records

Every API call including:

* Console clicks (translated to API calls).
* AWS CLI, SDK, Terraform, CloudFormation actions.
* AWS service-to-service calls.
* Failed API calls (permission denied).

### Event Types

| Type                        | Description                                                             | Cost                            |
| --------------------------- | ----------------------------------------------------------------------- | ------------------------------- |
| **Management Events**       | Control plane operations (`RunInstances`, `CreateBucket`, `AssumeRole`) | Free (first copy of read+write) |
| **Data Events**             | Data plane operations (S3 object-level `GetObject`, Lambda `Invoke`)    | Paid (high volume)              |
| **Insights Events**         | ML detects unusual API activity spikes                                  | Paid                            |
| **Network Activity Events** | VPC endpoint calls (newer, 2024+)                                       | Paid                            |

### CloudTrail Trails

| Trail Type             | Scope                                       |
| ---------------------- | ------------------------------------------- |
| **Management Trail**   | One region or all regions                   |
| **Organization Trail** | Applies to all accounts in AWS Organization |
| **Event History**      | Last 90 days, free, console-only search     |

### CloudTrail vs CloudWatch — Critical Distinction

| Aspect                | **CloudTrail**                                  | **CloudWatch**                                    |
| --------------------- | ----------------------------------------------- | ------------------------------------------------- |
| **Purpose**           | Audit "**who did what**" (API activity)         | Monitor "**how is it performing**" (metrics/logs) |
| **Data Type**         | API call events                                 | Metrics + application logs                        |
| **Question Answered** | "Who deleted the S3 bucket at 3 AM?"            | "Was CPU high at 3 AM?"                           |
| **Retention**         | 90 days (Event History), forever if S3-archived | 15 months (metrics), configurable (logs)          |
| **Primary Users**     | Security, compliance, auditors                  | DevOps, SRE, developers                           |
| **Free Tier**         | Management events free                          | Basic metrics + limited alarms free               |
| **Format**            | Structured JSON per event                       | Metrics or free-form log lines                    |
| **Integration**       | Delivers to S3 / CloudWatch Logs / EventBridge  | Alarms / SNS / Lambda / EventBridge               |

### CloudTrail Event Structure (Sample)

```json
{
  "eventVersion": "1.09",
  "eventTime": "2026-07-02T12:34:56Z",
  "eventSource": "s3.amazonaws.com",
  "eventName": "DeleteBucket",
  "awsRegion": "us-east-1",
  "sourceIPAddress": "203.0.113.42",
  "userAgent": "aws-cli/2.13.0",
  "userIdentity": {
    "type": "IAMUser",
    "userName": "rajesh.singh",
    "arn": "arn:aws:iam::123456789012:user/rajesh.singh"
  },
  "requestParameters": { "bucketName": "prod-critical-data" },
  "responseElements": null,
  "errorCode": null
}
```

### Create a Trail

```bash
# Create S3 bucket
aws s3 mb s3://my-cloudtrail-logs-123456

# Create trail
aws cloudtrail create-trail \
  --name org-trail \
  --s3-bucket-name my-cloudtrail-logs-123456 \
  --is-multi-region-trail \
  --enable-log-file-validation \
  --cloud-watch-logs-log-group-arn arn:aws:logs:us-east-1:123456789012:log-group:CloudTrail:* \
  --cloud-watch-logs-role-arn arn:aws:iam::123456789012:role/CloudTrail-CWL-Role

aws cloudtrail start-logging --name org-trail
```

### Common Security Detections via CloudTrail

| Scenario                         | Detection Approach                        |
| -------------------------------- | ----------------------------------------- |
| Root user login                  | EventBridge rule → SNS/PagerDuty          |
| IAM policy attached with `*:*`   | Metric filter + alarm                     |
| S3 bucket made public            | Config rule + EventBridge                 |
| Console login without MFA        | EventBridge rule                          |
| API calls from unusual IPs       | GuardDuty (uses CloudTrail)               |
| Unauthorized AssumeRole attempts | Metric filter on `errorCode=AccessDenied` |

### CloudTrail Log Insights Query

```sql
fields @timestamp, userIdentity.userName, eventName, sourceIPAddress
| filter eventName = "ConsoleLogin"
| filter errorMessage like /Failed/
| stats count() by userIdentity.userName
| sort count desc
```

### Best Practices

* ✅ Enable a **multi-region trail** in every account.
* ✅ Enable **log file validation** (integrity via SHA-256).
* ✅ Store logs in a **dedicated logging account** with restricted access.
* ✅ Send to CloudWatch Logs → metric filters → alarms.
* ✅ Enable **CloudTrail Insights** for anomaly detection.
* ✅ Enable **S3 data events** for critical buckets only (cost).
* ✅ Integrate with **AWS Security Hub, GuardDuty, Athena** for querying.

***

## 35. What is AWS SQS and When Should You Use It?

**Amazon SQS (Simple Queue Service)** is a **fully managed message queuing service** that decouples producers and consumers of messages. It's serverless, scales infinitely, and reliably delivers messages between distributed system components.

### Key Characteristics

* **Fully managed** — no servers, no clustering, no patching.
* **Scales automatically** to millions of msgs/sec.
* **At-least-once delivery** (Standard) or **exactly-once** (FIFO).
* **Message retention**: 1 min to 14 days (default 4 days).
* **Message size**: up to 256 KB (larger via S3 pointer with SQS Extended Client).
* **Pull-based** — consumers poll for messages.
* **Encryption at rest** via KMS.
* **Cheap** — $0.40 per million requests.

### Core Concepts

| Concept                     | Description                                                   |
| --------------------------- | ------------------------------------------------------------- |
| **Producer**                | App that sends messages                                       |
| **Consumer**                | App that polls & processes messages                           |
| **Queue**                   | The message buffer                                            |
| **Message**                 | The payload                                                   |
| **Visibility Timeout**      | Duration a message is hidden after being polled (default 30s) |
| **Long Polling**            | Wait up to 20s for messages (reduces empty responses & cost)  |
| **Dead Letter Queue (DLQ)** | Holds messages that repeatedly fail processing                |
| **Delay Queue**             | Delays delivery of new messages (0–15 min)                    |
| **Message Attributes**      | Metadata alongside body (up to 10)                            |

### When to Use SQS

| Use Case                       | Why                                               |
| ------------------------------ | ------------------------------------------------- |
| **Decouple microservices**     | Producer and consumer scale/deploy independently  |
| **Buffering spikes**           | Absorb traffic bursts (e.g., Black Friday orders) |
| **Async task processing**      | Emails, image thumbnails, PDF generation          |
| **Order processing pipelines** | Ensure no order is lost if downstream is slow     |
| **Retry logic**                | Failed messages auto-retry via visibility timeout |
| **Fan-out with SNS**           | Multiple consumers get copies                     |
| **Batch processing**           | Kinesis alternative for less strict ordering      |
| **Load leveling**              | Smooth out variable load on databases             |

### Basic Flow

```
Producer → SQS Queue → Consumer(s)
             │
             ↓ (after N failures)
           DLQ → Manual/Auto reprocessing
```

### CLI Examples

```bash
# Create queue
aws sqs create-queue --queue-name orders-queue \
  --attributes VisibilityTimeout=60,MessageRetentionPeriod=345600

# Send message
aws sqs send-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/orders-queue \
  --message-body '{"orderId":"1001","amount":499}'

# Receive (long polling)
aws sqs receive-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/orders-queue \
  --wait-time-seconds 20 \
  --max-number-of-messages 10

# Delete after processing
aws sqs delete-message \
  --queue-url ... \
  --receipt-handle <handle>
```

### Lambda Consumer (Event Source Mapping)

Lambda automatically polls SQS → invokes function → deletes on success.

```yaml
Events:
  MyQueue:
    Type: SQS
    Properties:
      Queue: !GetAtt OrdersQueue.Arn
      BatchSize: 10
      MaximumBatchingWindowInSeconds: 5
```

### Best Practices

* ✅ Always configure a **DLQ** (redrive policy after 3–5 attempts).
* ✅ Use **long polling** (`WaitTimeSeconds=20`) — reduces cost & latency.
* ✅ Set **visibility timeout** > longest expected processing time.
* ✅ Use **batching** (SendMessageBatch, up to 10) to reduce API calls.
* ✅ Enable **encryption at rest** (SSE-SQS or SSE-KMS).
* ✅ Monitor `ApproximateAgeOfOldestMessage` — signal of consumer lag.
* ✅ Use **FIFO** only when strict ordering / dedup is required.

***

## 36. Difference Between SQS Standard and FIFO Queues

SQS offers two queue types optimized for different guarantees.

### Comparison Table

| Feature              | **Standard Queue**                               | **FIFO Queue**                                                       |
| -------------------- | ------------------------------------------------ | -------------------------------------------------------------------- |
| **Ordering**         | Best-effort — messages *may* arrive out of order | **Strict FIFO** ordering                                             |
| **Delivery**         | **At-least-once** (duplicates possible)          | **Exactly-once** (dedup within 5 min window)                         |
| **Throughput**       | Unlimited (nearly infinite TPS)                  | 300 msg/s (batching: 3,000 msg/s) — 70,000 with High-Throughput mode |
| **Naming**           | Any name                                         | Name must end with `.fifo`                                           |
| **Message Group ID** | Not used                                         | Required — messages within a group are ordered                       |
| **Deduplication ID** | Not used                                         | Required (or use content-based dedup)                                |
| **Price**            | $0.40 per M requests                             | $0.50 per M requests                                                 |
| **Use Case**         | High-throughput, order not critical              | Financial transactions, order fulfillment, event replay              |

### Standard Queue Behavior

* **Duplicates possible** (rare, but consumer must be idempotent).
* **Out-of-order possible** (usually rare).
* **Massive scale** — millions of TPS.
* Used for **99%** of typical use cases.

### FIFO Queue Behavior

* **Message Group ID** partitions ordering (all messages with same group ID are strictly ordered relative to each other).
* **Deduplication ID** prevents duplicates within a **5-min window**.
* **Content-based deduplication** — hash of message body used as dedup ID.

### When to Use FIFO

| Scenario                   | Why FIFO                                           |
| -------------------------- | -------------------------------------------------- |
| **Financial transactions** | Debit before credit must happen in order           |
| **User registration flow** | Signup → email verify → account create in sequence |
| **Order fulfillment**      | Place → pay → ship must be ordered                 |
| **Change data capture**    | DB events must apply in commit order               |
| **Command processing**     | Enable → configure → start must sequence           |

### FIFO Example

```bash
aws sqs create-queue --queue-name orders.fifo \
  --attributes FifoQueue=true,ContentBasedDeduplication=true

aws sqs send-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/orders.fifo \
  --message-body '{"orderId":"1001"}' \
  --message-group-id "customer-42" \
  --message-deduplication-id "order-1001"
```

### Rule of Thumb

* **Default to Standard** for scale + cost.
* Use **FIFO only when ordering is a business requirement**.
* Design consumers to be **idempotent regardless** (defense in depth).

***

## 37. What is Amazon SNS Used For?

**Amazon SNS (Simple Notification Service)** is a **fully managed pub/sub messaging service** for **fan-out** and **push notifications**. Producers publish to a **topic**; SNS delivers to **all subscribers**.

### Core Concepts

| Concept                | Description                                                    |
| ---------------------- | -------------------------------------------------------------- |
| **Topic**              | Communication channel (e.g., `order-events`)                   |
| **Publisher**          | Sends messages to topic                                        |
| **Subscriber**         | Receives messages (Lambda, SQS, HTTP, email, SMS, mobile push) |
| **Message**            | Payload up to 256 KB                                           |
| **Filter Policy**      | JSON rules to filter which subscribers receive a message       |
| **Message Attributes** | Metadata for filtering                                         |
| **Delivery Retry**     | SNS retries failed deliveries (varies by protocol)             |
| **DLQ**                | Failed deliveries go to SQS DLQ                                |

### Supported Subscriber Protocols

| Protocol                  | Use Case                                  |
| ------------------------- | ----------------------------------------- |
| **Lambda**                | Trigger serverless function               |
| **SQS**                   | Queue message for async processing        |
| **HTTP/HTTPS**            | Webhook to external system                |
| **Email / Email-JSON**    | Notify humans                             |
| **SMS**                   | Text messages (200+ countries)            |
| **Mobile Push**           | APNS (iOS), FCM (Android), etc.           |
| **Kinesis Data Firehose** | Archive to S3 / send to Splunk/OpenSearch |

### Topic Types

| Type             | Guarantees                                                     |
| ---------------- | -------------------------------------------------------------- |
| **Standard SNS** | At-least-once, best-effort order, massive throughput           |
| **FIFO SNS**     | Exactly-once, strict order (only Lambda/SQS/HTTPS subscribers) |

### Common Use Cases

| Use Case                           | Pattern                                                             |
| ---------------------------------- | ------------------------------------------------------------------- |
| **Fan-out to multiple SQS queues** | SNS → SQS\_A, SQS\_B, SQS\_C (each service processes independently) |
| **Alerting**                       | CloudWatch Alarm → SNS → Email/SMS/Slack                            |
| **Application notifications**      | Order placed → SNS → email + SMS + push                             |
| **Cross-account events**           | SNS in central account → subscribers in other accounts              |
| **Mobile push notifications**      | App backend → SNS → APNS/FCM → millions of devices                  |
| **Broadcast config changes**       | Deploy → SNS → all consumers reload config                          |

### CLI Example

```bash
# Create topic
aws sns create-topic --name order-events

# Subscribe SQS queue
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789012:order-events \
  --protocol sqs \
  --notification-endpoint arn:aws:sqs:us-east-1:123456789012:orders-queue

# Subscribe email
aws sns subscribe --topic-arn arn:aws:sns:us-east-1:123456789012:order-events \
  --protocol email --notification-endpoint ops@example.com

# Publish
aws sns publish --topic-arn arn:aws:sns:us-east-1:123456789012:order-events \
  --message '{"orderId":"1001","status":"placed"}' \
  --message-attributes '{"eventType":{"DataType":"String","StringValue":"OrderPlaced"}}'
```

### Filter Policy (route messages to right subscriber)

```json
{
  "eventType": ["OrderPlaced", "OrderShipped"],
  "amount": [{"numeric": [">", 100]}]
}
```

Subscriber only receives matching messages — reduces cost & filtering logic.

### Best Practices

* ✅ Use **SNS + SQS fan-out pattern** for durable async processing.
* ✅ Configure **DLQ** on subscriptions.
* ✅ Use **filter policies** to avoid client-side filtering.
* ✅ Encrypt topics with **KMS**.
* ✅ Use **FIFO SNS** only when strict ordering required.
* ✅ Monitor `NumberOfNotificationsFailed`.

***

## 38. How Do SNS and SQS Differ and Complement Each Other?

Both are messaging services, but with **fundamentally different models** — they're often used together in the powerful **Fan-Out pattern**.

### SNS vs SQS

| Aspect                  | **SNS** (Pub/Sub)                     | **SQS** (Queue)                                |
| ----------------------- | ------------------------------------- | ---------------------------------------------- |
| **Model**               | Push (topic broadcasts)               | Pull (consumer polls)                          |
| **Consumers**           | Many (fan-out)                        | Typically one (or group processing same queue) |
| **Message Persistence** | No (delivered or dropped after retry) | Yes (retained 1 min–14 days)                   |
| **Delivery**            | Push to subscribers                   | Consumer pulls when ready                      |
| **Ordering**            | Best-effort (or FIFO topic)           | Best-effort (or FIFO queue)                    |
| **Retry**               | SNS retries per protocol              | Consumer retries via visibility timeout        |
| **Coupling**            | Loose — many consumers                | Loose — buffer between producer & consumer     |
| **Use Case**            | Broadcast events                      | Buffer work items                              |
| **Cost**                | $0.50 per M publishes                 | $0.40 per M requests                           |

### The Fan-Out Pattern — Best of Both Worlds

```
                            ┌─→ SQS Queue A → Order Service
                            │
Producer → SNS Topic ───────┼─→ SQS Queue B → Email Service
                            │
                            ├─→ SQS Queue C → Analytics Service
                            │
                            └─→ Lambda D → Slack notification
```

### Why This Pattern is Powerful

1. **Producer knows nothing about consumers** — full decoupling.
2. **Each consumer has its own SQS queue** → independent scaling, retry, DLQ.
3. **Slow consumer** doesn't block others (their queues buffer).
4. **Add/remove subscribers** without touching producers.
5. **Filter policies** target specific consumers.

### Direct SNS → Consumer (No SQS)

* ✅ Simpler for stateless notifications (email, SMS).
* ❌ If subscriber is down during publish → message may be lost (limited retries).
* ❌ No buffering — Lambda could get overwhelmed by a spike.

### Direct SQS (No SNS)

* ✅ Simple one-producer / one-consumer pattern.
* ❌ Only ONE application processes each message — no broadcast.

### When to Use Which

| Scenario                                         | Choice                          |
| ------------------------------------------------ | ------------------------------- |
| **One producer, one consumer, async processing** | SQS only                        |
| **One event, multiple consumers**                | SNS + SQS fan-out               |
| **Real-time notifications (email/SMS)**          | SNS direct                      |
| **Guaranteed durable async work**                | SNS → SQS (never lose messages) |
| **Ordered financial transactions**               | FIFO SNS → FIFO SQS             |
| **Stream processing at massive scale**           | Kinesis instead                 |

### SNS+SQS Fan-Out Setup Example

```bash
# Create SNS topic
TOPIC=$(aws sns create-topic --name order-events --query TopicArn --output text)

# Create SQS queues
QUEUE1=$(aws sqs create-queue --queue-name order-processor --query QueueUrl --output text)
QUEUE2=$(aws sqs create-queue --queue-name email-sender --query QueueUrl --output text)

# Subscribe queues to topic
aws sns subscribe --topic-arn $TOPIC --protocol sqs \
  --notification-endpoint arn:aws:sqs:us-east-1:123456789012:order-processor
aws sns subscribe --topic-arn $TOPIC --protocol sqs \
  --notification-endpoint arn:aws:sqs:us-east-1:123456789012:email-sender
```

Don't forget the **SQS queue policy** allowing SNS to send messages!

***

## 39. What is Amazon SES and Common Use Cases?

**Amazon SES (Simple Email Service)** is a **cloud-based email sending and receiving service** for transactional, marketing, and notification emails. It's cost-effective, scalable, and integrates with the rest of AWS.

### Key Features

* **Send transactional & marketing emails** at high scale.
* **Receive inbound emails** (SES → S3/Lambda/SNS).
* **Sender authentication** — DKIM, SPF, DMARC.
* **Dedicated IP pools** for warm-up and reputation isolation.
* **Configuration Sets** — group settings & tracking.
* **Bounce/complaint feedback** loops.
* **Cost**: $0.10 per 1,000 emails sent (from EC2: 62,000/month free).

### Common Use Cases

| Use Case                         | Example                                             |
| -------------------------------- | --------------------------------------------------- |
| **Transactional emails**         | Signup verification, password reset, order receipts |
| **Notifications**                | Alerts, invoices, monthly reports                   |
| **Marketing campaigns**          | Newsletters, promotions                             |
| **Application-triggered emails** | Backup completion, alarm notifications              |
| **Bulk email sending**           | SaaS platforms sending on behalf of users           |
| **Inbound email processing**     | `support@` → S3 → Lambda → ticket creation          |
| **Contact forms**                | Website form → Lambda → SES                         |

### SES Sending Modes

| Mode                  | Purpose                                             |
| --------------------- | --------------------------------------------------- |
| **Sandbox** (default) | Test only — verified recipients only, 200/day limit |
| **Production**        | Full sending — request via AWS Support              |
| **Dedicated IP**      | Isolated IPs for high-reputation sending            |
| **Shared IP**         | Cheaper, uses AWS pool                              |

### Sender Authentication Setup (mandatory for deliverability)

1. **Verify Domain** in SES → adds DKIM records to DNS.
2. **SPF Record** — `v=spf1 include:amazonses.com ~all`.
3. **DMARC Policy** — `v=DMARC1; p=quarantine; rua=mailto:dmarc@example.com`.

### Send Email — AWS SDK

```python
import boto3
ses = boto3.client('ses', region_name='us-east-1')

ses.send_email(
    Source='no-reply@example.com',
    Destination={'ToAddresses': ['user@example.com']},
    Message={
        'Subject': {'Data': 'Welcome to our service!'},
        'Body': {
            'Html': {'Data': '<h1>Welcome, Rajesh!</h1>'},
            'Text': {'Data': 'Welcome, Rajesh!'}
        }
    },
    ConfigurationSetName='TransactionalEmails'
)
```

### Handling Bounces & Complaints

Critical for reputation — subscribe SES to SNS topics:

```
SES → SNS (bounces) → Lambda → Update DB (mark email invalid)
SES → SNS (complaints) → Lambda → Unsubscribe user
```

### Configuration Set — Track Opens, Clicks, Bounces

Enable event publishing to CloudWatch/Kinesis/SNS for analytics.

### SES vs Other Email Services

| Feature                  | SES                        | SendGrid/Mailgun    |
| ------------------------ | -------------------------- | ------------------- |
| **Cost**                 | $0.10/1000                 | \$$$                |
| **AWS Integration**      | Native                     | External            |
| **Analytics**            | Basic (via CloudWatch)     | Rich dashboards     |
| **Deliverability tools** | Good                       | Excellent           |
| **Marketing UI**         | Basic                      | Full-featured       |
| **Best For**             | Transactional, dev-managed | Marketing, non-devs |

### Best Practices

* ✅ **Verify domain**, not just email address.
* ✅ Enable **DKIM + SPF + DMARC**.
* ✅ Handle **bounces + complaints** — auto-suppress bad addresses.
* ✅ Warm up **dedicated IPs** gradually.
* ✅ Segment **transactional vs marketing** into different configuration sets.
* ✅ Monitor **bounce rate < 5%** and **complaint rate < 0.1%** (SES may pause you above these).
* ✅ Use **VDM (Virtual Deliverability Manager)** for advanced insights.

***

## 40. What is EKS?

**Amazon EKS (Elastic Kubernetes Service)** is AWS's **fully managed Kubernetes service** that runs upstream, CNCF-conformant Kubernetes without needing to install, operate, or maintain the control plane.

### What AWS Manages vs What You Manage

| AWS Manages                                                     | You Manage                   |
| --------------------------------------------------------------- | ---------------------------- |
| Control plane (API server, etcd, scheduler, controller manager) | Worker nodes (EC2 / Fargate) |
| Multi-AZ HA of control plane                                    | Kubernetes manifests (YAML)  |
| Automatic patching of control plane                             | Application deployments      |
| etcd backups                                                    | RBAC, network policies       |
| Certificate rotation                                            | Monitoring, logging setup    |
| API server upgrades (assisted)                                  | Node group scaling           |

### EKS Architecture

```
┌──────────────────────────────────────────┐
│  AWS-Managed EKS Control Plane           │
│  (3 API servers + 3 etcd across 3 AZs)   │
└──────────────────────────────────────────┘
                    ▲
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
   ┌────────┐  ┌────────┐  ┌────────┐
   │ Worker │  │ Worker │  │ Fargate│
   │ Node   │  │ Node   │  │ Pod    │
   │(EC2/MNG)│  │(Karpenter)│ (serverless)│
   └────────┘  └────────┘  └────────┘
        AZ-1a      AZ-1b       AZ-1c
```

### Worker Node Options

| Option                        | Description                                                      | Best For                        |
| ----------------------------- | ---------------------------------------------------------------- | ------------------------------- |
| **Managed Node Groups (MNG)** | AWS provisions/manages EC2 nodes via ASG                         | Standard workloads              |
| **Self-managed Nodes**        | You manage EC2 lifecycle                                         | Custom AMIs, GPUs, Windows      |
| **Fargate**                   | Serverless pods — no nodes to manage                             | Bursty, small workloads         |
| **Karpenter**                 | Fast, flexible auto-scaler (recommended over Cluster Autoscaler) | Cost-optimized, mixed instances |

### Key EKS-Specific Features

| Feature                                   | Purpose                                       |
| ----------------------------------------- | --------------------------------------------- |
| **VPC CNI**                               | Pods get real VPC IPs (native networking)     |
| **IAM Roles for Service Accounts (IRSA)** | Fine-grained IAM per pod                      |
| **EKS Pod Identity**                      | Newer, simpler alternative to IRSA            |
| **AWS Load Balancer Controller**          | Manages ALB/NLB for Ingress/Services          |
| **EBS/EFS/FSx CSI Drivers**               | Persistent storage                            |
| **App Mesh / VPC Lattice**                | Service mesh                                  |
| **EKS Add-ons**                           | Managed vpc-cni, kube-proxy, CoreDNS, EBS CSI |
| **EKS Anywhere**                          | Run EKS on-prem                               |
| **EKS Distro**                            | Open-source Kubernetes distribution           |
| **Access Entries**                        | Modern IAM auth (replaces aws-auth configmap) |

### Create a Cluster with `eksctl` (fastest)

```bash
eksctl create cluster \
  --name prod-cluster \
  --region us-east-1 \
  --version 1.30 \
  --nodegroup-name workers \
  --node-type t3.large \
  --nodes 3 --nodes-min 2 --nodes-max 10 \
  --managed \
  --with-oidc \
  --ssh-access --ssh-public-key my-key
```

### Deploy an App

```bash
kubectl create deployment nginx --image=nginx:latest --replicas=3
kubectl expose deployment nginx --port=80 --type=LoadBalancer
```

### IRSA (IAM Roles for Service Accounts) — Critical Concept

Grants **fine-grained AWS permissions to specific pods**, not entire nodes.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/S3ReadRole
---
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      serviceAccountName: s3-reader
      containers:
        - name: app
          image: my-app:v1
```

### EKS vs ECS vs Self-Managed K8s

| Aspect                     | EKS                             | ECS                         | Self-managed K8s on EC2 |
| -------------------------- | ------------------------------- | --------------------------- | ----------------------- |
| **Control plane**          | AWS-managed ($0.10/hr)          | Free                        | You manage everything   |
| **Kubernetes conformance** | ✅ Upstream K8s                  | ❌ AWS-proprietary           | ✅ Upstream K8s          |
| **Portability**            | High (K8s standard)             | Low (AWS-only)              | High                    |
| **Ops overhead**           | Medium                          | Low                         | High                    |
| **Best for**               | Portable, complex microservices | AWS-only, simple containers | Full control needed     |

### Common EKS DevOps Stack

```
- Cluster: EKS 1.30 with Karpenter
- Ingress: AWS Load Balancer Controller (ALB)
- DNS: ExternalDNS + Route 53
- Certs: cert-manager + ACM
- Secrets: External Secrets Operator + Secrets Manager
- Monitoring: Prometheus + Grafana + CloudWatch Container Insights
- Logging: FluentBit → CloudWatch/OpenSearch
- Service Mesh: Istio or App Mesh
- GitOps: ArgoCD or Flux
- CI/CD: GitHub Actions / Azure DevOps → ECR → EKS
- Security: Falco + Kyverno/OPA + AWS Security Hub
```

### Best Practices

* ✅ Use **Managed Node Groups** or **Karpenter** — avoid self-managed nodes.
* ✅ Enable **IRSA / Pod Identity** — never attach broad IAM to nodes.
* ✅ Enable **EKS control plane logging** to CloudWatch (audit, authenticator, API server).
* ✅ Enable **envelope encryption** with KMS.
* ✅ Use **private endpoints** for the API server in prod.
* ✅ Apply **Pod Security Standards** (restricted profile).
* ✅ Enable **VPC CNI prefix delegation** for higher pod density.
* ✅ Use **GitOps** (ArgoCD/Flux) — no manual `kubectl apply`.
* ✅ Regularly upgrade Kubernetes (AWS supports N-3 minor versions).

### EKS Pricing

* **Control plane**: $0.10/hour per cluster (\~$73/month).
* **Worker nodes**: standard EC2/Fargate pricing.
* **Data transfer**: standard AWS rates.
* **Add-ons like Load Balancers**: standard ELB pricing.

***

## Summary Cheat Sheet (Q31–Q40)

| #  | Concept                  | One-Liner                                                                 |
| -- | ------------------------ | ------------------------------------------------------------------------- |
| 31 | **CloudWatch Logs**      | Centralized log aggregation, search, retention, insights                  |
| 32 | **CloudWatch Metrics**   | Numeric time-series store; standard + custom metrics                      |
| 33 | **CloudWatch Alarms**    | Threshold-based automation (notify/scale/remediate)                       |
| 34 | **CloudTrail**           | API audit log (who did what, when, from where)                            |
| 35 | **SQS**                  | Managed queue for decoupling producers & consumers                        |
| 36 | **Standard vs FIFO SQS** | Standard = high TPS, best-effort order; FIFO = strict order, exactly-once |
| 37 | **SNS**                  | Pub/sub topic broadcasting to many subscribers                            |
| 38 | **SNS + SQS**            | Fan-out pattern — SNS broadcasts, SQS durably buffers per consumer        |
| 39 | **SES**                  | Managed email sending/receiving service                                   |
| 40 | **EKS**                  | AWS-managed Kubernetes control plane + worker options (EC2/Fargate)       |

***

## Bonus: Interview Scenarios

**Q:** *"You want to detect if someone disables CloudTrail. How?"*
**A:** EventBridge rule matching `eventName IN ("StopLogging","DeleteTrail","UpdateTrail")` → SNS → PagerDuty. Also enable **CloudTrail log file validation** and **AWS Config rule** `cloud-trail-enabled`. Store logs in a **separate logging account** with S3 Object Lock.

**Q:** *"An SQS consumer keeps timing out on a poison message. How to handle?"*
**A:** Configure a **DLQ** with `maxReceiveCount=3`. Poison message moves to DLQ after 3 failures. Alert on DLQ depth > 0. Add manual reprocessing via Lambda or `redrive-message-batch`.

**Q:** *"Design a system where an order event triggers: (a) database update, (b) email confirmation, (c) analytics ingest, (d) inventory update."*
**A:** SNS topic `order-events` with 4 SQS subscribers → each backed by a Lambda/service. Use filter policies. DLQ per queue. Bonus: FIFO SNS+SQS if strict ordering matters.

**Q:** *"How do you give an EKS pod access to an S3 bucket without hardcoding AWS keys?"*
**A:** **IRSA (IAM Roles for Service Accounts)** or **EKS Pod Identity** — create IAM role with S3 policy, associate with a Kubernetes ServiceAccount, mount SA to Pod. AWS SDK auto-retrieves temp creds via OIDC/STS.

**Q:** *"You need to send 10 million marketing emails/day and track opens & clicks. How?"*
**A:** SES with **dedicated IP pool** (warmed up), separate **configuration set** for marketing, enable **event publishing to Firehose → S3 → Athena/QuickSight**, use **SNS bounce/complaint handling** to auto-suppress, monitor bounce/complaint rates.

***

# AWS Container, Scaling, Networking & Edge Services – Detailed Interview-Ready Answers (Q41–Q50)

Here's a comprehensive walkthrough of your next 10 questions covering **EKS vs ECS, Auto Scaling, ELB, Route 53, Lambda cold starts, Direct Connect, VPN, and CloudFront** — structured for a **Senior DevOps interview** with practical examples, CLI snippets, architecture patterns, and real-world scenarios.

***

## 41. What is the Difference Between EKS and ECS?

Both are AWS-managed container orchestration services, but with **fundamentally different philosophies**: **ECS is AWS-native** and simple; **EKS is upstream Kubernetes** and portable.

### Core Comparison

| Aspect                 | **ECS (Elastic Container Service)** | **EKS (Elastic Kubernetes Service)**      |
| ---------------------- | ----------------------------------- | ----------------------------------------- |
| **Orchestrator**       | AWS-proprietary                     | Upstream Kubernetes (CNCF-conformant)     |
| **Launch Types**       | EC2, Fargate                        | EC2, Fargate, Hybrid (Outposts, Anywhere) |
| **Learning Curve**     | Low — AWS Console/CLI               | Steep — kubectl, YAML, K8s concepts       |
| **Control Plane Cost** | Free                                | $0.10/hr (\~$73/mo per cluster)           |
| **Portability**        | ❌ AWS-only                          | ✅ Runs anywhere (on-prem, other clouds)   |
| **Community**          | AWS-only                            | Massive CNCF ecosystem                    |
| **Config Language**    | Task Definitions (JSON)             | Kubernetes YAML manifests                 |
| **Networking**         | awsvpc / bridge / host              | VPC CNI (pods get real ENI IPs)           |
| **Service Mesh**       | AWS App Mesh                        | Istio, Linkerd, App Mesh, VPC Lattice     |
| **Ecosystem Tools**    | Limited (Copilot, AWS-native)       | Helm, ArgoCD, Flux, Kustomize, Prometheus |
| **Multi-tenancy**      | Basic                               | Rich (namespaces, RBAC, network policies) |
| **Ideal Team Size**    | Small teams, AWS-only shops         | Larger DevOps orgs, platform teams        |
| **Upgrade Model**      | AWS handles (transparent)           | You manage cluster upgrades (N-3 policy)  |

### ECS Key Concepts

* **Task Definition** — blueprint (like Docker Compose).
* **Task** — running instance of a task definition.
* **Service** — maintains desired count of tasks (like Deployment).
* **Cluster** — logical grouping of tasks/services.
* **Capacity Provider** — EC2 ASG or Fargate.

### EKS Key Concepts

* **Cluster** — control plane + worker nodes.
* **Pod** — smallest deployable unit.
* **Deployment / ReplicaSet / StatefulSet / DaemonSet** — workload types.
* **Service** — stable networking abstraction.
* **Ingress** — L7 routing.
* **Namespace** — logical isolation.

### When to Choose Which

| Scenario                                          | Choose                      |
| ------------------------------------------------- | --------------------------- |
| **AWS-only shop, small team, quick start**        | **ECS**                     |
| **Multi-cloud or hybrid strategy**                | **EKS**                     |
| **Team already knows Kubernetes**                 | **EKS**                     |
| **Complex microservices with service mesh needs** | **EKS**                     |
| **Simple web apps, batch jobs**                   | **ECS + Fargate**           |
| **Need Helm charts / operators / CRDs**           | **EKS**                     |
| **Regulatory/portability requirements**           | **EKS**                     |
| **Minimize ops overhead**                         | **ECS + Fargate**           |
| **Cost-sensitive small workload**                 | ECS (no control plane cost) |

### Real-World Example

* **Startup with 5 microservices, AWS-only** → **ECS + Fargate** (fastest path).
* **Enterprise with 200 microservices, on-prem + AWS** → **EKS** (Kubernetes portability + tooling ecosystem).

### Cost Comparison Example (10 small services)

|                     | ECS + Fargate | EKS + Fargate |
| ------------------- | ------------- | ------------- |
| Control plane       | $0            | \~$73/mo      |
| Fargate pods (same) | Same          | Same          |
| Ops effort          | Low           | Medium-High   |

For your **Senior DevOps interview prep** — expect follow-up questions like:

* *"Can EKS run Windows containers?"* → Yes (managed Windows node groups).
* *"How does IRSA differ from ECS Task Role?"* → IRSA uses OIDC federation with fine-grained pod-level roles; ECS Task Role is task-level via ECS agent.

***

## 42. What is Auto Scaling and How Does It Work with EC2?

**AWS Auto Scaling** dynamically adjusts capacity to maintain performance and cost efficiency. **EC2 Auto Scaling** specifically manages **groups of EC2 instances** — automatically launching/terminating them based on demand.

### Core Components

| Component                    | Description                                                         |
| ---------------------------- | ------------------------------------------------------------------- |
| **Launch Template**          | Blueprint for new instances (AMI, type, SG, IAM role, user data)    |
| **Auto Scaling Group (ASG)** | Logical group of EC2s with min/desired/max capacity                 |
| **Scaling Policies**         | Rules that trigger scaling actions                                  |
| **Health Checks**            | EC2 status + ELB health                                             |
| **Cooldown Period**          | Wait time between scaling actions                                   |
| **Lifecycle Hooks**          | Pause instance launch/terminate for custom actions                  |
| **Termination Policies**     | Which instance to kill first (OldestInstance, NewestInstance, etc.) |
| **Warm Pools**               | Pre-initialized instances (faster scaling)                          |

### Scaling Policy Types

| Type                   | Behavior                                           | Best For                             |
| ---------------------- | -------------------------------------------------- | ------------------------------------ |
| **Target Tracking**    | Maintain a metric at target value (e.g., CPU=50%)  | Most workloads — recommended default |
| **Step Scaling**       | Add/remove N instances based on alarm severity     | Variable load with tiered thresholds |
| **Simple Scaling**     | Add/remove N on alarm; wait for cooldown           | Legacy — avoid                       |
| **Scheduled Scaling**  | Scale at specific times (e.g., 9 AM: 10 instances) | Predictable patterns                 |
| **Predictive Scaling** | ML-based forecasting from historical data          | Recurring daily/weekly patterns      |

### How Auto Scaling Works — Flow

```
1. CloudWatch metric crosses threshold (e.g., CPU > 70%)
2. Scaling policy triggers → ASG increases desired capacity
3. ASG launches new EC2 from Launch Template
4. Instance passes EC2 status checks + ELB health checks
5. ASG registers instance with ELB target group
6. Traffic starts flowing to new instance
7. Cooldown period prevents rapid oscillation
```

### Auto Scaling Group Sizes

| Parameter   | Meaning                               |
| ----------- | ------------------------------------- |
| **Minimum** | Never fewer than this (baseline)      |
| **Desired** | Current target (adjusted by policies) |
| **Maximum** | Never more than this (cost guardrail) |

### Real-World Setup (CLI)

```bash
# Create Launch Template
aws ec2 create-launch-template \
  --launch-template-name web-lt \
  --version-description v1 \
  --launch-template-data '{
    "ImageId": "ami-0abc",
    "InstanceType": "t3.medium",
    "IamInstanceProfile": {"Name": "EC2-SSM-Role"},
    "SecurityGroupIds": ["sg-0web"],
    "UserData": "IyEvYmluL2Jhc2gKZG5mIHVwZGF0ZSAteQpkbmYgaW5zdGFsbCAteSBuZ2lueA=="
  }'

# Create Auto Scaling Group
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name web-asg \
  --launch-template LaunchTemplateName=web-lt,Version='$Latest' \
  --min-size 2 --max-size 10 --desired-capacity 3 \
  --vpc-zone-identifier "subnet-0a,subnet-0b,subnet-0c" \
  --target-group-arns arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/web-tg/abc \
  --health-check-type ELB --health-check-grace-period 300

# Target Tracking Policy — keep CPU at 50%
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name web-asg \
  --policy-name cpu-target-50 \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification": {"PredefinedMetricType": "ASGAverageCPUUtilization"},
    "TargetValue": 50.0
  }'
```

### Best Practices

* ✅ **Use Target Tracking** by default (self-adjusting).
* ✅ Deploy across **≥ 3 AZs** for HA.
* ✅ Use **ELB health checks** (not just EC2 status).
* ✅ Set **health check grace period** ≥ boot time.
* ✅ Use **Mixed Instances Policy** for Spot + On-Demand cost optimization.
* ✅ Enable **Instance Refresh** for rolling AMI updates.
* ✅ Use **Lifecycle Hooks** to drain connections before termination.
* ✅ Use **Warm Pools** for faster scaling (pre-warmed but stopped instances).
* ✅ Enable **Predictive Scaling** for predictable traffic patterns.

### Application Auto Scaling

Beyond EC2, applies to:

* ECS Services
* DynamoDB read/write capacity
* Aurora Replicas
* Spot Fleet
* Lambda Provisioned Concurrency
* SageMaker endpoints
* Neptune, ElastiCache, and more

***

## 43. What is an Elastic Load Balancer (ELB)?

**Elastic Load Balancer (ELB)** is a **fully managed AWS service** that distributes incoming traffic across multiple targets (EC2, containers, IPs, Lambda) in one or more AZs. It's the entry point for most highly available AWS applications.

### Why Use an ELB?

* **High Availability** — traffic routed to healthy targets across AZs.
* **Fault Tolerance** — unhealthy instances are auto-excluded.
* **Scalability** — automatically scales to millions of requests.
* **Security** — SSL/TLS termination, WAF integration, SG-based access.
* **Elasticity** — no capacity provisioning needed.

### Core Components

| Component                     | Description                                      |
| ----------------------------- | ------------------------------------------------ |
| **Listener**                  | Port + protocol combination (e.g., HTTPS:443)    |
| **Target Group**              | Set of registered targets (EC2, IP, Lambda, ECS) |
| **Health Check**              | Periodic probe to determine target health        |
| **Rules**                     | Routing conditions (ALB only)                    |
| **Cross-Zone Load Balancing** | Distribute evenly across all AZ targets          |
| **Sticky Sessions**           | Route same client to same target                 |
| **Connection Draining**       | Graceful shutdown (deregistration delay)         |

### High-Level Flow

```
Client → Route 53 (DNS)
          ↓
        ELB (public/internal)
          ↓
     Target Group → Health Check
          ↓
    ┌─────┴─────┐
    ▼           ▼
  EC2 in     EC2 in
  AZ-1a      AZ-1b
```

### Key Features Across All ELB Types

* Multi-AZ HA
* IPv4/IPv6
* CloudWatch metrics + access logs
* Integration with Auto Scaling
* WAF integration (ALB/CloudFront)
* Server-side TLS termination
* Access logs to S3
* AWS Certificate Manager (ACM) for free SSL certs

### Common Pricing

* **Hourly + LCU** (Load Balancer Capacity Units) — usage-based charge for new connections, active connections, processed bytes, rules.
* Roughly **$16–25/month** for basic loads + traffic charges.

### Best Practices

* ✅ Use **HTTPS listener** with ACM-issued certs.
* ✅ Enable **access logging** to S3 for forensics.
* ✅ Set **health check path** to a lightweight endpoint (`/health`).
* ✅ Use **HTTP → HTTPS redirect** at ALB level.
* ✅ Attach **WAF** for L7 protection (ALB/CloudFront).
* ✅ Enable **cross-zone load balancing** (free on ALB, chargeable on NLB).
* ✅ Configure **deregistration delay** = your longest request duration.

***

## 44. Types of ELB (Classic, ALB, NLB, GWLB)

AWS offers **four ELB types**, each optimized for different use cases and network layers.

### The Four Types

| ELB Type                            | Layer | Protocol                     | Best For                           |
| ----------------------------------- | ----- | ---------------------------- | ---------------------------------- |
| **Classic Load Balancer (CLB)**     | 4 & 7 | HTTP, HTTPS, TCP, SSL        | Legacy — avoid for new workloads   |
| **Application Load Balancer (ALB)** | 7     | HTTP, HTTPS, gRPC, WebSocket | Modern web apps, microservices     |
| **Network Load Balancer (NLB)**     | 4     | TCP, UDP, TLS                | Extreme perf, static IPs, non-HTTP |
| **Gateway Load Balancer (GWLB)**    | 3     | IP                           | Third-party firewalls/IDS/IPS      |

### Detailed Comparison

| Feature                 | **CLB**        | **ALB**                   | **NLB**                    | **GWLB**         |
| ----------------------- | -------------- | ------------------------- | -------------------------- | ---------------- |
| **OSI Layer**           | 4 & 7          | 7 (App)                   | 4 (Transport)              | 3 (Network)      |
| **Protocols**           | HTTP/HTTPS/TCP | HTTP/HTTPS/gRPC/WebSocket | TCP/UDP/TLS                | GENEVE           |
| **Static IP**           | ❌              | ❌                         | ✅ (Elastic IP per AZ)      | ❌                |
| **Path/Host Routing**   | ❌              | ✅                         | ❌                          | ❌                |
| **Preserves Source IP** | ❌              | ❌ (X-Forwarded-For)       | ✅                          | ✅                |
| **Latency**             | Moderate       | \~10 ms                   | Ultra-low (\~100 μs)       | Adds latency     |
| **Requests/sec**        | 10K            | Millions                  | Millions                   | Millions         |
| **Sticky Sessions**     | Duration-based | Duration + app-cookie     | Source IP                  | N/A              |
| **WebSocket**           | ❌              | ✅                         | ✅                          | N/A              |
| **Lambda Targets**      | ❌              | ✅                         | ❌                          | ❌                |
| **IP as Target**        | ❌              | ✅                         | ✅                          | ✅                |
| **HTTP/2 & gRPC**       | ❌              | ✅                         | ❌                          | N/A              |
| **Container-aware**     | ❌              | ✅                         | ✅                          | N/A              |
| **SSL Offload**         | ✅              | ✅                         | ✅ (TLS listener)           | ❌                |
| **WAF Integration**     | ❌              | ✅                         | ❌                          | ❌                |
| **Cost**                | Legacy pricing | Moderate                  | Higher for high throughput | Per GB inspected |

### Application Load Balancer (ALB) — Deep Dive

**Layer 7** — inspects HTTP headers, cookies, URL paths.

**Advanced Routing Rules:**

* **Path-based**: `/api/*` → API target group, `/*` → web target group.
* **Host-based**: `api.example.com` vs `www.example.com`.
* **HTTP header**: `X-Version: v2` → new version.
* **Query string**: `?beta=true` → canary.
* **Source IP**: internal-only routing.
* **HTTP method**: `POST` vs `GET`.

**Key Features:**

* WebSocket & HTTP/2 support.
* Redirect actions (HTTP → HTTPS).
* Fixed response actions (return static 200/404).
* Authentication via Cognito/OIDC.
* Lambda as target.
* Weighted target groups (canary/blue-green).
* Server Name Indication (SNI) — multiple certs per listener.

### Network Load Balancer (NLB) — Deep Dive

**Layer 4** — pure TCP/UDP forwarding.

**When to Use:**

* Non-HTTP protocols (gaming, IoT, MQTT, VoIP).
* Ultra-low latency (< 1 ms).
* Millions of concurrent connections.
* **Need static IPs / Elastic IPs** (whitelisting, PrivateLink).
* Preserve **client source IP** end-to-end.
* Handle TLS termination for TCP protocols.

### Gateway Load Balancer (GWLB) — Deep Dive

**Layer 3** — transparently inserts third-party network appliances into your traffic path.

**Use Cases:**

* Palo Alto, Fortinet, Check Point firewalls in AWS.
* Intrusion Detection/Prevention Systems (IDS/IPS).
* Deep Packet Inspection (DPI).
* Centralized traffic inspection VPC.

### Classic Load Balancer — Legacy

* Predates ALB/NLB.
* Still supported but **not recommended for new deployments**.
* No advanced features (path routing, containers, etc.).
* Migrate to ALB/NLB.

### Choosing the Right ELB

```
Is it HTTP/HTTPS?
├─ Yes → ALB (99% of web apps)
└─ No →
    Need static IP or ultra-low latency?
    ├─ Yes → NLB
    └─ No →
        Inserting 3rd-party security appliance?
        ├─ Yes → GWLB
        └─ No → NLB
```

### Best Practices

* ✅ **ALB** for web/microservices (default choice).
* ✅ **NLB** for TCP/UDP, IoT, gaming, whitelisted IPs.
* ✅ Migrate **CLB → ALB** for legacy.
* ✅ Use **AWS Global Accelerator + NLB** for global low-latency routing.
* ✅ Combine **CloudFront + ALB** for edge caching + origin routing.

***

## 45. What is Route 53 and What is it Used For?

**Amazon Route 53** is AWS's **highly available and scalable DNS + domain registration + health-checking service**. It's named after DNS port 53. It handles both public and private DNS zones.

### Core Capabilities

| Capability                   | Description                                         |
| ---------------------------- | --------------------------------------------------- |
| **Domain Registration**      | Buy/manage domains directly (.com, .in, .io, etc.)  |
| **Authoritative DNS**        | Answer DNS queries for your domains                 |
| **Traffic Routing Policies** | Advanced routing (weighted, latency, geo, failover) |
| **Health Checks**            | Monitor endpoint health globally                    |
| **Private DNS**              | Internal DNS for VPCs (no internet exposure)        |
| **DNS Resolver**             | Route 53 Resolver for hybrid DNS with on-prem       |
| **DNSSEC**                   | Cryptographic signing of DNS responses              |
| **Traffic Flow**             | Visual policy editor for complex routing            |

### Route 53 SLA

* **100% availability SLA** (only AWS service with 100%!).

### Supported Record Types

| Record               | Purpose                                              |
| -------------------- | ---------------------------------------------------- |
| **A**                | IPv4 address                                         |
| **AAAA**             | IPv6 address                                         |
| **CNAME**            | Canonical name (alias to another DNS name)           |
| **MX**               | Mail exchange                                        |
| **TXT**              | Arbitrary text (SPF, DMARC, domain verification)     |
| **NS**               | Nameservers                                          |
| **SOA**              | Start of Authority                                   |
| **PTR**              | Reverse lookup                                       |
| **SRV**              | Service records                                      |
| **CAA**              | Which CAs can issue certs                            |
| **Alias** (AWS-only) | Route to AWS resources (ELB, CloudFront, S3) — free! |

### Alias Records — Route 53's Superpower

Unlike CNAME, **Alias records**:

* Work at the **zone apex** (root domain `example.com`, not just `www.example.com`).
* Are **free** (no query charge for AWS-resource aliases).
* Auto-track IP changes of the target AWS resource.
* Support ELB, CloudFront, S3 website, API Gateway, Global Accelerator, VPC endpoints.

### Routing Policies

| Policy                | Behavior                                       | Use Case                                 |
| --------------------- | ---------------------------------------------- | ---------------------------------------- |
| **Simple**            | Single value                                   | Basic DNS                                |
| **Weighted**          | % of traffic per record                        | Canary deployments, A/B testing          |
| **Latency-based**     | Lowest-latency AWS region                      | Global apps                              |
| **Failover**          | Primary → secondary on health check fail       | Active-passive DR                        |
| **Geolocation**       | Route by user's country/continent              | Compliance, localization                 |
| **Geoproximity**      | Route by geographic distance with bias         | Advanced geo routing (Traffic Flow only) |
| **Multivalue Answer** | Up to 8 healthy records returned (like DNS RR) | Basic client-side LB                     |
| **IP-based**          | Route by client subnet                         | ISP-specific routing                     |

### Common Use Cases

| Use Case                         | Example                                         |
| -------------------------------- | ----------------------------------------------- |
| **Public website DNS**           | `www.example.com` → CloudFront                  |
| **Blue/Green deployments**       | Weighted: 90% to prod, 10% to green             |
| **Global multi-region app**      | Latency routing to nearest region               |
| **DR failover**                  | Health check on primary; auto-fail to us-west-2 |
| **Private DNS in VPC**           | `db.internal` → RDS endpoint                    |
| **Hybrid DNS**                   | Route 53 Resolver → on-prem AD                  |
| **Third-party service SPF/DKIM** | TXT records                                     |
| **Domain verification**          | TXT records for Google/Microsoft                |

### Setup Alias Record (CLI)

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890 \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "www.example.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z35SXDOTRQ7X7K",
          "DNSName": "my-alb-123.us-east-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'
```

### Health Checks

Route 53 can monitor:

* **HTTP/HTTPS/TCP** endpoints.
* **CloudWatch alarms**.
* **Calculated health checks** (combine multiple).

Used with **failover routing** for auto-DR.

### Best Practices

* ✅ Use **Alias records** for AWS resources (free + faster).
* ✅ Enable **DNSSEC** for public zones.
* ✅ Set appropriate **TTL** (60s for dynamic, 3600s for static).
* ✅ Use **Traffic Flow** for complex multi-region setups.
* ✅ Enable **query logging** to CloudWatch (audit + troubleshooting).
* ✅ Use **private hosted zones** for internal services (never expose to internet).
* ✅ Use **failover routing + health checks** for HA.

***

## 46. What is a Hosted Zone in Route 53?

A **Hosted Zone** is a **container for DNS records** for a specific domain (and its subdomains). It's Route 53's equivalent of a DNS zone file.

### Types of Hosted Zones

| Type                    | Scope               | Access               | Use Case                          |
| ----------------------- | ------------------- | -------------------- | --------------------------------- |
| **Public Hosted Zone**  | Internet-accessible | Anyone on internet   | Public websites (`example.com`)   |
| **Private Hosted Zone** | VPC-only            | Associated VPCs only | Internal services (`db.internal`) |

### Public Hosted Zone

* Answers queries from the **public internet**.
* Assigned **4 unique NS records** by AWS.
* You must **update your domain registrar's nameservers** to point to these NS records.
* **Cost**: $0.50/month per hosted zone + $0.40/million queries.

### Private Hosted Zone

* Only resolves within **associated VPCs**.
* Requires **DNS support** and **DNS hostnames** enabled on VPC.
* Great for microservices communication, internal databases.
* Can associate **multiple VPCs** across accounts (with resource sharing).
* **Cost**: Same as public zones.

### Hosted Zone Structure

```
Hosted Zone: example.com
├── NS records (auto-created)
├── SOA record (auto-created)
├── A record: www → 1.2.3.4
├── A alias: example.com → ALB
├── MX: → smtp.example.com
├── TXT: SPF/DMARC records
├── CNAME: blog → hosted-blog.io
└── Subdomains:
    ├── api.example.com → API Gateway
    └── admin.example.com → CloudFront
```

### Create a Hosted Zone (CLI)

**Public:**

```bash
aws route53 create-hosted-zone \
  --name example.com \
  --caller-reference $(date +%s)
```

**Private (VPC-associated):**

```bash
aws route53 create-hosted-zone \
  --name internal.example.com \
  --caller-reference $(date +%s) \
  --vpc VPCRegion=us-east-1,VPCId=vpc-0abc \
  --hosted-zone-config PrivateZone=true
```

### Delegating Subdomains

You can delegate `dev.example.com` to a separate hosted zone (or another team's account).

Parent zone has NS records:

```
dev.example.com  NS  ns-abc.awsdns-01.com
                 NS  ns-xyz.awsdns-45.net
```

### Split-Horizon DNS

Same domain name resolves differently based on where the query originates:

* **Public zone** for `example.com` → CloudFront IP.
* **Private zone** for `example.com` → internal ALB IP (in VPC).

Route 53 automatically picks the right zone based on requester (VPC vs internet).

### Best Practices

* ✅ Use **private hosted zones** for all internal traffic.
* ✅ Enable **query logging** for debugging + audit.
* ✅ Use **DNSSEC** on public zones.
* ✅ Automate hosted zone changes via **Terraform/CloudFormation**.
* ✅ Keep **NS record TTL high** (172800s default) — rarely change.

***

## 47. What is AWS Lambda Cold Start?

**Cold Start** is the additional latency incurred when Lambda has to **initialize a new execution environment** to handle an invocation — because no warm one is available. It's the biggest performance concern in serverless design.

### Lifecycle of a Lambda Invocation

```
COLD START (only on new environment):
1. Download function code (from S3 / container registry)
2. Provision execution environment (micro-VM)
3. Bootstrap language runtime (Python/Node/Java)
4. Run INIT phase (imports, connections, static init)
5. Invoke handler
   [Warm starts skip steps 1–4]
```

### Typical Cold Start Latency

| Runtime              | Cold Start  | Warm Invocation |
| -------------------- | ----------- | --------------- |
| **Node.js**          | 100–400 ms  | < 10 ms         |
| **Python**           | 100–500 ms  | < 10 ms         |
| **Go** (compiled)    | 100–300 ms  | < 5 ms          |
| **Java**             | 500–5000 ms | < 20 ms         |
| **.NET**             | 500–3000 ms | < 20 ms         |
| **Rust**             | 50–200 ms   | < 5 ms          |
| **Custom container** | 200–2000 ms | Depends         |

### When Cold Starts Happen

1. **First invocation** ever.
2. **First invocation after idle** (\~15 min timeout typically).
3. **Scaling up** — concurrent requests exceed warm instances.
4. **Code/config update** — deployment invalidates warm envs.
5. **VPC-attached Lambda** — historically much slower (mostly fixed by AWS in 2019 via Hyperplane ENIs).

### Cold Start Contributors

| Factor                     | Impact                                    |
| -------------------------- | ----------------------------------------- |
| **Language runtime**       | Java/.NET slowest, Rust/Go fastest        |
| **Package size**           | Larger ZIP = slower download              |
| **Number of dependencies** | More imports = longer init                |
| **VPC**                    | ENI attachment adds \~100 ms (better now) |
| **Memory setting**         | Higher memory = more CPU = faster init    |
| **Container image size**   | Larger images take longer                 |
| **Static initialization**  | Heavy work in constructor/module level    |

### Mitigation Strategies

**1. Provisioned Concurrency** — pre-warm N instances (eliminates cold start).

```bash
aws lambda put-provisioned-concurrency-config \
  --function-name myApi \
  --qualifier prod \
  --provisioned-concurrent-executions 10
```

* ✅ Best mitigation.
* ❌ Costs even when idle.

**2. Lambda SnapStart** (Java 11+, Python 3.12+, .NET 8+)

* Snapshots initialized VM → resumes on invocation.
* **\~10× cold-start improvement** for Java.
* Free feature.

**3. Reduce Package Size**

* Strip unused dependencies.
* Use **Lambda Layers** for shared libs.
* Tree-shake JS bundles.

**4. Use Lightweight Runtimes**

* Prefer **Node.js/Python/Go/Rust** over Java/.NET for latency-sensitive.

**5. Warm-up Invocations** (legacy approach)

* Schedule EventBridge cron every 5 min → invoke Lambda with dummy event.
* Less clean than Provisioned Concurrency but cheaper.

**6. Optimize Initialization**

* Move heavy imports outside handler (cached across warm invocations).
* Lazy-load rarely used modules.
* Reuse connections (DB, HTTP clients) outside handler.

**7. Increase Memory**

* More memory → proportionally more CPU → faster init.
* Test 512 MB, 1024 MB, 2048 MB.

**8. Avoid VPC when possible**

* If Lambda doesn't need private resources, don't attach it to VPC.

### Real-World Design Recommendations

| Use Case                           | Approach                                              |
| ---------------------------------- | ----------------------------------------------------- |
| **User-facing API (< 100 ms P99)** | Provisioned Concurrency + Node/Python + memory tuning |
| **Async workloads**                | Ignore cold starts — non-user-facing                  |
| **Java Spring Boot APIs**          | SnapStart + Provisioned Concurrency                   |
| **Batch/ETL**                      | Don't optimize cold starts — save cost                |
| **Global apps**                    | Lambda\@Edge / CloudFront Functions for tiny latency  |

### Cold Start Detection in Lambda

```python
import time

# INIT phase (module load) — cold start work here
init_start = time.time()
import boto3, json
s3 = boto3.client('s3')
print(f"INIT took {time.time() - init_start:.3f}s")

def lambda_handler(event, context):
    # HANDLER phase
    return {"statusCode": 200}
```

CloudWatch REPORT log shows `Init Duration:` only on cold starts.

### Best Practices Summary

* ✅ Use **Provisioned Concurrency** for latency-critical APIs.
* ✅ Use **SnapStart** for Java/.NET/Python.
* ✅ Right-size **memory** — 1769 MB gives 1 full vCPU.
* ✅ Move imports/static config **outside handler**.
* ✅ Reuse SDK clients across invocations.
* ✅ Monitor `IteratorAge`, `Duration`, `Init Duration` in CloudWatch.

***

## 48. What is AWS Direct Connect?

**AWS Direct Connect (DX)** is a **dedicated private network connection** between your on-premises data center (or colocation facility) and AWS — bypassing the public internet entirely.

### Key Characteristics

* **Dedicated fiber connection** (1 Gbps, 10 Gbps, or 100 Gbps).
* **Private, low-latency, high-throughput**.
* **Consistent bandwidth & latency** (unlike internet VPN).
* **Delivered through AWS Direct Connect locations** (100+ globally — Equinix, Digital Realty, etc.).
* **BGP routing** between your router and AWS.
* **Higher cost than VPN**, lower long-term cost for high-traffic use cases.

### How It Works

```
┌────────────────────┐        ┌────────────────────┐
│ Your Data Center   │        │  AWS Direct        │
│                    │───────►│  Connect Location  │──► AWS Region
│  Customer Router   │  Fiber │  Cross-connect     │    (VPCs, S3)
└────────────────────┘        └────────────────────┘
```

### Connection Types

| Type                         | Speed             | Use                           |
| ---------------------------- | ----------------- | ----------------------------- |
| **Dedicated Connection**     | 1/10/100 Gbps     | Direct port at DX location    |
| **Hosted Connection**        | 50 Mbps – 10 Gbps | Via AWS Partner Network (APN) |
| **Hosted Virtual Interface** | Any               | Shared partner infra          |

### Virtual Interfaces (VIFs)

| VIF Type        | Access To                                                         |
| --------------- | ----------------------------------------------------------------- |
| **Private VIF** | Your VPCs (via Virtual Private Gateway or Direct Connect Gateway) |
| **Public VIF**  | AWS public services (S3, DynamoDB) via public IPs                 |
| **Transit VIF** | Transit Gateway (multi-VPC/multi-region)                          |

### Direct Connect Gateway

Connects one DX connection to **multiple VPCs across multiple regions** globally.

### Use Cases

| Use Case                 | Why DX                                       |
| ------------------------ | -------------------------------------------- |
| **Hybrid cloud**         | Consistent, private network to on-prem       |
| **Data migration**       | Terabytes/day (faster than internet)         |
| **Regulated industries** | Finance/healthcare — no public internet      |
| **Real-time apps**       | Low, predictable latency                     |
| **Big data transfer**    | Cheaper egress than internet ($/GB)          |
| **Multicast workloads**  | Media/broadcasting                           |
| **VDI / Citrix**         | Desktop virtualization requiring low latency |

### DX vs Internet

| Aspect                   | Direct Connect                        | Public Internet           |
| ------------------------ | ------------------------------------- | ------------------------- |
| **Bandwidth**            | Guaranteed 1/10/100 Gbps              | Variable                  |
| **Latency**              | Predictable, low                      | Variable                  |
| **Security**             | Private (still encrypt in transit!)   | Public (needs encryption) |
| **Cost (large traffic)** | Cheaper egress ($0.02/GB vs $0.09/GB) | High egress cost          |
| **Setup time**           | Weeks to months                       | Immediate                 |
| **HA**                   | Need 2 connections                    | Not applicable            |

### HA Design for Direct Connect

**Best Practice**: Dual DX connections at **different DX locations**, plus **VPN as backup**.

```
DC1 ─── DX1 (Ashburn) ─────────┐
                               ├─► AWS Region
DC2 ─── DX2 (Reston)  ─────────┘
                               ⇅
                               VPN (backup)
```

### DX Pricing

* **Port hours** (\~$0.30/hr for 1 Gbps, $2.25/hr for 10 Gbps).
* **Data transfer out** (much cheaper than internet — $0.02/GB vs $0.09/GB).
* **Cross-connect fees** (paid to colo provider).
* **Partner fees** (for hosted connections).

### Best Practices

* ✅ **Two connections** at **different DX locations** for HA.
* ✅ **VPN as backup** to DX (BGP failover).
* ✅ Use **Direct Connect Gateway** for multi-region VPC access.
* ✅ Encrypt data even over private DX (defense in depth).
* ✅ Use **MACsec encryption** on 10/100 Gbps for L2 encryption.
* ✅ Monitor **DX metrics** in CloudWatch (`ConnectionState`, `BitsInFromCustomer`).

***

## 49. What is AWS-Managed VPN and When Do You Use It?

**AWS Site-to-Site VPN** creates a **secure, encrypted IPsec tunnel** between your on-prem network and your AWS VPC — over the **public internet**.

### Components

| Component                         | Description                                       |
| --------------------------------- | ------------------------------------------------- |
| **Virtual Private Gateway (VGW)** | AWS-side VPN endpoint attached to your VPC        |
| **Customer Gateway (CGW)**        | Represents your on-prem VPN device                |
| **VPN Connection**                | Two IPsec tunnels (redundant) between VGW and CGW |
| **Transit Gateway VPN**           | VPN attached to TGW instead of a single VPC       |
| **Client VPN**                    | For individual user devices (like OpenVPN)        |

### Two Tunnels for HA

Every AWS VPN provides **2 IPsec tunnels by default** — terminating in different AWS AZs. You should configure your on-prem device to use both.

### Routing Options

| Option                    | Description                                 |
| ------------------------- | ------------------------------------------- |
| **Static Routing**        | Manually configure prefixes                 |
| **Dynamic Routing (BGP)** | AWS and your device exchange routes via BGP |

### Use Cases

| Scenario                     | Use VPN                                     |
| ---------------------------- | ------------------------------------------- |
| **Quick hybrid setup**       | Fast to provision (minutes vs weeks for DX) |
| **Small-to-medium traffic**  | Up to \~1.25 Gbps per tunnel                |
| **DR / backup for DX**       | Redundancy for private network              |
| **Branch office → AWS**      | Site-to-site VPN                            |
| **Remote workers → AWS**     | Client VPN                                  |
| **Multi-cloud connectivity** | VPN to Azure/GCP via BGP                    |

### VPN vs Direct Connect

| Aspect                 | **AWS VPN**                  | **Direct Connect**              |
| ---------------------- | ---------------------------- | ------------------------------- |
| **Underlying network** | Public internet              | Private fiber                   |
| **Setup time**         | Minutes                      | Weeks/months                    |
| **Bandwidth**          | Up to \~1.25 Gbps per tunnel | 1/10/100 Gbps                   |
| **Latency**            | Variable (internet)          | Consistent, low                 |
| **Encryption**         | Built-in IPsec               | You add (MACsec/app-level)      |
| **Cost**               | Low ($0.05/hr + data)        | Higher fixed cost               |
| **Use for**            | Quick, low-medium traffic    | Large-scale, prod hybrid        |
| **HA**                 | 2 tunnels built-in           | Need 2 connections + VPN backup |

### Client VPN (Remote User Access)

* Fully managed OpenVPN-based service.
* Users connect from laptops using AWS Client VPN Software.
* Authentication: AD, SAML, mutual certs.
* Perfect for **remote workforce** needing VPC access.

### Setup Site-to-Site VPN (CLI)

```bash
# Create Customer Gateway
aws ec2 create-customer-gateway \
  --bgp-asn 65000 --public-ip 203.0.113.42 --type ipsec.1

# Create Virtual Private Gateway
aws ec2 create-vpn-gateway --type ipsec.1

# Attach VGW to VPC
aws ec2 attach-vpn-gateway --vpc-id vpc-0abc --vpn-gateway-id vgw-0xyz

# Create VPN Connection
aws ec2 create-vpn-connection \
  --customer-gateway-id cgw-0abc \
  --vpn-gateway-id vgw-0xyz \
  --type ipsec.1 \
  --options StaticRoutesOnly=false
```

### Best Practices

* ✅ Use **both tunnels** — configure BGP for auto-failover.
* ✅ Enable **VPN CloudWatch logs** for tunnel state changes.
* ✅ Use **Transit Gateway VPN** for multi-VPC hub-and-spoke.
* ✅ Use **VPN as backup for Direct Connect**.
* ✅ Set **DPD (Dead Peer Detection)** for fast failover.
* ✅ Rotate **pre-shared keys** periodically.
* ✅ For remote workers → **Client VPN**, not Site-to-Site.

### Common Hybrid Architecture

```
On-Prem DC ─── DX (primary) ────────► AWS TGW ─► VPC A
            └── VPN (backup, BGP) ───►         └► VPC B
                                              └► VPC C
```

***

## 50. What is AWS CloudFront?

**Amazon CloudFront** is AWS's **global Content Delivery Network (CDN)** — caches content at 600+ **edge locations** worldwide to deliver low-latency access to users, protect origins, and reduce bandwidth costs.

### Key Benefits

* **Global low latency** — content served from nearest edge (\~10–50 ms).
* **Reduced origin load** — cached content = fewer origin hits.
* **DDoS protection** — via AWS Shield Standard (free) or Advanced.
* **WAF integration** — L7 protection.
* **HTTPS/TLS termination at edge** — SNI + custom certs (ACM).
* **HTTP/2, HTTP/3, gRPC** support.
* **Signed URLs/Cookies** — restrict access to premium content.
* **Lambda\@Edge / CloudFront Functions** — run code at the edge.
* **Origin Shield** — regional cache layer to further reduce origin load.

### Core Concepts

| Concept                         | Description                                                     |
| ------------------------------- | --------------------------------------------------------------- |
| **Distribution**                | The CDN configuration (a `d123.cloudfront.net` endpoint)        |
| **Origin**                      | Source of content (S3, ALB, EC2, MediaStore, custom HTTP)       |
| **Cache Behavior**              | Rules per path pattern (`/api/*` vs `/*`)                       |
| **Edge Location**               | Global PoP where content is cached (600+ globally, incl. India) |
| **Regional Edge Cache**         | Larger cache layer between edges & origin                       |
| **Origin Access Control (OAC)** | Restrict S3 to CloudFront-only access                           |
| **TTL**                         | How long objects stay cached                                    |
| **Invalidation**                | Force cache refresh (`/api/*` → purge)                          |

### Supported Origins

| Origin                                | Use                        |
| ------------------------------------- | -------------------------- |
| **S3 bucket**                         | Static websites, downloads |
| **S3 static website**                 | Custom index/error pages   |
| **ALB / NLB**                         | Dynamic web apps           |
| **EC2 / on-prem HTTP**                | Custom origins             |
| **Elemental MediaStore/MediaPackage** | Video streaming            |
| **API Gateway**                       | Cached REST APIs           |

### CloudFront Distribution Architecture

```
User (India) ──► Edge Location (Mumbai)   ← Cache hit (~10 ms)
                       │
                       ▼ Cache miss
              Regional Edge Cache
                       │
                       ▼
              Origin Shield (optional)
                       │
                       ▼
                Origin (S3/ALB in us-east-1)
```

### Common Use Cases

| Use Case                      | Example                                               |
| ----------------------------- | ----------------------------------------------------- |
| **Static website hosting**    | S3 + CloudFront (SPA React/Angular apps)              |
| **Dynamic API acceleration**  | CloudFront → ALB → EC2 (cache GET, pass-through POST) |
| **Video streaming**           | Live/VOD with MediaPackage + CloudFront               |
| **Software distribution**     | Downloads for global users                            |
| **Image/asset delivery**      | Product images, videos, thumbnails                    |
| **Security layer**            | AWS Shield + WAF at edge                              |
| **SEO/performance**           | Reduce TTFB globally                                  |
| **Geographic access control** | Geo-blocking by country                               |

### Cache Behaviors

Order behaviors by path pattern:

```
Path: /static/*    → Cache TTL: 1 year, compression enabled
Path: /api/*       → Cache TTL: 0 (pass-through), forward headers
Path: /images/*    → Cache TTL: 30 days
Default: /*        → Cache TTL: 1 hour
```

### Signed URLs & Signed Cookies

Restrict access to paid/premium content:

* **Signed URL**: for one specific object.
* **Signed Cookie**: for multiple objects (e.g., video streaming session).
* Uses **RSA private key** and short-lived expiration.

### Lambda\@Edge vs CloudFront Functions

| Feature            | Lambda\@Edge                                     | CloudFront Functions             |
| ------------------ | ------------------------------------------------ | -------------------------------- |
| **Runtime**        | Node.js/Python                                   | JavaScript only                  |
| **Max exec time**  | 5–30 seconds                                     | 1 ms                             |
| **Memory**         | 128 MB – 10 GB                                   | 2 MB                             |
| **Cost**           | Higher                                           | \~1/6 the cost                   |
| **Use case**       | Complex logic, SDK access                        | Simple header/URL rewrites, auth |
| **Trigger points** | Viewer request/response, Origin request/response | Viewer request/response only     |

### Real-World Setup — S3 Static Site + CloudFront + ACM

```bash
# Create S3 bucket
aws s3 mb s3://my-site-bucket
aws s3 sync ./build s3://my-site-bucket

# Create ACM cert (must be in us-east-1 for CloudFront)
aws acm request-certificate --domain-name www.example.com \
  --validation-method DNS --region us-east-1

# Create CloudFront distribution (config JSON)
aws cloudfront create-distribution --distribution-config file://dist.json

# Point Route 53 alias to CloudFront
aws route53 change-resource-record-sets --hosted-zone-id Z123 \
  --change-batch file://alias.json
```

### Cache Invalidation

```bash
aws cloudfront create-invalidation \
  --distribution-id EABC123 \
  --paths "/index.html" "/api/*"
```

* **First 1,000 paths/month** are free.
* Beyond that: $0.005/path.
* Prefer **versioned filenames** (`app.v42.js`) over invalidations.

### Security Features

* **Origin Access Control (OAC)** — force S3 access only through CloudFront (no public S3).
* **AWS WAF** — block SQLi, XSS, bots, rate limiting.
* **AWS Shield** — DDoS protection (Standard free, Advanced $3K/mo).
* **Geo-restriction** — allow/deny by country.
* **Field-level encryption** — encrypt specific form fields end-to-end.
* **Custom SSL certs** via ACM (SNI-only free; dedicated IP costs $600/mo).

### Pricing

* **Data transfer out** to viewers (per region tier).
* **HTTPS requests** (per 10,000).
* **Invalidations** (after 1,000/month).
* **Lambda\@Edge / CF Functions** invocations.
* **Origin Shield** (regional cache tier).

### Best Practices

* ✅ Use **OAC** to lock down S3 origins.
* ✅ Set **long TTLs** with **versioned URLs** (`app.v1.js`, `app.v2.js`).
* ✅ Enable **HTTP/2 & HTTP/3**.
* ✅ Enable **compression** (Gzip/Brotli).
* ✅ Attach **WAF** for L7 protection.
* ✅ Use **CloudFront Functions** for cheap edge logic.
* ✅ Monitor **cache hit ratio** — target > 90%.
* ✅ Use **Origin Shield** for very high traffic to reduce origin load.
* ✅ Use **Real-time logs** for debugging (Kinesis).

***

## Summary Cheat Sheet (Q41–Q50)

| #  | Concept               | One-Liner                                                                              |
| -- | --------------------- | -------------------------------------------------------------------------------------- |
| 41 | **EKS vs ECS**        | EKS = upstream K8s + portable; ECS = AWS-proprietary + simpler                         |
| 42 | **Auto Scaling**      | Adjusts EC2 count based on demand via ASG + policies                                   |
| 43 | **ELB**               | Managed load balancer distributing traffic across healthy targets                      |
| 44 | **ELB Types**         | CLB (legacy), ALB (L7), NLB (L4, static IP), GWLB (L3, appliances)                     |
| 45 | **Route 53**          | 100% SLA DNS + domain registration + health checks + advanced routing                  |
| 46 | **Hosted Zone**       | DNS record container — public (internet) or private (VPC)                              |
| 47 | **Lambda Cold Start** | Init latency for new execution env — mitigate with Provisioned Concurrency / SnapStart |
| 48 | **Direct Connect**    | Dedicated private fiber to AWS — consistent bandwidth & low latency                    |
| 49 | **AWS VPN**           | IPsec tunnels over internet — quick, cheap hybrid connectivity                         |
| 50 | **CloudFront**        | Global CDN with 600+ edges — caching, security, low latency                            |

***

## Bonus: Interview Scenarios

**Q:** *"You're getting frequent Lambda cold-start complaints on a customer-facing API. How do you fix?"*
**A:**

1. Enable **Provisioned Concurrency** for baseline traffic.
2. Use **Application Auto Scaling** on provisioned concurrency to match traffic patterns.
3. Switch to **SnapStart** if Java.
4. Move heavy imports **outside handler**.
5. Right-size **memory** (test 512/1024/2048).
6. Consider migrating latency-critical endpoints to **CloudFront Functions** or **API Gateway with cache**.

**Q:** *"Design a global multi-region web app with < 100 ms latency for users worldwide."*
**A:**

* **CloudFront** (edge caching) →
* **Route 53 latency-based routing** to regional endpoints (us-east-1, ap-south-1, eu-west-1) →
* **ALB + ASG** in each region →
* **Aurora Global Database** for cross-region reads →
* **DynamoDB Global Tables** for session state.

**Q:** *"On-prem app needs to reach 3 VPCs in 2 regions privately. Design?"*
**A:**

* **Direct Connect** to nearest DX location (dual for HA) →
* **Direct Connect Gateway** →
* **Transit Gateway (multi-region peered)** →
* All VPCs.
* **VPN as failover** with BGP for automatic route change.

**Q:** *"Your NLB in front of a legacy TCP app needs to allow only certain corporate IPs. How?"*
**A:** NLB doesn't support Security Groups on the LB itself (until recent update), so:

* Assign **Elastic IPs** to NLB.
* Apply **Security Group on target EC2** allowing NLB Elastic IPs' subnet.
* OR use **AWS Network Firewall** upstream for IP whitelisting.
* OR use **PrivateLink** if consumer is in AWS.

**Q:** *"Why choose ALB over CLB for a new microservices app?"*
**A:** ALB provides path/host-based routing, WebSocket/gRPC support, native container/Lambda targets, weighted target groups (canary), Cognito auth integration, HTTP/2, redirect actions, and lower cost per LCU. CLB has none of these.

***

 

