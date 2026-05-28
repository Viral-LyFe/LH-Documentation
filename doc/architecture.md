# LH App — Architecture & Coding Conventions

> This is the developer reference. For command-line usage see `apps/lh/CLAUDE.md`.

---

## Module Layout

The app has two top-level modules inside `apps/lh/lh/`:

```
lh/
├── lyfe_hardware/          # Core business domain
│   ├── doctype/            # Custom DocType controllers
│   ├── doc_events/         # Hooks on ERPNext-owned DocTypes
│   ├── overrides/          # Full controller replacements
│   ├── shipping/           # Rate, label, box-packing logic + carrier API clients
│   ├── integrations/       # Shopify, 17Track, token rotation
│   ├── tasks/              # Scheduled background jobs
│   └── utils/              # Shared helpers
│
└── lh_project/             # Project & task management layer
    ├── doctype/             # SLA Rule, Task Automation Rule, Project Task Status
    ├── automation/          # Event-driven task creation engine
    └── sla/                 # SLA monitoring engine
```

---

## Architectural Rules

### 1. DocType logic stays in its own folder

All business logic for a DocType we own lives in `doctype/<doctype_name>/<doctype_name>.py`.

```python
# apps/lh/lh/lyfe_hardware/doctype/gate_pass/gate_pass.py
class GatePass(Document):
    def before_submit(self):
        _validate_items_before_submit(self)
        _validate_material_issue_submitted(self)
```

Never put DocType logic in `doc_events/` or anywhere else.

### 2. doc_events/ is for ERPNext-owned DocTypes only

When you need to react to a Quotation, Item, Sales Invoice, or any DocType you do not own, put the handler in `doc_events/`:

```python
# apps/lh/lh/lyfe_hardware/doc_events/quotation.py

def validate(doc, method=None):
    _check_required_fields(doc)

def on_update(doc, method=None):
    _sync_shopify_draft(doc)
```

Register it in `hooks.py`:

```python
doc_events = {
    "Quotation": {
        "validate": "lh.lyfe_hardware.doc_events.quotation.validate",
        "on_update": "lh.lyfe_hardware.doc_events.quotation.on_update",
    },
}
```

### 3. External APIs live in integrations/ or shipping/

- `integrations/` — third-party SaaS APIs that are not shipping-specific (Shopify, 17Track)
- `shipping/` — carrier APIs (CJ, FedEx, ShipStation, ShipEngine), rate calculation, box selection

Each file is a standalone module. No cross-imports between integration files.

### 4. Scheduled tasks register in hooks.py

```python
# tasks/my_task.py
def run():
    ...

# hooks.py
scheduler_events = {
    "cron": {
        "0 9 * * 1-5": [  # 9 AM Mon–Fri
            "lh.lyfe_hardware.tasks.my_task.run",
        ],
    },
}
```

### 5. lh_project is self-contained

The `lh_project` module does not import from `lyfe_hardware` and vice versa. They communicate only through doc_events hooks and the shared `frappe.db` layer.

---

## Python Conventions

### File structure

```python
"""
Module docstring — public API, what it does, config keys it reads.
"""
import frappe

# ── Private helpers ───────────────────────────────────────────────────────────

def _helper_one():
    ...

def _helper_two():
    ...

# ── Public / whitelisted ──────────────────────────────────────────────────────

@frappe.whitelist()
def my_api_function(doc_name):
    doc = frappe.get_doc("My DocType", doc_name)
    ...

# ── Controller class ──────────────────────────────────────────────────────────

class MyDocType(Document):
    def validate(self):
        _helper_one()

    def before_submit(self):
        _helper_two()
```

### Error handling

```python
# User-facing validation — stops the save, shown in UI
frappe.throw(_("Weight is required for all rows."))

# Multi-issue errors — use HTML list
errors = ["Row 1: missing weight", "Row 3: missing carrier"]
frappe.throw("<ul>" + "".join(f"<li>{e}</li>" for e in errors) + "</ul>")

# Background errors — log, don't crash the user
try:
    sync_to_shopify(doc)
except Exception:
    frappe.log_error(title="Shopify sync failed", message=frappe.get_traceback())
```

### Reading config

```python
# Single DocType config
settings = frappe.get_single("ShipStation Settings")
api_key = settings.ss_api_key

# Password fields
secret = settings.get_password("ss_api_secret")

# Frappe site config (frappe.conf)
webhook_secret = frappe.conf.get("shopify_webhook_secret")
```

### Database queries

```python
# Simple lookup
order = frappe.get_doc("Lyfe Order", order_name)

# Filtered list
orders = frappe.get_all("Lyfe Order",
    filters={"status": "Factory Assignment", "warehouse": "Factory - LH"},
    fields=["name", "customer", "order_date"],
    order_by="order_date asc",
)

# Child table JOIN (correct pattern)
rows = frappe.db.sql("""
    SELECT item.lyfe_order, item.item_code, item.issue_qty
    FROM `tabMaterial Issue Order Item` item
    JOIN `tabMaterial Issue for Order` mifo ON mifo.name = item.parent
    WHERE item.lyfe_order = %s AND mifo.docstatus = 1
""", (order_name,), as_dict=True)

# NEVER use frappe.client.get_value on child DocTypes from JS (causes 403)
# Use the parent with child filter instead:
frappe.db.get_list("Gate Pass", filters=[["Gate Pass Item", "order_id", "=", name]])
```

---

## JavaScript Conventions

```js
frappe.ui.form.on("My DocType", {
    // Field change handler
    my_field(frm) {
        frm.set_value("other_field", frm.doc.my_field.toUpperCase());
    },

    // Toolbar buttons on refresh
    refresh(frm) {
        if (frm.doc.docstatus === 1) {
            frm.add_custom_button("Generate Label", () => {
                frappe.call({
                    method: "lh.lyfe_hardware.doctype.lyfe_order.lyfe_order.generate_label",
                    args: { order_name: frm.doc.name },
                    callback: r => {
                        if (r.message) frm.reload_doc();
                    },
                });
            }, "Shipping");
        }
    },

    // Block a workflow transition
    before_workflow_action(frm) {
        if (frm.doc.workflow_action !== "Ready for Dispatch") return;
        return new Promise((resolve, reject) => {
            const missing = [];
            if (!frm.doc.cj_export_partner) missing.push("Export Partner");
            if (missing.length) {
                frappe.msgprint(`Required: ${missing.join(", ")}`);
                reject();
            } else {
                resolve();
            }
        });
    },
});
```

---

## Adding a New Feature — Checklist

1. **DocType changes** — edit the `.json` file, run `bench migrate`
2. **Python logic** — add to the right folder (see table above)
3. **hooks.py** — register any new doc_events or scheduled tasks
4. **Patches** — if migrating existing data, add to `patches/`, register in `patches.txt`
5. **Documentation** — add or update a file in `/home/frappe/frappe-bench/LH Documentation/doc/`

---

## Documentation Index

| Area | File |
|---|---|
| This file | `doc/architecture.md` |
| Lyfe Order lifecycle | `doc/lyfe_order/lifecycle.md` |
| Lyfe Order usecases | `doc/lyfe_order/usecases.md` |
| Gate Pass | `doc/shipping/gate_pass.md` |
| Shipping rates & box selection | `doc/shipping/rates_and_box_selection.md` |
| ShipStation integration | `doc/shipping/shipstation.md` |
| CJ Logistics / FedEx | `doc/shipping/cj_fedex.md` |
| Shopify integration | `doc/integrations/shopify.md` |
| 17Track integration | `doc/integrations/seventeen_track.md` |
| SLA Engine | `doc/sla_engine/overview.md` |
| Task Automation | `doc/lh_project/task_automation.md` |
| Project Board & Kanban | `doc/lh_project/project_board.md` |
