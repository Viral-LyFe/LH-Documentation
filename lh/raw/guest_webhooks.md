# Guest endpoints and webhooks (allow_guest=True)

All 16 `allow_guest=True` methods in lh were read in full. There are no WhatsApp or 1Click inbound webhooks in this codebase; 1Click is outbound only.

| Endpoint | Auth | Verdict |
|---|---|---|
| `slack.api.events` / `interactions` | Slack v0 HMAC + 5-min replay window, fail-closed | OK |
| `lh_project.gorgias.webhook.receive` | shared header secret, constant-time, fail-closed | OK |
| `integrations.shipstation_webhook.receive` | HMAC, **fail-open** when secret empty | **G-1 Critical** |
| `integrations.shopify.handle_orders_create` / `handle_orders_paid` / `handle_refunds_create` | HMAC, **fail-open** when secret empty | **G-2 High** |
| `order_tracking_service.track_17track_webhook` (+ `tracking.track_17track_webhook` alias) | sha256 sign, **fail-open** when key empty | **G-3 High** |
| `report.shipment_analysis_dashboard.update_all_real_time_tracking` | **none** | **G-4 High** |
| `doc_events.quotation.approve_quotation` | HMAC token | **G-5 Medium** |
| `integrations.shopify.track_quotation_link` / `doc_events.quotation.track_email_open` | HMAC token | **G-6 Low** |
| `slack.api.oauth_redirect` | none | **G-7 Low** |
| `clarity_settings.get_clarity_config` | returns empty for Guest | OK |

---

## G-1 Critical: ShipStation webhook lets anyone create or overwrite Lyfe Orders when the secret is unset

`lh/lyfe_hardware/integrations/shipstation_webhook.py:19-29`

```python
secret = frappe.get_single("Oneclick Settings").get_password("webhook_secret") or ""
...
if secret:          # no secret -> no verification
    ...
```

`_upsert_lyfe_order` then looks up `Lyfe Order` by the attacker-supplied `orderNumber` and either **overwrites** the existing order or inserts a new one, both with `ignore_permissions=True`. It replaces customer, email, phone, the full ship-to address, totals and the whole `order_items` table, and forces `status = "Pending"`.

**Why this is probably exploitable in production:** ShipStation's v1 webhooks do **not** sign requests. They POST a `resource_url` and send no `X-Shipstation-Hmac-Sha256` header. If this endpoint is registered and working, the secret must be empty, which means there is no verification at all. If the secret is set, every real ShipStation call fails with 403, so the endpoint is dead code.

**Attack:** `POST /api/method/lh.lyfe_hardware.integrations.shipstation_webhook.receive` with `{"orderNumber":"<existing #>","shipTo":{...attacker address...},"items":[...]}`. This redirects a real customer's shipment, rewrites its items and pricing, and resets its workflow `status`. The update also breaks the CLAUDE.md rule that `status`, `workflow_status` and `workflow_state` change together.

**Fix:** Fail closed (`if not secret or not compare_digest(...): 403`). Better: delete this endpoint if the scheduled ShipStation sync (`shipstation_orders.py`) is the real ingestion path. If it is kept, treat the body only as a trigger and re-fetch the order from the ShipStation API using the stored API key. Never trust payload fields. Also, a ShipStation secret should not live in `Oneclick Settings`.

## G-2 High: Shopify webhooks skip HMAC when no secret is configured

`lh/lyfe_hardware/integrations/shopify.py:2394-2421`

```python
if not secret:
    return  # secret not configured — skip verification
```

`handle_orders_paid` / `handle_orders_create` mark quotations paid (Won), add payment entries and clear payment blocks. `handle_refunds_create` records refunds. If `Shopify Setting.custom_client_secret`, `shared_secret` and `site_config.shopify_webhook_secret` are all blank (for example after a settings reset or on a restored or staging copy), anyone can forge a "paid" event for any draft order and get goods released without paying.

Secondary issue: on a mismatch, line 2429 writes the **expected** HMAC into Error Log. That gives a valid signature for the attacker's own body to anyone who can read Error Log. The exposure is limited, but there is no reason to log it.

**Fix:** Raise `frappe.PermissionError` when no secret is configured. Remove `Expected:` from the log message.

## G-3 High: 17Track webhook is unauthenticated when `webhook_security_key` is empty

`lh/lyfe_hardware/doctype/lyfe_order/order_tracking_service.py:325-337`. The comment says "Only skip verification when no key has been configured yet". A forged `TRACKING_UPDATED` payload can set orders to Delivered/Shipped. That drives status transitions, payment-stage notifications to customers, and the Shipped/Completed emails.

**Fix:** Fail closed, as `gorgias/webhook.py:_verify_secret` already does.

## G-4 High: Unauthenticated trigger for mass 17Track API calls

`lh/lyfe_hardware/report/shipment_analysis_dashboard/shipment_analysis_dashboard.py:200`

`@frappe.whitelist(allow_guest=True) def update_all_real_time_tracking()` runs `track_order()` **synchronously** on up to 200 orders per request. Each order triggers outbound 17Track register/gettrackinfo calls and DB writes. Any anonymous user can loop this. The result is web-worker exhaustion (each request ties up a gunicorn worker for minutes), a burned 17Track quota, and broadcast `publish_realtime` events that include order names.

**Fix:** Drop `allow_guest`, add `frappe.only_for([...])`, and run it with `frappe.enqueue`. The enqueued sibling above line 200 already does this.

## G-5 Medium: `approve_quotation` can approve any quotation from any state, and the key is weak

`lh/lyfe_hardware/doc_events/quotation.py:2109-2165`, `lh/api.py:18`

1. **HMAC key:** `frappe.conf.get("secret_key") or frappe.conf.get("db_password") or "lh_approval_secret"`. Frappe does not define `secret_key`, so in practice the key is the **DB password**. That ties two secrets together, and rotating the DB password silently invalidates every link that has been sent. If both are absent (some containerised or managed setups), the key is the public constant `lh_approval_secret`, and anyone can forge approval tokens for every quotation name. Names are sequential and guessable.
2. **No state check:** any quotation in any `workflow_state` (Draft, Pending Review, Lost, Cancelled…) is set to `Won` with `frappe.db.set_value`. That bypasses workflow, validation and `on_update` guards such as the payment-stage enforcement.
3. **No expiry:** the token is deterministic, so a link from months ago still works after a requote.
4. It is a GET request with side effects, so mail-scanner link prefetching can "approve" quotations.

**Fix:** Use a dedicated random secret (`frappe.conf.lh_link_secret`, generated once), and **refuse when it is missing** with no hardcoded fallback. Include the quotation's `modified` timestamp or an expiry in the signed message. Only allow a transition from `Sent to Customer`. Show a confirm page and do the state change on POST.

The same key derivation appears in `shopify._make_tracking_token` (shopify.py:2519) and has the same issue.

## G-6 Low: Tracking endpoints: brute-force scan and email spam

- `track_email_open` (quotation.py:2067) and `_resolve_token` slow path (shopify.py:2545): every guest request computes up to 500 HMACs over a DB scan. This is a cheap amplification vector, not a bypass.
- `track_quotation_link?ref=bank_transfer` sends the bank-details email **on every click** (shopify.py:2492). Someone holding a link can mail-bomb the customer and support@.
- `ref` is stored unvalidated in `custom_cta_clicked_ref`. Allow-list it.

## G-7 Low: Reflected HTML in Slack OAuth redirect

`lh/slack/api.py:142-148` returns `f"<h3>Slack install failed: {error}</h3>"` with the guest-supplied `error` unescaped. Exploitability depends on how Frappe renders the return value for `response["type"]="page"` with `page_name=None`, which was not verified because no code could be run. Escape it anyway (`frappe.utils.escape_html(error)`), or delete the endpoint, since the code is unused.
