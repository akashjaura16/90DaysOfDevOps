# Day 62 -- Providers, Resources and Dependencies

## Overview

Today I built a complete AWS networking stack using Terraform — VPC, subnet, internet gateway, route table, security group, and an EC2 instance. The key learning was understanding how Terraform resolves **dependency order** automatically through implicit dependencies (interpolation), and when to use **explicit dependencies** with `depends_on`.

---

## Task 1: AWS Provider

### `providers.tf`

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
  region = "ap-southeast-2"
}
```

### Version Constraint Explained

| Constraint | Meaning |
|---|---|
| `~> 5.0` | Allows `5.x` (any patch/minor), but **not** `6.0` |
| `>= 5.0` | Allows anything `5.0` and above, including `6.0`, `7.0` |
| `= 5.0.0` | Pinned to **exactly** `5.0.0` — no updates at all |

`~> 5.0` is the sweet spot for real projects — you get bug fixes and minor features automatically, but no breaking major version changes.

### `.terraform.lock.hcl`

The lock file records the exact provider version and checksums installed during `terraform init`. It ensures every team member and every CI run uses the **identical** provider binary — no surprise behaviour from version drift. Always commit this file to source control.

---

## Task 2: VPC from Scratch

### `main.tf` (Full File with Comments)

```hcl
# Automatically fetch the latest Amazon Linux 2 AMI for the region
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# -------------------------------------------------------------------
# NETWORKING
# -------------------------------------------------------------------

# VPC — the umbrella that all other resources live inside
# CIDR 10.0.0.0/16 gives us 65,536 IPs to work with
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "TerraWeek-VPC"
  }
}

# Subnet — a subdivision of the VPC's address space
# 10.0.1.0/24 gives us 256 IPs
# map_public_ip_on_launch = true so EC2 instances get a public IP automatically
resource "aws_subnet" "main" {
  vpc_id                  = aws_vpc.main.id        # implicit dependency on VPC
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = true

  tags = {
    Name = "TerraWeek-Public-Subnet"
  }
}

# Internet Gateway — the door between the VPC and the public internet
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id                         # implicit dependency on VPC

  tags = {
    Name = "TerraWeek-IGW"
  }
}

# Route Table — rules for how traffic is routed inside the VPC
# 0.0.0.0/0 means "all internet traffic goes through the IGW"
resource "aws_route_table" "main" {
  vpc_id = aws_vpc.main.id                         # implicit dependency on VPC

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id      # implicit dependency on IGW
  }

  tags = {
    Name = "TerraWeek-Route-Table"
  }
}

# Associate the route table with the subnet so the subnet uses these routes
resource "aws_route_table_association" "main" {
  subnet_id      = aws_subnet.main.id              # implicit dependency on subnet
  route_table_id = aws_route_table.main.id         # implicit dependency on route table
}

# -------------------------------------------------------------------
# SECURITY
# -------------------------------------------------------------------

# Security Group — acts as a virtual firewall for the EC2 instance
resource "aws_security_group" "main" {
  name   = "TerraWeek-SG"
  vpc_id = aws_vpc.main.id                         # implicit dependency on VPC

  # Allow inbound SSH from anywhere
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # Allow inbound HTTP from anywhere
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # Allow all outbound traffic
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"           # -1 = all protocols
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "TerraWeek-SG"
  }
}

# -------------------------------------------------------------------
# COMPUTE
# -------------------------------------------------------------------

# EC2 instance placed in the public subnet
# Uses data source to get the latest Amazon Linux 2 AMI automatically
resource "aws_instance" "main" {
  ami                         = data.aws_ami.amazon_linux_2.id
  instance_type               = "t2.micro"
  subnet_id                   = aws_subnet.main.id              # implicit dependency
  vpc_security_group_ids      = [aws_security_group.main.id]    # implicit dependency
  associate_public_ip_address = true

  tags = {
    Name = "TerraWeek-Server"
  }

  # create_before_destroy: spin up the new instance before tearing down the old one
  # Prevents downtime when the AMI or instance config changes
  lifecycle {
    create_before_destroy = true
  }
}

# -------------------------------------------------------------------
# STORAGE
# -------------------------------------------------------------------

# S3 bucket for application logs
# No direct reference to the EC2 instance, but we want it created after
# depends_on makes the dependency explicit
resource "aws_s3_bucket" "app_logs" {
  bucket = "terraweek-app-logs-akash-2026"

  depends_on = [aws_instance.main]   # explicit dependency

  tags = {
    Name = "TerraWeek-App-Logs"
  }
}
```

---

## Task 3: Implicit Dependencies

### How does Terraform know to create the VPC before the subnet?

Because of interpolation in the subnet resource:

```hcl
vpc_id = aws_vpc.main.id
```

When Terraform sees `aws_vpc.main.id` referenced inside another resource, it knows it needs that resource's output first. It builds an internal **dependency graph** and resolves creation order automatically — no manual ordering needed.

### What would happen if the subnet was created before the VPC?

AWS would reject the API call with a **"VPC not found"** error — a subnet cannot physically exist without a VPC to subdivide. Terraform prevents this entirely because interpolation creates an automatic dependency. The only way to hit this problem is if you hardcode the VPC ID instead of referencing it:

```hcl
vpc_id = "vpc-0abc1234"  # hardcoded — no implicit dependency, dangerous!
```

### All Implicit Dependencies

| Resource | Depends On | Via |
|---|---|---|
| `aws_subnet.main` | `aws_vpc.main` | `vpc_id = aws_vpc.main.id` |
| `aws_internet_gateway.main` | `aws_vpc.main` | `vpc_id = aws_vpc.main.id` |
| `aws_route_table.main` | `aws_vpc.main` | `vpc_id = aws_vpc.main.id` |
| `aws_route_table.main` | `aws_internet_gateway.main` | `gateway_id = aws_internet_gateway.main.id` |
| `aws_route_table_association.main` | `aws_subnet.main` | `subnet_id = aws_subnet.main.id` |
| `aws_route_table_association.main` | `aws_route_table.main` | `route_table_id = aws_route_table.main.id` |
| `aws_security_group.main` | `aws_vpc.main` | `vpc_id = aws_vpc.main.id` |
| `aws_instance.main` | `aws_subnet.main` | `subnet_id = aws_subnet.main.id` |
| `aws_instance.main` | `aws_security_group.main` | `vpc_security_group_ids` |
| `aws_instance.main` | `data.aws_ami.amazon_linux_2` | `ami = data.aws_ami.amazon_linux_2.id` |

### About Interpolation

The `aws_vpc.main.id` syntax is called **interpolation** — Terraform replaces it with the actual AWS-assigned value at runtime (e.g. `vpc-0355a6732c78149d9`). It does two things at once: passes the real value AND signals a dependency.

```
resource_type . resource_name . attribute
   aws_vpc   .     main       .    id
```

---

## Task 5: Explicit Dependencies

### When to use `depends_on` in real projects

**Example 1 — IAM policy lag:**
An IAM role policy attachment has no direct reference in a Lambda function, but the function will fail at deploy time if the policy isn't fully propagated yet. `depends_on` forces Lambda to wait.

```hcl
resource "aws_lambda_function" "main" {
  depends_on = [aws_iam_role_policy_attachment.main]
}
```

**Example 2 — S3 bucket policy before application deploy:**
An Elastic Beanstalk environment needs read access to an S3 bucket, but there's no direct Terraform reference between the policy resource and the environment resource. Without `depends_on`, Terraform might try to launch the app before the bucket policy is in place.

```hcl
resource "aws_elastic_beanstalk_environment" "main" {
  depends_on = [aws_s3_bucket_policy.main]
}
```

### Dependency Tree

```
data.aws_ami.amazon_linux_2
aws_vpc.main
  ├── aws_subnet.main
  │     └── aws_instance.main ──────────────────┐
  ├── aws_internet_gateway.main                  │
  │     └── aws_route_table.main                 │
  │           └── aws_route_table_association    │
  │                 (also depends on subnet)     │
  ├── aws_route_table.main                       │
  └── aws_security_group.main                   │
        └── aws_instance.main                   │
                                                 ▼
                                    aws_s3_bucket.app_logs
                                    (depends_on instance)
```

---

## Task 6: Lifecycle Rules

### The Three Lifecycle Arguments

**1. `create_before_destroy`**

```hcl
lifecycle {
  create_before_destroy = true
}
```

Default Terraform behaviour: destroy old → create new. This reverses it to: create new → destroy old. Use when you can't afford downtime — EC2 instances, load balancers, RDS instances. Without it, changing an AMI ID causes a brief outage while Terraform tears down the old instance before the new one is ready.

**2. `prevent_destroy`**

```hcl
lifecycle {
  prevent_destroy = true
}
```

Terraform will error and refuse to destroy this resource even during `terraform destroy`. Use on critical resources you never want accidentally deleted — production RDS databases, S3 buckets with compliance data, or core VPCs.

**3. `ignore_changes`**

```hcl
lifecycle {
  ignore_changes = [ami, tags]
}
```

Tells Terraform to ignore drift on specific attributes. Use when something outside Terraform is legitimately modifying a resource — for example, an Auto Scaling Group adjusting instance count, or someone adding a cost-allocation tag manually in the console. Without this, Terraform would detect the drift and try to revert it on every `terraform plan`.

### Destroy Order (Reverse Dependencies)

Terraform destroys in the **exact reverse** of creation order:

```
# Creation order
aws_vpc → aws_subnet, aws_igw, aws_sg → aws_route_table → aws_rt_assoc → aws_instance → aws_s3_bucket

# Destroy order (reversed)
aws_s3_bucket → aws_instance → aws_route_table_association → aws_route_table
→ aws_subnet → aws_security_group → aws_internet_gateway → aws_vpc
```

The VPC is always destroyed last — you can't delete the umbrella while resources still live inside it.

---

## Implicit vs Explicit Dependencies — In My Own Words

**Implicit dependency** is when Terraform figures out the order automatically because you used interpolation (`aws_vpc.main.id`) to pass a value from one resource into another. Terraform sees the reference, knows it needs that resource's output, and builds the right order without you saying anything extra.

**Explicit dependency** via `depends_on` is for situations where there's no direct value being passed between resources, but one still needs to exist before the other. The classic case is IAM permissions — a Lambda function doesn't reference a policy attachment resource directly, but it will fail if the policy isn't ready. `depends_on` tells Terraform: "even though there's no reference here, don't create this until that other thing is done."

Rule of thumb: use interpolation wherever you can (it's cleaner and self-documenting), and reach for `depends_on` only when there's a real dependency that Terraform can't see on its own.

---

## Resources Applied

```
aws_vpc.main                        created
aws_subnet.main                     created
aws_internet_gateway.main           created
aws_route_table.main                created
aws_route_table_association.main    created
aws_security_group.main             created
aws_instance.main                   created
aws_s3_bucket.app_logs              created
```

**Apply complete! Resources: 8 added, 0 changed, 0 destroyed.**
