# Lyfe Order — Use Cases & Examples

---

## Use Case 1: Get all orders currently in Factory Assignment

```python
import frappe

orders = frappe.get_all("Lyfe Order",
    filters={"status": "Factory Assignment"},
    fields=["name", "customer", "order_date", "warehouse"],
    order_by="order_date asc",
)
for o in orders:
    print(o.name, o.customer)
```

---

## Use Case 2: Get all shipped items for an order

```python
import frappe

def get_issued_items(order_name):
    """Returns net issued qty per item_code (submitted MIFOs minus returns)."""
    return frappe.db.sql("""
        SELECT item.item_code, item.item_name, SUM(item.issue_qty) as qty
        FROM `tabMaterial Issue Order Item` item
        JOIN `tabMaterial Issue for Order` mifo ON mifo.name = item.parent
        WHERE item.lyfe_order = %s AND mifo.docstatus = 1
        GROUP BY item.item_code
    """, (order_name,), as_dict=True)

items = get_issued_items("LO-001234")
for item in items:
    print(item.item_code, item.qty)
```

---

## Use Case 3: Move an order to On Hold (via Python)

```python
import frappe

doc = frappe.get_doc("Lyfe Order", "LO-001234")
doc.status = "On Hold"
doc.hold_start_date = frappe.utils.now_datetime()
doc.hold_resume_date = frappe.utils.add_days(frappe.utils.nowdate(), 7)
doc.save()
frappe.db.commit()
```

---

## Use Case 4: Check if an order has a submitted Material Issue

```python
import frappe

def has_submitted_mifo(order_name):
    result = frappe.db.sql("""
        SELECT 1
        FROM `tabMaterial Issue Order Item` item
        JOIN `tabMaterial Issue for Order` mifo ON mifo.name = item.parent
        WHERE item.lyfe_order = %s AND mifo.docstatus = 1
        LIMIT 1
    """, (order_name,))
    return bool(result)

print(has_submitted_mifo("LO-001234"))  # True or False
```

---

## Use Case 5: Find orders with tracking number but not marked Shipped

```python
import frappe

orders = frappe.get_all("Lyfe Order",
    filters={
        "tracking_number": ["!=", ""],
        "status": ["not in", ["Shipped", "Completed", "Delivered", "Cancelled"]],
    },
    fields=["name", "status", "tracking_number", "customer"],
)
```

---

## Use Case 6: Get the cj_shipment_items for an order (from Python)

```python
import frappe

order = frappe.get_doc("Lyfe Order", "LO-001234")
for row in order.cj_shipment_items:
    print(row.item_code, row.item_name, row.qty, row.rate_usd)
```

---

## Use Case 7: Find all orders linked to a Gate Pass

```python
import frappe

def get_orders_for_gate_pass(gate_pass_name):
    return frappe.db.sql("""
        SELECT gpi.order_id
        FROM `tabGate Pass Item` gpi
        WHERE gpi.parent = %s AND gpi.order_id IS NOT NULL
    """, (gate_pass_name,), as_dict=True)

orders = get_orders_for_gate_pass("GPASS-2026-00042")
```

---

## Use Case 8: Log a custom action to the order log (Code field)

```python
import frappe
from frappe.utils import now_datetime

def append_to_log(order_name, message):
    doc = frappe.get_doc("Lyfe Order", order_name)
    timestamp = now_datetime().strftime("%Y-%m-%d %H:%M")
    existing = doc.log or ""
    doc.log = f"[{timestamp}] {message}\n{existing}"
    doc.save(ignore_permissions=True)
    frappe.db.commit()

append_to_log("LO-001234", "Tracking updated by scheduler")
```

---

## Use Case 9: Trigger a status change from a background task

```python
import frappe

def mark_as_delivered(order_name):
    doc = frappe.get_doc("Lyfe Order", order_name)
    if doc.status not in ("Shipped", "Awaiting Tracking"):
        return
    doc.status = "Completed"
    doc.delivered = frappe.utils.today()
    doc.save(ignore_permissions=True)
    frappe.db.commit()
```

---

## Use Case 10: Count orders by status (for a dashboard)

```python
import frappe

counts = frappe.db.sql("""
    SELECT status, COUNT(*) as count
    FROM `tabLyfe Order`
    WHERE status NOT IN ('Cancelled', 'Completed', 'Delivered')
    GROUP BY status
    ORDER BY count DESC
""", as_dict=True)

for row in counts:
    print(f"{row.status}: {row.count}")
```

---

## Use Case 11: Client script — auto-fill CJ consignor fields when Export Partner is selected

```js
frappe.ui.form.on("Lyfe Order", {
    cj_export_partner(frm) {
        if (!frm.doc.cj_export_partner) return;
        frappe.db.get_doc("Export Partner", frm.doc.cj_export_partner).then(ep => {
            frm.set_value("cj_consignor_name", ep.export_partner_name);
            frm.set_value("cj_consignor_address1", ep.address_line_1);
            frm.set_value("cj_consignor_city", ep.city);
            frm.set_value("cj_consignor_state", ep.state);
            frm.set_value("cj_consignor_postcode", ep.pincode);
            frm.set_value("cj_consignor_phone", ep.phone);
        });
    },
});
```

---

## Use Case 12: API endpoint — get order summary (whitelisted Python)

```python
@frappe.whitelist()
def get_order_summary(order_name):
    doc = frappe.get_doc("Lyfe Order", order_name)
    return {
        "name": doc.name,
        "status": doc.status,
        "customer": doc.customer,
        "items": [
            {"sku": row.item_code, "qty": row.qty}
            for row in doc.cj_shipment_items
        ],
        "tracking": doc.tracking_number,
    }
```

Call from JS:
```js
frappe.call({
    method: "lh.lyfe_hardware.doctype.lyfe_order.lyfe_order.get_order_summary",
    args: { order_name: frm.doc.name },
    callback: r => console.log(r.message),
});
```
