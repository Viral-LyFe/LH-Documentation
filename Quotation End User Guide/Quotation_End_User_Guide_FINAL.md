# LYFE HARDWARE — Quotation End User Guide

**Version 3.0 • October 2026 • Internal Use**

This guide explains how to work with Quotations in the Lyfe Hardware system, from the first inquiry to collecting payment. It describes how the system behaves today.

> Items marked **[CONFIRM WITH TEAM]** could not be fully verified and should be checked before this guide is circulated.
> Items marked **[IMAGE PLACEHOLDER: …]** show where a screenshot should be added.

---

## Table of Contents

1. What Is a Quotation?
2. Roles — Who Does What
3. Workflow Overview and All States
4. Creating a Quotation
5. Filling in the Items
6. Order Type — Standard and Custom
7. Shipping, Charges and Taxes
8. Payment Terms and Payment Schedule
9. The Approval Process
10. Engineer Review, Senior Review and Drawings
11. Factory Review (Cost of Goods)
12. Urgent and VIP Quotations — Dispatch Dates
13. Sending the Quotation to the Customer
14. Email Tracking Panel
15. Payments — Card and Bank Transfer
16. Revising a Sent Quotation (Cancel and Amend)
17. Automatic Follow-Ups
18. Expiry and Lost
19. Won and What Happens Next
20. Dashboards
21. Common Errors and How to Fix Them
22. Quick Reference — Workflow Diagram
23. Frequently Asked Questions

---

## 1. What Is a Quotation?

A **Quotation** is the formal price proposal sent to a customer before they place an order. In Lyfe Hardware, a Quotation:

- Lists every product the customer wants, with quantities and prices.
- Calculates shipping, taxes and the grand total.
- Is emailed to the customer as a PDF, with a button to pay by card (when enabled) and a button to pay by bank transfer.
- Tracks whether the customer opened the email and clicked a payment button.
- Creates follow-up tasks for Customer Service if the customer does not respond.
- Records every payment received (full or partial). The Quotation becomes **Won** when it is fully paid.

[IMAGE PLACEHOLDER: A completed Quotation form, full screen]

---

## 2. Roles — Who Does What

| Role | What they do on a Quotation |
|---|---|
| **Customer Service (CS)** | Creates quotations, reviews inquiries, sends them for approval and to the customer, confirms payments, marks Won/Lost, cancels and amends. Performs almost every workflow action. |
| **Engineer** | Reviews custom items, attaches the Order Drawing, completes the engineering review, can request a Senior Review. |
| **Senior Engineer** | Sees only the quotations assigned to them for Senior Review, and approves or returns them. Customer, address and price fields are read-only for them. |
| **Factory** | Fills in **Cost of Goods (COGS)** and shipping details, then marks the quotation Feasibility Approved or Not Feasible. Only Factory (and Super Admin) can see and edit Cost of Goods. |
| **Quotation Approvar** (the “manager” role) | Approves quotations that need approval, or sends them back to CS. |
| **Super Admin** | Administrative access. The only workflow action they can take is manually expiring a Sent quotation. |
| **Sales User / Sales Manager** | Can open and edit quotations at some stages, but **cannot click any workflow action buttons** unless they also hold the Customer Service role. Sales Manager also sees the Senior Review “Red Flag” banner. |

> **Note:** Cost of Goods is hidden from Customer Service. If you do not see a Cost of Goods column, that is intentional.

---

## 3. Workflow Overview and All States

A new Quotation starts in **Inquiry** and moves through the states below. The buttons you see at the top of the form depend on your role and on the quotation’s current state.

### All states

| State | What it means | Who acts |
|---|---|---|
| Draft | Only reached when an Engineer sends an inquiry back using **Need More Information**. CS re-submits it with **New Inquiry**. | CS / Engineer |
| Inquiry | Where every new Quotation starts. CS decides whether the inquiry is viable. | CS |
| Inquiry Approved | CS accepted the inquiry. Next step depends on the items (see section 22). | CS |
| Inquiry Rejected | CS rejected the inquiry (a reason is required). **This is a dead end — no further actions.** | — |
| Pending Engineer Review | Engineer reviews feasibility and attaches the Order Drawing. | Engineer |
| Senior Review Pending | A Senior Engineer is reviewing a custom item. | Senior Engineer |
| Production Details Review | Factory fills in Cost of Goods and shipping details. | Factory |
| Not Feasible | Factory marked the quotation as not feasible (reason recorded). | CS |
| Feasibility Approved | Factory confirmed the quotation is feasible and costed. | CS |
| Edit Request Created | CS asked for a change after Feasibility Approved. | Engineer / Factory |
| Sales User Review | Engineer finished with no factory costing needed; CS reviews. | CS |
| Pending Approval | Waiting for the manager (Quotation Approvar) to approve. | Manager |
| Approved – Pending Send | Approved; ready for CS to send to the customer. | CS |
| Sent to Customer | Email sent; waiting for the customer. | CS |
| Bank Transfer Selected | Customer chose bank transfer. CS confirms payment when it arrives. | CS |
| Partially Paid | Some, but not all, of the grand total has been received. | CS |
| Payment Received | Older state. No new quotation enters it; you will only see it on older quotations. | CS |
| Won | Fully paid. | — |
| Lost | Customer did not respond or declined. | — |
| Expired | The Valid Till date passed while the quotation was still “Sent to Customer”. | — |
| Cancelled | Quotation cancelled (see section 16 for how to revise). | — |

> **Important:** The quotation is **locked (submitted)** from the moment it is sent to the customer. After that, items cannot be edited — to change anything you must Cancel and Amend (section 16).

[IMAGE PLACEHOLDER: Workflow state bar showing the current state on a Quotation]

---

## 4. Creating a Quotation

### Step 1 — Open a new Quotation
Go to **Selling → Quotation → New**.

[IMAGE PLACEHOLDER: New Quotation form]

### Step 2 — Choose who the quotation is for
- In **Quotation To**, choose **Customer**, **Lead**, **Prospect** or **CRM Deal**, then pick the name.
- Make sure a **Contact** with an **email address** is selected. Without it you will be blocked from sending (see section 21).
- Make sure an address is set (Shipping Address or Customer Address). It is also required before sending, and it is used to calculate US sales tax and the shipping destination.

### Step 3 — Set the Valid Till date
**Valid Till** is shown in the email to the customer. If the quotation is still in **Sent to Customer** after this date, the system marks it **Expired** automatically (section 18). By default the date is 30 days after the quotation date. **[CONFIRM WITH TEAM: source of the 30-day default]**

### Step 4 — Terms
- **Payment Terms** defaults to **100 % Advance**.
- **Terms and Conditions** defaults to the standard template.
- Change these only for a special arrangement. If you change payment terms into stages, see section 8.

### Step 5 — Bank Transfer checkbox (optional)
If the customer will pay by bank transfer, tick **Payment Via Bank Transfer**.

> **Warning:** Ticking it sets the discount on **every item row to 8%**. Unticking it sets every row’s discount back to **0%**, erasing any discounts you typed in manually. Tick or untick it *before* entering custom discounts.

### Step 6 — Save
After the first save, the quotation moves to **Inquiry** automatically. The system also adds a **Shipping Charges** row and a **Custom Fee** row to the taxes table (section 7).

### Step 7 — Submit the inquiry
From **Inquiry**, CS clicks one of:
- **Inquiry Approved** — the inquiry is viable; continue.
- **Inquiry Rejected** — a dialog asks for a **Rejected Reason** (required). The quotation then ends in Inquiry Rejected and cannot be reopened.

If the quotation is in **Draft** (because an Engineer sent it back), update it and click **New Inquiry** to resubmit.

[IMAGE PLACEHOLDER: Inquiry Rejected reason dialog]

---

## 5. Filling in the Items

### Adding items
In the **Items** table, click **Add Row** and search by item name or SKU.

[IMAGE PLACEHOLDER: Items table with a few rows]

### What you see in the item table
The Items table shows: **Item Code**, **Is Custom Order**, **Drawing Required ?**, **Lead Time**, **Qty**, **Price List Rate**, **Amount**, and a flag when the box size is incomplete. The *Rate* column is hidden — the table shows **Price List Rate** instead.

| Field | Notes |
|---|---|
| Item Code / Name | The product quoted. |
| Qty | Quantity requested. |
| Price List Rate | The list price per unit. For custom items it is calculated from Cost of Goods (see below). |
| Discount % | Optional. Subject to approval limits (section 9). |
| Is Custom Order | Row turns **purple**. Also applies automatically if the item name contains a custom-item word set in Selling Settings (for example “Custom”). |
| Drawing Required ? | Tick if an Order Drawing is needed. See section 10. |
| Lead Time | Free text, e.g. “4 weeks”. |
| Technical Specs | Free text for the engineer, inside the row form. |
| Cost of Goods | Factory only. Hidden from CS. |

[IMAGE PLACEHOLDER: A custom item row highlighted purple]

### Price List Rate for custom items
When Factory enters **Cost of Goods**, the system sets the **Price List Rate** automatically: Cost of Goods × the Item Group multiplier (default 3.3) × the finish multiplier. A message like “Price List Rate set to 3.3x COGS …” appears. If a rate is left at 0, the item’s retail price is used instead.

### Box sizes on item rows
The system fills box dimensions and weight from the item master, or from its bill of materials, and flags rows where dimensions are incomplete. Use **Tools → Fix Missing Box Sizes** (section 7).

### Calculations
- Amount = Qty × Rate after discount.
- Grand Total = item total + Shipping Charges + Custom Fee + taxes (US sales tax and any processing fee).
- IGST rows are removed automatically on save.

> **Note:** The Additional Discount section and the Shipping Rule field are hidden. Apply discounts per item row only.

---

## 6. Order Type — Standard and Custom

**Order Type** is calculated automatically every time you save:

- **Custom** — at least one item is marked **Is Custom Order**, or an item name contains a custom-item word.
- **Standard** — no custom items.

If you change Order Type by hand it is overwritten at the next save. **Export** appears in some code but cannot currently be selected, so there is no Export routing. **[CONFIRM WITH TEAM: is Export still planned?]**

What Order Type affects: **Request Senior Review** is only available on **Custom** orders (section 10).

---

## 7. Shipping, Charges and Taxes

### Shipping Charges and Custom Fee rows
Every quotation has two charge rows at the bottom of the **Taxes and Charges** table:

| Charge | What to enter |
|---|---|
| Shipping Charges | Estimated shipping cost in the Amount column. |
| Custom Fee | Customs or duty fee. Leave 0 if none. |

[IMAGE PLACEHOLDER: Taxes and Charges table with Shipping Charges and Custom Fee rows]

**When shipping is required:** The system does **not** stop CS from sending the quotation with 0 shipping. The shipping check happens when **Factory** finishes its review: Factory cannot click **Feasibility Approve** unless Shipping Charges is greater than 0 **or Free Shipping is ticked**. Still, always check the shipping amount before sending.

### Free Shipping
Tick **Free Shipping** when shipping is not charged. It satisfies the Factory shipping check.

### Getting a shipping rate (estimate only)
Use this to find out what shipping will cost.

1. **Fill in box details first.** Either fill in the box **Length, Width and Height**, or add rows to the **Parcel Size** table (click **Update Parcel Size**). The button will not show until one of these is filled in.
2. Go to **Tools → Get Shipping rate**. This is available while the quotation is not yet submitted (any state before it is sent), not only in Draft.
3. In the dialog choose:
   - **Carrier** (a default is pre-filled),
   - **Destination Country** (filled from the address; read-only),
   - **Signature Required** — tick if signature on delivery is needed,
   - **Is Commercial** — tick if the destination is a commercial address.
4. Review the result. You will see either one breakdown (chargeable weight, zone, base rate, surcharges, fuel surcharge, total), or a comparison across packing types.
5. **Apply it yourself.** The result is only a popup. It is **not** saved to the Quotation. Enter the amount in the **Shipping Charges** row and save.

[IMAGE PLACEHOLDER: Get Shipping rate dialog]
[IMAGE PLACEHOLDER: Shipping rate result popup]

### Update Parcel Size
**Update Parcel Size** opens a dialog with three tables — **Assembled**, **Semi Assembled**, **Disassembled**. Each row needs a weight and a count. A message “Parcel Size updated – N row(s)” appears; the change is saved only when you save the quotation. The button is visible to every user who can open the quotation.

### Tools → Fix Missing Box Sizes
Shown when a row has a bill of materials or a box size is missing. It lists the missing dimensions; enter Length, Width and Height, then click **Save & Recalculate**. If nothing is missing, you see “All BOM child items have box dimensions. Nothing to fix.”

### Box Suggestions
The system suggests catalogue boxes (“Best fit”, “Comfortable fit”, “Roomier fit”) with the billable weight, or says “No catalogue box found for combined shipment.” This is advice only.

### Shipment Contents
Optional field describing the shipment (Commercial, Gift, Sample, Returned Goods, Documents, Other), for customs on international shipments.

### Taxes
- **US sales tax** is added automatically when you save, if the shipping address is in the US. Tick **Is customer exempted from sales tax?** to skip it.
- **Apply Additional Charges** (only shown if enabled in Payment Fee Settings) adds a **Payment Processing Fee** row as a percentage of the grand total after tax.

---

## 8. Payment Terms and Payment Schedule

By default the whole amount is due in advance. If the customer agrees to pay in stages, edit the **Payment Schedule** table.

- Each row needs an **Invoice Portion** greater than 0 and not more than 100%.
- All portions must add up to exactly **100%**.
- Each row must be tied to a stage by ticking **one** stage checkbox on that row (only one per row, and each stage can be used once).
- If you choose the **On Delivery** stage, a warning reminds you this carries a collection risk.
- If you change the order total after setting the schedule, a warning “Payment Schedule Needs Review” tells you to update the amounts. It does not block saving.

**Why this matters:** After the quotation becomes an order, the order will be **blocked at the matching stage** (photo approval, before dispatch, before shipped, on delivery) if the money received so far is less than the schedule requires. CS sees “Payment required before <stage> …”. Other users see “This order cannot proceed – payment is pending. The CS team has been notified.” CS also gets a daily payment reminder.

[IMAGE PLACEHOLDER: Payment Schedule table with stage checkboxes]

---

## 9. The Approval Process

### When is approval needed?
The system re-checks this on **every save**. Approval is required if **any** of these is true:

1. An item discount is above its allowed maximum. The maximum comes from the item, else the item group, else the default of **5%** (**8%** when Payment Via Bank Transfer is ticked).
2. An item’s selling rate after discount is below its Cost of Goods plus the minimum margin.
3. The **grand total is 10,000 or more** (in the quotation’s currency).

If you tick **Approval Required** by hand it will be reset on the next save, so it cannot be forced manually. **[CONFIRM WITH TEAM]** The amount threshold is a raw number — it does not convert currency, even though some messages show “₹10,000”.

While you edit, an orange message may appear, for example *“Discount 12% exceeds the maximum allowed 5% for item …. Approval Required.”*

### Which button do I use?
Buttons depend on state:

| You are in… | Approval needed? | Button |
|---|---|---|
| Sales User Review | Yes (and Cost of Goods filled in) | **Send for Approval** |
| Sales User Review | No | **Send Quotation to Client** |
| Inquiry Approved / Feasibility Approved | Either | **Send for Approval** is always available; **Send Quotation to Client** only if no approval is needed |

### What CS does
1. Click **Send for Approval**.
2. If any item is missing Cost of Goods, you are blocked with “Production Cost details are missing…”. Route it to Factory first (**Assign To Factory** button) and try again after Factory completes.
3. The quotation moves to **Pending Approval**. A **Slack card** “Quotation Approval Needed” is posted to the CS Tech channel when moving from Draft, Sales User Review or Feasibility Approved.

### What the manager does
Review prices and discounts, then click:
- **Approve** → Approved – Pending Send.
- **Send Back to Sales User** → back to Sales User Review (the Slack version asks for a reason).
- **Send Quotation to Client** — only offered if no approval is actually required.

When a quotation is in **Pending Approval**, the manager sees a yellow prompt banner at the top of the form.

### Approving from Slack
Only users with the approver role can use the Slack buttons. Others see “You’re not authorized to act on Quotation approvals from Slack.” If the quotation is no longer Pending Approval, the card changes to “Out of Date”. **Slack’s “Send Quotation to Client” only changes the status — it does NOT email the customer or create a payment link.** Always send from the quotation form in the system, or use **Resend Email** afterwards (section 13).

> **Note:** When Factory approves feasibility on an order of 10,000 or more, the quotation goes to Pending Approval directly **without** a Slack card and without the Cost of Goods check. Managers should look in the system for such quotations.

### Urgent priority shortcut
For **Urgent** quotations, the “Cost of Goods missing” check on Send for Approval and the “Order Drawing required” check on Completed are skipped. Other conditions still apply.

[IMAGE PLACEHOLDER: Pending Approval banner and Approve / Send Back buttons]
[IMAGE PLACEHOLDER: Slack approval card]

---

## 10. Engineer Review, Senior Review and Drawings

This applies when a quotation needs engineering input.

### Request for Technical Details (CS)
From **Inquiry Approved** or **Sales User Review**, CS can click **Request for Technical Details** when **Drawing Required** is ticked. The quotation moves to **Pending Engineer Review**.

### What the Engineer does
1. Review each custom item for feasibility.
2. If a drawing is required, attach the **Order Drawing** (one drawing for the whole quotation — there is no per-item image attachment).
3. Click:
   - **Completed** → goes to **Production Details Review** (if a feasibility test is needed) or **Sales User Review**.
   - **Need More Information** → sends it back to **Draft** for CS to update and resubmit.
   - **Request Senior Review** → Custom orders only (see below).

The system blocks **Completed** if Drawing Required is on and no Order Drawing is attached: *“Drawing Required is enabled but no Order Drawing is provided. Please add a drawing before completing.”* (Not enforced for Urgent quotations.)

### Attaching the Order Drawing
1. **Log a Timesheet** on the quotation first. Without it: *“Please log a Timesheet for this Quotation before uploading a drawing.”*
2. Attach the drawing file.
3. A **Drawing Change Details** dialog opens. You must say what changed on the drawing. It cannot be dismissed. If you click **Cancel Upload**, the drawing is removed. If the dialog is left unanswered, the upload is **automatically undone after about 15 minutes**.
4. Pages in a PDF titled “BOM / Breakdown of Components” are stripped out automatically.
5. Each change is logged as a Comment and notified by email to the drawing-changes mailbox.

Once an Order Drawing is attached, **Drawing Required** is cleared automatically.

[IMAGE PLACEHOLDER: Drawing Change Details dialog]

### Senior Review
On a **Custom** order in **Pending Engineer Review**, the Engineer can click **Request Senior Review**. The dialog asks for:
- **Reason Code** (New geometry, Structural concern, Material/finish uncertainty, Customer spec ambiguity, Mould/tooling doubt, Other) — required,
- **Short Note** — required,
- Finish Family, Category and Key Dimensions.

It shows earlier Senior Reviews for context. The quotation goes to **Senior Review Pending** and is assigned to a Senior Engineer, who clicks **Senior Review Approve** or **Senior Review Return**; either way it returns to **Pending Engineer Review**.

**If the specification or drawing changes after approval**, the earlier approval is cleared automatically, and you see *“The drawing/specification changed after approval… routed back to Pending Engineer Review.”*

Sales Managers and managers see a **Red Flag** banner on quotations with senior review concerns.

### Other automatic routing
- Ticking **Feasibility Test Required** moves a Draft, Sales User Review or Pending Engineer Review quotation to **Production Details Review**.
- If **Drawing Required** is ticked while in Production Details Review, the quotation goes back to **Pending Engineer Review**.
- The feasibility flag clears itself once all Cost of Goods are filled in and shipping is entered (or Free Shipping is ticked).

---

## 11. Factory Review (Cost of Goods)

In **Production Details Review** the Factory sees an orange **Cost of Goods Pending** banner listing the rows that need attention.

1. Enter **Cost of Goods** for every item. The Price List Rate is calculated automatically.
2. Enter **Shipping Charges** (or tick **Free Shipping**).
3. Click one of:
   - **Feasibility Approve** → **Feasibility Approved** (or **Pending Approval** if the grand total is 10,000 or more).
   - **Not Feasible** → a dialog requires a **Not Feasible Reason**; the quotation goes to **Not Feasible**. CS then clicks **CS Review** to move it to **Sales User Review**.

After **Feasibility Approved**, CS can **Send for Approval**, **Send Quotation to Client** (if no approval is needed), or **Required a Change** (→ Edit Request Created; Engineer or Factory then click **Changes Completed** to return it to Feasibility Approved).

If you try **Feasibility Approve** with missing details you see: *“Details missing from Row 1, Row 2 Factory. Please fill in the required fields before completing.”* (Factory) or *“This quotation cannot be completed yet — factory still needs to update the item details.”* (others).

### Factory Quotation Dashboard
Factory can work from the **Factory Quotation Dashboard** (queue of quotations in Production Details Review, with what is blocking each one — Cost of Goods or shipping). From it you can edit shipping, Cost of Goods and parcels, comment with @mentions, and use Feasibility Approve / Not Feasible. Only Factory, System Manager or Super Admin can save from it.

[IMAGE PLACEHOLDER: Factory Quotation Dashboard]

---

## 12. Urgent and VIP Quotations — Dispatch Dates

**Priority** can be **Normal**, **Urgent** or **VIP**. Selecting Urgent or VIP opens the **Calculate Dispatch Dates** dialog, which gives a verdict of **Possible**, **Borderline** or **Not Possible**. Click **Confirm & Apply** to set Order Type, Priority and Estimated Delivery Date, or **Skip for now**. You can also open it from the **Urgent** menu while the quotation is not submitted.

A banner on the form shows the working days left, **AT RISK** when 2 days or fewer remain, and **OVERDUE** after that. Urgent quotations skip the Cost of Goods and Order Drawing checks described in sections 9 and 10.

**[CONFIRM WITH TEAM: whether CS users can save the Estimated Delivery Date from this dialog]**

[IMAGE PLACEHOLDER: Calculate Dispatch Dates dialog]

---

## 13. Sending the Quotation to the Customer

### Before you send
The **Send Quotation to Client** button is only shown when **all** are true:
- No approval is outstanding (**Approval Required** is off, or the quotation has been **Approved – Pending Send**),
- Drawing Required is not ticked,
- No feasibility test is pending,
- The grand total is greater than 0.

The system also requires a **contact email** and an **address** (section 21).

### Steps
1. Open the quotation and click **Send Quotation to Client**.
2. If payment links are enabled and this is not a bank-transfer quotation, the system shows “Generating payment link…”.
3. “Sending quotation email…” appears, then “Quotation email sent.”
4. The quotation becomes **Sent to Customer** and is **locked**.

[IMAGE PLACEHOLDER: Send Quotation to Client button and progress messages]

### What the customer receives
- Subject: **“Your Quotation <number> from Lyfe Hardware”** (with revision number if applicable).
- The quotation PDF is **attached** (“Quotation – No Bank Details”). The email has no separate PDF download link.
- A summary of items (with list price and discount), Shipping, Custom Fee, Tax and the Grand Total.
- The **Valid until** date.
- A **“Pay now – secure payment link”** button — shown only when payment links are enabled and the quotation is not a bank-transfer one (a 3% card processing fee note is shown).
- A **“Pay by bank transfer”** button — always shown (ACH/Wire, no fee).
- “What happens next”: make payment; production begins within 1 business day of payment confirmation; photo approval before dispatch.
- Support contact details.

Customer replies come back on the same email thread.

### Resend Email
After the email has been sent, a **Resend Email** button appears. It asks for confirmation (and tells you if it will resend only the bank-transfer version). Use it if the customer says they did not get the email, or if you sent the quotation from Slack.

### If the email cannot be sent
If no default outgoing email account is configured you will see “No default outgoing email account is configured…”. Ask an administrator.

[IMAGE PLACEHOLDER: Customer email as received]

---

## 14. Email Tracking Panel

After sending, open the **Payment & Tracking** tab. A tracking panel (visible to Customer Service and Super Admin) shows:

| Signal | Meaning |
|---|---|
| Email Sent | The email was sent. |
| Email Opened | The customer opened the email. This is best-effort; some email apps (e.g. Apple Mail, Outlook) block it, so *not opened* does not always mean unread. |
| Payment Link Clicked | The customer clicked the card payment button. |
| Action Taken | The last button the customer used: Pay by Card, Pay by Bank Transfer, or PDF Viewed. |

These are **indicators only** and do not change anything — **with one exception:** when the customer clicks **Pay by Bank Transfer** while the quotation is **Sent to Customer**, the quotation automatically moves to **Bank Transfer Selected**, the customer is emailed bank instructions (every time they click), and an internal notice is sent to support.

> **Note:** The internal notice still says to set the status to “Payment Received”. Ignore that — use **Confirm Payment** (section 15).

[IMAGE PLACEHOLDER: Email & Payment Tracking panel]

---

## 15. Payments — Card and Bank Transfer

All payments are listed in the **Payment History** table (Payment & Tracking tab). **Total Received Amount** is the sum.

| Column | Meaning |
|---|---|
| Payment Date | When it was received |
| Payment Type | Full Payment or Partial Payment |
| Payment Mode | Manual Payment or Payment Link |
| Payment Amount | Amount of that payment |
| Reference / Note | Bank reference, Shopify order number or note |

Payment History rows are locked. If a row was entered by mistake, contact the system administrator.

[IMAGE PLACEHOLDER: Payment History table and Total Received Amount]

### Card payment (automatic)
1. The customer pays through the secure payment link.
2. The system detects it immediately (and re-checks every 5 minutes, and every few seconds for 15 minutes after sending).
3. A Payment History row is added (Mode = Payment Link, Reference = the Shopify order number).
4. The quotation moves to **Won** if fully paid, otherwise to **Partially Paid**.

CS sees a green message **“Payment confirmed! Quotation marked as Won.”** This message appears even for a partial payment — always check the status. No action is needed from CS. If an online payment is not reflected after 10 minutes, contact the administrator.

### Bank transfer (manual confirmation)
1. The customer clicks **Pay by Bank Transfer** (or CS clicks **Bank Transfer Selected** from Sent to Customer). Status: **Bank Transfer Selected**.
2. The customer receives bank instructions by email.
3. When the money arrives, open the quotation and click **Confirm Payment**.
4. Fill in the **Record Payment** dialog:

| Field | What to enter |
|---|---|
| Payment Type | **Full Payment** (pre-fills the remaining balance) or **Partial Payment** (blank) |
| Payment Mode | **Manual Payment** for bank transfers/cheques |
| Payment Amount | Pre-filled with the remaining balance; change for partial |
| Reference / Note | Bank reference/UTR or cheque number |
| Confirm Order in Shopify after Payment Recording | Shown only when a Shopify draft order exists. Ticked by default. Converts the draft to a confirmed Shopify order. |

The dialog also shows *Grand Total, Already Received, Remaining*.

5. Click **Record Payment**. Status becomes **Won** (fully paid) or **Partially Paid**.

[IMAGE PLACEHOLDER: Record Payment dialog]

### Further payments
From **Partially Paid**, click **Confirm Remaining Payment**. Repeat until the balance reaches zero. The quotation then becomes **Won**.

### Protections and warnings
- **Overpayment:** If the amount is higher than the remaining balance you see *“Amount entered (…) exceeds the remaining balance (…). Please verify.”* Nothing is recorded until you correct it. (Amounts show in the quotation’s own currency.)
- **Duplicates:** A payment with the same reference as one already recorded is skipped.
- **Already Won / Cancelled:** you cannot record more payments (*“This quotation is already Won…”*, *“This quotation has been cancelled. No payments can be recorded against it.”*).
- **Zero amount** is rejected (*“Received Amount must be greater than zero.”*).
- **Marking Won with no payment:** *“Cannot mark as Won — no payment has been recorded…”* Record a payment first. If only part was paid, the status becomes Partially Paid instead of Won.
- **Shopify confirmation failed:** you see a message that the payment **was recorded** but confirming the Shopify order failed. Retry confirmation or inform the administrator.
- Pending bank payments on Shopify are **not** counted as received.

---

## 16. Revising a Sent Quotation (Cancel and Amend)

Once sent, a quotation is locked: **items cannot be edited**. To change a sent quotation:

1. Open the quotation and click **Cancel** (available from Sent to Customer, Bank Transfer Selected, Partially Paid, Won, Payment Received, Expired and Lost). Confirm.
2. Click **Amend** on the cancelled quotation. A new quotation is created with the same name plus **-1** (then -2, …).
3. Make the changes. The amended quotation **starts again at Inquiry** and goes through the normal flow, including approval and sending.

> **Important:** The earlier payment link is **not** cancelled automatically. If a customer pays through the old link after you cancel, the payment is **not recorded** on the cancelled quotation and an alert is sent to Slack. Tell the customer to use the link in the new email, and check Shopify if they have already paid.

Quotations that have not yet been sent (before **Sent to Customer**) have no Cancel button; edit them directly.

There is no automatic revision numbering or “revise 3 times” limit in normal use.

[IMAGE PLACEHOLDER: Cancel and Amend buttons]

---

## 17. Automatic Follow-Ups

The system checks every **Sent to Customer** quotation each weekday at **2 PM Eastern Time** and creates a follow-up **Task** for CS at set business days (Mon–Fri) after sending.

| Business days after sending | What happens |
|---|---|
| Day 3 | A follow-up Task is created |
| Day 10 | A second Task (asks about questions or concerns) |
| Day 25 | An urgency Task (“expires soon”) |
| Day 30 | A final Task is created **and** the quotation is moved to **Lost** in the same run |

### What CS does
1. Open the **Task** (project “CS Ticket”, subject “Follow-up review – <quotation> (Day N)”). It is assigned to the support mailbox.
2. Click **Compose Follow-up Email**. The recipient, subject and body are pre-filled.
3. Review, edit if needed, and send. Sending marks that follow-up as done and closes the Task.

> **Notes:** The system **never sends the follow-up automatically**. The Day 25 message says the quotation expires in about 5 business days, but because Valid Till is usually 30 calendar days, the quotation may already have expired by then. Because the Day 30 quotation is marked Lost immediately, you will normally not get a chance to send the Day 30 email. If the contact has no email, no Task email is prepared.

[IMAGE PLACEHOLDER: Follow-up Task with Compose Follow-up Email button]

---

## 18. Expiry and Lost

| Scenario | What happens |
|---|---|
| Valid Till date passes while **Sent to Customer** | Automatically marked **Expired** (checked daily at 2 PM ET, before follow-ups). |
| 30 business days after sending with no payment | Automatically marked **Lost**. |
| Customer declines | CS clicks **Lost** (from Sent to Customer or Partially Paid). |
| Quotation needs to be withdrawn after sending | CS clicks **Cancel**. |
| Super Admin | Can manually move Sent to Customer to **Expired**. |

Quotations in **Bank Transfer Selected** or **Partially Paid** are **not** auto-expired.

When a quotation is Lost or Cancelled, payment holds on related orders are released.

---

## 19. Won and What Happens Next

A quotation becomes **Won** when the **total received is at least the grand total**.

**The Lyfe Order is not created by the Won status itself.** It is created when the matching Shopify order is imported by the shipping system sync. The Lyfe Order links back to the quotation, and the Quotation form shows **Lyfe Order(s)** links when it exists. If the payment was confirmed without a Shopify order, or the Shopify order failed, no Lyfe Order appears — check with the administrator.

Priority and Order Type carry over to the Lyfe Order. The Order Drawing is copied with it.

---

## 20. Dashboards

- **Factory Quotation Dashboard** — for Factory (see section 11).
- **Quotation Analysis Dashboard** — for System Manager, Sales Manager and Sales User. It shows stage counts, funnel, win rate, aging, bottlenecks, SLA violations, follow-up engagement, payment tracking, category breakdown, items needing attention and month-on-month trends.
  Won counts **Won** and **Payment Received**; Lost counts **Lost** and **Expired**. Test customers are excluded.

[IMAGE PLACEHOLDER: Quotation Analysis Dashboard]

---

## 21. Common Errors and How to Fix Them

| Message | Why | Fix |
|---|---|---|
| “Customer email is not set. Please link a Contact with an email address.” | The contact has no email. | Add an email to the Contact, reload the quotation. |
| “Customer address is not set. Please add a Shipping Address or Customer Address.” | No address on the quotation. | Add an address. |
| “This quotation requires approval before it can be sent to the client (grand total ≥ ₹10,000 or a discount/margin violation is present). Please use Send for Approval first.” | Approval is required. | Click **Send for Approval**. |
| “Production Cost details are missing for the following item(s) … Please click Actions → Assign To Factory …” | Cost of Goods is missing. | Use the **Assign To Factory** workflow button (shown as a state action at the top, not under Actions) so Factory fills it in. Urgent quotations skip this check. |
| “Cost of Goods is missing for the following item(s) … Please fill in Cost of Goods for all items before proceeding.” | Factory tried to complete without Cost of Goods. | Enter Cost of Goods on each listed row. |
| “Details missing from Row … Factory.” | Same, shown in the form. | Fill in the listed rows. |
| “Shipping Charges must be added before sending back to Customer.” / “Shipping Charges must be added before proceeding.” | Factory tried Feasibility Approve with 0 shipping. | Enter Shipping Charges or tick **Free Shipping**. |
| “Drawing Required is enabled but no Order Drawing is provided.” | Engineer tried Completed with no drawing. | Attach the Order Drawing. |
| “Please log a Timesheet for this Quotation before uploading a drawing.” | Drawing upload needs a timesheet. | Log a Timesheet, then upload. |
| “Drawing Change Details is mandatory …” / “cannot be empty.” | Drawing change reason missing. | Enter what changed. |
| “Not Feasible Reason is mandatory …” | Factory tried Not Feasible without a reason. | Enter a reason. |
| “Rejected Reason is mandatory when marking this quotation as Inquiry Rejected.” | No rejection reason. | Enter a reason. |
| “A Reason Code is required to request Senior Review.” / “A short note is required …” | Senior Review form incomplete. | Fill in both. |
| “Senior Review can only be requested from the Pending Engineer Review state.” | Wrong state. | Move the quotation to Pending Engineer Review first. |
| “Payment Schedule row N has a zero percentage…” | A milestone has 0%. | Enter a portion above 0, or delete the row. |
| “Payment Schedule row N has no trigger event…” | No stage checkbox ticked. | Tick one stage on the row. |
| “Payment Schedule amounts sum to … but the order total is …” | Milestones do not add up. | Adjust amounts (check rounding such as 33/33/34). |
| “Invoice Portions sum to X%. They must total exactly 100%.” | Portions do not total 100. | Correct the portions. |
| “Amount entered … exceeds the remaining balance …” | Overpayment entered. | Correct the amount. |
| “Received Amount must be greater than zero.” | Zero amount. | Enter a positive amount. |
| “Cannot mark as Won — no payment has been recorded …” | No payment recorded. | Record a payment first. |
| “This quotation is already Won.” | Duplicate payment attempt. | Nothing to do. |
| “No default outgoing email account is configured …” | Email not set up. | Ask the administrator. |
| “Failed to generate payment link.” / “Shopify API error …” | Payment link could not be created. | Try again; if it persists, contact the administrator. |
| “A payment link has already been generated for this quotation. Please reload the form …” | Link exists. | Reload the form. |
| “You do not have permission to edit Cost of Goods.” | A non-Factory user edited Cost of Goods. | Leave it to Factory. **[CONFIRM WITH TEAM: whether CS sees this each time an item is added]** |
| “Payment required before <stage> …” / “This order cannot proceed — payment is pending …” | The order is waiting for payment per the Payment Schedule. | Record the payment received, or follow up with the customer. |

---

## 22. Quick Reference — Workflow Diagram

```
New Quotation → Inquiry
                  ├─[Inquiry Rejected + reason]──► Inquiry Rejected (end)
                  └─[Inquiry Approved]──► Inquiry Approved
                         │
        ┌────────────────┼───────────────────────────────┐
        │                │                               │
  Drawing needed   Feasibility test needed         Ready to price
        │                │                               │
        ▼                ▼                               │
 Pending Engineer   Production Details Review           │
 Review ◄─► Senior       │                               │
 Review Pending     ┌────┴─────┐                         │
        │       Not Feasible  Feasibility Approve        │
        │           │          │ (≥10,000 → Pending      │
        │      CS Review       │  Approval directly)     │
        │           ▼          ▼                         │
        └──► Sales User Review ◄──────── Feasibility Approved
                    │                       │ (Required a Change → Edit Request Created
                    │                       │  → Changes Completed → back to Feasibility Approved)
        ┌───────────┴───────────┐
   Approval needed        No approval needed
        │                       │
        ▼                       │
 Pending Approval               │
   │ Approve / Send Back        │
   ▼                            │
 Approved – Pending Send        │
        └───────────┬───────────┘
                    ▼
            [Send Quotation to Client]
                    ▼
              Sent to Customer ──► Expired (Valid Till passed) / Lost / Cancel
          ┌─────────┴──────────┐
     Card payment         Bank transfer click
    (automatic)                 │
          │             Bank Transfer Selected
          │                [Confirm Payment]
          └──────────┬──────────┘
                     ▼
        paid in full → Won      part paid → Partially Paid
                                       │ [Confirm Remaining Payment]
                                       └──► … until fully paid → Won
```

[IMAGE PLACEHOLDER: Workflow diagram screenshot from the system, if available]

### Which button for which state

| State | Button | Who |
|---|---|---|
| Draft | New Inquiry | CS / Engineer |
| Inquiry | Inquiry Approved / Inquiry Rejected | CS |
| Inquiry Approved | Request for Technical Details / Assign To Factory / Send for Approval / Send Quotation to Client | CS |
| Pending Engineer Review | Completed / Need More Information / Request Senior Review | Engineer |
| Senior Review Pending | Senior Review Approve / Senior Review Return | Senior Engineer |
| Production Details Review | Feasibility Approve / Not Feasible | Factory |
| Not Feasible | CS Review | CS |
| Feasibility Approved | Send for Approval / Send Quotation to Client / Required a Change | CS |
| Edit Request Created | Changes Completed | Engineer / Factory |
| Sales User Review | Request for Technical Details / Assign To Factory / Send for Approval / Send Quotation to Client | CS |
| Pending Approval | Approve / Send Back to Sales User / Send Quotation to Client | Manager |
| Approved – Pending Send | Send Quotation to Client | CS |
| Sent to Customer | Bank Transfer Selected / Lost / Cancel (Expired: Super Admin) | CS |
| Bank Transfer Selected | Confirm Payment / Cancel | CS |
| Partially Paid | Confirm Remaining Payment / Lost / Cancel | CS |
| Payment Received (older) | Mark as Won / Cancel | CS |
| Won / Expired / Lost | Cancel | CS |

> Buttons only appear when their conditions are met (for example, **Send Quotation to Client** is hidden while approval is outstanding).

---

## 23. Frequently Asked Questions

**Q: Why does my new quotation say “Inquiry” and not “Draft”?**
Every new quotation starts in Inquiry. CS must click **Inquiry Approved** to continue.

**Q: I cannot see any workflow buttons.**
Your role may have no action in that state. Sales User and Sales Manager cannot click workflow actions unless they also have Customer Service.

**Q: Why can’t I see Cost of Goods?**
Only Factory and Super Admin see it.

**Q: Can I edit a quotation after it is sent?**
No. Cancel it and Amend it (section 16). The amended copy restarts at Inquiry.

**Q: The customer paid online, but the status is not Won.**
Wait 5 minutes and refresh. If it is **Partially Paid**, the amount received is less than the grand total — check the Payment History. If nothing changes after 10 minutes, contact the administrator.

**Q: The customer paid part of the amount.**
Record it with **Confirm Payment** (first payment) or **Confirm Remaining Payment** (later payments). The quotation becomes Won when the balance reaches zero.

**Q: Can I delete or edit a payment row?**
No. Rows are kept for audit. Ask the administrator.

**Q: What goes in Reference / Note?**
The bank UTR or transaction ID, cheque number or any note useful for bank reconciliation.

**Q: The dialog shows the wrong Already Received amount.**
It uses the value saved when you opened the form. Refresh before clicking Confirm Payment.

**Q: I approved a quotation in Slack but the customer did not get an email.**
Slack only changes the status. Open the quotation and click **Resend Email**. **[CONFIRM WITH TEAM: preferred process for Slack approvals]**

**Q: Does the Get Shipping rate change the total?**
No. Copy the amount into the **Shipping Charges** row yourself.

**Q: The Lyfe Order is missing after Won.**
It is created when the Shopify order syncs from the shipping system. Contact the administrator if it does not appear.

---

*Last updated: October 2026 • Lyfe Hardware Internal Documentation*
