# 1. What is Ansible Vault?

## Definition

[Ansible Vault Documentation](https://docs.ansible.com/ansible/latest/vault_guide/index.html?utm_source=chatgpt.com)

Ansible Vault is a security feature in [Ansible](https://www.ansible.com/?utm_source=chatgpt.com) used to encrypt sensitive data such as:

* Passwords
* API Keys
* SSH Keys
* Database credentials
* Cloud secrets

It prevents secrets from being exposed inside playbooks or Git repositories.

---

## Why Ansible Vault is Required?

In real projects, DevOps engineers store automation code in GitHub or Bitbucket.

Without Vault:

```yaml
db_password: admin123
```

Anyone can see the password.

With Vault:

```yaml
db_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          6438666235356134...
```

Now the secret is encrypted.

---

## Common Vault Commands

### Create Encrypted File

```bash
ansible-vault create secrets.yml
```

### Edit Encrypted File

```bash
ansible-vault edit secrets.yml
```

### Encrypt Existing File

```bash
ansible-vault encrypt secrets.yml
```

### Decrypt File

```bash
ansible-vault decrypt secrets.yml
```

### Run Playbook with Vault Password

```bash
ansible-playbook site.yml --ask-vault-pass
```

---

## Real-Time Example

### Scenario

You are deploying a Java application on Azure VMs.

The application requires:

* DB username
* DB password
* API token

You store them securely in:

```yaml
# secrets.yml
db_user: admin
db_password: MySecurePass
api_token: xxxxxxxxx
```

Encrypt it:

```bash
ansible-vault encrypt secrets.yml
```

During deployment:

```bash
ansible-playbook deploy.yml --ask-vault-pass
```

This is widely used in:

* Banking projects
* Healthcare applications
* Enterprise CI/CD pipelines

---

# 2. What are the Components of Ansible?

## Main Components

| Component     | Purpose                                |
| ------------- | -------------------------------------- |
| Control Node  | Machine where Ansible is installed     |
| Managed Nodes | Target servers                         |
| Inventory     | List of managed servers                |
| Modules       | Small programs performing tasks        |
| Playbooks     | YAML automation scripts                |
| Roles         | Reusable automation structure          |
| Plugins       | Extend Ansible functionality           |
| Facts         | System information gathered from nodes |

---

## Architecture Flow

```text
Control Node
     |
 SSH/WinRM
     |
Managed Nodes
```

---

## Real-Time Example

Suppose your company has:

* 20 Linux servers
* 10 Web servers
* 5 Database servers

Ansible Control Node:

* Executes automation

Inventory:

* Stores server IPs

Playbook:

* Installs Apache

Modules:

* yum
* service
* copy

Roles:

* reusable webserver setup

---

# 3. What is Inventory in Ansible?

## Definition

Inventory is a file that contains:

* Server IP addresses
* Hostnames
* Groups of servers

It tells Ansible:

> “Which servers should I manage?”

---

## Inventory File Example

### INI Format

```ini
[web]
192.168.1.10
192.168.1.11

[db]
192.168.1.20
```

---

## YAML Inventory

```yaml
all:
  children:
    web:
      hosts:
        web1:
          ansible_host: 192.168.1.10
```

---

## Real-Time Example

In a production environment:

| Server Type | Count |
| ----------- | ----- |
| Web Servers | 20    |
| App Servers | 10    |
| DB Servers  | 5     |

Inventory helps organize them efficiently.

---

# 4. What are the Types of Inventories?

## 1. Static Inventory

Manually maintained inventory file.

Example:

```ini
[web]
10.0.0.1
10.0.0.2
```

### Used In:

* Small environments
* Lab setup
* Fixed infrastructure

---

## 2. Dynamic Inventory

Automatically fetches server details from cloud providers.

Supports:

* Azure
* AWS
* GCP
* VMware

---

## Real-Time Example

### Azure Dynamic Inventory

When new VM is created:

* Inventory updates automatically

Useful in:

* Auto-scaling
* Kubernetes worker nodes
* CI/CD pipelines

---

# 5. What is Play & Playbook?

## What is a Play?

A Play maps:

```text
Hosts → Tasks
```

Example:

```yaml
- hosts: web
  tasks:
```

---

## What is a Playbook?

A Playbook is a YAML file containing:

* One or more plays
* Tasks
* Variables
* Handlers

---

## Playbook Example

```yaml
---
- name: Install Apache
  hosts: web
  become: yes

  tasks:
    - name: Install package
      yum:
        name: httpd
        state: present

    - name: Start service
      service:
        name: httpd
        state: started
```

---

## Real-Time Example

CI/CD Deployment Pipeline:

* Jenkins triggers Ansible playbook
* Playbook deploys application
* Restarts services
* Updates configurations

---

# 6. Difference Between Hosts & Groups?

| Hosts              | Groups                 |
| ------------------ | ---------------------- |
| Individual servers | Collection of servers  |
| Unique machine     | Logical categorization |
| Example: web1      | Example: webservers    |

---

## Example

```ini
[web]
web1
web2

[db]
db1
```

Here:

* web1 → host
* web → group

---

## Real-Time Example

Production Environment:

```ini
[frontend]
10 servers

[backend]
15 servers

[database]
5 servers
```

You can target:

```bash
ansible frontend -m ping
```

Instead of individual servers.

---

# 7. What is Roles?

## Definition

Roles are a structured way to organize playbooks.

They improve:

* Reusability
* Maintainability
* Scalability

---

## Role Structure

```text
roles/
└── webserver/
    ├── tasks/
    ├── handlers/
    ├── templates/
    ├── files/
    ├── vars/
    ├── defaults/
    └── meta/
```

---

## Benefits

* Reusable automation
* Clean project structure
* Easy collaboration
* Better CI/CD integration

---

## Real-Time Example

Suppose you configure:

* Apache
* Nginx
* Docker
* Java

Instead of writing repeated tasks,
you create reusable roles:

* apache-role
* docker-role
* java-role

Used across:

* Dev
* QA
* UAT
* Production

---

# 8. How to Install a Role?

## Using Ansible Galaxy

[Ansible Galaxy](https://galaxy.ansible.com/?utm_source=chatgpt.com)

### Install Command

```bash
ansible-galaxy role install geerlingguy.nginx
```

Installed in:

```text
~/.ansible/roles/
```

---

## Use Role in Playbook

```yaml
---
- hosts: web
  roles:
    - geerlingguy.nginx
```

---

## Real-Time Example

Instead of manually configuring Nginx:

* Download trusted community role
* Customize variables
* Deploy quickly

Common in enterprise DevOps teams.

---

# 9. How to Install Multiple Roles?

## Using requirements.yml

```yaml
---
roles:
  - name: geerlingguy.nginx
  - name: geerlingguy.mysql
  - name: geerlingguy.docker
```

---

## Install Command

```bash
ansible-galaxy install -r requirements.yml
```

---

## Real-Time Example

Project Setup:

* Nginx
* MySQL
* Docker
* Java

Single command installs all dependencies.

Useful in:

* New environment provisioning
* CI/CD bootstrap
* Infrastructure standardization

---

# 10. How to Create Roles?

## Command

```bash
ansible-galaxy init webserver
```

---

## Generated Structure

```text
webserver/
├── defaults/
├── files/
├── handlers/
├── meta/
├── tasks/
├── templates/
├── tests/
├── vars/
```

---

## Add Tasks

### tasks/main.yml

```yaml
---
- name: Install Apache
  yum:
    name: httpd
    state: present

- name: Start Apache
  service:
    name: httpd
    state: started
```

---

## Use Role

```yaml
---
- hosts: web
  roles:
    - webserver
```

---

## Real-Time Example

Enterprise Project:

* Separate teams manage separate roles.
* Security team manages hardening role.
* Middleware team manages Tomcat role.
* DB team manages MySQL role.

This improves:

* Scalability
* Reusability
* Standardization
* Faster deployments

---

# Senior-Level Interview Tips

## Common Follow-Up Questions

### Q1: Why Roles are Better Than Playbooks?

Answer:

* Modular
* Reusable
* Easier maintenance
* Better team collaboration

---

### Q2: Where Do You Store Vault Passwords?

Answer:

* Azure Key Vault
* Jenkins Credentials
* HashiCorp Vault
* GitHub Secrets

Never hardcode vault passwords.

---

### Q3: How Do You Secure Ansible?

Answer:

* Use SSH keys
* Use Vault
* RBAC with AWX/Tower
* Least privilege access
* Store secrets securely

---

### Q4: Difference Between Role Variables and Defaults?

| defaults          | vars             |
| ----------------- | ---------------- |
| Lower priority    | Higher priority  |
| Easily overridden | Hard to override |

---

# Real-Time Enterprise Ansible Workflow

```text
Developer Pushes Code
        ↓
Jenkins Pipeline Triggered
        ↓
Ansible Playbook Executes
        ↓
Roles Configure Servers
        ↓
Vault Injects Secrets
        ↓
Application Deployed
        ↓
Monitoring Validated
```
# 11. What is Dynamic Inventory & when do we use it & for what?

## Definition

Dynamic Inventory automatically fetches server details from cloud platforms or external systems instead of manually maintaining inventory files.

Unlike static inventory:

```ini id="f0mswd"
[web]
10.0.0.1
10.0.0.2
```

Dynamic inventory updates automatically whenever infrastructure changes.

---

## Why Dynamic Inventory is Needed?

Modern infrastructure is dynamic:

* VMs are created automatically
* Containers scale up/down
* Auto Scaling Groups change instances
* Kubernetes nodes change frequently

Manually updating inventory becomes impossible.

---

## Supported Platforms

Dynamic Inventory supports:

* Azure
* AWS
* GCP
* VMware
* Kubernetes
* OpenStack

---

## Azure Dynamic Inventory Example

Install Azure collection:

```bash id="6a0yr8"
ansible-galaxy collection install azure.azcollection
```

Inventory File:

```yaml id="gjjlwm"
plugin: azure.azcollection.azure_rm
include_vm_resource_groups:
  - Production-RG
auth_source: auto
```

Run:

```bash id="rkrm0y"
ansible-inventory -i azure_rm.yml --graph
```

---

## Real-Time Example

### Scenario

Your company uses:

* Azure VM Scale Sets
* AKS worker nodes
* Auto-scaled web servers

When new servers are created:

* Dynamic inventory automatically detects them
* No manual update required

Used heavily in:

* Cloud DevOps
* Kubernetes
* CI/CD pipelines
* Auto-scaling environments

---

## Interview Tip

### Difference Between Static & Dynamic Inventory

| Static             | Dynamic                         |
| ------------------ | ------------------------------- |
| Manual             | Automatic                       |
| Fixed servers      | Cloud/ephemeral servers         |
| Small environments | Enterprise cloud infrastructure |

---

# 12. What are Ansible Roles?

## Definition

Roles are reusable, modular components in Ansible used to organize automation into structured directories.

Roles help:

* Reuse code
* Standardize deployments
* Simplify maintenance

---

## Role Structure

```text id="nlx3v6"
roles/
└── apache/
    ├── tasks/
    ├── handlers/
    ├── templates/
    ├── files/
    ├── vars/
    ├── defaults/
    └── meta/
```

---

## Purpose of Each Folder

| Folder    | Purpose               |
| --------- | --------------------- |
| tasks     | Main automation tasks |
| handlers  | Restart services      |
| templates | Jinja2 templates      |
| files     | Static files          |
| vars      | Variables             |
| defaults  | Default variables     |
| meta      | Role dependencies     |

---

## Real-Time Example

Suppose your organization deploys:

* Apache
* Docker
* Java
* Tomcat

Instead of rewriting tasks:

* Create reusable roles

Examples:

* apache-role
* docker-role
* tomcat-role

Used across:

* Dev
* QA
* UAT
* Production

---

## Role Example

```yaml id="hy5azv"
---
- hosts: web
  roles:
    - apache
```

---

## Enterprise Benefits

* Easy collaboration
* Faster deployments
* Standard configurations
* Reusable automation
* Better Git management

---

# 13. What’s your Ansible experience in real projects?

## Sample Senior DevOps Answer

### Real-Time Experience Answer

“I have worked extensively with Ansible in enterprise DevOps and cloud environments for infrastructure automation, configuration management, and CI/CD deployments.”

---

## Areas Worked On

### 1. Server Provisioning

* Configured Linux servers
* Installed middleware
* User management
* Security hardening

### 2. Application Deployment

* Java applications
* Docker containers
* Nginx/Apache setup
* Rolling deployments

### 3. CI/CD Integration

Integrated Ansible with:

* Jenkins
* Azure DevOps
* GitHub Actions

---

## Real Project Example

### Project:

Banking Application Deployment on Azure

### Responsibilities:

* Automated VM configuration
* Installed Java/Tomcat
* Configured Nginx reverse proxy
* Managed secrets using Vault
* Integrated with Jenkins pipeline

### Tools Used:

* Ansible
* Azure
* Jenkins
* Docker
* Kubernetes

---

## Business Impact

* Reduced deployment time from 4 hours to 20 minutes
* Eliminated manual configuration errors
* Improved deployment consistency

---

# 14. How do you manage sensitive data like passwords in Ansible?

## Best Practice

Sensitive data should never be stored in plain text.

Use:

* Ansible Vault
* Azure Key Vault
* Jenkins Credentials
* HashiCorp Vault

---

## Using Ansible Vault

### Create Secret File

```bash id="o7wb2m"
ansible-vault create secrets.yml
```

Example:

```yaml id="6w8tds"
db_password: MySecurePassword
api_key: XXXXXX
```

---

## Use in Playbook

```yaml id="1n5dlm"
vars_files:
  - secrets.yml
```

Run:

```bash id="b3t9kt"
ansible-playbook deploy.yml --ask-vault-pass
```

---

## Real-Time Example

In production:

* Database passwords
* SSL certificates
* API tokens
* Cloud credentials

All encrypted using Vault.

---

# 15. How do you securely run playbooks that use secrets?

## Methods Used

### 1. Ansible Vault

Encrypt secrets.

### 2. Jenkins Credentials

Store secrets securely in CI/CD pipeline.

### 3. Azure Key Vault

Fetch secrets dynamically.

### 4. Environment Variables

Avoid hardcoding credentials.

---

## Secure Execution Example

```bash id="3q67r4"
ansible-playbook deploy.yml --vault-password-file vault_pass.txt
```

---

## Jenkins Pipeline Example

```groovy id="w5k0h8"
withCredentials([string(credentialsId: 'vault-pass', variable: 'VAULT_PASS')]) {
    sh 'ansible-playbook deploy.yml --vault-password-file vault_pass.txt'
}
```

---

## Real-Time Enterprise Practice

Most enterprises:

* Store secrets centrally
* Restrict RBAC access
* Rotate secrets periodically
* Audit secret access

---

# 16. Write an Ansible playbook for automation

## Example: Install Apache Web Server

```yaml id="k2izkp"
---
- name: Configure Web Server
  hosts: web
  become: yes

  tasks:

    - name: Install Apache
      yum:
        name: httpd
        state: present

    - name: Start Apache Service
      service:
        name: httpd
        state: started
        enabled: yes

    - name: Copy index file
      copy:
        src: index.html
        dest: /var/www/html/index.html
```

---

## Run Playbook

```bash id="qk65lt"
ansible-playbook -i inventory.ini apache.yml
```

---

## Real-Time Example

This type of playbook is used to:

* Configure 100+ servers
* Standardize deployments
* Deploy applications automatically

---

# 17. What is an Ansible inventory file, and how is it used?

## Definition

Inventory file contains:

* Hostnames
* IP addresses
* Groups of servers

It tells Ansible:

> Which servers to manage.

---

## Example Inventory

```ini id="d0d9sm"
[web]
10.0.0.10
10.0.0.11

[db]
10.0.0.20
```

---

## Usage

Run command:

```bash id="x75ntw"
ansible web -m ping
```

Ansible connects to all servers in `web` group.

---

## Real-Time Example

Enterprise Infrastructure:

```text id="nhqzn6"
Web Servers → web
Application Servers → app
Database Servers → db
```

Inventory organizes infrastructure logically.

---

# 18. Where is the Ansible Configuration file located?

## Default Location

```bash id="cbdv0s"
/etc/ansible/ansible.cfg
```

---

## Other Possible Locations

Ansible checks config files in this order:

| Priority | Location                         |
| -------- | -------------------------------- |
| 1        | ANSIBLE_CONFIG variable          |
| 2        | ansible.cfg in current directory |
| 3        | ~/.ansible.cfg                   |
| 4        | /etc/ansible/ansible.cfg         |

---

## Check Current Config

```bash id="q0gl2k"
ansible --version
```

---

## Real-Time Example

In enterprise projects:

* Different environments use different configs
* Separate configs for:

  * Dev
  * QA
  * Production

---

# 19. What are the different ways other than SSH by which Ansible can connect to remote hosts?

## Linux Connection Methods

| Method   | Usage                      |
| -------- | -------------------------- |
| SSH      | Default Linux connection   |
| Paramiko | Python SSH library         |
| Local    | Run locally                |
| Docker   | Container connection       |
| Podman   | Containerized environments |

---

## Windows Connection Methods

| Method | Usage                     |
| ------ | ------------------------- |
| WinRM  | Windows remote management |
| PSRP   | PowerShell Remoting       |

---

## Network Devices

| Method      | Usage                 |
| ----------- | --------------------- |
| Network CLI | Cisco/Juniper devices |
| HTTPAPI     | REST APIs             |

---

## Example

### Windows Inventory

```ini id="ow0j7r"
[windows]
10.0.0.50

[windows:vars]
ansible_connection=winrm
ansible_user=Administrator
ansible_password=Password123
```

---

## Real-Time Example

In enterprise environments:

* Linux → SSH
* Windows → WinRM
* Cisco Switches → Network CLI
* Docker containers → Docker connection

---

# 20. What is variable in Ansible? What are different types of variables?

## Definition

Variables store dynamic values used in playbooks.

They help:

* Reuse code
* Avoid hardcoding
* Build flexible automation

---

## Example

```yaml id="s2w48r"
vars:
  package_name: httpd
```

Use:

```yaml id="zyw9lr"
- name: Install package
  yum:
    name: "{{ package_name }}"
    state: present
```

---

# Types of Variables in Ansible

| Variable Type        | Description                |
| -------------------- | -------------------------- |
| Playbook Variables   | Defined inside playbook    |
| Inventory Variables  | Defined in inventory       |
| Host Variables       | Specific to hosts          |
| Group Variables      | Specific to groups         |
| Extra Variables      | Passed at runtime          |
| Registered Variables | Store task output          |
| Facts                | System-generated variables |
| Role Variables       | Used in roles              |

---

## 1. Playbook Variables

```yaml id="y0eqzt"
vars:
  app_port: 8080
```

---

## 2. Inventory Variables

```ini id="mfx7w2"
[web]
web1 ansible_host=10.0.0.1 ansible_user=azureuser
```

---

## 3. Extra Variables

```bash id="byes6h"
ansible-playbook deploy.yml -e "env=prod"
```

---

## 4. Registered Variables

```yaml id="4wns89"
- name: Check disk usage
  shell: df -h
  register: disk_output
```

---

## 5. Facts

```yaml id="t8kwru"
{{ ansible_hostname }}
{{ ansible_os_family }}
```

Gathered automatically by Ansible.

---

## Real-Time Example

### Multi-Environment Deployment

Variables used for:

* Dev environment
* QA environment
* Production environment

Example:

```yaml id="dspg7d"
db_name: prod_db
db_server: prod-sql-server
```

Same playbook works for all environments using different variable values.

---

# Senior-Level Interview Tips

## Q1: Why are Roles Important?

Answer:

* Reusability
* Scalability
* Clean structure

---

## Q2: How do you avoid hardcoding variables?

Answer:

* group_vars
* host_vars
* Vault
* CI/CD variables

---

## Q3: Difference Between vars and defaults?

| vars               | defaults           |
| ------------------ | ------------------ |
| High priority      | Low priority       |
| Harder to override | Easier to override |

---

# Enterprise Ansible Workflow

```text id="7l6m5n"
GitHub Commit
      ↓
Jenkins Pipeline
      ↓
Ansible Playbook
      ↓
Dynamic Inventory Fetches Servers
      ↓
Vault Loads Secrets
      ↓
Roles Configure Infrastructure
      ↓
Deployment Completed
```
# 21. How to assign variables in group_vars & host_vars?

## Definition

In Ansible:

* `group_vars` → Variables assigned to a group of servers
* `host_vars` → Variables assigned to a specific server

These help manage environment-specific configurations cleanly.

---

# Directory Structure

```text id="px24o4"
inventory/
├── hosts
├── group_vars/
│   ├── web.yml
│   ├── db.yml
│
├── host_vars/
│   ├── web1.yml
│   ├── db1.yml
```

---

# group_vars Example

## Inventory

```ini id="mk4hrq"
[web]
web1
web2
```

---

## group_vars/web.yml

```yaml id="jlwm2x"
http_port: 80
package_name: httpd
```

All servers in `web` group receive these variables.

---

# host_vars Example

## host_vars/web1.yml

```yaml id="2wh46d"
http_port: 8080
```

This variable applies only to `web1`.

---

# Priority Example

If:

```yaml id="kttjlwm"
group_vars/web.yml → http_port: 80
host_vars/web1.yml → http_port: 8080
```

Then:

* web1 → 8080
* web2 → 80

Host vars override group vars.

---

# Real-Time Example

## Production Environment

### group_vars/prod.yml

```yaml id="4xv1y3"
environment: production
db_server: prod-sql.company.com
```

### host_vars/web-prod-01.yml

```yaml id="13kqql"
max_connections: 5000
```

Used heavily in:

* Multi-environment deployments
* Enterprise automation
* Kubernetes clusters
* Cloud infrastructure

---

# 22. Difference between File & Template directory in Roles?

| files                   | templates            |
| ----------------------- | -------------------- |
| Static files            | Dynamic files        |
| No variable replacement | Supports variables   |
| Uses copy module        | Uses template module |
| Plain file transfer     | Jinja2 rendering     |

---

# Files Directory

## Purpose

Stores:

* Static configs
* Scripts
* Binary files
* Certificates

---

## Example

```text id="s7f7i4"
roles/web/files/index.html
```

Task:

```yaml id="jlwmgb"
- copy:
    src: index.html
    dest: /var/www/html/index.html
```

Exact file copied.

---

# Templates Directory

## Purpose

Stores dynamic configuration files using Jinja2 variables.

---

## Example

### Template File

```text id="p5thpc"
roles/web/templates/httpd.conf.j2
```

Content:

```jinja2 id="i3t9zw"
Listen {{ http_port }}
ServerAdmin {{ admin_email }}
```

Task:

```yaml id="4ktj44"
- template:
    src: httpd.conf.j2
    dest: /etc/httpd/conf/httpd.conf
```

Variables replaced dynamically.

---

# Real-Time Example

## Files Directory Used For

* SSL certificates
* Static scripts
* WAR files

## Templates Directory Used For

* Nginx configs
* Apache configs
* Kubernetes YAMLs
* Application properties

---

# 23. Difference between defaults & vars directory in Roles?

| defaults               | vars                    |
| ---------------------- | ----------------------- |
| Low priority variables | High priority variables |
| Easily overridden      | Hard to override        |
| User customizable      | Internal role variables |

---

# defaults Directory

```text id="8xq0qc"
roles/web/defaults/main.yml
```

Example:

```yaml id="rn75px"
http_port: 80
```

Can be overridden:

* Inventory
* Playbook vars
* Extra vars

---

# vars Directory

```text id="jlwmzq"
roles/web/vars/main.yml
```

Example:

```yaml id="2zyr2o"
service_name: httpd
```

Higher priority.

Harder to override.

---

# Variable Precedence

```text id="jlwmn1"
Extra Vars
   ↓
Task Vars
   ↓
Role Vars
   ↓
Inventory Vars
   ↓
Role Defaults
```

---

# Real-Time Example

## defaults

Used for customizable settings:

```yaml id="jlwmrj"
app_port: 8080
```

## vars

Used for fixed internal values:

```yaml id="jlwm3m"
service_name: nginx
```

---

# 24. What is Jinja2 Template?

## Definition

[Jinja Documentation](https://jinja.palletsprojects.com/?utm_source=chatgpt.com)

Jinja2 is a templating engine used in Ansible for generating dynamic configuration files.

It allows:

* Variables
* Loops
* Conditions
* Expressions

---

# Example

## Template File

```jinja2 id="3aqv73"
ServerName {{ inventory_hostname }}
Listen {{ http_port }}
```

---

# Playbook

```yaml id="jlwm9q"
- template:
    src: httpd.conf.j2
    dest: /etc/httpd/conf/httpd.conf
```

---

# Features

| Feature    | Example   |
| ---------- | --------- |
| Variables  | {{ var }} |
| Loops      | {% for %} |
| Conditions | {% if %}  |

---

# Loop Example

```jinja2 id="jlwmvb"
{% for user in users %}
user {{ user }}
{% endfor %}
```

---

# Real-Time Example

Jinja2 is heavily used for:

* Nginx configs
* Kubernetes manifests
* Application configs
* Docker compose files

---

# 25. What is Modules in Ansible?

## Definition

Modules are reusable units of work in Ansible.

They perform actual tasks on remote systems.

Think of modules as:

> “Building blocks of Ansible automation”

---

# Common Modules

| Module  | Purpose                   |
| ------- | ------------------------- |
| yum     | Install packages          |
| apt     | Ubuntu package management |
| service | Manage services           |
| copy    | Copy files                |
| file    | Manage files/directories  |
| user    | Create users              |
| shell   | Execute shell commands    |

---

# Example

```yaml id="jlwmh4"
- name: Install Apache
  yum:
    name: httpd
    state: present
```

Here:

* `yum` is the module.

---

# Real-Time Example

In enterprise projects modules are used for:

* Server provisioning
* Docker deployment
* Kubernetes automation
* Middleware installation
* User management

---

# 26. Difference between COPY & FILE modules?

| COPY Module                | FILE Module                   |
| -------------------------- | ----------------------------- |
| Copies files               | Manages file properties       |
| Transfers content          | Changes permissions/ownership |
| Creates files with content | Creates directories/symlinks  |

---

# COPY Module Example

```yaml id="jlwm8g"
- copy:
    src: app.conf
    dest: /etc/app.conf
```

Copies file from control node to target node.

---

# FILE Module Example

```yaml id="jlwm9w"
- file:
    path: /opt/app
    state: directory
    mode: '0755'
```

Creates/manages directory.

---

# Real-Time Example

## COPY Module

Used for:

* Application configs
* Deployment files
* SSL certificates

## FILE Module

Used for:

* Creating directories
* Setting permissions
* Symlinks

---

# 27. Difference between SHELL & COMMAND modules?

| SHELL                      | COMMAND                 |
| -------------------------- | ----------------------- |
| Executes through shell     | Executes directly       |
| Supports pipes/redirection | No shell features       |
| Less secure                | More secure             |
| Use for complex commands   | Use for simple commands |

---

# COMMAND Example

```yaml id="n75xpw"
- command: uptime
```

Simple command execution.

---

# SHELL Example

```yaml id="jlwm2r"
- shell: "ps -ef | grep java"
```

Uses pipe operator.

---

# Key Difference

This works only in shell:

```bash id="jlwm8l"
cat file.txt | grep error
```

Because:

* Pipe (`|`) requires shell processing.

---

# Security Recommendation

Prefer:

* `command`

Use:

* `shell`
  only when required.

---

# Real-Time Example

## COMMAND

* Service checks
* Simple commands
* Secure execution

## SHELL

* Complex scripts
* Pipes
* Redirections
* Multi-command execution

---

# 28. What is Setup module? What does it do?

## Definition

`setup` module gathers facts about remote systems.

Facts include:

* OS details
* IP address
* Hostname
* Memory
* CPU
* Disk info

---

# Example

```bash id="2njlwm"
ansible all -m setup
```

---

# Sample Output

```json id="jlwmh5"
"ansible_hostname": "web01",
"ansible_os_family": "RedHat",
"ansible_memtotal_mb": 4096
```

---

# Use in Playbook

```yaml id="jlwmwv"
- debug:
    var: ansible_hostname
```

---

# Real-Time Example

Used for:

* Dynamic configuration
* OS-based package installation
* Conditional tasks

Example:

```yaml id="jlwmj1"
when: ansible_os_family == "RedHat"
```

---

# 29. What is register & debug in Ansible?

# Register

## Definition

`register` stores output of a task into a variable.

---

## Example

```yaml id="jlwm2u"
- name: Check disk usage
  shell: df -h
  register: disk_output
```

Now:

```yaml id="jlwm0w"
disk_output.stdout
```

contains command output.

---

# Debug

## Definition

`debug` prints variables/output during playbook execution.

---

## Example

```yaml id="jlwmcr"
- debug:
    var: disk_output.stdout
```

---

# Real-Time Example

Used for:

* Troubleshooting
* Logging
* Validation
* Dynamic decision making

---

# Full Example

```yaml id="jlwmxv"
- name: Check Java Version
  shell: java -version
  register: java_output

- debug:
    var: java_output.stderr
```

---

# What is changed_when in Ansible?

## Definition

`changed_when` controls whether Ansible reports a task as:

* Changed
  or
* Unchanged

Useful when commands do not actually modify anything.

---

# Example

```yaml id="jlwmx8"
- name: Check service status
  shell: systemctl status httpd
  register: result
  changed_when: false
```

Even if command runs successfully:

* Task will show `ok`
  instead of `changed`.

---

# Why Important?

Without `changed_when`:

* Every shell/command task may show as changed.

This causes:

* Incorrect reporting
* Unnecessary handler execution

---

# Real-Time Example

## Health Check Tasks

```yaml id="jlwm8t"
- name: Check disk usage
  shell: df -h
  register: disk
  changed_when: false
```

Health checks should not mark servers as changed.

---

# Advanced Example

```yaml id="jlwmvu"
- shell: cat /tmp/app_status
  register: output
  changed_when: "'updated' in output.stdout"
```

Only marks task changed if:

```text id="jlwm0z"
updated
```

exists in output.

---

# Senior-Level Interview Tips

## Q1: Why use Templates instead of Copy?

Answer:
Templates support dynamic configuration.

---

## Q2: Why avoid shell module?

Answer:

* Less secure
* Not idempotent
* Vulnerable to shell interpretation

---

## Q3: What are Facts?

Answer:
System information automatically gathered by setup module.

---

# Enterprise Workflow Example

```text id="jlwm1d"
Dynamic Inventory
        ↓
Group Variables Loaded
        ↓
Role Executes
        ↓
Templates Render Configurations
        ↓
Modules Configure Servers
        ↓
Register Captures Outputs
        ↓
Debug Validates Deployment
```
# 21. How to assign variables in group_vars & host_vars?

## Definition

In Ansible:

* `group_vars` → Variables assigned to a group of servers
* `host_vars` → Variables assigned to a specific server

These help manage environment-specific configurations cleanly.

---

# Directory Structure

```text id="px24o4"
inventory/
├── hosts
├── group_vars/
│   ├── web.yml
│   ├── db.yml
│
├── host_vars/
│   ├── web1.yml
│   ├── db1.yml
```

---

# group_vars Example

## Inventory

```ini id="mk4hrq"
[web]
web1
web2
```

---

## group_vars/web.yml

```yaml id="jlwm2x"
http_port: 80
package_name: httpd
```

All servers in `web` group receive these variables.

---

# host_vars Example

## host_vars/web1.yml

```yaml id="2wh46d"
http_port: 8080
```

This variable applies only to `web1`.

---

# Priority Example

If:

```yaml id="kttjlwm"
group_vars/web.yml → http_port: 80
host_vars/web1.yml → http_port: 8080
```

Then:

* web1 → 8080
* web2 → 80

Host vars override group vars.

---

# Real-Time Example

## Production Environment

### group_vars/prod.yml

```yaml id="4xv1y3"
environment: production
db_server: prod-sql.company.com
```

### host_vars/web-prod-01.yml

```yaml id="13kqql"
max_connections: 5000
```

Used heavily in:

* Multi-environment deployments
* Enterprise automation
* Kubernetes clusters
* Cloud infrastructure

---

# 22. Difference between File & Template directory in Roles?

| files                   | templates            |
| ----------------------- | -------------------- |
| Static files            | Dynamic files        |
| No variable replacement | Supports variables   |
| Uses copy module        | Uses template module |
| Plain file transfer     | Jinja2 rendering     |

---

# Files Directory

## Purpose

Stores:

* Static configs
* Scripts
* Binary files
* Certificates

---

## Example

```text id="s7f7i4"
roles/web/files/index.html
```

Task:

```yaml id="jlwmgb"
- copy:
    src: index.html
    dest: /var/www/html/index.html
```

Exact file copied.

---

# Templates Directory

## Purpose

Stores dynamic configuration files using Jinja2 variables.

---

## Example

### Template File

```text id="p5thpc"
roles/web/templates/httpd.conf.j2
```

Content:

```jinja2 id="i3t9zw"
Listen {{ http_port }}
ServerAdmin {{ admin_email }}
```

Task:

```yaml id="4ktj44"
- template:
    src: httpd.conf.j2
    dest: /etc/httpd/conf/httpd.conf
```

Variables replaced dynamically.

---

# Real-Time Example

## Files Directory Used For

* SSL certificates
* Static scripts
* WAR files

## Templates Directory Used For

* Nginx configs
* Apache configs
* Kubernetes YAMLs
* Application properties

---

# 23. Difference between defaults & vars directory in Roles?

| defaults               | vars                    |
| ---------------------- | ----------------------- |
| Low priority variables | High priority variables |
| Easily overridden      | Hard to override        |
| User customizable      | Internal role variables |

---

# defaults Directory

```text id="8xq0qc"
roles/web/defaults/main.yml
```

Example:

```yaml id="rn75px"
http_port: 80
```

Can be overridden:

* Inventory
* Playbook vars
* Extra vars

---

# vars Directory

```text id="jlwmzq"
roles/web/vars/main.yml
```

Example:

```yaml id="2zyr2o"
service_name: httpd
```

Higher priority.

Harder to override.

---

# Variable Precedence

```text id="jlwmn1"
Extra Vars
   ↓
Task Vars
   ↓
Role Vars
   ↓
Inventory Vars
   ↓
Role Defaults
```

---

# Real-Time Example

## defaults

Used for customizable settings:

```yaml id="jlwmrj"
app_port: 8080
```

## vars

Used for fixed internal values:

```yaml id="jlwm3m"
service_name: nginx
```

---

# 24. What is Jinja2 Template?

## Definition

[Jinja Documentation](https://jinja.palletsprojects.com/?utm_source=chatgpt.com)

Jinja2 is a templating engine used in Ansible for generating dynamic configuration files.

It allows:

* Variables
* Loops
* Conditions
* Expressions

---

# Example

## Template File

```jinja2 id="3aqv73"
ServerName {{ inventory_hostname }}
Listen {{ http_port }}
```

---

# Playbook

```yaml id="jlwm9q"
- template:
    src: httpd.conf.j2
    dest: /etc/httpd/conf/httpd.conf
```

---

# Features

| Feature    | Example   |
| ---------- | --------- |
| Variables  | {{ var }} |
| Loops      | {% for %} |
| Conditions | {% if %}  |

---

# Loop Example

```jinja2 id="jlwmvb"
{% for user in users %}
user {{ user }}
{% endfor %}
```

---

# Real-Time Example

Jinja2 is heavily used for:

* Nginx configs
* Kubernetes manifests
* Application configs
* Docker compose files

---

# 25. What is Modules in Ansible?

## Definition

Modules are reusable units of work in Ansible.

They perform actual tasks on remote systems.

Think of modules as:

> “Building blocks of Ansible automation”

---

# Common Modules

| Module  | Purpose                   |
| ------- | ------------------------- |
| yum     | Install packages          |
| apt     | Ubuntu package management |
| service | Manage services           |
| copy    | Copy files                |
| file    | Manage files/directories  |
| user    | Create users              |
| shell   | Execute shell commands    |

---

# Example

```yaml id="jlwmh4"
- name: Install Apache
  yum:
    name: httpd
    state: present
```

Here:

* `yum` is the module.

---

# Real-Time Example

In enterprise projects modules are used for:

* Server provisioning
* Docker deployment
* Kubernetes automation
* Middleware installation
* User management

---

# 26. Difference between COPY & FILE modules?

| COPY Module                | FILE Module                   |
| -------------------------- | ----------------------------- |
| Copies files               | Manages file properties       |
| Transfers content          | Changes permissions/ownership |
| Creates files with content | Creates directories/symlinks  |

---

# COPY Module Example

```yaml id="jlwm8g"
- copy:
    src: app.conf
    dest: /etc/app.conf
```

Copies file from control node to target node.

---

# FILE Module Example

```yaml id="jlwm9w"
- file:
    path: /opt/app
    state: directory
    mode: '0755'
```

Creates/manages directory.

---

# Real-Time Example

## COPY Module

Used for:

* Application configs
* Deployment files
* SSL certificates

## FILE Module

Used for:

* Creating directories
* Setting permissions
* Symlinks

---

# 27. Difference between SHELL & COMMAND modules?

| SHELL                      | COMMAND                 |
| -------------------------- | ----------------------- |
| Executes through shell     | Executes directly       |
| Supports pipes/redirection | No shell features       |
| Less secure                | More secure             |
| Use for complex commands   | Use for simple commands |

---

# COMMAND Example

```yaml id="n75xpw"
- command: uptime
```

Simple command execution.

---

# SHELL Example

```yaml id="jlwm2r"
- shell: "ps -ef | grep java"
```

Uses pipe operator.

---

# Key Difference

This works only in shell:

```bash id="jlwm8l"
cat file.txt | grep error
```

Because:

* Pipe (`|`) requires shell processing.

---

# Security Recommendation

Prefer:

* `command`

Use:

* `shell`
  only when required.

---

# Real-Time Example

## COMMAND

* Service checks
* Simple commands
* Secure execution

## SHELL

* Complex scripts
* Pipes
* Redirections
* Multi-command execution

---

# 28. What is Setup module? What does it do?

## Definition

`setup` module gathers facts about remote systems.

Facts include:

* OS details
* IP address
* Hostname
* Memory
* CPU
* Disk info

---

# Example

```bash id="2njlwm"
ansible all -m setup
```

---

# Sample Output

```json id="jlwmh5"
"ansible_hostname": "web01",
"ansible_os_family": "RedHat",
"ansible_memtotal_mb": 4096
```

---

# Use in Playbook

```yaml id="jlwmwv"
- debug:
    var: ansible_hostname
```

---

# Real-Time Example

Used for:

* Dynamic configuration
* OS-based package installation
* Conditional tasks

Example:

```yaml id="jlwmj1"
when: ansible_os_family == "RedHat"
```

---

# 29. What is register & debug in Ansible?

# Register

## Definition

`register` stores output of a task into a variable.

---

## Example

```yaml id="jlwm2u"
- name: Check disk usage
  shell: df -h
  register: disk_output
```

Now:

```yaml id="jlwm0w"
disk_output.stdout
```

contains command output.

---

# Debug

## Definition

`debug` prints variables/output during playbook execution.

---

## Example

```yaml id="jlwmcr"
- debug:
    var: disk_output.stdout
```

---

# Real-Time Example

Used for:

* Troubleshooting
* Logging
* Validation
* Dynamic decision making

---

# Full Example

```yaml id="jlwmxv"
- name: Check Java Version
  shell: java -version
  register: java_output

- debug:
    var: java_output.stderr
```

---

# What is changed_when in Ansible?

## Definition

`changed_when` controls whether Ansible reports a task as:

* Changed
  or
* Unchanged

Useful when commands do not actually modify anything.

---

# Example

```yaml id="jlwmx8"
- name: Check service status
  shell: systemctl status httpd
  register: result
  changed_when: false
```

Even if command runs successfully:

* Task will show `ok`
  instead of `changed`.

---

# Why Important?

Without `changed_when`:

* Every shell/command task may show as changed.

This causes:

* Incorrect reporting
* Unnecessary handler execution

---

# Real-Time Example

## Health Check Tasks

```yaml id="jlwm8t"
- name: Check disk usage
  shell: df -h
  register: disk
  changed_when: false
```

Health checks should not mark servers as changed.

---

# Advanced Example

```yaml id="jlwmvu"
- shell: cat /tmp/app_status
  register: output
  changed_when: "'updated' in output.stdout"
```

Only marks task changed if:

```text id="jlwm0z"
updated
```

exists in output.

---

# Senior-Level Interview Tips

## Q1: Why use Templates instead of Copy?

Answer:
Templates support dynamic configuration.

---

## Q2: Why avoid shell module?

Answer:

* Less secure
* Not idempotent
* Vulnerable to shell interpretation

---

## Q3: What are Facts?

Answer:
System information automatically gathered by setup module.

---

# Enterprise Workflow Example

```text id="jlwm1d"
Dynamic Inventory
        ↓
Group Variables Loaded
        ↓
Role Executes
        ↓
Templates Render Configurations
        ↓
Modules Configure Servers
        ↓
Register Captures Outputs
        ↓
Debug Validates Deployment
```

# 30. What is changed_when in Ansible?

## Definition

`changed_when` is used to control the task status in Ansible.

Normally:

* Ansible marks shell/command tasks as `changed`
  even if nothing actually changed.

`changed_when` helps override this behavior.

---

# Syntax

```yaml id="7a1wpm"
changed_when: false
```

---

# Example

```yaml id="g6j8vw"
- name: Check Apache Status
  shell: systemctl status httpd
  register: apache_status
  changed_when: false
```

Output:

```text id="bwhx2j"
ok
```

instead of:

```text id="p8o3wd"
changed
```

---

# Why Important?

Without `changed_when`:

* Health-check tasks appear as changed
* Handlers may trigger unnecessarily
* CI/CD reports become inaccurate

---

# Real-Time Example

Production Monitoring Playbook:

```yaml id="phv1lw"
- name: Check Disk Usage
  shell: df -h
  register: disk
  changed_when: false
```

This is only validation, not configuration change.

---

# Advanced Example

```yaml id="1n6g5v"
- shell: cat /tmp/deploy.log
  register: deploy
  changed_when: "'SUCCESS' in deploy.stdout"
```

Task marked changed only when:

```text id="mrls6j"
SUCCESS
```

exists.

---

# 31. Can we disable automatic facts gathering in Ansible?

## Yes

By default Ansible gathers facts using the `setup` module.

This increases execution time.

---

# Disable Facts Gathering

```yaml id="h4hzq2"
---
- hosts: web
  gather_facts: no

  tasks:
    - name: Install Apache
      yum:
        name: httpd
        state: present
```

---

# Why Disable Facts?

Useful when:

* Large infrastructure
* Faster execution required
* Facts not needed

---

# Real-Time Example

Suppose:

* 500 servers
* CI/CD deployment pipeline

Fact gathering adds extra time.

Disabling facts improves deployment speed.

---

# Important Note

If facts are disabled:

```yaml id="cbsmwh"
{{ ansible_hostname }}
```

won’t work unless setup module is run manually.

---

# Manual Fact Gathering

```yaml id="m3k91w"
- setup:
```

---

# 32. How error handling can be done in Ansible?

## Methods for Error Handling

| Method              | Purpose                  |
| ------------------- | ------------------------ |
| ignore_errors       | Ignore failed tasks      |
| failed_when         | Custom failure condition |
| changed_when        | Custom changed status    |
| block/rescue/always | Exception handling       |
| retries/until       | Retry failed tasks       |

---

# Using block & rescue

## Example

```yaml id="0z0kdb"
- block:

    - name: Install Package
      yum:
        name: nginx
        state: present

  rescue:

    - name: Print Failure Message
      debug:
        msg: "Package installation failed"

  always:

    - name: Always Execute
      debug:
        msg: "Task completed"
```

---

# Real-Time Example

Used in:

* Production deployments
* Database migrations
* Critical application deployment

If deployment fails:

* Rollback tasks run automatically.

---

# 33. How to ignore failed commands in Ansible?

## Using ignore_errors

```yaml id="f2l3dy"
- name: Stop Service
  service:
    name: testservice
    state: stopped
  ignore_errors: yes
```

Even if service does not exist:

* Playbook continues.

---

# Real-Time Example

Suppose:

* Old package may or may not exist

```yaml id="y7n47t"
- name: Remove old package
  yum:
    name: oldapp
    state: absent
  ignore_errors: yes
```

Useful during migrations.

---

# Interview Tip

Use carefully.

Avoid hiding critical failures.

---

# Alternative Better Approach

```yaml id="6m8d4n"
failed_when: false
```

Provides better control.

---

# 34. What is handlers? Why do we use Handlers in Ansible?

## Definition

Handlers are special tasks that run only when notified.

Used mainly for:

* Restarting services
* Reloading configurations

---

# Why Use Handlers?

Without handlers:

* Services restart unnecessarily

Handlers improve:

* Performance
* Efficiency
* Idempotency

---

# Example

```yaml id="p8s1y2"
tasks:

- name: Update Apache Config
  template:
    src: httpd.conf.j2
    dest: /etc/httpd/conf/httpd.conf
  notify:
    - Restart Apache

handlers:

- name: Restart Apache
  service:
    name: httpd
    state: restarted
```

---

# Behavior

Handler runs only if:

```text id="o1m4sz"
template task changed
```

---

# Real-Time Example

Enterprise Deployment:

* Update Nginx config
* Restart Nginx only if config changes

Prevents unnecessary downtime.

---

# 35. What is Privilege Escalation in Ansible?

## Definition

Privilege escalation allows Ansible to execute tasks as another user.

Usually:

* Connect using normal user
* Execute tasks as root/sudo user

---

# Keywords

| Keyword       | Purpose                     |
| ------------- | --------------------------- |
| become        | Enable privilege escalation |
| become_user   | Run as specific user        |
| become_method | sudo/su/pbrun               |

---

# Example

```yaml id="4fjc5r"
- hosts: web
  become: yes

  tasks:
    - name: Install Apache
      yum:
        name: httpd
        state: present
```

Runs task as:

```text id="jlwmvb"
root
```

---

# Connect with One User & Execute with Another

## Inventory

```ini id="jlwm03"
[web]
10.0.0.10 ansible_user=devops
```

---

## Playbook

```yaml id="jlwm7r"
- hosts: web
  become: yes
  become_user: root

  tasks:
    - name: Create Directory
      file:
        path: /opt/app
        state: directory
```

---

# Real-Time Example

Enterprise Security Policy:

* SSH access only through service account
* Administrative tasks via sudo

Common in:

* Banking
* Healthcare
* Government projects

---

# 36. How to decrypt a vault file? How to encrypt a string using Ansible Vault?

# Decrypt Vault File

```bash id="jlwmz2"
ansible-vault decrypt secrets.yml
```

---

# Encrypt File

```bash id="jlwmn5"
ansible-vault encrypt secrets.yml
```

---

# Encrypt a String

```bash id="jlwm1k"
ansible-vault encrypt_string 'MyPassword'
```

---

# Example Output

```yaml id="jlwm9c"
my_secret: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          3234323432...
```

---

# Pass Password While Decrypting

```bash id="jlwm9r"
ansible-vault decrypt secrets.yml --ask-vault-pass
```

---

# Using Password File

```bash id="jlwmq1"
ansible-vault decrypt secrets.yml --vault-password-file vault.txt
```

---

# 37. If a file is encrypted using password & password is stored in a file how to pass the file to decrypt the file?

## Example

### Password File

```text id="jlwmh7"
vault_pass.txt
```

Contains:

```text id="jlwm1b"
MyVaultPassword
```

---

# Decrypt Command

```bash id="jlwm6u"
ansible-vault decrypt secrets.yml --vault-password-file vault_pass.txt
```

---

# Real-Time Example

CI/CD pipelines use:

* Jenkins secret files
* Azure DevOps secure files
* GitHub Actions secrets

---

# 38. If password file is encrypted then how to provide password while decrypting?

## Enterprise Approach

Usually:

* Password file itself stored securely
  using:
* Jenkins Credentials
* Azure Key Vault
* HashiCorp Vault

---

# Example with Environment Variable

```bash id="jlwm2y"
export ANSIBLE_VAULT_PASSWORD_FILE=vault_pass.txt
```

Run:

```bash id="jlwm0c"
ansible-playbook deploy.yml
```

---

# Advanced Enterprise Practice

Most enterprises avoid:

* Nested password encryption

Instead use:

* Secret management systems

Examples:

* [HashiCorp Vault](https://www.vaultproject.io/?utm_source=chatgpt.com)
* [Azure Key Vault](https://azure.microsoft.com/en-in/products/key-vault?utm_source=chatgpt.com)

---

# Real-Time Example

Jenkins Pipeline:

* Fetches vault password dynamically
* Stores in temporary runtime file
* Deletes after execution

---

# 39. What is Ansible Galaxy?

## Definition

[Ansible Galaxy](https://galaxy.ansible.com/?utm_source=chatgpt.com)

Ansible Galaxy is a repository for:

* Roles
* Collections
* Modules

It helps share reusable automation.

---

# Why Use It?

Instead of creating everything manually:

* Download prebuilt roles

Examples:

* Nginx
* Docker
* Kubernetes
* MySQL

---

# Install Role

```bash id="jlwmx0"
ansible-galaxy role install geerlingguy.nginx
```

---

# Install Collection

```bash id="jlwmw5"
ansible-galaxy collection install azure.azcollection
```

---

# Real-Time Example

Used heavily in:

* Enterprise automation
* Cloud deployments
* Kubernetes provisioning

Saves development time.

---

# 40. What is Tags in Ansible? Why it is used?

## Definition

Tags allow execution of specific tasks from a playbook.

Useful when:

* Running only selected tasks
* Faster deployments
* Troubleshooting

---

# Example

```yaml id="jlwmx6"
tasks:

- name: Install Apache
  yum:
    name: httpd
    state: present
  tags:
    - install

- name: Start Apache
  service:
    name: httpd
    state: started
  tags:
    - service
```

---

# Run Specific Tag

```bash id="jlwm4g"
ansible-playbook apache.yml --tags install
```

Runs only:

```text id="jlwmr2"
Install Apache
```

---

# Skip Tags

```bash id="jlwmwr"
ansible-playbook apache.yml --skip-tags service
```

---

# Real-Time Example

CI/CD Deployment:

* Only deploy application
* Skip DB migration
* Skip OS patching

Tags provide flexibility.

---

# Enterprise Use Cases of Tags

| Tag     | Purpose                |
| ------- | ---------------------- |
| install | Package installation   |
| config  | Configuration update   |
| deploy  | Application deployment |
| restart | Service restart        |
| backup  | Backup tasks           |

---

# Senior-Level Interview Tips

## Q1: Why use Handlers?

Answer:
Avoid unnecessary service restarts.

---

## Q2: Why disable facts gathering?

Answer:
Improve performance in large infrastructures.

---

## Q3: Difference Between ignore_errors & failed_when?

| ignore_errors   | failed_when                  |
| --------------- | ---------------------------- |
| Ignores failure | Defines custom failure logic |

---

# 41. What is lookup in Ansible playbook?

## Definition

Lookup plugins in Ansible are used to retrieve data from external sources into playbooks.

They fetch data from:

* Files
* Environment variables
* DNS
* APIs
* Password stores
* Cloud services

Lookup runs on:

```text id="wh1yfx"
Control Node
```

not on remote servers.

---

# Syntax

```yaml id="i8x7gd"
{{ lookup('plugin_name', 'source') }}
```

---

# Common Lookup Plugins

| Lookup   | Purpose                   |
| -------- | ------------------------- |
| file     | Read local file           |
| env      | Read environment variable |
| password | Generate password         |
| pipe     | Execute local command     |
| url      | Fetch URL content         |

---

# Example 1 — Read File

```yaml id="7qv72w"
- debug:
    msg: "{{ lookup('file', 'myfile.txt') }}"
```

---

# Example 2 — Environment Variable

```yaml id="h9p8sr"
- debug:
    msg: "{{ lookup('env', 'HOME') }}"
```

---

# Real-Time Example

CI/CD pipeline:

* Jenkins exports environment variables
* Ansible retrieves them dynamically

Example:

```yaml id="nvn7mz"
db_password: "{{ lookup('env', 'DB_PASS') }}"
```

Avoids hardcoding secrets.

---

# 42. How to control command failure in Ansible?

## Methods to Control Failure

| Method        | Purpose                  |
| ------------- | ------------------------ |
| ignore_errors | Ignore task failure      |
| failed_when   | Custom failure condition |
| block/rescue  | Exception handling       |
| retries/until | Retry failed commands    |

---

# 1. ignore_errors

```yaml id="7ly73z"
- shell: systemctl stop oldservice
  ignore_errors: yes
```

Playbook continues even if task fails.

---

# 2. failed_when

```yaml id="ff1vwv"
- shell: cat app.log
  register: result
  failed_when: "'ERROR' in result.stdout"
```

Task fails only if:

```text id="d2b6zc"
ERROR
```

exists.

---

# 3. retries & until

```yaml id="h8q9m3"
- shell: curl http://localhost
  register: output
  retries: 5
  delay: 10
  until: output.rc == 0
```

Retries until successful.

---

# Real-Time Example

Used in:

* Application deployment
* Kubernetes startup validation
* Database availability checks

---

# 43. How to debug your playbook?

# Methods Used for Debugging

| Method                          | Usage           |
| ------------------------------- | --------------- |
| debug module                    | Print variables |
| -vvv                            | Verbose output  |
| register                        | Capture output  |
| ansible-playbook --syntax-check | Validate syntax |
| check mode                      | Dry run testing |

---

# Using debug Module

```yaml id="i8x4pv"
- debug:
    var: ansible_hostname
```

---

# Using register + debug

```yaml id="q3k7yn"
- shell: uptime
  register: uptime_output

- debug:
    var: uptime_output.stdout
```

---

# Syntax Check

```bash id="qulxs9"
ansible-playbook site.yml --syntax-check
```

---

# Verbose Mode

```bash id="w0f7sp"
ansible-playbook site.yml -vvv
```

Shows detailed logs.

---

# Real-Time Example

In enterprise production:

* Validate deployments
* Troubleshoot failures
* Debug variable values
* Check task execution flow

---

# 44. What is Dry Run in Ansible & how to do that?

## Definition

Dry Run means:

> Simulating execution without making actual changes.

Also called:

```text id="gw7r3z"
Check Mode
```

---

# Command

```bash id="py0dz2"
ansible-playbook site.yml --check
```

---

# What Happens?

Ansible:

* Checks what would change
* Does not modify servers

---

# Example

```yaml id="7w9gmx"
- name: Install Apache
  yum:
    name: httpd
    state: present
```

Dry run shows:

```text id="9h2k5t"
Would install Apache
```

without actually installing it.

---

# Real-Time Example

Used before:

* Production deployments
* Infrastructure changes
* OS patching
* Security updates

Prevents accidental outages.

---

# Additional Validation

## Diff Mode

```bash id="udc5w9"
ansible-playbook site.yml --check --diff
```

Shows exact configuration changes.

---

# 45. How you can run all tasks at once?

## Default Behavior

By default Ansible runs tasks:

```text id="8b4s4x"
Sequentially
```

on all hosts.

---

# Parallel Execution

Controlled using:

```text id="2w7s8h"
forks
```

---

# Example

```bash id="w9x8mc"
ansible-playbook site.yml --forks 20
```

Runs tasks on:

```text id="q6s3dj"
20 hosts simultaneously
```

---

# Configuration

Inside:

```ini id="x3m7vt"
ansible.cfg
```

```ini id="8m2s0v"
forks = 20
```

---

# Strategy Plugins

## Free Strategy

```yaml id="m5h8rf"
strategy: free
```

Allows hosts to execute tasks independently.

---

# Example

```yaml id="t4p9dz"
- hosts: web
  strategy: free
```

---

# Real-Time Example

Enterprise deployments:

* Deploy application on 200 servers simultaneously
* Reduce deployment time dramatically

---

# 46. What is block in Ansible?

## Definition

`block` groups multiple tasks together.

Useful for:

* Error handling
* Logical grouping
* Exception handling

---

# Example

```yaml id="m8n2pk"
- block:

    - name: Install Apache
      yum:
        name: httpd
        state: present

    - name: Start Apache
      service:
        name: httpd
        state: started

  rescue:

    - debug:
        msg: "Installation failed"

  always:

    - debug:
        msg: "Execution completed"
```

---

# Sections

| Section | Purpose                |
| ------- | ---------------------- |
| block   | Main tasks             |
| rescue  | Runs if failure occurs |
| always  | Runs always            |

---

# Real-Time Example

Production Deployment:

* Deploy application
* If failure occurs → rollback
* Always send notification

---

# 47. What are different variable scopes?

## Variable Scopes in Ansible

| Scope      | Visibility      |
| ---------- | --------------- |
| Global     | Entire playbook |
| Play Scope | Specific play   |
| Host Scope | Specific host   |
| Role Scope | Inside role     |
| Task Scope | Specific task   |

---

# Example

## Play Scope

```yaml id="k8x5vr"
vars:
  app_port: 8080
```

Available only within that play.

---

# Host Scope

```yaml id="j9n3wc"
host_vars/web1.yml
```

Applies only to:

```text id="2d4s7y"
web1
```

---

# Real-Time Example

Enterprise multi-environment setup:

* Global vars → company-wide configs
* Host vars → server-specific configs
* Role vars → application configs

---

# 48. How variable precedence takes place?

## Definition

When same variable exists in multiple places:

* Ansible follows precedence order.

Higher precedence overrides lower precedence.

---

# Simplified Precedence Order

```text id="y2r5tm"
Extra Vars
   ↓
Task Vars
   ↓
Block Vars
   ↓
Role Vars
   ↓
Play Vars
   ↓
Inventory Vars
   ↓
Role Defaults
```

---

# Example

## Role Default

```yaml id="t5z7ql"
app_port: 80
```

---

## Extra Variable

```bash id="y6k3mf"
ansible-playbook site.yml -e "app_port=8080"
```

Final value:

```text id="x1w9nd"
8080
```

---

# Real-Time Example

Production deployment:

* Default values in role
* Environment-specific values in inventory
* Emergency override using extra vars

---

# 49. Difference between include & import?

| include                   | import                   |
| ------------------------- | ------------------------ |
| Dynamic                   | Static                   |
| Processed at runtime      | Processed during parsing |
| Supports loops/conditions | Limited dynamic behavior |
| More flexible             | Faster                   |

---

# include_tasks Example

```yaml id="r8w2pv"
- include_tasks: install.yml
  when: ansible_os_family == "RedHat"
```

Dynamic inclusion.

---

# import_tasks Example

```yaml id="m7p3zd"
- import_tasks: install.yml
```

Loaded before execution starts.

---

# Key Difference

## include

Evaluated during execution.

## import

Evaluated before execution.

---

# Real-Time Example

## include

Used for:

* Conditional execution
* Dynamic task loading

## import

Used for:

* Static reusable playbooks
* Standard deployment flow

---

# 50. How to include custom modules in Ansible?

## Definition

Custom modules are user-created modules used when built-in modules are insufficient.

Usually written in:

* Python
* PowerShell

---

# Directory Structure

```text id="g7w4pm"
project/
├── library/
│   └── mymodule.py
├── playbook.yml
```

---

# Example Custom Module

## mymodule.py

```python id="3z8d5q"
from ansible.module_utils.basic import AnsibleModule

def main():
    module = AnsibleModule(
        argument_spec=dict(
            name=dict(type='str', required=True)
        )
    )

    result = dict(
        changed=False,
        message=f"Hello {module.params['name']}"
    )

    module.exit_json(**result)

if __name__ == '__main__':
    main()
```

---

# Use in Playbook

```yaml id="z4n6qx"
- hosts: localhost

  tasks:
    - name: Run Custom Module
      mymodule:
        name: Rajesh
```

---

# Output

```text id="m1q8kt"
Hello Rajesh
```

---

# Real-Time Example

Custom modules used for:

* Internal APIs
* Proprietary tools
* Custom monitoring systems
* Enterprise integrations

Example:

* Internal CMDB integration
* Custom cloud provisioning
* Security validation

---

# Senior-Level Interview Tips

## Q1: Why use include instead of import?

Answer:
Use include for dynamic execution and conditions.

---

## Q2: Why Dry Run is important?

Answer:
Prevents production failures before actual deployment.

---

## Q3: Why use custom modules?

Answer:
To automate organization-specific operations not supported by built-in modules.

---

# 51. Ansible Task

# Task 1 – Enterprise DevOps Automation

# Part 1: Write Ansible Playbook to Automate Jenkins Deployment

## Real-Time Scenario

Company Requirement:

* Deploy Jenkins automatically on Linux server
* Configure Java
* Start Jenkins service
* Enable firewall rules

Used in:

* CI/CD environments
* DevOps automation platforms
* Enterprise build servers

---

# Jenkins Deployment Playbook

```yaml id="h1m7xq"
---
- name: Install and Configure Jenkins
  hosts: jenkins
  become: yes

  vars:
    jenkins_port: 8080

  tasks:

    - name: Install Java
      yum:
        name: java-17-openjdk
        state: present

    - name: Add Jenkins Repository
      get_url:
        url: https://pkg.jenkins.io/redhat-stable/jenkins.repo
        dest: /etc/yum.repos.d/jenkins.repo

    - name: Import Jenkins Key
      rpm_key:
        state: present
        key: https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key

    - name: Install Jenkins
      yum:
        name: jenkins
        state: present

    - name: Start Jenkins Service
      service:
        name: jenkins
        state: started
        enabled: yes

    - name: Open Firewall Port
      firewalld:
        port: "{{ jenkins_port }}/tcp"
        permanent: yes
        state: enabled
        immediate: yes
```

---

# Inventory File

```ini id="c7n5dv"
[jenkins]
192.168.1.50
```

---

# Run Playbook

```bash id="m4q9zs"
ansible-playbook -i inventory.ini jenkins.yml
```

---

# Real-Time Enterprise Workflow

```text id="k2x7mp"
Developer Push
      ↓
Jenkins Trigger
      ↓
Build Docker Image
      ↓
Push to Registry
      ↓
Deploy to Kubernetes
```

---

# Part 2: Write Ansible Role to Install Docker & Setup Kubernetes Cluster

# Role Structure

```text id="n8r2vw"
roles/
├── docker/
│   ├── tasks/
│   └── handlers/
│
├── kubernetes/
│   ├── tasks/
│   └── templates/
```

---

# Docker Role

## roles/docker/tasks/main.yml

```yaml id="z6t4ph"
---
- name: Install Docker
  yum:
    name: docker
    state: present

- name: Start Docker
  service:
    name: docker
    state: started
    enabled: yes
```

---

# Kubernetes Role

## roles/kubernetes/tasks/main.yml

```yaml id="y9p3wc"
---
- name: Disable Swap
  command: swapoff -a

- name: Install Kubernetes Packages
  yum:
    name:
      - kubeadm
      - kubelet
      - kubectl
    state: present

- name: Start Kubelet
  service:
    name: kubelet
    state: started
    enabled: yes
```

---

# Main Playbook

```yaml id="w8r5zn"
---
- hosts: kubernetes
  become: yes

  roles:
    - docker
    - kubernetes
```

---

# Real-Time Example

Used for:

* AKS worker preparation
* Kubernetes cluster bootstrap
* Enterprise container platforms

---

# Automate Jenkins Pipeline Creation for Docker & Kubernetes Deployment

# Jenkins Pipeline Example

## Jenkinsfile

```groovy id="m1x8pv"
pipeline {
    agent any

    stages {

        stage('Clone Code') {
            steps {
                git 'https://github.com/company/app.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t company/app:v1 .'
            }
        }

        stage('Push Docker Image') {
            steps {
                sh 'docker push company/app:v1'
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh 'kubectl apply -f deployment.yaml'
            }
        }
    }
}
```

---

# Real-Time Enterprise Example

This workflow automates:

* CI/CD pipeline
* Docker build
* Kubernetes deployment
* Rolling updates

Used in:

* Banking applications
* E-commerce platforms
* Microservices deployments

---

# Task 2 – Apache Deployment Automation

# Requirement

1. Install Apache
2. Create `/var/www/example.com`
3. Deploy sample HTML application
4. Configure Virtual Host
5. Set as default VirtualHost

---

# Ansible Playbook

```yaml id="j7p4wd"
---
- name: Configure Apache Web Server
  hosts: web
  become: yes

  vars:
    web_root: /var/www/example.com

  tasks:

    - name: Install Apache
      yum:
        name: httpd
        state: present

    - name: Create Web Directory
      file:
        path: "{{ web_root }}"
        state: directory
        mode: '0755'

    - name: Deploy Sample HTML Page
      copy:
        dest: "{{ web_root }}/index.html"
        content: |
          <html>
          <h1>Welcome to Example.com</h1>
          </html>

    - name: Configure Virtual Host
      copy:
        dest: /etc/httpd/conf.d/example.com.conf
        content: |
          <VirtualHost *:80>
              ServerName example.com
              DocumentRoot {{ web_root }}

              <Directory {{ web_root }}>
                  AllowOverride All
                  Require all granted
              </Directory>
          </VirtualHost>

    - name: Start Apache
      service:
        name: httpd
        state: started
        enabled: yes

    - name: Restart Apache
      service:
        name: httpd
        state: restarted
```

---

# Inventory

```ini id="y2m8sq"
[web]
192.168.1.100
```

---

# Run Playbook

```bash id="u6w4pk"
ansible-playbook -i inventory.ini apache.yml
```

---

# Real-Time Example

Used for:

* Hosting enterprise web applications
* Internal portals
* Reverse proxy configurations
* Static web deployments

---

# 52. How do you connect Ansible master to a client? What is the process?

## Architecture

```text id="f3n7mv"
Ansible Control Node
        |
        | SSH / WinRM
        ↓
Managed Nodes (Clients)
```

---

# Step-by-Step Process

# Step 1: Install Ansible on Control Node

```bash id="x9q4hz"
yum install ansible -y
```

---

# Step 2: Configure Inventory

```ini id="v5r2wk"
[web]
192.168.1.10
```

---

# Step 3: Generate SSH Key

```bash id="b1t7np"
ssh-keygen
```

---

# Step 4: Copy SSH Key to Client

```bash id="q4z8yc"
ssh-copy-id user@192.168.1.10
```

This enables passwordless authentication.

---

# Step 5: Test Connectivity

```bash id="m8w5rl"
ansible all -m ping
```

---

# Successful Output

```text id="t7p2vh"
SUCCESS => pong
```

---

# Windows Client Connection

Uses:

```text id="j2n4xp"
WinRM
```

Example Inventory:

```ini id="r9v6zd"
[windows]
10.0.0.50

[windows:vars]
ansible_connection=winrm
ansible_user=Administrator
ansible_password=Password123
```

---

# Real-Time Enterprise Setup

## Linux Servers

* SSH keys
* Service accounts
* Sudo access

## Windows Servers

* WinRM
* Domain authentication

---

# Enterprise Security Best Practices

| Best Practice        | Purpose               |
| -------------------- | --------------------- |
| SSH Keys             | Secure authentication |
| Vault                | Protect secrets       |
| RBAC                 | Access control        |
| Separate inventories | Dev/QA/Prod isolation |
| Bastion host         | Secure remote access  |

---

# Senior-Level Interview Tips

## Q1: Why SSH Keys Instead of Passwords?

Answer:

* More secure
* Automation friendly
* No manual password entry

---

## Q2: Why use Roles in Kubernetes setup?

Answer:

* Reusable automation
* Modular structure
* Easier maintenance

---

## Q3: How Jenkins integrates with Ansible?

Answer:
Jenkins triggers Ansible playbooks during CI/CD deployments.

---



