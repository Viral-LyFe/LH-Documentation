# MCP API Documentation — Dashboard Tools

> Covers every MCP tool for the five dashboards below, organized dashboard-by-dashboard.
> Companion to `mcp-full-api-documentation.md`, which lists all 165 MCP tools (these 76
> included, in summary form) plus server-wide behavior common to every tool.
>
> - `apps/lh/lh/lyfe_hardware/page/founder_dashboard/`
> - `apps/lh/lh/lyfe_hardware/page/customer_intelligence_dashboard/`
> - `apps/lh/lh/lyfe_hardware/page/quotation_analysis_dashboard/`
> - `apps/lh/lh/lh_project/page/pm_operations_dashboard/`
> - `apps/lh/lh/lyfe_hardware/page/order_analysis/` (shared backend for that page and
>   `order_analysis_tw/`)
>
> Tool source: `apps/lh/lh/lyfe_hardware/mcp_tools/founder_dashboard.py`,
> `customer_intelligence.py`, `quotation_analysis.py`,
> `pm_operations_dashboard.py`, `order_analysis.py`.
>
> Last verified against source: 2026-09-11. **If you add, remove, rename, or change the
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

- **Allowed roles:** `System Manager`, `Super Admin`, `Customer Service` (+ `Administrator`, implicit). **Widened 2026-09-10** to add `Customer Service` — previously System Manager/Super Admin only. `customer_intelligence_dashboard.json`'s Page `roles` list was updated to match, and `bench migrate` run to sync `Has Role`.
- **Gate mechanics:** every tool calls `require_dashboard_role("customer_intelligence_dashboard")` first. This dashboard's underlying `cid/shared.py::_check_permission()` was, uniquely among the three dashboards, already re-checking this same role set server-side before 2026-09-10 — as of that date it was refactored to call `require_dashboard_role()` directly too, so Page `.json`, backend, and MCP all read from the one `DASHBOARD_ROLES` dict rather than three independently-maintained copies of the same role set. This means the 2026-09-10 Customer Service widening took effect on all three surfaces (Desk page, backend, MCP) from a single dict change.
- **Two write endpoints are permanently excluded from MCP, at any role:** `set_component_gap_state` (mutates a Component Gap Flag record) and `confirm_customer_type` (mutates `Customer.custom_customer_type`). No tool wraps either — this connector is read-only project-wide.
- **Field-level sensitivity stripping — the defining security feature of this dashboard's tools.** In addition to the dashboard-level role gate above, six of these eleven tools apply an *independent, per-field-category* role check before returning their result. This exists so that if the dashboard-level gate is ever widened in the future (e.g. to include `Factory`), sensitive fields do not become visible to the newly-added role automatically just because the tool call itself now succeeds — each category below still requires its own explicit role grant, checked separately:

  | Category | Fields stripped if not allowed | Allowed roles |
  |---|---|---|
  | Email / contact details | `_identity_key`, `identity_key` (these carry the customer's raw email, e.g. `"email:name@example.com"`) | Super Admin, System Manager |
  | COGS / margin (backs the Customer List tile's **Profit** column) | `cogs`, `cogs_resolved`, `margin`, `margin_pct` (including nested occurrences inside `matrix[category][population]` and `metrics.margin`) | Super Admin, System Manager |
  | Quotation pricing tied to a named customer | `quotations` (whole array) | Super Admin, System Manager, Customer Service |
  | Order-level history tied to identity | `orders`, `order_exceptions` | Super Admin, System Manager, Customer Service |
  | Business-sensitive, not personal | `value_at_risk`, `total_opportunity`, `opportunity_amount` | Super Admin, System Manager, Customer Service |

  **COGS/margin role set (2026-09-10, revised same day):** `Customer Service` was briefly added to `_COGS_ROLES` (matching the base dashboard widening), then explicitly excluded again per a follow-up instruction — Profit/margin visibility is Super Admin/System Manager only, even though Customer Service can open the dashboard and call every other tool in this file. `Factory` remains listed even though it has no tool-call access to this dashboard at all today — kept for the same "future widening" reason as elsewhere. Customer Service IS still included in `_QUOTATION_ROLES`/`_ORDER_HISTORY_ROLES`/`_BUSINESS_SENSITIVE_ROLES` — only the COGS/margin category is restricted narrower than the base dashboard gate. Verified live: a real Customer Service-role test user can open the dashboard and call `get_cid_customer_list`, but the response's rows do not contain `margin`/`margin_pct`/`cogs_resolved`; Super Admin and System Manager both still see those fields.

  The Desk page's own JS mirrors this exactly: `customer_intelligence_dashboard.js`'s `canSeeProfit()` checks only `Super Admin`/`System Manager` via `frappe.user.has_role()`, independent of the page-level role gate above — a deliberate, explicit check rather than "same as the page," so a future role added to the page's `roles` list does not automatically also see the Profit column. Customer Service can open the page and see every other Customer List column, but the Profit `<th>`/`<td>` are omitted from the rendered table entirely for that role. **Note:** this client-side restriction only hides the column in the rendered table — `cid/customers.py::get_customer_list()` (the whitelisted Desk-side function) still returns `margin`/`margin_pct`/`cogs_resolved` in its raw response to any caller who can open the page; there is no server-side masking layer on the Desk-side whitelisted method itself, only on the MCP tool's response (see the COGS/margin row above) and the rendered UI. A Customer Service-role user calling `cid/customers.get_customer_list()` directly (bypassing the Desk page's JS) would still receive the real `margin` value in the raw API response — this is a UI-layer restriction, not a data-access control, on the Desk side; the MCP tool's stripping is the actual access control for that surface.
  - Stripping deletes the key entirely (`dict.pop()`) — it never sets the value to `null`/`0`, so a masked field is never confused with a genuine empty/zero value.

## 2.1 `get_cid_filter_options`

**API Details** — delegates to `cid/shared.get_filter_options`.

**Functional Details** — distinct item-category values for the dashboard's category filter dropdown.

**Technical Details** — `() -> dict`, returns `{"categories": [...]}`.

**Security** — System Manager/Super Admin/Customer Service (§2.0). No field-stripping applies (no sensitive fields in this response).

## 2.2 `get_cid_founder_summary`

**API Details** — delegates to `cid/founder.get_founder_summary`.

**Functional Details** — a category × customer-population profit matrix (revenue, COGS, margin) for a period.

**Technical Details** — `(filters: str | None) -> dict`, shaped `{"matrix": {category: {population: {revenue, cogs, order_count, cogs_resolved, margin, ...}}}}`.

**Security** — System Manager/Super Admin/Customer Service (§2.0). **COGS field-stripping applies**: `cogs`, `cogs_resolved`, `margin`, `margin_pct` are removed from every `matrix[category][population]` cell for a caller not in `_COGS_ROLES` (Super Admin, System Manager, Factory — Customer Service is NOT in this set, despite having tool-call access to this dashboard).

## 2.3 `get_cid_founder_trends`

**API Details** — delegates to `cid/founder.get_founder_trends`.

**Functional Details** — new-vs-returning customer revenue split, plus a 12-month LTV/repeat-rate trend.

**Technical Details** — `(filters: str | None) -> dict`.

**Security** — System Manager/Super Admin/Customer Service. No field-stripping applied to this tool (contains no email/COGS/quotation/order-history fields per current inspection).

## 2.4 `get_cid_at_risk_summary`

**API Details** — delegates to `cid/founder.get_at_risk_summary`.

**Functional Details** — count and dollar value of at-risk customers, by population (e.g. B2B/Retail).

**Technical Details** — `(filters: str | None) -> dict`, keyed by population → `{"at_risk_count": int, "value_at_risk": float}`.

**Security** — System Manager/Super Admin/Customer Service (§2.0). **Business-sensitive-$ stripping applies**: `value_at_risk` is removed from every population's cell for a caller not in `_BUSINESS_SENSITIVE_ROLES` (Super Admin, System Manager, Customer Service); `at_risk_count` alone is never stripped.

## 2.5 `get_cid_segment_movement`

**API Details** — delegates to `cid/customers.get_segment_movement`.

**Functional Details** — customers whose segment (population) recently changed, with what was gained/lost.

**Technical Details** — `() -> list`, each row: `display_name`, `customer`, `population`, `gained: [...]`, `lost: [...]`.

**Security** — System Manager/Super Admin/Customer Service. No field-stripping currently applied to this tool (returns customer name/segment data, not email/COGS/quotation/order fields).

## 2.6 `get_cid_sample_conversion_summary`

**API Details** — delegates to `cid/customers.get_sample_conversion_summary`.

**Functional Details** — the B2B sample-order-to-real-order conversion aggregate.

**Technical Details** — `() -> dict`.

**Security** — System Manager/Super Admin/Customer Service. No field-stripping applied (aggregate-only, no per-customer identity or COGS/quotation data).

## 2.7 `get_cid_customer_list`

**API Details** — delegates to `cid/customers.get_customer_list`.

**Functional Details** — the per-customer list: revenue, average order value, margin (rendered on the Desk page's Customer List table as the **Profit** column), return rate, custom-order share, last order date.

**Technical Details** — `(filters: str | None, limit: int = 200) -> list`. Each row includes `display_name`, `customer`, `population`, `segments`, `order_count`, `revenue`, `aov`, `last_order_date`, `custom_share_pct`, `return_rate_pct`, `margin`, `margin_pct`, `cogs_resolved`, `_identity_key`.

**Security** — System Manager/Super Admin/Customer Service (§2.0). **Both email and COGS stripping apply** to every row: `_identity_key` (email) is removed for a caller not in `_EMAIL_ROLES` (Super Admin, System Manager); `margin`, `margin_pct`, `cogs_resolved` — the fields backing the Desk page's Profit column — are removed for a caller not in `_COGS_ROLES` (Super Admin, System Manager, Factory). **Customer Service can call this tool (base dashboard access) but never receives the Profit fields** — the one tool in this file where the field-level restriction is narrower than the tool-call gate for a role that otherwise has full access to every other field this tool returns.

## 2.8 `get_cid_return_outliers`

**API Details** — delegates to `cid/customers.get_return_outliers`.

**Functional Details** — customers with an unusually high return rate.

**Technical Details** — `(filters: str | None, min_orders: int = 3, outlier_threshold_pct: int = 25) -> list`.

**Security** — System Manager/Super Admin/Customer Service. No field-stripping currently applied (returns `display_name`/`customer`/order and return counts — no email key, COGS, or quotation/order-history fields in the current implementation).

## 2.9 `get_cid_component_gap_summary`

**API Details** — delegates to `cid/component_gap.get_component_gap_summary`.

**Functional Details** — the total component-gap upsell opportunity amount across all flagged customers, plus counts (customers flagged, awaiting approval).

**Technical Details** — `() -> dict`, returns `{"total_opportunity": float, "customers_flagged": int, "awaiting_approval": int, "last_refreshed": ...}`.

**Security** — System Manager/Super Admin/Customer Service (§2.0). **Business-sensitive-$ stripping applies**: `total_opportunity` is removed for a caller not in `_BUSINESS_SENSITIVE_ROLES`.

## 2.10 `get_cid_component_gap_list`

**API Details** — delegates to `cid/component_gap.get_component_gap_list`.

**Functional Details** — the per-customer component-gap rows: what they've bought, what's missing, and the upsell opportunity amount for each.

**Technical Details** — `(filters: str | None, include_completed: int = 0, from_date: str | None, to_date: str | None) -> list`.

**Security** — System Manager/Super Admin/Customer Service (§2.0). **Business-sensitive-$ stripping applies** per row: `opportunity_amount` is removed for a caller not in `_BUSINESS_SENSITIVE_ROLES`.

## 2.11 `get_cid_customer_profile`

**API Details** — delegates to `cid/profile.get_customer_profile`.

**Functional Details** — the full profile for one customer: segments, full metrics (order count, revenue, cadence, category affinity, returns, margin), order history, and quotation history. The most complete single-customer view on this dashboard.

**Technical Details** — `(identity_key: str) -> dict`. `identity_key` is the customer's identity key as returned by `get_cid_customer_list`/`get_cid_segment_movement` (e.g. `"email:name@example.com"`). Returns `identity_key`, `display_name`, `customer`, `population`, `segments`, `metrics` (nested dict including `metrics.margin.*`), `order_exceptions`, `orders` (array: name, order_date, order_type, status, total_amount, source_quotation, order_source), `quotations` (array: name, transaction_date, grand_total, status, custom_order_type).

**Security** — System Manager/Super Admin/Customer Service (§2.0). **This tool has the most extensive field-stripping of any tool in this file** — all four applicable categories:
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

# 4. PM Operations Dashboard

**File:** `lh/lyfe_hardware/mcp_tools/pm_operations_dashboard.py` — 9 tools, `get_pm_*` prefix.
**Underlying Desk page:** `lh/lh_project/page/pm_operations_dashboard/pm_operations_dashboard.py` (lh_project module, not lyfe_hardware — this is the one dashboard in this document that lives outside the `lyfe_hardware` module).

## 4.0 Dashboard-level Security

- **Allowed roles (8 of 9 tools):** `System Manager`, `Projects Manager`, `Founder`, `Customer Service`, `Factory` (mirrors `pm_operations_dashboard.json`'s Page `roles` list exactly, via `DASHBOARD_ROLES["pm_operations_dashboard"]`).
- **One tool is gated narrower than the rest:** `get_pm_user_efficiency` — see §4.1.
- **Gate mechanics:** every tool calls `require_dashboard_role("pm_operations_dashboard")` (or, for the one exception, `require_roles()` directly) as its first line, before any `frappe.call()`.
- **Backend has no `frappe.has_permission()` check of its own.** Unlike Founder/Quotation Analysis, `pm_operations_dashboard.py`'s whitelisted functions run raw `frappe.db.sql()` with no doctype-level permission check at all — the MCP tool file's `require_dashboard_role()` call is the only role enforcement on this surface for MCP callers. (The Desk page itself is still gated by its Page `.json` `roles` list, enforced by the Desk framework separately.)
- **Project-membership scoping is inherited, not duplicated.** `get_user_efficiency_data`, `get_project_health_scores`, and others call `lh.lh_project.permissions.accessible_projects_subquery()` internally — a caller only sees rows for projects they're a member of (or all projects, if that subquery returns empty/unrestricted for their role). This scoping happens inside the underlying Desk function itself, so it applies identically whether called via MCP or directly.
- **Per-request result caching.** Every tool except `get_pm_sla_risk`'s raw variant and `get_pm_action_center_task_list` caches its computed result in Redis for 120–180 seconds, keyed by `(tool name, all parameters, requesting user)` — so cached results are never shared across users, but two calls with identical parameters from the same user within the TTL window return the same cached payload rather than re-querying.
- **No write endpoints exist on this Desk page** — nothing excluded from MCP for write-safety reasons.
- **No per-field stripping beyond the two-tier role gate.** `get_pm_user_efficiency` returns named individual staff (`user` = a Frappe User) with per-person task/SLA/efficiency figures — the entire tool is gated to a narrower role set rather than gated broadly with per-field masking, unlike Customer Intelligence Dashboard's approach.

## 4.1 `get_pm_user_efficiency`

**API Details** — delegates to `pm_operations_dashboard.get_user_efficiency_data`.

**Functional Details** — a compact, ranked user-efficiency leaderboard: per person, open task count, overdue task count, SLA-linked task count, tasks closed in the date range, a computed `efficiency_score`, and an `overloaded`/`balanced` status flag. Named, individual staff-performance data — not an operational aggregate like this file's other 8 tools.

**Technical Details** — `(project: str | None, department: str | None, priority: str | None, date_from: str | None, date_to: str | None) -> dict`. `date_from`/`date_to` default to first-of-current-month → today if omitted (see `_resolve_dashboard_dates`). Returns `{"rows": [{user, open_tasks, overdue_tasks, sla_tasks, closed_in_range, efficiency_score, status}, ...] (top 12), "summary": {top_closer, top_efficiency, leader_score}}`. `efficiency_score = closed_in_range*5 − open_tasks*0.5 − overdue_tasks*2 − sla_tasks*1.5`; `status = "overloaded"` if `open_tasks > 12` or `overdue_tasks > 3`, else `"balanced"`.

**Security** — **Narrower than the rest of this file.** Gated by `require_roles({"Super Admin", "HR User", "HR Manager"}, "the User Efficiency leaderboard")`, not `require_dashboard_role("pm_operations_dashboard")` — Customer Service and Factory can open the Desk dashboard and call this file's other 8 tools, but cannot call this one via MCP, since it is the one tool here returning named individual-performance rankings rather than operational aggregate counts.

## 4.2 `get_pm_action_center`

**API Details** — delegates to `pm_operations_dashboard.get_action_center_data`.

**Functional Details** — six top-priority operational counts for the Action Center panel: overdue SLA tasks, blocked (dependency-waiting) tasks, escalation candidates, urgent overdue items, unassigned tasks, and inactive tasks (no activity for `days_inactive`+ days).

**Technical Details** — `(project: str | None, department: str | None, priority: str | None, days_inactive: int = 3, date_from: str | None, date_to: str | None) -> dict`. Returns `{"cards": [{key, label, count, severity, route}, ...]}` — one card per metric, `key` ∈ `sla_overdue, blocked, escalation, urgent, unassigned, inactive`; `severity` ∈ `critical, danger, warn`; `route` carries the filter values `get_pm_action_center_task_list` (§4.3) needs to drill into that same card's task list.

**Security** — System Manager/Projects Manager/Founder/Customer Service/Factory (§4.0).

## 4.3 `get_pm_action_center_task_list`

**API Details** — delegates to `pm_operations_dashboard.get_action_center_task_list`.

**Functional Details** — the row-level task list backing one Action Center card — the drill-down for whichever `key` a caller clicked/requested.

**Technical Details** — `(key: str, project: str | None, department: str | None, priority: str | None, days_inactive: int = 3, date_from: str | None, date_to: str | None) -> list`. `key` must be one of the category keys `get_pm_action_center` returns (`sla_overdue`, `blocked`, `escalation`, `urgent`, `unassigned`, `inactive`) — not independently validated against an enum beyond whatever the underlying SQL branch does with an unrecognized value.

**Security** — System Manager/Projects Manager/Founder/Customer Service/Factory (§4.0).

## 4.4 `get_pm_sla_risk`

**API Details** — delegates to `pm_operations_dashboard.get_sla_risk_data`.

**Functional Details** — active SLA violation counts, escalation risk, a daily breach trend series, and a derived trend direction (`improving`/`stable`/`increasing`, from comparing the most recent up-to-7-day window against the prior equal-length window).

**Technical Details** — `(project: str | None, department: str | None, priority: str | None) -> dict`. No date range parameters — this tool is always a current-state snapshot plus its own internal trend window. Returns `{"counts": {...}, "violations": [...], "trend": [{day, breach_count}, ...], "trend_summary": {direction, percent, current, previous}, "by_project": [...]}`.

**Security** — System Manager/Projects Manager/Founder/Customer Service/Factory (§4.0).

## 4.5 `get_pm_project_health_scores`

**API Details** — delegates to `pm_operations_dashboard.get_project_health_scores`.

**Functional Details** — a summary health-score card per project: task counts (open/closed/overdue/SLA-linked/blocked), average task age, and a single 0–100 health `score` with a derived severity status — the score is `100` minus weighted penalties for overdue %, SLA %, blocked-task count, and average aging.

**Technical Details** — `(project: str | None, department: str | None, priority: str | None, date_from: str | None, date_to: str | None) -> list`. Top 15 projects, ordered worst-first (`overdue_tasks DESC, sla_tasks DESC, open_tasks DESC`). Each row: `project, project_name, department, total_tasks, open_tasks, closed_tasks, overdue_tasks, sla_tasks, blocked_tasks, avg_age_days, completion_pct, overdue_pct, sla_pct, score, status`.

**Security** — System Manager/Projects Manager/Founder/Customer Service/Factory (§4.0).

## 4.6 `get_pm_overdue_summary`

**API Details** — delegates to `pm_operations_dashboard.get_overdue_summary`.

**Functional Details** — overdue tasks bucketed by severity, plus the worst-offender projects by overdue count.

**Technical Details** — `(project: str | None, department: str | None, priority: str | None, date_from: str | None, date_to: str | None) -> dict`. Returns `{"buckets": [...], "by_project": [...]}`.

**Security** — System Manager/Projects Manager/Founder/Customer Service/Factory (§4.0).

## 4.7 `get_pm_department_goals`

**API Details** — delegates to `pm_operations_dashboard.get_department_goals_data`.

**Functional Details** — per-project/department goal tracking: tasks created, tasks closed, completion %, overdue %, SLA %, for the date range.

**Technical Details** — `(project: str | None, department: str | None, priority: str | None, date_from: str | None, date_to: str | None) -> list`. Returns `{"rows": [...]}`.

**Security** — System Manager/Projects Manager/Founder/Customer Service/Factory (§4.0).

## 4.8 `get_pm_average_metrics`

**API Details** — delegates to `pm_operations_dashboard.get_average_metrics`.

**Functional Details** — average task Response Time (`task.creation` → first non-migration closed-stage exit) and Close Time (`task.creation` → actual/terminal-stage close), each with a trend comparison against the immediately preceding equal-length period.

**Technical Details** — `(project: str | None, department: str | None, priority: str | None, date_from: str | None, date_to: str | None) -> dict`. Returns `{response_hours, response_samples, close_hours, close_samples, response_trend, close_trend}` (trend fields carry direction + percent vs. the prior period, `None`/`0` when there are no comparable prior-period samples).

**Security** — System Manager/Projects Manager/Founder/Customer Service/Factory (§4.0).

## 4.9 `get_pm_senior_review_summary`

**API Details** — delegates to `pm_operations_dashboard.get_senior_review_summary`.

**Functional Details** — three manager-facing tiles for the Senior Drawing Review feature: request rate per engineer, outcome mix (Approved as-is vs. Changed vs. Returned), and drawing-fault rate on self-approved work.

**Technical Details** — `(date_from: str | None, date_to: str | None) -> dict`. Unlike this file's other 8 tools, does **not** accept `project`/`department`/`priority` scoping — this feature has no such dimension on its underlying doctype (Quotation-based, not Task-based); the underlying function accepts them via `**kwargs` for call-signature compatibility with the page's shared filter helper, but ignores them. Returns `{"request_rate": [...], "outcome_mix": {...}, "drawing_fault_rate": {total_orders, rejected_orders, fault_rate_pct}, "period": {date_from, date_to}}`.

**Security** — System Manager/Projects Manager/Founder/Customer Service/Factory (§4.0).

---

# 5. Order Analysis Dashboard

**File:** `lh/lyfe_hardware/mcp_tools/order_analysis.py` — 14 tools, `get_order_analysis_*` prefix. All 14 delegate to `lh.lyfe_hardware.page.order_analysis.order_analysis` via `frappe.call()`.

**Underlying Desk pages:** this is the one dashboard in this document backed by **two** Desk pages sharing one Python module — `lh/lyfe_hardware/page/order_analysis/` (`order_analysis.json`, `roles: []` — open to any Desk user) and `lh/lyfe_hardware/page/order_analysis_tw/` (`order_analysis_tw.json`, restricted to Factory/Customer Service/Super Admin/Engineer/System Manager). This MCP surface mirrors the narrower `order_analysis_tw` role list, not the open `order_analysis` one — see §5.0.

## 5.0 Dashboard-level Security (applies to all 14 tools)

- **Allowed roles:** `Factory`, `Customer Service`, `Super Admin`, `Engineer`, `System Manager` (mirrors `order_analysis_tw.json`'s Page `roles` list, via `DASHBOARD_ROLES["order_analysis"]` — deliberately the narrower of this dashboard's two Desk pages' role lists, since `order_analysis_tw` is the page people actually navigate to).
- **Gate mechanics:** every tool calls `require_dashboard_role("order_analysis")` as its first line, before any `frappe.call()`.
- **Backend has no `frappe.has_permission()` check of its own.** `order_analysis.py`'s whitelisted functions run raw `frappe.db.sql()` with no doctype-level permission check at all — the MCP tool file's `require_dashboard_role()` call is the only role enforcement on this surface for MCP callers.
- **Field-level stripping: `customer`.** Several tools return `customer` (Lyfe Order permlevel 2) per row — same real-world field as every other dashboard tool file that surfaces it (`status_overview.py`, `comparison_dashboard.py`). Stripped for a caller who holds one of the allowed roles above but lacks Lyfe Order's `customer` permlevel access, mirroring `status_overview.py`'s `_strip_sensitive_dashboard_fields()` pattern exactly. No `cost_of_goods`/COGS field appears anywhere in this file's output (confirmed by inspection) — nothing to strip there.
- **One write endpoint is permanently excluded from MCP:** `backfill_item_group` — a bulk maintenance action (mass `frappe.db.set_value()` + `commit()` across `ShipStation Order Item` rows, backfilling `item_group` from the linked Item). No tool wraps it; this connector is read-only project-wide.
- **`get_order_analysis_tat_by_category` ignores its own date parameters.** Documented per-tool below (§5.9) — it always uses a fixed trailing-3-month window regardless of what's passed, by design (smaller category buckets need a stable sample size).

## 5.1 `get_order_analysis_dashboard`

**API Details** — delegates to `order_analysis.get_dashboard_data`.

**Functional Details** — the dashboard overview: active/not-yet-delivered/out-from-factory/on-hold order counts, photo re-upload and revision counts, and three KPI percentages — Delivery Health (`on_time_delivery_rate`, factory-dispatch SLA compliance, not customer-promise compliance — see the full API doc's note), photo-rejection rate, and revision rate.

**Technical Details** — `(from_date: str | None, to_date: str | None, based_on: str | None) -> dict`. `from_date`/`to_date` default to first/last day of the current month. `based_on="Live Data"` drops the date filter for most cards; On Hold counts and the Delivery Health KPI are always a live snapshot regardless of this parameter (documented in the underlying function — gating them on the date filter would make Delivery Health zero by construction whenever "Live Data" excludes terminal-state orders).

**Security** — Factory/Customer Service/Super Admin/Engineer/System Manager (§5.0). `customer` is not present in this tool's aggregate-count response — no stripping applies here in practice.

## 5.2 `get_order_analysis_card_orders`

**API Details** — delegates to `order_analysis.get_card_orders`.

**Functional Details** — the row-level order list backing one dashboard card — the drill-down for whichever card a caller clicked. Covers roughly 39 distinct card keys spanning urgent/VIP dispatch risk, customs holds, tracking alerts, gate-pass delays, and SLA breaches.

**Technical Details** — `(card_key: str, from_date: str | None, to_date: str | None, based_on: str | None, sub_status: str | None) -> list`. `card_key` must be one of `CARD_QUERIES`' keys in the underlying module (e.g. `total_active`, `on_hold`, `breach_promised_dispatch`, `tracking_alert_possible_lost`, `slow_gate_pass_custom`, `breach_dispatch_standard` — see the tool's own docstring for the full list) — not independently validated against an enum beyond whatever the underlying SQL branch does with an unrecognized key. `sub_status` is only meaningful for the small subset of card keys that support a further breakdown.

**Security** — Factory/Customer Service/Super Admin/Engineer/System Manager (§5.0). **`customer` field-stripping applies** per row.

## 5.3 `get_order_analysis_pipeline`

**API Details** — delegates to `order_analysis.get_pipeline_data`.

**Functional Details** — order counts and average days spent per pipeline stage (CS Queue through Shipped), each with a configured day-count threshold and whether it's measured in calendar or business days.

**Technical Details** — `() -> list`. No parameters — always a live current-state snapshot, by design (documented in the underlying function: date filtering pipeline-stage counts would misrepresent "how many orders are in this stage right now"). Each row: `{stage, count, avg_days, states, threshold, day_type}`; the terminal "Shipped" row additionally carries `informational: true` and `threshold: None`.

**Security** — Factory/Customer Service/Super Admin/Engineer/System Manager (§5.0). No `customer` field in this aggregate response.

## 5.4 `get_order_analysis_pipeline_orders`

**API Details** — delegates to `order_analysis.get_pipeline_orders`.

**Functional Details** — the row-level order list for one or more pipeline workflow states — the drill-down for a `get_order_analysis_pipeline` stage. Always a live snapshot.

**Technical Details** — `(states: str, order_status: str | None) -> list`. `states` is a JSON array of `workflow_state` values, e.g. `'["Factory Assignment", "Ready for Dispatch"]'`. Throws if `states` resolves to an empty list (the underlying function requires at least one filter beyond its base Cancelled/On Hold/Merged/Return Successfully exclusion).

**Security** — Factory/Customer Service/Super Admin/Engineer/System Manager (§5.0). **`customer` field-stripping applies** per row.

## 5.5 `get_order_analysis_category_sla`

**API Details** — delegates to `order_analysis.get_category_sla_data`.

**Functional Details** — the category (item group) SLA grid: for each category, total active orders and how many are on-track / approaching / overdue against that category's configured SLA min/max thresholds (`Item Group.custom_sla_min_days`/`custom_sla_max_days`, falling back to a default 7/14-day window when unset).

**Technical Details** — `(from_date: str | None, to_date: str | None, based_on: str | None) -> list`. Uses a wider "effective start date" fallback chain (`factory_first_action` → `approved_date` → `factory_assignment_date`) than the Standard/Custom SLA cards elsewhere on this dashboard, so every active order with any of those three dates set is covered, not just factory-acknowledged ones.

**Security** — Factory/Customer Service/Super Admin/Engineer/System Manager (§5.0). No `customer` field in this aggregate response.

## 5.6 `get_order_analysis_category_sla_orders`

**API Details** — delegates to `order_analysis.get_category_sla_orders`.

**Functional Details** — the row-level order list for one category's SLA drill-down, filtered by on-track/approaching/overdue status.

**Technical Details** — `(item_group: str, status: str = "overdue", from_date: str | None, to_date: str | None, based_on: str | None) -> list`. `item_group="Other"` matches orders with no mapped item group. `status` must be `"on_track"`, `"approaching"`, or `"overdue"` — any other value is treated the same as `"overdue"` by the underlying function's `.get(status, ...)` fallback, not rejected.

**Security** — Factory/Customer Service/Super Admin/Engineer/System Manager (§5.0). **`customer` field-stripping applies** per row.

## 5.7 `get_order_analysis_category_trend`

**API Details** — delegates to `order_analysis.get_category_trend`.

**Functional Details** — order volume per item-group category, broken down by week or month — the data behind the category trend chart.

**Technical Details** — `(period: str = "monthly", from_date: str | None, to_date: str | None, based_on: str | None, lookback_months: int | None) -> list`. `period` is `"monthly"` or `"weekly"`. `lookback_months` (3, 6, or 12) overrides `from_date`/`to_date`/`based_on` when provided. `based_on="Live Data"` (with no `lookback_months`) defaults to the trailing 6 months. Each row: `{period_key, period_label, item_group, order_count}`.

**Security** — Factory/Customer Service/Super Admin/Engineer/System Manager (§5.0). No `customer` field in this aggregate response.

## 5.8 `get_order_analysis_order_type_split`

**API Details** — delegates to `order_analysis.get_order_type_split`.

**Functional Details** — Standard vs. Custom order count and total revenue for a date range.

**Technical Details** — `(from_date: str | None, to_date: str | None, based_on: str | None) -> dict`, shaped `{"standard_count", "custom_count", "standard_amount", "custom_amount"}`.

**Security** — Factory/Customer Service/Super Admin/Engineer/System Manager (§5.0). No `customer` field in this aggregate response.

## 5.9 `get_order_analysis_revenue_by_category`

**API Details** — delegates to `order_analysis.get_revenue_by_category`.

**Functional Details** — the top N item-group categories by total revenue for a date range.

**Technical Details** — `(from_date: str | None, to_date: str | None, based_on: str | None, top_n: int = 10) -> list`, largest-revenue-first.

**Security** — Factory/Customer Service/Super Admin/Engineer/System Manager (§5.0). No `customer` field in this aggregate response.

## 5.10 `get_order_analysis_order_source_breakdown`

**API Details** — delegates to `order_analysis.get_order_source_breakdown`.

**Functional Details** — order counts grouped by `order_source` (e.g. Shopify, Etsy) for a date range. Uses the same date/live-mode logic as `get_order_analysis_dashboard`.

**Technical Details** — `(from_date: str | None, to_date: str | None, based_on: str | None) -> list`.

**Security** — Factory/Customer Service/Super Admin/Engineer/System Manager (§5.0). No `customer` field in this aggregate response.

## 5.11 `get_order_analysis_tat_by_category`

**API Details** — delegates to `order_analysis.get_tat_by_category`.

**Functional Details** — the 5 Average Turnaround Time metrics (CS Review, Factory Dispatch, Photo Approval, End-to-End, Factory Upload) broken down by item-group category, each using a fallback chain through progressively earlier/looser milestone pairs since most of the narrowest fields are populated on well under 5% of orders. "Products" (the ERPNext default root category) and the unmapped "Other" bucket are excluded so only genuine product categories appear.

**Technical Details** — `(from_date: str | None, to_date: str | None, based_on: str | None) -> list`. **All three parameters are accepted for call-signature consistency with this file's other tools but intentionally ignored** — this tool always uses a fixed trailing-3-calendar-months-from-today window, by explicit design documented in the underlying function's docstring (smaller category buckets need a wider, consistent sample to produce a stable average rather than reacting to whatever ad-hoc period the rest of the dashboard happens to show). Each row also excludes any milestone-pair duration that computes negative (a known historical data-quality issue on a small number of orders, not filtered elsewhere).

**Security** — Factory/Customer Service/Super Admin/Engineer/System Manager (§5.0). No `customer` field in this aggregate response.

## 5.12 `get_order_analysis_reshipment_sla`

**API Details** — delegates to `order_analysis.get_reshipment_sla_data`.

**Functional Details** — the Reshipment SLA dashboard data: on-track/approaching/overdue counts for in-progress `Lyfe Order Reshipment` records still with the factory, plus an on-time dispatch rate for reshipments that have reached a dispatched state — split by Standard/Custom order type. Also includes a data-capture diagnostic count (`reship_missing_factory_assignment`).

**Technical Details** — `(from_date: str | None, to_date: str | None, based_on: str | None) -> dict`. The parent Lyfe Order's workflow state is deliberately **not** used to gate visibility (a reshipment's parent order is normally already Completed — that's typically why the reshipment exists). `based_on="Live Data"` drops the date filter entirely, so all currently in-progress/dispatched reshipments show regardless of creation date.

**Security** — Factory/Customer Service/Super Admin/Engineer/System Manager (§5.0). No `customer` field in this aggregate response (reshipment rows carry `lo.customer` only in the row-level drill-down tool below, not here).

## 5.13 `get_order_analysis_reshipment_sla_orders`

**API Details** — delegates to `order_analysis.get_reshipment_sla_orders`.

**Functional Details** — the row-level in-progress reshipment list for one Reshipment SLA card. `reshipment_id` (the `Lyfe Order Reshipment` docname) is included so a caller can link each row to its Reshipment document.

**Technical Details** — `(order_type: str, sub_status: str, from_date: str | None, to_date: str | None, based_on: str | None) -> list`. `order_type` must be `"Standard"` or `"Custom"`; `sub_status` must be `"on_track"`, `"approaching"`, or `"overdue"` — both throw `frappe.ValidationError` on any other value (unlike the category-SLA drill-down's silent fallback, this one validates explicitly). Limited to 500 rows, oldest-first by days elapsed.

**Security** — Factory/Customer Service/Super Admin/Engineer/System Manager (§5.0). **`customer` field-stripping applies** per row.

## 5.14 `get_order_analysis_sla_task_links`

**API Details** — delegates to `order_analysis.get_sla_task_links`.

**Functional Details** — a mapping of `{docname: {"task": ..., "status": ...}}` for every given docname that has an active `SLA Task Link` — used by other tools' row-level results to show a PM task badge alongside each order.

**Technical Details** — `(erp_docnames: str) -> dict`. `erp_docnames` is a JSON array of Lyfe Order (or other ERP doc) names, e.g. `'["LYF-SH-2026-1875"]'`. Returns `{}` for an empty input.

**Security** — Factory/Customer Service/Super Admin/Engineer/System Manager (§5.0). No `customer` field in this response (keyed by docname → task/status only).

---

## Cross-dashboard verification summary (2026-09-10)

**Scope note:** the verification pass below covers Founder, Customer Intelligence, and
Quotation Analysis only — it predates PM Operations Dashboard's (2026-09-11, §4) and
Order Analysis Dashboard's (2026-09-11, §5) addition to this document. Both were
confirmed by direct code inspection instead — PM Operations Dashboard's role gate
matches `pm_operations_dashboard.json`'s Page `roles` list and
`DASHBOARD_ROLES["pm_operations_dashboard"]` exactly (§4.0); Order Analysis
Dashboard's tools were built new in this change (not a pre-existing gap fixed), with
its role gate, `frappe.call()` delegation, and `customer` field-stripping verified
live against the actual site — a real call as Administrator returned genuine
pipeline data, a call as a role-holding-nothing test user was correctly denied with
`frappe.PermissionError`, and both calls produced the expected `MCP Audit Log` rows
(Success/row_count=12 and Permission Denied/row_count=0 respectively). Neither
dashboard was run through the same live bypass/parameter-manipulation/field-stripping
test battery as the three below.

- **Direct-backend-API bypass testing:** for all three dashboards, the underlying whitelisted Python functions were called directly (not through MCP) as 8 different roles. 24/24 cells matched the intended policy exactly, and matched what MCP itself returns for the same role — confirming MCP and the backend enforce the identical authorization decision.
- **Parameter manipulation testing:** owner/date-range/customer/company/warehouse manipulation, malformed JSON, and SQL-injection-style filter values were tested against Quotation Analysis and Customer Intelligence tools. No combination bypassed a role gate or widened a denied/limited caller's visible dataset; query builders use parameterized placeholders throughout (no raw string interpolation of filter values found).
- **Field-stripping simulation:** Customer Intelligence Dashboard's per-category field stripping was verified against a temporarily-widened (in-memory only, never persisted) role set, confirming Factory sees COGS but not email/quotations/orders, and Customer Service sees quotations/orders/business-$ but not email/COGS — 13/13 expectations matched.
- **Audit log / error leakage:** no `Permission Denied` audit row for any of these tools carries `output_result`; invalid/nonexistent identity keys and IDs return the same generic sanitized message regardless of the underlying cause.
