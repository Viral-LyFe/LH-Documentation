# Slack Integration — Complete Documentation

This document lists **every** Slack message our system sends or receives, in plain language. Use this to understand what each Slack alert means, where it comes from, and how to configure it.

> **Note on "Settings":** Most settings below are configured once by an admin (System Manager). Regular users don't need to touch these — they're listed here so you know where to look if a message isn't showing up.

---

# Section 1: Order Tracking Alerts (17Track)

These alerts come from our shipment tracking system (17Track), which monitors every package after dispatch.

## 1.1 Tracking Problem Alert

**Details:**
This is a general "something's wrong with this shipment" alert. It covers 4 different problems, each shown with its own icon:
- 🚨 **Possible Lost** — carrier hasn't updated tracking in over 30 days
- ❌ **Delivery Exception** — carrier reported a delivery failure/exception
- 🐌 **Stale Tracking** — package is in transit but hasn't moved in 7+ days
- 📭 **Carrier Never Scanned** — label was created but carrier hasn't scanned the package in 5+ days

The alert only fires once when the problem is first detected (or when it changes), not repeatedly.

**Settings if need to do:**
- Doctype: **Seventeen Track Settings**
- Turn on: `Enable 17Track` and `Enable Tracking Alerts`
- Set: `Tracking Alert Slack Webhook URL`
- Optional: adjust `Stale Tracking Days` (default 7) and `Unscanned Label Days` (default 5)
- A backup email is also sent to the recipients listed in `Tracking Alert Recipients`

**Channel name:** `[Placeholder: Tracking Alerts Channel]`

**Image placeholder:**
`[Screenshot Placeholder: Tracking Problem Alert message in Slack]`

**How it can help users:**
Lets the CS/ops team catch problem shipments early and proactively contact the carrier or customer — instead of waiting for the customer to complain.

---

## 1.2 US Warehouse Delivered Notice

**Details:**
Fires one time, the moment an order's US-leg shipment is marked "Delivered" by the carrier and the order status changes to "US Warehouse Delivered."

**Settings if need to do:**
- Doctype: **LH Slack Settings**
- Turn on: `Enable Consolidated Alerts`
- Set: `Consolidated Alerts Channel ID`

**Channel name:** Consolidated Alerts Channel

**Image placeholder:**
`[Screenshot Placeholder: US Warehouse Delivered message]`

**How it can help users:**
Tells the CS/dispatch team the exact moment an order lands at the US warehouse, so they know it's ready for the next step (posting to 1Click for final delivery).

---

# Section 2: Order & Quotation Interactive Alerts (Consolidated Alerts Channel)

All 4 of the below go to the same Slack channel, configured once in **LH Slack Settings**.

## 2.1 1Click Error Alert

**Details:**
Fires the instant an order fails to be created in 1Click (our last-mile delivery partner system).

**Settings if need to do:**
- Doctype: **LH Slack Settings** → `Enable Consolidated Alerts` + `Consolidated Alerts Channel ID`

**Channel name:** Consolidated Alerts Channel

**Image placeholder:**
`[Screenshot Placeholder: 1Click Error Alert]`

**How it can help users:**
Alerts the ops/engineering team immediately so they can fix the issue and retry, instead of the order silently getting stuck.

---

## 2.2 SKU Auto-Registration Alert

**Details:**
Fires when 1Click rejects one or more product SKUs on an order because it doesn't recognize them. The system automatically tries to register the missing SKUs with 1Click and reports whether it succeeded.

**Settings if need to do:**
- Doctype: **LH Slack Settings** → `Enable Consolidated Alerts` + `Consolidated Alerts Channel ID`

**Channel name:** Consolidated Alerts Channel

**Image placeholder:**
`[Screenshot Placeholder: SKU Auto-Registration Alert]`

**How it can help users:**
Tells the team whether they can simply retry the order now, or whether a SKU needs manual setup — saves time diagnosing 1Click failures.

---

## 2.3 SLA Breach Alert

**Details:**
Fires when an order gets stuck longer than allowed at a specific stage (e.g. India dispatch, US warehouse receiving, 1Click dispatch, final delivery). A task is automatically created to track the resolution, and this message links to it.

**Settings if need to do:**
- Doctype: **LH Slack Settings** → `Enable Consolidated Alerts` + `Consolidated Alerts Channel ID`
- Doctype: **SLA Rule** → each rule needs `Notify on Slack` checked

**Channel name:** Consolidated Alerts Channel

**Image placeholder:**
`[Screenshot Placeholder: SLA Breach Alert]`

**How it can help users:**
Surfaces stuck orders to operations/warehouse/CS teams before they become customer complaints.

---

## 2.4 Quotation Approval Card (Interactive — Two-Way)

**Details:**
When a quotation needs approval, a card is posted with **Approve**, **Send Back**, and **Send to Client** buttons. Clicking a button (or filling a short popup form for "Send Back") updates the quotation in ERPNext immediately — no need to open the system.

**Settings if need to do:**
- Doctype: **LH Slack Settings** → `Enable Quotation Approval`
- Set: `CS Quotation Channel ID`
- Only users with the `Quotation Approvar` role can click the buttons

**Channel name:** CS Quotation Channel

**Image placeholder:**
`[Screenshot Placeholder: Quotation Approval card with buttons]`
`[Screenshot Placeholder: Send Back reason popup]`

**How it can help users:**
Approvers can review and act on quotations directly from Slack — no login required — speeding up the sales approval cycle.

---

# Section 3: Factory / Order Comment Routing (Interactive — Two-Way)

## 3.1 Comment Routed to Channel

**Details:**
When someone comments on an Order or Quotation and @-mentions a person with the Engineer role or CS role, that comment is automatically posted to the matching Slack channel. If someone **replies in the Slack thread**, that reply is automatically added back as a comment in ERPNext — keeping both places in sync.

**Settings if need to do:**
- Doctype: **LH Slack Settings** → `Enable Factory Routing`
- Set: `CS Tech Channel ID` (for Engineer-tagged comments), `CS General Channel ID` / `CS Quotation Channel ID` (for CS-tagged comments)
- Set: `Engineer Role`, `CS Role`, `Factory User` (auto-notified on every factory reply)

**Channel name:** CS Tech Channel / CS General Channel (depends on who's tagged)

**Image placeholder:**
`[Screenshot Placeholder: Comment routed to Slack channel]`
`[Screenshot Placeholder: Slack thread reply synced back to ERPNext]`

**How it can help users:**
Lets CS and factory/engineering staff collaborate in real time without either side needing to constantly check the other system.

---

## 3.2 Tag-to-Note (Interactive — Two-Way)

**Details:**
Mentioning the bot in Slack (or DM-ing it) with an order/quotation number plus a note creates that note as a comment on the order in ERPNext automatically. The bot confirms with "✅ Noted."

**Settings if need to do:**
- Doctype: **LH Slack Settings** → `Enable Tag to Note`

**Channel name:** Any channel the bot is in, or Direct Message

**Image placeholder:**
`[Screenshot Placeholder: Tagging bot to leave a note]`

**How it can help users:**
Quick way to log information on an order without opening ERPNext — useful when working entirely from Slack.

---

## 3.3 Order/Quotation Lookup (Interactive — Two-Way, Read-Only)

**Details:**
Mentioning the bot with just an order or quotation number (no extra text) returns a live status card: customer, status, tracking number, order timeline, latest comment, and item list.

**Settings if need to do:**
- Doctype: **LH Slack Settings** → `Enable Order Lookup` (on by default)

**Channel name:** Any channel the bot is in, or Direct Message

**Image placeholder:**
`[Screenshot Placeholder: Order lookup result card]`

**How it can help users:**
Instant status check from Slack — no need to search for the order in ERPNext.

---

# Section 4: Founder Daily Briefing

## 4.1 Daily Founder Briefing

**Details:**
A once-daily executive summary sent at a configured hour, covering: yesterday's revenue and order count, month-to-date revenue/profit, orders with low margins, stuck orders, open escalations, and products at risk of running out of stock.

**Settings if need to do:**
- Doctype: **Founder Briefing Settings**
- Turn on: `Is Active`
- Set: `Slack Webhook URL`, `Briefing Send Hour (IST)` (default 8 AM)

**Channel name:** `[Placeholder: Founder Briefing Channel]`

**Image placeholder:**
`[Screenshot Placeholder: Daily Founder Briefing message]`

**How it can help users:**
Gives founders/leadership a single daily snapshot of the business without needing to open dashboards.

---

# Section 5: Dispatch & Payment Alerts

## 5.1 Urgent Dispatch Reminder

**Details:**
Daily reminder for Urgent/VIP priority orders approaching or past their promised dispatch date. Fires starting 2 working days before the deadline and every day after (including overdue days) until dispatched.

**Settings if need to do:**
- Doctype: **Access Settings** → `Dispatch Reminder Slack Webhook`

**Channel name:** `[Placeholder: Dispatch Reminders Channel]`

**Image placeholder:**
`[Screenshot Placeholder: Urgent Dispatch Reminder]`

**How it can help users:**
Keeps the warehouse/dispatch team on top of high-priority order deadlines before they're missed.

---

## 5.2 Payment Block Reminder

**Details:**
Fires when an order gets blocked because a payment stage isn't fully paid, then repeats daily. From day 5 onward, it's marked "Escalated to Founders."

**Settings if need to do:**
- Doctype: **Access Settings** → `Urgent Slack Channel` (this field is shared/reused for several payment alerts below)

**Channel name:** `[Placeholder: Payment Alerts Channel]`

**Image placeholder:**
`[Screenshot Placeholder: Payment Block Reminder — Day N]`

**How it can help users:**
Ensures CS follows up on unpaid balances promptly, and founders get visibility once it's overdue.

---

## 5.3 Upcoming Payment Stage Due

**Details:**
Fires once when the previous payment stage is completed and the next stage becomes due.

**Settings if need to do:** Same as 5.2 — **Access Settings** → `Urgent Slack Channel`

**Channel name:** `[Placeholder: Payment Alerts Channel]`

**Image placeholder:**
`[Screenshot Placeholder: Upcoming Payment Stage Due]`

**How it can help users:**
Lets CS collect the next payment proactively instead of waiting for it to become overdue.

---

## 5.4 Before-Shipped Payment Pending

**Details:**
Fires once when an order is marked Shipped but the required "before shipped" payment amount hasn't been fully received.

**Settings if need to do:** Same as 5.2 — **Access Settings** → `Urgent Slack Channel`

**Channel name:** `[Placeholder: Payment Alerts Channel]`

**Image placeholder:**
`[Screenshot Placeholder: Before Shipped Payment Pending]`

**How it can help users:**
Flags a payment gap right after shipping, while there's still a chance to collect before delivery.

---

## 5.5 On-Delivery Payment Pending

**Details:**
Fires once when an order is marked Completed but the "on delivery" payment amount hasn't been fully received.

**Settings if need to do:** Same as 5.2 — **Access Settings** → `Urgent Slack Channel`

**Channel name:** `[Placeholder: Payment Alerts Channel]`

**Image placeholder:**
`[Screenshot Placeholder: On Delivery Payment Pending]`

**How it can help users:**
Last-chance visibility for finance/CS to collect an outstanding balance after delivery.

---

# Section 6: ShipStation (Shipping) Alerts

## 6.1 Order Import Failure

**Details:**
Fires when one or more orders fail to import from ShipStation into ERPNext during a sync.

**Settings if need to do:**
- Doctype: **ShipStation Settings** → `Slack Webhook URL (Failures)`

**Channel name:** `[Placeholder: ShipStation Alerts Channel]`

**Image placeholder:**
`[Screenshot Placeholder: ShipStation Order Import Failure]`

**How it can help users:**
Lets the fulfillment team catch and fix import problems quickly, before an order is missed.

---

## 6.2 Sync Crash Alert

**Details:**
Fires when the background job that syncs with ShipStation crashes unexpectedly.

**Settings if need to do:**
- Doctype: **ShipStation Settings** → `Slack Webhook URL (Failures)` and `Slack Webhook URL (Mark-as-Shipped Success)` (sent to both)

**Channel name:** `[Placeholder: ShipStation Alerts Channel]`

**Image placeholder:**
`[Screenshot Placeholder: ShipStation Sync Crash Alert]`

**How it can help users:**
Alerts engineering/ops immediately when shipping sync breaks, minimizing downtime.

---

## 6.3 Monthly Shipment Count Warning

**Details:**
Fires once per month when the number of shipments tracked through ShipStation crosses 450, as a heads-up to review the subscription plan limit.

**Settings if need to do:**
- Doctype: **ShipStation Settings** → same two webhook fields as 6.2

**Channel name:** `[Placeholder: ShipStation Alerts Channel]`

**Image placeholder:**
`[Screenshot Placeholder: Monthly Shipment Count Warning]`

**How it can help users:**
Gives finance/ops advance notice before hitting the ShipStation plan's shipment limit.

---

## 6.4 Mark-as-Shipped Success

**Details:**
Confirms that tracking info was successfully pushed to ShipStation for an order.

**Settings if need to do:**
- Doctype: **ShipStation Settings** → `Slack Webhook URL (Mark-as-Shipped Success)`

**Channel name:** `[Placeholder: ShipStation Alerts Channel]`

**Image placeholder:**
`[Screenshot Placeholder: Mark-as-Shipped Success]`

**How it can help users:**
Quick confirmation that a shipment update went through correctly.

---

## 6.5 Mark-as-Shipped Failure

**Details:**
Fires when pushing tracking info to ShipStation fails for an order.

**Settings if need to do:** Same as 6.4

**Channel name:** `[Placeholder: ShipStation Alerts Channel]`

**Image placeholder:**
`[Screenshot Placeholder: Mark-as-Shipped Failure]`

**How it can help users:**
Alerts the team to manually fix a failed tracking update so the customer still gets correct info.

---

## 6.6 Orders Missing From ERP

**Details:**
Hourly check for ShipStation orders (from the last 3 hours) that have no matching record in ERPNext.

**Settings if need to do:**
- Doctype: **ShipStation Settings** → `Slack Webhook URL (Mark-as-Shipped Success)`

**Channel name:** `[Placeholder: ShipStation Alerts Channel]`

**Image placeholder:**
`[Screenshot Placeholder: Orders Missing From ERP]`

**How it can help users:**
Catches sync gaps early so no customer order slips through unprocessed.

---

## 6.7 Shopify Refund Unresolved After 24h

**Details:**
Fires when a Shopify refund fails to process and remains unresolved for more than 24 hours; repeats hourly until fixed.

**Settings if need to do:**
- Doctype: **ShipStation Settings** → `Slack Webhook URL (Mark-as-Shipped Success)`

**Channel name:** `[Placeholder: ShipStation Alerts Channel]`

**Image placeholder:**
`[Screenshot Placeholder: Shopify Refund Unresolved Alert]`

**How it can help users:**
Ensures finance/ops manually resolves stuck customer refunds instead of them going unnoticed.

---

# Section 7: HR / Team Alerts (LH HRMS)

## 7.1 Birthday Shoutout

**Details:**
Daily message celebrating any employee(s) with a birthday that day.

**Settings if need to do:**
- Doctype: **LH HRMS Settings** → `Birthday Reminder Enabled` + `Slack Webhook URL`

**Channel name:** `[Placeholder: Company/Birthday Channel]`

**Image placeholder:**
`[Screenshot Placeholder: Birthday Shoutout message]`

**How it can help users:**
Builds team culture — a lightweight, fun, org-wide message.

---

## 7.2 Clock-In Summary

**Details:**
Posted when an employee clocks in for the day, showing their planned tasks for the day.

**Settings if need to do:**
- Doctype: **LH HRMS Settings** → `Daily Log Slack Webhook URL`

**Channel name:** `[Placeholder: Daily Work Log Channel]`

**Image placeholder:**
`[Screenshot Placeholder: Clock-In Summary]`

**How it can help users:**
Gives the team/manager visibility into what each person plans to work on that day.

---

## 7.3 Clock-Out Summary

**Details:**
Posted when an employee clocks out, summarizing hours worked, task-by-task progress, and any missing task entries.

**Settings if need to do:** Same as 7.2 — **LH HRMS Settings** → `Daily Log Slack Webhook URL`

**Channel name:** `[Placeholder: Daily Work Log Channel]`

**Image placeholder:**
`[Screenshot Placeholder: Clock-Out Summary]`

**How it can help users:**
Gives managers a daily record of completed work and flags anything left incomplete.

---

## 7.4 Missing Tasks Added Notice

**Details:**
Posted when an employee or HR adds missing task entries after the fact (via a confirmation step), showing what was added and updated total hours.

**Settings if need to do:**
- Doctype: **LH HRMS Settings** → `Founder Digest Slack Webhook URL` (falls back to Daily Log webhook if not set)

**Channel name:** `[Placeholder: Founder Digest Channel]`

**Image placeholder:**
`[Screenshot Placeholder: Missing Tasks Added Notice]`

**How it can help users:**
Keeps a transparent record when work-log entries are corrected after submission.

---

## 7.5 Missed Clock-In/Out Founder Digest

**Details:**
Once-daily summary listing any employees who missed clocking in or out the previous day.

**Settings if need to do:**
- Doctype: **LH HRMS Settings** → `Founder Digest Slack Webhook URL`, `Founder Digest Hour (IST)` (default 9 AM), `Enable Clock-In/Out Enforcement`

**Channel name:** `[Placeholder: Founder Digest Channel]`

**Image placeholder:**
`[Screenshot Placeholder: Missed Clock-In/Out Digest]`

**How it can help users:**
Gives leadership daily visibility into attendance compliance.

---

## 7.6 Hours Reconciliation Flag

**Details:**
Nightly check comparing logged work hours against time-tracking software (Insightful) hours. Flags significant mismatches per employee, grouped and sent to their manager. Only sends when a flag is new or has changed significantly (to avoid spam).

**Settings if need to do:**
- Doctype: **LH HRMS Settings** → `Reconciliation Slack Webhook URL`, `Enable Hours Reconciliation`, `Insightful Hours Reconciliation Threshold`

**Channel name:** `[Placeholder: Manager/Reconciliation Channel]`

**Image placeholder:**
`[Screenshot Placeholder: Hours Reconciliation Flag]`

**How it can help users:**
Helps managers coach team members on accurate time logging — framed as a coaching signal, not a penalty.

---

## 7.7 Reconciliation Auto-Resolved Notice

**Details:**
Fires when an employee corrects their work log and it brings a previously flagged mismatch back within the acceptable range.

**Settings if need to do:** Same as 7.6

**Channel name:** `[Placeholder: Manager/Reconciliation Channel]`

**Image placeholder:**
`[Screenshot Placeholder: Reconciliation Auto-Resolved]`

**How it can help users:**
Closes the loop automatically so managers know an issue was fixed without manual follow-up.

---

## 7.8 Weekly Points & Recognition Digest

**Details:**
Weekly, company-wide leaderboard showing top point earners and badges awarded that week.

**Settings if need to do:**
- Doctype: **LH HRMS Settings** → `Points Digest Slack Webhook URL`, `Enable Weekly Reports`, `Weekly Report Day`, `Weekly Report Hour (IST)`

**Channel name:** `[Placeholder: Company-Wide Recognition Channel]`

**Image placeholder:**
`[Screenshot Placeholder: Weekly Points & Recognition Digest]`

**How it can help users:**
Recognizes high performers company-wide and encourages engagement — visible to everyone, not just managers.

---

## 7.9 Weekly Report Failure Alert

**Details:**
Internal alert if any scheduled weekly report (like the points digest) fails to run due to an error.

**Settings if need to do:**
- Doctype: **LH HRMS Settings** → `Reports Failure Alert Webhook URL` (falls back to Founder Digest webhook)

**Channel name:** `[Placeholder: Engineering/Ops Alerts Channel]`

**Image placeholder:**
`[Screenshot Placeholder: Weekly Report Failure Alert]`

**How it can help users:**
Ensures engineering/ops knows immediately if an automated report silently breaks.

---

# Quick Reference — All Slack Integrations

| # | Message | Area | Type |
|---|---|---|---|
| 1.1 | Tracking Problem Alert | Order Tracking | One-way |
| 1.2 | US Warehouse Delivered | Order Tracking | One-way |
| 2.1 | 1Click Error Alert | Orders | One-way |
| 2.2 | SKU Auto-Registration Alert | Orders | One-way |
| 2.3 | SLA Breach Alert | Orders | One-way |
| 2.4 | Quotation Approval Card | Quotations | **Two-way** |
| 3.1 | Comment Routed to Channel | Factory/Orders | **Two-way** |
| 3.2 | Tag-to-Note | Factory/Orders | **Two-way** |
| 3.3 | Order/Quotation Lookup | Factory/Orders | **Two-way** |
| 4.1 | Daily Founder Briefing | Leadership | One-way |
| 5.1 | Urgent Dispatch Reminder | Dispatch | One-way |
| 5.2 | Payment Block Reminder | Payments | One-way |
| 5.3 | Upcoming Payment Stage Due | Payments | One-way |
| 5.4 | Before-Shipped Payment Pending | Payments | One-way |
| 5.5 | On-Delivery Payment Pending | Payments | One-way |
| 6.1 | Order Import Failure | ShipStation | One-way |
| 6.2 | Sync Crash Alert | ShipStation | One-way |
| 6.3 | Monthly Shipment Count Warning | ShipStation | One-way |
| 6.4 | Mark-as-Shipped Success | ShipStation | One-way |
| 6.5 | Mark-as-Shipped Failure | ShipStation | One-way |
| 6.6 | Orders Missing From ERP | ShipStation | One-way |
| 6.7 | Shopify Refund Unresolved | ShipStation | One-way |
| 7.1 | Birthday Shoutout | HR | One-way |
| 7.2 | Clock-In Summary | HR | One-way |
| 7.3 | Clock-Out Summary | HR | One-way |
| 7.4 | Missing Tasks Added Notice | HR | One-way |
| 7.5 | Missed Clock-In/Out Digest | HR | One-way |
| 7.6 | Hours Reconciliation Flag | HR | One-way |
| 7.7 | Reconciliation Auto-Resolved | HR | One-way |
| 7.8 | Weekly Points & Recognition Digest | HR | One-way |
| 7.9 | Weekly Report Failure Alert | HR | One-way |

**Total: 30 Slack message types — 26 one-way notifications, 4 two-way interactive features.**

---

## Notes for Admins

- Several settings fields are **shared/reused** across multiple unrelated alerts (e.g. `Access Settings → Urgent Slack Channel` sends 4 different payment alerts; `ShipStation Settings → Slack Webhook URL (Mark-as-Shipped Success)` sends 4 different alerts). If you need these split into separate channels, this requires a code change, not just a settings change.
- `Oneclick Settings → US Warehouse Delivered Slack Webhook` currently does **not** send any message — that alert instead goes through the Consolidated Alerts Channel. This field can be ignored or removed.
