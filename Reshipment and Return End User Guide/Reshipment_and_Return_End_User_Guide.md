# Reshipment and Return — End User Guide

| | |
|---|---|
| **Features** | **Lyfe Order Return** (goods coming back from the customer: refund, restock fee, replacement) and **Lyfe Order Reshipment** (a second shipment to the customer, including the Return link and the Via-US-Warehouse route) |
| **Who it is for** | Customer Service, Factory, Shipping Forwarder, Finance, Super Admin, System Manager |
| **Document version** | 2.0 |
| **Last updated** | 2026-10-12 |
| **Screenshots** | Every `📷 [SCREENSHOT RS-xx]` (Reshipment) or `📷 [SCREENSHOT RT-xx]` (Return) line is a placeholder. Take the screenshot, save it as the same ID (for example `RT-01.png`) and insert it there. |

---

## Contents

1. [What are a Reshipment and a Return?](#1-what-are-a-reshipment-and-a-return)
2. [Who can do what](#2-who-can-do-what)

**PART A — RESHIPMENT**

3. [Reshipment — all the details](#3-reshipment--all-the-details-of-the-feature)
4. [Reshipment — how to use it, step by step](#4-reshipment--how-to-use-the-feature-step-by-step)
5. [Reshipment — use cases](#5-reshipment--use-cases-step-by-step)

**PART B — RETURN**

6. [Return — all the details](#6-return--all-the-details-of-the-feature)
7. [Return — how to use it, step by step](#7-return--how-to-use-the-feature-step-by-step)
8. [Return — use cases](#8-return--use-cases-step-by-step)

**BOTH**

9. [Where to see returns and reshipments](#9-where-to-see-returns-and-reshipments)
10. [FAQ and troubleshooting](#10-faq-and-troubleshooting)
11. [Glossary](#11-glossary)
12. [Document change log](#12-document-change-log)

---

## 1. What are a Reshipment and a Return?

**In short:** a **Return** is goods coming **back** from the customer (and the refund or replacement that follows). A **Reshipment** is goods going **out again** to the customer. A customer who sends goods back and wants a replacement needs **both**, linked to each other. Part A explains the Reshipment, Part B explains the Return.

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

How the two link together is explained in [3.6](#36-reshipment-and-return) and in Return use case 3 in Part B.

📷 **[SCREENSHOT RS-01]** — A finished Reshipment form (top part): Lyfe Order, Reason, Workflow State and the Reshipment Items table.

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

**Return**

| Role | Create | Edit | Workflow steps | Cancel |
|---|:---:|:---:|---|:---:|
| **Customer Service** | ✅ | ✅ | Mark In Transit, Mark Received, Close Return | ❌ |
| **Factory** | ❌ | ✅ | Mark Received | ❌ |
| **System Manager** | ✅ | ✅ | Any | ✅ **(only role that can cancel)** |

> The **Create → Return** button on a Lyfe Order is shown only to **Customer Service** and **System Manager**.

---

**PART A — RESHIPMENT**

## 3. Reshipment — all the details of the feature

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

## 4. Reshipment — how to use the feature (step by step)

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

## 5. Reshipment — use cases (step by step)

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

**PART B — RETURN**

## 6. Return — all the details of the feature

A **Lyfe Order Return** is created for an order when the customer sends goods back, or when we refund the customer (with or without taking the goods back). It tracks the returned goods, the refund, the restock fee and any replacement. The details are kept in the Return itself — not on the Lyfe Order.

### 6.1 Naming

Every return is named **`{Order ID}-RT-{number}`**, counting per order. For order `LH2058`: `LH2058-RT-01`, then `LH2058-RT-02`, and so on. An order can have several returns over time, but only **one open return at a time**.

### 6.2 When you can create a Return

- The order must be **Completed**, **Shipped**, **Reshipment In Progress** or **Return Successfully**.
- It must **not** be a **split child** order (raise the return on the main order) and must **not** be a **merged** order (raise it on the order it was merged into).
- The order must not already have an **open** return (Return Requested, Return In Transit or Return Received).
- The **Create → Return** button is shown only to Customer Service and System Manager, and the system checks the same rules again when you save.

### 6.3 The form — every section and field

📷 **[SCREENSHOT RT-01]** — A Return form (top part): Lyfe Order, Return Type, Reason, dates and the Return Items table.

| Field | What it is |
|---|---|
| **Lyfe Order** | The order being returned. Filled automatically, read-only. |
| **Customer** | Taken from the order. |
| **Workflow State** | Current status (see 6.4). Read-only — change it with the buttons at the top. |
| **Return Type** | **Refund** (money back), **Reship** (replacement only), or **Refund & Reship** (both). |
| **Return Reason** | **Preference**, **Damaged**, **Exchange**, **Wrong Item Received** or **Others**. Choosing **Others** makes the **Other** text box required. |
| **Is RTO** | Tick if the courier returned the parcel to us (return to origin). |
| **No Physical Return** | Tick if the customer keeps the goods and only a refund is given. The In Transit step is skipped. |
| **Return Initiated Date** | Defaults to today. |
| **Return Received Date** | Filled automatically when the goods are received. Read-only. |
| **Return To Warehouse** | **Factory - LH** or **US Warehouse - LH** — where the goods are coming back to. |

**Return Items**

| Column | What it is |
|---|---|
| Item Name, SKU | From the order. |
| Ordered Qty | Quantity on the order. |
| **Return Qty** | How many come back. Defaults to the full quantity; **lower it for a partial return**. It cannot be more than the ordered quantity. At least one row must have a quantity above 0. |
| Unit Price | From the order. |

**Refund**

| Field | What it is |
|---|---|
| **Refund Status** | PENDING / ON GOING / **SUCCESSFUL** / NOT APPLICABLE / UNSUCCESSFUL. |
| **Refund Amount**, **Refund Date** | **Both required** when Refund Status is **SUCCESSFUL**. |
| **Restock Fee %** | Percentage kept as a restock fee, if any. |
| **Restock Fee** | Calculated automatically: order total × Restock Fee %. Read-only. |

**Reship**

| Field | What it is |
|---|---|
| **Reship Status** | PENDING / ON GOING / **SUCCESSFUL** / NOT APPLICABLE / UNSUCCESSFUL. Follows the linked Reshipment(s) automatically. |
| **Reship Date** | Set automatically when the replacement is completed. |
| **Reshipments** | Number of reshipments linked to this return. |

**Return Tracking tab**

| Field | What it is |
|---|---|
| **Return Tracking Number**, **Return Carrier** | Entered when the customer ships the goods back. Saving **both** moves the return to *Return In Transit* by itself. |
| **Track Status**, **Tracking Details** | Filled automatically by the hourly tracking job. Read-only. |
| **Tracking Alert** (+ reason, since, last event) | Set automatically when the carrier never scans the parcel, tracking goes stale, an exception is reported, or the parcel looks lost. |

**Connections tab:** shows the **Reshipments** linked to this return.

### 6.4 The workflow (statuses)

```
Return Requested ──(Customer Service: Mark In Transit)──► Return In Transit
      │                                                         │
      │                                              (carrier says Delivered — automatic)
      │                                                         ▼
      └──(No Physical Return: automatic)──────────────► Return Received
                                                                │ (Customer Service: Close Return)
                                                                ▼
                                                             Closed
   Cancelled — System Manager only, from Return Requested or Return In Transit
```

📷 **[SCREENSHOT RT-02]** — The workflow buttons at the top of a Return (for example "Mark Received").

| Status | Meaning | Moved by |
|---|---|---|
| **Return Requested** | The return is opened; goods not on their way yet. | Created by Customer Service |
| **Return In Transit** | The customer has shipped the goods back. | **Automatic** when tracking number and carrier are saved (or *Mark In Transit*) |
| **Return Received** | The goods are back with us. | **Automatic** when the carrier reports Delivered, or *Mark Received* by Customer Service / Factory |
| **Closed** | Everything is settled: refund and/or replacement done. | **Manual** — Customer Service clicks *Close Return* |
| **Cancelled** | The return did not go ahead. | System Manager only |

> **Closing is always manual.** The system never closes a return by itself.

**What you must complete before *Close Return* works**

| Check | Required when |
|---|---|
| **Return Received Date** is set | Always, unless **No Physical Return** is ticked |
| **Refund Status** is SUCCESSFUL or NOT APPLICABLE | Return Type is **Refund** or **Refund & Reship** |
| **Reship Status** is SUCCESSFUL or NOT APPLICABLE | Return Type is **Reship** or **Refund & Reship** |

### 6.5 What the system does for you automatically

| When | What happens |
|---|---|
| A Return is created | A line is added to the order's return history. The order's status becomes **Return InProgress**. |
| Tracking number **and** carrier are saved | The return moves to **Return In Transit**. |
| Every hour | Return tracking is checked. On "Delivered" the return moves to **Return Received** and the received date is stamped. Alerts are sent for lost / stale / exception parcels. |
| **No Physical Return** is ticked | The return goes straight to **Return Received**. |
| The return reaches **Return Received** | A **follow-up Task** is created (see 6.6). |
| A linked Reshipment changes status | The return's **Reship Status** and **Reship Date** update by themselves. |
| Any important change | A line is written to the **timeline of the Lyfe Order** (status, type, reason, dates, refund, restock fee, reship status, tracking, items). |
| The return is **Closed** | The follow-up Task is marked **Completed**. |

### 6.6 The follow-up Task (when goods arrive)

When a return reaches **Return Received**, a **Task** is created in the **CS Ticket** project:

- **Assigned to** Customer Service (`support@lyfehardware.com`), and also to the **Factory user** if the goods were returned to **Factory - LH**.
- **Subject:** "Return received — {Return} ({Order}): update refund/restock & close".
- **Contents:** links to the Return and Lyfe Order, customer, return type, reason, initiated and received dates, warehouse, tracking, refund and reship status, restock fee, the return items (returned / ordered) and a **"Still to update"** checklist (refund, reship, restock fee, then *Close Return*).
- One task per return. It is marked **Completed** automatically when the return is **Closed**.
- A comment on the Lyfe Order timeline links to the task.

📷 **[SCREENSHOT RT-03]** — The follow-up Task with the return details and the "Still to update" list.

### 6.7 How a Return affects the original Lyfe Order

| Return event | Lyfe Order status becomes |
|---|---|
| Created / In Transit / Received | **Return InProgress** |
| **Closed** | **Return Successfully** — or **Completed** if the return had reshipments (the reshipment decides the final status) |
| **Cancelled** | **Completed** |

- While a **Reshipment is in progress** (order status *Reshipment In Progress*), the Return does not change the order's status.
- When a Return is **cancelled**, its line is removed from the order's return history and the order's old return fields are cleared (or set back to the previous return, if there is one), so reports do not count a return that never happened.
- The order's older return fields (return type, refund status and amount, dates, tracking) are kept in step with the latest active Return so existing dashboards (customer intelligence, founder dashboard) keep working.

### 6.8 Return and Reshipment

- On a Return, click **Create → Reshipment** to create the replacement. It works for **any Return that is not Cancelled**, even after it is closed. See [3.6](#36-reshipment-and-return).
- A **Refund-only** return that gets a reshipment becomes **Refund & Reship** (while it is still open), so it cannot be closed before the replacement is done.
- A Return can have several Reshipments. The Return's **Reship Status** is **SUCCESSFUL** only when all of its (non-cancelled) reshipments are Completed.

### 6.9 Rules and error messages

| Message / situation | Why | What to do |
|---|---|---|
| "A Return can only be created for an order that is Completed, Shipped, Reshipment In Progress, Return Successfully" | The order is in another status (New, Cancelled, Awaiting Shipping …) | Wait until the order has shipped |
| "… is a split order. Raise the Return against the main order …" | You tried a split child order | Open the main order |
| "… was merged into …" | You tried a merged order | Open the order it was merged into |
| "… is still open for this order. Close or cancel it before creating another return." | Another return is open | Finish or cancel it first |
| "Please describe the return reason when the category is Others." | Reason = Others with no text | Fill the **Other** box |
| "Return Qty cannot exceed Ordered Qty" / "At least one return item needs a Return Qty greater than 0" | Quantities | Correct the item rows |
| "Refund Amount and Refund Date are required when Refund Status is SUCCESSFUL." | Incomplete refund | Fill both |
| "Mark the return as Received before closing it." | Closing too early | Mark it Received first (or tick No Physical Return) |
| "Refund Status / Reship Status must be SUCCESSFUL or NOT APPLICABLE before closing." | Outcome not finished | Complete or mark it NOT APPLICABLE |
| Cannot save a change on an **old imported return** | Returns imported from before this feature have no item rows, and a return must have at least one item | Ask System Manager / Admin to correct it |

---

## 7. Return — how to use the feature (step by step)

### 7.1 Create a Return

1. Open the **Lyfe Order** the customer is returning.
2. Click **Create → Return**.

   📷 **[SCREENSHOT RT-04]** — Lyfe Order toolbar: **Create → Return**.

3. The Return opens with the order and **all items** filled in, and the Return Initiated Date set to today.

   📷 **[SCREENSHOT RT-05]** — The new, unsaved Return with the items pre-filled.

4. Choose the **Return Type** (Refund / Reship / Refund & Reship) and the **Return Reason**. Tick **Is RTO** or **No Physical Return** if they apply.
5. For a **partial return**, lower the **Return Qty** of the items (delete nothing — just change the quantity).
6. Choose **Return To Warehouse**.
7. **Save.** The order now shows **Return InProgress**.

### 7.2 Enter the return tracking

1. When the customer ships the goods back, open the Return's **Return Tracking** tab.
2. Enter the **Return Tracking Number** and **Return Carrier** and save.
3. The Return moves to **Return In Transit** by itself. You can click **Track Return** to refresh the carrier status at any time.

   📷 **[SCREENSHOT RT-06]** — The Return Tracking tab and the **Track Return** button.

### 7.3 When the goods arrive

- The hourly job moves the Return to **Return Received** when the carrier reports Delivered. If the goods arrive without tracking, click **Mark Received**.
- A **follow-up Task** appears for Customer Service (and the Factory user). Open it from your task list — it lists everything still to update.

### 7.4 Update the details (Customer Service)

1. **Refund:** set **Refund Status**, **Refund Amount** and **Refund Date**. For a refund that is done choose **SUCCESSFUL**; if no refund is given choose **NOT APPLICABLE**.
2. **Restock fee:** enter **Restock Fee %** if a fee applies — the amount is calculated.
3. **Replacement:** if the customer needs a replacement, click **Create → Reshipment**, complete the Reshipment (Part A) and wait until it is Completed. If no replacement is needed, set **Reship Status** to **NOT APPLICABLE**.
4. Save.

   📷 **[SCREENSHOT RT-07]** — The Refund and Reship sections of a Return in *Return Received* state.

### 7.5 Close the Return

Click **Close Return**. If something is missing, the message tells you what (see 6.9). On success the Return becomes **Closed**, the follow-up Task completes, and the order shows **Return Successfully** (or **Completed** if there was a reshipment).

### 7.6 Cancel a Return

Only a **System Manager** can cancel, and only from **Return Requested** or **Return In Transit**. The order goes back to **Completed** and the return disappears from the order's return history.

---

## 8. Return — use cases (step by step)

### Return use case 1 — Customer returns the goods for a full refund

**Situation:** Order `LH3045` was delivered; the product is damaged and the customer wants a refund.

1. Customer Service opens `LH3045` → **Create → Return**.
2. Return Type = **Refund**, Reason = **Damaged**, Return To = **Factory - LH**. Save → `LH3045-RT-01`, status **Return Requested**. The order shows **Return InProgress**.
3. The customer ships the goods back. Customer Service enters the **Return Tracking Number** and **Return Carrier** and saves → **Return In Transit**.
4. The carrier delivers it. The hourly job sets **Return Received**; a follow-up Task is created for Customer Service and the Factory user.
5. Customer Service gives the refund, then sets **Refund Status = SUCCESSFUL**, **Refund Amount** and **Refund Date**. If a restock fee applies, enters **Restock Fee %**.
6. Customer Service clicks **Close Return** (for a Refund-only return nothing else is needed). The task completes and the order shows **Return Successfully**.

📷 **[SCREENSHOT RT-08]** — The closed Return `LH3045-RT-01` and its timeline entries on the Lyfe Order.

### Return use case 2 — Partial return

1. Create the Return as above.
2. In **Return Items**, lower the **Return Qty** for the items that stay with the customer (for example 2 of 5 coming back).
3. Continue as in Return use case 1. The Refund Amount is entered by hand for the returned part only.

### Return use case 3 — Replacement return ("Reship")

1. Create the Return with Return Type **Reship** (or **Refund & Reship** if money is also refunded).
2. Follow the tracking steps until **Return Received**.
3. On the Return click **Create → Reshipment**. The reshipment opens pre-filled and linked to the Return (reason "Customer Return Replacement"). Complete it as in Reshipment use case 1.
4. When the Reshipment is **Completed**, the Return's **Reship Status** becomes **SUCCESSFUL** by itself.
5. Complete the refund details if the type was Refund & Reship, then click **Close Return**.
6. Final order status: **Completed**.

### Return use case 4 — Refund only; the customer keeps the goods (No Physical Return)

1. Create the Return with Return Type **Refund** and tick **No Physical Return**.
2. After you save, the Return goes directly to **Return Received** (no tracking needed) and the follow-up Task is created.
3. Set **Refund Status = SUCCESSFUL**, **Refund Amount**, **Refund Date** and click **Close Return**.

### Return use case 5 — Courier returned the parcel (RTO)

1. Create the Return and tick **Is RTO**. Return Type is usually **Reship** (send it again) or **Refund**.
2. Tracking is optional. When the goods are back, mark the Return **Received** (automatic if tracked).
3. Continue with the refund or **Create → Reshipment**, then **Close Return**.

### Return use case 6 — A second return on the same order

1. After the first return is **Closed** (the order shows Return Successfully), **Create → Return** is available again.
2. The new return is `…-RT-02`. If the first return is still open you must close or cancel it first.

### Return use case 7 — Refund-only return that later needs a replacement

1. Open the return (open or already closed) → **Create → Reshipment**.
2. While the Return is open its type changes to **Refund & Reship**, so it can only be closed once the reshipment is done.

### Return use case 8 — Cancelling a return

1. A **System Manager** opens the Return (Return Requested or Return In Transit) and clicks **Cancel**.
2. The order goes back to **Completed**; the return is removed from the order's return history and from the customer's return count.

---

**BOTH**

## 9. Where to see returns and reshipments

| Place | What you see |
|---|---|
| **Lyfe Order → Links → Returns / Reshipments** (buttons) and the **Return** and **Reshipment** groups in Connections | All returns and reshipments of that order |
| **Lyfe Order timeline** | Every return and reshipment creation and change |
| **Lyfe Order Return list** | All returns. Filter by state, type, reason, order, customer, or *Reshipments* (0 = return only) |
| **Lyfe Order Reshipment list** | Filter by Return (`is set` / `is not set`), state, order, reason |
| **Returns Dashboard** | Open it from the search bar or from the Status Overview page. Cards: **Total Returns**, **Pending Refunds**, **Pending Reshipping**, **Avg Cycle Time**, **Return Only** (no reshipment), **Return + Reshipment**, **Reshipment Only** (no return), **Reshipments From Returns**, **Awaiting Reshipment**; filters for date, type and status; charts by type and by week; recent returns table. Click a card to open the matching list. (Set a wide **From Date** to include older returns.) |
| **Lyfe Orders — Status Overview** | **Active Returns** card (number of returns still open — Requested, In Transit or Received) — click it for a table with return, order, customer, state, type, reason, initiated date and days open, tracking/carrier, received date, return-to warehouse, refund and reship status; buttons to open the **Returns Dashboard** and the returns list. Also the **Active Reshipment** card (open reshipments — not Completed, Cancelled or Shipped) and the collapsible **↩ Reshipments** panel with a state filter. The page refreshes every 15 seconds. |
| **PnL dashboard / Founder dashboard** | Reshipment cost as a separate cost line |

📷 **[SCREENSHOT RT-09]** — Returns Dashboard cards.

📷 **[SCREENSHOT RT-10]** — Status Overview with the **Active Returns** card and its details dialog.

📷 **[SCREENSHOT RS-19]** — Status Overview with the **Active Reshipment** card and the **↩ Reshipments** panel.

---

## 10. FAQ and troubleshooting

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

**Return questions**

**The Create → Return button is not on my order.**
The order must be *Completed*, *Shipped*, *Reshipment In Progress* or *Return Successfully*, must not be a split child or merged order, and you must be Customer Service or System Manager. If another return is still open, finish or cancel it first.

**I entered the tracking number but the return is still "Return Requested".**
Both **Return Tracking Number** and **Return Carrier** must be filled and saved. Then it moves to *Return In Transit* by itself.

**The goods arrived but the return is still "Return In Transit".**
The hourly job moves it when the carrier reports Delivered. If there is no tracking, click **Mark Received**.

**I cannot click Close Return / it shows an error.**
The message lists what is missing: received date, a finished refund (SUCCESSFUL or NOT APPLICABLE) and a finished reship (SUCCESSFUL or NOT APPLICABLE) when the type includes them.

**Nobody told me the goods arrived.**
A follow-up **Task** is created in the CS Ticket project when the return reaches *Return Received*. Check your tasks (or the order's timeline for the link).

**The order still says "Return InProgress".**
The return has not been closed or cancelled yet, or a reshipment is still active.

**The order says "Return Successfully", can I raise another return?**
Yes — **Create → Return** is available again.

**I cannot edit an old return.**
Returns imported from before this feature have no item rows and were imported as *Closed*; the system requires at least one item row to save. Ask a System Manager.

**A cancelled return still shows in some report.**
Cancelling removes it from the order's return history and the customer's return count; the new Returns Dashboard never counts cancelled returns.

📷 **[SCREENSHOT RS-20]** — A completed Reshipment for reference, fully filled in.

---

## 11. Glossary

| Term | Meaning |
|---|---|
| **Lyfe Order** | The customer's original order. |
| **Reshipment** | A new shipment for an order that already shipped. |
| **Return** | Goods coming back from the customer (document: Lyfe Order Return, named `…-RT-01`). |
| **RTO** | Return to origin — the courier sent the parcel back to us. |
| **No Physical Return** | A refund where the customer keeps the goods; skips the In Transit step. |
| **Restock Fee** | A fee kept from the refund, calculated as the order total × Restock Fee %. |
| **Follow-up Task** | The task created in the CS Ticket project when a return arrives. |
| **Return only / Return + Reshipment / Reshipment only** | Dashboard groups: a return with no reshipment / a return that has a reshipment / a reshipment with no return behind it. |
| **MIFO** | Material Issue for Order — the document that issues stock for the shipment. |
| **Gate Pass** | Dispatch document for parcels leaving the factory. |
| **CJ label** | International shipping label (FedEx via CJ) created from the CJ Shipping tab. |
| **Via US Warehouse** | Route Factory → 1Click US Warehouse → Customer, with two tracking numbers. |
| **1Click** | The US warehouse partner. |
| **Bulk Transfer Batch** | A group of orders/reshipments sent to the US warehouse together. |
| **Tracking #1 / #2** | Factory → US leg / US → customer leg. |
| **Reship Status** | PENDING / ON GOING / SUCCESSFUL / NOT APPLICABLE / UNSUCCESSFUL, shown on the Lyfe Order and on the Return. |

---

## 12. Document change log

> **Maintainer rule:** this is the single end-user document for the **Reshipment and Return** features. Any change to the Reshipment or Return doctypes, their workflows, buttons, automatic behaviour, messages, the Return link, the follow-up Task, the Status Overview Active Returns / Reshipment cards, or the Returns Dashboard must be reflected here **in the same change**, with a new row below and an updated *Last updated* date at the top. If the document and the system disagree, the system is right — fix the document.

| Version | Date | What changed |
|---|---|---|
| 2.0 | 2026-10-12 | Added **Part B — Return**: details, workflow, automatic behaviour, follow-up Task, effect on the order, rules/messages, how-to steps and eight use cases; Return rows in roles, where-to-see, FAQ and glossary. Sections renumbered (Where to see = 9, FAQ = 10, Glossary = 11, Change log = 12). Document renamed to *Reshipment and Return End User Guide*. |
| 1.0 | 2026-10-12 | First end-user guide: full feature details, how-to steps, use cases. Replaces the older `docs/reshipment_user_guide.md` (which described Customer Service as able to cancel and said tracking never moves the status — both are no longer true). |
