# Authorization audit — lh_project / lh_hrms / lh_lms / slack (non-guest)

Scope: every non-guest `@frappe.whitelist()` in `lh/lh_project/`, `lh/lh_hrms/`, `lh/lh_lms/`, `lh/slack/`;
the Slack interaction handlers dispatched from `slack/api.py`; `lh/lh_project/permissions.py`.
Read-only review (2026-10-23). Paths relative to `/home/frappe/frappe-bench/apps/lh/lh/`.

Inventory: lh_project 59 endpoints, lh_hrms 47, lh_lms 0 (no whitelisted methods), slack 0 non-guest
(3 guest receivers covered elsewhere; their background handlers are reviewed here).

Counts: **High 4 · Medium 6 · Low 9**

---

## HIGH

### H1. Appraisal score readable/writable by any user (permlevel bypass)
- **File:** `lh_hrms/points/appraisal_bridge.py:37-42`
- **Method:** `lh.lh_hrms.points.appraisal_bridge.recalculate_weighted_score`
- **Impact:** Any logged-in user (any employee) passes any Appraisal name. The method loads it, recomputes, **saves it with `ignore_permissions=True`**, and returns `custom_weighted_energy_score`. That field was set to `permlevel: 1` (`patches/add_appraisal_weighted_score_field.py:27-32`) so ordinary users can't see it. Appraisal names follow a predictable series, so a user can enumerate and read every employee's appraisal score and force saves (which run hooks and bump `modified`) on appraisals they can't access.
- **Evidence:**
  ```python
  @frappe.whitelist()
  def recalculate_weighted_score(appraisal):
      doc = frappe.get_doc("Appraisal", appraisal)
      _calculate(doc)
      doc.save(ignore_permissions=True)
      return doc.custom_weighted_energy_score
  ```
- **Fix:** `frappe.only_for(["HR Manager", "HR User", "System Manager"])` (or `doc.check_permission("write")` plus a permlevel-1 check). Drop `ignore_permissions`. If nothing calls it yet (docstring says "future HR dashboard button"), remove `@frappe.whitelist`.

### H2. Task/Project row-level access applies to lists only; direct doc access bypasses it
- **File:** `lh_project/permissions.py:82-129`, `hooks.py:206-224`
- **What:** `permission_query_conditions` is registered for Project and Task, but there is **no `has_permission` hook** for either. The `has_permission` map only covers Lyfe Order and External Learning Record, even though the comment above it says it "mirrors the query conditions". In Frappe, query conditions only filter `get_list`/`get_all`. Single-document checks (`/api/resource/Task/<name>`, `frappe.client.get`, form load, `doc.save()`, `frappe.has_permission("Task", ..., doc=...)`) ignore them.
- **Impact:** A user with the Task/Project role who is not a member of a project can GET or PUT any Task or Project by name (names are sequential). This defeats the membership model. Several lh endpoints also rely on `frappe.has_permission(doc=...)` (see H3), so they inherit the gap.
- **Secondary:** tasks with `project` empty/NULL are visible to everyone (`permissions.py:120-122`). Combined with the missing hook, every personal task can be edited by any Task-role user. `save_task_with_overdue_reason` and `stage_tracker` also skip the check when `task.project` is empty.
- **Fix:** Add `has_permission` hooks for `Task` and `Project` that return `False` unless `_is_privileged(user)` or the user is a member of the task's or project's project (for no-project tasks: owner or assignee). Register them in `hooks.py`. The SQL in this file is built correctly (`frappe.db.escape(user)`), so there is no injection.

### H3. Gorgias reply/sync: project membership is bypassed, and the ticket id can be chosen by the caller
- **Files:** `lh_project/gorgias/reply.py:24-60`, `lh_project/gorgias/recovery.py:32-60`; field definition at `lyfe_hardware/doc_events/custom_field.py:361-368`
- **Methods:** `lh.lh_project.gorgias.reply.reply_to_customer`, `lh.lh_project.gorgias.recovery.sync_task_from_gorgias`
- **Impact:**
  1. The only check is `frappe.has_permission("Task","write",doc=task_name)`. Because of H2 this ignores project membership, so any Task-role user can email customers (as the company's Gorgias agent) from any Gorgias-linked task.
  2. `custom_gorgias_ticket_id` is a plain writable Data field (only `hidden`, permlevel 0). A user can set it on their own task (form save, REST PUT, or the mass-assignment in M1) to **any Gorgias ticket id**. Then:
     - `sync_task_from_gorgias` imports that ticket's whole conversation (customer PII and messages) into a task they can read.
     - `reply_to_customer` sends arbitrary HTML to that ticket's customer.

  The docstring's claim "not an arbitrary ticket id" is false.
- **Evidence:** `if not frappe.has_permission("Task", "write", doc=task_name): ...` then `gorgias_ticket_id = frappe.db.get_value("Task", task_name, "custom_gorgias_ticket_id")` → `send_ticket_message(gorgias_ticket_id, ...)`
- **Fix:**
  - Make `custom_gorgias_ticket_id` (and `_url`) `read_only` + `permlevel: 1`, written only by the sync job.
  - In both endpoints, call `has_project_access(task.project)` and require a support role (e.g. Customer Service).
  - Fix H2.

### H4. Projection lets a project member read tasks from projects they can't access
- **Files:** `lh_project/page/project_board/project_board.py:1752-1800` → `lh_project/projection_write_service.py:128-165` → `lh_project/projection_service.py:32-91`
- **Method:** `lh.lh_project.page.project_board.project_board.add_task_to_board`
- **Impact:** Authorization only checks membership of the **guest** board (`require_projection_access("create", guest_project=...)`, `projection_service.py:52`). Nothing checks that the caller can access the task's **home** project. A member of project A can project any Task name (e.g. from a confidential project B) onto A. `get_board_data`/`get_column_tasks` then return the full enriched task (subject, assignee, dates, SLA, reference doc) through the projected arm of `get_visible_tasks_for_board`. Task names are sequential, so this can be scripted.
- **Evidence (`projection_write_service.project_task_to_board`):** checks only that the task exists and that the status is valid. The docstring says "caller must have access to guest_project", but the source task's project is never checked.
- **Fix:** In `create_projection`, add `has_project_access(home_project)` when `home_project` is set (and the same before `move_visible_task`/`reorder` if desired).

---

## MEDIUM

### M1. Mass assignment of any Task field in `save_task_with_overdue_reason`
- **File:** `lh_project/overdue_tracker.py:343-394` (loop at 374-381, save at 393)
- **Method:** `lh.lh_project.overdue_tracker.save_task_with_overdue_reason`
- **Impact:**
  - `doc_json` is applied to every Task field except layout fields, including `read_only` fields (`completed_by`, `progress`, stage/SLA custom fields, `custom_board_status_kanban`, `custom_gorgias_ticket_id`), `project` (a task can be moved into a project the caller isn't a member of), and `parent_task`/`depends_on`.
  - The project check runs only against the old project, and only when it's set.
  - `task.save()` still enforces doctype write permission and permlevel, so the impact is bypassing read-only fields and project boundaries, not role permissions.
- **Fix:** Allowlist the fields the overdue dialog needs (status, completed_on, exp_end_date, etc.). Reject changes to `project`. Also check `has_project_access(new_project)`.

### M2. `set_column_terminal` lets any member mass-complete a column, bypassing doctype permission
- **File:** `lh_project/page/project_board/project_board.py:1627-1647`
- **Method:** `...project_board.set_column_terminal`
- **Impact:**
  - Project Task Status is writable only by System Manager / Projects Manager (doctype JSON), and `delete_column` enforces that role (line 1538).
  - This endpoint only checks membership and then saves with `ignore_permissions=True`.
  - The 0→1 transition runs `_sync_terminal_on_flag_set` (`doctype/project_task_status/project_task_status.py:21+`), which bulk-UPDATEs every open task in the column to Completed. That skips per-task validation, overdue reasons, and SLA logic.
- **Fix:** Apply the same `{"System Manager","Projects Manager"}` role gate as `delete_column` and drop `ignore_permissions`. Apply the same role gate to `reorder_columns` (L4).

### M3. Manager award: no bounds on points (negative or unlimited)
- **File:** `lh_hrms/points/manager_award.py:14-88` (cap check at 32-57)
- **Method:** `lh.lh_hrms.points.manager_award.award_manager_points`
- **Impact:**
  - The manager check (session user's Employee == target's `reports_to`) is correct.
  - However, `points = int(points)` is unbounded. A manager can award large negative values (the cap check `existing_total + points > cap` never fires for negatives). When `manager_points_monthly_cap` is 0/unset, any positive amount is allowed.
  - Energy points feed appraisal scores (H1 `_track_totals`), so a manager can arbitrarily sink a report's appraisal.
- **Fix:** Require `0 < points <= settings.max_single_award`. Treat `cap == 0` as "no awards" or require HR to configure it.

### M4. HR dashboard "manager" scope is department-wide, not reports-only
- **File:** `lh_hrms/page/hr_dashboard_data.py:85-127`
- **Methods:** every scoped endpoint in this module (e.g. `get_missing_checkins` :1013, `get_leave_summary` :481, `get_attendance_trend` :397, `get_reconciliation` :269, `get_probation_tracker` :1159, `get_compliance` :220, `get_team_workload` :1362, `get_productivity_trend` :1663, `get_workforce_movement` :1692, ...)
- **Impact:**
  - Any employee with **one** direct report becomes `level="manager"` and sees attendance, leave, missing check-ins, reconciliation flags, probation, and hours for the **entire department**, including peers and their own manager.
  - If that employee has no `department`, the filter becomes `{"department": None}`, which matches every active employee with a blank department.
  - The module docstring documents department scope as intended, but it's broader than "an employee's manager" for sensitive HR data. Everything else in the module is well built: scope is resolved on the server and the client cannot widen it.
- **Fix:** Scope managers to their `reports_to` subtree (`lft/rgt` on Employee, or a recursive `reports_to`). If `emp.department` is falsy, return an empty list.

### M5. Slack quotation approval: disabled ERP users can still approve; submission is not re-authorized
- **File:** `slack/quotation_approval.py:188-216`, `251-327`
- **Handlers:** `lh.slack.quotation_approval.handle_action` / `handle_view_submission` (enqueued from `slack/api.py:109-139`)
- **What's right:** The Slack user is mapped to ERP by Slack profile email (`users.info_for_slack_user`) and must hold `approver_role`. The workflow then runs under `frappe.set_user(erp_user)`, so ERPNext workflow role checks apply.
- **Gaps:**
  - `_authorize` uses `frappe.db.exists("User", email)` and `frappe.get_roles(email)`. **Disabled users keep their roles**, so an offboarded approver who is still in Slack can approve quotations.
  - `handle_view_submission` takes `erp_user` from `private_metadata` and never re-checks `payload["user"]["id"]` or the role. A role revoked between opening and submitting the modal is not caught.
  - `_run_action` doesn't check that the Slack Thread Link found by (channel, ts) references `quotation_name`.
- **Fix:**
  - In `_authorize`, also require `User.enabled == 1` and `user_type == "System User"`.
  - Call `_authorize(payload["user"]["id"])` again in `handle_view_submission` and confirm the result equals the metadata `erp_user`.
  - Assert `link.reference_doctype == "Quotation" and link.reference_name == quotation_name`.

### M6. Slack order lookup / notes have no authorization; unmapped Slack users write as Administrator
- **Files:** `slack/tag_to_note.py:44-162`, `slack/order_lookup.py:21-53, 205-235`, `slack/factory_routing.py:288-342`
- **Handlers:** `tag_to_note.on_direct_message`, `tag_to_note.on_mention`, `factory_routing.handle_reply` (enqueued from `slack/api.py:events`)
- **Impact:**
  - Any member of the Slack workspace (including guests or Slack Connect partners, such as the factory) can DM the bot an order or quotation id and get the customer name, items, tracking number, and quotation grand total. The Lyfe Order `has_permission` and Quotation `permission_query_conditions` row restrictions are skipped.
  - When the Slack email doesn't map to an ERP user, the note or reply comment is inserted with `frappe.set_user("Administrator")` and `notify_mentions_now(..., "Administrator")`, so the comment appears to come from Administrator and can trigger @mention notifications.
- **Fix:**
  - Require a mapped, enabled ERP user and then check `frappe.has_permission(reference_doctype, "read", reference_name, user=erp_user)` before lookup or note.
  - For unmapped users, use a dedicated low-privilege "Slack Bot" user, never Administrator.
  - Optionally restrict to configured channels.

---

## LOW

| # | file:line | method | issue | fix |
|---|---|---|---|---|
| L1 | `lh_project/task_status.py:37-70` | `lh.lh_project.task_status.get_project_statuses` | No `has_project_access`. Any user can read column names, colors, and terminal flags of any project (SELECT *). | `has_project_access(project)` |
| L2 | `lh_project/task_status.py:81-130` | `...task_status.get_task_kanban_counts` | Comment and subtask counts for arbitrary task names, with no project check (small leak; confirms a task exists). | Filter `task_names` through `frappe.get_list("Task", filters={"name":["in",...]})` |
| L3 | `lh_project/page/project_board/project_board.py:1964-2030, 2111` | `get_projection_health`, `get_projection_integrity_summary`, `get_projection_cache_health`, `get_projection_realtime_health`, `get_projection_request_metrics` | Callable by any authenticated user (stated in their docstrings). They expose internal counts and the Redis error text and run integrity COUNT scans on demand. `run_projection_integrity` returns the same data behind `_require_admin_or_sm`, so the gating is inconsistent. | Call `_require_admin_or_sm()` in all of them |
| L4 | `project_board.py:1592-1624` | `...project_board.reorder_columns` | Raw SQL UPDATE of Project Task Status order by any member (the doctype is Projects Manager only). Integrity impact only. | Role gate as for `delete_column` |
| L5 | `lh_project/api/dispatch_calc.py:70-110` | `lh.lh_project.api.dispatch_calc.get_dispatch_window_for_order` | `frappe.db.get_value("Lyfe Order", ...)` bypasses the Lyfe Order `has_permission` hook. Leaks `promised_dispatch_by`, `order_type`, and whether the order exists. | `frappe.has_permission("Lyfe Order","read",lyfe_order_name,throw=True)` |
| L6 | `lh_project/automation/rule_engine.py:576-583` | `...rule_engine.get_rules_with_status` | Read permission on the source doc is checked, but `existing_task` returns the name, subject, and status of tasks in projects the caller isn't a member of. | Filter `existing_tasks_raw` by `accessible_projects_subquery` / `has_project_access` |
| L7 | `lh_hrms/doctype/daily_work_log/daily_work_log.py:104-107` | `...daily_work_log.get_employee_for_user` | Maps any user email to an Employee ID (enumeration). | Ignore the `user` arg unless the caller has an HR role |
| L8 | `daily_work_log.py:205, 408-412` (used at 448, 523) | `add_task_details_to_dwl`, `save_dwl_tasks`, `save_dwl_missing_tasks` | `erp_task` can be any Task. Its `subject` is copied into the caller's own log via `frappe.db.get_value`, bypassing Task project restrictions. Owner and HR checks on the log itself are correct. | Validate `erp_task` with `frappe.has_permission("Task","read",...)` plus `has_project_access` |
| L9 | `lh_hrms/page/hr_dashboard_data.py:115-118` | all scoped endpoints | A manager whose `department` is NULL matches all no-department employees (see M4). | Return `[]` when the department is falsy |

---

## Reviewed — no issue found (authorization present and adequate)
- `permissions.py` SQL: user is escaped with `frappe.db.escape`, and there is no default-allow except for privileged users and no-project tasks (covered in H2).
- `project_board.py`:
  - These check `has_project_access` (or `_require_admin_or_sm`) correctly: `get_column_tasks`, `get_column_all_tasks`, `update_task_status` (both paths), `create_quick_task` (insert without ignore), `get_dashboard_metrics` (subquery scoped), `get_all_project_tasks`, `join_board`/`leave_board`, `assign_task_to_user`, `complete_task`/`bulk_complete_tasks`, `create_column` (insert respects doctype perms), `delete_column` (role + membership), `get_board_data`, `remove_task_from_board`, `reorder_projected_task`, the `get_projection_*` analytics (membership checked in `projection_analytics.py`), `get_projection_audit`, `run_projection_integrity`, `clear_projection_cache`.
  - Caveat: H2 still applies to any `task.save()` they make.
- `pm_operations_dashboard.py`: all 9 endpoints call `_check_permission()` / `_check_user_efficiency_permission()`, and every query is scoped via `accessible_projects_subquery`.
- `stage_tracker.py` `get_stage_history` / `get_current_stage` / `get_stage_averages`: membership checked (no-project tasks skip the check, see H2).
- `rule_engine.preview_rule` / `apply_rule_manually` and `comment_task.create_task_from_comment` check read access on the source doc and destination project membership.
- `overdue_tracker.get_active_reasons`: reference data only.
- `api/dispatch_calc.get_dispatch_window`: pure calculation.
- `daily_work_log.py` `clock_out`, `reopen_daily_work_log`, `save_dwl_tasks`, `save_dwl_missing_tasks`, `notify_founder_missing_tasks`, `resend_slack_post`, `add_dwl_addendum`: owner or HR Manager / System Manager check present. `is_task_in_open_dwl` and `add_task_details_to_dwl` operate only on the caller's own Employee.
- `hr_dashboard_data.py` (37 endpoints): scope is resolved on the server via `_viewer_scope`/`_scoped_employees`, and client-supplied employee/department is honoured only for the founder level. Founder-only tiles (`get_open_positions`, `get_recruitment_pipeline`, `get_upcoming_interviews`, `get_department_scorecard`, `get_vacancy_aging`) are gated. `get_my_*` read only the caller's own data. `get_workload_by_department` and `get_org_health_score` reuse scoped functions. Only issues: M4 and L9.
- `lh_lms`: no whitelisted methods.
