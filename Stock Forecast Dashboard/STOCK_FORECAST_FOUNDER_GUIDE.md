# Stock Forecast Dashboard — What It Does (Plain English Guide)

*Last updated: 7 Oct 2026*

This guide is for the Founder and anyone at Lyfe Hardware who opens the **Stock Forecast** page. You do not need any technical background. If you know our orders, the factory, tubes, rods and brackets, you can read this guide and understand every number on the page.

---

## 1. What is the Stock Forecast page for?

It answers four questions:

1. **Which items will run out, and when?**
2. **How much should we buy to get back to a healthy level?**
3. **Which open orders are being held up by missing material, and how much money is that?**
4. **Can we trust these numbers?**

Every number comes from the records the team already works in every day: stock on the shelf, material issued to the factory, and open orders. Nothing is typed in by hand.

**Who can open it:** people with the role **System Manager**, **Super Admin** or **Factory**. A Factory user can see this page but not the Founder Dashboard's money pages (revenue, profit). The Founder Dashboard itself only shows one small **Stock** banner that links here.

**What the page does not do:** it does not know about purchase orders, suppliers or delivery times. It tells you how long stock will last and how much to buy. When to place the order is your decision.

---

## 2. Glossary — the words used on the page

| Word | Plain meaning |
|---|---|
| **Stock on Hand** | What is physically on our shelves now, added up across all warehouses, **except** the Rejected Material warehouse. |
| **Committed** | Material already promised to open orders that have not had their material issued yet. |
| **Available** | Stock on Hand minus Committed. If it is below zero, orders already need more than we have. |
| **Required Qty** | How much of an item the orders need in total. Example: 10 per order x 5 active orders = 50. |
| **Consumption / Daily use** | How many units the factory actually took from stock per day, on average, over the last 30 days (or the Trailing days you choose). |
| **Days of Cover** | How many days the stock on hand lasts at the recent daily use. |
| **Stock-out date** | Today plus the whole days of cover. The day we expect to hit zero if nothing is added. |
| **Risk level** | Critical, High, Medium or Low, based on days of cover. Default: **Critical** under 7 days, **High** 7 to under 15, **Medium** 15 to under 30, **Low** 30 or more. These cut-offs can be changed by an administrator. |
| **Suggested (Order) Qty** | How many units to buy to get back to the target cover. |
| **Target cover** | How many days of stock we want to hold. Default 30. An item can have its own target. |
| **Minimum Order Qty (MOQ)** | The smallest pack or lot we can buy. The suggestion is rounded up to a whole multiple of it. |
| **Confidence** | How much history sits behind the daily-use number. **High** = the item moved in 8 or more of the last 13 weeks, **Medium** = 4 to 7 weeks, **Low** = under 4 weeks. |
| **Revenue exposed** | The dollar value of open-order lines that contain an item at risk (Critical, High or Medium). It is a warning, not proof that the order cannot be built. |
| **Blocked** (Buildable Orders) | The dollar value of orders that truly cannot be fully built from current stock. This is the real "blocked" figure. |
| **Trend arrow** | Shown on Stockout Risk rows. "up +40%" means recent daily use is 40% higher than the 90-day average. "down -30%" means lower. Shown only when the change is 25% or more. |
| **Lyfe BOM** | Our own recipe for a product: which components, how many of each, and the scrap allowance. It is the only recipe the page uses. |
| **Child SKU** | A component SKU that belongs to a product. In the fallback rule (section 3.7), the parts a previous order actually used. |
| **Live vs Snapshot** | **Live** means calculated right now. **Snapshot** means taken from last night's saved copy. The Stockout Risk tile says which one it is using. |

---

## 3. How the numbers are made

### 3.1 Stock on hand
We add up the stock of the item in every warehouse **except** the Rejected Material warehouse. Rejected material cannot be used, so it is not counted. This matches the stock balance in the system exactly.

### 3.2 Consumption: what the factory actually issued
Daily use is based on **material actually issued to the factory** (the Material Issue entries created when an order's material is issued). It is **not** based on sales. Reasons: sales lines only explain a small part of what the factory really takes, and transfers, receipts and stock corrections are not real use, so they are ignored. Material taken from the Rejected warehouse is ignored too.

> **Example.** In the last 30 days the factory issued 90 units of item A. Daily use = 90 / 30 = **3 per day**.

### 3.3 The trailing window and the Trailing days filter
"Trailing 30 days" means the last 30 days up to now, including today so far. The total is divided by 30. On the tiles that have it, you can type your own **Trailing days** (1 to 365) in the filter. Leave it blank for 30. Only the average-use figures change; the Critical/High/Medium cut-offs stay the same.

### 3.4 Days of cover and stock-out date

> **Example.** 60 on hand, 3 used per day. Days of cover = 60 / 3 = **20 days**. That is Medium (15 up to 30). Stock-out date = today + 20 days.

If nothing is on hand, cover is 0 days and the item is **Critical** ("stocked out"). Negative stock is also Critical.

### 3.5 Risk levels

| Level | Days of cover (default) |
|---|---|
| Critical | under 7 |
| High | 7 to under 15 |
| Medium | 15 to under 30 |
| Low | 30 or more |

One extra rule: if **Available is below zero** (open orders already need more than we have), the item is always shown as **Critical**, whatever its days of cover.

### 3.6 Committed demand from open orders
Committed = material that open orders still need but that has **not** been issued yet.

- An order counts if it is not Cancelled, Merged, Split, Shipped, Completed or Return Successfully.
- An order whose material has **already been issued** is left out. Its stock has already left the shelf, and counting it again would double count.
- Lines marked as adjustments, or with no linked Item, are not counted. Lines with no linked Item appear in Stock Data Quality.

### 3.7 How Lyfe BOM components are used
For each open order, the page works out what material it needs, in this order:

1. **The order's own component list**, if the order has one. It is used for the whole order.
2. **The Lyfe BOM** of each line. Quantity per unit **includes scrap**. Only one level is opened (a component is treated as a stocked part).
3. If a line has no BOM: **what a previous Shipped or Completed order of the same product actually used** (the parts recorded on that order's shipment). Each part is counted as a child SKU.
4. If nothing else exists: **the item itself**, 1 per unit sold.

> **BOM example.** Kit K has a Lyfe BOM: component C1 x 2 with 5% scrap, component C2 x 1. Per kit, C1 = 2 x 1.05 = **2.1**. An order of 3 kits needs C1 = 6.3 and C2 = 3. Another order of 2 kits needs C1 = 4.2 and C2 = 2.

**Unreliable BOMs are ignored.** The system checks the last 90 days: for orders already issued, it compares what the BOM predicted with what the factory really issued. If, over at least 20 orders, actual is outside 70% to 130% of the prediction, that component's BOM is marked **Unreliable**. Its numbers are still shown as "BOM committed" for information but are **not** counted in Committed. With fewer than 20 orders the rating is **Unknown**.

### 3.8 Available
**Available = Stock on Hand - Committed.** Below zero = always Critical. "Available days" = Available / daily use (never below 0).

### 3.9 Suggested quantity
Default method: **daily use x target cover - Available**, rounded up to a whole number, then rounded up to the Minimum Order Qty if the item has one. If the result is zero or less, nothing is suggested.

> **Example.** On hand 60, daily use 3, Committed 28, so Available 32. Target cover 30 days.
> Need = 3 x 30 - 32 = 58. Suggested = **58**.
> If the item's Minimum Order Qty is 25 (packs of 25): 58 rounds up to 75. Suggested = **75**.
> If the BOM were Unreliable, Committed drops to 10, Available is 50, and the suggestion is 90 - 50 = **40**.

**Optional backlog-aware method** (off by default; an administrator can switch it on in Stock Forecast Settings). Idea: future daily use already includes the open orders, so the default counts them twice. This method takes whichever is bigger, the committed amount or daily use x target cover, and subtracts what is on hand.
Same example: the bigger of 28 and 90 is 90; minus 60 on hand = **30**.
The weekly accuracy check (section 9) shows which method fits better before you switch.

### 3.10 How the forecast looks ahead
The page does **not** predict the future cleverly. It assumes **future use = recent use**. Cover is simply "stock divided by recent daily use".
- No seasonality (history only starts in May 2026, so there is not enough to learn from).
- No promotions, new launches or known big orders unless they are already open orders.
- Items with no use in the window are not listed (an idle item is not about to run out).

So a sudden spike or a quiet spell will make the forecast wrong for a while. The **trend arrow** helps you spot this.

### 3.11 Categories (Item Groups)
Items belong to Item Groups. The Category view counts items by risk level for each **top-level group**, then its sub-groups, then the items. Items with no group fall into **Other**. The page counts items and dollars only. It never adds up units of different things.

### 3.12 Tube and acrylic rods have their own units
- **Tube** is tracked in **feet**, one row per tube material and diameter. Cut lengths and finishes are rolled up, because finish is applied after cutting and one finish uses several materials. Rods are assumed to be **100 ft** long.
- **Acrylic rods** are tracked in **inches**, by diameter and stick-length class (up to 12", 13-24", 25-48", over 48"). Sticks come in different lengths, so counting "sticks" would be misleading.

These items do **not** appear in Stockout Risk or the category bars (they have their own tiles).

---

## 4. Each tile, in page order

### 4.1 Summary cards (top strip)
Four clickable cards. Click one to jump to the related tile.

| Card | What it shows |
|---|---|
| **Orders short now** | Number and dollar value of orders that are Partial or Short in Buildable Orders. Sub-line shows how many are Ready. |
| **Items to reorder** | Number of rows in the Restock List. Sub-line shows how many are Critical and High. |
| **Shortest cover** | The one item with the fewest days of stock, and its days. "No usage data" if nothing has been used. |
| **Data issues** | Number of data problems, or "None". Click it to open the Stock Data Quality checklist. Shown only to people allowed to see data quality. |

A small refresh arrow under the cards reloads just the cards.

### 4.2 Stockout Risk
**What it shows:** every item that was used in the window and has **under 30 days** of cover (Medium, High or Critical). Low-risk items are not listed.

**Each row:** item code, "used N in 30d" (with the trend arrow when relevant), available, suggested order quantity (only when above zero), days of cover, and "exposes $X" (Revenue exposed) or "no open orders". A risk badge with a symbol (not just colour). The footer shows the count of Critical / High / Medium.

**Sort order:** items where open orders already exceed stock first, then Critical, High, Medium, then higher dollars exposed, then fewer days of cover.

**Filters:** Category (includes its sub-groups) and Trailing days.

**Left out on purpose:** items with no use in the window, tube and acrylic rods (own tiles), and items whose stock policy is "Make to Order" or "Do Not Stock". Acrylic rod caps are not rods and stay here.

**What to click:** **View forecast detail** opens the full list with a search box and a risk filter. Click a row to see more: daily use, committed, confidence, BOM committed with its reliability, recommended action, **Where used** (which Lyfe BOMs contain the item and how many open orders carry them) and **Go to** links to the Item and its stock ledger. **Download restock list (CSV)** exports every row.

The subtitle says **Live** or **Snapshot - date**.

### 4.3 Material Usage
**What it shows:** the items most issued to the factory for orders over the window (default 30 days), most used first, in a scroll box. It is a mix of raw materials (feet, sticks) and finished pieces, so the units differ per row. It is labelled "Most-Issued", not "Most-Used Material".

**Numbers:** net quantity = issued minus returned. Only items above zero are shown.

**Filter:** Trailing days. The full-list popup always uses 30 days.

### 4.4 Category-wise Stock Forecast
**What it shows:** one stacked bar per top-level category, split into Critical, High, Medium and Low item counts. It covers every item with use in the last 30 days, including Low cover.

**Toggle:** switch bars to **Revenue exposed** to see dollars. The other figure is printed at the end of each row.

**Sort:** most Critical first, then High, then dollars, then name. The first 6 groups show; click "show all" for the rest.

**Click-through:**
1. Click a bar (or a coloured part) to see its sub-groups, with a Back link.
2. Click a sub-group to open the item list for it (search, risk filter, details, CSV, plus Revenue exposed and Orders columns).
3. A pinned row **All critical items** opens the list of Critical items.

Two extra rows, **Tube (feet)** and **Acrylic rod (inches)**, open their own detail so those items are not left out. The footer counts **dead** stock (nothing used in 90 days) and **excess** stock (more than 3x target cover). The chart always uses 30 days.

There is no "average days" per group on purpose, so one stocked-out item can never be hidden by a healthy average.

### 4.5 Tube Stock (feet)
**What it shows:** one row per tube pipe (material x diameter) that was used in the window.

| Column | Meaning |
|---|---|
| On hand (ft) | Feet of rod in stock plus pre-cut pieces (pieces x their length). |
| Rods | Feet on hand / 100. |
| Used / per day | Feet used in the window (net of returns), and per day. |
| Days left, risk | Cover and risk level (same 7/15/30 scale). |
| Keep | Daily feet x target cover (default 30). |
| Order ft / rods | Feet short of "keep", and that divided by 100 rounded **up** to whole rods. |

**Sorted** fewest days first. The card shows the 5 most urgent; click "show details" for all. The popup gives one row per pipe.

**Pre-cut pieces** (for example a 7.5 ft cut piece) count as pieces x length and are added to the tube they were cut from. Cut pieces that have never been cut from a known tube appear as one "(cut pieces)" row with no order suggestion.

> **Tube example.** A brass tube: 79.6 ft of rod + 5 pre-cut pieces of 4 ft (20 ft) = **99.6 ft** on hand. Last 30 days: 400 ft of rod issued, 10 ft returned, 12 pieces of 4 ft issued (48 ft). Used = 400 - 10 + 48 = **438 ft**. Per day = 438 / 30 = **14.6 ft**. Cover = 99.6 / 14.6 = **6.8 days** (Critical). Keep = 14.6 x 30 = 438 ft. Order = 438 - 99.6 = 338.4 ft = 3.384 rods, rounded up to **4 rods**.

### 4.6 Acrylic Rod Stock (inches)
**What it shows:** one row per diameter and stick-length class that was used in the window.

- **On hand** = sticks on hand x stick length, in inches.
- **Demand** = inches of **whole sticks issued**, minus sticks returned. Waste is included, because an unusable offcut is never booked back.
- Per day, days left and risk work as for tube.

**Sort / display:** the card shows the 5 with the most use; "show details" shows all. Sorted fewest days first in the full list.

A longer stick can serve a shorter cut, so the short classes look worse than they really are. The long sticks of heavily used diameters are usually the real risk. The popup shows pieces cut by diameter and cut length.

> **Example.** 1.5" rods, over 48": 10 sticks of 60" + 5 sticks of 72" = 600 + 360 = **960 inches**. Last 30 days: 6 sticks of 60" issued, 1 returned, 2 sticks of 72" issued = (6-1) x 60 + 2 x 72 = **444 inches**. Per day = 14.8. Cover = 960 / 14.8 = **64.9 days** (Low).

### 4.7 Buildable Orders
**Question answered:** "Which active orders can we build with the stock we have today?"

Orders are taken **oldest order date first**. Stock is handed out in that order:
- **Ready**: every component is available. This order **takes** that stock.
- **Partial**: some components are short, others are fine.
- **Short**: every component is short.

**Only Ready orders take stock.** Partial and Short orders take nothing, so an order that cannot be built never holds material back from a later order that can.

> **Example.** Item X, 10 in stock. Order A (older) needs 6. Order B needs 6. A is **Ready** and takes 6, leaving 4. B needs 6 but only 4 are left, so B is **Short** on X (needs 6, has 4, missing 2).

**Columns:** Internal Order ID (link), Order Type, Order Date, Order Value, Status, and the limiting components ("item (need N, have N, missing N)"). The summary line shows Ready / Partial / Short counts, **Blocked** (value of Partial + Short orders) and the value of Ready orders.

**Filters:** column filter boxes and **Export**. No other filter.

**Which orders are included:** active, not-yet-dispatched orders (statuses from New up to Pending India Dispatch / Awaiting India Components). Orders whose material is already issued are left out. Custom-order lines with no BOM are skipped.

Priority is order date, not promised ship date, and the allocation is simple first-come.

### 4.8 Restock List
**Question answered:** "What do we need to buy?" One list in one place.

**What is included** (most urgent first):
- **Items**: Critical, High or Medium items with a suggested quantity above zero (uses target cover, MOQ and the backlog setting).
- **Tube**: pipes that need at least one rod, quantity in **rods**, with the feet short shown beside it.
- **Acrylic**: diameter/length classes at risk, in **inches** to order = daily inches x target cover - inches on hand.

**Columns:** Item, Type, Risk, On Hand, Available, Days Left, Order Qty, Unit, and **Last Received**.

**Last Received** = the date the item last came into stock. Restocking is recorded only when the goods arrive, so an order already placed is invisible until delivery. The page has **no supplier or purchase order information**. Use Last Received as a hint, and check with whoever placed the order before buying twice. Acrylic rows have no Last Received.

**Filters:** column boxes and Export.

### 4.9 Material Not In Stock - Total Required
**Question answered:** "For everything we still have to build, which materials are short in total?"

**Columns:** Item Code (link, with a tick box), Required Qty, In Stock, Missing Qty (= Required - In Stock). Only items with Required above In Stock are listed. Numbers are rounded up. Sorted by item code.

This tile **adds up all active orders together** before comparing with stock. Empty message: "All material for active orders is in stock."

Active orders here means undispatched orders: New through Awaiting India Components, including Factory Assignment and Ready for Dispatch. On Hold, Shipped, Completed and later stages are out.

**Tick box:** tick an item to filter the next tile to just the orders that need it. Only one tick at a time. Un-tick to clear.

### 4.10 Active Orders - Material Not In Stock
**Question answered:** "Which individual orders cannot be completed from current stock?"

**One row per order and component** where that order alone needs more than we have. **Columns:** Internal Order ID (link), Order Type, Item Code (link), Required Qty, In Stock, Missing Qty, Order Date, Order Value. Sorted by order date.

**Stock is not shared out between orders here.** Each order is compared with the full stock on its own.

> **Example.** Item Y, 10 in stock. Two orders need 6 each. In **this** tile, neither appears (6 is not more than 10). In the **Total** tile, Y appears: 12 required, 10 in stock, **2 missing**. In **Buildable Orders**, the older order is Ready and the second is Short.

Use the tick box in the Total tile to see which orders are behind a shortage.

### 4.11 Stock Data Quality
A checklist of problems that make the forecasts wrong. Each line shows a count (or OK). Click a line to see **where to fix it** and the affected records (up to 200, each a link).

| Line | What it means |
|---|---|
| Lyfe BOM components not matched to an Item | A BOM component code matches no item, so it is ignored. |
| Open order lines without an Item | Their quantity cannot be counted as committed. |
| Components where the Lyfe BOM disagrees with actual issues | BOM marked Unreliable. |
| Acrylic rods with unit Inch | Should be counted in "Nos". |
| Acrylic rods with a decimal written as P | For example a 3.75" rod written with a P. It is not recognised as a rod. |
| Pending Material Requests (not used) | For information; the page ignores them. |
| Stock with no use in 90 days | Dead stock, for information. |
| Stock above 3x target cover | Excess stock, for information. |
| Last week's forecast bias beyond +/-30% | The forecast has been consistently too high or too low. |
| Nightly snapshot older than a day (or missing) | The nightly update did not run. |

The aim is to keep this at zero. The count is the number of lines that are not OK, not the number of records.

---

## 5. Filters and drill-downs

| Control | What it does |
|---|---|
| **Category filter** | Limits Stockout Risk and the Category view to one Item Group and its sub-groups. |
| **Trailing days filter** | On Stockout Risk, Tube, Acrylic and Material Usage. Sets how many days of history define "daily use". |
| **Refresh** | The main button reloads every tile. Each tile also has its own refresh arrow. A tile that fails shows "Data unavailable" with a **Retry** button; the others keep working. |
| **Column filters** | On the Restock List, Buildable Orders and both Not In Stock tiles: type in the box under a heading to hide rows that do not contain that text. All boxes combine. |
| **Tick box** | In the Total tile; filters the order table to the exact item. |
| **Export to CSV** | Downloads what the table currently shows (header plus visible rows). |
| **View forecast detail** | Full Stockout Risk list with search, risk filter and CSV. |
| **Where used** | Shows which Lyfe BOMs contain the item and how many open orders carry each. |
| **Go to links** | Open the Item or its stock ledger from an expanded row. |
| **Click-through from cards** | Each summary card scrolls to its tile and flashes it. |
| **Data issues checklist** | Click the Data issues card to open it in place. |

---

## 6. Which view answers which question

| View | Best for | Counts stock between orders? | Unit |
|---|---|---|---|
| **Stockout Risk** | "Which items will run out soon?" | n/a (uses daily use) | Units |
| **Category-wise** | "Which product groups are in trouble?" | n/a | Item counts or dollars |
| **Restock List** | "What should I buy, and how much?" | n/a | Units, rods, inches |
| **Buildable Orders** | "Which orders can we build today?" | **Yes**, oldest first; only Ready orders take stock | Orders and dollars |
| **Material Not In Stock - Total** | "Which materials are short for all active orders together?" | No (adds all orders, compares once) | Units |
| **Active Orders - Not In Stock** | "Which single order alone is short?" | **No**, each order sees full stock | Units |
| **Tube / Acrylic** | "How long will tube / acrylic last?" | n/a | Feet / inches |

A simple rule: **Stockout Risk and Restock List look at the pace of use. The three order views look at what open orders need.**

---

## 7. What to do — playbooks

**A Critical item**
1. Open **View forecast detail** and expand the item.
2. Check **Confidence**. If Low, the daily use may be unreliable.
3. Check **Last Received** in the Restock List to see if stock arrived recently.
4. Confirm with the buyer that nothing is already on the way.
5. Order the **Suggested Qty** (or more if a big order is coming).

**An item with negative Available**
1. Open orders already need more than we hold. It is Critical whatever the cover.
2. Check how much is **BOM committed** and whether the BOM is reliable.
3. In Material Not In Stock, tick the item to see which orders are affected.
4. Buy at least the shortfall and tell Customer Support if ship dates will slip.

**A Short order**
1. In Buildable Orders, read the limiting components.
2. Check each in the Restock List.
3. Buy the missing items. Until then the order stays Short.

**A stockout warning (Slack or the page)**
1. Open the item on the Stockout Risk tile.
2. Note the Stock-out date and Revenue exposed.
3. Decide whether to expedite. Remember the page does not know supplier lead times.

**An unreliable BOM**
1. Open the item's detail and look at the BOM ratio.
2. Ask Engineering to correct the Lyfe BOM (components, quantities, scrap).
3. Until fixed, committed numbers for that component are lower than reality. Add a safety margin yourself.

**Data issues**
1. Click the Data issues card.
2. Open each line marked in amber or red and use its "Where to fix" note.
3. Re-check after a refresh.

**A spike (trend arrow shows "up")**
1. Look at the arrow: how far above the 90-day average?
2. Ask whether it is a one-off (large order) or lasting.
3. If lasting, raise the target cover or buy extra. The forecast will take time to catch up.

---

## 8. Slack alerts

The page can send a **Stock cover alert** to Slack. It is **off by default**. An administrator switches it on in **Stock Forecast Settings** ("Send Risk Alerts to Slack").

**When it is sent:** once a day, right after the nightly update (see section 9). It needs at least two days of saved data. An item is included if it is **Critical or High today** and one of these is true:
- it moved to a **worse level** than yesterday (for example High to Critical);
- it is **new** (it had no row yesterday); or
- it has stayed **Critical for 7 days in a row** (a reminder, marked "still critical"; it repeats at 14 days, 21 days and so on).

**What it says:** one grouped message listing, for each item, name and code, the level, days of stock and units on hand. Most severe first, then fewest days. **At most 15 items** per message, then "+N more".

A stocked-out item is simply Critical. There is no separate rule for zero stock.

**Daily Founder Briefing.** The morning briefing also has stock sections, sent once a day when the briefing is switched on:
- **Projected Stockouts:** items due to run out within the briefing's look-ahead (default 7 days, so effectively Critical items). Top 10, with name, code, days left and units, risk, and the suggested action. It also shows how the count changed since yesterday.
- **Tube & Acrylic Stock:** tube and acrylic rows that are Medium or worse, with feet or inches on hand and use per day.

The briefing list uses the 7-day look-ahead, while the page uses 30 days. The Critical items are the same.

---

## 9. The nightly update and forecast accuracy

**Nightly update (snapshot).** Every night at about 1:15 AM the system saves a copy of every item's forecast: on hand, daily use, risk, available, stock-out date, suggested quantity, confidence. This copy is used for:
- the alerts and history;
- the Stockout Risk and Category tiles, if an administrator turns on "Stockout Tile Reads Snapshot" (off by default; faster page);
- the accuracy check.

If the nightly update is missing or older than a day, Stock Data Quality shows a red line. A custom Trailing days is always calculated live.

**Forecast accuracy (weekly self-check).** Every Monday the system takes the saved forecast from about a week ago and compares what it predicted for that week with what was actually issued. It reports:
- **Error %** — how far off the forecast was overall. Lower is better.
- **Bias %** — whether it was consistently too high (positive) or too low (negative).

A **high error** means daily use is jumpy and the forecast is a rough guide only. A **bias beyond +/-30%** raises a warning in Stock Data Quality. Use the numbers to decide whether to switch on the backlog-aware suggestion.

Each Monday the system also tags items **A, B or C** by how much of the volume they make up (A = the fast movers that make up the first 80%). This is only used to report accuracy by group.

**Who can see Data Quality:** by default, everyone who can open this page. An administrator can restrict it to the Administrator user only in Stock Forecast Settings, which also hides the Data issues card.

---

## 10. Limits and things to keep in mind

- **History is short.** Stock history starts in May 2026. There is no seasonality.
- **No lead times and no supplier or PO data.** Incoming stock appears only when it is received. There is no reorder point, safety stock or "order by" date.
- **The forecast is recent use.** It assumes the next weeks look like the last 30 days.
- **BOM numbers are estimates.** The Lyfe BOM has historically explained only part of what the factory actually issues. Unreliable BOMs are ignored.
- **Stock is not shared out between orders** in the two Not In Stock tiles. Use Buildable Orders for a first-come view.
- **First-come allocation.** Buildable Orders goes by order date, not promised dispatch date, and is not an optimiser.
- **Acrylic is in inches.** Stick counts are not shown, and short classes look conservative.
- **Tube feet can be overstated** where pre-cut pieces were booked without a matching rod issue. A monthly physical count corrects this. Offcut waste is not modelled for tube.
- **Items that were never used** do not appear, even if open orders need more than the stock.
- **Dead stock can be seasonal.** With 5 months of history, treat the dead list as a review list.
- **Revenue exposed is not blocked.** It is a risk figure. Use Buildable Orders for the real blocked value.

---

## 11. FAQ

**Why is an item missing from Stockout Risk?**
It has 30 or more days of cover, had no use in the window, is tube or acrylic (own tiles), or is set to Make to Order / Do Not Stock.

**Why does Available differ from Stock on Hand?**
Available subtracts what open orders have already promised.

**Why is an item Critical when it has 25 days of cover?**
Its Available is below zero: open orders already need more than we have.

**Why is the suggested quantity bigger than I expected?**
It restores the target cover (default 30 days) and is rounded up to the Minimum Order Qty.

**Does the page know what I have already ordered?**
No. It sees stock only when received. Check Last Received and ask the buyer.

**Why does a material appear in the Total tile but not the order tile?**
The Total tile adds all orders together. The order tile checks each order alone against full stock. See the example in section 4.10.

**Why does a Short order exist when stock looks fine?**
An older Ready order has already taken the stock.

**Is rejected material counted?**
Not in on-hand stock or in use. (The old Not In Stock tiles used to count it; they now exclude it too.)

**Why is the trend arrow missing?**
The change is under 25%, the window is 90 days or more, or there is no 90-day history.

**Can I change the 7 / 15 / 30 day levels?**
Yes, an administrator can, in the briefing settings. The page and the nightly update use the same levels.

**How fresh are the numbers?**
Most tiles are live when you load or refresh. The "Last updated" time is at the top.

---

Last verified against the live implementation: 2026-10
