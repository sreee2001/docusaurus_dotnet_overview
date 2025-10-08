## Idea 17 — Procurement Inbox Assistant

Exact text (as presented earlier)

- Procurement Inbox Assistant
- Who: Finance/Procurement teams.
- MVP: Parse quotes/POs/invoices, validate terms against policy, enrich vendor data, and populate ERP or a tracker; route exceptions.
- Pricing: $1,000–$3,000/mo.
- Validate: One procurement mailbox; reduction in manual entry and exception handling time.

### Explain it simply (2–4 short passages)

- Procurement teams get lots of emails with quotes, POs, contracts, and invoices. This assistant reads them, extracts key details, and checks them against company rules.
- It fills in your purchasing system or a spreadsheet automatically and flags anything odd for a human to review.
- It keeps buyers and requesters updated via Teams, so fewer emails and faster approvals.

Key terms (simple)

- PO: Purchase Order, a document authorizing a purchase.
- Net terms: Payment timeframe (e.g., Net 30 = pay in 30 days).
- ERP: Enterprise Resource Planning, often the system storing POs/invoices.

---

### Research, planning, market, and competition

Research checklist

- Interview 6 procurement/finance managers; catalog email volume, formats (PDF, docx, csv), current systems (SAP/Oracle/Dynamics/NetSuite), and approval flows.
- Baseline: manual entry time per email, exception rate, cycle time to approval/payment.
- Must‑haves: vendor matching, policy checks (terms, discounts, PO vs invoice matching), audit trail.

Competitive scan

- OCR/RPA invoice tools (Stampli, Tipalti, UiPath)
  - Pros: strong invoice processing.
  - Cons: inbox classification and policy logic across quotes/POs often weak, Teams integration limited.
- Email + spreadsheets
  - Pros: flexible.
  - Cons: manual, error‑prone, poor visibility.

Your wedge (advantages)

- Email‑first classification across all procurement artifacts (quotes→POs→invoices) with policy checks and lightweight ERP integration; strong Teams updates.

Why now

- Cost control is a priority; procurement teams are lean; email remains the main channel.

---

### A persuasive pitch (researcher + salesperson)

- Needs: “Reduce manual entry,” “Catch policy violations early,” “Shorten cycle times,” “Improve visibility without a big ERP project.”
- Market: Heavy AP automation exists, but inbox‑first, policy‑aware routing is underserved. Your assistant plugs this gap.
- Why try: 2‑week pilot on one mailbox to show 40–60% reduction in manual steps and faster approvals.

---

### Step-by-step plan (with checklists and trackers)

Phase 0: Discovery (1 week)

- [ ] Map current mailbox flows and policies (terms, thresholds, approvals).
- [ ] Define minimal fields for quotes/POs/invoices; identify ERP endpoints or export format.

Phase 1: MVP build (2 weeks)

- [ ] Classifier (quote/PO/invoice/contract/other); extraction of vendor, amounts, dates, PO numbers.
- [ ] Policy checks (terms, thresholds, missing PO, mismatch); Teams notifications; human‑in‑the‑loop queue for exceptions.
- [ ] ERP integration (API or CSV export) or tracker spreadsheet; audit log.

Phase 2: Pilot (2–3 weeks)

- [ ] Enable live on one mailbox; measure manual time saved and exception rate.
- [ ] Iterate extraction accuracy and policy rules.

Phase 3: Rollout (2 weeks)

- [ ] Pricing $1k–$3k/mo; onboarding kit; security document.
- [ ] Optional modules: vendor onboarding form, contract clause check.

Simple tracker

- Email | Type | Vendor | Amount | Terms | PO# | Status | Policy flags | Owner | Notes

---

### Risks (financial, legal, technical)

- Financial: small teams may see this as “nice‑to‑have” → focus on time savings and early‑pay discounts.
- Legal: contract data sensitivity → limit data stored; access controls and encryption.
- Technical: ERP variations → start with CSV export and webhook connectors; add APIs for top ERPs later.

---

### Glossary (simple)

- 2‑way/3‑way match: Comparing PO, receiving, and invoice before payment.
- Early‑pay discount: Vendor discount for paying early (e.g., 2/10 Net 30).
- Exception: A case that doesn’t pass automated checks and needs review.

---

### Confirm to proceed (unattended mode)

Continuing unattended in Normal mode.
