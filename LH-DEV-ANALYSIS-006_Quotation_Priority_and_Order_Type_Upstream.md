# Capturing Priority & Order Type at the Quotation Stage

**Doc ID:** LH-DEV-ANALYSIS-006  
**Related:** LH-DEV-ANALYSIS-BR2 (Reverse SLA), LH-DEV-ANALYSIS-005 (Dashboards), LH-BRD-DEV-004 v1.1, LH_PM_Rules_Developer_Spec  
**Date:** 2026-06-18  
**Prepared for:** Viral (ERPNext Dev), CS Team, Operations Manager  
**Status:** Feasibility + Design Proposal — review before implementation

---

## 1. The Request in One Line

Move the **Priority** decision (and ensure **Order Type** is captured) to the **Quotation** stage — when CS already knows the customer's urgency — instead of after the Lyfe Order exists, so every downstream date, SLA, and routing calculation is correct from the very first stage.

---

## 1a. Deployment Scope — New Orders Only

**This feature applies only to orders created AFTER deployment.** Existing orders in production are not retrofitted.

Two consequences, both in our favour:

| Concern | Resolution |
|---|---|
| Existing Lyfe Orders without a Quotation priority | Left untouched. They keep whatever `order_priority` they have. No backfill. |
| Migration of old `order_priority` values (Open Q2) | **Largely a non-issue.** Production has *very few* high-priority orders today, so re-labeling the field options has minimal data impact. We still map old → new for display consistency (Medium/Low → Normal, High → High), but there is no risky bulk migration of urgent orders. |

**Implementation guard:** key the new behaviour off a deployment cutover — e.g. only run the Quotation→Lyfe Order sync and the urgent BR-2 trigger for documents created on/after the go-live date (or simply: the sync only fires when `custom_priority` is present, which by definition only new Quotations will have). No special dating logic needed — old orders never had `custom_priority`, so the sync naturally skips them.

This keeps the rollout safe: nothing about live, in-production orders changes on deploy day.

---

## 2. What Already Exists (Critical — read before designing)

I checked the code. The starting point is better than the request assumes:

| Field | Lives On | Type / Options | Status |
|---|---|---|---|
| `custom_order_type` | **Quotation** (already exists) | Select: `Standard` / `Custom` | ✅ Already there — just not synced downstream |
| `custom_order_complexity` | **Quotation** (already exists) | Select: `Simple` / `Complex` | ✅ Already there |
| `custom_estimated_delivery_date` | **Quotation** (already exists) | Date | ✅ Already there — the natural input for BR-2 reverse calc |
| `order_type` | **Lyfe Order** | Select: `Standard` / `Custom` | ⚠ Set from **ShipStation/Shopify source**, NOT from Quotation |
| `order_priority` | **Lyfe Order** | Select: `High` / `Medium` / `Low` (default Medium) | ⚠ No "Urgent"; set manually; no logic attached |
| **Priority** | **Quotation** | — | ❌ **Does not exist** — this is the new field to add |

**Link between them:** Lyfe Order has a `source_quotation` (Link → Quotation) field. This is the **sync anchor** — the one join we build everything on.

**Hooks already wired on Quotation:** `validate`, `after_insert`, `on_update` (in `doc_events/quotation.py`). We extend these — no new hook registration needed.

### Two important realities this changes

1. **Order Type is NOT currently flowing Quotation → Lyfe Order.** Today, Lyfe Order's `order_type` comes from the ShipStation/Shopify `is_custom_order` flag (`shipstation_orders.py:167`, `shopify_order.py:27`). The Quotation's `custom_order_type` is captured but **never copied** to the Lyfe Order. So the request to "sync Order Type from Quotation" is partly **building a link that does not exist yet**, not just moving an existing one.

2. **"Urgent" is not a valid `order_priority` value yet.** BR-2 (LH-DEV-ANALYSIS-BR2 §9 Q1) already flagged this. Whatever priority vocabulary we choose must be decided **once** and used on both Quotation and Lyfe Order — see §4.

---

## 3. Feasibility Verdict

**Feasible, and architecturally sound — with three caveats that must be decided up front.**

| Aspect | Verdict |
|---|---|
| Add Priority field to Quotation | ✅ Trivial (custom field) |
| Prompt for Order Type when Priority is set | ✅ Standard client-script + server validation |
| Sync both to Lyfe Order on conversion | ✅ The `source_quotation` link already exists; add a copy step |
| Drive SLA / date / routing from synced values | ✅ Aligns with BR-2 and the dashboard plan |
| **Caveat 1 — vocabulary** | ⚠ Priority values must be reconciled across Quotation, Lyfe Order, and BR-2's "Urgent" (§4) |
| **Caveat 2 — source-of-truth conflict** | ⚠ Lyfe Order `order_type` already comes from ShipStation/Shopify. Quotation vs source — which wins? (§5) |
| **Caveat 3 — post-conversion edits** | ⚠ Changing Priority/Type after the Lyfe Order exists must recompute everything (§7) |

---

## 4. Caveat 1 — Decide the Priority Vocabulary ONCE

The request proposes: `Normal / Urgent / High Priority / VIP / Custom`.  
BR-2 and the existing Lyfe Order field use: `Urgent` (new) on top of `High / Medium / Low`.  
These **must be the same list** or sync becomes a lossy mapping nightmare.

### Recommendation

Adopt **one** priority list, identical on both doctypes:

```
Normal   (replaces "Medium" as the default)
High
Urgent
VIP
```

Rationale:
- `Urgent` is required by BR-2 (triggers the reverse-SLA compressed-timeline rules).
- `VIP` maps to the payment-terms / milestone work (BR-1 in the BRD — VIP customers get split payments).
- Drop `Low` and `Custom` from the request: "Low" adds noise (everything not-urgent is just Normal); "Custom" is not a priority, it's an order *type* (already covered by `custom_order_type`).
- Migrate existing Lyfe Order values: `Medium → Normal`, `Low → Normal`, `High → High`.

**This is the single most important decision in this doc.** Everything else (sync, SLA, dashboards) keys off it. See Open Questions §10 Q1.

---

## 5. Caveat 2 — Resolve the Order-Type Source-of-Truth Conflict

Today there are **two independent origins** for whether an order is Custom:

```
ShipStation / Shopify  ──(is_custom_order)──►  Lyfe Order.order_type   (current path)
Quotation.custom_order_type  ──────X──────►   (not connected today)
```

If we now also sync from Quotation, an order created from a Quotation that is *also* later touched by a ShipStation sync could get conflicting values.

### Recommendation: Quotation wins when a Quotation exists

Precedence rule:
1. **If `Lyfe Order.source_quotation` is set** → `order_type` and `priority` come from the Quotation. The Quotation is the human's deliberate decision and should not be overwritten by an integration.
2. **If no Quotation** (pure ShipStation/Shopify order) → keep today's behaviour (`is_custom_order` drives `order_type`; priority defaults to `Normal`).

Implement as a guard in the sync step: integration code only sets `order_type` if `source_quotation` is empty.

---

## 6. Recommended Design

### 6.1 Where the fields live — answer to Feasibility Q1

**Maintain on Quotation as the master; sync (copy) downstream to Lyfe Order; keep an editable copy on Lyfe Order.**

Not "only on Quotation" — because:
- Not every Lyfe Order has a Quotation (ShipStation/Shopify direct orders). Lyfe Order must hold its own value.
- Operations sometimes need to change priority *after* conversion (rush escalation mid-production). Lyfe Order must be editable.

So: **Quotation is the origin/master at creation; Lyfe Order holds the live working value.** After conversion they are independent (with a recompute trigger on change — §7).

### 6.2 New / changed fields

| Field | DocType | Change |
|---|---|---|
| `custom_priority` | Quotation | **New** Select: `Normal / High / Urgent / VIP`, default `Normal` |
| `custom_order_type` | Quotation | **Exists** — make it the synced source; add validation (§6.3) |
| `custom_estimated_delivery_date` | Quotation | **Exists** — becomes the BR-2 reverse-calc input at quote stage |
| `order_priority` | Lyfe Order | **Change options** to `Normal / High / Urgent / VIP`; migrate old values |
| `order_type` | Lyfe Order | **No schema change**; add Quotation-precedence sync logic |

### 6.3 Validation & prompt flow — answer to Expected Workflow steps 2–4

**Client-side (UX, instant feedback) — `quotation.js`:**
- When `custom_priority` is set to anything other than `Normal` **and** `custom_order_type` is empty → open a prompt/dialog forcing the user to pick Order Type before continuing.
- Optional: when Priority = `Urgent`, immediately run the BR-2 dispatch calculator using `custom_estimated_delivery_date` and show the Possible/Borderline/Not-Possible verdict right on the Quotation (earliest possible feasibility check — exactly what the BRD wants).

**Server-side (enforcement, cannot be bypassed) — `doc_events/quotation.py validate()`:**
```python
def validate(doc, method=None):
    # ...existing validation...
    if doc.custom_priority and doc.custom_priority != "Normal" and not doc.custom_order_type:
        frappe.throw(_("Order Type is required when Priority is {0}.").format(doc.custom_priority))
```
Per the CLAUDE.md rule: **JS is UX only; the throw in `validate` is the real gate.** A Quotation cannot be saved/submitted with a special priority but no order type.

### 6.4 Sync on conversion — answer to Expected Workflow step 5

At the point a Lyfe Order is created from a Quotation (where `source_quotation` is set), copy:
```
Quotation.custom_priority        → Lyfe Order.order_priority
Quotation.custom_order_type      → Lyfe Order.order_type
Quotation.custom_estimated_delivery_date → Lyfe Order (drives BR-2 promised_dispatch_by calc)
```
Guarded by the §5 precedence rule (Quotation wins; integration does not overwrite).

### 6.5 Driving SLA & dates — answer to Expected Workflow steps 6–7

Once `order_type` + `order_priority` land on the Lyfe Order:
- If `order_priority = Urgent` → fire the **BR-2 reverse calculator** (LH-DEV-ANALYSIS-BR2 §8.5): compute `promised_dispatch_by`, create the urgent PM task, apply compressed-timeline rules.
- The forward SLA detectors (R1–R7, per LH_PM_Rules spec) already key off `order_type` — they now get the correct value **from the first stage**, fixing the current gap where `order_type` was integration-derived.
- The dashboard exclusion (LH-DEV-ANALYSIS-005 §3.1, §11.2) keys off `order_priority = Urgent` — now reliably set.

---

## 7. Caveat 3 / Edge Cases — Post-Conversion Changes (Feasibility Q3, Q4)

This is where most of the risk lives. Every edge case below must be handled.

| Edge case | Required behaviour |
|---|---|
| Priority changed on Lyfe Order **after** creation (e.g. Normal → Urgent mid-production) — *incl. when CS never set it on the Quotation* | **Fully supported — see §7a.** `on_update` detects the change via `get_doc_before_save()` and runs the same urgent logic as the Quotation path. The source does not matter, only the value. |
| Priority **downgraded** from Urgent → Normal on Lyfe Order | BR-2 downgrade handler: close urgent PM task, clear `promised_dispatch_by`, log. (BR-2 §8.5.) |
| `order_type` changed Standard → Custom after creation | Recompute all SLA targets (custom uses 10 wd internal-review / 14–16 wd dispatch vs standard 3 wd). Re-evaluate any open SLA tasks against the new threshold. |
| Quotation Priority/Type changed **after** Lyfe Order already created | **Do NOT silently overwrite the Lyfe Order.** Surface a warning/notification to CS: "Quotation X priority changed; linked Order Y may need review." Manual confirm — consistent with the BRD's "do not silently update" stance on milestone changes. |
| Quotation revised (new version) before conversion | Carry the **latest** Priority/Type to the Lyfe Order (matches BRD payment-schedule edge case: "carry the latest schedule version, not the original"). |
| Priority set but `custom_estimated_delivery_date` empty (Urgent with no target date) | Block: Urgent requires a target date to compute dispatch-by. Validation throw. |
| Two Quotations → one merged Lyfe Order | Merge logic must pick a priority (highest wins: VIP > Urgent > High > Normal) and flag for CS review. |
| Order has no Quotation (ShipStation/Shopify direct) | §5 fallback: `is_custom_order` drives type; priority defaults Normal; no prompt (no Quotation UI involved). |

### 7a. CS did NOT set priority on the Quotation, and now wants to set it directly on the Lyfe Order

This is the most common real-world path (CS often realises urgency only after the order exists), so it gets its own design. **It is fully supported — the Lyfe Order is editable and is treated as an independent live value after creation (§6.1).**

**What happens when CS edits `order_priority` directly on the Lyfe Order:**

The Lyfe Order controller already uses the `get_doc_before_save()` pattern in `on_update` (confirmed at `lyfe_order.py:282` and used in ~7 places). We add a priority-change branch using the exact same idiom:

```python
def on_update(self):
    before = self.get_doc_before_save()
    old_priority = before.order_priority if before else None

    if self.order_priority != old_priority:
        _handle_priority_change(self, old_priority, self.order_priority)
```

`_handle_priority_change` then:

| New priority | Action on direct Lyfe Order edit |
|---|---|
| → **Urgent** | Require `expected_shipping_date` (or prompt for a target date); run BR-2 reverse calc; set `promised_dispatch_by`; create the urgent PM task; apply compressed-timeline rules. Identical to the Quotation-sourced path — the trigger is the *field value*, not *where it came from*. |
| **Urgent** → Normal/High | BR-2 downgrade handler: close urgent PM task, clear `promised_dispatch_by`, log the downgrade + who did it. |
| → High / VIP | Update routing/labels; no compressed-timeline calc (only Urgent triggers BR-2). VIP may flag payment-preset review (BR-1). |

**Key design principle — the source does not matter, the value does.** Because the urgent logic lives in the Lyfe Order's `on_update` keyed on the *field value changing*, it fires identically whether the priority arrived:
- synced from the Quotation at creation, **or**
- typed directly into the Lyfe Order later by CS.

There is **no separate code path** to maintain for "set on Quotation" vs "set on Lyfe Order." The Quotation sync (§6.4) simply *sets the field*; the `on_update` handler does the rest. This is why §6.1 keeps an editable copy on the Lyfe Order rather than locking it to the Quotation.

**Validation parity:** the same rule as the Quotation (§6.3) applies on the Lyfe Order — if priority is set to a special value but `order_type` is missing, `validate()` throws. So a CS user cannot mark a Lyfe Order Urgent without an order type, exactly as they cannot on the Quotation.

**No drift back to the Quotation.** Editing priority on the Lyfe Order does **not** write back to the source Quotation. The Quotation records what was decided at quote time; the Lyfe Order records the live operational reality. They are intentionally independent post-creation (§6.1, §7). If management wants quote-vs-actual reporting later, that is a read-only comparison, not a sync.

**UX:** add the same client-side prompt to `lyfe_order.js` — when CS picks Urgent and `order_type`/target date is blank, prompt before save. (UX only; the `validate()` throw is the real gate, per CLAUDE.md.)

### Should changes auto-recalculate? (Feasibility Q4)

**Yes for dates/SLA on the Lyfe Order itself** (priority/type change → recompute dispatch-by, SLA targets, PM task). This is deterministic and safe.

**No for cross-document overwrite** (Quotation change → Lyfe Order). Notify + manual confirm. Auto-overwriting a live production order from an upstream quote edit is dangerous (the order may already be in the factory).

---

## 8. Impact Analysis (Feasibility Q2)

| Area | Impact | Action |
|---|---|---|
| **Existing workflows** | Lyfe Order `order_type` now sometimes sourced from Quotation | Add §5 precedence guard in ShipStation/Shopify sync |
| **SLA rules (R1–R7)** | Positive — they get correct `order_type` from stage 1 | No rule change; verify detectors read the synced value |
| **Date calculations (BR-2)** | Positive — `custom_estimated_delivery_date` available at quote stage = earliest feasibility check | Wire BR-2 calc to Quotation form too |
| **Reports / dashboards** | `order_priority` vocabulary changes (new values) | Update filters/labels in `order_analysis*` + status overview; migrate old values |
| **Integrations (ShipStation/Shopify)** | Must not overwrite Quotation-sourced values | §5 guard |
| **Notifications** | Existing payment/SLA notifications key off Lyfe Order fields — unaffected | Add new "priority changed post-conversion" notification (§7) |
| **Payment terms (BR-1 / VIP)** | `VIP` priority can gate VIP payment presets | Coordinate with BR-1 milestone work |

---

## 9. Synchronization Strategy (Summary)

```
┌─────────────────────────────────────────────────────────────┐
│ QUOTATION (master at creation)                              │
│  custom_priority        (Normal/High/Urgent/VIP)            │
│  custom_order_type      (Standard/Custom)                   │
│  custom_estimated_delivery_date                             │
│                                                             │
│  validate(): if priority != Normal AND no order_type → THROW│
└───────────────────────────┬─────────────────────────────────┘
                            │ on conversion (source_quotation set)
                            │ COPY — Quotation wins over integration (§5)
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ LYFE ORDER (live working value — independent after copy)    │
│  order_priority   ← custom_priority                         │
│  order_type       ← custom_order_type (if no quotation:     │
│                      fallback to is_custom_order)           │
│                                                             │
│  on priority/type change → RECOMPUTE dispatch-by, SLA, task │
│  if Urgent → BR-2 reverse calc + urgent PM task             │
└───────────────────────────┬─────────────────────────────────┘
                            │ feeds (correct values from stage 1)
                            ▼
   Forward SLA (R1–R7) · BR-2 reverse SLA · Dashboards · Alerts
```

After the initial copy, the two are **independent**: editing the Quotation does not auto-push to a live Lyfe Order (notify + manual confirm instead, §7).

---

## 10. Open Questions Before Building

| # | Question | Who Decides | Why It Matters |
|---|---|---|---|
| 1 | **Final priority vocabulary** — adopt `Normal/High/Urgent/VIP`? Or keep the request's `Normal/Urgent/High Priority/VIP/Custom`? | CS / Srishti / Founders | Everything keys off this. Must be ONE list on both doctypes. Recommend dropping "Low" and "Custom" (Custom is a type, not a priority). |
| 2 | Migration of existing Lyfe Order `order_priority` values (Medium/High/Low → new list) | Operations | **Low risk — new-orders-only scope (§1a) + very few high-priority orders in production today.** Still map for display (Medium/Low→Normal, High→High), but no risky bulk migration. |
| 3 | When a Quotation's priority/type changes after the Lyfe Order exists — notify-only, or allow a "push update" button? | CS | Auto-overwrite is risky for in-production orders. Recommend notify + manual confirm. |
| 4 | Should the BR-2 dispatch calculator run automatically on the Quotation when Priority=Urgent, or only on a button click? | CS | Auto = earliest warning; button = less noise. Recommend auto-run + show verdict banner. |
| 5 | Does `VIP` priority also unlock the VIP payment-milestone presets (BR-1)? | Founders | Couples this work to the payment-terms feature; confirm the intended link. |
| 6 | Is "merged orders → highest priority wins" the right rule? | CS | Affects merge logic; alternative is "manual pick on merge." |

---

## 11. Recommended Build Order

| Step | What | Depends On |
|---|---|---|
| 1 | **Decide vocabulary (§4) + migration mapping** | Q1, Q2 — blocking |
| 2 | Change `order_priority` options on Lyfe Order + migrate existing values | Step 1 |
| 3 | Add `custom_priority` to Quotation + `validate()` throw (Order Type required when priority ≠ Normal) | Step 1 |
| 4 | `quotation.js` prompt for Order Type + (optional) inline BR-2 verdict | Step 3 |
| 5 | Sync on conversion (Quotation → Lyfe Order) with §5 precedence guard | Steps 2–3 |
| 6 | Recompute-on-change handler on Lyfe Order (priority/type change → BR-2 recalc + SLA retarget) — **covers both the Quotation-sync path AND direct CS edits on the Lyfe Order (§7a)** | BR-2 live |
| 7 | Cross-doc "quotation changed post-conversion" notification | Step 5 |
| 8 | Update dashboards/reports for new vocabulary | Step 2 + LH-DEV-ANALYSIS-005 |

**Sequencing note:** Steps 1–5 can be built **before or alongside BR-2** (they prepare the data). Step 6 needs the BR-2 calculator to exist. This work is the *upstream feeder* for BR-2 and the dashboards — doing it first means BR-2 and the SLA engine receive correct `order_type`/`order_priority` from day one, instead of being retrofitted.

**Scope reminder (§1a):** all steps apply to **new orders only**. The sync (Step 5) naturally skips existing orders because they have no `custom_priority`. Step 2's option re-label is low risk given how few high-priority orders exist in production today — no risky bulk migration. Nothing about live in-production orders changes on deploy day.

**Step 6 covers both entry points:** because the recompute handler keys off the *field value changing* in `on_update` (§7a), it fires identically whether priority was synced from the Quotation or typed directly into the Lyfe Order by CS — one code path, no duplication.

---

## 12. Risks & Better Alternatives (Feasibility Q6)

| Risk / Alternative | Assessment |
|---|---|
| **Risk:** vocabulary drift between Quotation and Lyfe Order | Mitigated by §4 (single list) — but only if enforced. Use the SAME select options string in both JSONs. |
| **Risk:** integration overwrites human decision | Mitigated by §5 precedence guard. Must be tested with a real ShipStation sync on a Quotation-linked order. |
| **Risk:** post-conversion auto-recalc surprises factory mid-production | Mitigated by §7 — auto-recalc only on the Lyfe Order's own field edits (deliberate human action), never silently from upstream. |
| **Alternative considered:** keep priority only on Lyfe Order (status quo) | Rejected — defeats the business reason (capture urgency when CS knows it, at quote stage). |
| **Alternative considered:** make Quotation the *sole* store, no copy on Lyfe Order | Rejected — breaks ShipStation/Shopify direct orders (no Quotation) and blocks mid-production priority changes. |
| **Better-practice add:** run BR-2 feasibility on the Quotation | Recommended — gives CS a Possible/Not-Possible answer *before* committing to the customer, which is the BRD's core intent for urgent orders. |

---

## 13. One-Paragraph Summary for Management

Capturing Priority and Order Type at the Quotation stage is feasible and is in fact the *right* foundation for the Reverse-SLA (BR-2) and dashboard work — it ensures every date, SLA, and routing decision is correct from the first stage instead of being patched after the order exists. The Quotation already has Order Type and an Estimated Delivery Date; we mainly need to **add a Priority field**, **build the Quotation→Lyfe Order sync that does not exist today** (Order Type currently comes from ShipStation/Shopify, not the Quotation), and **decide one shared priority vocabulary** (recommended: Normal / High / Urgent / VIP) used identically on both documents. Three decisions must be locked first: the vocabulary + migration of existing values, the rule that the Quotation's human-entered values win over integration-derived ones, and that post-conversion changes recalculate the order's own dates but never silently overwrite a live production order from an upstream quote edit. Done in this order, this becomes the upstream feeder that makes BR-2 and the SLA dashboards accurate by construction. The rollout is **new-orders-only** — existing in-production orders are untouched on deploy day, and because production has very few high-priority orders today, re-labeling the priority field carries minimal data risk. CS does not have to set priority on the Quotation: if they skip it there and later realise the order is urgent, they can set it **directly on the Lyfe Order**, and the exact same urgent logic (reverse-SLA calc, PM task, compressed timeline) fires automatically — because the behaviour keys off the priority *value changing*, not where it was entered, so there is a single code path for both routes.
