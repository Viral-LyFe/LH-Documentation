# ShipStation Label Generation

## Overview

Lyfe Order supports generating shipping labels directly via the **Generate ShipStation Label** button on the CJ Shipping tab (Label Generation section). The integration uses the ShipStation classic REST API (`ssapi.shipstation.com`) in production and the ShipEngine API (`api.shipengine.com`) for sandbox/testing.

**Button visibility:** System Manager and Administrator roles only.

---

## How It Works

### Production Mode (`ss_label_sandbox = OFF`)

Calls `POST https://ssapi.shipstation.com/orders/createlabelfororder` using the linked ShipStation Order's numeric ID. Carrier and service are read from ShipStation Settings. Returns a Base64-encoded PDF label saved to `Lyfe Order.shipping_label` and updates `tracking_number`.

### Sandbox Mode (`ss_label_sandbox = ON`)

Calls `POST https://api.shipengine.com/v1/labels` using the ShipEngine API with a `TEST_` key. Builds a full shipment payload from the CJ Shipping tab fields (consignee address + parcel details). No real shipment is created and no charge is applied. Works for all carriers including FedEx.

---

## Prerequisites

### Production

| Requirement | Where to check |
|---|---|
| ShipStation plan that includes API label generation | ShipStation account billing page |
| `ss_api_key` and `ss_api_secret` configured | ShipStation Settings → ShipStation section |
| Lyfe Order must be linked to a ShipStation Order (`shipstation_order` field) | Lyfe Order form |
| ShipStation Order must have a numeric `shipstation_order_id` | Auto-filled on sync |

### Sandbox

| Requirement | Where to get it |
|---|---|
| ShipEngine account (free) | [app.shipengine.com](https://app.shipengine.com/) |
| ShipEngine `TEST_` API key | ShipEngine dashboard → API Keys → Sandbox |
| ShipEngine carrier ID (e.g. `se-123456`) | `GET https://api.shipengine.com/v1/carriers` with your TEST_ key |

---

## Production Deployment Checklist

After deploying to production, complete these steps in order.

### Step 1 — Run Migration

```bash
bench --site <your-site> migrate
```

This creates the new `ss_label_carrier`, `ss_label_service_code`, `ss_label_sandbox`, and `se_label_carrier_id` columns in the ShipStation Settings single doctype.

### Step 2 — Sync Carriers

Open **ShipStation Settings** → click **Sync Carriers**. This populates the Carrier doctype with your live carrier records including their `carrier_id` and `carrier_code` values.

### Step 3 — Configure Label Generation Defaults

Go to **ShipStation Settings → Label Generation Defaults** and set:

| Field | Value | Notes |
|---|---|---|
| **Sandbox Mode** | ☐ OFF | Must be OFF for production |
| **Default Carrier (Labels)** | Select from dropdown | e.g. `FedEx One Balance (USD)` — `carrier_code` and `carrier_id` are read automatically from the selected Carrier record |
| **Default Service Code (Labels)** | e.g. `fedex_international_priority` | Must match the selected carrier's available service codes |

To find the correct service code for your carrier, check the Carrier record's **Services** child table or refer to the table below.

#### FedEx One Balance — International Service Codes

| Service Code | Service Name |
|---|---|
| `fedex_international_priority` | FedEx International Priority® |
| `fedex_international_economy` | FedEx International Economy® |
| `fedex_international_first` | FedEx International First® |
| `fedex_international_priority_express` | FedEx International Priority® Express |
| `fedex_international_connect_plus` | FedEx International Connect Plus® |
| `fedex_ground_international` | FedEx International Ground® |

### Step 4 — Verify ShipStation Plan

The `/orders/createlabelfororder` endpoint requires a ShipStation plan that includes API label generation. To verify:

1. Log into ShipStation → Settings → Account → Billing
2. Confirm the plan includes **API Access** or **Label Generation via API**
3. If you see `This shipment would exceed your subscription limit` when testing, contact ShipStation support to upgrade

### Step 5 — Test with a Real Order

1. Open a Lyfe Order that has:
   - `shipstation_order` field linked to a ShipStation Orders record
   - Consignee address filled (cj_consignee_* fields)
   - At least one row in `cj_parcel_details` with weight and dimensions
2. Click **Generate ShipStation Label**
3. Confirm the carrier/service shown in the dialog is correct
4. Click **Generate Label**
5. On success: `shipping_label` field is populated, `tracking_number` is updated, and an Integration Request log is created with service name `ShipStation Label`

---

## Sandbox / Testing Setup

### Step 1 — Get a ShipEngine Sandbox Key

1. Sign up at [app.shipengine.com](https://app.shipengine.com/) (free)
2. Go to **API Management** → **Sandbox** tab
3. Generate a new API key — it will start with `TEST_`

### Step 2 — Get Your ShipEngine Carrier ID

Run this once with your TEST_ key to find the carrier ID:

```bash
curl -H "API-Key: TEST_your_key_here" https://api.shipengine.com/v1/carriers
```

Or in Python:

```python
import requests
resp = requests.get(
    "https://api.shipengine.com/v1/carriers",
    headers={"API-Key": "TEST_your_key_here"}
)
for c in resp.json().get("carriers", []):
    print(c["carrier_id"], c["carrier_code"], c["friendly_name"])
```

Note the `carrier_id` for FedEx (e.g. `se-123456`).

### Step 3 — Configure Sandbox Settings

Go to **ShipStation Settings** and set:

| Section | Field | Value |
|---|---|---|
| ShipEngine | **Api Secret** (`se_api_secret`) | Your `TEST_xxxx` key |
| Label Generation Defaults | **Sandbox Mode** | ☑ ON |
| Label Generation Defaults | **Default Carrier (Labels)** | Select the Carrier record whose `carrier_id` matches what the ShipEngine sandbox returned |
| Label Generation Defaults | **Default Service Code** | e.g. `fedex_international_priority` |

> **Note:** When Sandbox Mode is ON, the ShipEngine carrier ID (`carrier_id` from the linked Carrier record) is used instead of the carrier code. Make sure the selected Carrier record has the correct `carrier_id` from ShipEngine.

### Step 4 — Test

Click **Generate ShipStation Label** on any Lyfe Order. The dialog will show an orange **Sandbox Mode is ON** banner. After generation, the label PDF is saved but no real shipment or charge is created.

---

## Integration Request Logs

Every label generation attempt (success or failure) is logged as a Frappe **Integration Request** document.

| Field | Value |
|---|---|
| Integration Request Service | `ShipStation Label` (production) or `ShipEngine Label (Sandbox)` |
| Reference DocType | `Lyfe Order` |
| Reference DocName | The Lyfe Order name |
| Status | `Queued` → `Completed` or `Failed` |

To view logs: **Home → Integrations → Integration Request** → filter by service name.

---

## Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `This shipment would exceed your subscription limit` | ShipStation plan does not include API label generation | Upgrade ShipStation plan, or use Sandbox Mode with ShipEngine for testing |
| `Unauthorized (401)` | Wrong API key or secret | Check `ss_api_key` / `ss_api_secret` in ShipStation Settings |
| `No carrier is configured` | `Default Carrier (Labels)` not set | Set the carrier in ShipStation Settings → Label Generation Defaults |
| `ShipStation Order ID Missing` | ShipStation Orders record has no `shipstation_order_id` | Sync orders from ShipStation Settings → Sync Now |
| `This Lyfe Order is not linked to a ShipStation order` | `shipstation_order` field is empty on Lyfe Order | Link the ShipStation order manually or sync from ShipStation |
| ShipEngine sandbox `Access denied` | `se_api_secret` is not a valid ShipEngine `TEST_` key | Generate a sandbox key at app.shipengine.com and update `se_api_secret` |
| Label PDF saved but tracking number blank | ShipStation/ShipEngine returned empty `trackingNumber` | Check Integration Request log output for the full API response |

---

## File Reference

| File | Purpose |
|---|---|
| `lh/lyfe_hardware/shipping/shipstation_label_api.py` | All label generation logic — production (ShipStation) and sandbox (ShipEngine) paths |
| `lh/lyfe_hardware/doctype/shipstation_settings/shipstation_settings.json` | ShipStation Settings schema — Label Generation Defaults section |
| `lh/lyfe_hardware/doctype/lyfe_order/lyfe_order.js` | `_generateShipStationLabel()` dialog and button handler |
| `lh/lyfe_hardware/doctype/lyfe_order/lyfe_order.json` | `generate_ss_label_btn` button field definition |
