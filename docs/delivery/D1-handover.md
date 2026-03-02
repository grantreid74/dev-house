# D1 — Handover Delivery Model

**Status**: Current default

Dev-House builds the system, provisions it in the customer's cloud account, and hands it over. The customer operates it.

---

## What the Customer Receives

| Deliverable | Description |
|-------------|-------------|
| **Code** | All services committed to customer's git repos (frontend, backend, infrastructure) |
| **Provisioned infrastructure** | Terraform applied into customer's cloud account — running, not just planned |
| **Runbooks** | How to operate, restart, scale, and monitor each component |
| **Handover session** | Walkthrough with customer team: credentials, access, operations basics |
| **30-day support window** | Questions, setup issues, minor fixes — not ongoing managed ops |

**Definition of done**: System is live in the customer's cloud account and their team can operate it independently.

---

## What Dev-House Does Not Provide (by Default)

- Ongoing infrastructure monitoring
- Incident response / on-call
- Scaling decisions or capacity planning
- Day-to-day operational management

These are possible as a separately-scoped retainer engagement. They are not included in the base delivery.

---

## Ongoing Dev-House Involvement (Optional)

After handover, the customer may engage Dev-House for:

| Engagement | What It Covers | What It Does Not Cover |
|------------|----------------|------------------------|
| **Dev support retainer** | New features, PRD iterations, bug fixes, refactoring | Infrastructure operations, on-call |
| **Ops retainer** | Infrastructure monitoring, incident response, scaling | New feature development |

These are separate commercial agreements, negotiated per customer.

---

## Handover Checklist

Before closing a D1 engagement:

- [ ] All code committed to customer's repos
- [ ] Terraform state stored in customer's account (not Dev-House's)
- [ ] All credentials transferred to customer (API keys, cloud access)
- [ ] Dev-House credentials revoked from customer infrastructure
- [ ] Runbooks reviewed with customer team
- [ ] Customer team can independently: restart services, view logs, run `terraform plan`
- [ ] 30-day support window communicated and start date agreed

---

## Pricing Model (Reference)

| Component | Cost to Customer |
|-----------|-----------------|
| Discovery + PRD | Fixed fee (time-boxed) |
| Generation + build | Fixed fee per PRD |
| Provision + handover | Included in build fee |
| Infrastructure (post-handover) | Customer's cloud bill — their account, their cost |
| Dev support retainer | Monthly, scoped separately |

*Exact pricing TBD — see CURRENT_REVIEW.md for open commercial questions.*

---

## See Also

- **[docs/deployment/customer-deployment.md](../deployment/customer-deployment.md)** — Technical setup: how the customer runs what we've built
- **[docs/deployment/deployment-patterns.md](../deployment/deployment-patterns.md)** — What the customer's system looks like (D-T3, D-T4, D-BYOC)
- **[CURRENT_REVIEW.md](../../CURRENT_REVIEW.md)** — Open commercial and product questions
