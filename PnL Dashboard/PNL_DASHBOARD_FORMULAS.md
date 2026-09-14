# PnL Dashboard — Formulas & Logic Reference

Source: `lh/lyfe_hardware/page/pnl_dashboard/pnl_dashboard.py` (all whitelisted endpoints).
This is the single place documenting **what math backs every tile** on this dashboard.
If you change a formula in the code, update it here in the same commit.

---

## 1. Shared building blocks (used by almost every tile)

### 1.1 Which orders count at all — `_build_where()`

Every tile queries `tabLyfe Order` and always excludes:

- `status IN ('Cancelled', 'Merged', 'Split')`
- Orders where `merged_into` is set (already absorbed into another order)

Optional filters (from the dashboard's filter bar), all AND'ed in: `from_date`/`to_date`
(on `order_date`), `order_status` (workflow_status, comma-separated = OR), `order_source`
(with grouping — see 1.4), `order_type` (Standard/Custom), `customer`.

**Why:** Cancelled = no revenue. Merged/Split = the money already lives on a sibling
order; counting both would double-count revenue.

### 1.2 Revenue — always net of Shopify refunds

```
revenue = total_amount − shopify_refund_amount
```

`total_amount` already includes `ss_shipping_amount` baked in — it is never added a
second time anywhere.

### 1.3 The core P&L formula — `_calc()`

```
total_expenses = mfg_cost + custom_charges + additional_charges
                + shipping_charges + shipping_charges_us
                + payment_processing_fee + tax

gross_profit   = revenue − total_expenses

profit_margin  = gross_profit / revenue × 100   (0 if revenue is 0)
```

Where each expense component comes from (all summed across the filtered orders):

| Component | Field(s) | Notes |
|---|---|---|
| `mfg_cost` | `cost_of_goods` | Per-order COGS rollup (see CLAUDE.md's COGS section) |
| `custom_charges` | `custom_charges + custom_duty_changes_us_tram + Σ(reshipment_cogs + shipment_cost)` | Reshipment cost is pulled from **non-cancelled** `Lyfe Order Reshipment` child rows and folded in here — it is not a separate line anywhere |
| `additional_charges` | `additional_charges` | As stored |
| `shipping_charges` / `shipping_charges_us` | as stored | Domestic + US-leg shipping, kept as two columns internally but usually shown combined |
| `payment_processing_fee` | as stored | Only used in `get_summary`/order-detail aggregation, not in every endpoint |
| `tax` | `ss_tax_amount` | ShipStation tax amount |

`direct_cost` (used by the category tile) = `custom_charges + additional_charges + shipping + shipping_us` (mfg_cost and tax excluded).

### 1.4 Source filter grouping

The dashboard's "Source" filter is a label, not a raw DB value:

- `"Shopify"` → matches `order_source IN ('Shopify', 'Shopify - W', 'Shopify - D')`
- `"Etsy"` → matches `order_source = 'Etsy'`
- anything else → matched literally

### 1.5 Fee-row exclusion (category/item-group tiles only)

Line items whose `item_name` matches this pattern are **not real products** and are
excluded whenever revenue is computed at the line-item level (category breakdown, MoM
by category, margin trend by category, item-group summaries):

```
custom.?fee | customs fee | customization fee | shipping.?(charges?|&.?handling)
| advance( payment)? | overdue( payment)? | additional payment | installment
| return label fee
```

**Why:** ShipStation doesn't flag these with its own `adjustment` field, and some
inherit a real product's `item_group`, which would otherwise leak revenue into the
wrong category or let a fee row hijack an order's whole COGS/shipping attribution.

---

## 2. Tile-by-tile

### 2.1 Summary KPIs (`get_summary`)

Runs the standard `_calc()` aggregation three times: current period, same filters
shifted back **1 month**, and shifted back **1 year** — for MoM/YoY deltas on the KPI
cards. No extra math beyond §1.

### 2.2 Monthly P&L Trend (`get_monthly_trend`)

Same `_calc()` per bucket, bucketed by day/week/month/year. Granularity auto-picked
from the date span (`_pick_granularity`):

| Span | Granularity |
|---|---|
| ≤ 31 days | daily |
| ≤ 120 days | weekly (ISO week, keyed to Monday) |
| ≤ 730 days | monthly |
| > 730 days | yearly |

Can be overridden by an explicit `granularity` filter.

### 2.3 Month-on-Month Comparison (`get_mom_summary`)

**Deliberately ignores the dashboard's global date filter** — takes only `months`
(trailing window, default 12) or a specific `year` (Jan–Dec, or Jan–today if current
year; `year` replaces `months` entirely, never combined).

- No category: same order-level `_calc()`, monthly buckets.
- With a category: revenue switches to **line-item revenue** (`quantity × unit_price`
  from `ShipStation Order Item`, fee rows excluded) for that item_group; expenses
  (COGS/shipping/etc.) are still attributed at the **order level** via a
  single-item_group-per-order simplification (§2.9) — so a multi-category order's full
  cost lands on whichever category wins `MIN(item_group)`. This is a known, accepted
  tradeoff, consistent with how Category Breakdown does it.

```
avg_order_value = revenue / order_count
```

### 2.4 Category (Item Group) Breakdown (`get_category_breakdown`)

Two separate queries, joined in Python:

- **Revenue** — pure line-item: `Σ(quantity × unit_price)` grouped by `item_group`,
  fee rows excluded. This is the only tile that splits **revenue** correctly across a
  multi-category order.
- **Expenses** (COGS, custom, additional, shipping, shipping_us, tax, refunds) — order
  level, attributed to **one** item_group per order via `_CATEGORY_SUBQ`
  (`MIN(item_group)` of that order's real product rows). A multi-category order's
  entire expense total lands on whichever group MIN() picks — not split.

Then per category:

```
revenue        = line_revenue − shopify_refund_amount(attributed)
total_expenses, gross_profit, profit_margin = _calc(...)   (§1.3)

revenue_share  = revenue / total_revenue_across_all_categories × 100
avg_order_value = revenue / order_count
shipping_pct   = (shipping + shipping_us) / revenue × 100
cog_pct        = mfg_cost / revenue × 100
```

Sorted by `gross_profit` descending.

### 2.5 Profit Margin Trend (`get_margin_trend`)

Same math as §2.2/§2.4 combined — computes `_calc()` per period bucket (any
granularity) and returns **only** `profit_margin`. Fully independent filter/date-range
state from Monthly P&L Trend (separate Item Group filter + granularity + optional
year-replaces-range, same pattern as MoM).

### 2.6 Trending Categories — monthly matrix (`get_category_monthly_trend`)

Per (item_group, month) bucket: `order_count`, `qty` (Σquantity), `revenue`
(Σquantity×unit_price). No cost/margin here — pure revenue/qty history, always the
trailing `months` window (ignores the dashboard's own date filter, same reasoning as
MoM). Every month bucket is always present (zero-filled) so a category with a gap
month doesn't visually shift.

### 2.7 Per-Order Detail table (`get_orders_detail` / `get_category_orders`)

Row-by-row version of `_calc()` — same formula, one row per order (top 500 by date).
`custom` column = `custom_charges + additional_charges` combined; `shipping` column =
`shipping + shipping_us` combined.

### 2.8 Standard vs Custom (`get_order_type_breakdown`)

Groups by `order_type` (defaults missing values to `"Standard"`), runs `_calc()` per
group, plus:

```
mean_aov   = AVG(total_amount)               (SQL-side average, before refunds)
median_aov = median(total_amount) in Python  (MariaDB has no PERCENTILE_CONT)
gp_pct     = gross_profit / revenue × 100
cm_pct     = profit_margin (same value, aliased for the UI)
```

### 2.9 Cost Waterfall (`get_waterfall`)

```
gross_rev     = Σ total_amount
refunds       = Σ shopify_refund_amount            ← real deduction
discounts     = Σ ss_discount_amount               ← DISPLAY ONLY, does not reduce anything
cogs          = Σ cost_of_goods
shipping      = Σ (shipping_charges + shipping_charges_us)
other_costs   = Σ (custom_charges + custom_duty_changes_us_tram + reshipment_cost + additional_charges)
tax           = Σ ss_tax_amount

contribution  = gross_rev − refunds − cogs − shipping − other_costs − tax
```

**Discounts are shown on the waterfall for context but never subtracted** — they're
already baked into `total_amount`, so subtracting them again would double-count.

### 2.10 Contribution Margin by Source (`get_source_margin`)

```
revenue     = Σ(total_amount − shopify_refund_amount)   per order_source
cogs        = Σ cost_of_goods
direct_cost = Σ(custom_charges + additional_charges + shipping + shipping_us)
              (reshipment cost folded into custom_charges, same as everywhere else)

contribution = revenue − cogs − direct_cost
margin_pct   = contribution / revenue × 100
```

Note: **tax is not subtracted here** (unlike `_calc()`'s total_expenses) — this is a
simpler contribution-margin view, not full gross profit. Sources with `revenue ≤ 0`
are dropped. Sorted by margin descending.

### 2.11 Samples Sent (`get_samples_sent`)

Sums COGS for every order line item whose `item_name` contains `"sample"`
(case-insensitive), across all matching order+item rows in the period:

```
per-line cogs = Item.cost_of_goods_sold   if non-zero
                else the order's own cost_of_goods   (fallback)
total_sample_cogs = Σ(per-line cogs)
```

### 2.12 Top Customer (`get_top_customer`)

```
revenue(customer) = Σ(total_amount − shopify_refund_amount)  grouped by customer
top = highest revenue(customer) in the period
pct = top.revenue / total_period_revenue × 100
```

### 2.13 Top-5 Customer Concentration (`get_top5_concentration`)

Same revenue definition as §2.12, but sums the **top 5** customers:

```
pct = Σ(top 5 customers' revenue) / total_period_revenue × 100
```

### 2.14 Top Customer × Item Group breakdown (`get_top_customer_item_group_breakdown`)

Takes the customer from §2.12, then splits **only that customer's** line-item revenue
by item_group (same line-item method as §2.4's revenue query — quantity × unit_price,
fee rows excluded):

```
pct(group) = group_revenue / that_customer's_total_revenue × 100
```

### 2.15 Revenue Concentration Curve (`get_revenue_concentration_curve`)

Ranks all customers by revenue (top 50), builds a running cumulative % of total
revenue, and reports cut points:

```
cumulative[i] = Σ(revenue of rank 1..i) / total_revenue × 100
cut_points = cumulative % captured by top 1 / top 3 / top 5 / top 10 / top 20 customers
```

This is the Pareto ("how dependent are we on a few customers") curve.

### 2.16 Item Group Revenue Summary (`get_item_group_revenue_summary`)

Same line-item revenue method as §2.4's revenue side, plus distinct counts:

```
pct = group_revenue / total_revenue × 100
```

Also returns `customer_count` (distinct customers) and `order_count` (distinct
orders) per group — the one thing this endpoint adds beyond Category Breakdown.

### 2.17 Customer × Item Group Matrix (`get_customer_item_group_matrix`)

Top-N customers (default 15, ranked by their own line-item revenue — not
`total_amount`, to avoid ranking in a customer with $0 in every visible cell) ×
item_group grid of line-item revenue. Everyone outside the top N is folded into one
"Remaining Customers" row per item_group so column totals still reconcile to the
period's real total.

### 2.18 Repeat Customers (`get_repeat_customers` / `..._list`)

Customers are grouped by **normalized email** (`LOWER(TRIM(customer_email))`), not the
`customer` Link field (often unresolved for Shopify/ShipStation orders). "Repeat" =
`order_count > 1` within the filtered period.

```
pct_customers = repeat_customer_count / total_customer_count × 100
pct_revenue   = repeat_customers'_revenue / total_revenue × 100
```

`get_repeat_customers_list` returns the actual repeat customers (name, order count,
total value) for drill-down; email is stripped unless `include_email=True` **and**
caller is Super Admin/System Manager (customer_email is a permission-gated field —
raw SQL can't inherit the ORM's permlevel, so it's re-checked explicitly here).

### 2.19 Standard vs Custom within one Category (`get_category_order_type_breakdown`)

Same as §2.8, but scoped to a single `item_group` (via the same `_CATEGORY_SUBQ`
single-item_group-per-order join as §2.4). Adds:

```
revenue_share = order_type's revenue / category's total revenue × 100
```

---

## 3. Things that look like formulas but aren't

- **Discounts** never reduce revenue or profit anywhere on this dashboard — they're
  display-only (already inside `total_amount`). Only the Waterfall (§2.9) surfaces the
  number, purely for context.
- **Payment processing fee** is only subtracted in `_calc()`'s general form (used by
  `get_summary`); several other endpoints call `_calc()` without passing it, so it's
  effectively 0 there — check each tile's call to `_calc(...)` for which arguments it
  actually passes.
- **Tax** is subtracted in `_calc()`-based tiles but explicitly **not** subtracted in
  Source Margin (§2.10) — that's a deliberate simpler "contribution margin" view, not
  an oversight.
