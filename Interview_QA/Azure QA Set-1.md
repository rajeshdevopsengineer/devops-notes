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

# 31. What is Azure App Service?

Azure App Service is a fully managed Platform as a Service (PaaS) offering used to build, host, and scale web applications, REST APIs, and mobile backends.

---

## Supported Technologies

* .NET
* Java
* Node.js
* Python
* PHP
* Ruby

---

## Features

* Auto scaling
* Load balancing
* CI/CD integration
* SSL support
* Custom domains
* Built-in security

---

## Types of App Services

| Type        | Purpose                 |
| ----------- | ----------------------- |
| Web Apps    | Host websites/web apps  |
| API Apps    | Host REST APIs          |
| Mobile Apps | Mobile backend services |

---

## Benefits

* No server management
* Fast deployments
* High availability
* Integrated DevOps support

---

## Interview Answer

“Azure App Service is a fully managed PaaS service used to host web applications, APIs, and mobile backends with built-in scaling, security, and CI/CD capabilities.”

---

# 32. What are App Service Deployment Slots?

Deployment Slots are live application environments within an App Service used for safe deployments and testing.

---

## Common Slots

* Production
* Staging
* Testing

---

## Features

* Zero-downtime deployment
* Slot swapping
* Easy rollback
* Environment testing

---

## How It Works

1. Deploy new version to staging slot
2. Test application
3. Swap staging with production

---

## Benefits

* Reduced downtime
* Safer deployments
* Faster rollback

---

## Interview Answer

“Deployment Slots are separate live environments in Azure App Service that allow zero-downtime deployments and easy swapping between staging and production.”

---

# 33. What is Azure Kubernetes Service (AKS)?

Azure Kubernetes Service is a managed Kubernetes container orchestration service in Azure.

It simplifies deployment, scaling, and management of containerized applications.

---

## Features

* Managed control plane
* Auto scaling
* Self-healing
* Load balancing
* Integrated monitoring
* CI/CD support

---

## Benefits

* Reduced Kubernetes management overhead
* High availability
* Faster container deployments

---

## Use Cases

* Microservices
* Container orchestration
* Cloud-native applications

---

## Interview Answer

“AKS is a managed Kubernetes service in Azure used to deploy, scale, and manage containerized applications efficiently.”

---

# 34. Difference Between AKS and ACI

| Feature       | AKS                      | ACI                           |
| ------------- | ------------------------ | ----------------------------- |
| Full Form     | Azure Kubernetes Service | Azure Container Instances     |
| Orchestration | Yes                      | No                            |
| Complexity    | Higher                   | Simple                        |
| Scaling       | Advanced                 | Limited                       |
| Best For      | Production microservices | Short-lived/simple containers |
| Management    | Kubernetes cluster       | Single container execution    |

---

# AKS

Best for:

* Enterprise applications
* Microservices
* Long-running workloads

---

# ACI

Azure Container Instances allows running containers without managing servers or orchestration.

---

## Best For

* Batch jobs
* Quick testing
* Lightweight workloads

---

## Interview Answer

“AKS is a managed Kubernetes orchestration platform for large-scale containerized applications, while ACI is a lightweight serverless container service for quickly running individual containers.”

---

# 35. What is Azure Functions?

Azure Functions is a serverless compute service used to execute code based on events or triggers without managing infrastructure.

---

## Trigger Examples

* HTTP request
* Blob upload
* Queue message
* Timer schedule

---

## Features

* Serverless
* Auto scaling
* Pay-per-execution
* Event-driven

---

## Use Cases

* Automation
* API processing
* Scheduled jobs
* Event processing

---

## Example

Automatically resize images when uploaded to Blob Storage.

---

## Interview Answer

“Azure Functions is a serverless event-driven compute service that executes code on demand without managing servers.”

---

# 36. What is Logic Apps?

Azure Logic Apps is a cloud-based workflow automation service used to integrate applications, systems, and services.

---

## Features

* Visual workflow designer
* Prebuilt connectors
* Low-code/no-code automation
* Enterprise integration

---

## Supported Integrations

* Office 365
* SAP
* Salesforce
* SQL
* Service Bus
* APIs

---

## Use Cases

* Workflow automation
* Email notifications
* Data synchronization
* Business process automation

---

## Example

When an email arrives:

1. Save attachment to Blob Storage
2. Send Teams notification

---

## Interview Answer

“Logic Apps is a low-code workflow automation service used to integrate applications and automate business processes.”

---

# 37. Difference Between Logic Apps and Functions

| Feature           | Logic Apps            | Azure Functions     |
| ----------------- | --------------------- | ------------------- |
| Purpose           | Workflow automation   | Code execution      |
| Development Style | Low-code/no-code      | Code-first          |
| Best For          | Integration workflows | Custom logic        |
| State Management  | Built-in              | Limited             |
| Connectors        | Extensive             | Limited             |
| Complexity        | Simple orchestration  | Complex programming |

---

# Logic Apps

Best for:

* Business workflows
* Integrations
* Automation

---

# Azure Functions

Best for:

* Custom code
* APIs
* Event processing

---

## Interview Answer

“Logic Apps are used for workflow automation and integrations using low-code design, while Azure Functions are used for running custom serverless code triggered by events.”

---

# 38. What is Azure Service Bus?

Azure Service Bus is a fully managed enterprise messaging service used for reliable communication between distributed applications.

---

## Messaging Models

| Model | Description                   |
| ----- | ----------------------------- |
| Queue | One-to-one communication      |
| Topic | One-to-many publish/subscribe |

---

## Features

* Reliable messaging
* Dead-letter queues
* Message ordering
* Transactions
* Duplicate detection

---

## Use Cases

* Microservices communication
* Decoupled applications
* Order processing systems

---

## Example

E-commerce application:

* Order service sends message
* Payment and shipping services consume messages

---

## Interview Answer

“Azure Service Bus is a managed enterprise messaging service that enables secure and reliable communication between distributed applications.”

---

# 39. What is Event Hub?

Azure Event Hubs is a big-data streaming platform and event ingestion service capable of receiving millions of events per second.

---

## Features

* Massive scale ingestion
* Real-time analytics
* Partitioned consumers
* Stream processing integration

---

## Use Cases

* IoT telemetry
* Log streaming
* Application monitoring
* Real-time analytics

---

## Example

Millions of IoT devices sending sensor data continuously.

---

## Interview Answer

“Event Hub is a highly scalable event ingestion and streaming platform used for telemetry, logging, and real-time analytics.”

---

# 40. Difference Between Event Grid and Event Hub

| Feature             | Event Grid              | Event Hub                    |
| ------------------- | ----------------------- | ---------------------------- |
| Purpose             | Event routing           | Event streaming              |
| Communication Style | Reactive events         | High-throughput data streams |
| Scale               | Lightweight events      | Massive telemetry ingestion  |
| Delivery            | Push model              | Pull model                   |
| Best For            | Event-driven automation | Big data streaming           |

---

# Event Grid

Azure Event Grid is an event-routing service for reactive event-driven architectures.

---

## Example

Trigger Function when:

* Blob uploaded
* VM created

---

# Event Hub

Best for:

* Streaming millions of events
* IoT data pipelines

---

## Interview Answer

“Event Grid is used for lightweight event routing and reactive automation, while Event Hub is designed for large-scale event streaming and telemetry ingestion.”

# 41. What is Azure Active Directory (AAD)?

Microsoft Entra ID (formerly Azure Active Directory) is Microsoft’s cloud-based Identity and Access Management (IAM) service.

It helps organizations manage:

* Users
* Groups
* Applications
* Authentication
* Access policies

---

## Features

* Single Sign-On (SSO)
* Multi-Factor Authentication (MFA)
* Conditional Access
* Identity Protection
* Role-Based Access Control (RBAC)

---

## Supported Integrations

* Microsoft 365
* Azure services
* SaaS applications
* On-premises Active Directory

---

## Use Cases

* User authentication
* Secure application access
* Centralized identity management

---

## Interview Answer

“Azure AD, now called Microsoft Entra ID, is Microsoft’s cloud-based identity and access management service used for authentication, authorization, SSO, and secure access management.”

---

# 42. What is Azure AD Domain Services?

Microsoft Entra Domain Services provides managed domain services such as:

* Domain Join
* Group Policy
* LDAP
* Kerberos
* NTLM

without deploying traditional domain controllers.

---

## Features

* Managed Active Directory domain
* Domain join support
* Group Policy support
* Secure LDAP

---

## Benefits

* No domain controller management
* Simplified hybrid identity
* Supports legacy applications

---

## Use Cases

* Legacy app authentication
* Lift-and-shift migrations
* Domain-joined Azure VMs

---

## Interview Answer

“Azure AD Domain Services provides managed domain capabilities such as LDAP, Kerberos, NTLM, and Group Policy without requiring domain controller management.”

---

# 43. Difference Between Azure AD B2B and B2C

| Feature        | Azure AD B2B                   | Azure AD B2C                 |
| -------------- | ------------------------------ | ---------------------------- |
| Purpose        | External partner collaboration | Customer identity management |
| Users          | Vendors, partners, suppliers   | End customers                |
| Authentication | Organization identities        | Social/local identities      |
| Example Login  | Corporate email                | Gmail/Facebook/Phone         |
| Use Case       | Partner access                 | Consumer applications        |

---

# Azure AD B2B

Business-to-Business collaboration allows external users to securely access organizational resources.

### Example

A vendor accessing:

* Teams
* SharePoint
* Azure Portal

using their own company credentials.

---

# Azure AD B2C

Business-to-Consumer identity service for customer-facing applications.

### Supports

* Google login
* Facebook login
* Email/Password authentication

---

## Interview Answer

“Azure AD B2B is used for secure collaboration with external business partners, while Azure AD B2C is used for managing customer identities in public-facing applications.”

---

# 44. What is Conditional Access in AAD?

Conditional Access is a policy-based security feature in:

* Microsoft Entra ID

that controls access based on conditions.

---

## Conditions Can Include

* User identity
* Device compliance
* Location
* Risk level
* Application

---

## Actions

* Allow access
* Deny access
* Require MFA
* Require compliant device

---

## Example

Require MFA when:

* User logs in from outside India

---

## Benefits

* Improved security
* Zero Trust implementation
* Risk-based authentication

---

## Interview Answer

“Conditional Access is a policy engine in Azure AD that controls user access based on conditions such as location, device state, and risk level.”

---

# 45. What is Azure Key Vault?

Azure Key Vault is a secure Azure service used to store and manage:

* Secrets
* Passwords
* Encryption keys
* Certificates

---

## Features

* Centralized secret management
* Hardware Security Module (HSM) support
* Access control integration
* Key rotation

---

## Types of Stored Items

| Item         | Example          |
| ------------ | ---------------- |
| Secrets      | DB passwords     |
| Keys         | Encryption keys  |
| Certificates | SSL certificates |

---

## Benefits

* Improved security
* Prevent hardcoding credentials
* Secure application integration

---

## Use Cases

* Store database credentials
* TLS/SSL certificate management
* Encrypt sensitive data

---

## Interview Answer

“Azure Key Vault is a secure service used to centrally store and manage secrets, keys, and certificates for applications and infrastructure.”

---

# 46. What are Managed Identities?

Managed Identities for Azure Resources provide automatically managed identities for Azure resources to securely authenticate with Azure services.

---

## Types

| Type            | Description                   |
| --------------- | ----------------------------- |
| System-assigned | Linked directly to resource   |
| User-assigned   | Independent reusable identity |

---

## Benefits

* No credential management
* Secure authentication
* Automatic credential rotation

---

## Example

An Azure VM accesses:

* Key Vault
  without storing passwords.

---

## Use Cases

* Secure service-to-service authentication
* Accessing Azure APIs securely

---

## Interview Answer

“Managed Identities provide Azure resources with automatically managed identities to securely authenticate to Azure services without storing credentials.”

---

# 47. What is Azure Monitor?

Azure Monitor is Azure’s centralized monitoring solution for collecting, analyzing, and acting on telemetry data.

---

## Data Collected

* Metrics
* Logs
* Application telemetry
* Events

---

## Features

* Real-time monitoring
* Alerting
* Dashboards
* Performance analysis

---

## Benefits

* Improved visibility
* Faster troubleshooting
* Proactive monitoring

---

## Use Cases

* VM monitoring
* Application monitoring
* Infrastructure health analysis

---

## Interview Answer

“Azure Monitor is a centralized monitoring platform used to collect and analyze metrics, logs, and telemetry from Azure and hybrid environments.”

---

# 48. What is Application Insights?

Azure Application Insights is an Application Performance Monitoring (APM) service within Azure Monitor.

---

## Features

* Application telemetry
* Dependency tracking
* Performance monitoring
* Error tracking
* Live metrics

---

## Data Collected

* Request rates
* Response times
* Exceptions
* User behavior

---

## Use Cases

* Web app monitoring
* API performance analysis
* Troubleshooting application issues

---

## Example

Track slow API response times in production.

---

## Interview Answer

“Application Insights is an APM service used to monitor application performance, detect failures, and analyze user interactions.”

---

# 49. What is Log Analytics?

Azure Log Analytics is a service used to query and analyze log data collected by Azure Monitor.

---

## Features

* Centralized log analysis
* KQL querying
* Alert creation
* Dashboard integration

---

## Data Sources

* Azure resources
* VMs
* Containers
* Applications
* Security logs

---

## Query Language

Uses:

* Kusto Query Language (KQL)

---

## Example Query

Find failed VM logins:

```kql
SecurityEvent
| where EventID == 4625
```

---

## Interview Answer

“Log Analytics is a log management and analytics service used to query and analyze monitoring and security logs using KQL.”

---

# 50. What is Azure Security Center (Defender for Cloud)?

Microsoft Defender for Cloud is a cloud security management and threat protection service for Azure, hybrid, and multi-cloud environments.

---

## Main Functions

| Function          | Description                       |
| ----------------- | --------------------------------- |
| CSPM              | Cloud Security Posture Management |
| Threat Protection | Detect threats and attacks        |
| Recommendations   | Security best practices           |
| Compliance        | Regulatory compliance monitoring  |

---

## Features

* Secure score
* Vulnerability assessment
* Just-In-Time VM access
* Regulatory compliance dashboard
* Threat detection

---

## Benefits

* Improved security posture
* Continuous monitoring
* Automated recommendations

---

## Use Cases

* Hybrid cloud security
* Compliance auditing
* Threat monitoring

---

## Interview Answer

“Microsoft Defender for Cloud is a cloud security management service that provides security posture management, threat protection, compliance monitoring, and security recommendations for Azure and hybrid environments.”

# 51. What is Azure Policy?

Azure Policy is a governance service used to enforce organizational standards and compliance across Azure resources.

It helps ensure resources follow predefined rules.

---

## Functions

* Enforce compliance
* Restrict resource creation
* Audit configurations
* Apply governance rules automatically

---

## Example Policies

* Allow resources only in specific regions
* Require mandatory tags
* Restrict VM SKUs
* Enforce encryption

---

## Policy Effects

| Effect            | Purpose                         |
| ----------------- | ------------------------------- |
| Deny              | Block non-compliant resources   |
| Audit             | Log non-compliance              |
| Append            | Add configuration automatically |
| DeployIfNotExists | Auto-deploy required settings   |

---

## Benefits

* Centralized governance
* Security enforcement
* Regulatory compliance
* Standardization

---

## Interview Answer

“Azure Policy is a governance service used to enforce compliance and organizational standards across Azure resources using policy rules.”

---

# 52. What is Azure Blueprints?

Azure Blueprints allows organizations to define repeatable Azure environment templates with governance controls.

Blueprints can include:

* ARM templates
* Policies
* RBAC assignments
* Resource Groups

---

## Purpose

Automate deployment of compliant cloud environments.

---

## Benefits

* Standardized environments
* Faster provisioning
* Governance automation
* Reduced configuration drift

---

## Example

Create a secure production environment containing:

* VNets
* Policies
* RBAC
* Monitoring

in one deployment.

---

## Interview Answer

“Azure Blueprints provide reusable templates for deploying governed and compliant Azure environments with policies, RBAC, and infrastructure definitions.”

---

# 53. What is Azure Sentinel?

Microsoft Sentinel is a cloud-native Security Information and Event Management (SIEM) and Security Orchestration Automated Response (SOAR) solution.

---

## Features

* Threat detection
* Security analytics
* Incident investigation
* Automated response
* AI-powered analysis

---

## Data Sources

* Azure resources
* Firewalls
* Servers
* Microsoft 365
* Third-party tools

---

## Benefits

* Centralized security monitoring
* Real-time threat detection
* Automated incident response

---

## Use Cases

* SOC operations
* Threat hunting
* Security monitoring

---

## Interview Answer

“Azure Sentinel is a cloud-native SIEM and SOAR platform used for centralized threat detection, security monitoring, investigation, and automated incident response.”

---

# 54. What is Azure Backup?

Azure Backup is a cloud-based backup solution used to protect Azure and on-premises workloads.

---

## Supports Backup Of

* Azure VMs
* SQL databases
* File shares
* On-premises servers

---

## Features

* Centralized backup management
* Encryption
* Long-term retention
* Automated scheduling

---

## Benefits

* Disaster recovery readiness
* Secure backup storage
* Reduced operational overhead

---

## Use Cases

* VM backup
* Database protection
* File recovery

---

## Interview Answer

“Azure Backup is a managed cloud backup service used to protect Azure and on-premises workloads with secure, automated, and scalable backup solutions.”

---

# 55. What is Site Recovery (ASR)?

Azure Site Recovery is a disaster recovery service that replicates workloads to Azure or secondary sites for business continuity.

---

## Features

* VM replication
* Failover/failback
* Disaster recovery orchestration
* Minimal downtime

---

## Supported Environments

* Hyper-V
* VMware
* Physical servers
* Azure VMs

---

## Benefits

* Business continuity
* Reduced downtime
* Automated recovery plans

---

## Use Cases

* DR for production applications
* Datacenter outage recovery

---

## Interview Answer

“Azure Site Recovery is a disaster recovery service that replicates workloads and enables failover to Azure during outages or disasters.”

---

# 56. What is Azure Arc?

Azure Arc extends Azure management and governance to:

* On-premises servers
* Kubernetes clusters
* Multi-cloud environments

---

## Features

* Centralized governance
* Azure Policy integration
* Monitoring
* Security management

---

## Benefits

* Unified management
* Hybrid cloud governance
* Consistent security policies

---

## Use Cases

* Manage AWS/on-prem servers from Azure
* Govern Kubernetes clusters centrally

---

## Interview Answer

“Azure Arc extends Azure management and governance capabilities to on-premises, multi-cloud, and edge environments.”

---

# 57. What is Azure Lighthouse?

Azure Lighthouse enables service providers and enterprises to manage multiple Azure tenants centrally.

---

## Features

* Cross-tenant management
* Delegated resource management
* RBAC integration
* Centralized operations

---

## Benefits

* Simplified MSP management
* Secure delegated access
* Multi-tenant governance

---

## Use Cases

* Managed service providers
* Central IT operations teams

---

## Interview Answer

“Azure Lighthouse enables centralized and secure cross-tenant management of Azure environments using delegated access.”

---

# 58. What are Azure DevOps Pipelines?

[Azure DevOps Pipelines](https://azure.microsoft.com/products/devops/pipelines/?utm_source=chatgpt.com) are CI/CD services used to automate:

* Build
* Test
* Deployment

of applications and infrastructure.

---

## Types

| Pipeline Type | Purpose             |
| ------------- | ------------------- |
| CI Pipeline   | Build and test code |
| CD Pipeline   | Deploy applications |

---

## Features

* YAML pipelines
* Multi-stage deployments
* Integration with GitHub/Azure Repos
* Infrastructure automation

---

## Supported Platforms

* Azure
* AWS
* Kubernetes
* On-premises

---

## Benefits

* Faster deployments
* Automation
* Improved reliability
* Continuous delivery

---

## Interview Answer

“Azure DevOps Pipelines automate application build, testing, and deployment processes using CI/CD workflows.”

---

# 59. Difference Between Azure DevOps and GitHub Actions

| Feature            | Azure DevOps                  | GitHub Actions            |
| ------------------ | ----------------------------- | ------------------------- |
| Platform           | Complete DevOps suite         | CI/CD inside GitHub       |
| Source Control     | Azure Repos/GitHub            | GitHub only               |
| Pipelines          | Advanced enterprise pipelines | Simpler workflows         |
| Project Management | Boards/Test Plans included    | Limited                   |
| Best For           | Enterprise DevOps             | GitHub-centric automation |
| Learning Curve     | Higher                        | Easier                    |

---

# Azure DevOps

Provides:

* Boards
* Repos
* Pipelines
* Test Plans
* Artifacts

---

# GitHub Actions

[GitHub Actions](https://github.com/features/actions?utm_source=chatgpt.com) is GitHub’s automation and CI/CD platform.

---

## Best For

* GitHub-native CI/CD
* Lightweight automation
* Open-source workflows

---

## Interview Answer

“Azure DevOps is a complete enterprise DevOps platform with CI/CD and project management tools, while GitHub Actions is a GitHub-native workflow automation and CI/CD solution.”

---

# 60. What is Infrastructure as Code (IaC) in Azure?

Infrastructure as Code (IaC) is the practice of provisioning and managing infrastructure using code instead of manual processes.

---

## Azure IaC Tools

| Tool                                                                                                    | Purpose                  |
| ------------------------------------------------------------------------------------------------------- | ------------------------ |
| Azure Resource Manager Templates                                                                        | Native Azure IaC         |
| [Bicep](https://learn.microsoft.com/azure/azure-resource-manager/bicep/overview?utm_source=chatgpt.com) | Simplified ARM language  |
| [Terraform](https://www.terraform.io/?utm_source=chatgpt.com)                                           | Multi-cloud IaC          |
| [Ansible](https://www.ansible.com/?utm_source=chatgpt.com)                                              | Configuration automation |

---

## Benefits

* Automation
* Consistency
* Version control
* Faster deployments
* Reduced human error

---

## Example

Deploying:

* VNets
* VMs
* NSGs
* Storage

using ARM/Bicep/Terraform code.

---

## Interview Answer

“Infrastructure as Code in Azure is the practice of automating infrastructure deployment and management using code-based templates and tools such as ARM, Bicep, and Terraform.”

# 61. Difference Between ARM Templates, Bicep, and Terraform

| Feature          | ARM Templates            | Bicep                      | Terraform                |
| ---------------- | ------------------------ | -------------------------- | ------------------------ |
| Type             | Native Azure IaC         | Simplified ARM language    | Multi-cloud IaC          |
| Syntax           | JSON                     | Cleaner declarative syntax | HCL (HashiCorp Language) |
| Complexity       | Verbose                  | Easier than ARM            | Easy and modular         |
| Cloud Support    | Azure only               | Azure only                 | Multi-cloud              |
| State Management | No external state        | No external state          | Uses state file          |
| Learning Curve   | High                     | Medium                     | Medium                   |
| Recommended Use  | Native Azure deployments | Modern Azure IaC           | Multi-cloud environments |

---

# ARM Templates

Azure Resource Manager templates are JSON-based Infrastructure as Code files used to deploy Azure resources.

### Features

* Declarative syntax
* Native Azure integration
* Supports dependency management

### Limitation

* JSON is complex and verbose.

---

# Bicep

[Bicep](https://learn.microsoft.com/azure/azure-resource-manager/bicep/overview?utm_source=chatgpt.com) is a simplified abstraction over ARM templates.

### Features

* Cleaner syntax
* Easier readability
* Native Azure support
* Compiles to ARM templates

### Example

Much shorter and easier than JSON ARM templates.

---

# Terraform

[Terraform](https://www.terraform.io/?utm_source=chatgpt.com) is a multi-cloud Infrastructure as Code tool.

### Features

* Supports Azure, AWS, GCP
* Modular infrastructure
* State management
* Reusable code

---

## Interview Answer

“ARM templates are native JSON-based Azure deployment templates, Bicep is a simplified Azure-native IaC language built on ARM, and Terraform is a multi-cloud IaC tool using HCL with state management capabilities.”

---

# 62. What is Azure Advisor?

Azure Advisor is a personalized cloud consultant that provides recommendations to optimize Azure resources.

---

## Recommendation Categories

| Category               | Purpose               |
| ---------------------- | --------------------- |
| Cost                   | Reduce spending       |
| Security               | Improve protection    |
| Performance            | Improve efficiency    |
| Reliability            | Increase availability |
| Operational Excellence | Improve operations    |

---

## Examples

* Resize underutilized VMs
* Enable backups
* Add redundancy
* Improve security settings

---

## Benefits

* Cost optimization
* Better performance
* Improved security
* Best practice guidance

---

## Interview Answer

“Azure Advisor is a recommendation service that analyzes Azure resources and provides best-practice guidance for cost, security, reliability, and performance optimization.”

---

# 63. What is Azure Cost Management?

Azure Cost Management is a service used to monitor, analyze, and optimize Azure spending.

---

## Features

* Cost analysis
* Budget creation
* Cost alerts
* Resource usage tracking
* Forecasting

---

## Benefits

* Spending visibility
* Budget control
* Cost optimization
* Chargeback/showback reporting

---

## Example

Create alerts when monthly cloud spending exceeds:

* ₹1,00,000

---

## Interview Answer

“Azure Cost Management helps organizations monitor, analyze, and optimize Azure cloud spending through budgeting, reporting, and cost analysis tools.”

---

# 64. Difference Between Reserved Instances and Spot VMs

| Feature       | Reserved Instances            | Spot VMs                |
| ------------- | ----------------------------- | ----------------------- |
| Pricing Model | Long-term commitment discount | Unused Azure capacity   |
| Cost Savings  | Up to ~72%                    | Up to ~90%              |
| Availability  | Guaranteed                    | Can be evicted anytime  |
| Commitment    | 1 or 3 years                  | No commitment           |
| Reliability   | High                          | Low                     |
| Best For      | Stable workloads              | Interruptible workloads |

---

# Reserved Instances

Reserved capacity for:

* 1-year or 3-year term

### Best For

* Production workloads
* Predictable usage

---

# Spot VMs

Azure Spot Virtual Machines use unused Azure capacity at lower cost.

### Limitation

Azure can evict VMs when capacity is needed.

### Best For

* Batch jobs
* CI/CD agents
* Testing workloads

---

## Interview Answer

“Reserved Instances provide discounted pricing for long-term predictable workloads, while Spot VMs offer very low-cost compute using unused Azure capacity but can be interrupted anytime.”

---

# 65. What is Azure Pricing Calculator?

[Azure Pricing Calculator](https://azure.microsoft.com/pricing/calculator/?utm_source=chatgpt.com) is an online tool used to estimate Azure service costs before deployment.

---

## Features

* Cost estimation
* Region-based pricing
* Resource sizing
* Monthly cost forecasting

---

## Supports Estimation For

* VMs
* Storage
* Databases
* Networking
* AKS
* App Services

---

## Benefits

* Budget planning
* Cost comparison
* Architecture estimation

---

## Interview Answer

“Azure Pricing Calculator is a web-based tool used to estimate the cost of Azure resources and services before deployment.”

---

# 66. What is the SLA for Azure VM?

Azure Virtual Machine SLA depends on deployment architecture.

| Deployment Type                           | SLA    |
| ----------------------------------------- | ------ |
| Single VM with Premium SSD                | 99.9%  |
| Two or more VMs in Availability Set       | 99.95% |
| Two or more VMs across Availability Zones | 99.99% |

---

## Important Point

Higher availability requires:

* Availability Sets
* Availability Zones

---

## Interview Answer

“Azure VM SLA ranges from 99.9% to 99.99% depending on whether VMs are deployed standalone, in Availability Sets, or across Availability Zones.”

---

# 67. What is the SLA for Azure App Service?

Azure App Service provides:

* **99.95% SLA**

for applications running in:

* Standard
* Premium
* Isolated tiers

---

## Conditions

Requires:

* Two or more VM instances

---

## Interview Answer

“Azure App Service provides a 99.95% SLA when deployed in supported production tiers with multiple instances.”

---

# 68. What is the SLA for Azure SQL Database?

Azure SQL Database provides:

* **99.99% availability SLA**

---

## Features Supporting SLA

* Built-in HA
* Automatic backups
* Geo-replication
* Automatic failover

---

## Interview Answer

“Azure SQL Database offers a 99.99% availability SLA with built-in high availability and automated failover capabilities.”

---

# 69. What is Azure Hybrid Benefit?

Azure Hybrid Benefit allows organizations to use existing on-premises Windows Server and SQL Server licenses in Azure.

---

## Benefits

* Reduced Azure costs
* License portability
* Better ROI

---

## Eligible Licenses

* Windows Server
* SQL Server

with Software Assurance.

---

## Example

Reuse existing Windows Server licenses for Azure VMs instead of paying full license cost.

---

## Interview Answer

“Azure Hybrid Benefit allows customers to use existing Windows Server and SQL Server licenses with Software Assurance to reduce Azure infrastructure costs.”

---

# 70. What is Azure Dedicated Host?

Azure Dedicated Host provides physical servers dedicated to a single organization.

---

## Features

* Single-tenant physical server
* Hardware isolation
* Compliance support
* License control

---

## Benefits

* Regulatory compliance
* Improved isolation
* Bring-your-own-license support

---

## Use Cases

* Financial institutions
* Government workloads
* Compliance-sensitive applications

---

## Interview Answer

“Azure Dedicated Host provides isolated physical Azure servers dedicated to a single customer for compliance, security, and licensing requirements.”


# 71. What is Trusted Launch VM?

Trusted Launch for Azure Virtual Machines is a security feature that provides enhanced protection for Azure Generation 2 VMs against advanced threats and boot-level attacks.

---

## Security Features

| Feature                                | Purpose                           |
| -------------------------------------- | --------------------------------- |
| Secure Boot                            | Prevent unauthorized bootloaders  |
| vTPM (Virtual Trusted Platform Module) | Secure key and credential storage |
| Boot Integrity Monitoring              | Detect boot tampering             |

---

## Benefits

* Protection from rootkits and malware
* Secure OS boot process
* Improved compliance and security posture

---

## Supported Workloads

* Windows Server
* Linux Generation 2 VMs

---

## Use Cases

* Production workloads
* Financial systems
* Compliance-sensitive applications

---

## Interview Answer

“Trusted Launch VM is an Azure security feature that enhances VM security using Secure Boot, vTPM, and boot integrity monitoring to protect against boot-level attacks.”

---

# 72. What are Availability Zone Aware Services?

Availability Zone aware services are Azure services that can be deployed across multiple Availability Zones for higher availability and fault tolerance.

---

## Benefits

* Protection against datacenter failure
* High availability
* Improved resiliency

---

## Examples of Zone-Aware Services

| Service                   | Capability                |
| ------------------------- | ------------------------- |
| Azure Virtual Machines    | Zone deployment           |
| Azure Load Balancer       | Zone redundant            |
| Azure Kubernetes Service  | Multi-zone node pools     |
| Azure SQL Database        | Zone redundancy           |
| Azure Application Gateway | Zone redundant deployment |

---

## Interview Answer

“Availability Zone aware services are Azure services designed to operate across multiple Availability Zones to improve resiliency and high availability.”

---

# 73. What is Azure DevTest Labs?

Azure DevTest Labs is a service used to quickly create and manage development and testing environments in Azure.

---

## Features

* Preconfigured VM templates
* Cost control policies
* Automated VM shutdown/startup
* Environment automation

---

## Benefits

* Faster test environment provisioning
* Reduced cloud costs
* Simplified lab management

---

## Use Cases

* Developer sandboxes
* QA environments
* Training labs

---

## Example

Automatically shut down non-production VMs after office hours to save costs.

---

## Interview Answer

“Azure DevTest Labs is a service that helps teams quickly create, manage, and optimize development and testing environments while controlling costs.”

---

# 74. What is Azure Container Instances (ACI)?

Azure Container Instances is a serverless container service that allows users to run containers without managing virtual machines or Kubernetes clusters.

---

## Features

* Fast container startup
* Serverless execution
* Pay-per-second billing
* Supports Linux and Windows containers

---

## Benefits

* No infrastructure management
* Quick deployment
* Cost-efficient for short workloads

---

## Use Cases

* Batch jobs
* CI/CD tasks
* Event-driven processing
* Lightweight applications

---

## Interview Answer

“Azure Container Instances is a serverless container platform that runs containers without requiring VM or Kubernetes management.”

---

# 75. Difference Between Docker on VM vs ACI

| Feature                   | Docker on VM         | ACI                          |
| ------------------------- | -------------------- | ---------------------------- |
| Infrastructure Management | User manages VM      | Fully managed                |
| Scaling                   | Manual               | Limited automatic            |
| Startup Time              | Slower               | Faster                       |
| Cost Model                | VM-based             | Per-second container billing |
| Orchestration             | Self-managed         | None                         |
| Best For                  | Persistent workloads | Temporary/simple workloads   |

---

# Docker on VM

Containers run on:

* User-managed Azure VMs

### Responsibilities

* OS patching
* Docker installation
* Scaling
* Monitoring

---

# ACI

Serverless container execution without infrastructure management.

### Best For

* Short-lived workloads
* Quick deployments

---

## Interview Answer

“Docker on VM requires full VM and Docker management, while ACI provides serverless container execution without managing infrastructure.”

---

# 76. What is Azure Firewall?

Azure Firewall is a fully managed, stateful network security service that protects Azure VNets.

---

## Features

* Centralized traffic filtering
* Application rules
* Network rules
* Threat intelligence
* High availability

---

## Supported Filtering

| Type              | Example             |
| ----------------- | ------------------- |
| Network Rules     | IP/Port filtering   |
| Application Rules | FQDN filtering      |
| NAT Rules         | Inbound translation |

---

## Benefits

* Centralized security
* Scalable firewall management
* Hybrid cloud protection

---

## Use Cases

* Secure internet traffic
* Hub-spoke architectures
* Centralized security enforcement

---

## Interview Answer

“Azure Firewall is a managed stateful firewall service used to centrally control and secure inbound and outbound Azure network traffic.”

---

# 77. Difference Between Firewall and NSG

| Feature              | Azure Firewall                    | NSG                        |
| -------------------- | --------------------------------- | -------------------------- |
| Scope                | Network-wide centralized security | Subnet/NIC-level filtering |
| OSI Layer            | Layer 3–7                         | Layer 3–4                  |
| Application Rules    | Supported                         | Not supported              |
| Threat Intelligence  | Yes                               | No                         |
| Logging & Monitoring | Advanced                          | Basic                      |
| NAT Support          | Yes                               | No                         |
| Cost                 | Higher                            | Lower                      |

---

# NSG

Azure Network Security Group filters traffic using basic allow/deny rules.

---

# Firewall

Provides advanced centralized protection and application-level filtering.

---

## Interview Answer

“NSGs provide basic subnet/NIC-level traffic filtering, while Azure Firewall offers centralized advanced security with application filtering, NAT, and threat intelligence.”

---

# 78. What is DDoS Protection?

Azure DDoS Protection protects Azure applications from Distributed Denial-of-Service attacks.

---

## Types

| Type     | Description                    |
| -------- | ------------------------------ |
| Basic    | Default protection             |
| Standard | Enhanced enterprise protection |

---

## Features of Standard Protection

* Adaptive tuning
* Attack analytics
* Mitigation reports
* Cost protection

---

## Benefits

* Improved availability
* Automatic attack mitigation
* Network protection

---

## Use Cases

* Public-facing applications
* APIs
* E-commerce platforms

---

## Interview Answer

“Azure DDoS Protection safeguards Azure applications from distributed denial-of-service attacks through automatic traffic monitoring and attack mitigation.”

---

# 79. What is Private Link?

Azure Private Link provides private connectivity to Azure services over the Microsoft backbone network.

It eliminates public internet exposure.

---

## Features

* Private access to PaaS services
* Secure connectivity
* Data exfiltration protection

---

## Supported Services

* Storage Accounts
* SQL Database
* Key Vault
* Cosmos DB

---

## Benefits

* Improved security
* Private IP connectivity
* Reduced internet exposure

---

## Example

Access:

* Azure SQL Database
  using a private IP inside the VNet.

---

## Interview Answer

“Azure Private Link enables secure private connectivity to Azure services using private IP addresses over the Microsoft backbone network.”

---

# 80. Difference Between Private Endpoint and Service Endpoint

| Feature           | Private Endpoint       | Service Endpoint      |
| ----------------- | ---------------------- | --------------------- |
| IP Address        | Private IP inside VNet | Public service IP     |
| Traffic Path      | Private Link           | Azure backbone        |
| Internet Exposure | No                     | Service still public  |
| Security          | Higher                 | Moderate              |
| DNS Changes       | Required               | Not required          |
| Recommended For   | Maximum security       | Simpler secure access |

---

# Private Endpoint

Uses:

* Private IP address inside VNet

for accessing Azure services privately.

---

# Service Endpoint

Extends VNet identity to Azure services over Azure backbone but still uses public service endpoints.

---

## Interview Answer

“Private Endpoint provides private IP-based access to Azure services with no public exposure, while Service Endpoint secures traffic over the Azure backbone but still uses public service endpoints.”

# 81. What is Azure Resource Health?

Azure Resource Health is a service that provides information about the current and historical health status of Azure resources.

It helps determine whether an issue is caused by:

* Azure platform problems
* User configuration issues

---

## Health States

| State       | Meaning                     |
| ----------- | --------------------------- |
| Available   | Resource is healthy         |
| Unavailable | Resource issue detected     |
| Degraded    | Performance impacted        |
| Unknown     | Health cannot be determined |

---

## Features

* Root cause insights
* Health history
* Maintenance notifications
* Troubleshooting guidance

---

## Use Cases

* VM outage troubleshooting
* Service diagnostics
* Infrastructure monitoring

---

## Interview Answer

“Azure Resource Health provides real-time and historical health information for Azure resources to help identify platform-related issues.”

---

# 82. What is Azure Service Health?

Azure Service Health provides personalized alerts and guidance about Azure service issues, planned maintenance, and outages affecting subscriptions.

---

## Features

* Service issue alerts
* Planned maintenance notifications
* Health advisories
* Regional impact analysis

---

## Benefits

* Proactive monitoring
* Faster incident response
* Reduced downtime

---

## Example

Receive alerts when:

* West India region experiences VM outage

---

## Interview Answer

“Azure Service Health provides personalized alerts and updates about Azure outages, maintenance events, and service issues affecting resources.”

---

# 83. What is Azure Status Page?

[Azure Status Page](https://status.azure.com/?utm_source=chatgpt.com) is a public webpage that displays the global health status of Azure services and regions.

---

## Features

* Real-time Azure outage information
* Regional service status
* Historical incident reports

---

## Difference from Service Health

| Azure Status Page      | Azure Service Health           |
| ---------------------- | ------------------------------ |
| Public information     | Personalized subscription view |
| General service status | Subscription-specific alerts   |

---

## Interview Answer

“The Azure Status Page is a public dashboard showing the current health status of Azure services and regions globally.”

---

# 84. What are Role Assignments in RBAC?

Role Assignments are mappings that grant permissions to users, groups, or identities in Azure using:

* Azure Role-Based Access Control

---

## Components of Role Assignment

| Component          | Description                  |
| ------------------ | ---------------------------- |
| Security Principal | User/group/service principal |
| Role Definition    | Permissions granted          |
| Scope              | Resource level access        |

---

## Common Roles

| Role        | Access           |
| ----------- | ---------------- |
| Owner       | Full access      |
| Contributor | Manage resources |
| Reader      | Read-only access |

---

## Scope Levels

* Management Group
* Subscription
* Resource Group
* Resource

---

## Example

Assign:

* Contributor role
  to DevOps team at Resource Group level.

---

## Interview Answer

“Role Assignments in Azure RBAC grant specific permissions to users, groups, or applications at a defined scope.”

---

# 85. Difference Between RBAC and ABAC in Azure

| Feature         | RBAC                      | ABAC                           |
| --------------- | ------------------------- | ------------------------------ |
| Full Form       | Role-Based Access Control | Attribute-Based Access Control |
| Access Based On | Roles                     | Roles + Attributes             |
| Granularity     | Moderate                  | Fine-grained                   |
| Conditions      | Limited                   | Advanced conditional access    |
| Flexibility     | Lower                     | Higher                         |

---

# RBAC

Access is granted based on predefined roles.

### Example

Contributor role for:

* Entire Resource Group

---

# ABAC

Access decisions include attributes such as:

* Resource tags
* User department
* Environment

---

## Example

Allow access only if:

* Resource tag = Production

---

## Interview Answer

“RBAC grants access based on roles, while ABAC extends RBAC by using attributes and conditions for more granular access control.”

---

# 86. What are Classic Administrators vs RBAC Roles?

| Feature       | Classic Administrators | RBAC Roles      |
| ------------- | ---------------------- | --------------- |
| Model         | Legacy                 | Modern          |
| Granularity   | Limited                | Fine-grained    |
| Scope Control | Subscription only      | Multiple scopes |
| Flexibility   | Low                    | High            |
| Recommended   | No                     | Yes             |

---

# Classic Administrator Roles

Legacy Azure roles:

* Account Administrator
* Service Administrator
* Co-Administrator

---

# RBAC Roles

Modern access control system with:

* Least privilege access
* Granular permissions
* Custom roles

---

## Benefits of RBAC

* Better security
* Scoped permissions
* Easier governance

---

## Interview Answer

“Classic Administrators are legacy subscription-level admin roles, whereas RBAC provides modern granular role-based access management across Azure resources.”

---

# 87. What is Azure Managed Disk vs Unmanaged Disk?

| Feature                  | Managed Disk  | Unmanaged Disk |
| ------------------------ | ------------- | -------------- |
| Management               | Azure-managed | User-managed   |
| Storage Account Handling | Automatic     | Manual         |
| Scalability              | Better        | Limited        |
| Availability             | Higher        | Lower          |
| Simplicity               | Easy          | Complex        |
| Recommended              | Yes           | No             |

---

# Managed Disk

Azure Disk Storage automatically manages storage accounts for VM disks.

---

## Benefits

* Simplified management
* Better scalability
* Improved reliability

---

# Unmanaged Disk

VM disks stored manually in:

* Azure Storage Accounts

---

## Limitations

* Manual storage account management
* Storage scalability concerns

---

## Interview Answer

“Managed Disks are Azure-managed VM disks with automatic scalability and management, while Unmanaged Disks require manual storage account management.”

---

# 88. What is Azure Table Storage?

Azure Table Storage is a NoSQL key-value storage service for storing large amounts of structured non-relational data.

---

## Features

* Schema-less design
* Massive scalability
* Fast access
* Low-cost storage

---

## Structure

| Component | Description            |
| --------- | ---------------------- |
| Table     | Collection of entities |
| Entity    | Row of data            |
| Property  | Column value           |

---

## Use Cases

* Metadata storage
* User profile data
* IoT device data

---

## Interview Answer

“Azure Table Storage is a scalable NoSQL key-value storage service used for structured non-relational data.”

---

# 89. What is Queue Storage?

Azure Queue Storage is a messaging service used to store large numbers of messages for asynchronous communication between application components.

---

## Features

* Reliable messaging
* Decoupled applications
* Asynchronous processing
* Massive scalability

---

## Message Size

Supports messages up to:

* 64 KB

---

## Use Cases

* Background processing
* Task queues
* Order processing systems

---

## Example

Web app places image-processing tasks into a queue for backend workers.

---

## Interview Answer

“Azure Queue Storage is a scalable messaging service used for asynchronous communication between distributed application components.”

---

# 90. What is the Default Retention for Activity Logs?

Azure Activity Logs are retained by default for:

* **90 days**

in Azure.

---

## Activity Logs Track

* Resource creation
* Resource deletion
* Configuration changes
* Administrative operations

---

## Long-Term Retention Options

Logs can be exported to:

* Azure Log Analytics
* Storage Accounts
* Event Hubs

for extended retention and analysis.

---

## Interview Answer

“The default retention period for Azure Activity Logs is 90 days, after which logs can be archived externally for long-term retention.”

# 91. What is Azure Activity Log vs Diagnostic Log?

| Feature              | Activity Log                 | Diagnostic Log                      |
| -------------------- | ---------------------------- | ----------------------------------- |
| Scope                | Subscription-level events    | Resource-level logs                 |
| Purpose              | Administrative operations    | Resource diagnostics and monitoring |
| Captures             | Create/update/delete actions | Performance and operational logs    |
| Default Availability | Enabled automatically        | Must be configured                  |
| Retention            | 90 days by default           | Depends on configuration            |

---

# Azure Activity Log

Azure Activity Log records management-plane events related to Azure resources.

---

## Tracks

* Resource creation
* Resource deletion
* RBAC changes
* Policy updates
* Administrative operations

---

## Example

Track:

* Who deleted a VM
* When NSG rules changed

---

# Diagnostic Logs

Azure Diagnostic Logs capture resource-level operational and performance data.

---

## Tracks

* Application logs
* Metrics
* Firewall logs
* VM guest logs

---

## Example

* SQL query performance
* NSG flow logs
* App Service HTTP logs

---

## Interview Answer

“Activity Logs capture subscription-level administrative actions, while Diagnostic Logs capture detailed operational and performance data from Azure resources.”

---

# 92. What are Soft Delete and Immutable Storage?

# Soft Delete

Azure Blob Soft Delete protects deleted data by allowing recovery within a retention period.

---

## Features

* Recover deleted blobs/files
* Prevent accidental deletion
* Configurable retention period

---

## Example

Recover accidentally deleted Blob Storage files.

---

# Immutable Storage

Azure Immutable Blob Storage provides Write Once Read Many (WORM) protection.

Data cannot be:

* Modified
* Deleted

during retention period.

---

## Benefits

* Compliance support
* Protection against ransomware
* Legal retention requirements

---

## Use Cases

* Financial records
* Compliance archives
* Audit logs

---

## Interview Answer

“Soft Delete allows recovery of deleted data within a retention period, while Immutable Storage prevents data modification or deletion for compliance and security purposes.”

---

# 93. What are Availability Sets Limitations?

Although Availability Sets improve VM availability, they have limitations.

---

## Limitations

| Limitation            | Description                                |
| --------------------- | ------------------------------------------ |
| Same Datacenter       | VMs remain within one datacenter           |
| No Zone Protection    | Does not protect against datacenter outage |
| Limited Fault Domains | Fixed fault/update domains                 |
| Scaling Constraints   | Less flexible than VM Scale Sets           |
| Regional Limitation   | Works only within a region                 |

---

## Important Point

Availability Zones provide better resiliency than Availability Sets.

---

## Interview Answer

“Availability Sets improve VM availability within a datacenter but do not protect against complete datacenter failures like Availability Zones do.”

---

# 94. What is VM Custom Script Extension?

Azure Custom Script Extension is used to execute scripts on Azure Virtual Machines after deployment.

---

## Supported Scripts

* PowerShell
* Bash
* Shell scripts

---

## Use Cases

* Software installation
* VM configuration
* Automation tasks
* Post-deployment setup

---

## Example

Automatically:

* Install NGINX
* Configure IIS
* Deploy application packages

---

## Benefits

* Automated VM setup
* Reduced manual configuration
* Infrastructure automation

---

## Interview Answer

“Custom Script Extension allows administrators to run scripts on Azure VMs for automated configuration and post-deployment tasks.”

---

# 95. What is Desired State Configuration (DSC)?

PowerShell Desired State Configuration is a configuration management framework used to maintain systems in a desired configuration state.

---

## Features

* Infrastructure configuration
* Drift correction
* Automation
* Idempotent configuration

---

## Example

Ensure:

* IIS is installed
* Services remain running
* Specific configurations are maintained

---

## Benefits

* Consistency
* Automated configuration management
* Reduced configuration drift

---

## Interview Answer

“Desired State Configuration (DSC) is a PowerShell-based configuration management framework used to maintain servers in a predefined desired state.”

---

# 96. What is Azure Automation Account?

Azure Automation is a cloud automation service used to automate repetitive operational tasks.

---

## Features

* Runbooks
* Update management
* Configuration management
* Process automation

---

## Benefits

* Reduced manual effort
* Improved operational consistency
* Automated maintenance

---

## Use Cases

* VM start/stop automation
* Scheduled maintenance
* Patch management

---

## Interview Answer

“Azure Automation Account is a service used to automate cloud and infrastructure management tasks using runbooks and configuration management.”

---

# 97. What are Runbooks?

Runbooks are automation scripts used within:

* Azure Automation

to automate tasks and processes.

---

## Runbook Types

| Type       | Description                 |
| ---------- | --------------------------- |
| PowerShell | Standard automation scripts |
| Python     | Python-based automation     |
| Graphical  | Drag-and-drop workflows     |

---

## Use Cases

* Start/stop VMs
* Cleanup resources
* Send alerts
* Scheduled maintenance

---

## Example

Automatically stop Dev VMs every night to save cost.

---

## Interview Answer

“Runbooks are automation workflows in Azure Automation used to execute repetitive administrative and operational tasks.”

---

# 98. What is Azure Resource Graph?

Azure Resource Graph is a service used to query Azure resources across subscriptions using Kusto Query Language (KQL).

---

## Features

* Fast resource inventory
* Cross-subscription queries
* Governance reporting
* Resource discovery

---

## Example Queries

Find:

* All VMs without tags
* Unused public IPs
* Resources in a specific region

---

## Benefits

* Large-scale resource visibility
* Governance auditing
* Faster resource analysis

---

## Interview Answer

“Azure Resource Graph is a query service that enables fast and scalable querying of Azure resources across subscriptions using KQL.”

---

# 99. What is Azure Cloud Shell?

Azure Cloud Shell is a browser-based command-line environment for managing Azure resources.

---

## Supported Shells

* Bash
* PowerShell

---

## Features

* Pre-authenticated
* No local installation required
* Integrated Azure CLI and PowerShell
* Persistent storage

---

## Benefits

* Quick Azure management
* Accessible from anywhere
* Built-in tools

---

## Use Cases

* Azure administration
* Script execution
* Troubleshooting

---

## Interview Answer

“Azure Cloud Shell is a browser-based command-line environment that provides Azure CLI and PowerShell access without requiring local setup.”

---

# 100. What is the Difference Between PowerShell Az Module and Azure CLI?

| Feature            | PowerShell Az Module      | Azure CLI                |
| ------------------ | ------------------------- | ------------------------ |
| Language Style     | PowerShell cmdlets        | Command-line interface   |
| Best For           | Windows admins/automation | Cross-platform scripting |
| Output Format      | PowerShell objects        | JSON                     |
| Scripting Language | PowerShell                | Bash/Shell               |
| Cross Platform     | Yes                       | Yes                      |
| Learning Style     | Verb-based commands       | Linux-style commands     |

---

# PowerShell Az Module

[Azure PowerShell Az Module](https://learn.microsoft.com/powershell/azure/new-azureps-module-az?utm_source=chatgpt.com) provides PowerShell cmdlets for Azure administration.

---

## Example

```powershell id="9smyqg"
Get-AzVM
```

---

# Azure CLI

[Azure CLI](https://learn.microsoft.com/cli/azure/?utm_source=chatgpt.com) is a cross-platform command-line tool for Azure management.

---

## Example

```bash id="uzhhtq"
az vm list
```

---

## Interview Answer

“PowerShell Az Module uses PowerShell cmdlets and object-based scripting mainly for administrators, while Azure CLI uses cross-platform command-line syntax and JSON output suitable for DevOps and automation.”

🔹 Advanced Questions (100)

# 1. How do you secure communication between VNets?

Communication between Azure VNets can be secured using multiple Azure networking and security services.

---

## Common Methods

| Method         | Purpose                             |
| -------------- | ----------------------------------- |
| VNet Peering   | Private communication between VNets |
| NSGs           | Traffic filtering                   |
| Azure Firewall | Centralized security                |
| VPN Gateway    | Encrypted communication             |
| ExpressRoute   | Private dedicated connectivity      |
| Private Link   | Private access to services          |

---

## Best Practices

### Use VNet Peering

* Keeps traffic on Azure backbone network
* Avoids public internet exposure

### Apply NSGs

* Restrict inbound/outbound traffic
* Allow only required ports

### Use Azure Firewall

* Centralized inspection and filtering

### Enable Encryption

* Use VPN/IPSec for hybrid communication

### Use Private Endpoints

* Secure PaaS service access

---

## Example Architecture

* Hub-and-spoke topology
* Shared firewall in hub VNet
* Spoke VNets connected via peering

---

## Interview Answer

“To secure communication between VNets, we use VNet Peering, NSGs, Azure Firewall, Private Link, VPN encryption, and centralized security architectures such as hub-and-spoke.”

---

# 2. What is VNet Peering (Regional vs Global)?

Azure Virtual Network Peering connects two VNets privately using the Azure backbone network.

---

# Regional VNet Peering

Connects VNets:

* Within the same Azure region

### Features

* Low latency
* High bandwidth
* No gateway required

---

# Global VNet Peering

Connects VNets:

* Across different Azure regions

### Example

* West India ↔ East US

---

## Differences

| Feature  | Regional Peering           | Global Peering            |
| -------- | -------------------------- | ------------------------- |
| Scope    | Same region                | Different regions         |
| Latency  | Lower                      | Slightly higher           |
| Use Case | Intra-region communication | Multi-region architecture |

---

## Benefits

* Private connectivity
* No internet exposure
* Simplified networking

---

## Interview Answer

“Regional VNet Peering connects VNets within the same region, while Global VNet Peering connects VNets across different Azure regions using Microsoft’s private backbone network.”

---

# 3. What is UDR (User Defined Routes)?

Azure Route Tables with User Defined Routes (UDRs) allow administrators to control network traffic routing inside Azure VNets.

---

## Purpose

Override Azure’s default routing behavior.

---

## Common Use Cases

* Route traffic through Azure Firewall
* Route traffic through NVA (Network Virtual Appliance)
* Forced tunneling to on-premises network

---

## Route Components

| Component          | Example           |
| ------------------ | ----------------- |
| Destination Prefix | 0.0.0.0/0         |
| Next Hop Type      | Virtual Appliance |
| Next Hop IP        | Firewall IP       |

---

## Example

Force all outbound internet traffic through:

* Azure Firewall

---

## Interview Answer

“User Defined Routes allow custom routing of Azure network traffic through firewalls, virtual appliances, or on-premises networks.”

---

# 4. What is BGP in Azure Networking?

BGP (Border Gateway Protocol) is a dynamic routing protocol used in Azure networking to exchange routes automatically between networks.

---

## Used With

* VPN Gateway
* ExpressRoute

---

## Benefits

* Dynamic route updates
* Automatic failover
* Reduced manual route management

---

## Example

On-premises routers exchange routes dynamically with Azure VPN Gateway.

---

## Advantages

* Better scalability
* Faster failover
* Improved hybrid networking

---

## Interview Answer

“BGP is a dynamic routing protocol used in Azure hybrid networking to automatically exchange routes between Azure and on-premises networks.”

---

# 5. How do you configure multi-region failover in VMs?

Multi-region failover ensures business continuity during regional outages.

---

## Common Architecture

Primary Region:

* West India

Secondary Region:

* Central India

---

## Components Used

| Service               | Purpose                |
| --------------------- | ---------------------- |
| Azure Site Recovery   | VM replication         |
| Azure Traffic Manager | Global traffic routing |
| Azure Load Balancer   | Regional balancing     |
| Availability Zones    | Regional HA            |

---

## Steps

1. Deploy VMs in primary region
2. Replicate VMs using ASR
3. Configure failover policies
4. Use Traffic Manager/Front Door
5. Perform DR testing

---

## Interview Answer

“Multi-region VM failover is implemented using Azure Site Recovery for replication and Traffic Manager or Front Door for global traffic routing during regional outages.”

---

# 6. How do you implement Azure Virtual WAN?

Azure Virtual WAN is a networking service that provides centralized connectivity for:

* Branch offices
* VNets
* Remote users

---

## Components

| Component            | Purpose                  |
| -------------------- | ------------------------ |
| Virtual WAN          | Global network container |
| Virtual Hub          | Central routing hub      |
| VPN Gateway          | Branch connectivity      |
| ExpressRoute Gateway | Private connectivity     |

---

## Implementation Steps

1. Create Virtual WAN
2. Create Virtual Hub
3. Connect VNets
4. Configure VPN/ExpressRoute
5. Enable routing/security policies

---

## Benefits

* Simplified global networking
* Centralized management
* Scalable branch connectivity

---

## Interview Answer

“Azure Virtual WAN centralizes connectivity between VNets, branches, VPNs, and ExpressRoute using a hub-based global networking architecture.”

---

# 7. What are Azure Firewall Premium Features?

Azure Firewall Premium provides advanced threat protection capabilities.

---

## Premium Features

| Feature        | Purpose                           |
| -------------- | --------------------------------- |
| TLS Inspection | Decrypt and inspect HTTPS traffic |
| IDPS           | Intrusion Detection & Prevention  |
| URL Filtering  | Block malicious websites          |
| Web Categories | Filter website categories         |

---

## Benefits

* Advanced threat protection
* Deep packet inspection
* Improved compliance

---

## Use Cases

* Enterprise security
* Zero Trust architecture
* Internet traffic inspection

---

## Interview Answer

“Azure Firewall Premium adds advanced features such as TLS inspection, IDPS, URL filtering, and web category filtering for enhanced network security.”

---

# 8. How do you configure NSG rules vs ASG rules?

# NSG Rules

Azure Network Security Group rules control traffic using:

* IP addresses
* Ports
* Protocols

---

## Example

Allow:

* TCP 443
  from:
* 10.0.1.0/24

---

# ASG Rules

Azure Application Security Group simplify NSG management using application groups instead of IPs.

---

## Example

Allow:

* Web-ASG → App-ASG

---

## Key Difference

| NSG             | ASG                           |
| --------------- | ----------------------------- |
| IP-based rules  | Application-based grouping    |
| Harder to scale | Easier large-scale management |

---

## Interview Answer

“NSG rules filter traffic using IPs and ports, while ASGs simplify NSG management by grouping VMs logically based on application roles.”

---

# 9. What is Just-In-Time VM Access?

Just-In-Time (JIT) VM access is a security feature in:

* Microsoft Defender for Cloud

that reduces exposure of management ports.

---

## How It Works

Ports such as:

* RDP (3389)
* SSH (22)

remain closed by default.

Access is opened:

* Temporarily
* On request
* For approved users/IPs

---

## Benefits

* Reduced attack surface
* Protection from brute-force attacks
* Better VM security

---

## Use Cases

* Production VM administration
* Secure remote management

---

## Interview Answer

“Just-In-Time VM access temporarily opens management ports like RDP and SSH only when needed to reduce security risks.”

---

# 10. How do you secure Bastion host?

Azure Bastion should be secured using multiple best practices.

---

## Security Best Practices

### Deploy in Dedicated Subnet

Use:

* AzureBastionSubnet

---

### Restrict Access Using RBAC

Allow only authorized admins.

---

### Enable MFA

Use:

* Microsoft Entra ID MFA

---

### Use NSGs Carefully

Restrict unnecessary traffic.

---

### Disable Public IPs on VMs

Use Bastion for secure private access only.

---

### Enable Monitoring

Use:

* Azure Monitor
* Diagnostic logs

---

### Use JIT Access

Combine Bastion with:

* JIT VM Access

---

## Interview Answer

“To secure Azure Bastion, use RBAC, MFA, monitoring, dedicated subnets, NSGs, JIT access, and avoid exposing VMs with public IP addresses.”

# 11. What is Disk Bursting in Azure?

Azure Disk Storage bursting allows managed disks to temporarily increase IOPS and throughput beyond their standard performance limits during short-term spikes.

---

## Purpose

Handle sudden workload spikes without permanently upgrading disk size.

---

## Supported Disks

* Premium SSD
* Standard SSD (selected scenarios)

---

## Benefits

* Improved temporary performance
* Cost optimization
* Better application responsiveness

---

## Example

A database workload suddenly requires:

* Higher IOPS during peak transactions

Disk bursting temporarily boosts performance automatically.

---

## Interview Answer

“Disk bursting allows Azure managed disks to temporarily exceed their normal performance limits during workload spikes without permanently increasing disk size.”

---

# 12. How do you resize VMs with Minimal Downtime?

VM resizing changes CPU, RAM, or VM SKU while minimizing application disruption.

---

## Best Practices

### Use Availability Sets or Zones

Maintain redundancy during resize operations.

---

### Verify SKU Availability

Ensure target VM size is available in the region.

---

### Use Load Balancer

Redirect traffic to healthy instances during resize.

---

### Scale-Out Approach

1. Add new VM with desired size
2. Drain traffic from old VM
3. Remove old VM

Best for production workloads.

---

## Direct Resize Steps

1. Stop/deallocate VM
2. Change VM size
3. Start VM

---

## Minimal Downtime Strategy

* Resize secondary node first
* Failover traffic
* Resize primary node

---

## Interview Answer

“To minimize downtime during VM resize, use Availability Sets/Zones, load balancers, and rolling resize strategies or scale-out replacement approaches.”

---

# 13. How do you Backup and Restore VMs?

Azure VM backup and restore is performed using:

* Azure Backup

---

# Backup Process

## Steps

1. Create Recovery Services Vault
2. Configure backup policy
3. Enable VM backup
4. Schedule backups

---

## Backup Types

* Crash-consistent
* Application-consistent

---

# Restore Process

## Restore Options

| Option        | Purpose                  |
| ------------- | ------------------------ |
| Restore VM    | Create new VM            |
| Restore Disk  | Recover disks only       |
| File Recovery | Recover individual files |

---

## Benefits

* Disaster recovery
* Point-in-time recovery
* Automated backups

---

## Interview Answer

“Azure VMs are backed up using Azure Backup through Recovery Services Vaults, allowing full VM, disk, or file-level recovery.”

---

# 14. How do you Encrypt Azure Disks?

Azure disks can be encrypted using:

* Server-Side Encryption (SSE)
* Azure Disk Encryption (ADE)

---

## Encryption Methods

| Method | Description                            |
| ------ | -------------------------------------- |
| SSE    | Azure-managed storage encryption       |
| ADE    | OS-level BitLocker/DM-Crypt encryption |

---

# Using SSE

Encryption occurs automatically at storage level.

---

# Using ADE

Uses:

* BitLocker (Windows)
* DM-Crypt (Linux)

with:

* Azure Key Vault

---

## Benefits

* Data protection
* Compliance
* Secure storage

---

## Interview Answer

“Azure disks are encrypted using Server-Side Encryption or Azure Disk Encryption with Key Vault-managed keys.”

---

# 15. Difference Between SSE and CSE Encryption

| Feature             | SSE                    | CSE                     |
| ------------------- | ---------------------- | ----------------------- |
| Full Form           | Server-Side Encryption | Client-Side Encryption  |
| Encryption Location | Azure storage side     | Client/application side |
| Key Control         | Azure/Customer-managed | Customer-managed        |
| Performance Impact  | Minimal                | Higher                  |
| Complexity          | Simple                 | More complex            |

---

# SSE (Server-Side Encryption)

Encryption occurs:

* Automatically in Azure storage

---

## Types

* Microsoft-managed keys
* Customer-managed keys

---

# CSE (Client-Side Encryption)

Data is encrypted:

* Before sending to Azure

Azure stores only encrypted data.

---

## Benefits

* Full customer control
* Higher security

---

## Interview Answer

“SSE encrypts data at the Azure storage level, while CSE encrypts data before it is sent to Azure, giving customers full encryption control.”

---

# 16. How do you configure RBAC Custom Roles?

Custom roles in:

* Azure Role-Based Access Control

allow organizations to define granular permissions.

---

## Steps

### 1. Create Role Definition JSON

Define:

* Actions
* NotActions
* Assignable scopes

---

## Example Permissions

```json id="1h8v7z"
{
  "Actions": [
    "Microsoft.Compute/*/read",
    "Microsoft.Storage/*/read"
  ]
}
```

---

### 2. Create Role

Using:

* Azure Portal
* PowerShell
* Azure CLI

---

### 3. Assign Role

Assign to:

* User
* Group
* Managed Identity

---

## Benefits

* Least privilege access
* Fine-grained security
* Custom governance

---

## Interview Answer

“RBAC custom roles are configured by defining custom permissions in a role definition and assigning the role to users, groups, or identities.”

---

# 17. How do you configure PIM in Azure AD?

Microsoft Entra Privileged Identity Management (PIM) provides just-in-time privileged access management.

---

## Steps to Configure PIM

### 1. Enable PIM

In:

* Microsoft Entra ID

---

### 2. Select Roles

Examples:

* Global Administrator
* Contributor
* Owner

---

### 3. Configure Assignment Type

| Type     | Description           |
| -------- | --------------------- |
| Eligible | Activated when needed |
| Active   | Permanent access      |

---

### 4. Configure Security Controls

* MFA requirement
* Approval workflow
* Time-bound activation

---

## Benefits

* Reduced privileged exposure
* Better auditing
* Zero Trust security

---

## Interview Answer

“PIM is configured by enabling eligible role assignments with MFA, approval workflows, and time-based activation for privileged Azure access.”

---

# 18. What is Conditional Access MFA Enforcement?

Conditional Access MFA enforcement requires Multi-Factor Authentication based on defined conditions.

---

## Conditions Can Include

* User location
* Device compliance
* Risk level
* Application type

---

## Example Policy

Require MFA when:

* Login occurs outside corporate network

---

## Benefits

* Strong authentication
* Reduced account compromise
* Zero Trust implementation

---

## Components Used

* Microsoft Entra ID
* Conditional Access Policies
* MFA

---

## Interview Answer

“Conditional Access MFA enforcement applies Multi-Factor Authentication dynamically based on policies such as user risk, location, or device compliance.”

---

# 19. What is Identity Governance in Azure AD?

Microsoft Entra ID Governance manages identity lifecycle, access reviews, and privileged access governance.

---

## Features

| Feature                | Purpose                     |
| ---------------------- | --------------------------- |
| Access Reviews         | Validate user access        |
| Entitlement Management | Manage access packages      |
| Lifecycle Workflows    | Automate identity processes |
| PIM                    | Privileged access control   |

---

## Benefits

* Compliance
* Least privilege access
* Automated governance

---

## Use Cases

* Joiner/mover/leaver automation
* Third-party access reviews
* Privileged access auditing

---

## Interview Answer

“Identity Governance in Azure AD manages identity lifecycle, access reviews, and privileged access to improve compliance and security.”

---

# 20. How do you integrate Azure AD with On-Prem AD?

Azure AD integration with on-premises Active Directory is commonly implemented using:

* Microsoft Entra Connect

(previously Azure AD Connect).

---

# Integration Methods

| Method                      | Purpose                    |
| --------------------------- | -------------------------- |
| Password Hash Sync          | Sync passwords to cloud    |
| Pass-Through Authentication | Validate passwords on-prem |
| Federation (ADFS)           | Federated authentication   |

---

# Implementation Steps

### 1. Prepare On-Prem AD

* Verify domain health
* Configure UPN suffixes

---

### 2. Install Entra Connect

On a domain-joined server.

---

### 3. Configure Sync

Choose:

* Users
* Groups
* Password sync method

---

### 4. Enable Hybrid Identity

Users authenticate to:

* Azure
* Microsoft 365
* SaaS apps

using same credentials.

---

## Benefits

* Single Sign-On
* Hybrid identity
* Centralized authentication

---

## Interview Answer

“Azure AD integrates with on-prem AD using Microsoft Entra Connect, which synchronizes users, groups, and authentication between on-premises Active Directory and Azure AD.”

# 21. How do you implement Federation with ADFS?

Active Directory Federation Services (ADFS) enables federated authentication between on-premises Active Directory and cloud services such as Azure.

---

## Purpose

Provide:

* Single Sign-On (SSO)
* Centralized authentication
* Federation-based identity management

---

## Federation Architecture

Users authenticate:

1. Against on-prem AD
2. Through ADFS
3. Access Azure/Microsoft 365 resources

---

## Components Required

| Component             | Purpose                   |
| --------------------- | ------------------------- |
| ADFS Server           | Authentication federation |
| Web Application Proxy | External access           |
| Active Directory      | Identity source           |
| Microsoft Entra ID    | Cloud identity platform   |

---

## Implementation Steps

### 1. Install ADFS

On Windows Server.

### 2. Configure SSL Certificate

Secure federation endpoints.

### 3. Deploy Web Application Proxy

For external authentication requests.

### 4. Configure Entra Connect

Enable:

* Federation with ADFS

### 5. Verify Federation

Users authenticate using on-prem credentials.

---

## Benefits

* Centralized authentication
* Advanced authentication policies
* Smart card integration

---

## Interview Answer

“Federation with ADFS is implemented by integrating on-prem Active Directory with Azure AD using ADFS servers and Entra Connect to provide Single Sign-On and federated authentication.”

---

# 22. What is Hybrid Identity (Cloud Sync vs AAD Connect)?

Hybrid identity allows users to access:

* On-prem resources
* Cloud resources

using the same credentials.

---

# Microsoft Entra Connect (AAD Connect)

Microsoft Entra Connect synchronizes identities between on-prem AD and Azure AD.

---

## Features

* Password Hash Sync
* Pass-through Authentication
* Federation support
* Device sync

---

# Cloud Sync

Microsoft Entra Cloud Sync is a lightweight cloud-managed synchronization solution.

---

## Differences

| Feature            | AAD Connect      | Cloud Sync        |
| ------------------ | ---------------- | ----------------- |
| Installation       | Full sync server | Lightweight agent |
| Complexity         | Higher           | Simpler           |
| Federation Support | Yes              | Limited           |
| Advanced Features  | More features    | Basic sync        |
| Management         | Server-managed   | Cloud-managed     |

---

## Best Practice

* Use AAD Connect for enterprise hybrid identity
* Use Cloud Sync for simpler environments

---

## Interview Answer

“Hybrid identity integrates on-prem AD with Azure AD. AAD Connect provides full-featured synchronization and federation support, while Cloud Sync offers lightweight cloud-managed identity synchronization.”

---

# 23. How do you configure Key Vault Firewalls?

Azure Key Vault firewalls restrict access to Key Vault from trusted networks only.

---

## Configuration Methods

| Method           | Purpose              |
| ---------------- | -------------------- |
| IP Rules         | Allow specific IPs   |
| VNet Rules       | Restrict to VNets    |
| Private Endpoint | Private access       |
| Trusted Services | Allow Azure services |

---

## Steps

### 1. Open Key Vault Networking

Navigate to:

* Networking settings

---

### 2. Disable Public Access (Optional)

Increase security posture.

---

### 3. Add Allowed Networks

* Public IP ranges
* VNets/Subnets

---

### 4. Configure Private Endpoint

Enable private connectivity.

---

## Best Practices

* Use Private Link
* Disable unnecessary public access
* Restrict access using RBAC

---

## Interview Answer

“Key Vault firewalls are configured using IP restrictions, VNet rules, and Private Endpoints to limit access to trusted networks only.”

---

# 24. How do you configure Key Vault Secrets Rotation?

Secret rotation automatically updates secrets periodically to improve security.

---

## Supported Rotation

* Keys
* Certificates
* Secrets (custom automation)

---

## Methods

| Method                   | Purpose                |
| ------------------------ | ---------------------- |
| Native Rotation Policies | Keys/certificates      |
| Azure Automation         | Custom secret rotation |
| Event Grid + Functions   | Automated workflows    |

---

## Common Architecture

1. Secret nearing expiration
2. Event triggered
3. Azure Function rotates secret
4. Application retrieves updated secret

---

## Best Practices

* Use Managed Identities
* Automate secret renewal
* Monitor expiration alerts

---

## Interview Answer

“Key Vault secret rotation is configured using rotation policies and automation tools such as Azure Functions or Automation Accounts to periodically renew secrets securely.”

---

# 25. What is CMK Encryption for Storage?

CMK (Customer-Managed Key) encryption allows customers to manage encryption keys for Azure Storage.

---

## Uses

* Enhanced compliance
* Customer-controlled encryption
* Regulatory requirements

---

## Components

| Component        | Purpose                |
| ---------------- | ---------------------- |
| Azure Key Vault  | Stores encryption keys |
| Storage Account  | Encrypted data         |
| Managed Identity | Secure key access      |

---

## Benefits

* Full key ownership
* Key rotation control
* Compliance support

---

## Difference

| Microsoft-Managed Keys | Customer-Managed Keys  |
| ---------------------- | ---------------------- |
| Azure controls keys    | Customer controls keys |

---

## Interview Answer

“CMK encryption allows customers to use their own encryption keys stored in Azure Key Vault to encrypt Azure Storage data.”

---

# 26. How do you implement Secure SAS Tokens?

SAS (Shared Access Signature) tokens provide delegated temporary access to storage resources.

---

## Best Practices

| Practice                    | Purpose             |
| --------------------------- | ------------------- |
| Use HTTPS only              | Secure transmission |
| Short expiry time           | Reduce exposure     |
| Least privilege permissions | Minimize risk       |
| IP restrictions             | Limit access source |
| Stored Access Policies      | Centralized control |

---

## Types of SAS

| Type                | Description            |
| ------------------- | ---------------------- |
| User Delegation SAS | Uses Azure AD          |
| Service SAS         | Storage service access |
| Account SAS         | Broader storage access |

---

## Example

Generate read-only SAS valid for:

* 30 minutes

---

## Interview Answer

“Secure SAS tokens are implemented using least privilege permissions, short expiration times, HTTPS enforcement, and IP restrictions.”

---

# 27. What are Storage Account Firewall Restrictions?

Storage Account firewall restrictions limit storage access to trusted networks and endpoints.

---

## Supported Restrictions

| Restriction Type       | Purpose                       |
| ---------------------- | ----------------------------- |
| IP Rules               | Allow specific IP ranges      |
| VNet Rules             | Restrict to VNets             |
| Private Endpoints      | Private access                |
| Trusted Azure Services | Allow selected Azure services |

---

## Benefits

* Prevent public exposure
* Improve storage security
* Reduce attack surface

---

## Best Practices

* Disable public access if possible
* Use Private Link
* Enable logging and monitoring

---

## Interview Answer

“Storage Account firewalls restrict access using IP rules, virtual networks, and Private Endpoints to secure storage resources.”

---

# 28. How do you enable Private Link for Storage?

Azure Private Link enables private connectivity to Storage Accounts using private IP addresses.

---

## Steps

### 1. Open Storage Account

Go to:

* Networking → Private Endpoint Connections

---

### 2. Create Private Endpoint

Select:

* Target VNet/Subnet

---

### 3. Configure DNS

Use:

* Private DNS Zone

---

### 4. Approve Connection

Storage traffic now flows privately.

---

## Benefits

* No internet exposure
* Secure access
* Improved compliance

---

## Interview Answer

“Private Link for Storage is enabled by creating a Private Endpoint connected to a VNet and configuring private DNS for secure private access.”

---

# 29. What is Azure Files AD DS Authentication?

Azure Files supports authentication using:

* Active Directory Domain Services (AD DS)

---

## Purpose

Allow domain users to access Azure file shares using:

* Existing AD credentials

---

## Features

* SMB authentication
* NTFS permissions
* Kerberos authentication
* Hybrid identity support

---

## Requirements

* Domain-joined clients
* Network connectivity to AD DS
* Proper RBAC/share permissions

---

## Use Cases

* Lift-and-shift file servers
* Shared enterprise file storage

---

## Interview Answer

“Azure Files AD DS authentication allows users to access Azure file shares using on-premises Active Directory credentials and NTFS permissions.”

---

# 30. How do you configure Cross-Region Replication for Blob Storage?

Cross-region replication is configured using:

* Geo-redundant storage options

in:

* Azure Blob Storage

---

## Replication Types

| Type    | Description                    |
| ------- | ------------------------------ |
| GRS     | Geo-Redundant Storage          |
| RA-GRS  | Read-Access Geo-Redundant      |
| GZRS    | Geo-Zone-Redundant             |
| RA-GZRS | Read-access Geo-Zone-Redundant |

---

## Configuration Steps

### 1. Create/Modify Storage Account

Select replication option.

---

### 2. Choose Replication Type

Example:

* RA-GRS

---

### 3. Enable Failover (Optional)

For disaster recovery testing.

---

## Benefits

* Disaster recovery
* High durability
* Multi-region protection

---

## Example

Primary:

* West India

Secondary:

* South India paired region

---

## Interview Answer

“Cross-region Blob replication is configured using geo-redundant storage options such as GRS or RA-GRS to replicate data automatically to a secondary Azure region.”

# 31. What is ZRS, GRS, and RA-GRS Replication?

Azure Storage provides multiple replication options for durability and disaster recovery.

---

## ZRS (Zone-Redundant Storage)

Zone-Redundant Storage replicates data synchronously across multiple Availability Zones within the same region.

---

### Benefits

* Protects against datacenter failure
* High availability within region
* Low latency replication

---

### Best For

* High availability workloads within a region

---

# GRS (Geo-Redundant Storage)

Geo-Redundant Storage replicates data:

1. Locally within primary region
2. Asynchronously to secondary paired region

---

### Benefits

* Regional disaster recovery
* High durability

---

### Limitation

Secondary region is not readable unless failover occurs.

---

# RA-GRS (Read-Access Geo-Redundant Storage)

Read-Access Geo-Redundant Storage provides all GRS features plus read access to the secondary region.

---

### Benefits

* Read access during outages
* Improved availability

---

## Comparison Table

| Feature               | ZRS | GRS | RA-GRS |
| --------------------- | --- | --- | ------ |
| Zone Protection       | Yes | No  | No     |
| Regional DR           | No  | Yes | Yes    |
| Secondary Read Access | No  | No  | Yes    |

---

## Interview Answer

“ZRS replicates data across Availability Zones in the same region, GRS replicates data to a secondary region for disaster recovery, and RA-GRS additionally provides read access to the secondary region.”

---

# 32. What is Soft Delete for Blobs and Files?

Soft Delete protects deleted storage data by allowing recovery during a retention period.

---

## Blob Soft Delete

Azure Blob Soft Delete allows recovery of deleted blobs.

---

# Azure Files Soft Delete

Azure Files Soft Delete protects deleted file shares.

---

## Features

* Recover accidentally deleted data
* Configurable retention period
* Protection against accidental deletion

---

## Benefits

* Improved data protection
* Easier recovery
* Ransomware mitigation support

---

## Interview Answer

“Soft Delete allows recovery of deleted blobs and Azure file shares within a configured retention period.”

---

# 33. How do you implement Lifecycle Management in Blob?

Lifecycle management automates blob data movement and deletion based on policies.

---

## Used For

* Cost optimization
* Archival automation
* Retention management

---

## Common Actions

| Action               | Example        |
| -------------------- | -------------- |
| Move to Cool Tier    | After 30 days  |
| Move to Archive Tier | After 90 days  |
| Delete Blob          | After 365 days |

---

## Configuration Steps

### 1. Open Storage Account

Go to:

* Lifecycle Management

---

### 2. Create Rule

Define:

* Filters
* Conditions
* Actions

---

### 3. Apply Policy

Policy runs automatically.

---

## Example Policy

Move backups:

* Hot → Cool after 30 days
* Cool → Archive after 180 days

---

## Interview Answer

“Blob lifecycle management automates tier transitions and deletion policies to optimize storage costs and retention management.”

---

# 34. How do you configure Azure SQL Geo-Replication?

Azure SQL Database geo-replication creates readable secondary databases in another Azure region.

---

## Steps

### 1. Create Primary SQL Database

---

### 2. Enable Geo-Replication

Select:

* Add secondary replica

---

### 3. Choose Target Region

Example:

* West India → Central India

---

### 4. Configure Failover

Manual or automatic.

---

## Benefits

* Disaster recovery
* Read scalability
* Regional failover

---

## Use Cases

* Business continuity
* Global applications

---

## Interview Answer

“Azure SQL geo-replication creates synchronized secondary databases in another region for disaster recovery and high availability.”

---

# 35. What is Failover Groups in SQL?

Azure SQL Failover Groups manage automatic failover of multiple SQL databases together across regions.

---

## Features

* Automatic failover
* Read/write listener endpoint
* Multiple database failover
* Policy-based failover

---

## Benefits

* Simplified DR management
* Reduced downtime
* Centralized failover handling

---

## Example

Automatically failover production databases from:

* West India → South India

---

## Interview Answer

“SQL Failover Groups provide automatic cross-region failover and unified endpoints for multiple Azure SQL databases.”

---

# 36. Difference Between Active Geo-Replication and Failover Groups

| Feature            | Active Geo-Replication | Failover Groups          |
| ------------------ | ---------------------- | ------------------------ |
| Failover           | Manual                 | Automatic/manual         |
| Endpoint           | Separate DB endpoints  | Global listener endpoint |
| Multiple Databases | Limited management     | Group management         |
| DR Automation      | Basic                  | Advanced                 |
| Best For           | Single database DR     | Enterprise DR            |

---

# Active Geo-Replication

Provides readable secondary databases.

---

## Best For

* Individual databases
* Read scaling

---

# Failover Groups

Provide automatic coordinated failover.

---

## Best For

* Enterprise applications
* Multi-database DR

---

## Interview Answer

“Active Geo-Replication provides readable secondary databases with manual failover, while Failover Groups add automatic failover and centralized management for multiple databases.”

---

# 37. What is Always Encrypted in SQL?

Always Encrypted protects sensitive SQL data by encrypting it at the client side.

---

## Features

* Client-side encryption
* Sensitive column protection
* Encryption keys controlled by customer

---

## Protected Data Examples

* Credit card numbers
* Aadhaar/PAN data
* Personal information

---

## Benefits

* Data confidentiality
* Protection from DB admins
* Compliance support

---

## Difference from TDE

| Always Encrypted       | TDE                    |
| ---------------------- | ---------------------- |
| Column-level           | Database-level         |
| Client-side encryption | Server-side encryption |

---

## Interview Answer

“Always Encrypted protects sensitive SQL columns using client-side encryption so that plaintext data is never exposed to the database engine.”

---

# 38. What are DTU vs vCore Pricing Models?

Azure SQL Database supports two purchasing models.

---

## DTU Model

DTU = Combined measure of:

* CPU
* Memory
* IO

---

## Features

* Simpler pricing
* Preconfigured performance bundles

---

## Best For

* Smaller/simple workloads

---

# vCore Model

vCore provides separate control over:

* CPU
* Memory
* Storage

---

## Features

* More flexibility
* Better cost optimization
* Supports Azure Hybrid Benefit

---

## Comparison

| Feature                 | DTU     | vCore  |
| ----------------------- | ------- | ------ |
| Simplicity              | High    | Medium |
| Flexibility             | Low     | High   |
| Cost Transparency       | Lower   | Better |
| Enterprise Optimization | Limited | Better |

---

## Interview Answer

“DTU is a bundled performance pricing model, while vCore provides granular control over compute and storage resources with better flexibility.”

---

# 39. How do you secure SQL with Private Endpoint?

Azure Private Link enables private connectivity to Azure SQL Database.

---

## Steps

### 1. Open SQL Server Networking

---

### 2. Create Private Endpoint

Connect SQL server to:

* VNet/Subnet

---

### 3. Configure Private DNS

Use:

* Private DNS Zone

---

### 4. Disable Public Network Access

(Optional but recommended)

---

## Benefits

* No internet exposure
* Private IP access
* Improved security

---

## Best Practices

* Combine with NSGs
* Use firewall restrictions
* Enable auditing

---

## Interview Answer

“Azure SQL is secured with Private Endpoints by assigning a private IP within a VNet and disabling public access.”

---

# 40. How do you optimize Cosmos DB RU Consumption?

Azure Cosmos DB uses Request Units (RUs) to measure throughput consumption.

---

## Optimization Techniques

| Technique                      | Benefit                     |
| ------------------------------ | --------------------------- |
| Proper Partition Keys          | Balanced workload           |
| Optimize Queries               | Reduce RU cost              |
| Use Indexing Wisely            | Reduce unnecessary indexing |
| Use Point Reads                | Lowest RU consumption       |
| Enable Autoscale               | Dynamic throughput          |
| Reduce Cross-Partition Queries | Lower RU usage              |

---

## Best Practices

### Choose Good Partition Key

Avoid:

* Hot partitions

---

### Optimize Indexing Policy

Index only required fields.

---

### Use Pagination

Avoid large query results.

---

### Monitor Metrics

Use:

* RU consumption metrics
* Query insights

---

## Interview Answer

“Cosmos DB RU consumption is optimized by selecting efficient partition keys, reducing cross-partition queries, optimizing indexing, and using point reads and autoscaling.”


# 41. What are Cosmos DB Consistency Models?

Azure Cosmos DB provides five consistency levels that balance:

* Data consistency
* Availability
* Performance
* Latency

---

## Consistency Models

| Consistency Level | Description                             |
| ----------------- | --------------------------------------- |
| Strong            | Always returns latest committed data    |
| Bounded Staleness | Slight delay allowed                    |
| Session           | Consistent within user session          |
| Consistent Prefix | Reads in correct order only             |
| Eventual          | Lowest consistency, fastest performance |

---

# Strong Consistency

### Features

* Highest consistency
* Latest data guaranteed

### Tradeoff

* Higher latency
* Lower availability

---

# Bounded Staleness

### Features

* Reads may lag slightly
* Predictable staleness window

---

# Session Consistency

(Default)

### Features

* User/session-level consistency
* Good balance of performance and consistency

---

# Consistent Prefix

### Features

* Maintains write order
* May not show latest updates

---

# Eventual Consistency

### Features

* Fastest reads/writes
* Lowest consistency guarantee

---

## Interview Answer

“Cosmos DB provides five consistency models—Strong, Bounded Staleness, Session, Consistent Prefix, and Eventual—to balance consistency, latency, and availability requirements.”

---

# 42. How do you configure Multi-Region Writes in Cosmos DB?

Multi-region writes allow applications to write data to multiple Azure regions simultaneously.

---

## Benefits

* Lower write latency
* Global application performance
* High availability

---

## Configuration Steps

### 1. Open Cosmos DB Account

---

### 2. Add Multiple Regions

Configure:

* Read regions
* Write regions

---

### 3. Enable Multi-Region Writes

Turn on:

* Multi-region write capability

---

### 4. Configure Conflict Resolution

Choose:

* Last-write-wins
* Custom conflict resolver

---

## Example

Users in:

* India write to West India
* US users write to East US

simultaneously.

---

## Interview Answer

“Multi-region writes are configured by enabling write regions in Cosmos DB and configuring conflict resolution policies for globally distributed applications.”

---

# 43. What is Cosmos DB Partitioning?

Partitioning distributes data across multiple physical partitions for scalability and performance.

---

## Partition Key

A property used to distribute data evenly.

---

## Example Partition Keys

* UserID
* TenantID
* Region

---

## Benefits

* Horizontal scaling
* Improved throughput
* Balanced RU consumption

---

## Types of Partitions

| Type               | Description                     |
| ------------------ | ------------------------------- |
| Logical Partition  | Data sharing same partition key |
| Physical Partition | Underlying storage partition    |

---

## Best Practices

* Choose high-cardinality keys
* Avoid hot partitions
* Distribute traffic evenly

---

## Interview Answer

“Cosmos DB partitioning distributes data across partitions using a partition key to achieve scalability, performance, and balanced throughput.”

---

# 44. How do you secure Cosmos DB with VNet?

Cosmos DB can be secured using:

* Azure Private Link
* Firewall rules
* VNet integration

---

## Security Methods

| Method                 | Purpose                  |
| ---------------------- | ------------------------ |
| Private Endpoint       | Private IP access        |
| Firewall Rules         | Restrict IP access       |
| VNet Service Endpoints | Secure VNet connectivity |

---

## Steps

### 1. Disable Public Access (Optional)

---

### 2. Configure Private Endpoint

Connect Cosmos DB to:

* VNet/Subnet

---

### 3. Configure Private DNS

Ensure proper name resolution.

---

### 4. Restrict Firewall Rules

Allow only trusted sources.

---

## Benefits

* No internet exposure
* Secure private communication
* Compliance support

---

## Interview Answer

“Cosmos DB is secured with VNets using Private Endpoints, firewall restrictions, and private DNS integration to eliminate public exposure.”

---

# 45. What are Service Bus Topics and Subscriptions?

Azure Service Bus Topics and Subscriptions implement:

* Publish/Subscribe messaging pattern

---

# Topics

A Topic receives messages from publishers.

---

# Subscriptions

Subscriptions receive copies of messages from the topic.

---

## Example

### Topic

* Orders

### Subscriptions

* Billing Service
* Inventory Service
* Shipping Service

Each service receives the message independently.

---

## Benefits

* Decoupled applications
* One-to-many messaging
* Scalability

---

## Features

* Message filtering
* Dead-letter queues
* Durable messaging

---

## Interview Answer

“Service Bus Topics and Subscriptions implement publish-subscribe messaging where a topic distributes messages to multiple subscribers independently.”

---

# 46. Difference Between Event Grid and Event Hub

| Feature       | Event Grid          | Event Hub                   |
| ------------- | ------------------- | --------------------------- |
| Purpose       | Event routing       | Event streaming             |
| Data Volume   | Lightweight events  | Massive telemetry ingestion |
| Communication | Push model          | Pull model                  |
| Use Case      | Reactive automation | Real-time streaming         |
| Latency       | Very low            | Optimized for throughput    |

---

# Event Grid

Azure Event Grid routes lightweight events to subscribers.

---

## Best For

* Serverless triggers
* Workflow automation

---

# Event Hub

Azure Event Hubs handles large-scale event streaming.

---

## Best For

* IoT telemetry
* Real-time analytics

---

## Interview Answer

“Event Grid is used for lightweight event routing and automation, while Event Hub is designed for high-throughput event streaming and telemetry ingestion.”

---

# 47. How do you configure Event Hub Capture?

Event Hub Capture automatically stores streaming data into:

* Blob Storage
* Data Lake Storage

---

## Configuration Steps

### 1. Open Event Hub

---

### 2. Enable Capture

Select:

* Enable Capture

---

### 3. Choose Destination

* Storage Account
* Data Lake

---

### 4. Configure Capture Window

Example:

* Every 5 minutes
* Every 100 MB

---

## Benefits

* Long-term retention
* Analytics integration
* Stream archival

---

## Use Cases

* IoT telemetry archival
* Big data processing

---

## Interview Answer

“Event Hub Capture is configured by enabling capture and specifying Blob Storage or Data Lake as the destination for streaming event archival.”

---

# 48. What are Event Grid Domains?

Azure Event Grid Domains provide multi-tenant event management at scale.

---

## Purpose

Manage large numbers of:

* Topics
* Event publishers
* Subscribers

under one domain.

---

## Benefits

* Simplified event management
* Multi-tenant architecture
* Scalability

---

## Use Cases

* SaaS platforms
* Multi-customer event systems

---

## Example

Separate event topics per customer inside one Event Grid Domain.

---

## Interview Answer

“Event Grid Domains provide scalable multi-tenant event management by organizing multiple Event Grid topics under a single domain.”

---

# 49. What is Azure Monitor Autoscale?

Azure Monitor Autoscale automatically adjusts resource capacity based on metrics.

---

## Supported Resources

* VM Scale Sets
* App Services
* AKS node pools

---

## Scaling Types

| Type      | Purpose          |
| --------- | ---------------- |
| Scale Out | Add instances    |
| Scale In  | Remove instances |

---

## Common Metrics

* CPU usage
* Memory
* Queue length
* HTTP requests

---

## Example

Add 2 VM instances when:

* CPU > 70%

---

## Benefits

* Cost optimization
* High availability
* Automatic elasticity

---

## Interview Answer

“Azure Monitor Autoscale automatically increases or decreases resource instances based on performance metrics and predefined rules.”

---

# 50. How do you configure Application Insights Custom Telemetry?

Azure Application Insights custom telemetry tracks application-specific metrics and events.

---

## Types of Custom Telemetry

| Type           | Example        |
| -------------- | -------------- |
| Custom Events  | User actions   |
| Custom Metrics | Business KPIs  |
| Traces         | Debugging logs |
| Exceptions     | Error tracking |

---

## Configuration Steps

### 1. Integrate SDK

Install:

* Application Insights SDK

---

### 2. Instrument Application

Add telemetry calls in code.

---

## Example (.NET)

```csharp id="jhzx0t"
telemetryClient.TrackEvent("OrderPlaced");
```

---

### 3. Query Data

Use:

* Log Analytics
* KQL queries

---

## Benefits

* Business analytics
* Deep monitoring
* Custom observability

---

## Interview Answer

“Application Insights custom telemetry is configured by instrumenting applications with SDKs to capture custom events, metrics, traces, and exceptions.”

# 51. What is KQL in Log Analytics?

KQL (Kusto Query Language) is the query language used in:

* Azure Log Analytics
* Microsoft Sentinel
* Azure Monitor

to analyze and query large volumes of log and telemetry data.

---

## Features

* Fast querying
* Filtering
* Aggregation
* Visualization
* Time-series analysis

---

## Common Operations

| Operation | Purpose        |
| --------- | -------------- |
| where     | Filter records |
| summarize | Aggregate data |
| project   | Select columns |
| join      | Combine tables |
| order by  | Sort results   |

---

## Example Query

Find CPU usage logs:

```kql id="8kqz1r"
Perf
| where CounterName == "% Processor Time"
| summarize avg(CounterValue) by Computer
```

---

## Benefits

* Powerful analytics
* Security investigations
* Monitoring and troubleshooting

---

## Interview Answer

“KQL is the query language used in Azure Monitor, Log Analytics, and Sentinel to analyze logs, metrics, and telemetry data efficiently.”

---

# 52. How do you write a query to find Failed VM Deployments?

Failed VM deployments can be identified using:

* Azure Activity Logs
* KQL queries

---

## Example KQL Query

```kql id="0b4q3y"
AzureActivity
| where ResourceProvider == "Microsoft.Compute"
| where OperationNameValue contains "virtualMachines/write"
| where ActivityStatusValue == "Failure"
| project TimeGenerated, ResourceGroup, Resource, Caller, ActivityStatusValue
```

---

## Query Explanation

| Clause              | Purpose               |
| ------------------- | --------------------- |
| AzureActivity       | Activity log table    |
| ResourceProvider    | Filter VM operations  |
| ActivityStatusValue | Filter failures       |
| project             | Show important fields |

---

## Use Cases

* Deployment troubleshooting
* Audit analysis
* Operational monitoring

---

## Interview Answer

“Failed VM deployments can be identified using KQL queries against AzureActivity logs by filtering failed Microsoft.Compute VM write operations.”

---

# 53. What is Azure Sentinel Fusion Detection?

Microsoft Sentinel Fusion Detection uses machine learning and correlation analytics to detect advanced multi-stage attacks.

---

## Purpose

Correlate low-confidence alerts into high-confidence incidents.

---

## Features

* AI-driven correlation
* Multi-stage attack detection
* Threat intelligence integration
* Reduced false positives

---

## Example

Fusion may combine:

* Suspicious login
* Malware alert
* Data exfiltration event

into a single security incident.

---

## Benefits

* Faster threat detection
* Reduced analyst workload
* Improved SOC efficiency

---

## Interview Answer

“Sentinel Fusion Detection uses machine learning to correlate multiple low-level alerts into high-confidence security incidents.”

---

# 54. How do you integrate Sentinel with Logic Apps?

Azure Logic Apps integrates with Sentinel for automated incident response (SOAR).

---

## Integration Purpose

Automate:

* Incident handling
* Notifications
* Remediation workflows

---

## Steps

### 1. Create Logic App

---

### 2. Add Sentinel Trigger

Examples:

* Incident created
* Alert triggered

---

### 3. Add Automated Actions

Examples:

* Send Teams alert
* Create ServiceNow ticket
* Disable user account

---

### 4. Attach Playbook to Analytics Rule

Sentinel executes Logic App automatically.

---

## Benefits

* Automated response
* Reduced manual effort
* Faster incident remediation

---

## Interview Answer

“Sentinel integrates with Logic Apps using playbooks to automate incident response workflows such as notifications, ticket creation, and remediation.”

---

# 55. How do you configure Sentinel Workbooks?

Sentinel Workbooks provide interactive dashboards and visualizations.

---

## Features

* Visual reporting
* KQL integration
* Security dashboards
* Custom analytics views

---

## Configuration Steps

### 1. Open Sentinel

Navigate to:

* Workbooks

---

### 2. Create Workbook

Use:

* Templates
* Blank workbook

---

### 3. Add Data Sources

Examples:

* Log Analytics
* Security incidents
* Azure Activity Logs

---

### 4. Add Visualizations

* Charts
* Tables
* Maps
* Metrics

---

## Use Cases

* SOC dashboards
* Threat analytics
* Compliance reporting

---

## Interview Answer

“Sentinel Workbooks are configured using KQL-based visual dashboards to monitor security incidents, analytics, and operational insights.”

---

# 56. How do you configure Alert Suppression in Monitor?

Alert suppression prevents repeated alerts for the same issue.

---

## Methods

| Method                 | Purpose             |
| ---------------------- | ------------------- |
| Alert Processing Rules | Suppress alerts     |
| Action Rules           | Pause notifications |
| Dynamic Thresholds     | Reduce alert noise  |

---

## Steps

### 1. Open Azure Monitor

---

### 2. Create Alert Processing Rule

---

### 3. Define Scope

Examples:

* Resource Group
* Subscription

---

### 4. Configure Suppression

Suppress alerts:

* During maintenance windows
* For repeated conditions

---

## Benefits

* Reduced alert fatigue
* Cleaner operations
* Better incident management

---

## Interview Answer

“Alert suppression in Azure Monitor is configured using alert processing rules to reduce duplicate or unnecessary notifications.”

---

# 57. What is Azure Policy Remediation?

Azure Policy remediation automatically fixes non-compliant Azure resources.

---

## Purpose

Enforce governance automatically.

---

## Example

If a VM lacks required tags:

* Policy automatically adds tags

---

## Supported Effects

| Effect            | Purpose                    |
| ----------------- | -------------------------- |
| DeployIfNotExists | Deploy missing config      |
| Modify            | Modify resource properties |

---

## Steps

1. Assign policy
2. Identify non-compliance
3. Trigger remediation task

---

## Benefits

* Automated compliance
* Reduced manual work
* Governance consistency

---

## Interview Answer

“Azure Policy remediation automatically corrects non-compliant resources using remediation tasks and policy effects like DeployIfNotExists or Modify.”

---

# 58. How do you configure Blueprints for Governance?

Azure Blueprints standardize Azure environments using predefined governance artifacts.

---

## Blueprint Components

| Component        | Purpose                   |
| ---------------- | ------------------------- |
| ARM Templates    | Infrastructure deployment |
| Policies         | Governance enforcement    |
| RBAC Assignments | Access control            |
| Resource Groups  | Resource organization     |

---

## Configuration Steps

### 1. Create Blueprint Definition

---

### 2. Add Artifacts

Examples:

* Policies
* Templates
* Role assignments

---

### 3. Publish Blueprint

---

### 4. Assign Blueprint

Assign to:

* Subscription
* Management Group

---

## Benefits

* Consistent environments
* Faster provisioning
* Governance automation

---

## Interview Answer

“Blueprints are configured by combining ARM templates, policies, RBAC assignments, and resource groups into reusable governance templates.”

---

# 59. What is Resource Lock in Azure?

Azure Resource Locks prevent accidental modification or deletion of Azure resources.

---

## Lock Types

| Lock Type    | Effect                            |
| ------------ | --------------------------------- |
| CanNotDelete | Prevent deletion                  |
| ReadOnly     | Prevent modification and deletion |

---

## Scope

Locks can be applied at:

* Subscription
* Resource Group
* Resource level

---

## Example

Protect production database from accidental deletion.

---

## Benefits

* Prevent operational mistakes
* Improve governance
* Protect critical resources

---

## Interview Answer

“Azure Resource Locks protect resources from accidental deletion or modification using ReadOnly and CanNotDelete lock types.”

---

# 60. How do you implement Management Group Hierarchy?

Management Groups organize Azure subscriptions hierarchically for centralized governance.

---

## Typical Hierarchy

```text id="u2c9fy"
Tenant Root Group
│
├── Production Management Group
│   ├── Prod Subscription 1
│   └── Prod Subscription 2
│
├── Development Management Group
│   ├── Dev Subscription 1
│   └── Test Subscription 1
│
└── Shared Services Management Group
```

---

## Implementation Steps

### 1. Create Management Groups

Examples:

* Production
* Development
* Security

---

### 2. Move Subscriptions

Assign subscriptions to appropriate groups.

---

### 3. Apply Governance

Configure:

* RBAC
* Azure Policies
* Blueprints

at management group level.

---

### 4. Use Inheritance

Policies and permissions inherit downward.

---

## Benefits

* Centralized governance
* Simplified administration
* Enterprise-scale management

---

## Best Practices

* Separate prod/non-prod
* Centralize security policies
* Use least privilege RBAC

---

## Interview Answer

“Management Group hierarchy is implemented by organizing subscriptions into logical groups and applying centralized policies, RBAC, and governance controls with inheritance.”

# 61. How do you enforce Compliance with Policy?

Azure Policy enforces organizational standards and compliance across Azure resources.

---

## Methods to Enforce Compliance

| Method            | Purpose                       |
| ----------------- | ----------------------------- |
| Deny Policies     | Block non-compliant resources |
| Audit Policies    | Detect violations             |
| Modify Policies   | Auto-correct configurations   |
| DeployIfNotExists | Deploy required settings      |

---

## Common Compliance Policies

* Restrict allowed regions
* Enforce resource tagging
* Require encryption
* Restrict public IP creation

---

## Implementation Steps

### 1. Define Policy

Use:

* Built-in policy
* Custom policy

---

### 2. Assign Policy

Apply at:

* Management Group
* Subscription
* Resource Group

---

### 3. Monitor Compliance

Review:

* Compliance dashboard
* Non-compliant resources

---

### 4. Configure Remediation

Automatically fix violations.

---

## Benefits

* Governance automation
* Security enforcement
* Regulatory compliance

---

## Interview Answer

“Compliance is enforced using Azure Policy by assigning governance rules that audit, deny, or automatically remediate non-compliant resources.”

---

# 62. What is Service Principal Lifecycle Management?

A Service Principal is an identity used by applications and automation tools to access Azure resources securely.

---

## Lifecycle Stages

| Stage                 | Description                   |
| --------------------- | ----------------------------- |
| Creation              | Register application identity |
| Assignment            | Assign RBAC permissions       |
| Credential Management | Rotate secrets/certificates   |
| Monitoring            | Audit usage                   |
| Decommissioning       | Remove unused principals      |

---

## Best Practices

* Use Managed Identities where possible
* Rotate credentials regularly
* Apply least privilege access
* Monitor sign-in activity

---

## Example

Terraform pipeline uses:

* Service Principal
  with:
* Contributor access to subscription

---

## Interview Answer

“Service principal lifecycle management includes secure creation, permission assignment, credential rotation, monitoring, and decommissioning of application identities.”

---

# 63. How do you configure Key Vault Integration with DevOps?

Azure Key Vault integrates with DevOps pipelines to securely store secrets.

---

## Use Cases

* Store API keys
* Store passwords
* Store certificates
* Secure pipeline secrets

---

## Integration Steps

### 1. Create Key Vault

Add secrets/certificates.

---

### 2. Configure Access

Grant pipeline/service principal access.

---

### 3. Create Service Connection

In:

* [Azure DevOps](https://azure.microsoft.com/products/devops?utm_source=chatgpt.com)

---

### 4. Link Key Vault to Pipeline

Use:

* Variable groups
* Key Vault task

---

## Example YAML

```yaml id="7e4frt"
variables:
- group: prod-keyvault-secrets
```

---

## Benefits

* Secure secret management
* No hardcoded credentials
* Centralized control

---

## Interview Answer

“Key Vault integrates with DevOps pipelines through service connections and variable groups to securely retrieve secrets during deployments.”

---

# 64. How do you configure Azure Pipelines with Approvals?

Approvals add manual validation before deployments proceed.

---

## Common Approval Scenarios

* Production deployment approval
* Security approval
* CAB approval

---

## Configuration Steps

### 1. Create Environment

Example:

* Production

---

### 2. Configure Approvals

Add:

* Approvers
* Conditions

---

### 3. Add Deployment Stage

Pipeline pauses until approval is granted.

---

## Benefits

* Controlled deployments
* Governance enforcement
* Reduced deployment risk

---

## Example Workflow

Dev → QA → Approval → Production

---

## Interview Answer

“Azure Pipelines approvals are configured using environments and approval gates to require manual validation before deployment stages execute.”

---

# 65. What are YAML Pipeline Advantages?

YAML pipelines define CI/CD workflows as code.

---

## Advantages

| Advantage        | Benefit                      |
| ---------------- | ---------------------------- |
| Pipeline as Code | Version-controlled pipelines |
| Reusability      | Easier template reuse        |
| Automation       | Fully automated CI/CD        |
| Portability      | Easy migration               |
| Transparency     | Human-readable configuration |

---

## Features

* Multi-stage pipelines
* Conditional execution
* Template support
* Git integration

---

## Example

```yaml id="c84m1x"
trigger:
- main
```

---

## Benefits Over Classic Pipelines

* Easier maintenance
* Better collaboration
* Git-based change tracking

---

## Interview Answer

“YAML pipelines provide CI/CD as code with version control, reusability, automation, and easier maintenance compared to classic GUI pipelines.”

---

# 66. How do you configure Pipeline Templates?

Pipeline templates promote reusable YAML configurations.

---

## Types of Templates

| Template Type  | Purpose      |
| -------------- | ------------ |
| Stage Template | Reuse stages |
| Job Template   | Reuse jobs   |
| Step Template  | Reuse tasks  |

---

## Example Structure

```yaml id="1m4t7w"
# template.yml
steps:
- script: echo "Build Started"
```

```yaml id="wq5k9p"
# azure-pipelines.yml
steps:
- template: template.yml
```

---

## Benefits

* Standardization
* Reduced duplication
* Easier maintenance

---

## Use Cases

* Common build steps
* Security scans
* Shared deployment logic

---

## Interview Answer

“Pipeline templates are reusable YAML components used to standardize and simplify CI/CD pipeline configurations.”

---

# 67. How do you integrate Terraform in DevOps Pipelines?

[Terraform](https://www.terraform.io/?utm_source=chatgpt.com) integrates with CI/CD pipelines for Infrastructure as Code automation.

---

## Pipeline Workflow

| Stage    | Purpose                 |
| -------- | ----------------------- |
| Init     | Initialize Terraform    |
| Validate | Validate configuration  |
| Plan     | Generate execution plan |
| Apply    | Deploy infrastructure   |

---

## Integration Steps

### 1. Install Terraform Agent

---

### 2. Configure Service Connection

Use:

* Service Principal

---

### 3. Store State Securely

Use:

* Azure Blob Storage backend

---

### 4. Add Pipeline Tasks

Example:

```yaml id="x5bn0d"
- script: terraform init
- script: terraform plan
- script: terraform apply -auto-approve
```

---

## Best Practices

* Use remote state locking
* Separate environments
* Use approval gates

---

## Interview Answer

“Terraform integrates with DevOps pipelines by automating init, plan, and apply stages using service principals and remote state storage.”

---

# 68. How do you integrate GitHub Actions with Azure Resources?

[GitHub Actions](https://github.com/features/actions?utm_source=chatgpt.com) integrates with Azure for automated deployments.

---

## Integration Steps

### 1. Create Azure Service Principal

---

### 2. Store Credentials in GitHub Secrets

Examples:

* AZURE_CLIENT_ID
* AZURE_SECRET

---

### 3. Configure Workflow YAML

Example:

```yaml id="q1t7jf"
- uses: azure/login@v1
```

---

### 4. Deploy Resources

Examples:

* App Service
* AKS
* Terraform
* ARM/Bicep

---

## Benefits

* Automated CI/CD
* GitHub-native workflows
* Multi-cloud automation

---

## Use Cases

* App deployments
* Infrastructure automation
* Kubernetes deployments

---

## Interview Answer

“GitHub Actions integrates with Azure using service principals and workflow YAML files to automate deployments and infrastructure provisioning.”

---

# 69. How do you configure Azure Arc-enabled Kubernetes?

Azure Arc extends Azure management to Kubernetes clusters outside Azure.

---

## Supported Environments

* On-prem Kubernetes
* AWS EKS
* Google GKE

---

## Configuration Steps

### 1. Install Azure CLI Extensions

---

### 2. Connect Cluster

Use:

```bash id="f3q7nt"
az connectedk8s connect
```

---

### 3. Register Cluster

Cluster appears in Azure Arc.

---

### 4. Apply Governance

Enable:

* Azure Policy
* Monitoring
* Defender for Cloud

---

## Benefits

* Centralized governance
* Multi-cloud visibility
* Unified management

---

## Interview Answer

“Azure Arc-enabled Kubernetes is configured by connecting external Kubernetes clusters to Azure for centralized governance, monitoring, and policy management.”

---

# 70. What is Lighthouse Multi-Tenant Scenario?

Azure Lighthouse enables centralized management across multiple Azure tenants.

---

## Multi-Tenant Scenario Example

### Managed Service Provider (MSP)

Manages:

* Multiple customer Azure tenants

from:

* One centralized portal

---

## Features

* Delegated resource management
* Cross-tenant RBAC
* Centralized operations
* Secure tenant isolation

---

## Benefits

* Simplified management
* Operational efficiency
* Secure delegated access

---

## Common Use Cases

* MSP operations
* Enterprise subsidiaries
* Multi-tenant governance

---

## Interview Answer

“Azure Lighthouse enables secure centralized management of multiple Azure tenants using delegated access and cross-tenant RBAC.”

# 71. How do you configure Monitoring for AKS?

Azure Kubernetes Service monitoring is commonly implemented using:

* Azure Monitor
* Azure Container Insights
* Azure Log Analytics

---

## Monitoring Components

| Component            | Purpose               |
| -------------------- | --------------------- |
| Container Insights   | Cluster monitoring    |
| Log Analytics        | Centralized logging   |
| Prometheus/Grafana   | Metrics visualization |
| Application Insights | App telemetry         |

---

## Configuration Steps

### 1. Enable Monitoring During AKS Creation

Enable:

* Azure Monitor Container Insights

---

### 2. Connect Log Analytics Workspace

---

### 3. Enable Metrics Collection

Collect:

* CPU
* Memory
* Node metrics
* Pod metrics

---

### 4. Configure Alerts

Examples:

* High CPU
* Pod failures
* Node pressure

---

## Benefits

* Cluster visibility
* Performance troubleshooting
* Proactive alerting

---

## Interview Answer

“AKS monitoring is configured using Azure Monitor, Container Insights, and Log Analytics to collect cluster metrics, logs, and alerts.”

---

# 72. How do you implement Network Policies in AKS?

Network Policies control pod-to-pod communication inside AKS clusters.

---

## Supported Providers

| Provider             | Description       |
| -------------------- | ----------------- |
| Azure Network Policy | Native Azure      |
| Calico               | Advanced policies |

---

## Purpose

Restrict communication between:

* Pods
* Namespaces

---

## Example Policy

Allow:

* Frontend pods → Backend pods
  Block:
* Unauthorized pod access

---

## Steps

### 1. Enable Network Policy During Cluster Creation

---

### 2. Create YAML Policy

Example:

```yaml id="v2k8ms"
policyTypes:
- Ingress
```

---

### 3. Apply Policy

```bash id="h7s2ac"
kubectl apply -f policy.yaml
```

---

## Benefits

* Micro-segmentation
* Zero Trust networking
* Improved cluster security

---

## Interview Answer

“AKS network policies are implemented using Azure or Calico policies to control and restrict pod communication within the Kubernetes cluster.”

---

# 73. How do you integrate Key Vault with AKS CSI Driver?

Azure Key Vault integrates with AKS using the:

* Secrets Store CSI Driver

---

## Purpose

Mount secrets directly into pods securely.

---

## Components

| Component        | Purpose            |
| ---------------- | ------------------ |
| CSI Driver       | Secret integration |
| Managed Identity | Authentication     |
| Key Vault        | Secret storage     |

---

## Steps

### 1. Enable CSI Driver Add-on

During AKS setup.

---

### 2. Create Managed Identity

Grant:

* Key Vault access

---

### 3. Configure SecretProviderClass

Define:

* Secrets
* Certificates

---

### 4. Mount Secrets in Pods

---

## Benefits

* No hardcoded secrets
* Centralized secret management
* Automatic secret rotation

---

## Interview Answer

“Key Vault integrates with AKS using the CSI driver and managed identities to securely mount secrets and certificates directly into Kubernetes pods.”

---

# 74. What is Service Mesh in AKS?

A Service Mesh provides advanced traffic management, security, and observability for microservices communication.

---

## Common Service Meshes

| Mesh                    | Platform                 |
| ----------------------- | ------------------------ |
| Istio                   | Popular open-source mesh |
| Linkerd                 | Lightweight service mesh |
| Open Service Mesh (OSM) | Azure-supported          |

---

## Features

* mTLS encryption
* Traffic routing
* Load balancing
* Observability
* Retry policies

---

## Benefits

* Secure service communication
* Advanced traffic control
* Improved observability

---

## Example

Canary deployment:

* Route 10% traffic to new version

---

## Interview Answer

“A service mesh in AKS provides secure and observable service-to-service communication with advanced traffic management capabilities.”

---

# 75. How do you secure AKS Nodes with PIM?

Microsoft Entra Privileged Identity Management secures privileged access to AKS infrastructure.

---

## Security Approach

| Control           | Purpose                |
| ----------------- | ---------------------- |
| JIT Access        | Temporary admin access |
| MFA               | Strong authentication  |
| Approval Workflow | Controlled elevation   |
| RBAC              | Least privilege        |

---

## Steps

### 1. Enable PIM

---

### 2. Configure Eligible Roles

Examples:

* AKS Cluster Admin
* Contributor

---

### 3. Require MFA and Approval

---

### 4. Audit Privileged Access

---

## Benefits

* Reduced attack surface
* Better compliance
* Controlled administrative access

---

## Interview Answer

“AKS nodes are secured with PIM by enabling just-in-time privileged access, MFA, approval workflows, and RBAC-based least privilege access.”

---

# 76. How do you integrate Defender for Cloud with AKS?

Microsoft Defender for Cloud provides security monitoring and threat protection for AKS.

---

## Features

* Vulnerability assessment
* Runtime threat detection
* Kubernetes posture management
* Container image scanning

---

## Configuration Steps

### 1. Enable Defender for Containers

---

### 2. Connect AKS Cluster

---

### 3. Enable Monitoring Components

* Log Analytics
* Defender sensors

---

### 4. Review Recommendations

Examples:

* Privileged containers
* Exposed dashboards

---

## Benefits

* Container security
* Threat detection
* Compliance visibility

---

## Interview Answer

“Defender for Cloud integrates with AKS to provide runtime protection, vulnerability scanning, and Kubernetes security posture management.”

---

# 77. How do you troubleshoot Node Image Upgrade in AKS?

Node image upgrades update:

* OS image
* Security patches
* Kubernetes node components

---

## Common Troubleshooting Steps

| Step                  | Purpose                  |
| --------------------- | ------------------------ |
| Check Node Status     | Verify node health       |
| Review Upgrade Events | Identify failures        |
| Inspect Pods          | Detect scheduling issues |
| Verify Capacity       | Ensure enough resources  |
| Check PDBs            | Pod disruption policies  |

---

## Useful Commands

### Check Node Status

```bash id="5z8nqp"
kubectl get nodes
```

### Check Events

```bash id="c3tx6r"
kubectl get events
```

---

## Common Issues

* Insufficient resources
* Pod eviction failures
* PDB restrictions
* Network plugin problems

---

## Best Practices

* Use surge upgrades
* Test in staging first
* Maintain multiple node pools

---

## Interview Answer

“AKS node image upgrade troubleshooting involves checking node health, events, pod disruption policies, and cluster capacity during rolling node updates.”

---

# 78. What is Azure Private DNS Zone?

Azure Private DNS provides private DNS name resolution inside Azure VNets.

---

## Purpose

Resolve private IP addresses internally.

---

## Features

* VNet-integrated DNS
* Automatic name resolution
* Hybrid DNS support

---

## Example

Resolve:

```text id="e0f4wd"
sqlprod.privatelink.database.windows.net
```

to:

* Private IP address

---

## Benefits

* Secure internal DNS
* Simplified private networking
* Hybrid integration

---

## Interview Answer

“Azure Private DNS Zone provides private internal DNS resolution for Azure VNets and Private Endpoints.”

---

# 79. How do you configure Conditional Forwarding in Private DNS?

Conditional forwarding directs DNS queries for specific domains to designated DNS servers.

---

## Common Use Case

Forward:

* On-prem domain queries
  to:
* On-prem DNS servers

---

## Steps

### 1. Deploy DNS Forwarder VM

(Optional)

---

### 2. Configure Conditional Forwarder

Example:

* contoso.local → On-prem DNS

---

### 3. Configure VNet DNS Settings

Point VNets to:

* Custom DNS servers

---

## Benefits

* Hybrid DNS resolution
* Seamless name lookup
* Cross-environment integration

---

## Interview Answer

“Conditional forwarding in Azure Private DNS routes specific domain queries to custom DNS servers for hybrid name resolution.”

---

# 80. How do you implement Hybrid DNS Resolution?

Hybrid DNS resolution enables name resolution between:

* Azure
* On-premises environments

---

## Common Architecture

| Component          | Purpose             |
| ------------------ | ------------------- |
| Azure Private DNS  | Azure internal DNS  |
| DNS Forwarder VM   | Forward DNS queries |
| VPN/ExpressRoute   | Connectivity        |
| On-prem DNS Server | On-prem resolution  |

---

## Steps

### 1. Configure Network Connectivity

Use:

* VPN Gateway
* ExpressRoute

---

### 2. Deploy DNS Forwarder

In Azure hub VNet.

---

### 3. Configure Conditional Forwarders

Forward:

* Azure domains
* On-prem domains

---

### 4. Link Private DNS Zones

Attach VNets to private zones.

---

## Benefits

* Unified DNS experience
* Hybrid application support
* Seamless resource communication

---

## Interview Answer

“Hybrid DNS resolution is implemented using Azure Private DNS, DNS forwarders, and VPN or ExpressRoute connectivity to enable seamless name resolution between Azure and on-premises environments.”

# 81. How do you configure ExpressRoute FastPath?

Azure ExpressRoute FastPath improves network performance by bypassing the ExpressRoute Gateway for data traffic.

---

## Purpose

Reduce:

* Latency
* Gateway bottlenecks
* CPU overhead

---

## Benefits

* Higher throughput
* Lower latency
* Better performance for hybrid workloads

---

## Requirements

| Requirement          | Description                  |
| -------------------- | ---------------------------- |
| ExpressRoute Gateway | Ultra Performance or ErGw3AZ |
| Private Peering      | Required                     |
| Supported Regions    | Gateway-supported regions    |

---

## Configuration Steps

### 1. Create ExpressRoute Circuit

---

### 2. Deploy Supported Gateway SKU

---

### 3. Enable FastPath

During VNet gateway configuration.

---

### 4. Validate Connectivity

Check route propagation and performance.

---

## Use Cases

* SAP workloads
* High-throughput hybrid applications
* Large enterprise environments

---

## Interview Answer

“ExpressRoute FastPath improves hybrid network performance by bypassing the virtual network gateway for data traffic and reducing latency.”

---

# 82. How do you troubleshoot VPN Gateway Disconnections?

Azure VPN Gateway disconnections are commonly caused by network, routing, or configuration issues.

---

## Troubleshooting Steps

| Step                   | Purpose                 |
| ---------------------- | ----------------------- |
| Check Gateway Health   | Verify VPN status       |
| Validate Tunnel Status | Confirm IPSec tunnel    |
| Check NSGs/Firewall    | Ensure ports are open   |
| Verify Shared Keys     | Match both sides        |
| Review BGP Routes      | Validate route exchange |
| Analyze Logs           | Diagnose failures       |

---

## Important Ports

| Protocol | Port     |
| -------- | -------- |
| IKE      | UDP 500  |
| NAT-T    | UDP 4500 |

---

## Useful Tools

* Azure Network Watcher
* Connection Troubleshoot
* VPN diagnostics logs

---

## Common Causes

* ISP instability
* Expired shared keys
* Route conflicts
* Firewall blocking

---

## Interview Answer

“VPN Gateway disconnections are troubleshot by checking tunnel health, BGP routes, shared keys, NSGs, firewall rules, and diagnostic logs.”

---

# 83. How do you implement High Availability in ExpressRoute?

High availability ensures resilient hybrid connectivity.

---

## HA Best Practices

| Practice                        | Purpose            |
| ------------------------------- | ------------------ |
| Dual ExpressRoute Circuits      | Redundancy         |
| Multiple Connectivity Providers | Carrier resiliency |
| Zone-Redundant Gateway          | Gateway HA         |
| BGP Routing                     | Automatic failover |

---

## Recommended Architecture

Primary Circuit:

* Provider A

Secondary Circuit:

* Provider B

---

## Components Used

* ExpressRoute Gateway
* BGP
* Redundant edge routers

---

## Benefits

* Reduced downtime
* Automatic failover
* Enterprise resiliency

---

## Interview Answer

“High availability in ExpressRoute is implemented using redundant circuits, multiple providers, BGP routing, and zone-redundant gateways.”

---

# 84. What is Multi-VNet Hub-Spoke Design?

Hub-spoke is a centralized Azure network topology.

---

## Architecture

```text id="p7v3sk"
Hub VNet
│
├── Shared Services
├── Firewall
├── VPN/ER Gateway
│
├── Spoke VNet - Prod
├── Spoke VNet - Dev
└── Spoke VNet - Test
```

---

## Hub VNet

Hosts shared services:

* Firewall
* DNS
* VPN Gateway
* Bastion

---

## Spoke VNets

Contain:

* Application workloads

---

## Benefits

* Centralized security
* Reduced complexity
* Better governance
* Network segmentation

---

## Use Cases

* Enterprise landing zones
* Multi-environment deployments

---

## Interview Answer

“Multi-VNet hub-spoke design centralizes shared services and security in a hub VNet while application workloads run in isolated spoke VNets.”

---

# 85. How do you configure Firewalls in Hub-Spoke?

Azure Firewall is commonly deployed in the hub VNet.

---

## Configuration Steps

### 1. Deploy Azure Firewall in Hub

---

### 2. Configure Firewall Policies

Examples:

* Application rules
* Network rules

---

### 3. Create UDRs

Route spoke traffic through firewall.

---

## Example Route

```text id="5zqy0u"
0.0.0.0/0 → Azure Firewall
```

---

### 4. Peer Spoke VNets

Enable:

* Allow forwarded traffic

---

## Benefits

* Centralized inspection
* Consistent security
* Simplified governance

---

## Interview Answer

“Firewalls in hub-spoke architecture are configured centrally in the hub VNet with UDRs directing spoke traffic through the firewall.”

---

# 86. What is Azure Application Gateway Autoscaling?

Azure Application Gateway autoscaling automatically adjusts gateway instances based on traffic load.

---

## Features

* Dynamic scaling
* Automatic capacity management
* High availability

---

## Benefits

* Handles traffic spikes
* Improves performance
* Reduces manual scaling

---

## Configuration Steps

### 1. Enable Autoscaling

During gateway setup.

---

### 2. Configure Capacity Limits

Set:

* Minimum instances
* Maximum instances

---

## Example

Scale:

* 2 → 10 instances
  during high traffic.

---

## Interview Answer

“Application Gateway autoscaling dynamically increases or decreases gateway instances based on traffic demand to maintain performance and availability.”

---

# 87. How do you configure WAF Custom Rules?

Azure Web Application Firewall custom rules provide advanced traffic filtering.

---

## Rule Types

| Rule Type        | Purpose                |
| ---------------- | ---------------------- |
| Match Rules      | Match request patterns |
| Rate Limit Rules | Prevent abuse          |

---

## Example Conditions

* Block IP ranges
* Block SQL injection attempts
* Restrict countries

---

## Configuration Steps

### 1. Open WAF Policy

---

### 2. Add Custom Rule

Define:

* Match conditions
* Priority
* Action

---

### 3. Associate WAF Policy

Attach to:

* Application Gateway
* Front Door

---

## Benefits

* Enhanced application security
* Granular filtering
* Protection against attacks

---

## Interview Answer

“WAF custom rules are configured using match conditions and actions to block, allow, or rate-limit malicious web traffic.”

---

# 88. How do you configure SSL Offloading in App Gateway?

SSL offloading decrypts HTTPS traffic at:

* Azure Application Gateway

instead of backend servers.

---

## Benefits

* Reduced backend CPU usage
* Centralized certificate management
* Improved performance

---

## Configuration Steps

### 1. Upload SSL Certificate

Usually:

* PFX certificate

---

### 2. Create HTTPS Listener

---

### 3. Configure Backend Pool

---

### 4. Configure HTTP Backend Communication

Gateway handles HTTPS termination.

---

## Flow

```text id="u8m0cr"
Client HTTPS → App Gateway → Backend HTTP
```

---

## Interview Answer

“SSL offloading in Application Gateway terminates HTTPS traffic at the gateway and forwards decrypted traffic to backend servers.”

---

# 89. How do you integrate Front Door with CDN?

Azure Front Door integrates with CDN capabilities for global content delivery and acceleration.

---

## Benefits

* Global caching
* Reduced latency
* Faster static content delivery
* Improved user experience

---

## Configuration Steps

### 1. Create Front Door Profile

---

### 2. Configure Frontend Domains

---

### 3. Enable CDN Caching Rules

Cache:

* Images
* CSS
* JavaScript

---

### 4. Configure Origin Groups

Backend applications.

---

## Features

* Dynamic site acceleration
* WAF integration
* Global load balancing

---

## Interview Answer

“Front Door integrates with CDN by caching static content at Microsoft edge locations to improve performance and reduce latency globally.”

---

# 90. What are Azure Traffic Manager Routing Methods?

Azure Traffic Manager routes user traffic using DNS-based routing policies.

---

## Routing Methods

| Method      | Purpose                    |
| ----------- | -------------------------- |
| Priority    | Failover routing           |
| Weighted    | Traffic distribution       |
| Performance | Lowest latency             |
| Geographic  | Region-based routing       |
| MultiValue  | Multiple healthy endpoints |
| Subnet      | Route based on client IP   |

---

# Priority Routing

Primary endpoint used until failure occurs.

---

# Weighted Routing

Traffic distributed by assigned weights.

---

# Performance Routing

Routes users to:

* Closest/lowest latency endpoint

---

# Geographic Routing

Routes based on:

* User geographic location

---

## Use Cases

* Global applications
* Disaster recovery
* Blue-green deployments

---

## Interview Answer

“Azure Traffic Manager supports routing methods such as Priority, Weighted, Performance, Geographic, MultiValue, and Subnet for global traffic distribution and failover.”

# 91. Difference Between Weighted, Priority, and Performance Routing

These are routing methods in:

* Azure Traffic Manager

used to distribute user traffic globally.

---

## Comparison Table

| Routing Method | Purpose              | Traffic Behavior           | Best Use Case          |
| -------------- | -------------------- | -------------------------- | ---------------------- |
| Weighted       | Traffic distribution | Based on assigned weights  | Load balancing/testing |
| Priority       | Failover             | Primary endpoint preferred | Disaster recovery      |
| Performance    | Lowest latency       | Routes to closest endpoint | Global applications    |

---

# Weighted Routing

Traffic is distributed according to configured weights.

---

## Example

| Endpoint      | Weight |
| ------------- | ------ |
| West India    | 80     |
| Central India | 20     |

80% traffic goes to:

* West India

---

## Best For

* Blue-green deployments
* Canary releases
* Load distribution

---

# Priority Routing

Traffic always goes to:

* Primary endpoint

Secondary endpoints activate only during failure.

---

## Best For

* Disaster recovery
* Active-passive architecture

---

# Performance Routing

Routes users to:

* Lowest latency endpoint

based on geographic proximity and network latency.

---

## Best For

* Global applications
* Improved user experience

---

## Interview Answer

“Weighted routing distributes traffic proportionally, Priority routing supports failover using primary and secondary endpoints, and Performance routing sends users to the lowest latency endpoint.”

---

# 92. How do you configure Failover with Traffic Manager?

Failover routing is implemented using:

* Priority routing method

in:

* Azure Traffic Manager

---

## Configuration Steps

### 1. Create Traffic Manager Profile

---

### 2. Select Routing Method

Choose:

* Priority

---

### 3. Add Endpoints

| Endpoint         | Priority |
| ---------------- | -------- |
| Primary Region   | 1        |
| Secondary Region | 2        |

---

### 4. Configure Health Probes

Traffic Manager continuously checks endpoint health.

---

### 5. Test Failover

Simulate primary outage.

---

## Failover Flow

```text id="r9x7kp"
Primary Healthy → Traffic to Primary
Primary Down → Traffic to Secondary
```

---

## Benefits

* Automatic DR failover
* Improved availability
* Global resiliency

---

## Interview Answer

“Traffic Manager failover is configured using Priority routing with health probes that automatically redirect traffic to secondary endpoints during outages.”

---

# 93. What is Azure Lighthouse Delegated Resource Model?

Azure Lighthouse uses delegated resource management to securely manage resources across Azure tenants.

---

## Purpose

Allow:

* MSPs
* Central IT teams

to manage customer/subscription resources securely.

---

## How It Works

Customer tenant delegates access to:

* Service provider tenant

using:

* RBAC permissions

---

## Features

* Cross-tenant management
* Granular delegated access
* Centralized operations
* Tenant isolation

---

## Benefits

* Simplified management
* Secure multi-tenant administration
* Operational efficiency

---

## Interview Answer

“Azure Lighthouse delegated resource model enables secure cross-tenant Azure resource management using delegated RBAC permissions.”

---

# 94. How do you configure Delegated Subscriptions?

Delegated subscriptions allow external tenants or administrators to manage Azure subscriptions securely.

---

## Configuration Steps

### 1. Define Delegation

Create:

* ARM template
  or
* Azure Marketplace offer

---

### 2. Specify Authorized Tenant

Define:

* Managing tenant ID

---

### 3. Assign RBAC Roles

Examples:

* Reader
* Contributor
* Managed Services Registration Assignment Delete Role

---

### 4. Deploy Delegation

Customer approves delegation.

---

## Benefits

* Cross-tenant management
* Granular permissions
* MSP enablement

---

## Use Cases

* Managed services
* Centralized IT administration

---

## Interview Answer

“Delegated subscriptions are configured through Azure Lighthouse by assigning cross-tenant RBAC permissions to authorized service providers or administrators.”

---

# 95. What is Azure Arc for Servers?

Azure Arc for Servers extends Azure management to:

* On-prem servers
* AWS EC2
* Google Cloud VMs

---

## Supported Servers

* Windows
* Linux

---

## Features

* Azure Policy
* Monitoring
* Defender for Cloud
* Update management

---

## Benefits

* Centralized governance
* Hybrid visibility
* Unified security management

---

## Example

Manage AWS Linux VMs directly from Azure Portal.

---

## Interview Answer

“Azure Arc for Servers enables Azure management, governance, and monitoring for non-Azure Windows and Linux servers.”

---

# 96. How do you onboard Arc-enabled Servers?

Onboarding connects external servers to:

* Azure Arc

---

## Steps

### 1. Register Resource Providers

---

### 2. Generate Onboarding Script

From Azure Portal.

---

### 3. Install Connected Machine Agent

On:

* Windows/Linux server

---

### 4. Execute Registration Script

Server registers with Azure Arc.

---

### 5. Verify Connection

Server appears in:

* Azure Arc resources

---

## Benefits

* Hybrid management
* Azure governance
* Monitoring integration

---

## Interview Answer

“Arc-enabled servers are onboarded by installing the Connected Machine Agent and registering servers with Azure Arc using onboarding scripts.”

---

# 97. How do you configure Azure Policy across Arc Servers?

Azure Policy can govern Arc-enabled servers similarly to Azure resources.

---

## Configuration Steps

### 1. Connect Servers to Azure Arc

---

### 2. Assign Policies

Examples:

* Enforce tagging
* Audit missing patches
* Validate configurations

---

### 3. Enable Guest Configuration

For OS-level compliance checks.

---

### 4. Monitor Compliance

Use:

* Policy compliance dashboard

---

## Benefits

* Hybrid governance
* Compliance visibility
* Consistent security standards

---

## Interview Answer

“Azure Policy is applied to Arc-enabled servers by assigning policies and enabling Guest Configuration for hybrid compliance management.”

---

# 98. What is Azure Sphere Security Model?

Azure Sphere provides end-to-end security for IoT devices.

---

## Security Components

| Component                     | Purpose                 |
| ----------------------------- | ----------------------- |
| Secured MCU                   | Hardware security       |
| Secure OS                     | Hardened Linux-based OS |
| Azure Sphere Security Service | Cloud-based protection  |

---

## Security Features

* Hardware root of trust
* Secure boot
* Certificate-based authentication
* Automatic updates

---

## Benefits

* IoT device protection
* Threat resistance
* Secure connectivity

---

## Use Cases

* Industrial IoT
* Smart devices
* Connected appliances

---

## Interview Answer

“Azure Sphere security model provides hardware, OS, and cloud-based security for IoT devices using secure boot, hardware root of trust, and continuous updates.”

---

# 99. How do you configure IoT Hub DPS?

Azure IoT Hub Device Provisioning Service automates secure IoT device enrollment.

---

## Purpose

Automatically provision devices to:

* IoT Hubs

---

## Configuration Steps

### 1. Create DPS Instance

---

### 2. Link IoT Hub

---

### 3. Configure Enrollment Groups

Using:

* X.509 certificates
  or
* TPM attestation

---

### 4. Configure Devices

Devices connect using DPS endpoint.

---

### 5. Automatic Assignment

DPS assigns devices to appropriate IoT Hub.

---

## Benefits

* Automated onboarding
* Scalable provisioning
* Secure device registration

---

## Interview Answer

“IoT Hub DPS automates secure IoT device provisioning using certificate or TPM-based enrollment and automatic IoT Hub assignment.”

---

# 100. How do you secure Device Twins in IoT?

Device Twins store device metadata and configuration in:

* Azure IoT Hub

---

## Security Best Practices

| Practice           | Purpose                      |
| ------------------ | ---------------------------- |
| RBAC               | Restrict management access   |
| SAS Tokens         | Secure device authentication |
| X.509 Certificates | Strong authentication        |
| Private Endpoints  | Private connectivity         |
| Encryption         | Protect data in transit      |

---

## Additional Controls

* Use DPS secure provisioning
* Rotate credentials regularly
* Enable monitoring and auditing

---

## Device Twin Components

| Component           | Purpose                       |
| ------------------- | ----------------------------- |
| Desired Properties  | Cloud-to-device configuration |
| Reported Properties | Device status reporting       |

---

## Interview Answer

“Device Twins are secured using RBAC, SAS tokens or certificates, Private Endpoints, encryption, and controlled access policies within Azure IoT Hub.”

