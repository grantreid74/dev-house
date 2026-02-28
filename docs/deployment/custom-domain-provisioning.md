# Custom Domain Provisioning Pattern

**Purpose**: Document how customers can use custom domains instead of default cloud-provided URLs, including DNS setup, certificate provisioning, and Keycloak integration.

**Status**: Implementation pattern (from CompylotAI-terraform)

**Reference Implementation**: CompylotAI custom domain manager (`CustomDomainManager`)

---

## Overview

### Default vs Custom Domain

**Default Domain** (Cloud Provider):
```
https://ca-int-acme-frontend.thankfulbush-0447bc7e.westeurope.azurecontainerapps.io
```
- Automatically provisioned
- Free
- No customization
- Difficult for customers to remember

**Custom Domain** (Customer's Choice):
```
https://acme.dev-house.io  (subdomain on shared DNS zone)
OR
https://app.acme.com      (customer's own domain)
```
- Requires DNS configuration
- Requires SSL/TLS certificate
- Professional appearance
- Customer branding

---

## Architecture

```mermaid
graph TD
    A["Customer Requests<br/>Custom Domain"] -->|"Domain: acme.dev-house.io"| B["Provisioning Tool<br/>(Container App)"]

    B --> B1["1. Create DNS Records<br/>(CNAME + TXT validation)"]
    B1 --> B2["2. Bind Domain to<br/>Container App"]

    B2 --> B3["3. Azure Auto-Provisions<br/>Managed Certificate"]

    B3 --> B4["4. Poll Certificate Status<br/>(Wait for 'Succeeded')"]

    B4 --> B5{Success?}

    B5 -->|Yes| B6["5. Update Customer Config<br/>(FQDN, enable custom domain)"]
    B5 -->|Timeout| B7["Log Warning<br/>(customer can retry)"]

    B6 --> B8["✅ Customer Domain Ready"]

    style A fill:#ffebee
    style B fill:#fff3e0
    style B1 & B2 & B3 & B4 fill:#e3f2fd
    style B8 fill:#c8e6c9
```

---

## Implementation Details

### Step 1: Customer Provides Domain

PRD or API request specifies custom domain:

```yaml
customer:
  name: acme
  display_name: "ACME Corp"
  custom_domain: "acme.dev-house.io"   # OR "app.acme.com"
  custom_domain_type: "shared"          # "shared" or "customer_owned"
```

### Step 2: Create DNS Records

#### For Shared DNS Zone (`dev-house.io`)

```python
class DNSManager:
    def create_cname_record(self, customer_subdomain: str, container_fqdn: str):
        """Create CNAME record pointing to Container App"""
        # Record: acme.dev-house.io CNAME → ca-int-acme-frontend.azurecontainerapps.io
        az dns record-set cname create \
            --resource-group rg-compylotai-foundation \
            --zone-name dev-house.io \
            --name acme \
            --cname ca-int-acme-frontend.azurecontainerapps.io

    def create_txt_validation_record(self, customer_subdomain: str, validation_token: str):
        """Create TXT record for Azure domain validation"""
        # Record: asuid.acme.dev-house.io TXT → <validation-token>
        # This proves we own the dev-house.io zone
        az dns record-set txt create \
            --resource-group rg-compylotai-foundation \
            --zone-name dev-house.io \
            --name "asuid.{customer_subdomain}" \
            --text "{validation_token}"
```

#### For Customer-Owned Domain (`acme.com`)

Customer must create themselves:

```
CNAME Record:
Name:   app
Zone:   acme.com
Value:  ca-int-acme-frontend.azurecontainerapps.io
TTL:    3600 seconds

TXT Record (Validation):
Name:   asuid.app
Zone:   acme.com
Value:  <azure-validation-token>
TTL:    3600 seconds
```

### Step 3: Bind Custom Domain to Container App

```python
def bind_custom_domain(customer_name: str, container_app_name: str, fqdn: str):
    """
    Bind custom domain to Container App and trigger certificate provisioning.

    Azure automatically provisions a managed Let's Encrypt certificate after binding.
    """

    # Bind domain and enable SNI (Server Name Indication)
    result = az containerapp hostname bind \
        --name {container_app_name} \
        --resource-group {resource_group} \
        --hostname {fqdn} \
        --environment {container_env_id}

    return result
```

**Result**: Azure begins async certificate provisioning (5-15 minutes)

### Step 4: Poll Certificate Status

```python
class CertificateWaiter:
    def wait_for_certificate(
        self,
        fqdn: str,
        container_app_name: str,
        resource_group: str,
        timeout_seconds: int = 900,  # 15 minutes
        poll_interval: int = 30      # Check every 30 seconds
    ) -> bool:
        """
        Wait for Azure to provision managed Let's Encrypt certificate.

        Azure provisions certificate async after hostname binding.
        We poll the certificate list until status = 'Succeeded'.
        """

        start_time = time.time()

        while time.time() - start_time < timeout_seconds:
            elapsed = int(time.time() - start_time)

            # Check 1: Hostname binding status
            hostnames = az containerapp hostname list {container_app_name} {resource_group}
            for hostname in hostnames:
                if hostname['name'] == fqdn:
                    if hostname['bindingType'] == 'SniEnabled':
                        logger.info(f"✅ Certificate ready: {fqdn} ({elapsed}s)")
                        return True
                    elif hostname['bindingType'] == 'Disabled':
                        # Cert provisioning hasn't started - trigger it
                        az containerapp hostname bind ...

            # Check 2: Certificate state
            certs = az containerapp env certificate list {container_env} {resource_group}
            for cert in certs:
                if cert['subjectName'] == fqdn:
                    if cert['provisioningState'] == 'Succeeded':
                        logger.info(f"✅ Certificate provisioning succeeded")
                        return True
                    elif cert['provisioningState'] == 'Failed':
                        raise ProvisioningError(f"Certificate failed: {cert['failureReason']}")

            time.sleep(poll_interval)

        # Timeout - certificate still provisioning
        logger.warning(f"⚠️  Certificate timeout after {timeout_seconds}s")
        return False
```

**Key Insight**: Don't use the certificate state alone—use hybrid checking of both hostname binding status AND certificate state. This handles edge cases where Azure async operations get stuck.

### Step 5: Update Customer Configuration

Once certificate is ready, update customer metadata:

```python
def enable_custom_domain(customer_id: str, fqdn: str, container_app_id: str):
    """Update customer database with custom domain configuration"""

    # Store in metadata database
    UPDATE customers
    SET custom_domain = fqdn,
        custom_domain_enabled = true,
        container_app_id = container_app_id,
        certificate_provisioned_at = NOW()
    WHERE customer_id = customer_id;

    # Update environment config for container apps
    # so they know what FQDN they're running on
    az containerapp update \
        --name ca-int-{customer}-frontend \
        --set-env-vars CUSTOMER_DOMAIN={fqdn}
```

---

## Identity Provider Integration

### Keycloak Realm Setup (Not Auth0)

CompylotAI uses **Keycloak** (self-hosted identity provider) with per-tenant realms:

```python
class KeycloakService:
    def create_customer_realm(self, customer_id: str, custom_domain: str):
        """
        Create Keycloak realm for customer with custom domain.

        Each customer gets their own Keycloak realm for:
        - User management
        - OAuth2 / OpenID Connect endpoints
        - Custom branding
        """

        # Create realm
        realm_name = f"realm-{customer_id}"

        keycloak_client.realms.create({
            'realm': realm_name,
            'displayName': f'{customer_id} Realm',
            'enabled': True,
        })

        # Create OAuth2 client for the custom domain
        keycloak_client.clients.create(realm_name, {
            'clientId': f'client-{customer_id}',
            'name': 'Web Application',
            'redirectUris': [
                f'https://{custom_domain}/callback',
                f'https://{custom_domain}/auth/callback',
            ],
            'webOrigins': [
                f'https://{custom_domain}',
            ],
            'publicClient': True,
        })

        return {
            'realm_name': realm_name,
            'auth_endpoint': f'https://auth-int.dev-house.io/realms/{realm_name}/protocol/openid-connect/auth',
            'token_endpoint': f'https://auth-int.dev-house.io/realms/{realm_name}/protocol/openid-connect/token',
        }
```

### Custom Domain in Keycloak

```
Keycloak Instance: auth-int.dev-house.io (shared, single instance)
                   ↓
            Realm per Customer:
            ├─ realm-acme
            ├─ realm-contoso
            └─ realm-startup-xyz

Each realm has OAuth2 clients configured with customer's custom domain:
realm-acme:
  client: acme-web-app
  redirect_uris: https://acme.dev-house.io/callback
  web_origins: https://acme.dev-house.io
```

---

## Operational Challenges & Solutions

### Challenge 1: Azure Async Delays

**Problem**: Certificate provisioning can take 5-15 minutes, and it's async (no wait/polling API).

**Solution** (CompylotAI):
```python
# Don't wait indefinitely - log and continue
wait_for_certificate(
    fqdn=fqdn,
    timeout_seconds=900,  # 15 min
    poll_interval=30      # Check every 30s
)

# If timeout:
#   - Log warning (not error)
#   - Customer can access service via default URL immediately
#   - Custom domain will work within 15 min
#   - No blocking required
```

### Challenge 2: Cleanup on Deprovisioning

**Problem**: Leaving DNS records + certificates after customer deprovisioned creates orphaned resources.

**Solution**:
```python
def cleanup_custom_domain(customer_id: str, customer_subdomain: str, fqdn: str):
    """Remove all custom domain artifacts"""

    # 1. Delete hostname binding (triggers cert cleanup cascade)
    az containerapp hostname delete \
        --name ca-int-{customer}-frontend \
        --resource-group {rg} \
        --hostname {fqdn}

    # 2. Wait for certificate cleanup (Azure async)
    wait_for_certificate_cleanup(fqdn, timeout=300)

    # 3. Delete DNS records
    az dns record-set cname delete \
        --resource-group rg-foundation \
        --zone-name dev-house.io \
        --name {customer_subdomain}

    az dns record-set txt delete \
        --resource-group rg-foundation \
        --zone-name dev-house.io \
        --name asuid.{customer_subdomain}
```

### Challenge 3: Re-provisioning Same Customer

**Problem**: If customer is deprovisioned and reactivated, we might encounter pending-delete states.

**Solution**:
```python
def wait_for_pending_delete_window(fqdn: str, timeout_seconds: int = 300):
    """
    Azure has a ~5-minute pending-delete window before resources can be re-created.

    If reactivating customer too quickly, domain binding fails with "pending deletion".
    Solution: Wait for pending-delete to expire.
    """

    start_time = time.time()
    while time.time() - start_time < timeout_seconds:
        try:
            # Try to bind - if it fails with "pending deletion", wait
            result = az containerapp hostname bind ...
            if "pending delete" in result.stderr.lower():
                logger.info("Waiting for pending-delete window to expire...")
                time.sleep(10)
                continue
            else:
                return True
        except Exception as e:
            if "pending delete" in str(e).lower():
                logger.info("Still in pending-delete window...")
                time.sleep(10)
            else:
                raise

    raise ProvisioningError(f"Timeout waiting for pending-delete window to expire")
```

---

## Provisioning API Architecture

### Current State: CLI-Based

CompylotAI uses CLI tool running inside Container Apps (provisioning-tools):

```bash
# Current: Must have Keycloak authenticated user with VPN + kubectl access
kubectl exec -it pod/provisioning-tools -- /app/bin/compylot provision \
    --customer-name acme \
    --environment int \
    --enable-custom-domain
```

**Problems**:
- 🔴 Requires VPN access to cluster
- 🔴 Requires kubectl configured locally
- 🔴 CLI is synchronous (no progress updates)
- 🔴 No audit trail of who triggered what
- 🔴 Hard to integrate with web UI

### Proposed: REST API

Wrap provisioning tool with REST API that:
1. Runs inside VNET (no public exposure)
2. Handles long-running operations (async jobs)
3. Provides audit trail
4. Integrates with web dashboard

```
┌──────────────────────────────────┐
│   Dev-House Web Dashboard        │
│  (Customer-facing UI)            │
└────────────┬─────────────────────┘
             │ HTTPS
             ↓
┌──────────────────────────────────────────┐
│   Dev-House VNET (Secured)               │
├──────────────────────────────────────────┤
│                                          │
│  ┌──────────────────────────────┐       │
│  │  Provisioning API (FastAPI)  │       │
│  │  Listening on 0.0.0.0:8000   │       │
│  └──────┬───────────────────────┘       │
│         │                                │
│         ├─→ Authenticates with Keycloak │
│         │                                │
│         ├─→ Queues job in Redis        │
│         │                                │
│         ├─→ Worker picks up job        │
│         │                                │
│         ├─→ Calls provision_customer.py │
│         │                                │
│         └─→ Returns job_id + status     │
│                                          │
│  ┌──────────────────────────────┐       │
│  │  Job Queue (Redis)           │       │
│  └──────────────────────────────┘       │
│                                          │
│  ┌──────────────────────────────┐       │
│  │  Provisioning Workers (N)    │       │
│  │  (Running provision_*.py)    │       │
│  └──────────────────────────────┘       │
│                                          │
└──────────────────────────────────────────┘
```

### REST API Endpoints

```
POST /api/v1/customers/{customer_id}/provision
{
    "action": "provision",
    "display_name": "ACME Corp",
    "custom_domain": "acme.dev-house.io",
    "profile": "standard"
}
Response:
{
    "job_id": "job-uuid-123",
    "status": "queued",
    "customer_id": "acme",
    "action": "provision"
}

---

GET /api/v1/jobs/{job_id}
Response:
{
    "job_id": "job-uuid-123",
    "status": "in_progress",
    "step": "waiting_for_certificate",
    "progress": {
        "current_step": 5,
        "total_steps": 7,
        "elapsed_seconds": 180
    },
    "logs": [
        "Created resource group: rg-int-acme",
        "Created VNet: vnet-int-acme",
        "Created Container Apps Environment",
        "Created database schema: int_acme",
        "Binding custom domain: acme.dev-house.io",
        "Waiting for certificate provisioning... (120s elapsed)"
    ]
}

---

GET /api/v1/customers/{customer_id}
Response:
{
    "customer_id": "acme",
    "display_name": "ACME Corp",
    "status": "active",
    "app_url": "https://acme.dev-house.io",
    "default_url": "https://ca-int-acme-frontend.azurecontainerapps.io",
    "provisioning_state": {
        "resource_group": "ready",
        "vnet": "ready",
        "database": "ready",
        "certificate": "ready",
        "custom_domain": "ready"
    }
}

---

POST /api/v1/customers/{customer_id}/deprovision
{
    "action": "deprovision",
    "mode": "soft"    # soft | backup | destroy-all
}
Response:
{
    "job_id": "job-uuid-456",
    "status": "queued",
    "action": "deprovision"
}
```

### Provisioning API Implementation (FastAPI)

```python
from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel
import uuid
import redis
import json
from datetime import datetime

app = FastAPI()
redis_client = redis.Redis(host='localhost', port=6379)

class ProvisionRequest(BaseModel):
    action: str  # "provision" | "deprovision"
    display_name: Optional[str] = None
    custom_domain: Optional[str] = None
    profile: str = "standard"
    mode: Optional[str] = None  # For deprovision

@app.post("/api/v1/customers/{customer_id}/provision")
async def provision_customer(
    customer_id: str,
    request: ProvisionRequest,
    user = Depends(get_authenticated_user)  # Keycloak auth
):
    """Queue a customer provisioning job"""

    job_id = str(uuid.uuid4())

    # Store job in Redis
    job_data = {
        "job_id": job_id,
        "customer_id": customer_id,
        "action": request.action,
        "status": "queued",
        "created_at": datetime.utcnow().isoformat(),
        "created_by": user['sub'],  # User ID from Keycloak
        "request": request.dict(),
        "logs": [],
    }

    redis_client.set(f"job:{job_id}", json.dumps(job_data, default=str))
    redis_client.lpush("job_queue", job_id)  # Add to queue

    # Audit log
    audit_log(user['sub'], "provision_queued", customer_id, job_id)

    return {
        "job_id": job_id,
        "status": "queued",
        "customer_id": customer_id,
        "action": request.action,
    }

@app.get("/api/v1/jobs/{job_id}")
async def get_job_status(job_id: str, user = Depends(get_authenticated_user)):
    """Get job status and logs"""

    job_data = redis_client.get(f"job:{job_id}")
    if not job_data:
        raise HTTPException(status_code=404, detail="Job not found")

    return json.loads(job_data)

# Worker process (separate from API)
def job_worker():
    """Background worker that processes provisioning jobs"""

    while True:
        job_id = redis_client.rpop("job_queue")
        if not job_id:
            time.sleep(1)
            continue

        job_data = json.loads(redis_client.get(f"job:{job_id}"))

        try:
            job_data['status'] = 'in_progress'
            redis_client.set(f"job:{job_id}", json.dumps(job_data, default=str))

            # Execute provisioning
            if job_data['action'] == 'provision':
                result = provision_customer_via_terraform(
                    job_id,
                    job_data['customer_id'],
                    job_data['request']
                )
            elif job_data['action'] == 'deprovision':
                result = deprovision_customer_via_terraform(...)

            job_data['status'] = 'completed'
            job_data['result'] = result

        except Exception as e:
            job_data['status'] = 'failed'
            job_data['error'] = str(e)

        finally:
            redis_client.set(f"job:{job_id}", json.dumps(job_data, default=str))
```

### Security Considerations

1. **Keycloak Authentication**: All API requests require valid JWT from Keycloak
2. **VNET Isolation**: API runs inside VNET, not exposed to internet
3. **Audit Logging**: Every API call logged with user ID, timestamp, action
4. **Rate Limiting**: Limit provisioning jobs to prevent DoS (e.g., 10/min per user)
5. **RBAC**: Different scopes for admin vs customer (e.g., admin can provision, customer can only view)

---

## Implementation Roadmap

### MVP: CLI-Based (Current)

- ✅ Provisioning tool as Container App
- ✅ Custom domain support with Keycloak realms
- ✅ Manual execution via kubectl

### Phase 1: REST API

- Add FastAPI wrapper around provisioning tool
- Add Redis job queue for async operations
- Add Keycloak authentication to API
- Dashboard can call API instead of CLI

### Phase 2: Self-Service UI

- Customer-facing dashboard for domain management
- Real-time provisioning progress
- Certificate renewal automation
- DNS record validation helpers

### Phase 3: Advanced Patterns

- Multi-domain per customer (dev/staging/prod)
- Wildcard certificate support
- Domain migration (customer brings existing domain)
- Let's Encrypt renewal automation

---

## References

- **CompylotAI Custom Domain Manager**: `/docker/provisioning-tools/src/custom_domain_manager.py`
- **CompylotAI Provisioning Script**: `/docker/provisioning-tools/scripts/t3/provision_customer.py`
- **CompylotAI Keycloak Integration**: `/docker/provisioning-tools/src/keycloak_service.py`
- **Azure Container Apps Custom Domains**: https://learn.microsoft.com/en-us/azure/container-apps/custom-domains-certificates
- **Let's Encrypt Managed Certificates**: https://letsencrypt.org/

