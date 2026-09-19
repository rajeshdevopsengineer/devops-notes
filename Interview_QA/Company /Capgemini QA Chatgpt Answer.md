Below are detailed answers in your original order, with examples and practical considerations appropriate for **nine years of overall experience, including five years in DevOps**. Example hostnames, account IDs, image versions, and resource names should be replaced with your environment’s values.

**1. What is an Ansible playbook?**

An Ansible playbook is a YAML file that describes automation to execute against a set of hosts. It contains one or more **plays**, and each play maps hosts from an inventory to tasks.

A task normally invokes a module—for example, to install a package, create a user, deploy a configuration file, or manage a service.

```yaml
---
- name: Configure web servers
  hosts: web
  become: true

  tasks:
    - name: Install nginx
      ansible.builtin.package:
        name: nginx
        state: present

    - name: Start nginx and enable it at boot
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true
```

Execute it with:

```bash
ansible-playbook -i inventory.ini web.yml
```

An important concept is **idempotency**: rerunning a task should leave the system in the same desired state without making unnecessary changes. Many Ansible modules provide this behavior, but arbitrary shell commands do not automatically become idempotent. [Ansible playbooks documentation](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_intro.html)

---

**2. How do you install Ansible on Ubuntu and Red Hat, and start services?**

Install Ansible on the **control node**. Linux managed nodes generally require SSH access and a compatible Python interpreter; they do not need Ansible installed.

On Ubuntu, the documented PPA installation is:

```bash
sudo apt update
sudo apt install -y software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install -y ansible

ansible --version
```

On RHEL, with the appropriate subscribed repositories enabled:

```bash
sudo dnf install -y ansible-core

ansible --version
```

`ansible-core` supplies the automation engine and built-in modules. Additional collections must be installed separately, using versions compatible with the installed engine and Python.

**There is no Ansible daemon to start.** Ansible normally runs when you invoke its commands. Services such as SSH must be available on the managed hosts:

```bash
# Ubuntu managed host
sudo systemctl enable --now ssh

# RHEL managed host
sudo systemctl enable --now sshd
```

Verify connectivity:

```bash
ansible all -i inventory.ini -m ansible.builtin.ping
```

Ansible’s `ping` tests Ansible connectivity and Python execution; it is not an ICMP ping. Installation and support details depend on the distribution and repository. [Ansible installation instructions](https://docs.ansible.com/projects/ansible/latest/installation_guide/installation_distros.html), [Red Hat control-node and managed-node guidance](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html-single/automating_system_administration_by_using_rhel_system_roles/index)

---

**3. How do you create three users and map them to prod, task, and QA groups in a single task?**

Use the `user` module with a loop:

```yaml
- name: Create users and assign their supplementary groups
  become: true
  ansible.builtin.user:
    name: "{{ item.name }}"
    groups: "{{ item.group }}"
    append: true
    create_home: true
    state: present
  loop:
    - { name: "produser", group: "prod" }
    - { name: "taskuser", group: "task" }
    - { name: "qauser", group: "QA" }
```

This is **one task definition**, executed once for each loop item.

The groups must already exist. If they do not, create them beforehand with `ansible.builtin.group`; a single `user` module task does not also create arbitrary supplementary groups.

Two useful distinctions:

* `groups` specifies supplementary groups. `append: true` preserves existing memberships.
* `group` specifies the primary group.

Use the exact group name and case configured on the operating system. [Ansible user module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/user_module.html)

---

**4. How do you include content and input parameters in a playbook?**

Ansible supports several mechanisms:

| Requirement                   | Mechanism                       |
| ----------------------------- | ------------------------------- |
| Variables defined in the play | `vars`                          |
| Variables loaded from files   | `vars_files` or `include_vars`  |
| Interactive input             | `vars_prompt`                   |
| Command-line input            | `--extra-vars` or `-e`          |
| Dynamically load tasks        | `include_tasks`                 |
| Statically import tasks       | `import_tasks`                  |
| Include reusable automation   | `include_role` or `import_role` |

For example:

```yaml
---
- name: Deploy an application
  hosts: web

  vars_files:
    - vars/common.yml

  vars_prompt:
    - name: app_version
      prompt: Enter the application version
      private: false

  tasks:
    - name: Load deployment tasks
      ansible.builtin.include_tasks: tasks/deploy.yml
      vars:
        release_version: "{{ app_version }}"
```

The included tasks can reference `release_version`.

For unattended execution:

```bash
ansible-playbook -i inventory.ini deploy.yml \
  -e '{"app_version":"2.4.0"}'
```

Or supply a variable file:

```bash
ansible-playbook -i inventory.ini deploy.yml \
  -e @vars/prod.yml
```

**Imports are processed statically; includes are processed dynamically at execution time.** Includes are useful when task selection depends on runtime information or loops. Extra variables have the highest variable precedence, so use them deliberately. [Reusing Ansible content](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_reuse.html), [Ansible variables](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_variables.html)

---

**5. What are Ansible roles?**

A role packages related automation into a reusable directory structure. For example, an `nginx` role might install nginx, render its configuration, and restart or reload the service when necessary.

| Role path           | Purpose                             |
| ------------------- | ----------------------------------- |
| `tasks/main.yml`    | Main task entry point               |
| `handlers/main.yml` | Handlers triggered by notifications |
| `defaults/main.yml` | Easily overridden default variables |
| `vars/main.yml`     | Higher-precedence role variables    |
| `templates/`        | Jinja2 templates                    |
| `files/`            | Files copied without templating     |
| `meta/main.yml`     | Metadata and role dependencies      |

Use a role like this:

```yaml
---
- name: Configure web servers
  hosts: web
  become: true

  roles:
    - role: nginx
      nginx_port: 8080
```

Roles improve reuse, testing, ownership, and consistency across environments. Put configurable values in `defaults/main.yml` so callers can override them easily.

An Ansible role is an automation package; it does not represent an access-control role. [Ansible roles](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_reuse_roles.html)

---

**6. What are templates used for in Ansible?**

Templates generate configuration files using variables, conditions, and loops. Ansible uses the **Jinja2** templating language.

For example, `templates/site.conf.j2`:

```jinja2
server {
    listen {{ nginx_port }};
    server_name {{ server_name }};

    location / {
        proxy_pass http://{{ backend_host }}:{{ backend_port }};
    }
}
```

A play can render it and reload nginx only when the resulting file changes:

```yaml
tasks:
  - name: Deploy nginx site configuration
    ansible.builtin.template:
      src: site.conf.j2
      dest: /etc/nginx/conf.d/site.conf
      owner: root
      group: root
      mode: "0644"
    notify: Reload nginx

handlers:
  - name: Reload nginx
    ansible.builtin.service:
      name: nginx
      state: reloaded
```

Typical uses include web-server configuration, application properties, monitoring configuration, and database settings.

Use `copy` for an unchanged file and `template` when its contents must vary. Templates are rendered on the controller before transfer to the managed host. [Ansible template module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/template_module.html)

---

**7. What is the difference between templates and roles?**

| Aspect           | Template                              | Role                                                    |
| ---------------- | ------------------------------------- | ------------------------------------------------------- |
| Purpose          | Generate a file with variable content | Organize reusable automation                            |
| Typical contents | Jinja2 text                           | Tasks, handlers, variables, templates, files            |
| Example          | `nginx.conf.j2`                       | An entire nginx installation and configuration workflow |
| Execution        | Rendered through the template module  | Invoked from a playbook or another role                 |

They work together: a role commonly contains several templates.

For example, a MySQL role could install packages, create directories, render `my.cnf`, and notify a restart handler. The template handles only the configuration-file generation.

---

**8. How do you initialize a role and publish it to Ansible Galaxy?**

Initialize the role:

```bash
ansible-galaxy role init webserver
```

Then implement the tasks, defaults, handlers, and templates. Update `meta/main.yml` with the role’s author, description, license, supported platforms, and other required metadata.

Add a README explaining variables and usage, and test the role against the platforms you claim to support.

For a standalone role, push it to a public GitHub repository:

```bash
cd webserver

git init
git add .
git commit -m "Initial webserver role"
git branch -M main
git tag 1.0.0

git remote add origin \
  https://github.com/YOUR_USER/ansible-role-webserver.git

git push -u origin main
git push origin 1.0.0
```

After configuring Galaxy authentication, import it:

```bash
ansible-galaxy role import YOUR_USER ansible-role-webserver \
  --role-name webserver
```

Consumers can install an imported version:

```bash
ansible-galaxy role install YOUR_NAMESPACE.webserver,1.0.0
```

**Role import and collection publishing are different workflows.** If distributing roles inside a collection, initialize the collection, build its artifact, and publish it with `ansible-galaxy collection publish`. [Ansible Galaxy CLI](https://docs.ansible.com/projects/ansible/latest/cli/ansible-galaxy.html)

---

**9. How do you encrypt a playbook?**

Use **Ansible Vault**:

```bash
ansible-vault encrypt playbook.yml
```

Other useful operations are:

```bash
# Create an encrypted variables file
ansible-vault create secrets.yml

# Edit encrypted content
ansible-vault edit secrets.yml

# View without permanently decrypting the file
ansible-vault view secrets.yml

# Change the encryption password
ansible-vault rekey secrets.yml
```

Although you can encrypt the entire playbook, a common approach is to encrypt only sensitive variables. That keeps automation logic readable and reviewable in Git.

Vault protects content **at rest**. It does not automatically prevent a task from printing a decrypted value, so sensitive tasks may also require `no_log: true`. [Encrypting content with Ansible Vault](https://docs.ansible.com/projects/ansible/latest/vault_guide/vault_encrypting_content.html)

---

**10. How do you execute a Vault file?**

If the encrypted file is a playbook:

```bash
ansible-playbook -i inventory.ini encrypted-playbook.yml \
  --ask-vault-pass
```

If the encrypted file contains variables, load it from a normal playbook:

```yaml
---
- name: Use encrypted credentials
  hosts: db

  vars_files:
    - secrets.yml

  tasks:
    # Tasks reference variables from secrets.yml
```

Then run:

```bash
ansible-playbook -i inventory.ini database.yml \
  --ask-vault-pass
```

You can also use a named Vault identity:

```bash
ansible-playbook -i inventory.ini database.yml \
  --vault-id prod@prompt
```

Ansible decrypts the content when needed during execution. You do not need to permanently decrypt the file first.

An encrypted **variables file is not executable by itself**; it supplies data to tasks in a playbook. [Using encrypted variables and files](https://docs.ansible.com/projects/ansible/latest/vault_guide/vault_using_encrypted_content.html)

---

**11. How do you use an environment variable for passwords?**

For an application password, read an environment variable from the controller:

```yaml
vars:
  db_password: >-
    {{ lookup('ansible.builtin.env', 'DB_PASSWORD', default=undef()) }}
```

For interactive use:

```bash
read -rsp "Database password: " DB_PASSWORD
export DB_PASSWORD

ansible-playbook -i inventory.ini database.yml

unset DB_PASSWORD
```

The lookup reads the **controller’s environment**, not the remote host’s environment. The `environment:` playbook keyword serves a different purpose: it sets environment variables for remote task execution. [Ansible environment lookup](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/env_lookup.html)

For a **Vault password**, an executable password-provider script can read a variable injected by your CI secret store:

```python
#!/usr/bin/env python3
import os
import sys

password = os.environ.get("ANSIBLE_VAULT_PASSWORD")
if not password:
    sys.exit("ANSIBLE_VAULT_PASSWORD is not set")

sys.stdout.write(password + "\n")
```

Save it as `vault-pass.py` and run:

```bash
chmod 700 vault-pass.py

ansible-playbook -i inventory.ini database.yml \
  --vault-password-file ./vault-pass.py
```

Here, `ANSIBLE_VAULT_PASSWORD` is a convention used by your script; setting it alone does not automatically configure Ansible Vault. Avoid logging the environment or decrypted credentials. [Managing Vault passwords](https://docs.ansible.com/projects/ansible/latest/vault_guide/vault_managing_passwords.html)

---

**12. How do you create a MySQL dump using encrypted credentials?**

Assuming “encrypted data” means **database credentials stored in Vault**, the workflow is: create the encrypted variables, load them in the playbook, execute the dump, and verify the backup.

Current Ansible documentation uses `ansible.mysql.mysql_db`; older environments commonly use `community.mysql.mysql_db`. Install a collection version compatible with your Ansible environment. [MySQL collection naming change](https://docs.ansible.com/projects/ansible/latest/collections/community/mysql/mysql_db_module.html)

Install the collection:

```bash
ansible-galaxy collection install ansible.mysql
```

On the host executing the dump, install the MySQL client utilities, `mysqldump`, gzip, and PyMySQL for Ansible’s Python interpreter.

Create the encrypted variables:

```bash
ansible-vault create vault.yml
```

Enter:

```yaml
mysql_backup_user: backup_user
mysql_backup_password: replace_with_actual_password
```

Create `backup.yml`:

```yaml
---
- name: Back up MySQL databases
  hosts: db
  become: true
  gather_facts: false

  vars_files:
    - vault.yml

  vars:
    database_name: appdb
    backup_directory: /var/backups/mysql

  tasks:
    - name: Create a private backup directory
      ansible.builtin.file:
        path: "{{ backup_directory }}"
        state: directory
        owner: root
        group: root
        mode: "0700"

    - name: Allocate a unique backup file
      ansible.builtin.tempfile:
        state: file
        path: "{{ backup_directory }}"
        prefix: "{{ database_name }}-"
        suffix: .sql.gz
      register: backup_file

    - name: Create the database dump
      ansible.mysql.mysql_db:
        name: "{{ database_name }}"
        state: dump
        target: "{{ backup_file.path }}"
        login_host: 127.0.0.1
        login_user: "{{ mysql_backup_user }}"
        login_password: "{{ mysql_backup_password }}"
        config_file: ""
        single_transaction: true
        quick: true
        pipefail: true
      no_log: true

    - name: Restrict backup permissions
      ansible.builtin.file:
        path: "{{ backup_file.path }}"
        owner: root
        group: root
        mode: "0600"

    - name: Check gzip integrity
      ansible.builtin.command:
        argv:
          - gzip
          - -t
          - "{{ backup_file.path }}"
      changed_when: false
```

Execute:

```bash
ansible-playbook -i inventory.ini backup.yml --ask-vault-pass
```

The dump is written on each targeted database server. `single_transaction` is useful for transactional tables such as InnoDB; it does not provide the same consistency guarantee for nontransactional tables or arbitrary concurrent DDL. `pipefail` helps detect dump failures during compression. [MySQL database module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/mysql/mysql_db_module.html)

For production, also handle partial files on failure, backup retention, secure off-host storage, and restore testing. A gzip check is not a restore test.

**Encrypting the credentials does not encrypt the dump.** Database encryption at rest also does not imply that a logical SQL export is encrypted.

---

**13. How do you execute on all database servers except one?**

Use an inventory exclusion pattern:

```bash
ansible-playbook -i inventory.ini backup.yml \
  --limit 'db:!db03' \
  --ask-vault-pass
```

Here:

* `db` selects the database group.
* `!db03` excludes the inventory host named `db03`.

You can put the same pattern in a play:

```yaml
hosts: "db:!db03"
```

Preview the selection:

```bash
ansible 'db:!db03' -i inventory.ini --list-hosts
```

Use the host’s **inventory name**, which may differ from its IP address or DNS name. Quote patterns containing `!` to avoid shell interpretation. [Ansible host patterns](https://docs.ansible.com/projects/ansible/latest/inventory_guide/intro_patterns.html)

---

**14. How do you shut down all servers using an ad-hoc command?**

Using the shutdown module:

```bash
ansible-galaxy collection install community.general

ansible all -i inventory.ini --become \
  -m community.general.shutdown \
  -a "delay=60"
```

Or using the operating-system command:

```bash
ansible all -i inventory.ini --become \
  -m ansible.builtin.command \
  -a "/sbin/shutdown -h +1"
```

The second command schedules shutdown one minute later, allowing the remote command to return first.

`all` means all hosts in the supplied inventory. In a production procedure, coordinate application draining and database shutdown ordering before this fleet-wide step. [Ansible shutdown module](https://docs.ansible.com/projects/ansible/latest/collections/community/general/shutdown_module.html)

---

**15. How do you reduce execution time on an RDBMS server?**

This question has two possible meanings.

**For Ansible execution against database servers:**

* Disable fact gathering when facts are unnecessary.
* Increase `forks` for independent work across multiple hosts.
* Use SSH pipelining to reduce connection overhead.
* Use `strategy: free` when hosts can progress independently.
* Use asynchronous execution for suitable long-running operations.
* Skip unnecessary tasks using conditions and tags.

Increasing forks improves concurrency across hosts; it does not accelerate one SQL statement. Likewise, asynchronous execution changes how Ansible waits, rather than making the database operation inherently faster. [Ansible execution strategies](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_strategies.html)

**For database execution itself**, first identify the bottleneck: query plan, indexing, locking, CPU, memory, storage latency, or network transfer. The relevant optimization depends on that evidence.

For replicated or clustered databases, concurrency must also respect replication, quorum, and maintenance requirements.

---

**16. How do you configure those execution improvements in Ansible files?**

Set controller-wide concurrency and SSH pipelining in `ansible.cfg`:

```ini
[defaults]
forks = 20

[ssh_connection]
pipelining = True
```

Use playbook settings for host independence and asynchronous execution:

```yaml
---
- name: Run independent database backup jobs
  hosts: db
  become: true
  gather_facts: false
  strategy: free

  tasks:
    - name: Start the existing backup script
      ansible.builtin.command: /usr/local/sbin/backup-db
      async: 3600
      poll: 0
      register: backup_job

    # Other independent tasks can run here.

    - name: Wait for the backup to finish
      ansible.builtin.async_status:
        jid: "{{ backup_job.ansible_job_id }}"
      register: backup_result
      until: backup_result.finished
      retries: 360
      delay: 10

    - name: Remove the async status file
      ansible.builtin.async_status:
        jid: "{{ backup_job.ansible_job_id }}"
        mode: cleanup
```

This assumes the backup script already exists.

`poll: 0` launches the task without waiting. `async_status` subsequently checks completion and surfaces failures. Do not start overlapping operations that contend for the same exclusive database or package-manager locks. [Asynchronous execution](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_async.html)

For rolling database maintenance, `serial: 1` may be appropriate even though it reduces concurrency.

---

**17. How do you increase the debug log level?**

Increase command-line verbosity:

```bash
ansible-playbook -i inventory.ini site.yml -v
ansible-playbook -i inventory.ini site.yml -vv
ansible-playbook -i inventory.ini site.yml -vvv
ansible-playbook -i inventory.ini site.yml -vvvv
```

Higher levels expose progressively more execution and connection information. `-vvvv` is particularly useful for SSH troubleshooting.

You can configure persistent logging:

```ini
[defaults]
verbosity = 2
log_path = ./ansible.log
```

For Ansible’s internal debugging:

```bash
ANSIBLE_DEBUG=1 ansible-playbook -i inventory.ini site.yml
```

The `debug` module has a separate purpose: printing selected information from a playbook.

```yaml
- name: Show the deployment destination at higher verbosity
  ansible.builtin.debug:
    var: deployment_directory
    verbosity: 2
```

Protect log files and avoid printing secrets, particularly when enabling internal debug output. [Ansible playbook CLI](https://docs.ansible.com/projects/ansible/latest/cli/ansible-playbook.html)

---

**18. What is the difference between static and dynamic inventory?**

| Aspect       | Static inventory                        | Dynamic inventory                              |
| ------------ | --------------------------------------- | ---------------------------------------------- |
| Source       | Maintained INI or YAML                  | External system queried by a plugin or script  |
| Updates      | Edited manually or generated separately | Discovered when inventory is loaded            |
| Good fit     | Stable host lists                       | Cloud instances and frequently changing fleets |
| Dependencies | Inventory file                          | API access, credentials, plugin dependencies   |
| Main concern | Stale entries                           | API failures, permissions, cache freshness     |

Static example:

```ini
[db]
db01 ansible_host=10.0.1.10
db02 ansible_host=10.0.1.11
db03 ansible_host=10.0.1.12
```

AWS dynamic inventory example, saved as `prod.aws_ec2.yml`:

```yaml
plugin: amazon.aws.aws_ec2

regions:
  - us-east-1

filters:
  instance-state-name: running
  tag:Environment: prod

hostnames:
  - private-ip-address

keyed_groups:
  - key: tags.Environment
    prefix: env
```

Inspect the discovered inventory:

```bash
ansible-inventory -i prod.aws_ec2.yml --graph
```

This requires the AWS collection, its controller-side Python dependencies, and suitable AWS read permissions. Dynamic inventory does not necessarily refresh continuously during a running playbook. [Dynamic inventory](https://docs.ansible.com/projects/ansible/latest/inventory_guide/intro_dynamic_inventory.html), [AWS EC2 inventory plugin](https://docs.ansible.com/projects/ansible/latest/collections/amazon/aws/aws_ec2_inventory.html)

---

**19. What is the architecture of Kubernetes?**

Kubernetes consists of a **control plane** that manages cluster state and **worker nodes** that run workloads.

| Component                 | Responsibility                                           |
| ------------------------- | -------------------------------------------------------- |
| API server                | Entry point for Kubernetes API requests                  |
| etcd                      | Stores cluster configuration and state                   |
| Scheduler                 | Selects suitable nodes for unscheduled Pods              |
| Controller manager        | Runs controllers that reconcile actual and desired state |
| Cloud controller manager  | Integrates with cloud resources where applicable         |
| kubelet                   | Ensures assigned Pod containers run on a node            |
| Container runtime         | Runs containers, commonly through containerd or CRI-O    |
| kube-proxy or replacement | Implements Service networking                            |
| CNI networking plugin     | Provides Pod networking                                  |

A typical Deployment workflow is:

1. `kubectl` submits the Deployment to the API server.
2. The Deployment controller manages a ReplicaSet.
3. The ReplicaSet controller creates Pod objects.
4. The scheduler assigns those Pods to nodes.
5. Each node’s kubelet asks the runtime to start the containers.

The scheduler selects placement; the kubelet performs node-level execution. [Kubernetes components](https://kubernetes.io/docs/concepts/overview/components/)

---

**20. What does `kubectl apply` do, and how are Services used?**

`kubectl apply` creates or updates Kubernetes resources from declarative configuration:

```bash
kubectl apply -f deployment.yml
kubectl apply -f service.yml
```

It submits the desired configuration; controllers then work toward that state. A successful apply does not mean the application is already healthy. [kubectl apply](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_apply/)

A Service provides a stable network identity for a changing set of Pods:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: apps
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
    - name: http
      port: 80
      targetPort: http
```

Here:

* Clients connect to Service port `80`.
* `targetPort: http` references a named container port.
* The selector identifies Pods labeled `app: web`.
* EndpointSlices track eligible backends.

Creating a Deployment does **not** automatically create a Service. Services and Deployments are separate resources. [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)

---

**21. Can you explain a Deployment YAML file?**

Assuming the `apps` namespace exists:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: web
  namespace: apps

spec:
  replicas: 3

  selector:
    matchLabels:
      app: web

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0

  template:
    metadata:
      labels:
        app: web

    spec:
      containers:
        - name: nginx
          image: nginx:1.28

          ports:
            - name: http
              containerPort: 80

          resources:
            requests:
              cpu: "100m"
              memory: "64Mi"
            limits:
              cpu: "500m"
              memory: "128Mi"

          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 3
            periodSeconds: 5

          livenessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 10
            periodSeconds: 10
```

The Deployment maintains three replicas. During an update, it permits one additional Pod and requests zero unavailable replicas.

Its selector must match the Pod template’s labels. The readiness probe controls readiness for traffic; repeated liveness failures trigger container restarts. The image tag is illustrative—production releases should use an approved, reproducible image reference.

Deploy and inspect:

```bash
kubectl apply -f deployment.yml
kubectl rollout status deployment/web -n apps
kubectl get pods -n apps -l app=web
```

Roll back a previous rollout:

```bash
kubectl rollout undo deployment/web -n apps
```

`containerPort` documents the container’s listening port; it does not itself expose the application externally. [Deployment documentation](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

---

**22. What are labels and annotations?**

Both are metadata attached to Kubernetes objects.

| Labels                                | Annotations                                                |
| ------------------------------------- | ---------------------------------------------------------- |
| Identify and group objects            | Store additional descriptive or tool-specific metadata     |
| Used by selectors                     | Not used by standard label selectors                       |
| Examples: app, environment, component | Examples: build URL, description, controller configuration |

```yaml
metadata:
  labels:
    app: payments
    environment: prod
    component: api

  annotations:
    example.com/build-url: "https://ci.example.com/builds/125"
    example.com/description: "Payments API"
```

Find Pods using labels:

```bash
kubectl get pods -l app=payments,environment=prod
```

Both keys and values must be strings. Quote values that YAML might interpret as numbers or booleans.

Annotations can influence a controller’s behavior, but only when that controller recognizes the annotation. [Kubernetes annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/)

---

**23. How does traffic from outside reach a Kubernetes cluster?**

Common mechanisms are:

| Mechanism              | Typical use                                                           |
| ---------------------- | --------------------------------------------------------------------- |
| `NodePort` Service     | Expose an application on a port of eligible nodes                     |
| `LoadBalancer` Service | Request an external load balancer through an installed implementation |
| Ingress                | HTTP/HTTPS host and path routing                                      |
| Gateway API            | More expressive routing, including traffic splitting                  |

For an HTTP application, a common logical path is:

**Client → external load balancer → Ingress/Gateway implementation → selected application backends.**

Services identify those backends. The actual data path is implementation-dependent: some controllers route directly to Pod IPs rather than sending packets through the Service’s ClusterIP.

Ingress and Gateway resources require a compatible controller. Creating the resource alone does not install a proxy or provision all required infrastructure.

When troubleshooting, inspect DNS, load-balancer health, listeners, routing rules, Service selectors, EndpointSlices, Pod readiness, and network policies. [Kubernetes Services and load balancers](https://kubernetes.io/docs/concepts/services-networking/service/)

---

**24. What is the controller manager’s task?**

The correct component name is **kube-controller-manager**.

It runs controllers that repeatedly compare actual cluster state with the desired state and request changes through the API server.

Examples include:

* ReplicaSet controller: maintains the requested Pod count.
* Deployment controller: manages ReplicaSets and rollouts.
* Job controller: manages Pods for finite work.
* Node controller: observes node availability.
* Other controllers: handle namespaces, service accounts, endpoints, and related lifecycle operations.

Suppose a ReplicaSet needs three Pods and one disappears. Its controller creates a replacement Pod object. The scheduler assigns that Pod to a node, and the kubelet starts its containers.

The controller manager coordinates reconciliation; it does not directly start containers or choose their nodes. [Kubernetes controllers](https://kubernetes.io/docs/concepts/architecture/controller/)

---

**25. What are node affinity and anti-affinity?**

**Node affinity** selects nodes using node labels. **Pod affinity and Pod anti-affinity** use the placement and labels of other Pods.

| Mechanism         | Example                                    |
| ----------------- | ------------------------------------------ |
| Node affinity     | Place a workload on SSD-equipped nodes     |
| Pod affinity      | Place related workloads near each other    |
| Pod anti-affinity | Separate application replicas across nodes |

Two common rule strengths are:

* `requiredDuringSchedulingIgnoredDuringExecution`: a scheduling requirement.
* `preferredDuringSchedulingIgnoredDuringExecution`: a scheduling preference.

Example inside a Pod specification:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: disktype
              operator: In
              values:
                - ssd

  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: payments
        topologyKey: kubernetes.io/hostname
```

This requires an SSD-labeled node and avoids nodes already hosting matching `payments` Pods within the rule’s namespace scope.

There is no separate `nodeAntiAffinity` field. Use node-affinity operators such as `NotIn` or `DoesNotExist` to exclude nodes.

`IgnoredDuringExecution` means a later label change does not itself evict an already-running Pod. Hard rules can leave Pods Pending when no node satisfies them. [Assigning Pods to nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)

---

**26. How do you run two Pods when one depends on the other? Do you set roles?**

**Kubernetes RBAC roles control permissions; they do not control application startup order.**

For two independent workloads:

1. Expose the dependency through a Service.
2. Give it an appropriate readiness probe.
3. Make the dependent application retry connections with backoff.
4. If initial startup must wait, use an init container.

For example, a frontend can wait for the backend’s readiness endpoint:

```yaml
initContainers:
  - name: wait-for-backend
    image: busybox:1.36
    command:
      - sh
      - -c
      - |
        for attempt in $(seq 1 60); do
          if wget -q -T 2 -O /dev/null http://backend:8080/ready; then
            exit 0
          fi
          sleep 2
        done
        exit 1
```

The frontend’s regular containers start after the init container succeeds. This does not control when the backend Pod is created.

Application retries remain necessary because the dependency can fail again after startup. A readiness probe controls traffic eligibility, not the creation order of separate workloads. [Kubernetes init containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)

---

**27. What are taints and tolerations?**

A **taint** is applied to a node to repel Pods. A **toleration** is applied to a Pod to permit scheduling despite a matching taint.

Taint a node:

```bash
kubectl taint nodes worker-1 dedicated=database:NoSchedule
```

Matching Pod toleration:

```yaml
tolerations:
  - key: dedicated
    operator: Equal
    value: database
    effect: NoSchedule
```

| Effect             | Behavior                                         |
| ------------------ | ------------------------------------------------ |
| `NoSchedule`       | Prevents new non-tolerating Pods from scheduling |
| `PreferNoSchedule` | Tries to avoid non-tolerating Pods               |
| `NoExecute`        | Also evicts non-tolerating existing Pods         |

A toleration makes the node eligible; it does not force the Pod onto that node. Combine it with node affinity when a workload must use a particular node group.

Remove the taint:

```bash
kubectl taint nodes worker-1 dedicated=database:NoSchedule-
```

[Kubernetes taints and tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)

---

**28. Why are worker nodes tainted?**

Worker nodes can be tainted for deliberate workload placement or because of operational conditions.

Common reasons include:

* Reserving GPU nodes for workloads that need GPUs.
* Dedicating nodes to databases or particular tenants.
* Separating workloads with different operational requirements.
* Preventing new scheduling onto nodes experiencing memory or disk pressure.
* Handling unavailable or unhealthy nodes.

Inspect the reason:

```bash
kubectl describe node worker-1

kubectl get node worker-1 \
  -o jsonpath='{.spec.taints}'
```

For condition-related taints, investigate the underlying node condition before removing the taint. For example, disk pressure requires checking filesystem usage, image cleanup, and available inodes.

Node-pressure eviction is a separate kubelet mechanism that can terminate Pods to reclaim resources; adding tolerations does not eliminate resource pressure. [Node-pressure eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)

---

**29. How do you limit resources in Kubernetes?**

Set resource requests and limits on containers:

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"

  limits:
    cpu: "1000m"
    memory: "512Mi"
```

* **Requests** influence scheduling and resource allocation.
* **CPU limits** typically result in throttling when exceeded.
* **Memory limits** can result in an OOM kill when the container cannot stay within its limit.
* `1000m` equals one CPU unit.

At namespace level:

* **ResourceQuota** limits aggregate resource consumption or object counts.
* **LimitRange** provides defaults and enforces per-object constraints.

Example quota:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: application-quota
  namespace: apps
spec:
  hard:
    requests.cpu: "8"
    requests.memory: 16Gi
    limits.cpu: "16"
    limits.memory: 32Gi
    pods: "30"
```

Requests should reflect measured workload needs. Excessive requests waste schedulable capacity; requests that are too low can cause contention and misleading utilization-based autoscaling. [Kubernetes resource management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)

---

**30. What is a Blue-Green deployment?**

Blue-Green deployment maintains two separately identifiable application versions:

* **Blue:** current production version.
* **Green:** new version prepared and tested alongside Blue.

The production Service initially selects Blue:

```yaml
selector:
  app: web
  track: blue
```

After validating Green, change the selector:

```bash
kubectl patch service web -n apps --type merge \
  -p '{"spec":{"selector":{"app":"web","track":"green"}}}'
```

A rollback switches the selector back to Blue.

Each Deployment should have a distinct selector, such as `app=web,track=blue` and `app=web,track=green`, to avoid controller overlap.

This provides a quick traffic switch, but routing changes take time to propagate, and existing connections may remain active. Database changes must remain compatible with both application versions during the transition.

Blue-Green is a deployment pattern built using multiple workloads and routing controls. It is not a native value for a Deployment’s `strategy.type`.

---

**31. What is a Canary deployment?**

A Canary deployment releases a new version to a small portion of traffic, evaluates its behavior, and gradually increases exposure.

For example:

**5% → 20% → 50% → 100%**

At each stage, evaluate application errors, latency, saturation, and relevant business metrics.

A Gateway API route can express weighted routing between two Services:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web
  namespace: apps
spec:
  parentRefs:
    - name: public-gateway

  rules:
    - backendRefs:
        - name: web-stable
          port: 80
          weight: 90

        - name: web-canary
          port: 80
          weight: 10
```

This requires Gateway API resources and a compatible implementation. The two Services must select the corresponding application versions. [Gateway API traffic splitting](https://gateway-api.sigs.k8s.io/guides/user-guides/traffic-splitting/)

Using nine stable Pods and one canary Pod behind a normal Service does **not** guarantee an exact 90/10 request split. Connection reuse and routing behavior affect distribution.

Blue-Green switches traffic between environments; Canary shifts exposure gradually.

---

**32. What is CSI?**

**CSI stands for Container Storage Interface.** It is a standard interface that allows storage vendors to integrate their storage systems with Kubernetes.

A CSI driver can support operations such as:

* Creating and deleting volumes.
* Attaching and detaching volumes.
* Mounting and unmounting volumes.
* Expanding volumes.
* Snapshots and cloning, when supported.

A typical installation has controller components for storage operations and node components for mounting volumes.

For example, the Amazon EBS CSI driver lets Kubernetes workloads use EBS volumes. An application usually requests storage through a PVC, while a StorageClass identifies the provisioner and storage parameters.

CSI is the integration interface and driver ecosystem; the underlying storage remains EBS, a SAN, a distributed filesystem, or another storage system. [Kubernetes CSI documentation](https://kubernetes.io/docs/concepts/storage/volumes/#csi)

---

**33. What are AWS Lambda and AWS Step Functions?**

**AWS Lambda** runs function code without requiring you to provision and manage servers for each execution. Functions can be invoked through events, API requests, queues, schedules, and other integrations.

A conventional Lambda function invocation has a configurable timeout of up to **900 seconds, or 15 minutes**. [Lambda timeout documentation](https://docs.aws.amazon.com/lambda/latest/dg/configuration-timeout.html)

**AWS Step Functions** orchestrates workflows. A state machine can coordinate Lambda functions and other AWS services using branching, retries, waits, parallel execution, and error handling.

For an order-processing workflow, Step Functions might coordinate validation, inventory reservation, payment, and notification. It can call supported AWS services directly; every step does not require a Lambda function.

| Lambda                            | Step Functions                         |
| --------------------------------- | -------------------------------------- |
| Executes application code         | Coordinates workflow steps             |
| Suitable for a unit of processing | Suitable for a multi-step process      |
| Function-level failure handling   | Workflow-level retries and error paths |

Standard workflows can run for up to one year; Express workflows run for up to five minutes. Execution and retry semantics differ by workflow type, so external side effects should be designed carefully and often made idempotent. [Step Functions workflow types](https://docs.aws.amazon.com/step-functions/latest/dg/choosing-workflow-type.html)

---

**34. What are autoscaling policies and their uses?**

For Amazon EC2 Auto Scaling:

| Policy or mechanism | Behavior                                              | Typical use                            |
| ------------------- | ----------------------------------------------------- | -------------------------------------- |
| Target tracking     | Maintains a metric near a target                      | Keep average CPU around 50%            |
| Step scaling        | Uses different adjustments based on alarm severity    | Add more instances for larger breaches |
| Simple scaling      | Performs one adjustment followed by cooldown behavior | Straightforward alarm-based scaling    |
| Scheduled scaling   | Changes capacity at specified times                   | Known daily traffic increases          |
| Predictive scaling  | Forecasts recurring demand                            | Repeated historical traffic patterns   |

Target tracking, step scaling, and simple scaling are **dynamic scaling policies**. Scheduled scaling is a separate mechanism. [Dynamic scaling policies](https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-scale-based-on-demand.html), [Scheduled scaling](https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling-scheduled-scaling.html)

For example:

```text
Minimum capacity: 2
Desired capacity: 3
Maximum capacity: 10
Target average CPU: 50%
```

Choose a metric that reflects load per unit of capacity. Also account for instance warmup, health checks, startup time, and connection draining.

Predictive scaling is useful for recurring patterns and can be combined with dynamic scaling for unexpected demand. [Predictive scaling](https://docs.aws.amazon.com/autoscaling/ec2/userguide/predictive-scaling-policy-overview.html)

---

**35. What is a “target probe” in autoscaling?**

The term is ambiguous. In an interview, establish whether it means a **target-tracking metric**, a **health check**, or a **target group**.

| Possible meaning       | Explanation                                                         |
| ---------------------- | ------------------------------------------------------------------- |
| Target-tracking metric | Metric and desired value used to adjust capacity                    |
| Health-check probe     | Request or connection used to determine whether a target is healthy |
| Target group           | Collection of backends to which a load balancer routes traffic      |

For target tracking, an example is `ASGAverageCPUUtilization` with a target value of `50`. Auto Scaling manages the associated CloudWatch alarms. [Target tracking](https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-scaling-target-tracking.html)

For an ALB health check, you might configure HTTP path `/health`, an interval, timeout, and healthy/unhealthy thresholds. Those checks evaluate target health. [ALB health checks](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/target-group-health-checks.html)

**Load-based scaling and health-based replacement solve different problems.** A failed health check is not the same signal as an overloaded, otherwise healthy instance.

---

**36. Which role should you give to access AWS services?**

Use an **IAM role with the permissions required by the workload**. There is no single role that should be assigned for every AWS service.

An IAM role has two central parts:

* **Trust policy:** who or what may assume the role.
* **Permissions policies:** what the resulting identity may do.

| Workload                  | Typical role arrangement                               |
| ------------------------- | ------------------------------------------------------ |
| EC2 application           | IAM role delivered through an instance profile         |
| Lambda function           | Lambda execution role                                  |
| ECS application container | ECS task role                                          |
| EKS application           | EKS Pod Identity or IRSA                               |
| Human operator            | Federated access, commonly through IAM Identity Center |

For an application that only reads objects under a specific S3 prefix, a permissions policy could be:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::example-app-bucket/reports/*"
    }
  ]
}
```

Save this example as `app-permissions.json` for the next answer.

This does not grant bucket listing or object writes. Additional permissions, such as access to a customer-managed KMS key, depend on the resource configuration. [AWS IAM roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html)

---

**37. How do you configure an AWS role for Kubernetes service accounts?**

On EKS, two common mechanisms are:

| EKS Pod Identity                                                            | IRSA                                                      |
| --------------------------------------------------------------------------- | --------------------------------------------------------- |
| Associates a role with a cluster, namespace, and ServiceAccount through EKS | Uses an IAM OIDC provider and a ServiceAccount annotation |
| Uses the Pod Identity Agent on supported nodes                              | Uses web-identity federation                              |
| Role trusts `pods.eks.amazonaws.com`                                        | Role trusts the cluster’s IAM OIDC provider               |
| No IAM-role annotation required on the ServiceAccount                       | Uses `eks.amazonaws.com/role-arn`                         |

For an **EKS Pod Identity** example, assume an existing EKS cluster named `my-cluster`, an `apps` namespace, and supported Linux EC2 worker nodes. Applications need a supported AWS SDK using its default credential chain. [EKS Pod Identity](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html)

Create the ServiceAccount:

```bash
kubectl create serviceaccount app-sa -n apps
```

Ensure the Pod Identity Agent is installed; if it is absent:

```bash
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name eks-pod-identity-agent
```

The node role also needs the required EKS Auth permissions.

Create `pod-trust.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "pods.eks.amazonaws.com"
      },
      "Action": [
        "sts:AssumeRole",
        "sts:TagSession"
      ],
      "Condition": {
        "StringEquals": {
          "aws:RequestTag/kubernetes-namespace": "apps",
          "aws:RequestTag/kubernetes-service-account": "app-sa"
        }
      }
    }
  ]
}
```

These conditions constrain the role to the specified namespace and ServiceAccount using session tags. [Pod Identity trust policy](https://docs.aws.amazon.com/eks/latest/userguide/pod-id-role.html)

Create the policy from question 36, create the role, and attach the policy:

```bash
aws iam create-policy \
  --policy-name AppReportRead \
  --policy-document file://app-permissions.json

aws iam create-role \
  --role-name app-s3-reader \
  --assume-role-policy-document file://pod-trust.json

aws iam attach-role-policy \
  --role-name app-s3-reader \
  --policy-arn arn:aws:iam::123456789012:policy/AppReportRead
```

Use the actual account ID and returned policy ARN.

Create the association:

```bash
aws eks create-pod-identity-association \
  --cluster-name my-cluster \
  --namespace apps \
  --service-account app-sa \
  --role-arn arn:aws:iam::123456789012:role/app-s3-reader
```

New workload Pods must specify:

```yaml
spec:
  serviceAccountName: app-sa
```

Creating the association requires suitable AWS permissions, including `iam:PassRole`. It does not itself create the Kubernetes ServiceAccount. [Creating Pod Identity associations](https://docs.aws.amazon.com/eks/latest/userguide/pod-id-association.html)

For **IRSA**, create the cluster’s IAM OIDC provider, configure a role trust policy using `sts:AssumeRoleWithWebIdentity`, constrain `sub` to `system:serviceaccount:apps:app-sa` and `aud` to `sts.amazonaws.com`, then annotate the ServiceAccount:

```bash
kubectl annotate serviceaccount app-sa -n apps \
  eks.amazonaws.com/role-arn=arn:aws:iam::123456789012:role/app-irsa-role
```

Use the chosen mechanism consistently; the annotation alone is not a complete IRSA setup. [IRSA configuration](https://docs.aws.amazon.com/eks/latest/userguide/associate-service-account-role.html)

---

**38. Explain a service account end to end.**

A Kubernetes ServiceAccount gives a workload an identity. **Authentication establishes that identity; authorization determines its permissions.**

For access to Kubernetes resources, bind the ServiceAccount to RBAC permissions. For access to AWS resources, configure IAM through the mechanism described in question 37.

The following example defines a ServiceAccount and grants it permission to read Pods in `apps`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: apps
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: apps
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-pod-reader
  namespace: apps
subjects:
  - kind: ServiceAccount
    name: app-sa
    namespace: apps
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
```

Apply and verify:

```bash
kubectl apply -f service-account-rbac.yml

kubectl auth can-i list pods \
  --as=system:serviceaccount:apps:app-sa \
  -n apps

kubectl auth can-i get secrets \
  --as=system:serviceaccount:apps:app-sa \
  -n apps
```

Assuming no other bindings grant access, the results should be `yes` for listing Pods and `no` for reading Secrets. Running impersonation checks requires appropriate operator permissions. [Kubernetes RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

A Pod uses this identity through `serviceAccountName`. By default, Kubernetes supplies a projected API token with a bounded lifetime and handles its rotation. Disable automatic API-token mounting when the workload does not need it. [Kubernetes ServiceAccounts](https://kubernetes.io/docs/concepts/security/service-accounts/)

To verify the AWS association from question 37, a temporary diagnostic Pod can run:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: aws-identity-check
  namespace: apps
spec:
  serviceAccountName: app-sa
  restartPolicy: Never

  containers:
    - name: aws-cli
      image: public.ecr.aws/aws-cli/aws-cli:latest
      args:
        - sts
        - get-caller-identity
```

```bash
kubectl apply -f aws-identity-check.yml
kubectl logs -n apps aws-identity-check
```

The returned ARN should identify the intended assumed IAM role. This verifies identity; separately test the required S3 operation to verify its permissions.

**Kubernetes RBAC grants no S3 access, and an AWS IAM role grants no Kubernetes Pod-reading permission by itself.**

---

**39. How do you create a Secret for a service account?**

For most modern uses, request a short-lived ServiceAccount token:

```bash
kubectl create token app-sa -n apps --duration=1h
```

The requested duration is subject to API-server configuration. Pods normally receive projected tokens automatically rather than needing a manually created token Secret.

If a legacy integration explicitly requires a long-lived token Secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-sa-token
  namespace: apps
  annotations:
    kubernetes.io/service-account.name: app-sa
type: kubernetes.io/service-account-token
```

Apply it:

```bash
kubectl apply -f app-sa-token.yml
```

The control plane populates the token data for the existing ServiceAccount. Creating the Secret does not grant additional permissions; RBAC still determines access. Prefer TokenRequest-based credentials where supported. [ServiceAccount token configuration](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)

Also distinguish a **ServiceAccount token Secret** from an ordinary application Secret or an `imagePullSecret`. They serve different purposes. Base64 encoding in a Secret manifest is not encryption. [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)

---

**40. What is the difference between static and dynamic storage provisioning?**

| Aspect              | Static provisioning                         | Dynamic provisioning                        |
| ------------------- | ------------------------------------------- | ------------------------------------------- |
| Storage creation    | Administrator prepares storage              | Provisioner creates storage on demand       |
| PV creation         | Administrator creates the PV object         | Provisioning components create the PV       |
| Application request | PVC binds to a matching existing PV         | PVC requests storage through a StorageClass |
| Typical use         | Existing disks or specially managed storage | Repeatable, self-service provisioning       |

A **PersistentVolume** represents storage; a **PersistentVolumeClaim** requests it. A **StorageClass** identifies a provisioner and its parameters. [Dynamic volume provisioning](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/)

Example using the standard EBS CSI driver:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-retained

provisioner: ebs.csi.aws.com

parameters:
  type: gp3
  encrypted: "true"

reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-storage
  namespace: apps

spec:
  accessModes:
    - ReadWriteOnce

  storageClassName: gp3-retained

  resources:
    requests:
      storage: 20Gi
```

`WaitForFirstConsumer` delays provisioning or binding until workload placement can be considered. This helps choose a suitable availability zone. A PVC waiting for its first consumer can therefore remain Pending without indicating a fault.

`Retain` preserves the volume for administrator handling after the claim is deleted. `Delete` requests removal of the backing storage when supported. [StorageClasses](https://kubernetes.io/docs/concepts/storage/storage-classes/)

This example requires the standard EBS CSI driver and its IAM permissions. **EKS Auto Mode uses a different provisioner: `ebs.csi.eks.amazonaws.com`.** [EBS storage on EKS](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html)

---

**41. What types of errors can occur with Pod labels?**

There is no single “Pod label error.” Different mistakes produce different symptoms:

| Problem                                            | Typical symptom                        |
| -------------------------------------------------- | -------------------------------------- |
| Invalid label key or value syntax                  | API validation error                   |
| Numeric or boolean YAML value instead of a string  | Parsing or validation error            |
| Deployment selector does not match template labels | Deployment creation or update rejected |
| Changing an existing Deployment selector           | Immutable-field error                  |
| Service selector does not match Pod labels         | No matching backends; traffic fails    |
| Required affinity uses missing or incorrect labels | Pod remains Pending                    |
| Updating an existing label without `--overwrite`   | `kubectl label` rejects the overwrite  |

For example, these do not match:

```yaml
# Service selector
selector:
  app: payments
```

```yaml
# Pod labels
labels:
  app: payment
```

Labels are case-sensitive. The label name segment and value have length and character restrictions; a value containing spaces is invalid. [Label syntax](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)

Useful diagnostics:

```bash
kubectl get pods -n apps --show-labels

kubectl get pods -n apps -l app=payments

kubectl get service payments -n apps -o yaml

kubectl get endpointslices -n apps \
  -l kubernetes.io/service-name=payments

kubectl describe pod POD_NAME -n apps
```

A Service selector mismatch can be accepted by the API but produce no useful backends. A Deployment selector/template mismatch is rejected, and an existing Deployment’s selector is immutable. For managed Pods, correct the controller’s Pod template so future replicas receive the intended labels. [Deployment selector rules](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#label-selector-updates)
