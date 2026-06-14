# Day 64 — Terraform State Management and Remote Backends

## Overview

The state file is the single most important thing in Terraform. It is the map between your `.tf` files and what actually exists in AWS. Today I learned to manage state professionally — remote backends, locking, importing existing resources, state surgery, and drift detection.

---

## Task 1: Inspecting State

```bash
terraform show                          # Full state in human-readable format
terraform state list                    # All resources tracked by Terraform
terraform state show aws_instance.my_ec2
```

### What I found

- Terraform tracked **2 resources**: `aws_instance.my_ec2` and `aws_s3_bucket.logs_bucket`
- The state stores way more than what I defined — public IP, private IP, VPC ID, subnet ID, AMI details, security groups, root device, availability zone, and 38+ more attributes
- The `serial` number in `terraform.tfstate` represents how many times the state has been modified. It started at `1` and reached `5` by the end of today — each apply, import, and state mv incremented it

---

## Task 2: S3 Remote Backend

### Why remote state?

Local state is dangerous — one deleted file and Terraform forgets everything it manages. Remote state in S3 is versioned, encrypted, and shared across the team.

### Setup commands

```bash
# Create S3 bucket
aws s3api create-bucket \
  --bucket terraweek-state-akashjaura \
  --region ap-south-1 \
  --create-bucket-configuration LocationConstraint=ap-south-1

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket terraweek-state-akashjaura \
  --versioning-configuration Status=Enabled

# Create DynamoDB table for locking
aws dynamodb create-table \
  --table-name terraweek-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region ap-south-1
```

### Backend config in `main.tf`

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket         = "terraweek-state-akashjaura"
    key            = "dev/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraweek-state-lock"
    encrypt        = true
  }
}
```

After running `terraform init`, Terraform asked:
> "Do you want to copy existing state to the new backend?"

Said yes — state migrated to S3. The local `terraform.tfstate` was gone and `dev/terraform.tfstate` appeared in the S3 bucket. Running `terraform plan` showed **No changes** — migration was clean.

---

## Task 3: State Locking

### How I tested it

Opened two terminals in the same `day-64` directory:
- **Terminal 1:** ran `terraform apply` — waiting for confirmation
- **Terminal 2:** ran `terraform plan` immediately

### Error from Terminal 2

```
Error: Error acquiring the state lock

Error message: ConditionalCheckFailedException: The conditional request failed
Lock Info:
  ID:        <lock-id>
  Path:      terraweek-state-akashjaura/dev/terraform.tfstate
  Operation: OperationTypeApply
  Who:       Akash
  Created:   2026-06-14
```

### Why locking matters

When one developer runs `terraform apply`, DynamoDB puts a lock on the state file in S3. Anyone else who tries to run `plan` or `apply` at the same time gets blocked with this error.

Without locking — two applies run simultaneously, both modify state, state gets corrupted, Terraform loses track of what exists in AWS. In a team of 10 developers all pushing to the same infrastructure, this would be a disaster.

To release a stale lock:
```bash
terraform force-unlock <LOCK_ID>
```
Only use this when you are sure no other operation is actually running.

---

## Task 4: Importing an Existing Resource

### The difference between `terraform import` and creating from scratch

| | `terraform import` | Create from scratch |
|---|---|---|
| Resource exists in AWS? | **Yes** — already there | **No** — Terraform creates it |
| What Terraform does | Brings it into state, manages going forward | Builds it from your `.tf` file |
| Risk | None — nothing is created or destroyed | New resource is provisioned |

I created `s3-bucket-terraweek` manually in the AWS console, wrote the resource block in `main.tf`, then imported it:

```bash
terraform import aws_s3_bucket.imported s3-bucket-terraweek
```

Output:
```
aws_s3_bucket.imported: Importing from ID "s3-bucket-terraweek"...
aws_s3_bucket.imported: Import prepared!
aws_s3_bucket.imported: Refreshing state... [id=s3-bucket-terraweek]
Import successful!
```

`terraform state list` then showed:
```
aws_instance.my_ec2
aws_s3_bucket.imported
```

---

## Task 5: State Surgery — mv and rm

### `state mv` — rename a resource

```bash
terraform state mv aws_s3_bucket.imported aws_s3_bucket.logs_bucket
```

Then updated `main.tf` to use `aws_s3_bucket.logs_bucket`. After that `terraform plan` showed **No changes**.

**When to use `state mv` in real projects:** When you refactor `.tf` files and rename a resource block — use `state mv` so Terraform doesn't destroy and recreate the real resource in AWS. The actual resource stays untouched, only the tracking name changes.

### `state rm` — remove from tracking

```bash
terraform state rm aws_s3_bucket.logs_bucket
```

The actual S3 bucket in AWS **stays alive** — Terraform just forgets it exists. 

**When to use `state rm`:** When you want to stop managing a resource with Terraform without deleting it from AWS — for example handing it off to another team or another Terraform config.

### Re-import to bring it back

```bash
terraform import aws_s3_bucket.logs_bucket s3-bucket-terraweek
```

---

## Task 6: Simulate and Fix State Drift

### What is drift?

Drift happens when someone changes infrastructure outside of Terraform — through the AWS console, CLI, or another tool. Terraform's state no longer matches reality.

### What I did

1. Applied config — everything in sync
2. Went to AWS console → changed EC2 Name tag from `TerraWeek` to `manually changed`
3. Ran `terraform plan`

### Drift detected

```
~ aws_instance.my_ec2 will be updated in-place
  ~ tags = {
      ~ "Name" = "manually changed" -> "TerraWeek"
    }

Plan: 0 to add, 1 to change, 0 to destroy.
```

Terraform caught the manual change and wanted to revert it. That's Terraform being the source of truth.

### Fix — Option A (reconcile)

```bash
terraform apply
```

Terraform reverted the tag back to `TerraWeek`. Running `terraform plan` again showed **No changes**. Drift resolved.

### How teams prevent drift in production

- **Restrict console access** — developers cannot touch AWS console or CLI directly in production. All changes go through Terraform only
- **CI/CD pipeline** — every change goes through Git → PR review → pipeline runs `terraform apply`. No manual changes allowed
- **IAM permissions** — lock down who can make changes directly in AWS so only the CI/CD service account has write access
- **Regular drift detection** — run `terraform plan` on a schedule in CI to catch any drift early

---

## State Commands Reference

| Command | What it does |
|---------|-------------|
| `terraform state list` | List all resources tracked in state |
| `terraform state show <resource>` | Show all attributes of a resource |
| `terraform state mv <old> <new>` | Rename a resource in state |
| `terraform state rm <resource>` | Remove resource from state (keeps it in AWS) |
| `terraform import <resource> <id>` | Import existing AWS resource into state |
| `terraform force-unlock <id>` | Release a stale state lock |
| `terraform apply -refresh-only` | Update state to match real infrastructure |

---

## Local vs Remote State

```
LOCAL STATE                          REMOTE STATE (S3 + DynamoDB)
─────────────────                    ────────────────────────────────
terraform.tfstate                    S3: terraweek-state-akashjaura
  on your machine                       dev/terraform.tfstate
  
  ✗ Lost if machine dies             ✓ Versioned, recoverable
  ✗ Not shared with team             ✓ Shared across all team members
  ✗ No locking                       ✓ DynamoDB locking prevents corruption
  ✗ Not encrypted                    ✓ Encrypted at rest
```

---


