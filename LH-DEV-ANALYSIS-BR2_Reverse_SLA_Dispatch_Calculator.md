# BR-2 · Reverse SLA — Dispatch Date Calculator for Urgent Orders
## Developer Implementation Analysis

**Doc ID:** LH-DEV-ANALYSIS-BR2  
**Based on:** LH-BRD-DEV-004 v1.1  
**Date:** 2026-06-17  
**Prepared for:** Viral (ERPNext Dev)

---

## 1. What the Requirement Actually Asks

BR-2 is a **reverse SLA calculator**. Today's SLA runs *forward* (factory_assignment_date → is the order dispatched on time?). BR-2 adds a *backward* calculation: given a customer's required delivery date, work backward through the pipeline to tell CS:

1. **Latest feasible dispatch date** (from factory/India)
2. **Latest feasible order-placement date** (for the customer to still hit the delivery date)
3. **Possibility verdict** — Possible / Borderline / Not Possible
4. **Visibility on the order** — prominently display the dispatch-by date + Urgent flag
5. **PM task auto-creation** — create a tracking task in `lh_project` with the dispatch-by date when an order is flagged Urgent

---

## 2. Pipeline Stages & Buffer Values (from BRD Table 6)

The reverse calculation works backward through these stages:

| Stage | Standard | Custom |
|---|---|---|
| Delivery transit (India factory) | 5–7 wd | 5–7 wd |
| Customs buffer (commercial, India dispatch) | +2–7 wd | +2–7 wd |
| Gatepass + ship-out | 1–3 wd | 1–3 wd |
| Factory production | 1–3 wd (stock-dependent) | ~7+ wd per SOP Section 3 |
| Photo approval (custom urgent only) | — | +1–4 wd |

**Minimum total runway** (working days from today to promised delivery):

| Order Type | Minimum Runway |
|---|---|
| Standard (in stock) | ~10 wd |
| Custom | ~20 wd |

---

## 3. Current State — What Already Exists

### 3.1 Fields on Lyfe Order (already present, not yet used for this purpose)

| Field | Fieldname | Current State |
|---|---|---|
| Order Priority | `order_priority` | Select (High/Medium/Low). Set manually, no business logic attached. |
| Promised Dispatch By | `promised_dispatch_by` | Data field. Only copied during order merge. Never auto-calculated. |
| Factory Assignment Date | `factory_assignment_date` | Set automatically when CS transitions order to factory states (lyfe_order.py:356-387). |
| Total Hold Days | `total_hold_days` | Accumulated correctly when leaving On Hold (lyfe_order.py:528-535). |
| Ready for Dispatch Date | `ready_for_dispatch_date` | Stamped when order reaches dispatch-ready workflow state. |
| Warehouse | `warehouse` | Used by existing SLA detector to distinguish Standard vs Custom path. |

### 3.2 Existing SLA Infrastructure (lh_project module)

The `lh_project` SLA module already has a solid foundation:

- **`lh/lh_project/sla/detectors/promised_dispatch.py`** — forward SLA detector (factory → dispatch). Contains working-day calculation logic in SQL (excludes Sundays + hold days).
- **`lh/lh_project/sla/engine/severity.py`** — priority tiers (Urgent / High / Medium / Low) based on age_hours thresholds.
- **`lh/lh_project/sla/utils/age.py`** — `hours_since()` utility.
- **Scheduler** — already runs `sla_scan` every 15 min, `escalation_scan` hourly.
- **Task auto-creation** — engine already auto-creates PM tasks and auto-closes them.

### 3.3 What Is Missing

| Gap | Impact on BR-2 |
|---|---|
| No working-day **addition** utility (Python, not SQL) | Need to calculate `dispatch_by_date = today + N working days` forward, and `latest_order_date` backward from delivery date |
| No holiday calendar | BRD says exclude IN/US holidays. Currently only Sundays excluded. |
| `order_priority = "Urgent"` has no trigger code | Setting Urgent must auto-populate `promised_dispatch_by` and create a PM task |
| No reverse-calc UI | CS needs a dialog/panel on the Quotation or Lyfe Order to input delivery date and see the calc output |
| `promised_dispatch_by` is Data type, not Date/Datetime | Should be a proper Date for comparisons and scheduling |
| No "compressed vs standard timeline" distinction | BRD requires showing how much buffer is consumed vs standard |
| No "Not Possible" / "Borderline" escalation flow | Need founder escalation path |

---

## 4. Detailed Implementation Plan

### 4.1 New Python Utility — `working_days.py`

**Location:** `lh/lh_project/sla/utils/working_days.py` (new file)

This is the **core building block** for the entire feature. It needs to:

- Add or subtract N working days from a given date
- Exclude Sundays (current SLA logic)
- Exclude Indian public holidays (from a configurable list or ERPNext Holiday List)
- Snap a date that lands on a weekend/holiday to the next valid working day
- Return the number of working days between two dates (currently done in SQL in `promised_dispatch.py` — this Python version is needed for the reverse calc)

```python
# Signature sketch:
def add_working_days(start_date, n, holiday_list=None, direction="forward") -> date
def working_days_between(start_date, end_date, holiday_list=None) -> int
def snap_to_working_day(date, holiday_list=None, direction="forward") -> date
```

**Note:** The existing SQL formula in `promised_dispatch.py` only excludes Sundays (no Saturday, no holidays). The new Python utility should at minimum also exclude Saturdays (standard 5-day work week), then optionally holidays. This will be a slight behaviour change — confirm with Viral/Srishti whether the factory runs on Saturdays before coding.

---

### 4.2 New Service Class — `dispatch_calculator.py`

**Location:** `lh/lh_project/sla/utils/dispatch_calculator.py` (new file)

This class performs the reverse calculation. It is pure business logic — no Frappe DB calls — so it can be unit tested easily.

**Input:**
```python
@dataclass
class DispatchCalcInput:
    required_delivery_date: date
    order_type: str            # "Standard" | "Custom"
    fulfillment_source: str    # "India" | "US"
    is_stock_available: bool   # True for standard, False triggers custom path
    has_customs_buffer: bool   # True by default for India, CS can toggle off for samples
    today: date                # injected for testability
    holiday_list: list[date]   # from ERPNext Holiday List or hardcoded
```

**Output:**
```python
@dataclass
class DispatchCalcResult:
    verdict: str                     # "Possible" | "Borderline" | "Not Possible"
    latest_dispatch_date: date | None
    dispatch_date_range_start: date | None   # e.g. June 20
    dispatch_date_range_end: date | None     # e.g. June 25 (worked example in BRD)
    latest_order_placement_date: date | None
    delivery_date_range_start: date | None
    delivery_date_range_end: date | None
    standard_runway_used: int        # working days from today to dispatch (actual)
    compressed_runway_available: int # how many factory WD remain after all buffers
    standard_factory_days: int       # what standard SLA says (3 or 14 WD)
    is_new_product: bool             # if True, always Not Possible
    notes: list[str]                 # explanatory messages for CS
```

**Calculation Logic (backward from required_delivery_date):**

```
Step 1:  latest_delivery = snap(required_delivery_date, forward to next working day)
Step 2:  latest_ship_from_india = latest_delivery - transit_days_max (5 wd for standard, 7 for custom)
         → dispatch_range_start = latest_delivery - transit_days_max
         → dispatch_range_end   = latest_delivery - transit_days_min
Step 3:  if customs_buffer: subtract customs_wd_max (7 wd) → gatepass_by
Step 4:  subtract gatepass_wd_max (3 wd) → factory_ready_by
Step 5:  subtract photo_approval_wd (1-4 wd, custom only) → factory_start_by
Step 6:  subtract production_wd_max → latest_order_placement_date

Runway check:
  working_days_remaining = working_days_between(today, factory_start_by)
  if working_days_remaining < 0: Not Possible
  if working_days_remaining <= 2: Borderline (manual review required)
  else: Possible

US path (branch when fulfillment_source == "US"):
  transit = 3–4 wd (no customs buffer, no gatepass step)
  production = per US stock (no India factory lead time)
```

---

### 4.3 Whitelisted API Endpoint

**Location:** `lh/lh_project/api/dispatch_calc.py` (new file)

```python
@frappe.whitelist()
def calculate_dispatch_dates(
    required_delivery_date,
    order_type,
    fulfillment_source,
    is_stock_available,
    has_customs_buffer=True,
    lyfe_order=None        # optional — if called from an existing order
) -> dict:
    ...
```

This is called from the client script (dialog) and from the `on_update` hook when an order is flagged Urgent.

---

### 4.4 Client Script — Dispatch Calculator Dialog

**Where:** Lyfe Order form + Quotation form  
**Trigger:** 
- A "Calculate Dispatch Dates" button on the form toolbar (always visible when `order_priority` is empty or Urgent)
- Auto-fires when `order_priority` is set to "Urgent"

**Dialog fields:**
- Required Delivery Date (Date, required)
- Order Type (pre-filled from doc)
- Fulfillment Source (pre-filled from `fulfillment_source` field)
- Stock Available? (Check, default: True for Standard)
- Include Customs Buffer? (Check, default: True for India orders)

**Output display in dialog:**
- Verdict badge (green Possible / amber Borderline / red Not Possible)
- Latest Dispatch Date (or range)
- Latest Order Placement Date
- Standard runway vs compressed runway comparison
- Notes/warnings from the calc

**On "Confirm & Apply":**
- Sets `promised_dispatch_by` on the doc
- Sets `order_priority = "Urgent"` (if not already)
- Saves the doc (which triggers the task auto-creation hook — see 4.5)

---

### 4.5 Lyfe Order Hook — on_update (Urgent Flag Handler)

**Location:** `lh/lyfe_hardware/doctype/lyfe_order/lyfe_order.py` — add to existing `on_update()`  
**Also:** `lh/lh_project/automation/lyfe_order.py` — or a dedicated handler

**Trigger condition:** `order_priority` changed to "Urgent" AND `promised_dispatch_by` is set

**Actions:**
1. Create (or update) a PM task in `lh_project` with:
   - Subject: `"URGENT — Dispatch by {promised_dispatch_by} — {order_id}"`
   - Description: Full calc breakdown (compressed timeline vs standard)
   - Priority: "Urgent"
   - Due date: `promised_dispatch_by`
   - Assigned to: factory assignment owner (or `fallback_owner` from SLA Rule)
2. Log the calc result in the order's `log` field (existing Code field)
3. If `order_priority` is *downgraded from* Urgent:
   - Mark the PM task as cancelled/closed
   - Clear `promised_dispatch_by`
   - Log the downgrade

**Anti-orphan guard:** Before creating a new task, check if a task already exists (via `SLA Task Link` or a custom field `urgent_task_link`). If it exists, update rather than duplicate.

---

### 4.6 SLA Exclusion — Urgent Orders

The BRD says: *"Urgent orders running on compressed timelines must be excluded from standard SLA-breach dashboards."*

**Change in `promised_dispatch.py`:**

Add an exclusion clause to the `WHERE` condition in `get_candidates()`:

```sql
AND (order_priority IS NULL OR order_priority != 'Urgent')
```

Urgent orders have their own dispatch-by date (`promised_dispatch_by`) and their own PM task. They should not appear as forward-SLA breaches on the standard dashboard.

---

### 4.7 "At Risk" Re-flag on Client Delay (Edge Case: BRD row 7 in Table 8)

If client-side delays (photo approval, payment) eat into the compressed buffer after an urgent order is confirmed:

**How:** Add a scheduled check (daily, or piggyback on existing 15-min SLA scan) that:
1. Finds Urgent orders where `promised_dispatch_by` is within 2 working days
2. AND `ready_for_dispatch_date` is NOT set (still in factory)
3. Sets a flag or notification — "Urgent order At Risk: compressed buffer consumed"

This can be a lightweight addition to the existing `sla_scan.run` or a separate daily cron.

---

### 4.8 `promised_dispatch_by` Field Type Change

**Current:** Data (text)  
**Should be:** Date

**Migration:** This is a simple ALTER on the field definition in the DocType JSON. Existing values are mostly null/empty (only populated during merges), so data loss risk is minimal. Confirm with Viral before applying.

---

## 5. File Change Summary

| File | Change Type | What Changes |
|---|---|---|
| `lh/lh_project/sla/utils/working_days.py` | **New** | Working-day add/subtract/count utility |
| `lh/lh_project/sla/utils/dispatch_calculator.py` | **New** | Reverse SLA calc service class |
| `lh/lh_project/api/dispatch_calc.py` | **New** | `@frappe.whitelist` API endpoint |
| `lh/lh_project/sla/detectors/promised_dispatch.py` | **Edit** | Add `order_priority != 'Urgent'` exclusion to WHERE clause |
| `lh/lyfe_hardware/doctype/lyfe_order/lyfe_order.py` | **Edit** | Add Urgent flag handler in `on_update`: auto-create PM task, populate `promised_dispatch_by`, handle downgrade |
| `lh/lyfe_hardware/doctype/lyfe_order/lyfe_order.js` | **Edit** | Add "Calculate Dispatch Dates" toolbar button + dialog; auto-fire on priority = Urgent |
| `lh/lyfe_hardware/doctype/lyfe_order/lyfe_order.json` | **Edit** | Change `promised_dispatch_by` field type from Data → Date |
| `hooks.py` | **Possibly Edit** | Add daily "At Risk" re-check cron if not piggybacked on existing scan |

---

## 6. Data Flow Diagram

```
CS opens Quotation / Lyfe Order
         │
         ▼
Sets "Order Priority = Urgent"
    (or clicks "Calculate Dispatch Dates" button)
         │
         ▼
Dialog opens: enter Required Delivery Date + toggle customs/stock
         │
         ▼
Client calls API: dispatch_calc.calculate_dispatch_dates()
         │
         ▼
dispatch_calculator.py
  ├── Snap delivery date to working day
  ├── Subtract transit (5-7 wd India / 3-4 wd US)
  ├── Subtract customs buffer (2-7 wd, optional)
  ├── Subtract gatepass (1-3 wd)
  ├── Subtract photo approval (1-4 wd, custom only)
  ├── Subtract production (1-3 wd std / 7+ wd custom)
  └── Compare remaining runway vs minimum_runway
         │
         ▼
Returns: verdict + dispatch_date_range + latest_order_placement_date
         │
    ┌────┴──────────────────────────────┐
    │                                   │
Not Possible                        Possible / Borderline
    │                                   │
Escalate to founders            CS confirms → "Apply"
(Slack / ERP notif)                     │
                                        ▼
                          Sets promised_dispatch_by on order
                          Sets order_priority = "Urgent"
                                        │
                                        ▼
                          lyfe_order.on_update() fires
                                        │
                          ┌─────────────┴──────────────┐
                          │                            │
                 Create PM task in lh_project     Log calc result
                 (subject: URGENT — Dispatch by X)  in order.log
                 Priority = Urgent
                 Due = promised_dispatch_by
                          │
                          ▼
                 promised_dispatch.py (forward SLA) EXCLUDES Urgent orders
                 Urgent order has its own task + timeline
                          │
                          ▼ (ongoing — daily or 15-min scan)
                 If factory stalls AND date within 2 wd:
                 Flag "At Risk" → notify CS + factory
```

---

## 7. Open Items to Confirm Before Building

| # | Question | Why it matters |
|---|---|---|
| 1 | Does the factory work Saturdays? | Current SLA SQL only excludes Sundays. If factory is 6-day, the Python working_days utility must only exclude Sundays too, for consistency. If 5-day, we also exclude Saturdays — and the existing forward SLA will need updating. |
| 2 | Holiday calendar source | ERPNext has a Holiday List doctype. Should working_days.py read from it, or hardcode major Indian public holidays? US holidays needed too for US fulfilment path. |
| 3 | Founder escalation channel for "Not Possible" | BRD says "Slack alert or ERP notification" — which one (or both)? |
| 4 | "At Risk" re-notification frequency | Once only, or every N days while still at risk? |
| 5 | Should the calc also appear on the Quotation form? | BRD says "input = customer's required delivery date" but doesn't explicitly restrict it to Lyfe Order only. Having it on Quotation gives CS a "can we commit?" check before the order is placed. |
| 6 | New product / die/mould detection | BRD says new products go straight to Not Possible. Is there a field on Item or Lyfe Order that flags "new product requiring tooling"? If not, needs to be a manual toggle in the dialog. |
| 7 | Borderline threshold (within 2 wd) | BRD says auto-flag for manual review rather than hard pass/fail. Should this block saving, or just show a warning banner that CS must acknowledge? |

---

## 8. What Does NOT Need to Change

- The existing `promised_dispatch.py` forward SLA detector — only a one-line exclusion added.
- The `SLA Task Link`, `SLA Violation Cache`, `SLA Escalation Log` DocTypes — no schema changes needed.
- The `severity.py` priority engine — the PM task created for Urgent orders will already be set directly to "Urgent" priority; the severity engine is for auto-escalation of *breached* tasks, which is a separate path.
- The `total_hold_days` accumulation logic — already correct, already excluded from SLA clocks.

---

## 9. Recommended Build Order

1. `working_days.py` — foundation, write tests first
2. `dispatch_calculator.py` — pure logic, write tests against the worked example in the BRD (June 30 → June 20–25 delivery → June 12–14 dispatch)
3. API endpoint + client dialog on Lyfe Order
4. `on_update` Urgent handler + PM task creation
5. `promised_dispatch_by` field type change (Data → Date)
6. Exclusion clause in `promised_dispatch.py`
7. "At Risk" re-notification scan
8. Quotation form support (if confirmed in Open Items)
