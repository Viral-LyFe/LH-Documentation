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
| 5 | **Exclude Urgent from standard SLA metrics** (§3.1) | BR-2 live | Must happen the moment Urgent orders exist, to stop pollution |
| 6 | **Urgent SLA metric group + drill-through** (§3.2–3.3) | BR-2 live + Step 5 | Replaces the excluded urgent orders with a correct dedicated view |
| 7 | **Add Dispatch By / Days Left columns** to Priority View | BR-2 live | Completes the Priority View once dispatch-by data exists |

**Key sequencing rule:** Steps 1–4 deliver value **now** and do not depend on BR-2. Steps 5–7 must ship **together with or immediately after** BR-2 — never ship BR-2 without Step 5, or the analytics dashboard will display wrong SLA data from day one.

---

## 11. One-Paragraph Summary for Management

The two dashboards are already capable, but the upcoming **Reverse-SLA / Urgent feature (BR-2) will silently corrupt the SLA analytics dashboard** — urgent orders on compressed timelines would be measured against the wrong 3-/14-day rule, producing both false "on track" greens (hiding real lateness) and false breach reds (crying wolf). The BRD explicitly requires excluding urgent orders from standard SLA dashboards; we must honor that by adding an exclusion clause **and** a dedicated Urgent SLA card group that tracks them against their real dispatch-by dates. Separately, and independent of BR-2, we recommend adding a **priority-sorted order view** (Urgent → Low) on the operational board, plus **bottleneck/aging analysis** and **dispatch forecasting** on the analytics board, all backed by push **alerts** so managers are told where to act instead of having to watch the screen. The aging, forecasting, and priority-view work can ship first; the SLA-exclusion and Urgent-card work must ship with BR-2.
