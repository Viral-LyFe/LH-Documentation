# Stock Forecast Dashboard: Logic Guide (tile by tile, with examples)

**Page:** `/app/stock-forecast` · **Audience:** anyone who wants to understand *why* a number on this page is what it is: factory, purchasing, founders, new team members.
**Version:** 1.1 · **Written:** 2026-11-11 · **Source of truth for the code:** `docs/stock-forecast-dashboard.md` (technical). If this guide and the code ever disagree, the code is right.

All numbers in the examples are made-up round numbers unless marked "checked", which means the figure was run through the real calculation code.

---

## Part 0. Read this first: the 8 ideas behind every tile

| # | Idea | Plain words |
|---|---|---|
| 1 | **Use = what the factory actually issued** | Demand is measured from submitted **Material Issue** stock entries (the ones created by Material Issue for Order). Not sales lines, not transfers, not receipts. |
| 2 | **On hand** | Stock sitting in all warehouses **except** `FAC-Rejected Material Warehouse`. |
| 3 | **Committed** | Units already promised to open orders that have **no Material Issue yet**. They are still on the shelf but spoken for. |
| 4 | **Available = On hand − Committed** | Negative means open orders already need more than we have. |
| 5 | **Days of stock = On hand ÷ daily use** | How long the shelf lasts at the current speed. |
| 6 | **Risk level** | Critical < 7 days · High 7 to <15 · Medium 15 to <30 · Low 30+. (Defaults, set in Founder Briefing Settings.) |
| 7 | **Finishes are pooled** | `WFL2-100-PB`, `-SB`, `-PBU` … are one row "WFL2-100 ×3 finishes", because the blank is finished to order. |
| 8 | **Tube and acrylic are special** | They are forecast per pipe (feet) and per diameter × length class (inches), never as plain units. |

### The one item we follow through the whole guide

**Item `SPLC-150-PB`** (checked, 2026-11-11 style numbers):

- Bin: 45 + 15 = **60 on hand** (another 20 sit in the Rejected warehouse and are ignored).
- Material Issues in the last 30 days: **90 units** → plain average **3.0 per day**.
- Open orders with no Material Issue yet: 10 units directly + 18 units implied by Lyfe BOMs → **28 committed**.
- Target cover: 30 days.

| Step | Calculation | Result |
|---|---|---|
| Days of stock | 60 ÷ 3.0 | **20.0 days → Medium** |
| Stock-out date | today + 20 days | today + 20 |
| Available | 60 − 28 | **32** |
| Available days | 32 ÷ 3.0 | **10.7** |
| Suggested qty | ceil(3.0 × 30 − 32) | **58** |
| With a Minimum Order Qty of 25 | round 58 up to a multiple of 25 | **75** |

---

## Part 1. How to read the page (layout)

1. **Header**: "Last updated", **Refresh** (reloads everything), filter bar (**Category** and **Item Code**).
2. **KPI strip** (four cards, click = jump to that tile).
3. **Three tabs.** Tiles load only when their tab is first opened.

| Tab | Tiles |
|---|---|
| **Act now** | Restock List · Stockout Risk · US Warehouse Replenishment · Material Not In Stock (Total) · Active Orders Material Not In Stock |
| **Demand & forecast** | Demand Trends · Demand Forecast · Base-level Forecast · Category-wise Stock Forecast · Material Usage · Forecast vs Actual |
| **Materials** | Tube Stock · Acrylic Rod Stock · Buildable Orders |

**Who can open it:** System Manager, Super Admin, Factory. The **US Warehouse Replenishment** tile is shown only to System Manager, Super Admin and *US Replenishment Manager*.

**Which tiles follow the page filter bar:** KPI strip, Stockout Risk, Category-wise, Material Usage, Buildable Orders, Restock List, both Not In Stock tiles, Demand Trends, Demand Forecast, Forecast vs Actual. **Not affected:** Tube, Acrylic, Stock Data Quality. A tile's own funnel filter beats the page filter for that tile.

---

## Part 2. Tile-by-tile logic

Each tile has: **Question it answers → What you see → Logic → Example → Watch out for.**

---

### Tile 1: KPI strip (`stock_summary`)

**Question:** "In ten seconds, do I need to act?"

**What you see:** four cards.

| Card | Meaning | Click goes to |
|---|---|---|
| **Orders short now** | Count and **$ value** of active orders that are *Partial* or *Short* in Buildable Orders, plus "N ready" | Buildable Orders |
| **Items to reorder** | Number of rows on the Restock List; sub-line "N critical, N high" | Restock List |
| **Shortest cover** | The single item with the fewest days of stock, any risk level | Stockout Risk |
| **Data issues** | Number of failing data checks (only for users allowed to see data quality) | Stock Data Quality |

**Logic:** it re-uses the rows of the other tiles, so the numbers always reconcile with them. It never fails: if one decision card cannot be worked out it is left out.

**Example:** Buildable Orders shows 4 Partial (value $3,200) + 2 Short ($1,100); 9 Ready. Card: **Orders short now: 6 · $4,300 · 9 ready**.

**Watch out:** "Orders short now" ($ of orders that truly cannot be built today) and the Category chart's "Revenue exposed" ($ of open orders holding an at-risk item) are **different** numbers. See Tile 4.

---

### Tile 2: Stockout Risk (`stockout_risk`)

**Question:** "Which items will run out, and how soon?"

**What you see:** a list sorted by urgency. Each row: item, `used N in 30d` (+ trend arrow), `avail`, `order`, `Nd cover`, `exposes $X`, and a risk badge. "View forecast detail" opens the full table with CSV download.

**Logic, step by step**

1. **Which items are listed:** items with Material Issue consumption in the window **and** cover under the Medium threshold (30 days). Excluded: items with no consumption, tube/acrylic rods, items whose Stock Policy is *Make to Order* or *Do Not Stock*.
2. **Daily use** (for the default 30-day window): the **TSB forecast** on the last 26 weekly totals, with one-off bulk orders removed. A custom *Trailing days* value uses the plain average instead. (TSB is explained in the Demand Forecast tile.)
3. **Days of stock** = on hand ÷ daily use. **Risk** from the 7/15/30 scale.
4. **Override:** if *Available < 0* the item is **always Critical** ("Open orders already exceed stock - restock now"), whatever its days.
5. **Order qty (suggested_qty)** = `ceil(daily use × target cover + safety stock − available)`, rounded up to the Item's Minimum Order Qty.
6. **Sort:** open orders exceed stock first, then Critical > High > Medium, then higher Revenue exposed, then fewer days.
7. **Confidence:** weeks (of the last 13) with any use: 8+ = High, 4 to 7 = Medium, below 4 = Low.

**Example 1: the normal chain.** `SPLC-150-PB` above → 20.0 days, **Medium**, order **58**. Row reads: `used 90 in 30d · avail 32 · order 58 · 20d cover`.

**Example 2: committed demand flips the risk.** Item X: 40 on hand, uses 1/day → 40 days → **Low**, not even listed. But open orders carry 55 units. Available = 40 − 55 = **−15** → shown as **Critical**, "Open orders already exceed stock".

**Example 3: unreliable BOM.** Using the `SPLC-150-PB` numbers: if the Lyfe BOM for its parents is rated *Unreliable*, the 18 BOM-implied units are **not** counted in committed. Committed = 10, available = 50, suggested = ceil(90 − 50) = **40** (instead of 58). The 18 still show under "BOM committed".

**Example 4: one-off bulk order (checked).** Last 90 days of issue entries for an item: 5, 6, 4, 5, **300**. The 300 is at least half the total and ≥ 3× the median of the others → treated as a **one-off** and removed from the window before dividing, and the row shows "⚑ unexpected order". With 5, 6, 4, 5, **12** the 12 is *not* a one-off and stays in.

**Example 5: finish families.** `WFL2-100-PB` (20 on hand, uses 1/day), `-SB` (5 on hand, uses 1/day), `-PBU` (0, uses 0). Pooled row `WFL2-100 ×3 finishes`: on hand 25, use 2/day → **12.5 days → High**. Without pooling, `-PB` would look Medium and `-SB` Critical although either can be finished from the same blank.

**Example 6: trend arrow.** 30-day daily use 3.0, 90-day daily use 2.0 → +50% → "↑ +50%". The arrow appears only when the difference is 25% or more and the item had 3+ issues in 90 days.

**Watch out for**
- Trailing days (funnel icon) changes only the use window, never the risk thresholds. The drilldown and Category chart still use 30 days unless the drilldown inherits the tile filter (it does for Stockout Risk).
- Source line says `Live` or `Snapshot · date`. Snapshot is used only if the setting is on and the snapshot is from yesterday or today.

---

### Tile 3: Restock List (`restock_list`)

**Question:** "What exactly should we buy, and how much?"

**What you see:** one buy list: Item, Type (Item / Tube / Acrylic), Risk, On Hand, Available, Days Left, **Order Qty**, Unit, **Last Received**, **Why**. Line above the table: "X not enough · Y enough stock for the next N days".

**Logic**

- **Cover for next N days** (tile filter, 1 to 365). Blank = each item's own target cover (Settings default 30 or the Item override).
- An item is **not enough** when `daily use × N + safety stock − available` is above 0 (so safety stock counts, not just use). Order qty is that shortfall rounded up, and then up to the Minimum Order Qty. With the backlog setting on, the formula is `max(committed, use × N) + safety − on hand`.
- **Tube rows:** feet short ÷ 100 ft per rod, rounded up to **whole rods**.
- **Acrylic rows:** `max(inches/day × cover − inches on hand, 0)` in **inches** (no stick count, because a class is a length range).
- **Last Received** = date of the item's newest Material Receipt. There are no POs here: restocking is booked only when it arrives, so an order already placed is invisible. Last Received is the only hint that something is on its way.
- **Safety stock** is part of the order, so this list can be longer than the "will run short" count on Demand Forecast.

**Example 1 (the Why text).** Item uses 1.5/day, N = 14 → needs 21. On hand 32, committed 20 → available 12 → short 9. MOQ 10 → **order 10**.
> "Uses 1.5/day, so needs 21 for the next 14 days. 32 on hand, 20 already promised to open orders: short 9, order 10 (minimum order 10)."

**Example 2: safety stock (checked).** Weekly use over 13 weeks (weeks with use): 10, 12, 14, 9, 11, 13, 10, 12, 9, 11 (3 quiet weeks count as 0), lead time 14 days (not measured yet), class B (z = 1.65) → **safety stock 12**. Class A (z = 2.05) → **15**. Reorder point at 1.6/day = ceil(1.6 × 14 + 12) = **35**.

**Example 3: tube row.** Pipe needs 338.4 ft more → 3.384 rods → **order 4 rods** (shown as "4 rods, 338 ft short").

**Example 4: Low-risk item still listed.** Item has 90 days of stock on the shelf, but 80% of it is promised to open orders. Available covers only 18 days of use; target is 30 → it **is** listed even though its risk badge is Low. The list follows the *shortfall*, not the risk colour.

**Watch out for:** the "Show the Y items with enough stock" link lists the ones that are fine, least cover first. Acrylic rows disappear when a Category / Item filter is set (acrylic classes are not items).

---

### Tile 4: Category-wise Stock Forecast (`category_forecast`)

**Question:** "Which product families are in trouble?"

**What you see:** stacked horizontal bars, one per **top-level Item Group**, coloured Critical / High / Medium / Low by **item count**. Toggle to **Revenue exposed ($)**. Click a bar → its sub-groups → click again → item popup. Extra rows for Tube and Acrylic. Footer: dead and excess stock counts.

**Logic**

- It does **not** forecast anything new. It counts the same item rows as Stockout Risk (but **including Low** items), grouped by Item Group.
- **No averaged "days of stock" per group**, so one stocked-out item can never hide in an average.
- **Revenue exposed ($)** for an at-risk (Critical/High/Medium) item = Σ(quantity × unit price) of that item's own order lines on open orders that have no Material Issue yet. Distinct orders are counted once (union, never a sum of per-group counts).
- Sort: Critical count, High count, $ exposed, name. First 6 shown, "show all N" to expand.

**Example: Revenue exposed.** Group "Railings": Critical item A has line 4 × $50 = $200 on order O-1; High item B has 1 × $120 on O-1 and 1 × $120 on O-2. Exposed = $200 + $240 = **$440**, orders = {O-1, O-2} = **2** (not 3).

**Example: Revenue exposed vs Orders short now.** An order holds a Medium-risk item with 25 days of stock. The Category chart counts that order's line as exposed. Buildable Orders still marks the order **Ready**, because stock is enough *today*. So: exposed = "could be a problem soon"; short = "cannot be built now".

**Watch out for:** items with no Item Group are bucketed as "Other".

---

### Tile 5: US Warehouse Replenishment tile (`us_replenishment`)

A summary of the separate US page (counts of Urgent / Reorder / Overstock, suggested units and value, last 1Click sync, first 10 items needing action) with a link to the full page. Logic is explained in the companion guide: **US_REPLENISHMENT_LOGIC_GUIDE.md**.

---

### Tile 6: Material Not In Stock, Total Required (`material_shortfall`)

**Question:** "Across *all* active orders, which materials are short in total?"

**What you see:** Item Code, Required Qty, In Stock, **Missing Qty** (rounded up). Column filters, Export CSV, a tick box.

**Logic: the requirement engine (same engine as the next two tiles)**

Scope: Lyfe Orders in an **undispatched** status (New … Ready for Dispatch, Pending India Dispatch, Awaiting India Components). Orders that already have a submitted Material Issue are excluded. Required quantity for each order line, first rule that applies:

1. **Line's Lyfe BOM** (the BOM on the line, else the Lyfe BOM of the line's item): `line qty × component qty × (1 + scrap%/100)`.
2. **No BOM → CJ shipment items of a previous Shipped/Completed order of the same item** (what a real order of this item actually consumed). Per unit = CJ qty ÷ qty sold on that order.
3. **Otherwise the line itself**: 1 per unit.

Custom orders' lines with no BOM are skipped (no reliable component list). **In stock** = Bin over all warehouses except Rejected.

**Example (checked, from the engineering reference).** Stock: C1 = 8, C2 = 10. O-1 has 3 units of kit K, O-2 has 2 units. BOM of K: C1 qty 2 with 5% scrap (**2.1 per unit**), C2 qty 1.

| | C1 | C2 |
|---|---|---|
| O-1 (3 units) | 6.3 | 3 |
| O-2 (2 units) | 4.2 | 2 |
| **Total required** | **10.5** | **5** |
| In stock | 8 | 10 |
| Missing | 2.5 → shown **3** | none → not listed |

**Example: CJ fallback.** Item F has no BOM. A previous shipped order sold 2 F and its CJ rows hold 6 of F → 3 per unit. A new order of 4 F requires **12 F**.

**Tick box:** ticking an item fills the Item Code filter of the next tile (exact match) so you can see which orders need it.

---

### Tile 7: Active Orders, Material Not In Stock (`order_stock_shortfall`)

**Question:** "Which individual orders cannot be completed from current stock?"

**Logic:** same requirement engine; one row per (order, component) where **that order's requirement alone** is above total stock. Columns: Internal Order ID, Order Type, Item Code, Required, In Stock, Missing, Order Date, Order Value.

**Stock is not shared between orders on this tile.** Each order is compared with the *full* stock.

**Example (same data as Tile 6).** O-1 needs 6.3 of C1 (stock 8) → fine. O-2 needs 4.2 → fine. This tile shows **nothing**, but Tile 6 lists C1 (10.5 > 8). Another case: two orders each need 6 of an item with 10 in stock → neither appears here, Tile 6 lists it (12 > 10). To see who is behind a shortage, tick the item in Tile 6.

---

### Tile 8: Buildable Orders (`buildable_orders`)

**Question:** "Which active orders can we build today?"

**Logic:** same population and requirements as Tiles 6 and 7, but stock **is handed out**, oldest order date first:

- **Ready**: every component available → the order **takes** that stock.
- **Partial**: some components short, others available.
- **Short**: every component short.
- Partial and Short orders **take no stock**, so a blocked order never holds material back from a later order that can be built.

**Example (checked, from the engineering reference).** Two orders each need 6 of item Z; 10 in stock.
- Older order: needs 6 ≤ 10 → **Ready**, takes 6, leaves 4.
- Newer order: needs 6 > 4 → **Short** ("Z: need 6, have 4, missing 2").
Contrast with Tile 7, which would list neither.

**Summary line:** ready / partial / short counts, **Blocked** = $ of Partial + Short orders, **Ready** = $ of Ready orders.

**Watch out for:** priority is order *date*, not promised dispatch date; it is a first-come pass, not an optimiser. With a page filter, only orders that need a matching component are shown, but their Ready/Partial/Short status is still computed over *all* active orders.

---

### Tile 9: Tube Stock (`tube_stock`)

**Question:** "Which tube pipes will run out, and how many rods to order?"

**Logic**

- One row per **pipe** (tube material × diameter, the "store item"). Finishes and cut lengths are rolled up, because finish is applied after cutting.
- **Feet on hand** = rod feet (Bin of the store item) + pre-cut pieces × their length. Pre-cut SKUs are named `{length}FT-TB-…` (`7P5FT` = 7.5 ft); their quantity is a **piece count** even if the unit says Foot.
- A pre-cut SKU belongs to the pipe it was **most often cut from** (taken from Material Issue tube rows).
- **Used feet** = rod feet issued − rod feet returned + (pre-cut issued − returned pieces) × length.
- `ft/day = used ÷ window days` · `days = feet on hand ÷ ft/day` · risk 7/15/30.
- `keep_ft = ft/day × target cover` · `order_ft = max(keep_ft − feet on hand, 0)` · `order_rods = ceil(order_ft ÷ 100)` (a rod is assumed 100 ft).

**Example (checked).** Pipe "Brass Tube 1.5'' Dia": rod Bin 79.6 ft; 5 pre-cut `4FT-TB-150-AB` = 20 ft → **99.6 ft on hand** (0.996 rods). 30 days: rod issued 400 ft, returned 10 ft, pre-cut issued 12 × 4 = 48 ft → **used 438 ft**.

| Step | Calculation | Result |
|---|---|---|
| ft/day | 438 ÷ 30 | 14.6 |
| Days of stock | 99.6 ÷ 14.6 | **6.8 → Critical** |
| Keep | 14.6 × 30 | 438 ft |
| Order | 438 − 99.6 | 338.4 ft |
| Rods | ceil(3.384) | **4 rods** |

**Watch out for:** pre-cut pieces are booked as a Material Receipt without a rod issue, so rod feet on the books can be overstated until a physical count (Stock Reconciliation) corrects it. "Unlinked cut tubes" (cut SKUs never cut from a known pipe) get one "(family) (cut pieces)" row with no rod count.

---

### Tile 10: Acrylic Rod Stock (`acrylic_stock`)

**Question:** "Which acrylic diameter / stick-length classes are running low?"

**Logic**

- Sticks are SKUs `ACR-{diameter}-{length}L` (round) and `RAC-{width}-{length}L` (rectangular). Rod **caps** `RAC-…-{finish}` are *not* rods.
- Length classes: **≤12**, **13-24**, **25-48**, **>48** inches.
- **On hand (inches)** = Σ sticks × stick length, per diameter × class.
- **Demand (inches)** = whole sticks **issued − returned**, × stick length, in the class of the stick drawn (so waste is included).
- `inches/day = demand ÷ window` · `days = on hand ÷ inches/day` · risk 7/15/30.

**Example (checked).** 1.5" rods, class **>48**: 10 sticks of `ACR-150-60L` + 5 of `ACR-150-72L` → 600 + 360 = **960 in**. 30 days: 6 × 60L issued, 1 returned, 2 × 72L issued → (6 − 1) × 60 + 2 × 72 = **444 in**. Per day 14.8 · days = 960 ÷ 14.8 = **64.9 → Low**.

**Watch out for:** a longer stick can serve a shorter cut, so short classes look worse than they are. The drilldown shows *pieces cut* (demand shape); the tile counts *whole sticks drawn*.

---

### Tile 11: Demand Trends (`trending_items`)

**Question:** "What is selling more (or less) than before?"

**Logic:** compares the **last N days** with the **N days before** (equal periods, any N). Use = units issued (finishes pooled).
- **Categories table:** last N, previous N, change %, direction: ↑ more than +10%, ↓ below −10%, → steady, "new" if nothing before.
- **Rising items:** at least **3 issues** in the last N days, **≥ 25% more** than before (or new), and not a single bulk order. Ranked by unit rise.
- Chart: top 6 categories per period (day / week / 30-day blocks depending on N).

**Example.** Item: 40 units last 30 days, 25 before → **+60%**, 3+ issues → listed as rising (rise = +15 units). Item B: 100 vs 95 → +5% → not listed. Item C: 60 units but one single issue of 55 → bulk order → not listed.

**Watch out for:** Material Issue history starts 12 May 2026; if the "previous" period reaches before that the tile warns that rises look bigger. This is **not** the same rule as the arrow on Stockout Risk (which compares with the 90-day average).

---

### Tile 12: Demand Forecast (`demand_forecast`)

**Question:** "How much will we use in the next N days, and what is left after?"

**Logic**

- **Per day: TSB** (Teunter-Syntetos-Babai), built for items that sell in bursts. It keeps two smoothed numbers over the last 26 weeks: how **often** the item sells (probability, β = 0.1) and the **typical size** when it does (α = 0.1), and multiplies them: `per day = probability × size ÷ 7`. If an item stops selling, the probability decays towards 0, so dying items fade out.
- **Forecast** = per day × N · **On open orders** = committed · **Expected** = forecast + open orders (the larger of the two when the backlog setting is on) · **Left after N days** = on hand − expected (red = short).
- Sparkline: last 12 weeks issued (grey) + forecast (dashed, flat).
- Tube items sit in a collapsed "Tube items (N)" section.

**Example (checked).** An item sold in bursts: 12, 10, 14, 11, 13, 12 units in 6 of 26 weeks, zero in the others. Plain 26-week average = **0.396/day**. TSB = **0.436/day** because recent weeks sold more often. Next 30 days: forecast 13.1; open orders 8; expected 21.1; on hand 15 → **left −6.1 → short**.

**Why TSB.** Backtest (14 weekly origins, 2026-11-06): forecast error fell from 123% to 95%, bias from +43% to +9%, leftover stock from 83% to 52%.

**Watch out for:** "will run short" here counts only expected use, while the Restock List also orders safety stock, so the Restock List may be longer.

---

### Tile 13: Base-level Forecast (`base_forecast`)

**Question:** "Is demand clearer if I look at the base product instead of each finish?"

**Logic:** groups the finished SKUs of one **base product + base state** (e.g. CMBR-200 Matte) and forecasts them together. Only groups with **2+ finishes** and use in the last 30 days. Stock = finishes' stock added. Safety stock is sized on the *combined* use, so it is smaller than the sum of per-finish safety (773 vs 958 units over all groups on 2026-11-06). Order qty = cover + safety − available.

**Example (illustrative numbers).** Matte base with three finishes: each uses 4/day irregularly; combined demand is smoother, so one safety stock of 70 replaces three of 40 (120). On hand pooled 300, forecast 12/day → 25 days → Medium.

**Watch out for:** tube and acrylic are not included; base stock itself is not tracked in the ERP, only per-finish stock.

---

### Tile 14: Material Usage (`material_usage`)

**Question:** "What did the factory issue the most?"

**Logic:** net units per item = **Issues − Returns** on submitted *Material Issue for Order* documents, by the document's own Date, trailing N days (default 30), items above 0 only. Mixed units (tube feet, acrylic sticks, finished pieces), so it is labelled "Most-Issued".

**Example.** Item P: Issue 120, Return 8 → **112**. Item Q: Issue 30, Return 30 → 0 → not listed.

**Watch out for:** the drilldown always uses 30 days. Unlike the stockout tiles, no warehouse is excluded.

---

### Tile 15: Forecast vs Actual (`forecast_accuracy`)

**Question:** "Can we trust the forecast?"

**Logic:** for each complete week, rebuild what the **30-day-average rule** would have said that morning (Material Issues in the 30 days before ÷ 30 × 7) and compare with units actually issued. Reports total forecast vs used (bias) and product-by-product-week error (WAPE = Σ|error| ÷ Σ actual).

**Example (checked, real data, 12 weeks).** Forecast 5,075 vs used 5,407 → **6.1% too low overall**, but **93.8% off product by product, week by week**. Reading: the total is fine; individual products are hard to predict, so lean on safety stock and the Restock List, not a single-week number.

**Watch out for:** this tile still scores the *old* 30-day rule, not TSB. The nightly accuracy job scores whatever the snapshot stored (TSB from 2026-11-06).

---

### Tile 16: Stock Data Quality (`data_quality`)

**Question:** "Can I trust what I'm looking at?"

Checklist lines (click a line for where to fix and affected records): BOM components not matched to an Item · open order lines without an Item · components where BOM disagrees with actual issues (ratio outside 0.7 to 1.3 on 20+ orders) · acrylic rods with unit Inch · acrylic rods with decimal written as P · pending Material Requests (ignored) · **dead stock** (on hand, nothing issued in 90 days) · **excess stock** (> 3 × target cover days of use) · last week's forecast bias beyond ±30% · nightly snapshot older than a day.

**Example.** Item R: 50 on hand, 0 issued in 90 days → **dead**. Item S: 300 on hand, 90-day use 90 units (1/day), target cover 30 → limit 90 → **excess**.

---

## Part 3. Cross-cutting logic

### 3.1 The Lyfe BOM (not the ERPNext BOM)
One level only. Effective qty = `qty × (1 + scrap%/100)` (qty 2, 5% scrap → 2.1). Child item is matched by Item Code, else SKU as Item name, else custom SKU; unmatched children are ignored and counted in Data Quality. An order's own component list (if present) replaces line BOMs for the whole order.

**Calibration:** for orders with a Material Issue in the last 90 days, ratio = actually issued ÷ BOM-implied, per component, on orders where the BOM predicted it. **Reliable** = ratio 0.7 to 1.3 on ≥ 20 orders; **Unreliable** = outside that on ≥ 20 orders; otherwise **Unknown**. Unreliable BOM units are left out of committed.

### 3.2 What runs automatically

| When | What |
|---|---|
| Daily 01:15 | Nightly snapshot of every item's forecast row (Stock Forecast Snapshot); optional Slack risk alert |
| Monday 02:30 | ABC class refresh (A < 80% of volume, B < 95%, else C) · forecast accuracy scoring · lead-time update from Lyfe PO receipts |
| Daily briefing hour | Founder Briefing to Slack with projected stockouts and a Tube & Acrylic block |

**Lead time:** each Lyfe PO gets an "Ordered On" date; days from that to each receipt are samples. Median becomes `Item.lead_time_days` (per item at 3+ receipts, else per item group at 5+). Until measured, 14 days is used.

### 3.3 Settings that change the numbers

| Setting | Where | Effect |
|---|---|---|
| Target Cover (days), default 30 | Stock Forecast Settings (or per-Item override) | Order quantity |
| Demand Window (days), default 30 | Stock Forecast Settings | Default Trailing days |
| Stockout Tile Reads Snapshot | Stock Forecast Settings | Faster tiles, up to a day old |
| Backlog-aware Suggested Qty | Stock Forecast Settings | `max(committed, use × cover) − on hand` instead of `use × cover − available`; avoids counting open orders twice |
| Risk thresholds 7 / 15 / 30 | Founder Briefing Settings | Critical / High / Medium cut-offs |
| Send Risk Alerts to Slack | Stock Forecast Settings | Nightly alert (new or worsened Critical/High, plus a reminder every 7th consecutive Critical day) |

### 3.4 Known limits (so nobody is surprised)
- Only ~6 months of ledger history (from May 2026): no seasonality.
- Purchase Orders are not used. Incoming stock is seen only when received.
- Waste and offcut are not modelled for tube beyond what is issued.
- Priority in Buildable Orders is order date, not promised dispatch.

---

## Part 4. Quick "why is this number what it is?" checklist

1. **Item shows Critical but has many days of stock?** Check *Available*: committed demand may exceed stock.
2. **Item missing from Stockout Risk?** No Material Issue in the window, cover ≥ 30 days, Make to Order / Do Not Stock, or it is tube/acrylic (own tiles).
3. **Order qty is bigger than expected?** Safety stock, MOQ rounding, and backlog are all added.
4. **Item is on Restock List but Demand Forecast says it won't run short?** Restock includes safety stock and the cover N you typed.
5. **Total tile lists an item, order tile doesn't?** Stock is shared in the total, not per order.
6. **Order is "Ready" but its item is at risk?** Ready = stock is enough *today*; risk = enough for how long.
7. **Numbers differ from yesterday's Slack?** Slack uses a 7-day lookahead (Critical only) and no enrichment.
8. **Row is `Snapshot · date`?** The tile is reading last night's snapshot; press Refresh or turn the snapshot setting off for live.

---

## Part 5. Who can see what

| Surface | Who | Notes |
|---|---|---|
| Page `/app/stock-forecast` and every stock tile / drilldown | System Manager, Super Admin, Factory | A Factory user can open this page but is refused by every founder revenue / profit endpoint |
| Founder Dashboard `/app/founder-dashboard` | System Manager, Super Admin | Shows only a one-line **Stock banner** (Critical / Watch / No stockout risk, "N critical - N high", "$X exposed", link to this page) |
| **US Warehouse Replenishment** tile | System Manager, Super Admin, US Replenishment Manager | Factory does not get it; the endpoint refuses them |
| **Stock Data Quality** tile, badge and detail | Everyone who can open the page, if *Stock Data Quality (public view available)* is ticked in Stock Forecast Settings (default on). If unticked, only the user `Administrator` | Unticked also hides the tile, the KPI card and the MCP tool |
| AI / MCP tools for the stock tiles | System Manager, Super Admin | Same data as the page; Factory has no MCP access to these |
| MCP forecasting tools (`get_stockout_forecast`, tube, acrylic, component, dead/excess) | Super Admin or Founder-AI role | Raw lists, not the page's enriched rows |
| Slack posts | Whoever is in the webhook channel | Webhook is in Founder Briefing Settings |

**Showing / hiding tiles:** Founder Dashboard Settings > Tile Config. A tile with *Visible* unticked is not shown or loaded. A tile with **no** Tile Config row is shown. Tile **order and width on this page are fixed in code** (Part 1), not taken from Tile Config.

---

## Part 6. Filters, drilldowns and exports per tile

### 6.1 Which filters each tile accepts

| Tile | Page bar (Category, Item Code) | Own funnel filter |
|---|---|---|
| KPI strip | Yes | none |
| Stockout Risk | Yes | Category, **Trailing days** (1-365, blank = Settings window) |
| Category-wise | Yes | Category |
| Restock List | Yes | Category, Item, **Cover for next N days** (1-365) |
| Buildable Orders | Yes | Category, Item |
| Material Not In Stock (both) | Yes | Category, Item |
| Material Usage | Yes | **Trailing days** |
| Demand Trends | Yes | Category, Item, **Last N days**, **Top N** |
| Demand Forecast | Yes | Category, Item, **Next N days**, **Top N** |
| Base-level Forecast | Category only | Category, **Top N** (default 25, max 200) |
| Forecast vs Actual | Yes | Category, Item, **Last N weeks**, **Top N** |
| Tube Stock | No | **Trailing days** |
| Acrylic Rod Stock | No | **Trailing days** |
| Stock Data Quality | No | none |
| US Warehouse Replenishment | No | none (use the full US page) |

Rules: a blank or invalid number falls back to the default; the window is clamped to 1-365. A **custom Trailing days is always worked out live** (the nightly snapshot is 30-day only). Picking an Item Code finishes pools: choosing one finish of a product shows that product's merged "×N finishes" row.

### 6.2 What the drilldowns and downloads give

| Where | What you get |
|---|---|
| Stockout Risk > **View forecast detail** | Full list (search, risk filter). Row caret opens: uses per day, committed to open orders, confidence, BOM committed with reliability and ratio, **Where used**, recommended action, safety stock, reorder point, and **Go to** (Item form, Stock Ledger). **Download restock list (CSV)** has every row, not just those the search shows |
| Where used | Every Lyfe BOM containing the item: parent item, BOM, quantity per unit (scrap not added), number of open orders carrying that parent |
| Category chart | Level 1 = top-level groups; click a group → its own Item Groups (breadcrumb + Back); click a sub-group → item popup with extra **Revenue exposed** and **Orders** columns and CSV. A pinned **All critical items** row opens the popup preset to Critical |
| Material Not In Stock tiles, Buildable Orders, Restock List | Column filters (typing in a box hides rows that do not contain the text) and **Export** CSV of exactly what is visible. The count badge follows the filters |
| Material Usage > Full List | Every item with net qty issued (always 30 days) |
| Tube > use / keep / order per pipe | Table per pipe: used, per day, on hand, days left, keep, order ft, order rods, risk |
| Acrylic > usage by diameter and cut length | Pieces and inches by cut length (shape of demand; the tile itself counts whole sticks) |
| Stock Data Quality lines | Dialog with "Where to fix" and up to 200 affected records, each linking to its form |

### 6.3 Row fields of the stock forecast (what a Stockout Risk row contains)

| Field | Meaning |
|---|---|
| `current_qty` | On hand (all warehouses except Rejected) |
| `avg_daily_outflow` | Daily use (TSB for the default window, else the plain average) |
| `days_of_stock` | On hand ÷ daily use |
| `risk_level` | Critical / High / Medium / Low (Critical forced when available < 0) |
| `committed_qty`, `bom_committed_qty` | Promised to open orders; the BOM-implied part |
| `bom_calibration`, `bom_reliability` | Actual ÷ BOM ratio and Reliable / Unreliable / Unknown |
| `available_qty`, `available_days` | On hand − committed; and that ÷ daily use (never below 0) |
| `stockout_date` | Today + whole days of **on-hand** cover |
| `safety_stock`, `reorder_point` | Part 2, Tile 3 (Example 2) |
| `suggested_qty` | Order quantity (with MOQ rounding) |
| `confidence`, `movement_weeks` | High / Medium / Low and weeks of use out of the last 13 |
| `blocked_amount`, `blocked_orders` | Revenue / orders exposed (field names kept so old integrations don't break) |
| `variants` | The finishes pooled into this family row |
| `unexpected_order_qty`, `trend_pct` | One-off bulk order flag; change versus the 90-day rate |

---

## Part 7. Stock Data Quality: every check

| Line | What it counts | Severity | Why it matters |
|---|---|---|---|
| Lyfe BOM components not matched to an Item | BOM child rows whose code / SKU matches no Item | warn | Ignored in BOM demand, so committed is too low |
| Open order lines without an Item | Non-adjustment open-order lines with no ERP Item (fee, price-difference, fabrication and made-to-order CSTM lines are not listed; the name is shown when the line has no SKU) | warn | Their quantity cannot be counted as committed |
| Components where BOM disagrees with actual issues | Ratio outside 0.7-1.3 on at least 20 orders | warn | That BOM is left out of committed demand |
| Acrylic rods with unit Inch | Rod SKUs whose stock unit is Inch | warn | Quantities are misread |
| Acrylic rods with a decimal written as P | e.g. `ACR-075-3P75L` | warn | Not recognised as rods |
| Pending Material Requests (not used) | Submitted Material Requests still Pending | info | They inflate Bin "indented qty"; this page ignores them |
| Stock with no use in 90 days (dead) | On hand > 0, nothing issued in 90 days | info | Review list; seasonal items may look dead (history is short) |
| Stock above 3× target cover (excess) | On hand > 3 × target cover days of use (target = Item's own, else 30) | info | Money tied up |
| Last week's forecast bias beyond ±30% | Latest "All" accuracy record | warn | The forecast is drifting |
| Nightly snapshot older than a day (or missing) | Newest snapshot before yesterday | **bad** | History, alerts and accuracy stop |

The count shown on the badge / pill is the number of **lines** that are not OK (including informational ones), not the number of records. Dead and excess consider only stocked, non-disabled items outside the Rejected warehouse, excluding tube / acrylic and Make-to-Order / Do-Not-Stock items.

---

## Part 8. Snapshots, alerts, accuracy and Slack

### 8.1 Nightly snapshot (daily 01:15)
- One **Stock Forecast Snapshot** row per item with consumption, over a 365-day horizon (so Low-cover items are included), enriched with committed, available, stock-out date, suggested qty, confidence, safety stock and BOM columns. Re-running the same day deletes and rewrites that day's rows. Snapshots older than 400 days are purged.
- **Stockout Tile Reads Snapshot** (default off): when on, and the newest snapshot is today's or yesterday's, the Stockout Risk, Category, KPI and drilldown rows come from it (fast). Otherwise they are live. Revenue exposed is always live. Settings > *Last Snapshot Run* shows "date time OK - N rows in Ns" or "FAILED - see Error Log".
- If the snapshot is stale or missing the page quietly falls back to live; the Data Quality line goes **bad**.

### 8.2 Risk alerts to Slack (optional: *Send Risk Alerts to Slack*, default off)
After each snapshot, needs at least two snapshot dates. An item is alerted when it is **Critical or High today** and either is **worse than the previous snapshot**, or **is new**, or is a **Critical reminder** (every 7th consecutive Critical snapshot). One grouped message per day: most severe first then fewest days, at most 15 items plus "+N more", each line with name, code, level, days, units, and "_still critical_" for reminders.

**Example:** item is Medium on Monday, High on Tuesday → alerted Tuesday. Stays High on Wednesday → not alerted again. Critical on Thursday → alerted (worse). Stays Critical for 7 snapshots → one "still critical" reminder.

### 8.3 Weekly jobs (Monday 02:30)
- **ABC class:** volume only. Items ranked by 90-day Material Issue units; share before adding the item: below 80% = A, below 95% = B, else C. Skips items with *Keep ABC Class Manual*. The class sets the safety-stock z (A 2.05, B 1.65, C 1.28; unclassified uses B) and is used in accuracy reporting.
- **Accuracy:** scores the snapshot from about a week ago: forecast = daily use × 7, actual = units issued in the next 7 days. **WAPE** = Σ|error| ÷ Σ actual. **Bias** = (Σ forecast − Σ actual) ÷ Σ actual. Stored for All, A, B, C and Unclassified. Nothing is written until a snapshot is a week old.
- **Lead time:** writes the median measured lead time to `Item.lead_time_days` (item at 3+ receipts, else item group at 5+).

**Example (accuracy):** forecast 100, actual 120 for item 1; forecast 50, actual 30 for item 2. WAPE = (20 + 20) ÷ 150 = 26.7%. Bias = (150 − 150) ÷ 150 = 0%. Individually off, overall unbiased, the same pattern as the Forecast vs Actual tile.

### 8.4 Daily Founder Briefing (Slack)
Sent once a day at `briefing_send_hour_ist` (default 8), only if *Is Active* is on. Stock parts:
- **Projected Stockouts:** items under the lookahead (default 7 days, so effectively Critical), top 10 with name, code, days left, units, risk and action, "+N more", and a "vs yesterday" change. This list is **not** the page's enriched list.
- **Tube & Acrylic Stock:** rows that are not Low, with feet / inches on hand and use per day.
A failing section never blocks the others.

---

## Part 9. Caching, refresh and failure behaviour

- Per-tile **↻** refreshes just that tile; the page **Refresh** reloads everything. Tiles in a tab load only the first time the tab is opened. A filter change refreshes only tiles already loaded.
- The stockout rows are cached for about 45 seconds; the cache key includes the filters, the snapshot setting and the backlog setting.
- A tile that fails shows "Data unavailable - see Error Log" with **Retry**; the others are unaffected.
- No consumption at all, or everything filtered out, shows the tile's empty message, not an error.
- An item with no Bin row: on hand 0, Critical, "Stocked out - restock now".
- The tab you last used is remembered per browser.

### Edge cases that surprise people
| What you see | Why |
|---|---|
| Item not on Stockout Risk | No issue in the window, cover ≥ 30 days, Make to Order / Do Not Stock, or tube / acrylic |
| Item on Restock List but not on Stockout Risk | The list follows the shortfall (including safety stock and open orders), not the risk colour |
| Order Ready but item at risk | Ready = enough stock today; risk = enough for how long |
| Tube cut length shows 0 on hand and "short" | Cut lengths are cut from pipe as needed; look at the pipe row |
| Rod feet look too high | Pre-cut pieces are received without a rod issue until a physical count is booked |
| Numbers differ from Slack | Slack uses a 7-day lookahead and unenriched rows |

---

## Part 10. Settings reference

| Setting | Where | Default | What it does |
|---|---|---|---|
| Demand Window (days) | Stock Forecast Settings | 30 | Default Trailing days of the live tiles and the snapshot |
| Target Cover (days) | Stock Forecast Settings | 30 | Days of cover the order quantity restores |
| Stockout Tile Reads Snapshot | Stock Forecast Settings | off | Fast tiles from last night's data |
| Backlog-aware Suggested Qty | Stock Forecast Settings | off | `max(committed, use × cover) − on hand` instead of `use × cover − available` |
| Send Risk Alerts to Slack | Stock Forecast Settings | off | Nightly alert (8.2) |
| Stock Data Quality (public view available) | Stock Forecast Settings | on | Who can see Data Quality |
| Critical / High / Medium thresholds | Founder Briefing Settings | 7 / 15 / 30 days | Risk levels |
| Item > Target Cover (days) | Item | blank | Per-item cover override |
| Item > Stock Policy | Item | blank | *Make to Order* and *Do Not Stock* leave the item out |
| Item > Minimum Order Qty | Item | blank | Order quantities round up to a multiple |
| Item > ABC Class / Keep ABC Class Manual | Item | automatic | Safety-stock z |
| Item > Lead Time in days | Item | measured weekly | Safety stock and reorder point (14 days until measured) |

---

## Part 11. Glossary

| Term | Meaning |
|---|---|
| **MIFO** | Material Issue for Order: the document that issues material to an order |
| **Material Issue** | The stock entry type that records what the factory used; the only source of "use" |
| **Committed** | Units promised to open orders with no Material Issue yet |
| **Available** | On hand − committed |
| **Revenue exposed** | $ of order lines on open orders holding a Critical / High / Medium item |
| **Orders short now** | $ of active orders that cannot be fully built from stock today |
| **TSB** | Teunter-Syntetos-Babai: a forecast for items that sell in bursts (Tile 12) |
| **WAPE / bias** | Total absolute error ÷ total actual; and overall over- or under-forecast |
| **One-off order** | A single issue ≥ half the 90-day total and ≥ 3× the median of the others; removed from the daily rate |
| **Family / finishes** | One product sold in several finishes, shown as one pooled row |
| **Reliable / Unreliable BOM** | Whether the Lyfe BOM predicted real issues within 0.7-1.3× on 20+ orders |
| **Rod** | 100 ft of tube (assumed) |
| **Snapshot** | The nightly saved copy of every item's forecast row |

---

## Part 12. Getting the same information through MCP (AI tools)

Every tile and drilldown on this page has a read-only AI tool that returns exactly what the page shows (same numbers, same filters). Roles: System Manager and Super Admin (the US Warehouse Replenishment tool also allows US Replenishment Manager). Factory can use the page but has no MCP access to these.

| Page element | Tool | Useful arguments |
|---|---|---|
| KPI strip | `get_founder_tile_stock_summary` | `item_group`, `item_code` |
| Stockout Risk | `get_founder_tile_stockout_risk` | `item_group`, `item_code`, `days` |
| Stockout Risk > View forecast detail | `get_founder_stockout_risk_drilldown` | `item_group`, `item_code`, `days`, `include_all`, `exact` |
| Restock List | `get_founder_tile_restock_list` | `item_group`, `item_code`, `days` (cover for next N days) |
| Category-wise Stock Forecast (level 1) | `get_founder_tile_category_forecast` | `item_group`, `item_code` |
| Category-wise (level 2, a group's sub-groups) | `get_founder_category_forecast_children` | `group`, `item_group`, `item_code` |
| Material Usage | `get_founder_tile_material_usage` and `get_founder_material_usage_drilldown` | `period_days` |
| Demand Trends | `get_founder_tile_trending_items` | `item_group`, `item_code`, `days`, `top` |
| Demand Forecast | `get_founder_tile_demand_forecast` | `item_group`, `item_code`, `days`, `top` |
| Base-level Forecast | `get_founder_tile_base_forecast` | `item_group`, `top` |
| Forecast vs Actual | `get_founder_tile_forecast_accuracy` | `item_group`, `item_code`, `weeks`, `top` |
| Buildable Orders | `get_founder_tile_buildable_orders` | `item_group`, `item_code` |
| Material Not In Stock (Total) | `get_founder_tile_material_shortfall` | `item_group`, `item_code` |
| Active Orders Material Not In Stock | `get_founder_tile_order_stock_shortfall` | `item_group`, `item_code` |
| Tube Stock | `get_founder_tile_tube_stock` and `get_founder_tube_stock_drilldown` | `days` |
| Acrylic Rod Stock | `get_founder_tile_acrylic_stock` and `get_founder_acrylic_stock_drilldown` | `days` |
| Stock Data Quality | `get_founder_tile_data_quality` and `get_founder_data_quality_detail` | `key` (the check) |
| Where used | `get_founder_where_used` | `item_code` |
| Nightly forecast accuracy (WAPE / bias) | `get_founder_stock_forecast_accuracy` | `weeks` |
| US Warehouse Replenishment tile | `get_founder_tile_us_replenishment` | none (full page: see the US guide, Part 8) |
| Everything at once | `get_founder_summary` | |

Raw lists that match the Slack briefing (not the page's enriched rows): `get_stockout_forecast`, `get_tube_stock_forecast`, `get_acrylic_stock_forecast`, `get_component_forecast(item_code)`, `get_dead_and_excess_stock` (Super Admin or Founder-AI role).

**Example questions**
- "What should we order for the next 14 days?" → `get_founder_tile_restock_list(days=14)`.
- "Why is SPLC-150-PB Critical?" → `get_founder_stockout_risk_drilldown(item_code="SPLC-150-PB")` then `get_founder_where_used`.
- "Which orders can't we build today?" → `get_founder_tile_buildable_orders`.
- "Can we trust the forecast?" → `get_founder_tile_forecast_accuracy(weeks=12)` and `get_founder_stock_forecast_accuracy`.

