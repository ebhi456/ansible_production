# Ansible AWX / Tower — Real-Time Production Example

## 100 Production Servers: Nginx on 50 Servers and RabbitMQ on 50 Servers

---

## 1. Real-World Production Scenario

Assume a company has **100 production Linux servers**.

The servers are running different operating system versions and have different application requirements.

For example:

```text
Total Production Servers = 100

        Production Environment
                 │
        ┌────────┴────────┐
        │                 │
     50 Servers        50 Servers
        │                 │
      Nginx            RabbitMQ
```

The requirement is:

```text
Servers 001 - 050
        ↓
Install and configure Nginx

Servers 051 - 100
        ↓
Install and configure RabbitMQ
```

However, in a real production environment, we normally **do not depend on server numbering**.

Instead, servers are classified using meaningful inventory groups such as:

```text
[nginx_servers]

[rabbitmq_servers]
```

This makes the automation easier to maintain.

---

# 2. Why Ansible AWX/Tower?

Without AWX, an engineer might run:

```bash
ansible-playbook -i inventory.ini site.yml
```

from a workstation or Ansible control node.

That can work for a small environment.

But with 100 production servers, organizations usually need centralized:

* Inventory management
* Credentials
* Role-Based Access Control
* Job Templates
* Scheduling
* Job history
* Logs
* Execution Environments
* Git integration
* Approval/workflow processes
* API integration
* Notifications

AWX/Automation Controller provides these capabilities around Ansible automation.

A job template acts as a reusable definition for running an Ansible job and can contain the project, playbook, inventory, credentials, execution environment and other parameters.

---

# 3. Production Architecture

A simplified production architecture would look like this:

```text
                         USERS
                           │
                           │
                  ┌────────▼────────┐
                  │      AWX        │
                  │                 │
                  │ Web UI / API    │
                  │ Job Scheduler   │
                  │ RBAC            │
                  └────────┬────────┘
                           │
                           │
                    ┌──────▼──────┐
                    │   Project   │
                    │             │
                    │ Git Repo    │
                    └──────┬──────┘
                           │
                           │
                    ┌──────▼──────┐
                    │ Job Template│
                    └──────┬──────┘
                           │
                 ┌─────────▼─────────┐
                 │ Execution         │
                 │ Environment       │
                 │                   │
                 │ Ansible           │
                 │ Python            │
                 │ Collections       │
                 └─────────┬─────────┘
                           │
                    Execution Node
                           │
              ┌────────────┴────────────┐
              │                         │
              ▼                         ▼
       Nginx Servers               RabbitMQ Servers
          50 Servers                  50 Servers
```

The execution environment is a container image containing the Ansible runtime, collections, Python libraries and system dependencies required by the automation. This separates the automation execution environment from the control plane and helps keep execution consistent.

---

# 4. The Important Design Principle

The most important point is:

> **Do not create 100 different playbooks for 100 servers.**

Instead, create reusable automation.

For example:

```text
One playbook
      │
      ├── nginx_servers
      │
      └── rabbitmq_servers
```

Or, even better:

```text
Roles
 │
 ├── nginx
 │
 └── rabbitmq
```

Then inventory determines which servers receive which role.

---

# 5. Production Inventory Design

A simple inventory could look like this:

```ini
[nginx_servers]
prod-web-01
prod-web-02
prod-web-03
prod-web-04
prod-web-05
...
prod-web-50

[rabbitmq_servers]
prod-rmq-01
prod-rmq-02
prod-rmq-03
prod-rmq-04
prod-rmq-05
...
prod-rmq-50
```

In a real environment, these hosts might be IP addresses, DNS names, or dynamically generated cloud inventory entries.

For example:

```ini
[nginx_servers]
10.0.1.10
10.0.1.11
10.0.1.12

[rabbitmq_servers]
10.0.2.10
10.0.2.11
10.0.2.12
```

---

# 6. Better Production Inventory

Instead of putting everything into one large inventory file, we can organize servers by function and environment.

For example:

```text
inventory/
├── production/
│   ├── hosts.yml
│   └── group_vars/
│       ├── all.yml
│       ├── nginx_servers.yml
│       └── rabbitmq_servers.yml
│
└── staging/
    ├── hosts.yml
    └── group_vars/
        ├── all.yml
        ├── nginx_servers.yml
        └── rabbitmq_servers.yml
```

This allows us to maintain separate configuration for:

```text
Production
Staging
Development
```

---

# 7. Example Production Inventory

`inventory/production/hosts.yml`

```yaml
all:

  children:

    production:

      children:

        nginx_servers:

          hosts:
            prod-web-01:
              ansible_host: 10.0.1.10

            prod-web-02:
              ansible_host: 10.0.1.11

            prod-web-03:
              ansible_host: 10.0.1.12

            prod-web-04:
              ansible_host: 10.0.1.13

            prod-web-05:
              ansible_host: 10.0.1.14

            # ...
            # Continue until prod-web-50


        rabbitmq_servers:

          hosts:
            prod-rmq-01:
              ansible_host: 10.0.2.10

            prod-rmq-02:
              ansible_host: 10.0.2.11

            prod-rmq-03:
              ansible_host: 10.0.2.12

            prod-rmq-04:
              ansible_host: 10.0.2.13

            prod-rmq-05:
              ansible_host: 10.0.2.14

            # ...
            # Continue until prod-rmq-50
```

---

# 8. Handling Different Server Versions

This is where Ansible becomes very useful.

Suppose the 50 Nginx servers are running:

```text
Ubuntu 22.04
Ubuntu 24.04
RHEL 8
RHEL 9
```

We don't want four completely different playbooks.

Instead, Ansible can use variables and OS-specific tasks.

Example:

```text
nginx_servers
      │
      ├── Ubuntu
      │
      ├── RHEL 8
      │
      └── RHEL 9
```

The playbook can use:

```yaml
ansible_facts.os_family
```

For example:

```yaml
- name: Install Nginx
  ansible.builtin.package:
    name: nginx
    state: present
```

The `package` module provides a generic interface and allows Ansible to use the appropriate package manager.

Therefore, you don't necessarily need:

```text
apt install nginx
```

and:

```text
yum install nginx
```

in every playbook.

---

# 9. Production Git Repository Structure

A clean repository could look like this:

```text
ansible-production-automation/
│
├── ansible.cfg
├── requirements.yml
├── site.yml
├── README.md
│
├── inventory/
│   │
│   ├── production/
│   │   ├── hosts.yml
│   │   │
│   │   └── group_vars/
│   │       ├── all.yml
│   │       ├── nginx_servers.yml
│   │       └── rabbitmq_servers.yml
│   │
│   └── staging/
│       ├── hosts.yml
│       └── group_vars/
│
├── playbooks/
│   │
│   ├── nginx.yml
│   └── rabbitmq.yml
│
├── roles/
│   │
│   ├── nginx/
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   ├── handlers/
│   │   │   └── main.yml
│   │   ├── templates/
│   │   │   └── nginx.conf.j2
│   │   ├── defaults/
│   │   │   └── main.yml
│   │   └── vars/
│   │       └── main.yml
│   │
│   └── rabbitmq/
│       ├── tasks/
│       │   └── main.yml
│       ├── handlers/
│       │   └── main.yml
│       ├── templates/
│       │   └── rabbitmq.conf.j2
│       ├── defaults/
│       │   └── main.yml
│       └── vars/
│           └── main.yml
│
└── .gitignore
```

This is a good structure for a production-style demonstration.

---

# 10. ansible.cfg

Example:

`ansible.cfg`

```ini
[defaults]

inventory = inventory/production/hosts.yml

host_key_checking = False

retry_files_enabled = False

stdout_callback = default

interpreter_python = auto_silent

forks = 20
```

The `forks` setting controls how many hosts Ansible can process concurrently from an Ansible process.

For a production environment, the value should be selected according to the controller/execution-node capacity and workload rather than blindly setting a large number.

---

# 11. requirements.yml

If the automation requires Ansible collections, define them explicitly.

Example:

`requirements.yml`

```yaml
---
collections:

  - name: ansible.posix

  - name: community.general

  - name: community.rabbitmq
```

If your automation requires Docker:

```yaml
---
collections:

  - name: community.docker
```

The important production principle is:

> Dependencies should be version-controlled and included in the Execution Environment rather than relying on whatever happens to be installed on a particular machine.

Execution Environments are specifically designed to package Ansible Core, collections, Python libraries and system dependencies into a consistent runtime.

---

# 12. Nginx Playbook

`playbooks/nginx.yml`

```yaml
---
- name: Install and configure Nginx
  hosts: nginx_servers
  become: true

  roles:
    - nginx
```

Notice:

```yaml
hosts: nginx_servers
```

This is extremely important.

It means:

> Only machines belonging to the `nginx_servers` inventory group should receive this automation.

Therefore:

```text
100 servers
    │
    ├── 50 nginx_servers
    │       │
    │       └── nginx.yml
    │
    └── 50 rabbitmq_servers
            │
            └── NOT touched
```

---

# 13. Nginx Role

`roles/nginx/tasks/main.yml`

```yaml
---
- name: Install Nginx
  ansible.builtin.package:
    name: nginx
    state: present

- name: Deploy Nginx configuration
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: "0644"
  notify:
    - Restart Nginx

- name: Ensure Nginx is enabled and running
  ansible.builtin.service:
    name: nginx
    state: started
    enabled: true
```

---

# 14. Nginx Handler

`roles/nginx/handlers/main.yml`

```yaml
---
- name: Restart Nginx
  ansible.builtin.service:
    name: nginx
    state: restarted
```

The advantage of handlers is that Nginx is restarted only when the configuration changes.

For example:

```text
Configuration unchanged
        │
        └── No restart

Configuration changed
        │
        └── Restart Nginx
```

This is safer for production.

---

# 15. Nginx Template

`roles/nginx/templates/nginx.conf.j2`

Example:

```nginx
user nginx;

worker_processes auto;

events {
    worker_connections 1024;
}

http {

    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    sendfile on;

    keepalive_timeout 65;

    server {

        listen 80;

        server_name {{ nginx_server_name }};

        location / {

            proxy_pass {{ nginx_backend_url }};

            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        }
    }
}
```

The important part is:

```text
{{ nginx_server_name }}

{{ nginx_backend_url }}
```

These values can come from Ansible variables.

---

# 16. Nginx Variables

`inventory/production/group_vars/nginx_servers.yml`

```yaml
---
nginx_server_name: "app.example.com"

nginx_backend_url: "http://backend.internal:8080"
```

Now all 50 Nginx servers can receive the same configuration.

If one server needs a different configuration, you can override variables at the host level.

---

# 17. RabbitMQ Playbook

`playbooks/rabbitmq.yml`

```yaml
---
- name: Install and configure RabbitMQ
  hosts: rabbitmq_servers
  become: true

  roles:
    - rabbitmq
```

Again:

```yaml
hosts: rabbitmq_servers
```

means only the 50 RabbitMQ servers are targeted.

---

# 18. RabbitMQ Role

`roles/rabbitmq/tasks/main.yml`

```yaml
---
- name: Install RabbitMQ
  ansible.builtin.package:
    name: rabbitmq-server
    state: present

- name: Ensure RabbitMQ is enabled and running
  ansible.builtin.service:
    name: rabbitmq-server
    state: started
    enabled: true
```

---

# 19. RabbitMQ Configuration

Example:

`roles/rabbitmq/templates/rabbitmq.conf.j2`

```text
listeners.tcp.default = {{ rabbitmq_port }}

management.tcp.port = {{ rabbitmq_management_port }}

loopback_users.guest = false
```

---

# 20. RabbitMQ Variables

`inventory/production/group_vars/rabbitmq_servers.yml`

```yaml
---
rabbitmq_port: 5672

rabbitmq_management_port: 15672
```

---

# 21. Main Playbook

We can create a single entry point.

`site.yml`

```yaml
---
- import_playbook: playbooks/nginx.yml

- import_playbook: playbooks/rabbitmq.yml
```

Then:

```bash
ansible-playbook site.yml
```

would execute both playbooks.

However, in production, I would generally create **separate AWX Job Templates** for these independent services.

For example:

```text
AWX
 │
 ├── Install Nginx
 │
 └── Install RabbitMQ
```

This gives better control over permissions, approvals, schedules and audit history.

---

# 22. AWX Project

In AWX, create a Project:

```text
Project Name:
production-ansible

SCM Type:
Git

SCM URL:
https://github.com/company/ansible-production-automation.git
```

AWX synchronizes the project from Git.

Conceptually:

```text
GitHub
   │
   │ git clone / project sync
   ▼
AWX Project
   │
   ├── playbooks/
   ├── roles/
   ├── inventory/
   └── requirements.yml
```

---

# 23. AWX Inventory

Create:

```text
Inventory:
Production
```

Then create groups:

```text
Production
│
├── nginx_servers
│
└── rabbitmq_servers
```

The inventory can be static or dynamically sourced.

For cloud environments, organizations commonly use dynamic/cloud inventory sources rather than manually maintaining hundreds or thousands of IP addresses.

---

# 24. AWX Credentials

Create a Machine Credential.

Example:

```text
Credential Type:
Machine

Username:
ubuntu

SSH Private Key:
<private key>
```

AWX securely associates this credential with jobs.

The playbook should not contain:

```yaml
password: mypassword
```

or:

```yaml
private_key: |
  -----BEGIN PRIVATE KEY-----
```

Credentials should be managed by AWX.

---

# 25. Execution Environment

This is particularly important because of your previous `community.docker` issue.

For this example, you can create:

```text
production-ee
```

The Execution Environment can contain:

```text
Ansible Core
Python
ansible.posix
community.general
community.rabbitmq
```

For example:

```text
Execution Environment
│
├── Ansible Core
├── Python
├── ansible.posix
├── community.general
└── community.rabbitmq
```

Execution environments are container images and can be built using `ansible-builder`, pushed to a container registry, and selected on a job template.

---

# 26. Example Execution Environment Definition

Create:

`execution-environment.yml`

```yaml
---
version: 3

images:

  base_image:
    name: quay.io/ansible/awx-ee:latest

dependencies:

  galaxy:
    collections:

      - name: ansible.posix

      - name: community.general

      - name: community.rabbitmq
```

Then build:

```bash
ansible-builder build \
  -t company/production-ansible-ee:1.0
```

Login to your registry:

```bash
docker login
```

Push:

```bash
docker push company/production-ansible-ee:1.0
```

Then configure the image in AWX:

```text
AWX
 │
 └── Execution Environments
         │
         └── production-ansible-ee:1.0
```

---

# 27. Why Execution Environment Matters

Consider this situation:

```text
Developer laptop
│
├── Ansible
├── Python
└── community.rabbitmq
        │
        └── Works
```

But AWX:

```text
AWX
 │
 └── Default EE
       │
       └── community.rabbitmq missing
```

The job can fail.

Instead:

```text
AWX
 │
 └── production-ansible-ee
       │
       ├── Ansible
       ├── Python
       ├── ansible.posix
       ├── community.general
       └── community.rabbitmq
```

Now the automation runtime is explicitly defined.

This avoids the classic:

> "It works from my Ansible CLI but fails in AWX."

Execution environments are intended to provide consistent, shareable automation runtimes.

---

# 28. AWX Job Template — Install Nginx

Create:

```text
Job Template:
PROD - Install Nginx
```

Configure:

```text
Name:
PROD - Install Nginx

Inventory:
Production

Project:
production-ansible

Playbook:
playbooks/nginx.yml

Execution Environment:
production-ansible-ee

Credentials:
Production SSH
```

The relationship is:

```text
PROD - Install Nginx
        │
        ├── Project
        │     └── Git Repository
        │
        ├── Inventory
        │     └── nginx_servers
        │
        ├── Credential
        │     └── SSH
        │
        ├── Playbook
        │     └── nginx.yml
        │
        └── Execution Environment
              └── production-ansible-ee
```

---

# 29. AWX Job Template — Install RabbitMQ

Create another template:

```text
PROD - Install RabbitMQ
```

Configuration:

```text
Name:
PROD - Install RabbitMQ

Inventory:
Production

Project:
production-ansible

Playbook:
playbooks/rabbitmq.yml

Execution Environment:
production-ansible-ee

Credentials:
Production SSH
```

---

# 30. What Happens When Nginx Job Is Launched?

An engineer clicks:

```text
Launch
```

AWX starts the job.

Conceptually:

```text
User
 │
 ▼
AWX
 │
 ▼
Job Template
 │
 ├── Project
 ├── Inventory
 ├── Credentials
 ├── Playbook
 └── Execution Environment
 │
 ▼
Execution Node
 │
 ▼
Execution Environment
 │
 ▼
Ansible
 │
 ▼
nginx_servers
 │
 ├── Server 01
 ├── Server 02
 ├── Server 03
 ├── ...
 └── Server 50
```

Only the `nginx_servers` group is targeted.

---

# 31. What Happens When RabbitMQ Job Is Launched?

The process is similar:

```text
User
 │
 ▼
AWX
 │
 ▼
PROD - Install RabbitMQ
 │
 ├── Project
 ├── Inventory
 ├── Credentials
 ├── rabbitmq.yml
 └── Execution Environment
 │
 ▼
Execution Node
 │
 ▼
Ansible
 │
 ▼
rabbitmq_servers
 │
 ├── Server 01
 ├── Server 02
 ├── Server 03
 ├── ...
 └── Server 50
```

The Nginx servers are not targeted.

---

# 32. Complete Production Flow

The entire process looks like this:

```text
                         Developer
                            │
                            │ Git Push
                            ▼
                         GitHub
                            │
                            │ Project Sync
                            ▼
                    ┌─────────────────┐
                    │       AWX       │
                    │                 │
                    │ Project         │
                    │ Inventory       │
                    │ Credentials     │
                    │ Job Templates   │
                    └────────┬────────┘
                             │
                     ┌───────┴────────┐
                     │                │
                     ▼                ▼
               Nginx Job         RabbitMQ Job
               Template           Template
                     │                │
                     ▼                ▼
                nginx.yml       rabbitmq.yml
                     │                │
                     └───────┬────────┘
                             │
                             ▼
                    Execution Environment
                             │
                             ▼
                      Execution Node
                             │
                 ┌───────────┴───────────┐
                 │                       │
                 ▼                       ▼
           50 Nginx Servers        50 RabbitMQ Servers
```

---

# 33. What About Different Versions?

This is one of the most important real production considerations.

Suppose:

```text
Nginx Servers

20 servers → Ubuntu 22.04
20 servers → Ubuntu 24.04
10 servers → RHEL 9
```

You don't necessarily need:

```text
nginx-ubuntu22.yml
nginx-ubuntu24.yml
nginx-rhel9.yml
```

Instead:

```text
                    nginx.yml
                       │
            ┌──────────┼──────────┐
            │          │          │
        Ubuntu 22   Ubuntu 24   RHEL 9
            │          │          │
            └──────────┼──────────┘
                       │
                 Nginx Role
```

Ansible can use facts:

```yaml
ansible_facts.os_family
```

and variables.

---

# 34. OS-Specific Variables

You can structure variables like:

```text
roles/nginx/
│
├── vars/
│   ├── RedHat.yml
│   └── Debian.yml
```

For example:

`roles/nginx/vars/Debian.yml`

```yaml
nginx_service_name: nginx
nginx_config_path: /etc/nginx/nginx.conf
```

`roles/nginx/vars/RedHat.yml`

```yaml
nginx_service_name: nginx
nginx_config_path: /etc/nginx/nginx.conf
```

The role can load appropriate variables depending on the operating system.

---

# 35. Version-Specific Configuration

Suppose:

```text
Ubuntu 22.04
Nginx 1.18

Ubuntu 24.04
Nginx 1.24
```

You might have:

```yaml
nginx_config_version: "v1"
```

or:

```yaml
nginx_config_version: "v2"
```

based on inventory variables.

Example:

```yaml
nginx_config_version: "v1"
```

For specific hosts:

```yaml
prod-web-25:
  ansible_host: 10.0.1.35
  nginx_config_version: "v2"
```

This allows exceptions without duplicating the entire playbook.

---

# 36. Production Safety — Do Not Immediately Run Against All 50 Servers

This is extremely important.

In production, you normally should not do:

```text
50 servers
    ↓
Change configuration simultaneously
```

Instead:

```text
50 servers

     │
     ▼

Canary / Batch 1
5 servers

     │
     ▼
Validation

     │
     ▼

Batch 2
10 servers

     │
     ▼
Validation

     │
     ▼

Remaining servers
```

---

# 37. Using Serial

Ansible provides:

```yaml
serial:
```

Example:

```yaml
---
- name: Install Nginx
  hosts: nginx_servers
  become: true

  serial: 5

  roles:
    - nginx
```

Now Ansible processes approximately:

```text
Batch 1 → 5 servers
Batch 2 → 5 servers
Batch 3 → 5 servers
...
Batch 10 → 5 servers
```

This is much safer for production changes.

---

# 38. Production Deployment Strategy

A better approach could be:

```text
50 Nginx Servers

          │
          ▼

     Canary Group
       2 Servers

          │
          ▼

      Health Check

          │
       Success?
       /       \
     Yes        No
      │          │
      ▼          ▼
Continue       Stop
      │
      ▼
Remaining 48
```

You can implement this using inventory groups, `serial`, workflows and validation jobs.

---

# 39. AWX Workflow for Production

A mature AWX implementation might look like:

```text
                 Production Deployment
                         │
                         ▼
                 Pre-Deployment Check
                         │
                         ▼
                  Nginx Canary
                    2 Servers
                         │
                         ▼
                   Health Check
                    /       \
                Success     Failure
                   │           │
                   ▼           ▼
             Deploy Rest     Stop
             48 Servers
                   │
                   ▼
              Final Check
                   │
                   ▼
              Send Notification
```

AWX workflows can chain multiple job templates and other nodes together into one release process.

---

# 40. Example Health Check

After installing Nginx:

```yaml
- name: Verify Nginx service
  ansible.builtin.service_facts:

- name: Verify Nginx is running
  ansible.builtin.assert:
    that:
      - ansible_facts.services['nginx.service'].state == 'running'
    fail_msg: "Nginx is not running"
    success_msg: "Nginx is running successfully"
```

This provides an automated validation step.

---

# 41. RabbitMQ Production Consideration

RabbitMQ is different from Nginx.

Installing Nginx on 50 independent servers is relatively straightforward.

RabbitMQ often involves:

```text
RabbitMQ Cluster
       │
 ┌─────┼─────┐
 │     │     │
Node1 Node2 Node3
```

Therefore, in a real production environment, you need to think about:

* Cluster formation
* Node names
* Erlang cookie
* Cluster membership
* Queues
* Users
* Virtual hosts
* Permissions
* TLS
* High availability
* Monitoring
* Backup/recovery

So the RabbitMQ role would normally be more sophisticated than the simple installation example shown here.

---

# 42. Example RabbitMQ Inventory

Instead of simply:

```text
rabbitmq_servers
```

you might eventually use:

```text
rabbitmq_servers
rabbitmq_cluster_nodes
rabbitmq_primary
rabbitmq_secondary
```

For example:

```yaml
rabbitmq_cluster_nodes:

  hosts:

    prod-rmq-01:
      rabbitmq_node_role: primary

    prod-rmq-02:
      rabbitmq_node_role: secondary

    prod-rmq-03:
      rabbitmq_node_role: secondary
```

This allows the playbook to behave differently depending on the node role.

---

# 43. Idempotency

One of the biggest advantages of Ansible is idempotency.

Suppose Nginx is already installed.

Running:

```yaml
ansible.builtin.package:
  name: nginx
  state: present
```

again should not reinstall it unnecessarily.

Similarly:

```yaml
state: started
enabled: true
```

should ensure the desired state rather than blindly restarting the service every time.

The goal is:

```text
Current State
      │
      ▼
Ansible
      │
      ▼
Desired State
```

---

# 44. What Happens During a Re-Run?

Suppose the job succeeds:

```text
50 Nginx servers
        │
        ▼
Nginx installed
```

If you run the job again:

```text
AWX
 │
 ▼
Nginx Playbook
 │
 ▼
50 servers
 │
 └── Already configured
```

You should see mostly:

```text
ok
```

instead of:

```text
changed
```

This is one of the important characteristics of well-written Ansible automation.

---

# 45. Production Git Workflow

A realistic organization might use:

```text
Developer
    │
    ▼
Feature Branch
    │
    ▼
Pull Request
    │
    ▼
Code Review
    │
    ▼
CI Validation
    │
    ├── ansible-lint
    ├── YAML validation
    └── Test
    │
    ▼
Merge
    │
    ▼
main
    │
    ▼
AWX Project Sync
    │
    ▼
Production Job Template
```

This prevents engineers from directly modifying playbooks inside AWX.

The Git repository becomes the source of truth.

---

# 46. Example Git Branch Strategy

For example:

```text
main
 │
 ├── production
 │
 └── stable

feature/nginx-update
feature/rabbitmq-update
```

Engineer creates:

```text
feature/nginx-update
```

Changes:

```text
nginx.conf.j2
```

Then:

```text
Pull Request
     │
     ▼
Review
     │
     ▼
Merge
     │
     ▼
main
```

AWX then syncs the repository.

---

# 47. AWX Production Objects

The AWX side could look like:

```text
Organization
│
└── Production
    │
    ├── Project
    │    └── production-ansible
    │
    ├── Inventory
    │    └── Production
    │         ├── nginx_servers
    │         └── rabbitmq_servers
    │
    ├── Credentials
    │    └── Production SSH
    │
    ├── Execution Environments
    │    └── production-ansible-ee
    │
    ├── Job Templates
    │    ├── PROD - Install Nginx
    │    └── PROD - Install RabbitMQ
    │
    └── Workflow Templates
         └── Production Deployment
```

---

# 48. Who Does What?

This is useful for understanding the architecture.

```text
Git
│
└── Stores automation code

AWX Project
│
└── Syncs automation code

AWX Inventory
│
└── Defines target servers

AWX Credentials
│
└── Provides authentication

Job Template
│
└── Defines how a playbook is executed

Execution Environment
│
└── Provides Ansible + dependencies

Execution Node
│
└── Runs the automation

Ansible
│
└── Configures target servers

Target Servers
│
├── Nginx
└── RabbitMQ
```

---

# 49. Complete Real-Time Example

Imagine the ticket from the operations team says:

```text
CHG-12345

Install Nginx on the 50 production web servers.

Requirements:

- Nginx must be installed
- Nginx must be enabled
- Nginx must be running
- Deploy approved configuration
- Validate HTTP response
- Do not modify RabbitMQ servers
```

The engineer maps the requirement to:

```text
Inventory:
nginx_servers

Playbook:
playbooks/nginx.yml

Role:
roles/nginx

Job Template:
PROD - Install Nginx
```

Then the workflow becomes:

```text
CHG-12345
    │
    ▼
Git
    │
    ▼
AWX Project Sync
    │
    ▼
PROD - Install Nginx
    │
    ▼
Production Inventory
    │
    ▼
nginx_servers
    │
    ▼
50 Servers
    │
    ▼
Execution Environment
    │
    ▼
Ansible
    │
    ▼
Install Nginx
    │
    ▼
Configure Nginx
    │
    ▼
Start Nginx
    │
    ▼
Health Check
    │
    ▼
Job SUCCESS
```

---

# 50. What About the Other 50 Servers?

They are not touched.

This is because:

```yaml
hosts: nginx_servers
```

limits the playbook to:

```text
nginx_servers
```

The RabbitMQ servers belong to:

```text
rabbitmq_servers
```

Therefore:

```text
                 100 Servers
                     │
          ┌──────────┴──────────┐
          │                     │
          ▼                     ▼
   nginx_servers          rabbitmq_servers
      50 servers              50 servers
          │                     │
          ▼                     ▼
     nginx.yml             rabbitmq.yml
```

This separation is one of the fundamental concepts of Ansible inventory design.

---

# 51. Recommended Production Approach

For this scenario, I would recommend:

```text
1. Git Repository
        ↓
2. Inventory Groups
        ↓
3. Separate Roles
        ↓
4. Separate Playbooks
        ↓
5. AWX Project
        ↓
6. AWX Inventory
        ↓
7. AWX Credentials
        ↓
8. Custom Execution Environment
        ↓
9. Separate Job Templates
        ↓
10. Canary / Batch Deployment
        ↓
11. Validation
        ↓
12. Notifications
```

---

# 52. Final Repository Structure

The final demonstration repository could look like:

```text
ansible-production-automation/
│
├── README.md
├── ansible.cfg
├── requirements.yml
├── execution-environment.yml
├── site.yml
│
├── inventory/
│   └── production/
│       ├── hosts.yml
│       └── group_vars/
│           ├── all.yml
│           ├── nginx_servers.yml
│           └── rabbitmq_servers.yml
│
├── playbooks/
│   ├── nginx.yml
│   └── rabbitmq.yml
│
└── roles/
    │
    ├── nginx/
    │   ├── tasks/
    │   │   └── main.yml
    │   ├── handlers/
    │   │   └── main.yml
    │   ├── templates/
    │   │   └── nginx.conf.j2
    │   └── defaults/
    │       └── main.yml
    │
    └── rabbitmq/
        ├── tasks/
        │   └── main.yml
        ├── handlers/
        │   └── main.yml
        ├── templates/
        │   └── rabbitmq.conf.j2
        └── defaults/
            └── main.yml
```

---

# 53. Interview Explanation

If an interviewer asks:

> "How would you install Nginx on 50 production servers and RabbitMQ on another 50 servers using AWX?"

A strong answer would be:

> I would first classify the production servers into inventory groups such as `nginx_servers` and `rabbitmq_servers`. I would maintain the Ansible code, roles, templates and variables in Git and configure AWX as the centralized automation platform.
>
> I would create separate Ansible roles and playbooks for Nginx and RabbitMQ. The AWX Project would synchronize the Git repository, while the AWX Inventory would define the target groups. I would configure the required SSH credentials in AWX rather than hard-coding them in the playbooks.
>
> I would also use a controlled Execution Environment containing the required Ansible collections, Python libraries and system dependencies. AWX Job Templates would combine the project, inventory, playbook, credentials and execution environment.
>
> For production, I would avoid changing all 50 servers simultaneously. I would use a canary or batch approach, for example deploying to a small number of servers first, validating the service, and then continuing with the remaining servers. For Nginx, I could use `serial` in the playbook. For more complex deployments, I could use AWX Workflow Templates to chain deployment and validation jobs.
>
> Finally, I would enable job logging, notifications and RBAC so that the deployment is auditable and only authorized users can execute production changes.

---

# 54. The Architecture to Remember

The complete concept can be remembered as:

```text
                         GIT
                          │
                          ▼
                    AWX PROJECT
                          │
                          ▼
                    AWX INVENTORY
                          │
              ┌───────────┴───────────┐
              │                       │
              ▼                       ▼
        nginx_servers          rabbitmq_servers
              │                       │
              ▼                       ▼
         nginx.yml              rabbitmq.yml
              │                       │
              └───────────┬───────────┘
                          │
                          ▼
                   JOB TEMPLATE
                          │
                          ▼
              EXECUTION ENVIRONMENT
                          │
                          ▼
                  EXECUTION NODE
                          │
              ┌───────────┴───────────┐
              │                       │
              ▼                       ▼
        50 NGINX SERVERS        50 RABBITMQ SERVERS
              │                       │
              ▼                       ▼
          Validation              Validation
              │                       │
              └───────────┬───────────┘
                          ▼
                    AWX JOB RESULT
                          │
                  ┌───────┴───────┐
                  ▼               ▼
               SUCCESS          FAILURE
                  │               │
                  ▼               ▼
             Notification     Troubleshooting
```

---

# 55. Key Production Principles

Remember these principles when designing AWX automation:

```text
1. Git is the source of truth.

2. Inventory determines which servers are targeted.

3. Groups should represent server roles/functions.

4. Use reusable Ansible roles.

5. Avoid hard-coding credentials.

6. Use AWX Credentials.

7. Use Execution Environments for dependencies.

8. Use Job Templates for repeatable execution.

9. Use Workflows for multi-step deployments.

10. Use RBAC for production access control.

11. Use serial/batches for safer production changes.

12. Validate after deployment.

13. Keep production and staging inventories separate.

14. Use Git branches and pull requests for changes.

15. Make playbooks idempotent.

16. Maintain job history and auditability.

17. Use notifications for failures/success.

18. Treat RabbitMQ clustering as a separate architectural concern from simple package installation.
```

---

# 56. One-Line Mental Model

The easiest way to remember the complete architecture is:

```text
Git
 ↓
Project
 ↓
Inventory
 ↓
Job Template
 ↓
Execution Environment
 ↓
Execution Node
 ↓
Ansible
 ↓
Target Servers
```

And the inventory decides:

```text
nginx_servers
      ↓
Nginx automation

rabbitmq_servers
      ↓
RabbitMQ automation
```

This is the core production pattern for managing large server fleets with AWX/Ansible.
