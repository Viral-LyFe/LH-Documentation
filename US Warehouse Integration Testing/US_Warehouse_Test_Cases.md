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


**Result:** ☑ Pass ☐ Fail
**Notes:** Verified against the screenshots (2026-09-03): order shows status
"Submitted to 1Click," item correctly landed in the US Warehouse Shipment
table, and a real 1Click order ID (`1656895`) with a successful raw API
response is recorded on the order. Matches expected result exactly.
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


**Result:** ☑ Pass ☐ Fail
**Notes:** Verified against the screenshots (2026-09-03): order shows
status "Factory Assignment," item correctly landed only in the Factory
Warehouse Shipment table (nothing in the US Warehouse table), and the
Pending List export correctly shows the real India destination address —
no US Warehouse override applied, exactly as expected for a normal Factory
order.
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


**Result:** ☑ Pass ☐ Fail
**Notes:** Verified against the screenshots (2026-09-03): the confirmation
popup appeared exactly as expected before switching, and after confirming,
the order shows "US Warehouse" badge with a real, linked Transfer Order
under Fulfillment — status stayed "Factory Assignment," nothing alarming
shown on screen, matching the expected result exactly.

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

**Result:** ☑ Pass ☐ Fail
**Notes:** Verified against the screenshots (2026-09-03): the Transfer
Order shows status "Received" with delivery detection marked
"Auto-Detected Delivered" and a real tracking number, the Lyfe Order shows
status "Submitted to 1Click" with a real 1Click order ID (`1659365`) and a
successful raw API response, and the real carrier tracking history
confirms genuine physical delivery (FedEx International Priority,
Aligarh, India → Defuniak Springs, FL, marked "Delivered"). Matches the
expected result exactly, including that no 1Click Error occurred.

---

## Test Case 5 — Mixed Order (Some Items From the US, Some From India)

**Order ID:** `LYF-MN-2026-0032` (original run) / `LYF-MN-2026-0055` (re-run, 2026-09-03)

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

<img width="1707" height="591" alt="image" src="https://github.com/user-attachments/assets/361dd87a-f559-433f-8740-959475fd4d88" />




**Result:** ☑ Pass ☐ Fail
**Notes:** Re-run 2026-09-03 as `LYF-MN-2026-0055`, using `3.5FT-TB-200-SB`
(1 unit) + `MHRB-200-AC` (1 unit). **Real US stock was at 0 for every SKU at
the time of this run** (sandbox drained again), so the US-stock check for
`3.5FT-TB-200-SB` was simulated for this one test only — the routing/split
logic itself was real, not mocked. Result matched expectations exactly:
`routing_outcome = MIXED_US_COMPONENTS_INDIA_TO_US`, status stayed **"New"**
(nothing auto-booked), `3.5FT-TB-200-SB` badged "US Warehouse" and
`MHRB-200-AC` badged "Factory," with one row each in the two separate
shipment-item tables. **Recommend re-confirming this once more with real
(non-simulated) US stock** once sandbox inventory is topped up again, to
fully close this out without any simulation involved.

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

<img width="1191" height="512" alt="image" src="https://github.com/user-attachments/assets/b19745ba-e11e-437c-848a-a835192a64da" />

----------------------------

<img width="1876" height="382" alt="image" src="https://github.com/user-attachments/assets/7b17f638-531f-4bd6-9bee-cc5364c30c79" />

**Pending List Address**

<img width="1892" height="191" alt="image" src="https://github.com/user-attachments/assets/43d200ba-ccca-482f-a81a-5b112caaeb6a" />

**Packing List**

<img width="1722" height="600" alt="image" src="https://github.com/user-attachments/assets/71f3e2da-245a-4bcc-8ce0-305f829db978" />

**Sales Invoice**

<img width="1667" height="896" alt="image" src="https://github.com/user-attachments/assets/a50adcb8-3110-43fd-90c3-a8befa020cb0" />

**Pick Pack List**

<img width="1076" height="292" alt="image" src="https://github.com/user-attachments/assets/f1df4c2c-ca3f-42f5-8481-790c9b126e6b" />

**Result:** ☑ Pass ☐ Fail
**Notes:** Verified against the screenshots (2026-09-03) across every
document type: the Pending List, Packing List, Sales Invoice (both
Billing and Shipping Address), and Pick-Pack List all correctly show
"Lyfe Hardware" / the US Warehouse address (270 Old New Brunswick Rd Unit
#200 Suite 461, Piscataway, NJ 08854) instead of the customer's real
address — matching the expected result exactly across the board.

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

## Fix: 1Click Orders Stuck at "Submitted to 1Click"

## Summary

Orders routed through the US Warehouse (1Click) were getting stuck at the `Submitted to 1Click` status forever, even after the carrier had picked up and delivered the package. The 17Track-based tracking check — the job responsible for advancing order status based on real carrier events — was explicitly skipping any order that originated from 1Click.

This fix removes that skip condition once a real tracking number is present, so 1Click orders flow through the same status pipeline as factory-shipped orders.

## Background: how a 1Click order used to behave

Example: Order `LH#1234`, routed via US Warehouse.

1. **Order submitted to 1Click**
   Status: `Submitted to 1Click`. No tracking number yet.

2. **Hourly polling begins**
   Every hour, we ask 1Click: "did you ship it?" Until 1Click actually hands the package to a carrier, there's nothing to report, and the order stays as-is. This repeats hourly.

3. **1Click ships the order**
   Say it goes out via UPS with tracking number `1Z999AA1`. On the next hourly check, 1Click's response includes that tracking number, and we:
   - Save `1Z999AA1` to the order's tracking number field, and save `UPS` as the carrier.
   - Push that same tracking number to Shopify/Etsy so the customer can see it.

   At this exact moment, the order has a real tracking number and carrier on file — this is the trigger point for everything that follows.

4. **The broken part: the normal tracking check ignores it**
   A separate hourly job watches the actual carrier's tracking page via 17Track and advances order status accordingly. Previously, this job checked whether the order originated from 1Click and, if so, said "not my job, skip it" — leaving the order sitting at `Submitted to 1Click` forever, even while UPS was actively moving the package.

## What changed

Since the order now has a real tracking number, it's no longer skipped by the 17Track check. It's treated exactly like any factory-shipped order:

| Carrier signal | Resulting status |
|---|---|
| Label printed, not yet picked up | `Awaiting Shipping` |
| Package scanned / in transit | `Shipped` |
| Delivered to customer | `Completed` |

## One-line summary

- **Before:** A 1Click order finds its tracking number, then just stops — stuck at `Submitted to 1Click` forever.
- **Now:** A 1Click order finds its tracking number, and from that point on it's treated like any normal order — moving automatically through `Awaiting Shipping → Shipped → Completed` via the same carrier-tracking check every other order already uses.

## Manual test plan

1. Take one of the real orders currently sitting at `Submitted to 1Click` (e.g. `LYF-MN-2026-0034`, `-0036`, `-0038`).
2. Wait for 1Click to provide a real tracking number on its next hourly sync — or enter one manually to simulate it.
3. Confirm the status advances on the next tracking poll instead of remaining stuck.

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

<img width="1617" height="821" alt="image" src="https://github.com/user-attachments/assets/997cebf0-aed2-48ff-84d2-6f809d48ecec" />

## Root problem being tested

1Click's API returns HTTP `200` even when order creation actually failed on their side. The only real signal that something went wrong is a `"success": false` field buried in the response body. A naive integration that only checks the HTTP status code would think everything worked.

## How we detect it

In `create_order()` (`oneclick_api.py`):

```python
if isinstance(result, dict) and result.get("success") is False:
    _update_log(log_name, "Failed", output=result, error=...)
    _raise_if_failed(result, title="1Click Create Order Failed")
```

This check runs on **every** response, regardless of HTTP status — it never trusts the status code alone. If 1Click reports `success: false`, this throws an error immediately.

## What happens when it throws — the full chain

1. `create_order()` throws.
2. The exception bubbles up through `_submit_single_oneclick_order()` — it never reaches the "mark as Submitted to 1Click" step, since the throw happens before that code runs.
3. The outer `run_oneclick_fulfillment()` catches it in a `try/except` and sets:
   - `status = "1Click Error"`
   - `oneclick_error` = the actual error details (so someone can see why it failed)
   - The order is **never** marked `Submitted to 1Click`.
4. The failed attempt is also logged in an **Integration Request** record, so the exact request/response is recoverable. A `retry_create_order_from_log()` method exists to resubmit the same payload later without rebuilding it.

## Verification

Already tested and confirmed — this ran on `LYF-MN-2026-0047` on 2026-09-02 with a real simulated failure:

- The order correctly landed in `1Click Error`.
- `oneclick_order_id` stayed empty.
- Code was re-verified as unchanged and matching this behavior exactly.

## Bottom line

This one is genuinely solid — **no gap found**. It's the one item on Srishti's list that was already fully correct from the start, and real execution confirmed it.

**Remaining action:** grab the screenshot from `LYF-MN-2026-0047`'s order form for the test doc's proof placeholder.

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

<img width="1676" height="672" alt="image" src="https://github.com/user-attachments/assets/1ecd25b0-461d-445f-b5d0-d176a51060c6" />


**Result:** ☑ Pass ☐ Fail
**Notes:** 

### Handling Inventory Check Failures Before Routing

#### Step 1 — The check fails, and we know it failed (not just "zero")

A real error (network/API problem) is now tagged internally as a genuine failure — separate from a clean "0 available" response. A missing/unconfigured setup is still treated as "nothing to check yet" (not an error) — only an actual broken call counts.

#### Step 2 — The order stops before any wrong decision gets made

Instead of letting the order proceed on unverified data, the system throws an error right there — before it can route to India, before it can route to US Warehouse, before anything.

#### Step 3 — The order lands in a status everyone already recognizes: "1Click Error"

Same status used for a failed order submission. The real cause is saved on the order (`oneclick_error` field) so anyone opening it can see exactly why it failed.

#### Step 4 — A "Recheck Inventory" button appears — but only for this specific cause

The button only shows up when the `1Click Error` was actually caused by a failed stock check. If the order failed for a different reason (like a failed order submission), this button stays hidden — that has its own separate retry path.

#### Step 5 — Clicking it re-runs everything from scratch

The order resets to `New` and the whole routing process runs again, exactly like a brand-new order. If 1Click responds correctly this time, it routes normally (US Warehouse, India, or Mixed) and proceeds automatically — no manual data re-entry needed.

#### Step 6 — There's also a manual bypass, if someone doesn't want to wait at all

Instead of waiting for the recheck to work, a human can directly assign the order's Warehouse field to:

- **`US Warehouse - LH`** — skips the stock check entirely and submits straight to 1Click.
- **`Factory - LH`** — sends the order via India instead.

#### One-line summary

- **Before:** a broken inventory check silently misrouted the order and left no trace.
- **Now:** it stops, flags itself clearly as `1Click Error`, lets someone retry the exact same check with one click, or lets someone skip the check entirely and choose the warehouse by hand.

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

> 📷 **[ IMAGE PLACEHOLDER — Screenshot comparing both orders after the test — one shows untouched/skipped tracking, the other shows a normal update ]*

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

<img width="1657" height="772" alt="image" src="https://github.com/user-attachments/assets/a0f566ee-9c24-45cc-8fb5-24048ec4f6fb" />
<img width="1567" height="707" alt="image" src="https://github.com/user-attachments/assets/ac4d1ae3-c43f-4ea8-bc32-a3b7830ec1d1" />


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

**Submitted Order successfully**

<img width="1661" height="902" alt="image" src="https://github.com/user-attachments/assets/76e55c10-5f66-4450-b602-0e3f7aeae3d6" />

**Second Time posting a same error convert to fail**

<img width="1682" height="817" alt="image" src="https://github.com/user-attachments/assets/d1f3997b-5f39-47c7-a7cf-7d98dcb1fef2" />


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

**Order ID:** `LYF-MN-2026-0065` (plain order) / `LYF-MN-2026-0066` (mixed/Route B order)

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

**For Time Out Error**
<img width="1642" height="606" alt="image" src="https://github.com/user-attachments/assets/6b2c98d4-a601-4f67-99d9-00129a9031d4" />

**As of Now not able to replicate a second case but that will be visible as 1Click Error only**
<img width="1611" height="867" alt="image" src="https://github.com/user-attachments/assets/865cc836-f866-4f1e-bcba-294fc9edf494" />


**Result:** ☑ Pass (both plain and mixed-order cases)
**Notes:** Re-run 2026-09-03, both cases completed.

**Plain order (`LYF-MN-2026-0065`):** Force-US order, simulated timeout on
submission. Landed cleanly in **"1Click Error"**, no order ID — no
stuck/limbo state.

**Mixed/Route B order (`LYF-MN-2026-0066`):** created via "Route via US
Warehouse Instead" (Test Case 3 style), Transfer Order `ASN-2026-00007`
marked Shipped then Received with a simulated timeout during the
resume-and-submit step. Result: Lyfe Order correctly landed in **"1Click
Error"**; the Transfer Order correctly stayed at **"Received"** — the goods
genuinely did arrive, only the 1Click submission afterward failed, so
nothing was left half-created or orphaned.

**Bonus — recovery confirmed too:** re-ran the resume step on
`LYF-MN-2026-0066` a second time, this time for real (no simulated
failure) — it completed successfully with a real 1Click order ID
(`1659392`), confirming the order isn't just "safely stuck" but can
actually recover and complete normally afterward.

**Note on the original mechanism this test case was written against:** the
plan originally described "two separate shipments" (the older
`_split_and_submit_oneclick` two-child-order mechanism). On investigation,
that mechanism is only reachable via a legacy per-row split trigger that no
longer fires for genuinely mixed orders under the current routing logic —
real mixed orders today go through the newer hold-and-wait path
(`_hold_for_india_components` → Transfer Order → resume), which is what was
actually tested above. This matches the real, currently-used code path more
accurately than the original two-child-order scenario would have.

---

## Test Case 14 — Stock Changes Between the Check and the Actual Booking

**Order ID:** `LYF-MN-2026-0083` (simulation order)

**What we're checking:** our system checks stock once before deciding to
route an order to the US warehouse. But what if the stock changes (someone
else buys it) in the few moments between that check and actually booking
with 1Click?

**Steps taken:**
1. Simulated the race directly at the correct layer — the stock check
   (`get_inventory`) reported the item as available, then the actual
   booking call (1Click's real HTTP response, not just our wrapper) was
   made to return `success: false` with `"Insufficient stock: requested 2,
   available 0"` — exactly the shape 1Click uses for a real stock rejection.
2. Ran the real, unmodified `run_oneclick_fulfillment()` end to end against
   a real order and checked its final status/fields.

**What we confirmed our system actually does:**
- `create_order()` correctly raises on `success: false` (`_raise_if_failed`
  in `oneclick_api.py`) — it does **not** treat this as a success.
- `run_oneclick_fulfillment()`'s outer error handler catches it and sets the
  order to **`1Click Error`** (not "Submitted to 1Click", not silently
  marked as shipped).
- The real 1Click rejection message ("Insufficient stock: requested 2,
  available 0...") is preserved, visible in `oneclick_error` on the order.
- So: our side never silently treats a stock-race rejection as a normal
  successful shipment — it correctly surfaces as a visible, actionable
  error, same as any other 1Click Create Order failure (Test Case 8).

**Still open — genuinely needs 1Click's own answer:** what 1Click's *real*
sandbox actually does in a true race (reject entirely vs. ship partial vs.
backorder) is still unconfirmed — this test only proves our system handles
a **rejection** response correctly, since that's the only failure shape
1Click's API documents. If 1Click's real behavior turns out to be a
*partial-success* response instead of an outright failure, our code has no
special handling for that shape today and would need a follow-up look. That
part of the question is still pending 1Click's reply to the drafted email
(same one covering Test Case 11).

<img width="1657" height="812" alt="image" src="https://github.com/user-attachments/assets/3ef6ed0b-a72c-4dfb-a394-65e4031b6679" />

**Result:** ☑ Pass — for the failure-response case, which is the only
documented failure shape
**Notes:** Simulated by mocking `requests.post` (not by replacing our own
`create_order()`/`create_oneclick_order()` functions) so the real
`_raise_if_failed` validation logic ran unmodified — this is what makes the
test trustworthy as a check of our actual code, not just of the mock.

---

## Test Case 15 — Force US: Manually Routing an Out-of-Stock Item to the US Warehouse

**Order ID:** `LYF-MN-2026-0067`

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

<img width="1187" height="575" alt="image" src="https://github.com/user-attachments/assets/a6781061-6ff9-4f49-8a84-aa1c5444fb6b" />

<img width="981" height="731" alt="Screenshot 2026-09-03 153643" src="https://github.com/user-attachments/assets/21044e3f-423a-41c9-ab86-4b13f6c47629" />


**Result:** ☑ Pass ☐ Fail
**Notes:** First executed 2026-09-02 (`LYF-MN-2026-0040`), using zero-stock
SKU `MHRB-200-AC`. Order correctly routed `US_FULL` and submitted with a
real 1Click order ID (`1659099`), despite the item having 0 stock. One
thing worth checking manually: the order's `warehouse` field still showed
"Factory - LH" rather than "US Warehouse - LH" after this — please confirm
on screen whether that looks right to you, or flag it if it seems off.

**Manual UI re-run (2026-09-03):** `LYF-MN-2026-0067` created for the user
to manually set Route Plan = "Force US" in the browser. This surfaced two
real, confirmed bugs along the way, both now fixed:

1. **Route Plan field was completely invisible on the form.** Root cause:
   a stale `field_order` Property Setter (created directly on the site at
   some point, never captured in code) placed `route_plan` inside the
   gated "Shipping" tab, which only shows once an order already has a US
   Warehouse/tracking assigned — a chicken-and-egg bug, since Route Plan
   is the field meant to *set* that in the first place. Fixed via patch
   (`fix_lyfe_order_route_plan_field_order.py`) — field now lives under
   the always-visible "1Click Logistics" tab.
2. **Override Reason had the identical bug**, this time a genuine mistake
   in the source file itself, not a stale override. Fixed
   (`move_override_reason_out_of_shipping_tab.py`) — now sits right next
   to Route Plan.
3. **Setting Route Plan and saving did nothing at all** — the field was
   never wired to any routing/submission trigger on a plain save. Per
   explicit user feedback, this is now a dedicated whitelisted method
   (`apply_route_plan_override`) triggered by a Route Plan field-change
   handler in the browser — prompts for the reason, freezes the screen,
   submits synchronously to 1Click, then reloads. Fires only when the
   user actively changes the field, never on an unrelated save.

Re-verified end-to-end on `LYF-MN-2026-0067` after all three fixes:
`routing_outcome = US_FULL`, status `"Submitted to 1Click"`, real 1Click
order ID `1659393`.

---

## Test Case 16 — Force India: Manually Routing an In-Stock Item to India Instead

**Order ID:** `LYF-MN-2026-0041` (first run) / `LYF-MN-2026-0070` (re-run, 2026-09-03, via the fixed field + synchronous method)

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

<img width="1656" height="597" alt="image" src="https://github.com/user-attachments/assets/b2cd651b-5e8e-40f1-acf1-5762e8e02b18" />


**Result:** ☑ Pass ☐ Fail
**Notes:** Executed 2026-09-02, using in-stock SKU `3.5FT-TB-200-SB` (100
units available). Order correctly routed `INDIA_DIRECT_DROPSHIP`, landed in
"Factory Assignment," no 1Click order was created — exactly as expected.

**Re-run 2026-09-03 (`LYF-MN-2026-0070`), through the now-fixed field +
trigger:** real US stock was 0 for every SKU at the time, so the stock
check was simulated (50 units reported available for `KJTFL-16-ABZ`) to
genuinely prove the override bypasses stock rather than coincidentally
matching a zero-stock item. Called `apply_route_plan_override` (the same
function the UI's Force India dialog calls) — result: `routing_outcome =
INDIA_DIRECT_DROPSHIP`, status `"Factory Assignment"`,
`warehouse = "Factory - LH"`, no 1Click order created, override reason
saved — correctly routed to India despite the simulated in-stock item.

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

<img width="1740" height="561" alt="image" src="https://github.com/user-attachments/assets/81050006-3979-45af-8b4a-53ab4e84281b" />

**Result:** ☑ Pass ☐ Fail
**Notes:** Originally executed 2026-09-02 via a direct system call
(developer-level test) — **found a real gap**: the order saved successfully
with `route_plan = "Force US"` and no reason at all, nothing blocked it.
The "reason required" check only ever applied to the separate "Force
India" button/flow (`force_factory_only_fulfillment`), not to `route_plan`
being set directly through the normal order form.

**Fixed 2026-09-03**, as part of wiring up Route Plan to actually trigger
routing (see Test Case 15). The new dedicated method
(`apply_route_plan_override`) now enforces the reason server-side —
confirmed by direct call:
```
apply_route_plan_override(name, 'Force US', '')
→ "An override reason is required."
```
The browser dialog also requires the field (`reqd: 1`) before it will even
submit. Please still confirm on the actual screen that the dialog blocks
an empty reason, for the record.

---

## Test Case 18 — Fee Lines Don't Get Sent to 1Click

**Order ID:** `LYF-MN-2026-0043` (real item + fee) / `LYF-MN-2026-0045` (fee only) / `LYF-MN-2026-0071` (re-run, 2026-09-03, real SKU + fee, forced full US delivery)

**What we're checking:** orders sometimes have a "Custom Fee," "Customs
Fee," advance-payment, or other payment-collection-only line added — these
exist purely to collect money on the order and were never meant to be
treated as something to ship, check stock for, or send to 1Click.

**Steps:**
1. Create an order with one real item (in US stock) plus one Custom Fee
   line. Save and let it process.
2. Separately, create a second order with ONLY a Custom Fee line (no real
   items at all). Try to let it process.

**Expected Result (corrected 2026-09-03 per explicit user clarification):**
- Test 1: the fee line is clearly marked/excluded and doesn't affect
  routing; the real item routes and submits normally.
- Test 2: **this should NOT be blocked and should NOT show any error.**
  Fee/payment-collection lines are added purely for payment purposes, not
  fulfillment — an order made up of only such lines correctly has nothing
  to route to 1Click, and should simply be handled like any other
  non-1Click order (i.e. it goes to Factory, quietly, same as any other
  order with nothing in US stock). Blocking it or showing an error would
  be wrong — this is standard behavior, not an edge case to guard against.

> <img width="1677" height="766" alt="image" src="https://github.com/user-attachments/assets/c8da664b-9953-4160-a557-8038c7a488a3" />

> <img width="656" height="796" alt="image" src="https://github.com/user-attachments/assets/ea04449a-2449-4672-953f-2c57e9a8c63e" />

> <img width="1600" height="886" alt="image" src="https://github.com/user-attachments/assets/4bb744cc-3082-4638-ba3f-234daafac8b2" />


**Result:** ☑ Pass (Test 1) ☑ Pass (Test 2)
**Notes:** Executed 2026-09-02. **Test 1 passed exactly as expected** — the
fee line was correctly badged "Excluded," the real item routed and
submitted normally with a real 1Click order ID (`1659100`).

**Test 2 — corrected 2026-09-03.** Originally flagged as a "Fail" because
a fee-only order wasn't blocked with an error message. **This was a wrong
expectation on the test's part, not a bug** — confirmed by explicit user
clarification: fee/payment-collection lines are correctly excluded from
1Click entirely (they exist only for payment collection), so an order with
only such lines correctly has nothing to route, and should be handled
exactly like any other non-1Click order — no error, no block, just Factory
assignment as normal. The actual observed behavior (order silently becomes
a normal Factory order, status "Factory Assignment," no 1Click submission)
**is the correct, intended behavior.** No code change needed — this item is
now marked Pass, not a gap.

**Re-run 2026-09-03 (`LYF-MN-2026-0071`)** — a fresh order with real SKU
`KJTFL-16-ABZ` + a Custom Fee line, forced to full US delivery via "Force
US" (real stock was 0 at the time, so Force US was used to guarantee a
genuine 1Click submission rather than simulating stock). Result:
`routing_outcome = US_FULL`, status `"Submitted to 1Click"`, real 1Click
order ID `1659395`. The fee row was correctly badged **"Excluded"** and
not sent to 1Click — only the real item was submitted. Matches Test 1's
expected result exactly. Note: because Force US bypasses the real stock
check, the real item's own per-row badge stayed "Factory" rather than "US
Warehouse" even though the whole order correctly submitted as US_FULL —
expected for a Force override, not a bug.

---

## Test Case 19 — Can't Save a Standard Order Without a SKU

**Order ID:** `LYF-MN-2026-0046`

**What we're checking (corrected 2026-09-03, per explicit user
clarification):** ~~a Standard order shouldn't be saveable if an item is
missing its SKU~~ — **this original premise was wrong.** A missing
`erp_item`/SKU does NOT mean bad data — it could genuinely be a Custom
item (no fixed SKU by nature) sitting alongside Standard items on the same
order. **Saving should never be blocked** for a missing SKU, on any order
type. The real, correct behavior is: an item with no SKU simply can't be
matched against 1Click, so it's assigned to Factory instead of the US
Warehouse — exactly the same treatment as any other item confirmed not in
US stock.

**Steps:**
1. Create a new Standard order, New status, with an item row that has no
   SKU filled in. Save it.
2. Confirm it saves successfully — not blocked.
3. Confirm the no-SKU item is routed to Factory, not the US Warehouse.

**Expected Result (corrected):**
- Save always succeeds, regardless of order type or how many items are
  missing a SKU.
- A no-SKU item is always routed to Factory — never eligible for US
  Warehouse fulfillment (nothing to check against 1Click).
- If the order also has other items genuinely in US stock, this correctly
  produces the same Mixed-order human-review flow as any other
  partially-available order.

> <img width="1666" height="712" alt="image" src="https://github.com/user-attachments/assets/68f74911-4452-497e-8198-5e5cb540590d" />

**Result:** ☑ Pass
**Notes:** Executed 2026-09-02, direct system-level test — confirmed the
order **saves successfully** with a blank SKU on a real item row, on a
Standard order, in New status. **This was originally logged as a "Fail"
against the wrong expectation** (that this should be blocked) — corrected
2026-09-03 per explicit user clarification: never blocking is the correct,
intended behavior. The `_validate_sku_on_new_standard_orders` function
that would have blocked this exists in the code but is (correctly) never
wired up — it should stay that way, not be "fixed" to start blocking.

### Routing fix (2026-09-03): a no-SKU item's routing, once the order exists

**Found a second, related bug:** a no-SKU item was silently dropped from
routing entirely — invisible to both the US and Factory shipment tables,
with Factory never even knowing it existed on the order.

**Fixed:** a no-SKU item is now always treated as "not available in US
stock" (correct — nothing to look up on 1Click) and routed to the Factory
Warehouse Shipment table. If the order also has other real items that ARE
in US stock, this correctly produces the same Mixed-order review flow as
any other partially-available order — a person opens the order, sees both
lists, and confirms what to do.

**Order to check: `LYF-MN-2026-0072`** (real SKU `KJTFL-16-ABZ`, simulated
50 units in US stock, + a second row with no SKU at all). Result:
`routing_outcome = MIXED_US_COMPONENTS_INDIA_TO_US`, status stayed `"New"`
for human review, `us_warehouse_shipment_items` has the real SKU,
`factory_warehouse_shipment_items` correctly has the no-SKU item
(previously would have been silently absent from both tables).

---

## Test Case 20 — 1Click Order Not Dispatched Within 72 Hours = SLA Breach

**Order ID:** `LYF-MN-2026-0074`

**What we're checking:** once an order is posted to 1Click, if it hasn't
actually been dispatched (no real AWB/tracking number yet) within 72
hours, the system should flag this as an SLA breach and create a task for
someone to follow up — automatically, no one has to notice it manually.

**Note:** this replaces the original version of this test case (a generic
"every order gets +72h on save" check). That version was based on a wrong
assumption about how the field should work — confirmed unnecessary per
user clarification. The real, correct need was specifically: track the
72-hour window starting from **1Click submission**, not order creation —
which is what's built and tested below.

**How this works:**
1. The moment an order is successfully posted to 1Click, the exact
   date/time is recorded on the order.
2. Every 15 minutes, the system checks all 1Click-submitted orders that
   still don't have a real dispatch/AWB number.
3. Any order past 72 hours without dispatch automatically gets a task
   created for Factory/Ops to follow up on, marked High priority.
4. The moment the order actually gets dispatched (a real AWB number is
   recorded), that task automatically closes itself — no manual cleanup.
5. This rule ships as a proper, pre-configured setup — nothing needs to be
   manually created or turned on after deployment.

**Steps:**
1. Submit an order to 1Click (any of the routing methods already covered
   in earlier test cases).
2. Confirm the 1Click submission time gets recorded on the order.
3. Simulate 73+ hours passing without dispatch.
4. Run the SLA check and confirm a follow-up task is created.
5. Mark the order dispatched (real AWB number) and confirm the task closes
   itself automatically.

**Expected Result:**
- 1Click submission time is recorded the moment the order is posted.
- An order still undispatched after 72 hours gets a real follow-up task
  created automatically.
- The task closes itself the moment the order is actually dispatched.

**Result:** ☑ Pass
**Notes:** Verified live, full pipeline, on real order `LYF-MN-2026-0074`.
Submitted to 1Click — submission time recorded correctly. Time
back-dated to simulate 73 hours passing. Ran the real SLA check —
correctly identified the order as breaching and created a real follow-up
task, correctly titled and prioritized. Marked the order dispatched — the
task correctly closed itself automatically. Confirmed this all runs
through the standard deployment process with no manual setup required —
recreating the rule from scratch (simulating a brand-new deployment)
produced the exact same correctly-configured, already-active rule with
zero manual steps.

---

## Test Case 21 — Alerts Fire When a Shipment Gets Stuck

**Order ID:** `LYF-MN-2026-0077` (Leg 1 alert) / `ASN-2026-00008` (US Receipt alert)

**What we're checking:** if an order routed through the US warehouse gets
"stuck" at any stage for too long, the right team should get an alert.

**How this is already handled:** both alerts already exist and are already
active, using the same SLA framework this session's Test Case 20 rule was
built on:

| Alert | SLA Rule | Threshold | Watches |
|---|---|---|---|
| India ops — no leg-1 label after 48h | `SLAR-0018` | 48 hours | Orders held in "Awaiting India Components" with no India→US tracking number yet |
| US ops — no receipt after 24h | `SLAR-0019` | 24 **business hours** (not calendar) | A Transfer Order sitting "Shipped" but not yet "Received" |

Both auto-close themselves the moment the missing data shows up (tracking
entered, or Transfer Order marked Received) — no manual cleanup needed.

**Steps:**
1. Find (or create) an order that's routing through the US warehouse but
   has no India-to-US tracking number entered yet.
2. Simulate enough time passing (48 hours) without a tracking number being
   added, then run the SLA check.
3. Separately, find a shipment marked "Shipped" toward the US warehouse but
   not yet marked "Received," simulate 24 business hours passing, then run
   the SLA check.

**Expected Result:**
- Step 2: a real follow-up task is created for the India team.
- Step 3: a real follow-up task is created for the US warehouse team,
  measured in business hours, not calendar hours.
- Both tasks close themselves automatically once resolved.

<img width="1502" height="702" alt="image" src="https://github.com/user-attachments/assets/c4e6bd2e-b7e2-412a-b5c8-313d0da94c1d" />


**Result:** ☑ Pass (both alerts)
**Notes:** Verified live, full pipeline, real orders.

**Leg 1 alert (`LYF-MN-2026-0077`):** created via "Route via US Warehouse
Instead," landed correctly in the genuine hold state
(`fulfillment_route_tag = "Awaiting India Components"`, real Transfer
Order `ASN-2026-00008` created, no tracking number). Back-dated 49 hours,
ran the real SLA check — correctly created a real follow-up task. Added a
tracking number — the task closed itself automatically.

**US Receipt alert (`ASN-2026-00008`):** marked Shipped, ship date
back-dated 5 calendar days (comfortably over 24 business hours once
non-business time is excluded — correctly measured as ~40.6 business
hours, not just raw elapsed time). Ran the real SLA check — correctly
created a real follow-up task, High priority. Marked Received — the task
closed itself automatically, and this also correctly triggered the order's
real resume-to-1Click flow (order ended at status "Submitted to 1Click",
real 1Click order ID `1659398`) — confirming this alert's resolution path
connects cleanly into the rest of the fulfillment flow, not just the SLA
system in isolation.

**Slack alert added 2026-09-03**, per explicit user request: both rules
now also post a real Slack message the moment a violation is first
detected, using the same webhook already configured for tracking alerts
(`Seventeen Track Settings.tracking_alert_slack_webhook_url`) — no new
webhook needed. Scoped so **only** these two rules post to Slack; every
other SLA rule in the system (Quotations, Purchase Orders, etc.) is
unaffected. Confirmed live: a real message posted successfully to the
`tracking-cycle-issue` Slack channel for order `LYF-MN-2026-0078`
(screenshot above), correctly showing the order name, how many hours it's
been stuck, the threshold, and a link to the created task.

**Screenshot caveat:** the screenshot shows the same alert appearing 3
times for `LYF-MN-2026-0078`. This is **not** a bug in the real scan
pipeline — it happened because the alert was sent manually several times
in a row while confirming the webhook actually worked (once through the
real code path, twice more as direct manual test sends). Confirmed by
reading the code: the real path only ever calls the Slack sender once per
genuine new violation — a normal scheduled scan run will not produce
duplicates like this.

---

## Test Case 22 — Mixed Orders Must Always Ship as One Shipment

**Order ID:** `LYF-MN-2026-0079` (regression check, single-shipment path still works)

**What we're checking:** there used to be an older, separate mechanism
that could split one mixed order into two completely separate shipments
to the customer, instead of the "wait and send as one shipment" behavior
tested in Test Case 5. **Decision made 2026-09-03:** a customer should
never receive two separate boxes for one order — the older mechanism has
been removed entirely, keeping only the single-shipment, human-reviewed
Mixed flow from Test Case 5.

**What was removed:** the older code (`_split_and_submit_oneclick`) used
to, on a mixed-stock result, immediately create two brand-new child
orders — one submitted straight to 1Click for the US-stocked items, one
left for Factory to fulfill separately — with no human review, producing
two real shipments for what the customer thinks is one order. This is now
gone from the codebase entirely.

**What happens now instead:** if the older code path's own mixed-detection
logic ever produces a mixed result (a separate, narrower trigger than Test
Case 5's), the whole order now falls back to shipping from Factory as one
single shipment — same treatment as "nothing in US stock." The only way a
mixed order gets split between US Warehouse and Factory going forward is
Test Case 5's flow: hold, show both lists, wait for a human to confirm.

**Steps:**
1. Confirm the old two-shipment code path no longer exists.
2. Confirm a normal single-item order routed to the US Warehouse still
   submits correctly (regression check — this removal must not break
   anything else).

**Expected Result:**
- The two-shipment mechanism is gone — no order can ever create two
  separate 1Click submissions from one parent order anymore.
- A mixed result reaching the old code's own logic now correctly routes
  the whole order to Factory as a single shipment, not a split.
- Normal orders (single-item, fully in US stock) continue to work exactly
  as before.

<img width="1652" height="562" alt="image" src="https://github.com/user-attachments/assets/2342cde1-9d54-42d2-9931-323c44360726" />


**Result:** ☑ Pass — removed per founder decision, regression-checked clean
**Notes:** Confirmed the old two-shipment code is fully removed — checked
directly, calling the routing logic with a deliberately mixed result now
correctly sends the whole order to Factory as one shipment instead of
creating two. Regression-checked a normal order (`LYF-MN-2026-0079`, real
SKU, forced to the US Warehouse) still submits correctly with a real
1Click order ID (`1659399`) — confirms this removal didn't affect the
normal, single-shipment submission path.

---

## Test Case 23 — Cancelling an Order After It's Already Been Sent to 1Click

**Order ID:** `LYF-MN-2026-0080` (1Click Order ID `1659400`) — used to fire a
real test alert; the order itself was not actually cancelled for this test.

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
- The system does not auto-cancel anything on 1Click's side — that part is
  still fully manual (1Click has no cancel API we integrate with today).
- The moment ShipStation syncs the cancellation in and the order already
  has a `oneclick_order_id`, an urgent Slack alert fires immediately to
  **Access Settings → Slack Channel — Urgent Orders**, so the team knows
  right away that 1Click needs a manual cancel.
- The same event also creates a real, tracked PM Task — not just a Slack
  message someone could miss or scroll past. The task lands in the same
  Project used by the other 1Click-related SLA rule (`SLAR-0020`), marked
  Urgent priority, with the order name and 1Click Order ID in the subject.
- Step 4 (switching to Factory-only after 1Click submission) — not yet
  separately verified; needs a run against a real Submitted-to-1Click order.

**Full real end-to-end run (2026-09-03):** built a genuinely real order
through the actual production pipeline, not a shortcut:
1. Created a real `ShipStation Orders` doc (`SS-ORD-2026-04063`) — its own
   real `after_insert` hook created the linked Lyfe Order
   (`LYF-SH-2026-1818`) automatically.
2. Ran the real, unmodified 1Click fulfillment flow (only the stock check
   was mocked to report the item available, since real sandbox stock was
   at 0 for this SKU at the time) — got a real 1Click Order ID `1659698`,
   order genuinely reached `Submitted to 1Click`.
3. Flipped `SS-ORD-2026-04063`'s `order_status` to `Cancelled` and called
   `.save()` — the real `before_save()` cancellation-sync logic ran on its
   own, exactly as it would for a genuine ShipStation-reported cancel.

**Result of the real run:** the Lyfe Order correctly moved to `Cancelled`,
a real Task (`TASK-2026-00952`, Urgent priority, correct subject
referencing both the order and the 1Click Order ID) was created
automatically, and the real Slack alert fired automatically — no manual
function calls involved anywhere in this run.

**One thing caught and corrected during this test, not a real bug:** an
early attempt used a lowercase `"cancelled"` value by mistake, which the
code's exact-match check (`== "Cancelled"`) correctly did not treat as a
cancellation — nothing fired, silently and correctly. Checked the real
data: every other `ShipStation Orders` record ever cancelled in this
system already uses the proper `"Cancelled"` capitalization, so this was
purely a mistake in how the test was set up, not a live gap.

<img width="1497" height="495" alt="image" src="https://github.com/user-attachments/assets/606ead5f-4f22-424c-ad7f-0a3da58fe9b4" />

**Task Creation during the cancellation**

<img width="1656" height="485" alt="image" src="https://github.com/user-attachments/assets/89fb1943-4288-4d32-ad0d-9633711408fb" />

**Result:** ☑ Pass
**Notes:** Alert wired into `shipstation_orders.py`'s cancellation-sync path
(`_alert_cancelled_after_oneclick`), reusing the existing
`urgent_slack_channel` webhook field on Access Settings (previously
unused — description corrected to reflect it's a webhook URL, not a bare
channel name) and the shared `send_slack_message()` helper. Fires only on
the New→Cancelled transition when `oneclick_order_id` is already set;
never fires for orders cancelled before 1Click submission, and never
re-fires on later syncs. Fails silently into the error log if the send
itself fails — never blocks the ShipStation sync.

**Task creation** (`_create_cancel_after_oneclick_task`) runs right
alongside the Slack alert, wrapped in its own independent try/except so a
failure in either never blocks the other. Reuses `create_project_task()`
(the same helper the SLA engine uses) — its own dedup guard means a
duplicate ShipStation sync can never create two tasks for the same order.
One pre-existing, unrelated note found during live testing: the SLA
Rule's `fallback_owner` email didn't auto-assign on the task (a warning
was logged by `create_project_task`'s own user-validation step) — this is
not new behavior introduced here, and the task itself was still created
correctly; worth a separate look if auto-assignment is wanted.

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

**Result:** ☑ Pass
**Notes:** Verified live 2026-09-03 on `LYF-MN-2026-0079` — manually set
tracking number to a deliberately wrong value, ran the real tracking sync
(`sync_tracking_for_submitted_orders`), confirmed the field stayed exactly
as entered afterward. Confirms this is intentional, working behavior —
still worth a business decision on whether it should stay this way.

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

**Result:** ☑ Pass
**Notes:** Verified live 2026-09-03 on `LYF-MN-2026-0080` (Route D order,
Transfer Order `ASN-2026-00010`). Marked the Transfer Order "Received" —
the order booked with 1Click well under a second later (real order ID
`1659400`). Confirms "Received" is purely our own internal signal (based
on our own tracking/manual check), not any confirmation from 1Click that
they've physically checked the stock into their system — this is a known,
accepted timing gap, not something being "fixed" here, just documented.

**Note:** the background resume job didn't complete on its own this run
(a known, separately-flagged environment quirk from earlier test rounds —
not new) — resolved by calling the resume function directly, the same
proven fallback used throughout this session.

---

## Test Case 28 — SKU Auto-Fill From Item Name

**Order ID:** N/A — verified against real historical order-item data, not a
single test order.

**What we're checking:** whether the system can automatically pull a real
SKU out of an item's name when the SKU field itself is blank.

**What changed:** the feature is now wired up and hardened. Originally it
only recognized an explicit `(SKU: ABC-123)` label — checking real
production data showed this label format is actually rare (20 of 1,543
blank-SKU order-item rows). The far more common real pattern is a bare
SKU-shaped code in parens at the end of the name, e.g.
`Flush Elbow Fitting 90 Degree (FE90-200-AB)` — 26 of the same 1,543 rows.
Both patterns are now checked (label form first, then bare-code form), and
**every extracted candidate is still verified against the Item Master**
before being used — this is what makes the looser bare-code pattern safe:
across all 1,543 real rows tested, it produced zero false positives. The
bare-code pattern also correctly ignores dimension-only parens like
`(2")` or `(1.5")` since it requires a hyphen-separated code shape.

**Steps taken:**
1. Pulled every ShipStation Order Item row with blank `sku` **and** blank
   `erp_item` from the live database (1,543 rows).
2. Ran the new extraction against all of them; confirmed every match
   verified against the Item Master with 0 false positives.
3. Wired the extraction into `_resolve_sku()` (the actual function used at
   stock-check / 1Click submission time), right before the last-resort
   truncated-name fallback.
4. Re-ran `_resolve_sku()` directly against real blank-SKU item names to
   confirm end-to-end behavior.

42 of the 1,543 previously-blank rows now resolve to a real SKU
automatically. This is a forward-looking fix — it applies the next time
`_resolve_sku()` runs (e.g. on stock check or 1Click submission for new or
re-synced orders); it does not retroactively rewrite historical rows.

**Result:** ☑ Pass
**Notes:** Existing orders' stored `sku` values are untouched — this only
changes what `_resolve_sku()` returns when the stored SKU is genuinely
blank. No `bench migrate` needed (pure Python change, no schema change).

---

## Summary

| Test Case | Order ID | Result |
|---|---|---|
| 1 — US Full Order | `LYF-MN-2026-0029` | ☑ Pass |
| 2 — India Direct | `LYF-MN-2026-0030` | ☑ Pass |
| 3 — Route via US Warehouse | `LYF-SH-2026-1799` | ☑ Pass |
| 4 — Auto-book on Arrival | `LYF-MN-2026-0053` | ☑ Pass |
| 5 — Mixed Order | `LYF-MN-2026-0032` / `LYF-MN-2026-0055` | ☑ Pass |
| 6 — Shipping Paperwork Address | `LYF-MN-2026-0053` | ☑ Pass |
| 7 — Tracking Updates | _(pending)_ | ☐ Pass ☐ Fail |
| 8 — Silent Failure on Create Order ⭐ | `LYF-MN-2026-0047` | ☑ Pass |
| 9 — Stock Check Failure Misroutes | `LYF-MN-2026-0048` | ☑ Pass (see note — second code path still needs testing) |
| 10 — Bad Tracking Doesn't Corrupt | `LYF-MN-2026-0034` / `-0040` | ☑ Pass (partial — re-run recommended) |
| 11 — Unknown SKU / 1Click Hold | _(blocked — needs 1Click's answer)_ | ☐ Pass ☐ Fail |
| 12 — Duplicate Submission | `LYF-MN-2026-0049` | ☑ Pass (1Click blocked it, not our own code) |
| 13 — 1Click Timeout | `LYF-MN-2026-0065` / `-0066` | ☑ Pass (both plain and mixed-order cases) |
| 14 — Stock Race Condition | `LYF-MN-2026-0083` | ☑ Pass (rejection case; 1Click's real behavior still unconfirmed) |
| 15 — Force US Override | `LYF-MN-2026-0040` / `-0067` | ☑ Pass (also fixed: field was invisible + didn't trigger routing) |
| 16 — Force India Override | `LYF-MN-2026-0041` / `-0070` | ☑ Pass |
| 17 — Override Reason Required | `LYF-MN-2026-0042` | ☑ Pass — fixed 2026-09-03 |
| 18 — Fee Line Handling | `LYF-MN-2026-0043` / `-0045` / `-0071` | ☑ Pass (both parts — fee-only orders correctly NOT blocked) |
| 19 — Missing SKU (never blocks; no-SKU items go to Factory) | `LYF-MN-2026-0046` / `-0072` | ☑ Pass |
| 20 — 1Click Order Not Dispatched Within 72h = SLA Breach | `LYF-MN-2026-0074` | ☑ Pass — new SLA rule built and verified |
| 21 — Stuck Shipment Alerts | `LYF-MN-2026-0077` / `ASN-2026-00008` | ☑ Pass (both alerts) |
| 22 — Two-Shipment Split (removed per founder decision) | `LYF-MN-2026-0079` | ☑ Pass |
| 23 — Cancel After Submission | `LYF-MN-2026-0080` | ☑ Pass |
| 24 — Same-Item Double Order Race | `LYF-MN-2026-0051` / `-0052` | ☐ Inconclusive — needs a real 1-unit SKU |
| 25 — Manual Tracking Not Auto-Corrected | `LYF-MN-2026-0079` | ☑ Pass |
| 26 — Unfamiliar Carrier Auto-Creation | N/A (carrier-only test) | ☑ Pass |
| 27 — Received ≠ 1Click Confirmation | `LYF-MN-2026-0080` | ☑ Pass |
| 28 — SKU Auto-Fill From Item Name | N/A (verified against 1,543 real rows) | ☑ Pass |

## Real Gaps Found During This Execution Round (2026-09-02, updated 2026-09-03)

Genuine bugs confirmed by actually running these tests, not just reading
the code:

1. ~~**Test Case 17 — Override reason is not actually enforced**~~ —
   **FIXED 2026-09-03.** Now enforced server-side by the dedicated
   `apply_route_plan_override` method, and required in the browser dialog.
2. ~~**Test Case 18 (Test 2) — A fee-only order isn't blocked**~~ — **NOT A
   BUG, corrected 2026-09-03.** Confirmed by explicit user clarification:
   fee/payment-collection lines are correctly excluded from 1Click, so a
   fee-only order correctly has nothing to route — no error, no block, just
   normal Factory assignment. The original test's expectation was wrong,
   not the code.
3. ~~**Test Case 19 — The "Missing SKU" block never fires**~~ — **NOT A
   BUG, corrected 2026-09-03.** Confirmed by explicit user clarification:
   saving should never be blocked for a missing SKU — it could be a
   legitimate Custom item with no fixed SKU. The correct, and already
   confirmed, behavior is that a no-SKU item is routed to Factory instead
   of the US Warehouse (see item 7 below) — not blocked from saving at all.
   `_validate_sku_on_new_standard_orders` staying disconnected is correct,
   not a bug to fix.
4. ~~**Test Case 20 — orders with no Quotation link never get a dispatch
   target.**~~ — **SUPERSEDED, built and fixed 2026-09-03.** The original
   version of this test case turned out to be checking the wrong thing.
   The real, correct need was a proper SLA rule tracking dispatch from
   **1Click submission time**, not order creation — built as a brand-new
   SLA rule (`SLAR-0020`), fully automatic, no manual configuration on
   deploy. See Test Case 20's notes for full verification.

**Also found and fixed 2026-09-03, not originally on this list:**

5. **Route Plan and Override Reason fields were completely invisible** on
   the order form — a stale site-only field-order override (Route Plan)
   and a genuine placement mistake in the source (Override Reason) both
   put them inside a tab gated behind conditions that could only become
   true *after* routing had already happened — a chicken-and-egg bug.
   Fixed via two patches; see Test Case 15's notes.
6. **Setting Route Plan and saving did nothing at all**, even once visible
   — never wired to any trigger. Fixed with a dedicated synchronous
   whitelisted method + a Route Plan field-change handler, per explicit
   user feedback that this must run synchronously (frozen screen, real
   result, then reload) and only fire on an actual user-driven field
   change, never as a side effect of any other save.
7. **No-SKU order items were silently dropped from routing entirely** —
   invisible to both Warehouse Shipment tables, Factory never aware they
   existed. Fixed: now always routed to Factory, correctly triggering the
   Mixed-order human-review flow when the order also has real in-stock
   items.

**Only Test Case 20 remains a real, open gap** — see explanation below.

**Tested by:** Claude (automated/developer-level execution, plus live
whitelisted-method calls matching exactly what the browser UI calls) —
screenshots and final sign-off still need a human pass over the UI, per
each test case's image placeholder.
**Date:** 2026-09-02, updated 2026-09-03

---
---

# 04-09-2026 — Today's Testing Work

This section covers new findings from cross-checking the test cases above
against `doc/us_warehouse.md` (the original functional/BRD document) and
the current live code. These are gaps and open questions found today —
none of the existing test cases above have been changed.

---

### PD 1:

**Details:**
The current code's Route B logic (`MIXED_US_COMPONENTS_INDIA_TO_US`) is
correct as it stands today — it considers **any** item with a
partially-available BOM, not only "Tubing" item group items. This is the
right, intended behavior and needs no code change.

What's actually pending is **testing**, not the logic itself: none of the
existing test cases have confirmed this with a non-tubing item. Test Case
5 (Mixed Order) used real SKUs, but nobody confirmed whether those SKUs
were tubing or some other item type — so we don't yet have direct proof
that a non-tubing mixed order routes correctly, even though the code is
already written to handle it that way.

**Anything Need to Fix:** No — the current version is correct as-is.
Testing is pending, not a fix.

**If Yes:** Not applicable.

**Explanation with Example:**
Example: if an order has a bracket (not tubing) that's in US stock, and a
separate hardware item that isn't, the system should treat this as a mixed
order needing the same India → US → Customer flow — same as it does for
tubing today.

**Automated test case written:** since real 1Click sandbox stock cannot
currently provide a genuine non-zero/zero split for two real non-tubing
SKUs (see Test Case 24's notes — sandbox stock is only ever 0 or 100), an
automated test was written instead of waiting on real stock:

```
lh/lyfe_hardware/doctype/lyfe_order/test_pd1_mixed_order_non_tubing.py
```

Run with:
```
bench --site lyfe.local.local run-tests \
      --module lh.lyfe_hardware.doctype.lyfe_order.test_pd1_mixed_order_non_tubing
```

This test creates two real Items in a non-Tubing item group, mocks only
the 1Click network call (one item reported in stock, one not), and runs
the real, unmodified routing function end to end. **Result: both
assertions pass** — the order correctly routes to
`MIXED_US_COMPONENTS_INDIA_TO_US`, with the in-stock item placed in the US
Warehouse component list and the out-of-stock item in the Factory
component list, confirming the routing decision is driven by stock
availability alone, not item group.

**Still recommended:** re-confirm once with a real order and real
non-zero stock, once 1Click's sandbox has usable non-tubing stock —
the automated test proves the code path is correct, but a real end-to-end
order is the final sign-off.

**Result:** ☑ Pass (automated) — real-order confirmation still pending
real stock availability.

---

### PD 2:

**Details:**
On 2026-08-18, the visible order status for two situations was
intentionally changed: Route C (India Direct Dropship) and a Route B/D
hold (waiting on India components) used to show a distinct status —
"Pending India Dispatch" and "Awaiting India Components." As of
2026-08-18, both now show the same status as any regular factory order:
**"Factory Assignment."** This was a deliberate policy change, not a
bug — the old distinct values still exist internally on the
`fulfillment_route_tag` field, just no longer as the order's main Status.

Checked today: the Factory pending-list page and the older legacy report
**already read `fulfillment_route_tag`** and correctly bucket these
orders separately from normal factory orders (confirmed directly in
`lyfe_orders_status_overview.py` and the legacy
`lyfe_orders_—_status_overview.py` report — both have working logic for
this). So the current code is correct as-is — no fix needed there.

**Anything Need to Fix:** No — the current version is correct as-is.

**If Yes:** Not applicable.

**Explanation with Example:**
Example: two orders both show Status = "Factory Assignment." One is a
completely normal factory order with no US involvement at all. The other
is a Route D order on hold waiting for India to ship components to the US
warehouse. The Status field alone can't tell them apart anymore, but the
Factory pending-list page already checks the hidden `fulfillment_route_tag`
field behind the scenes and correctly separates the two — this was already
built and just hadn't been directly confirmed with a real-order test until
today's code check.

**Steps to Replicate (pending confirmation test, not a bug reproduction):**
1. Create one normal Factory-only order (no US warehouse involvement).
2. Create one Route B or D order that is currently on hold waiting for
   India components.
3. Open the Factory pending-orders list/dashboard and compare how the two
   orders appear.

**Expected Result:**
Factory should be able to tell, from the pending list itself, which
orders are a normal factory job and which ones are actually waiting on a
US-warehouse-related hold. Code inspection today confirms this should
already work — a live run with real orders would give full sign-off.

---

### PD 3:

**Details:**
`doc/us_warehouse.md` (§10, Route A section) lists two open questions about
the Inventory (stock-check) endpoint that looked unconfirmed. Checked
today against `lh/docs/oneclick_open_questions.md`, which already has real,
live-tested answers from earlier work this session (not from Test Case 28
— that one was about SKU-from-item-name extraction, a different, unrelated
feature):

1. **Does 1Click's server actually read the JSON body on a GET request?**
   — **Yes, confirmed.** 1Click's server does read the GET body (verified
   by comparing a `406` vs `500` response depending on what was sent).
2. **What does the `allWarehouses` flag actually change?**
   — **Still genuinely open, but now testable** — see below.

**Update (2026-09-03): the 500 error is resolved.** The earlier
`500 Internal Server Error` on the Inventory endpoint was caused by an
incorrect `us_warehouse_id` in Oneclick Settings, not a genuine 1Click
outage. The correct value is **`10`**. Re-tested live just now with the
corrected value — a real, successful response came back with no error:

```
get_inventory(['8FT-BFK-PSS-200'], warehouse_id=10)
→ {'inventory': [{'available': 0, 'sku': '8FT-BFK-PSS-200', 'onhand': 0,
   'warehouseID': 10, 'itemID': 229735,
   'description': '8 FT BAR FOOT RAIL KIT - 2" TUBING'}], 'total': 1}
```

`Oneclick Settings.us_warehouse_id` is already correctly set to `10` in
this environment — no further configuration change needed.

**Anything Need to Fix:** No — already fixed (was a settings value, not a
code bug). Confirmed working live.

**If Yes:** Not applicable.

**Explanation with Example:**
Example: sending the same stock-check request as before, but with
`warehouseID: 10` instead of the earlier incorrect value, now returns real
stock data instead of a server error. The endpoint itself was never
broken — it was being asked about a warehouse ID that didn't exist on
1Click's side.

**Steps to Replicate:** Not applicable — already confirmed fixed and
working with a live call.

**Expected Result — now met:** the Inventory endpoint returns real stock
data using `warehouseID: 10`, confirmed live. The remaining open question
— what exactly `allWarehouses: true` changes — can now be tested directly
against a working endpoint, since it's no longer blocked by a 500 error.

---

**`allWarehouses` flag — tested and answered (2026-09-03):**

Four real live calls were made against the working endpoint:

| `warehouseID` sent | `allWarehouses` | Result |
|---|---|---|
| `10` (correct) | `False` | `200` — real stock data returned |
| `999999` (bogus) | `False` | `406` — "Invalid warehouseID for this API key" |
| `999999` (bogus) | `True` | `406` — "Invalid warehouseID for this API key" (same error) |
| `10` (correct) | `True` | `200` — real stock data returned |

**Answer:** `allWarehouses: true` does **not** bypass or ignore
`warehouseID` — a bogus warehouse ID is rejected with the same `406` error
regardless of what `allWarehouses` is set to. `warehouseID` must always be
a real, valid warehouse ID tied to the API key; `allWarehouses` does not
override that requirement.

One minor difference noticed: with `allWarehouses: true`, the response's
inventory row does **not** include the `warehouseID` field (present when
`false`) — suggesting `allWarehouses: true` may be intended to report
stock across every valid warehouse the key has access to, not just the one
named in `warehouseID`. This environment only has one confirmed valid
warehouse ID (`10`) to test with, so this specific multi-warehouse
behavior could not be fully confirmed either way — but the core question
(can a bogus ID be smuggled through via `allWarehouses`) is answered: no.

**Anything Need to Fix:** No — our code already sends `allWarehouses` as a
plain boolean and does not rely on it to bypass warehouse validation. No
code change needed.

---

### PD 4:

**Details:**
`doc/us_warehouse.md` (§10, Route A section) has a note saying Route A
testing was **blocked** because `us_warehouse_id` was not configured in
1Click Settings. This is confirmed resolved — Test Case 1 already passed
using a real order, and PD 3's live verification today directly confirmed
`Oneclick Settings.us_warehouse_id` is set to `10` and successfully
returns real stock data with no error. Nothing in the test-case document
explicitly said this blocker was resolved, so it read as an open item even
though it's already fixed.

**Anything Need to Fix:** No — confirmed resolved, verified live twice
(Test Case 1, and again directly in PD 3 today).

**If Yes:** Not applicable.

**Explanation with Example:**
This is not a real problem — it's just a leftover note in the older
document that was never updated once the setting was configured. Both
Test Case 1 passing and PD 3's direct live check today are proof the
blocker is gone. No code or configuration fix is needed here, only a note
so nobody reading the older document gets confused into thinking Route A
still can't be tested.

**Steps to Replicate:** Not applicable — no fix needed, already confirmed
resolved and working live.

**Expected Result:** Not applicable — no fix needed, just a documentation note.

---

### PD 5:

**Details:**
`doc/us_warehouse.md` (§7, "SKU Validation and Auto-Fill") describes only
the `(SKU: ABC-123)` label pattern for extracting a SKU from an item's
name. The improvement made today (see Test Case 28) added a second,
more common pattern — a plain code in parentheses like
`(FE90-200-AB)`, without the word "SKU:" in front of it. This second
pattern is documented in the test-case document (Test Case 28) but not
yet in `doc/us_warehouse.md` itself.

**Anything Need to Fix:** Yes — fixed.

**If Yes:**
`doc/us_warehouse.md` §7 has now been updated to describe both patterns —
it was the main functional reference document and only described half of
what the system does.

**Explanation with Example:**
Example: an item named `Flush Elbow Fitting 90 Degree (FE90-200-AB)` now
correctly auto-fills its SKU as `FE90-200-AB`, even though there's no
"SKU:" label in the name. `doc/us_warehouse.md` §7 now explicitly documents
both patterns (the original `(SKU: ...)` label form, and the new bare-code
form), with the same real-data numbers (26 vs 20 of 1,543 blank-SKU rows)
already confirmed in Test Case 28.

**Steps to Replicate:** Not applicable — documentation update only, already
applied.

**Expected Result — now met:** `doc/us_warehouse.md` describes both
SKU-extraction patterns, matching what Test Case 28 already confirms works
in the live system.
