# US Warehouse (1Click) Integration — Test Cases

This document walks through how to test the US Warehouse feature, step by
step, in plain language. No technical background needed — just follow the
steps, compare what you see on screen to the "Expected Result," and mark it
Pass or Fail.

---

## Before You Start

- Make sure you're testing on the **sandbox** system, not the live/production
  one, unless a test specifically says otherwise.
- Have a real order ready, or create a new one as each test describes.
- Take a screenshot at each key step and drop it into the placeholder boxes
  below — this makes it much easier for anyone reviewing the results later.

---

## Test Case 1 — Order Ships Fully From the US Warehouse

**What we're checking:** when everything a customer ordered is already sitting
in the US warehouse, the order should go straight there — no detour through
India.

**Steps:**
1. Create a new order using an item you know is in stock at the US warehouse.
2. Save the order and let the system process it.

**Expected Result:**
- The order shows it's being fulfilled from the **US Warehouse**.
- The order's status changes to **"Submitted to 1Click"**.
- A real order number from 1Click appears on the order.

> 📷 **[ IMAGE PLACEHOLDER — Screenshot of the order form showing status "Submitted to 1Click" and the 1Click order ID field filled in ]**

**Result:** ☐ Pass ☐ Fail
**Notes:**

---

## Test Case 2 — Order Ships Directly From India (Nothing in US Stock)

**What we're checking:** when nothing the customer ordered is in the US
warehouse, the order should go straight from the factory in India to the
customer — it should not wait around for the US warehouse at all.

**Steps:**
1. Create a new order using an item you know is **not** in stock at the US
   warehouse.
2. Save the order and let the system process it.

**Expected Result:**
- The order is handed to the Factory team, exactly like a normal order.
- No order is created on the 1Click side.
- The order looks like any other Factory order — nothing unusual stands out
  about it on screen.

> 📷 **[ IMAGE PLACEHOLDER — Screenshot of the order showing Factory Assignment status ]**

**Result:** ☐ Pass ☐ Fail
**Notes:**

---

## Test Case 3 — Changing Your Mind: Route an India Order Through the US Warehouse Instead

**What we're checking:** sometimes an order that was going to ship directly
from India should instead go through the US warehouse first. There's a
button for this.

**Steps:**
1. Open an order that's currently set to ship directly from India (see Test
   Case 2) — one that hasn't shipped yet.
2. Look for the **"Route via US Warehouse Instead"** button.
3. Click it and confirm the popup that appears.

**Expected Result:**
- A confirmation popup appears explaining what will happen.
- After confirming, the order switches to the "ship via US Warehouse first"
  plan.
- A new internal shipping record is created to track the goods moving from
  the factory to the US warehouse.
- The order still looks like a normal Factory order — nothing alarming shows
  up on screen.

> 📷 **[ IMAGE PLACEHOLDER — Screenshot of the confirmation popup ]**

> 📷 **[ IMAGE PLACEHOLDER — Screenshot of the order after clicking Yes, showing the new internal shipping record ]**

**Result:** ☐ Pass ☐ Fail
**Notes:**

---

## Test Case 4 — Order Automatically Books Once It Arrives at the US Warehouse

**What we're checking:** once the goods physically arrive at the US
warehouse and someone checks them in, the order should automatically get
booked with 1Click — no extra clicking needed beyond marking it "received."

**Steps:**
1. Continue from Test Case 3's order.
2. Find the internal shipping record created in Test Case 3.
3. Mark it as **"Shipped."**
4. Once ready, mark it as **"Received."**

**Expected Result:**
- After marking "Shipped," the shipment shows up on the tracking dashboard
  as in transit.
- After marking "Received" — within a few seconds, without anyone clicking
  anything else — the order automatically books with 1Click.
- The order's status becomes **"Submitted to 1Click,"** with a real 1Click
  order number attached.
- If something goes wrong on 1Click's side, the order status becomes
  **"1Click Error"** instead, with details saved for someone to check.

> 📷 **[ IMAGE PLACEHOLDER — Screenshot of the shipment marked Received ]**

> 📷 **[ IMAGE PLACEHOLDER — Screenshot of the order automatically showing "Submitted to 1Click" afterward ]**

**Result:** ☐ Pass ☐ Fail
**Notes:**

---

## Test Case 5 — Mixed Order (Some Items From the US, Some From India)

**What we're checking:** when an order has a mix — some items already at the
US warehouse, some that still need to come from India — the system should
correctly split them and wait for a human to review before doing anything
automatically.

**Steps:**
1. Create an order with at least two items: one that's in US stock, one that
   isn't.
2. Save the order and let it process.

**Expected Result:**
- The order sits in a "waiting for review" status — nothing is booked
  automatically yet.
- Two separate lists appear on the order: one showing what will come from
  the US warehouse, one showing what still needs to come from the factory.
- A person then needs to review and confirm the split before anything moves
  forward.

> 📷 **[ IMAGE PLACEHOLDER — Screenshot of the order showing the two separate item lists ]**

**Result:** ☐ Pass ☐ Fail
**Notes:**

---

## Test Case 6 — Shipping Paperwork Shows the Right Address

**What we're checking:** when an order is going through the US warehouse,
any shipping labels or invoices generated for the factory's portion should
show the US warehouse's address — not the customer's address.

**Steps:**
1. Use an order that's routing through the US warehouse (from Test Case 3 or
   5).
2. Generate the shipping paperwork / invoice for it.

**Expected Result:**
- The address on the paperwork is the US warehouse's address, not the
  customer's.
- If you switch a different order back to "ship direct to customer," the
  address correctly goes back to the real customer address.

> 📷 **[ IMAGE PLACEHOLDER — Screenshot of the invoice/paperwork showing the US warehouse address ]**

**Result:** ☐ Pass ☐ Fail
**Notes:**

---

## Test Case 7 — Tracking Updates Show Up Correctly

**What we're checking:** once an order is booked with 1Click, the system
should automatically check in periodically and pull the tracking number once
the warehouse ships it.

**Steps:**
1. Take an order that's already "Submitted to 1Click" (from Test Case 1 or
   4).
2. Wait for the tracking check to run (this happens automatically on a
   schedule), or ask someone to trigger it manually.

**Expected Result:**
- A tracking number and carrier show up on the order once 1Click has shipped
  it.
- That same tracking number also shows up on the actual customer-facing
  order (Shopify/Etsy) — this is the part that matters most to the customer.

> 📷 **[ IMAGE PLACEHOLDER — Screenshot of the tracking number appearing on the order ]**

**Result:** ☐ Pass ☐ Fail
**Notes:**

---

## Summary

| Test Case | Result |
|---|---|
| 1 — US Full Order | ☐ Pass ☐ Fail |
| 2 — India Direct | ☐ Pass ☐ Fail |
| 3 — Route via US Warehouse | ☐ Pass ☐ Fail |
| 4 — Auto-book on Arrival | ☐ Pass ☐ Fail |
| 5 — Mixed Order | ☐ Pass ☐ Fail |
| 6 — Shipping Paperwork Address | ☐ Pass ☐ Fail |
| 7 — Tracking Updates | ☐ Pass ☐ Fail |

**Tested by:**
**Date:**
