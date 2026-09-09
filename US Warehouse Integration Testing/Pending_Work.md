# US Warehouse Integration — Pending Work

**Prepared:** 2026-09-08
**Purpose:** Single checklist for the final round of testing — what still
needs to be re-tested, what proof images are missing, what should be
double-checked directly on the 1Click portal (not just our own logs), and
general sanity checks before sign-off.

Source documents this pulls from:
- `US_Warehouse_Test_Cases.md`
- `US_Warehouse_BOM_Kit_Test_Cases.md`

---

## A. Re-test needed (real gaps or inconclusive results — do these first)

### 1. Test Case 24 — Two Orders for the Same Low-Stock Item at the Same Time
**Status:** ☐ Still inconclusive — **a "Pass" claimed here earlier today
was wrong and has been retracted.** Caught by the user checking the 1Click
portal directly: `LYF-MN-2026-0038` (claimed "correctly rejected as an
oversell") had actually already been successfully created on 1Click
(`1661692`) by an earlier, abandoned test attempt — the later "clean"
re-run was resubmitting a PO 1Click already had, not testing a fresh race.
1Click's rejection was a duplicate-PO rejection, not an oversell
rejection. Full correction written into `US_Warehouse_Test_Cases.md`, Test
Case 24.

**One real, separate issue this surfaced (still open):**
- When 1Click rejects a duplicate PO submission, our error message is a
  bare, unhelpful `"406 Client Error: for url: ..."` with no real reason
  shown to the user — a genuine gap, not just a test-methodology mistake.

(A second suspected issue — our `get_inventory()` reading `available: 10`
not matching the 1Click portal's `100` — turned out to be a stale/cached
portal page on the user's end. A manual refresh confirmed the portal
agrees with our API. No real discrepancy; nothing to fix here.)

**What to do:** Re-run with two genuinely fresh Lyfe Order records (never
reused), check the FULL Integration Request timeline (not just the latest
entry) before concluding anything, and cross-check the real result
directly on the 1Click portal — not just our own API responses — the same
way the user caught this correction.

### ~~2. Test Case 10 — One Bad Tracking Update Doesn't Corrupt Others~~ ✅ Done 2026-09-08
**Status:** ☑ Pass — bug found AND fixed. Healthy order half fully proven
(real order `LYF-SH-2026-1813`, real tracking number, unaffected by
anything). Broken order half initially **failed**: `apply_normalised_to_order`'s
fallback `else` branch treated any unrecognized/garbled tracking response
as "must have shipped," advancing status to "Shipped" off zero real data
— and on a second reproduction (real tracking number `383571424926`, real
FedEx carrier already on 17Track), a garbled response **destroyed real,
correct tracking data** on an already-Shipped order.

**Fixed:** added a guard in `apply_normalised_to_order` (`order_tracking.py`)
— a response with no real signal at all is now skipped entirely instead of
being written or advancing status. Verified all three scenarios post-fix
(fresh-order false-positive, real-data-still-works, real-data-preserved-
against-garbage). New regression suite added:
`test_order_tracking_garbled_response.py` (3 tests, passing). Full existing
suite (11 tests) still passing, no regressions. Committed on
`prod-us_warehouse_integration`.

### ~~3. TC-BOM-6 — BOM Component With No SKU~~ ✅ Done 2026-09-08
**Status:** ☑ Pass — real gap found (2026-09-04), fixed and
regression-tested (2026-09-08). `_explode_order_row_to_components`'s
`item_bom` branch now preserves a no-SKU BOM child row (flagged
`no_sku_item=True`, routed to Factory) instead of silently dropping it,
mirroring the existing plain-row fix. Test inverted to assert correct
behavior; all 9 tests in `test_bom_kit_routing.py` pass; full suite
(41 tests, 6 modules) re-run with no regressions. Live-verified against a
real Lyfe BOM with a genuinely blank-SKU child row.

### 4. TC-BOM-8 — `item_bom` Not Linked After Drawing-Based BOM Creation
**Status:** ☐ Pending your confirmation. The underlying feature ("Parse BOM
from Drawing") lives entirely on a separate git branch, `auto-bom`
(`lh/lyfe_hardware/utils/bom_from_drawing.py`), which is **not merged**
into `prod-us_warehouse_integration` — confirmed live: the button does not
even appear on real orders today (e.g. `LYF-MN-2026-0035`) because the code
simply isn't present on this branch.

**What to do:** Confirm whether `auto-bom` is in scope for this release.
- If yes → needs a real test pass once merged in.
- If no → mark this test case as out-of-scope/deferred for this round.

### 5. Test Case 12 — No pre-submission dedup guard on our side (code-level gap, NOT the same as item 1's error-message issue)
**Status:** ☐ Not fixed. Confirmed real, distinct from item A.1 above —
A.1 is only about making 1Click's duplicate-PO rejection message clearer
(surfaced during the TC 24 investigation); this item is about **never
attempting the second Create Order call at all**.

**What TC 12 found (2026-09-02, order `LYF-MN-2026-0049`):** submitting
the same order to 1Click twice was "safe" — 1Click itself rejected the
second attempt with a real 406, no duplicate order created. But the
protection came **entirely from 1Click's side**. Our own code has no
guard that stops it from attempting a second submission in the first
place — it only finds out the second call failed after actually calling
1Click again. The test case's own notes explicitly flag this: *"Worth
fixing on our side too, so a second attempt is stopped before ever
calling 1Click, rather than relying on them to reject it every time."*

**What to do:** Add a pre-submission check to `create_order()` (or its
callers) — if the order already has a real `oneclick_order_id` set, skip
the Create Order call entirely and surface a clear "already submitted"
message immediately, instead of relying on 1Click's own rejection as the
only safety net. Worth checking whether `apply_route_plan_override`'s
existing `if doc.oneclick_order_id: frappe.throw(...)` guard (used for
Force US/Force India) can be reused or generalized, rather than writing
a second, separate check.

---

## B. Screenshots/proof still missing (behavior already confirmed working — just needs the image)

| # | Test Case | What to screenshot |
|---|---|---|
| ~~5~~ | ~~**TC-BOM-10**~~ | ✅ Done 2026-09-08 — 10.1 screenshot added (field read-only on confirmed order). 10.2 has no UI screenshot by design (field is correctly uneditable in the UI, so the server-side guard is only reachable via script) — re-verified live instead, exact error text captured in the doc. |
| ~~6~~ | ~~**TC-BOM-12**~~ | ✅ Done 2026-09-08 — 2 screenshots already in the doc from the 2026-09-07 live test are sufficient proof; no separate 1Click portal screenshot needed (portal already confirmed directly by the user, see item C.1 below). |
| ~~7~~ | ~~**Test Case 8**~~ | ✅ Done 2026-09-09 — re-verified live on fresh order `LYF-MN-2026-0089` (`LH3026`), real screenshot uploaded showing "1Click Error" status, full traceback, and the raw `"success": false` API response. |
| ~~8~~ | ~~**Test Case 25**~~ | ✅ Done 2026-09-09 — re-verified live on fresh order `LYF-MN-2026-0029`, both screenshots captured (before: "Tracking Pending" message; after: real tracking check ran, field unchanged). |
| ~~9~~ | ~~**Test Case 26**~~ | ✅ Done 2026-09-09 — re-verified live via `_resolve_carrier`, screenshot captured, test record deleted afterward. |
| 10 | **Test Case 27** | Screenshot showing both timestamps (marked-Received vs. actual 1Click stock confirmation) side by side. |
| ~~11~~ | ~~**TC-BOM-16**~~ | ✅ Done 2026-09-09 — both screenshots added (button visible at "US Warehouse Delivered"; order at "Submitted to 1Click" after clicking it). |

---

## C. What to specifically confirm on the 1Click portal (not just our own logs)

Our Integration Request logs only prove **we sent the right call** — the
portal is what proves **1Click actually received and processed it
correctly**. Worth a direct spot-check for each of these:

1. ✅ **Done 2026-09-08** — **TC-BOM-12 / TC-BOM-15** — user confirmed both
   `TC12-VERIFY-MTW4CY` (Item ID `230024`) and `RETRY-TEST-C5FUIO` are real,
   registered items in 1Click's item master. Screenshots still pending for
   the record, but the behavior itself is confirmed.
2. **Test Case 12** — Submitting the Same Order Twice: confirm on 1Click's
   side that a duplicate PO number was actually rejected/deduped by them,
   not just that our code didn't send it twice.
3. **Any "Submitted to 1Click" order from today's testing** (e.g. real
   order IDs `1661566`, `1661097`) — spot-check that these match real
   orders visible in the 1Click portal, with the correct items, quantities,
   and shipping address. This is the strongest possible proof our payload
   mapping is correct end-to-end, not just "no error was thrown" on our
   side.
4. **Test Case 9** (Stock Check Fails) — if this was tested via a simulated
   failure, confirm with 1Click whether their real API has ever actually
   returned this kind of failure in production, so we know the handling
   isn't purely theoretical.
5. **Any order still sitting in "1Click Error" from today's tests** (e.g.
   `LYF-MN-2026-0028`, `LYF-MN-2026-0029`) — confirm on the 1Click side
   these were NOT accidentally created as duplicate/partial orders despite
   showing as failed on our end.
6. **Test Case 11's registered-but-0-stock order-lookup gap — ask 1Click
   directly.** A registered SKU with 0 stock (`MHRB-200-AC`, order
   `LYF-SH-2026-1847` / real 1Click order `1660658`) was accepted by
   Create Order (real HTTP 200, real order ID returned) — but querying
   1Click's own order-status endpoint (`/api/v2/orders`) for that same
   order moments later returned an empty list, `total: 0`, as if the
   order doesn't exist. Confirmed at the raw HTTP level, both via our
   `po` reference and via 1Click's own numeric order ID — ruled out as a
   lookup-key mismatch on our side. **Likely the same root cause as the
   PO-lookup mismatch Kevin's email later surfaced** (their status
   endpoint appears to key off a different internal PO/order number than
   what Create Order echoes back to us) — worth raising both findings
   together in the next message to 1Click: *"Create Order returns a real
   success + order ID, but your own /api/v2/orders status endpoint can't
   find that same order moments later, whether we query by the `po` we
   sent or by the `id` you returned. Confirmed on real orders `1660658`
   and `1664352`/`1662182` (2026-09-08/09) — can you confirm what
   identifier your status endpoint actually expects?"*

---

## D. General backend sanity checks before final sign-off

- Re-run the full automated test suite one more time to catch any
  regression from today's edits:
  ```
  bench --site lyfelocal.com set-config allow_tests true
  bench --site lyfelocal.com run-tests \
        --module lh.lyfe_hardware.doctype.order_leg.test_order_leg \
        --module lh.lyfe_hardware.doctype.lyfe_order.test_bom_kit_routing \
        --module lh.lyfe_hardware.doctype.lyfe_order.test_pd1_mixed_order_non_tubing \
        --module lh.lyfe_hardware.doctype.lyfe_order.test_mixed_order_combined_posting \
        --module lh.lyfe_hardware.doctype.lyfe_order.test_force_us_and_oneclick_error_alert
  bench --site lyfelocal.com set-config allow_tests false
  ```
- Confirm the Lyfe Order tab reorder (CJ Shipping → Warehouse Split →
  1Click Logistics → Connections) looks correct on a couple more real
  orders, not just `LYF-MN-2026-0028`.
- Clean up leftover test data from today's session if it shouldn't stay in
  the live sandbox: test orders `LYF-MN-2026-0028` through `LYF-MN-2026-0036`,
  and their `RETRY-TEST-*` / `TC12-VERIFY-*` / `TC14-*` test SKUs/Items.
  **Do not delete without explicit confirmation** — list and confirm first.

---

## Quick reference — real orders/SKUs created during this testing round

| Order | Purpose | SKU(s) involved |
|---|---|---|
| `LYF-MN-2026-0028` | TC-BOM-15 — SKU auto-register, no auto-resubmit | `RETRY-TEST-C5FUIO` |
| `LYF-MN-2026-0029` | TC-BOM-15 — Force US dialog proactive registration | `RETRY-TEST-O3MJTN` |
| `LYF-MN-2026-0030` | TC-BOM-15 — same, second confirmation | `RETRY-TEST-UBN46L` |
| `LYF-MN-2026-0031` | TC-BOM-14 — first Mixed order attempt (reverted by real-time resync) | `3.5FT-TB-200-SB`, `MHRB-200-AC` |
| `LYF-MN-2026-0034` | TC-BOM-14 — Via US Warehouse, confirmed working | `3.5FT-TB-200-SB`, `MHRB-200-AC` |
| `LYF-MN-2026-0035` (`LH2971`) | TC-BOM-13 — Direct to Customer, 2-leg confirmation | `3.5FT-TB-200-SB`, `MHRB-200-AC` |
| `LYF-MN-2026-0036` | TC-BOM-12 — stock-check-time auto-register re-verification | `TC12-VERIFY-MTW4CY` (1Click Item ID `230024`) |
| `LYF-MN-2026-0037` / `LYF-MN-2026-0038` | Test Case 24 — attempted concurrent test, **invalidated** (see correction) | `3.5FT-TB-200-SB` — both had real 1Click orders (`1661693`, `1661692`); 1Click's later resync cleared their order history, so these POs are free for a fresh re-test |
| `LYF-MN-2026-0039` | Test Case 10 — first broken-response bug demonstration, reverted after confirming | `TC10-BROKEN-TEST-001` (test tracking number, not a real SKU) |
| `LYF-MN-2026-0040` | Test Case 10 — second reproduction: real data destroyed by garbled response, reverted | Real tracking number `383571424926` (real FedEx order `LYF-SH-2026-1841`'s tracking, reused) |
| `LYF-MN-2026-0041` | Test Case 10 — post-fix verification (all 3 scenarios confirmed correct), reverted | Same real tracking number `383571424926` |
| `LYF-MN-2026-0028` (2026-09-09) | Real US_FULL order posted to 1Click after config restore | `3.5FT-TB-200-SB`, real 1Click order `1662182` |
| `LYF-MN-2026-0029` (`LH2968`) | Test Case 25 — wrong manually-entered tracking number stays unchanged | Test tracking number `WRONGTRACK123456` (not a real SKU issue) |
