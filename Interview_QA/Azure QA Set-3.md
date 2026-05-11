
***

## 1. What is Microsoft Azure and what are the main service categories (IaaS, PaaS, SaaS)?

**Microsoft Azure** is a **cloud computing platform** that provides on‑demand access to compute, storage, networking, databases, AI, security, and DevOps services over Microsoft’s global datacenters.

### Main Service Models

| Model                                  | What Azure Manages   | What You Manage | Examples                |
| -------------------------------------- | -------------------- | --------------- | ----------------------- |
| **IaaS** (Infrastructure as a Service) | Hardware, datacenter | OS, VM, apps    | Azure VMs, VNets        |
| **PaaS** (Platform as a Service)       | OS, runtime, scaling | App code, data  | App Service, Azure SQL  |
| **SaaS** (Software as a Service)       | Entire stack         | Usage only      | Microsoft 365, Dynamics |

➡️ **Key idea:** As you move from IaaS → SaaS, **management responsibility decreases**.

***

## 2. What is an Azure subscription and how does it relate to management groups and tenants?

*   **Azure Tenant**  
    Represents an instance of **Microsoft Entra ID (Azure AD)**.  
    It is the **identity boundary**.

*   **Management Group**  
    Used to **organize multiple subscriptions** and apply policies/RBAC at scale.

*   **Subscription**  
    A **billing + access boundary** where resources are created.

### Hierarchy

    Tenant
     └── Management Group(s)
          └── Subscription(s)
               └── Resource Group(s)
                    └── Resources

➡️ **Key idea:**

*   Tenant = Identity
*   Subscription = Billing & access
*   Management Groups = Governance

***

## 3. What is an Azure region and an availability zone?

### Azure Region

*   A **geographic area** with one or more datacenters
*   Examples: **Central India, East US, West Europe**

### Availability Zone (AZ)

*   **Physically separate datacenters** within a region
*   Independent power, cooling, and networking

➡️ **Purpose:** High availability & fault tolerance  
➡️ Not all regions support Availability Zones

***

## 4. What is an Azure resource group and why use one?

A **Resource Group** is a **logical container** for Azure resources.

### Why use resource groups?

*   **Lifecycle management** (delete all together)
*   **Role-Based Access Control (RBAC)**
*   **Cost tracking**
*   **Deployment scope**

➡️ Resources can exist in different regions but **belong to one resource group**

***

## 5. What are resource tags and how are they useful?

**Tags** are **key–value pairs** attached to resources or resource groups.

### Common Uses

*   Cost management (`Project=CRM`)
*   Ownership (`Owner=DevOps`)
*   Environment (`Env=Prod`)
*   Automation

➡️ Tags are **not inherited by default** (except via policies)

***

## 6. What is an Azure Virtual Network (VNet)?

A **VNet** is a **private, isolated network** in Azure.

### Features

*   IP addressing (CIDR)
*   Subnets
*   Routing
*   Network security
*   VPN / ExpressRoute connectivity

➡️ Equivalent to a **traditional on‑prem network** in the cloud.

***

## 7. What is a subnet within a VNet?

A **Subnet** is a **subdivision of a VNet’s address space**.

### Purpose

*   Segment workloads
*   Apply security rules (NSGs)
*   Isolate tiers (web / app / DB)

➡️ Each subnet operates at **Layer 3 (IP routing)**.

***

## 8. What is an Azure Network Security Group (NSG) and when do you use it?

An **NSG** is a **stateful firewall** that controls inbound and outbound traffic.

### Applied To

*   Subnets
*   Network interfaces (NICs)

### Rules

*   Allow/Deny
*   Based on IP, port, protocol
*   Priority-based (lower number = higher priority)

➡️ Use NSGs for **micro‑segmentation and traffic control**.

***

## 9. What is an Azure Route Table (UDR) and why use it?

**UDR (User Defined Route)** allows **custom routing** of network traffic.

### Why use UDRs?

*   Force traffic through firewall/NVA
*   Control internet vs on‑prem routing
*   Hub‑and‑spoke architectures

➡️ Overrides Azure’s **system routes**.

***

## 10. What is an Azure Virtual Machine (VM)?

An **Azure VM** is a **compute service** that provides a virtualized server.

### Key Characteristics

*   Full OS control
*   Windows or Linux
*   Scale up/down or out
*   Pay‑as‑you‑go

### Common Use Cases

*   Lift‑and‑shift workloads
*   Custom software
*   Legacy applications

➡️ Azure VMs are part of **IaaS**, offering maximum flexibility with maximum responsibility.

***

Based on your uploaded screenshot (Screenshot 2026-05-11 232217.png), here are comprehensive answers to questions 11-30:

---

## 11. **What VM Sizes Are and How to Choose Them**

**What they are:**
- Predefined configurations of CPU, memory, storage, and network capacity
- Organized into series/families optimized for different workloads

**VM Series:**
- **B-series**: Burstable, cost-effective for low CPU usage (dev/test)
- **D-series**: General purpose, balanced CPU-to-memory
- **E-series**: Memory-optimized (databases, caching)
- **F-series**: Compute-optimized (batch processing, web servers)
- **N-series**: GPU-enabled (AI/ML, rendering)
- **M-series**: Large memory workloads (SAP HANA)

**How to choose:**
1. **Workload requirements**: CPU vs. memory vs. storage intensive
2. **Performance needs**: IOPS, network throughput
3. **Cost constraints**: Balance performance with budget
4. **Scalability**: Can you resize later?
5. **Regional availability**: Not all sizes available in all regions

---

## 12. **What is Azure VM Scale Set (VMSS)?**

**What it is:**
- A group of identical, load-balanced VMs that automatically scale
- Managed as a single resource

**Key features:**
- **Auto-scaling**: Scale out/in based on metrics (CPU, memory, custom)
- **High availability**: Distributes VMs across fault domains and availability zones
- **Load balancing**: Integrated with Azure Load Balancer or Application Gateway
- **Large scale**: Support for thousands of VMs
- **Uniform configuration**: All VMs created from same image

**Use cases:**
- Web applications with variable traffic
- Batch processing workloads
- Microservices architectures
- Any application requiring horizontal scaling

---

## 13. **Azure Load Balancer vs. Application Gateway**

**Azure Load Balancer:**
- **Layer 4** (Transport layer - TCP/UDP)
- Distributes network traffic based on IP and port
- High performance, low latency
- Supports both inbound and outbound scenarios
- **Use for**: Non-HTTP workloads, TCP/UDP applications, simple load distribution

**Application Gateway:**
- **Layer 7** (Application layer - HTTP/HTTPS)
- Content-based routing (URL path, host headers)
- SSL termination, session affinity, URL rewriting
- Integrated Web Application Firewall (WAF)
- **Use for**: Web applications, microservices, API routing, SSL offloading

**Key difference:** Load Balancer = network-level; Application Gateway = application-level with advanced HTTP features

---

## 14. **Azure Traffic Manager and Use Scenarios**

**What it is:**
- DNS-based global traffic load balancer
- Routes users to optimal endpoint based on routing method
- Works across Azure regions and external endpoints

**Routing methods:**
- **Priority**: Failover to backup regions
- **Weighted**: Distribute traffic by percentage (A/B testing)
- **Performance**: Route to closest/fastest endpoint
- **Geographic**: Route based on user location (compliance)
- **Multivalue**: Return multiple healthy endpoints
- **Subnet**: Route based on user IP ranges

**Use scenarios:**
- **Disaster recovery**: Automatic failover between regions
- **Global applications**: Low-latency routing to nearest datacenter
- **Migration**: Gradual traffic shift to new infrastructure
- **Multi-region deployment**: High availability across geographies

---

## 15. **Azure Private Endpoint and Problem It Solves**

**What it is:**
- A network interface that connects privately to Azure PaaS services
- Uses a private IP from your VNet
- Traffic stays on Microsoft backbone network

**Problem it solves:**
- **Public exposure**: Eliminates public internet access to PaaS services
- **Data exfiltration**: Prevents unauthorized data transfer
- **Compliance**: Meets requirements for private connectivity
- **Network security**: Integrates PaaS into your private network topology

**Supported services:**
- Storage Accounts, SQL Database, Key Vault, Cosmos DB, Web Apps, Container Registry, etc.

**Benefits:**
- Access services via private IP (e.g., 10.0.1.5 instead of storageaccount.blob.core.windows.net)
- Works with on-premises via VPN/ExpressRoute
- No service endpoints needed

---

## 16. **Azure Storage Account and Storage Types**

**What it is:**
- A container for Azure Storage data services
- Provides unique namespace (e.g., mystorageaccount.blob.core.windows.net)

**Storage types provided:**
1. **Blob Storage**: Unstructured object storage (images, videos, backups)
   - Hot, Cool, Archive tiers
2. **File Storage**: Managed SMB/NFS file shares (lift-and-shift)
3. **Queue Storage**: Message queue for asynchronous processing
4. **Table Storage**: NoSQL key-value store (structured data)
5. **Disk Storage**: Managed disks for VMs (not directly in storage account)

**Performance tiers:**
- **Standard**: HDD-backed, cost-effective
- **Premium**: SSD-backed, low latency

**Redundancy options:**
- LRS, ZRS, GRS, GZRS (local, zone, geo-redundant)

---

## 17. **What is Azure Container Registry (ACR)?**

**What it is:**
- Private Docker container registry service
- Stores and manages container images and artifacts

**Key features:**
- **Private repositories**: Secure image storage
- **Geo-replication**: Replicate images across regions
- **Image scanning**: Vulnerability assessment
- **Webhooks**: Trigger actions on push/delete
- **Azure integration**: Works with AKS, App Service, Container Instances
- **Authentication**: Azure AD integration, service principals

**Use cases:**
- Store custom container images for AKS deployments
- CI/CD pipelines (build, store, deploy)
- Multi-region deployments with geo-replication
- Secure image management with RBAC

**Tiers:** Basic, Standard, Premium (with geo-replication)

---

## 18. **What is Azure Kubernetes Service (AKS)?**

**What it is:**
- Managed Kubernetes container orchestration service
- Azure handles control plane (master nodes)
- You manage worker nodes (or use virtual nodes)

**Key features:**
- **Managed control plane**: Free, Azure-managed Kubernetes API
- **Auto-scaling**: Cluster autoscaler and horizontal pod autoscaler
- **Integrated monitoring**: Azure Monitor, Container Insights
- **Security**: Azure AD integration, RBAC, network policies
- **DevOps integration**: CI/CD with Azure DevOps, GitHub Actions
- **Networking**: Azure CNI, Kubenet, private clusters

**Use cases:**
- Microservices architectures
- Container orchestration at scale
- Cloud-native applications
- DevOps and CI/CD workflows

**Benefits:** Reduced operational overhead, enterprise-grade security, Azure ecosystem integration

---

## 19. **Azure Web App (App Service) and Typical Use Cases**

**What it is:**
- Fully managed Platform as a Service (PaaS) for hosting web applications
- Supports multiple languages (.NET, Java, Node.js, Python, PHP)

**Key features:**
- **Auto-scaling**: Scale up/out based on demand
- **Deployment slots**: Blue-green deployments, staging environments
- **Built-in CI/CD**: GitHub, Azure DevOps, Bitbucket integration
- **Custom domains & SSL**: Free SSL certificates
- **Authentication**: Built-in Azure AD, social providers
- **Monitoring**: Application Insights integration

**Typical use cases:**
- Web applications and APIs
- RESTful services
- Mobile app backends
- Content management systems
- E-commerce sites
- Internal business applications

**App Service Plans:** Define compute resources (Free, Shared, Basic, Standard, Premium, Isolated)

---

## 20. **Azure Function App (Serverless) and Triggers**

**What it is:**
- Serverless compute service for event-driven code execution
- Pay only for execution time (consumption plan)
- No infrastructure management

**Triggers (what starts a function):**
- **HTTP Trigger**: REST API calls, webhooks
- **Timer Trigger**: Scheduled execution (cron jobs)
- **Blob Trigger**: File upload to storage
- **Queue Trigger**: Message in Azure Queue
- **Event Grid Trigger**: Azure events
- **Cosmos DB Trigger**: Database changes
- **Service Bus Trigger**: Enterprise messaging
- **Event Hub Trigger**: Big data streaming

**Use cases:**
- API backends
- Scheduled tasks (cleanup, reports)
- File processing pipelines
- IoT data processing
- Webhooks and integrations
- Microservices

**Hosting plans:** Consumption (serverless), Premium, Dedicated (App Service)

---

## 21. **Azure Container Apps vs. AKS**

**Azure Container Apps:**
- **Simplified container platform** built on Kubernetes
- Serverless, fully managed
- No cluster management
- Built-in features: auto-scaling, ingress, revisions
- **Use for**: Microservices, APIs, event-driven apps, simple container workloads

**AKS (Azure Kubernetes Service):**
- **Full Kubernetes control**
- Manage node pools, networking, storage
- Advanced configurations and customizations
- **Use for**: Complex orchestration, existing Kubernetes workloads, full control needed

**Key differences:**
| Feature | Container Apps | AKS |
|---------|---------------|-----|
| Complexity | Low | High |
| Control | Limited | Full |
| Management | Fully managed | Managed control plane |
| Kubernetes access | Abstracted | Direct |
| Best for | Simple containers | Complex orchestration |

---

## 22. **Azure Application Gateway and WAF Capability**

**What it is:**
- Layer 7 (HTTP/HTTPS) load balancer and application delivery controller
- Regional service with advanced routing capabilities

**Key features:**
- **URL-based routing**: Route by path (/api → backend1, /images → backend2)
- **SSL termination**: Offload SSL processing
- **Session affinity**: Sticky sessions
- **Autoscaling**: Scale based on traffic
- **Multi-site hosting**: Multiple websites on one gateway
- **Redirection**: HTTP to HTTPS, URL rewrites

**WAF (Web Application Firewall) capability:**
- **OWASP protection**: Top 10 vulnerabilities (SQL injection, XSS)
- **Custom rules**: Define your own security rules
- **Bot protection**: Identify and block malicious bots
- **Geo-filtering**: Block traffic from specific countries
- **Rate limiting**: Prevent DDoS attacks
- **Modes**: Detection (log only) or Prevention (block)

**Use cases:** Secure web applications, microservices routing, SSL offloading

---

## 23. **Key Vault and Types of Secrets It Can Store**

**What it is:**
- Centralized cloud service for securely storing and managing sensitive information
- Hardware Security Module (HSM) backed

**Types of secrets stored:**

1. **Secrets**: Passwords, connection strings, API keys, tokens
2. **Keys**: Cryptographic keys for encryption/decryption
   - RSA, EC keys
   - HSM-protected or software-protected
3. **Certificates**: SSL/TLS certificates
   - Import existing or generate new
   - Automatic renewal support

**Key features:**
- **Access control**: Azure AD authentication, RBAC
- **Auditing**: Track all access and operations
- **Versioning**: Maintain secret history
- **Soft delete**: Recovery protection
- **Integration**: Works with App Service, VMs, AKS, Functions
- **Managed identities**: Passwordless authentication

**Use cases:** Secure application secrets, certificate management, encryption key management

---

## 24. **How to Authenticate to Azure (Azure AD Basics)**

**Azure AD (now Microsoft Entra ID):**
- Cloud-based identity and access management service
- Centralized authentication for Azure resources

**Authentication methods:**

1. **User accounts**: Interactive login (portal, CLI, PowerShell)
   - Username/password
   - Multi-factor authentication (MFA)
   - Conditional access policies

2. **Service principals**: Non-interactive, application authentication
   - Client ID + secret or certificate

3. **Managed identities**: Automatic identity for Azure resources
   - No credentials to manage

4. **Azure CLI/PowerShell**: `az login` or `Connect-AzAccount`

5. **SDKs**: DefaultAzureCredential (tries multiple methods)

**Authorization:**
- **RBAC (Role-Based Access Control)**: Assign roles (Owner, Contributor, Reader)
- **Scope levels**: Management group, subscription, resource group, resource

---

## 25. **Service Principal and When to Use One**

**What it is:**
- An identity for applications, services, or automation tools
- Like a "user account" for non-human entities
- Consists of: Application ID (client ID) + credential (secret or certificate)

**When to use:**
- **CI/CD pipelines**: Azure DevOps, GitHub Actions deploying to Azure
- **Automation scripts**: Terraform, Ansible, custom scripts
- **Third-party applications**: Apps accessing Azure resources
- **Cross-tenant access**: Multi-tenant applications
- **Service-to-service authentication**: APIs calling Azure services

**Authentication methods:**
- Client secret (password)
- Certificate (more secure)

**Best practices:**
- Use least privilege (minimal RBAC roles)
- Rotate secrets regularly
- Prefer certificates over secrets
- Consider managed identities when possible (no credential management)

---

## 26. **Managed Identity (System-Assigned / User-Assigned)**

**What it is:**
- Automatic Azure AD identity for Azure resources
- **No credentials to manage** (Azure handles it)
- Eliminates need for storing secrets in code

**System-Assigned:**
- Tied to a single Azure resource lifecycle
- Created/deleted with the resource
- 1:1 relationship (one identity per resource)
- **Use when**: Identity only needed for that specific resource

**User-Assigned:**
- Standalone Azure resource
- Can be assigned to multiple resources
- Independent lifecycle
- **Use when**: Multiple resources need same identity, or identity should outlive resources

**How it works:**
1. Enable managed identity on resource (VM, App Service, Function, etc.)
2. Grant identity RBAC permissions to target resources
3. Application uses identity to authenticate (no secrets needed)

**Use cases:**
- VM accessing Key Vault
- App Service connecting to SQL Database
- Function App reading from Storage Account

---

## 27. **What is Azure Resource Manager (ARM)?**

**What it is:**
- The deployment and management service for Azure
- Unified management layer for all Azure operations
- Handles all requests from Azure portal, CLI, PowerShell, SDKs, REST API

**Key concepts:**
- **Consistent management**: Same API for all tools
- **Declarative templates**: Define infrastructure as code
- **Resource groups**: Logical containers for resources
- **RBAC**: Role-based access control
- **Tags**: Metadata for organization
- **Locks**: Prevent accidental deletion/modification

**How it works:**
```
User → Tool (Portal/CLI/PowerShell) → ARM → Resource Provider → Resource
```

**Benefits:**
- Deploy, manage, monitor resources as a group
- Consistent deployment across environments
- Define dependencies between resources
- Apply access control and policies
- Track costs and organize resources

---

## 28. **What are ARM Templates? (Basic Concept)**

**What they are:**
- JSON files that define Azure infrastructure declaratively
- Infrastructure as Code (IaC) for Azure
- Describe **what** to deploy, not **how**

**Basic structure:**
```json
{
  "$schema": "...",
  "contentVersion": "1.0.0.0",
  "parameters": { },      // Input values
  "variables": { },       // Reusable values
  "resources": [ ],       // Resources to deploy
  "outputs": { }          // Return values
}
```

**Key features:**
- **Idempotent**: Run multiple times, same result
- **Validation**: Check before deployment
- **Dependencies**: Automatic resource ordering
- **Modular**: Link templates together
- **Version control**: Store in Git

**Use cases:**
- Repeatable deployments (dev, test, prod)
- Disaster recovery
- Compliance and governance
- Multi-region deployments

**Modern alternative:** Bicep (simpler syntax, compiles to ARM)

---

## 29. **What is Azure CLI and How Is It Typically Used?**

**What it is:**
- Cross-platform command-line tool for managing Azure resources
- Available on Windows, macOS, Linux
- Uses `az` command prefix

**Typical usage:**

**Installation:**
```bash
# Install via package manager or download
```

**Authentication:**
```bash
az login
az account set --subscription "subscription-name"
```

**Common operations:**
```bash
# Create resource group
az group create --name myRG --location eastus

# Create VM
az vm create --resource-group myRG --name myVM --image UbuntuLTS

# List resources
az resource list --resource-group myRG

# Deploy ARM template
az deployment group create --resource-group myRG --template-file template.json
```

**When to use:**
- **Automation scripts**: Bash/shell scripts
- **CI/CD pipelines**: Linux-based build agents
- **Cross-platform**: Works on any OS
- **Quick commands**: Interactive management
- **JSON output**: Easy parsing with jq

**Output formats:** JSON (default), table, YAML, TSV

---

## 30. **Azure PowerShell and When to Use It Instead of CLI**

**What it is:**
- PowerShell module for managing Azure resources
- Uses cmdlets with `Verb-AzNoun` naming (e.g., `Get-AzVM`)
- Built on .NET

**When to use PowerShell instead of CLI:**

1. **Windows-centric environments**: Native PowerShell integration
2. **Complex scripting**: Object-oriented, rich data manipulation
3. **Existing PowerShell infrastructure**: Leverage existing scripts/modules
4. **Active Directory integration**: Better AD cmdlets
5. **Object pipelines**: Pass objects between cmdlets
6. **Windows administrators**: Familiar syntax and patterns

**Example:**
```powershell
# Connect
Connect-AzAccount

# Create resource group
New-AzResourceGroup -Name myRG -Location eastus

# Create VM
New-AzVM -ResourceGroupName myRG -Name myVM -Image UbuntuLTS

# Get resources (returns objects)
Get-AzResource -ResourceGroupName myRG | Where-Object {$_.Type -eq "Microsoft.Compute/virtualMachines"}
```

**CLI vs. PowerShell:**
| Feature | Azure CLI | Azure PowerShell |
|---------|-----------|------------------|
| Syntax | Bash-style | PowerShell cmdlets |
| Output | JSON/text | Objects |
| Platform | Cross-platform first | Windows-native |
| Scripting | Shell scripts | PowerShell scripts |
| Learning curve | Simpler | Steeper (but powerful) |

**Choice:** Use CLI for simplicity and cross-platform; use PowerShell for complex Windows automation

---

