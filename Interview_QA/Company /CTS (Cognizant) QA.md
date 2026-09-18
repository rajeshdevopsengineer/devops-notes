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
