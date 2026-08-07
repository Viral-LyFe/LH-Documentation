# MCP Connector Security Audit — Fix & Study Plan

> Source: `LyfeHardwareMCPConnectorAuditReport.docx` (audit of `lh` app + `Viral-LyFe/mcp` fork, commit `dc3ae7f`)
> Overall score: **3.4/10 — NOT READY FOR PRODUCTION USE**
> Prepared: 2026-08-07

## Scope of the Audit

- **Audited**: two design documents, the `lh` app code, and the private MCP server fork.
- **Not audited**: the live ERPNext site (no access), and 12 of the 24 exposed record types belonging to external apps.
- **Confirmed good**: read-only guarantee holds (no write tools found), no shared service account, disabling a user blocks Claude access instantly, settings/credential screens are unreachable, financial dashboards are Super-Admin-locked, no SQL injection found, audit log can't be edited/deleted via UI, server library up to date with upstream.

The core finding: the "only shows what you could already see" half of the design is broken — masking hides a field's value in the response but doesn't stop it being filtered/sorted on, so it can be inferred anyway. That undermines most other protections.

---

## Part 1 — Study First (context before touching code)

Read/trace these before making changes, in this order:

1. `lh/lyfe_hardware/mcp_tools/doctypes.py` — the `list` tool handler, `_strip_masked_keys()`, `_masked_fieldnames`, the "excluded sub-records" note at lines 16-24 vs. the actual exposed list.
2. The MCP server fork (`Viral-LyFe/mcp`, commit `dc3ae7f`) — `run_tool()` vs `handle_call_tool()`, and where/why validation (min/max record limits) is only exercised by tests, not the live path.
3. `lh/lyfe_hardware/integrations/mcp_audit.py` — logging wrapper, the "already wrapped" marker bug, the `_require_founder_or_administrator` naming mismatch, guidance-text injection at lines 81-93.
4. `lh/lyfe_hardware/integrations/mcp_scope_guard.py` — how the connector is identified (by display name, not stable ID).
5. The four undocumented dashboard modules: `status_overview.py`, `comparison_dashboard.py`, `item_data_completeness.py`, `pm_operations_dashboard.py` — confirm actual tool count (87) vs. documented (51), and which of these bypass doctype/field permission checks entirely (they query DB directly).
6. `docs/mcp-api-reference.md` and the stale `docs/mcp_tools_api_reference.md` — reconcile which is authoritative.
7. `pyproject.toml:13` — confirm the floating dependency on the MCP server fork.
8. ERPNext OAuth Client records `4sub5lulum` and `c0hsdkmtn4` — figure out which is actually in use (requires live-site access we don't currently have).

---

## Part 2 — CRITICAL: Fix Before Any Wider Use (8 issues)

| # | Issue | Location | Fix |
|---|-------|----------|-----|
| 1 | Masked fields can be inferred via filter/sort (biggest issue in the report) | fork `doctype.py:208`, `_strip_masked_keys()` in `doctypes.py` | Reject any filter/sort that references a field the caller can't see — hard error, not silent drop. |
| 2 | "Custom BOM Items" sub-record exposed despite the file's own rule against it | `doctypes.py:42` (contradicts note at :16-24) | Remove from exposed list. Add a startup check that fails if any sub-record (no parent-level permission check) is ever added again — mirror the existing settings-screen guard. |
| 3 | 3 status-overview tools return COGS with zero permission checks | `status_overview.py:25-61`; cost columns at `lyfe_orders_status_overview.py:126,233` | Apply the same permission check used elsewhere; strip the cost column unless caller is authorized. |
| 4 | `cost_of_goods_sold` custom field has no protection level set at all | `patches/add_item_custom_fieldss.py:16-20` | Set `permlevel` restricted to Factory + Super Admin; verify on the live site afterward. |
| 5 | Daily Work Log readable by every user (role "All") | Daily Work Log doctype permissions | Remove "All" role; grant HR roles + self-only access for employees. Copy the pattern already correct on "External Learning Record". |
| 6 | Field-masking safeguard only configured for 1 of 12 inspectable doctypes | `doctypes.py:139-140` (`_masked_fieldnames` early exit) | Audit each exposed doctype, define sensitive fields, set protection levels. Update docs to describe actual (not intended) coverage until done. |
| 7 | Record-count limits (min/max) exist only in tests, not the live path; 0/-1 bypasses limits entirely | fork `run_tool()` vs `handle_call_tool()`; zero-limit bug `item_data_completeness.py:158` | Wire validation into the live request path; enforce a hard max (e.g. 200); reject below-minimum requests instead of passing through. |
| 8 | MCP server dependency floats on a branch, no lock file | `pyproject.toml:13` | Pin to the exact commit/version audited (`dc3ae7f` baseline); add lock file; add an automated build check that fails if any write-capable tool ever appears. |

---

## Part 3 — HIGH PRIORITY (Issues 9–20)

| # | Issue | Fix |
|---|-------|-----|
| 9 | 36 of 87 tools (dashboards) never went through security design/docs | Document all 87 tools; review each dashboard tool's return data + caller roles; centralize role lists into one source instead of hand-copied lists. |
| 10 | Sensitive fields duplicated under different, unprotected field names (e.g. `cj_consignee_email` vs protected customer email; Shopify totals vs protected order value; `ship_to_city` vs protected location) | Raise protection level on every unprotected twin to match its protected counterpart; sweep all exposed doctypes for the same pattern. |
| 11 | Audit log stores full answers (up to 2000 chars incl. Super-Admin-only data); any System Manager can read/export it all | Restrict log read access to Super Admin; give others a redacted/summary view only. |
| 12 | Failed log writes are swallowed silently — request still succeeds unlogged | If the log write fails, fail the request (don't answer unrecorded). |
| 13 | Double-wrap bug causes 39 tools to log twice + run permission checks twice | Set the "already wrapped" marker correctly in `mcp_audit.py`/`doctypes.py:195`. |
| 14 | Free-text "guidance" field (edited by any System Manager, no length cap) is injected into every Claude response as if it were instructions | Restrict edit to Super Admin; cap length; pass as quoted reference data, not as instructions. |
| 15 | Token scope guard matches connector by *display name* — renaming it silently disables the guard | `mcp_scope_guard.py:52-60` — identify by permanent record ID or dedicated flag; fail closed (block) on lookup failure, not open. |
| 16 | Stale doc claims HR data is not exposed (it is) | Delete `docs/mcp_tools_api_reference.md` or replace with a pointer to the current doc. |
| 17 | Raw DB error messages returned to caller (leaks schema) | Return generic error to user; keep detail in internal logs only. |
| 18 | Staff performance rankings (by name) reachable by CS/Factory roles with no HR access | Restrict `get_pm_user_efficiency` to management roles, or aggregate to team level (strip names). |
| 19 | Two live OAuth Client records for the same connector (65 + 11 tokens), ownership unconfirmed | Confirm which is active; disable (don't delete) the unused one; revoke stale tokens. **Requires live-site access.** |
| 20 | Zero automated tests cover the connector despite 110+ test files in the app | Add tests for: field masking, sub-record exclusion, settings-screen block, token scope guard, disabled-account block, financial role gating, record-count limits. |

---

## Part 4 — MEDIUM / LOW PRIORITY (Issues 21–28)

| # | Issue | Fix |
|---|-------|-----|
| 21 | Audit log editable from server CLI (docs overclaim "nobody can edit it") | Correct documentation wording; restrict CLI-level access where feasible. |
| 22 | Logged answers truncated at 2000 chars (loses the useful tail); no row-count/requested-fields captured; connector-identity field never filled | Capture row count + requested fields before truncating; populate connector identity. |
| 23 | Audit log has no index, unbounded growth; one dashboard does up to 2000 per-row DB lookups | Add indexes; switch numbering scheme; batch the repeated lookups; define an archiving policy. |
| 24 | Role Visibility Guide has 2 factual errors (Engineer row, Super Admin row) | Correct both table entries. |
| 25 | `_require_founder_or_administrator` no longer checks what its name implies; 4 near-duplicate role checks; connector name hardcoded | Rename function; consolidate duplicated role-check logic; move connector name into settings. |
| 26 | Docs claim write-tools were "removed" from the server lib — they were never there | Correct the historical claim in docs. |
| 27 | Two independent field-masking layers, neither aware of the other; some dashboard docs mention fields the queries don't actually return | Document which layer is authoritative; correct mismatched descriptions. |
| 28 | Unclear whether fixes were verified in prod or only on test site (doc contradicts itself) | State the production verification status plainly and accurately. |

---

## Part 5 — Suggested Execution Order

1. **Critical fixes 1–8** — do not expand connector access to more users/roles until these are closed. Issue 1 (filter/sort inference) is the single highest-value fix — it undermines everything else while open.
2. **High-priority fixes 9, 13, 15** — these are cheap, mechanical, and remove entire classes of silent failure (undocumented tools, double logging, name-based guard bypass).
3. **High-priority fixes 10, 11, 14, 18** — data-exposure fixes affecting confidentiality of specific fields/roles.
4. **Issue 19** — needs live-site access; flag for whoever holds ERPNext admin credentials.
5. **Issue 20** — add regression tests for every fix above as it lands, so it can't silently regress.
6. **Medium/low items 21–28** — documentation accuracy and hygiene; batch these once critical/high items are done.

## Open Dependencies / Blockers

- No access to the live ERPNext site — Issue 19 (duplicate OAuth clients) and live verification of Issue 4's field-level permission (`cost_of_goods_sold`) cannot be completed from code alone.
- 12 of 24 exposed record types belong to external apps outside this audit's code access — these still need a pass once accessible.
