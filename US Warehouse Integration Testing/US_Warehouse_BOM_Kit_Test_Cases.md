# US Warehouse Integration — BOM / Kit Order Test Cases

**Organization:** Lyfe Hardware
**Prepared:** 2026-09-04
**Status:** DRAFT — for review before building `.py` test cases

---

## Why this document exists

All testing done so far in `US_Warehouse_Test_Cases.md` used order items where
the ordered SKU itself is what 1Click checks stock for — a plain,
non-BOM item.

This document covers the case that has **not** been tested yet: an order item
that is a **kit**, made of multiple individual components defined in a
**Lyfe BOM**. When this happens, the system doesn't check stock for the kit's
own SKU — it explodes the kit into its individual components (via
`explode_and_check_bom_availability()` in `lyfe_order_routing.py`) and checks
stock for each component separately. Some components might be sitting in the
US warehouse; others might not — meaning a single kit item can itself become
a "mixed" situation, on top of the order-level mixed case already covered by
Test Case 5.

## Background — how a row becomes BOM-driven

| Field | Where | What it does |
|---|---|---|
| `item_bom` | `ShipStation Order Item` (used for both `order_items` and `extra_items` on Lyfe Order) | Link to a `Lyfe BOM`. If set, that row's stock check explodes into the BOM's `child_items` instead of checking the row's own SKU. |
| `bom_not_required` | Same child table | Existing escape hatch — explicitly skip BOM checking for this row even if it looks like it should have one. |

**How `item_bom` actually gets set today — confirmed by reading the code:**
- It is **not** auto-filled anywhere on ShipStation sync. ShipStation has no
  concept of this field.
- It **is preserved** across re-syncs — once someone sets it, a later
  ShipStation sync will not wipe it out (`shipstation_orders.py`, the
  `saved_item_boms` restore logic).
- The **"Parse BOM from Drawing" feature** (branch `auto-bom`,
  `lh/lyfe_hardware/utils/bom_from_drawing.py`) reads the order's attached
  `preview_pdf`, extracts BOM tables, and creates/updates a `Lyfe BOM` record
  — but it does **not** write `item_bom` back onto the originating order row.
  **This is a real, confirmed gap**: even after using this feature to
  generate a BOM from the drawing, someone still has to manually link that
  BOM to the order row via `item_bom` before routing will actually use it.
  See PD-BOM-1 below.

**Component explosion logic** (`_explode_order_row_to_components`,
`lyfe_order_routing.py`):
- If `row.item_bom` is set → explode `Lyfe BOM.child_items` (each has its own
  `sku`, `item_code`, `quantity`) — quantities multiply by the row's own
  ordered quantity.
- If not set → the row itself is the only "component" (existing behavior,
  already tested in Test Cases 1–5).
- A component with no SKU at all can never be confirmed in US stock — it
  routes straight to Factory (existing fix from earlier this session, see
  Test Case 19's related note).

**Routing decision** (`compute_routing_outcome`):
```
All components (across every row, BOM-exploded or not) in US stock?
  → US_FULL
Some in US stock, some not?
  → MIXED_US_COMPONENTS_INDIA_TO_US   (generalized, not tubing-specific — see PD 1)
None in US stock?
  → INDIA_DIRECT_DROPSHIP or INDIA_TO_US_TO_CUSTOMER (existing Route C/D logic)
```

This means a kit whose components are split between US and Factory stock
produces exactly the same `MIXED_US_COMPONENTS_INDIA_TO_US` outcome as a
plain multi-row mixed order — the routing engine doesn't distinguish
"mixed because of one row's BOM" from "mixed because of two separate plain
rows." That's the core thing worth proving with real test cases: **the BOM
explosion path and the plain-row path must produce identical, correct
behavior downstream** (same two Warehouse Shipment tables, same hold state,
same Transfer Order creation).

---

## Standard format for every test case below

```
### TC-BOM-N — <title>

**What we're checking:** ...
**Order shape:** ...
**Steps:** ...
**Expected Result:** ...
```

---

### TC-BOM-1 — Single Kit Item, All Components in US Stock

**What we're checking:** an order with one kit item (BOM-linked), where
every one of its components is in US stock, routes exactly like a normal
`US_FULL` order — no unexpected Factory/mixed behavior just because it's a
kit.

**Order shape:** one `order_items` row, `item_bom` set to a real `Lyfe BOM`
with 2–3 `child_items`, all components in US stock.

**Steps:**
1. Create/confirm a `Lyfe BOM` where every child SKU has real US stock (or
   mock `get_inventory` to report all as available, matching this session's
   proven testing pattern where real sandbox stock isn't reliably available).
2. Create an order with one row linked to that BOM.
3. Run routing.

**Expected Result:**
- `routing_outcome = US_FULL`.
- `US Warehouse Shipment Item` table contains one row per BOM component
  (not one row for the kit itself) with correctly multiplied quantities.
- `Factory Warehouse Shipment Item` table is empty.
- Order proceeds straight to 1Click submission, same as any other US_FULL
  order.

---

### TC-BOM-2 — Single Kit Item, Some Components in US Stock, Some Not

**What we're checking:** the core new scenario — one kit item whose
components split between US and Factory triggers the Mixed route on its
own, without needing a second order row.

**Order shape:** one `order_items` row, `item_bom` set to a BOM with (for
example) 3 child components: 2 in US stock, 1 not.

**Steps:**
1. Build the BOM/order as above, with a deliberately mixed stock picture.
2. Run routing.
3. Check both Warehouse Shipment tables and the routing outcome.

**Expected Result:**
- `routing_outcome = MIXED_US_COMPONENTS_INDIA_TO_US`.
- `US Warehouse Shipment Item` contains the 2 in-stock components.
- `Factory Warehouse Shipment Item` contains the 1 out-of-stock component.
- `source_order_item` and `bom_reference` on each shipment-item row
  correctly point back to the originating order row and the BOM used
  (this is what the original plan document called out as the reason
  `bom_reference` exists at all — multiple BOMs on one order must stay
  attributable to the correct row).
- Order holds for human review — nothing auto-booked to 1Click yet, same
  as Test Case 5's existing expected behavior.

---

### TC-BOM-3 — Two Kit Items, Different BOMs, Each Fully in US Stock

**What we're checking:** an order with two separate kit rows, each backed
by its **own distinct** `Lyfe BOM`, both fully stocked in the US — should
still resolve to a clean `US_FULL`, and each row's components must not get
mixed up with the other row's.

**Order shape:** two `order_items` rows, each with a different `item_bom`,
no shared SKUs between the two BOMs.

**Steps:**
1. Create two Lyfe BOMs with non-overlapping child SKUs, all in US stock.
2. Create one order with both rows.
3. Run routing.

**Expected Result:**
- `routing_outcome = US_FULL`.
- `US Warehouse Shipment Item` contains all components from both BOMs.
- Each shipment-item row's `source_order_item` correctly identifies which
  of the two order rows it came from — this is the direct test of
  `bom_reference`/`source_order_item` attribution across multiple BOMs on
  one order, which nothing has exercised yet.

---

### TC-BOM-4 — Two Kit Items, Different BOMs, Only One Fully Stocked

**What we're checking:** with two separate kits on one order, if one kit's
components are entirely in US stock and the other kit's components are
entirely not, does the order still correctly resolve to Mixed (not
silently treat the fully-stocked kit as the whole order's answer, and not
silently treat the fully-unstocked kit as blocking the whole order to
Factory)?

**Order shape:** two `order_items` rows, two different BOMs — Kit A: 100%
in US stock. Kit B: 0% in US stock.

**Steps:**
1. Build as described.
2. Run routing.

**Expected Result:**
- `routing_outcome = MIXED_US_COMPONENTS_INDIA_TO_US`.
- `US Warehouse Shipment Item` contains 100% of Kit A's components.
- `Factory Warehouse Shipment Item` contains 100% of Kit B's components.
- No component from Kit A leaks into the Factory table or vice versa.

---

### TC-BOM-5 — Kit Item + Plain (Non-BOM) Item on the Same Order

**What we're checking:** a real-world shape — one row is a kit (BOM-linked),
another row is a plain single SKU with no BOM at all. Both paths
(`_explode_order_row_to_components`'s BOM branch and its plain-row branch)
must combine correctly into one routing decision.

**Order shape:** Row 1 — kit item, `item_bom` set, components split
between US/Factory. Row 2 — plain item, `item_bom` blank, in US stock.

**Steps:**
1. Build the order as described.
2. Run routing.

**Expected Result:**
- `routing_outcome = MIXED_US_COMPONENTS_INDIA_TO_US` (Row 1's Factory-bound
  component alone is enough to trigger Mixed, regardless of Row 2).
- `US Warehouse Shipment Item` contains Row 1's in-stock component(s) AND
  Row 2 itself (as its own single component, correctly not BOM-exploded).
- `Factory Warehouse Shipment Item` contains Row 1's out-of-stock
  component(s) only.
- This is the test that most directly answers the user's framing —
  "orders where Custom and Standard both are available in order_item" —
  since a kit/BOM row behaves like a Custom-style multi-part item while a
  plain SKU row behaves like a Standard single item, coexisting on one
  order.

---

### TC-BOM-6 — Kit Item Where a BOM Component Itself Has No SKU

**What we're checking:** the existing no-SKU-item fix (component with no
SKU routes straight to Factory rather than vanishing) still works when the
no-SKU component comes from inside a BOM explosion, not just a plain order
row.

**Order shape:** one kit item, BOM has one child row with a blank SKU (data
quality issue in the BOM itself — legitimate real-world case, e.g. a BOM
component added before its Item Master record existed).

**Steps:**
1. Create a `Lyfe BOM` with one `Custom BOM Items` child row missing `sku`.
2. Build an order using this BOM, with at least one other component that
   does have real US stock.
3. Run routing.

**Expected Result:**
- The no-SKU component appears in `Factory Warehouse Shipment Item` (never
  silently dropped), consistent with the existing fix for plain order
  rows.
- `routing_outcome = MIXED_US_COMPONENTS_INDIA_TO_US` if any other
  component in the order is in US stock (same "naturally produces the
  Mixed flow" behavior the original fix note describes).

---

### TC-BOM-7 — Kit Item Fully Resumed After Transfer Order Received

**What we're checking:** end-to-end — a Mixed kit order actually completes
the full Route B lifecycle: hold → Transfer Order created with the
Factory-bound BOM components → Transfer Order marked Received → order
resumes and submits the **complete** kit (US components + now-arrived
Factory components) to 1Click as one shipment.

**Order shape:** same as TC-BOM-2 (one kit, split stock).

**Steps:**
1. Build and route the order as in TC-BOM-2, confirming the hold state.
2. Confirm a Transfer Order was created, its `items` populated from
   `factory_warehouse_shipment_items` (the BOM-exploded Factory
   components, not the parent kit row).
3. Mark the Transfer Order Received.
4. Confirm `_maybe_resume_oneclick_order` resubmits successfully with a
   real 1Click Order ID.

**Expected Result:**
- Transfer Order's `items` table shows the individual BOM components
  needed from Factory — not "1 x Kit Item" as a single opaque line. This
  is important for the India/Factory team, who need to know exactly which
  physical parts to ship, not just "the kit."
- After Received, the order status moves off hold and 1Click submission
  succeeds with a real order ID, mirroring Test Case 27's already-proven
  pattern but for a BOM-driven order instead of a plain multi-row one.

---

### TC-BOM-8 — `item_bom` Manually Cleared After a BOM Was Already Auto-Created from a Drawing

**What we're checking:** the confirmed gap noted above — using "Parse BOM
from Drawing" creates/updates a `Lyfe BOM` but does **not** link it back to
the order row's `item_bom`. This test confirms what actually happens if
that manual link step is skipped.

**Order shape:** one order row, drawing attached, "Parse BOM from Drawing"
used to create a real `Lyfe BOM`, but `item_bom` deliberately left blank on
the order row (simulating someone forgetting the manual step).

**Steps:**
1. Run the drawing-to-BOM flow, confirm a `Lyfe BOM` was created.
2. Do **not** set `item_bom` on the order row.
3. Run routing.

**Expected Result:**
- The row is treated as a plain, non-BOM item — its own SKU (the parent
  kit's SKU, if one exists) is checked directly against 1Click, **not** its
  components.
- If the parent kit SKU itself has no real 1Click stock entry (likely,
  since kits are usually not stocked as a single unit), the row incorrectly
  routes to Factory even though its individual components might genuinely
  be available in the US warehouse.
- **This confirms the gap is real and has a visible, wrong-routing
  consequence** — not just a cosmetic omission. Worth deciding whether to
  build an explicit "link this BOM to the order row" step into the
  drawing-parsing confirm flow, or an explicit warning/reminder in the UI.

---

## Summary table (to fill in once test cases are executed)

| Test Case | Order/BOM Used | Result |
|---|---|---|
| TC-BOM-1 — Single Kit, All US Stock | _(pending)_ | ☐ Pass ☐ Fail |
| TC-BOM-2 — Single Kit, Mixed Stock | _(pending)_ | ☐ Pass ☐ Fail |
| TC-BOM-3 — Two Kits, Both Fully US Stock | _(pending)_ | ☐ Pass ☐ Fail |
| TC-BOM-4 — Two Kits, One Fully US / One Fully Factory | _(pending)_ | ☐ Pass ☐ Fail |
| TC-BOM-5 — Kit + Plain Item Together | _(pending)_ | ☐ Pass ☐ Fail |
| TC-BOM-6 — BOM Component With No SKU | _(pending)_ | ☐ Pass ☐ Fail |
| TC-BOM-7 — Full Resume Lifecycle for a Kit Order | _(pending)_ | ☐ Pass ☐ Fail |
| TC-BOM-8 — `item_bom` Not Linked After Drawing-Based BOM Creation | _(pending)_ | ☐ Pending confirmation this is a real gap |

---

## Open questions for you before I build the `.py` test cases

1. **TC-BOM-8** — is this a known, accepted manual step today (someone
   always links `item_bom` by hand after using "Parse BOM from Drawing"),
   or is this a genuine gap that should get a follow-up fix (auto-link
   `item_bom` to the newly created/updated BOM as part of
   `confirm_bom_from_drawing`)? I don't want to build a "bug" test case
   for something that's actually working as intended.
2. Should the `.py` tests use **mocked** 1Click stock responses (the proven
   pattern from this session, since real sandbox stock keeps draining to
   0/100 with nothing usable in between), or do you want to hold off until
   real stock is available for genuine end-to-end runs?
3. Any specific real `Lyfe BOM` records you want these tests built around
   (existing kits already in the system), or is it fine for me to create
   fresh, disposable test BOMs the same way `test_pd1_mixed_order_non_tubing.py`
   created disposable test Items?
