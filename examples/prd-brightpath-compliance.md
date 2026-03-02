# PRD: BrightPath Compliance Platform

> **Template version**: 1.0.0
> **Status**: SIGNED-OFF
> **Change freeze**: 2026-03-10
> **Customer ID**: brightpath

---

## 1. Business Context

- **Company**: BrightPath — 12-person SaaS startup, Series A, 18 months post-launch
- **Their customers**: SMBs (50-200 employees) pursuing SOC2 Type II certification. Currently 40 active customers, growing to 100 by year-end.
- **Current state**: BrightPath delivers compliance guidance via Notion + Google Sheets. Evidence collection is a manual email/file sharing process. Auditors can't access live data — everything is exported and emailed. Customer churn is driven by this friction.
- **Why now**: BrightPath raised $2M seed to replace their manual workflow with a proper SaaS product. Investors expect a working product within 2 months.

---

## 2. Product Vision

A multi-tenant SaaS platform where BrightPath's customers (compliance managers at SMBs) can track SOC2 control status, collect and upload evidence, invite auditors with read-only access, and generate audit-ready reports. BrightPath admins can manage all tenants from a central console. The product replaces the current Notion/Sheets process end-to-end. Each customer sees only their own data, with a custom subdomain.

---

## 3. Success Criteria

- [ ] BrightPath admin can create a new customer tenant and have it live within 5 minutes
- [ ] Customer compliance manager can log in, see their controls, and upload evidence
- [ ] Auditor (read-only) can access a tenant via invite link without creating a BrightPath account
- [ ] Report generation produces a PDF in under 30 seconds for a tenant with 100 controls
- [ ] Platform handles 50 concurrent tenants without cross-tenant data leakage (verified by UAT test)
- [ ] SOC2 audit trail: every evidence upload, control status change, and user action is logged with timestamp and user ID

---

## 4. User Types

| User Type | Description | Estimated Count | Access Level |
|-----------|-------------|-----------------|--------------|
| BrightPath Admin | Manages all tenants; onboards new customers; views usage | 3 | Super-admin |
| Compliance Manager | Customer's internal user; manages controls and evidence for their org | 2 per tenant (80 total) | Tenant admin |
| Team Contributor | Customer's staff who upload evidence for specific controls | 5 per tenant (200 total) | Scoped write |
| Auditor | External read-only; accesses one tenant via invite | 1 per audit cycle | Read-only |

**Peak concurrent users**: 100 (across all tenants combined)
**Growth trajectory**: 40 → 100 tenants in 12 months; 100 → 300 in 24 months

---

## 5. Core Features

### Authentication & Tenancy
- [ ] [AUTH] Email/password login with email verification (acceptance: user receives verification email; can't log in until verified)
- [ ] [AUTH] Password reset flow (acceptance: reset email received within 2 minutes; link expires in 1 hour)
- [ ] [AUTH] Tenant isolation: users can only see their tenant's data (acceptance: UAT to verify cross-tenant query returns nothing)
- [ ] [AUTH] Auditor invite link (acceptance: auditor clicks link, sets password, sees read-only tenant view — no BrightPath account needed)
- [ ] [AUTH] Session timeout after 8 hours of inactivity (acceptance: user redirected to login)

### Controls Management
- [ ] [CONTROL] Import a SOC2 control framework (acceptance: admin can upload a CSV of controls; system creates all rows)
- [ ] [CONTROL] Set control status: Not Started / In Progress / Implemented / Needs Attention (acceptance: status change logged in audit trail)
- [ ] [CONTROL] Assign a control owner (any team member) (acceptance: owner sees assigned controls in their dashboard)
- [ ] [CONTROL] Add notes to a control (acceptance: notes timestamped and attributed to user)
- [ ] [CONTROL] Control due date with overdue highlighting (acceptance: overdue controls shown in red in dashboard)
- [ ] [CONTROL] Filter controls by: status, owner, category, due date (acceptance: filters combinable)
- [ ] [CONTROL] Control completion percentage dashboard (acceptance: correct % based on Implemented / total)

### Evidence Collection
- [ ] [EVIDENCE] Upload evidence file against a control (acceptance: file stored; downloadable by tenant members and auditors)
- [ ] [EVIDENCE] Evidence file types: PDF, PNG, JPG, CSV, DOCX, XLSX (max 25MB per file) (acceptance: unsupported type rejected with clear error)
- [ ] [EVIDENCE] Evidence list per control with uploader name, upload date, and filename (acceptance: list accurate)
- [ ] [EVIDENCE] Delete evidence file (compliance manager only) (acceptance: file removed; audit log entry created)
- [ ] [EVIDENCE] Evidence expiry date (acceptance: expired evidence flagged; owner notified by email 14 days before expiry)

### Reporting
- [ ] [REPORT] Generate SOC2 readiness report as PDF (acceptance: PDF includes all controls, statuses, evidence counts, and completion %)
- [ ] [REPORT] Timestamp and version number on every generated report (acceptance: two reports generated at different times have different version numbers)
- [ ] [REPORT] Email report to specified recipients (acceptance: PDF arrives as attachment within 5 minutes)

### BrightPath Admin Console
- [ ] [ADMIN] Create/deactivate customer tenant (acceptance: new tenant provisioned within 5 minutes; deactivated tenant inaccessible)
- [ ] [ADMIN] View all tenants: name, user count, control completion %, last active (acceptance: dashboard data < 5 seconds stale)
- [ ] [ADMIN] Impersonate tenant for support (acceptance: admin can view a tenant as a read-only observer; impersonation logged)
- [ ] [ADMIN] Usage report: active tenants, logins last 30 days, evidence uploads (acceptance: correct counts)

### Notifications
- [ ] [NOTIFY] Email to control owner when a control is overdue (acceptance: email sent on due date if status is not Implemented)
- [ ] [NOTIFY] Email to compliance manager when evidence expires in 14 days (acceptance: email sent on the 14-day mark)
- [ ] [NOTIFY] Weekly summary email to compliance manager: overdue controls, controls due this week (acceptance: sent every Monday 9am)

---

## 6. Non-Functional Requirements

### Compliance & Security
- **Compliance mandates**: SOC2 Type I (BrightPath must be SOC2-compliant themselves to sell to SOC2-seeking customers)
- **Data sensitivity**: PII (customer employee names, emails); business-sensitive (control and audit data)
- **Audit trail required**: Yes — every action logged with user ID, timestamp, and action type. 7-year retention.
- **Data residency**: US only
- **Multi-tenancy**: Strict — tenants must be fully isolated; cross-tenant query must be impossible at the infrastructure level, not just application logic

### Availability & Performance
- **Target uptime**: 99.9% (BrightPath's customers rely on this for active audit cycles)
- **RTO**: 1 hour
- **RPO**: 4 hours (evidence uploads are critical)
- **Response time**: API p95 < 800ms; PDF generation < 30 seconds
- **Peak load**: 100 concurrent users across all tenants

### Integration
- **Third-party integrations required**: SendGrid (email), PDF generation library (server-side)
- **Existing systems**: None — greenfield
- **API requirements**: REST API (BrightPath may build a Slack integration in Phase 2)

---

## 7. Deployment Preferences

- **Cloud provider preference**: No preference — Azure or AWS both acceptable
- **Infrastructure budget**: $150-200/month (BrightPath pays; passed to customers at margin)
- **Kubernetes preference**: No — prefer simplest isolated option; team has no K8s expertise
- **Custom domain required**: Yes — each tenant gets `[customer].brightpath.io`
- **Existing infrastructure**: Greenfield
- **Self-hosted option**: Not required
- **Timeline to UAT**: 5 weeks from PRD sign-off

### Expected pattern selection: Tier 3 (Isolated Infrastructure)

Reasoning: SOC2 compliance + strict tenant isolation + custom subdomains = Tier 3. Budget ($150-200/month) fits comfortably. Scale-to-zero useful for inactive tenants. Tier 3 is the recommended MVP pattern.

---

## 8. Out of Scope

- Mobile app (web only)
- HIPAA or PCI-DSS compliance (SOC2 only in this phase)
- SSO / SAML integration (deferred to Phase 2)
- Automated evidence collection via API (Phase 2 — manual upload only in Phase 1)
- In-app auditor commenting (auditor is read-only; feedback happens outside the platform)
- Billing and subscription management (BrightPath invoices manually)
- Multi-framework support beyond SOC2 Type II (ISO27001, NIST deferred)

---

## 9. Open Questions

| Question | Owner | Impact if Unresolved |
|----------|-------|----------------------|
| Should evidence files be stored per-tenant (isolated blob storage) or in a shared bucket with tenant-prefixed paths? | Dev-House recommendation | Affects Tier 3 isolation purity. Recommend: per-tenant storage for SOC2 cleanliness. |
| Should the auditor invite link expire? If so, after how long? | Customer | Harness will default to 30-day expiry if unresolved. |
| PDF generation: server-side (Puppeteer/WeasyPrint) or a third-party service (DocRaptor)? | Dev-House recommendation | Affects infrastructure complexity. Server-side preferred to avoid another vendor dependency. |

---

## 10. Provenance

```
Template:  examples/prd-template.md v1.0.0
Customer:  brightpath
PRD owner: cto@brightpath.io
Signed:    2026-03-10
Reviewed:  Dev-House
```

---

## Harness Notes

*Added by Harness Initializer — do not edit manually*

```yaml
# To be populated by Harness Session 1
pattern_selected: tier3-isolated
services_identified: []
feature_count: 0
harness_status: pending
```
