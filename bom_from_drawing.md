# BOM from Drawing — Feature Documentation

## Overview

The **Parse BOM from Drawing** feature automatically extracts a Bill of Materials from a PDF drawing attached to a Lyfe Order (`preview_pdf` field) and creates one or more **Lyfe BOM** records in ERPNext.

It follows a two-phase, human-in-the-loop flow:

1. **Phase 1 — Parse**: Extract tables from the PDF and attempt to match each component row to an existing ERP Item.
2. **Phase 2 — Confirm**: The user reviews the matches in a dialog, corrects any unmatched rows, then confirms to create the BOMs.

---

## Entry Point (UI)

A custom button **"Parse BOM from Drawing"** appears in the **BOM** button group on the Lyfe Order form whenever:
- The document is not new (`!frm.is_new()`), and
- A PDF is attached in `preview_pdf`.

```js
// lyfe_order.js — line ~551
if (!frm.is_new() && frm.doc.preview_pdf) {
    frm.add_custom_button(__("Parse BOM from Drawing"), () => {
        _parseBomFromDrawing(frm);
    }, __("BOM"));
}
```

---

## Files Involved

| File | Purpose |
|---|---|
| `lh/lyfe_hardware/doctype/lyfe_order/lyfe_order.js` | UI button, preview dialog, autocomplete inputs |
| `lh/lyfe_hardware/utils/bom_from_drawing.py` | Whitelisted server API — Phase 1 & 2 orchestrators |
| `lh/lyfe_hardware/utils/pdf_bom_parser.py` | PDF extraction and item-matching engine |

---

## Phase 1 — Parse (`parse_bom_from_drawing`)

**Server method:** `lh.lyfe_hardware.utils.bom_from_drawing.parse_bom_from_drawing`

### What it does

1. Loads the Lyfe Order and reads `preview_pdf` (throws if empty).
2. Calls `parse_bom_from_pdf(file_url)` from `pdf_bom_parser.py` to extract raw BOM tables.
3. Reads `order_items` from the order and annotates each extracted table with the most likely linked order item (keyword match on table title vs. item name).
4. Returns the full structured result to the client.

### PDF extraction (`pdf_bom_parser.py`)

Uses the **`pdfplumber`** library to open the PDF.

**Table detection:**
- Iterates over every page, extracts tables using `page.extract_tables()`.
- Identifies BOM tables by looking for header rows containing keywords: `component`, `item`, `description`, `qty`, `quantity`.
- Skips rows/tables containing noise keywords: `note`, `view`, `projection`, `email`, `client`, etc.

**Column mapping (dynamic):**
Columns are detected by their header label — not by position — so the parser adapts to layout variations:

| Column | Detected by keywords |
|--------|----------------------|
| Component / Item | `component`, `item`, `description`, `part` |
| Diameter | `diameter`, `diam`, `dia`, `size` |
| Finish | `finish`, `color`, `colour` |
| Length | `length`, `len`, `dimension` |
| Quantity | `qty`, `quantity`, `count`, `nos`, `pcs` |
| SKU | `sku`, `part no`, `part#`, `item code`, `code` |

**Page title extraction:**
Scans the first non-noise text line for keywords like `bunk`, `rail`, `ladder` to use as the table title (e.g. "Bunk 1", "Bunk 2"). Falls back to `"Page N"`.

**Reference maps (built dynamically from DB):**
- `finish_map`: `"PB" → "Polished Brass"` — built from `Product Finish` DocType (`abbreviation` field).
- `diameter_map`: `"1.5\"" → "150"` — derived from a REGEXP query on `tabItem` names that match the `-NNN-` SKU segment pattern.

### Item matching strategy

Each BOM row is resolved via `_find_item()` using up to 5 strategies in order:

| # | Strategy | Used when |
|---|----------|-----------|
| 1 | **Exact SKU match** | PDF has a SKU column — matches `item_code` or `custom_sku` exactly |
| 2 | **Tubing matching** | Component name contains `tubing`, `tube`, `pipe`, `rail`, or `rod` |
| 3 | **Name + finish + diameter** | SQL: `item_name LIKE keywords AND custom_finish = X AND name LIKE %-DIA-%` |
| 4 | **Keyword + SKU suffix pattern** | SQL: `name LIKE %-{diameter}-{finish}` AND keywords in `item_name` |
| 5 | **Broad keyword search** | SQL: `item_name LIKE keywords` — returns shortest match, flagged `unmatched` |

**Match statuses returned:**
- `matched` — confident ERP item found; green in the dialog.
- `unmatched` — partial or no match found; red in the dialog. User must fill the item code.
- `new_required` — tubing or component that needs a new Item master creating; yellow in the dialog.

**Tubing-specific logic (`_find_tubing_item`):**
1. Queries all tubing items matching `%-TB-{diameter}-{finish}` sorted by length.
2. Parses foot values from the item name prefix (e.g. `7P5FT` = 7.5 ft, using `P` as decimal separator).
3. Tries exact length match first; if not found, finds the next standard length ≥ the required cut length.
4. If no standard lengths exist at all → `new_required`, with a suggested SKU like `6P5FT-TB-150-PB`.

---

## Phase 2 — Confirm (`confirm_bom_from_drawing`)

**Server method:** `lh.lyfe_hardware.utils.bom_from_drawing.confirm_bom_from_drawing`

### What it does

Receives the user-confirmed table data (all item codes filled in) and for each BOM table:

1. **Resolves the parent item** from `linked_order_item.erp_item`; if missing, calls `_ensure_parent_item()` to find or create it.
2. **Creates new ERP Items** for any row still marked `new_required` that doesn't exist yet (`_create_new_item()`).
3. **Creates or updates a Lyfe BOM** via `_create_or_update_lyfe_bom()`.
4. Returns `{"created": [...bom names...], "errors": [...]}`.

### New item creation (`_create_new_item`)

Builds an item name from finish, diameter, and component (e.g. `"Polished Brass 1.5" Tubing"`), generates a safe `item_code` (uppercase, hyphens, unique), infers `item_group` from DB lookup, and sets custom fields: `custom_finish`, `custom_diameter`, `custom_length`, `custom_is_custom_item`.

### Lyfe BOM create/update (`_create_or_update_lyfe_bom`)

- If a Lyfe BOM already exists for `parent_id`, clears and rebuilds its `child_items` table.
- If none exists, creates a new Lyfe BOM.
- Populates each child row with `item_code`, `sku` (from `custom_sku`), `item_name`, and `quantity`.

---

## Dialog (Client-Side)

`_showBomPreviewDialog(frm, data)` in `lyfe_order.js`:

- Renders one HTML table per BOM table found in the PDF.
- Colour-codes each row: green (matched), yellow (new_required), red (unmatched).
- Provides an editable **ERP Item Code** input on each row with live autocomplete (searches `Item` by `item_name`).
- Shows which order item each BOM table is linked to.
- On **"Confirm & Create BOM"**: reads all inputs, warns if any are empty, calls `confirm_bom_from_drawing`, and reloads the form.

---

## Data Flow Diagram

```
Lyfe Order (preview_pdf attached)
        │
        ▼ click "Parse BOM from Drawing"
_parseBomFromDrawing(frm)
        │  frappe.call → parse_bom_from_drawing(order_name)
        │
        ▼ Server
parse_bom_from_pdf(file_url)
  ├── pdfplumber extracts raw tables
  ├── _build_finish_map()  ← Product Finish DocType
  ├── _build_diameter_code_map()  ← tabItem names regex
  └── per row: _find_item() → match_status + item_code
        │
        ▼ returns { tables: [...] }
_annotate_tables_with_order_items()
        │
        ▼ Client
_showBomPreviewDialog(frm, data)
  ├── User reviews, edits unmatched rows
  └── clicks "Confirm & Create BOM"
        │  frappe.call → confirm_bom_from_drawing(order_name, confirmed_tables_json)
        │
        ▼ Server
per table:
  ├── _ensure_parent_item()
  ├── _create_new_item()  ← for new_required rows
  └── _create_or_update_lyfe_bom()
        │
        ▼ Result shown in msgprint
  { created: [...], errors: [...] }
```

---

## Dependencies

- **`pdfplumber`** — must be installed in the bench virtualenv (`pip install pdfplumber`).
- **`Product Finish`** DocType — finish abbreviations must be populated for accurate finish resolution.
- **`tabItem`** — SKU naming convention `{FT}FT-TB-{diameter}-{finish}` for tubing items is assumed by the parser.

---

## Known Limitations / Open Items (as of 2026-05-27)

- Feature is on the `auto-bom` branch (in progress).
- `_validate_preview_pdf_attachment()` in `lyfe_order.py` is currently commented out — no server-side validation that the attachment is a PDF before parsing is attempted.
- Multi-table PDFs are annotated by keyword matching table title to order item name; this can fail if titles don't share keywords with item names.
- Tubing `new_required` rows suggest a SKU but the user must still confirm creation.
- No rollback if BOM creation partially fails — errors are reported per-table but already-created BOMs are not reversed.
