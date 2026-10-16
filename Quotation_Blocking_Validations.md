# Quotation — Places Where the User Is Blocked

Scope: every place in the LH app's Quotation customisation that stops a user from saving or moving a Quotation forward.
Sources: `lh/lyfe_hardware/doc_events/quotation.py`, `lh/public/js/quotation.js`, `lh/lyfe_hardware/doc_events/drawing_upload.py`, `lh/lyfe_hardware/doc_events/senior_review.py`, and the Lyfe Order payment gates that read the Quotation's payment schedule.
Last reviewed: 2026-10-16.

**Legend** — Where: **Server** = Python hook (also covers API/bulk/Admin), **JS** = browser only.
"Hard block" = save/action is refused. "Soft" = warning only, user can continue.

---

## A. Save (validate) blocks

| # | Message on screen (title) | Why we block | Where | Trigger |
|---|---|---|---|---|
| A1 | "Payment Schedule row N has a zero percentage. Every milestone must have a non-zero Invoice Portion." (*Invalid Payment Schedule*) | A milestone with 0% can never be collected, so it is meaningless | Server `_validate_payment_schedule` | Any save with a Payment Schedule row at 0% |
| A2 | "Payment Schedule row N has no trigger event (no stage checkbox is checked). Every milestone must be tied to a workflow stage." (*Invalid Payment Schedule*) | The payment gate decides when to ask for money from the stage checkboxes. No checkbox means the milestone never fires | Server `_validate_payment_schedule` | Row with none of Advance / Photo Approval / Before Dispatch / Before Shipped / On Delivery ticked |
| A3 | "Payment Schedule amounts sum to X but the order total is Y. Please adjust the milestone amounts… (check rounding on splits like 33/33/34)." (*Invalid Payment Schedule*) | Milestones must add up to the grand total exactly, otherwise money is left uncollected or over-collected (tolerance 0.01) | Server `_validate_payment_schedule` | Save when grand total > 0 and milestone sum differs |
| A4 | "Payment Schedule: Invoice Portions sum to N%. They must total exactly 100%." | Same reason as A3, caught earlier in the browser | JS `validate` → `_lh_validate_invoice_portion_total` | Any save where portions ≠ 100% (±0.01) |
| A5 | "Invoice Portion cannot be negative." / "Invoice Portion cannot exceed 100%." | Invalid percentage. The field is reset to 0 | JS `invoice_portion` field handler | Typing a value < 0 or > 100 in a Payment Schedule row |
| A6 | "Drawing Change Details is mandatory when attaching or replacing the Order Drawing." (*Change Details Required*) | We need an audit note whenever the drawing changes. Engineers and the factory rely on it | Server `validate` | Save that changes Order Drawing without going through the popup (API, bulk edit). Browser uploads go via the popup instead (see E1) |
| A7 | "Order Type must be set before saving a {priority} priority quotation." (*Order Type Required*) | LH-DEV-006: High/Urgent/VIP orders need a confirmed order type for routing | Server `validate` | Priority not Normal and Order Type not Standard/Custom/Export. Order Type is auto-set earlier in the same function, so this should rarely fire |
| A8 | "This quotation has already been revised 3 times (Rev N). Please escalate to a manager before making further changes." (*Revision Limit Reached*) | Stops endless re-quoting of a quote that was already sent to the customer | Server `_handle_revision` | Item changes while state = *Sent to Customer* and revision number ≥ 3 |

## B. Workflow action blocks (before the state changes)

Checked in the browser first (`before_workflow_action` in `quotation.js`, shown as a red "Validation Error" popup). The same rule is repeated on the server, so API/bulk/Admin paths cannot bypass it.

| # | Message on screen (title) | Why we block | Action / state | Where |
|---|---|---|---|---|
| B1 | "Drawing Required is enabled but no Order Drawing is attached. Please add a drawing before completing." (*Drawings Required*) | Production needs the drawing. Required if the Drawing Required box or any item's Photo Required is set | **Completed** from *Pending Engineer Review*. Skipped when priority = Urgent | JS + Server (`before_workflow_action` and `on_update`) |
| B2 | "Cost of Goods is missing for the following item(s): …" (*Cost of Goods Incomplete*). Browser version: "Details missing from Row N Factory…" for Factory users, or "This quotation cannot be completed yet — factory still needs to update the item details." for others | Margin and approval checks need COGS on every line | **Completed** / **Feasibility Approve** from *Production Details Review*. *Not Feasible* is exempt | JS + Server (also re-checked in `on_update` when leaving that state) |
| B3 | "Shipping Charges must be added before sending back to Customer." / "Shipping Charges must be added before proceeding." (*Shipping Charges Required*) | The customer must not be quoted without freight | Leaving *Production Details Review* | JS + Server. **Not enforced if "Free Shipping" is ticked** |
| B4 | "This quotation requires approval before it can be sent to the client (grand total ≥ ₹10,000 or a discount/margin violation is present). Please use **Send for Approval** first." (*Approval Required*) | Large or discounted quotes need a manager's sign-off | **Send Quotation to Client** while Approval Required is set | Server `before_workflow_action` and `on_update` |
| B5 | "Production Cost details are missing for the following item(s): … Please click **Actions → Assign To Factory** so the factory team can fill in the Cost of Goods before this quotation is sent for approval." (*Production Cost Required*) | The approver needs COGS to judge margin | **Send for Approval** (to *Pending Approval*) from Draft / Sales User Review / Feasibility Approved. Skipped when priority = Urgent | Server `before_workflow_action` and `on_update` |
| B6 | "Customer email is not set. Please link a Contact with an email address." (*Customer Details Incomplete*) | The quote is emailed to the customer | **Send Quotation / Send Quotation to Client** | JS + Server (`on_update` on entering *Sent to Customer*) |
| B7 | "Customer address is not set. Please add a Shipping Address or Customer Address." (*Customer Details Incomplete*) | An address is needed for tax and shipping | **Send for Approval / Send Quotation to Client** | JS + Server (`on_update`) |
| B8 | "Not Feasible Reason is mandatory when marking this quotation as Not Feasible." (*Reason Required*). Browser dialog: "Please enter a reason before proceeding." | Keeps a record of why the order was dropped | **Not Feasible** | JS dialog + Server `on_update` |
| B9 | "Rejected Reason is mandatory when marking this quotation as Inquiry Rejected." (*Reason Required*). Browser dialog: "Please enter a reason before proceeding." | Same, for rejected inquiries | **Inquiry Rejected** | JS dialog + Server `on_update` |
| B10 | "Cannot mark as Won — no payment has been recorded for this quotation. Please record a payment first." (*Payment Required*) | Won means paid. With zero payment recorded we refuse rather than guess. If part-paid, the state is auto-corrected to *Partially Paid* instead of blocking | Setting state to **Won** | Server `_enforce_payment_state` |
| B11 | "A Reason Code is required to request Senior Review." / "A short note is required to request Senior Review." / "Senior Review can only be requested from the {Engineer Check} state." | Senior Review needs a reason and is only valid from the engineer-check state | **Request Senior Review** | Server `senior_review.py` (+ JS dialog with mandatory fields) |
| B12 | "Invalid Review Outcome." / "This Senior Review has already been completed." | Prevents bad or duplicate outcomes | **Complete Senior Review** | Server `senior_review.py` |

## C. Email / sending blocks

| # | Message on screen | Why we block | Where |
|---|---|---|---|
| C1 | "No contact email on the quotation. Please add a contact before sending." | There is nobody to send to | Server `send_quotation_email` |
| C2 | "No default outgoing email account is configured. Please set a default outgoing Email Account before sending quotations." (*Email Not Sent*, orange) | The email cannot go out. The workflow does **not** advance | Server `send_quotation_email` |
| C3 | "Failed to send quotation email. Please try again." | The email send failed. The workflow does not advance | JS (Send Quotation and Resend Email) |
| C4 | "Customer email is not set. Please link a Contact with an email address." | Resend Email button needs a contact email | JS Resend Email |

## D. Payment gates — block the downstream Lyfe Order, driven by the Quotation's Payment Schedule

The Quotation itself saves fine. The block hits the **Lyfe Order** when the cumulative payment received on the Quotation is less than the cumulative threshold for that stage (`_enforce_stage_payment`). Nothing is blocked if no milestone is ticked for that stage.

| # | Lyfe Order step blocked | Quotation stage checkbox that decides it |
|---|---|---|
| D1 | Assigning the order to Factory / US Warehouse (also via warehouse change to Factory) | Advance Amount |
| D2 | Marking the order **Approved** | Stage: Photo Approval |
| D3 | Marking **Ready for Dispatch** | Before Dispatch |

- **Message — Customer Service / System Manager:** "Payment required before *{step}*. Quotation: … Required (cumulative): … Received so far: …" (*Payment Incomplete*)
- **Message — everyone else:** "This order cannot proceed — payment is pending. The CS team has been notified to follow up on the payment status."
- **Why:** do not release production, dispatch, etc. before the customer has paid the agreed milestone.
- **Side effect:** the first block stamps the block stage/date on the Lyfe Order and queues the payment-reminder email/Slack to CS (reminders in `tasks/payment_reminders.py`).
- **Not blockable:** *Before Shipped* and *On Delivery* milestones have no hard gate. Setting On Delivery only shows a warning, because nothing comes after *Completed* (see F4).

## E. File / drawing blocks

| # | Message on screen | Why we block | Where |
|---|---|---|---|
| E1 | Popup "Drawing Change Details" ("What changed on the Order Drawing?" — mandatory). It cannot be closed with X/Esc. Only **Save** or **Cancel Upload** | Every drawing change needs a note. If the popup is abandoned, the upload is auto-reverted after the grace period (sweep every 15 min, `drawing_change_grace_sweep.py`) | JS + Server |
| E2 | "Drawing Change Details cannot be empty." | Empty note not accepted | Server `submit_drawing_change_details` |
| E3 | "Please log a Timesheet for this Quotation before uploading a drawing." | Drawing work must be time-tracked. Blocks the upload before the file is created | Server `drawing_upload.file_before_insert` / `check_drawing_timesheet` (sidebar "Is it a drawing attachment?") |

## F. Warnings only (do NOT block)

| # | Message | Where |
|---|---|---|
| F1 | "Amount entered (X) exceeds the remaining balance (Y). Please verify." (*Overpayment Warning*) — **actually returns and stops the Record Payment dialog until the amount is fixed**, so it behaves as a block on that dialog | JS Confirm Payment dialog |
| F2 | Invoice Portion total ≠ 100% while typing: orange alert only. The hard block is A4 at save | JS |
| F3 | "The drawing/specification changed after approval. The previous approval has been cleared and this quotation was routed back to Engineer Check." (*Approval Invalidated*) — approval is revoked and the quote goes back for re-check, not refused | Server `senior_review.on_update` |
| F4 | "On Delivery — Collection Risk": collection cannot be enforced, so only an email and Slack message fire at Completed | JS Payment Schedule row |
| F5 | Shopify confirmation failed after payment recorded (orange) | JS |
| F6 | "Please save the Quotation before fetching the shipping rate." / "Please set the Order Type…" / "No rates available to apply." / "Please enter Length, Width and Height for at least one item." / "Please enter the required delivery date." / "Please select the Order Type." | JS shipping-rate / box-size / delivery-date calculator dialogs. Block only that tool, not the Quotation |
| F7 | COGS Benchmark deviation check | Never blocks (explicitly designed not to throw) |

## G. Visibility restriction (not a message)

- **Senior Engineer** only sees Quotations assigned to them that are in *Senior Review Pending* or *Pending Engineer Review* (list permission query). Everything else is hidden. System Manager and Super Admin are unrestricted.
- **Not Feasible Reason** can be edited only by Factory, System Manager, Super Admin. Edits by others are silently reverted on save.

## H. Disabled (code exists but commented out — NOT blocking today)

- Shipping Charges required before **Send for Approval / Send Quotation to Client** (JS and Server, "Shipping Charges are required before sending the quotation to the client").
- "Custom orders cannot be sent directly to client from Draft" (JS).

## Notes / observations

1. Most workflow gates exist twice (JS for a friendly instant message, Server for enforcement). Where the two disagree, the Server wins. Example: JS checks `shipping_address`, Server checks `shipping_address_name` / `customer_address` for B7.
2. Priority = **Urgent** skips B1 (drawing) and B5 (COGS before approval) but **not** B2/B3 (leaving Production Details Review).
3. Approval is required when grand total ≥ ₹10,000 or an item breaks the discount/margin rule. It is cleared once the state has reached *Approved – Pending Send* or later.

---

# Part 2 — Founder Bypass Analysis (analysis only, nothing changed in code)

**Goal:** find which blocks from Part 1 can be skipped for the **Founder** role, while keeping mandatory everything that must exist before the quotation is sent to the client, because the payment link is generated at that moment.

Today no Quotation check treats "Founder" specially. Founder only appears in dashboard/MCP access lists. Any bypass would be a new role check next to each rule (`"Founder" in frappe.get_roles()`), in both the JS and Python copies of the rule.

## 1. What the payment link actually needs

When *Send Quotation / Send Quotation to Client* is clicked (`quotation.js` → `shopify.create_draft_order` → `send_quotation_email`), the link is built from the data below. If anything marked "needed" is missing, the link is wrong or the send fails.

| Data used | Used for | Needed? |
|---|---|---|
| Items: item name, SKU, qty, rate, discounts | Draft order line items and price | **Needed** (always present on a quotation) |
| Shipping Address (falls back to Customer Address) | Shopify draft order shipping/billing address. Also the **only input to Shopify Sales Tax** in `shopify_tax.set_sales_tax`. With no address the tax row is silently removed, so the link total would not match the quotation | **Needed** |
| Contact Email | Shopify sends its invoice to it, and our branded email (`send_quotation_email`) throws without it | **Needed** |
| "Shipping Charges" tax row | Sent as the Shopify shipping line. A 0 amount is accepted by Shopify | Not required for the link to work, but a forgotten value makes the customer's total wrong |
| Custom Fee, other tax rows, order discount | Extra line items / discount | Optional, only if present |
| Shopify Tax Settings → Payment Link enabled | Link generation is refused otherwise | Needed (configuration, not a per-quote condition) |
| No existing draft order id | A second link for the same quote is refused ("Payment Link Already Generated") | Needed (the quote must not already have a link) |
| Payment Schedule, COGS, drawing, approval flag, revision number | **Not read** by the link builder | Not needed |

## 2. Verdict for each blocking condition

| Part 1 ref | Condition | Needed for payment link? | Founder verdict |
|---|---|---|---|
| B6 / C1 / C4 | Customer email | **Yes**: Shopify invoice and our email both need it | **Keep mandatory for Founder** |
| B7 | Customer / shipping address | **Yes**: Shopify tax and draft-order addresses need it | **Keep mandatory for Founder** |
| C2 | Default outgoing email account configured | Yes (email cannot be sent) | **Keep**. It is a system setup issue, not a rule to bypass |
| (Shopify) | Link already generated / link disabled / Shopify API error | Yes | **Keep**. Technical, cannot be bypassed |
| B4 | Approval required before *Send Quotation to Client* (≥ ₹10,000 or discount/margin breach) | No | **Bypass candidate**. The Founder is the authority this approval exists for. Skipping *Send for Approval* and sending directly is safe for the link. Leave an audit comment saying "approval waived by Founder" |
| B5 | COGS missing before *Send for Approval* | No | **Bypass candidate**. The link doesn't use COGS. Risk: nobody sees the margin. The `cogs_pending` flag stays set so the gap remains visible |
| B2 | COGS missing before leaving *Production Details Review* | No | **Bypass candidate**, same risk as B5. The COGS checks in B2 and B5 should be waived together, otherwise Founder is stopped at the earlier step |
| B3 | Shipping Charges > 0 before leaving *Production Details Review* (unless Free Shipping ticked) | Not required, but affects the customer's total | **Conditional bypass**. Allow it only if the Founder confirms a popup like "No shipping charge will be added to the payment link. Continue?" Free Shipping already covers the intended case |
| B1 | Order Drawing missing before *Completed* | No | **Possible bypass, but not recommended**. The link doesn't need it, but production needs the drawing later. Safer: let the Founder skip it for now and keep the Drawing Required flag on so the factory side still sees it |
| A8 | Revision limit (Rev 3) | No (but each revision re-issues the Shopify draft order) | **Bypass candidate**. This rule literally says "escalate to a manager". The Founder is that escalation. Record the reason in a comment |
| A7 | Order Type for priority quotes | No | Not worth bypassing. It is auto-set and rarely fires |
| A1, A2 | Payment Schedule row 0% / no stage checkbox | Link doesn't use the schedule | **Do not bypass**. A bad row makes the later payment gates (Lyfe Order) misbehave or never fire |
| A3, A4 | Milestone amounts / portions ≠ total / 100% | Link uses grand total, not the schedule | **Do not bypass**. Same reason, and it would leave money uncollected or over-collected |
| A5 | Invoice portion < 0 or > 100 | No | Do not bypass (invalid data) |
| A6, E1–E3 | Drawing change details / timesheet before drawing upload | No | Do not bypass. These are audit and time-tracking rules, unrelated to sending |
| B8, B9 | Not Feasible / Inquiry Rejected reason | No | Do not bypass. Audit trail |
| B10 | Won with no payment | No | **Do not bypass**. Would mark an unpaid order as paid |
| B11, B12 | Senior Review rules | No | Do not bypass (process integrity) |
| D1–D3 | Lyfe Order payment gates (Factory assign, Approved, Ready for Dispatch) | No (they run after the quote is sent and paid) | **Do not bypass**. Releasing production or dispatch without the agreed payment is the loss these gates prevent. If ever needed, treat it as a separate decision per gate |

## 3. Summary

**Can be bypassed for Founder (not needed for the payment link):**
1. B4 — approval required before sending (recommended)
2. B5 + B2 — COGS missing (recommended together; margin visibility is lost)
3. A8 — revision limit (recommended)
4. B3 — shipping charge missing (with confirmation popup)
5. B1 — drawing missing (possible, not recommended)

**Must stay mandatory before sending to the client (needed by the payment link):**
- Customer email (B6)
- Customer / shipping address (B7), because Shopify tax depends on it
- Outgoing email account (C2) and Shopify link configuration
- No existing payment link on the quote

**Never bypass (even for Founder):** Payment Schedule rules (A1–A5), "Won" without payment (B10), Lyfe Order payment gates (D1–D3), and audit reasons (B8, B9, A6).

## 4. Things to check before implementing

1. **Role needs workflow permission.** A bypass only helps if Founder already has permission on the workflow transitions (*Send for Approval*, *Send Quotation to Client*, *Completed*). That has to be checked in the Quotation Workflow, because a role check in code cannot override a workflow transition permission.
2. **Fix both layers.** Each rule exists in the browser (`before_workflow_action` in `quotation.js`) and on the server (`before_workflow_action` + `on_update` in `quotation.py`). A bypass done in only one place still blocks Founder.
3. **Email and address must be enforced on the new path.** Today the server checks email/address only when the state becomes *Sent to Customer*. If Founder skips approval and sends directly, that same check still applies, so no new code is needed. Just don't add Founder to those checks.
4. **Audit.** Add a Comment on the Quotation every time a Founder bypass is used (rule name, who, when), so the waived check is traceable.
5. **Approval flag.** After a Founder sends without approval, `approval_required` stays set until the state reaches an approved state. Confirm reports and the Slack approval card don't show this as "pending approval" afterwards.
