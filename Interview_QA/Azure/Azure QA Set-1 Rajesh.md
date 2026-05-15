# Azure Cloud Interview Questions and Answers (1–50)

For 6–10 Years Experienced Candidates

---

# 1. What is Microsoft Azure?

### Answer:

Microsoft Azure is a cloud computing platform provided by Microsoft that offers services such as:

* Compute
* Networking
* Storage
* Databases
* Security
* AI/ML
* Monitoring
* Containers

Azure supports:

* Infrastructure as a Service (IaaS)
* Platform as a Service (PaaS)
* Software as a Service (SaaS)

Organizations use Azure to build, deploy, and manage applications through Microsoft-managed datacenters.

### Example:

Hosting:

* Virtual Machines
* Kubernetes clusters
* Web applications
* Disaster recovery solutions

### Enterprise Best Practice:

Use Landing Zone architecture with governance, RBAC, tagging, and policy enforcement.

---

# 2. Explain the difference between IaaS, PaaS, and SaaS.

### Answer:

| Service Model | Responsibility of Customer | Example       |
| ------------- | -------------------------- | ------------- |
| IaaS          | OS, apps, networking       | Azure VM      |
| PaaS          | Only application/data      | Azure Web App |
| SaaS          | Only usage                 | Microsoft 365 |

### Example:

* IaaS → Full VM management
* PaaS → Deploy code only
* SaaS → Use application directly

### Interview Tip:

Architects prefer PaaS whenever possible to reduce operational overhead.

---

# 3. What are Azure Regions and Availability Zones?

### Answer:

### Azure Region:

A geographical area containing one or more datacenters.

Example:

* East US
* Central India
* West Europe

### Availability Zone:

Physically separate datacenters inside a region.

Benefits:

* High availability
* Fault tolerance
* Low latency

### Example:

Deploy VMs across:

* Zone 1
* Zone 2
* Zone 3

### Best Practice:

Use Zone-redundant architecture for production workloads.

---

# 4. What is an Azure Resource Group?

### Answer:

A Resource Group is a logical container for Azure resources.

### Example:

A production resource group may contain:

* VM
* VNet
* NSG
* Storage Account

### Benefits:

* Access management
* Cost tracking
* Lifecycle management

### Best Practice:

Separate:

* Dev
* Test
* Production

Into different resource groups.

---

# 5. What is Azure Resource Manager (ARM)?

### Answer:

ARM is the deployment and management service for Azure.

### Features:

* Infrastructure as Code
* RBAC integration
* Policy enforcement
* Tagging
* Template deployment

### Example:

Deploying:

* VM
* VNet
* Storage

Using JSON templates.

### Best Practice:

Use ARM/Bicep/Terraform for repeatable deployments.

---

# 6. What is the difference between ARM Templates and Terraform?

### Answer:

| Feature    | ARM Template | Terraform   |
| ---------- | ------------ | ----------- |
| Language   | JSON         | HCL         |
| Vendor     | Azure-native | Multi-cloud |
| Complexity | Verbose      | Easier      |
| State File | No           | Yes         |

### Enterprise Usage:

* ARM/Bicep → Azure-native automation
* Terraform → Multi-cloud environments

### Best Practice:

Use Terraform modules for reusable enterprise infrastructure.

---

# 7. Explain Azure Virtual Machine (VM).

### Answer:

Azure VM is an IaaS compute service that provides scalable virtual servers.

### Supported OS:

* Windows
* Linux

### Components:

* OS Disk
* Data Disk
* NIC
* NSG
* Public IP

### Example:

Hosting:

* SQL Server
* Jenkins
* SAP workloads

### Best Practice:

Use Managed Disks and Availability Zones.

---

# 8. What are Azure VM Sizes?

### Answer:

VM sizes define:

* CPU
* RAM
* Disk capacity
* Network throughput

### Categories:

| Series   | Purpose           |
| -------- | ----------------- |
| B-series | Burst workloads   |
| D-series | General purpose   |
| E-series | Memory optimized  |
| F-series | Compute optimized |

### Example:

E-series for databases.

### Best Practice:

Right-size VMs to optimize cost.

---

# 9. What is Azure Availability Set?

### Answer:

Availability Set protects applications from:

* Hardware failures
* Maintenance downtime

### Components:

* Fault Domains
* Update Domains

### Example:

2 VMs in an availability set ensure uptime during maintenance.

### Best Practice:

Use VM Scale Sets or Availability Zones instead for newer deployments.

---

# 10. What is Azure VM Scale Set (VMSS)?

### Answer:

VMSS allows automatic scaling of identical VMs.

### Features:

* Auto scaling
* Load balancing
* High availability

### Example:

Web servers scaling from 2 to 20 instances based on CPU.

### Best Practice:

Use autoscaling rules with Azure Monitor metrics.

---

# 11. What is a Virtual Network (VNet)?

### Answer:

VNet is a logically isolated network in Azure.

### Features:

* Subnets
* Routing
* NSG integration
* Private communication

### Example:

Separate subnets for:

* Web tier
* App tier
* Database tier

### Best Practice:

Use hub-spoke topology in enterprises.

---

# 12. What is subnetting in Azure?

### Answer:

Subnetting divides a VNet into smaller networks.

### Benefits:

* Isolation
* Security
* Better IP management

### Example:

| Subnet    | Purpose    |
| --------- | ---------- |
| WebSubnet | Frontend   |
| AppSubnet | Middleware |
| DBSubnet  | Database   |

### Best Practice:

Reserve IP ranges for future growth.

---

# 13. What is an NSG (Network Security Group)?

### Answer:

NSG filters inbound and outbound traffic.

### Rules:

* Source
* Destination
* Port
* Protocol
* Action

### Example:

Allow:

* Port 80
* Port 443

Block:

* RDP from internet

### Best Practice:

Use least privilege rules.

---

# 14. Difference between NSG and Azure Firewall?

### Answer:

| Feature             | NSG     | Azure Firewall |
| ------------------- | ------- | -------------- |
| Layer               | L3/L4   | L3–L7          |
| Stateful            | Yes     | Yes            |
| Threat Intelligence | No      | Yes            |
| Centralized         | Limited | Yes            |

### Enterprise Practice:

* NSG → Subnet/VM filtering
* Firewall → Central security control

---

# 15. What are User Defined Routes (UDR)?

### Answer:

UDR allows custom routing in Azure VNets.

### Example:

Route internet traffic through:

* Azure Firewall
* NVA

### Scenario:

Forced tunneling via firewall appliance.

### Best Practice:

Document routing tables carefully.

---

# 16. What is Azure Bastion?

### Answer:

Azure Bastion provides secure RDP/SSH access via browser without exposing public IPs.

### Benefits:

* No inbound public ports
* Secure management

### Example:

SSH to Linux VM privately.

### Best Practice:

Use Bastion for production environments.

---

# 17. What is Azure Load Balancer?

### Answer:

Azure Load Balancer distributes traffic across backend servers.

### Types:

* Public
* Internal

### Layers:

Layer 4 (TCP/UDP)

### Example:

Distribute HTTP traffic to multiple VMs.

### Best Practice:

Use Standard Load Balancer in production.

---

# 18. Difference between Load Balancer and Application Gateway?

### Answer:

| Feature     | Load Balancer | App Gateway |
| ----------- | ------------- | ----------- |
| Layer       | L4            | L7          |
| SSL Offload | No            | Yes         |
| WAF         | No            | Yes         |
| URL Routing | No            | Yes         |

### Best Practice:

Use:

* LB for TCP workloads
* App Gateway for web apps

---

# 19. What is Azure Traffic Manager?

### Answer:

Traffic Manager distributes traffic globally using DNS-based routing.

### Routing Methods:

* Priority
* Weighted
* Performance
* Geographic

### Example:

Users routed to nearest region.

### Best Practice:

Combine with regional load balancers.

---

# 20. Explain Azure Storage Account.

### Answer:

Storage Account provides scalable cloud storage.

### Types:

* Blob
* File
* Queue
* Table

### Redundancy:

* LRS
* GRS
* ZRS

### Best Practice:

Use lifecycle policies for cost optimization.

---

# 21. What is Blob Storage?

### Answer:

Blob Storage stores unstructured data.

### Use Cases:

* Images
* Videos
* Backups
* Logs

### Access Tiers:

* Hot
* Cool
* Archive

### Best Practice:

Use private endpoints for secure access.

---

# 22. What is Azure Files?

### Answer:

Azure Files provides SMB/NFS managed file shares.

### Example:

Shared storage across VMs.

### Enterprise Use:

Lift-and-shift legacy applications.

---

# 23. What is Managed Disk in Azure?

### Answer:

Managed Disk is Azure-managed block storage for VMs.

### Types:

* Standard HDD
* Standard SSD
* Premium SSD
* Ultra Disk

### Best Practice:

Use Premium SSD for databases.

---

# 24. What is Azure Backup?

### Answer:

Azure Backup provides centralized backup management.

### Supported:

* VMs
* SQL
* File Shares

### Features:

* Recovery points
* Encryption
* Retention policies

### Best Practice:

Enable soft delete.

---

# 25. What is Azure Site Recovery (ASR)?

### Answer:

ASR provides disaster recovery and replication.

### Use Cases:

* VM failover
* DR orchestration

### Example:

Primary region → Secondary region replication.

### Best Practice:

Perform DR drills regularly.

---

# 26. What is Azure Key Vault?

### Answer:

Key Vault securely stores:

* Secrets
* Certificates
* Keys

### Example:

Store database passwords securely.

### Best Practice:

Use Managed Identity instead of hardcoded credentials.

---

# 27. What are Managed Identities?

### Answer:

Managed Identity provides Azure AD identity for resources.

### Benefits:

* No credential management
* Secure authentication

### Example:

Web App accessing Key Vault.

### Best Practice:

Use system-assigned identity wherever possible.

---

# 28. What is Azure App Service?

### Answer:

App Service is a PaaS platform for hosting web apps and APIs.

### Supports:

* .NET
* Java
* Python
* Node.js

### Benefits:

* Auto scaling
* CI/CD
* SSL support

### Best Practice:

Use deployment slots.

---

# 29. What are Deployment Slots in Web Apps?

### Answer:

Deployment slots allow zero-downtime deployments.

### Example:

* Staging slot
* Production slot

Swap after testing.

### Best Practice:

Always validate in staging before swap.

---

# 30. What is Azure Function App?

### Answer:

Function App is serverless compute service.

### Trigger Types:

* HTTP
* Timer
* Queue
* Event Grid

### Example:

Auto resize uploaded images.

### Best Practice:

Use consumption plan for unpredictable workloads.

---

# 31. What is Azure Container Registry (ACR)?

### Answer:

ACR stores private Docker container images.

### Features:

* Geo-replication
* Vulnerability scanning
* Webhooks

### Example:

AKS pulling images from ACR.

### Best Practice:

Use private endpoints and RBAC.

---

# 32. What is Azure Container Apps?

### Answer:

Container Apps run microservices without managing Kubernetes.

### Features:

* Auto scaling
* KEDA integration
* Dapr support

### Example:

Event-driven applications.

### Best Practice:

Use for lightweight microservices.

---

# 33. What is Azure Kubernetes Service (AKS)?

### Answer:

AKS is a managed Kubernetes service.

### Benefits:

* Managed control plane
* Scaling
* Integration with Azure services

### Example:

Hosting microservices architecture.

### Best Practice:

Use separate node pools.

---

# 34. What are AKS Node Pools?

### Answer:

Node pools are groups of nodes within AKS.

### Types:

* System node pool
* User node pool

### Example:

Separate GPU workloads.

### Best Practice:

Use taints and tolerations.

---

# 35. What is Azure Application Gateway?

### Answer:

Application Gateway is a Layer 7 load balancer.

### Features:

* WAF
* SSL termination
* URL routing

### Example:

Route traffic based on URL paths.

### Best Practice:

Enable WAF in prevention mode.

---

# 36. What is Web Application Firewall (WAF)?

### Answer:

WAF protects web applications from attacks.

### Protects Against:

* SQL Injection
* XSS
* OWASP Top 10

### Best Practice:

Enable logging and monitoring.

---

# 37. What is Azure DNS?

### Answer:

Azure DNS hosts DNS domains in Azure.

### Benefits:

* High availability
* Fast resolution

### Example:

Hosting company domain DNS records.

---

# 38. What is Azure Monitor?

### Answer:

Azure Monitor collects:

* Metrics
* Logs
* Alerts

### Components:

* Log Analytics
* Application Insights

### Best Practice:

Configure proactive alerts.

---

# 39. What is Log Analytics Workspace?

### Answer:

Central repository for logs.

### Query Language:

KQL (Kusto Query Language)

### Example:

Search failed login attempts.

### Best Practice:

Use retention policies.

---

# 40. What is Application Insights?

### Answer:

Application Insights monitors application performance.

### Features:

* Dependency tracking
* Live metrics
* Distributed tracing

### Best Practice:

Enable end-to-end monitoring.

---

# 41. What is Azure CLI?

### Answer:

Azure CLI is a command-line tool for Azure management.

### Example:

```bash
az vm list
```

### Benefits:

* Automation
* Scripting
* CI/CD integration

---

# 42. What is the difference between Azure CLI and PowerShell?

### Answer:

| Azure CLI      | PowerShell             |
| -------------- | ---------------------- |
| Cross-platform | Object-oriented        |
| Bash-friendly  | Windows admin friendly |

### Enterprise Usage:

* CLI → DevOps pipelines
* PowerShell → Admin automation

---

# 43. What is Infrastructure as Code (IaC)?

### Answer:

IaC automates infrastructure deployment through code.

### Tools:

* ARM
* Bicep
* Terraform

### Benefits:

* Consistency
* Repeatability
* Version control

---

# 44. Scenario: Users cannot RDP to VM. How do you troubleshoot?

### Answer:

Check:

1. VM running?
2. NSG allows 3389?
3. Public IP assigned?
4. Windows Firewall?
5. Route table issue?
6. Bastion access?

### Best Practice:

Use Bastion instead of public RDP.

---

# 45. Scenario: Website is slow in Azure. What will you check?

### Answer:

Investigate:

* CPU/memory metrics
* App Insights
* Backend latency
* Database performance
* Scaling issues

### Tools:

* Azure Monitor
* Log Analytics

---

# 46. Scenario: VM CPU utilization is constantly high.

### Answer:

Possible reasons:

* Under-sized VM
* Memory pressure
* Application bottleneck

### Solutions:

* Resize VM
* Scale out
* Optimize application

---

# 47. How do you secure Azure workloads?

### Answer:

Use:

* NSG
* Azure Firewall
* Key Vault
* Defender for Cloud
* RBAC
* MFA
* Private Endpoints

### Best Practice:

Follow Zero Trust architecture.

---

# 48. What is RBAC in Azure?

### Answer:

Role-Based Access Control controls permissions.

### Roles:

* Reader
* Contributor
* Owner

### Best Practice:

Apply least privilege principle.

---

# 49. What are Azure Tags?

### Answer:

Tags help organize resources.

### Example:

* Environment=Prod
* Owner=DevOps

### Benefits:

* Cost tracking
* Governance

### Best Practice:

Enforce tags using Azure Policy.

---

# 50. What are Azure Policies?

### Answer:

Azure Policy enforces governance rules.

### Example:

* Allow only approved VM sizes
* Restrict regions

### Best Practice:

Use initiatives for enterprise governance.

---

# Azure Cloud Interview Questions and Answers (51–100)

For 6–10 Years Experienced Candidates

---

# 51. What is Azure Availability Zone-aware architecture?

### Answer:

Availability Zone-aware architecture distributes workloads across multiple Availability Zones within a region to improve resiliency.

### Benefits:

* Protection from datacenter failure
* High availability
* Better SLA

### Example:

Deploy:

* VM1 → Zone 1
* VM2 → Zone 2
* Database → Zone redundant

### Best Practice:

Use zone-redundant services for mission-critical applications.

---

# 52. What is the difference between Availability Set and Availability Zone?

### Answer:

| Feature    | Availability Set  | Availability Zone     |
| ---------- | ----------------- | --------------------- |
| Scope      | Same datacenter   | Different datacenters |
| Protection | Hardware failures | Datacenter failures   |
| Latency    | Very low          | Slightly higher       |
| SLA        | Lower             | Higher                |

### Enterprise Recommendation:

Use Availability Zones for production applications.

---

# 53. What is Azure Dedicated Host?

### Answer:

Dedicated Host provides physical servers dedicated to one organization.

### Benefits:

* Compliance
* Licensing benefits
* Isolation

### Example:

Running licensed Windows Server workloads.

### Best Practice:

Use for regulatory or compliance-sensitive applications.

---

# 54. What is Azure Proximity Placement Group (PPG)?

### Answer:

PPG reduces network latency between Azure resources.

### Example:

Place:

* App VM
* Database VM

Close together physically.

### Use Cases:

* Trading applications
* SAP
* HPC workloads

---

# 55. What is Accelerated Networking in Azure?

### Answer:

Accelerated Networking improves VM network performance using SR-IOV.

### Benefits:

* Lower latency
* Reduced jitter
* Higher throughput

### Best Practice:

Enable for production workloads requiring high network performance.

---

# 56. What is Azure Virtual WAN?

### Answer:

Virtual WAN provides centralized networking management.

### Features:

* Site-to-site VPN
* ExpressRoute
* Remote users
* Branch connectivity

### Best Practice:

Use for global enterprise networking.

---

# 57. What is VNet Peering?

### Answer:

VNet Peering connects Azure VNets privately.

### Benefits:

* Low latency
* High bandwidth
* No internet traversal

### Types:

* Regional
* Global

### Best Practice:

Use hub-and-spoke architecture.

---

# 58. Difference between VPN Gateway and ExpressRoute?

### Answer:

| Feature      | VPN Gateway        | ExpressRoute       |
| ------------ | ------------------ | ------------------ |
| Connectivity | Internet           | Private connection |
| Speed        | Lower              | Higher             |
| Security     | Encrypted internet | Dedicated circuit  |
| Cost         | Lower              | Higher             |

### Enterprise Use:

Use ExpressRoute for critical enterprise workloads.

---

# 59. What is Azure ExpressRoute?

### Answer:

ExpressRoute provides private connectivity between on-premises and Azure.

### Benefits:

* Low latency
* High reliability
* Secure connectivity

### Example:

Hybrid cloud integration for enterprise datacenters.

### Best Practice:

Use redundant circuits.

---

# 60. What is Azure VPN Gateway?

### Answer:

VPN Gateway enables encrypted communication between:

* Azure VNets
* On-premises networks

### VPN Types:

* Site-to-site
* Point-to-site
* VNet-to-VNet

### Best Practice:

Use active-active gateways for redundancy.

---

# 61. What is Azure Private Endpoint?

### Answer:

Private Endpoint provides private IP connectivity to Azure services.

### Benefits:

* No public internet exposure
* Enhanced security

### Example:

Private access to Storage Account.

### Best Practice:

Disable public access after enabling Private Endpoint.

---

# 62. What is Azure Service Endpoint?

### Answer:

Service Endpoint extends VNet identity to Azure services.

### Difference:

* Service Endpoint → Public endpoint secured by VNet
* Private Endpoint → Private IP access

### Best Practice:

Prefer Private Endpoints for sensitive workloads.

---

# 63. What is Azure Firewall?

### Answer:

Azure Firewall is a managed cloud firewall service.

### Features:

* Stateful filtering
* Threat intelligence
* DNAT/SNAT
* Application rules

### Best Practice:

Deploy in hub network architecture.

---

# 64. What is Azure DDoS Protection?

### Answer:

DDoS Protection safeguards applications from distributed denial-of-service attacks.

### Plans:

* Basic
* Standard

### Features:

* Attack analytics
* Adaptive tuning

### Best Practice:

Enable Standard tier for internet-facing applications.

---

# 65. What is Azure Front Door?

### Answer:

Azure Front Door is a global application delivery service.

### Features:

* Global load balancing
* SSL offload
* WAF
* CDN integration

### Best Practice:

Use for globally distributed applications.

---

# 66. Difference between Traffic Manager and Front Door?

### Answer:

| Feature     | Traffic Manager | Front Door |
| ----------- | --------------- | ---------- |
| Layer       | DNS-based       | Layer 7    |
| Failover    | DNS             | Real-time  |
| WAF         | No              | Yes        |
| SSL Offload | No              | Yes        |

### Best Practice:

Use Front Door for web applications.

---

# 67. What is Azure CDN?

### Answer:

Azure CDN caches content closer to users globally.

### Benefits:

* Faster performance
* Reduced latency
* Lower backend load

### Example:

Delivering videos/images globally.

---

# 68. What is Azure Disk Encryption?

### Answer:

Encrypts VM disks using:

* BitLocker (Windows)
* DM-Crypt (Linux)

### Integration:

Uses Microsoft Azure Key Vault.

### Best Practice:

Enable encryption for all production workloads.

---

# 69. What is Azure Defender for Cloud?

### Answer:

Defender for Cloud provides:

* Security posture management
* Threat detection
* Recommendations

### Features:

* Secure score
* Vulnerability assessment

### Best Practice:

Continuously remediate recommendations.

---

# 70. What is Azure Security Center Secure Score?

### Answer:

Secure Score measures security posture.

### Example Recommendations:

* Enable MFA
* Encrypt disks
* Restrict ports

### Best Practice:

Maintain high secure score across subscriptions.

---

# 71. What is Azure Policy Initiative?

### Answer:

Policy Initiative groups multiple policies together.

### Example:

Enterprise governance initiative including:

* Tag enforcement
* Region restrictions
* VM SKU limitations

### Best Practice:

Use initiatives for organization-wide governance.

---

# 72. What is Management Group in Azure?

### Answer:

Management Groups organize subscriptions hierarchically.

### Example:

* Production
* Development
* Shared Services

### Benefits:

* Centralized governance
* RBAC inheritance

---

# 73. What is Azure Blueprints?

### Answer:

Azure Blueprints automate environment deployment with governance.

### Includes:

* Policies
* ARM templates
* RBAC assignments

### Best Practice:

Use for regulated enterprise environments.

---

# 74. What is Azure Landing Zone?

### Answer:

Landing Zone is a preconfigured Azure environment following best practices.

### Components:

* Identity
* Networking
* Governance
* Security

### Best Practice:

Adopt enterprise-scale landing zone architecture.

---

# 75. What is Azure Cost Management?

### Answer:

Azure Cost Management monitors and optimizes cloud spending.

### Features:

* Budgets
* Forecasting
* Recommendations

### Best Practice:

Implement tagging for accurate cost allocation.

---

# 76. How do you optimize Azure costs?

### Answer:

Methods:

* Reserved Instances
* Auto scaling
* Spot VMs
* Right sizing
* Shutdown unused resources

### Best Practice:

Review advisor recommendations monthly.

---

# 77. What are Azure Reserved Instances?

### Answer:

Reserved Instances provide discounted pricing for long-term usage.

### Terms:

* 1 year
* 3 years

### Savings:

Up to 72% compared to pay-as-you-go.

---

# 78. What are Spot VMs?

### Answer:

Spot VMs use unused Azure capacity at discounted prices.

### Limitation:

Can be evicted anytime.

### Use Cases:

* Batch processing
* Dev/Test

---

# 79. What is Azure Advisor?

### Answer:

Advisor provides recommendations for:

* Cost
* Security
* Performance
* Reliability

### Best Practice:

Review recommendations regularly.

---

# 80. What is Azure Monitor Alert?

### Answer:

Alerts notify administrators when conditions occur.

### Types:

* Metric alerts
* Log alerts
* Activity log alerts

### Example:

CPU > 80%

---

# 81. What is KQL (Kusto Query Language)?

### Answer:

KQL is used for querying logs in Log Analytics.

### Example:

```kusto id="0d7id9"
Heartbeat
| summarize count() by Computer
```

### Best Practice:

Use workbooks for visualization.

---

# 82. What is Azure Activity Log?

### Answer:

Activity Log records management operations.

### Tracks:

* Resource creation
* Deletion
* Updates

### Best Practice:

Enable log retention and export.

---

# 83. What is Diagnostic Logging?

### Answer:

Diagnostic logs capture resource-level operational data.

### Example:

* NSG flow logs
* App Gateway logs

### Best Practice:

Send logs to Log Analytics.

---

# 84. What is NSG Flow Log?

### Answer:

NSG Flow Logs capture network traffic information.

### Benefits:

* Troubleshooting
* Security analysis

### Example:

Identify blocked traffic.

---

# 85. Scenario: VM cannot access internet. How do you troubleshoot?

### Answer:

Check:

1. NSG outbound rules
2. Route tables
3. Public IP/NAT
4. Firewall restrictions
5. DNS settings

### Best Practice:

Use Network Watcher tools.

---

# 86. What is Azure Network Watcher?

### Answer:

Network Watcher monitors and diagnoses networking issues.

### Features:

* IP flow verify
* Connection troubleshoot
* Packet capture

### Best Practice:

Enable in all regions.

---

# 87. What is Connection Monitor?

### Answer:

Connection Monitor checks connectivity between endpoints.

### Example:

VM to database connectivity monitoring.

---

# 88. What is Azure DNS Private Zone?

### Answer:

Private DNS Zones provide internal DNS resolution.

### Example:

Resolve private endpoint names internally.

### Best Practice:

Link VNets appropriately.

---

# 89. What is Azure File Sync?

### Answer:

Azure File Sync synchronizes on-premises file servers with Azure Files.

### Benefits:

* Cloud tiering
* Centralized file management

### Enterprise Use:

Hybrid file server architecture.

---

# 90. What is Azure Queue Storage?

### Answer:

Queue Storage stores messages for asynchronous processing.

### Example:

Microservices communication.

### Best Practice:

Use retry policies.

---

# 91. What is Azure Table Storage?

### Answer:

NoSQL key-value storage service.

### Use Cases:

* Metadata
* Lightweight applications

---

# 92. What is Lifecycle Management in Storage?

### Answer:

Lifecycle policies automatically move/delete blobs.

### Example:

Move old data:

* Hot → Cool → Archive

### Best Practice:

Reduce storage costs.

---

# 93. What is Soft Delete in Azure Storage?

### Answer:

Soft delete protects against accidental deletion.

### Best Practice:

Enable for all production storage accounts.

---

# 94. What is Immutable Blob Storage?

### Answer:

Prevents modification/deletion for compliance requirements.

### Use Cases:

* Financial records
* Legal compliance

---

# 95. What is Shared Access Signature (SAS)?

### Answer:

SAS provides delegated limited access to storage resources.

### Example:

Temporary upload URL.

### Best Practice:

Use short expiration times.

---

# 96. Difference between Managed Identity and Service Principal?

### Answer:

| Managed Identity | Service Principal    |
| ---------------- | -------------------- |
| Azure-managed    | User-managed         |
| No secrets       | Requires credentials |
| Easier rotation  | Manual rotation      |

### Best Practice:

Prefer Managed Identity.

---

# 97. What is Azure RBAC inheritance?

### Answer:

Permissions inherit from:

* Management Group
* Subscription
* Resource Group
* Resource

### Best Practice:

Assign permissions at higher scopes carefully.

---

# 98. What is Just-In-Time VM Access?

### Answer:

JIT reduces attack surface by opening ports temporarily.

### Example:

Open RDP for 1 hour only.

### Best Practice:

Enable through Defender for Cloud.

---

# 99. Scenario: Storage Account access is denied. How do you troubleshoot?

### Answer:

Check:

* RBAC permissions
* Firewall rules
* SAS expiry
* Private endpoint DNS
* Access keys

### Best Practice:

Use Azure AD authentication instead of keys.

---

# 100. Scenario: Application Gateway returns 502 Bad Gateway.

### Answer:

Possible causes:

* Backend unhealthy
* NSG blocking traffic
* Wrong health probe
* SSL mismatch

### Troubleshooting:

* Check backend health
* Verify probe settings
* Review logs

### Best Practice:

Configure custom health probes correctly.

---

