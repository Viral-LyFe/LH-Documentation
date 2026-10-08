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

**Proposed (superseded by the answer below):** keep the rule (Load = Factory items only) and alert when a Load is skipped.

**Question:** does every order that comes from the Factory to the US always have a MIFO? Should a skipped Load alert you?

**ANS:**
1. A Load should be created only when items are being delivered from the factory to the US warehouse. That Load should contain only the items issued by the factory.
2. No Load should be created for orders that will be delivered entirely from the US warehouse.
3. The email currently sent when an order is posted should not be sent when the order will be delivered entirely from the US warehouse (1Click warehouse). Instead, a different email should be sent that lists the items to be delivered as soon as possible and mentions the PO number.

**Status (2026-10-08):** answers 1 and 2 already match the code (Load = Factory-issued items only; no Load without them). Answer 3 is **built**: see "US-only dispatch email" in the test report. Confirmed: when only the US warehouse delivers and no Factory is involved, the Load is skipped and **no alert** is raised (no Slack message, no Error Log entry).

&nbsp;

---

### 2. The new Route D trigger fires when tracking is filled
**What happens now:** when `tracking_number_us` and `carrier_us` are both filled on a Route D order held "Awaiting India Components", the order is posted to 1Click straight away. The post builds its items from the MIFO.

**Risk:** if the tracking number is entered before the MIFO is submitted, the post fails at once with the missing-MIFO error. The order moves to "1Click Error" and a Slack message goes out. It is recoverable, but it looks like a failure to the team. I have not checked how often the Factory fills tracking before submitting the MIFO.

**Proposed:** if the MIFO is missing when the trigger fires, do nothing (no error status, no alert) and let the later triggers post it once the MIFO exists.

**Question:** in your process, is the MIFO always submitted before the tracking number is entered? If not, should the trigger wait quietly?

**ANS:**
Yes, the MIFO will always be submitted before the tracking number is received from the exporter and entered in the ERP. However, this trigger is still required: if there is no Material Issue, the order should not be posted to 1Click.
This should apply to all orders fulfilled from the factory, regardless of whether they are shipped via the US warehouse or directly to the customer.

**Status (2026-10-08):** the trigger stays as it is (no quiet waiting). Route D and Mixed "Via US Warehouse" were already blocked without a MIFO. Added the same block for Mixed "Direct to Customer": its US portion is no longer posted until the MIFO exists (see test report DM-1). **New question raised:** the earlier rule (commit 475219aa, 2026-09-30) blocks *every* post without a MIFO, including orders that come entirely from US stock — see item 13.

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
Yes, fix.

**Status (2026-10-08):** done. All three places (and the Route D trigger) now read the setting through one helper, `_oneclick_flags()`, which returns real 0/1 numbers (the tracker uses `get_single_value` directly). The `is_active` flag had the same problem and is fixed in the same places. See test report section 2.9. Behaviour change to be aware of: on any site where "Manual 1Click checks" is unticked, those paths now run automatically.

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
Leave it.

**Status (2026-10-08):** no change. The MIFO quantities, the extra-SKU behaviour, the "added together" rule for a SKU in both lists and the Transfer Order being a record only all stay as they are.

&nbsp;

---

### 5. Stock can still be taken after the check
The US stock check runs just before Create Order. 1Click has no reservation, so another order can take the stock in between. The window is smaller, not closed.

**Question:** accept this, or should we ask 1Click whether a hold or reservation exists?

**ANS:**
Not answered directly. Instead the stock checks were switched to the **On Hand** column (see status).

**Findings (2026-10-08, sandbox icomwms.dev, warehouse 10):** the stock API returns `sku`, `onhand`, `available` (no separate reserved/allocated field). All 323 SKUs known to 1Click had zero stock, so a reservation test was not possible; the one test order that would have shown it was not placed (action not permitted). The order split is decided by the field named in Oneclick Settings > Response Available Qty Field Name (`inventory_response_qty_field`).

**Status (2026-10-08):** that setting was changed from `available` to `onhand` on this site, so the order split and the pre-post stock check now compare against On Hand. **Consequence:** On Hand ignores stock already promised to open orders (Demand), so two orders can both see the same units — the window described in this item is wider than with Available. **Not deployed anywhere else:** the setting lives in the site database, so production needs the same change.

&nbsp;

---

### 6. Mixed "Direct to Customer" has no Load and no inbound email
For this destination only the US part is posted. The Factory part ships straight to the customer and never goes to 1Click. No Load is created because nothing is inbound. I believe that is right, but I have not confirmed it with 1Click. The inbound email is never sent for these orders.

**Question:** is it right that these orders send no inbound email? Does 1Click expect a Load here?

**ANS:**
When an order is split into two parts, with two components delivered from the US warehouse and two from the factory, the US warehouse portion should be posted as an Order, but no Load should be created for it. For the second leg, which is delivered directly from the factory to the customer, no PO needs to be posted to 1Click and no Load should be created.

**Status (2026-10-08):** matches the code: the US portion is posted as an order (it needs the MIFO first, item 2), the Factory part is never posted. One edge was closed: the Load builder now returns at once for "Direct to Customer" orders, so a Load can no longer be created for them even if `tracking_number_us` is filled (the Factory-direct MIFO items must never go to 1Click as inbound). **Dispatch email for the US portion — answered:** "In this case we should send email to 1Click for CMC-02 ×10 to deliver." Done: the US portion of a Direct to Customer order now gets the "Dispatch ASAP" email listing only the posted US items (example: CMC-02 ×10, never the Factory-direct CMC-01). See test report DE-7 and DE-8.

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
Provided 16 orders shipped from the 1Click warehouse on 2026-10-07 (LH3097, LH3095, LH3083, LH3079, LH3059, LH3103, LH3100, LH2836, LH2882, LH3099, LH3102, LH3098, LH3058, LH3038, LH3094, LH3080). Once an issue is marked Resolved there should be **no** further alert for that order.

**Findings (2026-10-08, read-only lookups of all 16 in the 1Click sandbox):**
- Status is `"Shipped"` and `Shipped_Date` is filled (it arrives as text like `{ts '2026-10-07 14:05:25'}`), so the shipped signal the check uses is confirmed.
- Every response lists the shipped items with `Qty_Shipped` and the packages.
- **Multi-box:** LH3103-1 and LH3100-1 have 2 packages; their item lists were still complete, so no false "short" alert for multi-box orders in these examples.
- **Comparison:** all 16 shipped quantities match what was posted — zero differences, so zero alerts.
- **Resolved stays quiet:** matches how the check works (one log per order ever; any existing log, open or resolved, stops further logs and alerts). Test report SH-5.
- **These 16 were not checked or stored by the new code:** they shipped before the check existed and their ERP status is already "Shipped", which the hourly sync no longer polls. Their shipped items/packages are therefore not on the orders. For orders shipping from now on, the check runs in the same sync call that first sees "Shipped".

&nbsp;

---

### 8. `tracking_number_us` is not copied from the Transfer Order
It only goes from the Lyfe Order to the Transfer Order. If the Factory tracking is entered only on the Transfer Order, the Load waits until someone fills the Lyfe Order field.

**Question:** where does the Factory enter the US-leg tracking number in your process?

**ANS:**
Tracking number will be added only manually by the user on the Lyfe Order, not on the Transfer Order and not on the Order Leg.

**Status (2026-10-08):** no change needed. The Lyfe Order is the single place the number is entered; it is already copied from there to the Transfer Order and the Order Leg, which is the direction the code works in. The Load trigger and the Route D auto-post both read `tracking_number_us` on the Lyfe Order, so nothing waits on another document.

&nbsp;

---

### 9. The daily load-received check only looks at orders modified yesterday
`scheduled_check_us_warehouse_load_confirmation` picks orders marked delivered to the US warehouse that were **modified yesterday**. An order that arrives late, or is not modified that day, is never checked. Each order is checked once.

**Proposed:** check every order delivered and not yet confirmed, at least one day old, until it is confirmed.

**Question:** change it, or keep it as is?

**ANS:**
Change it as you proposed.

**Status (2026-10-08):** done. The daily check now covers every order delivered to the US warehouse at least one day ago whose receipt 1Click has not confirmed, and checks it again every day until 1Click shows it received (then it is marked confirmed and never checked again). The "not confirmed" Slack alert is still sent only once per order (new hidden field `us_warehouse_load_alert_sent`), so a day-by-day recheck does not repeat it. **One addition you did not ask for:** orders delivered more than 30 days ago are no longer checked (constant `_LOAD_CONFIRM_MAX_AGE_DAYS`), so a lost order is not polled forever and an old backlog does not flood Slack on the first run. Say if you want that cap changed or removed. Test report LC-1 to LC-4.

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
Yes, do fix.

**Status (2026-10-08):** done. All 36 tests in `test_oneclick_post_reliability.py` now pass: they pin the retry and wait settings they assert on (5 attempts, 2-second waits) instead of reading the site's values, and one Load-retry test now also provides the Factory-issued item the Load requires. The failing test in `test_us_warehouse_delivered_manual_post.py` now pins "auto post on delivery" off, since it checks the manual checkpoint. Still failing, unrelated and not touched: the 4 tests in `test_factory_to_us_order_leg.py` (a missing-Item link error when the Transfer Order is created). The leftover "Test Customer" orders in the database are not cleaned up.

&nbsp;

---

### 12. Nothing is committed
All of today's changes sit in the working tree. Other uncommitted changes are also there (`CLAUDE.md`, `lh_slack_settings.json`, `docs/material-issue-for-order.md`).

**Question:** commit now? On which branch? Include or leave out the other uncommitted files?

**ANS:**

&nbsp;

---

## 13. (New) Orders from US stock only cannot be posted without a MIFO
**What happens now:** since the 2026-09-30 change "Block 1Click posting when cj_shipment_items is empty", `_submit_single_oneclick_order` raises the missing-MIFO error for **every** order, including US_FULL and Force US orders where nothing comes from the Factory. If a Factory MIFO is never created for such an order, it can never post. On this site no US_FULL order has been posted yet, so I could not see it in real data.

**Why it matters:** it conflicts with your rule "no Load, and a Dispatch-ASAP email, for orders delivered entirely from the US warehouse". Those orders would not reach that point, and any order that does have a MIFO is treated as Factory-sourced (so it gets a Load and not the Dispatch email).

**Proposed:** only require the MIFO when the Factory is involved (the order has Factory items: Route D, Mixed, or Factory items on a Force US order). For an order fully in US stock, post from its US items (`us_warehouse_shipment_items`) without a MIFO, and send the Dispatch-ASAP email.

**Question:** should an order delivered entirely from US stock be allowed to post without a MIFO?

**ANS:**
MIFO we are using to issue the material from the factory Store rack, so when all components are available in the US warehouse and will be delivered from the US warehouse, that will never be assigned to Factory and they will not create a MIFO.

**Status (2026-10-08):** done. The MIFO is now required only when the Factory is involved. An order delivered entirely from US stock posts from its US items (or, for Force US, from its exploded order items) without a MIFO, gets no Load, and gets the "Dispatch ASAP" email. See test report DM-2 to DM-4.

&nbsp;
