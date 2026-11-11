# US Warehouse Replenishment Dashboard: Logic Guide (tile by tile, with examples)

**Page:** `/app/us-replenishment` · **Audience:** anyone who needs to understand why the page says "send 147 units of item X": purchasing, factory, customer service, founders, new joiners.
**Version:** 1.1 · **Written:** 2026-11-11 · **Technical reference:** `docs/us-replenishment-dashboard.md`. If this guide and the code disagree, the code is right.

Examples marked **(checked)** were run through the real calculation code (`us_replenishment/calc.py`) with the default settings.

---

## Part 0. The question this page answers

> For each standard item, **do we need to send more stock to the 1Click US warehouse, and how many units should go in the next transfer?**

It looks at three things:

1. **How fast does the US buy this item?** (weekly demand)
2. **How long does a replenishment take to arrive?** (lead time)
3. **What do we already have or have on the way?** (stock position)

Then it tells you the status (Urgent / Reorder / OK / Overstock / Not stocked yet) and a suggested quantity.

### Words used on this page

| Word | Plain meaning |
|---|---|
| **Available** | Units at 1Click that can be sold right now |
| **In transit** | Our own stock on a transfer that has not reached 1Click yet |
| **Stock position** | Available + in transit (+ in production, always 0 today). What we "have or have coming" |
| **Lead time** | How long a new shipment takes from order to being sellable at 1Click: 4.5 weeks by default |
| **Safety stock** | Extra stock for demand spikes while we wait for a shipment. Bumpier demand means more |
| **Reorder point** | The position at which we should already have ordered: demand during the lead time + safety stock |
| **Target** | The position we want to be back at: reorder point + 4 weeks of sales |
| **Weeks of cover** | How many weeks the stock lasts at the current weekly demand |
| **Suggested qty** | Target − stock position (never below 0) |

### Who can use it
System Manager, Super Admin and the role **US Replenishment Manager**. Creating the draft transfer also needs create permission on *Stock Transfer Order 1Click* (Factory, Customer Service, System Manager, Super Admin).

### Which items are on the page
Standard items only. **Left out:** custom items, items whose stock policy is *Make to Order* / *Do Not Stock*, tube and acrylic-rod items (they have their own forecasts), disabled items. US order lines with **no Item linked** are not counted (the page prints a note saying how many lines and units).

---

## Part 1. The numbers in one table (default settings)

All of these are editable in **US Replenishment Settings**. Nothing is hard-coded in the maths.

| Setting | Default | Used for |
|---|---|---|
| Demand country | US | Which orders count as US demand |
| History weeks | 52 | How far back to look |
| Minimum history weeks | 8 | Below this, confidence is **Low** |
| Recent weeks / weight | 4 / 50% | Newest weeks count most |
| Mid weeks end / weight | 12 / 30% | Weeks 5 to 12 |
| Old weeks end / weight | 26 / 20% | Weeks 13 to 26 |
| Factory working days | 3 | Production time |
| Working days per week | 6 | Converts factory days to weeks |
| Transit weeks | 3 | India to US |
| Customs weeks | 1 | Customs and receiving at 1Click |
| Service level | 95% | Safety stock (Z = 1.645) |
| Coverage weeks | 4 | How many weeks of sales to hold beyond the reorder point |
| Overstock weeks | 12 | Above this many weeks of cover = Overstock |
| New-item window / min selling weeks | 12 / 6 | "Not stocked yet" rule |
| Sync every / stale after | 4 h / 12 h | 1Click stock sync and warning banner |
| Snapshot retention | 180 days | How long stock history is kept |
| Value price list | Standard Selling | Prices for "Suggested value" |

**Lead time (weeks)** = factory working days ÷ working days per week + transit weeks + customs weeks = 3 ÷ 6 + 3 + 1 = **4.5 weeks** (checked).

---

## Part 2. Tile-by-tile logic

The page has: a **sync banner**, a **tile strip** (six tiles), a **filter bar**, the **item table**, the **item drill-down**, and three **actions**. Each section below follows: *What you see → Logic → Example → Watch out for*.

---

### Section 1: Sync banner

**What you see:** a yellow banner when something is wrong with the 1Click stock feed.

**Logic**
- The sync only runs when the 1Click integration is switched on (*Is Active* in Oneclick Settings); if it is off, nothing is ever synced and the banner says "never synced".
- The page never calls 1Click when you open it. It reads our own saved copy (the **OneClick Stock Snapshot**), refreshed by an hourly job that runs once `sync_every_hours` (4) have passed, or by the **Sync 1Click now** button.
- Banner rules:
  - Never synced: "1Click stock has never been synced, so there are no numbers to show."
  - Latest sync **failed**: "The latest 1Click sync failed. Showing the last good sync (time)."
  - Last good sync older than `stale_after_hours` (12): "The last good 1Click sync is N hours old."
- On failure the table shows the **last good snapshot, never zeros**. An empty or paged response counts as a failure. A response where everything is 0 is saved as real stock (a wrong warehouse id is rejected by 1Click, so zero stock is genuine).

**Example:** the 04:00 sync fails (1Click timeout). At 09:00 the page still shows the 00:00 data, with "The latest 1Click sync failed. Showing the last good sync (…)".

---

### Tile 1: Urgent

**Question:** "How many items need stock **now**?"

**Logic:** count of rows with status **Urgent**. An item is Urgent when **either**:
- its **stock position** is below its **safety stock**, **or**
- its **weeks of cover including stock on the way** ((1Click available + in transit) ÷ average weekly demand) is **less than the lead time** (4.5 weeks), i.e. it will run out before a new shipment could arrive.

**Example (checked).** Item A sells about 20/week (steady). 1Click has 30 available, nothing in transit.
- Cover = 30 ÷ 20 = **1.5 weeks** < 4.5 → **Urgent**, reason "1.5 weeks of cover is less than the 4.5-week lead time".

**Stock already on the way counts (changed 2026-11-11).** Item A with 60 available and **40 in transit**: position 100, cover (60 + 40) ÷ 20 = 5.0 weeks, not below 4.5, and position is above the reorder point of 97, so it is **OK** (suggested still 77, to reach the target of 177). With only **10** in transit the cover is 3.5 weeks, still **Urgent**: a transfer too small to help does not clear the warning. Limit: the page does not know when a transfer lands; if it arrives after the shelf empties there is a short gap that the page shows as OK. The weekly accuracy job (days 1Click sat at 0) is how to spot that.

---

### Tile 2: Reorder

**Question:** "How many items are getting low but are not yet critical?"

**Logic:** status **Reorder** = stock position is below the **reorder point** (but not Urgent).

**Example (checked).** Item A, 20/week, ROP 97. Position 90 (say 90 at 1Click, cover 4.5 weeks, exactly the lead time so not Urgent), safety 7: position 90 < 97 → **Reorder**; suggested = 177 − 90 = **87**.

---

### Tile 3: Overstock

**Question:** "What are we holding too much of?"

**Logic:** weeks of cover **above 12** (`overstock_weeks`). Also: stock at 1Click with **no US sales** in the history window ("stock is idle"). Suggested quantity is forced to **0**: we never send more.

**Example (checked).** Item A, 450 available → cover 22.5 weeks > 12 → **Overstock**, "22.5 weeks of cover is over 12". Item F, 50 on hand, no US sales in 52 weeks → **Overstock**, "No US sales in the history window, stock is idle".

---

### Tile 4: Suggested units

**Question:** "How many units in total would we ship if we followed every suggestion?"

**Logic:** sum of `suggested_qty` over the rows **currently shown** (the filters apply). The row maths is in Part 3.

---

### Tile 5: Suggested value (selling price)

**Question:** "What is that shipment worth?"

**Logic:** Σ (suggested qty × Item Price) using the price list chosen in settings (default *Standard Selling*). The latest price entry wins. It is a **selling-price** value, because `Item.valuation_rate` is a placeholder 1 on almost every item. Items with **no price in that list count as 0**.

**Example.** Item A suggested 147 × $12.50 = $1,837.50; item B suggested 94 × $8 = $752; item C has no price → $0. Tile shows **$2,589.50** (and the tile "Suggested units" shows 147 + 94 + C's units).

---

### Tile 6: Last 1Click sync

**What you see:** "3 hours ago", red if the banner is showing, "never" if it never ran. It tells you how old the stock numbers on the page are.

---

### Section 7: Filters

| Filter | Effect |
|---|---|
| **Item Group** | The group **and its child groups** |
| **Status** | Urgent / Reorder / OK / Overstock / Not stocked yet |
| **Min Weekly Demand** | Hide items whose weighted average weekly demand is below this |

The tile strip follows the filters.

---

### Section 8: The item table

Sorted **Urgent first**, then Reorder, Not stocked yet, OK, Overstock; inside a status, larger suggested quantity first.

| Column | Meaning |
|---|---|
| Item | Item code and name (click for the drill-down) |
| 1Click Available | Stock available at the US warehouse from the latest snapshot |
| In Transit | Units on transfers still on their way (see Part 4) |
| Avg Weekly Demand | The weighted average (Part 3, step 2) |
| Weeks of Cover | 1Click available ÷ average weekly demand (shelf only; the Urgent test adds stock on the way) |
| Reorder Point / Target | Part 3, steps 5 and 6 |
| Suggested Qty | Editable box; ticked rows go into the draft transfer |
| Status | Urgent / Reorder / OK / Overstock / Not stocked yet |
| Confidence | High / Medium / Low |

**Statuses in full (first rule that matches, top to bottom)**

| Order | Condition | Status |
|---|---|---|
| 1 | Never held at 1Click, no stock, and sold in **fewer than 6 of the last 12 weeks** | *hidden* (not worth stocking) |
| 2 | Never held at 1Click, no stock, sold in **6 or more of the last 12 weeks** | **Not stocked yet** |
| 3 | No demand at all | Overstock if there is stock, else OK |
| 4 | Position < safety stock | **Urgent** |
| 5 | Cover incl. stock on the way < lead time | **Urgent** |
| 6 | Position < reorder point | **Reorder** |
| 7 | Cover > 12 weeks | **Overstock** (suggested forced to 0) |
| 8 | Otherwise | **OK** ("Between reorder point and target") |

**Confidence (a separate column)**
- **Low** = fewer than 8 weeks of history. Rows are marked "(uncertain)".
- **Medium** = enough history but sold in fewer than 6 of the last 12 weeks.
- **High** = everything else.

**Example: "Not stocked yet" (checked).** Item E has never been at 1Click. Over the last 12 weeks it sold 3, 0, 4, 2, 5, 3, 4 (after 6 empty weeks; selling in 6 of the last 12 weeks). Status **Not stocked yet**, "Sold in 6 of the last 12 weeks, none at 1Click", suggested **24**. If it had sold in only 1 of the 12 weeks, it would be **hidden**, because demand is not steady.

**Example: stockout of an item we do stock (checked).** Item A (20/week) is at **0** at 1Click, and 1Click has held it before. Status **Urgent**, "Stock position is below safety stock", suggested **177** (the full target). A never-stocked item at 0 is "Not stocked yet" or hidden instead; the rule is "has 1Click ever had it".

---

### Section 9: Item drill-down

Click an item. The dialog shows the calculation line by line (the same numbers as Part 3), then:

- **Weekly US demand** bar chart (labels like "9 Nov").
- **1Click stock history** line chart (Available vs On hand). Needs snapshots on **at least 2 different days**; otherwise "Not enough snapshots yet".
- **Open transfers** table: transfer, Lyfe Order, state, quantity.

The "Why" rows are the best place to learn the logic: every row shows the formula with the real numbers, for example *"Safety stock: Z 1.645 (95% service) × std dev 2 × √4.5 = 7.0"*.

---

### Section 10: Actions

| Button | What it does |
|---|---|
| **Add selected to draft Stock Transfer** | Creates a **draft** *Stock Transfer Order 1Click* with the ticked items and the quantities in the boxes. **Never submitted, no Lyfe Order created.** Someone later presses that document's own "Create Lyfe Order" button. Rows with a quantity of 0 are ignored; an item that is not on this dashboard is refused; the current 1Click available is saved on each line as "stock before". |
| **Export to Excel** | Downloads the table (respecting filters) with more columns than the screen: stock position, demand std dev, lead time, safety stock, pack size, price, suggested value and the "Why" text. |
| **Sync 1Click now** | Reads 1Click stock immediately (read-only call) and reloads. |

---

## Part 3. The calculation, step by step

Applies to **each item**. All weeks are Monday-start weeks.

### Step 1: Weekly demand
Units on **US-destination Lyfe Order lines**, by the order's `order_date` week, **whichever warehouse shipped them** (a US order shipped from the Factory still counts as US demand).
- **Excluded:** cancelled / merged / split orders, stock-transfer orders (replenishment itself is not demand), adjustment lines, lines with no Item.
- **The current, incomplete week is not counted.**
- Weeks are **zero-filled from the item's first sale** (no zeros before the item existed).

### Step 2: Weighted average
Newest **4 weeks count 50%**, weeks **5 to 12 count 30%**, weeks **13 to 26 count 20%**. Each band is its own average, then the bands are combined by weight. A band with no weeks yet (new item) is skipped and the other weights are re-scaled.

**Example 1 (arithmetic).** Newest 4 weeks average 30/week, weeks 5-12 average 20, weeks 13-26 average 10:
30 × 0.5 + 20 × 0.3 + 10 × 0.2 = 15 + 6 + 2 = **23 per week**. A fast-rising item is weighted toward its recent pace.

**Example 2: new item (checked).** Only 6 weeks of sales: 5, 8, 6, 9, 7, 10. The "old" band does not exist and the mid band has just 2 weeks.
Recent band (newest 4: 10, 7, 9, 6 → 8.0) with weight 0.5; mid band (older 2: 8, 5 → 6.5) with weight 0.3. Re-scaled: 62.5% and 37.5% → 8.0 × 0.625 + 6.5 × 0.375 = **7.44/week**. Confidence is **Low** (6 weeks < 8 minimum), so the row is marked "(uncertain)".

### Step 3: Standard deviation
Population standard deviation of the weekly series (how bumpy demand is).

### Step 4: Lead time (weeks)
3 ÷ 6 + 3 + 1 = **4.5** (default).

### Step 5: Safety stock and reorder point

```
Safety stock  = Z × std dev × √(lead time)         (Z = 1.645 at 95%)
Reorder point = average × lead time + safety stock
```

### Step 6: Target

```
Target = reorder point + average × coverage weeks (4)
```

### Step 7: Stock position

```
Stock position = 1Click available + in transit + in production
```
*In production* is always 0 today (nothing tags a Lyfe PO as US stock).

### Step 8: Suggested quantity

```
Suggested = max(0, target − stock position)
```
Rounded **up** to a multiple of the item's `min_order_qty` if set (there is no pack-size field), otherwise to a whole unit. Forced to 0 for Overstock.

### Step 9: Weeks of cover
`1Click available ÷ average weekly demand` (blank when there is no demand).

---

### Worked example A: steady item (checked)

Weekly demand alternates 18, 22 over 26 weeks → average **20.0**, std dev **2.0**.

| Quantity | Calculation | Result |
|---|---|---|
| Lead time | 3 ÷ 6 + 3 + 1 | 4.5 weeks |
| Safety stock | 1.645 × 2.0 × √4.5 | **7.0** |
| Reorder point | 20 × 4.5 + 7 | **97** |
| Target | 97 + 20 × 4 | **177** |

Now vary the stock:

| 1Click available | In transit | Position | Cover (weeks) | Status | Suggested |
|---|---|---|---|---|---|
| 0 (was stocked before) | 0 | 0 | 0 | Urgent | **177** |
| 30 | 0 | 30 | 1.5 | Urgent (cover < 4.5) | **147** |
| 60 | 10 | 70 | 3.0 / 3.5 | Urgent (3.5 < 4.5) | **107** |
| 60 | 40 | 100 | 3.0 (shelf) / 5.0 (with transit) | OK (suggested still tops up to target) | **77** |
| 200 | 0 | 200 | 10.0 | OK | 0 |
| 450 | 0 | 450 | 22.5 | Overstock | 0 |

**With a minimum order quantity of 25** (checked): item with 30 available → 147 rounded up to **150**.

### Worked example B: spiky item (checked)

Demand comes in lumps: 60, 50 and 10 units in single weeks, zero in most weeks (24 weeks of history).

| Quantity | Result |
|---|---|
| Weighted average | **7.38/week** (recent weeks were quiet: 2.5) |
| Std dev | **20.41**, bigger than the average |
| Safety stock | 1.645 × 20.41 × √4.5 = **71.2** |
| Reorder point | 7.38 × 4.5 + 71.2 = **104.4** |
| Target | 104.4 + 7.38 × 4 = **133.9** |

With 40 available: position 40 < safety 71.2 → **Urgent**, "Stock position is below safety stock", suggested **94**, confidence **Medium** (sold in fewer than 6 of the last 12 weeks).

**Lesson:** for lumpy items the safety stock is large, so the page asks for stock even though the average looks tiny. That is intentional: one bulk order would wipe out a shelf sized for the average.

---

## Part 4. Stock in transit

**What counts:** quantity on *Stock Transfer Order 1Click* lines whose **Lyfe Order** (customer "Lyfe Hardware", via US warehouse) is **not** Cancelled / Completed / US Warehouse Delivered **and** has `delivered_to_us_warehouse = 0`.

**What does not count**
- **Drafts with no Lyfe Order yet.** They show in the drill-down as "Draft (not sent yet, not counted)".
- Transfers whose Lyfe Order has been marked delivered. From then on 1Click's own stock is expected to show the units, so counting them again would double-count. (A short double-count is possible if the delivered flag is set before 1Click updates its stock.)

**Example.** Monday you create a draft transfer for 100 units of item A. Stock position is unchanged (draft not counted). Tuesday someone presses "Create Lyfe Order" → 100 are now **in transit**, so position rises by 100 and the suggestion drops. Three weeks later the order is marked *US Warehouse Delivered* → the 100 leave "in transit"; by then 1Click's snapshot shows them in *Available*.

**Warehouse stays Factory:** a Lyfe Order's warehouse never flips to US after delivery; the `delivered_to_us_warehouse` flag is what the page uses.

---

## Part 5. Where the data comes from and when it refreshes

| Data | Source | Refresh |
|---|---|---|
| US demand | Lyfe Orders (`ship_to_country` = US) + order lines | Live when the page loads, row results cached 60 seconds |
| 1Click stock | *OneClick Stock Snapshot* saved from 1Click's inventory call (`/api/v1/inventory.cfm`, read-only) | Hourly job; runs every 4 hours, or press **Sync 1Click now** |
| In transit | Stock Transfer Order 1Click + Lyfe Order | Live |
| Prices | Item Price, price list from settings | Live |
| Weekly accuracy | *US Replenishment Suggestion* (one row per item per week) | Monday 02:30 |

### Weekly accuracy job (Monday 02:30)
1. **Scores last week:** for each stored suggestion, records actual units sold, absolute error, and **days 1Click sat at 0**.
2. **Stores this week's suggestions** (idempotent: running twice does nothing). Skipped if there is no 1Click snapshot yet.

This gives you evidence over time: did "Urgent" items really sell out, and did "Overstock" items really sit?

---

## Part 5b. Settings, roles and failure handling

### Roles and permissions
| Action | Who |
|---|---|
| Open the page and every number on it | System Manager, Super Admin, US Replenishment Manager (assign in User > Roles) |
| Create the draft Stock Transfer Order 1Click | Also needs create permission on that document: Factory, Customer Service, System Manager, Super Admin |
| Edit US Replenishment Settings | System Manager, Super Admin (the sync-status fields on it are read-only) |
| Stock Forecast page tile | The same three roles; Factory sees the page but not this tile |

### What each stored table holds
| Table | Holds | Kept for |
|---|---|---|
| *OneClick Stock Snapshot* | Per item: available, on hand, reserved (= on hand − available), fetched at, warehouse id | 180 days (setting) |
| *US Replenishment Suggestion* | One row per item per week: status, confidence, demand, std dev, lead time, safety, reorder point, target, available, in transit, position, suggested, and after scoring: actual units, error, days at 0 | Kept |
| *US Replenishment Settings* | All the numbers in Part 1, plus sync status (last attempt, status, last success, items saved, last error) | Single document |

### Failure handling
- The sync never raises. A failure is written to the Error Log and to the settings status (shown in the banner).
- An empty response or a paged response is treated as a failure so the last good snapshot is kept.
- A wrong warehouse id is rejected by 1Click (HTTP 406) and the sync fails; an all-zero answer is saved as real stock.
- If the weekly accuracy job fails, it logs "[us_replenishment][accuracy.run]: weekly tracking failed" and the dashboard is unaffected.
- A saved 0 in the settings no longer breaks the page: blank or 0 number fields fall back to the doctype default, and working days per week can never be below 1.

### Caching
The rows are cached for **60 seconds** and the cache is cleared by every sync, so a manual sync shows immediately.

### Not covered by the dashboard (by design)
- Customer orders via the US warehouse are demand, never stock (Part 4).
- Nothing is submitted or shipped by the dashboard; there is no Bulk Transfer Batch (that is for customer deliveries).
- Writes are not available through AI / MCP tools (see Part 8): no sync, no draft transfer, no Excel export. The page is not on the founder summary.
- Automated tests cover the maths only (`test_calc`: 13 cases); the page, the sync and the draft action are checked by hand.

---

## Part 6. Common questions

**Q: Why is an item Urgent when I have stock on the way?**
Because what is on the way is not enough. Urgent uses available + in transit; 60 available + 10 in transit is 3.5 weeks, below the 4.5-week lead time. With 40 in transit it is 5.0 weeks and the item is OK.

**Q: Why does an item with almost no sales have a big suggestion?**
Lumpy demand means a large standard deviation, so safety stock is large (worked example B).

**Q: Why is an item missing?**
Custom item, Make to Order / Do Not Stock, tube or acrylic, disabled, **or** never held at 1Click and sold in fewer than 6 of the last 12 weeks. Also lines with no linked Item are not counted (note under the tiles).

**Q: Why is "Suggested value" lower than expected?**
Items with no price in the chosen price list count as 0.

**Q: Where did my draft go?**
It opens as a draft *Stock Transfer Order 1Click*. Review, edit, then press its own "Create Lyfe Order" button. Nothing is sent by the dashboard itself.

**Q: Does an unfinished week count?**
No. Only complete Monday-to-Sunday weeks.

**Q: What if the numbers look old?**
Check the *Last 1Click sync* tile and the banner. Use **Sync 1Click now**.

---

## Part 7. Known open points
- Whether 1Click's `available` already excludes reserved stock, and whether its "get all inventory" call pages, are unconfirmed (`docs/oneclick_open_questions.md`).
- "In production" is always 0 (no way to tag a Lyfe PO as US stock yet).
- No pack-size field exists; `min_order_qty` is used as the rounding multiple.
- Price filtering ignores `valid_from` / `valid_upto` on Item Price.

---

## Part 8. Getting the same information through MCP (AI tools)

Everything you can read on this page is available to an AI assistant (Claude) through three read-only tools. They return exactly what the page returns, and they check the same roles: System Manager, Super Admin, US Replenishment Manager.

| What you want | Tool | Arguments |
|---|---|---|
| The whole dashboard: tiles, sync status, unmapped lines, every item row | `get_us_replenishment_dashboard` | `item_group`, `status` (Urgent / Reorder / OK / Overstock / Not stocked yet), `min_demand` |
| One item's drill-down: calculation, weekly demand, 1Click stock history, open transfers, lead-time parts | `get_us_replenishment_item_detail` | `item_code` |
| The small US tile on the Stock Forecast page | `get_founder_tile_us_replenishment` | none |

**Example questions and how they are answered**
- "Which items are Urgent?" → `get_us_replenishment_dashboard(status="Urgent")`.
- "Why does item A need 147 units?" → `get_us_replenishment_item_detail("A")`, read `row` (average demand, safety stock, reorder point, target, position, suggested) and `transfers`.
- "How old is the 1Click stock data?" → the `sync` block of the dashboard (`last_success_at`, `warning`).
- "What is the value of everything we would ship?" → `tiles.suggested_value`.

**Not available through MCP, on purpose (they change data or call 1Click):** Sync 1Click now, Add selected to draft Stock Transfer, Export to Excel (the same rows are in `get_us_replenishment_dashboard`).

