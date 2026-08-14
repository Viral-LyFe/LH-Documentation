# 1Click Logistics — Test Plan (All Implemented Use Cases + Edge Cases)

**Purpose:** manual test script for someone with real 1Click portal + API access. Every case below maps to an actual code path — file/function referenced so a failure can be traced directly to source.

**Source of truth for the design:** `lh/docs/us_warehouse.md`. This file is the executable checklist version of it, plus edge cases not written up there.

---

## 0. Before you start — setup checklist

### Which environment to point at

1Click (iComWMS) gave us two base URLs — **use the sandbox one for all testing in this doc**, never production:

| Environment | Base URL |
|---|---|
| Sandbox (use this for testing) | `https://icomwms.dev` |
| Production | `https://icomwms.com` |

You should have received a 46-character API key by email (per Kevin Yu / Mike Straight, 1Click). That key is tied to one specific environment — confirm with 1Click whether the key you were given is scoped to sandbox or production before testing, since submitting test orders against production would create real fulfillment requests.

1Click also gave us two separate documentation sources — treat the **Fern docs as authoritative** where they overlap with the older Postman collection, since Fern appears to be the newer/actively maintained one:

- **Legacy (Postman):** https://documenter.getpostman.com/view/18244556/2s93z58Pf5 — JS-rendered, must be opened in a browser (can't be scraped/curled).
- **Fern (current):** https://1-click-logistics.docs.buildwithfern.com/1-click-api-documentation — has an environment selector in the docs UI itself; make sure it's set to sandbox when reading request examples.

**As of 2026-08-13, all three endpoints we use are confirmed against real 1Click examples** (schema screenshots + sample responses, not just docs text) — see `lh/docs/oneclick_open_questions.md` for the full detail and history. Every mismatch found that way has already been fixed in code:

| Endpoint | Method + Path | Status |
|---|---|---|
| Tracking | `GET /api/v2/orders`, `Call-Type: Tracking` | ✅ Confirmed — code already matched, no changes needed. |
| Inventory | `GET /api/v1/inventory.cfm`, `Call-Type: Stock`, body key `"inventory"` + `allWarehouses` | ✅ Confirmed — code fixed (was `POST` to `/api/v1/inventory` with wrong body shape). |
| Create Order | `POST /api/v2/orders`, `Call-Type: CreateOrder`, fields nested under `"details"` | ✅ Confirmed — code fixed (was sending fields flat, missing `"details"` wrapper and `attachments`). |

There's also a **"Get Closed Orders"** report endpoint (`GET /v1/reports`, header `CallType` — no hyphen, unlike every other endpoint's `Call-Type`) that our code doesn't use anywhere. Not required for current functionality, but worth knowing it exists in case reconciliation/reporting needs it later.

Go to **Oneclick Settings** (single doctype) and fill in:

| Field | Required? | Notes |
|---|---|---|
| Enable 1Click Integration | Yes | Master switch. Nothing in this doc runs unless this is checked. |
| API Base URL | Yes | Use `https://icomwms.dev` for testing (see environment table above) — **not** the field's own default, which is the production URL. |
| API Key | Yes | Sent as `Token` header. Confirmed scoped to sandbox (per 1Click, 2026-08-13). |
| US Warehouse ID | Yes | Integer ID from 1Click's system. Confirmed the same ID is used in both sandbox and production — double check with 1Click whether sandbox inventory data reflects real physical stock or test data before relying on stock-check results during testing. |
| Factory Warehouse ID | Yes | Integer ID from 1Click's system. |
| Inventory Endpoint Path | Defaults to `/api/v1/inventory.cfm` | ✅ Confirmed correct. |
| Create Order Endpoint Path | Defaults to `/api/v2/orders` | ✅ Confirmed correct. |
| Tracking Endpoint Path | Defaults to `/api/v2/orders` | ✅ Confirmed correct. |
| Request SKU Field Name | Defaults to `sku` | ✅ Confirmed correct (lowercase, matches the real sample response). |
| Response Available Qty Field Name | Defaults to `available` | ✅ Confirmed correct — though see the open question on whether `available` accounts for stock already reserved to other pending orders. |
| Default Ship Carrier / Service | Optional | Sent on every order if set. |

### Historical note: the old "critical trap" is now fixed

Earlier versions of this doc warned that `get_inventory()` treated its own default path as a "not configured" sentinel. That guard has been removed now that the real path (`/api/v1/inventory.cfm`) is confirmed and set as the default. The old code for reference:

```python
if not cfg["inventory_endpoint"] or cfg["inventory_endpoint"] == "/api/v1/inventory":
    return {"_not_configured": True}
```

This compared against the wrong path (`/api/v1/inventory`, missing `.cfm`) — the guard has been simplified to only trigger when the field is genuinely blank. No action needed for testing; mentioned here only so the history is traceable if this surfaces again in an old branch/PR.

### Confirm the scheduler is live

`sync_tracking_for_submitted_orders` runs hourly (`lh/hooks.py`, `hourly` block). For faster test iteration, run it manually instead of waiting:

```python
bench --site lyfe.local.local console
>>> from lh.lyfe_hardware.integrations.oneclick_api import sync_tracking_for_submitted_orders
>>> sync_tracking_for_submitted_orders()
```

---

## 1. How to create a test order

The integration **only triggers for `order_source = "ShipStation"`** orders, on `after_insert` (`lyfe_order.py` → `after_insert` → enqueues `run_oneclick_fulfillment`). Manual orders, Shopify orders, and split/merge orders never trigger it.

Practical ways to create a test order:
- Push a real test order through your ShipStation → Lyfe Order sync pipeline (safest, exercises the full real path).
- Or in `bench console`, manually create a `ShipStation Orders` doc and a linked `Lyfe Order` with `order_source = "ShipStation"`, then call `run_oneclick_fulfillment(order_name)` directly to skip waiting for the background queue.

Since fulfillment is enqueued (`queue="default"`), check the RQ worker is running (`bench worker` or however your site runs queues) or the job will just sit pending.

---

## 2. Route A — US Full (all items in US stock)

**Setup:** order with 1–2 items, all SKUs confirmed in stock at your US Warehouse in the 1Click portal.

**Expected:**
1. `routing_outcome` = `US_FULL`
2. Every `order_items` row shows `fulfillment_source_display` = 🟢 **🇺🇸 US Warehouse** badge
3. `warehouse` = `US Warehouse - LH`
4. `two_leg_required` = unchecked
5. 1Click portal shows a new order with PO number = the Lyfe Order name (e.g. `LYF-ST-2026-0042`)
6. `status` / `workflow_status` / `workflow_state` = **Submitted to 1Click**
7. `oneclick_order_id` populated from the create-order response
8. `oneclick_raw_response` has the full JSON

**Then, after 1Click dispatches (or run the hourly sync manually):**
9. `oneclick_tracking_number` and `oneclick_status` populated
10. `tracking_number` and `carrier` populated (only if they were previously blank — see §7 edge case)
11. `leg2_tracking` stays **blank** — this field is only for two-leg orders (Route B/D), and Route A never sets `two_leg_required`

**Code path:** `run_oneclick_fulfillment` → `check_us_inventory` → `assign_warehouse` → `create_oneclick_order` → `_submit_single_oneclick_order`.

---

## 3. Route C — India Direct Dropship (nothing in US stock, non-US customer)

**Setup:** order with SKUs confirmed **not** in US stock, `ship_to_country` set to something other than US/USA/United States (case-insensitive — see `US_COUNTRY_VALUES` in `lyfe_order_routing.py`).

**Expected:**
1. `routing_outcome` = `INDIA_DIRECT_DROPSHIP`
2. **No 1Click order is created** — nothing appears in the 1Click portal for this order
3. `warehouse` = `Factory - LH`
4. `status` / `workflow_status` / `workflow_state` = **Pending India Dispatch**
5. `two_leg_required` = unchecked (single leg — India ships direct to the customer)

**Manual step to simulate India dispatch (no Transfer Order is auto-created for Route C):**
6. Enter tracking directly on the Lyfe Order (`tracking_number` + `carrier`), or create a `Transfer Order` manually and link it — either way, once `tracking_number` is set, the existing 17Track/ShipEngine polling should pick it up on its own schedule.

**Code path:** `run_oneclick_fulfillment` — the `INDIA_DIRECT_DROPSHIP` branch returns immediately after setting status/warehouse, before `check_us_inventory` or any 1Click API call.

---

## 4. Route B — Mixed (tubing in US, other components not)

**Setup:** order with at least one item in the **"Tubing"** Item Group (exact match, case-sensitive — `TUBING_ITEM_GROUP = "Tubing"` in `lyfe_order_routing.py`) confirmed in US stock, plus at least one non-tubing item **not** in US stock.

**Expected — on order creation:**
1. `routing_outcome` = `MIXED_TUBING_US_COMPONENTS_INDIA_TO_US`
2. `two_leg_required` = **checked**
3. `status` / `workflow_status` / `workflow_state` = **Awaiting India Components**
4. `warehouse` = `Factory - LH` (interim, until resumed)
5. **No 1Click order created yet** — nothing in the 1Click portal
6. A new **Transfer Order** is auto-created, linked via `lyfe_order` field, containing only the rows whose `fulfillment_source` came back `"Factory"` (i.e. the missing non-tubing components) — NOT the tubing item, since that's already in the US.
7. Transfer Order `status` = **Draft**

**Simulate India shipping the missing components:**
8. Open the Transfer Order, enter `leg1_tracking` + `carrier_leg1`, save.
   - Confirm: Lyfe Order's `leg1_tracking` field is now populated (via `TransferOrder._sync_leg1_to_lyfe_order`)
9. Change the Transfer Order's `status` to **Received**, save.
   - Confirm: this enqueues `_maybe_resume_oneclick_order` (`lh.lyfe_hardware.doctype.transfer_order.transfer_order` → `_maybe_resume_oneclick_order` hook → `lyfe_order.py::_maybe_resume_oneclick_order`)

**Expected — after Received triggers the resume:**
10. `check_us_inventory` re-runs — badges refresh
11. `warehouse` = `US Warehouse - LH`
12. **One** 1Click order is created (PO = Lyfe Order name), containing **all** items — the tubing that was already in the US **plus** the components that just arrived. This is a single shipment to the customer, not two.
13. `status` = **Submitted to 1Click**
14. `oneclick_order_id`, `oneclick_raw_response` populated

**Then, after dispatch:**
15. Hourly sync populates `oneclick_tracking_number` + `oneclick_status`
16. Because `two_leg_required` is checked, `leg2_tracking` is also populated (in addition to `tracking_number`) — confirm both fields show the same US→Customer tracking number.

**Code path:** `run_oneclick_fulfillment` → `_hold_for_india_components` → (wait) → `TransferOrder.on_update` → `_maybe_resume_oneclick_order` → `check_us_inventory` → `create_oneclick_order`.

---

## 5. Route D — India to US to Customer (nothing in US stock, US customer)

**Setup:** order with all SKUs confirmed **not** in US stock, `ship_to_country` = United States (or "US"/"USA", case-insensitive).

**Expected:** identical flow to Route B (§4, steps 1–16), except:
- `routing_outcome` = `INDIA_TO_US_TO_CUSTOMER` instead of `MIXED_TUBING...`
- The auto-created Transfer Order contains **all** the order's physical items (since none were in US stock), not just some.

**Code path:** same as Route B — `_hold_for_india_components` handles both outcomes identically.

---

## 6. Mixed-stock split (some items in US stock, some not — no tubing/routing involved)

This is a **different** mechanism from Routes B/D. It applies whenever `check_us_inventory` finds some items in stock and some not, in the plain `US_FULL` fallback path (i.e. whatever routing_outcome resolves to when it isn't one of the 4 named routes but `assign_warehouse` still finds a genuine mix — in practice this is what happens for a `US_FULL`-outcome order where the live stock check at submission time disagrees with the routing check moments earlier, a race condition — see §8).

**Expected:**
1. Parent order is archived: `status` = **Split**
2. Two new child Lyfe Orders are created:
   - `<parent>-US` — contains items with stock, `warehouse` = `US Warehouse - LH`, submitted to 1Click, `status` = **Submitted to 1Click**
   - `<parent>-FC` — contains items without stock, `warehouse` = `Factory - LH`, **not** submitted to 1Click, `status` = **Pending India Dispatch**
3. Both children link back via `parent_lyfe_order` = parent order name
4. Customer receives **two** separate shipments (unlike Route B/D, which deliberately waits to send one)

**Code path:** `create_oneclick_order` → `_split_and_submit_oneclick` → `_create_oneclick_child_order` (×2).

---

## 7. Manual override — Force US / Force India

**Setup:** on a new (not-yet-fulfilled) Lyfe Order, set `route_plan` = **Force US** or **Force India**, and fill in `override_reason` (should be required — see edge case below).

**Force US expected:**
- `compute_routing_outcome` skips the stock check entirely — `_full_us_avail = True`, `_tubing_us_avail = True`
- `routing_outcome` = `US_FULL` regardless of actual stock
- Proceeds through the normal Route A flow (§2) — **including calling 1Click even if the item genuinely isn't in stock there.** Confirm what 1Click does when you submit an order for an item you know isn't physically at the US warehouse (this is a real possible edge case in production, not just a test artifact).

**Force India expected:**
- `_full_us_avail = False`, `_tubing_us_avail = False`
- Falls into `_india_route(doc)` — same US/non-US destination check as normal, so it still resolves to either `INDIA_DIRECT_DROPSHIP` or `INDIA_TO_US_TO_CUSTOMER` depending on `ship_to_country`
- **Force India does NOT force Route C specifically** — if the customer is in the US, Force India still produces Route D (two-leg, Transfer Order, hold-and-resume), not a simple India-direct-dropship. Confirm this matches your intent; it might read as surprising ("I said Force India but it still tried to route through the US warehouse").

**Edge case to check:** `override_reason` mandatory-when-overridden is enforced by `lyfe_order.json`'s `mandatory_depends_on: eval:doc.route_plan!="Auto"` — this is **client-side only** (form-level validation). Confirm whether there's also a server-side `frappe.throw` if someone sets `route_plan` via the API directly, bypassing the form. (Per this app's own coding rules, there should be — worth flagging to us if testing finds there isn't.)

---

## 8. Edge case — Tubing detection depends on exact Item Group + `sku` = Item name

`_is_tubing_item` (`lyfe_order_routing.py`) does:

```python
item_group = frappe.db.get_value("Item", row.sku, "item_group")
return (item_group or "").strip() == "Tubing"
```

Two traps:
1. **Item Group must be spelled exactly `"Tubing"`** (capital T, singular, nothing else) — `"tubing"`, `"Tubing Products"`, etc. will not match.
2. **This looks up the Item by treating `row.sku` as the Item's primary name/`item_code`.** If your ShipStation SKU only matches via the Item's `custom_sku` field (a secondary field, not the primary key — see `resolve_erp_item` in `lh/lyfe_hardware/utils/item_lookup.py`, which explicitly supports this as its #2 priority match), this lookup will **silently fail and return "not tubing"** even for a genuine tubing item, since `frappe.db.get_value("Item", row.sku, ...)` only matches on the primary key.

**Test:** create a tubing item whose SKU on the order does **not** equal the Item's own `name` (only matches via `custom_sku`). Confirm whether Route B is correctly detected or whether it falls through to Route D/C instead. If it misroutes, that's a known gap in the current code, not a config issue on your end — report it back to us rather than trying to fix it via data cleanup.

---

## 9. Edge case — fee/charge lines never affect routing

**Setup:** an order with a line item named exactly **"Custom Fee"** or **"Customs Fee"** (case-insensitive; also matches anything *starting with* those strings — `_is_fee_item` uses `startswith`, not exact match, so "Custom Fee - Rush" would also match).

**Expected:**
- That row always shows badge ⚫ **Excluded**
- It is never sent in the 1Click order payload
- It is never checked against US stock (not counted toward "all items in stock" or "tubing in stock")
- If it's the *only* non-adjustment row candidate in an otherwise-empty order, `_submit_single_oneclick_order` throws `"No items to submit to 1Click — all lines are adjustments or excluded fees."` — confirm this error surfaces cleanly rather than silently submitting an empty order.

---

## 10. Edge case — missing SKU

Two different, independently-triggered behaviors:

**a) Auto-extraction attempt** (`_extract_sku_from_name`, regex `\(\s*SKU\s*:\s*([^\s)]+)\s*\)`): if an item's `sku` field is blank but `item_name` contains a pattern like `Custom Bunk Rail (SKU: 19L-R222-RB)`, the system extracts `19L-R222-RB`, verifies it exists in the Item Master (by `item_code` or `custom_sku`), and fills it in automatically. Test with an item name that has this pattern but references a SKU that does **not** exist in the Item Master — confirm it's left blank (not filled with a bogus value).

**b) Hard block for Standard orders** (`_validate_sku_on_new_standard_orders`): if `order_type = "Standard"` and the order is still in status `""`/`"New"`, saving is blocked with a "Missing SKU" error if any non-fee, non-adjustment row has no SKU after the auto-extraction above. Test:
- Standard order, missing SKU, still New → save should be **blocked**
- Same order, change Order Type to Custom → save should **succeed**
- Same order, but past New status (e.g. already Submitted to 1Click) → this validation should **not** re-fire (it only runs on New orders — confirm editing an already-submitted order for unrelated reasons doesn't retroactively block on this)

**c) What happens if a genuinely SKU-less item reaches 1Click submission anyway** (e.g. a Custom order, which skips the block in (b)): `_build_items_payload` falls back through `_resolve_sku` → `fulfillment_sku` → `erp_item` → truncated `item_name`, and sets `has_sku_warnings = True`, which appends a warning string into `oneclick_error` — **but this does not block submission**, it's informational only. Confirm the order still submits to 1Click and check what 1Click's response is when it receives a fabricated/fallback "SKU" like a truncated item name.

---

## 11. Edge case — order items locked after 1Click submission

**Setup:** an order at `status = "Submitted to 1Click"`. Try to edit an `order_items` row (change qty, SKU, item_name) or add/remove a row, then save.

**Expected:** blocked with `"This order has been submitted to 1Click Logistics — order items are locked. To make changes, cancel the 1Click submission first."` (`_validate_order_items_locked`).

**Also confirm:** `_sync_order_items_to_db` (the ShipStation-driven re-sync that normally wipes and re-inserts `order_items` on every update) is also skipped once `status = "Submitted to 1Click"` — so if the source ShipStation order changes after submission, those changes should **not** silently overwrite the locked items. Test by editing the source ShipStation order after your Lyfe Order has been submitted, and confirming the Lyfe Order's `order_items` don't change.

**Known gap — not yet built:** there is currently no "cancel the 1Click submission" action referenced in that error message. If you need to actually unlock a submitted order, there's no button/whitelisted method for it today — you'd have to intervene manually. Worth deciding whether that's needed before it comes up in production.

---

## 11a. Edge case — header naming is confirmed consistent for the endpoints we use (with one known exception elsewhere)

Our code sends `Call-Type` (with a hyphen) on every request — Inventory, Create Order, and Tracking alike (`_headers()` in `oneclick_api.py`). This is now **confirmed correct for all three** via real schema examples (Tracking: `Call-Type: Tracking`, Inventory: `Call-Type: Stock`, Create Order: `Call-Type: CreateOrder`). No action needed.

The one inconsistency that remains is 1Click's own **Get Closed Orders** report endpoint, which uses `CallType` — no hyphen — unlike everything else. We don't use that endpoint anywhere today, so this is informational only (relevant if we ever add reconciliation/reporting via that endpoint later).

---

## 12. Edge case — 1Click API failure at any step

**Setup:** temporarily break the integration — wrong API key, wrong warehouse ID, or simulate a network failure (e.g. point `api_base_url` somewhere invalid).

**Expected, for a Route A/split submission failure:**
1. `status` / `workflow_status` / `workflow_state` = **1Click Error**
2. `oneclick_error` contains the **last 1000 characters** of the Python traceback (not the full thing — long tracebacks get truncated from the front)
3. A Frappe Error Log entry titled `"1Click Error: <order name>"` is created
4. If the order had *already* been archived as `"Split"` before the failure (i.e. the US child submission failed after the FC child was already created), the **parent is left alone** — only the failing child gets the error status. Confirm the parent doesn't get incorrectly marked `1Click Error` after a legitimate split.

**Expected, for a Route B/D resume failure** (i.e. Transfer Order marked Received but the subsequent 1Click submission fails): same error status/log, but titled `"1Click Error (resume after Transfer Order Received): <order name>"` instead — confirm both error paths are distinguishable in the Error Log when triaging.

**Edge case:** trigger a failure specifically on the **first** submission attempt (e.g. bad API key), fix the config, then figure out how to retry. **Known gap:** there is no explicit "retry" button anywhere in the code — the only way to retry today is to manually re-enqueue `run_oneclick_fulfillment(order_name)` via console, or resave the order in a way that re-triggers the relevant code path. Confirm this matches your operational expectations, since it's not documented as a supported user action anywhere in the UI.

---

## 13. Edge case — hourly tracking sync batch behavior

**Setup:** have at least 2–3 orders sitting at `status = "Submitted to 1Click"` simultaneously, including at least one whose PO/tracking lookup will fail (e.g. manually corrupt its `oneclick_po` to something invalid).

**Expected (`sync_tracking_for_submitted_orders`):**
1. All qualifying orders are fetched in one query (`filters={"status": "Submitted to 1Click"}`)
2. Each is processed independently in a loop — **one order's failure does not stop the batch**; confirm the other, valid orders still get their tracking updated even though one threw
3. The failing order gets its own Error Log entry titled `"1Click Tracking Sync Error: <order name>"`, and its `oneclick_tracking_number`/status are simply left unchanged (no partial/corrupt write)
4. **Idempotency check:** once `tracking_number` is already populated (e.g. from a manual entry or a previous sync), re-running the sync should **not** overwrite it — confirm `doc.tracking_number = tracking` only fires `if tracking and not doc.tracking_number`. This means if someone manually types a wrong tracking number in first, the automated sync will never correct it. Worth knowing before you rely on this for QA.
5. **Carrier auto-creation:** if 1Click returns a carrier string (`Dispatch_Carrier`) that doesn't match any existing `Carrier` record (by `carrier_code` or `name`), `_resolve_carrier` **creates a brand new Carrier record automatically** using that raw string as both `carrier_id` and `carrier_code`. Test this deliberately with an unfamiliar carrier name from 1Click's sandbox/test data and confirm a new Carrier doc silently appears — this is expected behavior, not a bug, but it means typos in 1Click's carrier field will pollute your Carrier list.

---

## 14. Edge case — order that resolves to zero routable outcome

Not really possible given the current `if/elif/else` chain in `compute_routing_outcome` (every path terminates in one of the 4 named outcomes), but worth a sanity check: create an order with **zero** physical items (e.g. only a Custom Fee line, no real products). Confirm what `routing_outcome` resolves to and whether `run_oneclick_fulfillment` handles the "nothing to route" case gracefully rather than throwing an unhandled error. (`_submit_single_oneclick_order`'s "No items to submit" throw, per §9, is the expected safety net here — confirm it actually fires rather than 1Click receiving an empty order.)

---

## Quick reference — status meanings during testing

| Status | What it means | Where it's set |
|---|---|---|
| *(blank/New)* | Order created, routing not yet run, or Route A/split-US-child mid-flight before submission completes | default |
| **Awaiting India Components** | Route B/D — Transfer Order created, waiting for Received | `_hold_for_india_components` |
| **Submitted to 1Click** | Successfully submitted; waiting for 1Click dispatch | `_submit_single_oneclick_order`, `_split_and_submit_oneclick` (US child), `_maybe_resume_oneclick_order` |
| **Pending India Dispatch** | Route C, or the Factory child of a plain mixed-stock split | Route C branch, `_split_and_submit_oneclick` (FC child) |
| **Split** | Parent order archived after a plain mixed-stock split | `_split_and_submit_oneclick` |
| **1Click Error** | Something failed — check `oneclick_error` + Error Log | `run_oneclick_fulfillment` / `_maybe_resume_oneclick_order` except blocks |

---

*This test plan reflects the code as of the `inventory-us` → `prod-us_warehouse_integration` merge, updated 2026-08-13 after 1Click confirmed all three used endpoints (Inventory, Create Order, Tracking) via real schema examples. If any use case above doesn't match what you observe, that's most likely a real bug now — report it — rather than a config/spec mismatch, since the endpoint specs themselves are no longer placeholders. See `lh/docs/oneclick_open_questions.md` for the remaining lower-priority refinement questions and full before/after detail on every fix.*
