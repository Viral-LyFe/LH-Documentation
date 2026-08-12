# Order Return Process — User Guide

This guide explains how to handle a customer return, refund, or reshipment on a **Lyfe Order**.

---

## When do I use this?

Use the Return process whenever a customer wants to:
- Return an item for a **refund**
- Return an item to get a **replacement shipped again** (reship)
- Return an item for **both** — a partial refund and a replacement

---

## Step 1: Start the Return

Open the relevant **Lyfe Order** and click the **"Mark Return InProgress"** button.

> 📷 *[Screenshot: "Mark Return InProgress" button on the Lyfe Order]*

A dialog will open asking you to fill in the return details:

- **Return Reason** — choose the reason from the list (Preference, Damaged, Exchange, Wrong Item Received, or Others). If you choose **Others**, you must type in a short explanation.
- **Return Initiated Date** — the date the customer told you they want to return the item.
- **Return Received Date** — fill this in once the returned item physically arrives back at the warehouse (you can leave this blank when you first open the dialog and come back to fill it later).
- **Items table** — you'll see all the items on the order. Enter the **Return Qty** for each item the customer is returning. By default it shows the full quantity, so update it if only part of the order is being returned.
- **Refund details** (if applicable) — refund status, refund amount, and refund date.
- **Reship details** (if applicable) — reship status and reship date.
- **Restock Fee %** — if you're deducting a restocking fee from the refund, enter the percentage and the fee amount will be calculated for you.

> 📷 *[Screenshot: Return Details dialog — reason, dates, items table, refund/reship fields]*

Once you save this dialog, the order moves to the **"Return InProgress"** status, and this is recorded in the order's return history.

> 📷 *[Screenshot: Order status showing "Return InProgress"]*

> You don't need to fill in every field the first time — you can open this dialog again later to update information as the return progresses (e.g., add the received date once the package arrives).

---

## Step 2: Update as the Return Progresses

As the situation develops, reopen the return dialog to keep the order updated:

- Mark the **Return Received Date** once the item is back in-house.
- Update the **Refund Status** and **Refund Amount** once a refund is processed.
- Update the **Reship Status** and **Reship Date** once a replacement is sent out.

The system automatically determines whether this is a **Refund**, **Reship**, or **Refund & Reship** case based on which of these fields you've filled in — you don't need to select this yourself.

> 📷 *[Screenshot: Return Details dialog with refund/reship fields filled in]*

---

## Step 3: Mark the Return as Successful

Once everything for this return has been handled (refund issued and/or replacement shipped), click **"Mark Return Successful"**.

> 📷 *[Screenshot: "Mark Return Successful" button]*

The system will check that all the necessary information for this type of return has been filled in:
- For a **Refund**, it checks that refund details are complete.
- For a **Reship**, it checks that reship details are complete.
- For **Refund & Reship**, it checks both.
- It also confirms that at least one item has a return quantity entered.

If anything required is missing, you'll be prompted to fill it in before the order can move to **"Return Successfully"** status.

> 📷 *[Screenshot: Order status showing "Return Successfully"]*

---

## Sending a Replacement Again (Dispatch Again)

If a replacement needs to be dispatched, use the **"Dispatch Again"** option and enter the date it's being dispatched. This clears the old tracking/carrier information on the order so the team can enter fresh shipping details for the new shipment.

> 📷 *[Screenshot: "Dispatch Again" dialog]*

---

## Understanding the Order's Return History

Every time a return cycle is started (via "Mark Return InProgress") or a replacement is dispatched again, a new entry is added to the order's **Return / RTO History**. This gives you a complete record of every return cycle this order has gone through — useful if a single order has been returned, reshipped, and returned again.

This history also tracks whether a particular cycle was a **Return to Origin (RTO)** — meaning the shipment never reached the customer and came back on its own — versus a standard customer-initiated return. This distinction is for record-keeping and reporting; it does not change how you process the return.

> 📷 *[Screenshot: Return / RTO History table on the order]*

---

## Where to See Return Information

- **On the Order itself:** the "Return Details" section shows the current status of the return, refund, and reship for that order.
- **Order list view:** you can filter orders by status to see all orders that are currently **"Return InProgress"** or **"Return Successfully."**
- **Returns Dashboard:** provides a summary view of return volumes, trends over time, and recent returns across all orders.

> 📷 *[Screenshot: Returns Dashboard overview]*

> Note: A couple of tiles on the Returns Dashboard and the Founder Dashboard (related to SLA breach percentage and pending refund/reship counts) are currently not displaying accurate numbers. Don't rely on those specific figures until they've been fixed — the core return workflow on individual orders works correctly regardless.

---

## Quick Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Click "Mark Return InProgress" and fill in return details | Order status → Return InProgress |
| 2 | Update refund/reship info as it happens | Order stays in Return InProgress |
| 3 | Click "Mark Return Successful" once complete | Order status → Return Successfully |
| (if needed) | Click "Dispatch Again" | Clears old tracking, ready for new shipment info |
