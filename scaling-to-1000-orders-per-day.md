# Scaling to 1,000 Orders/Day — Technical Plan

> Environment: Frappe Cloud (Production)
> App: Lyfe Hardware (`lh`)
> Active Integrations: ShipStation, Shopify, CJ, 17Track, S3

---

## Overview

1,000 orders/day = ~42 orders/hour average, with peak potentially hitting **150–300/hour** in a 4-hour window. This document covers architecture, query optimization, failure handling, and a testing plan to reach this scale reliably.

---

## 1. Load Handling Strategy

### Frappe Cloud Configuration

- **Upgrade plan** to at least **4 vCPU / 8 GB RAM** to handle concurrent workers, DB load, and integration calls.
- **Worker configuration** (minimum recommended):

| Worker Type | Count | Purpose |
|---|---|---|
| `default` | 2 | Order creation, webhooks, standard jobs |
| `short` | 2 | Quick lookups, status updates |
| `long` | 1 | Batch syncs, large reports |

### Queue All Webhook Receivers

All incoming webhooks (Shopify, ShipStation) must enqueue work and return `200 OK` immediately. Never do heavy DB work or external API calls synchronously in a webhook handler.

```python
@frappe.whitelist(allow_guest=True)
def shopify_webhook():
    payload = frappe.request.get_json()
    frappe.enqueue(
        "lh.lyfe_hardware.integrations.shopify.process_order",
        queue="default",
        payload=payload,
        job_name=f"shopify_order_{payload.get('id')}"
    )
    return {"status": "ok"}
```

---

## 2. Database Index Additions

Add a migration patch to create indexes on high-traffic lookup fields.

**File:** `lh/patches/v1/add_performance_indexes.py`

```python
def execute():
    # SLA module
    frappe.db.add_index("SLA Violation Cache", ["is_resolved", "sla_rule"])
    frappe.db.add_index("SLA Task Link", ["erp_docname", "status"])

    # Lyfe Order — general filtering
    frappe.db.add_index("Lyfe Order", ["status", "warehouse"])

    # Lyfe Order — user search fields
    frappe.db.add_index("Lyfe Order", ["internal_order_id"])
    frappe.db.add_index("Lyfe Order", ["source_order_id"])

    # ShipStation
    frappe.db.add_index("ShipStation Orders", ["order_status", "modify_date"])
```

> `internal_order_id` and `source_order_id` get separate single-column indexes because users search them independently, not together.

**Register the patch** in `patches.txt`:
```
lh.patches.v1.add_performance_indexes
```

---

## 3. API Call Optimization

### Frappe Query Best Practices

```python
# Single value lookup — hits SQL cache
status = frappe.db.get_value("Lyfe Order", order_id, "status")

# Bulk read — always specify fields and filters
orders = frappe.db.get_all(
    "Lyfe Order",
    filters={"status": "Ready for Dispatch", "warehouse": warehouse},
    fields=["name", "tracking_number", "carrier"],
    limit=500
)

# Complex joins — use raw SQL instead of chaining get_all calls
results = frappe.db.sql("""
    SELECT lo.name, lo.status, c.carrier_code
    FROM `tabLyfe Order` lo
    JOIN `tabCarrier` c ON c.name = lo.carrier
    WHERE lo.status = 'Ready for Dispatch'
""", as_dict=True)
```

### Redis Caching for External APIs

Cache responses from ShipStation, CJ, and 17Track to avoid redundant calls:

```python
def get_shipstation_carriers():
    cache_key = "shipstation_carriers"
    cached = frappe.cache().get_value(cache_key)
    if cached:
        return cached
    data = call_shipstation_api("/carriers")
    frappe.cache().set_value(cache_key, data, expires_in_sec=3600)
    return data
```

### Retry with Exponential Backoff

Wrap all external API calls to handle rate limits gracefully:

```python
import time

def call_with_retry(fn, max_retries=3):
    for attempt in range(max_retries):
        try:
            return fn()
        except RateLimitError:
            time.sleep(2 ** attempt)
    frappe.throw("External API unavailable after retries")
```

### ShipStation Sync

- Always pass `modifyDateStart` using the last sync timestamp (already stored in `ShipStation Settings`)
- Process in batches of 100 (ShipStation's page size)
- Enqueue each page as a separate job rather than one long-running task

---

## 4. High-Risk Areas & Mitigations

| Area | Risk | Fix |
|---|---|---|
| `on_update` on Lyfe Order (3 hooks fire on every save) | Every save triggers SLA check + automation | Add guard: `if doc.has_value_changed("status"):` |
| ShipStation 30-min batch sync | Can overlap with itself under load | Add Redis mutex lock before starting sync |
| Shopify token rotation (every 45 min) | Race condition if two workers rotate simultaneously | Atomic Redis lock before rotation |
| SLA scan (every 15 min) | Table scans on growing dataset | Covered by the index patch above |
| `delete_versions_batch` (every 30 min) | DB lock contention during peak | Run in `long` queue, enforce `LIMIT 500` per batch |

### `on_update` Guard Pattern

```python
def on_update(self):
    if not self.has_value_changed("status"):
        return
    # ... rest of hook logic
```

### ShipStation Sync Mutex

```python
def run_sync_from_settings():
    lock_key = "ss_sync_lock"
    if frappe.cache().get_value(lock_key):
        return  # already running
    frappe.cache().set_value(lock_key, True, expires_in_sec=1800)
    try:
        # ... sync logic
    finally:
        frappe.cache().delete_value(lock_key)
```

---

## 5. Failure Handling & Alerts

Every critical failure sends an alert to **both Slack and developer email simultaneously**.

### Config Fields (add to ShipStation Settings or Access Settings)

| Field Name | Type | Label |
|---|---|---|
| `slack_webhook_url` | Password | Slack Webhook URL for Developer Alerts |
| `developer_alert_email` | Data | Developer Alert Email |

Both fire on every failure — no separate calls needed.

---

### Core Notifier Utility

**File:** `lh/lyfe_hardware/utils/alerts.py`

```python
import frappe
import requests
import json

def notify_developer(title, message, doctype=None, docname=None, extra=None):
    """Send instant alert to developer via Slack + email on any critical failure."""

    # Always log to Frappe Error Log
    frappe.log_error(message, title)

    # Build doc link
    site_url = frappe.utils.get_url()
    doc_link = ""
    if doctype and docname:
        doc_link = f"\n🔗 {site_url}/app/{doctype.lower().replace(' ', '-')}/{docname}"

    extra_info = ""
    if extra:
        extra_info = "\n```" + json.dumps(extra, indent=2, default=str) + "```"

    # Send Slack alert
    _send_slack(f":red_circle: *{title}*\n{message}{doc_link}{extra_info}")

    # Send email alert
    _send_email(title, message, doctype, docname, extra)


def _send_slack(text):
    try:
        webhook_url = frappe.db.get_single_value("ShipStation Settings", "slack_webhook_url")
        if webhook_url:
            requests.post(webhook_url, json={"text": text}, timeout=5)
    except Exception:
        pass  # Never let alert failure crash the main flow


def _send_email(title, message, doctype, docname, extra):
    try:
        developer_email = frappe.db.get_single_value("ShipStation Settings", "developer_alert_email")
        if not developer_email:
            return
        site_url = frappe.utils.get_url()
        body = f"<b>{title}</b><br><br>{message}"
        if doctype and docname:
            body += f'<br><br><a href="{site_url}/app/{doctype.lower().replace(" ", "-")}/{docname}">Open {doctype}: {docname}</a>'
        if extra:
            body += f"<br><br><pre>{json.dumps(extra, indent=2, default=str)}</pre>"
        frappe.sendmail(
            recipients=[developer_email],
            subject=f"[LH Alert] {title}",
            message=body,
            now=True  # send immediately, not via queue
        )
    except Exception:
        pass
```

---

### Wire-In: Shopify Order Creation

```python
# lh/lyfe_hardware/integrations/shopify.py
from lh.lyfe_hardware.utils.alerts import notify_developer

def process_order(payload):
    shopify_order_id = payload.get("id")
    try:
        # ... order creation logic
    except Exception as e:
        notify_developer(
            title="Shopify Order Creation Failed",
            message=str(e),
            doctype="Shopify Order",
            docname=str(shopify_order_id),
            extra={"shopify_order_id": shopify_order_id}
        )
        raise
```

### Wire-In: ShipStation → Lyfe Order Creation

```python
# ShipStation Orders controller — create_lyfe_order()
from lh.lyfe_hardware.utils.alerts import notify_developer

def create_lyfe_order(self):
    try:
        # ... existing logic
    except Exception as e:
        notify_developer(
            title="ShipStation → Lyfe Order Creation Failed",
            message=str(e),
            doctype="ShipStation Orders",
            docname=self.name,
            extra={"shipstation_order_id": self.shipstation_order_id}
        )
        raise
```

### Wire-In: Gate Pass Submission

```python
# GatePass controller — before_submit()
from lh.lyfe_hardware.utils.alerts import notify_developer

def before_submit(self):
    try:
        self.validate_lyfe_orders()
        self.validate_total_weight()
        self.validate_total_boxes()
    except Exception as e:
        notify_developer(
            title="Gate Pass Submission Failed",
            message=str(e),
            doctype="Gate Pass",
            docname=self.name,
            extra={"total_items": self.total_items, "date": str(self.date)}
        )
        raise
```

### Wire-In: ShipStation Sync (Background Job)

```python
# lh/lyfe_hardware/doctype/shipstation_settings/shipstation_settings.py
from lh.lyfe_hardware.utils.alerts import notify_developer

def run_sync_from_settings():
    try:
        # ... sync logic
    except Exception as e:
        notify_developer(
            title="ShipStation Sync Failed",
            message=str(e),
            extra={"job": "run_sync_from_settings"}
        )
        raise
```

### Wire-In: SLA Scan (Background Job)

```python
# lh/lh_project/sla/scheduler/sla_scan.py
from lh.lyfe_hardware.utils.alerts import notify_developer

def run():
    try:
        # ... scan logic
    except Exception as e:
        notify_developer(
            title="SLA Scan Failed",
            message=str(e),
            extra={"job": "sla_scan"}
        )
        raise
```

---

### Coverage Summary

| Failure Point | Slack | Email | Error Log |
|---|---|---|---|
| Shopify → Order creation | ✅ | ✅ | ✅ |
| ShipStation → Lyfe Order creation | ✅ | ✅ | ✅ |
| Gate Pass submission | ✅ | ✅ | ✅ |
| ShipStation sync job | ✅ | ✅ | ✅ |
| SLA scan job | ✅ | ✅ | ✅ |

---

### Sample Alert (Slack)

```
🔴 ShipStation → Lyfe Order Creation Failed
Duplicate entry for source_order_id SS-12345
🔗 https://yoursite.frappe.cloud/app/shipstation-orders/SS-ORD-2024-001
{
  "shipstation_order_id": "SS-12345"
}
```

**Email subject:** `[LH Alert] Gate Pass Submission Failed`

---

### Frappe Cloud Monitoring

- Enable **slow query log** in MariaDB settings — capture queries >1s for 24h, then index or optimize
- Set **Frappe Cloud email alerts** for: CPU > 80%, memory > 85%, queue depth > 100
- Watch metrics: `frappe_background_jobs_queued`, `frappe_db_query_duration_seconds`

---

## 6. Deployment & Testing Strategy

### Deployment Rules

- Never deploy during peak business hours (IST morning–evening)
- Use Frappe Cloud **maintenance mode** during deploys
- Keep a rollback-ready git branch at all times

### Load Testing Plan (Staging)

Use [Locust](https://locust.io/) against the staging site:

```python
# locustfile.py
from locust import HttpUser, task

class OrderUser(HttpUser):
    @task
    def create_order(self):
        self.client.post(
            "/api/method/lh.lyfe_hardware.integrations.shopify.webhook",
            json=sample_shopify_payload,
            headers={"X-Shopify-Hmac-Sha256": "..."}
        )
```

```bash
locust -f locustfile.py --host=https://your-staging.frappe.cloud -u 50 -r 5
```

**Test targets:**

| Metric | Target |
|---|---|
| Webhook response time | < 500ms |
| Order creation job duration | < 10s |
| Background job queue depth at peak | < 50 |
| DB query p95 | < 1s |
| Error rate | < 0.5% |

---

## 7. Implementation Checklist

- [ ] Add DB index patch (`lh/patches/v1/add_performance_indexes.py`) and register in `patches.txt`
- [ ] Add `has_value_changed` guard to Lyfe Order `on_update` hooks
- [ ] Add Redis mutex to ShipStation sync and Shopify token rotation
- [ ] Wrap all webhook handlers to enqueue and return immediately
- [ ] Add `notify_developer` utility (`lh/lyfe_hardware/utils/alerts.py`) and wire into Gate Pass, ShipStation Orders, Shopify, ShipStation sync, and SLA scan
- [ ] Add `slack_webhook_url` and `developer_alert_email` fields to ShipStation Settings
- [ ] Configure Frappe Cloud worker counts (2 default, 2 short, 1 long)
- [ ] Enable slow query log on staging, run 24h capture
- [ ] Run Locust load test on staging at 50 concurrent users
- [ ] Enable Frappe Cloud email/Slack alerts for CPU, memory, queue depth
- [ ] Verify all ShipStation syncs use `modifyDateStart` pagination
