# UI Testing Guide — BR-2 (Reverse SLA Calculator) & LH-DEV-006 (Priority Upstream)

**Date:** 2026-06-18  
**Site:** lyfe.local.local  
**Tester:** Viral (viralk@lyfehardware.com)

---

## Pre-Test Checklist

Before running any scenario, confirm the following:

- [ ] **Build is done:** `bench build --app lh` has been run — the dispatch calculator dialog will be a blank page until the JS bundle is rebuilt
- [ ] **Migrate is done:** `bench --site lyfe.local.local migrate` completed without errors
- [ ] **Notification constraint ACTIVE:** Only viralk@lyfehardware.com should receive any email or Slack notification during testing. Access Settings → Urgent Order Notifications must contain only that address.
- [ ] **Access Settings configured:** Open Access Settings (search: "Access Settings") → confirm `urgent_notification_users` has only `viralk@lyfehardware.com` and that `urgent_slack_channel` field is visible

---

## Quick Reference: Expected Pipeline Buffers

These are the working-day (Mon–Sat) numbers the calculator uses. Use them to sanity-check results manually.

| Segment | Standard | Custom |
|---|---|---|
| Production | 3 wd | 14 wd |
| Photo Approval | 0 wd | 4 wd |
| Gate Pass / Packing | 3 wd | 3 wd |
| India Transit | 7 wd | 7 wd |
| Customs (calendar days, not wd) | 7 cal | 7 cal |
| US Leg (if via US) | 4 wd | 4 wd |
| Last-Mile Delivery | 2 wd | 2 wd |

**Verdicts:**
- Buffer > 2 working days → **Possible** (green)
- Buffer = 0, 1, or 2 working days → **Borderline** (amber)
- Buffer < 0 → **Not Possible** (red)

---

## Scenario 1 — Standard Order, Possible Verdict

**Purpose:** Confirm the dialog opens, calculates, and applies correctly for a comfortable delivery window.

**Setup:** Open any existing Lyfe Order with `order_type = Standard` that is in progress (e.g., Factory Assignment or Ready for Dispatch state).

**Steps:**

1. Open the Lyfe Order form.
2. Look for the **Urgent** button group in the toolbar (top right, beside other action buttons).
3. Click **Urgent → Calculate Dispatch Dates**.
4. Dialog appears with fields: "Customer's Required Delivery Date", "Order Type" (pre-filled to Standard), "Route via US Warehouse", "Include Customs Clearance Buffer" (checked by default).
5. Enter a Required Delivery Date **30 working days from today** (roughly 6 calendar weeks out, avoiding Sundays).
6. Leave Order Type = Standard, US Warehouse = unchecked, Customs = checked.
7. Click **Calculate**.

**Expected result:**
- Spinner shows briefly, then result panel renders.
- Verdict badge is **green "Possible"** with a runway count (e.g., "8 working day(s) of runway").
- Three rows visible in the key dates table: Latest Dispatch Date, Dispatch Window, Latest Order Placement Date.
- Pipeline breakdown is collapsed but expandable — click it and verify production/gate pass/transit segments match the Standard column above.
- Primary button changes to **"Confirm & Apply"**.
- Secondary button **"Skip for now"** remains visible.

8. Click **Confirm & Apply**.

**Expected result:**
- Dialog closes.
- Green toast: "Dispatch dates applied. Save the document to confirm."
- `promised_dispatch_by` field on the form is filled with the Latest Dispatch Date.
- `order_priority` field is set to **Urgent**.
- Document is **NOT saved yet** — the Save button should be highlighted/dirty.

9. Save the document.

**Expected result:**
- Document saves without error.
- Urgent banner appears at the top of the form (colored band showing dispatch date and days remaining).

---

## Scenario 2 — Custom Order, Borderline Verdict (Warn Only)

**Purpose:** Confirm Borderline shows a warning but still allows the CS user to proceed after confirming with factory.

**Setup:** Open any Lyfe Order or create a test one with `order_type = Custom`.

**Steps:**

1. Open the Lyfe Order. Confirm `order_type = Custom` (visible in the form header area).
2. Click **Urgent → Calculate Dispatch Dates**.
3. Dialog opens. Order Type is pre-filled to Custom.
4. Enter a Required Delivery Date that leaves exactly **1–2 working days of buffer** after the Custom pipeline (14+4+3+7+7 calendar customs+2 = approximately 29–31 working days total plus customs). Try entering a date **31 working days from today**.
5. Click **Calculate**.

**Expected result:**
- Verdict badge is **amber "Borderline — Tight Timeline"**.
- Badge shows buffer count on the right side: "Buffer: 1 working day(s) — confirm with factory before applying."
- Warning message below the table: "This is achievable but very tight. Confirm with the factory before committing to the customer. Use the button below only after verbal confirmation."
- Primary button is **"Confirm & Apply"** (not blocked — warn-only behaviour).
- "Skip for now" is available.

6. Click **Skip for now**.

**Expected result:**
- Dialog closes. No fields changed. No save triggered.

7. Re-open the dialog. This time click **Confirm & Apply**.

**Expected result:**
- Same as Scenario 1 step 8 — fields stamped, toast shown, form dirty.

---

## Scenario 3 — Not Possible Verdict

**Purpose:** Confirm that Not Possible shows an escalation message and blocks the Apply button.

**Steps:**

1. Open any Lyfe Order (any order type).
2. Click **Urgent → Calculate Dispatch Dates**.
3. Enter a Required Delivery Date **5 calendar days from today** (entirely impossible for any order type).
4. Click **Calculate**.

**Expected result:**
- Verdict badge is **red "Not Possible"**.
- Red message below: "This date is not achievable. Please escalate to the founders or offer the customer a revised delivery date."
- Primary button is **"Recalculate"** — NOT "Confirm & Apply". There is no Apply button.
- "Skip for now" is available.

5. Change the date to something achievable and click **Recalculate**.

**Expected result:**
- Result updates to Possible or Borderline and the Apply button re-appears.

---

## Scenario 4 — Priority Dropdown → Urgent (Auto-Dialog Trigger)

**Purpose:** Confirm that changing `order_priority` to Urgent directly on the form auto-opens the dispatch calculator.

**Setup:** Any Lyfe Order where `promised_dispatch_by` is empty and `order_priority` is currently Normal.

**Steps:**

1. Open the Lyfe Order.
2. Find the `order_priority` field (labelled "Order Priority") — it shows "Normal".
3. Change the value to **Urgent** using the dropdown.

**Expected result:**
- A `frappe.confirm` dialog fires: "Set this order as Urgent? This will open the dispatch calculator to set a promised dispatch date."
- Two buttons: **Yes, set Urgent** and **No, keep current priority**.

4. Click **Yes, set Urgent**.

**Expected result:**
- Dispatch Calculator dialog opens automatically.
- The dialog pre-fills `order_type` from the current doc.

5. Enter a valid delivery date and click Calculate, then Confirm & Apply.

**Expected result:**
- `promised_dispatch_by` is stamped.
- `order_priority` remains Urgent.
- Form is dirty, save to confirm.

**Additional check — Already has promised_dispatch_by:**

6. On a different order that already has `promised_dispatch_by` filled, change priority to Urgent.

**Expected result:**
- Dialog does NOT auto-open (the date is already set).
- Only a green alert shows: "Order flagged as Urgent."

---

## Scenario 5 — Quotation custom_priority = Urgent → Lyfe Order Sync

**Purpose:** Confirm that setting priority on a Quotation syncs to the Lyfe Order when the order is created.

**Setup:** Open any Quotation in "Submitted" or an editable state. For the cleanest test, use one that hasn't had a Lyfe Order created yet.

**Steps:**

1. Open Quotation. Check that the `custom_order_type` field is set (Standard or Custom) — a priority gate requires this.
2. Find the `custom_priority` field (labelled "Priority", visible in the form near the order type field).
3. If `custom_order_type` is empty, set it first (e.g., Standard). Save.
4. Set `custom_priority` to **Urgent**.

**Expected result:**
- An amber banner appears at the top of the Quotation form: "URGENT — [inline 'Calculate Dispatch Dates' link]".
- No error on save.

5. Also set `custom_estimated_delivery_date` to a date 30+ working days from today.
6. Save the Quotation.
7. Open the dispatch calculator via **Urgent → Calculate Dispatch Dates** from the Quotation toolbar.

**Expected result:**
- Dialog auto-runs the calculation immediately using the pre-filled delivery date.
- Result shows immediately (no need to click Calculate).

8. If this is a new Quotation flow, progress it to the state where a Lyfe Order gets created (Submit / workflow action that triggers order creation). Then open the resulting Lyfe Order.

**Expected result:**
- Lyfe Order `order_priority` = **Urgent** (synced from Quotation).
- `order_type` matches the Quotation's `custom_order_type`.
- If `promised_dispatch_by` was already set on the Quotation form, it carries over.

---

## Scenario 6 — Downgrade from Urgent

**Purpose:** Confirm that removing Urgent priority asks for confirmation and optionally clears the dispatch date.

**Setup:** Any Lyfe Order currently set to Urgent with `promised_dispatch_by` filled.

**Steps:**

1. Open the Urgent Lyfe Order.
2. Change `order_priority` from Urgent to **Normal** (or High).

**Expected result:**
- `frappe.confirm` fires: "Remove Urgent flag? This will clear the promised dispatch date."
- Two buttons: **Yes, remove** and **Cancel**.

3. Click **Cancel**.

**Expected result:**
- Priority reverts to Urgent.
- `promised_dispatch_by` unchanged.
- No save triggered.

4. Change priority to Normal again and this time click **Yes, remove**.

**Expected result:**
- `promised_dispatch_by` is cleared.
- `order_priority` stays at Normal.
- Form is dirty. Save.
- Urgent banner disappears from the form.

---

## Scenario 7 — SLA Detector Exclusion (Urgent Orders Not in Breach Count)

**Purpose:** Confirm that Urgent orders are excluded from all SLA violation detectors.

**Steps:**

1. Open any Urgent Lyfe Order. Note its name (e.g., LYF-MN-2026-XXXX).
2. Open the ERP console or ask a developer to run the Promised Dispatch detector manually:

```python
import frappe
frappe.init(site="lyfe.local.local", sites_path="/home/frappe/frappe-bench/sites")
frappe.connect()
from lh.lh_project.sla.detectors import promised_dispatch
violations = promised_dispatch.detect()
names = [v["erp_docname"] for v in violations]
print("LYF-MN-2026-XXXX in violations:", "LYF-MN-2026-XXXX" in names)
frappe.destroy()
```

**Expected result:**
- The Urgent order does NOT appear in the `violations` list.

3. Repeat for other active detectors (Stuck Order, Awaiting Shipping, Approval, etc.) — same expectation.

**UI verification (no console):**
- Open the SLA Violation Cache list. Filter by ERP Document = your Urgent order name.
- Expect: no rows, or all rows showing `is_resolved = 1`.

---

## Scenario 8 — Quotation Form: "Calculate Dispatch Dates" Button Present

**Purpose:** Confirm the calculator button is wired up on the Quotation form, not just Lyfe Order.

**Steps:**

1. Open any Quotation.
2. Look for the **Urgent** button group in the toolbar.
3. Click **Urgent → Calculate Dispatch Dates**.

**Expected result:**
- Dialog opens with `order_type` pre-filled from `custom_order_type`.
- On this form, "Apply" sets `custom_priority` = Urgent (not `order_priority` — that field is on Lyfe Order).
- After Apply, the banner on the Quotation form shows the priority badge.

---

## Scenario 9 — Urgent Banner States

**Purpose:** Verify the banner on Lyfe Order form correctly transitions between states.

**Setup:** An Urgent Lyfe Order.

**Expected states to test:**

| `promised_dispatch_by` vs. today | Banner text / colour |
|---|---|
| More than 2 working days away | Blue/info banner — shows date, days remaining |
| Within 2 working days (AT RISK) | Orange banner — "AT RISK" label, days remaining |
| Past (overdue) | Red banner — "OVERDUE" label |

**Steps:**

1. Open an Urgent order. Note the banner colour and label.
2. To test AT RISK: temporarily set `promised_dispatch_by` to today+1 (directly in the field) and save. Reload. Banner should turn orange with "AT RISK".
3. To test OVERDUE: set `promised_dispatch_by` to yesterday and save. Reload. Banner should be red with "OVERDUE".
4. Restore the date to the correct value.

---

## Scenario 10 — Quotation priority gate (order type required)

**Purpose:** Confirm that setting a non-Normal priority on a Quotation without an order type is blocked.

**Steps:**

1. Open a Quotation where `custom_order_type` is empty.
2. Set `custom_priority` to High.
3. Attempt to Save.

**Expected result:**
- Frappe throws an error: "Order Type must be set before saving a High priority quotation."
- Save is blocked.

4. Set `custom_order_type` to Standard, then save again.

**Expected result:**
- Saves successfully.

---

## Edge Cases

### EC-1 — Sunday Delivery Date Input

**Steps:**
1. Open the dispatch calculator.
2. Enter a Required Delivery Date that falls on a **Sunday**.
3. Calculate.

**Expected result:**
- Calculation succeeds (Sunday as an input is valid — the calculator works backwards from it using calendar math).
- The Latest Dispatch Date returned is a working day (Mon–Sat); if it falls on a Sunday the calculator snaps it to the previous Saturday.

---

### EC-2 — Past or Today Delivery Date

**Steps:**
1. Open the dispatch calculator.
2. Enter yesterday's date as the Required Delivery Date.
3. Calculate.

**Expected result:**
- Verdict is **Not Possible** (red).
- No Apply button.

---

### EC-3 — Missing custom_estimated_delivery_date on Quotation

**Steps:**
1. Open a Quotation where `custom_estimated_delivery_date` is empty.
2. Set `custom_priority` to Urgent.
3. Open the calculator via the toolbar button.

**Expected result:**
- Dialog opens WITHOUT auto-running the calculation (no delivery date to pre-fill).
- "Customer's Required Delivery Date" field is empty and required.
- User must fill it manually before clicking Calculate.

---

### EC-4 — Via US Warehouse + Customs Path

**Steps:**
1. Open the dispatch calculator on a Standard order.
2. Check **Route via US Warehouse** = on, **Include Customs** = on.
3. Enter a delivery date 50+ working days out.
4. Calculate.

**Expected result:**
- Pipeline breakdown shows "US Warehouse Leg: 4 wd" row.
- Latest Dispatch Date is earlier than the non-US-warehouse result (more pipeline segments consumed).
- Still shows Possible with runway.

---

### EC-5 — VIP Priority (No Auto-Dialog)

**Steps:**
1. Open a Lyfe Order.
2. Change `order_priority` to **VIP**.

**Expected result:**
- NO dispatch calculator dialog auto-opens (VIP is a priority tier, not the same as Urgent — the auto-dialog only fires for Urgent).
- The priority change is accepted without prompting the calculator.
- If the Quotation form has a `custom_priority = VIP`, the banner shows purple "VIP" badge.

---

### EC-6 — Direct Lyfe Order Priority Change Without Quotation Link

**Steps:**
1. Open a Lyfe Order that has NO linked Quotation (`source_quotation` is empty).
2. Change `order_priority` to Urgent.
3. Confirm the dialog. Calculator opens.
4. Enter a delivery date and Apply.

**Expected result:**
- Works identically to Scenario 4.
- `promised_dispatch_by` is stamped from the calculator result.
- No error about missing quotation (the sync path only runs if a quotation exists; priority change path is independent).

---

### EC-7 — Borderline + Confirm + Immediate Save

**Steps:**
1. Get a Borderline result in the calculator.
2. Click Confirm & Apply.
3. Immediately click Save (without doing anything else).

**Expected result:**
- Document saves cleanly.
- `order_priority = Urgent` and `promised_dispatch_by` both persisted to DB.
- Server-side `_handle_priority_change` fires via `on_update` and the urgent PM task is created in the project (check the Tasks linked to this order).

---

### EC-8 — Double-click Calculate (Rapid Re-fire)

**Steps:**
1. Open the calculator.
2. Enter a delivery date and click Calculate rapidly twice before the spinner disappears.

**Expected result:**
- No crash, no duplicate results. The second click re-runs the calculation, replacing the first result.
- Spinner appears and result renders once.

---

### EC-9 — Recalculate After Apply

**Steps:**
1. Apply a Possible result to a Lyfe Order (do not save yet).
2. Re-open the calculator from the toolbar.
3. Change the delivery date and recalculate.
4. Apply the new result.

**Expected result:**
- `promised_dispatch_by` is overwritten with the new Latest Dispatch Date.
- No stale data from the first apply.

---

## Pass Criteria Summary

| # | Scenario | Pass When |
|---|---|---|
| 1 | Standard / Possible | Apply stamps fields, banner appears after save |
| 2 | Custom / Borderline | Warn shown, user can still apply |
| 3 | Not Possible | No Apply button, escalation message shown |
| 4 | Dropdown → Urgent | Auto-opens dialog if `promised_dispatch_by` empty |
| 5 | Quotation sync | Lyfe Order inherits priority and order type from Quotation |
| 6 | Downgrade from Urgent | Confirm dialog, clears date on Yes |
| 7 | SLA exclusion | Urgent order absent from all detector violation lists |
| 8 | Quotation button | Calculator accessible on Quotation form |
| 9 | Banner states | Blue / Orange AT RISK / Red OVERDUE at correct thresholds |
| 10 | Priority gate | Save blocked if order type missing on non-Normal priority Quotation |
| EC-1 | Sunday delivery date | Calculation succeeds, dispatch date is a working day |
| EC-2 | Past delivery date | Not Possible verdict |
| EC-3 | Missing delivery date on Quotation | Dialog opens without auto-running |
| EC-4 | US warehouse path | US Leg row visible in breakdown |
| EC-5 | VIP priority | No auto-dialog |
| EC-6 | No linked Quotation | Priority change + calculator works independently |
| EC-7 | Borderline → Save | PM task created in project after save |
| EC-8 | Rapid double-click | No crash, single result |
| EC-9 | Recalculate after apply | Date overwritten correctly |

---

## After Testing

If all scenarios pass:
1. Remove the viralk@lyfehardware.com-only restriction from `urgent_notification_users` and add the real CS team.
2. Set `urgent_slack_channel` to the production Slack channel.
3. Run `bench build --app lh` on the production server before deploying.
4. Announce the feature to the CS team: Urgent orders are now tracked end-to-end with automatic PM tasks and SLA exclusion.
