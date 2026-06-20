# Day 66 — Provision an EKS Cluster with Terraform Modules

## Overview

Today I provisioned a production-grade AWS EKS cluster entirely through Terraform registry modules — VPC, subnets, NAT gateway, IAM roles, KMS encryption, managed node groups, and a live Nginx workload. Everything created with code, everything destroyed with one command.

**Total resources created: 57**

---

## File Structure

```
terraform-eks/
  providers.tf        # AWS + Kubernetes provider config
  vpc.tf              # VPC module (subnets, NAT gateway, routing)
  eks.tf              # EKS module (cluster, node group, IAM, KMS)
  variables.tf        # Input variables
  outputs.tf          # Cluster endpoint, name, region
  terraform.tfvars    # Variable values
  k8s/
    nginx-deployment.yaml   # Nginx Deployment + LoadBalancer Service
```

---

## Key Configuration Files

### providers.tf
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
  }
}

provider "aws" {
  region = var.region
}
```

### vpc.tf
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "my-eks-vpc"
  cidr = var.vpc_cidr

  azs             = ["ap-south-1a", "ap-south-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway   = true
  single_nat_gateway   = true
  enable_dns_hostnames = true

  public_subnet_tags = {
    "kubernetes.io/role/elb" = 1
  }

  private_subnet_tags = {
    "kubernetes.io/role/internal-elb" = 1
  }
}
```

### eks.tf
```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = var.cluster_name
  cluster_version = var.cluster_version

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  cluster_endpoint_public_access           = true
  enable_cluster_creator_admin_permissions = true

  eks_managed_node_groups = {
    terraweek_nodes = {
      ami_type       = "AL2_x86_64"
      instance_types = [var.node_instance_type]

      min_size     = 1
      max_size     = 3
      desired_size = var.node_desired_count
    }
  }

  tags = {
    Environment = "dev"
    Project     = "TerraWeek"
    ManagedBy   = "Terraform"
  }
}
```

---

## Task 2: VPC Design — Why Public + Private Subnets?

EKS requires both subnet types for different purposes:

| Subnet | Purpose |
|---|---|
| **Private** | Worker nodes run here — not directly internet-exposed |
| **Public** | NAT Gateway and AWS Load Balancers live here |

The **NAT Gateway** in the public subnet allows worker nodes in private subnets to pull container images and reach AWS APIs without being publicly accessible themselves.

**Subnet tags** tell the AWS Load Balancer Controller which subnets to use:
- `kubernetes.io/role/elb = 1` → public-facing LoadBalancer Services use these
- `kubernetes.io/role/internal-elb = 1` → internal LoadBalancers use these

Without these tags, EKS cannot automatically provision load balancers for Kubernetes Services.

---

## Task 3 & 4: Apply and Connect

### terraform apply output
```
Apply complete! Resources: 57 added, 0 changed, 0 destroyed.

Outputs:
cluster_endpoint = "https://D21E3E6AA34DFF104F616AB70C2122B5.gr7.ap-south-1.eks.amazonaws.com"
cluster_name     = "my-eks-cluster"
cluster_region   = "ap-south-1"
```

### kubeconfig update
```bash
aws eks update-kubeconfig --name my-eks-cluster --region ap-south-1
```

### kubectl get nodes
```
NAME                                        STATUS   ROLES    AGE     VERSION
ip-10-0-3-205.ap-south-1.compute.internal   Ready    <none>   3m58s   v1.31.13-eks-ecaa3a6
ip-10-0-4-128.ap-south-1.compute.internal   Ready    <none>   3m57s   v1.31.13-eks-ecaa3a6
```

Both nodes **Ready** ✅

### kubectl get pods -A
```
NAMESPACE     NAME                       READY   STATUS    RESTARTS   AGE
kube-system   aws-node-69b2d             2/2     Running   0          4m9s
kube-system   aws-node-rc62m             2/2     Running   0          4m8s
kube-system   coredns-6d86b49686-bcvf5   1/1     Running   0          7m30s
kube-system   coredns-6d86b49686-x6dxm   1/1     Running   0          7m30s
kube-system   kube-proxy-9fqdb           1/1     Running   0          4m9s
kube-system   kube-proxy-q4d9g           1/1     Running   0          4m8s
```

All kube-system pods running ✅

---

## Task 5: Nginx Workload on EKS

### Deployment + Service
```bash
kubectl apply -f k8s/nginx-deployment.yaml
```

### Verification
```
NAME                               READY   STATUS    RESTARTS   AGE
nginx-terraweek-54b9c68f67-59w27   1/1     Running   0          80s
nginx-terraweek-54b9c68f67-rdt6d   1/1     Running   0          80s
nginx-terraweek-54b9c68f67-sxbc6   1/1     Running   0          80s

NAME              READY   UP-TO-DATE   AVAILABLE   AGE
nginx-terraweek   3/3     3            3           83s

NAME            TYPE           CLUSTER-IP      EXTERNAL-IP                                                                PORT(S)
nginx-service   LoadBalancer   172.20.179.76   a479fa2092660401890e34dd385a70ec-1228099967.ap-south-1.elb.amazonaws.com   80:31048/TCP
```

**Nginx accessible at:** `http://a479fa2092660401890e34dd385a70ec-1228099967.ap-south-1.elb.amazonaws.com`

✅ Nginx welcome page confirmed via browser and AWS ELB DNS

---

## Task 6: Destroy

```bash
# Remove k8s resources first (frees the ELB)
kubectl delete -f k8s/nginx-deployment.yaml

# Destroy all Terraform infrastructure
terraform destroy
```

Post-destroy verification checklist:
- EKS clusters: empty ✅
- EC2 instances: terminated ✅
- VPC: deleted ✅
- NAT Gateways: deleted ✅
- Elastic IPs: released ✅

---

## Issues Encountered & Fixes

During this exercise, multiple failed apply attempts left orphaned resources in AWS that Terraform couldn't track. Here's the troubleshooting pattern used:

| Error | Cause | Fix |
|---|---|---|
| `CloudWatch log group already exists` | Persists after manual cluster delete | `terraform import` or `aws logs delete-log-group` |
| `KMS alias already exists` | Persists after manual cluster delete | `aws kms delete-alias` then re-apply |
| `NAT Gateway EIP already associated` | Failed NAT GW not cleaned up | `aws ec2 delete-nat-gateway` on all failed GWs |
| `EKS cluster already exists` | Partial state mismatch | `terraform state rm` + `terraform import` |
| `kubectl: server asked for credentials` | EKS module v20 doesn't grant creator access by default | Add `enable_cluster_creator_admin_permissions = true` |

**Key lesson:** Always delete orphaned AWS resources AND wipe `terraform.tfstate` together. Doing one without the other causes a mismatch loop.

---

## Reflection: Terraform EKS vs Manual kind/minikube (Day 50)

| Aspect | kind/minikube (Day 50) | Terraform EKS (Day 66) |
|---|---|---|
| **Setup time** | ~5 minutes | ~20 minutes |
| **Resources created** | 1 local cluster | 57 AWS resources |
| **Repeatability** | Manual steps | `terraform apply` |
| **Production-ready** | No | Yes |
| **Cost** | Free | ~$0.10/hr (NAT + nodes) |
| **Destroy** | `kind delete cluster` | `terraform destroy` |
| **Networking** | Local only | VPC, subnets, IGW, NAT, SGs |
| **IAM** | None | Full role + policy chain |

kind/minikube is perfect for local development and learning Kubernetes concepts. Terraform EKS is what you actually use in production — every resource is auditable, version-controlled, and reproducible across environments.

The biggest mindset shift: with Terraform, **infrastructure is just code in a Git repo**. Any team member can clone it, run `terraform apply`, and get an identical cluster in a different AWS account in under 20 minutes.
