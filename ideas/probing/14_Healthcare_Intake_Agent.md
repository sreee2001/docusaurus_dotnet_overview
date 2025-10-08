---
title: "Healthcare Intake Agent (Compliance-first)"
description: "De-identifies PHI in emails/forms, routes to queues, creates tickets, and summarizes for staff. Designed for clinics and health vendors with compliance guardrails."
tags:
  - Healthcare
  - Intake
  - Compliance
  - PHI
  - DeIdentification
  - Audit
---

## Idea 14 — Healthcare Intake Agent (Compliance-first)

Exact text (as presented earlier)

- Healthcare Intake Agent (Compliance-first)
- Who: Clinics/health vendors handling patient emails/forms.
- MVP: De‑identify PHI in emails/forms, route to correct queue, create tickets, summarize for staff.
- Pricing: $1,500–$5,000/mo.
- Validate: HIPAA-lite pilot; DUA in place.

### Explain it simply (2–4 short passages)

- Patients send messages with sensitive info. This tool reads them, removes personal details where not needed, and routes them correctly.
- Staff get a clean summary with the right next step (schedule, refill, billing), reducing back‑and‑forth.
- It logs actions for audits and keeps data in the right region/storage.
- Designed to respect privacy rules from day one.

Key terms (simple)

- PHI: Protected Health Information.
- De‑identify: Remove/blur personal details when not required.
- DUA: Data Use Agreement outlining allowed data handling.

---

### Research, planning, market, and competition

Research checklist

- Identify 8 clinics/health vendors; confirm PHI flows and tools (EHR, ticketing).
- Baseline: message volume, triage time, privacy incidents.
- Must‑haves: HIPAA guardrails, audit logging, region pinning, DUA/BAA readiness.

Competitive scan

- EHR portals (MyChart etc.)
  - Pros: integrated.
  - Cons: email remains common; limited automation; weak de‑identification.
- Generic AI triage tools
  - Pros: quick start.
  - Cons: not compliance‑first; weak PHI handling.

Your wedge (advantages)

- Privacy‑first ingestion, strong de‑identification, staff‑friendly summaries, auditable.

Why now

- Staffing pressure + privacy risk → automation with guardrails is attractive.

---

### A persuasive pitch (researcher + salesperson)

- Target needs: “Faster triage,” “Lower privacy risk,” “Less staff burden.”
- Market: Legacy tools miss email intake and de‑ID; compliance‑grade AI is scarce.
- Advantage: Designed for HIPAA expectations; BYOC option; clear audit trail.
- Why try: 2–4 week pilot to show safer, faster triage without changing the EHR.

---

### Step-by-step plan (with checklists and trackers)

Phase 0: Discovery (1 week)

- [ ] Confirm data flows, consent, and BAAs/DUAs.
- [ ] Define categories (schedule, refill, billing, records).

Phase 1: MVP build (2 weeks)

- [ ] Email/form ingestion; PHI redaction; category classifier.
- [ ] Ticket creation and Teams/secure messaging notifications.
- [ ] Audit logs; retention controls; region pinning.

Phase 2: Pilot (2–3 weeks)

- [ ] Shadow mode; validate de‑ID; measure triage speed.
- [ ] Enable for top 2 categories.

Phase 3: Rollout (2 weeks)

- [ ] Pricing $1.5k–$5k/mo; DUA templates; security docs.

Simple tracker

- Clinic | Volume/day | Categories | De‑ID accuracy | Triage time | Incidents | Status | Notes

---

### Risks (financial, legal, technical)

- Financial: slow adoption → target midsize groups; show compliance value.
- Legal: HIPAA violations → strict guardrails, BYOC, audits.
- Technical: EHR integration limits → start with ticketing + summaries.

---

### Glossary (simple)

- EHR: Electronic Health Record system.
- BAA: Business Associate Agreement (HIPAA contract).
- Region pinning: Keep data in a specific geography.

---

### Confirm to proceed (unattended mode)

Continuing unattended in Normal mode.
