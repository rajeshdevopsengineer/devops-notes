# CTS (Cognizant) Azure DevOps Interview Answers

**Company:** CTS (Cognizant)  
**Focus:** Azure networking, Terraform remote state, private VM access, Application Gateway, and application troubleshooting

> The answers are interview-ready. For troubleshooting questions, use a layer-by-layer approach: identify the failing hop, gather evidence, isolate the component, remediate safely, and verify recovery.

---

## 1. What is connection draining?

Connection draining is the process of stopping **new connections** from reaching a backend instance while allowing its **existing active connections** to complete for a configured period.

It is commonly used during:

- Rolling deployments
- VM or instance maintenance
- Autoscaling scale-in
- Backend deregistration
- Application shutdown
- Load balancer configuration changes

### Request flow

```text
Backend marked for removal
        |
Load balancer stops sending new requests
        |
Existing connections continue until completion or timeout
        |
Backend is safely removed or stopped
```

Connection draining reduces interrupted user requests and errors during planned changes. The application should also implement graceful shutdown so it stops accepting new work, completes in-flight requests, closes resources, and exits before the termination deadline.

### In Azure Application Gateway

Azure Application Gateway supports connection draining in backend settings. When enabled, an affected backend is given a configured interval to finish existing connections after removal or a relevant backend-setting change. Long-lived connections may still be closed when the configured timeout expires.

### Kubernetes equivalent

For a Kubernetes Pod, a similar graceful-removal flow involves:

1. The Pod becomes unready and is removed from Service endpoints.
2. A `preStop` hook can initiate graceful shutdown.
3. `terminationGracePeriodSeconds` allows in-flight work to finish.
4. The ingress/load balancer must receive and act on the endpoint update.

```yaml
spec:
  terminationGracePeriodSeconds: 60
  containers:
    - name: web
      lifecycle:
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 10"]
```

---

## 2. `backend.tf` is visible in the repository, but not in the Azure Storage Account. What may be the issue?

The key clarification is that **`backend.tf` itself is not uploaded to Azure Storage**. It is a Terraform configuration file stored in the source repository. It tells Terraform where to keep the **state blob**, normally a file such as `terraform.tfstate`, inside an Azure Blob container.

The AzureRM backend stores state as a blob named by the configured `key`. It supports Azure Blob native state locking and consistency checks. citeturn10search221turn10search223

### Example `backend.tf`

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-tfstate-prod"
    storage_account_name = "sttfstateprod001"
    container_name       = "tfstate"
    key                  = "network/prod.tfstate"
    use_azuread_auth     = true
  }
}
```

Expected storage location:

```text
Storage account: sttfstateprod001
Container:       tfstate
Blob key:        network/prod.tfstate
```

You should not expect to see `backend.tf` in the container.

### Reasons why the state blob may not appear

1. **Only the code was committed**
   - Committing `backend.tf` does not create the storage blob.
   - Run `terraform init`, followed by an operation that writes state, such as `terraform apply` or state migration.

2. **Terraform was not reinitialized after changing the backend**

```bash
terraform init -reconfigure
```

If migrating existing local state:

```bash
terraform init -migrate-state
```

`terraform init` initializes the configured backend and should be run after cloning a configuration or changing backend settings. citeturn10search226

3. **Wrong subscription, tenant, resource group, storage account, container, or blob key**

```bash
az account show
az storage blob list \
  --account-name sttfstateprod001 \
  --container-name tfstate \
  --auth-mode login \
  --output table
```

4. **Authentication or RBAC failure**
   - The pipeline identity needs storage data-plane access.
   - For Microsoft Entra authentication, assign an appropriate Blob Data role at the narrowest practical scope.
   - The AzureRM backend recommends Microsoft Entra authentication and supports workload identity federation, managed identities, service principals, and Azure CLI user authentication. citeturn10search221

5. **Storage firewall, private endpoint, or DNS issue**
   - The build agent may not resolve or reach the Blob endpoint.
   - A Microsoft-hosted agent normally cannot enter a private VNet unless the architecture provides suitable connectivity. A self-hosted agent in the connected network may be required.

6. **The backend block is not loaded**
   - Terraform loads `.tf` files in the current root module directory.
   - The pipeline may be executing from the wrong working directory.
   - The file may have an unsupported extension, be excluded from checkout, or reside in a child module.

7. **The backend configuration was passed elsewhere**
   - A pipeline may supply backend settings with `-backend-config` or use a generated backend configuration.
   - Confirm the initialization logs and `.terraform/terraform.tfstate` backend metadata without exposing credentials.

8. **State is still local or stored in another backend/workspace**

```bash
find . -name 'terraform.tfstate*' -o -name '.terraform'
terraform workspace show
terraform state pull > state-check.json
```

Remote backends define where state is stored. Terraform uses a local backend by default when no remote backend is configured. citeturn10search222turn10search225

### Recommended troubleshooting sequence

```bash
terraform version
terraform init -reconfigure
terraform validate
terraform providers
terraform workspace show
terraform state pull
```

Then verify the storage account, container, key, identity, data-plane role, firewall rules, private DNS, and pipeline working directory.

---

## 3. How do you log in to a VM that has only a private IP?

A VM with only a private IP must be accessed through a trusted private path. Do not add a public IP merely for routine administration.

### Option 1: Azure Bastion

Azure Bastion provides browser-based or native-client SSH/RDP access through a managed service. The target VM does not require a public IP. Azure guidance recommends Bastion as one of the primary secure remote-access options. citeturn10search229turn10search232

```bash
az network bastion ssh \
  --name bastion-prod \
  --resource-group rg-network-prod \
  --target-resource-id <vm-resource-id> \
  --auth-type ssh-key \
  --username azureuser \
  --ssh-key ~/.ssh/id_rsa
```

Azure Bastion can also reach targets by IP over connected networks in supported SKUs and topologies. citeturn10search229

### Option 2: Point-to-Site or Site-to-Site VPN

Connect the administrator workstation or on-premises network to the Azure VNet, then access the VM's private IP:

```bash
ssh azureuser@10.20.1.10
```

### Option 3: ExpressRoute

Use private enterprise connectivity from the corporate network to Azure. This is suitable for established hybrid environments.

### Option 4: Jump host

Connect to a hardened jump VM that can reach the target private VM. Avoid public jump hosts where Bastion or private connectivity is available. If used, enforce MFA/JIT, restricted NSGs, logging, patching, and no shared credentials.

```bash
ssh -J admin@jump-host azureuser@10.20.1.10
```

### Option 5: Azure VM Run Command or Serial Console

Use these for controlled administration or recovery when normal SSH/RDP connectivity is unavailable. They are not a replacement for a complete access architecture.

### Security checks

- Restrict SSH/RDP with NSGs.
- Use Microsoft Entra login where supported.
- Use Defender for Cloud Just-in-Time access where appropriate.
- Prefer SSH keys over passwords.
- Enable activity, Bastion, OS authentication, and session logging according to policy.
- Verify UDRs, forced tunneling, NVA routing, DNS, and NSGs if Bastion cannot connect. These can disrupt the Bastion control or data path. citeturn10search227turn10search228

---

## 4. A web application was working but went down. Networking and ports are fine. How do you troubleshoot?

If networking is already proven healthy, move from the outside inward and identify the first failing layer. Do not assume the web server is the only possible cause.

### 1. Confirm scope and symptoms

- Is the failure for all users, one region, one URL, or one browser?
- What is the exact status code: timeout, `502`, `503`, `504`, or TLS error?
- When did it start?
- Did a deployment, certificate rotation, configuration update, patch, scaling event, or dependency change occur?

```bash
curl -vkI https://app.example.com/
curl -vk https://app.example.com/health
curl -w '%{http_code} %{time_connect} %{time_starttransfer}\n' \
  -o /dev/null -s https://app.example.com/
```

Capture the response's `Server`, request ID, and timing headers to identify whether the response comes from the application, reverse proxy, Application Gateway, Front Door, or another intermediary.

### 2. Check Azure platform and load-balancer health

- Azure Resource Health and Service Health
- Application Gateway backend health
- Health probe path, host header, protocol, timeout, and accepted status code
- Backend pool membership and backend settings
- Access, performance, and WAF logs

Application Gateway continuously probes backend servers and routes traffic only to healthy backends. Its backend report shows `Healthy`, `Unhealthy`, or `Unknown`, along with the associated port, protocol, and latest probe response. citeturn10search215turn10search216

```bash
az network application-gateway show-backend-health \
  --resource-group rg-app-prod \
  --name appgw-prod
```

### 3. Check the web server or process

Yes, checking Nginx or Apache is valid when one of them hosts or proxies the application.

```bash
sudo systemctl status nginx
sudo systemctl status apache2
sudo systemctl status httpd
sudo systemctl status application.service
sudo journalctl -u nginx --since '30 minutes ago'
sudo journalctl -u application.service --since '30 minutes ago'
```

Check whether the process is listening locally:

```bash
sudo ss -lntup
curl -v http://127.0.0.1:8080/health
```

If local health fails, the issue is on the VM/application side rather than the external network path.

### 4. Inspect application logs and recent changes

```bash
sudo journalctl -p err --since '1 hour ago'
sudo tail -n 200 /var/log/nginx/error.log
sudo tail -n 200 /var/log/apache2/error.log
sudo git log -1 --oneline
```

Look for:

- Unhandled exceptions
- Invalid environment variables
- Missing configuration or secrets
- Failed migrations
- Dependency connection failures
- Port conflicts
- File-permission failures
- Expired tokens or certificates
- Crash loops or out-of-memory kills

### 5. Check host resources

```bash
uptime
free -m
df -hT
df -i
top
vmstat 1
iostat -xz 1
sudo dmesg -T | grep -iE 'oom|killed process|error|I/O'
```

Check CPU, memory, load, swap, filesystem capacity, inode exhaustion, disk latency, file descriptors, and OOM events. Memory pressure is one valid possibility, but disk or inode exhaustion is also common.

### 6. Check dependencies

An application may be locally healthy at the process level but unable to serve traffic because a database, cache, API, DNS resolver, Key Vault, storage service, or identity endpoint is unavailable.

```bash
getent hosts database.example.internal
curl -vk https://dependency.example.internal/health
nc -vz database.example.internal 5432
```

Even if the external ingress network was previously declared healthy, targeted connection tests still make sense for **backend dependencies** and the **gateway-to-backend path**.

### 7. Check TLS and certificates

```bash
openssl s_client \
  -connect app.example.com:443 \
  -servername app.example.com \
  -showcerts
```

Check certificate expiry, hostname/SAN match, chain completeness, private-key access, TLS policy, backend trust, and certificate binding. Application Gateway supports TLS termination and end-to-end TLS to the backend. citeturn10search219turn10search215

### 8. Remediate and verify

- Restore the last known-good configuration or image.
- Restart only after gathering evidence and understanding impact.
- Add capacity if resources are exhausted.
- Correct probes, secrets, certificates, permissions, or dependencies.
- Drain the backend before maintenance.
- Confirm local health, backend health, end-to-end requests, metrics, and logs.
- Document the timeline and preventive action.

---

## 5. Types of load balancers in Azure

“Load balancer” can mean several Azure traffic-distribution services. Select based on layer, scope, and protocol.

### Azure Load Balancer

- Layer 4 TCP/UDP load balancing
- Regional service for public or internal traffic
- Suitable for VM/VMSS workloads and non-HTTP protocols
- Supports health probes, load-balancing rules, inbound NAT rules, and outbound connectivity designs

### Azure Application Gateway

- Regional Layer 7 web traffic load balancer
- HTTP/HTTPS routing by host and path
- TLS termination and end-to-end TLS
- Web Application Firewall option
- Cookie-based affinity, redirects, URL rewrite, autoscaling, and zone-redundant v2 designs

Application Gateway is a web traffic load balancer and supports TLS termination, end-to-end encryption, WAF, autoscaling, and multi-zone deployment on applicable v2 SKUs. citeturn10search219turn10search220

### Azure Front Door

- Global Layer 7 entry point and application delivery network
- Anycast-based global routing
- Edge acceleration, TLS termination, health probing, caching/CDN features, and WAF
- Suitable for multi-region web applications and global failover

### Azure Traffic Manager

- DNS-based global traffic distribution
- Directs clients to endpoints based on routing method and health
- Does not proxy application traffic
- Suitable for DNS-based failover and geographic/performance routing

### Azure Cross-region Load Balancer

- Global Layer 4 load-balancing option for supported regional public Load Balancers
- Useful for cross-region TCP/UDP availability patterns

### Gateway Load Balancer

- Used to insert and scale network virtual appliances transparently
- Suitable for firewalls, packet inspection, and security appliances

### Selection summary

```text
Regional TCP/UDP             -> Azure Load Balancer
Regional HTTP/HTTPS + WAF    -> Application Gateway
Global HTTP/HTTPS            -> Azure Front Door
Global DNS routing           -> Traffic Manager
Cross-region TCP/UDP         -> Cross-region Load Balancer
Virtual network appliances   -> Gateway Load Balancer
```

---

## 6. What is Azure Application Gateway, and how does it encrypt HTTP/HTTPS traffic?

Azure Application Gateway is a regional Layer 7 web traffic load balancer. It understands HTTP/HTTPS and can make routing decisions based on host name, path, and other web request properties. It supports WAF, autoscaling, zone redundancy, redirects, rewrites, health probes, and session affinity on supported SKUs. citeturn10search219turn10search220

### Important correction

HTTP is plaintext. Application Gateway does not “encrypt HTTP” without TLS. Encryption is provided by using **HTTPS/TLS**.

### TLS termination

```text
Client -- HTTPS/TLS --> Application Gateway -- HTTP --> Backend
```

- The TLS certificate and private key are configured on the HTTPS listener.
- Application Gateway authenticates itself to the client and decrypts the request.
- Traffic to the backend is HTTP.
- This offloads TLS processing but leaves the gateway-to-backend leg unencrypted.

### End-to-end TLS

```text
Client -- HTTPS/TLS --> Application Gateway -- HTTPS/TLS --> Backend
```

- The gateway terminates the client-side TLS session.
- It creates a separate TLS session with the backend.
- The backend certificate must satisfy trust and name-validation requirements.
- For private/self-signed PKI, configure the trusted root certificate correctly.

Application Gateway supports both TLS termination and end-to-end TLS encryption. Backend health tooling can identify trust-chain and gateway-to-backend TLS failures. citeturn10search219turn10search215

### Certificate sources

- Upload a certificate to the listener.
- Integrate an appropriate Application Gateway configuration with Azure Key Vault for managed certificate retrieval/rotation.
- Restrict Key Vault access and monitor certificate expiry and retrieval errors.

### WAF

WAF protects HTTP applications against common web attacks such as SQL injection and cross-site scripting using managed and custom rules. It complements, but does not replace, secure application code, authentication, patching, and DDoS controls. citeturn10search219

---

## 7. The application is down and returns HTTP 503. What steps should be taken?

HTTP `503 Service Unavailable` means a server or intermediary is temporarily unable to handle the request. First determine **which component generated the 503**.

### Step 1: Capture evidence

```bash
curl -vk https://app.example.com/health
curl -vkI https://app.example.com/
```

Record:

- Timestamp
- Request/correlation ID
- Response headers
- Exact URL and host header
- Whether all users and routes are affected
- Deployment or configuration changes

### Step 2: Check Application Gateway backend health

```bash
az network application-gateway show-backend-health \
  --resource-group rg-app-prod \
  --name appgw-prod
```

If every backend is `Unhealthy` or `Unknown`, clients can fail because Application Gateway only forwards to healthy servers. Verify probe protocol, port, path, host header, timeout, and expected result. citeturn10search215turn10search216

### Step 3: Test backend directly from a trusted network location

```bash
curl -v http://10.20.2.10:8080/health
curl -vk --resolve app.example.com:443:10.20.2.10 \
  https://app.example.com/health
```

This separates gateway behavior from backend behavior.

### Step 4: Check service and application

```bash
sudo systemctl status nginx application.service
sudo journalctl -u application.service --since '30 minutes ago'
sudo ss -lntup
curl -v http://127.0.0.1:8080/health
```

For AKS:

```bash
kubectl get pods -n production -o wide
kubectl get endpoints,endpointslices -n production
kubectl describe pod <pod> -n production
kubectl logs <pod> -n production --previous
kubectl rollout status deployment/<app> -n production
```

### Step 5: Check saturation and capacity

```bash
free -m
df -hT
df -i
uptime
vmstat 1
iostat -xz 1
```

Investigate:

- No healthy instances or ready Pods
- CPU/memory saturation
- OOM termination
- Disk or inode exhaustion
- Worker/thread/connection-pool exhaustion
- Autoscaling at maximum capacity
- Downstream dependency outage
- Invalid secrets or expired credentials
- Failed deployment or migration

### Step 6: Check TLS and access controls

For HTTPS backends, inspect certificate trust, hostname, chain, expiry, and TLS settings. Also check NSGs, UDRs, firewalls, DNS, and backend IP restrictions even if frontend networking appears normal. Application Gateway's backend report can expose backend TLS trust-chain issues. citeturn10search215turn10search216

### Step 7: Recover safely

- Roll back the latest deployment/configuration if correlated.
- Restore healthy backend capacity.
- Correct the probe rather than weakening it without justification.
- Restart a failed service only after collecting logs and evidence.
- Drain affected backends during maintenance.
- Validate end-to-end traffic, error rate, latency, and dependency health.
- Complete root-cause analysis and preventive remediation.

---

## 8. Review of the proposed answer for Question 4

Your proposed answer is **partly correct**, but it needs prioritization and a few corrections.

### 1. Check Nginx/Apache service status

**Correct and useful.** Also check the actual application service, process, local listening port, and logs.

```bash
systemctl status nginx application.service
journalctl -u application.service --since '30 minutes ago'
ss -lntup
curl -v http://127.0.0.1:8080/health
```

### 2. Check firewall/security rules

**Correct, but lower priority if network connectivity has already been proven.** “Networking is fine” should mean the complete path is proven, including Application Gateway-to-backend, UDR, NSG, firewall, DNS, and backend dependency paths. A frontend check alone does not prove every required flow.

### 3. Ping/telnet will not make sense because networking is fine

**Partially correct.** Repeating generic ping after network health is proven adds little value. However:

- ICMP ping does not prove that the application port works.
- `telnet`/`nc` can prove basic TCP reachability to a specific backend or dependency.
- `curl` is better for HTTP because it verifies DNS, TCP, TLS, HTTP, status, headers, and response behavior.

Use targeted tests instead of abandoning network testing entirely:

```bash
nc -vz backend.internal 443
curl -vk https://backend.internal/health
```

### 4. Clear browser cache/cookies

**Possible only for a client-specific symptom, not a primary action for a confirmed application outage.** Test in a private browsing session or with `curl` first. If all users or synthetic monitors fail, browser cache is unlikely to be the cause.

### 5. Run `nslookup`

**Correct if DNS is part of the symptom.** Compare public and private resolution from relevant locations. `nslookup` proves name resolution, not application health.

```bash
nslookup app.example.com
getent hosts app.example.com
```

### 6. Check SSL certificate corruption

**Reasonable, but phrase it more precisely.** Certificates usually fail because they are expired, not yet valid, hostname-mismatched, missing an intermediate, untrusted, incorrectly bound, inaccessible through Key Vault, or paired with the wrong private key. “Corrupt” is less common.

```bash
openssl s_client -connect app.example.com:443 \
  -servername app.example.com -showcerts
```

Application Gateway supports TLS termination and end-to-end TLS, and its backend-health tooling can reveal backend certificate-chain problems. citeturn10search219turn10search215

### 7. Check lack of memory

**Correct.** Also check CPU, load, swap, OOM kills, disk capacity, inode usage, disk latency, file descriptors, process/thread limits, and application connection pools.

```bash
free -m
uptime
df -hT
df -i
dmesg -T | grep -i oom
```

### Better interview answer

> Since networking and required ports are already confirmed, I first identify the exact response and the component generating it. I check Application Gateway backend health and probes, then test the application locally on the VM. I verify Nginx/Apache and the application process, inspect service and application logs, and check CPU, memory, disk, inodes, OOM events, and connection pools. Next, I validate dependencies such as the database, cache, Key Vault, DNS, and certificates. I correlate the outage with recent deployments or configuration changes, restore or roll back safely, and then verify local health, gateway health, and end-to-end requests. Browser cache is considered only if the issue affects a specific client.

---

## Quick Revision

- Connection draining stops new requests while allowing existing connections to finish.
- `backend.tf` stays in Git; Azure Blob Storage contains the Terraform **state object**, not the backend configuration file.
- Access private VMs through Bastion, VPN, ExpressRoute, a controlled jump host, Run Command, or Serial Console.
- Troubleshoot a web outage from response origin to load balancer, backend process, resources, dependencies, TLS, and recent changes.
- Azure traffic-distribution options include Load Balancer, Application Gateway, Front Door, Traffic Manager, cross-region Load Balancer, and Gateway Load Balancer.
- Application Gateway is a Layer 7 regional web load balancer with TLS termination and end-to-end TLS options.
- For a 503, identify the component generating it and check backend health, readiness, capacity, dependencies, and recent releases.
- Checking services, DNS, certificates, and memory is valid, but browser cache is secondary and network tests should be targeted rather than dismissed.

---

## Official References

- Terraform AzureRM backend: https://developer.hashicorp.com/terraform/language/backend/azurerm
- Terraform backend configuration: https://developer.hashicorp.com/terraform/language/backend
- Terraform state backends: https://developer.hashicorp.com/terraform/language/state/backends
- `terraform init`: https://developer.hashicorp.com/terraform/cli/commands/init
- Azure Bastion remote access: https://learn.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/plan-for-virtual-machine-remote-access
- Azure Bastion IP-based connection: https://learn.microsoft.com/azure/bastion/connect-ip-address
- Application Gateway backend health: https://learn.microsoft.com/azure/application-gateway/application-gateway-backend-health
- Troubleshoot Application Gateway backend health: https://learn.microsoft.com/troubleshoot/azure/application-gateway/application-gateway-backend-health-troubleshooting
- Application Gateway features: https://learn.microsoft.com/azure/application-gateway/features
- Application Gateway backend settings: https://learn.microsoft.com/azure/application-gateway/configuration-http-settings


# DevOps Interview Questions and Answers

> **Experience level:** 3 to 4 years  
> **Topics:** Terraform, AWS networking, Jenkins, Kubernetes RBAC, NetworkPolicy, and workload controllers

## 1. How do you manage the Terraform state file?

Terraform state maps resources in the configuration to real infrastructure objects. In a team environment, I do not keep production state only on a laptop or commit `terraform.tfstate` to Git.

### Recommended approach

1. Store state in a **remote backend**, such as Amazon S3 or HCP Terraform.
2. Enable state locking using a backend that supports locking. Locking prevents concurrent writers from corrupting state. Terraform automatically locks state for write operations when the backend supports it. citeturn7search74turn7search75
3. Enable S3 versioning so an earlier state version can be recovered.
4. Encrypt the backend with AWS KMS and block public access.
5. Restrict access through least-privilege IAM roles. Separate plan and apply permissions where practical.
6. Use different state keys or separate backends for development, testing, and production.
7. Run `plan` and `apply` through CI/CD rather than individual laptops.
8. Never edit the state JSON manually. Use commands such as `terraform import`, `terraform state mv`, and `terraform state rm` under review.
9. Back up state before exceptional recovery operations.
10. Treat state as sensitive because it can contain resource attributes and secret values.

### S3 backend example

```hcl
terraform {
  backend "s3" {
    bucket       = "company-terraform-state"
    key          = "payment-service/prod/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true
    kms_key_id   = "alias/terraform-state"
  }
}
```

Initialize or migrate the backend:

```bash
terraform init
terraform init -migrate-state
```

Useful commands:

```bash
terraform state list
terraform state show aws_instance.app
terraform state pull > state-backup.json
terraform force-unlock LOCK_ID
```

`terraform force-unlock` should be used only when I am sure the original process no longer owns the lock. Incorrectly unlocking an active operation can permit multiple writers. citeturn7search75

---

## 2. How would you design architecture for a two-tier application?

A two-tier design normally has:

- **Application tier:** Web and business logic
- **Database tier:** Persistent data

### Highly available AWS design

```text
Users
  |
Route 53
  |
AWS WAF + Application Load Balancer
  |
Application replicas in private subnets
across Availability Zone A and B
  |
RDS/Aurora Multi-AZ in isolated DB subnets
```

### Design steps

1. Create one VPC across at least two Availability Zones.
2. Create public subnets for the internet-facing ALB and NAT connectivity.
3. Create private application subnets for EC2 Auto Scaling, ECS, or EKS workloads.
4. Create isolated database subnets with no direct internet route.
5. Allow HTTPS from users to the ALB.
6. Allow the application port from the ALB security group to the application security group.
7. Allow the database port only from the application security group to the database security group.
8. Deploy multiple stateless application replicas with health checks and Auto Scaling.
9. Use RDS or Aurora Multi-AZ, encryption, backups, and point-in-time recovery.
10. Store secrets in Secrets Manager, logs and metrics in CloudWatch, and audit events in CloudTrail.
11. Use VPC endpoints for supported AWS services to reduce internet/NAT dependency.
12. Test AZ failure, deployment rollback, backup restoration, and scaling.

For a simple lift-and-shift, the application may run on EC2. For a modernized platform, it can run on ECS/Fargate or EKS, while the data tier remains a managed database.

---

## 3. Difference between a subnet and a NACL

### Subnet

A subnet is an IP address range inside a VPC and belongs to one Availability Zone. Resources receive addresses from the subnet. A subnet is considered public when its route table provides a route to an Internet Gateway; otherwise it can be private or isolated depending on its routes.

### Network ACL

A Network Access Control List is a stateless, subnet-level traffic filter. It contains numbered inbound and outbound allow or deny rules. Because it is stateless, return traffic must be explicitly permitted.

### Key differences

- A subnet provides **network segmentation and IP allocation**.
- A NACL provides **subnet-boundary filtering**.
- A subnet is associated with a route table and a NACL.
- A NACL is stateless and supports both allow and deny rules.
- Security groups are stateful and apply to elastic network interfaces, while NACLs apply at the subnet boundary.
- NACL rules are evaluated in number order, and the first matching rule is applied.

A NACL does not replace routing or security groups. Routing determines where traffic goes; NACLs and security groups determine whether permitted traffic can pass.

---

## 4. Difference between NAT Gateway and Internet Gateway

### Internet Gateway

An Internet Gateway is attached to a VPC and enables communication between the VPC and the public internet when routing and addressing requirements are satisfied. A public subnet normally has `0.0.0.0/0` routed to the Internet Gateway. A public IPv4 workload also needs a public IPv4 address or Elastic IP.

### Public NAT Gateway

A public NAT Gateway allows resources in private subnets to initiate outbound connections while preventing unsolicited inbound internet connections. Traditionally, it is placed in a public subnet, uses an Elastic IP, and reaches the internet through the VPC Internet Gateway. citeturn7search81turn7search85

```text
Private instance
  |
Private route table: 0.0.0.0/0 -> NAT Gateway
  |
Public subnet route: 0.0.0.0/0 -> Internet Gateway
  |
Internet
```

### Selection

- Use an **Internet Gateway** for publicly addressed resources that must communicate directly with the internet, such as an internet-facing load balancer.
- Use a **NAT Gateway** for outbound connectivity from private IPv4 workloads.
- Use gateway or interface VPC endpoints for supported AWS services when NAT is unnecessary.
- Use an egress-only Internet Gateway for outbound-only IPv6 communication.

A NAT Gateway does not accept arbitrary internet-initiated traffic for private instances. AWS documents NAT as permitting connections initiated from private workloads while blocking unsolicited inbound initiation. citeturn7search81

---

## 5. How would you trigger Pipeline B automatically after Pipeline A?

The question likely means triggering **Pipeline B after Pipeline A**. In Pipeline A, use the Jenkins `build` step.

### Wait for Pipeline B and propagate its result

```groovy
pipeline {
    agent any

    stages {
        stage('Build and Test') {
            steps {
                sh './ci/build-and-test.sh'
            }
        }

        stage('Trigger Pipeline B') {
            steps {
                build job: 'folder/pipeline-b',
                    wait: true,
                    propagate: true,
                    parameters: [
                        string(name: 'IMAGE_TAG', value: env.BUILD_NUMBER),
                        string(name: 'SOURCE_COMMIT', value: env.GIT_COMMIT)
                    ]
            }
        }
    }
}
```

Jenkins' Pipeline Build Step triggers another job, can pass parameters, and can propagate the downstream result to the upstream pipeline. citeturn7search92turn7search95

### Fire asynchronously

```groovy
build job: 'folder/pipeline-b',
    wait: false,
    propagate: false,
    parameters: [
        string(name: 'IMAGE_TAG', value: env.BUILD_NUMBER)
    ]
```

Use `wait: true` when Pipeline A must know whether B succeeded. Use asynchronous triggering when they are operationally independent. Pass an immutable artifact version or image digest, not `latest`. Use credential parameters rather than plaintext password parameters when a downstream job needs credentials. citeturn7search92turn7search95

---

## 6. How will you know whether NetworkPolicy is enabled in Kubernetes?

Creating a `NetworkPolicy` object does not prove that it is enforced. The cluster's CNI/network plugin must support NetworkPolicy. Without an implementing controller or data plane, the object can exist but have no traffic effect. citeturn7search87

### Check the installed CNI

```bash
kubectl get pods -n kube-system
kubectl get daemonsets -n kube-system
kubectl get crds | grep -Ei 'calico|cilium|antrea|networkpolicy'
```

Review the network plugin's configuration and documentation. Examples of implementations that can enforce policies include Calico, Cilium, and Antrea. Support and configuration depend on the cluster platform.

### List current policies

```bash
kubectl get networkpolicy -A
kubectl describe networkpolicy -n application
```

### Perform an enforcement test

Create two test Pods and confirm traffic works. Then apply a default-deny policy in the test namespace:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: netpol-test
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

Test again from the source Pod:

```bash
kubectl exec -n netpol-test client -- \
  wget -T 3 -qO- http://server:8080
```

If communication remains possible after confirming selectors and endpoints, enforcement may be absent or misconfigured. Test in a controlled namespace because default-deny policies can interrupt workloads. Kubernetes NetworkPolicy controls Layer 3 and Layer 4 traffic and defines ingress and egress isolation independently. citeturn7search87

---

## 7. Difference between ClusterRole and ClusterRoleBinding

### ClusterRole

A `ClusterRole` defines a set of permissions. It is cluster-scoped and can describe access to:

- Cluster-scoped resources, such as nodes
- Namespaced resources across namespaces
- Non-resource URLs, depending on the rules

It defines **what actions are allowed**, but it does not grant those permissions to anyone by itself.

### ClusterRoleBinding

A `ClusterRoleBinding` grants the permissions in a `ClusterRole` to users, groups, or service accounts across the cluster. It defines **who receives the permissions**.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: pod-reader-binding
subjects:
  - kind: Group
    name: operations-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: pod-reader
```

A namespaced `RoleBinding` can also reference a `ClusterRole`; in that case, it grants those permissions only within the RoleBinding's namespace. Follow least privilege and verify access:

```bash
kubectl auth can-i list pods --as USER_NAME -A
kubectl auth can-i delete nodes --as USER_NAME
```

---

## 8. What happens when an IaC-managed resource is modified manually?

The real infrastructure no longer matches the Terraform configuration and recorded state. This is called **configuration drift**.

On the next `terraform plan`, Terraform refreshes resource information and normally proposes one of the following:

- Revert the manual change to match the code
- Update or replace the resource when required
- Recreate a resource that was manually deleted
- Show no change when the attribute is not managed, is provider-computed, or is intentionally ignored

### Prevention

1. Restrict production write access with IAM and AWS Organizations SCPs.
2. Allow changes through approved CI/CD roles only.
3. Give engineers read-only access and controlled, audited break-glass roles.
4. Enforce pull-request review and plan approval.
5. Use AWS Config and CloudTrail to detect and attribute manual changes.
6. Apply tags identifying IaC ownership.
7. Do not overuse `ignore_changes`, because it can hide genuine drift.

### Automated drift detection

```bash
set +e
terraform init -input=false
terraform plan -detailed-exitcode -input=false -lock-timeout=5m
status=$?
set -e

case "$status" in
  0) echo "No drift or pending changes" ;;
  1) echo "Terraform plan failed"; exit 1 ;;
  2) echo "Drift or pending changes detected"; exit 2 ;;
esac
```

Schedule this in Jenkins or another CI platform. Do not automatically apply every drift correction without review, especially for production resources.

`prevent_destroy` protects against Terraform-planned destruction while the resource remains in configuration, but it cannot block an authorized user from changing or deleting the object through the cloud console.

---

## 9. Difference between DaemonSet and StatefulSet

### DaemonSet

A DaemonSet ensures one Pod, or one Pod on each selected eligible node. It is commonly used for node-level functions such as:

- Log collection
- Node monitoring
- Security agents
- CNI components
- Storage node plugins

When a matching node joins the cluster, the DaemonSet controller schedules the Pod there. When the node is removed, that Pod disappears.

### StatefulSet

A StatefulSet manages replicas that need stable identity. Each Pod receives a predictable ordinal name, stable network identity, ordered lifecycle, and can receive its own persistent volume claim. Kubernetes defines StatefulSet Pods as having consistent network and storage identities. citeturn7search86

Common uses include databases, Kafka, ZooKeeper, and other clustered stateful systems.

### Summary

- DaemonSet placement is based primarily on **nodes**.
- StatefulSet replicas are based on a desired count and have **stable identities**.
- DaemonSet Pods generally perform node-local services.
- StatefulSet Pods commonly own distinct persistent state.
- Use a Deployment for interchangeable stateless application replicas.

---

## 10. How would you set up networking in a VPC?

### Step-by-step design

1. Select a non-overlapping VPC CIDR, considering future VPC, on-premises, and Transit Gateway connectivity.
2. Use at least two Availability Zones for production.
3. Create subnet tiers in each AZ:
   - Public subnets for internet-facing load balancers and traditional public NAT Gateways
   - Private application subnets for EC2, ECS, or EKS
   - Isolated database subnets for RDS and caches
4. Attach an Internet Gateway to the VPC.
5. Associate public route tables with a default route to the Internet Gateway.
6. Provide controlled private-subnet egress through NAT, VPC endpoints, proxies, or centralized egress.
7. Keep database subnets without a direct default internet route.
8. Use security groups as the primary stateful workload firewall.
9. Use NACLs as coarse stateless subnet guardrails where required.
10. Add gateway endpoints for S3/DynamoDB and interface endpoints for supported AWS APIs.
11. Configure Route 53 private hosted zones and DNS Resolver endpoints/rules if hybrid DNS is needed.
12. Use Transit Gateway, Cloud WAN, peering, VPN, or Direct Connect according to scale and topology.
13. Enable VPC Flow Logs and monitor rejected traffic, IP usage, NAT errors, and endpoint health.

Example:

```text
VPC: 10.20.0.0/16

AZ-a                         AZ-b
Public:  10.20.0.0/24        10.20.1.0/24
App:     10.20.10.0/23       10.20.12.0/23
DB:      10.20.20.0/24       10.20.21.0/24
```

AWS's traditional resilient VPC pattern places application servers in private subnets across AZs, receives requests through an ALB, provides outbound access through NAT, and can use an S3 gateway endpoint. citeturn7search80turn7search85

---

## 11. How will you direct traffic to and from an instance in a private subnet?

### Inbound application traffic

Do not expose the private instance directly. Use:

```text
Internet -> Internet Gateway -> Public ALB/NLB
         -> private instance target
```

The private subnet does not need a route to the Internet Gateway for return traffic to an ALB target. Security groups should allow only the application port from the load balancer's security group.

### Outbound internet traffic

```text
Private instance -> private route table -> public NAT Gateway
                 -> Internet Gateway -> internet
```

A traditional public NAT Gateway is placed in a public subnet and receives an Elastic IP. The private subnet sends its default IPv4 route to the NAT Gateway. citeturn7search81turn7search85

### Access to AWS services

Use VPC endpoints where supported:

- Gateway endpoints for S3 and DynamoDB
- Interface endpoints for services such as Systems Manager, ECR APIs, CloudWatch Logs, and Secrets Manager

This can avoid NAT for AWS service traffic.

### Administrative access

Prefer Systems Manager Session Manager instead of opening SSH through a bastion. The instance requires an IAM role, Systems Manager agent, and connectivity to the Systems Manager endpoints through NAT or VPC interface endpoints.

### Private connectivity from other networks

Use:

- VPC peering
- Transit Gateway
- Site-to-Site VPN
- Direct Connect
- Client VPN
- PrivateLink for suitable service-provider patterns

### Verification

```bash
ip route
ss -lntp
curl -v http://127.0.0.1:8080/health
traceroute DESTINATION
aws ec2 describe-route-tables
aws ec2 describe-security-groups
```

Also verify NACLs, security groups, source/destination routing, DNS, load-balancer target health, application listener binding, and VPC Flow Logs.

---

## Quick interview revision

- Store Terraform state remotely with encryption, versioning, access control, and locking.
- A two-tier system has application and database tiers; distribute both according to their availability requirements.
- A subnet allocates and segments IP space; a NACL filters subnet traffic.
- An Internet Gateway connects a VPC to the internet; NAT provides outbound translation for private workloads.
- Trigger Pipeline B from Pipeline A with Jenkins' `build` step.
- A NetworkPolicy resource works only when the CNI or networking implementation enforces it.
- ClusterRole defines permissions; ClusterRoleBinding grants them cluster-wide to subjects.
- Manual infrastructure changes cause drift; prevent them with IAM/SCP controls and detect them with scheduled plans.
- DaemonSet runs node-oriented Pods; StatefulSet provides stable workload identities and storage relationships.
- Design VPCs with non-overlapping CIDRs, multiple AZs, separate subnet tiers, controlled routing, endpoints, and flow logs.
- Private instances receive application traffic through a load balancer and use NAT or endpoints for controlled egress.
# DevOps Interview Questions and Answers

> **Topics:** Docker, CI/CD, Kubernetes, Terraform, cost optimization, Python, and Ansible  
> **Note:** Adapt the sample answers to your real project experience. Do not claim tools, incidents, or results you have not personally handled.

## 1. Day-to-day tasks

### Sample answer

My daily work usually includes:

- Reviewing monitoring dashboards, alerts, overnight failures, and production health.
- Troubleshooting application, Kubernetes, Linux, network, and CI/CD issues.
- Managing CI/CD pipelines and resolving failed builds or deployments.
- Reviewing pull requests for Terraform, Helm, Ansible, Dockerfiles, and pipeline code.
- Deploying services to development, testing, staging, and production environments.
- Managing Kubernetes Deployments, Services, Ingress, ConfigMaps, Secrets, autoscaling, and resource limits.
- Writing and maintaining reusable Terraform modules and reviewing infrastructure plans.
- Automating configuration and patching through Ansible.
- Managing container images, vulnerability findings, and registry lifecycle policies.
- Performing cost, capacity, security, backup, certificate, and compliance checks.
- Participating in stand-ups, change reviews, incident calls, root-cause analysis, and sprint planning.
- Updating runbooks, architecture documents, and operational knowledge-base articles.

A good answer should conclude with impact, for example: “My focus is to keep deployments repeatable, reduce manual work, improve reliability, and maintain measurable operational controls.”

---

## 2. How did you reduce Docker image sizes?

I first measured the image with `docker image inspect`, `docker history`, and a layer-analysis tool. I then applied these improvements:

1. Used **multi-stage builds** so compilers, SDKs, source code, and test tools remained in the builder stage.
2. Used a smaller approved runtime image such as `slim`, Alpine where compatible, distroless, or `scratch` for a static binary.
3. Copied only the final application artifact and required runtime files.
4. Added `.dockerignore` to exclude `.git`, test reports, documentation, local dependencies, temporary files, credentials, and build output.
5. Installed production dependencies only, such as `npm ci --omit=dev`.
6. Removed package-manager caches in the same layer in which packages were installed.
7. Combined related `RUN` instructions where it prevented temporary files from remaining in earlier layers.
8. Removed unnecessary packages, shells, debugging utilities, and duplicate files.
9. Optimized Java runtime content with an appropriate JRE or `jlink` where operationally suitable.
10. Rebuilt and tested the final image, scanned it, and compared image size and vulnerability count.

Docker multi-stage builds use multiple `FROM` instructions and selectively copy artifacts into the final stage, leaving unwanted build tools and intermediate files behind. citeturn8search119turn8search120

### Example

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.26-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build \
    -trimpath -ldflags="-s -w" -o /out/service ./cmd/service

FROM scratch
COPY --from=build /out/service /service
USER 10001:10001
EXPOSE 8080
ENTRYPOINT ["/service"]
```

An interview-quality result should be measurable: “The image decreased from X MB to Y MB, reducing pull time and the runtime attack surface.”

---

## 3. Explain the CI/CD pipeline used

### Sample pipeline flow

```text
Developer commit
  -> Pull-request validation
  -> Checkout
  -> Lint and unit tests
  -> SAST, SCA, secret and IaC scans
  -> Build artifact
  -> Build and scan container image
  -> Generate SBOM and sign image
  -> Push immutable image to registry
  -> Deploy to development
  -> Integration and smoke tests
  -> Approval
  -> Staging/production deployment
  -> Health verification and rollback
```

### Detailed explanation

1. A commit or pull request triggers Jenkins, GitHub Actions, GitLab CI, or Azure Pipelines.
2. The pipeline checks out the exact commit and assigns a traceable version using the Git SHA.
3. Linting, unit tests, and fast validation run first.
4. Independent SAST, dependency, secret, image, and Terraform scans run in parallel.
5. The application artifact is built once.
6. A multi-stage Dockerfile creates an immutable container image.
7. The image is scanned, an SBOM is generated, and critical policy violations fail the pipeline.
8. The image is signed and pushed to a registry using a commit-based tag and digest.
9. Helm or GitOps deploys the same image to development.
10. Integration and smoke tests validate the deployment.
11. Production requires approved promotion based on the project’s change policy.
12. The release uses rolling, canary, or blue-green deployment with automatic health verification and rollback.
13. Logs, metrics, deployment status, and notifications provide an audit trail.

Important practices:

- Build once and promote the same image digest.
- Keep credentials in a credential manager and use short-lived identities.
- Parallelize independent checks.
- Cache dependencies safely.
- Use protected branches and reviewed pipeline code.
- Retain test, scan, SBOM, plan, and deployment evidence.

---

## 4. What have you done in Kubernetes?

### Sample answer

I have worked on:

- Deploying microservices through Deployments and Helm charts.
- Creating ClusterIP Services and exposing HTTP applications through an Ingress controller.
- Managing ConfigMaps and integrating Secrets from a controlled secret store.
- Defining CPU/memory requests and limits.
- Configuring liveness, readiness, and startup probes.
- Configuring HPA, Pod Disruption Budgets, and topology-spread constraints.
- Using PVCs and StorageClasses for applications requiring persistent storage.
- Applying RBAC and service accounts using least privilege.
- Defining NetworkPolicies for ingress and egress isolation.
- Troubleshooting pending, crashing, OOM-killed, image-pull, DNS, CNI, storage, and rollout issues.
- Performing rolling updates, rollback, node drain, and controlled upgrades.
- Monitoring Pods, nodes, events, logs, and application metrics.

Useful troubleshooting commands:

```bash
kubectl get pods -A -o wide
kubectl describe pod POD -n NAMESPACE
kubectl logs POD -n NAMESPACE --all-containers --previous
kubectl get events -n NAMESPACE --sort-by=.metadata.creationTimestamp
kubectl get deploy,svc,ingress,endpointslice -n NAMESPACE
kubectl rollout status deployment/APP -n NAMESPACE
kubectl rollout undo deployment/APP -n NAMESPACE
```

---

## 5. How did you write microservice Deployment files and configure Services and Ingress?

I created a reusable Helm chart with environment-specific values. Each microservice had a Deployment, Service, ConfigMap/Secret references, autoscaling settings, disruption controls, and optional Ingress rules.

### Deployment and Service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-api
  namespace: production
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app: orders-api
  template:
    metadata:
      labels:
        app: orders-api
    spec:
      serviceAccountName: orders-api
      terminationGracePeriodSeconds: 30
      containers:
        - name: orders-api
          image: registry.example.com/orders-api@sha256:REPLACE_DIGEST
          ports:
            - name: http
              containerPort: 8080
          envFrom:
            - configMapRef:
                name: orders-api-config
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: 1
              memory: 512Mi
          startupProbe:
            httpGet:
              path: /health/startup
              port: http
            failureThreshold: 30
            periodSeconds: 5
          readinessProbe:
            httpGet:
              path: /health/ready
              port: http
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health/live
              port: http
            periodSeconds: 20
---
apiVersion: v1
kind: Service
metadata:
  name: orders-api
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: orders-api
  ports:
    - name: http
      port: 80
      targetPort: http
```

### Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: application
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /orders
            pathType: Prefix
            backend:
              service:
                name: orders-api
                port:
                  number: 80
```

The request path is:

```text
Client -> external load balancer -> Ingress controller
       -> ClusterIP Service -> ready Pod
```

An Ingress exposes HTTP/HTTPS routes to Services and needs an Ingress controller; creating the resource alone has no effect. Kubernetes now recommends evaluating Gateway API for new features because the Ingress API is stable but frozen. citeturn8search108

---

## 6. Did you configure ingress and egress rules?

Yes. I used Kubernetes NetworkPolicy for workload-level Layer 3/4 controls and cloud security groups/NACLs for infrastructure-level controls.

### Example policy

This allows traffic to `orders-api` only from Pods labelled `app=frontend` in namespaces labelled `access=application`, and allows egress only to DNS and PostgreSQL.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: orders-api-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: orders-api
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              access: application
          podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
    - to:
        - ipBlock:
            cidr: 10.30.20.0/24
      ports:
        - protocol: TCP
          port: 5432
```

I normally start with default-deny policies in a test environment, then add explicit DNS, metrics, service-to-service, database, and approved external access. NetworkPolicy controls Pod traffic at Layer 3/4 and requires a network plugin that enforces it; creating a policy without an implementing plugin has no effect. citeturn8search107

I verify policies using test Pods, application traces, CNI flow visibility, and both positive and negative connectivity tests.

---

## 7. What have you done in Terraform, and how did you integrate it?

### Sample answer

I used Terraform to provision and manage VPCs, subnets, route tables, security groups, load balancers, compute, container clusters, IAM roles, databases, storage, DNS, monitoring, and supporting services.

I organized the code into reusable modules and environment-specific root modules:

```text
terraform/
├── modules/
│   ├── network/
│   ├── kubernetes/
│   └── database/
└── environments/
    ├── dev/
    ├── test/
    └── prod/
```

### CI/CD integration

1. Pull request triggers `terraform fmt -check`, `validate`, `tflint`, and security/policy scans.
2. Pipeline authenticates through a short-lived cloud role.
3. `terraform plan` creates a saved plan.
4. The plan is attached to the pull request for review.
5. Approved changes are merged.
6. The protected apply job uses the reviewed saved plan.
7. State is stored remotely with encryption, versioning, access control, and locking.
8. A scheduled plan detects drift.

```bash
terraform fmt -check -recursive
terraform init -input=false
terraform validate
terraform plan -input=false -out=tfplan
terraform apply -input=false tfplan
```

I keep provider versions pinned, commit the dependency lock file, isolate production state, avoid secrets in code, and restrict apply permissions to the pipeline role.

---

## 8. Difference between `terraform destroy` and refresh

### `terraform destroy`

`terraform destroy` creates and applies a plan that deletes all objects managed by the selected Terraform configuration and state, subject to lifecycle rules, dependencies, permissions, and provider behavior.

```bash
terraform plan -destroy
terraform destroy
```

It is destructive and should require explicit approval in protected environments.

### Refresh behavior

A refresh reads remote infrastructure and updates Terraform's understanding of resource attributes. Modern `terraform plan` and `terraform apply` refresh by default unless refresh is disabled.

The legacy `terraform refresh` command updates state to match remote objects without changing infrastructure. It can record out-of-band changes into state, so it should not be used casually. Prefer reviewable refresh-only operations:

```bash
terraform plan -refresh-only
terraform apply -refresh-only
```

### Key difference

```text
Destroy      -> removes managed infrastructure
Refresh-only -> reconciles Terraform state with existing infrastructure
```

Refresh-only does not restore manually changed infrastructure to match the code. A normal plan shows the proposed reconciliation. Terraform lifecycle rules customize creation, replacement, and destruction behavior. citeturn8search101turn8search102

---

## 9. How do you prevent someone from running `terraform destroy`?

Use multiple controls because no single safeguard is sufficient.

### Terraform safeguard

```hcl
resource "aws_db_instance" "production" {
  identifier = "production-db"
  # Other arguments omitted

  deletion_protection = true
  skip_final_snapshot = false

  lifecycle {
    prevent_destroy = true
  }
}
```

`prevent_destroy` blocks a Terraform plan that would destroy the resource while the resource and lifecycle rule remain in the configuration. Removing the resource block can remove that protection from Terraform's evaluation, so additional controls are required. citeturn8search102

### Stronger operational controls

- Deny destructive cloud API actions through IAM and organization policies except for a protected break-glass role.
- Give the normal CI plan role read-only permissions and the apply role controlled write access.
- Remove production cloud credentials from engineers' local environments.
- Protect the production pipeline with approvals, branch controls, and environment permissions.
- Reject plans containing delete/replace operations unless explicitly approved.
- Enable service-side deletion protection for databases and other supported resources.
- Enable backup, versioning, object lock, and recovery controls where appropriate.
- Protect and audit state-backend access.
- Alert on deletion API calls through audit logs.

```bash
terraform show -json tfplan > tfplan.json
# A policy engine evaluates resource_changes[].change.actions
```

Do not rely on hiding the command or renaming the binary. Authorization and policy enforcement must prevent destructive API operations.

---

## 10. How did you perform cost optimization?

I used a measure, optimize, verify approach.

### Actions

- Tagged resources by application, environment, owner, and cost center.
- Reviewed cost reports, budgets, anomaly alerts, and service-level usage.
- Rightsized EC2, database, Kubernetes, and container CPU/memory based on historical percentiles.
- Scheduled non-production shutdown and startup.
- Used Auto Scaling and removed permanently idle capacity.
- Purchased Savings Plans or reservations for stable baseline compute after usage analysis.
- Used Spot capacity only for interruptible workloads with graceful interruption handling.
- Removed unattached EBS volumes, old snapshots, idle load balancers, stale Elastic IPs, and unused NAT paths.
- Added S3 lifecycle rules and log-retention policies.
- Reduced NAT data-processing cost through VPC endpoints and same-AZ routing where appropriate.
- Optimized container requests to improve node utilization.
- Added registry image-retention policies.
- Scaled development clusters down outside working hours.
- Verified that cost changes did not reduce reliability, security, performance, or recovery capability.

### Sample result statement

> We identified underutilized non-production resources, scheduled them outside business hours, reduced oversized Kubernetes requests, and added storage lifecycle rules. We measured the monthly baseline before and after the change and achieved a documented reduction without violating application SLOs.

Use your actual percentage and amount if you have evidence. Do not invent savings.

---

## 11. Python script to separate list items beginning with `a` or `b`

The supplied list appears to contain a punctuation typo. This example assumes:

```python
items = ["abc", "vca", "abc", "bca"]
```

### Simple solution

```python
items = ["abc", "vca", "abc", "bca"]

starts_with_a = []
starts_with_b = []
others = []

for item in items:
    normalized = item.strip().lower()

    if normalized.startswith("a"):
        starts_with_a.append(item)
    elif normalized.startswith("b"):
        starts_with_b.append(item)
    else:
        others.append(item)

print("Starts with a:", starts_with_a)
print("Starts with b:", starts_with_b)
print("Others:", others)
```

Output:

```text
Starts with a: ['abc', 'abc']
Starts with b: ['bca']
Others: ['vca']
```

### Reusable function

```python
from collections import defaultdict
from typing import Iterable


def group_by_first_letter(items: Iterable[str]) -> dict[str, list[str]]:
    grouped: dict[str, list[str]] = defaultdict(list)

    for item in items:
        if not isinstance(item, str):
            raise TypeError(f"Expected a string, got {type(item).__name__}")

        normalized = item.strip().lower()
        key = normalized[0] if normalized else "empty"
        grouped[key].append(item)

    return dict(grouped)


items = ["abc", "vca", "abc", "bca"]
groups = group_by_first_letter(items)

print("a:", groups.get("a", []))
print("b:", groups.get("b", []))
print("other:", [
    item
    for key, values in groups.items()
    if key not in {"a", "b"}
    for item in values
])
```

---

## 12. What have you done in Ansible?

### Sample answer

I used Ansible for repeatable configuration management and operational automation, including:

- Installing and configuring application and platform packages.
- Managing users, groups, SSH settings, files, templates, services, and scheduled jobs.
- Applying OS hardening and patching.
- Configuring web servers, monitoring agents, log collectors, and middleware.
- Performing rolling deployments with `serial` and health validation.
- Creating reusable roles with defaults, variables, handlers, templates, and tests.
- Managing inventories using groups and environment-specific variables.
- Protecting secrets with Ansible Vault or an external secret manager.
- Running syntax checks, `ansible-lint`, check mode, and CI tests.
- Integrating playbooks with Jenkins and Automation Controller.

Ansible playbooks are YAML-based, repeatable automation definitions that execute ordered plays and tasks across target hosts. Tasks invoke modules, and playbooks should aim to be idempotent. citeturn8search115

### Example

```yaml
---
- name: Configure web servers
  hosts: web
  become: true
  serial: 25%

  pre_tasks:
    - name: Verify available disk space
      ansible.builtin.assert:
        that:
          - ansible_facts.mounts
        fail_msg: "Mount facts are unavailable"

  roles:
    - role: company.nginx

  post_tasks:
    - name: Verify application health
      ansible.builtin.uri:
        url: http://127.0.0.1:8080/health
        status_code: 200
        timeout: 10
```

---

## 13. An Ansible playbook has been running for two to three hours. What next?

Do not immediately terminate it, because it may be performing a non-interruptible package, database, storage, or deployment operation.

### Troubleshooting sequence

1. Identify the currently running task and affected hosts from the job output.
2. Determine whether it is actually progressing or blocked.
3. Check controller CPU, memory, disk, network, process state, and job queue.
4. Test SSH connectivity and latency to the affected host.
5. Check the remote process, package-manager locks, DNS, repository access, sudo prompts, storage I/O, and external API calls.
6. Review task design for missing timeouts, unlimited retries, blocking commands, or very large loops.
7. Reduce scope safely with `--limit` and retry failed hosts.
8. If safe, stop the run gracefully, fix the task, and restart from a known idempotent point.

Commands:

```bash
ansible all -m ansible.builtin.ping -i inventory
ansible-playbook site.yml -i inventory -vvv
ansible-playbook site.yml -i inventory --list-hosts
ansible-playbook site.yml -i inventory --list-tasks
ansible-playbook site.yml -i inventory --limit problem-host
ansible-playbook site.yml -i inventory --start-at-task "Task name"
ansible-playbook site.yml -i inventory --step
```

Ansible supports `--start-at-task` and `--step` for debugging and controlled re-execution. citeturn8search116

### Typical causes

- SSH timeout, packet loss, or DNS delay
- Unreachable package repository
- Package-manager lock
- A command waiting for interactive input
- Missing `become` credentials
- Slow fact gathering
- External API without a timeout
- Retry loop with a long delay
- Serial execution across many hosts
- Large file transfer or template
- Handler or service restart waiting indefinitely
- Controller disk-space or execution-capacity issue

### Improvements

- Use module and API timeouts.
- Use `async` and `poll` for legitimate long operations.
- Add bounded `retries`, `delay`, and `until`.
- Apply `serial`, `throttle`, and controlled forks.
- Make tasks idempotent and tag them logically.
- Record per-task timing through callback plugins.
- Add prechecks and post-deployment health validation.

For Automation Controller jobs stuck in `Pending`, verify execution capacity and controller health; official troubleshooting also calls out services and available `/var` space as important checks. citeturn8search113turn8search114

---

## 14. What is Ansible Tower?

Ansible Tower was the enterprise web and API layer for Ansible automation. Its modern product name is **automation controller**, a component of Red Hat Ansible Automation Platform.

It provides:

- Centralized web UI and REST API
- Projects synchronized from source control
- Inventories and dynamic inventory sources
- Machine, cloud, Vault, and other credential management
- Job templates and workflow templates
- Schedules and notifications
- Role-based access control
- Surveys and runtime parameters
- Job logs, audit history, and centralized visibility
- Execution environments and scalable execution capacity
- Integration with CI/CD, ITSM, and event-driven workflows

Automation controller is described as the command-and-control component of Red Hat Ansible Automation Platform, with a web UI, API, RBAC, workflows, credential management, and CI/CD integrations. citeturn8search117

### Typical flow

```text
Git repository -> Project sync -> Job template
               -> Inventory + Credential + Execution Environment
               -> Managed hosts -> Logs/notifications/audit
```

### Tower versus command line

- CLI is suitable for local development and controlled ad hoc execution.
- Automation Controller is more suitable for team governance, credential separation, schedules, approvals, workflows, centralized logs, API integration, and auditability.

Credentials should be stored in Controller or an integrated secret manager, not embedded in playbooks or inventory files.

---

## Quick interview revision

- Describe day-to-day work in terms of reliability, automation, security, delivery, and measurable outcomes.
- Reduce image size with multi-stage builds, minimal runtime images, `.dockerignore`, and production-only dependencies.
- Build once and promote the same image digest through CI/CD environments.
- In Kubernetes, use Deployments, Services, Ingress/Gateway, probes, resources, RBAC, autoscaling, and NetworkPolicy.
- Ingress routes HTTP/HTTPS traffic; NetworkPolicy controls allowed Pod flows.
- Integrate Terraform through PR validation, saved plans, approvals, remote state, and protected applies.
- `terraform destroy` deletes managed infrastructure; refresh-only updates state from remote infrastructure.
- Prevent destruction with layered Terraform, cloud authorization, service deletion protection, pipeline policy, and backup controls.
- Optimize cost only after measuring usage, and verify SLOs after each change.
- In Ansible, use reusable, idempotent roles and investigate long jobs before stopping them.
- Ansible Tower is now known as automation controller within Red Hat Ansible Automation Platform.
