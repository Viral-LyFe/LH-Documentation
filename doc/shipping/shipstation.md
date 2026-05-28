# ShipStation Integration — Overview & Use Cases

## What Does It Do?

ShipStation is the primary order intake source. The integration:
1. Syncs new ShipStation orders into `ShipStation Orders` every 30 minutes
2. Automatically creates a `Lyfe Order` for each new ShipStation Order
3. Pushes status updates back to ShipStation when orders ship
4. Sends failure notifications to configured users when push fails

Controller: `lyfe_hardware/doctype/shipstation_orders/shipstation_orders.py`  
Settings: `ShipStation Settings` (Single DocType)  
Sync runner: `lyfe_hardware/doctype/shipstation_settings/shipstation_settings.py`

---

## Data Flow

```
ShipStation API
    ↓  (every 30 min — run_sync_from_settings)
ShipStation Orders (doctype)
    ↓  (after_insert)
Lyfe Order (doctype)
    ↓  (when shipped — sync_order_status_to_shipstation)
ShipStation API ← status update pushed back
```

---

## Key Fields on ShipStation Orders

| Field | Purpose |
|---|---|
| `shipstation_order_id` | Unique ID from ShipStation (used as name) |
| `shipstation_order_number` | Human-readable order number |
| `order_status` | Status from ShipStation (awaiting_shipment, shipped, etc.) |
| `order_items` | Child table — each line item with SKU, qty, price |
| `ship_to_*` | Shipping address fields |
| `carrier_code`, `service_code` | Carrier and service to use |
| `is_custom_order` | Checkbox — set if order contains custom-manufactured items |
| `item_list` | Text field — ERP item codes resolved from SKUs |
| `error_log` | Code field — errors from failed pushes |

---

## Sync Process (every 30 minutes)

`run_sync_from_settings()` in `shipstation_settings.py`:
1. Reads `last_sync_timestamp` from settings
2. Calls ShipStation API `GET /orders` with `modifyDateStart` filter
3. For each returned order, upserts `ShipStation Orders` doctype
4. Updates `last_sync_timestamp`
5. `after_insert` hook fires `create_lyfe_order()` for brand-new orders

### Backfilling ERP Items

`backfill_erp_items_from_item_name()` runs during sync to map `order_items[].sku` → `erp_item` (Link to Item doctype). Uses a lookup by item name or `item_code` field.

---

## Pushing Status Back to ShipStation

`sync_order_status_to_shipstation()` is called when a Lyfe Order is marked shipped:
- Calls ShipStation `POST /orders/markasshipped`
- Payload: `{ orderId, trackingNumber, carrierCode, notifyCustomer: true }`
- On failure: logs to `error_log` field, sends email notification via `_notify_shipstation_order_failure()`

---

## Failure Notifications

Recipients are configured in `Access Settings.shipstation_failure_alert_users` (child table of users).

```python
# _get_failure_alert_recipients() in shipstation_orders.py
setting = frappe.get_single("Access Settings")
return [row.user for row in setting.shipstation_failure_alert_users]
```

---

## Use Cases

### Use Case 1: Manually trigger a ShipStation sync

```python
import frappe
from lh.lyfe_hardware.doctype.shipstation_settings.shipstation_settings import run_sync_from_settings

# In bench console
frappe.init(site="lyfe.local.local")
frappe.connect()
frappe.set_user("Administrator")
run_sync_from_settings()
frappe.db.commit()
```

Or from bench CLI:
```bash
bench --site lyfe.local.local execute lh.lyfe_hardware.doctype.shipstation_settings.shipstation_settings.run_sync_from_settings
```

### Use Case 2: Find ShipStation orders that failed to create a Lyfe Order

```python
import frappe

# ShipStation Orders without a linked Lyfe Order
orphans = frappe.db.sql("""
    SELECT sso.name, sso.shipstation_order_number, sso.order_date
    FROM `tabShipStation Orders` sso
    LEFT JOIN `tabLyfe Order` lo ON lo.shipstation_order = sso.name
    WHERE lo.name IS NULL
    AND sso.docstatus = 0
    ORDER BY sso.order_date DESC
    LIMIT 50
""", as_dict=True)

for o in orphans:
    print(o.name, o.shipstation_order_number)
```

### Use Case 3: Push a tracking number back to ShipStation

```python
import frappe

ss_order = frappe.get_doc("ShipStation Orders", "SS-ORDER-12345")
ss_order.sync_order_status_to_shipstation(
    tracking_number="1Z999AA10123456784",
    carrier_code="ups",
)
```

### Use Case 4: List all unshipped custom orders

```python
import frappe

orders = frappe.get_all("ShipStation Orders",
    filters={
        "is_custom_order": 1,
        "order_status": "awaiting_shipment",
    },
    fields=["name", "shipstation_order_number", "order_date", "ship_to_name"],
    order_by="order_date asc",
)
```

### Use Case 5: Re-create a Lyfe Order for a ShipStation Order that failed

```python
import frappe

ss_order = frappe.get_doc("ShipStation Orders", "SS-ORDER-12345")
lyfe_order_name = ss_order.create_lyfe_order()
print("Created:", lyfe_order_name)
frappe.db.commit()
```

### Use Case 6: Client script — add a "Push to ShipStation" button

```js
frappe.ui.form.on("ShipStation Orders", {
    refresh(frm) {
        frm.add_custom_button("Push to ShipStation", () => {
            frappe.confirm("Push this order status to ShipStation?", () => {
                frappe.call({
                    method: "lh.lyfe_hardware.doctype.shipstation_orders.shipstation_orders.push_to_shipstation",
                    args: { name: frm.doc.name },
                    callback: r => {
                        frappe.show_alert({ message: "Pushed successfully", indicator: "green" });
                        frm.reload_doc();
                    },
                });
            });
        });
    },
});
```

---

## Configuration Reference

All in `ShipStation Settings` (Single DocType):

| Field | Purpose |
|---|---|
| `is_active` | Enable/disable auto-sync |
| `ss_api_key` | ShipStation API key |
| `ss_api_secret` | ShipStation API secret |
| `ss_store_id` | ShipStation store ID (filters which store's orders to sync) |
| `last_sync_timestamp` | Auto-updated after each sync run |
| `us_warehouse_*` | US warehouse address for outbound labels |
| `factory_*` | Factory address for outbound labels |
| `enable_shipengine_tracking` | Use ShipEngine for tracking events |
| `enable_paccurate` | Use Paccurate for box optimisation |
