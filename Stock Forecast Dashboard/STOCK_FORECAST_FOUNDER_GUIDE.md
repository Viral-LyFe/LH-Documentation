# Stock Forecast Dashboard: User Guide

*Last updated: 9 Nov 2026*

> 📸 **Screenshot 1: Stock Forecast page overview**
> - **Where to take it:** Stock Forecast page, **Act now** tab, no filters, scrolled to the top.
> - **What must be visible:** the four summary cards, the Category and Item Code boxes, the three tabs, and the first two tiles with real data.
> - **Highlight:** draw an arrow to the three tabs and a circle around the four summary cards.
> - **File name:** `images/stock-forecast/01-overview.png`
>
> `![Screenshot 1: Stock Forecast page overview](images/stock-forecast/01-overview.png)`

---

## 1. What is this dashboard?

The Stock Forecast dashboard tells you what will run out, when it will run out, and how much to buy or send. It also shows which open orders are held up by missing material.

Everything on it comes from records the team already keeps: stock on the shelf, material issued to the factory, and open orders. Nobody types numbers into it.

**Who uses it:**
- The Founder checks the big picture: what is at risk and what it may cost.
- The Operations and Factory team decides what to buy and what to build first.
- The CS team checks why an order may be late because of missing material.

**What it does not do:** it does not know about purchase orders, suppliers or delivery promises. It tells you how long stock will last and how much to buy. You decide when to place the order.

---

## 2. How to open it

- Type **Stock Forecast** in the search bar at the top of the system, or go to `/app/stock-forecast`.
- The Founder Dashboard also has a small **Stock** banner. Click "Open Stock Forecast" on it.

**Who can open it:** people with the role **System Manager**, **Super Admin** or **Factory**. A Factory user sees this page but not the Founder Dashboard's money pages (revenue and profit).

**One tile has its own access.** The **US Warehouse Replenishment** tile is shown only to System Manager, Super Admin and people who have the **US Replenishment Manager** role. Ask an administrator if you need it.

---

## 3. Quick start: read the dashboard in 90 seconds

**The page layout**
1. **Four summary cards** at the top. Click a card to jump to the related tile.
2. **Category and Item Code boxes.** They narrow the whole page. Leave them empty to see everything.
3. **Three tabs.** The page opens on the tab you used last.
   - **Act now:** what to buy, what is about to run out, what to send to the US, and which materials are short for active orders.
   - **Demand & forecast:** what is selling more or less, what we expect to need, and how accurate the forecast is.
   - **Materials:** tube, acrylic rods and which orders can be built today.

A tab loads the first time you open it, so that first click takes a moment. If one tile fails, it shows **Retry** and the others keep working.

**The five things to look at first**
1. **The four cards.** Anything red, or "Data issues" not saying *None*, is where to start.
2. **Restock List** (Act now). This is the buying list. Read the **Why** column.
3. **Stockout Risk** (Act now). Critical and High rows run out first.
4. **Material Not In Stock** tiles (Act now). They show which materials active orders still need.
5. **Data issues card.** The numbers are only as good as the records behind them. Aim for *None*.

**Once a week** (for example Monday): open **Demand & forecast** to see what is rising and how close the forecast was.

**Colours used on the page**
- **Red:** bad. For example, Critical items, or an item that will run short.
- **Amber:** a warning. Look at it soon.
- **Grey or neutral:** nothing special.

Colours are never the only signal. Risk badges also carry a symbol.

---

## 4. Tile-by-tile guide

Each tile has the same five parts: **What it shows**, **How to read it**, **What to do**, **Example** and a screenshot.

Tiles are grouped by tab:

| Tab | Tiles |
|---|---|
| Top of page | Summary cards |
| Act now | Restock List, Stockout Risk, US Warehouse Replenishment, Material Not In Stock - Total Required, Active Orders - Material Not In Stock |
| Demand & forecast | Demand Trends, Demand Forecast, Base-level Forecast, Category-wise Stock Forecast, Material Usage, Forecast vs Actual |
| Materials | Tube Stock, Acrylic Rod Stock, Buildable Orders |
| Header chip | Stock Data Quality |

---

### 4.1 Summary cards (top of the page)

**What it shows**
Four clickable cards. Each one points to a tile.

| Card | What it shows |
|---|---|
| **Orders short now** | How many orders are Partial or Short in Buildable Orders, and their dollar value. The small line says how many are Ready. |
| **Items to reorder** | How many rows are in the Restock List. The small line says how many are Critical and High. |
| **Shortest cover** | The one item with the fewest days of stock, and its days. It says "No usage data" if nothing has been used. |
| **Data issues** | How many data problems exist, or "None". Click it to open the Stock Data Quality checklist. Only people allowed to see data quality get this card. |

A small refresh arrow under the cards reloads only the cards.

**How to read it**
- **Orders short now** is red when any order is short. Otherwise it is grey.
- **Items to reorder** is red when any Critical item exists, amber when the worst items are High, and grey otherwise.
- **Shortest cover** is always grey.
- **Data issues** is amber when there is any issue, and grey when it says *None*.

**What to do**
Click the card that worries you. The page opens the right tab and jumps to that tile.

**Example**
"Orders short now: 6 · $4,200. 2 ready." Click it. Buildable Orders opens and shows which six orders lack material.

> 📸 **Screenshot 2: Summary cards**
> - **Where to take it:** top of the Stock Forecast page, no filters.
> - **What must be visible:** all four cards with numbers, and their colours.
> - **Highlight:** circle the "Data issues" card.
> - **File name:** `images/stock-forecast/02-summary-cards.png`
>
> `![Screenshot 2: Summary cards](images/stock-forecast/02-summary-cards.png)`

---

### 4.2 Restock List (Act now)

**What it shows**
What we need to buy, and which items have enough stock for the next N days. It is one list in one place. It includes ordinary items, tube pipes (in rods) and acrylic rods (in inches). Most urgent comes first.

**How to read it**
- The line at the top says how many items have **not enough** stock (in red) and how many have **enough** (in green).
- Each row shows: Item, Type, Risk, On Hand, Available, Days Left, Order Qty, Unit, **Why** and **Last Received**.
- **Why** explains the row in plain words. Example: *"Uses 1.5/day, so needs 21 for the next 14 days. 32 on hand, 20 already promised to open orders: short 9, order 10 (minimum order 10)."*
- **Last Received** is the date the item last came into stock. Acrylic rows have none.
- The list also covers a **safety buffer** (extra stock for a busy week). So it can list more items than the Demand Forecast tile says "will run short".
- The **Risk** column uses the same levels as Stockout Risk: Critical is red, High and Medium are amber.

**What to do**
1. Read **Order Qty** and **Why**.
2. Check **Last Received** and ask the buyer whether an order is already on its way. The page cannot see purchase orders. It sees stock only when it arrives.
3. Export to CSV and send it to the buyer.

**Choose the number of days:** use the funnel icon on the tile and type days, for example 14. If you leave it empty, each item uses its own target cover (30 days unless the item sets its own). Click **Show the items with enough stock** to see the second list (Item, Type, Available and Days covered).

**Example**
An item uses 1.5 a day. For 14 days it needs 21. We hold 32, but 20 are promised to open orders, so only 12 are free. It is short by 9, so it goes on the list.

> 📸 **Screenshot 3: Restock List**
> - **Where to take it:** Act now tab, Restock List tile, days left empty.
> - **What must be visible:** the summary line, at least five rows, and the Why and Last Received columns.
> - **Highlight:** circle one **Why** sentence.
> - **File name:** `images/stock-forecast/03-restock-list.png`
>
> `![Screenshot 3: Restock List](images/stock-forecast/03-restock-list.png)`

> 📸 **Screenshot 4: Restock List, cover for the next N days**
> - **Where to take it:** Restock List tile, funnel icon open, 14 typed in the days box.
> - **What must be visible:** the filter box with 14, and the updated summary line.
> - **Highlight:** arrow to the days box.
> - **File name:** `images/stock-forecast/04-restock-list-days.png`
>
> `![Screenshot 4: Restock List days filter](images/stock-forecast/04-restock-list-days.png)`

---

### 4.3 Stockout Risk (Act now)

**What it shows**
Every item that was used recently and has **under 30 days** of stock left. Items with 30 or more days are not listed.

**How to read it**
- Each row shows the item code, "used N in 30d", stock available, the suggested order quantity (only when above zero), **days of cover** and "exposes $X" or "no open orders".
- A product sold in several finishes is one row marked "×N finishes".
- The **risk badge** says how soon the item runs out. Critical is red (▲), High and Medium are amber (●), Low is grey (◆). The symbol means colour is never the only signal.

| Level | Days of cover (default) |
|---|---|
| Critical (red) | under 7 |
| High | 7 to under 15 |
| Medium | 15 to under 30 |
| Low | 30 or more (not listed) |

- If **Available is below zero**, the item is always Critical. Open orders already need more than we have.
- A **trend arrow** ("up +40%") means recent use is 40% higher than the 90-day average. It shows only when the change is 25% or more, and only when the item was issued at least 3 times in 90 days. An up arrow is amber. A down arrow is grey.
- **⚑ unexpected order** appears instead of the arrow when one big order makes up at least half of the item's use. It is not treated as normal demand.
- **Days of cover** uses the forecast daily rate (see "Daily use" in the glossary). The "used N in 30d" text is the plain count of units used. So the two can differ a little.
- The subtitle says **Live** (calculated now) or **Snapshot** with a date (last night's saved copy).
- Rows are sorted: items where open orders already exceed stock first, then Critical, High, Medium, then more dollars exposed, then fewer days of cover.

**What to do**
1. Work from Critical down.
2. Click **View forecast detail** for the full list, with a search box and a risk filter.
3. Click a row to see more: daily use, committed, confidence, safety stock, reorder point, **Where used** and links to the Item and its stock ledger.
4. Use **Download restock list (CSV)** to export every row.

**Left out on purpose:** items with no use in the window, tube and acrylic rods (they have their own tiles) and items set to "Make to Order" or "Do Not Stock".

**Filters:** Category (includes its sub-groups) and Trailing days.

**Example**
CMBR-200-PB: 60 on hand, forecast daily use 3. Cover = 60 ÷ 3 = **20 days**. That is Medium. The stock-out date is today plus 20 days.

> 📸 **Screenshot 5: Stockout Risk**
> - **Where to take it:** Act now tab, Stockout Risk tile.
> - **What must be visible:** at least one Critical and one High row, a trend arrow, the footer counts and the Live or Snapshot label.
> - **Highlight:** circle the risk badge and the "exposes $X" text. Blur nothing (no customer data here).
> - **File name:** `images/stock-forecast/05-stockout-risk.png`
>
> `![Screenshot 5: Stockout Risk](images/stock-forecast/05-stockout-risk.png)`

> 📸 **Screenshot 6: Stockout Risk, item detail**
> - **Where to take it:** click **View forecast detail**, then click one row to expand it.
> - **What must be visible:** the search box, risk filter, the expanded row with daily use, committed, confidence, Where used and the Go to links.
> - **Highlight:** arrow to **Where used**.
> - **File name:** `images/stock-forecast/06-stockout-risk-detail.png`
>
> `![Screenshot 6: Stockout Risk detail](images/stock-forecast/06-stockout-risk-detail.png)`

---

### 4.4 US Warehouse Replenishment (Act now)

**What it shows**
Which items we should send to the 1Click warehouse in the US, and how many. US customers get items faster when we hold them there. The tile looks at what the US has been buying, what 1Click holds now and what is already on its way from India.

**How to read it**
- The top line shows how many items are urgent, how many to reorder, how many are overstocked, how many units we suggest sending (and roughly what that is worth at selling price) and when 1Click stock was last read.
- The table shows the first 10 items that need action. For each: stock at 1Click, stock on its way, units sold per week, weeks the stock will last and the suggested quantity.
- **(uncertain)** means the item has too little sales history to trust the number.
- A **warning line** means the last read of 1Click stock failed or is old. The numbers then come from the last good read, shown with its time.
- Only standard items appear. Custom (made-to-order) items never do.

**What to do**
1. Click **Open the full US Warehouse Replenishment page**.
2. Follow the steps below.

**Steps on the full page (`/app/us-replenishment`)**
1. **Check the top of the page.** A yellow banner means the last read of 1Click stock failed or is old. You can press **Sync 1Click now** to read it again.
2. **Work from the top.** Rows are sorted Urgent first. Use the Item Group, Status and *Min Weekly Demand* boxes to narrow the list.

| Status | Meaning |
|---|---|
| **Urgent** | We will run out before a new shipment could arrive. |
| **Reorder** | Below the level where we should already be re-stocking. |
| **OK** | Enough stock. |
| **Overstock** | More stock than we need. Nothing is suggested. |
| **Not stocked yet** | Sells steadily in the US, but 1Click has none. A new candidate. |

3. **See why.** Click an item code. The window shows the numbers step by step, the weekly demand chart, the 1Click stock history and open transfers. Stock already on its way is counted, so it is not suggested twice.
4. **Choose quantities.** Change the Suggested Qty in a row if you want a different number. Tick the rows you want.
5. Press **Add selected to draft Stock Transfer** and confirm. You need permission to create a Stock Transfer Order 1Click (Factory, Customer Service, System Manager or Super Admin). This creates a **draft**. **Nothing is sent yet.**
6. Open the draft and review it. When you are ready, press **Create Lyfe Order** on it. That starts the shipment to the US warehouse through the factory. Only Factory, System Manager and Super Admin can press that button.

**Export:** the full page has **Export to Excel**.

**Example**
An item sells 10 a week. The shipment takes about 4.5 weeks (3 working days at the factory, 3 weeks of transit, 1 week for customs and receiving). So the reorder point is 45. We want 4 more weeks of cover, so the target is 85. If 10 are at 1Click and none are on the way, the suggestion is 75. If 75 are already on the way, the suggestion is 0.

> 📸 **Screenshot 7: US Warehouse Replenishment tile**
> - **Where to take it:** Act now tab, US Warehouse Replenishment tile (needs the US Replenishment Manager role).
> - **What must be visible:** the counts line, the sync time, the table rows and the "Open the full page" link.
> - **Highlight:** circle the last 1Click sync time.
> - **File name:** `images/stock-forecast/07-us-tile.png`
>
> `![Screenshot 7: US tile](images/stock-forecast/07-us-tile.png)`

> 📸 **Screenshot 8: US Warehouse Replenishment full page**
> - **Where to take it:** `/app/us-replenishment`, no filters.
> - **What must be visible:** the tiles on top, the filters, the table with tick boxes, quantity boxes and status pills, and the three buttons.
> - **Highlight:** arrow to the Suggested Qty box and to **Add selected to draft Stock Transfer**.
> - **File name:** `images/stock-forecast/08-us-page.png`
>
> `![Screenshot 8: US page](images/stock-forecast/08-us-page.png)`

> 📸 **Screenshot 9: US page, item detail ("why")**
> - **Where to take it:** click an item code on the full page.
> - **What must be visible:** the step-by-step numbers, the weekly demand chart and the stock history chart.
> - **Highlight:** circle the "Suggested qty" line.
> - **File name:** `images/stock-forecast/09-us-item-why.png`
>
> `![Screenshot 9: US item detail](images/stock-forecast/09-us-item-why.png)`

> 📸 **Screenshot 10: Draft Stock Transfer Order**
> - **Where to take it:** the draft opened after pressing **Add selected to draft Stock Transfer**.
> - **What must be visible:** the item rows with quantities, status Draft and the **Create Lyfe Order** button.
> - **Highlight:** arrow to **Create Lyfe Order**.
> - **File name:** `images/stock-forecast/10-draft-transfer.png`
>
> `![Screenshot 10: Draft transfer](images/stock-forecast/10-draft-transfer.png)`

---

### 4.5 Material Not In Stock - Total Required (Act now)

**What it shows**
For everything we still have to build, which materials are short in total. It adds up all active orders first, then compares with stock.

**How to read it**
- Columns: Item Code (a link, with a tick box), Required Qty, In Stock, Missing Qty (Required minus In Stock).
- Only items where Required is above In Stock are listed. Numbers are rounded up. Sorted by item code.
- "Active orders" means orders not yet dispatched, from New to Awaiting India Components. On Hold, Shipped, Completed and later stages are out.
- If nothing is missing, it says: "All material for active orders is in stock."

**What to do**
- Buy the missing quantities.
- **Tick** an item to see only the orders that need it in the next tile. Only one tick at a time. Un-tick to clear.

**Example**
Item Y: 10 in stock. Two orders each need 6. Total required = 12. Missing = **2**.

> 📸 **Screenshot 11: Material Not In Stock - Total Required**
> - **Where to take it:** Act now tab, this tile.
> - **What must be visible:** the columns, at least four rows and a tick box.
> - **Highlight:** circle the Missing Qty column.
> - **File name:** `images/stock-forecast/11-not-in-stock-total.png`
>
> `![Screenshot 11: Not in stock total](images/stock-forecast/11-not-in-stock-total.png)`

---

### 4.6 Active Orders - Material Not In Stock (Act now)

**What it shows**
Which single orders cannot be completed from current stock. One row per order and component where that order alone needs more than we have.

**How to read it**
- Columns: Internal Order ID (a link), Order Type, Item Code (a link), Required Qty, In Stock, Missing Qty, Order Date, Order Value. Sorted by order date.
- **Stock is not shared between orders here.** Each order is compared with the full stock on its own.

**What to do**
Use the tick box in the Total tile to see which orders are behind a shortage. Tell the CS team if ship dates will slip.

**Example**
Item Y: 10 in stock. Two orders need 6 each. In **this** tile neither order appears, because 6 is not more than 10. In the **Total** tile, Y appears with 2 missing. In **Buildable Orders**, the older order is Ready and the second is Short.

> 📸 **Screenshot 12: Active Orders - Material Not In Stock**
> - **Where to take it:** Act now tab, this tile, after ticking one item in the Total tile.
> - **What must be visible:** the filtered rows and the columns.
> - **Highlight:** blur customer names if any are shown. Circle the Missing Qty column.
> - **File name:** `images/stock-forecast/12-active-orders-not-in-stock.png`
>
> `![Screenshot 12: Active orders not in stock](images/stock-forecast/12-active-orders-not-in-stock.png)`

---

### 4.7 Demand Trends (Demand & forecast)

**What it shows**
Which categories we are using more or less of, and which items are suddenly in demand. It compares the **last N days** with the **N days before**, counting stock issued to orders.

**How to read it**
- **Categories:** a line chart shows each big category's use week by week. A table shows use now, use before and the change.
  - **↑** (amber) using more (over 10% up).
  - **↓** using less (over 10% down).
  - **→** about the same.
- **Rising items:** the items whose use went up the most, each with a small trend line and an **Orders** count (how many different orders it was issued on). The list shows items with the most orders first.
- An item needs at least 3 separate issues and 25% more use than before. One big one-off order does not count as a trend.
- Finishes of one product count as one row. Click the finish name to see all finishes.
- If the tile says the history is incomplete, the "before" period goes back before our records start (May 2026), so rises look bigger than they are.
- **Tube items (N):** tube is kept in its own section at the bottom. Pipes are in feet and cut lengths are as issued.
- The category table sits in its own collapsed section ("Category table"). Click it to open.

**What to do**
Spot demand picking up early and check that rising items have enough stock. See which categories are slowing down before you over-stock them.

**Choose the days and how many items:** use the funnel icon. Defaults: 30 days, top 10.

**Example**
UHRB-150: 50 used in the last 30 days, against 6 in the 30 days before. That is ↑ +733%, with 4 days of stock left. Check it now.

> 📸 **Screenshot 13: Demand Trends**
> - **Where to take it:** Demand & forecast tab, Demand Trends tile, 30 days, top 10.
> - **What must be visible:** the category chart, the rising-items table with the Orders column and the trend lines.
> - **Highlight:** circle the Orders column and one ↑ arrow.
> - **File name:** `images/stock-forecast/13-demand-trends.png`
>
> `![Screenshot 13: Demand Trends](images/stock-forecast/13-demand-trends.png)`

---

### 4.8 Demand Forecast (Demand & forecast)

**What it shows**
How much each product will need over the next N days, and whether stock will last. Default: 30 days, top 10, biggest need first.

**How to read it**
- **Per day:** average use per day (the forecast uses a method called TSB, explained in the glossary).
- **Forecast:** per day × the number of days.
- **On open orders:** units already promised to orders not yet issued.
- **Expected:** forecast plus open orders.
- **Left after N days:** on hand minus expected. A negative number in **red** means the product will run short.
- The small line in each row shows the last 12 weeks of real use (grey) and the forecast continuing as a dashed line. The dashed part is flat: the forecast is a steady rate and does not guess at peaks.
- The line at the top sums it up. Example: "Expected 2,043 units across 103 products … 82 products will run short."
- **Tube items** are in their own section at the bottom. Cut lengths are cut from pipe when needed, so they usually show 0 on hand and appear as short. Check the pipe rows and the Tube Stock tile before ordering.

**What to do**
Use it to plan ahead for the next 30 or 60 days. Items that run short are ordered by the Restock List.

**Example (CMBR-200, next 30 days)**
- Per day: 7.7.
- Forecast: 7.7 × 30 = 231.
- On open orders: 16.
- Expected: 231 + 16 = 247.
- Left after 30 days: 51 on hand − 247 = **−196** (red). It will run short.

> 📸 **Screenshot 14: Demand Forecast**
> - **Where to take it:** Demand & forecast tab, Demand Forecast tile, 30 days.
> - **What must be visible:** the summary line, rows with the grey and dashed lines, and at least one red "left after" number.
> - **Highlight:** circle a red negative number.
> - **File name:** `images/stock-forecast/14-demand-forecast.png`
>
> `![Screenshot 14: Demand Forecast](images/stock-forecast/14-demand-forecast.png)`

---

### 4.9 Base-level Forecast (Demand & forecast)

**What it shows**
The forecast for a base product with all its finishes added together. Many parts are cut and finished to order. For example, CMBR-200 Matte is one base product that later becomes several finishes. Their combined demand is smoother than each finish alone.

**How to read it**
- Each row is one base product and base state with **2 or more finishes** and use in the last 30 days.
- Columns: base product, state, finishes, risk, on hand, available (on hand minus open orders), forecast per day, days left, safety stock, reorder point and order quantity.
- Stock is the finishes' stock added together. Safety stock is worked out on the combined use, so it is smaller than keeping safety stock for each finish.
- Most urgent comes first (risk, then biggest use). Tube and acrylic are not included.

**What to do**
Plan unfinished (matte or polished) stock for the base product, not for each finish.

**Example**
Across all groups on 6 Nov 2026, combined safety stock was 773 units. Keeping safety stock per finish would have been 958 units.

**Choose how many rows:** use the funnel icon (default 25).

> 📸 **Screenshot 15: Base-level Forecast**
> - **Where to take it:** Demand & forecast tab, Base-level Forecast tile.
> - **What must be visible:** rows with Finishes, Days left, Safety stock and Order qty.
> - **Highlight:** circle the Finishes column.
> - **File name:** `images/stock-forecast/15-base-forecast.png`
>
> `![Screenshot 15: Base-level Forecast](images/stock-forecast/15-base-forecast.png)`

---

### 4.10 Category-wise Stock Forecast (Demand & forecast)

**What it shows**
One bar per top-level category, split into Critical, High, Medium and Low item counts. It covers every item with use in the last 30 days.

**How to read it**
- A longer red part means more Critical items in that category.
- Use the toggle to switch the bars to **Revenue exposed** (dollars). The other figure is printed at the end of each row.
- Sorted by most Critical first, then High, then dollars, then name. The first 6 show. Click "show all" for the rest.
- There is no "average days" per group on purpose, so one stocked-out item cannot hide behind a healthy average.
- The footer counts **dead** stock (nothing used in 90 days) and **excess** stock (more than 3 times the target cover).

**What to do**
1. Click a bar (or a coloured part) to see its sub-groups.
2. Click a sub-group to open its item list (search, risk filter, details, CSV, plus Revenue exposed and Orders columns).
3. Click the pinned row **All critical items** to list every Critical item.
4. Two extra rows, **Tube (feet)** and **Acrylic rod (inches)**, open their own details.

**Example**
"Railings" shows 12 Critical items. Click the bar, then a sub-group, and you see which items need buying.

> 📸 **Screenshot 16: Category-wise Stock Forecast**
> - **Where to take it:** Demand & forecast tab, Category tile.
> - **What must be visible:** at least five bars with the four colours and the toggle.
> - **Highlight:** circle the toggle and one red segment.
> - **File name:** `images/stock-forecast/16-category-forecast.png`
>
> `![Screenshot 16: Category forecast](images/stock-forecast/16-category-forecast.png)`

> 📸 **Screenshot 17: Category drill-down**
> - **Where to take it:** click one bar, then one sub-group.
> - **What must be visible:** the sub-group list or item list with the search box and risk filter.
> - **Highlight:** arrow to the Back link.
> - **File name:** `images/stock-forecast/17-category-drilldown.png`
>
> `![Screenshot 17: Category drill-down](images/stock-forecast/17-category-drilldown.png)`

---

### 4.11 Material Usage (Demand & forecast)

**What it shows**
The items most issued to the factory for orders over the window (default 30 days), most used first. It is labelled "Most-Issued", not "Most-Used Material".

**How to read it**
- It mixes raw materials (feet, sticks) and finished pieces, so units differ per row.
- Numbers are **net**: issued minus returned. Only items above zero are shown.

**What to do**
Use it to see where the factory's material is really going. Check that the biggest items have enough stock.

**Filters:** Category and Trailing days. The full-list popup always uses 30 days.

**Example**
Brass tube: 438 ft issued in 30 days, 10 ft returned. The tile shows the net amount.

> 📸 **Screenshot 18: Material Usage**
> - **Where to take it:** Demand & forecast tab, Material Usage tile.
> - **What must be visible:** the scroll box with at least eight rows.
> - **Highlight:** circle the top row.
> - **File name:** `images/stock-forecast/18-material-usage.png`
>
> `![Screenshot 18: Material Usage](images/stock-forecast/18-material-usage.png)`

---

### 4.12 Forecast vs Actual (Demand & forecast)

**What it shows**
How close our forecast was to what we really used. For each of the last N complete weeks (default 12), it compares what the forecast said at the start of that week with what was issued.

**How to read it**
Read two numbers.
- **In total:** for example "forecast 5,075, used 5,407, so the forecast was 6.1% too low". This is close.
- **Product by product, week by week:** for example "off by 93.8%". A product's use jumps around from week to week, so single weeks are hard to predict. This is why safety stock matters more than the formula.
- The chart shows each week's real use (bars) against the forecast (line).
- **Biggest misses** lists the products the forecast got most wrong, marked "over" (forecast too high) or "under" (too low).
- This tile rebuilds the check from our records, so it works from day one. History starts in May 2026 and each week needs the 30 days before it, so it looks back about 19 weeks for now. It says so when that limit applies.
- **Tube items** are in their own section at the bottom.

**What to do**
Use it to judge how much to trust the forecast. A high "off by" figure means treat the forecast as a rough guide.

**Example**
Last 12 weeks: forecast 5,075 against 5,407 used. The total is 6.1% low, but product by product it is 93.8% off.

> 📸 **Screenshot 19: Forecast vs Actual**
> - **Where to take it:** Demand & forecast tab, Forecast vs Actual tile, 12 weeks.
> - **What must be visible:** the plain-words summary, the bar and line chart and the Biggest misses list.
> - **Highlight:** circle the "In total" and "Product by product" sentences.
> - **File name:** `images/stock-forecast/19-forecast-vs-actual.png`
>
> `![Screenshot 19: Forecast vs Actual](images/stock-forecast/19-forecast-vs-actual.png)`

---

### 4.13 Tube Stock, in feet (Materials)

**What it shows**
One row per tube pipe (material and diameter) that was used in the window. Tube is kept as long rods and cut to order, so it is counted in **feet**.

**How to read it**

| Column | Meaning |
|---|---|
| On hand (ft) | Feet of rod in stock, plus pre-cut pieces (pieces × their length). |
| Rods | Feet on hand ÷ 100. Rods are assumed to be 100 ft long. |
| Used / per day | Feet used in the window (net of returns), and per day. |
| Days left, risk | Cover and risk level (same 7 / 15 / 30 scale). |
| Keep | Daily feet × target cover (default 30 days). |
| Order ft / rods | Feet short of "keep", and that ÷ 100 rounded **up** to whole rods. |

- Sorted fewest days first. The card shows the 5 most urgent. Click "show details" for all.
- Pre-cut pieces are added to the tube they were cut from. Cut pieces never cut from a known tube appear as one "(cut pieces)" row with no order suggestion.

**What to do**
Order the rods shown in the **Order ft / rods** column. Do a physical stick count once a month, because feet can be overstated when cut pieces were booked without a matching rod issue.

**Example**
A brass tube has 79.6 ft of rod plus 5 pre-cut pieces of 4 ft (20 ft), so **99.6 ft** on hand. Used in 30 days: 400 ft − 10 ft returned + 12 pieces × 4 ft = **438 ft**. Per day = 14.6 ft. Cover = 99.6 ÷ 14.6 = **6.8 days** (Critical). Keep = 438 ft. Order = 438 − 99.6 = 338.4 ft, which is 4 rods rounded up.

> 📸 **Screenshot 20: Tube Stock**
> - **Where to take it:** Materials tab, Tube Stock tile.
> - **What must be visible:** the 5 urgent rows with all columns.
> - **Highlight:** circle the Order ft / rods column.
> - **File name:** `images/stock-forecast/20-tube-stock.png`
>
> `![Screenshot 20: Tube Stock](images/stock-forecast/20-tube-stock.png)`

> 📸 **Screenshot 21: Tube Stock details**
> - **Where to take it:** click "show details" on the Tube Stock tile.
> - **What must be visible:** the popup with one row per pipe.
> - **Highlight:** circle the "(cut pieces)" row if shown.
> - **File name:** `images/stock-forecast/21-tube-stock-details.png`
>
> `![Screenshot 21: Tube details](images/stock-forecast/21-tube-stock-details.png)`

---

### 4.14 Acrylic Rod Stock, in inches (Materials)

**What it shows**
One row per diameter and stick-length class that was used in the window. Sticks come in different lengths, so counting sticks would mislead. The tile counts **inches**.

**How to read it**
- Length classes: up to 12", 13-24", 25-48" and over 48".
- **On hand** = sticks on hand × stick length, in inches.
- **Demand** = inches of whole sticks issued, minus sticks returned. Waste is included, because an unusable offcut is never booked back.
- Per day, days left and risk work as for tube.
- The card shows the 5 with the most use. "Show details" shows all, fewest days first.
- A longer stick can serve a shorter cut, so short classes look worse than they are. The long sticks of heavily used diameters are usually the real risk.
- The popup also shows pieces cut by diameter and cut length.

**What to do**
Order long sticks for diameters that are heavily used. Book unused remainders as a **Return**, not as a plain receipt.

**Example**
1.5" rods, over 48": 10 sticks of 60" + 5 sticks of 72" = 600 + 360 = **960 inches**. Used in 30 days: (6 − 1) × 60 + 2 × 72 = **444 inches**. Per day = 14.8. Cover = 960 ÷ 14.8 = **64.9 days** (Low).

> 📸 **Screenshot 22: Acrylic Rod Stock**
> - **Where to take it:** Materials tab, Acrylic Rod Stock tile.
> - **What must be visible:** the five rows with diameter, length class, inches and days left.
> - **Highlight:** circle the length class column.
> - **File name:** `images/stock-forecast/22-acrylic-stock.png`
>
> `![Screenshot 22: Acrylic Rod Stock](images/stock-forecast/22-acrylic-stock.png)`

---

### 4.15 Buildable Orders (Materials)

**What it shows**
Which active orders we can build with the stock we have today.

**How to read it**
Orders are taken **oldest order date first**. Stock is handed out in that order.
- **Ready** (grey): every component is available. This order **takes** that stock.
- **Partial** (amber): some components are short, others are fine.
- **Short** (red): every component is short.
- **Only Ready orders take stock.** A Partial or Short order takes nothing, so an order that cannot be built never holds back a later one that can.
- Columns: Internal Order ID (a link), Order Type, Order Date, Order Value, Status, and the limiting parts ("item (need N, have N, missing N)").
- The summary line shows Ready, Partial and Short counts, **Blocked** (the value of Partial and Short orders) and the value of Ready orders.
- Included: active, not-yet-dispatched orders (New up to Pending India Dispatch or Awaiting India Components). Orders whose material is already issued are left out. Custom-order lines with no BOM are skipped.

**What to do**
Read the limiting parts of each Short order. Check them in the Restock List and buy what is missing. Until then the order stays Short.

**Filters:** Category, column filter boxes and **Export**.

**Limits:** priority is order date, not the promised ship date, and the method is first come, first served.

**Example**
Item X: 10 in stock. Order A (older) needs 6 and is **Ready**. It takes 6, leaving 4. Order B needs 6 but only 4 are left, so B is **Short**: it needs 6, has 4 and is missing 2.

> 📸 **Screenshot 23: Buildable Orders**
> - **Where to take it:** Materials tab, Buildable Orders tile.
> - **What must be visible:** the summary line and rows with Ready, Partial and Short status and their limiting parts.
> - **Highlight:** blur customer names and order values if shown. Circle the **Blocked** figure.
> - **File name:** `images/stock-forecast/23-buildable-orders.png`
>
> `![Screenshot 23: Buildable Orders](images/stock-forecast/23-buildable-orders.png)`

---

### 4.16 Stock Data Quality (header chip and checklist)

**What it shows**
A checklist of problems that make the forecast wrong. Each line shows a count, or OK. Click a line to see **where to fix it** and the affected records (up to 200, each a link).

**How to read it**
- Aim for **zero**. The count is the number of lines that are not OK, not the number of records.
- Amber or red lines need action.

| Line | What it means |
|---|---|
| Lyfe BOM components not matched to an Item | A BOM component code matches no item, so it is ignored. |
| Open order lines without an Item | Their quantity cannot be counted as committed. |
| Components where the Lyfe BOM disagrees with actual issues | The BOM is marked Unreliable. |
| Acrylic rods with unit Inch | Should be counted in "Nos". |
| Acrylic rods with a decimal written as P | For example, 2P75L should be written 2.75L. It is not recognised as a rod. |
| Pending Material Requests (not used) | For information. The page ignores them. |
| Stock with no use in 90 days | Dead stock, for information. |
| Stock above 3x target cover | Excess stock, for information. |
| Last week's forecast bias beyond +/-30% | The forecast has been consistently too high or too low. |
| Nightly snapshot older than a day (or missing) | The nightly update did not run. |

**What to do**
1. Click the **Data issues** card.
2. Open each amber or red line and follow its "Where to fix" note.
3. Refresh and check again.

**Who can see it:** by default, everyone who can open the page. An administrator can limit it to the Administrator user in Stock Forecast Settings. That also hides the Data issues card.

**Example**
"Open order lines without an Item: 7". Click it, open each order and link the correct item. Next day the line shows OK.

> 📸 **Screenshot 24: Stock Data Quality**
> - **Where to take it:** click the **Data issues** card so the checklist opens.
> - **What must be visible:** the checklist lines with counts, and one line opened to show "Where to fix" and the record links.
> - **Highlight:** blur any customer or order details in the record links.
> - **File name:** `images/stock-forecast/24-data-quality.png`
>
> `![Screenshot 24: Stock Data Quality](images/stock-forecast/24-data-quality.png)`

---

## 5. Common questions (FAQ)

**How do I find a tile I used to see?**
Tiles are now in three tabs: Act now, Demand & forecast and Materials. See the table at the start of section 4.

**Why is the first click on a tab slow?**
A tab loads its tiles only when you open it. That keeps the page quick to open. **Refresh** reloads everything.

**What filters can I use?**

| Control | What it does |
|---|---|
| **Category and Item Code boxes (top)** | Narrow the whole page. A main category includes its sub-categories. Picking one finish shows the product's combined "×N finishes" row. Tube and Acrylic are not affected. |
| **Funnel on a tile** | Sets that tile's own filter (days, top N, category). It wins over the top boxes for that tile. |
| **Trailing days** | On Stockout Risk, Tube, Acrylic and Material Usage. Sets how many days of history define "daily use" (1 to 365). Leave it blank for the default, 30 days. |
| **Refresh** | The main button reloads the page. Each tile also has its own refresh arrow. |
| **Column boxes** | Under a table heading, hide rows that do not contain what you type. |
| **Export** | CSV on the tables. Excel on the US page. |

**Where does "daily use" come from?**
From material **actually issued to the factory** for orders. It is not based on sales. Sales lines explain only a small part of what the factory takes. The page turns the last 26 weeks of issues into a forecast daily rate (the TSB method), leaving out one-off big orders. If you type your own Trailing days, it uses the plain average over those days instead. Transfers, receipts and stock corrections are not real use, so they are ignored. So is material taken from the Rejected Material warehouse.

**What counts as "stock on hand"?**
What is on our shelves, added up across all warehouses, **except** the Rejected Material warehouse. Rejected material cannot be used.

**What is "Committed"?**
Material already promised to open orders whose material has not been issued yet. Orders already issued are left out, so nothing is counted twice. Lines marked as adjustments, or with no linked Item, are not counted.

**Why does Available differ from Stock on Hand?**
Available is Stock on Hand minus Committed.

**Why is an item Critical when it has 25 days of cover?**
Its Available is below zero. Open orders already need more than we have.

**Why is an item missing from Stockout Risk?**
It has 30 or more days of cover, had no use in the window, is tube or acrylic (they have their own tiles), or is set to Make to Order or Do Not Stock.

**Why is the suggested quantity bigger than I expected?**
It restores the target cover (default 30 days), adds a safety buffer, and is rounded up to the Minimum Order Qty.

**Does the page know what I have already ordered?**
No. It sees stock only when received. Check **Last Received** and ask the buyer.

**Why does a material appear in the Total tile but not in the order tile?**
The Total tile adds all orders together. The order tile checks each order alone against full stock.

**Why is there a Short order when stock looks fine?**
An older Ready order already took the stock.

**Why can I not see the US Warehouse Replenishment tile?**
It is shown only to System Manager, Super Admin and people with the US Replenishment Manager role.

**Why is the trend arrow missing?**
The change is under 25%, there is no 90-day history, or the item was issued fewer than 3 times in 90 days. If one big order caused the jump, you see "⚑ unexpected order" instead.

**What is a Lyfe BOM and how is it used?**
It is our own recipe for a product: which components, how many, and the scrap allowance. For each open order, the page works out the material it needs. Details are in section 7.

**What if a Lyfe BOM is wrong?**
The system checks the last 90 days. If, over at least 20 orders, what the BOM predicted is far from what the factory really issued, that BOM is marked **Unreliable** and its numbers are not counted in Committed. Ask Engineering to correct it.

**Can I change the 7 / 15 / 30 day levels?**
Yes. An administrator can change them in the briefing settings. The page and the nightly update use the same levels.

**How fresh are the numbers?**
Most tiles are live when you load or refresh. The "Last updated" time is at the top.

**Does the page send alerts?**
It can, but the Slack alert is **off** by default. An administrator turns it on in Stock Forecast Settings ("Send Risk Alerts to Slack"). When on, it sends one message a day, after the nightly update, listing items that are Critical or High today and either moved to a worse level, are new, or stayed Critical for 7 days in a row. A message lists at most 15 items.

The morning **Founder Briefing** has stock sections too (Projected Stockouts and Tube & Acrylic Stock), when the briefing is switched on.

**What are the limits of the forecast?**
- History starts in May 2026, so there is no seasonality.
- The forecast does not know about promotions or new launches unless they are already open orders.
- Incoming stock appears only when it is received. The page has no supplier or purchase order information.
- BOM numbers are estimates. Unreliable BOMs are ignored.
- Items never used do not appear, even if open orders need more than the stock.
- Dead stock can be seasonal. With a few months of history, treat the dead list as a review list.
- **Revenue exposed** is a risk figure, not proof an order is blocked. Use Buildable Orders for the real blocked value.

**A tile says "Data unavailable".**
Press **Retry** on the tile. If it keeps failing, tell the system team.

**What should I do about a Critical item?**
1. Open **View forecast detail** and expand the item.
2. Check **Confidence**. If Low, the daily use may be unreliable.
3. Check **Last Received** in the Restock List.
4. Confirm with the buyer that nothing is already on the way.
5. Order the **Suggested Qty**, or more if a big order is coming.

**What should I do about an item with negative Available?**
1. Open orders already need more than we hold, so it is Critical whatever the cover.
2. Check how much is **BOM committed** and whether the BOM is reliable.
3. In Material Not In Stock, tick the item to see which orders are affected.
4. Buy at least the shortfall and tell Customer Support if ship dates will slip.

**What should I do about a Short order?**
1. In Buildable Orders, read the limiting parts.
2. Check each part in the Restock List.
3. Buy the missing items. Until then the order stays Short.

**What should I do when a trend arrow shows "up"?**
1. Look at how far it is above the 90-day average.
2. Ask whether it is a one-off or lasting. A single outsized order shows "⚑ unexpected order" instead of an arrow, so an arrow means several orders over time.
3. If lasting, raise the target cover or buy extra.

---

## 6. Glossary (A to Z)

| Term | Plain meaning |
|---|---|
| **ABC class** | A label for how much of the volume an item makes up. Items are ranked by units issued in the last 90 days. A = the fast movers that make up the first 80%, B = the next 15% (up to 95%), C = the rest. It is worked out every Monday. It is used to report accuracy by group and to size safety stock. |
| **Available** | Stock on Hand minus Committed. Below zero means orders already need more than we have. |
| **Backlog-aware suggestion** | An optional way to work out Suggested Qty. It takes whichever is bigger, the committed amount or daily use × target cover, and subtracts what is on hand. Off by default. An administrator can turn it on in Stock Forecast Settings. |
| **Base-level forecast** | A forecast for a base product (for example CMBR-200 Matte) with all its finishes added together. |
| **Bias %** | Whether the forecast was consistently too high (positive) or too low (negative). |
| **Blocked** | The dollar value of orders that truly cannot be fully built from current stock (Buildable Orders). |
| **Child SKU** | A component SKU that belongs to a product. Used when a line has no BOM: the parts a previous order actually used. |
| **Committed** | Material already promised to open orders that have not had their material issued yet. |
| **Confidence** | How much history sits behind the daily-use number. **High** = the item moved in 8 or more of the last 13 weeks, **Medium** = 4 to 7 weeks, **Low** = under 4 weeks. |
| **Consumption / Daily use** | How many units the factory takes from stock per day. By default this is a forecast rate worked out from the last 26 weeks of issues (the TSB method), with one-off big orders left out. If you choose your own Trailing days, it is the plain average over those days. |
| **Days of cover** | How many days the stock on hand lasts at the daily use rate. |
| **Dead stock** | Stock with no use in 90 days. |
| **Error %** | How far off the forecast was overall. Lower is better. |
| **Excess stock** | Stock above 3 times the target cover. |
| **Finish** | The surface treatment of a product, for example polished brass. One product can have several finishes. |
| **×N finishes** | One row standing for the same product in several finishes. Their stock, use and open orders are added together. |
| **Forecast accuracy** | How close the forecast was to what was really used. Checked every Monday. |
| **Lead time** | How long it takes from ordering to having the item. For safety stock, the page uses the item's lead time, or 14 days until a real one is measured. For the US warehouse, lead time is the factory time plus transit plus customs. |
| **Live vs Snapshot** | **Live** means calculated right now. **Snapshot** means taken from last night's saved copy. |
| **Lyfe BOM** | Our own recipe for a product: which components, how many of each, and the scrap allowance. |
| **Minimum Order Qty (MOQ)** | The smallest pack or lot we can buy. The suggestion is rounded up to a whole multiple of it. |
| **Nightly update (snapshot)** | A copy of every item's forecast saved at about 1:15 AM each night. |
| **On hand** | What is physically on our shelves now, across all warehouses except Rejected Material. |
| **One-off (unexpected) order** | One big issue that makes up at least half of an item's 90-day use and is at least 3 times the typical other issue. It is not treated as normal demand. |
| **Reorder point** | The stock level at which we should order. It is daily use × lead time + safety stock. |
| **Required Qty** | How much of an item the orders need in total. Example: 10 per order × 5 active orders = 50. |
| **Revenue exposed** | The dollar value of open-order lines that contain an item at risk (Critical, High or Medium). A warning, not proof that the order cannot be built. |
| **Risk level** | Critical, High, Medium or Low, based on days of cover. Default: Critical under 7 days, High 7 to under 15, Medium 15 to under 30, Low 30 or more. |
| **Safety stock** | Extra stock we keep so a busy week does not leave us out. It is bigger for fast items and for items whose use jumps around. |
| **Service level** | The share of the time we want to avoid running out. The US page uses 95%. A higher service level means more safety stock. |
| **Stock position** | On the US page: stock at 1Click + stock on its way + stock in production. |
| **Stock-out date** | Today plus the whole days of cover. The day we expect to hit zero if nothing is added. |
| **Suggested (Order) Qty** | How many units to buy to get back to the target cover (plus safety stock). |
| **Target cover** | How many days of stock we want to hold. Default 30. An item can have its own target. |
| **Trailing days** | The number of past days used to work out daily use. Default 30. |
| **Trend arrow** | Shown on Stockout Risk rows. "up +40%" means recent daily use is 40% higher than the 90-day average. |
| **TSB** | The forecast method the page uses. It is made for items that sell now and then, with many empty weeks. It tracks how often the item sells and how much per sale, gives recent weeks the most weight and fades the forecast when an item stops selling. One-off big orders are left out of the rate. |
| **Weeks of cover** | On the US page: stock at 1Click ÷ average weekly demand. |

---

## 7. For developers

This section is for the people who maintain the dashboard. The full technical reference is `docs/stock-forecast-dashboard.md` in the `lh` repository. The US page is in `docs/us-replenishment-dashboard.md`. Every API is in `STOCK_FORECAST_API_DOCUMENTATION.md`. If this guide and the code disagree, the code is right.

<details>
<summary><strong>Data sources</strong></summary>

- **Stock on hand:** stock balance of the item in every warehouse except the Rejected Material warehouse.
- **Consumption:** Stock Ledger Entries of **Material Issue** stock entries (not cancelled, outflow only), excluding the Rejected warehouse. Transfers, receipts and reconciliations are not counted. Returns are netted for tube and acrylic.
- **Committed:** open Lyfe Orders (not Cancelled, Merged, Split, Shipped, Completed or Return Successfully) whose material has not been issued. Lines marked as adjustments or with no Item are skipped.
- **US replenishment:** US-destination Lyfe Order lines (demand), the 1Click stock snapshot, Stock Transfer Order 1Click lines (stock on its way) and Item Prices (value).
</details>

<details>
<summary><strong>How each order's material is worked out (BOM logic)</strong></summary>

In this order:
1. The order's own component list, if it has one. It is used for the whole order.
2. The Lyfe BOM of each line. Quantity per unit includes scrap. Only one level is opened.
3. If a line has no BOM: what a previous Shipped or Completed order of the same product actually used (the child SKUs).
4. Otherwise the item itself, 1 per unit sold.

BOM example: kit K has component C1 × 2 with 5% scrap and component C2 × 1. Per kit, C1 = 2 × 1.05 = 2.1. An order of 3 kits needs C1 = 6.3 and C2 = 3.

A BOM is marked **Unreliable** when, over at least 20 orders in the last 90 days, actual issues fall outside 70% to 130% of the prediction. With fewer than 20 orders the rating is **Unknown**.
</details>

<details>
<summary><strong>Calculations (stock tiles)</strong></summary>

- **Daily use:** the TSB forecast on the item's last 26 weekly totals, with one-off bulk orders removed. A custom Trailing days window, or an item whose TSB is 0, uses the plain average (units issued in the window ÷ days in the window).
- **Days of cover** = stock on hand ÷ daily use. Zero or negative stock is Critical.
- **Available** = on hand − committed. Below zero is always Critical.
- **Suggested qty (default)** = daily use × target cover − available, rounded up to a whole number, then rounded up to the Minimum Order Qty if the item has one. Worked example: on hand 60, daily use 3, committed 28, so available 32. Target cover 30 days. Need = 3 × 30 − 32 = 58. With a minimum order of 25, the suggestion is 75.
- **Backlog-aware (optional):** the bigger of committed and daily use × target cover, minus on hand. Same example: the bigger of 28 and 90 is 90, minus 60 on hand = 30.
- **Safety stock** = z × spread of weekly use (last 13 weeks, quiet weeks count as 0) × √(lead time ÷ 7). z depends on the ABC class: A 2.05 (98%), B 1.65, C 1.28. Unclassified items use B. It needs 4 or more weeks with use, otherwise it is 0, and it is capped at 30 days of use.
- **Reorder point** = daily use × lead time + safety stock.
- **ABC class:** items are ranked by units issued in the last 90 days. A is the first 80% of units, B up to 95%, C the rest. Items you lock in the Item are skipped.
- **Target cover** = the item's own Target Cover (days) if set, otherwise the Stock Forecast Settings value (default 30).
- **Tube:** used = rod feet issued − returned + cut pieces issued × their length. Order rods = (keep − on hand) ÷ 100, rounded up.
- **Acrylic:** on hand = sticks × length in inches. Demand = whole sticks issued minus returned, in inches.
- **Buildable Orders:** oldest order date first. Only Ready orders take stock.
- **Forecast vs Actual:** forecast for a week = units issued in the 30 days before the week ÷ 30 × 7. It rebuilds and scores the older 30-day-average rule, so it measures that method, not TSB.
- **Base-level Forecast:** reads Material Issue entries of the last 26 weeks, groups finished SKUs by base product and base state, adds open-order demand and stock.
</details>

<details>
<summary><strong>Calculations (US Warehouse Replenishment)</strong></summary>

- **Weekly demand:** units on US-destination Lyfe Order lines per Monday-start week, whichever warehouse shipped them. Cancelled, Merged and Split orders and stock-transfer orders are left out. The current incomplete week is not counted. Weeks are zero-filled from the item's first sale.
- **Weighted average:** newest 4 weeks 50%, weeks 5 to 12 30%, weeks 13 to 26 20%. A band with no weeks yet is skipped and the others re-weighted.
- **Lead time (weeks)** = factory working days ÷ working days per week + transit weeks + customs weeks. Defaults: 3 ÷ 6 + 3 + 1 = 4.5.
- **Safety stock** = Z × standard deviation of weekly demand × √lead time. Z comes from the service level (95% = 1.645).
- **Reorder point** = average × lead time + safety stock. **Target** = reorder point + average × coverage weeks (default 4).
- **Stock position** = 1Click available + in transit + in production. In production is 0 for now.
- **Suggested qty** = max(0, target − position), rounded up to whole units, or to Minimum Order Qty multiples when set. Overstock suggests 0.
- **Stock on its way** = Stock Transfer Order 1Click lines whose Lyfe Order is not yet delivered to the US warehouse.
- **Suggested value** = suggested qty × Item Price from the price list set in US Replenishment Settings (default Standard Selling).
- Every number is set in **US Replenishment Settings**.
</details>

<details>
<summary><strong>Refresh timing and scheduled jobs</strong></summary>

| When | What runs |
|---|---|
| On page load or Refresh | Live tiles read the database. Stockout rows are cached for about 45 seconds. US rows are cached for 60 seconds. |
| Every night, about 1:15 AM | **Nightly snapshot** of every item's forecast. Used for history, alerts, the accuracy check, and the Stockout and Category tiles if "Stockout Tile Reads Snapshot" is on (off by default). If it is missing or older than a day, Stock Data Quality shows a red line. A custom Trailing days is always live. |
| Every Monday, about 2:30 AM | Item ABC classes, last week's forecast accuracy (the saved forecast from about a week ago against what was issued), measured lead times from Lyfe POs, and the US replenishment weekly log. |
| Hourly check, every 4 hours by default | **1Click stock sync** into the US stock snapshot. A failed sync keeps the last good snapshot and shows a warning. |
| Once a day, after the nightly update | Optional Slack risk alert (off by default). |

Who can see what: Stock Forecast endpoints use the `stock_forecast` role set (System Manager, Super Admin, Factory). The US tile and page use the `us_replenishment` role set (System Manager, Super Admin, US Replenishment Manager).
</details>

---

## Screenshot checklist

| # | File name | Page / tab | What to capture | Done |
|---|---|---|---|---|
| 1 | `images/stock-forecast/01-overview.png` | Stock Forecast, Act now | Whole page top: cards, filters, tabs, first tiles | ☐ |
| 2 | `images/stock-forecast/02-summary-cards.png` | Top of page | Four cards with numbers | ☐ |
| 3 | `images/stock-forecast/03-restock-list.png` | Act now | Restock List with Why and Last Received | ☐ |
| 4 | `images/stock-forecast/04-restock-list-days.png` | Act now | Restock List funnel open with 14 days | ☐ |
| 5 | `images/stock-forecast/05-stockout-risk.png` | Act now | Stockout Risk rows, badges, trend arrow | ☐ |
| 6 | `images/stock-forecast/06-stockout-risk-detail.png` | Act now | View forecast detail with an expanded row | ☐ |
| 7 | `images/stock-forecast/07-us-tile.png` | Act now | US Warehouse Replenishment tile | ☐ |
| 8 | `images/stock-forecast/08-us-page.png` | `/app/us-replenishment` | Full page: tiles, filters, table, buttons | ☐ |
| 9 | `images/stock-forecast/09-us-item-why.png` | `/app/us-replenishment` | Item detail window with the step-by-step numbers | ☐ |
| 10 | `images/stock-forecast/10-draft-transfer.png` | Stock Transfer Order 1Click | Draft with items and Create Lyfe Order button | ☐ |
| 11 | `images/stock-forecast/11-not-in-stock-total.png` | Act now | Material Not In Stock - Total Required | ☐ |
| 12 | `images/stock-forecast/12-active-orders-not-in-stock.png` | Act now | Active Orders - Material Not In Stock (blur names) | ☐ |
| 13 | `images/stock-forecast/13-demand-trends.png` | Demand & forecast | Demand Trends with Orders column | ☐ |
| 14 | `images/stock-forecast/14-demand-forecast.png` | Demand & forecast | Demand Forecast with a red "left after" | ☐ |
| 15 | `images/stock-forecast/15-base-forecast.png` | Demand & forecast | Base-level Forecast rows | ☐ |
| 16 | `images/stock-forecast/16-category-forecast.png` | Demand & forecast | Category bars and toggle | ☐ |
| 17 | `images/stock-forecast/17-category-drilldown.png` | Demand & forecast | Sub-group or item list after clicking a bar | ☐ |
| 18 | `images/stock-forecast/18-material-usage.png` | Demand & forecast | Material Usage list | ☐ |
| 19 | `images/stock-forecast/19-forecast-vs-actual.png` | Demand & forecast | Summary, chart and Biggest misses | ☐ |
| 20 | `images/stock-forecast/20-tube-stock.png` | Materials | Tube Stock card | ☐ |
| 21 | `images/stock-forecast/21-tube-stock-details.png` | Materials | Tube Stock details popup | ☐ |
| 22 | `images/stock-forecast/22-acrylic-stock.png` | Materials | Acrylic Rod Stock card | ☐ |
| 23 | `images/stock-forecast/23-buildable-orders.png` | Materials | Buildable Orders (blur names and values) | ☐ |
| 24 | `images/stock-forecast/24-data-quality.png` | Data issues card | Checklist with one line opened | ☐ |

---

Last verified against the live implementation: 2026-11
