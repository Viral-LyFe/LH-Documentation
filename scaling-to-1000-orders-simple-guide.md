# Handling 1,000 Orders Per Day — Simple Guide

> This document explains what changes we are making to the system so it can handle 1,000 orders per day without breaking, slowing down, or missing anything.
> Written for everyone — no technical background needed.

---

## What Are We Trying to Do?

Right now the system works well. But as the business grows and we reach **1,000 orders per day**, the system needs to be prepared for that load.

1,000 orders/day means roughly **42 orders every hour** — but during busy hours (like mornings or sale days) it could jump to **150–300 orders in a single hour**. That's a lot happening at the same time.

This guide explains:
- How we are making the system faster and stronger
- What could go wrong and how we are preventing it
- How the developer gets notified instantly if anything fails
- How we are testing everything before it goes live

---

## 1. Making the System Handle More Orders at Once

Think of the system like a restaurant kitchen. If only 1 chef is cooking, orders will pile up. We need more chefs working at the same time.

**What we are doing:**
- Upgrading the server to a bigger, more powerful one (more CPU and memory)
- Running **5 background workers** at the same time — like having 5 chefs in the kitchen
  - 2 workers for normal order processing
  - 2 workers for quick small tasks (status updates, lookups)
  - 1 worker for heavy long-running jobs (bulk syncs, reports)

**How incoming orders are handled:**
When a new order comes in from Shopify or ShipStation, the system immediately puts it in a **queue** (like a waiting list) and says "got it, we'll process it." The actual order creation then happens in the background. This way:
- The system never gets stuck waiting
- No orders are lost even during a rush
- Shopify/ShipStation never times out waiting for a response

---

## 2. Making Order Search Faster

When your team searches for an order using the **Internal Order ID** or **Source Order ID**, the system currently looks through every single record in the database to find a match — like searching a book without an index.

**What we are doing:**
Adding **search shortcuts** (called database indexes) on the most commonly searched fields:

| What Gets Indexed | Why |
|---|---|
| Internal Order ID | Teams search by this directly |
| Source Order ID | Teams search by this directly |
| Order Status + Warehouse | Used in filters frequently |
| ShipStation Order Status + Date | Used in sync and reports |
| SLA fields | Used every 15 minutes by the SLA scanner |

**Result:** Searches that used to take 2–3 seconds will complete in milliseconds.

---

## 3. Smarter Communication With External Services

Our system talks to several outside services every day:

| Service | What It Does |
|---|---|
| **ShipStation** | Syncs orders, shipping labels |
| **Shopify** | Receives orders, payment status |
| **CJ** | International shipment creation |
| **17Track** | Order tracking updates |
| **S3** | Stores files (labels, factory images, invoices) |

**Problems at high volume:**
- Calling the same service 500 times with the same question wastes time
- If a service is temporarily slow, it can block everything else
- If a sync job starts while the previous one is still running, data can get confused

**What we are doing:**

**Remembering answers (Caching):**
If we already asked ShipStation for its carrier list 5 minutes ago, we remember the answer for 1 hour instead of asking again. Same for other repeated lookups.

**Retrying automatically:**
If a service is temporarily busy, instead of failing immediately, the system tries again — first after 1 second, then after 2 seconds, then after 4 seconds. Most short outages resolve within this window.

**Preventing overlapping syncs:**
The ShipStation sync runs every 30 minutes. We now put a "lock" on it — if it's already running, the next trigger skips instead of starting a second copy. This prevents duplicate orders and data conflicts.

**S3 file uploads:**
All file uploads (shipping labels, factory images) happen in the background, never during an order save. This keeps the save operation fast even when files are large.

---

## 4. Preventing Known Problem Areas

We identified the parts of the system most likely to cause issues at high volume and fixed them proactively.

| What Could Go Wrong | Plain English Explanation | How We Fixed It |
|---|---|---|
| Every time a Lyfe Order is saved, 3 automatic checks run | Even saving a small note triggers heavy background checks | Now checks only run if the **order status actually changed** |
| ShipStation sync could run twice at the same time | Two copies running = duplicate data or conflicts | Added a lock so only one copy runs at a time |
| Shopify access token rotation (every 45 min) | Two workers could try to rotate it simultaneously | Only one worker can rotate at a time |
| Old history/version records pile up in database | Slows down the whole system over time | Cleanup job already exists — now runs more carefully with limits |
| SLA scanner runs every 15 min on growing data | Gets slower as more orders accumulate | Fixed with database indexes (see Section 2) |

---

## 5. Instant Alerts When Something Goes Wrong

If anything fails — an order doesn't get created, a Gate Pass can't be submitted, or a background sync crashes — **the developer gets notified immediately** through both:

- **Slack message** — instant notification in the developer's Slack account
- **Email** — detailed email with the error and a direct link to the affected record

### What the alert looks like

**Slack:**
```
🔴 ShipStation → Lyfe Order Creation Failed
Duplicate entry for source_order_id SS-12345
🔗 Open ShipStation Orders: SS-ORD-2024-001
```

**Email subject:** `[LH Alert] Gate Pass Submission Failed`

The email includes the exact error message and a clickable link to open the failed document directly in the system.

### Which failures trigger an alert

| What Failed | Slack | Email |
|---|---|---|
| Shopify order not created | ✅ | ✅ |
| ShipStation order not converted to Lyfe Order | ✅ | ✅ |
| Gate Pass could not be submitted | ✅ | ✅ |
| ShipStation sync job crashed | ✅ | ✅ |
| SLA scan job crashed | ✅ | ✅ |

Every alert is also saved in the system's **Error Log** so nothing is ever lost.

### How to set it up
Two things need to be configured in the system settings (one-time setup by developer):
- Slack webhook URL (get from Slack settings)
- Developer email address

---

## 6. How We Deploy Changes Safely

Deploying new code to a live system always carries some risk. Here is how we minimize it:

**Rules we follow:**
- Never push updates during busy business hours (IST mornings/evenings)
- Always keep a backup version of the code ready to switch back to instantly
- Test everything on a **staging site** (an exact copy of production) before going live
- Put the system in "maintenance mode" during deployment so no orders come in mid-update

**Before any major change goes live, we test it by:**
- Simulating 50 users placing orders at the same time on the staging site
- Checking that every order goes through in under 10 seconds
- Making sure error rate stays below 0.5% (less than 5 failures per 1,000 orders)
- Checking that the system doesn't get overloaded (queue stays below 50 pending jobs)

---

## 7. What We Are Monitoring 24/7

Once everything is set up, these things are watched automatically:

| What's Monitored | Alert Triggers When |
|---|---|
| Server CPU usage | Goes above 80% |
| Server memory usage | Goes above 85% |
| Background job queue | More than 100 jobs waiting |
| Database query speed | Any query takes more than 1 second |
| Any job failure | Immediately on failure |

---

## 8. Full To-Do List (For Developer Reference)

- [ ] Add database search indexes for Internal Order ID, Source Order ID, and other key fields
- [ ] Make Lyfe Order automatic checks only run when order status changes
- [ ] Add overlap protection to ShipStation sync and Shopify token rotation
- [ ] Make all webhook handlers queue work and respond instantly
- [ ] Set up `alerts.py` and connect it to Gate Pass, ShipStation, Shopify, and SLA scan
- [ ] Add Slack webhook URL and developer email fields to ShipStation Settings
- [ ] Set worker counts on Frappe Cloud (2 default, 2 short, 1 long)
- [ ] Run slow query capture on staging for 24 hours
- [ ] Run load test with 50 simultaneous users on staging
- [ ] Enable CPU/memory/queue alerts on Frappe Cloud dashboard
- [ ] Confirm ShipStation sync always uses last-sync timestamp (not full resync)

---

## Summary

| Area | What We Are Doing |
|---|---|
| Server capacity | Upgrading to handle peak load |
| Order processing | Queue-based, never blocks incoming orders |
| Search speed | Adding indexes on most-searched fields |
| External services | Caching, retries, overlap protection |
| Known risks | Fixed proactively before they become problems |
| Failure alerts | Instant Slack + email to developer on any failure |
| Deployment | Safe rollout process with staging tests |
| Monitoring | 24/7 automated alerts on key metrics |

The goal is simple — **the system should handle 1,000 orders a day without the team noticing any difference in speed or reliability.**
