# Ansible

Ansible

Author

Benedict Thekkel

## 🧰 What Is Ansible?

**Ansible** is an open-source IT automation and configuration management tool originally created by **Michael DeHaan**, now maintained by **Red Hat**.

It automates:

- Server configuration
- Application deployment
- Infrastructure orchestration
- Networking
- Security hardening
- CI/CD tasks

Unlike many tools, it is **agentless** (uses SSH by default).

------------------------------------------------------------------------

## 🧠 Core Philosophy

| Principle       | Meaning                                  |
|-----------------|------------------------------------------|
| Agentless       | No software installed on target machines |
| Declarative-ish | You define desired state                 |
| Idempotent      | Safe to run multiple times               |
| YAML-based      | Human-readable playbooks                 |
| Push model      | Controller pushes changes                |

------------------------------------------------------------------------

## 🏗 How Ansible Works

``` text
Control Node (your machine)
        |
        | SSH
        v
Managed Nodes (servers, VMs, containers)
```

It:

1.  Connects via SSH
2.  Copies module temporarily
3.  Executes
4.  Removes module
5.  Reports result

------------------------------------------------------------------------

## 📦 Core Components

### 1️⃣ Inventory

Defines hosts:

``` ini
[web]
192.168.1.10
192.168.1.11

[db]
db.local
```

Or dynamic (AWS, Azure, etc.).

------------------------------------------------------------------------

### 2️⃣ Playbooks

YAML automation definitions:

``` yaml
- name: Install Nginx
  hosts: web
  become: yes

  tasks:
    - name: Install package
      apt:
        name: nginx
        state: present
```

Run:

``` bash
ansible-playbook site.yml
```

------------------------------------------------------------------------

### 3️⃣ Modules

Reusable actions.

Examples:

| Category   | Modules          |
|------------|------------------|
| Packages   | apt, yum         |
| Files      | copy, template   |
| Services   | service          |
| Cloud      | ec2, azure_rm    |
| Containers | docker_container |
| K8s        | k8s              |

Over 3000+ modules available.

------------------------------------------------------------------------

### 4️⃣ Roles (Reusable Structure)

Standard layout:

``` text
roles/
  nginx/
    tasks/
    handlers/
    templates/
    vars/
```

Use in playbook:

``` yaml
roles:
  - nginx
```

------------------------------------------------------------------------

## 🔁 Idempotency

If nginx is already installed:

``` yaml
state: present
```

It does nothing.

Safe to rerun anytime.

------------------------------------------------------------------------

## 🧩 Variables

``` yaml
vars:
  http_port: 80
```

Override with:

- inventory
- group_vars
- host_vars
- extra vars

------------------------------------------------------------------------

## 🔐 Privilege Escalation

``` yaml
become: yes
become_method: sudo
```

------------------------------------------------------------------------

## 🌍 Dynamic Inventory

For cloud:

``` bash
ansible-inventory -i aws_ec2.yml --graph
```

Supports:

- AWS
- Azure
- GCP
- VMware

------------------------------------------------------------------------

## 🏗 Real-World Example

Provision + configure:

``` yaml
- hosts: all
  become: yes

  tasks:
    - name: Update apt
      apt:
        update_cache: yes

    - name: Install Docker
      apt:
        name: docker.io
        state: present
```

------------------------------------------------------------------------

## 🧠 Advanced Features

### 1️⃣ Templates (Jinja2)

``` yaml
- name: Configure Nginx
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
```

Dynamic config generation.

------------------------------------------------------------------------

### 2️⃣ Handlers

Triggered only on change:

``` yaml
handlers:
  - name: Restart nginx
    service:
      name: nginx
      state: restarted
```

------------------------------------------------------------------------

### 3️⃣ Loops

``` yaml
- name: Create users
  user:
    name: "{{ item }}"
  loop:
    - ben
    - admin
```

------------------------------------------------------------------------

### 4️⃣ Conditionals

``` yaml
when: ansible_os_family == "Debian"
```

------------------------------------------------------------------------

### 5️⃣ Tags

``` bash
ansible-playbook site.yml --tags nginx
```

------------------------------------------------------------------------

## ☁ Ansible + Cloud

You can:

- Create EC2 instances
- Configure them
- Deploy apps

Often used alongside:

**Terraform**

Common pattern:

| Tool      | Purpose               |
|-----------|-----------------------|
| Terraform | Create infrastructure |
| Ansible   | Configure OS + apps   |

------------------------------------------------------------------------

## 🏢 Ansible Tower / Automation Platform

**Red Hat Ansible Automation Platform**

Provides:

- Web UI
- RBAC
- Scheduling
- API
- Credential storage

------------------------------------------------------------------------

## 🔄 Push vs Pull

| Model | Example      |
|-------|--------------|
| Push  | Ansible      |
| Pull  | Puppet, Chef |

Push = you run it. Pull = nodes check in.

------------------------------------------------------------------------

## 🛠 Comparison With Other Tools

| Tool      | Agentless | Language | Focus              |
|-----------|-----------|----------|--------------------|
| Ansible   | ✅        | YAML     | Config mgmt        |
| Puppet    | ❌        | DSL      | Large infra        |
| Chef      | ❌        | Ruby     | Enterprise         |
| SaltStack | Mixed     | YAML     | Speed              |
| Terraform | N/A       | HCL      | Infra provisioning |

------------------------------------------------------------------------

## 🧱 When To Use Ansible

Use it for:

- OS configuration
- Deploying Django app
- Installing Docker
- Hardening Linux
- Rolling updates
- Managing clusters

------------------------------------------------------------------------

## 🚫 When NOT To Use It

Not ideal for:

- Creating thousands of cloud resources (Terraform better)
- High-frequency config drift enforcement
- Event-driven infra (use Kubernetes or cloud-native tools)

------------------------------------------------------------------------

## 🏗 Production Best Practices

### Structure

``` text
inventory/
  production
  staging

roles/
  app/
  db/

site.yml
```

------------------------------------------------------------------------

### Use Vault

Encrypt secrets:

``` bash
ansible-vault encrypt secrets.yml
```

------------------------------------------------------------------------

### Lint & Testing

- ansible-lint
- molecule (role testing)
- CI/CD validation

------------------------------------------------------------------------

## 🔥 Example: Deploy Django App

1.  Install Python
2.  Install Postgres
3.  Clone repo
4.  Setup venv
5.  Run migrations
6.  Configure Gunicorn
7.  Configure Nginx
8.  Setup systemd

All via one playbook.

------------------------------------------------------------------------

## 📊 Ansible Execution Flow

``` text
Inventory → Playbook → Tasks → Modules → Results
```

------------------------------------------------------------------------

## 🧠 Scaling Considerations

For large infra:

- Use forks
- Use SSH control master
- Use fact caching
- Use Tower for job scheduling

------------------------------------------------------------------------

## 🔮 Future of Ansible

- Event-driven automation
- Integration with GitOps
- Cloud-native modules
- Improved execution speed

------------------------------------------------------------------------

## 🏁 Summary

Ansible is best for:

✔ Server configuration ✔ Application deployment ✔ Cloud VM setup ✔ Security automation ✔ CI/CD provisioning

Pair it with Terraform for full infra lifecycle.

------------------------------------------------------------------------

If you’d like, I can next give:

- 🏗 Ansible for Proxmox homelab
- 🚀 Deploy Django + Postgres + Nginx with Ansible
- 🔐 Advanced Ansible security patterns
- ⚔ Terraform vs Ansible deep comparison
- 🧠 Ansible performance tuning guide

Tell me what direction you want.

------------------------------------------------------------------------

Back to top
