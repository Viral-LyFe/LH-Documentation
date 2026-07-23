# Lyfe Hardware MCP Tools — API/Tools Reference

**Applies to:** the live MCP connector (`lh.mcp.handle_mcp`, private `frappe-mcp` fork). Read/write status, exact tool list, and permission checks below are drawn directly from the code — `apps/lh/lh/lyfe_hardware/mcp_tools/doctypes.py`, `apps/lh/lh/lyfe_hardware/mcp_tools/pnl.py`, and `apps/frappe-mcp/frappe_mcp/server/tools/doctype.py` — not just the setup narrative.
**Companion doc:** `apps/lh/docs/mcp-readonly-allowlist.md` covers OAuth setup, rotation/revocation, and the "late/SLA" and order-lineage domain glossary. This doc is the flat, per-tool parameter/return reference for wiring tools safely.
**All 25 tools below are read-only.** There is no `create`/`update`/`delete`/`submit`/`cancel` tool anywhere in this integration — the underlying fork has no code path that can write to Frappe (`build_tool()` raises `ValueError` for any operation other than `"get"`/`"list"`).

---

## Access gate that applies to every tool call, before anything else

Every tool listed below — the 22 doctype tools and the 3 PnL tools alike — is wrapped by `audited_tool()` (`lh/lyfe_hardware/integrations/mcp_audit.py`), which runs `_require_founder_or_administrator()` first:

- Allowed callers **today**: the literal `Administrator` account, or any user holding the `Super Admin` role.
- Everyone else gets `frappe.PermissionError` before the tool body executes at all — this is a blanket, current-phase gate, independent of whatever a caller's normal Desk roles would otherwise permit on the doctype.
- This gate is layered **on top of** (not a replacement for) each tool's own per-doctype/permlevel permission check described per-tool below. Both must pass.
- Every call — success, error, or permission denial — is written to `MCP Audit Log` regardless of outcome (`user`, `tool_name`, `status`, `timestamp`, `client_id`, `execution_time_ms`, `input_args`, `output_result` truncated at 2000 chars, `error_message`).
- A successful dict-shaped response (not list-shaped) may carry an extra `_lh_mcp_guidance` key — free-text guidance sourced live from the `LH MCP Settings` singleton (`enabled` + `assistant_guidance` fields). This is advisory text for the assistant, not part of the queried data, and can change at any time without a code deploy. `list_*` tools and `get_pnl_repeat_customers` never get it (bare list return, no place to hang it).

---

## Doctype tools (22) — auto-generated, one `get_*` + one `list_*` per doctype

Source: `expose_doctype()` in `apps/frappe-mcp/frappe_mcp/server/tools/doctype.py`, registered for these 11 doctypes in `doctypes.py`:

Lyfe Order · Item · Lyfe BOM · Custom BOM Items · Project · Task · Project Task Status · SLA Rule · SLA Task Link · SLA Violation Cache · SLA Escalation Log

### `get_<doctype>` (11 tools, e.g. `get_lyfe_order`, `get_item`, `get_task`, ...)

| | |
|---|---|
| **Read/write** | Read-only |
| **Params** | `name` (string, required) — the doctype's `name`/primary key |
| **Returns** | `dict` — full document as `doc.as_dict()` |
| **Permission check** | `frappe.has_permission(doctype, ptype="read", doc=name, throw=False)` — denies with `PermissionError` if the caller lacks read access to that specific document. Applied **in addition to** the Super-Admin/Administrator gate above. |
| **Field masking** | Calls `doc.apply_fieldlevel_read_permissions()` explicitly before returning — permlevel-restricted fields (see `Lyfe Order` masking below) come back with the field **key present but value `None`** if the caller's role doesn't clear that permlevel. Do not treat key-presence as "has access"; check the value. |

### `list_<doctype>` (11 tools, e.g. `list_lyfe_order`, `list_item`, `list_task`, ...)

| | |
|---|---|
| **Read/write** | Read-only |
| **Params** | `filters` (dict, optional, default `{}`) — Frappe filter dict, e.g. `{"status": "Open"}`<br>`fields` (list[string], optional, default `["name"]`) — fieldnames to return<br>`limit` (int, optional, default `20`, minimum `1`) — max rows<br>`offset` (int, optional, default `0`, minimum `0`) — row offset for pagination<br>`order_by` (string, optional) — e.g. `"modified desc"` |
| **Returns** | `list[dict]` — rows via `frappe.db.get_list()`, each dict containing only the requested `fields` |
| **Permission check** | `frappe.has_permission(doctype, ptype="read", throw=False)` — a doctype-level check, not per-row; row-level `User Permission` scoping is still applied by `frappe.db.get_list()` itself |
| **Caveat** | Default `fields=["name"]` means callers must explicitly request other fields — a naive `list_lyfe_order()` call with no `fields` arg returns bare names only |

### Per-doctype notes

| Doctype | Why it's exposed | Notes specific to this doctype |
|---|---|---|
| **Lyfe Order** | Central order record | Two permlevel-masked field groups (see below); `cost_of_goods`/`shipping_cost` at permlevel 4, `customer_email`/`phone_no` at permlevel 7 |
| **Item** | Product master (PIM) | — |
| **Lyfe BOM** | Bill of materials | — |
| **Custom BOM Items** | BOM child table | Child table — query the parent (`Lyfe BOM`) with a child-row filter rather than this doctype directly, per the app's standard child-table pattern; `get_custom_bom_items`/`list_custom_bom_items` exist but behave like any other doctype tool (no special child-aware logic in the MCP layer) |
| **Project** | PM/task grouping | — |
| **Task** | PM tasks | — |
| **Project Task Status** | Task status master | — |
| **SLA Rule** | SLA monitor configuration | — |
| **SLA Task Link** | Live SLA violation tracker | `status IN ("Active","Escalated")` = currently late; see companion doc for the full "late" glossary |
| **SLA Violation Cache** | SLA scan cache | Denormalized, faster version of the same "is this late" signal; `is_resolved = 0` = still open |
| **SLA Escalation Log** | Escalation audit trail | Records *what happened* at each escalation (old/new priority, who it went to) — not itself a "is this late" signal |

### Lyfe Order field masking detail

| Fields | Permlevel | Visible to |
|---|---|---|
| `cost_of_goods`, `shipping_cost` | 4 | Factory, System Manager, Super Admin |
| `customer_email`, `phone_no` | 7 | Customer Service, System Manager, Super Admin |

This masking is enforced by `apply_fieldlevel_read_permissions()` in the `get_lyfe_order` tool handler and by `frappe.db.get_list()`'s normal permlevel behavior in `list_lyfe_order`. It does **not** apply to raw-SQL tools (the PnL tools below) — those use their own separate guard.

---

## PnL tools (3) — hand-written, delegate to the existing PnL dashboard report

Source: `apps/lh/lh/lyfe_hardware/mcp_tools/pnl.py`. None of these rebuild any calculation — each calls the same whitelisted method the Desk PnL dashboard page uses, via `frappe.call()`.

**Extra restriction beyond the blanket gate:** all three call `_require_super_admin()` as their first line — `"Super Admin" not in frappe.get_roles()` → `frappe.PermissionError`. Since the blanket `audited_tool` gate already limits *all* MCP tools to Administrator/Super Admin, this is currently a redundant-but-intentional belt-and-suspenders check specific to financial data — if the blanket gate is ever relaxed for other roles, this second check keeps PnL data Super-Admin-only regardless.

### `get_pnl_summary`

| | |
|---|---|
| **Read/write** | Read-only |
| **Delegates to** | `lh.lyfe_hardware.page.pnl_dashboard.pnl_dashboard.get_summary()` |
| **Params** | `filters` (string, optional) — JSON string, e.g. `'{"from_date": "2026-01-01", "to_date": "2026-06-30"}'`. Omit for the default range. |
| **Returns** | `dict` — revenue/cost/margin KPIs with month-over-month and year-over-year comparison |
| **Permission** | Blanket gate + `_require_super_admin()` |

### `get_pnl_monthly_trend`

| | |
|---|---|
| **Read/write** | Read-only |
| **Delegates to** | `lh.lyfe_hardware.page.pnl_dashboard.pnl_dashboard.get_monthly_trend()` |
| **Params** | `filters` (string, optional) — same JSON-string shape as above |
| **Returns** | `list` — up to 12 months of trend rows (revenue, cost, profit per month) |
| **Permission** | Blanket gate + `_require_super_admin()` |

### `get_pnl_repeat_customers`

| | |
|---|---|
| **Read/write** | Read-only |
| **Delegates to** | `lh.lyfe_hardware.page.pnl_dashboard.pnl_dashboard.get_repeat_customers_list()` |
| **Params** | `filters` (string, optional) — same JSON-string shape as above |
| **Returns** | `list` — customer name, order count, total order value for repeat customers (>1 order) in the period |
| **Permission** | Blanket gate + `_require_super_admin()` |
| **Email handling** | This tool **hardcodes `include_email=False`** in its `frappe.call()` — it can never surface customer email regardless of caller, even Super Admin. The underlying function has a separate `include_email=True` path gated by its own `_check_email_permission()`, but the MCP tool never exercises it. |

---

## What is NOT exposed (confirm before assuming a tool exists)

- No doctype outside the 11-item allowlist above has any MCP tool — notably **not** Employee, Daily Work Log, LH HRMS Settings, any Payroll/Salary doctype, Material Issue for Order, or Gate Pass. There is no `expose_doctype()` call for any of them.
- No write operation of any kind (`create`/`update`/`delete`/`submit`/`cancel`/`amend`) exists as an MCP tool for any doctype, including the 11 allowlisted ones.
- No tool other than the 3 PnL tools reads from `pnl_dashboard.py` — general financial rollups are not derivable from the doctype tools alone without re-deriving the same SQL client-side.
- No bulk/batch tool exists — every `get_*` call is single-document.

## Quick lookup — all 25 tool names

`get_lyfe_order`, `list_lyfe_order`, `get_item`, `list_item`, `get_lyfe_bom`, `list_lyfe_bom`, `get_custom_bom_items`, `list_custom_bom_items`, `get_project`, `list_project`, `get_task`, `list_task`, `get_project_task_status`, `list_project_task_status`, `get_sla_rule`, `list_sla_rule`, `get_sla_task_link`, `list_sla_task_link`, `get_sla_violation_cache`, `list_sla_violation_cache`, `get_sla_escalation_log`, `list_sla_escalation_log`, `get_pnl_summary`, `get_pnl_monthly_trend`, `get_pnl_repeat_customers`
