# Deployment Pattern Types for Dev-House

**Purpose**: Define architectural patterns for different customer profiles, and show how pattern selection drives cloud provider and Terraform component choices.

**Status**: Pattern Library (derived from CompylotAI-terraform)

---

## Pattern Selection Framework

### How to Choose a Pattern

```
Customer PRD Analysis
    ↓
Determine Customer Profile:
  - Scale: startup, SMB, enterprise
  - Compliance: none, PCI/HIPAA, SOC2, GDPR
  - Performance: best-effort, SLA-required, mission-critical
  - Customization: standard, moderate, extensive
  - Budget: development, standard, enterprise
    ↓
Select Deployment Pattern
  - Tier 1: Fully Shared (lowest cost)
  - Tier 2: Shared Compute (balanced)
  - Tier 3: Isolated Infrastructure (recommended)
  - Tier 4: Enterprise Isolated (maximum isolation)
  - Single-Tenant Cluster (customer-managed)
    ↓
Determine Cloud Provider
  - Pattern may require specific provider capabilities
  - Scale-to-zero requires serverless (Azure Container Apps, AWS Fargate)
  - Enterprise scale requires Kubernetes (AKS, EKS, GKE)
    ↓
Select Terraform Components
  - Each pattern has specific modules (vpc, database, cache, compute)
    ↓
Generate Infrastructure as Code
  - Claude generates provider-specific Terraform
```

---

## Pattern 1: Tier 1 (Fully Shared)

### Overview

All customers share the same infrastructure. Application code handles multi-tenancy routing and data isolation.

```
┌─────────────────────────────────────────┐
│         Shared Foundation               │
├─────────────────────────────────────────┤
│ ┌─────────────────────────────────────┐ │
│ │  Shared Container Apps Environment  │ │
│ │  ┌──────────┐ ┌──────────┐         │ │
│ │  │  API     │ │  Frontend│         │ │
│ │  │  Shared  │ │  Shared  │  (*)    │ │
│ │  └──────────┘ └──────────┘         │ │
│ │  *App routes based on domain       │ │
│ └─────────────────────────────────────┘ │
│                                         │
│ Shared Database:                        │
│ ├─ Schema per customer                 │
│ │  (public.{customer}_* tables)        │
│                                         │
│ Shared Cache:                           │
│ ├─ DB number per customer              │
│   (redis DB 0 = customer 1, etc.)      │
└─────────────────────────────────────────┘

┌────────┐  ┌────────┐  ┌────────┐
│Customer│  │Customer│  │Customer│
│  API   │  │  API   │  │  API   │
│domain1 │  │domain2 │  │domain3 │
└────────┘  └────────┘  └────────┘
     ↓            ↓           ↓
     └────────────┬───────────┘
                  ↓
           Same Container Apps
```

### Characteristics

| Aspect | Details |
|--------|---------|
| **Isolation** | Application-level only (no network isolation) |
| **Cost** | ✅ Lowest (~$200-500/month for foundation) |
| **Scale** | Shared resource contention if one customer spikes |
| **Compliance** | ❌ Difficult (shared data path = audit complexity) |
| **Customization** | Limited (all customers use same code) |
| **Best For** | Startups, free tiers, internal tools, MVPs |

### Use Case Example

```yaml
Customer: A startup
Size: 10-100 users
Budget: Minimal
Requirements: "Just need basic task management app"
Pattern: Tier 1 ✅
```

### Terraform Components Needed

```hcl
modules/
├── shared-vpc/
├── shared-database/         # Single PostgreSQL + schema per customer
├── shared-redis/            # Single Redis + DB number per customer
├── shared-container-apps-env/
├── shared-frontend/         # Hosts all customers
└── shared-backend/          # Hosts all customers
```

### Provisioning

```python
# Minimal provisioning
class Tier1CustomerProvisioner:
    def provision_customer(self, customer_id):
        # Create database schema
        self.create_database_schema(customer_id)

        # Create environment variables pointing to shared services
        self.store_env_config(customer_id, {
            'DATABASE_NAME': f'{customer_id}_db',
            'REDIS_DB': customer_id % 16,  # Redis has 16 DBs
        })

        # Done - application code handles routing
        return {
            'app_url': f'https://app.dev-house.io/{customer_id}',
            'database_id': f'{customer_id}_db',
        }
```

### Limitations

- 🔴 One customer's bug can affect all (shared code)
- 🔴 One customer's spike can starve others (shared resources)
- 🔴 Compliance audits complex (shared infrastructure)
- 🔴 Feature toggles needed for customization
- 🔴 Scale-to-zero not possible (always running)

---

## Pattern 2: Tier 2 (Shared Compute with Namespace Isolation)

### Overview

Multiple customers share a Kubernetes cluster (or Container Apps Environment), but each gets their own namespace with network policies enforcing isolation.

```
┌──────────────────────────────────────────────┐
│         Shared Kubernetes Cluster            │
│          (e.g., AKS, EKS, GKE)               │
├──────────────────────────────────────────────┤
│ ┌─────────────────┐ ┌─────────────────┐     │
│ │  Namespace:     │ │  Namespace:     │     │
│ │  customer-1     │ │  customer-2     │     │
│ │ ┌─────────────┐ │ │ ┌─────────────┐ │     │
│ │ │API Service  │ │ │ │API Service  │ │     │
│ │ │Deployment   │ │ │ │Deployment   │ │     │
│ │ │Secrets      │ │ │ │Secrets      │ │     │
│ │ │ConfigMaps   │ │ │ │ConfigMaps   │ │     │
│ │ └─────────────┘ │ │ └─────────────┘ │     │
│ │ (Network Policy) │ │ (Network Policy) │     │
│ └─────────────────┘ └─────────────────┘     │
│                                              │
│ Shared Foundation:                           │
│ ├─ Shared Ingress (with SNI routing)         │
│ ├─ Shared Database (schema per customer)    │
│ ├─ Shared Cache (DB/key per customer)       │
│ └─ Shared Volumes (path per customer)       │
└──────────────────────────────────────────────┘

┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ Customer 1  │  │ Customer 2  │  │ Customer 3  │
│ Namespace   │  │ Namespace   │  │ Namespace   │
└─────────────┘  └─────────────┘  └─────────────┘
```

### Characteristics

| Aspect | Details |
|--------|---------|
| **Isolation** | Namespace-level (network policies, RBAC) |
| **Cost** | ✅ Medium (~$500-1500/month) |
| **Scale** | Better (namespaces isolated from each other) |
| **Compliance** | ⚠️ Medium (network isolation helps, shared cluster complicates) |
| **Customization** | Moderate (can customize per-namespace) |
| **Best For** | SMBs, moderate scale, cost-conscious compliance |

### Use Case Example

```yaml
Customer: A SMB SaaS company
Size: 100-1000 users
Budget: $500-1000/month
Requirements: "Need isolation from other customers, but ok with shared cluster"
Pattern: Tier 2 ✅
```

### How It Works

1. **Namespace per customer**: `kubectl create namespace customer-acme`
2. **Network policies**: Prevent inter-namespace traffic
3. **RBAC**: Each customer's service account can't access others' resources
4. **Shared services**: Database (schema isolation), cache (DB/key isolation)
5. **Shared ingress**: SNI or hostname-based routing to correct namespace

### Terraform Components Needed

```hcl
modules/
├── kubernetes-cluster/         # AKS, EKS, or GKE
├── cluster-networking/         # Network policies
├── shared-database/            # PostgreSQL with schema per customer
├── shared-redis/               # Redis with DB per customer
├── shared-ingress-controller/
├── shared-storage/             # Persistent volumes
└── customer-namespace/         # Per-customer namespace template
    ├── deployment.tf
    ├── service.tf
    ├── configmap.tf
    └── secrets.tf
```

### Provisioning

```python
class Tier2CustomerProvisioner:
    def provision_customer(self, customer_id, cluster_kubeconfig):
        # Create namespace
        self.create_namespace(f'customer-{customer_id}', cluster_kubeconfig)

        # Create network policy (deny all ingress, allow from ingress controller)
        self.create_network_policy(customer_id, cluster_kubeconfig)

        # Create service account and RBAC
        self.create_rbac(customer_id, cluster_kubeconfig)

        # Create database schema
        self.create_database_schema(customer_id)

        # Create ConfigMap and Secrets
        self.create_configmap(customer_id, cluster_kubeconfig, {
            'DATABASE_SCHEMA': f'customer_{customer_id}',
            'REDIS_DB': customer_id % 16,
            'LOG_PREFIX': customer_id,
        })

        # Deploy application
        self.deploy_application(customer_id, cluster_kubeconfig)

        return {
            'namespace': f'customer-{customer_id}',
            'service_url': f'https://{customer_id}.dev-house.io',
            'database_schema': f'customer_{customer_id}',
        }
```

### Advantages

- ✅ Better isolation than Tier 1 (namespace enforcement)
- ✅ Easier compliance audits (network policies logged)
- ✅ Cost-effective scaling
- ✅ Per-namespace customization possible
- ✅ Good for multi-tenant SaaS

### Limitations

- 🟡 Cluster admin access could access customer data
- 🟡 Noisy neighbor possible (one customer's spike affects others)
- 🟡 Requires Kubernetes expertise
- 🟡 Still scale-to-zero not great (cluster always running)

### Migration Path

If customer outgrows Tier 2 → upgrade to Tier 3 (move to dedicated infrastructure)

---

## Pattern 3: Tier 3 (Isolated Infrastructure) — RECOMMENDED

### Overview

Each customer gets dedicated infrastructure (container apps environment or managed service). Shared Foundation layer (database, cache, identity, registry) reduces costs.

**This is the CompylotAI pattern.**

```
┌─────────────────────────────────────────────────┐
│            Shared Foundation Layer              │
├─────────────────────────────────────────────────┤
│ ┌────────────────────────────────────────────┐  │
│ │     PostgreSQL Flexible Server              │  │
│ │  (database per customer)                    │  │
│ └────────────────────────────────────────────┘  │
│                                                 │
│ ┌────────────────────────────────────────────┐  │
│ │     Redis Cache                            │  │
│ │  (DB/key per customer)                     │  │
│ └────────────────────────────────────────────┘  │
│                                                 │
│ ┌────────────────────────────────────────────┐  │
│ │     Container Registry (ACR)               │  │
│ │  (shared image storage)                    │  │
│ └────────────────────────────────────────────┘  │
│                                                 │
│ ┌────────────────────────────────────────────┐  │
│ │     Keycloak / Identity Provider           │  │
│ │  (realms per customer)                     │  │
│ └────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
                      ↑
          (VNet peering from customers)
                      ↓
┌──────────────────────┬──────────────────────┐
│                      │                      │
│   Customer A         │   Customer B         │
│   ┌────────────────┐ │ ┌────────────────┐   │
│   │ Dedicated ACA  │ │ │ Dedicated ACA  │   │
│   │ VNet           │ │ │ VNet           │   │
│   │ Resource Group │ │ │ Resource Group │   │
│   │ Storage        │ │ │ Storage        │   │
│   └────────────────┘ │ └────────────────┘   │
│                      │                      │
└──────────────────────┴──────────────────────┘
```

### Characteristics

| Aspect | Details |
|--------|---------|
| **Isolation** | ✅ Complete network + data isolation |
| **Cost** | ~$50-200/customer/month (dedicated ACA environment) |
| **Scale** | ✅ Each customer scales independently |
| **Compliance** | ✅ Excellent (full audit trail per customer) |
| **Customization** | ✅ Complete (customer-specific configs) |
| **Best For** | Mid-market, SMB SaaS, compliance-sensitive |

### Use Case Example

```yaml
Customer: A growing SaaS company
Size: 1000-10000 users
Budget: $100-300/month
Requirements: "Need compliance audit trail, custom domains, scale independently"
Pattern: Tier 3 ✅
```

### Architecture Layers

#### Layer 1: Foundation (One-Time)

```hcl
modules/
├── postgresql-database-foundation/
├── redis-cache-foundation/
├── container-registry/
├── identity-provider/          # Keycloak
├── key-vault/                  # Secrets
├── foundation-vnet/
├── dns-zone/                   # Custom domains
├── monitoring/                 # Log Analytics, App Insights
└── provisioning-tools/         # Python orchestrator
```

**Cost**: ~€50-100/month (foundation only, amortized across all customers)

#### Layer 2: Per-Customer

```hcl
modules/
├── customer-resource-group/
├── customer-vnet/              # Isolated VNet
├── customer-container-apps-env/
├── customer-container-app/     # API, Frontend, etc.
├── customer-storage/           # Blob storage
├── customer-domain/            # Custom domain setup
└── customer-secrets/           # Per-customer credentials
```

**Cost per customer**:
- Container Apps Environment: ~€20-50/month
- Storage: ~$1-10/month
- Bandwidth: ~$0.10-20/month
- **Total**: ~€25-80/month per customer

### Provisioning Flow

```python
class Tier3CustomerProvisioner:
    def provision_customer(self, customer_id, deployment_plan):
        # Step 1: Create resource group
        rg = self.create_resource_group(customer_id)

        # Step 2: Create VNet
        vnet = self.create_vnet(customer_id, rg)

        # Step 3: Create VNet peering to Foundation
        self.create_vnet_peering(customer_id, vnet, foundation_vnet)

        # Step 4: Create Container Apps Environment
        cae = self.create_container_apps_env(customer_id, rg, vnet)

        # Step 5: Create database
        self.create_database(customer_id, shared_postgresql_server)

        # Step 6: Create storage account
        storage = self.create_storage_account(customer_id, rg)

        # Step 7: Setup custom domain (if provided)
        if deployment_plan.get('custom_domain'):
            self.setup_custom_domain(customer_id, deployment_plan['custom_domain'])

        # Step 8: Deploy container apps
        self.deploy_container_apps(customer_id, cae, deployment_plan)

        return {
            'resource_group': f'rg-int-{customer_id}',
            'vnet_id': vnet.id,
            'cae_id': cae.id,
            'database_name': f'int_{customer_id}',
            'app_url': f'https://ca-int-{customer_id}-frontend.azurecontainerapps.io',
            'custom_domain': deployment_plan.get('custom_domain', None),
        }
```

### Key CompylotAI Implementation Details

**VNet CIDR Allocation**:
```
Foundation VNet: 10.100.0.0/16
Customer-N VNet: 10.{100 + 2*N}.0.0/23

Example:
- Customer 0: 10.100.0.0/23
- Customer 1: 10.102.0.0/23
- Customer 2: 10.104.0.0/23
- ...
- Customer 2048: 10.112.0.0/23 (max)
```

**Database Naming**:
```
Shared PostgreSQL Server: pg-compylotai-shared
Customer Databases:
- {env}_{customer} (main)
- {env}_{customer}_audit (audit logs)

Example:
- int_acme (main)
- int_acme_audit (logs)
```

**Keycloak Multi-Tenancy**:
```
Single Keycloak Instance with Realms:
- realm: acme
- realm: contoso
- realm: startup-xyz

Each realm has:
- Own users
- Own clients
- Own federation settings
```

**Scale-to-Zero** (Cost Optimization):
```
For inactive customers:
1. Set Container Apps replicas to 0
2. Cost drops to ~€0.10/month
3. Activate = scale back to 1+ replicas

Example: Suspended dev customer:
- Full running: €30/month
- Scale-to-zero: €0.10/month
- Savings: 99% cost reduction
```

### Advantages

- ✅ Complete isolation (network + data)
- ✅ Independent scaling per customer
- ✅ Scale-to-zero for cost optimization
- ✅ Excellent compliance (full audit trail)
- ✅ Custom domains per customer
- ✅ Straightforward to understand

### Limitations

- 🟡 Higher per-customer cost than Tier 1/2
- 🟡 More complex provisioning
- 🟡 Higher operational overhead
- 🟡 Still shared Foundation (database/cache) = some constraint

---

## Pattern 4: Tier 4 (Enterprise Isolated)

### Overview

Complete isolation: dedicated database, cache, and compute per customer. Tier 3 + dedicated data layer.

```
Customer A                    Customer B
┌───────────────────────┐    ┌───────────────────────┐
│ Dedicated PostgreSQL  │    │ Dedicated PostgreSQL  │
│ Dedicated Redis       │    │ Dedicated Redis       │
│ Dedicated ACA environment         │    │ Dedicated ACA environment         │
│ Dedicated VNet        │    │ Dedicated VNet        │
│ Resource Group        │    │ Resource Group        │
└───────────────────────┘    └───────────────────────┘

No shared Foundation (except optional: registry, identity provider)
```

### Characteristics

| Aspect | Details |
|--------|---------|
| **Isolation** | ✅✅ Maximum (complete separation) |
| **Cost** | ~$300-500+/customer/month |
| **Scale** | ✅ Unlimited per customer |
| **Compliance** | ✅✅✅ Perfect (zero shared data layer) |
| **Customization** | ✅✅ Unlimited |
| **Best For** | Enterprise, HIPAA/PCI compliance, very large customers |

### Use Case Example

```yaml
Customer: Enterprise healthcare company
Size: 10000+ users
Budget: $500-1000/month
Requirements: "HIPAA compliance, dedicated infrastructure, isolated database"
Pattern: Tier 4 ✅
```

### Terraform Components

```hcl
modules/
├── customer-postgresql-dedicated/
├── customer-redis-dedicated/
├── customer-container-apps-env/
├── customer-vnet/
├── customer-backup-vault/
├── customer-monitoring/
└── customer-network-security/
```

### Cost Breakdown (Example)

```
Dedicated PostgreSQL Flexible Server (large):  ~€200/month
Dedicated Redis (large):                       ~€100/month
Container Apps Environment + apps:             ~€50-100/month
Storage + bandwidth:                           ~$20-50/month
Backup + monitoring:                           ~$10-30/month
─────────────────────────────────────────────
Total:                                         ~€380-480/month
```

### Advantages

- ✅ Complete isolation
- ✅ HIPAA/PCI/SOC2 compliance easy
- ✅ Unlimited scaling
- ✅ Customer can migrate to own cloud account

### Limitations

- 🔴 Expensive (~$300-500+/customer)
- 🔴 Operational overhead (manage N dedicated databases)
- 🔴 No resource sharing = inefficient utilization

---

## Pattern 5: Single-Tenant Cluster (Bring Your Own Infrastructure)

### Overview

Customer brings their own cloud account and infrastructure. Dev-House provides Terraform modules and orchestration, customer manages (or we manage on their account).

```
Customer's AWS Account                Dev-House Modules
┌─────────────────────────────────────────────────────┐
│                                                      │
│  VPC, Subnets, Security Groups, EKS cluster        │
│  RDS PostgreSQL, ElastiCache Redis                 │
│  ECR registry, IAM roles, etc.                      │
│                                                      │
│  (Customer manages billing, compliance, access)     │
│                                                      │
└─────────────────────────────────────────────────────┘
```

### Characteristics

| Aspect | Details |
|--------|---------|
| **Isolation** | ✅✅✅ Complete (separate cloud account) |
| **Cost** | Variable (customer pays cloud provider) |
| **Scale** | ✅ Unlimited |
| **Compliance** | ✅✅✅ Perfect |
| **Customization** | ✅✅✅ Unlimited |
| **Best For** | Enterprise, regulated industries, existing cloud accounts |

### Use Case Example

```yaml
Customer: Fortune 500 enterprise
Size: 100000+ users
Budget: Unlimited
Requirements: "Own cloud account, custom VPC, compliance requirements, on-premises integration"
Pattern: Single-Tenant ✅
```

### How It Works

1. Customer creates AWS/Azure/GCP account
2. Dev-House provides pre-built Terraform modules
3. Customer (or Dev-House hired as consultant) runs Terraform in their account
4. Dev-House provides management API to query deployed infrastructure
5. Customer retains full control and billing

### Terraform Components

```hcl
# Dev-House provides these modules for customer to use:
modules/
├── customer-vpc/
├── customer-kubernetes-cluster/
├── customer-postgresql/
├── customer-redis/
├── customer-container-registry/
├── customer-monitoring/
├── customer-backup/
└── customer-security-groups/
```

### Advantages

- ✅ Customer has full control
- ✅ No vendor lock-in
- ✅ Can integrate with existing infrastructure
- ✅ Billing transparent (customer pays cloud provider)
- ✅ Compliance: customer's responsibility

### Limitations

- 🔴 High operational complexity for customer
- 🔴 Dev-House has limited visibility/support
- 🔴 Variable success (depends on customer's ops skills)
- 🔴 Not suited for SMBs

---

## Pattern Decision Matrix

Choose your pattern based on customer profile:

```
Customer Scale  │ Startup   │ SMB      │ Mid-Market │ Enterprise
─────────────────┼──────────┼──────────┼────────────┼──────────────
Users           │ <100     │ 100-1K   │ 1K-10K     │ 10K+
Budget          │ Minimal  │ $200-500 │ $500-2K    │ $2K+
─────────────────┼──────────┼──────────┼────────────┼──────────────
Pattern Options:│          │          │            │
─────────────────┼──────────┼──────────┼────────────┼──────────────
Tier 1          │ ✅       │ ⚠️        │ ❌         │ ❌
(Shared)        │ YES      │ Maybe    │ No         │ No
─────────────────┼──────────┼──────────┼────────────┼──────────────
Tier 2          │ ⚠️       │ ✅       │ ⚠️        │ ❌
(K8s Namespace) │ Maybe    │ YES      │ Maybe      │ No
─────────────────┼──────────┼──────────┼────────────┼──────────────
Tier 3          │ ❌       │ ✅       │ ✅         │ ⚠️
(Isolated)      │ No       │ YES      │ YES        │ Maybe
─────────────────┼──────────┼──────────┼────────────┼──────────────
Tier 4          │ ❌       │ ❌       │ ⚠️        │ ✅
(Enterprise)    │ No       │ No       │ Maybe      │ YES
─────────────────┼──────────┼──────────┼────────────┼──────────────
Single-Tenant   │ ❌       │ ❌       │ ⚠️        │ ✅
(BYOC)          │ No       │ No       │ Maybe      │ YES
─────────────────┴──────────┴──────────┴────────────┴──────────────

✅ = Recommended    ⚠️ = Consider    ❌ = Not typical
```

---

## Implementation Roadmap

### MVP: Start with Tier 3

**Why**:
- Good balance of cost and isolation
- CompylotAI has proven implementation
- Scales from SMB to mid-market
- Can migrate to Tier 4 later
- Can simplify to Tier 1/2 if needed

### Phase 1: Support Tier 2 (K8s Namespace)

- Implement shared Kubernetes cluster
- Add namespace provisioning
- Add network policies

### Phase 2: Add Tier 4 (Enterprise)

- Provision dedicated databases per customer
- Provision dedicated caches
- Implement cross-tier migration

### Phase 3: Support Single-Tenant (BYOC)

- Package Terraform modules for customer use
- Create provisioning API for multi-account management
- Add support for AWS/Azure/GCP BYOC

---

## Pattern Selection in Harness

The Harness should detect pattern from PRD and automatically:

```python
class PatternSelector:
    def select_pattern(self, prd_analysis) -> str:
        """Determine deployment pattern from PRD"""

        compliance_level = self.analyze_compliance(prd_analysis)
        customer_scale = self.estimate_scale(prd_analysis)
        budget = prd_analysis.get('budget_tier')
        isolation_required = prd_analysis.get('data_isolation_required')

        # Decision logic
        if compliance_level in ['HIPAA', 'PCI-DSS']:
            return 'Tier 4'  # Enterprise isolated

        if isolation_required and customer_scale > 1000:
            return 'Tier 3'  # Isolated infrastructure

        if budget == 'minimal' and customer_scale < 100:
            return 'Tier 1'  # Fully shared

        # Default for most customers
        return 'Tier 3'

    def get_terraform_components(self, pattern: str) -> List[str]:
        """Map pattern to required Terraform components"""

        patterns = {
            'Tier 1': [
                'shared-vpc',
                'shared-database',
                'shared-redis',
                'shared-container-apps',
            ],
            'Tier 2': [
                'kubernetes-cluster',
                'cluster-networking',
                'shared-database',
                'shared-redis',
                'customer-namespace',
            ],
            'Tier 3': [
                'postgresql-database-foundation',
                'redis-cache-foundation',
                'container-registry',
                'customer-vnet',
                'customer-container-apps-env',
                'customer-container-app',
            ],
            'Tier 4': [
                'customer-postgresql-dedicated',
                'customer-redis-dedicated',
                'customer-container-apps-env',
                'customer-vnet',
                'customer-backup-vault',
            ],
        }

        return patterns.get(pattern, [])
```

---

## Reusable Components (from CompylotAI)

### Extract These Modules

These CompylotAI modules are highly reusable for Dev-House:

```hcl
# Proven in production, can be extracted:

1. customer-provision-t3/              # Core Tier 3 provisioning module
   - Resource group creation
   - VNet creation with peering
   - Container Apps Environment setup
   - Storage account setup

2. postgresql/                         # Database module
   - Flexible Server configuration
   - Database-per-customer pattern
   - Backup/restore configuration
   - Private endpoint setup

3. redis/                              # Cache module
   - Redis cache with naming conventions
   - Database isolation patterns
   - Private endpoint setup

4. container-apps-env/                # Compute environment
   - Multi-replica configuration
   - Scale-to-zero setup
   - Environment variables management

5. keycloak/                           # Identity provider
   - Multi-realm setup
   - Custom domain support
   - Admin account management

6. foundation-networking/              # Foundation VNet
   - Shared VNet with peering
   - Private endpoint zones
   - DNS configuration

7. provisioning-tools/                 # Provisioning orchestrator
   - Python CLI framework
   - Terraform orchestration
   - State validation
   - RBAC for provisioning
```

### Adapt for Dev-House

These modules need provider-agnostic variants:

```
CompylotAI (Azure-only):
  modules/postgresql/main.tf
    → Uses: azurerm_postgresql_flexible_server

Dev-House (multi-provider):
  modules/postgresql/
  ├── aws/main.tf
  │   → Uses: aws_db_instance
  ├── azure/main.tf
  │   → Uses: azurerm_postgresql_flexible_server
  └── gcp/main.tf
      → Uses: google_sql_database_instance
```

---

## References

- **CompylotAI Pattern**: [CompylotAI-terraform/TENANCY_ARCHITECTURE.md](file:///home/grantr/proj/CompylotAI-terraform/docs/stateful/architecture/TENANCY_ARCHITECTURE.md)
- **CompylotAI Provisioning**: [CompylotAI-terraform/PROVISIONING_INFRASTRUCTURE_OVERVIEW.md](file:///home/grantr/proj/CompylotAI-terraform/docs/stateful/architecture/PROVISIONING_INFRASTRUCTURE_OVERVIEW.md)
- **K8s Namespace Isolation**: https://kubernetes.io/docs/concepts/security/network-policies/
- **Azure Container Apps**: https://learn.microsoft.com/en-us/azure/container-apps/
- **Container Apps Scaling**: https://learn.microsoft.com/en-us/azure/container-apps/scale-app

