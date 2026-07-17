# PRD: GitHub Copilot User Management System

**Owner:** Dafna Elkayam
**Status:** Draft v1
**Last updated:** 2026-07-17

---

## 1. Summary

A custom in-house web application that centralizes management of GitHub Copilot seats for the organization. It replaces the manual, ad-hoc process of provisioning, tracking, and reclaiming Copilot licenses with a self-service request/approval flow, cost visibility for finance, productivity insights for engineering leadership, and an audit trail suitable for SOC 2.

## 2. Problem

Today, Copilot seats are assigned manually by a small number of GitHub admins. This creates three concrete pain points:

1. **No self-service.** Engineers request seats over Slack/email; requests get lost, and approval is inconsistent.
2. **No cost accountability.** Finance cannot attribute Copilot spend to teams or cost centers, and idle seats accumulate silently.
3. **No productivity signal.** Engineering managers cannot see whether Copilot is actually being used or is delivering value on their teams.

A supporting audit trail is also missing, which will block the org's SOC 2 controls around access management.

## 3. Goals & Non-goals

### Goals

- Reduce time from Copilot request to provisioned seat from days to under one business day.
- Give every team lead a live view of their team's Copilot utilization and cost.
- Surface idle seats (30 days no activity) to admins for reclamation decisions.
- Produce an immutable, exportable audit log of every seat grant / revoke / request.
- Enforce SSO for all access to the tool.

### Non-goals (v1)

- Managing non-Copilot GitHub entitlements (repos, teams, Actions minutes).
- Automatic seat reclamation without human review.
- Managing Copilot for non-GitHub IDEs or third-party AI coding tools.
- Real-time coaching / prompt-quality analytics for individual developers.

## 4. Users & personas

| Persona | Primary needs |
|---|---|
| **Engineering manager / team lead** | See their team's seat list, utilization, and productivity signals; approve/deny requests from their reports. |
| **Finance / procurement** | Monthly cost per team, chargeback report, trend of active vs. idle seats. |
| **IT / GitHub admin** | Second-step approver; owner of idle-seat review; SCIM/SSO configuration; audit log export. |
| **End user (developer)** | Request a seat via a simple form; receive email when granted. (Not a primary UI persona in v1 — request form only.) |

## 5. Scope & scale

- **Seat count:** 100–1,000 active Copilot seats.
- **Users of the tool:** ~50–200 (managers, IT admins, finance).
- **GitHub orgs supported:** single org in v1, multi-org in a later phase.

## 6. Functional requirements

### 6.1 Identity & access

- SCIM 2.0 provisioning from the corporate IdP (Okta, Entra ID, or Google Workspace).
- SAML 2.0 SSO for all sign-in. No local passwords.
- Role model:
  - **Admin** (IT/GitHub admin) — full access.
  - **Manager** (team lead) — access scoped to their reports (derived from IdP manager attribute) and any teams they explicitly own.
  - **Finance viewer** — read-only access to cost and utilization dashboards, no PII beyond seat holder name/team.
- Manager-to-report mapping comes from the IdP; the system does not maintain its own org chart.

### 6.2 Request & approval flow

- Developer submits a request via a short form (justification, expected use).
- **Step 1 — Manager approval.** Request routes to the requester's IdP manager. Approve / deny / request-more-info. SLA 2 business days.
- **Step 2 — IT admin approval.** After manager approval, request routes to the IT admin queue. SLA 1 business day.
- On final approval, the system calls the GitHub Copilot API to assign the seat and emails the requester.
- On denial at any step, requester is emailed with the reason.
- Requesters can view their own request status.
- Auto-expiry: pending requests are auto-closed after 10 business days of inaction.

### 6.3 Idle-seat detection & flagging

- Nightly job pulls per-user Copilot activity from GitHub's Copilot usage API.
- A seat is **flagged idle** when it has had zero Copilot activity for **30 consecutive days**.
- Idle seats appear in a dedicated **Idle seats** view for IT admins with: user, team, days idle, last-active date, monthly cost.
- Admins take action per seat: **Revoke**, **Keep (30 days)**, or **Keep (90 days)**. All three actions are logged and reset the flag as appropriate.
- v1 does **not** auto-revoke and does **not** notify the seat holder — this is an admin-only workflow to avoid annoying legitimate but sporadic users while trust in the signal is built.

### 6.4 Analytics & reporting

#### Cost / chargeback (finance view)

- Monthly Copilot spend per team, per cost center, per manager.
- Trendline of active vs. idle seats over the last 12 months.
- CSV export for finance systems.
- Seat cost is configurable (default: current GitHub Copilot Business list price); the tool does not read invoices from GitHub Billing in v1.

#### Productivity signals (manager view)

- Per-team and per-user, over a selectable window (7 / 30 / 90 days):
  - Suggestion acceptance rate (% of Copilot suggestions accepted).
  - Total suggestions shown / accepted.
  - Language breakdown of accepted suggestions.
  - Active days in the window.
- All per-user views are visible **only** to the user's manager chain and to IT admins. Finance sees team-level aggregates only.

### 6.5 Audit log

- Immutable append-only log of every state-changing event: request submitted, approved, denied, seat granted, seat revoked, idle seat reviewed, role change, SCIM sync change.
- Each entry records: timestamp (UTC), actor (SSO identity), action, target user, before/after state, source (UI / API / SCIM / cron job).
- Retention: 7 years.
- Filterable UI + CSV export for auditors.

### 6.6 Notifications (v1)

- **Email only.** No Slack/Teams integration in v1.
- Triggered emails:
  - Request submitted → confirmation to requester.
  - Awaiting-approval → to current approver.
  - Approved / denied → to requester.
  - Weekly digest of pending approvals → to each approver with an open queue.
  - Weekly idle-seat summary → to IT admins.

## 7. Non-functional requirements

- **Availability:** 99.5% during business hours.
- **Performance:** dashboards render < 2s at p95 for the target scale.
- **Security:** SSO-only sign-in, TLS 1.2+, encryption at rest, secrets in a managed vault, least-privilege GitHub token scoped to Copilot admin APIs.
- **Compliance:** SOC 2 Common Criteria alignment for access management (CC6). Audit log designed to be evidence for CC6.1, CC6.2, CC6.3.
- **Data minimization:** the system stores identity, seat state, and activity aggregates — not the content of Copilot suggestions or code.

## 8. Integrations

| System | Purpose | Direction |
|---|---|---|
| Corporate IdP (Okta / Entra ID / Google) | SSO + SCIM user/manager sync | Inbound |
| GitHub Copilot admin API | Assign / revoke seats, list seats | Outbound |
| GitHub Copilot usage API | Pull per-user activity & acceptance signals | Inbound |
| Corporate SMTP / transactional email | Notifications | Outbound |

## 9. Success metrics

- **Time-to-provision:** median < 1 business day (from < 3 today).
- **Idle-seat rate:** < 10% of assigned seats idle >30 days within 6 months of GA (baseline TBD).
- **Chargeback coverage:** 100% of Copilot spend attributable to a team/cost center.
- **Admin toil:** IT admin hours/week on Copilot provisioning down ≥ 70%.
- **Auditor readiness:** SOC 2 access-management evidence generated in < 1 hour.

## 10. Recommended phasing

You asked for a phasing recommendation. Given the scope, a three-phase rollout is the safest path — each phase ships something usable and unblocks the next.

### Phase 1 — MVP (weeks 1–6)

- SSO sign-in + SCIM sync (managers and admins only; developers don't need accounts yet).
- Request form (public link, SSO-gated).
- Two-step approval flow (manager → IT admin), email notifications.
- GitHub Copilot seat assignment on final approval.
- Audit log (data model + read-only UI + CSV export).

**Exit criteria:** every new Copilot seat in the org is granted through the tool.

### Phase 2 — Visibility (weeks 7–12)

- Nightly Copilot usage ingestion.
- Idle-seat view (30-day rule, admin actions).
- Cost / chargeback dashboard for finance.
- Team-level utilization dashboard for managers.

**Exit criteria:** finance can produce a monthly chargeback report from the tool alone.

### Phase 3 — Insight (weeks 13–18)

- Per-user productivity signals (acceptance rate, language breakdown) for managers.
- Weekly digests (pending approvals, idle seats).
- Historical trend views (12-month).
- SOC 2 evidence-pack export.

**Exit criteria:** SOC 2 auditor accepts the tool's log + evidence pack as the system-of-record for Copilot access.

Total: ~18 weeks to full v1. MVP is usable at week 6.

## 11. Open questions

1. **Multi-org.** Do we anticipate managing more than one GitHub org (e.g. after an acquisition) within 12 months? If yes, the data model should be org-scoped from day 1.
2. **Seat cost source.** Configurable manual value in v1, or should we plan to read from GitHub's billing API in Phase 2?
3. **Manager attribute.** Does our IdP reliably populate the manager field for every employee? If not, we need a fallback (team-owner mapping).
4. **Contractor accounts.** How are contractors identified in the IdP, and do they follow the same manager approval flow or a different one (e.g. sponsor)?
5. **Denial appeal.** If a manager denies a request, should the developer be able to escalate, or is denial final?
6. **Idle threshold trust.** Is 30 days the right initial threshold, or start looser (e.g. 45 days) and tighten once we see real activity data?

## 12. Out of scope for v1 (revisit later)

- Slack/Teams approvals and DMs.
- Automatic idle-seat revocation.
- Chargeback push into the ERP / finance system.
- Non-GitHub AI coding tools (Cursor, Cody, etc.).
- Individual developer coaching / prompt quality.
- Multi-region data residency.
