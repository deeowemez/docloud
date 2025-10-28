---
sidebar_position: 1
---

# Project Directory Overview

### Directory Structure

```
.
├── backend
│   ├── main.tf
│   ├── outputs.tf
│   └── terraform.tfstate
├── env
│   ├── dev
│   │   ├── main.tf
│   │   ├── outputs.tf
│   │   └── variables.tf
│   ├── integ
│   └── prod
└── modules
    ├── compute
    ├── data
    ├── frontend
    │   ├── main.tf
    │   ├── outputs.tf
    │   └── variables.tf
    ├── iam
    │   ├── main.tf
    │   ├── outputs.tf
    │   └── variables.tf
    └── network
        ├── main.tf
        ├── outputs.tf
        └── variables.tf
```

---

## Directory Breakdown

#### **backend**
- Contains the configuration for the **remote Terraform state** backend
- This directory is deployed **once** to bootstrap the remote state infrastructure
#### **env/**
- Manage multiple environments (e.g., `dev`, `integ`, `prod`) using the same module sets with different parameters
#### **modules/**
- Reusable Terraform modules encapsulating core infrastructure components

---
## How It Works Together

1. **Backend** creates the S3 bucket and configures the remote state.
2. Each **environment** (`dev`, `prod`, etc.) uses that remote state to read outputs and maintain isolation.
3. **Modules** define reusable components that can be versioned and shared across environments.
4. The result is a **clean, maintainable, and scalable Terraform setup** aligned with best practices for IaC projects.

---
### Sample File
###### <i>env/dev/main.tf</i>
```
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
  backend "s3" {
    bucket       = "mc-remote-state"
    key          = "dev/terraform.tfstate"
    region       = "us-east-1"
    use_lockfile = true
  }
}

provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "use1"
  region = "us-east-1"
}

module "frontend_use1" {
  source                 = "../../modules/frontend"
  providers              = { aws = aws.use1 }
  aws_region             = var.aws_region_use1
  project_name           = "${var.project_name}-${var.aws_region_alias_use1}"
  db_remote_state_bucket = var.db_remote_state_bucket
  db_remote_state_key    = var.db_remote_state_key
}

module "network_use1" {
  source                 = "../../modules/network"
  providers              = { aws = aws.use1 }
  aws_region             = var.aws_region_use1
  vpc_cidr               = "10.20.0.0/16"
  project_name           = "${var.project_name}-${var.aws_region_alias_use1}"
  db_remote_state_bucket = var.db_remote_state_bucket
  db_remote_state_key    = var.db_remote_state_key
}

module "iam_use1" {
  source                 = "../../modules/iam"
  frontend_bucket_arn    = module.frontend_use1.aws_s3_bucket_arn
  frontend_bucket_name   = module.frontend_use1.aws_s3_bucket_name
  aws_region             = var.aws_region_use1
  db_remote_state_bucket = var.db_remote_state_bucket
  db_remote_state_key    = var.db_remote_state_key
}
```
