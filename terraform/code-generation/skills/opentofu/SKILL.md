---
name: opentofu
description: Generate OpenTofu configurations with OpenTofu-specific features and best practices. Use when working with OpenTofu (the open-source Terraform fork) including state encryption, registry differences, and tofu CLI commands.
---

# OpenTofu

OpenTofu is an open-source fork of Terraform, maintained by the Linux Foundation. It is fully compatible with Terraform configurations while adding additional features and maintaining an open governance model.

**Reference:** [OpenTofu Documentation](https://opentofu.org/docs/)

## When to Use This Skill

Use this skill when:
- The project uses OpenTofu instead of Terraform
- You need OpenTofu-specific features like state encryption
- Working with the OpenTofu Registry
- Using `tofu` CLI commands

## Key Differences from Terraform

### CLI Commands

Replace `terraform` with `tofu`:

```bash
# Terraform              # OpenTofu
terraform init           tofu init
terraform plan           tofu plan
terraform apply          tofu apply
terraform destroy        tofu destroy
terraform fmt            tofu fmt
terraform validate       tofu validate
terraform test           tofu test
```

### Provider Registry

OpenTofu maintains its own registry at `registry.opentofu.org`. Providers from the Terraform Registry work transparently, but you can explicitly use the OpenTofu registry:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"  # Works - resolves via OpenTofu registry
      version = "~> 5.0"
    }
  }
}
```

For providers only in the OpenTofu registry:

```hcl
terraform {
  required_providers {
    example = {
      source  = "registry.opentofu.org/example/provider"
      version = "~> 1.0"
    }
  }
}
```

### Version Constraints

OpenTofu versions follow their own release schedule. Specify OpenTofu version requirements:

```hcl
terraform {
  required_version = ">= 1.6.0"  # OpenTofu version
}
```

## OpenTofu-Specific Features

### State Encryption

OpenTofu supports encrypting state files at rest (available since OpenTofu 1.7.0):

```hcl
terraform {
  encryption {
    key_provider "pbkdf2" "main" {
      passphrase = var.state_encryption_passphrase
    }

    method "aes_gcm" "default" {
      keys = key_provider.pbkdf2.main
    }

    state {
      method = method.aes_gcm.default
    }

    plan {
      method = method.aes_gcm.default
    }
  }
}
```

**Key Providers:**
- `pbkdf2` - Password-based key derivation
- `aws_kms` - AWS Key Management Service
- `gcp_kms` - Google Cloud KMS
- `openbao` - OpenBao (HashiCorp Vault fork)

**AWS KMS Example:**

```hcl
terraform {
  encryption {
    key_provider "aws_kms" "main" {
      kms_key_id = "alias/opentofu-state"
      region     = "us-west-2"
    }

    method "aes_gcm" "default" {
      keys = key_provider.aws_kms.main
    }

    state {
      method = method.aes_gcm.default
    }
  }
}
```

### Early Variable/Local Evaluation

OpenTofu allows variables and locals in backend and provider configurations:

```hcl
variable "environment" {
  type = string
}

terraform {
  backend "s3" {
    bucket = "tfstate-${var.environment}"  # Variable in backend config
    key    = "state.tfstate"
    region = "us-west-2"
  }
}
```

### Provider-Defined Functions

OpenTofu 1.7+ supports provider-defined functions:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Use provider-defined function
locals {
  arn_parts = provider::aws::arn_parse("arn:aws:s3:::my-bucket")
}
```

### For Each with Maps in Modules

OpenTofu has improved `for_each` support in module blocks:

```hcl
variable "environments" {
  type = map(object({
    instance_type = string
    instance_count = number
  }))
}

module "app" {
  for_each = var.environments
  source   = "./modules/app"

  environment    = each.key
  instance_type  = each.value.instance_type
  instance_count = each.value.instance_count
}
```

## Testing with OpenTofu

OpenTofu uses the same testing framework as Terraform:

```bash
tofu test
tofu test -verbose
tofu test tests/specific_test.tftest.hcl
```

Test files use identical syntax:

```hcl
# tests/example.tftest.hcl
run "test_defaults" {
  command = plan

  assert {
    condition     = aws_instance.example.instance_type == "t2.micro"
    error_message = "Instance type should be t2.micro"
  }
}
```

## Migration from Terraform

### State File Compatibility

OpenTofu can read Terraform state files directly. No migration is required for state:

```bash
# Simply switch to using tofu commands
tofu init
tofu plan  # Works with existing Terraform state
```

### Lock File Updates

After switching, regenerate the lock file:

```bash
rm .terraform.lock.hcl
tofu init
```

### CI/CD Updates

Update CI/CD pipelines to use the OpenTofu CLI:

**GitHub Actions:**

```yaml
- name: Setup OpenTofu
  uses: opentofu/setup-opentofu@v1
  with:
    tofu_version: 1.7.0

- name: OpenTofu Init
  run: tofu init

- name: OpenTofu Plan
  run: tofu plan
```

**GitLab CI:**

```yaml
image: ghcr.io/opentofu/opentofu:1.7

stages:
  - validate
  - plan

validate:
  stage: validate
  script:
    - tofu init
    - tofu validate
    - tofu fmt -check

plan:
  stage: plan
  script:
    - tofu init
    - tofu plan
```

## File Organization

OpenTofu uses the same file organization as Terraform:

| File | Purpose |
|------|---------|
| `terraform.tf` or `opentofu.tf` | Version requirements and encryption |
| `providers.tf` | Provider configurations |
| `main.tf` | Primary resources and data sources |
| `variables.tf` | Input variable declarations |
| `outputs.tf` | Output value declarations |
| `locals.tf` | Local value declarations |

**Note:** The filename `opentofu.tf` is not required - OpenTofu reads all `.tf` files the same as Terraform.

## Example: Complete OpenTofu Configuration

```hcl
# terraform.tf
terraform {
  required_version = ">= 1.7.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  # OpenTofu-specific: state encryption
  encryption {
    key_provider "aws_kms" "state" {
      kms_key_id = "alias/opentofu-state"
      region     = "us-west-2"
    }

    method "aes_gcm" "secure" {
      keys = key_provider.aws_kms.state
    }

    state {
      method = method.aes_gcm.secure
    }

    plan {
      method = method.aes_gcm.secure
    }
  }

  backend "s3" {
    bucket         = "my-opentofu-state"
    key            = "prod/terraform.tfstate"
    region         = "us-west-2"
    encrypt        = true
    dynamodb_table = "opentofu-locks"
  }
}

# providers.tf
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      ManagedBy = "OpenTofu"
      Project   = var.project_name
    }
  }
}

# variables.tf
variable "aws_region" {
  description = "AWS region for resources"
  type        = string
  default     = "us-west-2"
}

variable "project_name" {
  description = "Name of the project"
  type        = string
}

variable "environment" {
  description = "Deployment environment"
  type        = string

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

# main.tf
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true

  tags = {
    Name        = "${var.project_name}-${var.environment}-vpc"
    Environment = var.environment
  }
}

# outputs.tf
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}
```

## Validation Commands

```bash
tofu fmt -recursive      # Format all files
tofu validate            # Validate configuration
tofu test                # Run tests
```

## Version Control

**Never commit:**
- `terraform.tfstate`, `terraform.tfstate.backup`
- `.terraform/` directory
- `*.tfplan`
- `.tfvars` files with sensitive data

**Always commit:**
- All `.tf` configuration files
- `.terraform.lock.hcl` (dependency lock file)

## References

- [OpenTofu Documentation](https://opentofu.org/docs/)
- [OpenTofu Registry](https://registry.opentofu.org/)
- [State Encryption](https://opentofu.org/docs/language/state/encryption/)
- [OpenTofu GitHub](https://github.com/opentofu/opentofu)
- [Migration Guide](https://opentofu.org/docs/intro/migration/)
