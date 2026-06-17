# Bank Transfer Payment Flow — Full Technical Documentation

> **Scope:** What happens from the moment a Quotation is sent to the customer through bank transfer selection, CS confirmation, and final ERP sync.  
> **Last updated:** 2026-06-17

---

## Table of Contents

1. [Overview — Two Payment Paths](#1-overview--two-payment-paths)
2. [Full Bank Transfer Flow — Step by Step](#2-full-bank-transfer-flow--step-by-step)
3. [What Gets Updated in ERP at Each Step](#3-what-gets-updated-in-erp-at-each-step)
4. [CS Confirmation — The `complete_draft_order` API](#4-cs-confirmation--the-complete_draft_order-api)
5. [Quotation Workflow States During Bank Transfer](#5-quotation-workflow-states-during-bank-transfer)
6. [Notifications Sent During the Flow](#6-notifications-sent-during-the-flow)
7. [Error Scenarios & What Happens](#7-error-scenarios--what-happens)
8. [Comparison: Bank Transfer vs Card Payment](#8-comparison-bank-transfer-vs-card-payment)

---

## 1. Overview — Two Payment Paths

When a Quotation is sent to the customer, they receive an email with two payment options:

```
┌────────────────────────────────────────────────────────┐
│  Customer email contains two CTAs:                     │
│                                                        │
│  [Pay by Card]         → Shopify checkout (instant)    │
│  [Pay by Bank Transfer] → Bank details page (manual)   │
└────────────────────────────────────────────────────────┘
```

Both CTAs route through a **tracking endpoint** before reaching their destination. This is how the ERP knows which method the customer chose.

**Card payment:** Fully automatic — Shopify webhook fires when payment clears, ERP updates without CS involvement.

**Bank transfer:** Semi-manual — customer transfers money independently, CS confirms receipt in ERP. This document covers bank transfer in full.

---

## 2. Full Bank Transfer Flow — Step by Step

```
STEP 1: CS generates payment link
    CS opens Quotation → clicks "Generate Payment Link"
    → ERP calls Shopify API (create_draft_order)
    → Shopify returns draft_order_id + invoice_url (payment page URL)
    → ERP stores both on the Quotation
    → Quotation state: "Approved – Pending Send"

STEP 2: Quotation sent to customer
    CS clicks "Send Quotation to Client" workflow action
    → ERP sends branded email to customer
    → Email contains two CTA links, both routed through tracking endpoint:
        [Pay by Card]          → /track?token=XXX&ref=pay_card
        [Pay by Bank Transfer] → /track?token=XXX&ref=bank_transfer
    → Quotation state: "Sent to Customer"

STEP 3: Customer clicks "Pay by Bank Transfer"
    Customer's browser hits the tracking endpoint:
        GET /api/method/lh.lyfe_hardware.integrations.shopify.track_quotation_link
            ?token=<hmac_token>&ref=bank_transfer

    ERP (track_quotation_link) does:
    a) Verifies the HMAC token → resolves to a Quotation name
    b) Sets on Quotation:
         custom_link_clicked_at    = now
         custom_cta_clicked_ref    = "bank_transfer"
         workflow_state            = "Bank Transfer Selected"   ← state changes here
         custom_payment_method     = "Bank Transfer"
    c) Sends notification email to support@lyfehardware.com    ← CS is notified
    d) Sends bank details follow-up email to customer           ← customer gets bank details
    e) Redirects customer browser to /bank-transfer-confirm    ← thank-you / instructions page

STEP 4: Customer makes the bank transfer
    Customer transfers money via their bank.
    ERP is NOT notified — nothing happens automatically at this point.
    CS must wait for the transfer to arrive in the company account.

STEP 5: CS confirms the payment received
    CS opens the Quotation in ERP → clicks "Confirm Payment" button
    → CS enters the received amount and reference number (e.g. UTR / transaction ID)
    → ERP calls: complete_draft_order(quotation_name, received_amount, reference)

    complete_draft_order does:
    a) Validates: quotation not already Won, amount > 0
    b) Calls Shopify API to complete the draft order:
         POST /admin/api/{version}/draft_orders/{draft_id}/complete.json
              ?payment_pending=False
         → Shopify converts draft order → real order
         → Returns the new Shopify order_id
    c) Calls add_payment_entry() to record the payment in ERP:
         - Appends row to custom_payment_entries child table
         - Recalculates custom_total_received_amount
         - Saves the Quotation
    d) Determines new workflow state:
         total_received >= grand_total  →  "Won"
         total_received < grand_total   →  "Partially Paid"
    e) Sets on Quotation:
         workflow_state         = "Won" or "Partially Paid"
         custom_payment_method  = "Bank Transfer"
    f) Commits to database
    g) Clears payment block on linked Lyfe Orders (if any were blocked)
    h) Updates per-milestone payment status on Payment Schedule rows

STEP 6: ERP is fully synced
    Quotation is now "Won" (or "Partially Paid" if partial).
    Payment history row is recorded on the Quotation.
    Linked Lyfe Order (if blocked) is unblocked.
    Milestone statuses updated (→ Paid for covered stages).
```

---

## 3. What Gets Updated in ERP at Each Step

### Step 1 — Generate Payment Link

| Field | DocType | Value Set |
|---|---|---|
| `custom_shopify_draft_order_id` | Quotation | Shopify draft order ID (e.g. `7291038294`) |
| `custom_shopify_payment_url` | Quotation | Shopify invoice URL (the payment page link) |

### Step 3 — Customer Clicks Bank Transfer

| Field | DocType | Value Set |
|---|---|---|
| `custom_link_clicked_at` | Quotation | Datetime of click |
| `custom_cta_clicked_ref` | Quotation | `"bank_transfer"` |
| `workflow_state` | Quotation | `"Bank Transfer Selected"` |
| `custom_payment_method` | Quotation | `"Bank Transfer"` |

### Step 5 — CS Confirms Payment (`complete_draft_order`)

| Field | DocType | Value Set |
|---|---|---|
| `custom_payment_entries` (child table) | Quotation | New row appended (see below) |
| `custom_total_received_amount` | Quotation | Updated sum of all payment rows |
| `workflow_state` | Quotation | `"Won"` or `"Partially Paid"` |
| `custom_payment_method` | Quotation | `"Bank Transfer"` |
| `custom_payment_block_stage` | Lyfe Order | Cleared (if was blocked) |
| `custom_payment_block_date` | Lyfe Order | Cleared (if was blocked) |
| `custom_payment_reminder_count` | Lyfe Order | Reset to 0 (if was blocked) |
| `custom_payment_status` | Payment Schedule rows | Stages now covered → `"Paid"` |

### Payment Entry Row (appended to `custom_payment_entries`)

| Column | Value |
|---|---|
| `payment_date` | Today's date |
| `payment_type` | `"Full Payment"` or `"Partial Payment"` |
| `payment_mode` | `"Manual Payment"` (bank transfer) |
| `payment_amount` | Amount entered by CS |
| `reference` | CS-entered UTR / transaction ID, or `"Shopify Order {id}"` if Shopify returned an order |

---

## 4. CS Confirmation — The `complete_draft_order` API

This is the function CS triggers when clicking **"Confirm Payment"** on the Quotation form.

### Function signature

```
POST /api/method/lh.lyfe_hardware.integrations.shopify.complete_draft_order

Parameters:
  quotation_name   (string, required)  — e.g. "SAL-QTN-2024-00123"
  received_amount  (float, required)   — amount received in INR/USD
  payment_type     (string, optional)  — defaults to "Full Payment"
  payment_mode     (string, optional)  — defaults to "Manual Payment"
  reference        (string, optional)  — UTR / transaction ID / cheque number
```

### What it does internally

```
1. Read current workflow_state — throw if already "Won"
2. Validate received_amount > 0 — throw if zero
3. Read custom_shopify_draft_order_id from Quotation
4. If draft_id exists:
     POST Shopify API: draft_orders/{id}/complete.json?payment_pending=False
     → If Shopify returns error: throw Shopify API error to CS
     → If OK: read the returned order_id
5. Call add_payment_entry() — appends history row, recalculates total
6. Compute new_state:
     total_received >= grand_total → "Won"
     total_received < grand_total  → "Partially Paid"
7. SET on Quotation: workflow_state, custom_payment_method = "Bank Transfer"
8. frappe.db.commit()
9. (Post-payment side effects via add_payment_entry):
     - clear_payment_blocks_if_resolved()
     - update_payment_milestone_statuses()
10. Return: { shopify_order_id, total_received, new_state, status }
```

### Validation errors CS might see

**Quotation already Won:**
```
┌──────────────────────────────────────────────────┐
│ (frappe.throw)                                   │
├──────────────────────────────────────────────────┤
│ This quotation is already Won.                   │
└──────────────────────────────────────────────────┘
```

**Zero amount entered:**
```
┌──────────────────────────────────────────────────┐
│ (frappe.throw)                                   │
├──────────────────────────────────────────────────┤
│ Received Amount must be greater than zero.       │
└──────────────────────────────────────────────────┘
```

**Shopify API fails during completion:**
```
┌──────────────────────────────────────────────────────────────┐
│ (frappe.throw)                                               │
├──────────────────────────────────────────────────────────────┤
│ Shopify API error (422): {"errors": {"base": ["..."]}}       │
└──────────────────────────────────────────────────────────────┘
```
The payment is **NOT recorded in ERP** if Shopify returns an error. CS must investigate and retry.

### Partial payment scenario

If a customer agreed to pay 50% upfront and 50% later:

**First confirmation (₹5,000 received, grand total ₹10,000):**
- CS enters `received_amount = 5000`
- `total_received = 5000`, `grand_total = 10000` → `5000 < 10000`
- `workflow_state` → `"Partially Paid"`
- Quotation stays active; payment block on Lyfe Order is checked (if advance threshold was ₹5,000, it clears)

**Second confirmation (remaining ₹5,000 received):**
- CS enters `received_amount = 5000`
- `total_received = 10000`, `grand_total = 10000` → `10000 >= 10000`
- `workflow_state` → `"Won"`
- All payment blocks cleared, all milestone statuses set to `"Paid"`

---

## 5. Quotation Workflow States During Bank Transfer

```
Draft
  │
  ▼
Sent to Customer          ← CS sends the quotation email
  │
  │  [Customer clicks Bank Transfer link in email]
  ▼
Bank Transfer Selected    ← ERP state set by tracking endpoint (automatic)
  │
  │  [CS waits for bank transfer to arrive]
  │  [CS clicks "Confirm Payment" on the Quotation form]
  ▼
Partially Paid            ← if total received < grand total
  │   or
  ▼
Won                       ← if total received >= grand total
```

### State: `"Bank Transfer Selected"`

Set automatically the instant the customer clicks the bank transfer CTA link in the email.

**What CS sees on the Quotation form at this state:**
- `custom_payment_method` = `"Bank Transfer"` (visible in Payment tab)
- `custom_link_clicked_at` = the datetime the customer clicked
- A **"Confirm Payment"** button becomes visible on the form
- A **"Confirm Remaining Payment"** button also visible (for partial payments)

**No Shopify webhook fires at this state** — the draft order is still open. Shopify only fires `orders/create` and `orders/paid` for card payments (when Shopify processes the money). Bank transfers are outside Shopify's payment processing.

---

## 6. Notifications Sent During the Flow

### When customer clicks "Bank Transfer" (Step 3)

**Email 1 — CS notification** (to `support@lyfehardware.com`):

```
Subject: Bank Transfer Selected — SAL-QTN-2024-00123

{Customer Name} has selected bank transfer for quotation
SAL-QTN-2024-00123.

Grand Total: INR 10,000.00

Please confirm receipt once the transfer arrives and update the
quotation status to Payment Received.

[Link to Quotation in ERP]
```

**Email 2 — Customer follow-up** (to customer's email on Quotation):

```
Subject: (from bank_transfer_followup email template)
```

Sent using the `bank_transfer_followup` Frappe email template. This email contains the company's bank account details, IFSC code, reference instructions, and confirmation instructions.

> **Configuration note:** The `bank_transfer_followup` email template must exist in `Email Templates` on the live site. If missing, this email silently fails (logged as an error).

### When payment block is active (payment not received, order trying to advance)

See [Payment Terms System documentation](Payment_Terms_System.md) — Section 6 (Reminder & Escalation System) for the full reminder schedule.

---

## 7. Error Scenarios & What Happens

### Customer clicks bank transfer but ERP tracking fails

**Cause:** Token in the CTA URL is invalid or expired.

**What customer sees:** `"Invalid or expired link."` error page.

**ERP state:** Quotation stays in `"Sent to Customer"`. Nothing is updated.

**Fix:** CS must manually set `workflow_state = "Bank Transfer Selected"` and `custom_payment_method = "Bank Transfer"` on the Quotation, then click "Confirm Payment" when money arrives.

---

### CS confirms payment but Shopify draft order no longer exists

**Cause:** The draft order was deleted on Shopify manually, or it expired.

**What happens:** `complete_draft_order` checks `custom_shopify_draft_order_id`. If blank, it **skips the Shopify API call** entirely and proceeds to record the payment in ERP directly.

**Effect:** Payment is recorded in ERP (`custom_payment_entries` row added, state → Won/Partially Paid). No Shopify order is created.

**When this is normal:** If the quotation was created before the Shopify integration was set up, `custom_shopify_draft_order_id` will be blank. `complete_draft_order` still works — it just skips the Shopify step.

---

### CS confirms payment but Shopify API returns an error (e.g. 422)

**Cause:** The draft order may already be completed, the Shopify store is in a bad state, or the access token expired.

**What CS sees:**
```
Shopify API error (422): {"errors": {"base": ["Draft Order has already been completed"]}}
```

**ERP state:** Nothing is written — the throw happens before `add_payment_entry` is called.

**Fix options:**
1. If the Shopify order was actually created (check Shopify admin), record the payment manually in ERP:
   - Use `add_payment_entry` directly via the console or the "Add Payment" button
   - Then set `workflow_state = "Won"` manually
2. If the draft order was already completed earlier (e.g. via webhook), check `custom_payment_entries` — the payment may already be recorded. If so, no action needed.

---

### Partial payment and CS clicks "Confirm Payment" a second time with same reference

**Cause:** CS accidentally submits the same confirmation twice.

**What happens:** `add_payment_entry` checks the `reference` field. If the same reference already exists in `custom_payment_entries`, it **skips and returns** `{"status": "duplicate_skipped"}`. The total is not double-counted.

**Effect on `complete_draft_order`:** Since `add_payment_entry` returns without appending, `total_received` stays the same, so `new_state` is computed correctly (no double-counting).

---

### Payment arrives but Quotation is already Won

**Cause:** A second bank transfer arrived (overpayment or customer error).

**What happens:** `add_payment_entry` throws:
```
This quotation is already Won. No further payments can be recorded.
```

**Fix:** Record the overpayment as a separate credit note or refund via standard ERP accounting — not through the Quotation payment system.

---

## 8. Comparison: Bank Transfer vs Card Payment

| | Bank Transfer | Card (Shopify Checkout) |
|---|---|---|
| **Customer action** | Clicks CTA → sent bank details → transfers money manually | Clicks CTA → Shopify checkout → pays online |
| **ERP state change to "Bank Transfer Selected"** | Automatic (tracking endpoint) | N/A |
| **ERP state change to "Won"** | Manual — CS clicks "Confirm Payment" | Automatic — Shopify webhook or poller |
| **Payment recorded by** | CS (manually, via `complete_draft_order`) | Webhook (`handle_orders_create` / `handle_orders_paid`) or poller |
| **`payment_mode` in history** | `"Manual Payment"` | `"Payment Link"` |
| **`custom_payment_method` on Quotation** | `"Bank Transfer"` | `"Card"` |
| **Shopify order created** | Yes — when CS calls `complete_draft_order` (or skipped if no draft ID) | Yes — automatically when customer pays |
| **CS notification email** | Yes — to `support@lyfehardware.com` immediately | No |
| **Customer bank details email** | Yes — sent automatically on click | No |
| **Risk of double-recording** | Low — reference dedup prevents it | Low — webhook dedup by Shopify order name |
| **Supports partial payment** | Yes — CS enters partial amount, state → Partially Paid | Yes — Shopify `financial_status = partially_paid` |
| **Latency (ERP knows about payment)** | Minutes to days (depends on bank + CS) | Seconds (webhook) to 5 minutes (poller fallback) |

---

## Quick Reference — CS Actions in ERP

| Situation | Action in ERP |
|---|---|
| Customer has not paid yet, Quotation in `"Sent to Customer"` | Wait. Check `custom_link_clicked_at` to see if customer opened the email. |
| Quotation shows `"Bank Transfer Selected"` | Customer clicked the link. Wait for bank transfer to arrive. |
| Bank transfer received (confirmed in company account) | Open Quotation → click **"Confirm Payment"** → enter amount + reference → Save |
| Bank transfer received partially | Open Quotation → click **"Confirm Payment"** → enter partial amount → state becomes `"Partially Paid"` |
| Remaining balance received later | Open Quotation → click **"Confirm Remaining Payment"** → enter remaining amount → state becomes `"Won"` |
| Order still blocked after confirming payment | Check `custom_payment_block_stage` on the Lyfe Order — if not cleared, check that `custom_total_received_amount` on the Quotation was recalculated correctly |
| Shopify API error during confirmation | See [Error Scenarios](#7-error-scenarios--what-happens) |

---

*End of document — Bank Transfer Payment Flow*
