# 1Click Changes of 2026-10-08 — Behaviour, Test Cases and Test Report

Site tested: `lyfelocal.com` (testing site, developer mode on). All 1Click calls in the tests are mocked — nothing here was run against live 1Click.

---

## 1. What changed (plain language)

| # | Change | What now happens |
|---|---|---|
| A | **Route D auto-post on US tracking** | An order routed India → US → Customer and held as "Awaiting India Components" is posted to 1Click (unknown SKUs registered first, then Create Order, then Create Load) the moment **both** `tracking_number_us` and `carrier_us` are filled and saved — instead of waiting for the hourly carrier poll. Only when Oneclick Settings is active and "manual 1Click checks" is **off**. |
| B | **Create Load = Factory items only** | Create Load lists only what physically left the Factory (net quantity from the submitted Material Issue for Order). Create Order still lists every item to be delivered to the customer. If nothing was issued by the Factory, no Load is sent and an Error Log entry is written. |
| C | **US stock re-check before posting** | Before Create Order, the US-covered items are checked against 1Click's available quantity. If short, the order is **not posted**: status → "1Click Error", the reason is stored on the order, and one 1Click Slack message lists each short SKU. Factory-issued items are never checked. A failed inventory call never blocks posting. |
| D | **Shipped-vs-posted difference log** | Once 1Click reports an order shipped, the SKUs/quantities it shipped are compared with what was posted. Any difference creates one **OneClick Shipment Issue** log (status *Issue* → *Resolved* by hand) and sends **one** Slack alert to the 1Click webhook. |
| E | **Shipped items and packages stored on the order** | The hourly 1Click tracking sync saves the shipped SKUs/quantities and the packages (tracking, size, weight) on the Lyfe Order (1Click tab). |
| F | **Slack routing** | All 1Click Slack messages go through one helper. Developer mode on → only the `oneclick_slack_webhook_url` webhook (nothing if blank). Developer mode off → Consolidated Alerts Channel as before. The shipment-difference alert always uses the webhook. Test runs never send to Slack. |
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
| LD-2 | No Material Issue for Order submitted | No Load sent | `test_load_skipped_when_nothing_issued_by_factory` |
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

### 2.5 Checked by hand rather than automated
- **E (shipped items and packages stored):** run against a real logged 1Click tracking response (order LYF-SH-2026-1984): 3 SKUs and 2 packages stored; saving, an unchanged re-sync and an empty response all behaved correctly. No automated test file.
- **Shipment check against real data:** the same real response, treated as shipped, correctly raised **no** difference.

---

## 3. Test report (2026-10-08)

| Test file | Ran | Result |
|---|--:|---|
| `test_oneclick_oct2026_scenarios.py` | 11 | **All passed** |
| `test_oneclick_shipment_issue.py` | 4 | **All passed** |
| `test_oneclick_post_reliability.py` | 36 | 29 passed, **7 failed** (pre-existing, see below) |
| `test_mixed_order_combined_posting.py`, `test_pd1_mixed_order_non_tubing.py`, `test_bom_kit_routing.py` | 21 | All passed (regression) |

### 3.1 The 7 failures in `test_oneclick_post_reliability.py`
They fail identically **without** any of today's changes. They expect the default retry settings (5 attempts, 2-second waits), but this site's Oneclick Settings use different values (the tests saw 1 attempt and a 0-second wait). They are about Create Order retry timing, Create Load retry timing and Create Order retry flows, not about today's changes:

`test_item_not_in_master_twice_then_success_no_slack`, `test_order_found_in_1click_before_retry_is_success`, `test_po_already_used_on_retry_counts_as_success`, `test_gap_before_load_then_create_load`, `test_load_failing_five_times_keeps_status_one_slack_no_email`, `test_five_failures_raise_with_attempt_count`, `test_sku_added_then_wait_verify_then_create_order`.

Also failing before and after: `test_us_warehouse_delivered_manual_post.test_delivery_stops_at_us_warehouse_delivered_not_ready_for_dispatch`.

### 3.2 Bug found by the tests
RD-1 first failed: the new trigger read the "manual 1Click checks" setting with `frappe.db.get_value`, which returns an unticked checkbox as the text `"0"` — and `"0"` counts as true, so the trigger would never have fired. Fixed by reading it with `get_single_value`, which returns a real 0/1.

---

## 4. Open items for review

1. **Same pattern in existing code.** These places read the same setting the same way and would treat an unticked box stored as `"0"` as "manual checks on": the after-insert auto-run (`lyfe_order.py`, `after_insert`), the re-route after a ShipStation sync (`maybe_reroute_after_shipstation_sync`), and the tracker's auto-post on "Shipped" (`order_tracking.py`). On this site the setting is currently ticked, so nothing is affected today. Not changed — fixing it would start auto-posting wherever the box is unticked. Decide before go-live.
2. **"Shipped" signal not yet seen in real data.** Every logged 1Click tracking response shows status "Open". The shipment check fires on status "Shipped" or a shipped date; please confirm on a real shipped order that the shipped items are still included.
3. **Known gaps (by decision, not changed):** a Material Issue for Order that issues the missing material still leaves the US stock check blocked; the posted quantity for a SKU in both lists is the sum; the stock check narrows but cannot close the window for another order to take the stock.
4. **Test orders left in the database.** The code under test commits, so orders for "Test Customer" (for example `LYF-MN-2026-0031`, `0035`, `0038`, `0039`) remain. Harmless on this testing site.

---

## 5. How to run

```bash
cd /home/frappe/frappe-bench
bench --site lyfelocal.com run-tests --app lh --module lh.lyfe_hardware.test_oneclick_oct2026_scenarios
bench --site lyfelocal.com run-tests --app lh --module lh.lyfe_hardware.test_oneclick_shipment_issue
bench --site lyfelocal.com run-tests --app lh --module lh.lyfe_hardware.test_oneclick_post_reliability
```

Related references: `apps/lh/docs/slack-notifications.md` (events 28 and 29), `.ai-worklog/2026-10-08.md`.
