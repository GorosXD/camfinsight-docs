# finance-service

JSON is snake_case; query parameters use their Java names. Lists use finance's paging: `page` (from 0), `size` (default 20,
clamped to 1–200), `data` = `{ items, page, size, total_elements, total_pages }`.

| Status | Method | Path | Who | Spec |
|---|---|---|---|---|
| 📝 | POST | /api/v1/financial-records/receipts | Owner, Manager | [receipts](../specs/receipts/spec.md) RCT-R01–R08, R41–R43 |
| 📝 | GET | /api/v1/financial-records/receipts | Owner, Manager | RCT-R30–R32 |
| 📝 | GET | /api/v1/financial-records/receipts/summary | Owner, Manager | RCT-R33 |
| 📝 | GET | /api/v1/financial-records/receipts/{id} | Owner, Manager | RCT-R34, R40, R46 |
| 📝 | GET | /api/v1/financial-records/receipts/{id}/document | Owner, Manager | RCT-R35 |
| 📝 | PUT | /api/v1/financial-records/receipts/{id} | Owner, Manager | RCT-R09–R15 |
| 📝 | POST | /api/v1/financial-records/receipts/{id}/post | Owner, Manager | RCT-R16–R22, R44, R45 |
| 📝 | POST | /api/v1/financial-records/receipts/{id}/not-duplicate | Owner, Manager | RCT-R25, R26 |
| 📝 | DELETE | /api/v1/financial-records/receipts/{id} | Owner, Manager | RCT-R36, R37 |
| 📝 | POST | /api/v1/financial-records/receipts/{id}/reverse | Owner, Manager | RCT-R38, R39, R45, R51 |
| 📝 | POST | /api/v1/bank-statements/transactions/{id}/reverse (existing, changed) | Owner, Manager | RCT-R47 |
| 📝 | PATCH | /api/v1/bank-statements/transactions/{id} (existing, changed) | Owner, Manager | RCT-R48, R53 |
| 📝 | POST | /api/v1/bank-statements/transactions/categorize (existing, changed) | Owner, Manager | RCT-R52, R53 |

A **receipt** is the [`receipts`](../data-dictionary/finance.md#receipts) fields plus `lines` (the `receipt_lines` fields),
`category_name`, `account_name`, `branch_name`. The detail adds `document` `{ document_id, file_name, content_type,
size_bytes }` (or null), `extraction`, `transaction_reference`, `reconciliation_state` (`NOTHING_TO_RECONCILE`, `WAITING`,
`MATCHED`, or null when not posted), `history` (the `change_logs` rows) and `duplicate_match_scope` (`SAME_SCOPE`, or
`OTHER_BRANCH` for a manager, who then gets no id or details of the match, RCT-R46). Any receipt not in the caller's scope answers
`404 NOT_FOUND` (RCT-M08).

### POST /api/v1/financial-records/receipts

Multipart: `file` (optional; JPG, PNG, WebP or PDF, at most 10 MB) and `receipt` (JSON): `receipt_number`, `vendor_name`,
`receipt_date`, `business_category_id`, `financial_account_id`, `branch_id`, `subtotal`, `tax_rate`, `tax_amount`,
`original_amount`, `original_currency`, `lines` [{ `description`, `quantity`, `unit_price`, `amount` }] (at most 100),
`extraction` (JSON object, at most 256 KB). Every field is optional. `201 CREATED`, `data`: the **receipt** (status
`NEEDS_REVIEW` or `POSSIBLE_DUPLICATE`).

| Status | `code` | When |
|---|---|---|
| 400 | `VALIDATION_FAILED` | a field breaks its length or range, too many lines, reading too large (RCT-M03) |
| 403 | `BRANCH_ACCESS_DENIED` | a manager names another branch's account (RCT-M11) |
| 413 | `FILE_TOO_LARGE` | file over 10 MB (RCT-M02) |
| 415 | `UNSUPPORTED_FILE_TYPE` | file isn't JPG, PNG, WebP or PDF (RCT-M01) |
| 503 | `DEPENDENCY_UNAVAILABLE` | the file couldn't be stored (RCT-M19) |

### GET /api/v1/financial-records/receipts

Query: `status`, `businessCategoryId`, `financialAccountId`, `dateFrom`, `dateTo` (inclusive), `branchId`, `search`,
`page`, `size`. Order: `receipt_date` desc (no date last), `created_at` desc, `receipt_id`. `200 OK`, `data`: a page of
**receipt** rows without `lines`.

| Status | `code` | When |
|---|---|---|
| 403 | `BRANCH_ACCESS_DENIED` | a manager asks for another branch (RCT-M22) |

### GET /api/v1/financial-records/receipts/summary

Query: `branchId` (a manager asking for another branch gets `403 BRANCH_ACCESS_DENIED`, RCT-M22). `200 OK`, `data`: `{ posted_this_month: { count, usd_total }, possible_duplicate: { count, usd_total },
needs_review: { count }, waiting_to_reconcile: { count } }` (RCT-R33).

### GET /api/v1/financial-records/receipts/{id}

`200 OK`, `data`: the **receipt** detail.

### GET /api/v1/financial-records/receipts/{id}/document

`200 OK`, `data`: `{ url, expires_at }`, a pre-signed link valid 15 minutes. `404 NOT_FOUND` when the receipt has no file.

### PUT /api/v1/financial-records/receipts/{id}

Body: the `receipt` JSON of the create, replacing the fields and lines. `200 OK`, `data`: the **receipt** (duplicate check run again).

| Status | `code` | When |
|---|---|---|
| 400 | `VALIDATION_FAILED` | RCT-M03 |
| 403 | `BRANCH_ACCESS_DENIED` | RCT-M11 |
| 409 | `RECEIPT_NOT_EDITABLE` | the receipt is `POSTED` or `REVERSED` (RCT-M09) |

### POST /api/v1/financial-records/receipts/{id}/post

No body. `200 OK`, `data`: the **receipt**, `POSTED`, with `transaction_id` (RCT-M16).

| Status | `code` | When |
|---|---|---|
| 409 | `RECEIPT_NOT_EDITABLE` | already `POSTED` or `REVERSED` (RCT-M09) |
| 409 | `RECEIPT_POSSIBLE_DUPLICATE` | not resolved yet (RCT-M10) |
| 409 | `PERIOD_LOCKED` | the date is in a signed-off period (RCT-M23) |
| 422 | `RECEIPT_INCOMPLETE` | missing or invalid fields, all listed in `data` (RCT-M04) |
| 422 | `RECEIPT_DATE_IN_FUTURE` | RCT-M05 |
| 422 | `ACCOUNT_INACTIVE` | RCT-M06 |
| 422 | `ACCOUNT_CURRENCY_MISMATCH` | RCT-M24 |
| 422 | `CATEGORY_INACTIVE` | RCT-M07 |
| 422 | `CATEGORY_TYPE_MISMATCH` | RCT-M25 |

### POST /api/v1/financial-records/receipts/{id}/not-duplicate

No body. `200 OK`, `data`: the **receipt**, `NEEDS_REVIEW`. `409 RECEIPT_NOT_DUPLICATE` when it isn't `POSSIBLE_DUPLICATE` (RCT-M12).

### DELETE /api/v1/financial-records/receipts/{id}

`200 OK`, `data`: null (RCT-M17). `409 RECEIPT_NOT_EDITABLE` when `POSTED` or `REVERSED` (RCT-M09).

### POST /api/v1/financial-records/receipts/{id}/reverse

Body: `reason` (required, 1–500). The opposite transaction is dated today (RCT-R51). `200 OK`, `data`: the **receipt**, `REVERSED` (RCT-M18).

| Status | `code` | When |
|---|---|---|
| 400 | `VALIDATION_FAILED` | reason empty (RCT-M13) or too long (RCT-M21) |
| 409 | `RECEIPT_NOT_EDITABLE` | not `POSTED` (RCT-M09) |
| 409 | `TRANSACTION_ALREADY_MATCHED` | matched to a statement line (RCT-M14) |
| 409 | `TRANSACTION_ALREADY_REVERSED` | a reversal raced (RCT-M26) |
| 409 | `PERIOD_LOCKED` | RCT-M23 |

### POST …/transactions/{id}/reverse · PATCH …/transactions/{id} · POST …/transactions/categorize (existing; change)

Unchanged, except that a transaction a receipt points to (`receipts.transaction_id`) is refused, and so is that transaction's reversal by PATCH and categorize. categorize refuses the whole batch:

| Status | `code` | When |
|---|---|---|
| 409 | `TRANSACTION_NOT_REVERSIBLE` | "Change this from its receipt" (RCT-M28, RCT-R47, RCT-R48, RCT-R52, RCT-R53) |

