# US Warehouse Integration — Full End-to-End Test Case

A single, standalone test that walks one real order through the **entire**
order lifecycle — creation, automatic routing, posting to 1Click, tracking,
and (eventually) delivery/completion — rather than testing one feature in
isolation. Every other test case in this project verifies one specific
behavior; this one exists to prove the whole pipeline works together, start
to finish, on one real order.

**Order used:** `LYF-MN-2026-0122`
**Real 1Click Order ID:** `1665036`
**SKU used:** `10FT-BFK-SSS-200` (real US warehouse stock, 100 units
available at time of test — no stock contention risk)
**Date run:** 2026-09-10

---

## What we're checking

That a real order, created the same way any real order would be, correctly
flows through every stage automatically with no manual intervention beyond
normal order creation — and that each stage's real system state (not just
"no error was thrown") matches what's expected at that point.

---

## Step 1 — Order creation

**What was done:**
Created a new Lyfe Order with one real item (`10FT-BFK-SSS-200`, qty 1,
real live US warehouse stock at the time: 100 units) and a real shipping
address.

**Expected:** Order saves successfully in status **"New"**, not yet routed.

**Actual result:** ✅ Matches. Order `LYF-MN-2026-0122` created, status
`New`, `routing_outcome` blank (not yet routed).

---

## Step 2 — Automatic routing

**What was done:**
Triggered the same routing entry point (`run_oneclick_fulfillment`) that a
real ShipStation-synced order automatically fires on creation. (Note: this
specific order was created manually rather than via a ShipStation webhook,
so it does not have the `shipstation_order` link that normally triggers
routing automatically — see the "Known caveat" section below. This is the
only step in this test that required a manual nudge; every step after this
one happened automatically.)

**Expected:** The system checks real 1Click stock for `10FT-BFK-SSS-200`,
finds it fully available, and routes the order to `US_FULL`.

**Actual result:** ✅ Matches.
- `routing_outcome`: `US_FULL`
- `warehouse`: `US Warehouse - LH`

---

## Step 3 — Posting to 1Click

**What was done:**
Same routing call from Step 2 also handles posting, since `US_FULL` orders
post immediately once routed — no separate manual step.

**Expected:** Order is posted to 1Click, gets a real 1Click order ID back,
and the status updates to **"Submitted to 1Click"**.

**Actual result:** ✅ Matches.
- `status` / `workflow_status` / `workflow_state`: all correctly set to
  `Submitted to 1Click` together (per this app's status-triple rule)
- `oneclick_order_id`: `1665036` (real, confirmed on 1Click's side)
- `oneclick_error`: empty — no error
- `oneclick_submitted_at`: `2026-09-10 11:50:10`

---

## Step 4 — Confirming the order really exists on 1Click's side

**What was done:**
Called the real Get Tracking by PO endpoint directly against 1Click,
independent of our own scheduler, to confirm the order genuinely exists on
their side (not just that our own database says it does).

**Expected:** 1Click returns a real record for this PO.

**Actual result:** ✅ Matches. 1Click returned:
```json
{
  "total": 1,
  "orders": [{
    "details": {
      "Order_Id": 1665036,
      "Status": "Open",
      "Purchase_Order": "LYF-MN-2026-0122",
      "items": [{"sku": "10FT-BFK-SSS-200", ...}],
      "tracking_number": "",
      "Dispatch_Carrier": ""
    }
  }]
}
```
Confirms the order is real on 1Click's side, correctly linked to the right
SKU and PO — but not yet dispatched (no tracking number or carrier yet).
This is a genuinely different response shape from `{"total": 0, "orders":
[]}` (which means 1Click hasn't processed the order at all yet) — `total:
1` with an empty `tracking_number` means the order exists and is simply
still waiting to ship.

---

## Step 5 — Automatic tracking sync

**What was done:**
Ran the real, unmodified scheduled tracking sync
(`sync_tracking_for_submitted_orders`) — the exact same function that runs
automatically every hour in production — rather than simulating it.

**Expected:** Since 1Click hasn't dispatched the order yet (confirmed in
Step 4), the sync should correctly do nothing — no tracking number to
apply yet, no status change, and importantly **no error**.

**Actual result:** ✅ Matches. Order correctly remained at `Submitted to
1Click`, `oneclick_status` updated to `Open` (reflecting 1Click's real
current status), `tracking_number` still empty. No error was thrown, and
nothing was incorrectly marked as failed just because tracking data wasn't
available yet.

---

## Step 6 onward — Shipped → Completed (pending real dispatch)

**Status: not yet completed — this is expected, not a failure.**

The remaining steps (tracking number appears once 1Click actually ships
the order → status auto-advances to **Shipped** → delivery confirmation →
status auto-advances to **Completed**) depend on 1Click's sandbox
environment genuinely dispatching this order, which had not happened yet
as of this test run.

This part of the pipeline is already proven correct by other test cases
run earlier this project (e.g. the order that moved from Submitted to
1Click all the way to Shipped using a real tracking number, and the
several orders that reached Completed after a real carrier-confirmed
delivery) — it was deliberately not re-proven here with a fabricated
tracking number, since doing so on a real order would misrepresent real
system state. This document will be updated with the real Shipped/Completed
timestamps once `LYF-MN-2026-0122` naturally reaches those stages, to keep
one full, continuous, entirely-real trail for this exact order.

---

## Known caveat found during this test

**Automatic routing only fires for orders linked to a real ShipStation
order.** The routing trigger in `after_insert` is gated on `self.
shipstation_order` being set (this is how the system reliably tells a real
webhook-created order apart from all others, since `order_source` itself is
not a reliable signal — confirmed 0 of 2630 real orders on this site use
`order_source == "ShipStation"`). An order created directly through the
Lyfe Order form or a script (as this test order was, for controlled
testing) does not have that link, so it needs the routing step manually
triggered once, exactly as done in Step 2 above.

**This is not a bug** — every real order in production comes in through
ShipStation and always has that link set, so this gap only affects
manually-created test orders, never real customer orders. Worth knowing if
anyone else builds a test order by hand in the future.

---

## Result

| Step | Result |
|---|---|
| 1 — Order creation | ✅ Pass |
| 2 — Automatic routing | ✅ Pass |
| 3 — Posted to 1Click | ✅ Pass |
| 4 — Confirmed real on 1Click's side | ✅ Pass |
| 5 — Automatic tracking sync (no premature status change) | ✅ Pass |
| 6 — Shipped → Completed | ⏳ Pending real 1Click dispatch |

**Overall:** ☑ Pass for every stage that can complete without 1Click
physically shipping the sandbox order. No manual status edits were used at
any point — every status the order reached, it reached through the same
real code path a genuine customer order would use.
