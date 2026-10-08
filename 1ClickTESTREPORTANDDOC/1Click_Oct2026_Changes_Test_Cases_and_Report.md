# 1Click Changes of 2026-10-08 — Behaviour, Test Cases and Test Report

Site tested: `lyfelocal.com` (testing site, developer mode on). All 1Click calls in the tests are mocked — nothing here was run against live 1Click.

---

## 1. What changed (plain language)

| # | Change | What now happens |
|---|---|---|
| A | **Route D auto-post on US tracking** | An order routed India → US → Customer and held as "Awaiting India Components" is posted to 1Click (unknown SKUs registered first, then Create Order, then Create Load) the moment **both** `tracking_number_us` and `carrier_us` are filled and saved — instead of waiting for the hourly carrier poll. Only when Oneclick Settings is active and "manual 1Click checks" is **off**. |
| B | **Create Load = Factory items only** | Create Load lists only what physically left the Factory (net quantity from the submitted Material Issue for Order). Create Order still lists every item to be delivered to the customer. If nothing was issued by the Factory (US warehouse only), no Load is sent — silently, with no alert. |
| C | **US stock re-check before posting** | Before Create Order, the US-covered items are checked against 1Click's available quantity. If short, the order is **not posted**: status → "1Click Error", the reason is stored on the order, and one 1Click Slack message lists each short SKU. Factory-issued items are never checked. A failed inventory call never blocks posting. |
| D | **Shipped-vs-posted difference log** | Once 1Click reports an order shipped, the SKUs/quantities it shipped are compared with what was posted. Any difference creates one **OneClick Shipment Issue** log (status *Issue* → *Resolved* by hand) and sends **one** Slack alert to the 1Click webhook. |
| E | **Shipped items and packages stored on the order** | The hourly 1Click tracking sync saves the shipped SKUs/quantities and the packages (tracking, size, weight) on the Lyfe Order (1Click tab). |
| F | **Slack routing** | All 1Click Slack messages go through one helper. Developer mode on → only the `oneclick_slack_webhook_url` webhook (nothing if blank). Developer mode off → Consolidated Alerts Channel as before. The shipment-difference alert always uses the webhook. Test runs never send to Slack. |
| H | **US-only dispatch email** | An order delivered entirely from the US warehouse (no Factory items in the posted order, not in a batch; this includes the US part of a Mixed "Direct to Customer" order) gets no Load and no inbound email. Instead, once it is posted to 1Click, **one** email goes to the 1Click support address: "Dispatch ASAP — PO <number>" with a table of the SKUs, quantities and descriptions posted. Sent once per order (`dispatch_email_sent_at`); a failed send is released so it can retry, and a Slack message reports the failure. |
| G | **Unreadable load response logged** | If the saved 1Click load-status text on an order cannot be read, the hourly job now writes an Error Log entry with the exact error instead of ignoring it. |

### Worked example — order LH8989
Items: CMC-01 ×5 (coming from the Factory), CMC-02 ×10 (already in US stock). Destination "Via US Warehouse".

| Step | CMC-01 | CMC-02 | Note |
|---|--:|--:|---|
| Transfer Order (shipment record only) | 5 | — | Not sent to 1Click |
| Stock re-check before posting | not checked | needs 10, 1Click must show ≥ 10 | Short → 1Click Error + Slack, nothing posted |
| Create Order (to customer) | 5 | 10 | Everything the customer receives |
| Create Load (inbound) | 5 | — | Only the Factory dispatch |
| After 1Click ships | shipped 5 | shipped 8 | → Issue log: CMC-02 **Short** (posted 10, shipped 8) + one Slack alert |

---

## 2. Test cases

Automated tests live in `apps/lh/lh/lyfe_hardware/`. Every case below passed on 2026-10-08 unless noted.

### 2.1 Route D auto-post on US tracking — `test_oneclick_oct2026_scenarios.py` (TestRouteDAutoPostOnUsTracking)

| ID | Scenario | Expected | Test |
|---|---|---|---|
| RD-1 | Route D order held "Awaiting India Components"; both tracking fields filled and saved | The 1Click post is queued once for that order | `test_queues_post_when_both_tracking_fields_filled` |
| RD-2 | Carrier left blank | Nothing queued | `test_not_queued_without_carrier` |
| RD-3 | "Manual 1Click checks" is on | Nothing queued | `test_not_queued_when_manual_1click_checks_on` |
| RD-4 | Order already posted (has a 1Click order ID) | Nothing queued | `test_not_queued_when_already_posted` |
| RD-5 | Another route (US_FULL) | Nothing queued | `test_not_queued_for_other_routes` |

### 2.2 Create Load content and US stock re-check — `test_oneclick_post_reliability.py`

| ID | Scenario | Expected | Test |
|---|---|---|---|
| LD-1 | Order payload has a Factory SKU and a US-stocked SKU | Load contains only the Factory SKU | `test_load_has_only_factory_items_not_whole_order` |
| LD-2 | No Factory-issued items (US warehouse only) | No Load sent, no alert | `test_load_skipped_when_nothing_issued_by_factory` |
| ST-1 | US item short in 1Click | Raises the shortfall error, only US SKU queried, order status "1Click Error", details stored, Slack message sent | `test_short_stock_blocks_sets_status_and_sends_slack` |
| ST-2 | Same shortfall on the Mixed "Direct to Customer" US leg | Status left unchanged, Slack still sent | `test_us_leg_shortfall_leaves_status_alone` |
| ST-3 | US stock covers the order | Passes | `test_enough_stock_passes` |
| ST-4 | Order with no US-covered items (Force US / Route D) | No inventory call at all | `test_no_us_items_makes_no_call` |
| ST-5 | Inventory call fails | Does not block posting | `test_failed_inventory_call_does_not_block` |

### 2.3 Shipment difference log — `test_oneclick_shipment_issue.py`

| ID | Scenario | Expected | Test |
|---|---|---|---|
| SH-1 | Posted A×3, B×10, C×1; shipped A×3, B×8, X×1; checked twice | One log with B **Short** (−2), C **Missing** (−1), X **Extra** (+1); exactly one Slack alert, sent to the webhook only | `test_differences_logged_once_with_one_alert` |
| SH-2 | 1Click status still "Open" | No log, no alert | `test_nothing_when_not_shipped_yet` |
| SH-3 | Shipped items equal posted (SKU case differs) | No log, no alert | `test_nothing_when_everything_matches` |
| SH-5 | Log marked Resolved, then 1Click data differs again (even a different difference) | No new log, no Slack alert | `test_no_new_log_or_alert_after_resolved` |
| SH-4 | Log set to Resolved | Resolved time and user stamped | `test_resolving_stamps_who_and_when` |

### 2.4 Slack routing and logging — `test_oneclick_oct2026_scenarios.py`

| ID | Scenario | Expected | Test |
|---|---|---|---|
| SL-1 | Developer mode on | Posts to the webhook only | `test_dev_mode_goes_to_webhook_only` |
| SL-2 | Developer mode off | Posts to the Consolidated Alerts Channel via the bot only | `test_production_goes_to_bot_channel` |
| SL-3 | `webhook_only` requested, any mode | Webhook only | `test_webhook_only_ignores_mode` |
| SL-4 | Developer mode on, webhook blank | Nothing sent | `test_dev_mode_without_webhook_sends_nothing` |
| SL-5 | Running inside a test | Nothing sent | `test_test_runs_never_send` |
| LG-1 | Saved load-status text is not valid JSON | Error Log entry with the order name and the exact error; order still re-checked | `test_bad_saved_response_is_logged_with_traceback_and_order_still_checked` |

### 2.5 US-only dispatch email — `test_oneclick_oct2026_scenarios.py` (TestUsOnlyDispatchEmail)

| ID | Scenario | Expected | Test |
|---|---|---|---|
| DE-1 | US-only order posted (PO LH8989, CMC-02 ×10, A-1 ×2) | One email to the support address, subject has the PO, body lists the SKUs/quantities and says "as soon as possible"; a repeat never re-sends | `test_sent_once_with_po_and_items` |
| DE-2 | Order has Factory-issued items | No dispatch email (it goes the Load / inbound route) | `test_not_sent_when_factory_issued_items_exist` |
| DE-7 | Mixed "Direct to Customer" order (MIFO for the Factory-direct CMC-01 ×5 exists); US part CMC-02 ×10 posted | Dispatch email is sent and lists only CMC-02 (never CMC-01) | `test_sent_for_us_part_of_direct_to_customer_order_with_only_us_items` |
| DE-8 | Mixed "Via US Warehouse" order with Factory items | No dispatch email (goes the Load / inbound route) | `test_not_sent_when_via_us_warehouse_has_factory_items` |
| DE-3 | Order not posted to 1Click | No email | `test_not_sent_when_not_posted` |
| DE-4 | Order is in a Bulk Transfer Batch | No email | `test_not_sent_for_batch_orders` |
| DE-5 | Mail sending fails | The once-only flag is released so it can retry | `test_failed_send_is_released_for_retry` |
| DE-6 | Posting an order (`save_oneclick_response`) | Triggers the dispatch-email check with the posted payload | `test_save_oneclick_response_triggers_it` |

### 2.6 No MIFO, no post — Mixed "Direct to Customer" US portion — `test_oneclick_oct2026_scenarios.py` (TestNoMifoNoPostForDirectToCustomerUsLeg)

| ID | Scenario | Expected | Test |
|---|---|---|---|
| DM-1 | Confirmed Mixed order, US portion present, no Material Issue for Order submitted; "Post US Portion to 1Click" | Blocked with the missing-MIFO message; nothing sent to 1Click; the order is not marked posted | `test_blocked_without_mifo` |

Confirm Split itself no longer fails when the MIFO is missing: it confirms the split and tells the user the US portion will wait for the MIFO and can then be posted with **Post US Portion to 1Click**. (Checked by reading the code; not covered by an automated test.)

### 2.7 MIFO required only when the Factory is involved — `test_oneclick_oct2026_scenarios.py` (TestUsOnlyOrderPostsWithoutMifo)

The Factory is "involved" when the order has Factory-issued items (a MIFO), Factory shipment items, or a Route D / Mixed routing outcome.

| ID | Scenario | Expected | Test |
|---|---|---|---|
| DM-2 | US_FULL order, CMC-02 ×10 in US stock, no MIFO | Posts to 1Click with CMC-02 ×10 | `test_us_only_order_posts_us_items_without_mifo` |
| DM-3 | Route D order, no MIFO | Blocked with the missing-MIFO error; nothing sent | `test_factory_involved_order_is_blocked_without_mifo` |
| DM-4 | Order with a MIFO (CMC-01 ×5) and US items (CMC-02 ×10) | Posts both | `test_order_with_factory_items_posts_us_plus_mifo` |

### 2.8 The "Manual 1Click checks" setting is read as a real 0/1 — `test_oneclick_oct2026_scenarios.py` (TestManualChecksSettingIsReadAsRealZeroOne)

An unticked checkbox is stored as the text `"0"`, which Python treats as true. These tests set the setting both ways and check each automatic path.

| ID | Scenario | Expected | Test |
|---|---|---|---|
| MC-1 | Read the setting as ticked and as unticked | Real numbers 1 and 0 | `test_flags_are_ints` |
| MC-2 | New ShipStation-linked order inserted, box **unticked** | Auto-run of 1Click fulfilment is queued | `test_after_insert_auto_runs_when_manual_checks_unticked` |
| MC-3 | Same, box **ticked** | Nothing queued | `test_after_insert_waits_when_manual_checks_ticked` |
| MC-4 | Re-route after a ShipStation sync, box unticked | Fulfilment is queued | `test_reroute_runs_when_manual_checks_unticked` |
| MC-5 | Same, box ticked | Nothing queued | `test_reroute_waits_when_manual_checks_ticked` |
| MC-6 | US-leg tracking shows shipped, box unticked | Order is auto-posted to 1Click | `test_auto_post_on_shipped_runs_when_manual_checks_unticked` |
| MC-7 | Same, box ticked | Not posted | `test_auto_post_on_shipped_waits_when_manual_checks_ticked` |

Proof the tests catch the bug: with the old way of reading the setting put back, MC-2 and MC-4 both fail.

### 2.9 Which stock column the checks use — `test_oneclick_oct2026_scenarios.py` (TestStockCheckUsesConfiguredQuantityField)

| ID | Scenario | Expected | Test |
|---|---|---|---|
| QF-1 | 1Click shows On Hand 10, Available 0; 10 needed; setting = `onhand` | Stock check passes | `test_on_hand_field_counts_stock_held_for_open_orders` |
| QF-2 | Same data; setting = `available` | Stock check blocks (shortfall) | `test_available_field_does_not` |

### 2.10 No Load for Mixed "Direct to Customer" — `test_oneclick_oct2026_scenarios.py` (TestNoLoadForDirectToCustomerOrders)

| ID | Scenario | Expected | Test |
|---|---|---|---|
| DC-1 | Direct to Customer order with a MIFO and `tracking_number_us` filled | No Load is created | `test_no_load_for_direct_to_customer` |
| DC-2 | Same order but "Via US Warehouse" | Load is created with the MIFO items only | `test_load_still_created_when_via_us_warehouse` |

### 2.11 Daily load-received check — `test_oneclick_oct2026_scenarios.py` (TestDailyLoadReceivedCheck)

| ID | Scenario | Expected | Test |
|---|---|---|---|
| LC-1 | Delivered 5 days ago, not modified recently; 1Click shows the load closed | Checked and marked confirmed; no alert (the old "modified yesterday" rule skipped it) | `test_old_unmodified_order_is_checked_and_confirmed` |
| LC-2 | Delivered 3 days ago, 1Click shows nothing; run on three days | Day 1: one alert. Day 2: checked again, no second alert. Day 3: 1Click shows it received → confirmed | `test_not_confirmed_alerts_once_and_is_checked_again_next_day` |
| LC-3 | Delivered today | Left alone (needs at least a day) | `test_delivered_less_than_a_day_ago_is_left_alone` |
| LC-4 | Delivered more than 30 days ago | Skipped (safety cap) | `test_orders_older_than_the_cap_are_skipped` |

### 2.12 Checked by hand rather than automated
- **E (shipped items and packages stored):** run against a real logged 1Click tracking response (order LYF-SH-2026-1984): 3 SKUs and 2 packages stored; saving, an unchanged re-sync and an empty response all behaved correctly. No automated test file.
- **Shipment check against real data:** the same real response, treated as shipped, correctly raised **no** difference.

---

## 3. Test report (2026-10-08)

| Test file | Ran | Result |
|---|--:|---|
| `test_oneclick_oct2026_scenarios.py` | 38 | **All passed** |
| `test_oneclick_shipment_issue.py` | 5 | **All passed** |
| `test_oneclick_post_reliability.py` | 36 | **All passed** (7 earlier failures fixed, see 3.1) |
| `test_us_warehouse_delivered_manual_post.py` | 4 | **All passed** (1 earlier failure fixed) |
| `test_mixed_order_combined_posting.py`, `test_pd1_mixed_order_non_tubing.py`, `test_bom_kit_routing.py`, `test_bulk_transfer_batch_tracking.py` | 29 | All passed (regression) |
| `test_factory_to_us_order_leg.py` | 4 | **4 errors, not related to today's work** (a missing-Item link error when the Transfer Order is created; fails the same way without any of today's changes) |

### 3.1 Test fixes
- **`test_oneclick_post_reliability.py`:** 6 of the 7 earlier failures assumed 5 retry attempts and 2-second waits, but this site's Oneclick Settings are 1 attempt and 0 seconds. The tests now pin those settings. The 7th (`test_load_failing_five_times_keeps_status_one_slack_no_email`) lost its Factory-issued item when the test reloaded the order, and Create Load now needs that item; the test adds it back.
- **`test_us_warehouse_delivered_manual_post.py`:** the test checks that delivery stops at "US Warehouse Delivered" for a manual click, but this site has "send automated email / auto post on delivery" switched on, which posts the order. The test now pins that setting off.

### 3.2 Bug found by the tests
RD-1 first failed (the same mistake was then fixed in the three existing places, section 2.8): the new trigger read the "manual 1Click checks" setting with `frappe.db.get_value`, which returns an unticked checkbox as the text `"0"` — and `"0"` counts as true, so the trigger would never have fired. Fixed by reading it with `get_single_value`, which returns a real 0/1.

---

## 4. Open items for review

1. **Same pattern in existing code — fixed 2026-10-08** (see 2.8 and item 3 of the decisions file). On any site where "Manual 1Click checks" is unticked, the after-insert auto-run, the re-route after a ShipStation sync and the auto-post on "Shipped" now run automatically.
2. **"Shipped" signal — confirmed 2026-10-08** on 16 real shipped orders (status "Shipped", shipped date filled, items and packages listed, multi-box orders complete, no differences). Decided: once an issue is Resolved there is no further alert for that order.
3. **Known gaps (by decision, not changed):** a Material Issue for Order that issues the missing material still leaves the US stock check blocked; the posted quantity for a SKU in both lists is the sum; the stock check narrows but cannot close the window for another order to take the stock.
4. **Dispatch email scope.** "US-only" means no Factory-issued items on the order and not in a batch (since 2026-10-08 such an order also posts without a MIFO). Recipient is the existing 1Click Support Email setting; blank disables it.
5. **Test orders left in the database.** The code under test commits, so orders for "Test Customer" (for example `LYF-MN-2026-0031`, `0035`, `0038`, `0039`) remain. Harmless on this testing site.

---

## 5. How to run

```bash
cd /home/frappe/frappe-bench
bench --site lyfelocal.com run-tests --app lh --module lh.lyfe_hardware.test_oneclick_oct2026_scenarios
bench --site lyfelocal.com run-tests --app lh --module lh.lyfe_hardware.test_oneclick_shipment_issue
bench --site lyfelocal.com run-tests --app lh --module lh.lyfe_hardware.test_oneclick_post_reliability
```

Related references: `apps/lh/docs/slack-notifications.md` (events 28 and 29), `.ai-worklog/2026-10-08.md`.
