# 1Click / US Warehouse Integration Test — Proof of Execution

**Site:** `lyfe.local.local`
**Date run:** 2026-08-24
**Executed by:** Administrator (via `bench console`, scripts `_test_a1_scratch.py` / `_test_a2_scratch.py` / `_test_a3_scratch.py`)

This single document replaces the three earlier per-test-case files (`oneclick_test_proof_A1_2026-08-24.md`, `oneclick_test_proof_A2_2026-08-24.md`, `oneclick_test_proof_A3_2026-08-24.md`) — merged at your request, content unchanged.

**Common test approach across all cases:** since the real 1Click API is not configured on this site (`Oneclick Settings.api_key` is empty), every test below ran against the **real order-routing and 1Click-submission code** in `lyfe_order.py`, with only the 4 outbound network calls in `oneclick_api.py` (`get_settings`, `get_inventory`, `create_order`, `get_order_status`) replaced by fakes for the duration of each run, monkey-patched onto the `oneclick_api` module and restored in a `finally` block. Dummy US stock is read from `stock_update_dummy_50.csv`'s `units_per_case` column, per explicit instruction to use that file as the stand-in for "what's in US warehouse stock" during testing.

---

## Table of Contents

- [A1 — Route A — US Full (all items in US stock)](#a1--route-a--us-full-all-items-in-us-stock)
- [A2 — Route C — India Direct Dropship](#a2--route-c--india-direct-dropship)
- [A3 — Override: "Route via US Warehouse Instead" — RESOLVED](#a3--override-route-via-us-warehouse-instead--resolved-no-discussion-required)
- [GAP RESOLVED — Business Workflow Mismatch (FIXED 2026-08-25)](#gap-resolved--business-workflow-mismatch-fixed-2026-08-25)
- [Related Fixes Made During This Testing Session](#related-fixes-made-during-this-testing-session)

**Note (2026-08-25): all "Route via US Warehouse Instead" gaps found in A2.6/A3 below are fixed.** No discussion required on this topic — jump straight to [GAP RESOLVED](#gap-resolved--business-workflow-mismatch-fixed-2026-08-25) for the fix summary.

---

# A1 — Route A — US Full (all items in US stock)

**Test order created:** `LYF-MN-2026-0034` — **kept live in the UI, not deleted**, per explicit instruction

## A1.1 Test Setup

| Function faked | Replaced with |
|---|---|
| `get_settings()` | Fake config dict (fake API key, warehouse IDs, endpoints) |
| `get_inventory()` | Reads `stock_update_dummy_50.csv`'s `units_per_case` column as "available qty" per SKU |
| `create_order()` | Returns a canned success response: `{"content": {"success": [{"id": "TEST-ONECLICK-ORDER-123"}]}}` |
| `get_order_status()` | Returns an empty tracking response |

Every other function in the real code path — `compute_routing_outcome`, `check_us_inventory`, `assign_warehouse`, `create_oneclick_order`, `_submit_single_oneclick_order`, `save_oneclick_response` — ran unmodified, exactly as it would in production.

**Dummy stock used for the two test SKUs** (from `/home/frappe/frappe-bench/stock_update_dummy_50.csv`, `units_per_case` column):

| SKU | Dummy US Stock Available | Ordered Qty |
|---|---|---|
| `3.5FT-TB-200-SB` | 10 | 2 |
| `KJTFL-16-ABZ` | 100 | 5 |

Both ordered quantities are well within the dummy-available quantity — this is the deliberate setup for Route A ("all items in US stock").

## A1.2 Test Order Created

**`LYF-MN-2026-0034`**

| Field | Value |
|---|---|
| Order Source | Manual |
| Customer | Mackenzy Melendez |
| Ship To Country | United States |
| Created | 2026-08-24 10:16:12 |

Order Items:

| SKU | Item Name | Qty | Item Group | Fulfillment Source |
|---|---|---|---|---|
| `3.5FT-TB-200-SB` | 3.5FT-TB-200-SB | 2 | Products | US Warehouse |
| `KJTFL-16-ABZ` | KJTFL-16-ABZ | 5 | Products | US Warehouse |

(`item_group = "Products"` was set manually after the run, at request, purely for UI display/testing — it played no role in the routing decision itself.)

## A1.3 Result Immediately After the Test Run

Actual output captured from the console run, before any manual edits:

```
DUMMY STOCK for test SKUs: 3.5FT-TB-200-SB = 10 | KJTFL-16-ABZ = 100
CREATED ORDER: LYF-MN-2026-0034

--- RESULT ---
routing_outcome: US_FULL
warehouse: US Warehouse - LH
two_leg_required: 0
fulfillment_route_tag:
status / workflow_status / workflow_state: Submitted to 1Click / Submitted to 1Click / Submitted to 1Click
oneclick_order_id: TEST-ONECLICK-ORDER-123
oneclick_raw_response present: True
oneclick_error: None
  item: 3.5FT-TB-200-SB qty: 2 fulfillment_source: US Warehouse
  item: KJTFL-16-ABZ qty: 5 fulfillment_source: US Warehouse

KEPT FOR UI REVIEW (not deleted): LYF-MN-2026-0034
```

## A1.4 Pass/Fail Assessment

| Expectation for Route A (US Full) | Actual Result | Pass? |
|---|---|---|
| `routing_outcome` = `US_FULL` | `US_FULL` | ✅ |
| `warehouse` = `US Warehouse - LH` | `US Warehouse - LH` | ✅ |
| No split (`two_leg_required` = 0) | `0` | ✅ |
| Every item tagged `fulfillment_source = US Warehouse` | Both items tagged `US Warehouse` | ✅ |
| Order submitted to 1Click (status flips to `Submitted to 1Click`) | `Submitted to 1Click` on all 3 fields (`status`/`workflow_status`/`workflow_state`) | ✅ |
| 1Click response parsed and `oneclick_order_id` stored | `TEST-ONECLICK-ORDER-123` | ✅ (fake ID, since API was mocked) |
| No error recorded | `oneclick_error = None` | ✅ |

**Result: PASS.**

## A1.5 Known Limitation — Scope of What This Proves

- Proven: given "these SKUs are fully in US stock," the real routing/submission/status-update logic in `lyfe_order.py` behaves correctly end-to-end.
- **Not proven:** whether the real 1Click API accepts the payload shape this code builds, or returns a response in the shape `save_oneclick_response` expects. This cannot be verified until `Oneclick Settings.api_key` is configured. This is the explicitly agreed tradeoff for testing without live API access.

## A1.6 Post-Test Note — Status Drift Since the Run

As of writing this record, `LYF-MN-2026-0034`'s `status`/`workflow_status`/`workflow_state` showed **`Completed`** at one point, not the `Submitted to 1Click` the test actually produced. This order was deliberately left live in the UI for review — the state change happened afterward (e.g. via a workflow action taken while inspecting it in the UI), not as part of the test itself. The test's own result (§A1.3 above) is the actual proof of the code's behavior at the moment it ran.

---

# A2 — Route C — India Direct Dropship

**Test order created:** `LYF-MN-2026-0035` — **kept live in the UI, not deleted**, per explicit instruction

## A2.1 Test Setup

Same mocking approach as A1.

| Function faked | Replaced with |
|---|---|
| `get_settings()` | Fake config dict |
| `get_inventory()` | Reads dummy stock from `stock_update_dummy_50.csv`'s `units_per_case` column |
| `create_order()` | Canned response — **deliberately included as a trip-wire**: Route C must never call this, so if it fires, the fake `id` (`SHOULD-NOT-BE-CALLED`) would show up in `oneclick_order_id` and immediately expose the bug |
| `get_order_status()` | Fake empty tracking response |

**Important routing-logic correction found while preparing this test:** the original test-plan description of Route C ("India ships direct because the destination is outside the US") is **no longer how the code decides this**, per the 2026-08-18 policy change documented directly in `lyfe_order_routing.py::_india_route()`. The current rule is:

> Whenever nothing is in US stock, `INDIA_DIRECT_DROPSHIP` is now the **default** outcome — regardless of destination country, including for US customers. `INDIA_TO_US_TO_CUSTOMER` (Route D) only happens if a human explicitly opts in via the pre-existing `order_via_us_warehouse` checkbox.

So this test used a non-US destination (India) for clarity, matching the classic description, but the actual trigger for `INDIA_DIRECT_DROPSHIP` was "nothing in US stock + `order_via_us_warehouse` unchecked" — not the country.

**SKU used:** `7FT-TB-150-PB` — a real, active Item on the site, deliberately chosen because it is **not present at all** in `stock_update_dummy_50.csv`. The faked `get_inventory()` therefore correctly reports `0` available for it, guaranteeing "nothing in US stock."

| SKU | Dummy US Stock Available | Ordered Qty |
|---|---|---|
| `7FT-TB-150-PB` | 0 (not in dummy stock file) | 3 |

## A2.2 Test Order Created

**`LYF-MN-2026-0035`**

| Field | Value |
|---|---|
| Order Source | Manual |
| Ship To Country | India |
| `order_via_us_warehouse` | 0 (unchecked — default) |

Order Items:

| SKU | Item Name | Qty | Item Group |
|---|---|---|---|
| `7FT-TB-150-PB` | 7FT-TB-150-PB | 3 | Products |

(`item_group = "Products"` set manually after the run, for UI display consistency with A1 — no role in the routing decision.)

## A2.3 Result Immediately After the Test Run

Actual captured console output:

```
DUMMY STOCK for test SKU: 7FT-TB-150-PB = 0 (0 = not in US stock)
CREATED ORDER: LYF-MN-2026-0035

--- RESULT ---
routing_outcome: INDIA_DIRECT_DROPSHIP
warehouse: Factory - LH
two_leg_required: 0
fulfillment_route_tag: Pending India Dispatch
status / workflow_status / workflow_state: Factory Assignment / Factory Assignment / Factory Assignment
oneclick_order_id: None
oneclick_raw_response present: False
oneclick_error: None
  item: 7FT-TB-150-PB qty: 3 fulfillment_source:

KEPT FOR UI REVIEW (not deleted): LYF-MN-2026-0035
```

## A2.4 Pass/Fail Assessment

| Expectation for Route C (India Direct Dropship) | Actual Result | Pass? |
|---|---|---|
| `routing_outcome` = `INDIA_DIRECT_DROPSHIP` | `INDIA_DIRECT_DROPSHIP` | ✅ |
| `warehouse` = `Factory - LH` (no US Warehouse leg at all) | `Factory - LH` | ✅ |
| No 1Click submission — order does not touch the US Warehouse or 1Click | `oneclick_order_id = None`, `oneclick_raw_response = False` (the trip-wire fake `create_order()` was never called) | ✅ |
| No split (`two_leg_required` = 0) | `0` | ✅ |
| `fulfillment_route_tag` = `"Pending India Dispatch"` (hidden marker for this specific route, per the 2026-08-18 policy change) | `Pending India Dispatch` | ✅ |
| Visible status behaves like a **normal Factory order** — not a special hold state | `status`/`workflow_status`/`workflow_state` all = `Factory Assignment` — the normal Factory flow, exactly as documented in the code's own comment at `run_oneclick_fulfillment`'s `INDIA_DIRECT_DROPSHIP` branch | ✅ |
| No error recorded | `oneclick_error = None` | ✅ |

**Result: PASS.**

## A2.5 Known Limitation — Scope of What This Proves

- Proven: given "nothing in US stock, no US-warehouse opt-in," the real routing logic correctly identifies Route C, sets `warehouse = "Factory - LH"`, skips 1Click entirely, and lets the order flow through the normal Factory-Assignment auto-logic exactly like a non-1Click order would.
- Confirmed via the trip-wire: **`create_order()` was never invoked** for this route — the fake ID (`SHOULD-NOT-BE-CALLED`) does not appear anywhere in the result, proving the code correctly skips 1Click submission for this route rather than accidentally submitting it.
- Not tested here: the `INDIA_TO_US_TO_CUSTOMER` (Route D) opt-in path, or the generalized mixed-stock split (Route B) — those are separate test cases.

## A2.6 Follow-up Test — "Route via US Warehouse Instead" Button — RESOLVED, no discussion required

> **Status update (2026-08-25): the gap documented below (§A2.6.2/§A2.6.3) has been fixed.** No further discussion needed on this topic — see the full fix write-up in [GAP RESOLVED](#gap-resolved--business-workflow-mismatch-fixed-2026-08-25) below. The original test narrative is kept as-is for the historical record of how the gap was found.

**Note on `LYF-MN-2026-0035`:** by the time this follow-up was written, this order's `routing_outcome` had already changed to `INDIA_TO_US_TO_CUSTOMER` in the live UI (someone had already exercised the button on it, or a similar path, since the A2 run above). Confirmed live by attempting to re-call `route_via_us_warehouse('LYF-MN-2026-0035')` directly — it correctly refused with *"Only applicable to an India-direct order (current routing_outcome: INDIA_TO_US_TO_CUSTOMER)"*. This is expected, correct guard behavior, not a bug — it just meant a **second, fresh order** was needed to actually demonstrate the button from a clean `INDIA_DIRECT_DROPSHIP` starting state.

### A2.6.1 What the button does, end to end (confirmed against the real code, `lyfe_order.py::route_via_us_warehouse`)

This button appears under the **"Warehouse Split"** button group, only when `routing_outcome == "INDIA_DIRECT_DROPSHIP"` and `cj_awb_no` is empty (no shipment committed with a carrier yet).

Clicking it opens a confirmation popup. Clicking **Yes** does the following, in order:

1. **Screen freezes** with the message *"Re-routing via US Warehouse..."* while the server works.
2. **Server runs `route_via_us_warehouse()`**:
   - Re-checks both guards server-side (`routing_outcome == INDIA_DIRECT_DROPSHIP`, `cj_awb_no` empty) — blocks with an error if either fails.
   - Deletes any **Draft** Gate Pass / Sales Invoice / Purchase Order tied to this order (only Draft — a submitted one blocks the whole action with an error instead).
   - Sets `order_via_us_warehouse = 1` and `routing_outcome = "INDIA_TO_US_TO_CUSTOMER"`, `two_leg_required = 1`, saves the order.
   - Calls `_hold_for_india_components(doc, "INDIA_TO_US_TO_CUSTOMER")`, which:
     - **If `factory_warehouse_shipment_items` has rows** (components genuinely missing from US stock): creates a new **Transfer Order** (Draft) listing those components, back-links them to it, sets `warehouse = "Factory - LH"` and `fulfillment_route_tag = "Awaiting India Components"`. Visible `status`/`workflow_status`/`workflow_state` stay `"Factory Assignment"` (per the 2026-08-18 policy — no special 1Click-hold status shown).
     - **If that table is empty** (see §A2.6.2 below — this is the path this test actually hit): falls back to `assign_warehouse(doc); create_oneclick_order(doc)` — i.e. submits the order to 1Click immediately instead, exactly like a normal US_FULL order.
3. **Green toast**: *"Order re-routed via the US Warehouse — a Transfer Order has been created."*
4. **Form reloads** (`frm.reload_doc()`) so the updated `routing_outcome`, `warehouse`, and Transfer Order are visible immediately.

### A2.6.2 Test run — real result (root cause found and fixed, see below)

**Test order:** `LYF-MN-2026-0036` — a fresh, simple order (no BOM/`item_bom` set on its line), created starting from `routing_outcome = INDIA_DIRECT_DROPSHIP` (same setup as A2 — SKU `7FT-TB-150-PB`, 0 dummy US stock).

Ran the actual `route_via_us_warehouse()` function directly (this is exactly what the confirmation popup's "Yes" calls server-side) — **first attempt genuinely crashed**, not simulated:

```
frappe.exceptions.ValidationError: Password not found for Oneclick Settings Oneclick Settings api_key
```

Traceback confirmed this came from `_hold_for_india_components` → its **fallback branch** → `create_oneclick_order(doc)` → `get_settings()`, i.e. it tried to make a real, live call to 1Click (unmocked) because `doc.factory_warehouse_shipment_items` was empty on this simple order.

**Why the table was empty:** `factory_warehouse_shipment_items` is populated by the BOM-explosion logic (`explode_and_check_bom_availability`) earlier in `run_oneclick_fulfillment`'s Auto-routing path — but this test order had no `item_bom` set on its line, so nothing was ever exploded into that table in the first place. **This is a real, reachable state** — any plain, non-BOM order with nothing in US stock lands here.

**RESOLVED (2026-08-25):** the button's docstring/comment said "a Transfer Order will be created" and the confirmation-dialog text told the user the same thing — but for this exact, real scenario (a simple order, no BOM components tracked), no Transfer Order was getting created at all; it silently fell through to submitting the order straight to 1Click as if it were a normal US_FULL order, skipping the "wait for the Transfer Order to be received" hold entirely. Confirmed this was not a mocking artifact — the crash was against the real, unmocked `create_order()`/`get_settings()` call, proving the code really did attempt an immediate 1Click submission in this branch. **Fixed** — see [GAP RESOLVED](#gap-resolved--business-workflow-mismatch-fixed-2026-08-25) below for the fix.

**Completed the run** by mocking `get_settings()` / `create_order()` / `get_order_status()` only for this one fallback call (matching the same trip-wire-safe approach as A1/A2), to see the intended end state:

```
AFTER FALLBACK COMPLETES:
 routing_outcome: INDIA_TO_US_TO_CUSTOMER
 warehouse: Factory - LH
 fulfillment_route_tag: Pending India Dispatch
 status/workflow_status/workflow_state: Submitted to 1Click / Submitted to 1Click / Submitted to 1Click
 oneclick_order_id: TEST-ONECLICK-ORDER-FALLBACK-456
```

Note `fulfillment_route_tag` still read `"Pending India Dispatch"` (left over from the original Route C state) rather than being cleared or updated to reflect this order was now `Submitted to 1Click` — a minor inconsistency that became moot once the root-cause fix landed (the fallback that produced this stale value no longer fires for a genuinely-missing-components order; see [GAP RESOLVED](#gap-resolved--business-workflow-mismatch-fixed-2026-08-25)).

**Order `LYF-MN-2026-0036` kept live in the UI, not deleted**, per standing instruction — available for direct inspection.

### A2.6.3 Summary — both points below fixed, no discussion required

1. ~~For a plain order with no missing BOM components, "Route via US Warehouse Instead" does **not** create a Transfer Order and does **not** put the order on hold — it immediately submits to 1Click instead, contradicting the button's own confirmation text.~~ **FIXED** — see [GAP RESOLVED](#gap-resolved--business-workflow-mismatch-fixed-2026-08-25).
2. ~~`fulfillment_route_tag` is left stale (`"Pending India Dispatch"`) after this fallback path completes, even though the order has moved on to `Submitted to 1Click`.~~ Moot — the fallback this stale value depended on no longer fires for a genuinely-missing-components order; the fix in §GAP RESOLVED means `factory_warehouse_shipment_items` is correctly populated before this point, so the real Transfer-Order-creation branch runs instead.

---

# A3 — Override: "Route via US Warehouse Instead" — RESOLVED, no discussion required

> **Status update (2026-08-25): the root cause documented in this section (§A3.4.1/§A3.7) has been fixed.** No further discussion needed on this topic — see [GAP RESOLVED](#gap-resolved--business-workflow-mismatch-fixed-2026-08-25) below. The original test narrative is kept as-is for the historical record of how the gap was found and confirmed.

**Test order created:** `LYF-MN-2026-0038` — **kept live in the UI, not deleted**, per standing instruction

**Note on order numbering:** the test was originally run once, creating `LYF-MN-2026-0037`. At request, that order was then deleted and the identical test re-run from scratch to confirm the result reproduces. Frappe's naming series does not reuse a deleted number, so the re-run order was auto-named `LYF-MN-2026-0038` instead. The re-run produced an **identical result** to the original `-0037` run (same routing outcome, same fallback-branch behavior, same root cause) — confirming the finding below is reproducible, not a one-off fluke.

## A3.1 Why a fresh order was needed (not `LYF-MN-2026-0036`)

`LYF-MN-2026-0036` (used for the A2.6 ad-hoc button demo) is **not reusable**: by the time this test started, its `routing_outcome` had already advanced to `INDIA_TO_US_TO_CUSTOMER` (it was pushed all the way to `Submitted to 1Click` in the A2.6 follow-up). The button's own server-side guard requires `routing_outcome == "INDIA_DIRECT_DROPSHIP"` — re-attempting on `0036` would just repeat the same guard error already confirmed in A2.6. A new order was created instead: **`LYF-MN-2026-0038`**.

## A3.2 Test Setup — deliberately different from the `LYF-MN-2026-0036` demo

The A2.6 ad-hoc demo used a **plain order row with no BOM** — which meant `factory_warehouse_shipment_items` was empty, and the button fell into `_hold_for_india_components`'s **fallback** branch (submits straight to 1Click, no Transfer Order).

This time, the order row was deliberately given a real `item_bom` link (`Lyfe BOM` = `2FT-BFK-PN-150`, whose parent Item is also `2FT-BFK-PN-150`) — the intent was to reach the button's **other** branch: genuinely missing components → real Transfer Order created.

**BOM components used** (all confirmed real, active Items, and deliberately **not present** in `stock_update_dummy_50.csv` — guaranteeing 0 US stock for all of them):

| SKU | Role | Dummy US Stock |
|---|---|---|
| `2FT-BFK-PN-150` | Parent / ordered item | 0 |
| `CMBR-150-PN` | BOM child component | 0 |
| `2FT-TB-150-PN` | BOM child component | 0 |
| `FEC-150-PN` | BOM child component | 0 |

Same mocking approach as A1/A2 — only the 4 `oneclick_api` network functions faked; all routing/submission logic in `lyfe_order.py` ran unmodified. `create_order()` was faked with an obviously-wrong ID (`SHOULD-NOT-BE-CALLED-STEP1` / `-STEP2`) at each step, as a trip-wire — if it ever showed up in `oneclick_order_id`, that would prove an unwanted 1Click submission occurred.

## A3.3 Test Order Created

**`LYF-MN-2026-0038`**

| Field | Value |
|---|---|
| Order Source | Manual |
| Ship To Country | India |
| `order_via_us_warehouse` | 0 (unchecked — default) |

Order Items:

| SKU | Item Name | Qty | Item Group | `item_bom` |
|---|---|---|---|---|
| `2FT-BFK-PN-150` | 2FT-BFK-PN-150 | 1 | Products | `2FT-BFK-PN-150` |

## A3.4 Step 1 — Initial routing run (same as A2's setup)

Actual captured output:

```
DUMMY STOCK for BOM parent + components: {'2FT-BFK-PN-150': 0, 'CMBR-150-PN': 0, '2FT-TB-150-PN': 0, 'FEC-150-PN': 0}
CREATED ORDER: LYF-MN-2026-0038

--- STEP 1: BEFORE BUTTON CLICK ---
routing_outcome: INDIA_DIRECT_DROPSHIP
warehouse: Factory - LH
fulfillment_route_tag: Pending India Dispatch
status/workflow_status/workflow_state: Factory Assignment / Factory Assignment / Factory Assignment
factory_warehouse_shipment_items: 0
us_warehouse_shipment_items: 0
```

`routing_outcome = INDIA_DIRECT_DROPSHIP` is correct and matches A2. **But `factory_warehouse_shipment_items` is empty here too** — despite this order genuinely having a BOM with 3 components, all confirmed at 0 US stock. This was unexpected going into the test, since the whole point of using a BOM this time was to populate that table.

### A3.4.1 Root cause — confirmed directly against the code (not a mocking artifact)

In `run_oneclick_fulfillment()` (`lyfe_order.py`, lines ~4027–4069):

```python
if outcome == "INDIA_DIRECT_DROPSHIP":
    ...
    doc.warehouse = "Factory - LH"
    doc.fulfillment_route_tag = "Pending India Dispatch"
    doc.save(ignore_permissions=True)
    frappe.db.commit()
    return                                    # <-- returns HERE

# Stamp per-item fulfillment_source badges ...
check_us_inventory(doc)

# Populate the two Warehouse Shipment Item tables ...
_populate_warehouse_shipment_tables(doc)      # <-- never reached for INDIA_DIRECT_DROPSHIP
```

**The `INDIA_DIRECT_DROPSHIP` branch returns before `check_us_inventory()` and `_populate_warehouse_shipment_tables()` ever run.** Those two calls are what would normally populate `factory_warehouse_shipment_items` from the BOM explosion that `compute_routing_outcome()` already performed in memory (via `explode_and_check_bom_availability()`). Since the function returns early for this specific outcome, that in-memory BOM-explosion result is simply discarded — the table is never written, no matter what the order's BOM actually contains.

**This means: for every plain `INDIA_DIRECT_DROPSHIP` order, `factory_warehouse_shipment_items` will always be empty at the point "Route via US Warehouse Instead" is clicked** — not just for simple non-BOM orders (as seen with `LYF-MN-2026-0036`), but for BOM-backed orders too. This directly explains why the fallback branch in `_hold_for_india_components` is the one that actually runs every time this button is used from a normal India-Direct-Dropship order — the "create a real Transfer Order" branch may be effectively unreachable via this button as currently wired.

## A3.5 Step 2 — Click "Route via US Warehouse Instead" → Yes

Ran the real `route_via_us_warehouse()` function directly (exactly what the confirmation popup's "Yes" calls server-side):

```
--- STEP 2: AFTER BUTTON CLICK (Yes) ---
routing_outcome: INDIA_TO_US_TO_CUSTOMER
order_via_us_warehouse: 1
two_leg_required: 1
warehouse: Factory - LH
fulfillment_route_tag: Pending India Dispatch
status/workflow_status/workflow_state: Submitted to 1Click / Submitted to 1Click / Submitted to 1Click
oneclick_order_id: SHOULD-NOT-BE-CALLED-STEP2
Transfer Order created: None
```

Confirms exactly the same outcome as the `LYF-MN-2026-0036` demo: because `factory_warehouse_shipment_items` was empty (root cause above — not this order's fault), `_hold_for_india_components` again took the **fallback** branch — `assign_warehouse(doc); create_oneclick_order(doc)` — submitting straight to 1Click, with **no Transfer Order created at all**.

The trip-wire fired as designed: `oneclick_order_id` shows the fake `SHOULD-NOT-BE-CALLED-STEP2` ID, proving the mocked `create_order()` really was called at this step — confirming the fallback branch, not the Transfer Order branch, is what actually ran.

## A3.6 Pass/Fail Assessment

| Expectation | Actual Result | Pass? |
|---|---|---|
| Button only usable from `INDIA_DIRECT_DROPSHIP` | Confirmed via `LYF-MN-2026-0036` (already past this state) correctly refusing with the guard error; `LYF-MN-2026-0038` (freshly in that state) correctly proceeding | ✅ |
| Clicking Yes sets `order_via_us_warehouse = 1`, `routing_outcome = INDIA_TO_US_TO_CUSTOMER`, `two_leg_required = 1` | All three set correctly | ✅ |
| **A Transfer Order is created for the missing components**, order goes on hold | **No Transfer Order created** — order was submitted straight to 1Click instead | ❌ — see §A3.4.1 root cause |
| No Draft Gate Pass / Sales Invoice / Purchase Order existed to clean up (none were created for this simple test order) | N/A — none existed | N/A |

**Result (at the time): the button's core state changes worked correctly, but the "create a Transfer Order and hold" behavior it advertises — and that its own confirmation dialog text promises — could not be demonstrated, because the upstream data (`factory_warehouse_shipment_items`) it depends on was never populated for an `INDIA_DIRECT_DROPSHIP` order in the first place. This is now fixed — see below.**

## A3.7 Root Cause and Fix — RESOLVED (2026-08-25), no discussion required

The A2.6 follow-up already flagged that a *plain, non-BOM* order hits the fallback branch instead of creating a Transfer Order. This A3 test deliberately controlled for that by using a BOM-backed order instead — and **still** hit the same fallback, for a different, more fundamental reason: `run_oneclick_fulfillment`'s `INDIA_DIRECT_DROPSHIP` branch returned before ever running the BOM-explosion/stock-check step (`check_us_inventory` + `_populate_warehouse_shipment_tables`) that would populate `factory_warehouse_shipment_items` in the first place.

**Net effect (before the fix):** as wired at the time, "Route via US Warehouse Instead" hit the fallback (immediate 1Click submission, no Transfer Order) for any order reached via the normal `INDIA_DIRECT_DROPSHIP` path — regardless of whether the order had a BOM or what that BOM contained.

**Fix applied:** `run_oneclick_fulfillment`'s `INDIA_DIRECT_DROPSHIP` branch now runs `check_us_inventory` + `_populate_warehouse_shipment_tables` before its early return — so the component data needed to build a real Transfer Order exists by the time a human later clicks "Route via US Warehouse Instead." Confirmed fixed live: re-running this exact A3 scenario (`2FT-BFK-PN-150` BOM order) after the fix now shows `factory_warehouse_shipment_items` populated with all 3 components *before* the button is ever clicked, and clicking it creates a real Transfer Order (`Draft`, all 3 components) instead of falling through to an immediate 1Click submission. Full detail in [GAP RESOLVED](#gap-resolved--business-workflow-mismatch-fixed-2026-08-25) below.

---

# GAP RESOLVED — Business Workflow Mismatch (FIXED 2026-08-25)

Flagged 2026-08-24, discussed directly with the business (Viral). **Fixed 2026-08-25 — no further discussion required on this topic.** This section now records what was fixed, how, and how it was verified.

**Intended workflow, as described by the business:**

1. "Route via US Warehouse Instead" means: the order is prepared in **India** and physically shipped/delivered **to the US Warehouse** first.
2. **Only once the order has actually been delivered to the US Warehouse** should ERP call the 1Click API and book the order.
3. **Only at that point** should status change to `Submitted to 1Click`.
4. After that, the US Warehouse team reviews the order and works on delivering it to the end customer.

**What the current code actually does (confirmed by the A3 test above, not theoretical):**

1. The moment the button is clicked and confirmed, the code checks whether `factory_warehouse_shipment_items` (the "what's missing from India" list) has any rows.
2. Because of the root-cause bug in §A3.4.1, that list is **always empty** for an order that arrived via the normal `INDIA_DIRECT_DROPSHIP` (Route C) path — regardless of whether the order genuinely has missing components.
3. Seeing an empty list, the code concludes "nothing missing — ship it now" and **immediately calls 1Click's Create Order API and sets status to `Submitted to 1Click`** — on the same click, with no physical transfer to the US Warehouse having happened yet.
4. No Transfer Order (the India → US Warehouse shipment record) is ever created. There is nothing left in the system representing "wait for physical arrival at the US Warehouse."

**In one sentence: the code skips steps 1–2 of the intended workflow entirely and jumps straight to step 3 (booking with 1Click) at the moment the button is clicked, instead of waiting for physical delivery to the US Warehouse.**

**Confirmed live on `LYF-MN-2026-0038`:** its `workflow_state` shows `Submitted to 1Click` immediately after the button click — with `oneclick_order_id` holding the test's trip-wire value (`SHOULD-NOT-BE-CALLED-STEP2`), proving the (mocked) 1Click Create Order call really fired at that moment, not after any transfer/receipt step.

**Important — the correct machinery for this already exists elsewhere in the codebase, it's just not being reached from this button:**
- `_hold_for_india_components()` — when it *does* find rows in `factory_warehouse_shipment_items` — correctly creates a Transfer Order and puts the order on hold (`fulfillment_route_tag = "Awaiting India Components"`), with no 1Click call at that point.
- `_maybe_resume_oneclick_order()` — a separate function, wired to fire when a Transfer Order is marked **"Received"** — is what's supposed to resume the order and *then* call 1Click / set `Submitted to 1Click`.
- So the design for "wait for physical arrival, then book" already exists and works when reached correctly (this is the same mechanism Route D / `INDIA_TO_US_TO_CUSTOMER` uses when it doesn't go through this override button). The bug is specifically that **this button's entry point never gets to use that machinery**, because the missing-components list it depends on was never populated for a Route C order in the first place.

**What was fixed:** `run_oneclick_fulfillment`'s `INDIA_DIRECT_DROPSHIP` branch now populates `factory_warehouse_shipment_items` (via `check_us_inventory` + `_populate_warehouse_shipment_tables`) before it returns — so that when a human later clicks "Route via US Warehouse Instead," the code correctly sees the real missing components, creates the Transfer Order, and holds the order until it's marked Received — instead of wrongly concluding nothing is missing and booking with 1Click immediately.

**Also built as part of this fix — the "wait for physical delivery" step is now real, both automated and manual:**

1. **Automated detection** (`transfer_order_tracking.py`, new hourly cron `scheduled_track_transfer_orders`): polls Shipped Transfer Orders with a leg1 tracking number set. On delivery detected — stamps the Lyfe Order `US Warehouse Delivered` first (a real Workflow transition), then marks the Transfer Order `Received`, which fires the existing `TransferOrder.on_update → _maybe_resume_oneclick_order` chain to actually post to 1Click.
2. **Manual fallback** (`Transfer Order.mark_delivered_manually()` + "Mark US Warehouse Delivered" button): for when automation can't confirm delivery on its own (no tracking number, unsupported carrier, etc.) — a human clicks one button and the order goes through the identical two-step sequence automation would have, converging on the same end state.
3. Two Workflow transitions that were missing (`Factory Assignment → US Warehouse Delivered`, later also `Awaiting Tracking → US Warehouse Delivered` and `Shipped`/`Awaiting Shipping → US Warehouse Delivered` for a related tracking-mirroring feature) were added via proper, idempotent patches.
4. A separate, genuinely pre-existing bug was found and fixed along the way: `apply_workflow()` internally calls `doc.load_from_db()`, which was silently discarding `warehouse`/`delivered_to_us_warehouse` field values set on the in-memory doc immediately before calling it — fixed by persisting those fields via `frappe.db.set_value` first.

**Confirmed fixed live** (2026-08-25, re-running the exact A3 scenario): `factory_warehouse_shipment_items` now correctly populated with all 3 BOM components before the button click; clicking "Route via US Warehouse Instead" creates a real `Draft` Transfer Order with those 3 components instead of falling through to immediate 1Click submission; `oneclick_order_id` stays `None` until the Transfer Order is actually marked Received (automated or manual). Full end-to-end cycle (save tracking info → poll in-transit → poll delivered → `US Warehouse Delivered` → `Ready for Dispatch` / `Submitted to 1Click`) verified on fresh test orders with real and mocked carrier responses.

**Additional note on 1Click and stock:** confirmed by scanning every function in `oneclick_api.py` — this app only ever **reads** inventory from 1Click (`get_inventory()`); there is no API call anywhere in this codebase that writes/updates stock at 1Click. The physical India → US Warehouse transfer is not something ERP pushes to 1Click — 1Click's own inventory count presumably updates on their side once goods are physically received at their warehouse, outside this app's control. The "Received" marking on the Transfer Order (whether automated or manual) is an internal ERP record, not a live confirmation from 1Click itself — this remains true after the fix and is a known, accepted characteristic of the design, not an open question.

**Status: FIXED AND VERIFIED.** No further action or discussion needed on this topic.

---

# Related Fixes Made During This Testing Session

All live-environment gaps discovered while preparing and following up on these tests, all fixed via proper Frappe patches (not ad-hoc live edits), per standing instruction. All items below are resolved — no discussion required.

1. **`lh/patches/add_oneclick_workflow_states.py`** — added the 4 missing 1Click Workflow States/Transitions (`1Click Error`, `Submitted to 1Click`, `Pending India Dispatch`, `Awaiting India Components`) to the live `Lyfe Order` Workflow, which had never been captured in a patch.
2. **`lh/patches/sync_lyfe_order_status_property_setter.py`** — fixed two stale Property Setters (`status` and `workflow_status` fields) that were silently overriding the DocType JSON's Select options at runtime with an older, pre-1Click list of values. Also added "Size Approved" to `workflow_status`'s own JSON options for parity with `status`'s JSON.
3. **`lh/patches/add_factory_assignment_to_us_warehouse_delivered_transition.py`** — added the missing `Factory Assignment → US Warehouse Delivered` Workflow transition (needed for the delivery-detection fix below).
4. **`lh/patches/add_awaiting_tracking_to_us_warehouse_delivered_transition.py`** — added the missing `Awaiting Tracking → US Warehouse Delivered` Workflow transition (an order with its Gate Pass submitted can still be genuinely awaiting US-leg delivery).
5. **`lh/patches/add_shipped_to_us_warehouse_delivered_transition.py`** — added the missing `Shipped → US Warehouse Delivered` and `Awaiting Shipping → US Warehouse Delivered` Workflow transitions (needed for the US-leg tracking flow mirroring the main-leg's `tracking_number` status flow).
6. **`run_oneclick_fulfillment`'s `INDIA_DIRECT_DROPSHIP` early-return fix** — the core fix behind [GAP RESOLVED](#gap-resolved--business-workflow-mismatch-fixed-2026-08-25) above.
7. **New `transfer_order_tracking.py`** — automated hourly polling of the India → US Warehouse leg, plus the manual `mark_delivered_manually()` fallback button.
8. **`fetch_ready_orders_us()` filter fix** (`order_tracking.py`) — dropped an over-restrictive `workflow_state` allowlist that was silently hiding orders whose Gate Pass had already been submitted while their US-leg tracking was still pending.
9. **US-leg tracking status flow** (`_validate_tracking_number_us`, `on_update`, `track_and_update_order_us`) — mirrors the main leg's `Awaiting Shipping → Shipped → [terminal]` flow exactly, with `US Warehouse Delivered` as the terminal state instead of `Completed`.
10. **`apply_workflow`/`load_from_db` field-loss bug** (`track_and_update_order_us`) — `warehouse`/`delivered_to_us_warehouse` were being silently discarded before `apply_workflow()` could persist them; fixed by setting them via `frappe.db.set_value` first.
11. **Gate Pass Sales Invoice billing address fix** (`gate_pass.py`) — a US-Warehouse-bound order's Sales Invoice was throwing "Billing Address does not belong to the {Customer}" on Gate Pass submit, because the resolved warehouse Address was never linked to the Customer. Fixed so the billing address always matches the actual delivery address (the US Warehouse, when applicable) — the Address is now linked to the Customer in both cases.

All patches are idempotent and safe to re-run; all have been executed successfully on `lyfe.local.local` and verified against real orders and, where possible, real carrier tracking data.
