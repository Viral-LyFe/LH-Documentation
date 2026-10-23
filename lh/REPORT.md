# Security audit: `lh` app (Lyfe Hardware)

**Date:** 2026-10-23
**Repo:** `apps/lh` @ `0e003915` (branch `prod`). First pass was at `6f8c2ff6`. Re-checked at `0e003915` (see "Revision" at the end). The earlier version is in `raw/REPORT_6f8c2ff6.md`.
**Method:** static review only. No bench, tests or project code were run. Frappe/ERPNext source was not audited.

**Scope:** 1,240 Python files (~215k lines)
- 521 `@frappe.whitelist` methods
- 16 `allow_guest=True` endpoints
- ~1,040 raw SQL calls
- print formats, templates, www pages, file handling, secrets in tree and git history

Detailed findings, with file:line, evidence and fixes, are in `raw/`:

| File | Covers |
|---|---|
| `raw/guest_webhooks.md` | all 16 guest endpoints and webhook receivers |
| `raw/authz_core.md` | whitelisted methods in `api.py`, `lyfe_hardware/{doctype,doc_events,shipping,integrations,report,mcp_tools,…}`, permission hooks, MCP |
| `raw/authz_pages.md` | 168 dashboard/page endpoints in `lyfe_hardware/page/` |
| `raw/authz_modules.md` | `lh_project`, `lh_hrms`, `lh_lms`, Slack action handlers |
| `raw/sqli.md` | SQL injection |
| `raw/jinja_uploads_secrets.md` | Jinja/XSS, file uploads/SSRF, hardcoded secrets |

I spot-checked every Critical finding against the source myself. All of them hold.

---

## Executive summary

**The main problem is authorization, not injection.** The SQL is in good shape: about 289 dynamically built queries were traced and every one is parameterized, allow-listed, int-cast or escaped, so no exploitable SQLi was found. Template rendering is also clean: no user-controlled template strings, and the www pages are static.

The weak point is that **a large share of whitelisted methods trust "is logged in" as the only check**. In Frappe, any enabled user can call any whitelisted method directly via `/api/method/…`, whatever Desk page or button roles say. That includes the external **Shipping Forwarder** role and every employee. Several of these methods move money, create billable labels, or redirect integrations.

The second theme is **webhooks that fail open when their secret is blank** (ShipStation, Shopify, 17Track), plus **credentials that have leaked** (a 1Click key in publicly served docs, and Shopify/1Click secrets in git history).

| Severity | Count (deduplicated) |
|---|---|
| Critical | 6 (1Click public-docs key resolved) |
| High | 15 |
| Medium | ~20 |
| Low | ~25 |

---

## Fix first (Critical)

| # | Finding | Where | Who can exploit |
|---|---|---|---|
| C1 | **Leaked 1Click keys in git history:** `zXTs…` (8df6c4d1, 1dff520f; publicly served via `/assets/lh_docs` until f9f83402) and the Postman collection keys `GbOI…`, `Ak7M…`, `OekL…` (e69a1f25). Deleting files does not revoke them. _Status: the Shopify secrets in this finding are done (client secret rotated in prod on 2026-10-23, access tokens rotate every 45 min; confirm `shpat_0217…` was not a separate custom-app token). The 1Click keys are still open._ | git history | Anyone with repo access, now or in the past |
| C2 | **ShipStation webhook has no auth when the secret is blank.** It creates or overwrites Lyfe Orders (address, items, totals, status) by `orderNumber`, with `ignore_permissions`. ShipStation v1 does not sign webhooks, so if this endpoint works in prod it is effectively unauthenticated. | `integrations/shipstation_webhook.py:19-29` | Anyone (unauthenticated) |
| C3 | **Fake payments:** `add_payment_entry` and `complete_draft_order` have no permission check. They record a payment (`ignore_permissions`), move the Quotation to Won and complete the Shopify draft order, so goods ship unpaid. | `integrations/shopify.py:597`, `:732` | Any logged-in user |
| C4 | **Shopify webhook hijack:** `register_shopify_webhooks(site_url=…)` and `deregister_shopify_webhooks` have no checks. A user can point the order, payment and refund webhooks at their own host (customer PII stream), or delete them so payments go undetected. | `integrations/shopify.py:236`, `:296` | Any logged-in user |
| C5 | **Forged order import:** `process_shipstation_response` is whitelisted with no check. It creates Lyfe Orders, Customers and Addresses (`ignore_permissions`), and new orders then start 1Click fulfilment. It is only called internally. | `doctype/shipstation_orders/shipstation_orders.py:944` | Any logged-in user |
| C6 | **Billable shipping labels:** `generate_cj_label`, `generate_cj_label_for_reshipment` and `retry_from_integration_log` have no check, and the consignee address comes from the caller's payload. | `shipping/cj_api.py:558`, `:435`; `shipping/cj_api_reshipment.py:142` | Any logged-in user |

**Immediate actions (hours, not sprints):**
1. Rotate the 1Click keys (including `zXTs…`). The Shopify client secret is done (rotated 2026-10-23).
2. Remove `@frappe.whitelist()` from `process_shipstation_response`. Add `frappe.only_for([...])` to the Shopify payment, webhook-registration and CJ label methods.
3. Make all webhook signature checks **fail closed** (`raise` when no secret is configured). `lh/lh_project/gorgias/webhook.py:_verify_secret` is the in-repo pattern to copy. Delete the ShipStation webhook if the scheduled sync is the real ingestion path.

---

## High

| # | Finding | Where | Detail |
|---|---|---|---|
| H1 | Shopify and 17Track webhooks skip verification when no secret is configured. Forged "paid" or "delivered" events become possible. Shopify also writes the expected HMAC to Error Log. | `shopify.py:2410`, `order_tracking_service.py:325` | guest_webhooks G-2/G-3 |
| H2 | **SSRF + session-cookie leak** in the `File` override: `get_content` fetches any http(s) `file_url` and sends the caller's `sid` cookie. It is reachable through `download_file` for public Files. | `overrides/file.py:112-136` | jinja H4 |
| H3 | Guest-callable `update_all_real_time_tracking` synchronously runs up to 200 17Track lookups per request. Repeated calls exhaust workers and burn API quota. | `report/shipment_analysis_dashboard/…py:200` | guest G-4 |
| H4 | About 40 Lyfe Order and accounting actions have no permission check: revert workflow, split order, bulk factory assign, set Shipped via tracking, post to 1Click, ShipStation mark-shipped, create/submit Purchase Orders and Sales Invoices, gate pass creation and `mark_orders_ready_for_dispatch`. | many; see list | core H1/H2, pages H2/L3 |
| H5 | **Mass assignment through `run_doc_method`:** `backfill_erp_items_from_item_name` and `fetch_shopify_details` save a client-built doc with `ignore_permissions`. Any Lyfe Order reader (including Shipping Forwarder) can write any field. | `doctype/lyfe_order/lyfe_order.py` | core H6 |
| H6 | **Financial data leaks:** the P&L dashboard (22 endpoints) is gated only by Lyfe Order *read*. `export_cogs_by_internal_id_range` and `factory_quotation_dashboard.get_quotation_detail` return COGS and margin to roles that the permlevel design hides them from. | `page/pnl_dashboard/pnl_dashboard.py:35`, etc. | pages H1/H4, core H4 |
| H7 | **Bulk customer PII with no check:** order lifecycle heal, shipment analysis, the status-overview leg endpoints, order exports and `generate_pick_pack_pdf`. Pick-pack also has SSRF, `../` traversal, and copies private files into `public/`. | `page/order_lifecycle_heal/…:44`, `shipping/pick_pack.py:197`, … | pages H3, core H7, jinja M4 |
| H8 | **No `has_permission` hook for Task, Project, Quotation, CRM Lead or CRM Deal.** Row rules only filter list views, so any record opens by name via `/api/resource`, forms and MCP get tools. The `hooks.py` comment claims they are mirrored; they are not. | `hooks.py:206-224`, `lh_project/permissions.py` | modules H2, core M3 |
| H9 | **Appraisal:** `recalculate_weighted_score` lets any user read any employee's hidden (permlevel 1) appraisal score and force a save with `ignore_permissions`. | `lh_hrms/points/appraisal_bridge.py:37` | modules H1 |
| H10 | **Gorgias:** `reply_to_customer` / `sync_task_from_gorgias` let a user set any `custom_gorgias_ticket_id` on their task, then read that customer's conversation or email them as the company. | `lh_project/gorgias/reply.py:24`, `recovery.py:32` | modules H3 |
| H11 | **Project board:** `add_task_to_board` lets a member pull tasks from projects they can't access onto their own board. Task names are sequential, so this can be scripted. | `lh_project/page/project_board/project_board.py:1752` | modules H4 |
| H12 | **Engineering approval bypass:** any user can call `submit_senior_review` or cancel drawing uploads. | see core H3 | core H3 |
| H13 | **Master data overwrite:** Item and BOM creation and overwrite with `ignore_permissions`. | see core H5 | core H5 |

---

## Medium (summary; details in `raw/`)

- **Quotation approval link** (guest_webhooks G-5): the key is the DB password, with a hardcoded fallback of `"lh_approval_secret"`. Links never expire, approve from any state with no state check, and change state on GET.
- `safe_eval` in Task Automation Rules receives the whole `frappe` module, so a rule author gets arbitrary DB read/write, including password hashes (`lh_project/automation/rule_engine.py:409`).
- Arbitrary S3 object read via `?key=` file URLs and the `get_s3_image_url` Jinja helper.
- Sales Invoice PDFs are saved as public files (`doc_events/sales_invoice.py:187`). Export helpers let the caller pick `is_private=0`.
- **MCP:** the masked-field filter and sort guard can be bypassed (JSON-string filters, the 4-element filter form, a backticked `order_by`), so hidden COGS can be inferred. The 4 status-overview MCP tools have no role gate.
- Status overview, returns dashboard and factory step turnaround have no backend role check, although their MCP equivalents are gated.
- `set_column_terminal` lets any member bulk-complete a column. `save_task_with_overdue_reason` copies every Task field from the client (mass assignment).
- Manager energy-point awards have no bounds. The HR dashboard's "manager" scope is department-wide, not direct reports only.
- **Slack:** disabled ERP users can still approve quotations, and the modal submission is not re-authorized. Any Slack workspace member, including guests, can look up orders. Notes from unmapped Slack users are written as Administrator.
- **Permission hooks:** `_assign LIKE '%user%'` matches substrings. External Learning Record has no list condition. The order-visibility engine allows any role it has no rule for.
- `backfill_item_group` lets dashboard viewers trigger a full-table write via GET.
- A Slack incoming-webhook URL is hardcoded in `patches/add_us_warehouse_delivered_slack_webhook.py:3`.

## Low (summary)

- `ss_api_key` and 9 Slack webhook fields are `Data`, not `Password`. (S3 `aws_secret` was fixed in `0e003915`. See Revision.)
- Tokens and HMACs are written to Error Log.
- Print formats use `|safe` on address fields (mitigated by save-time sanitizing). Several dashboard JS files build HTML from customer data without escaping.
- Slack `oauth_redirect` reflects `error` unescaped.
- The bank-transfer tracking link re-sends its email on every click.
- None of the state-changing endpoints restrict to POST, so they can be triggered from a crafted link (CSRF).
- SQL L-1: link-search queries trust a caller-supplied `doctype` for `get_match_cond`.

## Checked and clean

- **SQL injection:** none exploitable.
- **Slack and Gorgias webhooks:** fail closed. Slack also has replay protection.
- `render_template` always uses fixed template paths.
- The www bank-transfer pages are static.
- `drawing_upload.py` escapes its output.
- `mcp_scope_guard.py` logic is sound.
- These areas have adequate checks: Daily Work Log writes, the PM operations dashboard, and the 106 role-gated founder, quotation, customer-intelligence and order-analysis endpoints.
- There are no WhatsApp or 1Click inbound webhooks in this codebase.

---

## Systemic fixes (instead of patching 100+ methods one by one)

1. **Default-deny for whitelisted methods.** Add a CI check (a short AST script) that fails when a `@frappe.whitelist` function contains no call to `frappe.only_for`, `frappe.has_permission`, `doc.check_permission`, `require_dashboard_role` or an explicit `# authz: public` marker. Treat the current failures as the backlog.
2. **One webhook-verification helper** that raises when the secret is missing, used by every inbound receiver.
3. **Add `has_permission` hooks** for Task, Project, Quotation, CRM Lead and CRM Deal that mirror their `permission_query_conditions`.
4. **Add `methods=["POST"]`** to every state-changing whitelisted method.
5. **Avoid `ignore_permissions=True`** in any code path a whitelisted method reaches, unless that method has already checked the caller's role.
6. **Secrets hygiene:** no credentials in `lh/docs` or patches; don't serve `lh/docs`; add a pre-commit secret scanner such as gitleaks.

## Not verified (needs a live site)

- Which webhook secrets are actually set in prod: Oneclick Settings `webhook_secret`, Shopify Setting, Seventeen Track Settings. This decides whether C2 and H1 are open right now.
- Custom DocPerm rows. Real grants may be wider than the JSON permissions show.
- Whether Frappe sanitizes Comment HTML, and how the Slack `oauth_redirect` return value is rendered.

---

## Revision: re-check at `0e003915` (2026-10-23)

Changes since `6f8c2ff6`: one commit plus uncommitted working-tree files.

- **`0e003915` (store S3 `aws_secret` encrypted):** this improves security. `aws_secret` moves to `__Auth` via a Property Setter, and the column is masked. `S3Operations.__init__` and `delete_from_s3` now read the secret with `get_password()`. No new whitelisted entry points. `custom_delete_from_s3` is reached only from the framework's File delete path.
  - **Follow-up (Medium):** until this patch runs in prod, the AWS secret sits in plain text in `tabSingles`, so every DB backup taken before then contains it. **Rotate the AWS access key** after the patch is deployed, and give the IAM user only the access it needs to the one bucket.
  - Still open: `custom_file_upload_to_s3` keeps its unnecessary `@frappe.whitelist()` (core L7).
- **Untracked `stock_forecast/segments.py`:** an analysis script run via `bench execute`. It has no whitelisted functions, no raw SQL and no user input. Clean.
- **Untracked `print_format/sdf/__init__.py`:** empty package marker. Clean.
- **Modified `docs/*.md`:** documentation only. No secrets added.

**I re-verified all seven Critical findings against `0e003915` by parsing the functions.** None of `add_payment_entry`, `complete_draft_order`, `register_shopify_webhooks`, `deregister_shopify_webhooks`, `process_shipstation_response`, `generate_cj_label`, `retry_from_integration_log` or `generate_cj_label_for_reshipment` contains any permission call. The ShipStation webhook still skips HMAC when `webhook_secret` is blank. The `sites/assets/lh_docs` symlink still exists, and both docs still contain the 1Click key. **All Criticals remain open.**

---

## Status check: 2026-10-23, second pass

| Finding | Status | Evidence |
|---|---|---|
| Former C1 / raw H1: 1Click key in publicly served docs | **Resolved in code** (`f9f83402`). Removed from the Critical list. | Key redacted from both `lh/docs/oneclick_*_2026-08-25.md` files. Takes effect on the live site once pushed and deployed. Rotating the key is tracked under C1. |
| C1: Shopify client secret | **Rotated** (2026-10-23, reported by the team) | Note: the 45-min job rotates only the access and refresh tokens. It reads `custom_client_secret` but never writes it, so this had to be a manual rotation. |
| C1: 1Click keys (`zXTs…`, `GbOI…`, `Ak7M…`, `OekL…`) | **Open** | Ask 1Click to revoke them and issue new keys, then update Oneclick Settings → `api_key`. |

Code changes on `prod` since the audit: `0e003915` (S3 secret encrypted) and `f9f83402` (1Click key redacted from docs). This bench could not fetch `origin`, so fixes that were pushed but not pulled here are not reflected.
