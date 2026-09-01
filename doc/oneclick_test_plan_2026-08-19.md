# 1Click / US Warehouse Integration — Full Test Plan (2026-08-19)

**Supersedes:** `lh/docs/oneclick_test_plan.md` (dated 2026-08-13 — kept for history, but stale: written before F2/F3/F5/F8/F9/G1, the Route D routing default flip, the two "also show Factory Assignment" status changes, `fulfillment_route_tag`, the SLA seeding patch, and the KPI dashboard were built). **Use this file, not the old one, for testing from today onward.**

**Branch this reflects:** `prod-us_warehouse_integration`, as of commit `da2cbbd` + the layout fix (`6226fdb`).

**Purpose:** a single script covering every real use case and edge case built across this whole session, PLUS an explicit "does this leak into other branches" section — because this branch has genuinely diverged from `prod` and several other active branches in shared files, and that risk needs to be checked deliberately, not assumed away.

---

## Part A — Core 1Click routing & fulfillment

### A1. Route A — US Full (all items in US stock)

**Setup:** order with 1–2 items, all SKUs confirmed in stock at the US Warehouse in 1Click.

**Expected:**
1. `routing_outcome` = `US_FULL`
2. `warehouse` = `US Warehouse - LH`
3. `two_leg_required` unchecked
4. `fulfillment_route_tag` stays blank (this tag only exists for the two other routes below — never set here)
5. `status`/`workflow_status`/`workflow_state` = **Submitted to 1Click**
6. `oneclick_order_id`, `oneclick_raw_response` populated
7. After hourly sync (or manual `sync_tracking_for_submitted_orders()`): `oneclick_tracking_number`, `tracking_number`, `carrier` populated; **the order is auto-pushed to ShipStation → Shopify/Etsy** (G1) — confirm `ss_tracking_pushed` flips to 1 and the tracking shows up on the actual Shopify/Etsy order, not just in ERPNext.

**Code path:** `run_oneclick_fulfillment` → `check_us_inventory` → `assign_warehouse` → `create_oneclick_order` → `_submit_single_oneclick_order`.

---

### A2. Route C — India Direct Dropship (nothing in US stock, ANY destination — 2026-08-19 policy change)

**⚠️ This is the single biggest behavior change from the old test plan — test it first and carefully.**

**Old behavior (pre-2026-08-19):** nothing in US stock + US customer → auto-routed via the US Warehouse (old Route D).
**New behavior:** nothing in US stock → **always** ships direct from Factory to the customer, **regardless of destination country** — a US customer no longer forces a US Warehouse hop.

**Setup:** order with all SKUs confirmed **not** in US stock. Test with **both** a US customer and a non-US customer — both must behave identically now.

**Expected (both cases):**
1. `routing_outcome` = `INDIA_DIRECT_DROPSHIP`
2. No 1Click order created
3. `warehouse` = `Factory - LH`
4. `status`/`workflow_status`/`workflow_state` = **Factory Assignment** — ⚠️ **NOT** "Pending India Dispatch" (that string no longer appears as a status; see A2a below)
5. `fulfillment_route_tag` = **"Pending India Dispatch"** (hidden field — check via API/report, it has no UI presence by design)
6. Order gets the same auto-assignment (`factory_assignment_date`, `cs_team_first_action` stamped, ToDo assigned to Factory) as any normal Factory order
7. Appears in the Factory pending-list page (`lyfe_orders_status_overview`) and its legacy Report under "Pending By Factory" — via the FIRST branch of that query now (plain `status = 'Factory Assignment'` match), not the old second branch

**A2a. Regression check — a real historical order** already shows this new behavior even without any manual action: `LYF-MN-2026-0032` (created during this session's testing) has `status = "Factory Assignment"` with `routing_outcome = INDIA_DIRECT_DROPSHIP` — confirm it's still bucketed correctly on the pending-list page today.

---

### A3. Override — "Route via US Warehouse Instead" (new button, replaces old Route D auto-selection)

**Setup:** take a real order currently at `routing_outcome = INDIA_DIRECT_DROPSHIP`, `fulfillment_route_tag = "Pending India Dispatch"`, no `cj_awb_no` yet. Open the Lyfe Order form.

**Expected:**
1. Button **"Route via US Warehouse Instead"** is visible (only when `routing_outcome == INDIA_DIRECT_DROPSHIP` and `cj_awb_no` is empty)
2. Clicking it, confirming the dialog:
   - Sets `order_via_us_warehouse = 1` (reuses the **existing** checkbox — deliberately not a new field; also activates that checkbox's other, older behaviors: item-source dialog, pending-list address swap, dispatch SLA math — this is intentional, not a side effect to "fix")
   - `routing_outcome` becomes `INDIA_TO_US_TO_CUSTOMER`
   - `two_leg_required` becomes checked
   - A real **Transfer Order** is created (status Draft), containing the Factory-owed components
   - `status`/`workflow_status`/`workflow_state` = **Factory Assignment** (⚠️ NOT "Awaiting India Components" — same policy change as A2, applied a second time this session)
   - `fulfillment_route_tag` = **"Awaiting India Components"**
3. **Cleanup behavior**, if the order already had a Draft Gate Pass / Draft Sales Invoice / Draft Purchase Order attached: all three are auto-cancelled/deleted. If any of those three is **already submitted** (not draft), the whole action is blocked with an explicit error naming which one — confirm this, don't just test the empty-order case.
4. **Hard block** if `cj_awb_no` is already set (a real shipment has already gone out) — confirm the button either doesn't appear or the call throws clearly, telling the user to cancel the carrier shipment first.

**Code path:** `route_via_us_warehouse()` in `lyfe_order.py`.

---

### A4. Resume after Transfer Order marked Received (fully automatic — no button)

**Setup:** an order in the state left by A3 above (real Transfer Order in Draft/Shipped).

**Steps:**
1. Mark the Transfer Order `Shipped`, confirm it now shows up in the **In-Transit Aging** KPI card (Part D below) and in F3's own SLA alert candidate list.
2. Mark the Transfer Order `Received`.

**Expected — automatically, no human action beyond step 2:**
1. `TransferOrder.on_update()` fires → `_maybe_resume_oneclick_order()` (Transfer Order side) → enqueues the Lyfe Order side `_maybe_resume_oneclick_order(lyfe_order_name)` — this is a background job (`enqueue_after_commit=True`), so allow a few seconds, don't assume it's synchronous.
2. Gate check reads `fulfillment_route_tag == "Awaiting India Components"` (⚠️ NOT `doc.status` — that check was deliberately moved off status this session; if you're testing against an *old* order created before this session's fix, its tag may be blank even in this state — check the tag exists before relying on this step).
3. `warehouse` flips to `US Warehouse - LH`.
4. `fulfillment_route_tag` is cleared to blank/None.
5. Order is submitted to 1Click as **one single shipment** covering all items — the ones already in US stock plus the just-arrived India components.
6. **On success:** `status` = **Submitted to 1Click**. ⚠️ **Confirm this actually succeeds without a `WorkflowPermissionError`** — a real bug was found and fixed this session where the Workflow definition has no transition from `Factory Assignment` to `Submitted to 1Click`/`1Click Error` (only from `New` or the old `Awaiting India Components` status); the fix bypasses `validate_workflow()` via `frappe.db.set_value` whenever `workflow_state != "New"` at the moment of transition. **This is the single most important regression to re-test** — it was fixed today but never verified against a real (non-mocked) 1Click success response, only a real failure response.
7. **On failure** (1Click API error): `status` = **1Click Error**, `fulfillment_route_tag` cleared, `oneclick_error` populated, Error Log entry titled `"1Click Error (resume after Transfer Order Received): <order name>"`.
8. Confirm the order **immediately drops off** "Pending By Factory" on both the Page and legacy Report the moment it's marked Received — **do not wait for the resume job to actually run**; a real bug was found and fixed this session where the order stayed visible in that bucket until the background resume job completed, because the bucket's first branch matched on plain `status = 'Factory Assignment'` with no awareness of the hold tag.

**Code path:** `TransferOrder._maybe_resume_oneclick_order` → `lyfe_order._maybe_resume_oneclick_order` → `check_us_inventory` → `create_oneclick_order` → `_submit_single_oneclick_order`.

---

### A5. Route B — MIXED (some components in US stock, some not)

**Setup:** order with a BOM-backed item where some exploded components are in US stock and some aren't (or a mix of plain order rows, some in stock, some not).

**Expected — on order creation:**
1. `routing_outcome` = `MIXED_US_COMPONENTS_INDIA_TO_US` (⚠️ renamed this session from the old tubing-specific `MIXED_TUBING_US_COMPONENTS_INDIA_TO_US` — new orders always get the new string; historical orders keep the old string and both are still treated as equivalent everywhere — grep `MIXED_OUTCOMES`/`_MIXED_ROUTING_OUTCOMES` if you need the full list of places that must recognize both)
2. `two_leg_required` checked
3. `status`/`workflow_status`/`workflow_state` = **New** (unchanged this session — no auto-hold)
4. `us_warehouse_shipment_items` and `factory_warehouse_shipment_items` both populated correctly — confirm this actually persists to the DB, not just in-memory (`doc.save()` must have actually run; a real bug existed here before this session where the function returned before saving)
5. **No 1Click order yet, no Transfer Order yet** — everything waits for a human

**Confirm Split (manual step):**
6. Click "Confirm Split" on the order form.
7. `warehouse_split_confirmed` = 1, `warehouse_split_confirmed_on` populated, Factory-covered items handed to Factory (`status` → Factory Assignment via `_set_factory_assignment_status`), US-covered items tracked separately (not yet posted to 1Click).

**Post US Portion (separate manual step):**
8. Click "Post US Portion to 1Click" — submits only the US-covered items, `us_leg_oneclick_posted` = 1.

**Leg1 tracking + F2 alert:**
9. Confirm F2's SLA alert (`SLAR-0018`) fires if `tracking_number_us` stays blank >48h after `warehouse_split_confirmed_on` — see Part C below for the full SLA alert test.

**Code path:** `run_oneclick_fulfillment` → (stops, waits) → `confirm_warehouse_split` → `post_us_leg_to_oneclick`.

---

### A6. Hourly tracking sync — batch behavior, idempotency, carrier auto-creation, G1 push

This is its own test, distinct from A1/A5's "confirm tracking eventually shows up" step — it exercises the actual scheduled job (`sync_tracking_for_submitted_orders`, `lh/hooks.py`'s `hourly` block) directly, including what happens when part of the batch fails.

**Setup:** have at least 2–3 real orders sitting at `status = "Submitted to 1Click"` simultaneously. Deliberately corrupt one order's `oneclick_po` to something invalid (so its 1Click lookup will fail) while leaving the others valid.

**Expected:**
1. All qualifying orders are fetched in one query (`frappe.get_all("Lyfe Order", filters={"status": "Submitted to 1Click"}, ...)`).
2. Each order is processed independently in a loop, inside its own `try/except` — **the corrupted order's failure does not stop the batch**; confirm the other, valid orders still get their tracking updated.
3. The failing order gets its own Error Log entry titled `"[oneclick_api][sync_tracking_for_submitted_orders]: sync failed — <order name>"`, and nothing on that order is partially written.
4. **Idempotency:** once `tracking_number` is already populated (from a manual entry or a previous sync run), re-running the sync must **not** overwrite it — the write only happens `if tracking and not doc.tracking_number`. Test deliberately: manually type an obviously-wrong tracking number into a `Submitted to 1Click` order, run the sync, confirm it's left untouched even though 1Click has a different, real number. This is expected behavior, not a bug — but worth knowing before relying on this sync to "self-correct" a bad manual entry.
5. **Carrier auto-creation:** if 1Click returns a `Dispatch_Carrier` string that doesn't match any existing `Carrier` record by `carrier_code` or `name`, `_resolve_carrier` creates a brand-new `Carrier` doc automatically, using the raw string as both `carrier_id` and `carrier_code`. Test with an unfamiliar carrier name from your 1Click sandbox data and confirm a new Carrier record silently appears — expected, but means a typo in 1Click's carrier field pollutes the Carrier list.
6. **G1 push fires exactly once per order:** confirm `_maybe_push_oneclick_tracking_to_shipstation` only actually pushes when `ss_tracking_pushed` is not already set, and that flag is set inside `_mark_shipstation_order_as_shipped` itself (the same guard every tracking-push caller shares, including the pre-existing 17Track path) — so re-running the sync a second time on an already-pushed order must **not** push again or duplicate the customer-facing notification. Test by running the sync twice in a row on the same order and confirming only one push occurs (check `ss_tracking_pushed` before/after each run, don't just check the final state after both).
7. **Leg1 never reaches this push:** confirm `_apply_tracking_update`/`_maybe_push_oneclick_tracking_to_shipstation` only ever read `doc.tracking_number`/`doc.carrier` (the canonical leg2 fields), never `tracking_number_us` — structurally guaranteed by the code, but worth a direct check on a real two-leg (MIXED/Route D) order: confirm only the leg2 (US→Customer) tracking ever reaches Shopify/Etsy, never the leg1 (India→US) number.

**Code path:** `sync_tracking_for_submitted_orders` → `get_order_status` → `_apply_tracking_update` → `_resolve_carrier` / `_maybe_push_oneclick_tracking_to_shipstation` → `_mark_shipstation_order_as_shipped`.

---

## Part B — F8/F9 address & tracking-field fixes

### B1. F9 — canonical tracking fields only

**Confirm these two retired fields no longer exist on either doctype** (schema-level, not just "unused"):
- `Lyfe Order.leg1_tracking` / `Lyfe Order.leg2_tracking`
- `Transfer Order.leg1_tracking`

Only `tracking_number_us` (leg1, India→US) and `tracking_number` (leg2, US→Customer) should exist. If any form/report/script anywhere still references the old field names, that's a real bug — grep for them.

### B2. F8 — address resolution for a MIXED or Route-via-US-Warehouse order

**Setup:** a real MIXED order (A5) with `factory_leg_destination` explicitly set to **"Via US Warehouse"**, OR a Route-via-US-Warehouse order from A3.

**Expected:**
1. CJ Logistics label generation (`cj_api.py`) writes to `tracking_number_us`/`carrier_us`, **not** `tracking_number`/`carrier` — confirm by actually generating a label, not just reading code.
2. Pending-list export and Sales Invoice generation both resolve the ship-to address to **Lyfe Hardware's own US Warehouse address** (via `get_effective_ship_to`), not the customer's real address — confirm by generating a real Excel export and a real Sales Invoice, opening both, checking the literal address text.
3. Set `factory_leg_destination` to **"Direct to Customer"** on a different order — confirm the address correctly falls back to the customer's real address instead.

### B3. F8 item-list fix — pending-list export shows only Factory's items on a MIXED split

**Setup:** the same MIXED order from A5/B2.

**Expected:**
1. Pending-list Excel/PDF export's item column shows **only** the items in `factory_warehouse_shipment_items` — not the US-covered items, not the Custom Fee row.
2. A plain, non-split order's export still shows **all** its items (confirm this doesn't regress — the filter must return `None`/no-filtering for anything that isn't a genuine MIXED split, including `INDIA_DIRECT_DROPSHIP` and `INDIA_TO_US_TO_CUSTOMER`).

---

## Part C — SLA alerts (F2, F3) + the seeding patch

### C1. Confirm the SLA Rules actually exist on whichever environment you're testing

These are **not** seeded automatically by `bench migrate` alone in every historical deploy — check first:

```python
frappe.get_all("SLA Rule", filters={"detector_class": ["in", ["Lyfe Order Leg1", "Transfer Order Receipt"]]}, fields=["name", "is_active", "threshold_hours"])
```

If empty, run: `bench --site <site> run-patch lh.patches.seed_leg1_receipt_sla_rules` — confirm it creates both records with `threshold_hours` 48 and 24 respectively, `target_project = PROJ-0003`, `fallback_owner = support@chandakbrothers.net`. Run it a second time (`--force` if already recorded) and confirm it does **not** create duplicates — matches by `(erp_doctype, detector_class)`, not by hardcoded name.

### C2. F2 — no Leg1 tracking >48h (now covers BOTH MIXED and Route D — generalized this session)

**Setup A (MIXED):** a confirmed-split MIXED order (A5, post-Confirm-Split) with `tracking_number_us` left blank.
**Setup B (Route D, new coverage):** a Route-via-US-Warehouse order (A3) with `fulfillment_route_tag = "Awaiting India Components"` and `tracking_number_us` left blank.

**Expected (both):**
1. Detector candidate query picks up the order — MIXED clock starts at `warehouse_split_confirmed_on`; Route D clock starts at `modified` (a proxy for "entered the hold"), keyed by `fulfillment_route_tag`, **not** `status` (status is `Factory Assignment` for both now, so this must be tag-driven or it will silently stop working).
2. Force the age past 48h (see the session's own live-test pattern: set `candidate["age_hours"] = 50.0` and call `detector.is_violated()` directly, or wait for real time to pass) — confirm a real PM Task is created, assigned to `support@chandakbrothers.net`, subject `"No Leg 1 tracking >48h: {order name}"` (literal `{doc.name}` substitution — if you see the literal string `{name}` instead of the order name, that's the exact bug this session already caught and fixed once; it should not recur).
3. Fill in `tracking_number_us` — confirm the task auto-closes within ~15 minutes (next SLA scan pass), not just on a save-triggered `on_update`. The 15-minute sweep (`sweep_close_conditions_for_rule`) is the real safety net here, independent of whatever wrote the field.

### C3. F3 — no US receipt >24 business hours

**Setup:** a Transfer Order marked `Shipped`, left there.

**Expected:**
1. Detector picks it up, age computed in **business hours** (Mon–Sat 9am–6pm IST, Sundays excluded) — not calendar hours. Confirm with a Transfer Order shipped right before a Sunday that the age calculation correctly skips Sunday.
2. Past 24 business hours → PM Task created.
3. Mark it Received → task auto-closes.

### C4. The 15-min sweep applies to every rule, not just F2/F3

**Setup:** pick any real, currently-open SLA Task Link on the system for a rule that has a `close_condition_field`/`close_condition_value`. Update the underlying field via `frappe.db.set_value` directly (bypassing `doc.save()`, which normally would never trigger `on_update`/auto-close).

**Expected:** within one 15-minute scan cycle, the task auto-closes anyway — via `sweep_close_conditions_for_rule`, wired into every rule's scan pass, not something you have to trigger manually. This closes a real gap found this session (a `set_value` write would previously leave a task open for up to 24h, until the old daily 3am sweep caught it).

---

## Part D — KPI Dashboard (Order Analysis page)

All 4 new sections live on `order-analysis-tw`. Open the page and confirm each renders without a console error, then check the actual numbers against a direct DB query for at least one of them (don't just trust the UI).

### D1. Manual Override Rate

- Shows `overridden_orders / total_orders` for the selected date range (or Live Data).
- An order counts as overridden via **any** of: `route_plan != 'Auto'`, `override_reason` set, `order_via_us_warehouse = 1`.
- Click the card → drill-down list shows the real overridden orders with `route_plan`/`override_reason`/`order_via_us_warehouse` columns visible.
- **Test the new A3 override specifically** — confirm an order you routed via "Route via US Warehouse Instead" shows up here (it sets `order_via_us_warehouse = 1`, one of the three qualifying conditions).

### D2. OTIF by Route Plan (on-time half only — In-Full deliberately not built)

- Three cards: US Full / MIXED Split / India Only.
- Reuses the exact same "on-time" math as the pre-existing `on_time_delivery_rate` metric elsewhere on this same page (factory dispatch SLA compliance, not a delivery-date comparison — confirm both numbers are internally consistent if you cross-check them for the same date range).
- **Expected to read 0% across all three** until a real order with `routing_outcome` set reaches a dispatched state (`out_from_factory` + `factory_first_action` both populated) — every historical dispatched order predates this field. Push at least one A1/A2 test order all the way to dispatch and confirm it starts appearing in the correct bucket.

### D3. WIP Aging

- Three buckets: 0–2 / 3–5 / 6+ days, based on `factory_assignment_date` (or `creation` if not yet at Factory Assignment).
- Live/current snapshot — no date filter affects this card.
- Confirm `Return Successfully`/`Return InProgress` orders are correctly **excluded** (they're not forward-fulfillment WIP).
- Click a bucket → drill-down shows real orders with correct ages.

### D4. In-Transit Aging (India→US)

- Reuses F3's exact detector/threshold — a visual restating of the same alert data.
- Expected empty until a real Transfer Order is Shipped (test via A4 above).

### D5. Undeliverable / Delivery Exception (sits alongside the pre-existing "In Customs Hold" tile, not a separate new section)

- Text-matches `Delivery exception`, `Returning to Sender`, `Returning package to shipper` across `track_status`/`tracking_status_desc`/`tracking_status_us`/`tracking_details`.
- Confirm the pre-existing "In Customs Hold" card (immediately next to this one) still shows its correct count — this was edited in the same file/dict, confirm no regression.

### D6. Layout — full width + compact cards

- Confirm the page fills the full browser width, no white margins on either side (matches PnL Dashboard's layout).
- Confirm Manual Overrides / OTIF / WIP Aging cards are visibly smaller/denser than Order Volume / SLA Breach cards elsewhere on the same page — this was a deliberate, scoped-only-to-these-3-sections change; **In-Transit Aging should still be full-size** (uses a different, shared component with two other pre-existing sections — deliberately not shrunk).

---

## Part E — Cross-branch / prod-impact check (do this section regardless of whether Part A–D pass)

**Why this section exists:** this branch (`prod-us_warehouse_integration`) has been developed independently from `prod` and several other active feature branches for a long time. It last shared history with `prod` at commit `2df9968c`. Since then, both branches have changed **the same files independently** — this is a real merge-conflict and silent-regression risk, not a hypothetical one. Two specific things are already confirmed diverged as of this session:

### E1. `order_tracking.py` — confirmed real divergence, needs manual reconciliation before any future merge

- **On `prod` (commit `3c5f2ca`, this morning, 06:00 AM):** fixed a bug where 17Track webhook delivery confirmations could revert a Return-flow order back to `Completed`. Added return-state exclusions to `fetch_ready_orders()` **and** a defense-in-depth guard directly inside `apply_normalised_to_order()` itself, so the protection holds regardless of caller.
- **On `prod-us_warehouse_integration` (this branch):** `fetch_ready_orders()` **already independently excludes** `Return Successfully`/`Return InProgress` (confirmed live, line ~123 of `order_tracking.py` on this branch — pre-existing, not added this session) — this branch was never exposed to the specific bug `prod` fixed via the outer filter. This branch also has its **own**, different addition to the same filter dict: orders with `oneclick_tracking_number` set are excluded, since those are tracked via the 1Click hourly sync instead of 17Track/ShipEngine.
- **The one real, confirmed gap:** this branch does **not** have `prod`'s inner defense-in-depth guard inside `apply_normalised_to_order()` itself. Confirmed by tracing every caller of that function on this branch — both call sites (`track_and_update_order`, the batch scheduler) are fed exclusively by `fetch_ready_orders()`'s already-filtered list, and no other webhook/entry point on this branch calls it directly (the file named `shipstation_webhook.py` is an unrelated ShipStation *order* webhook, not the 17Track *tracking* webhook — confirmed by reading it). So today, on this branch, there is no live path that reaches `apply_normalised_to_order()` while bypassing the outer filter — the missing inner guard is a real hardening gap, not a currently-exploitable one.
- **Still a real merge risk:** if `prod`'s `apply_normalised_to_order()` changes (the inner guard) and this branch's `fetch_ready_orders()` changes (the oneclick-tracking-number exclusion) are merged automatically, confirm the merged file keeps **all three** protections — the two filter exclusions AND the inner guard — since a future refactor could add a new caller to `apply_normalised_to_order()` that bypasses `fetch_ready_orders()` the way `prod`'s fix anticipated, and only the inner guard protects against that case.

### E2. Six SLA detectors — confirmed NOT yet ported to this branch

`prod` (commit `887e24a`, this morning, 06:38 AM) fixed six SLA detectors (`internal_review.py`, `awaiting_shipping.py`, `internal_review_approval.py`, `pending_customer_approval.py`, `shipped_to_completed.py`, `promised_dispatch.py`) to stop falsely reopening SLA breaches for orders sitting in a completed Return/Reship/RTO state. **This fix does not exist on `prod-us_warehouse_integration`** — confirmed explicitly in this session's own worklog, and the user was told and accepted no action needed on this branch "for now."

**Test:** if this branch is heading toward a merge with `prod` (or a promotion to become the new prod) soon, decide explicitly whether to port this fix now or accept the same false-SLA-reopen exposure on this branch until it does. Not a blocker for 1Click testing itself, but a real gap if this branch goes live before that decision is made.

### E3. General reconciliation checklist for every shared file this session touched

Run this diff yourself before merging either direction, and manually review each file (not just line counts) — this list is exhaustive for what this session changed in files that also exist on `prod`'s lineage:

```bash
git diff 2df9968ca66f770f45f3288d18b2abc2d0bdff91 HEAD --stat -- \
  lh/hooks.py \
  lh/lh_project/doctype/sla_rule/sla_rule.json \
  lh/lh_project/sla/autoclose/close_engine.py \
  lh/lh_project/sla/detectors/__init__.py \
  lh/lh_project/sla/detectors/lyfe_order_leg1.py \
  lh/lh_project/sla/detectors/transfer_order_receipt.py \
  lh/lh_project/sla/engine/rule_evaluator.py \
  lh/lh_project/sla/utils/business_hours.py \
  lh/lh_project/sla/utils/leg1_sla.py \
  lh/lyfe_hardware/doctype/lyfe_order/create_sales_invoice.py \
  lh/lyfe_hardware/doctype/lyfe_order/lyfe_order.js \
  lh/lyfe_hardware/doctype/lyfe_order/lyfe_order.json \
  lh/lyfe_hardware/doctype/lyfe_order/lyfe_order.py \
  lh/lyfe_hardware/doctype/lyfe_order/order_tracking.py \
  lh/lyfe_hardware/integrations/oneclick_api.py \
  lh/lyfe_hardware/integrations/shipstation_webhook.py \
  lh/lyfe_hardware/lyfe_order_routing.py \
  "lh/lyfe_hardware/page/lyfe_orders_status_overview/lyfe_orders_status_overview.py" \
  lh/lyfe_hardware/page/order_analysis/order_analysis.py \
  lh/lyfe_hardware/page/order_analysis_tw/order_analysis_tw.js \
  lh/lyfe_hardware/page/order_analysis_tw/order_analysis_tw.py \
  lh/lyfe_hardware/report/order_export_utils.py \
  "lh/lyfe_hardware/report/lyfe_orders_—_status_overview/lyfe_orders_—_status_overview.py"
```

For each file, ask specifically:
1. Has `prod` (or any other active branch) also changed this exact file since the common ancestor? (`git log 2df9968c..prod -- <file>`)
2. If yes, do the two sets of changes touch the **same function**? (E1/E2 above are the two confirmed cases — there may be others not yet checked.)
3. Is this a shared, general-purpose file (`hooks.py`, `order_tracking.py`, SLA detectors) or something scoped purely to 1Click/US-Warehouse logic that `prod` has no reason to touch? Files in the second category are lower risk by nature.

**Known LOW risk** (scoped purely to new 1Click/routing logic, `prod` has no reason to have touched these independently): `lyfe_order_routing.py`, `oneclick_api.py`, the `leg1_sla.py`/`lyfe_order_leg1.py`/`transfer_order_receipt.py` SLA detector trio, `sla_rule.json`'s two new Select options, `order_analysis_tw.*` (this whole page is this session's own build), `create_sales_invoice.py`, `order_export_utils.py`.

**Known HIGHER risk** (general-purpose files `prod` is also actively changing): `hooks.py`, `order_tracking.py`, `lyfe_order.py` (very large, high-traffic file — a full order-lifecycle file `prod` almost certainly also touches for unrelated reasons), `close_engine.py`/`rule_evaluator.py` (the SLA engine core, shared by every SLA rule in the system, not just F2/F3), `lyfe_orders_status_overview.py` and its legacy Report twin (the Factory pending-list — a page other teams rely on daily, touched twice this session for the two `Factory Assignment` status changes).

---

## Quick reference — status & tag meanings (2026-08-19, current)

| Status | Meaning | Where set | Old status this replaced |
|---|---|---|---|
| **Factory Assignment** | Normal Factory handoff — used for a plain order AND now also for `INDIA_DIRECT_DROPSHIP` AND the "Awaiting India Components" hold | `on_update`'s generic warehouse-changed logic, `_set_factory_assignment_status` | — |
| **Submitted to 1Click** | Successfully submitted, awaiting dispatch | `_submit_single_oneclick_order`, `_split_and_submit_oneclick`, `_maybe_resume_oneclick_order` | — |
| **1Click Error** | Submission or resume failed | `run_oneclick_fulfillment`/`_maybe_resume_oneclick_order` except blocks | — |
| **Split** | Parent archived after a plain mixed-stock split (§ old doc's §6 — a *different* mechanism from Route B/D, still exists unchanged) | `_split_and_submit_oneclick` | — |
| **New** | MIXED order awaiting human Confirm Split | default | — |

| `fulfillment_route_tag` value | Meaning | Cleared when |
|---|---|---|
| *(blank)* | Not on a special India-only hold | — |
| **Pending India Dispatch** | `INDIA_DIRECT_DROPSHIP` — nothing route-specific pending, informational only | Never explicitly cleared (order proceeds through normal Factory dispatch) |
| **Awaiting India Components** | Genuine Route D hold — waiting for Transfer Order Received | `_maybe_resume_oneclick_order`, on both success and failure |

---

*This file reflects the code as of commit `6226fdb` on `prod-us_warehouse_integration`. If any expected result above doesn't match what you observe, treat it as a real bug and report it — every behavior described here has been directly verified against live code and/or live data at least once during this session, not assumed from the design doc.*
