Subject: 1Click API — 2 confirmed bugs (Inventory 500, Tracking SQL error) + 4 open questions from our own Postman collection findings

Hi team,

We've completed integration testing against the sandbox environment (`https://icomwms.dev`) using our confirmed API key. Below are two confirmed server-side bugs, followed by a short list of remaining open questions — all with exact reproduction details so no clarification should be needed on your end.

Environment used for all tests below: base URL `https://icomwms.dev`, warehouse ID `1`, API key ending in the credential you provided us on 2026-08-17.

---

## BUG 1 — Inventory endpoint returns 500 Internal Server Error (both variants)

**Endpoint:** `GET /api/v1/inventory.cfm`
**Headers:** `Token: <our key>`, `Call-Type: Stock`, `Content-Type: application/json`

We tested this three separate times, including using your own real Postman collection example data, and got the same error every time:

**Test 1 — using your own documented example SKUs exactly as shown in your Postman collection ("Get Inventory by SKU"):**
```
Request body:
{
  "inventory": [{"SKU": "AD-56330015"}, {"SKU": "ADR-1415-10"}],
  "warehouseID": 1,
  "allWarehouses": false
}
```
Response: `500 Internal Server Error`
```json
{"content":{"message":"Internal Server Error","status":500}}
```

**Test 2 — using your "Get All Inventory" example pattern:**
```
Request body:
{
  "inventory": [{"SKU": "all"}],
  "warehouseID": 1,
  "allWarehouses": false
}
```
Response: same `500 Internal Server Error`.

**Test 3 — using a real item we registered ourselves via Add New Item (SKU `LH-FULLTEST-1`, itemID `224498`, confirmed to exist via Get Item by SKU):**
```
Request body:
{
  "inventory": [{"SKU": "LH-FULLTEST-1"}],
  "warehouseID": 1,
  "allWarehouses": false
}
```
Response: same `500 Internal Server Error`.

**Why we're confident this is a server-side bug and not a mistake on our end:** when we deliberately sent a malformed request (no body, or query parameters instead of a JSON body) we correctly got back `406 Body Not Acceptable` — proving the server is receiving and evaluating our request, not silently ignoring it. It's specifically when we send a **well-formed** request body — including your own documented example values — that it crashes with a 500.

**Ask:** Please reproduce this on your end using the exact request above (SKUs `AD-56330015` / `ADR-1415-10`, `warehouseID: 1`, against `icomwms.dev`) and let us know once it's fixed. This endpoint is required for our automatic US-warehouse stock-check logic, so we cannot proceed with that part of the integration until it's resolved.

---

## BUG 2 — Get Tracking by PO fails with a SQL syntax error, and the response leaks internal database schema

**Endpoint:** `GET /api/v2/orders`
**Headers:** `Token: <our key>`, `Call-Type: Tracking`, `Content-Type: application/json`

**Reproduction steps, in order:**

1. We submitted a real order via Create Order (`POST /api/v2/orders`, `Call-Type: CreateOrder`) with PO number `LH-FULLTEST-PO-1`, `warehouseID: 1`. This succeeded and returned:
   ```json
   {"content":{"status":200,"success":[{"id":"1645981","po":"LH-FULLTEST-PO-1"}]}}
   ```
   (Please note: this order has `"DO NOT SHIP THIS ORDER - LH INTEGRATION TEST"` in the warehouse instructions field — it is a test order only. Please let us know if it needs to be cancelled or cleaned up on your end.)

2. Seconds later, we called Get Tracking by PO for that same order:
   ```
   Request body:
   {
     "orders": [{"po": "LH-FULLTEST-PO-1"}],
     "warehouseID": 1
   }
   ```

3. Response was `HTTP 200`, but with `"success": false` and a raw SQL error in the body:
   ```json
   {
     "success": false,
     "details": {
       "Message": "You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near ')  AND TRIM(wo.poNumber) IN ('LH-FULLTEST-PO-1')' at line 16",
       "queryError": "SELECT wo.id, ... FROM wh_orders wo LEFT JOIN ioms_orders io ... LEFT JOIN carrier_services cs ... LEFT JOIN accounts a ... WHERE wo.accountID IN ()  AND TRIM(wo.poNumber) IN ('LH-FULLTEST-PO-1')",
       "Sql": "..."
     }
   }
   ```

**Root cause visible directly in your own error message:** the query contains `WHERE wo.accountID IN ()` — an **empty** `IN ()` clause. This is invalid SQL on its own, independent of the PO lookup that follows it. Something in resolving our API key/warehouse to an account ID is returning no value, and the query is being built without a guard against that empty case.

**This is not a "too soon after order creation, not found yet" situation** — we would expect a clean "not found" response in that case, not a database error. We also want to flag, separately from the functional bug: **your error response includes the full raw SQL query text**, exposing internal table and column names (`wh_orders`, `ioms_orders`, `carrier_services`, `accounts`, and their joins) to any API caller. We'd recommend not returning raw query text in production error responses, independent of fixing the underlying bug.

**Ask:**
1. Please reproduce using PO `LH-FULLTEST-PO-1` / order ID `1645981` / `warehouseID: 1`.
2. Is there an account-level identifier (separate from `warehouseID`) that we should be sending with our requests, which might explain why `accountID` resolved to an empty set on our account?
3. Please confirm whether raw SQL error text should be suppressed from API responses going forward.

---

## Open questions — not bugs, just need your confirmation

**Q1 — `priorityOrder` type.** You told us verbally this field expects an integer. However, your own Postman collection's saved Create Order example sends it as the string `"false"`. We tested sending an integer (`0`) and it was accepted without error — but we'd like to know definitively: is integer the correct/required type, or does the endpoint also accept a string? Does `"false"` in your example actually work, or was that example simply never validated?

**Q2 — `shipservice` casing.** Your Postman collection's Create Order example uses `"UPS_GROUND"` (uppercase with underscore). The Service Codes by Carrier spreadsheet you separately provided lists the same service as `ups_ground` (lowercase). We tested uppercase and it was accepted — but please confirm which casing is authoritative, since your two sources disagree with each other.

**Q3 — SKU field casing on Inventory.** Your Fern documentation text uses lowercase `"sku"` for the Inventory request body's per-item key. Your own Postman collection's real saved example uses uppercase `"SKU"`. Separately, we noticed your "Get all Items" endpoint (`getItems`) returns lowercase `"sku"` in its response for the same items that "Add New Item" and "Get Item by SKU" return with uppercase `"SKU"`. Please confirm which casing is correct/required for the Inventory endpoint specifically (we cannot fully test this yet since Bug 1 above blocks all Inventory calls), and clarify if the casing inconsistency across your item endpoints is intentional.

**Q4 — Create Load v1 vs v2.** We found two different specs for Create Load in your Postman collection: a v1 using query-string parameters, and a v2 using a JSON request body (a completely different, better-structured shape). Is v1 deprecated in favor of v2? Should we build exclusively against v2 going forward?

---

Happy to hop on a call if it's faster to walk through any of the above live. Let us know once Bug 1 (Inventory) and Bug 2 (Tracking) are addressed so we can resume end-to-end testing.

Thanks,
[Your name]
