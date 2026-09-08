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

**Two real, separate issues this surfaced (both still open):**
- When 1Click rejects a duplicate PO submission, our error message is a
  bare, unhelpful `"406 Client Error: for url: ..."` with no real reason
  shown to the user — a genuine gap, not just a test-methodology mistake.
- Our `get_inventory()` API reading (`available: 10`) did not match what
  the user saw on the actual 1Click portal (`100`) — not yet explained;
  needs investigation into what "available" means in the API response vs.
  the portal UI.

**What to do:** Re-run with two genuinely fresh Lyfe Order records (never
reused), check the FULL Integration Request timeline (not just the latest
entry) before concluding anything, and cross-check the real result
directly on the 1Click portal — not just our own API responses — the same
way the user caught this correction.

### 2. Test Case 10 — One Bad Tracking Update Doesn't Corrupt Others
**Status:** ☑ Pass, but with a caveat — only the "broken response doesn't
corrupt data" half was proven. Neither real order used (`LYF-MN-2026-0034`
control / `LYF-MN-2026-0040` simulated failure) had a real tracking number
on file at test time, so the "healthy order still updates normally
alongside a broken one" half was never actually confirmed.

**What to do:** Re-run with one order that already has a real tracking
number on file, run alongside a simulated-broken-response order in the same
tracking check pass. Confirm the healthy order's tracking still updates
correctly.

### 3. TC-BOM-6 — BOM Component With No SKU
**Status:** ☑ Real gap confirmed — **not yet fixed**. This is a genuine
known bug, not a testing gap.

**What to do:** Decide whether to fix this before final sign-off, or
explicitly accept and document it as a known limitation for this release.

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

---

## B. Screenshots/proof still missing (behavior already confirmed working — just needs the image)

| # | Test Case | What to screenshot |
|---|---|---|
| 5 | **TC-BOM-10** | 10.1: Factory Leg Destination field shown read-only/greyed-out on a confirmed Mixed order's form. 10.2: the validation error shown when attempting to change it after Confirm Split (via form save or API). |
| 6 | **TC-BOM-12** | Reference already captured: `LYF-MN-2026-0036`, SKU `TC12-VERIFY-MTW4CY`, 1Click Item ID `230024`. Go to the **1Click portal** and screenshot that item master record to prove the auto-registration reached 1Click's own system, not just our Integration Request logs. |
| 7 | **Test Case 8** | Result line already says Pass (`LYF-MN-2026-0047`, 2026-09-02) — confirm the existing embedded screenshot in the doc is actually from this order's form; replace if not. |
| 8 | **Test Case 25** | Screenshot of the order before and after the tracking check, showing the wrong manually-entered tracking number stayed unchanged. |
| 9 | **Test Case 26** | Screenshot of the new `"UPSS-TYPO-TEST-26"` Carrier record in the list. |
| 10 | **Test Case 27** | Screenshot showing both timestamps (marked-Received vs. actual 1Click stock confirmation) side by side. |

---

## C. What to specifically confirm on the 1Click portal (not just our own logs)

Our Integration Request logs only prove **we sent the right call** — the
portal is what proves **1Click actually received and processed it
correctly**. Worth a direct spot-check for each of these:

1. **TC-BOM-12 / TC-BOM-15** — SKUs `TC12-VERIFY-MTW4CY` (Item ID `230024`)
   and `RETRY-TEST-C5FUIO` should now appear as real registered items in
   1Click's item master.
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
| `LYF-MN-2026-0037` / `LYF-MN-2026-0038` | Test Case 24 — attempted concurrent test, **invalidated** (see correction) | `3.5FT-TB-200-SB` — both have real 1Click orders (`1661693`, `1661692`); test itself does not prove/disprove the race condition |
