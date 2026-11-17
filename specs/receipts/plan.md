# Plan: Receipts

**Spec:** `RCT` v6.0.0, commit `41da3a3` · **Approved:** 2026-11-17 by Vuththana (previous: 2026-11-16 by Vuththana, against v5.0.0 `514f871`)
**Repos:** ledgervision-finance-service (backend: owns `receipts`, `receipt_lines`, `documents`, all 10 receipt endpoints; changes 3 existing transaction endpoints; reads branches from identity-service), ledgervision-web-ui (web front end, Next.js BFF)

## 1. Scope (changed)

- **In:**
  - Spec §4a–§4g.
  - Requirements RCT-R01–R04, R06–R14, R16–R26, R30–R33, R35–R38, R40–R66. Withdrawn:
    - RCT-R05 in v6.0.0, replaced by RCT-R60;
    - RCT-R15, R34 and R39 in v5.0.0, replaced by RCT-R56, R55 and R54.
  - Edge cases RCT-E01–E12, E14–E19, E21–E40. Withdrawn: RCT-E13 in v5.0.0 (→ E32) and RCT-E20 in v4.0.0 (→ E29).
  - Messages RCT-M01–M32 (M20 unused).
  - Dictionary §8:
    - `receipts`, `receipt_lines`, `documents`, `documents.extraction`, `document_sequences`, `change_logs.record_type` `RECEIPT`;
    - **v6:** trimmed `vendor_name` / `receipt_number`, `receipt_date` ≥ 2000-01-01, `receipts.branch_id` only set to an active branch.
  - The change to the existing finance transaction endpoints: reverse, PATCH and categorize refuse a receipt's transaction and its reversal row (RCT-R47, R48, R52, R53).
- **Out:**
  - Android (spec §1).
  - CSV export (RCT-D21). There is no Export button on receipts.
  - `POST /ai/receipts/extract` itself, and AI reading of PDFs (RCT-D70).
  - Bank reconciliation of the posted expense. Only its state is shown (RCT-R40).
  - The sidebar "receipts" badge and the legacy `transaction-panel.tsx`, which stay as they are.
  - No change to `LedgerService`, `AccountLookup`, `CategoryLookup`, the attachments endpoint, the transfers endpoints or the internal sale-refund endpoint.
  - No change to `BranchScope.requireAccess` / `branchNames` or to any other finance feature's branch rules. v6 adds receipt-only methods next to them (§6).
  - No change to identity-service.

**Base branches (§10 Q1, answered):**
- v4 / v5: `feature/receipts` from `origin/dev-cleaned` in both repos; merged into `dev-cleaned` (finance #16, web-ui #61).
- **v6:** `feature/receipts-v6` from `origin/dev-cleaned` in both repos; PRs into `dev-cleaned`.

## 2. Repos and parts (changed)

Parts A–H are built and merged (v5). Parts I–N are the v6 work.

| Repo | Role (agent) | Part | Order |
|---|---|---|---|
| finance-service | api-dev | A. Migration `V5__receipts.sql`, entities, `DocumentSequenceService`, receipt `ErrorCode`s (RCT-R01, R03, R07, R08, R24) | built, merged |
| finance-service | api-dev | B. Create, request parsing and M03 validation, file storage, duplicate check (RCT-R01–R04, R06–R08, R23, R24, R41–R43, R49, R57, R58) | built, merged |
| finance-service | api-dev | C. List, summary, detail, document link (RCT-R30–R33, R35, R40, R46, R50, R55, RCT-E31, E33) | built, merged |
| finance-service | api-dev | D. PUT, post, not-duplicate, discard, reverse (RCT-R09–R14, R16–R22, R25, R26, R36–R38, R44, R45, R51, R54, R56, RCT-E32, E34) | built, merged |
| finance-service | api-dev | E. Receipt lock in the transactions reverse, PATCH and categorize (RCT-R47, R48, R52, R53) | built, merged |
| web-ui | web-dev | F–H. Types, data layer, server actions, screens, live e2e | built, merged |
| finance-service | api-dev | **I. (new)** Branches in every state:<br>• `IdentityGateway.getBusinessBranches` (all branches, with `active`), backed by `IdentityDatabase`<br>• `BranchScope.requireAccessAnyState` and `allBranchNames`<br>• the stub gateway's `active` flag<br>• the list and summary filters accept a turned-off branch and keep 403 for one that isn't the business's<br>• rows of a turned-off branch keep their `branch_name`<br>(RCT-R31, R33, RCT-E38) | **first** of v6 |
| finance-service | api-dev | **J. (new)** Save rules on create and PUT:<br>• trimmed text (RCT-R60) and the date bound (RCT-R62)<br>• file type from the bytes (RCT-R61); no reading stored for a PDF<br>• the turned-off-branch refusal `422 BRANCH_INACTIVE` (RCT-R59, M31 / M32) in the RCT-D84 order<br>• post, discard and reverse untouched (RCT-D83)<br>(RCT-R03, R09, R59–R62, RCT-E17, E35, E39, E40) | after I (uses its branch reads) |
| finance-service | api-dev | **K. (new)** Request shape on every receipt endpoint:<br>• a malformed query value or a wrong-type body field → M03 `INVALID` list (RCT-R63)<br>• a non-UUID receipt id → 404 M08 (RCT-E36)<br>• the reverse body read as JSON (closes QA R38-V5)<br>• the page past the end (RCT-E37, test only) | after I; parallel with J |
| web-ui | web-dev | **L. (new)** Upload, form and detail:<br>• file kind from the bytes; a PDF skips AI and gets no "couldn't read" card (RCT-E35, E40)<br>• leave prompt on upload (RCT-R65); a retry calls AI again (RCT-R66)<br>• trimming and the date bound in the form (RCT-R60, R62)<br>• M31 / M32 shown on the field<br>• the manager's own branch name on the form and detail (RCT-R64)<br>• a non-UUID id shows "Receipt not found." (RCT-E36)<br>• the list moves to the last page (RCT-E37); the owner's branch filter includes turned-off branches (RCT-R31)<br>• tests renamed from RCT-R05 to RCT-R60 | **parallel with I–K from day one**; needs only §3 |
| web-ui | web-dev | **M. (new)** Live e2e additions against a local finance-service: RCT-R59, R64, R65, E35, E36, E37, R31 (turned-off filter) | after I–K run locally, and after L |
| docs (lead) | lead | **N. (new)** `api/finance-service.md`:<br>• `422 BRANCH_INACTIVE` and the save refusal order<br>• `TOO_EARLY`; the `INVALID` shape for queries and wrong types<br>• non-UUID id → 404<br>• the file type from the bytes; no reading for a PDF<br>• trimmed text; the owner filter by a turned-off branch<br>Then 📝 → 🚧 → ✅ | after I–K merge |

The web has no mock (its README forbids sample data):
- Normalizers and pure helpers are tested with §3 fixtures (`pnpm test:logic`).
- Screen states are checked against a local finance-service (part M).

## 3. API contract (changed)

All receipt endpoints are finance-service, under `/api/v1/financial-records/receipts`. The details below go into `api/finance-service.md` when the lead updates it.

- JSON is snake_case; query parameters are camelCase.
- The envelope is `{ success, code, message, data, timestamp }`.

**Who, on every receipt endpoint (changed):**

- Owner: all branches, or one with `branchId` where a filter exists.
- Manager: own branch only.
- A SERVICE token: `403 FORBIDDEN_ROLE` "You don't have permission to do this.".
- `404 NOT_FOUND` "Receipt not found." (RCT-M08) for:
  - a receipt of another business;
  - for a manager, a receipt of another branch or one with no branch;
  - a receipt removed by a concurrent discard (RCT-E02–E04, E29);
  - **(changed)** a `{id}` in the path that isn't a UUID, on every `{id}` endpoint, including `/document`, `/post`, `/not-duplicate`, `/reverse` and DELETE (RCT-E36, D76).
- **Body before lookup (RCT-D54):** on PUT and reverse, a body that fails RCT-M03, M13 or M21 gets 400 before the receipt is looked up.
- **(changed) One 400 shape (RCT-R63, D75):** every 400 on a receipt endpoint is `VALIDATION_FAILED` with the RCT-M03 list (below). That covers:
  - a malformed query value (`status=BOGUS`, `page=x`, a non-UUID `branchId`, a bad date);
  - a body field of the wrong type (e.g. a non-text `reason`).
  - Never the map-shaped `data` of the global handler.

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
- `branch_name` goes to both roles (RCT-R32, D60). **(changed)** It is filled for a turned-off branch too (read from all of the business's branches, §6).
- **RCT-R46 redaction:** for a manager, when the stored match is outside their branch or has no branch (RCT-D82), both `duplicate_of_*` are null in every response.

**The detail object** (GET `{id}`, RCT-R55) is the receipt object plus:

- `document`: `{ document_id, file_name, content_type, size_bytes }`, or null. **(changed)** `content_type` is the type found in the file's bytes (RCT-R61).
- `extraction`: the stored object, or null. Always null for a PDF (RCT-D70).
- `transaction_reference`: set when `POSTED` or `REVERSED`. It is the transaction's `reference_number`, or its id as text when that is empty or blank (RCT-D69).
- `reconciliation_state`: `NOTHING_TO_RECONCILE` | `WAITING` | `MATCHED`, only when `POSTED` (RCT-R40, D49).
- `duplicate_match_scope`:
  - `SAME_SCOPE` when the receipt is `POSSIBLE_DUPLICATE`, a match is stored, and the caller is an owner or the match is in the manager's branch;
  - `OTHER_BRANCH` when the stored match is outside the manager's branch or has no branch;
  - null otherwise (RCT-E14).
- `history`: `[{ change_log_id, action, previous_value, new_value, change_at, user_id }]`, newest first.
  - `action` is one of `CREATED`, `UPDATED`, `NOT_DUPLICATE`, `POSTED`, `REVERSED`.
  - For a manager, `duplicate_of_*` values inside `UPDATED` rows are null unless that record is in their branch (RCT-R46).

**Validation error shape (RCT-M03) (changed):**

- `400 VALIDATION_FAILED`, `message` = the first problem's text, `data` = `[{ field, problem, message }]`, one entry per field.
- `field`:
  - in a body, the request's snake_case path (e.g. `lines[3].quantity`, `reason`);
  - **(changed)** in a query, the parameter's name as sent (e.g. `businessCategoryId`, `page`).
- Lengths count characters (code points), after trimming for `vendor_name` and `receipt_number` (RCT-D72).

| `problem` | English text |
|---|---|
| `REQUIRED` | "{field} is required". For `reason`, RCT-M13's "Enter a reason for reversing." |
| `TOO_LONG` | "{field} is at most {max} characters". For `reason`, RCT-M21's "The reason is at most 500 characters." |
| `NEGATIVE` | "{field} is 0 or more" |
| `OUT_OF_RANGE` | "{field} is from 0 to 100" |
| `TOO_MANY_DIGITS` | "{field} has at most {int} digits and {dec} decimals" |
| `KHR_DECIMALS` | "original_amount has no decimals for KHR" |
| `INVALID` | "{field} isn't a valid {kind}". `{kind}`: `date`, `number`, `currency`, `id`, `text`, `list`, `object`, `JSON object` (as the parser does today), and **(changed)** `status` for the `status` query |
| `NOT_FOUND` | "{field} wasn't found" |
| `TOO_MANY` | "lines has at most 100 items" |
| `NOT_AN_OBJECT` | "extraction must be a JSON object" |
| `TOO_LARGE` | "extraction is at most 256 KB" |
| **`TOO_EARLY` (new)** | "{field} is 2000-01-01 or later" (RCT-R62, D74) |

**The `receipt` JSON** (create and PUT):

- Every field is optional. On PUT, a missing field means null (replace).
- These are save-time rules; posting rules are under `/post`. A total or line amount of 0 or less is accepted at save (RCT-D55).
- Numbers are JSON numbers. Unknown properties are ignored. A field of the wrong JSON type gets `INVALID` (RCT-R63).

| Field | Rule at save |
|---|---|
| `receipt_number` | **(changed)** Trimmed; only spaces → null; ≤ 50 after trimming (RCT-R60) |
| `vendor_name` | **(changed)** Trimmed; only spaces → null; ≤ 100 after trimming (RCT-R60) |
| `receipt_date` | ISO date. **(changed)** 2000-01-01 or later, else `TOO_EARLY` (RCT-R62). A future date is allowed until posting |
| `business_category_id` | A category of this business, of any type or state; else `NOT_FOUND` |
| `financial_account_id` | An account of this business; else `NOT_FOUND`. A manager may use only their branch's account or a business-wide one; else `403 BRANCH_ACCESS_DENIED` "You can only use accounts of your branch." (RCT-M11) |
| `branch_id` | Owner: **(changed)** a branch of the business **in any state**; else `403 BRANCH_ACCESS_DENIED` "You don't have access to this branch" (RCT-M22). The check is skipped when it equals the receipt's current branch (RCT-E17). A turned-off branch then meets RCT-R59 (below). Manager: ignored; their own branch is used (RCT-R04, R13) |
| `subtotal` | ≥ 0, (13,2) |
| `tax_rate` | 0–100, (3,2) |
| `tax_amount` | ≥ 0, (13,2) |
| `original_amount` | (13,2); no decimals for KHR (RCT-E08) |
| `original_currency` | `USD` \| `KHR` |
| `lines` | ≤ 100 items of `{ description ≤ 200, quantity ≥ 0 (12,3), unit_price ≥ 0 (13,2), amount (13,2) }`; `line_order` = the position sent, from 1 |
| `extraction` | **Create only.** A JSON object of ≤ 262 144 bytes, or null. Checked even without a file. Not stored when there is no file (RCT-R57) **or (changed) when the file's bytes are a PDF** (RCT-R03, D70). PUT ignores it |

**Branch on save** (RCT-R04, R12, R13):

- An account with a branch: that branch.
- Otherwise, a manager: their own branch.
- Otherwise, an owner: the sent `branch_id` (null allowed).

**(new) Turned-off branch on save (RCT-R59, D71, D83, D85):**

- The branch from the rules above is checked with `IdentityGateway.isActiveBranch(branch)` (identity-service, not cached):
  - on create, when it is not null;
  - on PUT, when it is not null **and** differs from the receipt's current branch.
- Turned off → `422 BRANCH_INACTIVE`, nothing saved:
  - owner: "'{branch_name}' is turned off. Pick another branch or account." (RCT-M31);
  - manager: "Your branch is turned off. Ask the owner to turn it on." (RCT-M32).
- A branch going to null, and a PUT that keeps a turned-off branch, are allowed (RCT-E39).
- Post, discard and reverse never run this check (RCT-D83).

### `POST /api/v1/financial-records/receipts` (changed) — RCT-R01–R04, R06–R08, R23, R24, R41–R43, R49, R57–R63

- **Who:** Owner, Manager.
- **Body:** `multipart/form-data`:
  - `file`: optional, 1 to 10 485 760 bytes.
    - **(changed)** Its type is read from its first bytes (RCT-R61, D73), whatever `Content-Type` or file name it declares:
      - JPG `FF D8 FF`;
      - PNG `89 50 4E 47 0D 0A 1A 0A`;
      - WebP `RIFF` at 0 and `WEBP` at 8;
      - PDF `%PDF-` at 0.
    - Anything else gets M01.
    - `documents.content_type` and the storage key's extension come from the type found (RCT-E40).
    - A file name over 255 characters gets M03 on `file_name`.
  - `receipt`: optional part holding the receipt JSON text. Missing means `{}`.
- **Errors, checked in this order. All run before the file is stored:**
  1. `415 UNSUPPORTED_FILE_TYPE` (RCT-M01): an empty file, or bytes of no allowed type
  2. `413 FILE_TOO_LARGE` (RCT-M02), also when the multipart limit trips
  3. `400 VALIDATION_FAILED` (RCT-M03), incl. `TOO_EARLY` and `NOT_FOUND` references
  4. `403 BRANCH_ACCESS_DENIED` (RCT-M11, manager's account)
  5. `403 BRANCH_ACCESS_DENIED` (RCT-M22, owner's `branch_id`)
  6. **new** `422 BRANCH_INACTIVE` (RCT-M31 / M32)
  7. after storing: `503 DEPENDENCY_UNAVAILABLE` "We couldn't save the file. Try again." (RCT-M19)
- With a file: one `documents` row. Without a file, `document_id` is null (RCT-R08).
- The duplicate check (§5) never takes a reversal transaction (RCT-R58).
- **201 CREATED**, "Receipt saved" (RCT-M15), `data`: the receipt, `NEEDS_REVIEW` or `POSSIBLE_DUPLICATE`.

### `GET /api/v1/financial-records/receipts` (changed) — RCT-R30, R31, R32, R46, R50, R63, RCT-E37, E38

- **Who:** Owner, Manager.
- **Query:**
  - `status`
  - `businessCategoryId`, `financialAccountId`
  - `dateFrom`, `dateTo`: inclusive, on `receipt_date`. Receipts with no date drop out when either is set.
  - `branchId`
  - `search`: `vendor_name` or `receipt_number` contains it, case-insensitive. A trimmed search matching `^[0-9]+(\.[0-9]+)?$` also matches `original_amount` by value (RCT-R50).
  - `page`: from 0; **(changed)** below 0 counts as 0. `size`: default 20, clamped to 1–200. `sort` is ignored.
- **Order:** `receipt_date DESC NULLS LAST, created_at DESC, receipt_id`.
- **Scope (changed):**
  - Owner without `branchId`: the whole business, including null `branch_id`.
  - Owner with `branchId`: that branch, **also a turned-off one** (RCT-R31, D71). A `branchId` that isn't a branch of the business gets 403 M22 (RCT-E38, D78).
  - Manager: their own branch; another `branchId` gets 403 M22.
- **200 OK** `data`: `{ items, page, size, total_elements, total_pages }`. A page past the last one has no items, with the real `total_elements` and `total_pages` (RCT-E37).
- **Errors:**
  - `403 BRANCH_ACCESS_DENIED` (RCT-M22);
  - **(changed)** `400 VALIDATION_FAILED` with `[{ field: <param>, problem: "INVALID", message }]` for a malformed value (RCT-R63).

### `GET /api/v1/financial-records/receipts/summary` (changed) — RCT-R33, R63, RCT-E09, E16, E31, E33

- **Who:** Owner, Manager. **Query:** `branchId`, scoped as in the list. **(changed)** That includes a turned-off branch, and the 403 for an unknown one.
- **200 OK** `data`:
  - `posted_this_month: { count, usd_total }`: `POSTED` with `receipt_date` in the current Asia/Phnom_Penh month; the sum of the stored `usd_equivalent`; never null.
  - `possible_duplicate: { count, usd_total }`:
    - USD receipts add `original_amount` as it is;
    - non-USD receipts are converted at today's rate;
    - the rate is read only when a non-USD `POSSIBLE_DUPLICATE` is in scope (RCT-D51, D64);
    - if that read fails with `DEPENDENCY_UNAVAILABLE`, `usd_total` is null and `count` is still given (RCT-D63).
  - `needs_review: { count }`.
  - `waiting_to_reconcile: { count }`: `POSTED`, a `BANK_ACCOUNT` or `MOBILE_WALLET` account, and no `bank_transactions.matched_transaction_id` = `transaction_id`.
  - All zeros when nothing is in scope (RCT-E16). A non-null `usd_total` is rounded to 2 decimals.
- **Errors:** `403` (RCT-M22); **(changed)** `400` `INVALID` for a malformed `branchId` (RCT-R63). There is no 503 for the rate.

### `GET /api/v1/financial-records/receipts/{id}` (changed) — RCT-R55, R40, R46, R24, R63, RCT-E36

- **Who:** Owner, Manager. **200 OK** `data`: the detail object.
- **Errors:** `404` (RCT-M08), **(changed)** also for a non-UUID `{id}`.

### `GET /api/v1/financial-records/receipts/{id}/document` (changed) — RCT-R35, R63, RCT-E19, E36

- **Who:** Owner, Manager.
- **200 OK** `data`: `{ url, expires_at }`, a pre-signed GET valid for 15 minutes, from the receipt's `documents` row.
- **Errors:** `404` (RCT-M08) when the receipt is out of scope, has no file, or **(changed)** the id isn't a UUID.

### `PUT /api/v1/financial-records/receipts/{id}` (changed) — RCT-R09–R14, R23, R24, R49, R56, R59, R60, R62, R63, RCT-E14, E15, E17, E39

- **Who:** Owner, Manager.
- **Body:** `application/json`, the receipt JSON. It replaces the fields and lines; `extraction` is ignored.
- **Errors, checked in this order (changed, RCT-D84: 400 → 404 → access → state):**
  1. `400 VALIDATION_FAILED` (RCT-M03): the body (incl. trimming, `TOO_EARLY`, wrong types), and a `business_category_id` or `financial_account_id` not in the business. All of it before the receipt is looked up (RCT-D54).
  2. `404` (RCT-M08), also for a non-UUID `{id}`.
  3. `403 BRANCH_ACCESS_DENIED` (RCT-M11): a manager sends another branch's account. This now comes **after** the lookup (it was before it in v5).
  4. `403 BRANCH_ACCESS_DENIED` (RCT-M22): an owner's `branch_id` isn't a branch of the business (any state), unless it is the current branch.
  5. `409 RECEIPT_NOT_EDITABLE` "This receipt is posted and can't be changed." / "… reversed …" (RCT-M09).
  6. **new** `422 BRANCH_INACTIVE` (RCT-M31 / M32) when the branch would change to a turned-off one.
- **Duplicate step** (RCT-R10, D59, D66):
  - Skipped when the receipt is `NEEDS_REVIEW` before the save and the latest `NOT_DUPLICATE` snapshot has the same key fields as the new ones (vendor compared trimmed and case-insensitive).
  - Otherwise the check of §5 runs.
- **No-change save** (RCT-R56, D50, D65):
  - Fields (money by value, text after trimming), lines (content and order), status and both `duplicate_of_*` are compared with the stored ones.
  - If nothing differs, nothing is written.
  - Otherwise: save, one `UPDATED` row (§5), and `updated_at` moves.
- **200 OK**, "Receipt saved" (RCT-M15), `data`: the receipt after the duplicate step.

### `POST /api/v1/financial-records/receipts/{id}/post` (changed) — RCT-R16–R22, R44, R45, R63

- **Who:** Owner, Manager. **Body:** none.
- **200 OK**, "{receipt_code} is posted as {transaction_reference}." (RCT-M16), `data`: the receipt, `POSTED`.
- **Errors, checked in this order:**
  1. `404` (RCT-M08), **(changed)** also for a non-UUID `{id}`
  2. `409 RECEIPT_NOT_EDITABLE` (RCT-M09)
  3. `409 RECEIPT_POSSIBLE_DUPLICATE` (RCT-M10)
  4. `422 RECEIPT_INCOMPLETE` (RCT-M04): "Fix {n} fields to save this receipt", or "Fix 1 field to save this receipt". `data` `[{ field, problem }]`:
     - `REQUIRED` for `vendor_name`, `receipt_date`, `original_currency`, `business_category_id`, `financial_account_id`, `branch_id`;
     - `NOT_POSITIVE` for `original_amount`.
  5. `422 RECEIPT_DATE_IN_FUTURE` (RCT-M05)
  6. `403 BRANCH_ACCESS_DENIED` (RCT-M11, manager)
  7. `422 ACCOUNT_INACTIVE` (RCT-M06)
  8. `422 ACCOUNT_CURRENCY_MISMATCH` (RCT-M24)
  9. `422 CATEGORY_INACTIVE` (RCT-M07)
  10. `422 CATEGORY_TYPE_MISMATCH` (RCT-M25)
  11. `409 PERIOD_LOCKED` (RCT-M23)
  12. `503 DEPENDENCY_UNAVAILABLE` when the rate can't be read
- There is no turned-off-branch check (RCT-D83). A refusal changes nothing (RCT-E05).

### `POST /api/v1/financial-records/receipts/{id}/not-duplicate` (changed) — RCT-R25, R26, R63

- **Who:** Owner, Manager. **Body:** none.
- Writes `NOT_DUPLICATE` with the snapshot `{vendor_name, receipt_date, original_amount, original_currency}`.
- **200 OK**, "Marked as not a duplicate" (RCT-M29), `data`: the receipt, `NEEDS_REVIEW`, with both `duplicate_of_*` null.
- **Errors:** `404` (RCT-M08, **(changed)** also for a non-UUID `{id}`); `409 RECEIPT_NOT_DUPLICATE` (RCT-M12).

### `DELETE /api/v1/financial-records/receipts/{id}` (changed) — RCT-R36, R37, R63

- **Who:** Owner, Manager.
- **200 OK**, `data`: null, with the message:
  - "{receipt_code} from {vendor_name} was removed. Nothing was counted.";
  - when there is no vendor: "{receipt_code} was removed. Nothing was counted." (RCT-M17).
- There is no turned-off-branch check (RCT-D83).
- **Errors:** `404` (RCT-M08, **(changed)** also for a non-UUID `{id}`); `409` (RCT-M09).

### `POST /api/v1/financial-records/receipts/{id}/reverse` (changed) — RCT-R38, R45, R51, R54, R63, RCT-E30, E32, E34

- **Who:** Owner, Manager.
- **Body (changed):** `{ "reason": string }`, read as a JSON tree, never bound to a Java type (Jackson would turn `5` into `"5"`). All of these run before the lookup (RCT-D54):
  - A body that isn't JSON or isn't an object → `[{ field: "body", problem: "INVALID", message: "body isn't a valid JSON object" }]`.
  - A missing body or a missing / null `reason` → `REQUIRED` (RCT-M13).
  - A `reason` that isn't a JSON string → `[{ field: "reason", problem: "INVALID", message: "reason isn't a valid text" }]` (RCT-R63; QA R38-V5).
  - A string is trimmed, then must be 1–500 characters: an empty result → M13 (`REQUIRED`); over 500 → M21 (`TOO_LONG`).
- **Errors, checked in this order (RCT-D61):**
  1. `400` as above;
  2. `404` (RCT-M08), also for a non-UUID `{id}`;
  3. `409 RECEIPT_NOT_REVERSIBLE` (RCT-M30) unless `POSTED`;
  4. `409 TRANSACTION_ALREADY_MATCHED` (RCT-M14);
  5. `409 PERIOD_LOCKED` (RCT-M23), from `LedgerService.requireOpenPeriod(account, today)` before `refund`;
  6. `409 TRANSACTION_ALREADY_REVERSED` (RCT-M26), from `refund` in a race.
- There is no receipt row lock. There are no active checks on the account, category (RCT-E34) or branch (RCT-D83).
- The reversal is dated today in Asia/Phnom_Penh (`LedgerService.refund`, RCT-R51).
- **200 OK**, "{receipt_code} is now Reversed. {amount} went back to {account_name}." (RCT-M18), `data`: the receipt, `REVERSED`.
  - `{amount}`: "1,250.50 USD", "41,000 KHR", with fixed separators.

### `POST /api/v1/bank-statements/transactions/{id}/reverse` — existing, changed · RCT-R47

- Who, body and response are unchanged.
- After the in-scope find, before the transfer and source checks: if a `receipts.transaction_id` = `{id}`, refuse with `409 TRANSACTION_NOT_REVERSIBLE` "Change this from its receipt" (RCT-M28).

### `PATCH /api/v1/bank-statements/transactions/{id}` — existing, changed · RCT-R48, R53

- Who, body and response are unchanged.
- Before the signed-off check and before any change: refuse with RCT-M28 when either holds:
  - a `receipts.transaction_id` = `{id}`;
  - the transaction's `reverses_transaction_id` is a receipt's `transaction_id`.

### `POST /api/v1/bank-statements/transactions/categorize` — existing, changed · RCT-R52, R53

- Who, body and response are unchanged.
- All ids are resolved first (404 out of scope).
- Then, before any change: if any id is a receipt's transaction or its reversal, the whole batch is refused with RCT-M28.

**Existing endpoints the web also uses (unchanged):** `GET /api/v1/bank-statements/accounts`, `GET /api/v1/financial-records/categories`, `GET /api/v1/bank-statements/transactions/{id}` (RCT-R24), `POST /ai/receipts/extract`.

## 4. Screens, routes and states (changed)

| Route | Who | States | IDs |
|---|---|---|---|
| `/financial-records/receipts` | Owner, Manager | Cards and table load separately, with skeletons and retry; empty states; tabs; filters; rows; paging; a null card total shows "—"; **(changed)** a page past the end moves to the last page; the owner's branch filter lists turned-off branches too | RCT-R30–R33, R50, R63, RCT-E16, E31, E33, E37, E38 |
| `/financial-records/receipts/upload` | Owner, Manager | Pick, reading, saving, error, success. **(changed)** File kind from the bytes; PDF not sent to AI; leave prompt; a retry calls AI again; M31 / M32 | RCT-R01–R04, R41, R57, R59, R61, R65, R66, RCT-E10, E11, E21, E24, E35, E40 |
| `/financial-records/receipts/new` | Owner, Manager | Empty form, save for later, save as expense, field errors. **(changed)** Trimmed text, the date bound, M31 / M32, the manager's branch name | RCT-R08, R42, R43, R57, R59, R60, R62, R64 |
| `/financial-records/receipts/[id]` | Owner, Manager | Loading; 404 → not-found screen (M08, also after a concurrent discard and **(changed)** for a non-UUID id); load error with retry; one state per status | RCT-R09–R14, R16–R26, R35–R38, R40, R44–R46, R49, R51, R54–R56, R59, R60, R62, R64, RCT-E05–E07, E14, E15, E17, E25, E26, E28–E30, E32, E34–E36, E39 |
| `/financial-records/receipts/[id]/review` (legacy) | — | Redirects to `/financial-records/receipts/[id]` | — |
| Banking screens that reverse or categorize | Owner, Manager | On `409 TRANSACTION_NOT_REVERSIBLE`, the server message (RCT-M28) | RCT-R47, R48, R52, R53 |

**`/financial-records/receipts` (changed):**

- Empty with no receipts at all: "No receipts yet". Empty for a search or filters: "No receipts found" or "No receipts match these filters".
- Tabs All / Needs review / Possible duplicate / Posted, sent as `status`; `REVERSED` shows under All.
- Filters: category, paid from, date range, branch (owner only), search.
  - **(changed)** The branch filter offers every branch of the business, with turned-off branches labelled "Turned off" (RCT-R31).
- Rows: code, vendor, date, category, account, total with currency, USD ("—" while null), status, and the branch for an owner only (the API's `branch_name`).
- Cards: a null `possible_duplicate.usd_total` shows the count and "—" (RCT-E33).
- 20 a page.
  - **(changed)** When a page comes back empty with `total_pages` > 0 and `page` ≥ `total_pages`, the list loads page `total_pages − 1` (RCT-E37).
  - M22 and any 400 are shown as the list error.
- No Export button.

**`/financial-records/receipts/upload` (changed):**

- The client check reads the file's first bytes (`file-signature.ts`): JPG, PNG, WebP or PDF, up to 10 MB; else M01 / M02 (RCT-R61, E40).
- Images (by bytes) go to AI extract. A PDF (by bytes) skips it and is saved with no reading (RCT-D70, E35).
- **Leave prompt** from the moment a file is picked until it is saved (`use-leave-prompt.ts`, RCT-R65).
- Server errors M01 / M02 / M03 / M11 / M19 / M22 / M31 / M32 show with a retry. M31 / M32 show the server's message.
  - **Retry** sends the same file through `uploadReceiptAction` again, which calls AI extract again. No reading is kept between tries (RCT-R66, E24).
- On success: toast M15, then `/receipts/{id}`.
- When AI fails on an image, the receipt is saved with no reading and no fields (RCT-E10).

**`/financial-records/receipts/new` (changed):**

- An empty form with no file; no `extraction` is sent (RCT-R57).
- `vendor_name` and `receipt_number` are trimmed before sending, and only spaces is sent as null. The length limit counts after trimming (RCT-R60).
- The date picker's earliest date is 2000-01-01; an earlier typed date shows the `TOO_EARLY` text (RCT-R62).
- Branch field:
  - an owner picks only active branches;
  - **a manager sees their own branch's name** (`viewer.ownBranch.branch_name`, RCT-R64).
- "Save for later" → POST. "Save as expense" → POST, then `/post`; a refused post goes to `/receipts/{id}` with the error shown.
- M03 errors show under each field.
- `422 BRANCH_INACTIVE` (M31 / M32, the server's message) shows under the branch field, or under "Paid from" when the branch came from the account.
- Leave prompt when the form is dirty.

**`/financial-records/receipts/[id]`, by status:**

- **(changed) Any status:**
  - The route checks the id's UUID shape first. A non-UUID shows the not-found screen "Receipt not found." without calling the API (RCT-E36).
  - A manager sees the receipt's `branch_name` (RCT-R64).
- **`NEEDS_REVIEW`:**
  - The editable form, with the document viewer.
  - Category: active EXPENSE categories plus the current one. Paid from: for a manager, own-branch and business-wide accounts.
  - Branch: from the account; an owner picks an active one for a business-wide account; fixed (with the name) for a manager.
  - Totals are typed; nothing is computed. Lines ≤ 100.
  - **(changed)** The "We couldn't read this receipt" card shows only when the file is an image (`document.content_type` starts with `image/`) and the reading has no values. Never for a PDF (RCT-E10, E35).
  - Save for later → PUT → toast M15.
    - **(changed)** M31 / M32 under the branch or account field. A receipt whose branch is turned off but unchanged saves normally (RCT-E39).
  - Save as expense → PUT, then `/post`. Post errors:
    - M04 banner with the count, fields marked;
    - M05;
    - M06 / M07 / M23 / M24 / M25 as the server's message;
    - M09 as an alert, then the receipt reloads read-only.
  - Discard → confirm → DELETE → toast M17 → back to the list.
- **`POSSIBLE_DUPLICATE`:** the same form plus a compare panel.
  - `SAME_SCOPE`: the match is shown beside this one, with a link or a read-only transaction sheet.
  - `OTHER_BRANCH`: M27 only.
  - No stored match: the M10 text.
  - "Same expense, discard receipt" or "Not a duplicate, continue review" (toast M29; M12 as an alert). "Save as expense" is disabled.
- **`POSTED`:**
  - Read-only, with `transaction_reference`, a reconciliation chip and the document link.
  - Reverse dialog (dated today):
    - the reason is trimmed; empty → M13, over 500 → M21 inline;
    - a server `INVALID` on `reason` shows inline too;
    - M14 / M23 as alerts;
    - M30 / M26: an alert, then the receipt reloads;
    - toast M18.
- **`REVERSED`:** read-only, with the reason and `transaction_reference`.
- **History card** on every status.
- Toasts M16, M17, M18 and M29 show the server's `message`.

Confidence numbers are not shown anywhere.

## 5. Data (changed)

| Table.field (dictionary name) | Read / write | Repo | IDs |
|---|---|---|---|
| `receipts` (all fields) | Write. V5 schema, CHECKs, FKs and indexes as built. **(changed)**:<br>• `vendor_name` / `receipt_number` are written trimmed, null for only spaces (RCT-R60)<br>• `receipt_date` < 2000-01-01 never reaches the table (parser, RCT-R62). There is no migration: the rule is checked on save, and no stored row breaks it<br>• `branch_id` is written only to an active branch, or kept unchanged, or null (RCT-R59)<br>A no-change PUT writes nothing (RCT-R56) | finance | RCT-R01, R03, R04, R06–R09, R12, R13, R17–R19, R23–R25, R36, R38, R49, R56, R59, R60, R62, RCT-E14, E39 |
| `receipts.receipt_code` | Written once at create: `RCP-{yyyy}-{seq:04d}`, the year of `BusinessTime.today()` | finance | RCT-R01 |
| `document_sequences` | Write; `DocumentSequenceService.next` upserts it | finance | RCT-R01 |
| `receipt_lines` (all fields) | Write; replaced on a changing save with `line_order` 1..n | finance | RCT-R09, R42, R36, R56 |
| `documents` (all fields), `documents.extraction` | Written at create only when a file is sent; deleted at discard. `storage_key` = `finance/attachments/{business_id}/receipts/{uuid}.{ext}`. **(changed)**:<br>• `content_type` and `{ext}` come from the file's bytes (RCT-R61, E40)<br>• `extraction` is null for a PDF, also when one was sent (RCT-R03, D70, E35) | finance | RCT-R03, R08, R35, R36, R41, R57, R61, RCT-E35, E40 |
| `transactions` (via `LedgerService.record` / `refund`) | Written at post and at reverse, as built. Read by the duplicate check (no reversal rows) | finance | RCT-R17–R19, R23, R38, R45, R51, R54, R58, RCT-E30, E32, E34 |
| `receipts.transaction_id`, `transactions.reverses_transaction_id` | Read by the transactions guard | finance | RCT-R47, R48, R52, R53 |
| `receipts.exchange_rate`, `receipts.usd_equivalent` | Copied from the new transaction at post | finance | RCT-R18 |
| `financial_accounts`, `business_categories` | Read (not their active state at reverse) | finance | RCT-R12–R14, R22, R32, R33, R40, R44, RCT-E34 |
| `bank_transactions.matched_transaction_id` | Read | finance | RCT-R33, R40, R54 |
| `change_logs` (`record_type` `RECEIPT`) | Write one row per action (below); read for `history` and RCT-R10 | finance | RCT-R07, R10, R19, R25, R36, R38, R55, R56 |
| `businesses.exchange_rate` | Read at post, and in the summary only when needed | finance | RCT-R18, R33, RCT-E31, E33 |
| **(new)** identity-service `branches` (`branch_id`, `branch_name`, `is_active`, `business_id`) | Read only, through `IdentityGateway`:<br>• `isActiveBranch(id)`: identity's `GET /api/v1/branches/{id}` with the caller's token, not cached. Used for the RCT-R59 check<br>• **new** `getBusinessBranches(businessId)`: every branch of the business with its `active` flag. Read from `identity.branches` by `IdentityDatabase` (no `is_active` filter), cached 10 minutes like `getBranches`. Used for the owner's filter scope (any state), the M22 check on save, `branch_name` of a turned-off branch, and `{branch}` in M31 | finance | RCT-R31, R32, R33, R59, RCT-E38 |
| All receipt fields, via §3 | Read / sent | web-ui | as §4 |

**`change_logs` rows written for receipts:**

- **CREATED:** new = the saved fields as JSON (no extraction).
- **UPDATED:** only when something changed (fields, lines, status, `duplicate_of_*`). previous / new = the changed fields, plus `status` and `duplicate_of_*` when they changed.
- **NOT_DUPLICATE:** new = `{vendor_name, receipt_date, original_amount, original_currency}`.
- **POSTED:** new = `transaction_id`.
- **REVERSED:** new = the trimmed reason.
- **DISCARDED:** previous = `{receipt_code, vendor_name, original_amount, original_currency}`.

**Duplicate check** (RCT-R23, R24, R49, R58, R10). It runs on create and on each PUT, inside the save transaction:

1. Take `pg_advisory_xact_lock` on the business.
2. Run only when vendor, date, amount and currency are all set.
3. One UNION of:
   - unposted receipts of the business;
   - `POSTED` `EXPENSE` transactions of the business's accounts that aren't a reversal, have no reversal, and aren't this receipt's own.
   - They are matched on the trimmed, lower-cased vendor, date, amount and currency.
4. Order: a manager's own branch first; then transactions before receipts; then `created_at`; then id. Keep the first.
5. On PUT, skip only from `NEEDS_REVIEW` with an unchanged `NOT_DUPLICATE` snapshot.

The stored text is trimmed now (RCT-R60), so the stored and compared vendor agree.

## 6. Roles and branch scope (changed)

**finance, receipt endpoints:**

- `CurrentUserProvider.getUser()`; a SERVICE caller gets 403 FORBIDDEN_ROLE.
- Every query has `business_id = :business`. A manager is limited to `branch_id = user.branchId` (RCT-R30, E04, E22).
- **(changed) Branch filter (list, summary):** `BranchScope.requireAccessAnyState(user, branchId)`:
  - owner: the branch must be in `getBusinessBranches` (any state), else 403 M22 (RCT-R31, E38);
  - manager: only their own branch.
  - `BranchScope.requireAccess` (active branches only) stays as it is for the rest of finance.
- **(changed) Branch names:** receipts use `BranchScope.allBranchNames(user)` (from `getBusinessBranches`), so a turned-off branch keeps its name. `branchNames` stays for the rest of finance.
- Finding by id:
  - a business mismatch, or for a manager a branch mismatch or a null branch, gives 404 M08;
  - a concurrent discard gives 404 M08 (`EntityNotFoundException` from `RowLocks.lock`);
  - **(changed)** a non-UUID id gives 404 M08, through `ReceiptController`'s handler for `MethodArgumentTypeMismatchException` on `receiptId` (RCT-E36).
- **(changed) Save order (RCT-D84):**
  - create: 400 → M11 → M22 → M31 / M32;
  - PUT: 400 → 404 → M11 → M22 → M09 → M31 / M32.
  - The PUT's `references()` is split: the `NOT_FOUND` part runs before the lookup; the `mayUseAccount` part (M11) runs after it.
- Accounts: `AccountLookup.find` plus `CurrentUser.mayUseAccount`, else M11 (RCT-R14). Categories: `CategoryLookup.find`.
- Duplicate check: business-wide, with a manager's own branch first (RCT-R49). One mapper redacts per viewer, also inside history (RCT-R46). A match with no branch counts as another branch's for a manager (RCT-D82).
- `branch_name` goes to both roles (RCT-D60).
- **Turned-off branch (RCT-R59):** checked only on create and on a branch-changing PUT, with `isActiveBranch`. Post, discard and reverse don't check it (RCT-D83).

**Identity source (changed):**

- `getBusinessBranches` reads `identity.branches` through `IdentityDatabase`, the configured path in dev (`application.yml`) and k8s (`consul/finance-service-k8s.yml`).
- Without `IdentityDatabase`, it falls back to the existing `IdentityClient.getBranches` internal call. That call returns active branches only, so a turned-off branch is then unknown, the same limit as today's `getBranches`. identity-service gains no endpoint in this plan.

**finance, transaction endpoints:** unchanged. The receipt guard runs after scope (RCT-R47, R48, R52, R53).

**web-ui:**

- Both roles reach every receipt route.
- The branch column and branch filter show only for an owner.
- **(changed)** The manager's form and detail show their own branch name (RCT-R64).
- The account picker uses the backend's scoped list.
- The web never filters scope itself.

## 7. Files (changed)

**Built and merged in v4 / v5 (unchanged unless listed below):**

- finance:
  - `V5__receipts.sql`, `DocumentSequenceService`, `CurrentUserProvider.getUser()`, `document/{entity,repository}`;
  - `receipt/{entity,repository,dto}`;
  - `receipt/service/{ReceiptInputParser, DuplicateCheck, ReceiptFiles, ReceiptHistory, ReceiptMapper, ReceiptQueries, ReceiptPosting, ReceiptService, ReceiptTransactions}`, `receipt/service/impl/ReceiptServiceImpl`, `receipt/controller/ReceiptController`;
  - the guard in `TransactionServiceImpl`; the receipt `ErrorCode`s (`UNSUPPORTED_FILE_TYPE`, `FILE_TOO_LARGE`, `RECEIPT_INCOMPLETE`, `RECEIPT_DATE_IN_FUTURE`, `RECEIPT_NOT_EDITABLE`, `RECEIPT_POSSIBLE_DUPLICATE`, `RECEIPT_NOT_DUPLICATE`, `RECEIPT_NOT_REVERSIBLE`);
  - test support and the receipt ITs; `README.md`.
- web:
  - `next.config.ts`, `src/lib/types/{receipts,documents}.ts`, `src/lib/api/{financial-records,receipt-normalizers,banking}.ts`, `actions.ts`, `src/features/banking/actions.ts`;
  - the `receipt-*` screens and helpers, `discard-receipt-dialog.tsx`, `reverse-expense-dialog.tsx`, `matching-transaction-sheet.tsx`, `content.ts`, `rules.ts`, the route pages;
  - `e2e/receipts/logic.spec.ts`, `e2e/live/receipts.spec.ts`, `e2e/contracts/api-responses.spec.ts`.

**v6 (`feature/receipts-v6`):**

| Repo | Create / change | Why (IDs) |
|---|---|---|
| finance | change `exception/ErrorCode.java`: add `BRANCH_INACTIVE(HttpStatus.UNPROCESSABLE_ENTITY)` | RCT-M31, M32, R59 |
| finance | change `client/identity/IdentityGateway.java`: add `List<BranchState> getBusinessBranches(UUID businessId)` and `record BranchState(UUID branchId, String branchName, boolean active)` | RCT-R31, R59, RCT-E38 |
| finance | change `client/identity/IdentityDatabase.java`: `getBusinessBranches`, `SELECT branch_id, branch_name, is_active FROM identity.branches WHERE business_id = :businessId` (no `is_active` filter) | RCT-R31, R59 |
| finance | change `client/identity/FeignIdentityGateway.java`: `getBusinessBranches` with its own 10-minute `TtlCache`; `IdentityDatabase` when present, else `client.getBranches` mapped as active | RCT-R31, R59 |
| finance | change `client/identity/StubIdentityGateway.java` and `config/FinanceProperties.StubBranch`: an optional `active` flag (null = true). `isActiveBranch` honours it; `getBranches` returns active ones; `getBusinessBranches` returns all. Add a turned-off branch to the test stub config | tests, RCT-R59 |
| finance | change `common/security/BranchScope.java`: add `requireAccessAnyState(user, branchId)` and `allBranchNames(user)`. `requireAccess` and `branchNames` are unchanged | RCT-R31, R33, RCT-E38 |
| finance | change `receipt/service/ReceiptQueries.java`: the list and summary use `requireAccessAnyState`; list, detail and `get` use `allBranchNames` | RCT-R31, R32, R33, RCT-E37, E38 |
| finance | change `receipt/service/ReceiptFiles.java`: `detectType(first 12 bytes)` → JPG / PNG / WebP / PDF or M01. The declared type is ignored; `StoredFile.contentType` and the key extension come from the detected type | RCT-R61, RCT-E40 |
| finance | change `receipt/service/ReceiptInputParser.java`: trim `vendor_name` / `receipt_number` (`strip()`; empty → null; length after trimming); `TOO_EARLY` for `receipt_date` < 2000-01-01 | RCT-R60, R62 |
| finance | change `receipt/service/impl/ReceiptServiceImpl.java`:<br>• the RCT-R59 check (`isActiveBranch`; M31 with the name from `allBranchNames`, M32 for a manager) on create and on a branch-changing PUT<br>• PUT order per §6 (split `references()`)<br>• M22 via `requireAccessAnyState`<br>• `extraction` dropped when the detected type is PDF<br>• `reverse(String body)` parsed as a JSON tree per §3 | RCT-R03, R09, R59, R63, RCT-E17, E35, E39 |
| finance | change `receipt/controller/ReceiptController.java`:<br>• a local `@ExceptionHandler(MethodArgumentTypeMismatchException)`: `receiptId` → 404 M08; any other parameter → 400 `[{field: <name>, problem: "INVALID", message}]`, `{kind}` by target type (UUID → id, LocalDate → date, int → number, enum → status)<br>• reverse takes `@RequestBody(required = false) String` | RCT-R63, RCT-E36 |
| finance | change `receipt/dto/ReceiptDtos.java`: remove `ReverseRequest` (no longer bound) | RCT-R63 |
| finance | tests:<br>• rename the RCT-R05 tests to RCT-R60 (`ReceiptCreateIT`, `ReceiptInputParserTest`)<br>• `ReceiptInputParserTest`: RCT-R60, R62<br>• `ReceiptFilesTest`: RCT-R61, E40<br>• `ReceiptCreateIT`: RCT-R59 (owner M31, manager M32, account's branch), R60, R61, R62, E35<br>• `ReceiptReviewIT`: RCT-R59 on PUT, E39, R60, R62, the D84 order<br>• `ReceiptListIT`: RCT-R31 (turned-off filter, also summary), R32 (turned-off branch name), R63 (`status=BOGUS`, `page=x`, bad `branchId`), E37, E38<br>• `ReceiptScopeIT`: RCT-E36 on every `{id}` endpoint<br>• `ReceiptDiscardReverseIT`: RCT-R63 (non-string reason, non-object body)<br>• `ReceiptPostIT` / `ReceiptDiscardReverseIT`: post, discard and reverse on a turned-off branch (RCT-E17) | every v6 ID; names start with `RCT-…` |
| finance | change `README.md`: `422 BRANCH_INACTIVE` and the save order; the byte check; `TOO_EARLY`; trimming; the owner filter by a turned-off branch; non-UUID id → 404 | docs |
| web | create `src/features/financial-records/file-signature.ts`: `fileKind(file)` reads `file.slice(0, 12)` → `"jpeg" \| "png" \| "webp" \| "pdf" \| null` | RCT-R61, RCT-E35, E40 |
| web | change `rules.ts`: the receipt file check uses `fileKind`; the receipt date's earliest is 2000-01-01 | RCT-R61, R62 |
| web | change `actions.ts` `uploadReceiptAction`: kind by bytes; images → AI extract on every call (no reading reused); PDF → no AI, `extraction` null | RCT-R66, RCT-E35, E40 |
| web | change `receipt-upload.tsx`: client check by bytes; leave prompt via `use-leave-prompt.ts` from pick to save; retry re-sends the file; M31 / M32 shown | RCT-R59, R65, R66 |
| web | change `receipt-values.ts`: trim the two text fields, only spaces → null, length after trimming; client `TOO_EARLY` | RCT-R60, R62 |
| web | change `receipt-form.tsx`: a manager's branch shows `viewer.ownBranch.branch_name` or the receipt's `branch_name`; the owner's picker lists active branches; `BRANCH_INACTIVE` under the branch or account field; the "couldn't read" card only for an image `content_type` | RCT-R59, R64, RCT-E10, E35 |
| web | change `receipt-readonly.tsx`: the branch name for a manager | RCT-R64 |
| web | change `[receipt_id]/page.tsx` (and `receipt-details.tsx`): a non-UUID id renders the not-found screen without an API call | RCT-E36 |
| web | change `receipts-screen.tsx` and `receipt-state.ts`: `pageToShow(page, total_pages)` moves past-the-end to the last page; the owner's branch filter lists every branch, "Turned off" labelled | RCT-R31, RCT-E37 |
| web | change `content.ts`: M31 / M32 fallbacks, the `TOO_EARLY` problem text, the "Turned off" label | RCT-R59, R62 |
| web | tests:<br>• rename the RCT-R05 tests to RCT-R60 in `e2e/receipts/logic.spec.ts` and `e2e/contracts/api-responses.spec.ts`<br>• logic tests: RCT-R60, R62, E37 (`pageToShow`), E40 (`fileKind` on byte fixtures)<br>• contract fixtures: `BRANCH_INACTIVE` and an `INVALID` query error<br>• live e2e per part M | every v6 web ID |
| docs (lead) | `api/finance-service.md` per §3; `specs/README.md` stage | — |

## 8. Edge cases (changed)

| ID | Where it's handled (repo, layer) |
|---|---|
| RCT-E01 | finance post: the row lock and re-read; the second gets M09; the unique `transaction_id` is the backstop. `ReceiptConcurrencyIT` |
| RCT-E02 | finance `find`: 404 M08; a failing PUT or reverse body gets 400 first. `ReceiptScopeIT` |
| RCT-E03 | finance `find`: 404 M08; a failing body gets 400 first. `ReceiptScopeIT` |
| RCT-E04 | finance `find` and the list SQL: a null branch is hidden from managers |
| RCT-E05 | finance post: M06, rolled back; the web keeps the form |
| RCT-E06 | finance post: M07 |
| RCT-E07 | finance: `BusinessTime.today()`, tested with a fixed `Clock` |
| RCT-E08 | finance parser: `KHR_DECIMALS` |
| RCT-E09 | finance: the rate at post; the summary converts at today's rate |
| RCT-E10 | **(changed)** web: an image AI couldn't read is saved with no fields; the card shows only for an image `content_type` |
| RCT-E11 | finance `ReceiptFiles`: M19 before any DB write |
| RCT-E12 | finance `ReceiptFiles`: delete after commit; a failure logs the key |
| ~~RCT-E13~~ | withdrawn in v5.0.0; replaced by RCT-E32 |
| RCT-E14 | finance: SET NULL; the next save re-checks and logs the status change |
| RCT-E15 | finance PUT: the row lock serializes saves; last save wins |
| RCT-E16 | finance: empty page and zeros; web "No receipts yet" |
| RCT-E17 | **(changed)** finance: edit without a branch change, post, discard and reverse run no branch-active check (RCT-D83). `ReceiptReviewIT`, `ReceiptPostIT`, `ReceiptDiscardReverseIT` |
| RCT-E18 | finance parser: `TOO_MANY`; the web stops at 100 lines |
| RCT-E19 | finance `/document`: 404 M08; the web hides the link |
| ~~RCT-E20~~ | withdrawn in v4.0.0; replaced by RCT-E29 |
| RCT-E21 | finance `create`: the file is deleted after a failure, then M19 |
| RCT-E22 | finance: scope from `/auth/me` on each request |
| RCT-E23 | finance `DuplicateCheck`: fields only |
| RCT-E24 | **(changed)** finance `DuplicateCheck` plus the advisory lock; web: the retry calls AI extract again (RCT-R66) |
| RCT-E25 | finance post: M24 before the ledger call |
| RCT-E26 | finance: `LedgerService.record` → M23 |
| RCT-E27 | finance parser: M03, nothing stored |
| RCT-E28 | finance: redaction and `OTHER_BRANCH`, also for a match with no branch (RCT-D82); web M27 |
| RCT-E29 | finance: row locks; a deleted row gives 404 M08. `ReceiptConcurrencyIT` |
| RCT-E30 | finance reverse: `requireOpenPeriod(account, today)` → M23 |
| RCT-E31 | finance summary: no non-USD duplicate means no rate read |
| RCT-E32 | finance reverse: M30 unless `POSTED`; a race gives M30 or M26; the web reloads |
| RCT-E33 | finance summary: null `possible_duplicate.usd_total`; web "—" |
| RCT-E34 | finance reverse: no active checks |
| RCT-E35 | **new** · web `uploadReceiptAction`: a PDF by bytes skips AI, `extraction` null; finance drops any `extraction` for a PDF; web shows no "couldn't read" card. `ReceiptCreateIT`, live e2e |
| RCT-E36 | **new** · finance `ReceiptController` handler: a non-UUID `receiptId` → 404 M08 on every `{id}` endpoint (`ReceiptScopeIT`); web route: not-found screen without an API call |
| RCT-E37 | **new** · finance `PagedQuery`: an empty page with real totals, `page` < 0 → 0 (`ReceiptListIT`); web `pageToShow` moves to the last page |
| RCT-E38 | **new** · finance `BranchScope.requireAccessAnyState`: a branch not in `getBusinessBranches` → 403 M22. `ReceiptListIT` |
| RCT-E39 | **new** · finance `ReceiptServiceImpl.update`: the RCT-R59 check runs only when the branch changes. `ReceiptReviewIT` |
| RCT-E40 | **new** · finance `ReceiptFiles.detectType`: the bytes decide the type and the stored `content_type` (`ReceiptFilesTest`); web `fileKind`: the bytes decide AI or not |

## 9. Packages and verification (changed)

- **Packages:** none in either repo.

**finance-service:**

- Run `./gradlew build`. Test names start with `RCT-…`; no test names a withdrawn ID (RCT-R05, R15, R34, R39, E13).
- **api-reviewer** checks:
  - every code and status in §3;
  - **(changed)** the RCT-D84 save order on create and PUT;
  - **(changed)** `BRANCH_INACTIVE` only on create and branch-changing PUT;
  - **(changed)** `isActiveBranch` for the state and `getBusinessBranches` for scope and names;
  - **(changed)** `requireAccess` / `branchNames` untouched for other features;
  - **(changed)** the byte detection and that the declared type is ignored;
  - **(changed)** trimming;
  - **(changed)** no map-shaped 400 from any receipt endpoint;
  - **(changed)** a non-UUID id → 404;
  - the post check order, the reverse order, the redaction, the RCT-R50 regex;
  - the file never stored before validation;
  - no change to `LedgerService` or the attachments endpoint.
- **api-qa** re-runs the failed R38-V5 and R31-C11, and adds:
  - a turned-off branch: as owner (sent, and from the account) and as manager; post, discard and reverse on it;
  - a PDF declared `image/jpeg`, and a text file named `.jpg`;
  - `receipt_date` 1999-12-31;
  - vendor "  " and "  Shop  ";
  - `status=BOGUS`, `page=x`, `page=-1`, `page=999`;
  - an owner's unknown `branchId`;
  - `/receipts/not-a-uuid` on every `{id}` endpoint.

**web-ui:**

- Run `pnpm lint`, `pnpm build`, `pnpm test:logic`, then `GATEWAY_URL=http://localhost:8080 pnpm test:live`.
- **web-reviewer** checks:
  - the byte check and no AI call for a PDF;
  - the leave prompt on upload;
  - a fresh AI call on retry;
  - M31 / M32 on the right field;
  - the manager's branch name;
  - the non-UUID route;
  - the last-page move;
  - turned-off branches in the owner's filter only;
  - no "couldn't read" card for a PDF.
- **web-qa** walks §4 as owner (All branches, one branch, a turned-off branch) and as manager, at desktop and phone width. It includes a retried upload (the AI service log shows a second extract call).

## 10. Questions and conflicts

- **Q1 · Base branches** → answered 2026-11-16 by Vuththana:
  - v4 / v5 work branched `feature/receipts` from `origin/dev-cleaned`, with PRs into `dev-cleaned` (web-ui has no `dev`).
  - For v6, the lead set `feature/receipts-v6` from `origin/dev-cleaned`, with PRs into `dev-cleaned` (2026-11-17).

## Changes

| Date | Change | Why | Approved by |
|---|---|---|---|
| 2026-11-16 | First plan, against spec v4.0.0. It includes:<br>• the receipt guard in the transactions reverse / PATCH / categorize endpoints (RCT-R47, R48, R52, R53)<br>• the duplicate pick order (RCT-R49)<br>• the exact-amount search (RCT-R50)<br>• the reversal dated today via `refund` (RCT-R51, RCT-E30)<br>• RCT-E29 replacing RCT-E20 | Gate 1; the planner's first-draft questions Q2–Q6 were settled by /sdd:amend (RCT-D38–D45) | Vuththana |
| 2026-11-16 | Updated for spec v5.0.0. Replacements: RCT-R56 for R15 (no row for a no-change save; status or match change logged), R55 for R34 (reference on `REVERSED`), R54 for R39 (new `409 RECEIPT_NOT_REVERSIBLE`, RCT-M30, the D61 order with `requireOpenPeriod` before `refund`), E32 for E13. New: RCT-R57 and R58 (tests only), E31 and E33 (summary reads the rate only for non-USD duplicates; null total, no 503), E34 (no active check at reverse), M29. Also: PUT body checks before the lookup (D54); skip only from `NEEDS_REVIEW` (D66); M04 / M09 / M18 texts (D46, D48); `REQUIRED` problem code (D53); `branch_name` for both roles (D60); history redaction of duplicate ids. §7 now lists the files already in the working trees (`ReceiptHistory`, `ReceiptMapper`, `ReceiptPosting`, `CurrentUserProvider`, test support, web `receipt-*` helpers, `receipt-normalizers.ts`, `receipt-duplicate-panel.tsx` instead of rewriting the bills' `duplicate-panel.tsx`) | /sdd:amend v5.0.0 after /sdd:verify on the finance build | Vuththana |
| 2026-11-17 | Updated for spec v6.0.0 (parts I–N on `feature/receipts-v6`). Replacements: RCT-R60 for R05 (trimmed `vendor_name` / `receipt_number`). New:<br>• RCT-R59: `422 BRANCH_INACTIVE`, M31 / M32, on create and branch-changing PUT only, via `IdentityGateway.isActiveBranch`<br>• RCT-R61: file type from the bytes<br>• RCT-R62: `TOO_EARLY`<br>• RCT-R63: the M03 `INVALID` list for malformed queries and wrong-type fields (controller handler; reverse body read as JSON)<br>• RCT-R64–R66 (web: manager's branch name, upload leave prompt, AI re-called on retry)<br>• RCT-E35–E40<br>Also:<br>• the RCT-D84 save order (M11 now after the 404)<br>• the owner's list and summary filter by a turned-off branch via new `IdentityGateway.getBusinessBranches` (`IdentityDatabase`) and `BranchScope.requireAccessAnyState` / `allBranchNames`<br>• no reading stored for a PDF<br>• non-UUID id → 404<br>The old v5 "(changed)" marks are cleared; only v6 changes are marked | /sdd:amend v6.0.0 after the second verify round and QA (R38-V5, R31-C11) | Vuththana |