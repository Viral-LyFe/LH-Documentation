# LH Payment Terms & Milestone System — Full Documentation

> **BRD Reference:** LH-BRD-DEV-004 Payment Terms and Urgent SLA v1.1  
> **Last updated:** 2026-06-17 (rev 2)  
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
9. [Shopify Order Confirmation on Payment](#9-shopify-order-confirmation-on-payment)
10. [Quotation Cancellation — Workflow](#10-quotation-cancellation--workflow)
11. [Workflow State Map](#11-workflow-state-map)
12. [Role Reference](#12-role-reference)

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

### 7.3 Next-Stage Due Notification (Proactive)

**Trigger:** A Payment Schedule row transitions from `Upcoming` → `Due` as a result of `update_payment_milestone_statuses()` being called. This happens when a previous stage has just been marked `Paid` (i.e. payment was received that satisfies the previous stage's threshold).

**Purpose:** CS receives a proactive heads-up to start collecting the next instalment *before* the order reaches the blocking point and production is held up.

**Recipients:** CS team only (`customer@lyfehardware.com`)

**Email subject:**
```
[Next Payment Due] LHQ-2026-0120 — Photo Approval (₹ 3,000.00 outstanding)
```

**Email body:**

```
The previous payment stage has been completed for Quotation LHQ-2026-0120
(Acme Corp). The next stage — Photo Approval — is now Due.

┌──────────────────────────────────────┬──────────────────────────────┐
│ Quotation                            │ LHQ-2026-0120                │
│ Customer                             │ Acme Corp                    │
│ Next Stage                           │ Photo Approval               │
│ Stage Amount Required (cumulative)   │ ₹ 6,000.00                   │
│ Received so far                      │ ₹ 3,000.00                   │
│ Outstanding                          │ ₹ 3,000.00                   │
│ Order Total                          │ ₹ 10,000.00                  │
└──────────────────────────────────────┴──────────────────────────────┘

Please follow up with the customer to collect the Photo Approval payment
before it becomes overdue and blocks the next workflow step.
```

**Slack notification** is also sent (if webhook is configured in Team Settings):

```
*Next Payment Stage Due* | Quotation: `LHQ-2026-0120` (Acme Corp)
Stage: *Photo Approval* is now *Due*.
Required: ₹ 6,000.00 | Received: ₹ 3,000.00 | Outstanding: *₹ 3,000.00*
Please collect this payment before the order is blocked.
```

**This is a one-time notification per stage transition.** It fires once when the stage becomes Due. If CS does not act and the order eventually gets blocked (Lyfe Order tries to advance), the daily block reminder system takes over.

#### Example — 40/30/30 split (₹10,000 order)

| Event | What CS receives |
|---|---|
| Quotation Won — Advance (40%) due | Nothing — this is the *first* stage; CS already knows from the won notification |
| Customer pays ₹4,000 — Advance Paid | ✅ **"Next Payment Due — Photo Approval (₹3,000 outstanding)"** |
| Customer pays ₹3,000 — Photo Approval Paid | ✅ **"Next Payment Due — Before Dispatch (₹3,000 outstanding)"** |
| Customer pays ₹3,000 — Before Dispatch Paid | No notification — all stages complete |

> **Important:** If Slack is not configured in Team Settings, the Slack part of this notification is silently skipped. The email still sends.

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

The system automatically runs four post-payment actions:

1. **Clear payment blocks** — `clear_payment_blocks_if_resolved` checks all linked Lyfe Orders. If `received ≥ threshold`, the block is cleared and the order can proceed.
2. **Update milestone statuses** — `update_payment_milestone_statuses` marks stages as `Paid` wherever the threshold is now met.
3. **Next-stage due notification** — if any Payment Schedule row transitions from `Upcoming` → `Due` as a result of the payment (i.e. a previous stage just became Paid), a proactive email + Slack notification is sent to CS. See [Section 7.3](#73-next-stage-due-notification-proactive).
4. **Quotation state update** — if total received now equals grand total, state can be advanced to `Won` via normal workflow.

---

## 9. Shopify Order Confirmation on Payment

When CS clicks **Confirm Payment** or **Confirm Remaining Payment** on a Quotation, a dialog appears to record the payment amount. An optional checkbox is shown if the Shopify draft order has not yet been confirmed as a real Shopify order.

### Checkbox Condition

| Condition | Checkbox Shown? |
|---|---|
| `custom_shopify_draft_order_id` is set **and** `custom_shopify_order_name` is blank | ✅ Yes — Shopify draft is still open |
| `custom_shopify_order_name` is already set | ❌ No — Shopify order already confirmed |
| No Shopify draft order on the Quotation | ❌ No — not a Shopify order |

### Checkbox Behaviour

**Label:** "Confirm Order in Shopify after Payment Recording"  
**Default:** Unchecked

When **unchecked:** Only the ERP payment entry is recorded. No Shopify API call is made.

When **checked:**
1. ERP payment entry is recorded and committed first (always succeeds).
2. After ERP commit, the Shopify draft order is confirmed via GraphQL `draftOrderComplete` mutation.
3. On success: `custom_shopify_order_name` is populated (via the Shopify webhook that fires after order creation).
4. On Shopify API failure: ERP payment remains saved. An orange warning dialog is shown:

```
┌─────────────────────────────────────────────────────────────────────┐
│ Shopify Confirmation Failed                             (orange)     │
├─────────────────────────────────────────────────────────────────────┤
│ Payment was recorded successfully, but the Shopify order            │
│ confirmation failed.                                                │
│                                                                     │
│ Error: <API error detail>                                           │
│                                                                     │
│ You may retry Shopify confirmation manually.                        │
└─────────────────────────────────────────────────────────────────────┘
```

> **Why GraphQL instead of REST?**  
> The Shopify REST endpoint `POST /draft_orders/{id}/complete.json` returns HTTP 406 when the draft order has no payment gateway set — which is always the case for manual bank transfer orders. The GraphQL `draftOrderComplete` mutation handles this correctly regardless of payment gateway.

### Slack Configuration Note

The Slack webhook field in Team Settings (`custom_payment_reminder_slack_webhook`) is **optional**. If blank, all Slack notifications (payment reminders, stage-due notifications, shipped/delivery alerts) are silently skipped. No error is raised and no email is affected. This applies to all payment notifications throughout the system.

---

## 10. Quotation Cancellation — Workflow

CS can cancel a Quotation from **any submitted state** using the standard workflow action button.

### States Where Cancel Is Available

| From State | Action | Next State | Allowed Role |
|---|---|---|---|
| Sent to Customer | Cancel | Cancelled | Customer Service |
| Bank Transfer Selected | Cancel | Cancelled | Customer Service |
| Payment Received | Cancel | Cancelled | Customer Service |
| Won | Cancel | Cancelled | Customer Service |
| Expired | Cancel | Cancelled | Customer Service |
| Lost | Cancel | Cancelled | Customer Service |
| Partially Paid | Cancel | Cancelled | Customer Service |

> **Note:** `Cancelled` state has `doc_status = 2`, which is the standard Frappe cancelled document status. This state also triggers EC-6: all payment blocks on linked Lyfe Orders are automatically released and no further reminders are sent.

### Patch

Added by: `lh.patches.add_cancel_transitions_to_quotation_workflow` (Workflow: `Quotation-3`)

---

## 11. Workflow State Map

### Quotation States Relevant to Payments

| State | Payment Effect |
|---|---|
| `Draft` | No payment action |
| `Partially Paid` | Advance stage marked Due; some payment received |
| `Won` | Full payment received; all stages should be Paid |
| `Lost` | All payment blocks on linked Lyfe Orders released (EC-6) |
| `Cancelled` | All payment blocks on linked Lyfe Orders released (EC-6) |

### Lyfe Order States Relevant to Payments

| State | Payment Stage Triggered | Blocks Workflow? |
|---|---|---|
| `Factory Assignment` (via warehouse change) | Advance Amount | **Yes** — blocks if not paid |
| `Approved` | Photo Approval | **Yes** — blocks if not paid |
| `Ready for Dispatch` | Before Dispatch | **Yes** — blocks if not paid |
| `Shipped` / `Awaiting Tracking` | Before Shipped | No — notification only |
| `Completed` | On Delivery | No — notification only |

---

## 12. Role Reference

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
| `add_cancel_transitions_to_quotation_workflow` | Cancel action from all 7 submitted Quotation states → Cancelled |

---

## Post-Deployment Configuration Guide

This section covers everything that must be done **manually on the live system** after running `bench migrate` and the patches. The patches add fields and schema — they do not populate live data or connect external services.

---

### Step 1 — Run the Migration

```bash
# On the production server, from frappe-bench directory:
bench --site <your-site> migrate
```

This runs all pending patches automatically (they are registered in `patches.txt`). Verify each payment-related patch ran without error in the migration output:

```
lh.patches.add_quotation_payment_tracking_fields    ✓
lh.patches.add_payment_block_tracking_fields        ✓
lh.patches.add_payment_reminder_slack_webhook_to_team_settings  ✓
lh.patches.add_payment_schedule_stage_checkboxes    ✓
lh.patches.add_payment_terms_template_detail_stage_checkboxes   ✓
lh.patches.add_payment_stage_summary_html_to_lyfe_order         ✓
lh.patches.move_quotation_payment_history_section               ✓
lh.patches.add_payment_milestone_status_field       ✓
lh.patches.add_before_shipped_stage_checkbox        ✓
```

If any patch shows an error, investigate before proceeding.

---

### Step 2 — Configure the Slack Webhook (Team Settings)

**Where:** Frappe Desk → Settings → Team Settings → *Payment Reminder Slack Webhook* field

**What to do:**

1. First, create a **Slack Webhook URL** document in Frappe:
   - Go to: `Integrations → Slack Webhook URL → New`
   - Paste your Slack incoming webhook URL (from Slack App configuration)
   - Save

2. Then open **Team Settings** and set the `Payment Reminder Slack Webhook` field to the Slack Webhook URL document you just created.

**If left blank:** Slack notifications are silently skipped. Email notifications still send. This is safe — blank = Slack disabled.

**Which Slack channel receives messages:** Whichever channel is configured in the Slack App's incoming webhook (this is set in Slack's app settings, not in Frappe).

---

### Step 3 — Configure Escalation Recipients (Team Settings → Founders)

**Where:** Frappe Desk → Settings → Team Settings → *Founders* child table

**What to do:**

The `Founders` child table must contain the Frappe User records for anyone who should receive **escalation emails from Day 5 onwards** when a payment block is unresolved.

1. Open **Team Settings**
2. In the **Founders** child table, add rows for each founder/escalation recipient
3. Each row links to a **User** — the system uses that user's registered email address

**If the table is empty:** Escalation emails on Day 5+ are sent only to `customer@lyfehardware.com`. No error occurs — the founders list is simply empty.

**Example configuration:**

| User | Email (auto-resolved) |
|---|---|
| viralk@lyfehardware.com | viralk@lyfehardware.com |
| ceo@lyfehardware.com | ceo@lyfehardware.com |

---

### Step 4 — Verify the CS Email Address

**File:** `apps/lh/lh/lyfe_hardware/tasks/payment_reminders.py`, line 21

```python
CS_EMAIL = "customer@lyfehardware.com"
```

This address is hardcoded and receives:
- All payment block reminder emails (Days 1 onwards)
- Before Shipped payment pending notification
- On Delivery payment pending notification

**Action:** Confirm `customer@lyfehardware.com` is the correct inbox for the CS team on the live site. If the address needs to change, update the `CS_EMAIL` constant and redeploy.

---

### Step 5 — Verify the Daily Scheduler Is Running

The payment reminder scheduler runs at 9:00 AM every day. After deployment, confirm Frappe's background workers are running:

```bash
bench status
```

Look for `frappe-schedule` in the process list. If not running:

```bash
bench start   # for development
# or
sudo supervisorctl start frappe-schedule   # for production
```

You can manually trigger the scheduler once to test it:

```bash
bench --site <your-site> execute \
  lh.lyfe_hardware.tasks.payment_reminders.run_payment_reminder_scheduler
```

Expected output: runs silently if no blocked orders exist, or sends reminders for any blocked orders found.

---

### Step 6 — Set Up Payment Schedule on Existing Quotations

The new `custom_payment_status` field on Payment Schedule rows defaults to blank for all existing quotations. **Existing live quotations are not retroactively updated.**

For any active quotation (not Lost/Cancelled) that has a Payment Schedule:

1. Open the Quotation
2. Save it once — this triggers `on_update` which runs `update_payment_milestone_statuses` and fills the status column

Alternatively, run the backfill from the bench console:

```bash
bench --site <your-site> console
```

```python
import frappe
from lh.lyfe_hardware.doc_events.quotation import update_payment_milestone_statuses

active = frappe.get_all(
    "Quotation",
    filters={"workflow_state": ["not in", ["Lost", "Cancelled", "Draft"]]},
    fields=["name"],
)
for q in active:
    try:
        update_payment_milestone_statuses(q.name)
        frappe.db.commit()
    except Exception as e:
        print(f"Failed for {q.name}: {e}")

print(f"Done — processed {len(active)} quotations")
```

---

### Step 7 — Configure the Default Payment Terms Template

**Where:** Frappe Desk → Accounts → Payment Terms Template

The system defaults new Quotations to the template named `"100 % Advance"`. Verify this template exists on the live site:

1. Go to: `Accounts → Payment Terms Template`
2. Search for `100 % Advance`
3. If it does not exist, create it with one row: 100%, `custom_advance_amount` checkbox checked

If the template name differs on your live site, update the `validate()` function in `quotation.py`:

```python
# apps/lh/lh/lyfe_hardware/doc_events/quotation.py, in validate()
if not doc.payment_terms_template:
    doc.payment_terms_template = "100 % Advance"   # ← update this name if different
```

---

### Step 8 — Verify Terms and Conditions Template

The system defaults to a Terms and Conditions doc named `"Terms and Conditions template"`. Verify it exists:

1. Go to: `CRM → Terms and Conditions`
2. Search for `Terms and Conditions template`
3. If missing, create the document (content can be your standard T&C text)

---

### Step 9 — Smoke Test the Full Flow

After all configuration is done, run a quick end-to-end test:

#### 9.1 Payment Gate Block Test

1. Create a test Quotation with grand total ₹1,000
2. Add a Payment Schedule row: 50% (₹500), check `Advance Amount`
3. Add another row: 50% (₹500), check `Photo Approval`  
4. Submit / move to `Sent to Customer` state
5. Create a linked Lyfe Order (or use an existing one with `source_quotation` set)
6. Set the Lyfe Order `warehouse` to `Factory - LH`
7. **Expected result:**
   - As **Factory role** → `Payment Incomplete — This order cannot proceed — payment is pending. The CS team has been notified.`
   - As **CS role** → `Payment Incomplete — Payment required before assigning order to Factory. Quotation: ... Required: ₹500.00. Received: ₹0.00`
8. Check the Lyfe Order: `custom_payment_block_stage` should be `custom_advance_amount`, `custom_payment_block_date` should be today

#### 9.2 Payment Recording Test

1. Continue from above — add a payment entry of ₹500 to the Quotation
2. **Expected result:**
   - `custom_payment_block_stage` is cleared on the Lyfe Order
   - Payment Schedule row 1 `custom_payment_status` = `Paid`
   - Payment Schedule row 2 `custom_payment_status` = `Upcoming`

#### 9.3 Slack Test

1. While a payment block is active, manually run:
   ```bash
   bench --site <your-site> execute \
     lh.lyfe_hardware.tasks.payment_reminders.run_payment_reminder_scheduler
   ```
2. **Expected result:** A message appears in your configured Slack channel with order details

#### 9.4 Escalation Test (optional)

1. Set `custom_payment_reminder_count = 5` on a blocked Lyfe Order directly via the console
2. Run the scheduler again
3. **Expected result:** Founders receive the email alongside CS

#### 9.5 Next-Stage Due Notification Test

1. Set up a Quotation with a 50/50 payment schedule (Advance + Photo Approval)
2. Record payment for the Advance amount
3. **Expected result:** CS inbox (`customer@lyfehardware.com`) receives an email with subject `[Next Payment Due] <quotation> — Photo Approval`
4. If Slack is configured, a message appears in the payment reminder channel

#### 9.6 Shopify Confirmation Checkbox Test

1. Open a Quotation in `Bank Transfer Selected` state that has `custom_shopify_draft_order_id` set and `custom_shopify_order_name` blank
2. Click **Confirm Payment**
3. **Expected result:** The payment dialog includes a checkbox "Confirm Order in Shopify after Payment Recording" (unchecked by default)
4. Check the box and record the payment
5. **Expected result:** Payment records successfully; Shopify draft order is converted to a real order; no error shown

#### 9.7 Cancel Workflow Test

1. Move a Quotation to `Sent to Customer` state
2. Click **Cancel** workflow action button
3. **Expected result:** Quotation moves to `Cancelled` (doc_status=2); all payment blocks on linked Lyfe Orders are released

---

### Configuration Summary Table

| # | What | Where in Frappe | Required? |
|---|---|---|---|
| 1 | Run `bench migrate` | Server terminal | **Mandatory** |
| 2 | Set Slack Webhook URL | Team Settings → Payment Reminder Slack Webhook | Optional (disables Slack if blank) |
| 3 | Add escalation recipients | Team Settings → Founders child table | Optional (CS-only if blank) |
| 4 | Verify `CS_EMAIL` constant | `payment_reminders.py` line 21 | **Mandatory** — confirm email is correct |
| 5 | Verify background workers running | `bench status` | **Mandatory** |
| 6 | Backfill milestone statuses on existing quotations | Bench console script | Recommended |
| 7 | Verify `100 % Advance` payment terms template exists | Accounts → Payment Terms Template | **Mandatory** |
| 8 | Verify `Terms and Conditions template` exists | CRM → Terms and Conditions | **Mandatory** |
| 9 | Run smoke tests | Frappe Desk + bench console | Strongly recommended |

---

### Hardcoded Values Reference

The following values are hardcoded in source and require a code change (not a config change) if they need to differ on the live site:

| Value | Location | Default | When to Change |
|---|---|---|---|
| CS email address | `payment_reminders.py:21` | `customer@lyfehardware.com` | If CS inbox changes |
| Escalation threshold | `payment_reminders.py:22` | Day 5 (`ESCALATION_DAYS = 5`) | If escalation timing needs adjustment |
| Default payment terms template | `quotation.py` in `validate()` | `"100 % Advance"` | If template name differs on live site |
| Default T&C template | `quotation.py` in `validate()` | `"Terms and Conditions template"` | If T&C doc name differs on live site |
| Approval threshold (grand total) | `quotation.py` in `validate()` | ₹10,000 | If approval threshold changes |
| Factory warehouse name | `lyfe_order.py` | `"Factory - LH"` | If warehouse is renamed |

---

*End of document — LH Payment Terms System (rev 2, 2026-06-17)*
