# Shopify Integration — Overview & Use Cases

## What Does It Do?

The Shopify integration handles the Quotation-to-payment workflow for custom orders:

1. **Draft Order creation** — when CS clicks "Generate Payment Link" on a Quotation, a Shopify Draft Order is created and a payment link is sent to the customer
2. **Payment polling** — every 5 minutes, pending quotation payment statuses are checked against Shopify
3. **Webhook handling** — when a customer pays by card, Shopify fires `orders/create` webhook → Lyfe Order is created
4. **Token rotation** — Shopify access token is rotated every 45 minutes (before the 1-hour expiry)

Module: `lyfe_hardware/integrations/shopify.py`  
Token rotation: `lyfe_hardware/integrations/shopify_token_rotation.py`

---

## Authentication

Credentials are read from the `Shopify Setting` doctype (from `ecommerce_integrations` app):
- `shopify_url` — store URL
- `password` (password field) — access token

Webhook HMAC secret is in `frappe.conf.shopify_webhook_secret` (site_config.json).

---

## Key Functions

| Function | Trigger | Purpose |
|---|---|---|
| `create_draft_order(quotation_name)` | CS clicks button | Creates Shopify Draft Order, sends payment link to customer |
| `update_draft_order(quotation_name)` | Quotation saved with items changed | Syncs item changes to existing Shopify Draft Order |
| `complete_draft_order(quotation_name)` | CS confirms bank transfer | Completes the Draft Order, marks quotation paid |
| `handle_orders_create()` | Shopify webhook `orders/create` | Creates Shopify Order → Lyfe Order for card payments |
| `poll_quotation_payment_statuses()` | Every 5 min (scheduler) | Checks if Draft Orders have been paid |
| `track_quotation_link(token, ref)` | Email CTA redirect | Tracks when customer clicks payment link |
| `rotate_shopify_token()` | Every 45 min (scheduler) | Rotates access token before expiry |

---

## Draft Order Lifecycle

```
Quotation (Draft)
    ↓ CS clicks "Generate Payment Link"
create_draft_order()
    ↓
Shopify Draft Order created
    ↓ payment link emailed to customer
Customer pays (card)
    ↓ Shopify fires orders/create webhook
handle_orders_create()
    ↓
Shopify Order doctype → Lyfe Order created

          -- OR --

Customer pays (bank transfer)
    ↓ CS confirms receipt
complete_draft_order()
    ↓
Draft Order completed → Lyfe Order created
```

---

## Webhook Verification

```python
def _verify_shopify_webhook(data, hmac_header):
    """Verifies HMAC-SHA256 signature on incoming webhooks."""
    secret = frappe.conf.shopify_webhook_secret.encode()
    computed = hmac.new(secret, data, hashlib.sha256).b64digest()
    return hmac.compare_digest(computed, hmac_header)
```

Called in `handle_orders_create()` before processing any payload.

---

## Use Cases

### Use Case 1: Create a Draft Order for a Quotation (Python)

```python
import frappe
from lh.lyfe_hardware.integrations.shopify import create_draft_order

result = create_draft_order("QTN-2026-00123")
print(result["draft_order_id"])   # Shopify Draft Order ID
print(result["payment_url"])      # URL sent to customer
```

### Use Case 2: Manually poll payment statuses

```bash
bench --site lyfe.local.local execute lh.lyfe_hardware.integrations.shopify.poll_quotation_payment_statuses
```

Or from Python:
```python
from lh.lyfe_hardware.integrations.shopify import poll_quotation_payment_statuses
poll_quotation_payment_statuses()
frappe.db.commit()
```

### Use Case 3: Find all quotations with pending Shopify payment

```python
import frappe

pending = frappe.get_all("Quotation",
    filters={
        "workflow_state": "Sent to Customer",
        "custom_shopify_draft_order_id": ["!=", ""],
        "custom_payment_status": ["in", ["Pending", ""]],
    },
    fields=["name", "customer_name", "custom_shopify_draft_order_id", "custom_email_sent_at"],
    order_by="custom_email_sent_at asc",
)
```

### Use Case 4: Client script — add "Generate Payment Link" button

```js
frappe.ui.form.on("Quotation", {
    refresh(frm) {
        if (frm.doc.workflow_state === "Approved" && !frm.doc.custom_shopify_draft_order_id) {
            frm.add_custom_button("Generate Payment Link", () => {
                frappe.confirm("Send payment link to customer?", () => {
                    frappe.call({
                        method: "lh.lyfe_hardware.integrations.shopify.create_draft_order",
                        args: { quotation_name: frm.doc.name },
                        callback: r => {
                            frappe.show_alert({ message: "Payment link sent!", indicator: "green" });
                            frm.reload_doc();
                        },
                    });
                });
            });
        }
    },
});
```

### Use Case 5: Manually rotate the Shopify access token

```bash
bench --site lyfe.local.local execute lh.lyfe_hardware.integrations.shopify_token_rotation.rotate_shopify_token
```

### Use Case 6: Check current Shopify connection

```python
import frappe
from lh.lyfe_hardware.integrations.shopify import _get_session

store_url, token, api_version = _get_session()
print(f"Store: {store_url}")
print(f"API version: {api_version}")
print(f"Token: {'set' if token else 'MISSING'}")
```

---

## Quotation Follow-up Emails

Separate from the Shopify Draft Order flow, the follow-up scheduler in `tasks/quotation_followup.py` sends follow-up emails to customers with unpaid quotations:

| Business Day | Action |
|---|---|
| Day 3 | Gentle check-in email + CS ToDo |
| Day 10 | Add-value email + CS ToDo |
| Day 25 | Urgency email + CS ToDo |
| Day 30 | Auto-move to Lost |

Runs Mon–Fri at 7 PM UTC via scheduler.

```bash
# Run manually
bench --site lyfe.local.local execute lh.lyfe_hardware.tasks.quotation_followup.run_followup_scheduler
```

---

## Configuration

| Where | What |
|---|---|
| `Shopify Setting` doctype | Store URL, access token |
| `frappe.conf.shopify_webhook_secret` | Webhook HMAC secret (site_config.json) |
| Scheduler (hooks.py) | `*/5` → poll payments, `*/45` → rotate token |
