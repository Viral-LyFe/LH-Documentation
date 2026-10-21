# Quotation E2E Test Report

Oct 21, 2026 · @Viral kansodiya

132 automated end-to-end tests now cover every LH Quotation customisation: 131 pass and 1 is marked as an expected failure. Nothing is committed or deployed to production yet.

## Results

All five modules pass on lyfelocal.com (Frappe 15.121, ERPNext 15.121). A full run takes about 75 seconds and leaves the database and disk exactly as it found them.

| Module | Area | Tests | Pass | Expected failure |
| --- | --- | --- | --- | --- |
| test\_lifecycle | Workflow walks, transition guards, approval, revisions | 43 | 43 | 0 |
| test\_payments | Payment schedule, fee, stage blocks, milestones, reminders | 27 | 27 | 0 |
| test\_integrations | Email, approval link, Shopify, follow-up, SLA, COGS benchmark, permissions | 24 | 24 | 0 |
| test\_validate | Order type, box sizes, discounts, approval / COGS flags | 21 | 21 | 0 |
| test\_drawings | Upload gate, PDF BOM stripper, change details, Senior Review | 17 | 16 | 1 |
| **Total** |  | **132** | **131** | **1** |

Every test runs inside one database transaction that is rolled back at the end. Outbound effects (background jobs, email, Slack, Shopify, S3, PDF rendering) are stubbed, so a run sends nothing. Quotation, Lyfe Order, File and Comment counts were identical before and after a full run.

## Business rules confirmed

The tests check these business rules.

| Rule | Behaviour |
| --- | --- |
| Payment stage unpaid: Advance, Photo Approval, Before Dispatch | Hard block on the Lyfe Order step, plus a reminder |
| Payment stage unpaid: Before Shipped, On Delivery | Reminder only, never a block |
| Quotation Lost or Cancelled (any route, including the follow-up scheduler) | No collection reminders; payment blocks on linked Lyfe Orders are released |
| Partial payment recorded | Advance stage becomes Due and CS gets the Due reminder |
| Send for Approval, from any state | COGS not required (0-rate quotations carrying only Custom Fee and Shipping) |
| Production Details Review → Feasibility Approve, to any state | COGS required |
| Production Details Review → Not Feasible | COGS not required |
| Entering Pending Approval, from any route | Slack approval card posted once, after the save commits |
| Pending Engineer Review → Completed with Drawing Required or Photo Required | Order Drawing required, except Urgent quotations |

## Coverage

Every item in the Asana scope is covered by at least one test class.

| Asana scope item | Test classes |
| --- | --- |
| Custom-quotation detection | OrderTypeDetectionTest |
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
| Shopify payment link + payment status polling | ShopifyLinkTest, ShopifyPollingTest |
| Follow-up scheduler | FollowupSchedulerTest |
| Linked Lyfe Orders, dashboard data | LinkedOrdersAndDashboardTest |
| Full lifecycle | HappyPathTest, RejectionBranchTest |

## Test case catalogue

All 132 test cases, grouped by module and test class. Each test lists the steps it performs and what it expects. All pass unless marked Expected failure.

### Common setup

The steps use these shorthands:

- **Standard quotation**: one item at rate 100 with COGS 40, payment terms "100 % Advance" (one Advance row). Saving it moves it from Draft to Inquiry.
- **Staged quotation**: one item at rate 1,000 with COGS 400 and a 40 / 30 / 30 schedule (Advance / Photo Approval / Before Dispatch unless the test says otherwise).
- **Ready to send**: standard quotation with a contact email and a customer address, moved to Inquiry Approved.
- **Sent quotation**: ready to send, then Send Quotation to Client (state Sent to Customer, submitted).
- **Linked Lyfe Order**: a minimal Lyfe Order whose Source Quotation is the test quotation.
- **Record payment**: add a Quotation Payment Entry row for the amount and set Total Received Amount.
- **Jump to state**: set the workflow state directly to skip setup steps that are tested elsewhere.

Slack, email, Shopify, S3 and background jobs are stubbed. When a step says "queued" or "sent", the test checks that the call was made.

### test\_lifecycle (43)

#### HappyPathTest

**1. test\_inquiry\_to\_won\_to\_cancelled**

1. Create a standard quotation
2. Add contact and address, then Inquiry Approved
3. Send Quotation to Client
4. Bank Transfer Selected, record full payment, Confirm Payment
5. Cancel

- **Expected:** State is Inquiry after insert, with Shipping Charges and Custom Fee rows; then Sent to Customer (submitted), Won, Cancelled (docstatus 2)

**2. test\_partial\_payment\_then\_remaining**

1. Sent quotation → Bank Transfer Selected
2. Record half the total, Confirm Payment
3. Record the full total, Confirm Remaining Payment

- **Expected:** Partially Paid, then Won

**3. test\_sent\_to\_customer\_needs\_email\_and\_address**

1. Standard quotation → Inquiry Approved, no contact
2. Send Quotation to Client
3. Add contact email only, send again

- **Expected:** Blocked with "Customer email", then with "Customer address"

#### PostSubmitHooksTest

**4. test\_cancel\_releases\_payment\_gates**

1. Sent quotation
2. Linked Lyfe Order with a Before Dispatch payment block
3. Cancel the quotation

- **Expected:** Lyfe Order payment block is cleared

**5. test\_lost\_releases\_payment\_gates**

1. Same as above, but mark the quotation Lost

- **Expected:** Lyfe Order payment block is cleared

**6. test\_won\_with\_no\_payment\_is\_blocked**

1. Sent quotation, jump to Payment Received
2. Mark as Won with no payment recorded

- **Expected:** Blocked with "no payment has been recorded"

**7. test\_won\_marks\_advance\_milestone\_paid**

1. Sent quotation → Bank Transfer Selected
2. Record full payment, Confirm Payment

- **Expected:** Advance payment schedule row shows Paid

#### ApprovalPathTest

**8. test\_over\_10k\_needs\_approval\_then\_sends**

1. Ready to send at rate 12,000 (COGS 5,000)
2. Send for Approval
3. Approve
4. Send Quotation to Client

- **Expected:** Approval Required set; Pending Approval; Approved – Pending Send with the flag cleared and spec hash stored; Sent to Customer

**9. test\_send\_back\_to\_sales\_user**

1. 12,000 quotation → Send for Approval
2. Send Back to Sales User

- **Expected:** State is Sales User Review

**10. test\_slack\_card\_from\_feasibility\_approved**

1. 12,000 quotation, jump to Feasibility Approved
2. Enable Quotation Approval on, Send for Approval

- **Expected:** Slack approval card queued for this quotation

**11. test\_slack\_card\_off\_when\_setting\_disabled**

1. Same as above with Enable Quotation Approval off

- **Expected:** No card queued

**12. test\_fee\_only\_quote\_sent\_for\_approval\_from\_inquiry\_approved**

1. Quotation with a 0-rate item, no COGS, Custom Fee 12,000
2. Inquiry Approved
3. Send for Approval

- **Expected:** COGS Pending and Approval Required are set, yet it reaches Pending Approval

**13. test\_fee\_only\_quote\_sent\_for\_approval\_from\_sales\_user\_review**

1. Same fee-only quotation, jump to Sales User Review, Send for Approval

- **Expected:** Pending Approval

**14. test\_no\_cogs\_needed\_from\_feasibility\_approved**

1. 12,000 quotation with COGS 0, jump to Feasibility Approved
2. Send for Approval

- **Expected:** Pending Approval (no COGS gate)

**15. test\_slack\_card\_from\_every\_route**

1. Enter Pending Approval three ways: Send for Approval from Inquiry Approved, Send for Approval from Sales User Review (fee-only), Feasibility Approve from Production Details Review (free shipping)

- **Expected:** Each route queues one card, after the save commits

**16. test\_no\_second\_card\_while\_already\_pending**

1. 12,000 quotation → Send for Approval
2. Save again in Pending Approval

- **Expected:** No second card queued

**17. test\_erp\_approve\_closes\_slack\_card**

1. 12,000 quotation → Send for Approval
2. Approve in ERPNext

- **Expected:** Job to close the Slack card queued

**18. test\_slack\_approve\_does\_not\_double\_close\_card**

1. Same, but Approve with the "from Slack" flag set

- **Expected:** Close job not queued (Slack closes its own card)

**19. test\_slack\_approve\_rejected\_by\_erp\_replies\_in\_thread**

1. 12,000 quotation in Pending Approval with an open Slack thread
2. Run the Slack Approve action, with ERPNext raising "COGS missing"

- **Expected:** Reply posted in the thread with "Reason: COGS missing"; card buttons and thread left open so the approver can retry

#### EngineeringBranchTest

**20. test\_drawing\_required\_routes\_to\_engineer\_then\_sales**

1. Quotation with Drawing Required → Inquiry Approved → Request for Technical Details
2. Attach the Order Drawing with change details, save
3. Completed

- **Expected:** Pending Engineer Review; Drawing Required auto-clears; Sales User Review

**21. test\_completed\_blocked\_without\_drawing**

1. Quotation in Pending Engineer Review with no drawing → Completed (server path, as list-view bulk action or API)

- **Expected:** Blocked with "Drawing"

**22. test\_completed\_to\_production\_review\_blocked\_without\_drawing**

1. Same, with Feasibility Test Required set so Completed leads to Production Details Review

- **Expected:** Blocked with "Drawing"

**23. test\_completed\_blocked\_by\_item\_photo\_required**

1. Pending Engineer Review, Drawing Required off but Photo Required on an item → Completed

- **Expected:** Blocked with "Drawing"

**24. test\_urgent\_completes\_without\_drawing**

1. Pending Engineer Review, Priority Urgent, no drawing → Completed

- **Expected:** Sales User Review

**25. test\_need\_more\_information\_not\_blocked**

1. Pending Engineer Review, no drawing → Need More Information

- **Expected:** Draft (not blocked)

**26. test\_feasibility\_flag\_auto\_advances\_to\_production\_review**

1. Standard quotation in Sales User Review, tick Feasibility Test Required, save

- **Expected:** Moves to Production Details Review

**27. test\_feasibility\_flag\_auto\_clears\_when\_cogs\_and\_shipping\_set**

1. In Production Details Review, tick Feasibility Test Required and Free Shipping (COGS already set), save

- **Expected:** Feasibility Test Required clears

**28. test\_production\_review\_needs\_shipping\_unless\_free**

1. Production Details Review, no shipping → Feasibility Approve
2. Tick Free Shipping, Feasibility Approve again

- **Expected:** Blocked with "Shipping Charges", then Feasibility Approved

**29. test\_feasibility\_approve\_needs\_cogs\_whatever\_the\_target**

1. For rates 100 and 12,000: quotation with COGS 0 and free shipping in Production Details Review → Feasibility Approve

- **Expected:** Both blocked with "Cost of Goods"

**30. test\_feasibility\_approve\_over\_10k\_goes\_to\_pending\_approval**

1. 12,000 quotation with COGS and free shipping in Production Details Review → Feasibility Approve

- **Expected:** Pending Approval

**31. test\_photo\_required\_in\_production\_review\_routes\_back\_to\_engineer**

1. In Production Details Review, tick Photo Required on an item, save

- **Expected:** Moves back to Pending Engineer Review

**32. test\_request\_senior\_review\_only\_for\_custom\_orders**

1. Standard-type quotation in Pending Engineer Review → Request Senior Review

- **Expected:** Rejected (action not available for Standard orders)

#### RejectionBranchTest

**33. test\_inquiry\_rejected\_needs\_reason**

1. Standard quotation → Inquiry Rejected with no reason
2. Fill Inquiry Rejected Reason, try again

- **Expected:** Blocked with "Rejected Reason", then Inquiry Rejected

**34. test\_not\_feasible\_needs\_reason\_then\_cs\_review**

1. Production Details Review → Not Feasible with no reason
2. Fill Not Feasible Reason, try again
3. CS Review

- **Expected:** Blocked with "Not Feasible Reason"; Not Feasible; Sales User Review

**35. test\_not\_feasible\_reason\_locked\_to\_factory**

1. Quotation with Not Feasible Reason "original"
2. As a Sales User (no Factory role), change it and save

- **Expected:** Reason stays "original"

#### BeforeWorkflowActionTest

**36. test\_guards**

Call the browser-path guard directly:

1. Completed with Drawing Required
2. Same with Priority Urgent
3. Feasibility Approve with COGS 0
4. Send for Approval with COGS 0
5. Send Quotation to Client with Approval Required
6. Lost

- **Expected:** Blocked; allowed; blocked with "Cost of Goods"; allowed; blocked with "requires approval"; allowed

#### RevisionTest

**37. test\_items\_change\_bumps\_revision\_and\_requeues\_shopify**

1. Sent quotation with follow-up day 3 already sent
2. Change an item rate to 90, run revision handling

- **Expected:** Revision 1 ("Rev 1"), follow-up days reset, Shopify draft update queued

**38. test\_no\_item\_change\_no\_revision**

1. Sent quotation, run revision handling with no change

- **Expected:** No revision, no Shopify update

**39. test\_rev3\_blocks\_further\_changes**

1. Sent quotation at revision 3, change item qty, run revision handling

- **Expected:** Blocked with "revised 3 times"

**40. test\_revision\_ignored\_outside\_sent\_to\_customer**

1. Unsent quotation, change item rate, run revision handling

- **Expected:** No revision

**41. test\_item\_edit\_after\_send\_is\_rejected**

1. Sent quotation, change item rate, save

- **Expected:** Rejected (items cannot change after submit)

#### ShopifyLinkTest

**42. test\_changed\_shopify\_link\_enqueues\_linking**

1. Standard quotation, paste a Shopify admin order link, save

- **Expected:** Shopify order-link job queued

**43. test\_invalid\_shopify\_link\_rejected**

1. Standard quotation, paste a non-Shopify URL, save

- **Expected:** Blocked with "Shopify Order Link"

### test\_payments (27)

#### PaymentScheduleTest

**1. test\_amounts\_follow\_grand\_total**

1. Staged quotation at 1,000
2. Change the item rate to 2,000, save

- **Expected:** Amounts 400 / 300 / 300, then 800 / 600 / 600

**2. test\_zero\_portion\_rejected**

1. Staged quotation, set row 2 portion to 0%, validate

- **Expected:** Blocked with "zero percentage"

**3. test\_row\_without\_stage\_rejected**

1. Staged quotation, untick Before Dispatch on row 3, save

- **Expected:** Blocked with "no trigger event"

**4. test\_amounts\_must\_sum\_to\_total**

1. Staged quotation, set row 3 amount to 250, validate

- **Expected:** Blocked with "sum to"

**5. test\_stage\_checkboxes\_copied\_from\_template**

1. Create a Payment Terms Template with Before Dispatch ticked
2. Create a quotation using it
3. Untick Before Dispatch, tick On Delivery, save

- **Expected:** Before Dispatch copied from the template; manual change kept after save

#### ProcessingFeeTest

**6. test\_fee\_added\_removed\_and\_not\_compounded**

1. Set the processing fee to 3%, create a standard quotation
2. Tick Apply Additional Charges, save
3. Save again
4. Untick, save

- **Expected:** No fee row; fee row = 3% of total; still 3% after re-save; fee row removed and total restored

#### StagePaymentEnforcementTest

**7. test\_underpaid\_blocks\_and\_tracks\_on\_lyfe\_order**

1. Staged quotation at 1,000 with a linked Lyfe Order
2. Record 400 (Advance only)
3. Check the Before Dispatch stage (needs 1,000)

- **Expected:** Blocked with "Payment required"; Lyfe Order block stage and date set; block reminder queued

**8. test\_repeat\_block\_does\_not\_re\_notify**

1. Staged quotation with a linked Lyfe Order, no payment; check Before Dispatch twice

- **Expected:** Blocked both times, one reminder queued

**9. test\_cumulative\_threshold\_met\_passes**

1. Staged quotation, record 700, check Photo Approval (needs 400 + 300)

- **Expected:** Passes

**10. test\_non\_cs\_user\_gets\_generic\_message**

1. Staged quotation, no payment; as a Factory user check Advance

- **Expected:** Blocked with generic "payment is pending"

**11. test\_unconfigured\_stage\_is\_noop**

1. Staged quotation, check On Delivery (not in the schedule)

- **Expected:** Passes

#### EnforcePaymentStateTest

**12. test\_fully\_paid\_stays\_won**

1. Standard quotation, record full payment, set Won, run the payment-state check

- **Expected:** Stays Won

**13. test\_underpaid\_becomes\_partially\_paid**

1. Same, recording only 10

- **Expected:** Changed to Partially Paid, Total Received 10

**14. test\_unpaid\_is\_rejected**

1. Same, no payment

- **Expected:** Blocked with "no payment has been recorded"

#### PaymentGateReleaseTest

**15. test\_lost\_or\_cancelled\_clears\_block**

1. For Lost and Cancelled: linked Lyfe Order with a block and 3 reminders; move the quotation from Sent to Customer to that state

- **Expected:** Block stage cleared, reminder count 0

**16. test\_non\_terminal\_state\_keeps\_block**

1. Same, moving to Won

- **Expected:** Block stays

#### MilestoneStatusTest

**17. test\_status\_progression**

1. Staged quotation, update milestone statuses
2. Set Partially Paid, update
3. Record 400, update with the Lyfe Order Approved

- **Expected:** Upcoming ×3; Due / Upcoming / Upcoming with Due notification queued; Paid / Due / Upcoming

**18. test\_overdue\_and\_escalated\_follow\_reminder\_count**

1. Won staged quotation, linked Lyfe Order with 2 reminders, update
2. Raise to 5 reminders, update

- **Expected:** First milestone Overdue, then Escalated

**19. test\_due\_notification\_only\_on\_fresh\_due**

1. Won staged quotation, update statuses twice

- **Expected:** One Due notification queued

#### NoRemindersWhenClosedTest

**20. test\_block\_reminder\_skipped\_and\_cleared\_for\_lost**

1. Staged quotation, linked Lyfe Order blocked at Before Dispatch
2. Set the quotation Lost directly (as the follow-up scheduler does)
3. Send the block reminder

- **Expected:** No email sent; block and reminder count cleared

**21. test\_block\_reminder\_still\_sent\_for\_open\_quotation**

1. Same, with the quotation left open

- **Expected:** One email sent (control)

**22. test\_due\_and\_stage\_notifications\_skipped\_for\_cancelled**

1. Staged quotation with a Before Shipped stage and a linked Lyfe Order
2. Set Cancelled
3. Send the Due and Before Shipped notifications

- **Expected:** No email sent

#### PartialPaymentDueTest

**23. test\_partial\_payment\_marks\_advance\_due\_and\_reminds**

1. Staged quotation → sent → Bank Transfer Selected
2. Confirm Payment dialog: 200, Partial Payment

- **Expected:** Partially Paid; Due / Upcoming / Upcoming; Advance Due notification queued

**24. test\_full\_payment\_marks\_all\_paid**

1. Same, confirming the full total as Full Payment

- **Expected:** Won; all milestones Paid

#### ReminderOnlyStagesTest

**25. test\_unpaid\_shipped\_stage\_does\_not\_block\_dispatch**

1. Staged quotation Advance / Before Dispatch / Before Shipped, record 700, check Before Dispatch

- **Expected:** Passes even though Before Shipped is unpaid

**26. test\_unpaid\_shipped\_stage\_sends\_reminder**

1. Same, with a linked Lyfe Order; send the Before Shipped notification

- **Expected:** One email sent; no payment block set

**27. test\_order\_code\_never\_hard\_blocks\_reminder\_only\_stages**

1. Scan the Lyfe Order code for every stage-payment gate

- **Expected:** Gates exist, and none use Before Shipped or On Delivery

### test\_integrations (24)

#### SendQuotationEmailTest

**1. test\_no\_outgoing\_account\_skips**

1. Disable the outgoing email account, send the quotation email for a sent quotation

- **Expected:** Returns "no\_email\_account"; nothing sent

**2. test\_sends\_branded\_email\_and\_tracks**

1. Sent quotation, payment link on, send the email

- **Expected:** "sent"; card pay link included; email to the contact; Email Sent, Sent At and tracking token stored; Shopify payment poll queued

**3. test\_bank\_transfer\_never\_gets\_card\_link**

1. Sent quotation with Payment via Bank Transfer, send the email

- **Expected:** Bank-transfer link included, no card link

**4. test\_requires\_contact\_email**

1. Standard quotation with no contact, send the email

- **Expected:** Blocked with "No contact email"

#### EmailOpenTrackingTest

**5. test\_pixel\_records\_first\_open\_only**

1. Sent quotation, open the tracking pixel with a valid token
2. Open it again

- **Expected:** Email Opened set, GIF returned; Opened At unchanged on the second open

**6. test\_bad\_token\_records\_nothing**

1. Sent quotation, open the pixel with an invalid token

- **Expected:** Email Opened stays unset

#### CustomerApprovalLinkTest

**7. test\_valid\_token\_marks\_won**

1. Sent quotation, open the approval link with a valid token
2. Open it again

- **Expected:** "Approved" page and state Won; then "Already Approved"

**8. test\_bad\_or\_missing\_token\_rejected**

1. Sent quotation, open the link with a forged token, then with none

- **Expected:** "Invalid or Expired Link", "Invalid Link"; state stays Sent to Customer

#### ShopifyPollingTest

**9. test\_paid\_draft\_records\_payment\_and\_wins**

1. Sent quotation with a Shopify draft; poll returns a completed draft whose order is paid in full

- **Expected:** Won; Shopify order name stored; one payment row for the full total

**10. test\_partially\_paid\_order**

1. Same, order partially paid at half the total

- **Expected:** Partially Paid

**11. test\_open\_draft\_changes\_nothing**

1. Same, draft still open

- **Expected:** Stays Sent to Customer; no payment rows

**12. test\_pending\_bank\_transfer\_order\_not\_recorded**

1. Same, order status pending (bank transfer)

- **Expected:** No payment rows

#### FollowupSchedulerTest

**13. test\_followup\_days**

1. Emailed sent quotation, run the scheduler at business day 2, 3, 10 and 25

- **Expected:** No task on day 2; follow-up tasks on days 3, 10 and 25

**14. test\_day\_already\_sent\_is\_skipped**

1. Day 3 already recorded as sent, run at day 4

- **Expected:** No task

**15. test\_day\_30\_final\_task\_then\_lost**

1. Run at day 30

- **Expected:** Final day-30 task created; quotation Lost

**16. test\_valid\_till\_passed\_expires**

1. Valid Till set to yesterday, run at day 3

- **Expected:** No task; quotation Expired

**17. test\_never\_emailed\_is\_skipped**

1. Sent quotation with no Email Sent At, run at day 30

- **Expected:** No task

#### SlaAndCogsWiringTest

**18. test\_sla\_close\_check\_runs\_on\_update\_and\_submit**

1. Quotation with an active SLA Task Link
2. Save, Inquiry Approved, Send Quotation to Client

- **Expected:** SLA close check runs at least 3 times (save, workflow save, submit)

**19. test\_cogs\_deviation\_enqueues\_alert\_only\_when\_cogs\_changes**

1. Benchmark returns a large deviation, save without changing COGS
2. Change item COGS to 999, save

- **Expected:** No alert; then deviation alert email queued

#### PermissionQueryTest

**20. test\_admins\_see\_everything**

1. Get the list filter for System Manager, and for Super Admin + Senior Engineer

- **Expected:** No filter

**21. test\_senior\_engineer\_scoped\_to\_assigned\_review\_states**

1. Get the filter for Senior Engineer + Sales User, run it as SQL

- **Expected:** Filter names the user and Senior Review Pending / Pending Engineer Review; SQL is valid

**22. test\_other\_roles\_unrestricted**

1. Get the filter for Sales User

- **Expected:** No filter

#### LinkedOrdersAndDashboardTest

**23. test\_linked\_lyfe\_orders**

1. Standard quotation with a linked Lyfe Order, fetch linked orders

- **Expected:** Returns that Lyfe Order

**24. test\_dashboard\_shows\_timesheets**

1. Read the Quotation dashboard config

- **Expected:** Timesheet listed, linked via the custom Quotation field

### test\_validate (21)

#### OrderTypeDetectionTest

**1. test\_plain\_item\_is\_standard**

1. Create a standard quotation

- **Expected:** Order Type Standard

**2. test\_custom\_prefix\_in\_item\_name\_is\_custom**

1. Create a quotation for an item named "CSTM Bracket" (custom prefixes "cstm", "custom")

- **Expected:** Order Type Custom

**3. test\_is\_custom\_order\_checkbox\_is\_custom**

1. Create a quotation with Is Custom Order ticked on the item

- **Expected:** Order Type Custom

#### ItemDimensionTest

**4. test\_dims\_from\_item\_master**

1. Item master box 30 x 20 x 10 cm, weight 2; create a quotation

- **Expected:** Row box 30 / 20 / 10, weight 2, not flagged

**5. test\_manual\_row\_dims\_are\_kept**

1. Same item, row box sizes entered as 5 x 5 x 5

- **Expected:** Row keeps 5

**6. test\_bom\_with\_missing\_child\_dims\_is\_flagged**

1. Item with no box size and a BOM whose child CHILD-1 has no sizes; create a quotation

- **Expected:** Row flagged with CHILD-1 in the reason; no partial size applied

#### CombinedBoxTest

**7. test\_combined\_box\_and\_parcels\_from\_catalogue**

1. 3D bin packing and Paccurate off; quotation for 2 × a 30 x 20 x 10 item

- **Expected:** Combined box set, Box Catalogue match selected, "Best fit" suggestion, single-box size on the row, parcel plan built

**8. test\_3dbinpacking\_takes\_priority\_when\_enabled**

1. Same with 3D bin packing on (returns one 40 cm parcel)

- **Expected:** 3D bin packing called; parcels = one 40 cm box

**9. test\_paccurate\_used\_when\_3dbin\_off**

1. Same with only Paccurate on (returns two 50 cm parcels and an SVG)

- **Expected:** Parcels = two 50 cm boxes; Paccurate SVG stored

#### ApprovalAndCogsFlagTest

**10. test\_missing\_cogs\_sets\_cogs\_pending**

1. Create a quotation with item COGS 0

- **Expected:** COGS Pending set

**11. test\_grand\_total\_over\_10k\_requires\_approval**

1. Create a quotation at rate 12,000

- **Expected:** Approval Required set

**12. test\_approval\_not\_reraised\_once\_approved**

1. 12,000 quotation, jump to Approved – Pending Send, save

- **Expected:** Approval Required stays 0

#### DiscountViolationE2ETest

**13. test\_card\_payment\_over\_5pct\_needs\_approval**

1. Item and Item Group with no discount limit; 6% discount, card payment

- **Expected:** Approval Required set (over the 5% card cap)

**14. test\_bank\_transfer\_allows\_up\_to\_8pct**

1. Same with Payment via Bank Transfer

- **Expected:** Approval Required stays 0 (under the 8% cap)

**15. test\_item\_group\_limit\_applies\_when\_item\_has\_none**

1. Item has no limit, Item Group allows 15%; check 12% then 16%

- **Expected:** 12% allowed, 16% a violation

**16. test\_item\_limit\_beats\_item\_group**

1. Item allows 20%, Item Group 5%; check 15%

- **Expected:** Allowed (Item limit wins)

#### AutoClearAndCleanupTest

**17. test\_drawing\_clears\_drawing\_and\_photo\_required**

1. Quotation with Drawing Required and item Photo Required; attach the drawing with details, save

- **Expected:** Both flags cleared

**18. test\_igst\_row\_is\_stripped**

1. Add an IGST Payable tax row, save

- **Expected:** IGST row removed

**19. test\_defaults\_terms\_template**

1. Create a standard quotation

- **Expected:** Payment terms "100 % Advance"; T&C "Terms and Conditions template"

**20. test\_after\_insert\_rows\_added\_once**

1. Create a standard quotation, save again

- **Expected:** One Shipping Charges row and one Custom Fee row

**21. test\_after\_insert\_clears\_copied\_shopify\_and\_email\_fields**

1. Create a quotation carrying a Shopify draft ID, payment URL and Email Sent (as a duplicate would)

- **Expected:** All three cleared

### test\_drawings (17)

#### DrawingUploadGateTest

**1. test\_engineer\_without\_timesheet\_is\_blocked**

1. As an Engineer with no Timesheet, attach a PDF to Order Drawing

- **Expected:** Blocked with "log a Timesheet"

**2. test\_engineer\_with\_timesheet\_is\_allowed**

1. Same, with a Timesheet logged

- **Expected:** File attached

**3. test\_customer\_service\_upload\_queues\_engineering\_task**

1. As Customer Service, attach a PDF to Order Drawing

- **Expected:** Engineering Task created for the quotation

**4. test\_other\_fields\_are\_not\_gated**

1. As an Engineer with no Timesheet, attach a PDF to Order Drawing with BOM

- **Expected:** File attached

#### PdfBomStripperTest

**5. test\_bom\_page\_stripped**

1. Upload a 2-page PDF (drawing + "BOM / Breakdown of Components") as Order Drawing, save

- **Expected:** Uploaded file keeps only the drawing page; Order Drawing with BOM filled

**6. test\_original\_with\_bom\_kept**

1. Same upload, open the Order Drawing with BOM file

- **Expected:** **Expected failure**: should keep both pages, but on local storage the copy shares the upload's file and is stripped too (S3 hides this)

**7. test\_pdf\_without\_bom\_left\_alone**

1. Upload a 1-page PDF with no BOM page

- **Expected:** Untouched; Order Drawing with BOM stays empty

**8. test\_unchanged\_drawing\_not\_reprocessed**

1. After the BOM upload, save again without changing the drawing

- **Expected:** Stripper not called

#### DrawingChangeDetailsTest

**9. test\_first\_upload\_needs\_details\_then\_notifies**

1. Set Order Drawing with no change details, save
2. Add details "rev A", save

- **Expected:** Blocked with "Drawing Change Details is mandatory"; then one notification email to support@chandakbrothers.net

**10. test\_clear\_then\_reattach\_counts\_as\_replacement**

1. Attach a drawing with details
2. Clear it, save
3. Attach a new drawing with no details, save

- **Expected:** Replacement Pending set; save blocked with "Drawing Change Details is mandatory"

**11. test\_popup\_flow\_mark\_pending\_then\_submit\_details**

1. Mark change pending, attach a drawing, save
2. Submit blank details
3. Submit "moved holes 5mm"

- **Expected:** Save goes through with Pending set; blank rejected ("cannot be empty"); details stored, Pending cleared, Comment added

**12. test\_cancel\_upload\_reverts\_drawing**

1. Mark change pending, set a drawing, Cancel Upload

- **Expected:** Drawing removed, Pending cleared

**13. test\_endpoints\_reject\_unknown\_quotation**

1. Call Mark Pending and Cancel Upload for a quotation that doesn't exist

- **Expected:** Both rejected with "does not exist"

#### GraceSweepTest

**14. test\_sweep**

1. Three pending uploads: 20 min old with no details, 5 min old, 20 min old with details filled; run the grace sweep

- **Expected:** 20-min unanswered: drawing reverted with an "automatically reverted" Comment; 5-min: untouched; answered: drawing kept, flag cleared

#### SeniorReviewTest

**15. test\_request\_and\_complete\_senior\_review**

1. Custom-order quotation in Pending Engineer Review
2. Request Senior Review with no Reason Code
3. Request with "New geometry"
4. Request again
5. Submit the review "Approved as-is"

- **Expected:** Blocked ("Reason Code"); Senior Review Pending with the log linked; second request blocked; back to Pending Engineer Review. Skipped if the site has no Product Finish

**16. test\_spec\_change\_after\_approval\_routes\_back**

1. Quotation in Pending Approval → Approve
2. Change item qty to 3, save

- **Expected:** Spec hash stored on approval; then Pending Engineer Review with the hash cleared

**17. test\_non\_spec\_change\_keeps\_approval**

1. Approved quotation, change Priority to VIP, save

- **Expected:** Stays Approved – Pending Send

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
- [ ] In the Quotation-3 Workflow form, set both "Sales User Review → Send for Approval" conditions to `doc.approval_required == 1`
