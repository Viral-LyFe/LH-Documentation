# Merge Order & Split Order — Complete Guide

## Table of Contents

1. [Overview](#overview)
2. [Merge Order](#merge-order)
   - [How Detection Works](#how-detection-works)
   - [Merge Requirements](#merge-requirements)
   - [Use Cases with Examples](#merge-use-cases)
   - [Reverting a Merge](#reverting-a-merge)
   - [Bulk Merge from List View](#bulk-merge-from-list-view)
3. [Split Order](#split-order)
   - [How Splitting Works](#how-splitting-works)
   - [Split Requirements](#split-requirements)
   - [Use Cases with Examples](#split-use-cases)
   - [Reverting a Split](#reverting-a-split)
4. [State Machine Reference](#state-machine-reference)
5. [API Reference](#api-reference)

---

## Overview

| Feature | Purpose | Who Uses It |
|---------|---------|-------------|
| **Merge Order** | Combine multiple orders from the same customer into one shipment | Customer Service |
| **Split Order** | Divide one order across multiple warehouses | Warehouse / Ops |

---

## Merge Order

### How Detection Works

Every time a Lyfe Order is created or updated, the system automatically scans for other orders that could be merged with it. A **Merge Suggestion** is created when:

- Another order exists from the **same customer email**
- The orders share the **same shipping postal code and country**
- Both orders are within a **72-hour window** of each other
- All orders are in an eligible state: `New`, `Internal Review`, or `Factory Assignment`

The Merge Suggestion appears in the **Merge Suggestion** doctype list for CS review.

---

### Merge Requirements

| Rule | Details |
|------|---------|
| Minimum orders | At least 2 orders |
| Customer match | Exact email match |
| Address match | Same postal code + country |
| Eligible states | New, Internal Review, Factory Assignment |
| Not already merged | `merged_into` must be empty |
| Not a split child | `parent_lyfe_order` must be empty |
| Time window | Within 72 hours of each other |

---

### Merge Use Cases

#### Use Case 1 — Customer places two orders on the same day

**Scenario:** Sarah orders a monitor stand (LYF-SH-2025-0010) at 9 AM and a cable kit (LYF-SH-2025-0011) at 11 AM. Same email, same address.

**What happens automatically:**
- LYF-SH-2025-0011 is saved → system detects LYF-SH-2025-0010 within 72 hours
- Merge Suggestion `MS-0001` is created with status `Pending CS Review`

**CS action — Confirm Merge:**
1. Open `MS-0001` in Merge Suggestion list
2. Review the side-by-side comparison (order details, items, validation status)
3. Click **Confirm Merge**

**Result:**
```
Before:
  LYF-SH-2025-0010  (New)  → 1x Monitor Stand
  LYF-SH-2025-0011  (New)  → 1x Cable Kit

After:
  LYF-SH-2025-0010-M  (New)  → 1x Monitor Stand + 1x Cable Kit   ← new merged order
  LYF-SH-2025-0010    (Merged, merged_into = LYF-SH-2025-0010-M)  ← archived
  LYF-SH-2025-0011    (Merged, merged_into = LYF-SH-2025-0010-M)  ← archived
```

The merged order takes the `-M` suffix and uses header fields (customer name, address, dates) from the **earliest** source order.

---

#### Use Case 2 — Customer places orders over two days but within 72 hours

**Scenario:** John orders on Monday evening (LYF-ET-2025-0050) and adds more on Tuesday morning (LYF-ET-2025-0051). Both arrive within 72 hours.

**What happens:** System auto-detects and creates Merge Suggestion `MS-0002`.

**CS action — Dismiss:**
CS reviews the orders and decides the Monday order is already being packed.

1. Open `MS-0002`
2. Click **Dismiss**
3. Enter reason: "Monday order already picked"

**Result:**
```
MS-0002 status → Dismissed
  dismissed_by = "cs@lyfehardware.com"
  dismissed_reason = "Monday order already picked"

LYF-ET-2025-0050  → unchanged (still New)
LYF-ET-2025-0051  → unchanged (still New)
```

No orders are affected.

---

#### Use Case 3 — Three orders from the same customer

**Scenario:** Emma submits three separate orders within 48 hours for the same delivery address.

**What happens:**
- System creates Merge Suggestion `MS-0003` with LYF-MN-2025-0020 as primary and LYF-MN-2025-0021, LYF-MN-2025-0022 as secondaries

**After confirming:**
```
Before:
  LYF-MN-2025-0020  (New)  → 2x Desk Lamp
  LYF-MN-2025-0021  (New)  → 1x USB Hub
  LYF-MN-2025-0022  (New)  → 3x Sticky Note Set

After:
  LYF-MN-2025-0020-M  (New)  → 2x Desk Lamp + 1x USB Hub + 3x Sticky Note Set
  LYF-MN-2025-0020    (Merged)
  LYF-MN-2025-0021    (Merged)
  LYF-MN-2025-0022    (Merged)
```

---

#### Use Case 4 — Merge from the List View (manual selection)

**Scenario:** CS spots two related orders in the list view that weren't auto-detected (e.g., slightly different name but same address).

1. In **Lyfe Order** list view, select 2+ orders using checkboxes
2. Click **Merge Orders** button (visible to Customer Service and System Manager)
3. System runs validation — shows a dialog with order comparison cards
4. If validation passes, click **Confirm**

**Result:** Same as Use Case 1 — a new `-M` merged order is created.

**Validation errors shown in dialog (merge blocked):**
- Orders have different postal codes
- One order is in `Factory Assignment` and another is `Completed`
- An order is already a split child

---

#### Use Case 5 — Bulk merge multiple suggestions at once

**Scenario:** CS has 10 pending merge suggestions built up overnight.

1. Go to **Merge Suggestion** list view
2. Filter to `Pending CS Review`
3. Select all suggestions to process
4. Click **Merge Selected** action

**Result:**
```
Processed: 10
  Merged: 8
  Errors: 2  (shown with reason — e.g., "Order LYF-SH-2025-0099 is no longer in a mergeable state")
```

---

### Reverting a Merge

**Scenario:** CS merged two orders but the customer called to change one of them.

**Requirement:** The merged order must still be in `New`, `Internal Review`, or `Factory Assignment`. Once it has progressed past that, revert is blocked.

**Steps:**
1. Open the Merge Suggestion (e.g., `MS-0001`) — status `Confirmed`
2. Click **Revert Merge**
3. Enter reason: "Customer requested change to order"

**Result:**
```
Before:
  LYF-SH-2025-0010-M  (New)
  LYF-SH-2025-0010    (Merged)
  LYF-SH-2025-0011    (Merged)

After:
  LYF-SH-2025-0010-M  → deleted
  LYF-SH-2025-0010    → restored to original workflow state (e.g., New)
  LYF-SH-2025-0011    → restored to original workflow state (e.g., New)
  MS-0001  status → Reverted
```

The system snapshots each order's original `workflow_state` at merge time, so revert always restores the correct state.

---

## Split Order

### How Splitting Works

Splitting divides one Lyfe Order into N **child split orders**, each assigned to a specific warehouse. The parent order's status becomes `Split` and is no longer processed directly.

Each split order gets:
- A suffix: `S1/2`, `S2/2`, `S3/3`, etc.
- A `parent_lyfe_order` link back to the original
- Its own subset of items (quantities redistributed from the parent)
- Status `New`
- Auto-assignment to Factory if warehouse name contains "india"

---

### Split Requirements

| Rule | Details |
|------|---------|
| Eligible parent states | Any state except `Cancelled`, `Completed`, `Split` |
| No split-of-split | Parent must not already be a split child (`parent_lyfe_order` must be empty) |
| Full distribution | All item quantities must be completely assigned across splits — no partial or leftover quantities allowed |
| Warehouse required | Every split must have a warehouse assigned |

---

### Split Use Cases

#### Use Case 1 — Order with items from two warehouses

**Scenario:** LYF-SH-2025-0030 contains:
- 2x Laptop Stand (stocked in US warehouse)
- 3x Phone Mount (stocked in India warehouse)

**CS/Ops action:**
1. Open LYF-SH-2025-0030
2. Click **Split Order**
3. Dialog shows a table with item rows and split columns:

```
Split Dialog:
+--------+----------------+--------+--------+-----------+
| Item   | Total Qty      | Split 1| Split 2| Remaining |
+--------+----------------+--------+--------+-----------+
| Laptop Stand     | 2    |   2    |   0    |     0     |
| Phone Mount      | 3    |   0    |   3    |     0     |
+--------+----------------+--------+--------+-----------+
  Warehouse:         [US WH] [India WH]
```

4. Click **Confirm Split**

**Result:**
```
LYF-SH-2025-0030         → status: Split (parent)
LYF-SH-2025-0030-S1/2    → 2x Laptop Stand, Warehouse: US WH
LYF-SH-2025-0030-S2/2    → 3x Phone Mount,  Warehouse: India WH (auto Factory-assigned)
```

---

#### Use Case 2 — Splitting one item across two locations

**Scenario:** LYF-ET-2025-0080 contains 10x Notebook. 6 are in Warehouse A, 4 are in Warehouse B.

**Split Dialog:**
```
+----------+-----------+--------+--------+-----------+
| Item     | Total Qty | Split 1| Split 2| Remaining |
+----------+-----------+--------+--------+-----------+
| Notebook |    10     |   6    |   4    |     0     |
+----------+-----------+--------+--------+-----------+
  Warehouse:            [WH-A]  [WH-B]
```

**Result:**
```
LYF-ET-2025-0080        → status: Split
LYF-ET-2025-0080-S1/2   → 6x Notebook, Warehouse: WH-A
LYF-ET-2025-0080-S2/2   → 4x Notebook, Warehouse: WH-B
```

---

#### Use Case 3 — Three-way split (multiple items, three warehouses)

**Scenario:** LYF-MN-2025-0100 has:
- 4x Desk Organizer
- 2x Monitor Arm
- 6x Cable Pack

Ops splits into 3 warehouses based on stock availability.

**Split Dialog:**
```
+----------------+-----------+--------+--------+--------+-----------+
| Item           | Total Qty | Split 1| Split 2| Split 3| Remaining |
+----------------+-----------+--------+--------+--------+-----------+
| Desk Organizer |     4     |   2    |   2    |   0    |     0     |
| Monitor Arm    |     2     |   0    |   0    |   2    |     0     |
| Cable Pack     |     6     |   3    |   0    |   3    |     0     |
+----------------+-----------+--------+--------+--------+-----------+
  Warehouse:              [WH-EU] [WH-US] [WH-IN]
```

**Result:**
```
LYF-MN-2025-0100        → status: Split
LYF-MN-2025-0100-S1/3   → 2x Desk Organizer + 3x Cable Pack, Warehouse: WH-EU
LYF-MN-2025-0100-S2/3   → 2x Desk Organizer,               Warehouse: WH-US
LYF-MN-2025-0100-S3/3   → 2x Monitor Arm   + 3x Cable Pack, Warehouse: WH-IN (Factory-assigned)
```

---

#### Use Case 4 — Validation failure: quantities not fully distributed

**Scenario:** Ops tries to split LYF-SH-2025-0055 (5x Pen Set) but only assigns 3 to Split 1 and 1 to Split 2.

```
Remaining = 1  ← shown in red in the dialog
```

**System blocks the split** with error: "All item quantities must be fully distributed before splitting."

Ops must assign the remaining 1x Pen Set before the Confirm button becomes active.

---

### Reverting a Split

**Requirement:** Each split order must be in `New` status. If any split has progressed (e.g., to `Factory Assignment`), only the `New` ones can be individually reverted.

#### Revert all splits at once

**Scenario:** Ops split LYF-ET-2025-0090 but then realized the warehouse assignment was wrong. All splits are still `New`.

1. Open LYF-ET-2025-0090 (status: `Split`)
2. Click **Revert Splits** button (visible only when all splits are in `New`)
3. Confirm

**Result:**
```
Before:
  LYF-ET-2025-0090        (Split)
  LYF-ET-2025-0090-S1/2   (New)
  LYF-ET-2025-0090-S2/2   (New)

After:
  LYF-ET-2025-0090-S1/2   → deleted
  LYF-ET-2025-0090-S2/2   → deleted
  LYF-ET-2025-0090        → status restored to New
```

#### Revert one split (partial revert)

Only possible via the API: `revert_split(split_order_name)`. If other splits still exist, the parent remains in `Split` status. The parent only returns to `New` when **all** split children are gone.

---

## State Machine Reference

### Merge Order States

```
Merge Suggestion lifecycle:
  Pending CS Review
    ├─→ Confirmed   (CS clicks Confirm Merge)
    │     └─→ Reverted  (CS clicks Revert Merge, merged order still mutable)
    └─→ Dismissed   (CS clicks Dismiss)

Source orders:
  New / Internal Review / Factory Assignment
    └─→ Merged  (after confirm — archived, not deleteable)

Merged order:
  New → [normal workflow] → Completed
```

### Split Order States

```
Parent order:
  New / Internal Review / ...
    └─→ Split  (after split created)
          └─→ New  (after all splits reverted)

Split child orders:
  New → [normal workflow] → Completed
  New → deleted  (if reverted while still New)
```

---

## API Reference

All methods are Frappe whitelisted and callable via `frappe.call`.

### Merge Order API

| Method | Parameters | Returns | Notes |
|--------|-----------|---------|-------|
| `detect_merge_candidates(order_name)` | `order_name: str` | suggestion name or None | Auto-called on order insert/update |
| `confirm_merge(suggestion_name)` | `suggestion_name: str` | merged order name | Creates `-M` order, archives sources |
| `dismiss_suggestion(suggestion_name, reason=None)` | `suggestion_name, reason` | None | Records dismissal metadata |
| `revert_merge(suggestion_name, reason=None)` | `suggestion_name, reason` | None | Deletes merged order, restores sources |
| `bulk_confirm_merge(suggestion_names)` | `suggestion_names: list` | `{"merged": [...], "errors": [...]}` | Batch confirm |
| `validate_orders_for_merge(order_names)` | `order_names: list` | validation result + order details | Pre-confirm check from list view |
| `merge_orders_from_list(order_names)` | `order_names: list` | `{"merged_order": ..., "suggestion": ...}` | One-shot merge from list view |
| `get_suggestion_details(suggestion_name)` | `suggestion_name: str` | full order details with validation | Used by form UI |

### Split Order API

| Method | Parameters | Returns | Notes |
|--------|-----------|---------|-------|
| `split_order(order_name, splits)` | `order_name: str`, `splits: JSON list` | list of created order names | `splits` format: `[{warehouse, items: [{sku, quantity, ...}]}]` |
| `revert_split(split_order_name)` | `split_order_name: str` | parent order name | Deletes one split, restores parent if last |
| `revert_splits_bulk(split_order_names)` | `split_order_names: list` | `{"reverted": [...], "errors": [...]}` | Batch revert |

### Example API Call — Split Order

```javascript
frappe.call({
  method: "lh.lyfe_hardware.doctype.lyfe_order.lyfe_order.split_order",
  args: {
    order_name: "LYF-SH-2025-0030",
    splits: JSON.stringify([
      {
        warehouse: "US Warehouse - LH",
        items: [{ sku: "LAPTOP-STAND-BLK", item_name: "Laptop Stand", quantity: 2 }]
      },
      {
        warehouse: "India Warehouse - LH",
        items: [{ sku: "PHONE-MNT-SLV", item_name: "Phone Mount", quantity: 3 }]
      }
    ])
  },
  callback(r) {
    console.log("Split orders created:", r.message);
    // ["LYF-SH-2025-0030-S1/2", "LYF-SH-2025-0030-S2/2"]
  }
});
```

### Example API Call — Confirm Merge

```javascript
frappe.call({
  method: "lh.lyfe_hardware.doctype.lyfe_order.merge_order.confirm_merge",
  args: { suggestion_name: "MS-0001" },
  callback(r) {
    console.log("Merged order:", r.message);
    // "LYF-SH-2025-0010-M"
  }
});
```

---

## Quick Reference Card

### Can I merge these orders?

| Situation | Can Merge? |
|-----------|-----------|
| Same customer, same address, both New | ✅ Yes |
| Same customer, one is Internal Review | ✅ Yes |
| Same customer, one is Completed | ❌ No |
| Same customer, different postal code | ❌ No |
| One order is already a split child | ❌ No |
| One order is already merged (archived) | ❌ No |
| Orders placed 80 hours apart | ❌ No (outside 72-hour window) |

### Can I split this order?

| Situation | Can Split? |
|-----------|-----------|
| Order is New | ✅ Yes |
| Order is Internal Review | ✅ Yes |
| Order is Factory Assignment | ✅ Yes |
| Order is Split (already split) | ❌ No |
| Order is Completed | ❌ No |
| Order is Cancelled | ❌ No |
| Order is a split child itself | ❌ No |

### Can I revert?

| Situation | Can Revert? |
|-----------|-----------|
| Merged order is New | ✅ Yes |
| Merged order is Internal Review | ✅ Yes |
| Merged order is Factory Assignment | ✅ Yes |
| Merged order is Completed | ❌ No |
| Split child is New | ✅ Yes |
| Split child is Factory Assignment | ❌ No |
