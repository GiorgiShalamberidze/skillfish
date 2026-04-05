---
name: terraform-engineer
description: Terraform/OpenTofu IaC: module design, state management, workspaces, provider patterns, drift detection, testing with Terratest, and multi-cloud patterns.
---

# Terraform Engineer

Infrastructure as Code specialist for Terraform and OpenTofu. Covers the full
lifecycle: writing idiomatic HCL, designing reusable modules, managing remote state,
structuring environments, configuring multi-cloud providers, testing infrastructure,
CI/CD integration, and advanced deployment patterns like blue-green and
zero-downtime migrations.

---

## Table of Contents

1. [HCL Fundamentals](#1-hcl-fundamentals)
2. [Module Design](#2-module-design)
3. [State Management](#3-state-management)
4. [Workspaces and Environments](#4-workspaces-and-environments)
5. [Provider Patterns](#5-provider-patterns)
6. [Testing](#6-testing)
7. [CI/CD Integration](#7-cicd-integration)
8. [Advanced Patterns](#8-advanced-patterns)

---

## 1. HCL Fundamentals

### Resources and Data Sources

Resources declare infrastructure objects. Data sources fetch read-only information
from existing infrastructure or other configurations.

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
  owners = ["099720109477"]
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type
  tags = {
    Name        = "${var.project}-web"
    Environment = var.environment
  }
}
```

### Variables and Outputs

Always set `type`, `description`, and use `validation` blocks for domain constraints.
Mark sensitive outputs explicitly.

```hcl
variable "instance_type" {
  description = "EC2 instance size for the web tier."
  type        = string
  default     = "t3.micro"
  validation {
    condition     = can(regex("^t3\\.", var.instance_type))
    error_message = "Only t3 instance types are allowed."
  }
}

output "instance_id" {
  description = "ID of the web server instance."
  value       = aws_instance.web.id
}

output "db_password" {
  description = "Generated database password."
  value       = random_password.db.result
  sensitive   = true
}
```

### Locals

Compute intermediate values and reduce repetition.

```hcl
locals {
  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "terraform"
  }
  name_prefix = "${var.project}-${var.environment}"
}
```

### Dynamic Blocks

Generate repeated nested blocks from a collection. Use sparingly -- they hurt
readability when overused.

```hcl
resource "aws_security_group" "web" {
  name   = "${local.name_prefix}-web-sg"
  vpc_id = var.vpc_id

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }
}
```

### for_each vs count

Prefer `for_each` -- it keys resources by map key, so adds/removes are surgical.
`count` uses numeric indices, meaning removing a middle item forces recreation of
all subsequent resources.

```hcl
# for_each: stable resource addresses
resource "aws_s3_bucket" "this" {
  for_each = var.buckets
  bucket   = "${local.name_prefix}-${each.key}"
}

# count: acceptable for simple on/off toggles
resource "aws_cloudwatch_log_group" "app" {
  count = var.enable_logging ? 1 : 0
  name  = "/app/${var.project}"
}
```

### Moved Blocks

Tell Terraform a resource changed address so it does not destroy and recreate.

```hcl
moved {
  from = aws_instance.web
  to   = aws_instance.app
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type
}
```

### Anti-Patterns

- **Hardcoded values** inside resource blocks. Extract to variables or locals.
- **`count` with lists that may reorder.** Switch to `for_each` with a map.
- **Deeply nested dynamic blocks.** More than two levels deep -- refactor into a child module.
- **Missing descriptions** on variables and outputs.

---

## 2. Module Design

### Module Structure

```
modules/vpc/
  main.tf          # Resource definitions
  variables.tf     # Input variables
  outputs.tf       # Output values
  versions.tf      # required_version + required_providers
  README.md        # Auto-generated with terraform-docs
  examples/simple/main.tf
```

### Input/Output Contracts

Design interfaces like API contracts. Use `object` types for grouped inputs.

```hcl
variable "network" {
  description = "Network configuration for the VPC."
  type = object({
    cidr_block         = string
    availability_zones = list(string)
    public_subnets     = list(string)
    private_subnets    = list(string)
  })
  validation {
    condition     = length(var.network.availability_zones) >= 2
    error_message = "At least two AZs are required for HA."
  }
}

output "vpc_id" {
  description = "The ID of the VPC."
  value       = aws_vpc.this.id
}

output "private_subnet_ids" {
  description = "List of private subnet IDs."
  value       = [for s in aws_subnet.private : s.id]
}
```

### Composition Patterns

Compose from small, focused modules. Root modules wire them together.

```hcl
module "vpc" {
  source  = "./modules/vpc"
  network = {
    cidr_block         = "10.0.0.0/16"
    availability_zones = ["us-east-1a", "us-east-1b"]
    public_subnets     = ["10.0.1.0/24", "10.0.2.0/24"]
    private_subnets    = ["10.0.10.0/24", "10.0.11.0/24"]
  }
}

module "ecs_cluster" {
  source     = "./modules/ecs"
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnet_ids
}
```

### Versioning

Pin module versions in production. For registry modules use `version`; for Git use
`ref`.

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
}

module "internal" {
  source = "git::https://github.com/acme/tf-modules.git//vpc?ref=v2.1.0"
}
```

### Anti-Patterns

- **Mega-modules** managing dozens of resources. Split into focused modules.
- **Unpinned module sources.** Always pin a version or Git ref.
- **Leaking implementation details** through outputs. Expose only what callers need.
- **No examples.** Every module needs at least one working example.

---

## 3. State Management

### Remote Backends

Store state remotely with locking to prevent concurrent writes.

```hcl
# AWS S3 + DynamoDB
terraform {
  backend "s3" {
    bucket         = "acme-terraform-state"
    key            = "prod/network/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-lock"
  }
}

# GCS
terraform {
  backend "gcs" {
    bucket = "acme-terraform-state"
    prefix = "prod/network"
  }
}
```

### State Locking

S3 uses DynamoDB; GCS uses native object locking. If a lock is stuck after a crash:

```bash
# Only after verifying no other operation is running
terraform force-unlock LOCK_ID
```

### Import

Bring existing infrastructure under management. Terraform 1.5+ supports declarative
import blocks.

```hcl
import {
  to = aws_s3_bucket.legacy
  id = "my-existing-bucket"
}

resource "aws_s3_bucket" "legacy" {
  bucket = "my-existing-bucket"
}
```

```bash
# CLI import (all versions)
terraform import aws_s3_bucket.legacy my-existing-bucket
```

After importing, run `terraform plan` and adjust until the plan shows no changes.

### State mv and rm

```bash
terraform state mv aws_instance.old aws_instance.new
terraform state mv aws_instance.app module.compute.aws_instance.app
terraform state rm aws_instance.temporary   # Remove without destroying
```

Prefer `moved` blocks over `terraform state mv` -- they are version-controlled.

### Sensitive Data in State

1. **Encrypt at rest.** S3: `encrypt = true`; GCS encrypts by default.
2. **Restrict access.** Least-privilege IAM on state bucket.
3. **Never commit state.** Add `*.tfstate`, `*.tfstate.*`, `.terraform/` to `.gitignore`.
4. **Mark outputs sensitive.** Hides from CLI but still exists in state.

### Anti-Patterns

- **Local state in production.** Always use a remote backend with locking.
- **Single state file for everything.** Split by service/layer to reduce blast radius.
- **`force-unlock` without verifying** no other operation is active.
- **Secrets in plain-text tfvars.** Use a secrets manager and data sources.

---

## 4. Workspaces and Environments

### Workspace Strategies

Workspaces manage multiple state instances of the same configuration.

```bash
terraform workspace new staging
terraform workspace select staging
```

```hcl
locals {
  environment   = terraform.workspace
  instance_type = { staging = "t3.small", production = "t3.large" }
}

resource "aws_instance" "app" {
  instance_type = local.instance_type[local.environment]
}
```

### Directory-Based vs Workspace-Based

**Workspace-based** -- single codebase, separate state per workspace:

```bash
terraform workspace select staging
terraform apply -var-file=envs/staging.tfvars
```

**Directory-based** -- separate directories per environment:

```
environments/
  staging/
    main.tf    # calls shared modules
    backend.tf
    terraform.tfvars
  production/
    main.tf
    backend.tf
    terraform.tfvars
modules/
  app/
  network/
```

Directory-based is more explicit and avoids applying to the wrong workspace.
Workspace-based is simpler for small teams with near-identical environments.

### tfvars per Environment

```hcl
# envs/staging.tfvars
environment    = "staging"
instance_type  = "t3.small"
instance_count = 2

# envs/production.tfvars
environment    = "production"
instance_type  = "t3.large"
instance_count = 6
```

Never put secrets in tfvars -- reference a secrets manager instead.

### Anti-Patterns

- **Applying without the correct workspace or var-file.** Automate selection in CI.
- **Divergent environment configs.** Keep them similar; differ only on scale/secrets.
- **Unchecked `.terraform.lock.hcl` changes.** Commit the lock file but review version bumps.

---

## 5. Provider Patterns

### Provider Configuration

```hcl
terraform {
  required_version = ">= 1.5"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
  default_tags {
    tags = local.common_tags
  }
}
```

### Provider Aliases

Manage resources across multiple regions or accounts.

```hcl
provider "aws" {
  alias  = "us_east"
  region = "us-east-1"
}

provider "aws" {
  alias  = "eu_west"
  region = "eu-west-1"
}

resource "aws_s3_bucket" "eu_bucket" {
  provider = aws.eu_west
  bucket   = "acme-eu-assets"
}

module "cdn" {
  source    = "./modules/cdn"
  providers = { aws = aws.us_east }
}
```

### AWS: Cross-Account Access

```hcl
provider "aws" {
  alias  = "shared_services"
  region = "us-east-1"
  assume_role {
    role_arn     = "arn:aws:iam::123456789012:role/TerraformAdmin"
    session_name = "terraform"
  }
}
```

### GCP: API Enablement

```hcl
provider "google" {
  project = var.gcp_project
  region  = var.gcp_region
}

resource "google_project_service" "compute" {
  service            = "compute.googleapis.com"
  disable_on_destroy = false
}

resource "google_compute_instance" "app" {
  depends_on = [google_project_service.compute]
  # ...
}
```

### Azure: Resource Groups

```hcl
provider "azurerm" {
  features {}
  subscription_id = var.subscription_id
}

resource "azurerm_resource_group" "app" {
  name     = "rg-${var.project}-${var.environment}"
  location = var.azure_region
}
```

### Anti-Patterns

- **No version constraints** on `required_providers`. Pin to a minor range.
- **Default provider with multiple regions.** Be explicit with aliases.
- **Hardcoded account/project IDs.** Pass as variables or read from data sources.

---

## 6. Testing

### terraform validate

Quickest sanity check -- validates syntax without remote API calls.

```bash
terraform init -backend=false
terraform validate
```

### Plan Review

```bash
terraform plan -out=tfplan
terraform show -json tfplan | jq '.resource_changes[] | {address, actions: .change.actions}'
terraform apply tfplan   # Apply the exact saved plan
```

### Terratest

Go library that deploys real infrastructure, validates it, and tears it down.

```go
func TestVpcModule(t *testing.T) {
    t.Parallel()
    opts := &terraform.Options{
        TerraformDir: "../modules/vpc",
        Vars: map[string]interface{}{
            "cidr_block": "10.99.0.0/16",
            "azs":        []string{"us-east-1a", "us-east-1b"},
        },
    }
    defer terraform.Destroy(t, opts)
    terraform.InitAndApply(t, opts)
    assert.NotEmpty(t, terraform.Output(t, opts, "vpc_id"))
}
```

### Native Testing (tftest)

Terraform 1.6+ built-in test framework using `.tftest.hcl` files.

```hcl
# tests/vpc.tftest.hcl
run "create_vpc" {
  command = apply
  variables {
    cidr_block = "10.99.0.0/16"
    azs        = ["us-east-1a", "us-east-1b"]
  }
  assert {
    condition     = output.vpc_id != ""
    error_message = "VPC ID must not be empty."
  }
}
```

```bash
terraform test
```

### Static Analysis (checkov / tfsec)

```bash
checkov -d . --framework terraform
tfsec .
trivy config .     # Trivy is the successor to tfsec
```

### Sentinel (Terraform Cloud / Enterprise)

Policy-as-code for governance.

```python
import "tfplan/v2" as tfplan

allowed_types = ["t3.micro", "t3.small", "t3.medium", "t3.large"]

main = rule {
    all tfplan.resource_changes as _, rc {
        rc.type is "aws_instance" implies
            rc.change.after.instance_type in allowed_types
    }
}
```

### Anti-Patterns

- **Skipping `validate` in CI.** It catches syntax errors before plan.
- **Applying without a saved plan.** Infrastructure may drift between plan and apply.
- **No real infrastructure tests.** Test critical modules with Terratest/tftest in a sandbox.
- **Ignoring static analysis.** Treat security scanner results as CI failures.

---

## 7. CI/CD Integration

### GitHub Actions

```yaml
name: Terraform
on:
  pull_request:
    paths: ["infra/**"]
  push:
    branches: [main]
    paths: ["infra/**"]

permissions:
  contents: read
  pull-requests: write
  id-token: write

jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with: { terraform_version: "1.7.x" }
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.TF_ROLE_ARN }}
          aws-region: us-east-1
      - run: terraform init && terraform validate && terraform plan -no-color -out=tfplan
        working-directory: infra

  apply:
    needs: plan
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with: { terraform_version: "1.7.x" }
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.TF_ROLE_ARN }}
          aws-region: us-east-1
      - run: terraform init && terraform apply -auto-approve
        working-directory: infra
```

### Atlantis

PR-driven plan/apply. Comment `atlantis apply` to apply.

```yaml
# atlantis.yaml
version: 3
projects:
  - name: network
    dir: infra/network
    autoplan:
      when_modified: ["*.tf", "*.tfvars"]
      enabled: true
  - name: app
    dir: infra/app
    autoplan:
      when_modified: ["*.tf", "*.tfvars"]
      enabled: true
```

### Terraform Cloud

Remote execution, state management, and policy enforcement.

```hcl
terraform {
  cloud {
    organization = "acme-corp"
    workspaces { name = "app-production" }
  }
}
```

### Plan-Apply Workflow

1. **PR opened** -- CI runs `terraform plan`, posts output to PR.
2. **Review** -- team reviews HCL changes and plan output together.
3. **PR merged** -- CI applies with the saved plan or fresh plan in an
   environment-gated job.
4. **Post-apply** -- verify success and run smoke tests.

### Drift Detection

Schedule periodic plans to catch out-of-band changes. Use
`terraform plan -detailed-exitcode` which returns 2 for drift, 0 for clean,
1 for error.

```yaml
name: Drift Detection
on:
  schedule:
    - cron: "0 8 * * 1-5"
jobs:
  detect:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.TF_ROLE_ARN }}
          aws-region: us-east-1
      - run: terraform init
        working-directory: infra
      - name: Detect Drift
        working-directory: infra
        run: |
          terraform plan -detailed-exitcode -no-color || exit_code=$?
          [ "$exit_code" = "2" ] && echo "::warning::Drift detected!" && exit 1
```

### Anti-Patterns

- **`apply -auto-approve` on PRs.** Only auto-approve on main after review.
- **Skipping plan.** Always plan first, even in automation.
- **No drift detection.** Console changes silently diverge from code.
- **Long-lived credentials.** Use OIDC / workload identity federation.

---

## 8. Advanced Patterns

### Conditional Resources

```hcl
variable "enable_cdn" {
  description = "Whether to provision a CloudFront distribution."
  type        = bool
  default     = false
}

resource "aws_cloudfront_distribution" "cdn" {
  count   = var.enable_cdn ? 1 : 0
  enabled = true
  origin {
    domain_name = aws_s3_bucket.assets.bucket_regional_domain_name
    origin_id   = "s3-assets"
  }
}

output "cdn_domain" {
  value = var.enable_cdn ? aws_cloudfront_distribution.cdn[0].domain_name : null
}
```

### Zero-Downtime with create_before_destroy

```hcl
resource "aws_launch_template" "app" {
  name_prefix   = "${local.name_prefix}-"
  image_id      = var.ami_id
  instance_type = var.instance_type
  lifecycle { create_before_destroy = true }
}

resource "aws_autoscaling_group" "app" {
  desired_capacity = var.instance_count
  max_size         = var.instance_count * 2
  min_size         = var.instance_count
  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }
  instance_refresh {
    strategy = "Rolling"
    preferences { min_healthy_percentage = 90 }
  }
}
```

### Blue-Green with Weighted Routing

```hcl
variable "blue_weight" {
  description = "Traffic weight for blue (0-100)."
  type        = number
  default     = 100
}

resource "aws_route53_record" "blue" {
  zone_id        = var.zone_id
  name           = "app.example.com"
  type           = "A"
  set_identifier = "blue"
  weighted_routing_policy { weight = var.blue_weight }
  alias {
    name                   = module.blue.alb_dns_name
    zone_id                = module.blue.alb_zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "green" {
  zone_id        = var.zone_id
  name           = "app.example.com"
  type           = "A"
  set_identifier = "green"
  weighted_routing_policy { weight = 100 - var.blue_weight }
  alias {
    name                   = module.green.alb_dns_name
    zone_id                = module.green.alb_zone_id
    evaluate_target_health = true
  }
}
```

Deploy: set `blue_weight = 100`, deploy green, smoke test, shift to `blue_weight = 0`.

### Data Source Chaining

Chain data sources to look up dependent infrastructure dynamically.

```hcl
data "aws_vpc" "main" {
  filter {
    name   = "tag:Name"
    values = ["${var.project}-vpc"]
  }
}

data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }
  tags = { Tier = "private" }
}

locals {
  private_subnet_ids = data.aws_subnets.private.ids
}
```

### Moved Blocks for Module Refactoring

Chain `moved` blocks to keep state intact across renames and module extractions.

```hcl
moved {
  from = aws_security_group.app
  to   = module.network.aws_security_group.app
}

moved {
  from = module.network.aws_security_group.app
  to   = module.network.aws_security_group.main
}
```

### Preventing Accidental Destruction

```hcl
resource "aws_rds_instance" "primary" {
  identifier     = "${local.name_prefix}-db"
  engine         = "postgres"
  instance_class = "db.r6g.large"
  lifecycle { prevent_destroy = true }
}
```

To intentionally remove: set `prevent_destroy = false`, apply, then remove the block.

### OpenTofu Compatibility

Most patterns apply to both. OpenTofu adds native state encryption and uses MPL 2.0
(Terraform is BSL 1.1 since v1.6). The CLI is a drop-in replacement:

```bash
tofu init && tofu plan && tofu apply
```

### Anti-Patterns

- **No `prevent_destroy` on stateful resources.** Databases and buckets need guards.
- **Skipping `moved` blocks during refactors.** Resources get destroyed and recreated.
- **Hardcoded routing weights.** Parameterize so cutover needs no code change.
- **Data sources for resources you manage.** Reference via module outputs instead.
