# Getting a Shipping Rate on a Quotation — User Guide

This guide explains how to use the **"Get Shipping rate"** tool on a Quotation to estimate shipping cost, and how that number needs to be applied to the Quotation total.

---

## When do I use this?

Use this whenever you need to estimate the shipping cost for an order **before** it is confirmed — for example, while preparing a Quotation and you want to know roughly what shipping will cost so you can quote the customer accurately or add the charge to the Quotation.

---

## Step 1: Fill in the Box / Parcel Details

Before the shipping rate tool becomes available, the Quotation needs its package size filled in — the system needs to know what it's shipping. Either of the following works:

- **Combined box dimensions** — fill in Box Length, Box Width, and Box Height, **or**
- **Parcel Size table** — add one or more rows to the Parcel Size table (used when items ship as separate parcels rather than one combined box)

> The **"Get Shipping rate"** button will not appear until one of these is filled in. It's also only available while the Quotation is still in **Draft**.

---

## Step 2: Open the Rate Tool

Go to **Tools → Get Shipping rate** on the Quotation form.

> 📷 *[Screenshot: "Get Shipping rate" under the Tools menu]*

A dialog will open asking for:

- **Carrier** — defaults to the standard carrier, but can be changed.
- **Destination Country** — automatically filled in from the Shipping Address or Customer Address on the Quotation (read-only).
- **Signature Required** — check if the shipment needs signature confirmation on delivery.
- **Is Commercial** — check if the destination is a commercial address (vs. residential).

> 📷 *[Screenshot: Get Shipping Rate dialog]*

---

## Step 3: Review the Rate

Click through to calculate. Depending on whether you used combined box dimensions or the Parcel Size table, you'll see either:

- A single rate breakdown (chargeable weight, zone, base rate, surcharges, fuel surcharge, and total), or
- A comparison table showing rates across different packing types, so you can compare options.

> 📷 *[Screenshot: Rate result — breakdown table]*

This is a **quote/estimate only** — it tells you what the shipping is expected to cost, based on box size, weight, destination zone, and carrier rate card.

---

## Step 4: Apply the Charge to the Quotation

**This is the important part: the rate shown in Step 3 is not applied automatically.** It's shown to you in a popup so you can read the number — it does not get written anywhere on the Quotation by itself.

To make it count toward the Quotation total, you need to manually enter it:

1. Scroll to the **Sales Taxes and Charges** table on the Quotation.
2. Find (or add) the row labeled **"Shipping Charges."**
3. Enter the amount from Step 3 into that row's charge amount.
4. Save the Quotation.

Once entered, this behaves like a normal tax/charge row — it flows into **Total Taxes and Charges**, and from there into the Quotation's **Grand Total**.

---

## How the Charges Are Applied — Technical Summary

- The **"Get Shipping rate"** tool (Tools menu) is a **read-only calculator**. It reads the Quotation's box/parcel dimensions and destination, runs the rate-card calculation (zone, base rate, ODA/DRS surcharges, fuel surcharge), and displays the result in a dialog. It does **not** save anything back to the Quotation.
- The actual shipping cost only affects the Quotation's total once it is **manually entered into the "Shipping Charges" row** of the Sales Taxes and Charges table. That row is a standard "Actual" charge type, so once entered it adds directly to Total Taxes and Charges → Grand Total, the same as any other tax/charge line.
- In short: **quoting a rate and applying a rate are two separate manual steps.** Getting the rate does not change the order total by itself — you must copy the number into the Shipping Charges row yourself.

---

## Quick Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Fill in box dimensions or Parcel Size rows | Rate tool becomes available |
| 2 | Tools → Get Shipping rate → fill dialog | Rate calculated |
| 3 | Review the quoted rate | Estimate only — nothing saved yet |
| 4 | Manually enter the amount into the "Shipping Charges" row in Sales Taxes and Charges | Amount now included in Grand Total |
