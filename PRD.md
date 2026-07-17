# PRD: GitHub Copilot User Management System

**Owner:** Dafna Elkayam
**Status:** Draft v1
**Last updated:** 2026-07-17 (rev 3 — usage/cost visibility is now v1; seat management deferred)

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
- Group sync: Active Directory security groups (synced to the IdP, e.g. via Entra ID/Okta) are pulled in via the same SCIM feed and mirrored as filterable groups, preserving the org structure (department, team, cost center) already defined in AD.

### 6.2 Usage & cost dashboards (core of v1)

> **Key dependency:** everything in this section depends on GitHub's Copilot usage/metrics telemetry being available at daily, per-user granularity — including per-model and per-feature/agent breakdown. This needs to be validated against the GitHub Enterprise Cloud Copilot Metrics API before development starts; see [open question 1](#7-open-questions). Some fields (notably per-model cost and agent/skill-level usage) may only be available in aggregate today and would need to be estimated or descoped until GitHub's API matures.

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

#### 6.2.4 Feature, skill & agent usage

- A dedicated view showing, per person (or rolled up per team/group via the standard filters), which Copilot surfaces they've used and how often over the selected window:
  - Code completions (inline IDE suggestions)
  - Copilot Chat (IDE, github.com, mobile, CLI)
  - Code review / PR summaries
  - Copilot coding agent (autonomous PR-opening agent)
  - Extensions / skillsets (including custom, org-published ones)
  - Custom / MCP-connected agents
- Table shows: feature/skill/agent name, request count, last-used date, per the applied filters.
- Drill-down from a team rollup to see which individuals are driving usage of a given feature.

#### 6.2.5 Cost dashboard

- Standalone dashboard showing Copilot charges for team members, using the same three filters (§6.2.2) and saved views (§6.2.3).
- Top level: cost per team/group over the selected window, with a trend chart.
- **Drill-down path:** team/group total → individual team members → that individual's cost broken down by model → (where available) by feature/agent.
- CSV export at any drill-down level.
- Cost figures reconcile between this dashboard and the per-user detail in §6.2.1/§6.2.4 — same underlying data, different entry points.

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

### Phase 1 — Usage & Cost Visibility (MVP)

- SSO sign-in + SCIM sync (users, managers, groups).
- Daily usage ingestion from GitHub's Copilot usage/metrics API.
- Manager/Team Lead dashboard with daily per-user roster (§6.2.1).
- Filters: Users, Group, LLM model (§6.2.2) + saved filters (§6.2.3).
- Feature/skill/agent usage view (§6.2.4).
- Cost dashboard with team → individual → model drill-down (§6.2.5).
- Basic audit log (access + saved-view events).

**Exit criteria:** a manager can open the tool, apply a saved filter, and see daily usage, feature adoption, and cost drill-down for their team without leaving the dashboard.

### Phase 2 — Seat Management (deferred scope)

- Self-service request form.
- Two-step approval (manager → IT admin).
- GitHub Copilot seat assignment via admin API.
- Idle-seat detection and admin-reviewed reclamation.

**Exit criteria:** every new Copilot seat in the org is granted through the tool; idle seats surface to admins.

### Phase 3 — Governance & Scale

- Weekly digests (usage summaries, idle seats, approvals).
- Historical trend views (12-month).
- SOC 2 evidence-pack export.
- Chargeback export into finance systems (ERP).
- Multi-org support.

**Exit criteria:** SOC 2 auditor accepts the tool's log + evidence pack as the system-of-record for Copilot access and usage.

## 11. Open questions

1. **API granularity (critical).** Does GitHub's Copilot usage/metrics API expose daily, per-user data broken down by model and by feature/agent (chat, coding agent, extensions, custom/MCP agents)? This is a hard dependency for §6.2.4 and the model/feature portions of §6.2.5 — needs validation against the GitHub Enterprise Cloud Copilot Metrics API before Phase 1 is scoped in detail.
2. **Cost source.** Is per-model cost derived from GitHub's premium-request multipliers, or a separate billing export? Affects how §6.2.5 sources its numbers.
3. **Manager attribute.** Does our IdP reliably populate the manager field for every employee? If not, we need a fallback (team-owner mapping) for dashboard scoping.
4. **AD group depth.** How many levels of AD/IdP groups do we need to support for filtering (just team/department, or arbitrary nested org units)?
5. **Saved-view sharing scope.** Should "shared with my team" saved views be visible to any manager in the org, or only to managers with overlapping report scope?
6. **Seat management timing.** Any hard deadline (e.g. cost overrun, audit finding) that would pull Phase 2 forward?

## 12. Out of scope (not planned)

- Slack/Teams notifications.
- Managing usage for non-GitHub AI coding tools (Cursor, Cody, etc.).
- Individual developer coaching / prompt-quality analytics.
- Multi-region data residency.
