## Idea 23 — AI Knowledge Drift Monitor

Exact text (as presented earlier)

- AI Knowledge Drift Monitor
- Who: Support and documentation teams.
- MVP: Detect stale KB content vs incoming tickets/queries; suggest updates and owners.
- Pricing: $500–$2,000/mo.
- Validate: 100 KB pages; accepted update suggestions.

### Explain it simply (2–4 short passages)

- Over time, help articles go out of date. This tool compares what people are asking with what the articles say and flags pages that need updates.
- It suggests changes and pings the right owner.

Key terms (simple)

- KB: Knowledge base, a set of help articles.
- Drift: When content slowly becomes wrong or incomplete.
- Signal: Evidence from tickets/queries that content is outdated.

---

### Research, planning, market, and competition

Research checklist

- Interview 5 support/docs leaders; list top KBs, update cadence, pain points, and ownership model.
- Baseline: article views vs ticket deflection; update rate; age distribution.
- Must‑haves: source integrations (Zendesk/ServiceNow/Intercom), owner mapping, review workflow.

Competitive scan

- KB platforms with analytics
  - Pros: views and search terms.
  - Cons: limited semantic drift detection and ticket linkage.
- Manual reviews
  - Pros: expert eyes.
  - Cons: slow, inconsistent.

Your wedge (advantages)

- Semantic drift detection from tickets + search queries with prioritized suggestions and owner routing.

Why now

- Product change velocity is high; stale KBs drive costs and bad CX.

---

### A persuasive pitch (researcher + salesperson)

- Needs: “Reduce tickets,” “Keep KB fresh,” “Clear ownership,” “Prioritized updates.”
- Market: Analytics exist; semantic, actionable drift detection is lacking.
- Why try: 2‑week pilot on one KB to show deflection gains and update acceptance.

---

### Step-by-step plan (with checklists and trackers)

Phase 0: Discovery (1 week)

- [ ] Pick one high‑impact KB; map owners.
- [ ] Define signals (ticket intents, search queries, negative feedback).

Phase 1: MVP build (2 weeks)

- [ ] Ingest KB and ticket/search data; compute drift signals and scores.
- [ ] Suggestions with diff; owner notifications; review/approve flow.

Phase 2: Pilot (2–3 weeks)

- [ ] Run weekly suggestions; measure accepted updates and deflection changes.
- [ ] Iterate scoring and thresholds.

Phase 3: Rollout (2 weeks)

- [ ] Pricing $500–$2,000/mo; onboarding and security docs.
- [ ] Optional: auto‑PRs for docs repos.

Simple tracker

- Article | Score | Suggestion | Owner | Status | Accepted? | Impact | Notes

---

### Risks (financial, legal, technical)

- Financial: unclear ROI → focus on deflection and ticket reduction.
- Legal: customer data → anonymize and filter PII; BYOC connectors when needed.
- Technical: false positives → allow feedback and tuning; human review before publish.

---

### Glossary (simple)

- Deflection: Customers help themselves without contacting support.
- PII: Personal data that must be protected.
- Threshold: The score level at which we alert.

---

### Confirm to proceed (unattended mode)

Continuing unattended in Normal mode.
