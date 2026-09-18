# Capgemini DevOps Interview Answers

**Experience:** 9 years total, 5 years relevant DevOps  
**Topics:** Ansible, Kubernetes, AWS, RDS, Terraform

> Use these as interview frameworks. Replace sample tools, metrics, and ownership claims with your real experience.

---

# Part 1: Ansible

## 1. What is an Ansible playbook?

An Ansible playbook is a YAML file containing one or more plays. Each play maps a host pattern to ordered tasks, variables, handlers, roles, and execution controls. Tasks call modules such as `package`, `service`, `template`, and `user`. Good playbooks are idempotent, version-controlled, linted, and tested.

```yaml
---
- name: Configure web servers
  hosts: web
  become: true
  tasks:
    - name: Install Nginx
      ansible.builtin.package:
        name: nginx
        state: present

    - name: Start and enable Nginx
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true
```

Run it with:

```bash
ansible-playbook -i inventory.ini web.yml --check --diff
ansible-playbook -i inventory.ini web.yml
```

## 2. Install Ansible on Ubuntu and Red Hat, and start services

Ansible normally runs on a control node and is agentless for Linux managed nodes. There is no mandatory `ansible` daemon to start. Managed Linux hosts need SSH access and a compatible Python runtime for most modules.

### Ubuntu control node

```bash
sudo apt update
sudo apt install -y python3 python3-pip python3-venv
python3 -m venv ~/venvs/ansible
source ~/venvs/ansible/bin/activate
python -m pip install --upgrade pip
pip install ansible-core
ansible --version
```

### Red Hat control node

```bash
sudo dnf install -y python3 python3-pip
python3 -m venv ~/venvs/ansible
source ~/venvs/ansible/bin/activate
pip install ansible-core
ansible --version
```

For an enterprise setup, prefer Red Hat Ansible Automation Platform or a pinned execution environment. Test connectivity:

```bash
ansible all -i inventory.ini -m ansible.builtin.ping
```

If the interviewer asks which service to start, clarify that community Ansible CLI has no server daemon. Automation Controller is a separate platform with its own services or containers.

## 3. Create three users and map them to prod, task, and QA groups in a single task

```yaml
- name: Create application users with groups
  become: true
  ansible.builtin.user:
    name: "{{ item.name }}"
    groups: "{{ item.group }}"
    append: true
    state: present
    create_home: true
  loop:
    - { name: alice, group: prod }
    - { name: bob, group: task }
    - { name: carol, group: qa }
```

Ensure groups exist first:

```yaml
- name: Ensure required groups exist
  become: true
  ansible.builtin.group:
    name: "{{ item }}"
    state: present
  loop:
    - prod
    - task
    - qa
```

## 4. Include files and pass input parameters in a playbook

Use `import_tasks` for static reuse parsed before execution and `include_tasks` for dynamic runtime inclusion.

```yaml
- name: Configure service
  hosts: app
  tasks:
    - name: Include deployment tasks dynamically
      ansible.builtin.include_tasks: tasks/deploy.yml
      vars:
        app_name: payments
        app_version: "2.4.1"
```

Pass command-line variables:

```bash
ansible-playbook deploy.yml -e app_version=2.4.2 -e target_env=prod
```

Use `vars_files`, inventory variables, role defaults, or an interactive `vars_prompt` where appropriate. Avoid command-line secrets because process listings and logs may expose them.

## 5. What are Ansible roles?

Roles package related tasks, handlers, defaults, variables, files, templates, metadata, and optional plugins into a standard reusable structure.

```text
roles/web/
├── defaults/main.yml
├── files/
├── handlers/main.yml
├── meta/main.yml
├── tasks/main.yml
├── templates/
├── tests/
└── vars/main.yml
```

Use roles for reusable capabilities such as `nginx`, `mysql_backup`, or `node_hardening`. Keep environment-specific data in inventory/group variables, not hard-coded in roles.

## 6. Use of templates in Ansible

The `template` module renders a Jinja2 file with variables and copies the result to a managed host. It is used for environment-specific configuration while keeping one source template.

```jinja2
# templates/app.conf.j2
server.port={{ app_port }}
database.host={{ db_host }}
{% if enable_tls %}
tls.enabled=true
{% endif %}
```

```yaml
- name: Render application configuration
  ansible.builtin.template:
    src: app.conf.j2
    dest: /etc/payment/app.conf
    owner: root
    group: payment
    mode: "0640"
    validate: '/usr/local/bin/app --check-config %s'
  notify: Restart payment service
```

## 7. Templates vs roles

- A template is a Jinja2-rendered file used to create one target file.
- A role is a complete reusable automation package that can contain tasks, templates, handlers, defaults, files, and dependencies.
- A role may use many templates; a template is not an alternative to a role.

## 8. Initialize a role and publish it to Ansible Galaxy

```bash
ansible-galaxy role init acme.nginx_hardening
cd acme.nginx_hardening
```

Add tasks, defaults, handlers, templates, tests, documentation, license, supported platforms, and metadata. Validate it:

```bash
ansible-lint .
yamllint .
molecule test
```

Push the role to a public GitHub repository, create tags/releases, then import it through Ansible Galaxy using a linked GitHub account or supported namespace workflow. Use semantic versioning and avoid publishing credentials or organization-specific secrets. Modern reusable content is often published as an Ansible collection.

## 9. How do you encrypt a playbook?

Ansible Vault can encrypt a full file, but best practice is to encrypt only secret variable files or individual sensitive values so normal tasks remain reviewable.

```bash
ansible-vault create group_vars/prod/vault.yml
ansible-vault encrypt group_vars/prod/vault.yml
ansible-vault edit group_vars/prod/vault.yml
ansible-vault view group_vars/prod/vault.yml
ansible-vault encrypt_string 'MySecret' --name db_password
```

## 10. How do you execute a playbook with a Vault file?

```bash
ansible-playbook -i inventories/prod site.yml --ask-vault-pass
```

Using a password source:

```bash
ansible-playbook -i inventories/prod site.yml   --vault-id prod@/secure/path/vault-pass.sh
```

For multiple environments:

```bash
ansible-playbook site.yml   --vault-id dev@prompt   --vault-id prod@/secure/path/prod-vault-client
```

Do not commit a plaintext Vault password file.

## 11. Use an environment variable for Vault passwords

Do not put the secret directly in a command argument. Use an executable Vault password client that reads a protected environment variable managed by the CI secret store.

```bash
#!/usr/bin/env bash
set -euo pipefail
: "${ANSIBLE_VAULT_PASSWORD:?Not set}"
printf '%s' "$ANSIBLE_VAULT_PASSWORD"
```

```bash
chmod 700 vault-pass-client.sh
export ANSIBLE_VAULT_PASSWORD='injected-by-ci-secret-store'
ansible-playbook site.yml --vault-id prod@./vault-pass-client.sh
unset ANSIBLE_VAULT_PASSWORD
```

Restrict logs, environment inspection, workspace access, and process debugging. Prefer an external secret manager where possible.

## 12. Create a MySQL dump using encrypted credentials

Encrypted variables:

```yaml
# group_vars/db/vault.yml, encrypted with ansible-vault
vault_mysql_backup_user: backup_user
vault_mysql_backup_password: strong-secret
```

Playbook:

```yaml
---
- name: Back up MySQL databases
  hosts: db
  become: true
  serial: 1
  vars:
    backup_dir: /var/backups/mysql
  tasks:
    - name: Create protected backup directory
      ansible.builtin.file:
        path: "{{ backup_dir }}"
        state: directory
        owner: root
        group: root
        mode: "0700"

    - name: Dump database
      community.mysql.mysql_db:
        name: payments
        state: dump
        target: "{{ backup_dir }}/payments-{{ ansible_date_time.iso8601_basic_short }}.sql.gz"
        login_user: "{{ vault_mysql_backup_user }}"
        login_password: "{{ vault_mysql_backup_password }}"
      no_log: true

    - name: Find older backups
      ansible.builtin.find:
        paths: "{{ backup_dir }}"
        age: 14d
        patterns: '*.sql.gz'
      register: old_backups

    - name: Remove older backups
      ansible.builtin.file:
        path: "{{ item.path }}"
        state: absent
      loop: "{{ old_backups.files }}"
```

Execute:

```bash
ansible-galaxy collection install community.mysql
ansible-playbook -i inventory.ini mysql-backup.yml --ask-vault-pass
```

Test restore regularly. Encryption of credentials does not automatically encrypt the dump; encrypt and transfer the backup separately if required.

## 13. Execute on all DB servers except one

Inventory pattern:

```bash
ansible-playbook -i inventory.ini db.yml --limit 'db:!db03.example.com'
```

In a playbook:

```yaml
hosts: "db:!db03.example.com"
```

Prefer a maintenance group or variable for long-term exclusions rather than hard-coding a hostname.

## 14. Shut down all servers with an ad-hoc command

```bash
ansible all -i inventory.ini -b   -m community.general.shutdown   -a 'delay=0 msg="Approved maintenance shutdown"'
```

Validate inventory and use `--limit` for staged execution. Never shut down the Ansible control path, bastion, or critical quorum members first.

## 15. Reduce execution time on RDBMS servers

- Use facts only when needed: `gather_facts: false` or `gather_subset`.
- Increase `forks` carefully.
- Enable SSH pipelining and connection multiplexing.
- Use efficient modules rather than shell loops.
- Cache facts.
- Use `strategy: free` where independent hosts do not need lockstep execution.
- Use asynchronous tasks for long-running independent operations.
- Limit inventory to relevant hosts and use tags.
- Avoid repeatedly downloading large artifacts.
- Use `serial` for safe database operations even though it reduces speed.

For databases, safety and quorum are more important than maximum parallelism.

## 16. Configure performance options from Ansible files

`ansible.cfg`:

```ini
[defaults]
forks = 20
gathering = smart
fact_caching = jsonfile
fact_caching_connection = .ansible/facts
fact_caching_timeout = 3600
host_key_checking = True

[ssh_connection]
pipelining = True
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
```

Playbook controls:

```yaml
- hosts: db
  gather_facts: false
  strategy: free
  serial: "25%"
  tasks:
    - name: Long-running maintenance operation
      ansible.builtin.command: /opt/db/maintenance.sh
      async: 3600
      poll: 15
```

Do not use `strategy: free` when tasks require coordinated ordering across hosts.

## 17. Increase Ansible debug log level

```bash
ansible-playbook site.yml -v
ansible-playbook site.yml -vv
ansible-playbook site.yml -vvv
ansible-playbook site.yml -vvvv
```

- `-v`: task results
- `-vv`: more details
- `-vvv`: connection information
- `-vvvv`: very detailed connection debugging

Configure a log file carefully:

```ini
[defaults]
log_path = /var/log/ansible/ansible.log
```

Use `no_log: true` for secret-bearing tasks. Verbosity can expose sensitive data.

## 18. Static vs dynamic inventory

- Static inventory is a manually maintained INI or YAML list of hosts and groups. It is simple and suitable for stable environments.
- Dynamic inventory queries an API or source of truth, such as AWS, Azure, VMware, or a CMDB. It suits ephemeral cloud instances and can group systems by tags.

```bash
ansible-inventory -i inventory.ini --graph
ansible-inventory -i aws_ec2.yml --list
```

Example AWS inventory plugin:

```yaml
plugin: amazon.aws.aws_ec2
regions:
  - ap-south-1
filters:
  instance-state-name: running
keyed_groups:
  - key: tags.Environment
    prefix: env
```

---

# Part 2: Kubernetes

## 19. Kubernetes architecture

A cluster has a control plane and worker nodes.

- API server: validates and serves Kubernetes API requests.
- etcd: strongly consistent cluster-state store.
- Scheduler: selects a node for unscheduled Pods.
- Controller manager: runs reconciliation controllers.
- Cloud controller manager: integrates supported cloud infrastructure.
- Kubelet: ensures assigned Pods run on a node.
- Container runtime: runs containers through CRI.
- CNI: provides Pod networking.
- kube-proxy or an equivalent data plane: implements Service traffic forwarding.

## 20. What happens with `kubectl apply`, and how are Services used?

`kubectl apply -f` sends the desired object configuration to the API server. Authentication, authorization, admission, validation, and persistence occur. Controllers reconcile the desired state; the scheduler assigns Pods; kubelets start containers.

A Service provides a stable virtual IP/DNS name and selects ready Pod endpoints using labels. The Service controller and EndpointSlice controllers maintain endpoint data, while kube-proxy or an eBPF data plane forwards traffic.

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl get deploy,pods,svc,endpointslices
```

## 21. Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api
  namespace: payments
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app: payment-api
  template:
    metadata:
      labels:
        app: payment-api
    spec:
      containers:
        - name: payment-api
          image: registry.example.com/payment-api:2.4.1
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: "1"
              memory: 512Mi
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
```

## 22. Labels and annotations

- Labels are indexed key/value metadata used for selection and grouping by Services, Deployments, affinity, and commands.
- Annotations carry non-identifying metadata for tools, controllers, owners, checksums, or documentation and are not used as selectors.

```yaml
metadata:
  labels:
    app: payment-api
    environment: prod
  annotations:
    owner: payments-team
    change-ticket: CHG12345
```

## 23. Traffic from outside a cluster

Typical path:

```text
Client -> DNS -> cloud load balancer -> Ingress/Gateway controller
       -> Service -> ready Pod endpoint
```

Options include `Service` type `LoadBalancer`, Ingress for HTTP/HTTPS, and Gateway API. Use TLS, WAF where applicable, health probes, NetworkPolicy, authentication, and observability.

## 24. Controller manager tasks

The controller manager runs controllers that continuously reconcile actual state toward desired state. Examples include Deployment/ReplicaSet, Job, node, namespace, ServiceAccount, and EndpointSlice-related controllers. Controllers watch the API server and create, update, or delete resources; they do not usually manipulate etcd directly.

## 25. Node affinity and anti-affinity

Node affinity schedules Pods according to node labels.

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: workload
                operator: In
                values: [payments]
```

Pod anti-affinity spreads replicas away from each other:

```yaml
podAntiAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: payment-api
      topologyKey: kubernetes.io/hostname
```

Use `required` for hard constraints and `preferred` for soft preferences. Overly strict rules can leave Pods Pending.

## 26. Run two Pods where one depends on another

Do not depend on Pod startup order across separate Deployments. Expose the dependency through a Service and make the consumer retry with backoff. Use readiness probes. An init container can wait for a prerequisite, but avoid infinite waits.

```yaml
initContainers:
  - name: wait-for-database
    image: busybox:1.36
    command: ['sh', '-c', 'until nc -z database 5432; do sleep 2; done']
```

If both containers must share lifecycle, localhost, and volumes, place them in the same Pod using a sidecar pattern. Kubernetes Roles/RBAC authorize API actions; they do not define startup dependency.

## 27. Taints and tolerations

A taint repels Pods from a node unless they have a matching toleration.

```bash
kubectl taint nodes node01 dedicated=database:NoSchedule
```

```yaml
tolerations:
  - key: dedicated
    operator: Equal
    value: database
    effect: NoSchedule
```

A toleration permits scheduling but does not guarantee it. Combine it with node affinity for dedicated placement.

## 28. Why taint worker nodes?

- Reserve nodes for database, GPU, security, or system workloads.
- Prevent ordinary workloads from using specialized or expensive nodes.
- Evict Pods during node problems with `NoExecute` taints.
- Represent temporary node conditions such as pressure or unreachability.
- Separate trust, compliance, or noisy workloads.

Effects are `NoSchedule`, `PreferNoSchedule`, and `NoExecute`.

## 29. Limit Kubernetes resources

At container level, set requests and limits. At namespace level, use `ResourceQuota` and `LimitRange`.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: payments
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "100"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: defaults
  namespace: payments
spec:
  limits:
    - type: Container
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      default:
        cpu: 500m
        memory: 512Mi
```

## 30. Blue-green deployment

Run complete blue and green versions simultaneously. Test the inactive version, then switch the Service selector or load-balancer route atomically.

```text
Service -> blue
Deploy/test green
Service selector -> green
Retain blue briefly for rollback
```

Advantages are fast cutover and rollback. Disadvantages are double capacity and database compatibility requirements.

## 31. Canary deployment

Release the new version to a small share of users or traffic, compare SLIs, then increase gradually.

```text
95% stable + 5% canary -> 75/25 -> 50/50 -> 0/100
```

Use ingress/service-mesh weighted routing or a rollout controller. Define automated success metrics, pause points, abort thresholds, and rollback. Replica ratios alone may not produce exact traffic percentages.

## 32. What is CSI?

CSI means Container Storage Interface. It standardizes how Kubernetes integrates storage systems. A CSI driver handles operations such as provisioning, attaching, mounting, resizing, snapshotting, and deleting volumes, depending on capabilities. StorageClass, PVC, PV, and CSI drivers work together for dynamic provisioning.

---

# Part 3: AWS and EKS

## 33. AWS Lambda function and Step Functions

AWS Lambda executes event-driven code without managing servers. AWS Step Functions orchestrates workflows using a state machine with task, choice, parallel, map, wait, success, and failure states. Step Functions can coordinate Lambda and other AWS service integrations, retries, catches, timeouts, and execution history.

Use Lambda for one unit of compute and Step Functions for multi-step orchestration, long waits, branching, compensation, and auditability.

## 34. Autoscaling policies and uses

For EC2 Auto Scaling:

- Target tracking maintains a metric near a target value.
- Step scaling changes capacity based on alarm-breach magnitude.
- Simple scaling is older and uses cooldown-based single adjustments.
- Scheduled scaling handles known time patterns.
- Predictive scaling forecasts recurring demand.

Use lifecycle hooks, instance warmup, health checks, and minimum/maximum/desired capacity. For ECS and DynamoDB, Application Auto Scaling provides similar policy patterns.

## 35. What is a target probe in autoscaling?

The likely term is **target tracking policy**, not target probe. Target tracking adjusts capacity to keep a metric at a target, for example average CPU at 50% or ALB requests per target at a defined value.

Health probes/checks are separate. They determine whether an instance or target is healthy and eligible for traffic or replacement. Explain which one the interviewer means.

## 36. Which role should be given to access AWS services?

Use an IAM role with least-privilege policies, selected by workload type:

- EC2 instance profile for EC2 applications
- ECS task role for ECS tasks
- Lambda execution role for Lambda
- EKS Pod Identity or IRSA for individual Kubernetes service accounts
- Cross-account assumable role for access between accounts

Do not embed IAM user access keys in code, images, Pods, or configuration files.

## 37. Configure an IAM role for a Kubernetes service account

On EKS, use EKS Pod Identity or IAM Roles for Service Accounts.

IRSA flow:

1. Enable the EKS OIDC provider.
2. Create a least-privilege IAM policy.
3. Create a role whose trust policy allows the specific OIDC subject `system:serviceaccount:<namespace>:<serviceaccount>`.
4. Annotate the Kubernetes ServiceAccount with the role ARN.
5. Configure the Pod to use that ServiceAccount.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: reports
  namespace: finance
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/finance-reports
```

```yaml
spec:
  serviceAccountName: reports
```

Use one role per trust boundary and restrict both the trust policy and permissions policy.

## 38. Service account end-to-end flow

```text
Pod uses Kubernetes ServiceAccount
-> projected signed token
-> AWS SDK credential provider
-> STS validates OIDC trust and subject
-> temporary role credentials returned
-> SDK calls permitted AWS service
-> IAM evaluates policy
```

With EKS Pod Identity, an EKS-managed association maps the ServiceAccount to an IAM role and provides temporary credentials through the Pod Identity Agent. In either model, avoid static keys.

## 39. Create a Secret and ServiceAccount

A ServiceAccount provides workload identity. A Secret stores sensitive data. Modern Kubernetes does not automatically create a long-lived ServiceAccount token Secret; use projected short-lived tokens.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payment-api
  namespace: payments
---
apiVersion: v1
kind: Secret
metadata:
  name: payment-api-secret
  namespace: payments
type: Opaque
stringData:
  API_TOKEN: replace-through-secret-manager
```

```yaml
spec:
  serviceAccountName: payment-api
  containers:
    - name: app
      envFrom:
        - secretRef:
            name: payment-api-secret
```

Prefer an external secret manager, such as AWS Secrets Manager with a supported CSI/external-secrets integration, rather than committing secret YAML.

## 40. Static vs dynamic storage provisioning

- Static provisioning: an administrator creates PVs and backing storage before claims. PVCs bind to matching PVs.
- Dynamic provisioning: a PVC references a StorageClass; its CSI provisioner creates storage automatically.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data
spec:
  storageClassName: gp3
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 20Gi
```

Review reclaim policy, expansion, topology, encryption, snapshots, and backup.

## 41. Types of errors in Pod labels

Common label-related problems:

- Invalid label key/value syntax
- Deployment selector does not match Pod-template labels
- Service selector matches no Pods, producing no endpoints
- Selector accidentally matches the wrong Pods
- Immutable Deployment selector changed after creation
- Affinity expressions refer to nonexistent labels
- Duplicate or inconsistent environment labels

```bash
kubectl get pods --show-labels
kubectl get pods -l app=payment-api
kubectl describe deployment payment-api
kubectl get svc payment-api -o yaml
kubectl get endpointslices -l kubernetes.io/service-name=payment-api
```

---

# Part 4: Additional Capgemini Questions

## 42. NAT Gateway vs Internet Gateway

- Internet Gateway is a horizontally scaled VPC component enabling communication between a VPC and the internet. Public subnet routes point to it. A resource also needs a public IPv4/EIP or appropriate IPv6 design and permissive security controls.
- Public NAT Gateway lets private IPv4 resources initiate outbound internet connections while preventing unsolicited inbound connections. It resides in a public subnet with an EIP and routes to an IGW.

```text
Public EC2 -> IGW -> Internet
Private EC2 -> NAT Gateway in public subnet -> IGW -> Internet
```

For S3/DynamoDB, prefer gateway endpoints where suitable. For IPv6 outbound-only access, use an egress-only Internet Gateway.

## 43. Troubleshoot a Pending Pod

```bash
kubectl describe pod <pod> -n <namespace>
kubectl get events -n <namespace> --sort-by=.lastTimestamp
kubectl get nodes -o wide
kubectl describe nodes
kubectl get pvc -n <namespace>
kubectl get storageclass
```

Check:

- Insufficient CPU/memory/GPU or Pod count
- Requests too large
- Node selector/affinity mismatch
- Taints without tolerations
- Unbound PVC or storage topology conflict
- Pod anti-affinity/topology constraint impossible
- ResourceQuota or LimitRange admission problem
- Host port conflict
- Cluster Autoscaler at maximum or unable to satisfy constraints
- Unschedulable nodes, cordons, or node pressure

The scheduler event message usually gives the fastest clue.

## 44. Secondary RDS vs read-only RDS

The likely comparison is **Multi-AZ standby** versus **read replica**.

- Multi-AZ standby is primarily for high availability and automated failover. The standby is normally not used for application reads in the traditional DB instance architecture.
- Read replica is used for read scaling and can often be promoted. Replication is generally asynchronous, so lag can occur.
- Multi-AZ is not a read-scaling solution; a read replica is not automatically equivalent to synchronous HA.

Aurora and newer Multi-AZ DB cluster architectures have different reader capabilities, so identify the exact RDS deployment type.

## 45. Have you built or only managed RDS?

Answer honestly. A complete build includes:

- Requirements, engine/version, licensing, instance class, storage/IOPS
- DB subnet group across AZs
- Security groups and private DNS/connectivity
- Parameter and option groups
- KMS encryption, Secrets Manager credentials
- Multi-AZ, read replicas, backups and retention
- Maintenance, monitoring, Performance Insights/Database Insights
- Terraform/IaC, validation, migration, restore and failover tests

A strong response distinguishes what you designed, implemented, reviewed, and operated.

## 46. Allow only one user to access RDS at a time

Clarify the requirement. If it literally means one concurrent database connection, configure it at the database/application layer, not only in RDS networking.

Options:

- Create one least-privilege DB user and revoke others.
- Restrict network access to one application/host security group.
- Set a user-specific connection limit where supported, for example MySQL `MAX_USER_CONNECTIONS 1`.
- Use application-side concurrency control or locking if the business operation must be serialized.
- Do not set global `max_connections=1`; RDS monitoring, administration, replication, and failover operations need connections.

```sql
ALTER USER 'single_user'@'%' WITH MAX_USER_CONNECTIONS 1;
```

Test your exact engine/version syntax and behavior. A connection limit is different from allowing only one transaction or one human user.

## 47. Where should RDS be deployed?

Deploy RDS in private subnets through a DB subnet group spanning at least two Availability Zones. Do not make production RDS publicly accessible unless there is an approved exceptional requirement. Allow the database port only from the application security group or approved private sources. Use Multi-AZ, encryption, backups, DNS, monitoring, and controlled administrative access.

## 48. Upgrade RDS MySQL 7.0 to 8.0+

MySQL 7.0 is not a normal RDS MySQL major release, so first confirm the actual source version. For a major upgrade:

1. Review RDS-supported upgrade paths and engine release notes.
2. Check application, driver, SQL mode, authentication plugin, charset, removed feature, and parameter compatibility.
3. Take a manual snapshot and test its restoration.
4. Clone/restore to a non-production DB and perform a dry run.
5. Run engine-specific prechecks.
6. Upgrade replicas in the required order and plan replication impact.
7. Schedule a maintenance window or use an approved blue-green strategy where supported.
8. Upgrade, monitor events/logs, and execute smoke/regression tests.
9. Validate backups, replication, performance, and application behavior.
10. Preserve a rollback strategy. Major upgrades are not normally reversed in place; restore or cut back to the old environment.

## 49. Cross-account user access to resources

Use an IAM role in the resource account:

1. Create a role with least-privilege permissions.
2. Trust the external account or a specific role, not `*`.
3. Require an External ID for applicable third-party access and MFA where required.
4. Grant source identities permission to call `sts:AssumeRole`.
5. Use temporary STS credentials.
6. Add a resource policy if the service requires one.
7. Ensure SCPs, permissions boundaries, KMS policies, and endpoint policies allow access.

## 50. What about VPC connectivity for cross-account access?

IAM authorization alone does not create network connectivity. Depending on topology, use:

- VPC peering for simple non-transitive connectivity
- Transit Gateway for scalable multi-VPC/multi-account routing
- AWS PrivateLink for provider-consumer service access with reduced network exposure
- Site-to-Site VPN or Direct Connect for hybrid connectivity
- Shared VPC through AWS RAM for centrally managed networks

Update route tables, security groups, NACLs, DNS resolution, and endpoints. Ensure CIDRs do not overlap. Some AWS APIs can be reached publicly or by VPC endpoint and do not require VPC-to-VPC routing.

## 51. Migrate EC2 from one Region to another

An EC2 instance cannot simply be moved in place across Regions.

1. Quiesce the application and ensure data consistency.
2. Create an AMI or EBS snapshots.
3. Copy the AMI/snapshots to the target Region and re-encrypt with a target-Region KMS key if needed.
4. Recreate VPC, subnets, routes, security groups, IAM profile, keys, load balancer, and dependencies using IaC.
5. Launch and validate the target instance.
6. Replicate external data and update secrets/configuration.
7. Perform testing and controlled DNS cutover.
8. Monitor, retain rollback, then retire the source.

Use AWS Application Migration Service for larger migration programs where appropriate.

## 52. EC2 creation fails because IP addresses are exhausted

Check subnet capacity:

```bash
aws ec2 describe-subnets   --subnet-ids subnet-1234567890abcdef0   --query 'Subnets[0].[CidrBlock,AvailableIpAddressCount]'
```

AWS reserves five IPv4 addresses in each subnet. Investigate stale ENIs, load balancers, VPC endpoints, Lambda ENIs, NAT gateways, databases, and stopped resources retaining interfaces. Fix options:

- Delete genuinely unused ENIs/resources.
- Choose another subnet with capacity.
- Add a secondary VPC CIDR and create new subnets.
- Redesign subnet sizing and IP address management.
- Use IPv6 where supported.

## 53. Can an existing subnet CIDR be extended?

No. You cannot resize an existing AWS subnet CIDR in place. Add a secondary VPC CIDR if necessary, create a new larger subnet, migrate resources, and update routing and dependencies. Plan CIDRs with IPAM to prevent overlap and exhaustion.

## 54. Can instances in the new subnet communicate with old instances?

Yes, if both subnets are in the same VPC, the VPC local route covers their CIDRs and security groups/NACLs allow the traffic. If a middlebox, custom route, overlapping CIDR, appliance, DNS, or peering/transit design is involved, validate those paths separately.

## 55. Terraform provisioners

- `local-exec`: runs on the Terraform runner.
- `remote-exec`: runs commands on the remote resource.
- `file`: copies files to a remote resource.

Provisioners are a last resort because Terraform cannot model their side effects reliably. Prefer cloud-init/user data, images, configuration management, or provider-native resources. If used, add explicit connections, timeouts, safe secret handling, and idempotent scripts.

## 56. Remove a Terraform state lock

First prove no active `plan` or `apply` owns the lock. Then use the lock ID from the error:

```bash
terraform force-unlock LOCK_ID
```

For a local state lock, stop the owning Terraform process rather than deleting files blindly. For a remote backend, inspect the backend's lock mechanism and pipeline activity. Back up state before recovery. Never force-unlock merely because an operation is slow; concurrent writers can corrupt state.

## 57. Manage a console-created EC2 instance with Terraform

1. Write a matching `aws_instance` resource block.
2. Add an import block or run the CLI import.
3. Run `terraform plan`.
4. Reconcile arguments until no unintended change is proposed.
5. Move state to the approved remote backend and protect it.

```hcl
resource "aws_instance" "legacy" {
  ami           = "ami-xxxxxxxx"
  instance_type = "t3.medium"
}

import {
  to = aws_instance.legacy
  id = "i-0123456789abcdef0"
}
```

```bash
terraform plan
terraform apply
```

Traditional CLI form:

```bash
terraform import aws_instance.legacy i-0123456789abcdef0
```

Import binds the object to state; it does not guarantee that your HCL fully matches the resource.

## 58. What are Terraform resources?

A resource block declares infrastructure Terraform should create and manage.

```hcl
resource "aws_s3_bucket" "logs" {
  bucket = "example-company-logs"
}
```

- `aws_s3_bucket` is the resource type.
- `logs` is the local label.
- `aws_s3_bucket.logs` is its address.

Resources have arguments, computed attributes, dependencies, lifecycle behavior, and provider-specific identity.

## 59. Kubernetes Service types

- `ClusterIP`: internal virtual IP, default.
- `NodePort`: opens a port on each node and forwards to the Service.
- `LoadBalancer`: asks the cloud provider to provision an external/internal load balancer.
- `ExternalName`: returns a DNS CNAME and does not proxy traffic.
- Headless Service: `clusterIP: None`, returns endpoint addresses directly and supports discovery patterns for stateful systems.

Ingress and Gateway API are separate HTTP/network routing APIs, not Service types.

## 60. Prevent a Pod from scheduling on a particular node

Options:

- Taint that node and omit a matching toleration.
- Use required node affinity with `NotIn` or positive labels for acceptable nodes.
- Cordon the node for temporary maintenance.
- Use admission policy for organization-wide constraints.

```bash
kubectl taint node node01 blocked=payment:NoSchedule
kubectl cordon node01
```

For multi-cloud clusters or node pools, apply consistent provider/region/pool labels and use node affinity. Avoid deprecated direct `nodeName` control except for specialized cases.

## 61. PVC

A PersistentVolumeClaim is a namespace-scoped request for storage. It specifies capacity, access modes, StorageClass, and volume mode. It binds to a matching PV or triggers dynamic provisioning.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-data
spec:
  storageClassName: gp3
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```

Check `Pending` PVCs with:

```bash
kubectl describe pvc database-data
kubectl get pv,storageclass
kubectl get events --sort-by=.lastTimestamp
```

## 62. Terraform import vs data source

They solve different problems.

- `terraform import` brings an existing object under Terraform lifecycle management by binding it to a `resource` address in state. Terraform can then propose updates or deletion according to configuration.
- A `data` source reads information about an existing object without managing its lifecycle.

```hcl
# Read an existing VPC but do not manage it
data "aws_vpc" "shared" {
  tags = {
    Name = "shared-vpc"
  }
}

resource "aws_subnet" "app" {
  vpc_id     = data.aws_vpc.shared.id
  cidr_block = "10.20.10.0/24"
}
```

Use import when your Terraform configuration should own the existing EC2 instance. Use a data source when another team or stack owns the VPC and your code only needs its ID or attributes.

---

# Rapid Revision

- Ansible is agentless for Linux and has no mandatory daemon on managed nodes.
- Roles package reusable tasks, handlers, defaults, files, and templates.
- Encrypt secret variables with Ansible Vault, not necessarily the whole playbook.
- Use `include_tasks` dynamically and `import_tasks` statically.
- Kubernetes controllers continuously reconcile desired and actual state.
- Affinity attracts/selects; anti-affinity spreads; taints repel; tolerations permit.
- Requests affect scheduling; limits constrain usage; quotas control namespaces.
- Multi-AZ RDS is for HA; read replicas are primarily for read scaling.
- IAM authorization and VPC connectivity are separate requirements.
- AWS subnet CIDRs cannot be expanded in place.
- `terraform force-unlock` is only for a confirmed stale lock.
- Import manages an existing object; a data source only reads it.

---

# Official Documentation

- Ansible documentation: https://docs.ansible.com/projects/ansible/latest/index.html
- Ansible roles: https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_reuse_roles.html
- Ansible Vault: https://docs.ansible.com/projects/ansible/latest/vault_guide/index.html
- Kubernetes architecture: https://kubernetes.io/docs/concepts/architecture/
- Kubernetes Services: https://kubernetes.io/docs/concepts/services-networking/service/
- Kubernetes scheduling: https://kubernetes.io/docs/concepts/scheduling-eviction/
- Kubernetes storage: https://kubernetes.io/docs/concepts/storage/
- AWS EC2 Auto Scaling: https://docs.aws.amazon.com/autoscaling/ec2/userguide/what-is-amazon-ec2-auto-scaling.html
- Amazon RDS: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Welcome.html
- Terraform import: https://developer.hashicorp.com/terraform/language/import
- Terraform data sources: https://developer.hashicorp.com/terraform/language/data-sources
- Terraform provisioners: https://developer.hashicorp.com/terraform/language/resources/provisioners/syntax
- Terraform force-unlock: https://developer.hashicorp.com/terraform/cli/commands/force-unlock
