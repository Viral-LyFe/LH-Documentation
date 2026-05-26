# 1Click Logistics & US Warehouse — User Guide

**Organisation:** Lyfe Hardware
**Last Updated:** 2026-05-26

---

## Table of Contents

1. [Overview](#overview)
2. [How Orders Are Routed](#how-orders-are-routed)
3. [Route Suggestion Tool](#route-suggestion-tool)
4. [What You See on Each Order](#what-you-see-on-each-order)
5. [Use Cases](#use-cases)
   - [Case 1 — All items in US Warehouse](#case-1--all-items-in-us-warehouse)
   - [Case 2 — All items ship from Factory (India direct)](#case-2--all-items-ship-from-factory-india-direct)
   - [Case 3 — Mixed stock — order auto-split](#case-3--mixed-stock--order-auto-split)
   - [Case 4 — Two-leg shipment (Factory → US Warehouse → Customer)](#case-4--two-leg-shipment-factory--us-warehouse--customer)
   - [Case 5 — US warehouse dispatches separately, Factory dispatches separately](#case-5--us-warehouse-dispatches-separately-factory-dispatches-separately)
   - [Case 6 — Factory forwards to US Warehouse, combined shipment to customer](#case-6--factory-forwards-to-us-warehouse-combined-shipment-to-customer)
   - [Case 7 — Force US Warehouse override](#case-7--force-us-warehouse-override)
   - [Case 8 — Force Factory / India override](#case-8--force-factory--india-override)
   - [Case 9 — Order fails to submit (1Click Error)](#case-9--order-fails-to-submit-1click-error)
   - [Case 10 — Tracking not showing yet](#case-10--tracking-not-showing-yet)
   - [Case 11 — Order contains a fee line](#case-11--order-contains-a-fee-line)
   - [Case 12 — CS team assigns FC child to Factory](#case-12--cs-team-assigns-fc-child-to-factory)
6. [Which Tracking Number Is Synced to ShipStation?](#which-tracking-number-is-synced-to-shipstation)
7. [Split Orders — How They Work](#split-orders--how-they-work)
8. [Transfer Orders (Two-Leg Shipments)](#transfer-orders-two-leg-shipments)
9. [Order Statuses Explained](#order-statuses-explained)
10. [Item Badges Explained](#item-badges-explained)
11. [Route Plan Options](#route-plan-options)
12. [Oneclick Settings](#oneclick-settings)
13. [Quick Reference](#quick-reference)

---

## Overview

When a ShipStation order arrives, the system automatically:

1. Checks real-time stock at the US Warehouse (via 1Click Logistics)
2. Decides where to fulfil from — US Warehouse, Factory, or both
3. Submits the order to 1Click Logistics for picking and shipping
4. Syncs tracking numbers back to the order every hour
5. Pushes the **last-leg tracking number** (the final delivery to the customer) to ShipStation automatically — even when the order has two legs

**No manual steps are needed for standard orders.** Staff only need to act when an order shows `1Click Error`, or when a CS team member assigns a factory child order.

> This integration only activates for **ShipStation orders**. Orders created manually or from Shopify do not trigger 1Click automatically.

---

## How Orders Are Routed

The system checks live US Warehouse inventory for every item and applies these rules:

| Stock situation | Routing outcome | What happens |
|---|---|---|
| **All items in US stock** | `US_FULL` | One order — ships entirely from US Warehouse via 1Click |
| **Tubing in US stock, other components not** | `MIXED_TUBING_US_COMPONENTS_INDIA_TO_US` | Components ship India → US Warehouse, kit with US tubing, ship combined to customer. Saves volumetric weight vs. shipping tubing internationally |
| **No items in US stock — US destination** | `INDIA_TO_US_TO_CUSTOMER` | Factory ships to US Warehouse first, then US Warehouse ships to customer (two-leg) |
| **No items in US stock — non-US destination** | `INDIA_DIRECT_DROPSHIP` | Factory ships direct to customer. CS assigns to Factory via normal workflow |
| **Some items in US, some not** | Split | Order is split — US items go to 1Click, Factory items go through CS → Factory workflow |

The routing decision happens automatically in the background within seconds of the order arriving.

**The CS team does not set the route manually.** Use the [Route Suggestion Tool](#route-suggestion-tool) to review the recommendation and optionally accept it.

---

## Route Suggestion Tool

The **Get Route Suggestion** button lets the CS team see the system's recommended fulfilment route for any order — **before committing to it**. It is purely advisory: the ERP never auto-assigns a warehouse or auto-routes without CS approval, except when all items are confirmed in US stock.

### How to use it

1. Open a Lyfe Order in **New**, **Internal Review**, or **Factory Assignment** status
2. Click **Actions → Get Route Suggestion**
3. A coloured banner appears at the top of the form showing:
   - The recommended route
   - Why the system is suggesting it
   - An "Assign to US Warehouse" button — **only visible when all items are in US stock** (`US_FULL`)

### What each suggestion means

| Suggestion | What it means | What to do |
|---|---|---|
| **All items available in US Warehouse — ship from US (1Click)** | Live stock check confirmed all items are in US stock. | Click **Assign to US Warehouse** if you agree. The order is assigned and submitted to 1Click. |
| **2-Leg: Components India → US (combine with US tubing) → Customer** | Tubing is in the US. Other parts must come from India. The combined package ships from US. | No button — CS team manually decides how to split and routes accordingly. |
| **2-Leg: India → US Warehouse → Customer** | No US stock; customer is in the US. Factory will ship to US Warehouse first. | No button — Factory will create a Transfer Order for the India leg, then 1Click handles the US leg. |
| **Direct: Ship from India directly to customer** | No US stock; non-US destination. | No button — treat as a normal factory order. |

### Example

> Order `LYF-SH-2026-0512` arrives for a US customer with 2x Desk Stand and 4x 19L Tubing.
>
> CS clicks **Get Route Suggestion**. The banner shows:
> *"2-Leg: Ship components India → US Warehouse (combine with US tubing) → Customer"*
>
> This tells CS: the tubing is already in the US Warehouse; only the Desk Stand needs to come from India. CS creates the arrangement (Factory ships Desk Stand to US Warehouse, US Warehouse kits with tubing and ships combined to customer).

> Changing **Route Plan** to `Force US` or `Force India` overrides the suggestion and bypasses the live stock check entirely.

---

## What You See on Each Order

### Item Badges

Every item row on the order shows a badge indicating where it will be fulfilled from:

| Badge | Meaning |
|---|---|
| 🟢 **US Warehouse** | This item has stock at the US Warehouse — will ship from there |
| 🟡 **Factory** | This item is not in US stock — will come from Factory |
| ⚫ **Excluded** | Fee or charge line — not a physical item, not checked |

### Key Fields

| Field | Where to find it | What it shows |
|---|---|---|
| **Warehouse (Fulfilled From)** | Order header | Which warehouse this order is assigned to |
| **Route Plan** | Order header | Auto / Force US / Force India |
| **Routing Outcome** | Order header (read-only) | The decision the system made |
| **1Click Order ID** | 1Click Logistics tab | The ID 1Click assigned to this order |
| **1Click Tracking Number** | 1Click Logistics tab | Tracking number from 1Click |
| **1Click Status** | 1Click Logistics tab | Current status reported by 1Click |
| **1Click Error** | 1Click Logistics tab | Error details if submission failed |

---

## Use Cases

---

### Case 1 — All items in US Warehouse

**Scenario:** A customer orders 2x Laptop Stand and 1x USB Hub. Both items are in stock at the US Warehouse.

**What happens automatically:**

- Every item row shows 🟢 **US Warehouse**
- Warehouse is set to `US Warehouse - LH`
- Order is submitted to 1Click within seconds
- Status changes to **Submitted to 1Click**
- 1Click Order ID is populated on the 1Click tab

**Within the hour:**

- Tracking number, carrier, and 1Click Status are pulled in automatically
- Customer receives a domestic shipment (typically 2–5 business days)

**No action needed by the team.**

---

### Case 2 — All items ship from Factory (India direct)

**Scenario:** A customer with a non-US shipping address orders 1x Desk Organiser. The US Warehouse has none in stock.

**What happens automatically:**

- Every item row shows 🟡 **Factory**
- Warehouse is set to `Factory - LH`
- Order is **not** submitted to 1Click
- Status stays **New**
- The order appears in the normal CS → Factory Assignment workflow

**What the CS team does:**

- Treats this exactly like any other new order
- Reviews and assigns to the Factory user
- Status moves to **Factory Assignment** and factory processes it normally

---

### Case 3 — Mixed stock — order auto-split

**Scenario:** A customer orders 2x Phone Mount (in US stock) and 1x Custom Acrylic Stand (not in US stock).

**What happens automatically:**

- Phone Mount rows show 🟢 **US Warehouse**, Acrylic Stand shows 🟡 **Factory**
- The system creates two child orders and archives the original:

| Order | Contains | Status |
|---|---|---|
| **LYF-SH-2025-0400-US** | 2x Phone Mount | Submitted to 1Click |
| **LYF-SH-2025-0400-FC** | 1x Custom Acrylic Stand | New |
| **LYF-SH-2025-0400** (parent) | — archived — | Split |

**What the CS team does for the FC child:**

- The `-FC` order appears as a normal **New** order in the list
- CS team reviews and assigns to Factory (same as any other order)
- Factory processes and dispatches it

**Customer receives two separate shipments** — US items arrive first, Factory items follow.

---

### Case 4 — Two-leg shipment (Factory → US Warehouse → Customer)

**Scenario:** A US customer orders items that are not in US stock. The system decides Factory should ship to US Warehouse first, then US Warehouse ships to the customer.

**What happens automatically:**

- Routing Outcome is set to `INDIA_TO_US_TO_CUSTOMER`
- **Two-Leg Shipment** flag is checked on the order
- Order is submitted to 1Click for the US → Customer leg

**Leg 1 (Factory → US Warehouse):**

- A **Transfer Order** is created to track the inbound India shipment
- Factory ships the goods and enters the Leg 1 tracking number on the Transfer Order
- US Warehouse staff mark the Transfer Order as **Received** when goods arrive

**Leg 2 (US Warehouse → Customer):**

- 1Click ships from US Warehouse to the customer
- Leg 2 tracking number is synced automatically
- Both Leg 1 and Leg 2 tracking numbers are visible on the order

---

### Case 5 — US Warehouse dispatches separately, Factory dispatches separately

**Scenario:** Order `LYF-SH-2026-0880` is split. The `-US` child contains 2x Phone Stand (in US stock) and the `-FC` child contains 1x Custom Frame (from Factory). Both warehouses ship their part independently on the same day.

**What happens:**

- `-US` child ships first → 1Click generates tracking `1Z999AA10123456784` (UPS). This tracking is pushed to ShipStation immediately.
- Factory ships the `-FC` child the same day → tracking `784899456781` (FedEx) is entered on the order. The system detects this is a new last-leg tracking and re-pushes `784899456781` to ShipStation automatically.

**Result on ShipStation:**

| Order | Tracking pushed to ShipStation |
|---|---|
| `LYF-SH-2026-0880-US` | `1Z999AA10123456784` (UPS) |
| `LYF-SH-2026-0880-FC` | `784899456781` (FedEx) |

**Customer receives two separate shipments.** Each order's ShipStation record shows the correct tracking for that shipment.

> The system always pushes the **most recently set** tracking number to ShipStation. Whichever warehouse dispatches last wins. No manual intervention is needed.

---

### Case 6 — Factory forwards to US Warehouse, combined shipment to customer

**Scenario:** Order `LYF-SH-2026-0991` is a two-leg US shipment. The Factory ships components to the US Warehouse first (Leg 1). The US Warehouse waits for the goods to arrive, then kits everything and ships the combined package to the customer (Leg 2).

**What happens step by step:**

1. Factory ships to US Warehouse → enters Leg 1 tracking `JD014600006228082320` on the Transfer Order
2. US Warehouse receives the goods → marks the Transfer Order as **Received**
3. 1Click picks, kits, and ships the combined package → new tracking `1Z999AA10987654321` (UPS) is generated for the customer delivery
4. The hourly tracking sync pulls `1Z999AA10987654321` into the **Leg 2 Tracking** field on the order
5. The system immediately re-pushes `1Z999AA10987654321` to ShipStation — **replacing** the earlier Leg 1 tracking

**Fields on the order after completion:**

| Field | Value |
|---|---|
| Leg 1 Tracking | `JD014600006228082320` (India → US Warehouse) |
| Leg 2 Tracking | `1Z999AA10987654321` (US Warehouse → Customer) |
| Tracking pushed to ShipStation | `1Z999AA10987654321` ← always the last-mile number |

> Leg 2 Tracking (final delivery to customer) always takes priority over Leg 1 Tracking. ShipStation and the customer see only the delivery tracking, not the inter-warehouse transfer.

---

### Case 7 — Force US Warehouse override

**Scenario:** The CS team knows stock is available at the US Warehouse, but the system is routing to Factory (for example, stock just arrived and the inventory count hasn't updated yet).

**Steps:**
1. Open the Lyfe Order
2. Set **Route Plan → Force US**
3. Save

The stock check is skipped. The order goes directly to the US Warehouse and is submitted to 1Click immediately.

**When to use:**
- Stock was just received and the live count hasn't refreshed yet
- 1Click inventory API had a temporary outage during routing
- A customer arrangement specifically requires US-only fulfilment

---

### Case 8 — Force Factory / India override

**Scenario:** An order has custom-engraved items that must come from India, even though the standard SKU may show US stock.

**Steps:**
1. Open the Lyfe Order
2. Set **Route Plan → Force India**
3. Save

The stock check is skipped. The order goes to the Factory workflow. Nothing is submitted to 1Click.

**When to use:**
- Custom or personalised items that the US Warehouse cannot fulfil
- Customer has specifically requested factory fulfilment
- US Warehouse is temporarily unavailable

---

### Case 9 — Order fails to submit (1Click Error)

**Scenario:** The order could not be submitted to 1Click — for example, the API was temporarily down, or an item SKU was not recognised.

**What you see:**

- Status changes to **1Click Error**
- The **1Click Error** field on the 1Click tab shows a description of what went wrong

**Recovery steps:**

1. Open the order
2. Go to the **1Click Logistics** tab and read the **1Click Error** field
3. Fix the underlying issue:

| Error cause | Fix |
|---|---|
| Item SKU missing or unrecognised | Add or correct the **SKU** field on the item row |
| API credentials expired | Go to **Oneclick Settings** and update the API Key |
| Temporary API outage | Wait a few minutes, then retry |
| Warehouse ID not configured | Go to **Oneclick Settings** and set the correct Warehouse ID |

4. Contact your system administrator to re-trigger the fulfilment once the issue is fixed

---

### Case 10 — Tracking not showing yet

**Scenario:** An order was submitted to 1Click but the tracking number field is still empty.

This is normal. Tracking is updated **automatically every hour** for all orders in **Submitted to 1Click** status. No manual action is needed.

**Typical progression:**

| Time | What you see |
|---|---|
| Just after submission | Tracking: *(empty)*, 1Click Status: *(empty)* |
| Once 1Click processes it | Tracking number and carrier appear |
| Once carrier picks up | 1Click Status updates (e.g. *Shipped*, *In Transit*) |

If tracking is still missing after **24 hours**, check the **1Click Error** field. If it is blank, contact 1Click support and quote the **1Click PO** number shown on the order.

---

### Case 11 — Order contains a fee line

**Scenario:** An order includes a `"Customs Fee"` line alongside physical products.

Fee lines (any item named exactly **Custom Fee** or **Customs Fee**) are automatically excluded from everything:

- They do **not** affect the stock check
- They do **not** affect which warehouse is chosen
- They are **not** sent to 1Click
- They show the ⚫ **Excluded** badge

**Example:**

```
Items:
  1x Laptop Stand    🟢 US Warehouse  — checked, routed, sent to 1Click
  Customs Fee        ⚫ Excluded       — ignored completely
```

The order routes and submits based on the physical items only.

---

### Case 12 — CS team assigns FC child to Factory

**Scenario:** An order was split (Case 3). The `-FC` child order is sitting in **New** status and needs to be sent to the factory.

**What the CS team does:**

This is identical to the standard factory assignment flow:

1. Open the `-FC` child order (e.g. `LYF-SH-2025-0400-FC`)
2. Assign it to the Factory user
3. Status automatically moves to **Factory Assignment**
4. Factory team processes and dispatches the items
5. Tracking for the factory shipment is entered manually as usual

The `-FC` child order behaves exactly like any other new Lyfe Order routed to factory.

---

## Which Tracking Number Is Synced to ShipStation?

For single-leg orders this is straightforward — the one tracking number is synced. For two-leg and split orders, the system always pushes the **last-mile tracking number** (the number the customer uses to track their delivery), not the inter-warehouse transfer tracking.

### Priority order

| Priority | Field | When it is set |
|---|---|---|
| **1 (highest)** | **Leg 2 Tracking** | Set automatically when 1Click ships the US → Customer leg |
| **2** | **US Leg Tracking** | Set automatically when 17Track detects the US domestic leg |
| **3 (fallback)** | **Tracking Number** | The primary international or factory tracking number |

The system always pushes whichever of these is set at the highest priority. When a higher-priority tracking number arrives (e.g. Leg 2 comes in after the primary tracking was already pushed), ShipStation is updated automatically — no manual action needed.

### Examples

**Single-leg order (direct from India):**
- Tracking `784899456781` (FedEx) is entered → pushed to ShipStation immediately.

**Two-leg order (Factory → US Warehouse → Customer):**
- Factory enters `JD014600006228082320` as the India leg tracking. ShipStation is updated with this temporarily.
- 1Click ships to the customer and the system pulls in Leg 2 Tracking `1Z999AA10987654321`.
- ShipStation is automatically updated to `1Z999AA10987654321` — this is what the customer tracks.

**Split order (both legs dispatch independently):**
- `-US` child ships → `1Z999AA10123456784` pushed to ShipStation for the `-US` order.
- `-FC` child ships later → `784899456781` pushed to ShipStation for the `-FC` order.
- Each child order has its own ShipStation record with its own correct tracking.

---

## Split Orders — How They Work

When an order is split, three orders exist in the system:

| Order | Role | Status |
|---|---|---|
| `LYF-SH-2025-0400` | Parent (archived) | **Split** |
| `LYF-SH-2025-0400-US` | US Warehouse child | **Submitted to 1Click** |
| `LYF-SH-2025-0400-FC` | Factory child | **New** (until CS assigns) |

**Finding split child orders:**
- Open the parent order and navigate to its child orders, or
- Search directly for `LYF-SH-2025-0400-US` or `LYF-SH-2025-0400-FC`

**Important:**
- The parent order (`Split` status) is archived — do not try to work on it
- All activity happens on the two child orders
- The `-US` child gets its own 1Click Order ID, tracking number, and status
- The `-FC` child follows the normal CS → Factory Assignment workflow

---

## Transfer Orders (Two-Leg Shipments)

Transfer Orders are used when Factory needs to send goods **to the US Warehouse first**, before the US Warehouse ships to the customer.

### Lifecycle

```
Draft  →  Shipped  →  Received
```

| Stage | Who acts | What to do |
|---|---|---|
| **Draft** | Created automatically | — nothing required |
| **Shipped** | Factory team | Enter Ship Date, Leg 1 tracking number, and Carrier |
| **Received** | US Warehouse staff | Change status to Received when goods arrive |

Once **Received**, the US Warehouse can proceed to fulfil the customer leg via 1Click.

### Key Fields on a Transfer Order

| Field | Description |
|---|---|
| **Linked Order** | The Lyfe Order this transfer belongs to |
| **Ship Date** | Date the factory dispatched the goods |
| **Expected Arrival** | Estimated date goods arrive at US Warehouse |
| **Leg 1 Tracking** | Tracking number for India → US Warehouse leg |
| **Carrier** | Carrier used for the India → US leg |
| **Carton Count** | Number of cartons shipped |

---

## Order Statuses Explained

| Status | What it means | Action needed |
|---|---|---|
| **New** | Order received, routing complete or awaiting CS action | CS team assigns to factory if it is a factory order |
| **Factory Assignment** | CS has assigned to factory | Factory team processes and dispatches |
| **Submitted to 1Click** | Fulfillment sent to US Warehouse via 1Click | None — tracking syncs automatically every hour |
| **1Click Error** | Submission to 1Click failed | Check the 1Click Error field, fix the issue, contact admin to retry |
| **Split** | Parent order archived — child orders are active | Work on the -US and -FC child orders |
| **Pending India Dispatch** | *(Reserved — not currently in active use)* | — |

---

## Item Badges Explained

| Badge | What it means |
|---|---|
| 🟢 **US Warehouse** | Item is in stock at the US Warehouse and will ship from there |
| 🟡 **Factory** | Item is not in US stock and will ship from Factory |
| ⚫ **Excluded** | Fee or charge line — not a physical product, ignored for routing and fulfilment |

These badges are set automatically when the order arrives and are visible on every row in the Order Items table.

---

## Route Plan Options

The **Route Plan** field lets you override the automatic routing decision.

| Option | What it does | When to use |
|---|---|---|
| **Auto** (default) | Checks live US Warehouse stock and routes accordingly | Always — for standard orders |
| **Force US** | Sends to US Warehouse regardless of stock check | When stock just arrived and hasn't synced, or a specific arrangement requires US fulfilment |
| **Force India** | Sends to Factory regardless of stock check | Custom/personalised items, or when US Warehouse is unavailable |

> Changing Route Plan and saving re-triggers the routing and submission immediately.

---

## Oneclick Settings

The integration is configured in **Oneclick Settings** (single settings page, administrator access only).

| Setting | What it controls |
|---|---|
| **Enable 1Click Integration** | Turns the entire integration on or off |
| **API Key** | Authentication credential for the 1Click API |
| **US Warehouse ID** | The ID of the US Warehouse in 1Click's system |
| **Factory Warehouse ID** | The ID of the Factory in 1Click's system |
| **Default Ship Carrier** | Carrier used for all orders (e.g. UPS, FedEx) |
| **Default Ship Service** | Service level (e.g. Ground, 2-Day) |
| **Webhook Secret** | Security key for verifying incoming ShipStation webhooks |

> Only administrators should modify Oneclick Settings. Incorrect Warehouse IDs or an invalid API Key will cause all orders to fail with `1Click Error`.

---

## Quick Reference

### Where will this order be fulfilled from?

| Situation | Route Plan | Result |
|---|---|---|
| All items in US stock | Auto | → US Warehouse — submitted to 1Click |
| Tubing in US, components not | Auto | → 2-Leg: India components → US Warehouse (kit with tubing) → customer |
| No items in US stock, non-US address | Auto | → Factory direct — stays New, CS assigns to factory |
| No items in US stock, US address | Auto | → 2-Leg: Factory → US Warehouse → customer |
| Some in US, some not (non-tubing) | Auto | → Auto-split: -US child to 1Click, -FC child stays New for CS |
| Need to force US | Force US | → US Warehouse — no stock check |
| Need to force India | Force India | → Factory — no stock check |

### Which tracking goes to ShipStation?

| Order type | Tracking pushed to ShipStation |
|---|---|
| Single-leg (direct) | The primary tracking number |
| Two-leg (Factory → US Warehouse → Customer) | Leg 2 Tracking (US → Customer) once it arrives — overrides Leg 1 |
| Split (both dispatch separately) | Each child order pushes its own tracking independently |

> ShipStation is updated automatically whenever a higher-priority tracking number (Leg 2 > US Leg > Primary) becomes available.

### How to check what route the system recommends

1. Open the order → click **Actions → Get Route Suggestion**
2. Read the banner — it explains the recommended route and why
3. For `US_FULL` orders, click **Assign to US Warehouse** if you agree
4. For all other routes, act manually based on the suggestion

### Something looks wrong?

| Symptom | What to check |
|---|---|
| Order stuck at `New` for more than 10 minutes | Background jobs may be paused — contact tech support |
| Status shows `1Click Error` | Open order → 1Click tab → read **1Click Error** field → fix → ask admin to retry |
| Wrong warehouse assigned | Set **Route Plan** to Force US or Force India |
| No tracking after 24 hours | Check **1Click Error** field; if blank, contact 1Click with the **1Click PO** number |
| ShipStation shows old India tracking, not US tracking | Check if **Leg 2 Tracking** or **US Leg Tracking** is set — system should have re-pushed automatically; if not, contact tech support |
| FC child order not moving | CS team needs to assign it to factory via normal workflow |
| Fee item affecting routing | Rename item to exactly `Custom Fee` or `Customs Fee` |
| Split order — where are the child orders? | Search for parent name + `-US` or `-FC` (e.g. `LYF-SH-2025-0400-US`) |

---

*For configuration changes or technical issues, contact your system administrator.*
