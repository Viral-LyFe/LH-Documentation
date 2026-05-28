# 17Track Integration — Overview & Use Cases

## What Does It Do?

17Track is used to track outbound shipments. The integration:
1. Registers new tracking numbers with 17Track when an order is shipped
2. Polls tracking status hourly for all active shipments
3. Updates `Lyfe Order.track_status` and `tracking_status_desc` with the latest event

Module: `lyfe_hardware/integrations/seventeentrack.py`  
Settings: `Seventeen Track Settings` (Single DocType)  
Scheduled tasks: `lyfe_hardware/doctype/lyfe_order/order_tracking.py`

---

## Configuration

`Seventeen Track Settings`:

| Field | Purpose |
|---|---|
| `enable_17track` | Master toggle — disables all tracking if unchecked |
| `api_key` | 17Track API key (password field) |
| `webhook_security_key` | HMAC key for incoming webhook verification |

Carrier codes are stored on the `Carrier` doctype:

| Field | Purpose |
|---|---|
| `seventeen_track_carrier_code` | Integer carrier code used by 17Track API |

---

## Scheduled Tracking

Two hourly tasks in `order_tracking.py`:

| Function | Orders targeted |
|---|---|
| `scheduled_track_ready_orders()` | India-shipped orders (tracking_number set, not `order_via_us_warehouse`) |
| `scheduled_track_ready_orders_us()` | US warehouse forwarded orders (tracking_number_us set) |

Each function:
1. Queries Lyfe Orders with tracking number but not yet delivered
2. Calls 17Track `POST /track/v2.1/gettrackinfo` with batch of tracking numbers
3. Updates `track_status` and `tracking_status_desc` on each order
4. Sets `delivered` date if status indicates delivery
5. Sets `is_17track_registered` = 1 after first successful registration

---

## Use Cases

### Use Case 1: Register a single tracking number with 17Track

```python
import frappe
from lh.lyfe_hardware.integrations.seventeentrack import register_tracking_numbers

register_tracking_numbers(
    tracking_numbers=["1Z999AA10123456784"],
    carrier_code=3011,  # UPS code in 17Track
)
```

### Use Case 2: Manually poll all active tracking

```bash
bench --site lyfe.local.local execute lh.lyfe_hardware.doctype.lyfe_order.order_tracking.scheduled_track_ready_orders
```

### Use Case 3: Find orders with stale tracking (no update in 3 days)

```python
import frappe
from frappe.utils import add_days, today

stale = frappe.db.sql("""
    SELECT name, tracking_number, track_status, customer
    FROM `tabLyfe Order`
    WHERE tracking_number != ''
    AND is_17track_registered = 1
    AND delivered IS NULL
    AND status NOT IN ('Cancelled', 'Completed')
    AND modified < %s
""", (add_days(today(), -3),), as_dict=True)

for o in stale:
    print(o.name, o.track_status)
```

### Use Case 4: Get latest tracking status for an order

```python
import frappe
from lh.lyfe_hardware.integrations.seventeentrack import get_tracking_info

order = frappe.get_doc("Lyfe Order", "LO-001234")
info = get_tracking_info(tracking_number=order.tracking_number)
print(info["status"])       # e.g. "In Transit"
print(info["description"])  # e.g. "Package arrived at Chicago facility"
```

### Use Case 5: Check which Carrier doctype maps to which 17Track code

```python
import frappe

carriers = frappe.get_all("Carrier",
    filters={"seventeen_track_carrier_code": ["!=", 0]},
    fields=["name", "carrier_code", "friendly_name", "seventeen_track_carrier_code"],
)
for c in carriers:
    print(f"{c.friendly_name}: 17Track code = {c.seventeen_track_carrier_code}")
```

### Use Case 6: Disable tracking for a specific order

```python
import frappe

# Setting is_17track_registered to 0 makes the scheduler skip it
doc = frappe.get_doc("Lyfe Order", "LO-001234")
doc.is_17track_registered = 0
doc.save(ignore_permissions=True)
frappe.db.commit()
```

---

## Webhook (Incoming)

When 17Track sends a status update webhook:
1. Signature is verified using `webhook_security_key`
2. Tracking number is matched to a Lyfe Order
3. `track_status` and `tracking_status_desc` are updated

The webhook endpoint is registered as a Frappe Page or API route (check `www/` or `api.py`).
