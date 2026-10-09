# Quotation End User Guide — Change Log (June 2026 Google Doc → v3.0, Oct 2026)

Source of truth for every item: code in `apps/lh/lh` (`QPY` = `lyfe_hardware/doc_events/quotation.py`, `QJS` = `public/js/quotation.js`) plus the live Quotation workflow `Quotation-3` read from the local site database (the workflow is not stored in the repo). Full evidence with line numbers: audit file `quotation_audit.md` (scratchpad).

## A. Corrections (statements in the Google Doc that were wrong or are now wrong)

| # | Old doc said | Actual behaviour | Evidence |
|---|---|---|---|
| 1 | Quotation starts in **Draft** | Starts in **Inquiry**. Draft is only reached via Engineer “Need More Information”. | QPY:941-955 |
| 2 | 12 workflow states; “Pending Engineer & Factory Review”, “Pending Manager Approval” | 21 states; those names are retired. Now: Pending Engineer Review, Senior Review Pending, Production Details Review, Not Feasible, Feasibility Approved, Edit Request Created, Sales User Review, Pending Approval, etc. | Workflow `Quotation-3` |
| 3 | “Manager / Super Admin approves” | Approval role is **Quotation Approvar**. Super Admin’s only action is manual Expire. | Workflow roles |
| 4 | Shipping Charges must be > 0 before sending | Not enforced on send (check commented out). Enforced when Factory clicks Feasibility Approve (unless Free Shipping ticked). | QJS:452-458, QJS:662, QPY:1356-1363, 1401 |
| 5 | Order Type can be Standard / Custom / **Export** and CS can override | Standard/Custom is recalculated on every save; Export cannot be selected. | QPY:854-858; field options |
| 6 | “Custom orders must go through Engineer review first” error | That block is commented out; message no longer appears. | QJS:413-422 |
| 7 | Yellow banner appears when approval is required | No such banner. Only a yellow prompt for approvers in Pending Approval. | QJS:1754 |
| 8 | Manual tick to send for approval without reason | `Approval Required` is recalculated on every save; manual tick does not persist. | QPY:806-845 |
| 9 | Image Attachment field on each item row for drawings | Removed. One **Order Drawing** on the quotation, with Timesheet and change-details rules. | Quotation Item meta; drawing_upload.py |
| 10 | “Drawing Required” blocks Completed on the server | Enforced in the browser only. | QJS:490; QPY:1296 (dead) |
| 11 | Email contains a link to view/download the PDF | PDF is attached; no PDF link. Bank-transfer button always shown; card button only when payment links are enabled and not a bank-transfer quote. | quotation_email.html; QPY:1889-2068 |
| 12 | Tracking panel at the top of the form with 4 signals (incl. “PDF Viewed”) | Panel is in the **Payment & Tracking** tab, visible to CS and Super Admin; rows: Email Sent, Email Opened, Payment Link Clicked, Action Taken. | QJS:1875-1950 |
| 13 | Card payment: moves to Payment Received then Won | Goes directly to **Won** or **Partially Paid**. “Payment Received” is legacy. | shopify.py; QPY |
| 14 | Bank transfer: Confirm Payment → Payment Received → Mark as Won | Confirm Payment goes to Won or Partially Paid directly. | QJS:732-883 |
| 15 | Revising: edit sent quotation, auto Rev 1/2/3, limit of 3 (technical note) | Sent quotations are locked; revision is **Cancel → Amend**; amended copy restarts at Inquiry. No revision numbers produced. Old payment link is not cancelled. | docstatus 1; QJS:238-241; QPY:994-1045; DB shows 0 revisions |
| 16 | Follow-ups create a **ToDo** assigned to CS; draft email in timeline | Creates a **Task** (project “CS Ticket”) assigned to the support mailbox; CS sends from **Compose Follow-up Email**. | tasks/quotation_followup.py |
| 17 | Day 30 → Lost | Correct, but the Day 30 Task and Lost happen in the same run, so the final email is effectively never sent. | tasks/quotation_followup.py |
| 18 | Expired for any quote past Valid Till | Only quotations in **Sent to Customer** auto-expire. | tasks/quotation_followup.py |
| 19 | Won automatically creates a **Lyfe Order** | Lyfe Order is created by the Shopify/ShipStation order sync, not by Won. | shipstation_orders.py:533-557 |
| 20 | “Get Shipping rate” only while the quotation is in **Draft** | Available while not submitted (any pre-send state) when box dimensions or Parcel Size rows exist. | QJS:2146-2160 |
| 21 | “Parcel Size … visible to all users” | Correct (kept), but the Factory-only note in the earlier guide is no longer true. | Property Setter |
| 22 | Overpayment message shows “₹X” | Uses the quotation’s own currency. | QJS:732-883 |
| 23 | Toast “Payment confirmed! Quotation marked as Won.” on card payment | Still shown, but also appears for partial payments (known inaccuracy; documented as a caution). | QJS:197 |
| 24 | Step instructions for “Mark as Won” | Only used for legacy Payment Received quotations. | Workflow transitions |
| 25 | “Fix Missing Box Sizes (Actions menu)” | It is under **Tools**. | QJS |
| 26 | “Payment Link Sent” state in quick reference | Does not exist. | Workflow |
| 27 | “Track Payment Status” (referenced in later versions) | Button never displays due to a bug; removed from the guide. | QJS:2438 |
| 28 | “Assign to Factory under Actions” hint in error | Message wording is outdated; Assign To Factory is a workflow button. Guide explains this. | QJS:437-441 |

## B. Outdated (still partly true, now changed)

- Approval triggers: still discount / margin / total ≥ 10,000, but now with specific defaults (5%, 8% for bank transfer) and recomputed on every save.
- Creation flow: new Inquiry step, Quotation To allows Customer / Lead / Prospect / CRM Deal.
- Roles table: added Senior Engineer, Quotation Approvar; clarified Sales User/Manager have no workflow actions.
- Payment terms: default is “100 % Advance”.
- Item table: Rate hidden; Price List Rate shown; “Drawing Required ?” column; Lead Time free text.

## C. Added (functionality missing from the old doc)

1. Inquiry, Inquiry Approved, Inquiry Rejected states and reasons.
2. Senior Review flow (reason code, note, Senior Engineer, Red Flag banner, approval invalidated on spec change).
3. Order Drawing rules: Timesheet first, Drawing Change Details popup, 15-minute auto-revert, BOM page stripping.
4. Price List Rate auto-calculation from Cost of Goods (× 3.3 × finish multiplier).
5. Bank-transfer checkbox effect on discounts (8% / 0%).
6. Urgent / VIP priority, Calculate Dispatch Dates dialog and banner, and the Urgent shortcut on Cost of Goods and Order Drawing checks.
7. Slack approval card (who may act, out-of-date card, and that Slack’s “Send Quotation to Client” does not email the customer).
8. Resend Email button.
9. Payment Schedule rules, stage gating on orders, warnings, reminders.
10. Free Shipping checkbox, Shipment Contents, Update Parcel Size dialog, Box Suggestions, Fix Missing Box Sizes details.
11. US sales tax automation and Apply Additional Charges (processing fee).
12. Cancel → Amend behaviour and the old payment link risk.
13. Customer bank-transfer click behaviour (instructions email, internal notice).
14. Payment detection: webhook, 5-minute check, post-send quick check; Shopify confirmation checkbox; Shopify failure message.
15. Payment guards: Won with no payment, cancelled quotation, duplicate reference, pending bank payments ignored.
16. Factory Quotation Dashboard and Quotation Analysis Dashboard.
17. Full list of states and transitions with roles, plus the full error message table.
18. Not Feasible, Edit Request Created, Feasibility Approved paths.

## D. Accurate and kept

- COGS hidden from CS and filled by Factory.
- Purple highlight for custom rows.
- Contact email requirement before sending.
- Valid Till drives Expired.
- Overpayment warning and duplicate protection.
- Payment History columns, Total Received Amount, Partially Paid logic.
- “Get Shipping rate” is an estimate and is not saved to the quotation.
- Business-day follow-up schedule (3/10/25/30) and that follow-ups are not auto-sent.
- Bank-transfer discount behaviour and Payment History locking.

## E. Unclear — marked **[CONFIRM WITH TEAM]** in the guide

1. Production workflow equals the local `Quotation-3` (user confirmed it does).
2. Source of the default 30-day Valid Till.
3. Whether CS users see the “no permission to edit Cost of Goods” red message on each item add.
4. Whether CS can save Estimated Delivery Date from the dispatch dialog (field is permission level 1).
5. Whether Export order type is planned.
6. Preferred process for Slack approvals (Slack “Send Quotation to Client” flips state only).
7. Not verified: live Shopify/Slack settings in production (`enable_payment_link` is 0 locally), live SLA rules, whether any live email template still uses the old guest “approve” link (it marks a quote Won without checking payment), downstream use of Shipment Contents, Analysis Dashboard formulas (taken from its own doc).

## F. Code issues found during the audit (not changed — for the dev team, not end users)

- `QJS:2438` compares to “Send to Customer” instead of “Sent to Customer”, so **Track Payment Status** never shows.
- Python `before_workflow_action` (QPY:1130) is never called by Frappe; its checks are dead.
- Slack “Send Quotation to Client” sends no email and creates no payment link.
- Factory Feasibility Approve ≥ 10,000 skips the Slack card and the Cost of Goods check.
- Realtime toast says “Marked as Won” for partial payments.
- Bank-transfer internal email still says to set “Payment Received”.
- Approval messages hard-code “₹10,000” though the comparison ignores currency.
- Old Shopify payment link is not cancelled on Cancel/Amend.
- Quotation docs `docs/quotation_request_technical_details_validations.md` and `docs/Quotation_End_User_Guide_v2.md` contain statements that no longer match the code.
