# SQL Injection Audit: `lh` app (read-only)

**Scope:** `/home/frappe/frappe-bench/apps/lh/lh` (Python). Frappe 15.121.1 is treated as trusted.
**Date:** 2026-10-23

## Method

1. AST scan of every `.sql / .sql_list / .multisql / .sql_ddl` call (about 1,039 call sites; `frappe.db.sql` text matches about 1,087).
   - 750 have a constant query string.
   - 289 build the query dynamically: 189 f-strings, 83 variable-held queries, 13 `.format()`/`.replace()`, and 4 `+`/`%`.
2. Intra-procedural taint scan. Function parameters and the variables derived from them are marked as tainted. The scan flags every f-string, `.format()` or `%` that holds SQL keywords and interpolates a tainted value. About 380 hits came back and each was triaged by hand.
3. I traced every dynamically built query in `page/*.py`, `report/*.py`, `doc_events`, `lh_project`, `lh_hrms`, `customer_intelligence`, `forecasting`, `stock_forecast`, `utils`, `integrations` (webhooks) and `mcp_tools` back to its whitelisted entry point, `execute(filters)` or webhook.
4. Other vectors checked:
   - `frappe.get_all/get_list` with `fields=`, `order_by=` or `group_by=` taken from input.
   - `frappe.qb` raw strings and `CustomFunction` (none used).
   - `permission_query_conditions` hooks.
   - Link-search queries (`searchfield`).
   - `allow_guest` endpoints.
   - Query Reports (none have stored SQL).
5. Out of scope: `patches/`, `scripts/` and `tests/`. They have no request entry point. Their interpolation uses constants or table names from code.

## Confirmed findings

**No exploitable SQL injection was found.** No Critical, High or Medium findings.

### L-1 (Low): Link-search queries take match conditions from a caller-supplied `doctype`, which bypasses Lyfe Order row visibility

This is SQL built from untrusted input, but the impact is an authorization bypass, not injection.

- **Files:**
  - `lh/lyfe_hardware/doctype/material_issue_for_order/material_issue_for_order.py:352-395` (`query_lyfe_order_for_issue`)
  - the same file, `:399-418` (`query_lyfe_order_reshipment_for_issue`)
- **Snippet:**
  ```python
  @frappe.whitelist()
  @frappe.validate_and_sanitize_search_inputs
  def query_lyfe_order_for_issue(doctype, txt, searchfield, start, page_len, filters):
      ...
      """ ... and ({text_cond}) {mcond} ...""".format(text_cond=text_cond, mcond=get_match_cond(doctype))
  ```
- **Tainted input:** the `doctype` request argument, at entry point `/api/method/lh.lyfe_hardware.doctype.material_issue_for_order.material_issue_for_order.query_lyfe_order_for_issue` (and `..._reshipment_for_issue`).
- **Why it is not injection:**
  - `searchfield` is checked by `sanitize_searchfield`, which rejects any `\W` character (`frappe/database/schema.py:8`).
  - `start` and `page_len` are cast with `cint`.
  - `txt` is bound as `%(txt)s`.
  - `doctype` must be the name of an existing DocType. `get_match_cond` builds the conditions with Frappe itself.
- **The actual problem:** the row-level filter (`mcond`) is computed for whatever `doctype` the caller passes, not for `Lyfe Order` / `Lyfe Order Reshipment`. If the caller passes `doctype=Item`, or any other DocType with no permission-query conditions for that user, `mcond` is `""`. The `apply_visibility_conditions` filter from Order Visibility Settings (Factory/warehouse/ToDo scoping, Shipping Forwarder status scoping) is then skipped. The function also has no `frappe.has_permission` check of its own.
- **Who can reach it:** any logged-in Desk user.
- **What leaks:** `name` and `internal_order_id` of every issuable Lyfe Order or open Reshipment, including orders the caller's visibility rule should hide.
- **Fix:**
  ```python
  frappe.has_permission("Lyfe Order", "read", throw=True)
  ... .format(text_cond=text_cond, mcond=get_match_cond("Lyfe Order"))
  ```
  For the reshipment query, use `"Lyfe Order Reshipment"` in both places.

### I-1 (Informational): `_` and `%` act as wildcards inside escaped LIKE values in permission-query hooks

- **Files:**
  - `lh/lyfe_hardware/doc_events/quotation.py:1842`
  - `lh/lyfe_hardware/doc_events/crm_lead.py:14`
  - `lh/lyfe_hardware/doc_events/crm_deal.py:14`
- **Snippet:**
  ```python
  f"`tabQuotation`.`_assign` LIKE {frappe.db.escape('%' + user + '%')}"
  ```
- **Assessment:** `frappe.db.escape` quotes the value, so this is not injection. However, a `_` in the session user's email matches any single character. For example, `john_doe@x` would also match assignments to `johnXdoe@x`.
- **Impact:** negligible, because the value is the session user, not free input.
- **Optional fix:** escape `_` and `%` before wrapping, or match on the JSON form `'%"' + user + '"%'`.

## Needs review (taint plausible, not confirmed as SQLi)

1. **`lh/lyfe_hardware/mcp_tools/generic_list.py:107-131` (`list_any_doctype`):** passes a caller-supplied `order_by` (and `filters`) straight to `frappe.get_list`.
   - Fieldnames are validated, and `_require_no_masked_filter_or_sort` parses `order_by`.
   - Safety against SQL in `order_by` depends entirely on Frappe's `DatabaseQuery` order_by/group_by sanitizer. This is the same exposure as the standard `/api/resource` endpoint.
   - Acceptable if the framework is trusted. Recommended hardening: accept only `<fieldname> asc|desc` with a regex such as `^[a-z0-9_]+( (asc|desc))?$`.
2. **Second-order sources:**
   - Values read from settings documents and interpolated into SQL are all either escaped or `int()`-cast:
     - Item Group SLA days: `int()` cast, item-group names via `frappe.db.escape`.
     - Order Visibility Settings role states: `frappe.db.escape`.
     - SLA Rule `extra_filters_json`: keys checked with the `^[a-zA-Z_][a-zA-Z0-9_]*$` regex, operator allow-listed (`normalize_filter_operator`), values bound.
     - SLA Rule `filter_operator`: allow-listed.
   - Nothing to fix. Re-check if anyone adds a new settings-driven fragment.

## Safe patterns (aggregate, not listed per call)

All 289 dynamic queries fall into one or more of these patterns:

- **Interpolated module constants:**
  - Status tuples
  - `_CATEGORY_ITEMS_SUBQ`, `_WIP_AGE_EXPR`, `IDENTITY_KEY_SQL`
  - `landed_cost.sql_*` helpers with literal aliases
  - `_biz_days` / `_factory_days` with literal column names
  - `RETURN` / `RESHIP` table names
- **Allow-list lookups:**
  - order_analysis: `card_key in CARD_QUERIES`, `order_type in ("Standard","Custom")`, `sub_status`/`route`/`bucket` checked against fixed sets, `order_dir` either "ASC" or "DESC"
  - quotation dashboard: `dimension` via `allowed_dims.get(...)`
  - item_data_completeness: `field` / `fields` filtered against `TRACKED_FIELDS`
  - lyfe_order_profit_and_loss: `group_by` via `group_expr_map.get(...)`
  - pnl: `granularity` maps to constant formats
  - stock_planner: `flag` is one of two literals
- **WHERE builders that only append constant fragments with bound parameters:**
  - `%(x)s` / `%s`, with IN-lists built from generated placeholders (`%(order_status_0)s, ...`) or a bound tuple (`IN %(item_group)s`)
  - LIKE values bound as `%(q)s` with the `%...%` added in Python
  - Builders covered: pnl, founder, quotation, CID `shared._build_where`, returns, comparison, shipment, status-overview page and report, order_lifecycle_heal, factory_step_turnaround, pm_operations `_scope_filters`, project_board, stage_tracker, forecasting, appraisal_bridge, senior_review_dashboard
- **Integer coercion:** `int()` / `cint()` before interpolation, for example `_sanitize_limit`, `top_n`, `lookback_months`, `days_inactive` and `row_limit`.
- **`frappe.db.escape()`:** used for user/session values in permission-query hooks, `accessible_projects_subquery`, and `names_sql` in the status-overview export.
- **Table names:** `tab{doctype}` only takes code literals, in `oneclick_posting._claim_email_flag` and `missing_dimensions` (the latter gated by `_REQUIRED_FIELDS`).
- **Webhooks (`allow_guest`):** ShipStation, Shopify, 17Track, Gorgias and Slack. No payload value reaches SQL text; every query is constant with bound parameters.

## Out-of-scope observation (hand off to the authorization audit)

These whitelisted functions run raw `frappe.db.sql`, which bypasses DocPerm, and have no explicit role or permission gate in the function body. The list comes from a heuristic scan. Some apply `apply_visibility_conditions` instead of a gate (marked †).

- `doc_events/senior_review_dashboard.py`: `get_request_rate_per_engineer`, `get_outcome_distribution`, `get_drawing_fault_rate_self_approved`
- `page/factory_step_turnaround/factory_step_turnaround.py:61` `get_orders`
- `page/item_data_completeness/item_data_completeness.py` (all 8 endpoints)
- `page/order_lifecycle_heal/order_lifecycle_heal.py:45` `get_dashboard_data`
- `page/shipment_analysis_da/shipment_analysis_ds.py:74` `get_dashboard_data` †
- `page/lyfe_orders_status_overview/lyfe_orders_status_overview.py`: `get_orders` †, `get_reshipments`, `get_active_reshipments`, `get_active_returns`, `get_order_legs`, `get_active_order_legs`, `get_split_delivery_cards`
- `page/returns_dashboard/returns_dashboard.py`: `get_returns_summary`, `get_return_reship_breakdown`
- `page/comparison_dashboard/comparison_dashboard.py`: `get_open_orders_aging`, `get_lead_time_trend`, `get_factory_stage_timing`, `get_hold_analysis`
- `report/lyfe_orders_status_overview_report/...:405` `get_status_chart_data` †
- `doctype/lyfe_order/lyfe_order.py:8246` `get_mifo_count`
- `doctype/material_issue_for_order/material_issue_for_order.py`: `get_items_from_lyfe_order`, `get_prior_issues_for_order`, plus the two search queries in L-1
- `lh_project/task_status.py`: `get_project_statuses`, `get_task_kanban_counts`
- `overrides/s3_override.py:70` `custom_file_upload_to_s3`

Also: `report/shipment_analysis_dashboard/shipment_analysis_dashboard.py:200` `update_all_real_time_tracking` is `allow_guest=True`. Any anonymous caller can trigger up to 200 outbound carrier-tracking API calls and Lyfe Order writes. This is not SQLi, but it is a guest-reachable abuse and DoS path.

## Summary

| Severity | Count |
|---|---|
| Critical | 0 |
| High | 0 |
| Medium | 0 |
| Low | 1 (L-1) |
| Informational | 1 (I-1) |
| Needs review | 2 |
