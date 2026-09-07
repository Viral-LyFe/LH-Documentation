# Work Summary — August & September 2026

Covers `prod` and `prod-us_warehouse_integration` branches (lh app). Compiled from git history, grouped by feature area rather than commit-by-commit, since a single feature is often built across many commits.

Branches share the same history through ~Aug 24, 2026 (common ancestor). After that point they diverge: `prod` continued with day-to-day features/fixes, while `prod-us_warehouse_integration` became the dedicated branch for the US Warehouse / 1Click logistics integration.

---

## August 2026

### Shared work (both branches — through Aug 24, common history)

**Founder Dashboard & BI (BRD-DEV-009)** — biggest body of work in August.
- Built from scratch: Daily Founder Briefing (Slack digest), demand/stockout forecasting via MCP (later upgraded to Holt/Holt-Winters models, backtested against real data), Founder Dashboard consolidating 4 older dashboards into one.
- Heavy live-iteration once in front of the founder: MoM/YoY trend, per-tile drill-down, Top Customer (period + all-time), Material Usage card, Stockout/Shipment/Hold visibility, tile show/hide/reorder config, tile-level refresh/filter infrastructure, config-driven layout engine ("true modularity"), date-range filters, geography (country/state) revenue tile.
- Companion dashboards: Customer Intelligence (Phases 1-4: customer_type field, segmentation engine, dashboard, at-risk cache) and Quotation Analysis Dashboard (win-rate fixes, Pipedrive-style redesign, region dimension, stage donut).

**Quotation / Payments**
- Bank-transfer payment method built: auto-discount (3% → raised to 8%), grid rate swap, no payment link, discount shown in email.
- SKU/ERP item mismatch validator; price_list_rate backfill and blinking-field fix.
- COGS-aware pricing: category-wise and product-finish COGS multipliers, benchmark deviation detection (tuned to kill false positives).
- Drawing/change-details tracking on Quotation, permission fixes.

**Returns / Reshipments**
- Reworked full Return Details flow (reasons, statuses, validation), added Restock Fee, auto-transition InProgress → Successfully on delivery, return-leg and reship-leg tracking (mirrored fields, hourly polling), backfilled historical tracker data onto Lyfe Order.

**Order fulfillment / ShipStation / Shopify**
- SKU change detection + auto-resolve erp_item on sync; COGS preserved through sync; address/field drift healing with deadlock retry.
- Shopify refund capture fixed (missed refunds + retry/dedup/Slack alert); mark-as-shipped made async with retry verification.

**Security / MCP hardening** — recurring audit-and-close cycle across the month:
- Closed data-leak gaps in Founder AI MCP (COGS/customer masking, permlevel enforcement, record-count limits, ungated tools, double-audit-wrap bug affecting 46 tools), auto-expiry sweep fixed to revoke whole sessions.
- HRMS: restricted Daily Work Log and Employee salary/PII fields by role.

**Gorgias integration** — Phase 1 (inbound webhook → Task) shipped Aug 11; Phases 2-4 (conversation sync, reply, SLA linking) shipped Aug 17.

**Shipping calculator** — contract-rate customer markup, ODA tier surcharge, per-shipment vs per-box billing (shipped, reverted, re-fixed correctly).

**Load/scale testing & launch monitoring** — Locust + scheduler soak tests (fixed HTTPS-redirect login failure, save-task crash), week-one go-live monitoring deployed around the Aug 11 launch date, later removed once stable.

---

### prod-us_warehouse_integration only (from ~Aug 13 — 1Click/US Warehouse project kicks off)

- Discovery phase: documented and live-tested 1Click's API (Inventory, Create Order, Tracking, Create Load, Add Item) against sandbox, corrected several contract misunderstandings (priorityOrder typing, GET-body quirks).
- Built the order routing engine: Factory-only / US-Warehouse-only / MIXED (split) order detection, hold-and-resume for partial-BOM availability, manual override ("Force US"/"Force India"), human confirmation gate for mixed orders.
- New doctypes: US/Factory Warehouse Shipment Item, Transfer Order tracking, Lyfe Order Leg1 Event.
- Address routing fixed across Sales Invoice, Pick-Pack PDF, Packing List for Via-US-Warehouse vs Direct-to-Customer vs India-direct-dropship.
- Dual-leg tracking (India→US leg + US→customer leg) wired in; 1Click calls logged through Integration Request like CJ's pattern.
- SLA rules added: Leg 1 / Transfer Order Receipt detectors.
- KPI dashboard Phases 1-4: Manual Override Rate, OTIF by Route Plan, WIP/In-Transit Aging, Delivery Exceptions — all shipped by Aug 19.
- Test plan + proof docs for test cases A1-A3 compiled through Aug 24-25.

---

## September 2026 (through Sep 7)

### prod branch

- **Print formats**: new Packing List format cloned from Commercial Invoice (renamed to resolve a naming collision), Net/Gross Weight and Box Number added to Sales Invoice Item and grouped into Packing List print, consignee-address fix for direct-to-customer orders (was wrongly showing Lyfe Hardware), US Warehouse consignee address added to invoice prints.
- **New Lyfe PO doctype** with dialog-based creation from Lyfe Order, plus a new Lyfe Stock Balance report wired into the Stock Balance workspace shortcut.
- Continued dashboard modularization: PnL dashboard and Quotation Analysis dashboard both refactored into tile-registry pattern, MoM comparison sections added to both, merged-order exclusion bug fixed across PnL/comparison/order-analysis dashboards.
- Misc fixes: "Order Modified" infinite re-trigger loop (unsynced customer_notes), duplicate CRM Product error on Item creation, Email Queue auto-resume when developer_mode is off, 17Track registration casing bug, Revert Workflow State permission fix, Frappe Cloud build failure from an em-dash in a file path.
- Material Issue for Order: finish vs acrylic-cuts dialog made mutually exclusive, RAC/ACR SKU handling refined.
- Customer Intelligence: tile framework/filters/layout polish, backend split into a `cid/` subpackage, Quotation end-user guide rewritten against the live Quotation-3 workflow.

### prod-us_warehouse_integration branch — 1Click stabilization phase

September here reads as **testing, fixing real gaps, and hardening** the routing engine built in August, not new scope:
- Configured live Oneclick Settings (token/warehouse ID) and re-ran full 1Click test suite (TC1-TC5, then TC1-TC23) — all passing.
- Fixed real bugs found by testing: US-leg tracking delivery never posted Route D orders to 1Click; US-leg tracking not syncing to Transfer Order / not marking it Received; Pending List export missing US Warehouse address/item filter for MIXED orders; Pick-Pack PDF/Packing List missing US Warehouse address; Route Plan field invisible on new orders (stale field order); no-SKU order items vanishing instead of routing to Factory.
- Routing refinements: wired up `route_plan` so Force US/Force India actually routes (made synchronous instead of background job); combined MIXED orders into a single 1Click shipment instead of two (this was the last big structural fix — completed Sep 4-5); locked `factory_leg_destination` server-side after Confirm Split; fixed BOM-explosion bug in Force US plus added a confirmation dialog.
- New SLA rule: 1Click order not dispatched within 72h of submission, with a dedicated Slack alert; urgent Slack alert added for orders cancelled after 1Click posting (with a tracked PM Task).
- 1Click orders now flow through the normal 17Track tracking pipeline instead of a separate path.
- Fixed a dialog-dismiss bug that could skip the Force US/Force India confirmation entirely (Sep 7 — most recent commit on this branch).

**Bottom line for September**: `prod` shipped print-format/reporting polish and routine fixes; `prod-us_warehouse_integration` spent the month closing out real gaps found by end-to-end testing on the 1Click integration, and by Sep 7 looks feature-complete and hardened, pending merge back into `prod`.
