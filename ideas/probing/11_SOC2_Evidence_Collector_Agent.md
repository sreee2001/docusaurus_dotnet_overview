## Idea 11 — SOC2 Evidence Collector Agent (M365/Azure/GitHub → auditor-ready)

Exact text (as presented earlier)

- SOC2 Evidence Collector Agent (pull from M365/Azure/GitHub; package for auditors)
- Who: Startups and mid‑market teams preparing for SOC2/ISO audits.
- MVP: Connectors that pull required evidence (users/roles, MFA, repo settings, CI logs, backups, change management) into an organized bundle with timestamps.
- Pricing: $500–$1,500/mo.
- Validate: Complete an evidence pack for one audit period with auditor acceptance.

### Explain it simply (2–4 short passages)

- Audits need proof: “Who has access?” “Is MFA on?” “Are backups working?” This tool automatically collects that proof from Microsoft 365, Azure, GitHub, and your ticket system.
- It organizes files and CSVs into the exact folders auditors expect, with dates and screenshots/logs if needed.
- You can run it monthly to stay ready. When the auditor asks, you hand over a clean, labeled package.
- Less scramble, fewer mistakes, and faster audits.

Key terms (simple)

- Evidence: Logs, settings, and reports that prove a control exists.
- Control: A rule/process that reduces risk (e.g., MFA required for admins).
- Period of performance: The time window the audit covers.

---

### Research, planning, market, and competition

Research checklist

- Identify 10 companies working on SOC2/ISO. Ask: controls in scope, target date, auditor, current tools (Vanta/Drata/DIY), and evidence pain points.
- Map required evidence by framework (SOC2 CCs, ISO Annex A): access reviews, MFA, change management, backups/DR, vulnerability scans.
- Confirm data boundaries: what can be exported automatically vs requires screenshots.
- Define success: auditor accepts the package with minimal rework.

Competitive scan

- Compliance platforms (Vanta, Drata, Secureframe, Hyperproof)
  - Pros: broad automation; policies; task tracking.
  - Cons: subscriptions cost; sometimes heavy; still gaps for custom evidence.
- DIY scripts + spreadsheets
  - Pros: control.
  - Cons: manual, error‑prone, hard to repeat.

Your wedge (advantages)

- Evidence‑first, lightweight, connector‑driven approach; great for teams not ready for full platforms or needing extra coverage.
- Auditor‑friendly packaging and audit trail; BYOC option to store evidence in customer tenant.

Why now

- Many teams racing to SOC2 need automation without committing to a full platform, or they need supplements to fill gaps.

---

### A persuasive pitch (researcher + salesperson)

- Target user needs: “Cut audit prep time,” “Reduce manual screenshots,” “Standardize evidence,” “Keep artifacts current.”
- Market state: Big platforms do a lot, but many teams still scramble for specific artifacts. Your tool fills that gap with focused automation.
- Alternatives trade‑offs: Platforms are powerful but heavy; DIY is flexible but slow. You deliver repeatable evidence runs with minimal setup.
- Why try it: In a week, generate a draft evidence pack your auditor can review.

---

### Step-by-step plan (with checklists and trackers)

Phase 0: Discovery (1 week)

- [ ] Identify controls in scope and auditor requirements (sample checklist).
- [ ] Confirm systems: Entra ID (AAD), Azure subscriptions, M365, GitHub, ticketing (Jira/ServiceNow), backups.
- [ ] Define acceptance: produce a draft pack for last month that the auditor accepts with <10% rework.

Phase 1: MVP build (1–2 weeks)

- [ ] Connectors:
  - [ ] Entra ID: users, groups, privileged roles, MFA status
  - [ ] Azure: RBAC assignments, activity logs, backup vault status
  - [ ] M365: Exchange/SharePoint settings, DLP summaries
  - [ ] GitHub: repo settings, branch protection, required reviews, Actions settings
  - [ ] Ticketing: change tickets, approvals, deployment links
- [ ] Packaging:
  - [ ] Map data to controls; folder structure (e.g., 01_Access, 02_Change, 03_Backups)
  - [ ] CSV + PDF reports, plus JSON for audits that accept raw
  - [ ] Manifest.json with timestamps, collection hashes
- [ ] Scheduling & storage:
  - [ ] Monthly run; storage in customer tenant (Blob/SharePoint)
  - [ ] Retention settings and encryption

Phase 2: Pilot (1–2 weeks)

- [ ] Run collection for one month; review with auditor or advisor.
- [ ] Address gaps (screenshots where APIs lack data).
- [ ] Add reviewer sign‑off and exceptions notes.

Phase 3: Packaging & rollout (1–2 weeks)

- [ ] Pricing: $500–$1,500/mo depending on systems and cadence.
- [ ] Onboarding: app registrations, scopes, storage location, control mapping selection.
- [ ] Security docs: data scope, retention, encryption; DPA.
- [ ] Case study: hours saved and rework ratio.

Simple tracker (columns)

- Company | Framework | Systems connected | Evidence coverage % | Rework % | Next audit date | Storage location | Status | Notes

---

### Risks (financial, legal, technical)

Financial

- Teams may graduate to full platforms.
  - Mitigation: integrate; offer as add‑on or export to those platforms.

Legal/compliance

- Sensitive evidence handling.
  - Mitigation: run in customer tenant; encrypt; minimal retention; access logging.

Technical

- API gaps require manual artifacts.
  - Mitigation: guided screenshot capture and notarization; templated evidence.

Operational

- Control scoping varies by auditor.
  - Mitigation: customizable mappings; auditor‑specific templates.

---

### Glossary (simple)

- RBAC: Role-Based Access Control—who can do what.
- DLP: Data Loss Prevention—policies to protect sensitive data.
- Manifest: A summary file describing what’s in the evidence pack.

---

### Confirm to proceed (unattended mode)

Continuing unattended in Normal mode; I can proceed to Idea 12 next unless you say “pause” or “stop”.
