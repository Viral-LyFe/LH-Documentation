# 1Click — Open Risks and Decisions (2026-10-08)

How to use: read each item, then write your answer under **ANS:**. Nothing here has been run against live 1Click. Items are ranked by how much they matter. "Proposed" is what I would do; nothing in this file has been changed in code unless it says so.

Related: `1Click_Oct2026_Changes_Test_Cases_and_Report.md` (what was built and tested today).

---

## A. Likely to bite

### 1. Create Load now depends on the Material Issue for Order (MIFO)
**What happens now:** Create Load lists only what the Factory issued (the order's issued-items list, built from submitted MIFOs). If the order has no submitted MIFO, no Load is sent and only an Error Log entry is written.

**Who is affected:**
- Force US orders that skipped the MIFO.
- US_FULL orders where someone filled `tracking_number_us`.
- Bulk Transfer Batch orders missing a MIFO.

**Why it matters:** the inbound-material email needs both the order and the Load. Those orders will never send it, and nothing alerts you.

**Proposed:** keep the rule (Load = Factory items only), and add one 1Click Slack message when a Load is skipped for this reason, so it is not silent.

**Question:** does every order that comes from the Factory to the US always have a MIFO? Should a skipped Load alert you?

**ANS:**

&nbsp;

---

### 2. The new Route D trigger fires when tracking is filled
**What happens now:** when `tracking_number_us` and `carrier_us` are both filled on a Route D order held "Awaiting India Components", the order is posted to 1Click straight away. The post builds its items from the MIFO.

**Risk:** if the tracking number is entered before the MIFO is submitted, the post fails at once with the missing-MIFO error. The order moves to "1Click Error" and a Slack message goes out. It is recoverable, but it looks like a failure to the team. I have not checked how often the Factory fills tracking before submitting the MIFO.

**Proposed:** if the MIFO is missing when the trigger fires, do nothing (no error status, no alert) and let the later triggers post it once the MIFO exists.

**Question:** in your process, is the MIFO always submitted before the tracking number is entered? If not, should the trigger wait quietly?

**ANS:**

&nbsp;

---

### 3. The "manual 1Click checks" setting is read wrongly in three existing places
**What happens now:** `frappe.db.get_value` on a single settings document returns an unticked checkbox as the text `"0"`, and `"0"` counts as true. So these places treat an unticked box as "manual checks on":
1. The auto-run after a new Lyfe Order is inserted (`after_insert` in `lyfe_order.py`).
2. The re-route after a ShipStation sync (`maybe_reroute_after_shipstation_sync` in `lyfe_order.py`).
3. The tracker's auto-post on "Shipped" (`order_tracking.py`).

**Today:** the box is ticked on this site, so nothing is affected. If someone unticks it, those three paths would still act as if it were ticked. My new Route D trigger was fixed already.

**Impact of fixing:** it would start auto-posting wherever the box is unticked. That is probably what the setting is meant to do, but it is a behaviour change.

**Proposed:** read the setting with `get_single_value` in all three places, with a test for each.

**Question:** fix all three now, or leave until go-live?

**ANS:**

&nbsp;

---

## B. Gaps I know about and left alone (by your decision)

### 4. MIFO versus plan
**What happens now:**
- The posted order and the Load use the MIFO quantities, whatever the Transfer Order or the order says.
- A SKU the Factory issued that was not ordered is posted as if it were ordered.
- A SKU that is in both the US list and the MIFO has its quantities added together.
- After a US stock shortfall, issuing the missing material through a MIFO still gets blocked by the new stock check, because the check ignores the MIFO.
- The Transfer Order's items are only a shipment record. They are never sent to 1Click.

**Proposed (earlier discussion):** compare the MIFO against the order's two lists when posting. On a difference, stop with "1Click Error" and a Slack message, with an "Approve MIFO variance" checkbox to allow a known difference. Treat the quantity for a SKU in both lists as the ordered quantity, not the sum. Skip tube, finish and acrylic rows at first, because their SKU changes.

**Decision recorded:** "leave it, no change required."

**Question:** still leave it? If yes, is the "added together" rule for a SKU in both lists confirmed as intended?

**ANS:**

&nbsp;

---

### 5. Stock can still be taken after the check
The US stock check runs just before Create Order. 1Click has no reservation, so another order can take the stock in between. The window is smaller, not closed.

**Question:** accept this, or should we ask 1Click whether a hold or reservation exists?

**ANS:**

&nbsp;

---

### 6. Mixed "Direct to Customer" has no Load and no inbound email
For this destination only the US part is posted. The Factory part ships straight to the customer and never goes to 1Click. No Load is created because nothing is inbound. I believe that is right, but I have not confirmed it with 1Click. The inbound email is never sent for these orders.

**Question:** is it right that these orders send no inbound email? Does 1Click expect a Load here?

**ANS:**

&nbsp;

---

## C. Things that may not behave as expected

### 7. The shipped-vs-posted check
- **The "Shipped" signal is unconfirmed.** It fires when 1Click's status is "Shipped" or a shipped date is present. Every logged response so far shows "Open", so I have not seen a real shipped response or whether it still lists items.
- **Multi-box orders:** an order reported as shipped before all boxes are listed would give a false alert.
- **One alert per order:** a real problem found later on the same order would stay silent.
- **Polling window:** the hourly sync only polls orders at "Submitted to 1Click" or "US Warehouse Delivered". If 1Click's first "Shipped" response has no items and the order then moves on, the check never runs again.

**Proposed:** look at the first real shipped response and adjust. Possibly allow a second alert when the differences change after the first one is resolved.

**Question:** can you send one real shipped order (name or tracking number) so I can check its response? Should a changed difference alert again after Resolved?

**ANS:**

&nbsp;

---

### 8. `tracking_number_us` is not copied from the Transfer Order
It only goes from the Lyfe Order to the Transfer Order. If the Factory tracking is entered only on the Transfer Order, the Load waits until someone fills the Lyfe Order field.

**Question:** where does the Factory enter the US-leg tracking number in your process?

**ANS:**

&nbsp;

---

### 9. The daily load-received check only looks at orders modified yesterday
`scheduled_check_us_warehouse_load_confirmation` picks orders marked delivered to the US warehouse that were **modified yesterday**. An order that arrives late, or is not modified that day, is never checked. Each order is checked once.

**Proposed:** check every order delivered and not yet confirmed, at least one day old, until it is confirmed.

**Question:** change it, or keep it as is?

**ANS:**

&nbsp;

---

## D. Operations and testing

### 10. Production setup
- The new shipment-difference alert always uses `oneclick_slack_webhook_url`, so it must be set on production.
- The new DocTypes (OneClick Shipment Issue and its child tables, OneClick Shipped Item and Package) need `bench migrate` on production.
- All other 1Click Slack messages go to the Consolidated Alerts Channel when developer mode is off.

**Question:** is the webhook field set on production? Who runs the migrate?

**ANS:**

&nbsp;

---

### 11. Test hygiene
- 7 tests in `test_oneclick_post_reliability.py` fail even without today's changes, because they expect retry settings this site does not use. One test in `test_us_warehouse_delivered_manual_post.py` also fails before and after.
- Test runs leave committed "Test Customer" orders in the database (for example `LYF-MN-2026-0031`, `0035`, `0038`, `0039`). Harmless on this testing site.
- Test runs no longer post to Slack. That guard was added only after some real messages went out.

**Question:** should I fix the 7 failing tests so they use the site's own settings, or leave them?

**ANS:**

&nbsp;

---

### 12. Nothing is committed
All of today's changes sit in the working tree. Other uncommitted changes are also there (`CLAUDE.md`, `lh_slack_settings.json`, `docs/material-issue-for-order.md`).

**Question:** commit now? On which branch? Include or leave out the other uncommitted files?

**ANS:**

&nbsp;
