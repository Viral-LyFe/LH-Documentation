# Authorization audit — whitelisted methods under `lh/lyfe_hardware/page/`

Scope: 168 `@frappe.whitelist()` methods in 19 files (read-only review, no code run).
Base path for all file refs: `apps/lh/lh/lyfe_hardware/page/`.
Callable by any logged-in user as `/api/method/lh.lyfe_hardware.page.<page>.<module>.<fn>`.
None are `allow_guest`. None declare `methods=[...]`, so every method (writes included) also accepts GET.

Background that affects severity:
- Desk Page `roles` lists do not protect these methods. `integrations/mcp_audit.py:147-152` also notes the `Has Role` rows for several pages are empty in the DB, so `Page.is_permitted()` returns True for everyone. The Desk page gate is not reliable either.
- Lyfe Order DocPerm (`doctype/lyfe_order/lyfe_order.json`): permlevel 0 read is held by System Manager, Customer Service, Factory, Shipping Forwarder and Engineer, plus Founder via `patches/grant_founder_role_lyfe_order_read.py`. Sensitive fields are behind permlevels: 2 = `customer`, `cj_consignee_*`, refund fields; 3 = `ship_to_*`; 4 = `cost_of_goods`, `shipping_cost`, `batch_landed_cost` (Factory and System Manager only); 5 = `total_amount`, `paid_amount`, `source_order_id`; 7 = `customer_email`, `phone_no`. Raw SQL and `get_all` skip all of these.
- `docs/mcp-role-guide.md` says the intended policy is: Customer Service has no cost data, Engineer has no cost data and no customer contact, and Shipping Forwarder sees shipping data only.
- Custom DocPerm rows in the DB were not inspected. Real grants may be wider than the JSON shows.

## Inventory by guard status

| Group | Methods | Guard | Verdict |
|---|---|---|---|
| founder_dashboard.py | 46 | `_check_permission()` (Lyfe Order read + `require_dashboard_role("founder_dashboard")` = SM/Super Admin), `_check_stock_permission()` (SM/SA/Factory), `_check_super_admin_permission()` for MCP-access tiles | OK |
| quotation_analysis_dashboard.py | 17 | `_check_permission()` (Quotation read + SM/Sales Manager/Sales User) | OK (no COGS/PII returned) |
| customer_intelligence_dashboard/cid/*.py | 13 | `_check_permission()` (SM/SA/CS), COGS gated by `COGS_ROLES`, writes add doctype write checks | OK |
| order_analysis.py | 25 | `require_dashboard_role("order_analysis")` (Factory/CS/SA/Engineer/SM) | OK, except M3 and L2 |
| order_analysis_tw.py | 2 | `get_tat_by_category_steps` checks directly. `get_category_sla_data_by_parent` (:151) has no own check but calls `get_category_sla_data`, which runs `_check_permission()` | OK |
| lyfe_stock_po_planner.py | 3 | `has_permission("Item","read")`; PO write goes through doc perms | OK (Item read is broad, but the data is stock only) |
| pnl_dashboard.py | 22 | `_check_pnl_permission()` = **Lyfe Order read only** | **H1** |
| factory_quotation_dashboard.py | 8 | Reads: Quotation read only. Save: role check | **H4**, L1 |
| lyfe_orders_status_overview.py | 9 | **none** (field masking only, on 4 of them) | **H2, H3, M1, L3** |
| order_lifecycle_heal.py | 1 | **none** | **H3** |
| shipment_analysis_da/shipment_analysis_ds.py | 1 | **none** | **H3** |
| factory_step_turnaround.py | 2 | **none** | **M2** |
| returns_dashboard.py | 2 | **none** | **M2** |
| comparison_dashboard.py | 8 | **none** | L4 |
| item_data_completeness.py | 9 | **none** (COGS column hidden by role) | L4 |

Patterns not found: no `frappe.get_doc(json.loads(arg))` mass-assignment, no `doc.update(user_dict)`, and no caller-controlled doctype or fieldname reaching `get_all`/`get_value`/`set_value`. `item_data_completeness.get_missing_items` interpolates `field` into SQL, but only after checking it against the `_visible_fields()` allowlist (:158).

---

## HIGH

### H1 — The P&L dashboard's 22 endpoints only require Lyfe Order read, so cost, margin, revenue and customer names reach roles that must not see them
- Files: `pnl_dashboard/pnl_dashboard.py:35` (gate). Endpoints at :295, 372, 506, 704, 729, 1129, 1291, 1394, 1467, 1547, 1578, 1620, 1663, 1722, 1798, 1855, 1953, 1997, 2044, 2089, 2138, 2200.
- Methods: `lh.lyfe_hardware.page.pnl_dashboard.pnl_dashboard.get_summary`, `get_orders_detail`, `get_margin_trend`, `get_waterfall`, `get_top_customer`, `get_customer_item_group_matrix`, `get_repeat_customers_list`, and the other 15 listed above.
- What a low-privilege user can do: any Customer Service, Factory, Engineer, Shipping Forwarder or Founder user can pull every order's revenue (`total_amount`, permlevel 5), manufacturing cost (`cost_of_goods`, permlevel 4), landed cost, custom charges, margin, and customer names (permlevel 2). The page's own roles are System Manager, Accounts Manager and Accounts User. The MCP layer for the same data is Super Admin only (`mcp_tools/pnl.py`). This contradicts the documented "Customer Service: no cost data", "Engineer: no cost data" and "Shipping Forwarder: shipping only" policy. Shipping Forwarder may be an external party. The email column is the only field with its own gate (`_check_email_permission`, :41).
- Evidence:
  ```python
  def _check_pnl_permission():
      ...
      frappe.has_permission("Lyfe Order", "read", throw=True)
  # get_orders_detail SELECT: lo.customer, lo.total_amount AS revenue, lo.cost_of_goods AS mfg_cost, ... batch_landed_cost
  ```
- Fix: add a `"pnl_dashboard"` entry to `DASHBOARD_ROLES` = {System Manager, Super Admin, Accounts Manager, Accounts User} and call `require_dashboard_role("pnl_dashboard")` inside `_check_pnl_permission()`. Optionally also require permlevel-4/5 read: `4 in frappe.get_meta("Lyfe Order").get_permlevel_access("read")`.

### H2 — `create_gate_pass_for_order` lets any logged-in user add any order to today's Gate Pass
- File: `lyfe_orders_status_overview/lyfe_orders_status_overview.py:1191`
- Method: `lh.lyfe_hardware.page.lyfe_orders_status_overview.lyfe_orders_status_overview.create_gate_pass_for_order`
- What a low-privilege user can do: any logged-in user, with no role at all on Lyfe Order or Gate Pass, can push any Lyfe Order into today's draft Gate Pass. They can also create that Gate Pass if none exists, and rebuild existing rows. `Lyfe Order.create_gate_pass_if_not_exists()` (`doctype/lyfe_order/lyfe_order.py:441, ~518`) uses `insert/save(ignore_permissions=True)` and then commits. Gate Pass submission creates Sales Invoices and moves orders to Awaiting Tracking, so this corrupts the dispatch flow. The method accepts GET, so a single crafted link is enough (CSRF-able).
- Evidence:
  ```python
  @frappe.whitelist()
  def create_gate_pass_for_order(order_name):
      order = frappe.get_doc("Lyfe Order", order_name)
      gp_name = order.create_gate_pass_if_not_exists()
      frappe.db.commit()
  ```
- Fix: `@frappe.whitelist(methods=["POST"])`; `order.check_permission("write")`; `frappe.has_permission("Gate Pass", "create", throw=True)`. Also add a role gate that matches the Lyfe Order JS rule (the Gate Pass button is hidden for Shipping Forwarder, per TEST_CASES EC-14a).

### H3 — Order and customer data dumps that any logged-in user can call (no role check, page `roles: []`)
Any logged-in user, including HR User, Employee or Projects User with no Lyfe Order access, can enumerate orders together with customer names (permlevel 2) and tracking numbers:
- `order_lifecycle_heal/order_lifecycle_heal.py:44` `...order_lifecycle_heal.order_lifecycle_heal.get_dashboard_data`. Returns every non-cancelled Lyfe Order with no LIMIT: `name, internal_order_id, customer, status, order_date, tracking_status_desc, tracking_number`. It also takes a `search_text` LIKE filter on customer, so a caller can search orders by a customer's name. The unbounded query also makes it a cheap DoS.
- `shipment_analysis_da/shipment_analysis_ds.py:73` `...shipment_analysis_da.shipment_analysis_ds.get_dashboard_data`. Same query shape (`customer`, `tracking_number`, `tracking_status_desc`, customer LIKE search, :56-104).
- `lyfe_orders_status_overview/lyfe_orders_status_overview.py:878` `get_order_legs`, `:967` `get_active_order_legs`, `:1015` `get_split_delivery_cards`. These return `lo.customer`, carrier and tracking number (:913-930, :1054, :1090, :1142) and do not call `_strip_sensitive_dashboard_fields()` (returns at :964, :1012, :1183). Customer names are therefore unmasked even for users without permlevel 2.
- Evidence (lifecycle heal):
  ```python
  rows = frappe.db.sql(f"""SELECT name AS order_name, internal_order_id, customer, status AS order_status,
         order_date, tracking_status_desc, tracking_number, modified FROM `tabLyfe Order` {where_clause} ...""")
  ```
- Fix: add `frappe.has_permission("Lyfe Order", "read", throw=True)` plus a role gate (`require_dashboard_role`, with new keys for these pages) at the top of each method. Pass leg and split-card results through `_strip_sensitive_dashboard_fields()`. Add a LIMIT to the lifecycle heal and shipment analysis queries.

### H4 — `get_quotation_detail` returns per-item COGS to every Quotation reader
- File: `factory_quotation_dashboard/factory_quotation_dashboard.py:328` (gate at :43, data at :306-309)
- Method: `lh.lyfe_hardware.page.factory_quotation_dashboard.factory_quotation_dashboard.get_quotation_detail`
- What a low-privilege user can do: `_check_permission()` only checks Quotation read. Sales User, Sales Manager and Maintenance users therefore get `Quotation Item.cost_of_goods` through `frappe.get_all`, which ignores permlevel. `patches/separate_quotation_item_cogs_permlevel.py` moved that field to permlevel 2 specifically to hide it from those roles (Factory, System Manager and Super Admin only). The method also never checks the specific Quotation by name, so User Permission restrictions on Quotation are bypassed. The other read methods in this file (:96, :176, :191, :220, :478) share the same weak gate. The page's roles are System Manager and Factory.
- Evidence:
  ```python
  def _check_permission():
      frappe.has_permission("Quotation", "read", throw=True)
  ...
  header["items"] = frappe.get_all("Quotation Item", ..., fields=["name","item_code","item_name","qty","cost_of_goods"])
  ```
- Fix: change `_check_permission()` to `frappe.only_for(["Factory", "System Manager", "Super Admin"])`, or a `"factory_quotation_dashboard"` entry in `DASHBOARD_ROLES`. Add `frappe.get_doc("Quotation", quotation_name).check_permission("read")` in `get_quotation_detail` and `add_factory_comment`.

---

## MEDIUM

### M1 — The status overview's main read endpoints have no role or doctype check, and their field masking misses financial fields
- File: `lyfe_orders_status_overview/lyfe_orders_status_overview.py:197` `get_orders`, `:637` `get_reshipments`, `:740` `get_active_reshipments`, `:808` `get_active_returns`
- What a low-privilege user can do: any logged-in user gets up to 2000 orders, reshipments or returns. `_masked_dashboard_fields()` (:21) strips only `customer` and the cost fields. It does not strip `total_amount` (permlevel 5, :376), `source_order_id` (permlevel 5), `erp_sales_invoice`, `refund_amount`/`refund_status` (:833), or `return_reason`. As a result Factory, Engineer, Shipping Forwarder and roleless users all see order values and refund amounts.
- Fix: add `frappe.has_permission("Lyfe Order","read",throw=True)` plus a role gate. Add `total_amount`, `source_order_id`, `refund_amount` and the other `refund_*` fields to `_SENSITIVE_ROW_FIELDS`.

### M2 — Returns dashboard and factory step turnaround have no backend check, while their MCP equivalents are gated
- `returns_dashboard/returns_dashboard.py:28` `get_returns_summary` returns the 20 most recent returns with `customer` and `refund_amount` (:71-77). `:93` `get_return_reship_breakdown` returns counts only. `DASHBOARD_ROLES["returns_dashboard"]` exists (`mcp_audit.py:199`) but these Desk methods never call it.
- `factory_step_turnaround/factory_step_turnaround.py:60` `get_orders` returns every order in the date range with `customer`, status and step timestamps (`_select_list`, :27). `:171` `get_step_summary` returns aggregates. Page roles are SM, Factory and CS. Any logged-in user can call both.
- Fix: `require_dashboard_role("returns_dashboard")`; add a `"factory_step_turnaround"` entry in `DASHBOARD_ROLES` and call it.

### M3 — `backfill_item_group`: dashboard viewers can trigger a full-table bulk write, and it accepts GET
- File: `order_analysis/order_analysis.py:3679`
- Method: `lh.lyfe_hardware.page.order_analysis.order_analysis.backfill_item_group`
- What a low-privilege user can do: any Factory, CS or Engineer user (read-dashboard roles) can run `frappe.db.set_value` on every ShipStation Order Item row, synchronously and in one request, then commit. The values are derived (Item.item_group), so the main impact is DB load / DoS and silently overwriting manual `item_group` edits. It accepts GET.
- Evidence: `_check_permission()` then a loop of `frappe.db.set_value("ShipStation Order Item", r.name, "item_group", ig, update_modified=False)` and `frappe.db.commit()`.
- Fix: `frappe.only_for("System Manager")` and `methods=["POST"]`. Better: turn it into a patch or an enqueued job.

---

## LOW

### L1 — `add_factory_comment`: no per-document check; mention label not escaped
- `factory_quotation_dashboard/factory_quotation_dashboard.py:499`. Any Quotation reader can post a Comment on any Quotation by name, using `insert(ignore_permissions=True)` (:536) with no check on that specific document. `mention_users` is not limited to Factory users. `_mention_span` (:495) puts the target user's `full_name` into HTML unescaped, so a user who sets their full name to markup gets it stored inside Comment content. Frappe's HTML sanitisation of Comment.content was not verified.
- Fix: `frappe.get_doc("Quotation", quotation_name).check_permission("read")`; wrap the label in `frappe.utils.escape_html(label)`; filter `mention_users` against `_factory_users()`.

### L2 — Order Analysis revenue goes to roles without permlevel 5
- `order_analysis/order_analysis.py:3733` `get_order_type_split` (SUM(total_amount), :3750) and `:3778` `get_revenue_by_category` (:3818) return revenue figures to Factory and Engineer, who have no permlevel-5 (`total_amount`) read. The data is aggregated, not per order. Decide whether this is intended. If not, gate these two methods to SM/SA/Accounts roles.

### L3 — `mark_orders_ready_for_dispatch` has no explicit check, takes a bulk list, and accepts GET
- `lyfe_orders_status_overview/lyfe_orders_status_overview.py:1200`. It relies on `order.save()`, which enforces doc write permission and Frappe's `validate_workflow` role and transition rules, so this is not a direct bypass. However: it is CSRF-able through GET; per-order exception text (including PermissionError messages) is echoed back; and it commits partial batches. Fix: `methods=["POST"]`; `order.check_permission("write")` up front.

### L4 — Aggregate and item-master dashboards have no backend gate (MCP has one)
- `comparison_dashboard/comparison_dashboard.py:124, 139, 154, 168, 182, 219, 262, 329`: on-time counts, aging, stage timing and holds. Aggregates only, but the `customer=` filter lets a caller check whether a given customer has orders, and how many.
- `item_data_completeness/item_data_completeness.py:92, 98, 155, 211, 281, 300, 345, 408, 477`: item master completeness, including `retail_price`. COGS is already filtered by `_can_see_cogs()` (:80). Inputs are allowlisted.
- Fix: `require_dashboard_role("comparison_dashboard")` and `require_dashboard_role("item_data_completeness")`. Both keys already exist in `DASHBOARD_ROLES`.

### L5 — None of the state-changing methods restrict the HTTP verb
None of these declare `methods=["POST"]`: `create_gate_pass_for_order`, `mark_orders_ready_for_dispatch`, `backfill_item_group`, `save_factory_quotation_edits` (:340), `add_factory_comment` (:499), `set_component_gap_state` (`cid/component_gap.py:206`), `confirm_customer_type` (`cid/profile.py:111`), `create_or_update_lyfe_po` (`lyfe_stock_po_planner.py:25`). Frappe does not CSRF-check GET, and SameSite=Lax cookies are sent on top-level GET navigation. H2 is the one where this matters most.

### Informational
- `save_factory_quotation_edits` (:340) saves with `ignore_permissions=True` when the caller is System Manager without Factory/Super Admin (:451-457). This is gated by an explicit role check and documented as intended.
- Founder `get_tile_data_quality` / `get_data_quality_detail` are visible to all stock-dashboard roles by default (`data_quality_public_view` defaults to on, :1121). This is intended.

## Summary
- Critical: 0. High: 4 (H1–H4, about 34 methods). Medium: 3 (M1–M3, 9 methods). Low: 5 groups plus info.
- Properly gated: founder (46), quotation analysis (17), CID (13), order analysis (25+2), PO planner (3) = 106 of 168.
- Unguarded entirely: 32 methods (status overview 9, lifecycle heal 1, shipment analysis 1, factory step turnaround 2, returns 2, comparison 8, item data completeness 9).
- Weakly gated (doctype read only, while returning restricted fields): pnl (22) and factory quotation reads (6).
