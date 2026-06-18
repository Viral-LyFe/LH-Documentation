# BR-2 · Reverse SLA — Dispatch Date Calculator for Urgent Orders

**Doc ID:** LH-DEV-ANALYSIS-BR2  
**Based on:** LH-BRD-DEV-004 v1.1  
**Date:** 2026-06-18  
**Prepared for:** Viral (ERPNext Dev), CS Team, Operations Manager

---

## Who Should Read This

| Section | Audience |
|---|---|
| 1 — What Problem Are We Solving? | Everyone |
| 2 — How It Works (Plain Language) | CS Team, Managers |
| 3 — Step-by-Step: How CS Uses It | CS Team (end users) |
| 4 — How Managers Monitor It | Operations Manager |
| 5 — The Math Behind It | Managers who want the detail |
| 6 — What the System Does Automatically | Everyone |
| 7 — Current State & What Needs to Be Built | Developer |
| 8 — Detailed Technical Plan | Developer |
| 9 — Open Questions Before Building | Developer + Management |

---

## 1. What Problem Are We Solving?

### Today's situation

When a customer says *"I need this by June 30"*, CS currently has no tool to quickly answer: **Can we actually do it? And if yes, when must the factory ship it?**

Right now CS either:
- Makes an educated guess and commits, then discovers too late the date is impossible
- Is overly cautious and turns down orders that were actually feasible
- Commits to a date, factory ships late, customer is unhappy

### What this feature does

It gives CS a **calculator button** on every order. CS enters the customer's required delivery date, clicks a button, and instantly gets back:

- **Can we do it?** — Green (Yes), Amber (Tight but possible — needs manual review), or Red (Not possible, escalate to founders)
- **When must the factory dispatch?** — e.g. "Factory must ship by June 14"
- **A date range to give the customer** — e.g. "Tell the customer June 20–25 delivery"
- **When is the latest the customer can place the order?** — e.g. "Order must be placed by June 3"

When the answer is **Possible**, CS confirms and the system:
1. Stamps the dispatch-by date on the order so the factory cannot miss it
2. Flags the order as **Urgent** with a prominent visual indicator
3. Automatically creates a tracking task in the project board so nothing falls through the cracks

---

## 2. How It Works — Plain Language

Think of it like planning a road trip **in reverse**.

Instead of asking *"if I leave today, when do I arrive?"*, we ask: *"I need to arrive by June 30 — what is the latest I can leave, and the latest I can start preparing the car?"*

The calculator works backward through every step of the delivery pipeline:

```
Customer needs delivery by:  June 30
                                ↑
minus  Delivery transit time:   5–7 working days (shipping from India)
                                ↑
minus  Customs clearance:       2–7 working days (for international shipments)
                                ↑
minus  Gatepass + handover:     1–3 working days (paperwork at factory)
                                ↑
minus  Factory production:      1–3 days (standard, if stock exists)
                                   OR 7+ days (custom / made-to-order)
                                ↑
= Latest possible factory dispatch date + order placement deadline
```

### Real example from the BRD

> *Customer asks: "I need delivery by June 30."*

Working backward:

| Step | Calculation | Result |
|---|---|---|
| Customer's required delivery | — | June 30 |
| Minus transit (5–7 wd) | June 30 − 7 wd | Factory must dispatch by ~June 20 |
| Minus customs buffer (2–7 wd) | June 20 − 5 wd | Gatepass must be done by ~June 13 |
| Minus gatepass (1–3 wd) | June 13 − 2 wd | Factory must be ready to ship by ~June 11 |
| Minus production (1–3 wd std) | June 11 − 2 wd | Order must be in factory by ~June 9 |

**What CS tells the customer:** "We can deliver June 20–25. We need your confirmed order by June 9."  
**What the factory sees on the order:** "URGENT — Dispatch by June 14."

### What "working days" means

All calculations skip Sundays and public holidays (Indian holidays for India dispatch; US holidays for US warehouse orders). A date that falls on a holiday automatically snaps to the next valid working day.

---

## 3. Step-by-Step: How CS Uses This Feature

### Scenario A — Customer calls asking for a tight delivery date

**Step 1: Open the order (or quotation)**

CS opens the Lyfe Order or Quotation in ERPNext as normal.

**Step 2: Click "Calculate Dispatch Dates"**

A new button appears in the toolbar at the top of the form. It is always visible.

**Step 3: Fill in the small dialog that pops up**

The dialog has just a few fields — most are pre-filled:

```
┌─────────────────────────────────────────────────────┐
│  Dispatch Date Calculator                           │
├─────────────────────────────────────────────────────┤
│  Customer's Required Delivery Date:  [ June 30 ]   │
│  Order Type:          [ Standard ▾ ]  (pre-filled) │
│  Fulfilled From:      [ India     ▾ ]  (pre-filled) │
│  Stock Available?     [✓]              (check if yes)│
│  Include Customs Buffer? [✓]          (default: yes)│
└─────────────────────────────────────────────────────┘
                              [ Calculate ]
```

**Step 4: Read the result**

The result appears in the same dialog:

```
┌─────────────────────────────────────────────────────┐
│  ✅  POSSIBLE                                        │
├─────────────────────────────────────────────────────┤
│  Tell the customer:   Delivery by  June 20–25       │
│  Factory must ship:   By June 14   (latest)         │
│  Order must be placed: By June 9   (latest)         │
│                                                     │
│  Timeline used:    12 working days (compressed)     │
│  Standard timeline: 14 working days                 │
│  Buffer consumed:   2 working days of standard time │
├─────────────────────────────────────────────────────┤
│         [ Cancel ]    [ Confirm & Flag Urgent ]     │
└─────────────────────────────────────────────────────┘
```

**Step 5: Click "Confirm & Flag Urgent"**

The system automatically:
- Marks the order as **Urgent**
- Stamps "Dispatch by June 14" on the order (visible prominently)
- Creates a factory task: *"URGENT — Dispatch by June 14 — ORDER-001"*
- Logs the full calculation in the order history

CS can now tell the customer the date range with confidence.

---

### Scenario B — The date is NOT possible

The result dialog shows:

```
┌─────────────────────────────────────────────────────┐
│  🔴  NOT POSSIBLE                                   │
├─────────────────────────────────────────────────────┤
│  Required delivery:   June 20                       │
│  Today is:            June 13                       │
│  Working days available: 5 wd                       │
│  Minimum needed (standard order): 10 wd             │
│                                                     │
│  ⚠ We would need to dispatch by June 10 — that     │
│    date is already in the past.                     │
│                                                     │
│  This order has been flagged for founder review.    │
├─────────────────────────────────────────────────────┤
│  [ Close ]     [ Escalate to Founders ]             │
└─────────────────────────────────────────────────────┘
```

CS does NOT commit to the customer. Instead:
- Clicks "Escalate to Founders"
- Founders receive a notification (Slack / ERP) and decide whether to accept it as a special case
- CS waits for founder decision before responding to the customer

---

### Scenario C — Borderline (tight but not impossible)

```
┌─────────────────────────────────────────────────────┐
│  🟡  BORDERLINE — Manual Review Required            │
├─────────────────────────────────────────────────────┤
│  Factory dispatch by:  June 14                      │
│  Working days available for factory: 2 wd           │
│                                                     │
│  ⚠ This is within 2 working days of the minimum.   │
│    CS must confirm with the factory before          │
│    committing this date to the customer.            │
├─────────────────────────────────────────────────────┤
│  [ Cancel ]   [ I've Confirmed with Factory — Apply ]│
└─────────────────────────────────────────────────────┘
```

CS calls/messages the factory floor, gets verbal confirmation, then clicks Apply.

---

### Scenario D — Order was flagged Urgent, but customer changes their delivery date

CS opens the order, clicks "Calculate Dispatch Dates" again with the new date. The system automatically:
- Recomputes all dates
- Updates the dispatch-by date on the order
- Updates the existing factory task (does not create a duplicate)
- Re-runs the possibility check

If the order is **downgraded from Urgent** (e.g. customer agreed to a later date):
- CS changes Order Priority from Urgent back to Normal
- System closes the Urgent factory task automatically
- Clears the dispatch-by date
- Logs "Order downgraded from Urgent by [user] on [date]"

---

## 4. How Managers Monitor Urgent Orders

### 4.1 The Urgent Orders List View

A filtered list view will show all currently active Urgent orders at a glance:

```
Urgent Orders (Active)        [ Refresh ]

Order ID    Customer        Dispatch By    Days Left    Status         At Risk?
ORDER-001   ABC Corp        June 14        3 wd         In Factory     —
ORDER-002   XYZ Ltd         June 12        1 wd         In Factory     ⚠ YES
ORDER-003   Patel Co.       June 18        7 wd         Ready to Ship  —
ORDER-004   US Client       June 11        0 wd         In Factory     🔴 OVERDUE
```

**Columns explained:**
- **Dispatch By** — the date stamped by the calculator when the order was flagged Urgent
- **Days Left** — working days remaining until that dispatch deadline (calculated in real time)
- **At Risk** — automatically flagged when Days Left drops to 2 or fewer and the order is still in factory

### 4.2 Automatic Alerts the Manager Receives

The system sends automatic notifications in these situations:

| Situation | Who Is Notified | When |
|---|---|---|
| Order flagged as Urgent (new) | CS owner + Factory assignee | Immediately when Urgent is applied |
| Order is "Not Possible" — needs founder decision | Founders | Immediately when CS escalates |
| Dispatch deadline is 2 working days away, still in factory | Manager + CS owner + Factory | Daily scan (runs every morning) |
| Dispatch deadline has passed, order not dispatched | Manager + CS owner + Founders | Daily scan |
| Client delay (photo approval / payment) has eaten the buffer | CS owner + Factory | Daily scan |
| Order downgraded from Urgent | Manager (informational) | Immediately when changed |

### 4.3 The Project Board (lh_project)

Every Urgent order automatically gets a task card on the PM project board. Managers can see all Urgent tasks in one place, with:

- Task title: `URGENT — Dispatch by June 14 — ORDER-001`
- Due date visible on the card
- Priority badge: **Urgent** (red)
- Assigned to: the factory team member responsible
- Description: full calculation breakdown (what timeline was used, how much buffer is left)

Tasks auto-close when the order reaches "Ready for Dispatch" status — no manual cleanup needed.

### 4.4 What a Manager Should Review Daily

1. **Check the "At Risk" column** in the Urgent Orders list — any row with ⚠ needs immediate follow-up with factory or CS
2. **Check the PM board "Urgent" priority tasks** — overdue tasks (red due date) need escalation
3. **Check the "Escalated to Founders" queue** — any Not Possible orders waiting for a decision

### 4.5 What the Dashboard Does NOT Do Automatically (requires manager action)

- Deciding whether to accept a "Not Possible" order — this is always a founder/manager decision
- Chasing the customer when their delay has caused an At Risk situation — CS does this
- Adjusting the dispatch date if factory confirms they can do it faster — CS updates the order manually

---

## 5. The Math Behind the Calculation

This section is for managers who want to understand exactly how the numbers are derived.

### 5.1 The Buffer Values (from SOP LH-CS-SOP-001 v1.7)

Each stage of the delivery has a minimum and maximum buffer in working days:

| Stage | Min (wd) | Max (wd) | Notes |
|---|---|---|---|
| Delivery transit — India to customer | 5 | 7 | International shipping time |
| Customs clearance | 2 | 7 | Applies to commercial shipments from India. CS can turn off for sample shipments < 20 kg. |
| Gatepass + handover to carrier | 1 | 3 | Paperwork at factory before handing to courier |
| Factory production — Standard order | 1 | 3 | Only if stock exists. If out of stock, treated as Custom. |
| Factory production — Custom order | 7 | 14+ | Per SOP Section 3. New products (requiring die/mould) = 25–30 days → always Not Possible for urgent. |
| Photo approval — Custom urgent orders | 1 | 4 | Time reserved for client to review factory photos |

**The calculator always uses the maximum buffers** to give the factory the tightest deadline. The date range shown to the customer is built using min vs max transit time, which is why there is a range (e.g. "June 20–25").

### 5.2 US Warehouse Orders — Different Path

US warehouse orders skip the India customs and gatepass steps entirely:

| Stage | US Path |
|---|---|
| Transit to customer | 3–4 wd (domestic US shipping) |
| Customs | None |
| Gatepass | None |
| Production | Per US stock availability |

This means US standard orders have a **much shorter minimum runway** than India orders.

### 5.3 Minimum Runway Thresholds

| Order Type | Minimum Working Days from Today to Delivery |
|---|---|
| Standard order (India, in stock) | 10 working days |
| Custom order (India) | 20 working days |
| New product (die/mould required) | Not Possible — always escalate to founders |

### 5.4 Borderline Zone

If the working days available for the factory fall within **2 working days of the minimum**, the system flags it as Borderline instead of Possible. CS must verbally confirm with the factory before committing the date to the customer.

**Example of Borderline:**
- Standard order minimum runway: 10 wd
- Available runway today: 11 wd → only 1 wd of breathing room → **Borderline**
- Available runway today: 14 wd → 4 wd of breathing room → **Possible**

---

## 6. What the System Does Automatically (No Human Action Needed)

| Event | Automatic Action |
|---|---|
| CS clicks "Confirm & Flag Urgent" | Order priority set to Urgent; dispatch-by date stamped on order; PM task created; calculation logged |
| Order dispatch deadline arrives and order still in factory | "At Risk" flag set; notifications sent to CS + factory + manager |
| Order reaches "Ready for Dispatch" status | Urgent PM task auto-closed; no manual cleanup needed |
| Customer's delivery date changes on order | Dispatch-by date recomputed; existing PM task updated |
| Order priority changed from Urgent to Normal | PM task closed; dispatch-by date cleared; downgrade logged |
| "Not Possible" escalated to founders | Founder notification sent immediately via ERP/Slack |
| Urgent order excluded from standard SLA breach reports | This happens automatically — Urgent orders have their own tracking and do not pollute the standard SLA dashboard |

---

## 7. Current State — What Already Exists vs What Is New

### 7.1 Already in the System (No Build Required)

| What | Where It Lives | Status |
|---|---|---|
| Order Priority field (High / Medium / Low) | Lyfe Order | Exists, but currently no business logic attached to it |
| Promised Dispatch By field | Lyfe Order | Exists, but currently filled manually (only during order merges) |
| Factory Assignment Date | Lyfe Order | Already auto-set when CS assigns to factory |
| Total Hold Days | Lyfe Order | Already calculated correctly |
| SLA task auto-creation engine | lh_project module | Already works for standard SLA breaches |
| 15-minute SLA scanner | Scheduler | Already running |
| Working-day calculation (SQL, Sundays only) | promised_dispatch.py | Already exists; needs a Python version for the reverse calc |

### 7.2 Gaps — What Needs to Be Built

| Gap | Why It Matters |
|---|---|
| No working-day add/subtract in Python | The reverse calculation requires going backward by N working days — this cannot be done in SQL alone |
| No holiday calendar integration | Currently only Sundays are excluded. Indian public holidays (and US holidays for US orders) need to be respected. |
| "Urgent" priority has no trigger code | Setting Urgent currently does nothing. It needs to fire: populate dispatch-by date, create PM task, send notifications. |
| No reverse-calc UI (dialog + button) | CS has no tool to do this today — they calculate manually or guess |
| `promised_dispatch_by` is a text field, not a date | Cannot be sorted, compared, or used in date math without converting it to a proper Date field |
| No "Not Possible" / Borderline escalation path | No notification goes to founders today when CS cannot commit to a date |
| Urgent orders not excluded from standard SLA reports | Without the exclusion, an Urgent order running on a compressed 10-day timeline would incorrectly appear as a standard SLA breach after 3 days |

---

## 8. Detailed Technical Plan (Developer Reference)

### 8.1 New Python Utility — `working_days.py`

**Location:** `lh/lh_project/sla/utils/working_days.py` (new file)

This is the **core building block** for the entire feature. It needs to:

- Add or subtract N working days from a given date
- Exclude Sundays (current SLA logic)
- Exclude Indian public holidays (from ERPNext Holiday List or a configurable list)
- Snap a date that lands on a weekend/holiday to the next valid working day
- Count working days between two dates (currently done in SQL in `promised_dispatch.py` — a Python version is needed for the reverse calc)

```python
# Signature sketch:
def add_working_days(start_date, n, holiday_list=None, direction="forward") -> date
def working_days_between(start_date, end_date, holiday_list=None) -> int
def snap_to_working_day(date, holiday_list=None, direction="forward") -> date
```

**Note:** The existing SQL formula in `promised_dispatch.py` only excludes Sundays (no Saturday, no holidays). The new Python utility should at minimum also exclude Saturdays (standard 5-day work week). **Confirm with Viral/Srishti whether the factory runs on Saturdays before coding** — if yes, the utility should only exclude Sundays to stay consistent with the existing SLA.

---

### 8.2 New Service Class — `dispatch_calculator.py`

**Location:** `lh/lh_project/sla/utils/dispatch_calculator.py` (new file)

This class performs the reverse calculation. It contains pure business logic — no Frappe database calls — so it can be unit tested easily and independently.

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
    verdict: str                          # "Possible" | "Borderline" | "Not Possible"
    latest_dispatch_date: date | None
    dispatch_date_range_start: date | None    # e.g. June 20
    dispatch_date_range_end: date | None      # e.g. June 25
    latest_order_placement_date: date | None
    delivery_date_range_start: date | None
    delivery_date_range_end: date | None
    standard_runway_used: int             # working days from today to dispatch (actual)
    compressed_runway_available: int      # factory WD remaining after all buffers
    standard_factory_days: int            # what standard SLA allows (3 or 14 WD)
    is_new_product: bool                  # if True, always Not Possible
    notes: list[str]                      # human-readable messages for CS
```

**Calculation Logic (working backward from required_delivery_date):**

```
Step 1:  latest_delivery = snap(required_delivery_date, to next working day)
Step 2:  latest_dispatch = latest_delivery − transit_days_max
         → dispatch_range_start = latest_delivery − transit_days_max
         → dispatch_range_end   = latest_delivery − transit_days_min
Step 3:  if customs_buffer: subtract customs_wd_max (7 wd) → gatepass_deadline
Step 4:  subtract gatepass_wd_max (3 wd) → factory_ready_deadline
Step 5:  subtract photo_approval_wd (1–4 wd, custom only) → factory_start_deadline
Step 6:  subtract production_wd_max → latest_order_placement_date

Runway check:
  working_days_remaining = working_days_between(today, factory_start_deadline)
  if working_days_remaining < 0:   verdict = "Not Possible"
  if working_days_remaining <= 2:  verdict = "Borderline"
  else:                            verdict = "Possible"

US path (when fulfillment_source == "US"):
  transit = 3–4 wd (skip customs buffer and gatepass steps entirely)
```

---

### 8.3 Whitelisted API Endpoint

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

Called from: the client-side dialog (button click) and from the `on_update` hook when `order_priority` is set to Urgent and `promised_dispatch_by` is not yet set.

---

### 8.4 Client Script — Dispatch Calculator Dialog

**Where:** Lyfe Order form + Quotation form  
**Trigger:**
- A "Calculate Dispatch Dates" button on the form toolbar (always visible)
- Auto-fires when `order_priority` is changed to "Urgent" and `promised_dispatch_by` is empty

**Dialog fields:**
- Required Delivery Date (Date, required)
- Order Type (pre-filled from doc, editable)
- Fulfillment Source (pre-filled from `fulfillment_source` field)
- Stock Available? (Check, default: True for Standard orders)
- Include Customs Buffer? (Check, default: True for India dispatch)

**Output shown in dialog:**
- Verdict badge (green Possible / amber Borderline / red Not Possible)
- Latest Dispatch Date and date range
- Latest Order Placement Date
- Standard timeline vs compressed timeline comparison
- Notes/warnings from the calc result

**On "Confirm & Apply":**
1. Sets `promised_dispatch_by` on the doc
2. Sets `order_priority = "Urgent"`
3. Saves the doc (which fires the `on_update` hook — see 8.5)

---

### 8.5 Lyfe Order Hook — on_update (Urgent Flag Handler)

**Location:** `lh/lyfe_hardware/doctype/lyfe_order/lyfe_order.py` — add to existing `on_update()`

**Trigger condition:** `order_priority` changed to "Urgent" AND `promised_dispatch_by` is set

**Actions on Urgent flag:**
1. Create (or update) a PM task in `lh_project` with:
   - Subject: `"URGENT — Dispatch by {promised_dispatch_by} — {order_id}"`
   - Description: full calc breakdown (compressed vs standard timeline, notes from calc result)
   - Priority: "Urgent"
   - Due date: `promised_dispatch_by`
   - Assigned to: factory assignment owner (or `fallback_owner` from SLA Rule)
2. Log the calc result in the order's `log` field (existing Code field on Lyfe Order)
3. Send notification to CS owner + factory assignee

**Actions on Urgent downgrade (priority changed away from Urgent):**
1. Mark the existing PM task as Cancelled
2. Clear `promised_dispatch_by`
3. Log: `"Order downgraded from Urgent by {user} on {date}. Reason: {reason}"`

**Anti-orphan guard:** Before creating a new PM task, check if one already exists (via `SLA Task Link` or a stored `urgent_task_link` field). If it exists, update it rather than creating a duplicate.

---

### 8.6 SLA Exclusion — Urgent Orders

The BRD states: *"Urgent orders running on compressed timelines must be excluded from standard SLA-breach dashboards."*

**Change required in `promised_dispatch.py`:**

Add one exclusion clause to the `WHERE` condition inside `get_candidates()`:

```sql
AND (order_priority IS NULL OR order_priority != 'Urgent')
```

Urgent orders have their own `promised_dispatch_by` date and their own PM task. They must not also appear as standard SLA breaches — that would double-count them and pollute the factory workload dashboard.

---

### 8.7 "At Risk" Re-flag on Client Delay

**Trigger:** Client-side delays (photo approval not done, payment pending) eat into the compressed buffer after an order is already flagged Urgent.

**How — add to existing 15-min SLA scan or as a separate daily cron:**
1. Find all Urgent orders where `promised_dispatch_by` is within 2 working days from today
2. AND `ready_for_dispatch_date` is NOT yet set
3. Set an At Risk indicator (a field or a notification)
4. Send notification to: CS owner + Factory assignee + Manager

---

### 8.8 `promised_dispatch_by` Field Type Change

**Current type:** Data (plain text)  
**Required type:** Date

**Why it matters:** A text field cannot be sorted chronologically, cannot trigger date-based alerts, and cannot be used in working-day arithmetic.

**Migration risk:** Low. The field is currently only populated during order merges. The vast majority of existing records have it empty. A simple DocType field type change in `lyfe_order.json` is all that is needed. Confirm with Viral before applying.

---

## 9. File Change Summary (Developer)

| File | Change Type | What Changes |
|---|---|---|
| `lh/lh_project/sla/utils/working_days.py` | **New** | Working-day add/subtract/count utility with holiday support |
| `lh/lh_project/sla/utils/dispatch_calculator.py` | **New** | Reverse SLA calc service class — pure logic, fully testable |
| `lh/lh_project/api/dispatch_calc.py` | **New** | `@frappe.whitelist` API endpoint called from the dialog |
| `lh/lh_project/sla/detectors/promised_dispatch.py` | **Edit** | Add `order_priority != 'Urgent'` exclusion to `WHERE` clause |
| `lh/lyfe_hardware/doctype/lyfe_order/lyfe_order.py` | **Edit** | Add Urgent flag handler in `on_update`: create PM task, populate `promised_dispatch_by`, handle downgrade cleanup |
| `lh/lyfe_hardware/doctype/lyfe_order/lyfe_order.js` | **Edit** | Add "Calculate Dispatch Dates" toolbar button + dialog; auto-fire on priority change to Urgent |
| `lh/lyfe_hardware/doctype/lyfe_order/lyfe_order.json` | **Edit** | Change `promised_dispatch_by` field type: Data → Date |
| `hooks.py` | **Possibly Edit** | Add daily "At Risk" cron if not piggybacked on the existing 15-min scan |

---

## 10. Full Flow Diagram

```
CS opens Quotation or Lyfe Order
             │
             ▼
  Clicks "Calculate Dispatch Dates" button
  (or changes Order Priority to "Urgent")
             │
             ▼
  Dialog opens — CS enters Required Delivery Date
  (Order Type, Fulfillment Source pre-filled)
             │
             ▼
  System calls dispatch_calculator.py
    ├── Snap delivery date to next working day
    ├── Subtract transit (5–7 wd India / 3–4 wd US)
    ├── Subtract customs buffer (2–7 wd, India only)
    ├── Subtract gatepass (1–3 wd)
    ├── Subtract photo approval (1–4 wd, custom only)
    ├── Subtract production (1–3 wd standard / 7+ wd custom)
    └── Compare remaining runway vs minimum runway
             │
             ▼
   Result shown to CS in dialog
             │
   ┌─────────┼──────────────────┐
   │         │                  │
🔴 Not    🟡 Borderline     ✅ Possible
Possible      │                  │
   │       CS confirms       CS clicks
   │       with factory    "Confirm & Apply"
   │       before applying       │
   ▼                             ▼
Escalate to              Sets promised_dispatch_by
Founders                 Sets order_priority = Urgent
(Slack / ERP)                    │
                                 ▼
                    lyfe_order.on_update() fires
                                 │
                    ┌────────────┴─────────────┐
                    │                          │
           Create PM task               Log calc result
           in lh_project                in order.log
           "URGENT — Dispatch           + send notification
            by June 14 — ORDER-001"     to CS + factory
           Priority: Urgent
           Due: June 14
                    │
                    ▼
   promised_dispatch.py (standard SLA) automatically
   EXCLUDES this order — it has its own tracking
                    │
                    ▼  (runs daily via scheduler)
   If factory stalls AND dispatch deadline ≤ 2 wd away:
   → Flag "At Risk" → notify CS + factory + manager
                    │
                    ▼
   Order reaches "Ready for Dispatch"
   → PM task auto-closes (no manual action needed)
   → Urgent tracking complete
```

---

## 11. Open Questions to Confirm Before Building

| # | Question | Who Decides | Why It Matters |
|---|---|---|---|
| 1 | Does the factory work Saturdays? | Operations / Factory | If yes, the working-day utility should only skip Sundays (to match the existing SLA). If no, it should also skip Saturdays — but then the existing SLA will also need updating to be consistent. |
| 2 | Holiday calendar source | Operations | ERPNext has a built-in Holiday List. Should the system read from it, or should we hardcode the major Indian public holidays? US holidays also needed for US warehouse path. |
| 3 | Founder escalation channel for "Not Possible" | Founders / Srishti | BRD says "Slack alert or ERP notification" — which, or both? |
| 4 | "At Risk" re-notification frequency | Operations | Should the system notify once only, or send a reminder every working day until the order is dispatched or the date passes? |
| 5 | Should the calculator also appear on the Quotation form? | CS / Srishti | Putting it on the Quotation lets CS check feasibility before the order is even placed — this is earlier and more useful. The BRD does not restrict it to Lyfe Order only. |
| 6 | How to detect "New Product / Die & Mould" orders | Developer + Operations | BRD says these should always show "Not Possible" for urgent. Is there a field on the Item or Order that marks this? If not, a manual toggle in the dialog is needed. |
| 7 | Borderline behaviour — block or warn? | CS / Srishti | Should Borderline orders be blocked from saving until CS acknowledges, or just show a warning banner that CS can dismiss? |

---

## 12. What Does NOT Change

These parts of the existing system need no modification:

- The forward SLA `promised_dispatch.py` detector — only a one-line exclusion is added.
- `SLA Task Link`, `SLA Violation Cache`, `SLA Escalation Log` DocTypes — no schema changes.
- The `severity.py` priority engine — Urgent PM tasks are created with "Urgent" priority directly; the severity engine handles *breach escalation* on existing tasks, which is a separate path.
- The `total_hold_days` accumulation logic — already correct; hold time is already excluded from SLA clocks.

---

## 13. Recommended Build Order

| Step | What | Why This Order |
|---|---|---|
| 1 | `working_days.py` utility | Foundation; everything else depends on this |
| 2 | `dispatch_calculator.py` | Pure logic; write tests against the June 30 worked example from BRD |
| 3 | API endpoint | Connects the calculator to the UI |
| 4 | Client dialog + button on Lyfe Order | First visible result; CS can test end-to-end |
| 5 | `on_update` Urgent flag handler + PM task creation | Automation layer on top of confirmed UI |
| 6 | `promised_dispatch_by` field type: Data → Date | Low risk; enables date sorting and alerts |
| 7 | Exclusion clause in `promised_dispatch.py` | Must be done before Urgent orders go live, to avoid false SLA breaches |
| 8 | "At Risk" daily re-notification scan | Last — runs independently; can be added after launch |
| 9 | Quotation form support | Confirm in Open Questions first |
