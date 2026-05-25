# 1Click Logistics & US Warehouse — User Guide

**Organisation:** Lyfe Hardware
**Last Updated:** 2026-05-25

---

## Table of Contents

1. [Overview](#overview)
2. [How Orders Are Routed](#how-orders-are-routed)
3. [What You See on Each Order](#what-you-see-on-each-order)
4. [Use Cases](#use-cases)
   - [Case 1 — All items in US Warehouse](#case-1--all-items-in-us-warehouse)
   - [Case 2 — All items ship from Factory (India direct)](#case-2--all-items-ship-from-factory-india-direct)
   - [Case 3 — Mixed stock — order auto-split](#case-3--mixed-stock--order-auto-split)
   - [Case 4 — Two-leg shipment (Factory → US Warehouse → Customer)](#case-4--two-leg-shipment-factory--us-warehouse--customer)
   - [Case 5 — Force US Warehouse override](#case-5--force-us-warehouse-override)
   - [Case 6 — Force Factory / India override](#case-6--force-factory--india-override)
   - [Case 7 — Order fails to submit (1Click Error)](#case-7--order-fails-to-submit-1click-error)
   - [Case 8 — Tracking not showing yet](#case-8--tracking-not-showing-yet)
   - [Case 9 — Order contains a fee line](#case-9--order-contains-a-fee-line)
   - [Case 10 — CS team assigns FC child to Factory](#case-10--cs-team-assigns-fc-child-to-factory)
5. [Split Orders — How They Work](#split-orders--how-they-work)
6. [Transfer Orders (Two-Leg Shipments)](#transfer-orders-two-leg-shipments)
7. [Order Statuses Explained](#order-statuses-explained)
8. [Item Badges Explained](#item-badges-explained)
9. [Route Plan Options](#route-plan-options)
10. [Oneclick Settings](#oneclick-settings)
11. [Quick Reference](#quick-reference)

---

## Overview

When a ShipStation order arrives, the system automatically:

1. Checks real-time stock at the US Warehouse (via 1Click Logistics)
2. Decides where to fulfil from — US Warehouse, Factory, or both
3. Submits the order to 1Click Logistics for picking and shipping
4. Syncs tracking numbers back to the order every hour

**No manual steps are needed for standard orders.** Staff only need to act when an order shows `1Click Error`, or when a CS team member assigns a factory child order.

> This integration only activates for **ShipStation orders**. Orders created manually or from Shopify do not trigger 1Click automatically.

---

## How Orders Are Routed

The system checks live US Warehouse inventory for every item and applies these rules:

| Stock situation | What happens |
|---|---|
| **All items in US stock** | One order — ships entirely from US Warehouse via 1Click |
| **No items in US stock — US destination** | One order — Factory ships to US Warehouse, which ships to customer (two-leg) |
| **No items in US stock — non-US destination** | One order — Factory ships direct to customer. CS team then assigns to Factory via normal workflow |
| **Some items in US, some not** | Order is split — US items go to 1Click, Factory items go through CS → Factory workflow |

The routing decision happens automatically in the background within seconds of the order arriving.

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

### Case 5 — Force US Warehouse override

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

### Case 6 — Force Factory / India override

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

### Case 7 — Order fails to submit (1Click Error)

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

### Case 8 — Tracking not showing yet

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

### Case 9 — Order contains a fee line

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

### Case 10 — CS team assigns FC child to Factory

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
| No items in US stock, non-US address | Auto | → Factory — stays New, CS assigns to factory |
| No items in US stock, US address | Auto | → Two-leg: Factory ships to US Warehouse, then to customer |
| Some in US, some not | Auto | → Auto-split: -US child to 1Click, -FC child stays New for CS |
| Need to force US | Force US | → US Warehouse — no stock check |
| Need to force India | Force India | → Factory — no stock check |

### Something looks wrong?

| Symptom | What to check |
|---|---|
| Order stuck at `New` for more than 10 minutes | Background jobs may be paused — contact tech support |
| Status shows `1Click Error` | Open order → 1Click tab → read **1Click Error** field → fix → ask admin to retry |
| Wrong warehouse assigned | Set **Route Plan** to Force US or Force India |
| No tracking after 24 hours | Check **1Click Error** field; if blank, contact 1Click with the **1Click PO** number |
| FC child order not moving | CS team needs to assign it to factory via normal workflow |
| Fee item affecting routing | Rename item to exactly `Custom Fee` or `Customs Fee` |
| Split order — where are the child orders? | Search for parent name + `-US` or `-FC` (e.g. `LYF-SH-2025-0400-US`) |

---

*For configuration changes or technical issues, contact your system administrator.*
