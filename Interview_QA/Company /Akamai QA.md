Below are concise, interview-ready answers with commands/code and expected outputs.

### 1) IPv4 vs IPv6

IPv4 uses **32-bit addresses**, while IPv6 uses **128-bit addresses**.

| IPv4                         | IPv6                                                |
| ---------------------------- | --------------------------------------------------- |
| 32-bit                       | 128-bit                                             |
| ~4.3 billion addresses       | Extremely large address space                       |
| Decimal notation             | Hexadecimal notation                                |
| Example: `192.168.1.10`      | Example: `2001:db8::1`                              |
| NAT commonly used            | NAT generally not required for address conservation |
| ARP                          | Neighbor Discovery Protocol                         |
| Broadcast supported          | No broadcast; uses multicast                        |
| Header more complex/variable | Simplified fixed base header                        |

Example IPv4:

```text
10.20.30.40
```

Example IPv6:

```text
2001:db8:abcd:12::1
```

A good interview answer is: **IPv4 is a 32-bit addressing protocol with limited address space, while IPv6 is 128-bit and was introduced mainly to provide a much larger address space and improve IP networking.**

---

### 2) IPv4 address starts from?

The complete IPv4 address space ranges from:

```text
0.0.0.0
```

to:

```text
255.255.255.255
```

Each octet can have a value between:

```text
0 - 255
```

For example:

```text
192.168.10.25
```

Historically IPv4 was discussed using Class A, B and C ranges, but modern networking primarily uses **CIDR** rather than classful addressing.

---

### 3) Python script to validate an IPv4 address

The safest approach is Python's built-in `ipaddress` module.

```python
import ipaddress

def is_valid_ipv4(ip):
    try:
        ipaddress.IPv4Address(ip)
        return True
    except ipaddress.AddressValueError:
        return False


ip = input("Enter IPv4 address: ")

if is_valid_ipv4(ip):
    print("Valid IPv4 address")
else:
    print("Invalid IPv4 address")
```

Input:

```text
192.168.1.10
```

Output:

```text
Valid IPv4 address
```

Input:

```text
192.168.1.300
```

Output:

```text
Invalid IPv4 address
```

The reason is that an IPv4 address has four octets, and every octet must be between `0` and `255`.

---

### 4) Unix command to find all files larger than 1 GB

From the current directory:

```bash
find . -type f -size +1G
```

Across the entire filesystem:

```bash
find / -type f -size +1G 2>/dev/null
```

To display their sizes too:

```bash
find / -type f -size +1G -exec ls -lh {} \; 2>/dev/null
```

Example output:

```text
-rw-r--r-- 1 root root 2.4G Sep 14 10:00 /var/log/application.log
-rw-r--r-- 1 root root 1.8G Sep 14 11:00 /data/backup.tar
```

Here:

```text
-type f    = files only
-size +1G  = larger than 1 GB
```

---

### 5) Find `ERROR` in a text file, case-insensitive

Use:

```bash
grep -i "ERROR" application.txt
```

`-i` means ignore case.

So it matches:

```text
ERROR
Error
error
eRrOr
```

To also display line numbers:

```bash
grep -in "ERROR" application.txt
```

Example:

```text
12:ERROR Database connection failed
25:Error connecting to API
41:error timeout occurred
```

For all `.txt` files recursively:

```bash
grep -Rin "ERROR" --include="*.txt" .
```

---

### 6) If you're logging into a Linux machine for the first time, what is the process?

Normally I connect using SSH:

```bash
ssh username@server-ip
```

For example:

```bash
ssh ec2-user@10.10.1.20
```

With a private key:

```bash
ssh -i server.pem ec2-user@10.10.1.20
```

During the first connection, SSH may display:

```text
The authenticity of host '10.10.1.20' can't be established.
Are you sure you want to continue connecting?
```

After verifying the host fingerprint, accept it:

```text
yes
```

The host key is then stored in:

```text
~/.ssh/known_hosts
```

Once logged in, I normally verify:

```bash
whoami
hostname
hostname -I
pwd
df -h
free -m
uptime
```

For cloud servers I also confirm I'm connecting through the approved path, such as a bastion host, VPN, AWS Systems Manager, or private networking rather than unnecessarily exposing SSH to the internet.

---

### 7) If you're unable to access a Linux machine, what will you do?

I troubleshoot layer by layer.

First verify DNS/IP:

```bash
nslookup server.example.com
```

Then test connectivity:

```bash
ping <server-ip>
```

But ping being blocked does **not** necessarily mean the server is unavailable.

Check SSH port:

```bash
nc -zv <server-ip> 22
```

Then enable verbose SSH logging:

```bash
ssh -vvv user@server-ip
```

I check:

```text
DNS resolution
        ↓
Routing/VPN connectivity
        ↓
Security Group / NACL
        ↓
Linux firewall
        ↓
Port 22
        ↓
sshd service
        ↓
Username / SSH key
        ↓
File permissions
        ↓
OS/resource problems
```

For AWS specifically, I would check Security Groups, NACLs, route tables, instance state, subnet routing and whether the instance has the required public/private connectivity.

If normal SSH is unavailable, depending on the environment I could use **AWS Systems Manager Session Manager, EC2 Serial Console, or another authorized out-of-band management method**.

---

### 8) What are `5/2` and `5//2` in Python?

```python
print(5 / 2)
```

Output:

```text
2.5
```

`/` performs normal division.

Now:

```python
print(5 // 2)
```

Output:

```text
2
```

`//` is **floor division**.

Another example:

```python
7 // 2
```

returns:

```text
3
```

So:

```text
/  -> division
// -> floor division
```

---

### 9) `a = [0]`, `b = {0}` — what are `a[0]` and `b[0]`?

```python
a = [0]
b = {0}
```

`a` is a **list**:

```python
print(a[0])
```

Output:

```text
0
```

Lists support indexing.

But `b` is a **set**:

```python
print(b[0])
```

produces an error:

```text
TypeError: 'set' object is not subscriptable
```

Sets don't support positional indexing because they are not sequence types.

So:

```text
a[0] -> 0
b[0] -> TypeError
```

---

### 10) How do you check firewall protection in Linux?

It depends on the Linux distribution and firewall system.

For `firewalld`:

```bash
systemctl status firewalld
```

Check active rules:

```bash
firewall-cmd --list-all
```

For Ubuntu `ufw`:

```bash
sudo ufw status verbose
```

For modern `nftables`:

```bash
sudo nft list ruleset
```

For systems still using iptables:

```bash
sudo iptables -L -n -v
```

I would also check listening ports:

```bash
ss -tulpn
```

In AWS, Linux firewall rules are only one layer. I would also check:

```text
Security Groups
NACLs
Route Tables
AWS Network Firewall, if used
```

---

### 11) Explain the OSI model

OSI has **7 layers**:

```text
7  Application
6  Presentation
5  Session
4  Transport
3  Network
2  Data Link
1  Physical
```

Examples:

| Layer          | Purpose                        | Examples                                    |
| -------------- | ------------------------------ | ------------------------------------------- |
| 7 Application  | User/application communication | HTTP, HTTPS, DNS, SSH                       |
| 6 Presentation | Encoding/encryption            | TLS-related presentation concepts, encoding |
| 5 Session      | Session management             | Session establishment/management            |
| 4 Transport    | End-to-end transport           | TCP, UDP                                    |
| 3 Network      | Routing                        | IP, routers                                 |
| 2 Data Link    | Frames/MAC communication       | Ethernet, MAC                               |
| 1 Physical     | Physical transmission          | Cables, fiber, radio                        |

A useful troubleshooting approach is to work upward:

```text
Physical connectivity
       ↓
Network/IP
       ↓
TCP/UDP
       ↓
Application
```

For example, if HTTPS isn't working:

```text
Can I reach the IP?
Can I reach port 443?
Is TLS working?
Is the application responding?
```

---

### 12) Difference between Public and Private Hosted Zones in Route 53

A **Public Hosted Zone** contains DNS records that can be resolved from the public internet.

Example:

```text
www.example.com -> Public ALB
```

Architecture:

```text
Internet User
     |
Route 53 Public Hosted Zone
     |
Public ALB
```

A **Private Hosted Zone** is used for DNS resolution inside associated VPCs.

For example:

```text
database.internal.example.com
```

may resolve to:

```text
10.10.20.15
```

Architecture:

```text
EC2 / EKS inside VPC
       |
Route 53 Private Hosted Zone
       |
database.internal.example.com
       |
10.10.20.15
```

The interview summary is:

> **Public hosted zones are for internet-resolvable DNS records. Private hosted zones provide DNS resolution within associated VPCs and are commonly used for internal services.**

---

### 13) How do you create a Kustomize file?

Kustomize uses a file named:

```text
kustomization.yaml
```

Suppose I have:

```text
deployment.yaml
service.yaml
```

Create:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
```

Directory:

```text
app/
├── deployment.yaml
├── service.yaml
└── kustomization.yaml
```

Preview the generated manifests:

```bash
kubectl kustomize .
```

Deploy:

```bash
kubectl apply -k .
```

For multiple environments, I commonly use:

```text
k8s/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
│
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    └── prod/
        └── kustomization.yaml
```

For example, production can increase replicas without modifying the base manifest.

That is one of the main advantages of Kustomize: **reuse common Kubernetes YAML and apply environment-specific customization without duplicating all manifests.**

---

### 14) Difference between Docker `CMD` and `ENTRYPOINT`

Both define what runs when a container starts, but they have different purposes.

`ENTRYPOINT` defines the **main executable**.

Example:

```dockerfile
ENTRYPOINT ["python", "app.py"]
```

The container is primarily designed to run:

```text
python app.py
```

`CMD` commonly provides a **default command or default arguments** that can easily be overridden.

Example:

```dockerfile
CMD ["python", "app.py"]
```

Running:

```bash
docker run myimage
```

executes:

```text
python app.py
```

But:

```bash
docker run myimage bash
```

replaces the `CMD` with:

```text
bash
```

A very common pattern is combining them:

```dockerfile
ENTRYPOINT ["ping"]
CMD ["google.com"]
```

Running:

```bash
docker run myimage
```

effectively executes:

```text
ping google.com
```

Running:

```bash
docker run myimage example.com
```

effectively executes:

```text
ping example.com
```

So the interview answer is:

> **ENTRYPOINT defines the container's primary executable and is harder to override accidentally. CMD defines the default command or arguments and is easily overridden at runtime. They can also be combined, where ENTRYPOINT provides the executable and CMD provides its default arguments.**
