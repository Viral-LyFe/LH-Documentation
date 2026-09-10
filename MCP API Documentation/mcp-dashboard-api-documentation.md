# MCP API Documentation — Dashboard Tools

> Covers every MCP tool for the three dashboards below, organized dashboard-by-dashboard.
> Companion to `mcp-full-api-documentation.md`, which lists all 151 MCP tools (these 53
> included, in summary form) plus server-wide behavior common to every tool.
>
> - `apps/lh/lh/lyfe_hardware/page/founder_dashboard/`
> - `apps/lh/lh/lyfe_hardware/page/customer_intelligence_dashboard/`
> - `apps/lh/lh/lyfe_hardware/page/quotation_analysis_dashboard/`
>
> Tool source: `apps/lh/lh/lyfe_hardware/mcp_tools/founder_dashboard.py`,
> `customer_intelligence.py`, `quotation_analysis.py`.
>
> Last verified against source: 2026-09-10. **If you add, remove, rename, or change the
> permission behavior of any tool in this document, update it in the same change — see
> the rule in `apps/lh/CLAUDE.md`.**

---

## How to call these tools

Every tool below is called through the **same one endpoint** — there is no per-tool
URL. See `mcp-full-api-documentation.md`'s "How to call these tools" section for the
full request/response reference (headers, JSON-RPC body shape, response envelope,
error format). Quick summary:

```
POST https://<your-site>/api/method/lh.mcp.handle_mcp
Content-Type: application/json
Authorization: token <api_key>:<api_secret>

{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "get_founder_summary",
    "arguments": { "global_filters": "{\"from_date\": \"2026-01-01\"}" }
  }
}
```

Replace `params.name` with any tool name from the sections below, and
`params.arguments` with that tool's parameters (as a JSON object keyed by parameter
name — omit `arguments` entirely, or pass `{}`, for a tool with no parameters).

---

## Server-wide behavior (applies to every tool below — read once)

- **Read-only.** No tool in any of these three files performs a write — confirmed by inspection: `founder_dashboard.py` and `quotation_analysis_dashboard.py` (the underlying Desk-page Python modules) contain zero `.insert(`/`.save(`/`.submit(`/`db.set_value(`/`frappe.enqueue(` calls anywhere in their files. The Customer Intelligence Dashboard's underlying module does contain two writes (`set_component_gap_state`, `confirm_customer_type`) — both are excluded from MCP entirely (see that dashboard's section below).
- **Every call is audited** via `audited_tool()` — one `MCP Audit Log` row per call (`user`, `tool_name`, `status`, `input_args`, `output_result`, `row_count`, `execution_time_ms`). A denied call's row never contains `output_result` — the role gate throws before `frappe.call()` ever reaches the underlying dashboard function, so no data is fetched (let alone logged) for a call that's rejected.
- **Single source of truth for roles.** All three dashboards' base access gate reads from one dict — `DASHBOARD_ROLES` in `lh/lyfe_hardware/integrations/mcp_audit.py`. That same dict is also read directly by each dashboard's own backend `_check_permission()` (as of 2026-09-10), and each dashboard's Page `.json` `roles` list is hand-verified to match it. This means: calling a dashboard's whitelisted Python function directly (bypassing MCP and the Desk page entirely) enforces the *same* role policy as calling it through MCP — verified live, 24/24 matched cells across all three dashboards' full role matrices, both via MCP and via direct backend calls.
- **Errors are sanitized.** Non-permission exceptions never reach the caller with their original message/traceback — see the full API doc's server-wide section for detail.

---

# 1. Founder Dashboard

**File:** `lh/lyfe_hardware/mcp_tools/founder_dashboard.py` — 25 tools, `get_founder_*` prefix.
**Underlying Desk page:** `lh/lyfe_hardware/page/founder_dashboard/founder_dashboard.py`.

## 1.0 Dashboard-level Security (applies to all 25 tools)

- **Allowed roles:** `System Manager`, `Super Admin` (+ `Administrator`, which is not a role but the built-in superuser and always passes every role check).
- **`Founder` does NOT grant access**, as of 2026-09-10 — removed per explicit decision, both from `DASHBOARD_ROLES["founder_dashboard"]` and from `founder_dashboard.json`'s own Page `roles` list (`bench migrate` run to sync `Has Role`). A user holding only the `Founder` role is denied on this dashboard via both MCP and the Desk page.
- **Gate mechanics:** every tool calls `require_dashboard_role("founder_dashboard")` as its first line, before any `frappe.call()`. This throws `frappe.PermissionError` ("You do not have access to the Founder Dashboard.") if the caller holds none of the allowed roles.
- **Backend consistency:** `founder_dashboard.py`'s own `_check_permission()` (called by the underlying Desk-side whitelisted functions) also calls `require_dashboard_role("founder_dashboard")`, in addition to its pre-existing `frappe.has_permission("Lyfe Order", "read", throw=True)` base check. This closes a gap that existed before 2026-09-10, where any role with base `Lyfe Order` read (e.g. Customer Service, Sales User) could call these functions directly and bypass the page's intended role list.
- **Two endpoints are permanently excluded from MCP, at any role:** `get_mcp_access_summary`/`get_mcp_access_drilldown` — these list which users currently hold live OAuth Bearer Tokens for this very MCP connector. No tool wraps them; surfacing "who is logged into MCP" *through* MCP would be circular and security-sensitive in its own right.
- **Legacy `get_founder_dashboard_summary` (v1)** is also excluded — superseded by `get_founder_dashboard_summary_v2`, which `get_founder_summary` (below) delegates to.

## 1.1 `get_founder_summary`

**API Details** — delegates to `founder_dashboard.get_founder_dashboard_summary_v2`.

**Functional Details** — the entire Founder Dashboard summary in one call: every tile (Sales, Margin, Best Sellers, Geography, Slow Movers, Trending Categories, Money at Risk, Stockout Risk, Board Health, PM Task Risk, Look Ahead, Production, Month-over-Month, Custom Revenue, Material Usage, Customer Concentration, Cash In) aggregated together, same as loading the full Desk dashboard page once.

**Technical Details** — `(global_filters: str | None = None) -> dict`. `global_filters` is a JSON string applying to every tile at once (the same filter rail the Desk page uses).

**Security** — System Manager/Super Admin only (see §1.0). No per-tile field stripping beyond the doctype-level permlevel masking already applied by the underlying aggregation (this is a report/aggregation endpoint, not a raw doctype record).

## 1.2 `get_founder_tile_sales`

**API Details** — delegates to `founder_dashboard.get_tile_sales`.

**Functional Details** — the Sales tile: revenue and order count for a period/channel/order-type/category filter.

**Technical Details** — `(period: str | None, channel: str | None, order_type: str | None, category: str | None) -> dict`. Returns `revenue`, `order_count`, `target`, `pct_of_target`, `paced_target`, `pct_of_paced_target`, `status`, `period_label`, `mom_pct`, `yoy_pct`.

**Security** — System Manager/Super Admin only (see §1.0).

## 1.3 `get_founder_tile_margin`

**API Details** — delegates to `founder_dashboard.get_tile_margin`.

**Functional Details** — the Margin tile: gross profit and margin % for a period/channel/order-type/category filter.

**Technical Details** — `(period, channel, order_type, category) -> dict`.

**Security** — System Manager/Super Admin only. This tile returns aggregate margin/gross-profit figures — genuinely sensitive financial data, restricted to the same two roles as everything else on this dashboard (no additional narrowing beyond the dashboard-level gate).

## 1.4 `get_founder_tile_best_sellers`

**API Details** — delegates to `founder_dashboard.get_tile_best_sellers`.

**Functional Details** — best-selling SKUs by revenue, with per-SKU margin %.

**Technical Details** — `(period, channel, order_type, category) -> dict`.

**Security** — System Manager/Super Admin only.

## 1.5 `get_founder_tile_geography`

**API Details** — delegates to `founder_dashboard.get_tile_geography`.

**Functional Details** — revenue and margin broken down by shipping country/state.

**Technical Details** — `(period, channel, order_type, category) -> dict`.

**Security** — System Manager/Super Admin only.

## 1.6 `get_founder_tile_slow_movers`

**API Details** — delegates to `founder_dashboard.get_tile_slow_movers`.

**Functional Details** — slow-moving inventory items and their tied-up inventory value.

**Technical Details** — `(period, channel, order_type, category) -> dict`.

**Security** — System Manager/Super Admin only.

## 1.7 `get_founder_tile_trending_categories`

**API Details** — delegates to `founder_dashboard.get_tile_trending_categories`.

**Functional Details** — revenue/margin trend by item category.

**Technical Details** — `(period, channel, order_type, category) -> dict`.

**Security** — System Manager/Super Admin only.

## 1.8 `get_founder_tile_money_at_risk`

**API Details** — delegates to `founder_dashboard.get_tile_money_at_risk`.

**Functional Details** — the dollar amount currently tied up in stuck/held/lost orders.

**Technical Details** — `() -> dict`. No parameters — always a current snapshot.

**Security** — System Manager/Super Admin only.

## 1.9 `get_founder_tile_stockout_risk`

**API Details** — delegates to `founder_dashboard.get_tile_stockout_risk`.

**Functional Details** — the dollar amount of open orders blocked on stockout risk, optionally scoped to one item group.

**Technical Details** — `(item_group: str | None) -> dict`.

**Security** — System Manager/Super Admin only.

## 1.10 `get_founder_tile_board_health`

**API Details** — delegates to `founder_dashboard.get_tile_board_health`.

**Functional Details** — PM task board health counts, optionally scoped to a department/priority.

**Technical Details** — `(department: str | None, priority: str | None) -> dict`.

**Security** — System Manager/Super Admin only.

## 1.11 `get_founder_tile_pm_task_risk`

**API Details** — delegates to `founder_dashboard.get_tile_pm_task_risk`.

**Functional Details** — PM/SLA task risk counts, optionally scoped to a project/department/priority.

**Technical Details** — `(project: str | None, department: str | None, priority: str | None) -> dict`.

**Security** — System Manager/Super Admin only.

## 1.12 `get_founder_tile_look_ahead`

**API Details** — delegates to `founder_dashboard.get_tile_look_ahead`.

**Functional Details** — the forward-looking revenue forecast tile.

**Technical Details** — `() -> dict`.

**Security** — System Manager/Super Admin only.

## 1.13 `get_founder_tile_production`

**API Details** — delegates to `founder_dashboard.get_tile_production`.

**Functional Details** — counts of orders currently stuck in production.

**Technical Details** — `() -> dict`.

**Security** — System Manager/Super Admin only.

## 1.14 `get_founder_tile_mom`

**API Details** — delegates to `founder_dashboard.get_tile_mom`.

**Functional Details** — month-over-month revenue/margin trend for the last N months.

**Technical Details** — `(months: int = 12, year: int | None) -> dict`.

**Security** — System Manager/Super Admin only.

## 1.15 `get_founder_tile_custom_revenue`

**API Details** — delegates to `founder_dashboard.get_tile_custom_revenue`.

**Functional Details** — Custom vs Standard order revenue and margin % comparison.

**Technical Details** — `(period, channel, category) -> dict`. Returns `custom_margin_pct`/`standard_margin_pct` fields.

**Security** — System Manager/Super Admin only.

## 1.16 `get_founder_tile_material_usage`

**API Details** — delegates to `founder_dashboard.get_tile_material_usage`.

**Functional Details** — material/BOM usage summary for the last N days.

**Technical Details** — `(period_days: int = 30) -> dict`.

**Security** — System Manager/Super Admin only.

## 1.17 `get_founder_tile_customer_concentration`

**API Details** — delegates to `founder_dashboard.get_tile_customer_concentration`.

**Functional Details** — top-customer revenue concentration and a customer × item-group matrix. Returns real customer names/revenue-share figures.

**Technical Details** — `(period, channel, customer: str | None) -> dict`.

**Security** — System Manager/Super Admin only. This is the one Founder Dashboard tile returning named customer-level concentration data (not just aggregate revenue) — still gated at the same two roles as the rest of the dashboard, no separate narrower gate.

## 1.18 `get_founder_tile_cash_in`

**API Details** — delegates to `founder_dashboard.get_tile_cash_in`.

**Functional Details** — cash collected for a period, optionally scoped by payment type/mode.

**Technical Details** — `(period, payment_type: str | None, payment_mode: str | None) -> dict`.

**Security** — System Manager/Super Admin only.

## 1.19 `get_founder_sales_drilldown`

**API Details** — delegates to `founder_dashboard.get_sales_drilldown`.

**Functional Details** — the row-level order list backing the Sales tile: order name, customer, date, revenue.

**Technical Details** — `(global_filters: str | None) -> dict`. Row-level output, not aggregate — includes real customer identity per row.

**Security** — System Manager/Super Admin only.

## 1.20 `get_founder_margin_drilldown`

**API Details** — delegates to `founder_dashboard.get_margin_drilldown` (reuses `pnl_dashboard.get_orders_detail`'s underlying logic).

**Functional Details** — the row-level order cost/margin breakdown backing the Margin tile: manufacturing cost, custom/additional charges, shipping charge, tax, total expense, profit, margin — per order.

**Technical Details** — `(global_filters: str | None) -> dict`. This is the single most financially detailed tool on this dashboard — full per-order cost breakdown, not an aggregate.

**Security** — System Manager/Super Admin only.

## 1.21 `get_founder_production_drilldown`

**API Details** — delegates to `founder_dashboard.get_production_drilldown`.

**Functional Details** — the list of orders currently stuck in production (docnames).

**Technical Details** — `() -> dict`.

**Security** — System Manager/Super Admin only.

## 1.22 `get_founder_cs_drilldown`

**API Details** — delegates to `founder_dashboard.get_cs_drilldown`.

**Functional Details** — the row-level RTO/return records for the last N days.

**Technical Details** — `(period_days: int = 30) -> dict`.

**Security** — System Manager/Super Admin only.

## 1.23 `get_founder_cash_in_drilldown`

**API Details** — delegates to `founder_dashboard.get_cash_in_drilldown`.

**Functional Details** — the row-level payment entries backing the Cash In tile: quotation, payment date/type/mode/amount.

**Technical Details** — `(global_filters: str | None) -> dict`. Row-level payment data, tied to specific quotations.

**Security** — System Manager/Super Admin only.

## 1.24 `get_founder_material_usage_drilldown`

**API Details** — delegates to `founder_dashboard.get_material_usage_drilldown`.

**Functional Details** — the row-level material usage records for the last N days.

**Technical Details** — `(period_days: int = 30) -> dict`.

**Security** — System Manager/Super Admin only.

## 1.25 `get_founder_stockout_risk_drilldown`

**API Details** — delegates to `founder_dashboard.get_stockout_risk_drilldown`.

**Functional Details** — the row-level list of open orders blocked on stockout risk.

**Technical Details** — `() -> dict`.

**Security** — System Manager/Super Admin only.

---

# 2. Customer Intelligence Dashboard

**File:** `lh/lyfe_hardware/mcp_tools/customer_intelligence.py` — 11 tools, `get_cid_*` prefix.
**Underlying Desk page:** `lh/lyfe_hardware/page/customer_intelligence_dashboard/cid/*.py` (split across `shared.py`, `founder.py`, `customers.py`, `component_gap.py`, `profile.py`).

## 2.0 Dashboard-level Security (applies to all 11 tools)

- **Allowed roles:** `System Manager`, `Super Admin` (+ `Administrator`, implicit).
- **Gate mechanics:** every tool calls `require_dashboard_role("customer_intelligence_dashboard")` first. This dashboard's underlying `cid/shared.py::_check_permission()` was, uniquely among the three dashboards, already re-checking this same role set server-side before 2026-09-10 — as of that date it was refactored to call `require_dashboard_role()` directly too, so Page `.json`, backend, and MCP all read from the one `DASHBOARD_ROLES` dict rather than three independently-maintained copies of the same role set.
- **Two write endpoints are permanently excluded from MCP, at any role:** `set_component_gap_state` (mutates a Component Gap Flag record) and `confirm_customer_type` (mutates `Customer.custom_customer_type`). No tool wraps either — this connector is read-only project-wide.
- **Field-level sensitivity stripping (2026-09-10) — the defining security feature of this dashboard's tools.** In addition to the dashboard-level role gate above, six of these eleven tools apply an *independent, per-field-category* role check before returning their result. This exists so that if the dashboard-level gate is ever widened in the future (e.g. to include `Factory` or `Customer Service`), sensitive fields do not become visible to the newly-added role automatically just because the tool call itself now succeeds — each category below still requires its own explicit role grant, checked separately:

  | Category | Fields stripped if not allowed | Allowed roles |
  |---|---|---|
  | Email / contact details | `_identity_key`, `identity_key` (these carry the customer's raw email, e.g. `"email:name@example.com"`) | Super Admin, System Manager |
  | COGS / margin | `cogs`, `cogs_resolved`, `margin`, `margin_pct` (including nested occurrences inside `matrix[category][population]` and `metrics.margin`) | Super Admin, System Manager, Factory |
  | Quotation pricing tied to a named customer | `quotations` (whole array) | Super Admin, System Manager, Customer Service |
  | Order-level history tied to identity | `orders`, `order_exceptions` | Super Admin, System Manager, Customer Service |
  | Business-sensitive, not personal | `value_at_risk`, `total_opportunity`, `opportunity_amount` | Super Admin, System Manager, Customer Service |

  **Today, in production, this is a no-op**: only Super Admin and System Manager can call these tools at all, and both are allowed in every category above, so nothing is currently stripped for any real caller. It activates automatically the moment the tool-level gate is widened — verified live by temporarily widening `DASHBOARD_ROLES["customer_intelligence_dashboard"]` in-memory (never committed) and confirming Factory sees COGS/margin but not email or quotation/order history, and Customer Service sees quotations/orders/business-$ figures but not email or COGS.
  - Stripping deletes the key entirely (`dict.pop()`) — it never sets the value to `null`/`0`, so a masked field is never confused with a genuine empty/zero value.

## 2.1 `get_cid_filter_options`

**API Details** — delegates to `cid/shared.get_filter_options`.

**Functional Details** — distinct item-category values for the dashboard's category filter dropdown.

**Technical Details** — `() -> dict`, returns `{"categories": [...]}`.

**Security** — System Manager/Super Admin only (§2.0). No field-stripping applies (no sensitive fields in this response).

## 2.2 `get_cid_founder_summary`

**API Details** — delegates to `cid/founder.get_founder_summary`.

**Functional Details** — a category × customer-population profit matrix (revenue, COGS, margin) for a period.

**Technical Details** — `(filters: str | None) -> dict`, shaped `{"matrix": {category: {population: {revenue, cogs, order_count, cogs_resolved, margin, ...}}}}`.

**Security** — System Manager/Super Admin only (§2.0). **COGS field-stripping applies**: `cogs`, `cogs_resolved`, `margin`, `margin_pct` are removed from every `matrix[category][population]` cell for a caller not in `_COGS_ROLES` (Super Admin, System Manager, Factory).

## 2.3 `get_cid_founder_trends`

**API Details** — delegates to `cid/founder.get_founder_trends`.

**Functional Details** — new-vs-returning customer revenue split, plus a 12-month LTV/repeat-rate trend.

**Technical Details** — `(filters: str | None) -> dict`.

**Security** — System Manager/Super Admin only. No field-stripping applied to this tool (contains no email/COGS/quotation/order-history fields per current inspection).

## 2.4 `get_cid_at_risk_summary`

**API Details** — delegates to `cid/founder.get_at_risk_summary`.

**Functional Details** — count and dollar value of at-risk customers, by population (e.g. B2B/Retail).

**Technical Details** — `(filters: str | None) -> dict`, keyed by population → `{"at_risk_count": int, "value_at_risk": float}`.

**Security** — System Manager/Super Admin only (§2.0). **Business-sensitive-$ stripping applies**: `value_at_risk` is removed from every population's cell for a caller not in `_BUSINESS_SENSITIVE_ROLES` (Super Admin, System Manager, Customer Service); `at_risk_count` alone is never stripped.

## 2.5 `get_cid_segment_movement`

**API Details** — delegates to `cid/customers.get_segment_movement`.

**Functional Details** — customers whose segment (population) recently changed, with what was gained/lost.

**Technical Details** — `() -> list`, each row: `display_name`, `customer`, `population`, `gained: [...]`, `lost: [...]`.

**Security** — System Manager/Super Admin only. No field-stripping currently applied to this tool (returns customer name/segment data, not email/COGS/quotation/order fields).

## 2.6 `get_cid_sample_conversion_summary`

**API Details** — delegates to `cid/customers.get_sample_conversion_summary`.

**Functional Details** — the B2B sample-order-to-real-order conversion aggregate.

**Technical Details** — `() -> dict`.

**Security** — System Manager/Super Admin only. No field-stripping applied (aggregate-only, no per-customer identity or COGS/quotation data).

## 2.7 `get_cid_customer_list`

**API Details** — delegates to `cid/customers.get_customer_list`.

**Functional Details** — the per-customer list: revenue, average order value, margin, return rate, custom-order share, last order date.

**Technical Details** — `(filters: str | None, limit: int = 200) -> list`. Each row includes `display_name`, `customer`, `population`, `segments`, `order_count`, `revenue`, `aov`, `last_order_date`, `custom_share_pct`, `return_rate_pct`, `margin`, `margin_pct`, `cogs_resolved`, `_identity_key`.

**Security** — System Manager/Super Admin only (§2.0). **Both email and COGS stripping apply** to every row: `_identity_key` (email) is removed for a caller not in `_EMAIL_ROLES`; `margin`, `margin_pct`, `cogs_resolved` are removed for a caller not in `_COGS_ROLES`.

## 2.8 `get_cid_return_outliers`

**API Details** — delegates to `cid/customers.get_return_outliers`.

**Functional Details** — customers with an unusually high return rate.

**Technical Details** — `(filters: str | None, min_orders: int = 3, outlier_threshold_pct: int = 25) -> list`.

**Security** — System Manager/Super Admin only. No field-stripping currently applied (returns `display_name`/`customer`/order and return counts — no email key, COGS, or quotation/order-history fields in the current implementation).

## 2.9 `get_cid_component_gap_summary`

**API Details** — delegates to `cid/component_gap.get_component_gap_summary`.

**Functional Details** — the total component-gap upsell opportunity amount across all flagged customers, plus counts (customers flagged, awaiting approval).

**Technical Details** — `() -> dict`, returns `{"total_opportunity": float, "customers_flagged": int, "awaiting_approval": int, "last_refreshed": ...}`.

**Security** — System Manager/Super Admin only (§2.0). **Business-sensitive-$ stripping applies**: `total_opportunity` is removed for a caller not in `_BUSINESS_SENSITIVE_ROLES`.

## 2.10 `get_cid_component_gap_list`

**API Details** — delegates to `cid/component_gap.get_component_gap_list`.

**Functional Details** — the per-customer component-gap rows: what they've bought, what's missing, and the upsell opportunity amount for each.

**Technical Details** — `(filters: str | None, include_completed: int = 0, from_date: str | None, to_date: str | None) -> list`.

**Security** — System Manager/Super Admin only (§2.0). **Business-sensitive-$ stripping applies** per row: `opportunity_amount` is removed for a caller not in `_BUSINESS_SENSITIVE_ROLES`.

## 2.11 `get_cid_customer_profile`

**API Details** — delegates to `cid/profile.get_customer_profile`.

**Functional Details** — the full profile for one customer: segments, full metrics (order count, revenue, cadence, category affinity, returns, margin), order history, and quotation history. The most complete single-customer view on this dashboard.

**Technical Details** — `(identity_key: str) -> dict`. `identity_key` is the customer's identity key as returned by `get_cid_customer_list`/`get_cid_segment_movement` (e.g. `"email:name@example.com"`). Returns `identity_key`, `display_name`, `customer`, `population`, `segments`, `metrics` (nested dict including `metrics.margin.*`), `order_exceptions`, `orders` (array: name, order_date, order_type, status, total_amount, source_quotation, order_source), `quotations` (array: name, transaction_date, grand_total, status, custom_order_type).

**Security** — System Manager/Super Admin only (§2.0). **This tool has the most extensive field-stripping of any tool in this file** — all four applicable categories:
  - `identity_key` removed if caller not in `_EMAIL_ROLES`.
  - `quotations` (entire array) removed if caller not in `_QUOTATION_ROLES`.
  - `orders`, `order_exceptions` removed if caller not in `_ORDER_HISTORY_ROLES`.
  - `metrics.margin` removed if caller not in `_COGS_ROLES`.

  An invalid or nonexistent `identity_key` (tested: bogus string, SQL-injection-style string, empty string) always produces the same sanitized generic error — no distinguishing signal between "not found" and "malformed input," confirmed by direct testing.

---

# 3. Quotation Analysis Dashboard

**File:** `lh/lyfe_hardware/mcp_tools/quotation_analysis.py` — 17 tools, `get_quotation_*` prefix.
**Underlying Desk page:** `lh/lyfe_hardware/page/quotation_analysis_dashboard/quotation_analysis_dashboard.py`.

## 3.0 Dashboard-level Security (applies to all 17 tools)

- **Allowed roles:** `System Manager`, `Sales Manager`, `Sales User`.
- **Gate mechanics:** every tool calls `require_dashboard_role("quotation_analysis_dashboard")` first.
- **Backend consistency:** `quotation_analysis_dashboard.py`'s own `_check_permission()` also calls `require_dashboard_role("quotation_analysis_dashboard")` as of 2026-09-10, in addition to its pre-existing `frappe.has_permission("Quotation", "read", throw=True)` base check — closing the same class of gap as Founder Dashboard (any role with base Quotation read, e.g. Customer Service, could previously call these functions directly, bypassing the page's intended role list).
- **No write endpoints exist on this Desk page at all** — every whitelisted method here was always read-only; nothing was excluded from MCP for write-safety reasons on this dashboard.
- **No per-field stripping on this dashboard.** `owner` (quotation creator) and `customer_name`/`grand_total` (pricing) are returned by several tools exactly as the Desk dashboard itself returns them today — a Sales User with dashboard access already sees company-wide quotation data (not scoped to their own quotations) in the Desk UI; MCP mirrors that access level, it does not narrow or widen it. Parameter manipulation (owner/date/customer filters) was tested and confirmed unable to widen a caller's visible dataset beyond what their role already sees.

## 3.1 `get_quotation_summary`

**API Details** — delegates to `quotation_analysis_dashboard.get_summary`.

**Functional Details** — quotation KPIs: total/won/lost/open count and value, win rate, average quote value, with comparison to the prior period.

**Technical Details** — `(filters: str | None) -> dict`. `filters` JSON string, e.g. supports `from_date`/`to_date`/`order_type`/`owner`/`priority` (parameterized SQL — confirmed no injection surface; unrecognized filter keys are silently ignored, not rejected).

**Security** — System Manager/Sales Manager/Sales User only (§3.0).

## 3.2 `get_quotation_funnel`

**API Details** — delegates to `quotation_analysis_dashboard.get_funnel`.

**Functional Details** — quotation counts and values by workflow stage, including Cancelled as its own stage (the one place Cancelled isn't excluded — everywhere else on this dashboard it's filtered out of every metric).

**Technical Details** — `(filters: str | None) -> list`.

**Security** — System Manager/Sales Manager/Sales User only.

## 3.3 `get_quotation_win_loss_breakdown`

**API Details** — delegates to `quotation_analysis_dashboard.get_win_loss_breakdown`.

**Functional Details** — win/loss breakdown grouped by a chosen dimension.

**Technical Details** — `(dimension: str = "region", filters: str | None) -> list`. `dimension` is one of `owner`, `territory`, `country`, `state`, `order_type`, `priority`. When `dimension="owner"`, rows are keyed by the quotation's creating user — i.e. individual salesperson performance is visible to any role with dashboard access, not scoped to "your own performance only."

**Security** — System Manager/Sales Manager/Sales User only. No additional narrowing on the `owner` dimension — matches this dashboard's stated policy of mirroring Desk-level company-wide visibility (§3.0).

## 3.4 `get_quotation_open_aging`

**API Details** — delegates to `quotation_analysis_dashboard.get_open_quotation_aging`.

**Functional Details** — open quotations bucketed by age.

**Technical Details** — `(filters: str | None) -> list`.

**Security** — System Manager/Sales Manager/Sales User only.

## 3.5 `get_quotation_review_bottlenecks`

**API Details** — delegates to `quotation_analysis_dashboard.get_review_bottlenecks`.

**Functional Details** — average and oldest days stuck, per workflow review state.

**Technical Details** — `(filters: str | None) -> list`.

**Security** — System Manager/Sales Manager/Sales User only.

## 3.6 `get_quotation_sla_violations`

**API Details** — delegates to `quotation_analysis_dashboard.get_sla_violations`.

**Functional Details** — SLA rule breach counts for quotations.

**Technical Details** — `() -> dict`. No parameters.

**Security** — System Manager/Sales Manager/Sales User only.

## 3.7 `get_quotation_followup_engagement`

**API Details** — delegates to `quotation_analysis_dashboard.get_followup_engagement`.

**Functional Details** — follow-up email open/click/win rates.

**Technical Details** — `(filters: str | None) -> dict`.

**Security** — System Manager/Sales Manager/Sales User only.

## 3.8 `get_quotation_revision_churn`

**API Details** — delegates to `quotation_analysis_dashboard.get_revision_churn`.

**Functional Details** — revision count vs. win rate — do quotations that get revised more end up winning more or less often.

**Technical Details** — `(filters: str | None) -> list`.

**Security** — System Manager/Sales Manager/Sales User only.

## 3.9 `get_quotation_payment_tracking`

**API Details** — delegates to `quotation_analysis_dashboard.get_payment_tracking`.

**Functional Details** — payment tracking: amount received, outstanding value, and payment method breakdown.

**Technical Details** — `(filters: str | None) -> dict`. Real payment/collections figures — the most financially detailed tool on this dashboard.

**Security** — System Manager/Sales Manager/Sales User only. No additional narrowing beyond the base dashboard gate — payment data on this dashboard is treated the same as the rest of its quotation data (company-wide, visible to any role with dashboard access).

## 3.10 `get_quotation_category_breakdown`

**API Details** — delegates to `quotation_analysis_dashboard.get_category_breakdown`.

**Functional Details** — quotation revenue and win rate by item category.

**Technical Details** — `(filters: str | None) -> list`.

**Security** — System Manager/Sales Manager/Sales User only.

## 3.11 `get_quotation_needs_attention`

**API Details** — delegates to `quotation_analysis_dashboard.get_needs_attention`.

**Functional Details** — row-level list of quotations flagged as needing attention (stuck, aging, at-risk), each with the specific reason(s) it was flagged.

**Technical Details** — `(filters: str | None) -> list`. Each row: `name`, `customer_name`, `workflow_state`, `grand_total`, `age_days`, `risk`, `reasons`.

**Security** — System Manager/Sales Manager/Sales User only. Returns customer name + grand_total per row, same as every other row-level tool on this dashboard — no additional restriction.

## 3.12 `get_quotation_detail`

**API Details** — delegates to `quotation_analysis_dashboard.get_quotations_detail`.

**Functional Details** — the row-level quotation list: date, customer, workflow state, order type, priority, value, who created it, revision count, linked Lyfe Orders.

**Technical Details** — `(filters: str | None, workflow_state: str | None) -> list`. Each row: `name`, `transaction_date`, `customer_name`, `workflow_state`, `order_type`, `priority`, `grand_total`, `owner`, `revision_number`, `lyfe_orders`.

**Security** — System Manager/Sales Manager/Sales User only. Verified: manipulating `filters` (owner, extreme date ranges, bogus customer/company/warehouse values) cannot widen results beyond what the caller's role already sees — an `owner` filter for a nonexistent user correctly returns 0 rows, not more; unrecognized filter keys are ignored, not exploited.

## 3.13 `get_quotation_tile_drilldown`

**API Details** — delegates to `quotation_analysis_dashboard.get_tile_drilldown`.

**Functional Details** — the row-level quotation list backing a specific summary tile/value — a generic drill-down endpoint used across the dashboard's clickable tiles.

**Technical Details** — `(tile: str | None, value: str | None, filters: str | None) -> list`. Same row shape as `get_quotation_detail` (`customer_name`, `grand_total`, `owner` included).

**Security** — System Manager/Sales Manager/Sales User only.

## 3.14 `get_quotation_stage_composition`

**API Details** — delegates to `quotation_analysis_dashboard.get_stage_composition`.

**Functional Details** — quotation counts by workflow stage, for a donut/pie chart breakdown.

**Technical Details** — `(filters: str | None) -> list`.

**Security** — System Manager/Sales Manager/Sales User only.

## 3.15 `get_quotation_value_trend`

**API Details** — delegates to `quotation_analysis_dashboard.get_value_trend`.

**Functional Details** — monthly quoted value vs. won value trend.

**Technical Details** — `(filters: str | None) -> list`.

**Security** — System Manager/Sales Manager/Sales User only.

## 3.16 `get_quotation_mom_summary`

**API Details** — delegates to `quotation_analysis_dashboard.get_mom_summary`.

**Functional Details** — month-over-month quoted/won value and win rate trend for the last N months.

**Technical Details** — `(months: int = 12, year: int | None) -> dict`.

**Security** — System Manager/Sales Manager/Sales User only.

## 3.17 `get_quotation_yesterday_activity`

**API Details** — delegates to `quotation_analysis_dashboard.get_yesterday_activity`.

**Functional Details** — quotations modified yesterday: workflow state, customer, value, and who last modified them.

**Technical Details** — `() -> list`. Each row includes `modified_by` (the user who last edited the quotation) in addition to the usual `customer_name`/`grand_total`/`workflow_state`/`order_type`/`lyfe_orders` fields.

**Security** — System Manager/Sales Manager/Sales User only. `modified_by` is returned to any role with dashboard access — same company-wide visibility policy as the rest of this dashboard (§3.0).

---

## Cross-dashboard verification summary (2026-09-10)

- **Direct-backend-API bypass testing:** for all three dashboards, the underlying whitelisted Python functions were called directly (not through MCP) as 8 different roles. 24/24 cells matched the intended policy exactly, and matched what MCP itself returns for the same role — confirming MCP and the backend enforce the identical authorization decision.
- **Parameter manipulation testing:** owner/date-range/customer/company/warehouse manipulation, malformed JSON, and SQL-injection-style filter values were tested against Quotation Analysis and Customer Intelligence tools. No combination bypassed a role gate or widened a denied/limited caller's visible dataset; query builders use parameterized placeholders throughout (no raw string interpolation of filter values found).
- **Field-stripping simulation:** Customer Intelligence Dashboard's per-category field stripping was verified against a temporarily-widened (in-memory only, never persisted) role set, confirming Factory sees COGS but not email/quotations/orders, and Customer Service sees quotations/orders/business-$ but not email/COGS — 13/13 expectations matched.
- **Audit log / error leakage:** no `Permission Denied` audit row for any of these tools carries `output_result`; invalid/nonexistent identity keys and IDs return the same generic sanitized message regardless of the underlying cause.
