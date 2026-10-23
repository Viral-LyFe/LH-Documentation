# ERP Workflow Review, October 2026

| | |
|---|---|
| **Owner** | Viral |
| **Prepared** | 23 Oct 2026 |
| **Due to TJ** | Mon 26 Oct 2026, 7 pm IST (Slack thread in #erpnext-lh) |
| **cc** | Ravin, Arth, Srishti |
| **Status** | Draft. User walkthroughs are still to be done (see section 9) |

---

## 1. What was asked

On 19 Oct TJ raised that too many errors reach users of the ERP, particularly on the client-facing side. Usability is being treated as secondary instead of being built in from the start. On 23 Oct he followed up that no action was yet visible and asked for two things by **Mon 26 Oct, 7 pm IST**:

1. A review of the **existing** ERP workflows from the users' side: which workflows, for which users (CS, Factory, others), and what was found. That means where users get stuck, which steps are unnecessary, and where errors keep coming back.
2. What will be fixed, in what order, and by when. **Every item goes on Asana with a due date.**

His standing questions for every workflow:
- How will CS or Factory actually use it?
- Is the workflow intuitive?
- Does it make their jobs easier, or add unnecessary steps?
- What could go wrong?
- Have the different scenarios and edge cases been considered?
- Would it be acceptable to release without someone else pointing out the obvious gaps?

The goal: *catch problems before they reach users, not rely on users to discover them.*

### 1.1 Related asks from others

| Who | Date | Ask |
|---|---|---|
| Srishti | 17 and 20 Oct | An end-to-end test report on Asana and Slack after every change. Nothing goes live on the quotation or payment flow until it's shared, including internal changes like timesheets. Also: an RCA for missing payment links, an alert when a quotation has no payment link, and a plan for the cancel/timesheet blocker. |
| Srishti | 1 Oct | Test the workflow yourself before handing it over. |
| Srishti | 30 Sep | Post department-specific updates in that department's channel. |
| Srishti | 27 Jul | Users should not have to "prove" there is a bug. Asana links for all tasks. |
| Arth | 7 Oct | All ERP requests go through the Asana form. Fill in Task Type and the PR / Commit link. |
| Arth | 21 Oct | Add a "Report bug" button and minimise clicks. |
| TJ | 3 Aug | "Run it there before a bug report comes back to me" (MCP). |
| TJ | 22–23 Oct | A separate **security posture review** across 6 areas, with a reply due 23 Oct 7 pm IST. The audit subtasks are dated 26–31 Oct. |

---

## 2. Method and sources

| Source | What was reviewed | Window |
|---|---|---|
| Codebase (`apps/lh`) | Workflow inventory, 1,439 commits, fix and revert history, user-facing error messages, test coverage | 1 Jul – 23 Oct 2026 |
| `.ai-worklog/` | Problems recorded while fixing | Jul – Oct 2026 |
| Slack | #erpnext-lh; DMs with Srishti, Cherry, Gaudioso and Mark; searches of #general, #dsm, #bot-testing, #1click-notification | 15 Jul – 23 Oct 2026 |
| Asana | ERPNext, ERP System Issue Management, ERPNext V2, ERP Warehouse & Shipping Module Dev | All open tasks; bugs since 20 Jul |
| Existing docs | `Lyfe_Order_Blocking_Validations.md`, `Quotation_Blocking_Validations.md` | Current |

**Gaps, to be closed before Monday:**
- #cs and #tech-cs could not be read, so CS-reported issues are probably undercounted.
- Factory (Rohit) does not use Slack. His issues arrive through Srishti or WhatsApp.
- No live walkthroughs with users yet. This is the most important gap (see section 9).
- The git history is from the local dev box, and commit-area classification is keyword-based (about ±10–15%).

---

## 3. Headline findings

1. **Fixes are a large share of the work.** 417 of the 1,439 commits since 1 Jul (**29%**) have fix, bug, error, hotfix, revert, broken, crash or wrong in the subject (662 if commit bodies are counted). There were **13 reverts**. **85** commits are data patches and `patches.txt` changed **146** times, which means a lot of bad data is being cleaned up after the fact.
2. **Changes break other features.** Srishti, 17 Oct: *"whenever a new change is done some other functionality gets affected."* The most-changed code, `lyfe_order.py` (8.5k lines, 85 user-facing throws, in 89 fix commits), has only a stub test.
3. **Users report problems by DM.** Most ERP problems reach Viral by DM. There is no shared intake, no record of how often each one recurs, and Factory has no direct channel.
4. **Users can't get out of a stuck state themselves.** When an order or quote gets into an unexpected state, CS or Ops can't move it on, so every edge case becomes a DM and a manual fix.
5. **Error messages are written for developers.** Messages show raw field names, vendor error codes, Python exceptions, or "check the Error Log", which CS can't open.
6. **Asana doesn't show the real state of work.** Viral has 61 open tasks: **38 overdue** and **12** with no due date. Only about 2 of 16 typed bug fixes have the PR / Commit link filled in.

---

## 4. Workflow-by-workflow review

Ranked by frequency × impact on users.

### 4.1 Quotation → approval → payment link

**Users:** CS (Gau, Cherry), TechCS (Mark), Ops (Srishti), Engineers, Founders (approval)
**Main code:** `doc_events/quotation.py` (29 throws), `public/js/quotation.js`, Quotation Payment Entry, Payment Terms
**Steps:** create quote → drawing upload (requires a Timesheet) → COGS / production cost → send for approval → founder approval → payment link → payment stages (these later block the Lyfe Order at Ready for Dispatch)

**What users hit (about 18 Slack reports, 6 Asana bugs, 55 fix commits, 1 revert):**
| Date | Reporter | Symptom |
|---|---|---|
| 15 Sep | Gau (CS) | "quite a few errors when creating a quote" ([link](https://lyfehardware.slack.com/archives/D0ADEPP54TY/p1789486694877299)) |
| 1 Oct | Srishti | Can't reject after internal review; "CS always reports new bugs" ([link](https://lyfehardware.slack.com/archives/D0ADDCW9XRB/p1790796690408879)) |
| 9 Oct | Srishti | An urgent quote couldn't be approved by the founders because of an error ([link](https://lyfehardware.slack.com/archives/D0ADDCW9XRB/p1791545652635679)) |
| 12 Oct | Mark (TechCS) | Send-for-approval error even though production cost was filled ([link](https://lyfehardware.slack.com/archives/D0ADT2BBXK6/p1791812316679469)) |
| 13 Oct | Gau (CS) | A customer could pay only by bank transfer; there was no card option ([link](https://lyfehardware.slack.com/archives/D0ADEPP54TY/p1791903164217729)). Fixed in about 1 hour, then it came back. LHQ-2026-0309 went unnoticed for about a week. |
| 20 Oct | Srishti | Cancelling a quote was still blocked by a linked timesheet |

Asana bugs: quote email not received (1216799585938751); quotes over 10k (1217493605921925); payment options (1219596035589526); tax doesn't match the Shopify link (1219616495754362); auto-follow-up, **still open and overdue since 27 Aug** (1217715464478670).
Worklog notes: CS is always blocked on drawing upload by the Timesheet rule; quotes stuck as Partially Paid over a few dollars; "BOM does not exist" after a BOM edit.

**Why it matters:** this is the one workflow customers see directly. A missing payment link or card option means a delayed or lost sale.

**Unnecessary steps and friction:**
- The Timesheet requirement on drawing upload blocks CS, and it later blocks cancelling too.
- Approval gates fail with unclear messages.

**Edge cases not covered:** quotes over 10k, tax rounding against Shopify, partial payments within a few dollars, rejection after internal review, cancelling with linked documents.

### 4.2 Lyfe Order lifecycle: status dead-ends

**Users:** CS, Factory, Engineer, Shipping Forwarder, Super Admin
**Main code:** `lyfe_order.py` (8.5k lines, 85 throws), `lyfe_order.js` (5k lines), the site Workflow, QC Item
**Steps:** New → Assign To Factory → Engineer Checks → Factory Upload → Internal Review → Ready for Dispatch → Dispatched → Awaiting Tracking → Shipped → (US Warehouse Delivered) → Completed

Ready for Dispatch alone stacks about 6 gates: Export Partner, about 15 export fields, HSN/HTS on each row, COGS on each item, a submitted MIFO, QC complete, and payment received.

**What users hit (about 14 Slack reports, almost weekly; 114 fix commits out of 338 in this area):**
| Date | Reporter | Symptom |
|---|---|---|
| 4 Sep | Cherry (CS) | "suddenly i dont have the option" ([link](https://lyfehardware.slack.com/archives/D0ADVP68JLR/p1788469252068889)) |
| 21 Sep | Srishti | "Factory has marked ready for dispatch and I am not able to change it" (5 orders) ([link](https://lyfehardware.slack.com/archives/D0ADDCW9XRB/p1790004393340699)) |
| 6 Oct | Gau (CS) | "Ready for dispatch" missing after assigning to the US warehouse ([link](https://lyfehardware.slack.com/archives/D0ADEPP54TY/p1791227347478809)) |
| 20 Oct | Cherry (CS) | Order stuck in "awaiting engineer check" ([link](https://lyfehardware.slack.com/archives/D0ADVP68JLR/p1792440958686739)) |

Asana bugs: LH3254 can't be assigned to the factory (1219592318920794); the Actions button disappears (1219204844680542); Route via US resets the workflow (1219720813583320); LH2583 total wrong (1216893924824267).
Example commits: 98c00c32 (keep the workflow state on Route via US Warehouse Instead); 39f72a13 → 825a32e2 (fix, then a backfill for Factory Leg Destination); 3aa1b233 (a throw added then removed).

**Why it matters:** CS and Ops have no way to recover by themselves, so each wrong click or edge case waits on Viral.

**Unnecessary steps and friction:**
- The Ready for Dispatch gates are discovered one at a time, one error per attempt.
- Rerouting (Force US / Force India / Mixed) changes which actions are available in ways users don't expect.

### 4.3 Order intake: Shopify / ShipStation sync

**Users:** automatic sync; CS handles the exceptions
**Main code:** `shipstation_orders.py`, `integrations/shopify.py`, ShipStation Settings, `merge_order.py` / Merge Suggestion (no tests)
**Steps:** sync every 15 minutes → match SKU to an ERP item → apply address and edit changes → merge suggestions → CS sets Item Group on each line

**What users hit (about 10 Slack reports; 54 of 112 commits in this area are fixes, **48%**, the highest share):**
| Date | Reporter | Symptom |
|---|---|---|
| 27 Jul | Srishti | Refunds missing on LH2583/LH2249 (ERP $1,284.98 vs Shopify $4,392.91). An hourly catch-up job was added on 31 Aug and recovered $3,406.90 across 9 orders. **Why Shopify's refund notifications don't arrive is still not known.** |
| 31 Jul | Cherry (CS) | "why edit is not reflecting" ([link](https://lyfehardware.slack.com/archives/D0ADVP68JLR/p1785436476570959)) |
| 10 Sep | Cherry (CS) | "keeps going to order modified… reverted it… multiple times" ([link](https://lyfehardware.slack.com/archives/D0ADVP68JLR/p1789064429583689)). Fixed 16–17 Sep. |
| 18 Sep | Gau (CS) | "line items are not reflected" ([link](https://lyfehardware.slack.com/archives/D0ADEPP54TY/p1789749917872999)) |
| 20 Oct | Cherry (CS) | Still unsure why edits don't apply to On Hold and New orders |

Commits: 7ae952fd (reverify all in-flight orders); e502ecfb and 937463cb (deadlock retry, fixed twice); 1319925c (Error Log noise).
Asana: customer-note modify (1218396920979879), refund error (1217990844270587).

**Why it matters:** bad data at intake flows into Factory, shipping and the P&L.

**Friction:**
- CS can't see *why* an edit wasn't applied.
- Item Group has to be set by hand on every line, and miscategorisation keeps coming back (25 Sep, 6 Oct, 21 Oct).

### 4.4 Routing, 1Click and US warehouse

**Users:** CS, Logistics, Ops; Factory catches problems
**Main code:** `oneclick_api.py`, `bulk_transfer_batch.py`, Order Leg, Transfer Order, US Warehouse Replenishment, OneClick Shipment Issue
**Steps:** choose route (Force US / Force India / Mixed) → confirm the warehouse split → post to 1Click (Create Order and Create Load) → send the inbound email → check stock. A failure parks the order in "1Click Error".

**What users hit (about 8 user reports plus many bot alerts; 50 of 198 commits in this area are fixes):**
| Date | Reporter | Symptom |
|---|---|---|
| 20 Jul | Srishti | LH2145 address change not applied; "Rohit noticed this in time" ([link](https://lyfehardware.slack.com/archives/D0ADDCW9XRB/p1784547134030119)) |
| 24–30 Sep | 1Click bot | "SKU does not exist" alerts. 6 POs were cancelled and reposted as "-1" on 7 Oct ([example](https://lyfehardware.slack.com/archives/C0C4LF2BXTN/p1790773957004549)) |
| 16 Oct | Srishti | The customer's address showed where the 1Click address should have ([link](https://lyfehardware.slack.com/archives/D0ADDCW9XRB/p1792162500895819)) |

**Fixes that needed fixing:**
- The Create Load `loadType` changed 4 times: 44b70916 → 01225b8a → 4dc23c0f → **42b20373 Revert**.
- Cancelled orders could still be posted (f6bb3461).
- The 30 Sep rule requiring a MIFO before 1Click posting blocked orders whose stock was fully in the US. It was reversed in October.
- The 1Click "available" → "onhand" setting was changed locally only, and production still needs the same change.

**Wrong items in a Load (mixed orders):** 6 of 16 orders in BTB-04492 and BTB-04534 needed corrections, an estimated **$374.90 loss**.

Asana bugs: address fixes (1216703051547672), Forwarded Orders (1217066517095516); 1219387648417971 is in For Review and overdue.

**Why it matters:**
- A wrong address means a parcel goes to the wrong place. The safety net today is Rohit noticing.
- Users see raw API exceptions from `oneclick_api.py:188`, `:437` and `:639`.

### 4.5 Factory pending list and visibility

**Users:** Factory (Rohit), relayed by Srishti
**What users hit (about 5 reports, a July cluster):**
- 20 Jul: "this did not come in Pending list".
- 22 Jul: LH2337 on the pending list but not assigned.
- 28 Jul: LH2588's drawing missing from the pending list.

Asana: main order hidden from Factory on reshipment (1218340402116133).

**Why it matters:** Factory is the user group with the least voice. They have no Slack and no direct way to report problems, so their problems are found late.

### 4.6 Material Issue for Order (MIFO)

**Users:** Factory stores
**Steps:** pick rack, finish or acrylic cuts and gift items → submit. Ready for Dispatch and 1Click posting are blocked until this is done.

**Problems found:**
- Gifts forced onto reshipment MIFOs (0f91a4ef).
- A race on the stock balance (17df20e5).
- The finish-vs-acrylic dialog fixed twice (641f4007, 5c7694c8).
- "Creation Failed" on concurrent Material Receipts.

Gift-rule bugs (4 in Asana): blank Extra Items row (1219634760678587), MI gift (1219430567957202), gift in the packing list (1219259884131682), 1Click gift payload (1219346551952632).

**Tests:** stub only.

### 4.7 Reshipment and Return / RTO

**Users:** CS creates them; Factory fulfils and marks returns received
**Problems found:**
- Parent order not updated (11abe843).
- **622d4425 Revert** (Active Reshipment filter).
- Orders stuck in "Return InProgress" with an empty return_type.
- The scheduler filter excluded every eligible order (df02bbed).
- Photos and signature (1216961576402756).

**Why it matters:** status has to stay in step between the parent order and the child document, and that sync is fragile. Tests here are comparatively good.

### 4.8 Tracking jobs overwrite statuses

**Users:** CS (sees the result)
**Problem:** the order's status changes under the user. For example, Return orders flipped back to Completed (3c5f2cac), plus dd94f92d, a563cddc and ea314538. That's 38 fix commits in this area.
**Meest:** CS could add Meest tracking numbers before and now can't (Gau, 15 Oct, [link](https://lyfehardware.slack.com/archives/D0ADEPP54TY/p1792006230549429)).
**Why it matters:** there is no error message; the order silently shows the wrong status.

### 4.9 Shipping labels and export documents (CJ)

**Users:** Factory
**Problems found:**
- 46 throws across `cj_api.py` and `cj_api_reshipment.py`, which pass raw vendor codes to the user.
- **No tests.**
- Print-format logos and signatures fixed 3 times (860fc392, 2802d463, 3ed7320a).
- Per-box flat charges reverted (d6d828fc → aae6bcdb).

### 4.10 Notifications, email and the Slack bot

**Problems found:**
- A workflow email went to TJ for a quote that was already won (27 Jul).
- A dev-environment "returned to sender" email reached a customer ticket (25 Aug).
- A comment didn't appear in #tech-cs (Mark, 6 Oct).
- The ERP-BOT tag flip-flopped (861bd10d → eb5d132d → d19e1f8b).
- One commit notes the bot was "not actually notifying" (98b69044).

### 4.11 Dashboards (internal)

**Users:** Founders, Ops, Accounts
**Problems found:**
- The SLA dashboard used 2 days instead of 3 and dropped custom orders (11 Aug).
- The Founder Dashboard was down (31 Aug).
- MCP returned non-JSON dates (3 Aug).
- The Shopify refund netting changed and a 7-day window was reverted (3c47a86f, 38565ad0).

That's 54 fix commits; `founder_dashboard.js` appears in 40 fix commits.

### 4.12 HRMS, tasks and SLA

**Users:** all staff, HR
**Problems found:**
- The weekend fix took two passes (086463a6 → 9d1d0ada).
- Deadlock (f082d17a).
- SLA comment spam on tasks.
- Timesheet bugs (1219205599514064, 1219149161944478).
- Flexible attendance has **no tests**.

---

## 5. Cross-cutting evidence

### 5.1 Fix churn by area (1 Jul – 23 Oct)

| Area | Fix commits / all commits |
|---|---|
| Lyfe Order core (form, workflow, routing) | 114 / 338 |
| Quotation / Sales / Invoice | 55 / 218 |
| ShipStation / Shopify sync | 54 / 112 (48%) |
| Founder / PnL / analysis dashboards | 54 / 241 |
| 1Click / US warehouse / BTB | 50 / 198 |
| HRMS / Tasks / SLA | 46 / 132 |
| BOM / Item / SKU / Kit | 42 / 148 |
| Tracking (17track, order_tracking) | 38 / 111 |
| Factory / print / export | 28 / 89 |
| Shipping rates / CJ | 25 / 108 |
| Slack | 19 / 54 |
| Reshipment | 15 / 77 |
| Return / RTO | 14 / 39 |
| MIFO | 10 / 28 |

### 5.2 Files most often touched by fixes

| File | Fix commits |
|---|---|
| `lyfe_order.py` | 89 |
| `patches.txt` | 63 |
| `lyfe_order.js` | 41 |
| `founder_dashboard.js` | 40 |
| `hooks.py` | 33 |
| `founder_dashboard.py` | 29 |
| `shipstation_orders.py` | 27 |
| `order_tracking.py` | 27 |
| `oneclick_api.py` | 22 |
| `public/js/quotation.js` | 17 |
| `quotation_analysis_dashboard.js` | 17 |
| `slack/tag_to_note.py` | 15 |
| `pnl_dashboard.py` | 15 |
| `order_analysis.py` | 15 |
| `integrations/shopify.py` | 14 |

### 5.3 Error messages written for developers, not users

There are about 650 `frappe.throw` calls in Python, 47 Python `msgprint` calls and 125 JS throw/msgprint calls. Examples:

| Location | Message | Problem |
|---|---|---|
| `lyfe_order.py:5376` | `route_plan must be "Force US" or "Force India".` | Shows the raw field name |
| `lyfe_order.py:2591` | "Could not acquire lock to assign internal_order_id." | Internal detail; the user has no action to take |
| `lyfe_order.py:3042` | "Failed to fetch order from Shopify. Check error logs for details." | CS can't open the Error Log |
| `lyfe_order.py:7793` | "No saved 1Click request payload found… cannot resend items." | Jargon |
| `lyfe_order.py:5396–5416` | "Factory Leg Destination set… from a Mixed order split — Force US…" | Jargon |
| `cj_api.py:486`, `:518` | `CJ API network error: {e}` / `CJ API error [{code}]: {desc}` | Raw vendor codes |
| `cj_api.py:453`, `oneclick_api.py:561` | `Could not parse log payload: {e}` | Raw exception |
| `oneclick_api.py:188`, `:437`, `:639` | `frappe.throw(str(exc))` | Raw exception |
| `lyfe_order_reshipment/oneclick.py:93`, `:143` | "No shipment items (CJ Shipment Items)…" / "Posting to 1Click failed: {exc[:300]}" | Jargon and raw exception |
| `shipstation_settings.py:589` | "Failed fetching ShipEngine carriers (status N). See Error Log. {e}" | Raw exception plus the Error Log |
| `bulk_transfer_batch.py:721` | "check 1Click Support Email in Oneclick Settings and the Error Log" | Tells the user to debug settings |

### 5.4 Test coverage

- **No tests at all:**
  - merge_order / Merge Suggestion
  - flexible attendance (`lh_hrms/attendance/*`)
  - CJ label API (`cj_api*.py`)
  - Lyfe Export Document
  - Order Visibility
  - Quotation Analysis
- **Stub tests only:**
  - `test_lyfe_order.py`, a 9-line stub covering 85 throws and the full workflow
  - `test_material_issue_for_order.py`
  - gate_pass, kit_planning, shipstation_settings
  - 28 doctype stubs in total
- **Well covered:** 1Click (scenarios and reliability), reshipment two-leg, returns, stock forecast, MCP, SLA.
- **Known failing:** `test_factory_to_us_order_leg` has 4 errors that predate the recent work.

### 5.5 Asana state (as of 23 Oct)

| | |
|---|---|
| Viral's open tasks | 61 |
| Overdue | **38** (the oldest from Mar–Apr: master data import, shipping logic engine, Metal Tubing, factory multi-user RBAC) |
| No due date | 12 (7 Forecast v2 steps, plus 1219444388063209, 1214422446941989, 1214422143814579, 1216250064107910, 1215396365043156) |
| Duplicates | 1213506263587573 = 1217389190181027 (push shipping rate to Shopify); 1213506263587597 overlaps 1219574218280557 (WhatsApp) |
| Status mismatch | 1219729341041359 (Lyfe PO): Status says Completed but the task is still open |
| PR / Commit link filled | about 2 of 16 typed bug fixes |
| ERP System Issue Management | 8 issues since 20 Jul, 7 resolved; Priority, Issue Type and Raised By are empty on nearly all |

---

## 6. Root causes

1. **No end-to-end check before release.** Fixes go out after a unit-level or happy-path check. The users' path (CS or Factory clicking through) isn't tested, so side effects reach users.
2. **The core files are too big and too coupled.** `lyfe_order.py` holds routing, workflow, export, 1Click and tracking hooks. A change in one area silently affects the others.
3. **Status is changed by several actors:** user actions, the workflow, the tracking jobs and the sync. Nothing guards against one overwriting another.
4. **Recovering from a stuck state needs a developer.** There's no role-based "move back" or override, so every edge case is a DM.
5. **Bug intake is spread across DMs, WhatsApp and Asana.** Recurrence isn't visible, users feel they must "prove" a bug, and Factory is left out.
6. **Error messages are written for developers.** Users can't act on them without escalating.
7. **Requirements get settled in production.** The 1Click rules ("open items 1–13") were worked out live, hence the four changes to the Create Load setting and then a revert.

---

## 7. Fix plan (ordered)

Dates assume the security audit (26–31 Oct) runs in parallel. Confirm the priority between the two with TJ.

| # | Workflow / item | Users | What changes | Done when | Due |
|---|---|---|---|---|---|
| 0 | **Release process** | All | Per-workflow test checklists (Quotation, Lyfe Order, MIFO, 1Click, Reshipment/Return). An end-to-end test report on the Asana task before every release (Srishti's ask). A regression test for every bug fix. PR / Commit link filled every time. | Checklist docs published in `LH Documentation/`; first report posted | 27 Oct |
| 0b | **Bug intake** | All | One route: the Asana form plus a "Report bug" button in the ERP (Arth's ask) that captures the doc, user, URL and screenshot. Set Priority, Issue Type and Raised By on every issue. A weekly 15-minute check-in with CS and Factory (Rohit via Srishti). | Button live; first check-in held | 30 Oct |
| 0c | **Asana clean-up** | — | Re-date or close the 38 overdue tasks, date the 12 undated ones, merge the duplicates, close 1219729341041359 | No overdue task without a new date | 26 Oct |
| 1 | **Quotation → payment link** | CS, TechCS, Ops | RCA for the missing payment links and card option. An alert when an approved quote has no payment link. Fix the cancel/timesheet blocker and decouple drawing upload from the Timesheet. Fix rejection after internal review. Tolerance for Partially Paid. Automated tests for the quote → approve → link → pay path. Close 1217715464478670. | Walkthrough signed off by Gau or Cherry; tests green | 30 Oct |
| 2 | **Lyfe Order status recovery** | CS, Ops, Factory | A role-based "Move back / override status" action with a mandatory reason and logging. Fix the known dead-ends (Ready for Dispatch after a US-warehouse assign, Awaiting Engineer Check, the Actions button disappearing, Route via US resetting the workflow). Show every Ready for Dispatch gate as one checklist, not one error per attempt. | Zero "please change the status" DMs for 2 weeks | 4 Nov |
| 3 | **Error messages** | All | Rewrite the top ~25 messages (Lyfe Order, CJ, 1Click, ShipStation, BTB) in plain language: what happened, what to do, who to contact. Never show a raw exception or "see the Error Log" to the user; log it and give a reference instead. | Message list reviewed with CS | 6 Nov |
| 4 | **Shopify / ShipStation sync** | CS | Show on the order *why* an edit wasn't applied (On Hold / New / merged). RCA for the missing refund notifications. Item Group defaulting to cut manual entry. Tests for merge_order. | Cherry confirms edits behave as she expects | 7 Nov |
| 5 | **Address, 1Click and US warehouse** | Ops, Logistics, Factory | An address check before posting to 1Click or forwarding (the 1Click address vs the customer's). Confirm the SKU auto-register shipped in September has stopped the "SKU does not exist" alerts. Apply the "onhand" setting on production. Check that a mixed-order Load contains the right items (to prevent a repeat of the $374.90 loss). Plain messages in place of the raw exceptions. | No wrong-address or SKU-missing incidents for 2 weeks | 11 Nov |
| 6 | **Factory pending list and MIFO** | Factory | Walk through the pending list with Rohit (assignment, drawings, visibility of reshipment parents). Settle the gift rules on MIFO and reshipment. | Rohit signs off | 13 Nov |
| 7 | **Tracking and status integrity** | CS | Stop the tracking and sync jobs from overwriting a status a user has set (Return, Completed, reshipment parent). Restore the Meest tracking entry. Real tests for `test_lyfe_order.py` covering the workflow transitions. | No silent status flips; test suite > stub | 18 Nov |
| 8 | **CJ labels and export docs** | Factory | Plain messages for CJ errors, basic tests for `cj_api*.py`, and settle the print formats | Factory confirms | 20 Nov |

Later, not in this cycle: split `lyfe_order.py` by concern (routing, export, 1Click, tracking) so changes stop affecting unrelated areas. Plan this after item 7, once tests exist.

---

## 8. Asana structure

- **Project:** ERPNext (gid 1211921346208625). It's the only project with the PR / Commit link, Task Type, area (Project) and Status fields.
- **New section:** "ERP Workflow Review – Oct 2026".
- **Tasks:** one parent per row in section 7, with dated subtasks. On each one set:
  - Task Type (Bug fix / Improvement)
  - Project area (Orders, Quotation & COGS, Factory & Reshipment, One-Click Logistics / US Warehouse, Integrations, Shipping & Tracking)
  - Status = ToDo
  - a due date
- **Naming:** `<Area>: <what changes>`, with the reference ID in brackets where there is one.
- **Description template:** Problem → Users affected → Target → How it's checked (walkthrough with whom + tests) → Already done (commits).
- **On completion:** an "After Fix" comment (root cause, scope, files, commit, test result), the PR / Commit link filled, and a test report posted.
- **Visibility for reporters:** also add CS and Factory bugs to ERP System Issue Management so whoever raised them can follow progress.

---

## 9. Next actions before Monday 26 Oct, 7 pm IST

1. **Today (23 Oct) by 7 pm IST:** reply to TJ's security posture request (a separate deadline).
2. **Today:** post a holding update in the thread so progress is visible:
   > TJ — understood, and you're right that nothing visible came out of my reply. Review is underway: I've gone through the last 3 months of reported issues and fix history across Quotation, Lyfe Order, Shopify sync, 1Click/US warehouse, Factory pending list and tracking. This weekend and Monday morning I'm doing short walkthroughs with Cherry, Gau, Srishti and Rohit. Full review + dated plan on Asana here by Mon 7 pm.
3. **Weekend / Mon morning:** 15-minute walkthroughs with Cherry, Gau, Srishti and Rohit (via Srishti if needed). Ask each:
   - Where do you get stuck?
   - Which steps do you repeat?
   - What would you remove?
   - What errors keep coming back?

   Add what they say to section 4, which today is based only on the research.
4. **Mon:** the Asana clean-up (item 0c), then create the section and tasks from section 7.
5. **Mon by 7 pm IST:** post the summary in the thread, with a link to this document and the Asana section:
   - workflows reviewed and their users
   - top findings
   - the ordered plan with dates

---

## 10. Caveats

- Commit counts come from keyword and path heuristics (about ±10–15%). One commit can count in several areas.
- Slack counts are approximate, and #cs and #tech-cs were not readable, so CS issues are probably undercounted.
- Some DM quotes are cited by date only; permalinks weren't available.
- The git history was read on the local dev box. Production state (for example the 1Click "onhand" setting) should be checked on prod.
