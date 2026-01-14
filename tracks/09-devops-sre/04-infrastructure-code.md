# 04. Infrastructure as Code

[← Назад к списку тем](README.md)

---

## IaC Concepts

```
┌─────────────────────────────────────────────────────────────────────┐
│                  Infrastructure as Code                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Benefits:                                                          │
│  - Version controlled                                               │
│  - Reproducible environments                                        │
│  - Self-documenting                                                 │
│  - Collaboration via PRs                                            │
│  - Audit trail                                                      │
│  - Testable                                                         │
│                                                                     │
│  Types:                                                             │
│  ─────────                                                          │
│  Declarative: Define end state (Terraform, CloudFormation)          │
│  Imperative: Define steps (scripts, Ansible)                        │
│                                                                     │
│  Immutable: Replace, don't modify (containers, AMIs)                │
│  Mutable: Modify in place (config management)                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Terraform Basics

### Configuration Structure

```hcl
# providers.tf
terraform {
  required_version = ">= 1.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}

provider "aws" {
  region = var.aws_region
}
```

```hcl
# variables.tf
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
}

variable "tags" {
  description = "Common tags"
  type        = map(string)
  default     = {}
}
```

```hcl
# main.tf
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true

  tags = merge(var.tags, {
    Name = "${var.environment}-vpc"
  })
}

resource "aws_subnet" "public" {
  count = 2

  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.${count.index}.0/24"
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = merge(var.tags, {
    Name = "${var.environment}-public-${count.index}"
  })
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type
  subnet_id     = aws_subnet.public[0].id

  tags = merge(var.tags, {
    Name = "${var.environment}-app"
  })
}
```

```hcl
# outputs.tf
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

output "instance_public_ip" {
  description = "Public IP of the instance"
  value       = aws_instance.app.public_ip
}
```

---

## Terraform Commands

```bash
# Initialize
terraform init                    # Download providers, init backend

# Plan
terraform plan                    # Show what will change
terraform plan -out=tfplan        # Save plan to file

# Apply
terraform apply                   # Apply changes (with confirmation)
terraform apply tfplan            # Apply saved plan
terraform apply -auto-approve     # Skip confirmation (CI/CD)

# Destroy
terraform destroy                 # Destroy all resources

# State management
terraform state list              # List resources in state
terraform state show aws_vpc.main # Show resource details
terraform state mv                # Move/rename resource
terraform state rm                # Remove from state (not destroy)

# Import existing resources
terraform import aws_vpc.main vpc-123456

# Format and validate
terraform fmt                     # Format files
terraform validate                # Validate configuration

# Workspaces (multiple environments)
terraform workspace list
terraform workspace new staging
terraform workspace select staging
```

---

## State Management

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Terraform State                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  State file contains:                                               │
│  - Resource IDs                                                     │
│  - Resource attributes                                              │
│  - Dependencies                                                     │
│  - Metadata                                                         │
│                                                                     │
│  Remote State Backend (recommended):                                │
│  - S3 + DynamoDB (AWS)                                              │
│  - GCS (GCP)                                                        │
│  - Azure Storage                                                    │
│  - Terraform Cloud                                                  │
│                                                                     │
│  Benefits:                                                          │
│  - Team collaboration                                               │
│  - State locking (prevent concurrent changes)                       │
│  - State versioning                                                 │
│  - Encrypted at rest                                                │
│                                                                     │
│  Security:                                                          │
│  - State contains sensitive data                                    │
│  - Enable encryption                                                │
│  - Limit access to state bucket                                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### State Locking

```hcl
# S3 backend with DynamoDB locking
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"  # For locking
    encrypt        = true
  }
}
```

---

## Modules

```hcl
# modules/vpc/main.tf
variable "name" {
  type = string
}

variable "cidr_block" {
  type    = string
  default = "10.0.0.0/16"
}

resource "aws_vpc" "this" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = true

  tags = {
    Name = var.name
  }
}

output "vpc_id" {
  value = aws_vpc.this.id
}

output "vpc_cidr" {
  value = aws_vpc.this.cidr_block
}
```

```hcl
# Using module
module "vpc" {
  source = "./modules/vpc"

  name       = "production-vpc"
  cidr_block = "10.0.0.0/16"
}

# Using public module
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway = true
}
```

---

## Environment Management

### Workspaces

```bash
# Create environments with workspaces
terraform workspace new staging
terraform workspace new production

# Switch
terraform workspace select staging

# In code
resource "aws_instance" "app" {
  instance_type = terraform.workspace == "production" ? "t3.large" : "t3.micro"
}
```

### Directory Structure

```
infrastructure/
├── modules/
│   ├── vpc/
│   ├── ecs/
│   └── rds/
├── environments/
│   ├── staging/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   └── production/
│       ├── main.tf
│       ├── variables.tf
│       └── terraform.tfvars
└── global/
    ├── iam/
    └── dns/
```

### Terragrunt (DRY Environments)

```hcl
# terragrunt.hcl (root)
remote_state {
  backend = "s3"
  config = {
    bucket         = "my-terraform-state"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
  }
}

# environments/staging/terragrunt.hcl
include {
  path = find_in_parent_folders()
}

terraform {
  source = "../../modules/app"
}

inputs = {
  environment   = "staging"
  instance_type = "t3.micro"
}
```

---

## Drift Detection

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Configuration Drift                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Drift: Actual infrastructure differs from Terraform state          │
│                                                                     │
│  Causes:                                                            │
│  - Manual changes in console                                        │
│  - Changes by other tools                                           │
│  - External automation                                              │
│                                                                     │
│  Detection:                                                         │
│  terraform plan                # Shows drift                        │
│  terraform refresh             # Update state from actual           │
│                                                                     │
│  Prevention:                                                        │
│  - No manual changes policy                                         │
│  - Regular plan runs in CI                                          │
│  - AWS Config / CloudTrail alerts                                   │
│  - Automated drift remediation                                      │
│                                                                     │
│  Resolution:                                                        │
│  - terraform apply  (revert to desired)                             │
│  - Update code to match reality                                     │
│  - terraform import (bring into state)                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## CI/CD for Terraform

```yaml
# .github/workflows/terraform.yml
name: Terraform

on:
  pull_request:
    paths:
      - 'infrastructure/**'
  push:
    branches:
      - main
    paths:
      - 'infrastructure/**'

jobs:
  terraform:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: infrastructure/environments/staging

    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3

      - name: Terraform Init
        run: terraform init

      - name: Terraform Format Check
        run: terraform fmt -check

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan
        run: terraform plan -out=tfplan
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

      - name: Terraform Apply
        if: github.ref == 'refs/heads/main'
        run: terraform apply -auto-approve tfplan
```

---

## Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Terraform Best Practices                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Version control everything                                      │
│     - Code, modules, tfvars                                         │
│     - NOT .terraform directory                                      │
│                                                                     │
│  2. Use remote state with locking                                   │
│     - Never local state in teams                                    │
│                                                                     │
│  3. Use modules for reusability                                     │
│     - Keep modules small and focused                                │
│     - Version your modules                                          │
│                                                                     │
│  4. Separate environments                                           │
│     - Different state files                                         │
│     - Different variable files                                      │
│                                                                     │
│  5. Use variables and locals                                        │
│     - No hardcoded values                                           │
│     - Default sensible values                                       │
│                                                                     │
│  6. Always plan before apply                                        │
│     - Review changes                                                │
│     - Save plan in CI/CD                                            │
│                                                                     │
│  7. Handle secrets properly                                         │
│     - Use secret managers                                           │
│     - Use sensitive = true                                          │
│     - Don't commit tfvars with secrets                              │
│                                                                     │
│  8. Tag everything                                                  │
│     - Environment, owner, project                                   │
│     - Use default_tags in provider                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## На интервью

### Типичные вопросы

1. **What is Terraform state?**
   - Mapping of config to real resources
   - Contains resource IDs, attributes
   - Used to plan changes
   - Should be remote, encrypted, locked

2. **How do you manage multiple environments?**
   - Workspaces (simple)
   - Directory per environment (more isolation)
   - Terragrunt (DRY)
   - Same modules, different variables

3. **What is drift?**
   - When real infrastructure differs from state
   - Caused by manual changes
   - Detected by terraform plan
   - Fix by apply or import

4. **How do you handle secrets?**
   - Never in code or tfvars
   - Use secret managers (Vault, AWS Secrets Manager)
   - Reference via data sources
   - Mark outputs as sensitive

5. **Modules best practices?**
   - Single responsibility
   - Version pinning
   - Clear inputs/outputs
   - Documentation
   - Don't nest too deep

6. **terraform import vs terraform state mv?**
   - import: bring existing resource into state
   - state mv: rename/move resource in state
   - Both require code changes to match

---

[← Назад к списку тем](README.md)
