# Plan: Receipts

**Spec:** `RCT` v4.0.0, commit `e92089f` · **Approved:** 2026-11-16 by Vuththana
**Repos:** ledgervision-finance-service (backend: owns `receipts`, `receipt_lines`, `documents`, all 10 receipt endpoints; changes 3 existing transaction endpoints), ledgervision-web-ui (web front end, Next.js BFF)

## 1. Scope

- **In:**
  - Spec §4a–§4g.
  - RCT-R01–R26, RCT-R30–R53.
  - RCT-E01–E19 and RCT-E21–E30 (RCT-E20 is withdrawn).
  - Messages RCT-M01–M28 (M20 unused).
  - Dictionary §8: `receipts`, `receipt_lines`, `documents`, `documents.extraction`, `change_logs.record_type` `RECEIPT`.
  - The change to the existing finance transaction endpoints: reverse, PATCH and categorize refuse a receipt's transaction and its reversal row (RCT-R47, R48, R52, R53).
- **Out:**
  - Android (spec §1).
  - CSV export (RCT-D21). The web's Export button on receipts is removed.
  - `POST /ai/receipts/extract` itself.
  - Bank reconciliation of the posted expense. Only its state is shown (RCT-R40).
  - The sidebar "receipts" badge and the legacy `transaction-panel.tsx`. Both read the in-memory `FinanceRows`, which is always empty now, and stay as they are.
  - No change to `LedgerService`, `AccountLookup`, `CategoryLookup`, the attachments endpoint, the transfers endpoints or the internal sale-refund endpoint.

**Base branches (§10 Q1, answered):** `feature/receipts` starts from `origin/dev-cleaned` in both repos, and the PRs go into `dev-cleaned`.

## 2. Repos and parts

| Repo | Role (agent) | Part | Order |
|---|---|---|---|
| finance-service | api-dev | A. Migration `V5__receipts.sql`, entities, `DocumentSequenceService`, new `ErrorCode`s (RCT-R01, R03, R07, R08, R24, §5) | **first**; everything in finance builds on it |
| finance-service | api-dev | B. Create, request parsing and M03 validation, file storage, duplicate check with the RCT-R49 pick order (RCT-R01–R08, R23, R24, R41–R43, R49) | after A |
| finance-service | api-dev | C. List with the RCT-R50 number search, summary, detail, document link (RCT-R30–R35, R40, R46, R50) | after A; parallel with B |
| finance-service | api-dev | D. PUT, post, not-duplicate, discard, reverse dated today (RCT-R09–R22, R25, R26, R36–R39, R44, R45, R51) | after B (shares the parser and duplicate check) |
| finance-service | api-dev | E. Receipt lock in the existing transactions reverse, PATCH and categorize (RCT-R47, R48, R52, R53) | after A (needs `receipts.transaction_id`); parallel with B–D |
| web-ui | web-dev | F. Types, data layer, normalizers, server actions, reading mapper; contract and logic tests on §3 fixtures | **parallel with A–E from day one**; needs only §3 |
| web-ui | web-dev | G. Screens: list, upload, new, detail (review / duplicate / posted / reversed), dialogs; route pages; `next.config.ts` body limit | after F; parallel with B–E |
| web-ui | web-dev | H. Live e2e against a local finance-service through the gateway; check the banking screens show RCT-M28 | after B–E run locally, and after G |
| docs (lead) | lead | `api/finance-service.md` brought to §3: error `data` shapes, problem codes, history actions; status 📝 → 🚧 → ✅; `specs/README.md` stage | after B–E merge |

**About the web mock.** web-ui has no receipts mock to build on:

- Its README forbids seeded or sample records as a fallback.
- The `FinanceRows` receipt functions in `financial-service.ts` are 503 stubs, which this plan removes.

So the web builds against §3 in two steps. Normalizers and the reading mapper are tested with JSON fixtures copied from §3 (`pnpm test:logic`, no backend). Screen states are checked against a local finance-service once B–E run (part H).

## 3. API contract

All receipt endpoints are finance-service, under `/api/v1/financial-records/receipts`. They and the three changed transaction endpoints are listed in `api/finance-service.md` as 📝 planned. The details below go into that file when the lead updates it.

- JSON is snake_case; query parameters are camelCase.
- The envelope is `{ success, code, message, data, timestamp }`.

**Who, every receipt endpoint:**

- Owner: all branches, or one with `branchId` where a filter exists.
- Manager: own branch only.
- A SERVICE token: `403 FORBIDDEN_ROLE` "You don't have permission to do this.".
- `404 NOT_FOUND` "Receipt not found." (RCT-M08) for:
  - a receipt of another business;
  - for a manager, a receipt of another branch or one with no branch;
  - a receipt that a concurrent discard has removed (RCT-E02–E04, E29).

**The receipt object** (`data` of create, PUT, post, not-duplicate and reverse; list rows omit `lines`):

- `receipt_id`, `receipt_code`, `business_id`, `branch_id`, `branch_name`
- `receipt_number`, `vendor_name`, `receipt_date`
- `business_category_id`, `category_name`, `financial_account_id`, `account_name`
- `subtotal`, `tax_rate`, `tax_amount`, `original_amount`, `original_currency`, `exchange_rate`, `usd_equivalent`
- `status`, `duplicate_of_transaction_id`, `duplicate_of_receipt_id`, `transaction_id`, `reversal_reason`, `document_id`
- `created_by_user_id`, `created_at`, `updated_at`
- `lines`: `[{ receipt_line_id, description, quantity, unit_price, amount, line_order }]`

Rules for the object:

- Money is JSON numbers. `exchange_rate` and `usd_equivalent` are null until posted (RCT-D27).
- **RCT-R46 redaction:** for a manager, when the stored match is a transaction or receipt outside their branch (or a receipt with no branch), `duplicate_of_transaction_id` and `duplicate_of_receipt_id` are null in every response.

**The detail object** (GET `{id}`) is the receipt object plus:

- `document`: `{ document_id, file_name, content_type, size_bytes }`, or null.
- `extraction`: the stored object, or null.
- `transaction_reference`: the transaction's `reference_number`, or its id as text when that is empty. Null unless `POSTED`.
- `reconciliation_state`: `NOTHING_TO_RECONCILE` | `WAITING` | `MATCHED`. Null unless `POSTED` (RCT-R40).
- `duplicate_match_scope`:
  - `SAME_SCOPE` when the receipt is `POSSIBLE_DUPLICATE`, a match id is stored, and the caller is an owner or the match is in the manager's branch;
  - `OTHER_BRANCH` when it is `POSSIBLE_DUPLICATE` with a stored match outside the manager's branch or with no branch;
  - null otherwise, including a match that was discarded (RCT-E14).
- `history`: `[{ change_log_id, action, previous_value, new_value, change_at, user_id }]`, newest first.
  - `action` is one of `CREATED`, `UPDATED`, `NOT_DUPLICATE`, `POSTED`, `REVERSED`.
  - What each row holds: §5.

**Validation error shape (RCT-M03):**

- `400 VALIDATION_FAILED`, with `message` = the first problem's text.
- `data` = `[{ field, problem, message }]`, one entry per field. `field` is the request's snake_case path, e.g. `lines[3].quantity`.
- Lengths count characters (code points).

| `problem` | English text |
|---|---|
| `TOO_LONG` | "{field} is at most {max} characters" |
| `NEGATIVE` | "{field} is 0 or more" |
| `OUT_OF_RANGE` | "{field} is from 0 to 100" |
| `TOO_MANY_DIGITS` | "{field} has at most {int} digits and {dec} decimals" |
| `KHR_DECIMALS` | "original_amount has no decimals for KHR" |
| `INVALID` | "{field} isn't a valid {date / number / currency / id}" |
| `NOT_FOUND` | "{field} wasn't found" |
| `TOO_MANY` | "lines has at most 100 items" |
| `NOT_AN_OBJECT` | "extraction must be a JSON object" |
| `TOO_LARGE` | "extraction is at most 256 KB" |

**The `receipt` JSON** (create and PUT):

- Every field is optional. On PUT, a missing field means null (replace).
- These are save-time rules; posting rules are in `/post`.
- Numbers are JSON numbers. Unknown properties are ignored.

| Field | Rule at save |
|---|---|
| `receipt_number` | ≤ 50 |
| `vendor_name` | ≤ 100 |
| `receipt_date` | ISO date; a future date is allowed until posting |
| `business_category_id` | a category of this business, any type or state; else `NOT_FOUND` |
| `financial_account_id` | an account of this business; else `NOT_FOUND`. A manager may use only their branch's account or a business-wide one; else `403 BRANCH_ACCESS_DENIED` "You can only use accounts of your branch." (RCT-M11) |
| `branch_id` | Owner: a branch of the business, else `403 BRANCH_ACCESS_DENIED` "You don't have access to this branch" (RCT-M22); the check is skipped when it equals the receipt's current branch (RCT-E17). Manager: ignored, their own branch is used (RCT-R04, R13) |
| `subtotal` | ≥ 0, (13,2) |
| `tax_rate` | 0–100, (3,2) |
| `tax_amount` | ≥ 0, (13,2) |
| `original_amount` | (13,2); no decimals for KHR (RCT-E08) |
| `original_currency` | `USD` \| `KHR` |
| `lines` | ≤ 100 items of `{ description ≤ 200, quantity ≥ 0 (12,3), unit_price ≥ 0 (13,2), amount (13,2) }`; `line_order` = position sent, from 1 |
| `extraction` | **Create only.** A JSON object of ≤ 262 144 bytes (UTF-8 of its compact serialization), or null. PUT ignores it |

**Branch on save** (RCT-R04, R12, R13):

- An account with a branch: that branch.
- Otherwise, a manager: their own branch.
- Otherwise, an owner: the sent `branch_id` (null allowed).

### `POST /api/v1/financial-records/receipts` — RCT-R01–R08, R23, R24, R41–R43, R49

- **Who:** Owner, Manager.
- **Body:** `multipart/form-data`:
  - `file`: optional. Content type `image/jpeg`, `image/png`, `image/webp` or `application/pdf`; 1 to 10 485 760 bytes. A file name over 255 characters gets M03 on `file_name`.
  - `receipt`: optional part holding the receipt JSON text. Missing means `{}`.
- **201 CREATED**, "Receipt saved" (RCT-M15), `data`: the receipt, `NEEDS_REVIEW` or `POSSIBLE_DUPLICATE`.
- **Errors:**
  - `415 UNSUPPORTED_FILE_TYPE` (RCT-M01), also for an empty file
  - `413 FILE_TOO_LARGE` (RCT-M02), also when the multipart limit trips
  - `400 VALIDATION_FAILED` (RCT-M03)
  - `403 BRANCH_ACCESS_DENIED` (RCT-M11 / RCT-M22)
  - `503 DEPENDENCY_UNAVAILABLE` "We couldn't save the file. Try again." (RCT-M19)
- All checks run before the file is stored.

### `GET /api/v1/financial-records/receipts` — RCT-R30, R31, R32, R46, R50

- **Who:** Owner, Manager.
- **Query:**
  - `status`
  - `businessCategoryId`, `financialAccountId`
  - `dateFrom`, `dateTo`: inclusive, on `receipt_date`. Receipts with no date drop out when either is set.
  - `branchId`
  - `search`: `vendor_name` or `receipt_number` contains it, case-insensitive. When the trimmed search matches `^[0-9]+(\.[0-9]+)?$`, it also matches `original_amount = <number>` by numeric equality (RCT-R50). Any of these matching counts.
  - `page`: from 0. `size`: default 20, clamped to 1–200. `sort` is ignored.
- **Order:** `receipt_date DESC NULLS LAST, created_at DESC, receipt_id`.
- **Scope:**
  - Owner without `branchId`: the whole business, including null `branch_id`.
  - Owner with `branchId`: that branch.
  - Manager: their own branch.
- **200 OK** `data`: `{ items, page, size, total_elements, total_pages }`.
- **Errors:** `403 BRANCH_ACCESS_DENIED` (RCT-M22); `400 VALIDATION_FAILED` for a malformed query value.

### `GET /api/v1/financial-records/receipts/summary` — RCT-R33, RCT-E09, RCT-E16

- **Who:** Owner, Manager. **Query:** `branchId`, scoped as in the list.
- **200 OK** `data`:
  - `posted_this_month: { count, usd_total }`: `POSTED` with `receipt_date` in the current Asia/Phnom_Penh month; sum of `usd_equivalent`.
  - `possible_duplicate: { count, usd_total }`: converted at today's rate; a null total counts 0.
  - `needs_review: { count }`.
  - `waiting_to_reconcile: { count }`: `POSTED`, a `BANK_ACCOUNT` or `MOBILE_WALLET` account, and no `bank_transactions.matched_transaction_id` = `transaction_id`.
  - All zeros when nothing is in scope. `usd_total` is rounded to 2 decimals.
- **Errors:** `403` (RCT-M22); `503 DEPENDENCY_UNAVAILABLE` when the rate can't be read.

### `GET /api/v1/financial-records/receipts/{id}` — RCT-R34, R40, R46, R24

- **Who:** Owner, Manager. **200 OK** `data`: the detail object.
- **Errors:** `404` (RCT-M08).

### `GET /api/v1/financial-records/receipts/{id}/document` — RCT-R35, RCT-E19

- **Who:** Owner, Manager.
- **200 OK** `data`: `{ url, expires_at }`, a pre-signed GET valid 15 minutes. The key is always read from the receipt's `documents` row.
- **Errors:** `404` (RCT-M08) when the receipt is out of scope or has no file.

### `PUT /api/v1/financial-records/receipts/{id}` — RCT-R09–R15, R23, R24, R49

- **Who:** Owner, Manager.
- **Body:** `application/json`, the receipt JSON. It replaces the fields and lines; `extraction` is ignored.
- **200 OK**, "Receipt saved", `data`: the receipt after the duplicate check (RCT-R10).
- **Errors:**
  - `400` (RCT-M03)
  - `403` (RCT-M11 / RCT-M22)
  - `404` (RCT-M08)
  - `409 RECEIPT_NOT_EDITABLE` "This receipt is {status} and can't be changed." (RCT-M09)

### `POST /api/v1/financial-records/receipts/{id}/post` — RCT-R16–R22, R44, R45

- **Who:** Owner, Manager. **Body:** none.
- **200 OK**, "{receipt_code} is posted as {transaction_reference}." (RCT-M16), `data`: the receipt, `POSTED`, with `transaction_id`, `exchange_rate` and `usd_equivalent`.
- **Errors, checked in this order:**
  1. `404` (RCT-M08)
  2. `409 RECEIPT_NOT_EDITABLE` (RCT-M09)
  3. `409 RECEIPT_POSSIBLE_DUPLICATE` (RCT-M10)
  4. `422 RECEIPT_INCOMPLETE` "Fix {n} fields to save this receipt" (RCT-M04). `data` = `[{ field, problem }]`:
     - `REQUIRED` for `vendor_name`, `receipt_date`, `original_currency`, `business_category_id`, `financial_account_id`, `branch_id`;
     - `NOT_POSITIVE` for `original_amount`.
  5. `422 RECEIPT_DATE_IN_FUTURE` (RCT-M05)
  6. `403 BRANCH_ACCESS_DENIED` (RCT-M11, manager)
  7. `422 ACCOUNT_INACTIVE` "'{account}' is inactive" (RCT-M06)
  8. `422 ACCOUNT_CURRENCY_MISMATCH` (RCT-M24)
  9. `422 CATEGORY_INACTIVE` (RCT-M07)
  10. `422 CATEGORY_TYPE_MISMATCH` (RCT-M25), also for Uncategorized
  11. `409 PERIOD_LOCKED` (RCT-M23) when `receipt_date` is in a signed-off period
  12. `503 DEPENDENCY_UNAVAILABLE` when the rate can't be read
- A refusal changes nothing (RCT-E05).

### `POST /api/v1/financial-records/receipts/{id}/not-duplicate` — RCT-R25, R26

- **Who:** Owner, Manager. **Body:** none.
- **200 OK** `data`: the receipt, `NEEDS_REVIEW`, with both `duplicate_of_*` null.
- **Errors:** `404` (RCT-M08); `409 RECEIPT_NOT_DUPLICATE` (RCT-M12).

### `DELETE /api/v1/financial-records/receipts/{id}` — RCT-R36, R37

- **Who:** Owner, Manager.
- **200 OK**, `data`: null, with the RCT-M17 message:
  - "{receipt_code} from {vendor_name} was removed. Nothing was counted.";
  - when there is no vendor: "{receipt_code} was removed. Nothing was counted.".
- **Errors:** `404` (RCT-M08); `409` (RCT-M09).

### `POST /api/v1/financial-records/receipts/{id}/reverse` — RCT-R38, R39, R45, R51

- **Who:** Owner, Manager.
- **Body:** `{ "reason": string }`, trimmed, 1–500 characters.
- The opposite transaction is dated today in Asia/Phnom_Penh (`LedgerService.refund`), and the signed-off check is on today.
- **200 OK** (RCT-M18), `data`: the receipt, `REVERSED`.
- **Errors:**
  - `400` (RCT-M13 for an empty reason; RCT-M21 for one over 500)
  - `404` (RCT-M08)
  - `409 RECEIPT_NOT_EDITABLE` (RCT-M09)
  - `409 TRANSACTION_ALREADY_MATCHED` (RCT-M14)
  - `409 PERIOD_LOCKED` (RCT-M23, RCT-E30)
  - `409 TRANSACTION_ALREADY_REVERSED` (RCT-M26)

### `POST /api/v1/bank-statements/transactions/{id}/reverse` — existing, changed · RCT-R47

- Who, body and response are unchanged.
- **New check**, after the in-scope find and before the transfer and source checks: if a `receipts.transaction_id` = `{id}`, refuse with `409 TRANSACTION_NOT_REVERSIBLE` "Change this from its receipt" (RCT-M28).
- A receipt's reversal row is already refused by "A reversal can't be reversed".

### `PATCH /api/v1/bank-statements/transactions/{id}` — existing, changed · RCT-R48, R53

- Who, body and response are unchanged.
- **New check**, before the signed-off check and before any change: refuse with RCT-M28 when either holds:
  - a `receipts.transaction_id` = `{id}`;
  - the transaction's `reverses_transaction_id` is a receipt's `transaction_id`.

### `POST /api/v1/bank-statements/transactions/categorize` — existing, changed · RCT-R52, R53

- Who, body and response are unchanged.
- All ids are resolved first; an id out of scope gives 404, as today.
- Then, before any change: if any id is a receipt's transaction or the reversal of one, refuse the whole batch with RCT-M28. Nothing in the batch changes.

**Existing endpoints the web also uses (unchanged):**

- `GET /api/v1/bank-statements/accounts`
- `GET /api/v1/financial-records/categories`
- `GET /api/v1/bank-statements/transactions/{id}` (RCT-R24)
- `POST /ai/receipts/extract`

## 4. Screens, routes and states

| Route | Who | States | IDs |
|---|---|---|---|
| `/financial-records/receipts` | Owner, Manager | Cards and table load separately, with skeletons and retry; empty states; tabs; filters; rows; paging (detail below) | RCT-R30–R33, R50, RCT-E16 |
| `/financial-records/receipts/upload` | Owner, Manager | Pick, reading, saving, error, success (detail below) | RCT-R01–R04, R41, RCT-E10, E11, E21, E24 |
| `/financial-records/receipts/new` | Owner, Manager | Empty form, save for later, save as expense, field errors (detail below) | RCT-R05, R08, R42, R43 |
| `/financial-records/receipts/[id]` | Owner, Manager | Loading skeleton; 404 → not-found screen (M08, also after a concurrent discard, RCT-E29); load error with retry; one state per status (detail below) | RCT-R09–R26, R34–R40, R44–R46, R49, R51, RCT-E05–E07, E13–E15, E25, E26, E28–E30 |
| `/financial-records/receipts/[id]/review` (legacy) | — | Redirects to `/financial-records/receipts/[id]` | — |
| Banking screens that reverse or categorize | Owner, Manager | On `409 TRANSACTION_NOT_REVERSIBLE`, the server message (RCT-M28) is shown. Changed only where the error is dropped today | RCT-R47, R52, R53 |

**`/financial-records/receipts`:**

- Empty with no receipts at all: "No receipts yet". Empty for a search or filters: "No receipts found" or "No receipts match these filters".
- Tabs All / Needs review / Possible duplicate / Posted, sent as `status`; REVERSED receipts show under All.
- Filters: category, paid from, date range, branch (owner only), search.
- Rows: code, vendor, date, category, account, total with currency, USD ("—" while null), status, branch (owner only).
- 20 a page. M22 is shown as an error. There is no Export button.

**`/financial-records/receipts/upload`:**

- JPG, PNG, WebP or PDF, up to 10 MB; the client check shows M01 / M02.
- Images go to AI extract. A PDF skips it, because the AI readers take images only.
- Server errors M01 / M02 / M03 / M11 / M19 show with a retry.
- On success: toast M15, then `/receipts/{id}`.
- When AI fails, the receipt is saved with no reading and no fields (RCT-E10).

**`/financial-records/receipts/new`:**

- An empty form with no file.
- "Save for later" → POST.
- "Save as expense" → POST, then `/post`. If the post is refused, the page goes to `/receipts/{id}` with the error shown.
- M03 errors show under each field. Leave prompt when the form is dirty.

**`/financial-records/receipts/[id]`, by status:**

- **`NEEDS_REVIEW`:**
  - The editable form, with the document viewer.
  - Category: active EXPENSE categories plus the current one.
  - Paid from: for a manager, own-branch and business-wide accounts.
  - Branch: from the account; an owner picks one for a business-wide account; fixed for a manager.
  - Totals are typed; nothing is computed. Lines ≤ 100.
  - "We couldn't read this receipt" card when there is a file and the reading has no values.
  - Save for later → PUT. Save as expense → PUT, then `/post`.
  - Post errors: M04 (fields marked, banner), M05, and M06 / M07 / M23 / M24 / M25 as the server's message.
  - Discard → confirm dialog → DELETE → toast M17 → back to the list.
- **`POSSIBLE_DUPLICATE`:** the same form plus a compare panel.
  - `SAME_SCOPE`: the match is fetched and shown beside this one. "Open matching …" goes to the receipt's page, or opens a read-only sheet for a transaction.
  - `OTHER_BRANCH`: M27 only, no link or details.
  - No stored match: the M10 text alone.
  - Choices: "Same expense, discard receipt" or "Not a duplicate, continue review". "Save as expense" is disabled, with the M10 text.
- **`POSTED`:**
  - Read-only, with `transaction_reference`, a reconciliation chip and the document link.
  - "Reverse" dialog, saying the reversal is dated today. Reason 1–500 with M13 / M21 inline; M14 / M23 / M26 / M09 as alerts; toast M18 on success.
- **`REVERSED`:** read-only, with the reason.
- **History card** on every status.

Confidence numbers are not shown anywhere: `jsonb-shapes.md` says not to assume a scale.

## 5. Data

| Table.field (dictionary name) | Read / write | Repo | IDs |
|---|---|---|---|
| `receipts` (all fields) | Write. V5 creates it with:<br>• CHECKs: status, currency, `tax_rate` 0–100, `subtotal` / `tax_amount` ≥ 0<br>• FKs to `business_categories`, `financial_accounts`, `transactions` (`duplicate_of_transaction_id`; `transaction_id` unique), `documents`, and `receipts` (`duplicate_of_receipt_id` ON DELETE SET NULL)<br>• Indexes: unique (`business_id`, `receipt_code`); unique `transaction_id`; (`business_id`, `receipt_date`, `original_amount`); (`branch_id`, `status`) | finance | RCT-R01, R03–R09, R12, R13, R17–R19, R23–R25, R36, R38, R49, RCT-E14 |
| `receipts.receipt_code` | Written once at create: `RCP-{yyyy}-{seq:04d}`, with the year of `BusinessTime.today()` | finance | RCT-R01 |
| `document_sequences` (`business_id`, `doc_type`, `seq_year`, `last_value`) | Write. V5 creates it in schema `finance`. `DocumentSequenceService.next(business, "RCP", year)` upserts it | finance | RCT-R01 |
| `receipt_lines` (all fields) | Write. V5 creates it with `receipt_id` ON DELETE CASCADE. A save deletes the lines and re-inserts them with `line_order` 1..n | finance | RCT-R09, R42, R36 |
| `documents` (all fields), `documents.extraction` | Written at create, deleted at discard. `storage_key` = `finance/attachments/{business_id}/receipts/{uuid}.{ext}` | finance | RCT-R03, R41, R35, R36 |
| `transactions` (via `LedgerService.record`) | Written at post: EXPENSE, POSTED; source `AI_RECEIPT` with a file, `MANUAL_ENTRY` without; `transaction_date` = `receipt_date`; amount, account, branch, category (`category_source` USER); `counterparty_name` = vendor; `reference_number` = receipt number; `description` "Receipt {code} · {vendor}"; `attachment` = `storage_key`; `created_by_user_id` = poster. Read by the duplicate check | finance | RCT-R17–R19, R23 |
| `transactions` reversal row (via `LedgerService.refund`) | Written at reverse: the opposite row with `reverses_transaction_id`, dated today | finance | RCT-R38, R45, R51, RCT-E13, E30 |
| `receipts.transaction_id`, `transactions.reverses_transaction_id` | Read by the transactions guard | finance | RCT-R47, R48, R52, R53 |
| `receipts.exchange_rate`, `receipts.usd_equivalent` | Copied from the new transaction at post | finance | RCT-R18 |
| `financial_accounts` (`account_name`, `account_type`, `currency`, `is_business_wide`, `is_active`, `branch_id`, `business_id`) | Read | finance | RCT-R12–R14, R22, R33, R40, R44 |
| `business_categories` (`category_name`, `category_type`, `is_active`) | Read | finance | RCT-R22, R32 |
| `bank_transactions.matched_transaction_id` | Read | finance | RCT-R33, R39, R40 |
| `change_logs` (`record_type` `RECEIPT`) | Write; one row per action, values below. Read for `history` and RCT-R10 | finance | RCT-R07, R10, R15, R19, R25, R34, R36, R38 |
| `businesses.exchange_rate` | Read at post and in the summary | finance | RCT-R18, R33 |
| All receipt fields, via §3 | Read / sent | web-ui | as §4 |

**`change_logs` rows written for receipts:**

- **CREATED:** new = the saved fields as JSON (no extraction).
- **UPDATED:** previous / new = the changed fields only, plus `status` when it changed. Never the duplicate ids.
- **NOT_DUPLICATE:** new = `{vendor_name, receipt_date, original_amount, original_currency}`.
- **POSTED:** new = `transaction_id`.
- **REVERSED:** new = the reason.
- **DISCARDED:** previous = `{receipt_code, vendor_name, original_amount, original_currency}`.

**Duplicate check** (RCT-R23, R24, R49, R10). It runs on create and on each PUT, inside the save transaction.

1. Take `pg_advisory_xact_lock` on the business.
2. Run only when `btrim(vendor_name)`, `receipt_date`, `original_amount` and `original_currency` are all set.
3. One UNION query of the candidates:
   - (a) the business's other receipts with `status IN (NEEDS_REVIEW, POSSIBLE_DUPLICATE)`;
   - (b) `POSTED` `EXPENSE` transactions of the business's accounts with no reversal row, other than this receipt's own, matched on `lower(btrim(counterparty_name))`, date, amount and currency.
   - A posted receipt is seen only through its transaction.
4. Order the candidates: for a manager caller, own branch first; then transactions before receipts; then `created_at` ascending; then id.
5. Keep the first. No candidate sets `NEEDS_REVIEW` and clears both ids.
6. On PUT, skip the check when the latest `NOT_DUPLICATE` snapshot equals the new key fields (RCT-R10).

## 6. Roles and branch scope

**finance, receipt endpoints:**

- `CurrentUserProvider.get()`; a SERVICE caller gets 403 FORBIDDEN_ROLE.
- Every query has `business_id = :business`.
- A manager is limited to `branch_id = user.branchId`, so null-branch receipts are excluded (RCT-R30, E04, E22).
- An owner sees everything, or `BranchScope.requireAccess` on `branchId` (RCT-R31, M22).
- Finding by id: a business mismatch, or for a manager a branch mismatch or null, gives 404 M08 (RCT-E02–E04). A row removed by a concurrent discard also gives 404 M08: `EntityNotFoundException` from `RowLocks.lock` is mapped to it (RCT-E29).
- Accounts: `AccountLookup.find` plus `CurrentUser.mayUseAccount`, else M11 (RCT-R14).
- Categories: `CategoryLookup.find`.
- Duplicate check: business-wide, with a manager's own branch first (RCT-R49). One mapper builds every receipt object and redacts per viewer (RCT-R46).

**finance, transaction endpoints:** the existing scope is unchanged. The receipt guard runs after scope, so a transaction out of scope still gives 404 (RCT-R47, R48, R52, R53).

**web-ui:**

- Both roles reach every receipt route.
- The branch column and branch filter show only for an owner.
- The account picker uses the backend's scoped list.
- The web never filters scope itself.

## 7. Files

| Repo | Create / change | Why (IDs) |
|---|---|---|
| finance | create `src/main/resources/db/migration/V5__receipts.sql` | Creates `documents`, `receipts`, `receipt_lines`, `document_sequences` (RCT-R01, R03, R24, R36) |
| finance | create `common/sql/DocumentSequenceService.java` | RCT-R01 |
| finance | create `document/entity/Document.java` and `document/repository/DocumentRepository.java` | RCT-R03 |
| finance | create `receipt/entity/{Receipt, ReceiptLine, ReceiptStatus, ReconciliationState, DuplicateMatchScope}.java` and `receipt/repository/{ReceiptRepository, ReceiptLineRepository}.java` | RCT-R01, R09, R40, R46, R47 |
| finance | create `receipt/dto/ReceiptDtos.java` | §3 |
| finance | create `receipt/service/ReceiptInputParser.java` | RCT-R05, R41–R43, RCT-E08, E18, E27 |
| finance | create `receipt/service/DuplicateCheck.java` | RCT-R06, R10, R23, R24, R49, RCT-E14, E23, E24 |
| finance | create `receipt/service/ReceiptFiles.java` | RCT-R02, R03, R36, RCT-E11, E12, E21 |
| finance | create `receipt/service/ReceiptQueries.java` | RCT-R30–R34, R40, R46, R50 |
| finance | create `receipt/service/ReceiptService.java` and `service/impl/ReceiptServiceImpl.java`: `RowLocks` for update / post / not-duplicate / discard, a deleted row → M08; reverse via `LedgerService.refund` | RCT-R01–R26, R36–R39, R44, R45, R51, RCT-E01, E13, E15, E29, E30 |
| finance | create `receipt/service/ReceiptTransactions.java`: `isLocked(Transaction)` | RCT-R47, R48, R52, R53 |
| finance | create `receipt/controller/ReceiptController.java`, with a local `MaxUploadSizeExceededException` handler → 413 FILE_TOO_LARGE | §3, RCT-R02 |
| finance | change `transaction/service/impl/TransactionServiceImpl.java`: the receipt guard in `reverse` (before the transfer and source checks), in `updateDetails` (before `requireOpen`) and in `categorize` (all ids checked before any change) | RCT-R47, R48, R52, R53 |
| finance | change `exception/ErrorCode.java`: add `UNSUPPORTED_FILE_TYPE(415)`, `FILE_TOO_LARGE(413)`, `RECEIPT_INCOMPLETE(422)`, `RECEIPT_DATE_IN_FUTURE(422)`, `RECEIPT_NOT_EDITABLE(409)`, `RECEIPT_POSSIBLE_DUPLICATE(409)`, `RECEIPT_NOT_DUPLICATE(409)` | RCT-M01, M02, M04, M05, M09, M10, M12 |
| finance | change `src/test/.../support/AbstractIT.java`: `@MockitoBean FileStorageService` | tests |
| finance | tests: `receipt/service/ReceiptInputParserTest`, `receipt/ReceiptCreateIT`, `ReceiptReviewIT`, `ReceiptDuplicateIT`, `ReceiptPostIT`, `ReceiptDiscardReverseIT`, `ReceiptListIT`, `ReceiptScopeIT`, `ReceiptConcurrencyIT`, `transaction/TransactionReceiptLockIT`; change `integration/MigrationIT.java` | every R / E, test names start with `RCT-…` |
| finance | change `README.md` | docs |
| web | change `next.config.ts`: `experimental.serverActions.bodySizeLimit: "11mb"` | RCT-R02, R03 |
| web | change `src/lib/types/receipts.ts` and `src/lib/types/documents.ts` | RCT-D14, D17 |
| web | change `src/lib/api/financial-records.ts` (the receipt calls and normalizers) and `src/lib/api/banking.ts` (`getBankTransaction(id)`) | §3 |
| web | create `src/features/financial-records/receipt-reading.ts` | RCT-R05, RCT-E10 |
| web | change `actions.ts`: the receipt actions, incl. `uploadReceiptAction` | RCT-R01–R04, R41 |
| web | rewrite `receipts-screen.tsx`, `receipt-row.tsx`, `receipt-cards.tsx`, `receipt-upload.tsx`, `receipt-new.tsx`, `receipt-details.tsx`, `receipt-form.tsx`, `duplicate-panel.tsx`, `discard-receipt-dialog.tsx`, `reverse-expense-dialog.tsx`; create `matching-transaction-sheet.tsx`; delete `receipt-review.tsx` and `receipts-unavailable.tsx` | §4 |
| web | change `content.ts`, `rules.ts` and `queries.ts` | §4 |
| web | change `financial-service.ts`, `src/lib/finance-figures.ts` and `src/lib/finance-rows.ts` | RCT-D14 |
| web | change `src/features/banking/banking-service.ts` and `banking-service-imports.ts`, only where a 409 is dropped | RCT-R47, R52, R53 |
| web | change the route pages `receipts/page.tsx`, `upload/page.tsx`, `new/page.tsx`, `[receipt_id]/page.tsx`; turn `[receipt_id]/review/page.tsx` into a redirect | §4 |
| web | create `e2e/receipts/logic.spec.ts` and `e2e/live/receipts.spec.ts`; change `e2e/contracts/api-responses.spec.ts` and `playwright.logic.config.ts` | tests |
| docs (lead) | `api/finance-service.md` per §3; the stage in `specs/README.md` | — |

## 8. Edge cases

| ID | Where it's handled (repo, layer) |
|---|---|
| RCT-E01 | finance: post locks the receipt row and re-reads it, so the second request gets M09; the unique `transaction_id` is the backstop. `ReceiptConcurrencyIT` |
| RCT-E02 | finance `find`: a manager asking for another branch's receipt gets 404 M08. `ReceiptScopeIT` |
| RCT-E03 | finance `find`: another business's receipt gives 404 M08. `ReceiptScopeIT` |
| RCT-E04 | finance `find` and the list SQL: a null branch is hidden from managers. `ReceiptScopeIT` |
| RCT-E05 | finance post: an inactive account gives M06 and the transaction rolls back; the web keeps the form |
| RCT-E06 | finance post: M07 |
| RCT-E07 | finance: `BusinessTime.today()`; tested with a fixed `Clock` |
| RCT-E08 | finance parser: `KHR_DECIMALS` |
| RCT-E09 | finance: the rate is read at post; the summary converts at today's rate |
| RCT-E10 | web `uploadReceiptAction`: saved with no reading and no fields. finance: empty fields never match |
| RCT-E11 | finance `ReceiptFiles`: a failed upload gives M19 before any DB write |
| RCT-E12 | finance `ReceiptFiles`: the file is deleted after commit; a failure logs the key |
| RCT-E13 | finance: a non-`POSTED` receipt gives M09. In a race, the ledger gives M26 |
| RCT-E14 | finance V5: SET NULL; the status is kept; detail scope is null |
| RCT-E15 | finance PUT: the row lock serializes saves; last save wins |
| RCT-E16 | finance: empty page and zeros; the web shows "No receipts yet" |
| RCT-E17 | finance: no branch-active check |
| RCT-E18 | finance parser: `TOO_MANY`; the web stops adding lines at 100 |
| RCT-E19 | finance `/document`: 404 M08; the web hides the link |
| ~~RCT-E20~~ | withdrawn in v4.0.0; replaced by RCT-E29 |
| RCT-E21 | finance `create`: the file is deleted after a failure, then M19 |
| RCT-E22 | finance: scope comes from `/auth/me` on each request |
| RCT-E23 | finance `DuplicateCheck`: fields only, never files |
| RCT-E24 | finance `DuplicateCheck` plus the advisory lock |
| RCT-E25 | finance post: M24 before the ledger call |
| RCT-E26 | finance: `LedgerService.record` → M23 |
| RCT-E27 | finance parser: M03, nothing stored |
| RCT-E28 | finance: redaction and `OTHER_BRANCH`; the web shows M27 |
| RCT-E29 | finance: the row locks; a deleted row gives 404 M08. If post wins, discard gets M09. Reverse gets 404 once a discard has committed. `ReceiptConcurrencyIT` |
| RCT-E30 | finance: `LedgerService.refund` → `requireOpenPeriod(today)` gives M23. `ReceiptDiscardReverseIT` |

## 9. Packages and verification

- **Packages:** none in either repo. finance uses Jackson, the AWS S3 SDK and Spring JDBC, which it already has. web-ui gets a `next.config.ts` key only.

**finance-service:**

- Run `./gradlew build` (unit tests plus the Testcontainers ITs). Test names start with `RCT-…`.
- **api-reviewer** checks:
  - every code and status in §3, and the post check order;
  - the redaction and the RCT-R49 order;
  - the RCT-R50 regex;
  - the reversal uses `refund`;
  - V5 against the dictionary;
  - no change to `LedgerService` or the attachments endpoint;
  - the file is never stored before validation;
  - the transactions guard runs after scope, and categorize is all-or-nothing.
- **api-qa** runs C / V / M scenarios through the gateway, as owner and manager. Include:
  - the 404s for another branch and another business;
  - a 10 MB + 1 byte file, a 101-line body and a 257 KB reading;
  - a reversal after today's period is signed off;
  - reverse, PATCH and categorize on a receipt's transaction and on its reversal row.

**web-ui:**

- Run `pnpm lint`, `pnpm build` and `pnpm test:logic`, then `GATEWAY_URL=http://localhost:8080 pnpm test:live`.
- **web-reviewer** checks: dictionary names, no sample-data fallback, no confidence scale, the owner-only branch column and filter, every state in §4, and RCT-M28 on the banking screens.
- **web-qa** walks §4 as owner (All branches and one branch) and as manager, including at phone width.

## 10. Questions and conflicts

- **Q1 · Base branches** → answered 2026-11-16 by Vuththana. Both repos branch `feature/receipts` from `origin/dev-cleaned` and target PRs into `dev-cleaned`. web-ui has no `dev` branch, and `dev-cleaned` is the team's live line after the rename.

## Changes

| Date | Change | Why | Approved by |
|---|---|---|---|
| 2026-11-16 | First plan, against spec v4.0.0. It includes the receipt guard in the transactions reverse / PATCH / categorize endpoints (RCT-R47, R48, R52, R53), the duplicate pick order (RCT-R49), the exact-amount search (RCT-R50), the reversal dated today via `refund` (RCT-R51, RCT-E30), and RCT-E29 replacing RCT-E20 | Gate 1; the planner's first-draft questions Q2–Q6 were settled by /sdd:amend (RCT-D38–D45) | Vuththana |
