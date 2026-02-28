# Multi-Provider Terraform Strategy

**Status**: Architecture & Pattern Design
**Scope**: AWS, Google Cloud, Microsoft Azure
**Goal**: Enable customers to choose their cloud provider with identical Dev-House deployment workflow

---

## The Problem

Current Terraform modules are **AWS-specific**:
- `modules/rds-postgres` → `aws_db_instance`
- `modules/vpc` → `aws_vpc`
- `modules/ecs-service` → `aws_ecs_task_definition`

**Challenge**: Azure and Google Cloud have fundamentally different APIs, naming conventions, and architectural patterns. A `db.t3.micro` in AWS doesn't translate 1:1 to Azure or GCP.

**Opportunity**: Dev-House can use Claude to generate provider-specific Terraform because each provider has well-documented APIs and patterns.

---

## Strategy Overview

### Option 1: Provider-Specific Module Variants (Recommended)

Create parallel module structures for each provider:

```
terraform/
├── modules/
│   ├── postgresql-database/
│   │   ├── aws/
│   │   │   ├── main.tf (aws_db_instance)
│   │   │   ├── variables.tf
│   │   │   └── outputs.tf
│   │   ├── azure/
│   │   │   ├── main.tf (azurerm_postgresql_server)
│   │   │   ├── variables.tf
│   │   │   └── outputs.tf
│   │   ├── gcp/
│   │   │   ├── main.tf (google_sql_database_instance)
│   │   │   ├── variables.tf
│   │   │   └── outputs.tf
│   │   └── README.md (provider-agnostic interface)
│   │
│   ├── vpc-network/
│   │   ├── aws/
│   │   │   ├── main.tf (aws_vpc, aws_subnet)
│   │   │   ├── variables.tf
│   │   │   └── outputs.tf
│   │   ├── azure/
│   │   │   ├── main.tf (azurerm_virtual_network)
│   │   │   ├── variables.tf
│   │   │   └── outputs.tf
│   │   ├── gcp/
│   │   │   ├── main.tf (google_compute_network)
│   │   │   ├── variables.tf
│   │   │   └── outputs.tf
│   │   └── README.md
│   │
│   └── [... other modules with same provider split ...]
│
├── deployments/
│   └── acme-corp-prod/
│       ├── terraform.tfvars      # Customer config
│       ├── providers.tf           # Provider selection (AWS/Azure/GCP)
│       ├── main.tf               # Calls modules with $provider variable
│       └── backend.tf            # State management
│
└── templates/
    ├── rest-api-postgres-redis/
    │   ├── aws/
    │   │   ├── main.tf
    │   │   ├── variables.tf
    │   │   └── outputs.tf
    │   ├── azure/
    │   │   ├── main.tf
    │   │   ├── variables.tf
    │   │   └── outputs.tf
    │   ├── gcp/
    │   │   ├── main.tf
    │   │   ├── variables.tf
    │   │   └── outputs.tf
    │   └── README.md (abstract interface)
```

**Pros**:
- Clear separation: each provider is explicit
- Easy to customize per-provider differences
- No abstraction layer complexity
- Claude can reason about provider-specific features
- Simple module selection logic

**Cons**:
- Code duplication (~20-30% of content varies per provider)
- Maintenance burden (3x the module count)
- Onboarding difficulty (which version to use?)

**Mitigation**:
- Document common interface (variables, outputs)
- Abstract naming convention (PostgreSQL Database = `aws/main.tf` = `azure/main.tf`)
- Shared README explains equivalences

---

### Option 2: Terragrunt Provider Abstraction

Use Terragrunt to inject provider-specific values:

```hcl
# terragrunt.hcl (top level)
terraform {
  source = "."
}

generate "providers" {
  path      = "providers.tf"
  if_exists = "overwrite"
  contents  = templatefile("${get_repo_root()}/provider-templates/${var.cloud_provider}.tfpl", {
    region              = var.region
    subscription_id     = var.subscription_id
    project_id          = var.project_id
  })
}

# Merge provider-specific variables
inputs = merge(
  local.common_vars,
  local.provider_vars[var.cloud_provider]
)

locals {
  common_vars = {
    database_instance_size = "small"
    enable_backups         = true
  }

  provider_vars = {
    aws = {
      instance_class = "db.t3.micro"
      multi_az       = true
    }
    azure = {
      sku_name       = "B_Gen5_1"
      backup_retention = 7
    }
    gcp = {
      tier           = "db-f1-micro"
      availability_type = "REGIONAL"
    }
  }
}
```

**Pros**:
- Single module can have conditional logic per provider
- Cleaner merge of provider-specific configs
- DRY (Don't Repeat Yourself) principle
- Good for per-environment variations

**Cons**:
- Complex templating logic
- Hard to understand at first glance
- Terraform + Terragrunt complexity
- Harder for Claude to reason about

---

### Option 3: Hybrid (Provider-Specific Templates + Common Abstractions)

Best of both worlds:

```
terraform/
├── modules/
│   └── [Truly provider-agnostic shared modules]
│       ├── main.tf (uses provider variables)
│       ├── variables.tf
│       └── locals.tf (common config)
│
├── providers/
│   ├── aws/
│   │   └── [AWS-specific resource definitions]
│   ├── azure/
│   │   └── [Azure-specific resource definitions]
│   └── gcp/
│       └── [GCP-specific resource definitions]
│
└── deployments/
    ├── common.tfvars       # Shared config
    ├── aws.tfvars
    ├── azure.tfvars
    ├── gcp.tfvars
    └── main.tf             # Routes to provider-specific implementation
```

**Recommended approach for MVP**: **Option 1 (Provider-Specific Variants)** because:
1. Explicit and understandable
2. Claude can generate each variant independently
3. No abstraction layer bugs
4. Easy customer onboarding ("choose your cloud")
5. Future migration to Terragrunt is straightforward

---

## Customer Provider Selection

### In the PRD (Customer Input)

```markdown
# REST API for Task Management

## Cloud Provider
- **Provider**: AWS | Azure | Google Cloud
- **Region**: us-east-1 (AWS) | eastus (Azure) | us-central1 (GCP)
- **Budget Limit**: $5000/month

## Infrastructure Preferences
- Multi-region? Yes/No
- Disaster recovery? Yes/No
- Compliance**: HIPAA, SOC2, etc.
```

### In the Deployment Plan (Harness Output)

```yaml
metadata:
  customer: "acme-corp"
  prd_id: "prd_12345"
  cloud_provider: "aws"         # or "azure", "gcp"
  region: "us-east-1"
  availability_zones:
    - "us-east-1a"
    - "us-east-1b"

components:
  - id: "database"
    name: "PostgreSQL"
    type: "data-store"
    cloud_provider: "aws"
    provider_config:
      aws:
        engine: "postgres"
        instance_class: "db.t3.micro"
        allocated_storage: 20
      azure:
        sku_name: "B_Gen5_1"
        storage_mb: 51200
      gcp:
        tier: "db-f1-micro"
        activation_policy: "ALWAYS"
```

---

## Claude Code Generation Strategy

### For Each Component, Claude Generates Provider-Specific Code

```
Customer PRD
    ↓
Harness Analysis → Identifies provider (AWS/Azure/GCP)
    ↓
Claude (Sonnet) → Generates provider-specific Terraform
    ↓
Output: main.tf, variables.tf, outputs.tf in provider-specific syntax
    ↓
Agent commits to: terraform/modules/[component]/[provider]/
    ↓
Final merge: Terragrunt or provider-selection logic chooses the right module
```

### Claude Prompt Template

```
You are generating Terraform code for [COMPONENT].

**Provider**: [aws|azure|gcp]
**Region**: [region]
**Requirements**:
- [requirement 1]
- [requirement 2]

Generate:
1. main.tf - Resource definitions for [PROVIDER]
2. variables.tf - Input variables
3. outputs.tf - Outputs that other components depend on

**Provider-Specific Notes**:
- AWS: Use aws_db_instance for PostgreSQL
- Azure: Use azurerm_postgresql_server with flexible server
- GCP: Use google_sql_database_instance with postgres version

Ensure outputs are consistent across providers:
- endpoint: Database connection endpoint
- port: Connection port
- username: Admin username
- password_secret_name: Where the password is stored
```

---

## Module Interface Contract

Each module variant MUST have the same **logical interface** across providers:

```hcl
# All three providers implement this contract:

variable "cluster_name" {
  description = "Name of the database cluster"
  type        = string
}

variable "engine_version" {
  description = "Database engine version (14.0, 14.1, etc.)"
  type        = string
  default     = "14.0"
}

variable "instance_size" {
  description = "Compute size (small, medium, large)"
  type        = string
  default     = "small"
}

variable "backup_retention_days" {
  description = "Number of days to retain backups"
  type        = number
  default     = 7
}

# Output contract (same across all providers)
output "endpoint" {
  description = "Database connection endpoint"
  value       = "..."
}

output "port" {
  description = "Database port"
  value       = 5432  # Standardized
}

output "username" {
  description = "Admin username"
  value       = "..."
}

output "password_secret_name" {
  description = "Secrets manager key for password"
  value       = "..."
}
```

**Key Principle**: Variables are **logical** (small/medium/large), outputs are **universal** (endpoint, port, username).

---

## Provider-Specific Implementation Examples

### Example 1: PostgreSQL Database

#### AWS Implementation
```hcl
# terraform/modules/postgresql-database/aws/main.tf

resource "aws_db_instance" "postgres" {
  identifier        = var.cluster_name
  engine            = "postgres"
  engine_version    = var.engine_version
  instance_class    = local.instance_class_map[var.instance_size]
  allocated_storage = 20

  backup_retention_period = var.backup_retention_days
  backup_window           = "03:00-04:00"
  skip_final_snapshot     = false

  db_name  = "postgres"
  username = "admin"

  vpc_security_group_ids = [aws_security_group.postgres.id]
  db_subnet_group_name   = aws_db_subnet_group.postgres.name

  tags = {
    Name = var.cluster_name
  }
}

locals {
  instance_class_map = {
    small   = "db.t3.micro"
    medium  = "db.t3.small"
    large   = "db.t3.medium"
  }
}

output "endpoint" {
  value = aws_db_instance.postgres.endpoint
}

output "password_secret_name" {
  value = aws_secretsmanager_secret.postgres_password.name
}
```

#### Azure Implementation
```hcl
# terraform/modules/postgresql-database/azure/main.tf

resource "azurerm_postgresql_flexible_server" "postgres" {
  name                          = var.cluster_name
  resource_group_name           = var.resource_group_name
  location                       = var.location

  administrator_login           = "psqladmin"
  administrator_password        = random_password.postgres.result

  sku_name       = local.sku_name_map[var.instance_size]
  storage_mb     = 32768
  backup_retention_days         = var.backup_retention_days

  version = var.engine_version

  tags = {
    Name = var.cluster_name
  }
}

locals {
  sku_name_map = {
    small   = "B_Standard_B1s"
    medium  = "B_Standard_B2s"
    large   = "B_Standard_B4ms"
  }
}

output "endpoint" {
  value = "${azurerm_postgresql_flexible_server.postgres.fqdn}:5432"
}

output "password_secret_name" {
  value = azurerm_key_vault_secret.postgres_password.name
}
```

#### GCP Implementation
```hcl
# terraform/modules/postgresql-database/gcp/main.tf

resource "google_sql_database_instance" "postgres" {
  name             = var.cluster_name
  database_version = "POSTGRES_${replace(var.engine_version, ".", "")}"
  region           = var.region

  settings {
    tier              = local.tier_map[var.instance_size]
    availability_type = "REGIONAL"
    backup_configuration {
      enabled  = true
      start_time = "03:00"
    }
    backup_retention_settings {
      retained_backups = var.backup_retention_days
    }
  }

  tags = {
    Name = var.cluster_name
  }
}

locals {
  tier_map = {
    small   = "db-f1-micro"
    medium  = "db-g1-small"
    large   = "db-n1-standard-1"
  }
}

output "endpoint" {
  value = google_sql_database_instance.postgres.private_ip_address
}

output "password_secret_name" {
  value = google_secret_manager_secret.postgres_password.name
}
```

**Observations**:
- Variables are identical across providers
- Outputs have same semantic meaning but different syntax
- Internal resources vary (provider-specific resource types)
- Sizing maps translate logical names to provider-specific SKUs

---

## Customer Deployment Workflow

### Step 1: Specify Provider in PRD
```markdown
# My App

## Infrastructure
- Provider: AWS
- Region: us-east-1
```

### Step 2: Harness Analyzes and Creates Deployment Plan
```yaml
cloud_provider: aws
region: us-east-1
components:
  - id: database
    provider_config:
      aws:
        instance_class: db.t3.micro
```

### Step 3: Agents Generate Provider-Specific Terraform
```
Agent 1: Generate database Terraform
→ Uses prompt: "Generate PostgreSQL for AWS"
→ Outputs: terraform/modules/postgresql-database/aws/
→ Commits to: agent/1/database-aws branch
```

### Step 4: Final Deployment
```
Generated Terraform (AWS-specific)
  ↓
terraform init (AWS credentials)
  ↓
terraform plan
  ↓
terraform apply
  ↓
Running system on AWS
```

### Provider Migration (Future)
```
Change PRD: "Provider: GCP"
  ↓
Harness re-analyzes with GCP
  ↓
Agents generate GCP-specific Terraform
  ↓
Customer can compare AWS vs GCP configs
  ↓
Choose to migrate or keep as-is
```

---

## Implementation Phases

### MVP (Phase 1)
- ✅ AWS modules fully working
- ✅ Azure modules (basic: VPC, database, compute)
- ✅ GCP modules (basic: VPC, database, compute)
- ✅ Module interface contract documented
- ✅ Claude prompts for provider-specific generation

### Phase 1 (Phase 2)
- Azure & GCP feature parity with AWS
- Terragrunt integration (optional, if needed)
- Provider cost comparison tools
- Migration guides (AWS → Azure, AWS → GCP)

### Phase 2 (Phase 3+)
- Advanced features per provider (Azure Blueprints, GCP Organization Policy)
- Multi-cloud deployments (VPN between AWS and Azure)
- Provider-agnostic service mesh (Istio)
- Cost optimization per provider

---

## Module Organization Decision Matrix

| Aspect | Provider-Specific Variants | Terragrunt Abstraction | Hybrid |
|--------|---------------------------|----------------------|--------|
| **Clarity** | High (explicit) | Medium (complex) | High |
| **Maintenance** | Higher (3x modules) | Medium (shared logic) | Medium |
| **Claude reasoning** | Easy | Medium | Easy |
| **Scalability** | Good | Excellent | Good |
| **Customer onboarding** | Straightforward | Confusing | Good |
| **Time to MVP** | Fast | Slower | Medium |
| **Future flexibility** | Can migrate to Terragrunt | Harder to split later | Already flexible |

**Recommendation**: Start with **Provider-Specific Variants** (MVP), graduate to **Terragrunt** if maintenance burden grows (Phase 2+).

---

## Open Questions

1. **SKU Mapping**: How much do instance size semantics vary per provider?
   - AWS: `db.t3.micro` vs Azure: `B_Gen5_1` — is `small` enough?
   - Or do we need cost parity calculations?

2. **Regional Differences**: Some services don't exist in all regions
   - Strategy: Validate region support in Phase 1?
   - Or fail gracefully in deployment plan?

3. **Cost Comparison**: Should Harness generate cost estimates per provider?
   - AWS: CloudFormation cost calculator
   - Azure: Pricing Calculator API
   - GCP: Pricing API

4. **Network Topology**: Are there provider-specific network patterns we should enforce?
   - VPC peering (AWS) vs VNet peering (Azure) vs VPC peering (GCP)?
   - Overlay with VPN standardization?

5. **Secrets Management**: How to abstract provider-specific secret stores?
   - AWS: Secrets Manager
   - Azure: Key Vault
   - GCP: Secret Manager
   - Standardized interface?

---

## References

- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [Terraform Azure Provider](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)
- [Terraform GCP Provider](https://registry.terraform.io/providers/hashicorp/google/latest/docs)
- [Terragrunt Documentation](https://terragrunt.gruntwork.io/)
