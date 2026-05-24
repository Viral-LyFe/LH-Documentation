# 1Click Logistics & US Warehouse — User Guide

## Table of Contents

1. [Overview](#overview)
2. [How It Works](#how-it-works)
3. [Order Routing](#order-routing)
4. [Use Cases](#use-cases)
   - [Case 1 — All items in US Warehouse](#case-1)
   - [Case 2 — All items ship from Factory](#case-2)
   - [Case 3 — Mixed stock (order auto-split)](#case-3)
   - [Case 4 — Two-leg shipment (Factory → US Warehouse → Customer)](#case-4)
   - [Case 5 — Force US Warehouse](#case-5)
   - [Case 6 — Force Factory / India](#case-6)
   - [Case 7 — Submission error](#case-7)
   - [Case 8 — Tracking updates](#case-8)
   - [Case 9 — Orders with fee lines](#case-9)
5. [Transfer Orders](#transfer-orders)
6. [Order Statuses](#order-statuses)
7. [Quick Reference](#quick-reference)

---

## Overview

When a customer places an order, the system automatically:

1. Receives the order from ShipStation
2. Checks real-time stock at the US warehouse
3. Routes the order to the US warehouse, Factory, or splits it across both
4. Submits the fulfillment request to 1Click Logistics
5. Syncs the tracking number back to the order every hour

No manual steps are needed for standard orders. Staff only need to act when an order shows `1Click Error` or when a manual routing override is required.

---

## How It Works

```
New order arrives from ShipStation
          │
    Check US warehouse stock
          │
    ┌─────┴──────────────┐
All in US           All out of US       Some in US / some not
    │                    │                       │
Submit to           Route to               Auto-split order:
1Click (US)         Factory                  US child → 1Click
                                             Factory child → India dispatch
```

Every order automatically moves through this flow in the background within minutes of arriving.

---

## Order Routing

The **Route Plan** field on a Lyfe Order controls how the system decides where to fulfill from:

| Route Plan | What it does |
|-----------|-------------|
| **Auto** (default) | Checks live US warehouse stock and routes accordingly |
| **Force US** | Sends everything to US warehouse — skips stock check |
| **Force India** | Sends everything to factory — skips stock check |

Each item on the order gets a colored badge showing where it will be fulfilled from:

- 🟢 **US Warehouse** — in stock, fulfilling from US
- 🔵 **Factory** — not in US stock, fulfilling from India
- ⚪ **Excluded** — fee line, not a physical item

---

## Use Cases

---

### Case 1 — All items in US Warehouse {#case-1}

**Scenario:** A customer in Texas orders 2x Laptop Stand and 1x USB Hub. Both items are stocked at the US warehouse.

**What the team sees:**

The order arrives, stock is confirmed, and within minutes the order is submitted to 1Click for picking and shipping. No action needed.

```
Order: LYF-SH-2025-0200
  Status:    Submitted to 1Click
  Warehouse: US Warehouse

  Items:
    2x Laptop Stand   🟢 US Warehouse
    1x USB Hub        🟢 US Warehouse
```

**Next hourly sync:**

```
  Tracking:  1Z999AA10123456784
  Status:    Shipped
```

Customer receives a domestic shipment — typically 2–3 business days.

---

### Case 2 — All items ship from Factory {#case-2}

**Scenario:** A customer orders 1x Desk Organizer. The US warehouse has none in stock.

**What the team sees:**

The order is routed to the factory and does **not** go to 1Click. It appears in the India dispatch workflow.

```
Order: LYF-ET-2025-0301
  Status:    Pending India Dispatch
  Warehouse: Factory

  Items:
    1x Desk Organizer   🔵 Factory
```

The factory team picks this up and dispatches it directly to the customer.

---

### Case 3 — Mixed stock (order auto-split) {#case-3}

**Scenario:** A customer orders 2x Phone Mount (in US stock) and 1x Custom Acrylic Stand (not in US stock).

**What the team sees:**

The system automatically splits the order into two child orders:

```
LYF-SH-2025-0400-US
  Status:    Submitted to 1Click
  Warehouse: US Warehouse
  Items:     2x Phone Mount   🟢

LYF-SH-2025-0400-FC
  Status:    Pending India Dispatch
  Warehouse: Factory
  Items:     1x Custom Acrylic Stand   🔵

LYF-SH-2025-0400 (parent)
  Status:    Split
```

The customer receives two separate shipments — the US items arrive first, factory items follow.

---

### Case 4 — Two-leg shipment (Factory → US Warehouse → Customer) {#case-4}

**Scenario:** A B2B order needs factory goods to first be sent to the US warehouse, which then ships to the customer.

**What the team sees — Leg 1 (Factory → US Warehouse):**

A **Transfer Order** is created to track the inbound shipment:

```
Transfer Order: TO-0042
  Linked Order:      LYF-MN-2025-0501
  Status:            Draft
  Items:             5x Monitor Arm
```

When the factory ships:

```
TO-0042
  Status:              Shipped
  Ship Date:           10 Aug 2025
  Leg 1 Tracking:      1234567890 (DHL)
  Expected Arrival:    17 Aug 2025
```

When goods arrive at US warehouse, staff update status to **Received**.

**Leg 2 (US Warehouse → Customer):**

```
LYF-MN-2025-0501
  Status:         Submitted to 1Click
  US Tracking:    1Z999AA10123456784 (UPS)
```

---

### Case 5 — Force US Warehouse {#case-5}

**Scenario:** CS knows the US warehouse has the stock, but the system is defaulting to Factory (e.g., stock just arrived and hasn't synced yet).

**Steps:**
1. Open the Lyfe Order
2. Set **Route Plan → Force US**
3. Save

The stock check is skipped. All items are sent to the US warehouse and submitted to 1Click immediately.

**When to use:**
- Stock was just received and the live count hasn't updated yet
- 1Click inventory API is temporarily unavailable
- A specific customer arrangement requires US-only fulfillment

---

### Case 6 — Force Factory / India {#case-6}

**Scenario:** An order contains custom-engraved items that must ship from India, even though standard SKUs may show US stock.

**Steps:**
1. Open the Lyfe Order
2. Set **Route Plan → Force India**
3. Save

The stock check is skipped. All items go to the factory workflow. Nothing is submitted to 1Click.

**When to use:**
- Custom or personalised items that cannot ship from US warehouse
- US warehouse is temporarily unavailable
- Customer has specifically requested factory shipment

---

### Case 7 — Submission error {#case-7}

**Scenario:** The order failed to submit to 1Click (e.g., API was down, or item SKU was not recognised).

**What the team sees:**

```
Order: LYF-SH-2025-0600
  Status:         1Click Error
  1Click Error:   "ConnectionError: Max retries exceeded..."
```

**Recovery steps:**
1. Open the order and read the **1Click Error** field to understand the cause
2. Fix the issue:
   - SKU not recognised → correct the item's **Fulfillment SKU** field
   - API credentials expired → update in **Oneclick Settings**
   - Temporary outage → wait and retry
3. Change **Status** back to `New` and save — the submission will retry automatically

---

### Case 8 — Tracking updates {#case-8}

**Scenario:** An order was submitted to 1Click but no tracking number is showing yet.

Tracking syncs **automatically every hour** for all orders in `Submitted to 1Click` status. No manual action is needed.

**Typical progression:**

```
Just submitted:
  Tracking:  (empty)
  Status:    (empty)

A few hours later (carrier picked up):
  Tracking:  1Z999AA10123456784
  Status:    In Transit

After delivery:
  Status:    Delivered
```

If tracking is still missing after 24 hours, check the **1Click Error** field or contact 1Click support with the **1Click PO** number from the order.

---

### Case 9 — Orders with fee lines {#case-9}

**Scenario:** An order has a "Customs Fee" line alongside physical products.

Fee lines (items named **Custom Fee** or **Customs Fee**) are automatically excluded from the stock check. They do not affect routing decisions.

```
Order: LYF-SH-2025-0700
  Items:
    1x Laptop Stand   🟢 US Warehouse
    Customs Fee       ⚪ Excluded
```

The fee line is passed to 1Click for billing purposes but plays no role in deciding which warehouse fulfils the order.

---

## Transfer Orders

Transfer Orders track the **Factory → US Warehouse** leg for two-leg shipments.

### Lifecycle

```
Draft  →  Shipped  →  Received
```

| Stage | Who updates it | What to fill in |
|-------|---------------|----------------|
| **Draft** | Created automatically | — |
| **Shipped** | Factory team | Ship date, Leg 1 tracking number, carrier |
| **Received** | US warehouse staff | Change status to Received |

Once marked **Received**, the US warehouse can proceed to submit Leg 2 to 1Click.

### Key fields

| Field | Description |
|-------|-------------|
| Linked Order | The Lyfe Order this transfer belongs to |
| Ship Date | Date the factory dispatched the goods |
| Expected Arrival | Estimated date goods arrive at US warehouse |
| Leg 1 Tracking | Tracking number for India → US leg |
| Carrier | Carrier used for this leg |
| Carton Count | Number of cartons shipped |

---

## Order Statuses

| Status | Meaning | Action needed |
|--------|---------|--------------|
| **New** | Order received, awaiting routing | None — automatic |
| **Submitted to 1Click** | Fulfillment sent to US warehouse | None — tracking syncs hourly |
| **Pending India Dispatch** | Routed to factory | Factory team to dispatch |
| **1Click Error** | Submission failed | Check error field, fix, retry |
| **Split** | Parent split into US + Factory child orders | See child orders |

---

## Quick Reference

### Which warehouse will this order go to?

| Situation | Route Plan | Result |
|-----------|-----------|--------|
| All items in US stock | Auto | → US Warehouse, submitted to 1Click |
| All items out of US stock | Auto | → Factory, pending India dispatch |
| Some in US, some not | Auto | → Auto-split into two child orders |
| Override needed | Force US | → US Warehouse, no stock check |
| Override needed | Force India | → Factory, no stock check |

### Something looks wrong?

| Symptom | What to check |
|---------|--------------|
| Order stuck at `New` for more than 10 minutes | Background jobs may be paused — contact tech support |
| Status shows `1Click Error` | Open order → read **1Click Error** field → fix → reset to `New` |
| Wrong warehouse assigned | Set **Route Plan** to Force US or Force India |
| No tracking after 24 hours | Check **1Click Error** field; if empty, contact 1Click with the **1Click PO** number |
| Fee item affecting routing | Rename item to exactly `Custom Fee` or `Customs Fee` |
