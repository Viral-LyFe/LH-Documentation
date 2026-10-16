# Quotation — Where the System Stops the User

This document lists every place where the Quotation stops a user from saving it or moving it to the next step. For each one it gives the message the user sees and the reason the check exists.

Last reviewed: 2026-10-16

**How to read the "Checked" column**
- **Screen** = the check runs in the user's browser, so the message appears instantly.
- **System** = the check runs in the system itself, so it also applies to imports, bulk actions and Admin.
- **Both** = the browser shows the message early and the system checks again. This is the usual case.

**Where the rules live:** all rules in Part 1 are in the LH app, except the two items in the box below.

> **Not part of the LH app**
> 1. **Sales tax calculation** is done by a separate app called **Shopify Tax** (`shopify_tax`). LH only builds the payment link. The tax needs a shipping address, which is why the address check matters.
> 2. **Workflow setup** (which steps exist and which role can click which button) is configured in the site's Workflow, not in LH code. It has not been reviewed here.

---

# Part 1 — Where users are blocked

## A. When saving the Quotation

| # | Message the user sees | Why we block | Checked |
|---|---|---|---|
| A1 | "Payment Schedule row N has a zero percentage. Every milestone must have a non-zero Invoice Portion." | A milestone worth 0% can never be collected. | System |
| A2 | "Payment Schedule row N has no trigger event… Every milestone must be tied to a workflow stage." | The system decides when to ask for payment from the stage ticked on each row. With nothing ticked, the payment is never requested. | System |
| A3 | "Payment Schedule amounts sum to X but the order total is Y…" | The milestones must add up exactly to the order total, otherwise money is left uncollected or over-collected. | System |
| A4 | "Payment Schedule: Invoice Portions sum to N%. They must total exactly 100%." | Same as A3, caught earlier on screen. | Screen |
| A5 | "Invoice Portion cannot be negative." / "cannot exceed 100%." | Not a valid percentage. The value is reset to 0. | Screen |
| A6 | "Drawing Change Details is mandatory when attaching or replacing the Order Drawing." | Every drawing change needs a note so engineers and the factory know what changed. | System |
| A7 | "Order Type must be set before saving a {priority} priority quotation." | High, Urgent and VIP orders need an order type for routing. In practice it is set automatically, so this rarely appears. | System |
| A8 | "This quotation has already been revised 3 times… Please escalate to a manager before making further changes." | Stops endless re-quoting after the quote has been sent to the customer. | System |

## B. When moving the Quotation to the next step

| # | Message the user sees | Why we block | When | Checked |
|---|---|---|---|---|
| B1 | "Drawing Required is enabled but no Order Drawing is attached. Please add a drawing before completing." | Production needs the drawing. | *Completed* from Pending Engineer Review. Not applied to Urgent orders. | Both |
| B2 | "Cost of Goods is missing for the following item(s)…" (Factory users see "Details missing from Row N…"; others see "This quotation cannot be completed yet — factory still needs to update the item details.") | Pricing and approval need the cost of every item. | *Completed* or *Feasibility Approve* from Production Details Review. *Not Feasible* is exempt. | Both |
| B3 | "Shipping Charges must be added before proceeding." | The customer must not be quoted without freight. Not applied if Free Shipping is ticked. | Leaving Production Details Review | Both |
| B4 | "This quotation requires approval before it can be sent to the client (grand total ≥ ₹10,000 or a discount/margin violation is present). Please use Send for Approval first." | Large or discounted quotes need a manager's sign-off. | *Send Quotation to Client* | System |
| B5 | "Production Cost details are missing for the following item(s)… Please click Actions → Assign To Factory…" | The approver needs the cost to judge the margin. | *Send for Approval*. Not applied to Urgent orders. | System |
| B6 | "Customer email is not set. Please link a Contact with an email address." | The quotation is emailed to the customer. | Sending to the customer | Both |
| B7 | "Customer address is not set. Please add a Shipping Address or Customer Address." | An address is needed for tax and shipping. | Sending for approval or to the customer | Both |
| B8 | "Not Feasible Reason is mandatory…" / "Please enter a reason before proceeding." | Keeps a record of why the order was dropped. | *Not Feasible* | Both |
| B9 | "Rejected Reason is mandatory when marking this quotation as Inquiry Rejected." | Same, for rejected inquiries. | *Inquiry Rejected* | Both |
| B10 | "Cannot mark as Won — no payment has been recorded for this quotation. Please record a payment first." | Won means paid. If only part is paid, the system moves it to *Partially Paid* instead of blocking. | Marking as Won | System |
| B11 | "A Reason Code is required to request Senior Review." / "A short note is required…" / "Senior Review can only be requested from the Engineer Check state." | Senior Review needs a reason and is only valid at the engineer-check step. | *Request Senior Review* | Both |
| B12 | "Invalid Review Outcome." / "This Senior Review has already been completed." | Prevents wrong or duplicate outcomes. | *Complete Senior Review* | System |

## C. Sending the email

| # | Message the user sees | Why we block |
|---|---|---|
| C1 | "No contact email on the quotation. Please add a contact before sending." | There is nobody to send to. |
| C2 | "No default outgoing email account is configured…" | The email cannot go out, so the quotation does not move forward. |
| C3 | "Failed to send quotation email. Please try again." | The send failed, so the quotation does not move forward. |
| C4 | "Customer email is not set…" (Resend Email button) | Resend needs a contact email. |

## D. Payment checks that stop the Lyfe Order

The Quotation saves normally. The block appears later on the **Lyfe Order** when the customer has not yet paid the amount agreed for that step in the Payment Schedule.

| # | Lyfe Order step that is blocked | Payment Schedule stage that decides it |
|---|---|---|
| D1 | Assigning the order to Factory / US Warehouse | Advance Amount |
| D2 | Marking the order Approved | Photo Approval |
| D3 | Marking the order Ready for Dispatch | Before Dispatch |

- **Customer Service and System Manager** see: "Payment required before {step}. Quotation… Required (cumulative)… Received so far…"
- **Everyone else** sees: "This order cannot proceed — payment is pending. The CS team has been notified to follow up on the payment status."
- **Why:** production and dispatch should not start before the agreed payment is received.
- The first block also records the date and triggers payment reminders to CS.
- Nothing is blocked if no payment stage is ticked in the schedule. The *Before Shipped* and *On Delivery* stages have no hard stop.

## E. Drawings

| # | What happens | Why |
|---|---|---|
| E1 | After uploading a drawing, a popup "Drawing Change Details" appears and cannot be closed. The user must **Save** a note or **Cancel Upload**. If it is abandoned, the upload is removed automatically after a short wait. | Every drawing change needs an explanation. |
| E2 | "Drawing Change Details cannot be empty." | An empty note is not accepted. |
| E3 | "Please log a Timesheet for this Quotation before uploading a drawing." | Drawing work must be time-tracked. The upload is stopped before the file is saved. |

## F. Warnings and special cases (these do not stop the Quotation)

| # | What the user sees | Notes |
|---|---|---|
| F1 | "Amount entered (X) exceeds the remaining balance (Y). Please verify." | Shown in the Record Payment dialog. The user must correct the amount before continuing, so it acts as a block for that dialog. |
| F2 | Orange alert while typing if portions do not total 100% | A warning only. The real block is A4 on save. |
| F3 | "The drawing/specification changed after approval. The previous approval has been cleared and this quotation was routed back to Engineer Check." | The approval is cancelled and the quote goes back for re-check. |
| F4 | "On Delivery — Collection Risk" | Reminds CS that payment on delivery cannot be enforced by the system. |
| F5 | "Shopify confirmation failed" | The payment was recorded. Only the Shopify confirmation needs a retry. |
| F6 | Messages in the shipping-rate, box-size and delivery-date tools, such as "Please save the Quotation before fetching the shipping rate." | These only stop the tool, not the Quotation. |
| F7 | Cost benchmark deviation check | Never blocks. It only flags. |

## G. Visibility rules

- A **Senior Engineer** sees only Quotations assigned to them that are in *Senior Review Pending* or *Pending Engineer Review*. System Manager and Super Admin see everything.
- **Not Feasible Reason** can be edited only by Factory, System Manager and Super Admin. Changes by anyone else are quietly undone.

## H. Switched off today (these do not block)

- "Shipping Charges are required before sending the quotation to the client" is written but turned off.
- "Custom orders cannot be sent directly to the client from Draft" is written but turned off.

## Observations

1. Most step checks run twice (on screen and in the system). If they disagree, the system's answer wins. Example: for the address check, the screen and the system look at slightly different address fields.
2. **Urgent** priority skips B1 and B5, but not B2 and B3.
3. Approval is required when the grand total is ₹10,000 or more, or when a discount or margin rule is broken. It stops applying once the quote reaches *Approved – Pending Send* or later.

---

# Part 2 — Which checks could be skipped for the Founder

**Purpose:** decide which blocks the Founder could skip, while keeping everything that must be in place before the quotation goes to the client, because the payment link is created at that moment.

**Current state:** nothing in the Quotation treats the Founder role differently today. A bypass would need a new Founder check next to each rule, on screen and in the system. This part is analysis only. No change has been made.

## 1. What the payment link needs

| Information | Used for | Needed? |
|---|---|---|
| Items, quantity, price, discounts | Lines and prices on the payment page | **Yes** (always present) |
| Shipping / customer address | Address on the payment page, and the **sales tax** (calculated by the separate Shopify Tax app). With no address, tax is silently left out and the link total will not match the quotation | **Yes** |
| Customer email | Shopify sends its invoice to it, and our own email needs it | **Yes** |
| Shipping charge | Added as the shipping line. A zero amount is accepted, but a forgotten value makes the customer's total wrong | Not required, but affects the total |
| Payment link switched on in Shopify Tax settings | The link cannot be created otherwise | **Yes** (setting) |
| No link already created for this quote | A second link for the same quote is refused | **Yes** |
| Payment Schedule, cost, drawing, approval, revision number | Not used by the link | No |

## 2. Verdict for each block

| Ref | Check | Needed for the link? | Founder verdict |
|---|---|---|---|
| B6, C1, C4 | Customer email | Yes | **Keep mandatory** |
| B7 | Customer / shipping address | Yes (tax and address) | **Keep mandatory** |
| C2 | Outgoing email account | Yes | **Keep.** This is a setup issue, not a rule |
| — | Link already created / link switched off / Shopify error | Yes | **Keep.** Technical, cannot be skipped |
| B4 | Approval before sending to client | No | **Can skip (recommended).** The Founder is the person this approval is meant for. Add a note on the quote: "Approval waived by Founder" |
| B5 | Cost missing before Send for Approval | No | **Can skip (recommended).** Risk: nobody sees the margin, though the "cost pending" flag stays visible |
| B2 | Cost missing before leaving Production Details Review | No | **Can skip (recommended).** Skip B2 and B5 together, otherwise the Founder is stopped at the earlier step |
| B3 | Shipping charge missing | Not required, but changes the total | **Skip only with a confirmation** such as "No shipping charge will be added to the payment link. Continue?" Free Shipping already covers the normal case |
| B1 | Drawing missing | No | **Possible but not recommended.** The link does not need it, but production does later |
| A8 | Revision limit (Rev 3) | No | **Can skip (recommended).** The rule itself says "escalate to a manager". The Founder is that escalation. Note the reason on the quote |
| A7 | Order type for priority quotes | No | Not worth changing. It is set automatically |
| A1–A5 | Payment Schedule rules | No | **Do not skip.** A wrong schedule breaks the later payment checks and can leave money uncollected |
| A6, E1–E3 | Drawing note and timesheet | No | Do not skip. Audit and time-tracking rules |
| B8, B9 | Reasons for Not Feasible / Inquiry Rejected | No | Do not skip. Audit trail |
| B10 | Won without payment | No | **Do not skip.** An unpaid order would be shown as paid |
| B11, B12 | Senior Review rules | No | Do not skip |
| D1–D3 | Payment checks on the Lyfe Order | No | **Do not skip.** They protect against starting production or dispatch without payment. Any exception should be a separate decision |

## 3. Summary

**Can be skipped for the Founder (the link does not need them):**
1. Approval before sending (B4), recommended
2. Missing cost (B5 and B2 together), recommended
3. Revision limit (A8), recommended
4. Missing shipping charge (B3), with a confirmation
5. Missing drawing (B1), possible but not recommended

**Must stay mandatory before sending to the client (the payment link needs them):**
- Customer email
- Customer / shipping address, because the tax depends on it
- Outgoing email account and the Shopify payment link setting
- No link already created for the quote

**Never skip, even for the Founder:** Payment Schedule rules, Won without payment, the Lyfe Order payment checks, and the mandatory reasons.

## 4. To confirm before building

1. **Workflow permission (outside LH):** the Founder can only use a bypass if the site's Workflow already lets the Founder role click *Send for Approval*, *Send Quotation to Client* and *Completed*. A rule change in LH cannot override the Workflow setup.
2. **Both places must change:** each rule exists on screen and in the system. If only one is changed, the Founder is still blocked.
3. **Email and address stay enforced:** these checks apply when the quote moves to *Sent to Customer*, so they still apply if the Founder skips approval. Do not add the Founder as an exception there.
4. **Keep a record:** add a comment on the Quotation each time a Founder skip is used (which rule, who, when).
5. **Approval flag:** after a Founder sends without approval, the quote still shows "approval required" until it reaches an approved step. Check that reports and the Slack approval card do not then show it as pending approval.
