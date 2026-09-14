Below are **brief, interview-ready answers** with the **code, how it works, and expected output**, which is exactly how you can explain them in an interview.

## 1. Python script to monitor a directory and print new files added every minute

### Code

```python
import os
import time

directory = "/tmp/test"

# Store files that already exist
existing_files = set(os.listdir(directory))

while True:
    time.sleep(60)

    current_files = set(os.listdir(directory))

    # Find newly added files
    new_files = current_files - existing_files

    if new_files:
        for file in new_files:
            print(f"New file added: {file}")
    else:
        print("No new files added")

    # Update existing file list
    existing_files = current_files
```

### How it works

First, this line reads the files already available in the directory:

```python
existing_files = set(os.listdir(directory))
```

Suppose `/tmp/test` initially contains:

```text
file1.txt
file2.txt
```

Then:

```python
existing_files
```

contains:

```text
{"file1.txt", "file2.txt"}
```

After 60 seconds, the script checks the directory again:

```python
current_files = set(os.listdir(directory))
```

Suppose someone added:

```text
file3.txt
```

Now:

```text
current_files =
{"file1.txt", "file2.txt", "file3.txt"}
```

This operation:

```python
new_files = current_files - existing_files
```

produces:

```text
{"file3.txt"}
```

### Output

```text
New file added: file3.txt
```

If nothing was added:

```text
No new files added
```

### Interview explanation

You can say:

> I am using polling. First I store the existing filenames in a Python set. Every 60 seconds I read the directory again and use set difference to identify new files. After checking, I update the existing file set.

A production-grade version might use filesystem events with `watchdog` instead of polling, but for an interview this solution is simple and clear.

---

# 2. Difference between `set` and `list` in Python

The main difference is that a **list allows duplicate values and maintains order**, whereas a **set stores unique values and is primarily used for fast membership checks and set operations**.

### List

```python
numbers = [1, 2, 2, 3]

print(numbers)
```

Output:

```text
[1, 2, 2, 3]
```

Duplicates are allowed.

You can access elements using indexes:

```python
print(numbers[0])
```

Output:

```text
1
```

### Set

```python
numbers = {1, 2, 2, 3}

print(numbers)
```

Conceptually the result is:

```text
{1, 2, 3}
```

The duplicate `2` is removed.

You cannot do:

```python
numbers[0]
```

because sets do not provide positional indexing.

### Why did we use a set in the directory-monitoring script?

Because sets make it very easy to calculate differences:

```python
new_files = current_files - existing_files
```

Example:

```python
old = {"a.txt", "b.txt"}

new = {"a.txt", "b.txt", "c.txt"}

print(new - old)
```

Output:

```text
{'c.txt'}
```

A good interview answer is:

> A list is ordered, supports duplicates and indexing. A set contains unique values and provides efficient membership testing and operations such as union, intersection and difference.

---

# 3. SQL: Customers who placed more than 3 orders in the last 90 days

Tables:

```text
Customers
---------
customer_id
customer_name
```

and:

```text
Orders
------
order_id
customer_id
order_date
amount
```

### PostgreSQL-style query

```sql
SELECT
    c.customer_name
FROM Customers c
JOIN Orders o
    ON c.customer_id = o.customer_id
WHERE o.order_date >= CURRENT_DATE - INTERVAL '90 days'
GROUP BY
    c.customer_id,
    c.customer_name
HAVING COUNT(o.order_id) > 3;
```

### How it works

The `JOIN` connects customers with their orders:

```sql
ON c.customer_id = o.customer_id
```

The `WHERE` condition keeps only orders from the last 90 days:

```sql
WHERE o.order_date >= CURRENT_DATE - INTERVAL '90 days'
```

Then:

```sql
GROUP BY c.customer_id, c.customer_name
```

groups orders customer by customer.

Finally:

```sql
HAVING COUNT(o.order_id) > 3
```

returns only customers with more than three orders.

Suppose:

```text
John   -> 5 orders
David  -> 2 orders
Mary   -> 4 orders
```

Output:

```text
customer_name
-------------
John
Mary
```

### Important interview point

`WHERE` filters individual rows before grouping.

`HAVING` filters aggregated/grouped results.

If they ask for MySQL syntax, you can use:

```sql
WHERE o.order_date >= CURRENT_DATE - INTERVAL 90 DAY
```

---

# 4. Python function to return job IDs where status is FAILED

Your example has some syntax errors, so the corrected input would be:

```python
logs = [
    {
        "job_id": 101,
        "status": "SUCCESS",
        "timestamp": "2025-06-10T10:00:00"
    },
    {
        "job_id": 102,
        "status": "FAILED",
        "timestamp": "2025-06-10T10:05:00"
    },
    {
        "job_id": 103,
        "status": "FAILED",
        "timestamp": "2025-06-10T10:10:00"
    },
    {
        "job_id": 104,
        "status": "SUCCESS",
        "timestamp": "2026-06-10T10:16:00"
    }
]
```

### Function

```python
def get_failed_jobs(logs):
    failed_jobs = []

    for log in logs:
        if log["status"] == "FAILED":
            failed_jobs.append(log["job_id"])

    return failed_jobs


result = get_failed_jobs(logs)

print(result)
```

### Output

```text
[102, 103]
```

### How it works

The function loops through each dictionary:

```python
for log in logs:
```

Then checks:

```python
if log["status"] == "FAILED":
```

If it failed, we add the job ID:

```python
failed_jobs.append(log["job_id"])
```

Then return the result.

### Shorter Python version

You can also use list comprehension:

```python
def get_failed_jobs(logs):
    return [
        log["job_id"]
        for log in logs
        if log["status"] == "FAILED"
    ]
```

Output is still:

```text
[102, 103]
```

For interviews, I'd first show the loop version because it is easier to explain, then mention that it can be simplified using list comprehension.

---

# 5. What is CI/CD?

**CI/CD means Continuous Integration and Continuous Delivery/Deployment.**

### Continuous Integration

Developers frequently push code to a shared repository.

Every code change automatically runs things such as:

```text
Code Commit
    |
    v
Build
    |
    v
Unit Tests
    |
    v
Code Quality
    |
    v
Security Scan
```

For example:

```text
Developer
    |
   Git
    |
 GitHub
    |
 Jenkins
    |
 Build
    |
 Test
```

The goal is to identify problems early.

### Continuous Delivery

After CI completes, the application is automatically prepared for deployment.

For example:

```text
Code
 |
Build
 |
Test
 |
Docker Image
 |
Push to ECR
 |
Deploy to Dev
 |
Deploy to QA
 |
Manual approval
 |
Production
```

### Continuous Deployment

Continuous Deployment goes one step further.

If all automated checks pass, production deployment happens automatically:

```text
Commit
  |
 Build
  |
 Test
  |
 Security Scan
  |
 Deploy
  |
Production
```

### Tools I've commonly seen

```text
Source Control -> GitHub / GitLab / Bitbucket

CI/CD -> Jenkins / GitHub Actions / GitLab CI

Containers -> Docker

Registry -> ECR / Docker Hub

Deployment -> Argo CD / Helm / Kubernetes
```

### Interview answer

> CI/CD automates the software delivery lifecycle. CI automatically builds and tests code whenever developers make changes. CD automates packaging and deployment of that validated code into environments such as development, QA and production.

---

# 6. Explain Kubernetes architecture, components and their uses

Kubernetes architecture has two major parts:

```text
Control Plane
     +
Worker Nodes
```

A simplified architecture is:

```text
                     CONTROL PLANE

                    API Server
                       |
             +---------+---------+
             |         |         |
          Scheduler Controller  etcd
                       |
                       |
 ------------------------------------------------
                     CLUSTER
 ------------------------------------------------

               WORKER NODE
                    |
                 kubelet
                    |
              Container Runtime
                    |
             +------+------+
             |             |
           Pod A          Pod B
```

## API Server

The API Server is the main entry point into Kubernetes.

For example:

```bash
kubectl get pods
```

The request goes to:

```text
kubectl
   |
   v
API Server
```

It validates and processes Kubernetes API requests.

---

## etcd

`etcd` is Kubernetes' distributed key-value store.

It stores cluster state such as:

```text
Deployments
Pods
Services
ConfigMaps
Secrets
Cluster configuration
```

If you create:

```yaml
replicas: 3
```

the desired state is stored through the Kubernetes control plane.

For self-managed Kubernetes, etcd backup is extremely important.

---

## Scheduler

The scheduler decides **which worker node should run a new pod**.

Suppose:

```text
Node A
CPU available: 1

Node B
CPU available: 8
```

And the new pod needs:

```text
CPU: 2
```

The scheduler may select:

```text
Node B
```

It considers things such as:

```text
CPU/memory requests
Node affinity
Pod affinity/anti-affinity
Taints and tolerations
Topology constraints
```

---

## Controller Manager

Controllers continuously compare the **desired state** against the **actual state**.

Suppose:

```yaml
replicas: 3
```

Current state:

```text
Pod 1 = Running
Pod 2 = Running
Pod 3 = Crashed
```

The controller detects:

```text
Desired = 3
Actual = 2
```

and Kubernetes creates another pod.

This is Kubernetes' **self-healing** behavior.

---

# Worker-node components

## kubelet

The kubelet runs on every worker node.

It communicates with the API server and makes sure containers described by Pods are running on that node.

```text
API Server
    |
    v
 kubelet
    |
    v
 Container Runtime
    |
    v
   Pod
```

---

## Container runtime

The container runtime actually runs the containers.

Modern Kubernetes commonly uses:

```text
containerd
CRI-O
```

Conceptually:

```text
Kubernetes
    |
 containerd
    |
 Container
```

---

## kube-proxy

`kube-proxy` traditionally helps implement Kubernetes service networking.

For example:

```text
Service
   |
   +---- Pod 1
   |
   +---- Pod 2
   |
   +---- Pod 3
```

It helps route service traffic toward appropriate backend pods, though some modern CNI implementations can replace parts of this functionality.

---

# Pod

A Pod is the smallest deployable unit in Kubernetes.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx
```

Create it:

```bash
kubectl apply -f pod.yaml
```

Check it:

```bash
kubectl get pods
```

Possible output:

```text
NAME    READY   STATUS    RESTARTS
nginx   1/1     Running   0
```

---

# Deployment

Normally we don't manually create production Pods.

We create a Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
```

This creates and maintains:

```text
Deployment
    |
 ReplicaSet
    |
 +-- Pod
 +-- Pod
 +-- Pod
```

If one Pod crashes, Kubernetes creates a replacement.

---

# Service

Pod IP addresses can change.

A Service provides a stable endpoint.

```text
            Service
               |
       +-------+-------+
       |       |       |
      Pod     Pod     Pod
```

For example:

```yaml
kind: Service
spec:
  selector:
    app: web
  ports:
    - port: 80
```

---

# Ingress

Ingress provides HTTP/HTTPS routing into Kubernetes.

Example:

```text
Internet
    |
Load Balancer
    |
Ingress
    |
    +---- /frontend -> Frontend Service
    |
    +---- /api      -> Backend Service
```

---

# ConfigMap and Secret

ConfigMap stores non-sensitive configuration:

```text
APP_ENV=production
LOG_LEVEL=INFO
```

Secret stores sensitive values such as:

```text
DB_PASSWORD
API_TOKEN
```

Although in AWS environments I often prefer external secret systems such as **AWS Secrets Manager** with EKS workload identity rather than manually storing production credentials inside Kubernetes.

---

# Final Kubernetes architecture answer for an interview

If they ask you to explain it **briefly**, say:

> Kubernetes has a control plane and worker nodes. The API Server handles API requests, etcd stores cluster state, the Scheduler selects worker nodes for Pods, and Controller Manager maintains the desired state. Worker nodes run kubelet, the container runtime and networking components. Applications run inside Pods, Deployments manage replicas and updates, Services provide stable networking, and Ingress exposes HTTP/HTTPS applications externally.

That answer is usually enough initially. Then expand individual components only if the interviewer asks follow-up questions.
