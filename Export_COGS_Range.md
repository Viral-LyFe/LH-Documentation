# Export COGS Range — Feature Documentation

## Overview

The **Export COGS Range** button on the Lyfe Order list view lets authorized users download a subset of orders as an Excel file containing Cost of Goods Sold (COGS) data. The factory reviews this file, corrects any missing or wrong values, and re-imports it back into ERPNext using the **Data Import** tool.

---

## Who Can Use It

Only users with the **Factory** or **System Manager** role see this button.

---

## Step 1 — Export the COGS File

1. Open **Lyfe Order** list view in ERPNext.
2. Click **Export COGS Range** (top-right area of the page).
3. A dialog appears with two required fields:

   | Field | Description |
   |---|---|
   | **From Internal Order ID** | The starting order in the range (e.g. `LO-0101`) |
   | **To Internal Order ID** | The ending order in the range (e.g. `LO-0200`) |

4. Click **Download Excel**.
5. The file downloads as `COGS_<from_id>_to_<to_id>.xlsx` (e.g. `COGS_LO-0101_to_LO-0200.xlsx`).

> **Note:** Cancelled orders are automatically excluded from the export.

---

## Excel File Structure

The exported sheet is named **COGS Export** and contains four columns:

| Column | Header | Description |
|---|---|---|
| A | Document Name | The ERPNext document name (used as the import key) |
| B | Order ID | The internal order ID (`internal_order_id` field) |
| C | Cost of Goods Sold | The current COGS value from the `cost_of_goods` field |
| D | Fulfilled From | `US Warehouse` if warehouse is "US Warehouse - LH", otherwise `Factory` |

---

## Step 2 — Factory Review and Update

The factory team opens the Excel file and:

- Verifies the **Cost of Goods Sold** value for each order.
- Fills in any missing values.
- Corrects any wrong values.
- Leaves **Column A (Document Name)** and all other columns unchanged — Column A is the import key ERPNext uses to match rows back to the correct document.

---

## Step 3 — Re-import via Data Import

Once the file is updated, upload it back to ERPNext using **Data Import**:

1. Go to **Data Import** (search in the top bar).
2. Click **New**.
3. Set **Document Type** to `Lyfe Order`.
4. Set **Import Type** to `Update Existing Records`.
5. Upload the corrected `.xlsx` file.
6. Click **Map Columns** and confirm that **Document Name** maps to `name` (ERPNext's primary key).
7. Map **Cost of Goods Sold** to the `cost_of_goods` field.
8. Click **Start Import**.

ERPNext will update the `cost_of_goods` field on each matched Lyfe Order document.

---

## Important Notes

- The `cost_of_goods` field becomes **read-only** once an order reaches **Completed** or **Cancelled** status. Import updates to those orders will be rejected.
- Do not alter **Column A (Document Name)** in the Excel file — it is the only way ERPNext knows which record to update.
- The ID range is **inclusive** on both ends (`from_id` and `to_id` are both included in the export).
- Both IDs must share the same prefix (e.g. both must start with `LO`).
