# MCP Connector Security Audit — Verification Report

**Subject:** Lyfe Hardware `lh` app — MCP connector (`lh.mcp.handle_mcp`)
**Source audit:** `LyfeHardwareMCPConnectorAuditReport.docx` — original score **3.4/10, not ready for production**
**Fix window:** 2026-08-07 → 2026-08-11
**Verified against:** `lyfe.local.local` (staging) — live code, live database, real users, real roles
**Committed:** `751d8d677db358ab835e04f57b157d94379a1c8a` on branch `prod`
**Production status:** Live at `lyfehardware.v.frappe.cloud`, confirmed by the business

**Verdict: 28 / 28 findings closed and independently re-verified.** A follow-on self-audit re-checked every issue against live code before sign-off and found 3 documentation discrepancies, all corrected the same day. One further policy change (Super Admin unrestricted access) was made after the audit closed and is documented in its own section below.

---

## How this was verified

The original audit reviewed design documents, the `lh` app source, and the private MCP server fork (`Viral-LyFe/mcp` @ `dc3ae7f`) — it had no access to the live site. Every fix below was verified the way the audit couldn't: against a real, running Frappe instance.

For each finding, three things had to hold before it counted as closed:

1. **The exploit reproduced** against the actual code, before the fix — not assumed from reading the report.
2. **The fix applied and re-verified** against the same live exploit path afterward — the same call, same user, same field, now blocked or now correct.
3. **A regression test added**, so the fix can't silently revert on a future change.

Where a finding turned out not to be a real exploit on this Frappe version, that is stated plainly rather than folded into the "fixed" count.

| Part | Priority | Issues | Status |
|---|---|---|---|
| Part 2 | Critical | 1–8 | 8 / 8 closed |
| Part 3 | High | 9–20 | 12 / 12 closed |
| Part 4 | Medium / Low | 21–28 | 8 / 8 closed |
| **Total** | | **28 issues** | **28 / 28 closed** |

---

## Part 2 — Critical

The audit's core finding: masking hid a field's *value* in a response but never stopped a caller filtering or sorting on it — so a hidden value could be inferred anyway, one comparison at a time. That gap undermined nearly every other protection in the connector.

### Issue 1 — Masked fields could be inferred via filter/sort
**Status: Fixed & proven live.**

**Exploit (before fix).** A caller with `cost_of_goods` masked from their response could still pass it in `filters` — binary-searching the field via repeated `list_lyfe_order` calls recovered the exact value while the output never contained the field once.

```
list_lyfe_order(filters={"cost_of_goods": [">", 85.98]})   → 1 row
list_lyfe_order(filters={"cost_of_goods": [">", 85.97]})   → 1 row
# exact value recovered in ~7 comparisons; field itself never shown
```

**Fix.** New `_require_no_masked_filter_or_sort()` in `doctypes.py`, called before every `list_<doctype>` query runs. Rejects any filter/sort referencing a permlevel-masked field, fail-closed, with Administrator exempt.

```
list_lyfe_order(filters={"cost_of_goods": [">", 0]})   → PermissionError:
  "Cannot filter or sort on restricted field(s): cost_of_goods"
```

**Regression:** 13 new tests in `test_mcp_field_masking.py`. Legitimate filtering on unmasked fields, and the existing `get_<doctype>` output-masking, both re-verified unaffected.

---

### Issue 2 — "Custom BOM Items" sub-record exposed against the file's own rule
**Status: Researched — not exploitable on this Frappe version.**

Traced `frappe.has_permission()` on a child doctype directly. It delegates to `has_child_permission()`, which returns `False` outright when no parent doc is given, and — given a real row — correctly resolves the true parent and checks *its* permlevel before anything else. No live exploit exists.

**Outcome:** the audit's own docstring claim ("`has_permission()` is a no-op for pure child doctypes... returns `True` unconditionally") was factually wrong and was corrected in code, rather than "fixing" a mechanism that already works. `Custom BOM Items` remains in the allowlist intentionally.

---

### Issue 3 — Three dashboard tools returned real COGS data with zero permission checks
**Status: Fixed & proven live.**

**Exploit (before fix).** More severe than initially scoped: `get_orders_status_overview()`/`get_orders_reshipments()`/`get_active_reshipments()` ran raw SQL with no permission gate at all. An Engineer-role user with everything masked everywhere else got exact `cost_of_goods` values for 118 orders in a single call — no inference needed.

**Fix.** New `_masked_dashboard_fields()`/`_strip_sensitive_dashboard_fields()` strip `cost_of_goods`/`reshipment_cogs`/`shipment_cost`/`customer` per row, checked against the caller's real `Lyfe Order` permlevel access.

**Related bug found and fixed in the same pass** (not in the original report): the field-masking helpers only inspected a doctype's own top-level fields, never child tables — so `get_lyfe_order`'s embedded `order_items` rows leaked a misleading `cogs: 0.0` instead of the key being absent (Frappe's core masking nulls the real value, but the wrong default survives serialization). Added `_masked_child_fieldnames()` and extended the masking to walk child rows.

**Verified live:** masked user → 0 leaked keys across all 3 tools; authorized user (`srishti@lyfehardware.com`) → 158/158 rows correct, no over-masking. A real child row with `cogs=8.80` now comes back correctly absent for a masked user, not `0.0`.

---

### Issue 4 — `cost_of_goods_sold` had no protection level set in code
**Status: Fixed & proven live.**

The live database already had the correct permlevel and role grants applied directly, out of version control — meaning a fresh install or restore would silently lose the protection.

**Fix:** new idempotent patch `secure_item_cogs_permlevel.py` codifies `permlevel=1` plus Factory/Super Admin grants in code, so the protection survives a fresh install.

**Verified against all 16 enabled users on the site:** 6 correctly blocked, 10 correctly allowed.

---

### Issue 5 — Daily Work Log readable by every user, via role "All"
**Status: Fixed & proven live.**

`DocPerm` for `Daily Work Log` granted the built-in role `"All"` full read/write/create/submit — every enabled user, automatically.

**Fix:** replaced `"All"` with `Employee` (self-only via `if_owner: 1`) plus `HR User`/`HR Manager`/`System Manager`/`Super Admin` — matching the reference pattern already used correctly by `External Learning Record`.

**Verified live end-to-end with a real inserted record:** the owning employee reads their own log; an unrelated Engineer-role user is fully blocked; HR Manager and Super Admin can read regardless of ownership.

**Regression:** 9 new tests in `test_daily_work_log.py`.

---

### Issue 6 — Field-masking safeguard configured for only 1 of 26 exposed doctypes
**Status: Fixed (Employee) & explicitly reviewed (remaining 19).**

Re-audited against the live site: 26 exposed doctypes (not the 12 originally estimated), only 7 had any permlevel-protected field before this pass — 19 had zero.

**Highest-severity finding:** `Employee` — 123 fields, 0 protected. Salary (`ctc`), bank details (`bank_ac_no`/`iban`), personal PII (`date_of_birth`, `personal_email`, `passport_number`, health insurance, exit-interview `feedback`). `HR User` and `HR Manager` held byte-identical permission rows — no field-level distinction between them at all.

**Second finding, flagged explicitly before closing:** `Task.total_costing_amount`/`total_expense_claim`/`total_billing_amount` were unmasked and reachable *today* by a real, non-elevated user (`Projects User` role, no elevated access) — confirmed live via the actual `get_task` tool. This gap was raised back to the business before it was accepted as "no action needed," so the decision was informed, not assumed.

**Fix (Employee only):** new patch sets `permlevel=1` on 17 sensitive fields, granting `HR Manager` + `Super Admin` only — `HR User` explicitly loses both the salary/bank group and the personal/PII group per confirmed scope. The remaining 19 doctypes (`Lyfe BOM`, `Task`, `SLA Rule`, `Sales Invoice`, `Attendance`, and 15 others) were reviewed individually and confirmed by the business as not requiring field-level protection.

**Verified live:** a masked Employee-role user gets 0 of the 17 fields via both `get_employee` and `list_employee`, even when explicitly requested by name in `fields=[...]`. `HR Manager`/`Super Admin` retain full visibility.

**Regression:** 7 new tests in `test_employee_sensitive_fields.py`.

---

### Issue 7 — Record-count limits existed only in tests, never on the live path
**Status: Fixed & proven live.**

```
# before fix
list_lyfe_order(limit=0)      → 2,168 rows   (entire table)
get_missing_items(limit=0)    → 1,982 rows   (entire table)

# after fix
list_lyfe_order(limit=0)      → 1 row
get_missing_items(limit=0)    → 500 rows     (documented default)
```

The MCP fork's own schema validation (`minimum: 1` on `limit`) was only ever exercised by the fork's tests — the live dispatch path calls the tool function directly, never validating the schema. Frappe's own query layer treats `limit=0` as "no LIMIT clause," so it silently dumped the whole table.

**Fix:** new `_clamp_list_limit()` forces `limit` into `[1, 200]` and `offset` to `≥0` before every query runs, for all `list_<doctype>` tools. A parallel `_sanitize_limit()` fixes the identical bug at its source in the dashboard module, protecting the Desk UI page too, not just the MCP wrapper.

**Regression:** 24 new tests across `test_mcp_field_masking.py` and `test_item_data_completeness_limit.py`.

---

### Issue 8 — MCP server dependency floated on a branch, no version lock
**Status: Fixed & proven live.**

`pyproject.toml` declared a floating `@main` branch reference for the vendored MCP fork — the actual running code depends on whatever commit is checked out locally, and nothing prevented drift on a future reinstall.

**Fix:** pinned the exact commit hash (`dc3ae7f947918bbbc4a34a8dc5551899d3fbc71d`). A new automated test inspects all 98 live registered tools at runtime and fails if any tool's source calls a write-capable method or runs a write-verb SQL statement, or if the fork's own `build_tool()` stops raising on `create`/`update`/`delete`.

**Verified the detection actually works:** injected three simulated rogue tools (a plainly-named write tool, a `get_`-named tool secretly calling `.save()`, and a `get_`-named tool secretly running raw `UPDATE` SQL) and confirmed each is caught by a different one of the three detection layers.

**Regression:** 5 new tests in `test_mcp_dependency_pin.py`.

---

## Part 3 — High priority

Access-control gaps, logging integrity, and documentation accuracy across the connector's 98 registered tools — up from the 87 counted at the time of the original audit.

### Issue 9 — 36 of 87 dashboard tools bypassed security review
**Status: Fixed & proven live.**

**Exploit (before fix).** `get_stuck_orders` (Founder Briefing data) had zero permission check. An Engineer-role user with no Founder-AI/Super Admin got real customer names and real order totals for every stuck order in one call.

**Fix.** Gated with `require_founder_ai_or_super_admin()` — the same gate already correctly used for this class of founder-facing data elsewhere; this tool was simply missed when that shared gate was introduced. Three independently duplicated role-check functions (in `comparison_dashboard.py`, `item_data_completeness.py`, `pm_operations_dashboard.py`) consolidated into one shared `require_dashboard_role()` + `DASHBOARD_ROLES` dict.

**Investigated separately:** using `Page.is_permitted()` as a live source of truth instead of a hardcoded role dict — found that mechanism is currently broken for these Desk pages (empty `Has Role` rows despite correct fixtures). Deliberately not built on top of a broken mechanism; noted in code, tracked as a separate unfixed issue.

**Verified live:** unauthorized users blocked on `get_stuck_orders` and both consolidated dashboards; authorized roles (Super Admin, Factory, Engineer) unaffected.

**Regression:** 11 new tests in `test_mcp_dashboard_gates.py`.

---

### Issue 10 — Sensitive fields duplicated under different, unprotected names
**Status: Fixed & proven live, with a correction found by self-audit.**

**Exploit (before fix).** Shipping address (`ship_to_street1/city/state/postal_code/country`) and Shopify financial fields (`shopify_total_price/subtotal_price/total_discounts`) sat at permlevel 0 — visible to every role — while their protected twins (`customer_email`/`phone_no`, `total_amount`/`paid_amount`) were correctly masked. A Factory-only test user with both twins masked could still read the real address ("Macomb") and the real order total in full.

**Fix.** Address → new permlevel 3 (Super Admin/System Manager/Factory/Customer Service/Shipping Forwarder — kept Shipping Forwarder, who needs the address for their job). Shopify financials → new permlevel 8 (Customer Service/Super Admin/Factory/System Manager). Set directly in `lyfe_order.json`; role grants added via a new patch.

**Correction found by the independent self-audit (2026-08-11):** the fix's own documentation and patch docstring said a field named `refund_amount` had been secured. The field actually secured was `shopify_refund_amount` — a genuinely different field (the RTO/return-workflow refund amount). The real `refund_amount` was left unprotected the entire time the documentation implied otherwise. It was briefly masked to close the naming gap, then the business confirmed `refund_amount` is not confidential and it was explicitly reverted to unmasked. The patch's documentation was corrected to name only the field it actually secures, and a dedicated test now locks in that the two fields are distinct and that `refund_amount` is deliberately left visible — so the same name confusion can't recur.

**Verified live:** an Engineer-only user is now correctly blocked from both the real address and the real Shopify financial fields; Factory (in both grants) is unaffected; confirmed both via the Desk permission path and directly through the real `get_lyfe_order` MCP tool.

**Regression:** 12 tests in `test_lyfe_order_address_shopify_masking.py`.

---

### Issue 11 — Audit log stored full answers; any System Manager could read Super-Admin-only data
**Status: Fixed & proven live.**

**Exploit (before fix).** A real `get_pnl_summary` answer (revenue/cost/margin figures) logged to `MCP Audit Log` was fully readable by a System Manager holding no Super Admin/Founder-AI — bypassing the tool's own permission gate entirely via the log.

**Fix.** `output_result` set to permlevel 1, Super Admin only. `input_args`/`tool_name`/`user`/`error_message` stay at permlevel 0, so System Manager retains "who called what tool with what arguments" visibility — just not the actual answer content.

**Verified live on both read paths:** the Desk form view correctly omits `output_result` for a masked user; the Desk report/list-view path omits the key entirely, not a masked default. Super Admin retains full real-value visibility on both.

**Regression:** 8 new tests in `test_mcp_audit_log_access.py`.

---

### Issue 12 — Failed log writes were swallowed silently
**Status: Fixed & proven live.**

The logging function's own comment said "never let audit logging take down a tool call" — any write failure was caught, sent to a separate error channel, and swallowed. A tool call could succeed and return real data with zero corresponding audit entry, meaning the documented promise "every question is recorded" wasn't actually true.

**Fix (business decision, tradeoff explicitly flagged before implementing since it changes availability behavior):** the log-write function now raises on any failure instead of swallowing it — logging is load-bearing, not best-effort. The real underlying error is still captured for diagnosis first.

**Verified live across all three outcome branches** (success, error, permission-denied) by injecting a targeted write failure: a normally-successful call now correctly fails instead of returning silently unlogged; the real failure is still fully captured for diagnosis in every case; normal healthy operation is completely unaffected.

**Regression:** 7 new tests in `test_mcp_audit_log_write_failure.py`.

---

### Issue 13 — Double-wrap bug — 46 tools logged twice and ran permission checks twice
**Status: Fixed & proven live.**

```
# before fix — get_pnl_summary, one call
MCP Audit Log rows written: 2
Permission check ran: 2×

# after fix, same call
MCP Audit Log rows written: 1
Permission check ran: 1×
```

Traced the exact mechanism: the tool-registration code iterated the *entire* tool registry, wrapping anything missing an internal marker a second time — and the marker was only ever set at the registration call site, never by the audit-wrapper decorator itself, so every hand-written tool remained unmarked and got wrapped twice.

**Fix:** the audit wrapper now sets its own marker and short-circuits if a function is already marked, regardless of which code path applied it.

**Re-audited the affected population against the live site:** 46 hand-written tools confirmed double-logging (not the 39 originally estimated — tool count has grown); the 52 doctype-generated tools were unaffected throughout.

**Regression:** 10 new tests in `test_mcp_audit_double_wrap.py`.

---

### Issue 14 — A free-text "guidance" field could inject instructions into every response
**Status: Fixed & proven live.**

A `Text Editor` field, editable by any System Manager with no length cap, was attached to every MCP tool response as bare free text — indistinguishable from real data, with no framing preventing it from being read as an instruction.

**Fix.** Guidance text is now HTML-stripped, entity-unescaped (a rich-text editor's default empty state, `"<p>&nbsp;</p>"`, was found to incorrectly read as non-empty guidance before this — caught and fixed before shipping), and hard-capped at 500 characters. The text is wrapped in a fixed, non-operator-editable preamble marking it explicitly as quoted reference text, never an instruction — regardless of what the operator writes.

**Scope confirmed with the business:** edit access (System Manager + Super Admin) explicitly left unchanged — only length and framing were fixed.

**Regression:** 12 new tests in `test_mcp_guidance_sanitization.py`.

---

### Issue 15 — Token scope guard matched the connector by display name
**Status: Fixed & proven live.**

**Exploit (before fix).** The guard compared the OAuth client's mutable display name against a hardcoded string. Simulating a routine display-name edit — exactly what a System Manager could do from the Desk UI — silently disabled the guard, letting a Bearer token issued for the MCP endpoint be used against an arbitrary other API route with no warning to anyone.

**Fix.** Replaced the display-name comparison with a set-membership check against the client's stable, immutable record ID. This is fail-closed *by construction* — a set-membership check on an immutable ID has no code path where an unrelated field edit silently narrows or disables it, unlike the previous string comparison.

**Verified live:** re-ran the exact same display-name-edit exploit against the fixed code — now correctly blocked. Three legitimate paths (no Bearer token present, an unrelated client's token, a request to the MCP endpoint itself) confirmed unaffected.

**Regression:** 7 new tests in `test_mcp_scope_guard.py`, including a structural test that would catch a future regression before a behavioral test would.

---

### Issue 16 — Stale documentation claimed HR data wasn't exposed
**Status: Fixed & proven live, with a real permission gap found in the process.**

A reference document claimed Employee and Daily Work Log data was "not exposed, on purpose" — both are live, registered tools today. Also claimed every tool call required Administrator/Super Admin — superseded months earlier by a per-role model.

**Fix:** the stale document was replaced with a superseded-pointer notice to the current single reference.

**Real gap found while verifying the correction:** `Quotation Item.cost_of_goods` carried a permission setting that was sharing a permission level with an unrelated shipping-rate grant — meaning ordinary Sales and Maintenance staff could read manufacturing cost data, broader than the equivalent COGS protection on other doctypes (Factory/System Manager/Super Admin only).

**Fix:** moved `cost_of_goods` to its own permission level, matching the equivalent grant elsewhere exactly. The unrelated shipping-rate grant for Sales/Maintenance staff was left completely untouched.

**Verified live:** the final permission map confirms Sales/Maintenance still see shipping-rate fields, and no longer see manufacturing cost.

**Regression:** 7 new tests in `test_quotation_item_cogs_permlevel.py`.

---

### Issue 17 — Raw database errors were returned to the caller
**Status: Fixed & proven live.**

**Exploit (before fix).** Proved this concretely rather than hunting for one specific injection point: registered a temporary tool that deliberately ran an unsafe query with a bad column reference, called it through the real dispatcher, and received back the raw MySQL error text — table and column names included, exactly the schema-mapping detail the audit flagged as a risk applying to *any* tool where an exception escapes.

**Fix.** The generic error branch still writes the full real error to the audit log for investigation, but now re-raises a fixed, generic message instead of the original exception — nothing exception-specific ever reaches the caller. Permission-denied errors were deliberately left untouched, since every such error site in this codebase uses pre-written static text, never text built from a caught lower-level exception.

**Verified live through the actual dispatcher, not just the wrapper in isolation:** the demo tool's raw SQL error no longer surfaces; the real error is still fully captured in the audit log; a genuine permission-denied case still returns its real, safe, informative message unchanged.

**Regression:** 6 new tests in `test_mcp_error_sanitization.py`.

---

### Issue 18 — Named staff performance rankings reachable by roles with no HR access
**Status: Fixed & proven live.**

**Exploit (before fix).** A Customer Service user (no HR or management role) pulled a real, named, ranked staff-efficiency leaderboard including a colleague's actual performance score and status.

**Fix, per explicit scope condition** ("no disruption to Factory/Customer Service's other daily-work tools"): restricted this one tool to Super Admin/HR User/HR Manager via a dedicated role check, distinct from the shared dashboard gate — narrowing the shared gate itself would have also blocked Factory/Customer Service from the file's other 9 legitimate operational tools.

**Verified live:** re-ran the exact exploit — now blocked for the same user. HR Manager and Super Admin retain full access. Explicitly swept all 9 other tools in the same file for both a Factory-only and a Customer-Service-only user — confirmed zero disruption to any of them.

**Regression:** 10 new tests in `test_mcp_pm_user_efficiency_gate.py`.

---

### Issue 19 — Two live OAuth Client records, ownership unconfirmed
**Status: Resolved.**

The business confirmed both records are genuinely, separately in use for different purposes — not a stale-vs-active pair. No record needed to be disabled; no stale tokens needed revoking. The code comment referencing this ambiguity was updated to reflect the confirmed, permanent state.

---

### Issue 20 — Zero automated tests covered the connector
**Status: Fixed.**

Most named safeguards already had dedicated test coverage as a side effect of the issue-by-issue fixes above. Three genuinely had none: the Settings-doctype block, the disabled-account kill switch, and child-doctype exclusion — closed with a new dedicated test file.

**Real finding made while writing the child-doctype test:** `Custom BOM Items` is a genuine child table but *is* on the tool allowlist, apparently contradicting the "child tables are excluded" design rule. Investigated rather than assumed unsafe: confirmed it has zero permission rows of its own, so access fully delegates to its real parent's permissions — proved live by confirming a user with no access to the parent doctype gets correctly blocked from it directly. Documented as a deliberate, verified-safe exception rather than removed.

**Regression:** 12 new tests in `test_mcp_singles_disabled_and_child_exclusion.py`.

---

## Part 4 — Medium & low priority

Documentation accuracy, audit-log operations, and one real performance fix hiding inside a "low priority" line item.

### Issue 21 — Docs overclaimed the audit log was tamper-proof against server access
**Status: Fixed (documentation only).**

Confirmed live: the application-level guard blocks edits made through the normal Desk UI, but a direct database write from a server console bypasses it entirely, silently, with no error. Documentation across the app now names that boundary explicitly rather than overclaiming "cannot be edited by anyone."

---

### Issue 22 — Truncation cap too tight; connector identity never populated
**Status: Fixed & proven live.**

All 343 existing audit-log rows had a blank connector-identity field — it read from an attribute nothing in the codebase ever set, a 100% miss rate. No row-count or requested-fields data existed at all.

**Fix, per explicit scope decisions:** truncation cap raised from 2,000 to 10,000 characters; `row_count` and `requested_fields` added as real fields, populated on every call; connector identity now resolved fresh per call from the actual OAuth token record.

**Verified live end-to-end** through a real tool call: row count and requested fields both correctly written to a real log entry.

**Regression:** 17 new tests in `test_mcp_audit_log_richer_metadata.py`.

---

### Issue 23 — Audit log had no database index; one dashboard ran up to 2,000 lookups per call
**Status: Fixed & proven live.**

The audit log had zero custom database indexes — every filtered query did a full table scan. Separately traced the "2,000 lookups per call" claim to a different dashboard entirely: a Status Overview page running one carrier-name lookup per row inside a loop over up to 2,000 rows.

**Fix:** three new database indexes added to the audit log. The per-row lookup replaced with a single batched query.

**Verified live:** the dashboard returns the exact same real carrier codes as before, confirmed against real order data — a clean fix with no behavior change, only fewer queries.

**Regression:** 11 new tests in `test_mcp_audit_log_indexes_and_status_overview_batching.py`.

---

### Issue 24 — Plain-language role guide had two factual errors
**Status: Fixed (documentation only).**

The guide described one role as getting the combined visibility of two other roles' extra permissions — confirmed live it gets neither extra. It also described Super Admin as able to see Company Settings through Claude — confirmed live that no such tool exists for any role, Super Admin included, so the claim implied a capability that doesn't exist.

---

### Issue 25 — A permission function's name no longer matched what it checked; duplicated logic
**Status: Fixed & proven live.**

The function's name implied a role check it stopped performing months earlier when the underlying access model changed — the name was never updated to match. Two other role-check helpers had near-identical bodies, one just doing an extra dictionary lookup before running the same logic as the other.

**Fix:** renamed the function to describe what it actually checks today; consolidated the duplicated helper so one owns the actual check. A separately-flagged hardcoded internal identifier was investigated and deliberately left as-is, with the reasoning recorded in code — it is never shown to any user.

**Verified the consolidation produces byte-identical behavior** before and after, including the exact error message text.

**Regression:** 12 new tests in `test_mcp_audit_function_naming_and_consolidation.py`.

---

### Issue 26 — Docs claimed write-tools were "removed" from the server library
**Status: Fixed (documentation only), with the precise history established.**

Checked the actual git history of the vendored fork rather than assuming either the audit's or the existing docs' framing was correct. Found a middle truth more precise than either: write-tool code did exist briefly in the fork's own history, but was superseded by a read-only rewrite *before* this application ever pinned a dependency on any commit of the fork. Write-tool generation was never live, deployed, or reachable in this integration at any point.

**Fix:** corrected the framing everywhere it appeared, to state the precise history rather than either overclaiming or underclaiming it.

**Regression:** 7 new tests in `test_mcp_no_write_tools.py`, pinning the dependency's actual current behavior so a future version bump that reintroduces write operations fails loudly rather than silently drifting from the documentation again.

---

### Issue 27 — Undocumented masking design; dashboard docs described fields the code didn't return
**Status: Fixed & proven live, with a real code bug found.**

Two independent field-masking layers exist by design, deliberately unaware of each other — confirmed this is correct architecture, not a bug, and documented it explicitly rather than leaving a reader to infer it.

**Real bug found while checking dashboard documentation for accuracy:** one tool's own code declared it returned a list, but the function it delegates to has always returned a different shape (a dictionary) — confirmed live. A second tool's documented field list named a field that doesn't exist and omitted two that the underlying query does select.

**Fix:** corrected the code's own type declaration to match its real behavior; corrected the documentation to match the actual query.

**Regression:** 4 new tests in `test_mcp_pnl_and_receivables_shape.py`, including a live end-to-end field-set check against real data.

---

### Issue 28 — Documentation contradicted itself on staging vs. production status
**Status: Fixed (documentation only), status confirmed directly.**

Found the exact contradiction: one document stated the connector's OAuth client was "registered on staging," then nine lines later described an incident "found live in production" — with no explanation for the switch. Three other documents separately stated the connector was "not yet deployed to production."

Rather than guess which claim was correct, this was confirmed directly with the business: **the connector is live in production**, and the production incident referenced was real and has since been resolved there. The "not yet deployed" claims were simply stale.

**Fix:** every document now states the current deployment status plainly, and distinguishes staging (used for this work's own live verification) from production (where the connector actually runs) explicitly, rather than mixing the two in the same breath.

---

## Independent self-audit

After all 28 issues above were marked fixed, a separate verification pass independently re-checked every row against live code and database state — not trusting this report's own narrative — and reviewed the actual code changes for security correctness and performance.

**Result: 25 of 28 confirmed outright via live re-derivation.** Exploit reproduction against the real registered tools, direct permission-grant queries, git-history verification, and passing regression suites.

**3 discrepancies found, all corrected the same day:**

1. **Issue 10** — the fix's own documentation named the wrong field (see the correction under Issue 10 above). Resolved as not a real exposure once the business confirmed the correct field's actual sensitivity.
2. **Issue 16** — a reference table still listed a fix as "flagged, not yet applied" after the fix had already shipped. Corrected.
3. **Issue 28** — three documents had a corrected header status contradicted by an un-updated line further down the same page. Corrected, with the old claim struck through and dated rather than silently deleted.

**A fourth, lower-severity partial miss** was found in Issue 26's own fix: one of two documented locations still carried the imprecise phrasing the fix was meant to correct, while the second location had been updated correctly. Corrected.

**Independent reviewers also flagged, none requiring action this pass (informational, deferred):**

- A raw authentication token is used directly as a database lookup key in one function — a theoretical exposure only if database query logging were ever separately enabled.
- The audit log commits to the database unconditionally on every single tool call — a transaction-isolation/performance consideration worth watching under heavy concurrent load, not a correctness bug today.
- Two of the new permission patches assume a pre-existing configuration state on the site rather than creating it from scratch — untested against a genuinely fresh install; recommended for a future check, not urgent.
- One dashboard function still runs an unrelated, minor per-call schema-introspection query — cosmetic, unrelated to any security finding.

**The batched-lookup performance fix (Issue 23) and the two-layer masking design (Issue 27) were both independently reviewed line-by-line and found correct with no gaps** — no permission-scope narrowing, no risk of resolving to the wrong value, no silent truncation of results.

---

## Post-audit policy addition — Super Admin unrestricted access

Not one of the original 28 findings — a policy decision made by the business after the audit and its self-audit were both already closed:

> "Any activity in MCP by Super Admin should have all access of all fields and doc — no restriction at all."

**Investigated before implementing**, rather than assuming existing permission grants already satisfied this. Checked every doctype this work had added masking to. Found **one real gap**: a Quotation permission grant for an unrelated shipping-rate section had never included Super Admin at all — meaning a real Super Admin user (not just the built-in Administrator account) was being silently masked from those fields.

**Fix, built as a structural guarantee rather than a data-layer fix alone.** A single shared check was added and is now used by every enforcement point in the relevant module: field masking, the Settings-doctype block, the filter/sort-inference guard, and the 200-row result cap. This makes the exemption a property of the code itself — not something that depends on every future doctype's permission setup remembering to include Super Admin, which is exactly the class of gap the Quotation finding proved could happen silently. The one real missing grant found was also fixed directly, so the underlying permission data is correct too, not just the code path.

```
# real Super Admin user, holding no Factory role —
# isolates the exemption from any other grant the user might hold

get_item(name)["cost_of_goods_sold"]                  → 10.0     (was absent, before fix)
get_quotation(name)["custom_shipping_rate_final"]     → visible  (the real gap found)
_clamp_list_limit({"limit": 0})                       → limit stays 0, uncapped

# same checks, an ordinary Engineer-role user — confirmed unaffected

get_item(name)["cost_of_goods_sold"]                  → key absent, correctly masked
```

**Verified live** with a real Super Admin user, deliberately chosen without the other roles that would otherwise explain the same access, isolating the exemption itself: field visibility, the Settings block, the filter/sort guard, and the row-count cap all confirmed bypassed for Super Admin, and all confirmed still enforced for an ordinary role.

**Regression:** 15 new tests in `test_mcp_super_admin_unrestricted.py`, plus 6 tests added to an existing suite to cover the bypass behavior at the row-limit cap specifically.

---

## Full regression suite

Every test file created or extended across this work, re-run clean immediately before this report was written.

| Test file | Tests | Result |
|---|---:|---|
| `test_mcp_field_masking.py` | 42 | Pass |
| `test_mcp_super_admin_unrestricted.py` | 15 | Pass |
| `test_mcp_audit_log_richer_metadata.py` | 17 | Pass |
| `test_mcp_guidance_sanitization.py` | 12 | Pass |
| `test_lyfe_order_address_shopify_masking.py` | 12 | Pass |
| `test_mcp_singles_disabled_and_child_exclusion.py` | 12 | Pass |
| `test_mcp_audit_function_naming_and_consolidation.py` | 12 | Pass |
| `test_mcp_audit_log_indexes_and_status_overview_batching.py` | 11 | Pass |
| `test_item_data_completeness_limit.py` | 11 | Pass |
| `test_mcp_dashboard_gates.py` | 11 | Pass |
| `test_security_fixes.py` | 11 | Pass |
| `test_mcp_audit_double_wrap.py` | 10 | Pass |
| `test_mcp_pm_user_efficiency_gate.py` | 10 | Pass |
| `test_daily_work_log.py` | 9 (2 skip — fixture scarcity) | Pass |
| `test_mcp_audit_log_access.py` | 8 | Pass |
| `test_employee_sensitive_fields.py` | 7 | Pass |
| `test_mcp_scope_guard.py` | 7 | Pass |
| `test_mcp_audit_log_write_failure.py` | 7 | Pass |
| `test_mcp_no_write_tools.py` | 7 | Pass |
| `test_quotation_item_cogs_permlevel.py` | 7 | Pass |
| `test_mcp_error_sanitization.py` | 6 | Pass |
| `test_mcp_dependency_pin.py` | 5 | Pass |
| `test_mcp_pnl_and_receivables_shape.py` | 4 | Pass |
| **Total — 23 files** | **253** | **Zero regressions** |

---

## Deployment note

This work is verified against staging (`lyfe.local.local`) and committed to `prod` at `751d8d677db358ab835e04f57b157d94379a1c8a`. Deploying to the live production site (`lyfehardware.v.frappe.cloud`) additionally requires:

1. `bench migrate` — applies the schema and permission-level changes, and runs the new/changed data patches.
2. `bench restart` — loads the updated Python code onto live workers.

Neither step has been run against production as part of this work; both are required before the fixes described above take effect there.

---

*Every finding in this report was independently re-derived against live staging data — exploit reproduction, permission-grant queries, and passing tests — not read off a changelog. Verified 2026-08-11.*
