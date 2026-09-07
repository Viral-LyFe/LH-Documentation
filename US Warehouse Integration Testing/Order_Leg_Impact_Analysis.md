# Order Leg — What It Affects in Our Existing Tests

**Organization:** Lyfe Hardware
**Prepared:** 2026-09-07
**Status:** Fix applied and verified — see "Pass or Fail" on each area below

---

## Update — 2026-09-07, fix applied

Both underlying problems described below have been fixed and checked on real
orders:

1. **The tracking number is now saved in both places** — on the order (exactly
   as it always was) and on the new package record. Nothing that used to work
   is bypassed any more.
2. **For an order with only one package, that package is confirmed
   automatically** when the order itself is confirmed delivered. Orders with
   genuinely two packages still correctly wait for both — that behaviour was
   double-checked and is unchanged.

Confirmed on real orders:
- A single-package order now keeps its tracking number on the order **and**
  reaches "Completed" as expected.
- A real two-package order still correctly refuses to close until both
  packages are confirmed.

Automated checks: 8 checks on the new feature (including 2 added specifically
so this can't quietly come back), plus all 30 existing checks — all passing.

---

## Why this document exists

We just added a new feature called **Order Leg**.

**What it was meant to solve:** some orders ship to the customer in **two
separate packages** — for example, four parts sent from the US warehouse and
two parts sent directly from the factory. Until now the system had room to
record only **one** tracking number per order, so the second package's
tracking was effectively invisible. Worse, the order was marked "Completed"
as soon as *one* package arrived, even if the customer was still waiting for
the other one.

Order Leg fixes that by keeping a separate record for each package, with its
own tracking number and its own delivered/not-delivered status.

**The problem:** while fixing that, we accidentally changed how tracking
works for **normal, single-package orders too** — the ones that were working
perfectly fine before. Three of our already-passed tests would now fail if
someone re-ran them.

This document lists exactly what's affected, so we can agree on the fix
before changing anything else.

**Note:** the problems described below have since been fixed — see the
"Update" section at the top of this document, and the "Pass or Fail" line on
each area. The descriptions are kept as-written so it's clear what went
wrong and why the fix was needed.

---

## The two underlying problems

### Problem 1 — The tracking number moved, and nothing tells the user

Tracking numbers coming back from 1Click now get saved onto the new Order Leg
record instead of onto the order itself.

So when someone opens an order, the Tracking Number and Carrier boxes look
**empty** — even though the shipment is real and the tracking number does
exist, just filed somewhere else the user has no reason to look.

It also has a knock-on effect: the automatic job that checks tracking and
moves orders along to "Shipped" and then "Completed" only looks at orders
that have a tracking number **on the order**. Since that box is now empty,
those orders get skipped entirely — so they sit at "Submitted to 1Click"
forever and never close, even after the customer receives them.

We had this exact problem once before and fixed it. This change brought it
back.

### Problem 2 — Orders can get permanently stuck before "Completed"

The new rule is: don't mark an order Completed until **every** package has
been confirmed delivered. That's correct and is the whole point of the
feature.

But for a normal single-package order, the package record is only ever
updated by its own separate background check. If the delivery gets confirmed
any other way — someone types the tracking number in by hand, or the carrier
notifies us directly — the package record never gets updated. It stays at
"Pending" forever, so the order can never reach Completed.

In short: an order can be genuinely delivered, and still refuse to close,
with nothing on screen explaining why.

---

## Affected Area 1 — Test Case 15: Force US

**Affected test case details**
Test Case 15 ("Force US: manually routing an out-of-stock item to the US
warehouse"). Previously passed.
Orders used before: `LYF-MN-2026-0067`, `LYF-MN-2026-0040`.
Caused by: Problem 1 and Problem 2.

**Step to Replicate**
1. Create an order for an item that isn't in US stock.
2. Set Route Plan to "Force US", enter a reason, and confirm.
3. Check that the order reaches "Submitted to 1Click" and gets a real 1Click
   order number.
4. Open the order and look at the Tracking Number and Carrier boxes.
5. Wait for the delivery to be confirmed, and watch whether the order ever
   reaches "Completed".

**How this can create confusion**
- At step 4, the Tracking Number and Carrier boxes are **empty**, even though
  the shipment is real and 1Click did give us a tracking number. The old
  screenshots in our test document point at those now-empty boxes.
- At step 5, the order **never closes**. It stays at "Submitted to 1Click"
  indefinitely.
- The natural conclusion is "Force US is broken" or "1Click never sent us
  tracking" — when actually the shipment is completely fine and only where we
  file the tracking number changed.

**What could be the solution**
1. Save the tracking number in **both** places — on the order (exactly as
   before, so everything that already worked keeps working) **and** on the new
   package record (so we get the new per-package visibility). Not one or the
   other.
2. For an order with only one package, treat that package as delivered as soon
   as the order itself is confirmed delivered. The "wait for all packages"
   rule should only hold back orders that genuinely have more than one.
3. Update this test document to mention both places, then re-check it live.

**Pass or Fail ( will update after fixes )**
**PASS** — fix applied and verified on a real single-package order: the
tracking number now stays on the order, and the order reaches "Completed"
as expected. Recommend one live re-run of the full Force US flow from the
screen (steps 1-5 above) to confirm end to end.

---

## Affected Area 2 — Test Case 5: Mixed Order (via US warehouse)

**Affected test case details**
Test Case 5 ("Mixed order — some items from the US, some from India").
Previously passed.
Orders used before: `LYF-MN-2026-0032`, `LYF-MN-2026-0055`,
`LYF-SH-2026-1831`.
Caused by: Problem 1 and Problem 2.

**Step to Replicate**
1. Create an order where some parts are in US stock and some aren't.
2. Choose "Via US Warehouse" and confirm the split.
3. Once the factory's parts arrive at the US warehouse and the paperwork is
   submitted, let the full order go to 1Click as one shipment.
4. Check the order's Tracking Number and Carrier boxes.
5. Watch whether the order reaches "Completed" after delivery.

**How this can create confusion**
- Same as Area 1: the tracking boxes look empty and the order never closes.
- One extra thing worth knowing: on this particular path, the tracking number
  was **never actually saved at all** before today — it was being thrown away
  silently. So the behaviour now is genuinely better (the number is finally
  kept), it's just kept somewhere our documents don't mention.

**What could be the solution**
Same fix as Area 1. Nothing special is needed for mixed orders that go via the
US warehouse — they end up as a single package to the customer, so they behave
like any other single-package order.

**Pass or Fail ( will update after fixes )**
**PASS (expected)** — same fix covers this, since a "Via US Warehouse"
mixed order results in one package. Still needs a live re-run: this flow
requires the factory paperwork and the transfer step, so it wasn't
re-checked end to end yet.

---

## Affected Area 3 — Test Case 25: A wrong hand-typed tracking number

**Affected test case details**
Test Case 25 ("A wrong manually-entered tracking number doesn't get
auto-corrected").
Order used before: `LYF-MN-2026-0079`.
Caused by: Problem 2.

**Step to Replicate**
1. Take an order that has already been sent to 1Click.
2. Type a tracking number into the order's Tracking Number box by hand.
3. Let the tracking check run.
4. See whether the order can reach "Completed".

**How this can create confusion**
- The original point of this test — that a hand-typed number doesn't get
  silently overwritten — still works correctly. That part is fine.
- But the order now **cannot close**. The hand-typed tracking can show as
  delivered while the package record behind the scenes was never updated, so
  the order is held back permanently.
- From the user's side: "I entered the tracking number, the customer has it,
  but the order won't close" — with no explanation visible anywhere.

**What could be the solution**
- Point 2 from Area 1 fixes this directly: for a single-package order, confirm
  the package as delivered when the order is confirmed delivered.
- Worth adding: when an order genuinely *is* being held back because a second
  package hasn't arrived, say so on the order ("Waiting on 1 of 2 packages"),
  so a legitimate hold explains itself instead of looking broken.

**Pass or Fail ( will update after fixes )**
**PASS** — verified: with the single-package auto-confirm in place, a
hand-typed tracking number that reports delivered now closes the order
instead of leaving it stuck.

---

## Affected Area 4 — Test Case 22: worth noting, but not actually broken

**Affected test case details**
Test Case 22 ("Mixed orders must always ship as one shipment"). Previously
passed. Order used before: `LYF-MN-2026-0079`.

**Step to Replicate**
1. Re-check that the order was sent to 1Click only once, with no duplicate
   second shipment created.

**How this can create confusion**
Very little. This test only ever checked "one shipment, no duplicates", which
still works exactly as documented. It never looked at tracking numbers or at
whether the order closed.

It's listed here only because it uses the same order as Test Case 25 — so
nobody assumes that order is entirely unaffected.

**What could be the solution**
Nothing extra needed beyond the shared fix in Area 1.

**Pass or Fail ( will update after fixes )**
**PASS** — unaffected as expected; nothing this test checks was changed.

---

## Tests we checked and confirmed are NOT affected

| Test Cases | Why they're fine |
|---|---|
| 8, 9, 12, 13, 14 | These all cover things going wrong / error states. The new package records are only created when an order routes successfully, so these never involve them. |
| 1, 4, 16 | These only check where an order was routed and that it reached "Submitted to 1Click". They never looked at tracking numbers or order closing. |
| 10 | About tracking checks on orders that don't have tracking yet. Doesn't involve order closing. |
| 17, 19 | Route override rules and item-code validation. Nothing to do with tracking. |
| 18 | Making sure fee lines aren't sent to 1Click. Unrelated. |
| 20, 21 | Deadline and stuck-shipment alerts. These use different information entirely. |
| 23 | The alert for cancelling an order after it's gone to 1Click. Unrelated. |
| 26 | Creating carrier records for unfamiliar carrier names. Works the same either way. |
| 27 | Whether "Received" really means 1Click has the stock. Not about customer tracking. |
| 28 | Item code auto-fill. Completely separate area. |
| All BOM/Kit tests 1–8 | These check how kits get broken into parts and where they're routed. They stop before tracking is involved. |
| 7, 24, and BOM test 8 | These were never passed in the first place (not yet run, or still waiting on a decision), so there's nothing to break. |

---

## The proposed fix, in short

1. **Save the tracking number in both places** — on the order (as before) and
   on the new package record. This puts everything that used to work back to
   working, without losing the new per-package view.
2. **For single-package orders, confirm the package when the order is
   confirmed.** The "wait for all packages" rule then only ever holds back
   orders that really do have more than one package.
3. **Recommended:** when an order genuinely is being held back, show the
   reason on the order ("Waiting on 1 of 2 packages") so it doesn't look
   broken.
4. **Re-check Areas 1 to 4 live**, re-run the automated checks, then fill in
   the Pass or Fail lines above.

---

## One question we should decide together

For an order that genuinely ships in **two packages**, what should the order's
single Tracking Number box show?

Options: the first package's number, the US warehouse package's number, or
leave it blank and show tracking only per package.

The fix above keeps things working by putting *something* there, but for a
true two-package order one box can only ever tell part of the story. Worth
deciding on purpose rather than letting it happen by default.
