# Scalability Test Results — Section 1.1 & 1.2

Format per the Scalability Verification Test document's "Key Details" section. Excludes 1.3 (scheduler execution) — pending separate write-up due to the dev-site `stopped=1` finding.

---

## Task 1: 17Track Batch Consolidation Proof

**Task Name:** Scalability - 17Track batch consolidation

**Asana List:** Scalability Verification — Step 1: Developer Validation

**What was tested:**
Whether `lh.lyfe_hardware.integrations.seventeentrack.bulk_call_17track()` correctly chunks tracking numbers into groups of `batch_size` (default 40) with no data loss across batch boundaries. Ran via `apps/lh/lh/patches/verify_17track_batching.py` (`execute()`), sending 101 fake tracking numbers through the real chunking loop with only the outbound 17Track registration call mocked (no live network, no real orders).

**What the outcome was:**
```
Total items sent in: 101
Register-phase chunk sizes: [40, 40, 21]
All chunks <= batch_size? True
All new items deferred (pending=True)? True
Count of items returned as pending: 101
Newly registered count: 101
No item lost? (in == accounted for): True
RESULT: PASS
```

**Whether it worked or not:** Worked.

**If it worked, why it worked:**
The real `bulk_call_17track` loop uses `for chunk_start in range(0, len(new_items), batch_size)` slicing — plain, dependency-free chunking logic. With 101 items and `batch_size=40`, this correctly produced chunks of 40, 40, and 21 (never exceeding the limit), and all 101 items were accounted for in the output (`newly_registered_count == total_items_in`), proving no item is dropped across the 40/41 batch boundary.

**If it did not work, why did it fail:** N/A — passed on first run, no failure.

**What fixes we made:** None needed — no bug found in this logic.

**What was the latest outcome:** PASS (unchanged since first run).

---

## Task 2: ShipStation 429 Retry / Backoff (honors X-Rate-Limit-Reset)

**Task Name:** Scalability - ShipStation 429 retry/backoff

**Asana List:** Scalability Verification — Step 1: Developer Validation

**What was tested:**
Whether `ShipStationSettings._fetch_all_orders()` / `_fetch_orders_page()` correctly retries on HTTP 429, honoring the `X-Rate-Limit-Reset` header, and eventually succeeds once a 200 is returned. Tested via a **real local HTTP stub server** (not mocked Python objects) — `apps/lh/lh/patches/verify_retry_mechanisms.py`, `test_shipstation_429_retry()` — pointing the real client code at the stub via its `SHIPSTATION_BASE` constant so genuine HTTP requests hit real sockets.

**What the outcome was:**
```
real_http_requests_made: 3
sleep_durations: [0.05, 0.05]
honored_retry_after_header: True
eventually_succeeded: True
RESULT: PASS
```

<img width="792" height="476" alt="image" src="https://github.com/user-attachments/assets/dd35e2d0-502c-49cd-99b7-6f3d230c3e2f" />


**Whether it worked or not:** Worked.

**If it worked, why it worked:**
The stub returned 429 (with `X-Rate-Limit-Reset: 0.05`) on the first two real HTTP requests, then 200 on the third. The real retry loop in `_fetch_all_orders()` slept for exactly the header-specified duration both times (`time.sleep(e.retry_after_seconds)`), then returned the parsed order data once the stub succeeded — confirming the code reads and honors ShipStation's documented rate-limit header rather than using blind backoff.

**If it did not work, why did it fail:** N/A — passed on first run.

**What fixes we made:** None needed.

**What was the latest outcome:** PASS (unchanged).

---

## Task 3: ShipStation 429 — Gives Up Cleanly After max_attempts (No Infinite Loop)

**Task Name:** Scalability - ShipStation 429 gives up after max_attempts (no infinite loop)

**Asana List:** Scalability Verification — Step 1: Developer Validation

**What was tested:**
Whether repeated 429 responses from ShipStation cause the retry loop to give up cleanly after a bounded number of attempts, rather than retrying forever. Real local HTTP stub returning 429 on every request; real client code and real retry loop exercised.

**What the outcome was:**
```
real_http_requests_made: 5
gave_up_cleanly: True
exception_type: ValidationError
RESULT: PASS
```

<img width="882" height="315" alt="image" src="https://github.com/user-attachments/assets/42b1de1f-6a3a-440c-bf96-6e89163b2d9f" />


**Whether it worked or not:** Worked.

**If it worked, why it worked:**
`_fetch_all_orders()` has `max_attempts = 5` (confirmed in source at `shipstation_settings.py:474`). The stub was hit exactly 5 times, then the loop raised via `frappe.throw()` — which surfaces as `frappe.ValidationError` (Frappe's default exception type for `throw()`, not a bug or an unhandled crash) — instead of looping indefinitely.

**If it did not work, why did it fail:** N/A — passed on first run.

**What fixes we made:** None needed.

**What was the latest outcome:** PASS (unchanged).

---

## Task 4: ShipStation Transient Network Error Retry (Exponential Backoff)

**Task Name:** Scalability - ShipStation transient network error retry

**Asana List:** Scalability Verification — Step 1: Developer Validation

**What was tested:**
Whether a real dropped TCP connection (not a mocked exception) is caught as `ShipStationTransientError` and retried with exponential backoff, eventually succeeding once the connection is available again. The local stub server closed the socket mid-request twice (`connection.shutdown(socket.SHUT_RDWR)`), then returned 200 on the third attempt.

**What the outcome was:**
```
real_http_requests_made: 3
sleep_durations: [2, 4]
exponential_backoff: True
eventually_succeeded: True
RESULT: PASS
```

<img width="840" height="350" alt="image" src="https://github.com/user-attachments/assets/07ec1806-5855-4fb5-b163-6897228c4db1" />


**Whether it worked or not:** Worked.

**If it worked, why it worked:**
The dropped socket raised a genuine `requests.exceptions.ConnectionError` client-side (real network failure, not simulated in Python). The code's generic `except Exception` branch in `_fetch_all_orders()` caught it and slept `2 ** attempts` seconds (`2s`, then `4s` — matches the real formula at `shipstation_settings.py:493`), then succeeded on the 3rd real HTTP attempt.

**If it did not work, why did it fail:** N/A — passed on first run.

**What fixes we made:** None needed.

**What was the latest outcome:** PASS (unchanged).

---

## Task 5: 17Track 429 Bounded Retry (No Cascade)

**Task Name:** Scalability - 17Track 429 bounded retry

**Asana List:** Scalability Verification — Step 1: Developer Validation

**What was tested:**
Whether a 17Track batch that keeps returning 429 is retried exactly `_MAX_429_RETRIES` (2) times with the fixed 5s fallback wait, then fails that batch in isolation — without crashing the run or cascading into other batches. Real local HTTP stub returning 429 on every request; real `bulk_call_17track` code exercised (registration phase skipped via `_is_registered=True` to isolate the fetch-phase retry logic under test).

**What the outcome was:**
```
real_http_requests_made: 3
expected_attempts: 3
sleep_durations: [5, 5]
batch_failed_without_crash: True
RESULT: PASS
```

<img width="880" height="397" alt="image" src="https://github.com/user-attachments/assets/1d11f4d9-f530-4369-a2c7-be7990aaca24" />

**Whether it worked or not:** Worked.

**If it worked, why it worked:**
3 total requests = 1 initial attempt + 2 retries, matching `_MAX_429_RETRIES = 2` in source. No `Retry-After` header was sent by the stub, so the code correctly fell back to its fixed `_RETRY_WAIT_SECONDS = 5` wait both times. After exhausting retries, the batch returned `ok: False` for all 3 items in the batch — a clean, logged failure with no exception propagating out and no effect on other (unrelated) batches.

**If it did not work, why did it fail:** N/A — passed on first run.

**What fixes we made:** None needed.

**What was the latest outcome:** PASS (unchanged).

---

## Task 6: Permanent Failure (401) Is Not Retried

**Task Name:** Scalability - permanent failure isolation (no cascade)

**Asana List:** Scalability Verification — Step 1: Developer Validation

**What was tested:**
Whether a permanent error (HTTP 401, representing a real auth failure, not a transient condition) is retried at all, or fails immediately without wasting retry attempts. Real local HTTP stub returning 401 once; real client code exercised.

**What the outcome was:**
```
real_http_requests_made: 1
failed_immediately_no_retry: True
exception_type: ValidationError
RESULT: PASS
```
<img width="667" height="292" alt="image" src="https://github.com/user-attachments/assets/8a020dcc-b4df-40c5-b33f-bdbe0e1861d9" />

**Whether it worked or not:** Worked.

**If it worked, why it worked:**
`_fetch_orders_page()` handles 401 as a distinct, separate branch from the 429/generic-error retry paths — it calls `frappe.throw()` directly on the first 401 response, with no retry loop involved at all. Only 1 real HTTP request was made before the function raised, confirming permanent auth failures don't consume the retry budget or get retried like transient errors.

**If it did not work, why did it fail:** N/A — passed on first run.

**What fixes we made:** None needed.

**What was the latest outcome:** PASS (unchanged).

---

## Task 7: version_cleanup — Overlapping Run Protection (Code Fix)

**Task Name:** Scalability - version_cleanup deduplication gap

**Asana List:** Scalability Verification — Step 1: Developer Validation

**What was tested:**
Whether `lh.lyfe_hardware.tasks.version_cleanup.delete_versions_batch` (cron `*/30 * * * *`) is protected against a second overlapping run if a large Version-record backlog causes one run to take longer than 30 minutes. This was a code-review finding during scheduler-overlap analysis (see 1.3 discussion), not a runtime script execution — traced by reading the source and comparing it against the SLA module's already-correct pattern (`sla_scan.py`, which explicitly documents and uses `job_id` + `deduplicate=True` for exactly this reason).

**What the outcome was:**
Found that `delete_versions_batch()` enqueues its actual work (`_do_delete`) as a separate inner background job — the same fan-out shape as `sla_scan.run`'s per-rule sub-jobs — but with **no `job_id` / `deduplicate=True`** on that inner `frappe.enqueue()` call. Frappe's outer scheduler-level dedup (`is_job_in_queue()`, keyed on `scheduled_job::<method>`) only protects the lightweight outer `delete_versions_batch` call itself, not the actual `_do_delete` work it enqueues — so a slow cleanup run could have a second `_do_delete` stacked on top of it, racing on the same `Version` table rows.

**Whether it worked or not:** Did not work as originally written — confirmed gap, not a false alarm (cross-checked against Frappe's `scheduled_job_type.py:71-95` scheduler-tick logic and the working `sla_scan.py` pattern in the same codebase).

**If it worked, why it worked:** N/A.

**If it did not work, why did it fail:**
The inner `frappe.enqueue("lh.lyfe_hardware.tasks.version_cleanup._do_delete", queue="long", timeout=3600, is_async=True)` call had no `job_id` or `deduplicate=True`. Frappe's RQ dedup only works when a fixed `job_id` is supplied — without one, every enqueue call gets a unique auto-generated ID, so nothing prevents two `_do_delete` runs from being queued and executing concurrently if the first one overruns its own 30-minute cron interval.

**What fixes we made:**
Added `job_id="version_cleanup_do_delete"` and `deduplicate=True` to the inner `frappe.enqueue()` call in `apps/lh/lh/lyfe_hardware/tasks/version_cleanup.py`, matching the exact pattern already used correctly in `sla_scan.py`, `escalation_scan.py`, and `stale_cleanup.py`. Committed as `11b53b9` — "Prevent overlapping version_cleanup runs from stacking".

**What was the latest outcome:**
Fixed. `_do_delete` now has the same job-level dedup guarantee as the other fan-out cron jobs in the codebase — a second overlapping run is now silently skipped by Frappe's RQ dedup rather than stacking. (Not yet re-verified with a live overlap-timing test — that would require deliberately inflating the Version-table backlog to force a run past 30 minutes; flagged as a follow-up if you want that specific proof captured too.)

---

## Note on other scheduler jobs reviewed (no fix needed)

During the same review, `run_sync_from_settings` (ShipStation sync), `rotate_shopify_token`, and `poll_quotation_payment_statuses` were also checked for the same overlap risk. None of them fan out into separately-enqueued inner sub-jobs the way `sla_scan` and `version_cleanup` do — each runs as a single job body, so Frappe's outer scheduler-level dedup (`is_job_in_queue()`) already fully covers them. No code change was made or needed for these three.
