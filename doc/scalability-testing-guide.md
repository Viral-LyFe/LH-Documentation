# Scalability Verification Test — Manual Testing Guide

Source: `Scalability Verification Test - ERP.docx`
Target: production rollout at **2,000 orders/day**
Testing mode: **Manual** — one Asana task per test, results also logged in a shared tracking doc (see [Reporting](#reporting-format) at the bottom).

This guide translates the document's 3-step validation strategy into concrete, repo-specific steps you can actually execute against the `lh` app. Each test references the real file/function it exercises so you know exactly what you're validating and where to look if it fails.

---

## How to use this file

Work through **Step 1 → Step 2 → Step 3** in order. Do not promote to the next step until every test in the current step has a recorded PASS (or an accepted/explained deviation).

For every checkbox below:
1. Create an Asana task for it (use the "Task Name" given).
2. Do the test.
3. Record the result in your shared tracking doc using the [Reporting](#reporting-format) template.
4. Check the box here once logged.

---

## Step 1 — Developer Validation (Development Server)

Goal: confirm architectural correctness — batching, retries, schedulers, queues, indexes, and failure recovery behave as designed. Machine size doesn't matter here.

### 1.1 Batching behaviour — 17Track bulk path

**File:** `apps/lh/lh/lyfe_hardware/integrations/seventeentrack.py` → `bulk_call_17track(items, timeout=15, batch_size=40)`
**Entry points:** `order_tracking.py` → `scheduled_track_ready_orders()` (line ~1009, India), `scheduled_track_ready_orders_us()` (line ~1415, US)

- [ ] **Task: "Scalability - 17Track batch consolidation"**
  - Create/seed more than one batch's worth of trackable orders (> 40, e.g. 90–100) in dev so at least 2 full batches + 1 partial batch occur.
  - Run `scheduled_track_ready_orders()` manually via bench console:
    ```
    bench --site <site> console
    >>> from lh.lyfe_hardware.doctype.lyfe_order.order_tracking import scheduled_track_ready_orders
    >>> scheduled_track_ready_orders(limit=200)
    ```
  - Confirm tracking numbers are chunked into groups of 40 (check logs / add a temporary print, or step through with a debugger) — no batch should exceed 40.
  - Confirm every order that went in got a tracking result or was correctly deferred (newly-registered numbers are deferred to next run — that's expected, not a bug).
  - Confirm no order is silently dropped across a batch boundary (count in == count accounted for, either updated or deferred).

- [ ] **Task: "Scalability - 17Track batch boundary data-loss check"**
  - Seed exactly 41 orders (one full batch + 1 overflow) and re-run.
  - Confirm the 41st order is processed in a second chunk, not lost.

### 1.2 Retry mechanisms

**Files:**
- ShipStation: `shipstation_settings.py` → `_fetch_orders_page()` (429 → `ShipStationRateLimited`), `_fetch_all_orders()` (retry loop, `time.sleep`)
- 17Track: fixed retry inside `bulk_call_17track` (`_MAX_429_RETRIES = 2`, `_RETRY_WAIT_SECONDS = 5`)
- Shopify: no formal backoff loop — relies on scheduler re-poll `poll_quotation_payment_statuses` (every 5 min) + manual "Update Payment Link" retry button
- CJ/FedEx: no automatic retry — `cj_api.py` → `retry_from_integration_log(log_name)` is manual, user-triggered

- [ ] **Task: "Scalability - ShipStation 429 retry/backoff"**
  - Use WireMock/Mockoon (or a quick local stub server) to simulate ShipStation returning HTTP 429 with `X-Rate-Limit-Reset` header set.
  - Trigger `sync_now()` / `_fetch_all_orders()` against the stub.
  - Confirm the code sleeps for the header-specified duration, retries, and eventually succeeds once the stub returns 200.
  - Confirm it gives up cleanly after `max_attempts` if the stub keeps returning 429 (no infinite loop).

- [ ] **Task: "Scalability - ShipStation transient network error retry"**
  - Stub a connection-reset / timeout instead of 429.
  - Confirm `ShipStationTransientError` path sleeps and retries, and eventually surfaces a clear failure if retries are exhausted.

- [ ] **Task: "Scalability - 17Track 429 bounded retry"**
  - Stub 17Track's `/gettrackinfo` to return 429 for a batch.
  - Confirm exactly 2 retries (`_MAX_429_RETRIES`) with ~5s wait, then the batch fails without cascading into other batches or crashing the whole run.

- [ ] **Task: "Scalability - permanent failure isolation (no cascade)"**
  - Stub one external call (any of the above) to return a permanent error (e.g. 400/404, not 429/5xx).
  - Confirm the run does NOT retry a permanent error, logs it, and continues processing remaining orders/batches — one bad order/batch must not halt the whole run.

### 1.3 Scheduler execution

**Registered in:** `apps/lh/lh/hooks.py` (`scheduler_events`)

| Job | Cron | File |
|---|---|---|
| `version_cleanup.delete_versions_batch` | `*/30 * * * *` | `lh/lyfe_hardware/tasks/version_cleanup.py` |
| `sla_scan.run` | `*/15 * * * *` | `lh/lh_project/sla/scheduler/sla_scan.py` |
| `escalation_scan.run` | `0 * * * *` | `lh/lh_project/sla/scheduler/escalation_scan.py` |
| `stale_cleanup.run` | `0 3 * * *` | `lh/lh_project/sla/scheduler/stale_cleanup.py` |
| `scheduled_track_ready_orders` / `_us` | hourly | `lh/lyfe_hardware/doctype/lyfe_order/order_tracking.py` |
| `run_sync_from_settings` | `*/15 * * * *` | `lh/lyfe_hardware/doctype/shipstation_settings/shipstation_settings.py` |
| `shopify_token_rotation.rotate_shopify_token` | `*/45 * * * *` | `lh/lyfe_hardware/integrations/shopify_token_rotation.py` |
| `poll_quotation_payment_statuses` | `*/5 * * * *` | `lh/lyfe_hardware/integrations/shopify.py` |

- [ ] **Task: "Scalability - scheduler fires on configured interval"**
  - For each job above, use `bench doctor` and the RQ/scheduler logs (`bench --site <site> show-config`, `logs/scheduler.log`, `logs/worker.log`) to confirm each job actually fired at its cron interval over a test window (e.g. 2 hours).

- [ ] **Task: "Scalability - scheduler completes within one cycle (no self-queueing)"**
  - For the most frequent jobs (`*/5`, `*/15`), seed a data volume that makes a single run take noticeably long (e.g. hundreds of orders for `poll_quotation_payment_statuses`, dozens of SLA rules for `sla_scan`).
  - Confirm the job finishes before its next scheduled fire, OR (if it can't) confirm the `deduplicate=True` / fixed `job_id` mechanism prevents a second instance from stacking behind it (see 1.4).

### 1.4 Queue processing / deduplication

**Confirmed dedup call sites:**
- `sla_scan.py` line ~34 — `job_id=f"sla_scan_{rule_name}"`, `deduplicate=True`
- `escalation_scan.py` line ~15 — `job_id="sla_escalation_scan"`, `deduplicate=True`
- `stale_cleanup.py` line ~24 — `job_id="sla_stale_cleanup"`, `deduplicate=True`
- `tar_escalation_scan.py` line ~9 — deduplicated
- `shipstation_settings.py` (sync jobs) — fixed `job_id` to prevent overlapping "Sync Now" clicks

- [ ] **Task: "Scalability - deduplicate=True prevents stacking"**
  - Manually trigger the same job twice in quick succession (e.g. click "Sync Now" on ShipStation Settings twice back-to-back, or call `sla_scan.run()` twice from console before the first returns).
  - Confirm via RQ/Redis (`redis-cli`, or Frappe's Background Jobs list) that only one job with that `job_id` is queued/running at a time — the second call is dropped/ignored, not queued behind.

- [ ] **Task: "Scalability - queue drains after a burst"**
  - Enqueue a burst of background jobs (e.g. trigger notifications for 50+ orders at once via `order_tracking.py` lines ~811/820/835, or bulk-update Lyfe Orders to trigger `on_update` hooks).
  - Watch the RQ queue (`bench --site <site> console` → check `frappe.utils.background_jobs` or the Background Jobs desk page, or `redis-cli llen <queue>`).
  - Confirm the queue drains to zero within a reasonable time and no jobs are stuck/orphaned.

### 1.5 Database query efficiency

**Index patches:**
- `apps/lh/lh/patches/add_scalability_indexes.py` — explicitly written for the 2000 orders/day target. Indexes: `Item.custom_sku`, `Item.custom_ss_product_id`, `Customer.custom_shipstation_customer_id`, `Lyfe Order.workflow_state`, `Quotation.workflow_state`, composite `SLA Task Link(erp_doctype, erp_docname, status)`, composite `Lyfe Order(customer_email, ship_to_postal_code)`
- `apps/lh/lh/patches/add_sla_query_indexes.py` — composites on `SLA Task Link`, `SLA Violation Cache`, `Task Automation Rule`

- [ ] **Task: "Scalability - confirm patches applied"**
  - Run `bench --site <site> migrate` (if not already done) and confirm both patches show as applied in `tabPatch Log`.

- [ ] **Task: "Scalability - EXPLAIN confirms index usage on hot paths"**
  - For each indexed table/column above, run the actual query the code uses (find it in `sla_scan.py`, `order_tracking.py`, `shipstation_settings.py`) with `EXPLAIN` in MariaDB, e.g.:
    ```sql
    EXPLAIN SELECT * FROM `tabSLA Task Link` WHERE erp_doctype=%s AND erp_docname=%s AND status=%s;
    ```
  - Confirm `key` column shows the composite index is used (not `type: ALL` / full table scan).
  - Repeat for `tabLyfe Order` filtered by `customer_email` + `ship_to_postal_code`, and by `workflow_state`.

- [ ] **Task: "Scalability - test_hardening.py index assertions pass"**
  - Run: `bench --site <site> run-tests --app lh --module lh.lh_project.tests.test_hardening`
  - Confirm `test_sla_task_link_dedup_index`, `test_sla_violation_cache_index`, `test_task_automation_rule_index`, `test_patch_uses_add_index` all pass.

### 1.6 Failure recovery (mid-batch failure simulation)

- [ ] **Task: "Scalability - mid-batch failure recovers without duplication"**
  - Seed 3+ batches worth of trackable orders (120+).
  - Using a stub, force the external call to fail on batch 2 only (batches 1 and 3 succeed).
  - Run `scheduled_track_ready_orders()`.
  - Confirm: batch 1's results are saved, batch 2's failure is logged (check Error Log / Integration Request), batch 3 still processes (no cascade).
  - Re-run the scheduler (simulating the next scheduled cycle) and confirm batch 2's orders are retried/picked up again **without duplicating** already-saved tracking data from batches 1 and 3.

### 1.7 Integration behaviour — non-blocking saves

- [ ] **Task: "Scalability - Shopify/ShipStation calls don't block order save"**
  - Open a Lyfe Order / ShipStation Order record and save it while watching request timing (browser dev tools Network tab, or server response time).
  - Confirm the save returns quickly and any Shopify/ShipStation sync work happens via `frappe.enqueue` (background), not synchronously inside the save request.
  - Cross-check against `doc_events` in `hooks.py` for `Lyfe Order.on_update`, `Quotation.on_update` — confirm the SLA/automation hooks (`lh.lh_project.sla.autoclose.close_engine.check_and_close`, etc.) don't make live external calls inline.

### Step 1 sign-off

- [ ] **Task: "Scalability - Step 1 sign-off"** — all above tests logged as PASS (or deviations explicitly accepted); ready to promote to staging.

---

## Step 2 — Staging / Production-like Infrastructure

Goal: validate actual throughput, sustained load behaviour, and infrastructure sizing at production scale. Requires a production-equivalent environment (same DB/Redis/worker sizing as prod, not dev laptop).

**Recommended tooling:** Locust (load simulation), Redis monitoring (queue depth), Grafana/Prometheus (system health), MariaDB Slow Query Log, `bench doctor`.

### 2.1 Sustained scheduler load

- [ ] **Task: "Scalability - sustained tracking scheduler run"**
  - Seed a realistic order pool sized for 2,000 orders/day (i.e. enough active/trackable orders to represent a full day's volume).
  - Let `scheduled_track_ready_orders` / `_us` run continuously (via real cron, not manual trigger) for several hours.
  - Watch Grafana/Prometheus (or server `top`/`htop` + Redis) for CPU, memory, and job duration trend over time.
  - Confirm job duration stays flat/stable across runs — no gradual slowdown (which would indicate a leak or unbounded growth in the query set).

### 2.2 Concurrent user load

- [ ] **Task: "Scalability - concurrent staff load vs background scheduler"**
  - Use Locust to simulate realistic staff concurrency against order-facing endpoints (Lyfe Order list/save, Gate Pass, Material Issue) while the schedulers above are running in the background.
  - Confirm response times for staff-facing pages stay acceptable (define an acceptable threshold with the team beforehand, e.g. < 2s) even while background jobs are active.
  - Watch for DB connection pool exhaustion or lock contention during overlap.

### 2.3 Long-duration queue behaviour

- [ ] **Task: "Scalability - queue drains under sustained ingestion"**
  - Run a sustained ingestion test (e.g. repeated ShipStation syncs, or a Locust script hitting webhook endpoints) over multiple hours.
  - Monitor Redis queue depth continuously.
  - Confirm the queue does not accumulate unbounded (depth trends back to baseline between bursts) and no worker sits idle while jobs pile up (worker starvation).

### 2.4 Infrastructure sizing at 2,000 orders/day

- [ ] **Task: "Scalability - infra headroom at target volume"**
  - Run the full pipeline (ShipStation sync → Lyfe Order creation → tracking → SLA scan) against a dataset representing 2,000 orders/day.
  - Confirm CPU, memory, and DB connection pool usage stay within safe headroom (not pegged at 100%) throughout.
  - Use `bench doctor` before and after to confirm workers, scheduler, and Redis connectivity remain healthy.

### 2.5 Establish monitoring baselines

- [ ] **Task: "Scalability - establish Step 2 baselines"**
  - From the above runs, record baseline numbers for: queue depth (normal + peak), job duration (per scheduler job), query latency (hot-path queries), error rate.
  - These become the reference numbers Step 3 is compared against — write them down in the tracking doc, not just observed and discarded.

### Step 2 sign-off

- [ ] **Task: "Scalability - Step 2 sign-off"** — all above tests logged, baselines recorded; ready for pilot production rollout.

---

## Step 3 — Pilot Production Monitoring

Goal: validate real production behaviour on a limited traffic segment against the Step 2 baselines before full rollout.

- [ ] **Task: "Scalability - pilot rollout to limited traffic segment"**
  - Roll out to a defined limited segment of real production traffic (agree the scope/segment with the team beforehand).
  - Actively monitor (Grafana/Prometheus) against the Step 2 baselines for the pilot duration.

- [ ] **Task: "Scalability - pilot tracking coverage matches Step 2"**
  - Confirm tracking update coverage (% of eligible orders successfully tracked per run) matches expectations from Step 2.

- [ ] **Task: "Scalability - pilot background job success rate matches Step 2"**
  - Confirm background job success rates (scheduler completions, no unexpected failures) match Step 2 baseline.

- [ ] **Task: "Scalability - pilot queue drain times match Step 2"**
  - Confirm queue drain times during the pilot match Step 2 baseline; investigate and resolve any gap before continuing.

- [ ] **Task: "Scalability - resolve staging-vs-live gaps"**
  - Document and resolve any discrepancy found between staging (Step 2) and live pilot (Step 3) behaviour.

- [ ] **Task: "Scalability - Step 3 sign-off"** — obtain sign-off from engineering leadership that production health metrics are stable before completing full rollout.

---

## Bulk Order Volume Tests (2,000 orders)

Two success criteria from the document require actually pushing 2,000 orders through the system:

### A. 2,000 orders in a single run (synthetic/fake orders)

- [ ] **Task: "Scalability - 2000 fake orders single-run test"**
  - There is currently **no existing fake-order generator** for `Lyfe Order` (the only bulk generator in the repo, `apps/lh/lh/scripts/generate_pm_dummy_data.py`, creates PM/SLA test data, not orders — not reusable as-is).
  - Write a one-off script (or extend `generate_pm_dummy_data.py`'s batching pattern — see `_flush_task_batch` for the batch-flush approach) that creates 2,000 synthetic `Lyfe Order` records in one run, ideally via the same path real orders take (`ShipStation Orders.process_shipstation_response()` / `_process_single_order()`) using synthetic ShipStation-shaped JSON payloads, rather than inserting `Lyfe Order` directly — this exercises the real ingestion path, not a shortcut.
  - Run the batch and confirm: no timeout, no partial failure, all 2,000 records created, DB/queue stayed healthy throughout (reuse Step 2 monitoring).
  - Clean up test data afterward (tag with a clear prefix like `LO-LOADTEST-` so it's identifiable and easy to remove, following the `LO-TEST-`/`LO-SLA-TEST-` convention already used in `generate_pm_dummy_data.py`).

### B. 2,000 orders from the real ShipStation API in four steps

- [ ] **Task: "Scalability - 2000 real ShipStation orders in 4 batches"**
  - Using `ShipStationSettings.sync_now()` / `run_sync_from_settings()` (`shipstation_settings.py`), pull orders from the actual ShipStation API in **4 separate steps** (e.g. 4 batches of ~500), as specified in the document.
  - After each of the 4 steps, confirm: all orders in that batch were correctly created/updated as `Lyfe Order` records, no duplicates, no dropped orders, pagination (`_fetch_orders_page`/`_fetch_all_orders`) worked correctly across the batch boundary.
  - Confirm total after all 4 steps = 2,000 orders processed with no functional regressions in order processing, tracking, or SLA evaluation (cross-check against Success Criteria below).

---

## Success Criteria (from the source document)

Confirm ALL of the following hold across Steps 1–3 before calling this complete:

- [ ] No functional regressions in order processing, tracking, or SLA evaluation.
- [ ] Batching consolidates external API calls to the expected rate with no data loss.
- [ ] Retry logic fires correctly on transient failures and does not retry on permanent errors.
- [ ] Schedulers complete each cycle within their interval without accumulation or stacking.
- [ ] Background queues drain reliably under sustained load with no worker starvation.
- [ ] Composite indexes are active and selected by the query planner on all hot paths.
- [ ] Production monitoring metrics remain stable and within the baselines established in Step 2.
- [ ] The system can process 2,000 orders in a single run by generating fake orders.
- [ ] The system can process 2,000 orders from the ShipStation API in four steps.

---

## Reporting Format

For **every** test task above, log the following in the shared tracking doc (in addition to the Asana task):

| Field | Description |
|---|---|
| Task Name | Same name as the Asana task |
| Asana List | Which Asana list/section the task lives in |
| What was tested | Plain description of the test performed |
| What the outcome was | What actually happened |
| Worked? | Yes / No |
| If it worked, why | Brief technical reason it passed |
| If it failed, why | Root cause |
| What fixes were made | Any code/config changes made in response |
| Latest outcome | Result after any fixes — did it now pass? |

Write this in simple, clear language — someone who wasn't in the room should be able to read the doc later and understand exactly what was tested and what happened, without needing to read code.
