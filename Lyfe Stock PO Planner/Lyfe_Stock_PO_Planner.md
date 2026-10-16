# Lyfe Stock PO Planner — End User Guide

## What is this screen for?

It tells you **which items to order from the factory, and how many**, so we don't run out of stock. It is based on what was actually used in recent days, not guesses.

When you open it, you see the list **"Forecast: needs ordering"**: only items that were used recently and need an order.

---

## 1. The filter bar (top of the screen)

| Filter | What it does | Default |
|---|---|---|
| **Show** | Chooses the list you want to see (see section 2) | Forecast: needs ordering |
| **Days of usage** | How many past days to look at when checking what was used | 30 |
| **Min orders (trend)** | An item used in at least this many orders is called "frequently ordered" | 3 |
| **Cover days** | How many days of stock you want to hold. Leave blank to use the company setting | blank |
| **Item Type** | Child Items (specific SKUs) or Parent Items | Child Items |
| **Item Group** | Show one group only | all |
| **Item Code** | Show one item only | all |

You can also use the **search box** above the table, and click a **status count** (for example "Order now") to show only those rows.

---

## 2. The lists you can choose in "Show"

| List | What it shows |
|---|---|
| **Forecast: needs ordering** | Used items that need ordering. **This is the main list.** |
| **Trend: frequently ordered** | Every item used in at least "Min orders" orders, even if stock is fine. Most-ordered first |
| **Forecast: all used items** | All items used in the period, whether they need an order or not |
| **Minimum / Target: needs ordering** | Items below the fixed Minimum stock set on the item |
| **Always in stock (all)** | All items that have a Minimum set |
| **All items** | Every item (slow, large list) |

---

## 3. Every column explained

### Status
Tells you what to do about this item.

| Status | Meaning | Action |
|---|---|---|
| 🔴 **Order now** | We need more, and open POs don't cover it yet | Order it |
| 🟠 **On order** | We need more, but POs already placed will cover it | Wait, nothing to order |
| 🟢 **OK** | No order needed | Nothing |
| ⚪ **No minimum set** | No Minimum stock is set on this item (only in the Minimum/Target lists) | Set a Minimum if it must always be in stock |

### Item
The item code on the first line, then the item name and item group in small text.
Items that differ only by **finish** (for example Polished Brass and Satin Brass of the same size) are shown together as **one product row**. See "Finishes" below.

### On Hand
How much is physically in stock right now. Stock in the Rejected Material warehouse is **not** counted.

### Committed
Quantity already **promised to open customer orders** that haven't been issued from stock yet. This stock is spoken for.

> Example: On Hand 100, Committed 30. Only 70 is really free to use.

### On Order
Quantity on **Lyfe POs we've already placed** that haven't arrived yet. Closed and Cancelled POs are not counted.

### Used
How much of this item was **used in the "Days of usage" period** (last 30 days by default).

### Orders
The number of **different customer orders** that used this item in the same period.

> Example: 500 units used but in only 2 orders → Used 500, Orders 2.
> 50 units used across 20 orders → Used 50, Orders 20. This item is needed all the time, even though the quantity is small.

### Trend
Shows whether the item is being ordered **more or less often** than in the period before. It compares the **number of orders**, not the quantity.

| Trend | Meaning | When it appears |
|---|---|---|
| ↑ **Rising** | Used in more orders than before | Orders up by at least 25% **and** by at least 2 orders |
| → **Steady** | About the same | Change is smaller than that |
| ↓ **Falling** | Used in fewer orders than before | Orders down by at least 25% **and** by at least 2 orders |
| ★ **New** | Wasn't used at all before, but is now | 0 orders before, 1 or more now |

**Examples (30-day periods):**
- 4 orders before → 8 now = **↑ Rising**
- 10 orders before → 6 now = **↓ Falling**
- 2 orders before → 3 now = **→ Steady** (only 1 more order, too small to count)
- 0 orders before → 5 now = **★ New**

Hover your mouse over the Trend to see how many orders there were in the earlier period.

### Days Left
How many days the stock on hand will last at the current rate of use. It is colour-coded:

| Colour | Meaning |
|---|---|
| 🔴 Red | Critical — about to run out (or open orders already need more than we have) |
| 🟠 Orange | High / Medium risk |
| 🟢 Green | Low risk — plenty of days left |

> Example: On Hand 60, used about 3 per day → **20 days left**.

If the item hasn't been used, Days Left is blank.

### Minimum and Target
*(Only in the Minimum / Target lists.)*
- **Minimum** — the lowest stock we're willing to hold. Below this, order.
- **Target** — the level we want to bring stock back up to when we order.

### Order Qty
**How many to order.** It is filled in for you, and you can change it. Typing a quantity ticks the row automatically.

In the Forecast lists the suggestion is:

1. Enough to cover your **Cover days** of expected use, **minus** what is free in stock, rounded up to the item's minimum order quantity.
2. For frequently ordered items, the planner also works out how many orders to expect and the usual quantity per order, and uses whichever number is **bigger**.
3. **Minus** what is already On Order.

> Example: uses 5 per day, Cover days 30 → need 150. Free stock is 40, already on order 50. Order Qty = 150 − 40 − 50 = **60**.

### Why
Tells you *why* the item is on the list:
- **Low stock** — stock is running low based on usage.
- **Frequent orders** — it is used in many orders and stock may not cover the orders we expect.
- Both can show together.

### Finishes
Small coloured tags showing each finish of the product (like PBU, SBU) and its stock. Red or orange tags mean that finish needs an order or is already on order.
Click a product row to open a window with **each finish separately**, where you can tick items and change the Order Qty for each one.

---

## 4. Buttons at the top

| Button | What it does |
|---|---|
| **Create / Update Lyfe PO** | Makes a new Draft Lyfe PO from the ticked rows, or adds them to an existing Draft or Ordered PO. Items already on that PO are left unchanged and reported to you |
| **Select all 'Order now'** | Ticks every red row in one click |
| **Column filters** | Shows a filter box under each column heading. For number columns you can type things like `>5` or `<=100` |
| **Export Excel** | Downloads the list to Excel |

You can also click any column heading to sort by it.

---

## 5. How to use it, step by step

1. Open the planner. The main list is already showing.
2. Look at the 🔴 **Order now** rows first.
3. Check **Days Left**, **Trend** and **Why** to understand each one.
4. Change **Order Qty** if you know better (for example, a big order is coming).
5. Tick the rows you want. Or click **Select all 'Order now'**.
6. Click **Create / Update Lyfe PO**.

---

## 6. Example walkthrough

**Product: 10L Handle, Polished Brass**

| Column | Value |
|---|---|
| On Hand | 40 |
| Committed | 10 |
| On Order | 0 |
| Used (30 days) | 120 |
| Orders | 12 |
| Trend | ↑ Rising (it was 6 orders before) |
| Days Left | 10 (shown in red) |
| Why | Low stock |
| Status | 🔴 Order now |
| Order Qty | 110 |

**What this tells you:** we use about 4 a day. Only 30 are free (40 − 10 committed), and sales are growing. Stock is low and demand is rising, so order now.

---

## 7. Good to know

- Numbers are live and refresh when you change a filter.
- Suggestions come from **past usage**. They don't know about future promotions, so adjust Order Qty when needed.
- Only items used in the chosen period appear in the Forecast lists.
- After a PO is placed, its quantity shows under **On Order** and the row changes from "Order now" to "On order", so the same item isn't ordered twice.
- Once the PO arrives, use **Add to Stock** on the Lyfe PO to book it into the warehouse.
