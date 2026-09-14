# Quotation Analysis Dashboard — Full Documentation

- **Route:** `/app/quotation-analysis-dashboard`
- **Module:** Lyfe Hardware
- **Roles with access:** System Manager, Sales Manager, Sales User
- **Files:**
  - `quotation_analysis_dashboard.json` — Page doctype registration
  - `quotation_analysis_dashboard.py` — whitelisted backend endpoints
  - `quotation_analysis_dashboard.js` — single-file Desk page renderer
- **Pattern:** built following the same structure as the P&L Dashboard (`lh/lyfe_hardware/page/pnl_dashboard/`)
- **Purpose:** gives sales/ops visibility into Quotation pipeline health — funnel progression, win/loss patterns, review bottlenecks, follow-up email effectiveness, revision churn, and payment collection status.

---

## Part 1 — Functional Documentation

What the dashboard does, in business terms. No SQL, no field names — just what a sales/ops user sees and can act on.

### 1.1 Filter Bar

- **Period presets:** Last 30 Days, Last 90 Days, This Year, Last Year, All Time, or a Custom date range.
- **Order Type filter:** All / Standard / Custom.
- **Refresh button:** reloads every period-filtered section plus Yesterday's Activity.
- All filters apply globally to every chart/table on the page except Yesterday's Activity, which always shows literally "yesterday" regardless of the filter bar.

### 1.2 KPI Strip (top of page)

Six at-a-glance numbers:

1. **Total Quotations** — count and total quoted value for the period.
2. **Open** — quotations still in progress (not Won/Lost/Expired).
3. **Won** — count and total won value.
4. **Lost / Expired** — count and value; dated by when it was marked Lost, not the original quote date.
5. **Win Rate** — Won ÷ (Won + Lost + still-open), shown with a "X won / Y lost / Z still open" note so the percentage is never read alone. "Won" includes quotations in either the "Won" or "Payment Received" state (§2.5).
6. **Avg Quote Value** — average value per quotation, excluding $0 quotations (and telling you how many were excluded).

### 1.3 Opportunity Count by Stage (donut chart)

A donut chart showing what share of quotations in the selected period sit in each pipeline stage: Draft, In Review, Sent to Customer, Won, Lost/Expired. Answers "where is our pipeline sitting right now" in one glance.

### 1.4 Quoted vs Won Value (chart)

A month-by-month bar chart comparing total quoted value against total won value. Shows whether won value is keeping pace with what's being quoted.

### 1.5 Quotation Funnel

A stage-by-stage bar chart of the real Quotation workflow: Draft → Pending Engineer Review → Production Details Review → Sales User Review → Pending Approval → Approved – Pending Send → Sent to Customer → Won. Below it, four "exit" stages (Not Feasible, Lost, Expired, Cancelled) show quotations that left the pipeline rather than progressing.

This is a **snapshot of where quotations sit right now**, not a strict drop-off funnel — the real workflow branches (engineer/senior review are optional paths), so a literal funnel would misrepresent it.

Clicking any bar filters the Quotation Details table (1.12) down to just that stage.

### 1.6 Win Rate by Dimension

A pie chart plus a detail list showing win rate broken down by a chosen dimension — **Region (Country & State)** (default), Order Type, Priority, or Owner (switchable via a dropdown). Region groups by the customer's linked Address (country, and state indented beneath it) rather than the old "Territory" field, since customer_address is populated far more consistently. Only submitted quotations (`docstatus = 1`) count here — Drafts never left the building, so they're excluded from "in progress."

**Known data-quality caveat:** "Owner" isn't currently a useful sales-rep leaderboard — quotations come through a shared inbox/integration, not per-rep, so `Owner` only has 3 real values today. The toggle still works correctly; it just won't show a meaningful breakdown until a real sales-rep field exists.

### 1.7 Open Quotation Aging

A bar chart bucketing every currently-open quotation by how many days it's been open: 0–3, 4–10, 11–25, 26+ days. These buckets match the automated follow-up scheduler's own nudge cadence (it emails at day 3/10/25 and auto-marks Lost at day 30), so this chart shows which quotations are approaching each milestone.

### 1.8 Review & Approval Bottlenecks

A bar chart of how many quotations are stuck in each in-flight review/approval state, plus the average number of days they've been stuck (flagged in red past 14 days). Answers "where in our review process are things piling up."

### 1.9 SLA Violations

Shows any currently-open SLA breaches touching Quotations — e.g. "Drawing Overdue" or "Feasibility Response Overdue" — with count and average age. This is always live (not affected by the period filter).

### 1.10 Follow-up Engagement

Measures whether the automated follow-up email system is working: how many emails were sent, what % were opened, what % clicked the payment link, and what % of sent/opened quotations eventually won. Also tracks how many quotations the scheduler gave up on (Day 30 auto-Lost).

### 1.11 Revision Churn vs Win Rate

A stacked bar chart answering "does re-quoting hurt our win rate?" — buckets quotations by how many times they were revised (0, 1, 2, 3+) and shows the win rate for each bucket.

**Current data note:** every quotation in the system is still on its original (0) revision as of this writing, so this panel has nothing to compare against yet. It will become useful once re-quoting starts happening in practice.

### 1.12 Payment Tracking

For Won/paid quotations, shows how much has actually been collected vs. the full quoted amount, grouped by payment method — flags any quotation that's only partially paid.

### 1.13 Category Breakdown

Shows quoted value, quotation count, and win rate broken down by product category (Item Group), as both a chart (top 8 categories) and a full table.

### 1.13b Needs Attention

⚠️ New — not present when this doc was last written, and previously listed under "not built" in §2.9. A risk-tiered (High/Medium) list combining aging, open SLA breaches, review bottlenecks, and partial-payment/follow-up signals into one actionable list, instead of making someone cross-reference three separate charts. High-risk reasons: an open SLA violation, 26+ days open, or stuck in review 14+ days. Medium-risk reasons: 14+ days open, a partially-received payment, or approaching the Day-30 auto-Lost cutoff. Sorted worst-first.

### 1.13c Month-on-Month Comparison

⚠️ New — not present when this doc was last written. A trailing 12-month (or a specific year's) view of quoted value, won value, quotation count, win rate, and average quote value per month — independent of the filter bar above, always showing its own trailing window or chosen year.

### 1.14 Quotation Details (full drill-down table)

The complete list of quotations (up to 500 most recent) matching whatever filters are active, with columns for status, owner, type, priority, revision number, value, and any linked Lyfe Order. Every column has its own search/filter box. An **Export Excel** button downloads exactly what's on screen.

### 1.15 Yesterday's Worked Quotations

A separate table, always showing yesterday specifically, listing every quotation that was created, edited, or moved forward in its workflow the previous day — independent of whatever period is selected above.

### 1.16 What is always excluded from every number on this page

- **Cancelled quotations** — never counted anywhere, except as their own bar in the Funnel (1.5), so you can still see how many were cancelled without them polluting any other metric.
- **Test/internal customer quotations** — quotations for **"Viral Kansodiya"**, **"Cherry"**, and **"Gaudioso"** are excluded everywhere on this dashboard. These are not real sales activity.

---

## Part 2 — Technical Documentation

For anyone maintaining or extending this dashboard.

### 2.1 Architecture

- Backend: Python whitelisted methods in `quotation_analysis_dashboard.py`, one per dashboard section, all reading from `tabQuotation` (plus a few joined tables).
- Frontend: a single `quotation_analysis_dashboard.js` file that builds the whole page's HTML/CSS in JS and calls each backend endpoint via `frappe.call`.
- No database schema changes were needed to build this dashboard — everything reads existing fields.

### 2.2 Every whitelisted endpoint

| # | Endpoint | Arguments | Purpose |
|---|---|---|---|
| 1 | `get_summary` | `filters` | KPI strip numbers |
| 2 | `get_stage_composition` | `filters` | Donut chart data (1.3) |
| 3 | `get_value_trend` | `filters` | Quoted vs Won monthly chart (1.4) |
| 4 | `get_funnel` | `filters` | Funnel bars (1.5) |
| 5 | `get_win_loss_breakdown` | `dimension`, `filters` | Win rate by dimension (1.6) |
| 6 | `get_open_quotation_aging` | `filters` | Aging buckets (1.7) |
| 7 | `get_review_bottlenecks` | `filters` | Bottleneck chart (1.8) |
| 8 | `get_sla_violations` | *(none)* | Live SLA breach list (1.9) |
| 9 | `get_followup_engagement` | `filters` | Follow-up email stats (1.10) |
| 10 | `get_revision_churn` | `filters` | Revision churn chart (1.11) |
| 11 | `get_payment_tracking` | `filters` | Payment collection status (1.12) |
| 12 | `get_category_breakdown` | `filters` | Category chart + table (1.13) |
| 13 | `get_needs_attention` | `filters` | Needs Attention list (1.13b) |
| 14 | `get_tile_drilldown` | `tile`, `value`, `filters` | Generic per-tile drill-down (1.5, 1.6, 1.8, 1.11, 1.13 — click-through) |
| 15 | `get_quotations_detail` | `filters`, `workflow_state` | Drill-down table (1.14) |
| 16 | `get_mom_summary` | `months`, `year` | Month-on-Month Comparison (1.13c) |
| 17 | `get_yesterday_activity` | *(none)* | Yesterday's Activity table (1.15) |

`filters` is always a JSON string of `{ preset, from_date, to_date, order_type, priority }`, built client-side as `gFilter` and sent unchanged to every period-aware endpoint. `get_mom_summary` and `get_sla_violations`/`get_yesterday_activity` are the exceptions — they deliberately ignore this filter rail (see §2.5/§1.9/§1.13c/§1.15).

### 2.3 Core constants (top of `quotation_analysis_dashboard.py`)

| Constant | Value | Used for |
|---|---|---|
| `WON_STATES` | `("Won", "Payment Received")` | Classifying a `workflow_state` as won — see §2.5 |
| `LOST_STATES` | `("Lost", "Expired")` | Classifying a `workflow_state` as lost |
| `OPEN_REVIEW_STATES` | 9 in-flight review states | Bottleneck/aging/Needs-Attention queries |
| `EXCLUDED_CUSTOMER_NAMES` | `("Viral Kansodiya", "Cherry", "Gaudioso")` | Test-customer exclusion, everywhere |
| `FUNNEL_STAGES` | 8-stage ordered list | Main funnel path (1.5) |
| `FUNNEL_EXIT_STAGES` | `["Not Feasible", "Lost", "Expired", "Cancelled"]` | Funnel's muted exit bars |
| `AGING_BUCKETS` | 4 day-range buckets | Aging chart (1.7) |
| `TREND_STAGE_GROUPS` | 5 coarse stage bands | Donut chart (1.3) |
| `NEEDS_ATTENTION_HIGH_AGE_DAYS` | `26` | Needs Attention (1.13b) — High-risk age cutoff, mirrors Aging's 26+ bucket |
| `NEEDS_ATTENTION_MEDIUM_AGE_DAYS` | `14` | Needs Attention (1.13b) — Medium-risk age cutoff |
| `NEEDS_ATTENTION_BOTTLENECK_HIGH_DAYS` | `14` | Needs Attention (1.13b) — mirrors Bottlenecks' own red threshold |
| `_PARTIAL_TOLERANCE` | `0.01` | Payment Tracking (1.12) — rounding-noise tolerance for "fully paid" |
| `_NO_PREVIOUS_PERIOD_PRESETS` | `("all_time",)` | `get_summary`'s period-over-period comparison — presets with no meaningful "previous period" |

### 2.4 The exclusion pattern — `_build_where()`

Every period-aware endpoint calls `_build_where(filters, ...)`, which:

1. Excludes `workflow_state = 'Cancelled'` (unless called with `include_cancelled=True`).
2. Excludes any quotation whose `customer_name` is in `EXCLUDED_CUSTOMER_NAMES`.
3. Applies `from_date`/`to_date`/`order_type`/`owner`/`priority` filters if present.

**Three endpoints don't call `_build_where()` and apply the same exclusion manually instead:**
- `get_yesterday_activity()` — has its own hardcoded WHERE clause (it needs `DATE(modified) = yesterday`, which `_build_where` doesn't support).
- `get_sla_violations()` — has no `tabQuotation` in its main query (it reads `SLA Violation Cache`/`SLA Rule`), so it `LEFT JOIN`s back to `tabQuotation` on `erp_docname = q.name` solely to apply the customer-name exclusion.
- `get_mom_summary()` — deliberately independent of the whole filter rail (§1.13c), so it builds its own inline WHERE clause with the same Cancelled + customer-name exclusion baked in, rather than calling `_build_where()`.

**Rule for future changes:** to exclude a new customer everywhere, add their name to `EXCLUDED_CUSTOMER_NAMES` — never add a one-off exclusion inside a single endpoint. This tuple is the single source of truth for exclusion.

### 2.5 Canonical formulas used across multiple endpoints

- ⚠️ **Corrected — Win Rate** = `Won ÷ (Won + Lost/Expired + In Progress)`, the whole population, **not** just closed quotations (this doc previously stated the closed-only formula, which is stale and — per README.md — was itself deliberately changed away from on 2026-08-31, since a closed-only denominator let a dimension misleadingly read 100% while deals were still undecided). Consequence: the rate moves as quotations sit unresolved — an in-progress deal later Lost leaves it unchanged, one later Won raises it. Single source of truth: the `_win_rate(won, lost, in_progress)` helper — used in `get_summary`, `get_win_loss_breakdown`, `get_followup_engagement`, `get_revision_churn`, `get_category_breakdown`.
- ⚠️ **Corrected — `WON_STATES` now includes "Payment Received", not just "Won".** The customer has paid and the only forward transition out of that state is "Mark as Won" — leaving it counted as open understated the win rate on money already collected. "Bank Transfer Selected" deliberately does **not** count (payment isn't confirmed there yet).
- **`_closed_states_sql()`** — a helper building `WON_STATES + LOST_STATES` placeholders, used by "is this quotation closed at all" queries (Aging, Needs Attention) so they can't silently drift out of step with `WON_STATES` the way a hardcoded `('Won','Lost','Expired')` list previously did.
- **Lost/Expired counting** uses `modified` date, not `transaction_date` — quotations only transition to Lost/Expired ~30 days after being quoted (the follow-up scheduler's auto-expire cadence), so filtering by quote date would undercount recent Lost/Expired quotations. Used in: `get_summary`, `get_funnel`, `get_mom_summary`.
- **Follow-up Engagement's own two rates** (`win_rate_after_send`, `win_rate_after_open`) are a *different* metric by design (% of sent/opened quotations later Won) — not the canonical Win Rate, and labeled separately in the UI so they're never confused.
- **Period-over-period comparison** — `get_summary` also returns a `previous` key (same computation run again on the immediately-preceding period of equal length), `None` for "All Time" or when no fixed date range exists.

### 2.6 Key schema facts (verified against the live database, not assumed)

- Active Quotation workflow is **`Quotation-3`** — `Quotation-1`/`Quotation-2` are inactive/superseded. Field: `workflow_state`.
- Quotation has **no plain `customer` column** — use `party_name` (Link) or `customer_name` (display text).
- `Lyfe Order.source_quotation` links back to the originating Quotation — one quotation can produce multiple Lyfe Orders (via splits), so any join uses `GROUP_CONCAT`. Population rate is only ~17% overall but ~86% for currently-Won quotations.
- `Quotation Item.item_group` is joined via `JOIN tabQuotation Item qi ON qi.parent = q.name` — this app's convention is always to join child tables on the child's `parent` field, never the parent doctype's field name.
- `SLA Violation Cache` fields: `sla_rule`, `erp_doctype`, `erp_docname`, `is_resolved`, `age_hours`.

### 2.7 RBAC

⚠️ **Corrected (Security review 2026-09-10)** — `_check_permission()` is **no longer** a single doctype-level gate. It now does both:

```python
frappe.has_permission("Quotation", "read", throw=True)
require_dashboard_role("quotation_analysis_dashboard")
```

The single check alone previously let any role with base Quotation read access call this page's whitelisted methods directly, bypassing both the Page doctype's own `roles` list and the MCP layer's `require_dashboard_role` gate. `require_dashboard_role` is backed by the single `DASHBOARD_ROLES` source of truth in `mcp_audit.py` (currently System Manager/Sales Manager/Sales User for this page — unchanged from before, so no legitimate user lost access; only the enforcement mechanism changed). Same fix pattern as `founder_dashboard.py`'s `_check_permission()`.

- **Every Sales User sees every Quotation** — there is no per-rep/per-owner row-level restriction on this dashboard, because there is none on the Quotation doctype itself (`DocPerm.if_owner = 0` for Sales User; the only row-level restriction anywhere is for Senior Engineer, scoped to their own review states). This is existing app-wide behavior, not a dashboard bug — worth stating explicitly since a Sales Manager might otherwise assume reps are scoped to their own numbers.

### 2.8 Known limitations

- No manual browser QA pass has been done beyond endpoint-level testing against production data — every endpoint has been executed directly and cross-checked, but rendered charts/interactions haven't been visually reviewed in a browser.
- Excel export only includes the columns shown in the Quotation Details table — no expense/margin breakdown (this is a pipeline dashboard, not a P&L one).
- "Owner" dimension (2.6, 1.6) doesn't currently have good enough data to be a useful leaderboard — only 3 distinct values exist, since quotations come through a shared inbox/integration rather than per-rep.
- Revision Churn (1.11) currently has no data to compare against — every quotation is still on revision 0.

### 2.9 Ideas explored but not built

| Idea | Why not built |
|---|---|
| ~~At-Risk / Action Needed panel~~ | ⚠️ **Built** — see §1.13b "Needs Attention". No longer an open idea. |
| ~~Month-on-Month Comparison~~ | ⚠️ **Built** — see §1.13c. |
| ~~Generic per-tile drill-down~~ | ⚠️ **Built** — see §1.5/§1.6/§1.8/§1.11/§1.13's click-through, backed by `get_tile_drilldown`. |
| Conversion lead time (Won → Lyfe Order creation) | Blocked — no `won_on` timestamp exists; `modified` isn't reliable for this (can be bumped by unrelated later edits) |
| Customer-level repeat/quotation history | Feasible, no data blockers — just not built yet |
| Lost Reason breakdown | Blocked — both candidate fields are 0% populated in the data |
| Slack digest of dashboard highlights | Blocked — requires Slack/n8n connector authorization not available in this environment |

### 2.10 Change history (most recent first)

- **2026-09-10** — Security fix: `_check_permission()` now also enforces `require_dashboard_role()`, closing a bypass of the Page's own role list (§2.7). `WON_STATES` extended to include "Payment Received" (§2.5). Added `get_needs_attention` (§1.13b/§2.2), `get_mom_summary` (§1.13c/§2.2), and generic `get_tile_drilldown` (§2.2) covering most tiles' click-through. Win/Loss by Dimension's "Territory" option replaced by "Region (Country & State)" off the linked Address, plus a `docstatus=1` filter added to that endpoint (§1.6/§2.5). `get_summary` gained a `previous`-period comparison (§2.5).
- **2026-08-22** — Added "Gaudioso" to `EXCLUDED_CUSTOMER_NAMES`.
- **2026-08-21** — Fixed `get_sla_violations()` to apply the customer exclusion (previously the one endpoint that didn't). Converted several bar-lists to real charts (Bottlenecks, Revision Churn, Category Breakdown top-8, Win/Loss pie). Added the Stage Composition donut (replacing an earlier monthly-stacked-bar version) and the Quoted vs Won Value trend chart. Reordered the page to put charts before detail tables.
- **2026-08-03** — Added Category Breakdown, Owner column on Details table, $0-quotation flagging on Avg Quote Value, canonical win rate formula, Lost/Expired date-field fix, Payment Tracking's Partially Paid logic fix, and the original customer exclusion (Viral Kansodiya, Cherry).

---

*This is the single reference file for this dashboard — functional (Part 1) and technical (Part 2) facts both live here. Keep it in sync with `quotation_analysis_dashboard.py`/`.js` in the same commit as any change.*
