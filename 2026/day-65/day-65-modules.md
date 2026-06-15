# Day 65 – Terraform Modules: Build Reusable Infrastructure

## What I Did Today

Today was a big one. I finally stopped writing everything in one massive `main.tf` file and learned how to properly structure Terraform code using modules. The idea clicked for me when I thought of modules like functions in programming — you write the logic once, then call it as many times as you need with different inputs.

---

## My Module Structure

Here's the directory layout I built from scratch:

```
terraform-modules/
├── main.tf              # Root module — wires everything together
├── variables.tf         # Input variables for the root module
├── outputs.tf           # Exposes outputs from child modules
├── providers.tf         # AWS provider configuration
├── terraform.tfvars     # Actual values for variables
└── modules/
    ├── ec2-instance/
    │   ├── main.tf      # Defines the aws_instance resource
    │   ├── variables.tf # Inputs: ami, instance type, subnet, etc.
    │   └── outputs.tf   # Outputs: instance ID, public IP, private IP
    └── security-group/
        ├── main.tf      # Defines the aws_security_group resource
        ├── variables.tf # Inputs: vpc_id, sg_name, ingress_ports
        └── outputs.tf   # Output: sg_id
```

**Root module vs child module:** The root module is the entry point — it's what you run `terraform apply` from. Child modules are the reusable building blocks the root calls. The root passes in values; the child does the actual resource creation.

---

## Task 2 – EC2 Module

### `modules/ec2-instance/variables.tf`
```hcl
variable "ami_id" {
  type = string
}
variable "instance_type" {
  type    = string
  default = "t2.micro"
}
variable "subnet_id" {
  type = string
}
variable "security_group_ids" {
  type = list(string)
}
variable "instance_name" {
  type = string
}
variable "tags" {
  type    = map(string)
  default = {}
}
```

### `modules/ec2-instance/main.tf`
```hcl
resource "aws_instance" "my_instance" {
  ami                    = var.ami_id
  instance_type          = var.instance_type
  subnet_id              = var.subnet_id
  vpc_security_group_ids = var.security_group_ids

  tags = merge(var.tags, {
    Name = var.instance_name
  })
}
```

### `modules/ec2-instance/outputs.tf`
```hcl
output "instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.my_instance.id
}
output "instance_public_ip" {
  description = "Public IP address of the EC2 instance"
  value       = aws_instance.my_instance.public_ip
}
output "instance_private_ip" {
  description = "Private IP address of the EC2 instance"
  value       = aws_instance.my_instance.private_ip
}
```

---

## Task 3 – Security Group Module

The interesting part here was the `dynamic` block. Instead of hardcoding one ingress rule at a time, it loops over a list of port numbers and generates a rule for each one automatically.

### `modules/security-group/main.tf`
```hcl
resource "aws_security_group" "this" {
  name   = var.sg_name
  vpc_id = var.vpc_id
  tags   = var.tags

  dynamic "ingress" {
    for_each = var.ingress_ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---

## Task 4 – Calling Modules from Root

This is where everything came together. The same EC2 module was called twice with different names — `web_server` and `api_server` — deploying two completely separate instances from the exact same module code.

### Root `main.tf` (Tasks 4–5 combined)
```hcl
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

locals {
  common_tags = merge(var.tags, {
    Project = var.project_name
  })
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name            = "terraweek-vpc"
  cidr            = "10.0.0.0/16"
  azs             = ["us-west-2a", "us-west-2b"]
  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnets = ["10.0.3.0/24", "10.0.4.0/24"]

  enable_nat_gateway   = false
  enable_dns_hostnames = true
  tags                 = local.common_tags
}

module "web_sg" {
  source        = "./modules/security-group"
  vpc_id        = module.vpc.vpc_id
  sg_name       = "terraweek-web-sg"
  ingress_ports = [22, 80, 443]
  tags          = local.common_tags
}

module "web_server" {
  source             = "./modules/ec2-instance"
  ami_id             = data.aws_ami.amazon_linux.id
  instance_type      = "t2.micro"
  subnet_id          = module.vpc.public_subnets[0]
  security_group_ids = [module.web_sg.sg_id]
  instance_name      = "terraweek-web"
  tags               = local.common_tags
}

module "api_server" {
  source             = "./modules/ec2-instance"
  ami_id             = data.aws_ami.amazon_linux.id
  instance_type      = "t2.micro"
  subnet_id          = module.vpc.public_subnets[1]
  security_group_ids = [module.web_sg.sg_id]
  instance_name      = "terraweek-api"
  tags               = local.common_tags
}

output "web_server_ip" {
  value = module.web_server.instance_public_ip
}
output "api_server_ip" {
  value = module.api_server.instance_public_ip
}
output "vpc_id" {
  value = module.vpc.vpc_id
}
output "public_subnets" {
  value = module.vpc.public_subnets
}
```

---

## Task 5 – Public Registry Module

Instead of manually writing 50+ lines of VPC resources (vpc, subnets, route tables, IGW, route associations), I replaced all of it with one registry module call. The `terraform-aws-modules/vpc/aws` module handles all of that internally.

**Where does Terraform download registry modules?**
They go into `.terraform/modules/` inside your project directory. After `terraform init`, you can see the full source code of the downloaded module there — useful for understanding what it actually creates.

**Resources created by the registry VPC module vs hand-written:**

| Hand-written (Day 62 style) | Registry module |
|---|---|
| `aws_vpc` | `aws_vpc` + default SG + default NACL + default route table |
| `aws_subnet` (1 subnet) | 4 subnets (2 public + 2 private) |
| Manual route table | Public + private route tables with associations |
| Manual IGW | Internet gateway + public route automatically |

The registry module created roughly 14 resources vs about 3–4 hand-written. Much more production-ready out of the box.

---

## Task 6 – Versioning and State

### Version pinning options
```hcl
version = "5.1.0"          # Exact pin — no surprises, best for production
version = "~> 5.0"         # Any 5.x — allows patch/minor upgrades
version = ">= 5.0, < 6.0"  # Same result, more explicit about the ceiling
```

I'm using `~> 5.0` for this project. In a real production environment I'd pin to an exact version.

### terraform state list output
```
module.api_server.aws_instance.my_instance
module.vpc.aws_default_network_acl.this[0]
module.vpc.aws_default_route_table.default[0]
module.vpc.aws_default_security_group.this[0]
module.vpc.aws_internet_gateway.this[0]
module.vpc.aws_route.public_internet_gateway[0]
module.vpc.aws_route_table.private[0]
module.vpc.aws_route_table.private[1]
module.vpc.aws_route_table.public[0]
module.vpc.aws_subnet.private[0]
module.vpc.aws_subnet.private[1]
module.vpc.aws_subnet.public[0]
module.vpc.aws_subnet.public[1]
module.vpc.aws_vpc.this[0]
module.web_server.aws_instance.my_instance
module.web_sg.aws_security_group.this
```

The `module.<name>.` prefix in state makes it immediately obvious which module owns each resource, which makes debugging and targeted destroys (`terraform destroy -target=module.web_sg`) much cleaner.

---

## Five Module Best Practices (In My Own Words)

1. **Always pin versions for registry modules** — `~> 5.0` is fine for learning, but in production you want `5.1.0` so a maintainer's update can't break your infrastructure without you knowing.

2. **One concern per module** — my EC2 module only knows about EC2. It doesn't touch security groups or VPCs. Keeping scope tight means you can reuse it anywhere without dragging in unrelated resources.

3. **Use variables for everything, hardcode nothing** — if an AMI ID, region, or CIDR block is hardcoded inside a module, you can't reuse it somewhere else. Every value that might change goes in `variables.tf`.

4. **Always define outputs** — a module that doesn't expose outputs is a black box. The caller needs IDs and IPs to wire things together. If I hadn't added `sg_id` to the security group module, the EC2 module couldn't reference it.

5. **Add a README.md to every custom module** — future you (and teammates) will thank you. Document what inputs are required, what outputs are produced, and give a quick usage example. Treat your module like a mini open source project.

---

