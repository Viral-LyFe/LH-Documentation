# Reshipments Through the US Warehouse — User Guide

This guide explains, in plain language, how a **Reshipment** can now travel **through the 1Click US Warehouse**, who does what, and what happens at each step. It also covers two related changes: how the **Bulk Transfer Batch** is filled, and the new rules on the **Gate Pass**.

---

## 1. The two ways a reshipment can travel

**Way 1 — Direct to the customer (nothing has changed)**

```
Factory  ───────────────►  Customer
         one tracking number
```

The Factory ships the package straight to the customer and enters the tracking number on the reshipment. This works exactly as it always has.

**Way 2 — Through the 1Click US Warehouse (new)**

```
Factory ──► 1Click US Warehouse ──► Customer
   Tracking #1              Tracking #2
```

The package first goes to the US warehouse. 1Click then ships it from there to the customer. This trip has **two tracking numbers**:

| | What it tracks | Where you see it |
|---|---|---|
| **Tracking #1** | Factory → US Warehouse | "Tracking Number (to US Warehouse)" on the reshipment |
| **Tracking #2** | US Warehouse → Customer | "Tracking Number" on the reshipment |

Only **Tracking #2** is the customer's tracking number. Tracking #1 is for our team only.

---

## 2. Who does what

| Team | Responsibility |
|---|---|
| **Customer Service** | Decides the reshipment goes through the US warehouse, and posts it to 1Click once the package has arrived there. |
| **Factory** | Prepares the reshipment, adds it to the Bulk Transfer Batch, and enters the batch's tracking number. |
| **Finance / Accounts** | Enters the batch shipping charges, even days after the batch is submitted. |
| **Admin / System Manager** | The only people who can cancel a reshipment. |

---

## 3. Step by step (Way 2)

### Step 1 — Mark the reshipment "Via US Warehouse"
Open the reshipment and tick **Via US Warehouse**. Leave it unticked for a direct reshipment.
- Do this **before** the reshipment goes into a batch. After that the box is locked and cannot be changed.
- Every reshipment that existed before this change is treated as **direct**.

### Step 2 — Gate Pass
The reshipment must be on a **submitted Gate Pass** before its batch can be submitted (see section 7 for the Gate Pass rules).

### Step 3 — Add it to a Bulk Transfer Batch
1. Open the batch and click **Update Orders**. It now has two choices:
   - **From Lyfe Order** — for normal orders.
   - **From Reshipment Order** — for reshipments.
2. Type or paste the IDs in the box, **one per line**, for example:
   ```
   LH2946-R01
   LH3045-R01
   ```
3. Click **Update**. The items appear in the batch table.

Good to know:
- Blank lines and repeated IDs are ignored.
- If an ID does not exist, nothing is added and the message names the ID.
- A reshipment that is **not** ticked "Via US Warehouse" is refused, and the message tells you which one.
- A reshipment that is already **Shipped** or **Completed** is skipped, with a message.
- A reshipment with no shipment items is skipped, with a message.

### Step 4 — Enter the batch tracking number and submit
Enter the batch's tracking number and carrier. The system copies them onto each reshipment as **Tracking #1** (only if that reshipment doesn't already have one). Then submit the batch.
- Once the batch is submitted, a reshipment in it **cannot be cancelled**.

### Step 5 — Watch the package reach the US
The system checks Tracking #1 automatically every hour:
- On its way → status **In Transit to US Warehouse**.
- Arrived → status **US Warehouse Delivered**.

If the tracking is late or missing, use **1Click → Mark Delivered to US Warehouse** on the reshipment to move it forward by hand.

### Step 6 — Customer Service posts it to 1Click
When the status is **US Warehouse Delivered** (or **1Click Error**), click **1Click → Post to 1Click**.
- This is **always done by a person**. Nothing is posted automatically.
- 1Click receives the reshipment ID (for example `LH2946-R01`) as the PO number, the reshipment's items, and the customer's address from the original order.
- Any item 1Click doesn't know yet is registered with 1Click automatically.
- The status becomes **Submitted to 1Click**.
- If 1Click rejects it, the status becomes **1Click Error**, the reason is saved on the reshipment, and a message goes to the Slack alerts channel. Fix the problem and click **Post to 1Click** again.

### Step 7 — The customer's tracking number appears by itself
Once 1Click ships the package to the customer, its tracking number is filled into **Tracking Number** and **Carrier** automatically. You don't type it. The system checks every hour.

### Step 8 — Shipped and Completed
- **Shipped** — when the carrier first scans the package on its way to the customer.
- **Completed** — when the package is delivered to the customer.

If the customer's shipment shows as delivered but Tracking #1 was never marked delivered, the system marks the US arrival for you, so the statuses stay consistent.

---

## 4. The status journey (Way 2)

| Status | What it means | Who moves it |
|---|---|---|
| Ready for Dispatch | Factory has prepared the reshipment | Factory |
| Awaiting Tracking | Dispatched; waiting for the tracking number | Factory |
| **In Transit to US Warehouse** | Package is travelling to the US warehouse | System (hourly) |
| **US Warehouse Delivered** | Package has reached the US warehouse | System, or Customer Service by hand |
| **Submitted to 1Click** | Order and load sent to 1Click | Customer Service (button) |
| **1Click Error** | 1Click rejected it; needs fixing and re-posting | System |
| Awaiting Shipping | 1Click has a label; carrier hasn't scanned it yet | System |
| Shipped | Carrier scanned it on the way to the customer | System |
| Completed | Delivered to the customer | System |

Direct reshipments keep their old statuses and skip the bold ones.

---

## 5. Costs — who enters what

A reshipment through the US warehouse has two shipping costs:

| Cost | Covers | How it gets there |
|---|---|---|
| **Shipment Cost** | Factory → US Warehouse (the reshipment's share of the batch) | **Automatic.** Don't type it. |
| **Customer Leg Shipment Cost** | US Warehouse → Customer | **Typed by hand** on the reshipment. |

**How the batch cost is shared**
- The batch charges (shipping, customs, other) are split between everything in the batch **by weight**.
- If a reshipment's items have no weights, its **parcel weight** is used instead.
- If some rows in the batch have no weight at all, the whole batch is split by **quantity** instead.
- Charges are usually entered **days after the batch is submitted**. That is fine: when Finance enters or changes the charges, every share updates automatically.
- Until the charges are entered, each share shows 0.

Both costs are included in the reshipment cost shown in the **PnL dashboard**, the **Founder dashboard** and the **Status Overview** page (as "Shipment Cost" and "Customer Leg Cost"). People who are not allowed to see costs do not see these columns.

---

## 6. Alerts and automatic checks

| What happens | When | Where it goes |
|---|---|---|
| **No customer tracking number** | The package reached the US warehouse more than **48 hours** ago and there is still no customer tracking number | One Slack message to the alerts channel, **once** per reshipment |
| **1Click did not confirm the load** | A day after the package reached the US warehouse, 1Click still doesn't show it received | One Slack message, then re-checked daily until confirmed |
| **1Click Error** | 1Click rejected the order or the load | Slack message, with the reason saved on the reshipment |
| **Inbound email to 1Click** | Every order **and reshipment** in the batch has been posted to 1Click | Sent **once** to 1Click support. Until then it waits. It also covers batches that contain only reshipments. "Send Inbound Email" on the batch still works for a manual resend. |

You can also click **1Click → Check Load Status** on a reshipment at any time to see what 1Click says about its load.

---

## 7. Gate Pass — new rules

When submitting a **Gate Pass**, every row now needs all of these:

- Weight (zero does not count)
- By (Carrier)
- Size, written as **length x width x height**, for example `30x20x10`
- Signature Required
- DDP
- Box Group

If anything is missing or the size is not in the `30x20x10` form, the Gate Pass will not submit and the message tells you which row to fix.

**Why:** when the Gate Pass is submitted, each row's size and weight are copied to the order, or to the reshipment if the row is for a reshipment, as its parcel details. An unreadable size used to be skipped quietly, so the order kept old box information.

---

## 8. What the customer sees

- Only the **customer's tracking number (Tracking #2)** is the customer-facing one.
- **Nothing is sent** to Shopify, ShipStation or the customer by this process for reshipments. Customer Service shares tracking with the customer the same way as before.
- Tracking #1 and the 1Click details are for our team only.

---

## 9. Cancelling

- Only **Admin / System Manager** can cancel a reshipment. Customer Service cannot.
- A reshipment **cannot be cancelled once its batch is submitted**.
- Cancelling does not cancel anything already created in 1Click. That has to be handled with 1Click.

---

## 10. Common questions

**I ticked "Via US Warehouse" by mistake and the reshipment is in a batch.**
The box is locked once the reshipment is in a batch. Ask the Factory to remove it from the batch while the batch is still a draft. Then you can change the box.

**A reshipment was refused when I added it to the batch.**
Read the message. The usual reasons are that "Via US Warehouse" is not ticked, it is already Shipped or Completed, or it has no shipment items.

**The package reached the US but the status didn't change.**
Use **1Click → Mark Delivered to US Warehouse**. Then continue with **Post to 1Click**.

**"Post to 1Click" isn't showing.**
It only appears when the status is **US Warehouse Delivered** or **1Click Error**, and only on a reshipment marked "Via US Warehouse".

**1Click Error.**
Open the reshipment and read the 1Click Error text on the 1Click Logistics tab. Fix the cause, then click **Post to 1Click** again. If an order was already created in 1Click, it is not created twice.

**The customer tracking number hasn't appeared.**
Check that the status is Submitted to 1Click. 1Click only gives the number once it has shipped the package. If 48 hours pass after the US arrival without it, you will get the Slack alert.

**The batch shipping cost shows 0.**
The batch charges haven't been entered yet. Finance enters them after the batch is submitted, and the shares fill in then.

**Can I type the Shipment Cost myself?**
Please don't for reshipments in a batch. The batch fills it in and will overwrite your number. Use **Customer Leg Shipment Cost** for the US → Customer shipping.

**Does a direct reshipment change in any way?**
No. Leave the box unticked and everything works as before.
