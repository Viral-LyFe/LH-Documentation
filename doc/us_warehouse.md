# India / US Fulfillment Routing — Functional Guide

**Organization:** Lyfe Hardware
**Last Updated:** 2026-08-19

---

## Table of Contents

1. [Overview](#1-overview)
2. [The Four Fulfillment Routes](#2-the-four-fulfillment-routes)
3. [How the System Decides Which Route to Use](#3-how-the-system-decides-which-route-to-use)
4. [Manual Override](#4-manual-override)
5. [Two-Leg Shipments Explained](#5-two-leg-shipments-explained)
6. [What You See on the Order](#6-what-you-see-on-the-order)
7. [Transfer Order — Tracking India to US](#7-transfer-order--tracking-india-to-us)
8. [Tracking Updates for Customers](#8-tracking-updates-for-customers)
9. [Dispatch Deadline (72-Hour Rule)](#9-dispatch-deadline-72-hour-rule)
10. [Route-by-Route Walkthrough](#10-route-by-route-walkthrough)
11. [Order Statuses Explained](#11-order-statuses-explained)
12. [Quick Reference](#12-quick-reference)

---

## 1. Overview

Lyfe Hardware fulfills orders from two locations:

| Location | Who Handles It | Used When |
|---|---|---|
| **US Warehouse** (1Click Logistics) | 1Click picks, packs, and ships to the customer | Items are available in US stock |
| **India (Factory)** | India team finishes and ships | Items are not in US stock |

When an order arrives, the system automatically checks what stock is available at the US Warehouse and decides the best fulfillment path — no manual steps needed for the decision itself.

There are **four possible routes**, and the system picks the right one automatically. A manager can also force a specific route when needed.

---

## 2. The Four Fulfillment Routes

### Route A — US Full
**All items are available at the US Warehouse.**

- 1Click ships directly to the customer from the US.
- This is the fastest path for the customer.
- No involvement from India.

---

### Route B — Mixed (Tubing US + Components from India → US → Customer)
**Tubing is in the US Warehouse, but other components (brackets, hardware, fittings, etc.) are not.**

- India ships the missing components to the US Warehouse.
- Once components arrive, 1Click assembles the full kit and ships to the customer.
- The customer receives one shipment from the US.

---

### Route C — India Direct Dropship
**Nothing is in the US Warehouse, and the customer is outside the US.**

- India ships the complete order directly to the customer.
- No US leg involved.
- The customer receives an international shipment from India.

---

### Route D — India to US to Customer
**Nothing is in the US Warehouse, and the customer is in the US.**

- India ships the complete order to the US Warehouse first.
- Once 1Click receives the goods, they ship to the customer.
- The customer receives a US domestic shipment for the best delivery experience.

---

## 3. How the System Decides Which Route to Use

When a ShipStation order arrives, the system immediately checks the 1Click US Warehouse inventory for every item in the order. Based on what is available, it applies this logic:

```
Are ALL items in US stock?
  → YES  →  Route A (US Full)
  → NO   →  Is tubing in US stock?
               → YES  →  Route B (Mixed)
               → NO   →  Is the customer in the US?
                            → YES  →  Route D (India → US → Customer)
                            → NO   →  Route C (India Direct Dropship)
```

This decision is fully automatic. The result is shown on the order as the **Routing Outcome**.

---

## 4. Manual Override

If a manager needs to force a specific fulfillment path (for example, due to a customer deadline or a special request), they can set the **Route Plan** field on the order:

| Route Plan Setting | What It Does |
|---|---|
| **Auto** (default) | System decides based on stock availability |
| **Force US** | Always use US Warehouse, regardless of stock |
| **Force India** | Always use India; system then decides direct vs forward-to-US based on destination |

When Force US or Force India is selected, an **Override Reason** must be entered. This is recorded for audit purposes.

---

## 5. Two-Leg Shipments Explained

Routes B and D involve two physical shipments before the customer receives their order:

| Leg | Journey | Who Ships | Tracking Shown |
|---|---|---|---|
| **Leg 1** | India → US Warehouse | India team | Internal use (ops visibility) |
| **Leg 2** | US Warehouse → Customer | 1Click Logistics | Sent to customer via ShipStation sync |

The customer only ever sees **Leg 2 tracking** — the US domestic shipment. They receive a normal domestic tracking number and are never aware of the India-to-US leg.

For **Route C (India Direct)**, there is only one leg. The India tracking number is sent directly to the customer.

---

## 6. What You See on the Order

Every Lyfe Order shows the following in the **1Click Logistics** tab:

| Field | What It Shows |
|---|---|
| **Routing Outcome** | Which of the 4 routes was assigned (e.g., US_FULL, INDIA_DIRECT_DROPSHIP) |
| **Route Plan** | Auto, Force US, or Force India |
| **Override Reason** | Why a manual override was used (if applicable) |
| **Two-Leg Shipment** | Checked if this order requires a Leg 1 + Leg 2 journey |
| **Leg 1 Tracking** | Tracking number for India → US leg (Routes B and D) |
| **Leg 2 Tracking** | Tracking number for US → Customer leg (Routes A, B, D) |
| **All Components in US Stock** | Whether all items were found in US stock |
| **Tubing in US Stock** | Whether tubing specifically was found in US stock |
| **Promised Dispatch By** | Deadline by which the order must be dispatched (72 hours from order creation, IST) |
| **Warehouse (Fulfilled From)** | US Warehouse or Factory |
| **1Click Order ID** | The order reference in 1Click's system |
| **1Click Tracking Number** | Tracking from 1Click (same as Leg 2 for US routes) |
| **1Click Status** | Current status from 1Click (e.g., Processing, Shipped) |

---

## 7. Transfer Order — Tracking India to US

For Routes B and D, when India ships goods to the US Warehouse, a **Transfer Order** is created in the system to track that shipment.

The Transfer Order records:
- Which Lyfe Order it belongs to
- The items being shipped and quantities
- The India-to-US carrier and **Leg 1 tracking number**
- Expected arrival date at the US Warehouse
- Number of cartons and any notes

**Status flow:** Draft → Shipped → Received

When the Leg 1 tracking number is entered on the Transfer Order, it is automatically reflected on the linked Lyfe Order as well.

The Transfer Order naming follows the format: `ASN-2026-00001`

---

## 8. Tracking Updates for Customers

The system automatically pushes the correct tracking number to Shopify or Etsy based on the route:

| Route | Tracking Sent to Customer |
|---|---|
| Route A — US Full | Leg 2 (US domestic tracking from 1Click) |
| Route B — Mixed | Leg 2 (US domestic tracking from 1Click) |
| Route C — India Direct | Leg 1 (India-to-customer international tracking) |
| Route D — India to US | Leg 2 (US domestic tracking from 1Click) |

Tracking numbers are updated in ShipStation automatically. ShipStation then notifies Shopify or Etsy as part of its normal order sync — no separate push is needed from this system.

---

## 9. Dispatch Deadline (72-Hour Rule)

Every order is automatically stamped with a **Promised Dispatch By** time at the moment the order is created. This is set to **72 hours after order creation**, calculated in Indian Standard Time (IST).

This is a dispatch deadline — not a delivery deadline. The clock starts the moment the Lyfe Order is created in the system.

If an order is approaching or past its dispatch deadline without a tracking number, it should be flagged for review.

---

## 10. Route-by-Route Walkthrough

### Route A — US Full

1. Order arrives from ShipStation.
2. System checks 1Click US Warehouse — all items are in stock.
3. Routing Outcome is set to **US_FULL**.
4. Order is submitted to 1Click US Warehouse.
5. 1Click picks, packs, and ships to the customer.
6. 1Click sends tracking back to the system.
7. Leg 2 tracking is synced to ShipStation → ShipStation notifies the customer.

**⚠️ Open, unconfirmed semantics that affect correctness — verify these once live testing is possible:**

- **`available` vs `onhand` reservation semantics.** It is not confirmed whether 1Click's `available` field already excludes stock reserved by other pending orders. If it doesn't, two separate orders for the same SKU could both be told "yes, in stock" for a single physical unit, and Route A would submit both to 1Click as if fulfillable. See `oneclick_open_questions.md` Q7.
- **`allWarehouses` flag behavior.** Unconfirmed whether sending `allWarehouses: true` on the Inventory call ignores the `warehouseID` we send and returns stock across every warehouse instead of just the one we asked about. See `oneclick_open_questions.md` Q6.
- **Partial fulfillment at 1Click's end.** Even with the ordered-quantity-vs-available-quantity fix now in place on our side (`check_us_inventory` no longer treats "any stock at all" as "fully in stock" — see the fix noted below), a race condition or stale inventory snapshot between our stock check and 1Click's own fulfillment could still mean the real warehouse has less than what we believed at submission time. What 1Click actually does in that case — reject, partial-ship, or silently backorder — is unconfirmed. See `oneclick_open_questions.md` (partial fulfillment question).
- **GET-with-body on the Inventory endpoint.** Our code sends `inventory`/`warehouseID`/`allWarehouses` as a JSON body on a `GET` request. It is not yet confirmed whether 1Click's server actually reads a body on GET, or only honors query parameters — 1Click's own doc example for this endpoint sends no parameters at all yet still returns data, which raised further unanswered questions rather than resolving this one. See `oneclick_open_questions.md` Q35–39.
- **"Always 200" error pattern, unconfirmed for Create Order specifically.** Other 1Click endpoints we've live-tested (Add New Item) return HTTP 200 with `"success": false"` in the body on failure, rather than a 4xx/5xx status code. If Create Order behaves the same way, our current error handling (`raise_for_status()`-based) would silently treat a failed Route A submission as successful. This must be confirmed before trusting a "successful"-looking Create Order response. See `oneclick_open_questions.md` Q30.

**✅ Already fixed this session (2026-08-14):** `check_us_inventory` previously marked a SKU as "in US stock" if 1Click reported *any* quantity at all (`>= 1`), regardless of how many units the order actually needed — a 2-unit order line with only 1 available was wrongly treated as fully stocked and would have routed as Route A. Now fixed to sum the required quantity across all rows sharing a SKU and compare against that. See `oneclick_test_plan.md` §9a for the test case.

**🔴 Currently blocking any live test of Route A:** `us_warehouse_id` is not yet configured in Oneclick Settings. Every step above from step 2 onward depends on it — without it, `check_us_inventory` short-circuits and every order defaults to Factory regardless of real US stock, so Route A cannot fire in this environment until 1Click provides the correct warehouse ID.

---

### Route B — Mixed

1. Order arrives. System finds tubing in US stock but other components are not.
2. Routing Outcome is set to **MIXED_TUBING_US_COMPONENTS_INDIA_TO_US**. Two-Leg Shipment is checked.
3. Order is submitted to 1Click US Warehouse (on hold, awaiting India components).
4. India team ships the missing components to the US Warehouse.
5. A **Transfer Order** is created with Leg 1 tracking.
6. 1Click receives the components, assembles the complete kit, and ships to the customer.
7. 1Click sends Leg 2 tracking back to the system.
8. Leg 2 tracking is synced to ShipStation → ShipStation notifies the customer.

---

### Route C — India Direct Dropship

1. Order arrives. No items in US stock. Customer is outside the US.
2. Routing Outcome is set to **INDIA_DIRECT_DROPSHIP**. Status set to **Pending India Dispatch**.
3. India team finishes and packs the order.
4. India ships directly to the customer.
5. Leg 1 tracking is entered on the Transfer Order (or directly on the order).
6. Leg 1 tracking is synced to ShipStation → ShipStation notifies the customer.

---

### Route D — India to US to Customer

1. Order arrives. No items in US stock. Customer is in the US.
2. Routing Outcome is set to **INDIA_TO_US_TO_CUSTOMER**. Two-Leg Shipment is checked.
3. Order is submitted to 1Click US Warehouse (on hold, awaiting India shipment).
4. India team ships the complete order to the US Warehouse.
5. A **Transfer Order** is created with Leg 1 tracking.
6. 1Click receives the goods, picks and packs, and ships to the customer.
7. 1Click sends Leg 2 tracking back to the system.
8. Leg 2 tracking is synced to ShipStation → ShipStation notifies the customer.

---

## 11. Order Statuses Explained

| Status | Meaning |
|---|---|
| *(blank / new)* | Order just created, routing not yet run |
| **Awaiting India Components** | Route B / Route D — a Transfer Order was auto-created for the missing components; order is held until it is marked Received |
| **Submitted to 1Click** | Successfully submitted to 1Click; waiting for dispatch |
| **Pending India Dispatch** | India Direct route, or the Factory child of a mixed-stock split — waiting for India/Factory to ship |
| **Split** | Mixed-stock order archived after being split into a US child and a Factory child |
| **1Click Error** | Something went wrong — check the 1Click Error field |

---

## 12. Quick Reference

| Question | Answer |
|---|---|
| Which orders go through this routing? | ShipStation orders only |
| When does the system use the US Warehouse? | When all items are in US stock (Route A), or when tubing is in US stock (Route B), or when India forwards to US (Route D) |
| When does India ship directly to the customer? | Route C — non-US customer, nothing in US stock |
| What is Leg 1? | The India → US Warehouse shipment (Routes B and D) |
| What is Leg 2? | The US Warehouse → Customer shipment (Routes A, B, D) |
| What tracking does the customer see? | Leg 2 for US-destined orders; Leg 1 for India Direct orders |
| Can I force a specific route? | Yes — set Route Plan to Force US or Force India, and enter an Override Reason |
| What is Promised Dispatch By? | 72 hours from order creation (IST) — the dispatch deadline |
| What is a Transfer Order? | The document that tracks India-to-US shipments (Leg 1) |
| Where do I see routing details? | Open the Lyfe Order → 1Click Logistics tab |

---

*For technical issues or configuration changes, contact your system administrator.*

---

---

# BRD Cross-Check — Gaps vs. `ERPNext-India-US-logic-warehouse.docx`

**Compared against:** the routing BRD ("Fulfillment routing brain")
**Last checked:** 2026-08-19
**Terminology note:** the BRD's "Packiyo" = our "1Click Logistics" everywhere below; naming is otherwise 1:1.

This section tracks BRD items **not yet documented above**, or that exist under a different name/shape than the BRD specifies. Verified against code, not just against this doc.

| # | BRD Item | Status | Detail |
|---|---|---|---|
| 1 | `priority_flag` checkbox — bypass queueing | **Different shape** | No boolean bypass flag. Implemented as `order_priority` (Select: Normal/Urgent, etc.) on Lyfe Order, driving SLA stamping + urgent-dispatch reminders (`_handle_priority_change()`, `tasks/urgent_dispatch_reminders.py`). Similar intent, not a literal bypass-queue mechanic. |
| 2 | `leg2_push_to_channel` checkbox (default true) | ✅ Implemented | `leg2_push_to_channel` (Check, default 1) on Lyfe Order — matches BRD exactly. Already listed in §6 above. |
| 3 | `promised_dispatch_by` = order time + 72h, IST | ✅ Implemented | `_stamp_promised_dispatch_by()` computes now(IST)+72h and stores in UTC — matches BRD §9 timezone rule. |
| 4 | **Landed Cost Voucher** for India→US transfer costing (BRD §8) | ❌ Not implemented | No Landed Cost Voucher usage anywhere in the app. Freight/duty apportionment by HSN for US-bound transfers is **not modeled**. Real gap if landed-cost-adjusted margins are needed for US shipments. |
| 5 | Escalation alerts: >48h no Leg-1 label; >24h no US receipt after ASN (BRD §9) | ✅ Implemented | Live SLA detectors: `LyfeOrderLeg1Detector` (48h, no `tracking_number_us`) and `TransferOrderReceiptDetector` (24 **business** hours, Transfer Order Shipped but not Received). Wired into the `*/15 * * * *` SLA scan + hourly escalation scan. **The "Open Items" table below is stale** — it still says this is "Not yet built." |
| 6 | "Leg View" status widget: Label Created → Picked up → Exported → Arrived US → Delivered to US WH (BRD §7) | **Partial** | `Lyfe Order Leg1 Event` child table (`leg1_events`, Warehouse Split tab) + "Log Leg1 Event" button log milestones (Label Created / Exported / Arrived US / Delivered to WH). Manual log, not a carrier-fed automated widget; no "Picked up" stage; Leg-2 has no equivalent sub-status log. |
| 7 | Manual Overrides dashboard tile + Communication/comment audit trail (BRD §5) | **Partial** | Dashboard tile exists: `get_override_rate_data()` (Order Analysis page) counts overrides via `route_plan != Auto`, `override_reason` set, or `order_via_us_warehouse=1`. No Communication/comment-log audit trail — overrides tracked only via plain fields. |
| 8 | US-only Returns/RMA with dispositions (Resell/Discard) — BRD backlog R4 | ❌ Not implemented | A generic Returns Dashboard and `Lyfe Order Return Item` exist, but nothing scoped specifically to US-warehouse RMA dispositions. |
| 9 | Replenishment via min/max or days-of-cover, auto Transfer Orders — BRD backlog R1 | ❌ Not implemented | `US Warehouse Replenishment` doctype exists but is a **manual** batch-tracking record only — no reorder-point/days-of-cover calculation, no auto-TO creation. |
| 10 | `kit_expanded_lines` — persisted exploded-component table for routing checks | **Different shape** | Kit/BOM explosion for routing does happen (`explode_and_check_bom_availability()`, `_explode_order_row_to_components()` in `lyfe_order_routing.py`, backed by Lyfe BOM), but it's an **in-memory runtime explosion**, not a persisted child table named `kit_expanded_lines`. Functionally equivalent for routing purposes. |
| 11 | OTIF / KPI set: cycle time by leg, WIP aging, in-transit aging, override rate, landed cost/unit (BRD §10) | **Partial (2026-08-19 build in progress)** | Override rate exists (item 7). OTIF's on-time half now exists too: `get_otif_by_route_data()` (Order Analysis page) reuses the existing `on_time_delivery_rate` SLA math (factory dispatch vs. Std/Custom/Rework thresholds — the app's real "on-time" proxy, since `promised_delivery_date` is confirmed never populated), split by route plan bucket (US Full / MIXED / India Only). In-Full deliberately deferred — no reliable "shipped complete, no backorder" signal exists on Lyfe Order. Currently reads 0% across all three buckets: every dispatched order predates `routing_outcome`, so none carry a route plan yet — will populate as new routed orders reach dispatch. Still missing: leg-level cycle time, WIP/in-transit aging, landed-cost-per-unit (landed cost explicitly out of scope — Customer Charges fields already cover freight/duty per standing decision). |
| 12 | Compliance data centralization (HS/FDA/FCC/ECCN) — BRD backlog R3 | ❌ Not implemented | Not referenced anywhere in the app; backlog item in both the BRD and here. |

**Action needed:** the "Open Items (Next Steps)" table further below still lists *"SLA breach alerts (>48h no Leg 1, >24h no US Warehouse receipt) — Not yet built"* — this is now incorrect (see item 5). That line should be corrected to reflect the live detectors.

---

---

# India / US Fulfillment Routing — What Was Built & Use Case Verification

**Organization:** Lyfe Hardware
**Last Updated:** 2026-05-11

---

## What Was Built

The following features are now deployed and live.

### 1. Routing Decision Engine

Every ShipStation order automatically runs through a routing check when it is created. The system queries the 1Click US Warehouse inventory for every item in the order and determines which of the four routes applies. The result is stored in the **Routing Outcome** field on the order's 1Click Logistics tab.

The routing decision uses the existing `route_plan` field (Auto / Force US / Force India) as the starting point. If set to Auto, the stock check determines the outcome. If set to Force US or Force India, the stock check is skipped and the outcome is forced.

**Tubing detection:** An item is treated as a tubing item when its Item Group in the Item Master is exactly `"Tubing"`. This is used to distinguish Route B from Route D when only tubing is in US stock.

### 2. New Fields on Lyfe Order (1Click Logistics Tab — Routing Section)

| Field | What It Shows |
|---|---|
| **Routing Outcome** | Which of the 4 routes was assigned |
| **Two-Leg Shipment** | Checked automatically when Leg 1 + Leg 2 are needed |
| **Push Leg 2 to Channel** | Whether Leg 2 tracking is the customer-facing tracking (default: on; flows to ShipStation) |
| **Leg 1 Tracking** | India → US tracking number (auto-copied from Transfer Order) |
| **Leg 2 Tracking** | US → Customer tracking number (auto-filled from 1Click sync) |

### 3. Promised Dispatch By — Auto-Stamped

Every new order is automatically stamped with **Promised Dispatch By = order creation time + 72 hours (IST)**. This field is already on the order and now gets populated automatically on every new order. It was blank before.

### 4. Transfer Order Doctype

A new **Transfer Order** document is now available. Reference number format: `ASN-2026-00001`.

**Auto-created for Route B / Route D:** when an order routes to `MIXED_TUBING_US_COMPONENTS_INDIA_TO_US` or `INDIA_TO_US_TO_CUSTOMER`, the system automatically creates a Transfer Order listing exactly the items that are *not* in US stock (the ones India needs to ship). The Lyfe Order's status is set to **Awaiting India Components** and it is **not** submitted to 1Click yet.

When the India team enters a **Leg 1 Tracking** number on the Transfer Order and saves, the system automatically copies that tracking number to the linked Lyfe Order's **Leg 1 Tracking** field.

**Resume on Received:** once the India team marks the Transfer Order **Received** (goods physically arrived at the US Warehouse), the system automatically re-checks US stock and submits the **full** order — original US-stock items plus the now-arrived India components — to 1Click as a single shipment. The customer gets one package. If the resubmission fails for any reason, the order is marked **1Click Error** the same way a normal submission failure is.

For Route C (India Direct), a Transfer Order is not used — the order is never submitted to 1Click at all; India ships directly, and Leg 1 tracking is entered directly on the Lyfe Order (or, if a Transfer Order is created manually for record-keeping, its Leg 1 tracking is also copied into the main **Tracking Number** field so the existing hourly 17Track/ShipEngine polling picks it up).

### 5. Leg 2 Tracking — Auto-Filled from 1Click

When 1Click dispatches a two-leg order (Routes B or D) after being resumed, the hourly 1Click tracking sync automatically populates **Leg 2 Tracking** on the order in addition to the main Tracking Number field.

### 6. India Direct Dropship — No 1Click Submission

Route C orders (INDIA_DIRECT_DROPSHIP) are not submitted to 1Click. Instead, their status is set to **Pending India Dispatch** and their warehouse is set to `Factory - LH`. Fulfillment is handled entirely by the India team.

### 7. Split-Order Factory Child — Also Pending India Dispatch

For a plain mixed-stock split (Route A logic: some items in US stock, some not, no tubing/routing involvement — the original simpler split), the Factory-bound child order (e.g. `LH001-FC`) is also set to status **Pending India Dispatch**, matching Route C's convention, so CS/Factory can filter for all orders awaiting Factory fulfillment in one place regardless of which path produced them.

### 7. SKU Validation and Auto-Fill

**Auto-extraction from item name:** When an order item has no SKU but the item name contains a pattern like `(SKU: 19L-R222-RB)`, the system automatically extracts the SKU value, checks it against the Item Master, and fills in the `sku` and `erp_item` fields if a match is found.

**Auto-fill erp_item:** When an item has a SKU but no ERP Item linked, the system tries to find and fill the ERP Item automatically.

**Validation block:** For Standard orders in New status, the system blocks saving if any physical item (non-fee, non-adjustment) is missing a SKU. The user must either add the SKU or change the Order Type to Custom. Fee lines (`Custom Fee`, `Customs Fee` in any capitalisation) are always exempt.

---

## Use Case Verification

### Use Case 1 — Route A: All Items in US Stock

**Scenario:** ShipStation order arrives. All items are found in 1Click US Warehouse inventory.

**What the system does:**
1. Routing Outcome set to `US_FULL`
2. Two-Leg Shipment remains unchecked
3. Warehouse set to `US Warehouse - LH`
4. Order submitted to 1Click US Warehouse
5. Per-item badges show 🟢 US Warehouse for all rows
6. Hourly 1Click sync populates Tracking Number and 1Click Tracking Number
7. Order moves to Submitted to 1Click status

**What you see on the order:**
- Routing Outcome: `US_FULL`
- Warehouse: `US Warehouse - LH`
- Status: `Submitted to 1Click`
- Leg 2 Tracking: filled once 1Click ships

---

### Use Case 2 — Route B: Tubing in US, Other Components from India

**Scenario:** ShipStation order arrives. The tubing SKU is found in US stock but one or more other component SKUs are not.

**What the system does:**
1. Routing Outcome set to `MIXED_TUBING_US_COMPONENTS_INDIA_TO_US`
2. Two-Leg Shipment is checked
3. Warehouse set to `US Warehouse - LH` (final shipment comes from US)
4. Order submitted to 1Click US Warehouse
5. Per-item badges show 🟢 US Warehouse for tubing rows, 🟡 Factory for other rows
6. India team ships missing components and creates a Transfer Order with Leg 1 Tracking
7. Leg 1 Tracking is auto-copied to the Lyfe Order
8. Once 1Click ships the complete order, Leg 2 Tracking is populated by hourly sync

**What you see on the order:**
- Routing Outcome: `MIXED_TUBING_US_COMPONENTS_INDIA_TO_US`
- Two-Leg Shipment: ✓
- Warehouse: `US Warehouse - LH`
- Leg 1 Tracking: filled after India ships
- Leg 2 Tracking: filled after 1Click ships

---

### Use Case 3 — Route C: India Direct Dropship (Non-US Customer)

**Scenario:** ShipStation order arrives. No items are in US stock. Customer is outside the US.

**What the system does:**
1. Routing Outcome set to `INDIA_DIRECT_DROPSHIP`
2. Two-Leg Shipment remains unchecked
3. Warehouse set to `Factory - LH`
4. Order is **NOT submitted to 1Click** — no 1Click Order ID is created
5. Status set to `Pending India Dispatch`
6. India team ships directly to customer and creates a Transfer Order with Leg 1 Tracking
7. Leg 1 Tracking is auto-copied to the Lyfe Order and also to the main Tracking Number field
8. The existing hourly 17Track/ShipEngine poll picks up the Tracking Number and updates status to Shipped → Completed when delivered

**What you see on the order:**
- Routing Outcome: `INDIA_DIRECT_DROPSHIP`
- Status: `Pending India Dispatch` → `Shipped` → `Completed`
- Warehouse: `Factory - LH`
- Leg 1 Tracking: filled after India ships
- Tracking Number: same value as Leg 1 Tracking (auto-copied)
- No 1Click Order ID (not submitted to 1Click)

---

### Use Case 4 — Route D: India to US to Customer (US Customer, Nothing in US)

**Scenario:** ShipStation order arrives. No items are in US stock. Customer is in the United States.

**What the system does:**
1. Routing Outcome set to `INDIA_TO_US_TO_CUSTOMER`
2. Two-Leg Shipment is checked
3. Warehouse set to `US Warehouse - LH` (final shipment from US)
4. Order submitted to 1Click US Warehouse (awaiting India inbound)
5. India team ships entire order to US Warehouse and creates a Transfer Order with Leg 1 Tracking
6. Leg 1 Tracking is auto-copied to the Lyfe Order
7. Once 1Click receives and ships, Leg 2 Tracking is populated by hourly 1Click sync
8. Leg 2 Tracking is also stored in the main Tracking Number field

**What you see on the order:**
- Routing Outcome: `INDIA_TO_US_TO_CUSTOMER`
- Two-Leg Shipment: ✓
- Warehouse: `US Warehouse - LH`
- Leg 1 Tracking: filled after India ships
- Leg 2 Tracking: filled after 1Click ships
- Status: `Submitted to 1Click` → `Shipped` → `Completed`

---

### Use Case 5 — Manual Override: Force US

**Scenario:** An order would normally route to India (nothing in US stock) but the manager sets Route Plan to `Force US` due to a customer deadline.

**What the system does:**
1. Stock check is bypassed entirely
2. Routing Outcome set to `US_FULL` regardless of actual stock
3. Order submitted to 1Click US Warehouse
4. Override Reason field is mandatory — save is blocked until a reason is entered

**What you see on the order:**
- Route Plan: `Force US`
- Routing Outcome: `US_FULL`
- Override Reason: manager's note (required)

---

### Use Case 6 — Manual Override: Force India

**Scenario:** An order has all items in US stock, but the manager sets Route Plan to `Force India` (e.g. US stock is reserved for another purpose).

**What the system does:**
1. Stock check is bypassed
2. System checks destination country to decide sub-route
3. If US customer → `INDIA_TO_US_TO_CUSTOMER`; if non-US customer → `INDIA_DIRECT_DROPSHIP`
4. For India Direct → status becomes `Pending India Dispatch`, not submitted to 1Click
5. For India to US → submitted to 1Click US Warehouse

---

### Use Case 7 — Order with Fee Line

**Scenario:** Order contains a `Custom Fee` or `Customs Fee` line item alongside physical items.

**What the system does:**
1. Fee lines are excluded from the stock check — they do not affect routing
2. Fee lines are excluded from 1Click submission
3. Per-item badge shows ⚫ Excluded for fee rows
4. Routing is decided based on physical items only

---

### Use Case 8 — Standard Order with Missing SKU

**Scenario:** ShipStation sends an order where one item has no SKU. The item name is `"Bar Foot Rail Kit (SKU: 19L-R222-RB)"`.

**What the system does:**
1. On import, the system scans the item name and finds `(SKU: 19L-R222-RB)`
2. It checks the Item Master — if a match exists, SKU and ERP Item are auto-filled
3. If no match is found in Item Master, the SKU field stays blank
4. If the order is Standard and in New status and any item still has no SKU, saving is blocked with a clear error message
5. User must either enter the correct SKU or change Order Type to Custom

---

### Use Case 9 — Two-Leg Order Tracking Flow End to End

**Scenario:** Route D order (INDIA_TO_US_TO_CUSTOMER). India ships to US Warehouse. 1Click delivers to customer.

**Step-by-step:**
1. Order created → `INDIA_TO_US_TO_CUSTOMER`, `Two-Leg Shipment` checked, submitted to 1Click
2. India team creates Transfer Order → enters Leg 1 Tracking → saves
3. System auto-copies Leg 1 Tracking to Lyfe Order `leg1_tracking` field
4. 1Click receives goods, packs, ships
5. Next hourly 1Click sync runs → populates `oneclick_tracking_number`, `tracking_number`, and `leg2_tracking`
6. 17Track/ShipEngine hourly poll reads `tracking_number` → updates status to Shipped
7. When delivered → status transitions to Completed, `delivered` date stamped

---

## What Is NOT Changing

- The existing 1Click integration for US Warehouse and Factory routing continues to work for all routes that use 1Click (A, B, D).
- ShipStation remains the source of orders. The routing runs automatically on every ShipStation order.
- The per-item stock badges (🟢 US Warehouse / 🟡 Factory / ⚫ Excluded) on Order Items continue to work as before.
- Manual order creation is not affected by routing. The existing ERPNext split/merge is not part of this system — a separate ShipStation-level split/merge is planned for multi-warehouse fulfillment (see Open Items).
- The hourly ShipEngine/17Track polling is unchanged. Route C orders feed into it automatically via the Leg 1 → Tracking Number copy.
- 1Click orders remain excluded from 17Track/ShipEngine polling (handled by separate 1Click sync).

---

## Open Items (Next Steps)

| Item | Status |
|---|---|
| Tracking pushed to Shopify/Etsy | Handled by ShipStation — tracking numbers flow into ShipStation, which syncs to Shopify/Etsy automatically. No separate push needed. |
| SLA breach alerts (>48h no Leg 1, >24h no US Warehouse receipt) | **Built** — live SLA detectors (`LyfeOrderLeg1Detector`, `TransferOrderReceiptDetector`), scanned every 15 min via the SLA scheduler. See "BRD Cross-Check" section above, item 5. |
| Tubing Item Group name to confirm | Must be set to exactly `"Tubing"` in Item Master for MIXED route detection to work |
| **ShipStation Split & Merge for multi-warehouse fulfillment** | Planned — this is a new ShipStation-specific split/merge, separate from the existing ERPNext split/merge. When a ShipStation order contains items that must ship from different warehouses (e.g. some from US Warehouse, some from Factory), the system will split it in ShipStation into per-warehouse shipments and merge the resulting tracking numbers back to the parent Lyfe Order. This will work alongside the existing 4-route routing logic. |

---

*For technical implementation details, refer to the development plan.*

---

---

# SKU Validation & Item Identification — Functional Plan

**Organization:** Lyfe Hardware
**Last Updated:** 2026-05-11

---

## The Problem

When orders arrive from ShipStation, each order item should have a SKU that links it to a product in the Item Master. In practice, this does not always happen cleanly:

- ShipStation sometimes sends items with no SKU at all.
- The item description (name) typed by a user sometimes contains the SKU embedded in it, like `(SKU: 19L-R222-RB)`, but the SKU field itself is left blank.
- When no SKU is linked to a real Item Master record, the system cannot check US stock, cannot route the order correctly, and cannot submit it to 1Click.

Currently, orders with missing SKUs move through the system without any warning until they fail at fulfillment. This plan adds validation to catch and resolve missing SKUs early — before the order reaches dispatch.

---

## Routing Clarification

The routing logic is **not category-based**. The rule is simple:

> **Any item that is available in the US Warehouse will be picked, packed, and shipped from the US.** The US Warehouse is not reserved for a specific product type — it is used for any item where stock exists.

The system checks every item in the order against the 1Click US Warehouse inventory. If all items are available, the order ships from the US. If any item is missing, the system falls back to the India routing rules. This applies regardless of what type of product the item is.

---

## What the System Will Do Automatically

### 1. Extract SKU from Item Name Description

When an order item has no SKU, the system will scan the `item_name` field for a pattern like:

```
(SKU: 19L-R222-RB)
```

If found, it extracts the SKU value (`19L-R222-RB`), checks whether it exists in the Item Master, and if it does:
- Sets the `sku` field to that value
- Sets the `erp_item` field to the matched Item Master record

This extraction happens automatically when the order is saved or synced from ShipStation.

---

### 2. Auto-Fill erp_item from SKU

If an order item has a SKU but no `erp_item` linked, the system will attempt to find the matching Item Master record using the SKU. If a match is found, `erp_item` is set automatically.

This covers cases where the SKU was present but the Item Master lookup was missed during import.

---

## What the System Will Enforce (Validation)

### 3. Block New Orders with Missing SKU

When an order is in **New** status and the order type is **Standard**, every item row must have a SKU (except fee lines — see below). If any item is missing a SKU, the system will block saving and show an error:

> "SKU is required for Standard orders. Please enter a SKU for: **[item name]**, or change the Order Type to Custom."

The user has two options to resolve this:
- Enter or correct the SKU on the item row
- Change the **Order Type** from Standard to Custom (which removes the SKU requirement)

This block only applies to orders in **New** status. Once an order has moved past New (e.g. it is In Progress or Submitted), this check does not re-run.

---

### 4. Fee Lines Are Always Exempt

Any item whose name is **"Custom Fee"** or **"Customs Fee"** (in any combination of upper or lower case, e.g. `custom fee`, `CUSTOMS FEE`, `Custom Fee`) is treated as a fee line and is **never required to have a SKU**. Fee lines are excluded from all stock checks and from fulfillment submission.

---

## Order Type Behaviour Summary

| Order Type | SKU Required | Stock Checked | Submitted to 1Click |
|---|---|---|---|
| **Standard** | Yes (except fee lines) | Yes | Yes |
| **Custom** | No | No | No (manual fulfillment) |

Changing an order to Custom is the intended escape hatch for orders that contain items with no SKU, one-off custom products, or orders that will be handled outside the normal fulfillment flow.

---

## What the User Sees

### On the Order Items Table

If an item row has no SKU and the order is Standard:
- The row is highlighted with a warning indicator
- Saving is blocked with a clear message naming the problematic item(s)

If the system auto-extracted a SKU from the item name:
- The `sku` and `erp_item` fields are filled in automatically
- No user action needed unless the extracted SKU did not match any Item Master record

### SKU Auto-Extraction — What Gets Extracted

The system looks for this pattern anywhere in the `item_name` field:

| Example Item Name | Extracted SKU |
|---|---|
| `Bar Foot Rail Kit (SKU: 19L-R222-RB)` | `19L-R222-RB` |
| `Custom bracket (SKU:LH-BRACKET-01)` | `LH-BRACKET-01` |
| `Mounting hardware` | *(no extraction — no pattern found)* |

If the extracted SKU does not exist in the Item Master, the field is not changed and the normal validation applies.

---

## Summary of Changes

| What | When It Happens | Who Does It |
|---|---|---|
| Extract SKU from item name description | On every save / ShipStation sync | Automatic |
| Auto-fill erp_item from SKU if missing | On every save / ShipStation sync | Automatic |
| Block save if Standard order has items with no SKU | On save, only in New status | System (user must fix) |
| Fee lines (Custom Fee / Customs Fee) exempt from SKU requirement | Always | Automatic |
| User can bypass by changing Order Type to Custom | Manual | User decision |

---

## What Is Not Changing

- Fee line detection logic (Custom Fee / Customs Fee) remains the same as used in 1Click submission today.
- The existing `erp_item` resolution from ShipStation product ID and line item key continues to work as a fallback after the SKU-based lookup.
- Orders that are already past New status are not re-validated for missing SKUs — only new or freshly imported orders are checked.

---

*For technical implementation details, refer to the development plan.*

---

---

# Tracking — What Exists Today and What Will Change

**Organization:** Lyfe Hardware
**Last Updated:** 2026-05-11

---

## How Tracking Works Today

### The Current Flow (Single-Leg Orders)

Every hour, the system automatically checks the tracking status of all orders that have a tracking number and carrier set. It queries ShipEngine first, then falls back to 17Track if ShipEngine cannot find the shipment. The result updates the order automatically — no manual action needed.

In addition, 17Track sends a real-time webhook to the system whenever a shipment status changes. This means some updates arrive within minutes rather than waiting for the next hourly poll.

**What happens when tracking is updated:**

| Tracking Event | What the System Does |
|---|---|
| Any update (In Transit, Out for Delivery, etc.) | Sets order status to **Shipped**, sets workflow state to **Shipped** |
| Delivery confirmed (status contains "Delivered") | Sets order status to **Completed**, sets workflow state to **Completed**, stamps the **Delivered** date |
| Customs processing detected | Stamps **Customs Entry Date** |
| Customs cleared | Stamps **Customs Release Date** |

When the order is marked Shipped or Completed, the system also pushes the status update back to ShipStation automatically.

### The Two Tracking Legs That Already Exist

The system currently tracks two separate legs for certain orders:

| Field | Label | Used For | Workflow Change on Delivery |
|---|---|---|---|
| `tracking_number` + `carrier` | Main Tracking | Customer-facing shipment | **Yes** — triggers Shipped → Completed workflow |
| `tracking_number_us` + `carrier_us` | US Warehouse Tracking | Forwarded-to-US leg | **No** — updates tracking status fields only, no workflow change |

The US Warehouse tracking (leg forwarded to US) is already being tracked hourly but intentionally does not change the order workflow — the order is only considered "delivered" when the final leg reaches the customer.

### 1Click Orders Are Skipped

Orders that have a 1Click tracking number are automatically excluded from the 17Track/ShipEngine polling. Their tracking is handled separately via the 1Click API sync, which also runs hourly.

---

## What Will Change with the New Routing System

The new 4-route model introduces two explicit tracking legs — **Leg 1** (India → US Warehouse) and **Leg 2** (US Warehouse → Customer). The existing tracking fields and behaviour need to be extended to support this clearly.

### Changes to How Tracking Legs Are Used

| Leg | Route(s) | Tracking Field | Workflow Change on Delivery |
|---|---|---|---|
| **Leg 1 — India → US** | Route B (Mixed), Route D (India→US→Customer) | `leg1_tracking` (new) | **No** — internal ops visibility only |
| **Leg 1 — India → Customer** | Route C (India Direct) | `leg1_tracking` (new) | **Yes** — this is the customer-facing leg; triggers Shipped → Completed |
| **Leg 2 — US → Customer** | Routes A, B, D | `leg2_tracking` (new), also stored in existing `tracking_number` | **Yes** — triggers Shipped → Completed |

### What This Means in Practice

**For Route A (US Full):**
No change. A single tracking number from 1Click is the customer-facing tracking. The existing flow handles it as today.

**For Route B (Mixed) and Route D (India → US → Customer):**
- Leg 1 tracking (India → US Warehouse) is stored but does **not** trigger any workflow change. It is for ops visibility only — so the team can see when the India shipment has arrived at the US Warehouse.
- Leg 2 tracking (US → Customer, from 1Click) is the one that drives the order through Shipped → Completed. This tracking flows into ShipStation, which handles notifying the customer on Shopify/Etsy.

**For Route C (India Direct Dropship):**
- There is only one leg. The India tracking number is both Leg 1 and the customer-facing tracking.
- This tracking number drives the workflow: when it shows Delivered, the order moves to Completed.
- This tracking flows into ShipStation, which notifies the customer on Shopify/Etsy.

### Tracking Number in the Main Field

To keep the existing workflow machinery working without rebuilding it, the **customer-facing tracking number** (whichever leg that is for the route) is also stored in the existing main `tracking_number` field. This means:
- The existing hourly poll and 17Track webhook continue to work without changes.
- The Shipped → Completed transition continues to fire based on `tracking_number` as today.
- Leg 2 (or Leg 1 for India Direct) is always the value in `tracking_number`.

---

## Summary: Tracking Fields Before and After

| Field | Today | After New Routing |
|---|---|---|
| `tracking_number` | Single customer-facing tracking | Still the customer-facing tracking (Leg 2 for US routes; Leg 1 for India Direct) |
| `carrier` | Carrier for the customer-facing shipment | Unchanged |
| `tracking_number_us` | US warehouse forwarding leg (no workflow) | Kept for backwards compatibility; `leg1_tracking` is the new equivalent |
| `carrier_us` | Carrier for US forwarding leg | Kept for backwards compatibility |
| `leg1_tracking` (new) | Does not exist | India → US tracking (ops visibility); for Route C, also the customer tracking copied to `tracking_number` |
| `leg2_tracking` (new) | Does not exist | US → Customer tracking from 1Click; same value also stored in `tracking_number` for Routes A, B, D |
| `oneclick_tracking_number` | 1Click tracking (already skipped by 17Track) | Unchanged |

---

## What Does Not Change

- The hourly ShipEngine/17Track polling continues to run as-is.
- The 17Track real-time webhook continues to update orders when events arrive.
- The Shipped → Completed detection logic (checking for "Delivered" in the tracking status) is unchanged.
- The `delivered` date stamp, `shipped_from_carrier` date stamp, and ShipStation status push all continue to work as today.
- 1Click orders remain excluded from 17Track/ShipEngine polling.

---

*For technical implementation details, refer to the development plan.*
