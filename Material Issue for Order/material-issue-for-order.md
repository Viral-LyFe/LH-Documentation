# Material Issue for Order — User Guide

> **Single source of truth** for how Material Issue for Order works. Written for factory staff, store staff, customer service and managers. No technical knowledge needed.
> Short name used throughout: **MIFO** (the document numbers look like `MIFO-2026-0123`).

---

## 1. What is a Material Issue for Order?

A Material Issue for Order is the record of **taking material out of the factory store and handing it to production for a specific customer order**. It answers three questions:

1. **What** was taken (which items, how many)?
2. **From where** was it taken (which rack / store location)?
3. **For which order** was it taken?

When you submit it, three things happen automatically:

- The store's stock goes **down** by the quantity you issued.
- The customer order records exactly **what was physically issued** for it. This list is later used for the Gate Pass, the invoice, the packing list and customs paperwork.
- The order is allowed to move forward toward dispatch (an order cannot be marked *Ready for Dispatch* without one).

There is also a **Return** type, used to put unused material back into the store. A Return does the opposite: stock goes back up and the order's issued list goes down.

---

## 2. When should I use it?

| Situation | What to do |
|---|---|
| Production is about to start on an order and material must be taken from the store | Create an **Issue** MIFO |
| Material was issued but some was not used or was issued by mistake | Create a **Return** MIFO |
| A customer's order is being sent again (a reshipment) | Create a MIFO from the **Reshipment** record |
| An issue was entered wrongly and not yet submitted | Edit the draft |
| An issue was entered wrongly and already submitted | Cancel it (or Amend it) — see section 11 |

Every order that goes to dispatch from the factory **must have at least one submitted MIFO**.

---

## 3. Who can do what?

| Role | Create / edit | Submit | Cancel | Amend |
|---|---|---|---|---|
| **Factory** | Yes | Yes | **No** | Yes |
| **System Manager** | Yes | Yes | Yes | Yes |

If you are in the Factory role and need to cancel a submitted MIFO, ask a System Manager.

---

## 4. The complete workflow (step by step)

### 4.1 Start a new MIFO

There are three ways:

1. **From the order (recommended).** Open the Lyfe Order → **Create** → **Material Issue for Order**. The order is filled in for you. This button appears only when the order is in the **Factory** warehouse, is in one of the active factory stages (*Approved, Factory Assignment, Awaiting Factory Upload, Ready for Dispatch, Dispatched, Rework Required*), and you are in the Factory or System Manager role.
2. **From a reshipment.** Open the Reshipment → **Create** → **Material Issue for Order**. The reshipment is filled in and its original order is filled in automatically.
3. **From the list.** Go to the Material Issue for Order list → **New**.

### 4.2 Fill in the top section

Date, Issued By, Type, and the order (see the field list in section 5).

### 4.3 Add the items

Pick whichever method suits you:

- **Get Items from Lyfe Order** — pulls in everything the order needs automatically (section 7).
- **Scan a barcode** — each scan adds one unit (section 8).
- **Add rows by hand** — pick an item, then adjust quantity and rack.

### 4.4 Review the rows

Check the quantity, rack and stock status on every row. Fix any shortages (section 9).

### 4.5 Tick the gift items (if the order has any)

See section 12.

### 4.6 Save, then Submit

Save first. Then press **Submit**. If something is missing the system tells you exactly what (section 10). Once submitted, the stock moves and the order is updated.

---

## 5. Fields explained

### 5.1 Top section

| Field | Meaning | Notes |
|---|---|---|
| **Date** | The date material is issued | Fills with today. Required. This is also the date shown on the stock movement. |
| **Issued By** | Name of the person issuing | Required. |
| **Type** | **Issue** (take out of store) or **Return** (put back) | Fills with *Issue*. Required. |
| **For Lyfe Order** | The customer order this is for | Optional at the top, but every row must have an order (see below). When filled, it is copied to every row that has no order. |
| **For Lyfe Order Reshipment** | The reshipment this is for, if any | Only shows reshipments that are still open (not Shipped / Completed). Picking one fills in its original order for you. |
| **Stock Entry** | The stock movement record created on submit | Read-only. Filled in after submitting. |
| **Order Dispatched** | Set by the system once the order has reached *Ready for Dispatch* | Read-only. Controls what happens when you cancel (section 11). |
| **Barcode Scanner** | Click here and scan | See section 8. |

### 5.2 Item rows

| Column | Meaning |
|---|---|
| **Item (SKU)** | The store item being issued. Item Code, SKU and item are the same thing in this system. |
| **Item Name** | Filled in automatically. |
| **Issue Qty** | How many to issue. Cannot be negative. Starts at 1. For tube items it is calculated as length × number of tubes (feet). |
| **Rack** | The store location the material comes from (Issue) or goes back to (Return). Only individual racks can be chosen, not warehouse groups. Filled automatically with the rack that has the most stock. |
| **Available Qty** | Stock currently in that rack. Read-only. Refreshed when you change the rack or save. |
| **Stock Status** | **In Stock** (enough), **Low Stock** (some, but less than needed), **Out of Stock** (none). Read-only. |
| **Create Material Receipt** | Button shown only when the rack has less stock than you need (section 9.3). |
| **Lyfe Order** | The order this row is for. **Required on every row.** |
| **Finished Item / Finished Item Name** | What the customer is actually receiving, when it differs from the store item (cut tube, different finish, cut acrylic). |
| **No. of Tubes** | Number of tubes cut (tube items only). |
| **Length (Ft)** | Length of each tube in feet (tube items only). |
| **Internal Order ID** | Filled in from the order. |
| **Acrylic Cuts** | The list of cut pieces for acrylic items (edited through a dialog, not typed). |

**Row colours:** a row with *Out of Stock* is shaded red; *Low Stock* is shaded orange.

---

## 6. Issue vs Return

| | **Issue** | **Return** |
|---|---|---|
| Stock in the rack | Goes **down** | Goes **up** |
| Rack means | Where material is taken **from** | Where material is put **back to** |
| Stock availability check | **Yes** | **No** (there is nothing to be short of) |
| Gift item selection required | Yes (if the order has gifts) | No |
| Effect on the order's issued list | Adds | Subtracts |
| Registers new items with 1Click | Yes | No |

The order's issued list always shows **net quantity = all Issues − all Returns**. Items whose net quantity falls to zero or below disappear from the list.

---

## 7. Getting items automatically from an order

Press **Get Items from Lyfe Order** (visible while the document is a draft), choose one or more orders, and press **Fetch Items**.

**What it does for each line on the order:**

| Order line | What gets added |
|---|---|
| Item **has a BOM** (recipe / bill of materials) | Every **component** of the BOM, with quantity = component quantity in the BOM × quantity ordered |
| Item has **no BOM** | The item itself, with the ordered quantity |
| Item has **no linked ERP item** (custom item) | Not added automatically — a second window opens so you can choose the SKU yourself (section 7.2) |
| Fee lines such as *customs fee* or *shipping charge* | Skipped completely |

Each added row gets the **best rack automatically** — the rack, within the factory store, holding the most stock of that item (the rejected-material warehouse is never chosen). Stock status is worked out straight away.

**Example 1 — BOM item.** Order wants **3 × Handle Set**. The Handle Set BOM contains **2 × Screw** and **1 × Plate**. Result: 6 Screws and 3 Plates are added.

**Example 2 — No BOM.** Order wants **10 × Plain Washer** with no BOM. Result: 10 Plain Washer are added.

**Example 3 — Several orders at once.** Select orders A and B. Rows from both are added, and each row keeps the order it belongs to. One MIFO can therefore cover several orders.

The order list in this window leaves out orders that are already past the point of needing material (*Shipped, Completed, Awaiting Tracking, Rework Required, CS Team Assignment, Size Approved, Internal Review, Pending Customer Approval*).

If nothing eligible is found, you see **"No eligible items were found for the selected Lyfe Order(s)."**

### 7.1 Which orders can I pick in the order field?

An order is offered if:

- it has **no invoice yet** and is **not** Completed, Shipped or Cancelled; **or**
- it has an **open reshipment** (not Shipped / Completed) — even if the order itself is already Completed and invoiced, because material must be issued again to fulfil the reshipment.

You only see orders your role is allowed to see.

### 7.2 Custom items with no SKU ("Unresolved Order Items")

For lines with no linked item, a window lists them as yellow tags and gives you a **SKU Selection** table. Choose the item, quantity and order for each (you can also scan barcodes here), then press **Add to Form**. Stock and rack are looked up for you. Rows missing an item or an order are ignored.

---

## 8. Scanning barcodes

Click the **Barcode Scanner** box on the form and scan.

- The system recognises an item code, a SKU, or a barcode registered on the item.
- **First scan** of an item → a new row with quantity **1**.
- **Scan again** → quantity goes up by **1** on the same row.
- A small message shows the item and the running quantity (green = enough stock, orange = not enough).
- Unknown barcode → red message **"Item not found"**.
- If the top of the form has an order, new scanned rows get that order. If not, you must fill the order on the row yourself.

---

## 9. Stock availability

### 9.1 How it is checked

Available quantity is the stock sitting in the **chosen rack** for that item. It is shown on each row and re-checked when the document is saved. If the rack has less than the issue quantity, the row shows *Low Stock* (some) or *Out of Stock* (none).

A red banner at the top of a draft **Issue** lists every item that is short, for example:
*"⚠ Insufficient stock for: SCR-100 (need 20, have 12)"*.

### 9.2 What happens when you submit with a shortage

1. The screen shows a warning: **"The following items have insufficient stock… Proceed anyway?"**
2. Even if you choose to proceed, the system **stops the submit** with **"Insufficient Stock"**, listing each row (*available* vs *need*).

So in practice a shortage always blocks submission. You must fix it first. Fractions are ignored when checking stock (a need of 2.5 is checked as 2).

### 9.3 Fixing a shortage — Create Material Receipt

If the physical stock is actually there but was never recorded in the system, use the **Create Material Receipt** button on the row (it appears only when a rack is chosen and the issue quantity is greater than the rack's stock).

1. Press the button and confirm.
2. The system adds **only the missing amount** to that rack (e.g. need 20, have 12 → receives 8).
3. The row refreshes to show the new available quantity, with a message **"Material Receipt … created. Stock updated."**

Notes:
- It works even if the MIFO is not saved yet.
- It always uses the real current stock, so two people doing it at once will not double-receive.
- It cannot be used on a group warehouse ("Invalid Warehouse") or if the stock is already enough ("No Shortage").
- Use it only when the stock truly exists. It increases recorded stock.

**Other ways to fix a shortage:** choose a different rack that has stock, reduce the quantity, or issue the rest in a later MIFO.

---

## 10. Validations and rules (everything that can stop you)

### 10.1 While editing / saving

| Rule | What you see |
|---|---|
| Date, Issued By and Type are required | Standard "mandatory" message |
| Each row must have a **Lyfe Order** | Standard "mandatory" message |
| Issue quantity cannot be negative | Standard message |
| Rack must be an individual rack, not a group | The group cannot be selected |

### 10.2 When you press Submit

These are checked **on the server**, so they apply whichever way the document is submitted.

| # | Rule | Message title |
|---|---|---|
| 1 | **Gift items:** for an **Issue**, every order on the MIFO that has gift items needs **at least one gift ticked** | *Gift Item Required* — "Select at least one issued gift item for: …" |
| 2 | **Every row must have an Item Code** (rows added empty) | *Incomplete Rows* — "Row N: Item Code is required." |
| 3 | **Enough stock** in the rack for every row (**Issue only**) | *Insufficient Stock* |
| 4 | **Tube items must have a Finished Item** | *Finished Item Required* — "Row N: … is a tube item — Finished Item is required." |
| 5 | **Cut-length acrylic items must have at least one cut piece** | *Acrylic Cuts Required* — "Row N: … is an acrylic item — at least one cut piece is required." |

A MIFO also needs at least one item row and a rack on every row — otherwise the stock movement cannot be created and submission fails.

### 10.3 Rules that apply elsewhere because of MIFO

| Where | Rule |
|---|---|
| **Lyfe Order → Ready for Dispatch** | Blocked with **"Missing Material Issue"** unless the order has a submitted MIFO. (Exception: orders already delivered to the US warehouse — their material was issued in the first leg.) |
| **Gate Pass → Submit** | Blocked with **"Missing Material Issue"** if any order row has no submitted MIFO (an admin can switch on *Bypass Material Issue Check* for a row). |
| **Posting an order to 1Click** | Needs a submitted MIFO so the real issued items are sent. Otherwise: "no Material Issue for Order has been submitted… submit the MIFO first." |
| **Reverting a split order** | Blocked while the split order has a submitted MIFO — cancel the MIFO first. |
| **Cancelling a MIFO after dispatch** | See section 11. |

---

## 11. What happens after submission

### 11.1 Immediately

1. **Stock moves.** A stock movement record (Stock Entry) is created and submitted automatically, dated with the MIFO's **Date**, and its number appears in the *Stock Entry* field.
   - **Issue** → *Material Issue* (stock out of the rack).
   - **Return** → *Material Receipt* (stock into the rack).
2. **The order's issued list is rebuilt.** The order's shipment items now show the net quantity physically issued, together with HSN/HTS codes, unit and price from the order's lines. If you had manually edited things like price on that list, your edits are kept; only quantities are refreshed.
3. **Draft documents follow along.** A *draft* Sales Invoice for the order is updated to match. A *draft* Purchase Order for the order is updated to match (it is never left with zero items). Submitted invoices and POs are **not** changed — they must be amended by hand.
4. **1Click registration (Issue only).** Any item that 1Click does not yet recognise is registered with 1Click before the stock moves. If 1Click is unavailable or fails, **nothing is blocked** — the issue still goes through and the problem is logged for support.
5. The order's progress bar shows **✓ Material Issued** (green).

Automatic creation of Purchase Orders from a MIFO is switched off.

### 11.2 How special items appear on the order's issued list

| Item type | What the order's list shows |
|---|---|
| Normal item | The item and issued quantity |
| **Tube** item | The **Finished Item** (e.g. `8FT-TB-150-SB`) with quantity = **number of tubes** |
| **Finish-changed** item | The **Finished Item** with the issued quantity |
| **Acrylic** item | One line per **cut piece** (the uncut source item is not listed) |

Lines for the same finished item are merged into one.

### 11.3 After submission you cannot edit

The form becomes read-only. Gift tick boxes become locked. To change anything, **Cancel** (and optionally **Amend**).

### 11.4 Cancelling

Only a **System Manager** can cancel.

- The stock movement is cancelled, so stock goes back to how it was.
- The order's issued list is rebuilt without this MIFO. If that was the only Issue, the list becomes empty.
- **If the order has already reached Ready for Dispatch** (the MIFO shows *Order Dispatched*), extra care applies because a Purchase Order may exist:

| Linked Purchase Order | Result |
|---|---|
| None | Cancel proceeds |
| **Draft** | The draft PO is **deleted automatically**, then cancel proceeds |
| **Submitted** | Cancel is **blocked**: *"Cancellation Blocked — Purchase Order Is Submitted"*. Cancel or amend the Purchase Order first, then retry. |

Because the Purchase Order belongs to the **order**, not to one MIFO, cancelling any one dispatched MIFO removes that order's draft PO even if other MIFOs exist.

### 11.5 Amending

Amend creates a corrected copy of a cancelled MIFO. The copy keeps the *Order Dispatched* marker. Edit, then submit as normal.

---

## 12. Gift items

Some orders include free **gift items**. They are shown at the bottom of an **Issue** MIFO in the **Gift Items Issued** section, one card per order, each listing the gifts with tick boxes and a counter such as *"1 of 2 selected"* (green when at least one is ticked, red when none).

How it works:

- **Tick the gifts you physically issued.** Each tick is saved to the order immediately — before you submit — and updates the order's cost of goods (ticked gifts count toward cost; unticked do not).
- **Rule:** for every order on the MIFO that has gift items, **at least one gift must be ticked**, otherwise submission is blocked with *Gift Item Required*. The screen scrolls you to the section.
- The section appears **after the MIFO has been saved once**.
- It does **not** appear for **Returns**. If the orders have no gifts, it says *"No gift items on these orders."*
- After submission the tick boxes are locked.
- If a tick fails to save, the box flips back.

**Example.** One MIFO covers orders A (two gifts: Keychain, Sticker) and B (no gifts). Tick *Keychain* for A. Order B needs nothing. Submit works. If you had ticked nothing for A, submit would be blocked: "Select at least one issued gift item for: A".

---

## 13. Special item types

### 13.1 Tube items

When you pick a **tube store item** (a long stock tube), a **Tube Details** window opens. It cannot be closed — you must press **Update**.

| Ask | Meaning |
|---|---|
| Finish | Colour/finish of the finished tube |
| Diameter (inches) | Pre-filled from the item; e.g. 1.5 |
| Length (Feet) | Length of each tube |
| Number of Tubes | How many tubes to cut |

The system works out the **cut-tube SKU** (e.g. 8 ft, 1.5", Satin Black → `8FT-TB-150-SB`), creating it if it does not exist yet, and puts it in **Finished Item**. **Issue Qty becomes length × number of tubes**, i.e. total feet taken from stock.

**Example.** 4 tubes of 8 ft → Issue Qty = **32** (feet) from the store; the order's list shows **4** of the finished tube.

Changing **No. of Tubes** or **Length (Ft)** later re-works the SKU and quantity. The **Edit Tube Details** button (draft only) reopens the window; with several tube rows, you pick the row first.

A tube row cannot be submitted without a Finished Item.

### 13.2 Finish-change items

If the item code contains a recognised finish (e.g. `PB` in `5FT-HRK-PB-200-WR`), a **Select Finish** window opens:

| Choice | Result |
|---|---|
| **Same Finish (no change)** | Nothing changes |
| **Choose a different Finish** | The matching SKU for the new finish is found (or created) and set as the Finished Item |
| **Deliver As Item / SKU** | Use when the whole SKU is different, not just the finish. Overrides the finish choice. |

Stock is **still issued from the original item**. Only the Finished Item (what is delivered, used in paperwork) changes. A new SKU, if created, is a copy of the original without its online-store links or barcodes.

### 13.3 Acrylic rod items

For cut-length acrylic items (round `ACR-…` or rectangular `RAC-…`), an **Acrylic Cut Pieces** window opens.

- Round rod: confirm the **diameter**. Rectangular rod: confirm the **width code**.
- Add one line per cut: **cut length** and **number of pieces**. A SKU preview appears (e.g. `ACR-075-24L`).
- **Save Cuts** creates any missing SKUs and stores the list.
- At least one cut is needed to submit.
- **Edit Acrylic Cuts** (draft only) reopens it; with several acrylic rows you pick the row first.
- If a cut resolves to the same SKU as the source item, the Issue Qty is set to the number of those pieces.
- Acrylic items that only differ by finish (e.g. `RAC-175-SN`) use the **Finish** window, not the cuts window.

---

## 14. Dialogs, warnings and messages

| Message | When | What to do |
|---|---|---|
| **Material Already Issued** (table of earlier issues) | You choose an order on a row and that order already has submitted Issue MIFOs | Press **Yes, Issue Again** to continue, or **No** to clear the order from that row. Only previous *Issues* are listed (not Returns). |
| **Warning: items have insufficient stock — Proceed anyway?** | Submit with a shortage | You can continue, but the system will still stop it (section 9.2) |
| **Insufficient Stock** | Submit with a shortage | Fix stock/rack/quantity |
| **Incomplete Rows** | A row has no item | Fill or delete the row |
| **Finished Item Required** | Tube row without finished item | Use Edit Tube Details |
| **Acrylic Cuts Required** | Acrylic row without cuts | Use Edit Acrylic Cuts |
| **Gift Item Required** | An order's gifts all unticked | Tick the issued gift(s) |
| **Missing Material Issue** | Dispatch / Gate Pass with no submitted MIFO | Submit a MIFO for the order |
| **Cancellation Blocked — Purchase Order Is Submitted** | Cancel after dispatch with a submitted PO | Cancel/amend the PO first |
| **No Items Found** | Get Items finds nothing eligible | Choose different orders or add items by hand |
| **Item not found** | Unknown barcode | Check barcode / item master |
| **Could not resolve tube SKU / finish SKU** | Missing finish or diameter | Check finish and diameter, try again |
| **Missing Abbreviation** | The chosen finish has no short code set up | Ask an administrator to set the finish abbreviation |
| **Invalid SKU** | The item code is not in an expected pattern for finishes/acrylic | Pick the correct item or ask an administrator |
| **Invalid Warehouse / No Shortage / Creation Failed** | Material Receipt button problems | See section 9.3 |

---

## 15. Common scenarios

### 15.1 Normal issue for one order
1. Order → Create → Material Issue for Order.
2. **Get Items from Lyfe Order** → pick the order → **Fetch Items**.
3. Check racks and stock; fix any red/orange rows.
4. Enter *Issued By*, tick gifts if any, Save, Submit.

### 15.2 One MIFO for several orders
Leave the top order blank, use **Get Items from Lyfe Order** and select several orders. Every row keeps its own order. On submit each order's issued list is updated separately.

### 15.3 Issue in two parts (partial issue)
There is no "partial" setting — a MIFO is simply the quantity you enter.
- Order needs 10 screws, only 6 available now → issue **6** in the first MIFO.
- When stock arrives, create a second MIFO for the remaining **4**.
- The order's issued list adds up to **10**. When you open the order in the second MIFO, the **Material Already Issued** window reminds you of the first one — choose **Yes, Issue Again**.

### 15.4 Issuing the full amount
Issue the whole quantity in one MIFO. A single submitted MIFO is enough for dispatch.

### 15.5 Material issued by mistake or not used
Create a **Return** MIFO for the same order, item and quantity, with the rack to put it back into. Stock goes up; the order's list goes down by that quantity. If it falls to zero the item leaves the list.

### 15.6 Wrong item or quantity on a submitted MIFO
- If you are a System Manager: **Cancel**, then **Amend** and submit the corrected one.
- Otherwise: ask a System Manager, or create a **Return** for the mistake and a new **Issue** for the correct material.

### 15.7 Order with an item that has no linked ERP item
Get Items puts it in the "Unresolved Order Items" window. Choose the correct SKU there.

### 15.8 Reshipment
Create the MIFO from the Reshipment. Issues for a reshipment are tracked **separately**: they build the reshipment's own issued list and never change the original order's list (and the original order's issues never change the reshipment's).

### 15.9 Cut tubes
See 13.1: choose the tube, fill the Tube Details, Issue Qty is calculated in feet.

### 15.10 Stock is physically there but the system says zero
Use **Create Material Receipt** (section 9.3) or, for a larger correction, ask whoever manages stock to record a proper receipt.

---

## 16. Troubleshooting

| Problem | Likely reason | What to do |
|---|---|---|
| "Create → Material Issue for Order" button is missing on the order | Order not in the Factory warehouse, not in an active factory stage, or you are not Factory/System Manager | Check the order's warehouse and stage, or create from the MIFO list |
| I can't find my order in the order field | It already has an invoice or is Completed/Shipped/Cancelled with no open reshipment, or your role can't see it | Check the order's status. For a reshipment, create the MIFO from the Reshipment instead. |
| Can't submit: "Insufficient Stock" | Rack has less than you need | Pick a rack with stock, lower the quantity, or add a Material Receipt if the stock exists physically |
| Can't submit: "Gift Item Required" | An order has gifts and none ticked | Tick the gifts actually issued (the section appears after first Save) |
| Can't submit: "Finished Item Required" / "Acrylic Cuts Required" | Tube or acrylic details not completed | Use **Edit Tube Details** / **Edit Acrylic Cuts** |
| Can't submit: "Incomplete Rows" | A blank row | Fill it or delete it |
| Gift section is empty | MIFO not saved yet, or Type is Return | Save first; gifts only apply to Issues |
| Order can't go to Ready for Dispatch: "Missing Material Issue" | No submitted MIFO for the order | Submit one |
| Gate Pass blocked: "Missing Material Issue" | One of the orders has no submitted MIFO | Submit it (or ask an admin about the bypass) |
| Can't cancel a MIFO (no Cancel button) | Factory role cannot cancel | Ask a System Manager |
| Can't cancel: "Purchase Order Is Submitted" | The order's PO is already submitted | Cancel/amend the PO, then cancel the MIFO |
| Order's issued list not what I expected | List is **net** (Issues − Returns), tubes/acrylic show finished items, and only submitted MIFOs count | Check all the order's MIFOs, including Returns and cancelled ones |
| Rack not offered | Group warehouses can't be chosen | Pick an individual rack |
| Quantity changed after choosing tube details | Issue Qty is length × tubes | Edit the tube details instead of typing over it |
| A new item showed up that I did not create | The system created the cut-tube / finish / acrylic SKU for you | Normal — see section 13 |
| 1Click error appears in logs | 1Click was unreachable during submit | Nothing to do; issue was not blocked. Tell support if the item is missing in 1Click. |

If a problem is not on this list, note the **MIFO number**, the **order number**, and the **exact message**, and send them to the support / development team.

---

## 17. Quick reference

- **Issue** = stock out, needs stock check, needs gift tick. **Return** = stock back in.
- Every row needs an **item, a rack and an order**.
- Tube → **Finished Item**. Cut acrylic → **cut pieces**. Gifts → **at least one ticked per order**.
- Shortage always blocks submit — fix stock first.
- Submitted MIFO → stock moves, the order's issued list updates, draft invoice/PO update, order can proceed to dispatch.
- Cancel = **System Manager only**; blocked if the order's PO is submitted.
- Order's issued list = **Issues − Returns**, submitted MIFOs only.
