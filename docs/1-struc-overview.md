---
sidebar_position: 1
---

# Project Directory Overview

### Directory Structure

This structure organizes Terraform code into reusable modules and environment-specific configurations. It ensures clean separation between backend state, environments, and core infrastructure components.

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

### **backend**
- Contains the configuration for the **remote Terraform state** backend
- This directory is deployed **once** to bootstrap the remote state infrastructure
- The backend must be deployed **before** any other environment (dev, prod, etc.). Without the remote state bucket in place, Terraform cannot initialize or store state remotely.

### **env/**
- Manage multiple environments (e.g., `dev`, `integ`, `prod`) using the same module sets with different parameters
- Extend this pattern by creating additional environments such as (e.g., `staging` or `test`)
- Ensure each environment uses a unique state key (e.g., `dev/terraform.tfstate`, `prod/terraform.tfstate`) to prevent environments from overwriting each other’s state

### **modules/**
- Reusable Terraform modules encapsulating core infrastructure components
- Version-control modules independently by moving them to their own repositories and referencing them via Git URLs or Terraform Registry sources

---
## How It Works Together

1. **Backend** creates the S3 bucket and configures the remote state.
2. Each **environment** (`dev`, `prod`, etc.) uses that remote state to read outputs and maintain isolation.
3. **Modules** define reusable components that can be versioned and shared across environments.

:::important
- Always run terraform init from inside the environment folder (e.g., env/dev/) — not from the project root — so Terraform can load the corresponding backend and module paths.
- Avoid directly modifying files inside terraform.tfstate or reusing the same state file between environments — this can lead to state corruption.
:::

---
## Reference

- **Github:** [Project Directory Structure](https://github.com/deeowemez/minicommerce/tree/main/infra) | [Environment Sample Configuration File](https://github.com/deeowemez/minicommerce/blob/main/infra/env/dev/main.tf)
