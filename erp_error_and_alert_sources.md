# Where Our ERP Records Errors and Alerts

**For:** Arth, who is building the automation that turns ERP errors into Asana tasks
**From:** Viral · **Date:** 28 Oct 2026
**Request:** Srishti's Slack thread of 28 Oct 2026

---

## What this document is

Our ERP (ERPNext, hosted at `lyfehardware.v.frappe.cloud`) records problems in several places. Some go to internal log tables, some go to Slack, some go by email.

This document lists **every place a problem can show up**, explains what each one means, and says which ones the Asana automation should read.

You don't need to know the LH app to use it. Section 1 explains the terms. Section 2 gives the short answer. The rest is detail.

> All numbers come from a copy of the live database taken on 27 Oct 2026.

---

## 1. Terms you will see

### 1.1 Systems we connect to

| Term | Plain meaning |
|---|---|
| **ERPNext / Frappe** | Our ERP system. Frappe is the framework it is built on. |
| **LH app** | Our own custom code added on top of ERPNext. Almost every custom error comes from here. |
| **DocType** | A table in the ERP, such as "Quotation" or "Error Log". Each row is called a document. |
| **Shopify** | Our online store. Customers pay their quotations through Shopify payment links. |
| **ShipStation** | The shipping platform. Orders from Shopify go here, and the ERP pulls them in every 15 minutes. |
| **17Track** | A tracking service. The ERP asks it hourly where each shipment is. |
| **1Click** | Our US warehouse partner (3PL). Orders that ship from the US warehouse are sent to 1Click through their API. |

### 1.2 Our main documents

| Term | Plain meaning |
|---|---|
| **Quotation** (number like `LHQ-2026-0247`) | A price quote sent to a customer. When the customer pays, it becomes "Won". |
| **Lyfe Order** (number like `LYF-SH-2026-1874`) | Our main order record, one per customer order. It follows the order from factory to delivery. |

### 1.3 Jobs, alerts and logs

| Term | Plain meaning |
|---|---|
| **Scheduled job** | Code that runs automatically on a timer, like a cron job. For example: "sync ShipStation every 15 minutes". |
| **Background job** | Code that runs in a queue after a user action, so the screen doesn't freeze. |
| **Webhook (Slack)** | A URL that posts a message into one Slack channel. Most of our alerts use these. |
| **Slack bot** | Our Slack app. It posts into a channel by channel ID instead of a webhook URL. |
| **Retention** | How long the ERP keeps a log before deleting it automatically. |

---

## 2. The short answer

The automation only needs to read **3 tables**, through the ERP's REST API. Together they catch nearly every real problem:

| # | Table (DocType) | What a row means | Rows that should become tickets | Deleted after ⚠️ | Check every |
|---|---|---|---|---|---|
| 1 | **Error Log** | Something in the ERP failed: a crash, or our code recorded a problem | All new rows, except the noise in §4 | **14 days** | 10 min |
| 2 | **Scheduled Job Log** | One run of a timed job | `status = "Failed"` | **7 days** | 15 min |
| 3 | **Integration Request** | One call to or from Shopify, ShipStation, 17Track or 1Click | `status = "Failed"` | **2 days** | 10 min |

Five more tables are useful as a second phase:

| # | Table | What it catches | Rows that should become tickets | Deleted after | Check every |
|---|---|---|---|---|---|
| 4 | **Slack Notification Log** | Slack alerts that could not be sent | `status` is `"Failed"` or `"Blocked"` | never | 15 min |
| 5 | **Email Queue** | Emails that could not be sent | `status = "Error"` | 10 days | 30 min |
| 6 | **OneClick Shipment Issue** | The US warehouse shipped something different from what we sent | every new row | never | 30 min |
| 7 | **SLA Escalation Log** | A task missed its deadline and was escalated to a manager | every new row | never | 60 min |
| 8 | **Lyfe Order** (two fields) | An order with a US-warehouse error or a tracking problem | `oneclick_error` is not empty, or `tracking_status_desc` shows an exception | n/a | 30 min |

### ⚠️ The most important thing

The ERP **deletes logs automatically**:
- Integration Request after **2 days**
- Scheduled Job Log after **7 days**
- Error Log after **14 days**

Because of this:
1. The automation must check more often than that.
2. Each Asana task must **contain a copy of the error** (title, details, time, linked document). A link back to the ERP will stop working once the log is deleted.

### How to read a table (example)

```
GET https://lyfehardware.v.frappe.cloud/api/resource/Error Log
    ?fields=["name","method","error","reference_doctype","reference_name","creation"]
    &filters=[["creation",">","2026-10-28 10:00:00"]]
    &order_by=creation asc
Authorization: token <api_key>:<api_secret>
```

Use a dedicated ERP user with **read-only** access to these tables for the API key.

---

## 3. Error Log (the main source)

### 3.1 What goes into it

Every time something fails in the ERP, a row is added. This happens in two ways:
- **Unexpected crashes**, for example a bug or a database timeout.
- **Problems our code reports on purpose**, for example "Shopify returned HTTP 401". The LH app does this in about 600 places.

You can view it in the ERP at **/app/error-log**.

### 3.2 The fields you need

| Field | What it holds |
|---|---|
| `method` | **The title of the error.** Despite the name, this is the field to read first. |
| `error` | The full details (message or technical trace) |
| `reference_doctype` + `reference_name` | The document it relates to, for example *Lyfe Order LYF-SH-2026-2101*. Often empty. |
| `creation` | When it happened |

### 3.3 Sorting errors by area (for the Asana section or assignee)

Our custom errors start with a tag in square brackets, which tells you the area:

| The title starts with or contains | Area |
|---|---|
| `[shopify]`, `Shopify ...`, or mentions `myshopify.com` | **Shopify / payments** |
| `[shipstation_settings]`, `[shipstation_orders]`, `ShipStation ...` | **ShipStation / order sync** |
| `Lyfe - 17Track Error` | **Shipment tracking** |
| `[oneclick...]`, `1Click` | **US warehouse (1Click)** |
| `[lyfe_order]`, `[material_issue_for_order]` | **Orders / factory** |
| `[reports]` | **HR reports** |
| A number in brackets, like `(1205, ...)` or `(1406, ...)` | **Database error → Engineering** |
| Anything else | **Engineering** (to be sorted by hand) |

---

## 4. Noise: errors that should NOT become tickets

These show up often but are harmless or temporary. Filter them out:

| Title contains | Why to ignore it |
|---|---|
| `Session Stopped` | A user closed the page or the request was cut off. About 86 every 2 weeks. |
| `make_notification_logs` | A harmless timing glitch inside the ERP's own "you were mentioned" notifications. Users still get notified. |
| `[shipstation_orders][skip]` | Not an error. It records on purpose why an order was skipped during sync. |
| `Read timed out`, `504 Gateway Time-out`, `network error` | Shopify, ShipStation or Clarity was slow for a moment. Ticket it only if the same one repeats many times in a day. |
| `Scheduled jobs can't run in maintenance mode` | Happens during every deployment |
| `Failed to aquire ... bench_migrate.lock` | Happens during deployment |

**Grouping tip:** remove order, quotation and tracking numbers from the title, then make **one Asana task per title per day**. Add repeats as a comment or a counter. A job that fails every hour should be one task, not 24.

---

## 5. Timed and background jobs

### 5.1 Where failures go

| Kind of job | Where a failure is recorded |
|---|---|
| **Scheduled job** (runs on a timer; the LH app has about 60) | **Scheduled Job Log** with `status = Failed`, plus the Error Log |
| **Background job** (queued after a user action) | **Error Log**. The ERP's "RQ Job" screen (/app/rq-job) also shows failed jobs, but only for a short time and not reliably through the API. |
| A job that **finished but hit a problem inside** | **Error Log only**. The job still shows as "Complete". Most of our integration jobs work like this, which is why the Error Log matters most. |

### 5.2 Current state

In the last 14 days, scheduled jobs ran about 27,000 times. Only 2 runs failed:

| Job | Failures |
|---|---|
| Daily S3 backup | 1 |
| Email inbox fetch | 1 |

(Another job failed 146 times in this window. It was fixed on 28 Oct and is not an open issue.)

### 5.3 Does anyone get alerted today?

**Mostly no.** A failed job alerts nobody unless that feature has its own alert built in. Only these do:

| Job | Has its own alert? |
|---|---|
| ShipStation order sync | ✅ Email + Slack |
| Weekly HR reports | ✅ Slack |
| US warehouse (1Click) steps | ✅ Hourly Slack check for anything stuck |
| Shopify refund confirmation | ✅ Hourly Slack re-check |
| **Everything else**: deadline (SLA) checks, tracking updates, stock forecasting, attendance, founder briefing | ❌ **No alert.** This is the main gap the ticketing system will close. |

---

## 6. Integrations

Every call to an outside system is recorded in **Integration Request**.
- The `integration_request_service` field tells you which system.
- `status = "Failed"` means the call failed.
- **Kept for 2 days only.** Real failures are also copied to the Error Log, which is kept for 14 days.

| System | Names in `integration_request_service` | Other places it reports problems |
|---|---|---|
| **Shopify** | `Shopify`, `Shopify Tax`, `Shopify Order Verification Check`, `Shopify Payment Link - Attempt 1/5` … `5/5`, `Shopify Incomming Refund webhook` (spelled this way) | Error Log titles starting `[shopify]` or `Shopify ...`. Slack alerts are listed in §8. |
| **ShipStation** | `ShipStation`, `ShipStation Store Refresh` | Error Log `[shipstation_...]`. Email + Slack for sync problems (§8). |
| **17Track** | `17Track`, `17Track-Scheduler`, `17Track-Webhook` | Error Log `Lyfe - 17Track Error — <tracking numbers>`. "No tracking info yet" is normal and is not logged as an error. |
| **1Click (US warehouse)** | `1Click-CreateOrder`, `1Click-CreateOrder-Retry`, `1Click-CreateLoad`, `1Click-AddItemMaster`, `1Click-CheckInventory`, `1Click-GetTracking`, `1Click-GetInboundLoads` | The `oneclick_error` field on the Lyfe Order. **OneClick Shipment Issue** table for shipped-quantity differences. |
| **CJ Logistics** (shipping labels) | `CJ Logistics` | Error Log |
| Box and rate calculators (Paccurate, 3DBin, ShipEngine) | their own names | Error Log |

**Ignore:** the **Ecommerce Integration Log** table. It belongs to an old Shopify connector we no longer use; its last entry is from May 2026.

---

## 7. Workflow notifications

A "workflow" is the series of steps a document moves through, for example a Quotation going from *Draft* to *Pending Approval* to *Won*.

We have 4 active workflows: **Lyfe Order**, **Quotation**, **Lyfe Order Return** and **Lyfe Order Reshipment**.

**None of them uses the ERP's built-in workflow emails.** All workflow alerts come from our own code:

| When this happens | Who is told | How |
|---|---|---|
| A Quotation is sent for approval | Approvers | Slack card with Approve / Send Back buttons |
| An order is blocked because a payment stage is unpaid | Customer support, then the founders if it stays unpaid | Email + Slack, daily reminder |
| A payment stage becomes due | Customer support | Email + Slack |
| An order is shipped or completed without its stage payment | Customer support | Email + Slack |
| An order is marked shipped in ShipStation | Shipping team | Slack (one channel for success, one for failure) |
| An order arrives at the US warehouse | US-warehouse team | Slack |

**ERP "Notification" records:** 9 are switched on. All are standard HR, training and accounting emails (training scheduled, exit interview, new fiscal year and similar). None relates to orders or integrations.

**Server Scripts:** 2 exist. Neither sends alerts. One fills in a default shipping address; the other unlinks orders from a cancelled invoice.

---

## 8. All custom alerts (Slack and email)

**How to read the tables:**
- **Problem?** ✅ means the alert reports something broken (a ticket candidate). ➖ means it is a normal business update.
- **In Slack log?** means whether the alert appears in the Slack Notification Log (§9).

### Orders and ShipStation

| Alert | When it fires | Sent to | Problem? | In Slack log? |
|---|---|---|---|---|
| Couldn't mark order shipped in ShipStation | Push to ShipStation fails | Slack (ShipStation failures channel) | ✅ | ✅ |
| Order marked shipped in ShipStation | Push succeeds | Slack (ShipStation success channel) | ➖ | ✅ |
| Orders failed to import | After a sync with errors | Email + Slack | ✅ | ✅ |
| ShipStation sync crashed | Sync job crashes | Email + Slack | ✅ | ✅ |
| ShipStation API limit reached | ShipStation refuses requests | Email + Slack | ✅ | ✅ |
| Monthly shipment limit nearly reached | Counter passes threshold | Email + Slack | ✅ (warning) | ✅ |
| Orders in ShipStation missing from the ERP | After sync and hourly | Email + Slack | ✅ | ✅ |
| Shopify orders with no ERP order | After sync | Email + Slack | ✅ | ✅ |
| Order cancelled after it was sent to the US warehouse | Sync sees the cancellation | Slack (urgent orders channel); also creates a task | ✅ | ✅ |
| Order items changed by sync | Sync changes items | ERP notification + email to factory and support | ➖ | – |
| Urgent order close to its dispatch deadline | Daily 9 AM | Slack | ➖ | ✅ |
| Items missing a product group | Twice daily | Email | ➖ | – |
| Product group not found during Shopify item sync | On sync | Slack (US-warehouse channel) | ✅ | ❌ |
| New custom orders summary | Weekdays 9 AM and 3 PM | Email to support | ➖ | – |
| Factory viewed customer contact details | On order save | Email | ➖ | – |
| Comment that tags customer support or an engineer | On comment | Slack bot (CS channels) | ➖ | ❌ |

### US warehouse (1Click)

All of these go to the US-warehouse Slack channel. **None** appear in the Slack Notification Log.

| Alert | When it fires | Problem? |
|---|---|---|
| Order failed to send to 1Click | On error | ✅ |
| Order or shipment still failing after all retries | After retries | ✅ |
| Not enough US stock to send the order | Before sending | ✅ |
| Product code unknown to 1Click (auto-registered, needs a manual retry) | On send | ✅ |
| 1Click shipped a different quantity or product | Hourly check | ✅ |
| An automatic step didn't run (stuck order) | Hourly check | ✅ |
| Delivery to the US warehouse not confirmed by 1Click | Daily check | ✅ |
| Reshipment failed to send, or has no tracking | Hourly / daily | ✅ |
| Shipment already created (duplicate attempt) | On send | ➖ |
| Packing email to the warehouse failed | On send | ✅ |

### Shopify and payments

| Alert | When it fires | Sent to | Problem? | In Slack log? |
|---|---|---|---|---|
| Payment link could not be created | "Generate Payment Link" fails | Slack (urgent orders channel) | ✅ | ✅ |
| **Customer paid against a cancelled quotation** | Shopify reports a payment for a cancelled quotation | Slack bot → **CS Quotation channel** (changed 28 Oct) | ✅ | ❌ |
| Payment draft could not be updated | Update fails | ERP notification to the quotation owner | ✅ | – |
| Shopify refund could not be confirmed | Retries run out; re-checked hourly | Slack (ShipStation success channel) | ✅ | ✅ |
| Customer chose bank transfer | Customer action | Email to support and the customer | ➖ | – |

### Shipment tracking (17Track)

| Alert | When it fires | Sent to | Problem? | In Slack log? |
|---|---|---|---|---|
| Package possibly lost, delivery exception, no updates, or never scanned | Hourly tracking check + daily scan | Slack + email | ✅ | ✅ |

### Quotations

| Alert | When it fires | Sent to | Problem? |
|---|---|---|---|
| Quotation approval card | Quotation sent for approval | Slack bot (CS Tech channel) | ➖ |
| Approval clicked in Slack was rejected by the ERP | On button click | Reply in the Slack thread | ✅ |
| Drawing file replaced | On quotation update | Email to the factory | ➖ |
| Work timer stopped | Drawing change or quotation cancel | Email to the person timing it | ➖ |

### Costs, stock and founder reports

| Alert | When it fires | Sent to | Problem? |
|---|---|---|---|
| Product cost far from its benchmark | On order or quotation update | Email | ✅ (warning) |
| Daily cost summary | Daily 7 AM | Email | ➖ |
| Item stock risk became High or Critical | Nightly | Slack (founder channel) | ➖ |
| Daily founder briefing | Once a day | Slack (founder channel) | ➖ |

### HR

| Alert | When it fires | Sent to | Problem? |
|---|---|---|---|
| Missed clock-in or clock-out | Every 15 min | Email to the employee | ➖ |
| Daily attendance summary | Daily | Slack (founder) | ➖ |
| Logged hours don't match tracked hours | Nightly | Slack + ERP notification to the manager | ➖ |
| Birthdays | Daily | Slack | ➖ |
| Daily work log posts | On clock-in and clock-out | Slack | ➖ |
| Weekly HR reports | Weekly | Email or Slack | ➖ |
| **A weekly HR report failed to send** | When it fails | Slack (report-failure channel) | ✅ |

### Task deadlines (SLA)

| Alert | When it fires | Sent to | Problem? |
|---|---|---|---|
| Task created because a deadline was missed | Every 15 min | Slack bot (alerts channel) | ✅ |
| Overdue task escalated to a manager | Hourly | ERP pop-up + email; also recorded in the **SLA Escalation Log** | ✅ |
| Task created automatically by a rule | On document events | ERP pop-up | ➖ |

---

## 9. Slack Notification Log

This table records Slack messages sent through **webhooks**:
- `status = "Success"`: sent.
- `status = "Failed"`: Slack rejected it. The reason is in `error`.
- `status = "Blocked"`: not sent because the URL is wrong, or because this is a test site.

**It does not record everything.** These are missing:
- Messages sent by the **Slack bot**: quotation approval cards, deadline (SLA) alerts, comment routing, and the cancelled-quotation payment alert.
- All **US warehouse (1Click)** alerts.
- The product-group alert.
- Any alert that was skipped because its Slack channel is not set up. These fail silently with no row.

**Use it only to catch Slack messages that failed to send.** It is not a full list of alerts.

---

## 10. Recommendations for the Asana automation

1. **Start with the 3 main tables** (§2): Error Log, Scheduled Job Log (`Failed`) and Integration Request (`Failed`). Check every 10–15 minutes, because the logs are deleted quickly.
2. **Filter the noise** listed in §4.
3. **Group repeats.** One Asana task per error title per day; add further repeats as comments.
4. **Assign by area** using the title tags in §3.3.
5. **Copy the error details into the task.** Don't rely only on a link to the ERP, since the log will be deleted.
6. **Leave Slack alerts out of phase 1.** People already read them in Slack. Add them later only if a channel is not being watched.
7. **Use a read-only ERP API user** for the automation.
