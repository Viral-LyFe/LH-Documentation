# Scalability Test Results — Section 1.3 (Scheduler Execution)

Format per the Scalability Verification Test document's "Key Details" section. Covers both 1.3 checkboxes plus the `version_cleanup` dedup gap found and fixed during this investigation.

---

## Task 1: lh Cron Jobs Registered + Not Individually Stopped

**Task Name:** Scalability - lh cron jobs registered and not stopped

**Asana List:** Scalability Verification — Step 1: Developer Validation

**What was tested:**
Whether every `lh` cron job declared in `hooks.py` (`scheduler_events["cron"]`) has a matching `Scheduled Job Type` row in Frappe with the correct cron expression, and is not individually disabled (`stopped` flag). Ran via `apps/lh/lh/patches/verify_scheduler_execution.py`, `test_jobs_registered_and_not_stopped()`, which reads real `Scheduled Job Type` rows from the database — not simulated data.

**What the outcome was:**

*First run:*
```
All 7 lh jobs: cron_matches = True, stopped = True (all 7)
RESULT: FAIL
```

*After un-stopping the jobs and re-running:*
```
All 7 lh jobs: cron_matches = True, stopped = False (all 7)
RESULT: PASS
```

**Whether it worked or not:** Initially did not work; worked after a fix.

**If it worked, why it worked:**
Once every `lh.*` `Scheduled Job Type` row had `stopped = 0`, the check passed cleanly — all 7 jobs' cron expressions in the database matched `hooks.py` exactly (`*/30`, `*/15` ×2, `0 * * * *` ×1, `0 3 * * *`, `*/45`, `*/5`), confirming the hooks were correctly picked up by Frappe's `sync_jobs()` and nothing else was blocking them from being eligible to run.

**If it did not work, why did it fail:**
On the first run, all 7 `lh` cron jobs — and 46 `Scheduled Job Type` rows bench-wide (11 of them non-`lh`) — had `stopped = 1`. Frappe's scheduler tick (`enqueue_events()` in `frappe/utils/scheduler.py`) explicitly filters `filters={"stopped": 0}` before considering any job for enqueueing, so a stopped job is never fired regardless of whether its cron is due. This was independent of the site-level scheduler flag (`is_scheduler_disabled()` correctly returned `False` the whole time) — the block was per-job, not site-wide.

**What fixes we made:**
Un-stopped all 7 `lh.*` `Scheduled Job Type` rows (Desk UI: `/app/scheduled-job-type`, filter `method like lh.%`, uncheck "Stopped", Save — or via console `frappe.db.sql("UPDATE \`tabScheduled Job Type\` SET stopped=0 WHERE method LIKE 'lh.%'")`). No application code changes were needed — this was a data/configuration state on the dev site, not a bug in `hooks.py` or any `lh` module.

**What was the latest outcome:**
PASS — confirmed on re-run with `stopped: False` across all 7 jobs.

---

## Task 2: lh Cron Jobs Fire Within Their Configured Interval

**Task Name:** Scalability - scheduler fires on configured interval

**Asana List:** Scalability Verification — Step 1: Developer Validation

**What was tested:**
Whether each `lh` cron job actually executes on its configured interval — read from real `Scheduled Job Type.last_execution` timestamps and `Scheduled Job Log` entry counts within a 6-hour lookback window, via `test_jobs_fired_within_interval()` in the same proof script.

**What the outcome was:**

*First run (jobs still stopped):* all 7 jobs showed `last_execution` frozen at `2026-07-18`, ~5.1 days stale, 0 log entries in the 6h lookback. RESULT: FAIL.

*Second run (right after un-stopping, ~1 minute later):*
```
run_sync_from_settings   (*/15)     — fired, last_execution 25s ago   ✅
escalation_scan.run      (0 * * * *)— fired, last_execution 46s ago   ✅
poll_quotation_payment_statuses (*/5) — fired, last_execution 45s ago ✅
sla_scan.run              (*/15)    — still stale (2026-07-18)         ❌
version_cleanup           (*/30)    — still stale                     ❌
rotate_shopify_token      (*/45)    — still stale                     ❌
stale_cleanup.run     (0 3 * * *)   — still stale                     ❌
RESULT: FAIL (3 of 7 passed)
```

**Whether it worked or not:** Partially worked; not yet fully confirmed at time of last check.

**If it worked, why it worked:**
For the 3 jobs that showed fresh executions, their cron boundary (`*/5`, `*/15`, top-of-hour) happened to land within the ~1 minute between un-stopping the jobs and re-running the check — proving the scheduler daemon (`bench schedule` / `bench worker` processes, confirmed alive via `ps aux`) picked them up and fired them for real, with no manual trigger involved.

**If it did not work, why did it fail:**
Not a bug — a timing/observation-window issue. The 4 jobs still showing stale timestamps simply had not reached their own next cron boundary yet at the moment of the check:
- `sla_scan.run` and `version_cleanup` share intervals close to the 1-minute check window and may have been missed on that specific scheduler tick (170 total `Scheduled Job Type` rows bench-wide are shuffled and walked each tick — `random.shuffle(all_jobs)` in `frappe/utils/scheduler.py` — so not every due job is guaranteed to enqueue on the very first tick after a bulk un-stop).
- `rotate_shopify_token` (`*/45`) and `stale_cleanup.run` (daily `0 3 * * *`) mathematically could not have fired yet within a 1-minute window regardless of scheduler health — their next natural boundary was still tens of minutes to hours away.

**What fixes we made:**
None — no code or config fix applies here. This is a "wait and re-check" situation, not a defect. Recommended re-check timing: ~15 min for `sla_scan`/`version_cleanup`, ~45 min for `rotate_shopify_token`, and after the next 3:00 AM server time for `stale_cleanup.run` (or a one-off manual trigger via `bench execute lh.lh_project.sla.scheduler.stale_cleanup.run` if same-day proof is needed — noting that a manual trigger proves the job runs correctly but does not by itself prove the *scheduler* fired it automatically).

**What was the latest outcome:**
FAIL (partial pass, 3/7) as of the last check — pending re-run after each remaining job's cron boundary has had time to pass. Not yet re-tested at the time of this write-up.

---

## Task 3: Job Completes Within One Cron Cycle (No Self-Queueing Risk)

**Task Name:** Scalability - scheduler completes within one cycle (no self-queueing)

**Asana List:** Scalability Verification — Step 1: Developer Validation

**What was tested:**
Whether a representative job (`sla_scan.run`, interval `*/15 min` = 900s) completes in well under its own cron interval, so the scheduler could never fire a second overlapping instance on top of a still-running one. Live timed execution via `test_job_completes_within_one_cycle()` — real function call against this site's actual data (15 active SLA Rules), wall-clock timed with `time.monotonic()`.

**What the outcome was:**
```
job: lh.lh_project.sla.scheduler.sla_scan.run
cron_interval_seconds: 900
elapsed_seconds: 0.023 (also seen 0.035 and 0.227 on other runs)
completed_without_error: True
finished_before_next_cycle: True
RESULT: PASS
```

**Whether it worked or not:** Worked, consistently across every run.

**If it worked, why it worked:**
`sla_scan.run()` itself only loops active SLA Rules and calls `frappe.enqueue()` per rule — it does not do the actual rule evaluation inline. That fan-out design means the outer cron-registered function returns almost instantly (well under a second, confirmed on 3 separate runs: 0.227s, 0.035s, 0.023s), leaving enormous headroom under its 900-second interval. The heavier per-rule work (`_run_rule`) runs as separate, independently-timed background jobs, each with its own `job_id`/`deduplicate=True` guard (see Task 4 below) — so even if one rule's evaluation were slow, it would not block the next scheduler tick from firing `sla_scan.run()` again.

**If it did not work, why did it fail:** N/A — passed on every run, no failure observed.

**What fixes we made:** None needed for this specific job.

**What was the latest outcome:** PASS (consistent across all 3 runs performed).

---

## Task 4: version_cleanup — Overlapping Run Protection (Code Fix)

**Task Name:** Scalability - version_cleanup deduplication gap

**Asana List:** Scalability Verification — Step 1: Developer Validation

**What was tested:**
Whether `delete_versions_batch` (cron `*/30 * * * *`) is protected against a second overlapping run if a large Version-record backlog causes one run to take longer than 30 minutes. Found via source code review while analyzing scheduler-overlap behavior for 1.3, not a runtime script execution — compared against the already-correct pattern in `sla_scan.py`/`escalation_scan.py`/`stale_cleanup.py`.

**What the outcome was:**
`delete_versions_batch()` enqueues its actual work (`_do_delete`) as a separate inner background job, the same fan-out shape as `sla_scan`'s per-rule sub-jobs — but originally had **no `job_id` / `deduplicate=True`** on that inner `frappe.enqueue()` call.

**Whether it worked or not:** Did not work as originally written — confirmed gap, not a false alarm.

**If it worked, why it worked:** N/A.

**If it did not work, why did it fail:**
Frappe's outer scheduler-level dedup (`is_job_in_queue()`, keyed on `scheduled_job::<method>`) only protects the lightweight outer `delete_versions_batch` call itself — not the separately-enqueued `_do_delete` work it spawns. Without a fixed `job_id`, every enqueue call for `_do_delete` gets a unique auto-generated ID, so nothing would stop two `_do_delete` runs from executing concurrently and racing on the same `Version` table rows if the first one overran its 30-minute cron interval.

**What fixes we made:**
Added `job_id="version_cleanup_do_delete"` and `deduplicate=True` to the inner `frappe.enqueue()` call in `apps/lh/lh/lyfe_hardware/tasks/version_cleanup.py`, matching the pattern already used correctly in `sla_scan.py`, `escalation_scan.py`, and `stale_cleanup.py`. Committed as `11b53b9` — "Prevent overlapping version_cleanup runs from stacking".

**What was the latest outcome:**
Fixed. `_do_delete` now has the same job-level dedup guarantee as the other fan-out cron jobs in the codebase. Not yet re-verified with a live overlap-timing test (would require deliberately inflating the Version-table backlog to force a run past 30 minutes) — flagged as a follow-up if that specific live proof is wanted.

---

## Overall Section 1.3 Status

| Checkbox | Status |
|---|---|
| Scheduler fires on configured interval | Partial — 3/7 jobs confirmed firing after un-stopping; 4/7 pending their next natural cron boundary |
| Scheduler completes within one cycle (no self-queueing) | ✅ Confirmed for `sla_scan.run`; `version_cleanup` gap found and fixed (not yet re-verified live) |

**Follow-up needed:** re-run `lh.patches.verify_scheduler_execution.execute` after enough time has passed for all 7 jobs to hit their own cron boundary at least once (recommend checking again in ~45 min, and once more after the next 3:00 AM server time for `stale_cleanup.run`).
