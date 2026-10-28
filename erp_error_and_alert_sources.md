# ERP Error & Alert Sources — Map for Asana Ticket Automation

**Owner:** Viral · **Prepared for:** Arth (automated ticketing) · **Date:** 2026-10-28
**Source request:** Slack thread from Srishti, #C0A1Z6XQE7M, 2026-10-28

> **Purpose.** This document lists every place where ERPNext errors, failures and notifications are recorded today, so they can be polled and turned into Asana tasks.
>
> **About the numbers.** Every count below comes from a copy of the production database taken on 2026-10-27. Production is hosted at `lyfehardware.v.frappe.cloud` (Frappe Cloud).
>
> **Apps in scope:** frappe, erpnext, hrms, crm, lms, insights, payments, ecommerce_integrations, shopify_tax, frappe_s3_attachment and **lh**, our custom app. Almost all custom logic lives in `lh`.

---

## 0. TL;DR for the automation

Poll these **8 database tables** through the Frappe REST API (`/api/resource/<DocType>`). Between them they cover nearly everything:

| # | DocType | What it catches | Filter for "needs a ticket" | Retention ⚠️ | Suggested poll |
|---|---|---|---|---|---|
| 1 | **Error Log** | All Python exceptions plus every `frappe.log_error()` call (custom code, integrations, jobs) | `creation > last_poll` minus the noise list in §1.3 | **14 days** | 10 min |
| 2 | **Scheduled Job Log** | Scheduled (cron) jobs that crashed | `status = "Failed"` | **7 days** | 15 min |
| 3 | **Integration Request** | API calls to and from Shopify, ShipStation, 17Track, 1Click, CJ and others | `status = "Failed"` | **2 days** | 10 min |
| 4 | **Slack Notification Log** | Slack alerts that failed to send (and the alerts themselves) | `status in ("Failed","Blocked")` | none | 15 min |
| 5 | **Email Queue** | Outgoing emails that failed | `status = "Error"` (or `"Not Sent"` older than 1 h) | **10 days** | 30 min |
| 6 | **OneClick Shipment Issue** | Differences between 1Click shipped qty/SKU and the ERP | new records | none | 30 min |
| 7 | **SLA Escalation Log** | Task SLA breaches that were escalated | new records | none | 60 min |
| 8 | **Lyfe Order** (fields) | Per-order integration errors | `oneclick_error is set`; `tracking_status_desc` in exception states (§4.4) | n/a | 30 min |

**Retention is the biggest risk.** Integration Request rows are deleted after **2 days** and Scheduled Job Log rows after **7 days**. Because of the purge, the automation must poll more often than that and must copy the payload into the Asana task. A link back to the ERP record will not work for long.

**Deduplication key (suggested):** `doctype + normalized title + reference_name`. The `method` field of Error Log already holds a structured prefix such as `[module][function]: ...`.

---

## 1. Error Logs

### 1.1 Where errors are logged

| Location | What goes there | How to access | Usable by automation? |
|---|---|---|---|
| **Error Log** DocType | Unhandled exceptions in web requests, background jobs and scheduler jobs, plus every explicit `frappe.log_error(title, message)` in our code. `lh` has **~600 `log_error` calls**. | Desk: `/app/error-log` · API: `GET /api/resource/Error Log` | ✅ **Primary source** |
| **Scheduled Job Log** DocType | One row per scheduler run, with `status` = Complete/Failed/Start and `details` holding the traceback | Desk: `/app/scheduled-job-log` · API | ✅ |
| **RQ Job** (virtual DocType that reads Redis) | Background (RQ) jobs: queued, started, finished and **failed** | Desk: `/app/rq-job` (filter Status = failed) | ⚠️ Reads live from Redis, so verify that REST works on Frappe Cloud. The failures are also in Error Log. |
| **Frappe Cloud server logs** | `web.error.log`, `worker.error.log`, `scheduler.log`, `frappe.log`, plus app logs (`lh.log`, `lh_project.log`, `shopify.log`) | Frappe Cloud dashboard → Bench/Site → **Logs** tab | ❌ No API. Use only for manual investigation. |
| **Email Queue** | Outgoing emails, with `error` on failure | `/app/email-queue` · API | ✅ |

### 1.2 Error Log field reference (needed for parsing)

| Field | Content |
|---|---|
| `method` | **The title.** For our custom errors this is the `title=` passed to `log_error`, for example `[shopify][create_draft_order] LHQ-2026-0406 — HTTP 401`. |
| `error` | Full traceback or message body |
| `reference_doctype` / `reference_name` | The linked document, when the code passed one (not always populated) |
| `creation` | Timestamp |
| `seen` | 0/1. Set when someone opens the error in Desk. |

**Title conventions used in `lh`. Use these prefixes to route tickets:**

| Prefix in `method` | Area / suggested Asana section |
|---|---|
| `[shopify]...`, `Shopify ...`, `HTTPSConnectionPool(host='edaf9f-cb.myshopify.com'...` | Shopify |
| `[shipstation_settings]...`, `[shipstation_orders]...`, `ShipStation ...` | ShipStation |
| `Lyfe - 17Track Error — <tracking nos>` | 17Track |
| `[oneclick...]`, `[oneclick_inbound_email]...`, 1Click | 1Click (US warehouse 3PL) |
| `[lyfe_order]...`, `[material_issue_for_order]...` | Orders / Factory |
| `[reports][<name>] failed` | HR weekly reports |
| `[clarity_usage]...` | Microsoft Clarity |
| `[excel_export]...`, `[pdf_bom_stripper]...`, `upload_child_sku_images...` | Misc. / internal tools |
| `(1205, 'Lock wait timeout...')`, `(1406, 'Data too long...')`, `(1048, ...)` | DB errors (Engineering) |
| `frappe.core.doctype.scheduled_job_type.scheduled_job_type.run_scheduled_job` | Scheduler crash (also in Scheduled Job Log) |

### 1.3 Noise to filter out (these must NOT create tickets)

| `method` value | Count (14 days) | Why to ignore |
|---|---|---|
| `Session Stopped` | 86 | A user navigated away or the request timed out |
| `frappe.desk.doctype.notification_log.notification_log.make_notification_logs` | 49 | Frappe's in-app notification fan-out. Low value; review it in a weekly digest at most. |
| `[shipstation_orders][skip]: ...` | varies | **Intentionally informational.** Records why the ShipStation sync skipped an order. Keep for audit; no ticket. |
| `Unable to send new password notification` | 10 | User-management email noise |

### 1.4 Error Log volume snapshot (last 30 days, top titles)

Total rows currently held: **236**, the 14-day window.

| Title (`method`) | Count |
|---|---|
| Session Stopped | 86 |
| make_notification_logs | 49 |
| [excel_export][_add_picture]: image not loaded | 16 |
| scheduled_job_type.run_scheduled_job | 12 |
| Bulk action failed | 10 |
| Unable to send new password notification | 10 |
| (1406, "Data too long for column 'content_hash'") | 7 |
| Shopify HTTPS read timeout | 4 |
| [clarity_usage] network error | 4 |
| [shipstation_orders][_log_order_skip]: failed to log a skip reason | 3 |
| [material_issue_for_order][create_material_receipt_for_row] | 3 |
| Shopify Tax API error | 2 |
| Lock wait timeout exceeded | 2 |
| Other (one-off Shopify HTTP 401/409/422, ShipStation Mark-as-Shipped API error, refund recheck, 1Click packing list, …) | ~20 |

---

## 2. Workflow notifications

### 2.1 Active workflows

| Workflow | DocType | Built-in "Send Email Alert" |
|---|---|---|
| Lyfe Order | Lyfe Order | **Off** |
| Quotation-3 | Quotation | **Off** |
| Lyfe Order Return | Lyfe Order Return | **Off** |
| Lyfe Order Reshipment | Lyfe Order Reshipment | **Off** |

**No workflow uses Frappe's built-in workflow email or any Notification doc.** Every workflow-driven alert is **custom Python code** in `lh`:

| Workflow event | Alert | Channel | Destination |
|---|---|---|---|
| Quotation → *Pending Approval* | Approval card with Approve/Reject buttons | Slack bot | LH Slack Settings → `cs_tech_channel_id` |
| Quotation Slack button action rejected by ERP | Error reply in the card thread | Slack bot | same thread |
| Quotation workflow blocked by unpaid stage (`_enforce_stage_payment`) | Payment-block reminder, escalating to founders after N days | Email + Slack | support@lyfehardware.com (hardcoded) + Team Settings founders; Access Settings → `urgent_slack_channel` |
| Payment stage Upcoming → Due | Stage due notice | Email + Slack | same as above |
| Lyfe Order → Shipped / Completed | Stage payment notification | Email + Slack | same as above |
| Lyfe Order marked shipped in ShipStation | Success / failure post | Slack | ShipStation Settings → `success_slack_webhook_url` / `slack_webhook_url` |
| Lyfe Order reaches *US Warehouse Delivered* | Notice | Slack | LH Slack Settings (1Click channel) |

**Audit trail:** each Lyfe Order transition is recorded in the **Lyfe Order Workflow Log** child table (1,234 rows). This is history, not an alert source.

### 2.2 Notification DocType (Frappe "Notification" records)

Nine are enabled. **All are email and all are standard HR/LMS/Accounts.** None relates to orders or integrations.

| Notification | DocType | Event | Recipients |
|---|---|---|---|
| Customer Contact Access Request – Notify Approver | Customer Contact Access Request | New | System Manager |
| Material Request Receipt Notification | Material Request | status → Received / Partially Received | (owner) |
| Notification for new fiscal year | Fiscal Year | New (auto-created) | Accounts User / Manager |
| New job alert | Job Opportunity | New | System Manager |
| Exit Interview Scheduled | Exit Interview | Days Before | employee email |
| Retention Bonus | Retention Bonus | Days Before | — |
| Training Scheduled / Training Feedback | Training Event / Result | Submit | employees |
| Certificate Request Reminder | LMS Certificate Request | Days Before | evaluator, member |

A Quotation "Value Change" notification exists but is **disabled**.

---

## 3. Background job failures

### 3.1 Where failures are recorded

| Job type | Recorded in | Notes |
|---|---|---|
| **Scheduled jobs** (cron/hourly/daily from `lh/hooks.py`; ~60 in `lh`) | **Scheduled Job Log** (`status=Failed`, traceback in `details`) **and** **Error Log** | Scheduled Job Log is kept for 7 days only |
| **Enqueued jobs** (`frappe.enqueue`, e.g. post-sync verification, payment reminders, bulk actions) | **RQ Job** (failed registry in Redis) **and** **Error Log** | RQ failed jobs expire from Redis |
| Errors caught inside a job (the job "completes" but logs a problem) | **Error Log** only | Most integration jobs handle errors this way, so the job still shows *Complete*. **Error Log is the only reliable signal.** |

### 3.2 Current failure snapshot (Scheduled Job Log, last 14 days)

Totals: **27,030 complete / 148 failed.**

| Job | Failed runs |
|---|---|
| `lh_project.sla.scheduler.tar_escalation_scan.run` (hourly) | **146** ← failing almost every run; needs a fix independent of the ticketing work |
| `s3_backup_settings.take_backups_daily` | 1 |
| `email_account.pull` | 1 |

### 3.3 Existing alerting on job failures

**There is no generic alert.** A failed scheduled or background job alerts nobody unless that specific feature has its own handler:

| Job | Has its own failure alert? |
|---|---|
| ShipStation sync (`run_sync_from_settings`, every 15 min) | ✅ Email + Slack (`_notify_sync_failure`) |
| HR weekly reports (`lh_hrms.reports.*`) | ✅ Slack (`reports_failure_alert_webhook_url`) |
| 1Click automated steps | ✅ Hourly "stuck" sweep to Slack (`oneclick_stuck_alerts`) |
| Shopify refund capture | ✅ Hourly recheck to Slack |
| **Everything else**, including `tar_escalation_scan`, SLA scans, tracking crons, forecasting, attendance and the founder briefing | ❌ **None.** These are the main gap the ticketing system should close. |

---

## 4. Integration error capture

Integration calls are logged in **Integration Request**. ⚠️ **Retention is 2 days.** Use `integration_request_service` to identify the integration and `status = Failed` to find failures. Genuine failures are **also** written to Error Log.

### 4.1 Shopify (custom `lh` integration, not the stock ecommerce_integrations connector)

| Where | Detail |
|---|---|
| Integration Request services | `Shopify`, `Shopify Tax`, `Shopify Order Verification Check`, `Shopify Payment Link - Attempt n/5`, `Shopify Incomming Refund webhook` (sic) |
| Error Log | Titles `[shopify][<function>] <Quotation> — HTTP <code>`, `Shopify Tax API error`, `Shopify Webhook — ...`, HTTPS timeouts to `edaf9f-cb.myshopify.com` |
| Slack alerts | Payment link failed, payment on a cancelled quotation (Access Settings `urgent_slack_channel`); refund capture failed / still stuck (ShipStation Settings `success_slack_webhook_url`); draft tax mismatch (1Click webhook + email) |
| In-app | Draft order update failed → Notification Log alert to the Quotation owner |
| Ecommerce Integration Log (stock connector) | 416 "Error" rows, the latest from **May 2026**. The stock connector is no longer the active path. **Ignore.** |
| Failed now (snapshot) | 1 Shopify, 1 refund webhook, 5 payment-link attempts (one quotation), 2 Shopify Tax |

### 4.2 ShipStation

| Where | Detail |
|---|---|
| Integration Request services | `ShipStation`, `ShipStation Store Refresh` |
| Error Log | `[shipstation_settings][...]`, `[shipstation_orders][...]`, `ShipStation Mark as Shipped - API Error`; **skip records** `[shipstation_orders][skip]: <order> -- <reason>` are informational (see §1.3) |
| Email + Slack alerts | Order import failures, sync crash, API quota exceeded (403), monthly shipment counter threshold, orders in ShipStation missing from ERP (post-sync and hourly), Shopify orders with no Lyfe Order, order cancelled after posting to 1Click (also creates a Task) |
| Recipients | Email: Access Settings → **ShipStation Failure Alert Users**. Slack: ShipStation Settings `slack_webhook_url` (failures) **and** `success_slack_webhook_url`. ⚠️ Despite its name, the "success" channel also receives failure alerts. |
| In-app | When the sync changes items on a Lyfe Order: Notification Log alert to Factory users plus email |

### 4.3 17Track (shipment tracking)

| Where | Detail |
|---|---|
| Integration Request services | `17Track` (manual), `17Track-Scheduler` (cron), `17Track-Webhook` (inbound push) |
| Error Log | `Lyfe - 17Track Error — <tracking numbers>`. The body includes the Lyfe Order(s), URL and HTTP status. "No tracking info yet" responses are treated as transient and **not** logged as errors. |
| Business alerts (Slack + Email) | Order newly flagged as **Possible Lost, Delivery Exception, Stale Tracking or Carrier Never Scanned** (hourly tracking crons and a daily untracked-order scan). Seventeen Track Settings `tracking_alert_slack_webhook_url` and `tracking_alert_recipients`. |

### 4.4 Other integrations (for completeness)

| Integration | Where errors go |
|---|---|
| **1Click** (US warehouse 3PL) | Integration Request `1Click-CreateOrder`, `-CreateOrder-Retry`, `-CreateLoad`, `-AddItemMaster`, `-CheckInventory`, `-GetTracking`, `-GetInboundLoads`. Per-order error in **Lyfe Order.`oneclick_error`**. Qty/SKU differences create **OneClick Shipment Issue** records. All Slack alerts go to LH Slack Settings (consolidated alerts channel or `oneclick_slack_webhook_url`). |
| **CJ Logistics** (labels) | Integration Request `CJ Logistics` (has a "retry from log" action) |
| Paccurate / 3DBin / ShipEngine (box and rate calcs) | Integration Request plus Error Log |
| Gorgias | **Gorgias Integration Log** DocType (0 rows, inactive) |
| Claude / MCP | **MCP Audit Log** (6,928 rows; audit trail, not errors) |

Lyfe Order fields that show tracking state: `tracking_status_desc`, `tracking_status_us`, `reship_tracking_status`.

---

## 5. Custom alerts (scripts, server scripts, Notification docs)

### 5.1 Server Scripts / Client Scripts

| Name | Type | What it does | Alerts? |
|---|---|---|---|
| quotation | DocType Event · Quotation · Before Validate | Defaults the shipping address to the customer address | No |
| Sales invoice | DocType Event · Sales Invoice · Before Cancel | Unlinks Lyfe Order / Reshipment from a cancelled invoice | No |
| 4 Client Scripts | UI | UI behaviour only | No |

### 5.2 Custom alerts in `lh` Python code (complete list)

Legend: **E** = error/failure alert (a ticket candidate), **I** = informational/business notification. **SNL** = recorded in Slack Notification Log (§6).

#### Orders / ShipStation
| Alert | Trigger | Channel → Destination | Kind | SNL |
|---|---|---|---|---|
| Mark-as-shipped failure | ShipStation mark-shipped API/validation fails | Slack → ShipStation Settings `slack_webhook_url` | E | ✅ |
| Mark-as-shipped success | succeeds | Slack → `success_slack_webhook_url` | I | ✅ |
| Order import failures | ShipStation sync has errors | Email (Failure Users) + Slack | E | ✅ |
| Sync crash | 15-min ShipStation sync raises | Email + Slack ×2 | E | ✅ |
| API quota exceeded | ShipStation returns 403 | Email + Slack ×2 | E | ✅ |
| Shipment counter threshold | monthly count ≥ threshold (sent once) | Email + Slack ×2 | E (warn) | ✅ |
| ERP missing ShipStation orders | after sync + hourly check | Email + Slack | E | ✅ |
| Shopify orders with no Lyfe Order | after sync | Email + Slack | E | ✅ |
| Order cancelled after posting to 1Click | ShipStation sync (also creates a Task) | Slack → Access Settings `urgent_slack_channel` | E | ✅ |
| Order items changed by sync | ShipStation sync | In-app (Factory role) + Email (Factory & Support, hardcoded) | I | – |
| Urgent dispatch reminder | daily 09:00 | Slack → Access Settings `dispatch_reminder_slack_webhook` | I | ✅ |
| Missing Item Group digest | daily 10:30 / 15:30 | Email → Access Settings `ig_users` | I | – |
| Item Group fallback on Shopify item sync | on sync | Slack → 1Click webhook | E | ❌ |
| New custom orders digest | Mon–Fri 09:00 / 15:00 | Email → support@ | I | – |
| Customer contact displayed by Factory | Lyfe Order save | Email → Access Settings `contact_display_notify_users` | I | – |
| Factory comment routing | Comment on Quotation/Lyfe Order involving Engineer/CS | Slack bot → CS channels | I | ❌ |

#### 1Click (all go to LH Slack Settings; none are logged in SNL)
| Alert | Trigger | Kind |
|---|---|---|
| 1Click fulfilment error | create/resume order raises | E |
| Final Create Order / Load failure after retries | posting | E |
| US stock too low to post | before post | E |
| SKU not found → auto-registration | create_order rejected | E |
| Shipped qty/SKU differs (creates OneClick Shipment Issue) | hourly tracking sync | E |
| Stuck automated steps (not routed, shipped not posted, etc.) | hourly sweep | E |
| US-warehouse load not confirmed in 1Click | daily | E |
| Reshipment create/load failures, no customer tracking | hourly/daily | E |
| Duplicate load attempted | on post | I |
| Inbound / dispatch email to the 3PL (Slack alert if the email fails) | batch / delivery | I / E |

#### Shopify
| Alert | Trigger | Channel → Destination | Kind | SNL |
|---|---|---|---|---|
| Payment link failed | draft order create raises | Slack → `urgent_slack_channel` | E | ✅ |
| Payment received on cancelled quotation | payment/webhook | Slack → `urgent_slack_channel` | E | ✅ |
| Draft order update failed | update raises | In-app → Quotation owner | E | – |
| Tax mismatch on draft | after create/update | Slack (1Click webhook) + Email (Failure Users) | E | ❌ |
| Refund capture failed / still stuck | webhook retries exhausted; hourly recheck | Slack → ShipStation `success_slack_webhook_url` | E | ✅ |
| Bank transfer chosen | customer action | Email → support@ and the customer | I | – |

#### 17Track
| Alert | Trigger | Channel | Kind | SNL |
|---|---|---|---|---|
| Possible Lost / Delivery Exception / Stale / Never Scanned | hourly tracking crons + daily scan | Slack + Email (Seventeen Track Settings) | E | ✅ |

#### Quotation
| Alert | Trigger | Channel | Kind |
|---|---|---|---|
| Approval card | → Pending Approval | Slack bot | I |
| Drawing file replaced | Quotation update | Email → support@chandakbrothers.net (hardcoded) | I |
| Timer stopped on drawing / quotation cancel | Quotation update/cancel | Email → timesheet owner | I |

#### COGS
| Alert | Trigger | Channel | Kind |
|---|---|---|---|
| COGS deviation | Lyfe Order / Quotation update | Email → Access Settings `cogs_alert_emails` | E (warn) |
| COGS digest | daily 07:00 | Email (same) | I |

#### Forecast / Founder
| Alert | Trigger | Channel | Kind |
|---|---|---|---|
| Stock risk worsened to High/Critical | nightly snapshot | Slack → Founder Briefing Settings | I |
| Daily founder briefing | hourly, self-gated | Slack (same) | I |

#### HR (LH HRMS Settings)
| Alert | Trigger | Channel | Kind |
|---|---|---|---|
| Missed clock-in / clock-out | every 15 min | Email → employee | I |
| Founder attendance digest | daily (IST hour gate) | Slack | I |
| Hours reconciliation flags / resolutions | 02:30 daily; Daily Work Log update | Slack + In-app → manager | I |
| Birthdays | daily | Slack | I |
| Daily Work Log posts | DWL submit / clock-out | Slack bot or webhook | I |
| Weekly reports (timesheet, SLA health, hiring, reconciliation, learning, points) | weekly, self-gated | Email / Slack | I |
| **Weekly report failed** | any report raises | Slack → `reports_failure_alert_webhook_url` | **E** |

#### SLA / Task automation
| Alert | Trigger | Channel | Kind |
|---|---|---|---|
| New SLA violation task | 15-min SLA scan, when SLA Rule `notify_on_slack` is on | Slack bot → consolidated alerts channel | E |
| SLA escalation to manager | hourly escalation scan | In-app toast + Email; record in **SLA Escalation Log** | E |
| Task auto-created by rule | doc events | In-app toast | I |

---

## 6. Slack Notification Log: how it works and its gaps

- Every Slack **webhook** post made through `lh_hrms/notifications/slack.py: send_slack_message` writes one row.
- `source` holds the calling function's path, for example `lh.lyfe_hardware.tasks.refund_capture_recheck._notify_slack_refund_still_stuck`.
- `status` is one of three values:
  - `Success`
  - `Failed`, with the error text in `error`
  - `Blocked`, meaning the URL is invalid, or developer mode is on and Slack is disabled
- **Gaps.** The following are **not** logged:
  - Bot posts (`chat.postMessage`): Quotation approval, SLA, factory routing
  - Every 1Click alert
  - The Shopify tax-mismatch and Item Group alerts
  - Any alert skipped because its webhook URL is empty. These fail silently with no row.
- **Conclusion:** use SNL only to detect **failed Slack deliveries**. It is not a complete list of alerts sent.
- Snapshot (all `Success`):
  - mark-as-shipped success: 234
  - refund still stuck: 188
  - tracking alerts: 93
  - refund capture failed: 23
  - payment reminders: 3
  - cancelled-after-1Click: 1
  - cancelled quotation payment: 1

---

## 7. Recommendations for the Asana automation

1. **Primary feed:**
   - Error Log, minus the noise list
   - Scheduled Job Log (Failed)
   - Integration Request (Failed)

   These three cover real errors. Poll every 10–15 minutes **because of the 2-day and 7-day purges**.
2. **Group, don't spam.** Normalise the title by removing order, quotation and tracking numbers, then create **one Asana task per normalised title per day**. Add repeats as comments or a counter. Example: 146 `tar_escalation_scan` failures should be one task.
3. **Route by title prefix** (§1.2): Shopify / ShipStation / 17Track / 1Click / Orders / HR / Engineering.
4. **Copy the payload** (title, traceback, reference doc, timestamp) into the task body. The ERP record will be purged.
5. **Use the Slack alerts as context only.** They are already human-facing. Ticket them only if a channel is unmonitored, or as a later phase.
6. **Fix first:** `tar_escalation_scan` is failing about every hour (146 failures in 14 days).
7. **API access:** create a dedicated ERP API user with read-only roles on the DocTypes above, then use token auth with `GET /api/resource/<DocType>?filters=...&fields=...`.

### Open questions for Arth / Srishti
- Should the automation tick `seen = 1` on Error Log rows it has ticketed? That requires write permission.
- Asana project, sections and assignee mapping per area (Shopify, ShipStation, 1Click, HR, Engineering)?
- Severity rules, for example: payment and shipping failures = High, report or digest failures = Low?
- Should retention be raised (Integration Request 2 → 14 days, Scheduled Job Log 7 → 14 days) as a safety margin? It is changed in Log Settings.
