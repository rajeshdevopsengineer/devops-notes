## Basic Questions (100)
# 1. What is Microsoft Azure, and what are its Main Service Categories?

[Microsoft Azure](https://azure.microsoft.com/?utm_source=chatgpt.com) is Microsoft’s cloud computing platform that provides services for building, deploying, managing, and scaling applications and infrastructure through Microsoft-managed datacenters worldwide.

Azure supports:

* Infrastructure hosting
* Application development
* Networking
* Security
* Databases
* AI/ML
* DevOps automation

---

## Main Azure Service Categories

| Category            | Purpose                       | Examples                                                                           |
| ------------------- | ----------------------------- | ---------------------------------------------------------------------------------- |
| Compute             | Run applications and VMs      | Azure Virtual Machines, Azure App Service                                          |
| Storage             | Store data and files          | Azure Blob Storage                                                                 |
| Networking          | Connectivity and security     | Azure Virtual Network                                                              |
| Databases           | Managed databases             | Azure SQL Database                                                                 |
| Security & Identity | Authentication and protection | Microsoft Entra ID                                                                 |
| Monitoring          | Logs and performance tracking | Azure Monitor                                                                      |
| DevOps              | CI/CD and automation          | [Azure DevOps](https://azure.microsoft.com/products/devops?utm_source=chatgpt.com) |
| AI & Analytics      | Data processing and AI        | Azure Machine Learning                                                             |

---

# 2. Differentiate between IaaS, PaaS, and SaaS with Azure Examples

| Feature            | IaaS                        | PaaS                      | SaaS                     |
| ------------------ | --------------------------- | ------------------------- | ------------------------ |
| Full Form          | Infrastructure as a Service | Platform as a Service     | Software as a Service    |
| User Manages       | OS, apps, runtime           | Application & data        | Only usage/configuration |
| Azure Manages      | Hardware & virtualization   | Infrastructure + platform | Everything               |
| Flexibility        | High                        | Medium                    | Low                      |
| Maintenance Effort | High                        | Medium                    | Very Low                 |

---

## IaaS

Infrastructure is provided as virtual resources.

### Azure Examples

* Azure Virtual Machines
* Azure Virtual Network

### Use Case

Lift-and-shift migration of on-prem servers.

---

## PaaS

Azure manages infrastructure and platform; users deploy applications.

### Azure Examples

* Azure App Service
* Azure SQL Database

### Use Case

Web app deployment without managing servers.

---

## SaaS

Complete software delivered over the internet.

### Examples

* [Microsoft 365](https://www.microsoft.com/microsoft-365?utm_source=chatgpt.com)
* [Dynamics 365](https://dynamics.microsoft.com/?utm_source=chatgpt.com)

### Use Case

Using Outlook or Teams through a browser.

---

# 3. What are Azure Regions and Availability Zones?

## Azure Regions

A region is a geographical location containing Azure datacenters.

### Examples

* West India
* Central India

### Benefits

* Low latency
* Compliance
* Disaster recovery

---

## Availability Zones

Availability Zones are physically separate datacenters inside a region.

### Benefits

* High availability
* Fault isolation
* Protection against datacenter failure

---

## Interview Answer

“Regions are geographical Azure datacenter locations, while Availability Zones are isolated datacenters within a region used for high availability.”

---

# 4. What is a Resource Group in Azure?

A Resource Group is a logical container that holds related Azure resources.

### Resources Can Include

* VMs
* VNets
* Databases
* Storage accounts

---

## Benefits

* Resource organization
* RBAC management
* Cost tracking
* Lifecycle management

---

## Important Point

Deleting a Resource Group deletes all resources inside it.

---

## Interview Answer

“A Resource Group is a logical container used to organize and manage Azure resources together.”

---

# 5. Difference Between ARM and Classic Deployment

| Feature                  | ARM       | Classic       |
| ------------------------ | --------- | ------------- |
| Deployment Type          | Modern    | Legacy        |
| Resource Groups          | Supported | Not Supported |
| RBAC                     | Supported | Limited       |
| Templates                | ARM/Bicep | No            |
| Tags                     | Supported | No            |
| Microsoft Recommendation | Yes       | No            |

---

## ARM

Azure Resource Manager is Azure’s modern deployment model.

### Features

* Infrastructure as Code
* RBAC
* Policy enforcement
* Template deployments

---

## Interview Answer

“ARM is the modern Azure deployment framework supporting Resource Groups, RBAC, templates, and governance, while Classic deployment is the older legacy model.”

---

# 6. What is an Azure Subscription?

An Azure Subscription is a logical unit for:

* Billing
* Resource management
* Access control
* Service quotas

---

## Use Cases

Separate subscriptions for:

* Dev
* Test
* Production

---

## Interview Answer

“An Azure Subscription is a billing and management boundary used to organize Azure resources and control access.”

---

# 7. What are Management Groups in Azure?

Management Groups organize multiple subscriptions hierarchically for centralized governance.

---

## Benefits

* Centralized policy management
* RBAC inheritance
* Enterprise governance

---

## Example Hierarchy

Tenant Root Group
→ Production MG
→ Development MG
→ Multiple Subscriptions

---

## Interview Answer

“Management Groups are hierarchical containers used to manage and govern multiple Azure subscriptions centrally.”

---

# 8. What are Azure Tags?

Azure Tags are key-value metadata pairs attached to resources.

---

## Example

| Key         | Value      |
| ----------- | ---------- |
| Environment | Production |
| Owner       | DevOpsTeam |
| CostCenter  | Finance    |

---

## Benefits

* Cost management
* Automation
* Organization
* Governance

---

## Interview Answer

“Azure Tags are metadata key-value pairs used to categorize and manage Azure resources.”

---

# 9. What are Availability Sets in Azure?

Availability Sets distribute VMs across:

* Fault Domains
* Update Domains

to improve availability.

---

## Fault Domain

Protects against hardware failure.

## Update Domain

Protects during planned maintenance.

---

## Benefits

* Reduced downtime
* Improved SLA
* High availability

---

## Interview Answer

“Availability Sets improve VM availability by distributing virtual machines across fault domains and update domains.”

---

# 10. What are VM Scale Sets?

Azure Virtual Machine Scale Sets allow automatic deployment and scaling of identical VMs.

---

## Features

* Auto-scaling
* Load balancing
* High availability
* Centralized management

---

## Use Cases

* Web applications
* APIs
* Microservices
* High-traffic workloads

---

## Example

Automatically scale from:

* 2 VMs → 10 VMs
  during peak traffic.

---

## Interview Answer

“VM Scale Sets are Azure services used to automatically deploy and scale multiple identical virtual machines based on workload demand.”

# 11. What is the Difference Between Availability Sets and Availability Zones?

| Feature            | Availability Set                | Availability Zone                  |
| ------------------ | ------------------------------- | ---------------------------------- |
| Scope              | Inside a single datacenter      | Across multiple datacenters        |
| Protection Against | Hardware failures & maintenance | Entire datacenter failure          |
| Architecture       | Fault Domains & Update Domains  | Physically separate zones          |
| SLA                | High availability               | Higher availability                |
| Network Latency    | Very low                        | Slightly higher                    |
| Use Case           | Basic HA for VMs                | Enterprise-grade disaster recovery |

---

## Availability Sets

Availability Sets distribute VMs across:

* Fault Domains
* Update Domains

### Benefits

* Protects against hardware failure
* Protects during planned maintenance

### Example

Two application VMs placed in different fault domains.

---

## Availability Zones

Availability Zones are physically separate datacenters within the same Azure region.

### Benefits

* Protects against entire datacenter outage
* Better disaster recovery
* Higher resiliency

### Example

Deploying VMs across:

* Zone 1
* Zone 2
* Zone 3

in the same region.

---

## Interview Answer

“Availability Sets protect VMs within the same datacenter using fault and update domains, whereas Availability Zones provide higher availability by distributing resources across physically separate datacenters in a region.”

---

# 12. What is Azure Bastion?

Azure Bastion is a fully managed Azure service that provides secure RDP and SSH connectivity to Azure VMs through the Azure Portal without exposing public IP addresses.

---

## Key Features

* Browser-based RDP/SSH
* No public IP required on VMs
* Secure access over HTTPS
* Protection against port scanning

---

## Benefits

* Improved security
* Reduced attack surface
* Centralized secure VM access

---

## How It Works

1. Bastion is deployed inside a VNet.
2. Admin connects through Azure Portal.
3. Traffic stays within Azure backbone network.

---

## Use Case

Securely access production Linux and Windows VMs without opening:

* Port 22 (SSH)
* Port 3389 (RDP)

to the internet.

---

## Interview Answer

“Azure Bastion is a managed service that enables secure RDP and SSH access to Azure VMs directly through the Azure Portal without requiring public IP addresses.”

---

# 13. What is Azure Virtual Network (VNet)?

Azure Virtual Network is a logically isolated private network in Azure used for secure communication between Azure resources and external networks.

---

## Functions of VNet

* Resource communication
* Network isolation
* Internet connectivity
* Hybrid connectivity
* Traffic routing

---

## Resources Connected to VNet

* Virtual Machines
* AKS clusters
* Databases
* App services (integration)

---

## Features

* Subnets
* NSGs
* Route tables
* VPN Gateway
* Peering

---

## Real-Time Example

A production VNet may contain:

* Web subnet
* App subnet
* Database subnet

for multi-tier application architecture.

---

## Interview Answer

“Azure VNet is a private virtual network in Azure that enables secure communication between Azure resources, on-premises networks, and the internet.”

---

# 14. What are Subnets in Azure?

Subnets are logical subdivisions of a Virtual Network.

They help organize and isolate resources within a VNet.

---

## Example

VNet: 10.0.0.0/16

| Subnet    | CIDR        |
| --------- | ----------- |
| WebSubnet | 10.0.1.0/24 |
| AppSubnet | 10.0.2.0/24 |
| DBSubnet  | 10.0.3.0/24 |

---

## Benefits

* Traffic isolation
* Better security
* Network segmentation
* Easier management

---

## Common Use Cases

* Separate web and database tiers
* Apply different NSGs
* Control traffic flow

---

## Important Point

Resources in different subnets communicate internally unless restricted by NSGs or route tables.

---

## Interview Answer

“Subnets are logical network segments within an Azure VNet used for organizing resources, improving security, and controlling network traffic.”

---

# 15. What is the Difference Between Public and Private IPs?

| Feature           | Public IP               | Private IP             |
| ----------------- | ----------------------- | ---------------------- |
| Accessibility     | Internet accessible     | Internal network only  |
| Communication     | External communication  | Internal communication |
| Security Exposure | Higher                  | Lower                  |
| Assigned To       | Public-facing resources | Internal resources     |
| Example Use       | Web server              | Database server        |

---

## Public IP Address

A Public IP allows Azure resources to communicate over the internet.

### Examples

* Web servers
* VPN gateways
* Load balancers

---

## Features

* Internet routable
* Globally unique
* Can be static or dynamic

---

## Private IP Address

A Private IP is used for internal communication inside a VNet.

### Examples

* Internal databases
* Backend servers
* Application servers

---

## Features

* Not internet accessible
* More secure
* Used within private networks

---

## Real-Time Example

| Resource      | IP Type    |
| ------------- | ---------- |
| Web VM        | Public IP  |
| SQL Server VM | Private IP |

---

## Interview Answer

“Public IPs are internet-accessible addresses used for external communication, while Private IPs are used for secure internal communication within Azure VNets.”

# 16. What is Network Security Group (NSG)?

Azure Network Security Group is a security feature in Azure used to filter and control inbound and outbound network traffic for Azure resources.

It acts like a virtual firewall at:

* Subnet level
* Network Interface (NIC) level

---

## NSG Rules

NSG rules are based on:

* Source IP
* Destination IP
* Port
* Protocol
* Direction
* Priority

---

## Types of Rules

| Rule Type      | Purpose                  |
| -------------- | ------------------------ |
| Inbound Rules  | Control incoming traffic |
| Outbound Rules | Control outgoing traffic |

---

## Example

Allow:

* HTTP (Port 80)
* HTTPS (Port 443)

Block:

* All unnecessary internet traffic

---

## Components of NSG Rule

| Component   | Example    |
| ----------- | ---------- |
| Source      | Any        |
| Destination | VM subnet  |
| Port        | 22         |
| Protocol    | TCP        |
| Action      | Allow/Deny |
| Priority    | 100        |

---

## Use Cases

* Restrict SSH/RDP access
* Secure application tiers
* Network segmentation
* Protect backend servers

---

## Interview Answer

“An NSG is an Azure security feature used to filter inbound and outbound traffic using security rules applied at subnet or NIC level.”

---

# 17. What is Application Security Group (ASG)?

Azure Application Security Group is a logical grouping of virtual machines used to simplify NSG rule management.

Instead of using IP addresses, ASGs allow rules based on application groups.

---

## Example

### ASGs

* Web-ASG
* App-ASG
* DB-ASG

### NSG Rule

Allow:

* Web-ASG → App-ASG on Port 8080

---

## Benefits

* Simplified NSG management
* Easier scalability
* Reduced administrative overhead
* Application-centric security

---

## Real-Time Use Case

When new web servers are added:

* Add them to Web-ASG
* Existing NSG rules automatically apply

---

## Interview Answer

“ASG is a logical grouping of Azure VMs that simplifies NSG rule management by using application groups instead of IP addresses.”

---

# 18. What is Azure Load Balancer?

Azure Load Balancer is a Layer 4 (Transport Layer) load balancer that distributes incoming network traffic across multiple backend resources.

It supports:

* TCP
* UDP

---

## Types of Azure Load Balancer

| Type                   | Purpose                       |
| ---------------------- | ----------------------------- |
| Public Load Balancer   | Internet-facing applications  |
| Internal Load Balancer | Internal/private applications |

---

## Features

* High availability
* Automatic traffic distribution
* Health probes
* Zone redundancy
* Auto scaling support

---

## Use Cases

* Web server farms
* High-availability applications
* VM Scale Sets
* Backend services

---

## Example

Traffic distributed across:

* VM1
* VM2
* VM3

for better performance and availability.

---

## Interview Answer

“Azure Load Balancer is a Layer 4 load balancing service that distributes TCP/UDP traffic across multiple Azure resources for high availability and scalability.”

---

# 19. Difference Between Azure Load Balancer and Application Gateway

| Feature                  | Azure Load Balancer       | Azure Application Gateway |
| ------------------------ | ------------------------- | ------------------------- |
| OSI Layer                | Layer 4                   | Layer 7                   |
| Traffic Type             | TCP/UDP                   | HTTP/HTTPS/WebSocket      |
| Routing Capability       | Port-based                | URL/path-based            |
| SSL Offloading           | No                        | Yes                       |
| Web Application Firewall | No                        | Yes                       |
| Session Affinity         | Limited                   | Supported                 |
| Best For                 | Generic traffic balancing | Web applications          |

---

## Azure Load Balancer

### Works At

Transport layer (L4)

### Best For

* Non-HTTP traffic
* TCP/UDP workloads
* High-performance applications

---

## Azure Application Gateway

Azure Application Gateway is a Layer 7 web traffic load balancer.

### Features

* URL-based routing
* SSL termination
* Cookie-based session affinity
* WAF integration

---

## Example

### Load Balancer

Distributes traffic to:

* SQL servers
* Game servers
* VM Scale Sets

### Application Gateway

Routes:

* /api → Backend API
* /images → Image servers

---

## Interview Answer

“Azure Load Balancer works at Layer 4 and distributes TCP/UDP traffic, whereas Application Gateway works at Layer 7 and provides advanced web routing features like URL-based routing, SSL termination, and WAF.”

---

# 20. What is Azure Front Door?

Azure Front Door is a global Layer 7 load balancing and application acceleration service.

It routes user traffic to the nearest and healthiest backend application globally.

---

## Key Features

* Global load balancing
* CDN capabilities
* SSL offloading
* Web Application Firewall
* URL-based routing
* Fast failover

---

## How It Works

Users connect to the nearest Microsoft edge location, and Front Door routes traffic to the optimal backend.

---

## Benefits

* Reduced latency
* Improved performance
* Global high availability
* Enhanced security

---

## Use Cases

* Global web applications
* Multi-region deployments
* Disaster recovery
* Content acceleration

---

## Example

Users from:

* India → Routed to India region
* Europe → Routed to Europe region

automatically.

---

## Interview Answer

“Azure Front Door is a global Layer 7 application delivery and load balancing service that improves performance, security, and availability by routing traffic to the nearest healthy backend globally.”

# 21. What is Azure ExpressRoute?

Azure ExpressRoute is a private dedicated network connection between an on-premises environment and Azure datacenters.

Unlike internet-based connections, ExpressRoute uses a private connection through a connectivity provider.

---

## Key Features

* Private connectivity
* Low latency
* High bandwidth
* More reliable than internet
* Enhanced security

---

## Connectivity Types

* Any-to-any IP VPN
* Point-to-point Ethernet
* Co-location connectivity

---

## Benefits

* Improved performance
* Secure hybrid connectivity
* Predictable network performance
* Reduced internet exposure

---

## Use Cases

* Hybrid cloud deployments
* Large enterprise migrations
* Disaster recovery
* High-volume data transfer

---

## Interview Answer

“Azure ExpressRoute provides a private, dedicated connection between on-premises infrastructure and Azure without using the public internet.”

---

# 22. What is the Difference Between ExpressRoute and VPN Gateway?

| Feature        | ExpressRoute           | VPN Gateway               |
| -------------- | ---------------------- | ------------------------- |
| Connectivity   | Private dedicated link | Encrypted internet tunnel |
| Internet Usage | No                     | Yes                       |
| Security       | Higher                 | Good                      |
| Performance    | More reliable          | Depends on internet       |
| Latency        | Low and consistent     | Variable                  |
| Speed          | Up to 100 Gbps         | Lower compared to ER      |
| Cost           | Expensive              | Cost-effective            |
| Best For       | Enterprises            | Small/medium workloads    |

---

## VPN Gateway

Azure VPN Gateway provides secure encrypted connectivity between:

* Azure VNets
* On-premises networks
  over the internet.

---

## Interview Answer

“ExpressRoute provides private dedicated connectivity with higher reliability and performance, while VPN Gateway uses encrypted internet-based tunnels for secure connectivity.”

---

# 23. What is Azure Blob Storage?

Azure Blob Storage is Azure’s object storage solution for storing massive amounts of unstructured data.

---

## Examples of Blob Data

* Images
* Videos
* Backups
* Logs
* Documents
* Data lake files

---

## Features

* Highly scalable
* Durable
* Secure
* Accessible over HTTP/HTTPS

---

## Components

| Component       | Description         |
| --------------- | ------------------- |
| Storage Account | Top-level container |
| Container       | Similar to a folder |
| Blob            | Actual file/object  |

---

## Use Cases

* Backup and archival
* Media storage
* Big data analytics
* Static website hosting

---

## Interview Answer

“Azure Blob Storage is a scalable object storage service used to store unstructured data such as files, images, videos, backups, and logs.”

---

# 24. What are the Types of Blob Storage?

Azure Blob Storage supports three blob types:

| Blob Type   | Purpose                      |
| ----------- | ---------------------------- |
| Block Blob  | General-purpose file storage |
| Page Blob   | Random read/write operations |
| Append Blob | Append-only logging data     |

---

# Block Blob

Used for:

* Documents
* Images
* Videos
* Backups

### Features

* Most commonly used
* Optimized for streaming and uploads

---

# Page Blob

Used mainly for:

* Virtual machine disks

### Features

* Random read/write access
* Stores data in pages

---

# Append Blob

Used for:

* Logging
* Audit trails

### Features

* Data can only be appended
* Optimized for append operations

---

## Interview Answer

“Block blobs are used for general file storage, page blobs are optimized for random read/write operations like VM disks, and append blobs are designed for append-only scenarios such as logging.”

---

# 25. What is Azure File Share?

Azure Files is a fully managed file-sharing service in Azure using the SMB and NFS protocols.

---

## Features

* Shared file storage
* Accessible from cloud and on-premises
* Fully managed
* Supports Windows and Linux

---

## Protocols Supported

* SMB (Server Message Block)
* NFS (Network File System)

---

## Use Cases

* Shared application data
* Lift-and-shift migrations
* Centralized file storage
* User profile storage

---

## Example

Multiple VMs can access the same Azure File Share simultaneously.

---

## Interview Answer

“Azure File Share is a managed file storage service that provides shared SMB/NFS-based file access for Azure and on-premises systems.”

---

# 26. What are Storage Tiers (Hot, Cool, Archive)?

Azure storage tiers optimize cost based on access frequency.

| Tier    | Access Frequency      | Cost                 |
| ------- | --------------------- | -------------------- |
| Hot     | Frequently accessed   | Highest storage cost |
| Cool    | Infrequently accessed | Lower storage cost   |
| Archive | Rarely accessed       | Lowest storage cost  |

---

# Hot Tier

### Best For

* Active application data
* Frequently accessed files

### Features

* Fast access
* Higher storage cost
* Lower access cost

---

# Cool Tier

### Best For

* Backup data
* Short-term archival

### Features

* Lower storage cost
* Higher access charges

---

# Archive Tier

### Best For

* Long-term backup
* Compliance data

### Features

* Cheapest storage
* Retrieval delay
* Highest retrieval cost

---

## Interview Answer

“Hot tier is used for frequently accessed data, Cool tier for infrequently accessed data, and Archive tier for long-term rarely accessed data at the lowest storage cost.”

---

# 27. What is Azure Disk (HDD vs SSD vs Ultra)?

Azure Disk Storage provides persistent block-level storage for Azure VMs.

---

# Types of Azure Disks

| Disk Type    | Performance | Cost      | Best For                       |
| ------------ | ----------- | --------- | ------------------------------ |
| Standard HDD | Low         | Cheapest  | Backup/dev environments        |
| Standard SSD | Medium      | Moderate  | Web servers/general workloads  |
| Premium SSD  | High        | Expensive | Production databases           |
| Ultra Disk   | Very High   | Highest   | High IOPS enterprise workloads |

---

# Standard HDD

### Features

* Magnetic disks
* Low-cost storage
* Lower performance

### Use Cases

* Dev/Test
* Backup workloads

---

# SSD (Standard/Premium)

### Features

* Faster performance
* Lower latency
* Better reliability

### Use Cases

* Production apps
* Databases

---

# Ultra Disk

### Features

* Extremely high IOPS
* Very low latency
* Dynamic performance tuning

### Use Cases

* SAP HANA
* Enterprise databases
* Mission-critical apps

---

## Interview Answer

“Azure managed disks provide persistent VM storage. HDD disks are low-cost and low-performance, SSD disks provide faster performance, and Ultra disks offer extremely high IOPS and low latency for enterprise workloads.”

---

# 28. What is Azure SQL Database?

Azure SQL Database is a fully managed relational database service based on Microsoft SQL Server.

---

## Features

* Fully managed PaaS database
* Automatic patching
* Built-in backups
* High availability
* Auto-scaling

---

## Benefits

* Reduced administration
* Built-in disaster recovery
* Intelligent performance tuning

---

## Use Cases

* Web applications
* Enterprise applications
* SaaS platforms

---

## Interview Answer

“Azure SQL Database is a fully managed PaaS relational database service that provides high availability, automatic backups, patching, and scalability.”

---

# 29. Difference Between SQL Managed Instance and SQL on VM

| Feature            | SQL Managed Instance | SQL on Azure VM |
| ------------------ | -------------------- | --------------- |
| Service Type       | PaaS                 | IaaS            |
| OS Management      | Microsoft            | Customer        |
| SQL Patching       | Automatic            | Manual          |
| Full SQL Control   | Limited              | Full            |
| Backup Management  | Automatic            | Manual          |
| Maintenance Effort | Low                  | High            |
| Compatibility      | Near 100% SQL Server | Full SQL Server |

---

# SQL Managed Instance

Azure SQL Managed Instance provides near full SQL Server compatibility as a managed service.

### Benefits

* Minimal administration
* Automated maintenance
* Easier migration

---

# SQL on VM

SQL Server installed on:

* Azure Virtual Machines

### Benefits

* Full control
* Custom configurations
* OS-level access

---

## Interview Answer

“SQL Managed Instance is a managed PaaS database service with automated maintenance, while SQL on VM provides full administrative control over SQL Server and the operating system.”

---

# 30. What is Azure Cosmos DB?

Azure Cosmos DB is a globally distributed, multi-model NoSQL database service.

---

## Supported APIs

* SQL API
* MongoDB API
* Cassandra API
* Gremlin API
* Table API

---

## Features

* Global distribution
* Low latency
* Automatic scaling
* Multi-region replication
* High availability

---

## Use Cases

* IoT applications
* Gaming platforms
* Real-time analytics
* Global applications

---

## Benefits

* Millisecond response times
* Elastic scalability
* Multi-master replication

---

## Interview Answer

“Azure Cosmos DB is a globally distributed NoSQL database service that provides low latency, automatic scaling, and multi-region replication for modern cloud applications.”

