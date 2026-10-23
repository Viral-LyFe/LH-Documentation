# Security fix schedule: `lh` app

**Week:** Mon 26 Oct – Sat 31 Oct 2026
**Source:** `REPORT.md` and `raw/*.md` in this folder. IDs such as `C2` and `H4` refer to `REPORT.md`. IDs such as `guest G-5` or `jinja M3` refer to the finding in that raw file.
**Goal:** every Critical and High fixed and deployed by **Fri 30 Oct**. Medium and Low fixed by **Sat 31 Oct**, or written up with an agreed date.

---

## How each day works

1. Fix in code, on `prod` (or a short-lived branch merged the same day).
2. **Check every fix with a low-privilege user.** Create two test users on staging before Monday:
   - `sec-test-forwarder@…` with only the **Shipping Forwarder** role
   - `sec-test-employee@…` with only the **Employee** role

   Call the fixed method directly while logged in as one of them, for example `POST /api/method/lh.lyfe_hardware.integrations.shopify.add_payment_entry`. **Pass = 403 / PermissionError.** Then repeat as a user who *should* have access. **Pass = it still works.**
3. Mark the row in the [tracker](#tracker) and add the commit hash.
4. Deploy to staging every day. **Deploy to prod Mon evening (Criticals), Wed evening and Fri evening.**

**Main risk:** adding role checks can break buttons that legitimate users rely on. That's why the role matrix (Mon, step 0.3) comes first. After each prod deploy, smoke-test the main flows: Shopify payment → Quotation Won, CJ label generation, ShipStation sync, 1Click posting, gate pass, Daily Work Log.

---

## Mon 26 Oct: secrets, config, Criticals

**Morning: no code, can run in parallel**

| # | Task | Closes |
|---|---|---|
| 0.1 | Ask 1Click to **revoke** `zXTs…`, `GbOI…`, `Ak7M…`, `OekL…` and issue new keys. Put the new key in Oneclick Settings → `api_key`. This depends on the vendor, so send it first. | C1 |
| 0.2 | Confirm `shpat_0217…` came from the same Shopify app (dead after token rotation). If not, revoke it in Shopify Admin → Apps. | C1 |
| 0.3 | **Role matrix:** for each sensitive action, write down which roles may call it: Shopify payment/complete, webhook registration, CJ labels, order workflow/split/assign, PO/SI create, Item/BOM create, P&L/COGS view, senior review. Everything after Monday depends on this list. | prerequisite |
| 0.4 | On prod, check whether these secrets are set: Oneclick Settings `webhook_secret`, Shopify Setting client secret, Seventeen Track `webhook_security_key`. Decide whether the ShipStation webhook is used at all; the scheduled sync may be the real path. | C2, H1 (status) |
| 0.5 | Push and deploy `f9f83402` and `0e003915`. Then **rotate the AWS access key**. Add an nginx `deny` for `/assets/lh_docs/`. | C1 follow-up, Revision |
| 0.6 | Rotate the Slack incoming webhook hardcoded in `patches/add_us_warehouse_delivered_slack_webhook.py:3`, and remove the value from the patch. | jinja M8 |

**Afternoon: code (small diffs, hotfix to prod tonight)**

| # | Task | Closes |
|---|---|---|
| 1.1 | Write one `verify_webhook(secret, ...)` helper that **raises when the secret is empty**. Copy the pattern from `lh_project/gorgias/webhook.py:_verify_secret`. Use it in the ShipStation, Shopify and 17Track receivers. Stop writing the expected HMAC to Error Log. If 0.4 says ShipStation isn't used, delete that endpoint instead. | C2, H1, jinja M1, part of L2 |
| 1.2 | Remove `@frappe.whitelist()` from `process_shipstation_response`. | C5 |
| 1.3 | Add `frappe.only_for([...])` (roles from 0.3) to `add_payment_entry`, `complete_draft_order`, `register_shopify_webhooks`, `deregister_shopify_webhooks`, `generate_cj_label`, `generate_cj_label_for_reshipment`, `retry_from_integration_log`. | C3, C4, C6 |
| 1.4 | Prod deploy and smoke test. | |

---

## Tue 27 Oct: guardrail + order/accounting actions

| # | Task | Closes |
|---|---|---|
| 2.1 | **CI check:** a short AST script that fails when a `@frappe.whitelist` function has no `frappe.only_for`, `frappe.has_permission`, `doc.check_permission`, `require_dashboard_role` or `# authz: public` marker. Run it once and save the list. **That list is the exact backlog for Tue–Thu.** | Systemic 1 |
| 2.2 | Add role or doc permission checks to the ~40 Lyfe Order and accounting actions (revert workflow, split, bulk factory assign, set Shipped, post to 1Click, ShipStation mark-shipped, PO/SI create and submit). | H4, core H1/H2 |
| 2.3 | `create_gate_pass_for_order` and `mark_orders_ready_for_dispatch`: add a check and `methods=["POST"]`. | H4, pages H2/L3 |
| 2.4 | `backfill_erp_items_from_item_name`, `fetch_shopify_details`: reload the doc from the DB, check write permission, and drop `ignore_permissions`. | H5 |
| 2.5 | `submit_senior_review` and the drawing-upload cancel: restrict to the reviewer roles. | H12 |
| 2.6 | Item/BOM create and overwrite: role check, and drop `ignore_permissions`. | H13 |
| 2.7 | `update_all_real_time_tracking`: remove `allow_guest`, enqueue instead of running synchronously, add a rate limit. Rate-limit the guest tracking endpoints too. | H3, guest G-6, jinja M7, core M10 |

---

## Wed 28 Oct: data exposure (finance, PII, files)

| # | Task | Closes |
|---|---|---|
| 3.1 | P&L dashboard (22 endpoints): one role gate at the top of every method. Gate `export_cogs_by_internal_id_range` and `get_quotation_detail` on the COGS permlevel. | H6, pages H1/H4, core H4 |
| 3.2 | Order lifecycle heal, shipment analysis, status-overview leg endpoints, order exports: add a role gate. | H7, pages H3, core H7/M11 |
| 3.3 | `generate_pick_pack_pdf`: per-order permission check, allow only internal file paths (no http, no `../`), and keep output private. | H7, jinja M4 |
| 3.4 | `CustomFile.get_content`: stop fetching remote `file_url` and never send the `sid` cookie. Reject `?key=` S3 file URLs in `CustomFile` and in `get_s3_image_url`. | H2, jinja M5/L8, core M8 |
| 3.5 | Make Sales Invoice PDFs `is_private=1`. Export helpers: force private and ignore the caller's `is_private`/attach target. | jinja M6/L6 |
| 3.6 | Status overview, returns dashboard, factory step turnaround and aggregate dashboards: add the same backend gate as their MCP tools. | pages M1/M2/L4, core M7 |
| 3.7 | `backfill_item_group`: role gate + POST only. | pages M3 |
| 3.8 | Prod deploy and smoke test. | |

---

## Thu 29 Oct: projects, HR, Gorgias, MCP, Slack

| # | Task | Closes |
|---|---|---|
| 4.1 | Add `has_permission` hooks for Task, Project, Quotation, CRM Lead and CRM Deal that mirror their `permission_query_conditions`. Fix the `hooks.py` comment. | H8, Systemic 3, core M3 |
| 4.2 | Fix `_assign LIKE '%user%'` to match the exact user (JSON contains / escaped quotes). Add a list condition to External Learning Record. Make the order-visibility engine deny roles it has no rule for. | core M4/M5/L2, sqli I-1 |
| 4.3 | `recalculate_weighted_score`: allow only the appraisal owner or HR, and don't return hidden fields. | H9 |
| 4.4 | Gorgias `reply_to_customer` / `sync_task_from_gorgias`: check project membership, and don't accept a caller-set ticket id. Gorgias attachment download: send credentials only to the Gorgias host. | H10, jinja L7 |
| 4.5 | `add_task_to_board`: check read access on the source task. | H11 |
| 4.6 | `save_task_with_overdue_reason`: allow-list the fields. `set_column_terminal`: require Project Manager. Manager award: bound the points. HR dashboard: direct reports only. | modules M1–M4 |
| 4.7 | Task Automation Rules: pass `safe_eval` only the needed values, not the `frappe` module. | jinja M3 |
| 4.8 | MCP: block JSON-string and 4-element filters and backticked `order_by` on masked fields. Add role gates to the 4 status-overview tools. | core M1/M2 |
| 4.9 | Slack: re-check that the ERP user is enabled on modal submit. Order lookup only for mapped users. Unmapped users get rejected, not written as Administrator. | modules M5/M6 |
| 4.10 | Arbitrary workflow state with write permission only: validate transitions. | core M6 |

---

## Fri 30 Oct: Mediums left, Lows, hardening

| # | Task | Closes |
|---|---|---|
| 5.1 | Quotation approval link: an HMAC key from site config (not the DB password, no hardcoded fallback), an expiry, a state check, and approve on POST from a confirm page instead of on GET. Stop the tracking-link email from resending on every click. | guest G-5, jinja M2, guest G-6 |
| 5.2 | Add `methods=["POST"]` to every state-changing whitelisted method. Use the 2.1 list. | Systemic 4, pages L5 |
| 5.3 | Change `ss_api_key` and the 9 Slack webhook fields from `Data` to `Password`, using the same Property Setter + `__Auth` patch as `encrypt_s3_aws_secret`. | jinja L1 |
| 5.4 | Remove the remaining secrets and tokens from Error Log messages. | jinja L2 |
| 5.5 | Escape output: `|safe` address fields in print formats, desk JS HTML builders, HTML email bodies, Slack `oauth_redirect` `error`, `add_factory_comment` mention label. | jinja L3/L4/L5, guest G-7, pages L1 |
| 5.6 | Link-search queries: ignore the caller-supplied `doctype`. | sqli L-1 |
| 5.7 | Small items: remove `@frappe.whitelist()` from `custom_file_upload_to_s3`. Fix `merge_orders_from_list` committing before confirm, the `start_revision` / `create_and_start_timer` checks, read-only controller methods that write, the name/count info leaks, and Order Analysis revenue vs permlevel 5. | core L1/L3/L4/L5/L7, pages L2, jinja L9 |
| 5.8 | Write-helper checks and paid-API quotas left over from 2.1. | core M9/M10 |
| 5.9 | Add a gitleaks pre-commit hook and a CI step. Turn the 2.1 AST check from a report into a failing check. | Systemic 1, 6 |
| 5.10 | Prod deploy and smoke test. | |

---

## Sat 31 Oct: verify and close

| # | Task |
|---|---|
| 6.1 | Re-run the static audit on the new HEAD (Claude can do this) and update `REPORT.md`. |
| 6.2 | Retest every Critical and High on prod with the two test users. |
| 6.3 | Anything not done: write the reason and a new date in the tracker. |
| 6.4 | Optional, only after every leaked key is rotated: rewrite git history to remove the secrets, force-push, everyone re-clones, and ask GitHub Support to purge cached views. |

---

## Tracker

| ID | Finding (short) | Day | Owner | Commit | Verified (403 as low-priv user) |
|---|---|---|---|---|---|
| C1 | Leaked 1Click keys in git history | Mon | | vendor | |
| C2 | ShipStation webhook fail-open | Mon | | | |
| C3 | Fake payments | Mon | | | |
| C4 | Shopify webhook hijack | Mon | | | |
| C5 | Forged ShipStation import | Mon | | | |
| C6 | Billable CJ labels | Mon | | | |
| H1 | Shopify / 17Track webhooks fail-open | Mon | | | |
| H2 | File override SSRF + cookie leak | Wed | | | |
| H3 | Guest mass 17Track trigger | Tue | | | |
| H4 | ~40 order/accounting actions unguarded | Tue | | | |
| H5 | `run_doc_method` mass assignment | Tue | | | |
| H6 | P&L / COGS leak | Wed | | | |
| H7 | Bulk PII + pick-pack | Wed | | | |
| H8 | No `has_permission` for Task/Project/Quotation/CRM | Thu | | | |
| H9 | Appraisal score read/write | Thu | | | |
| H10 | Gorgias ticket id / reply | Thu | | | |
| H11 | Project board task pull | Thu | | | |
| H12 | Senior review bypass | Tue | | | |
| H13 | Item/BOM overwrite | Tue | | | |
| M | Quotation approval link | Fri | | | |
| M | `safe_eval` gets `frappe` | Thu | | | |
| M | S3 `?key=` read | Wed | | | |
| M | Public Sales Invoice PDFs | Wed | | | |
| M | MCP masked-field bypass + ungated tools | Thu | | | |
| M | Status overview / returns dashboards ungated | Wed | | | |
| M | Task mass-assign, column terminal, points, HR scope | Thu | | | |
| M | Slack approval / lookup | Thu | | | |
| M | `_assign LIKE`, ELR list, visibility default-allow | Thu | | | |
| M | `backfill_item_group` | Wed | | | |
| M | Hardcoded Slack webhook | Mon | | | |
| M | AWS key rotation after S3 patch | Mon | | | |
| L | Data → Password fields | Fri | | | |
| L | Secrets in Error Log | Fri | | | |
| L | Escaping (print, JS, email, OAuth) | Fri | | | |
| L | POST-only for state changes | Fri | | | |
| L | Link-search `doctype` | Fri | | | |
| L | Misc small items (5.7) | Fri | | | |

---

## If the week slips

The order is by risk, so cut from the bottom. Critical and High (Mon–Wed, plus H8–H11 on Thu) must ship this week. The Friday Lows can move to the next week without much added risk. The CI check (2.1, 5.9) is what stops new unguarded methods appearing in the meantime.
