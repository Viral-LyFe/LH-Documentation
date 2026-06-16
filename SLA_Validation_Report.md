# SLA Escalation System — Validation Report

**Date:** 2026-06-16
**Reference spec:** `Copy of LH_PM_Rules_Developer_Spec.docx`
**Site tested:** `lyfe.local.local` (test site — data updates allowed, **all emails muted** via `frappe.flags.mute_emails = True` for the entire test session; no external notifications were sent)
**Module:** `lh.lh_project.sla`

---

## 1. Executive Summary

The SLA framework (detectors, scan scheduler, auto-close engine, escalation engine, TAR escalation) is structurally complete and, after the fixes below, all 8 rules (R1, R2, R3, R4, R5, R7, Q1, Q2) function end-to-end.

**3 critical bugs** were found and fixed during validation — two of which meant **R3/R4/R5/R7 and Q1 never created any task** (silent failures swallowed by try/except). **1 config error** (R5 on wrong board) and several minor items were also corrected or flagged.

| Severity | Count | Status |
|---|---|---|
| 🔴 Critical (blocks task creation / audit) | 3 | ✅ All fixed |
| 🟠 Config mismatch vs spec | 2 | ✅ Both fixed |
| 🟡 Minor / cosmetic | 4 | ✅ 1 fixed (TZ), 3 flagged |

**Fixes applied during this validation (all on test site, verified by regression test):**
- BUG-1 (candidate key) — `task_linker.py`
- BUG-2 (SLAR-0007 stale filter) — cleared `extra_filters_json`
- BUG-3 (`allocated_to`) — `escalation_engine.py` + `manager_lookup.py`
- BUG-4 (R5 board) — SLAR-0013 → PROJ-0004
- ISSUE-5 (Q1/Q2 escalation → srishti) — SLAR-0007/0008
- ISSUE-8 (timezone basis) — `age.py`

Remaining flagged (need business confirmation, not code-blocking): ISSUE-6 (boundary operator), ISSUE-7 (tag casing), ISSUE-9 (log priority label), and Saturday/holiday working-day policy.

**R6:** Confirmed intentionally merged into R5 (one rule, SLAR-0013). Not missing.

---

## 2. Requirement → Implementation Map

| Spec | Transition | SLA | Detector | Rule | Board (spec → actual) | Owner | Status |
|---|---|---|---|---|---|---|---|
| R1 | Factory Assign → Ready for Dispatch (Std) | 3 WD | Promised Dispatch | SLAR-0009 | Factory → PROJ-0003 ✓ | Factory ✓ | ✅ |
| R2 | Factory Assign → Internal Review (Custom) | 10 WD | Factory to Internal Review | SLAR-0010 | Factory → PROJ-0003 ✓ | Factory ✓ | ✅ |
| R3 | Awaiting Tracking → Awaiting Shipping | 1 WD | Awaiting Shipping | SLAR-0011 | Factory → PROJ-0003 ✓ | Factory ✓ | ✅ (after fix) |
| R4 | Shipped → Completed | 9 cal days | Shipped to Completed | SLAR-0012 | CS → PROJ-0001 ✓ | CS ✓ | ✅ (after fix) |
| R5/R6 | Internal Review → Approved / Pending Cust. | 1 WD | Approval | SLAR-0013 | Engineer → **PROJ-0004 (fixed)** | Engineer ✓ | ✅ (after fix) |
| R7 | Pending Cust. Approval → Approved | 4 WD | Pending Customer Approval | SLAR-0014 | CS → PROJ-0001 ✓ | CS ✓ | ✅ (after fix) |
| Q1 | Drawing Required (Quotation) | 1 day due | Aging Quotation (TAR-0014 + SLAR-0007) | SLAR-0007 | Engineer → PROJ-0004 ✓ | Engineer ✓ | ✅ (after fix) |
| Q2 | Feasibility Required (Quotation) | 1 day due | Aging Quotation (TAR-0012 + SLAR-0008) | SLAR-0008 | Factory → PROJ-0003 ✓ | Factory ✓ | ✅ |

---

## 3. Bugs Found & Fixed

### 🔴 BUG-1 — Candidate key mismatch broke R3/R4/R5/R7 task creation (CRITICAL)

**Symptom:** The four newer detectors (`awaiting_shipping`, `shipped_to_completed`, `internal_review_approval`, `pending_customer_approval`) return candidate dicts keyed `erp_docname`. But `task_linker.process_violation()` and `_create_new()` read `candidate["name"]`. On every real breach this raised `KeyError: 'name'`, which `rule_evaluator._safe_run_rule` swallowed and logged. **Net effect: R3/R4/R5/R7 detected breaches but never created a task.**

**Proof:** SLAR-0012 (R4) returned 9 live breach candidates, all keyed `erp_docname` with no `name` key.

**Fix:** `apps/lh/lh/lh_project/sla/engine/task_linker.py` — read `candidate.get("name") or candidate.get("erp_docname")` in both `process_violation()` and `_create_new()`.

**Verified:** After fix, R3/R4/R5/R7 each create a task and SLA Task Link correctly (Test 1, Test 6).

---

### 🔴 BUG-2 — SLAR-0007 (Q1 Drawing) crashed every scan (CRITICAL)

**Symptom:** SLAR-0007 had `extra_filters_json = {"drawing_uploaded": 0}`. The `drawing_uploaded` column does not exist on Quotation (only `drawing_required` exists). Every scan raised `OperationalError (1054): Unknown column 'tabQuotation.drawing_uploaded'`, swallowed and logged. **Net effect: Q1 drawing-overdue SLA tasks were never created.**

**Fix:** Cleared `extra_filters_json` on SLAR-0007. The `filter_field = drawing_required = 1` already scopes candidates; `close_condition drawing_required = 0` handles closing.

**Verified:** SLAR-0007 now returns 9 candidates and runs clean in the full scan.

---

### 🔴 BUG-3 — `_get_task_owner()` reads wrong ToDo field (CRITICAL for audit accuracy)

**Symptom:** Both `escalation_engine._get_task_owner()` and `manager_lookup._get_task_owner()` query `ToDo.owner` (the record *creator*, almost always Administrator) instead of `ToDo.allocated_to` (the actual *assignee*). This makes `previous_owner` in the escalation log always wrong, and the "manager != current owner" guard unreliable.

**Fix:** Changed `pluck="owner"` → `pluck="allocated_to"` in both `escalation_engine._get_task_owner()` and `manager_lookup._get_task_owner()`.

**Verified:** Regression test confirms `previous_owner` is now logged correctly as the real prior assignee (`support@lyfehardware.com`) instead of `Administrator`.

---

### 🟠 BUG-4 — R5 on wrong board (CONFIG)

**Symptom:** SLAR-0013 (R5) `target_project = PROJ-0001 (CS Factory Ticket)`. Spec requires the **Engineer Board**.

**Fix:** Set `target_project = PROJ-0004 (Engineer Board)`. Owner was already correctly `mark` (Engineer).

---

## 4. Issues Flagged (not auto-fixed — need business decision)

### 🟠 ISSUE-5 — Q1/Q2 SLA-breach escalation manager ≠ srishti — ✅ FIXED
- Was: SLAR-0007 `escalation_manager = ravin`, SLAR-0008 `escalation_manager = tj`.
- Spec §4 says **all** escalation goes to **srishti**.
- **Fix applied:** both set to `srishti@lyfehardware.com`.
- Q1/Q2 immediate-TAR-task escalation is also handled separately by `tar_escalation_engine` (hardcoded srishti) ✓.

### 🟡 ISSUE-6 — Boundary operator inconsistency
- R1/R2 fire at `working_days > threshold` (strict — breaches at threshold **+1**).
- R3/R5/R7 fire at `age_wd >= threshold`, R4 at `age_hours >= threshold` (inclusive — breaches **at** threshold).
- Minor semantic inconsistency. For a "3 working day" SLA, R1 tasks appear on day 4; R3's "1 day" appears on day 1. **Recommend:** confirm intended boundary; align all to `>=` or `>`.

### 🟡 ISSUE-7 — Tag casing differs from spec
- Spec: `sla-breach, dispatch, standard, order` (lowercase).
- Config: `SLA-Breach, Dispatch, Standard, Order` (mixed case) on several rules.
- Frappe tags are case-sensitive — cosmetic, but inconsistent across rules (SLAR-0013 uses lowercase, others mixed). **Recommend:** standardize to lowercase.

### 🟡 ISSUE-8 — Timezone basis mismatch in age calc — ✅ FIXED
- Was: `hours_since()` used Python `datetime.now()` (server OS local = UTC) while `first_seen` is stored via `now_datetime()` (Frappe system TZ = Asia/Kolkata, +5.5h) — a 5.5h discrepancy that skewed escalation timing.
- **Fix applied:** `age.py::hours_since()` now uses `frappe.utils.now_datetime()` as the basis.
- **Verified:** backdating a link by 25h now reports exactly 25.0h.

### 🟡 ISSUE-9 — `escalation` log records priority "downgrade"
- A task already at Urgent (age ≫ critical) escalates to Level 1 = "High". The `priority_rank` guard correctly prevents the actual downgrade (stays Urgent), but the escalation log records `new_priority = High`, which is misleading. Cosmetic.

---

## 5. Edge Cases Tested

| # | Edge case | Method | Result |
|---|---|---|---|
| 1 | Breach task creation (R4, real data) | `process_violation` on live candidate | ✅ Task + Link created, correct subject/project/owner/tags/due |
| 2 | Duplicate prevention | Run `process_violation` twice on same doc | ✅ 1 link only; `violation_count` → 2; no 2nd task |
| 3 | Escalation after 1 day | Backdate `first_seen` −25h, run engine | ✅ status→Escalated, level 1, srishti assigned (`allocated_to`), log written |
| 4 | Auto-close after transition | Set order → Completed, fire `check_and_close` | ✅ Task→Closed, Link→Resolved + `resolved_at`, cache→resolved |
| 5 | No breach (fast transition) | Order shipped <9d ago | ✅ Not a candidate (threshold gate works) |
| 6 | R3/R5/R7 detectors create tasks | Synthetic order in each state | ✅ All three create task + link after BUG-1 fix |
| 7 | Full scan, all rules | `run_all_rules()` end-to-end | ✅ 302 tasks created, **0 errors** (R4=9, Q1=8, Q2=15, R1=268, R2=2) |
| 8 | Boundary (exact expiry) | SQL inspection of `>=` vs `>` | ⚠️ Inconsistent (ISSUE-6) |
| 9 | Weekend handling (working-day rules) | SQL `DATEDIFF - FLOOR(...)` review | ✅ Sundays excluded via `DAYOFWEEK` formula; ⚠️ Saturdays **not** excluded, no holiday calendar (see note) |
| 10 | Missing assignee | `_resolve_owner` → `_user_enabled` fallback chain | ✅ Falls back to `fallback_owner`, then unassigned; disabled users skipped |

**Working-day formula note:** `DATEDIFF(CURDATE(), d) - FLOOR((DAYOFWEEK(d) + DATEDIFF(...) - 1)/7)` subtracts **Sundays only**. Saturdays are counted as working days, and there is **no public-holiday calendar integration**. If the business runs a 5-day week or observes holidays, working-day SLAs will be slightly lenient. Confirm whether Saturday is a working day at Lyfe Hardware.

---

## 6. Remaining Items (need business decision — not code-blocking)

All critical and config bugs are fixed. The following need a business answer before any further code change:

1. **ISSUE-6 (boundary)** — Confirm whether SLA should breach **at** the threshold (`>=`, current R3–R7) or **after** it (`>`, current R1/R2). Then align all detectors to one convention.
2. **ISSUE-7 (tag casing)** — Confirm preferred casing (spec uses lowercase `sla-breach`); standardize all rules if desired.
3. **ISSUE-9 (log label)** — Optional: suppress the misleading "→ High" label when a task is already Urgent.
4. **Working days** — Confirm Saturday/holiday policy. Current formula excludes **Sundays only**, no holiday calendar. If Lyfe runs a 5-day week or observes holidays, integrate the Frappe Holiday List into the WD formula.

### Files changed during validation
- `lh/lh_project/sla/engine/task_linker.py` (BUG-1)
- `lh/lh_project/sla/utils/age.py` (ISSUE-8)
- `lh/lh_project/sla/escalation/escalation_engine.py` (BUG-3)
- `lh/lh_project/sla/escalation/manager_lookup.py` (BUG-3)
- DB config: SLAR-0007 (extra_filters cleared, escalation_mgr), SLAR-0008 (escalation_mgr), SLAR-0013 (board → PROJ-0004)

---

## 7. Production Readiness

| Area | Verdict |
|---|---|
| R1, R2 (legacy detectors) | ✅ Production-ready |
| R3, R4, R5, R7 | ✅ Production-ready **after BUG-1 fix** (applied) |
| Q1 | ✅ Production-ready **after BUG-2 fix** (applied) |
| Q2 | ✅ Production-ready |
| Auto-close | ✅ Verified working |
| Escalation (R1–R7) | ✅ Functional; apply BUG-3 fix for accurate audit |
| Q1/Q2 TAR escalation | ✅ Separate engine, hardcoded srishti, hourly |
| Dedup | ✅ Verified |
| Email safety in test | ✅ Muted; 0 emails sent |

**Overall:** With BUG-1, BUG-2, BUG-4 fixed (done), the system meets the spec for all 8 rules. BUG-3 and ISSUE-5/8 should be addressed before relying on escalation audit accuracy and exact timing, but do not block core SLA breach/task/close behaviour.

---

## 8. Test Cleanup

All 302 test-scan tasks, links, violation-cache rows, and escalation logs created during validation were **deleted** after testing. The site was returned to its pre-test state (102 original tasks, 0 SLA artifacts). The sacrificial test order's field snapshot was restored.
