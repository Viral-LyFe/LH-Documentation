# Stock Forecast Dashboard - API Documentation

Audience: developers. Scope: every API that the Stock Forecast page (`/app/stock-forecast`) calls, plus the MCP tools and legacy whitelisted functions that expose the same data, plus the scheduled jobs that feed it.

Source of truth is the code (paths relative to `apps/lh/lh/lyfe_hardware/`):

| Area | File |
|---|---|
| Desk endpoints | `page/founder_dashboard/founder_dashboard.py` |
| Page JS (tile engine, shared with the Founder Dashboard) | `page/founder_dashboard/founder_dashboard.js` |
| Stock page bootstrap | `page/stock_forecast/stock_forecast.js`, `stock_forecast.json` |
| Forecast core | `founder_briefing/data.py`, `stock_forecast/{snapshot,policy,metrics,bom,quality,alerts,accuracy}.py` |
| MCP tools (dashboard) | `mcp_tools/founder_dashboard.py` |
| MCP tools (forecasting) | `mcp_tools/forecasting.py` |
| Role sets, MCP audit wrapper | `integrations/mcp_audit.py` |
| Schedules | `../hooks.py` (`scheduler_events["cron"]`) |

Example responses below were produced on 2026-10-07 on site `lyfelocal.com` as `Administrator`, by calling the functions directly and trimming arrays (`...` = truncated). Values are real data at that time.

---

## Contents

1. Overview and data-fetch flow
2. Authentication and calling conventions
3. Permissions matrix
4. Settings that change responses
5. Caching, snapshot vs live
6. Shared row fields (stockout row)
7. Common errors
8. Desk endpoints (19)
9. MCP tools, dashboard family (19)
10. MCP tools, forecasting family (5) and legacy whitelisted function (1)
11. Scheduled jobs
12. Calls that exist in the shared JS but are NOT made by the stock page
13. Verification notes

---

## 1. Overview and data-fetch flow

`/app/stock-forecast` is a thin page. `hooks.py` appends `founder_dashboard.js` to the page script (`page_js = {"stock-forecast": [...]}`), which defines `window.lh_mount_founder_dashboard`. `stock_forecast.js` calls it with `{mode: "stock", title: "Stock Forecast"}` on `on_page_load`, and re-mounts on `on_page_show` if the Founder Dashboard took over `window.lh_fd_active`. `STOCK_MODE = opts.mode === "stock"`.

All Desk calls use `frappe.call` with `method = MP + "." + <function>` where `MP = "lh.lyfe_hardware.page.founder_dashboard.founder_dashboard"`.

```
page load  -> load() -> STOCK_MODE -> loadStockPage()
   1. skeleton grid, "Loading..."
   2. get_stock_forecast_page()         -> {tiles[], categories[]}
        - sets TILE_VISIBILITY for every key in STOCK_PAGE_TILES from `tiles`
        - sets FILTER_CATEGORY_OPTIONS = categories (Category filter dropdown)
        - on error: "Failed to load - see Error Log."
   3. builds one slot per visible tile (KPI strip, hidden data-quality block, STOCK_PAGE_GRID rows)
   4. STOCK_PAGE_TILES.filter(isTileVisible).forEach(refreshTile)   <- all tiles fire in PARALLEL
        refreshTile(key): frappe.call(MP.TILE_ENDPOINTS[key], args = resolveTileFilters(key))
        - each tile has its own loading / error state; a failing tile never blanks the others
interactions
   - per-tile refresh button  -> refreshTile(key)
   - per-tile filter (funnel) -> popover writes tileState[key].filters -> refreshTile(key)
   - tile error state         -> "Retry" button (data-retry-tile) -> refreshTile(key)   (addStockRetry)
   - primary action "Refresh" -> load() (whole page again, starting at get_stock_forecast_page)
   - "View forecast detail", chart segment, Tube/Acrylic/Material Usage "full list" -> openDrilldown(key, ...)
   - "BOMs" link in a stockout detail row -> get_where_used
   - category chart level-1 node -> get_category_forecast_children
   - Data Issues KPI / header chip -> toggles #fd-sf-dq; clicking a check line -> get_data_quality_detail
```

Stock-mode specifics:

- `get_tile_config` (founder-gated) and `get_filter_options` are NOT called in stock mode (see section 12). Layout comes from `get_stock_forecast_page`, which applies the same Tile Config visibility server-side.
- `STOCK_PAGE_TILES` (JS) mirrors `STOCK_PAGE_TILES` (Python): `stock_summary, stockout_risk, category_forecast, tube_stock, acrylic_stock, material_usage, buildable_orders, restock_list, material_shortfall, order_stock_shortfall, data_quality`.
- Fixed grid (`STOCK_PAGE_GRID`, col-span of 12): stockout_risk 6, material_usage 6, category_forecast 12, tube_stock 6, acrylic_stock 6, buildable_orders 12, restock_list 12, material_shortfall 5, order_stock_shortfall 7. `stock_summary` is the KPI strip above the grid (rendered by `renderStockKpis`, not `renderStockSummary`, in stock mode), `data_quality` is a hidden block toggled from the header.
- `refreshTile` falls back to the full `load()` for a tile without a `TILE_ENDPOINTS` entry; every stock tile has an entry.

### Tile to endpoint map (`TILE_ENDPOINTS`, `TILE_SUPPORTED_FIELDS`)

| Tile key | Endpoint | Fields `resolveTileFilters` may send | Filter UI (`TILE_FILTER_FIELDS`) |
|---|---|---|---|
| stock_summary | get_tile_stock_summary | none (`[]`) | none (refresh only) |
| stockout_risk | get_tile_stockout_risk | `item_group`, `days` | Category select, Trailing days (number) |
| category_forecast | get_tile_category_forecast | `item_group` | Category select |
| tube_stock | get_tile_tube_stock | `days` | Trailing days (number) |
| acrylic_stock | get_tile_acrylic_stock | `days` | Trailing days (number) |
| material_usage | get_tile_material_usage | `period_days` | Trailing days (number, "default 30") |
| buildable_orders | get_tile_buildable_orders | none | none |
| restock_list | get_tile_restock_list | none | none |
| material_shortfall | get_tile_material_shortfall | none | none |
| order_stock_shortfall | get_tile_order_stock_shortfall | none | none |
| data_quality | get_tile_data_quality | none | none |

Filter resolution (`resolveTileFilters`): tile-specific override first, then global rail value only for the four standard fields (`period/channel/order_type/category`) when a tile does not declare its own list. Because every stock tile declares an explicit `TILE_SUPPORTED_FIELDS` list (none of which contains a standard field), the global filter rail never reaches a stock endpoint; the rail is hidden in stock mode (`$("#fd-filters").hide()`). Only fields in the tile's declared list are sent, so the server never sees an unexpected kwarg.

---

## 2. Authentication and calling conventions

**Desk endpoints** are `@frappe.whitelist()` methods, so they are reachable at

```
/api/method/lh.lyfe_hardware.page.founder_dashboard.founder_dashboard.<function>
```

- Browser: the normal Desk session cookie (`sid`) plus `X-Frappe-CSRF-Token` for POST. `frappe.call` does both.
- Scripts: `Authorization: token <api_key>:<api_secret>` of an enabled user holding one of the roles in section 3. Guests are rejected (`PermissionError`, see 7).
- All `@frappe.whitelist()` functions here use the default (no `allow_guest`, no `methods=` restriction). `frappe.call` sends POST; a GET with query-string args also works. Over HTTP every argument arrives as a string, so the functions coerce (`int(...)`). Where a function does not coerce (noted per API) pass the documented type.
- Return value is wrapped by Frappe as `{"message": <return>}`. The examples below show the bare return value (`message`).

**MCP tools** are served from `POST /api/method/lh.mcp.handle_mcp` (`lh/mcp.py`, private fork `frappe_mcp`), authenticated with an OAuth2 Bearer token (per user). Each tool is `@mcp.tool()` + `@audited_tool`:

- `audited_tool` first runs `_require_enabled_user()` (the User must be enabled; a disabled user's still-valid token is refused), then the tool body, then writes one row to `MCP Audit Log` (status `Success`, `Permission Denied` or `Error`) and appends the optional LH MCP Settings assistant guidance to dict results (`_attach_guidance`; list results get no sidecar key).
- Tools are invoked with the standard MCP `tools/call` request (`name`, `arguments`). The exact JSON-RPC envelope is handled by `frappe_mcp` and was not re-verified here; the examples show `name` + `arguments` only.

---

## 3. Permissions matrix

Role sets (`DASHBOARD_ROLES` in `integrations/mcp_audit.py`):

| Key | Roles |
|---|---|
| `stock_forecast` | System Manager, Super Admin, Factory |
| `founder_dashboard` | System Manager, Super Admin |
| (forecasting MCP gate) `require_founder_ai_or_super_admin()` | Super Admin, Founder-AI (no System Manager!) |
| (legacy `get_projected_stockouts_api` inline check) | System Manager, Super Admin, Founder-AI |

`Administrator` passes every role check (it carries all roles). Page `/app/stock-forecast` (`stock_forecast.json`) lists System Manager, Super Admin, Factory.

Gate implementation:

- Desk stock endpoints: `_check_stock_permission()` = `require_dashboard_role("stock_forecast")` -> `require_roles(...)`; throws `frappe.PermissionError("You do not have access to the Stock Forecast.")` (message built as `"You do not have access to {0}."` with label `the Stock Forecast`).
- Stock Data Quality adds `_require_data_quality_access()` (below).
- MCP dashboard tools: `require_dashboard_role("founder_dashboard")` first, then `frappe.call(...)` into the Desk endpoint, which applies `_check_stock_permission()` again. Net effect: the caller needs a `founder_dashboard` role AND a `stock_forecast` role, which in practice means System Manager or Super Admin (Factory cannot use MCP).
- `get_founder_stock_forecast_accuracy` is the exception: it only has the `founder_dashboard` gate (it reads `Stock Forecast Accuracy` directly, no Desk endpoint behind it).

**Data-quality visibility switch.** `_data_quality_allowed()`: `Administrator` always allowed; otherwise reads the raw `tabSingles` value of `Stock Forecast Settings.data_quality_public_view` (the raw row is read because `get_single_value` returns 0 for a never-saved Check); no row = allowed (default on); stored `0` = denied. Effects when denied: `get_tile_data_quality` and `get_data_quality_detail` throw `PermissionError("Not permitted")`; `get_tile_stock_summary` silently omits `issue_count`; `get_stock_forecast_page` omits `data_quality` from `tiles` (via `_visible_tile_keys`).

| API | System Manager | Super Admin | Factory | Founder-AI | Other roles |
|---|---|---|---|---|---|
| 19 Desk stock endpoints (section 8) | yes | yes | yes | no | no |
| of which `get_tile_data_quality`, `get_data_quality_detail` | yes if switch on | yes if switch on | yes if switch on | no | Administrator always |
| 18 `get_founder_*` MCP tools that delegate to Desk (section 9) | yes | yes | no | no | no |
| `get_founder_stock_forecast_accuracy` (MCP) | yes | yes | no | no | no |
| `get_stockout_forecast`, `get_tube_stock_forecast`, `get_acrylic_stock_forecast`, `get_component_forecast`, `get_dead_and_excess_stock` (MCP) | no | yes | no | yes | no |
| `founder_briefing.data.get_projected_stockouts_api` (Desk, direct) | yes | yes | no | yes | no |

---

## 4. Settings that change responses

**Stock Forecast Settings** (single; read through `policy.forecast_settings()`, cached doc):

| Field | Type / default | Effect on APIs |
|---|---|---|
| `demand_window_days` | Int, 30 | Default trailing window for daily use. Used when `days` is blank/invalid (`_window_days`). Also the window of the nightly snapshot. |
| `target_cover_days` | Int, 30 | `suggested_qty` restores this many days of cover; tube `keep_ft`/`order_ft`; acrylic restock; dead/excess "Excess" = more than 3x this. Per-item override: `Item.custom_target_cover_days` (when > 0). |
| `use_snapshot_tile` | Check, 0 | When on and no custom `days`, stockout rows come from the latest `Stock Forecast Snapshot` if it is at most 1 day old; otherwise live. |
| `use_backlog_aware_suggestion` | Check, 0 | Off: `suggested = daily_use x cover - available`. On: `suggested = max(committed, daily_use x cover) - on_hand`. Both rounded up, then to a multiple of MOQ. Exposed in code as `forecast_settings().backlog_aware`. |
| `enable_risk_alerts` | Check, 0 | Nightly job posts a Slack message for items that moved into Critical/High (see section 11). |
| `data_quality_public_view` | Check, 1 | Data-quality visibility switch (section 3). |
| `last_run` | Small Text, read-only | Written by the nightly snapshot job; shown by the `snapshot` data-quality check and `get_founder_stock_forecast_accuracy.last_snapshot_run`. |

**Founder Briefing Settings** (read through `forecast_settings()` / `projected_stockouts()`):

| Field | Default used when empty or 0 | Effect |
|---|---|---|
| `stock_risk_critical_days` | 7 | `days_of_stock < 7` = Critical |
| `stock_risk_high_days` | 15 | `< 15` = High |
| `stock_risk_medium_days` | 30 | `< 30` = Medium, else Low. Also the horizon of the Stockout Risk tile (`_stockout_window_days()`): only rows with `days_of_stock <` this are returned unless `include_all`. |
| `stockout_lookahead_days` | 7 | Only the legacy `get_projected_stockouts_api` / `get_stockout_forecast` when `lookahead_days` is omitted. |
| `slack_webhook_url` | - | Target of the risk alert. |

(On `lyfelocal.com` these four fields are stored as 0 at the time of writing, so the code defaults apply.)

Code constants: `EXCLUDED_STOCK_WAREHOUSES = ("FAC-Rejected Material Warehouse",)` (excluded from on hand and from consumption); `_AT_RISK_LEVELS = (Critical, High, Medium)`; `snapshot.MAX_DAYS_STORED = 365`; `ROD_LENGTH_FT` 100 (tube rods); acrylic length classes `<=12, 13-24, 25-48, >48`; `bom.MIN_CALIBRATION_ORDERS = 20`, `RELIABLE_RATIO = (0.7, 1.3)`.

`_window_days(days)`: `min(max(int(days), 1), 365)` when `days` is not `None`/`""`; on `TypeError`/`ValueError`, or blank, returns `demand_window_days`.

---

## 5. Caching, snapshot vs live

- **45 s row cache.** `_stockout_rows(item_group, include_all, days)` caches the result of `_stockout_rows_uncached` in `frappe.cache()` for 45 seconds under the key `fd_stockout_rows:{item_group}:{include_all}:{days}:{use_snapshot_tile}:{backlog_aware}`. The key includes the two settings flags, so toggling either takes effect immediately (a different key). It is not user-specific. It is shared by: `get_tile_stockout_risk`, `get_tile_stock_summary`, `get_tile_category_forecast`, `get_category_forecast_children`, `get_tile_restock_list`, `get_stock_forecast_page` (category list), `get_stockout_risk_drilldown`. Changes to stock entries or orders can therefore take up to 45 s to appear in those endpoints. Tube, acrylic, material usage, buildable orders, material/order shortfall, data quality and where-used are not cached.
- **Snapshot vs live.** `_stockout_rows_uncached`: if `days is None` (no custom window) and `use_snapshot_tile` is on, `snapshot.latest_rows(item_group)` is used; it returns `None` (-> live) when there is no snapshot or the newest `snapshot_date` is older than yesterday (`max_age_days=1`). With `include_all` all stored rows are returned; otherwise rows with `days_of_stock < stock_risk_medium_days`. A custom `days` is always live. Live path: `projected_stockouts(lookahead_days=window, velocity_window_days=days or demand_window_days, item_group)` then `snapshot.enrich_rows(..., target_cover_days)`; `window = snapshot.MAX_DAYS_STORED (365)` when `include_all`, else the Medium threshold.
- **Snapshot row differences.** Rows read from the table have `snapshot_date` and no `movement_weeks` (not in `latest_rows` field list). `get_tile_stockout_risk` reports `source = "Snapshot · <date>"` (the separator is a middle dot) when the first row has `snapshot_date`, else `"Live"`.
- On `lyfelocal.com` `Stock Forecast Snapshot` is currently empty (`Last Snapshot Run` blank) and `use_snapshot_tile = 0`, so all examples are Live.

---

## 6. Shared row fields (stockout row)

Rows returned by `get_tile_stockout_risk.top_items`, `get_stockout_risk_drilldown`, `get_component_forecast`, the snapshot table. Built by `get_projected_stockouts` (first 8 fields) then `snapshot.enrich_rows`.

| Field | Type | Meaning / units |
|---|---|---|
| `item_code` | string | Item name. |
| `item_name` | string | Item.item_name (falls back to code). |
| `item_group` | string/null | Item.item_group. |
| `current_qty` | float | On hand: `SUM(Bin.actual_qty)` over all warehouses except `FAC-Rejected Material Warehouse`. Item's own unit. |
| `avg_daily_outflow` | float | Units per day: Material Issue stock-entry outflow in the window / window days, rounded to 2 dp. Only items with outflow > 0 appear. |
| `days_of_stock` | float | `round(current_qty / avg_daily_outflow, 1)`. |
| `risk_level` | string | Critical / High / Medium / Low from the Founder Briefing thresholds; forced to `Critical` when `available_qty < 0` (`metrics.effective_risk`). |
| `recommended_action` | string | Stock-only text; replaced with "Open orders already exceed stock - restock now." when available < 0 and the base level was not Critical. |
| `committed_qty` | float | Open-order demand not yet issued (order lines taken as themselves + quantities implied by the Lyfe BOM). The BOM part is dropped when the component's BOM reliability is `Unreliable`. |
| `bom_committed_qty` | float | The BOM-implied part, always shown. |
| `bom_calibration` | float/null | Ratio actual issue / BOM-predicted over 90 days (null when no calibration). |
| `bom_reliability` | string | `Reliable` / `Unreliable` / `Unknown`, or `""` when there is neither BOM quantity nor calibration. |
| `available_qty` | float | `current_qty - committed_qty` (negative = open orders exceed stock). |
| `available_days` | float/null | `available_qty` (floored at 0) / daily use, 1 dp. |
| `stockout_date` | date string/null | today + floor(days_of_stock). |
| `suggested_qty` | number | Quantity to order to restore target cover, MOQ applied (see section 4 for the two formulas). UI column "Order qty". 0 when nothing needed. |
| `confidence` | string | `High` (>= 8 of the last 13 weeks with consumption), `Medium` (>= 4), else `Low`. |
| `movement_weeks` | int | Weeks (of the last 13, 91 days) with any Material Issue. Absent on snapshot-sourced rows. |
| `trend_pct` | int/null | Added only by `get_tile_stockout_risk`: current window daily use vs the 90-day average, `round((use/avg90 - 1) x 100)`; null when there is no 90-day history or the window is >= 90. Not on drilldown rows. |
| `blocked_amount` | float | UI label "Revenue exposed". `SUM(quantity x unit_price)` of this SKU's order lines (`ShipStation Order Item`, `adjustment = 0`) on Lyfe Orders whose status is not Cancelled/Merged/Split/Shipped/Completed/Return Successfully AND that have no submitted `Material Issue` Stock Entry (`custom_lyfe_order`). Currency: order currency as stored (USD). Present on `get_tile_stockout_risk` rows and on drilldown rows only with `include_all=1` (0 for non-at-risk items). |
| `blocked_orders` | int | UI label "Orders exposed": distinct Lyfe Orders in that sum. |
| `snapshot_date` | date | Snapshot rows only. |

---

## 7. Common errors

| Situation | Behaviour |
|---|---|
| Caller lacks role (Desk) | `frappe.PermissionError`, message `You do not have access to the Stock Forecast.` (HTTP 403). Real output for a Guest: `PermissionError: You do not have access to the Stock Forecast.` |
| Data quality switch off, caller is not Administrator | `PermissionError`, message `Not permitted`. |
| Unexpected exception inside a `get_tile_*` / `get_category_forecast_children` body | `_log_and_reraise(tile_key)`: `frappe.log_error(title="[founder_dashboard][tile:<key>]: independent refresh failed", message=traceback)` (title for children: `...[tile:category_forecast_children]...`) then re-raises the original exception. The page `error:` callback sets `t.error = true`, renders the error panel and appends a **Retry** button (`addStockRetry`). Other tiles are unaffected. The permission check runs BEFORE the try block, so permission errors are not logged to Error Log. |
| Endpoints with no `_log_and_reraise` wrapper | `get_stock_forecast_page`, `get_data_quality_detail`, `get_where_used`, all four `*_drilldown` functions: exceptions propagate as plain Frappe errors (no tile-specific Error Log entry). `get_stock_forecast_page` failing shows "Failed to load - see Error Log."; a drilldown failing leaves the dialog on its skeleton. |
| `get_data_quality_detail` unknown key | `frappe.ValidationError("Unknown data quality check")` (HTTP 417). |
| `stock_summary` decision cards fail | Try/except inside `_stock_summary_section` around `_buildable_orders_section()` + `_restock_list_section()`: logs `[stock_summary] decision cards failed`, and the response simply omits `short_orders`, `short_value`, `ready_orders`, `reorder_count`; the rest of the summary is still returned. The KPI strip only renders those cards when the field is defined. |
| `_rolled_up_rows` (category forecast) tube/acrylic failure | That pseudo row is skipped and `[founder_dashboard][_rolled_up_rows]: <key>` logged; the tile still returns. |
| MCP: not an enabled user | `_require_enabled_user()` fails; logged as `Error`/`Permission Denied` per exception type and re-raised. |
| MCP: role missing | `PermissionError` re-raised unchanged; audit row status `Permission Denied`. |
| MCP: any other exception (including `ValidationError` from the Desk layer, bad arguments, DB errors) | Audit row status `Error` with the real message, then re-raised as `frappe.ValidationError("An internal error occurred while running this tool. It has been logged.")` so SQL/schema text never reaches the client. |
| Non-numeric `days` | `_window_days` swallows it and uses the Settings window. `period_days` on `get_material_usage_drilldown`, and `include_all`/`exact` on `get_stockout_risk_drilldown`, are coerced with `int(...)` without a guard, so non-numeric input raises `ValueError`. |

---

## 8. Desk endpoints

Base path for every endpoint: `/api/method/lh.lyfe_hardware.page.founder_dashboard.founder_dashboard.<name>`.
Permission for all of them: `_check_stock_permission()` (role key `stock_forecast`: System Manager, Super Admin, Factory), unless stated.

Generic call forms used in the examples:

```js
frappe.call({ method: "lh.lyfe_hardware.page.founder_dashboard.founder_dashboard.<name>", args: {...} })
```
```bash
curl -s -X POST "https://<site>/api/method/lh.lyfe_hardware.page.founder_dashboard.founder_dashboard.<name>" \
  -H "Authorization: token <api_key>:<api_secret>" -H "Content-Type: application/json" -d '{...}'
```
```bash
# local, as Administrator
bench --site lyfelocal.com execute lh.lyfe_hardware.page.founder_dashboard.founder_dashboard.<name> --kwargs '{...}'
```

### 8.1 `get_stock_forecast_page`

- **Kind / path:** Desk whitelisted method, `.../founder_dashboard.get_stock_forecast_page`.
- **Purpose:** page layout. Which stock tiles Tile Config leaves visible, plus the Item Group list for the Category filter. Needed because `get_tile_config` is founder-gated and Factory cannot call it. Tile data itself comes from each tile's own endpoint.
- **Parameters:** none.
- **Filters:** none.
- **Permission:** `_check_stock_permission()`. `data_quality` is included in `tiles` only if `_data_quality_allowed()`.
- **Implementation:** `visible = _visible_tile_keys()` (Founder Dashboard Settings > tile_config rows with `visible = 1`; a tile with no config row defaults to visible; keys come from `TILE_REGISTRY`). `categories` = sorted distinct `item_group` of `_dedupe_rows(_stockout_rows(None, include_all=True))` (shares the 45 s cache).
- **Response:**

| Field | Type | Meaning |
|---|---|---|
| `tiles` | string[] | Keys of `STOCK_PAGE_TILES`, in priority order, that are visible. |
| `categories` | string[] | Item Groups of the items on screen (those with usage), alphabetical. Not the full Item Group tree. |

- **Errors:** `PermissionError` only; no `_log_and_reraise`.
- **Example request:** `frappe.call({method: MP + ".get_stock_forecast_page"})`
- **Example response (real, `categories` truncated):**

```json
{
  "tiles": ["stock_summary", "stockout_risk", "category_forecast", "tube_stock", "acrylic_stock",
            "material_usage", "buildable_orders", "restock_list", "material_shortfall",
            "order_stock_shortfall", "data_quality"],
  "categories": ["ACCESSORIES", "ACRYLIC ROD", "..."]
}
```

- **Page usage:** first call of `loadStockPage()`; sets `TILE_VISIBILITY` and `FILTER_CATEGORY_OPTIONS`. If no tile and no `stock_summary` is visible the page shows "No tiles configured for this page - check Founder Dashboard Tile Config."

### 8.2 `get_tile_stock_summary`

- **Kind / path:** Desk whitelisted, `.../get_tile_stock_summary`.
- **Purpose:** KPI strip numbers over the FULL at-risk list (not just the top 10), plus decision cards. Same rows and blocked-$ helper (`_blocked_by_item`) as the category chart, so numbers reconcile with it.
- **Parameters:**

| Name | Type | Req | Default | Meaning / validation |
|---|---|---|---|---|
| `item_group` | string | no | none | Narrow to an Item Group (includes child groups). Empty string treated as none (`item_group or None`). Not validated; an unknown group yields zero counts. The JS never sends it (`TILE_SUPPORTED_FIELDS.stock_summary = []`), MCP can. |

- **Permission:** `_check_stock_permission()`; `issue_count` only if `_data_quality_allowed()`.
- **Response:**

| Field | Type | Meaning |
|---|---|---|
| `critical_count` / `high_count` / `medium_count` | int | At-risk items per level (deduplicated by item_code). |
| `total_at_risk` | int | Sum of the three. |
| `shortest` | object/null | `{item_code, item_group, days_of_stock}` for the item with the least cover among rows that have a `days_of_stock`; null if none. |
| `blocked_amount` | float | "Revenue exposed": sum of line values on at-risk items. |
| `blocked_orders` | int | Distinct open orders in that sum. |
| `short_orders` | int | Buildable Orders `blocked_orders` (Partial + Short). Omitted if the decision-card block failed. |
| `short_value` | float | Buildable `blocked_value`. |
| `ready_orders` | int | Buildable `ready_count`. |
| `reorder_count` | int | Number of rows in the Restock List. |
| `issue_count` | int | Data-quality checks with severity not `ok`. Omitted when data quality is not visible to the caller. |
| `period_label` | string | Static: `Live - projected from trailing 30-day usage` (em dash in the code). |

- **Errors:** `_log_and_reraise("stock_summary")`; decision cards fail soft (section 7).
- **Example request:** `frappe.call({method: MP + ".get_tile_stock_summary"})` (no args from the page).
- **Example response (real):**

```json
{"critical_count": 141, "high_count": 6, "medium_count": 15, "total_at_risk": 162,
 "shortest": {"item_code": "20L-R001-SBU", "item_group": "Iron Door Pulls", "days_of_stock": 0.0},
 "blocked_amount": 455.0, "blocked_orders": 1,
 "period_label": "Live — projected from trailing 30-day usage",
 "short_orders": 13, "short_value": 7073.33, "ready_orders": 6, "reorder_count": 168, "issue_count": 6}
```

- **Page usage:** `TILE_ENDPOINTS.stock_summary`; rendered by `renderStockKpis` (KPI cards "Orders short now", "Items to reorder", "Shortest cover", "Data issues"; each card scrolls to its tile via `data-sf-go`, the data-issues card toggles `#fd-sf-dq`). No filters.

### 8.3 `get_tile_stockout_risk`

- **Kind / path:** Desk whitelisted, `.../get_tile_stockout_risk`.
- **Purpose:** Stockout Risk tile: every item whose cover is under the Medium threshold, with committed demand, stock-out date, suggested order quantity and revenue exposed.
- **Parameters:**

| Name | Type | Req | Default | Meaning / validation |
|---|---|---|---|---|
| `item_group` | string | no | none | Item Group incl. descendants. `""` = no filter. |
| `days` | int (or numeric string) | no | Settings `demand_window_days` (30) | Trailing window for daily use. Clamped 1-365 by `_window_days`; blank or non-numeric = Settings window. A window different from the Settings window forces the live path (no snapshot, `source = "Live"`). |

- **Filters supported:** category (`item_group`), trailing days (`days`). No warehouse filter by design (on hand is summed across warehouses).
- **Response:**

| Field | Type | Meaning |
|---|---|---|
| `critical_count`, `high_count`, `medium_count` | int | Counts per level among returned rows. |
| `total_at_risk` | int | `len(top_items)`. |
| `top_items` | object[] | EVERY row with `days_of_stock <` Medium threshold, most urgent first (name kept for compatibility; the founder page slices to 6, the stock page scrolls all). Row fields: section 6 including `trend_pct`, `blocked_amount`, `blocked_orders`. Tube and acrylic rod items are not here (own tiles); Make-to-Order / Do-Not-Stock policy items excluded. |
| `window_days` | int | Effective (clamped) window. |
| `source` | string | `Live` or `Snapshot · YYYY-MM-DD`. |
| `period_label` | string | `Live - projected from trailing {days}-day usage`. |

  If there are no rows, `top_items` is empty and `blocked_amount`/`trend_pct` enrichment is skipped.
- **Errors:** `_log_and_reraise("stockout_risk")`.
- **Example request:** `args: {item_group: "Iron Door Pulls", days: 30}`
- **Example response (real, 1 row):**

```json
{"critical_count": 1, "high_count": 0, "medium_count": 0, "total_at_risk": 1,
 "top_items": [{
   "item_code": "20L-R001-SBU", "item_name": "Regency- Iron Door Pull Handle — Matte Black",
   "item_group": "Iron Door Pulls", "current_qty": 0.0, "avg_daily_outflow": 0.07, "days_of_stock": 0.0,
   "risk_level": "Critical", "recommended_action": "Stocked out — restock now.",
   "committed_qty": 0.0, "bom_committed_qty": 0.0, "bom_calibration": null, "bom_reliability": "",
   "available_qty": 0.0, "available_days": 0.0, "stockout_date": "2026-10-07", "suggested_qty": 3,
   "confidence": "Low", "movement_weeks": 1, "blocked_amount": 0.0, "blocked_orders": 0, "trend_pct": 215}],
 "window_days": 30, "source": "Live", "period_label": "Live — projected from trailing 30-day usage"}
```

- **Page usage:** `stockout_risk`; renderer `renderStockoutRisk`; filter popover (funnel) sends `{item_group, days}` from `resolveTileFilters("stockout_risk")`; "View forecast detail" opens the drilldown (8.15); segments of the Category chart reuse it.

### 8.4 `get_tile_material_usage`

- **Kind / path:** Desk whitelisted, `.../get_tile_material_usage`.
- **Purpose:** what production actually issued, from `Material Issue for Order` (MIFO), net of Returns, per item over the trailing window. Mix of raw materials and finished SKUs, as recorded by MIFO.
- **Parameters:**

| Name | Type | Req | Default | Meaning / validation |
|---|---|---|---|---|
| `period_days` | int | no | `30` | Trailing days; `_window_days` clamps 1-365; `None`/`""`/invalid = Settings window. Note the signature default is 30, so omitting the arg gives 30, not the Settings value. |

- **Filters:** trailing days only. (Page filter label "Trailing days (default 30)".)
- **Query:** `Material Issue Order Item` joined to submitted `Material Issue for Order` (`docstatus = 1`, `date >= today - period_days`), `net_qty = SUM(Issue +issue_qty, otherwise -issue_qty)`, `HAVING net_qty > 0`.
- **Response:**

| Field | Type | Meaning |
|---|---|---|
| `most_used` | `{item_code, item_name, net_qty}`[] | EVERY item, most used first (the tile scrolls). `net_qty` = issues minus returns, item's own unit. |
| `least_used` | same[] | The bottom 10 (ascending), only when more than 10 items; else `[]`. |
| `total_items` | int | Item count. |
| `window_days` | int | Effective window. |
| `period_label` | string | `Trailing {n} days`. |

- **Errors:** `_log_and_reraise("material_usage")`.
- **Example request:** `args: {period_days: 30}`
- **Example response (real, 334 items):**

```json
{"most_used": [{"item_code": "Brass Tube Material 1.5'' Dia", "item_name": "Brass Tube Material 1.5'' Dia", "net_qty": 252.21},
               {"item_code": "Brass Tube Material 2'' Dia", "item_name": "Brass Tube Material 2'' Dia", "net_qty": 186.7}, "..."],
 "least_used": [{"item_code": "2FT-TB-150-AB", "item_name": "Round Tubing — Antique Brass / 1.5\" / 2 FT", "net_qty": 1.0}, "..."],
 "total_items": 334, "period_label": "Trailing 30 days", "window_days": 30}
```

- **Page usage:** `material_usage`; renderer `renderMaterialUsage`; filter sends `period_days`; full list: drilldown 8.17.

### 8.5 `get_tile_tube_stock`

- **Kind / path:** Desk whitelisted, `.../get_tile_tube_stock`.
- **Purpose:** tube cover in FEET, one row per pipe (tube material x diameter; cut lengths rolled up), via `founder_briefing.data.get_tube_stock(days)`.
- **Parameters:**

| Name | Type | Req | Default | Meaning / validation |
|---|---|---|---|---|
| `days` | int | no | Settings `demand_window_days` | Clamped 1-365 (`_window_days`); blank/invalid = Settings window. |

- **Response:**

| Field | Type | Meaning |
|---|---|---|
| `rows` | object[] | Sorted by `days_of_stock` ascending. Only pipes with consumption in the window. |
| `rows[].item_code`, `item_name` | string | Store item (or, for unlinked cut tubes, the family code and "<family> (cut pieces)"). |
| `rows[].pipe_linked` | bool | False for cut-tube families not yet linked to a pipe (then `order_rods` is 0). |
| `rows[].used_ft` | float | Feet used in the window (rod issued + pre-cut pieces x length, less MIFO returns). |
| `rows[].ft_per_day` | float | `used_ft / window`. |
| `rows[].rod_ft`, `precut_ft`, `feet_on_hand` | float | On hand: rod Bin feet, pre-cut pieces x length, total. |
| `rows[].rods_on_hand` | float | `feet_on_hand / 100`. |
| `rows[].days_of_stock` | float | `feet_on_hand / ft_per_day`. |
| `rows[].risk_level` | string | Same thresholds as stockout. |
| `rows[].target_cover_days` | int | Item override or Settings `target_cover_days`. |
| `rows[].keep_ft` | float | `ft_per_day x target_cover_days`. |
| `rows[].order_ft` | float | `max(keep_ft - feet_on_hand, 0)`. |
| `rows[].order_rods` | int | `ceil(order_ft / 100)` (0 if not linked). |
| `critical_count`, `high_count` | int | Row counts per level. |
| `window_days` | int | Effective window. |
| `period_label` | string | `Live - trailing {n}-day usage, feet`. |

- **Errors:** `_log_and_reraise("tube_stock")`.
- **Example request:** `args: {days: 30}` (or none)
- **Example response (real, 13 rows):**

```json
{"rows": [{"item_code": "SS SOLID ROD 7MM", "item_name": "SS SOLID ROD 7MM", "pipe_linked": true,
           "used_ft": 10.7, "target_cover_days": 30, "keep_ft": 10.7, "order_ft": 10.7, "order_rods": 1,
           "rod_ft": 0.0, "precut_ft": 0.0, "feet_on_hand": 0.0, "rods_on_hand": 0.0,
           "ft_per_day": 0.36, "days_of_stock": 0.0, "risk_level": "Critical"}, "..."],
 "critical_count": 9, "high_count": 0, "window_days": 30, "period_label": "Live — trailing 30-day usage, feet"}
```

- **Page usage:** `tube_stock`; `renderTubeStock`; filter `{days}`; drilldown 8.16.

### 8.6 `get_tile_acrylic_stock`

- **Kind / path:** Desk whitelisted, `.../get_tile_acrylic_stock`.
- **Purpose:** acrylic rod cover in INCHES per diameter and stick-length class (`<=12`, `13-24`, `25-48`, `>48`), via `get_acrylic_stock(days)`. Demand is whole sticks issued (waste included) less Returns, counted in the class of the stick drawn; only classes with demand are returned.
- **Parameters:** `days` (int, optional): same as 8.5 (clamp 1-365, blank = Settings window).
- **Response:**

| Field | Type | Meaning |
|---|---|---|
| `rows[]` | object | Sorted by `days_of_stock`. |
| `rows[].diameter` | string | e.g. `1.5"`; `RAC` rod caps map via a width label. |
| `rows[].length_class` | string | `<=12`, `13-24`, `25-48`, `>48` (inches). |
| `rows[].sticks_on_hand` | int | Sticks in stock (Bins, Rejected warehouse excluded). |
| `rows[].inches_on_hand` | int | Sticks x stick length from the SKU code. |
| `rows[].inches_per_day` | float | Demand inches / window. |
| `rows[].days_of_stock` | float | `inches_on_hand / inches_per_day`. |
| `rows[].risk_level` | string | Same thresholds. |
| `critical_count`, `high_count`, `window_days` | int | As tube. |
| `period_label` | string | `Live - trailing {n}-day usage, inches`. |

- **Errors:** `_log_and_reraise("acrylic_stock")`.
- **Example response (real, 11 rows):**

```json
{"rows": [{"diameter": "1.5\"", "length_class": ">48", "sticks_on_hand": 151, "inches_on_hand": 9408,
           "inches_per_day": 158.0, "days_of_stock": 59.5, "risk_level": "Low"}, "..."],
 "critical_count": 0, "high_count": 0, "window_days": 30, "period_label": "Live — trailing 30-day usage, inches"}
```

- **Page usage:** `acrylic_stock`; `renderAcrylicStock`; filter `{days}`; drilldown `get_acrylic_stock_drilldown` (8.17).

### 8.7 `get_tile_category_forecast`

- **Kind / path:** Desk whitelisted, `.../get_tile_category_forecast`.
- **Purpose:** level 1 of the Category-wise Stock Forecast chart: the stockout dataset including items with plenty of cover (`include_all=True`: every item with usage, 365-day horizon), rolled up per top-level Item Group (children of the tree root). Also tube/acrylic pseudo rows and dead/excess counts.
- **Parameters:**

| Name | Type | Req | Default | Meaning |
|---|---|---|---|---|
| `item_group` | string | no | none | Narrows the forecast rows first, exactly like the Stockout Risk filter (includes child groups). `""` = none. |

- **Filters:** category only (`TILE_SUPPORTED_FIELDS.category_forecast = ["item_group"]`).
- **Response:**

| Field | Type | Meaning |
|---|---|---|
| `nodes[]` | object | One per top-level group, sorted by critical desc, high desc, blocked_value desc, label. |
| `nodes[].key`, `label` | string | Top-level Item Group (`"Other"` for items without a group). |
| `nodes[].items` | int | Items with usage in that group. |
| `nodes[].critical/high/medium/low` | int | Items per risk level. |
| `nodes[].at_risk` | int | critical + high + medium. |
| `nodes[].blocked_value` | float | "Revenue exposed" of at-risk items (line-level, never double counted). |
| `nodes[].blocked_orders` | int | Distinct orders (a parent is the union of children, not the sum). |
| `nodes[].direct_only` | bool | True when every item sits directly in that top-level group (no sub-groups to expand). |
| `pseudo[]` | object | Rows for `tube` ("Tube (feet)", tile `tube_stock`) and `acrylic` ("Acrylic rod (inches)", tile `acrylic_stock`): `key, label, tile, items, critical, high, medium, low, at_risk`. |
| `totals` | object | `items, critical, at_risk, blocked_value, blocked_orders` (distinct). |
| `dead`, `excess` | int | Counts from `policy.dead_and_excess()` (90-day window, Settings target cover; not filtered by `item_group`). |
| `period_label` | string | Static `Live - items with usage in the trailing 30 days`. |

- **Errors:** `_log_and_reraise("category_forecast")`.
- **Example response (real, trimmed):**

```json
{"nodes": [{"key": "Railings", "label": "Railings", "items": 150, "critical": 84, "high": 6, "medium": 12, "low": 48,
            "blocked_value": 455.0, "at_risk": 102, "blocked_orders": 1, "direct_only": false}, "... (6 total)"],
 "pseudo": [{"key": "tube", "label": "Tube (feet)", "tile": "tube_stock", "items": 13, "critical": 9, "high": 0,
             "medium": 1, "low": 3, "at_risk": 10}, {"key": "acrylic", "...": "..."}],
 "totals": {"items": 225, "critical": 141, "at_risk": 162, "blocked_value": 455.0, "blocked_orders": 1},
 "dead": 220, "excess": 66, "period_label": "Live — items with usage in the trailing 30 days"}
```

- **Page usage:** `category_forecast`; `renderCategoryForecast`; clicking a node: no children (`direct_only`) -> popup drilldown; else `get_category_forecast_children`.

### 8.8 `get_category_forecast_children`

- **Kind / path:** Desk whitelisted, `.../get_category_forecast_children`.
- **Purpose:** level 2 of the chart: the items' own Item Groups under one top-level group.
- **Parameters:**

| Name | Type | Req | Default | Meaning |
|---|---|---|---|---|
| `group` | string | yes | - | Top-level group name, taken from `get_tile_category_forecast().nodes[].key`. Not validated; unknown group returns empty `nodes`. A missing argument raises a Frappe "missing argument" error. |

- **Response:** `{group, nodes[], direct_only}`. `nodes[]` has the level-1 fields without the per-node `direct_only` (`key, label, items, critical, high, medium, low, blocked_value, at_risk, blocked_orders`), grouped by the items' own `item_group`. `direct_only` is true when all nodes equal the group itself.
- **Errors:** `_log_and_reraise("category_forecast_children")`.
- **Example request:** `args: {group: "Railings"}`
- **Example response (real, trimmed):**

```json
{"group": "Railings",
 "nodes": [{"key": "BRACKETS", "label": "BRACKETS", "items": 54, "critical": 26, "high": 1, "medium": 5, "low": 22,
            "blocked_value": 455.0, "at_risk": 32, "blocked_orders": 1},
           {"key": "SHORT POSTS", "label": "SHORT POSTS", "items": 15, "critical": 14, "high": 0, "medium": 0,
            "low": 1, "blocked_value": 0.0, "at_risk": 14, "blocked_orders": 0}, "..."],
 "direct_only": false}
```

- **Page usage:** `[data-cf-node]` click handler in `renderCategoryForecast` (sets `catFc.group`, `catFc.loading`; on error it resets to level 1). Clicking a level-2 node opens drilldown `category_forecast` with `{item_group, include_all: 1, exact: 1}`.

### 8.9 `get_tile_buildable_orders`

- **Kind / path:** Desk whitelisted, `.../get_tile_buildable_orders`.
- **Purpose:** which active orders can be built from stock on hand. Active = Lyfe Orders in `_UNDISPATCHED_STATUSES` (New ... Awaiting India Components, 15 statuses) with Custom lines without a BOM skipped. Requirements come from `bom.open_order_requirements` (order's own component list, else Lyfe BOM components with scrap, else child SKUs of a previous shipped order, else the item). Stock = Bin quantity minus the Rejected warehouse. Orders are served oldest `order_date` first (then name); only a Ready order takes stock (`bom.allocate_orders`).
- **Parameters:** none. **Filters:** none.
- **Response:**

| Field | Type | Meaning |
|---|---|---|
| `orders[]` | object | One per active order, oldest first. |
| `orders[].order_name` | string | Lyfe Order name. |
| `orders[].internal_order_id` | string | LH id. |
| `orders[].order_type` | string | Standard / Custom. |
| `orders[].order_date` | string | `YYYY-MM-DD`. |
| `orders[].order_value` | float | Order value. |
| `orders[].status` | string | `Ready` (all components available), `Partial` (some short), `Short` (every component short). |
| `orders[].short[]` | object | Limiting components `{item_code, required_qty, available_qty (what was left at its turn), missing_qty}`. |
| `ready_count`, `partial_count`, `short_count` | int | Orders per status. |
| `blocked_orders` | int | Partial + Short. |
| `blocked_value` | float | Their order value ("Blocked $"). |
| `ready_value` | float | Value of Ready orders. |

- **Errors:** `_log_and_reraise("buildable_orders")`.
- **Example response (real, 19 orders, trimmed):**

```json
{"orders": [{"order_name": "LYF-SH-2026-1965", "internal_order_id": "LH3082", "order_type": "Standard",
             "order_date": "2026-09-23", "order_value": 1019.98, "status": "Ready", "short": []},
            {"order_name": "LYF-SH-2026-1972", "internal_order_id": "LH3090", "order_type": "Standard",
             "order_date": "2026-09-23", "order_value": 544.02, "status": "Partial",
             "short": [{"item_code": "DEC-150-AB", "required_qty": 2.0, "available_qty": 0.0, "missing_qty": 2.0}, "..."]}, "..."],
 "ready_count": 6, "partial_count": 7, "short_count": 6, "blocked_orders": 13,
 "blocked_value": 7073.33, "ready_value": 2185.06}
```

- **Page usage:** `buildable_orders`; `renderBuildableOrders`; also feeds the KPI strip via `get_tile_stock_summary`.

### 8.10 `get_tile_restock_list`

- **Kind / path:** Desk whitelisted, `.../get_tile_restock_list`.
- **Purpose:** one buy list: every Critical/High/Medium item with `suggested_qty > 0` (from the include_all stockout rows, deduplicated), plus tube pipes with `order_rods > 0` and acrylic classes with inches to order (`max(inches_per_day x target_cover - inches_on_hand, 0)`), in the unit they are bought. Restocking is booked as Stock Entry (Material Receipt), so there is no "on order" signal; `last_received` shows the latest receipt instead.
- **Parameters:** none. **Filters:** none (tube/acrylic use the default window).
- **Sort:** `(available_qty >= 0, Critical < High < Medium, days_of_stock)`: open orders already above stock first.
- **Response:**

| Field | Type | Meaning |
|---|---|---|
| `rows[].item_code` | string | Item code; for Acrylic a label `<diameter> · <class>"`. |
| `rows[].item_name` | string | Name (Acrylic: "Acrylic rod (diameter and stick-length class)"). |
| `rows[].kind` | string | `Item`, `Tube`, `Acrylic`. |
| `rows[].risk_level` | string | Critical/High/Medium. |
| `rows[].on_hand` | float | Units / feet / inches by kind. |
| `rows[].available_qty` | float | On hand minus committed (Tube/Acrylic: equals on hand). |
| `rows[].days_of_stock` | float/null | Cover. |
| `rows[].order_qty` | float | Units (MOQ applied) / rods / inches. |
| `rows[].unit` | string | `units`, `rods (100 ft; N ft short)`, `inches`. |
| `rows[].last_received` | string | Latest submitted Material Receipt date, `""` if none (always `""` for Acrylic). |
| `total` | int | Row count. |
| `target_cover_days` | int | Settings value used. |

- **Errors:** `_log_and_reraise("restock_list")`.
- **Example response (real, 168 rows, trimmed):**

```json
{"rows": [{"item_code": "BOP-100-SB", "item_name": "Ball Open Short Post for 1-inch OD Tubing", "kind": "Item",
           "risk_level": "Critical", "on_hand": 0.0, "available_qty": -3.0, "days_of_stock": 0.0,
           "order_qty": 21.0, "unit": "units", "last_received": "2026-10-01"}, "..."],
 "total": 168, "target_cover_days": 30}
```

- **Page usage:** `restock_list`; `renderRestockList` (with Export button `.fd-sf-export`); count shown as the "Items to reorder" KPI.

### 8.11 `get_tile_material_shortfall`

- **Kind / path:** Desk whitelisted, `.../get_tile_material_shortfall`.
- **Purpose:** "Material Not In Stock - Total Required": the same active-order demand as 8.9, summed per component across all active orders, listing components whose total required exceeds stock.
- **Parameters / filters:** none.
- **Response:** `rows[]` = `{item_code, required_qty (summed over active orders), in_stock}` sorted by item_code; `total` = row count. Missing = required - in_stock (the dashboard rounds required and missing up).
- **Errors:** `_log_and_reraise("material_shortfall")`.
- **Example response (real, 21 rows):**

```json
{"rows": [{"item_code": "11L-R002-RB", "required_qty": 1.0, "in_stock": 0.0},
          {"item_code": "2FT-TB-200-SB", "required_qty": 1.0, "in_stock": 0.0}, "..."],
 "total": 21}
```

- **Page usage:** `material_shortfall`; `renderMaterialShortfall`.

### 8.12 `get_tile_order_stock_shortfall`

- **Kind / path:** Desk whitelisted, `.../get_tile_order_stock_shortfall`.
- **Purpose:** "Active Orders - Material Not In Stock": one row per active order and component where the requirement exceeds the FULL stock. Stock is not shared between orders (use Buildable Orders for allocation).
- **Parameters / filters:** none.
- **Response:** `rows[]` = `{order_name, internal_order_id, order_type, order_date, order_value, item_code, required_qty, in_stock}`; `total`.
- **Errors:** `_log_and_reraise("order_stock_shortfall")`.
- **Example response (real, 22 rows):**

```json
{"rows": [{"order_name": "LYF-SH-2026-1972", "internal_order_id": "LH3090", "order_type": "Standard",
           "order_date": "2026-09-23", "order_value": 544.02, "item_code": "DEC-150-AB",
           "required_qty": 2.0, "in_stock": 0.0}, "..."],
 "total": 22}
```

- **Page usage:** `order_stock_shortfall`; `renderOrderStockShortfall`; order and item cells link to `/app/lyfe-order/<name>` and `/app/item/<code>`.

### 8.13 `get_tile_data_quality`

- **Kind / path:** Desk whitelisted, `.../get_tile_data_quality`.
- **Purpose:** Stock Data Quality checklist (BOM components with no Item, open order lines without `erp_item`, unreliable BOMs, acrylic unit/naming, pending Material Requests, dead and excess stock, forecast bias, snapshot freshness), from `stock_forecast.quality.get_data_quality()`.
- **Parameters / filters:** none.
- **Permission:** `_check_stock_permission()` then `_require_data_quality_access()` (section 3).
- **Response:**

| Field | Type | Meaning |
|---|---|---|
| `checks[].key` | string | One of: `bom_unmatched, lines_no_item, bom_unreliable, acrylic_inch, acrylic_p, pending_mr, dead, excess, bias, snapshot` (the keys accepted by 8.14). |
| `checks[].label` | string | Human label. |
| `checks[].count` | int | Number of offending records. |
| `checks[].severity` | string | `ok` when count is 0, else `warn` / `bad` / `info` as set per check. |
| `checks[].hint` | string | What goes wrong. |
| `checks[].fix` | string | Plain-words instruction. |
| `checks[].doctype` | string | DocType to open. |
| `issue_count` | int | Checks with severity not `ok`. |
| `period_label` | string | `Live`. |

- **Errors:** `PermissionError("Not permitted")` when hidden; otherwise `_log_and_reraise("data_quality")`.
- **Example response (real, 9 checks truncated to 2):**

```json
{"checks": [{"key": "bom_unmatched", "label": "Lyfe BOM components not matched to an Item", "count": 0, "severity": "ok",
             "hint": "Component SKU on a Lyfe BOM is not an Item code or custom SKU, so it is ignored in BOM demand.",
             "fix": "Open the Lyfe BOM listed, find the component row (Components table) and set Item Code to a real Item (or correct the SKU spelling).",
             "doctype": "Lyfe BOM"},
            {"key": "lines_no_item", "label": "Open order lines without an Item (erp_item)", "count": 17, "severity": "warn",
             "hint": "Their quantity cannot be counted as committed demand.",
             "fix": "Open the Lyfe Order listed, find the order line in Order Items and set its ERP Item to the right Item.",
             "doctype": "Lyfe Order"}, "... (9 total)"],
 "issue_count": 6, "period_label": "Live"}
```

- **Page usage:** `data_quality` (header chip / hidden `#fd-sf-dq` block), `renderDataQuality`; each row has `data-dq-key` that opens 8.14. The chip also relies on `issue_count` from the summary.

### 8.14 `get_data_quality_detail`

- **Kind / path:** Desk whitelisted, `.../get_data_quality_detail`.
- **Purpose:** the records behind one Stock Data Quality line, with where to fix them (`quality.get_quality_detail(key)`).
- **Parameters:**

| Name | Type | Req | Default | Meaning / validation |
|---|---|---|---|---|
| `key` | string | yes | - | A check key (see 8.13). Unknown key -> `frappe.ValidationError("Unknown data quality check")`. |

- **Permission:** stock gate + data-quality switch.
- **Response:** `{key, fix, doctype, columns[], rows[], total}`. `rows[]` = `{doctype, name, cells[]}`, capped at 200 (`LIMIT`); `total` = uncapped count; `name` is the record to open.
- **Errors:** no `_log_and_reraise` wrapper; validation and permission errors propagate.
- **Example request:** `args: {key: "snapshot"}`
- **Example response (real):**

```json
{"key": "snapshot",
 "fix": "Open Stock Forecast Settings and check Last Snapshot Run; if old, check the scheduler is running and the Error Log for 'stock_forecast.snapshot'.",
 "doctype": "Stock Forecast Settings", "columns": ["Setting", "Latest snapshot"],
 "rows": [{"doctype": "Stock Forecast Settings", "name": "Stock Forecast Settings", "cells": ["none yet"]}], "total": 1}
```

- **Page usage:** `openDataQualityDetail(key)` dialog "Stock Data Quality" (a "Where to fix" box with a link to the DocType list, then the table).

### 8.15 `get_stockout_risk_drilldown`

- **Kind / path:** Desk whitelisted, `.../get_stockout_risk_drilldown`.
- **Purpose:** item list behind the Stockout Risk card ("View forecast detail" and its CSV, the "restock-list" export) and the Category chart popups (same rows).
- **Parameters:**

| Name | Type | Req | Default | Meaning / validation |
|---|---|---|---|---|
| `item_group` | string | no | none | Includes descendants; with `exact=1` only the group's own items. `"Other"` = items without an Item Group (chart bucket). |
| `include_all` | int/bool (`0`/`1`) | no | `0` | `1` keeps items above the Medium threshold too (every item with usage, 365-day horizon), deduplicates by item_code and adds `blocked_amount` / `blocked_orders`. Coerced with `bool(int(x or 0))`; `"true"` raises ValueError. |
| `exact` | int (`0`/`1`) | no | `0` | `1`: only items whose `item_group == item_group` (a chart sub-category). |
| `days` | int | no | none | Clamped 1-365; ignored (treated as default, snapshot allowed) when equal to the Settings window. The page sends it only for `stockout_risk` when the tile has a custom `days`; the chart popup stays on the Settings window. |

- **Response:** array of rows (section 6 fields except `trend_pct`). With `include_all=1`, each row also carries `blocked_amount` (0 for non-at-risk) and `blocked_orders`. Without `include_all` the rows have no `blocked_*` fields (live path).
- **Errors:** no wrapper; `ValueError` on bad `include_all`/`exact`.
- **Example request:** `args: {item_group: "Iron Door Pulls", include_all: 1, exact: 1}`
- **Example response (real, 1 row):**

```json
[{"item_code": "20L-R001-SBU", "item_name": "Regency- Iron Door Pull Handle — Matte Black", "item_group": "Iron Door Pulls",
  "current_qty": 0.0, "avg_daily_outflow": 0.07, "days_of_stock": 0.0, "risk_level": "Critical",
  "recommended_action": "Stocked out — restock now.", "committed_qty": 0.0, "bom_committed_qty": 0.0,
  "bom_calibration": null, "bom_reliability": "", "available_qty": 0.0, "available_days": 0.0,
  "stockout_date": "2026-10-07", "suggested_qty": 3, "confidence": "Low", "movement_weeks": 1,
  "blocked_amount": 0, "blocked_orders": 0}]
```

- **Page usage:** `DRILLDOWN_CONFIG.stockout_risk` (method `get_stockout_risk_drilldown`; search box, risk filter, expandable detail rows, CSV `restock-list`), and `DRILLDOWN_CONFIG.category_forecast` (spreads stockout_risk, adds "Revenue exposed" and "Orders" columns; opened with `{item_group, include_all: 1, exact: 0|1}` and a preset risk level from the clicked chart segment; level-1 "All critical items" row sends `item_group: ""`).

### 8.16 `get_tube_stock_drilldown`

- **Kind / path:** Desk whitelisted, `.../get_tube_stock_drilldown`.
- **Purpose:** one row per pipe over the fixed last 30 days: the same rows as `get_tile_tube_stock().rows` but a bare array and always the 30-day default (`get_tube_stock()` with no window; it ignores the tile's `days` filter).
- **Parameters / filters:** none. **Response:** array of the tube row fields in 8.5. **Errors:** none wrapped.
- **Example response (real, 13 rows, 1 shown):**

```json
[{"item_code": "SS SOLID ROD 7MM", "item_name": "SS SOLID ROD 7MM", "pipe_linked": true, "used_ft": 10.7,
  "target_cover_days": 30, "keep_ft": 10.7, "order_ft": 10.7, "order_rods": 1, "rod_ft": 0.0, "precut_ft": 0.0,
  "feet_on_hand": 0.0, "rods_on_hand": 0.0, "ft_per_day": 0.36, "days_of_stock": 0.0, "risk_level": "Critical"}, "..."]
```

- **Page usage:** `DRILLDOWN_CONFIG.tube_stock` ("Tube Stock - Use, Stock to Keep and Order per Pipe (Trailing 30 Days)"); columns Used (30d, ft), Per day, On hand, Days left, Keep (ft) (+ target days), Order (ft), Order (rods), Risk.

### 8.17 `get_acrylic_stock_drilldown`, `get_material_usage_drilldown` (two small drilldowns)

**`get_acrylic_stock_drilldown`**: Desk whitelisted, `.../get_acrylic_stock_drilldown`. No parameters. Calls `get_acrylic_usage_breakdown()` (30 days): pieces and inches used per diameter and cut length, largest inches first. Response: array of `{diameter (string), cut_length (float, inches), length_class (string), pieces (int), inches (int)}`. No error wrapper. Page: `DRILLDOWN_CONFIG.acrylic_stock`.

```json
[{"diameter": "1.5\"", "cut_length": 58.0, "length_class": ">48", "pieces": 18, "inches": 1044},
 {"diameter": "1.5\"", "cut_length": 72.0, "length_class": ">48", "pieces": 14, "inches": 1008}, "... (45 total)"]
```

**`get_material_usage_drilldown`**: Desk whitelisted, `.../get_material_usage_drilldown`. Parameter `period_days` (int, default 30). Unlike the tile this one is NOT clamped: `window_start = add_days(today, -int(period_days))`; non-numeric raises `ValueError`. Same SQL as 8.4 without the 10-row split; response is a bare array `[{item_code, item_name, net_qty}]` (most issued first). Page: `DRILLDOWN_CONFIG.material_usage` ("Material Usage - Full List (Trailing 30 Days)"), called without args, so always 30 days even if the tile filter was changed.

```json
[{"item_code": "Brass Tube Material 1.5'' Dia", "item_name": "Brass Tube Material 1.5'' Dia", "net_qty": 252.21},
 {"item_code": "Brass Tube Material 2'' Dia", "item_name": "Brass Tube Material 2'' Dia", "net_qty": 186.7}, "... (334 total)"]
```

### 8.18 `get_where_used`

- **Kind / path:** Desk whitelisted, `.../get_where_used`.
- **Purpose:** Lyfe BOMs that use a component item, with how many open orders carry the parent (`stock_forecast.bom.where_used`).
- **Parameters:**

| Name | Type | Req | Meaning |
|---|---|---|---|
| `item_code` | string | yes | Matched against `Custom BOM Items.item_code` OR `.sku`. Unknown/unused returns `[]`. |

- **Response:** array sorted by `open_orders` desc then `parent_item`: `{parent_item (Lyfe BOM parent_id), parent_name, qty_per_unit (SUM of component quantity per parent), open_orders (distinct open orders with the parent as erp_item)}`. `open_orders` counts orders in `bom.open_order_names()` (not closed status and no submitted Material Issue).
- **Errors:** none wrapped.
- **Example request:** `args: {item_code: "CMBR-150-AB"}`
- **Example response (real):**

```json
[{"parent_item": "10FT-BFK-AB-150", "parent_name": "Custom Antique Brass Bar Foot Rail Kit — 10 FT / 1.5”", "qty_per_unit": 3.0, "open_orders": 0},
 {"parent_item": "12FT-BFK-AB-150", "parent_name": "Custom Antique Brass Bar Foot Rail Kit — 12 FT / 1.5”", "qty_per_unit": 4.0, "open_orders": 0}, "..."]
```

  (For `20L-R001-SBU`, which is in no Lyfe BOM, the result is `[]`; the dialog then shows "Not used in any Lyfe BOM.")
- **Page usage:** the "BOMs" link (`.fd-where-used`) in the detail row of the Stockout Risk drilldown; shown in `frappe.msgprint` "Where used - <item>".

---

## 9. MCP tools, dashboard family (`mcp_tools/founder_dashboard.py`)

All tools: `@mcp.tool()` + `@audited_tool`; first `require_dashboard_role("founder_dashboard")` (System Manager, Super Admin); then `frappe.call(f"{_MODULE}.<desk endpoint>", ...)` so the stock gate and data-quality switch apply again. Returns exactly what the Desk endpoint returns (dict results also get `_attach_guidance`). Errors per section 7 (permission passes through, anything else becomes the generic `ValidationError`). Wire call shape: `tools/call` with `name` and `arguments`.

| MCP tool | Arguments (type, default) | Delegates to | Response = |
|---|---|---|---|
| `get_founder_tile_stock_summary` | `item_group: str\|None = None` | `get_tile_stock_summary` (8.2) | 8.2 |
| `get_founder_tile_stockout_risk` | `item_group: str\|None = None`, `days: int\|None = None` (1-365; custom = live) | `get_tile_stockout_risk` (8.3) | 8.3 |
| `get_founder_tile_category_forecast` | `item_group: str\|None = None` | `get_tile_category_forecast` (8.7) | 8.7 |
| `get_founder_category_forecast_children` | `group: str` (required) | `get_category_forecast_children` (8.8) | 8.8 |
| `get_founder_tile_material_usage` | `period_days: int = 30` | `get_tile_material_usage` (8.4) | 8.4 |
| `get_founder_tile_tube_stock` | `days: int\|None = None` | `get_tile_tube_stock` (8.5) | 8.5 |
| `get_founder_tile_acrylic_stock` | `days: int\|None = None` | `get_tile_acrylic_stock` (8.6) | 8.6 |
| `get_founder_tile_restock_list` | none | `get_tile_restock_list` (8.10) | 8.10 |
| `get_founder_tile_buildable_orders` | none | `get_tile_buildable_orders` (8.9) | 8.9 |
| `get_founder_tile_material_shortfall` | none | `get_tile_material_shortfall` (8.11) | 8.11 |
| `get_founder_tile_order_stock_shortfall` | none | `get_tile_order_stock_shortfall` (8.12) | 8.12 |
| `get_founder_tile_data_quality` | none | `get_tile_data_quality` (8.13) | 8.13 |
| `get_founder_data_quality_detail` | `key: str` (required) | `get_data_quality_detail` (8.14) | 8.14 |
| `get_founder_stockout_risk_drilldown` | `item_group: str\|None`, `include_all: int = 0`, `exact: int = 0`, `days: int\|None` | `get_stockout_risk_drilldown` (8.15) | 8.15 (array) |
| `get_founder_tube_stock_drilldown` | none | `get_tube_stock_drilldown` (8.16) | array |
| `get_founder_acrylic_stock_drilldown` | none | `get_acrylic_stock_drilldown` (8.17) | array |
| `get_founder_material_usage_drilldown` | `period_days: int = 30` | `get_material_usage_drilldown` (8.17) | array |
| `get_founder_where_used` | `item_code: str` (required) | `get_where_used` (8.18) | array |
| `get_founder_stock_forecast_accuracy` | `weeks: int = 8` | none (direct query, see 9.1) | see 9.1 |

Notes:

- There is no MCP tool for `get_stock_forecast_page` (layout only, nothing to read for an AI client). The `STOCK_PAGE_TILES` tools (summary, stockout, category, tube, acrylic, material_usage, buildable, restock, material_shortfall, order_stock_shortfall, data_quality) all exist.
- Return type annotations in the code (`-> dict` / `-> list`) are informational; MCP tools returning arrays (drilldowns, where_used) get no guidance sidecar.
- `get_founder_tile_material_usage` and the data-quality tools are described in the tool docstrings as "same result as the dashboard tile".
- A typed `int` MCP argument that the client sends as a string is the client's concern; the Desk endpoint re-coerces (`days`, `period_days`, `include_all`, `exact`).

### 9.0 Example (MCP)

```json
{"name": "get_founder_tile_stockout_risk", "arguments": {"item_group": "Iron Door Pulls", "days": 30}}
```
Result: identical to the 8.3 response (plus an optional guidance key).

```json
{"name": "get_founder_tile_stock_summary", "arguments": {}}
```
Result: identical to the 8.2 response.

### 9.1 `get_founder_stock_forecast_accuracy`

- **Kind / path:** MCP tool (no Desk endpoint).
- **Purpose:** how accurate the stock forecast has been: per recent week, WAPE % and bias % of the 7-day forecast against what was really issued, overall and per ABC class, plus when the nightly snapshot last ran. Empty until the snapshot has a week of history.
- **Parameters:** `weeks: int = 8`. The query is `frappe.get_all("Stock Forecast Accuracy", fields=[week_start, abc_class, items, forecast_units, actual_units, wape_pct, bias_pct], order_by="week_start desc, abc_class asc", limit_page_length=max(1, int(weeks)) * 5)`; the `x 5` assumes about five class rows per week (`All`, A, B, C, Unclassified). Not clamped above; `int(weeks)` raises on non-numeric (-> generic ValidationError).
- **Permission:** `founder_dashboard` gate only.
- **Response:** `{"last_snapshot_run": <Stock Forecast Settings.last_run string or null/"">, "weeks": [{week_start, abc_class, items, forecast_units, actual_units, wape_pct, bias_pct}]}`. `wape_pct = sum|forecast-actual| / sum actual`; `bias_pct = (sum forecast - sum actual) / sum actual` (written by `accuracy.run`). Real run on 2026-10-07: `weeks` is `[]` (no snapshot history yet) and `last_snapshot_run` is empty.
- **Example request:** `{"name": "get_founder_stock_forecast_accuracy", "arguments": {"weeks": 8}}`
- **Example response (shape; the live site has no rows yet):** `{"last_snapshot_run": "2026-10-06 01:15 OK — 512 rows in 41s", "weeks": [{"week_start": "2026-09-28", "abc_class": "All", "items": 480, "forecast_units": 1234.5, "actual_units": 1180.0, "wape_pct": 38.2, "bias_pct": 4.6}]}` (illustrative values, format of `last_run` from `snapshot._run`).
- **Page usage:** not called by the page; the data-quality `bias` / `snapshot` checks read the same tables.

---

## 10. MCP tools, forecasting family, and the legacy whitelisted function

Only the stock-related tools of `mcp_tools/forecasting.py` are documented. The sales-demand tools in that file (`get_demand_forecast`, `get_sku_demand_forecast`, `get_channel_demand_forecast`) are NOT stock APIs and are out of scope.

Gate for every tool in 10.1 to 10.5: `require_founder_ai_or_super_admin()` -> roles Super Admin or Founder-AI; else `PermissionError("Only Super Admin or Founder-AI may view financial/forecasting data via this connector.")`. System Manager alone is NOT enough here, and Factory has no access. All are `@audited_tool`. None of these tools is used by the page.

### 10.1 `get_stockout_forecast`

- **Kind / path:** MCP tool; delegates `frappe.call("lh.lyfe_hardware.founder_briefing.data.get_projected_stockouts_api", lookahead_days=...)`. Note: that function has its own role check (10.6): System Manager / Super Admin / Founder-AI, so Founder-AI and Super Admin pass both.
- **Purpose:** items projected to run out of stock within N days. Same calculation as the Daily Founder Briefing. Trailing 30-day Material Issue consumption (fixed, `velocity_window_days = 30`; no window argument), Rejected warehouse excluded; idle items excluded; tube/acrylic rod items rolled up in their own tools; Make-to-Order / Do-Not-Stock excluded.
- **Parameters:** `lookahead_days: int | None = None`; flag items with `days_of_stock <` this; empty/0 -> Founder Briefing Settings `stockout_lookahead_days` or 7.
- **Response:** array (not the enriched stockout row): `{item_code, item_name, item_group, current_qty, avg_daily_outflow, days_of_stock, risk_level, recommended_action}`, sorted by `days_of_stock`. No `committed_qty`, `available_qty` or `suggested_qty` (those need `get_founder_tile_stockout_risk` or `get_component_forecast`).
- **Errors:** section 7 (MCP).
- **Example request:** `{"name": "get_stockout_forecast", "arguments": {"lookahead_days": 30}}`
- **Example response (real via the underlying function, 2 of N rows):**

```json
[{"item_code": "20L-R001-SBU", "item_name": "Regency- Iron Door Pull Handle — Matte Black", "item_group": "Iron Door Pulls",
  "current_qty": 0.0, "avg_daily_outflow": 0.07, "days_of_stock": 0.0, "risk_level": "Critical",
  "recommended_action": "Stocked out — restock now."},
 {"item_code": "38X38-SSANGSK", "item_name": "38x38 Stainless Steel Angle 24\" Long for Sink", "item_group": "Products",
  "current_qty": 0.0, "avg_daily_outflow": 0.03, "days_of_stock": 0.0, "risk_level": "Critical",
  "recommended_action": "Stocked out — restock now."}]
```

### 10.2 `get_tube_stock_forecast`

- **Kind / path:** MCP tool; `frappe.call("lh.lyfe_hardware.founder_briefing.data.get_tube_stock")` (non-whitelisted callable invoked through `frappe.call`; fixed 30-day window).
- **Purpose:** tube cover in feet per pipe, same as the tube tile rows but a bare array.
- **Parameters:** none. **Response:** array of the tube row fields in 8.5 (`rows[]`). Caveats in the docstring: cutting of pre-cut stock is not booked against the rod (rod feet can be overstated); about 4 months of history.
- **Example:** identical in structure to the 8.16 example.

### 10.3 `get_acrylic_stock_forecast`

- **Kind / path:** MCP tool; `frappe.call("lh.lyfe_hardware.founder_briefing.data.get_acrylic_stock")` (30-day window).
- **Purpose / response:** array of the acrylic row fields in 8.6 (`diameter, length_class, sticks_on_hand, inches_on_hand, inches_per_day, days_of_stock, risk_level`). Acrylic rod SKUs are excluded from `get_stockout_forecast`; rod caps are not.
- **Parameters:** none.
- **Example row (real):** `{"diameter": "1.5\"", "length_class": ">48", "sticks_on_hand": 151, "inches_on_hand": 9408, "inches_per_day": 158.0, "days_of_stock": 59.5, "risk_level": "Low"}`

### 10.4 `get_component_forecast`

- **Kind / path:** MCP tool (no Desk equivalent).
- **Purpose:** cover, committed demand and Lyfe BOM usage for ONE stocked item. Runs `get_projected_stockouts(snapshot.MAX_DAYS_STORED)` (live, 365-day horizon, thresholds default `(7, 15, 30)` from the function default, not from Founder Briefing Settings), filters to the item, enriches with `snapshot.enrich_rows` (Settings target cover), adds `where_used`.
- **Parameters:** `item_code: str` (required).
- **Response:** the section 6 fields (without `trend_pct`/`blocked_*`) plus `where_used[]` (shape of 8.18). If the item has no consumption in the last 30 days, or is rolled up into tube/acrylic: `{"error": "No consumption in the last 30 days, or the item is rolled up into the tube/acrylic tools."}` (a normal return, not an exception).
- **Example request:** `{"name": "get_component_forecast", "arguments": {"item_code": "20L-R001-SBU"}}`
- **Example response (real via the underlying calls):**

```json
{"item_code": "20L-R001-SBU", "item_name": "Regency- Iron Door Pull Handle — Matte Black", "item_group": "Iron Door Pulls",
 "current_qty": 0.0, "avg_daily_outflow": 0.07, "days_of_stock": 0.0, "risk_level": "Critical",
 "recommended_action": "Stocked out — restock now.", "committed_qty": 0.0, "bom_committed_qty": 0.0,
 "bom_calibration": null, "bom_reliability": "", "available_qty": 0.0, "available_days": 0.0,
 "stockout_date": "2026-10-07", "suggested_qty": 3, "confidence": "Low", "movement_weeks": 1, "where_used": []}
```

### 10.5 `get_dead_and_excess_stock`

- **Kind / path:** MCP tool; `policy.dead_and_excess()`.
- **Purpose:** stocked items that are not moving: `Dead` (stock on hand > 0, nothing issued in 90 days) and `Excess` (cover more than 3x target cover at the current 90-day rate; per-item `custom_target_cover_days` honoured). Tube, acrylic rod and Make-to-Order / Do-Not-Stock items are excluded; disabled items and the Rejected warehouse are excluded. Review list, not a disposal list (about 5 months of history).
- **Parameters:** none (the Python function accepts `target_cover_days`, `window_days`, `excess_factor`, but the tool does not expose them).
- **Response:** array sorted by `kind`, then on hand desc: `{item_code, item_name, kind ("Dead"|"Excess"), on_hand (float), avg_daily_use (float, units/day), days_of_stock (float|null; null for Dead)}`.
- **Example response (real, 2 of 286 rows):**

```json
[{"item_code": "FEC-150-SBU", "item_name": "Flush End Caps for 1.5-inch OD Tubing", "kind": "Dead", "on_hand": 252.0, "avg_daily_use": 0.0, "days_of_stock": null},
 {"item_code": "FE90-100-PB", "item_name": "90 Degree Flush Elbow for 1-inch OD Tubing", "kind": "Dead", "on_hand": 208.0, "avg_daily_use": 0.0, "days_of_stock": null}]
```

  (the page's `dead`/`excess` counts in 8.7 are these two kinds counted, 220 + 66 = 286.)

### 10.6 `get_projected_stockouts_api` (legacy Desk whitelisted)

- **Kind / path:** Desk whitelisted method, `/api/method/lh.lyfe_hardware.founder_briefing.data.get_projected_stockouts_api`. Legacy entry point kept for MCP and direct callers; the Stock Forecast page does NOT call it (the page uses the ungated sibling `projected_stockouts()` after its own `stock_forecast` role check).
- **Purpose:** projected stockouts within `lookahead_days`: `current_qty` (Bin minus Rejected warehouse) / `avg_daily_outflow` (Material Issue over `velocity_window_days`).
- **Parameters:**

| Name | Type | Req | Default | Meaning |
|---|---|---|---|---|
| `lookahead_days` | int | no | Founder Briefing Settings `stockout_lookahead_days` or 7 | Only items with `days_of_stock <` this. |
| `velocity_window_days` | int | no | 30 | Trailing window for daily use (`int(x or 30)`; no clamp). |
| `item_group` | string | no | none | Item Group incl. descendants (filtered after the risk calculation). |

- **Permission:** explicit inline check `{"System Manager", "Super Admin", "Founder-AI"} & set(frappe.get_roles())`, else `frappe.PermissionError("Not permitted")`. (Explicit rather than `frappe.only_for`, which is skipped under `in_test`.) Thresholds from Founder Briefing Settings (`stock_risk_*_days`).
- **Response:** array of `{item_code, item_name, item_group, current_qty, avg_daily_outflow, days_of_stock, risk_level, recommended_action}`, sorted by `days_of_stock` asc (same as 10.1).
- **Example request:** `curl ".../api/method/lh.lyfe_hardware.founder_briefing.data.get_projected_stockouts_api?lookahead_days=30&item_group=Iron%20Door%20Pulls" -H "Authorization: token K:S"`
- **Example response:** identical to the 10.1 example.
- **Other users of the same engine:** the Daily Founder Briefing (`founder_briefing/scheduler.py`) calls `data.get_projected_stockouts`, `get_tube_stock`, `get_acrylic_stock` directly with the Settings thresholds.

---

## 11. Scheduled jobs (not request APIs)

From `hooks.py` `scheduler_events["cron"]`:

| Schedule | Function | Writes / effect |
|---|---|---|
| `15 1 * * *` (01:15 daily) | `lh.lyfe_hardware.stock_forecast.snapshot.run` | Enqueues `snapshot._run` on the `long` queue (`timeout=3600`, `job_id="stock_forecast_snapshot"`, `deduplicate=True`). `_run`: `build_rows(demand_window_days, target_cover_days)` (live `projected_stockouts(365, velocity_window_days=demand_window_days)` + `enrich_rows`) -> `write_snapshot` (deletes today's `Stock Forecast Snapshot` rows, inserts one per item; idempotent per day) -> `purge_old(400)` (delete rows older than 400 days) -> `alerts.run()` (nested try/except; failure logged `[stock_forecast] risk alert failed`, never fails the snapshot) -> writes `Stock Forecast Settings.last_run` = `"<YYYY-MM-DD HH:MM> OK - <n> rows in <s>s"` and commits. On any failure: rollback, log `[stock_forecast] nightly snapshot failed`, `last_run = "<ts> FAILED - see Error Log"`, commit, re-raise. Read by the tile only when `use_snapshot_tile` is on. |
| (inside the snapshot job) | `lh.lyfe_hardware.stock_forecast.alerts.run` | Needs `Stock Forecast Settings.enable_risk_alerts` on, at least 2 snapshot dates and `Founder Briefing Settings.slack_webhook_url`. Compares the latest snapshot with the previous one and posts one Slack message (`send_slack_message`) listing items that newly entered Critical/High (or got worse) plus Critical items that stay Critical every 7th consecutive day ("still critical"); max 15 items listed ("+N more"). Returns the number alerted. Writes nothing to the DB. |
| `30 2 * * 1` (Mon 02:30) | `lh.lyfe_hardware.stock_forecast.policy.run_abc` | Writes `Item.custom_abc_class` and `Item.custom_class_updated_on` from 90-day Material Issue volume (cumulative share: A up to 80%, B up to 95%, C rest); skips items with `custom_policy_locked = 1`; clears the class of items that lost consumption. No-op if `custom_stock_policy` column is missing. Returns the number changed. |
| `30 2 * * 1` (Mon 02:30) | `lh.lyfe_hardware.stock_forecast.accuracy.run` | Scores the latest snapshot on or before 7 days ago (needs `snapshot_date + 7 <= today`): forecast = `avg_daily_outflow x 7` vs actual Material Issue units in the following 7 days; writes/updates one `Stock Forecast Accuracy` row per class (`name = "<snapshot_date>-<All|A|B|C|Unclassified>"`, fields `week_start, abc_class, items, forecast_units, actual_units, wape_pct, bias_pct`). Returns rows written; does nothing until a week of snapshots exists. |

Cron timezone is the bench scheduler's. The briefing job `founder_briefing.scheduler.run` (hourly entry) reuses the stock functions for the Slack briefing; it is a separate feature.

---

## 12. Calls that exist in the shared JS but are NOT made by the stock page

`founder_dashboard.js` is shared with the Founder Dashboard. The following calls are in the same file but are unreachable in `STOCK_MODE` (mount skips them or they belong to other tiles), so they are not part of the Stock Forecast API surface:

| Method | Why not used on the stock page |
|---|---|
| `get_filter_options` (`loadFilterOptions`) | Only called when `!STOCK_MODE`. Founder-gated (`_check_permission`). The stock page takes its Category list from `get_stock_forecast_page().categories`. |
| `get_tile_config` (`loadTileConfig`) | Only called for the founder page; founder-gated. Replaced by `get_stock_forecast_page`. |
| `get_founder_dashboard_summary_v2` (`load` in founder mode) | Founder page only (`if (STOCK_MODE) return loadStockPage();` runs first). Also contains stock tiles for the founder page's Stock section. |
| `get_mcp_access_summary` and all non-stock `get_tile_*` / `get_*_drilldown` | Other dashboards' tiles. |

---

## 13. Verification notes

Verified directly in code: all 19 Desk endpoints, their guards and wrappers; `TILE_ENDPOINTS`, `TILE_SUPPORTED_FIELDS`, `TILE_FILTER_FIELDS`, `DRILLDOWN_CONFIG` and every `frappe.call` in `loadStockPage` / `openDrilldown` / the category and data-quality handlers; role sets; the 45 s cache key; the snapshot fallback; Settings fields and defaults; the 19 MCP dashboard tools, 5 forecasting stock tools; the cron entries. Example responses for all Desk endpoints and the underlying functions of 10.1, 10.4, 10.5 were produced by running the code as Administrator on `lyfelocal.com`. Guest rejection and the `Unknown data quality check` error were also reproduced.

Not run live (read from code only): the MCP tools themselves (running them writes `MCP Audit Log` rows), the `get_founder_stock_forecast_accuracy` populated shape (no accuracy rows exist on the site), the Slack alert, the snapshot/accuracy/ABC jobs, the snapshot-sourced response path (no snapshot exists), and the MCP JSON-RPC envelope.

Differences found between `apps/lh/docs/stock-forecast-dashboard.md` / `mcp-dashboard-api-documentation.md` and the code (the code is authoritative and is what this document states):

1. `stock-forecast-dashboard.md` (cache paragraph) says the 45 s `_stockout_rows` cache is "keyed by item group and include_all" and "does not include ... the `use_snapshot_tile` flag, so a settings change takes up to 45 s to show". The code key is `fd_stockout_rows:{item_group}:{include_all}:{days}:{use_snapshot_tile}:{backlog_aware}`, so it includes `days` and both settings flags.
2. `stock-forecast-dashboard.md` (frontend section) says the Material Usage filter is a "Trailing window select 7/14/30/60". The code (`TILE_FILTER_FIELDS.material_usage`) is a free number input "Trailing days (default 30)" -> `period_days`; the server clamps 1-365.
3. The Stockout Risk filter text says blank trailing days = 30; the code uses Stock Forecast Settings `demand_window_days` (30 by default) - and for `material_usage` the signature default `period_days=30` is a literal 30 when the argument is omitted (blank/None only then follows the Settings value).
4. `mcp-dashboard-api-documentation.md` 1.26/1.27 field lists (which end in "...") do not name tube `pipe_linked`, `rod_ft`, `precut_ft`, `target_cover_days` and describe `order_rods` without the `pipe_linked` condition; the code returns those fields and `order_rods = 0` for unlinked cut-tube families.
5. `get_founder_tile_stockout_risk` MCP docstring says tube/acrylic rod are excluded and that `trend_pct`/`blocked_*` are on rows; correct for the tile, but the drilldown rows never carry `trend_pct`, and carry `blocked_*` only with `include_all=1` (documented in 8.15; the older docs treat the rows as identical).
6. `get_founder_stock_forecast_accuracy` is called out in the older docs only as a tool; it has no Desk twin and only the `founder_dashboard` gate.
7. The `categories` list in `get_stock_forecast_page` is the Item Groups of items with usage, not the Item Group tree (consistent with the older docs, noted here because the name suggests otherwise).
8. The materialised Founder Briefing Settings values on `lyfelocal.com` are 0 for `stock_risk_*_days` and `stockout_lookahead_days`; the code treats 0/empty as 7 / 15 / 30 / 7, so the defaults apply.

Last verified against the live implementation: 2026-10
