# Manual test plan: restored prod copy (`lyfelocal.com`, local bench)

**Code:** `apps/lh` `prod` at `12844570` (includes 5.5 Desk JS and print formats).
**Setup already done (12 Nov):** latest prod DB restored and migrated; both Email Accounts outgoing off, Email Queue suspended; Slack webhooks and bot token blanked; scheduler on; Shopify / ShipStation / 1Click / Gorgias / 17Track keys still live, so **read-only actions only**; S3 live, so upload tests are fine.

**Do not click on this copy:** order sync / re-register buttons, 1Click add-SKU (Material Issue submit), CJ label purchase, "Resend email", quotation Approve that calls Shopify.

**Test users:** `sec-test-forwarder` (Shipping Forwarder only), `sec-test-employee` (Employee only), plus real accounts: Customer Service, Factory, one System Manager. Use a private window per user.

Mark each row ✅ / ❌ and note the record you used. Anything ❌ is fixed on this copy first.

---

## Part A: 5.5 escaping (new, 12 Nov)

### A0. Prepare test data (local copy only)
1. Create or pick one Quotation, one Lyfe Order, one Lyfe Order Return and one Sales Invoice, and set the customer name to each of these, one test each:
   - `D J Johnson & Co, Inc.`
   - `O'Brien <b>bold</b>`
   - `Click <a href="https://example.com">here</a>`
2. For the print formats, also put the same strings into: the Export Party full address, the consignee and buyer address on a Bulk Transfer Batch, `custom_description_of_goods` on a Sales Invoice, and one shipping address line on a Lyfe Order.

**Expected everywhere:** the text shows literally (`<b>bold</b>` visible, link not clickable), `&` shows as `&`, apostrophe shows as `'`, and nothing shows `&amp;` or `&lt;` as visible text.

### A1. Desk dashboards (open each page, find the test record)
| # | Page | Where to look | Pass |
|---|---|---|---|
| 1 | P&L dashboard | order detail table: customer, source, status; repeat-customer table: name and email; hover the customer cell (title) | |
| 2 | Returns dashboard | return list: customer, type, workflow state | |
| 3 | Quotation Analysis | Needs Attention table, all-quotations table: customer name, workflow state, order type; SLA rule rows; payment breakdown | |
| 4 | Customer Intelligence | search the test customer: order list (order type, status, source), quotation list, profile header link | |
| 5 | Factory Quotation dashboard | open the test quotation: the address block under the customer name | |
| 6 | Click a row link in each page (order or quotation name) | the form opens, no console error | |
| 7 | Browser console on all five pages | no new JS errors | |

### A2. Print formats and PDFs (Print preview, then Download PDF)
| # | Format | Check | Pass |
|---|---|---|---|
| 1 | Commercial Invoice, Sample Invoice, LH Tax Invoice, Packing List (Sales Invoice) | goods description with 2 lines: both lines show, a `<br>` break between them | |
| 2 | Same four | company address and shipping address still lay out as before (Frappe-rendered, left as `\|safe`) | |
| 3 | BTB Commercial Invoice, BTB Tax Invoice, BTB Sample Invoice, Packing List BTB, Bulk Transfer Batch Packing List (1 and 2) | consignee / buyer / exporter address with a name containing `&` and `<b>` shows literally, multi-line address keeps its line breaks | |
| 4 | Lyfe Export CI / PL / SI / TI, Packing List Export Doc | exporter and consignee address shows literally, line breaks kept | |
| 5 | Empty consignee / buyer address | prints blank, not the word `None` | |
| 6 | Pick-pack PDF from a Lyfe Order with the test address | each address line shows literally, lines separated | |
| 7 | Compare one old and one new PDF of the same invoice | layout is the same apart from the escaped text | |

### A3. Scripted check
Run `bench --site lyfelocal.com run-tests --app lh --module lh.tests.test_output_escaping` (existing 8 tests must still pass).

---

## Part B: Critical and High retest (6.2)

For each endpoint call it as the user shown. **Refused** = HTTP 403 (or a permission error); **Works** = HTTP 200.

Call form: `curl -s -o /dev/null -w "%{http_code}" -b cookie.txt -X POST -H "X-Frappe-CSRF-Token: <token>" https://<local-site>/api/method/<path>`

| # | Finding | Endpoint (prefix `lh.lyfe_hardware.`) | Forwarder | Employee | Allowed role |
|---|---|---|---|---|---|
| C3 | Fake payments | `integrations.shopify.complete_draft_order` | Refused | Refused | CS, System Manager works (on a test quotation in Bank Transfer Selected) |
| C4 | Shopify webhook hijack | `integrations.shopify.register_shopify_webhooks` | Refused | Refused | System Manager only. **Do not run as System Manager** (it registers real webhooks); a 403 for the others is enough |
| C6 | Billable CJ label | `shipping.cj_api.generate_cj_label` | Refused | Refused | Factory works. **Do not run the allowed case** (buys a label); the refusals are the test |
| C5 | Forged ShipStation import | `process_shipstation_response` | 404 (not whitelisted) | 404 | |
| C2 / H1 | Webhooks fail-open | POST to the Shopify and 17Track webhooks with no signature | 401/403 | | |
| H4 | Read-guarded writers | `doctype.bulk_transfer_batch.bulk_transfer_batch.send_batch_inbound_email`, `validation.engine.record_validation_feedback`, `doctype.lyfe_order.lyfe_order.check_load_status_for_order` | Refused | Refused | needs write on the doc; **do not run `send_batch_inbound_email` as an allowed user** (emails 1Click; email is off here, but still skip) |
| H2 A | `get_content` SSRF | attach a link File, open it | error, no outbound call | | |
| H2 B | S3 `?key=` | `generate_file?key=<key of a private Lyfe Order file>` | 403 | 403 | CS, Factory open it; System Manager and Super Admin never blocked; unknown key 404 |
| 3.5 | Invoice PDFs | open an old `*-Invoice-*.pdf` link | refused if user cannot read the order | | CS opens it |
| 3.6 / H7 | Dashboard backends, exports | open Founder, Customer Intelligence and Quotation Analysis as Factory | blocked | | System Manager opens |
| 5.2 | POST-only | call one of the 91 POST-only APIs with GET | 405 | | POST works |

Also confirm: System Manager and Super Admin are never refused on any of these, except where the row says to skip.

---

## Part C: open items from the testing list (that can run here)

| Item | Check | Pass |
|---|---|---|
| 2 | The Part B 403 calls as `sec-test-forwarder` and `sec-test-employee` | |
| 4 | Error Log: no new "Secret Not Configured", "Key Not Configured", "failed to create Integration Request" after the session | |
| 9a | Open an old quotation approval link: "Already Approved" | |
| 9c | P&L page roles: System Manager and the intended roles can open it | |
| 13a | CS, Factory, System Manager can open the Status Overview page | |
| 14 | Upload an attachment (S3), open an existing S3 attachment, open an Item image | |
| 15b | As Factory and CS the 20 removed APIs are not needed by any page (no 403/404 in the request log while browsing) | |
| 17 | Open a print format that shows a signature image (uses `get_s3_image_url`) | |
| 20 | Migrate output count of invoice PDFs made private; old invoice link opens for CS and is refused for a user who cannot read the order | |

**Not testable here (prod only, passive):** 17Track and Shopify test pushes, ShipStation sync, 1Click posting, Frappe Cloud proxy behaviour. Watch prod's Error Log and request logs after a normal day instead.

---

## After testing
Tell Claude the results. Rows that pass get marked completed in `pending.md` / `pending_final.md` (6.2, 3.8, 5.10, 5.5). A failure is fixed on this copy first, then pushed.
