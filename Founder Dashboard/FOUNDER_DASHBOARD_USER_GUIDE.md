# Founder Dashboard — What It Does (Plain English Guide)

*Last updated: 14 Sep 2026*

This page is for anyone who uses the Founder Dashboard and wants to understand what it shows and how to use it — no technical background needed.

---

## What is the Founder Dashboard?

One screen that answers five questions a founder actually asks about the business:

1. **Are we making money?**
2. **What's selling — and what's dying?**
3. **Where am I bleeding?**
4. **Is the team keeping up?**
5. **What's coming?**

Every number on the page comes straight from the same records the team already works in every day — orders, quotations, tasks, stock. Nothing is a separate spreadsheet someone updates by hand.

---

## The five sections, in plain terms

### 1. Are we making money?
Four boxes at the top:
- **Revenue** — total sales so far this month, compared to last month and to target.
- **Gross Margin** — how much of that revenue is actually profit after costs.
- **Cash In** — money that's actually landed in the bank, not just invoiced.
- **Custom Revenue** — how much comes from custom (quote-built) orders vs. off-the-shelf orders, and which is more profitable.

Below those, a **Month-on-Month** panel (added 2026-09-03) trends revenue and margin over the trailing 6 or 12 months (switchable), always showing whole calendar months side by side — never the whole page's Today/MTD/90-day filter, since that would defeat the point of a month-by-month trend. The current, still-in-progress month is called out separately (labelled "MTD, in progress") rather than plotted next to completed months, so a business doing fine two days into September never looks like it's collapsing.

### 2. What's selling — and what's dying?
- **Best Sellers** — top products, and you can sort by revenue, units sold, or profit margin.
- **Slow Movers** — products tying up money in stock while barely selling.
- **Category Performance** — the main screen now shows just your top few categories and whether each is trending up or down since last month. Click **"view detail →"** for the full picture: month-by-month figures per category, revenue mix, what's driving the change, and which categories are rising, declining, or brand new — all still there, just one click away instead of loading automatically.
- **Where the Revenue Is** — revenue and profit broken down by country and by state, so you can see geographically where the business's sales actually come from, not just who's buying.

### 3. Where am I bleeding?
- **Money at Risk** — a dollar total of orders currently stuck, on hold, possibly lost in transit, or waiting on a customer service decision. Always live, not a monthly total.
- **Stockout Risk** — items about to run out, and whether that's actually about to hold up real orders.
- **Material Usage** — what's actually being used in production over the last 30 days.

### 4. Is the team keeping up?
- **Board Health** — each internal team's workload (Customer Service, Factory, Engineering, etc.) shown side by side, not blended into one number.
- **Team Task Risk** — internal tasks at risk of missing their deadline.
- **Customer Concentration** — how dependent the business is on a handful of big customers, and whether growth is coming from new customers or repeat ones. The main card is a quick read; click **"view detail →"** for the full breakdown (concentration chart, per-category revenue, customer × category table).

### 5. What's coming?
- **Look Ahead** — a forecast of expected sales over the next 30 days, based on past patterns.
- **Production** — how many orders are currently stuck in the factory pipeline longer than expected.

There's also a **"Who's connected right now?"** panel, visible only to Super Admins — a security check on who currently has system access.

---

## The filters at the top

Four controls above the tiles affect the whole page at once today:
- **Period** — Today, Month to date, Last full month, Last 90 days, or a custom date range.
- **Channel** — one sales channel at a time, or all combined.
- **Order Type** — Standard catalog orders vs. custom quote-built orders.
- **Category** — one product category at a time, or all.

Change any of these and the whole page re-loads with the new filter applied.

---

## What's changed

The dashboard now has **every tile working on its own**, instead of the whole page reloading together:

- **Refresh one tile at a time.** Click Refresh on just "Best Sellers" and only that box updates — nothing else on the page reloads or resets.
- **Filter one tile at a time.** For example, set "Category Performance" to only show the "Storage" category, while every other tile keeps showing all categories. Your choice on one tile never changes another tile.
- **A tile's own filter always wins.** If you've set a specific filter on a tile, changing the page-wide filters at the top won't override it — your tile keeps its own setting until you clear it.
- **If one tile has a problem loading, the rest of the page keeps working.** A broken tile shows its own "try again" message while every other tile stays fully usable.
- **Not every tile gets filters** — only where it actually makes sense for that tile's data. For example, a live security/access panel doesn't get a "sales channel" filter, since that has nothing to do with what it shows.
- **Two sections are collapsed by default** (added 2026-09-03): Category Performance and Customer Concentration's full breakdown now load as a short summary with a **"view detail →"** link, instead of pre-loading every chart and table on first screen. Nothing was removed — click through for the same detail as before.

---

## Where to go for more detail

- **This page** — plain-English overview, for anyone using the dashboard.
- **Technical reference** (for developers) — `FOUNDER_DASHBOARD_REFERENCE.md`, in the same folder — covers exactly how each number is calculated, which system functions back each tile, and settings admins can tune.
