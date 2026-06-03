# `get_lyfe_orders` — API Reference

## Endpoint

```
GET /api/method/lh.api.get_lyfe_orders
```

Authentication required — standard Frappe session cookie **or** API key/secret header pair.

---

## How to Call

### Option 1 — Browser / Session Cookie (logged-in user)

```
GET https://<your-site>/api/method/lh.api.get_lyfe_orders?workflow_state=Shipped&page=1&page_size=20
```

### Option 2 — API Key + Secret (recommended for integrations)

Generate keys in **User → API Access** in Frappe desk, then pass them as a header:

```
Authorization: token <api_key>:<api_secret>
```

Example with `curl`:
```bash
curl -X GET \
  "https://<your-site>/api/method/lh.api.get_lyfe_orders?workflow_state=Shipped" \
  -H "Authorization: token abc123:xyz789"
```

Example with Python `requests`:
```python
import requests

response = requests.get(
    "https://<your-site>/api/method/lh.api.get_lyfe_orders",
    params={
        "workflow_state": "Shipped",
        "order_date_from": "2025-01-01",
        "order_date_to": "2025-12-31",
        "total_amount_gt": 1000,
        "page": 1,
        "page_size": 50,
    },
    headers={"Authorization": "token abc123:xyz789"},
)
data = response.json()
```

---

## Query Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `order_date_from` | date / datetime | Filter: `order_date` ≥ value (e.g. `2025-01-01`) |
| `order_date_to` | date / datetime | Filter: `order_date` ≤ value |
| `creation_from` | date / datetime | Filter: record creation date ≥ value |
| `creation_to` | date / datetime | Filter: record creation date ≤ value |
| `workflow_state` | string | Exact match on `workflow_state` (e.g. `Shipped`, `Completed`) |
| `customer` | string | Exact match on `customer` field |
| `warehouse` | string | Exact match on `warehouse` field |
| `total_amount_gt` | number | Filter: `total_amount` **>** value |
| `total_amount_lt` | number | Filter: `total_amount` **<** value |
| `page` | int | Page number, 1-based (default: `1`) |
| `page_size` | int | Records per page, max 200 (default: `20`) |

All parameters are optional. Omitting a parameter means no filter is applied for that field.

---

## Response

```json
{
  "message": {
    "data": [
      {
        "name": "LO-2025-001234",
        "internal_order_id": "INT-001",
        "erp_sales_order": "SAL-ORD-2025-00012",
        "ss_shipping_amount": 15.00,
        "ss_tax_amount": 0.00,
        "ss_discount_amount": 5.00,
        "total_amount": 250.00,
        "order_date": "2025-06-01 10:30:00",
        "workflow_state": "Shipped",
        "cost_of_goods": "180.00",
        "shipping_charges": 15.00,
        "custom_charges": 0.00,
        "additional_charges": 0.00,
        "warehouse": "Factory - LH",
        "customer": "John Doe",
        "creation": "2025-06-01 10:30:00",
        "modified": "2025-06-02 08:00:00",
        "order_items": [
          {
            "erp_item": "ITEM-CODE-001",
            "item_name": "Brass Cabinet Handle",
            "item_group": "Hardware",
            "unit_price": 120.00
          }
        ]
      }
    ],
    "total": 1,
    "page": 1,
    "page_size": 20,
    "total_pages": 1
  }
}
```

### Response Fields

#### Order-level fields

| Field | Description |
|-------|-------------|
| `name` | Lyfe Order document name (e.g. `LO-2025-001234`) |
| `internal_order_id` | Internal order reference ID |
| `erp_sales_order` | Linked ERP Sales Order name |
| `total_amount` | **Total order amount** — the final amount the customer paid |
| `ss_shipping_amount` | ShipStation shipping amount — **informational only, already included in `total_amount`** |
| `ss_tax_amount` | ShipStation tax amount — **informational only, already included in `total_amount`** |
| `ss_discount_amount` | ShipStation discount amount — **informational only, already included in `total_amount`** |
| `order_date` | Date/time the order was placed |
| `workflow_state` | Current workflow status (e.g. `New`, `Factory Assignment`, `Shipped`, `Completed`) |
| `cost_of_goods` | Cost of goods for the order |
| `shipping_charges` | Shipping charges charged to the customer |
| `custom_charges` | Custom/special charges |
| `additional_charges` | Any additional charges |
| `warehouse` | Warehouse (fulfillment location) |
| `customer` | Customer name |
| `creation` | Record creation timestamp |
| `modified` | Last modified timestamp |

#### `order_items` array

| Field | Description |
|-------|-------------|
| `erp_item` | Linked ERP Item code |
| `item_name` | Item description / name |
| `item_group` | Item group / category |
| `unit_price` | Unit price of the item |

> **Note on ss_shipping_amount / ss_tax_amount / ss_discount_amount:**
> These three fields are pulled from ShipStation and are provided for informational/reconciliation purposes only.
> They are **already accounted for inside `total_amount`** — do not add them again when computing the order total.

---

## Pagination

The response always includes:
- `total` — total number of matching records
- `page` — current page (1-based)
- `page_size` — records returned per page
- `total_pages` — total pages available

To fetch all records, iterate pages until `page >= total_pages`.

```python
all_orders = []
page = 1
while True:
    resp = requests.get(url, params={**filters, "page": page, "page_size": 100}, headers=headers)
    result = resp.json()["message"]
    all_orders.extend(result["data"])
    if page >= result["total_pages"]:
        break
    page += 1
```

---

## Example Requests

### Orders shipped in June 2025
```
GET /api/method/lh.api.get_lyfe_orders?workflow_state=Shipped&order_date_from=2025-06-01&order_date_to=2025-06-30
```

### High-value orders (above ₹5000) for a specific customer
```
GET /api/method/lh.api.get_lyfe_orders?customer=John+Doe&total_amount_gt=5000
```

### Orders created today from a specific warehouse
```
GET /api/method/lh.api.get_lyfe_orders?creation_from=2025-06-03&warehouse=Factory+-+LH
```

### Orders with amount between ₹1000 and ₹10000
```
GET /api/method/lh.api.get_lyfe_orders?total_amount_gt=1000&total_amount_lt=10000
```
