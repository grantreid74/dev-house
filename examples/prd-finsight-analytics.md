# PRD: FinSight Transaction Analytics Module

> **Template version**: 1.0.0
> **Status**: REVIEW — 2 open questions require customer input before sign-off
> **Change freeze**: Pending sign-off
> **Customer ID**: finsight

---

## 1. Business Context

- **Company**: FinSight — 500-person fintech company, Series C, publicly traded on AIM
- **Their customers**: Institutional investors and corporate treasury departments using FinSight's existing trading platform
- **Current state**: Transaction data flows into an existing AWS data warehouse (Redshift). Analysts run ad-hoc SQL queries to detect anomalies. No real-time alerting. When something looks wrong, detection is hours or days after the fact. Compliance team manually reviews exports.
- **Why now**: A near-miss regulatory incident 3 months ago. Compliance missed a suspicious transaction pattern for 72 hours. The CEO committed to the board that real-time anomaly detection would be live within 2 quarters.

---

## 2. Product Vision

A real-time transaction monitoring module that plugs into FinSight's existing AWS data infrastructure. Analysts and compliance officers see a live dashboard of flagged transactions. Automated alerts notify the right team within 2 minutes of a detected anomaly. The module is governed by the same PCI-DSS and SOC2 standards as the rest of FinSight's platform. It integrates with their existing VPC, RDS, and identity provider — it does not rebuild what already works.

---

## 3. Success Criteria

- [ ] Anomaly alerts generated and notification sent within 2 minutes of a flagged transaction
- [ ] Dashboard shows live transaction feed (< 30-second lag from Redshift)
- [ ] Compliance officer can investigate a flagged transaction and export a complete audit trail as PDF
- [ ] System handles 10,000 transactions/hour without alert latency degrading beyond 2-minute SLA
- [ ] Zero cross-customer data leakage (FinSight is multi-client; each institutional client's data must be isolated)
- [ ] Deploys into FinSight's existing AWS VPC without requiring new account or VPC — verified by Terraform plan review

---

## 4. User Types

| User Type | Description | Estimated Count | Access Level |
|-----------|-------------|-----------------|--------------|
| Compliance Officer | Reviews flagged transactions; writes investigation notes; exports reports | 12 | Compliance view |
| Analyst | Monitors live feed; tunes detection rules; views dashboards | 8 | Analyst |
| Client Manager | Views flagged transactions for their assigned institutional clients only | 30 | Scoped read-only |
| FinSight Admin | Manages user roles; configures alert thresholds; onboards new client portfolios | 3 | Super-admin |

**Peak concurrent users**: 50
**Growth trajectory**: Stable (internal tool; not customer-facing)

---

## 5. Core Features

### Authentication & Authorization
- [ ] [AUTH] SSO via FinSight's existing Okta tenant (SAML 2.0) (acceptance: existing Okta credentials work; no new password to set)
- [ ] [AUTH] Role-based access: Compliance Officer, Analyst, Client Manager, Admin (acceptance: roles enforced at API layer; not just UI)
- [ ] [AUTH] Client Manager scoping: Client Manager sees only their assigned client portfolios (acceptance: query with client ID outside scope returns 403)
- [ ] [AUTH] Session expiry: 4 hours (PCI-DSS requirement) (acceptance: session invalidated at 4 hours; token refresh not automatic)
- [ ] [AUTH] All auth events logged to existing SIEM (acceptance: login, logout, failed auth events visible in Splunk within 60 seconds)

### Transaction Monitoring
- [ ] [MONITOR] Live transaction feed: display transactions from Redshift with < 30-second lag (acceptance: timestamp visible; lag shown on dashboard)
- [ ] [MONITOR] Colour-coded risk scoring per transaction: Green (clean) / Amber (review) / Red (alert) (acceptance: scoring consistent with configured rule thresholds)
- [ ] [MONITOR] Filter feed by: client portfolio, transaction type, risk score, amount range, date range (acceptance: filters apply server-side; not just UI filter)
- [ ] [MONITOR] Transaction detail view: all available fields, related transactions in same session, raw event payload (acceptance: all fields visible; no truncation)
- [ ] [MONITOR] Pagination and infinite scroll on transaction feed (acceptance: 1,000+ transactions loadable without browser crash)

### Anomaly Detection Rules
- [ ] [RULES] Pre-configured rule library: velocity breach, unusual amount, geographic anomaly, counterparty mismatch (acceptance: all 4 rules active after deployment)
- [ ] [RULES] Analyst can adjust rule thresholds (e.g., velocity breach at > 50 transactions/hour vs > 100) (acceptance: threshold change takes effect within 60 seconds; change logged)
- [ ] [RULES] Analyst can disable/enable individual rules (acceptance: disabled rule produces no alerts; re-enabling takes effect within 60 seconds)
- [ ] [RULES] Rule trigger history: which rule fired on which transaction (acceptance: queryable for the past 90 days)

### Alerting
- [ ] [ALERT] Real-time alert when a transaction hits Red risk score: notification via email + Slack (acceptance: both notifications arrive within 2 minutes of the triggering transaction reaching Redshift)
- [ ] [ALERT] Alert routing: alerts go to the Compliance Officer assigned to the triggering client portfolio (acceptance: correct officer notified based on portfolio assignment)
- [ ] [ALERT] Alert acknowledgement: Compliance Officer marks alert as: Acknowledged / False Positive / Escalated (acceptance: status change logged; escalated alerts cc the Head of Compliance)
- [ ] [ALERT] Unacknowledged alert escalation: if not acknowledged within 30 minutes, auto-escalate to Head of Compliance (acceptance: escalation email sent at the 30-minute mark; escalation logged)
- [ ] [ALERT] Alert summary email: daily report of alert counts, acknowledgement rate, false positive rate (acceptance: sent at 7am GMT each weekday)

### Investigation & Reporting
- [ ] [INVEST] Compliance Officer can open an investigation on a flagged transaction: add notes, attach files, link related transactions (acceptance: investigation persists; attachments stored)
- [ ] [INVEST] Investigation status workflow: Open / Under Review / Closed — Requires Justification (acceptance: status transitions logged with officer ID and timestamp)
- [ ] [INVEST] Export investigation as PDF: includes transaction detail, all notes, all linked transactions, rule triggers, audit trail (acceptance: PDF generated < 30 seconds)
- [ ] [INVEST] Search past investigations by: date range, client portfolio, rule type, officer, status (acceptance: results returned < 3 seconds)

### Admin
- [ ] [ADMIN] Onboard new client portfolio: link to existing Redshift data source, assign Client Managers and Compliance Officers (acceptance: new portfolio monitored within 5 minutes)
- [ ] [ADMIN] User management: invite user, assign role, deactivate user (acceptance: deactivated user's session invalidated within 60 seconds)
- [ ] [ADMIN] System health dashboard: Redshift lag, alert latency p95, alert volume last 24h, false positive rate (acceptance: data < 60 seconds stale)

---

## 6. Non-Functional Requirements

### Compliance & Security
- **Compliance mandates**: PCI-DSS Level 1; SOC2 Type II (FinSight is already certified — new module must not create gaps)
- **Data sensitivity**: Payment card data (PCI scope); transaction records treated as financial secrets
- **Audit trail required**: Yes — 7-year retention. Every user action, every data access, every alert event logged. Logs must be immutable (append-only, tamper-evident).
- **Data residency**: AWS eu-west-2 only (existing FinSight constraint). No cross-region replication.
- **Multi-tenancy**: Per-institutional-client isolation. One client's transaction data must not be accessible to users scoped to another client.

### Availability & Performance
- **Target uptime**: 99.99% (financial compliance — downtime is a regulatory incident)
- **RTO**: 15 minutes
- **RPO**: 0 (no data loss acceptable — event-driven architecture; replay from Redshift on failure)
- **Response time**: Dashboard load < 2 seconds; alert delivery < 2 minutes; investigation export < 30 seconds
- **Peak load**: 10,000 transactions/hour sustained; burst to 50,000/hour for end-of-month batch processing

### Integration
- **Third-party integrations required**: Okta (SAML SSO), Slack (webhook alert delivery), Splunk (SIEM log forwarding)
- **Existing systems to integrate with**: AWS Redshift (data source — existing VPC, read-only access provided); FinSight's existing RDS instance (user/portfolio data — read-only access provided)
- **API requirements**: REST API (for future integration with FinSight's trading platform). Webhook outbound for alert events.

---

## 7. Deployment Preferences

- **Cloud provider preference**: AWS eu-west-2 ONLY (existing account — BYOC model)
- **Infrastructure budget**: FinSight pays their own AWS bill — no additional budget constraint beyond cost justification to CTO
- **Kubernetes preference**: Existing EKS cluster in use. Prefer deploying to existing cluster (new namespace) rather than new compute.
- **Custom domain required**: Internal only — `analytics.internal.finsight.com` (private Route 53 zone; not public-facing)
- **Existing infrastructure**: Significant brownfield:
  - AWS account: `finsight-prod` (eu-west-2)
  - VPC: `vpc-0a1b2c3d4e` (existing; do not create new VPC)
  - EKS: `finsight-eks-prod` (v1.32; existing; deploy as new namespace `finsight-analytics`)
  - RDS: `finsight-pg-prod` (existing; add new schemas, do not create new instance)
  - Redshift: `finsight-dw-prod` (existing; read-only access; no modifications)
  - Okta tenant: `finsight.okta.com` (existing; Dev-House given SAML metadata)
  - Secrets Manager: `finsight-prod-secrets` (existing; module should read from here, not create separate store)
- **Self-hosted option**: Yes — BYOC. FinSight operates everything. Dev-House provides Terraform and hands over.
- **Timeline to UAT**: 6 weeks from PRD sign-off

### Expected pattern selection: BYOC / Single-Tenant (Tier 2 internal — existing EKS namespace)

Reasoning: FinSight has a mature AWS environment. They are not a new cloud customer — they have VPC, EKS, RDS, Redshift, Okta already running. The correct pattern is BYOC: provide Terraform that deploys into their existing account without creating parallel infrastructure. K8s namespace isolation within their existing EKS cluster. This is the most constrained and complex PRD — the Harness must generate Terraform that references existing resource IDs rather than creating everything from scratch.

*Critical Harness note*: All Terraform `data` sources must reference existing resource IDs supplied by FinSight. The Harness should produce a `FINSIGHT_INFRA_VARS.yaml` in Session 1 listing every variable that must be supplied by FinSight before Terraform can be applied.

---

## 8. Out of Scope

- Rebuilding or migrating the existing Redshift data warehouse
- Replacing or modifying the existing FinSight trading platform
- Creating new AWS account, VPC, or EKS cluster (use existing)
- Mobile app
- Customer-facing features (this is an internal compliance tool only)
- ML-based anomaly detection training pipeline (rule-based only in Phase 1)
- Data archiving or cold storage management (Redshift manages this already)
- Billing or subscription management

---

## 9. Open Questions

| Question | Owner | Impact if Unresolved |
|----------|-------|----------------------|
| Which Redshift tables contain transaction data, and what is the schema? Can Dev-House have read-only Redshift access during development? | Customer (must provide before code generation begins) | Harness cannot generate correct queries without schema. **Blocks Session 1.** |
| PCI-DSS scope: does this module ever see raw card PANs, or only masked values? | Customer + Compliance | If raw PANs: significant additional PCI controls required. If masked: simpler. **Blocks Session 1 if unresolved.** |
| Which Slack workspace and channel for alerts? Dev-House needs webhook URL. | Customer (easy — can provide day of) | Defaults to no Slack integration; email-only alerts until webhook provided |
| Should investigation notes be encrypted at rest, or is access control sufficient for PCI? | Customer + Legal | Affects schema design; Harness defaults to encrypted if unresolved |

---

## 10. Provenance

```
Template:  examples/prd-template.md v1.0.0
Customer:  finsight
PRD owner: cto@finsight.com
Signed:    Pending (2 open questions block sign-off — see Section 9)
Reviewed:  Dev-House
```

---

## Harness Notes

*Added by Harness Initializer — do not edit manually*

```yaml
# To be populated by Harness Session 1
pattern_selected: byoc-eks-namespace
services_identified: []
feature_count: 0
harness_status: blocked  # 2 open questions must be resolved before Session 1 proceeds
blocking_questions:
  - redshift_schema_access
  - pci_pan_scope
```
