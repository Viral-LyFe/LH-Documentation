# Shipping Rates & Box Selection — Overview & Use Cases

## Architecture

The shipping calculation system is in `lyfe_hardware/shipping/`. It is stateless — no writes happen during rate calculation. The user sees all options and confirms one.

```
shipping_calculator.py      ← main entry point (whitelisted)
├── box_selection.py        ← 3D bin packing, box size selection
├── bom_resolver.py         ← BOM expansion for item dimensions
├── bom_box_calculator.py   ← dimension aggregation from BOM
├── shipstation_rates_api.py ← ShipStation carrier rates
├── shipengine_rates_api.py  ← ShipEngine multi-carrier rates
├── paccurate_api.py        ← Paccurate box optimisation
└── threedbin_api.py        ← 3DBinPacking optimisation
```

---

## Flow: calculate_shipping_rate()

Called from the Quotation form when CS clicks "Calculate Shipping".

1. **Resolve items** — reads order items and expands BOMs one level deep via `bom_resolver.py`
2. **Get dimensions** — reads `length_cm`, `width_cm`, `height_cm`, `weight_kg` from each item's BOM/Item Master
3. **Build packing strategies** — four variants:
   - All items in one combined box
   - Each item as a separate package
   - Custom items combined + standard items separate
   - Custom items separate + standard items combined
4. **Call carrier APIs** — ShipStation or ShipEngine for each strategy
5. **Return all strategies** — frontend displays them; user picks one

No data is saved until `apply_shipping_rate()` is called (on user confirmation).

---

## Box Selection (box_selection.py)

Uses `py3dbp` (3D bin packing library) with a hardcoded list of candidate bins sorted by volume.

```python
def get_best_box(items):
    """
    items: list of {"length": float, "width": float, "height": float, "weight": float}
    Returns the smallest Box Catalogue entry that fits all items.
    """
```

Fallback: if no single box fits, returns `get_separate_packages()` — one package per item.

The active box sizes come from the Box Catalogue (filtered `is_active = 1`, sorted by volume ascending).

---

## Use Cases

### Use Case 1: Get a shipping rate for a quotation (Python)

```python
import frappe
from lh.lyfe_hardware.shipping.shipping_calculator import calculate_shipping_rate

result = calculate_shipping_rate(quotation_name="QTN-2026-00123")
# Returns dict of strategies: {"combined": {...}, "separate": {...}, ...}
for strategy, rate_info in result.items():
    print(strategy, rate_info.get("total_rate"), rate_info.get("currency"))
```

### Use Case 2: Apply a chosen rate to the quotation

```python
from lh.lyfe_hardware.shipping.shipping_calculator import apply_shipping_rate

apply_shipping_rate(
    quotation_name="QTN-2026-00123",
    strategy="combined",
    carrier_code="fedex",
    service_code="FEDEX_INTERNATIONAL_PRIORITY",
    rate=45.00,
    currency="USD",
)
```

### Use Case 3: Find the best box for a set of items

```python
from lh.lyfe_hardware.shipping.box_selection import get_best_box

items = [
    {"length": 30, "width": 20, "height": 10, "weight": 1.5},
    {"length": 15, "width": 10, "height": 5, "weight": 0.3},
]
best_box = get_best_box(items)
if best_box:
    print(f"Use: {best_box.box_name} ({best_box.length_cm} × {best_box.width_cm} × {best_box.height_cm} cm)")
else:
    print("No single box fits — use separate packages")
```

### Use Case 4: Get ShipStation carrier rates directly

```python
from lh.lyfe_hardware.shipping.shipstation_rates_api import get_rates

rates = get_rates(
    carrier_code="stamps_com",
    from_postal_code="110001",
    from_country="IN",
    to_postal_code="10001",
    to_country="US",
    weight_kg=2.5,
    length_cm=30,
    width_cm=20,
    height_cm=10,
)
for rate in rates:
    print(rate["serviceCode"], rate["shipmentCost"])
```

### Use Case 5: Get carrier options for a frontend dropdown (whitelisted)

```js
// In a client script — populate a carrier service selector
frappe.call({
    method: "lh.lyfe_hardware.shipping.shipstation_rates_api.get_carrier_options",
    args: { carrier_code: "fedex" },
    callback: r => {
        // r.message = [{ value: "FEDEX_INTERNATIONAL_PRIORITY", label: "FedEx Int'l Priority" }, ...]
        frm.set_df_property("service_code", "options", r.message.map(o => o.value).join("\n"));
    },
});
```

### Use Case 6: Expand a BOM to get dimensions

```python
from lh.lyfe_hardware.shipping.bom_resolver import resolve_bom_dimensions

dimensions = resolve_bom_dimensions(item_code="CUSTOM-LAMP-001", qty=2)
# Returns list of {"item_code", "length_cm", "width_cm", "height_cm", "weight_kg", "qty"}
for d in dimensions:
    print(d["item_code"], d["length_cm"], "×", d["width_cm"], "×", d["height_cm"])
```

---

## Configuration

Shipping features are toggled in `ShipStation Settings` (Single DocType):

| Field | Controls |
|---|---|
| `enable_shipengine_tracking` | Use ShipEngine for tracking |
| `enable_paccurate` | Use Paccurate for box optimisation |
| `enable_3dbinpacking` | Use 3DBinPacking service |
| `us_warehouse_auto_detect` | Auto-detect if order needs US warehouse routing |
| `us_warehouse` | Link to the US Warehouse record |

FedEx API credentials are in `FedEx Settings` (Single DocType).  
CJ Logistics credentials are in `CJ Settings` (Single DocType).

---

## Adding a New Carrier API

1. Create `shipping/<carrier>_api.py`
2. Implement `get_rates(...)` returning a list of `{"serviceCode", "shipmentCost", "currency"}`
3. Implement `create_label(...)` returning `{"tracking_number", "label_url"}`
4. Import and call from `shipping_calculator.py` based on a setting toggle
5. Add the toggle field to `ShipStation Settings` doctype
