# PRD: GitHub Copilot User Management System

**Owner:** Dafna Elkayam
**Status:** Draft v1
**Last updated:** 2026-07-17 (rev 5 — first MVP defined: per-user daily use + cost signal, filter by User only)

---

## 1. Summary

A custom in-house web application that gives engineering managers and finance daily visibility into how the organization's GitHub Copilot seats are actually used — per user, per LLM model, per feature/skill/agent, and per dollar — with saved, reusable filters and cost drill-down. Seat provisioning and reclamation workflows are a real need but are **deferred to a later phase**; v1 focuses entirely on usage and cost visibility for seats that already exist.

## 2. Problem

1. **No usage visibility.** Managers have no way to see, day by day, whether their team is actually using Copilot, which models they're using, or which features (chat, code review, coding agent, extensions/skills) are getting adopted.
2. **No cost accountability.** Finance and managers cannot see what each team member is costing in Copilot usage, let alone drill from a team total down to an individual and their model mix.
3. **No repeatable analysis.** Managers re-build the same filter combinations (their team, a specific model, a specific person) every time they check in — there's no way to save and reuse a view.

## 3. Goals & Non-goals

### Goals

- Give every team lead a **daily**, per-user view of Copilot usage for their team.
- Let team leads **save filter combinations** (users, group, LLM model) and reuse them.
- Show managers **which features, skills, and agents** their team members are actually using.
- Give managers and finance a **cost dashboard** with drill-down from team → individual → model/feature.
- Enforce SSO for all access to the tool.

### Non-goals (v1)

- **Seat request/approval workflow** — deferred, see [§6.5](#65-seat-management-deferred).
- **Idle-seat detection & reclamation** — deferred, see [§6.5](#65-seat-management-deferred).
- Managing non-Copilot GitHub entitlements (repos, teams, Actions minutes).
- Managing usage for non-GitHub IDEs or third-party AI coding tools.
- Real-time coaching / prompt-quality analytics for individual developers.

## 4. Users & personas

| Persona | Primary needs |
|---|---|
| **Engineering manager / team lead** | Daily usage per person on their team; which models, features, skills, and agents are used; cost drill-down; saved filters for repeat checks. |
| **Finance / procurement** | Team-level cost dashboard with drill-down; no per-user activity detail beyond cost. |
| **IT / GitHub admin** | Org-wide (unscoped) version of every view above; audit log; SCIM/SSO configuration. |

## 5. Scope & scale

- **Seat count:** 100–1,000 active Copilot seats (seats themselves are provisioned outside this tool in v1 — see §6.5).
- **Users of the tool:** ~50–200 (managers, IT admins, finance).
- **GitHub orgs supported:** single org in v1, multi-org in a later phase.

## 6. Functional requirements

### 6.1 Identity & access

- SCIM 2.0 provisioning from the corporate IdP (Okta, Entra ID, or Google Workspace).
- SAML 2.0 SSO for all sign-in. No local passwords.
- Role model:
  - **Admin** (IT/GitHub admin) — full access, org-wide, unscoped.
  - **Manager** (team lead) — access scoped to their reports (derived from IdP manager attribute) and any teams they explicitly own.
  - **Finance viewer** — cost dashboard only, team-level aggregates, no per-user usage/feature detail.
- Manager-to-report mapping comes from the IdP; the system does not maintain its own org chart.
- Group sync: for the **Group filter** (§6.2.2) to work against Copilot usage data, the relevant AD security groups must be synced to **GitHub Teams** via IdP team sync (a separate, org-level GitHub Enterprise feature — up to 5 IdP groups per team, one-way from the IdP, hourly refresh). This is a setup prerequisite, not something this tool configures. Our backend joins GitHub's `user-teams-1-day` report against the per-user usage report to resolve team/group membership per day, since GitHub does not provide a single pre-aggregated team-scoped usage endpoint.

### 6.2 Usage & cost dashboards (core of v1)

> **API validation (done — 2026-07-17):** confirmed against GitHub's current Copilot usage metrics API (`users-1-day` / `users-28-day` reports).
> - ✅ Daily, per-user data — real, ~2-day reporting lag after a day closes.
> - ✅ Per-user, per-model **request counts** — real (`totals_by_model[]` breakdown, confirmed for chat; coding agent/CLI model attribution to be confirmed during implementation).
> - ✅ Per-user **total** AI credits (`ai_credits_used`) — real, but a single daily total, **not** split by model. GitHub's own docs label it "a metrics signal for analyzing consumption, not a billed total."
> - ❌ Per-user, per-model **credits/cost** — does not exist in the API. Not being estimated (decided 2026-07-17) — the cost dashboard uses the per-user total instead (§6.2.5).
> - ❌ Extensions/skillsets and custom/MCP agent usage — no signal in the API today.
> - ⚠️ Group filtering requires AD groups to be synced to **GitHub Teams** via IdP team sync (org-level prerequisite) plus joining a separate `user-teams-1-day` report ourselves — GitHub does not provide one pre-built team-scoped endpoint.

#### 6.2.0 First MVP — daily use & cost signal

The very first release ships the smallest useful slice: enough for a manager to answer "is this person using Copilot every day, and is their spend under control" — one user at a time.

**Daily-use metrics per user:**
- Active / inactive per day — presence in the API's daily per-user report, no extra computation needed.
- Adoption phase (`ai_adoption_phase`) — GitHub's own cohort classification (e.g. new / engaged / at-risk), surfaced as-is instead of building custom thresholds.
- Feature-used flags per day: completions, chat, CLI, coding agent, code review (`used_*` fields).

**Cost metrics per user:**
- Daily and 28-day-rolling `ai_credits_used` total, trended over time.
- Model mix: request count per LLM model (`totals_by_model[]`) — a cost-coaching signal (frontier models cost more per request) even without exact per-model dollars.
- Trend against the org's monthly AI-credit allowance, to flag a user approaching their cap before overage.

**Filter:** **User only** in this first release — type-ahead search, one or more users, scoped to the manager's own reports (admins unscoped). Group and LLM-model filters, saved views, and the cost drill-down dashboard (§6.2.1–§6.2.5 below) follow once this ships. Decoupling Group filtering from the MVP also avoids the GitHub Team/IdP-sync prerequisite blocking the first release.

**Exit criteria:** a manager can search for any one of their reports and see, for a selected date range: active/inactive per day, adoption phase, which features they used, their credit trend, and their model mix.

---

The rest of §6.2 describes the fuller dashboard this MVP grows into (team rosters, Group/LLM-model filters, saved views, drill-down cost view) — see phasing in §10.

#### 6.2.1 Manager / Team Lead dashboard

- Landing view scoped automatically to the signed-in manager's reports (direct + indirect, from the IdP org chart) plus any teams they explicitly own.
- Team roster table: one row per person, with **daily usage** (not just weekly/monthly rollups) — active/inactive per day, requests, model(s) used, cost for the day.
- Date range picker (default: last 7 days, daily granularity) sitting alongside the roster.
- Drill-down from any roster row into that person's full usage detail.
- Same dashboard shell serves IT admins (org-wide, unscoped) and finance (cost view only, no per-user activity detail — see §6.1).

#### 6.2.2 Filters

Exactly three filter fields, available on every dashboard/table view:

- **Users** — one or more individuals (typed search / multi-select). Managers are restricted to users within their own scope; admins can filter across the org.
- **Group** — one or more AD/IdP-synced groups (§6.1), e.g. department, team, cost center.
- **LLM model** — one or more models available in Copilot's model picker (e.g. GPT-4.1, Claude Sonnet, Gemini, o-series).

Filters combine with AND logic across fields, OR logic within a field's multi-select (e.g. "Group = Platform Team AND Model = Claude Sonnet OR Model = GPT-4.1").

#### 6.2.3 Saved filters

- Any team lead (or admin) can save the current filter combination as a **named view** (e.g. "My team — Claude only").
- Saved views are private to the user who created them by default; a "shared with my team" option makes a view visible to other managers with overlapping scope.
- Saved views appear in a dropdown on every dashboard page and can be set as that user's default landing view.
- Users can rename, update (overwrite with current filters), or delete their own saved views.

#### 6.2.4 Model & feature usage

- Per-person (or rolled up per team/group via the standard filters), over the selected window:
  - **Requests per LLM model** — hard data, sourced directly from the API's per-user model breakdown (e.g. "14 requests via `claude-sonnet-4.6`, 6 via `gpt-5.4`" for that day). This is the data the **LLM model filter** (§6.2.2) narrows.
  - Code completions (inline IDE suggestions) — request count.
  - Copilot Chat — request count.
  - Copilot CLI activity — request count.
  - Code review — used yes/no per person (org aggregates counts, not rich per-user detail).
  - Copilot coding agent — used yes/no per person (the API exposes this as a flag, not a request count).
- **Not available, not shown in v1:** Extensions/skillsets and custom/MCP-connected agent usage — GitHub's usage metrics API has no signal for these today. Revisit if/when GitHub adds it.
- Drill-down from a team rollup to see which individuals are driving usage of a given model or feature.

#### 6.2.5 Cost dashboard

- Standalone dashboard showing Copilot charges for team members, using the same three filters (§6.2.2) and saved views (§6.2.3).
- Data source: each user's daily `ai_credits_used` total — the most direct cost-like field the API exposes. Displayed as credits and, optionally, an equivalent dollar figure (credits × $0.01 list rate). **Not** broken down by model — see §6.2 API validation note.
- Top level: cost per team/group over the selected window, with a trend chart.
- **Drill-down path:** team/group total → individual team members → that individual's daily/period total. Drill-down stops there; there is no per-model or per-feature cost split (decided 2026-07-17 — not estimating it).
- The **LLM model filter** narrows *which users* appear (those with activity on that model, per §6.2.4's request-count data) but does not split any individual's cost number by model.
- CSV export at any drill-down level.
- Labelled in the UI as a usage-based cost **signal**, not an invoice-grade figure — matches GitHub's own caveat on `ai_credits_used`. Finance should reconcile against GitHub Billing for actual invoicing.

### 6.3 Audit log

- Immutable append-only log covering: sign-ins, dashboard/report access, saved-filter create/update/delete, role changes, SCIM sync changes, and (once §6.5 ships) seat lifecycle events.
- Each entry records: timestamp (UTC), actor (SSO identity), action, target (user/view/role), before/after state where applicable.
- Retention: 7 years.
- Filterable UI + CSV export for auditors.

### 6.4 Notifications (v1 — minimal)

- **Email only.**
- v1 scope is limited to account/access notifications (e.g. role changed, saved view shared with you). Usage-workflow notifications (approval digests, idle-seat summaries) are deferred along with seat management (§6.5).

### 6.5 Seat management (deferred)

Out of scope for the current build, kept here so the requirements aren't lost:

- **Self-service request + two-step approval** (manager → IT admin) for assigning new seats.
- **Idle-seat detection** (e.g. 30 days with zero activity) and admin-reviewed reclamation.
- GitHub Copilot admin API integration to assign/revoke seats.

This becomes Phase 2 (§10) once the usage/cost dashboards are live and validated.

## 7. Non-functional requirements

- **Availability:** 99.5% during business hours.
- **Performance:** dashboards render < 2s at p95 for the target scale, including daily-granularity queries over a 90-day window.
- **Data freshness:** usage data reflects GitHub's own reporting lag — typically available ~2 days after a given day closes. Dashboards should surface the "as of" date so this isn't mistaken for real-time.
- **Security:** SSO-only sign-in, TLS 1.2+, encryption at rest, secrets in a managed vault, least-privilege GitHub token scoped to read-only Copilot usage/metrics APIs (no admin/write scope needed until §6.5 ships).
- **Compliance:** SOC 2 Common Criteria alignment for access management (CC6). Audit log designed to be evidence for CC6.1, CC6.2, CC6.3.
- **Data minimization:** the system stores identity, usage aggregates, and cost data — not the content of prompts, suggestions, or code.

## 8. Integrations

| System | Purpose | Direction |
|---|---|---|
| Corporate IdP (Okta / Entra ID / Google) | SSO + SCIM user/manager/group sync | Inbound |
| GitHub Copilot usage / metrics API | Daily per-user activity, per-model, and per-feature/agent usage | Inbound |
| Corporate SMTP / transactional email | Account-related notifications | Outbound |
| GitHub Copilot admin API | Assign/revoke seats — **deferred**, needed only when §6.5 ships | Outbound (future) |

## 9. Success metrics

- **Manager adoption:** % of managers who view their team's usage dashboard at least weekly, within 6 weeks of launch.
- **Saved-view usage:** % of managers with at least one saved filter view, within 4 weeks of launch.
- **Cost visibility coverage:** 100% of Copilot spend attributable to a team/individual/model via the cost dashboard.
- **Drill-down usage:** % of finance/manager sessions that use at least one drill-down level.
- **Auditor readiness:** SOC 2 access-management evidence generated in < 1 hour.

## 10. Recommended phasing

### Phase 1 — First MVP: daily use & cost signal (§6.2.0)

- SSO sign-in + SCIM sync (users, managers — group sync not required yet).
- Daily usage ingestion from GitHub's Copilot usage/metrics API.
- Per-user search + detail view: active/inactive per day, adoption phase, feature-used flags, credit trend, model mix.
- Filter by **User** only.
- Basic audit log (access events).

**Exit criteria:** see §6.2.0's exit criteria — a manager can look up any one report and see their daily activity, adoption phase, feature usage, and cost/model trend.

### Phase 2 — Full usage & cost dashboards

- Manager/Team Lead roster dashboard, team-wide (§6.2.1).
- Group and LLM-model filters, saved filter views (§6.2.2, §6.2.3).
- Model & feature usage rollups at team level (§6.2.4).
- Cost dashboard with team → individual drill-down (§6.2.5).
- Resolve the GitHub Team/IdP-sync prerequisite for Group filtering (§6.1) — an org-side dependency, flag early.

**Exit criteria:** a manager can open the tool, apply a saved filter, and see daily usage, feature adoption, and cost drill-down for their whole team without leaving the dashboard.

### Phase 3 — Seat Management (deferred scope)

- Self-service request form.
- Two-step approval (manager → IT admin).
- GitHub Copilot seat assignment via admin API.
- Idle-seat detection and admin-reviewed reclamation.

**Exit criteria:** every new Copilot seat in the org is granted through the tool; idle seats surface to admins.

### Phase 4 — Governance & Scale

- Weekly digests (usage summaries, idle seats, approvals).
- Historical trend views (12-month).
- SOC 2 evidence-pack export.
- Chargeback export into finance systems (ERP).
- Multi-org support.

**Exit criteria:** SOC 2 auditor accepts the tool's log + evidence pack as the system-of-record for Copilot access and usage.

## 11. Open questions

1. ~~**API granularity.**~~ **Resolved 2026-07-17** — see the API validation note in §6.2. Per-model request counts: real. Per-user total credits: real. Per-model credits, extensions/agent usage: not available; not being estimated.
2. ~~**Cost source.**~~ **Resolved 2026-07-17** — §6.2.5 uses `ai_credits_used` (the API's per-user daily total), labeled as a usage signal, not an invoice-grade figure.
3. **Manager attribute.** Does our IdP reliably populate the manager field for every employee? If not, we need a fallback (team-owner mapping) for dashboard scoping.
4. **AD group depth & GitHub Team sync readiness.** How many levels of AD/IdP groups need to map to GitHub Teams for §6.2.2's Group filter, and has GitHub Team IdP-sync already been set up for the org, or does that need to happen before Phase 1 can ship the Group filter?
5. **Saved-view sharing scope.** Should "shared with my team" saved views be visible to any manager in the org, or only to managers with overlapping report scope?
6. **Seat management timing.** Any hard deadline (e.g. cost overrun, audit finding) that would pull Phase 2 forward?
7. **Coding-agent/CLI model attribution.** Confirm during implementation whether the per-model request-count breakdown (§6.2.4) covers coding agent and CLI activity, or only chat — the researched docs were explicit about chat but ambiguous on the others.

## 12. Out of scope (not planned)

- Slack/Teams notifications.
- Managing usage for non-GitHub AI coding tools (Cursor, Cody, etc.).
- Individual developer coaching / prompt-quality analytics.
- Multi-region data residency.
