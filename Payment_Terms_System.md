# LH Payment Terms & Milestone System — Full Documentation

> **BRD Reference:** LH-BRD-DEV-004 Payment Terms and Urgent SLA v1.1  
> **Last updated:** 2026-06-17  
> **Status:** Implemented (EC-3, EC-4, EC-6, EC-7, EC-9, EC-11)

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Payment Stage Model](#2-payment-stage-model)
3. [Milestone Status Field (EC-4)](#3-milestone-status-field-ec-4)
4. [Payment Gate Enforcement — Role-Based Messages](#4-payment-gate-enforcement--role-based-messages)
5. [Edge Cases — Full Detail](#5-edge-cases--full-detail)
   - [EC-3 Order Value Change Warning](#ec-3-order-value-change-warning)
   - [EC-4 Per-Milestone Payment Status](#ec-4-per-milestone-payment-status)
   - [EC-6 Auto-Release on Cancellation](#ec-6-auto-release-on-cancellation)
   - [EC-7 On Delivery Collection Risk Warning](#ec-7-on-delivery-collection-risk-warning)
   - [EC-9 Payment Schedule Sum Mismatch](#ec-9-payment-schedule-sum-mismatch)
   - [EC-11 Invalid Payment Schedule Row](#ec-11-invalid-payment-schedule-row)
6. [Payment Block — Reminder & Escalation System](#6-payment-block--reminder--escalation-system)
7. [Notification Reference](#7-notification-reference)
8. [Payment Recording](#8-payment-recording)
9. [Workflow State Map](#9-workflow-state-map)
10. [Role Reference](#10-role-reference)

---

## 1. System Overview

The payment terms system enforces milestone-based payment collection on **Quotations** before allowing a linked **Lyfe Order** to advance through the production workflow.

### How It Works — End to End

```
Quotation created
    └─ CS adds Payment Schedule rows (milestones)
           │
           ├─ Each row tied to one stage checkbox
           ├─ Amounts must sum to grand total (EC-9, EC-11)
           └─ On Delivery rows get a collection-risk warning (EC-7)

Order placed → Lyfe Order created
    │
    ├─ [Advance] Warehouse set to Factory
    │      → Payment gate: is advance received? Block if not.
    │
    ├─ [Photo Approval] Lyfe Order → Approved
    │      → Payment gate: is photo approval payment received? Block if not.
    │
    ├─ [Before Dispatch] Lyfe Order → Ready for Dispatch
    │      → Payment gate enforced via lyfe_order.py
    │
    ├─ [Before Shipped] Lyfe Order → Shipped
    │      → No block (order already packed/dispatched)
    │      → CS receives one-time email + Slack notification if unpaid
    │
    └─ [On Delivery] Lyfe Order → Completed
           → No block (cannot stop delivery)
           → CS receives one-time email + Slack notification if unpaid

Payment recorded (via Shopify webhook / manual)
    └─ Clears any active payment block on linked Lyfe Orders
    └─ Updates per-milestone status (→ Paid)
    └─ Daily scheduler re-checks all still-blocked orders
```

---

## 2. Payment Stage Model

### Stage Priority Order

Stages are evaluated cumulatively in this exact order:

| Priority | Field Name | Label | Trigger Point |
|---|---|---|---|
| 1 | `custom_advance_amount` | Advance Amount | Warehouse set to Factory |
| 2 | `custom_stage_photo_approval` | Photo Approval | Lyfe Order → Approved |
| 3 | `custom_before_dispatch` | Before Dispatch | Lyfe Order → Ready for Dispatch |
| 4 | `custom_before_shipped` | Before Shipped | Lyfe Order → Shipped |
| 5 | `custom_on_delivery` | On Delivery | Lyfe Order → Completed |

### What "Cumulative Threshold" Means

The system does **not** check individual milestone amounts in isolation. It computes a running total.

**Example:** 3-milestone schedule on a ₹10,000 order:

| Row | Stage | Amount | Cumulative Threshold |
|---|---|---|---|
| 1 | Advance Amount | ₹3,000 | ₹3,000 |
| 2 | Photo Approval | ₹3,000 | ₹6,000 |
| 3 | Before Dispatch | ₹4,000 | ₹10,000 |

When the order reaches **Photo Approval**, the system checks: `received ≥ ₹6,000` (not just the ₹3,000 for that row alone). This means a customer who paid ₹5,000 upfront is still blocked at Photo Approval because ₹5,000 < ₹6,000.

---

## 3. Milestone Status Field (EC-4)

Every Payment Schedule row on a Quotation has a `custom_payment_status` field that is **automatically maintained** by the system.

### Status Values

| Status | When Set | Color |
|---|---|---|
| `Upcoming` | Row's trigger stage has not been reached yet | Grey |
| `Due` | Trigger stage reached, payment not yet received | Orange |
| `Overdue` | Due + 1 or more daily reminders sent (`reminder_count ≥ 1`) | Red |
| `Escalated` | Due + 5 or more daily reminders sent (`reminder_count ≥ 5`) | Dark Red |
| `Paid` | Cumulative received ≥ threshold for this stage | Green |

### When Status Is Recalculated

Status is recalculated automatically in three situations:

1. **Quotation workflow changes** (Won / Partially Paid) — marks Advance stage as Due
2. **Lyfe Order workflow changes** (Approved / Ready for Dispatch / Shipped / Completed) — marks corresponding stage as Due
3. **Payment is recorded** (`add_payment_entry`) — marks stages as Paid where threshold is now met

### Example — 50/50 Order (₹10,000)

**Setup:** Row 1 = Advance 50% (₹5,000), Row 2 = Before Dispatch 50% (₹5,000)

| Event | Row 1 Status | Row 2 Status |
|---|---|---|
| Quotation created, Draft state | Upcoming | Upcoming |
| Quotation → Won | **Due** | Upcoming |
| Customer pays ₹5,000 | **Paid** | Upcoming |
| Lyfe Order → Ready for Dispatch | Paid | **Due** |
| No payment for 1 day (reminder sent) | Paid | **Overdue** |
| No payment for 5 days | Paid | **Escalated** |
| Customer pays ₹5,000 | Paid | **Paid** |

### Example — 30/30/40 Order (₹10,000)

| Row | Stage | Amount | Cumulative |
|---|---|---|---|
| 1 | Advance | ₹3,000 | ₹3,000 |
| 2 | Photo Approval | ₹3,000 | ₹6,000 |
| 3 | Before Dispatch | ₹4,000 | ₹10,000 |

| Event | Row 1 | Row 2 | Row 3 |
|---|---|---|---|
| Quotation → Won | **Due** | Upcoming | Upcoming |
| Pay ₹3,000 | **Paid** | Upcoming | Upcoming |
| Lyfe Order → Approved | Paid | **Due** | Upcoming |
| Pay ₹3,000 (total ₹6,000) | Paid | **Paid** | Upcoming |
| Lyfe Order → Ready for Dispatch | Paid | Paid | **Due** |
| Pay ₹4,000 (total ₹10,000) | Paid | Paid | **Paid** |

---

## 4. Payment Gate Enforcement — Role-Based Messages

When a Lyfe Order tries to advance to a stage where payment is insufficient, **the save is blocked**. The error message shown depends on the user's role.

### Stages That Block the Workflow

| Transition | Stage Checked | Context Label |
|---|---|---|
| Warehouse set to `Factory - LH` | Advance Amount | "assigning order to Factory" |
| Lyfe Order → Approved | Photo Approval | "marking order as Approved" |
| Lyfe Order → Ready for Dispatch | Before Dispatch | "Ready for Dispatch" |

> **Shipped and Completed do NOT block the workflow.** The order has already left the factory. Instead, a one-time notification is sent to CS.

---

### 4.1 Message Seen by Customer Service / System Manager

**Roles:** `Customer Service`, `System Manager`

These users see the **full payment breakdown** so they can take action immediately.

```
┌─────────────────────────────────────────────────────────┐
│ Payment Incomplete                                      │
├─────────────────────────────────────────────────────────┤
│ Payment required before assigning order to Factory.     │
│                                                         │
│ Quotation:             SAL-QTN-2024-00123               │
│ Required (cumulative): ₹ 5,000.00                       │
│ Received so far:       ₹ 2,000.00                       │
└─────────────────────────────────────────────────────────┘
```

**Example scenario:** Factory user tries to assign order LH-ORD-2024-00456 linked to quotation SAL-QTN-2024-00123 (advance = ₹5,000, received = ₹2,000).

The CS user sees exactly what is missing: ₹3,000 more is needed before the order can proceed.

---

### 4.2 Message Seen by Factory / Engineer / All Other Roles

**Roles:** `Factory`, `Engineer`, `Shipping Forwarder`, and all others

These users see a **generic message** without financial details. They are not responsible for collecting payment.

```
┌─────────────────────────────────────────────────────────┐
│ Payment Incomplete                                      │
├─────────────────────────────────────────────────────────┤
│ This order cannot proceed — payment is pending.         │
│                                                         │
│ The CS team has been notified to follow up on the       │
│ payment status.                                         │
└─────────────────────────────────────────────────────────┘
```

---

### 4.3 No Payment Recorded At All (Won State)

If a Quotation is set to `Won` via an API call or bulk action but **no payment has been recorded at all**, the save is blocked with:

```
┌─────────────────────────────────────────────────────────┐
│ Payment Required                                        │
├─────────────────────────────────────────────────────────┤
│ Cannot mark as Won — no payment has been recorded for   │
│ this quotation. Please record a payment first.          │
└─────────────────────────────────────────────────────────┘
```

> **Note:** This guard runs server-side and covers API calls, bulk actions, and Administrator paths that bypass the JS dialog.

---

### 4.4 Won with Partial Payment → Auto-Corrected to Partially Paid

If a Quotation is set to `Won` but `received < grand_total`, the system **silently corrects** the state to `Partially Paid` (instead of blocking). An error is logged.

This covers integrations or admin saves that set `Won` without checking the payment amount. The order proceeds but remains in `Partially Paid` until fully paid.

---

## 5. Edge Cases — Full Detail

### EC-3 Order Value Change Warning

**Trigger:** Quotation is saved and `grand_total` has changed since the last save, AND a Payment Schedule exists.

**Type:** Non-blocking warning (orange `msgprint`)

**Who sees it:** All users who save the Quotation.

#### Scenario A — No payments yet received

The order total changed from ₹10,000 to ₹12,000 and no payment has been received:

```
┌──────────────────────────────────────────────────────────────┐
│ Payment Schedule Needs Review                    (orange)    │
├──────────────────────────────────────────────────────────────┤
│ Order total has changed from ₹ 10,000.00 to ₹ 12,000.00     │
│ but the Payment Schedule milestone amounts have not been     │
│ updated. Please review and adjust each milestone amount so   │
│ they sum to the new total.                                   │
└──────────────────────────────────────────────────────────────┘
```

**Action required:** CS must manually update the Payment Schedule rows so they sum to ₹12,000.

#### Scenario B — Payment already received

The order total changed from ₹10,000 to ₹12,000 and ₹3,000 has already been paid:

```
┌──────────────────────────────────────────────────────────────┐
│ Payment Schedule Needs Review                    (orange)    │
├──────────────────────────────────────────────────────────────┤
│ Order total has changed from ₹ 10,000.00 to ₹ 12,000.00     │
│ but the Payment Schedule milestone amounts have not been     │
│ updated. Please review and adjust each milestone amount so   │
│ they sum to the new total.                                   │
│                                                              │
│ Warning: ₹ 3,000.00 has already been received against this   │
│ quotation. The paid amount may no longer match the intended  │
│ milestone percentage. Please inform the finance team.        │
└──────────────────────────────────────────────────────────────┘
```

**This is a warning only — the save is NOT blocked.** CS must review and update the schedule.

---

### EC-4 Per-Milestone Payment Status

See [Section 3](#3-milestone-status-field-ec-4) for complete detail.

**Summary:** Every Payment Schedule row automatically tracks its collection status (`Upcoming` → `Due` → `Overdue` → `Escalated` → `Paid`). No manual intervention needed — the system updates this on every relevant workflow change and payment recording.

---

### EC-6 Auto-Release on Cancellation / Lost

**Trigger:** Quotation workflow state changes to `Lost` or `Cancelled` for the **first time** (not on re-saves in the same terminal state).

**Effect:** All linked Lyfe Orders that have an active payment block are **automatically released** — the three tracking fields (`custom_payment_block_stage`, `custom_payment_block_date`, `custom_payment_reminder_count`) are cleared.

**Why:** A cancelled order should never keep generating payment reminders or blocking production reassignment.

#### Example

Order LH-ORD-2024-00456 is blocked at Advance Amount. CS marks the Quotation as `Lost`.

**Before:**
- `custom_payment_block_stage` = `custom_advance_amount`
- `custom_payment_block_date` = `2026-06-10`
- `custom_payment_reminder_count` = 3

**After (automatic):**
- `custom_payment_block_stage` = *(cleared)*
- `custom_payment_block_date` = *(cleared)*
- `custom_payment_reminder_count` = 0

No further reminders are sent. No visible message to the user — this happens silently in the background.

---

### EC-7 On Delivery Collection Risk Warning

**Trigger:** User checks the `custom_on_delivery` checkbox on any Payment Schedule row in the browser.

**Type:** Non-blocking warning (orange `msgprint`), shown in the browser only.

**Who sees it:** Any user who ticks the On Delivery checkbox.

```
┌──────────────────────────────────────────────────────────────────┐
│ On Delivery — Collection Risk                        (orange)    │
├──────────────────────────────────────────────────────────────────┤
│ Note: Nothing comes after the Completed step, so the system      │
│ cannot block the workflow to enforce this payment. An email      │
│ and Slack notification will be sent to CS when the order is      │
│ marked Completed, but collection relies entirely on manual       │
│ follow-up. Ensure CS/Sales is aware before committing to the     │
│ customer.                                                        │
└──────────────────────────────────────────────────────────────────┘
```

**What happens at Completed:** CS receives an email and Slack message with the shortfall amount. There is no workflow block — the order is already with the customer.

---

### EC-9 Payment Schedule Sum Mismatch

**Trigger:** Quotation `validate` — runs every time the Quotation is saved.

**Type:** Hard block (`frappe.throw`) — the save is rejected.

**Who sees it:** All users who save the Quotation.

#### Example — Rounding error on 33/33/34 split

Order total: ₹9,999. CS enters three rows: ₹3,333 + ₹3,333 + ₹3,333 = ₹9,999. ✓ (this would pass)

But if CS enters: ₹3,333 + ₹3,333 + ₹3,334 = ₹10,000 on a ₹9,999 order:

```
┌──────────────────────────────────────────────────────────────────┐
│ Payment Schedule Mismatch                                        │
├──────────────────────────────────────────────────────────────────┤
│ Payment Schedule amounts sum to ₹ 10,000.00 but the order total  │
│ is ₹ 9,999.00. Please adjust the milestone amounts so they sum   │
│ exactly to the order total (check rounding on splits like        │
│ 33/33/34).                                                       │
└──────────────────────────────────────────────────────────────────┘
```

**Tolerance:** The system allows a ±₹0.01 difference to handle floating-point rounding. Any difference larger than ₹0.01 triggers this error.

---

### EC-11 Invalid Payment Schedule Row

**Trigger:** Quotation `validate` — runs every time the Quotation is saved.

**Type:** Hard block (`frappe.throw`) — the save is rejected.

**Who sees it:** All users who save the Quotation.

#### EC-11a — Row with Zero Percentage

A Payment Schedule row has `invoice_portion = 0%`:

```
┌──────────────────────────────────────────────────────────────────┐
│ Invalid Payment Schedule                                         │
├──────────────────────────────────────────────────────────────────┤
│ Payment Schedule row 2 has a zero percentage. Every milestone    │
│ must have a non-zero Invoice Portion.                            │
└──────────────────────────────────────────────────────────────────┘
```

#### EC-11b — Row with No Stage Checkbox

A Payment Schedule row exists but none of the five stage checkboxes (`Advance Amount`, `Photo Approval`, `Before Dispatch`, `Before Shipped`, `On Delivery`) is checked:

```
┌──────────────────────────────────────────────────────────────────┐
│ Invalid Payment Schedule                                         │
├──────────────────────────────────────────────────────────────────┤
│ Payment Schedule row 3 has no trigger event (no stage checkbox   │
│ is checked). Every milestone must be tied to a workflow stage.   │
└──────────────────────────────────────────────────────────────────┘
```

**Note:** The error message identifies the specific row number. If multiple rows are invalid, the first failing row is reported (the rest would be caught on subsequent saves).

---

## 6. Payment Block — Reminder & Escalation System

When a payment gate blocks a Lyfe Order, the system immediately records the block and begins a daily reminder cycle.

### Tracking Fields on Lyfe Order

| Field | Type | Purpose |
|---|---|---|
| `custom_payment_block_stage` | Data | Which stage blocked this order (e.g. `custom_advance_amount`) |
| `custom_payment_block_date` | Date | Date the block was first triggered |
| `custom_payment_reminder_count` | Int | Number of daily reminders sent so far |

### Reminder Schedule

| Day | Recipients | Email Subject Example |
|---|---|---|
| Day 1 | CS team only | `[Payment Reminder — Day 1] LH-ORD-2024-00456 — Advance Amount` |
| Day 2 | CS team only | `[Payment Reminder — Day 2] LH-ORD-2024-00456 — Advance Amount` |
| Day 3 | CS team only | `[Payment Reminder — Day 3] LH-ORD-2024-00456 — Advance Amount` |
| Day 4 | CS team only | `[Payment Reminder — Day 4] LH-ORD-2024-00456 — Advance Amount` |
| Day 5+ | CS team + **Founders** | `[Payment Reminder — Day 5] LH-ORD-2024-00456 — Advance Amount` |

- **CS team email:** `customer@lyfehardware.com`
- **Founder emails:** fetched from `Team Settings → founders` (Team Member child table)
- **Daily scheduler:** runs at 9:00 AM every day

### Email Body (Day 1–4, CS Only)

```
This is an automated payment reminder for Lyfe Order LH-ORD-2024-00456.

┌────────────────────────────┬──────────────────────────┐
│ Field                      │ Value                    │
├────────────────────────────┼──────────────────────────┤
│ Lyfe Order                 │ LH-ORD-2024-00456        │
│ Quotation                  │ SAL-QTN-2024-00123       │
│ Payment Stage              │ Advance Amount           │
│ Block Date                 │ 2026-06-12               │
│ Required (cumulative)      │ ₹ 5,000.00               │
│ Received so far            │ ₹ 2,000.00               │
│ Shortfall                  │ ₹ 3,000.00               │
│ Reminder Day               │ 1                        │
└────────────────────────────┴──────────────────────────┘

Please follow up with the customer and record the payment in
Frappe once received.

This reminder will repeat daily until the required payment is received.
```

### Email Body (Day 5+, Escalated to Founders)

Same table as above, plus:

```
Day 5 — Escalated to Founders: founder1@company.com, founder2@company.com
```

### Slack Notification (sent alongside each email)

```
*Payment Reminder — Day 1* | Order: `LH-ORD-2024-00456`
Stage: *Advance Amount* | Quotation: SAL-QTN-2024-00123
Required: ₹ 5,000.00 | Received: ₹ 2,000.00 | Shortfall: *₹ 3,000.00*
Please record payment in Frappe to unblock this order.
```

Day 5+ adds: `🚨 *Day 5 — Escalated to Founders.*`

**Slack webhook:** Configured in `Team Settings → custom_payment_reminder_slack_webhook`. If blank, Slack notifications are silently skipped. Email still sends.

### When Reminders Stop

Reminders stop automatically in three situations:

1. **Payment is recorded** — `add_payment_entry` calls `clear_payment_blocks_if_resolved`, which checks if `received ≥ threshold` and clears the three tracking fields.
2. **Quotation is Lost or Cancelled** — EC-6 releases all blocks immediately.
3. **Daily scheduler re-check** — if payment was recorded by another path, the scheduler detects `received ≥ threshold` and clears before sending the next reminder.

---

## 7. Notification Reference

### Before Shipped Notification (non-blocking)

**Trigger:** Lyfe Order transitions to `Shipped` or `Awaiting Tracking`, AND the `Before Shipped` stage payment has not been received.

**Recipients:** CS team only (`customer@lyfehardware.com`)

**Email subject:**
```
[Before Shipped Payment Pending] LH-ORD-2024-00456 — Order Shipped
```

**Email body excerpt:**
```
Order LH-ORD-2024-00456 has been marked Shipped, but the Before Shipped
payment has not yet been received.

┌───────────────────────┬─────────────────────┐
│ Lyfe Order            │ LH-ORD-2024-00456   │
│ Quotation             │ SAL-QTN-2024-00123  │
│ Payment Stage         │ Before Shipped      │
│ Required (cumulative) │ ₹ 8,000.00          │
│ Received so far       │ ₹ 6,000.00          │
│ Shortfall             │ ₹ 2,000.00          │
└───────────────────────┴─────────────────────┘

Please follow up with the customer and record the payment in
Frappe once received.
```

**This is a one-time notification only.** It does not repeat daily. CS must follow up manually.

---

### On Delivery Notification (non-blocking)

**Trigger:** Lyfe Order transitions to `Completed`, AND the `On Delivery` stage payment has not been received.

**Recipients:** CS team only (`customer@lyfehardware.com`)

**Email subject:**
```
[On Delivery Payment Pending] LH-ORD-2024-00456 — Order Completed
```

Same body format as Before Shipped, with stage = `On Delivery` and trigger state = `Completed`.

**This is a one-time notification only.** No daily reminders for On Delivery (the order is already delivered — the system cannot block anything).

---

## 8. Payment Recording

### How Payments Are Recorded

Payments are recorded via the **Add Payment Entry** endpoint (`lh.lyfe_hardware.integrations.shopify.add_payment_entry`), called from:
- Shopify webhook (automatic on customer payment)
- CS manual entry from the Quotation form

### Validation at Payment Entry

**Zero or negative amount:**
```
┌─────────────────────────────────────────────────────┐
│ (frappe.throw)                                      │
├─────────────────────────────────────────────────────┤
│ Payment Amount must be greater than zero.           │
└─────────────────────────────────────────────────────┘
```

**Quotation already Won (fully paid):**
```
┌─────────────────────────────────────────────────────┐
│ (frappe.throw)                                      │
├─────────────────────────────────────────────────────┤
│ This quotation is already Won. No further payments  │
│ can be recorded.                                    │
└─────────────────────────────────────────────────────┘
```

**Duplicate reference:**
No error shown. Returns `{"status": "duplicate_skipped", "total_received": ...}` silently. No payment is double-counted.

### After Payment Is Recorded

The system automatically runs three post-payment actions:

1. **Clear payment blocks** — `clear_payment_blocks_if_resolved` checks all linked Lyfe Orders. If `received ≥ threshold`, the block is cleared and the order can proceed.
2. **Update milestone statuses** — `update_payment_milestone_statuses` marks stages as `Paid` wherever the threshold is now met.
3. **Quotation state update** — if total received now equals grand total, state can be advanced to `Won` via normal workflow.

---

## 9. Workflow State Map

### Quotation States Relevant to Payments

| State | Payment Effect |
|---|---|
| `Draft` | No payment action |
| `Partially Paid` | Advance stage marked Due; some payment received |
| `Won` | Full payment received; all stages should be Paid |
| `Lost` | All payment blocks on linked Lyfe Orders released |
| `Cancelled` | All payment blocks on linked Lyfe Orders released |

### Lyfe Order States Relevant to Payments

| State | Payment Stage Triggered | Blocks Workflow? |
|---|---|---|
| `Factory Assignment` (via warehouse change) | Advance Amount | **Yes** — blocks if not paid |
| `Approved` | Photo Approval | **Yes** — blocks if not paid |
| `Ready for Dispatch` | Before Dispatch | **Yes** — blocks if not paid |
| `Shipped` / `Awaiting Tracking` | Before Shipped | No — notification only |
| `Completed` | On Delivery | No — notification only |

---

## 10. Role Reference

### Who Can See What

| Feature | Factory | Engineer | Customer Service | System Manager | Shipping Forwarder |
|---|---|---|---|---|---|
| Payment block — full details | ❌ | ❌ | ✅ | ✅ | ❌ |
| Payment block — generic message | ✅ | ✅ | — | — | ✅ |
| Payment Schedule fields (edit) | ❌ | ❌ | ✅ | ✅ | ❌ |
| Milestone Status field | Read-only | Read-only | Read/Write | Read/Write | Read-only |
| Daily reminder emails | ❌ | ❌ | ✅ (Days 1–4+) | — | ❌ |
| Escalation emails (Day 5+) | ❌ | ❌ | ✅ | — | ❌ |
| Founder escalation emails | Founders only | Founders only | Founders only | Founders only | Founders only |
| Before Shipped / On Delivery notifications | ❌ | ❌ | ✅ | — | ❌ |

### Role Definitions (for Payment System)

- **Customer Service** — responsible for all payment follow-up; sees full amounts; receives all reminder and notification emails
- **Factory** — production team; sees generic block message; does not receive payment emails
- **Engineer** — technical review team; same as Factory for payment purposes
- **System Manager** — sees full payment details (same as CS); does NOT automatically receive emails unless also added to Team Settings founders
- **Founders** — receive escalation emails from Day 5 onwards; configured in `Team Settings → founders` child table

---

## Appendix — Configuration Reference

### Team Settings Fields

| Field | Purpose | Where to Set |
|---|---|---|
| `founders` | Child table of User links — escalation recipients from Day 5 | Team Settings form |
| `custom_payment_reminder_slack_webhook` | Slack webhook URL for all payment notifications; blank = Slack disabled | Team Settings form |

### Scheduler Entry

```python
"0 9 * * *": [  # 9 AM daily
    "lh.lyfe_hardware.tasks.payment_reminders.run_payment_reminder_scheduler"
]
```

### Patch List (Payment System)

| Patch | What It Adds |
|---|---|
| `add_quotation_payment_tracking_fields` | `custom_payment_entries` table + `custom_total_received_amount` on Quotation |
| `add_payment_block_tracking_fields` | `custom_payment_block_stage`, `custom_payment_block_date`, `custom_payment_reminder_count` on Lyfe Order |
| `add_payment_reminder_slack_webhook_to_team_settings` | `custom_payment_reminder_slack_webhook` on Team Settings |
| `add_payment_schedule_stage_checkboxes` | 5 stage checkboxes on Payment Schedule |
| `add_before_shipped_stage_checkbox` | `custom_before_shipped` checkbox on Payment Schedule |
| `add_payment_milestone_status_field` | `custom_payment_status` (Select) on Payment Schedule |

---

*End of document — LH Payment Terms System*
