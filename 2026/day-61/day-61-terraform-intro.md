# Day 61 — Introduction to Terraform and Your First AWS Infrastructure

## What I Did Today
Installed Terraform, configured AWS CLI, and used Infrastructure as Code to create and destroy real AWS resources (S3 bucket + EC2 instance) using nothing but a `.tf` file and a terminal.

---

## Task 1: Infrastructure as Code — Concepts in My Own Words

### 1. What is IaC and why does it matter in DevOps?
Infrastructure as Code means writing your servers, networks, and cloud resources in code files instead of clicking around in a console. It matters in DevOps because it makes infrastructure repeatable, version-controlled, and automated — the same way we treat application code. If your environment burns down, you can recreate it in minutes by running a single command.

### 2. What problems does IaC solve vs. manual AWS console clicks?
Manually creating resources in the AWS console is slow, error-prone, and impossible to reproduce exactly. If you create a VPC by hand, nobody knows what settings you chose six months later. IaC solves this by keeping everything in a file — you can review it, diff it, roll it back, and share it with teammates. It also eliminates "works on my environment" problems because everyone provisions from the same code.

### 3. How is Terraform different from CloudFormation, Ansible, and Pulumi?

| Tool | Key Difference |
|------|---------------|
| **AWS CloudFormation** | AWS-only. You're locked into one cloud. Terraform works across AWS, Azure, GCP, and more. |
| **Ansible** | Ansible is a configuration management tool — it installs software and configures running servers. Terraform provisions the infrastructure itself (VMs, buckets, networks). They complement each other. |
| **Pulumi** | Pulumi lets you write infrastructure in real programming languages (Python, TypeScript). Terraform uses its own language (HCL) which is simpler but less flexible for complex logic. |

### 4. What does "declarative" and "cloud-agnostic" mean?
**Declarative** means you describe *what* you want, not *how* to get there. You write `resource "aws_instance"` and Terraform figures out the API calls, the order of operations, and whether it needs to create, update, or skip. You don't write step-by-step instructions.

**Cloud-agnostic** means Terraform uses the same workflow (`init → plan → apply`) regardless of whether you're provisioning AWS, Azure, GCP, or even Kubernetes. You just swap the provider block.

---

## Task 2: Installation and Configuration

### Terraform Installation (Windows)
```bash
choco install terraform
```

### Verify Terraform
```bash
terraform -version
# Terraform v1.x.x
```

### AWS CLI Configuration
```bash
aws configure
# AWS Access Key ID: ****************
# AWS Secret Access Key: ****************
# Default region name: ap-south-1
# Default output format: json
```

### Verify AWS Access
```bash
aws sts get-caller-identity
```
```json
{
    "UserId": "AIDASAMPLEUSERID",
    "Account": "704930896680",
    "Arn": "arn:aws:iam::704930896680:user/terraform-user"
}
```

---

## Task 3: First Terraform Config — S3 Bucket

### main.tf
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-south-1"
}

resource "aws_s3_bucket" "my_bucket" {
  bucket = "terraweek-akash-2026"

  tags = {
    Name        = "TerraWeek-Day1"
    Environment = "Dev"
  }
}
```

### Commands Run
```bash
terraform init    # Downloaded the AWS provider plugin
terraform plan    # Previewed: 1 to add, 0 to change, 0 to destroy
terraform apply   # Created the S3 bucket (typed 'yes' to confirm)
```

### What did `terraform init` download?
`terraform init` downloaded the **HashiCorp AWS provider plugin** (`hashicorp/aws ~> 5.0`) from the Terraform Registry. It created a `.terraform/` directory containing:
- `providers/` — the compiled provider binary for the local OS/arch
- `.terraform.lock.hcl` — locks the exact provider version so the team uses the same one

### ✅ S3 Bucket confirmed in AWS Console
- Bucket name: `terraweek-akash-2026`
- Region: `ap-south-1` (Mumbai)
- Created: June 4, 2026

---

## Task 4: Adding an EC2 Instance

### Added to main.tf
```hcl
resource "aws_instance" "my_ec2" {
  ami           = "ami-0f5ee92e2d63afc18"
  instance_type = "t2.micro"

  tags = {
    Name = "TerraWeek-Day1"
  }
}
```

### terraform plan output
```
Plan: 1 to add, 0 to change, 0 to destroy.
```

### How did Terraform know the S3 bucket already exists?
Terraform reads `terraform.tfstate` before planning. The state file records every resource it has previously created, including the S3 bucket's ID and all its attributes. When `terraform plan` runs, it compares the state file against the current `main.tf` config — the bucket is already in state, so it skips it and only plans to create the new EC2 instance.

### ✅ EC2 Instance confirmed in AWS Console
- Instance ID: `i-07e5eb9f062dad40d`
- Type: `t2.micro`
- AMI: `ami-0f5ee92e2d63afc18` (Amazon Linux 2, ap-south-1)
- Tag: `Name = TerraWeek-Day1`

---

## Task 5: Understanding the State File

### Commands and What They Return

```bash
terraform show
```
Prints a human-readable view of every resource Terraform currently manages — all attributes, ARNs, IDs, tags, and configuration values. Like a full snapshot of your infrastructure.

```bash
terraform state list
```
```
aws_instance.my_ec2
aws_s3_bucket.my_bucket
```
Lists every resource address Terraform is tracking. Useful for quickly seeing what's under Terraform's control.

```bash
terraform state show aws_s3_bucket.my_bucket
```
```
# aws_s3_bucket.my_bucket:
resource "aws_s3_bucket" "my_bucket" {
    arn                         = "arn:aws:s3:::terraweek-akash-2026"
    bucket                      = "terraweek-akash-2026"
    bucket_regional_domain_name = "terraweek-akash-2026.s3.ap-south-1.amazonaws.com"
    region                      = "ap-south-1"
    ...
}
```
Shows the full detailed state of one specific resource, including every attribute AWS returned after creation.

```bash
terraform state show aws_instance.my_ec2
```
Shows the EC2 instance's full state — instance ID, private/public IPs, VPC, subnet, security groups, AMI, tags, and 38+ other attributes.

### State File Q&A

**What information does the state file store about each resource?**
The state file stores the resource type, logical name, provider, and every attribute AWS returned — ARNs, IDs, IP addresses, region, tags, encryption settings, creation metadata, and more. It is a complete JSON snapshot of your real infrastructure.

**Why should you never manually edit the state file?**
The state file is Terraform's source of truth. If you manually change it, Terraform may believe resources exist that don't, or vice versa — leading to failed plans, accidental destroys, or duplicate resources. Any state changes should be done via `terraform state` commands.

**Why should the state file not be committed to Git?**
The state file can contain sensitive data like passwords, private IPs, and secret ARNs. It also creates merge conflicts in team environments. The correct approach is to store state remotely in S3 + DynamoDB (remote backend) so it's shared, locked, and never in version control.

---

## Task 6: Modify, Plan, and Destroy

### Changed tag in main.tf
```hcl
tags = {
  Name = "TerraWeek-Modified"   # was "TerraWeek-Day1"
}
```

### terraform plan output
```
~ resource "aws_instance" "my_ec2" {
    ~ tags = {
        ~ "Name" = "TerraWeek-Day1" -> "TerraWeek-Modified"
      }
  }

Plan: 0 to add, 1 to change, 0 to destroy.
```

### What do the symbols mean?

| Symbol | Meaning |
|--------|---------|
| `~` | Update in-place — resource stays running, only the attribute changes |
| `+` | Create — a new resource will be added |
| `-` | Destroy — resource will be deleted |
| `-/+` | Destroy and recreate — the change requires replacement |

The tag change was a `~` (in-place update) — Terraform just called the AWS tag API on the running instance. No downtime, no destroy.

### terraform apply output
```
aws_instance.my_ec2: Modifying... [id=i-07e5eb9f062dad40d]
aws_instance.my_ec2: Modifications complete after 3s

Apply complete! Resources: 0 added, 1 changed, 0 destroyed.
```

### terraform destroy
```bash
terraform destroy
# type 'yes'
```
```
aws_instance.my_ec2: Destroying... [id=i-07e5eb9f062dad40d]
aws_s3_bucket.my_bucket: Destroying... [id=terraweek-akash-2026]
aws_instance.my_ec2: Destruction complete
aws_s3_bucket.my_bucket: Destruction complete

Destroy complete! Resources: 2 destroyed.
```

✅ Both the EC2 instance and S3 bucket were confirmed deleted in the AWS Console.

---

## Terraform Command Reference

| Command | What it does |
|---------|-------------|
| `terraform init` | Downloads provider plugins and sets up the working directory |
| `terraform plan` | Previews what will be created, changed, or destroyed — no real changes |
| `terraform apply` | Executes the plan and makes real changes to AWS |
| `terraform destroy` | Destroys all resources managed by the current state |
| `terraform show` | Prints a human-readable view of the current state |
| `terraform state list` | Lists all resource addresses Terraform is tracking |
| `terraform state show <resource>` | Shows detailed attributes of one specific resource |
| `terraform fmt` | Auto-formats `.tf` files to canonical HCL style |
| `terraform validate` | Checks syntax without connecting to AWS |

---

## .gitignore for Terraform Projects

```gitignore
# Terraform state files — contain sensitive data
*.tfstate
*.tfstate.backup

# Provider plugins — large binaries, re-downloaded by terraform init
.terraform/

# Variable files that may contain secrets
*.tfvars
*.tfvars.json
```

---

## Key Takeaways

- Terraform lets you define cloud infrastructure in `.tf` files and manage it with a simple `init → plan → apply` workflow
- The **state file** is Terraform's memory — it tracks what exists so it only changes what needs to change
- **Declarative IaC** means you describe the end state; Terraform figures out how to get there
- Always add `*.tfstate` and `.terraform/` to `.gitignore` — never commit them
- `terraform destroy` cleanly removes everything Terraform created — no orphaned resources

---

*Day 61 of #90DaysOfDevOps | #TerraWeek | #DevOpsKaJosh | #TrainWithShubham*
