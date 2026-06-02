# ShipStation Label Generation

## Overview

Shipping labels are generated directly from a **Lyfe Order** using the ShipStation V2 API (`POST https://api.shipstation.com/v2/labels`). All label options are set on the order form itself — no separate configuration screen is needed per shipment.

**API version:** ShipStation V2  
**Auth:** `API-Key` header — key stored in **ShipStation Settings → API Key**  
**Code:** `lh/lyfe_hardware/shipping/shipstation_label_api.py`

---

## How to Generate a Label

1. Open the **Lyfe Order**.
2. Go to the **CJ Shipping** tab.
3. Fill in Consignor, Consignee, Parcel Details, and Shipment Items (required before dispatch anyway).
4. Optionally configure the **ShipStation Label Options** section (see fields below).
5. Click **Generate ShipStation Label** button in the **Label Generation** section.
6. A dialog confirms the carrier and service code from ShipStation Settings.
7. Click **Generate Label** — the PDF is downloaded and saved to the order, and the Tracking Number and Label ID are populated automatically.

---

## Where Data Comes From

| What | Source | Notes |
|------|--------|-------|
| **Ship From** (sender) | `cj_consignor_*` fields on the order | Populated automatically when you select an Export Partner |
| **Ship To** (receiver) | `cj_consignee_*` fields on the order | Filled from ShipStation / Shopify order data |
| **Weight & Dimensions** | `cj_parcel_details` table | Weight summed across all parcels; dimensions taken from first parcel |
| **Customs Items** | `cj_shipment_items` table | Auto-built from issued items (description, qty, USD rate, HTS code) |
| **IOSS Number** | `ioss_no` field on order | Sent as EU tax identifier if present |
| **Carrier & Service** | ShipStation Settings → Default Carrier / Service Code | Can be overridden per order via `ss_package_code` / service dialog |
| **External Reference** | Lyfe Order name | Sent as `external_shipment_id` to ShipStation |

---

## ShipStation Label Options Fields

These fields are in the **ShipStation Label Options** section under the CJ Shipping tab. All are optional — leave blank to use the carrier's default behaviour.

---

### Package

| Field | Type | Description |
|-------|------|-------------|
| **Package Code** | Text | Carrier-specific package type identifier. Use this when shipping in a carrier-provided box (e.g. `small_flat_rate_box` for USPS flat-rate, `fedex_box` for FedEx). Leave blank to use a custom box with dimensions from Parcel Details. |
| **Ship Date Override** | Date | The date printed on the label as the ship date. Defaults to today. Set a future date if you are printing labels in advance for a later dispatch. |

---

### Delivery

| Field | Type | Description |
|-------|------|-------------|
| **Delivery Confirmation** | Select | Controls what proof-of-delivery the carrier collects. Options: `none` (no confirmation), `delivery` (electronic delivery scan), `signature` (any adult signature), `adult_signature` (ID-verified adult), `direct_signature` (FedEx only — must sign in person). Leave blank for no confirmation. |
| **Residential Address** | Checkbox | Tick if the destination is a home address rather than a business. Affects carrier surcharges — residential deliveries typically cost more. If unticked, the address is treated as commercial. |

---

### Advanced Options

| Field | Type | Description |
|-------|------|-------------|
| **Saturday Delivery** | Checkbox | Request Saturday delivery from the carrier. Only available on select services (e.g. UPS, FedEx). Carrier will add a surcharge. |
| **Non-Machinable** | Checkbox | Flag this package as non-machinable — meaning it cannot be processed by carrier sorting machines due to irregular shape, rigid contents, or clasps/buttons. USPS charges an extra fee for non-machinable parcels. |

---

### Third-Party Billing

Use these fields when the shipping cost should be billed to someone other than Lyfe Hardware — for example, when a customer provides their own carrier account.

| Field | Type | Description |
|-------|------|-------------|
| **Bill To Party** | Select | Who pays the freight. `recipient` = the delivery recipient pays on receipt. `third_party` = a separate account is billed. Leave blank to bill the shipper (Lyfe Hardware). |
| **Bill To Account No.** | Text | The carrier account number to be billed. Required when Bill To Party is set. |
| **Bill To Postal Code** | Text | Postal code of the billing account holder. Required by some carriers (UPS, FedEx) to verify the account. |
| **Bill To Country Code** | Text | ISO 2-letter country code of the billing account (e.g. `US`, `GB`). Required by some carriers alongside the account number. |

---

### Insurance

| Field | Type | Description |
|-------|------|-------------|
| **Insurance Provider** | Select | Who provides the shipment insurance. `carrier` = the carrier's own built-in insurance (e.g. UPS declared value). `shipsurance` = third-party insurance via Shipsurance (lower rates, broader coverage). `shipsurance_declared` = Shipsurance with declared value coverage. Leave blank for no insurance. |
| **Insured Value (USD)** | Currency | The declared value of the shipment contents in USD. This is the maximum amount that can be claimed if the package is lost or damaged. Only sent to ShipStation when an Insurance Provider is selected. |

---

### International Options

Used for cross-border shipments to instruct customs authorities about the nature of the goods.

| Field | Type | Description |
|-------|------|-------------|
| **Customs Contents** | Select | Describes the nature of the goods for customs purposes. Options: `merchandise` (goods being sold), `gift` (personal gift, no commercial value), `documents` (papers/documents only), `returned_goods` (items being sent back), `sample` (commercial samples not for sale), `other`. Most outbound shipments should use `merchandise`. |
| **Non-Delivery Action** | Select | What the carrier should do if the shipment cannot be delivered and cannot be forwarded. `return_to_sender` = ship it back to the consignor. `treat_as_abandoned` = leave it with customs / discard. Choosing `return_to_sender` avoids customs seizure but incurs a return shipping fee. |

> **Note:** Customs line items (item description, quantity, unit value, HTS code, country of origin) are built **automatically** from the `cj_shipment_items` table on the order — no manual entry needed.

---

### Label Reference Messages

Text fields printed directly on the shipping label and packing slip. Availability depends on the carrier — not all carriers support all three reference lines.

| Field | Type | Description |
|-------|------|-------------|
| **Reference 1** | Text | Primary reference field. Common uses: customer order number, invoice number, or internal job ID. |
| **Reference 2** | Text | Secondary reference field. Common uses: purchase order number or shipment batch ID. |
| **Reference 3** | Text | Tertiary reference field. Use for any additional reference the customer or carrier needs on the label. |

---

## Label Generation Section (Read-Only Fields)

These fields are populated automatically after a label is generated and are read-only.

| Field | Description |
|-------|-------------|
| **ShipStation Label ID** | The unique ID assigned by ShipStation (e.g. `se-396884371`). Required to void the label. Cleared automatically if the label is voided. |
| **Tracking Number** | The carrier tracking number from the label. Also used by the 17Track integration for shipment tracking updates. |
| **Shipping Label** | Attached PDF file of the label (4×6 inch format). Click to download and print. |

---

## Voiding a Label

If a label needs to be cancelled (wrong address, order change, etc.):

1. Open the Lyfe Order menu (⋮ top right).
2. Click **Void ShipStation Label**.
3. Confirm the dialog.

**What happens:**
- ShipStation calls the carrier to cancel the shipment and request a refund of the label cost.
- If the carrier approves → Label ID, Tracking Number, and Shipping Label are cleared from the order. A new label can then be generated.
- If the carrier rejects (e.g. package already scanned/in transit) → nothing is cleared; the rejection reason is shown.

**Carrier time limits:**
- USPS: 28 days from label creation
- UPS / FedEx / others: 30 days from label creation
- Label must be unused (not yet picked up or scanned by the carrier)

The "Void ShipStation Label" menu item only appears on the order when a Label ID is present.

---

## Audit Log

Every label generation and void attempt is recorded in **Integration Request** (ERPNext → Integrations → Integration Request), with:
- Full request payload sent to ShipStation
- Full response received
- Status: Queued → Completed / Failed
- Linked to the Lyfe Order for easy tracing

---

## Configuration (ShipStation Settings)

| Setting | Description |
|---------|-------------|
| **API Key** (`ss_api_key`) | ShipStation V2 API key. Used for all label operations. |
| **Default Carrier (Labels)** (`ss_label_carrier`) | The carrier used when generating labels. Links to the Carrier doctype which holds the `carrier_id` required by the v2 API. |
| **Default Service Code (Labels)** (`ss_label_service_code`) | The default shipping service (e.g. `fedex_international_priority`). Shown in the generate dialog and sent to ShipStation unless overridden. |
