# Reshipment Feature — Design & Implementation Guide

Status: **Design locked, not yet implemented**
Owner: Lyfe Hardware / lh app
Related doctype: `Lyfe Order` (module `Lyfe Hardware`)

---

## 1. Business Problem

Today, when a customer needs a replacement shipment (carrier lost/damaged the package,
wrong item sent, manufacturing defect, warranty replacement, etc.), there is no dedicated
record for it. The only trace lives in scalar fields on the **Lyfe Order** itself
(`reship_date`, `sla_reship_by`, `return_type`, `rto_history` child table) — which only
support **one reship cycle's worth of context at a time** and mix return/refund and
reship concerns together.

This feature adds a first-class, linked record — **Lyfe Order Reshipment** — so that:

- Multiple reshipments per order are trackable independently (with their own reason,
  cost, tracking number, courier).
- The reshipment has its own lifecycle (workflow) separate from the original order's
  status.
- It reuses existing infrastructure (Gate Pass, Material Issue for Order) instead of
  duplicating shipping/inventory logic.
- Existing SLA scanning logic (which reads `reship_date` / `rto_history` on Lyfe Order)
  keeps working without modification.

---

## 2. Data Model

### 2.1 New doctype: `Lyfe Order Reshipment`

Standalone, linked doctype (not a child table) — non-submittable, workflow-driven,
same shape as `Lyfe Order` itself.

| Field | Type | Notes |
|---|---|---|
| `lyfe_order` | Link → Lyfe Order | Required. The order this reshipment belongs to. |
| `workflow_state` | (auto) | Added automatically once a Workflow is attached. |
| `reason` | Select | Carrier Lost / Carrier Damaged / Wrong Item Sent / Missing Items / Manufacturing Defect / Customer Return Replacement / Warranty Replacement / Other |
| `description` | Small Text | Free-text detail. |
| `items` | Table → Lyfe Order Reshipment Item | SKUs/qty being reshipped. |
| `reshipment_cogs` | Currency | Header rollup of item-level COGS. **Editable**, same as `Lyfe Order.cost_of_goods` (see §2.3). |
| `tracking_number` | Data | Independent of the parent order's `tracking_number` — see §2.4. |
| `carrier` | Link → Carrier | Independent of the parent order's `carrier`. |
| `track_status` | Small Text | Independent of the parent order's `track_status`. |
| `shipment_cost` | Currency | |
| `total_cost` | Currency | `shipment_cost + reshipment_cogs` (or similar rollup — finalize during implementation). |
| `gate_pass` | Link → Gate Pass | Read-only. Set once a Gate Pass is created for this reshipment. |

### 2.2 New child table: `Lyfe Order Reshipment Item`

| Field | Type |
|---|---|
| `item_code` | Link → Item |
| `item_name` | Data |
| `qty` | Int |
| `reshipment_cogs` | Currency (editable) |

### 2.3 COGS editability

`Lyfe Order.cost_of_goods` is a read-only rollup made editable via:
- `lyfe_order.js` — field is included in the `alwaysEditable` list (line ~508–530).
- Property setters (`lh/patches/make_order_item_cogs_editable.py`) that unlock the field
  at the DocType level.

`Lyfe Order Reshipment.reshipment_cogs` (and the child table's `reshipment_cogs`) must
follow the **same pattern**: added to `alwaysEditable` in the new doctype's JS, and a
property setter (or `in_list_view`/`read_only: 0` set directly in the child table JSON,
since this is a new doctype with no legacy read-only default to override).

### 2.4 Why tracking fields are NOT shared with Lyfe Order

`Lyfe Order.tracking_number` / `carrier` / `track_status` are **single scalar fields**
driven by the tracking cron (`order_tracking.py` → `scheduled_track_ready_orders`).
A reshipment is a *separate physical shipment* with its own tracking number and carrier —
reusing the parent's fields would overwrite the original shipment's tracking data.
Each `Lyfe Order Reshipment` therefore carries its own copy of these fields.

### 2.5 Sync back to legacy SLA fields

To avoid touching the existing SLA scanner (`lh_project/sla/scheduler/`), when a
reshipment reaches **Ready for Dispatch**:

1. Set `Lyfe Order.reship_date` = today (or the reshipment's dispatch date).
2. Append a row to `Lyfe Order.rto_history` summarizing this reshipment cycle
   (`reship_date`, `return_type` if applicable, reason).

This keeps `sla_reship_by` / SLA breach logic working exactly as it does today, driven
off the parent Lyfe Order, while the new doctype becomes the detailed source of truth
per-cycle.

---

## 3. Workflow

Created via the Frappe UI (**Workflow** doctype) — matching the existing convention used
for the `Lyfe Order` workflow itself (no JSON fixture file; lives only in the database).

```
Reshipment Request  --[Factory: Start Progress]-->  Reshipment In Progress
Reshipment In Progress --[Factory: Ready for Dispatch]--> Ready for Dispatch
```

| State | doc_status | Who can be here / create |
|---|---|---|
| Reshipment Request | 0 | Created by **Customer Service** |
| Reshipment In Progress | 0 | Transitioned by **Factory** |
| Ready for Dispatch | 0 | Transitioned by **Factory** |

Transition rules:

| From | To | Action label | Allowed Role |
|---|---|---|---|
| Reshipment Request | Reshipment In Progress | Start Progress | Factory |
| Reshipment In Progress | Ready for Dispatch | Ready for Dispatch | Factory |

`allow_self_approval = 1` on all transitions (matches Lyfe Order's workflow convention).

**No automatic Material Issue for Order creation** on any transition — this remains a
manual action, same as it is today for regular Lyfe Orders (`frappe.new_doc("Material
Issue for Order", {for_lyfe_order: ...})`).

---

## 4. Automation — Gate Pass on "Ready for Dispatch"

Mirrors the existing pattern in `lyfe_order.py` (`on_update`, lines ~495–509), applied to
the new doctype's `on_update`:

```python
old_state = self.get_doc_before_save().workflow_state if self.get_doc_before_save() else None

if self.workflow_state == "Ready for Dispatch" and old_state != "Ready for Dispatch":
    gp_name = self.create_gate_pass_for_reshipment()
    if gp_name:
        self.gate_pass = gp_name
        frappe.msgprint(f'Gate Pass created: <a href="/app/gate-pass/{gp_name}">{gp_name}</a>', alert=True)
```

`create_gate_pass_for_reshipment()` mirrors `Lyfe Order.create_gate_pass_if_not_exists()`:

- Creates (or reuses today's draft) `Gate Pass`.
- Appends a `Gate Pass Item` row with `order_id = self.lyfe_order` — **note:**
  `Gate Pass Item.order_id` is hardcoded as `Link → Lyfe Order`, so it always points to
  the parent order, not the reshipment doc itself. Distinguish reshipment-originated Gate
  Pass rows via a note in `Gate Pass Item.destination`/a small marker field if needed
  during implementation.

---

## 5. UI — "Create Reshipment" button

On `Lyfe Order` form (`lyfe_order.js`), same group/pattern as the existing
`Material Issue for Order` button (~line 999):

```js
if (!frm.is_new() && hasAnyRole(["Customer Service", "System Manager"])) {
    frm.add_custom_button(__("Reshipment"), () => {
        frappe.new_doc("Lyfe Order Reshipment", { lyfe_order: frm.doc.name });
    }, __("Create"));
}
```

Optionally add an "Open Reshipments" links-group button (server round-trip to list
existing reshipments for this order), matching the "Open Gate Pass" button pattern
(`lyfe_order.js` ~line 1024).

---

## 6. Navigation (once implemented)

| Action | Path |
|---|---|
| Create a reshipment | Open a **Lyfe Order** → **Create** button group → **Reshipment** |
| View all reshipments for an order | **Lyfe Order** form → **Create** group → **Open Reshipments** (if added), or Desk → **Lyfe Order Reshipment** list, filtered by `lyfe_order` |
| Reshipment list/report | Desk → search bar → **Lyfe Order Reshipment** |
| Move reshipment through workflow | Open the **Lyfe Order Reshipment** doc → use workflow action buttons in the top-right (e.g. "Start Progress", "Ready for Dispatch") |
| View the Gate Pass created from a reshipment | **Lyfe Order Reshipment** doc → `gate_pass` field (Link) → click to open |
| Check SLA sync | **Lyfe Order** form → `reship_date` field and `rto_history` table — should reflect the latest reshipment once it hits Ready for Dispatch |

---

## 7. Steps to Replicate the Implementation

1. **Create the child table doctype** `Lyfe Order Reshipment Item`
   (`bench --site <site> new-doctype` or manually under
   `lh/lyfe_hardware/doctype/lyfe_order_reshipment_item/`), `istable: 1`, fields per §2.2.

2. **Create the parent doctype** `Lyfe Order Reshipment`
   (`lh/lyfe_hardware/doctype/lyfe_order_reshipment/`), fields per §2.1, `is_submittable: 0`.

3. **Add editability** for `reshipment_cogs`:
   - Set `read_only: 0` directly in both the parent and child table JSON (no legacy
     property-setter override needed since this is a new doctype).
   - Add `reshipment_cogs` to the `alwaysEditable` list in the new doctype's JS if any
     other read-only-by-role logic is introduced later.

4. **Create the Workflow** via Desk UI:
   - Desk → **Workflow** → New → Document Type = `Lyfe Order Reshipment`,
     `workflow_state_field = workflow_state`.
   - Add the 3 states and 2 transitions per §3.

5. **Implement `on_update` hook** in `lyfe_order_reshipment.py`:
   - Gate Pass creation logic (§4).
   - Legacy field sync back to `Lyfe Order` (§2.5).

6. **Register `doc_events`** in `hooks.py` for `Lyfe Order Reshipment` → `on_update`.

7. **Add the "Create Reshipment" button** to `lyfe_order.js` (§5).

8. **Run migrations**: `bench --site <site> migrate`.

9. **Assign/verify roles**: confirm `Customer Service` and `Factory` roles have
   create/write permissions on `Lyfe Order Reshipment` (Role Permissions Manager).

---

## 8. Use Case to Test

**Scenario: Carrier damaged a package in transit; customer needs a replacement for 2 of the 3 items originally ordered.**

### Setup
- Pick an existing **Lyfe Order** that is already `Delivered` (or any post-dispatch status), e.g. order `LH-ORD-XXXX`.

### Steps

1. Open the Lyfe Order. Confirm a **Create → Reshipment** button is visible (logged in as
   a Customer Service user).
2. Click it. Confirm a new **Lyfe Order Reshipment** form opens with `lyfe_order`
   pre-filled to the order you opened.
3. Fill in:
   - `reason` = **Carrier Damaged**
   - `description` = "2 of 3 items damaged in transit, customer requested replacement"
   - `items` = add the 2 damaged SKUs with their quantities and a `reshipment_cogs` value
     per line (confirm the field is editable, not greyed out).
4. Save. Confirm `workflow_state` defaults to **Reshipment Request**.
5. As a **Factory** user, open the doc and transition it to **Reshipment In Progress**
   using the workflow action button. Confirm:
   - No Material Issue for Order is auto-created (this is expected — manual step).
   - Confirm the state actually changed and is visible on reload.
6. Manually create a **Material Issue for Order** for the 2 reshipped SKUs (via existing
   `frappe.new_doc` flow), submit it, and confirm stock is deducted correctly and does
   **not** touch the original order's already-issued items.
7. Transition the reshipment to **Ready for Dispatch**. Confirm:
   - A **Gate Pass** is auto-created (check for the `frappe.msgprint` alert and the
     `gate_pass` field getting populated on save/reload).
   - The Gate Pass contains a `Gate Pass Item` row with `order_id` = the original Lyfe
     Order (not the reshipment doc).
8. Fill in `tracking_number` and `carrier` on the **reshipment** doc. Confirm this does
   **not** overwrite the original Lyfe Order's `tracking_number`/`carrier` fields (open
   the Lyfe Order and verify they're unchanged from before this reshipment).
9. Reload the original **Lyfe Order**. Confirm:
   - `reship_date` has been updated to today's date.
   - A new row has been appended to `rto_history` referencing this reshipment cycle.
   - Existing SLA fields (`sla_reship_by`, `sla_breached`) behave as they did before —
     i.e. no regression in SLA scan behavior.
10. Repeat steps 2–9 for a **second** reshipment on the **same** order (e.g. a
    Warranty Replacement weeks later). Confirm:
    - Both reshipments exist independently and are both linked to the same `lyfe_order`.
    - `rto_history` now has two rows, one per cycle.
    - The original order's data isn't corrupted or overwritten by the second cycle.

### Pass criteria

- Reshipment can be created, moved through all 3 workflow states, and produces a Gate
  Pass only at the final state.
- COGS field is editable on the reshipment (mirrors `cost_of_goods` behavior).
- Reshipment tracking/carrier fields are fully independent of the parent order's.
- Legacy SLA fields on Lyfe Order sync correctly and multiple reshipments don't clobber
  each other.
- No Material Issue for Order is created automatically at any workflow transition.
