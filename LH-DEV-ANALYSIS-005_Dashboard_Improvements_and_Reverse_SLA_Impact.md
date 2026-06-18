# Dashboard Improvements + Reverse-SLA Data Impact

**Doc ID:** LH-DEV-ANALYSIS-005  
**Related:** LH-DEV-ANALYSIS-BR2 (Reverse SLA), LH-BRD-DEV-004 v1.1  
**Date:** 2026-06-18  
**Prepared for:** Viral (ERPNext Dev), CS Team, Operations Manager  
**Status:** Proposal — review before implementation

---

## Who Should Read This

| Section | Audience |
|---|---|
| 1 — The Two Dashboards Today | Everyone |
| 2 — ⚠ Will Reverse SLA Pollute the Analytics Dashboard? | Everyone — **most important section** |
| 3 — Fix Plan for the SLA Dashboard | Developer + Manager |
| 4 — New Priority View on Status Overview | CS Team + Manager |
| 5 — Bottleneck & Aging Analysis | Manager |
| 6 — Forecasting / Throughput | Manager |
| 7 — Alerts: Who Gets Told What | Everyone |
| 8 — File Change Summary | Developer |
| 9 — Open Questions | Developer + Management |
| 10 — Recommended Build Order | Developer |

---

## 1. The Two Dashboards Today

### 1.1 Status Overview (`lyfe_orders_status_overview`) — the operational board

This is the **live "what's happening right now"** screen.

| What it shows | Detail |
|---|---|
| 6 summary cards | Count of orders Pending By: Customer Support, Factory, Shipping Forwarder, Engineer, Export Order, Completed |
| Live order table | Up to 2000 orders — Internal ID, Customer, Order Status, Workflow Status, Order Date, Expected Shipping Date, **Order Priority**, Tracking/Carrier |
| Filters | Date range, Status category, Show Completed checkbox, free-text search |
| Actions | Bulk "Mark Ready for Dispatch", per-order dispatch dialog, Excel/PDF export |
| Refresh | Auto-refreshes every 15 seconds |

**Note:** The table *already pulls* `order_priority` (line 132 / 223 in the .py) and *already displays* it as a column — but there is **no sorting, no grouping, and no filtering by priority** today. It just shows "Medium" as plain text.

### 1.2 Order Analysis TW (`order_analysis_tw`) — the SLA analytics board

This is the **"how are we performing"** screen — a Triple-Whale-style dashboard with dark/light themes. It re-uses all the data functions from the original `order_analysis` page.

It already computes ~40 metrics, including the SLA-critical ones:

| Metric (fieldname) | What it measures |
|---|---|
| `breach_promised_dispatch` | Orders that blew past the factory→dispatch SLA (3 wd standard / 14–16 wd custom) |
| `custom_dispatch_on_track / approaching / overdue` | Custom orders bucketed by days since factory acknowledgement |
| `india_factory_delivery_*` / `us_warehouse_delivery_*` | Transit SLA buckets |
| `customs_at_risk` | Orders stuck in customs |
| `on_time_delivery_rate` | % delivered within promised window |
| `avg_factory_dispatch`, `avg_end_to_end`, `avg_cs_review`, `avg_photo_approval` | Average cycle times |
| `revision_rate`, `photo_rejection_rate` | Quality / rework metrics |
| Pipeline funnel, category SLA, category trend, revenue by category, order-type split | Breakdown views |

This dashboard is genuinely strong. The problem is **not missing metrics** — it is that the reverse-SLA feature will **feed wrong numbers into the existing metrics** unless we change them. That is Section 2.

---

## 2. ⚠ Will Reverse SLA Pollute the Analytics Dashboard?

**Short answer: YES.** If we ship the BR-2 Reverse SLA / Urgent feature without changing this dashboard, several metrics will become **incorrect and misleading**. Here is exactly why and where.

### 2.1 The root cause

The standard SLA dashboard measures every order against the **forward SLA thresholds**:

- Standard order: must dispatch within **3 working days** of factory acknowledgement
- Custom order: within **14 working days** (16 if rejected twice)

But an **Urgent order runs on a compressed timeline** (BR-2). Its real deadline is the `promised_dispatch_by` date calculated backward from the customer's required delivery date — which may be **shorter OR longer** than the standard 3/14-day window.

The dashboard does not know about `promised_dispatch_by`. It will keep applying the standard 3/14-day rule to urgent orders. Result: **the dashboard lies about urgent orders.**

### 2.2 Concrete example of the wrong data

> **Urgent custom order**, customer needs delivery in 12 working days.  
> Reverse calc says: factory must dispatch by **Day 8**.  
> CS flags it Urgent, dispatch-by = Day 8.

**What the standard dashboard shows on Day 9:**
- Order is 9 working days since factory acknowledgement
- Standard custom threshold = 14 wd → **dashboard says "On Track"** ✅ (green)
- **Reality:** the order is already 1 day past its true urgent deadline → it should be **Overdue** 🔴

The factory sees green on the dashboard and relaxes. The customer misses their date. **This is the exact failure BR-2 was meant to prevent — but the dashboard would silently undo it.**

The reverse is also true:

> **Urgent standard order**, customer has a generous date — dispatch-by = Day 6.  
> Standard threshold = 3 wd. On Day 4 the dashboard flags it as a **breach** 🔴 — but it is actually **fine** (4 < 6). This inflates the breach count and triggers false alarms.

### 2.3 Exactly which metrics are affected

| Metric (in `order_analysis.py`) | What goes wrong | Severity |
|---|---|---|
| `breach_promised_dispatch` (+ standard/custom) | Counts urgent orders against the wrong threshold — both false positives and false negatives | 🔴 High |
| `custom_dispatch_on_track / approaching / overdue` | Urgent custom orders bucketed by standard 7/14-day bands, not their real dispatch-by date | 🔴 High |
| `on_time_delivery_rate` | Mixes urgent (compressed) and standard orders into one rate — the % becomes meaningless for both | 🟠 Medium |
| `avg_factory_dispatch` | Urgent compressed times drag the average down or up; not comparable to standard | 🟠 Medium |
| `india_factory_delivery_*`, `us_warehouse_delivery_*` | Transit buckets unaffected by urgency directly, but the "approaching/overdue" thresholds assume standard pacing | 🟡 Low |

**The BRD itself calls this out** (Edge Cases, Table 8): *"Urgent orders running on compressed timelines must be excluded from standard SLA-breach dashboards. The Urgent flag is the exclusion criterion."*

So this is not optional — it is a stated requirement that this dashboard currently does not honor.

---

## 3. Fix Plan for the SLA Dashboard

There are two things to do: **(A) stop the pollution**, and **(B) add a dedicated urgent lens + alerting** so urgent orders are still tracked — just correctly.

### 3.1 (A) Exclude Urgent orders from the standard SLA metrics

In every affected query in `order_analysis.py`, add an exclusion clause:

```sql
AND (order_priority IS NULL OR order_priority != 'Urgent')
```

This mirrors the exact same change being made in the `promised_dispatch.py` detector (see LH-DEV-ANALYSIS-BR2 §8.6). After this, the standard SLA cards (`breach_promised_dispatch`, `custom_dispatch_*`, `on_time_delivery_rate`, `avg_factory_dispatch`) measure **only standard-paced orders** — clean, comparable, trustworthy.

### 3.2 (B) Add a dedicated "Urgent SLA" metric group

Urgent orders still need monitoring — just against their **own** `promised_dispatch_by` date, not the 3/14-day rule. Add a new metric group to `get_dashboard_data`:

| New metric | Definition |
|---|---|
| `urgent_active` | Count of orders where `order_priority = 'Urgent'` and not yet dispatched |
| `urgent_on_track` | Urgent orders where `promised_dispatch_by` is **> 2 working days away** |
| `urgent_approaching` | Urgent orders where `promised_dispatch_by` is **within 2 working days** (At Risk) |
| `urgent_overdue` | Urgent orders where `promised_dispatch_by` is **in the past** and order not yet `ready_for_dispatch_date` |
| `urgent_avg_buffer_consumed` | Average working days of standard buffer being spent to hit urgent dates (shows factory pressure) |

These render as a new card row on the TW dashboard — visually distinct (e.g. a red/amber "URGENT" band) so it is never confused with the standard SLA cards.

### 3.3 Card drill-through

The TW dashboard already supports click-to-drill on cards (`get_card_orders`). Wire the new urgent cards to a drill query that lists the actual orders behind each count — so a manager clicking "3 Urgent Overdue" sees exactly which 3 orders, with their dispatch-by dates.

### 3.4 Before / After

```
BEFORE (wrong):
┌─────────────────────────────────────────────────┐
│  SLA Breach — Promised Dispatch:  12             │  ← includes 4 urgent
│  Custom Dispatch Overdue:          5             │     orders measured
│  On-Time Delivery Rate:           87%            │     against wrong rule
└─────────────────────────────────────────────────┘

AFTER (correct):
┌─────────────────────────────────────────────────┐
│  STANDARD SLA (urgent excluded)                  │
│  SLA Breach — Promised Dispatch:   8  (was 12)   │  ← clean
│  Custom Dispatch Overdue:          3  (was 5)    │
│  On-Time Delivery Rate:           91%            │
├─────────────────────────────────────────────────┤
│  🔴 URGENT SLA (own dispatch-by dates)           │
│  Active Urgent:        6                          │
│  On Track:             3                          │
│  ⚠ Approaching (≤2wd): 2   ← click to see orders │
│  🔴 Overdue:           1   ← click to see orders │
│  Avg buffer consumed:  4.2 wd                     │
└─────────────────────────────────────────────────┘
```

---

## 4. New Priority View on Status Overview

You asked for a **separate view where the user can see the order list sorted Urgent → Low priority.** Here is the plan.

### 4.1 What the user sees

A new value in the existing **Status filter dropdown** (or a separate toggle): **"By Priority"**. When selected, the order table:

1. **Groups and sorts** by `order_priority`: Urgent → High → Medium → Low
2. Adds a colored **priority badge** in the priority column (red / orange / blue / grey)
3. Adds two new columns for urgent orders: **Dispatch By** and **Days Left** (working days until `promised_dispatch_by`)
4. Highlights **At Risk** rows (Days Left ≤ 2 and not yet dispatched) with a red left border

### 4.2 Mockup

```
View: [ By Priority ▾ ]     Search: [____________]

PRIORITY   ORDER ID    CUSTOMER     STATUS            DISPATCH BY   DAYS LEFT
🔴 Urgent  ORD-014     ABC Corp     Factory Assign    June 19       1 wd  ⚠ AT RISK
🔴 Urgent  ORD-021     XYZ Ltd      Awaiting Upload   June 20       2 wd
🟠 High    ORD-008     Patel Co     Factory Assign    —             —
🟠 High    ORD-033     Mehta Inc    Ready for Disp    —             —
🔵 Medium  ORD-002     Shah Bros    Internal Review   —             —
⚪ Low     ORD-045     Gupta & Co   New               —             —
```

### 4.3 Why this is low-effort

- `order_priority` is **already fetched and displayed** — no backend query change needed for the basic sort
- Sorting/grouping is a pure front-end change in `render_table()`
- The only backend addition is pulling `promised_dispatch_by` and computing `Days Left` (one extra column in the existing `get_orders` SELECT)

### 4.4 Decision needed

Today `order_priority` options are **High / Medium / Low** (no "Urgent"). BR-2 introduces "Urgent". Confirm: do we **add "Urgent" as a new option** to the select field (recommended — keeps all four tiers), or repurpose "High" to mean urgent? This must be decided before either BR-2 or this view is built. See Open Questions §9.

---

## 5. Bottleneck & Aging Analysis

This is the highest-value *new* analytical capability — it answers **"where are orders getting stuck, and which specific orders are stuck longest?"**

### 5.1 Per-stage aging buckets

For each pipeline stage, show how many orders are sitting there and for how long:

```
STAGE                        0–2 wd   3–5 wd   6–10 wd   10+ wd   OLDEST
CS Review (New→Assigned)       8        3        1         0       4 wd
Photo Approval                 5        4        2         1       12 wd  ⚠
Factory (Assign→Dispatch)     12        6        3         2       18 wd  🔴
Awaiting Tracking              4        2        0         0       3 wd
In Transit                     7        5        1         0       8 wd
```

The "10+ wd" column and "OLDEST" column immediately surface the bottleneck stage. Red/amber flags draw the eye.

### 5.2 "Oldest stuck orders" table

A ranked list of the single oldest-sitting orders across all active stages, so the manager can act on the worst offenders first:

```
ORDER ID    STAGE              SITTING FOR   OWNER          PRIORITY
ORD-009     Factory            18 wd  🔴     factory_a      Medium
ORD-014     Photo Approval     12 wd  ⚠      cs_priya       Urgent
ORD-022     In Transit          8 wd         forwarder_1    High
```

### 5.3 Where the data comes from (mostly already exists)

The timestamps needed are **already on Lyfe Order**: `cs_team_first_action`, `factory_first_action`, `factory_assignment_date`, `ready_for_dispatch_date`, `out_from_factory`, `gate_pass_submitted_date`, `delivered`. The working-day math is already proven in `order_analysis.py` (the `_biz_days` / `_factory_days` helpers). This is largely **new SQL using existing columns + existing helpers** — no schema change.

### 5.4 Hold-time visibility

`total_hold_days` and `hold_start_date` are already tracked (and `on_hold_max_days`, `on_hold_over_7d` already computed). Surface them in the aging view so a manager can distinguish "stuck because of us" vs "stuck waiting on customer/payment hold."

---

## 6. Forecasting / Throughput

This answers **"what's coming, and can we handle it?"**

### 6.1 Expected dispatch load (next 7 / 14 days)

Using `promised_dispatch_by` (urgent orders) + `expected_shipping_date` + standard SLA projections (factory_assignment_date + 3/14 wd), project how many orders are **due to dispatch each day** for the next two weeks:

```
DISPATCH FORECAST (next 7 working days)
Mon 23   ████████  8 orders   (2 urgent)
Tue 24   ██████    6 orders   (1 urgent)
Wed 25   ████████████ 12 orders  ⚠ peak load
Thu 26   ███       3 orders
Fri 27   █████     5 orders
```

The peak-load day (Wed, 12 orders) tells the factory/CS to staff up or pull work forward.

### 6.2 Throughput vs demand

- **Throughput** = average orders dispatched per working day over the last 30 days (derived from `out_from_factory` dates)
- **Demand** = orders currently in factory that must dispatch in the same window

If demand for a window exceeds historical throughput, flag it: *"Next week needs 45 dispatches; your 30-day average is 30/week. Likely backlog."*

### 6.3 Expected completions this week

Count of orders in transit whose `promised_delivery_date` falls this week — gives CS a heads-up list for proactive customer communication.

### 6.4 Honesty caveat

Forecasting is **projection, not promise.** Every forecast card should carry a small note: *"Projected from current SLA pace; holds and rework will shift these."* This keeps managers from treating it as a guarantee.

---

## 7. Alerts — Who Gets Told What

Surfacing data on a screen only helps if someone is looking. Add **push alerts** for the conditions that need action, so managers don't have to babysit the dashboard.

| Condition | Who is alerted | When | Channel |
|---|---|---|---|
| Urgent order Approaching (≤2 wd to dispatch-by, not dispatched) | CS owner + Factory assignee + Manager | Daily morning scan | ERP notification (+ Slack — confirm) |
| Urgent order Overdue (past dispatch-by) | CS owner + Manager + Founders | Daily scan | ERP + Slack |
| Stage bottleneck: any order sitting 10+ wd in one stage | Manager + stage owner | Daily scan | ERP notification |
| Dispatch forecast peak: any day > throughput average | Manager | Weekly (Monday) | ERP notification |
| On-hold > 7 days (already computed) | CS owner + Manager | Daily scan | ERP notification |

These can piggyback on the **existing 15-min SLA scan** (`sla_scan.run`) or a new lightweight daily cron. The infrastructure (`SLA Escalation Log`, scheduler, notification patterns) already exists in `lh_project`.

---

## 8. File Change Summary

| File | Change Type | What Changes |
|---|---|---|
| `lh/lyfe_hardware/page/order_analysis/order_analysis.py` | **Edit** | (A) Add `order_priority != 'Urgent'` exclusion to all standard SLA queries. (B) Add new `urgent_*` metric group. (C) Add aging-bucket queries + forecast queries. (D) New drill-through queries for urgent + aging cards. |
| `lh/lyfe_hardware/page/order_analysis_tw/order_analysis_tw.js` | **Edit** | Render the new URGENT SLA card band, aging table, forecast chart, with theme support |
| `lh/lyfe_hardware/page/lyfe_orders_status_overview/lyfe_orders_status_overview.py` | **Edit** | Add `promised_dispatch_by` to the SELECT; compute Days Left; add "By Priority" sort option |
| `lh/lyfe_hardware/page/lyfe_orders_status_overview/lyfe_orders_status_overview.js` | **Edit** | Add "By Priority" view: grouping, colored badges, Dispatch By / Days Left columns, At Risk highlighting |
| `lh/lyfe_hardware/doctype/lyfe_order/lyfe_order.json` | **Edit** | Add "Urgent" to `order_priority` select options (if confirmed) |
| `hooks.py` / `lh_project` scan | **Possibly Edit** | Add the new alert conditions (urgent approaching/overdue, bottleneck, forecast peak) |

**Note:** Items (A) and the urgent group depend on BR-2 being built (it creates the `promised_dispatch_by` data and the "Urgent" priority). The aging and forecasting work in Sections 5–6 are **independent of BR-2** and can ship first.

---

## 9. Open Questions Before Building

| # | Question | Who Decides | Why It Matters |
|---|---|---|---|
| 1 | Add "Urgent" as a 4th `order_priority` option, or repurpose "High"? | CS / Srishti | Both BR-2 and the Priority View depend on this. Recommended: add "Urgent" so all four tiers exist. |
| 2 | Should the standard SLA cards show "(urgent excluded)" labels, or silently exclude? | Manager | Transparency vs clutter. Recommended: small label so managers know why the number dropped. |
| 3 | Alert channel — ERP notification, Slack, or both? | Founders / Srishti | Same open question as BR-2 §9.3. Decide once for both features. |
| 4 | Aging bucket thresholds (0–2 / 3–5 / 6–10 / 10+ wd) — are these the right bands? | Operations | Bands should match how the team thinks about "late." Easy to adjust. |
| 5 | Forecast horizon — 7 days, 14 days, or both? | Manager | Affects chart width and query range. |
| 6 | Does the Priority View replace the current default table view, or sit alongside it as an extra option? | CS Team | Recommended: extra option in the existing Status dropdown — non-disruptive. |
| 7 | Should aging/forecast respect the role-based filtering already in Status Overview? | Manager | Engineers currently see a restricted set. Decide if analytics honor the same restriction. |

---

## 10. Recommended Build Order

| Step | What | Depends On | Why This Order |
|---|---|---|---|
| 1 | **Priority View** on Status Overview (sort + badges, without dispatch-by) | Q1 (add Urgent option) | Smallest change, immediate value, no BR-2 dependency |
| 2 | **Aging buckets + Oldest-stuck table** on TW dashboard | None | Pure new SQL on existing columns; high manager value; BR-2-independent |
| 3 | **Forecasting / throughput** cards | None | Builds on existing date columns; BR-2-independent |
| 4 | **Bottleneck + on-hold alerts** | Steps 2 | Turns the new aging data into push notifications |
| 5 | **Exclude Urgent from ALL forward SLA queries** (§3.1 + §11.2) — dashboard AND all detectors (R1/R2/R3/R5/R7) | BR-2 live | Must happen the moment Urgent orders exist, to stop pollution. **Engine-wide, not dispatch-only** — a partial exclusion makes an urgent order vanish from one card while still breaching on another |
| 6 | **Urgent SLA metric group + drill-through** (§3.2–3.3) | BR-2 live + Step 5 | Replaces the excluded urgent orders with a correct dedicated view |
| 7 | **Add Dispatch By / Days Left columns** to Priority View | BR-2 live | Completes the Priority View once dispatch-by data exists |

**Key sequencing rule:** Steps 1–4 deliver value **now** and do not depend on BR-2. Steps 5–7 must ship **together with or immediately after** BR-2 — never ship BR-2 without Step 5, or the analytics dashboard will display wrong SLA data from day one.

---

## 11. SLA Rule Spec Alignment & BR-2 Edge Cases (from LH_PM_Rules_Developer_Spec)

The PM Rules spec defines the **forward SLA rules (R1–R7, Q1–Q2)** that already drive the `lh_project` SLA engine. These are the *authoritative thresholds, assignees, and auto-close conditions*. BR-2 (reverse SLA) and the dashboard exclusion logic must stay **consistent with these exact values** — otherwise the dashboard and the engine will disagree.

### 11.1 The forward SLA rules (already implemented — confirmed in code)

Each rule below maps to a detector that **already exists** under `lh/lh_project/sla/detectors/`:

| Rule | Trigger / Monitor state | SLA | Assign To | Auto-close when | Detector file (exists) |
|---|---|---|---|---|---|
| **R1** Standard Dispatch | `Factory Assignment` + `order_type=Standard` | **3 working days** | Factory (support@chandakbrothers.net) | `Ready for Dispatch` OR `Dispatched` | `promised_dispatch.py` |
| **R2** Custom Internal Review | `Factory Assignment` + `order_type=Custom` | **10 working days** | Factory | `Internal Review` or beyond | `internal_review.py` |
| **R3** Awaiting Tracking | state = `Awaiting Tracking` | **1 working day** | Factory | `Awaiting Shipping` OR `Shipped` | `awaiting_shipping.py` |
| **R4** Shipped → Delivered | state = `Shipped` | **9 calendar days** | CS (support@lyfehardware.com) | `Completed` | `shipped_to_completion.py` / `delayed_shipment.py` |
| **R5/R6** Engineer Internal Review | state = `Internal Review` | **1 working day** | Engineer (confirm w/ Careers) | `Size Approved` OR `Pending Customer Approval` (i.e. *changed from* Internal Review) | `internal_review_approval.py` |
| **R7** Pending Customer Approval | state = `Pending Customer Approval` | **4 working days** | CS | `Approved` | `pending_customer_approval.py` / `approval_pending.py` |
| **Q1** Drawing Required | Quotation `drawing_required → 1` | **1 working day due** | Engineer | `drawing_uploaded = 1` | (TAR + due date) |
| **Q2** Feasibility Required | Quotation `feasibility_test_required → 1` | **1 working day due** | Factory | `feasibility_complete = 1` | (TAR + due date) |

**Universal escalation (applies to R1–R7 and Q1–Q2):** if a task is **not actioned within 1 day** of creation, **reassign to Srishti (srishti@lyfehardware.com)** and **raise priority to High**. This is the existing escalation pattern (`escalation_scan.run`, hourly).

### 11.2 ⚠ Threshold discrepancy the dashboard must respect

The spec and the code use **two different custom thresholds for two different rules** — do not confuse them:

| Concept | Threshold | Source |
|---|---|---|
| Custom order → reach **Internal Review** | **10 wd** | Spec R2 + `internal_review.py` |
| Custom order → **Dispatch** (factory→out) | **14 wd** (16 if rejected >1×) | `promised_dispatch.py` (hardcoded) |

The dashboard's `custom_dispatch_*` buckets and `breach_promised_dispatch` use the **14/16** dispatch threshold. The `internal_review` breach uses **10**. When we add the `order_priority != 'Urgent'` exclusion (§3.1), it must be applied to **every** detector query and **every** dashboard SLA query that touches these — not just the dispatch one — so an urgent order is excluded consistently across *all* forward SLA rules, not partially.

**Action:** §3.1 exclusion clause must be added to the dashboard queries backing R1 (dispatch), R2 (internal review), R3 (tracking), R5 (engineer), R7 (customer approval) — wherever an urgent order could appear. A partial exclusion is worse than none (the order vanishes from one card but still shows as a breach on another).

### 11.3 BR-2 edge cases — coverage check against this spec

Confirming the BR-2 reverse calculator (LH-DEV-ANALYSIS-BR2) and these dashboard changes together cover every edge case. **Each row below is a stated edge case; the right column is where it is handled.**

| Edge case (from BRD Table 8 + PM spec) | Handled by | Status |
|---|---|---|
| Urgent orders excluded from standard SLA-breach dashboards | §3.1 exclusion + §11.2 (all detectors, not just dispatch) | ✅ Planned |
| Urgent order tracked against its own dispatch-by date | §3.2 Urgent SLA card group | ✅ Planned |
| US warehouse fulfilment (transit 3–4 wd, no customs/gatepass) | BR-2 calc US branch; dashboard `us_warehouse_delivery_*` already separate | ✅ Planned |
| Customs buffer toggle for sample shipments | BR-2 calc input `has_customs_buffer` | ✅ Planned |
| New product (die/mould) → always Not Possible | BR-2 calc `is_new_product` short-circuit | ✅ Planned |
| Dispatch-by date in the past/today → Not Possible | BR-2 calc runway check < 0 | ✅ Planned |
| Required date on weekend/holiday → snap forward | BR-2 `working_days.py` snap (IN + US calendars) | ✅ Planned |
| Photo-approval window inside compressed custom timeline | BR-2 calc photo_approval_wd step | ✅ Planned |
| **Client delay eats buffer → auto-flag At Risk + re-notify** | BR-2 §8.7 At Risk scan **+ this doc §3.2 `urgent_approaching` card + §7 alert** | ✅ Planned (now visible on dashboard) |
| Compressed timeline excluded from SLA breach reporting | §3.1 (the exclusion **is** the mechanism) | ✅ Planned |
| Order downgraded from Urgent → clean up task + indicators | BR-2 §8.5 downgrade handler; dashboard auto-recounts (no orphan) | ✅ Planned |
| Required delivery date changed post-placement → recompute | BR-2 §8.4 recompute on date change; dashboard reads new date | ✅ Planned |
| **Borderline runway (within 2 wd) → manual review, not hard fail** | BR-2 Borderline verdict **+ this doc §3.2 `urgent_approaching` = "within 2 wd"** | ✅ Planned — *note: §3.2 must use the same 2-wd boundary as BR-2's Borderline, see 11.4* |
| Out-of-stock standard order → treat as custom production time | BR-2 calc `is_stock_available=False` switches to custom path | ✅ Planned |
| **SLA task not actioned within 1 day → escalate to Srishti, raise to High** | Existing `escalation_scan` (hourly) — applies to urgent tasks too | ✅ Existing — verify urgent PM task is registered so escalation fires |
| Engineer email for R5/Q1 | **Open** — spec says "confirm with Careers" | ⚠ Blocked on confirmation |

### 11.4 Consistency rules the implementation must enforce

1. **One source of truth for "2 working days."** BR-2's *Borderline* threshold, the dashboard's `urgent_approaching` bucket, and the "At Risk" alert must all use the **same 2-working-day boundary**. Define it once (a constant in `working_days.py` / a config) and import it everywhere.
2. **Urgent PM task must be escalation-eligible.** The Urgent task BR-2 creates must follow the same R1–R7 escalation pattern: not actioned within 1 day → reassign to Srishti, raise to High. Confirm the urgent task is created with the fields `escalation_scan.run` looks for, or it will silently never escalate.
3. **Auto-close parity.** BR-2's urgent task auto-closes on `ready_for_dispatch_date` (same signal as R1). Keep this identical to R1 so an order does not show closed on one board and open on another.
4. **Exclusion applies engine-wide.** As §11.2 states — exclude urgent from R1, R2, R3, R5, R7 dashboard queries, not just dispatch.

### 11.5 Updated file-change note

Add to §8:

| File | Change Type | What Changes |
|---|---|---|
| `lh/lyfe_hardware/page/order_analysis/order_analysis.py` | **Edit (expanded)** | Apply the `order_priority != 'Urgent'` exclusion to **all** forward-SLA queries (dispatch R1, internal review R2, tracking R3, engineer R5, customer approval R7) — not only `breach_promised_dispatch` |
| `lh/lh_project/sla/detectors/*.py` | **Edit** | Same exclusion in each detector's `get_candidates()` (R1, R2, R3, R5, R7) so engine and dashboard agree |
| `lh/lh_project/sla/utils/working_days.py` | **New (shared constant)** | Define `BORDERLINE_WD = 2` (or config) and reuse for BR-2 Borderline, dashboard `urgent_approaching`, and the At Risk alert |

---

## 12. One-Paragraph Summary for Management

The two dashboards are already capable, but the upcoming **Reverse-SLA / Urgent feature (BR-2) will silently corrupt the SLA analytics dashboard** — urgent orders on compressed timelines would be measured against the wrong 3-/14-day rule, producing both false "on track" greens (hiding real lateness) and false breach reds (crying wolf). The BRD explicitly requires excluding urgent orders from standard SLA dashboards; we must honor that by adding an exclusion clause **and** a dedicated Urgent SLA card group that tracks them against their real dispatch-by dates. Separately, and independent of BR-2, we recommend adding a **priority-sorted order view** (Urgent → Low) on the operational board, plus **bottleneck/aging analysis** and **dispatch forecasting** on the analytics board, all backed by push **alerts** so managers are told where to act instead of having to watch the screen. The aging, forecasting, and priority-view work can ship first; the SLA-exclusion and Urgent-card work must ship with BR-2. Crucially, after cross-checking the PM Rules spec (R1–R7), the urgent exclusion must be applied **engine-wide across every forward SLA rule** — dispatch, internal review, tracking, engineer review, and customer approval — not just the dispatch metric; a partial exclusion would make an urgent order disappear from one card while still firing a false breach on another, and all components (BR-2's Borderline, the dashboard's "approaching" bucket, and the At Risk alert) must share a single 2-working-day boundary so the engine and the dashboards never disagree.
