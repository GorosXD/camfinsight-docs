# sales-service

Endpoints used by [mobile-sales-management](../specs/mobile-sales-management/spec.md). They exist in code on `dev`; shapes are taken from the code (sales-service `SaleController`, `ItemController` and their DTOs) until the data dictionary has the tables. JSON is snake_case; query parameters use their Java names (camelCase). Timestamps without an offset are UTC. Money is `numeric`.

| Status | Method | Path | Who | Spec |
|---|---|---|---|---|
| 📝 | GET | /api/v1/sales/summary | Owner, Manager | MSL-R01, MSL-R06 |
| 📝 | GET | /api/v1/sales | Owner, Manager | MSL-R02 … MSL-R08 |
| 📝 | GET | /api/v1/sales/{saleId} | Owner, Manager | MSL-R28 … MSL-R30 |
| 📝 | POST | /api/v1/sales | Owner, Manager | MSL-R25 … MSL-R27, MSL-R40, MSL-R45 |
| 📝 | POST | /api/v1/sales/{saleId}/reverse | Owner, Manager | MSL-R31 … MSL-R35, MSL-R46 |
| 📝 | GET | /api/v1/items | Owner, Manager | MSL-R44 |
| 📝 | GET | /api/v1/items/{itemId} | Owner, Manager | MSL-R44, MSL-R15, MSL-R17, MSL-R18, MSL-R43 |

📝 here means "exists in code, entry written from the code"; it becomes ✅ when the team confirms it against the dictionary.

A `BRANCH_MANAGER` is always scoped to their own branch (`BranchScope`): reads outside it answer `404`, writes to another branch `403 BRANCH_ACCESS_DENIED`, writes to an inactive branch `409 BRANCH_INACTIVE`.

### GET /api/v1/sales/summary

Query: `date` (`yyyy-MM-dd`, Asia/Phnom_Penh day; default today), `branchId` (owner only; omitted = all branches).

`200 OK`, `data`: `{ date, compared_with, total, total_change_percent, count, count_change_percent, average_sale, average_sale_change_percent, khqr_share_percent, khqr_share_change_points }` — completed `WALK_IN` sales only, in USD, compared with the same weekday last week. A `*_change_*` is `null` when last week had nothing.

| Status | `code` | When |
|---|---|---|
| 403 | `BRANCH_ACCESS_DENIED` | A manager asks for another branch (MSL-M11) |

### GET /api/v1/sales

Query: `search` (sale number or item name), `from`, `to` (`yyyy-MM-dd` local dates, inclusive), `paymentMethod` (`CASH` `KHQR` `ABA` `WING` `CARD` `OTHER`), `source`, `financialAccountId`, `status` (`COMPLETED` `REVERSED`), `branchId` (owner only), `page` (0-based), `size` (default 20), `sort` (`sale_date`, `sale_reference`, `total_amount`; default `sale_date` newest first).

`200 OK`, `data`: `{ items: [row], page, size, total_elements, total_pages }`; `row` = `{ sale_id, sale_date, sale_reference, items_summary, payment_method, financial_account_id, account_name, branch_id, branch_name, total_amount, currency, source, status }`. `source` ∈ `WALK_IN` `POS` `SHOPIFY` `WEBILL365` `FILE_IMPORT`.

| Status | `code` | When |
|---|---|---|
| 403 | `BRANCH_ACCESS_DENIED` | A manager asks for another branch (MSL-M11) |

### GET /api/v1/sales/{saleId}

`200 OK`, `data` = **sale detail**: `{ sale_id, sale_reference, source, sale_date, status, currency, subtotal, discount_amount, tax_amount, shipping_amount, total_amount, payment_method, financial_account_id, account_name, branch_id, branch_name, transaction_id, recorded_by_user_id, recorded_by_name, reversed_at, reverse_reason, reversed_by_user_id, reversed_by_name, created_at, items: [{ sale_item_id, item_id, item_name, sku, quantity_sold, unit_price, unit_cost, line_total }] }`.

| Status | `code` | When |
|---|---|---|
| 404 | `NOT_FOUND` | "Sale not found" — unknown, or outside the caller's scope (MSL-M12) |

### POST /api/v1/sales

Body: `branch_id` (required), `sale_date` (optional, ISO-8601 with offset, default now), `source` (`WALK_IN` default, or `POS`), `currency` (required, `USD`/`KHR`), `items` (required, not empty): `[{ item_id, quantity, unit_price }]` — `quantity` > 0, ≤ 9 digits and 3 decimals; `unit_price` optional (default the item's selling price, converted for KHR), ≥ 0, ≤ 13 digits and 2 decimals — `discount_amount`, `tax_amount` (≥ 0, ≤ 13 digits and 2 decimals), `payment_method` (required), `financial_account_id` (required). The request line field is `quantity`; the response field is `quantity_sold`.

`201 CREATED`, message "Sale recorded", `data`: the **sale detail**. The server computes subtotal, total, `sale_reference` (WS-{yyyy}-{seq}), decreases stock (SALE movements) and has finance-service record the income transaction.

| Status | `code` | When |
|---|---|---|
| 400 | `BAD_REQUEST` | `source` is not `WALK_IN` or `POS` |
| 400 | `VALIDATION_FAILED` | A field rule above (field errors in `data`) |
| 403 | `BRANCH_ACCESS_DENIED` | A manager writes another branch (MSL-M11) |
| 409 | `BRANCH_INACTIVE` | "This branch is inactive. Choose an active branch" (MSL-M07) |
| 409 | `PERIOD_LOCKED` | Finance refuses income dated in a signed-off period (MSL-M17) |
| 422 | `INSUFFICIENT_STOCK` | "Not enough stock of '{item}' (available {n})" (MSL-M02) |
| 422 | `DECIMALS_NOT_ALLOWED` | "'{item}' is counted in whole numbers only" (MSL-M03) |
| 422 | `INVALID_QUANTITY` | "Quantities have at most 3 decimals" (MSL-M20) |
| 422 | `INVALID_AMOUNT` | Discount above subtotal, negative amounts, total too large (MSL-M04) |
| 422 | `ITEM_INACTIVE` | "'{item}' is inactive and can't be sold" (MSL-M10) |
| 422 | `FINANCE_REJECTED` | Finance refused the income transaction (MSL-M11) |
| 503 | `DEPENDENCY_UNAVAILABLE` | Finance or identity service unavailable (MSL-M09) |

No idempotency key (MSL-D16): a retried request records a second sale.

### POST /api/v1/sales/{saleId}/reverse

Body: `reason` (required, not blank). `200 OK`, message "Sale reversed", `data`: the **sale detail** with `status` `REVERSED`. Stock is restored (REFUND movements) and finance posts a reversing entry.

| Status | `code` | When |
|---|---|---|
| 400 | `VALIDATION_FAILED` | "reason is required" |
| 403 | `BRANCH_ACCESS_DENIED` | (MSL-M11) |
| 404 | `NOT_FOUND` | "Sale not found" (MSL-M12) |
| 409 | `SALE_ALREADY_REVERSED` | "This sale was already reversed" (MSL-M16) |
| 409 | `PERIOD_LOCKED` | (MSL-M17) |
| 422 | `FINANCE_REJECTED` | (MSL-M11) |
| 503 | `DEPENDENCY_UNAVAILABLE` | (MSL-M09) |

### GET /api/v1/items

Query: `search` (name or SKU), `status` (`ACTIVE` …), `branchId` (scopes `total_stock` and `stock_status` to that branch), `page` (0-based).

`200 OK`, `data`: `{ items: [row], page, size, total_elements, total_pages }`; `row` includes `item_id`, `item_name`, `sku` (may be `null`), `unit_name`, `selling_price` (USD), `total_stock`, `stock_status` (`IN_STOCK` `LOW` `OUT`).

| Status | `code` | When |
|---|---|---|
| 403 | `BRANCH_ACCESS_DENIED` | A manager passes another branch |

### GET /api/v1/items/{itemId}

`200 OK`, `data`: `{ item: { item_id, item_name, sku, unit_name, cost_price, selling_price, allows_decimals, is_active, … }, kpis, stock_by_branch: [{ branch_id, branch_name, quantity, stock_status }] }`. Prices are USD. `stock_by_branch` lists every branch the caller may read (a manager: only their own), with `quantity` 0 where there is no stock row. Returns inactive items too.

| Status | `code` | When |
|---|---|---|
| 404 | `NOT_FOUND` | Unknown item |
