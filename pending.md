# Security audit fixes: lh app — status

**Asana:** [Security audit fixes: lh app (26–31 Oct)](https://app.asana.com/1/1204809689822175/project/1211921346208625/task/1219812548257717)
**Plan:** `LH Documentation/lh/SCHEDULE.md` · **Findings:** `LH Documentation/lh/REPORT.md`
**Last updated:** Wed 28 Oct 2026

✅ Completed · ⏳ Pending · ⚠️ Done, but needs a follow-up

## Summary

| | Completed | Pending |
|---|---|---|
| Criticals (C1–C6) | C2, C3, C4, C5, C6 (code); C1 Shopify token | C1: 1Click keys still need revoking |
| Highs (H1–H13) | H1 | H2–H13 |
| Mediums / Lows | 0.6 code; 5.1 partly | All others |

**Not pushed yet:** `753af930` (C4), `b63895e8` (C6), `026bb5c5` (0.6). The 1.4 prod deploy did not include them.

---

## Mon 26 Oct: secrets, config, Criticals

### Non-code steps

| # | Task | Status | Notes |
|---|---|---|---|
| 0.1 | Ask 1Click to revoke the 4 leaked keys and issue a new one | ⏳ Pending | C1 stays open until 1Click revokes them |
| 0.2 | Find where the `shpat_0217…` token came from | ✅ Completed | It's an old value of `Shopify Setting.password`. The token is rotated every 45 minutes (`shopify_token_rotation.rotate_shopify_token`) and expires after 1 hour, so the old one no longer works. |
| 0.3 | Role matrix | ✅ Completed | |
| 0.4 | Check prod webhook secrets; is the ShipStation webhook used? | ✅ Completed | 17Track key and Shopify secrets are set. ShipStation webhook isn't used, so it was deleted (C2). |
| 0.5 | Rotate the AWS access key | ⏳ Pending | `0e003915` and `f9f83402` are already deployed. The nginx deny isn't possible on Frappe Cloud and isn't needed anymore. |
| 0.6 | Rotate the Slack webhook and remove it from the patch | ✅ Completed | URL removed from the patch in `026bb5c5`. The old #order-via-us-warehouse webhook (`B0C2XQ3QKNV`) was revoked on 28 Oct; Slack now returns `no_service` (404) for it. Nothing in the ERP used it, so no settings needed updating. The Oneclick Settings field is kept for future use. |

### Code

| # | Finding | Status | Commit | What changed |
|---|---|---|---|---|
| C1 | Leaked 1Click keys in the repo | ✅ Code side | `6f8c2ff6`, `f9f83402`, `8e7f2410` | No key prefixes left in the current code. Revoking the keys is 0.1. |
| — | S3 secret stored encrypted | ✅ Completed | `0e003915` | |
| C2 | ShipStation webhook fail-open | ✅ Completed | `1e1be4a7` | Endpoint deleted; it was never used |
| H1 | Shopify / 17Track webhooks fail-open | ✅ Completed | `1e1be4a7` | Both reject requests when the secret is blank. The expected HMAC is no longer logged. |
| C5 | Forged ShipStation import | ✅ Completed | `6a977c35` | `process_shipstation_response` is no longer whitelisted |
| C3 | Fake payments | ✅ Completed | `9368226f` | `add_payment_entry` is no longer whitelisted. `complete_draft_order` allows only CS and System Manager, from Bank Transfer Selected or Partially Paid. Tested in the browser. |
| C4 | Shopify webhook hijack | ✅ Completed (not pushed) | `753af930` | `register_` / `deregister_shopify_webhooks`: System Manager only |
| C6 | Billable CJ labels | ✅ Completed (not pushed) | `b63895e8` | `generate_cj_label`, `generate_cj_label_for_reshipment`, `retry_from_integration_log`: Factory, Founder, Super Admin, System Manager |
| 1.1 | Shared `verify_webhook` helper | ✅ Completed | `1e1be4a7` | Done inline in each receiver; a shared helper isn't needed. The ShipStation webhook was deleted, and Shopify and 17Track each reject a blank secret and compare signatures in constant time. Each provider signs differently, so there's no logic to share. New webhooks are covered by the 2.1 / 5.9 CI check. |
| — | 17Track Integration Request logging | ✅ Completed | `ed0404af` | `17Track-Webhook` / `17Track-Scheduler` |
| 1.4 | Prod deploy and smoke test | ⚠️ Partly | | Deployed C2, C3, C5 and H1. C4, C6 and 0.6 need a push and another deploy. |

---

## Tue 27 Oct: CI guardrail + order/accounting actions — ⏳ not started

| # | Task | Status | Current state in code |
|---|---|---|---|
| 2.1 | AST CI check for whitelisted methods with no permission check | ⏳ Pending | No script in the repo |
| 2.2 | ~40 Lyfe Order and accounting actions (H4) | ⏳ Pending | No new checks |
| 2.3 | `create_gate_pass_for_order`, `mark_orders_ready_for_dispatch` (H4) | ⏳ Pending | No check; not POST-only |
| 2.4 | `backfill_erp_items_from_item_name`, `fetch_shopify_details` (H5) | ⏳ Pending | No check; use `ignore_permissions` |
| 2.5 | `submit_senior_review` and drawing-upload cancel (H12) | ⏳ Pending | No check; uses `ignore_permissions` |
| 2.6 | Item/BOM create and overwrite (H13) | ⏳ Pending | Unchanged |
| 2.7 | `update_all_real_time_tracking` (H3) | ⏳ Pending | Still `allow_guest=True`; no rate limit |

## Wed 28 Oct: finance, customer data, files — ⏳ not started

| # | Task | Status | Current state in code |
|---|---|---|---|
| 3.1 | P&L dashboard, COGS export, `get_quotation_detail` (H6) | ⏳ Pending | P&L gated only by Lyfe Order read; `export_cogs_by_internal_id_range` has no check |
| 3.2 | Lifecycle heal, shipment analysis, status-overview legs, exports (H7) | ⏳ Pending | |
| 3.3 | `generate_pick_pack_pdf` (H7) | ⏳ Pending | No check; uses `ignore_permissions` |
| 3.4 | `CustomFile.get_content`, S3 `?key=` (H2) | ⏳ Pending | Still fetches remote URLs with the `sid` cookie; `get_s3_image_url` accepts `?key=` |
| 3.5 | Sales Invoice PDFs private | ⏳ Pending | `sales_invoice.py:193` still sets `is_private=0` |
| 3.6 | Status overview / returns / aggregate dashboards gate | ⏳ Pending | |
| 3.7 | `backfill_item_group`: role gate + POST only | ⏳ Pending | Still accepts GET |
| 3.8 | Prod deploy and smoke test | ⏳ Pending | |

## Thu 29 Oct: projects, HR, Gorgias, MCP, Slack — ⏳ not started

| # | Task | Status | Current state in code |
|---|---|---|---|
| 4.1 | `has_permission` for Task, Project, Quotation, CRM Lead/Deal (H8) | ⏳ Pending | `hooks.py` covers only Lyfe Order and External Learning Record |
| 4.2 | Exact-match `_assign`; ELR list condition; visibility deny-by-default | ⏳ Pending | |
| 4.3 | `recalculate_weighted_score` (H9) | ⏳ Pending | No check |
| 4.4 | Gorgias reply/sync (H10) | ⏳ Pending | Existing checks predate the audit; finding still applies |
| 4.5 | `add_task_to_board` (H11) | ⏳ Pending | Existing check predates the audit; finding still applies |
| 4.6 | Task field allow-list, `set_column_terminal`, points, HR scope | ⏳ Pending | No checks |
| 4.7 | `safe_eval` without the `frappe` module | ⏳ Pending | `rule_engine.py:409` still passes `frappe` |
| 4.8 | MCP masked-field bypass; status-overview tool gates | ⏳ Pending | |
| 4.9 | Slack: enabled-user re-check, mapped users only | ⏳ Pending | |
| 4.10 | Validate workflow transitions | ⏳ Pending | |

## Fri 30 Oct: remaining Mediums, Lows, hardening — ⏳ not started (except 5.1, partly done)

| # | Task | Status | Current state in code |
|---|---|---|---|
| 5.1 | Quotation approval link | ⚠️ Partly | `e1cb6c1b` refuses closed quotations. Still to do: the HMAC key falls back to `db_password` and then to a hardcoded value (`quotation.py:2142`); no expiry; no POST confirm. |
| 5.2 | `methods=["POST"]` on state-changing methods | ⏳ Pending | |
| 5.3 | `ss_api_key` and Slack webhook fields → Password | ⏳ Pending | `ss_api_key` is still `Data` |
| 5.4 | Remove secrets from Error Log messages | ⏳ Pending | |
| 5.5 | Escaping: print formats, JS, email, OAuth | ⏳ Pending | |
| 5.6 | Link search ignores caller `doctype` | ⏳ Pending | |
| 5.7 | Small items | ⏳ Pending | `custom_file_upload_to_s3` is still whitelisted; `merge_orders_from_list`, `start_revision`, `create_and_start_timer` have no checks |
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
| 1 | Prod deploy of the Monday fixes | ✅ Done for C2, C3, C5, H1 · ⏳ C4, C6, 0.6 after push |
| 2 | 403 test with `sec-test-forwarder` / `sec-test-employee` on prod | ⏳ Pending. So far tested only on `lyfelocal.com`: C3 in the browser; C4 and C6 by script (Shipping Forwarder, CS and Employee blocked; Factory and System Manager allowed) |
| 3 | 17Track test push creates a `17Track-Webhook` Integration Request | ⏳ Pending |
| 4 | Error Log watch: "Secret Not Configured", "Key Not Configured", "failed to create Integration Request" | ⏳ Pending |
| 5 | Shopify webhooks added by hand (Settings → Notifications) | ⏳ Pending. Only the 3 API-registered webhooks were checked. |
| 6 | Smoke test: Shopify payment → Won, CJ label (as a Factory user), ShipStation sync, 1Click posting | ⏳ Pending |
| 7 | `execute_via_http` in `patches/generate_fake_orders_2000.py` broken by C5 | ⏳ Pending: delete it or leave it |

## Records

| Record | Status |
|---|---|
| Tracker table in `SCHEDULE.md` | ⏳ Blank |
| Asana subtasks | ⏳ All 6 open; Mon 26 Oct can close once 0.1, 0.5 and the C4/C6 deploy are done |
| Asana "PR / Commit link" field | ⏳ Empty; fill in after push |
