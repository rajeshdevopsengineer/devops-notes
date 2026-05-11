
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


