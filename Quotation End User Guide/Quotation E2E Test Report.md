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

All 132 test cases, grouped by module. All pass unless marked Expected failure.

### test\_lifecycle (43)

| Class | Test case | Checks |
| --- | --- | --- |
| HappyPathTest | test\_inquiry\_to\_won\_to\_cancelled | Inquiry → Inquiry Approved → Sent to Customer → Bank Transfer → Won → Cancelled; Shipping and Custom Fee rows added on insert |
| HappyPathTest | test\_partial\_payment\_then\_remaining | Confirm Payment below total → Partially Paid; Confirm Remaining → Won |
| HappyPathTest | test\_sent\_to\_customer\_needs\_email\_and\_address | Send to Client blocked without contact email, then without address |
| PostSubmitHooksTest | test\_cancel\_releases\_payment\_gates | Cancel clears the payment block on linked Lyfe Orders |
| PostSubmitHooksTest | test\_lost\_releases\_payment\_gates | Lost clears the payment block on linked Lyfe Orders |
| PostSubmitHooksTest | test\_won\_with\_no\_payment\_is\_blocked | Mark as Won with no payment recorded is blocked |
| PostSubmitHooksTest | test\_won\_marks\_advance\_milestone\_paid | Advance milestone shows Paid after Won |
| ApprovalPathTest | test\_over\_10k\_needs\_approval\_then\_sends | 10,000 or more needs approval; Approve clears the flag and binds the spec hash; then Send to Client |
| ApprovalPathTest | test\_send\_back\_to\_sales\_user | Approver sends back → Sales User Review |
| ApprovalPathTest | test\_slack\_card\_from\_feasibility\_approved | Card posted from Feasibility Approved |
| ApprovalPathTest | test\_slack\_card\_off\_when\_setting\_disabled | No card when Enable Quotation Approval is off |
| ApprovalPathTest | test\_fee\_only\_quote\_sent\_for\_approval\_from\_inquiry\_approved | 0-rate + Custom Fee quotation, no COGS, can be sent for approval |
| ApprovalPathTest | test\_fee\_only\_quote\_sent\_for\_approval\_from\_sales\_user\_review | Same, from Sales User Review |
| ApprovalPathTest | test\_no\_cogs\_needed\_from\_feasibility\_approved | Same, from Feasibility Approved |
| ApprovalPathTest | test\_slack\_card\_from\_every\_route | Card from Inquiry Approved, Sales User Review and Factory Feasibility Approve; queued after commit |
| ApprovalPathTest | test\_no\_second\_card\_while\_already\_pending | Re-saving in Pending Approval posts no second card |
| ApprovalPathTest | test\_erp\_approve\_closes\_slack\_card | Approving in ERPNext closes the Slack card (added separately) |
| ApprovalPathTest | test\_slack\_approve\_does\_not\_double\_close\_card | Approving from Slack does not close the card twice (added separately) |
| ApprovalPathTest | test\_slack\_approve\_rejected\_by\_erp\_replies\_in\_thread | ERPNext rejecting a Slack approval replies in the thread (added separately) |
| EngineeringBranchTest | test\_drawing\_required\_routes\_to\_engineer\_then\_sales | Drawing Required → Pending Engineer Review; attaching clears the flag; Completed → Sales User Review |
| EngineeringBranchTest | test\_completed\_blocked\_without\_drawing | Completed without drawing blocked (server path, as bulk action / API) |
| EngineeringBranchTest | test\_completed\_to\_production\_review\_blocked\_without\_drawing | Same when Completed leads to Production Details Review |
| EngineeringBranchTest | test\_completed\_blocked\_by\_item\_photo\_required | Item-level Photo Required also needs the drawing |
| EngineeringBranchTest | test\_urgent\_completes\_without\_drawing | Urgent quotations are exempt |
| EngineeringBranchTest | test\_need\_more\_information\_not\_blocked | Need More Information is not blocked |
| EngineeringBranchTest | test\_feasibility\_flag\_auto\_advances\_to\_production\_review | Ticking Feasibility Test Required moves the quotation to Production Details Review |
| EngineeringBranchTest | test\_feasibility\_flag\_auto\_clears\_when\_cogs\_and\_shipping\_set | Flag clears once COGS and shipping (or free shipping) are set |
| EngineeringBranchTest | test\_production\_review\_needs\_shipping\_unless\_free | Feasibility Approve blocked without shipping, allowed with Free Shipping |
| EngineeringBranchTest | test\_feasibility\_approve\_needs\_cogs\_whatever\_the\_target | Feasibility Approve blocked without COGS, to Feasibility Approved and to Pending Approval |
| EngineeringBranchTest | test\_feasibility\_approve\_over\_10k\_goes\_to\_pending\_approval | 10,000 or more lands on Pending Approval |
| EngineeringBranchTest | test\_photo\_required\_in\_production\_review\_routes\_back\_to\_engineer | Photo Required in Production Details Review routes back to Pending Engineer Review |
| EngineeringBranchTest | test\_request\_senior\_review\_only\_for\_custom\_orders | Request Senior Review unavailable for Standard orders |
| RejectionBranchTest | test\_inquiry\_rejected\_needs\_reason | Inquiry Rejected needs a reason |
| RejectionBranchTest | test\_not\_feasible\_needs\_reason\_then\_cs\_review | Not Feasible needs a reason; CS Review → Sales User Review |
| RejectionBranchTest | test\_not\_feasible\_reason\_locked\_to\_factory | Non-Factory edits to the reason are reverted |
| BeforeWorkflowActionTest | test\_guards | Browser-path guards: drawing, COGS on Feasibility Approve, approval before Send to Client; no COGS gate on Send for Approval |
| RevisionTest | test\_items\_change\_bumps\_revision\_and\_requeues\_shopify | Item change in Sent to Customer → Rev 1, follow-ups reset, Shopify draft update queued |
| RevisionTest | test\_no\_item\_change\_no\_revision | No change → no revision |
| RevisionTest | test\_rev3\_blocks\_further\_changes | Rev 3 blocks further changes |
| RevisionTest | test\_revision\_ignored\_outside\_sent\_to\_customer | No revision outside Sent to Customer |
| RevisionTest | test\_item\_edit\_after\_send\_is\_rejected | Items cannot be edited after Send |
| ShopifyLinkTest | test\_changed\_shopify\_link\_enqueues\_linking | Pasting a Shopify order link queues order linking |
| ShopifyLinkTest | test\_invalid\_shopify\_link\_rejected | Non-Shopify link rejected |

### test\_payments (27)

| Class | Test case | Checks |
| --- | --- | --- |
| PaymentScheduleTest | test\_amounts\_follow\_grand\_total | Milestone amounts = total × portion, recalculated when the total changes |
| PaymentScheduleTest | test\_zero\_portion\_rejected | 0% row rejected |
| PaymentScheduleTest | test\_row\_without\_stage\_rejected | Row with no stage checkbox rejected |
| PaymentScheduleTest | test\_amounts\_must\_sum\_to\_total | Amounts not summing to the total rejected |
| PaymentScheduleTest | test\_stage\_checkboxes\_copied\_from\_template | Checkboxes copied from a template with Payment Term set; manual edits kept |
| ProcessingFeeTest | test\_fee\_added\_removed\_and\_not\_compounded | Fee = % of total, removed when unticked, not stacked on re-save |
| StagePaymentEnforcementTest | test\_underpaid\_blocks\_and\_tracks\_on\_lyfe\_order | Underpaid stage blocks, sets the order's block fields, queues a reminder |
| StagePaymentEnforcementTest | test\_repeat\_block\_does\_not\_re\_notify | Repeated block sends one reminder |
| StagePaymentEnforcementTest | test\_cumulative\_threshold\_met\_passes | Cumulative payment at threshold passes |
| StagePaymentEnforcementTest | test\_non\_cs\_user\_gets\_generic\_message | Non-CS users see a generic message |
| StagePaymentEnforcementTest | test\_unconfigured\_stage\_is\_noop | Stage not in the schedule never blocks |
| EnforcePaymentStateTest | test\_fully\_paid\_stays\_won | Fully paid stays Won |
| EnforcePaymentStateTest | test\_underpaid\_becomes\_partially\_paid | Underpaid Won corrected to Partially Paid |
| EnforcePaymentStateTest | test\_unpaid\_is\_rejected | Won with no payment rejected |
| PaymentGateReleaseTest | test\_lost\_or\_cancelled\_clears\_block | Lost and Cancelled clear the block and reminder count |
| PaymentGateReleaseTest | test\_non\_terminal\_state\_keeps\_block | Other states keep the block |
| MilestoneStatusTest | test\_status\_progression | Upcoming → Due → Paid, Due notification queued |
| MilestoneStatusTest | test\_overdue\_and\_escalated\_follow\_reminder\_count | 2 reminders → Overdue, 5 → Escalated |
| MilestoneStatusTest | test\_due\_notification\_only\_on\_fresh\_due | Due notification sent once |
| NoRemindersWhenClosedTest | test\_block\_reminder\_skipped\_and\_cleared\_for\_lost | Lost quotation: no block reminder, block cleared |
| NoRemindersWhenClosedTest | test\_block\_reminder\_still\_sent\_for\_open\_quotation | Open quotation still gets the reminder (control) |
| NoRemindersWhenClosedTest | test\_due\_and\_stage\_notifications\_skipped\_for\_cancelled | Cancelled: no Due or Before Shipped notification |
| PartialPaymentDueTest | test\_partial\_payment\_marks\_advance\_due\_and\_reminds | Partial Confirm Payment → Advance Due + reminder |
| PartialPaymentDueTest | test\_full\_payment\_marks\_all\_paid | Full payment → Won, all milestones Paid |
| ReminderOnlyStagesTest | test\_unpaid\_shipped\_stage\_does\_not\_block\_dispatch | Unpaid Before Shipped never blocks Ready for Dispatch |
| ReminderOnlyStagesTest | test\_unpaid\_shipped\_stage\_sends\_reminder | Unpaid Before Shipped sends a reminder, sets no block |
| ReminderOnlyStagesTest | test\_order\_code\_never\_hard\_blocks\_reminder\_only\_stages | No Lyfe Order gate uses Before Shipped or On Delivery |

### test\_integrations (24)

| Class | Test case | Checks |
| --- | --- | --- |
| SendQuotationEmailTest | test\_no\_outgoing\_account\_skips | No outgoing account → nothing sent |
| SendQuotationEmailTest | test\_sends\_branded\_email\_and\_tracks | Email to contact, sent flags + tracking token stored, payment poll queued |
| SendQuotationEmailTest | test\_bank\_transfer\_never\_gets\_card\_link | Bank-transfer quotations get no card payment link |
| SendQuotationEmailTest | test\_requires\_contact\_email | No contact email → error |
| EmailOpenTrackingTest | test\_pixel\_records\_first\_open\_only | Open pixel records the first open only |
| EmailOpenTrackingTest | test\_bad\_token\_records\_nothing | Bad token records nothing |
| CustomerApprovalLinkTest | test\_valid\_token\_marks\_won | Valid approval link → Won; second click → Already Approved |
| CustomerApprovalLinkTest | test\_bad\_or\_missing\_token\_rejected | Forged or missing token rejected, state unchanged |
| ShopifyPollingTest | test\_paid\_draft\_records\_payment\_and\_wins | Paid Shopify draft → payment row recorded, Won |
| ShopifyPollingTest | test\_partially\_paid\_order | Partially paid order → Partially Paid |
| ShopifyPollingTest | test\_open\_draft\_changes\_nothing | Unpaid draft changes nothing |
| ShopifyPollingTest | test\_pending\_bank\_transfer\_order\_not\_recorded | Pending (bank transfer) order not recorded as paid |
| FollowupSchedulerTest | test\_followup\_days | Follow-up tasks on business days 3, 10, 25 |
| FollowupSchedulerTest | test\_day\_already\_sent\_is\_skipped | Day already sent is skipped |
| FollowupSchedulerTest | test\_day\_30\_final\_task\_then\_lost | Day 30 → final task, then Lost |
| FollowupSchedulerTest | test\_valid\_till\_passed\_expires | Past Valid Till → Expired |
| FollowupSchedulerTest | test\_never\_emailed\_is\_skipped | Never emailed → skipped |
| SlaAndCogsWiringTest | test\_sla\_close\_check\_runs\_on\_update\_and\_submit | SLA auto-close runs on save, workflow save and submit |
| SlaAndCogsWiringTest | test\_cogs\_deviation\_enqueues\_alert\_only\_when\_cogs\_changes | COGS benchmark alert only when COGS changes |
| PermissionQueryTest | test\_admins\_see\_everything | System Manager / Super Admin unrestricted |
| PermissionQueryTest | test\_senior\_engineer\_scoped\_to\_assigned\_review\_states | Senior Engineer sees only assigned quotations in review states |
| PermissionQueryTest | test\_other\_roles\_unrestricted | Other roles unrestricted |
| LinkedOrdersAndDashboardTest | test\_linked\_lyfe\_orders | Linked Lyfe Orders listed |
| LinkedOrdersAndDashboardTest | test\_dashboard\_shows\_timesheets | Connections tab shows Timesheets |

### test\_validate (21)

| Class | Test case | Checks |
| --- | --- | --- |
| OrderTypeDetectionTest | test\_plain\_item\_is\_standard | Plain item → Standard |
| OrderTypeDetectionTest | test\_custom\_prefix\_in\_item\_name\_is\_custom | Custom prefix in item name → Custom |
| OrderTypeDetectionTest | test\_is\_custom\_order\_checkbox\_is\_custom | Is Custom Order ticked → Custom |
| ItemDimensionTest | test\_dims\_from\_item\_master | Box size and weight filled from the Item master |
| ItemDimensionTest | test\_manual\_row\_dims\_are\_kept | Manual row sizes kept |
| ItemDimensionTest | test\_bom\_with\_missing\_child\_dims\_is\_flagged | BOM with a child missing sizes is flagged, no partial size |
| CombinedBoxTest | test\_combined\_box\_and\_parcels\_from\_catalogue | Combined box, catalogue match, suggestions, parcel plan |
| CombinedBoxTest | test\_3dbinpacking\_takes\_priority\_when\_enabled | 3D bin packing used when enabled |
| CombinedBoxTest | test\_paccurate\_used\_when\_3dbin\_off | Paccurate used when 3D bin packing is off |
| ApprovalAndCogsFlagTest | test\_missing\_cogs\_sets\_cogs\_pending | Missing COGS → COGS Pending |
| ApprovalAndCogsFlagTest | test\_grand\_total\_over\_10k\_requires\_approval | 10,000 or more → Approval Required |
| ApprovalAndCogsFlagTest | test\_approval\_not\_reraised\_once\_approved | Not re-raised once approved |
| DiscountViolationE2ETest | test\_card\_payment\_over\_5pct\_needs\_approval | 6% discount on card payment needs approval |
| DiscountViolationE2ETest | test\_bank\_transfer\_allows\_up\_to\_8pct | 6% allowed for bank transfer |
| DiscountViolationE2ETest | test\_item\_group\_limit\_applies\_when\_item\_has\_none | Item Group limit applies when the Item has none |
| DiscountViolationE2ETest | test\_item\_limit\_beats\_item\_group | Item limit wins over Item Group |
| AutoClearAndCleanupTest | test\_drawing\_clears\_drawing\_and\_photo\_required | Attaching a drawing clears Drawing / Photo Required |
| AutoClearAndCleanupTest | test\_igst\_row\_is\_stripped | IGST tax row removed |
| AutoClearAndCleanupTest | test\_defaults\_terms\_template | Default payment terms and T&C set |
| AutoClearAndCleanupTest | test\_after\_insert\_rows\_added\_once | Shipping and Custom Fee rows added once |
| AutoClearAndCleanupTest | test\_after\_insert\_clears\_copied\_shopify\_and\_email\_fields | Duplicated quotations start with no Shopify / email data |

### test\_drawings (17)

| Class | Test case | Checks |
| --- | --- | --- |
| DrawingUploadGateTest | test\_engineer\_without\_timesheet\_is\_blocked | Engineer upload without a Timesheet blocked |
| DrawingUploadGateTest | test\_engineer\_with\_timesheet\_is\_allowed | With a Timesheet allowed |
| DrawingUploadGateTest | test\_customer\_service\_upload\_queues\_engineering\_task | CS upload creates the Engineering Task |
| DrawingUploadGateTest | test\_other\_fields\_are\_not\_gated | Other attachment fields not gated |
| PdfBomStripperTest | test\_bom\_page\_stripped | BOM page removed from the uploaded PDF |
| PdfBomStripperTest | test\_original\_with\_bom\_kept | Original with BOM kept separately — **Expected failure** |
| PdfBomStripperTest | test\_pdf\_without\_bom\_left\_alone | PDF without a BOM page untouched |
| PdfBomStripperTest | test\_unchanged\_drawing\_not\_reprocessed | Unchanged drawing not reprocessed |
| DrawingChangeDetailsTest | test\_first\_upload\_needs\_details\_then\_notifies | Change details required; notification email sent |
| DrawingChangeDetailsTest | test\_clear\_then\_reattach\_counts\_as\_replacement | Clear then re-attach counts as a replacement |
| DrawingChangeDetailsTest | test\_popup\_flow\_mark\_pending\_then\_submit\_details | Popup flow: pending, then details saved with a Comment |
| DrawingChangeDetailsTest | test\_cancel\_upload\_reverts\_drawing | Cancel Upload reverts the drawing |
| DrawingChangeDetailsTest | test\_endpoints\_reject\_unknown\_quotation | Endpoints reject unknown quotations |
| GraceSweepTest | test\_sweep | After 15 minutes without details the upload is reverted; within 15 minutes untouched |
| SeniorReviewTest | test\_request\_and\_complete\_senior\_review | Request → Senior Review Pending; complete → back to Pending Engineer Review |
| SeniorReviewTest | test\_spec\_change\_after\_approval\_routes\_back | Spec change after approval clears it and routes back |
| SeniorReviewTest | test\_non\_spec\_change\_keeps\_approval | Non-spec change keeps approval |

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
