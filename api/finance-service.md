# finance-service

JSON is snake_case; query parameters use their Java names (camelCase). The envelope is `{ success, code, message, data,
timestamp }`. Lists use finance's paging: `page` (from 0), `size` (default 20, clamped to 1–200), `data` = `{ items, page,
size, total_elements, total_pages }`.

| Status | Method | Path | Who | Spec |
|---|---|---|---|---|
| 📝 | GET | /api/v1/bank-statements/accounts (existing; entry written from the code) | Owner, Manager | [mobile-sales-management](../specs/mobile-sales-management/spec.md) MSL-R22 |
| 🚧 | POST | /api/v1/financial-records/receipts | Owner, Manager | [receipts](../specs/receipts/spec.md) RCT-R01–R04, R06–R08, R41–R43, R57–R63 |
| 🚧 | GET | /api/v1/financial-records/receipts | Owner, Manager | RCT-R30–R32, R50, R63, E37, E38 |
| 🚧 | GET | /api/v1/financial-records/receipts/summary | Owner, Manager | RCT-R33, R63, E31, E33 |
| 🚧 | GET | /api/v1/financial-records/receipts/{id} | Owner, Manager | RCT-R40, R46, R55, R63, E36 |
| 🚧 | GET | /api/v1/financial-records/receipts/{id}/document | Owner, Manager | RCT-R35, R63, E36 |
| 🚧 | PUT | /api/v1/financial-records/receipts/{id} | Owner, Manager | RCT-R09–R14, R56, R59, R60, R62, R63, E39 |
| 🚧 | POST | /api/v1/financial-records/receipts/{id}/post | Owner, Manager | RCT-R16–R22, R44, R45 |
| 🚧 | POST | /api/v1/financial-records/receipts/{id}/not-duplicate | Owner, Manager | RCT-R25, R26 |
| 🚧 | DELETE | /api/v1/financial-records/receipts/{id} | Owner, Manager | RCT-R36, R37 |
| 🚧 | POST | /api/v1/financial-records/receipts/{id}/reverse | Owner, Manager | RCT-R38, R45, R51, R54 |
| ✅ | POST | /api/v1/bank-statements/transactions/{id}/reverse (existing, changed) | Owner, Manager | RCT-R47 |
| ✅ | PATCH | /api/v1/bank-statements/transactions/{id} (existing, changed) | Owner, Manager | RCT-R48, R53 |
| ✅ | POST | /api/v1/bank-statements/transactions/categorize (existing, changed) | Owner, Manager | RCT-R52, R53 |
| ✅ | GET | /api/v1/bank-statements/transactions/{id} (existing, unchanged; the receipt duplicate panel opens a matching transaction with it) | Owner, Manager | RCT-R24 |

## Receipts

**Who, on every receipt endpoint.** An owner sees all branches (or one, with `branchId` where a filter exists); a manager
only their own branch. A SERVICE token gets `403 FORBIDDEN_ROLE` "You don't have permission to do this."; no token gets
`401`. A receipt of another business, or for a manager one of another branch or with no branch, or one a concurrent discard
removed, answers `404 NOT_FOUND` "Receipt not found." (RCT-M08), and so does an id that isn't a UUID (RCT-E36). On PUT
and reverse a body that fails validation gets `400` before the receipt is looked up, whether or not it exists (RCT-D54). A
malformed query value (e.g. `status=BOGUS`, `page=x`) or a body field of the wrong type gets the `VALIDATION_FAILED` list
below with `problem` `INVALID`, on every receipt endpoint (RCT-R63).

**The receipt object** (`data` of create, PUT, post, not-duplicate and reverse; list rows omit `lines`): the
[`receipts`](../data-dictionary/finance.md#receipts) fields — `receipt_id`, `receipt_code`, `business_id`, `branch_id`,
`receipt_number`, `vendor_name`, `receipt_date`, `business_category_id`, `financial_account_id`, `subtotal`, `tax_rate`,
`tax_amount`, `original_amount`, `original_currency`, `exchange_rate`, `usd_equivalent`, `status`,
`duplicate_of_transaction_id`, `duplicate_of_receipt_id`, `transaction_id`, `reversal_reason`, `document_id`,
`created_by_user_id`, `created_at`, `updated_at` — plus `branch_name`, `category_name`, `account_name` and `lines`
`[{ receipt_line_id, description, quantity, unit_price, amount, line_order }]`.

- Money is JSON numbers; `exchange_rate` and `usd_equivalent` are null until posted.
- `branch_name` is returned to both roles (the web shows it to an owner only).
- For a manager, `duplicate_of_transaction_id` and `duplicate_of_receipt_id` are null when the match is outside their branch
  or has no branch (RCT-R46).

**The detail object** (`GET /{id}`) is the receipt object plus:

- `document`: `{ document_id, file_name, content_type, size_bytes }`, or null.
- `extraction`: the stored AI reading, or null (always null for a PDF, RCT-E35).
- `transaction_reference`: for `POSTED` and `REVERSED`, the posted transaction's `reference_number`, or its id when that is
  empty; otherwise null (RCT-R55).
- `reconciliation_state`: `NOTHING_TO_RECONCILE` | `WAITING` | `MATCHED` when `POSTED`; null otherwise (RCT-R40).
- `duplicate_match_scope`: `SAME_SCOPE`, `OTHER_BRANCH` (a manager's match outside their branch; no id or details), or null.
- `history`: `[{ change_log_id, action, previous_value, new_value, change_at, user_id }]`, newest first. `action` is
  `CREATED`, `UPDATED`, `NOT_DUPLICATE`, `POSTED` or `REVERSED`. `UPDATED` rows hold the changed fields, and `status` /
  `duplicate_of_*` when they changed; for a manager those ids are null unless the record is in their branch.

**Validation errors** (RCT-M03): `400 VALIDATION_FAILED`, `message` = the first problem's text, `data` =
`[{ field, problem, message }]` with `field` the snake_case path (e.g. `lines[3].quantity`). Lengths count characters.

| `problem` | Text |
|---|---|
| `REQUIRED` | "{field} is required" (for the reverse `reason`: "Enter a reason for reversing.") |
| `TOO_LONG` | "{field} is at most {max} characters" (for the reverse `reason`: "The reason is at most 500 characters.") |
| `NEGATIVE` | "{field} is 0 or more" |
| `OUT_OF_RANGE` | "{field} is from 0 to 100" |
| `TOO_MANY_DIGITS` | "{field} has at most {int} digits and {dec} decimals" |
| `KHR_DECIMALS` | "original_amount has no decimals for KHR" |
| `INVALID` | "{field} isn't a valid {date / number / currency / id / status / text / list / object}"; also a non-object body: field `body`, "body isn't a valid JSON object" |
| `NOT_FOUND` | "{field} wasn't found" |
| `TOO_MANY` | "lines has at most 100 items" |
| `NOT_AN_OBJECT` | "extraction must be a JSON object" |
| `TOO_LARGE` | "extraction is at most 256 KB" |
| `TOO_EARLY` | "{field} is 2000-01-01 or later" (`receipt_date`, RCT-R62) |

**The `receipt` JSON** (create and PUT). Every field is optional; on PUT a missing field becomes null. A total or line
amount of 0 or less is accepted when saving and refused when posting.

| Field | Rule at save |
|---|---|
| `receipt_number` | ≤ 50 characters; stored trimmed, only spaces → null (RCT-R60) |
| `vendor_name` | ≤ 100 characters; stored trimmed, only spaces → null (RCT-R60) |
| `receipt_date` | ISO date, 2000-01-01 or later (`TOO_EARLY`); a future date is allowed until posting |
| `business_category_id` | a category of this business; else `NOT_FOUND` |
| `financial_account_id` | an account of this business; else `NOT_FOUND`. A manager may use only their branch's account or a business-wide one, else `403 BRANCH_ACCESS_DENIED` "You can only use accounts of your branch." (RCT-M11) |
| `branch_id` | Owner: a branch of the business, else `403 BRANCH_ACCESS_DENIED` "You don't have access to this branch" (RCT-M22), skipped when it is the receipt's current branch. Manager: ignored. An account with a branch always sets the branch. A new receipt, or a change of branch, onto a turned-off branch is refused with `422 BRANCH_INACTIVE` (RCT-R59) |
| `subtotal`, `tax_amount` | ≥ 0, 13 digits and 2 decimals |
| `tax_rate` | 0–100, 2 decimals |
| `original_amount` | 13 digits and 2 decimals; no decimals for KHR |
| `original_currency` | `USD` \| `KHR` |
| `lines` | ≤ 100 of `{ description ≤ 200, quantity ≥ 0, unit_price ≥ 0, amount }`; `line_order` is the position sent, from 1 |
| `extraction` | create only: a JSON object ≤ 256 KB, or null. Stored only with an image file (checked, then dropped for a PDF or no file); PUT ignores it |

### POST /api/v1/financial-records/receipts

Multipart: `file` (optional; JPG, PNG, WebP or PDF, 1 byte to 10 MB) and `receipt` (the receipt JSON; missing means `{}`).
The file's type is read from its bytes, whatever type or name it is sent with; that type is stored as `content_type` and
names the stored file (RCT-R61, RCT-E40). A PDF is saved with no reading (RCT-E35).
With a file, a `documents` row holds the file and the `extraction`; without one, `document_id` is null and the `extraction`
is dropped after it is checked. The duplicate check runs (a reversal transaction is never a match). All checks run before
the file is stored. `201 CREATED` "Receipt saved", `data`: the receipt, `NEEDS_REVIEW` or `POSSIBLE_DUPLICATE`.

Errors, in this order:

| Status | `code` | When |
|---|---|---|
| 415 | `UNSUPPORTED_FILE_TYPE` | the file's bytes aren't a JPG, PNG, WebP or PDF, or it's empty (RCT-M01) |
| 413 | `FILE_TOO_LARGE` | "The file is larger than 10 MB." (RCT-M02), also when the multipart limit trips |
| 400 | `VALIDATION_FAILED` | RCT-M03 |
| 403 | `BRANCH_ACCESS_DENIED` | a manager names another branch's account (RCT-M11) |
| 403 | `BRANCH_ACCESS_DENIED` | an owner names a branch outside the business (RCT-M22) |
| 422 | `BRANCH_INACTIVE` | the receipt's branch (sent, from the account, or the manager's own) is turned off: owner "'{branch}' is turned off. Pick another branch or account." (RCT-M31); manager "Your branch is turned off. Ask the owner to turn it on." (RCT-M32) |
| 503 | `DEPENDENCY_UNAVAILABLE` | "We couldn't save the file. Try again." (RCT-M19) |

### GET /api/v1/financial-records/receipts

Query: `status`, `businessCategoryId`, `financialAccountId`, `dateFrom`, `dateTo` (inclusive, on `receipt_date`; undated
receipts drop out when either is set), `branchId`, `search`, `page`, `size` (`sort` is ignored). `search` matches
`vendor_name` or `receipt_number` (contains, case-insensitive), and when it is a plain number (`12.5`) also an
`original_amount` equal to it. Order: `receipt_date` desc (no date last), `created_at` desc, `receipt_id`. An owner without
`branchId` sees the whole business, including receipts with no branch; an owner may filter by a turned-off branch
(RCT-R31). A page past the last one returns no items with the real `total_elements` / `total_pages`; `page` below 0 counts
as 0 (RCT-E37). `200 OK`, `data`: a page of receipt rows without `lines`.

| Status | `code` | When |
|---|---|---|
| 400 | `VALIDATION_FAILED` | a malformed query value, `problem` `INVALID` (RCT-R63) |
| 403 | `BRANCH_ACCESS_DENIED` | a manager asks for another branch, or an owner a branch outside the business (RCT-M22) |

### GET /api/v1/financial-records/receipts/summary

Query: `branchId`, scoped as the list (an owner may pick a turned-off branch). `200 OK`, `data`:

- `posted_this_month: { count, usd_total }`: `POSTED`, `receipt_date` in the current Asia/Phnom_Penh month; `usd_total` is
  the sum of the stored `usd_equivalent`.
- `possible_duplicate: { count, usd_total }`: USD receipts add their total as it is; non-USD ones are converted at today's
  rate. The rate is read only when such a receipt is in scope. If it can't be read, `usd_total` is **null** and `count` is
  still given — there is no 503.
- `needs_review: { count }`.
- `waiting_to_reconcile: { count }`: posted from a bank or wallet account and not yet matched to a statement line.

All zeros when nothing is in scope. `403 BRANCH_ACCESS_DENIED` (RCT-M22).

### GET /api/v1/financial-records/receipts/{id}

`200 OK`, `data`: the detail object. `404` (RCT-M08).

### GET /api/v1/financial-records/receipts/{id}/document

`200 OK`, `data`: `{ url, expires_at }`, a pre-signed link valid 15 minutes. `404` (RCT-M08) when the receipt is out of
scope or has no file.

### PUT /api/v1/financial-records/receipts/{id}

Body: the receipt JSON, replacing the fields and lines (`extraction` ignored). The duplicate check runs again, except while
the receipt is `NEEDS_REVIEW` and its vendor, date, total and currency equal those saved by "Not a duplicate". A save that
changes nothing (fields, lines, status, match) writes nothing: no history row and `updated_at` unchanged. Otherwise one
`UPDATED` row is written. `200 OK` "Receipt saved", `data`: the receipt.

Errors, in this order:

| Status | `code` | When |
|---|---|---|
| 400 | `VALIDATION_FAILED` | the body, or a category or account not in the business (RCT-M03); before the lookup |
| 404 | `NOT_FOUND` | RCT-M08 |
| 403 | `BRANCH_ACCESS_DENIED` | a manager names another branch's account (RCT-M11) |
| 403 | `BRANCH_ACCESS_DENIED` | an owner names a branch outside the business, unless it's the receipt's current branch (RCT-M22) |
| 409 | `RECEIPT_NOT_EDITABLE` | "This receipt is posted and can't be changed." / "… reversed …" (RCT-M09) |
| 422 | `BRANCH_INACTIVE` | the branch changes to a turned-off one (RCT-M31 / RCT-M32); a receipt kept on a turned-off branch saves (RCT-E39) |

### POST /api/v1/financial-records/receipts/{id}/post

No body. `200 OK` "{receipt_code} is posted as {transaction_reference}." (RCT-M16), `data`: the receipt, `POSTED`, with
`transaction_id`, `exchange_rate` and `usd_equivalent`. A refusal changes nothing. A receipt on a turned-off branch can still
be posted, discarded and reversed (RCT-E17).

Errors, in this order:

| Status | `code` | When |
|---|---|---|
| 404 | `NOT_FOUND` | RCT-M08 |
| 409 | `RECEIPT_NOT_EDITABLE` | already posted or reversed (RCT-M09) |
| 409 | `RECEIPT_POSSIBLE_DUPLICATE` | not resolved yet (RCT-M10) |
| 422 | `RECEIPT_INCOMPLETE` | "Fix {n} fields to save this receipt" ("Fix 1 field …" when n is 1) (RCT-M04); `data` = `[{ field, problem }]`: `REQUIRED` for `vendor_name`, `receipt_date`, `original_currency`, `business_category_id`, `financial_account_id`, `branch_id`; `NOT_POSITIVE` for `original_amount` |
| 422 | `RECEIPT_DATE_IN_FUTURE` | RCT-M05 |
| 403 | `BRANCH_ACCESS_DENIED` | a manager and another branch's account (RCT-M11) |
| 422 | `ACCOUNT_INACTIVE` | RCT-M06 |
| 422 | `ACCOUNT_CURRENCY_MISMATCH` | RCT-M24 |
| 422 | `CATEGORY_INACTIVE` | RCT-M07 |
| 422 | `CATEGORY_TYPE_MISMATCH` | RCT-M25 |
| 409 | `PERIOD_LOCKED` | the receipt date is in a signed-off period (RCT-M23) |
| 503 | `DEPENDENCY_UNAVAILABLE` | today's rate can't be read |

### POST /api/v1/financial-records/receipts/{id}/not-duplicate

No body. Records `NOT_DUPLICATE` with the vendor, date, total and currency. `200 OK` "Marked as not a duplicate"
(RCT-M29), `data`: the receipt, `NEEDS_REVIEW`, both match ids null. `404` (RCT-M08); `409 RECEIPT_NOT_DUPLICATE` when it
isn't `POSSIBLE_DUPLICATE` (RCT-M12).

### DELETE /api/v1/financial-records/receipts/{id}

Deletes the receipt, its lines, its `documents` row and the file. `200 OK`, `data`: null, message "{receipt_code} from
{vendor_name} was removed. Nothing was counted." or, with no vendor, "{receipt_code} was removed. Nothing was counted."
(RCT-M17). `404` (RCT-M08); `409 RECEIPT_NOT_EDITABLE` when posted or reversed (RCT-M09).

### POST /api/v1/financial-records/receipts/{id}/reverse

Body: `{ "reason": string }`, trimmed, then 1–500 characters. Posts the opposite transaction dated today (Asia/Phnom_Penh);
a turned-off account or category doesn't block it. `200 OK` "{receipt_code} is now Reversed. {amount} went back to
{account_name}." with `{amount}` like "1,250.50 USD" or "41,000 KHR" (RCT-M18), `data`: the receipt, `REVERSED`.

Errors, in this order:

| Status | `code` | When |
|---|---|---|
| 400 | `VALIDATION_FAILED` | reason empty, `problem` `REQUIRED` (RCT-M13), or too long, `TOO_LONG` (RCT-M21); `reason` not text, or the body not a JSON object, `INVALID` (RCT-R63); before the lookup |
| 404 | `NOT_FOUND` | RCT-M08 |
| 409 | `RECEIPT_NOT_REVERSIBLE` | "Only a posted receipt can be reversed." — any status but `POSTED` (RCT-M30) |
| 409 | `TRANSACTION_ALREADY_MATCHED` | matched to a statement line (RCT-M14) |
| 409 | `PERIOD_LOCKED` | today is in a signed-off period (RCT-M23) |
| 409 | `TRANSACTION_ALREADY_REVERSED` | only when two reversals race (RCT-M26) |

## Transactions (existing endpoints, changed by receipts)

### POST …/transactions/{id}/reverse · PATCH …/transactions/{id} · POST …/transactions/categorize

Who, body and response are unchanged. A transaction a receipt points to (`receipts.transaction_id`) can't be reversed,
PATCHed or categorized here, and neither can that transaction's reversal row by PATCH or categorize. Reverse checks it after
the in-scope find; PATCH before any change; categorize resolves every id first and refuses the whole batch.

| Status | `code` | When |
|---|---|---|
| 409 | `TRANSACTION_NOT_REVERSIBLE` | "Change this from its receipt" (RCT-M28, RCT-R47, RCT-R48, RCT-R52, RCT-R53) |

## Accounts

Used by [mobile-sales-management](../specs/mobile-sales-management/spec.md). The endpoint exists in code on `dev`; its shape is taken from the code (finance-service `AccountController`, `AccountResponse`). 📝 here means "exists in code, entry written from the code".

### GET /api/v1/bank-statements/accounts

Query: `branchId` (a manager is always forced to their own branch), `status` (`ACTIVE` …).

`200 OK`, `data`: a list (not paged), ordered by `account_name`, of the branch's accounts **and** the business-wide ones: `{ financial_account_id, account_name, account_type, currency, starting_balance, current_balance, usd_equivalent, is_business_wide, is_active, branch_id, branch_name, created_at, updated_at }`. A manager gets business-wide accounts without their balances (`null`).

Shared errors only (`401`, `403`).

The income transaction for a sale is written by sales-service calling finance-service internally (`FeignFinanceGateway.recordSaleIncome`); clients never write it.
