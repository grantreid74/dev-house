# PRD: CareLink Patient Scheduling

> **Template version**: 1.0.0
> **Status**: SIGNED-OFF
> **Change freeze**: 2026-03-10
> **Customer ID**: carelink

---

## 1. Business Context

- **Company**: CareLink — 8-person healthcare startup, pre-seed, 6 months old
- **Their customers**: Patients seeking specialist appointments; NHS-adjacent private clinics (UK)
- **Current state**: Clinics post availability on a spreadsheet. Patients email or call to book. Confirmation is a manual email. Cancellations cause double-booking. Patient records are in paper files.
- **Why now**: CareLink signed a letter of intent with 3 UK clinics. They need a working system to convert those LOIs into contracts. They're under time pressure — competitor launched last month.

---

## 2. Product Vision

A two-sided platform: patients discover specialists, book appointments, and receive secure reminders. Clinics manage their provider calendars, confirm bookings, and send pre-appointment documentation. All communications involving patient data are encrypted. HIPAA-equivalent UK standards (GDPR + NHS DSP Toolkit) apply. The system does not store diagnostic or treatment data — it is a scheduling and communication tool only.

---

## 3. Success Criteria

- [ ] Patient can search for a specialist by specialty and location, and book an available slot in under 3 minutes
- [ ] Clinic coordinator can view all upcoming appointments for all providers in a single calendar view
- [ ] Appointment confirmation SMS sent to patient within 60 seconds of booking
- [ ] Patient can cancel or reschedule up to 24 hours before appointment
- [ ] Secure messaging: patient and provider can exchange text messages; messages encrypted at rest
- [ ] PHI audit log: every access to patient records logged with user ID, timestamp, action type
- [ ] Terraform validates and deploys to AWS without manual intervention

---

## 4. User Types

| User Type | Description | Estimated Count | Access Level |
|-----------|-------------|-----------------|--------------|
| Patient | Books and manages appointments; receives reminders | 5,000 (Year 1) | Self-service |
| Provider (Specialist) | Manages own calendar; views patient pre-info | 200 | Provider portal |
| Clinic Coordinator | Manages all providers in a clinic; handles cancellations | 20 (3 clinics) | Clinic admin |
| CareLink Admin | Platform management; onboards clinics; system health | 3 | Super-admin |

**Peak concurrent users**: 200 (patients searching/booking during peak hours)
**Growth trajectory**: 5K patients / 200 providers at launch; 25K patients / 1K providers at 18 months

---

## 5. Core Features

### Authentication
- [ ] [AUTH] Patient self-registration with email verification (acceptance: patient can register without clinic involvement)
- [ ] [AUTH] Provider and coordinator login (clinic-created accounts; clinic coordinator invites providers) (acceptance: provider receives invite, sets password, can log in)
- [ ] [AUTH] Password reset (acceptance: email received within 2 minutes; link expires in 1 hour)
- [ ] [AUTH] Session expiry after 30 minutes inactivity for clinic staff (GDPR requirement) (acceptance: redirected to login after 30 minutes idle)
- [ ] [AUTH] PHI access gating: patient data visible only to assigned provider + clinic coordinators (acceptance: Provider A cannot access Provider B's patients)

### Discovery & Search
- [ ] [SEARCH] Patient searches specialists by: specialty (dropdown), location (postcode + radius), availability window (date range) (acceptance: results returned in <2 seconds)
- [ ] [SEARCH] Provider profile page: photo, credentials, specialty, clinic location, available slots (acceptance: information editable by coordinator; visible to patients)
- [ ] [SEARCH] Show next available 5 slots for a provider (acceptance: only slots not already booked are shown)

### Booking
- [ ] [BOOKING] Patient selects a slot and books (acceptance: slot marked unavailable immediately; confirmation sent)
- [ ] [BOOKING] Booking confirmation: email to patient + SMS to patient phone number (acceptance: both arrive within 60 seconds)
- [ ] [BOOKING] Clinic coordinator receives booking notification (acceptance: email within 60 seconds)
- [ ] [BOOKING] Patient can cancel/reschedule up to 24 hours before appointment (acceptance: cancellation releases slot; patient and clinic notified)
- [ ] [BOOKING] Clinic coordinator can cancel any appointment with a reason (acceptance: patient notified with reason; slot released)
- [ ] [BOOKING] Booking history: patient can view past and upcoming appointments (acceptance: list sorted by date)

### Provider Calendar Management
- [ ] [CALENDAR] Provider sets recurring availability (e.g., Monday 9am-1pm, Wednesday 2pm-5pm) (acceptance: recurring slots appear for next 12 weeks)
- [ ] [CALENDAR] Provider can block specific dates/times (acceptance: blocked slots not shown to patients)
- [ ] [CALENDAR] Coordinator view: all providers' calendars in a single day/week view (acceptance: colour-coded per provider; conflicts visible)
- [ ] [CALENDAR] Coordinator can add manual appointments (for walk-ins / phone bookings) (acceptance: manual bookings appear in calendar and count against availability)

### Secure Messaging
- [ ] [MSG] Patient can send a message to their provider pre-appointment (acceptance: message stored encrypted; provider notified by email)
- [ ] [MSG] Provider can reply to patient messages (acceptance: patient notified; encrypted in transit and at rest)
- [ ] [MSG] Coordinator can send an attachment to a patient (e.g., pre-appointment form as PDF) (acceptance: patient can download; file stored encrypted)
- [ ] [MSG] Messages auto-deleted after 90 days post-appointment (GDPR retention) (acceptance: deletion logged in audit trail)

### Reminders
- [ ] [REMIND] SMS reminder to patient 48 hours before appointment (acceptance: sent at a consistent time; includes appointment details)
- [ ] [REMIND] SMS reminder to patient 2 hours before appointment (acceptance: sent; includes clinic address)
- [ ] [REMIND] Patient can opt out of SMS reminders (acceptance: opt-out persists; no SMS sent after opt-out)

### CareLink Admin
- [ ] [ADMIN] Onboard a new clinic: create clinic profile, add coordinators (acceptance: clinic live within 15 minutes)
- [ ] [ADMIN] Platform health dashboard: active clinics, bookings today, error rate (acceptance: dashboard data < 30 seconds stale)
- [ ] [ADMIN] PHI audit log viewer: search by date range, user, action type (acceptance: log queryable; exportable as CSV)

---

## 6. Non-Functional Requirements

### Compliance & Security
- **Compliance mandates**: GDPR (UK variant); NHS Data Security and Protection (DSP) Toolkit Tier 1; HIPAA-equivalent design patterns (customer may expand to US market)
- **Data sensitivity**: PHI — patient identifiers, appointment reasons, messages. Must be encrypted at rest (AES-256) and in transit (TLS 1.3+)
- **Audit trail required**: Yes — 7-year retention. Every access to patient-identifiable data logged.
- **Data residency**: UK only (London region). No data to leave the UK.
- **Multi-tenancy**: Strict per-clinic isolation. Provider from Clinic A cannot access Clinic B's patients.

### Availability & Performance
- **Target uptime**: 99.9% (patients book during clinic hours; downtime = missed appointments = patient harm risk)
- **RTO**: 1 hour
- **RPO**: 1 hour (appointment data is PHI; loss is a reportable incident)
- **Response time**: Search results < 2 seconds; booking confirmation < 5 seconds
- **Peak load**: 200 concurrent users at school holiday start (high booking season)

### Integration
- **Third-party integrations required**: Twilio (SMS), SendGrid (email), Stripe (future payment processing — out of scope Phase 1)
- **Existing systems**: None — greenfield
- **API requirements**: REST API. Webhook support for clinic systems that want to pull booking events.

---

## 7. Deployment Preferences

- **Cloud provider preference**: AWS (eu-west-2, London) — CareLink has AWS credits from Activate; team has prior AWS experience
- **Infrastructure budget**: $350-450/month (CareLink pays; factored into clinic subscription pricing)
- **Kubernetes preference**: No — prefer managed container services (ECS Fargate or equivalent)
- **Custom domain required**: Yes — `carelink.health` (patient-facing); `portal.carelink.health` (clinic-facing)
- **Existing infrastructure**: Greenfield. AWS Activate account with credits only.
- **Self-hosted option**: Not required
- **Timeline to UAT**: 5 weeks from PRD sign-off

### Expected pattern selection: Tier 4 (Enterprise Isolated)

Reasoning: PHI + GDPR + NHS DSP Toolkit require dedicated data stores per clinic. Shared Foundation database acceptable for non-PHI (user accounts), but appointment data and messages require per-clinic database instances. $350-450/month budget can support Tier 4 at the clinic count in scope (3 clinics initially). AWS required (data residency + team familiarity).

*Note for Harness*: PHI means `feature_list.json` items involving patient data must include encryption steps. The Harness should flag any feature touching patient records to confirm encryption is in scope for that feature's implementation.

---

## 8. Out of Scope

- Payment processing (Stripe integration deferred to Phase 2)
- Diagnostic data storage or EHR integration (CareLink is scheduling only)
- Video consultation (telemedicine) features
- AI triage or symptom checker
- Multi-country deployment (UK only in Phase 1)
- Mobile app (responsive web only)
- Automated insurance/billing integration

---

## 9. Open Questions

| Question | Owner | Impact if Unresolved |
|----------|-------|----------------------|
| Are appointment reasons (what the patient is coming for) considered PHI and must be encrypted? | Customer + Legal | If yes: field encrypted at rest; if no: can be stored as plain text for search performance |
| Twilio number: shared short code or dedicated long number per clinic? | Dev-House recommendation | Dedicated number per clinic is cleaner for patient experience; costs ~£1/month per clinic number |
| Should the NHS DSP Toolkit compliance self-assessment be in-system or external? | Customer | If in-system: adds ~8 features. If external (PDF self-assessment): out of scope. |
| Should patients be able to upload documents (referral letters)? | Customer | Adds file storage + PHI handling complexity; deferred to Phase 2 recommended |

---

## 10. Provenance

```
Template:  examples/prd-template.md v1.0.0
Customer:  carelink
PRD owner: ceo@carelink.health
Signed:    2026-03-10
Reviewed:  Dev-House
```

---

## Harness Notes

*Added by Harness Initializer — do not edit manually*

```yaml
# To be populated by Harness Session 1
pattern_selected: tier4-enterprise
services_identified: []
feature_count: 0
harness_status: pending
phi_flagging_required: true
```
