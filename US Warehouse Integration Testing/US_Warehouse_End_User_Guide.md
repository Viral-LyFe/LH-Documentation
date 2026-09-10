# US Warehouse Order Fulfillment — End User Guide

A plain-language guide to how orders flow between the Factory (India) and
the US Warehouse (via 1Click), and what CS/Factory should do in each
situation. No technical terms — just "when you see X, do Y."

---

## The big picture

Every order automatically gets checked to see where it should ship from:

- **All items in stock in the US warehouse** → order is posted straight to
  1Click. No factory involvement needed.
- **No items in the US warehouse** → order goes to the Factory as normal
  (build and ship from India).
- **Some items in the US warehouse, some not** → this is a **Mixed order**.
  CS confirms how to split it (see Use Case 4 below).

You never need to manually decide which of these applies — the system
checks real, live 1Click stock and decides automatically. You only step in
for the situations below.

---

## Use Case 1 — A normal order, fully available in the US warehouse

**What happens automatically:**
The order is posted to 1Click right away. Status shows **"Submitted to
1Click."**

**What you need to do:**
Nothing. Just track it like any other order. Once 1Click ships it, the
tracking number comes in automatically and the order moves to **"Shipped"**,
then **"Completed"** once delivered.

---

## Use Case 2 — A normal order, nothing available in the US warehouse

**What happens automatically:**
The order goes to the Factory like a standard India order — Factory
Assignment → Ready for Dispatch → Shipped → Completed.

**What you need to do:**
Nothing different — treat it exactly like any other Factory order.

---

## Use Case 3 — Changing your mind: route an India order through the US
warehouse instead

**What it means:**
An order was going to ship directly from India, but you've decided it
should go through the US warehouse first instead (e.g. to combine with
other items, or speed up final delivery).

**What you need to do:**
1. Open the order (it must not have shipped yet).
2. Click **"Route via US Warehouse Instead."**
3. Confirm the popup.

**What happens after:**
The order switches to the "ship via US warehouse first" plan. A new
internal shipping record is created to track the goods moving from the
Factory to the US warehouse. The order otherwise still looks and behaves
like a normal Factory order.

---

## Use Case 4 — Mixed order (some items in US warehouse, some not)

**What it means:**
Part of the order can ship immediately from the US warehouse. The rest has
to come from the Factory first.

**What you need to do:**
1. Open the order and go to the **Warehouse Split** section.
2. Confirm the split — the system shows you exactly which items are US-side
   and which are Factory-side.
3. Choose how the Factory portion should travel:
   - **"Direct to Customer"** — Factory ships its portion straight to the
     customer, and the US warehouse ships its portion separately. The
     customer gets two packages.
   - **"Via US Warehouse"** — Factory ships its portion to our US warehouse
     first, it gets combined with the US-stock items, and the whole order
     goes out from the US warehouse as one shipment.
4. Once confirmed, the Factory portion is created as a **Transfer Order**
   (if "Via US Warehouse") and you track it like a normal Factory shipment.

**One rule to know:** a customer should never receive two separate boxes
for what they think is one order unless you specifically chose "Direct to
Customer." The system does not silently split shipments on its own.

---

## Use Case 5 — Factory shipment to the US warehouse (the "Via US
Warehouse" case)

**What it means:**
This only applies after a Mixed order (Use Case 4) is confirmed as "Via US
Warehouse." Factory now needs to ship their portion to our own US
warehouse — not to the customer.

**What Factory needs to do — the real, correct process:**
1. Factory ships the package to the US warehouse address (already filled in
   on the order).
2. **Factory enters the real tracking number and carrier into the
   "Tracking Number (US)" and "Carrier (US)" fields on the order** — this is
   the only manual step needed.
3. **Do not manually change the order's status.** The system watches that
   tracking number and, once the carrier confirms delivery to the US
   warehouse, automatically updates the order to **"US Warehouse
   Delivered."**

**If Factory needs to correct the status manually** (rare — only if the
tracking number can't be used for some reason): change the status directly
to **"US Warehouse Delivered."** Never leave it on "Received" — that status
means something different in the system and does not trigger the next step
correctly.

**Worth knowing:** marking the internal shipment "Received" reflects our
own team's belief that the goods arrived — it is not 1Click confirming they
physically checked the stock into their own system. In practice the order
books with 1Click almost instantly afterward, but this is a known small
timing gap, not something to worry about day to day.

---

## Use Case 6 — Order shows "US Warehouse Delivered" — posting the final
order to 1Click

**What it means:**
The Factory portion has arrived at the US warehouse. Everything for this
order (Factory items + any US-stock items) is now physically together and
ready to go out to the customer — but the order has **not** been posted to
1Click yet. This is deliberate — it needs a human check before it goes out.

**What you need to do:**
1. Open the order. You'll see a **"Post to 1Click"** button (only appears
   when status is "US Warehouse Delivered").
2. Do a quick sanity check — item quantities look right, shipping address
   is correct.
3. Click **"Post to 1Click."** Confirm when prompted.
4. The order status updates to **"Submitted to 1Click"** and the outbound
   shipment process starts automatically from there — tracking number and
   delivery status will update on their own after this point.

**Note:** there is a second, older button called **"Post US Portion to
1Click"** on some orders (Warehouse Split section) — that one is for a
different, older scenario (Direct to Customer split) and posts only part of
the order. Don't confuse the two. For "US Warehouse Delivered" orders,
always use the plain **"Post to 1Click"** button.

---

## Use Case 7 — "Force US" — manually deciding an order should ship from
the US warehouse

**When to use this:**
The system's automatic check said an item isn't available in the US
warehouse, but you have a reason to believe it should still ship from
there (e.g. you just confirmed stock directly with 1Click, or a
correction is needed).

**What you need to do:**
1. Open the order.
2. Use the **route override** option and choose **"Force US."**
3. Enter a reason (required — the system will not let you save without
   one).
4. Confirm. The system will attempt to post directly to 1Click.

**If it fails:** the order will show an error explaining why (e.g. genuinely
no stock). Don't retry blindly — check with the 1Click team if you believe
the stock should be there.

**One rule to know:** Force US cannot be used on an order that has already
gone through a Mixed-order split (Use Case 4) with a Factory portion
assigned. If you see an error about this, it means the order is already
committed to a Mixed shipping plan — clear that plan first if you really
want to switch it to Force US.

---

## Use Case 7b — "Force India" — manually deciding an order should ship
from India instead

**When to use this:**
The opposite of Force US — the item is sitting in the US warehouse, but you
want it to ship from India instead for some reason.

**What you need to do:**
1. Open the order.
2. Use the route override option and choose **"Force India."**
3. Enter a reason (required, same as Force US).
4. Confirm.

**What happens after:**
The order routes to Factory/India as normal. No order is created on 1Click,
even though stock was available there. Your reason is saved and visible on
the order for later reference.

---

## Use Case 8 — Tracking updates aren't showing / order status hasn't
moved

**What it means:**
The system checks tracking automatically in the background (roughly every
few hours, more often right after an order ships). If nothing has updated
in a day or two, it's worth a manual check.

**What you need to do:**
1. Open the order and click **"Track Order"** — this manually re-checks the
   tracking status right now instead of waiting for the next automatic
   check.
2. Refresh the page after a few seconds — the first click sometimes shows
   an in-between status, and the real final status appears on refresh.
3. If tracking still shows nothing after a manual check, the carrier likely
   hasn't scanned the package yet — this is normal in the first day after
   shipping, not a system issue.

---

## Use Case 9 — A brand-new item that 1Click doesn't recognize yet

**What happens automatically:**
If an order contains an item 1Click has never seen before, the system
automatically registers it with 1Click first, then posts the order. You
don't need to do anything for this to happen.

**What you need to do:**
Nothing, in the normal case. If the order still ends up in "1Click Error"
after this (see Use Case 10), escalate — there may be a genuine problem
with that item's setup.

**Also worth knowing:** if the SKU field on an order item is blank but the
item's name contains a recognizable code (e.g. a code in parentheses), the
system can often figure out the right SKU on its own. This happens
automatically in the background — nothing to do here unless an item is
still stuck unresolved.

---

## Use Case 10 — Order stuck in "1Click Error"

**What it means:**
The system tried to check stock or post the order to 1Click, and something
went wrong — this covers a few different real situations:

- 1Click reported success on the surface but actually rejected the order
  behind the scenes (the system always checks for this, never just trusts
  a generic "OK").
- 1Click didn't respond at all (timed out).
- An item code wasn't recognized (usually auto-resolves itself — see Use
  Case 9).
- The order was accidentally submitted a second time (1Click itself
  currently blocks true duplicates, so this is safe, but it will show an
  error rather than silently succeeding).

**What you need to do:**
1. Open the order and look at the error message shown on the order.
2. If it mentions a missing/unregistered item — the system usually
   auto-registers it and retries by itself. Wait a few minutes and refresh.
3. If it's still stuck, use the **"Recheck Inventory"** button on the order.
   This re-runs the stock check from scratch.
4. If it keeps failing, escalate to the technical team with the order
   number — don't manually change the status.

**Important:** never manually mark this order as if it were successfully
posted. Let the system resolve it or escalate. Also — an order sitting in
"1Click Error" never gets left in limbo indefinitely; it always ends up in
this clearly-flagged state so someone can look at it, rather than silently
stuck with no explanation.

---

## Use Case 11 — Cancelling an order that's already been sent to 1Click

**What it means:**
A customer wants to cancel after the order has already been posted to
1Click.

**What you need to do:**
Follow the normal manual cancellation process with CS as you do today —
this still requires a person to coordinate the cancellation with 1Click
directly. There is currently an automatic alert that can fire when this
situation is detected, so the right people are notified quickly, but the
actual cancellation itself is still a manual step.

---

## Use Case 12 — An order sits too long without shipping (SLA follow-up)

**What happens automatically:**
Once an order is posted to 1Click, the system tracks how long it's been
sitting without a real dispatch/tracking number. If it's been more than 72
hours with no movement, a follow-up task is automatically created for
Factory/Ops to chase it down — you don't need to remember to check this
yourself.

The same kind of automatic follow-up also applies to:
- An order waiting on a Factory-to-US-warehouse shipment with no tracking
  number entered after 48 hours.
- A shipment marked "Shipped" toward the US warehouse but not yet marked
  "Received" after 24 business hours.

**What you need to do:**
Nothing to set up — these checks run automatically and post an alert (email
task, and in some cases a Slack message) the moment something is stuck.
Just act on the task/alert when you see one; it will close itself
automatically once the order catches up (tracking number added, dispatched,
or received).

---

## Quick reference — status meanings

| Status | What it means | Who acts next |
|---|---|---|
| New | Just created, not yet routed | System (automatic) |
| Factory Assignment | Going to Factory for production | Factory |
| Ready for Dispatch | Factory has it ready to ship | Factory / CS |
| Submitted to 1Click | Posted to 1Click, waiting on shipment | System (automatic) |
| US Warehouse Delivered | Factory's portion has arrived at the US warehouse; needs a manual "Post to 1Click" click | CS |
| 1Click Error | Something needs a human look | CS / escalate |
| Shipped | On the way to the customer | System (automatic) |
| Completed | Delivered to the customer | Nobody — done |

---

## When in doubt

- **Never manually change status to skip a step** — every automatic step
  (stock check, posting to 1Click, tracking updates) exists to keep data
  accurate. If something looks stuck, use the provided buttons
  ("Recheck Inventory," "Track Order," "Post to 1Click") rather than
  editing fields directly.
- **If an order looks wrong or stuck for more than a day**, escalate with
  the order number rather than trying to fix it manually — some of these
  steps talk to 1Click's live system, and manual edits can create a
  mismatch between our records and 1Click's.
