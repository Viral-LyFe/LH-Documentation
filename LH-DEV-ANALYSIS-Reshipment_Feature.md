# Reshipment Feature — User Guide

Status: **Planned — not yet available in the system**

---

## 1. What is this feature?

Sometimes an order needs to be shipped again after it has already gone out — for
example, the carrier lost or damaged the package, the wrong item was sent, there was a
manufacturing defect, or a customer is owed a warranty replacement.

Today, there's no proper place to record this — it just gets noted informally on the
original order. This feature adds a dedicated **Reshipment** record so every replacement
shipment is tracked properly: why it happened, what's being sent again, what it cost, and
where it currently stands.

An order can have more than one reshipment over its lifetime (e.g. a carrier loses the
package once, and months later the customer needs a warranty replacement too) — each one
is tracked separately, but always linked back to the original order.

---

## 2. Where do I find it?

- **Starting a reshipment**: Open the original **Order**. You'll see a **Reshipment**
  option in the **Create** menu at the top of the order.
- **Viewing existing reshipments for an order**: An **Open Reshipments** link/button
  will be available on the order once one exists.
- **Viewing all reshipments in the system**: Search for **Lyfe Order Reshipment** in the
  main search bar.

---

## 3. How a reshipment moves forward

A reshipment goes through three stages:

1. **Reshipment Request** — created when the issue is first reported. This is the
   starting point.
2. **Reshipment In Progress** — the warehouse/factory has picked it up and is preparing
   the replacement.
3. **Ready for Dispatch** — the replacement is packed and ready to be shipped. At this
   point, a **Gate Pass** (the same shipping paperwork used for regular orders) is
   created for you automatically.

You move a reshipment from one stage to the next using buttons on the reshipment record
itself — the same way you already move regular orders through their statuses.

Note: creating a Material Issue (the warehouse stock-out step) is **not** automatic —
you'll still do that manually, just like you do today for a regular order.

---

## 4. What information do I fill in?

When you create a reshipment, you'll record:

- **Which order** it belongs to (filled in automatically when you start it from the
  order).
- **Reason** — pick from a list: Carrier Lost, Carrier Damaged, Wrong Item Sent, Missing
  Items, Manufacturing Defect, Customer Return Replacement, Warranty Replacement, or
  Other.
- **Description** — a short note with extra detail.
- **Items being reshipped** — which SKUs and how many of each.
- **Cost of the reshipped goods** — this is an editable field, just like the Cost of
  Goods field on the original order, so you can correct it if needed.
- **Tracking number and courier** for this specific replacement shipment — kept separate
  from the tracking number on the original order, so shipping the replacement never
  overwrites or confuses the original shipment's tracking.
- **Shipping cost and total cost** of the reshipment.

---

## 5. What happens on the original order?

Once a reshipment reaches **Ready for Dispatch**, the original order is updated
automatically so its history stays accurate — the "reshipped on" date and reshipment
history section on the order will reflect the latest reshipment. You don't need to update
the original order by hand.

---

## 6. Something to try out (test scenario)

**Situation:** A carrier damaged a package in transit. The customer received 1 out of 3
items in good condition, and needs the other 2 replaced.

**Try this:**

1. Open an order that has already been delivered.
2. Go to **Create → Reshipment**.
3. Fill in:
   - Reason: **Carrier Damaged**
   - Description: "2 of 3 items damaged in transit, customer needs a replacement"
   - Items: add the 2 damaged items with their quantities and cost.
4. Save the reshipment. It should start out in the **Reshipment Request** stage.
5. Move it forward to **Reshipment In Progress**.
6. Separately, do the usual warehouse stock-out step for the 2 replacement items (this is
   still a manual step, same as normal orders).
7. Move the reshipment forward to **Ready for Dispatch**.
   - Check that a Gate Pass has been created for it automatically.
8. Add a tracking number and courier on the reshipment.
   - Go back to the original order and confirm its own tracking number and courier are
     unchanged — the reshipment's shipping details should not affect the original
     shipment's tracking.
9. Check the original order — it should now show that a reshipment happened, with the
   date updated automatically.
10. Repeat the process a second time on the **same order** (e.g. for a different reason,
    like a warranty replacement weeks later) and confirm:
    - Both reshipments show up, linked to the same order.
    - The order's history shows both events, not just the most recent one.
    - Nothing from the first reshipment gets overwritten or lost.

**What "working correctly" looks like:**
- You can create a reshipment from an order and move it through all three stages.
- A Gate Pass appears automatically once it's marked Ready for Dispatch.
- The reshipment's cost is editable.
- The reshipment's tracking info never overwrites the original order's tracking info.
- The original order's history correctly reflects one or more reshipments over time.
- No stock-out step happens automatically — you always do that step yourself.
