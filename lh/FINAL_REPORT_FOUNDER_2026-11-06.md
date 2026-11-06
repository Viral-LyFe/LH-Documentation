# Security audit of the Lyfe Hardware ERP (`lh` app): final report

| | |
|---|---|
| **For** | Founder and management review |
| **Prepared by** | Viral Kansodiya, with Claude (AI assistant) |
| **Date** | Friday 6 November 2026 |
| **Period covered** | 21 October – 6 November 2026 (206 code changes reviewed) |
| **Code version** | `apps/lh`, branch `prod`. Last application change `415c4583` (pushed). One later commit, `3c3f5b5e`, changes tests only and is not pushed yet. Changes up to 5 Nov are deployed; the 6 Nov changes deploy on 7 Nov |
| **Task** | Asana: "Security audit fixes: lh app (26–31 Oct)" |
| **Supporting files** | `LH Documentation/pending_final.md` (item-by-item tracker), `LH Documentation/lh/REPORT.md` (technical findings and the 6 Nov re-check), `LH Documentation/lh/SCHEDULE.md` (plan and tracker) |

---

## 1. Summary in plain language

**What this was.** On 23 October an automated security review of our ERP custom code found that many actions could be performed by *any logged-in user*, including external users such as the Shipping Forwarder and ordinary employees. Examples: recording a fake payment so goods ship unpaid, buying shipping labels, pointing our Shopify order feed at someone else's server, reading customers' names and addresses in bulk, and reading cost-of-goods and profit figures that only Factory and managers should see. We also found secrets (API keys) that had leaked into the code history.

**What we did.** Between 23 October and 6 November we fixed every serious finding, added automatic safeguards so the same mistakes cannot return unnoticed, and wrote tests for the fixes.

**Result.**

| Measure | Result |
|---|---|
| Critical findings (could cause financial loss or data theft) | **6 of 6 fixed** |
| High findings | **13 of 13 fixed or closed by a recorded decision** |
| Medium and Low findings | **All fixed or closed by a recorded decision, except a few small items listed in section 7 (print-format escaping, the last GET links, and running the invoice-PDF patch)** |
| Work items in the tracker | **56 rows: 37 fully complete (66%), 14 complete in code and waiting for a check on a copy of the live site (25%), 2 partly done (4%), 3 verification-only rows not started (5%)** |
| Code work complete | **about 91%** (51 of 56 rows); the rest is two small items plus the checks |
| Checks that need the real site | 5 of 20 done; **15 are intentionally pending** (section 7) |
| Leaked secrets | All of them were rotated (replaced). The old values still exist in the code history, which cannot be rewritten, but they no longer work |
| New protections | An automatic check now blocks any new action that has no permission check, and a secret scanner blocks passwords and keys from being committed |

**What is still open, and why.** Everything still open is **pending on purpose**: a copy of the live (prod) site will be created on Frappe Cloud, and the remaining checks and any fixes will be done there rather than on the live site. Details are in section 7.

**Two things to know.**
1. **Deployment status.** Every change made up to 5 November is already deployed to the live site. The changes made on 6 November (listed below) are on the production branch and will be deployed on 7 November. They are: the dashboard and MCP checks (3.6, 4.8), Slack unmapped-user naming (4.9), removal of two unused actions (5.8), the whole "every outside-callable action needs a permission check" set (2.1: forecast roles, S3 helpers, tracking buttons, record-access checks, markers, the CJ prefill), the H4 write checks, the invoice-PDF patch (3.5) and the two commit-time safeguards (permission check and secret scanner). The checks planned on the Frappe Cloud copy therefore verify code that is already live (up to 5 Nov) and code that goes live tomorrow.
2. **Some people will notice changes.** Several buttons and permissions were tightened or removed (section 5). Viral Kansodiya, as business owner, approved each of these; the full list is in section 5 so support staff are not surprised.


### 1.1 What the audit reported, and what we completed

The 23 October report listed the items below (its own tables group them this way; counting every sub-case it said about 15 High, 20 Medium and 25 Low). **Every one of the 50 items was acted on; none was ignored.**

| Category | Reported (23 Oct) | Fully fixed | Fixed, with one part left (decided or pending) | Closed by decision only (no change) | Not done |
|---|---|---|---|---|---|
| Steps outside the code (rotate keys, check secrets) | 6 | **6** | 0 | 0 | 0 |
| **Critical** | 6 | **6** | 0 | 0 | 0 |
| **High** | 13 | **12** | 0 | 1 (H8) | 0 |
| **Medium** | 12 | **6** | 6 | 0 | 0 |
| **Low** | 7 | **3** | 2 | 2 | 0 |
| Systemic fixes recommended | 6 | **4** | 1 | 1 | 0 |
| **Total** | **50** | **37 (74%)** | **9 (18%)** | **4 (8%)** | **0** |

How to read it:
- **Fully fixed** = changed in the code, tested, pushed.
- **Fixed, with one part left** = the main problem is fixed; a smaller part was either decided to stay (for example old quotation links) or waits for the Frappe Cloud checks (for example the old invoice PDFs and the last print-format fields). The six Medium items in this column are the approval link (old quotations decided), invoice PDFs (patch for old files to run), task mass-assign and column terminal, manager points and HR scope, Slack (unmapped users decided), and the `_assign` / learning-record / visibility rules. The two Low items are print-format and dashboard escaping, and the last GET links. The one systemic item is POST-only for state-changing actions.
- **Closed by decision only** = we tested it and the business chose to leave it: H8 (records can be opened by name), the plain-text API-key fields, the bank-transfer link re-sending its email, and the recommendation to add record-level permission hooks (same as H8).

Where the items are, finding by finding, is in section 4.

### 1.2 What we found while fixing (not in the 23 October report)

Fixing the reported items turned up more. These were fixed or decided too.

| What we found | What we did | Status |
|---|---|---|
| About 25 more outside-callable functions with no permission check, or with only a "read" check where "write" was needed (forecast and stuck-order data, tracking buttons, payment-stage summary, shipping-rate calculators, S3 helpers, the reshipment label prefill, kanban counts and more). Found when we built the automatic check on 6 Nov | Each gated by role or by access to the record, or removed as an outside action | **Fixed** (`a07c0ded` `20b0c718` `d1c48af5` `49ee0c0e` `e73f7495` `89370477`) |
| Three functions that checked only "read" but still changed data (email to 1Click, validation feedback, load status) | Now need write | **Fixed** (`89370477`) |
| "Put On Hold" had no real check (any user with order access could use it) | Customer Service, System Manager and Super Admin only, and not on shipped or completed orders | **Fixed** (`6c04d319`) |
| Many of our own role checks blocked System Manager and Super Admin, against the rule that they are never blocked | Every such check now allows both | **Fixed** (`cee31478` `0238ab12`) |
| The audit said `get_quotation_detail` had no check; it had a weak one, and it also returned item cost to every quotation reader | Needs read on that quotation; cost shown only to Factory, System Manager, Super Admin | **Fixed** (`cee31478` `20aed3db`) |
| Two typos in one of our own fixes (`ignore_permission`) broke the ShipStation re-route on every order sync | Corrected the same day | **Fixed** (`a553f6ef`) |
| Engineers saw a "No permission for Oneclick Settings" popup opening a Lyfe Order | The flag is now sent with the page | **Fixed** (`f970a196`) |
| New public item-image endpoint added by the team on 29 Oct | Reviewed (low risk); limited to Shopify-listed items, rate-limited, clean errors | **Fixed** (`45e77120` `ba4d1987`) |
| New COGS-based price API (2 Nov) lets anyone with quotation access work out item costs | Reviewed | **Decision: left as is** (4 Nov) |
| New "Create Lyfe Order from Replenishment" action (2 Nov) | Reviewed: Factory, System Manager, Super Admin only, POST only | **Checked, no change needed** |
| 487 other public PDFs attached to Lyfe Orders on the local copy (shipping labels and similar) | Not part of any reported item | **Open: to look at on the Frappe Cloud copy** |

**Scorecard.** Reported items: 50, all acted on (74% fully fixed, 18% fixed with a part left, 8% closed by decision). Extra items found while fixing: 11, of which 8 fixed, 1 decided, 1 checked with no change needed and 1 still open.

---

---

## 2. How the work was done

1. **Audit (23 Oct).** A static code review (no code was run) of 1,240 Python files (about 215,000 lines): 521 actions callable from outside, 16 public endpoints, about 1,040 database queries, print formats, templates, file handling, and secrets in the code and its history. Findings are in `REPORT.md`.
2. **Plan (`SCHEDULE.md`).** Fixes were ordered by risk: Monday Criticals, Tuesday–Thursday Highs, Friday Mediums and Lows, Saturday verify and close.
3. **Fix and test.** Each fix was reproduced first where possible, changed, and covered by a test that fails on the old code and passes on the new. Decisions that change how people work were taken by the business owner (Viral) item by item.
4. **Tracking.** Every fix is a separate, described commit, linked on the Asana task. A daily work log and a status tracker were kept.
5. **Re-check (6 Nov, task 6.1).** At the final code version we re-ran the automated checks, re-verified the Critical findings in the code, reviewed what had changed since the audit, and ran the test suites (section 8).

**Scope of the re-check.** It was a re-check against the original findings plus automated scans of everything that changed. It was **not** a new line-by-line audit of the whole application, and nothing was tested on the live site.

---

## 3. Everything that changed between 21 October and 6 November

We reviewed **all 206 code changes** (commits) made in this period.

| Group | Changes | What it is |
|---|---|---|
| **Security audit work** | **80** | Fixes, safeguards, removals, hotfixes and reviews that belong to the audit (listed in Appendix A) |
| **Other business work** (not part of the audit) | **126** | New features and bug fixes by the team in the same period (summarised in Appendix B) |

We checked the 126 non-audit changes with the automated checks, because new features can reopen old problems: **no new public endpoint, no new action without a permission check, no new injectable database query, and no new secret** was introduced (details in section 6).

---

## 4. Findings and how each was resolved

Status key: **Fixed** = changed in the code, tested, pushed. **Verify on copy** = fixed and tested locally; a check on the Frappe Cloud copy is still to do. **Decision** = the business chose to leave it as it is; the risk is accepted and recorded.

### 4.1 Critical (6)

| # | Problem in plain terms | Fix | Commit(s) | Status |
|---|---|---|---|---|
| C1 | Secret keys for 1Click and Shopify had leaked into the code and its history (and one was served publicly) | Keys revoked and replaced; documents holding them deleted or redacted; keys no longer in the code | `6f8c2ff6` `f9f83402` `8e7f2410` | **Fixed.** Old values still in history but dead |
| C2 | The ShipStation webhook (an inbound order feed) accepted anyone when no secret was set, and could create or overwrite orders | The endpoint was deleted; it was never used | `1e1be4a7` | **Fixed** |
| C3 | Any logged-in user could record a fake payment and move a quotation to Won, so goods shipped unpaid | Payment recording is no longer callable from outside; completing a draft order is limited to Customer Service and System Manager, and only from the right states | `9368226f` | **Fixed** (also tested in the browser) |
| C4 | Any user could redirect our Shopify order, payment and refund feeds to their own server (a customer-data leak) or delete them | Registering and removing the feeds is System Manager / Super Admin only | `753af930` | **Fixed** |
| C5 | Any user could inject a fake ShipStation order list and start 1Click fulfilment | The function is no longer callable from outside | `6a977c35` | **Fixed** |
| C6 | Any user could generate billable CJ shipping labels to any address | Limited to Factory, Founder, Super Admin, System Manager (also the reshipment label and its prefill step) | `b63895e8` `e73f7495` | **Fixed** |

### 4.2 High (13)

| # | Problem in plain terms | Fix | Commit(s) | Status |
|---|---|---|---|---|
| H1 | Shopify and 17Track webhooks accepted forged messages when no secret was set (fake "paid" or "delivered") | They now reject every message when the secret is missing; the expected signature is no longer logged | `1e1be4a7` | **Fixed** |
| H2 | The file system could be made to fetch any web address using the user's login cookie (a server-side request forgery); S3 files could be opened by anyone who had the link | The server no longer fetches remote addresses or sends the cookie; S3 files are now served only if the user may read that file | `1f073e60` `a0611537` | **Fixed** (S3 part needs a restart on deploy; verify on copy) |
| H3 | A public endpoint let anyone trigger 200 tracking look-ups per call (cost and slowdown) | Removed as an endpoint; only the background job runs it, one at a time | `efc88a26` | **Fixed** |
| H4 | About 40 order and accounting actions had no permission check (revert state, split, assign to factory, post to 1Click, create Purchase Orders and Sales Invoices, gate passes, ...) | 124 functions reviewed and each gated by role or record access, or removed; later 3 more that checked only "read" but still wrote were tightened; the new automatic check stops new ones | `df92578f` `f0ab2241` `22037fda` `da55c582` `10459745` `89370477` and others | **Fixed** (verify on copy) |
| H5 | A user could write any field of a Lyfe Order through two helper actions | Both are no longer callable from outside (nothing used them) | `1ffe7b1e` | **Fixed** |
| H6 | Profit and cost figures leaked: P&L dashboard, COGS exports, quotation detail | P&L follows the page roles (System Manager, Super Admin); COGS exports for Factory, System Manager, Super Admin; item cost is returned only to those roles | `8d79ed5f` `cee31478` `20aed3db` | **Fixed** (verify on copy as Customer Service) |
| H7 | Bulk customer details (names, addresses, tracking) readable by anyone via dashboards, exports and the pick-pack list | Each now needs read access to the order; status-overview, returns and turnaround dashboards check the caller | `61cc5da0` `5f4ffb09` `8f956d99` `07c68446` | **Fixed** (verify on copy) |
| H8 | The rules that hide Tasks, Projects, Quotations, CRM Leads and Deals only filter lists; a record can still be opened by its name | Tested; **decision to leave as is** (30 Oct) | none | **Decision** |
| H9 | Any user could read an employee's hidden appraisal score and force a save | No longer callable from outside | `c9a49cf6` | **Fixed** |
| H10 | A user could set any Gorgias ticket id on their task and read or reply to that customer's conversation | The ticket id field is now visible and editable only to Customer Service, System Manager, Super Admin (a data patch applies it) | `b517ae90` | **Fixed** (needs migrate; verify on copy) |
| H11 | A project-board member could pull tasks from projects they have no access to, by guessing numbers | Requires access to the task's home project | `7fd178fc` | **Fixed** |
| H12 | Anyone could approve a Senior Review or clear a quotation's drawing | Requires write access to that quotation. Engineers completing a review directly is accepted | `f7a9f6b8` `016295d4` | **Fixed** / **Decision** on the Engineer review |
| H13 | Item and BOM (bill of materials) records could be created or overwritten by anyone | "Parse BOM from Drawing" limited to Engineer, Factory, System Manager, Super Admin; created with the user's own permissions | `369d6dcf` `49f6f9c2` | **Fixed** |

### 4.3 Medium (12 groups)

| Finding | Resolution | Commit(s) | Status |
|---|---|---|---|
| Quotation approval link used the database password as its key, never expired, could approve from any state | Links are signed with a dedicated secret (`approval_link_secret`, no fallback) and expire after 45 days. **Existing quotations created before 30 Oct keep working with the old key** (decision, section 7); the GET-changes-state behaviour also stays (decision) | `be7e3c9a` | **Fixed for new quotations / Decision for old ones** |
| Automation rules could run arbitrary code with full access | The `frappe` module was removed from the rule evaluator | `f30f3d9f` | **Fixed** |
| Arbitrary S3 files readable by key | Access checks the file record; a print-template helper and a file hook are no longer web endpoints | `a0611537` `20b0c718` | **Fixed** (print formats and upload to verify on copy) |
| Sales Invoice PDFs saved as public files | New PDFs are private. A patch makes the old ones private (78 on the local copy), with a rollback | `78dbd84a` `415c4583` | **Fixed for new; patch for old committed, runs on migrate** |
| MCP (the AI connector): hidden cost fields could be inferred through filters; some tools had no role gate | Filter guard now catches every form; the status-overview tools need Lyfe Order read | `a2504e19` `07c68446` | **Fixed** |
| Status overview, returns and turnaround dashboards had no backend role check | Each now checks the caller | `07c68446` | **Fixed** |
| Task field copying, bulk-complete column, manager points, HR dashboard scope | Bulk-complete and points gated; the rest left by decision | `10459745` | **Fixed / Decision** |
| Slack: disabled users could still approve; any workspace member could look up orders; unmapped users wrote as Administrator | Disabled users blocked in every Slack flow. Unmapped users still write as Administrator (decision) but their name now appears in the text | `c6ef02d2` `cc108a9b` | **Fixed / Decision** |
| `_assign` row rules matched substrings; External Learning Record list; order-visibility default | Exact match fixed; the list rule needed no change; default left by decision | `8899faf6` `96df0650` | **Fixed / Decision** |
| `backfill_item_group` could write the whole table through a link | Removed (the button and the endpoint) | `4b1d0824` | **Fixed** |
| Hardcoded Slack webhook in a data patch | Removed and the webhook replaced | `026bb5c5` | **Fixed** |
| AWS key had been stored in readable form | Stored encrypted; the key was rotated on 2 Nov | `0e003915` | **Fixed** |

### 4.4 Low (7 groups)

| Finding | Resolution | Status |
|---|---|---|
| `ss_api_key` and Slack webhook fields stored as normal text | Decision: no change (they do not reveal confidential data) | **Decision** |
| Tokens and signatures written to the Error Log | Removed from the two places the audit named (Shopify signature check, token rotation) | **Fixed** |
| Customer text shown without escaping in print formats and dashboard scripts | Emails and comment mentions are escaped. **Print-format address fields and dashboard scripts remain** (section 7) | **Partly fixed** |
| The Slack install page printed an error text without escaping | Escaped | **Fixed** |
| The bank-transfer tracking link re-sends its email on every click | Decision: left as is | **Decision** |
| State-changing actions could be triggered by a crafted link (not POST-only) | 24 + 67 actions are now POST-only and 19 were removed; a few GET links stay on purpose | **Mostly fixed** |
| Link-search queries trusted a caller-supplied document type | Now use the correct type | **Fixed** |

### 4.5 Safeguards added so the problems do not come back

| Safeguard | What it does | Commit |
|---|---|---|
| **Whitelist authorization check** (`scripts/check_whitelist_authz.py`) | Reads all 433 outside-callable actions and fails a commit if one has no permission check or reasoned marker. Today 433 of 433 pass | `a3cadb7f` |
| **Markers on intentional public endpoints** | Each of the 15 public endpoints states what protects it (signature, token, secret) or why it exposes nothing | `c98db616` |
| **Secret scanner (gitleaks)** | Blocks commits containing keys, tokens or webhook addresses. Scanned all 2,750 commits and the current code: no new secrets | `9cf8121a` |
| **Rules for developers** (`CLAUDE.md`) | Written rules: every outside-callable action needs a permission check; System Manager and Super Admin are never blocked; secrets never go in code | `a3cadb7f` `9cf8121a` |
| **Tests** | New test modules for each fix (section 8) | various |

**The audit's six systemic recommendations:**

| Recommendation (23 Oct) | Result |
|---|---|
| 1. A check that fails when an outside-callable action has no permission check | **Done** (the whitelist authorization check, a pre-commit hook and a test) |
| 2. One webhook-verification routine that rejects when the secret is missing | **Done** (each receiver rejects a missing secret; the ShipStation one was deleted) |
| 3. Record-level permission hooks for Task, Project, Quotation, CRM Lead and Deal | **Decision: not added** (H8) |
| 4. POST-only for state-changing actions | **Mostly done** (the rest is pending) |
| 5. Avoid `ignore_permissions` in code an action reaches, unless the role was checked | **Done where reached** (removed from the gated order, Gate Pass, Purchase Order, Item and BOM paths) |
| 6. Secrets hygiene: nothing in docs or patches, pre-commit scanner | **Done** (documents deleted, patch cleaned, gitleaks hook) |

---

## 5. What users will notice (behaviour changes)

Each change below was approved by the business owner. This list is for support staff.

| Area | Change |
|---|---|
| **Payments and Shopify** | Recording a payment by hand is no longer possible outside the approved flow; completing a Shopify draft order is for Customer Service and System Manager only. Webhook registration is System Manager only |
| **Shipping labels** | CJ labels (including reshipments and the reshipment prefill) are for Factory, Founder, Super Admin, System Manager only |
| **Order actions** | 13 order actions (assign to factory, split, create gate pass, create load, purchase order, ...) need Factory, Customer Service, System Manager or Super Admin. Customer Service and Super Admin can only create a Gate Pass or Purchase Order if that permission has been granted in Role Permission Manager (reported done on prod, not checked from here) |
| **Track Order / Track Reship / Track Return** | Need permission to edit the Lyfe Order |
| **Put On Hold** | Customer Service, System Manager, Super Admin only, and not once the order is shipped or completed |
| **Costs and profit** | P&L: System Manager and Super Admin. Item cost of goods: Factory, System Manager, Super Admin (other roles see no cost field at all) |
| **Files** | Attachments on S3 open only for people allowed to read the file. New invoice PDFs are private; old ones become private when the patch runs |
| **Removed buttons / actions** | "Sync Item Groups", "Re-run Validation", "Skip Image Upload"; the Fetch Shopify Details and backfill helpers (nothing used them) |
| **Quotation links** | New quotation emails use the new signing key, and approval links expire after 45 days. Old quotations keep working as before |
| **Slack** | A disabled ERP user can no longer approve, send back, look up orders or add notes from Slack. Notes from Slack users with no ERP account show their name in the text |
| **Public item image link** | Serves only items listed on Shopify, is rate-limited, and returns clear errors |
| **MCP (AI connector)** | Status-overview tools need Lyfe Order read; hidden-field filtering is blocked in every form; forecast and stuck-order data need Factory, Founder, Founder-AI, Super Admin or System Manager |
| **Gorgias ticket id on a Task** | Visible and editable only by Customer Service, System Manager, Super Admin |

---

## 6. Re-check results at the final code version (6 Nov)

| Check | Result |
|---|---|
| Outside-callable actions without a permission check or marker | **0 of 433** (was about 205 of 482 in a one-off scan on 29 Oct) |
| Public (guest) endpoints | 15, each verifies a signature, token or secret or is deliberately public; no new ones since 23 Oct |
| Database queries built from text | 181 (180 at the audit); 5 changed, all safe (internal queries, integer limits, or a search field that is cleaned by the framework). **No injectable query found** |
| Secrets (gitleaks, 2,750 commits and current code) | 9 findings, all in old commits and all rotated; **0 in the current code** except one false alarm (a field name) |
| Critical findings re-verified in the code | C2–C6 and H1 read directly: all hold |
| Dependencies (this development copy) | Frappe 15.124.0, ERPNext 15.124.1, HRMS 15.64.6, CRM 1.89.1 (all up to date for their line). **LMS 2.45.2** (2 critical advisories) and **Insights 2.2.14** are unchanged. Live-site versions are not checked |

---

## 7. What is still pending, and why (intentional)

**Decision (Viral, 6 Nov):** everything below stays pending until a copy of the live site exists on Frappe Cloud. The checks and any fixes will be done there.

| Item | What it is | Risk while pending |
|---|---|---|
| **Retest of every Critical and High with the two test users (6.2)** | Confirms each fix on the real site and data | Medium: fixes are tested locally and by script, not on live data |
| **Live-site smoke tests (3.8, 5.10)** | Shopify payment to Won, CJ label, ShipStation sync, 1Click posting, dashboards, exports, S3 upload, print formats with signatures | Medium: a role change could block a legitimate user; the 15 pending checks are listed in `pending_final.md` |
| **Invoice PDF patch (3.5)** | Make the old public invoice PDFs private (runs with `bench migrate` on the 7 Nov deploy; has a rollback command) | Low: the file link contains a hard-to-guess key |
| **Print-format address fields and dashboard scripts (5.5 rest)** | Customer text shown without escaping in some print formats and dashboard scripts | Low: Frappe cleans the text on save |
| **POST-only for the last few links (5.2 rest)** | A few GET links left on purpose (two Excel downloads, three email links) | Low |
| **Dependencies** | LMS and Insights versions; live-site versions | Medium for LMS (published advisories), to be decided after the live versions are known |

**Items closed by recorded decision (risk accepted):**

| Item | Decision and reason |
|---|---|
| H8: records can be opened by name despite list rules | Tested; left as is (30 Oct) |
| Quotation links for quotations created before 30 Oct | **Left unchanged (6 Nov):** the old key still works for them, because it is not certain that changing it would cause no problem for existing quotations. New quotations use only the new key. The old key is the database password, so the risk is forged links for the 351 older quotations (about 26 still open on the local copy) |
| Approval link changes state on GET; bank-transfer and payment links | Left as is (5 Nov) |
| Engineer can complete a Senior Review | Accepted; reviewer and approver are different people with the same authority |
| 4.2 part C (order-visibility default), 4.6 parts 1, 3, 4, `get_cogs_based_rate`, `mark_order_as_shipped`, `item_image` | Left as is by decision |
| 5.8 paid-API limits | `rerun_validation` removed (integration disabled); 1Click stock check is a free API |
| 6.4 Rewriting the code history to remove old keys | Not possible; all leaked keys were rotated so the old values are dead |

**Known issues found (not caused by the audit work).** Each failing test was investigated on 6 November; none points to a fault in the application code:

| Test group | Failures | Reason | Proof |
|---|---|---|---|
| `test_mcp_dashboard_gates` | 3 errors (fixed 6 Nov) | **Out-of-date test, now corrected.** It logged in as a Factory user and expected access to *every* dashboard in the role list. The list had grown to include the Founder dashboard (System Manager and Super Admin only), Customer Intelligence (Customer Service) and Quotation Analysis (Sales roles). Factory is deliberately **not** allowed to open those three (business decision, 6 Nov), and the code already refuses it | The test now checks both directions: Factory opens only the dashboards whose role list includes Factory, and is refused for those three. A new test pins the rule, and it fails if Factory is ever added to one of them. 12 of 12 pass |
| `test_mcp_field_masking` | 2 errors (fixed 6 Nov) | **Test-data problem, now corrected.** The test picked the first user who can read Lyfe Orders but not cost fields. Locally that was `api-audit-sf@example.com`, a throw-away Shipping Forwarder created by our own audit tests, who can read no orders. The cost-field hiding itself was never at fault | The test now runs as real accounts only (`shobhit@`, `arth@`, then the two support accounts; today it uses shobhit) and picks an order that user may read. 47 of 47 pass. With the hiding code switched off, 3 tests fail, so the tests do catch a real leak |
| `test_quotation` (an older test file from 21 Oct) | 5 of 25 (fixed 6 Nov) | **Out-of-date tests, now corrected.** 3 tested a retail-price function that was removed from the quotation code in an earlier change; they were deleted. 1 expected a box-size check that has since changed (the check now reads the combined box size field): updated, with a second test for the missing-box-size case. 1 expected an item with a zero list price to force approval; since commit `dff84694` the rule is a discount or margin violation or a grand total of 10,000 or more, so the test now checks the 10,000 rule | 23 of 23 pass; with the 10,000 rule and the box-size check switched off, 2 of them fail. **One open question for the business:** should a zero list price also require approval? The code today does not, and nobody has confirmed that this is intended |
| Payment-link sweeper test (quotation e2e) | Occasionally fails | **Flaky by design.** It depends on a cache entry that a cache wipe during other work removes; at worst a Slack alert is delayed after a deploy. It passed in the final run (85 of 85) | Passes alone, with its class, and in the final run |

Also: the framework function `frappe.only_for` does nothing while tests run, so an older test of a gate that relies only on it may pass for the wrong reason.

All three test fixes are committed (`3c3f5b5e`); they are tests only, no application code changed.

---

## 8. Tests

Run on the local copy of the site on 6 November. "Tests" is the number of automated tests in each module.

| Test module | Tests | Result |
|---|---|---|
| `test_whitelist_authz_nov2026` | 13 | OK |
| `test_api_access_audit_nov2026` | 5 | OK |
| `test_sm_sa_never_restricted` | 19 | OK |
| `test_security_fixes` | 9 | OK |
| `test_dashboard_backend_gates` | 4 | OK |
| `test_mcp_field_masking` | 47 | OK (fixed 6 Nov; was 2 errors from the test setup) |
| `test_mcp_pm_user_efficiency_gate` | 10 | OK |
| `test_mcp_dashboard_gates` | 12 | OK (fixed 6 Nov; was 3 errors from an out-of-date test) |
| `test_mcp_audit_double_wrap` | 10 | OK |
| `test_s3_download_access` | 4 | OK |
| `test_set_on_hold` | 2 | OK |
| `test_slack_disabled_user` | 6 | OK |
| `test_slack_unmapped_user_name` | 4 | OK |
| `test_link_search_doctype` | 2 | OK |
| `test_error_log_no_secrets` | 1 | OK |
| `test_output_escaping` | 8 | OK (skipped=1) |
| `test_assign_condition` | 6 | OK |
| `test_external_learning_record_access` | 2 | OK |
| `quotation_e2e.test_api_access` | 8 | OK |
| `quotation_e2e.test_integrations` | 85 | OK |
| `test_quotation` | 23 | OK (corrected 6 Nov; was 5 of 25 failing from out-of-date tests) |

Plus `test_make_invoice_pdfs_private` (4, new) and the older `test_quotation` (23, corrected). Total about 283 tests run. All pass. Three groups that failed earlier on 6 November (`test_mcp_dashboard_gates`, `test_mcp_field_masking`, `test_quotation`) were out-of-date or badly seeded tests, not application faults, and were corrected; details in section 7. New tests were checked by reverting the fix and confirming they fail.

---

## 9. Transparency: incidents during the work

| Event | Effect | Action |
|---|---|---|
| While proving a new test fails on the old code, one test ran the old (unchecked) function and made **one real read-only look-up to Shopify** for quotation LHQ-2026-0446, writing two Integration Request log rows (local copy; quotation unchanged) | None on data | The test now mocks Shopify |
| A failing test run committed 5 test records to the **local** database | None on prod | Rows deleted; the commit removed |
| An early test moved one Task between projects on the local copy | Local only | Restored |
| A 30 Oct decision to keep two helpers as web endpoints was **reversed on 6 Nov** because they are only used by the server | Print formats and S3 upload need a check on the copy | Added as check 17 |
| Several earlier worklog times and one test count were wrong | Documentation only | Corrected |

**No change was made to the live site by this work.** All code went through GitHub; deploys are done by the team.

---

## 10. How to undo the riskiest changes

| Change | How to undo |
|---|---|
| Any single fix | `git revert <commit>` for that commit (all are separate), then deploy |
| Invoice PDFs made private | `bench --site <site> execute lh.patches.make_existing_invoice_pdfs_private.rollback` (restores exactly the files the patch changed) |
| Ticket-id field permission (H10) | Re-grant read on `Task.custom_gorgias_ticket_id` in Role Permission Manager |
| A role gate that blocks a legitimate user | Grant the permission in Role Permission Manager, or revert that commit |

---

## 11. Decisions already taken, and next steps (for your review)

These decisions were taken by the product owner (Viral Kansodiya) during the work and are recorded here for your information. If you disagree with any of them, tell us and we will change it.

| # | Decision or plan | Status |
|---|---|---|
| 1 | The remaining items (section 7) stay pending **on purpose** until a copy of the live site exists on Frappe Cloud; the checks and any fixes are done there | Decided 6 Nov |
| 2 | Changes up to 5 Nov are deployed; the changes made on 6 Nov are deployed on 7 Nov (they include the invoice-PDF patch, which runs automatically and has a rollback command). The S3 and print-format checks follow on the Frappe Cloud copy | Planned for 7 Nov |
| 3 | The recorded risks in section 7 are accepted, mainly: old quotations keep the old link key (not changed, because it is not certain a change would cause no problem for them), H8, and the other items listed there | Decided (30 Oct – 6 Nov) |
| 4 | LMS (2.45.2, 2 published critical advisories) and Insights (2.2.14): update or restrict them once the live-site versions are known | **Still to be decided by us** after the Frappe Cloud copy shows the live versions |
| 5 | Who runs the Frappe Cloud checks and by when | **Still to be set** once the copy exists |
| 6 | One question for the quotation owner: should a quotation item with a zero list price also need approval? The code does not (since an earlier change); not confirmed as intended | Open question |

No action is needed from you unless you want to change something above.

---

## Appendix A. Security audit changes (all, newest first by date in each group)

### A1. Audit fixes, safeguards and removals (68)

| Commit | Date | Change |
|---|---|---|
| `0e003915` | 2026-10-23 | fix: store S3 File Attachment aws_secret encrypted |
| `6f8c2ff6` | 2026-10-23 | chore(security): remove 1Click Postman collection from public docs |
| `f9f83402` | 2026-10-23 | chore(security): redact 1Click API key from public docs |
| `1e1be4a7` | 2026-10-26 | fix(security): reject webhooks when no secret is configured |
| `6a977c35` | 2026-10-26 | fix(security): stop exposing process_shipstation_response as an API |
| `8e7f2410` | 2026-10-26 | chore(security): delete 1Click doc files that held a leaked API key |
| `9368226f` | 2026-10-26 | fix(security): restrict payment recording to CS and System Manager |
| `026bb5c5` | 2026-10-28 | chore(security): remove hardcoded Slack webhook from patch |
| `753af930` | 2026-10-28 | fix(security): restrict Shopify webhook register/deregister to System Manager |
| `b63895e8` | 2026-10-28 | fix(security): restrict CJ label generation to Factory, Founder, Super Admin, System Manager |
| `1ffe7b1e` | 2026-10-29 | fix(security): remove API access to backfill_erp_items_from_item_name and fetch_shopify_details (H5) |
| `369d6dcf` | 2026-10-29 | fix(security): gate "Parse BOM from Drawing" to Engineer, Factory, System Manager, Super Admin (H13) |
| `49f6f9c2` | 2026-10-29 | fix(security): create Material Issue items with the user's own permissions (H13, Medium) |
| `7fd178fc` | 2026-10-29 | fix(security): require access to the task's home project when adding it to a board (H11) |
| `8d79ed5f` | 2026-10-29 | fix(security): gate P&L and COGS exports by role; log every workflow revert |
| `acd99e30` | 2026-10-29 | fix(security): role-gate tracking.track_order; drop ignore_permissions on its save |
| `be7e3c9a` | 2026-10-29 | fix(security): sign quotation approval/tracking links with approval_link_secret, with expiry |
| `c9a49cf6` | 2026-10-29 | fix(security): remove API access to recalculate_weighted_score (H9) |
| `df92578f` | 2026-10-29 | fix(security): role-gate Lyfe Order API actions; drop ignore_permissions where roles have permission |
| `efc88a26` | 2026-10-29 | fix(security): make update_all_real_time_tracking a background-only job (H3) |
| `f7a9f6b8` | 2026-10-29 | fix(security): require Quotation access for the senior review and drawing APIs (H12) |
| `016295d4` | 2026-10-30 | Remove @frappe.whitelist from submit_senior_review (audit 5.2) |
| `1f073e60` | 2026-10-30 | fix(security): CustomFile.get_content no longer fetches remote URLs or sends the sid cookie (H2, part A) |
| `22037fda` | 2026-10-30 | Gate document and stock APIs (audit H4, batch 2) |
| `4924aa55` | 2026-10-30 | Remove @frappe.whitelist from 13 unused write APIs (audit 5.2) |
| `4b1d0824` | 2026-10-30 | Remove Sync Item Groups button and backfill_item_group API (audit 3.7) |
| `5f4ffb09` | 2026-10-30 | Lock down order export APIs (audit H7 3.2, exports) |
| `61cc5da0` | 2026-10-30 | Require Lyfe Order read on status/lifecycle/shipment page APIs (audit H7 3.2) |
| `78dbd84a` | 2026-10-30 | Save Sales Invoice PDFs as private files (audit 3.5) |
| `8479367b` | 2026-10-30 | Revert "POST-only for 29 state-changing Lyfe Order APIs (audit 5.2, batch 1)" |
| `8f956d99` | 2026-10-30 | fix(security): require read access to the order for the pick-pack list (H7) |
| `a2db010b` | 2026-10-30 | Remove @frappe.whitelist from 4 unused Lyfe Order write APIs (audit 5.2) |
| `a6786aa2` | 2026-10-30 | POST-only for 67 JS-called state-changing APIs (audit 5.2) |
| `b517ae90` | 2026-10-30 | fix(security): restrict Task.custom_gorgias_ticket_id to Customer Service, System Manager, Super Admin (H10) |
| `b8a94fe4` | 2026-10-30 | POST-only for 24 button-driven Lyfe Order APIs (audit 5.2, lyfe_order.py) |
| `c95dcabb` | 2026-10-30 | Remove @frappe.whitelist from register_shopify_webhooks (audit 5.2) |
| `d7a9cfe8` | 2026-10-30 | POST-only for 29 state-changing Lyfe Order APIs (audit 5.2, batch 1) |
| `da55c582` | 2026-10-30 | Gate Material Issue for Order APIs by document permission (audit H4, batch 3) |
| `f0ab2241` | 2026-10-30 | Gate outside-service APIs (audit H4, batch 1) |
| `f30f3d9f` | 2026-10-30 | Remove frappe module from legacy rule condition eval (audit 4.7) |
| `10459745` | 2026-11-04 | security: permission checks and un-whitelisting for audit H4 / 5.x (Asana 1219812548257717) |
| `0092e347` | 2026-11-05 | fix(security): Material Issue link searches ignore the caller's doctype for visibility (audit 5.6) |
| `0238ab12` | 2026-11-05 | fix(security): Super Admin / System Manager also allowed in HR, project-board and Slack checks |
| `20aed3db` | 2026-11-05 | fix(security): get_quotation_detail returns COGS only to Factory, System Manager and Super Admin |
| `6c04d319` | 2026-11-05 | fix(security): set_on_hold only for Customer Service, System Manager and Super Admin (audit 4.10) |
| `8899faf6` | 2026-11-05 | fix(security): _assign row rules match the exact user, not a substring (audit 4.2 part A) |
| `8cb9d2ae` | 2026-11-05 | fix(security): Shopify token rotation no longer writes the token response to Error Log (audit 5.4) |
| `93970d25` | 2026-11-05 | fix(security): escape customer and user text in emails, Slack OAuth page and comment mentions (audit 5.5, first part) |
| `96df0650` | 2026-11-05 | test(security): employees see only their own External Learning Record (audit 4.2 part B) |
| `9b23fd48` | 2026-11-05 | fix(security): Lyfe Order create_gate_pass is no longer a whitelisted API (audit 5.2) |
| `a0611537` | 2026-11-05 | fix(security): check File permission before opening S3 files by key (audit H2 part B) |
| `a760832c` | 2026-11-05 | fix(security): bulk_create_gate_pass is no longer a whitelisted API (audit 5.2/5.7) |
| `c6ef02d2` | 2026-11-05 | fix(security): disabled ERP users can no longer act from Slack (audit 4.9) |
| `cee31478` | 2026-11-05 | fix(security): System Manager and Super Admin never blocked by role checks; 3.1 and 5.7 permission checks |
| `07c68446` | 2026-11-06 | fix(security): dashboard backends check who is calling (audit 3.6) |
| `20b0c718` | 2026-11-06 | fix(security): get_s3_image_url and custom_file_upload_to_s3 are no longer web APIs (audit 2.1, M8) |
| `415c4583` | 2026-11-06 | fix(security): patch makes invoice PDFs created before 30 Oct private, with rollback (audit 3.5) |
| `49ee0c0e` | 2026-11-06 | fix(security): order, quotation and shipping-rate APIs check access to the record they read (audit 2.1) |
| `89370477` | 2026-11-06 | fix(security): three read-only-checked functions that still write now need write (audit H4) |
| `9cf8121a` | 2026-11-06 | feat(security): gitleaks pre-commit hook and a secrets rule (audit 5.9) |
| `a07c0ded` | 2026-11-06 | fix(security): demand-forecast and stuck-order APIs check the caller's role (audit 2.1) |
| `a2504e19` | 2026-11-06 | fix(security): MCP filter/sort guard catches JSON-string, 4-element and quoted forms (audit 4.8) |
| `a3cadb7f` | 2026-11-06 | feat(security): pre-commit check that every whitelisted function has a permission check (audit 2.1, 5.9) |
| `aa55d40e` | 2026-11-06 | chore(security): remove unused skip_factory_images and rerun_validation APIs (audit 5.8) |
| `c98db616` | 2026-11-06 | docs(security): mark intentional public whitelisted functions with # authz (audit 2.1) |
| `cc108a9b` | 2026-11-06 | fix(security): Slack comments from users with no ERP account name the sender (audit 4.9) |
| `d1c48af5` | 2026-11-06 | fix(security): Track Order, Track Reship and Track Return need Lyfe Order write (audit 2.1) |
| `e73f7495` | 2026-11-06 | fix(security): reshipment CJ label prefill needs the label roles (audit 2.1) |

### A2. Related work reviewed or fixed during the audit (12)

Public item-image endpoint and its limits, the COGS-based price API and Replenishment order creation (reviewed), the Engineer permission popup hotfix, the ShipStation reroute hotfix, 17Track logging, the Cypress test fix, a comment cleanup, and the removal of the load-test scripts.

| Commit | Date | Change |
|---|---|---|
| `ed0404af` | 2026-10-26 | feat(17track): log webhook pushes and scheduler calls as Integration Requests |
| `3d107ec1` | 2026-10-29 | Add public item_image endpoint serving only Item.image by item code |
| `a553f6ef` | 2026-10-29 | fix(lyfe-order): remove invalid ignore_permission kwarg that broke the ShipStation sync reroute |
| `f970a196` | 2026-10-29 | fix(lyfe-order): stop "No permission for Oneclick Settings" when Engineers open a Lyfe Order |
| `529e91be` | 2026-10-30 | Remove obsolete load-test scripts |
| `d025a2cf` | 2026-10-30 | docs: remove misleading has_permission comment from hooks.py |
| `0f34b898` | 2026-11-02 | fix: Quotation — CS users get the COGS-based price and no false COGS permission error |
| `45e77120` | 2026-11-02 | Restrict item_image API to items listed on Shopify |
| `6c801305` | 2026-11-02 | Create Lyfe Order from Replenishment: Factory, System Manager, Super Admin only |
| `ba4d1987` | 2026-11-02 | Return flat JSON errors with error_code from item_image API |
| `63a72307` | 2026-11-05 | test(cypress): C12 looks up comment-linked tasks with frappe.client.get_list |
| `14d5cb7e` | 2026-11-06 | feat: rename Access Settings to LH Settings, add Item Image API Key to item_image |

---

## Appendix B. Other business work in the same period (126 changes, not part of the audit)

These changes were made by the team in the same period. Each was covered by the automated checks in section 6.

| Theme | Changes |
|---|---|
| Quotation features and fixes | 28 |
| Stock transfer, Bulk Transfer Batch and 1Click | 26 |
| ShipStation, Shopify, 17Track and Item data | 24 |
| Print formats, Excel export and Sales Invoice | 11 |
| SLA, projects, Clarity, tests and chores | 10 |
| Stock forecast dashboard | 9 |
| Status overview, Lyfe Order and reshipment | 9 |
| Timesheet | 5 |
| Other | 2 |
| Slack | 2 |

Full list:

| Commit | Date | Change |
|---|---|---|
| `4fe12655` | 2026-10-21 | feat(stock_forecast): clean analysis dataset export for Forecasting v2 |
| `5b423b3e` | 2026-10-21 | feat(stock_forecast): demand hypothesis tests and forecast method backtest |
| `11401647` | 2026-10-22 | feat(sales-invoice): Exchange Rate and Description of Goods fields, hide Invoice Type |
| `2802d463` | 2026-10-22 | fix(print formats): load logos from app assets and inline S3 signatures for PDF |
| `39130d54` | 2026-10-22 | fix(bulk-transfer-batch): drop submitted Gate Pass check on submit |
| `39f72a13` | 2026-10-22 | fix(lyfe order): set Factory Leg Destination to Via US Warehouse on "Route via US Warehouse Instead" |
| `3c35b1a2` | 2026-10-22 | feat(status overview): count dispatched via-US-Warehouse orders under Customer Support |
| `4503a1e9` | 2026-10-22 | feat(item-data-completeness): add SKU Type filter to dashboard |
| `4e87d133` | 2026-10-22 | fix(sales-invoice): Export lists only enabled print formats |
| `50cde52d` | 2026-10-22 | fix(bulk transfer batch): exclude cancelled orders from posting, 1Click email, stock file and packing lists |
| `5249b2b2` | 2026-10-22 | feat(print-format): Sales Invoice Packing List and Tax Invoice follow BTB layouts |
| `5f791487` | 2026-10-22 | feat(bulk-transfer-batch): update HTML structure and content for packing list |
| `644bdb87` | 2026-10-22 | fix(excel-export): place images by cell alignment so the logo doesn't cover text |
| `825a32e2` | 2026-10-22 | fix(lyfe order): backfill Factory Leg Destination "Via US Warehouse" on re-routed orders |
| `92ecfd4f` | 2026-10-22 | feat(status overview): count Submitted to 1Click orders under Customer Support |
| `97c743e9` | 2026-10-22 | fix: copy Lyfe Order cj_exporter_inv_no to Sales Invoice custom_export_invoice_no |
| `af80be42` | 2026-10-22 | feat(status overview): hide Completed reshipments unless Show Completed Orders is ticked |
| `b15c373c` | 2026-10-22 | chore(print-format): disable unused Sales Invoice print formats |
| `f6bb3461` | 2026-10-22 | fix(lyfe order): hide 1Click buttons and block 1Click actions on cancelled orders |
| `fb652d90` | 2026-10-22 | fix(bulk transfer batch): copy batch tracking to each order's Factory -> US Warehouse leg |
| `3ed7320a` | 2026-10-23 | fix(excel export): load /assets/ logos and inlined data: URI signatures |
| `4ebbdec3` | 2026-10-23 | feat(timesheet): TechCS steps, revision timesheets, non-drawing work |
| `51340e08` | 2026-10-23 | feat(timesheet): make Work Type a Link so Engineers can add options |
| `666b4165` | 2026-10-23 | update patch |
| `7ae952fd` | 2026-10-23 | fix(shipstation): reverify all in-flight Lyfe Orders, not an allow-list |
| `860fc392` | 2026-10-23 | fix(print formats): flatten S3 images onto white so signatures aren't black |
| `9548b90f` | 2026-10-23 | fix(quotation): set payment method from how the payment was actually made |
| `9965a9f9` | 2026-10-23 | fix(refund capture): prevent Slack notifications from dev sites by returning early in _notify_slack_refund_still_stuck |
| `c482cf23` | 2026-10-23 | fix(bulk-transfer-batch): include cancelled-order items in 1Click stock sheet |
| `e38fd57d` | 2026-10-23 | fix(quotation): show Track Payment Status button on submitted unpaid quotations |
| `013c2b5a` | 2026-10-26 | chore(print_format): add missing __init__.py for sdf |
| `0699dae8` | 2026-10-26 | fix(timesheet): hide HRMS Create Salary Slip button |
| `0ddd1d8d` | 2026-10-26 | feat(stock-forecast): Restock List cover for next N days with reasons |
| `15b6f9b5` | 2026-10-26 | fix(patches): verify S3 upload using the key stored on the File row |
| `32deb1b3` | 2026-10-26 | docs(slack): note refund escalation alert is skipped on developer_mode sites |
| `33dd65d1` | 2026-10-26 | chore(patches): remove superseded Item image scripts |
| `45d90229` | 2026-10-26 | feat(stock-forecast): Demand Trends tile with last N days and top N |
| `4b457650` | 2026-10-26 | feat(timesheet): hide "Work on Same" once a revision is started |
| `59b575ba` | 2026-10-26 | feat(stock-forecast): v2 research — H2 buffer valuation and finished/unfinished segments |
| `656b4dee` | 2026-10-26 | fix(patches): link Shopify CDN URL in Item.image instead of uploading |
| `7f7fbcba` | 2026-10-26 | fix(shopify): retry draft-order payment link creation |
| `89440726` | 2026-10-26 | feat(lh_project): route tracking SLA tasks to an "Order Tracking" column |
| `96a10404` | 2026-10-26 | feat(stock-forecast): Demand Forecast and Forecast vs Actual tiles |
| `c3daa582` | 2026-10-26 | feat(sla): one tracking task per Bulk Transfer Batch on the India -> US leg |
| `d4700bdb` | 2026-10-26 | fix(patches): Shopify-safe child SKU image correction from price list |
| `e1cb6c1b` | 2026-10-26 | fix(quotation): reject negative item rate and approval link on closed quotations |
| `f7b13efd` | 2026-10-26 | feat(stock-forecast): page filter bar and Trending Items tile |
| `f8dbf9aa` | 2026-10-26 | feat(timesheet): "Add New Log" dropdown to switch to any work step |
| `16cb1369` | 2026-10-27 | feat(bulk-transfer-batch): show cancelled orders on packing lists |
| `1d63cfd1` | 2026-10-27 | fix(quotation): resend email status check, poll error log title, test fixtures |
| `244bad57` | 2026-10-27 | feat(stock-forecast): show tube items in a separate collapsible section |
| `25bed8db` | 2026-10-27 | fix(shipstation): exclude Shopify-cancelled orders from Shopify gap check |
| `2fdebc27` | 2026-10-27 | test(quotation): 30 negative E2E tests across quotation features |
| `4cb80ef8` | 2026-10-27 | fix(sales-invoice): keep Draft SI in sync with Lyfe Order cj_shipment_items |
| `4fbfbd37` | 2026-10-27 | fix(project_board): cap board tasks per column, not globally |
| `51205002` | 2026-10-27 | feat(stock-forecast): Category filter on every stock tile, larger tile buttons |
| `69d0f605` | 2026-10-27 | feat(quotation): log each payment link attempt; dev-mode failure alert to 1Click webhook |
| `702689bf` | 2026-10-27 | feat(quotation): stop linked Timesheet timers on drawing attach; allow cancel with Timesheets |
| `7ab3ea64` | 2026-10-27 | test(quotation): Sales User Review → Send for Approval requires COGS |
| `baf5ebbe` | 2026-10-27 | fix(shipstation): skip missing-order alerts for orders cancelled on Shopify |
| `01322b2d` | 2026-10-28 | fix(shopify): mark restock-only refunds Completed instead of Failed |
| `07c5a1d4` | 2026-10-28 | fix(quotation): auto-advance Inquiry Approved to Production Details Review on Feasibility Test Required |
| `2a857527` | 2026-10-28 | feat(quotation): CS Suggested / Factory Amount on taxes + amount choice on approval |
| `7336ae37` | 2026-10-28 | feat(reshipment): New Delivery Address + Add New Address button |
| `8c378c19` | 2026-10-28 | feat(lyfe_order): Export COGS Range — reshipment-only export by Ready for Dispatch Date |
| `93290118` | 2026-10-28 | chore(quotation): disable combined box update in validate |
| `af36d1e6` | 2026-10-28 | fix(sla): pass job_id to tar_escalation_scan enqueue |
| `bcfd40cc` | 2026-10-28 | feat(lyfe_order): Export COGS Range — qty, extra item names/COGS, reshipment sheets |
| `e0413109` | 2026-10-28 | feat(shopify): send cancelled-quotation payment alert to CS Quotation channel |
| `f8214348` | 2026-10-28 | feat(status-overview): Active Reshipment export uses reshipment consignee address |
| `04e62643` | 2026-10-29 | feat(stock-forecast): finish-wise details, top placement and collapsible categories |
| `07b4dcfd` | 2026-10-29 | fix(clarity): retry transient timeouts/5xx on daily Clarity pull |
| `0ab6283e` | 2026-10-29 | MIFO: link quick Material Receipt failures to the MIFO in Error Log |
| `0f4b6db3` | 2026-10-29 | Log all Slack sends (bot + 1Click) to Slack Notification Log |
| `279c9a34` | 2026-10-29 | Add Get Shipping Rate + Create Gate Pass to Bulk Transfer Batch |
| `45833d76` | 2026-10-29 | fix(sla): stop repeat Task links and alerts for breaches whose Task is closed |
| `5de77ed3` | 2026-10-29 | test(sla): update tag-forwarding mock for create_project_task return_created |
| `5efefb9b` | 2026-10-29 | ShipStation: truncate skip Error Log title to 140 chars |
| `665c8c33` | 2026-10-29 | fix(sla): Aging Quotation detector skips terminal quotations |
| `e48c3c7d` | 2026-10-29 | Slack log: don't drop rows with bad references; document the log |
| `05f45964` | 2026-10-30 | Lyfe Order is source of truth for linked draft Sales Invoice |
| `0be789ea` | 2026-10-30 | revert(quotation): remove Factory / CS Suggested Amount feature |
| `4e6449a5` | 2026-10-30 | fix: qualify Contact columns in quotation email fallback query |
| `56552907` | 2026-10-30 | Reshipment is source of truth for its draft SI; add box_number and reshipment header fields |
| `ab18b406` | 2026-10-30 | fix: fall back to Contact email when quotation.contact_email is blank |
| `1e26bbfe` | 2026-11-02 | fix(quotation): retry + Integration Request log for Shopify draft-order cancel calls |
| `3eaab7ef` | 2026-11-02 | fix: set item_group on Replenishment-created Lyfe Order items + backfill patch |
| `4869f1c6` | 2026-11-02 | Set Is Listed Product from published Shopify SKUs in missing-items patch |
| `4d5ea0b9` | 2026-11-02 | Stock-transfer orders: 1Click inbound Load only, block Create Order |
| `5d2ac0c4` | 2026-11-02 | Stock-transfer orders: route like Via US Warehouse, single leg, complete on delivery |
| `6cd2b2fd` | 2026-11-02 | Add patch to fill empty Item images from Shopify |
| `6ee5d920` | 2026-11-02 | Add Create Lyfe Order to US Warehouse Replenishment |
| `92841c16` | 2026-11-02 | Stock-transfer orders: invoice to US warehouse Customer, complete without Load if nothing issued |
| `d32dfbd7` | 2026-11-02 | Patch: upgrade earlier Replenishment-created Lyfe Orders to the Via-US-Warehouse process |
| `d83845f2` | 2026-11-02 | Set CJ consignee to US warehouse on Replenishment-created Lyfe Order |
| `e5db0a55` | 2026-11-02 | feat(quotation): delete Shopify draft order on cancel; block cancel if draft is not open |
| `f99f084a` | 2026-11-02 | Add patch to create Item master rows for Shopify SKUs missing in ERP |
| `20e8166a` | 2026-11-03 | patch: delete open Shopify draft orders of cancelled Quotations and clear payment link |
| `5676d630` | 2026-11-03 | feat(lyfe_order): USR connections + Create Load for stock-transfer orders |
| `70f647ee` | 2026-11-03 | fix(quotation): delete the Shopify draft order and clear the payment link when marked Lost |
| `9439279a` | 2026-11-03 | test(quotation): cover POST-only APIs, API access gates and the CS pricing API |
| `a75c6ef8` | 2026-11-03 | Lyfe Order: prefer cj_export_partner over Gate Pass shipper when syncing to Sales Invoice |
| `abd085d1` | 2026-11-03 | feat(quotation): payment link required before send; sweeper creates missing links |
| `b131c066` | 2026-11-03 | feat(lyfe_order): add Reason and Description columns to Reshipment COGS export |
| `bb123ba5` | 2026-11-03 | test(quotation): remove Combined Box tests; CS guide: Cancel deletes the payment link |
| `cae2bf10` | 2026-11-03 | fix(quotation): extract the "did the email send?" check and test it with Node |
| `cbceb142` | 2026-11-03 | test(quotation): cover cancel deleting the Shopify draft order and Resend Email |
| `d89fece6` | 2026-11-03 | feat(lyfe_order): send 1Click inbound email when Create Load is posted for a stock-transfer order |
| `e5b34ee3` | 2026-11-03 | Patch: correct ship-to/consignee address of LH3316 (Cape Resorts, Attn Hadley Kronemeyer) |
| `eee385f4` | 2026-11-03 | feat(lyfe_order): add bulk edit option and update descriptions for shipment items and packages |
| `351457b3` | 2026-11-04 | Fix 27 items showing the Antique Brass photo instead of their own finish |
| `c148d021` | 2026-11-04 | fix(lyfe-order): take paid_amount from Shopify's received payments, not ShipStation's amountPaid |
| `f41bc817` | 2026-11-04 | fix(oneclick): allow Post to 1Click while the order is In Transit to US Warehouse |
| `f9adebb7` | 2026-11-04 | feat(oneclick): two inbound emails per order/batch, logged on the document |
| `26137e1b` | 2026-11-05 | feat(slack): two-way edit/delete sync, attachments, factory tag only when asked |
| `2cdc7bc1` | 2026-11-05 | Quotation: restore feasibility_reviewed from version history |
| `77ed33e9` | 2026-11-05 | Quotation: Not Feasible closes the feasibility request |
| `94238b25` | 2026-11-05 | fix(lyfe-order): use assign_to._remove when undoing Factory assignment |
| `9e9b8c21` | 2026-11-05 | fix(sales-invoice): rebuild draft SI rows 1:1 from Lyfe Order cj_shipment_items |
| `aee8c403` | 2026-11-05 | fix(us-warehouse-replenishment): update STOCK_TRANSFER_CUSTOMER to "Lyfe Hardware" |
| `d3399a40` | 2026-11-05 | Rename US Warehouse Replenishment doctypes to Stock Transfer Order 1Click |
| `eb9b7b69` | 2026-11-05 | feat(quotation): choose reviewer engineer in Request Senior Review dialog |
| `2087c2b2` | 2026-11-06 | Slack: sync thread replies to ERP only when tagged @factory or ERP-BOT |
| `935f44ed` | 2026-11-06 | feat(stock-forecast): TSB forecast, one-off exclusion, safety stock and base-level tile |
| `c933e52a` | 2026-11-06 | Fix item photos, batch 2: MMP-200-ABZ and UMMP-200-PSS |
| `cc3290fd` | 2026-11-06 | feat: daily Shopify -> ERP item sync (image, name, price, listed flag, new items) |

---

## Appendix C. Glossary

| Term | Meaning |
|---|---|
| **Whitelisted function / API** | An action the browser or an outside tool can call by name. Any logged-in user can call any of them unless it checks permissions itself |
| **Permission check** | Code that asks "is this user allowed to do this to this record?" before acting |
| **Webhook** | A message another system (Shopify, 17Track, Slack) sends to ours. We must verify it really came from them |
| **HMAC / signature** | A cryptographic stamp that proves a message or link was made by someone holding our secret |
| **SSRF** | Tricking our server into fetching a web address on an attacker's behalf |
| **Permission level (permlevel)** | A field-level lock inside a record: a user may open the order but not see the cost fields |
| **COGS** | Cost of goods sold: what the goods cost us |
| **MCP** | The connector that lets AI assistants read our ERP data |
| **System Manager / Super Admin** | Administrator roles; by our rule they are never blocked by a permission check |
| **Patch** | A one-time data update that runs with `bench migrate` on deploy |
| **Frappe Cloud copy** | A test copy of the live site, used to check changes against real data without touching the live site |
| **Gitleaks** | A tool that scans code for keys, tokens and passwords |
