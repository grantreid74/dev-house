# PRD: [Product Name]

> **Template version**: 1.0.0
> **Status**: DRAFT | REVIEW | SIGNED-OFF
> **Change freeze**: [Date PRD was signed off — 1-month UAT clock starts here]
> **Customer ID**: [slug used in all generated artefacts]

---

## 1. Business Context

*Who is this customer? What do they do? Who are their customers?*

- **Company**: [Name, size, industry]
- **Their customers**: [End users — consumers, businesses, internal teams]
- **Current state**: [How do they solve this problem today — manual process, spreadsheet, legacy tool?]
- **Why now**: [What changed that makes this the right time to build it?]

---

## 2. Product Vision

*One paragraph. What are we building and why does it matter?*

[Write 3-5 sentences. Focus on the outcome the customer needs, not the technical solution. Example: "A web portal that lets clinic staff book specialist appointments on behalf of patients, reducing scheduling time from 48 hours to under 10 minutes."]

---

## 3. Success Criteria

*How will we know it worked? Measurable, not subjective.*

- [ ] [Metric 1 — e.g., "All core features deployable and passing smoke tests"]
- [ ] [Metric 2 — e.g., "System handles 100 concurrent users without degradation"]
- [ ] [Metric 3 — e.g., "Customer can log in, perform core workflow end-to-end in UAT"]
- [ ] [Metric 4 — e.g., "Terraform plan validates and applies without errors"]

---

## 4. User Types

*Who uses the system and roughly how many?*

| User Type | Description | Estimated Count | Access Level |
|-----------|-------------|-----------------|--------------|
| [e.g. Admin] | [What they do] | [N] | Full |
| [e.g. End User] | [What they do] | [N] | Standard |
| [e.g. Viewer] | [What they do] | [N] | Read-only |

**Peak concurrent users**: [estimate]
**Growth trajectory**: [stable / 2x in 12 months / 10x in 24 months]

---

## 5. Core Features

*The features this system must have. Write each as a discrete, independently testable capability — the Harness converts these into `feature_list.json` items.*

*Format: `[AREA] Feature description (acceptance criteria)`*

### [Area 1: e.g., Authentication]
- [ ] [AREA] Feature one (acceptance: user can log in with email/password)
- [ ] [AREA] Feature two (acceptance: password reset via email works end-to-end)

### [Area 2: e.g., Core Workflow]
- [ ] [AREA] Feature three (acceptance: ...)
- [ ] [AREA] Feature four (acceptance: ...)

### [Area 3: e.g., Admin]
- [ ] [AREA] Feature five (acceptance: ...)

### [Area 4: e.g., Notifications]
- [ ] [AREA] Feature six (acceptance: email notification sent within 60 seconds of trigger)

*Aim for 15-40 features. Too few = ambiguous scope. Too many = poorly decomposed — break into sub-features.*

---

## 6. Non-Functional Requirements

### Compliance & Security
- **Compliance mandates**: [None / SOC2 / HIPAA / PCI-DSS / GDPR / FedRAMP]
- **Data sensitivity**: [Public / Internal / PII / PHI / Payment data]
- **Audit trail required**: [Yes / No]
- **Data residency**: [No requirement / US only / EU only / Specific country]
- **Multi-tenancy**: [Single customer / Multi-tenant (customers must be isolated from each other)]

### Availability & Performance
- **Target uptime**: [Best effort / 99.5% / 99.9% / 99.99%]
- **Recovery time objective (RTO)**: [e.g., 4 hours / 1 hour / 15 minutes]
- **Recovery point objective (RPO)**: [e.g., 24 hours / 4 hours / 1 hour]
- **Response time**: [e.g., API p95 < 500ms]
- **Peak load**: [N requests/second at peak]

### Integration
- **Third-party integrations required**: [List services — e.g., Stripe, SendGrid, Slack, Salesforce]
- **Existing systems to integrate with**: [Any legacy systems the new system must talk to]
- **API requirements**: [REST only / GraphQL / Webhook support / gRPC]

---

## 7. Deployment Preferences

*Harness uses this section to select deployment tier and cloud provider.*

- **Cloud provider preference**: [AWS / Azure / GCP / No preference / Existing account — specify]
- **Infrastructure budget**: [$/month customer is willing to pay for their cloud costs]
- **Kubernetes preference**: [Yes — we have K8s expertise / No — prefer simpler / Indifferent]
- **Custom domain required**: [Yes — domain.com / No]
- **Existing infrastructure**: [Greenfield (nothing exists) / Brownfield — describe what exists]
- **Self-hosted option**: [Not required / Required — customer must run on-premise]
- **Timeline to UAT**: [N weeks from PRD sign-off]

---

## 8. Out of Scope

*Explicitly what we are NOT building. Prevents scope creep and Harness overreach.*

- [Feature/system explicitly excluded]
- [Feature/system explicitly excluded]
- [Integrations explicitly deferred to later phases]

---

## 9. Open Questions

*Unresolved decisions. Harness will flag these and halt for human input before generating code. Resolve before PRD sign-off where possible.*

| Question | Owner | Impact if Unresolved |
|----------|-------|----------------------|
| [Question] | [Customer / Dev-House] | [What breaks or gets guessed] |

---

## 10. Provenance

```
Template:  examples/prd-template.md v1.0.0
Customer:  [customer-id]
PRD owner: [name@company.com]
Signed:    [Date]
Reviewed:  [Dev-House reviewer name]
```
