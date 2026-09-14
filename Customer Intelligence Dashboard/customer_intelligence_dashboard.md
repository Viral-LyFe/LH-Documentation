# Customer Intelligence Dashboard — Functionality & Calculations

**BRD:** LH-BRD-DEV-012 · **Route:** `/app/customer-intelligence-dashboard` · **Module:** Lyfe Hardware

This document explains what the dashboard shows and how every number on it is calculated.
Each section opens with a **plain-language summary** anyone can follow, then gives the exact
formula/field names for anyone who needs to verify or extend the logic. Every technical detail
below is traceable to a real line of code, not a paraphrase of it.

---

## 0. BRD ask vs. what's delivered

**In plain terms:** the manager's brief (LH-BRD-DEV-012) asked for a specific list of things.
This table goes through it point by point — what was asked, and what actually shipped — so
nothing gets lost between the request and the build.

| # | What the BRD asked for | What's delivered |
|---|---|---|
| 1 | One screen answering who customers are, which are worth most, which need attention — ranked by profit once Manufacturing cost exists, by revenue until then | ✅ Delivered. Founder Overview section, §7.1; margin/provisional-revenue fallback, §5.8 |
| 2 | Category is the primary lens — every metric breaks down category-wise, never a single global number | ✅ Delivered. Category affinity (§5.4) and the Profit by Category × Population tile (§7.1) — nothing on the page collapses category into one flat figure |
| 3 | Two populations (B2B/trade, Retail/homeowner), same segment types, different thresholds per population | ✅ Delivered. §3 (population), §4 (per-population config), §6 (shared 8 segments) |
| 4 | Classification is customer-level for now; orders that don't match are tagged as exceptions, not used to reclassify the customer | ✅ Delivered. §6.3 (order-level exceptions) |
| 5 | All tiles filterable by population, defaulting to both side by side | ✅ Delivered. The B2B / Retail / Both population filter on the filter bar applies across every tile |
| 6 | System suggests B2B/Retail, a named owner confirms — never silently auto-classified | ✅ Delivered. §3.1 (suggestion layers), §3.2 (Confirm button — the one write action on the page) |
| 7 | Identification layer 1 — Quotation history auto-flags likely B2B | ✅ Delivered. §3.1 layer 1 |
| 8 | Identification layer 2 — behavioural signals (bulk orders, cadence, custom share, category spread) raise a "possible B2B" flag | ✅ Delivered. §3.1 layer 2 (three signals, 2-of-3 rule) |
| 9 | Identification layer 3 — manual `customer_type` tag set/corrected by Ops or Sales | ✅ Delivered. §3.1 layer 3, §3.2 |
| 10 | Identification layer 4 — checkout capture (a "business?" field at checkout) | ⏸️ Explicitly deferred by the BRD itself — not part of this build. §10 |
| 11 | Orders, revenue, AOV, first/last order, days since last | ✅ Delivered. §5.1 |
| 12 | Repeat rate + reorder cadence per category | ⚠️ Partially delivered — **cadence** is genuinely computed per category (§5.2, `category_cadence_days`), but **repeat rate** is delivered as a business-wide monthly trend (§7.1), not broken out per category the way cadence is |
| 13 | Standard vs. Custom order mix | ✅ Delivered. §5.3 |
| 14 | Category affinity | ✅ Delivered. §5.4 |
| 15 | Finish affinity (e.g. "always buys Matte Black") | ❌ Not built. No part of the system records which finish was bought — agreed, documented gap, §5.4 and §10 |
| 16 | Custom cost-to-serve: revision count, photo-rejection rate | ✅ Delivered. §5.5 |
| 17 | Bulk/volume flag, kept out of normal AOV math | ✅ Delivered. §5.1 (bulk-aware AOV), §5.6 (bulk flag) |
| 18 | B2B sample-to-order — do samples earn their cost | ✅ Delivered. §5.6 (sample-conversion, B2B-scoped), and the Sample Conversion (B2B) tile, §7.2 |
| 19 | Return count + return rate | ✅ Delivered. §5.7 |
| 20 | Margin/profit per customer, provisional badge until Manufacturing cost is ready | ✅ Delivered. §5.8 |
| 21 | Segment: VIP/high-value (top X% by profit) | ✅ Delivered. §6.2 |
| 22 | Segment: Repeat vs. one-time | ✅ Delivered. §6 |
| 23 | Segment: Project/burst buyer — NOT treated as churned | ✅ Delivered. §6 |
| 24 | Segment: Steady buyer | ✅ Delivered. §6 |
| 25 | Segment: At-risk, cadence-aware, not fixed days — the BRD's own "critical" requirement | ✅ Delivered. §6.1 — the one rule this document calls out explicitly as critical, matching the BRD's own emphasis |
| 26 | Segment: Custom-heavy vs. standard-only | ✅ Delivered. §6 |
| 27 | Segment: New | ✅ Delivered. §6 |
| 28 | Founder tile: profit by category × segment (margin-adjusted) | ✅ Delivered, as category × **population** rather than category × segment — segments are a many-to-one tag set per customer (§6), not a clean single axis a customer sits on, so population (the BRD's other required split) is the matrix's second axis. Segment breakdowns are available per-customer in the Customer List and Profile instead. |
| 29 | Founder tile: repeat rate + LTV trend | ✅ Delivered. §7.1 |
| 30 | Founder tile: new vs. returning revenue | ✅ Delivered. §7.1 |
| 31 | Founder tile: at-risk high-value count + value at risk, split B2B/Retail | ✅ Delivered. §7.1 |
| 32 | Manager tile: customer list, sortable/filterable by every metric | ✅ Delivered. §7.2 |
| 33 | Manager tile: at-risk outreach list feeding CRM priority | ⚠️ Partially delivered — At-risk customers are tagged and visible in the Customer List/Segment Movement, but there is no separate dedicated "at-risk outreach list" tile, and nothing pushes into a CRM yet (no CRM integration exists in this app to feed) |
| 34 | Manager tile: return-rate outliers | ⚠️ Partially delivered — the calculation exists and works (`get_return_outliers`), but it isn't wired into a visible tile on the page yet. §10 |
| 35 | Manager tile: sample-conversion (B2B) | ✅ Delivered. §7.2 |
| 36 | Manager tile: segment movement this month | ✅ Delivered. §7.2, §8 |
| 37 | Operator tile: single-customer profile — full order history, quotations, CS tickets, returns, finish/category affinity, segment, LTV | ⚠️ Mostly delivered. §7.3 — order history ✅, quotations ✅, category affinity ✅, segment ✅, returns ✅ (part of the metric bundle); **CS tickets** ❌ not built (Gorgias integration is separate, in progress — §10); **finish affinity** ❌ not built (§15/§10); **LTV** is delivered as a business-wide trend metric (§7.1), not yet as a single number on this specific customer's own profile card |
| 38 | Every tile drills to the customer list, then the single-customer profile | ✅ Delivered — though the navigation model changed from the BRD's implied tab/drill structure: this page has no tabs at all (§7), so "drilling" is a click on any customer row, which loads the profile in place on the same page rather than navigating to a separate view |
| 39 | All thresholds config, not hardcoded | ✅ Delivered. §4 |
| 40 | RAG (red/amber/green) on at-risk and return-rate tiles | ✅ Delivered for both — at-risk RAG on the Overview tile (§7.1), return-rate RAG on the Customer List (§7.2) |
| 41 | Metrics pull from the Metric Dictionary once Reporting is live | ⏸️ Not applicable yet — no Metric Dictionary/Reporting layer exists in the app today for this to plug into. Tracked as a future dependency, not ignored |
| 42 | Margin views carry a data-quality badge until production cost is complete | ✅ Delivered. §5.8 (the "provisional" badge) |
| 43 | Prerequisite: dedupe Customer records before aggregating | ⚠️ Worked around, not fully solved — true Customer-record merging is still a separate, tracked project (§10); this dashboard doesn't wait on it because it identifies customers by email/name identity instead of the raw Customer record (§2), so duplicate Customer records no longer fragment any metric on this page, even though the duplicates themselves still exist elsewhere in the system |
| 44 | Build revenue-side now; margin degrades to revenue + badge, layers in later | ✅ Delivered. §5.8 |
| 45 | Confirm per-population thresholds at kickoff | ✅ Delivered as an editable config screen rather than a one-time kickoff conversation — every threshold can be changed at any time by an admin, not just agreed once up front. §4 |

**Two access-model changes made after the BRD, at the user's explicit request (not gaps —
deliberate decisions), covered in §7 and §9:**
- The BRD's Founder/Manager/Operator language was originally read as three different tiles for
  three different audiences with a tab/tier switcher between them. The user clarified this
  dashboard has one real audience — the Founder plus a small set of other authorized people —
  so the tier switcher was removed; everything now renders on one continuous page instead.
- Page access is granted to **System Manager, Super Admin, and Customer Service** (see §9 for the
  current, corrected access model — this is narrower than the BRD's original four-role ask, but
  not as narrow as "System Manager/Super Admin only").

---

## 1. What problem this dashboard answers

**In plain terms:** this is one screen that tells the business who its customers are, which
ones make the most money, and which ones are drifting away and need a phone call — before it's
too late to save them.

**Technical detail:** ranked by profit once Manufacturing cost data is reliable, by revenue
until then. Every metric is broken down by **category** first (a business-unit lens), and by
**population** (B2B/trade vs. Retail/homeowner) second — never shown as one flattened global
number.

---

## 2. How a customer is identified

**In plain terms:** the same customer might show up under slightly different records in the
system (a Shopify order, an Etsy order, a manual order) — so instead of trusting one shaky
"Customer" record, the dashboard recognizes people mainly by their **email address**. And for the
roughly 1-in-10 customers whose orders never captured an email at all, it falls back to
recognizing them by their **customer record/name** instead — so nobody is left out just because
an email was missing on their order.

**Why this matters, in plain terms:** without this fallback, ~200 real customers would be
completely invisible on this dashboard — not shown in any list, not counted in any total, not
flagged even if they were a top spender. That's a real gap this design closes.

**Technical detail:** a live audit found real fragmentation across `Customer` records — 2,112
distinct `Customer` link values vs. only 1,921 distinct customer emails on `Lyfe Order`. Several
emails map to two separate `Customer` records (a pre-existing data-quality issue, not something
this dashboard introduced). No Customer-record merge tool exists in the app yet.

A second, separate finding: **249 real, non-excluded orders (237 distinct customers, 202 of them
with no email anywhere on any of their orders)** carry no `customer_email` at all — only a
`customer` Link. An email-only design would make these customers permanently invisible.

**The fix — a unified identity key**, used in every calculation in this module:

```
identity_key = "email:<normalized email>"        if the order has an email
             = "cust:<Customer record name>"      if it doesn't, but has a Customer link
             = (excluded from every metric)       if it has neither
```

The two halves can never collide (the `email:`/`cust:` prefix guarantees that), so a customer is
always counted exactly once, under whichever identity actually exists on their orders.

- `identity.build_identity_key(customer_email, customer)` / `parse_identity_key(identity_key)` —
  build/split this key.
- `identity.get_best_customer_doc(identity_key)` — for an `email:` key, picks the `Customer`
  Link most frequently used by that email's own orders (ties broken by most recent order); for a
  `cust:` key, the Customer doc **is** the key's value. Display/profile-link convenience only —
  never used as the grouping key for any metric.
- `identity.get_display_name(identity_key)` — **what the UI actually shows.** Resolves
  `Customer.customer_name` first; if the Customer record is stale/deleted, falls back to the raw
  customer-name string still sitting on one of that identity's own orders; if literally nothing
  resolves, shows the identity key itself as a last resort so the UI is never blank.
- Orders in status `Cancelled`, `Merged`, or `Split` are excluded from every calculation on this
  dashboard (`identity.EXCLUDED_STATUSES`) — they don't represent real, standalone customer
  activity.

**Names, not emails, everywhere in the UI.** The identity key is internal plumbing the code uses
to group and calculate correctly — it is never shown to a person. Every list, table, and profile
header displays `display_name` (the real customer name) instead.

---

## 3. Population: B2B vs. Retail

**In plain terms:** every customer gets tagged as either a **B2B/trade buyer** (a contractor,
designer, or builder) or a **Retail/homeowner buyer**. The system makes its best guess
automatically — mainly by checking whether they've ever gone through a Quotation (the sales-
assisted process only trade customers use) — but a human (Ops or Sales) always has the final say
and can correct the guess with one click. The system never silently overrides a person's
decision.

Two populations, sharing the same eight segment types but with **different thresholds** per
population (config, not hardcoded — see §4).

### 3.1 How population is decided (`classification.py`, `segments.get_population`)

**In plain terms — the decision order:**
1. If a human has already confirmed this customer's type, that decision always wins.
2. Otherwise, the system checks: *have they ever gone through a Quotation?* If yes → likely
   **B2B**, with high confidence.
3. If not, it looks for a pattern of behaviors that *together* suggest a business buyer (heavy
   custom orders, buying across many product categories, ordering often) — no single signal
   decides it alone, but two or more together suggest **B2B**; otherwise it defaults to
   **Retail**.

**Technical detail — resolution order, checked in this exact sequence:**

1. **Confirmed tag wins first.** If `Customer.custom_customer_type` (the field added for this
   BRD) is already set to `"B2B"` or `"Retail"` on the best-guess Customer doc, that value is
   used, tagged `population_source = "confirmed"`.
2. **Otherwise, the system suggests** (`classification.suggest_customer_type`), tagged
   `population_source = "suggested"`:
   - **Layer 1 — Quotation history (strongest signal).** If this identity has **any** order with
     `source_quotation` set → suggestion = **B2B**, confidence = `quotation_history`. Rationale:
     every Quotation represents the sales-assisted custom flow that self-serve Shopify/Etsy
     checkout never goes through.
   - **Layer 2 — Behavioural signals**, only reached if layer 1 found nothing. Three independent
     signals, each worth one point:
     - `custom_share_pct >= custom_heavy_share_threshold_percent` (default 50%)
     - distinct category count (`item_group`) `>= 3`
     - total order count `>= 5`

     **2 or more points → suggestion = B2B** (confidence `behavioural`). Fewer than 2 →
     suggestion = **Retail**.
   - No orders at all → suggestion = `"Unclassified"`, confidence `insufficient_data`.
3. **Layer 3 — manual tag.** `Customer.custom_customer_type` (Select: `Unclassified` / `Retail`
   / `B2B`) is the field a human writes to via the dashboard's **Confirm** button (see §8) or by
   editing the Customer record directly. Never set automatically by any code in this module.
4. **Layer 4 — checkout capture.** Deferred per the BRD; not built.

### 3.2 Confirming/correcting the suggestion — `confirm_customer_type`

**In plain terms:** whenever the system's guess hasn't been confirmed by a human yet, a
**Confirm** button appears right next to it — in the customer list and on the customer's own
profile page. One click accepts the system's guess as the official answer. This is the only
place on the whole dashboard where clicking something actually changes data (everything else is
read-only).

**Technical detail:** clicking it calls the whitelisted method
`confirm_customer_type(identity_key, customer_type)`, which:

- Requires real **write permission on `Customer`** (checked explicitly via
  `frappe.has_permission("Customer", "write")` — distinct from the page's general read-access
  check, because this is the one write action on an otherwise read-only dashboard).
- Resolves the best-guess `Customer` doc for the identity, throws if none exists.
- Writes via `doc.save()` (not a raw `db.set_value`, so Customer's own hooks still fire), setting
  `custom_customer_type` to the confirmed value.

---

## 4. Config — every threshold is adjustable, not hardcoded

**In plain terms:** the numbers that decide "is this customer VIP?" or "has this customer gone
quiet for too long?" aren't buried in code — they're editable settings any admin can tune,
separately for B2B and Retail customers, without needing a developer.

All thresholds live on the **Access Settings** singleton (`frappe.get_single("Access Settings")`,
read throughout `classification.py`/`metrics.py`/`segments.py`/`cache_refresh.py`) — there is no
separate `Customer Intelligence Settings` doctype or `/app/customer-intelligence-settings` route;
these fields sit alongside the app's other access-control config on the existing Access Settings
doc:

| Field | Default | In plain terms |
|---|---|---|
| `b2b_vip_percentile` | 90 | Top 10% of B2B customers by profit get the VIP tag |
| `retail_vip_percentile` | 90 | Top 10% of Retail customers by profit get the VIP tag |
| `b2b_at_risk_cadence_multiplier` | 2.5 | A B2B customer is "at-risk" once they're quiet 2.5× longer than their usual reorder gap |
| `retail_at_risk_cadence_multiplier` | 3 | Same idea for Retail, but a bit more lenient (3×) |
| `b2b_bulk_qty_cutoff` | 50 | A B2B order with 50+ units of one item counts as "bulk" |
| `retail_bulk_qty_cutoff` | 10 | A Retail order with 10+ units of one item counts as "bulk" |
| `custom_heavy_share_threshold_percent` | 50 | If half or more of a customer's orders are Custom, they're tagged "Custom-heavy" |
| `new_customer_window_days` | 60 | A customer is still "New" for their first 60 days |

---

## 5. Per-customer metrics — the building blocks

**In plain terms:** everything else on the dashboard is built from a handful of core numbers
computed for one customer at a time: how many orders they've placed, how often they reorder, how
much they spend, what they buy, how much gets returned, and whether the business actually makes
money on them. This section explains each of those numbers.

Every function below (`metrics.py`) is read-only — nothing here changes any data. All values are
computed for one `identity_key` at a time.

### 5.1 Core stats

**In plain terms:** basic order history — how many orders, total spend, average order size, and
how long since their last order. The average-order-size number deliberately ignores unusually
large bulk orders, so one huge one-off purchase doesn't make a customer look like a bigger
"typical" spender than they really are.

| Field | Formula |
|---|---|
| `order_count` | Count of non-excluded orders for this identity |
| `revenue` | `Σ (total_amount − shopify_refund_amount)` across all those orders (whole-history total, net of Shopify refunds) |
| `first_order_date` / `last_order_date` | Min / max `order_date` |
| `days_since_last_order` | `today − last_order_date`, in days |
| `aov` | **Bulk-aware.** `Σ total_amount / count` over only the orders whose largest line-item
  quantity is **below** the population's bulk cutoff (§4). Orders at/above the cutoff are
  excluded from both the numerator and denominator. `revenue`/`order_count` above are *not*
  affected — those stay whole-history totals. |
| `aov_order_count` | How many orders actually went into the AOV calculation |
| `bulk_excluded_count` | `order_count − aov_order_count` |

If `population` isn't known yet at call time, the **Retail cutoff (the stricter, lower one)** is
used as a conservative fallback.

### 5.2 Reorder cadence

**In plain terms:** how often does this customer typically reorder? This one number is the
foundation the "at-risk" flag is built on — instead of guessing that "60 days of silence = gone,"
the dashboard learns each customer's own normal rhythm and only worries once they've gone quiet
well past *their own* pattern.

**Technical detail:** for every distinct order date, compute the gaps (in days) between
consecutive orders, then take the **median** gap:

```
cadence = median(date[i+1] − date[i]) for all consecutive order dates
```

Computed twice: **overall** (`overall_cadence_days`) and **per category**
(`category_cadence_days`). Returns `None` when fewer than 2 dates exist — a customer with one
order has no "usual pattern" to measure against, which is exactly what keeps a one-time buyer
from ever being wrongly flagged as "at-risk forever."

### 5.3 Standard vs. Custom mix

**In plain terms:** what share of this customer's orders were custom/made-to-order vs.
off-the-shelf standard products. A high custom share is one of the signals that points toward
"this is probably a B2B/trade customer."

```
custom_share_pct = custom_order_count / total_order_count × 100
```

### 5.4 Category affinity

**In plain terms:** which product categories does this customer actually buy, and how much do
they spend in each? This is the main lens the whole dashboard is built around — nothing here is
reduced to one flat number without a category breakdown.

```
revenue     = Σ (unit_price × quantity)   per category
line_count  = COUNT(*)                    per category
```

Sorted by revenue, highest first.

> **Finish affinity is explicitly not built.** ("Hardware buyers are often finish-loyal — e.g.
> always Matte Black.") No part of the system currently records which *finish* a customer bought,
> only which *category* — so there's no data to build this from yet. This is a deliberate,
> agreed-upon gap, not something that got missed; it needs a new field added to the order-item
> data before it can exist.

### 5.5 Custom cost-to-serve

**In plain terms:** for customers who order custom/made-to-order products, this tracks how
"expensive to serve" they are — do their orders need a lot of rework/revisions before they're
happy? A high-revenue custom customer who constantly needs revisions might actually be less
profitable than they look.

```
avg_revisions            = Σ reject_count / custom_order_count
rejection_incidence_pct  = (orders with reject_count > 0) / custom_order_count × 100
```

Computed only over Custom orders. `reject_count` is the order's real rework counter, already
tracked by the factory workflow — this module just reads it.

### 5.6 Bulk flag & sample-conversion

**In plain terms — bulk:** flags any order where someone bought a large quantity of one item in
one go (a "bulk" purchase), so it can be treated differently from a normal order.

**In plain terms — sample conversion (B2B only):** the business sends out product samples to
prospective trade customers. This tracks: of the B2B customers who received a sample, how many
went on to actually place a real order afterward? A low number here would mean the sampling
program isn't earning its cost.

**Technical detail:**

```
bulk_order_count_b2b_threshold    = count of orders whose max line qty ≥ b2b_bulk_qty_cutoff
bulk_order_count_retail_threshold = count of orders whose max line qty ≥ retail_bulk_qty_cutoff
```

Sample detection reuses the same heuristic already live in the PnL dashboard — no stored
"is sample" flag exists in the schema — an order counts as a sample order if any line item name
contains "sample" (case-insensitive).

```
sample_order_count = count of this identity's orders flagged as sample orders
converted = TRUE if any later order (after the first sample order's date) is NOT itself a sample
sample_conversion_rate_pct = 100.0 if converted else 0.0   (only computed when population == "B2B")
```

For Retail or an unresolved population, this is `None` (not `0`) — the question doesn't apply
there, and showing a false `0%` would misreport "never converted" for a population the BRD never
asked this about.

### 5.7 Returns

**In plain terms:** how often does this customer send things back, and specifically how often is
it a full return-to-origin (the most costly kind of return)?

```
return_rate_pct = return_count / order_count × 100
rto_rate_pct    = rto_count    / order_count × 100
```

### 5.8 Margin

**In plain terms:** how much profit does this customer actually generate, not just how much
revenue? Because manufacturing cost data isn't complete for every product yet, some customers'
"true profit" can't be reliably calculated — for those, the dashboard shows their revenue instead
and marks it clearly as **"provisional"**, rather than showing a confidently wrong profit number.

**Technical detail:**

```
revenue = Σ (total_amount − shopify_refund_amount)   (net of Shopify refunds — same `_net_revenue`
                                                       helper used everywhere else in this module,
                                                       kept deliberately consistent with the PnL
                                                       dashboard's own revenue formula)
cogs    = Σ cost_of_goods
margin  = revenue − cogs
margin_pct = margin / revenue × 100
```

**The "provisional" guard:** the underlying cost rollup is all-or-nothing — if even one
manufactured item on an order is missing its cost data, the *entire order's* cost comes back as
`0` rather than "unknown." Treating that `0` as a real cost would make margin look artificially
inflated (up to a false 100%). So:

```
cogs_resolved = NOT (any order has real revenue but zero recorded cost)
```

When that's false, cost/margin are shown as unavailable, and the UI falls back to revenue with a
**"provisional" badge** — exactly matching the BRD's own instruction to use revenue until
Manufacturing cost data is ready.

---

## 6. Segment classification — the labels each customer gets

**In plain terms:** every customer is tagged with one or more descriptive labels based on their
real behavior — not a single fixed category, but a set of tags that can combine (a customer can
be both "Steady" *and* "At-risk" at the same time — Steady describes how they normally behave,
At-risk describes a recent worrying change from that normal behavior).

| Segment | In plain terms |
|---|---|
| **New** | First order was within the last 60 days (configurable) |
| **Repeat** / **One-time** | Ordered more than once, or only once |
| **Custom-heavy** | Half or more of their orders are Custom |
| **Project/Burst** | Buys in clusters — several categories close together, then goes quiet — but this is a normal pattern for them, not churn |
| **Steady** | Reorders at a regular, predictable rhythm |
| **At-risk** | Gone quiet well beyond *their own* usual reorder rhythm — see §6.1 |
| **VIP** | Among the top-profit customers *within their own population* — see §6.2 |

### 6.1 At-risk — the one rule the BRD calls "critical"

**In plain terms:** this is the flag meant to warn the business before a good customer is lost
for good. The key idea: **never judge "quiet too long" using a fixed number of days for
everyone.** A contractor who orders in big projects every few months isn't "gone" just because
they haven't ordered this week — they're following their normal pattern. This flag only fires
when someone has gone quiet well past *their own personal* usual gap between orders, not some
one-size-fits-all number.

**Technical detail:** a customer with only one order has no established cadence and can never be
At-risk under this rule. For a customer with a real, measurable cadence:

```
at_risk  IF  days_since_last_order  >  overall_cadence_days × at_risk_cadence_multiplier
```

using the customer's own population's multiplier (2.5× for B2B, 3.0× for Retail, by default).

### 6.2 VIP — top performers, judged fairly within their own group

**In plain terms:** the highest-profit customers get the VIP tag — but a B2B trade customer is
compared only against other B2B customers, and a Retail customer only against other Retail
customers, since the two groups spend very differently by nature. It's the top 10% by default,
using real profit where that's known, revenue where it isn't yet.

**Technical detail:** `rank_vip_within_population` splits the batch into B2B and Retail lists,
sorts each by profit descending (margin when resolved, else revenue), and tags the top
`(100 − vip_percentile)%` of each list as VIP.

### 6.3 Order-level exceptions

**In plain terms:** a customer keeps one consistent type (B2B or Retail) — but if one particular
order looks like it doesn't fit that type (say, an otherwise-Retail customer placed one
unusually large order), that single *order* gets flagged as an exception rather than relabeling
the whole customer over one unusual purchase.

- A **B2B** customer's order is flagged if it has no Quotation behind it *and* isn't a large
  order — i.e. it looks like a one-off retail-style purchase.
- A **Retail** customer's order is flagged if it clears the B2B bulk-quantity threshold — i.e. an
  unusually large one-off order.

---

## 7. Dashboard sections

**In plain terms:** this page has no tabs or view-switcher — everyone who can open it sees
everything, on one continuous scroll, in this order:

1. **Overview** — "how is the business doing overall, and where's the risk?" (strategic, big-picture)
2. **Customers** — "which specific customers need my attention today?" (an actionable list)
3. **Customer Profile** — "tell me everything about this one customer" (loads in-place, right
   below the list, the moment you click any customer row — no navigation away from the page)

There's no separate audience per section — the page is restricted to the Founder and a small
set of other authorized people (System Manager / Super Admin only, see §9), and anyone with
access sees the full page, start to finish.

### 7.1 Overview — strategic summary

| Tile | In plain terms | Calculation |
|---|---|---|
| **Profit by Category × Population** | Which product categories make the most money, split by B2B vs. Retail | Bulk SQL, grouped by category × identity, bucketed into a `matrix[category][population]`. Per bucket: `revenue − cogs = margin`, with the same provisional guard as §5.8 applied at the aggregate level. |
| **At-Risk (High-Value)** | How many valuable customers are going quiet, and how much revenue is at risk | Reads a pre-computed cache (see §8) — counts customers currently tagged At-risk, split B2B/Retail, plus their total revenue. |
| **New vs Returning Revenue** | Is growth coming from new customers or from existing ones coming back? | For every order in the filtered range: was it that customer's very first-ever order, or a later one? Revenue and order count split accordingly. |
| **Repeat Rate + LTV Trend** | Is customer loyalty and lifetime value trending up or down over the last year? | A 12-month trailing series. Per month: how many customers were active, what share of them were repeat buyers, and the cumulative average revenue-per-customer-to-date (a simple lifetime-value proxy). |

The At-Risk tile's color (green/amber/red) is based on at-risk customers as a **percentage of
the visible customer base**, not a raw headcount: `≤ 8% → green`, `≤ 15% → amber`, `> 15% → red`.

### 7.2 Customers — the operational list

| Tile | In plain terms | Calculation |
|---|---|---|
| **Segment Movement This Month** | Who changed status this month — e.g. who newly became At-risk, or who moved from At-risk back to Steady | Compares each cached customer's current tags against the tags they had at the start of this month. |
| **Sample Conversion (B2B)** | Is the sample program actually turning prospects into paying B2B customers? | Of B2B customers who ever received a sample, what share went on to place a real order. |
| **Customer List** | A sortable, filterable table of every customer and their key numbers — click any row to open their full profile | One row per customer: name, population, segment tags, orders, revenue, bulk-aware AOV, custom-order share, return rate, profit (or revenue + provisional badge). |
| **Return-Rate Outliers** | Which customers return an unusually high share of what they order | Customers with at least 3 orders **and** a return rate of 25% or higher (both numbers configurable). |

Return-rate coloring ("lower is better"): `≤ 10% → green`, `≤ 25% → amber`, `> 25% → red`.

### 7.3 Customer Profile — single-customer detail

**In plain terms:** the complete picture of one customer — every order they've placed, every
Quotation linked to them, what categories they buy, their segment tags, and their overall
numbers. Loads directly below the customer list the moment a row is clicked, and the page
scrolls straight to it — nothing to navigate away to.

`get_customer_profile(identity_key)` returns:

- Population + source (confirmed/suggested) + full segment tag set
- The complete metric bundle from §5
- **Order exceptions** (§6.3)
- **Order history** — every real order, with an "exception" pill where flagged
- **Quotations** — every Quotation linked to this customer's orders
- **CS Tickets** — intentionally absent for now; a note on the page states the support-ticket
  integration is under separate, active development elsewhere and isn't wired in here yet. A
  deliberate, tracked gap, not something silently missing.

---

## 8. Why some numbers come from a pre-computed cache instead of live

**In plain terms:** computing every customer's full profile from scratch is quick for looking at
*one* customer, but far too slow to do for *all* ~2,100 customers every time someone opens the
dashboard — it was actually measured taking over a minute. So a background job runs once an
hour, does that heavy computation for every customer in advance, and stores the results. The
handful of tiles that need "the whole customer base at once" (At-risk, Segment Movement, Sample
Conversion) read from those stored results instead of recalculating live — meaning those three
numbers can be up to an hour old, which is an acceptable tradeoff for a big-picture strategic
view. Everything else on the dashboard is always fully live and current.

**Technical detail:** the full per-customer engine costs roughly 37ms per customer (8 separate
queries). Looping it across the real ~2,100-customer base measured **65–70 seconds unfiltered**
— unusable for a page load. `get_founder_summary` (1.9s) and `get_customer_list`
(0.67s–4.5s) measured acceptable and were left as live queries; only the genuinely broken paths
moved to a cache.

`Customer Intelligence Cache` holds one row per identity, refreshed **hourly** (`hooks.py`'s
`"0 * * * *"` scheduler bucket calling `cache_refresh.run()`). Endpoints reading the cache
instead of computing live: `get_at_risk_summary`, `get_segment_movement`,
`get_sample_conversion_summary` — all three are **whole-history / current-state concepts**, not
naturally scoped to a date range, so ignoring the dashboard's date filter on these three tiles is
intentional.

**Segment movement's month-boundary rotation:** the "prior month" snapshot only updates when a
real calendar month has actually passed since the row was last refreshed — not simply "we
haven't rotated yet." This distinction matters: on a row's very first refresh, there is no real
prior month to compare against, and a naive check would wrongly treat that as a fresh rotation
and corrupt the very first comparison.

**Data-integrity guard:** the refresh job checks that a resolved Customer Link still actually
exists before storing it, since a live audit found stale Customer references still sitting on
real orders — this ensures one bad reference never silently drops a real customer from the whole
dashboard.

---

## 9. Permissions

⚠️ **Corrected:** an earlier version of this section claimed the page is restricted to System
Manager/Super Admin only, with Customer Service excluded entirely. That is no longer accurate (if
it ever was) — **Customer Service can open the page and call every endpoint**; what's actually
restricted to System Manager/Super Admin is narrower: just the **profit/margin figures**.

**In plain terms:** this page is open to **System Manager, Super Admin, and Customer Service** —
the Founder plus CS and other authorized people. It is not visible to Sales Manager or Sales User.
Within the page, the **Profit/Margin column and related profit figures are further restricted** to
System Manager/Super Admin only — Customer Service sees everything else (customer list, segments,
at-risk flags, category breakdowns, etc.) but not profit numbers. The one action that actually
*changes* data (confirming a customer's B2B/Retail type) requires an extra, specific permission
check on top of all of this.

**Technical detail:**

- **Page/endpoint access** — every whitelisted endpoint calls `_check_permission()`
  (`cid/shared.py`), which checks (1) read access to Lyfe Order, and (2) an explicit role check via
  `require_dashboard_role("customer_intelligence_dashboard")`, resolved from
  `DASHBOARD_ROLES["customer_intelligence_dashboard"]` in `mcp_audit.py` — currently
  `{System Manager, Super Admin, Customer Service}`. This matches the Page doctype's own `roles`
  list (`customer_intelligence_dashboard.json`), so a direct whitelisted-method call can't reach
  anything the page itself wouldn't show.
- **Profit visibility (client-side)** — `PROFIT_VISIBLE_ROLES = ["Super Admin", "System Manager"]`
  in `customer_intelligence_dashboard.js`; `canSeeProfit()` gates whether the profit/margin column
  and cells render at all. This is a UI-only gate on top of the page-level access above, not a
  server-side field-level restriction — the underlying endpoints still return the margin data to
  any role that can call them; Customer Service's client simply doesn't render that column.
- `confirm_customer_type` (the one write action) additionally requires real
  `frappe.has_permission("Customer", "write")`, confirmed necessary because Sales Manager has
  read-only access to Customer in this app.

---

## 10. Known, deliberate gaps (not missed — agreed and documented)

| Gap | In plain terms |
|---|---|
| **Finish affinity** | Can't track "this customer always buys Matte Black" yet — the system doesn't record which finish was bought on any order, only which category. Needs a new field added upstream. |
| **CS tickets (Gorgias) in the Operator profile** | Support-ticket history isn't shown on the customer profile yet — that integration is being built separately. |
| **True Customer-record dedup** | The dashboard correctly recognizes a customer across duplicate records by using their identity key (§2); actually merging the duplicate Customer records themselves is a separate, tracked project. |
| **Checkout-capture B2B signal** | A "are you a business?" checkbox at checkout was considered but explicitly deferred — not part of this build. |
| **Return-rate outliers tile has no UI yet** | The underlying calculation works and is available, but it isn't wired into a visible tile on the dashboard page yet (noted during the identity-key rework, not previously flagged). |

**Fixed, historical bugs worth knowing about (not open issues — noted for anyone touching this code):**
`get_customer_list`'s revenue ranking and `get_return_outliers` both previously summed
`total_amount` against a joined line-item table with no dedup, double-counting revenue for any
customer with more than one line item per order. Both are now fixed via a `SELECT DISTINCT`
subquery (`cid/customers.py`) — flagging this so nobody re-introduces the same join pattern
elsewhere in this module.

---

## 11. Annex C (LH-BRD-DEV-009) — trim-to-summary + component-gap engine

**In plain terms:** a follow-up brief asked for two things: (1) make the main screen readable
in under 90 seconds by trimming heavy tables to a summary + "view detail →" link, and (2) add a
new engine that flags customers who bought some but not all of a defined product "kit" (e.g.
Tubing + Brackets + End Caps for a Railing Kit) — a strong signal they have an unfinished or
externally-sourced project, worth a CS outreach.

Full change plan (rationale, sequencing, verification against the BRD text): see
`lh/docs/customer_intelligence_dashboard_v2_plan.md`.

### 11.1 Trim to summary

Every heavy table (category × population matrix, 12-month LTV trend, full customer list) now
renders a summary/KPI view by default, with a "View detail →" link that opens the full table in
a dialog (`openDetailDialog()` in the JS) — the underlying endpoints are unchanged, this is a
client-side rendering trim only. The Founder KPI row also shows the top 3 categories by profit
inline (Annex C: "trim the full table to top 3 by profit + view detail", not KPIs alone).

### 11.2 Staleness (CI-6)

Every cache-backed endpoint (`get_at_risk_summary`, `get_segment_movement`,
`get_sample_conversion_summary`, and the new component-gap endpoints below) now returns a
`last_refreshed` timestamp — the oldest `last_scanned_at`/`last_computed` among the rows feeding
that tile. The client greys the tile and shows "as of {timestamp}" once that timestamp is more
than ~90 minutes old (`STALE_AFTER_MINUTES` in the JS — a starting default, not a BRD-specified
number). This is distinct from the existing "Data unavailable" failure state: unavailable means
the endpoint threw; stale means it succeeded but the data behind it is old.

### 11.3 Component-Gap Recommendation Engine

**Kit Composition** (new doctype, `lh/lyfe_hardware/doctype/kit_composition/`) — ops-editable,
starts empty, category-level (Table MultiSelect of Item Group), deliberately NOT derived from
Lyfe BOM (a BOM is a per-product manufacturing build; this needs "has this customer bought ANY
tubing" — a different, category-level question). `validate()` rejects a rule with fewer than 2
required categories (CI-2's gap definition is meaningless below that) and rejects a category
appearing in both required and optional.

**The engine** (`lh/lyfe_hardware/customer_intelligence/component_gap.py`) runs as a second pass
inside the existing hourly `cache_refresh.run()` job — one cron bucket, not a new scheduler
entry. For each active Kit Composition rule and each customer, a gap = the customer has bought
at least one but not all required categories (CI-2). Opportunity $ is the **global** per-category
average order value (summed across the missing categories) — never the customer's own AOV in
that category, since by CI-2's own definition they have zero orders there. That global figure
(`get_global_category_avg_order_value()` in `customer_intelligence_dashboard.py`) is computed
**once per refresh cycle**, coarsening the same bulk SQL `get_founder_summary()` already runs —
deliberately not derived by looping the per-identity `get_category_affinity()` across the full
customer base, which would reproduce the exact 65–70s-unfiltered performance failure the base
BRD's cache was built to avoid (§8 above).

**Component Gap Flag** (new doctype) holds one row per (customer, kit) gap — `state` (Suggested
/ To Pursue / Needs Founder Approval / Don't Pursue), `reason` (required when Don't Pursue,
CI-4), and two suppression snapshots: `bought_snapshot` (the customer's category set at the time
of the Don't-Pursue decision) and `kit_rule_modified_snapshot` (the rule's `modified` timestamp
at that time). A suppressed row only re-surfaces if **either** the customer's bought-category set
or the rule itself has changed since — matching CI-4's exact wording ("until the rule **or**
their orders change"), not just the bought-set alone.

**New whitelisted endpoints:**

| Endpoint | Purpose |
|---|---|
| `get_component_gap_summary()` | Founder tile — total opportunity $, customers flagged, count awaiting founder approval. Stays small per the BRD's "do not let it grow the dashboard" instruction. |
| `get_component_gap_list(filters=None)` | CS working panel — Customer, Bought, Missing, Implied Project, Opportunity $ (estimate-labeled per CI-3), State, reason on hover. |
| `set_component_gap_state(name, state, reason=None)` | The one write action for this engine — same permission shape as `confirm_customer_type` (`_check_permission()` + an explicit `frappe.has_permission("Component Gap Flag", "write")` check, since this writes a dashboard-owned doctype, not `Customer`). Requires a reason when state is Don't Pursue. |

**Affinity engine** ("customers who bought X also bought Y" statistical affinity) is explicitly
out of scope per the BRD — noted, not built (DoD #5).

**Verified live** (2026-08-27): created a test Kit Composition rule against two real Item
Groups, confirmed a real customer with exactly one of the two categories was correctly flagged
with the right `bought`/`missing`/`opportunity_amount`, confirmed the Don't-Pursue suppression
holds across an unchanged re-run and correctly clears when the kit rule is edited. Test data
cleaned up after verification — no residue left in the live database.
