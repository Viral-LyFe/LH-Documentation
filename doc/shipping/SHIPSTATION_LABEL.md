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
6. A dialog confirms the carrier and service code from ShipStation Settings. Click **Generate Label**.
7. The PDF is downloaded, saved to the order, and the Tracking Number and Label ID are populated automatically.

### Pre-flight Validation

Clicking the button validates all fields required by the ShipStation V2 API before making the API call. The following are checked and must be filled:

**Consignor (Ship From)**
- Consignor Name, Address Line 1, City, State, Postal Code, Phone

**Consignee (Ship To)**
- Consignee Name, Address Line 1, City, State / Province, Postal Code, Country, Phone

**Package**
- At least one row in Parcel Details

If any are missing, a red error dialog lists them and the API call is blocked.

---

## Where Data Comes From

| What | Source | Notes |
|------|--------|-------|
| **Ship From** (sender) | `cj_consignor_*` fields on the order | Populated automatically when you select an Export Partner |
| **Ship To** (receiver) | `cj_consignee_*` fields on the order | Filled from ShipStation / Shopify order data |
| **Weight & Dimensions** | `cj_parcel_details` table | Weight summed across all parcels; dimensions taken from first parcel. Converted from kg/cm to oz/inches automatically. |
| **Customs Items** | `cj_shipment_items` table | Auto-built from issued items — description, qty, USD rate, HTS code, country of origin (hardcoded `IN`) |
| **IOSS Number** | `ioss_no` field on order | Sent as EU tax identifier (`identifier_type: ioss`) if present |
| **DDP Flag** | `ddp` field on order | If set to `Yes`, sends `delivered_duty_paid: true` in advanced options |
| **Carrier & Service** | ShipStation Settings → Default Carrier / Service Code | Carrier ID looked up from the linked Carrier doctype |
| **External Reference** | Lyfe Order name | Sent as `external_shipment_id` to ShipStation for traceability |

---

## ShipStation Label Options Fields

These fields are in the **ShipStation Label Options** section under the CJ Shipping tab (before the Invoice section). All are optional — leave blank to use the carrier's default behaviour.

---

### Package

| Field | Fieldname | Type | Description |
|-------|-----------|------|-------------|
| **Package Code** | `ss_package_code` | Text | Carrier-specific package type identifier. Use when shipping in a carrier-provided box (e.g. `small_flat_rate_box` for USPS flat-rate, `fedex_box` for FedEx). Leave blank to use a custom box with dimensions from Parcel Details. |
| **Ship Date Override** | `ss_ship_date` | Date | The date printed on the label as the ship date. Defaults to today. Set a future date if printing labels in advance for a later dispatch. |

---

### Delivery

| Field | Fieldname | Type | Description |
|-------|-----------|------|-------------|
| **Delivery Confirmation** | `ss_confirmation` | Select | Controls what proof-of-delivery the carrier collects. `none` = no confirmation. `delivery` = electronic delivery scan. `signature` = any adult signature. `adult_signature` = ID-verified adult. `direct_signature` = FedEx only, must sign in person. Leave blank for no confirmation. |
| **Residential Address** | `ss_residential` | Checkbox | Tick if the destination is a home address rather than a business. Residential deliveries typically incur a surcharge. If unticked, the address is treated as commercial. |

---

### Advanced Options

| Field | Fieldname | Type | Description |
|-------|-----------|------|-------------|
| **Saturday Delivery** | `ss_saturday_delivery` | Checkbox | Request Saturday delivery from the carrier. Only available on select services (e.g. UPS, FedEx). Carrier will add a surcharge. |
| **Non-Machinable** | `ss_non_machinable` | Checkbox | Flag this package as non-machinable — cannot be processed by carrier sorting machines due to irregular shape, rigid contents, or clasps/buttons. USPS charges an extra fee for non-machinable parcels. |

---

### Third-Party Billing

Use these fields when the shipping cost should be billed to someone other than Lyfe Hardware — for example, when a customer provides their own carrier account.

| Field | Fieldname | Type | Required | Description |
|-------|-----------|------|----------|-------------|
| **Bill To Party** | `ss_bill_to_party` | Select | — | Who pays the freight. `recipient` = the delivery recipient pays on receipt. `third_party` = a separate account is billed. Leave blank to bill Lyfe Hardware. |
| **Bill To Account No.** | `ss_bill_to_account` | Text | When Bill To Party is set | The carrier account number to be billed. |
| **Bill To Postal Code** | `ss_bill_to_postal_code` | Text | When Bill To Party is set | Postal code of the billing account holder. Required by UPS and FedEx to verify the account. |
| **Bill To Country Code** | `ss_bill_to_country_code` | Text | When Bill To Party is set | ISO 2-letter country code of the billing account (e.g. `US`, `GB`). |

> The Account No., Postal Code, and Country Code fields are **hidden** until Bill To Party is selected, and become **mandatory** once it is set.

---

### Insurance

| Field | Fieldname | Type | Required | Description |
|-------|-----------|------|----------|-------------|
| **Insurance Provider** | `ss_insurance_provider` | Select | — | Who provides the shipment insurance. `carrier` = the carrier's own built-in insurance (e.g. UPS declared value). `shipsurance` = third-party insurance via Shipsurance (lower rates, broader coverage). `shipsurance_declared` = Shipsurance with declared value coverage. Leave blank for no insurance. |
| **Insured Value (USD)** | `ss_insured_value` | Currency | When Insurance Provider is set | The declared value of the shipment in USD — the maximum claimable amount if lost or damaged. Hidden and mandatory only when an Insurance Provider is selected. |

---

### Duties & Taxes

Duty and tax handling is controlled by the **DDP** field in the main order (not inside the label options section), and by **IOSS No.** on the order.

| Field | Fieldname | Where | Description |
|-------|-----------|-------|-------------|
| **DDP** | `ddp` | Main order fields | `Yes` = Delivered Duty Paid — shipper pays all import duties and taxes upfront so the recipient pays nothing on delivery. `No` / blank = DDU — recipient pays customs on arrival. Sent to ShipStation as `delivered_duty_paid` in advanced options. |
| **IOSS No.** | `ioss_no` | Main order fields | EU Import One-Stop Shop registration number. Required for shipments to EU countries where the order value is under €150 and VAT was collected at point of sale. Sent as a tax identifier to ShipStation. |

---

### International Options

Used for cross-border shipments to instruct customs authorities about the nature of the goods.

| Field | Fieldname | Type | Description |
|-------|-----------|------|-------------|
| **Customs Contents** | `ss_customs_contents` | Select | Nature of goods for customs. `merchandise` = goods being sold (use for most outbound orders). `gift` = personal gift, no commercial value. `documents` = papers only. `returned_goods` = items being sent back. `sample` = commercial samples not for sale. `other` = anything else. |
| **Non-Delivery Action** | `ss_non_delivery` | Select | What the carrier should do if the shipment cannot be delivered. `return_to_sender` = ship back to consignor (incurs return fee, avoids customs seizure). `treat_as_abandoned` = leave with customs / discard. |

> **Customs line items** (item description, quantity, unit value in USD, HTS code, country of origin = IN) are built **automatically** from the `cj_shipment_items` table — no manual entry needed.

---

### Label Reference Messages

Text fields printed directly on the shipping label and packing slip. Availability depends on the carrier — not all carriers support all three lines.

| Field | Fieldname | Type | Description |
|-------|-----------|------|-------------|
| **Reference 1** | `ss_reference1` | Text | Primary reference. Common uses: customer order number, invoice number, or internal job ID. |
| **Reference 2** | `ss_reference2` | Text | Secondary reference. Common uses: purchase order number or shipment batch ID. |
| **Reference 3** | `ss_reference3` | Text | Tertiary reference for any additional information the customer or carrier needs on the label. |

---

## Label Generation Section (Read-Only Fields)

These fields are populated automatically after a label is generated and cannot be edited manually.

| Field | Fieldname | Description |
|-------|-----------|-------------|
| **ShipStation Label ID** | `ss_label_id` | Unique ID assigned by ShipStation (e.g. `se-396884371`). Used internally to void the label. Cleared automatically when the label is voided. |
| **Tracking Number** | `tracking_number` | Carrier tracking number from the label. Also picked up by the 17Track integration for shipment tracking updates. |
| **Shipping Label** | `shipping_label` | Attached PDF (4×6 inch format). Click to download and print. |

---

## Voiding a Label

If a label needs to be cancelled (wrong address, order change, etc.):

1. Open the Lyfe Order **Menu** (⋮ top right).
2. Click **Void ShipStation Label** — this option only appears when a Label ID is present.
3. Confirm the dialog.

**What happens:**
- ShipStation contacts the carrier to cancel the shipment and request a cost refund.
- If **approved** → Label ID, Tracking Number, and Shipping Label are all cleared from the order. A new label can then be generated.
- If **rejected** (e.g. package already scanned, in transit, or beyond time limit) → nothing is cleared; the carrier's rejection reason is shown in orange.

**Carrier time limits:**
- USPS: 28 days from label creation
- UPS / FedEx / others: 30 days from label creation
- Label must be unused — not yet picked up or scanned by the carrier

---

## Audit Log

Every label generation and void attempt is recorded in **Integration Request** (ERPNext → Integrations → Integration Request):

- Full JSON request payload sent to ShipStation
- Full JSON response received from ShipStation
- Status: `Queued` → `Completed` / `Failed`
- Error message on failure
- Linked to the Lyfe Order for easy lookup

---

## Configuration (ShipStation Settings)

| Setting | Fieldname | Description |
|---------|-----------|-------------|
| **API Key** | `ss_api_key` | ShipStation V2 API key. Used as the `API-Key` header for all label and void operations. |
| **Default Carrier (Labels)** | `ss_label_carrier` | The carrier used when generating labels. Links to the Carrier doctype which stores the `carrier_id` required by the v2 API. |
| **Default Service Code (Labels)** | `ss_label_service_code` | Default shipping service (e.g. `fedex_international_priority`). Shown in the generate dialog and passed to ShipStation. |
