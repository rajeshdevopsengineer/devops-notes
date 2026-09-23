Below are interview-ready answers with practical AWS, Terraform, HTTP, and Kubernetes examples.

**1. What is a Web Application Firewall?**

A **Web Application Firewall (WAF)** inspects HTTP and HTTPS requests and applies rules to protect a web application. It operates at the application layer—Layer 7.

Depending on its rules, a WAF can inspect request paths, query strings, headers, cookies, and request bodies to identify suspicious traffic.

Typical protections include:

* Detecting common SQL injection and cross-site scripting payloads.
* Restricting access by IP address or geography.
* Applying request-rate limits.
* Challenging or blocking suspicious automated clients.
* Blocking requests that violate application-specific rules.

In AWS, you associate a WAF web ACL with supported resources such as **CloudFront, an Application Load Balancer, or API Gateway**. You do not attach AWS WAF directly to an EC2 instance. Rules can allow, block, count, or challenge requests. ([docs.aws.amazon.com][1])

**Example:** A rule might block suspicious SQL expressions in a search request. However, the application must still use parameterized database queries; a WAF does not repair vulnerable code.

---

**2. How would you secure a cloud web application against OWASP Top 10 risks?**

I would apply security across the application, delivery pipeline, cloud infrastructure, and operations.

The **current OWASP Top 10 is the 2025 edition**. It covers broad risk categories; enabling a WAF does not address every category. ([owasp.org][2])

| Risk area                                 | Practical controls                                                                                                          |
| ----------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Unauthorized access to data or operations | Check permissions and resource ownership on every request; deny access by default.                                          |
| Unsafe configuration                      | Disable debug endpoints, remove default credentials, restrict public access, and manage configuration through reviewed IaC. |
| Compromised dependencies or build systems | Scan dependencies, use trusted registries, protect build credentials, and verify artifact provenance.                       |
| Weak protection of sensitive data         | Use TLS, encryption at rest, managed keys, and appropriate password hashing.                                                |
| Injection and browser scripting attacks   | Use parameterized queries, contextual output encoding, input validation, and supplementary WAF rules.                       |
| Insecure application design               | Perform threat modeling and review business workflows for abuse cases.                                                      |
| Weak login and session handling           | Use strong authentication, MFA where appropriate, secure cookies, and session expiration.                                   |
| Untrusted software or data changes        | Validate signatures and integrity; avoid unsafe deserialization.                                                            |
| Missing security visibility               | Collect authentication, authorization, administrative, and WAF events; create actionable alerts.                            |
| Unsafe handling of failures               | Fail securely, avoid exposing stack traces, bound resource consumption, and test dependency failures.                       |

These controls address the categories highlighted by OWASP; the implementation must match the application’s risks. ([OWASP Top 10:2025][3])

For example, changing `/accounts/123` to `/accounts/124` must not expose another customer’s account. The application must verify ownership even when the request looks completely valid to the WAF. ([OWASP Cheat Sheet Series][4])

In the pipeline, I would combine code analysis, dependency scanning, secret scanning, container scanning, and application security testing. For SQL injection specifically, parameterized queries remain a primary control. ([OWASP Cheat Sheet Series][5])

---

**3. What happens if the Terraform state file gets deleted?**

**The cloud resources remain running, but Terraform loses the mapping between its resource addresses and those resources.**

For example, state records that:

```text
aws_instance.web corresponds to EC2 instance i-0123456789abcdef0
```

Without that mapping, Terraform may plan to create another instance. For resources with unique names, creation may instead fail because the resource already exists. Terraform does not automatically rediscover all existing resources from matching names. ([HashiCorp Developer][6])

My recovery process would be:

1. **Stop Terraform apply jobs** to prevent changes while investigating.
2. **Confirm the backend and workspace.** Deleting a local file does not necessarily delete authoritative state stored remotely.
3. **Restore the latest valid state backup.** This could be a previous S3 object version, a managed backend state version, or a local backup.
4. **Review a new plan** against the actual infrastructure.
5. If no usable backup exists, **rebuild state by importing existing resources** into matching configuration. ([HashiCorp Developer][7])

Example import block, assuming `aws_instance.web` is already declared correctly:

```hcl
import {
  to = aws_instance.web
  id = "i-0123456789abcdef0"
}
```

Then:

```bash
terraform init
terraform plan -out=recovery.tfplan
terraform show recovery.tfplan
```

Apply only after checking that the plan imports the expected resources without unintended replacements or deletions. ([HashiCorp Developer][8])

**`terraform init` alone cannot reconstruct deleted state.**

---

**4. What is the `.terraform.lock.hcl` file?**

It is Terraform’s **provider dependency lock file**. It records:

* Selected provider versions.
* Relevant version constraints.
* Checksums used to verify provider packages.

This helps developers and CI agents install consistent provider versions.

Terraform normally creates or updates it during:

```bash
terraform init
```

To intentionally reconsider provider versions within configured constraints:

```bash
terraform init -upgrade
```

Commit the lock file to Git and review changes. It currently locks **provider dependencies**, not remote module versions; manage module versions separately. ([HashiCorp Developer][9])

These three concepts are different:

| Item                  | Purpose                                                                     |
| --------------------- | --------------------------------------------------------------------------- |
| `.terraform.lock.hcl` | Records provider dependency selections and checksums.                       |
| Terraform state       | Tracks managed resources and their attributes.                              |
| Backend state lock    | Prevents competing operations from modifying the same state simultaneously. |

The dependency lock file does **not** prevent two engineers from running competing applies.

---

**5. What Terraform best practices would you follow?**

I would focus on reproducibility, controlled changes, and recoverable state.

| Area                  | Practice                                                                                   |
| --------------------- | ------------------------------------------------------------------------------------------ |
| Code organization     | Use reusable modules with clear inputs and outputs.                                        |
| Environment isolation | Separate production and nonproduction state and access permissions.                        |
| Version management    | Constrain Terraform/provider versions, commit the provider lock file, and version modules. |
| Review                | Use pull requests and review the Terraform plan before applying.                           |
| Validation            | Run formatting, validation, linting, security checks, and relevant module tests.           |
| Resource changes      | Use imports and `moved` blocks for controlled adoption or refactoring.                     |
| Drift                 | Run scheduled plans and investigate unexpected changes.                                    |
| Operations            | Serialize applies for each state and document recovery procedures.                         |

Consistent structure and naming make changes easier to review and maintain. ([HashiCorp Developer][10])

For AWS, an example remote backend is:

```hcl
terraform {
  backend "s3" {
    bucket       = "example-company-terraform-state"
    key          = "prod/network/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

The bucket must already exist. Enable versioning and restrict access. Current S3 backends support native locking through `use_lockfile`; DynamoDB-based locking is deprecated. ([HashiCorp Developer][11])

A typical delivery sequence is:

```bash
terraform fmt -check -recursive
terraform init
terraform validate
terraform plan -out=tfplan
terraform show tfplan

# After reviewing the saved plan:
terraform apply tfplan
```

Use short-lived pipeline credentials. Protect saved plans and state because they can contain secrets. Marking a value `sensitive` hides it from normal display but does **not** automatically exclude it from state. ([HashiCorp Developer][12])

---

**6. If an instance has a security group, do we still need a NACL?**

Every AWS subnet is associated with a NACL, so the practical question is whether you need **custom restrictive NACL rules** in addition to security groups.

Security groups are normally the primary control. Custom NACLs provide additional subnet-level restrictions when required.

| Feature             | Security group                                  | Network ACL                                   |
| ------------------- | ----------------------------------------------- | --------------------------------------------- |
| Applies to          | Associated network interfaces/resources         | Traffic entering or leaving a subnet          |
| Connection tracking | Stateful                                        | Stateless                                     |
| Rules               | Allow rules                                     | Allow and deny rules                          |
| Evaluation          | Applicable allow rules are combined             | First matching rule, ordered by rule number   |
| Return traffic      | Automatically permitted for tracked connections | Must be explicitly allowed                    |
| Typical use         | Application-specific access                     | Broad subnet restrictions and explicit denies |

AWS recommends security groups as the primary mechanism, with NACLs used where additional coarse-grained controls are useful. ([Amazon Virtual Private Cloud][13])

**Example:** An application server’s security group allows port 8080 only from the ALB security group. A subnet NACL could additionally deny a known malicious network.

Restrictive NACLs require careful return-path rules, including appropriate ephemeral ports. Duplicating every security group rule into a NACL can create unnecessary operational complexity.

---

**7. What is the difference between Transit Gateway and a VPC?**

A **VPC is a virtual network where workloads run**. A **Transit Gateway connects multiple networks**.

| Aspect               | VPC                                             | Transit Gateway                                                      |
| -------------------- | ----------------------------------------------- | -------------------------------------------------------------------- |
| Purpose              | Provide an isolated network environment         | Provide centralized routing between networks                         |
| Contains or connects | Subnets, network interfaces, workload resources | VPC, VPN, Direct Connect gateway, and other supported attachments    |
| Routing              | VPC route tables                                | Transit Gateway route tables                                         |
| Example              | Host an application and its database            | Connect application VPCs to shared services and on-premises networks |

For example, three teams might each have a separate VPC. A Transit Gateway can connect those VPCs to shared DNS services and an on-premises network.

Connectivity requires appropriate **VPC routes, Transit Gateway routes, and security rules**. Creating attachments alone does not guarantee application connectivity. Transit Gateway route tables also support segmentation between attachments. ([Amazon Virtual Private Cloud][13])

If the interviewer meant **Transit Gateway versus VPC peering**: peering connects VPC pairs and is nontransitive, while Transit Gateway supports centralized transit routing.

---

**8. What cloud security best practices would you follow?**

I would begin with the data sensitivity, business impact, and access requirements, then apply controls across these areas:

**Identity**

* Federate human access through a central identity provider.
* Require MFA.
* Use workload roles and temporary credentials.
* Apply least privilege and remove unused permissions.
* Protect root access and avoid root access keys.

These align with AWS IAM guidance. ([AWS Identity and Access Management][14])

**Network and application protection**

* Keep databases and internal services private.
* Expose only required public endpoints.
* Restrict security groups and administrative access.
* Apply WAF, DDoS protection, and application authorization where appropriate.

**Data and software protection**

* Encrypt sensitive data in transit and at rest.
* Store credentials in a secret manager.
* Patch operating systems, containers, and dependencies.
* Scan and review infrastructure and application changes.

**Detection and recovery**

* Centralize audit and security logs.
* Alert on suspicious identity activity and configuration changes.
* Maintain tested backups and incident response procedures.
* Define ownership for findings and remediation.

The cloud provider’s controls do not remove the customer’s responsibility for identities, application code, data access, and service configuration. Security must be applied and verified across multiple layers. ([Security Pillar][15])

---

**9. What are HTTP request headers and HTTP methods?**

An **HTTP method** describes the requested operation. **Headers** provide information about the request, credentials, content, or expected response.

Example:

```http
GET /orders/123 HTTP/1.1
Host: api.example.com
Authorization: Bearer <access-token>
Accept: application/json
```

Common headers:

| Header          | Purpose                                                        |
| --------------- | -------------------------------------------------------------- |
| `Host`          | Identifies the target host in HTTP/1.1.                        |
| `Authorization` | Carries authentication credentials, such as a bearer token.    |
| `Content-Type`  | Describes the format of the request body.                      |
| `Accept`        | States acceptable response formats.                            |
| `Cookie`        | Sends applicable cookies to the server.                        |
| `User-Agent`    | Identifies client software.                                    |
| `If-Match`      | Makes an operation conditional on a matching resource version. |

For example, `Content-Type: application/json` describes what you are **sending**, while `Accept: application/json` describes what you want to **receive**. ([MDN][16])

Common methods:

| Method  | Typical purpose                                       | Idempotent by defined semantics? |
| ------- | ----------------------------------------------------- | -------------------------------- |
| GET     | Retrieve a representation                             | Yes                              |
| HEAD    | Retrieve response metadata without response content   | Yes                              |
| POST    | Submit data for processing, often creating a resource | No                               |
| PUT     | Create or replace the state of a specified resource   | Yes                              |
| PATCH   | Apply a partial modification                          | Not necessarily                  |
| DELETE  | Remove the target resource                            | Yes                              |
| OPTIONS | Discover communication options                        | Yes                              |

POST or PATCH operations can be designed with additional idempotency controls. Conversely, choosing the PUT method does not automatically make incorrectly implemented application code idempotent. ([MDN][17])

---

**10. If security groups, WAF, and DDoS protection are enabled, will they protect against bot attacks?**

**They help, but their presence alone does not guarantee bot protection.**

| Control              | Primary contribution                                                    |
| -------------------- | ----------------------------------------------------------------------- |
| Security group       | Restricts network access by protocol, port, and source/destination.     |
| DDoS protection      | Helps maintain availability during supported denial-of-service attacks. |
| WAF rules            | Inspect and filter application requests.                                |
| Bot-management rules | Identify and control suspicious automated clients.                      |
| Application controls | Enforce account limits, transaction rules, and fraud protections.       |

A bot can make valid HTTPS requests on port 443. It might scrape content or attempt credential stuffing slowly enough to avoid looking like a volumetric DDoS attack.

For an AWS application, I would consider:

* AWS WAF Bot Control.
* Rate limits appropriate to login, search, and purchase endpoints.
* Challenges or CAPTCHA where suitable.
* Account-level abuse detection and stronger authentication.
* Exceptions for legitimate bots and API clients.
* Origin restrictions so clients cannot bypass the protected entry point.

Bot Control supports specialized bot inspection, but it must be configured and tested for the application’s traffic. ([docs.aws.amazon.com][18])

I would initially observe rule matches, measure false positives, and then enforce blocking or challenges. ([docs.aws.amazon.com][19])

---

**11. If I send the same Name and Location through PUT twice, what happens?**

The answer depends on the **target resource URI and the API implementation**.

Suppose both requests are:

```bash
curl --request PUT \
  https://api.example.com/users/42 \
  --header 'Content-Type: application/json' \
  --data '{"name":"Anita","location":"Pune"}'
```

For a correctly implemented PUT:

| Request                  | Resulting resource                                |
| ------------------------ | ------------------------------------------------- |
| First request            | User `42` has name Anita and location Pune.       |
| Identical second request | User `42` still has name Anita and location Pune. |

The second request should not create another logical user merely because the request was repeated.

If the API allows creation at that URI, the first request may return `201 Created`. Updating an existing resource may return `200 OK` or `204 No Content`. The responses need not be identical. ([MDN][20])

HTTP does not directly control database inserts. If the backend blindly inserts a new user for each PUT, the implementation does not honor the intended idempotent semantics for that resource.

---

**12. Why is PUT idempotent? What if the name stays the same but the location changes?**

An operation is **idempotent when repeating the same request has the same intended effect as executing it once**.

Consider this sequence:

| Request                        | Intended state afterward      |
| ------------------------------ | ----------------------------- |
| PUT `/users/42`: Anita, Pune   | User 42 is Anita, Pune        |
| Repeat the same PUT            | User 42 remains Anita, Pune   |
| PUT `/users/42`: Anita, Mumbai | User 42 becomes Anita, Mumbai |
| Repeat that new PUT            | User 42 remains Anita, Mumbai |

Changing the location creates a **different request**. Idempotency does not mean a resource can never change; it concerns repetitions of an identical operation.

The resource identifier matters:

* `/users/42` with a new location normally replaces the state of user 42.
* `/users/43` may identify another user, even if the name matches.
* Database uniqueness constraints and API validation may reject particular combinations.

**The name is not automatically the primary key.**

Idempotency also does not require identical status codes, nor does it prohibit separate audit-log entries for repeated requests. The guarantee concerns the intended resource effect. For concurrent updates, conditional requests such as `If-Match` can help prevent overwriting someone else’s changes. ([rfc-editor.org][21])

---

**13. What happens when I type `www.google.com` into a browser?**

For a fresh HTTPS connection, the general sequence is:

1. **Interpret the address.** The browser determines the URL and applicable HTTPS behavior.
2. **Check caches.** It may already have usable DNS information, content, or an existing connection.
3. **Resolve DNS.** Obtain an IP address for the hostname.
4. **Establish a connection.** For HTTP/1.1 or HTTP/2 over HTTPS, this normally includes a TCP handshake.
5. **Negotiate TLS.** Authenticate the server and establish encryption keys.
6. **Send the HTTP request.** Usually an initial GET for `/`.
7. **Receive the response.** The response includes a status, headers, and content.
8. **Render the page.** Parse HTML and CSS, execute JavaScript, fetch additional resources, and paint the page.

TCP connection establishment uses **SYN → SYN-ACK → ACK**. Page loading may trigger further requests to multiple hostnames. ([MDN][22])

One qualification: **HTTP/3 uses QUIC over UDP**, incorporating TLS 1.3, so it does not follow the same TCP-handshake sequence. ([rfc-editor.org][23])

This is the general browser flow; the exact path depends on caching, protocol selection, redirects, and connection reuse.

---

**14. What is an SSL/TLS handshake?**

A TLS handshake establishes the security parameters for an encrypted connection.

Its main purposes are to:

* Authenticate the server.
* Agree on supported cryptographic parameters.
* Establish shared traffic keys.
* Confirm that the handshake has not been tampered with.

“SSL handshake” is still commonly said, but modern HTTPS uses **TLS**. ([SSL Handshake][24])

A simplified **full, certificate-based TLS 1.3 handshake** works as follows:

1. **ClientHello:** The client offers supported versions, cipher suites, and key-share information.
2. **ServerHello:** The server selects parameters and supplies its key share. Both sides can derive handshake keys.
3. **Server authentication:** The server sends its certificate chain, a signature proving possession of the certificate’s private key, and a Finished message.
4. **Client verification:** The client validates the certificate chain, hostname, validity, signature, and handshake integrity.
5. **Client Finished:** The client confirms its view of the handshake.
6. **Protected application traffic:** HTTP data is exchanged using symmetric encryption keys.

The server’s private key is never sent to the client. With ephemeral Diffie–Hellman, the shared secret is derived independently rather than transmitted.

In mutual TLS, the server also requests and verifies a client certificate. Resumed connections can follow a different sequence. ([rfc-editor.org][25])

---

**15. How does DNS resolution work on a new system with no cache?**

DNS resolves names to records, including the IP addresses applications use to connect.

**An empty cache on your computer does not imply an empty cache at the recursive resolver.** Your ISP or enterprise resolver may already know the answer. ([AWS][26])

Assuming no relevant cached answer or delegation is available:

1. **The application requests resolution** of `www.google.com`.
2. **The client checks local resolution sources**, such as applicable caches and hosts-file entries.
3. **The client contacts its configured recursive resolver.** A browser may instead use a configured DNS-over-HTTPS resolver.
4. **The resolver queries a root server**, using its knowledge of root-server addresses.
5. **The root returns a referral** to `.com` name servers.
6. **The resolver queries a `.com` server.**
7. **The `.com` server returns a referral** to authoritative servers for `google.com`.
8. **The resolver queries an authoritative server** for the requested record.
9. **The resolver follows aliases if necessary**, obtains the final answer, and returns it to the client.
10. **Results are cached according to their TTLs**, and the application can begin connecting.

The root and TLD servers normally provide referrals; they do not need to hold the final address for every website. ([AWS][26])

Useful commands:

```bash
dig www.google.com A
dig www.google.com AAAA
dig +trace www.google.com
```

`A` requests IPv4 addresses; `AAAA` requests IPv6 addresses. `+trace` helps inspect delegation, although its path may differ from the browser’s resolver path.

---

**16. What is Kubernetes? Explain its architecture.**

**Kubernetes**, abbreviated **K8s**, manages containerized workloads by continuously reconciling actual conditions with a declared desired state.

Its architecture consists of a **control plane** and **worker nodes**.

| Component                                   | Responsibility                                                                |
| ------------------------------------------- | ----------------------------------------------------------------------------- |
| API server                                  | Receives Kubernetes API requests and provides the interface to cluster state. |
| etcd                                        | Stores Kubernetes configuration and state.                                    |
| Scheduler                                   | Selects nodes for unscheduled Pods.                                           |
| Controller manager                          | Runs reconciliation controllers.                                              |
| Cloud controller manager                    | Integrates with supported cloud infrastructure.                               |
| kubelet                                     | Ensures assigned Pods’ containers run on a node.                              |
| Container runtime                           | Runs containers, for example through containerd.                              |
| kube-proxy or an alternative implementation | Implements Service traffic forwarding.                                        |
| CNI plugin                                  | Provides Pod network connectivity.                                            |

A **Pod** is the smallest deployable unit and contains one or more containers sharing networking and specified storage. ([Kubernetes][27])

For a Deployment requesting three replicas:

1. The API server accepts the declaration.
2. The Deployment controller manages a ReplicaSet.
3. The ReplicaSet controller creates the required Pods.
4. The scheduler assigns Pods to suitable nodes.
5. Kubelets coordinate container startup.
6. Controllers continue reconciling failures and changes.

A Service provides stable access to selected Pods as individual Pods are replaced. The Deployment manages workload rollout and replica lifecycle; the Service provides discovery and traffic access. ([Kubernetes][28])

---

**17. Can I run a Pod on the master node itself?**

**Yes, on a self-managed cluster when scheduling and node configuration allow it.** The current term is **control-plane node**.

In kubeadm clusters, control-plane nodes normally have this taint:

```text
node-role.kubernetes.io/control-plane:NoSchedule
```

A matching toleration allows a Pod to be considered for that node:

```yaml
# Within the Pod spec:
tolerations:
  - key: node-role.kubernetes.io/control-plane
    operator: Exists
    effect: NoSchedule
```

A toleration **permits** scheduling but does not force placement. Add an appropriate node selector or affinity rule if placement must be restricted to control-plane nodes. ([Kubernetes][29])

For a single-node lab, you can remove the taint from a particular node:

```bash
kubectl taint node control-plane-1 \
  node-role.kubernetes.io/control-plane:NoSchedule-
```

Production application workloads are usually kept on workers to preserve control-plane resources and isolation. Control-plane components themselves often already run as static Pods. ([Kubernetes][30])

In managed services such as EKS, the provider-managed control plane is not available for scheduling your application Pods.

---

**18. Have you deployed any security applications on Kubernetes?**

Answer this according to your actual experience. If you have only completed a lab, describe it as a lab.

A practical example to explain is **Falco for runtime threat detection**.

**Example approach:**

* Deploy Falco through Helm, typically as a DaemonSet on supported Linux nodes.
* Verify kernel compatibility, required permissions, and host access.
* Detect suspicious runtime behavior, such as unexpected shells, sensitive-file access, or unusual process execution.
* Forward alerts to the security monitoring system.
* Tune rules against expected workload behavior.
* Test that alerts include enough context to investigate the affected workload.

Falco observes runtime activity; its core detection function does not automatically block every suspicious operation. ([Falco][31])

A basic test-cluster installation is:

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update

helm upgrade --install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace

kubectl get pods -n falco
```

For production, pin a reviewed chart version, configure values explicitly, and validate alert delivery and resource overhead. Falcosidekick can forward Falco events to external destinations. ([Falco][32])

Another example is **Kyverno**, which can enforce admission policies such as approved image registries or required security settings. This complements runtime detection by checking workloads before admission. ([Kyverno][33])

A credible interview answer explains **what you deployed, why, how you tested it, and how the team responded to findings**, using only work you actually performed.

[1]: https://docs.aws.amazon.com/waf/latest/developerguide/how-aws-waf-works.html?utm_source=chatgpt.com "How AWS WAF works - AWS WAF, AWS Firewall Manager, AWS Shield Advanced, and AWS Shield network security director"
[2]: https://owasp.org/www-project-top-ten/?utm_source=chatgpt.com "OWASP Top Ten Web Application Security Risks"
[3]: https://owasp.org/Top10/2025/0x00_2025-Introduction/?utm_source=chatgpt.com "Introduction"
[4]: https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html?utm_source=chatgpt.com "Authorization"
[5]: https://cheatsheetseries.owasp.org/cheatsheets/Injection_Prevention_Cheat_Sheet.html?utm_source=chatgpt.com "Injection Prevention"
[6]: https://developer.hashicorp.com/terraform/language/state?utm_source=chatgpt.com "State | Terraform"
[7]: https://developer.hashicorp.com/terraform/cli/state/recover?utm_source=chatgpt.com "Recover state from backup | Terraform"
[8]: https://developer.hashicorp.com/terraform/language/import?utm_source=chatgpt.com "Import resources overview | Terraform"
[9]: https://developer.hashicorp.com/terraform/language/files/dependency-lock?utm_source=chatgpt.com "Dependency Lock File (.terraform.lock.hcl) - Configuration Language | Terraform"
[10]: https://developer.hashicorp.com/terraform/language/style?utm_source=chatgpt.com "Style Guide - Configuration Language | Terraform"
[11]: https://developer.hashicorp.com/terraform/language/backend/s3?utm_source=chatgpt.com "Backend Type: s3 | Terraform"
[12]: https://developer.hashicorp.com/terraform/language/manage-sensitive-data?utm_source=chatgpt.com "Manage sensitive data in your configuration | Terraform"
[13]: https://docs.aws.amazon.com/vpc/latest/userguide/infrastructure-security.html?utm_source=chatgpt.com "Infrastructure security in Amazon VPC"
[14]: https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html?utm_source=chatgpt.com "Security best practices in IAM"
[15]: https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/security.html?utm_source=chatgpt.com "Security foundations"
[16]: https://developer.mozilla.org/en-US/docs/Glossary/Request_header?utm_source=chatgpt.com "Request header - Glossary"
[17]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Methods?utm_source=chatgpt.com "HTTP request methods - HTTP"
[18]: https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-bot.html?utm_source=chatgpt.com "AWS WAF Bot Control rule group - AWS WAF, AWS Firewall Manager, AWS Shield Advanced, and AWS Shield network security director"
[19]: https://docs.aws.amazon.com/waf/latest/developerguide/waf-bot-control-deploying.html?utm_source=chatgpt.com "Testing and deploying AWS WAF Bot Control - AWS WAF, AWS Firewall Manager, AWS Shield Advanced, and AWS Shield network security director"
[20]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Methods/PUT?utm_source=chatgpt.com "PUT request method - HTTP"
[21]: https://www.rfc-editor.org/rfc/rfc9110.html?utm_source=chatgpt.com "RFC 9110: HTTP Semantics"
[22]: https://developer.mozilla.org/en-US/docs/Web/Performance/Guides/How_browsers_work?utm_source=chatgpt.com "Populating the page: how browsers work - Performance"
[23]: https://www.rfc-editor.org/rfc/rfc9114.html?utm_source=chatgpt.com "RFC 9114: HTTP/3"
[24]: https://www.cloudflare.com/learning/ssl/what-happens-in-a-tls-handshake/?utm_source=chatgpt.com "What Happens in a TLS Handshake?"
[25]: https://www.rfc-editor.org/rfc/rfc8446.html?utm_source=chatgpt.com "www.rfc-editor.org"
[26]: https://aws.amazon.com/route53/what-is-dns/?utm_source=chatgpt.com "What is DNS? – Introduction to DNS"
[27]: https://kubernetes.io/docs/concepts/overview/components/?utm_source=chatgpt.com "Kubernetes Components"
[28]: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/?utm_source=chatgpt.com "Deployments"
[29]: https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/?utm_source=chatgpt.com "Taints and Tolerations"
[30]: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/?utm_source=chatgpt.com "Creating a cluster with kubeadm"
[31]: https://falco.org/about/faq/?utm_source=chatgpt.com "FAQs"
[32]: https://falco.org/docs/getting-started/falco-kubernetes-quickstart/?utm_source=chatgpt.com "Try Falco on Kubernetes"
[33]: https://kyverno.io/docs/introduction/?utm_source=chatgpt.com "Introduction"
