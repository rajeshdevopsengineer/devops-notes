**1. Reverse proxy and forward proxy**

Both proxies forward requests, but they represent different sides of the connection.

| Aspect | Forward proxy | Reverse proxy |
|---|---|---|
| Represents | Clients accessing other services | Servers receiving client requests |
| Typical position | Between internal clients and external services | In front of application servers |
| Main purposes | Outbound access control, filtering, caching, auditing | Load balancing, TLS termination, routing, caching |
| Examples | Corporate HTTP proxy, Squid | NGINX, HAProxy, application load balancer |
| Client configuration | Usually configured explicitly or through organizational settings | Clients normally access the application URL |

**Forward proxy example:** A build agent must download dependencies through an approved corporate proxy.

```bash
curl --proxy http://proxy.example.com:3128 \
  https://downloads.example.net/package.zip
```

**Reverse proxy example:** Users access `https://app.example.com`, and NGINX forwards their requests to application servers on internal addresses.

NGINX implements this forwarding through configuration such as `proxy_pass`. [nginx.org](https://nginx.org/en/docs/beginners_guide.html?utm_source=chatgpt.com)

For a release engineer, proxies commonly affect artifact downloads, registry access, webhook connectivity, TLS configuration, and application routing. When troubleshooting, check the proxy’s connection logs, DNS resolution, upstream response, and timeout settings.

**2. What are ulimits?**

`ulimit` is a shell builtin that displays or changes resource limits for the current shell and its child processes.

Examples:

```bash
ulimit -a       # Display current limits
ulimit -Sn     # Soft limit for open file descriptors
ulimit -Hn     # Hard limit for open file descriptors
ulimit -u      # Process limit
ulimit -s      # Stack-size limit
ulimit -c      # Core-dump size limit
```

| Limit | Meaning |
|---|---|
| Soft limit | The limit currently enforced |
| Hard limit | The ceiling to which an unprivileged process can raise its soft limit |

For example:

```bash
ulimit -Sn 65535
```

This raises the shell’s soft file-descriptor limit, provided its hard limit permits that value.

Use `-S` explicitly when changing only the soft limit. In Bash, setting a limit without `-S` or `-H` changes both. [gnu.org](https://www.gnu.org/s/bash/manual/html_node/Bash-Builtins.html?utm_source=chatgpt.com)

Limits are inherited when processes start. Changing your interactive shell’s limit does not change an already-running Jenkins agent or application service.

Also, file descriptors include **sockets, pipes, and other handles**, as well as ordinary files.

**3. Your script reports “Too many open files.” How do you fix it?**

I first determine whether the application has a descriptor leak, excessive concurrency, or an appropriately sized workload hitting an insufficient limit.

Linux distinguishes two important errors:

- **`EMFILE`:** The process reached its file-descriptor limit.
- **`ENFILE`:** The system reached its limit for open file handles. [Linux manual page](https://man7.org/linux/man-pages/man2/open.2.html?utm_source=chatgpt.com)

**Inspect the affected process:**

```bash
pid=12345

grep 'Max open files' "/proc/$pid/limits"

ls -1 "/proc/$pid/fd" | wc -l

lsof -p "$pid"
```

The affected process’s limits are more useful than the limits of a different login shell.

**Check system-wide usage:**

```bash
cat /proc/sys/fs/file-nr

sysctl fs.file-max fs.nr_open
```

Then investigate:

- Files opened inside loops without being closed.
- HTTP connections or sockets not released.
- Subprocess pipes left open.
- Excessive parallel workers.
- Unbounded connection pools.

For Python, context managers help release resources:

```python
for filename in filenames:
    with open(filename, encoding="utf-8") as handle:
        process(handle)
```

If a larger limit is justified, configure it at the correct launch point.

For a **systemd service**:

```bash
sudo systemctl edit release-agent.service
```

Add:

```ini
[Service]
LimitNOFILE=65535:65535
```

Then apply the change through a controlled service restart:

```bash
sudo systemctl daemon-reload
sudo systemctl restart release-agent.service
```

Systemd supports separate soft and hard values. Check application compatibility before raising the soft limit substantially: older applications using `select()` cannot handle descriptors numbered 1024 or higher. [Linux manual page](https://man7.org/linux/man-pages/man5/systemd.exec.5.html?utm_source=chatgpt.com)

For PAM-managed login sessions, an example configuration is:

```text
# /etc/security/limits.d/99-release.conf
release soft nofile 65535
release hard nofile 65535
```

It applies to new sessions when `pam_limits` is configured. Systemd services have their own launch configuration. [Linux manual page](https://man7.org/linux/man-pages/man5/limits.conf.5.html?utm_source=chatgpt.com)

**Interview answer:** “I inspect descriptor usage and the process’s actual limits, fix leaks or excessive concurrency, and increase the applicable limit only when the workload requires it.”

**4. What is the difference between A and CNAME records?**

| Record | Maps a name to | Example |
|---|---|---|
| A | An IPv4 address | `app.example.com → 192.0.2.10` |
| AAAA | An IPv6 address | `app.example.com → 2001:db8::10` |
| CNAME | Another DNS name | `www.example.com → app.example.com` |

Example DNS records:

```dns
app.example.com.  300 IN A     192.0.2.10
www.example.com.  300 IN CNAME app.example.com.
```

When resolving the address of `www.example.com`, the resolver follows the alias and resolves `app.example.com`.

A CNAME target is a DNS name; it does not contain a URL path, protocol, or port. DNS defines A records as addresses and CNAME records as canonical-name references. [implementation and specification](https://datatracker.ietf.org/doc/html/rfc1035?utm_source=chatgpt.com)

Two practical points:

- A CNAME does not perform an HTTP redirect; the browser keeps the requested hostname.
- A conventional CNAME cannot coexist with ordinary records at the same name. This prevents its conventional use at a zone apex, where SOA and NS records are required. Some providers offer separate alias or flattening features. [Clarifications to the DNS Specification](https://datatracker.ietf.org/doc/html/rfc2181?utm_source=chatgpt.com)

**5. The VM’s date is behind the current date. How do you fix it?**

I first distinguish an incorrect timezone from actual clock drift.

```bash
date
date -u
timedatectl
```

An incorrect timezone changes the displayed local time. Clock drift means the underlying system time is wrong.

If the VM uses Chrony:

```bash
chronyc tracking
chronyc sources -v
```

Check:

- Whether a valid time source is selected.
- Whether the source is reachable.
- The measured offset.
- Whether the time service is running.
- Whether network rules allow the configured synchronization traffic.
- Whether hypervisor time synchronization is competing with the guest’s time service.

Common service names are:

| Distribution/configuration | Service |
|---|---|
| RHEL with Chrony | `chronyd` |
| Ubuntu with Chrony | `chrony` |

For example, on RHEL:

```bash
sudo systemctl enable --now chronyd
```

Once Chrony has a valid reference, a large correction can be applied with:

```bash
sudo chronyc makestep
```

This steps the clock immediately instead of gradually correcting it through slewing. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/configuring_time_synchronization/using-chrony?utm_source=chatgpt.com)

For production, coordinate large clock corrections because scheduled jobs, authentication, and timestamp-dependent applications can be affected. Fix the synchronization configuration so the problem does not recur.

If only the timezone is wrong:

```bash
sudo timedatectl set-timezone Asia/Kolkata
```

**6. What is Cross-Origin Resource Sharing—CORS?**

CORS is an HTTP mechanism through which a server tells browsers which origins may access its responses from JavaScript.

An **origin** consists of:

```text
scheme + hostname + port
```

Therefore, these are different origins:

```text
https://portal.example.com
https://api.example.com
https://portal.example.com:8443
```

Suppose JavaScript on `https://portal.example.com` calls `https://api.example.net/orders`.

The API can return:

```http
Access-Control-Allow-Origin: https://portal.example.com
Vary: Origin
```

Requests involving methods or headers outside the permitted “simple request” conditions normally require an **OPTIONS preflight**. [MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS?utm_source=chatgpt.com)

An example preflight response is:

```http
Access-Control-Allow-Origin: https://portal.example.com
Access-Control-Allow-Methods: GET, POST
Access-Control-Allow-Headers: Content-Type, Authorization
Vary: Origin
```

If credentialed browser requests are required, the appropriate response also includes:

```http
Access-Control-Allow-Credentials: true
```

Credentialed requests require an explicit allowed origin; `Access-Control-Allow-Origin: *` is unsuitable. For multiple approved origins, validate the incoming origin against an allowlist and return that specific origin. [MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Access-Control-Allow-Origin?utm_source=chatgpt.com)

Test a preflight response with:

```bash
curl -i -X OPTIONS https://api.example.net/orders \
  -H 'Origin: https://portal.example.com' \
  -H 'Access-Control-Request-Method: POST' \
  -H 'Access-Control-Request-Headers: content-type,authorization'
```

CORS governs browser access to responses. The API must separately enforce authentication and authorization.

**7. What is the purpose of `/proc` in Linux?**

`/proc` is a virtual filesystem exposing process information, system information, and some kernel settings. Its contents are generated by the kernel. [Linux manual page](https://man7.org/linux/man-pages/man5/proc.5.html?utm_source=chatgpt.com)

| Path | Information |
|---|---|
| `/proc/cpuinfo` | CPU information |
| `/proc/meminfo` | Memory statistics |
| `/proc/loadavg` | System load averages |
| `/proc/uptime` | System uptime |
| `/proc/stat` | CPU and other system counters |
| `/proc/<pid>/status` | Process status and resource information |
| `/proc/<pid>/cmdline` | Process command-line arguments |
| `/proc/<pid>/limits` | Process resource limits |
| `/proc/<pid>/fd/` | Open file descriptors |
| `/proc/sys/` | Runtime kernel parameters |

Examples:

```bash
head /proc/meminfo

cat "/proc/$$/limits"

ls -l /proc/12345/fd
```

`$$` is the current shell’s PID. `/proc/self` refers to the process accessing that path.

For release troubleshooting, `/proc` helps identify memory consumption, resource limits, open sockets/files, and the executable or arguments associated with a process.

Entries under `/proc/<pid>/fd` represent the process’s open descriptors, including standard input, output, error, pipes, and sockets. [Linux manual page](https://www.man7.org/linux/man-pages/man5/proc_pid_fd.5.html?utm_source=chatgpt.com)

**8. What is the purpose of `sed`?**

`sed` is a stream editor used to transform or select text, usually without opening an interactive editor.

Common examples:

```bash
# Replace the first occurrence on each line
sed 's/old/new/' file.txt

# Replace every occurrence on each line
sed 's/old/new/g' file.txt

# Print only lines 10–20
sed -n '10,20p' file.txt

# Delete lines containing DEBUG
sed '/DEBUG/d' application.log
```

Its commands can use line numbers or patterns as addresses. [gnu.org](https://www.gnu.org/software/sed/manual/html_node/sed-script-overview.html?utm_source=chatgpt.com)

By default, the transformed result goes to standard output. On GNU sed, modify a file while keeping a backup with:

```bash
sed -i.bak 's/old/new/g' file.txt
```

Release engineers use `sed` for controlled text transformations and log processing. For structured JSON, YAML, or XML, a format-aware parser helps preserve the document’s structure.

**9. Write Python code to find a missing file**

A missing file must be identified against an expected filename, manifest, or reference directory.

This script checks expected files inside a supplied directory and distinguishes missing files from inspection errors:

```python
import argparse
import stat
import sys
from pathlib import Path

parser = argparse.ArgumentParser(
    description="Check expected release files."
)
parser.add_argument("directory", type=Path)
parser.add_argument("files", nargs="+")
args = parser.parse_args()

try:
    directory_mode = args.directory.stat().st_mode
except OSError as exc:
    parser.error(f"Cannot inspect directory: {exc}")

if not stat.S_ISDIR(directory_mode):
    parser.error("The supplied path is not a directory.")

missing = False
inspection_failed = False

for filename in args.files:
    path = args.directory / filename

    try:
        mode = path.stat().st_mode
    except FileNotFoundError:
        print(f"MISSING: {path}")
        missing = True
    except OSError as exc:
        print(f"CHECK FAILED: {path}: {exc}", file=sys.stderr)
        inspection_failed = True
    else:
        if not stat.S_ISREG(mode):
            print(f"NOT A REGULAR FILE: {path}")
            missing = True

sys.exit(
    2 if inspection_failed
    else 1 if missing
    else 0
)
```

Usage:

```bash
python3 find_missing_files.py /opt/releases/app \
  application.jar config.yaml checksums.txt
```

Exit codes:

| Code | Meaning |
|---|---|
| `0` | All expected files are present |
| `1` | A file is missing or has the wrong type |
| `2` | An inspection or argument error occurred |

Using `stat()` allows errors such as permission failures to be handled explicitly. Current Python documentation notes that `exists()` and `is_file()` can return false for inaccessible paths as well as missing ones. [Python 3.14.8 documentation](https://docs.python.org/3/library/pathlib.html?utm_source=chatgpt.com)

For a release pipeline, follow presence checks with checksum or signature verification.

**10. Is it possible to add multiple aliases to a domain?**

Yes. Multiple names can point to the same canonical name.

```dns
app.example.com.      IN A     192.0.2.10
www.example.com.      IN CNAME app.example.com.
portal.example.com.   IN CNAME app.example.com.
reports.example.com.  IN CNAME app.example.com.
```

Here, three aliases resolve through `app.example.com`.

Each individual alias name has **one CNAME target**. DNS does not permit assigning several different CNAME targets to the same alias. Address distribution can instead use multiple A/AAAA records or an appropriate load-balancing service. [Clarifications to the DNS Specification](https://datatracker.ietf.org/doc/html/rfc2181?utm_source=chatgpt.com)

For a web application, configure the reverse proxy/application to accept each requested hostname and provide TLS certificates covering those names. DNS resolution alone does not configure virtual hosts or TLS identity.

**11. If you have two different domains, how do you enable communication between them?**

For web applications, I work through the connection layers.

1. **DNS:** The caller must resolve the destination name.
2. **Network:** Routing and firewall rules must permit the required connection.
3. **TLS:** The destination must present a trusted certificate for its hostname.
4. **Application access:** Configure authentication, authorization, and the appropriate API contract.

The caller determines the additional requirements:

| Scenario | Additional configuration |
|---|---|
| Browser on domain A calls an API on domain B | Configure the API’s CORS allowlist |
| Backend service A calls backend service B | Configure service credentials, OAuth, or mTLS as appropriate |
| Services communicate over private networks | Configure private routing, DNS, and network access |
| Users need a shared login across applications | Configure identity federation/SSO |

For example, a browser application on `https://portal.example.com` calling `https://api.example.net` requires the API to allow the portal’s origin.

For cookies across sites, also review `SameSite`, `Secure`, credential settings, and browser restrictions.

If “domains” means **Active Directory domains**, the answer involves appropriately scoped domain/forest trusts, DNS, network connectivity, and permissions. Establishing a trust and granting access are separate configuration steps.

**12. How do you create certificates for multiple subdomains?**

Two common approaches are **SAN certificates** and **wildcard certificates**.

| Approach | Example | Coverage |
|---|---|---|
| SAN certificate | `app.example.com`, `api.example.com`, `reports.example.com` | Explicitly listed names |
| Wildcard certificate | `*.example.com` | Matching subdomains at one label level |

A certificate for `*.example.com` covers `api.example.com`, but does not cover the apex `example.com` or `api.dev.example.com`.

TLS hostname verification uses the certificate’s Subject Alternative Name entries, and wildcard matching is constrained to the complete leftmost label. [Service Identity in TLS](https://datatracker.ietf.org/doc/html/rfc9525?utm_source=chatgpt.com)

An OpenSSL example for generating a private key and CSR containing several names:

```bash
umask 077

openssl req -new -newkey rsa:3072 \
  -keyout application.key \
  -out application.csr \
  -subj '/CN=app.example.com' \
  -addext \
  'subjectAltName=DNS:app.example.com,DNS:api.example.com,DNS:reports.example.com'
```

This prompts for private-key encryption. OpenSSL supports requesting SAN extensions through `-addext`. [OpenSSL Documentation](https://docs.openssl.org/3.6/man1/openssl-req/?utm_source=chatgpt.com)

The complete process is:

1. Generate the key and CSR, or use a managed certificate service.
2. Request issuance from a trusted public or internal CA.
3. Complete domain-control validation.
4. Install the certificate, chain, and protected key at the TLS endpoint.
5. Automate renewal and monitor expiry.

For Let’s Encrypt wildcard issuance, use DNS-01 validation. Automating DNS updates through a narrowly scoped provider integration also supports reliable renewal. [Let's Encrypt](https://letsencrypt.org/docs/challenge-types/?utm_source=chatgpt.com)

Verify an issued certificate:

```bash
openssl x509 -in application.crt \
  -noout -dates -ext subjectAltName
```

**13. Hard link versus soft link**

| Aspect | Hard link | Soft/symbolic link |
|---|---|---|
| References | The same underlying inode | A target pathname |
| Inode | Shares the target inode | Has its own inode |
| Across filesystems | Generally unsupported | Supported |
| Directory targets | Generally prohibited for ordinary users | Supported |
| Target pathname removed | Other hard links still access the file | Link becomes dangling if its target disappears |
| Target pathname moved | Existing hard links still access the inode | May break, depending on target path |

Create them with:

```bash
# Hard link
ln original.txt hard-link.txt

# Symbolic link
ln -s original.txt soft-link.txt
```

Hard links are equivalent names for the same underlying file. Symbolic links store a pathname that is resolved when accessed. [Linux manual page](https://man7.org/linux/man-pages/man7/symlink.7.html?utm_source=chatgpt.com)

Writing to the shared inode changes the content visible through its hard links. However, replacing one pathname with a new file creates a different inode—an important distinction for atomic updates and log rotation.

The original inode’s storage is released when no links and no remaining open references require it.

**14. What Linux command creates a soft link?**

```bash
ln -s TARGET LINK_NAME
```

Example:

```bash
ln -s /opt/app/releases/v3 /opt/app/current
```

Applications accessing `/opt/app/current` follow the link to `/opt/app/releases/v3`.

Inspect it with:

```bash
ls -l /opt/app/current

readlink /opt/app/current
```

A relative target is resolved relative to the directory containing the symlink.

By default, `ln` creates a hard link; `-s` selects a symbolic link. [Linux manual page](https://man7.org/linux/man-pages/man1/ln.1.html?utm_source=chatgpt.com)

**15. Using `sed`, how do you remove the first and last lines?**

```bash
sed '1d;$d' input.txt
```

Explanation:

- `1d`: Delete the first line.
- `$d`: Delete the last line.
- `;`: Separate the commands.

Save the result separately:

```bash
sed '1d;$d' input.txt > output.txt
```

Or modify the input while retaining a backup:

```bash
sed -i.bak '1d;$d' input.txt
```

GNU sed supports line addressing and the `d` delete command. [gnu.org](https://www.gnu.org/software/sed/manual/sed.html?utm_source=chatgpt.com)

Use single quotes so the shell passes `$` unchanged to sed. For an input with fewer than three lines, the resulting content is empty.

When redirecting, use a different output file: shell redirection truncates its destination before sed reads the input.

**16. What are stdin, stdout, and stderr in Linux?**

They are the three standard streams inherited by a process.

| Stream | Descriptor | Purpose |
|---|---:|---|
| stdin | `0` | Input |
| stdout | `1` | Normal output |
| stderr | `2` | Diagnostic/error output |

Examples:

```bash
# Supply standard input from a file
sort < input.txt

# Save normal output
./deploy.sh > deployment.out

# Save diagnostic output
./deploy.sh 2> deployment.err

# Save the streams separately
./deploy.sh > deployment.out 2> deployment.err

# Combine both streams
./deploy.sh > deployment.log 2>&1

# Append both streams
./deploy.sh >> deployment.log 2>&1
```

Redirections are processed from left to right. In:

```bash
./deploy.sh > deployment.log 2>&1
```

stdout is redirected first, then stderr is made to use the same destination.

A normal pipe carries stdout:

```bash
producer | consumer
```

To include stderr:

```bash
producer 2>&1 | consumer
```

In Bash release scripts, `set -o pipefail` helps detect failures in earlier commands of a pipeline.

**17. What is ingress versus egress?**

The direction is relative to the resource being discussed.

| Direction | Meaning | Example |
|---|---|---|
| Ingress | Traffic entering the resource | A client connects to the application on TCP 443 |
| Egress | Traffic leaving the resource | The application connects to a database on TCP 5432 |

For an application server:

- Allow ingress from approved clients or load balancers.
- Allow egress to required dependencies, registries, DNS services, and APIs.

A server can therefore have restricted inbound access while still needing outbound connectivity for dependency downloads or artifact uploads.

In Kubernetes:

- An **Ingress resource** configures incoming HTTP(S) routing through an Ingress controller.
- **NetworkPolicies** can control pod ingress and egress.

When both source egress and destination ingress are isolated by policies, both sides must permit the connection. NetworkPolicy enforcement requires a supporting network implementation. [Kubernetes](https://kubernetes.io/docs/concepts/services-networking/network-policies/?utm_source=chatgpt.com)

For troubleshooting, check the connection from both ends: source outbound permissions, destination inbound permissions, routing, and the listening service.

**18. How do you fetch errors from log files?**

For a simple text log:

```bash
# Case-insensitive search
grep -ni 'error' application.log

# Match ERROR as a whole word
grep -niw 'error' application.log

# Match several severity words
grep -niwE 'error|fatal|critical' application.log

# Save matching entries
grep -iw 'error' application.log > errors.log
```

Search several files while retaining filenames:

```bash
grep -HniwE 'error|fatal|critical' /var/log/myapp/*.log
```

Include surrounding context:

```bash
grep -ni -B 5 -A 10 'error' application.log
```

Follow a log, including rotation:

```bash
tail -F application.log |
  grep --line-buffered -i 'error'
```

For compressed logs:

```bash
zgrep -ni 'error' application.log*.gz
```

For a systemd service:

```bash
journalctl -u myapp.service \
  --since '1 hour ago' \
  -p err
```

The journal priority filter depends on correctly assigned journal severity. Applications may print error text without assigning an error priority.

In automation, handle grep’s exit status correctly:

| Status | Meaning |
|---|---|
| `0` | Matching lines found |
| `1` | No matching lines |
| `2` | Error occurred |

A no-match result can otherwise fail a script using `set -e`. [gnu.org](https://www.gnu.org/s/grep/manual/html_node/Exit-Status.html?utm_source=chatgpt.com)

For structured logs, filter the severity field through a suitable parser. During release troubleshooting, correlate errors with deployment time, release version, request IDs, and the affected instance or pod.
