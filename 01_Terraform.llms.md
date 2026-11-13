# Terraform

**Terraform** is an open-source Infrastructure as Code (IaC) tool created by **HashiCorp** that lets you define cloud and infrastructure resources using declarative configuration files.

Author

Benedict Thekkel

## 🧠 Core Concepts

### 1️⃣ Infrastructure as Code (IaC)

You define infrastructure in **HCL (HashiCorp Configuration Language)**:

``` hcl
provider "aws" {
  region = "ap-southeast-2"
}

resource "aws_instance" "web" {
  ami           = "ami-123456"
  instance_type = "t3.micro"
}
```

Then run:

``` bash
terraform init
terraform plan
terraform apply
```

------------------------------------------------------------------------

### 2️⃣ Declarative Model

You describe **what you want**, not how to build it.

Terraform:

- Builds dependency graph
- Determines order
- Calculates diff
- Applies minimal changes

------------------------------------------------------------------------

### 3️⃣ Providers

Providers connect Terraform to platforms:

| Category   | Examples             |
|------------|----------------------|
| Cloud      | AWS, Azure, GCP, OCI |
| Containers | Kubernetes, Docker   |
| SaaS       | GitHub, Cloudflare   |
| On-prem    | VMware, Proxmox      |
| Databases  | PostgreSQL, MongoDB  |

👉 Yes — Terraform works with AWS, Oracle Cloud, Proxmox, etc.

------------------------------------------------------------------------

### 4️⃣ State File (Very Important)

Terraform tracks infrastructure in a **state file (`terraform.tfstate`)**.

It stores:

- Resource IDs
- Metadata
- Dependency graph

#### Why state matters:

- Enables diffing
- Prevents recreation
- Tracks drift

#### Best practice:

Use remote backend:

| Backend         | Use case  |
|-----------------|-----------|
| S3 + DynamoDB   | AWS teams |
| Azure Blob      | Azure     |
| GCS             | GCP       |
| Terraform Cloud | Managed   |

------------------------------------------------------------------------

## 🏗 Terraform Workflow

``` text
Write config → Init → Plan → Apply → Destroy
```

#### Commands Explained

| Command    | What it does           |
|------------|------------------------|
| `init`     | Downloads providers    |
| `plan`     | Shows execution plan   |
| `apply`    | Executes changes       |
| `destroy`  | Deletes infrastructure |
| `validate` | Syntax check           |
| `fmt`      | Format code            |
| `state`    | Inspect state          |

------------------------------------------------------------------------

## 🔄 How Terraform Works Internally

1.  Parses HCL
2.  Builds dependency graph
3.  Compares desired vs actual state
4.  Calls provider APIs
5.  Updates state

------------------------------------------------------------------------

## 🧩 Modules (Reusability)

Modules are reusable infrastructure packages.

Example:

``` hcl
module "vpc" {
  source = "./modules/vpc"
  cidr   = "10.0.0.0/16"
}
```

You can publish modules in:

- Terraform Registry
- Private Git repos

------------------------------------------------------------------------

## 📦 Variables & Outputs

#### Variables

``` hcl
variable "instance_type" {
  default = "t3.micro"
}
```

#### Outputs

``` hcl
output "public_ip" {
  value = aws_instance.web.public_ip
}
```

------------------------------------------------------------------------

## 🌍 Multi-Cloud Architecture

Terraform supports:

| Cloud   | Supported |
|---------|-----------|
| AWS     | ✅        |
| Azure   | ✅        |
| GCP     | ✅        |
| OCI     | ✅        |
| Alibaba | ✅        |

You can define multiple providers in one config.

------------------------------------------------------------------------

## 🔐 Security Best Practices

- Never commit `.tfstate`
- Use remote encrypted backend
- Use IAM roles (not static keys)
- Use workspaces for environments
- Restrict state access

------------------------------------------------------------------------

## 🧪 Workspaces (Environments)

``` bash
terraform workspace new prod
terraform workspace select dev
```

Enables:

- dev
- staging
- production

------------------------------------------------------------------------

## ⚙ Advanced Features

### 1️⃣ Data Sources

Read existing infrastructure:

``` hcl
data "aws_vpc" "default" {
  default = true
}
```

------------------------------------------------------------------------

### 2️⃣ Lifecycle Rules

``` hcl
lifecycle {
  prevent_destroy = true
}
```

------------------------------------------------------------------------

### 3️⃣ Count & For_each

Create multiple resources:

``` hcl
resource "aws_instance" "web" {
  count = 3
}
```

------------------------------------------------------------------------

### 4️⃣ Provisioners (Use Carefully)

Run scripts after creation:

``` hcl
provisioner "remote-exec" {}
```

⚠️ Not recommended for complex config — use Ansible instead.

------------------------------------------------------------------------

## 🚀 Terraform vs Other Tools

| Tool           | Type                    | Declarative?      | Language     |
|----------------|-------------------------|-------------------|--------------|
| Terraform      | IaC                     | Yes               | HCL          |
| Pulumi         | IaC                     | Yes               | Python/TS/Go |
| CloudFormation | AWS only                | Yes               | YAML         |
| Ansible        | Config mgmt             | Mostly procedural | YAML         |
| Kubernetes     | Container orchestration | Declarative       | YAML         |

------------------------------------------------------------------------

## 🔄 Terraform vs Kubernetes

| Terraform                 | Kubernetes              |
|---------------------------|-------------------------|
| Provisions infrastructure | Orchestrates containers |
| Creates clusters          | Runs workloads          |
| Cloud agnostic            | Cluster specific        |

They often work together.

------------------------------------------------------------------------

## 📉 Common Problems

#### 1️⃣ State Corruption

Fix with:

``` bash
terraform state rm
terraform import
```

#### 2️⃣ Drift

Use:

``` bash
terraform plan
```

#### 3️⃣ Large Teams

Use:

- Remote backend
- Locking
- CI/CD

------------------------------------------------------------------------

## 🏢 Terraform Cloud & Enterprise

**Terraform Cloud** provides:

- Remote runs
- State storage
- Policy as code (Sentinel)
- Team management

------------------------------------------------------------------------

## 🔥 Production Best Practices

#### Structure

    infrastructure/
      modules/
      envs/
        dev/
        prod/

#### CI/CD

- Run `plan` in PR
- Require approval
- Auto apply on merge

#### Version Pinning

``` hcl
required_providers {
  aws = {
    version = "~> 5.0"
  }
}
```

------------------------------------------------------------------------

## 📊 When NOT To Use Terraform

- Short-lived experiments
- Very dynamic runtime infra
- Fine-grained config management

------------------------------------------------------------------------

## 🧠 Advanced Patterns

- Blue/Green infra
- Multi-region failover
- Immutable infrastructure
- Policy as Code
- GitOps workflows

------------------------------------------------------------------------

## 🏗 Example: AWS VPC + EC2 + RDS

Terraform can build:

    VPC
     ├─ Public Subnet
     ├─ Private Subnet
     ├─ EC2 Instance
     └─ RDS Database

All reproducible with a single `apply`.

------------------------------------------------------------------------

## 🔮 Future of Terraform

- Increased policy automation
- OpenTofu (community fork)
- More cloud-neutral infra design
- AI-generated infrastructure configs

------------------------------------------------------------------------

## 🧭 If You’re Running Proxmox

Terraform has:

- Proxmox provider
- VMware provider
- Kubernetes provider

So yes — you can manage:

- VMs
- Containers
- Networks

------------------------------------------------------------------------

## 🏁 Summary

Terraform is best when:

- You want reproducible infrastructure
- You manage multiple environments
- You need version-controlled infra
- You want multi-cloud flexibility

It is less suited for:

- App runtime orchestration
- Fine config management
- One-off manual setups

------------------------------------------------------------------------

If you want, I can next give:

- 🔥 Real production folder structure example
- 🧠 Terraform for Proxmox homelab guide
- ☁ Terraform for AWS startup SaaS
- 🔐 Terraform security deep dive
- 🚀 Terraform vs Pulumi vs OpenTofu

Just tell me what direction you want.

------------------------------------------------------------------------

Back to top
