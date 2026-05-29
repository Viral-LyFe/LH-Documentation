# ShipStation Label Generation

## What This Feature Does

The **Generate ShipStation Label** button on a Lyfe Order lets you create a shipping label directly from the order form, without logging into ShipStation separately. Once the label is generated, it is saved to the order and the tracking number is filled in automatically.

---

## Who Can Use It

This button is visible only to **System Manager** and **Administrator** users.

---

## Settings to Configure After Deployment

Go to **ShipStation Settings** and fill in the following under the **Label Generation Defaults** section:

| Setting | What to Enter |
|---|---|
| **Sandbox Mode** | Leave this OFF for live use. Turn it ON only for testing (see Testing section below). |
| **Default Carrier (Labels)** | Select the carrier you want to use for labels. The carrier list is populated when you click **Sync Carriers** in ShipStation Settings. |
| **Default Service Code (Labels)** | Type the service code for the shipping service you want to use. For FedEx International Priority, this is `fedex_international_priority`. |

> **Important:** Make sure you have clicked **Sync Carriers** at least once so the carrier list is populated and the correct carrier can be selected.

---

## How to Generate a Label

1. Open the Lyfe Order.
2. Go to the **CJ Shipping** tab → **Label Generation** section.
3. Click **Generate ShipStation Label**.
4. A dialog will appear showing the carrier and service that will be used.
5. Choose a **Confirmation** type if needed (default is None).
6. Click **Generate Label**.
7. On success, the label PDF is saved to the order and the tracking number is updated.

### What the Order Needs Before Generating a Label

- The order must be linked to a ShipStation order (the **ShipStation Order** field must be filled).
- The consignee address fields (name, address, city, country, etc.) should be filled on the CJ Shipping tab.
- At least one row should be present in the **Parcel Details** table with weight and dimensions.

---

## Testing (Sandbox Mode)

Before going live, you can test label generation without creating real shipments or being charged.

To use sandbox mode:

1. Go to **ShipStation Settings**.
2. Under **Label Generation Defaults**, turn **Sandbox Mode** ON.
3. Under the **ShipEngine** section, enter the test API key provided for sandbox testing (it starts with `TEST_`).
4. Select the carrier you want to test with under **Default Carrier (Labels)**.

When Sandbox Mode is ON, a warning banner will appear in the Generate Label dialog so you always know you are in test mode. Labels generated in sandbox mode are not real — no shipment is created and no charge is applied.

To go back to live mode, simply turn **Sandbox Mode** OFF.

---

## What Happens After a Label Is Generated

- The label PDF is saved to the **Shipping Label** field on the order.
- The **Tracking Number** field is updated automatically.
- A record of the API call is saved under **Integrations → Integration Request** — you can check this if something goes wrong.

---

## If Something Goes Wrong

| What you see | What it means | What to do |
|---|---|---|
| "This shipment would exceed your subscription limit" | Your ShipStation plan does not include API label generation | Contact ShipStation support to upgrade, or use Sandbox Mode for testing in the meantime |
| "No carrier is configured" | The Default Carrier field is empty in ShipStation Settings | Go to ShipStation Settings, click Sync Carriers, then select the carrier under Label Generation Defaults |
| "This Lyfe Order is not linked to a ShipStation order" | The ShipStation Order field on the order is empty | Link the ShipStation order manually, or sync orders from ShipStation Settings |
| "ShipStation Order ID Missing" | The linked ShipStation order does not have an order ID yet | Go to ShipStation Settings and click Sync Now to fetch the latest order data |
| "Unauthorized" | The API key or secret stored in ShipStation Settings is incorrect | Check the API Key and API Secret fields in ShipStation Settings |
| Label saved but tracking number is blank | ShipStation did not return a tracking number for this shipment | Check the Integration Request log for the full response from ShipStation |
| Sandbox error about access denied | The test API key under ShipEngine section is missing or incorrect | Get a sandbox key from ShipEngine (free account) and enter it in ShipStation Settings → ShipEngine → Api Secret |

---

## After Deployment — Quick Checklist

Complete these steps in order before anyone starts generating labels:

1. **Sync Carriers** — Open ShipStation Settings and click the **Sync Carriers** button. This loads the available carriers into the system.
2. **Set the Default Carrier** — Under Label Generation Defaults, select the carrier you will use for shipments.
3. **Set the Default Service Code** — Enter the service code for your preferred shipping service (e.g. `fedex_international_priority` for FedEx International Priority).
4. **Confirm Sandbox Mode is OFF** — Make sure the Sandbox Mode checkbox is unchecked before going live.
5. **Test on one order** — Open a Lyfe Order that has a ShipStation order linked and parcel details filled, click Generate ShipStation Label, and confirm the label and tracking number are saved correctly.
