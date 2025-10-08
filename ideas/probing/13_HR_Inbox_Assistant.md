## Idea 13 — HR Inbox Assistant (Onboarding/Offboarding)

Exact text (as presented earlier)

- HR Inbox Assistant (Onboarding/Offboarding)
- Who: HR teams in 50–500 employee companies.
- MVP: Classify HR emails (onboarding, offboarding, benefits, PTO), kick off tasks, post to Teams, update SharePoint checklists.
- Pricing: $500–$1,500/mo.
- Validate: One HR team; cycle time drop on onboarding tasks.

### Explain it simply (2–4 short passages)

- This is a helper for the HR email inbox. It reads messages like “We’re hiring Alex” or “Sam is leaving,” figures out what needs to happen, and starts the checklist.
- It creates tasks (accounts, equipment, badge), notifies the right people in Teams, and updates a shared tracker so everyone sees progress.
- For routine questions (benefits, PTO), it can reply with the correct link or steps, or draft a reply for HR to approve.
- That means faster onboarding/offboarding and fewer dropped balls.

Key terms (simple)

- Onboarding/offboarding: Steps to add a new employee or remove access when they leave.
- Checklists: Shared lists of tasks with owners and due dates.
- Classify: Detect the type of request from the email.

---

### Research, planning, market, and competition

Research checklist

- Identify 10 HR teams with shared inboxes (hr@, people@).
- Interview: current onboarding/offboarding flow, systems (M365, Azure AD, HRIS), pain points (missed steps, slow equipment orders).
- Baseline: average days to onboard/offboard, % rework, common FAQs.
- Confirm must‑haves: SSO, audit trail, consent, sensitive data handling.

Competitive scan

- HRIS workflow modules (BambooHR, Rippling, Workday)
  - Pros: integrated with HR data.
  - Cons: email intake weak; Teams notifications and custom checklists limited.
- Power Automate flows
  - Pros: quick to compose.
  - Cons: brittle logic; poor NLP; maintenance burden.
- Manual spreadsheets/SharePoint lists
  - Pros: simple.
  - Cons: error‑prone; no automation.

Your wedge (advantages)

- Email‑first capture plus reliable checklists and Teams notifications; integrates with HRIS/AD; permission‑aware links.

Why now

- Hybrid work increased coordination overhead; small HR teams need automation without re‑platforming.

---

### A persuasive pitch (researcher + salesperson)

- Target needs: “Fewer misses,” “Faster setup,” “Consistent steps,” “Clear status for managers.”
- Market: HRIS tools exist but don’t triage email well; DIY flows don’t scale.
- Advantage: Purpose‑built inbox triage + checklists + Teams; easy to adopt; measurable cycle time gains.
- Why try: 2‑week pilot to prove faster onboarding/offboarding and fewer misses.

---

### Step-by-step plan (with checklists and trackers)

Phase 0: Discovery (1 week)

- [ ] Map current checklists and owners; identify top FAQs.
- [ ] Baseline SLAs (days to onboard/offboard).

Phase 1: MVP build (2 weeks)

- [ ] Email classifier (on/off/FAQ); draft replies; Teams notifications.
- [ ] Checklist engine on SharePoint lists/Planner with owners/dates.
- [ ] HRIS/Azure AD hooks (create tickets or tasks, not direct changes initially).
- [ ] Audit trail and PII redaction in logs.

Phase 2: Pilot (2–3 weeks)

- [ ] Run with one department; measure cycle time and misses.
- [ ] Expand tasks and FAQs.

Phase 3: Rollout (2 weeks)

- [ ] Pricing $500–$1,500/mo; onboarding package.
- [ ] Security docs: data flows, retention, permissions.

Simple tracker

- Employee | Type (on/off) | Start/End date | Tasks open/closed | SLA days | Owner | Status | Notes

---

### Risks (financial, legal, technical)

- Financial: small teams churn → mitigate with low setup fees and clear ROI.
- Legal: PII handling → redact logs, least‑privilege, retention.
- Technical: HRIS API limits → start with task/ticket creation and human approvals.

---

### Glossary (simple)

- HRIS: Human Resources Information System.
- SLA: Target time to complete.
- PII: Personal data that must be protected.

---

### Confirm to proceed (unattended mode)

Continuing unattended in Normal mode.
