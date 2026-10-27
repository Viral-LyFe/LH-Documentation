# Quotation E2E Test Report

Oct 21, 2026 · updated Oct 27, 2026 · @Viral kansodiya

149 automated end-to-end tests cover every LH Quotation customisation. On the latest run (Oct 27) 148 pass and 1 is a known expected failure. Nothing is deployed to production yet.

## Results

Run on lyfelocal.com (Frappe 15.121, ERPNext 15.121) on Oct 26; test\_integrations (per-attempt logging change) and test\_lifecycle (Sales User Review approval tests) were re-run on Oct 27. A full run takes about 80 seconds and leaves the database and disk exactly as it found them.

| Module | Area | Tests | Pass | Fail | Expected failure |
| --- | --- | --- | --- | --- | --- |
| test\_lifecycle | Workflow walks, transition guards, approval, revisions | 45 | 45 | 0 | 0 |
| test\_payments | Payment schedule, fee, stage blocks, milestones, reminders | 27 | 27 | 0 | 0 |
| test\_integrations | Email, approval link, payment link retry, Shopify, follow-up, SLA, COGS benchmark, permissions | 36 | 36 | 0 | 0 |
| test\_validate | Order type, box sizes, qty / rate, discounts, approval / COGS flags | 24 | 24 | 0 | 0 |
| test\_drawings | Upload gate, PDF BOM stripper, change details, Senior Review | 17 | 16 | 0 | 1 |
| **Total** |  | **149** | **148** | **0** | **1** |

The expected failure is drawings test 6, a local-storage-only issue described in that test.

Every test runs inside one database transaction that is rolled back at the end. Outbound effects (background jobs, email, Slack, Shopify, S3, PDF rendering) are stubbed, so a run sends nothing. Quotation, Lyfe Order, File and Comment counts were identical before and after a full run.

## Live timeout test (Oct 27)

The automated tests simulate Shopify. This test used a real network timeout on lyfelocal.com (developer mode on) to confirm the 5-attempt retry, the per-attempt logging and the failure handling end to end.

**Setup**

- A local stand-in server on `127.0.0.1:9999` that accepts the connection but never replies, so every Shopify call waits for its full 20-second limit and fails with **ReadTimeout**.
- **Shopify Setting → Shopify URL** pointed at `https://127.0.0.1:9999` for the test only.
- Quotation **LHQ-2026-0410** (Sent to Customer, owner Administrator). For test A its draft order ID and payment link were cleared first.
- Afterwards the Shopify URL and the quotation's draft order ID and payment link were put back. Both were checked after the restore.

**A. Generate Payment Link: all 5 attempts time out** · Pass

| Check | Expected | Result |
| --- | --- | --- |
| Attempts | 5, each failing with ReadTimeout | 5 × `ReadTimeout … Read timed out. (read timeout=20)` |
| Integration Requests | Attempt 1/5 to 5/5, all Failed | g2mp573nbu, g9u2l7gs9e, gh6rn61dpf, godcprsu49, gvl56268lj: all Failed |
| Time between attempts | 20 s timeout + 3 s wait | about 23.2 s (04:25:44 → 04:26:07 → 04:26:31 → 04:26:54 → 04:27:17) |
| Total time | Under 120 s | 113.9 s |
| User sees | The error | ReadTimeout raised to the caller |
| Payment link | Not saved | Empty |
| Slack alert | Sent once, to the 1Click webhook (developer mode) | Sent: no "alert failed" Error Log. 📷 [SCREENSHOT PL-01: Slack message in the 1Click channel] |

**B. Update Payment Link after a revision: all 5 attempts time out** · Pass

Run by calling the update directly (the same call a revision save makes).

| Check | Expected | Result |
| --- | --- | --- |
| Integration Requests | Attempt 1/5 to 5/5, all Failed | h64hd0hj7v, hdb04dg7bt, hki5th33li, hrqae1lf0j, i32m4ffesp: all Failed, ReadTimeout |
| Crash | None | Finished normally after 113.8 s |
| Old payment link | Cleared | Cleared |
| Error Log | One entry | i9g8fqad21 `[shopify][update_draft_order] LHQ-2026-0410 — no response` |
| Owner alert | In-app notification | i9ij79tf0n "Payment link update failed for LHQ-2026-0410" (Alert, to Administrator) |
| Slack | None (this path alerts the owner only) | None |

**Findings**

- **Close to the 120-second limit.** 5 timeouts took 113.9 s, leaving about 6 s before the production web server's 120-second request limit. Fine today, but any extra attempt, a longer wait or a longer timeout would cross it.
- **No Error Log for a timeout on Generate Payment Link.** The code writes an Error Log for an HTTP error from Shopify but not for a timeout on this path. The 5 Integration Requests and the Slack alert are the record.

## Business rules confirmed

The tests check these business rules.

| Rule | Behaviour |
| --- | --- |
| Payment stage unpaid: Advance, Photo Approval, Before Dispatch | Hard block on the Lyfe Order step, plus a reminder |
| Payment stage unpaid: Before Shipped, On Delivery | Reminder only, never a block |
| Quotation Lost or Cancelled (any route, including the follow-up scheduler) | No collection reminders; payment blocks on linked Lyfe Orders are released |
| Partial payment recorded | Advance stage becomes Due and CS gets the Due reminder |
| Send for Approval from Inquiry Approved or Feasibility Approved | COGS not required (0-rate quotations carrying only Custom Fee and Shipping) |
| Send for Approval from Sales User Review | COGS required: the workflow condition is `doc.cogs_pending == 0 and doc.approval_required == 1` (intentional) |
| Production Details Review → Feasibility Approve, to any state | COGS required |
| Production Details Review → Not Feasible | COGS not required |
| Entering Pending Approval, from any route | Slack approval card posted once, after the save commits |
| Pending Engineer Review → Completed with Drawing Required or Photo Required | Order Drawing required, except Urgent quotations |
| Item rate | 0 allowed (fee-only quotes); negative rejected |
| Item qty | 0 rejected (ERPNext core) |
| Payment link generation or update fails (any error) | Retried every 3 seconds, up to 5 attempts; alert only if all 5 fail |
| Every payment link attempt to Shopify | Logged as its own Integration Request, **Shopify Payment Link - Attempt 1/5** … **5/5**, linked to the quotation, with the error or Shopify's reply |
| Payment link generation fails on a developer-mode (local) site | The Slack alert goes only to LH Slack Settings → 1Click Slack Webhook URL, never to the urgent channel |
| Shopify times out on every attempt | Each attempt waits at most 20 seconds. Generate Payment Link shows the error and alerts Slack; a revision update clears the old link and alerts the owner; the payment check job moves on to the next quotation |
| Customer approval link on a Lost, Cancelled or Expired quotation | Rejected ("Quotation No Longer Valid"); state unchanged |

## Coverage

Every item in the Asana scope is covered by at least one test class.

| Asana scope item | Test classes |
| --- | --- |
| Custom-quotation detection | OrderTypeDetectionTest |
| Item qty / rate limits | QtyAndRateTest |
| Item dimension fill | ItemDimensionTest |
| Combined box update | CombinedBoxTest |
| Order value change check | PaymentScheduleTest (amounts follow the total) |
| Payment schedule recalc, validation, stage checkbox sync | PaymentScheduleTest |
| Payment processing fee | ProcessingFeeTest |
| Shipping charges + custom fee rows | AutoClearAndCleanupTest, HappyPathTest |
| Revision handling, items-changed detection | RevisionTest |
| Item discount violation (incl. bank transfer) | DiscountViolationE2ETest |
| Workflow: before\_workflow\_action, urgent flag | BeforeWorkflowActionTest, EngineeringBranchTest, ApprovalPathTest |
| Payment gates released on cancel | PostSubmitHooksTest, PaymentGateReleaseTest |
| Slack approval card | ApprovalPathTest |
| Stage payment enforcement, milestone statuses | StagePaymentEnforcementTest, EnforcePaymentStateTest, MilestoneStatusTest, PartialPaymentDueTest, ReminderOnlyStagesTest, NoRemindersWhenClosedTest |
| Drawing upload, PDF BOM stripper, drawing change pending, grace sweep, cancel upload | DrawingUploadGateTest, PdfBomStripperTest, DrawingChangeDetailsTest, GraceSweepTest |
| Senior Drawing Review (approval binding, spec lock) | SeniorReviewTest |
| LH Project SLA auto-close | SlaAndCogsWiringTest |
| COGS benchmark check | SlaAndCogsWiringTest |
| Permission query conditions | PermissionQueryTest |
| Email: send, open tracking, approval token + approve\_quotation | SendQuotationEmailTest, EmailOpenTrackingTest, CustomerApprovalLinkTest |
| Shopify payment link + payment status polling | ShopifyLinkTest, PaymentLinkRetryTest, ShopifyPollingTest |
| Follow-up scheduler | FollowupSchedulerTest |
| Linked Lyfe Orders, dashboard data | LinkedOrdersAndDashboardTest |
| Full lifecycle | HappyPathTest, RejectionBranchTest |

## Negative test cases

69 of the 149 tests are negative tests: they feed in bad data or a disallowed action, or check that something does not happen. Numbers match the test case catalogue below.

### Blocked, rejected or flagged (46)

| Module | # | Test | Negative case |
| --- | --- | --- | --- |
| test\_lifecycle | 3 | test\_sent\_to\_customer\_needs\_email\_and\_address | Send to customer without email, then without address |
| test\_lifecycle | 6 | test\_won\_with\_no\_payment\_is\_blocked | Won with no payment recorded |
| test\_lifecycle | 13 | test\_fee\_only\_quote\_blocked\_from\_sales\_user\_review | Send for Approval from Sales User Review with COGS pending |
| test\_lifecycle | 20 | test\_slack\_approve\_rejected\_by\_erp\_replies\_in\_thread | Slack Approve rejected by ERPNext |
| test\_lifecycle | 22 | test\_completed\_blocked\_without\_drawing | Completed without drawing |
| test\_lifecycle | 23 | test\_completed\_to\_production\_review\_blocked\_without\_drawing | Completed → Production Details Review without drawing |
| test\_lifecycle | 24 | test\_completed\_blocked\_by\_item\_photo\_required | Completed with an item Photo Required and no drawing |
| test\_lifecycle | 30 | test\_production\_review\_needs\_shipping\_unless\_free | Feasibility Approve without shipping |
| test\_lifecycle | 31 | test\_feasibility\_approve\_needs\_cogs\_whatever\_the\_target | Feasibility Approve with COGS 0 |
| test\_lifecycle | 34 | test\_request\_senior\_review\_only\_for\_custom\_orders | Senior Review on a Standard order |
| test\_lifecycle | 35 | test\_inquiry\_rejected\_needs\_reason | Inquiry Rejected without a reason |
| test\_lifecycle | 36 | test\_not\_feasible\_needs\_reason\_then\_cs\_review | Not Feasible without a reason |
| test\_lifecycle | 37 | test\_not\_feasible\_reason\_locked\_to\_factory | Sales User edits Not Feasible Reason |
| test\_lifecycle | 38 | test\_guards | Form button checks: missing drawing, COGS 0, approval required |
| test\_lifecycle | 41 | test\_rev3\_blocks\_further\_changes | Change after Rev 3 |
| test\_lifecycle | 43 | test\_item\_edit\_after\_send\_is\_rejected | Item edit after submit |
| test\_lifecycle | 45 | test\_invalid\_shopify\_link\_rejected | Non-Shopify URL as Shopify Order Link |
| test\_payments | 2 | test\_zero\_portion\_rejected | 0% schedule row |
| test\_payments | 3 | test\_row\_without\_stage\_rejected | Schedule row with no stage |
| test\_payments | 4 | test\_amounts\_must\_sum\_to\_total | Schedule amounts not adding up to the total |
| test\_payments | 7 | test\_underpaid\_blocks\_and\_tracks\_on\_lyfe\_order | Unpaid stage on the Lyfe Order step |
| test\_payments | 8 | test\_repeat\_block\_does\_not\_re\_notify | Repeat block (no second reminder) |
| test\_payments | 10 | test\_non\_cs\_user\_gets\_generic\_message | Non-CS user sees only the general message |
| test\_payments | 14 | test\_unpaid\_is\_rejected | Unpaid quotation marked Won |
| test\_integrations | 1 | test\_no\_outgoing\_account\_skips | No outgoing email account |
| test\_integrations | 4 | test\_requires\_contact\_email | Send email with no contact email |
| test\_integrations | 6 | test\_bad\_token\_records\_nothing | Tracking image with a bad token |
| test\_integrations | 8 | test\_bad\_or\_missing\_token\_rejected | Approval link with a forged or missing token |
| test\_integrations | 9 | test\_closed\_quotation\_not\_approved | Approval link on Lost, Cancelled or Expired |
| test\_integrations | 10 | test\_cancelled\_docstatus\_not\_approved | Approval link on a cancelled document |
| test\_integrations | 12 | test\_gives\_up\_after\_5\_failed\_attempts | Shopify fails all 5 attempts: stops and reports the error |
| test\_integrations | 13 | test\_network\_error\_every\_attempt\_raises | Network error on all 5 attempts: error shown |
| test\_integrations | 16 | test\_generate\_link\_all\_timeouts\_alerts\_and\_saves\_nothing | Generate Payment Link: Shopify times out on all 5 attempts |
| test\_integrations | 17 | test\_revision\_update\_all\_timeouts\_clears\_link\_and\_alerts\_owner | Revision update: Shopify times out on all 5 attempts |
| test\_validate | 6 | test\_bom\_with\_missing\_child\_dims\_is\_flagged | BOM part with missing box size (flagged) |
| test\_validate | 10 | test\_missing\_cogs\_sets\_cogs\_pending | Missing COGS (COGS Pending flagged) |
| test\_validate | 14 | test\_negative\_rate\_rejected | Negative item rate |
| test\_validate | 15 | test\_zero\_qty\_rejected | Zero quantity |
| test\_validate | 16 | test\_card\_payment\_over\_5pct\_needs\_approval | Card discount over 5% (approval flagged) |
| test\_validate | 18 | test\_item\_group\_limit\_applies\_when\_item\_has\_none | Discount over the Item Group limit (approval flagged) |
| test\_drawings | 1 | test\_engineer\_without\_timesheet\_is\_blocked | Engineer upload without a Timesheet |
| test\_drawings | 9 | test\_first\_upload\_needs\_details\_then\_notifies | Drawing saved without change details |
| test\_drawings | 10 | test\_clear\_then\_reattach\_counts\_as\_replacement | Clear and re-attach without change details |
| test\_drawings | 11 | test\_popup\_flow\_mark\_pending\_then\_submit\_details | Blank change details |
| test\_drawings | 13 | test\_endpoints\_reject\_unknown\_quotation | Popup actions called for a quotation that does not exist |
| test\_drawings | 15 | test\_request\_and\_complete\_senior\_review | Senior Review without a Reason Code, and a duplicate request |

### Must not happen (23)

| Module | # | Test | Negative case |
| --- | --- | --- | --- |
| test\_lifecycle | 11 | test\_slack\_card\_off\_when\_setting\_disabled | No Slack card when the setting is off |
| test\_lifecycle | 17 | test\_no\_second\_card\_while\_already\_pending | No second Slack card |
| test\_lifecycle | 19 | test\_slack\_approve\_does\_not\_double\_close\_card | Slack card not closed twice |
| test\_lifecycle | 40 | test\_no\_item\_change\_no\_revision | No revision without an item change |
| test\_lifecycle | 42 | test\_revision\_ignored\_outside\_sent\_to\_customer | No revision outside Sent to Customer |
| test\_payments | 11 | test\_unconfigured\_stage\_is\_noop | Stage not in the schedule does nothing |
| test\_payments | 16 | test\_non\_terminal\_state\_keeps\_block | Non-closing state keeps the block |
| test\_payments | 20 | test\_block\_reminder\_skipped\_and\_cleared\_for\_lost | No block reminder for Lost |
| test\_payments | 22 | test\_due\_and\_stage\_notifications\_skipped\_for\_cancelled | No Due or stage notification for Cancelled |
| test\_payments | 25 | test\_unpaid\_shipped\_stage\_does\_not\_block\_dispatch | Unpaid Before Shipped does not block dispatch |
| test\_payments | 26 | test\_unpaid\_shipped\_stage\_sends\_reminder | Before Shipped reminder sets no payment block |
| test\_payments | 27 | test\_order\_code\_never\_hard\_blocks\_reminder\_only\_stages | No hard block for Before Shipped / On Delivery |
| test\_integrations | 3 | test\_bank\_transfer\_never\_gets\_card\_link | No card link for bank transfer |
| test\_integrations | 22 | test\_open\_draft\_changes\_nothing | Open Shopify draft records nothing |
| test\_integrations | 23 | test\_timeout\_on\_one\_quotation\_does\_not\_stop\_others | Shopify timeout on one quotation stops the others |
| test\_integrations | 24 | test\_pending\_bank\_transfer\_order\_not\_recorded | Pending bank-transfer order not recorded |
| test\_integrations | 26 | test\_day\_already\_sent\_is\_skipped | Follow-up day already sent is skipped |
| test\_integrations | 28 | test\_valid\_till\_passed\_expires | Expired quotation gets no follow-up task |
| test\_integrations | 29 | test\_never\_emailed\_is\_skipped | Never-emailed quotation is skipped |
| test\_integrations | 31 | test\_cogs\_deviation\_enqueues\_alert\_only\_when\_cogs\_changes | No COGS alert when COGS is unchanged |
| test\_drawings | 7 | test\_pdf\_without\_bom\_left\_alone | PDF without a BOM page left untouched |
| test\_drawings | 8 | test\_unchanged\_drawing\_not\_reprocessed | Unchanged drawing not reprocessed |
| test\_drawings | 14 | test\_sweep | Unanswered upload undone by the scheduled job |

## Terms used in this report

| Term | Meaning |
| --- | --- |
| State | The quotation's workflow state (Inquiry, Sent to Customer, Won …). |
| Action | A workflow button that moves the quotation to another state (Send for Approval, Completed …). |
| Submitted | From **Sent to Customer** onward the quotation is submitted: its item lines are locked. |
| Blocked | The action or save is refused and the user sees an error message. "Blocked with X" means the message contains X. |
| COGS | Cost of Goods per item line, entered by the factory. |
| COGS Pending | Ticked automatically when an item line has no COGS. |
| Approval Required | Ticked automatically when the total is 10,000 or more, or a discount is over its limit. |
| Spec hash | A snapshot of the items saved when the quotation is approved. If the items change later, the approval is cancelled. |
| Payment schedule / stage | The rows that split the total into parts. Each row has one stage: Advance, Photo Approval, Before Dispatch, Before Shipped or On Delivery. |
| Payment block | A hold on the Lyfe Order until the payment for the current stage is received. |
| Milestone status | The status shown on each schedule row: Upcoming, Due, Paid, Overdue or Escalated. |
| Shopify draft order / payment link | The Shopify order created for card payment; its link is the payment link in the customer email. |
| Token | A secret code inside a link (approval link, tracking image) that proves it was issued for that quotation. |
| Queued | A background job or Slack/email message was handed to the system to send. In tests nothing is really sent; the test checks it was queued. |
| Simulated / stand-in | Shopify, Slack, email, file storage and PDF printing are replaced by stand-ins, so a test run never contacts outside services. |

## How to read a test case

The steps use these starting points:

- **Standard quotation**: one item (qty 1, rate 100, COGS 40), payment terms "100 % Advance". Saving it moves it from Draft to **Inquiry** and adds empty Shipping Charges and Custom Fee rows.
- **Staged quotation**: one item at rate 1,000 (COGS 400) with a 40 / 30 / 30 schedule = 400 / 300 / 300, stages Advance / Photo Approval / Before Dispatch unless the test says otherwise.
- **Ready-to-send quotation**: a standard quotation with a contact email and a customer address, moved to **Inquiry Approved**.
- **Sent quotation**: a ready-to-send quotation after **Send Quotation to Client** (state Sent to Customer, submitted).
- **Linked Lyfe Order**: a minimal Lyfe Order whose Source Quotation is the test quotation.
- **Record a payment**: add a received-payment row to the quotation (and, where the step says so, update Total Received Amount).
- **Jumped to [state]**: the test puts the quotation straight into that state to skip setup that other tests already cover. No workflow rules run for the jump itself.
- **Called directly**: a few tests run one server check on its own instead of clicking through the form, because the situation cannot be reached with a normal save. The class description says when.

Each test lists **What it checks** (the business rule in one sentence), the **Starting point**, the **Steps**, and the **Expected result**.

## Test case catalogue

All 149 test cases, grouped by module and test class. Each test is labelled **Positive** (the normal case works), **Negative** (bad input or a disallowed action is stopped, or something must not happen) or **Expected failure**.

### test\_lifecycle (45)

Walks quotations through the Quotation-3 workflow and checks every rule that allows or blocks a workflow action.

#### HappyPathTest

The normal sales journey from inquiry to payment.

**1. Full journey: Inquiry → Sent → Won → Cancelled** · Positive

`test_inquiry_to_won_to_cancelled`

- **What it checks:** A normal quotation can go through the whole sales cycle, and each action lands in the right state.
- **Steps:**
  1. Create a standard quotation.
  2. Add a contact and address, save, then click **Inquiry Approved**.
  3. Click **Send Quotation to Client**.
  4. Click **Bank Transfer Selected**, record a payment for the full grand total, then click **Confirm Payment**.
  5. Click **Cancel**.
- **Expected result:**
  - After step 1: state is **Inquiry**, and the taxes table has a Shipping Charges row and a Custom Fee row.
  - After step 2: state is **Inquiry Approved**; Approval Required is not ticked (total is under 10,000).
  - After step 3: state is **Sent to Customer** and the quotation is submitted.
  - After step 4: state is **Won**.
  - After step 5: state is **Cancelled** and the quotation is cancelled.

**2. Part payment first, then the balance** · Positive

`test_partial_payment_then_remaining`

- **What it checks:** A customer can pay in two parts: the first part makes the quotation Partially Paid, the balance makes it Won.
- **Starting point:** A sent quotation.
- **Steps:**
  1. Click **Bank Transfer Selected**.
  2. Record a payment for half the grand total, then click **Confirm Payment**.
  3. Record payments up to the full grand total, then click **Confirm Remaining Payment**.
- **Expected result:**
  - After step 2: state is **Partially Paid**.
  - After step 3: state is **Won**.

**3. Cannot send without customer email and address** · Negative

`test_sent_to_customer_needs_email_and_address`

- **What it checks:** A quotation cannot be sent to the customer until it has a contact email and a customer address.
- **Starting point:** A standard quotation moved to Inquiry Approved, with no contact and no address.
- **Steps:**
  1. Click **Send Quotation to Client**.
  2. Add a contact with an email (still no address), save, and click **Send Quotation to Client** again.
- **Expected result:**
  - Step 1 is blocked with a message about **Customer email**.
  - Step 2 is blocked with a message about **Customer address**.

#### PostSubmitHooksTest

Actions taken after the quotation is submitted (Sent to Customer and later).

**4. Cancelling the quotation releases the order's payment block** · Positive

`test_cancel_releases_payment_gates`

- **What it checks:** When a quotation is cancelled, its Lyfe Order is no longer held waiting for payment.
- **Starting point:** A sent quotation with a linked Lyfe Order that is blocked waiting for the Before Dispatch payment.
- **Steps:**
  1. Click **Cancel** on the quotation.
- **Expected result:**
  - The Lyfe Order's payment block is cleared.

**5. Marking the quotation Lost releases the order's payment block** · Positive

`test_lost_releases_payment_gates`

- **What it checks:** Same as the test above, for Lost instead of Cancel.
- **Starting point:** A sent quotation with a linked Lyfe Order that is blocked waiting for the Before Dispatch payment.
- **Steps:**
  1. Click **Lost** on the quotation.
- **Expected result:**
  - The Lyfe Order's payment block is cleared.

**6. Cannot mark Won when nothing has been paid** · Negative

`test_won_with_no_payment_is_blocked`

- **What it checks:** A quotation cannot become Won if no payment is recorded. The workflow itself has no amount check on this action, so this rule is the only guard.
- **Starting point:** A sent quotation jumped to **Payment Received**, with no payment rows.
- **Steps:**
  1. Click **Mark as Won**.
- **Expected result:**
  - Blocked with "no payment has been recorded".

**7. Full payment marks the Advance milestone Paid** · Positive

`test_won_marks_advance_milestone_paid`

- **What it checks:** When the quotation is paid in full, the Advance row in its payment schedule shows Paid.
- **Starting point:** A sent quotation.
- **Steps:**
  1. Click **Bank Transfer Selected**.
  2. Record a payment for the full grand total, then click **Confirm Payment**.
- **Expected result:**
  - State is **Won** and the Advance payment schedule row shows **Paid**.

#### ApprovalPathTest

Quotations of 10,000 or more need manager approval before they can be sent. Approval can happen in ERPNext or from a Slack card.

**8. Quotation of 10,000+ goes through approval, then can be sent** · Positive

`test_over_10k_needs_approval_then_sends`

- **What it checks:** A large quotation is flagged for approval and can only be sent after it is approved.
- **Starting point:** A ready-to-send quotation with one item at rate 12,000 (COGS 5,000).
- **Steps:**
  1. Check the Approval Required flag.
  2. Click **Send for Approval**.
  3. Click **Approve**.
  4. Click **Send Quotation to Client**.
- **Expected result:**
  - Step 1: **Approval Required** is ticked.
  - Step 2: state is **Pending Approval**.
  - Step 3: state is **Approved – Pending Send**, Approval Required is cleared, and a snapshot of the approved items (spec hash) is saved.
  - Step 4: state is **Sent to Customer**.

**9. Approver can send the quotation back to sales** · Positive

`test_send_back_to_sales_user`

- **What it checks:** The approver can return a quotation to the sales user instead of approving it.
- **Starting point:** A 12,000 quotation in Pending Approval.
- **Steps:**
  1. Click **Send Back to Sales User**.
- **Expected result:**
  - State is **Sales User Review**.

**10. Slack approval card is posted from Feasibility Approved** · Positive

`test_slack_card_from_feasibility_approved`

- **What it checks:** When a quotation goes for approval and Slack approval is switched on, an approval card is posted to Slack.
- **Starting point:** A 12,000 quotation jumped to **Feasibility Approved**.
- **Steps:**
  1. Turn **Enable Quotation Approval** on in LH Slack Settings.
  2. Click **Send for Approval**.
- **Expected result:**
  - A Slack approval card is queued for this quotation.

**11. No Slack card when Slack approval is switched off** · Negative

`test_slack_card_off_when_setting_disabled`

- **What it checks:** The Slack setting really switches the card off.
- **Starting point:** A 12,000 quotation jumped to **Feasibility Approved**.
- **Steps:**
  1. Turn **Enable Quotation Approval** off.
  2. Click **Send for Approval**.
- **Expected result:**
  - No Slack card is queued.

**12. Fee-only quotation can go for approval without COGS (from Inquiry Approved)** · Positive

`test_fee_only_quote_sent_for_approval_from_inquiry_approved`

- **What it checks:** A quotation whose price is carried only by the Custom Fee (item at rate 0, no COGS) can still be sent for approval.
- **Starting point:** A fee-only quotation: one item at rate 0 with no COGS, and a Custom Fee of 12,000; moved to Inquiry Approved.
- **Steps:**
  1. Check the flags.
  2. Click **Send for Approval**.
- **Expected result:**
  - Step 1: **COGS Pending** and **Approval Required** are both ticked.
  - Step 2: state is **Pending Approval** (COGS is not required for Send for Approval).

**13. Fee-only quotation cannot go for approval from Sales User Review while COGS is pending** · Negative

`test_fee_only_quote_blocked_from_sales_user_review`

- **What it checks:** From Sales User Review, Send for Approval needs COGS. The workflow condition on this route is `doc.cogs_pending == 0 and doc.approval_required == 1`, kept on purpose.
- **Starting point:** The same fee-only quotation (COGS Pending ticked), jumped to **Sales User Review**.
- **Steps:**
  1. Click **Send for Approval**.
- **Expected result:**
  - Blocked with **Not a valid Workflow Action**; the action is not offered.

**14. Quotation with COGS can go for approval from Sales User Review** · Positive

`test_quote_with_cogs_sent_for_approval_from_sales_user_review`

- **What it checks:** Once COGS is filled in, Send for Approval from Sales User Review works.
- **Starting point:** A ready-to-send quotation at 12,000 with COGS 5,000, jumped to **Sales User Review**.
- **Steps:**
  1. Click **Send for Approval**.
- **Expected result:**
  - State is **Pending Approval**.

**15. No COGS needed to send for approval from Feasibility Approved** · Positive

`test_no_cogs_needed_from_feasibility_approved`

- **What it checks:** Send for Approval from Feasibility Approved does not require COGS.
- **Starting point:** A ready-to-send quotation at 12,000 with COGS 0, jumped to **Feasibility Approved**.
- **Steps:**
  1. Click **Send for Approval**.
- **Expected result:**
  - State is **Pending Approval**.

**16. One Slack card, whichever way the quotation reaches Pending Approval** · Positive

`test_slack_card_from_every_route`

- **What it checks:** There are three ways into Pending Approval; each must post exactly one Slack card, and only after the save is fully committed (so a later error in the same save cancels the card).
- **Steps:**
  1. With Slack approval on, take three quotations into Pending Approval:
     - a) 12,000 quotation: **Send for Approval** from Inquiry Approved.
     - b) 12,000 quotation with COGS: **Send for Approval** from Sales User Review.
     - c) 12,000 quotation with Free Shipping: **Feasibility Approve** from Production Details Review.
- **Expected result:**
  - Each route queues one card, set to post after the save commits.

**17. No second Slack card when a pending quotation is saved again** · Negative

`test_no_second_card_while_already_pending`

- **What it checks:** Saving a quotation that is already waiting for approval does not post a duplicate card.
- **Starting point:** A 12,000 quotation already in Pending Approval (its card was posted).
- **Steps:**
  1. With Slack approval on, save the quotation again.
- **Expected result:**
  - No second card is queued.

**18. Approving in ERPNext closes the Slack card** · Positive

`test_erp_approve_closes_slack_card`

- **What it checks:** If the approver approves in ERPNext, the Slack card is closed so nobody acts on it later.
- **Starting point:** A 12,000 quotation in Pending Approval.
- **Steps:**
  1. Click **Approve** in ERPNext.
- **Expected result:**
  - A job to close the Slack card is queued.

**19. Approving from Slack does not close the card twice** · Negative

`test_slack_approve_does_not_double_close_card`

- **What it checks:** When the approval comes from the Slack button, Slack updates its own card, so ERPNext must not also close it.
- **Starting point:** A 12,000 quotation in Pending Approval.
- **Steps:**
  1. Approve the quotation from Slack.
- **Expected result:**
  - ERPNext does not queue a close job.

**20. Slack approval refused by ERPNext is explained in the Slack thread** · Negative

`test_slack_approve_rejected_by_erp_replies_in_thread`

- **What it checks:** If the approver clicks Approve in Slack but ERPNext refuses (for example COGS is missing), the approver is told why and can try again.
- **Starting point:** A 12,000 quotation in Pending Approval with an open Slack approval thread.
- **Steps:**
  1. The approver clicks **Approve** on the Slack card; ERPNext refuses with "COGS missing".
- **Expected result:**
  - A reply is posted in the same Slack thread containing "Reason: COGS missing".
  - The card's buttons are not removed and the thread stays open, so the approver can retry after the fix.

#### EngineeringBranchTest

Quotations that need a drawing or a feasibility check from the engineer / factory. Several tests start from an **engineer review quotation**: a quotation with Drawing Required ticked, moved Inquiry Approved → Request for Technical Details, so it is in **Pending Engineer Review** with no drawing attached.

**21. Drawing Required sends the quotation to the engineer, then back to sales** · Positive

`test_drawing_required_routes_to_engineer_then_sales`

- **What it checks:** A quotation that needs a drawing goes to the engineer, and returns to sales once the drawing is attached.
- **Starting point:** A standard quotation with **Drawing Required** ticked.
- **Steps:**
  1. Click **Inquiry Approved**, then **Request for Technical Details**.
  2. Attach the Order Drawing with Drawing Change Details filled, and save.
  3. Click **Completed**.
- **Expected result:**
  - Step 1: state is **Pending Engineer Review**.
  - Step 2: **Drawing Required** is unticked automatically.
  - Step 3: state is **Sales User Review**.

**22. Engineer cannot complete without attaching the drawing** · Negative

`test_completed_blocked_without_drawing`

- **What it checks:** Completed is blocked when no drawing is attached. This runs on the server, the same way as a list-view bulk action or an API call, so it proves the rule does not depend on the browser.
- **Starting point:** An engineer review quotation.
- **Steps:**
  1. Click **Completed**.
- **Expected result:**
  - Blocked with a message about the **Drawing**.

**23. Completed is also blocked when it would go to Production Details Review** · Negative

`test_completed_to_production_review_blocked_without_drawing`

- **What it checks:** The drawing rule applies on the other Completed route too (Feasibility Test Required sends the quotation to Production Details Review).
- **Starting point:** An engineer review quotation with **Feasibility Test Required** ticked.
- **Steps:**
  1. Click **Completed**.
- **Expected result:**
  - Blocked with a message about the **Drawing**.

**24. Item Photo Required also needs the drawing** · Negative

`test_completed_blocked_by_item_photo_required`

- **What it checks:** Photo Required on any item line counts the same as Drawing Required.
- **Starting point:** An engineer review quotation with Drawing Required unticked but **Photo Required** ticked on an item.
- **Steps:**
  1. Click **Completed**.
- **Expected result:**
  - Blocked with a message about the **Drawing**.

**25. Urgent quotations can complete without a drawing** · Positive

`test_urgent_completes_without_drawing`

- **What it checks:** Urgent quotations are exempt from the drawing rule.
- **Starting point:** An engineer review quotation with Priority **Urgent**.
- **Steps:**
  1. Click **Completed**.
- **Expected result:**
  - Allowed; state is **Sales User Review**.

**26. Need More Information is not blocked by the drawing rule** · Positive

`test_need_more_information_not_blocked`

- **What it checks:** Only the forward action needs the drawing; sending the quotation back for more information is always allowed.
- **Starting point:** An engineer review quotation.
- **Steps:**
  1. Click **Need More Information**.
- **Expected result:**
  - Allowed; state is **Draft**.

**27. Ticking Feasibility Test Required moves the quotation to Production Details Review** · Positive

`test_feasibility_flag_auto_advances_to_production_review`

- **What it checks:** From Sales User Review, ticking the flag routes the quotation to the factory automatically.
- **Starting point:** A standard quotation jumped to **Sales User Review**.
- **Steps:**
  1. Tick **Feasibility Test Required** and save.
- **Expected result:**
  - State changes to **Production Details Review**.

**28. Ticking Feasibility Test Required in Inquiry goes straight to Production Details Review** · Positive

`test_feasibility_flag_in_inquiry_skips_inquiry_approved`

- **What it checks:** From Inquiry, the flag skips Inquiry Approved and routes straight to the factory.
- **Starting point:** A standard quotation in **Inquiry**.
- **Steps:**
  1. Tick **Feasibility Test Required** and save.
- **Expected result:**
  - State changes to **Production Details Review**.

**29. Feasibility flag clears itself once COGS and shipping are filled** · Positive

`test_feasibility_flag_auto_clears_when_cogs_and_shipping_set`

- **What it checks:** When the factory has entered everything needed (COGS and shipping), the Feasibility Test Required flag is unticked automatically.
- **Starting point:** A standard quotation (COGS 40) jumped to **Production Details Review**.
- **Steps:**
  1. Tick **Feasibility Test Required** and **Free Shipping**, and save.
- **Expected result:**
  - **Feasibility Test Required** is unticked.

**30. Feasibility Approve needs a shipping charge unless Free Shipping is ticked** · Negative

`test_production_review_needs_shipping_unless_free`

- **What it checks:** The factory cannot approve feasibility without either a shipping charge or Free Shipping.
- **Starting point:** A standard quotation in **Production Details Review** with no shipping charge.
- **Steps:**
  1. Click **Feasibility Approve**.
  2. Tick **Free Shipping**, save, and click **Feasibility Approve** again.
- **Expected result:**
  - Step 1 is blocked with a message about **Shipping Charges**.
  - Step 2: state is **Feasibility Approved**.

**31. Feasibility Approve always needs COGS** · Negative

`test_feasibility_approve_needs_cogs_whatever_the_target`

- **What it checks:** COGS is required for Feasibility Approve whether the next state is Feasibility Approved (under 10,000) or Pending Approval (10,000+).
- **Steps:**
  1. Create two quotations with COGS 0 and Free Shipping, at rates 100 and 12,000, both in Production Details Review.
  2. Click **Feasibility Approve** on each.
- **Expected result:**
  - Both are blocked with a message about **Cost of Goods**.

**32. Feasibility Approve on a 10,000+ quotation goes to Pending Approval** · Positive

`test_feasibility_approve_over_10k_goes_to_pending_approval`

- **What it checks:** A large quotation skips Feasibility Approved and goes straight to approval.
- **Starting point:** A 12,000 quotation (COGS 5,000) with Free Shipping, jumped to **Production Details Review**.
- **Steps:**
  1. Click **Feasibility Approve**.
- **Expected result:**
  - State is **Pending Approval**.

**33. Photo Required in Production Details Review sends it back to the engineer** · Positive

`test_photo_required_in_production_review_routes_back_to_engineer`

- **What it checks:** If the factory asks for a photo, the quotation returns to the engineer.
- **Starting point:** A standard quotation jumped to **Production Details Review**.
- **Steps:**
  1. Tick **Photo Required** on an item and save.
- **Expected result:**
  - State changes to **Pending Engineer Review**.

**34. Senior Review is only for Custom orders** · Negative

`test_request_senior_review_only_for_custom_orders`

- **What it checks:** The Request Senior Review action is not available for Standard orders.
- **Starting point:** An engineer review quotation whose Order Type is **Standard**.
- **Steps:**
  1. Click **Request Senior Review**.
- **Expected result:**
  - Rejected; the action is not available.

#### RejectionBranchTest

Rejecting an inquiry or marking it not feasible.

**35. Inquiry Rejected needs a reason** · Negative

`test_inquiry_rejected_needs_reason`

- **What it checks:** An inquiry cannot be rejected without saying why.
- **Starting point:** A standard quotation in Inquiry.
- **Steps:**
  1. Click **Inquiry Rejected** with the reason empty.
  2. Fill Inquiry Rejected Reason ("Out of scope"), save, and click **Inquiry Rejected** again.
- **Expected result:**
  - Step 1 is blocked with a message about **Rejected Reason**.
  - Step 2: state is **Inquiry Rejected**.

**36. Not Feasible needs a reason, then goes to CS review** · Negative

`test_not_feasible_needs_reason_then_cs_review`

- **What it checks:** The factory must give a reason for Not Feasible, and Customer Service can then pick it up.
- **Starting point:** A standard quotation jumped to **Production Details Review**.
- **Steps:**
  1. Click **Not Feasible** with the reason empty.
  2. Fill Not Feasible Reason ("Cannot machine"), save, and click **Not Feasible** again.
  3. Click **CS Review**.
- **Expected result:**
  - Step 1 is blocked with a message about **Not Feasible Reason**.
  - Step 2: state is **Not Feasible**.
  - Step 3: state is **Sales User Review**.

**37. Only the factory can change the Not Feasible Reason** · Negative

`test_not_feasible_reason_locked_to_factory`

- **What it checks:** A sales user cannot overwrite the factory's Not Feasible Reason. Only Factory, System Manager and Super Admin can change it.
- **Starting point:** A standard quotation with Not Feasible Reason "original".
- **Steps:**
  1. As a user with only the Sales User role, change the reason to "edited by sales" and save.
- **Expected result:**
  - The save goes through, but the reason stays **"original"**.

#### BeforeWorkflowActionTest

The checks that run in the form when a user clicks a workflow button. The test calls these checks directly.

**38. Workflow button checks in the form** · Negative

`test_guards`

- **What it checks:** Each forward action is blocked when its requirement is missing; closing actions are never blocked.
- **Steps:**
  1. Pending Engineer Review, Drawing Required, no drawing → **Completed**.
  2. Same, with Priority Urgent → **Completed**.
  3. Production Details Review, COGS 0, Priority Normal → **Feasibility Approve**.
  4. Same → **Send for Approval**.
  5. Approval Required ticked → **Send Quotation to Client**.
  6. → **Lost**.
- **Expected result:**
  - Step 1: blocked (**Drawing**).
  - Step 2: allowed (Urgent is exempt).
  - Step 3: blocked (**Cost of Goods**).
  - Step 4: allowed (no COGS check on Send for Approval).
  - Step 5: blocked (**requires approval**).
  - Step 6: allowed (closing actions are never checked).

#### RevisionTest

When items change on a quotation that has already been sent, it becomes a new revision (Rev 1, Rev 2, Rev 3). A sent quotation is submitted, so these tests call the revision logic directly; the last test in this class proves a normal save cannot change items.

**39. Changing items creates a revision and refreshes the payment link** · Positive

`test_items_change_bumps_revision_and_requeues_shopify`

- **What it checks:** A change to the items of a sent quotation creates Rev 1, restarts the follow-up cycle and updates the Shopify payment link.
- **Starting point:** A sent quotation whose day-3 follow-up has already been sent.
- **Steps:**
  1. Change the item rate from 100 to 90 and run the revision logic.
- **Expected result:**
  - Revision number is **1**, label **"Rev 1"**.
  - Follow-up history is reset, so the revised quotation gets a fresh follow-up cycle.
  - A Shopify draft order update is queued (the payment link shows the new amount).

**40. No item change, no revision** · Negative

`test_no_item_change_no_revision`

- **What it checks:** Saving a sent quotation without changing its items does not create a revision.
- **Starting point:** A sent quotation.
- **Steps:**
  1. Run the revision logic without changing anything.
- **Expected result:**
  - No revision number; no Shopify update queued.

**41. No more changes after Rev 3** · Negative

`test_rev3_blocks_further_changes`

- **What it checks:** A quotation can be revised at most 3 times.
- **Starting point:** A sent quotation already at revision 3.
- **Steps:**
  1. Change the item qty to 5 and run the revision logic.
- **Expected result:**
  - Blocked with "revised 3 times".

**42. Unsent quotations are not revised** · Negative

`test_revision_ignored_outside_sent_to_customer`

- **What it checks:** Only quotations that have been sent get revision numbers.
- **Starting point:** A standard quotation in Inquiry (not sent).
- **Steps:**
  1. Change the item rate to 50 and run the revision logic.
- **Expected result:**
  - No revision number.

**43. Items cannot be edited with a normal save after sending** · Negative

`test_item_edit_after_send_is_rejected`

- **What it checks:** Once sent (submitted), item lines are locked by ERPNext.
- **Starting point:** A sent quotation.
- **Steps:**
  1. Change the item rate to 90 and save.
- **Expected result:**
  - Rejected by ERPNext: items cannot be changed after submit.

#### ShopifyLinkTest

Linking an existing Shopify order to a quotation by pasting its link.

**44. Pasting a Shopify order link starts the linking job** · Positive

`test_changed_shopify_link_enqueues_linking`

- **What it checks:** A valid Shopify admin order link is accepted and the order is linked in the background.
- **Starting point:** A standard quotation.
- **Steps:**
  1. Paste `https://admin.shopify.com/store/x/orders/123` into **Shopify Order Link** and save.
- **Expected result:**
  - A job to link the Shopify order is queued.

**45. A non-Shopify link is rejected** · Negative

`test_invalid_shopify_link_rejected`

- **What it checks:** Only Shopify admin order or draft-order links are accepted.
- **Starting point:** A standard quotation.
- **Steps:**
  1. Paste `https://example.com/not-shopify` into **Shopify Order Link** and save.
- **Expected result:**
  - Blocked with a message about the **Shopify Order Link**.

### test\_payments (27)

Payment schedule, processing fee, payment stages that hold the Lyfe Order, milestone statuses and payment reminders.

#### PaymentScheduleTest

The payment schedule splits the total into stages (for example 40 / 30 / 30).

**1. Schedule amounts follow the quotation total** · Positive

`test_amounts_follow_grand_total`

- **What it checks:** When the total changes, every schedule row is recalculated from its percentage.
- **Starting point:** A staged quotation at 1,000.
- **Steps:**
  1. Change the item rate from 1,000 to 2,000 and save.
- **Expected result:**
  - Before: amounts are **400 / 300 / 300**.
  - After: amounts are **800 / 600 / 600**.

**2. A schedule row of 0% is rejected** · Negative

`test_zero_portion_rejected`

- **What it checks:** Every schedule row must carry part of the total.
- **Starting point:** A staged quotation.
- **Steps:**
  1. Set row 2 to 0% and validate the schedule.
- **Expected result:**
  - Blocked with "zero percentage".

**3. A schedule row without a stage is rejected** · Negative

`test_row_without_stage_rejected`

- **What it checks:** Every schedule row must say when it is due (Advance, Photo Approval, Before Dispatch, Before Shipped or On Delivery).
- **Starting point:** A staged quotation.
- **Steps:**
  1. Untick **Before Dispatch** on row 3 (so it has no stage) and save.
- **Expected result:**
  - Blocked with "no trigger event".

**4. Schedule amounts must add up to the total** · Negative

`test_amounts_must_sum_to_total`

- **What it checks:** The rows together must equal the quotation total.
- **Starting point:** A staged quotation at 1,000.
- **Steps:**
  1. Change row 3 to 250 (rows now add up to 950, not 1,000) and validate the schedule.
- **Expected result:**
  - Blocked with "sum to".

**5. Stage ticks come from the Payment Terms Template and can be changed** · Positive

`test_stage_checkboxes_copied_from_template`

- **What it checks:** The stage ticked on a Payment Terms Template is copied onto the quotation, and a manual change on the quotation is kept.
- **Steps:**
  1. Create a Payment Terms Template with one 100% term and **Before Dispatch** ticked.
  2. Create a quotation that uses this template.
  3. On the schedule row, untick Before Dispatch, tick **On Delivery**, and save.
- **Expected result:**
  - Step 2: the schedule row has **Before Dispatch** ticked.
  - Step 3: the manual change is kept (On Delivery ticked, Before Dispatch not).

#### ProcessingFeeTest

The optional card processing fee added as a separate tax row.

**6. Processing fee is added once, not stacked, and removed cleanly** · Positive

`test_fee_added_removed_and_not_compounded`

- **What it checks:** Ticking Apply Additional Charges adds a fee row of the set percentage; saving again does not add the fee on top of the fee; unticking removes it.
- **Steps:**
  1. Set the processing fee to 3% in Payment Fee Settings and create a standard quotation.
  2. Tick **Apply Additional Charges** and save.
  3. Save again.
  4. Untick **Apply Additional Charges** and save.
- **Expected result:**
  - Step 1: no fee row.
  - Step 2: a Payment Processing Fee row of **3% of the total**; grand total goes up 3%.
  - Step 3: the fee is still 3% (not 3% of the already-increased total).
  - Step 4: the fee row is gone and the grand total is back to the original.

#### StagePaymentEnforcementTest

When a Lyfe Order reaches Factory Assignment, Approved or Ready for Dispatch, the payment for that stage must already be received. Amounts are cumulative: Before Dispatch on a 40 / 30 / 30 schedule needs 100% paid. The test calls this check directly, with Ready for Dispatch as the Lyfe Order step.

**7. Unpaid stage blocks the Lyfe Order and sends a reminder** · Negative

`test_underpaid_blocks_and_tracks_on_lyfe_order`

- **What it checks:** If the stage is not paid, the order step is blocked, the block is recorded on the Lyfe Order, and a reminder goes out.
- **Starting point:** A staged quotation at 1,000 with a linked Lyfe Order.
- **Steps:**
  1. Record a payment of 400 (Advance only).
  2. Check the **Before Dispatch** stage (needs 1,000 paid).
- **Expected result:**
  - Blocked with "Payment required".
  - The Lyfe Order shows the block stage (Before Dispatch) and the block date.
  - A payment reminder is queued for the Lyfe Order.

**8. A repeated block does not send a second reminder** · Negative

`test_repeat_block_does_not_re_notify`

- **What it checks:** Retrying a blocked step blocks again but does not spam the customer.
- **Starting point:** A staged quotation with a linked Lyfe Order and no payment.
- **Steps:**
  1. Check the **Before Dispatch** stage twice.
- **Expected result:**
  - Blocked both times; only **one** reminder is queued.

**9. Stage passes once the running total is paid** · Positive

`test_cumulative_threshold_met_passes`

- **What it checks:** Photo Approval needs Advance + Photo Approval (400 + 300 = 700) paid.
- **Starting point:** A staged quotation at 1,000.
- **Steps:**
  1. Record a payment of 700 and check the **Photo Approval** stage.
- **Expected result:**
  - Allowed.

**10. Non-CS users see only a general payment message** · Negative

`test_non_cs_user_gets_generic_message`

- **What it checks:** Users outside Customer Service (for example Factory) see that payment is pending, without amounts or customer details.
- **Starting point:** A staged quotation with no payment.
- **Steps:**
  1. As a Factory user, check the **Advance** stage.
- **Expected result:**
  - Blocked with the general message "payment is pending".

**11. A stage that is not in the schedule is never checked** · Negative

`test_unconfigured_stage_is_noop`

- **What it checks:** If the schedule has no row for a stage, that stage never blocks.
- **Starting point:** A staged quotation (Advance / Photo Approval / Before Dispatch).
- **Steps:**
  1. Check the **On Delivery** stage (not in this schedule).
- **Expected result:**
  - Allowed; nothing happens.

#### EnforcePaymentStateTest

The check that runs when a quotation becomes Won. The test calls it directly.

**12. Fully paid quotation stays Won** · Positive

`test_fully_paid_stays_won`

- **What it checks:** A quotation paid in full keeps the Won state.
- **Starting point:** A standard quotation.
- **Steps:**
  1. Record a payment for the full total, set the state to Won, and run the check.
- **Expected result:**
  - State stays **Won**.

**13. Underpaid quotation is changed to Partially Paid** · Positive

`test_underpaid_becomes_partially_paid`

- **What it checks:** If only part is paid, Won is corrected to Partially Paid and the received total is updated.
- **Starting point:** A standard quotation.
- **Steps:**
  1. Record a payment of 10, set the state to Won, and run the check.
- **Expected result:**
  - State changes to **Partially Paid**; Total Received Amount is **10**.

**14. Unpaid quotation cannot be Won** · Negative

`test_unpaid_is_rejected`

- **What it checks:** Won is refused when nothing is paid.
- **Starting point:** A standard quotation.
- **Steps:**
  1. With no payment, set the state to Won and run the check.
- **Expected result:**
  - Blocked with "no payment has been recorded".

#### PaymentGateReleaseTest

Releasing Lyfe Order payment blocks when a quotation closes. The test calls the release logic directly.

**15. Lost or Cancelled clears the order's block and reminder count** · Positive

`test_lost_or_cancelled_clears_block`

- **What it checks:** Once a quotation is Lost or Cancelled, its Lyfe Order is not held for payment and reminders stop.
- **Starting point:** A standard quotation with a linked Lyfe Order blocked at Before Dispatch, 3 reminders already sent.
- **Steps:**
  1. For **Lost** and then **Cancelled**: move the quotation from Sent to Customer to that state.
- **Expected result:**
  - The Lyfe Order block is cleared and its reminder count is **0**.

**16. Other state changes keep the block** · Negative

`test_non_terminal_state_keeps_block`

- **What it checks:** Only Lost and Cancelled release the block.
- **Starting point:** A standard quotation with a linked Lyfe Order blocked at Before Dispatch.
- **Steps:**
  1. Move the quotation from Sent to Customer to **Won**.
- **Expected result:**
  - The Lyfe Order block stays.

#### MilestoneStatusTest

Each schedule row shows a status: **Upcoming** (not due yet), **Due** (payment expected now), **Paid**, **Overdue** (reminders sent, still unpaid) or **Escalated** (many reminders, still unpaid).

**17. Milestone statuses move from Upcoming to Due to Paid** · Positive

`test_status_progression`

- **What it checks:** Statuses update as the quotation and Lyfe Order progress and as payments arrive.
- **Starting point:** A staged quotation at 1,000.
- **Steps:**
  1. Update the statuses.
  2. Set the quotation to Partially Paid and update.
  3. Record 400 and update, with the Lyfe Order in Approved.
- **Expected result:**
  - Step 1: **Upcoming / Upcoming / Upcoming**.
  - Step 2: **Due / Upcoming / Upcoming**, and a "payment due" notification is queued for Advance.
  - Step 3: **Paid / Due / Upcoming**.

**18. Overdue and Escalated follow the number of reminders sent** · Positive

`test_overdue_and_escalated_follow_reminder_count`

- **What it checks:** An unpaid milestone becomes Overdue after reminders, then Escalated after more reminders.
- **Starting point:** A staged quotation in Won with no payment, and a linked Lyfe Order.
- **Steps:**
  1. Update the statuses (Lyfe Order has 2 reminders sent).
  2. Raise the reminder count to 5 and update.
- **Expected result:**
  - Step 1: the first milestone is **Overdue**.
  - Step 2: the first milestone is **Escalated**.

**19. The "payment due" notification is sent only once** · Positive

`test_due_notification_only_on_fresh_due`

- **What it checks:** Updating statuses twice does not repeat the Due notification.
- **Starting point:** A staged quotation in Won.
- **Steps:**
  1. Update the statuses twice.
- **Expected result:**
  - Only **one** Due notification is queued.

#### NoRemindersWhenClosedTest

No payment reminders for closed quotations. For these tests reminders really run: email is captured instead of sent, and Slack is replaced by a stand-in.

**20. No block reminder for a Lost quotation** · Negative

`test_block_reminder_skipped_and_cleared_for_lost`

- **What it checks:** Even when a quotation is closed by the follow-up scheduler (which skips the normal save rules), the customer gets no more reminders.
- **Starting point:** A staged quotation with a linked Lyfe Order blocked at Before Dispatch, 2 reminders sent.
- **Steps:**
  1. Set the quotation to **Lost** directly (as the follow-up scheduler does).
  2. Send the payment block reminder.
- **Expected result:**
  - No email is sent.
  - The Lyfe Order block and reminder count are cleared.

**21. Block reminder still goes out for an open quotation** · Positive

`test_block_reminder_still_sent_for_open_quotation`

- **What it checks:** Control test: the reminder works when the quotation is still open, so the test above proves something real.
- **Starting point:** Same as above, quotation left open.
- **Steps:**
  1. Send the payment block reminder.
- **Expected result:**
  - **One** email is sent.

**22. No Due or Before Shipped notification for a Cancelled quotation** · Negative

`test_due_and_stage_notifications_skipped_for_cancelled`

- **What it checks:** Cancelled quotations get no payment notifications of any kind.
- **Starting point:** A staged quotation (Advance / Before Dispatch / Before Shipped) with a linked Lyfe Order.
- **Steps:**
  1. Set the quotation to **Cancelled**.
  2. Send the Advance Due notification and the Before Shipped notification.
- **Expected result:**
  - No email is sent.

#### PartialPaymentDueTest

Customer Service records a bank transfer through the **Confirm Payment** dialog.

**23. Part payment marks Advance as Due and reminds the customer** · Positive

`test_partial_payment_marks_advance_due_and_reminds`

- **What it checks:** A partial bank transfer moves the quotation to Partially Paid and starts the Advance reminder.
- **Starting point:** A staged quotation at 1,000 with contact and address, sent, then Bank Transfer Selected.
- **Steps:**
  1. In the Confirm Payment dialog, enter **200** as a **Partial Payment** (not confirmed in Shopify).
- **Expected result:**
  - State is **Partially Paid**.
  - Milestones are **Due / Upcoming / Upcoming**.
  - An Advance Due notification is queued.

**24. Full payment marks every milestone Paid** · Positive

`test_full_payment_marks_all_paid`

- **What it checks:** Paying the whole amount wins the quotation and marks all stages Paid.
- **Starting point:** Same as above.
- **Steps:**
  1. In the Confirm Payment dialog, enter the full total as a **Full Payment**.
- **Expected result:**
  - State is **Won**; all milestones are **Paid**.

#### ReminderOnlyStagesTest

**Before Shipped** and **On Delivery** stages only send reminders; they must never block the Lyfe Order.

**25. Unpaid Before Shipped does not block dispatch** · Negative

`test_unpaid_shipped_stage_does_not_block_dispatch`

- **What it checks:** Dispatch is allowed once Advance and Before Dispatch are paid, even if Before Shipped is not.
- **Starting point:** A staged quotation: Advance 400 / Before Dispatch 300 / Before Shipped 300.
- **Steps:**
  1. Record 700 and check the **Before Dispatch** stage.
- **Expected result:**
  - Allowed, even though Before Shipped (300) is unpaid.

**26. Unpaid Before Shipped sends a reminder but sets no block** · Negative

`test_unpaid_shipped_stage_sends_reminder`

- **What it checks:** The customer is reminded about the Before Shipped payment, but the order is not held.
- **Starting point:** Same staged quotation, with a linked Lyfe Order.
- **Steps:**
  1. Record 700 and send the Before Shipped notification.
- **Expected result:**
  - **One** email is sent; the Lyfe Order has **no** payment block.

**27. Lyfe Order never checks the reminder-only stages** · Negative

`test_order_code_never_hard_blocks_reminder_only_stages`

- **What it checks:** A safety check on the Lyfe Order code: none of its payment checks use Before Shipped or On Delivery.
- **Steps:**
  1. Scan the Lyfe Order code for every payment check.
- **Expected result:**
  - Payment checks are found, and **none** of them use Before Shipped or On Delivery.

### test\_integrations (36)

Quotation email, email-open tracking, the customer approval link, the Shopify payment link and payment checks, follow-ups, SLA, the COGS benchmark, list permissions and the dashboard.

#### SendQuotationEmailTest

Sending the quotation email. Outgoing email is switched on for these tests; nothing is actually sent.

**1. No email is sent when there is no outgoing email account** · Negative

`test_no_outgoing_account_skips`

- **What it checks:** If the site has no working outgoing email account, the send is skipped cleanly.
- **Starting point:** A sent quotation.
- **Steps:**
  1. Disable the outgoing email account and send the quotation email.
- **Expected result:**
  - Result is **"no_email_account"**; nothing is sent.

**2. Quotation email is sent with the payment link and tracking** · Positive

`test_sends_branded_email_and_tracks`

- **What it checks:** The email goes to the customer with the card payment link, and the quotation records that it was sent.
- **Starting point:** A sent quotation.
- **Steps:**
  1. With the payment link on, send the quotation email.
- **Expected result:**
  - Result is **"sent"** and the email includes the **card payment link**.
  - The email goes to the contact's email and is linked to the quotation.
  - **Email Sent**, **Email Sent At** and the tracking token are saved.
  - A Shopify payment check is queued.

**3. Bank transfer quotations never get a card link** · Negative

`test_bank_transfer_never_gets_card_link`

- **What it checks:** If the customer pays by bank transfer, the email shows only the bank transfer link.
- **Starting point:** A sent quotation with **Payment via Bank Transfer** ticked.
- **Steps:**
  1. With the payment link on, send the quotation email.
- **Expected result:**
  - The **bank transfer** link is included; **no card link**.

**4. Email cannot be sent without a contact email** · Negative

`test_requires_contact_email`

- **What it checks:** The send is refused when there is no customer email to send to.
- **Starting point:** A standard quotation with no contact.
- **Steps:**
  1. Send the quotation email.
- **Expected result:**
  - Blocked with "No contact email".

#### EmailOpenTrackingTest

The email contains an invisible 1×1 image (tracking pixel). When the customer opens the email, the image loads and the quotation records the open.

**5. Email open is recorded once** · Positive

`test_pixel_records_first_open_only`

- **What it checks:** The first open is recorded; later opens do not change the time.
- **Starting point:** A sent quotation.
- **Steps:**
  1. Load the tracking image with a valid token.
  2. Load it again.
- **Expected result:**
  - Step 1: **Email Opened** is ticked, **Opened At** is set, and an image is returned.
  - Step 2: **Opened At** does not change.

**6. An invalid tracking token records nothing** · Negative

`test_bad_token_records_nothing`

- **What it checks:** A wrong or made-up link cannot mark the email as opened.
- **Starting point:** A sent quotation.
- **Steps:**
  1. Load the tracking image with an invalid token.
- **Expected result:**
  - **Email Opened** stays unticked.

#### CustomerApprovalLinkTest

The **Approve Quotation** button in the customer email. Each link carries a secret token for that quotation.

**7. Customer approves with a valid link** · Positive

`test_valid_token_marks_won`

- **What it checks:** Clicking the approval link marks the quotation Won; clicking again just says it is already approved.
- **Starting point:** A sent quotation.
- **Steps:**
  1. Open the approval link with the correct token.
  2. Open it again.
- **Expected result:**
  - Step 1: an **"Approved"** page; state is **Won**.
  - Step 2: an **"Already Approved"** page.

**8. Forged or missing approval link is rejected** · Negative

`test_bad_or_missing_token_rejected`

- **What it checks:** Nobody can approve a quotation without the correct link.
- **Starting point:** A sent quotation.
- **Steps:**
  1. Open the approval link with a made-up token.
  2. Open it with no token.
- **Expected result:**
  - Step 1: **"Invalid or Expired Link"** page.
  - Step 2: **"Invalid Link"** page.
  - State stays **Sent to Customer**.

**9. Approval link does not work on a Lost, Cancelled or Expired quotation** · Negative

`test_closed_quotation_not_approved`

- **What it checks:** A customer cannot revive a closed quotation by clicking an old approval link.
- **Steps:**
  1. For **Lost**, **Cancelled** and **Expired**: move a sent quotation to that state, then open the approval link with the correct token.
- **Expected result:**
  - **"Quotation No Longer Valid"** page.
  - State is unchanged (not Won).

**10. Approval link does not work on a cancelled document** · Negative

`test_cancelled_docstatus_not_approved`

- **What it checks:** Even if the state label was not updated, a cancelled document cannot be approved.
- **Starting point:** A sent quotation.
- **Steps:**
  1. Cancel the quotation document (state left as Sent to Customer), then open the approval link with the correct token.
- **Expected result:**
  - **"Quotation No Longer Valid"** page; state is not changed to Won.

#### PaymentLinkRetryTest

Creating or updating the Shopify payment link. If Shopify fails, the system waits 3 seconds and tries again, up to 5 attempts in total. Shopify's replies are simulated and the waits are skipped in the test.

**11. Payment link is retried until it succeeds** · Positive

`test_retries_until_link_generated`

- **What it checks:** A temporary Shopify problem does not stop the payment link from being created.
- **Steps:**
  1. Shopify times out, then returns a server error (500), then succeeds.
- **Expected result:**
  - The successful reply is used on the **3rd** attempt.
  - There is a **3-second** wait before each retry.

**12. Payment link stops after 5 failed attempts** · Negative

`test_gives_up_after_5_failed_attempts`

- **What it checks:** If Shopify keeps refusing, the system stops after 5 attempts and reports the error.
- **Steps:**
  1. Shopify rejects the request (422) every time.
- **Expected result:**
  - Stops after **5** attempts with **4** waits of 3 seconds; the error is returned.

**13. Payment link stops after 5 network errors** · Negative

`test_network_error_every_attempt_raises`

- **What it checks:** If Shopify cannot be reached at all, the system stops after 5 attempts and shows the error.
- **Steps:**
  1. Shopify times out every time.
- **Expected result:**
  - Stops after **5** attempts; the timeout error is raised.

**14. Generate Payment Link saves the link after retrying** · Positive

`test_create_draft_order_stores_link_after_retry`

- **What it checks:** The full Generate Payment Link action saves the link on the quotation even when Shopify fails at first, and no failure alert goes out.
- **Starting point:** A sent quotation.
- **Steps:**
  1. With the payment link on, generate the payment link while Shopify returns 503, then 429 (rate limit), then a draft order with a payment URL.
- **Expected result:**
  - The payment URL is returned and saved on the quotation.
  - No failure alert is sent.

**15. Each Shopify attempt waits at most 20 seconds** · Positive

`test_each_attempt_waits_at_most_20s`

- **What it checks:** Each attempt gives up after 20 seconds, so 5 attempts plus the waits (112 seconds at most) finish before the server's 120-second limit on a request.
- **Steps:**
  1. Create a payment link (Shopify answers straight away).
- **Expected result:**
  - The request to Shopify is sent with a **20-second** time limit.

**16. Generate Payment Link: Shopify times out on all 5 attempts** · Negative

`test_generate_link_all_timeouts_alerts_and_saves_nothing`

- **What it checks:** When Shopify cannot be reached at all, the user sees the error, the urgent Slack alert goes out, and no half-made link is saved.
- **Starting point:** A sent quotation.
- **Steps:**
  1. With the payment link on, generate the payment link while Shopify times out on every attempt.
- **Expected result:**
  - The timeout error is shown to the user.
  - The urgent Slack alert is sent **once**.
  - No payment link is saved on the quotation.

**17. Revision update: Shopify times out on all 5 attempts** · Negative

`test_revision_update_all_timeouts_clears_link_and_alerts_owner`

- **What it checks:** After a revision, if Shopify cannot be reached, the old payment link (with the old amount) is removed and the quotation owner is told, so the customer cannot pay the wrong amount.
- **Starting point:** A sent quotation with Shopify draft order 555 and an existing payment link.
- **Steps:**
  1. Run the payment link update while Shopify times out on every attempt.
- **Expected result:**
  - The update finishes without crashing.
  - The old payment link is **cleared**.
  - The quotation owner gets an alert.
- **Note:** Before Oct 26 a timeout here crashed the background job: the old link stayed on the quotation and nobody was told. Fixed on Oct 26.

**18. Each payment link attempt is logged separately** · Positive

`test_each_attempt_logs_its_own_integration_request`

- **What it checks:** Every try to reach Shopify leaves its own record, so you can see what happened on each of the 5 attempts.
- **Starting point:** A sent quotation.
- **Steps:**
  1. Send the payment link request while Shopify rejects it (401) every time.
- **Expected result:**
  - **5** Integration Requests linked to the quotation, named **Shopify Payment Link - Attempt 1/5** to **5/5**, all **Failed**.

**19. On a developer-mode site the failure alert goes to the 1Click webhook** · Positive

`test_failed_link_alert_goes_to_1click_webhook_in_developer_mode`

- **What it checks:** A local or test site never posts payment link failures to the real urgent channel, but the alert can still be seen.
- **Starting point:** A sent quotation, developer mode on.
- **Steps:**
  1. Send the payment link failure alert.
- **Expected result:**
  - The "Payment Link Generation Failed" message is sent to the **1Click Slack Webhook URL** only.
  - The normal Slack sender (urgent channel) is not used.

#### ShopifyPollingTest

A scheduled job checks each sent quotation's Shopify draft order to see if the customer has paid. Shopify's replies are simulated. Each test starts from a sent quotation with Shopify draft order 555.

**20. Paid Shopify order records the payment and wins the quotation** · Positive

`test_paid_draft_records_payment_and_wins`

- **What it checks:** When the customer pays in full by card, the payment is recorded and the quotation is Won.
- **Steps:**
  1. The job runs; Shopify says the draft is completed and order #1001 is **paid** in full.
- **Expected result:**
  - State is **Won**.
  - Shopify Order Name is **#1001**.
  - **One** payment row for the full total.

**21. Partly paid Shopify order makes the quotation Partially Paid** · Positive

`test_partially_paid_order`

- **What it checks:** A partial card payment is recorded as such.
- **Steps:**
  1. The job runs; Shopify says the order is **partially paid** (half the total).
- **Expected result:**
  - State is **Partially Paid**.

**22. Unpaid Shopify draft changes nothing** · Negative

`test_open_draft_changes_nothing`

- **What it checks:** If the customer has not paid yet, nothing changes.
- **Steps:**
  1. The job runs; Shopify says the draft is still **open**.
- **Expected result:**
  - State stays **Sent to Customer**; no payment rows.

**23. Shopify timeout on one quotation does not stop the others** · Negative

`test_timeout_on_one_quotation_does_not_stop_others`

- **What it checks:** If Shopify times out for one quotation, the job logs it and still checks the rest.
- **Steps:**
  1. Two sent quotations: Shopify times out for draft 555, and reports draft 556 as completed and paid in full. Run the job.
- **Expected result:**
  - The timed-out quotation stays **Sent to Customer**.
  - The other quotation is **Won**.

**24. Pending bank transfer order is not recorded as paid** · Negative

`test_pending_bank_transfer_order_not_recorded`

- **What it checks:** A Shopify order waiting for a bank transfer is not counted as money received.
- **Steps:**
  1. The job runs; Shopify says the order is **pending** (bank transfer not yet received).
- **Expected result:**
  - No payment rows.

#### FollowupSchedulerTest

A daily job creates follow-up tasks for sales on business days 3, 10, 25 and 30 after the quotation email was sent. Each test starts from a sent quotation with Email Sent At set and no Valid Till date; the number of business days is simulated and tasks are not really created.

**25. Follow-up tasks are created on days 3, 10 and 25** · Positive

`test_followup_days`

- **What it checks:** Follow-ups happen on the right days and not before.
- **Steps:**
  1. Run the job at business day 2, 3, 10 and 25.
- **Expected result:**
  - Day 2: no task.
  - Days 3, 10 and 25: one follow-up task each.

**26. A follow-up already sent is not repeated** · Negative

`test_day_already_sent_is_skipped`

- **What it checks:** The same follow-up day never creates two tasks.
- **Steps:**
  1. Record day 3 as already sent, then run the job at day 4.
- **Expected result:**
  - No task.

**27. Day 30: final follow-up, then Lost** · Positive

`test_day_30_final_task_then_lost`

- **What it checks:** After 30 business days without a result, a final task is created and the quotation is closed as Lost.
- **Steps:**
  1. Run the job at day 30.
- **Expected result:**
  - A final day-30 task is created; state is **Lost**.

**28. Quotation past its Valid Till date expires** · Negative

`test_valid_till_passed_expires`

- **What it checks:** A quotation past its Valid Till date gets no more follow-ups and is marked Expired.
- **Steps:**
  1. Set Valid Till to yesterday and run the job at day 3.
- **Expected result:**
  - No task; state is **Expired**.

**29. A quotation that was never emailed gets no follow-up** · Negative

`test_never_emailed_is_skipped`

- **What it checks:** Follow-ups only start after the quotation email was sent.
- **Steps:**
  1. Clear Email Sent At and run the job at day 30.
- **Expected result:**
  - No task.

#### SlaAndCogsWiringTest

Connections to the LH Project SLA system and the COGS benchmark check.

**30. SLA close check runs at every step** · Positive

`test_sla_close_check_runs_on_update_and_submit`

- **What it checks:** Open SLA tasks for a quotation are checked for closing whenever the quotation is saved, moved or sent.
- **Starting point:** A standard quotation with contact and address, and an active SLA Task Link.
- **Steps:**
  1. Save the quotation, click **Inquiry Approved**, then **Send Quotation to Client**.
- **Expected result:**
  - The SLA close check runs at least **3** times (save, workflow step, send).

**31. COGS benchmark alert only when COGS changes** · Negative

`test_cogs_deviation_enqueues_alert_only_when_cogs_changes`

- **What it checks:** The alert about COGS far from the usual price is sent only when someone changes COGS, not on every save.
- **Starting point:** A standard quotation; the benchmark is set to report a large difference (999 vs a usual 40).
- **Steps:**
  1. Save without changing COGS.
  2. Change the item COGS to 999 and save.
- **Expected result:**
  - Step 1: no alert.
  - Step 2: the COGS deviation alert email is queued.

#### PermissionQueryTest

The filter that limits which quotations a user sees in the list.

**32. Admins see every quotation** · Positive

`test_admins_see_everything`

- **What it checks:** System Manager and Super Admin are never filtered.
- **Steps:**
  1. Get the list filter for System Manager, then for Super Admin + Senior Engineer.
- **Expected result:**
  - No filter in both cases.

**33. Senior Engineers see only their review quotations** · Positive

`test_senior_engineer_scoped_to_assigned_review_states`

- **What it checks:** A Senior Engineer sees quotations assigned to them in Senior Review Pending or Pending Engineer Review.
- **Steps:**
  1. Get the list filter for a user with Senior Engineer + Sales User, and run it against the database.
- **Expected result:**
  - The filter names the user and the states **Senior Review Pending** and **Pending Engineer Review**; it runs without error.

**34. Other roles are not filtered** · Positive

`test_other_roles_unrestricted`

- **What it checks:** Sales users see the normal list.
- **Steps:**
  1. Get the list filter for Sales User.
- **Expected result:**
  - No filter.

#### LinkedOrdersAndDashboardTest

Links shown on the quotation form.

**35. Linked Lyfe Orders are listed** · Positive

`test_linked_lyfe_orders`

- **What it checks:** The quotation shows the Lyfe Orders created from it.
- **Starting point:** A standard quotation with a linked Lyfe Order.
- **Steps:**
  1. Fetch the linked Lyfe Orders.
- **Expected result:**
  - Returns that Lyfe Order.

**36. Timesheets appear in the quotation's Connections** · Positive

`test_dashboard_shows_timesheets`

- **What it checks:** The Connections panel lists Timesheets, linked through the Quotation field on Timesheet.
- **Steps:**
  1. Read the Quotation dashboard (Connections) settings.
- **Expected result:**
  - **Timesheet** is listed, linked through the custom Quotation field.

### test\_validate (24)

Rules that run every time a quotation is saved.

#### OrderTypeDetectionTest

The quotation's Order Type is set automatically to Standard or Custom.

**1. Normal item → Standard** · Positive

`test_plain_item_is_standard`

- **What it checks:** A quotation for a normal catalogue item is Standard.
- **Steps:**
  1. Create a standard quotation.
- **Expected result:**
  - Order Type is **Standard**.

**2. Item name starting with a custom prefix → Custom** · Positive

`test_custom_prefix_in_item_name_is_custom`

- **What it checks:** Items whose name starts with a custom prefix (here "cstm" or "custom") make the quotation Custom.
- **Steps:**
  1. Create a quotation for an item named **"CSTM Bracket"**.
- **Expected result:**
  - Order Type is **Custom**.

**3. Is Custom Order ticked on the item → Custom** · Positive

`test_is_custom_order_checkbox_is_custom`

- **What it checks:** Ticking Is Custom Order on an item line makes the quotation Custom.
- **Steps:**
  1. Create a quotation with **Is Custom Order** ticked on the item.
- **Expected result:**
  - Order Type is **Custom**.

#### ItemDimensionTest

Box size and weight on each item line, used for shipping.

**4. Box size and weight come from the Item master** · Positive

`test_dims_from_item_master`

- **What it checks:** If the item has a box size, the line is filled from it.
- **Steps:**
  1. Item master box size **30 × 20 × 10 cm**, weight **2 kg**; create a quotation.
- **Expected result:**
  - Line box size **30 / 20 / 10**, weight **2**, not flagged.

**5. Box size typed on the line is kept** · Positive

`test_manual_row_dims_are_kept`

- **What it checks:** A size entered by hand on the line is not overwritten by the Item master.
- **Steps:**
  1. Same item; type **5 × 5 × 5** on the line and save.
- **Expected result:**
  - The line keeps **5**.

**6. Missing box size on a BOM part is flagged** · Negative

`test_bom_with_missing_child_dims_is_flagged`

- **What it checks:** If the box size has to come from a BOM and a part has no size, the line is flagged instead of getting a wrong size.
- **Steps:**
  1. Item with no box size, using a BOM whose part **CHILD-1** has no size; create a quotation.
- **Expected result:**
  - The line is **flagged**, and the reason mentions **CHILD-1**.
  - No partial box size is filled in.

#### CombinedBoxTest

Packing all items into boxes for shipping. Each test uses an item of 30 × 20 × 10 cm, 2 kg, qty 2. Packing can come from 3D Bin Packing, Paccurate, or the built-in packer.

**7. Built-in packer picks a box from the Box Catalogue** · Positive

`test_combined_box_and_parcels_from_catalogue`

- **What it checks:** With both outside services off, the built-in packer still produces a box and parcel plan.
- **Steps:**
  1. Turn 3D Bin Packing and Paccurate off; create the quotation.
- **Expected result:**
  - A combined box size is set on the quotation.
  - A Box Catalogue box is selected and the suggestions include **"Best fit"**.
  - The line's Single Box Size is filled.
  - A parcel plan is built.

**8. 3D Bin Packing is used first when switched on** · Positive

`test_3dbinpacking_takes_priority_when_enabled`

- **What it checks:** When 3D Bin Packing is on, its result is used.
- **Steps:**
  1. Turn 3D Bin Packing on (it returns one 40 cm box); create the quotation.
- **Expected result:**
  - 3D Bin Packing is called; the parcel plan is **one 40 cm box**.

**9. Paccurate is used when 3D Bin Packing is off** · Positive

`test_paccurate_used_when_3dbin_off`

- **What it checks:** When only Paccurate is on, its result and its packing picture are used.
- **Steps:**
  1. Turn only Paccurate on (it returns two 50 cm boxes and a picture); create the quotation.
- **Expected result:**
  - The parcel plan is **two 50 cm boxes**; the Paccurate picture is saved.

#### ApprovalAndCogsFlagTest

The COGS Pending and Approval Required flags.

**10. Missing COGS ticks COGS Pending** · Negative

`test_missing_cogs_sets_cogs_pending`

- **What it checks:** A line without COGS is flagged so the factory fills it in.
- **Steps:**
  1. Create a quotation with item COGS **0**.
- **Expected result:**
  - **COGS Pending** is ticked.

**11. Total of 10,000+ ticks Approval Required** · Positive

`test_grand_total_over_10k_requires_approval`

- **What it checks:** Large quotations are flagged for manager approval.
- **Steps:**
  1. Create a quotation at rate **12,000**.
- **Expected result:**
  - **Approval Required** is ticked.

**12. Approval is not asked again once approved** · Positive

`test_approval_not_reraised_once_approved`

- **What it checks:** Saving an approved quotation does not tick Approval Required again.
- **Steps:**
  1. Jump a 12,000 quotation to **Approved – Pending Send** and save.
- **Expected result:**
  - **Approval Required** stays unticked.

#### QtyAndRateTest

Limits on item quantity and rate.

**13. Rate 0 is allowed** · Positive

`test_zero_rate_is_allowed`

- **What it checks:** Fee-only quotations need items at rate 0.
- **Steps:**
  1. Create a quotation with an item at rate **0** (no COGS).
- **Expected result:**
  - Saved; the line rate is **0**.

**14. Negative rate is rejected** · Negative

`test_negative_rate_rejected`

- **What it checks:** No item line may have a negative price. ERPNext itself only stops a negative grand total, so a negative line next to a positive one needs this rule.
- **Steps:**
  1. Create a quotation with one item at **100** and one at **-50** (total stays positive).
- **Expected result:**
  - Blocked with "cannot be negative".

**15. Quantity 0 is rejected** · Negative

`test_zero_qty_rejected`

- **What it checks:** Every item line needs a quantity (rule from ERPNext).
- **Steps:**
  1. Create a quotation with an item at qty **0**.
- **Expected result:**
  - Blocked with "cannot be zero".

#### DiscountViolationE2ETest

A discount above the allowed limit ticks Approval Required. Limits: card payment **5%**, bank transfer **8%**, unless the Item or Item Group sets its own limit (the Item limit wins over the Item Group).

**16. Card payment: discount over 5% needs approval** · Negative

`test_card_payment_over_5pct_needs_approval`

- **What it checks:** A 6% discount on a card-paid quotation needs approval.
- **Steps:**
  1. Item and Item Group with no limit; item at 100 with **6%** discount; card payment.
- **Expected result:**
  - **Approval Required** is ticked.

**17. Bank transfer: up to 8% discount is allowed** · Positive

`test_bank_transfer_allows_up_to_8pct`

- **What it checks:** The same 6% discount is fine when paid by bank transfer.
- **Steps:**
  1. Same quotation with **Payment via Bank Transfer** ticked.
- **Expected result:**
  - **Approval Required** stays unticked.

**18. Item Group limit is used when the Item has none** · Negative

`test_item_group_limit_applies_when_item_has_none`

- **What it checks:** If the Item has no limit, the Item Group's limit applies.
- **Steps:**
  1. Item has no limit, Item Group allows **15%**; check a **12%** discount, then **16%**.
- **Expected result:**
  - 12% is allowed; 16% is over the limit.

**19. Item limit wins over the Item Group limit** · Positive

`test_item_limit_beats_item_group`

- **What it checks:** The Item's own limit is used even when the Item Group's is lower.
- **Steps:**
  1. Item allows **20%**, Item Group **5%**; check a **15%** discount.
- **Expected result:**
  - Allowed (the Item limit wins).

#### AutoClearAndCleanupTest

Fields and rows the system fills or clears automatically.

**20. Attaching the drawing clears Drawing Required and Photo Required** · Positive

`test_drawing_clears_drawing_and_photo_required`

- **What it checks:** Once the drawing is attached, the request flags are cleared.
- **Steps:**
  1. Quotation with Drawing Required and an item with Photo Required; attach the drawing with change details and save.
- **Expected result:**
  - Both flags are unticked.

**21. IGST tax row is removed** · Positive

`test_igst_row_is_stripped`

- **What it checks:** IGST (Indian tax) never applies to these quotations and is removed if added.
- **Steps:**
  1. Add an **IGST Payable** tax row and save.
- **Expected result:**
  - The IGST row is removed.

**22. Default payment terms and T&C are filled** · Positive

`test_defaults_terms_template`

- **What it checks:** New quotations get the standard payment terms and terms and conditions.
- **Steps:**
  1. Create a standard quotation.
- **Expected result:**
  - Payment terms **"100 % Advance"**; T&C **"Terms and Conditions template"**.

**23. Shipping and Custom Fee rows are added only once** · Positive

`test_after_insert_rows_added_once`

- **What it checks:** Saving again does not add duplicate rows.
- **Steps:**
  1. Create a standard quotation and save it again.
- **Expected result:**
  - One Shipping Charges row and one Custom Fee row.

**24. A duplicated quotation does not keep the old payment link** · Positive

`test_after_insert_clears_copied_shopify_and_email_fields`

- **What it checks:** When a quotation is copied, the Shopify draft, payment link and Email Sent flag are cleared so the copy gets its own.
- **Steps:**
  1. Create a quotation carrying a Shopify draft ID, payment URL and Email Sent (as a duplicate would).
- **Expected result:**
  - All three are cleared.

### test\_drawings (17)

Order Drawing upload rules, removing the BOM page from drawings, the Drawing Change Details popup, and Senior Drawing Review.

#### DrawingUploadGateTest

Who can upload an Order Drawing.

**1. Engineer must log a Timesheet before uploading** · Negative

`test_engineer_without_timesheet_is_blocked`

- **What it checks:** An Engineer cannot upload a drawing without logging time on the quotation.
- **Starting point:** A standard quotation.
- **Steps:**
  1. As an Engineer with no Timesheet, attach a PDF to **Order Drawing**.
- **Expected result:**
  - Blocked with "log a Timesheet".

**2. Engineer with a Timesheet can upload** · Positive

`test_engineer_with_timesheet_is_allowed`

- **What it checks:** Once time is logged, the upload is allowed.
- **Starting point:** A standard quotation.
- **Steps:**
  1. Same, with a Timesheet logged.
- **Expected result:**
  - The file is attached.

**3. Customer Service upload creates an engineering task** · Positive

`test_customer_service_upload_queues_engineering_task`

- **What it checks:** When Customer Service uploads a drawing, an engineer is given a task to review it.
- **Starting point:** A standard quotation.
- **Steps:**
  1. As Customer Service, attach a PDF to **Order Drawing**.
- **Expected result:**
  - An engineering Task is created for the quotation.

**4. Other attachment fields are not restricted** · Positive

`test_other_fields_are_not_gated`

- **What it checks:** The Timesheet rule applies only to Order Drawing.
- **Starting point:** A standard quotation.
- **Steps:**
  1. As an Engineer with no Timesheet, attach a PDF to **Order Drawing with BOM**.
- **Expected result:**
  - The file is attached.

#### PdfBomStripperTest

If a drawing PDF contains a BOM page, the system removes it from the Order Drawing (which the customer sees) and keeps the full PDF in **Order Drawing with BOM**. "Upload" below means: attach the PDF as Order Drawing with change details "first upload", and save.

**5. BOM page is removed from the drawing** · Positive

`test_bom_page_stripped`

- **What it checks:** The customer-facing drawing has no BOM page.
- **Steps:**
  1. Upload a 2-page PDF: page 1 "Assembly drawing", page 2 "BOM / Breakdown of Components".
- **Expected result:**
  - The Order Drawing file has **only page 1**.
  - **Order Drawing with BOM** is filled.

**6. Full PDF is kept in Order Drawing with BOM** · Expected failure

`test_original_with_bom_kept`

- **What it checks:** The Order Drawing with BOM file keeps both pages.
- **Steps:**
  1. Upload the same 2-page PDF and open the Order Drawing with BOM file.
- **Expected result:**
  - Should have **both pages**.
- **Note:** **Expected failure (known issue, local file storage only).** On the local site the copy points at the same file as the upload, so it is stripped too and has 1 page. On production, files are stored on S3 and the copy gets its own file, so the issue does not appear there.

**7. PDF without a BOM page is not changed** · Negative

`test_pdf_without_bom_left_alone`

- **What it checks:** Drawings with no BOM page are left as they are.
- **Steps:**
  1. Upload a 1-page PDF with no BOM page.
- **Expected result:**
  - The file is unchanged; Order Drawing with BOM stays empty.

**8. Unchanged drawing is not processed again** · Negative

`test_unchanged_drawing_not_reprocessed`

- **What it checks:** Saving without changing the drawing does not run the BOM removal again.
- **Steps:**
  1. After the BOM upload, save again without changing the drawing.
- **Expected result:**
  - The BOM removal does not run.

#### DrawingChangeDetailsTest

Every new or changed Order Drawing needs a note in **Drawing Change Details**. In the form, a popup asks for this note right after upload.

**9. First drawing needs change details, then the factory is notified** · Negative

`test_first_upload_needs_details_then_notifies`

- **What it checks:** A drawing cannot be saved without a note; once saved, the factory gets an email.
- **Starting point:** A standard quotation.
- **Steps:**
  1. Attach an Order Drawing with no change details and save.
  2. Add change details "rev A" and save.
- **Expected result:**
  - Step 1 is blocked with "Drawing Change Details is mandatory".
  - Step 2: **one** notification email goes to support@chandakbrothers.net.

**10. Removing and re-attaching a drawing counts as a replacement** · Negative

`test_clear_then_reattach_counts_as_replacement`

- **What it checks:** Clearing the drawing and attaching a new one still needs a change note.
- **Starting point:** A standard quotation.
- **Steps:**
  1. Attach a drawing with change details and save.
  2. Clear the drawing and save.
  3. Attach a new drawing with no change details and save.
- **Expected result:**
  - Step 2: **Replacement Pending** is ticked.
  - Step 3 is blocked with "Drawing Change Details is mandatory".

**11. Upload popup: save first, then add the note** · Negative

`test_popup_flow_mark_pending_then_submit_details`

- **What it checks:** In the form, the upload saves straight away and the popup collects the note afterwards; a blank note is refused.
- **Starting point:** A standard quotation.
- **Steps:**
  1. The user attaches a drawing; the system marks the change as pending, and the form saves.
  2. The user submits a blank note in the popup.
  3. The user submits "moved holes 5mm".
- **Expected result:**
  - Step 1: the save goes through without a note; **Change Pending** is ticked.
  - Step 2: refused with "cannot be empty".
  - Step 3: the note is saved, Change Pending is unticked, and a Comment with the note is added.

**12. Cancel in the popup removes the new drawing** · Positive

`test_cancel_upload_reverts_drawing`

- **What it checks:** If the user cancels the popup, the upload is undone.
- **Starting point:** A standard quotation.
- **Steps:**
  1. Mark the change as pending, set a drawing, then click **Cancel** in the popup.
- **Expected result:**
  - The drawing is removed and Change Pending is unticked.

**13. Popup actions refuse a quotation that does not exist** · Negative

`test_endpoints_reject_unknown_quotation`

- **What it checks:** The popup's server actions cannot be used with a wrong quotation name.
- **Steps:**
  1. Call **Mark Pending** and **Cancel Upload** for a quotation name that does not exist.
- **Expected result:**
  - Both are refused with "does not exist".

#### GraceSweepTest

A scheduled job tidies up uploads whose popup was never answered (for example the browser was closed).

**14. Unanswered uploads are undone after the waiting time** · Negative

`test_sweep`

- **What it checks:** An upload left without a note is undone after the waiting time; recent ones and answered ones are left alone.
- **Steps:**
  1. Create three pending uploads: 20 minutes old with no note, 5 minutes old, and 20 minutes old with the note already filled.
  2. Run the job.
- **Expected result:**
  - 20 minutes, no note: the drawing is removed and a Comment says it was **automatically reverted**.
  - 5 minutes: untouched (still within the waiting time).
  - 20 minutes with a note: the drawing is kept and the pending flag is cleared.

#### SeniorReviewTest

Senior Drawing Review for Custom orders, and the lock on approved items.

**15. Request and complete a Senior Review** · Negative

`test_request_and_complete_senior_review`

- **What it checks:** An engineer can ask a senior engineer to review a Custom order; a reason is required, it cannot be requested twice, and completing it returns the quotation to the engineer.
- **Starting point:** A Custom-order quotation (item has Is Custom Order ticked) in Pending Engineer Review.
- **Steps:**
  1. Click **Request Senior Review** without a Reason Code.
  2. Request it with Reason Code "New geometry", a note and a Product Finish.
  3. Request it again.
  4. The senior engineer submits the review as "Approved as-is".
- **Expected result:**
  - Step 1 is blocked with a message about the **Reason Code**.
  - Step 2: state is **Senior Review Pending** and the Senior Review Log is linked.
  - Step 3 is blocked ("can only be requested").
  - Step 4: state is back to **Pending Engineer Review**.
- **Note:** Skipped automatically if the site has no Product Finish records.

**16. Changing items after approval sends the quotation back for review** · Positive

`test_spec_change_after_approval_routes_back`

- **What it checks:** Approval covers the exact items approved; changing them cancels the approval.
- **Starting point:** A standard quotation jumped to Pending Approval.
- **Steps:**
  1. Click **Approve**.
  2. Change the item qty to 3 and save.
- **Expected result:**
  - Step 1: a snapshot of the approved items (spec hash) is saved.
  - Step 2: state goes back to **Pending Engineer Review** and the snapshot is cleared.

**17. Changing non-item fields keeps the approval** · Positive

`test_non_spec_change_keeps_approval`

- **What it checks:** Fields that do not affect the items (like Priority) can change without losing approval.
- **Starting point:** An approved quotation (Approved – Pending Send).
- **Steps:**
  1. Change Priority to **VIP** and save.
- **Expected result:**
  - State stays **Approved – Pending Send**.

## How to run

The suite lives in `apps/lh/lh/tests/quotation_e2e/` and runs one module at a time on a site with tests allowed:

```bash
bench --site lyfelocal.com run-tests --app lh --module lh.tests.quotation_e2e.test_lifecycle
bench --site lyfelocal.com run-tests --app lh --module lh.tests.quotation_e2e.test_payments
bench --site lyfelocal.com run-tests --app lh --module lh.tests.quotation_e2e.test_integrations
bench --site lyfelocal.com run-tests --app lh --module lh.tests.quotation_e2e.test_validate
bench --site lyfelocal.com run-tests --app lh --module lh.tests.quotation_e2e.test_drawings
```

The site needs `allow_tests` enabled, the active Quotation-3 workflow, and the LH custom fields from the app's patches. Shared setup (base test class, factories, stubs) is in `_helpers.py`.

Before production:

- [ ] Commit the test suite and code, deploy, then `bench migrate` and `bench restart`
