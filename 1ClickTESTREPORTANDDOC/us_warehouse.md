# US / India Fulfillment Routing — Functional Guide

**Organization:** Lyfe Hardware

---

## Table of Contents

1. [Overview](#1-overview)
2. [The Four Fulfillment Routes](#2-the-four-fulfillment-routes)
3. [How the System Decides Which Route to Use](#3-how-the-system-decides-which-route-to-use)
4. [Manual Override — Force US / Force India](#4-manual-override--force-us--force-india)
5. [Mixed Orders — Confirm Split & Factory Leg Destination](#5-mixed-orders--confirm-split--factory-leg-destination)
6. [Order Legs — Tracking Each Physical Shipment](#6-order-legs--tracking-each-physical-shipment)
7. [Transfer Order — Tracking Factory to US Warehouse](#7-transfer-order--tracking-factory-to-us-warehouse)
8. ["US Warehouse Delivered" — Manual Post to 1Click](#8-us-warehouse-delivered--manual-post-to-1click)
9. [Tracking Updates for Customers](#9-tracking-updates-for-customers)
10. [Dispatch Deadline (72-Hour Rule)](#10-dispatch-deadline-72-hour-rule)
11. [Route-by-Route Walkthrough](#11-route-by-route-walkthrough)
12. [SKU Validation and Auto-Fill](#12-sku-validation-and-auto-fill)
13. [Order Statuses Explained](#13-order-statuses-explained)
14. [Quick Reference](#14-quick-reference)

---

## 1. Overview

Lyfe Hardware fulfills orders from two locations:

| Location | Who Handles It | Used When |
|---|---|---|
| **US Warehouse** (1Click Logistics) | 1Click picks, packs, and ships to the customer | Items are available in US stock |
| **India (Factory)** | India/Factory team finishes and ships | Items are not in US stock |

When an order arrives, the system automatically checks what stock is
available at the US Warehouse and decides the best fulfillment path — no
manual steps needed for the decision itself.

There are **four possible routes**, and the system picks the right one
automatically. A manager/CS can also force a specific route when needed.

---

## 2. The Four Fulfillment Routes

### Route A — US Full
**All items are available at the US Warehouse.**

- 1Click ships directly to the customer from the US.
- This is the fastest path for the customer.
- No involvement from Factory.

---

### Route B — Mixed (Some Items US, Some From Factory)
**Some components are in the US Warehouse, but others are not.**

Any BOM/kit component split between US stock and Factory triggers this
route. A human confirms the split (see §5) and chooses one of two
outcomes:

- **Via US Warehouse** (the default, preferred choice) — Factory ships
  its portion to the US warehouse first; once it arrives, everything is
  combined and shipped to the customer as **one single package**. Use
  this whenever there's no specific reason to do otherwise.
- **Direct to Customer** — Factory ships its portion straight to the
  customer as its own separate package, while the US-covered items ship
  separately from 1Click, at the same time. **The customer receives two
  packages.** This is a deliberate choice for a specific situation: when
  waiting for Factory's portion to first travel to the US warehouse
  would add real, avoidable delay, and getting the customer their US-
  stocked items sooner is worth splitting the shipment for. It is not
  the default and should not be picked just because it's available — use
  it only when that tradeoff genuinely applies.

---

### Route C — India Direct Dropship
**Nothing is in the US Warehouse, and the customer is outside the US.**

- Factory ships the complete order directly to the customer.
- No US leg involved.
- The customer receives an international shipment from India.

---

### Route D — India to US to Customer
**Nothing is in the US Warehouse, and the customer is in the US.**

- Factory ships the complete order to the US Warehouse first.
- Once 1Click receives the goods, they ship to the customer.
- The customer receives a US domestic shipment for the best delivery
  experience.

---

## 3. How the System Decides Which Route to Use

When a ShipStation order arrives, the system immediately checks live
1Click US Warehouse stock for every item in the order:

```
Are ALL items in US stock?
  → YES  →  Route A (US Full)
  → NO   →  Is SOME (not all) of it in US stock?
               → YES  →  Route B (Mixed) — holds for human review
               → NO   →  Is the customer in the US?
                            → YES  →  Route D (India → US → Customer)
                            → NO   →  Route C (India Direct Dropship)
```

This decision is fully automatic. The result is shown on the order as
the **Routing Outcome**. Routes A, C, and D post to 1Click (or ship
directly) automatically — no one needs to click anything. Route B always
stops and waits for a human to confirm the split (see §5).

**Known limitation:** if two orders for the same low-stock item are
processed at nearly the same time, 1Click's stock check does not
guarantee that only one of them gets accepted — both can be told "yes,
it's in stock" and both get submitted, even though only one physical
unit actually exists. This has been confirmed through real testing and
is tracked as an open item. If you notice two orders for the same item
being processed close together, especially when stock is low, it's
worth a manual check rather than assuming the system caught it.

---

## 4. Manual Override — Force US / Force India

If CS/a manager needs to force a specific fulfillment path (for example,
due to a customer deadline, or a correction to a stock check), they can
use the routing override on the order:

| Override | What It Does |
|---|---|
| **Auto** (default) | System decides based on stock availability |
| **Force US** | Always use US Warehouse, regardless of what the stock check found |
| **Force India** | Always use Factory; the system then decides direct-to-customer vs. forward-to-US based on the customer's country |

An **Override Reason** is required for either override — the save is
blocked without one, for audit purposes.

**One rule:** Force US cannot be used once an order already has a
confirmed Mixed split with a Factory Leg Destination set. If you see an
error about this, the order is already committed to a Mixed shipping
plan — that would need to be cleared first.

---

## 5. Mixed Orders — Confirm Split & Factory Leg Destination

**This is the core Route B flow**, and it always requires a human step —
the system never auto-posts a Mixed order to 1Click on its own.

### Step 1 — The order holds automatically

The moment the system detects a Mixed split, the order stays at status
**New**. Two tables show exactly what was found:
- **US Warehouse Shipment Item** — the components that ARE in US stock.
- **Factory Warehouse Shipment Item** — the components that are NOT.

Nothing is submitted to 1Click yet.

### Step 2 — Choose the Factory Leg Destination

Before confirming the split, someone must choose how Factory's portion
will travel:

- **"Via US Warehouse"** (the default choice) — Factory ships to the US
  warehouse first, everything combines into one shipment. Pick this
  unless there's a specific reason not to.
- **"Direct to Customer"** — Factory ships straight to the customer as
  its own separate package, at the same time the US-covered items ship
  from 1Click. Pick this only when getting the US-stocked items to the
  customer sooner is worth the tradeoff of a second package — not as a
  default or a shortcut.

### Step 3 — Confirm Split

Once the destination is chosen, clicking **Confirm Split** does one of
two things, depending on the choice above:

**"Via US Warehouse":**
- A **Transfer Order** is created for the Factory-bound components (see
  §7) and the order holds again, waiting for that Transfer Order to be
  marked **Received**.
- A real **"Factory → US Warehouse" Order Leg** is created at this exact
  moment (see §6) — it exists the whole time the shipment is in transit,
  not only once it's already delivered.
- Once the Transfer Order is marked Received, the system automatically
  posts the **whole order — every item, from both sides — to 1Click as
  one combined shipment.**

**"Direct to Customer":**
- Factory's portion goes straight into the normal Factory Assignment →
  Ready for Dispatch flow, exactly like any other Factory order.
- The US-covered items are posted to 1Click immediately, as their own
  separate order.
- **Two real Order Legs** are created right away — "US Warehouse →
  Customer" and "Factory → Customer" — since these are genuinely two
  independent physical packages from this point forward.

---

## 6. Order Legs — Tracking Each Physical Shipment

Every real physical shipment an order produces gets its own **Order Leg**
record — a single Lyfe Order can have 1 or 2 of these, depending on the
route. This is what lets the system tell "the whole order is done" apart
from "one of two packages arrived."

| Leg Type | Exists For | Tracking Comes From |
|---|---|---|
| **US Warehouse → Customer** | Routes A, D, and both Mixed outcomes | Automatically, from 1Click's own hourly tracking sync |
| **Factory → Customer** | Route C, and Mixed "Direct to Customer" | Whoever enters the plain **Tracking Number**/**Carrier** fields on the order — this automatically mirrors onto this leg |
| **Factory → US Warehouse** | Mixed "Via US Warehouse" (and Route D) | Whoever enters **Tracking Number (US)** / **Carrier (US)** on the order, or directly on the Transfer Order — both automatically mirror onto this leg |

**Where to see them:** open the order's **Warehouse Split tab** — the
**Order Legs** section at the top shows every leg for this order, with
its current status, carrier, and tracking number, all in one place — no
need to open each leg separately just to check tracking.

**The completion rule:** the order only reaches status **Completed**
once **every one of its legs** independently shows Delivered. A Mixed
"Direct to Customer" order with 2 real packages will not close out just
because one of them arrived — both have to.

Order Legs are what actually decides "Completed" for any order that has
them. An order with no Order Legs at all (this can only happen for
orders that existed before this feature was introduced) falls back to
the older rule instead — its single Tracking Number reaching "Delivered"
is enough on its own.

---

## 7. Transfer Order — Tracking Factory to US Warehouse

For Route D and Mixed "Via US Warehouse", when Factory ships goods to the
US Warehouse, a **Transfer Order** is created to track that shipment.

The Transfer Order records:
- Which Lyfe Order it belongs to
- The items being shipped and quantities
- The Factory-to-US carrier and tracking number
- Number of cartons and any notes

**Status flow:** Draft → Shipped → Received

Whatever tracking number/carrier gets entered here automatically flows
two ways:
- Onto the linked Lyfe Order's **Tracking Number (US)** / **Carrier
  (US)** fields.
- Onto the linked **"Factory → US Warehouse" Order Leg** (see §6) — so
  the leg's status also updates as the Transfer Order moves through
  Draft → Shipped → Received.

The Transfer Order naming follows the format: `ASN-2026-00001`.

**One important real-world rule:** marking this Transfer Order
"Received" is Factory's own internal confirmation that goods physically
arrived — it does **not** mean 1Click's own inventory system has
necessarily caught up to reflect it yet. In practice the order resumes
and posts to 1Click almost immediately after, but this is worth knowing
as a small, accepted timing gap.

---

## 8. "US Warehouse Delivered" — Manual Post to 1Click

Once Factory's shipment to the US warehouse is confirmed delivered, the
order stops at status **"US Warehouse Delivered"** and waits. A
dedicated **"Post to 1Click"** button appears on the order — only
visible in this exact status — for CS to review and click when ready.

This gives CS a deliberate checkpoint to confirm quantities and the
shipping address before the order actually goes out, rather than posting
automatically the instant a tracking number looks delivered.

**Note — a second, separate button exists too:** there's also a
**"Post US Portion to 1Click"** button (in the Warehouse Split area) —
that one is for the "Direct to Customer" scenario and only posts the
US-covered portion. The two buttons are for different scenarios and are
not interchangeable.

---

## 9. Tracking Updates for Customers

The system automatically pushes the correct tracking number to Shopify
or Etsy, via ShipStation, based on which leg is the customer-facing one:

| Route | Customer-Facing Tracking |
|---|---|
| Route A — US Full | The single US Warehouse → Customer leg |
| Route B — Mixed, Via US Warehouse | The single, combined US Warehouse → Customer leg |
| Route B — Mixed, Direct to Customer | **Both** legs push their own tracking — the customer genuinely gets two tracking numbers, one per package |
| Route C — India Direct | The single Factory → Customer leg |
| Route D — India to US | The single US Warehouse → Customer leg |

Tracking numbers are updated in ShipStation automatically. ShipStation
then notifies Shopify or Etsy as part of its normal order sync — no
separate push is needed from this system.

---

## 10. Dispatch Deadline (72-Hour Rule)

Every order is automatically stamped with a **Promised Dispatch By** time
at the moment the order is created — set to **72 hours after order
creation**, calculated in Indian Standard Time (IST).

This is a dispatch deadline, not a delivery deadline. If an order is
approaching or past its dispatch deadline with no tracking number yet, it
should be flagged for review — a real, automatic follow-up task is
created for exactly this once an order has been submitted to 1Click too
long without a real dispatch number (see the SLA/escalation system).

---

## 11. Route-by-Route Walkthrough

### Route A — US Full

1. Order arrives. System checks 1Click — all items are in stock.
2. Routing Outcome set to **US_FULL**.
3. Order is posted to 1Click automatically — no human step.
4. A single "US Warehouse → Customer" Order Leg is created.
5. 1Click picks, packs, and ships. Tracking flows in via the hourly sync,
   onto both the order and the leg.
6. Once the carrier confirms delivery, the leg (and, since it's the only
   one, the whole order) reaches Completed.

---

### Route B — Mixed, "Via US Warehouse"

1. Order arrives. Some components in US stock, some not.
2. Routing Outcome set to **MIXED_US_COMPONENTS_INDIA_TO_US**. Order
   holds at status New.
3. A human reviews the split and picks Factory Leg Destination = "Via US
   Warehouse", then clicks **Confirm Split**.
4. A Transfer Order is created for Factory's portion, and a "Factory → US
   Warehouse" Order Leg is created alongside it. Order holds again.
5. Factory ships; tracking is entered on the Transfer Order (or directly
   as Tracking Number (US)/Carrier (US) on the order) — both the Transfer
   Order and the Order Leg pick up the same tracking automatically.
6. Once the Transfer Order is marked Received, the order status becomes
   **"US Warehouse Delivered"** — it does NOT auto-post to 1Click.
7. CS reviews and clicks **"Post to 1Click"** (see §8).
8. The whole order — every item from both sides — posts to 1Click as one
   combined shipment. A "US Warehouse → Customer" leg is created for
   this final shipment.
9. Once 1Click ships and the carrier confirms delivery, that leg reaches
   Delivered, and since both legs are now Delivered, the order reaches
   **Completed**.

---

### Route B — Mixed, "Direct to Customer"

1. Order arrives. Some components in US stock, some not.
2. Routing Outcome set to **MIXED_US_COMPONENTS_INDIA_TO_US**. Order
   holds at status New.
3. A human picks Factory Leg Destination = "Direct to Customer", then
   clicks **Confirm Split**.
4. Two things happen immediately: Factory's portion goes into the normal
   Factory Assignment flow, and the US-covered items post to 1Click right
   away as their own order. Two Order Legs are created at this point —
   "US Warehouse → Customer" and "Factory → Customer".
5. Each leg's tracking is entered independently — the US leg fills in
   automatically from 1Click's hourly sync; the Factory leg fills in the
   moment someone enters the plain Tracking Number/Carrier fields on the
   order.
6. The customer genuinely receives two separate tracking numbers, one per
   package.
7. The order only reaches **Completed** once BOTH legs independently show
   Delivered — one arriving does not close out the order.

---

### Route C — India Direct Dropship

1. Order arrives. No items in US stock. Customer is outside the US.
2. Routing Outcome set to **INDIA_DIRECT_DROPSHIP**.
3. Order is **not** submitted to 1Click at all.
4. Factory finishes and ships directly to the customer.
5. A single "Factory → Customer" Order Leg exists; tracking is entered
   directly on the order (plain Tracking Number/Carrier fields).
6. The normal hourly carrier tracking check picks it up and advances the
   order through Shipped → Completed, same as any plain Factory order.

---

### Route D — India to US to Customer

1. Order arrives. No items in US stock. Customer is in the US.
2. Routing Outcome set to **INDIA_TO_US_TO_CUSTOMER**.
3. Order is posted to 1Click automatically — but held (awaiting Factory's
   shipment to arrive at the US warehouse first).
4. A Transfer Order is created for Factory's shipment.
5. Factory ships the complete order to the US Warehouse; tracking is
   entered on the Transfer Order.
6. Once the Transfer Order is marked Received, the order automatically
   resumes and posts the full order to 1Click.
7. 1Click ships to the customer; tracking flows in via the hourly sync.
8. Once delivered, the order reaches Completed.

---

## 12. SKU Validation and Auto-Fill

**Auto-extraction from item name:** when an order item has no SKU, the
system checks the item name for two patterns, in order:
1. An explicit label like `(SKU: 19L-R222-RB)`.
2. A plain SKU-shaped code in parentheses with no "SKU:" label, e.g.
   `Flush Elbow Fitting 90 Degree (FE90-200-AB)` — this is actually the
   more common real-world pattern.

Either way, the extracted value is only used if it's confirmed to match a
real Item Master record — an unmatched candidate is discarded, never
guessed.

**Auto-fill erp_item from SKU:** if an item has a SKU but no ERP Item
linked, the system tries to find and fill the ERP Item automatically.

**Validation block:** for Standard orders in New status, saving is
blocked if any physical item (non-fee, non-adjustment) is missing a SKU.
The user must either add the SKU or change the Order Type to Custom. Fee
lines (`Custom Fee`, `Customs Fee`, any capitalization) are always
exempt.

**Note:** in this system's real data, a `sku`, the ERP Item's own
`item_code`, and `erp_item` are always the same value for a given item —
there is no separate "SKU identity" apart from the ERP Item's own code.

---

## 13. Order Statuses Explained

| Status | Meaning |
|---|---|
| *(blank / New)* | Order just created, or a Mixed split detected and awaiting Confirm Split |
| **Factory Assignment** | Assigned to Factory for production/fulfillment |
| **Ready for Dispatch** | Factory has it ready to ship |
| **Submitted to 1Click** | Posted to 1Click, waiting for dispatch |
| **US Warehouse Delivered** | Factory's shipment to the US warehouse has arrived; waiting for CS to click "Post to 1Click" (§8) |
| **1Click Error** | Something went wrong posting to or checking with 1Click — check the error field |
| **Shipped** | On the way to the customer (at least one leg in transit) |
| **Completed** | Every leg for this order has independently shown Delivered |

**Note:** "Pending India Dispatch" and "Awaiting India Components" are
older names that still exist internally (used for background tracking
only) but no longer appear as an order's visible status — if you see
either mentioned elsewhere, it refers to that internal state, not
something you'll see in the Status field itself.

---

## 14. Quick Reference

| Question | Answer |
|---|---|
| Which orders go through this routing? | ShipStation orders only, automatically |
| When does the system use the US Warehouse? | Whenever an item is genuinely in US stock — not tied to product type |
| When does Factory ship directly to the customer? | Route C (non-US customer, nothing in US stock), or Mixed "Direct to Customer" |
| Can I force a specific route? | Yes — Force US or Force India, with a required Override Reason |
| What is an Order Leg? | One record per real physical shipment this order produces — 1 or 2 depending on the route |
| When does the order reach Completed? | Only once every one of its Order Legs is independently Delivered |
| What is a Transfer Order? | Tracks a Factory-to-US-warehouse shipment (Route D, Mixed "Via US Warehouse") |
| What is "US Warehouse Delivered"? | A deliberate pause — Factory's shipment arrived at the US warehouse, but the order needs a human to click "Post to 1Click" before it goes out |
| What is Promised Dispatch By? | 72 hours from order creation (IST) — the dispatch deadline |
| Where do I see tracking for each leg? | Open the Lyfe Order → Warehouse Split tab → Order Legs section |

---

*For technical implementation details, refer to `apps/lh/CLAUDE.md` and
the US Warehouse Integration Testing documents.*
