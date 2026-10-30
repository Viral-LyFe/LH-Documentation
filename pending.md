# Security audit fixes: lh app — status

**Asana:** [Security audit fixes: lh app (26–31 Oct)](https://app.asana.com/1/1204809689822175/project/1211921346208625/task/1219812548257717)
**Plan:** `LH Documentation/lh/SCHEDULE.md` · **Findings:** `LH Documentation/lh/REPORT.md`
**Last updated:** Fri 30 Oct 2026

✅ Completed · ⏳ Pending · ⚠️ Done, but needs a follow-up

## Summary

| | Completed | Pending |
|---|---|---|
| Criticals (C1–C6) | C1–C6 (C1: Shopify token and 1Click keys revoked) | None |
| Highs (H1–H13) | H1, H3, H5, H6, H9, H10, H11, H12, H13 (code); H2 part A; H7 (pick-pack, dashboards, exports); H4 partly; H8 closed by decision | H4 (the remaining writers found by the scan, 2.2), H2 part B (S3 `?key=`) |
| Mediums / Lows | 0.6; 5.1 (code, partly) | All others |

**Pushed (30 Oct):** every lh commit up to `f30f3d9f` (audit 4.7) is on `origin/prod` (checked against the local `origin/prod` ref; `git fetch` can't run here without credentials). Nothing is waiting to push, including the H2 part A, H7, H10, H4 batches 1 to 3, 3.5, 3.7 and 4.7 commits. Two patches need a migrate on deploy: H10 (`restrict_task_gorgias_ticket_id`) and the Order Leg read patch (3.2). Everything is deployed (per you: deploys run every morning and end of day, so pushed code counts as deployed). The after-deploy checks below are still to run.

---

## Mon 26 Oct: secrets, config, Criticals

### Non-code steps

| # | Task | Status | Notes |
|---|---|---|---|
| 0.1 | Ask 1Click to revoke the 4 leaked keys and issue a new one | ✅ Completed | Leaked keys revoked and a new token issued (30 Oct) |
| 0.2 | Find where the `shpat_0217…` token came from | ✅ Completed | It's an old value of `Shopify Setting.password`. The token is rotated every 45 minutes (`shopify_token_rotation.rotate_shopify_token`) and expires after 1 hour, so the old one no longer works. |
| 0.3 | Role matrix | ✅ Completed | |
| 0.4 | Check prod webhook secrets; is the ShipStation webhook used? | ✅ Completed | 17Track key and Shopify secrets are set. ShipStation webhook isn't used, so it was deleted (C2). |
| 0.5 | Rotate the AWS access key | ⏳ Pending | `0e003915` and `f9f83402` are already deployed. The nginx deny isn't possible on Frappe Cloud and isn't needed anymore. |
| 0.6 | Rotate the Slack webhook and remove it from the patch | ✅ Completed | URL removed from the patch in `026bb5c5`. The old #order-via-us-warehouse webhook (`B0C2XQ3QKNV`) was revoked on 28 Oct; Slack now returns `no_service` (404) for it. Nothing in the ERP used it, so no settings needed updating. The Oneclick Settings field is kept for future use. |

### Code

| # | Finding | Status | Commit | What changed |
|---|---|---|---|---|
| C1 | Leaked 1Click keys in the repo | ✅ Completed | `6f8c2ff6`, `f9f83402`, `8e7f2410` | No key prefixes left in the current code. Keys revoked and new token issued (0.1). |
| — | S3 secret stored encrypted | ✅ Completed | `0e003915` | |
| C2 | ShipStation webhook fail-open | ✅ Completed | `1e1be4a7` | Endpoint deleted; it was never used |
| H1 | Shopify / 17Track webhooks fail-open | ✅ Completed | `1e1be4a7` | Both reject requests when the secret is blank. The expected HMAC is no longer logged. |
| C5 | Forged ShipStation import | ✅ Completed | `6a977c35` | `process_shipstation_response` is no longer whitelisted |
| C3 | Fake payments | ✅ Completed | `9368226f` | `add_payment_entry` is no longer whitelisted. `complete_draft_order` allows only CS and System Manager, from Bank Transfer Selected or Partially Paid. Tested in the browser. |
| C4 | Shopify webhook hijack | ✅ Completed | `753af930` | `register_` / `deregister_shopify_webhooks`: System Manager only |
| C6 | Billable CJ labels | ✅ Completed | `b63895e8` | `generate_cj_label`, `generate_cj_label_for_reshipment`, `retry_from_integration_log`: Factory, Founder, Super Admin, System Manager |
| 1.1 | Shared `verify_webhook` helper | ✅ Completed | `1e1be4a7` | Done inline in each receiver; a shared helper isn't needed. The ShipStation webhook was deleted, and Shopify and 17Track each reject a blank secret and compare signatures in constant time. Each provider signs differently, so there's no logic to share. New webhooks are covered by the 2.1 / 5.9 CI check. |
| — | 17Track Integration Request logging | ✅ Completed | `ed0404af` | `17Track-Webhook` / `17Track-Scheduler` |
| 1.4 | Prod deploy and smoke test | ✅ Deployed (per you) · ⏳ smoke test | | Everything up to `8d79ed5f` is deployed. The smoke test is the checklist under Testing and verification (items 6 and 9). |

---

## Tue 27 Oct: CI guardrail + order/accounting actions — ⏳ not started

| # | Task | Status | Current state in code |
|---|---|---|---|
| 2.1 | AST CI check for whitelisted methods with no permission check | ⚠️ Partly | | One-off scan run on 29 Oct (not in the repo yet): 482 whitelisted methods, **205 have no visible check**, 46 of them high priority (write with `ignore_permissions`, or call an outside service). Still to do: save the list, add the script to the repo, and make CI fail on new ones (5.9). |
| 2.2 | ~40 Lyfe Order and accounting actions (H4) | ⚠️ Partly: deployed | `df92578f` | Role gate (Factory, Customer Service, System Manager, Super Admin) added to 13 order APIs: `save_dispatch_again`, `create_gate_pass`, `bulk_assign_to_factory`, `bulk_create_gate_pass`, `split_order`, `create_load_for_order`, `create_purchase_order_for_order`, `backfill_erp_items_from_item_name`, `fetch_shopify_details`, `create_gate_pass_if_not_exists`, `track_order_validate`, status overview `create_gate_pass_for_order` and `mark_orders_ready_for_dispatch`. `ignore_permissions` removed from Lyfe Order updates and creation, Gate Pass and Purchase Order saves (Factory has those permissions; per you, Customer Service, Super Admin and System Manager were also granted Gate Pass / Purchase Order create in Role Permission Manager on prod, not checked from here). Three server-only functions are no longer APIs. `tracking.track_order` gated (`acd99e30`). Done 30 Oct (`f0ab2241`, `22037fda`): outside-service calls (`send_quotation_email`, `create_draft_order`: write on the Quotation; `retry_create_order_from_log`: order action roles; `mark_order_as_shipped` unchanged by decision) and document / stock actions (Sales Invoice, Packing List, box catalogue sync: Factory; Bulk Transfer Batch from a Gate Pass: Factory + Customer Service; the four Bulk Transfer Batch pull / fetch methods: read on the orders and reshipments used; Transfer Order resync: Factory, mark delivered: Customer Service; the three US Warehouse Replenishment steps: Factory + Customer Service; Reshipment gate pass: Factory; `detect_merge_candidates` no longer an API; System Manager and Super Admin allowed everywhere). Material Issue for Order (batch 3): `add_item_by_barcode` and `create_material_receipt_for_row` need write on the MIFO, the three `resolve_or_create_*` Item creators need create on Item (System Manager and Super Admin never restricted). Still to do: the other writers found by the scan (project board, timers, ShipStation sync, shipping helpers, validation, tracking). `revert_workflow_state` was done earlier (`8d79ed5f`). Automated callers checked 29 Oct: the scheduler runs as Administrator (passes every gate); the one hook-driven path is `create_gate_pass_if_not_exists` from `on_update`, which runs as the user moving the order (Factory or Customer Service) or, for the US-warehouse tracking step, as the scheduler. No gated function is reachable from a Guest webhook. `tracking.track_order` is also gated (`acd99e30`). |
| 2.3 | `create_gate_pass_for_order`, `mark_orders_ready_for_dispatch` (H4) | ⚠️ Partly | `df92578f` | Role gate added (Factory, Customer Service, System Manager, Super Admin). Still to do: POST only. |
| 2.4 | `backfill_erp_items_from_item_name`, `fetch_shopify_details` (H5) | ⚠️ Deployed | `1ffe7b1e` | Both are no longer API endpoints (`@frappe.whitelist()` and the role gate removed). Nothing used them: the Fetch Shopify Details button is hidden with no way to show it, backfill has no button, and no code, script, job or database record refers to either. Python and `bench` can still call them. Tested through `run_doc_method` with a forged order as four roles: all refused, database unchanged. To check on prod: search the Frappe Cloud request logs for the two method names (zero hits = nothing outside used them). |
| 2.5 | `submit_senior_review`, `request_senior_review` and the drawing APIs (H12) | ⚠️ Deployed | `f7a9f6b8` | `submit_senior_review`, `request_senior_review`, `cancel_drawing_upload`, `mark_drawing_change_pending` and `submit_drawing_change_details` now require write access to that Quotation (Super Admin always passes), so any user with Quotation access can still use them (per you). Before, anyone could approve any Senior Review or clear a quotation's drawing. Tested: Engineer, Senior Engineer, Customer Service, Factory and System Manager allowed; Automation (read-only), Customer and Item Manager users refused; `test_drawings` passes. Update 30 Oct (`016295d4`): `submit_senior_review` is no longer an API (no caller; tests call it directly). Open: the Senior Review Log form still lets an Engineer complete a review directly, so there is no role-based segregation of duties. |
| 2.6 | Item/BOM create and overwrite: `parse_bom_from_drawing`, `confirm_bom_from_drawing` (H13) | ⚠️ Deployed | `369d6dcf`, `49f6f9c2` | Both now require Engineer, Factory, System Manager or Super Admin (per you) and access to the order (read for parse, write for confirm). `ignore_permissions` removed from the Item and BOM writes: Engineer, Factory and System Manager hold Item create and Lyfe BOM write (tested: create and overwrite work for them, Customer Service refused, nothing left behind). Super Admin has no Lyfe BOM write, so it works only if it is also System Manager. Overwriting an existing BOM stays allowed and unlogged (your decision). The three `resolve_or_create_*_item` Material Issue functions now create Items with the user's own permissions (Factory, System Manager and Customer Service can; an account without Item create is refused; tested with the insert intercepted). |
| 2.7 | `update_all_real_time_tracking` (H3) | ⚠️ Deployed | `efc88a26` | No longer an API (it was `allow_guest=True`); only the background job queued by the Real-Time Tracking button runs it. `start_real_time_tracking_job` stays open to every logged-in user, with one run at a time. The 17Track webhook is unchanged (checked: signed push 200, bad signature 401). Still to do: check the Frappe Cloud request logs for any outside caller of the old endpoint. |

## Wed 28 Oct: finance, customer data, files — ⏳ not started

| # | Task | Status | Current state in code |
|---|---|---|---|
| 3.1 | P&L dashboard, COGS export, `get_quotation_detail` (H6) | ⚠️ Deployed, partly | `8d79ed5f` | P&L (22 endpoints) now follows the page's role list: System Manager and Super Admin. COGS exports (`export_cogs_by_internal_id_range`, `export_reshipment_cogs_by_date_range`): Factory, System Manager, Super Admin. `get_quotation_detail` still to do. |
| 3.2 | Lifecycle heal, shipment analysis, status-overview legs, exports (H7) | ⚠️ Deployed | `61cc5da0`, `5f4ffb09` | Lifecycle and shipment dashboards, order legs, active legs and split-delivery cards require read on Lyfe Order (before, any logged-in user got customer names and tracking numbers); output unchanged (44 outputs compared with the baseline). Super Admin got Order Leg read (patch, needs migrate). Exports: `prepare_pdf` is no longer an API; both `export_orders` require Lyfe Order read and save only private, unattached files (the screens never sent the attach / public arguments). Tested: users without Lyfe Order read refused; Customer Service, Factory, System Manager, Super Admin still get Excel and PDF. Not changed: non-admin users only get orders assigned to support@chandakbrothers.net (hardcoded, existing behaviour). |
| 3.3 | `generate_pick_pack_pdf` (H7) | ⚠️ Deployed | `8f956d99` | Requires read access to the order (`has_permission` read, which applies User Permissions and the order-visibility rules); before, any user could pull any order's customer name and address by guessing its name. The private PDF is still saved with `ignore_permissions` so read-only users can use it. Tested: roles with access get the PDF; accounts without Lyfe Order access are refused; a Factory user is refused on an order hidden from them. Still open under H7: lifecycle heal, shipment analysis, status-overview legs and exports (3.2). |
| 3.4 | `CustomFile.get_content`, S3 `?key=` (H2) | ⚠️ Part A deployed; Part B pending | `1f073e60` | Part A: `get_content` no longer fetches `http(s)` or `/api/` URLs and no longer sends the `sid` cookie (it raises a clear error instead). Reproduced the bug locally with a listener (logged-in cookie leaked; the public `download_file` made the server call it with no login) and confirmed the same test now returns an error and the listener stays silent. Checked: attaching links, existing Drive and Shopify link Files (refused cleanly), a local file and an S3 file still read. Part B (pending): S3 `?key=` reads in `get_content` and `get_s3_image_url` must resolve to an existing File the user can read; check the ShipStation re-link flow and uploads first. After deploy: download an S3 attachment, a local file and open a link attachment on prod; check the Error Log for File errors. |
| 3.5 | Sales Invoice PDFs private | ⚠️ Deployed | `78dbd84a` | New invoice PDFs are saved as private files. Existing PDFs keep their old flag (patch not done). The S3 `generate_file?key=` link still doesn't check the File, so this is fully effective only after 3.4 Part B. |
| 3.6 | Status overview / returns / aggregate dashboards gate | ⏳ Pending | |
| 3.7 | `backfill_item_group`: role gate + POST only | ⚠️ Deployed | `4b1d0824` | Removed instead: the Sync Item Groups button on Order Analysis (TW) is gone and the function is no longer an API. Nothing else called it. Python / `bench` can still run it. |
| 3.8 | Prod deploy and smoke test | ⏳ Pending | |

## Thu 29 Oct: projects, HR, Gorgias, MCP, Slack — ⏳ not started

| # | Task | Status | Current state in code |
|---|---|---|---|
| 4.1 | `has_permission` for Task, Project, Quotation, CRM Lead/Deal (H8) | ✅ Closed by decision, no change (30 Oct) | | Tested on the local copy: the five row rules only shorten lists, and records can still be opened by name (Task 25 of 25 hidden ones, Project 3 of 3, Quotation 25 of 25 for the outside Senior Engineer account `production@cbdesigns.in`, which then returned a customer name and total). You decided this is by design: the rules exist to show each user only the relevant tasks and orders, not for confidentiality, and the document-level permission still applies. Risk accepted. The misleading comment above `has_permission` in `hooks.py` (it claimed these rules block direct form and API access) was removed on 30 Oct. |
| 4.2 | Exact-match `_assign`; ELR list condition; visibility deny-by-default | ⏳ Pending | |
| 4.3 | `recalculate_weighted_score` (H9) | ⚠️ Deployed | `c9a49cf6` | No longer an API endpoint (option A, per you). It had no check, so any user could read the hidden score and force a save. Nothing used it: no button, script or job; the HR dashboard says scores are never shown. The score is still recalculated on every Appraisal save. Tested: refused for Engineer, Customer Service, Factory, System Manager and Guest. |
| 4.4 | Gorgias reply/sync (H10) | ⚠️ Deployed | `b517ae90` | `Task.custom_gorgias_ticket_id` is now permission level 1: only Customer Service, System Manager and Super Admin can read or set it (it stays hidden on the form), so no other user can point a Task at another customer's ticket through the API. New patch `restrict_task_gorgias_ticket_id` (Task permission rows 11 to 14, additive); the field setting is re-applied on every migrate. The integration is unaffected (database write). `reply_to_customer` and `sync_task_from_gorgias` are unchanged. The Gorgias buttons now show only for those three roles. Tested through the API with real users. After deploy: open a Gorgias-linked Task as Customer Service (buttons) and as an Engineer (no buttons). |
| 4.5 | `add_task_to_board` (H11) | ⚠️ Deployed | `7fd178fc` | The projection service now requires access to the task's home project (members; System Manager and Administrator pass; tasks with no project fall back to Task read permission). Before, any board member could pull tasks from projects they had no access to, by guessing sequential names. Tested: outsider refused; member, Administrator and System Manager allowed; 216 multiboard tests pass. Super Admin without System Manager follows the board's existing rule (project members only). |
| 4.6 | Task field allow-list, `set_column_terminal`, points, HR scope | ⏳ Pending | No checks |
| 4.7 | `safe_eval` without the `frappe` module | ⚠️ Deployed | `f30f3d9f` | `frappe` removed from the legacy condition globals. The one condition in the database (TAR-0016) doesn't use it. Tested locally: valid conditions still work; `frappe.db.sql` and `frappe.get_site_config` are refused and logged. |
| 4.8 | MCP masked-field bypass; status-overview tool gates | ⏳ Pending | |
| 4.9 | Slack: enabled-user re-check, mapped users only | ⏳ Pending | |
| 4.10 | Validate workflow transitions | ⏳ Pending | |

## Fri 30 Oct: remaining Mediums, Lows, hardening — ⏳ not started (except 5.1, partly done)

| # | Task | Status | Current state in code |
|---|---|---|---|
| 5.1 | Quotation approval link | ⚠️ Deployed, partly | `be7e3c9a` | Approval and tracking links are signed only with the site-config key `approval_link_secret` (set on prod, checked 64 characters); no fallback. Approval links expire after 45 days. Old links keep working for quotations created up to 29 Oct. Still to do: check it on prod (testing item 9); approve on a confirm page instead of a plain GET; blank the old tracking tokens kept in `Quotation.custom_tracking_token` when the old-key code is removed. |
| 5.2 | `methods=["POST"]` on state-changing methods | ⚠️ Mostly done: 24 + 67 POST-only, 15 un-whitelisted | `b8a94fe4` | About 114 of 474 whitelisted functions look state-changing; only 2 had `methods=`. Done: the 24 button-driven write APIs in `lyfe_order.py` are POST-only (GET and PUT refused, POST allowed; every JS caller uses `frappe.call`). Also done (`4924aa55`): 13 write APIs with no caller (plus `submit_senior_review`, `016295d4`, and `register_shopify_webhooks`, `c95dcabb`) anywhere (confirmed by you: no outside callers) are no longer APIs (`@frappe.whitelist()` removed, functions kept): `mark_order_as_shipped`, `sync_customer`, `retry_from_integration_log`, `sync_from_all_gate_passes`, `create_gate_pass_for_order` (status overview), `track_order_validate`, `add_item_by_barcode`, `create_packing_list`, `pull_factory_items` (still called from Python), `retry_create_order_from_log`, `deregister_shopify_webhooks`, `get_monthly_snapshot`, `reopen_daily_work_log`. Kept as APIs: `get_s3_image_url` (9 print formats use it); decided: `custom_file_upload_to_s3` stays whitelisted (per you). Also done (`a6786aa2`): 67 more state-changing APIs with a JS caller are POST-only, in 35 files (checked: GET and PUT refused, POST allowed; all JS callers use `frappe.call` or `open_url_post`). Not changed on purpose: the two `export_print_format_excel` download links (opened by GET URL), the three email-link endpoints and the webhooks. Not done in `lyfe_order.py`: 5 writers with no JS caller (`bulk_assign_to_factory`, `bulk_create_gate_pass`, `revert_split`, `post_order_after_us_warehouse_delivery`, `create_purchase_order_for_order`): check the Frappe Cloud request logs for outside callers first. An earlier 29-endpoint attempt (`d7a9cfe8`) was reverted (`8479367b`). Still to decide: the 5 `lyfe_order.py` writers with no JS caller, and the few remaining no-caller writers (`custom_file_upload_to_s3`). Writers the scan missed (calls through helpers) are not covered. Keep GET on `approve_quotation`, `track_email_open`, `track_quotation_link`; webhooks last or skip. Risk: outside tools with an API key that send GET (Testing item 12). |
| 5.3 | `ss_api_key` and Slack webhook fields → Password | ✅ Closed, no change (30 Oct) | | Decision: not needed. These fields don't reveal confidential information, so they stay `Data`. |
| 5.4 | Remove secrets from Error Log messages | ⏳ Pending | |
| 5.5 | Escaping: print formats, JS, email, OAuth | ⏳ Pending | |
| 5.6 | Link search ignores caller `doctype` | ⏳ Pending | |
| 5.7 | Small items | ⏳ Pending | `custom_file_upload_to_s3` stays whitelisted (decided 30 Oct, still needed); `merge_orders_from_list`, `start_revision`, `create_and_start_timer` have no checks |
| 5.8 | Leftover write-helper checks, paid-API quotas | ⏳ Pending | |
| 5.9 | gitleaks + failing AST check | ⏳ Pending | No gitleaks step in `.pre-commit-config.yaml` |
| 5.10 | Prod deploy and smoke test | ⏳ Pending | |

## Sat 31 Oct: verify and close — ⏳ not started

| # | Task | Status |
|---|---|---|
| 6.1 | Re-run the static audit and update `REPORT.md` | ⏳ Pending |
| 6.2 | Retest every Critical and High on prod with the two test users | ⏳ Pending |
| 6.3 | Reason and new date for anything not done | ⏳ Pending |
| 6.4 | Optional: scrub secrets from git history | ⏳ Pending |

---

## Testing and verification

| # | Check | Status |
|---|---|---|
| 1 | Prod deploy of the Monday fixes and the 29 Oct commits | ✅ Deployed (per you) |
| 2 | 403 test with `sec-test-forwarder` / `sec-test-employee` on prod | ⏳ Pending. So far tested only on `lyfelocal.com`: C3 in the browser; C4 and C6 by script (Shipping Forwarder, CS and Employee blocked; Factory and System Manager allowed) |
| 3 | 17Track test push creates a `17Track-Webhook` Integration Request | ⏳ Pending |
| 4 | Error Log watch: "Secret Not Configured", "Key Not Configured", "failed to create Integration Request" | ⏳ Pending |
| 5 | Shopify webhooks added by hand (Settings → Notifications) | ⏳ Pending. Only the 3 API-registered webhooks were checked. |
| 6 | Smoke test: Shopify payment → Won, CJ label (as a Factory user), ShipStation sync, 1Click posting | ⏳ Pending |
| 7 | `execute_via_http` in `patches/generate_fake_orders_2000.py` broken by C5 | ✅ Done. The load-test scripts (`generate_fake_orders_2000`, `step2_infra_sizing_run`, `step2_sustained_scheduler_soak`, `locustfile_step2_concurrent_users`) were deleted. Not committed yet. |
| 8 | `approval_link_secret` set on prod | ✅ Done (checked: 64 characters) |
| 9 | After deploying the 29 Oct commits: (a) open an old approval link: "Already Approved"; (b) send a test quotation email and click Approve; (c) P&L page roles are System Manager and Super Admin; (d) Error Log has no "Signing key missing"; (e) as a Customer Service user, move a Factory-warehouse order to Ready for Dispatch: a Gate Pass is created (no "Auto gate pass creation failed" in the Error Log); (f) open a Lyfe Order as an Engineer: no "No permission for Oneclick Settings" popup | ⏳ Pending |
| 10 | `mcp@lyfehardware.com` disabled on prod; its API key no longer works | ✅ Done. Still check: OAuth Bearer Token list has no Active token for that user |
| 11 | ShipStation sync after the hotfix `a553f6ef`: no new "unexpected keyword argument 'ignore_permission'" in the Error Log; orders synced before the deploy are rerouted by the next sync (check any still stuck) | ⏳ Pending |
| 12 | Outside tools with an API key (`automations@`, `kushal@`, `viralk@` hold none of the four gate roles): confirm none of them call the gated order actions (`create_gate_pass`, `bulk_assign_to_factory`, `create_load_for_order`, ...); `automations@` has made no document changes | ⏳ Pending |
| 13 | After deploying the 30 Oct commits (run `bench migrate` first: H10 and the Order Leg patch): (a) Customer Service, Factory and System Manager can still open the Status Overview, Order Lifecycle and Shipment Analysis pages and the Excel / PDF export buttons work; a user with no Lyfe Order role is refused; (b) Super Admin can open Order Leg; (c) Customer Service: Send Quotation and Generate Payment Link work; (d) Factory: Create Sales Invoice, Create Packing List, Bulk Transfer Batch pulls, Material Issue receipt and finish / tube / acrylic item creation work; Customer Service: Mark Delivered on a Transfer Order and the Replenishment steps work; (e) Error Log has no new `PermissionError` from a user who should have access | ⏳ Pending |

## Added 29 Oct

| Item | Status | Notes |
|---|---|---|
| New public endpoint `item_image` (`3d107ec1`, no login) | ⚠️ Reviewed, low risk, 2 small hardening points | Rate-limited (300/min), takes only an item code and redirects to `Item.image` (S3 files get a fresh signed URL), so a caller can't choose the file. Open points: (1) it redirects to whatever is stored in `Item.image`, so an Item editor could store any URL and make it an open redirect on your domain; allow only your own paths, `cdn.shopify.com` and the S3 host. (2) An Item editor could point `Item.image` at a private S3 file and this would hand out a signed link; limit to image files or the Item image bucket path. Neither is reachable by an outsider. |
| Engineer: "No permission for Oneclick Settings" on opening a Lyfe Order | ✅ Code done, pushed | `f970a196`: the flag is sent with the page load instead of read from the browser. Needs migrate and a browser refresh (testing item 9f). |
| ShipStation reroute hotfix | ✅ Code done, pushed | `a553f6ef` fixed two typos (`ignore_permission`) that broke `maybe_reroute_after_shipstation_sync` on every order sync; it came in with `df92578f`. Check the Error Log after the deploy (testing item 11). |
| Rule: System Manager and Super Admin are never blocked by an API check | ⚠️ Partly | Followed in everything added since 28 Oct. About 15 older checks still block one of them: the Super-Admin-only P&L API (`api.py`), `founder_dashboard.py`, `mcp_audit.py`; the CS/System Manager-only checks in `lyfe_order.py` and `merge_order.py`; 7 HR Manager/System Manager checks in `daily_work_log.py`; `shopify_token_rotation.py`; `backtest.py`; and the shared `require_roles()` helper. Waiting for a go-ahead to change them in one commit. |
| Whitelisted APIs: where to act | ⏳ Pending | Order of work: P&L (done), COGS exports (done), `revert_workflow_state` (done), then `update_all_real_time_tracking` (H3), `create_load_for_order`, `recalculate_weighted_score` (H9), the 13 order actions, outside-service calls (`retry_create_order_from_log`, `create_draft_order`, `send_quotation_email` done 30 Oct; `mark_order_as_shipped` left as is by decision), documents and stock actions, Bulk Transfer Batch, Transfer Order, Replenishment, Reshipment and Material Issue for Order methods (done 30 Oct). Still open: project board actions, timers, ShipStation sync, shipping helpers, validation and tracking methods |

## Records

| Record | Status |
|---|---|
| Tracker table in `SCHEDULE.md` | ⏳ Blank |
| Asana subtasks | ⏳ All 6 open; Mon 26 Oct can close once 0.5 and the C4/C6 deploy are done |
| Asana "PR / Commit link" field | ⏳ Empty; fill in after push |
