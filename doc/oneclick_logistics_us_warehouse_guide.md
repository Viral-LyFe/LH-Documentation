# 1Click Logistics & US Warehouse Integration — Complete Guide

## Table of Contents

1. [Overview](#overview)
2. [How the Full Flow Works](#how-the-full-flow-works)
3. [Configuration (Oneclick Settings)](#configuration)
4. [Order Routing Logic](#order-routing-logic)
5. [Use Cases with Examples](#use-cases)
   - [Case 1 — All items in US Warehouse (Full US Fulfillment)](#case-1)
   - [Case 2 — All items out of stock (Factory Direct)](#case-2)
   - [Case 3 — Mixed stock (Auto Split: US + Factory)](#case-3)
   - [Case 4 — Two-leg shipment (Factory → US Warehouse → Customer)](#case-4)
   - [Case 5 — Force US Warehouse (manual override)](#case-5)
   - [Case 6 — Force India / Factory (manual override)](#case-6)
   - [Case 7 — 1Click API error during submission](#case-7)
   - [Case 8 — Tracking sync (hourly background job)](#case-8)
   - [Case 9 — Fee items excluded from stock check](#case-9)
6. [Transfer Orders (Two-Leg Tracking)](#transfer-orders)
7. [Status Reference](#status-reference)
8. [Field Reference](#field-reference)
9. [API & Webhook Reference](#api--webhook-reference)
10. [Quick Reference Card](#quick-reference-card)

---

## Overview

The 1Click Logistics integration automates the entire fulfillment journey for US customers:

1. A customer places an order on Shopify/Etsy/Manual channel
2. ShipStation receives the order and fires a **webhook** to Lyfe Hardware
3. The system **checks real-time stock** at the US warehouse via 1Click API
4. Based on availability, the order is **automatically routed** to US Warehouse, Factory, or split across both
5. The fulfillment order is **submitted to 1Click** for physical picking and shipping
6. **Tracking numbers** are synced hourly back into the Lyfe Order

```
ShipStation ──webhook──▶ Lyfe Order created
                              │
                         Stock check (1Click API)
                              │
               ┌──────────────┼──────────────┐
            All in US     All out of US    Mixed stock
               │               │               │
        US Warehouse        Factory       Auto-split
        submitted to         assigned      US child +
          1Click           (no 1Click      Factory child
                            submission)
```

---

## How the Full Flow Works

### Step-by-step from webhook to shipped

```
1. ShipStation POST  ──▶  /api/method/.../shipstation_webhook.receive
        │
        ▼
2. Verify HMAC-SHA256 signature (webhook_secret)
        │
        ▼
3. Create/Update Lyfe Order  (status: New)
        │
        ▼
4. Enqueue background job: run_oneclick_fulfillment()
        │
        ▼
5. compute_routing_outcome()
   - Check route_plan field: Auto / Force US / Force India
   - If Auto → call _check_1click_availability()
        │
        ▼
6. check_us_inventory()
   - POST to 1Click inventory endpoint with SKUs + us_warehouse_id
   - Each order_items row stamped with fulfillment_source badge:
       "US Warehouse" (in stock) / "Factory" (out of stock) / "Excluded" (fee item)
        │
        ▼
7. assign_warehouse()
   - ALL in US  → warehouse = "US Warehouse - LH", no split
   - ALL out    → warehouse = "Factory - LH",      no split
   - MIXED      → _split_required = True
        │
        ▼
8. create_oneclick_order()
   - No split  → _submit_single_oneclick_order()
   - Split     → _split_and_submit_oneclick()
        │
        ▼
9. 1Click API response saved
   - oneclick_order_id, oneclick_po set
   - Status → "Submitted to 1Click"
        │
        ▼
10. Hourly: sync_tracking_for_submitted_orders()
    - GET tracking from 1Click for all "Submitted to 1Click" orders
    - Updates oneclick_tracking_number, oneclick_status
```

---

## Configuration

Go to **Oneclick Settings** (Single DocType — one record for the whole site).

### Fields

| Field | Example Value | Description |
|-------|--------------|-------------|
| `is_active` | ✅ Checked | Enable/disable entire integration |
| `api_base_url` | `https://icomwms.com` | 1Click base URL |
| `api_key` | `eyJhbGci...` | API token (encrypted) |
| `us_warehouse_id` | `12` | 1Click warehouse ID for US warehouse |
| `factory_warehouse_id` | `7` | 1Click warehouse ID for Factory |
| `inventory_endpoint` | `/api/v1/inventory` | Endpoint for stock check |
| `create_order_endpoint` | `/api/v2/orders` | Endpoint for order submission |
| `tracking_endpoint` | `/api/v2/orders` | Endpoint for tracking fetch |
| `inventory_request_sku_field` | `sku` | Field name to send SKU in inventory request |
| `inventory_response_qty_field` | `available` | Field name to read qty from inventory response |
| `default_ship_carrier` | `UPS` | Carrier sent with all 1Click orders |
| `default_ship_service` | `GROUND` | Service level sent with all orders |
| `webhook_secret` | `abc123...` | ShipStation HMAC-SHA256 secret (encrypted) |

### Minimal setup checklist

- [ ] Set `is_active = 1`
- [ ] Enter `api_base_url` and `api_key` from 1Click
- [ ] Enter `us_warehouse_id` and `factory_warehouse_id` from 1Click
- [ ] Enter `webhook_secret` matching ShipStation's webhook config
- [ ] Set `default_ship_carrier` and `default_ship_service`
- [ ] Confirm correct endpoint paths with 1Click support

---

## Order Routing Logic

The `route_plan` field on a Lyfe Order controls how routing is decided:

| `route_plan` | Behaviour |
|-------------|-----------|
| `Auto` (default) | Calls 1Click inventory API; routes based on stock |
| `Force US` | Skips stock check; sends everything to US Warehouse |
| `Force India` | Skips stock check; sends everything to Factory |

### Routing outcomes

| `routing_outcome` | Meaning |
|------------------|---------|
| `US_FULL` | All items in US stock → single US order submitted to 1Click |
| `MIXED_TUBING_US_COMPONENTS_INDIA_TO_US` | Tubing in US, components from India → two-leg route |
| `INDIA_TO_US_TO_CUSTOMER` | Nothing in US, order ships India → US → Customer |
| `INDIA_DIRECT_DROPSHIP` | Factory ships direct to customer (no US leg) |

### `two_leg_required` flag

Set to `1` when the order must pass through the US warehouse before reaching the customer. This creates a **Transfer Order** for the India → US leg.

---

## Use Cases

---

### Case 1 — All items in US Warehouse (Full US Fulfillment) {#case-1}

**Scenario:** A customer in Texas orders 2x Laptop Stand and 1x USB Hub. Both items are in stock at the US warehouse.

**What happens automatically:**

1. ShipStation webhook fires → Lyfe Order `LYF-SH-2025-0200` created (status: `New`)
2. Stock check: Laptop Stand ✅ 15 units available, USB Hub ✅ 8 units available
3. All items stamped `fulfillment_source = "US Warehouse"`
4. `warehouse = "US Warehouse - LH"`, no split required
5. 1Click order submitted with both items and US warehouse ID
6. Status → `Submitted to 1Click`
7. Next hour: tracking number synced back

**Lyfe Order after processing:**

```
LYF-SH-2025-0200
  warehouse             = US Warehouse - LH
  routing_outcome       = US_FULL
  two_leg_required      = 0
  oneclick_order_id     = OCK-88231
  oneclick_po           = LYF-SH-2025-0200
  status                = Submitted to 1Click

  Items:
    Laptop Stand (SKU: LS-BLK)  → fulfillment_source: [US Warehouse] 🟢
    USB Hub (SKU: UH-SLV)       → fulfillment_source: [US Warehouse] 🟢
```

**Next day after tracking sync:**

```
  oneclick_tracking_number = 1Z999AA10123456784
  oneclick_status          = Shipped
```

---

### Case 2 — All items out of stock (Factory Direct) {#case-2}

**Scenario:** A customer orders 1x Desk Organizer (SKU: DO-WHT). The US warehouse has 0 units in stock.

**What happens automatically:**

1. Lyfe Order `LYF-ET-2025-0301` created
2. Stock check: Desk Organizer ❌ 0 units available
3. Item stamped `fulfillment_source = "Factory"`
4. `warehouse = "Factory - LH"`, no split, `two_leg_required = 0`
5. Order **NOT submitted to 1Click** — routed to India factory workflow
6. Status → `Pending India Dispatch`

**Lyfe Order after processing:**

```
LYF-ET-2025-0301
  warehouse           = Factory - LH
  routing_outcome     = INDIA_DIRECT_DROPSHIP
  two_leg_required    = 0
  status              = Pending India Dispatch

  Items:
    Desk Organizer (SKU: DO-WHT) → fulfillment_source: [Factory] 🔵
```

The factory team picks this up in the normal India dispatch workflow.

---

### Case 3 — Mixed stock (Auto Split: US + Factory) {#case-3}

**Scenario:** A customer orders 2x Phone Mount (in US stock) and 1x Custom Acrylic Stand (not in US stock). The system must split the order.

**What happens automatically:**

1. Lyfe Order `LYF-SH-2025-0400` created
2. Stock check:
   - Phone Mount ✅ 20 units in US warehouse
   - Custom Acrylic Stand ❌ 0 units in US warehouse
3. `_split_required = True` → `_split_and_submit_oneclick()` called
4. Two child orders created:

```
LYF-SH-2025-0400-US  (US child)
  warehouse = US Warehouse - LH
  Items: 2x Phone Mount
  Status → Submitted to 1Click
  oneclick_order_id = OCK-91002

LYF-SH-2025-0400-FC  (Factory child)
  warehouse = Factory - LH
  Items: 1x Custom Acrylic Stand
  Status → Pending India Dispatch

LYF-SH-2025-0400  (parent)
  Status → Split
```

**Customer receives two separate shipments:**
- One from US warehouse (fast — 2-3 days domestic)
- One from factory (longer — international)

---

### Case 4 — Two-leg shipment (Factory → US Warehouse → Customer) {#case-4}

**Scenario:** A large B2B order requires the factory to pre-ship inventory to the US warehouse, which then fulfills to the customer. `routing_outcome = INDIA_TO_US_TO_CUSTOMER`, `two_leg_required = 1`.

**What happens:**

1. Lyfe Order `LYF-MN-2025-0501` created with `two_leg_required = 1`
2. A **Transfer Order** is created to track Leg 1 (India → US):

```
Transfer Order: TO-0042
  lyfe_order     = LYF-MN-2025-0501
  status         = Draft
  leg1_tracking  = (empty until factory ships)
  carrier_leg1   = DHL
  items:
    5x Monitor Arm (SKU: MA-BLK)
```

3. Factory ships → Transfer Order updated:

```
TO-0042
  status        = Shipped
  ship_date     = 2025-08-10
  leg1_tracking = 1234567890
  expected_arrival_us = 2025-08-17
```

4. Goods arrive at US warehouse → Transfer Order status → `Received`
5. 1Click order now submitted for Leg 2 (US → Customer):

```
LYF-MN-2025-0501
  tracking_number_us = 1Z999AA10123456784
  carrier_us         = UPS
  status             = Submitted to 1Click
```

---

### Case 5 — Force US Warehouse (manual override) {#case-5}

**Scenario:** CS knows the US warehouse has stock even though the API might be stale. CS wants to force US fulfillment without waiting for a stock check.

**Steps:**
1. Open the Lyfe Order
2. Set `route_plan = "Force US"`
3. Save

**What happens:**
- Stock check is completely skipped
- All items stamped `fulfillment_source = "US Warehouse"`
- `warehouse = "US Warehouse - LH"`
- Order submitted to 1Click immediately

**Use when:**
- 1Click inventory API is temporarily down
- Stock was just received and the API hasn't updated yet
- Special customer arrangement

---

### Case 6 — Force India / Factory (manual override) {#case-6}

**Scenario:** A bulk order for custom-engraved items that can only be fulfilled from India, even though some SKUs may show US stock.

**Steps:**
1. Open the Lyfe Order
2. Set `route_plan = "Force India"`
3. Save

**What happens:**
- Stock check skipped
- All items stamped `fulfillment_source = "Factory"`
- `warehouse = "Factory - LH"`
- No 1Click submission — enters India factory workflow

**Use when:**
- Custom/personalized items that can't ship from US warehouse
- US warehouse is temporarily unavailable
- Order explicitly requested to be factory-shipped

---

### Case 7 — 1Click API error during submission {#case-7}

**Scenario:** The 1Click API is down or returns an error when submitting order `LYF-SH-2025-0600`.

**What happens automatically:**
- The `run_oneclick_fulfillment()` job catches the exception
- Status set to `1Click Error`
- Last 1000 characters of the traceback saved to `oneclick_error` field

**Lyfe Order after error:**

```
LYF-SH-2025-0600
  status          = 1Click Error
  oneclick_error  = "ConnectionError: HTTPSConnectionPool(host='icomwms.com', port=443):
                     Max retries exceeded with url: /api/v2/orders
                     [last 1000 chars of traceback]"
```

**Recovery steps:**
1. Check `oneclick_error` field for the root cause
2. Fix the issue (e.g., API key expired → update in Oneclick Settings)
3. Manually trigger fulfillment by changing `status` back to `New` and saving, or via developer console:
   ```python
   frappe.enqueue(
       "lh.lyfe_hardware.doctype.lyfe_order.lyfe_order.run_oneclick_fulfillment",
       lyfe_order_name="LYF-SH-2025-0600"
   )
   ```

---

### Case 8 — Tracking sync (hourly background job) {#case-8}

**Scenario:** 50 orders are in `Submitted to 1Click` status. The system syncs tracking every hour automatically.

**What the job does:**
1. Finds all Lyfe Orders with `status = "Submitted to 1Click"`
2. For each order: GET tracking from 1Click using `oneclick_po` + `us_warehouse_id`
3. Updates:
   - `oneclick_tracking_number`
   - `oneclick_status` (e.g., "Shipped", "In Transit", "Delivered")
   - Carrier doctype linked if a new carrier string is returned (creates new Carrier doc if not found)
4. If tracking not yet available: silently skips until next hour

**Example — order state before and after sync:**

```
Before sync (just submitted):
  oneclick_tracking_number = (empty)
  oneclick_status          = (empty)

After sync (carrier has picked up):
  oneclick_tracking_number = 1Z999AA10123456784
  oneclick_status          = In Transit

After final sync (delivered):
  oneclick_status          = Delivered
```

**Scheduler configuration** (runs hourly):
```
lh.lyfe_hardware.integrations.oneclick_api.sync_tracking_for_submitted_orders
```

---

### Case 9 — Fee items excluded from stock check {#case-9}

**Scenario:** An order includes a "Customs Fee" line item alongside physical products.

**What happens:**
- Items named exactly `"Custom Fee"` or `"Customs Fee"` are detected by `_is_fee_item()`
- They are stamped `fulfillment_source = "Excluded"` and skipped in the stock check
- Routing is decided based on physical items only
- Fee items are included in the order payload sent to 1Click (for billing purposes)

**Example:**

```
LYF-SH-2025-0700
  Items:
    Laptop Stand (SKU: LS-BLK) → fulfillment_source: [US Warehouse] 🟢
    Customs Fee                 → fulfillment_source: [Excluded]     ⚪
```

The Customs Fee line is passed through to 1Click but never used to determine warehouse routing.

---

## Transfer Orders (Two-Leg Tracking)

Transfer Orders track the **Factory → US Warehouse** leg of two-leg shipments. They are the record that warehouse staff update when goods arrive.

### Transfer Order Fields

| Field | Description |
|-------|-------------|
| `lyfe_order` | Link to the parent Lyfe Order |
| `status` | `Draft` → `Shipped` → `Received` |
| `ship_date` | Date factory dispatched the goods |
| `expected_arrival_us` | Estimated arrival at US warehouse |
| `leg1_tracking` | Tracking number for India → US leg |
| `carrier_leg1` | Carrier used for this leg |
| `carton_count` | Number of cartons |
| `notes` | Free text notes |
| `items` | Line items in transfer (child table) |

### Lifecycle

```
Transfer Order created (Draft)
  │
  ├── Factory ships → status: Shipped, leg1_tracking filled
  │
  └── US warehouse receives → status: Received
        └── 1Click fulfillment now triggered for Leg 2
```

---

## Status Reference

### Lyfe Order statuses added by this integration

| Status | When set | Meaning |
|--------|---------|---------|
| `Submitted to 1Click` | After successful API submission | 1Click has the order; awaiting tracking |
| `1Click Error` | After API call fails | Check `oneclick_error` field for details |
| `Pending India Dispatch` | Mixed split — factory child | Factory team to dispatch this portion |

### Full status flow with 1Click

```
New
 └─▶ [run_oneclick_fulfillment() runs]
       ├─▶ Submitted to 1Click  ──▶  [tracking synced hourly]  ──▶  Completed
       ├─▶ Pending India Dispatch  (factory child in split)
       └─▶ 1Click Error  (API failed — requires manual retry)
```

---

## Field Reference

### Per-order fields (1Click tab on Lyfe Order)

| Field | Read-only? | Description |
|-------|-----------|-------------|
| `oneclick_order_id` | Yes (after submit) | Order ID returned by 1Click |
| `oneclick_po` | No | PO number sent to 1Click (defaults to order name) |
| `oneclick_tracking_number` | No | Tracking number synced from 1Click |
| `oneclick_status` | No | Status string from 1Click (e.g., Shipped) |
| `oneclick_error` | No | Error traceback (last 1000 chars) |
| `oneclick_raw_response` | No | Full raw JSON response from 1Click |
| `routing_outcome` | No | US_FULL / INDIA_DIRECT_DROPSHIP / etc. |
| `route_plan` | No | Auto / Force US / Force India |
| `two_leg_required` | No | 1 = needs Transfer Order leg |
| `warehouse` | No | US Warehouse - LH or Factory - LH |

### Per-item fields (order_items child table)

| Field | Description |
|-------|-------------|
| `sku` | SKU from ShipStation |
| `fulfillment_sku` | Alternative SKU used in 1Click payload |
| `erp_item` | Link to ERP Item doctype |
| `fulfillment_source` | US Warehouse / Factory / Excluded |
| `fulfillment_source_display` | HTML badge rendered in item row |
| `adjustment` | Check = 1 → item excluded from stock check |

### SKU resolution priority (used when building 1Click payload)

```
1. fulfillment_sku  (if set)
2. sku              (if set)
3. erp_item         (if set)
4. item_name        (fallback)
```

---

## API & Webhook Reference

### ShipStation Webhook

| Property | Value |
|----------|-------|
| Endpoint | `/api/method/lh.lyfe_hardware.integrations.shipstation_webhook.receive` |
| Method | `POST` |
| Auth | HMAC-SHA256 via `X-Shipstation-Hmac-Sha256` header |
| Trigger | Every new/updated order in ShipStation |
| Effect | Creates/updates Lyfe Order, enqueues fulfillment job |

**Sample webhook payload (abbreviated):**
```json
{
  "orderNumber": "SS-10042",
  "customerEmail": "john@example.com",
  "orderDate": "2025-08-15T10:00:00",
  "orderTotal": 89.99,
  "shipTo": {
    "name": "John Smith",
    "street1": "123 Main St",
    "city": "Austin",
    "state": "TX",
    "postalCode": "78701",
    "country": "US"
  },
  "items": [
    {"name": "Laptop Stand (SKU: LS-BLK)", "quantity": 1, "unitPrice": 49.99},
    {"name": "USB Hub (SKU: UH-SLV)", "quantity": 2, "unitPrice": 19.99}
  ]
}
```

### 1Click API Calls (internal — not exposed to end users)

#### Inventory Check

```
POST <api_base_url>/api/v1/inventory
Headers:
  Token: <api_key>
  Call-Type: Inventory
  Content-Type: application/json

Body:
{
  "items": [
    {"sku": "LS-BLK"},
    {"sku": "UH-SLV"}
  ],
  "warehouseID": 12
}

Response (example):
[
  {"sku": "LS-BLK", "available": 15},
  {"sku": "UH-SLV", "available": 0}
]
```

#### Create Order

```
POST <api_base_url>/api/v2/orders
Headers:
  Token: <api_key>
  Call-Type: CreateOrder
  Content-Type: application/json

Body:
{
  "orders": [{
    "ponumber": "LYF-SH-2025-0200",
    "recipient": "John Smith",
    "address1": "123 Main St",
    "city": "Austin",
    "state": "TX",
    "zipcode": "78701",
    "country": "US",
    "phone": "5125551234",
    "email": "john@example.com",
    "warehouseID": 12,
    "carrier": "UPS",
    "service": "GROUND",
    "items": [
      {"lineID": "1", "sku": "LS-BLK", "description": "Laptop Stand", "qty": 2}
    ]
  }]
}

Response (example):
{
  "orderID": "OCK-88231",
  "ponumber": "LYF-SH-2025-0200",
  "status": "Received"
}
```

#### Get Tracking

```
GET <api_base_url>/api/v2/orders
Headers:
  Token: <api_key>
  Call-Type: Tracking
  Content-Type: application/json

Body:
{
  "orders": [{"po": "LYF-SH-2025-0200"}],
  "warehouseID": 12
}

Response (example):
{
  "ponumber": "LYF-SH-2025-0200",
  "trackingNumber": "1Z999AA10123456784",
  "carrier": "UPS",
  "status": "Shipped"
}
```

---

## Quick Reference Card

### Routing decision at a glance

| Stock situation | `route_plan` | Result |
|----------------|-------------|--------|
| All in US | Auto | Single order → 1Click US |
| All out of US | Auto | Single order → Factory |
| Mixed | Auto | Auto-split → US child + Factory child |
| Any | Force US | Single order → 1Click US (no stock check) |
| Any | Force India | Single order → Factory (no stock check) |

### Status meanings

| Status | Action needed |
|--------|--------------|
| `New` | Waiting for fulfillment job to run |
| `Submitted to 1Click` | No action — tracking will sync hourly |
| `Pending India Dispatch` | Factory team to dispatch |
| `1Click Error` | Check `oneclick_error`, fix and retry |
| `Split` | Parent archived — see child orders |

### Common issues

| Problem | Likely cause | Fix |
|---------|-------------|-----|
| Status stuck at `New` | Background job failed silently | Check Frappe background job queue |
| Status `1Click Error` | API down / bad credentials | Check `oneclick_error` field, verify Oneclick Settings |
| Wrong warehouse routed | Stale inventory cache | Set `route_plan = Force US/India` or wait for next sync |
| Tracking not appearing | 1Click not yet shipped | Wait for next hourly sync — expected 1-2 hours after submission |
| Fee item causing routing issue | Item name mismatch | Rename to exactly `"Custom Fee"` or `"Customs Fee"` |
| Split not created for mixed order | Bug in stock check response | Check `fulfillment_source` badges on each item row |
