# Reshipment and Return — Factory Guide

| | |
|---|---|
| **Features** | **Lyfe Order Reshipment** and **Lyfe Order Return** — the parts that matter to the **Factory** team |
| **Who it is for** | Factory team (and anyone covering Factory work) |
| **Document version** | 1.0 |
| **Last updated** | 2026-10-13 |
| **Full guide** | `Reshipment_and_Return_End_User_Guide.md` covers every role. This guide is the Factory-only cut of it. |

> **Your focus:** Prepare the replacement parcel, issue material, create the CJ label, dispatch it, move it through the US Warehouse when needed, and mark returned goods as received.

---
## Contents

1. [What are a Reshipment and a Return?](#what-are-a-reshipment-and-a-return)
2. [What Factory can do](#what-factory-can-do)

**PART A — RESHIPMENT (Factory does most of the work)**

3. [Reshipment — all the details](#reshipment--all-the-details-of-the-feature)
4. [Reshipment — step by step for Factory](#reshipment--step-by-step-for-factory)
5. [Reshipment — use cases for Factory](#reshipment--use-cases-for-factory)

**PART B — RETURN (Factory only marks goods received)**

6. [Return — what Factory does](#return--what-factory-does)

**BOTH**

7. [FAQ and troubleshooting](#faq-and-troubleshooting)
8. [Glossary](#glossary)

---

## What are a Reshipment and a Return?

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

How the two link together is explained in [Reshipment and Return](#reshipment-and-return) and in Return use case 3 in Part B.

<img width="1700" height="907" alt="image" src="https://github.com/user-attachments/assets/84fc0aed-8122-4c3c-9f49-b1998f5f6e11" />

---

## What Factory can do

| Document | Create | Edit | Workflow steps Factory performs | Cancel |
|---|:---:|:---:|---|:---:|
| **Reshipment** | ❌ | ✅ | **Start Progress**, **Internal Review**, **Ready for Dispatch**, **Dispatched** (and the Rework → Internal Review loop) | ❌ |
| **Return** | ❌ | ✅ | **Mark Received** | ❌ |

- **Customer Service** creates the Reshipment/Return, approves or rejects the optional Internal Review, and posts to 1Click.
- **Shipping Forwarder** moves the reshipment through *Awaiting Shipping → Shipped*; **Super Admin** can complete it.
- Only a **System Manager** can cancel. If a reshipment must be stopped, ask one.

---

**PART A — RESHIPMENT**

## Reshipment — all the details of the feature

### Naming

Every reshipment is named **`{Order ID}-R{number}`**, counting per order. For order `LH2946`: `LH2946-R01`, then `LH2946-R02`, and so on. You can create as many reshipments for one order as needed.

### The form — every section and field

<img width="1920" height="5135" alt="screencapture-172-17-59-31-8000-app-lyfe-order-reshipment-LH3045-R01-2026-10-12-17_43_44" src="https://github.com/user-attachments/assets/60bf12fc-7e68-46db-ac5e-f2c39b0d24ae" />


#### Header

| Field | What it is |
|---|---|
| **Lyfe Order** | The original order. Filled automatically and read-only. |
| **Return** | Optional link to the Lyfe Order Return this reshipment replaces goods for. Filled automatically when you create the reshipment from a Return. Only a Return of the **same order**, that is **not Cancelled**, is accepted. |
| **Customer** | Taken from the order. Read-only. |
| **Warehouse** | Where the reshipment ships from — **you must choose it**. Setting **Factory - LH** switches on the CJ Shipping tab, the Gate Pass and the invoice. |
| **Order Type** | Standard or Custom. Copied from the order. |
| **Workflow State** | Current status (see “The workflow (statuses)”). Read-only — change it with the buttons at the top. |
| **Factory Assignment Date / Ready for Dispatch Date / Reshipment Completed Date** | Stamped automatically the first time each step happens. Read-only. Used for SLA reporting. |
| **Reason** | **Required.** One of the reasons listed in “What are a Reshipment and a Return?”. |
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
| **Shipment Cost**, **Custom Charges**, **Additional Charges** | Costs of shipping to the customer (see “Costs”). |

#### US Warehouse Leg (only when "Via US Warehouse" is ticked)

| Field | What it is |
|---|---|
| **Carrier (to US) / Tracking Number (to US Warehouse)** | Tracking for Factory → US Warehouse ("Tracking #1"). Filled from the Bulk Transfer Batch. |
| **Delivered to US Warehouse**, **US Warehouse Delivered At** | Set when the package reaches the US. |
| **Bulk Transfer Batch** | The batch the reshipment was sent in. Read-only. |
| **Customer Leg / charges to US** | The cost fields for this leg (see “Costs”). |

#### CJ Shipping tab (only when Warehouse = Factory - LH)

Used to create the international shipping label (FedEx via CJ).

| Block | Fields |
|---|---|
| **Consignor (Shipper)** | Pick an **Export Partner** — name, contact, address, GST type and GSTIN fill in by themselves. |
| **Consignee (Receiver)** | Name, email, phone, address, city, state, zip, country. Copied from the order. |
| **Package Details** | **Parcel Details** table — weight, length, width, height. |
| **Customs & Shipment** | Customs Value and Currency, Goods Description, Ship Purpose, Terms of Sale, Duty & Tax, CSB Type, Package Type, DDP, Exporter Invoice No., Signature Required, Invoice Attachment (PDF). |
| **Shipment Items** | Item, Qty, **Rate (USD)**, **HSN Code**. Filled automatically when the Material Issue for Order is submitted (see step “Issue material (Factory)”). |
| **Output** | **AWB No.**, **CJ Tracking No.**, **Shipping Label** — filled when you click **Generate CJ Label**. |

<img width="1685" height="917" alt="image" src="https://github.com/user-attachments/assets/1f92cdaa-2dfe-4402-81a5-67bf975ea2b4" />

<img width="1757" height="916" alt="image" src="https://github.com/user-attachments/assets/92690cbb-f18e-41b0-9e41-1f664b256093" />


#### 1Click Logistics tab (only when "Via US Warehouse" is ticked)

Read-only status of the 1Click order and load: order id, submitted at, tracking number, status, error message, load status. See Use case 5.

#### Connections / Attachments / References

| Place | What it shows |
|---|---|
| **Connections** | Linked **Material Issue for Order**, **Sales Invoice** and **Gate Pass**. |
| **Attachments** | **Factory Images** (photos uploaded by the Factory). |
| **References** | **Gate Pass** and **ERP Sales Invoice** created for this reshipment. |

### The workflow (statuses)

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

<img width="1805" height="916" alt="image" src="https://github.com/user-attachments/assets/328c6dbc-5350-4b7f-a42a-5cadece9eb2d" />


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

> **Tracking moves the status for you.** Once a tracking number and carrier are saved, the hourly tracking job sets: *Delivered* → **Completed**, *label created but not scanned* → **Awaiting Shipping**, *anything else (moving)* → **Shipped**. It never moves a reshipment back out of Completed or Cancelled. When tracking moves the reshipment, the **original order** is updated in exactly the same way as when someone clicks the workflow button (Reshipment Completed Date, Reship Status, order status, and the order's reship tracking status), so nothing needs to be done by hand.

### What the system does for you automatically

| When | What happens |
|---|---|
| On save | **Reshipment COGS** is recalculated. The reshipment is named `{Order}-R{nn}`. |
| Warehouse is set to **Factory - LH** | The reshipment is **assigned to the Factory user**. The **Factory Assignment Date** is stamped. |
| Parent order is fulfilled from the US Warehouse (when the reshipment is created) | The **Lyfe Order** (not the reshipment) is assigned to the Factory user. |
| Status becomes **Ready for Dispatch** | The **Ready for Dispatch Date** is stamped. **Only if a submitted Material Issue for Order exists for this reshipment** — otherwise it is blocked (see “Rules and error messages”). If Warehouse = Factory - LH: a **Gate Pass** is created (or today's draft one is reused) and a draft **Sales Invoice** is created. |
| Tracking number / carrier saved | Copied to the Lyfe Order's reship tracking fields. |
| Every hour | Tracking is checked, the status advances (see “The workflow (statuses)”), and alerts are raised for lost / stale / exception parcels. |
| Status becomes **Completed** | **Reshipment Completed Date** is stamped. |
| Any important change | A line is added to the **timeline of the original Lyfe Order** (see “How a Reshipment affects the original Lyfe Order”). |

### How a Reshipment affects the original Lyfe Order

| Reshipment event | Lyfe Order status becomes | Other effects |
|---|---|---|
| Moves to **Reshipment In Progress** | **Reshipment In Progress** | — |
| Moves to **Ready for Dispatch** | (no change) | Order's Reship Status = **PENDING**, Reship Date set |
| Moves to **Shipped** | (no change) | Reship Status = **ON GOING** |
| Moves to **Completed** | **Completed** | Reship Status = **SUCCESSFUL** |
| **Cancelled** | **Completed** (the original order was already delivered) | — |

Exception: if a **Return on the same order is still open** (Requested / In Transit / Received), a Completed or Cancelled reshipment leaves the order as **Return InProgress** until that Return is closed or cancelled.

The table applies in every case — whether a person clicks the workflow button or the **hourly tracking job** moves the reshipment (for example, when the carrier reports Delivered the reshipment becomes **Completed**, the order becomes **Completed** with Reship Status **SUCCESSFUL**, and the **Reshipment Completed Date** is stamped).

**Timeline:** each creation and change (status, Return link, reason, warehouse, carrier, tracking numbers, costs, Gate Pass, invoice) is written to the Lyfe Order's timeline, so Customer Service can follow the reshipment without opening it.

<img width="1702" height="917" alt="image" src="https://github.com/user-attachments/assets/cf3c2ab4-54a8-49c9-a70c-6bd78afe582b" />

### Reshipment and Return

- **Create a Reshipment from a Return:** on a Lyfe Order Return, click **Create → Reshipment**. It opens a new reshipment, pre-filled from the order, with the **Return** field set and the reason "Customer Return Replacement". This works for **any Return that is not Cancelled**, even after it is Closed.
- The Return follows its reshipments:
  - **Reship Status** on the Return: *PENDING* (not started), *ON GOING* (shipping), *SUCCESSFUL* (all done).
  - A Return can have **several** reshipments (for example, the first got lost).
  - A **Refund-only** Return that gets a reshipment becomes **Refund & Reship** (while it is still open), so it can only be closed once the reshipment is done.
- A reshipment created directly from the order is a **"Reshipment only"** (no customer return behind it). You can still attach it to a Return later in the **Return** field.
- Returns Dashboard counts: **Return only**, **Return + Reshipment**, **Reshipment only**, **Reshipments from returns**.

### Costs

| Cost | Covers | How it is filled |
|---|---|---|
| **Reshipment COGS** | Cost of the goods sent | Automatic |
| **Shipment Cost** | Shipping to the customer (direct reshipment) or Factory → US (via-US reshipment, its share of the batch) | Typed (direct) / automatic from the Bulk Transfer Batch (via-US) |
| **Customer Leg Shipment Cost** | US Warehouse → Customer (via-US only) | Typed by hand |
| **Custom Charges / Additional Charges** | Customs and other charges, per leg | Typed (via-US batch shares are automatic) |

Reshipment cost appears as its own line in the **PnL dashboard**, the Founder dashboard and the Status Overview. Users who may not see costs do not see these columns.

### Rules and error messages

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

## Reshipment — step by step for Factory

> Customer Service creates the reshipment (from the Lyfe Order or from a Return) and chooses the reason, items and warehouse. Your work starts when it is in **Reshipment Request**.

### Start the work (Factory)

The Factory user clicks **Start Progress**. The Lyfe Order status changes to **Reshipment In Progress**.

### Issue material (Factory)

1. On the reshipment click **Create → Material Issue for Order**.
2. Complete and **submit** the Material Issue. This is **mandatory** before **Ready for Dispatch**.
3. Submitting it fills **Shipment Items** in the CJ Shipping tab.

   <img width="1632" height="560" alt="image" src="https://github.com/user-attachments/assets/6c60a3c9-03b6-4fb0-a303-a275189b32a5" />

   <img width="1631" height="796" alt="image" src="https://github.com/user-attachments/assets/ad6f5599-7f30-439f-bbb9-29910154b6d5" />



### Optional quality review

Factory clicks **Internal Review**. Customer Service clicks **Approve** (→ Approved) or **Reject** (→ Rework Required; Factory fixes it and sends it back to Internal Review).

<img width="1697" height="611" alt="image" src="https://github.com/user-attachments/assets/b5bbcb30-ccd8-401f-896a-c9ffffa15f58" />

<img width="1401" height="772" alt="image" src="https://github.com/user-attachments/assets/ac45b9c2-ddfd-4d9a-9034-05949a452c4d" />


### Create the CJ label (Factory warehouse only)

1. Open the **CJ Shipping** tab and complete all sections (see “The form — every section and field”).
2. Click **CJ Shipping → Generate CJ Label**.

  <img width="1691" height="465" alt="image" src="https://github.com/user-attachments/assets/9df57f7a-4bd7-4b95-a3fe-f17531fb6369" />

3. If something is missing, a message lists it. When it succeeds, **Parcel Details**, **Exporter Invoice No** and **Invoice Attachment (PDF)** are filled in.

   <img width="1562" height="535" alt="image" src="https://github.com/user-attachments/assets/729c2ca3-dc1e-4746-9e05-59a6b246499d" />

### Ready for Dispatch → Dispatched (Factory)

1. Click **Ready for Dispatch**. A **Gate Pass** and a draft **Sales Invoice** are created automatically and linked in **References**.

   <img width="1442" height="721" alt="image" src="https://github.com/user-attachments/assets/bf1c0469-3945-48d6-89a3-b5eb54da4701" />


2. Hand the parcel to the carrier, then click **Dispatched**.
3. If no invoice was created, use **Create → Sales Invoice**.
4. If the Gate Pass was cancelled (or was never created), use **Create → Gate Pass** while the reshipment is *Ready for Dispatch* and Warehouse is **Factory - LH**. It makes a new Gate Pass (or reuses today's draft one) and links it in **References**. It is refused if the reshipment already has a Gate Pass that is not cancelled. Each box row gets its **Box Group** filled in automatically (BOX-<Gate Pass number>-<row number>), the same as when you add rows on the Gate Pass by hand.

### Tracking

1. Enter **Carrier** and **Tracking Number** (Shipping section).
2. From now on **do nothing** — the hourly job fills **Tracking Status** and **Tracking Details** and moves the reshipment to **Awaiting Shipping → Shipped → Completed** by itself.

   <img width="1377" height="897" alt="image" src="https://github.com/user-attachments/assets/746854ab-fae1-4000-9021-725a4eb7cd4d" />

3. If **Tracking Alert** shows a value (for example *Possible Lost*), contact the carrier; Customer Service is notified by Slack/email.

### Completion

When the carrier reports delivery the reshipment becomes **Completed**, the original Lyfe Order becomes **Completed** and its Reship Status shows **SUCCESSFUL**. If it was created from a Return, the Return's Reship Status also becomes **SUCCESSFUL**.

<img width="1665" height="682" alt="image" src="https://github.com/user-attachments/assets/75c63fcd-e118-48f9-a5e8-e61cf9aa199a" />


---

## Reshipment — use cases for Factory

> In each case Customer Service creates the reshipment first; the steps below are what Factory does.

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

<img width="1722" height="791" alt="image" src="https://github.com/user-attachments/assets/4daf1fc8-39ce-44cf-86a9-4e228daccbd7" />

### Use case 5 — Reshipment through the US Warehouse

1. Create the reshipment and **tick Via US Warehouse** (do this before it goes into a batch).
2. Factory completes **Start Progress → material issue → Ready for Dispatch → Dispatched**. The reshipment must be on a **submitted Gate Pass**.
3. Factory opens a **Bulk Transfer Batch** → **Update Orders → From Reshipment Order**, pastes the reshipment IDs (one per line) and clicks **Update**.

   <img width="1687" height="416" alt="image" src="https://github.com/user-attachments/assets/276581cc-81b7-4486-b225-df39d9f0b5d0" />

   <img width="1567" height="782" alt="image" src="https://github.com/user-attachments/assets/8f1a8630-750d-49d7-9c65-03fea14198ee" />

   <img width="1652" height="837" alt="image" src="https://github.com/user-attachments/assets/256c52b3-6c24-4fb6-b162-00a27f52b537" />


4. Factory enters the batch tracking number and carrier and **submits** the batch. The number is copied to each reshipment as **Tracking #1**. After this the reshipment can no longer be cancelled and Via US Warehouse is locked.
5. The system tracks Tracking #1 hourly: **In Transit to US Warehouse** → **US Warehouse Delivered**. If tracking is late, click **1Click → Mark Delivered to US Warehouse**.
6. Customer Service clicks **1Click → Post to 1Click** (always manual). Status becomes **Submitted to 1Click**. On a rejection the status is **1Click Error** with the reason shown; fix it and click **Post to 1Click** again.
7. 1Click ships to the customer; the customer's tracking number and carrier appear automatically. The status moves to **Awaiting Shipping → Shipped → Completed**.
8. Finance enters the batch charges (even days later); each reshipment's **Shipment Cost** share updates automatically. **Customer Leg Shipment Cost** (US → customer) is typed by hand.

   <img width="1385" height="921" alt="image" src="https://github.com/user-attachments/assets/27314db6-b7fe-4cb7-a9b1-a22c931ecc7d" />

### Use case 6 — 1Click rejected the reshipment

1. The reshipment shows **1Click Error**; the reason is in the **1Click Logistics** tab and a Slack alert was sent.
2. Fix the cause (for example a missing address or item).
3. Click **1Click → Post to 1Click** again.

### Use case 7 — Package reached the US warehouse but status did not change

1. Open the reshipment and click **1Click → Mark Delivered to US Warehouse**, confirm.
2. Then continue with **Post to 1Click** (Use case 5, step 6).

---

**PART B — RETURN**

## Return — what Factory does

Customer Service creates and manages Returns. Factory's only job is to **confirm that returned goods arrived** at **Factory - LH**.

1. Open the **Lyfe Order Return** (list view → filter **Return To Warehouse = Factory - LH**, state **Return In Transit**).
2. If the carrier has already reported *Delivered*, the system has moved it to **Return Received** by itself — nothing to do.
3. If the parcel is physically in your hands but the status is still **Return In Transit** (or **Return Requested**), click **Mark Received**.
4. The system stamps the **Return Received Date** and creates a follow-up **Task** for Customer Service (`support@lyfehardware.com`) to handle the refund / restock fee and close the Return. Factory is not assigned to it.
5. Factory does **not** enter refund details and does **not** close the Return.

| Status | Meaning | Moved by |
|---|---|---|
| **Return Requested** | The return is opened; goods not on their way yet. | Customer Service |
| **Return In Transit** | The customer has shipped the goods back. | Automatic (tracking saved) |
| **Return Received** | The goods are back with us. | Automatic (carrier Delivered) or **Mark Received** — Customer Service / **Factory** |
| **Closed** | Refund and/or replacement settled. | Customer Service only |
| **Cancelled** | The return did not go ahead. | System Manager only |

If the customer is owed a replacement, Customer Service clicks **Create → Reshipment** on the Return; it then reaches you as a normal reshipment (reason *Customer Return Replacement*).

---

## FAQ and troubleshooting

**The CJ Shipping tab is missing.**
Set **Warehouse** to exactly **Factory - LH**.

**I cannot move to Ready for Dispatch.**
A submitted **Material Issue for Order** is required for this reshipment (see “Issue material (Factory)”).

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

**The goods arrived at the Factory but the return is still "Return In Transit".**
The hourly job moves it when the carrier reports Delivered. If there is no tracking, click **Mark Received**.

---

## Glossary

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

> Document changes are logged in the full guide (section *Document change log*). If this guide and the system disagree, the system is right.
