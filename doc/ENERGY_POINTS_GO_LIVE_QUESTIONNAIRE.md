# Energy Points — Go-Live Questionnaire for HR / Founders

**Purpose:** The Energy Points system is fully built and tested. It is currently switched OFF because the business decisions and master data below have not been provided yet. Once HR fills in this document, the system can go live the same day — no development work is needed for any of it.

**Who fills this in:** HR, with Founder sign-off on Sections B, C and D.
**Return to:** Viral
**Companion reading (plain-language, no tech):** [ENERGY_POINTS_HR_GUIDE.md](ENERGY_POINTS_HR_GUIDE.md) — explains what each feature does before you decide its numbers.

---

## How to use this document

- Every blank `_______` is a decision only HR/Founders can make.
- Where a **suggested value** is shown, that is what the system currently uses as a placeholder — you can accept it by writing "OK" or replace it with your own number.
- Section F is not questions — it is a menu of what else is *possible*, so you get ideas for rules we can add later. Nothing in Section F is required for go-live.

---

## Section A — Master Data HR Must Provide (required before go-live)

The engine reads real employee records. If these are wrong or missing, points go to the wrong person or nobody.

### A1. Employee list with system accounts

For **every employee** who should earn points, confirm:

| # | What we need | Why it matters |
|---|---|---|
| 1 | Full employee list (CS, Operations, Factory, Engineering) with their **ERPNext user email** | Points are awarded to the user account; an employee with no linked user account earns nothing |
| 2 | Each employee's **direct manager** (who they report to) | Manager Points button, External Learning approval, and dashboards all follow this line — if it's blank or wrong, the manager can't award points or approve learning |
| 3 | Each employee's **Department** and **Designation (role)** | Appraisal templates and track weights are set per role |
| 4 | Any employees to **exclude** from the points system entirely (e.g. contractors, founders themselves) | So the leaderboard only shows people it should |

> **Format:** a simple sheet — Name · Email · Department · Designation · Reports To · Include in points? (Y/N)

### A2. Skills list per role

External Learning approval can automatically bump an employee's skill rating — but only if a skills list exists per employee.

- [ ] Provide the list of **skills per role** (e.g. CS: customer communication, ERP order handling, escalation handling…)
- [ ] Or confirm: "skip skill tracking for now, points only" — this is fine; learning points still work without it

### A3. Team scope confirmation

The BRD sets these boundaries — confirm they still hold:

- [ ] Daily Work Log (and its compliance point) applies to **CS + Operations only** — Factory excluded. Confirm: _______
- [ ] Factory earns points **only** through PM tasks / SLA tickets (no attendance or daily-log points). Confirm: _______

---

## Section B — Point Values (Founder sign-off required)

These numbers exist in the system today as **placeholders**. Founders should confirm or change each one. Every value can be changed later by HR directly in the system — no developer needed — so don't agonise: pick numbers that feel roughly fair and tune after a month of real data.

| # | Event | Track | Current placeholder | Your value |
|---|---|---|---|---|
| 1 | Task completed (PM board) | Performance | **+5** (live) | _______ |
| 2 | Quotation submitted | Performance | **+2** (live) | _______ |
| 3 | SLA deadline missed (breach) | Performance | **−2** | _______ |
| 4 | Escalated issue resolved | Performance | **+2** | _______ |
| 5 | Factory SLA met (on-time dispatch / review) | Performance | **+2** | _______ |
| 6 | Daily Work Log filed on time | Performance | **+1** | _______ |
| 7 | External course approved | Growth | **+5** | _______ |
| 8 | Manager award — monthly cap per employee | Manager | **50 / month** | _______ |

### B1. Should the SLA breach penalty be the same for every kind of breach?

The system watches many kinds of deadlines (stuck orders, promised dispatch, quotation aging, awaiting shipping, pending approvals…). Today, **every breach costs the same** (−2).

- [ ] Same penalty for all breach types (simplest — recommended to start)
- [ ] Different penalties by severity — if so, tell us which breach types matter more: _______

> Note: shipment delays caused by the **carrier** (courier company) are already excluded — no employee is penalised for those.

### B2. Who is accountable for each SLA stage?

When a deadline is missed, the penalty must land on a **person**. Today the system falls back to shared mailboxes (support@…), which makes penalties meaningless. For each stage, name the responsible person or role:

| Stage | Responsible person / role |
|---|---|
| Order stuck before Factory Assignment | _______ |
| Factory not dispatching by promised date | _______ |
| Custom order not reaching Internal Review | _______ |
| Internal Review not actioned (CS/Engineer) | _______ |
| Order awaiting tracking number | _______ |
| Shipped but never marked Completed | _______ |
| Quotation open too long / drawing overdue | _______ |

---

## Section C — Appraisal Weights per Role (Founder sign-off required)

At appraisal time, each person's points from the three tracks are combined into one score using percentages set **per role**. The three numbers must add up to exactly 100.

| Role | Performance % | Growth % | Manager % | Total |
|---|---|---|---|---|
| Customer Service | _______ | _______ | _______ | must = 100 |
| Operations | _______ | _______ | _______ | must = 100 |
| Factory | _______ | _______ | _______ | must = 100 |
| Engineer | _______ | _______ | _______ | must = 100 |

*Example from the BRD: CS = 60% Performance / 20% Growth / 20% Manager.*

Also confirm:

- [ ] **Appraisal cycle frequency:** Quarterly / Half-yearly / Annual → _______
- [ ] **First cycle start date:** _______

---

## Section D — Fair-Play Settings (confirm the defaults)

These protections are already built. Just confirm you're happy with each:

| # | Rule (already enforced) | Confirm OK? |
|---|---|---|
| 1 | Only one Daily Work Log per person per day — the compliance point can never be earned twice in a day | _______ |
| 2 | If a deadline is missed after an earlier "met" credit was given for the same case, the credit is automatically taken back before the penalty applies | _______ |
| 3 | A manager can give at most the monthly cap (Section B #8) to any one employee per calendar month | _______ |
| 4 | External Learning points are awarded once per submission, no matter how many times approval is toggled | _______ |
| 5 | Managers can only award points to their **own direct reports** | _______ |

**Peer appreciation allowance** — employees can also gift each other small points from a personal allowance that refills periodically:

- [ ] Refill period: Daily / Weekly / **Monthly (suggested)** → _______

**Any additional caps you want?** (e.g. a maximum number of task-completion points per person per day) → _______

---

## Section E — Housekeeping decision (quick yes/no)

The system shipped with generic example rules that award points for things unrelated to employee performance (creating an Item, a Lead, a Purchase Order, etc.). We recommend **disabling all of them** so the leaderboard only reflects rules this company designed.

- [ ] Yes, disable all generic example rules (recommended): _______

---

## Section F — Idea Menu: What Kinds of Rules CAN We Configure?

Nothing here is required — this is to show what's possible, so HR can propose rules that fit how the teams actually work. A rule can be added by HR in the system in minutes, with:

- **Any document + event** — points when a document is created, saved with a certain status, submitted, or cancelled (cancel automatically takes the points back)
- **A condition** — e.g. only when order value > X, only for custom orders, only when a field equals a value
- **Who gets the points** — the document's owner, the person who completed it, or any person named in a field on the document
- **A multiplier** — points can scale with a number on the document (e.g. 1 point per item line)
- **One-time-only** — a rule can be set to never award twice for the same document

### Example rules other teams like — tick any you want us to set up:

**Sales / CS**
- [ ] Quotation converted to a paid order (bigger reward than just submitting it)
- [ ] Custom order approved by the customer **first time** (zero rejections)
- [ ] Order photos uploaded within X hours of request
- [ ] Customer contact request handled within same day

**Factory / Ops**
- [ ] Gate Pass submitted on the promised dispatch date
- [ ] Zero breakage recorded on an order
- [ ] Material issued same day as factory assignment

**Everyone**
- [ ] Milestone badges: 100 / 500 / 1000 lifetime points
- [ ] "Streak" recognition: daily log filed on time N days in a row (needs a small custom check — flag if wanted)

**Learning (activates when the Learning module is installed)**
- SOP course completed → Growth points (value: _______)
- Quiz passed with full marks → badge
- Entire onboarding learning path finished → badge
- [ ] Which SOPs form the first mandatory learning path? _______
- [ ] Quiz pass mark: _______ %

---

## Sign-off

| Section | Signed off by | Date |
|---|---|---|
| A — Master data delivered | | |
| B — Point values | | |
| C — Appraisal weights & cycle | | |
| D — Fair-play settings | | |
| E — Disable generic rules | | |

Once all five rows are signed, go-live is: flip one switch (Energy Point Settings → Enabled) + enter the confirmed numbers. Same-day.

---

*Document owner: Viral · Created 2026-07-03*
*Source: LH-BRD-DEV-006 §5, §12 · HR_IMPLEMENTATION_PLAN.md Step 2 · ENERGY_POINTS_HR_GUIDE.md*
