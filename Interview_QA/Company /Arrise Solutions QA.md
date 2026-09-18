# DevOps Interview Answers for 7 Years of Experience

> These responses are designed for a senior-level interview. Lead with the direct answer, then explain design choices, traffic flow, security, availability, failure handling, and operational trade-offs.

## 1. Explain three-tier architecture

Three-tier architecture separates an application into independently scalable and secured layers:

1. **Presentation tier:** Handles user-facing traffic. In AWS this can include Route 53, CloudFront, AWS WAF, an internet-facing ALB, and frontend services.
2. **Application tier:** Runs APIs and business logic on EC2, ECS, EKS, or Lambda. It normally runs in private subnets and accepts traffic only from the presentation tier.
3. **Data tier:** Stores persistent data in RDS, Aurora, DynamoDB, ElastiCache, or S3. Relational databases normally run in isolated/private subnets and accept traffic only from the application tier.

```text
Users
  |
Route 53 -> CloudFront -> WAF -> Public ALB
                                |
                       Application tier
                         private subnets
                                |
                     RDS/Aurora/ElastiCache
                       isolated subnets
```

For production, deploy across at least two Availability Zones, use Auto Scaling, health checks, TLS, least-privilege security groups, encryption, backups, monitoring, and controlled secrets. A public load balancer can route to targets in private subnets through the VPC local route.

## 2. Difference between ALB and NLB in depth

### Application Load Balancer

- Operates primarily at **Layer 7**.
- Supports HTTP, HTTPS, HTTP/2, WebSocket, and gRPC use cases.
- Can route using host name, URL path, HTTP headers, methods, query strings, and source IP conditions.
- Supports redirects, fixed responses, authentication integrations, AWS WAF, and target-group stickiness.
- Terminates TLS and makes routing decisions using application-level request data.
- Best for websites, REST APIs, microservices, ingress, and content-based routing.

### Network Load Balancer

- Operates primarily at **Layer 4**.
- Supports TCP, TLS, UDP, and TCP_UDP listeners.
- Routes connections based on protocol, port, and destination rather than URL content.
- Designed for very high throughput and low latency.
- Can provide static IP addresses per enabled Availability Zone and can use Elastic IP addresses for internet-facing IPv4 configurations.
- Preserves client source IP in supported target and protocol configurations.
- Best for non-HTTP protocols, high connection rates, fixed-IP allow-listing, TLS pass-through, gaming, IoT, and network appliances.

### Senior-level selection rule

Choose **ALB** when the routing decision needs application context. Choose **NLB** when the requirement is transport-level performance, static addressing, source-IP preservation, UDP, or non-HTTP protocols. Both can be internet-facing or internal, use target groups and health checks, and span enabled Availability Zones. NLB also supports traffic arriving through VPC peering, AWS VPN, Direct Connect, and supported third-party VPN paths.

## 3. Connect an AWS VPC to an IBM Cloud VPC

The common approach is a **site-to-site IPsec VPN**. Ensure that the two VPC CIDR ranges do not overlap.

### AWS side

1. Create a Customer Gateway using the public IP of the IBM Cloud VPN gateway.
2. Create and attach a Virtual Private Gateway to the AWS VPC, or use AWS Transit Gateway for a scalable multi-VPC design.
3. Create an AWS Site-to-Site VPN connection.
4. Configure static routes or BGP, depending on the IBM VPN mode and design.
5. Add IBM VPC CIDRs to AWS route tables through the VGW or TGW.
6. Permit traffic in security groups and network ACLs.

### IBM Cloud side

1. Create VPN for VPC in an IBM subnet.
2. Define the AWS peer public IPs.
3. Configure route-based or policy-based tunnels.
4. Match IKE/IPsec parameters, pre-shared keys, encryption, integrity, DH/PFS, and lifetimes.
5. Add AWS CIDRs to IBM routing and security controls.

### Validation

```bash
ping <remote-private-ip>
traceroute <remote-private-ip>
nc -vz <remote-private-ip> 443
```

Check both tunnels, routes, asymmetric routing, MTU/MSS, firewall rules, and overlapping CIDRs. IBM documents a specific AWS peer configuration and notes that AWS requires PFS in Phase 2. For higher bandwidth or stricter private-connectivity requirements, evaluate AWS Direct Connect plus IBM Cloud Direct Link through a compatible connectivity provider.

## 4. Public subnet vs private subnet

A subnet is considered public or private because of its **routing**, not its name.

- **Public subnet:** Its route table has `0.0.0.0/0` to an Internet Gateway. For IPv4 internet access, an instance also needs a public IPv4 or Elastic IP and permissive security controls.
- **Private subnet:** It has no direct default route to an Internet Gateway. Workloads normally use private IPs. Outbound internet can go through a NAT gateway, while private AWS-service access can use VPC endpoints.
- **Isolated subnet:** Has no default internet route through an IGW or NAT. Often used for databases.

## 5. Connect a private subnet to the internet

For IPv4 outbound internet access:

1. Attach an Internet Gateway to the VPC.
2. Create a **public NAT gateway** in a public subnet.
3. Associate an Elastic IP with the NAT gateway.
4. Add `0.0.0.0/0 -> NAT gateway` to the private subnet route table.
5. Ensure the NAT gateway's public subnet has `0.0.0.0/0 -> Internet Gateway`.
6. Allow required egress in security groups and NACLs.

```text
Private EC2 -> Private route table -> NAT gateway
           -> Public route table -> Internet Gateway -> Internet
```

A NAT gateway provides outbound initiation, not unsolicited inbound internet connectivity. For S3 and DynamoDB, prefer gateway endpoints where practical. For IPv6 outbound-only internet access, use an egress-only Internet Gateway.

## 6. Does a NAT gateway run in a public or private subnet?

- A **public NAT gateway** must be placed in a public subnet, use an Elastic IP, and reach an Internet Gateway.
- A **private NAT gateway** can translate traffic for private connectivity to other VPCs or on-premises networks through a Transit Gateway or Virtual Private Gateway. It does not provide internet access through an Internet Gateway.
- For resilience, use a public NAT gateway per Availability Zone and route private subnets to the NAT in the same AZ.

## 7. Kubernetes architecture

A Kubernetes cluster contains a **control plane** and **worker nodes**.

### Control plane

- `kube-apiserver`: Front door for the Kubernetes API and cluster operations.
- `etcd`: Consistent key-value store containing cluster state.
- `kube-scheduler`: Assigns unscheduled Pods to suitable nodes.
- `kube-controller-manager`: Runs reconciliation controllers such as Deployment, node, and endpoint controllers.
- `cloud-controller-manager`: Integrates supported cloud resources such as nodes, routes, and load balancers.

### Worker node

- `kubelet`: Ensures the Pods assigned to its node are running as specified.
- Container runtime: Runs containers through CRI, commonly containerd or CRI-O.
- `kube-proxy`: Implements Service traffic forwarding where the platform uses it.
- CNI plugin: Provides Pod networking.
- Pods: Smallest deployable Kubernetes workload units.

```text
kubectl/controllers -> API server -> etcd
                           |
                       scheduler
                           |
                       Worker node
             kubelet + runtime + CNI + kube-proxy
                           |
                          Pods
```

## 8. CoreDNS in Kubernetes

CoreDNS is the default Kubernetes cluster DNS implementation. Pods normally send DNS queries to the `kube-dns` Service IP, which forwards them to CoreDNS Pods.

It provides service discovery for names such as:

```text
service-name.namespace.svc.cluster.local
```

CoreDNS watches Kubernetes resources through the API and answers Service and Pod-related records. External lookups are forwarded to configured upstream resolvers. Configuration is stored in the `Corefile`, usually in a ConfigMap.

Troubleshooting:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl get svc -n kube-system kube-dns
kubectl get configmap -n kube-system coredns -o yaml
kubectl logs -n kube-system -l k8s-app=kube-dns
kubectl exec -it <pod> -- nslookup kubernetes.default.svc.cluster.local
```

## 9. Purpose of CNI

CNI means **Container Network Interface**. A compatible CNI plugin implements Pod networking by:

- Creating or attaching interfaces in Pod network namespaces
- Allocating and releasing Pod IP addresses
- Configuring routes
- Connecting Pods across nodes
- Cleaning up networking when Pods are deleted
- Optionally enforcing NetworkPolicy

Examples include Amazon VPC CNI, Calico, Cilium, Flannel, and Antrea. Kubernetes defines the desired network model; the CNI implementation supplies the data path.

## 10. How kube-proxy communicates with nodes

The better explanation is that **kube-proxy does not normally communicate node-to-node as a central proxy**. An instance runs on each node, usually as a DaemonSet.

1. Each kube-proxy watches Services and EndpointSlices through the API server.
2. It programs local packet-forwarding state, traditionally iptables or IPVS depending on the environment.
3. Traffic sent to a Service virtual IP is translated or forwarded to a healthy backend Pod endpoint.
4. The CNI and underlying network deliver packets to local or remote Pod IPs.

Therefore, kube-proxy handles Service-level forwarding rules, while the CNI provides Pod reachability. Some eBPF CNIs can replace kube-proxy functionality.

## 11. Purpose of the Kubernetes scheduler

The scheduler watches for Pods whose `spec.nodeName` is not assigned, filters feasible nodes, scores them, and binds each Pod to the selected node.

It considers:

- CPU, memory, and extended resource requests
- Node selectors and node affinity
- Pod affinity and anti-affinity
- Taints and tolerations
- Topology-spread constraints
- Volume topology and binding
- Scheduling policies and plugin configuration

The scheduler only selects a node. The kubelet on that node asks the runtime to create the Pod.

## 12. Layers in Docker

A Docker/OCI image consists of immutable filesystem layers. Filesystem-changing Dockerfile instructions, such as `RUN`, `COPY`, and `ADD`, create layers. Metadata-only instructions do not necessarily create filesystem layers.

When a container starts, Docker adds a thin writable layer over the read-only image layers using copy-on-write behavior.

```text
Container writable layer
Application layer
Dependency layer
Base image layer
```

Layers are content-addressed, reusable, cacheable, and shareable across images. Removing a secret in a later layer does not remove it from an earlier layer, so secrets must never be copied into an image.

## 13. VM vs Docker container

### Virtual machine

- Virtualizes hardware through a hypervisor.
- Includes a complete guest OS and its own kernel.
- Usually has a larger image and slower startup.
- Provides a stronger default isolation boundary.
- Can run a guest OS with a kernel different from the host.

### Container

- Isolates processes at the operating-system level.
- Shares the host kernel.
- Packages the application and user-space dependencies.
- Usually starts faster and has lower overhead.
- Requires careful hardening because the kernel is shared.

Use VMs for strong isolation, different OS kernels, or legacy requirements. Use containers for portable, rapidly scaled application workloads. They are often combined by running containers inside VMs.

## 14. Docker storage drivers

Do not confuse **storage drivers** with **persistent mounts**.

Classic Linux storage drivers include:

- `overlay2`: Broadly compatible and commonly used classic driver.
- `fuse-overlayfs`: Relevant to some rootless configurations.
- `btrfs`: Snapshot-capable and requires a suitable Btrfs backing filesystem.
- `zfs`: Uses ZFS features and requires ZFS configuration.
- `vfs`: No copy-on-write benefit; mainly for testing or compatibility.

Fresh Docker Engine 29 installations use the containerd image store by default rather than the classic model. Persistent data should normally use **volumes**, bind mounts, or `tmpfs`, rather than the container writable layer.

```bash
docker info | grep -iE 'storage|driver'
```

## 15. Docker network types

- **bridge:** Default single-host network. User-defined bridges provide container DNS by name and better isolation than putting everything on the default bridge.
- **host:** Container shares the host network namespace. There is no separate container IP or normal port publishing boundary.
- **none:** Container receives only loopback networking. Use for complete network isolation.
- **overlay:** Connects containers or services across multiple Docker hosts, commonly with Swarm.
- **macvlan:** Gives a container a MAC address and makes it appear directly on the physical network.
- **ipvlan:** Provides direct underlay integration while sharing or controlling Layer 2 behavior differently from macvlan.
- **Third-party plugins:** Integrate external SDN/networking systems.

```bash
docker network ls
docker network inspect <network>
docker network create --driver bridge isolated-app
```

## 16. Does Docker have its own kernel?

No. A Linux container normally uses the **host Linux kernel**. The image contains user-space files, libraries, binaries, and configuration, but not an independently booted kernel.

That is why a Linux container requires a Linux kernel. Docker Desktop runs Linux containers inside a managed Linux VM on non-Linux desktops. Kernel sharing improves density and startup speed but makes kernel patching and isolation controls important.

## 17. Cgroups and namespaces in Docker

The likely interview question is **cgroups and namespaces**.

### Namespaces provide isolation

- PID: Process IDs
- NET: Network interfaces, routes, and ports
- MNT: Mount points and filesystem view
- UTS: Hostname and domain name
- IPC: Shared memory and message queues
- USER: User and group ID mapping
- Cgroup namespace: View of cgroup hierarchy

### Cgroups provide accounting and limits

They control and measure resources such as CPU, memory, PIDs, and I/O.

```bash
docker run --memory=512m --cpus=1.5 --pids-limit=200 nginx
```

Namespaces answer “what can this process see?” Cgroups answer “how much can this process consume?”

## 18. Which network isolates communication between two containers?

Use separate **user-defined bridge networks**, or use the `none` driver for complete network removal.

Containers attached only to different bridge networks cannot directly communicate unless routing, port publishing, or a multi-homed container explicitly connects them.

```bash
docker network create frontend
docker network create backend

docker run -d --name web --network frontend nginx
docker run -d --name db --network backend postgres
```

For selective communication, connect only an application container to both networks. Do not rely on network isolation as the only security control.

## 19. How to check Linux processes

```bash
ps -ef
ps aux
pstree -p
top
htop
pgrep -af nginx
pidof nginx
systemctl status nginx
```

For a specific PID:

```bash
ps -p <PID> -o pid,ppid,user,stat,%cpu,%mem,etime,cmd
cat /proc/<PID>/status
ls -l /proc/<PID>/fd
lsof -p <PID>
```

Use `strace -p <PID>` carefully when diagnosing system calls because attaching can affect a busy process.

## 20. Linux boot process

1. **Firmware:** BIOS or UEFI initializes hardware and selects a boot device.
2. **Bootloader:** GRUB loads the selected kernel and initramfs.
3. **Kernel:** Initializes CPU, memory, drivers, scheduling, and mounts the temporary root filesystem.
4. **initramfs:** Loads drivers and discovers/unlocks the real root filesystem.
5. **PID 1:** Usually `systemd`, which mounts filesystems, starts targets, services, sockets, and timers.
6. **Login/graphical target:** The machine becomes available to users and applications.

Troubleshooting:

```bash
journalctl -b
journalctl -b -1
systemd-analyze
systemd-analyze blame
cat /proc/cmdline
```

## 21. How to check Linux machine load

```bash
uptime
cat /proc/loadavg
top
vmstat 1
mpstat -P ALL 1
iostat -xz 1
sar -q 1 5
```

Load average represents runnable tasks plus tasks in uninterruptible sleep, averaged over 1, 5, and 15 minutes. Compare it with logical CPU count, but do not conclude CPU saturation from load alone. Check `%idle`, run queue, I/O wait, memory pressure, swap, disk latency, and throttling.

```bash
nproc
free -m
vmstat 1
iostat -xz 1
```

## 22. Stages restarted during a Linux reboot

A normal reboot broadly performs:

1. Userspace receives the shutdown/reboot request.
2. `systemd` stops services in dependency order.
3. Filesystems are synchronized and unmounted or remounted read-only.
4. Swap and devices are deactivated as appropriate.
5. The current kernel invokes the reboot operation.
6. Firmware starts again.
7. Bootloader loads the kernel and initramfs.
8. Kernel reinitializes hardware and memory management.
9. PID 1 starts targets, services, networking, and applications.

A reboot restarts the kernel and all userspace processes. It does not reinstall the OS or normally erase persistent storage.

## 23. Where are kernel logs stored?

There is no single universal directory.

- Current kernel ring buffer: `dmesg`
- systemd journal: `journalctl -k`
- Current boot: `journalctl -k -b`
- Previous boot: `journalctl -k -b -1`
- Traditional distributions may write `/var/log/kern.log` or relevant messages to `/var/log/messages`.
- Persistent journal data is normally under `/var/log/journal` when persistent storage is enabled; otherwise it may exist under `/run/log/journal`.

```bash
dmesg -T | tail
journalctl -k --since today
```

## 24. How to kill a running process

Find and terminate it gracefully first:

```bash
pgrep -af application
kill -TERM <PID>
```

If it does not exit after an appropriate waiting period:

```bash
kill -KILL <PID>
```

Other options:

```bash
pkill -TERM -f 'application-pattern'
systemctl stop service-name
```

`SIGTERM` allows cleanup. `SIGKILL` cannot be caught or handled and should be the last resort. A zombie process cannot be killed because it has already exited; its parent must reap it or be restarted.

## 25. Linux process states

If “states of a Linux machine” means process states, common `ps` state codes are:

- `R`: Running or runnable
- `S`: Interruptible sleep
- `D`: Uninterruptible sleep, often waiting on I/O
- `T`: Stopped or traced
- `Z`: Zombie
- `I`: Idle kernel thread on supported displays
- `X`: Dead, rarely observed

```bash
ps -eo pid,ppid,stat,wchan:25,cmd
```

If the interviewer means systemd operating states, examples include `running`, `degraded`, `maintenance`, `stopping`, and `offline`:

```bash
systemctl is-system-running
```

## 26. What happens when you type google.com in a browser?

1. The browser normalizes the URL and applies policy such as HTTPS upgrade or HSTS.
2. It checks browser/OS caches and resolves DNS, possibly through a recursive resolver or encrypted DNS.
3. DNS returns an address suitable for the service or CDN edge.
4. The host determines the route, resolves the local next-hop MAC using ARP for IPv4 or Neighbor Discovery for IPv6, and sends packets through the gateway.
5. A connection is established using TCP plus TLS for HTTPS, or QUIC for HTTP/3 when available.
6. During TLS, the client validates the server certificate, hostname, trust chain, validity period, and policy.
7. The browser sends an HTTP request with headers and cookies allowed by policy.
8. CDN, load balancer, reverse proxy, and application services process the request.
9. The response is decompressed and parsed; the browser builds DOM and CSSOM, executes JavaScript, lays out, paints, and composites the page.
10. Additional resources are fetched, often over reused or multiplexed connections, while browser security policies and caching apply.

## 27. Components displayed by `top`

The default screen commonly shows:

1. **System summary:** Time, uptime, logged-in users, and load averages.
2. **Task summary:** Total, running, sleeping, stopped, and zombie tasks.
3. **CPU states:** User, system, nice, idle, I/O wait, hardware/software interrupts, and steal time.
4. **Memory:** Total, free, used, and buffer/cache.
5. **Swap:** Total, free, used, and available memory estimate.
6. **Process list:** PID, user, priority, nice value, virtual/resident/shared memory, state, CPU%, memory%, runtime, and command.

Useful keys:

```text
P = sort by CPU      M = sort by memory
1 = per-CPU view     H = threads
k = send signal      r = renice
c = full command     q = quit
```

## 28. CloudFront

Amazon CloudFront is AWS's content delivery network. It uses edge locations to cache and deliver content closer to users.

Typical flow:

```text
Viewer -> CloudFront edge -> Cache hit: return object
                          -> Cache miss: request origin -> cache -> viewer
```

Origins can include S3, ALB, API Gateway, Media services, or custom HTTP servers. Important features include TLS, cache policies, origin request policies, signed URLs/cookies, Origin Access Control for S3, AWS WAF integration, geo restrictions, invalidations, access logs, and Lambda@Edge or CloudFront Functions.

Use CloudFront for static and dynamic acceleration, global delivery, DDoS-resilient edge entry, reduced origin load, and controlled private-origin access. Cache keys and TTLs must be designed carefully to avoid serving user-specific content incorrectly.

## 29. How certificates work between a client and remote machine

In normal server-authenticated TLS:

1. The client connects and sends a ClientHello containing supported TLS versions, cipher capabilities, key-share data, and the requested server name.
2. The server responds with selected parameters and its certificate chain.
3. The client validates the certificate chain against its trusted CA store, including hostname, validity, signature, and policy checks.
4. The peers derive symmetric session keys using an authenticated key exchange.
5. Finished messages prove possession of the negotiated secrets.
6. Application data is encrypted and integrity-protected with symmetric keys.

In **mutual TLS**, the server also requests a client certificate. The client presents its chain and proves possession of the corresponding private key. This provides bidirectional identity authentication.

The private key never travels over the network. A certificate binds a public key to an identity, while trust comes from a configured CA chain or explicit trust anchor.

## 30. How SSL/TLS certificates work

“SSL certificate” is the common term, but modern systems use **TLS**. SSL versions are obsolete.

A certificate contains a subject or identity, public key, issuer, serial number, validity period, extensions, and the issuer's digital signature. For web TLS, the requested hostname must match the certificate's Subject Alternative Name.

### TLS 1.3 summary

1. ClientHello proposes algorithms and sends a key share.
2. ServerHello selects parameters and contributes its key share.
3. The server sends its certificate and proof that it controls the private key.
4. The client validates the certificate chain and identity.
5. Both sides derive symmetric traffic keys.
6. Encrypted application data begins.

Public-key cryptography authenticates and establishes secrets; symmetric cryptography protects bulk traffic efficiently. Certificate renewal, private-key protection, revocation strategy, hostname coverage, intermediate certificates, clock accuracy, and trust-store management are critical operational concerns.

Useful commands:

```bash
openssl s_client -connect example.com:443 -servername example.com -showcerts
openssl x509 -in certificate.pem -noout -text
curl -vI https://example.com
```

# Official References

- AWS Prescriptive Guidance, load balancer subnets and routing: https://docs.aws.amazon.com/prescriptive-guidance/latest/load-balancer-stickiness/subnets-routing.html
- AWS Network Load Balancers: https://docs.aws.amazon.com/elasticloadbalancing/latest/network/network-load-balancers.html
- AWS NAT gateways: https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html
- AWS private subnets and NAT: https://docs.aws.amazon.com/vpc/latest/userguide/vpc-example-private-subnets-nat.html
- IBM Cloud, connecting to an AWS peer: https://cloud.ibm.com/docs/vpc?topic=vpc-aws-config
- IBM Cloud site-to-site VPN gateways: https://cloud.ibm.com/docs/vpc?topic=vpc-using-vpn
- Kubernetes cluster architecture: https://kubernetes.io/docs/concepts/architecture/
- Kubernetes CoreDNS: https://kubernetes.io/docs/tasks/administer-cluster/coredns/
- Docker storage drivers and image layers: https://docs.docker.com/engine/storage/drivers/
- Docker storage: https://docs.docker.com/engine/storage/
- Docker storage-driver selection: https://docs.docker.com/engine/storage/drivers/select-storage-driver/
- Linux CPU load: https://docs.kernel.org/admin-guide/cpu-load.html
- Linux TLS: https://docs.kernel.org/networking/tls.html
