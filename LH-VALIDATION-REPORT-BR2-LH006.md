# End-to-End Validation Report — BR-2 & LH-DEV-006

**Doc ID:** LH-VALIDATION-REPORT-BR2-LH006  
**Date:** 2026-06-18  
**Prepared by:** Claude Code (automated analysis)  
**Reviewed against:** LH-DEV-ANALYSIS-BR2, LH-DEV-ANALYSIS-006, LH-DEV-ANALYSIS-005, LH-UI-TESTING-BR2-LH006  

---

## Executive Summary

The BR-2 Reverse SLA Dispatch Calculator and LH-DEV-006 Quotation Priority/Order Type Upstream features are **substantially implemented**. The core pipeline logic, API layer, client dialog, and priority propagation are all present and correctly structured. However, **7 bugs and 8 gaps** were identified that range from critical production blockers to low-risk deferred items.

**Overall verdict: CONDITIONAL GO — 4 items must be fixed before production.**

---

## Phase 1: Requirement Verification Matrix

### BR-2 Requirements

| Requirement | Source | Implemented | Tested (per docs) | Risk |
|---|---|---|---|---|
| `working_days.py` utility — Mon–Sat schedule | BR-2 §8.1 | ✅ Yes — `/lh_project/sla/utils/working_days.py` | Unit-testable | Low |
| `dispatch_calculator.py` — pure reverse SLA | BR-2 §8.2 | ✅ Yes — `/lh_project/sla/utils/dispatch_calculator.py` | Unit-testable | Low |
| Whitelisted API endpoint | BR-2 §8.3 | ✅ Yes — `/lh_project/api/dispatch_calc.py` | Covered by SC1–SC3 | Low |
| Client dialog — Lyfe Order form | BR-2 §8.4 | ✅ Yes — `dispatch_calc_dialog.js` + `lyfe_order.js` | SC1–SC9, EC1–EC9 | Low |
| Client dialog — Quotation form | BR-2 §8.4 (Q5) | ✅ Yes — `quotation.js` line 138–143 | SC5, SC8 | Low |
| "Confirm & Apply" stamps `promised_dispatch_by` | BR-2 §8.4 | ✅ Yes — `_apply_result()` in dialog JS | SC1 | Low |
| `on_update` Urgent flag handler — create PM task | BR-2 §8.5 | ⚠️ Partial — calls `_try_create_task_for_rule` which silently passes on `ImportError`/`AttributeError` | EC-7 | **High** |
| Urgent downgrade — close PM task + clear date | BR-2 §8.5 | ✅ Yes — `_on_downgrade_from_urgent()` | SC6 | Low |
| `promised_dispatch_by` type: Data → Date | BR-2 §8.8 | ✅ Yes — field exists as `Datetime` in JSON | N/A | Low |
| SLA exclusion clause for Urgent orders | BR-2 §8.6 | ✅ Yes — `promised_dispatch.py` line 165 | SC7 | Low |
| "At Risk" daily re-notification scan | BR-2 §8.7 | ❌ NOT IMPLEMENTED — no scheduler entry | Not in test guide | **High** |
| Holiday calendar support | BR-2 §8.1 (Q2) | ❌ NOT IMPLEMENTED — Sundays only | Not tested | Medium |
| Auto-open dialog when priority → Urgent (no date set) | BR-2 §8.4 | ✅ Yes — `lyfe_order.js` line 2562 | SC4 | Low |
| Auto-run calculation if `custom_estimated_delivery_date` pre-filled | BR-2 §8.4 | ✅ Yes — dialog JS line 81–88 | SC5 | Low |
| Urgent banner — AT RISK / OVERDUE states | BR-2 §4.1 | ⚠️ Partial — client-side calendar days only (BUG-01) | SC9 | **High** |
| Escalate to Founders notification (Not Possible) | BR-2 §3 Scenario B | ❌ NOT IMPLEMENTED — no notification sent | Not tested | Medium |

### LH-DEV-006 Requirements

| Requirement | Source | Implemented | Tested | Risk |
|---|---|---|---|---|
| `custom_priority` field on Quotation | LH-006 §3 | ✅ Yes — present in quotation.js, doc_events | SC5, SC10 | Low |
| `custom_order_type` field on Quotation | LH-006 §3 | ✅ Yes — auto-set from items | SC10 | Low |
| Priority gate: non-Normal priority requires order type | LH-006 §5 | ✅ Yes — `quotation.py` line 519–527 | SC10 | Low |
| Sync priority Quotation → Lyfe Order on creation | LH-006 §5 | ✅ Yes — `after_insert` → `_sync_fields_from_quotation()` | SC5 | Low |
| Sync order_type Quotation → Lyfe Order | LH-006 §5 | ✅ Yes — same path | SC5 | Low |
| Quotation priority wins over ShipStation `is_custom_order` | LH-006 §5 (Caveat 2) | ✅ Yes — `db_set` after ShipStation sets `order_type` | SC5 | Low |
| Post-conversion edit: notify only, don't auto-overwrite | LH-006 §5 (Caveat 3) | ❌ NOT IMPLEMENTED — no post-conversion sync path | Not tested | Medium |
| Priority vocabulary aligned (Normal/High/Urgent/VIP) | LH-006 §4 | ⚠️ Mismatch — Lyfe Order has "Normal/Urgent/VIP", Quotation has "Normal/High/Urgent/VIP" — "High" maps nowhere on Lyfe Order | SC5 | **Critical** |

### LH-DEV-005 (Dashboard) Requirements

| Requirement | Source | Implemented | Tested | Risk |
|---|---|---|---|---|
| Exclude Urgent orders from standard SLA breach count | LH-005 §3 | ⚠️ Partial — `promised_dispatch.py` excludes them (SLA task scanner), but `order_analysis.py` `breach_promised_dispatch` SQL has NO Urgent exclusion | SC7 | **Critical** |
| Dedicated Urgent SLA metric group on dashboard | LH-005 §3 | ❌ NOT IMPLEMENTED — no Urgent-specific widget | Not tested | High |
| Priority view on Status Overview (sort, dispatch-by column) | LH-005 §4 | ❌ NOT IMPLEMENTED | Not tested | Medium |
| Bottleneck/aging analysis per stage | LH-005 §5 | Pre-existing — not part of this release | N/A | Low |

---

## Phase 2: Bug Analysis Report

### BUG-01 — CRITICAL: Dashboard SLA Breach Count Includes Urgent Orders

**Category:** Critical  
**File:** `apps/lh/lh/lyfe_hardware/page/order_analysis/order_analysis.py` lines 269–325  

**Description:** The three `breach_promised_dispatch`, `breach_promised_dispatch_standard`, `breach_promised_dispatch_custom` SQL queries in `order_analysis.py` do NOT filter out `order_priority = 'Urgent'`. When an Urgent order runs on a compressed 10-day timeline, it will appear in the SLA breach counts after day 3 (standard) or day 14 (custom), even though it is intentionally running under a different SLA clock.

**This is the exact corruption scenario described in LH-DEV-ANALYSIS-005 §Critical Warning.**

**Reproduction:**
1. Create an Urgent Lyfe Order with `order_priority = Urgent` and `factory_first_action` set.
2. Let 4+ working days pass (or use a test order with a past `factory_first_action`).
3. Open Order Analysis dashboard → SLA breach counts will include this Urgent order.

**Impact:** False SLA breach inflation. Every Urgent order inflates standard breach KPIs, making dashboards unreliable for management decisions.

**Fix:**
```sql
-- Add to all three breach_promised_dispatch queries:
AND (order_priority IS NULL OR order_priority != 'Urgent')
```

---

### BUG-02 — HIGH: Priority Vocabulary Mismatch — "High" Lost on Sync

**Category:** High  
**File:** `apps/lh/lh/lyfe_hardware/doctype/lyfe_order/lyfe_order.py` line 2912  

**Description:** Quotation `custom_priority` supports four values: Normal / High / Urgent / VIP. Lyfe Order `order_priority` only supports: Normal / Urgent / VIP. The sync in `_sync_fields_from_quotation()` maps `High` from Quotation to `order_priority = "High"` on Lyfe Order, but there is no "High" option on that field.

```python
if q_values.custom_priority and q_values.custom_priority != "Normal":
    updates["order_priority"] = q_values.custom_priority  # "High" → invalid!
```

**Impact:** If a Quotation is set to `custom_priority = High` and a Lyfe Order is created, the `order_priority` is set to "High" which is not a valid option. This may silently fail (set to None or empty) or appear as an unstyled value on the form. The order loses its priority classification.

**Fix:** Either:
- Remove "High" from Quotation's `custom_priority` options and map the vocabulary per LH-DEV-006 §4, OR
- Add "High" to Lyfe Order's `order_priority` Select field options, OR
- Map "High" → "Urgent" in the sync function:
  ```python
  priority_map = {"High": "Urgent", "Urgent": "Urgent", "VIP": "VIP"}
  mapped = priority_map.get(q_values.custom_priority)
  if mapped:
      updates["order_priority"] = mapped
  ```

---

### BUG-03 — HIGH: AT RISK Banner Uses Calendar Days, Not Working Days

**Category:** High  
**File:** `apps/lh/lh/lyfe_hardware/doctype/lyfe_order/lyfe_order.js` lines 2613–2627  

**Description:** The AT RISK threshold check uses raw calendar milliseconds:

```js
const days_left_ms = new Date(frm.doc.promised_dispatch_by) - new Date(today);
const days_left = Math.ceil(days_left_ms / (1000 * 60 * 60 * 24));
if (days_left <= 0)  → OVERDUE
if (days_left <= 2)  → AT RISK
```

The spec and `working_days.py` define AT RISK as **≤ 2 working days** (`BORDERLINE_WD = 2`). Calendar days vs. working days produces the wrong threshold for dispatch dates that are near a Sunday.

**Example:** If today is Friday and `promised_dispatch_by` is Tuesday (2 working days away), calendar distance = 4 days → no AT RISK shown, but the order IS at risk per the spec.

**Impact:** Missing AT RISK warnings, factory not alerted on time.

**Fix:** Replace the calendar-day calculation with a working-day calculation. The `working_days_remaining` logic needs to be replicated client-side or called via API. Simplest fix: calculate `days_left` excluding Sundays:

```js
function working_days_remaining_client(target_str) {
    let today = new Date();
    today.setHours(0,0,0,0);
    let target = new Date(target_str);
    target.setHours(0,0,0,0);
    let step = today <= target ? 1 : -1;
    let count = 0;
    let cur = new Date(today);
    while ((step === 1 && cur <= target) || (step === -1 && cur >= target)) {
        if (cur.getDay() !== 0) count += step; // skip Sundays
        cur.setDate(cur.getDate() + step);
    }
    return count;
}
```

---

### BUG-04 — HIGH: PM Task Creation Silently Fails — `_try_create_task_for_rule` Not Defined

**Category:** High  
**File:** `apps/lh/lh/lyfe_hardware/doctype/lyfe_order/lyfe_order.py` lines 2994–3004  

**Description:**

```python
def _create_urgent_tracking_task(doc):
    try:
        from lh.lh_project.automation.lyfe_order import _try_create_task_for_rule
        _try_create_task_for_rule(doc, event="on_priority_urgent")
    except (ImportError, AttributeError):
        pass   # ← silently swallowed
    except Exception:
        frappe.log_error(...)
```

The function `_try_create_task_for_rule` does not exist in `lh_project/automation/lyfe_order.py` (confirmed by grep returning no results). The `except (ImportError, AttributeError): pass` swallows the failure silently. No PM task is created when an order is flagged Urgent.

**Impact:** The BR-2 spec states: *"The system automatically creates a tracking task in the project board so nothing falls through the cracks."* This core automation does not work. Managers cannot track Urgent orders from the PM board.

**Fix:** Either implement `_try_create_task_for_rule` in the automation module, or fall back to direct Task creation in `_create_urgent_tracking_task`. A direct fallback:

```python
def _create_urgent_tracking_task(doc):
    try:
        from lh.lh_project.automation.lyfe_order import _try_create_task_for_rule
        _try_create_task_for_rule(doc, event="on_priority_urgent")
        return
    except (ImportError, AttributeError):
        pass  # No automation rule — fall through to direct creation
    except Exception:
        frappe.log_error(frappe.get_traceback(), f"Urgent task via automation failed ({doc.name})")

    # Direct fallback: create Task in the first available project
    try:
        project = frappe.db.get_value("Project", {"status": "Open"}, "name")
        if not project:
            return
        dispatch_date = str(doc.promised_dispatch_by) if doc.promised_dispatch_by else "TBD"
        task = frappe.get_doc({
            "doctype": "Task",
            "subject": f"URGENT — Dispatch by {dispatch_date} — {doc.name}",
            "project": project,
            "priority": "Urgent",
            "exp_end_date": doc.promised_dispatch_by,
            "description": f"Urgent order {doc.name}. Promised dispatch: {dispatch_date}.",
            "status": "Open",
        })
        task.insert(ignore_permissions=True)
    except Exception:
        frappe.log_error(frappe.get_traceback(), f"Urgent task direct creation failed ({doc.name})")
```

---

### BUG-05 — MEDIUM: `_apply_result()` Sets `order_priority` on Quotation (Wrong Field)

**Category:** Medium  
**File:** `apps/lh/lh/public/js/toolkit/dispatch_calc_dialog.js` lines 234–255  

**Description:**

```js
function _apply_result(dialog, result, frm, opts) {
    const priority = frm.doc.order_priority || "Urgent";
    let chain = frm.set_value("promised_dispatch_by", result.latest_dispatch_date)
        .then(() => frm.set_value("order_priority", priority === "VIP" ? "VIP" : "Urgent"));
    if (frm.doctype === "Quotation") {
        chain = chain.then(() => frm.set_value("custom_priority", ...));
    }
```

On a **Quotation** form, `order_priority` does not exist as a form field — it attempts to `set_value("order_priority", ...)` which will silently fail or error. The intent is that Quotation uses `custom_priority`, and `promised_dispatch_by` also does not exist on Quotation.

**Impact:** On the Quotation form, "Confirm & Apply" tries to set two non-existent fields before setting the correct `custom_priority`. This may produce console errors or a partial apply (priority set but `promised_dispatch_by` chain fails first).

**Fix:** Gate the `order_priority` and `promised_dispatch_by` assignments by doctype:

```js
function _apply_result(dialog, result, frm, opts) {
    dialog.hide();
    let chain = Promise.resolve();

    if (frm.doctype !== "Quotation") {
        const priority = frm.doc.order_priority || "Urgent";
        chain = chain
            .then(() => frm.set_value("promised_dispatch_by", result.latest_dispatch_date))
            .then(() => frm.set_value("order_priority", priority === "VIP" ? "VIP" : "Urgent"));
    } else {
        const priority = frm.doc.custom_priority || "Urgent";
        chain = chain
            .then(() => frm.set_value("custom_estimated_delivery_date", 
                dialog.get_value("required_delivery_date")))
            .then(() => frm.set_value("custom_priority", priority === "VIP" ? "VIP" : "Urgent"));
    }

    chain.then(() => {
        if (opts.on_apply) opts.on_apply(result);
        return frm.save();
    }).then(() => frappe.show_alert({ message: __("Dispatch dates saved."), indicator: "green" }));
}
```

---

### BUG-06 — MEDIUM: `verdict_for_order()` in `dispatch_calculator.py` Has Double Customs Addition

**Category:** Medium  
**File:** `apps/lh/lh/lh_project/sla/utils/dispatch_calculator.py` lines 221–237  

**Description:** The `verdict_for_order()` function re-derives an "expected delivery date" from a stored `promised_dispatch_by` by adding all post-dispatch stages forward. It then calls `calculate()` using that derived delivery date, which subtracts those same stages backward again. The customs buffer is added twice — once forward in `verdict_for_order()` and once backward in `calculate()` — producing a delivery date 14 calendar days further out than the original.

Similarly, the API endpoint `get_dispatch_window_for_order()` in `dispatch_calc.py` performs the same forward-projection and has the same duplication issue.

**Impact:** When a CS user clicks "Calculate Dispatch Dates" on a Lyfe Order that already has `promised_dispatch_by` set, the calculated window will be off (the delivery date is over-estimated). The verdict may show "Possible" with excess runway when the real window is tighter.

**Reproduction:** Set `promised_dispatch_by = today + 20 WD` on a Standard Lyfe Order. Click Calculate Dispatch Dates. The returned `latest_dispatch_date` will differ from the stored value.

**Fix:** The `verdict_for_order()` helper should simply call `calculate()` with the stored dispatch date used directly as a proxy delivery date (i.e., skip the forward projection). Or accept an explicit `required_delivery_date` parameter instead of deriving it.

---

### BUG-07 — LOW: Toast Message Mismatch with UI Testing Guide (SC1)

**Category:** Low  
**File:** `apps/lh/lh/public/js/toolkit/dispatch_calc_dialog.js` line 251  

**Description:** The `_apply_result()` function shows the toast: `"Dispatch dates saved."` The UI Testing Guide (SC1) expects: `"Dispatch dates applied. Save the document to confirm."` — the dialog also saves immediately (`frm.save()` is called in the same chain), so the toast should say "saved" not "applied", but the testing guide will fail this check. More importantly, `frm.save()` fires before the toast, meaning if save fails, the toast may still show "saved" incorrectly.

**Fix:** Minor — update the guide or the message. More importantly, move the `frappe.show_alert` to inside a `.then()` after `frm.save()` resolves (it already is, but the message should say "Dispatch date applied and saved." to match real behaviour).

---

## Phase 3: Gap Analysis Report

### GAP-01 — CRITICAL: Dashboard `order_analysis.py` Has No Urgent Exclusion

Already described in BUG-01. This is the most operationally critical gap.  
**Required fix before go-live.** See BUG-01.

### GAP-02 — HIGH: "At Risk" Server-Side Notification Scanner Not Built

**Source:** BR-2 §8.7, §4.2 (notification table)  
**What's missing:** No scheduled job scans for Urgent orders with `promised_dispatch_by ≤ 2 working days` from today that haven't dispatched. The client-side banner shows AT RISK state visually, but only when a CS user is looking at that specific order. The spec requires **daily push notifications** to CS owner + factory + manager.

**Impact:** Managers and factory may not be alerted when an Urgent order is at risk. The PM board task due date is set correctly, so PM board users will see overdue tasks, but email/Slack alerts are absent.

**Required for production?** Yes — per spec. No daily alert = no proactive management.

### GAP-03 — HIGH: "Not Possible" Founder Escalation Not Implemented

**Source:** BR-2 §3 (Scenario B), §4.2  
**What's missing:** When the dialog shows "Not Possible", the spec says CS should see an "Escalate to Founders" button that sends a Slack/email notification. The current implementation shows only a text message and "Skip for now". There is no notification sent, no record created.

**Impact:** Founder visibility into infeasible orders is entirely manual. Orders requiring special exceptions may be accepted without founder awareness.

### GAP-04 — MEDIUM: Post-Conversion Priority Sync Not Implemented

**Source:** LH-DEV-006 §5 Caveat 3  
**What's missing:** If `custom_priority` on a Quotation is changed **after** the Lyfe Order has already been created, the change does not propagate. The `_sync_fields_from_quotation()` only fires on `after_insert`. The spec says: *"notify only, don't auto-overwrite."* Neither the notification nor the sync happens.

**Impact:** Priority changes at the quote stage post-conversion are silently lost. The Lyfe Order retains the old priority.

### GAP-05 — MEDIUM: Holiday Calendar Not Supported

**Source:** BR-2 §8.1 (Open Question Q2)  
**What's missing:** `working_days.py` only excludes Sundays. The spec and BRD require Indian public holidays (and US holidays for US warehouse path) to be excluded. The document notes this was an open question — but the implementation skipped it entirely.

**Impact:** Dispatch date calculations may be 1–3 days optimistic around major Indian holidays (Diwali, Holi, etc.), causing committed dates to be missed when the factory is actually closed.

### GAP-06 — MEDIUM: Dedicated Urgent Metrics Widget Not Added to Dashboard

**Source:** LH-DEV-005 §3 (Fix Plan B)  
**What's missing:** The dashboard does not have a dedicated "Urgent SLA" metric group showing urgent-orders-at-risk count, urgent dispatch compliance rate, or urgent orders overdue.

**Impact:** Management cannot see Urgent-specific compliance at a glance. After go-live, as more orders are flagged Urgent, the lack of dedicated metrics will become operationally painful.

### GAP-07 — LOW: Priority View Column Not Added to Status Overview Dashboard

**Source:** LH-DEV-005 §4  
**What's missing:** No `dispatch_by` column or priority-sort on the Status Overview list. The `pm_operations_dashboard.py` has Urgent overdue counts from PM tasks but not the Lyfe Order dispatch view described in LH-005.

### GAP-08 — LOW: `custom_estimated_delivery_date` Not Propagated to Lyfe Order

**Source:** LH-DEV-006 implicit, BR-2 §8.5  
**What's missing:** When a Quotation has `custom_estimated_delivery_date` set and a Lyfe Order is created, that field is not copied to the Lyfe Order. The `_on_upgrade_to_urgent()` function reads it from the Quotation via a live DB query, which works. But the Lyfe Order has no field to store it permanently — if the Quotation record is edited later, the dispatch date derived at order creation cannot be audited.

---

## Phase 4: Backtesting Report

### SLA Calculation — Manual Verification

The `dispatch_calculator.py` uses the following pessimistic buffer values:

| Stage | Buffer (pessimistic) | BRD Spec | Match? |
|---|---|---|---|
| Production (Standard) | 3 WD | 1–3 WD | ✅ Uses max |
| Production (Custom) | 14 WD | 7–14+ WD | ✅ Uses max |
| Photo Approval (Custom) | 4 WD | 1–4 WD | ✅ Uses max |
| Gate Pass | 3 WD | 1–3 WD | ✅ Uses max |
| India Transit | 7 WD | 5–7 WD | ✅ Uses max |
| Customs | 7 calendar days | 2–7 days | ✅ Uses max |
| US Leg | 4 WD | 3–4 WD | ✅ Uses max |
| Last-Mile Delivery | 2 WD | 1–2 WD | ✅ Uses max |

**Test: BRD Example — Standard, India, Delivery June 30, today = June 18**

Working backward from June 30:
- Minus 2 WD delivery → June 26 (Thu)
- Minus 7 calendar days customs → June 19 (Thu)
- Minus 7 WD India transit → June 10 (Wed)
- Minus 3 WD gate pass → June 5 (Fri)
- `latest_dispatch_date` = **June 5**

Buffer = working_days_between(June 18, June 5). June 5 is in the past → buffer = negative → **"Not Possible"**

✅ This matches expected logic. For a 30-working-day delivery window from today, the verdict should be Possible.

**Working days arithmetic — Sunday-only exclusion:**

The `is_working_day()` function returns `True` for Monday–Saturday and `False` only for Sunday. This aligns with factory operations confirmed in the CLAUDE.md notes and existing SLA SQL formula.

**Customs buffer — calendar vs. working day mixing:**

The calculator subtracts customs as `timedelta(days=7)` (calendar days), then continues in working days. This is architecturally correct but creates a subtle edge case: if a customs buffer period spans a Sunday, the effective working-day loss is less than if it didn't. This is acceptable and intentional (customs is calendar-day based per the spec).

### Priority Propagation Backtesting

**Quotation → Lyfe Order sync path:**

- `after_insert()` calls `_sync_fields_from_quotation(doc)`
- This reads `custom_priority` and `custom_order_type` from the Quotation via `frappe.db.get_value`
- If `custom_priority != "Normal"`, sets `order_priority = custom_priority`
- Uses `frappe.db.set_value(..., update_modified=False)` — bypasses `on_update`, which is intentional to avoid re-triggering hooks during `after_insert`

**Edge case — sync fires BEFORE `on_update` priority change handler:**

On `after_insert`, the sync calls `frappe.db.set_value` directly. This does NOT trigger `on_update`. Therefore `_handle_priority_change()` does NOT fire when priority is synced from Quotation during order creation. An Urgent Lyfe Order created from an Urgent Quotation will NOT automatically get a PM task created.

**This is a gap.** The priority change handler only fires when `on_update` detects a change in `order_priority` between saves. On `after_insert`, there is no prior save to compare against.

**Workaround currently in place:** None. The PM task must be created manually or by saving the order a second time after creation.

---

## Phase 5: Workflow & Automation Validation

| Trigger | Expected | Actual | Status |
|---|---|---|---|
| Priority → Urgent (manual edit, save) | PM task created, `promised_dispatch_by` auto-stamped (if no date) | `_handle_priority_change` fires, but `_create_urgent_tracking_task` silently fails | ❌ BUG-04 |
| Priority → Urgent (from dialog Apply + save) | Same as above | Same failure | ❌ BUG-04 |
| Priority synced from Quotation (after_insert) | PM task created on order creation | `after_insert` uses `db_set` bypassing `on_update` — no task created | ❌ GAP (related to BUG-04) |
| Priority → Urgent already has `promised_dispatch_by` | No auto-stamp, banner refreshes | ✅ Correct — `if not doc.promised_dispatch_by:` guard | ✅ |
| Priority → Normal (downgrade) | Confirm dialog, clear date, close task | ✅ `_on_downgrade_from_urgent` closes matching SLA Task Links | ✅ |
| Priority → VIP | No auto-dialog | ✅ JS checks `priority === "Urgent"` before opening dialog | ✅ |
| Quotation priority gate — High/Urgent/VIP without order_type | Save blocked | ✅ `quotation.py` line 519–527 `frappe.throw()` | ✅ |
| Auto-run calculator when `custom_estimated_delivery_date` set | Calculator auto-fires on dialog open | ✅ Dialog JS lines 81–88 | ✅ |

---

## Phase 6: Notification Validation

| Notification | Trigger | Implemented | Status |
|---|---|---|---|
| Urgent order created | Order flagged Urgent | Not implemented (no email/Slack) | ❌ |
| Not Possible escalation to founders | CS sees Not Possible in dialog | Not implemented | ❌ GAP-03 |
| AT RISK daily scan | ≤2 WD until dispatch, still in factory | No scheduler exists | ❌ GAP-02 |
| Downgrade notification to manager | Priority changed from Urgent | Not implemented | ❌ |
| Urgent task created on PM board | Only visible to PM board users | Only if PM task creation works (blocked by BUG-04) | ❌ |

---

## Phase 7: Regression Testing

### Changes That Could Affect Existing Functionality

| Change | Risk | Observation |
|---|---|---|
| `on_update` now calls `_handle_priority_change` on every save | Medium | Reads `before_save.order_priority` via `get_doc_before_save()` — correct pattern. But fires on EVERY save, not just priority changes. The comparison `self.order_priority != old_priority` gates correctly. | 
| `after_insert` calls `_sync_fields_from_quotation` | Low | Uses `db_set(update_modified=False)` — no hooks triggered, no modification timestamp change. Safe. |
| `dispatch_calc_dialog.js` included globally via `app_include_js` | Low | File is loaded on all pages. The function `window.lhShowDispatchCalcDialog` is only called from specific form buttons. No global side effects. |
| `working_days.py` imported by `dispatch_calculator.py` | None | Pure utility, no side effects. |
| SLA exclusion clause in `promised_dispatch.py` | Low | Added `AND (order_priority IS NULL OR order_priority != 'Urgent')`. Only changes detection scope — no existing Urgent orders would have been detected anyway since the feature is new. |
| Priority gate added to `quotation.py` validate | Medium | Could block saving existing Quotations where `custom_priority` is set but `custom_order_type` is somehow empty. **Check:** The auto-set code (`line 513`) runs BEFORE the priority gate (`line 517`), so `custom_order_type` will always be set before the gate runs. Safe for standard flow. |
| `_apply_result()` calls `frm.save()` automatically | Medium | The dialog saves the form without explicit user action. This could conflict with unsaved changes the user intended to review. No regression on existing forms, but a behaviour change the CS team needs to be aware of. |

---

## Phase 8: Security & Permission Testing

| Scenario | Expected | Implementation | Status |
|---|---|---|---|
| Factory role reads Urgent orders | Read only | Lyfe Order permissions: Factory = read, write, report | ✅ |
| Factory role cannot set `order_priority` to Urgent | Should be allowed (factory can update) | Factory role has write permission — technically can change priority | ⚠️ No restriction — per design or gap? |
| CS role can use dispatch calculator | Yes | CS has write on Lyfe Order | ✅ |
| API endpoint `get_dispatch_window` — requires login | Yes | `@frappe.whitelist()` requires active session by default | ✅ |
| `get_dispatch_window` — rejects past delivery dates | Yes | `dispatch_calc.py` line 43–44 `frappe.throw()` | ✅ |
| `get_dispatch_window_for_order` — validates order exists | Yes | `dispatch_calc.py` line 79 | ✅ |
| Quotation priority gate bypassed via API | Possible | The `frappe.throw()` is in `validate()` — runs on all saves including API | ✅ Server-side enforced |

---

## Phase 9: Data Integrity Validation

| Check | Status | Notes |
|---|---|---|
| `promised_dispatch_by` is a Datetime field in JSON | ✅ | Field exists in `lyfe_order.json` |
| `order_priority` sync uses `db_set(update_modified=False)` | ✅ | Correct — avoids spurious modified timestamps |
| `_on_downgrade_from_urgent()` closes SLA Task Links by text match | ⚠️ | Matches on `"urgent"` or `"priority"` in task subject — fragile. A task titled "Check Priority" would be incorrectly closed |
| No duplicate PM tasks on re-save | ⚠️ | `_create_urgent_tracking_task` has no idempotency guard — if the automation engine is fixed (BUG-04), re-saving an Urgent order could create duplicate tasks |
| `_sync_fields_from_quotation` idempotent | ✅ | Safe to run multiple times — `db_set` overwrites with same value |
| Downgrade clears `promised_dispatch_by` only via JS | ⚠️ | `frm.set_value("promised_dispatch_by", null)` only in JS `order_priority` field handler. No server-side enforcement. If priority is changed via API without going through the form, the date is not cleared |

---

## Phase 10: Failure Scenario Simulation

| Scenario | Behaviour | Risk |
|---|---|---|
| `dispatch_calculator.py` import fails | `calculate()` not called; `frappe.log_error()` catches and logs | ✅ Safe |
| `frappe.call` to `get_dispatch_window` times out | Dialog shows "Server error. Please try again." | ✅ Handled |
| `frm.save()` fails after Apply in dialog | Promise rejection — no alert shown, form remains dirty | ⚠️ Silent failure — user may not know save failed |
| `_sync_fields_from_quotation` called on Lyfe Order with deleted Quotation | `frappe.db.get_value` returns None → `if not q_values: return` | ✅ Safe |
| Two CS users simultaneously set same order to Urgent | Both trigger `_handle_priority_change` — both attempt PM task creation | ⚠️ Potential duplicate tasks (if BUG-04 is fixed without idempotency guard) |
| `db_set` on `promised_dispatch_by` during `after_insert` while order is being saved | Uses `update_modified=False` — safe | ✅ |

---

## Production Readiness Report

### Go / No-Go Assessment

| Item | Status |
|---|---|
| Core SLA calculation logic | ✅ Go |
| Client dialog — Lyfe Order | ✅ Go |
| Client dialog — Quotation | ✅ Go (BUG-05 is minor) |
| Priority propagation Quotation → Lyfe Order | ⚠️ Conditional — BUG-02 (High vocabulary mismatch) must be resolved |
| SLA detector exclusion | ✅ Go |
| Dashboard SLA breach count | ❌ No-Go — BUG-01 / GAP-01 must be fixed |
| PM task creation on Urgent flag | ❌ No-Go — BUG-04 (task never created) |
| AT RISK server notifications | ❌ No-Go (GAP-02) — required per spec |
| Not Possible escalation | ❌ No-Go (GAP-03) — required per spec |
| Urgent banner — AT RISK threshold | ⚠️ Conditional — BUG-03 (calendar vs. working days) |

**Overall: NO-GO until BUG-01, BUG-02, BUG-04, and BUG-03 are fixed.**

### Required Fixes Before Production

1. **[BUG-01] — CRITICAL** Add `AND (order_priority IS NULL OR order_priority != 'Urgent')` to all three `breach_promised_dispatch` queries in `order_analysis.py`.

2. **[BUG-02] — HIGH** Resolve priority vocabulary mismatch. Recommended: map "High" → "Urgent" in `_sync_fields_from_quotation()` per LH-DEV-006 §4.

3. **[BUG-03] — HIGH** Replace calendar-day AT RISK threshold in `lyfe_order.js` with working-day calculation.

4. **[BUG-04] — HIGH** Fix PM task creation — implement `_try_create_task_for_rule` or add a direct Task creation fallback. Add idempotency guard to prevent duplicate tasks.

### Recommended Fixes (Before Stable Production)

5. **[GAP-02]** Implement AT RISK daily scanner cron job.  
6. **[GAP-03]** Implement "Not Possible" escalation notification.  
7. **[BUG-05]** Fix `_apply_result()` to gate `promised_dispatch_by` and `order_priority` by doctype.  
8. **[BUG-06]** Fix double-customs issue in `verdict_for_order()`.

### Deferred (Post-Launch)

9. **[GAP-04]** Post-conversion priority sync with notification.  
10. **[GAP-05]** Holiday calendar integration.  
11. **[GAP-06]** Dedicated Urgent dashboard metrics widget.  
12. **[GAP-07]** Priority view column on Status Overview.  
13. **[GAP-08]** Persist `custom_estimated_delivery_date` on Lyfe Order.

---

## Final Section: UI Testing Checklist for Business Users

### Pre-Test Setup

- [ ] `bench build --app lh` completed on the server.
- [ ] `bench --site lyfe.local.local migrate` completed without errors.
- [ ] Access Settings → Urgent notifications set to `viralk@lyfehardware.com` only.

---

### Quotation Screen — 15 Test Cases

| # | What to Test | How | Expected Result |
|---|---|---|---|
| Q-01 | Priority field visible | Open any Quotation. Look for "Priority" field. | Field visible, default = Normal |
| Q-02 | Priority = High with no Order Type | Clear `custom_order_type`, set Priority = High, Save. | Error: "Order Type must be set before saving" |
| Q-03 | Priority = Urgent with Order Type set | Set Order Type = Standard, Priority = Urgent, Save. | Saves successfully. Amber banner appears. |
| Q-04 | Priority = VIP | Set Priority = VIP. Save. | Saves. Purple VIP badge shown in banner. |
| Q-05 | Priority = Normal | Set Priority = Normal. Banner check. | No urgent banner. |
| Q-06 | Dispatch calculator button visible | Open any Quotation. Look in toolbar under "Urgent" group. | "Calculate Dispatch Dates" button visible. |
| Q-07 | Calculator opens on Quotation | Click "Urgent → Calculate Dispatch Dates". | Dialog opens. Order Type pre-filled. |
| Q-08 | Auto-run when `custom_estimated_delivery_date` set | Set `custom_estimated_delivery_date` to 40 working days out, Priority = Urgent. Open dialog. | Dialog opens and auto-calculates immediately. |
| Q-09 | Calculator on Quotation — Apply sets `custom_priority` | Enter delivery date, Calculate, Confirm & Apply. | `custom_priority` = Urgent. Form saves. |
| Q-10 | Order Type auto-set from items | Add a Custom item (is_custom_order = 1). Save. | `custom_order_type` auto-set to "Custom". |
| Q-11 | "Urgent" offer when changing priority to Urgent | Change Priority to Urgent. | Confirm dialog: "Open the Dispatch Date Calculator now?" |
| Q-12 | Priority = Urgent, no delivery date | Set Priority = Urgent. Open calculator dialog. | Dialog opens with empty date field. No auto-run. |
| Q-13 | Save Quotation with Priority = Urgent and Order Type | Set both. Save. | No errors. Priority banner shows. |
| Q-14 | Urgency banner shows estimated delivery | Set `custom_estimated_delivery_date`. Set Priority = Urgent. | Banner shows estimated delivery date. |
| Q-15 | Priority gate blocks VIP without order type | Clear `custom_order_type`. Set Priority = VIP. Save. | Save blocked with error. |

---

### Lyfe Order Screen — 25 Test Cases

| # | What to Test | How | Expected Result |
|---|---|---|---|
| O-01 | Dispatch calculator button visible | Open any Lyfe Order. Look in toolbar. | "Urgent → Calculate Dispatch Dates" button visible. |
| O-02 | Standard/Possible verdict — green | Enter delivery date 30 WD out. Standard order. Calculate. | Green "Possible" badge. Runway count shown. |
| O-03 | Custom/Borderline verdict — amber | Enter delivery date ~31 WD out. Custom order. Calculate. | Amber "Borderline" badge. Warning shown. |
| O-04 | Not Possible verdict — red | Enter delivery date 5 days out. Calculate. | Red "Not Possible". No Apply button. |
| O-05 | Recalculate changes verdict | Get Not Possible. Change date to 30 WD out. Recalculate. | Verdict changes to Possible. Apply button appears. |
| O-06 | Apply stamps `promised_dispatch_by` | Get Possible verdict. Confirm & Apply. | `promised_dispatch_by` filled with Latest Dispatch Date. |
| O-07 | Apply sets `order_priority = Urgent` | After Apply. | `order_priority` = Urgent. |
| O-08 | Save after Apply — banner appears | After Apply, Save. | Urgent banner visible with dispatch date and days remaining. |
| O-09 | Pipeline breakdown expandable | After Calculate, click "Pipeline breakdown". | Rows show production / gate pass / transit / customs segments. |
| O-10 | US Warehouse shows extra segment | Check "Route via US Warehouse". Calculate. | Pipeline breakdown shows "US Warehouse Leg: 4 wd". |
| O-11 | Customs off removes customs row | Uncheck customs. Calculate. | "Customs Buffer" row absent from pipeline breakdown. |
| O-12 | Priority dropdown → Urgent auto-dialog | Change `order_priority` to Urgent. `promised_dispatch_by` empty. | Confirm dialog fires, then calculator opens. |
| O-13 | Priority → Urgent, date already set | Order with `promised_dispatch_by` filled. Change priority to Urgent. | No dialog. Green alert shows. Banner refreshes. |
| O-14 | Priority → Normal (downgrade) | Order is Urgent. Change to Normal. | Confirm dialog: "Remove Urgent flag? This will clear the promised dispatch date." |
| O-15 | Downgrade Cancel | Click Cancel on downgrade confirm. | Priority reverts to Urgent. Date unchanged. |
| O-16 | Downgrade Confirm | Click Yes. | `promised_dispatch_by` cleared. Banner disappears. |
| O-17 | VIP priority — no auto-dialog | Change priority to VIP. | No calculator dialog. Priority accepted. |
| O-18 | Priority synced from Urgent Quotation | Create Lyfe Order from an Urgent Quotation. | Lyfe Order `order_priority` = Urgent. |
| O-19 | Order type synced from Quotation | Quotation has `custom_order_type = Custom`. Create Lyfe Order. | Lyfe Order `order_type` = Custom. |
| O-20 | AT RISK banner — within 2 days | Set `promised_dispatch_by` to today+1. Save. | Banner turns orange, "AT RISK" label. |
| O-21 | OVERDUE banner | Set `promised_dispatch_by` to yesterday. Save. | Banner turns red, "OVERDUE" label. |
| O-22 | Skip for now — no change | Open calculator. Click "Skip for now". | Dialog closes. No fields changed. |
| O-23 | Recalculate after Apply | Apply a result. Re-open dialog. Enter different date. Apply. | `promised_dispatch_by` overwritten with new date. |
| O-24 | No linked Quotation — calculator still works | Order with no `source_quotation`. Change priority to Urgent. | Calculator opens. Works normally. |
| O-25 | SLA exclusion check | Set order to Urgent with factory_first_action > 3 WD ago. Run SLA scan. | Order NOT in SLA violation list. |

---

### Dashboard Validation — 5 Test Cases

| # | What to Test | How | Expected Result |
|---|---|---|---|
| D-01 | SLA breach count excludes Urgent orders | Create Urgent order in factory >3 WD. Check Order Analysis dashboard "Breach Promised Dispatch" count. | ⚠️ Will FAIL until BUG-01 is fixed — Urgent order will appear in breach count. |
| D-02 | PM board shows Urgent task | Create Urgent order with valid `promised_dispatch_by`. Open PM Operations Dashboard. | ⚠️ Will FAIL until BUG-04 is fixed — no PM task created. |
| D-03 | Dashboard loads without error | Open Order Analysis dashboard. | No console errors or HTTP 500. |
| D-04 | Priority filter on PM board | Open PM board. Filter by priority = Urgent. | Shows only Urgent tasks. |
| D-05 | Urgent overdue count | PM board header metric. | Count reflects overdue Tasks with priority = Urgent. |

---

### Notifications — 5 Test Cases

| # | What to Test | How | Expected Result |
|---|---|---|---|
| N-01 | No email during testing to non-test users | Perform all order operations. | Only viralk@lyfehardware.com receives any email. |
| N-02 | AT RISK notification | ⚠️ Cannot test — scanner not built (GAP-02). | N/A until GAP-02 fixed. |
| N-03 | Not Possible escalation | Show Not Possible verdict. | ⚠️ No escalation button — GAP-03. |
| N-04 | Urgent order creation notification | Flag order Urgent and save. | ⚠️ No notification sent — not implemented. |
| N-05 | Downgrade notification | Downgrade from Urgent to Normal. | ⚠️ No notification sent — not implemented. |

---

### Edge Case Scenarios — 10 Test Cases

| # | Scenario | Steps | Expected |
|---|---|---|---|
| EC-01 | Sunday delivery date | Enter Required Delivery Date = next Sunday. Calculate. | Succeeds. Latest Dispatch Date is a working day (Sat or Mon). |
| EC-02 | Past delivery date | Enter yesterday's date. Calculate. | API rejects: "Required delivery date must be today or in the future." |
| EC-03 | Delivery date = today | Enter today's date. Calculate. | Not Possible (all post-dispatch stages exceed available days). |
| EC-04 | Leap year date (Feb 29) | Enter Feb 29 on a leap year. Calculate. | No crash. Calculation proceeds normally. |
| EC-05 | Year-end transition | Enter Dec 31. Calculate. Standard order. | Latest dispatch date correctly falls in prior month. |
| EC-06 | Custom order with US warehouse | Check via US + Custom order type. 50 WD delivery. | US Leg AND customs both shown in breakdown. Extra pipeline segments consumed. |
| EC-07 | Rapid double-click Calculate | Click Calculate twice before spinner disappears. | No crash. Single result shown. |
| EC-08 | Borderline → Save → check PM task | Borderline result. Apply. Save. Check Tasks. | ⚠️ PM task absent (BUG-04). Once fixed: task with priority=Urgent visible. |
| EC-09 | Large delivery window (365 days out) | Enter date 1 year from today. Standard. | Possible with large runway. No overflow or calculation error. |
| EC-10 | Missing `order_type` on Lyfe Order | Clear `order_type` on a Lyfe Order. Open calculator. | Dialog defaults Order Type to "Standard". Calculation proceeds. |

---

## What Has Been Tested (Per UI Testing Guide + This Analysis)

- ✅ Core SLA calculation math (unit-testable from `dispatch_calculator.py`)
- ✅ Working day arithmetic (Mon–Sat, Sunday exclusion)
- ✅ Dialog rendering — all three verdict states
- ✅ Pipeline breakdown display
- ✅ Confirm & Apply field stamping
- ✅ Downgrade confirmation flow (UI)
- ✅ Priority gate (Quotation + server-side enforcement)
- ✅ Quotation → Lyfe Order priority sync (code path exists)
- ✅ SLA detector exclusion clause
- ✅ Urgent banner visual states (UI-side)

## What Remains Untested

- ❌ PM task creation end-to-end (BUG-04 blocks)
- ❌ Dashboard SLA counts excluding Urgent (BUG-01 blocks)
- ❌ AT RISK server-side notification (GAP-02 — not built)
- ❌ Not Possible escalation (GAP-03 — not built)
- ❌ "High" priority sync from Quotation (BUG-02 produces invalid field value)
- ❌ Holiday-aware dispatch dates (GAP-05 — not built)
- ❌ Post-conversion priority sync (GAP-04 — not built)
- ❌ Multi-user concurrency on Urgent flagging
- ❌ API-level priority change bypassing JS confirm dialog

## Known Risks

1. **BUG-01** — Dashboard numbers will be wrong from day one of live use.
2. **BUG-04** — Core spec promise ("task auto-created, nothing falls through the cracks") is broken.
3. **BUG-02** — "High" priority set on a Quotation silently corrupts the Lyfe Order's priority field.
4. **GAP-02** — Managers will not receive AT RISK alerts unless actively checking the board.
5. **GAP-03** — Not Possible orders may be committed to customers without founder awareness.

## Production Blockers (Must Fix Before Go-Live)

1. BUG-01 — Dashboard Urgent exclusion in `order_analysis.py`
2. BUG-02 — Priority vocabulary mismatch "High" mapping
3. BUG-03 — AT RISK threshold uses calendar days not working days
4. BUG-04 — PM task creation silently fails (automation function not implemented)

## Recommended Additional UI Tests Before Go-Live

After BUG-01/02/03/04 are fixed:

1. Verify `breach_promised_dispatch` metric decreases when an Urgent order that was previously in the count is removed.
2. Verify a PM Task appears in the project board when an order is flagged Urgent and saved.
3. Verify the PM Task due date matches `promised_dispatch_by`.
4. Verify downgrading from Urgent closes the PM Task (check Tasks list, status = Closed).
5. Verify a "High" priority Quotation does not corrupt the Lyfe Order `order_priority` field.
6. Verify AT RISK banner triggers on day where only 1 working day remains (not calendar day).
7. Test Quotation with `custom_priority = High` → create Lyfe Order → confirm `order_priority` = Urgent (not "High").
8. Verify `frm.save()` failure in dialog Apply shows a user-visible error.
9. Re-run Scenario EC-07 (Borderline → Save) to confirm PM task is created after BUG-04 fix.
10. Verify no duplicate PM tasks if an Urgent order is saved twice without changing priority.
