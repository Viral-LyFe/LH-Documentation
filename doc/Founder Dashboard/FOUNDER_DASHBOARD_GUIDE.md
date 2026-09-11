# How the Numbers on Your Dashboard Actually Get Calculated

**A plain-language guide to the Founder Dashboard** — what each tile means, where the number comes from, and the exact rule used to color it green, amber, or red.

*Audience: Founder. No technical knowledge required to read this.*

---

## Before you start

The dashboard is organized around five questions a founder actually asks, not around which system the data lives in. Every number you see is pulled live from the same records your team already works in every day — orders, quotations, tasks, stock — nothing here is a separate, hand-kept spreadsheet.

If a tile is clickable on the live dashboard, clicking it opens the exact list of records behind that number, so you can always check a figure against the real orders it's built from.

**Contents**

1. [Are we making money?](#i-are-we-making-money)
2. [What's selling — and what's dying?](#ii-whats-selling--and-whats-dying)
3. [Where am I bleeding?](#iii-where-am-i-bleeding)
4. [Is the team keeping up?](#iv-is-the-team-keeping-up)
5. [What's coming?](#v-whats-coming)
6. [How the filters at the top work](#how-the-filters-at-the-top-work)
7. [If a number ever looks wrong](#if-a-number-ever-looks-wrong)

---

## I. Are we making money?

Four tiles across the top row, covering revenue, margin, cash actually collected, and custom-order profitability.

### Revenue · MTD (Month to date)

**What it is:** Total sales booked so far this calendar month, and how that compares to last month and to your monthly sales target.

**How it's built:**
```
Revenue = Σ (order total − any Shopify refund)
          for every order dated from the 1st of this month through today
```

**Important nuance — "paced" target:** The color you see isn't judged against the *full* month's target — it's judged against a **paced** target: your monthly goal scaled down to "what we should have by today's date." On day 7 of a 31-day month, the bar you're compared to is roughly 7/31 of the full target, not the whole thing. Without this, every tile would show red for the first two or three weeks of every month simply because the month isn't over yet — that was a real problem we fixed.

**Color rule:**
| Status | Condition |
|---|---|
| 🟢 On track | At or above the paced target |
| 🟡 At risk | 80%+ of the paced target |
| 🔴 Off track | Below 80% of the paced target |

**Also shown:** order count, % change vs. last month, and % change vs. the same period last year (that comparison is hidden if last year's number was too small to make a meaningful percentage — a tiny base can turn a normal dollar increase into a misleading "+2000%").

---

### Gross Margin (Last full month)

**What it is:** What share of revenue is left over as profit after direct costs — manufacturing, shipping, custom fees — are subtracted.

**How it's built:**
```
Direct cost = Manufacturing cost + Custom/additional charges + Shipping (India + US leg)
Profit      = Revenue − Direct cost
Margin %    = Profit ÷ Revenue
```

**Why last month, not this month:** This tile deliberately looks at **last month**, not this month so far. Shipping cost is only entered once an order actually ships, so early in a new month almost every order is still missing that cost — which would make margin look artificially high. Waiting for a month to fully close avoids that false read.

**Color rule:**
| Status | Condition |
|---|---|
| 🟢 On track | At or above your margin target (default 20%) |
| 🟡 At risk | At or above 15% |
| 🔴 Off track | Below 15% |

---

### Cash In · MTD (Month to date)

**What it is:** Money that has actually landed — real payments received this month, not invoices that were merely issued.

**How it's built:**
```
Cash In = Σ payment amount, for every payment recorded against a Quotation this month
```

**Note:** This intentionally does **not** use the outstanding-balance figure on invoices — that field has been unreliable in this system. This tile only counts money that a real Shopify charge or bank transfer confirms was received. Same paced-target coloring as Revenue above.

---

### Custom Rev · MTD

**What it is:** Revenue from custom (quote-built) orders specifically, plus how their margin compares to standard, off-the-shelf orders.

**How it's built:** Same revenue/margin math as above, split into two buckets: **Custom** orders (from a Quotation) vs. **Standard** orders.

**Note:** No target/color on this one — it's context for the Revenue and Margin tiles above, showing whether growth is coming from higher-margin custom work or lower-margin standard catalog sales.

---

## II. What's selling — and what's dying?

Three panels showing which products are winning, which categories are trending, and which stock is quietly going stale.

### Best Sellers

**What it is:** Your top products for the selected period, switchable between ranking by revenue, units sold, or margin.

**How it's built:** Grouped by SKU from actual order line items. Cost is split across a SKU's lines in proportion to its share of that order's total revenue — the system doesn't track cost per individual line item, only per order, so this is the fairest available split.

**Note:** Deliberately shows all three rankings side by side because the best-selling item is often **not** the most profitable one — a quick toggle reveals that gap.

### Slow Movers

**What it is:** Products tying up the most inventory value while barely selling — candidates for a clearance or bundle deal.

**How it's built:**
```
Inventory value = stock on hand × cost per unit
                   ranked highest to lowest, for SKUs that sold in the selected period
```

**Note:** Ranked by **dollars tied up**, not by low sale count — a SKU with only 2 units sold but $8,000 of stock sitting on a shelf matters more than one that sold 2 units of a $10 item.

### Trending Categories

**What it is:** Revenue by product category for the selected period, with an arrow showing direction against a fixed 30-day baseline immediately before it — so you can spot a category heating up or cooling down.

**How it's built:**
```
% change = (this period's revenue − prior 30 days' revenue) ÷ prior 30 days' revenue
```

---

## III. Where am I bleeding?

Three panels that turn operational alerts into dollar figures, because "9 alerts" doesn't tell you anything — "$18,000 across 16 orders" does.

### Money at Risk (Live)

**What it is:** The total dollar value currently sitting in orders that are stuck, on hold, possibly lost in transit, or waiting on a Customer Service action — not a period total, but what's true right now.

**The four buckets that add up to the total:**
- **Dispatch breaches** — orders stuck past their factory deadline
- **On hold / delivery exception** — orders currently paused, or flagged as an exception in transit
- **Possible lost / never scanned** — shipments the tracking provider has flagged as exceptioned, expired, or not showing movement
- **Pending CS action** — orders currently waiting on a Customer Service team decision

**Note:** Each bucket's dollar figure is the order value of every order in that bucket (minus any Shopify refund already issued), summed up. This is always live — not filtered by the Period selector at the top of the page — because these are point-in-time problems, not a monthly total.

### Stockout Risk (Live)

**What it is:** Items projected to run out of stock soon, based on how fast they've been selling over the last 30 days — and, for the riskiest ones, whether they're actually blocking real open orders right now.

**How it's built:**
```
Days of cover = current stock ÷ average daily usage (trailing 30 days)
```

**Note:** Each critical item shows a note like "blocks $2,400" if it's currently needed by open orders that haven't shipped yet — distinguishing an item that's technically low but idle from one that's actually about to hold up real customer orders.

### Material Usage · 30d

**What it is:** What's actually being pulled off the shelf and used in production, trailing 30 days — the real physical consumption behind your orders.

**How it's built:**
```
Net qty = material issued to production − material returned
```

**Note:** Comes from the same paperwork the factory floor already fills out (Material Issue for Order) — nothing new was set up to track this, it's simply the first time it's shown on a founder-facing screen.

---

## IV. Is the team keeping up?

Three panels on internal execution — separate from customer-facing production, this is about whether the team's own work is on schedule.

### Board Health (Live)

**What it is:** Each internal team board (CS, Factory, Engineering, etc.) shown side by side with its own open and overdue task count — kept separate on purpose rather than collapsed into a single company-wide number, so you can see exactly which team needs attention.

**How it's built:**
```
Overdue = tasks still open whose due date has already passed, counted per board
```

### PM Task Risk (Live)

**What it is:** Internal team tasks at risk of breaching their service-level target — a separate signal from customer order delays, since this tracks the team's own work commitments.

**How it's built:** Counts of tasks flagged **Critical**, **Escalated**, or **Active** against internal SLA rules, plus a trend line on whether on-time performance is improving or slipping.

### Customer Concentration (Live · trailing 30 days)

**What it is:** How dependent the business currently is on a small handful of accounts, and whether growth is coming from new customers or the same repeat buyers.

**How it's built:**
```
Top-5 share = revenue from your 5 biggest customers ÷ total revenue (trailing 30 days)
```

---

## V. What's coming?

Two panels looking forward — a demand forecast, and the live state of production.

### Look Ahead · Next 30 Days

**What it is:** A forecast of expected sales revenue for the next 30 days, built from historical demand patterns — a directional signal, not a guarantee.

**How it's built:**
```
Projected revenue = forecasted units per category × that category's average selling price
                     (trailing 90 days)
```

**Note:** Also flags how many SKUs are projected to go critical within 7 days, and how many of those have no restock order in progress yet.

### Production (Live)

**What it is:** How many orders are currently stuck inside the factory pipeline longer than expected — scoped specifically to pre-shipping stages, so an order that has already shipped never shows up here as "stuck in production."

**How it's built:**
```
Stuck order = an order that has breached a factory-stage deadline and hasn't been
              resolved for longer than the configured threshold (default 5 days)
```

**Color rule:**
| Status | Condition |
|---|---|
| 🟢 On track | Fewer than 3 stuck orders |
| 🟡 At risk | 3–7 stuck orders |
| 🔴 Off track | 8 or more stuck orders |

---

## How the filters at the top work

Four controls at the top of the dashboard reshape most of what's below them.

- **Period** — Today, Month to date, Last full month, Last 90 days, or a custom range. Changes which orders count toward every panel below (the Sales and Margin tiles at the very top keep their own fixed windows regardless of this setting, for the reasons explained above).
- **Channel** — Restricts everything to one sales channel (e.g. one marketplace or storefront) instead of all of them combined.
- **Order Type** — Standard catalog orders vs. custom quote-built orders, or both.
- **Category** — One product category at a time, or all of them.

---

## If a number ever looks wrong

**Click the tile.** Every tile with a "click for detail" note opens the exact list of orders, payments, or records that were added up to produce that number — the same records your team works with every day, nothing hidden behind it.

**If a tile shows "Data unavailable"** — the underlying system had a hiccup pulling that one section — every other tile on the page keeps working independently, and the issue is logged for the team to check.

---

*Founder Dashboard — Functional Reference · For business use — no technical setup required to read this.*
