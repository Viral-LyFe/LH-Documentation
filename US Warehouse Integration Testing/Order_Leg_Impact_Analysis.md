# Order Leg — Impact Analysis on Existing Test Cases

**Organization:** Lyfe Hardware
**Prepared:** 2026-09-07
**Status:** For discussion — fixes NOT yet applied
**Related commit:** `a6ab023` (branch `prod-us_warehouse_integration`) — "Add Order Leg: per-shipment tracking + gate Completed on all legs delivered"
**Related test case:** TC-BOM-13 in `US_Warehouse_BOM_Kit_Test_Cases.md`

---

## Why this document exists

The Order Leg feature shipped today solves a real, confirmed problem (a Mixed
"Direct to Customer" order has 2 real physical shipments but only one
`tracking_number` field to hold both, and "Completed" fired blind to the
second package).

But it introduced **two side effects that affect orders that were working
fine before** — including several test cases already documented as Pass.
This document lists every genuinely affected area so we can agree on the fix
before changing anything further.

**Nothing in this document has been fixed yet.** The "Pass or Fail" line on
each item is deliberately left open, to be filled in after fixes are applied
and re-verified.

---

## The two root causes (shared by everything below)

### Root cause A — tracking number moved off the order, onto the leg

`save_oneclick_response()` in
`lh/lyfe_hardware/doctype/lyfe_order/lyfe_order.py` now writes the 1Click
tracking number and carrier onto the matching **Order Leg** row instead of
onto `Lyfe Order.tracking_number` / `Lyfe Order.carrier`.

It only falls back to the old order-level write if **no** Order Leg exists —
but every order now gets a leg, so in practice the order-level fields are
never populated any more.

**Why that breaks more than visibility:** `fetch_ready_orders()` in
`order_tracking.py` selects orders to poll by checking that
`tracking_number` and `carrier` are set **on the order**. With those fields
now empty, affected orders are never selected, so they never progress
`Submitted to 1Click → Shipped → Completed`.

This is the exact failure mode the comment at `order_tracking.py:111-120`
documents as a previously-fixed bug ("sat stuck at 'Submitted to 1Click'
forever even once delivered"). It has effectively been re-introduced.

### Root cause B — the Completed gate can strand an order

`all_legs_delivered()` in `lh/lyfe_hardware/doctype/order_leg/utils.py` is
called before writing `status = "Completed"` in both
`order_tracking.py::apply_normalised_to_order` and its duplicate in
`order_tracking_service.py` (the 17Track webhook path).

If an order has an Order Leg whose own `status` is still `Pending`, Completed
is skipped — even when the order's own tracking says Delivered. The Order
Leg's status is only ever advanced by its own separate hourly scheduler
(`order_leg/tracking.py`), reading the **leg's** carrier/tracking fields. Any
path that delivers via the order's own tracking (manual entry, webhook,
existing scheduler) never updates the leg, so the leg stays `Pending`
forever and the order can never complete.

---

## Affected Area 1 — Test Case 15: Force US

**Affected test case details**
`US_Warehouse_Test_Cases.md` → Test Case 15 ("Force US: Manually Routing an
Out-of-Stock Item to the US Warehouse"). Previously ☑ Pass.
Real orders referenced: `LYF-MN-2026-0067` (1Click order `1659393`),
`LYF-MN-2026-0040` (1Click order `1659099`).
Root causes: **A and B**.

**Step to Replicate**
1. Create a real order with a SKU that has 0 US stock.
2. Set Route Plan = "Force US", enter a reason, confirm the dialog.
3. Confirm the order reaches `Submitted to 1Click` with a real 1Click order ID.
4. Open the Lyfe Order and look at its Tracking Number and Carrier fields.
5. Let the hourly tracking scheduler run (or simulate a Delivered response)
   and watch whether the order ever reaches `Completed`.

**How this can create confusion**
- Step 4: the order shows a real 1Click submission but its **Tracking Number
  and Carrier fields are empty**. The tracking data is on a child Order Leg
  record the user has no reason to know exists. The test case's own
  screenshots point at the order-level fields, so the documented evidence no
  longer matches reality.
- Step 5: the order **never reaches Completed**, because it is not selected
  by `fetch_ready_orders()` (blank order-level tracking) and, even if it
  were, the Completed gate blocks on a leg that nothing marked Delivered.
- Ops' likely conclusion: "Force US is broken / 1Click never sent tracking" —
  when in fact the shipment is fine and only our storage location changed.

**What could be the solution**
1. Write tracking to **both** places — keep populating
   `Lyfe Order.tracking_number` / `carrier` exactly as before (this feeds
   `fetch_ready_orders()` and the entire existing status progression), and
   *also* populate the Order Leg for per-leg visibility. Not either/or.
2. When the parent order's tracking reports Delivered and the order has
   exactly **one** leg, mark that leg Delivered too — a single-leg order's
   delivery *is* that leg's delivery. Keep the gate strict only for genuine
   multi-leg orders, which is the only case it was designed for.
3. Update this test case to state that tracking appears in both places, then
   re-verify live.

**Pass or Fail ( will update after fixes )**
_Pending — to be filled in after the fix is applied and re-verified._

---

## Affected Area 2 — Test Case 5: Mixed Order (Via US Warehouse)

**Affected test case details**
`US_Warehouse_Test_Cases.md` → Test Case 5 ("Mixed Order — Some Items From
the US, Some From India"). Previously ☑ Pass.
Real orders referenced: `LYF-MN-2026-0032`, `LYF-MN-2026-0055`,
`LYF-SH-2026-1831`.
Root causes: **A and B**.

**Step to Replicate**
1. Create a Mixed order (some components in US stock, some not).
2. Set Factory Leg Destination = "Via US Warehouse", then Confirm Split.
3. Mark the resulting Transfer Order "Received" (with a submitted MIFO in
   place) so the combined order posts to 1Click as one shipment.
4. Check the Lyfe Order's Tracking Number and Carrier fields.
5. Watch whether the order ever reaches `Completed` after delivery.

**How this can create confusion**
- Same as Area 1: tracking lands on the Order Leg, order-level fields stay
  empty, order never progresses to Completed.
- Extra subtlety worth being explicit about: **before** today, this specific
  path never persisted tracking at all — `_submit_combined_mixed_order`'s
  caller never called `doc.save()`, so the value was silently discarded.
  So a user re-testing sees behaviour different from *both* the documented
  evidence *and* the previously-broken reality. It is genuinely better now
  (the data survives), just stored somewhere the docs don't mention.

**What could be the solution**
Same as Area 1 (write to both places; auto-deliver a sole leg; update docs).
No Mixed-specific handling needed — the "Via US Warehouse" flow produces
exactly one customer-facing leg, so it behaves like any single-leg order.

**Pass or Fail ( will update after fixes )**
_Pending — to be filled in after the fix is applied and re-verified._

---

## Affected Area 3 — Test Case 25: Wrong Manually-Entered Tracking Number

**Affected test case details**
`US_Warehouse_Test_Cases.md` → Test Case 25 ("A Wrong Manually-Entered
Tracking Number Doesn't Get Auto-Corrected").
Real order referenced: `LYF-MN-2026-0079`.
Root cause: **B** (the Completed gate specifically).

**Step to Replicate**
1. Take an order that has been submitted to 1Click (it now has an Order Leg).
2. Manually type a tracking number into the order's own Tracking Number
   field.
3. Run the tracking sync.
4. Observe whether the order can subsequently reach `Completed`.

**How this can create confusion**
- The test's original, narrow point (a manually-entered number is not
  silently overwritten) still works correctly — that part is unaffected.
- But the order now **cannot reach Completed** through this path. The
  manually-set tracking may report Delivered, while the Order Leg's own
  status was never touched and stays `Pending`, so the gate blocks
  indefinitely.
- Ops' likely conclusion: "I entered the tracking number and it's delivered,
  but the order won't close" — with nothing on screen explaining why.

**What could be the solution**
- Solution item 2 from Area 1 resolves this directly: if the parent order's
  tracking reports Delivered and there is exactly one leg, mark that leg
  Delivered as well.
- Optionally, surface the reason on the order when the gate does block a
  multi-leg order ("Waiting on 1 of 2 shipment legs"), so a stuck order is
  self-explanatory rather than silent.

**Pass or Fail ( will update after fixes )**
_Pending — to be filled in after the fix is applied and re-verified._

---

## Affected Area 4 — Test Case 22: note only, not a real conflict

**Affected test case details**
`US_Warehouse_Test_Cases.md` → Test Case 22 ("Mixed Orders Must Always Ship
as One Shipment"). Previously ☑ Pass. Real order: `LYF-MN-2026-0079`
(1Click order `1659399`).

**Step to Replicate**
1. Re-run the regression check that only one 1Click submission occurs for
   the order (no duplicate/second shipment).

**How this can create confusion**
Minimal. This test case's assertion is purely "one submission, no duplicate
shipment" — it never referenced `tracking_number`, `carrier`, or the
Completed transition, so its stated expected result is unaffected. The
tracking-location shift from Root cause A technically applies to this order
too, but nothing this test checks depends on it.

Recorded here only so the same order (`LYF-MN-2026-0079`, which is also used
by Test Case 25) isn't mistaken for unaffected across the board.

**What could be the solution**
No change required for this test case beyond the shared fix in Area 1.

**Pass or Fail ( will update after fixes )**
_Pending — expected to remain Pass; confirm during re-verification._

---

## Confirmed NOT affected

Checked against all four criteria (does it create legs / does it read
order-level tracking / does it depend on Completed / would a re-test differ):

| Test Cases | Why unaffected |
|---|---|
| TC 8, 9, 12, 13, 14 | All failure/`1Click Error` paths. Legs are only created on a *successful* routing outcome, so these never reach leg creation at all. |
| TC 1, 4, 16 | Route to US_FULL/Factory but their expected results only assert routing outcome / "Submitted to 1Click" — never tracking field location or Completed. |
| TC 10 | Concerns tracking-check corruption on un-tracked orders; does not involve the Completed transition. |
| TC 17, 19 | Routing override validation and SKU validation. No tracking fields involved. |
| TC 18 | Fee-line exclusion from the 1Click payload. Unrelated. |
| TC 20, 21 | SLA rules keyed off `oneclick_submitted_at` / Transfer Order status / `fulfillment_route_tag`. None read tracking or the leg gate. |
| TC 23 | Cancellation alert keyed off `oneclick_order_id` + ShipStation status. Unrelated. |
| TC 26 | Carrier auto-creation — fires identically regardless of where tracking is stored. |
| TC 27 | Transfer Order "Received" timing vs real 1Click stock. Not customer-facing tracking. |
| TC 28 | SKU auto-fill. Unrelated code path. |
| TC-BOM-1 … TC-BOM-8 | Routing-outcome and BOM-explosion tests, all mocked at the routing layer — never reach leg creation or tracking. |
| TC 7, TC 24, TC-BOM-8 | Not previously Pass (never run / inconclusive / pending decision), so out of scope for regression. |

---

## Proposed fix summary (single change set)

1. **`save_oneclick_response()`** — write tracking/carrier to the parent Lyfe
   Order **and** the matching Order Leg, rather than choosing one. Restores
   `fetch_ready_orders()` selection and the whole existing status
   progression; keeps new per-leg visibility.
2. **Single-leg auto-delivery** — when the parent order's tracking reports
   Delivered and the order has exactly one Order Leg, mark that leg
   Delivered. The `all_legs_delivered` gate then only ever holds back
   genuine multi-leg orders, which is its actual purpose.
3. **Optional, recommended** — when the gate does block a multi-leg order,
   record a visible reason on the order ("Waiting on 1 of 2 shipment legs")
   so a legitimately-held order is self-explanatory.
4. **Re-verify** Areas 1–4 live afterward, plus re-run all automated suites
   (currently 36 tests), then fill in each "Pass or Fail" line above.

## Open question for discussion

Should a **multi-leg** order's parent `tracking_number` field show the first
leg's tracking, the US leg's specifically, or stay blank with all tracking
visible only per-leg? Item 1 above keeps existing behaviour working, but for
a genuine 2-package order the single order-level field can only ever tell
part of the story — worth deciding deliberately rather than by default.
