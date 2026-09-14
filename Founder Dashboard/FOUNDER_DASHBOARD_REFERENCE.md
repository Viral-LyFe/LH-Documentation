# Founder Dashboard — Complete Reference

**One document, functional + technical.** Each tile below is explained twice, back to back: first in plain language (what it means, no technical knowledge needed), then immediately under it, how it's actually built (files, functions, SQL, data source, known caveats). Part 2 covers everything that isn't tile-specific — architecture, access control, settings, endpoints, and open issues.

**Location:** `lh/lyfe_hardware/page/founder_dashboard/`
**Page route:** `/app/founder-dashboard`
**Doctype:** Desk `Page`, name `founder-dashboard`
**BRD reference:** LH-BRD-DEV-009 deliverable 4, Annex A (iteration 2, 07 Aug 2026 memo)
**Also see:** `FOUNDER_DASHBOARD_USER_GUIDE.md` (same folder) — a plain-English version of this page, safe to share with end users directly.

---

## Contents

**Part 1 — Every tile**
1. [Are we making money?](#i-are-we-making-money)
2. [What's selling — and what's dying?](#ii-whats-selling--and-whats-dying)
3. [Where am I bleeding?](#iii-where-am-i-bleeding)
4. [Is the team keeping up?](#iv-is-the-team-keeping-up)
5. [What's coming?](#v-whats-coming)
6. [Who's connected right now? (Super Admin only)](#vi-whos-connected-right-now-super-admin-only)
7. [How the filters at the top work](#how-the-filters-at-the-top-work)
8. [If a number ever looks wrong](#if-a-number-ever-looks-wrong)

**Part 2 — How the page is built**
9. [Files in this folder](#9-files-in-this-folder)
10. [Access control](#10-access-control)
11. [Request flow (client → server)](#11-request-flow-client--server)
12. [Founder Dashboard Settings (tunable thresholds)](#12-founder-dashboard-settings-tunable-thresholds)
12a. [Per-tile filters](#12a-per-tile-filters-tile-level-refresh--custom-filters)
13. [External dependencies](#13-external-dependencies-other-modules-this-page-calls-into)
14. [Whitelisted endpoints — full list](#14-whitelisted-endpoints--full-list)
14a. [Independent per-tile endpoints](#14a-independent-per-tile-endpoints-tile-level-refresh--custom-filters)
15. [Client-side architecture](#15-client-side-architecture-founder_dashboardjs)
15a. [Tile-level refresh & filter infrastructure](#15a-tile-level-refresh--filter-infrastructure-added-2026-08-27)
15b. [Config-driven layout](#15b-config-driven-layout-founder-dashboard--true-modularity-added-2026-08-27)
16. [Known open items / follow-ups](#16-known-open-items--follow-ups)

---

# Part 1 — Every tile

The dashboard is organized around five questions a founder actually asks, not around which system the data lives in. Every number is pulled live from the same records the team already works in every day — orders, quotations, tasks, stock — nothing here is a separate, hand-kept spreadsheet.

If a tile is clickable on the live dashboard, clicking it opens the exact list of records behind that number (see "drill-down" below each tile for which server function backs that dialog).

---

## I. Are we making money?

Four tiles across the top row (`fd-grid`), covering revenue, margin, cash actually collected, and custom-order profitability.

### Revenue · MTD (Month to date)

**What it is:** Total sales booked so far this calendar month, compared to last month and to your monthly sales target.

**How it's built (plain):**
```
Revenue = Σ (order total − any Shopify refund)
          for every order dated from the 1st of this month through today
```

**Important nuance — "paced" target:** The color isn't judged against the *full* month's target — it's judged against a **paced** target: the monthly goal scaled down to "what we should have by today's date." On day 7 of a 31-day month, the bar is roughly 7/31 of the full target, not the whole thing. Without this, every tile would show red for the first two or three weeks of every month simply because the month isn't over yet.

**Color rule:** 🟢 On track — at/above paced target · 🟡 At risk — 80%+ of paced target · 🔴 Off track — below 80%.

**Also shown:** order count, % change vs. last month, and % change vs. the same period last year (hidden if last year's base was too small to make a meaningful percentage).

**Technical:**
- Function: `_sales_tile(settings, period_filters, channel, order_type, category)` in `founder_dashboard.py`.
- Plain-MTD/no-filter path delegates to `pnl_dashboard.get_summary()`; any Channel/Order Type/Category filter or non-MTD period falls back to `_build_where_v2()` + `_aggregate_rows(_fetch_orders(...))`.
- MoM/YoY deltas and the paced-target RAG logic **only compute on the plain-MTD path** — a filtered/non-MTD window has no single well-defined "prior period" to diff against, so those fields come back `None` rather than comparing the wrong two windows.
- YoY is suppressed entirely if the prior-year base was below `_YOY_MIN_PRIOR_REVENUE = 5000` (a tiny base can turn a normal dollar increase into a misleading "+2000%").
- Pacing formula: `_paced_target()` — `target × today.day / days_in_month`. Deliberately simple linear pacing, not a weighted historical curve.
- Drill-down: `get_sales_drilldown(global_filters)` — same filter-rail state as the tile.

---

### Gross Margin (Month to date, in practice)

**What it is:** What share of revenue is left over as profit after direct costs — manufacturing, shipping, custom fees, payment processing, tax.

**How it's built (plain):**
```
Direct cost = Manufacturing cost + Custom/additional charges + Shipping (India + US leg)
              + Payment processing fee + Tax
Profit      = Revenue − Direct cost
Margin %    = Profit ÷ Revenue
```

**Why "in practice" is MTD, not last month:** The backend function has a documented fallback to last full calendar month (see Technical below), motivated by shipping cost usually being missing on very recent orders (which inflates margin). **That fallback is dead code on the live page** — the JS always sends an explicit `{preset: "mtd"}` on every request, so the fallback's `period_filters is None` condition is never true in normal use. Confirmed via full-audit 2026-08-27: the number shown on this card is genuinely Month-to-Date, carrying the exact early-month-inflation risk the fallback was written to avoid. Not yet fixed — flagged as a known caveat below (§16), not a bug in the *math* (the MTD number itself is correctly computed for the window it uses).

**Color rule:** 🟢 On track — at/above target (default 20%) · 🟡 At risk — at/above 15% · 🔴 Off track — below 15%.

**vs. last full month (added 2026-08-27):** A `▲/▼ vs last full mo.` line under the profit/revenue sub-line — the current MTD margin % compared against the **last fully completed calendar month's** margin %, not a same-day-range slice of last month (deliberately different from Sales' `mom_pct`, which compares MTD-so-far vs. the same day-range last month — confirmed by design: a same-day-range prior slice would carry the identical early-month-cost-missing risk on both sides of the comparison, telling a founder nothing useful; a full prior month is a stable reference point instead).

**Technical:**
- Function: `_margin_tile(settings, period_filters, channel, order_type, category)`.
- Defaults to **last full calendar month** only when `period_filters is None` — legacy default, currently unreachable from the live page (see above).
- Root cause of the original default, confirmed on staging: `shipping_charges` was missing on ~100% of that month's orders 5 days in, vs. ~19% missing for the prior full month — still true, just not actually protecting against it today.
- Same `_build_where_v2()` / `_aggregate_rows()` chain as Sales.
- `mom_pct`: computed only on the plain-MTD path (period preset `mtd`, Channel/Order Type/Category all `"All"`) — a second `_fetch_orders()`/`_aggregate_rows()` call against the explicit last-full-month window, then `_pct_change(current_margin, prior_margin)`. `None` on any filtered/non-MTD path, same restriction `_sales_tile`'s `mom_pct`/`yoy_pct` already use.
- Drill-down: `get_margin_drilldown(global_filters)` — delegates to `pnl_dashboard.get_orders_detail()` (same columns as the P&L Dashboard's own Order Details table); Category is applied as a post-filter on that function's already-returned `item_group` field, since `get_orders_detail()` has no Category parameter of its own. The drilldown still reflects whichever window the tile itself is currently showing (MTD, by default) — not the new `mom_pct` comparison window.

---

### Cash In · MTD (Month to date)

**What it is:** Money that has actually landed — real payments received this month, not invoices merely issued.

**How it's built (plain):**
```
Cash In = Σ payment amount, for every payment recorded against a Quotation this month
```

**Note:** Deliberately does **not** use the outstanding-balance figure on invoices (that field has been unreliable). Only counts money a real Shopify charge or bank transfer confirms was received. Same paced-target coloring as Revenue.

**Technical:**
- Function: `_cash_in_tile(settings, period_filters)` — raw `SUM(Quotation Payment Entry.payment_amount)` by `payment_date`.
- Channel/Order Type/Category are **deliberately not applied** — `Quotation Payment Entry` is a child table on `Quotation`, which has no established join back to `Lyfe Order.order_source`/`order_type`/`item_group` anywhere in the app. Only Period is honoured.
- Paced-target RAG only applies on the `"mtd"` preset (pacing a monthly target only makes sense against month-to-date).
- Drill-down: `get_cash_in_drilldown(global_filters)` — Period only, same reasoning.

---

### Custom Rev · MTD

**What it is:** Revenue from custom (quote-built) orders specifically, plus how their margin compares to standard, off-the-shelf orders.

**How it's built:** Same revenue/margin math as above, split into **Custom** (from a Quotation) vs. **Standard** buckets. No target/color — it's context for the Revenue/Margin tiles, showing whether growth is coming from higher-margin custom work or lower-margin standard catalog sales.

**Technical:**
- Function: `_custom_revenue_tile(period_filters, channel, category)`.
- No-Category path delegates to `pnl_dashboard.get_order_type_breakdown()` (which groups `Lyfe Order` rows directly and has no Category concept).
- Category-filtered path falls back to a local `_build_where_v2()` query, grouped locally by `order_type` — still the same `_aggregate_rows()`/revenue formula, just re-grouped, never a second definition of margin.
- No dedicated drill-down endpoint.

---

### Month-on-Month (added 2026-09-03)

**What it is:** A trailing monthly trend of Lyfe Order revenue and margin %, independent of the dashboard's global Period filter — a "how has the business moved month to month" view alongside the point-in-time tiles above. Modeled on `quotation_analysis_dashboard.py`'s Month-on-Month tile, adapted from Quotation value/win-rate to Lyfe Order revenue/margin.

**How it's built:** One bar (Revenue) + one line (Margin %, rescaled into the revenue bar's range — `frappe.Chart` has no true second y-axis, so the real % is restored via the tooltip formatter) per completed calendar month. A **6M / 12M** toggle (default 12M) or a specific **year** selects the window; only months with real order data get a row (no zero-padding for a business too young to fill the window).

**The current (in-progress) month is never plotted as if it were complete** — it's excluded from the bar/line series entirely and shown instead as a separate "(MTD, in progress)" callout number beside the chart, naming the partial revenue/margin so far. This is the fix for the underlying bug the tile exists to avoid: a 2-of-30-day partial month plotted next to completed months would visually read as a sudden decline.

**Technical:**
- Backend: `_founder_mom_window(months, year)` — same window-boundary arithmetic as `quotation_analysis_dashboard._mom_window()` (trailing N months ending today, or a specific calendar year Jan–Dec / Jan–today), reimplemented (not imported) since the two tiles populate different doctypes. `_founder_mom_summary(months=12, year=None)` loops one calendar month at a time through the window, delegating to this file's own `_build_where_v2()`/`_fetch_orders()`/`_aggregate_rows()` — the same Lyfe Order revenue/margin formula every other tile on this page uses (BR-3), never a second metric definition. Returns `{months: [{period, label, is_mtd, revenue, margin_pct, revenue_mom_pct?, margin_mom_pct_pts?}], period_label}` — the deltas (`_pct_change`, no `_YOY_MIN_PRIOR_REVENUE` floor — a MoM comparison is between two similarly-sized recent months, not a YoY comparison against a much smaller prior-year base) are only computed on the latest row.
- Endpoint: `get_tile_mom(months=12, year=None)` — same `_check_permission()` gate as every other `get_tile_*` function.
- Self-check: `_mom_self_check()` in `founder_dashboard.py` — assert-based, not wired into any Frappe hook; run manually to sanity-check the window-boundary logic (a 6-month trailing window and a specific past year).
- **No standard Period/Channel/Order Type/Category filters** — this tile has its own months/year control instead (`TILE_SUPPORTED_FIELDS.mom = ["months", "year"]`), same precedent as Material Usage's own `period_days` selector. `TILE_ENDPOINTS.mom = "get_tile_mom"`.
- JS: `renderMom(d)` + `renderMomChart(months)` (the grouped bar/line chart, ports `quotation_analysis_dashboard.js`'s `renderMomChart`) + `renderMomTable(months, expanded)`. The 6M/12M toggle (`#fd-mom-toggle .fd-mom-chip`) sets `tileState.mom.filters = {months}` and calls `refreshTile("mom")` — the same independent per-tile refresh mechanism every other split-out tile uses, not a bespoke fetch.
- **Table collapsed to last 3 months by default (added 2026-09-03):** `renderMomTable()` shows only `months.slice(-3)` unless expanded, with a **"view detail →"**/"hide detail ▴" toggle (`#fd-mom-table-toggle`, same `.fd-cc-expand-toggle` styling as Trending Categories/Customer Concentration) only shown when there are more than 3 rows to hide. `bindMomTableToggle()` re-renders just `#fd-mom-table-wrap` — not the whole tile — on click, and resets to collapsed whenever the 6M/12M window changes (a fresh window shouldn't inherit the old one's expanded state).
- No dedicated drill-down endpoint; the tile itself is the detail view.

---

## II. What's selling — and what's dying?

Best Sellers and Slow Movers side by side (`fd-g2`), then Trending Categories as a full-width panel below (`fd-g1`).

### Best Sellers

**What it is:** Top products for the selected period, switchable between ranking by revenue, units sold, or margin — because the best-selling item is often **not** the most profitable one.

**How it's built (plain):** Grouped by SKU from actual order line items. Cost is split across a SKU's lines in proportion to its share of that order's total revenue — the system tracks cost per order, not per individual line, so this is the fairest available split.

**Technical:**
- Function: `_best_sellers_section(period_filters, channel, order_type, category)`.
- Raw SQL over `ShipStation Order Item` (`order_items` child field only, `adjustment=0`), grouped by `sku`.
- Margin % per SKU is **pro-rated**: since COGS/shipping are only tracked at order level, each SKU's line revenue is divided by that order's *total* line revenue to derive its cost share — the same simplification `pnl_dashboard.get_category_breakdown()` makes at the item_group level, one level finer here.
- Fee/payment rows ("Custom Fee", "Additional Payment for LH#...") are excluded from the SKU results themselves via `_exclude_fee_rows_sql()` (shared with `pnl_dashboard.py`, since ShipStation's own `adjustment` field does **not** flag these rows — confirmed live, every sampled fee row had `adjustment=0`) — but deliberately **kept in the proration denominator**, since a fee still represents real dollars the customer paid and dropping it would overstate each real line's cost share.
- No dedicated drill-down endpoint (the panel itself shows the ranked list directly).

---

### Slow Movers

**What it is:** Products tying up the most inventory value while barely selling — clearance/bundle candidates.

**How it's built (plain):**
```
Inventory value = stock on hand × cost per unit
                   ranked highest to lowest, for SKUs that sold in the selected period
```

**Note:** Ranked by **dollars tied up**, not low sale count — 2 units sold with $8,000 of stock sitting idle matters more than 2 units sold of a $10 item.

**Technical:**
- Function: `_slow_movers_section(period_filters, channel, order_type, category, limit=10)`.
- Sold-units query: same `ShipStation Order Item` join Best Sellers uses.
- Inventory value: `tabBin.actual_qty × tabBin.valuation_rate` (ERPNext's own per-item/warehouse stock-value rollup — not a new calculation).
- Only SKUs with `inventory_value > 0` are returned, sorted descending.

---

### Trending Categories

**What it is:** A full performance-and-trend view per product category — not just "up or down," but *why* the business is changing shape month to month.

**Collapsed by default (changed 2026-09-03):** The tile now shows only a compact summary — the top 3-4 categories by revenue this period, each with a MoM arrow — plus a **"view detail →"** toggle. Clicking it expands a detail panel containing everything below (an anchor Revenue & Margin chart, Monthly Category Performance, Total Revenue by Month, Revenue Mix, What's Driving The Change, Category Momentum, Category Health) — nothing was removed, it's just deferred behind the toggle so the main dashboard reads as a glance, not a wall of charts. Same collapse/expand mechanics as Customer Concentration's "Full Analysis" panel (`.fd-cc-panel`-style, deferred chart mount on expand — `renderTrendingCategoriesCharts()` only runs once the panel is actually in the DOM). The summary's MoM arrow reuses `monthly.categories[].months` (last two entries) — the same monthly-history data the detail panel's table already fetches, not a second query. A near-zero-base % (e.g. prior month was $0) is suppressed via the client-side `_fdPctChange()` mirror of `_pct_change()` and shown as "was $0 → now $Y" instead of a meaningless badge — same guard now applied to the Monthly Category Performance table's per-category trend arrow.

**Anchor chart — Revenue & Margin, Trailing Months (added 2026-09-03):** The detail panel's first chart is a combined bar(revenue)+line(margin %) view over the same trailing-months window as Monthly Category Performance — the exact combined-chart technique (rescale the line series into the bar range, restore the real % via `tooltipOptions.formatTooltipY`) the standalone Month-on-Month tile uses, reused rather than reimplemented. Sourced from `monthly.order_level_totals`/`monthly.order_level_margins` — `_monthly_order_level_revenue()` now returns both a revenue-per-month dict and a margin-per-month dict from the same per-month `_aggregate_rows()` call it already made (no second query loop).

**Label rename (2026-09-03):** All user-facing "Item Group" text on this page now reads "Category" (headers, table columns, dialog titles) — the underlying `item_group` fieldname/DB column is unchanged.

**Revenue vs. previous period (apples-to-apples):** Compares the selected period to the **immediately preceding period of the same length** — e.g. Aug 1–21 vs. Jul 11–31 (21 days each), not a fixed 30-day window. An earlier version compared against a fixed trailing 30 days regardless of the selected width, which could show a category as "declining" purely because the comparison window happened to be longer.

**Monthly Category Performance:** A month-by-month table (last 6 months by default) for top categories — sustained growth, a temporary spike, a seasonal dip, or a brand-new category, at a glance.

**Revenue Mix:** Each category's share of total revenue, this period vs. prior — catches a category whose revenue drops while its *share* of the (shrinking) business actually rises.

**What Is Driving The Change?:** Categories with the biggest dollar swing (up or down), and what % of the total company revenue change each accounts for — a different, usually more useful question than "which category grew fastest in percentage terms."

**Category Momentum — three buckets:**
- 🚀 **Rising** — meaningful growth (15%+) vs. prior, large enough in dollar terms to be a real signal.
- 🔻 **Declining** — the mirror case.
- 🆕 **Emerging** — little-to-no revenue in the prior period, real revenue now. Kept separate because a percentage from almost nothing isn't meaningful.

**Category Health:** Revenue, quantity sold, distinct customers, and average revenue per customer per category — the same customer-count definition used on the Customer Concentration tile.

**Full-month projection (Month-to-date only):** A separate call from the "vs previous period" comparison above — it answers a different question, "at this pace, will this month beat or miss last month's full total?" Takes this month's revenue-so-far and paces it up to a full-month equivalent (today's revenue × days-in-month ÷ days-elapsed), then compares that projection to last month's completed total. Added because a naive partial-vs-full-month comparison answers this wrong: 24 days of August ($105k) against the full 31 days of July ($128k) reads as −18% purely because July had more selling days, while the paced projection for the same data showed +3.7%. Only shown on the Month-to-date period — a projection is meaningless for an already-complete period ("Last full month") or a non-month-shaped window ("Last 90 days", custom range). The monthly history chart's last (partial) point carries a note pointing back to this projection.

**Note:** All thresholds (Rising/Declining/Emerging cutoffs, months of history shown) are configurable in Founder Dashboard Settings.

**Technical:**
- Function: `_trending_categories_section(settings, period_filters, channel, order_type, category)` — the single most complex section on the page. Rewritten 2026-08-21 per founder feedback from a simple "revenue + 30-day arrow" tile (built 2026-08-05).
- **Equal-length comparison**, computed inline: `span_days = (to_d - from_d).days + 1`, `prev_to = from_d - 1 day`, `prev_from = prev_to - (span_days - 1) days`. The exact same technique is reused in `_customer_concentration_section_v2`'s `previous_period` comparison.
- Sources: `pnl_dashboard.get_category_breakdown()` (current + prior), `get_category_monthly_trend()` (monthly history), `get_item_group_revenue_summary()` (customer counts — shared with Customer Concentration, so the two tiles can never silently disagree on how many customers bought a category).
- Momentum thresholds (all Settings-backed, see §12):
  - **Emerging** takes priority: `prev_rev < trending_categories_emerging_prior_max (500)` AND `rev >= trending_categories_emerging_current_min (2000)`.
  - **Rising:** `prev_rev >= trending_categories_min_prior_revenue (1000)` AND `pct >= 15`.
  - **Declining:** same floor AND `pct <= -15`.
  - Everything else is Stable/unranked — still shown in the main matrix.
- **Growth Contributors** = rows sorted by `abs(growth_delta)` descending, top 5 — independent of the %-based buckets above.
- **Full-month projection** (`full_month_projection`, added 2026-08-24): only computed when `preset == "mtd"`. `projected_total = total_current_revenue × days_in_month / days_elapsed` (same linear pacing `_paced_target()` uses for the Sales tile's target, inverted — projecting an actual figure forward instead of scaling a target back), compared against the full prior calendar month's total (a fresh `get_category_breakdown()` call for that window, same Category filter applied). `None` on every other period preset.
- No dedicated drill-down endpoint; the panel itself is the detail view.

---

### Where the Revenue Is (Geography) — ⚠️ added to this doc, previously undocumented

**What it is:** Revenue and margin broken down by ship-to country and by ship-to state — the one tile on this page that answers *where* the business is, as distinct from Customer Concentration's *who*.

**How it's built (plain):** Every order's shipping address already carries a country and a state — this tile groups the same period's orders by those two fields (independently, not nested) and runs the exact same revenue/margin math every other money tile on this page uses, so a country's numbers here are never a different definition of "revenue" than the Sales tile's.

**Technical:**
- Function: `_geography_section(period_filters, channel, order_type, category)`; independent endpoint `get_tile_geography(period, channel, order_type, category)`; registered in `TILE_REGISTRY["geography"]` and the JS's `TILE_ENDPOINTS`/`TILE_RENDERERS` (`renderGeography`).
- Both levels read straight off `Lyfe Order.ship_to_country`/`ship_to_state` — no Address join needed (unlike the Quotation Analysis dashboard's equivalent), since these are populated on 2743/2744 and 2738/2744 orders respectively as of the last check. State codes are already clean 2-letter values, grouped as stored (no "CA"/"California" split to worry about).
- Each level (`countries`, `states`) is aggregated independently through the shared `_aggregate_rows()` helper — the same one every other money tile calls — including the reshipment-cost LEFT JOIN (`Lyfe Order Reshipment`, non-cancelled only) folded into `custom_charges`, same as Sales/Margin/Custom Rev.
- Both lists total the full period's revenue independently — a country row is the roll-up of its own state rows, not a sibling row next to them.
- Both lists are returned in full (no truncation server-side) — the client's donut chart caps itself at 8 slices, but the table beneath it scrolls to show every region.
- Filterable by Channel / Order Type / Category, same as Sales (`TILE_FILTER_FIELDS.sales()`).
- No dedicated drill-down endpoint; the panel itself is the detail view.
- Lives in Section II ("What's selling — and what's dying?") as a full-width panel below Trending Categories.

---

## III. Where am I bleeding?

Three panels (`fd-g3`) turning operational alerts into dollar figures — "9 alerts" doesn't tell you anything; "$18,000 across 16 orders" does.

### Money at Risk (Live)

**What it is:** Total dollar value currently sitting in orders that are stuck, on hold, possibly lost in transit, or waiting on a Customer Service action — not a period total, what's true right now.

**The four buckets:**
- **Dispatch breaches** — orders stuck past their factory deadline.
- **On hold / delivery exception** — orders currently paused or flagged as an exception in transit.
- **Possible lost / never scanned** — shipments the tracking provider has flagged as exceptioned, expired, or not showing movement.
- **Pending CS action** — orders waiting on a Customer Service decision.

**Note:** Each bucket's dollar figure is the summed order value (minus any Shopify refund already issued) of every order in that bucket. Always live — not filtered by the Period selector, since these are point-in-time problems, not a monthly total.

**Technical:**
- Function: `_money_at_risk_section()`.
- Sources: `order_analysis.get_dashboard_data()` + `_get_factory_stuck_orders()` + three name-resolution helpers (`_get_order_names_by_flag`, `_get_order_names_by_tracking_alert`, `_get_order_names_pending_cs_action`), each summed via `_sum_order_amounts()` against `Lyfe Order.total_amount − shopify_refund_amount`.
- **Bug history:** `_get_order_names_pending_cs_action()` originally guessed the population directly (`workflow_state = 'CS Team Assignment'`) — that string **is not a real `workflow_state` value on this site** (confirmed via a live `GROUP BY` scan), so it silently matched zero rows and the tile showed a non-zero count next to a **$0** figure. Fixed by delegating to the canonical `order_analysis.get_card_orders("pending_cs_action", based_on="Live Data")` — the exact query that produces the count in the first place. **Never reintroduce a second, guessed population here.**
- No dedicated drill-down endpoint (each bucket's underlying dashboard has its own).

---

### Stockout Risk (Live)

**What it is:** Items projected to run out of stock soon, based on trailing 30-day sale velocity — and, for the riskiest, whether they're actually blocking real open orders right now.

**How it's built (plain):**
```
Days of cover = current stock ÷ average daily usage (trailing 30 days)
```

**Note:** Each critical item shows a note like "blocks $2,400" if open orders (not yet shipped) currently need it — distinguishing an item that's technically low but idle from one that's actually about to hold up real customer orders.

**Technical:**
- Function: `_stockout_risk_section_v2()`.
- Base data: `founder_briefing.data.get_projected_stockouts_api()` (same function/thresholds the Daily Founder Briefing uses — this card can never disagree with that briefing).
- `blocked_amount`/`blocked_orders` added via a join against open (`status NOT IN Cancelled/Merged/Split/Shipped/Completed`) `Lyfe Order`s currently containing that SKU.
- Drill-down: `get_stockout_risk_drilldown()` — full unfiltered list (the card itself shows only the top 10).

---

### Material Usage · 30d

**What it is:** What's actually being pulled off the shelf and used in production, trailing 30 days.

**How it's built (plain):**
```
Net qty = material issued to production − material returned
```

**Note:** Comes from the same paperwork the factory floor already fills out (Material Issue for Order) — nothing new was set up to track this.

**Technical:**
- Function: `_material_usage_section(period_days=30)` — `Material Issue Order Item` JOIN `Material Issue for Order` (submitted only, `docstatus=1`).
- Rejected alternative sources during design (documented in code): the `Material` doctype (unused 9-row lookup, zero real transaction references), Lyfe BOM (no material-type field), Stock Ledger Entry (too broad — includes non-order stock adjustments, not issuance specifically tied to an order).
- Drill-down: `get_material_usage_drilldown(period_days=30)` — full list (card shows top/bottom 10 only).

---

## IV. Is the team keeping up?

Three panels (`fd-g3`) on internal execution — separate from customer-facing production.

### Board Health (Live)

**What it is:** Each internal team board (CS, Factory, Engineering, etc.) shown side by side with its own open/overdue task count — kept separate on purpose so you can see exactly which team needs attention, not one collapsed company-wide number.

**How it's built (plain):**
```
Overdue = tasks still open whose due date has already passed, counted per board
```

**Technical:**
- Function: `_board_health_section()`.
- "Board" = `Project` (the PM module's grouping unit). Raw `Task` GROUP BY `project`.
- No dedicated drill-down endpoint.

---

### PM Task Risk (Live)

**What it is:** Internal team tasks at risk of breaching their service-level target — separate from customer order delays, since this tracks the team's own work commitments.

**How it's built:** Counts of tasks flagged Critical, Escalated, or Active against internal SLA rules, plus a trend line on whether on-time performance is improving or slipping.

**Technical:**
- Function: `_sla_risk_section()` (same function backs both this card and PM Task Risk — see §13). Task-level portion sources `pm_operations_dashboard.get_sla_risk_data()`.
- Also carries order-level (CS/Factory/photo-approval queues, dispatch breaches) and hold/tracking data from `order_analysis.get_dashboard_data()` — 3 distinct risk populations kept separate on purpose (different failure modes, different owners), even though this one function returns all three.

---

### Customer Concentration (Live · trailing 30 days by default, filter-rail aware)

**What it is:** How dependent the business currently is on a small handful of accounts, and whether growth is coming from new customers or the same repeat buyers.

**How it's built (plain):**
```
Top-5 share = revenue from your 5 biggest customers ÷ total revenue (selected period)
```

**Technical:**
- Function: `_customer_concentration_section_v2(period_filters, channel)` — expanded substantially 2026-08-21 (Phase 1, founder feedback) beyond the original top-customer/top-5 card.
- Sources: `pnl_dashboard.get_top_customer`, `get_top5_concentration`, `get_repeat_customers` (new-vs-returning split), `get_top_customer_item_group_breakdown`, `get_revenue_concentration_curve` (Pareto-style curve), `get_item_group_revenue_summary`, `get_customer_item_group_matrix` (limit 15).
- Also bundles a `previous_period` comparison via `_customer_concentration_period_kpis()`, using the same equal-length-window technique as Trending Categories.
- One combined payload — so the dashboard's expandable panel opens without a second round trip.
- Has its own drill-down machinery (`openTopCustomerItemGroupDialog`, `openCustomerItemGroupMatrixDialog` in the JS), separate from the generic `DRILLDOWN_CONFIG` table pattern, since its data shape (breakdown/matrix) doesn't fit a flat row list.
- **"Full Analysis" panel defaults to COLLAPSED (changed 2026-09-03)** — previously expanded by default; now matches Trending Categories' "view detail →" convention (same toggle text, same click mechanics). The panel's contents (and its item-group matrix chart) are only rendered on toggle-open, not eagerly on every page load.

---

## V. What's coming?

Two panels (`fd-g2`) looking forward.

### Look Ahead · Next 30 Days

**What it is:** A forecast of expected sales revenue for the next 30 days, built from historical demand patterns — directional, not a guarantee.

**How it's built (plain):**
```
Projected revenue = forecasted units per category × that category's average selling price
                     (trailing 90 days)
```

**Note:** Also flags how many SKUs are projected to go critical within 7 days, and how many of those have no restock order in progress.

**Technical:**
- Function: `_look_ahead_section()`.
- Unit forecast: `forecasting.exponential_smoothing.get_item_group_demand_forecast(history_months=6)` — that module only forecasts **units**, not revenue, so the $ conversion (units × trailing-90-day average unit price per item_group) is done here once, not as a second unit-forecast formula.
- `skus_critical_7d` / `suggested_reorder_count`: `founder_briefing.data.get_projected_stockouts_api(lookahead_days=7)`.
- No dedicated drill-down endpoint.

---

### Production (Live)

**What it is:** How many orders are currently stuck inside the factory pipeline longer than expected — scoped specifically to pre-shipping stages, so an order that has already shipped never shows up here as "stuck in production."

**How it's built (plain):**
```
Stuck order = an order that has breached a factory-stage deadline and hasn't been
              resolved for longer than the configured threshold (default 5 days)
```

**Color rule:** 🟢 On track — fewer than 3 stuck orders · 🟡 At risk — 3–7 · 🔴 Off track — 8+.

**Technical:**
- Function: `_production_tile(settings)`, backed by `_get_factory_stuck_orders(threshold_days)`.
- **Intentional divergence from the Daily Founder Briefing's `get_stuck_orders()`:** both share the same base definition (Active/Escalated SLA Task Link open ≥ threshold days), but this tile **excludes** SLA Rules whose `close_condition_value` is a post-shipment stage (`"Shipped"`/`"Completed"` — rules R3/R4/Stale Tracking Alert) — a shipped-but-not-closed order is a real signal but not a "stuck in production" one. Confirmed live: `LYF-SH-2026-0653` showed on this tile while `status=Shipped` before this scoping existed. The Daily Briefing correctly stays unscoped (full order-lifecycle view) on purpose — **if these two ever disagree, that's expected, not a bug to "fix" into agreement.**
- Drill-down: `get_production_drilldown()` — same query as the tile, full detail.

---

## VI. Who's connected right now? (Super Admin only)

**What it is:** How many distinct people currently hold a live Claude/MCP login — a security-visibility card, not a business metric. Only visible to Super Admin (or Administrator).

**Technical:**
- Function: `_mcp_access_tile()`, gated by `_check_super_admin_permission()` — **not** part of the main combined endpoint (see §10 for why).
- Mirrors exactly what `frappe.oauth.OAuthWebRequestValidator.validate_bearer_token()` checks at request time: `client IN MCP_OAUTH_CLIENT_IDS AND status != 'Revoked' AND expiration_time > now()`. The count is never higher than what would actually still authenticate right now.
- Counts distinct **users**, not tokens — one person can hold more than one live token (e.g. a stale browser tab plus a fresh one).
- MCP Connector Audit Issue 19 follow-up (2026-08-13).
- Drill-down: `get_mcp_access_drilldown()` — one row per user, most recent login.

---

## How the filters at the top work

Four controls reshape most of what's below them:

- **Period** — Today, Month to date, Last full month, Last 90 days, or a custom range. (Sales/Margin/Cash-In keep their own fixed-window behavior on the plain-MTD/no-other-filter path, per the nuances explained above.)
- **Channel** — restricts everything to one sales channel instead of all combined.
- **Order Type** — Standard catalog orders vs. custom quote-built orders, or both.
- **Category** — one product category at a time, or all. On Trending Categories, also scopes the monthly history table and momentum buckets to just that category.

All four round-trip as one JSON object on every request — see §11 for the exact shape.

## If a number ever looks wrong

**Click the tile.** Every tile with a "click for detail" note opens the exact underlying record list. **If a tile shows "Data unavailable"** — that one section had a hiccup; every other tile keeps working independently, and the failure is logged (`frappe.log_error`) for the team to check. If the number involves an **"Other"** category, see §16 item 2 below before assuming it's a bug.

---

# Part 2 — How the page is built

## 9. Files in this folder

| File | Role |
|---|---|
| `founder_dashboard.py` | All server-side data — tile/section functions, the combined summary endpoint, 18 independent per-tile endpoints (§14a), drill-down endpoints. |
| `founder_dashboard.js` | Entire client-side page — filter rail, tile-level refresh/filter infra, rendering, charts, drill-down dialogs. Single self-contained IIFE registered on `frappe.pages["founder-dashboard"].on_page_load`. |
| `founder_dashboard.json` | Page doctype record — title, roles (`System Manager`, `Super Admin` — `Founder` was removed 2026-09-10, see §10). |
| `__init__.py` | Empty, standard Frappe page-module marker. |
| `FOUNDER_DASHBOARD_REFERENCE.md` | This file — the single technical reference for the page. |
| `FOUNDER_DASHBOARD_USER_GUIDE.md` | Plain-English overview for end users — share this one with founders/CS/ops, not this file. |

**Related doctypes:**
- `Founder Dashboard Settings` (singleton, `lh/lyfe_hardware/doctype/founder_dashboard_settings/`) — every tunable threshold on this page lives there (§12), never hardcoded in Python.
- `Founder Dashboard Tile Config` (child table on the above, `tile_key`/`label`/`visible`/`sort_order`/`section`/`col_span`) — the single source of truth for the whole page's layout (§15b, added 2026-08-27, Founder Dashboard — True Modularity). `visible` (show/hide), `sort_order` (position on the page), `section` (which eyebrow heading a tile groups under), and `col_span` (how many of 12 grid columns it occupies) are all live and consumed by the JS's `renderLayout()` — editing any of these fields in Settings changes the live page with no code deploy. Adding a 17th tile means one row here (via a patch) + one entry in each of the server-side `TILE_REGISTRY` and client-side `TILE_ENDPOINTS`/`TILE_RENDERERS` — no template or hand-typed dict to edit by hand anymore.

## 10. Access control

⚠️ **Corrected — the `Founder` role was removed and `_check_permission()` is a two-part gate, not a single doctype-level check.**

- **Page-level:** the `Page` record's `roles` list gates whether the page appears in the Desk at all — `System Manager`, `Super Admin` only. `Founder` was **removed** from this list on 2026-09-10 as part of the same security fix described below.
- **Endpoint-level (Security review 2026-09-10):** `_check_permission()` now does **both**:

  ```python
  frappe.has_permission("Lyfe Order", "read", throw=True)
  require_dashboard_role("founder_dashboard")
  ```

  Previously it was `frappe.has_permission("Lyfe Order", "read", throw=True)` alone, which let *any* role with base Lyfe Order read (e.g. Customer Service, Sales User) call this page's whitelisted methods directly — confirmed live: a Customer Service-role user could call `get_tile_sales()`/`get_margin_drilldown()` etc. and get full data, denied only at the MCP layer. Fixed by delegating to the same `require_dashboard_role()` the MCP tools already use, backed by the single `DASHBOARD_ROLES` source of truth in `mcp_audit.py` (`{"System Manager", "Super Admin"}` for this dashboard) — the Page's own `.json`, this backend check, and the MCP tools now all read from one place and cannot silently drift apart. Same fix pattern applied to `pnl_dashboard.py` and `quotation_analysis_dashboard.py`.
  The MCP Access tile alone calls the stricter `_check_super_admin_permission()` (Administrator or `Super Admin` role only, narrower than the page's own role set).
- **Why MCP Access is a separate endpoint:** `get_founder_dashboard_summary_v2()` wraps every section in a generic try/except that would otherwise swallow a `PermissionError` from `_check_super_admin_permission()` and silently degrade the card to "data unavailable" instead of omitting it for an unauthorized viewer. So it's fetched by `get_mcp_access_summary()` on its own, and the JS checks `frappe.user.has_role("Super Admin") || frappe.session.user === "Administrator"` **before** even calling it — this also avoids Frappe's automatic "Not permitted" popup a 403 would otherwise trigger for every System Manager viewer, every page load.

## 11. Request flow (client → server)

**First paint (page load):**
1. `loadFilterOptions()` calls `get_filter_options()` to populate the Channel/Category `<select>`s in the global filter rail.
2. `loadTileConfig()` calls `get_tile_config()` to populate `TILE_VISIBILITY` — this must resolve **before** the first render, so it's sequenced ahead of `load()` (`loadTileConfig(() => load())`), not fired in parallel like `loadFilterOptions()` is.
3. `load()` calls `get_founder_dashboard_summary_v2(global_filters=JSON.stringify(state))` — one combined request that still computes every tile server-side (visibility isn't applied server-side, see §15a), used only for the fast first paint.
4. `render(data)` builds the whole page from that one response via `renderTileSlot()` (skips a tile entirely if `isTileVisible()` is false), binds drill-down/refresh/filter handlers, renders the Trending Categories charts, then separately calls `loadMcpAccess()`.

**After first paint — every tile is independent (Tile-Level Refresh & Custom Filters, added 2026-08-27):**
5. Each tile's own ↻ (refresh) and ▼ (filter, where applicable) icons live in its header. Clicking ↻ calls `refreshTile(tileKey)`, which resolves that tile's filters via `resolveTileFilters(tileKey)` and fires **that tile's own whitelisted endpoint** (see §14a) — a single, independent `frappe.call()`. Only that tile's DOM node (`#fd-tile-<key>`) shows a loading skeleton and updates; every other tile's state, DOM, and in-flight request (if any) is untouched. A failure here shows only inside that one tile (`d.error`/`errorTile` path) and never touches the rest of the page.
6. Clicking ▼ opens `openTileFilterPopover(tileKey, fieldsConfig)` — one shared dialog component, not one per tile — showing only the fields meaningful for that specific tile (see §12a for the full per-tile filter table). Applying calls `refreshTile(tileKey)` with the new filter; "Reset to global" clears that tile's override and re-fetches using the global rail's current values instead.
7. The global filter rail (Period buttons, Channel/Order Type/Category selects) still exists and still mutates `state`, then calls `load()` — the full combined re-fetch, same as before. This is the **fallback layer**: any tile with no tile-specific filter override picks up the new global values on its next fetch; a tile WITH an active override keeps using its own filter values regardless of what the global rail says (§12a's precedence rule) — the global rail change never silently overwrites a tile-specific filter.
8. The page-level **Refresh** toolbar button now means "refresh every tile independently, honoring each tile's own currently-resolved filters" — it calls `load()` for the fast path, but the intent for the tile-level system is N independent `refreshTile()` calls, not one combined re-fetch, once a founder has set per-tile filters.

**Filter rail state shape (global, unchanged):**
```js
{
  period:     { preset: "mtd" },   // "today" | "mtd" | "last_full_month" | "last_90d" | { preset: "custom", from_date, to_date }
  channel:    "All",               // or a real order_source value
  order_type: "All",               // "All" | "Standard" | "Custom"
  category:   "All",               // "All" | a real item_group value
}
```

This object round-trips as `global_filters` on the combined endpoint and every drill-down endpoint (parsed via `_parse_global_filters()`), so a drill-down always reconciles with whatever the tile is currently showing on the combined-endpoint path. The 18 independent per-tile endpoints do **not** take a `global_filters` blob — they take `period`/`channel`/`order_type`/`category` (and any tile-specific field) as **discrete kwargs**, parsed via the parallel `_parse_tile_filters()` helper — this is what `resolveTileFilters()` sends, per-field-resolved client-side before the call is made (see §15a).

**Fee/payment row exclusion (shared with `pnl_dashboard.py`):** `ShipStation Order Item` rows are sometimes not real products — a plain line item named `"Custom Fee"`, `"Customization Fee: $100"`, `"Additional Payment for LH#6214"`, a return-label surcharge, etc. ShipStation's own `adjustment` field does **not** flag these (confirmed live: every sampled fee row had `adjustment=0`, same as a normal product row — it only flags discount lines). Any line-item query that must never show a fee row as a "product" (Best Sellers, Slow Movers, the Category filter list, Look Ahead's avg-price-per-category) filters with `_exclude_fee_rows_sql()` / `_FEE_ROW_ITEM_NAME_PATTERN`, both imported from `pnl_dashboard.py` — never duplicate this pattern locally. It does **not** apply to Money at Risk / RAG tiles that sum `Lyfe Order.total_amount` directly (no line items read), or Material Usage/stockout queries (keyed off `item_code`/`erp_item`, naturally clean).

## 12. Founder Dashboard Settings (tunable thresholds)

Singleton doctype, System-Manager-only. **Never hardcode a threshold this doctype already owns** — always read via `frappe.get_cached_doc("Founder Dashboard Settings")` with an `or <fallback>`.

Also holds the `tile_config` child table (`Founder Dashboard Tile Config` — `tile_key`/`label`/`visible`/`sort_order`), a separate concern from the thresholds below: it controls which tiles render at all and (not yet wired, see §16 item 5) their order, not a business threshold.

| Section | Field | Default | Used by |
|---|---|---|---|
| Sales Tile | `sales_target_monthly` | 198000 | `_sales_tile` |
| | `sales_at_risk_pct` | 80 | `_sales_tile` |
| Margin Tile | `margin_target_pct` | 20 | `_margin_tile` |
| | `margin_at_risk_pct` | 15 | `_margin_tile` |
| Production Tile | `production_stuck_threshold_days` | 5 | `_production_tile`, `_get_factory_stuck_orders`, `_money_at_risk_section` |
| | `production_at_risk_count` | 3 | `_production_tile` |
| | `production_off_track_count` | 8 | `_production_tile` |
| CS Tile *(v1 only — see §16)* | `cs_sla_breach_at_risk_pct` | 15 | `_cs_tile` |
| | `cs_sla_breach_off_track_pct` | 30 | `_cs_tile` |
| Cash-In Tile | `cash_in_target_monthly` | 34700 | `_cash_in_tile` |
| | `cash_in_at_risk_pct` | 80 | `_cash_in_tile` |
| Trending Categories Tile | `trending_categories_months` | 6 | monthly history depth |
| | `trending_categories_min_prior_revenue` | 1000 | Rising/Declining ranking floor |
| | `trending_categories_emerging_prior_max` | 500 | Emerging bucket's "was near-zero" ceiling |
| | `trending_categories_emerging_current_min` | 2000 | Emerging bucket's "now real revenue" floor |

## 12a. Per-tile filters (Tile-Level Refresh & Custom Filters)

Every tile is independently refreshable (§14a). Only tiles with a genuinely meaningful filter dimension get a filter icon — added deliberately per tile, never blindly copied across all of them. **Precedence, always: tile-specific filter → global filter rail → backend default.** A tile's own filter is never silently overwritten by a global rail change.

| Tile | Filter icon? | Fields | Why (or why not) |
|---|---|---|---|
| Sales | Yes | Channel, Order Type, Category | Already fully filter-rail-aware; own popover just makes an override explicit and sticky. |
| Margin | Yes | Channel, Order Type, Category | Same as Sales. |
| Cash In | Yes | Payment Type (Full/Partial), Payment Mode (Payment Link/Manual) | Channel/Order Type/Category explicitly **not** offered — `Quotation Payment Entry` has no join back to Lyfe Order's channel/type/category fields (§11's "External dependencies" caveat). Payment Type/Mode are the only two real dimensions this child table has. |
| Custom Revenue | Yes | Channel, Category | **No** Order Type filter — Order Type *is* the tile's own output split (Custom vs Standard); filtering by it would just hide one side of the comparison. |
| Best Sellers | Yes | Channel, Order Type, Category | **No** Customer filter — this is an item-ranking tile, no customer dimension in its query at all. |
| Slow Movers | Yes | Channel, Order Type, Category | Same as Best Sellers. |
| Category Performance (Trending Categories) | Yes | Channel, Order Type, Category | Same as Sales. |
| Money at Risk | No | — | Always-live risk snapshot by design — a period filter would contradict the tile's purpose (docstring explicit). |
| Stockout Risk | Yes | Category (`item_group`) | **No** Warehouse split — `get_projected_stockouts()` sums `current_qty` **across** warehouses in its own aggregation; splitting by warehouse needs restructuring that math, not a WHERE clause. Explicitly out of scope this rollout. |
| Material Usage | Yes | Trailing window (7/14/30/60 days) | Its only filterable dimension (`period_days`) isn't the shared Period-preset shape — a dedicated day-count select, reusing the same generic popover. |
| Board Health | Yes | Department, Priority | **No** Project filter — the tile's whole point is comparing every board side by side; a Project filter would collapse it back to one board, defeating the tile's purpose (documented in the backend). |
| PM Task Risk | Yes | Project, Department, Priority | Pure passthrough into `pm_operations_dashboard.get_sla_risk_data()`, which already accepts all 3. |
| Customer Concentration | Yes | Channel, Customer | New Customer filter added to `pnl_dashboard._build_where()` (shared function, one column addition) — narrows the whole card to one customer; `get_top_customer_item_group_breakdown()` then naturally shows *that* customer's own breakdown (it recomputes "top customer" against the already-narrowed rows — no special-case code). **No** Order Type/Category — not wired upstream in the customer-facing pnl_dashboard functions. |
| Where the Revenue Is (Geography) | Yes | Channel, Order Type, Category | Same as Sales — straight `_build_where_v2()` filtering, no geography-specific caveat. |
| Look Ahead | No | — | Forward-looking forecast, filter-rail-independent by nature — not an order-filtered rollup. |
| Production | No | — | The tile's own funnel IS already grouped by `workflow_state`; filtering to one state would collapse it to a single row (same reasoning as Board Health's skipped Project filter). |
| MCP Access | Yes | OAuth Client | The one real dimension this security/access data has — Channel/Category etc. are nonsensical for a live access list. |
| Month-on-Month | No (own control) | Trailing 6M/12M or a specific Year | Standard Period/Channel/Order Type/Category are meaningless for a trailing-months trend — it has its own months/year control instead (`TILE_SUPPORTED_FIELDS.mom = ["months", "year"]`), same precedent as Material Usage's `period_days` selector. |

**Explicitly rejected app-wide** (confirmed absent from this app's schema, not an oversight): **Branch** (no Branch doctype/field exists anywhere in `lh`), **Sales Person** (no such field/concept — only Frappe's generic `owner`/`modified_by` audit fields exist), **Quotation Status** (the only Quotation-touching tile, Cash In, never reads `Quotation.status` — only the unrelated Payment Entry child table). **Comparison Period** (a second date-range query + delta UI) is a distinct, larger feature deliberately deferred, not dropped — a candidate follow-up, not part of this rollout.

## 13. External dependencies (other modules this page calls into)

Per **BR-3** ("no second definition of any metric"), this page delegates almost everything rather than recomputing:

| Module | What's used |
|---|---|
| `lh.lyfe_hardware.page.pnl_dashboard.pnl_dashboard` | `_build_where`, `_fetch_orders`, `_aggregate_rows`, `_exclude_fee_rows_sql`, `_FEE_ROW_ITEM_NAME_PATTERN`, `get_summary`, `get_category_breakdown`, `get_category_monthly_trend`, `get_item_group_revenue_summary`, `get_order_type_breakdown`, `get_top_customer`, `get_top5_concentration`, `get_repeat_customers`, `get_top_customer_item_group_breakdown`, `get_revenue_concentration_curve`, `get_customer_item_group_matrix`, `get_orders_detail`, `safe_float` |
| `lh.lyfe_hardware.page.order_analysis.order_analysis` | `get_dashboard_data`, `get_card_orders` |
| `lh.lh_project.page.pm_operations_dashboard.pm_operations_dashboard` | `get_sla_risk_data`, `get_project_health_scores` |
| `lh.lyfe_hardware.page.lyfe_orders_status_overview.lyfe_orders_status_overview` | `get_orders`, `get_active_reshipments` *(v1 only)* |
| `lh.lyfe_hardware.founder_briefing.data` | `get_projected_stockouts_api` |
| `lh.lyfe_hardware.forecasting.exponential_smoothing` | `get_item_group_demand_forecast` |
| `lh.lyfe_hardware.integrations.mcp_scope_guard` | `MCP_OAUTH_CLIENT_IDS` |

If a number ever looks wrong, the fix almost always belongs in one of these upstream modules — not as a new, second calculation inside `founder_dashboard.py`.

## 14. Whitelisted endpoints — full list

| Endpoint | Gate | Purpose |
|---|---|---|
| `get_filter_options()` | `_check_permission` | Distinct Channel/Category values for the filter rail. |
| `get_tile_config()` | `_check_permission` | Full layout config (`visible`/`sort_order`/`section`/`col_span`) from `Founder Dashboard Tile Config` (§9), explicitly sorted by `sort_order` in Python before returning — see §15b for why that sort can't be left to the doctype's own `sort_field`. |
| `get_founder_dashboard_summary_v2(global_filters)` | `_check_permission` | Combined call — first-paint snapshot only as of the tile-level rollout (§11), and (since 2026-08-27) skips computing any tile `Founder Dashboard Tile Config` marks not-visible; every tile's own Refresh uses its independent endpoint instead (§14a). |
| `get_mcp_access_summary(oauth_client=None)` | `_check_super_admin_permission` | Super-Admin-only MCP Access card; now also its own tile-level endpoint (§14a). |
| `get_sales_drilldown(global_filters)` | `_check_permission` | Orders behind the Sales tile. |
| `get_margin_drilldown(global_filters)` | `_check_permission` | Orders behind the Margin tile. |
| `get_production_drilldown()` | `_check_permission` | Stuck orders behind the Production tile. |
| `get_cs_drilldown(period_days=30)` | `_check_permission` | *(v1 only)* Returns/RTOs behind the CS tile. |
| `get_cash_in_drilldown(global_filters)` | `_check_permission` | Payment entries behind Cash In. |
| `get_material_usage_drilldown(period_days=30)` | `_check_permission` | Full most/least-issued item list. |
| `get_stockout_risk_drilldown()` | `_check_permission` | Full projected-stockout list. |
| `get_mcp_access_drilldown()` | `_check_super_admin_permission` | Individual currently-authenticated users. |
| `get_founder_dashboard_summary()` | `_check_permission` | v1 combined endpoint — dead from the current JS, kept for compatibility. |

## 14a. Independent per-tile endpoints (Tile-Level Refresh & Custom Filters)

Every tile has its own thin whitelisted wrapper around the same section function `get_founder_dashboard_summary_v2()` calls (BR-3 — wrap, never fork). Each is called independently by `refreshTile(tileKey)` (§11) — not through the combined endpoint — once a tile's own ↻/▼ icon is used. Failures here are logged **and re-raised** (`_log_and_reraise`), unlike the combined endpoint's `_safe()` wrapper which degrades a failing section to `{"status": "Unknown", "error": name}` — an independent endpoint's whole point is that its own failure surfaces as that one tile's retry state via the JS's `frappe.call` `error:` callback, not a silently-defaulted payload.

**`TILE_REGISTRY` (server-side, added 2026-08-27):** `get_founder_dashboard_summary_v2()` no longer hand-types a 14-entry dict literal — it loops over a module-level `TILE_REGISTRY` dict (`{tile_key: lambda settings, period_filters, channel, order_type, category: <section function call>}`), filtered to only the tiles `Founder Dashboard Tile Config` marks visible (`_visible_tile_keys()` — fail-open default to visible if a tile has no config row, matching `isTileVisible()`'s own client-side default). Every lambda is a one-line adapter to a uniform 5-arg signature, calling the exact same section function its `get_tile_*` sibling above already wraps — no calculation logic lives in the registry itself. `mcp_access` is deliberately excluded from `TILE_REGISTRY` (stays on its own Super-Admin-gated path, same PermissionError-swallowing reason `_mcp_access_tile`'s docstring already documents).

| Tile key | Endpoint | New/changed params this rollout |
|---|---|---|
| `sales` | `get_tile_sales(period, channel, order_type, category)` | — (endpoint split only) |
| `margin` | `get_tile_margin(period, channel, order_type, category)` | — |
| `cash_in` | `get_tile_cash_in(period, payment_type, payment_mode)` | `payment_type`, `payment_mode` (new) |
| `custom_revenue` | `get_tile_custom_revenue(period, channel, category)` | — |
| `best_sellers` | `get_tile_best_sellers(period, channel, order_type, category)` | — |
| `slow_movers` | `get_tile_slow_movers(period, channel, order_type, category)` | — |
| `trending_categories` | `get_tile_trending_categories(period, channel, order_type, category)` | — |
| `money_at_risk` | `get_tile_money_at_risk()` | — |
| `stockout_risk` | `get_tile_stockout_risk(item_group)` | `item_group` (new) |
| `material_usage` | `get_tile_material_usage(period_days=30)` | — (was already this shape, now independently callable) |
| `board_health` | `get_tile_board_health(department, priority)` | `department`, `priority` (new) |
| `pm_task_risk` | `get_tile_pm_task_risk(project, department, priority)` | `project`, `department`, `priority` (new) |
| `customer_concentration` | `get_tile_customer_concentration(period, channel, customer)` | `customer` (new) |
| `look_ahead` | `get_tile_look_ahead()` | — |
| `production` | `get_tile_production()` | — |
| `mcp_access` | `get_mcp_access_summary(oauth_client)` | `oauth_client` (new) |
| `mom` | `get_tile_mom(months, year)` | `months`, `year` (new tile, 2026-09-03) |
| `geography` | `get_tile_geography(period, channel, order_type, category)` | new tile — ⚠️ previously missing from this doc entirely, see "Where the Revenue Is" in Section II |

All 18 gated by `_check_permission()` except `mcp_access`'s `_check_super_admin_permission()` (unchanged from before this rollout).

**Discrete kwargs, not a JSON blob:** these endpoints receive `period`/`channel`/`order_type`/`category` (and any tile-specific field) as separate parameters — parsed via `_parse_tile_filters()` (mirrors `_parse_global_filters()`'s defaults, different input shape) — because `resolveTileFilters()` on the JS side already resolves tile-specific-vs-global precedence per field before the call is made, and only sends fields declared in `TILE_SUPPORTED_FIELDS[tileKey]` (§15a) — never a field a given endpoint doesn't accept.

**New backend params added this rollout, and what they extend (BR-3 — every one reuses or extends an existing shared function, never forks a parallel calculation):**
- `pnl_dashboard._build_where()` gained a `customer` condition (one column on the same driving table every caller already queries) — `get_top_customer`/`get_top5_concentration`/`get_repeat_customers`/`get_top_customer_item_group_breakdown` all pick it up automatically.
- `founder_briefing.data.get_projected_stockouts()`/`get_projected_stockouts_api()` gained `item_group=None` — filters the already-computed risk rows post-calculation (the risk math itself — `current_qty`/`avg_daily_outflow`/`days_of_stock` — is untouched); every existing caller (scheduler.py, the Daily Founder Briefing, MCP tool) is byte-for-byte unchanged when the param is omitted.
- `pm_operations_dashboard.get_sla_risk_data()`/`get_project_health_scores()` already accepted `project`/`department`/`priority` before this rollout — `_sla_risk_section()` and `_board_health_section()` now pass them through for the first time.
- `_cash_in_tile()` gained `payment_type`/`payment_mode` — new WHERE conditions on `Quotation Payment Entry`'s own two Select fields.

**Explicitly out of scope, investigated and rejected as non-trivial:** a Warehouse split on Stockout Risk (§12a); Comparison Period on any tile (§12a); any dimension not confirmed to exist in this app's schema (Branch, Sales Person, Quotation Status — §12a).

## 15. Client-side architecture (`founder_dashboard.js`)

- **Structure:** one closure via `frappe.pages["founder-dashboard"].on_page_load`. No build step, no separate modules — standard Frappe Desk Page convention.
- **Theming:** `THEME = { light, dark }` token object injected once as CSS custom properties on `#fd-root`, covering explicit `data-theme` overrides and `prefers-color-scheme`. Components reference `var(--fd-*)` only, never hardcoded colors.
- **Generic rendering helpers:** `tile()` (RAG KPI tiles), `panel()` (every other card), `skeletonGrid()`, `statusBadge()`, `orderLink()` (real `<a href>` so ctrl/cmd-click opens a new tab, plain clicks intercepted for Frappe's SPA route), `deltaHtml()`/`_fdTrendBadge()` (trend arrows), `categoryLabel()` ("Other" explainer, §16 item 2), `marginClass()` (color-coding), `bindDrilldowns()`/`bindPageLinks()` (re-bound after every render). Both `tile()` and `panel()` now take an optional `tileKey` param (§15a) — 7th positional arg on `panel()`, backward compatible, undefined for any call site that doesn't pass it.
- **Drill-down dialogs:** one generic `openDrilldown(key)` driven by a `DRILLDOWN_CONFIG` map with exactly these keys: `sales · margin · production · cash_in · material_usage · stockout_risk · mcp_access`. Each entry declares `title`, `method`, and `columns: [{label, render(row), align}]`. Customer Concentration has its **own** separate dialog machinery (`openTopCustomerItemGroupDialog`, `openCustomerItemGroupMatrixDialog`, `bindCustomerConcentrationDrilldown`) since its data shape doesn't fit a flat row list.
- **Charts:** `frappe.Chart` instances tracked in a `charts = {}` registry, `destroyChart(key)` called before every re-render (same leak-prevention convention as `pnl_dashboard.js`/`comparison_dashboard.js`). Chart functions: `renderTrendingCategoriesCharts()`, `renderParetoChart()`, `renderCustomerItemGroupMatrixChart()`.
- **Section renderers:** one `render<Name>(data)` per tile, each defensive against `d.error` (the Python `_safe()`/independent-endpoint failure marker) — a failed section shows its own "Data unavailable" card without breaking the rest of the page. Every renderer is also registered in `TILE_RENDERERS` (§15a) for `renderOneTile()`'s independent re-renders — the exact same function `render()`'s first paint uses, never a forked copy.
- **Page layout order** (as assembled in `render()` via `renderTileSlot()`, §15a): Are we making money? (`fd-grid`, 4 tiles) → What's selling & dying? (`fd-g2` + full-width `fd-g1`) → Where am I bleeding? (`fd-g3`) → Is the team keeping up? (`fd-g3`) → What's coming? (`fd-g2`) → Who's connected right now? (hidden until `loadMcpAccess()` succeeds for an authorized viewer AND `isTileVisible("mcp_access")`). A tile turned off in Tile Config (§9) renders as an empty string in its grid slot — the CSS grid reflows around the gap, no special layout handling needed.

## 15a. Tile-level refresh & filter infrastructure (added 2026-08-27)

Core state and functions, all module-scoped inside the same `on_page_load` closure:

| Name | Role |
|---|---|
| `TILE_KEYS` | The full list of tile keys — used to initialize `tileState` and for the first-paint data-bookkeeping loop. Its own array order is NOT what determines render order since §15b's layout engine landed — actual position/grouping/width comes from `TILE_LAYOUT` instead (§15b). ⚠️ `geography` was missing from this array from when the tile was added until it was found and fixed — since `tileState[tileKey]` is only ever created by iterating `TILE_KEYS`, that gap meant `tileState["geography"]` was `undefined`, and clicking that tile's own Refresh button threw a `TypeError` (`t.loading = true` on `undefined`). Fixed by adding `"geography"` to the array; if you add a new tile, `TILE_KEYS` is the one place this doc calls out as easy to forget even though `TILE_LAYOUT`/`TILE_ENDPOINTS`/`TILE_RENDERERS` were all updated correctly. |
| `tileState[tileKey]` | `{filters, loading, error, data, last_updated}` per tile — plain in-memory object, no shared mutable state between tiles, resets on page reload (no persistence layer). |
| `TILE_ENDPOINTS[tileKey]` | Maps a tile key to its independent whitelisted method name (§14a). A tile with no entry falls back to the full combined `load()` in `refreshTile()` — every tile has an entry today, kept as a safety net for any future tile added without its own endpoint yet. |
| `TILE_SUPPORTED_FIELDS[tileKey]` | Allow-list of filter field names that tile's endpoint actually accepts. `resolveTileFilters()` never sends a field not listed here — prevents a server-side "unexpected keyword argument" `TypeError` (e.g. `mcp_access` accepts none of the standard 4, so a non-default global Period/Channel/etc. must never be forwarded to it). A tile not listed here defaults to the full standard 4. |
| `TILE_FILTER_FIELDS[tileKey]` | Returns the field config array (`{key, label, type, options?}`) for that tile's filter popover — only present for tiles with a genuine filter icon (§12a). |
| `TILE_VISIBILITY[tileKey]` / `isTileVisible()` | Populated by `loadTileConfig()` from `get_tile_config()` — defaults every tile to visible if config hasn't loaded or the tile isn't listed, so this can never silently hide a tile the founder hasn't explicitly turned off. |
| `resolveTileFilters(tileKey)` | The precedence resolver — tile-specific filter → global rail (only if set to a non-default value) → omitted (backend default). Per-field, not all-or-nothing. Reads `state`/`tileState` live at call time, mutates neither. |
| `refreshTile(tileKey)` | Fires that tile's own `frappe.call()` using `resolveTileFilters(tileKey)`, shows a loading skeleton only in that tile's DOM node, updates only that tile's state/DOM on success or failure. |
| `renderOneTile(tileKey)` | Rebuilds one tile's `#fd-tile-<key>` DOM node from current `tileState`, using the same `TILE_RENDERERS[tileKey]` function `renderLayout()`'s first paint uses — no forked rendering logic. |
| `renderTileSlot(tileKey, renderFn, dataArg, extraClass?)` | Used inside `renderLayout()` (§15b) — returns `""` (nothing) if `isTileVisible(tileKey)` is false, otherwise a `<span id="fd-tile-...">` wrapper, optionally carrying a CSS class (used for the `fd-span-N` grid-width class, §15b). |
| `tileRefreshButtonHtml(tileKey)` / `tileFilterButtonHtml(tileKey)` | Small reusable header-button builders — only render a button if that tile is in `TILE_ENDPOINTS`/`TILE_FILTER_FIELDS` respectively. The filter button shows a filled/active state (`.fd-tile-filter-active`) when `tileHasCustomFilters(tileKey)` is true. |
| `bindTileRefresh(scope)` / `bindTileFilterOpen(scope)` | Generic click binders for `[data-refresh-tile]`/`[data-open-tile-filter]`, re-bound after every render (same convention as `bindDrilldowns()`). |
| `openTileFilterPopover(tileKey, fieldsConfig)` | **One** shared `frappe.ui.Dialog`-based popover, reused by every filterable tile — not 16 hand-rolled dialogs. Primary action applies the filter and calls `refreshTile()`; secondary action ("Reset to global") clears it and re-fetches with global-rail values. |
| `loadTileConfig(always)` | Fetches `get_tile_config()` once at bootstrap, before the first `load()` (see §11). |

**Every tile's DOM slot has a stable `id="fd-tile-<key>"`** (a `<span>`, `display: block` in CSS since §15b — a real grid item carrying its own `fd-span-N` width class) — this is what makes `renderOneTile()`/`refreshTile()` able to target and replace exactly one tile's inner content without touching `#fd-body`, any sibling, or the span's own class/position (`$slot.html(...)` only replaces inner content, confirmed — the wrapper span and its layout are untouched by a refresh).

**Known accepted tradeoff:** splitting tiles into independent endpoints means two request-scoped duplicate computations that existed inside the single combined endpoint can no longer be de-duplicated in-process: `order_analysis.get_dashboard_data()` (called by both `money_at_risk` and `pm_task_risk`) and `_get_factory_stuck_orders()` (called by both `production` and `money_at_risk`) each now genuinely run twice across two separate HTTP requests instead of twice within one Python process. No `frappe.cache()` was introduced to recover this — zero caching precedent existed anywhere in this file before this rollout, and there's no profiling data showing this is an actual measured cost. Revisit only if it's shown to matter.

## 15b. Config-driven layout (Founder Dashboard — True Modularity, added 2026-08-27)

Before this, tile grouping/width/order lived in one large hardcoded template literal inside `render()` — 4 separately-named CSS grids (`fd-grid`/`fd-g2`/`fd-g3`/`fd-g1`), one hardcoded `eyebrow()` call per section, 16 literal `renderTileSlot(...)` calls in a fixed order. Reordering tiles from `Founder Dashboard Tile Config`'s `sort_order` field (added in an earlier session) had **zero visible effect**, since nothing read it. That gap is closed:

| Name | Role |
|---|---|
| `TILE_LAYOUT` | The full sorted layout array (`{tile_key, visible, sort_order, section, col_span}` per row) from `get_tile_config()` (§14) — populated by `loadTileConfig()`, which now captures the whole row instead of just `visible`. |
| `FALLBACK_TILE_LAYOUT` | Hardcoded array reproducing today's exact layout, used only if `Founder Dashboard Tile Config` genuinely has no rows (a fresh/unmigrated site before the seed patch runs) — a safety net, not the source of truth once config exists. Must be kept in sync by hand with the seed/backfill patches' own default tables if either changes. |
| `renderLayout(data)` | Replaces the old `render()`. Walks `TILE_LAYOUT` (already sorted by `sort_order` — see below) once; opens a new `.fd-section-block` + `eyebrow()` + `.fd-health-grid` whenever a row's `section` differs from the previous row's, and emits each tile via `renderTileSlot(tileKey, renderer, data[tileKey], "fd-span-" + col_span)`. `mcp_access` is walked for LAYOUT purposes only (section/position/width) — its actual data-fetch and Super-Admin permission gate are entirely separate and untouched (`loadMcpAccess()`'s own role check + server-side `_check_super_admin_permission()`); a Tile Config mistake can move where it sits, never who can see it. |

**Grid system:** the 12-column `fd-health-grid` + `fd-span-3/4/6/12` CSS classes existed in the file well before this change but were never used anywhere — this is the exact primitive a config-driven layout needs (one grid, tiles declare their own width) instead of 4 separate hardcoded named grids. `fd-span-3` was added (didn't exist before) to cover the old `fd-grid`'s quarter-width KPI tiles.

**Backend sort gotcha (found and fixed during this rollout — do not reintroduce):** `Founder Dashboard Tile Config`'s doctype JSON sets `sort_field: "sort_order"` — this only controls how the **Desk UI grid displays/sorts rows for editing**. It does **not** affect a plain Python `doc.tile_config` iteration, which is always in `idx` (insertion) order regardless of that key. Confirmed live: changing a row's `sort_order` value alone, without an explicit sort, left `get_tile_config()`'s returned order completely unchanged. `get_tile_config()` now explicitly does `sorted(settings.tile_config, key=lambda row: row.sort_order)` before returning — this is what actually makes reordering work. Any future code reading an ordered child table by a user-editable sort field must sort explicitly in Python; never assume `sort_field` does it.

**Verified reproduces today's layout exactly, and reorder/hide genuinely work end-to-end** (2026-08-27): `get_tile_config()` returns all 16 rows' `section`/`col_span` matching the pre-refactor hardcoded template exactly; `get_founder_dashboard_summary_v2()`'s output diffed byte-for-byte identical against the pre-refactor function for the default all-visible case; moving `board_health`'s `sort_order` ahead of `sales` changed `get_tile_config()`'s returned order immediately; hiding `production` removed it from both `get_tile_config()` and the combined response, with zero effect on any other tile's computed value.

## 16. Known open items / follow-ups

0. **Margin tile shows MTD, not last-full-month** — the tile's own dead-code fallback claims to default to last full calendar month specifically to avoid an early-month-inflation risk (missing shipping costs), but the live page always sends an explicit `{preset: "mtd"}`, so that fallback never fires. Confirmed via full-audit 2026-08-27. The new `mom_pct` line (added the same day) compares MTD against the last full month specifically to give a stable reference point despite this — but the tile's own headline number is still MTD, carrying the inflation risk on its own. Not yet fixed; changing the actual default period is a separate decision (would change what number founders see day-to-day) from adding the comparison line.
1. **Order Pipeline sub-dashboard** — the v1 "stage bars" tile was deliberately demoted off the main screen per the Annex A memo; its backend (`_order_pipeline_section()`) still exists but nothing in the current JS renders it, and no replacement sub-dashboard has been built.
2. **The "Other" category bucket** — every category-grouping query uses `COALESCE(NULLIF(TRIM(item_group), ''), 'Other')`. `"Other"` is not a real Item Group; it covers (a) fee/sample/free-text rows with no real product (correctly unresolvable) and (b) real SKUs whose `ShipStation Order Item.item_group` is blank or **stale** (disagreeing with the current `Item` master, because `shipstation_orders.py` explicitly preserves a row's existing `item_group` on every re-sync rather than re-deriving it). Both populations were bulk-corrected via `lh/patches/backfill_and_fix_order_item_category_from_item_master.py` (85 blank rows filled, 2,039 stale rows corrected against the `Item` master as source of truth) — a narrower predecessor patch, `backfill_order_item_category_from_erp_item.py`, is superseded in scope but left in place. **The live sync path itself was not changed** — new rows can still arrive blank/stale, and this can reaccumulate. The JS's `categoryLabel()` helper appends `" (uncategorized — no Item Group set)"` wherever "Other" is shown as plain text (never in chart label/data arrays, which would corrupt the chart's own data).
3. **CS tile (`_cs_tile`)** — only reachable via the dead v1 endpoint; the current 5-question layout has no live replacement tile for "% of returns/RTOs with a breached SLA."
4. **v1 endpoint (`get_founder_dashboard_summary`)** and its dedicated-only sections (`_order_pipeline_section`, `_project_health_section`, the v1 `_customer_concentration_section`) are dead code from the UI's perspective — not deleted, per this file's established non-destructive-iteration convention, but they don't affect what a founder actually sees today.
5. ~~Tile Config reorder (`sort_order`) is stored but not yet consumed~~ — **resolved 2026-08-27** (Founder Dashboard — True Modularity). `sort_order`, `section`, and `col_span` are all live and consumed by `renderLayout()` (§15b) — reordering, regrouping, or resizing a tile in Settings now changes the live page with no code deploy.
6. **The two duplicate cross-tile computations noted in §15a** (`order_analysis.get_dashboard_data()` for `money_at_risk`+`pm_task_risk`, `_get_factory_stuck_orders()` for `production`+`money_at_risk`) now run twice across two separate HTTP requests instead of twice in one process, once every tile got its own independent endpoint. Accepted tradeoff, no caching introduced — revisit only if profiling shows this actually matters.
7. **Reshipment cost now folded into margin (2026-09-11)** — `Lyfe Order Reshipment.reshipment_cogs` + `.shipment_cost` (per-order reshipment COGS and shipping spend, previously tracked only on the Status Overview page) are now included in every margin/profit calculation on this page, via a `LEFT JOIN` subquery (`SUM(reshipment_cogs + shipment_cost) GROUP BY lyfe_order`, excluding `workflow_state = 'Cancelled'`) added everywhere `custom_charges` is summed — the two sites are the `order_type`-split query in `_custom_revenue_tile()` and the per-bucket query in `_geography_section()`'s `_by()` closure. Reshipment cost is folded into the `custom_charges` bucket (same treatment as `custom_duty_changes_us_tram`), not a separate line item. Same fix applied identically in `pnl_dashboard.py` (all `custom_charges` SUM sites) — both dashboards previously excluded reshipment cost entirely, silently overstating margin on any order with a reshipment.
8. **Security fix (2026-09-10) — `_check_permission()` now enforces `require_dashboard_role()`** in addition to the base `frappe.has_permission("Lyfe Order", "read")` check, and `Founder` was removed from the Page's own `roles` list — see §10 for full detail. Any doc anywhere still describing this page's roles as including `Founder` is stale; the current set is `System Manager`/`Super Admin` only.
9. **"Where the Revenue Is" (Geography) tile — undocumented until this pass, and shipped with a live bug (both now fixed).** The tile itself (`_geography_section()`, `get_tile_geography()`) had existed in the code with no writeup anywhere in this file — added under Section II, above. Separately, its tile key was missing from the JS's `TILE_KEYS` array, so `tileState["geography"]` was never initialized and that tile's own Refresh button threw a `TypeError` on click — fixed by adding `"geography"` to `TILE_KEYS` (see §15a for detail). Both gaps found during a documentation-accuracy audit, not reported by a user.

---

*Founder Dashboard — Complete Reference · Keep in sync with `founder_dashboard.py`/`founder_dashboard.js` whenever a tile, endpoint, or Settings field is added, renamed, or removed.*
