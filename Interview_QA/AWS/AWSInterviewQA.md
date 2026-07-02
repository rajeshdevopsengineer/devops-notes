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

