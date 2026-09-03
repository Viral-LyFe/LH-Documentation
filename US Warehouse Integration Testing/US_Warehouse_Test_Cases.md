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

**Order ID:** `LYF-MN-2026-0029`

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

><img width="1752" height="917" alt="image" src="https://github.com/user-attachments/assets/e0dc9b2f-0700-4903-b31d-acbb0050e188" />
><img width="1646" height="867" alt="image" src="https://github.com/user-attachments/assets/3a313162-f3c3-4ded-a2d8-d55d6d5cac08" />


**Result:** ☐ Pass ☐ Fail
**Notes:**

---

## Test Case 2 — Order Ships Directly From India (Nothing in US Stock)

**Order ID:** `LYF-MN-2026-0030`

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

<img width="1715" height="830" alt="image" src="https://github.com/user-attachments/assets/3da3195c-b677-4d95-bda1-6ea171ad18c8" />
<img width="1916" height="607" alt="image" src="https://github.com/user-attachments/assets/9f510ff8-07e5-49a4-8e51-e133c639aab8" />


**Result:** ☐ Pass ☐ Fail
**Notes:**

---

## Test Case 3 — Changing Your Mind: Route an India Order Through the US Warehouse Instead

**Order ID:** `LYF-MN-2026-0031`

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

<img width="1162" height="492" alt="image" src="https://github.com/user-attachments/assets/5fb1b22f-c600-4521-9832-30da27e4b2b0" />

<img width="1676" height="887" alt="image" src="https://github.com/user-attachments/assets/38fb3ba2-5c3c-4711-90a7-dd505b907a6d" />


**Result:** ☐ Pass ☐ Fail
**Notes:**

---

## Test Case 4 — Order Automatically Books Once It Arrives at the US Warehouse

**Order ID:** `LYF-MN-2026-0053` (same order as Test Case 3, continued)

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

<img width="1866" height="862" alt="image" src="https://github.com/user-attachments/assets/36e10562-5daa-4ba2-8235-c6abd5dac98b" />

<img width="1652" height="921" alt="image" src="https://github.com/user-attachments/assets/103203e6-1238-493c-9f1c-6d2f2b86f05d" />

<img width="1662" height="827" alt="image" src="https://github.com/user-attachments/assets/ff24886a-ac7a-450d-af0e-e444640590ac" />

**Result:** ☐ Pass ☐ Fail
**Notes:**

---

## Test Case 5 — Mixed Order (Some Items From the US, Some From India)

**Order ID:** `LYF-MN-2026-0032`

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

<img width="1722" height="917" alt="image" src="https://github.com/user-attachments/assets/73e530fb-1d1f-4739-b9c8-78ab7620418f" />

**Factory Lag Destination**

<img width="1722" height="917" alt="image" src="https://github.com/user-attachments/assets/8adc477e-8858-40ca-a1e0-b453d13bae0c" />

**Pending List Address**

<img width="1901" height="230" alt="image" src="https://github.com/user-attachments/assets/96e2e4e3-361b-4a98-b4a7-785b5cf36dd4" />

**When Factory Lag Destination is Direct to Customer**

<img width="1892" height="191" alt="image" src="https://github.com/user-attachments/assets/43d200ba-ccca-482f-a81a-5b112caaeb6a" />



**Result:** ☐ Pass ☐ Fail
**Notes:**

---

## Test Case 6 — Shipping Paperwork Shows the Right Address

**Order ID:** _(to be filled in when this test is run)_

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

**Order ID:** _(to be filled in when this test is run)_

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

## Test Case 8 — Order Fails Silently on 1Click's Side (Looks Successful, Isn't) ⭐ Highest Priority

**Order ID:** `LYF-MN-2026-0047`

**What we're checking:** if 1Click says "we got it" but secretly failed to
actually create the order, does our system notice — or does it show
"Submitted to 1Click" for an order that was never really booked?

**Steps:**
1. Create a normal order that would route fully to the US warehouse.
2. This needs a developer to simulate 1Click sending back a "failed" response
   while pretending everything is fine — ask a developer to set this up for
   you, then check the result on screen.

**Expected Result:**
- The order shows status **"1Click Error,"** not "Submitted to 1Click."
- No 1Click order number appears on the order.

> 📷 **[ IMAGE PLACEHOLDER — Screenshot of the order showing "1Click Error" status after the simulated failure ]**

**Result:** ☑ Pass ☐ Fail
**Notes:** Executed 2026-09-02 (developer-run, simulated response). Order
correctly landed in status **"1Click Error"** with `oneclick_order_id`
empty. Matches expected result exactly — please still grab the screenshot
from the order form for the record.

---

## Test Case 9 — Stock Check Fails (System Should Not Guess "Zero Stock")

**Order ID:** `LYF-MN-2026-0048`

**What we're checking:** if the system can't reach 1Click to check US
warehouse stock at all (their system is down or erroring), does our system
wrongly assume "nothing's in stock" and send a US-stocked order all the way
to India by mistake?

**Steps:**
1. Create an order using an item you know IS in US stock.
2. This needs a developer to simulate the stock-check call failing.
3. Check where the order ends up.

**Expected Result:**
- The order should NOT quietly route to India as if the item were out of
  stock.
- Instead, something should visibly flag that the stock check itself
  failed — either the order should be held for a human to review, or an
  error should show up somewhere.

> 📷 **[ IMAGE PLACEHOLDER — Screenshot of the order landing in "1Click Error" status after the simulated stock-check failure ]**

**Result:** ☑ Pass ☐ Fail
**Notes:** Executed 2026-09-02, using SKU `KJTFL-16-ABZ` (real US stock).
**Result was better than originally expected** — the order correctly landed
in status **"1Click Error"**, NOT silently routed to India. Important
nuance found only by running this test: there are actually **two different
code paths** that check US stock — a plain-item path (used here) which
correctly surfaces the failure as an error, and a separate BOM-expansion
path (used for mixed/multi-component orders) that DOES swallow the same
kind of failure and silently treats it as "zero stock." **Recommend
re-running this same test using a BOM-linked item** to confirm that second
path's behavior specifically — that's the one still flagged as a real risk.

---

## Test Case 10 — One Bad Tracking Update Doesn't Corrupt Others

**Order ID:** `LYF-MN-2026-0034` (control) / `LYF-MN-2026-0040` (simulated failure)

**What we're checking:** if 1Click sends back garbled/broken tracking
information for one order during a routine tracking check, does it mess up
that order's data, or worse, affect other orders being checked at the same
time?

**Steps:**
1. Pick two orders that are already "Submitted to 1Click" — one that already
   has a tracking number showing, one that doesn't yet.
2. This needs a developer to simulate a broken/garbled response from 1Click
   for one of the two orders during the next tracking check.
3. Check both orders afterward.

**Expected Result:**
- The order that already had a tracking number keeps it — untouched.
- The order that got the broken response does not end up with garbage data
  in its tracking field; it just skips that update and tries again next
  time.
- The other, healthy order's tracking check still completes normally in the
  same run.

> 📷 **[ IMAGE PLACEHOLDER — Screenshot comparing both orders after the test — one shows untouched/skipped tracking, the other shows a normal update ]**

**Result:** ☑ Pass ☐ Fail (with a caveat)
**Notes:** Executed 2026-09-02. No data corruption occurred — confirms the
core safety claim. **Caveat:** neither of the two real orders used for this
run had a tracking number yet at the time of testing, so the "healthy order
still updates normally" half of this test wasn't fully proven, only the
"broken one doesn't corrupt anything" half. **Recommend re-running once you
have a real order with an actual tracking number already on file**, to
fully confirm the healthy side too.

---

## Test Case 11 — Order for an Item 1Click Doesn't Recognize

**Order ID:** _(to be filled in when this test is run)_

**What we're checking:** if we submit an order for an item that was never
registered on 1Click's side, they told us it gets put "on hold" in their
system. Does our system realize that, or does it just think the order went
through fine?

**Steps:**
1. Create an order using an item/SKU you're confident has never been added
   to 1Click's system.
2. Let it route and submit normally.
3. Check the order's status in our system, then separately check 1Click's
   own dashboard for the same order.

**Expected Result:**
- **This is currently an open question we need 1Click to answer** — do they
  send back any signal that the order is on hold, or does their system just
  quietly hold it with no flag in their response?
- Whatever their answer, note here whether our order shows "Submitted to
  1Click" (looking fine) while 1Click's own dashboard actually shows it
  stuck/on hold. If so, that's the gap — write it down clearly with
  screenshots from both sides.

> 📷 **[ IMAGE PLACEHOLDER — Screenshot of our order status next to 1Click's dashboard status for the same order, side by side ]**

**Result:** ☐ Pass ☐ Fail
**Notes:**

---

## Test Case 12 — Submitting the Same Order Twice

**Order ID:** `LYF-MN-2026-0049`

**What we're checking:** if something goes wrong and the same order
accidentally gets sent to 1Click a second time (e.g. after a slow/timed-out
first attempt), does the customer end up with two boxes shipped instead of
one?

**Steps:**
1. Submit an order to 1Click normally and confirm it gets a real order
   number.
2. Ask a developer to trigger the submission a second time for that same
   order, simulating what would happen after a timeout/retry.
3. Check both our system and 1Click's dashboard afterward.

**Expected Result:**
- Ideally, either our system refuses to submit it a second time, or 1Click
  recognizes it's the same order and doesn't create a duplicate.

> 📷 **[ IMAGE PLACEHOLDER — Screenshot showing the 406 error from 1Click on the second submission attempt ]**

**Result:** ☑ Pass ☐ Fail (safe outcome, but not because of our own code)
**Notes:** Executed 2026-09-02. First submission succeeded normally (real
1Click order ID `1659101`). Second submission for the same order was
**rejected by 1Click itself** with an HTTP 406 error — no duplicate was
created. **Important:** this protection is coming entirely from 1Click's
side, not from any check in our own app — our code still has no guard that
stops it from attempting a second submission in the first place (it only
found out it failed after calling 1Click again). Worth fixing on our side
too, so a second attempt is stopped before ever calling 1Click, rather than
relying on them to reject it every time.

---

## Test Case 13 — 1Click Doesn't Respond At All (Times Out)

**Order ID:** `LYF-MN-2026-0050` (plain order — tested) / _(mixed-order version still pending)_

**What we're checking:** if 1Click's system just hangs and never responds
when we try to submit an order, does our order get stuck in limbo forever,
or does it fail cleanly so someone can retry?

**Steps:**
1. Ask a developer to simulate 1Click never responding to a submission
   attempt.
2. Check the order's status after waiting the normal amount of time.
3. Repeat the same test on a **mixed order** (Test Case 5 style) where two
   separate shipments would normally be created, to check nothing is left
   half-done.

**Expected Result:**
- The order ends up cleanly in **"1Click Error"** status, not stuck
  indefinitely with no explanation.
- For the mixed-order version: neither shipment should be left in a
  half-created, orphaned state.

> 📷 **[ IMAGE PLACEHOLDER — Screenshot of the order showing "1Click Error" after the simulated timeout, plus a second screenshot for the mixed-order case ]**

**Result:** ☑ Pass (plain order) ☐ Fail — mixed-order case still needs a run
**Notes:** Executed 2026-09-02 for the plain-order case only. Order landed
cleanly in **"1Click Error"** after the simulated timeout — no stuck/limbo
state. **Still need to run:** the mixed-order (two-shipment) version of
this test, which requires setting up a real mixed order first and timing
the simulated timeout to land between the two child submissions — more
setup than a single-order test, left for a follow-up run.

---

## Test Case 14 — Stock Changes Between the Check and the Actual Booking

**Order ID:** _(to be filled in when this test is run)_

**What we're checking:** our system checks stock once before deciding to
route an order to the US warehouse. But what if the stock changes (someone
else buys it) in the few moments between that check and actually booking
with 1Click?

**Steps:**
1. This is a timing-based scenario that's hard to trigger naturally — a
   developer will need to simulate 1Click reporting a different (lower)
   quantity at booking time than what the earlier stock check showed.
2. Check what the order's final status/details show.

**Expected Result:**
- **This is currently an open question we need 1Click to answer:** what do
  they actually do in this situation — reject the order, ship what they can
  (partial), or hold it as a backorder?
- Whatever they do, our system should show that clearly on the order — not
  silently mark it as a normal successful full shipment when it wasn't.

> 📷 **[ IMAGE PLACEHOLDER — pending: needs 1Click's answer before this can be fully tested and screenshotted ]**

**Result:** ☐ Pass ☐ Fail
**Notes:**

---

## Test Case 15 — Force US: Manually Routing an Out-of-Stock Item to the US Warehouse

**Order ID:** `LYF-MN-2026-0040`

**What we're checking:** CS can manually override routing and force an
order to the US warehouse even if the item isn't actually in stock there.
Does that override work, and what happens on 1Click's side?

**Steps:**
1. Create an order using an item you know is NOT in US stock.
2. On the order, find the routing override option and select **"Force
   US,"** entering a reason.
3. Save and let it submit.

**Expected Result:**
- The order routes to the US warehouse and submits to 1Click despite the
  item being out of stock.
- Check 1Click's own dashboard for what actually happens to that order on
  their side (may go on hold — note whatever you see).

> 📷 **[ IMAGE PLACEHOLDER — Screenshot of the order showing Force US override and 1Click submission despite 0 stock ]**

**Result:** ☑ Pass ☐ Fail
**Notes:** Executed 2026-09-02, using zero-stock SKU `MHRB-200-AC`. Order
correctly routed `US_FULL` and submitted with a real 1Click order ID
(`1659099`), despite the item having 0 stock. One thing worth checking
manually: the order's `warehouse` field still showed "Factory - LH" rather
than "US Warehouse - LH" after this — please confirm on screen whether that
looks right to you, or flag it if it seems off.

---

## Test Case 16 — Force India: Manually Routing an In-Stock Item to India Instead

**Order ID:** `LYF-MN-2026-0041`

**What we're checking:** the reverse override — CS forces an order to ship
from India even though everything is already sitting in the US warehouse.

**Steps:**
1. Create an order using an item you know IS in US stock.
2. On the order, find the routing override option and select **"Force
   India,"** entering a reason.
3. Save.

**Expected Result:**
- The order correctly routes to the Factory/India side, not the US
  warehouse.
- No order is created on 1Click.
- The reason you entered is saved and visible on the order.

> 📷 **[ IMAGE PLACEHOLDER — Screenshot of the order showing Force India override, routed to Factory despite full US stock ]**

**Result:** ☑ Pass ☐ Fail
**Notes:** Executed 2026-09-02, using in-stock SKU `3.5FT-TB-200-SB` (100
units available). Order correctly routed `INDIA_DIRECT_DROPSHIP`, landed in
"Factory Assignment," no 1Click order was created — exactly as expected.

---

## Test Case 17 — Can't Force a Route Without Giving a Reason

**Order ID:** `LYF-MN-2026-0042`

**What we're checking:** whether the system actually requires a reason
before letting someone force a routing override — and that this can't be
skipped even by someone going around the normal screen.

**Steps:**
1. On any order, try to select a "Force US" or "Force India" override and
   save it **without** typing a reason.

**Expected Result:**
- The system blocks the save and shows an error asking for a reason.
- (A developer should separately confirm this is also blocked if someone
  tries to do it through a direct system call, not just the screen — ask
  them to confirm and note the result here.)

> 📷 **[ IMAGE PLACEHOLDER — Screenshot of the blocked save with the "reason required" error, if/once this is fixed ]**

**Result:** ☐ Pass ☑ **Fail**
**Notes:** Executed 2026-09-02 via a direct system call (developer-level
test). **The order saved successfully with `route_plan = "Force US"` and no
reason at all — nothing blocked it.** This is a real gap: the "reason
required" check that exists in the code only applies to the specific
"Force India" button/flow (`force_factory_only_fulfillment`) — there is no
general check that covers `route_plan` being set directly, including
through the normal order form if a reason field is simply left blank and
saved. **Recommend this gets fixed before relying on override reasons being
present** — please also test directly on the form (not just via a direct
system call) to confirm whether the screen itself blocks it even though the
underlying save does not.

---

## Test Case 18 — Fee Lines Don't Get Sent to 1Click

**Order ID:** `LYF-MN-2026-0043` (real item + fee) / `LYF-MN-2026-0045` (fee only)

**What we're checking:** orders sometimes have a "Custom Fee" or "Customs
Fee" line added — these aren't real physical items and shouldn't be treated
as something to ship or check stock for.

**Steps:**
1. Create an order with one real item (in US stock) plus one Custom Fee
   line. Save and let it process.
2. Separately, create a second order with ONLY a Custom Fee line (no real
   items at all). Try to let it process.

**Expected Result:**
- Test 1: the fee line is clearly marked/excluded and doesn't affect
  routing; the real item routes and submits normally.
- Test 2: the system should block this from being sent to 1Click as an
  empty order — it should show a clear message instead.

> 📷 **[ IMAGE PLACEHOLDER — Screenshot of the fee line marked "Excluded," plus a screenshot of the fee-only order's Factory Assignment status ]**

**Result:** ☑ Pass (Test 1) ☑ **Fail** (Test 2)
**Notes:** Executed 2026-09-02. **Test 1 passed exactly as expected** — the
fee line was correctly badged "Excluded," the real item routed and
submitted normally with a real 1Click order ID (`1659100`). **Test 2 did
not behave as expected** — a fee-only order (no real items at all) was NOT
blocked with an error message. Instead it silently became a normal-looking
Factory order (status "Factory Assignment," no 1Click submission — so
nothing was actually sent to 1Click, which is good — but there was no clear
message telling anyone this order has nothing real to fulfill). The
existing "no items to submit" error only exists on the US-warehouse
submission path; an order with only a fee line never reaches that path
because it's routed to India by default instead. **Recommend adding a
similar check that fires regardless of which route the order takes.**

---

## Test Case 19 — Can't Save a Standard Order Without a SKU

**Order ID:** `LYF-MN-2026-0046`

**What we're checking:** a Standard (non-Custom) order shouldn't be saveable
if an item is missing its SKU — this prevents bad data going further down
the pipeline.

**Steps:**
1. Create a new Standard order, New status, with an item row that has no
   SKU filled in. Try to save.
2. On that same order, change the order type to **Custom**. Try to save
   again.
3. Change it back to Standard and move it past New status (e.g. Approve
   it). Then try editing it again with a blank SKU on a row and save.

**Expected Result:**
- Step 1: save is blocked with a "Missing SKU" style error.
- Step 2: save succeeds once it's a Custom order.
- Step 3: save succeeds — the block only applies to brand-new Standard
  orders, not ones further along.

> 📷 **[ IMAGE PLACEHOLDER — Screenshot showing the order saved successfully despite the missing SKU, once you've reproduced it on screen ]**

**Result:** ☐ Pass ☑ **Fail** (Step 1)
**Notes:** Executed 2026-09-02, direct system-level test. **Step 1 did not
block the save** — a Standard order in New status with a completely blank
SKU on a real item row saved without any error. This is a real gap: the
"Missing SKU" check exists correctly written in the code, but the function
that runs it is never actually wired up to fire when an order is saved — it
sits unused. **Recommend this gets connected and fixed**, since right now
Standard orders can go through the pipeline with no SKU at all, which is
exactly what this rule was meant to prevent. Please also confirm on the
actual screen (not just a direct system test) whether the form blocks it —
if it does, that means the safety net only exists in the browser and not on
the server, which is also worth flagging separately.

---

## Test Case 20 — Every New Order Gets a 72-Hour Dispatch Target

**Order ID:** `LYF-MN-2026-0046` (same order used in Test Case 19)

**What we're checking:** every new order should automatically get a target
dispatch date/time of 72 hours after it was created.

**Steps:**
1. Create any new order (Standard or Custom).
2. Look at the "Promised Dispatch By" field right after saving.

**Expected Result:**
- The field is automatically filled in with a date/time exactly 72 hours
  after the order was created.
- Saving the order again later does not change this value.

> 📷 **[ IMAGE PLACEHOLDER — Screenshot of the new order showing the empty Promised Dispatch By field, once reproduced on screen ]**

**Result:** ☐ Pass ☑ **Fail**
**Notes:** Executed 2026-09-02, direct system-level test. **The field was
left completely blank** on a freshly-created order — it was not
auto-filled with a 72-hour target at all. Same root cause as Test Case
19's failure: the code that's supposed to stamp this value is written
correctly, but it's never actually triggered when an order is created or
saved — it's disconnected from the rest of the system. **Recommend this
gets connected and fixed**, since this field is meant to be a key SLA
target used elsewhere. Please also confirm on the actual order screen
whether it's blank there too.

---

## Test Case 21 — Alerts Fire When a Shipment Gets Stuck

**Order ID:** _(to be filled in when this test is run)_

**What we're checking:** if an order routed through the US warehouse gets
"stuck" at any stage for too long, the right team should get an alert.

**Steps:**
1. Find (or create) an order that's routing through the US warehouse but
   has no India-to-US tracking number entered yet.
2. Ask a developer to help simulate enough time passing (48 hours) without
   a tracking number being added.
3. Separately, find a shipment marked "Shipped" toward the US warehouse but
   not yet marked "Received," and simulate 24 business hours passing.

**Expected Result:**
- Step 2: an alert/notification is created for the India team, since the
  shipment hasn't been labeled/shipped after 48 hours.
- Step 3: an alert/notification is created for the US warehouse team, since
  the shipment hasn't been checked in after 24 business hours.

> 📷 **[ IMAGE PLACEHOLDER — Screenshot of each alert/notification being created ]**

**Result:** ☐ Pass ☐ Fail
**Notes:**

---

## Test Case 22 — Mixed Orders Splitting Into Two Separate Shipments

**Order ID:** _(to be filled in when this test is run)_

**What we're checking:** there's an older, separate mechanism that can split
one mixed order into two completely separate shipments to the customer
(rather than the "wait and send as one shipment" behavior tested in Test
Case 5). We need to confirm whether this old behavior still exists and get
a decision on whether it should still be allowed.

**Steps:**
1. This requires a developer's help to identify and trigger an order that
   matches the OLDER split condition specifically (different from Test Case
   5's scenario).
2. Observe whether the order ends up creating two separate 1Click order
   numbers / two separate customer shipments.

**Expected Result:**
- Confirm whether this still happens today.
- This is not a pass/fail in the usual sense — it needs a decision from the
  founder on whether two-shipment splits should still be allowed for this
  older case, or retired in favor of the newer single-shipment approach
  (Test Case 5).

> 📷 **[ IMAGE PLACEHOLDER — Screenshot showing two separate 1Click order numbers coming from one original order, for founder review ]**

**Result:** ☐ Pass ☐ Fail ☐ Needs Founder Decision
**Notes:**

---

## Test Case 23 — Cancelling an Order After It's Already Been Sent to 1Click

**Order ID:** _(to be filled in when this test is run)_

**What we're checking:** what actually happens, end to end, when a customer
cancels an order that's already been submitted to 1Click.

**Steps:**
1. Take an order that's already "Submitted to 1Click."
2. Follow whatever the current manual cancellation process is with CS.
3. Check the order in our system afterward — does anything change
   automatically?
4. Separately, try switching that same order to "Factory only" fulfillment
   and confirm the system blocks it (since it's already been sent to
   1Click).

**Expected Result:**
- Nothing changes automatically in our system — this is confirmed to be a
  fully manual process today (write down exactly what CS does, step by
  step, as the actual expected result here).
- Step 4 should be blocked with a clear message.

> 📷 **[ IMAGE PLACEHOLDER — Screenshots of CS's manual cancellation steps, plus the blocked Factory-only attempt ]**

**Result:** ☐ Pass ☐ Fail
**Notes:**

---

## Test Case 24 — Two Orders for the Same Low-Stock Item at the Same Time

**Order ID:** `LYF-MN-2026-0051` and `LYF-MN-2026-0052` (run, but see note — not a full test)

**What we're checking:** if there's only 1 unit of an item actually
available, and two orders for that same item are created around the same
time, does the system correctly prevent both from being told "yes, it's in
stock" — or could it accidentally oversell it?

**Steps:**
1. Find (or ask a developer to help set up) an item with only 1 unit
   available in the US warehouse.
2. Create two separate orders for that same item, as close together in time
   as possible.
3. Check how each order routes.

**Expected Result:**
- Only one of the two orders should be treated as "in stock" and routed to
  the US warehouse; the other should not.

> 📷 **[ IMAGE PLACEHOLDER — pending: needs a genuine 1-unit-stock SKU to properly test ]**

**Result:** ☐ Pass ☐ Fail — **inconclusive, needs a real run**
**Notes:** Attempted 2026-09-02, but no SKU with exactly 1 unit of real
stock was available to test with at the time — only a fully zero-stock SKU
(`MHRB-200-AC`) was on hand, so both test orders correctly (and
unsurprisingly) routed to India, since there was genuinely nothing to fight
over. **This does not prove the actual race condition is safe** — it just
confirms the zero-stock case works, which was already known. **Please
re-run this properly once a SKU with exactly 1 real unit can be arranged**
(ask 1Click to set one up specifically for this test, or catch a real SKU
at that exact stock level) — this is the one that actually tests the
scenario.

---

## Test Case 25 — A Wrong Manually-Entered Tracking Number Doesn't Get Auto-Corrected

**Order ID:** _(to be filled in when this test is run)_

**What we're checking:** if someone manually types in the wrong tracking
number on an order, does the system's automatic tracking check later fix
it, or does the wrong number just stay there?

**Steps:**
1. On a submitted order, manually type in a tracking number you know is
   incorrect.
2. Wait for (or trigger) the next automatic tracking check.
3. Check the tracking number field afterward.

**Expected Result:**
- The wrong tracking number is confirmed to stay as-is — the automatic
  check does not overwrite it, even though 1Click's real data would say
  otherwise.
- This is intentional current behavior — the point of this test is to
  confirm it's actually true, and to flag whether the business is OK with
  it staying this way.

> 📷 **[ IMAGE PLACEHOLDER — Screenshot of the order before and after the tracking check, showing the wrong number unchanged ]**

**Result:** ☐ Pass ☐ Fail
**Notes:**

---

## Test Case 26 — Unfamiliar Carrier Names Create New Carrier Records

**Order ID:** N/A — carrier-only test, not tied to a specific order

**What we're checking:** when 1Click reports a carrier name we've never seen
before (including typos), our system automatically creates a brand new
Carrier record for it — this is expected, but worth confirming it doesn't
silently clutter up the Carrier list.

**Steps:**
1. Ask a developer to simulate 1Click reporting an unfamiliar/misspelled
   carrier name during a tracking check.
2. Check the Carrier list afterward.

**Expected Result:**
- A new Carrier record is created automatically with that exact name/typo.
- This is expected behavior, not a bug — but note it here so the team knows
  to periodically clean up the Carrier list if odd entries pile up.

> 📷 **[ IMAGE PLACEHOLDER — Screenshot of the new "UPSS-TYPO-TEST-26" Carrier record in the list ]**

**Result:** ☑ Pass ☐ Fail
**Notes:** Executed 2026-09-02. Simulated an unfamiliar carrier string
(`UPSS-TYPO-TEST-26`) and confirmed a brand new Carrier record was
auto-created with that exact typo'd name — matches expected behavior
exactly. Please grab a screenshot of this test record in the Carrier list,
then feel free to delete it afterward since it was created purely for this
test.

---

## Test Case 27 — Marking a Shipment "Received" Doesn't Mean 1Click Actually Has the Stock

**Order ID:** _(to be filled in when this test is run)_

**What we're checking:** when we mark an internal shipment "Received" (goods
arrived at the US warehouse), that's based on our own tracking/manual check
— not on 1Click confirming they've physically checked the stock into their
own system. We want to document this timing gap clearly.

**Steps:**
1. Take a shipment (from Test Case 3/4) and mark it "Received."
2. Note the exact time you marked it, and the time the order automatically
   got booked with 1Click right afterward.

**Expected Result:**
- The order books with 1Click almost immediately after being marked
  "Received" in our system.
- This confirms that "Received" is our own belief the goods arrived — not a
  guarantee that 1Click's inventory already reflects it. This is a known,
  accepted timing gap — the goal of this test is just to document it
  clearly, not to "fix" it.

> 📷 **[ IMAGE PLACEHOLDER — Screenshot showing both timestamps side by side ]**

**Result:** ☐ Pass ☐ Fail
**Notes:**

---

## Test Case 28 — SKU Auto-Fill From Item Name (Currently Not Active)

**Order ID:** N/A

**What we're checking:** there's a feature that was built to automatically
pull a SKU out of an item's name if it's written like `(SKU: ABC-123)` — but
it turns out this feature isn't actually connected to anything yet in the
live system.

**Steps:** None — there is nothing to click or test right now. This is
listed here so it's not forgotten, not because there's a test to run.

**Expected Result:** N/A — flagged for the development team to either
finish connecting this feature, or remove it if it's no longer needed.

> 📷 **[ IMAGE PLACEHOLDER — N/A, nothing to screenshot until this is connected ]**

**Result:** ☐ N/A — Dev Follow-up Needed
**Notes:**

---

## Summary

| Test Case | Order ID | Result |
|---|---|---|
| 1 — US Full Order | `LYF-MN-2026-0029` | ☐ Pass ☐ Fail |
| 2 — India Direct | `LYF-MN-2026-0030` | ☐ Pass ☐ Fail |
| 3 — Route via US Warehouse | `LYF-MN-2026-0031` | ☐ Pass ☐ Fail |
| 4 — Auto-book on Arrival | `LYF-MN-2026-0031` | ☐ Pass ☐ Fail |
| 5 — Mixed Order | `LYF-MN-2026-0032` | ☐ Pass ☐ Fail |
| 6 — Shipping Paperwork Address | _(pending)_ | ☐ Pass ☐ Fail |
| 7 — Tracking Updates | _(pending)_ | ☐ Pass ☐ Fail |
| 8 — Silent Failure on Create Order ⭐ | `LYF-MN-2026-0047` | ☑ Pass |
| 9 — Stock Check Failure Misroutes | `LYF-MN-2026-0048` | ☑ Pass (see note — second code path still needs testing) |
| 10 — Bad Tracking Doesn't Corrupt | `LYF-MN-2026-0034` / `-0040` | ☑ Pass (partial — re-run recommended) |
| 11 — Unknown SKU / 1Click Hold | _(blocked — needs 1Click's answer)_ | ☐ Pass ☐ Fail |
| 12 — Duplicate Submission | `LYF-MN-2026-0049` | ☑ Pass (1Click blocked it, not our own code) |
| 13 — 1Click Timeout | `LYF-MN-2026-0050` | ☑ Pass (plain order) — mixed-order case still pending |
| 14 — Stock Race Condition | _(blocked — needs 1Click's answer)_ | ☐ Pass ☐ Fail |
| 15 — Force US Override | `LYF-MN-2026-0040` | ☑ Pass |
| 16 — Force India Override | `LYF-MN-2026-0041` | ☑ Pass |
| 17 — Override Reason Required | `LYF-MN-2026-0042` | ☑ **Fail** — real gap found |
| 18 — Fee Line Handling | `LYF-MN-2026-0043` / `-0045` | ☑ Pass (real item) / ☑ **Fail** (fee-only order) |
| 19 — Missing SKU Block | `LYF-MN-2026-0046` | ☑ **Fail** — real gap found |
| 20 — 72h Dispatch Target | `LYF-MN-2026-0046` | ☑ **Fail** — real gap found |
| 21 — Stuck Shipment Alerts | _(pending — needs back-dated timestamps)_ | ☐ Pass ☐ Fail |
| 22 — Two-Shipment Split (Old Path) | _(pending)_ | ☐ Pass ☐ Fail ☐ Needs Founder Decision |
| 23 — Cancel After Submission | _(pending — needs CS's manual process documented)_ | ☐ Pass ☐ Fail |
| 24 — Same-Item Double Order Race | `LYF-MN-2026-0051` / `-0052` | ☐ Inconclusive — needs a real 1-unit SKU |
| 25 — Manual Tracking Not Auto-Corrected | _(pending)_ | ☐ Pass ☐ Fail |
| 26 — Unfamiliar Carrier Auto-Creation | N/A (carrier-only test) | ☑ Pass |
| 27 — Received ≠ 1Click Confirmation | _(pending)_ | ☐ Pass ☐ Fail |
| 28 — SKU Auto-Fill (Not Active) | N/A | ☐ N/A — Dev Follow-up Needed |

## Real Gaps Found During This Execution Round (2026-09-02)

Four genuine bugs were confirmed by actually running these tests, not just
reading the code — these need developer follow-up before they can be
marked resolved:

1. **Test Case 17 — Override reason is not actually enforced** when a
   route is forced directly (only the separate "Force India" button flow
   checks for it).
2. **Test Case 18 (Test 2) — A fee-only order isn't blocked** with a clear
   error; it silently becomes a normal-looking order instead.
3. **Test Case 19 — The "Missing SKU" block never fires** — the check
   exists in the code but isn't connected to anything.
4. **Test Case 20 — The 72-hour dispatch target is never stamped** — same
   root cause as #3, the code exists but isn't connected.

Items 3 and 4 share the same underlying cause and are likely fixable
together.

**Tested by:** Claude (automated/developer-level execution) — screenshots
and final sign-off still need a human pass over the UI, per each test
case's image placeholder.
**Date:** 2026-09-02
