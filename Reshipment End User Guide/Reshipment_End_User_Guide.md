# Reshipment — End User Guide

| | |
|---|---|
| **Feature** | Lyfe Order Reshipment (including reshipments raised from a Return and reshipments sent through the US Warehouse) |
| **Who it is for** | Customer Service, Factory, Shipping Forwarder, Finance, Super Admin, System Manager |
| **Document version** | 1.0 |
| **Last updated** | 2026-10-12 |
| **Screenshots** | Every `📷 [SCREENSHOT RS-xx]` line is a placeholder. Take the screenshot, save it as the same ID (for example `RS-01.png`) and insert it there. |

---

## Contents

1. [What is a Reshipment?](#1-what-is-a-reshipment)
2. [Who can do what](#2-who-can-do-what)
3. [All the details of the feature](#3-all-the-details-of-the-feature)
   - 3.1 Naming
   - 3.2 The form — every section and field
   - 3.3 The workflow (statuses)
   - 3.4 What the system does for you automatically
   - 3.5 How a Reshipment affects the original Lyfe Order
   - 3.6 Reshipment and Return
   - 3.7 Costs
   - 3.8 Rules and error messages
4. [How to use the feature — step by step](#4-how-to-use-the-feature--step-by-step)
5. [Use cases — step by step](#5-use-cases--step-by-step)
6. [Where to see reshipments (lists, dashboards, reports)](#6-where-to-see-reshipments)
7. [FAQ and troubleshooting](#7-faq-and-troubleshooting)
8. [Glossary](#8-glossary)
9. [Document change log](#9-document-change-log)

---

## 1. What is a Reshipment?

A **Reshipment** is a second shipment of goods for an order that has already been shipped. Typical reasons:

| Reason in the system | Example |
|---|---|
| Carrier Lost | The parcel never arrived |
| Carrier Damaged | The parcel arrived broken |
| Wrong Item Sent | We sent the wrong product |
| Missing Items | The box was short of items |
| Manufacturing Defect | The product is faulty |
| Customer Return Replacement | The customer returned goods and is owed a replacement |
| Warranty Replacement | A replacement under warranty |
| Other | Anything else |

Instead of editing the original **Lyfe Order**, you create a separate **Lyfe Order Reshipment** document linked to it. The reshipment has **its own** items, tracking number, cost, Gate Pass and invoice, so the original order's history stays untouched and the reshipment can be reported on its own.

**Reshipment vs Return**

| | Reshipment | Return |
|---|---|---|
| Goods move… | **From us to the customer** (again) | **From the customer back to us** |
| Document | Lyfe Order Reshipment | Lyfe Order Return |
| Money | Cost to us (cost of goods + shipping) | Refund to the customer |

A customer who sends goods back and wants a replacement needs **both**: a Return and a Reshipment linked to it (see [3.6](#36-reshipment-and-return)).

<img width="1700" height="907" alt="image" src="https://github.com/user-attachments/assets/84fc0aed-8122-4c3c-9f49-b1998f5f6e11" />

---

## 2. Who can do what

| Role | Create | Edit | Workflow steps they perform | Cancel |
|---|:---:|:---:|---|:---:|
| **Customer Service** | ✅ | ✅ | Approve / Reject an Internal Review, mark **US Warehouse Delivered**, **Post to 1Click** | ❌ |
| **Factory** | ❌ | ✅ | Start Progress, Internal Review, Ready for Dispatch, Dispatched | ❌ |
| **Shipping Forwarder** | ❌ | Tracking fields | Awaiting Shipping, Shipped | ❌ |
| **Super Admin** | ❌ | ✅ | Shipped → Completed | ❌ |
| **System Manager** | ✅ | ✅ | Any | ✅ **(only role that can cancel)** |

> The **Create → Reshipment** button on a Lyfe Order is shown only to **Customer Service** and **System Manager**.

---

## 3. All the details of the feature

### 3.1 Naming

Every reshipment is named **`{Order ID}-R{number}`**, counting per order. For order `LH2946`: `LH2946-R01`, then `LH2946-R02`, and so on. You can create as many reshipments for one order as needed.

### 3.2 The form — every section and field

📷 **[SCREENSHOT RS-02]** — The form with all tabs visible (Details, CJ Shipping, Connections, Attachments, 1Click Logistics).

#### Header

| Field | What it is |
|---|---|
| **Lyfe Order** | The original order. Filled automatically and read-only. |
| **Return** | Optional link to the Lyfe Order Return this reshipment replaces goods for. Filled automatically when you create the reshipment from a Return. Only a Return of the **same order**, that is **not Cancelled**, is accepted. |
| **Customer** | Taken from the order. Read-only. |
| **Warehouse** | Where the reshipment ships from — **you must choose it**. Setting **Factory - LH** switches on the CJ Shipping tab, the Gate Pass and the invoice. |
| **Order Type** | Standard or Custom. Copied from the order. |
| **Workflow State** | Current status (see 3.3). Read-only — change it with the buttons at the top. |
| **Factory Assignment Date / Ready for Dispatch Date / Reshipment Completed Date** | Stamped automatically the first time each step happens. Read-only. Used for SLA reporting. |
| **Reason** | **Required.** One of the reasons listed in section 1. |
| **Via US Warehouse** | Tick if the parcel should travel Factory → 1Click US Warehouse → Customer. Locked once the reshipment is in a Bulk Transfer Batch. See [Use case 5](#use-case-5--reshipment-through-the-us-warehouse). |
| **Description** | Free text (support ticket number, notes). |
| **Signature Required / Photo Required / Note Required** | Copied from the order. |

#### Items

| Field | What it is |
|---|---|
| **Reshipment Items** | The items going out again. Copied from the order's items when you create the reshipment. You can **add, remove or change the quantity** of rows (for example, only 1 of 3 items was lost). **COGS** on each row is always editable. |
| **Reshipment COGS** | Total cost of goods of the reshipment (sum of COGS × Quantity). Calculated on every save. Read-only. |

#### Shipping (customer leg)

| Field | What it is |
|---|---|
| **Carrier**, **Tracking Number** | Where the parcel is going to the customer. For via-US reshipments the tracking number is filled in automatically from 1Click. |
| **Tracking Status**, **Tracking Details** | Filled automatically by the hourly tracking job. Read-only. |
| **Tracking Alert** (+ reason, since, last event) | Set automatically when the carrier never scans the parcel, tracking goes stale, the carrier reports an exception, or the parcel looks lost. |
| **Shipment Cost**, **Custom Charges**, **Additional Charges** | Costs of shipping to the customer (see 3.7). |

#### US Warehouse Leg (only when "Via US Warehouse" is ticked)

| Field | What it is |
|---|---|
| **Carrier (to US) / Tracking Number (to US Warehouse)** | Tracking for Factory → US Warehouse ("Tracking #1"). Filled from the Bulk Transfer Batch. |
| **Delivered to US Warehouse**, **US Warehouse Delivered At** | Set when the package reaches the US. |
| **Bulk Transfer Batch** | The batch the reshipment was sent in. Read-only. |
| **Customer Leg / charges to US** | The cost fields for this leg (see 3.7). |

#### CJ Shipping tab (only when Warehouse = Factory - LH)

Used to create the international shipping label (FedEx via CJ).

| Block | Fields |
|---|---|
| **Consignor (Shipper)** | Pick an **Export Partner** — name, contact, address, GST type and GSTIN fill in by themselves. |
| **Consignee (Receiver)** | Name, email, phone, address, city, state, zip, country. Copied from the order. |
| **Package Details** | **Parcel Details** table — weight, length, width, height. |
| **Customs & Shipment** | Customs Value and Currency, Goods Description, Ship Purpose, Terms of Sale, Duty & Tax, CSB Type, Package Type, DDP, Exporter Invoice No., Signature Required, Invoice Attachment (PDF). |
| **Shipment Items** | Item, Qty, **Rate (USD)**, **HSN Code**. Filled automatically when the Material Issue for Order is submitted (see step 4.5). |
| **Output** | **AWB No.**, **CJ Tracking No.**, **Shipping Label** — filled when you click **Generate CJ Label**. |

📷 **[SCREENSHOT RS-03]** — The CJ Shipping tab filled in.

#### 1Click Logistics tab (only when "Via US Warehouse" is ticked)

Read-only status of the 1Click order and load: order id, submitted at, tracking number, status, error message, load status. See Use case 5.

#### Connections / Attachments / References

| Place | What it shows |
|---|---|
| **Connections** | Linked **Material Issue for Order**, **Sales Invoice** and **Gate Pass**. |
| **Attachments** | **Factory Images** (photos uploaded by the Factory). |
| **References** | **Gate Pass** and **ERP Sales Invoice** created for this reshipment. |

### 3.3 The workflow (statuses)

```
Reshipment Request ──(Factory: Start Progress)──► Reshipment In Progress
                                                       │
                          ┌── (optional quality check) ┤
                          │  Internal Review ──Approve──► Approved ─┐
                          │        │                                 │
                          │      Reject ──► Rework Required ──► (back to Internal Review)
                          │                                          │
                          └──────────────────(Factory: Ready for Dispatch)◄─┘
                                                       ▼
                                              Ready for Dispatch
                                                       │ (Factory: Dispatched)
                                                       ▼
                                               Awaiting Tracking
                                                       │ (carrier has a label / system)
                                                       ▼
                                               Awaiting Shipping
                                                       │ (carrier scans the parcel)
                                                       ▼
                                                    Shipped
                                                       │ (carrier says Delivered)
                                                       ▼
                                                   Completed
   Cancelled — System Manager only, from Request / In Progress / Ready for Dispatch / Awaiting Tracking / Awaiting Shipping
```

📷 **[SCREENSHOT RS-04]** — The workflow action button at the top of the form (for example "Start Progress").

| Status | Meaning | Who moves it |
|---|---|---|
| **Reshipment Request** | Just created. | Factory → *Start Progress* |
| **Reshipment In Progress** | Factory is preparing the replacement. | Factory → *Ready for Dispatch* or *Internal Review* |
| **Internal Review** | Optional quality check of the replacement. | Customer Service → *Approve* or *Reject* |
| **Rework Required** | The review was rejected. | Factory → back to *Internal Review* |
| **Approved** | The review passed. | Factory → *Ready for Dispatch* |
| **Ready for Dispatch** | Packed and ready. Gate Pass and invoice are created automatically. | Factory → *Dispatched* |
| **Awaiting Tracking** | Handed over; waiting for the tracking number / carrier scan. | Shipping Forwarder / system |
| **Awaiting Shipping** | Label exists; carrier has not scanned it yet. | Shipping Forwarder / system |
| **Shipped** | Carrier scanned it; on its way. | System (hourly tracking) |
| **Completed** | Delivered to the customer. | System, or Super Admin |
| **Cancelled** | Reshipment called off. | System Manager only |

**Extra statuses for "Via US Warehouse"** (replace the direct route): **In Transit to US Warehouse** → **US Warehouse Delivered** → **Submitted to 1Click** (or **1Click Error**) → Awaiting Shipping → Shipped → Completed.

> **Tracking moves the status for you.** Once a tracking number and carrier are saved, the hourly tracking job sets: *Delivered* → **Completed**, *label created but not scanned* → **Awaiting Shipping**, *anything else (moving)* → **Shipped**. It never moves a reshipment back out of Completed or Cancelled.

### 3.4 What the system does for you automatically

| When | What happens |
|---|---|
| On save | **Reshipment COGS** is recalculated. The reshipment is named `{Order}-R{nn}`. |
| Warehouse is set to **Factory - LH** | The reshipment is **assigned to the Factory user**. The **Factory Assignment Date** is stamped. |
| Parent order is fulfilled from the US Warehouse (when the reshipment is created) | The **Lyfe Order** (not the reshipment) is assigned to the Factory user. |
| Status becomes **Ready for Dispatch** | The **Ready for Dispatch Date** is stamped. **Only if a submitted Material Issue for Order exists for this reshipment** — otherwise it is blocked (see 3.8). If Warehouse = Factory - LH: a **Gate Pass** is created (or today's draft one is reused) and a draft **Sales Invoice** is created. |
| Tracking number / carrier saved | Copied to the Lyfe Order's reship tracking fields. |
| Every hour | Tracking is checked, the status advances (see 3.3), and alerts are raised for lost / stale / exception parcels. |
| Status becomes **Completed** | **Reshipment Completed Date** is stamped. |
| Any important change | A line is added to the **timeline of the original Lyfe Order** (see 3.5). |

### 3.5 How a Reshipment affects the original Lyfe Order

| Reshipment event | Lyfe Order status becomes | Other effects |
|---|---|---|
| Moves to **Reshipment In Progress** | **Reshipment In Progress** | — |
| Moves to **Ready for Dispatch** | (no change) | Order's Reship Status = **PENDING**, Reship Date set |
| Moves to **Shipped** | (no change) | Reship Status = **ON GOING** |
| Moves to **Completed** | **Completed** | Reship Status = **SUCCESSFUL** |
| **Cancelled** | **Completed** (the original order was already delivered) | — |

Exception: if a **Return on the same order is still open** (Requested / In Transit / Received), a Completed or Cancelled reshipment leaves the order as **Return InProgress** until that Return is closed or cancelled.

**Timeline:** each creation and change (status, Return link, reason, warehouse, carrier, tracking numbers, costs, Gate Pass, invoice) is written to the Lyfe Order's timeline, so Customer Service can follow the reshipment without opening it.

📷 **[SCREENSHOT RS-05]** — The Lyfe Order timeline showing reshipment activity lines.

### 3.6 Reshipment and Return

- **Create a Reshipment from a Return:** on a Lyfe Order Return, click **Create → Reshipment**. It opens a new reshipment, pre-filled from the order, with the **Return** field set and the reason "Customer Return Replacement". This works for **any Return that is not Cancelled**, even after it is Closed.
- The Return follows its reshipments:
  - **Reship Status** on the Return: *PENDING* (not started), *ON GOING* (shipping), *SUCCESSFUL* (all done).
  - A Return can have **several** reshipments (for example, the first got lost).
  - A **Refund-only** Return that gets a reshipment becomes **Refund & Reship** (while it is still open), so it can only be closed once the reshipment is done.
- A reshipment created directly from the order is a **"Reshipment only"** (no customer return behind it). You can still attach it to a Return later in the **Return** field.
- Returns Dashboard counts: **Return only**, **Return + Reshipment**, **Reshipment only**, **Reshipments from returns**.

### 3.7 Costs

| Cost | Covers | How it is filled |
|---|---|---|
| **Reshipment COGS** | Cost of the goods sent | Automatic |
| **Shipment Cost** | Shipping to the customer (direct reshipment) or Factory → US (via-US reshipment, its share of the batch) | Typed (direct) / automatic from the Bulk Transfer Batch (via-US) |
| **Customer Leg Shipment Cost** | US Warehouse → Customer (via-US only) | Typed by hand |
| **Custom Charges / Additional Charges** | Customs and other charges, per leg | Typed (via-US batch shares are automatic) |

Reshipment cost appears as its own line in the **PnL dashboard**, the Founder dashboard and the Status Overview. Users who may not see costs do not see these columns.

### 3.8 Rules and error messages

| Message / situation | Why | What to do |
|---|---|---|
| "Reshipment … has no submitted **Material Issue for Order**" | You moved to **Ready for Dispatch** without issuing material for this reshipment | Create **Create → Material Issue for Order**, submit it, then retry |
| "… is a split order. Raise the Reshipment against the **main order**" | Reshipments are not created on split child orders | Open the main (parent) order and create it there |
| "Return … does not belong to Lyfe Order …" | The **Return** field points at another order's Return | Pick a Return of this order |
| "Return … is Cancelled" | You linked a cancelled Return | Pick another Return or leave blank |
| **Via US Warehouse** is read-only | The reshipment is already in a Bulk Transfer Batch | Ask Factory to remove it from the draft batch first |
| Cannot cancel | Only **System Manager** can cancel; and a via-US reshipment cannot be cancelled after its batch is submitted | Ask a System Manager |
| "Missing Fields in CJ Shipping Tab" | A required CJ field is empty | The message lists exactly which fields |
| Create → Reshipment button not visible | The order is not **Completed** or **Shipped**, or it is a split child, or you are not Customer Service / System Manager | Use **Create → Reshipment from the Return**, or ask Customer Service |

---

## 4. How to use the feature — step by step

### 4.1 Create a Reshipment from a Lyfe Order

1. Open the **Lyfe Order** that needs a reshipment. (The order must be **Completed** or **Shipped**.)
2. Click **Create** in the toolbar, then **Reshipment**.

   📷 **[SCREENSHOT RS-06]** — Lyfe Order toolbar with the **Create** menu open and **Reshipment** highlighted.

3. A new reshipment opens with the **Lyfe Order** filled in and every order item copied into **Reshipment Items**.

   📷 **[SCREENSHOT RS-07]** — The new, unsaved reshipment with the items pre-filled.

### 4.2 Create a Reshipment from a Return

1. Open the **Lyfe Order Return**.
2. Click **Create → Reshipment**.

   📷 **[SCREENSHOT RS-08]** — The Return form with **Create → Reshipment**.

3. The reshipment opens pre-filled, with **Return** set and the reason "Customer Return Replacement". Continue from step 4.3.

### 4.3 Review the items, reason and warehouse

1. In **Reshipment Items**, delete rows or lower quantities so only what must be re-sent remains.
2. Choose the **Reason** (required) and write a **Description** if useful.
3. Choose the **Warehouse**. Pick **Factory - LH** to use the CJ Shipping tab, Gate Pass and invoice.
4. If the parcel must go through the US warehouse, tick **Via US Warehouse** (see Use case 5).
5. **Save**.

   📷 **[SCREENSHOT RS-09]** — Reason, Description, Warehouse and Via US Warehouse fields.

### 4.4 Start the work (Factory)

The Factory user clicks **Start Progress**. The Lyfe Order status changes to **Reshipment In Progress**.

### 4.5 Issue material (Factory)

1. On the reshipment click **Create → Material Issue for Order**.
2. Complete and **submit** the Material Issue. This is **mandatory** before **Ready for Dispatch**.
3. Submitting it fills **Shipment Items** in the CJ Shipping tab.

   📷 **[SCREENSHOT RS-10]** — The Material Issue for Order opened from the reshipment.

### 4.6 Optional quality review

Factory clicks **Internal Review**. Customer Service clicks **Approve** (→ Approved) or **Reject** (→ Rework Required; Factory fixes it and sends it back to Internal Review).

### 4.7 Create the CJ label (Factory warehouse only)

1. Open the **CJ Shipping** tab and complete all sections (see 3.2).
2. Click **CJ Shipping → Generate CJ Label**.

   📷 **[SCREENSHOT RS-11]** — **CJ Shipping → Generate CJ Label**.

3. If something is missing, a message lists it. When it succeeds, **AWB No.**, **CJ Tracking No.** and **Shipping Label** are filled in.

   📷 **[SCREENSHOT RS-12]** — "Missing Fields in CJ Shipping Tab" message and the success message.

### 4.8 Ready for Dispatch → Dispatched (Factory)

1. Click **Ready for Dispatch**. A **Gate Pass** and a draft **Sales Invoice** are created automatically and linked in **References**.

   📷 **[SCREENSHOT RS-13]** — "Gate Pass Created" and "Sales Invoice Created" alerts, and the References section.

2. Hand the parcel to the carrier, then click **Dispatched**.
3. If no invoice was created, use **Create → Sales Invoice**.

### 4.9 Tracking

1. Enter **Carrier** and **Tracking Number** (Shipping section).
2. From now on **do nothing** — the hourly job fills **Tracking Status** and **Tracking Details** and moves the reshipment to **Awaiting Shipping → Shipped → Completed** by itself.

   📷 **[SCREENSHOT RS-14]** — Tracking fields filled by the system.

3. If **Tracking Alert** shows a value (for example *Possible Lost*), contact the carrier; Customer Service is notified by Slack/email.

### 4.10 Completion

When the carrier reports delivery the reshipment becomes **Completed**, the original Lyfe Order becomes **Completed** and its Reship Status shows **SUCCESSFUL**. If it was created from a Return, the Return's Reship Status also becomes **SUCCESSFUL**.

### 4.11 Cancel a reshipment

Only a **System Manager** can cancel (workflow button **Cancel**), before the parcel is shipped. The original order returns to **Completed** (or stays **Return InProgress** if a Return is open).

---

## 5. Use cases — step by step

### Use case 1 — Carrier lost the parcel (direct from Factory)

**Situation:** Order `LH2946` was shipped; the carrier reports the parcel lost.

1. Customer Service opens `LH2946` → **Create → Reshipment**.
2. Reason = **Carrier Lost**. Keep all items. Warehouse = **Factory - LH**. Save.
3. Factory clicks **Start Progress** (order becomes *Reshipment In Progress*).
4. Factory creates and submits a **Material Issue for Order** from the reshipment.
5. Factory fills the **CJ Shipping** tab and clicks **Generate CJ Label**.
6. Factory clicks **Ready for Dispatch** (Gate Pass + invoice appear) and then **Dispatched**.
7. Carrier and tracking number are entered. The system tracks it to **Shipped** and **Completed**.
8. The order shows **Completed**; Reship Status = SUCCESSFUL.

📷 **[SCREENSHOT RS-15]** — The finished reshipment `LH2946-R01` and its timeline entries on the order.

### Use case 2 — Wrong item sent (partial reshipment)

1. Customer Service creates the reshipment from the order. Reason = **Wrong Item Sent**.
2. In **Reshipment Items**, delete every row except the one wrong item (correct the quantity if needed).
3. Follow steps 3–8 of Use case 1. Reshipment COGS only counts the items that were re-sent.

### Use case 3 — Customer returns goods and wants a replacement

1. Customer Service creates a **Return** on the order (Return Type **Reship** or **Refund & Reship**).
2. When the goods are received, the Return reaches **Return Received** and a follow-up task is created for Customer Service (and the Factory user, if the goods went to the factory).
3. On the Return, click **Create → Reshipment**. The reshipment opens pre-filled and linked to the Return.
4. Run the reshipment as in Use case 1.
5. When the reshipment is **Completed**, the Return's Reship Status becomes **SUCCESSFUL**. Finish the refund details on the Return and click **Close Return**.
6. Final order status: **Completed** (the reshipment decides the final status).

📷 **[SCREENSHOT RS-16]** — A Return with its linked Reshipment in the Connections tab.

### Use case 4 — Refund-only return where the customer later asks for a replacement

1. Open the (still open or already closed) Return → **Create → Reshipment**.
2. While the Return is still open its type changes to **Refund & Reship**, so it cannot be closed before the reshipment is done.
3. Continue as in Use case 3.

### Use case 5 — Reshipment through the US Warehouse

1. Create the reshipment and **tick Via US Warehouse** (do this before it goes into a batch).
2. Factory completes **Start Progress → material issue → Ready for Dispatch → Dispatched**. The reshipment must be on a **submitted Gate Pass**.
3. Factory opens a **Bulk Transfer Batch** → **Update Orders → From Reshipment Order**, pastes the reshipment IDs (one per line) and clicks **Update**.

   📷 **[SCREENSHOT RS-17]** — The Bulk Transfer Batch "Update Orders" dialog.

4. Factory enters the batch tracking number and carrier and **submits** the batch. The number is copied to each reshipment as **Tracking #1**. After this the reshipment can no longer be cancelled and Via US Warehouse is locked.
5. The system tracks Tracking #1 hourly: **In Transit to US Warehouse** → **US Warehouse Delivered**. If tracking is late, click **1Click → Mark Delivered to US Warehouse**.
6. Customer Service clicks **1Click → Post to 1Click** (always manual). Status becomes **Submitted to 1Click**. On a rejection the status is **1Click Error** with the reason shown; fix it and click **Post to 1Click** again.
7. 1Click ships to the customer; the customer's tracking number and carrier appear automatically. The status moves to **Awaiting Shipping → Shipped → Completed**.
8. Finance enters the batch charges (even days later); each reshipment's **Shipment Cost** share updates automatically. **Customer Leg Shipment Cost** (US → customer) is typed by hand.

   📷 **[SCREENSHOT RS-18]** — The **1Click** button group on the reshipment.

### Use case 6 — 1Click rejected the reshipment

1. The reshipment shows **1Click Error**; the reason is in the **1Click Logistics** tab and a Slack alert was sent.
2. Fix the cause (for example a missing address or item).
3. Click **1Click → Post to 1Click** again.

### Use case 7 — Package reached the US warehouse but status did not change

1. Open the reshipment and click **1Click → Mark Delivered to US Warehouse**, confirm.
2. Then continue with **Post to 1Click** (Use case 5, step 6).

### Use case 8 — Reshipment must be cancelled

1. Ask a **System Manager** to open the reshipment and click **Cancel** (not possible once the via-US batch is submitted).
2. The original order goes back to **Completed** (or **Return InProgress** if a Return is still open).
3. Cancelled reshipments are not counted in the Returns Dashboard.

### Use case 9 — Second reshipment for the same order

Create **Create → Reshipment** again. It becomes `LH2946-R02`. Each reshipment is independent (own items, tracking, cost).

---

## 6. Where to see reshipments

| Place | What you see |
|---|---|
| **Lyfe Order → Links → Reshipments** (button) and the **Reshipment** group in Connections | All reshipments of that order |
| **Lyfe Order timeline** | Every reshipment creation and change |
| **Lyfe Orders — Status Overview** | **Active Reshipment** card (number of reshipments still open — not Completed, Cancelled or Shipped; click for details) and the collapsible **↩ Reshipments** panel with a state filter (refreshes every 15 seconds) |
| **Returns Dashboard** | Counts: Return only, Return + Reshipment, Reshipment only, Reshipments from returns, Awaiting Reshipment |
| **Lyfe Order Reshipment list** | Filter by Return (`is set` / `is not set`), state, order, reason |
| **PnL dashboard / Founder dashboard** | Reshipment cost as a separate cost line |

📷 **[SCREENSHOT RS-19]** — Status Overview with the **Active Reshipment** card and the **↩ Reshipments** panel.

---

## 7. FAQ and troubleshooting

**The Create → Reshipment button is not on my order.**
The order must be *Completed* or *Shipped*, must not be a split child, and you must be Customer Service or System Manager. If the order is in *Return Successfully* or another state, create the reshipment from the **Return** instead.

**The CJ Shipping tab is missing.**
Set **Warehouse** to exactly **Factory - LH**.

**I cannot move to Ready for Dispatch.**
A submitted **Material Issue for Order** is required for this reshipment (see 4.5).

**The reshipment changed status by itself.**
That is the hourly tracking job (Delivered → Completed, label only → Awaiting Shipping, moving → Shipped).

**Tracking Status / Tracking Details are empty.**
They fill in automatically on the next hourly run after a tracking number and carrier are saved.

**I cannot cancel.**
Only a System Manager can. Via-US reshipments cannot be cancelled after their batch is submitted.

**Can I change Via US Warehouse?**
Only until the reshipment is added to a Bulk Transfer Batch.

**The order status shows "Return InProgress" after my reshipment completed.**
A Return on that order is still open. The order will update when that Return is closed or cancelled.

**The Reshipment Request state shows on my reshipment but nothing happens.**
Factory must click **Start Progress**; nothing moves it forward automatically at this step.

📷 **[SCREENSHOT RS-20]** — A completed Reshipment for reference, fully filled in.

---

## 8. Glossary

| Term | Meaning |
|---|---|
| **Lyfe Order** | The customer's original order. |
| **Reshipment** | A new shipment for an order that already shipped. |
| **Return** | Goods coming back from the customer. |
| **MIFO** | Material Issue for Order — the document that issues stock for the shipment. |
| **Gate Pass** | Dispatch document for parcels leaving the factory. |
| **CJ label** | International shipping label (FedEx via CJ) created from the CJ Shipping tab. |
| **Via US Warehouse** | Route Factory → 1Click US Warehouse → Customer, with two tracking numbers. |
| **1Click** | The US warehouse partner. |
| **Bulk Transfer Batch** | A group of orders/reshipments sent to the US warehouse together. |
| **Tracking #1 / #2** | Factory → US leg / US → customer leg. |
| **Reship Status** | PENDING / ON GOING / SUCCESSFUL / NOT APPLICABLE / UNSUCCESSFUL, shown on the Lyfe Order and on the Return. |

---

## 9. Document change log

> **Maintainer rule:** this is the single end-user document for the Reshipment feature. Any change to the Reshipment doctype, its workflow, buttons, automatic behaviour, messages, the Return link, the Status Overview reshipment card/panel, or the reshipment counts on the Returns Dashboard must be reflected here **in the same change**, with a new row below and an updated *Last updated* date at the top. If the document and the system disagree, the system is right — fix the document.

| Version | Date | What changed |
|---|---|---|
| 1.0 | 2026-10-12 | First end-user guide: full feature details, how-to steps, use cases. Replaces the older `docs/reshipment_user_guide.md` (which described Customer Service as able to cancel and said tracking never moves the status — both are no longer true). |
