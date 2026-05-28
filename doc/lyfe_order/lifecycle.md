# Lyfe Order — Lifecycle & Data Flow

## What is a Lyfe Order?

`Lyfe Order` is the central record for every outbound shipment. It is created from ShipStation (via webhook sync), from a Shopify Order (via `ShopifyOrder.create_lyfe_order()`), or manually. It tracks the order from factory assignment through shipping and delivery.

---

## Status Flow

```
New
 └─► Factory Assignment
      └─► Ready for Dispatch         ← Gate Pass auto-created here
           └─► Awaiting Tracking     ← Gate Pass submitted; Sales Invoice created
                └─► Shipped          ← Tracking number entered
                     └─► Completed / Delivered
                     └─► Return Initiated
                          └─► Return Received → Refunded / Reshipped
```

Additionally at any stage:
- `On Hold` — `hold_start_date` set; `hold_resume_date` expected
- `Cancelled`
- `Merged` — merged into another Lyfe Order

---

## Key Fields

| Field | Purpose |
|---|---|
| `status` | Current workflow state (select) |
| `workflow_status` | Internal workflow field used by Frappe workflow engine |
| `order_source` | ShipStation / Shopify / Manual |
| `shipstation_order` | Link to ShipStation Orders doctype |
| `erp_sales_order` | Link to ERPNext Sales Order |
| `erp_sales_invoice` | Link to ERPNext Sales Invoice |
| `warehouse` | Warehouse fulfilling this order |
| `cj_shipment_items` | Child table — what was physically issued (net qty after returns) |
| `order_items` | Child table — what the customer ordered (from ShipStation) |
| `factory_images` | Child table — photos attached by factory |
| `lyfe_order_item_map` | Child table — ERP item mapping per ordered SKU |
| `tracking_number` | Carrier tracking number (India → customer) |
| `tracking_number_us` | Tracking number India → US warehouse (for US warehouse orders) |

---

## How a Lyfe Order is Created

### From ShipStation
1. `ShipStation Settings.run_sync_from_settings()` runs every 30 min.
2. New ShipStation Orders are inserted.
3. `ShipStationOrders.after_insert()` calls `create_lyfe_order()`.
4. `LyfeOrder.autoname()` generates the name: source order ID + any split suffix.

### From Shopify
1. Shopify `orders/create` webhook → `handle_orders_create()` in `integrations/shopify.py`.
2. A `Shopify Order` (Sales Order subtype) is inserted.
3. `ShopifyOrder.after_insert()` calls `create_lyfe_order()`.

### Manual
Created directly in the Frappe desk. `autoname()` still runs to set the name.

---

## Material Issue → `cj_shipment_items` (single source of truth)

`cj_shipment_items` on Lyfe Order is the **only** record of what was physically issued.

**Write path:**
1. Factory creates a `Material Issue for Order` (MIFO) for the order.
2. On MIFO submit: `_rebuild_cj_shipment_items(order_id)` is called.
3. It queries **all submitted MIFOs** for this order, sums qty by `item_code`, subtracts returns, and writes the result back to `cj_shipment_items`.

**Read path (downstream):**
- Gate Pass → Sales Invoice creation reads `cj_shipment_items`
- Packing List reads `cj_shipment_items` (falls back to BOM expansion if empty)
- Lyfe Export Document reads `cj_shipment_items`

**Critical join pattern:**
```python
# Always join on child row's field, not MIFO header
SELECT item.lyfe_order, item.item_code, SUM(item.issue_qty) as qty
FROM `tabMaterial Issue Order Item` item
JOIN `tabMaterial Issue for Order` mifo ON mifo.name = item.parent
WHERE item.lyfe_order = %s AND mifo.docstatus = 1
GROUP BY item.item_code
```

---

## Gate Pass — What Happens on Submit

1. `GatePass.before_submit()` validates:
   - Each Gate Pass Item row has: weight, type, carrier, size, signature_required, DDP
   - Every linked Lyfe Order has a submitted MIFO (unless `bypass_material_issue_check` is checked)

2. `GatePass.on_submit()`:
   - `_create_sales_invoices()` — creates one Draft Sales Invoice per order using `cj_shipment_items`
   - `update_order_status()` — moves linked Lyfe Orders to `"Awaiting Tracking"`
   - `gate_pass_submitted_date` set on each order

3. `shipping/box_catalogue_sync.py` — parses the `size` field (e.g. `"30x20x10"`) from each Gate Pass Item, normalises dimensions, and upserts a Box Catalogue entry.

---

## Factory Images

Images are attached via `factory_images` child table (type: `Lyfe Order Images`).

- `first_photo_attachment_date` — auto-set when first image is attached (via `on_version_factory_first_action`)
- `second_photo_attachment_date` — set on the second distinct upload date
- `images_at_last_rejection` — count of images when CS last rejected; used to detect new images after a rejection

---

## Return / RTO Flow

When an order is returned:

1. `return_initiated_date` is set; `return_type` = "Customer Return" or "RTO"
2. `return_received_date` set when goods arrive back
3. SLA fields (`sla_return_received_by`, `sla_refund_by`, `sla_reship_by`) are deadlines
4. `sla_breached` is a flag if any deadline was missed
5. Each RTO cycle is archived into `rto_history` child table before overwriting the parent fields

---

## Hooks Wired on Lyfe Order

Registered in `hooks.py` under `"Lyfe Order"`:

| Event | Handler |
|---|---|
| `on_update` | `lh_project.automation.lyfe_order.on_doc_update` |
| `on_update` | `lh_project.sla.autoclose.close_engine.check_and_close` |

The controller itself (`lyfe_order.py`) handles `validate`, `on_update`, `after_insert` lifecycle hooks internally.

---

## Scheduled Tasks for Lyfe Orders

| Task | Schedule | What it does |
|---|---|---|
| `order_tracking.scheduled_track_ready_orders` | Hourly | Polls 17Track for updates on India-shipped orders |
| `order_tracking.scheduled_track_ready_orders_us` | Hourly | Polls 17Track for US warehouse forwarded orders |
