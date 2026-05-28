# Gate Pass — Overview, Lifecycle & Use Cases

## What is a Gate Pass?

A Gate Pass groups one or more Lyfe Orders for a single physical dispatch. When submitted, it:
1. Validates that all orders have stock issued (Material Issue for Order submitted)
2. Creates Draft Sales Invoices for each order
3. Moves orders to "Awaiting Tracking"
4. Syncs box dimensions to the Box Catalogue

Controller: `lyfe_hardware/doctype/gate_pass/gate_pass.py`

---

## Fields

| Field | Purpose |
|---|---|
| `date` | Dispatch date |
| `gate_pass_items` | Child table — one row per box/shipment |
| `total_items` | Auto-calculated total AWB count |
| `total_boxes` | Auto-calculated total boxes |
| `total_weight` | Auto-calculated total weight (kg) |
| `address` | Carrier pickup address |

### Gate Pass Item fields

| Field | Purpose |
|---|---|
| `order_id` | Link to Lyfe Order |
| `shipper_name` | Link to Export Partner (consignor) |
| `consignee_name` | Recipient name |
| `destination` | Country/city |
| `weight` | Weight in kg (string, e.g. "2.5") |
| `no_of_box` | Number of boxes in this shipment |
| `type` | Shipment type (CSB-V, CSB-IV, etc.) |
| `by_carrier` | Link to Carrier |
| `size` | Box dimensions as "LxWxH" string (e.g. "30x20x10") |
| `ddp` | DDP terms (select) |
| `signature_required` | Whether signature is required |
| `box_identity` | Auto-set to "Box N of Total" (first row only for multi-box) |
| `bypass_material_issue_check` | Skip MIFO validation for this row |

---

## Submission Lifecycle

### before_submit

**`_validate_items_before_submit(doc)`**
Checks every Gate Pass Item row has all required fields:
- weight, type, by_carrier, size, ddp, signature_required

Throws a formatted HTML error list if any are missing.

**`_validate_material_issue_submitted(doc)`**
For each order linked in Gate Pass Items (where `bypass_material_issue_check` is false):
- Checks that at least one submitted Material Issue for Order exists with a child row matching `lyfe_order = order_id`
- Uses child-table JOIN (never `mifo.for_lyfe_order`)

### on_submit

1. `_create_sales_invoices()` — creates one Draft Sales Invoice per order
   - Reads item rows from `Lyfe Order.cj_shipment_items`
   - Sets `item_code`, `qty`, `rate`, `custom_gst_hsn_code`, `hts_code`
2. `update_order_status()` — sets each linked order's `status` → "Awaiting Tracking" and `gate_pass_submitted_date` → today

### After submit (doc_event)

`shipping/box_catalogue_sync.py` → `sync_from_gate_pass_doc(doc)`
- Parses `size` field (e.g. "30x20x10", "30 x 20 x 10 cm") via regex
- Normalises by sorting dimensions descending (L ≥ W ≥ H)
- Upserts a Box Catalogue record with `source_gate_pass = gate_pass_name`

---

## Use Cases

### Use Case 1: Get all orders in a Gate Pass

```python
import frappe

def get_gate_pass_orders(gate_pass_name):
    return frappe.db.sql("""
        SELECT gpi.order_id, gpi.weight, gpi.no_of_box, gpi.destination
        FROM `tabGate Pass Item` gpi
        WHERE gpi.parent = %s AND gpi.order_id IS NOT NULL
    """, (gate_pass_name,), as_dict=True)

orders = get_gate_pass_orders("GPASS-2026-00042")
```

### Use Case 2: Find which Gate Pass an order belongs to

```python
import frappe

def find_gate_pass_for_order(order_name):
    result = frappe.db.sql("""
        SELECT gpi.parent as gate_pass, gp.date, gp.docstatus
        FROM `tabGate Pass Item` gpi
        JOIN `tabGate Pass` gp ON gp.name = gpi.parent
        WHERE gpi.order_id = %s
        ORDER BY gp.date DESC
        LIMIT 1
    """, (order_name,), as_dict=True)
    return result[0] if result else None
```

### Use Case 3: Check if a Gate Pass can be submitted (all MIFOs present)

```python
import frappe

def check_gate_pass_ready(gate_pass_name):
    doc = frappe.get_doc("Gate Pass", gate_pass_name)
    missing = []
    for row in doc.gate_pass_items:
        if row.bypass_material_issue_check or not row.order_id:
            continue
        has_mifo = frappe.db.sql("""
            SELECT 1
            FROM `tabMaterial Issue Order Item` item
            JOIN `tabMaterial Issue for Order` mifo ON mifo.name = item.parent
            WHERE item.lyfe_order = %s AND mifo.docstatus = 1
            LIMIT 1
        """, (row.order_id,))
        if not has_mifo:
            missing.append(row.order_id)
    return missing  # empty list = ready to submit
```

### Use Case 4: List today's Gate Passes with total weight

```python
import frappe
from frappe.utils import today

passes = frappe.get_all("Gate Pass",
    filters={"date": today(), "docstatus": 1},
    fields=["name", "total_items", "total_boxes", "total_weight"],
)
for gp in passes:
    print(f"{gp.name} — {gp.total_boxes} boxes, {gp.total_weight} kg")
```

### Use Case 5: Client script — auto-calculate total weight from child rows

```js
frappe.ui.form.on("Gate Pass Item", {
    weight(frm, cdt, cdn) {
        let total = 0;
        frm.doc.gate_pass_items.forEach(row => {
            total += parseFloat(row.weight) || 0;
        });
        frm.set_value("total_weight", total.toFixed(2));
    },
});
```

### Use Case 6: Add a custom button to download Gate Pass summary

```js
frappe.ui.form.on("Gate Pass", {
    refresh(frm) {
        if (frm.doc.docstatus === 1) {
            frm.add_custom_button("Download Summary", () => {
                window.open(
                    frappe.urllib.get_full_url(
                        `/api/method/lh.lyfe_hardware.doctype.gate_pass.gate_pass.get_summary?name=${frm.doc.name}`
                    )
                );
            }, "Export");
        }
    },
});
```

---

## Box Catalogue Sync

Every time a Gate Pass is submitted or updated after submit, `box_catalogue_sync.py` runs:

1. Reads the `size` field from each Gate Pass Item (e.g. `"40x30x20"`)
2. Parses via regex: `r'(\d+(?:\.\d+)?)\s*[xX×]\s*(\d+(?:\.\d+)?)\s*[xX×]\s*(\d+(?:\.\d+)?)'`
3. Sorts dimensions descending so `L >= W >= H` always
4. Generates a canonical name: `"40.0 × 30.0 × 20.0 cm"`
5. Inserts or updates the Box Catalogue record with `source_gate_pass`

This means the Box Catalogue is automatically populated from real shipments — no manual entry needed.
