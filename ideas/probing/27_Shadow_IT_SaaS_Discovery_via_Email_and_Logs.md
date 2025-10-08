---
title: "Shadow IT / SaaS Discovery via Email & Logs"
description: "Detects app signups, receipts, and OAuth grants from inboxes and logs; surfaces risks and owners; recommends remediation for IT governance."
tags:
  - ShadowIT
  - SaaS
  - Discovery
  - Email
  - Logs
  - Security
---

## Idea 27 — Shadow IT / SaaS Discovery via Email & Logs

Exact text (as presented earlier)

- Shadow IT / SaaS Discovery via Email & Logs
- Who: IT governance and security teams.
- MVP: Detect app signups, receipts, and OAuth grants from inboxes and logs; surface risks and owners; recommend remediation.
- Pricing: $500–$2,000/mo.
- Validate: Findings that lead to actions (block/approve/SAML) in a pilot department.

### Explain it simply (2–4 short passages)

- Employees sign up for apps with work email. This tool watches inboxes/logs to find those apps and shows IT what’s in use.
- It flags risky ones and helps move them under SSO or shut them down.

Key terms (simple)

- Shadow IT: Software used without IT approval.
- OAuth grant: Permission an app gets to access your data.
- SAML/SSO: Secure login controlled by your company.

---

### Research, planning, market, and competition

Research checklist

- Interview 5 IT/security leads; list common shadow apps, current discovery methods, and approval paths.
- Baseline: number of unapproved apps, time to remediate, risk categories.
- Must‑haves: M365 email and sign‑in logs, risk scoring, owner identification.

Competitive scan

- CASB/SSPM tools
  - Pros: broad coverage.
  - Cons: heavy and expensive; inbox‑first discovery weak.
- Manual audits
  - Pros: targeted.
  - Cons: slow, incomplete.

Your wedge (advantages)

- Inbox‑and‑logs‑first discovery with simple remediation workflows that fit M365.

Why now

- SaaS sprawl increases risk and cost; easy discovery helps govern without blocking teams.

---

### A persuasive pitch (researcher + salesperson)

- Needs: “See what’s in use,” “Prioritize risks,” “Remediate quickly,” “Track ownership.”
- Market: Big platforms exist; a lightweight M365‑centric wedge is attractive.
- Why try: Pilot that finds real risks and enables quick SSO/approval moves.

---

### Step-by-step plan (with checklists and trackers)

Phase 0: Discovery (1 week)

- [ ] Enable mailbox patterns and sign‑in log access; define risk rubric.
- [ ] Map approval path (approve/SAML/block).

Phase 1: MVP build (2 weeks)

- [ ] Parsers for receipts/welcome emails; OAuth grant analysis; risk scoring.
- [ ] Owner suggestion; Teams notifications; simple approval flow.
- [ ] Dashboard: apps by risk and owner; remediation tracker.

Phase 2: Pilot (2–3 weeks)

- [ ] One department; measure findings → actions.
- [ ] Iterate parsers and scoring.

Phase 3: Rollout (2 weeks)

- [ ] Pricing $500–$2,000/mo; onboarding and security docs.
- [ ] Optional: expense system integration for cross‑check.

Simple tracker

- App | Owner | Risk | OAuth? | Status | Action | Notes

---

### Risks (financial, legal, technical)

- Financial: budget competition with CASB → position as complement/feeder.
- Legal: privacy/monitoring → consent banners; scope to work mailboxes.
- Technical: email pattern variance → rule packs with quick updates.

---

### Glossary (simple)

- CASB: Cloud Access Security Broker.
- SSPM: SaaS security posture management.
- Remediation: Steps to fix a risk.

---

### Confirm to proceed (unattended mode)

Continuing unattended in Normal mode.
