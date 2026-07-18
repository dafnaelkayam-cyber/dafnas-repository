# PRD: GitHub Copilot User Management System

**Owner:** Dafna Elkayam
**Status:** Draft v1
**Last updated:** 2026-07-18 (rev 7 — manager/FinOps stress test: non-users, overage pace, premium-model %, Phase 2 usage alerts)

---

## 1. Summary

A custom in-house web application that gives engineering managers daily visibility into how their team actually uses GitHub Copilot — per person, per LLM model, and per credit — through five screens: a **Home** landing page, a **My Usage** single-person drill-down, a **Compare Users** side-by-side table, self-service **My Teams**, and a manager-only **Department** rollup. Seat provisioning and reclamation workflows are a real need but are **deferred to a later phase**; v1 focuses entirely on usage and cost visibility for seats that already exist.

## 2. Problem

1. **No usage visibility.** Managers have no way to see, day by day, whether their team is actually using Copilot, which models they're using, or which features (chat, code review, coding agent) are getting adopted.
2. **No cost accountability.** Managers cannot see what each team member is costing in Copilot credits, or roll that up to a team/department total.
3. **No easy way to compare or group people.** Checking on a handful of specific people, or a whole team, means looking them up one at a time — there's no comparison view and no lightweight way to group people who don't map cleanly to GitHub's org structure.

## 3. Goals & Non-goals

### Goals

- Give every manager a **daily**, per-user view of Copilot usage for their reports.
- Let managers **compare specific people side-by-side** and **group people into their own teams** without depending on GitHub/AD sync.
- Give managers a **department-level rollup** across their full reporting line.
- Show managers **which features and models** their team members are actually using, and their **credit spend**.
- **Proactively alert managers** to non-users, credit-overage risk, and heavy premium-model usage — not just leave it to a dashboard someone has to remember to check.
- Enforce SSO for all access to the tool.

### Non-goals (v1)

- **Seat request/approval workflow** — deferred, see [§6.5](#65-seat-management-deferred).
- **Idle-seat detection & reclamation** — deferred, see [§6.5](#65-seat-management-deferred).
- Managing non-Copilot GitHub entitlements (repos, teams, Actions minutes).
- Managing usage for non-GitHub IDEs or third-party AI coding tools.
- Real-time coaching / prompt-quality analytics for individual developers.
- Syncing teams from Active Directory/GitHub Teams — teams are **self-created** in this app (see §6.1).

## 4. Users & personas

| Persona | Primary needs |
|---|---|
| **Engineering manager / team lead** | Daily usage per direct/indirect report; compare specific people; group people into self-created teams; department rollup; credit spend per person/team. |
| **Individual contributor** | Sign in and see their **own** usage and credit spend only — no visibility into anyone else (see §6.1 access model). |
| **IT / GitHub admin** | Org-wide (unscoped) version of every view above; audit log; SCIM/SSO configuration. |
| **Finance / procurement** | Cost visibility, most likely team/department-level — **UI not yet designed**, see [open question 8](#11-open-questions). |

## 5. Scope & scale

- **Seat count:** 100–1,000 active Copilot seats (seats themselves are provisioned outside this tool in v1 — see §6.5).
- **Users of the tool:** ~50–200 (managers, individual contributors, IT admins).
- **GitHub orgs supported:** single org in v1, multi-org in a later phase.

## 6. Functional requirements

### 6.1 Identity & access

- SCIM 2.0 provisioning from the corporate IdP (Okta, Entra ID, or Google Workspace).
- SAML 2.0 SSO for all sign-in. No local passwords.
- Role model:
  - **Admin** (IT/GitHub admin) — full access, org-wide, unscoped.
  - **Manager** — anyone with ≥1 direct report per the IdP. Scoped to their full reporting line (direct + indirect) plus any teams they've personally created.
  - **Individual contributor** — anyone with 0 direct reports. Sees only their own usage; **My Usage** shows just themselves (no report list); **Compare Users**, **My Teams**, and **Department** are hidden or empty.
- Manager-to-report mapping comes from the IdP; the system does not maintain its own org chart.
- **Search/comparison scope:** a manager can search for, view, or add to Compare **anyone in their reporting line (any depth) or anyone in a team they've created** — matching the wireframe's stated MVP scoping rule.
- **Teams are self-created, not synced.** A manager builds a team by naming it and picking members from search (§6.2.5) — entirely inside this app's own data model. **No AD-group or GitHub-Team sync is required for v1** (a change from earlier drafts of this PRD, which assumed AD→GitHub-Team sync was a prerequisite for any grouping/filtering feature — the wireframe replaces that with self-service teams, removing that dependency from the critical path). AD-group import into Teams is a possible later convenience (§10, Phase 5), not a blocker.
- **Department** is not a separately synced entity — it's an automatic, read-only rollup of every team belonging to anyone in the signed-in manager's reporting line (§6.2.6). This is how the original "filter by group connected to org structure" requirement is satisfied, using data already available from SCIM.

### 6.2 Usage dashboards (core of v1)

> **API validation (2026-07-17):** confirmed against GitHub's current Copilot usage metrics API (`users-1-day` / `users-28-day` reports).
> - ✅ Daily, per-user data — real, ~2-day reporting lag after a day closes.
> - ✅ Per-user, per-model **request counts** — real (`totals_by_model[]` breakdown, confirmed for chat; coding agent/CLI model attribution to be confirmed during implementation — see [open question 7](#11-open-questions)).
> - ✅ Per-user **total** AI credits (`ai_credits_used`) — real, but a single daily total, **not** split by model. GitHub's own docs label it "a metrics signal for analyzing consumption, not a billed total."
> - ❌ Per-user, per-model **credits/cost** — does not exist in the API, and not being estimated (decided 2026-07-17). Every screen below shows model **request counts** as hard data and credits as an **unsplit total** — never "N credits on model X."
> - ❌ Extensions/skillsets and custom/MCP agent usage — no signal in the API today.
> - Team and Department aggregates require **no additional GitHub API access** — they're computed by our own backend summing per-user data across a team's stored member list (self-created, §6.1), not by calling any GitHub team-scoped endpoint.
>
> **Derived metrics (computed by us, not returned by the API):**
> - **Non-user flag** — zero activity across the selected window. Distinct from GitHub's `ai_adoption_phase` "at-risk" cohort (which is a blended, opaque classification) — this is a plain, explainable "0 active days" count.
> - **Overage pace** — projected end-of-cycle credit usage = credits used so far ÷ days elapsed in the current cycle × total days in the cycle, flagged when the projection exceeds the user's allowance. Pure arithmetic on data we already ingest (`ai_credits_used` + known cycle boundaries).
> - **Premium-model %** — share of a user's requests that fall on a "premium" model tier. Requires us to **maintain our own model → cost-tier mapping** (GitHub's API returns a model name, e.g. `claude-sonnet-4.6`, not a tier) — an ongoing maintenance item as GitHub adds or re-prices models, not a one-time build.

#### 6.2.0 First MVP — daily use & cost signal

The very first release ships the smallest useful slice: enough for a manager to answer "is this person using Copilot every day, and is their spend under control" — one user at a time. It corresponds to wireframe screens **1 (Sign in), 2 (Home), 3 (My Usage), and 9 (empty search state)**.

**Exit criteria:** a manager can sign in, search for any one of their reports, and see for a selected date range: active/inactive per day, adoption phase, which features they used, their credit trend, and their model mix.

#### 6.2.1 Screen inventory & navigation

Left-rail global nav: **Home, My Usage, Compare Users, My Teams, Department** (Department hidden for individual contributors). Screen numbers below match the wireframe.

| # | Screen | Phase |
|---|---|---|
| 1 | Sign in | 1 |
| 2 | Home | 1 |
| 3 | My Usage (single-user detail) | 1 |
| 9 | Empty / no-results search state | 1 |
| 4 | Compare Users | 2 |
| 5 | My Teams (list) | 2 |
| 6 | Create team | 2 |
| 7 | Team view (aggregate + members) | 2 |
| 8 | Department (manager rollup) | 3 |

#### 6.2.2 Sign in & Home

- **Sign in:** SSO only. Copy communicates that "access is scoped to your reporting line automatically" — sets expectations before the user even lands.
- **Home** is the landing page after every sign-in. Contents:
  - Global search box — search by user, team, or department name (scope per §6.1).
  - Summary tiles: employee count, team count, an **at-risk count** (`ai_adoption_phase` cohort), a **non-users count** (zero activity in window — see the derived-metrics note above), and a **trending-toward-overage count** (overage pace) — all headline numbers, not buried in a detail view.
  - **Recently viewed** — last few users/teams the manager looked at, mixed entity types, click-through. This replaces the previously-planned "saved filter views" feature — automatic and zero-effort instead of user-maintained (see [open question 5](#11-open-questions) on whether manual saving is still wanted later).
  - **"Your credits this cycle"** — the *signed-in manager's own* credit usage (used / left / days to reset), since managers use Copilot too. New addition versus earlier drafts, which only covered viewing *others'* usage.
  - Empty search → empty state (§6.2.7).
  - Hidden/empty for individual contributors: employee/team/at-risk tiles (not meaningful with zero reports); their own credits widget still shows.

#### 6.2.3 My Usage (single-user detail)

- Left rail: searchable list of "My Reports" (the manager's direct reports, expandable to indirect reports via search). For an individual contributor, this rail is absent — the panel always shows their own data.
- Right detail panel, for a selected date range (default last 7 days):
  - Adoption phase badge (engaged / new / at-risk — display labels; confirm exact mapping to GitHub's `ai_adoption_phase` values during implementation, [open question 7](#11-open-questions)).
  - **Daily activity** — a chart of active/inactive per day.
  - **Features** — checklist: Completions, Chat, Code review, CLI, Coding agent (✓ / — per the `used_*` API flags).
  - **Model mix** — list of request counts per model (e.g. "sonnet-4.6 — 14 req", "gpt-5.4 — 6 req"), from `totals_by_model[]`, plus **premium-model %** (share of requests on a premium-tier model — derived metric above).
  - **AI credits** — daily & 28-day trend chart, plus "% used · N credits left · resets in N days" against the org's monthly allowance, plus a **pace indicator** ("on track" / "trending over allowance" — derived metric above).
- Reachable from Home search, Compare Users (row click), or a Team view's member list.

#### 6.2.4 Compare Users

- A table for putting a handful of specific people side by side. Add people via search-and-add chips (any combination of direct/indirect reports or team members, per §6.1's scope rule).
- Columns: **Adoption**, **Active days (7)**, **Model mix**, **Premium-model %**, **Credits (7d)**, **Left till cycle end**, **Pace** (on track / trending over).
- Row names link to that person's My Usage (§6.2.3).
- No saved/named comparisons in v1 — the comparison is session-scoped (add/remove chips); persisting a comparison is a possible later enhancement.

#### 6.2.5 My Teams (list, create, team view)

- **My Teams (list):** teams the signed-in manager has personally created, each showing a member count. "+ New team" opens Create team. Clicking a team opens Team view.
- **Create team:** name the team, then search-and-add members (chips). Save returns to My Teams.
- **Team view:** aggregate stats for the selected date range —
  - Team active rate (%), total credits used/left, adoption ratio (e.g. "4/6 engaged").
  - **Non-users** (e.g. "1/6 non-users this window") and **trending-toward-overage** (e.g. "2/6 trending over allowance") counts — same derived metrics as Home, rolled up to team level.
  - **Active members per day** — bar chart.
  - **Feature usage across team** — fraction of members who used each feature (e.g. "Completions: 6/6 · Chat: 5/6").
  - **Model mix (team total)** — aggregated request counts per model.
  - **Members table** — same columns as Compare Users (§6.2.4); names link to My Usage.
- All team aggregates are computed by our backend from stored team membership + each member's usage data — no external team-sync dependency (§6.1).
- **Open:** whether a team is private to its creator or can be shared/co-owned by another manager — not shown in the wireframe ([open question 6](#11-open-questions)).

#### 6.2.6 Department (manager rollup)

- Manager-only tab (hidden entirely for individual contributors). Lists every team belonging to anyone in the signed-in manager's full reporting line — automatic, not manually curated, not a separately synced entity (§6.1).
- Table: Team, Members, Active rate, Non-users, Trending-toward-overage, Credits used/left, and a "drill in" link into that team's Team view (§6.2.5).
- **Open:** should a manager be able to manually add/exclude a team from their Department view, for cases where real department boundaries don't perfectly match the reporting line? ([open question 9](#11-open-questions))

#### 6.2.7 Empty / no-results search state

- Shown from the Home search box when a query matches nothing. Simple "No matches for '\<query\>'" state — no results list, no error.

### 6.3 Audit log

- Immutable append-only log covering: sign-ins, dashboard/report access, team create/update/delete/membership-change, role changes, SCIM sync changes, and (once §6.5 ships) seat lifecycle events.
- Each entry records: timestamp (UTC), actor (SSO identity), action, target (user/team/role), before/after state where applicable.
- Retention: 7 years.
- Filterable UI + CSV export for auditors.

### 6.4 Notifications & alerts

- **Email only.**
- **Account/access notifications** (Phase 1): role changed, added to a team.
- **Usage alerts (Phase 2):** a weekly email per manager, scoped to their reporting line, summarizing the derived metrics from §6.2:
  - Non-users this week (who, and how many days since last activity).
  - Members trending toward credit overage (pace flag).
  - Heavy users — members whose request volume or premium-model % crosses a threshold (see [open question 11](#11-open-questions) on how "heavy" is defined).
  - Ships alongside Compare/Teams in Phase 2 since it's computed entirely from Phase 1 data — no new ingestion needed.
- Other usage-workflow notifications (approval digests, idle-seat summaries) remain deferred along with seat management (§6.5) until Phase 4.

### 6.5 Seat management (deferred)

Out of scope for the current build, kept here so the requirements aren't lost:

- **Self-service request + two-step approval** (manager → IT admin) for assigning new seats.
- **Idle-seat detection** (e.g. 30 days with zero activity) and admin-reviewed reclamation.
- GitHub Copilot admin API integration to assign/revoke seats.

This becomes Phase 4 (§10) once the usage dashboards are live and validated.

## 7. Non-functional requirements

- **Availability:** 99.5% during business hours.
- **Performance:** dashboards render < 2s at p95 for the target scale, including daily-granularity queries over a 90-day window.
- **Data freshness:** usage data reflects GitHub's own reporting lag — typically available ~2 days after a given day closes. Dashboards should surface the "as of" date so this isn't mistaken for real-time.
- **Security:** SSO-only sign-in, TLS 1.2+, encryption at rest, secrets in a managed vault, least-privilege GitHub token scoped to read-only Copilot usage/metrics APIs (no admin/write scope needed until §6.5 ships).
- **Compliance:** SOC 2 Common Criteria alignment for access management (CC6). Audit log designed to be evidence for CC6.1, CC6.2, CC6.3.
- **Data minimization:** the system stores identity, usage aggregates, and credit data — not the content of prompts, suggestions, or code.

## 8. Integrations

| System | Purpose | Direction |
|---|---|---|
| Corporate IdP (Okta / Entra ID / Google) | SSO + SCIM user/manager sync | Inbound |
| GitHub Copilot usage / metrics API | Daily per-user activity, per-model, and per-feature usage | Inbound |
| Corporate SMTP / transactional email | Account-related notifications | Outbound |
| GitHub Copilot admin API | Assign/revoke seats — **deferred**, needed only when §6.5 ships | Outbound (future) |

Note: AD-group/GitHub-Team sync is **not** an integration this tool depends on for v1 — see §6.1.

## 9. Success metrics

- **Manager adoption:** % of managers who view their team's usage at least weekly, within 6 weeks of launch.
- **Team creation:** % of managers who create at least one team, within 4 weeks of launch.
- **At-risk follow-up:** % of users flagged "at-risk" who move to a healthier adoption phase within 30 days of their manager viewing them.
- **Compare usage:** % of manager sessions that use Compare Users at least once.
- **Non-user reduction:** % decrease in the org-wide non-users count within 60 days of the weekly alert email launching.
- **Overage prevention:** % of users flagged "trending toward overage" who finish the cycle under their allowance.
- **Alert engagement:** % of managers who open the weekly usage alert email.
- **Auditor readiness:** SOC 2 access-management evidence generated in < 1 hour.

## 10. Recommended phasing

### Phase 1 — First MVP: daily use & cost signal

- SSO sign-in + SCIM sync (users, managers).
- Daily usage ingestion from GitHub's Copilot usage/metrics API.
- Sign in, Home (search, tiles including non-users and trending-toward-overage counts, recently viewed, own-credits widget), My Usage (single-user drill-down, including pace indicator and premium-model %), empty search state — wireframe screens 1, 2, 3, 9.
- Model → cost-tier mapping (initial version) to support premium-model %.
- Basic audit log (access events).

**Exit criteria:** see §6.2.0.

### Phase 2 — Compare, self-service Teams & usage alerts

- Compare Users (screen 4), with Premium-model % and Pace columns.
- My Teams list, Create team, Team view (screens 5, 6, 7), including team-level non-users and trending-toward-overage counts.
- **Weekly usage alert email** per manager (§6.4) — non-users, overage-pace flags, heavy/premium-model users.
- No external dependency — teams, comparisons, and alerts are all computed entirely from data already ingested in Phase 1.

**Exit criteria:** a manager can compare any set of their people side by side, create a team to see its aggregate stats and member table, and receives a weekly email flagging non-users, overage risk, and heavy premium-model usage across their scope.

### Phase 3 — Department rollup

- Department view (screen 8), built on Phase 2's team data plus the manager's full reporting-line hierarchy, including department-level non-users and trending-toward-overage counts.

**Exit criteria:** a manager with reports who themselves manage teams can see every team under them in one rollup, including non-user and overage-risk counts per team, and drill into any of them.

### Phase 4 — Seat Management (deferred scope)

- Self-service request form.
- Two-step approval (manager → IT admin).
- GitHub Copilot seat assignment via admin API.
- Idle-seat detection and admin-reviewed reclamation.

**Exit criteria:** every new Copilot seat in the org is granted through the tool; idle seats surface to admins.

### Phase 5 — Governance & Scale

- Additional digest types once seat management exists (idle-seat summaries, approval queues) — the core usage alert email already ships in Phase 2.
- Historical trend views (12-month).
- SOC 2 evidence-pack export.
- Finance-facing cost view (pending [open question 8](#11-open-questions)).
- Optional: AD-group import as a convenience shortcut for populating Teams (not required — Teams already work without it).
- Multi-org support.

**Exit criteria:** SOC 2 auditor accepts the tool's log + evidence pack as the system-of-record for Copilot access and usage.

## 11. Open questions

1. ~~**API granularity.**~~ **Resolved 2026-07-17** — per-model request counts real; per-user total credits real; per-model credits and extensions/agent usage not available, not estimated.
2. ~~**Cost source.**~~ **Resolved 2026-07-17** — uses `ai_credits_used` (per-user daily total), labeled as a usage signal, not an invoice-grade figure.
3. **Manager attribute.** Does our IdP reliably populate the manager field for every employee? If not, we need a fallback for who counts as a "manager" and what their reporting line is.
4. ~~**AD group depth & GitHub Team sync readiness.**~~ **Resolved 2026-07-18** — moot; teams are self-created (§6.1), no AD/GitHub-Team sync required for v1.
5. **Should comparisons or "views" be savable?** The wireframe shows only session-scoped Compare and automatic "Recently viewed," with no manual save feature. Confirm this is sufficient, or whether managers will want to name and persist a specific comparison/team view.
6. **Team sharing/co-ownership.** Is a team private to its creator, or can it be shared with or co-owned by another manager? Not shown in the wireframe.
7. **Coding-agent/CLI model attribution.** Confirm during implementation whether the per-model request-count breakdown covers coding agent and CLI activity, or only chat.
8. **Finance persona UI.** The wireframe is entirely manager-facing. What does finance actually need — a cut-down Team/Department view with names hidden, a separate export, or something else?
9. **Department curation.** Should a manager be able to manually add/exclude a team from their Department rollup, for cases where the real department doesn't perfectly match the reporting line?
10. **Seat management timing.** Any hard deadline (e.g. cost overrun, audit finding) that would pull Phase 4 forward?
11. **"Heavy user" / premium-model threshold.** Is this a fixed absolute threshold (e.g. > N requests/day, > X% premium-model requests), a relative one (e.g. top 10% of the team), or manager-configurable? Needs a decision before the Phase 2 alert email ships.
12. **Model cost-tier mapping ownership.** Who maintains the model → tier table as GitHub adds or re-prices models — is this a manual admin task, or should it be reviewed on a schedule (e.g. monthly)?
13. **Alert cadence.** Is weekly the right frequency for the Phase 2 usage alert, or do managers want something closer to real-time for overage risk specifically?

## 12. Out of scope (not planned)

- Slack/Teams notifications.
- Managing usage for non-GitHub AI coding tools (Cursor, Cody, etc.).
- Individual developer coaching / prompt-quality analytics.
- Multi-region data residency.
