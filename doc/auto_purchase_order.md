# Auto Purchase Order Creation

## Overview

`lh/lyfe_hardware/tasks/auto_purchase_order.py` automatically creates ERPNext **Purchase Orders** for Lyfe Orders that have reached a dispatch/shipping stage but do not yet have a PO linked. It is designed to run as a **daily scheduled task**.

Each PO is raised against the **Export Partner** (`cj_export_partner`) on the Lyfe Order, which must exist as a **Supplier** in ERPNext under the same name.

---

## When a PO Is Created

The task scans all Lyfe Orders matching **all** of the following conditions:

| Condition | Value |
|---|---|
| `cj_export_partner` | Set (not empty) |
| `erp_purchase_order` | Not set (no existing PO) |
| `status` | One of: `Ready for Dispatch`, `Ready for Shipment`, `Dispatched`, `Shipped`, `Completed` |

---

## Item Resolution Logic

Items are resolved in two passes. The final quantity comes from Material Issues (most accurate); rates come from item masters.

### Pass 1 — ShipStation Order Items

Reads non-adjustment rows from the `order_items` child table of the Lyfe Order.

- If `erp_item` is already set → uses that item code directly.
- If `erp_item` is blank → tries to find an Item by `item_name` match; if not found, **auto-creates a placeholder Item** (non-stock, group: Products) and back-fills `erp_item` on the ShipStation row.
- **Rate:** For custom order rows (those with no `erp_item` before resolution), the Lyfe Order's `cost_of_goods` field is used as the rate. For standard rows, `_get_item_rate()` is called.

### Pass 2 — Submitted Material Issues (overrides Pass 1 qty)

Reads all submitted `Material Issue for Order` records linked to the order:

```
Net qty = SUM(Issue qty) − SUM(Return qty)   per item_code
```

Only rows with net qty > 0 are included. If an item from a Material Issue is not already in the map from Pass 1, it is added with its rate from `_get_item_rate()`.

> **Pass 2 takes priority for quantity.** If both passes resolve the same item, Pass 2's qty replaces Pass 1's qty.

### Rate Resolution (`_get_item_rate`)

1. Reads `cost_of_goods_sold` directly from the Item master.
2. If that is zero, finds the item's default active BOM and sums `child_item.cost_of_goods_sold × qty` across all BOM rows.
3. If any BOM child has zero COGS, returns 0 (to force manual review rather than submit with wrong rates).

---

## Purchase Order Creation

```
Supplier       = cj_export_partner  (must exist as a Supplier)
Company        = Global Default company
Schedule Date  = today()
Warehouse      = Factory - LH
custom_lyfe_order = Lyfe Order name   (custom field linking back)
```

After insert, `Lyfe Order.erp_purchase_order` is set to the new PO name immediately (before submit) to prevent duplicate creation in concurrent runs (row-level lock via `SELECT ... FOR UPDATE`).

### Auto-Submit Condition

| Condition | Outcome |
|---|---|
| All items have rate > 0 | PO is **submitted** automatically |
| Any item has rate = 0 | PO is saved as **Draft**; an Error Log entry is created listing the zero-rate items; PO must be reviewed and submitted manually |

---

## Scheduler Setup

> **The task is not yet wired into `hooks.py`.** Add the following to schedule it.

Open `lh/hooks.py` and add to the `scheduler_events` → `"cron"` block:

```python
"0 2 * * *": [  # 2 AM daily
    "lh.lyfe_hardware.tasks.auto_purchase_order.create_purchase_orders_for_pending_orders",
],
```

Then restart the scheduler:

```bash
bench restart
```

---

## Production Deployment Checklist

After deploying to production, verify the following before enabling the scheduler.

### 1. Export Partners are Suppliers

Every Export Partner that will appear on Lyfe Orders must exist as a **Supplier** in ERPNext with the **exact same name**.

To check:

```sql
SELECT lo.cj_export_partner
FROM `tabLyfe Order` lo
LEFT JOIN `tabSupplier` s ON s.name = lo.cj_export_partner
WHERE lo.cj_export_partner IS NOT NULL
  AND lo.cj_export_partner != ''
  AND s.name IS NULL
GROUP BY lo.cj_export_partner;
```

Any result means that Export Partner has no matching Supplier — create it before enabling the task.

### 2. Items Have `cost_of_goods_sold` Set

The PO will only auto-submit if every item has a rate. Check for items that will be used in POs but have zero COGS:

```sql
SELECT name, item_name, cost_of_goods_sold
FROM `tabItem`
WHERE is_sales_item = 1
  AND (cost_of_goods_sold IS NULL OR cost_of_goods_sold = 0)
  AND item_group = 'Products';
```

For BOM-based items, ensure every BOM child item also has `cost_of_goods_sold > 0`.

### 3. `custom_lyfe_order` Custom Field on Purchase Order

The task sets `po.custom_lyfe_order = lyfe_order_name`. Verify this custom field exists:

```bash
bench --site <your-site> execute frappe.db.exists --args '["Custom Field", "Purchase Order-custom_lyfe_order"]'
```

If it returns `None`, create it:

- DocType: `Purchase Order`
- Fieldname: `custom_lyfe_order`
- Field Type: Link → Lyfe Order

Then run `bench --site <your-site> migrate`.

### 4. Wire the Scheduler

Add the cron entry to `hooks.py` as shown in [Scheduler Setup](#scheduler-setup) above.

### 5. Test Before Going Live

Run the task manually in a bench console first:

```bash
bench --site <your-site> execute lh.lyfe_hardware.tasks.auto_purchase_order.create_purchase_orders_for_pending_orders
```

Check the output and review any newly created POs. Confirm:
- Correct supplier
- Correct items and quantities
- Correct rates (no zero-rate items if possible)
- `erp_purchase_order` is filled on the Lyfe Order

---

## Handling Draft POs (Zero-Rate Items)

When a PO is created as Draft due to zero-rate items:

1. Go to **Error Log** → filter title by `Auto PO` to find the entries
2. Open the Draft Purchase Order
3. Set the correct rate on each flagged item
4. Submit the PO manually
5. The `erp_purchase_order` field on the Lyfe Order is already set, so the order will not be picked up again by the scheduler

---

## Concurrency Safety

The task acquires a **row-level database lock** (`SELECT ... FOR UPDATE`) on each Lyfe Order before checking whether a PO exists. This prevents two scheduler workers from simultaneously creating duplicate POs for the same order. After the lock is acquired, the existing-PO check is re-run inside the locked transaction before proceeding.

---

## File Reference

| File | Purpose |
|---|---|
| `lh/lyfe_hardware/tasks/auto_purchase_order.py` | All task logic |
| `lh/hooks.py` | Scheduler registration (cron entry to be added) |

## Logs

The task writes to the Frappe logger channel `auto_purchase_order`. To view live:

```bash
tail -f /home/frappe/frappe-bench/logs/worker.log | grep auto_purchase_order
```

Completion summary is logged at INFO level:

```
auto_purchase_order run complete — created: 5, skipped: 2, failed: 0
```
