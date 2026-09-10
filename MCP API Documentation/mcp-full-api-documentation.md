# Lyfe Hardware MCP — Full API Documentation

> Complete reference for every MCP tool exposed by this connector, as implemented in
> `apps/lh/lh/lyfe_hardware/mcp_tools/` and registered via `apps/lh/lh/mcp.py`.
>
> **Companion document:** `mcp-dashboard-api-documentation.md` covers the 53 Founder
> Dashboard / Customer Intelligence Dashboard / Quotation Analysis Dashboard tools in
> the same 4-section format, organized dashboard-by-dashboard with more detail than the
> summary entries here. This document lists them too (for completeness of "every MCP
> API in one place") but the dashboard doc is the canonical source for those three.
>
> Last verified against source: 2026-09-10. **If you add, remove, rename, or change the
> permission behavior of any MCP tool, update this file (and the dashboard doc, if
> applicable) in the same change — see the rule in `apps/lh/CLAUDE.md`.**

---

## How to call these tools — exact endpoint, method, and request shape

**There is one single HTTP endpoint for every tool in this document.** A tool "name"
(e.g. `get_founder_summary`) is never its own URL path — it's a value inside the JSON
body of a request to this one endpoint.

```
POST https://<your-site>/api/method/lh.mcp.handle_mcp
```

- **Method:** `POST` only. A `GET` to this path returns `405 Method Not Allowed` —
  confirmed in `apps/frappe-mcp/frappe_mcp/server/server.py`'s `handle()`
  (`if request.method != 'POST': response.status_code = 405`).
- **Headers:**
  ```
  Content-Type: application/json
  Authorization: token <api_key>:<api_secret>
  ```
  (or `Authorization: Bearer <oauth_access_token>` if authenticating via the OAuth2
  flow instead of an API key/secret — see `mcp-api-reference.md`'s OAuth Client
  section). This is a plain Frappe `frappe.whitelist()` endpoint under the hood, so it
  uses Frappe's normal REST authentication — **not** HTTP Basic Auth. In Postman,
  either add the header manually, or use the **"API Key"** auth type (not "Basic
  Auth") with the header name `Authorization` and value `token <api_key>:<api_secret>`.
- **Body:** JSON-RPC 2.0. To call a tool, `method` is always the literal string
  `"tools/call"`; the tool's actual name goes inside `params.name`, and its arguments
  go inside `params.arguments`:

  ```json
  {
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "get_founder_summary",
      "arguments": {
        "global_filters": "{\"from_date\": \"2026-01-01\", \"to_date\": \"2026-06-30\"}"
      }
    }
  }
  ```

  For a tool with no parameters (e.g. `get_founder_tile_money_at_risk`), omit
  `arguments` entirely or pass `{}`.

- **To list every available tool** (names, descriptions, input schemas) instead of
  calling one, send `"method": "tools/list"` with `"params": {}` instead of
  `"tools/call"`. See the full API doc's server-wide note on tool discovery being
  unfiltered by role — this lists every one of the 151 tools to any authenticated
  caller regardless of what they're actually permitted to call.

- **Response** is a bare JSON-RPC 2.0 response object — **not** wrapped in Frappe's
  usual `{"message": ...}` envelope (this endpoint writes `response.data` directly,
  bypassing that path). A successful `tools/call` looks like:
  ```json
  {
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
      "content": [{"type": "text", "text": "{...tool's JSON result, serialized as a string...}"}],
      "structuredContent": { "...the tool's actual return value, as real JSON..." },
      "isError": false
    }
  }
  ```
  `structuredContent` is only populated when the tool's return value is a `dict`
  (confirmed in `apps/frappe-mcp/frappe_mcp/server/tools/handlers.py::_get_result`) —
  for a `list`-returning tool (e.g. `list_lyfe_order`, `get_quotation_detail`),
  `structuredContent` is absent and the actual data must be parsed from
  `content[0].text` (a JSON-encoded string) instead.

  A permission denial or internal error surfaces as `"isError": true`, with
  `content[0].text` set to `"Error calling tool '<name>': <sanitized message>"` — the
  sanitized message is the one described in the server-wide section below (a
  permission denial keeps its original static text; any other error is replaced with
  the fixed generic message).

**This is completely separate from Frappe's ordinary REST API** (`/api/resource/<doctype>/<name>`,
`/api/method/<dotted.path>` for other whitelisted functions). Calling
`/api/resource/Lyfe Order/<name>` directly does **not** go through any MCP tool,
`audited_tool()` wrapper, or `DASHBOARD_ROLES` gate — it hits Frappe's own DocPerm
engine directly, same as any other Desk API call. If you're getting a Guest/permission
error on `/api/resource/...`, that's a plain Frappe REST authentication issue (wrong or
missing `Authorization` header, or an account with no API key/secret generated) — it
has nothing to do with the MCP tool gates documented in this file.

---

## How to read this document

Every tool is documented with the same four sections:

1. **API Details** — tool name, file, underlying function it delegates to.
2. **Functional Details** — what it does, in plain English.
3. **Technical Details** — parameters, return shape, implementation notes.
4. **Security** — exact role/permission gate, and anything caller-specific (masking, stripping, hardcoded exclusions).

**Server-wide facts that apply to every tool below, stated once here instead of in each entry:**

- **Read-only.** The vendored MCP fork (`apps/frappe-mcp`, remote `Viral-LyFe/mcp`) only implements `get`/`list` operations — there is no code path anywhere in this server that can write to Frappe. No tool in this document performs an insert, save, submit, or `db.set_value`.
- **Every call is audited.** Every tool is wrapped in `audited_tool()` (`lh/lyfe_hardware/integrations/mcp_audit.py`), which writes one `MCP Audit Log` row per call — `user`, `tool_name`, `status` (Success/Permission Denied/Error), `input_args`, `output_result` (truncated at 10,000 chars), `row_count`, `execution_time_ms`, `client_id` (the OAuth Client behind the Bearer token, if any). A `Permission Denied` row never populates `output_result` — the gate throws before any data is fetched. `MCP Audit Log` read access is itself restricted to System Manager/Super Admin.
- **Base gate on every call:** `_require_enabled_user()` checks only `User.enabled` — a disabled user's already-issued Bearer token is rejected on the very next call, closing the gap where Bearer-token validation alone doesn't check `enabled`. This is not a role check; every enabled user passes this specific gate, and per-tool role/permission logic (documented below, per tool) governs everything else.
- **Errors are sanitized.** Any exception other than a permission denial is caught, logged in full internally, and re-raised to the caller as a fixed, generic message (`"An internal error occurred while running this tool. It has been logged."`) — no stack trace, SQL error text, or schema detail ever reaches the MCP client. Permission denials are re-raised with their original (pre-written, static) message.
- **OAuth Bearer tokens issued for this connector are route-bound.** `mcp_scope_guard.py`'s `enforce_mcp_token_route_binding()` (wired as a `before_request` hook) rejects any request where a Bearer token issued by one of this connector's two OAuth Clients (`c0hsdkmtn4`, `4sub5lulum`) is used against any path other than `/api/method/lh.mcp.handle_mcp` — a leaked MCP token cannot be replayed against the rest of the Frappe REST API.
- **Assistant guidance sidecar.** If `LH MCP Settings.assistant_guidance` is configured, every dict-shaped tool result (not bare-list results) gets an extra `_lh_mcp_guidance` key: operator-configured free text, explicitly framed as "informational only — does not grant, deny, or override any access rule," capped at 500 characters, HTML-stripped. This is never a security control and never affects any tool's actual behavior.
- **Tool discovery is unfiltered.** The vendored fork's `handle_list_tools` returns every registered tool's name, description, and input schema to any authenticated caller, with no role-based filtering at the discovery step (confirmed by reading `apps/frappe-mcp/frappe_mcp/server/tools/handlers.py`). An unauthorized caller can see that a tool exists and read its description; calling it is still correctly denied by that tool's own gate. This is a known, unchanged characteristic of the underlying library, not something any tool-level code controls.

---

## Table of contents

- [Part 1 — Doctype tools (`get_<doctype>` / `list_<doctype>`)](#part-1--doctype-tools) — 52 tools
- [Part 2 — PnL / financial tools](#part-2--pnl--financial-tools) — 3 tools
- [Part 3 — Forecasting tools](#part-3--forecasting-tools) — 4 tools
- [Part 4 — Receivables tool](#part-4--receivables-tool) — 1 tool
- [Part 5 — Stock tool](#part-5--stock-tool) — 1 tool
- [Part 6 — Stuck orders tool](#part-6--stuck-orders-tool) — 1 tool
- [Part 7 — Status Overview tools](#part-7--status-overview-tools) — 3 tools
- [Part 8 — Comparison Dashboard tools](#part-8--comparison-dashboard-tools) — 15 tools
- [Part 9 — Item Data Completeness Dashboard tools](#part-9--item-data-completeness-dashboard-tools) — 9 tools
- [Part 10 — PM Operations Dashboard tools](#part-10--pm-operations-dashboard-tools) — 9 tools
- [Part 11 — Founder / Customer Intelligence / Quotation Analysis Dashboard tools](#part-11--founder--customer-intelligence--quotation-analysis-dashboard-tools) — 53 tools (see companion doc for full detail)

**Total: 151 tools**, confirmed by reading `mcp._tool_registry` on a live site (2026-09-10).

---

## Part 1 — Doctype tools

**File:** `lh/lyfe_hardware/mcp_tools/doctypes.py`. One `get_<doctype>`/`list_<doctype>` pair per allowlisted doctype, auto-generated by the vendored fork's `expose_doctype()`, then wrapped by `lh`'s own `register()` for field masking, Single-doctype blocking, filter/sort-inference blocking, and list-limit clamping.

### 1. API Details

- **Tool names:** `get_<snake_case_doctype>`, `list_<snake_case_doctype>` — one pair per doctype below.
- **Allowlisted doctypes (26):**
  `Lyfe Order`, `Item`, `Lyfe BOM`, `Custom BOM Items`, `Project`, `Task`, `Project Task Status`, `SLA Rule`, `SLA Task Link`, `SLA Violation Cache`, `SLA Escalation Log`, `Quotation`, `Sales Invoice`, `Daily Work Log`, `External Learning Record`, `Reconciliation Flag`, `Employee`, `Attendance`, `Employee Checkin`, `Leave Application`, `Appraisal`, `Employee Skill Map`, `Energy Point Log`, `Energy Point Rule`, `Job Opening`, `LMS Enrollment` — 26 doctypes × 2 tools = 52.
- **Registration:** `expose_doctype(mcp, doctype)` from `apps/frappe-mcp/frappe_mcp/server/tools/doctype.py`, called once per doctype inside `register()`.

### 2. Functional Details

- **`get_<doctype>(name)`** — fetch one record by its name (primary key), returned as a full field dict, the same shape as opening that document in the Desk.
- **`list_<doctype>(filters, fields, limit, offset, order_by)`** — list matching records, similar to a Frappe list view or report.
- Every field on the doctype is potentially returned by `get_`; `list_` returns only the `fields` the caller asks for (default `["name"]` if omitted).

### 3. Technical Details

- **`get_<doctype>` signature:** `(name: str) -> dict`.
- **`list_<doctype>` signature:** `(filters: dict | None, fields: list[str] | None, limit: int = 20, offset: int = 0, order_by: str | None) -> list[dict]`.
- **Underlying implementation:**
  - `get_` calls `frappe.get_doc(doctype, name)`, then `doc.apply_fieldlevel_read_permissions()` (Frappe core), then `doc.as_dict()`.
  - `list_` calls `frappe.db.get_list(doctype, filters=filters or {}, fields=fields or ["name"], limit=limit, start=offset, order_by=order_by)` — a direct SQL query, no field-level filtering built in at the Frappe-core layer.
- **List-limit clamping (`_clamp_list_limit`):** the fork's own JSON-schema `minimum: 1` on `limit` is never actually enforced by the live request path (confirmed: the dispatcher calls the handler function directly, bypassing the schema validator) — a caller passing `limit=0` would otherwise get Frappe's own `DatabaseQuery` behavior of "no LIMIT clause at all," returning the entire table. `lh`'s wrapper force-clamps `limit` into `[1, 200]` before the query runs, for every caller except Super Admin/Administrator (see Security). `offset` is always normalized to a non-negative integer regardless of role.
- **Filter/sort-inference blocking (`_require_no_masked_filter_or_sort`):** a caller could otherwise infer a permlevel-masked field's real value without it ever appearing in the output — e.g. binary-searching a masked Currency field via repeated `filters={"cost_of_goods": [">", X]}` calls, or sorting by a masked email field and reading off the row order. Before the query runs, every fieldname referenced in `filters`/`order_by` is checked against the doctype's permlevel-masked fields for the caller; if any match, the whole call is rejected with `PermissionError` rather than silently answering.
- **Field masking (`_strip_masked_keys`):** applied to the *result* after the underlying handler runs. Deletes (does not null) any key on a permlevel-restricted field the caller's roles can't read — including fields nested inside child-table rows returned by `get_<doctype>` (e.g. `ShipStation Order Item.cogs` inside a `Lyfe Order`'s `order_items`). This exists because Frappe core's own masking (`apply_fieldlevel_read_permissions()`) only deletes the in-memory attribute; `Document.as_dict()` then re-populates the key from the field's type default (e.g. `0.0` for Currency) — so a masked field's key and a misleading default value would otherwise both survive into the JSON response. `frappe.db.get_list()` (used by `list_`) never applies permlevel masking at all on its own.

### 4. Security

- **Base permission gate:** `frappe.has_permission(doctype, "read", doc=name_or_None, throw=False)` inside the fork's own handler — raises the Python builtin `PermissionError` (not `frappe.PermissionError`) if the caller can't read the doctype at all. This is standard Frappe role/DocPerm/Custom DocPerm/User Permission evaluation — no MCP-specific role list layered on top for these 26 doctypes (contrast with the dashboard tools, which add an explicit dashboard-role gate).
- **Permlevel field masking:** any field with a non-zero `permlevel` the caller's role doesn't have read access to (per the doctype's `DocPerm`/`Custom DocPerm` rows) is stripped from the response entirely — this is what protects fields like `Lyfe Order.cost_of_goods` (permlevel 4), `Lyfe Order.customer_email`/`phone_no`, and `Item.cost_of_goods_sold` from callers who hold base doctype read but not the higher permlevel.
- **Single (Settings) doctypes are structurally excluded.** No Settings-type doctype (`ShipStation Settings`, `CJ Settings`, etc.) is ever on the allowlist — `_assert_no_single_doctypes_allowlisted()` fails the whole site's migrate/restart loudly if one is ever added by mistake, and `_require_not_single()` is a runtime backstop rejecting any Single doctype for non-Super-Admin callers even if that assertion were ever bypassed.
- **Child tables are not independently exposed.** `Daily Work Log Task/Addendum/Missing Task/Time Log`, `Employee Skill` have no standalone `get_`/`list_` tool — not a permission workaround, but because a child doctype called with no parent doc (`list_<child>`) is unreachable by any caller except Administrator anyway (Frappe's `has_child_permission()` returns `False` outright with no parent doc to resolve), and a standalone tool for them would be confusing/redundant since they only have meaning nested inside their parent. Child rows are still visible nested inside their parent's `get_<doctype>` result. `Custom BOM Items` is the one child table that *is* independently exposed — verified safe because it has no `DocPerm`/`Custom DocPerm` rows of its own, so `has_permission()` correctly delegates to its actual parent record's permission via `has_child_permission()`, not a bypass.
- **Super Admin / Administrator exemption:** both are structurally exempt from every guard in this file — field masking, filter/sort-inference blocking, and the list-limit clamp (their `limit` passes through untouched, including `0`/unset meaning "no cap," matching Frappe's own `DatabaseQuery` semantics). This is a deliberate, explicit policy (confirmed via a self-audit that found at least one doctype — `Quotation`, permlevel 1 — that had NOT granted Super Admin at that permlevel in its DocPerm rows, meaning Super Admin was being silently masked before this exemption was made structural rather than DocPerm-dependent).
- **HR doctypes (2026-07-27):** governed by the exact same mechanism as every other doctype here — no separate masking, no separate role. A Customer Service-role user calling `get_employee`/`list_employee` gets denied (`No read permission on Employee`); an HR User-role user succeeds and gets the full document, per `Employee`'s existing `Custom DocPerm` rows.

---

## Part 2 — PnL / financial tools

**File:** `lh/lyfe_hardware/mcp_tools/pnl.py`. Delegates to the existing PnL Dashboard report (`pnl_dashboard.py`) via `frappe.call()`.

### `get_pnl_summary`

**1. API Details** — delegates to `lh.lyfe_hardware.page.pnl_dashboard.pnl_dashboard.get_summary`.

**2. Functional Details** — revenue/cost/profit-margin KPIs for a period, with month-over-month and year-over-year comparison. Same numbers a founder sees on the Desk PnL Dashboard for the same filters.

**3. Technical Details** — `(filters: str | None = None) -> dict`. `filters` is a raw JSON string, e.g. `'{"from_date": "2026-01-01", "to_date": "2026-06-30"}'`; omitted for the dashboard's default range. No new calculation logic — pure passthrough.

**4. Security** — gated by `require_founder_ai_or_super_admin()`: caller must hold `Super Admin` or the narrowly-scoped `Founder-AI` role (granted only to named founder accounts). No other role, regardless of any Lyfe Order/Quotation permission they otherwise hold, can call this tool. `pnl_dashboard.py`'s own `_check_pnl_permission()` additionally requires base `Lyfe Order` read — both checks must pass.

### `get_pnl_monthly_trend`

**1. API Details** — delegates to `pnl_dashboard.get_monthly_trend`.

**2. Functional Details** — monthly revenue/cost/profit trend for up to the last 12 months.

**3. Technical Details** — `(filters: str | None = None) -> dict`, shaped `{"granularity": <str>, "rows": [<up to 12 monthly rows>]}` — a dict, not a bare list (the return annotation was previously wrong and has been corrected to match the always-dict shape).

**4. Security** — identical gate to `get_pnl_summary`: Super Admin/Founder-AI only.

### `get_pnl_repeat_customers`

**1. API Details** — delegates to `pnl_dashboard.get_repeat_customers_list`.

**2. Functional Details** — customers with more than one order in a period: name, order count, total order value.

**3. Technical Details** — `(filters: str | None = None) -> list`. The underlying function accepts an `include_email` parameter; this tool **always** calls it with `include_email=False`, hardcoded, regardless of what the MCP caller requests — there is no way for a caller to make this tool return email addresses.

**4. Security** — Super Admin/Founder-AI only, same gate as the other two PnL tools. Email is never returned by this tool at all, for any caller — a separate, code-level restriction independent of the role gate (since raw-SQL results in `pnl_dashboard.py` bypass Frappe's ORM-level permlevel masking entirely, the email exclusion has to be enforced here explicitly rather than relying on field masking).

---

## Part 3 — Forecasting tools

**File:** `lh/lyfe_hardware/mcp_tools/forecasting.py`. Delegates to real, backtested exponential-smoothing calculations (`lyfe_hardware/forecasting/exponential_smoothing.py`) and the Founder Briefing's stockout projection — not an invented model.

### `get_demand_forecast`

**1. API Details** — delegates to `exponential_smoothing.get_sku_demand_forecast_api` (if `sku` given) or `get_item_group_demand_forecast_api` (otherwise).

**2. Functional Details** — one-month-ahead demand forecast, by item group or by SKU, from Lyfe Order sales history. Every result includes a 0–100 confidence score, a `confidence_level` (High/Medium/Low), a reason string, and `trend_direction`/`trend_pct` versus the prior month.

**3. Technical Details** — `(item_group: str | None = None, sku: str | None = None, history_months: int | None = None) -> dict`. Provide `item_group` OR `sku` to scope to one; provide neither to get every item group ranked by forecasted demand (SKU-level with neither filter is capped at 50 results). `history_months` defaults to `Founder Briefing Settings.forecast_history_months` (12). Backtested accuracy: roughly 100–115% MAPE at the item-group level for the default engine — a directional signal, not a precise prediction; the tool's own docstring instructs Claude to present it that way.

**4. Security** — `require_founder_ai_or_super_admin()` — Super Admin or Founder-AI only. Sales/demand data is treated as commercially sensitive in the same class as PnL data.

### `get_sku_demand_forecast`

**1. API Details** — delegates to `exponential_smoothing.get_sku_demand_forecast_api`.

**2. Functional Details** — one-month-ahead demand forecast for every SKU with sales history, ranked highest-forecast first. Use for "which SKUs will have the highest demand next month"; use `get_demand_forecast(sku=...)` for one specific SKU.

**3. Technical Details** — `(history_months: int | None = None, limit: int = 50) -> dict`.

**4. Security** — Super Admin/Founder-AI only, same gate.

### `get_stockout_forecast`

**1. API Details** — delegates to `lh.lyfe_hardware.founder_briefing.data.get_projected_stockouts_api`.

**2. Functional Details** — items projected to run out of stock within a given number of days. Formula: current on-hand qty (Stock Ledger Entry) ÷ average daily sales outflow over the trailing 30 days. Items with no sales in the last 30 days are excluded (an idle SKU isn't "about to run out"). Same calculation the Daily Founder Briefing uses, so numbers always match.

**3. Technical Details** — `(lookahead_days: int | None = None) -> list`. Each result includes `risk_level` (Critical/High/Medium/Low, thresholds configurable in Founder Briefing Settings) and a `recommended_action` based on whether an open Purchase Order already covers the item. `lookahead_days` defaults to `Founder Briefing Settings.stockout_lookahead_days` (7).

**4. Security** — Super Admin/Founder-AI only, same gate.

### `get_channel_demand_forecast`

**1. API Details** — delegates to `exponential_smoothing.get_channel_demand_forecast_api`.

**2. Functional Details** — one-month-ahead demand forecast by sales channel (`order_source` — e.g. Shopify, Etsy). Same engine/confidence/trend pipeline as `get_demand_forecast`, grouped by channel.

**3. Technical Details** — `(order_source: str | None = None, history_months: int | None = None) -> dict`. Omit `order_source` for every channel, ranked by forecasted demand.

**4. Security** — Super Admin/Founder-AI only, same gate.

---

## Part 4 — Receivables tool

**File:** `lh/lyfe_hardware/mcp_tools/receivables.py`.

### `get_unpaid_orders`

**1. API Details** — raw SQL against `tabQuotation`, no delegation (this tool's own query, documented as deliberately not reusing Sales Invoice fields — see below).

**2. Functional Details** — confirmed orders (Quotations actually sent to or committed by the customer — not internal drafts/reviews) where the amount received is less than the order total. Answers "receivables overdue," but deliberately named and scoped for what this business actually tracks: Sales Invoices here exist only to generate export/GST PDFs, are almost always Draft, and every submitted one shows `outstanding_amount == grand_total` with no real credit terms — building a receivables tool on Sales Invoice fields was investigated and confirmed to produce a large, entirely fictitious "overdue" balance. The real payment signal is `Quotation.custom_total_received_amount`, populated by Shopify webhooks/CS-confirmed bank transfers, captured before any Sales Invoice exists.

**3. Technical Details** — `(limit: int = 50) -> list`. Only considers `docstatus = 1` (submitted) Quotations in one of the "committed, customer-facing" states — `Sent to Customer`, `Won`, `Bank Transfer Selected`, `Payment Received`, `Partially Paid` — explicitly excluding internal negotiation stages (Draft, Pending Approval, Pending Engineer Review, etc.), since a quotation still under internal discussion isn't an order yet and including it would overstate real exposure. Returns `name, customer_name, order_type, workflow_state, transaction_date, grand_total, custom_total_received_amount, unpaid_amount` (computed as `grand_total - custom_total_received_amount`), ordered largest-unpaid-first.

**4. Security** — `require_founder_ai_or_super_admin()` — Super Admin or Founder-AI only.

---

## Part 5 — Stock tool

**File:** `lh/lyfe_hardware/mcp_tools/stock.py`.

### `get_stock_level`

**1. API Details** — direct `frappe.get_all("Bin", ...)` query, no delegation.

**2. Functional Details** — current stock quantity for one item, summed across all warehouses, with a per-warehouse breakdown. Answers "what's our current stock for item X" directly (a raw Bin/Stock Ledger Entry lookup tool would only answer row-level questions). For "what's about to run out," use `get_stockout_forecast` instead — a distinct tool using sales velocity, not a point-in-time snapshot.

**3. Technical Details** — `(item_code: str) -> dict`, shaped `{"item_code": ..., "total_qty": ..., "by_warehouse": [{"warehouse": ..., "qty": ...}, ...]}`. Throws if `item_code` doesn't exist. Reads `Bin.actual_qty` only — deliberately does **not** include `Bin.valuation_rate`/`stock_value` (cost-adjacent fields, same sensitivity class as the permlevel-gated `Lyfe Order.cost_of_goods`, but carrying no permlevel restriction of their own on `Bin`) — excluded from this tool's output by design rather than left to leak inventory valuation.

**4. Security** — `require_founder_ai_or_super_admin()` — Super Admin or Founder-AI only. This tool was originally left open to any enabled user (reasoning at the time: stock quantity alone is less sensitive than financial data); on review that was judged inconsistent with gating every other Founder-AI-domain tool uniformly, and the gate was tightened to match.

---

## Part 6 — Stuck orders tool

**File:** `lh/lyfe_hardware/mcp_tools/stuck_orders.py`.

### `get_stuck_orders`

**1. API Details** — delegates to `lh.lyfe_hardware.founder_briefing.data.get_stuck_orders_api`.

**2. Functional Details** — Lyfe Orders stuck in production: an Active/Escalated SLA Task Link open for at least `threshold_days`, oldest first. Same data the Daily Founder Briefing's "Stuck Orders" section and the Founder Dashboard's Production tile use — no second definition of "stuck."

**3. Technical Details** — `(threshold_days: int | None = None, order_type: str | None = None) -> list`. `threshold_days` defaults to `Founder Briefing Settings.stuck_order_threshold_days` (5). `order_type` optionally restricts to `"Custom"` or `"Standard"`.

**4. Security** — `require_founder_ai_or_super_admin()` — Super Admin or Founder-AI only. This tool originally shipped with **no permission check at all** — a security audit confirmed an Engineer-role user (holding neither role) could get real customer names and real `total_amount` for every stuck order in one call; fixed by adding the same shared gate every other Founder-AI-domain tool uses.

---

## Part 7 — Status Overview tools

**File:** `lh/lyfe_hardware/mcp_tools/status_overview.py`. Wraps the Lyfe Orders Status Overview dashboard, which has **no Desk role restriction** (`roles: []` — visible to any Desk user).

### `get_orders_status_overview`

**1. API Details** — delegates to `lh.lyfe_hardware.page.lyfe_orders_status_overview.lyfe_orders_status_overview.get_orders`.

**2. Functional Details** — Lyfe Order rows and summary counts for the Status Overview dashboard — same data any Desk user can already see on that page.

**3. Technical Details** — `(filters: str | None = None) -> dict`. `filters` JSON string: `from_date`, `to_date`, `order_status`, `order_source`, `order_priority`, `show_completed`.

**4. Security** — **No role gate at all** (this tool has no `require_*` call) — matches the underlying page's own `roles: []`, i.e. any enabled Frappe user (past the base `_require_enabled_user()` check every tool has). However: the underlying dashboard's raw SQL selects `cost_of_goods` (Lyfe Order permlevel 4) and `customer` (permlevel 2) into every row with no permission awareness of its own. This tool adds its own field-stripping (`_strip_sensitive_dashboard_fields`) on top: `cost_of_goods` is removed from the response for any caller who lacks Lyfe Order's `cost_of_goods` permlevel access, and `customer` is removed for any caller who lacks the `customer` permlevel — checked independently of the page-level "anyone can see this" access, which governs the rows themselves but not these two specific fields. Administrator is exempt from this stripping.

### `get_orders_reshipments`

**1. API Details** — delegates to `lyfe_orders_status_overview.get_reshipments`.

**2. Functional Details** — Lyfe Order Reshipment rows and summary counts.

**3. Technical Details** — `(filters: str | None = None) -> dict`, same filter shape as `get_orders_status_overview`.

**4. Security** — same as `get_orders_status_overview`: no role gate, but field-stripping applies to `reshipment_cogs`/`shipment_cost` (masked by Lyfe Order's `cost_of_goods` permlevel — they represent the same real-world cost data, there's no separate "reshipment cost" permission concept in this business) and `customer` (masked by Lyfe Order's `customer` permlevel).

### `get_active_reshipments`

**1. API Details** — delegates to `lyfe_orders_status_overview.get_active_reshipments`.

**2. Functional Details** — all Lyfe Order Reshipments not yet Completed/Cancelled — a current snapshot, no date filters.

**3. Technical Details** — `() -> dict`.

**4. Security** — same as the other two tools in this file: no role gate, cost/customer field-stripping applies.

**Note on write endpoints:** the underlying Desk page also has `create_gate_pass_for_order` and `mark_orders_ready_for_dispatch` — these are intentionally **not** exposed as MCP tools; this server is read-only at the library level.

---

## Part 8 — Comparison Dashboard tools

**File:** `lh/lyfe_hardware/mcp_tools/comparison_dashboard.py`. All 15 tools delegate to `lh.lyfe_hardware.page.comparison_dashboard.comparison_dashboard` via `frappe.call()`.

**Shared Security note:** every tool in this file calls `require_dashboard_role("comparison_dashboard")` first — allowed roles: **Factory, Super Admin, System Manager, Customer Service** (mirrors `comparison_dashboard.json`'s Page role list; the underlying Python module runs raw SQL with no permission check of its own, so this MCP-layer gate is the only thing standing between an unauthorized caller and this dashboard's data via this connector).

| Tool | Signature | Functional summary |
|---|---|---|
| `get_comparison_weekly` | `(customer: str \| None) -> dict` | This-week vs last-week order comparison: count, on-time %, revenue. |
| `get_comparison_monthly` | `(customer: str \| None) -> dict` | This-month vs last-month, same metrics. |
| `get_comparison_weekly_trend` | `(weeks: int = 8, customer: str \| None) -> list` | Weekly order trend for the last N weeks. |
| `get_comparison_monthly_trend` | `(months: int = 12, customer: str \| None) -> list` | Monthly order trend for the last N months. |
| `get_comparison_top_customers` | `(from_date, to_date, limit: int = 10) -> list` | Top customers by order volume/value for a range. |
| `get_comparison_status_breakdown` | `(from_date, to_date, customer) -> list` | Order count by status (excludes Merged/Split). |
| `get_comparison_open_orders_aging` | `(customer) -> list` | Open (undelivered) orders bucketed by age. |
| `get_comparison_lead_time_trend` | `(weeks: int = 8, customer) -> list` | Avg days order-creation-to-delivery, per week. |
| `get_comparison_ontime_by_customer` | `(from_date, to_date, limit: int = 10, min_orders: int = 3) -> list` | On-time delivery % per customer. |
| `get_comparison_orders_by_country` | `(from_date, to_date, customer) -> list` | Order volume by shipping country. |
| `get_comparison_orders_by_state` | `(from_date, to_date, customer, limit: int = 15) -> list` | Order volume by US shipping state, top N. |
| `get_comparison_order_source_breakdown` | `(from_date, to_date, customer) -> list` | Order volume + on-time % by order source. |
| `get_comparison_order_type` | `(from_date, to_date, customer) -> dict` | Standard vs Custom: count, on-time %, avg lead time. |
| `get_comparison_factory_stage_timing` | `(from_date, to_date, customer) -> list` | Avg days spent between each factory pipeline milestone. |
| `get_comparison_hold_analysis` | `(from_date, to_date, customer) -> dict` | Orders on hold: count, %, avg hold days, held+delayed overlap. |

**Technical Details (all 15):** every tool is a thin `frappe.call(f"{_MODULE}.<underlying_function>", **kwargs)` passthrough — no aggregation logic duplicated in the MCP layer. All date/customer params are optional; omitting `customer` returns company-wide data, omitting date range uses that underlying function's own default window.

---

## Part 9 — Item Data Completeness Dashboard tools

**File:** `lh/lyfe_hardware/mcp_tools/item_data_completeness.py`. All 9 tools delegate to `lh.lyfe_hardware.page.item_data_completeness.item_data_completeness`.

**Shared Security note:** every tool calls `require_dashboard_role("item_data_completeness")` first — allowed roles: **Factory, Engineer, Customer Service, Super Admin, System Manager** (mirrors `item_data_completeness.json`).

| Tool | Signature | Functional summary |
|---|---|---|
| `get_item_completeness_field_list` | `() -> list` | Tracked data-completeness fields (see COGS note below). |
| `get_item_completeness_summary` | `(item_group) -> dict` | Missing-count per tracked field + total active item count. |
| `get_item_completeness_missing_items` | `(field, item_group, search, limit=500, variant_only=0) -> list` | Items missing a specific field. |
| `get_item_completeness_combined_missing_items` | `(fields, item_group, search, limit=500, variant_only=0) -> list` | Items missing ANY of the given fields. |
| `get_item_completeness_listed_product_counts` | `(item_group) -> dict` | Count of active items by "is listed product" YES/NO. |
| `get_item_completeness_listed_product_items` | `(value, item_group, search, limit=500, variant_only=0) -> list` | Active items filtered by that YES/NO value. |
| `get_item_completeness_sku_bom_counts` | `(item_group) -> dict` | Counts: child SKUs, multi-item-BOM parents, no-BOM items. |
| `get_item_completeness_sku_bom_items` | `(card_type, item_group, search, limit=500, variant_only=0) -> list` | Items for one SKU/BOM card type (`child_sku`/`parent_items`/`no_bom`). |
| `get_item_completeness_item_groups` | `() -> list` | Distinct item groups with ≥1 active item. |

**Security — Cost of Goods Sold field-level restriction (in addition to the base dashboard gate above):** `cost_of_goods_sold` is one of the tracked completeness fields, but it is additionally gated to **Factory, Super Admin, System Manager** only (narrower than the base 5-role dashboard set — Engineer and Customer Service can use every other tool/field in this file but not this one). `get_item_completeness_field_list`/`get_item_completeness_summary` already omit it from their output for non-allowed roles (delegated to the underlying module's own `_can_see_cogs()`/`_visible_fields()`). `get_item_completeness_missing_items`/`get_item_completeness_combined_missing_items` take an explicit `field`/`fields` argument, though — a caller could otherwise pass `field="cost_of_goods_sold"` directly to a drill-down endpoint and bypass the summary-level hiding; `_require_cogs_field_allowed()` blocks this explicitly at the MCP layer before the call is made, rather than relying on the underlying function to re-check it.

---

## Part 10 — PM Operations Dashboard tools

**File:** `lh/lyfe_hardware/mcp_tools/pm_operations_dashboard.py`. All 9 tools delegate to `lh.lh_project.page.pm_operations_dashboard.pm_operations_dashboard`.

**Shared Security note:** 8 of the 9 tools call `require_dashboard_role("pm_operations_dashboard")` — allowed roles: **System Manager, Projects Manager, Founder, Customer Service, Factory** (mirrors `pm_operations_dashboard.json`).

| Tool | Signature | Functional summary |
|---|---|---|
| `get_pm_action_center` | `(project, department, priority, days_inactive=3, date_from, date_to) -> dict` | Overdue/stalled/at-risk task counts for the Action Center panel. |
| `get_pm_action_center_task_list` | `(key, project, department, priority, days_inactive=3, date_from, date_to) -> list` | Task list for one Action Center drill-down category. |
| `get_pm_sla_risk` | `(project, department, priority) -> dict` | Active SLA violations, escalation risk, breach trend. |
| `get_pm_project_health_scores` | `(project, department, priority, date_from, date_to) -> list` | Summary health-score cards per project. |
| `get_pm_overdue_summary` | `(project, department, priority, date_from, date_to) -> dict` | Overdue task severity buckets + worst-offender projects. |
| `get_pm_department_goals` | `(project, department, priority, date_from, date_to) -> list` | Per-project/department goals: created/closed/completion %/overdue %/SLA %. |
| `get_pm_average_metrics` | `(project, department, priority, date_from, date_to) -> dict` | Average response/close-time metrics, trend vs previous period. |
| `get_pm_senior_review_summary` | `(date_from, date_to) -> dict` | Three manager-facing tiles for Senior Drawing Review. |

**Security — `get_pm_user_efficiency` is separately, more tightly gated:**

`get_pm_user_efficiency(project, department, priority, date_from, date_to) -> dict` — a compact user-efficiency leaderboard (named, ranked individual staff performance). This is **not** gated by `require_dashboard_role("pm_operations_dashboard")` like the other 8 — it calls `require_roles({"Super Admin", "HR User", "HR Manager"}, "the User Efficiency leaderboard")` instead, a narrower set. Reason: this is the one tool in the file returning named individual-performance data (`efficiency_score` per person, "overloaded" status) rather than operational aggregate counts — Customer Service/Factory legitimately need the other 8 tools for daily work, but individual performance rankings are restricted further, per explicit scope decision (this was fixed after a security audit found it reachable by Customer Service/Factory, which was correct for the file's other tools but not for this one).

---

## Part 11 — Founder / Customer Intelligence / Quotation Analysis Dashboard tools

**53 tools total** (25 Founder + 11 Customer Intelligence + 17 Quotation Analysis). Full detail — every tool's exact signature, permission gate, field-level stripping rules, and security rationale — is documented in the companion file **`mcp-dashboard-api-documentation.md`**, organized dashboard-by-dashboard. Summary only, here:

| Dashboard | Tool prefix | Count | Allowed roles (base gate) |
|---|---|---|---|
| Founder Dashboard | `get_founder_*` | 25 | System Manager, Super Admin |
| Customer Intelligence Dashboard | `get_cid_*` | 11 | System Manager, Super Admin, Customer Service |
| Quotation Analysis Dashboard | `get_quotation_*` | 17 | System Manager, Sales Manager, Sales User |

All three dashboards' base gate (`require_dashboard_role()`) is backed by the single `DASHBOARD_ROLES` dict in `mcp_audit.py`, which is also read directly by each dashboard's own backend `_check_permission()` — the same policy is enforced whether a caller reaches the data via MCP or by calling the whitelisted Desk-side Python function directly. See the companion document for the full per-tool breakdown, including the Customer Intelligence Dashboard's additional per-field sensitivity stripping (email, COGS, quotation pricing, order history, business-sensitive $ figures).
