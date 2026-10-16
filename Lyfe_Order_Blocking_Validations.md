# Lyfe Order — Where the System Stops the User

This document lists every place where the Lyfe Order stops a user from saving it, moving it to the next step, or running an action on it. For each one it gives the message the user sees and the reason the check exists.

Last reviewed: 2026-10-16

**How to read the "Checked" column**
- **Screen** = the check runs in the user's browser, so the message appears instantly.
- **System** = the check runs in the system itself, so it also applies to imports, bulk actions and Admin.
- **Both** = the browser shows the message early and the system checks again. This is the usual case.

**Where the rules live:** all rules below are in the LH app, except the items in the box.

> **Not part of the LH app**
> 1. **Workflow setup** (which steps exist and which role can click which button) is configured in the site's Workflow, not in LH code. Section G lists the steps and roles as found on the local site.
> 2. **Outside services** decide some of the messages: 1Click Logistics (US warehouse orders and stock), the tracking service that verifies tracking numbers, and Shopify. When the message depends on them, it is marked "(outside service)".

---

# Where users are blocked

## A. When saving the order

| # | Message the user sees | Why we block | Checked |
|---|---|---|---|
| A1 | "Carrier is mandatory when a Tracking Number is provided." | A tracking number is useless without knowing the carrier. | System |
| A2 | "Please provide a Tracking Number and Carrier (main or US leg) before marking the order as Shipped." | An order cannot be Shipped without tracking. Either the main shipment or the US-leg shipment is enough. | System |
| A3 | "Cannot save: tracking number {X} could not be verified." (also "Cannot save: US tracking number…") | A changed tracking number is checked with the tracking service. A number it rejects is not saved. A number that is only "not yet scanned" is accepted. (outside service) | System |
| A4 | "Shipping Charges" becomes a mandatory field | Once a tracking number and carrier are entered, the shipping cost must be filled. Not required for free shipments or orders that belong to a Bulk Transfer Batch (those take their costs from the batch). | Screen |
| A5 | "Item Group is not set for any order item. Please set Item Group for at least one row before saving." | Customer Service must classify the order's items before saving. Applies to the Customer Service role only. | Screen |
| A6 | "Item Group is not set for: {items}. Please update the Item Group in the Order Items table before uploading images." | Images should not be attached while items are unclassified. Fee and adjustment lines are exempt. | System |
| A7 | "Custom Charges (Factory) and Custom Duty Changes (US Tram) cannot both be set. Only one of these two fields may have a value at a time." | The two fields are alternatives. Having both would double count. | System |
| A8 | "Warehouse Shipment tables are locked once the linked Transfer Order ({X}) has shipped." | Changing the item split after the goods have shipped would no longer match what was sent. | System |
| A9 | "Factory Leg Destination is locked once the warehouse split has been confirmed." | The route cannot change after the split is confirmed. | System |
| A10 | "Route Plan cannot be changed to 'Force US / Force India' via a plain save — use the Force US / Force India confirmation dialog…" | The route must change together with the routing result. A plain save would leave them out of step. | System |
| A11 | Fields that become mandatory: **Skip QC Reason** (when Skip QC is ticked), **Override Reason** (when Route Plan is not Auto), **Notes** (when notes are required), **Insured Value** (when an insurance provider is chosen), **Bill To Account No., Postal Code and Country Code** (when a bill-to party is chosen) | Every override or special option must come with its details. | Screen |
| A12 | "Failed to save — please save manually." and "Failed to save Dispatch Again Date. Please try again." | A background save did not go through. | Screen |

## B. When moving the order to the next step

| # | Message the user sees | Why we block | When | Checked |
|---|---|---|---|---|
| B1 | "This order is assigned to US Warehouse and cannot be sent to the Factory." | US Warehouse orders are fulfilled from the US, not the Factory. | *Assign To Factory* | Both |
| B2 | "Please attach the Drawing (Preview PDF) before proceeding with Engineers Checks." | Engineers need the drawing to check. | *Engineers Checks* | Screen |
| B3 | "{N} QC item(s) on this order are marked **Fail**. This order cannot proceed to Awaiting Factory Upload — click Reject to send it for Rework instead." | A failed quality check must be reworked, not skipped past. | *Awaiting Factory Upload* from Awaiting Engineer Checks (Factory orders where QC is not skipped) | Both |
| B4 | "{N} QC item(s) on this order are still **Pending**. Every item must be marked Pass, Fail, or Skipped before this order can proceed to Awaiting Factory Upload." | Quality checks must be finished first. | Same as B3 | Both |
| B5 | "Please upload new factory images since the last rejection before submitting for Internal Review." | After a rejection, the factory must show the corrected work. | *Awaiting Internal Submission* | Screen |
| B6 | "Please provide a reason for rejection." | A rejection must explain what to fix. | *Reject* | Screen |
| B7 | **Export Partner Required:** "Please set the Export Partner (Consignor) before marking this order as Ready for Dispatch. A Purchase Order cannot be created without it." | The Purchase Order cannot be created without the export partner. | *Ready for Dispatch* (Factory orders; not for the second leg from the US Warehouse) | Both |
| B8 | "The following order items have no COGS: {items}. Set COGS before dispatching." | Every item needs its cost before the order leaves. Fee, adjustment and "BOM not required" lines are exempt. | *Ready for Dispatch* (same conditions as B7) | Both |
| B9 | **Required Fields Missing:** a list of everything still empty: Export Partner, Consignor name, contact, address, city, state and phone, GST type, GSTIN, Ship Purpose, Duty Tax, Package Type, CSB Type, DDP, Goods Description; at least one Shipment Item; each Shipment Item needs Item Name, Qty, Rate (USD), HSN code or HTS code. The system also needs End Use, Material, HSN, HTS and Net Weight on each row | The export paperwork must be complete before the goods are dispatched. | *Ready for Dispatch* (same conditions as B7) | Both |
| B10 | "No **QC** records exist for this order. Move the order back to Awaiting Engineer Checks to generate and complete QC before dispatching." / "{N} QC item(s) are still Pending…" | Custom orders must have every item quality-checked before dispatch. Standard orders are exempt. | *Ready for Dispatch* (Custom orders where QC is not skipped) | Both |
| B11 | "Order {X} has no submitted **Material Issue for Order**. Please submit a Material Issue before marking this order as Ready for Dispatch." | The stock used for the order must be recorded first. Not required for the second leg from the US Warehouse. | *Ready for Dispatch* from Factory Assignment | System |
| B12 | **Payment Incomplete** (see section D) | The agreed payment must be received before production or dispatch. | *Factory Assignment*, *Approved*, *Ready for Dispatch* | System |
| B13 | "Tracking number is required to mark as Shipped." / Shipping Label is mandatory in the Add Tracking dialog | Tracking details are needed to move the order to Shipped. | *Add Tracking* | Screen |
| B14 | "Cannot revert: Last Status is not set." | There is no earlier step to go back to. | *Revert Workflow State* | System |
| B15 | "The following orders are not in 'New' state…" / "The following orders are assigned to US Warehouse and cannot be sent to the Factory…" / "Please select at least one order." / "Please select a user to assign." | Bulk assigning is only for new, non-US orders and needs a user. | Bulk *Assign to Factory* | System |
| B16 | "The following orders are not in 'Ready for Dispatch' state…" / "Please select at least one Lyfe Order." | Gate Passes are only for orders that are ready to dispatch. | Bulk *Create Gate Pass* | System |

## C. Routing, 1Click and US Warehouse actions

| # | Message the user sees | Why we block |
|---|---|---|
| C1 | "An override reason is required." / "route_plan must be Force US or Force India." | A manual route change must be explained. |
| C2 | "This order has already been submitted to 1Click ({ID}) — nothing to route." | Routing cannot change after the order is in 1Click. |
| C3 | "This order is in status {X} — too far along to re-route." | Too late to change the route. |
| C4 | "This order already has a Factory Leg Destination set… Force US cannot be applied to the same order. Clear that split first…" | Force US and a mixed US/Factory split cannot both apply to one order. |
| C5 | "This order's routing outcome ({X}) is not a mixed US/Factory split — nothing to confirm." | Only mixed orders have a split to confirm. |
| C6 | "This order's warehouse split has already been confirmed." / "Factory Warehouse Shipment table is empty — nothing to confirm or assign." / "Set the Factory Leg Destination (Via US Warehouse or Direct to Customer) before confirming the split." | The split can be confirmed once, needs Factory items, and needs a destination. |
| C7 | "The warehouse split for this order is already confirmed — Factory Leg Destination can only be set beforehand." / "…is already set to {X}." / "Invalid Factory Leg Destination: {X}" / "This order was just set to Force US — a Factory Leg Destination cannot be added…" | The destination is set once, before confirming, and never on a Force US order. |
| C8 | "Confirm the warehouse split first before posting the US portion to 1Click." / "The US portion of this order has already been posted to 1Click." / "US Warehouse Shipment table is empty — nothing to post." | The US part is posted only after the split is confirmed, only once, and only if it has items. |
| C9 | "Not posted to 1Click: US stock no longer covers this order — {SKU}: needs X, 1Click US warehouse shows Y." The order moves to **1Click Error** and a Slack alert is sent. (outside service) | The US warehouse does not have enough stock. Posting would fail or oversell. |
| C10 | "1Click inventory check failed — could not verify US Warehouse stock. Use Recheck Inventory once the issue is resolved." (outside service) | If stock could not be checked, the order must not go ahead on unverified information. |
| C11 | "This order's 1Click Error was not caused by a failed inventory check — nothing to recheck here." | Recheck is only for inventory errors. |
| C12 | "No items to submit to 1Click — all lines are adjustments, excluded fees, or quantity=0." | There is nothing real to send. |
| C13 | "Only applicable to an India-direct order (current routing_outcome: {X})." / "This order already has a live shipment (AWB {X})…" / "…already has a submitted Sales Invoice ({X}) — cancel that first." / "…a submitted Purchase Order ({X}) — cancel that first." / "…a Factory → Customer leg with a tracking number ({X}) — cancel/clear that shipment first…" | Re-routing via the US Warehouse is only for India-direct orders and only if nothing is already committed. |
| C14 | "Cannot switch to Factory-only fulfillment — 1Click order {X} already exists…" / "This order is not currently in a US/Factory split fulfillment state…" / "An override reason is required to force Factory-only fulfillment." | The switch is allowed only for split orders without a 1Click order, with a reason. |
| C15 | "This action is only available while the order is Shipped, US Warehouse Delivered or 1Click Error (currently {X})." / "Posting to 1Click failed — see the error on this order for details: {X}" | Posting after US delivery is only valid at those steps. (outside service) |
| C16 | "No BOM items found. Set a BOM on this order first." / "US Warehouse ID is not configured in Oneclick Settings." / "BOM items have no SKU or Item Code set." / "1Click inventory endpoint is not configured. Please check Oneclick Settings." | The US stock check needs a BOM and the 1Click settings. |
| C17 | "This order has not been posted to 1Click yet." / "No saved 1Click request payload found for this order — cannot resend items." / "Create Load has not been posted for this order yet." | Create Load and its status check only work after posting. |
| C18 | "This order has no Factory-bound items to report." | There is nothing to put in the stock update file. |
| C19 | "1Click Check Failed" with the error from 1Click (outside service) | The pre-check against 1Click found a problem. |

## D. Payment checks that stop the Lyfe Order

The Lyfe Order is blocked when its linked Quotation has not received the amount agreed for that step in the Payment Schedule. These are the same checks described in the Quotation document.

| # | Step that is blocked | Payment Schedule stage that decides it |
|---|---|---|
| D1 | Assigning to Factory / moving to Factory Assignment (also when the warehouse is changed to Factory) | Advance Amount |
| D2 | Marking the order Approved | Photo Approval |
| D3 | Marking the order Ready for Dispatch (Factory orders only) | Before Dispatch |

- **Customer Service and System Manager** see: "Payment required before {step}. Quotation… Required (cumulative)… Received so far…"
- **Everyone else** sees: "This order cannot proceed — payment is pending. The CS team has been notified to follow up on the payment status."
- **Why:** production and dispatch should not start before the agreed payment is received.
- The first block records the date and sends payment reminders to CS.
- Nothing is blocked if the order has no Quotation, or no payment stage is ticked for that step.

## E. Splitting, merging and other actions

| # | Message the user sees | Why we block |
|---|---|---|
| E1 | "No splits defined." / "Warehouse is required for Split {X}." / "Split {X} has no items assigned." / "SKU {X}: assigned N but only M available." / "SKU {X}: N of M unassigned. All items must be distributed." | A split needs a warehouse, items, and every quantity accounted for. |
| E2 | "Cannot split a split order. Please split the original order instead." | Splits are one level only. |
| E3 | "Cannot revert {X} — it is in {state} state. Only New / On Hold orders can be reverted." / "…it has N submitted Material Issue(s). Please cancel them first." / "…it is referenced in Gate Pass {X}." / "{X} is not a split order." | A split is removed only if it has not progressed and has no linked documents. |
| E4 | "Only Customer Service or System Manager can revert split orders." / "Only Customer Service can perform this action." (Update Note, Run 1Click Checks) / "Only Factory users can request contact info." | Role restrictions. |
| E5 | "You already have a pending or approved request for this order." | One contact-info request at a time. |
| E6 | "{X} is now in status {Y} — merge is no longer possible. Only orders in New / Internal Review / Factory Assignment can be merged." | Orders that have moved on cannot be merged. |
| E7 | "{X} has already been merged into {Y}." / "{X} has N open split order(s). Revert or complete the splits before merging." / "Shipping address of {X} does not match {Y}. Cannot merge orders with different shipping addresses." / "Cannot merge orders that are in a Bulk Transfer Batch: {X}" | Orders can be merged only when they are the same shipment with no open splits or batch. |
| E8 | "This suggestion has already been {confirmed / dismissed}." / "Only confirmed merges can be reverted." / "No merged order found on this suggestion." / "Cannot revert — merged order {X} is already in status {Y}…" / "Cannot revert merge — the merged order has N submitted Material Issue(s)…" / "…is referenced in Gate Pass {X}." / "Only Customer Service or System Manager can perform this action." | A merge suggestion can be acted on once, and reverted only while nothing depends on it. |
| E9 | "Sales Invoice {X} already exists for this order." / "No ERP Customer found for {X}. Link a Sales Order or ensure the Customer exists in ERPNext." / "No issued items found for this order. Submit a Material Issue for Order first." | An invoice needs a customer and issued items, and only one invoice per order (a cancelled one does not count). |
| E10 | "Please set a Shopify Order ID first." / "Order {X} not found on Shopify." / "Failed to fetch order from Shopify. Check error logs for details." (outside service) | Fetching Shopify details needs a valid Shopify order. |
| E11 | **Generate Shipping Label** (CJ tab): "Missing Fields in CJ Shipping Tab: please fill in…" lists consignor, consignee, parcel weight and size, customs value, goods description, ship purpose, duty tax, CSB type, package type, DDP, exporter invoice number, invoice PDF and shipment items. A second message names any shipment item row missing Rate, End Use, Material, HSN or HTS. "Please use the Generate Shipping Label dialog to provide shipment details." | The carrier cannot create a label with incomplete details. |
| E12 | "Please enter the required delivery date." / "Please select the Order Type." | The delivery-date calculator needs both. It only stops the tool. |
| E13 | BOM from drawing: "No drawing PDF attached." / "No BOM table found in the attached PDF." / "Please approve at least one BOM section before confirming." / "Please set a Parent Item for every approved BOM section before confirming." | The BOM tool needs a drawing, a table in it, and an approved section. |
| E14 | "No order items found. Add items to the Order Items table first." / "No deliverable items found on this order — nothing to post to 1Click." | Item-based tools need items. |

## F. Warnings only (these do not stop the order)

| # | What the user sees | Notes |
|---|---|---|
| F1 | "Duplicate Tracking Number: Order {X} already has the same tracking number." | A warning only. Orange. |
| F2 | "Error fetching BOM" / "Could not generate pick-pack PDF." / "BOM US Stock Check" result | Information messages. |
| F3 | Cost benchmark deviation check | Never blocks. It only flags. |

## G. Workflow steps and who can use them (as set on the local site)

The Workflow decides which role can click which button. The checks above run on top of it.

- **Customer Service:** New → Assign To Factory / Assign to CS Team / Internal Review / Awaiting Factory Upload / Ready for Dispatch / Cancel; Approve; Reject; Pending Customer Approval; Size Approved; Dispatch Again; Reshipment In Progress / Completed; On Hold → Assign To Factory (Factory orders) or Back to New (US Warehouse orders).
- **Factory:** Factory Assignment → Ready for Dispatch / Awaiting Engineer Checks / Internal Review / Assign to CS Team; Dispatched → Awaiting Tracking; Awaiting Tracking → Awaiting Shipping; Awaiting Factory Upload → Approve.
- **Engineer:** Awaiting Engineer Checks → Awaiting Factory Upload / Internal Review.
- **Shipping Forwarder:** Awaiting Shipping → Shipped; Shipped / Awaiting Shipping → US Warehouse Delivered.
- **Super Admin:** Shipped → Completed.

## H. Visibility rules

- Which orders a user can see is controlled by **Order Visibility Settings** (set in the app, no code change needed). Customer Service, System Manager and Engineer see all orders. The Factory team sees Factory warehouse orders and orders assigned to them. Other roles (for example Shipping Forwarder) see only Factory warehouse orders in the statuses set for their role.

## I. Written but not active today (these do not block)

- "SKU is required for Standard orders" (with a prompt to switch to Custom) is written but is not called from anywhere.
- "Order items are locked once submitted to 1Click" is written but is not called from anywhere.
- "Cost of Goods is 0" (a general block before dispatch) is written but is not called. The per-item COGS check in B8 is the one that runs.
- "Attach a drawing PDF only after every item has an ERP Item and BOM" is switched off.

## Observations

1. Most step checks run twice (on screen and in the system). If they disagree, the system's answer wins. One example: for Ready for Dispatch, the screen accepts a Shipment Item with either an HSN or an HTS code, while the system asks for both, plus End Use, Material and Net Weight. A user can pass the screen check and still be stopped by the system.
2. The Ready for Dispatch checks apply only to **Factory** orders and not to the second leg from the US Warehouse.
3. A failed 1Click stock check does not just show a message. It also moves the order to **1Click Error** and posts a Slack alert, so the order is parked until someone fixes it.
4. Payment checks (section D) use the Quotation's Payment Schedule, so a Quotation with a wrong schedule can block its Lyfe Order later.
