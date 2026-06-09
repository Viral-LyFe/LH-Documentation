# ShipStation Sync Monitoring & Order Verification

**Audience:** Operations / Customer Service Team
**Last Updated:** June 2026

---

## What This System Does

This system automatically watches over two things every time orders sync from ShipStation into ERP:

1. **Alerts you if the sync itself crashes** — so you know immediately when something went wrong at a technical level.
2. **Verifies that recent Shopify orders actually made it into ERP** — so no customer orders slip through unnoticed.

---

## How Orders Normally Flow

```
Customer places order on Shopify
        ↓
ShipStation picks it up
        ↓
ERP syncs from ShipStation every 30 minutes
        ↓
Lyfe Order + ShipStation Order created in ERP
```

If anything breaks in this chain, orders can get lost. This system detects that and emails you.

---

## Alert 1 — Sync Crash Notification

### When does this trigger?

If the entire ShipStation sync job fails to run — for example due to a network outage, API being down, or a server error — you will receive an email immediately.

### What the email looks like

- **Subject:** `[ShipStation] Sync crashed — <date and time>`
- **Body:** States when the sync started and that it crashed, with technical error details included.

### Who receives it?

Users listed under **Access Settings → Notify on ShipStation Order Creation Failure** in ERP.

### What should you do?

Contact your technical team and share the email. The sync may recover on its own at the next 30-minute interval, but if the problem persists the technical team needs to investigate.

---

## Alert 2 — Missing Order Verification

### When does this run?

Automatically, **30 seconds after every successful sync**. It runs quietly in the background — you only hear from it if something is wrong.

### What does it check?

It looks at the **10 most recent orders on Shopify** and checks two things for each one:

| Check | What it means |
|-------|---------------|
| Is there a **Lyfe Order** in ERP for this Shopify order? | The order was received and created in ERP |
| Is that Lyfe Order linked to a **ShipStation Order**? | ShipStation has the order ready for fulfilment |

### What the email looks like

- **Subject:** `[ShipStation] 3 recent Shopify order(s) missing in ERP` *(number varies)*
- **Body:** A table listing which Shopify order numbers are missing and what specifically is missing for each one.

**Example:**

| Shopify Order # | Shopify ID | Missing In |
|-----------------|------------|------------|
| 5901 | 7661234567890 | Lyfe Order |
| 5899 | 7660987654321 | ShipStation Orders (Lyfe Order exists but no ShipStation link) |

### Who receives it?

Same users as Alert 1 — **Access Settings → Notify on ShipStation Order Creation Failure**.

### What should you do?

1. Note the Shopify order numbers from the email.
2. Check those orders on Shopify to confirm they are real, paid orders (not test orders or cancelled orders).
3. If they are real orders, contact your technical team — the orders need to be manually synced or investigated.

---

## How to Manage Who Receives These Alerts

1. Go to **ERP → Access Settings**
2. Scroll to the section **"Notify on ShipStation Order Creation Failure"**
3. Add or remove users from that list
4. Save

Any user listed there will receive both types of alert emails.

---

## Summary

| Alert | Trigger | Who Gets It | Action Required |
|-------|---------|-------------|-----------------|
| Sync Crash | Sync job fails entirely | Alert users in Access Settings | Contact tech team |
| Missing Orders | After every sync — checks last 10 Shopify orders | Alert users in Access Settings | Verify orders on Shopify, contact tech team if real |

Both alerts are **automatic** — no manual steps are needed to activate them. If you are not receiving alerts and believe you should be, ask your admin to add you to the Access Settings list.
