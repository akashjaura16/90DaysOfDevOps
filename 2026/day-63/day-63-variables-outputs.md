# Day 63 — Variables, Outputs, Data Sources and Expressions

## Overview

Day 62 left me with a working Terraform config full of hardcoded values — region, AMI IDs, CIDR blocks, instance types. One region change and everything breaks. Today I made it fully dynamic using variables, `.tfvars` files, outputs, data sources, locals, and built-in functions. The result is a config that works across environments without touching `main.tf`.

---

## Task 1: Extracting Variables

### `variables.tf`

```hcl
variable "region" {
  description = "AWS region to deploy into"
  type        = string
  default     = "ap-southeast-2"
}

variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "subnet_cidr" {
  description = "CIDR block for the public subnet"
  type        = string
  default     = "10.0.1.0/24"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

variable "project_name" {
  description = "Project name — used in resource names and tags (required)"
  type        = string
  # No default — Terraform will prompt if not provided
}

variable "environment" {
  description = "Deployment environment (dev, staging, prod)"
  type        = string
  default     = "dev"
}

variable "allowed_ports" {
  description = "List of ports to allow in the security group"
  type        = list(number)
  default     = [22, 80, 443]
}

variable "extra_tags" {
  description = "Additional tags to apply to all resources"
  type        = map(string)
  default     = {}
}
```

### The Five Terraform Variable Types

| Type | Example | Use Case |
|------|---------|----------|
| `string` | `"ap-southeast-2"` | Region names, AMI IDs, environment names |
| `number` | `8080` | Port numbers, counts, sizes |
| `bool` | `true` | Feature flags, enable/disable toggles |
| `list(type)` | `[22, 80, 443]` | Multiple values of the same type |
| `map(type)` | `{env = "dev"}` | Key-value pairs, tags, lookups |

> When I ran `terraform plan` without a `terraform.tfvars`, Terraform stopped and prompted: `var.project_name — Enter a value:` — exactly the behaviour you want for required inputs.

---

## Task 2: Variable Files and Precedence

### `terraform.tfvars` (dev — loaded automatically)

```hcl
project_name  = "terraweek"
environment   = "dev"
instance_type = "t2.micro"
```

### `prod.tfvars` (loaded explicitly with `-var-file`)

```hcl
project_name  = "terraweek"
environment   = "prod"
instance_type = "t3.small"
vpc_cidr      = "10.1.0.0/16"
subnet_cidr   = "10.1.1.0/24"
```

### Variable Precedence (lowest → highest)

| Priority | Source | Example |
|----------|--------|---------|
| 1 (lowest) | Variable `default` in `variables.tf` | `default = "dev"` |
| 2 | `terraform.tfvars` (auto-loaded) | `environment = "dev"` |
| 3 | `*.auto.tfvars` files (auto-loaded) | `custom.auto.tfvars` |
| 4 | `-var-file` flag | `terraform plan -var-file="prod.tfvars"` |
| 5 | `-var` flag | `terraform plan -var="instance_type=t2.nano"` |
| 6 (highest) | `TF_VAR_*` environment variables | `export TF_VAR_environment="staging"` |

**Practical takeaway:** CLI flags always win. `TF_VAR_*` env vars beat `.tfvars` files but lose to `-var`. A value in `terraform.tfvars` overrides the `default` in `variables.tf`.

```bash
# Dev apply — uses terraform.tfvars automatically
terraform plan

# Prod apply — explicitly load prod.tfvars
terraform plan -var-file="prod.tfvars"

# Override a single value inline
terraform plan -var="instance_type=t2.nano"

# Set via environment variable (overrides default, not tfvars)
export TF_VAR_environment="staging"
terraform plan
```

---

## Task 3: Outputs

### `outputs.tf`

```hcl
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "subnet_id" {
  description = "ID of the public subnet"
  value       = aws_subnet.public.id
}

output "instance_id" {
  description = "EC2 instance ID"
  value       = aws_instance.web.id
}

output "instance_public_ip" {
  description = "Public IP address of the EC2 instance"
  value       = aws_instance.web.public_ip
}

output "instance_public_dns" {
  description = "Public DNS name of the EC2 instance"
  value       = aws_instance.web.public_dns
}

output "security_group_id" {
  description = "ID of the security group"
  value       = aws_security_group.web_sg.id
}
```

### Output after `terraform apply`

```
Apply complete! Resources: 5 added, 0 changed, 0 destroyed.

Outputs:

instance_id        = "i-0abc123def456789"
instance_public_dns = "ec2-13-54-xx-xx.ap-southeast-2.compute.amazonaws.com"
instance_public_ip  = "13.54.xx.xx"
security_group_id  = "sg-0abc123def456789"
subnet_id          = "subnet-0abc123def456789"
vpc_id             = "vpc-0abc123def456789"
```

```bash
terraform output                         # all outputs
terraform output instance_public_ip      # single output
terraform output -json                   # JSON for scripting
```

---

## Task 4: Data Sources

### What is the difference between a `resource` and a `data` source?

| | `resource` | `data` |
|---|---|---|
| **Action** | Creates, updates, deletes infrastructure | Reads existing infrastructure (read-only) |
| **Manages state** | Yes — tracked in `terraform.tfstate` | No — just fetches, nothing is owned |
| **Example** | `resource "aws_vpc" "main"` | `data "aws_ami" "amazon_linux"` |
| **Use case** | Build new things | Reference existing things (AMIs, AZs, VPCs) |

A `resource` is declarative infrastructure you own. A `data` source is a lookup — fetch the latest Amazon Linux AMI, the available AZs, an existing VPC you didn't create. Data sources make configs portable because values are resolved at plan time rather than hardcoded.

### Dynamic AMI lookup

```hcl
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  filter {
    name   = "root-device-type"
    values = ["ebs"]
  }
}

data "aws_availability_zones" "available" {
  state = "available"
}
```

Then in `aws_instance`:

```hcl
resource "aws_instance" "web" {
  ami               = data.aws_ami.amazon_linux.id          # dynamic
  availability_zone = data.aws_availability_zones.available.names[0]
  instance_type     = var.instance_type
  ...
}
```

Now the config works in any region — no more hardcoded AMI IDs.

---

## Task 5: Locals for Dynamic Values

```hcl
locals {
  name_prefix = "${var.project_name}-${var.environment}"
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}
```

Resources use `merge()` to combine common tags with resource-specific ones:

```hcl
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-vpc"
  })
}

resource "aws_instance" "web" {
  ...
  tags = merge(local.common_tags, var.extra_tags, {
    Name = "${local.name_prefix}-server"
  })
}
```

Every resource gets consistent `Project`, `Environment`, and `ManagedBy` tags automatically. No copy-paste, no drift.

---

## Task 6: Built-in Functions and Conditional Expressions

Tested in `terraform console`:

```bash
terraform console
```

### Five Most Useful Functions

**1. `merge(map1, map2, ...)` — combine maps**
```hcl
merge({env = "dev"}, {owner = "akash"})
# → {env = "dev", owner = "akash"}
```
Essential for tag composition. Later maps win on duplicate keys.

**2. `lookup(map, key, default)` — safe map access**
```hcl
lookup({dev = "t2.micro", prod = "t3.small"}, "prod", "t2.micro")
# → "t3.small"
```
Use this instead of `map[key]` when the key might not exist — returns the default instead of erroring.

**3. `cidrsubnet(prefix, newbits, netnum)` — subnet math**
```hcl
cidrsubnet("10.0.0.0/16", 8, 1)
# → "10.0.1.0/24"
```
Generate subnet CIDRs dynamically from a VPC CIDR. No more manual subnet calculations.

**4. `join(separator, list)` — list to string**
```hcl
join("-", ["terra", "week", "2026"])
# → "terra-week-2026"
```
Useful for building resource names, ARNs, or any string built from a list.

**5. `toset(list)` — deduplicate a list**
```hcl
toset(["us-east-1a", "us-east-1b", "us-east-1a"])
# → {"us-east-1a", "us-east-1b"}
```
Use before `for_each` when your input list might have duplicates — `for_each` requires a set or map, not a list.

### Conditional Expression

```hcl
instance_type = var.environment == "prod" ? "t3.small" : "t2.micro"
```

Syntax: `condition ? value_if_true : value_if_false`

With `environment = "prod"` the plan shows `t3.small`. With `environment = "dev"` it shows `t2.micro`. One line replaces a whole variable override.

---

## Variable vs Local vs Output vs Data — Quick Reference

| Concept | Direction | Purpose |
|---------|-----------|---------|
| `variable` | **Input** — comes from outside | Accept values from the user, CI, or `.tfvars` |
| `local` | **Internal** — computed inside the module | Derived values, reused expressions, tag maps |
| `output` | **Output** — sent outside | Expose resource attributes to the user or parent module |
| `data` | **Lookup** — reads from AWS | Fetch existing infrastructure without owning it |

---

## Key Takeaways

- Variables with no `default` force the caller to provide a value — great for project name, environment
- `terraform.tfvars` loads automatically; anything else needs `-var-file`
- Data sources make configs portable — AMI IDs and AZ names resolve at plan time
- `locals` + `merge()` = consistent tagging across every resource with zero repetition
- `terraform console` is the fastest way to test functions before committing them to config

---

*Part of the #90DaysOfDevOps challenge — Day 63*
