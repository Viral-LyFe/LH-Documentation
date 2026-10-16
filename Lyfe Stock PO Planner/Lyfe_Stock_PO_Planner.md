# Lyfe Stock PO Planner

## What is this?

One screen that tells you **what to order, how much, and how urgently**, so we don't run out of stock and don't over-order.

---

## What you will see

### 1. Top summary boxes

| Box | What it means |
|---|---|
| **Items to Reorder** | How many products are running low and need a new order |
| **Running Out Soon** | Products that will sell out *before* a new order can arrive |
| **Open PO Value** | Total money on orders we've placed but not yet received |
| **Late POs** | Orders that should have arrived by now but haven't |
| **Suggested Order Value** | What it would cost to order everything the planner recommends |

### 2. The planner list (main part of the screen)

Each row is one product.

| Column | What it means |
|---|---|
| **Product** | The item name / SKU |
| **Supplier** | Who we buy it from |
| **In Stock** | What we physically have right now |
| **Already Promised** | Stock set aside for customer orders |
| **Already Ordered** | Stock we've ordered from the supplier but not received yet |
| **Sells Per Day** | Average units sold per day recently |
| **Days Left** | How many days our available stock will last |
| **Supplier Lead Time** | How many days the supplier takes to deliver |
| **Suggested Order Qty** | How many we should order now |
| **Status** | Colour tells you how urgent it is (see below) |

### 3. Status colours

| Colour | Meaning | What to do |
|---|---|---|
| 🔴 Red | Will run out before a new order can arrive | Order **today** |
| 🟠 Orange | Below our minimum stock level | Order **this week** |
| 🟡 Yellow | Fine for now, but getting low | Keep an eye on it |
| 🟢 Green | Plenty of stock | Nothing to do |

### 4. Orders already placed

A short list of open orders: who, what, how many we've received so far, due date, and how many days late.

---

## Example

**Product: Blue Water Bottle**

- In stock: **100**
- Already promised to customers: **20**
- Sells about **10 per day**
- Supplier takes **30 days** to deliver

**Days Left:** (100 − 20) ÷ 10 = **8 days**

The supplier needs 30 days, but we only have 8 days of stock. We'll run out for about 3 weeks. Status: 🔴 **Red**, order today.

**How many to order?** The planner covers the 30 days of delivery plus an extra 45-day buffer = 75 days × 10 per day = 750. Minus what we have left (80) = about **670 units**. That's the **Suggested Order Qty**.

---

## How to use it

1. Pick your **warehouse** and **supplier** at the top (optional).
2. Look at the 🔴 and 🟠 rows first.
3. Check the **Suggested Order Qty**. You can change the number if you know better (for example, a big sale is coming).
4. Tick the products you want to order.
5. Click **Create Purchase Order**. One order is made per supplier.

---

## Good to know

- The numbers update every time you open the screen.
- Suggestions are based on recent sales. They don't know about future promotions or seasonal peaks, so adjust the quantity yourself when needed.
- Products that are disabled or not stocked are not shown.

---

## Settings you can change

| Setting | What it does | Usual value |
|---|---|---|
| **Look-back days** | How many past days of sales to use when working out "Sells Per Day" | 30 |
| **Buffer days** | Extra days of stock you want to hold on top of the delivery time | 45 |
