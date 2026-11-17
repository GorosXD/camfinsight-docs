# Finance (finance-service, schema `finance`)

Tables written only by finance-service (TRD §1a). The finance tables that existed before this file (`financial_accounts`,
`business_categories`, `transactions`, `bank_transactions`, `change_logs` …) are not written up here yet; specs name them by
table and field as built.

## receipts

A paper or PDF receipt for an expense, saved for review and then posted as one `EXPENSE` transaction
([receipts spec](../specs/receipts/spec.md)). History rows go in `change_logs` with `record_type` `RECEIPT`.

| Field Name | Datatype | Length | Required | PK/FK | Default | Reference | Description |
|---|---|---|---|---|---|---|---|
| `receipt_id` | uuid |  | Y | PK |  |  | auto generated |
| `receipt_code` | varchar | 20 | Y |  |  |  | `RCP-{yyyy}-{seq}`, `{seq}` at least 4 digits; a per-business yearly counter by the year of creation in Asia/Phnom_Penh; unique per business |
| `business_id` | uuid |  | Y |  |  |  | identity-service business |
| `branch_id` | uuid |  |  |  |  |  | null until an account or branch is set; the account's branch when it has one; required to post |
| `receipt_number` | varchar | 50 |  |  |  |  | the number printed on the receipt |
| `vendor_name` | varchar | 100 |  |  |  |  | required to post |
| `receipt_date` | date |  |  |  |  |  | required to post; not after today in Asia/Phnom_Penh |
| `business_category_id` | uuid |  |  | FK |  | business_categories | an active `EXPENSE` category; required to post |
| `financial_account_id` | uuid |  |  | FK |  | financial_accounts | the account that paid, active, same currency as the receipt; required to post |
| `subtotal` | numeric(15,2) |  |  |  |  |  | as printed; 0 or more |
| `tax_rate` | numeric(5,2) |  |  |  |  |  | percent, 0 to 100; shown only, nothing is computed from it |
| `tax_amount` | numeric(15,2) |  |  |  |  |  | as printed; 0 or more |
| `original_amount` | numeric(15,2) |  |  |  |  |  | the total; more than 0 to post; no decimals for KHR |
| `original_currency` | enum |  |  |  |  |  | USD<br>KHR<br>required to post |
| `exchange_rate` | numeric(10,2) |  |  |  |  |  | null until posted, then copied from the transaction |
| `usd_equivalent` | numeric(15,2) |  |  |  |  |  | null until posted, then copied from the transaction |
| `status` | enum |  | Y |  | `NEEDS_REVIEW` |  | NEEDS_REVIEW = saved, not posted<br>POSSIBLE_DUPLICATE = matches a saved expense; can't post until resolved<br>POSTED = posted as `transaction_id`; read-only<br>REVERSED = its transaction was reversed; read-only |
| `duplicate_of_transaction_id` | uuid |  |  | FK |  | transactions | the matching transaction when `POSSIBLE_DUPLICATE` |
| `duplicate_of_receipt_id` | uuid |  |  | FK |  | receipts | the matching unposted receipt when `POSSIBLE_DUPLICATE`; set null when that receipt is discarded |
| `transaction_id` | uuid |  |  | FK |  | transactions | set when posted; unique |
| `reversal_reason` | varchar | 500 |  |  |  |  | required to reverse |
| `document_id` | uuid |  |  | FK |  | [documents](#documents) | the file; null for a typed-in receipt |
| `created_by_user_id` | uuid |  | Y |  |  |  | identity-service user |
| `created_at` | timestamp |  | Y |  | `current_timestamp` |  |  |
| `updated_at` | timestamp |  | Y |  | `current_timestamp` |  |  |

Indexes: unique (`business_id`, `receipt_code`); unique `transaction_id`; (`business_id`, `receipt_date`, `original_amount`)
for the duplicate check; (`branch_id`, `status`). The yearly counter is kept per (`business_id`, `RCP`, year), like
sales-service's `document_sequences`; see [document_sequences](#document_sequences).

## receipt_lines

| Field Name | Datatype | Length | Required | PK/FK | Default | Reference | Description |
|---|---|---|---|---|---|---|---|
| `receipt_line_id` | uuid |  | Y | PK |  |  | auto generated |
| `receipt_id` | uuid |  | Y | FK |  | [receipts](#receipts) | deleted with the receipt |
| `description` | varchar | 200 |  |  |  |  |  |
| `quantity` | numeric(15,3) |  |  |  |  |  | 0 or more |
| `unit_price` | numeric(15,2) |  |  |  |  |  | 0 or more |
| `amount` | numeric(15,2) |  |  |  |  |  | as printed; not checked against quantity × unit_price |
| `line_order` | integer |  | Y |  |  |  | 1, 2, 3 … in the order sent; at most 100 lines per receipt |

## documents

An uploaded file and its AI reading. Today only receipts use it.

| Field Name | Datatype | Length | Required | PK/FK | Default | Reference | Description |
|---|---|---|---|---|---|---|---|
| `document_id` | uuid |  | Y | PK |  |  | auto generated |
| `business_id` | uuid |  | Y |  |  |  | identity-service business |
| `file_name` | varchar | 255 | Y |  |  |  | as uploaded |
| `content_type` | varchar | 100 | Y |  |  |  | image/jpeg, image/png, image/webp or application/pdf |
| `size_bytes` | bigint |  | Y |  |  |  | at most 10 485 760 (10 MB) |
| `storage_key` | varchar | 500 | Y |  |  |  | RustFS (S3) key, private to the business |
| `extraction` | jsonb |  |  |  |  |  | the AI reading as received, at most 256 KB; see [jsonb-shapes.md](jsonb-shapes.md#documentsextraction) |
| `created_at` | timestamp |  | Y |  | `current_timestamp` |  |  |

## document_sequences

A per-business yearly counter for document codes. Today only receipts use it (`doc_type` `RCP`, giving
`receipt_code` `RCP-{yyyy}-{seq}`), the same shape as sales-service's `document_sequences`. It is advanced with
`INSERT … ON CONFLICT DO UPDATE … RETURNING`, so concurrent callers are serialised by the row lock.

| Field Name | Datatype | Length | Required | PK/FK | Default | Reference | Description |
|---|---|---|---|---|---|---|---|
| `business_id` | uuid |  | Y | PK |  |  | identity-service business |
| `doc_type` | varchar | 10 | Y | PK |  |  | the document kind: `RCP` (receipts) |
| `seq_year` | integer |  | Y | PK |  |  | the year in Asia/Phnom_Penh the code is created |
| `last_value` | bigint |  | Y |  |  |  | the last number handed out; the next code is `last_value + 1`, padded to at least 4 digits |

## change_logs (receipts)

`change_logs` is not fully written up yet (see the top of this file). Receipts write it with `record_type` `RECEIPT` and
`record_id` = `receipt_id`; `previous_value` and `new_value` are text:

| `action` | `previous_value` | `new_value` |
|---|---|---|
| `CREATED` | null | JSON object of the saved fields and lines (never the AI reading) |
| `UPDATED` | JSON object of the changed fields, before | JSON object of the changed fields, after; also `status` and `duplicate_of_transaction_id` / `duplicate_of_receipt_id` when they changed. A save that changes nothing writes no row (RCT-R56) |
| `NOT_DUPLICATE` | null | JSON object `{ vendor_name, receipt_date, original_amount, original_currency }`, the snapshot the duplicate re-check compares with (RCT-D59) |
| `POSTED` | null | the posted transaction's id |
| `REVERSED` | null | the reversal reason |
| `DISCARDED` | JSON object `{ receipt_code, vendor_name, original_amount, original_currency }` of the removed receipt | null |
