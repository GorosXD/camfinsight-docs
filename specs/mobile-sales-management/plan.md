# Plan: Mobile sales management

**Spec:** `MSL` v1.0.0, commit `20f73bd` · **Approved:** 2026-11-12 by Vuththana
**Repos:** ledgervision-identity-service (backend: new exchange-rate endpoint), ledgervision-android-ui (Android app: the whole feature)

## 1. Scope

- **In:** all of spec §4a–§4d and §5: MSL-R01 … MSL-R12, MSL-R14 … MSL-R49, MSL-E01 … MSL-E09, MSL-E11 … MSL-E31, decisions MSL-D01 … MSL-D19, MSL-D21 … MSL-D26. One new backend endpoint (MSL-D15); everything else calls endpoints that exist on `dev` today, unchanged.
- **Out:** spec §1 "Out" list (invoices, customers, payments, overview, offline queue, barcode, receipt number, printing, Khmer, idempotency key, server-side future-date rule). No change to sales-service or finance-service (read-only references). No migrations in any repo. `GET /api/v1/items/stocks` is not used (MSL-D24). Writing the `sales`, `sale_items`, `items`, `item_stocks`, `businesses.exchange_rate` tables into `data-dictionary/` (spec §8, "when the team writes the dictionary"). Removing the starter's `note` sample (it stays in the code, just no longer the home screen).
- **Source of field names:** `data-dictionary/` is empty and `api/` lists only identity-service, so every JSON name, query parameter and DTO shape below is **taken from the service code on `dev`** (sales-service `SaleController`, `SaleRequest`, `SaleLineRequest`, `SaleDetail`, `SaleListRow`, `SaleSummary`, `PageResponse`, `ItemController`, `ItemListRow`, `ItemDetail`/`ItemResponse`, `ErrorCode`, `BranchScope`; finance-service `AccountController`/`AccountResponse`; identity-service `BranchController`/`BranchResponse`, `MeDto` in the app). All three services serialise JSON in snake_case (`JacksonConfig`); query parameters are bound by their Java names (camelCase: `branchId`, `paymentMethod`), as web-ui already sends them.

## 2. Repos and parts

| Repo | Role (agent) | Part | Order |
|---|---|---|---|
| ledgervision-identity-service | api-dev | `GET /api/v1/businesses/me/exchange-rate` (controller, service method, response DTO, message, integration test). No migration, no gateway or security change (`/api/v1/businesses/**` is already routed; the route needs only an authenticated, onboarded user). MSL-R15, MSL-R19, MSL-R43 | First, but only the **contract** (§3) gates anyone. Branch `feature/mobile-sales-management` from `origin/dev`. Note: on `origin/dev` the root package is still `com.ledgervision.identity` (the local `rename-camfinsight` branch, not merged, renames it to `com.camfinsight.identity`); paths in §7 are written from `…/identity/` and use whatever `origin/dev` has when the branch is cut. |
| ledgervision-android-ui | android-dev | Everything else: session gets the manager's `branch_id`; typed destinations; bottom navigation (Sales, Profile); `sales` feature slice (data → domain → presentation) for the list + KPI card, record sale, sale detail, reverse | **Parallel** with identity-service. It builds against the existing sales/finance/identity endpoints directly; the exchange-rate call is written to the §3 contract and unit-tested with fakes. Device QA of KHR prefill, "≈" and below-cost-in-KHR needs the identity branch running locally (or merged into `dev`). Branch `feature/mobile-sales-management` from `origin/dev`. Internal order: (1) core + auth changes, (2) sales data/domain + tests, (3) screens, (4) `MainGraph` wiring |
| camfinsight-docs (lead) | lead | `api/identity-service.md` status 📝 → 🚧 → ✅ and its error rows corrected (§3.1); new `api/sales-service.md` and `api/finance-service.md` entries for the endpoints in §3.2–§3.10 (TEAM-RULES §7 "every endpoint is listed in `api/`"), on `feature/mobile-sales-management` | After the identity endpoint is built (status); the sales/finance entries can be written any time from §3 |

## 3. API contract

The contract both repos build against. §3.1 is from `api/identity-service.md` (📝 planned) and is **new**. §3.2–§3.10 exist in code and are not yet in `api/`; their shapes are from the code on `dev`. Paths are relative to the gateway base URL (`API_BASE_URL`; the app writes `api/v1/...` with no leading slash, like `AuthApi`). Every answer uses the envelope `{ success, code, message, data, timestamp }` (`BaseResponse`). Timestamps without an offset (`sale_date`, `reversed_at`, `created_at`) are UTC (MSL-D19, MSL-R39). Money is `numeric` on the server and `BigDecimal` in the app (never `Double`).

### 3.1 `GET /api/v1/businesses/me/exchange-rate` — **new** · MSL-R15, MSL-R19, MSL-R43, MSL-E18, MSL-E23, MSL-E26, MSL-E27

- **Who:** Owner and Manager. The business is the caller's own (`CurrentUser.businessId`, from the `users` row); no parameters, nothing is trusted from the request. No owner check (unlike `GET /businesses/me`).
- **200 OK** `message`: `business.exchange_rate.success` ("The business exchange rate."), `data`: `{ "exchange_rate": <integer> }` — `businesses.exchange_rate`, KHR per 1 USD (1–100000).
- **Errors:** `401` (no or bad token; `ACCOUNT_DEACTIVATED`; `SESSION_REVOKED` — existing `CurrentUserFilter`), `403 BUSINESS_DELETED` (existing filter), `403 ONBOARDING_INCOMPLETE` (existing filter, before the endpoint: an owner with no business, or a manager who hasn't changed the temporary password yet), `404 NOT_FOUND` (business row missing — only reachable if the row is gone). The app maps any failure to MSL-E26 / MSL-E27; none has its own message.
- **Docs fix:** `api/identity-service.md` says `404 NOT_FOUND` for "the caller has no business yet"; in code the filter answers `403 ONBOARDING_INCOMPLETE` first. The docs part (§2) rewrites the rows as above.

### 3.2 `GET /api/v1/sales/summary` — existing · MSL-R01, MSL-R06, MSL-R07, MSL-E13

- **Who:** Owner, Manager (a manager is always scoped to their branch by sales-service).
- **Query:** `date` = today in Asia/Phnom_Penh, `yyyy-MM-dd` (sent explicitly); `branchId` = the owner's branch filter, omitted for "All branches" and always omitted for a manager.
- **200 OK** `data`: `{ date, compared_with, total, total_change_percent, count, count_change_percent, average_sale, average_sale_change_percent, khqr_share_percent, khqr_share_change_points }` — completed `WALK_IN` sales only, in USD (MSL-D18). A `*_change_*` is `null` when last week had nothing.
- **Errors:** `403 BRANCH_ACCESS_DENIED` (MSL-M11; not expected, the owner only picks listed branches).

### 3.3 `GET /api/v1/sales` — existing · MSL-R02, MSL-R03, MSL-R04, MSL-R05, MSL-R06, MSL-R07, MSL-R08, MSL-E17, MSL-E21

- **Who:** Owner, Manager.
- **Query:** `search` (sale number or item name), `from`, `to` (`yyyy-MM-dd` local dates, inclusive), `paymentMethod` (`CASH` `KHQR` `ABA` `WING` `CARD` `OTHER`), `status` (`COMPLETED` `REVERSED`), `branchId` (owner only), `page` (0-based). `size` and `sort` are omitted (server default 20, `sale_date` newest first). Not sent: `source`, `financialAccountId`.
- **200 OK** `data`: `{ items: [row], page, size, total_elements, total_pages }`; `row` = `{ sale_id, sale_date, sale_reference, items_summary, payment_method, financial_account_id, account_name, branch_id, branch_name, total_amount, currency, source, status }`. `source` ∈ `WALK_IN` `POS` `SHOPIFY` `WEBILL365` `FILE_IMPORT`.
- **Errors:** `403 BRANCH_ACCESS_DENIED` (MSL-M11, not expected).

### 3.4 `GET /api/v1/sales/{saleId}` — existing · MSL-R28, MSL-R29, MSL-R30, MSL-E03, MSL-E17

- **Who:** Owner, Manager. A sale outside the caller's branch scope answers `404` (sales-service `requireVisible`).
- **200 OK** `data` = **sale detail**: `{ sale_id, sale_reference, source, sale_date, status, currency, subtotal, discount_amount, tax_amount, shipping_amount, total_amount, payment_method, financial_account_id, account_name, branch_id, branch_name, transaction_id, recorded_by_user_id, recorded_by_name, reversed_at, reverse_reason, reversed_by_user_id, reversed_by_name, created_at, items: [{ sale_item_id, item_id, item_name, sku, quantity_sold, unit_price, unit_cost, line_total }] }`.
- **Errors:** `404 NOT_FOUND` "Sale not found" (MSL-M12).

### 3.5 `POST /api/v1/sales` — existing · MSL-R10, MSL-R11, MSL-R12, MSL-R23, MSL-R25, MSL-R26, MSL-R27, MSL-R40, MSL-R45, MSL-E01, MSL-E02, MSL-E04, MSL-E05, MSL-E07, MSL-E08, MSL-E14, MSL-E22, MSL-E23, MSL-E29

- **Who:** Owner (any active branch of the business), Manager (own branch only).
- **Body** (`SaleRequest`): `branch_id` (`sales.branch_id`), `sale_date` (`sales.sale_date`, ISO-8601 with `+07:00`, e.g. `2026-11-12T10:15:00+07:00`), `source` = `"WALK_IN"` (`sales.source`, MSL-D12), `currency` (`sales.currency`, `USD`/`KHR`), `items`: `[{ item_id, quantity, unit_price }]` (`sale_items.item_id`, `sale_items.quantity_sold` — **the request field is `quantity`**, the response field is `quantity_sold`; `sale_items.unit_price`, always sent as shown on the line), `discount_amount` (`sales.discount_amount`, `0` when empty), `tax_amount` (`sales.tax_amount`, `0` when empty), `payment_method` (`sales.payment_method`), `financial_account_id` (`sales.financial_account_id`). Numbers are JSON numbers from `BigDecimal` (plain string form, no exponent).
- **201 CREATED** `message` "Sale recorded", `data` = sale detail (§3.4).
- **Errors** (the app shows the server `message` unless a client text is given):

| Status · code | App reaction | Message |
|---|---|---|
| `422 INSUFFICIENT_STOCK` | form kept, banner; every line's stock re-read with `GET /items/{id}` | MSL-M02 (server text) |
| `422 DECIMALS_NOT_ALLOWED` | form kept, banner | MSL-M03 (server text) |
| `422 INVALID_AMOUNT` | form kept, banner | MSL-M04 (server text) |
| `409 BRANCH_INACTIVE` | owner: branch picker reloads, branch cleared if no longer listed; manager: banner with only "Back", save disabled | MSL-M07 |
| `503 DEPENDENCY_UNAVAILABLE` | form kept, banner with Retry | MSL-M09 (server text) |
| `422 ITEM_INACTIVE` | form kept, banner naming the item | MSL-M10 (server text) |
| `422 FINANCE_REJECTED`, `403 BRANCH_ACCESS_DENIED` | form kept, banner | MSL-M11 (server text) |
| `409 PERIOD_LOCKED` | form kept, error under the sale-date field | MSL-M17 (server text), MSL-D21 |
| `422 INVALID_QUANTITY` | form kept, banner (not expected) | MSL-M20 (server text) |
| `400 VALIDATION_FAILED`, `404 NOT_FOUND`, `400 BAD_REQUEST`, any other code | form kept, banner (not expected; client checks first) | server text |
| **not sent**: failed before a connection was made (no network, DNS failure, connection refused, connect timeout, TLS handshake failure) | form kept, banner with Retry | MSL-M08 (MSL-D25) |
| **no answer**: failed after the connection was made (read timeout, connection reset / closed mid-call, unreadable `2xx` body) | form kept, dialog with "Check sales" and "Try again" | MSL-M19 |
| `401` | existing refresh-once; if refresh is rejected, sign-in (MSL-E15) | — |

- **How the app tells "not sent" from "no answer":** a connect timeout and a read timeout are both `SocketTimeoutException`, so the type alone can't decide. `SaleWriteApi.recordSale` carries a Retrofit `@Tag SendAttempt` (a small object with a `connected` flag); the write client has a **network interceptor** that sets `connected = true` (network interceptors run only once a connection is established). A transport failure with `connected == false` is "not sent"; with `connected == true` it is "no answer".

### 3.6 `POST /api/v1/sales/{saleId}/reverse` — existing · MSL-R31, MSL-R32, MSL-R33, MSL-R34, MSL-R35, MSL-R46, MSL-E06, MSL-E17, MSL-E24, MSL-E30

- **Who:** Owner, Manager, any sale they can see (MSL-D03), any `source`.
- **Body:** `{ "reason": <text> }` (`sales.reverse_reason`, trimmed, not blank — checked in the app first, MSL-M14).
- **200 OK** `message` "Sale reversed", `data` = sale detail (§3.4) with `status` `REVERSED`.
- **Errors:** `409 SALE_ALREADY_REVERSED` (MSL-M16, then reload the detail), `409 PERIOD_LOCKED` (MSL-M17), `503 DEPENDENCY_UNAVAILABLE` (MSL-M09, Retry), `422 FINANCE_REJECTED` and `403 BRANCH_ACCESS_DENIED` (MSL-M11, Retry), `404 NOT_FOUND` (MSL-M12), `400 VALIDATION_FAILED` (MSL-M14; not expected). **Any connection failure** (before sending or no answer): MSL-M08 with Retry, the dialog keeps the reason, the sale stays as last loaded (MSL-R46); a retry of a reverse that had gone through answers `409` → MSL-M16 and the reversed sale (MSL-E30).

### 3.7 `GET /api/v1/items` — existing · MSL-R44, MSL-E16, MSL-E20, MSL-E25, MSL-E31

- **Query:** `search` (name or SKU; empty when the picker opens), `status=ACTIVE`, `branchId` = the sale branch (so `total_stock` and `stock_status` are that branch's), `page` (0-based, next page on scroll).
- **200 OK** `data`: `{ items: [row], page, size, total_elements, total_pages }`; fields used from `row`: `item_id`, `item_name`, `sku` (may be `null`), `unit_name`, `selling_price`, `total_stock`, `stock_status` (`IN_STOCK` `LOW` `OUT`).

### 3.8 `GET /api/v1/items/{itemId}` — existing · MSL-R44, MSL-R15, MSL-R17, MSL-R18, MSL-R43, MSL-E05, MSL-E20, MSL-E31

- **200 OK** `data`: `{ item: { item_id, item_name, sku, unit_name, cost_price, selling_price, allows_decimals, is_active, … }, kpis, stock_by_branch: [{ branch_id, branch_name, quantity, stock_status }] }`. `selling_price` and `cost_price` are USD. The line's stock is the `quantity` of the `stock_by_branch` entry whose `branch_id` is the sale branch; no entry → 0 (MSL-D24). `stock_by_branch` lists every branch the caller may read (`BranchScope.readable`), so a manager sees only their own. `kpis` is not used.
- **Errors:** `404 NOT_FOUND` (item gone: server text).

### 3.9 `GET /api/v1/bank-statements/accounts` (finance-service) — existing · MSL-R22, MSL-E09

- **Query:** `branchId` = the sale branch, `status=ACTIVE`.
- **200 OK** `data`: list (not paged), ordered by `account_name`, of the branch's accounts **and** the business-wide ones: `{ financial_account_id, account_name, account_type, currency, starting_balance, current_balance, usd_equivalent, is_business_wide, is_active, branch_id, branch_name, created_at, updated_at }` (balances `null` for a manager on business-wide accounts; the app doesn't use balances).
- **App rule:** keep `currency == sale currency`; preselect the first with `is_business_wide == false`, else the first business-wide one (MSL-D22); none → MSL-M05.

### 3.10 `GET /api/v1/branches` (identity-service) — existing · MSL-R06, MSL-R11, MSL-R42, MSL-E07

- **Who:** Owner only (a manager gets `403 FORBIDDEN_ROLE`; the app never calls it for a manager).
- **Query:** none (`status` defaults to active).
- **200 OK** `data`: list of `{ branch_id, branch_name, address, contact_information, is_active, is_main_branch, managers_count, created_at, updated_at }`. Used: `branch_id`, `branch_name`.

### 3.11 `GET /api/v1/auth/me` (identity-service) — existing, already called by the app at sign-in · MSL-R10, MSL-E22

- `data.branch_id` (the manager's `users.branch_id`, `null` for an owner) is now kept in the session (§6). No new call.

## 4. Screens, routes and states

Destinations are typed in `core/navigation/AppDestinations.kt`; the graph is only in `app/.../MainGraph.kt`.

| Route | Who | States (loading / empty / error / each status) | IDs |
|---|---|---|---|
| Bottom navigation bar (in `MainGraph`, shown only on the two tab roots `SalesList()` and `Profile`) | Owner, Manager | Tabs "Sales" (start tab after sign-in, business setup and a remembered session) and "Profile"; switching keeps each tab's state (`saveState`/`restoreState`); Back from Profile returns to Sales. Profile as a tab has no back arrow. | MSL-R41 |
| `SalesList(showBack: Boolean = false)` — tab root | Owner, Manager | **KPI card** (own state): loading skeleton · error M08/server text + Retry · success: title "Walk-in sales today", total (USD, MSL-R37), count, average sale, KHQR share, each with its change vs the same weekday last week as returned (a `null` change is not shown) · follows the owner's branch filter only, never the date/method/status filters (MSL-D18). **List** (own state): first-page loading skeleton · first-page error M08/server text + Retry · empty: M01 "No sales yet." when no filter or search is active, else "No sales match these filters." with "Clear filters" (clears the search too) · rows (reference, local date-time, branch name for owner only, source label, payment-method label, total with currency, status; `REVERSED` rows carry a "Reversed" badge; text max 2 lines) · loading-more footer · next-page error footer M08 + Retry with rows kept. **Search** field (debounce 400 ms). **Filter chips:** date range (Material date-range picker, local dates), payment method, status, branch (owner only: "All branches" + active branches; "All branches" is the start value and not counted as a filter); a "Clear" chip when any is set. Pull-to-refresh reloads card + page 0. FAB "Record sale". Refreshes card + page 0 whenever a sale was recorded or reversed (§7 change signal). | MSL-R01, MSL-R02, MSL-R03, MSL-R04, MSL-R05, MSL-R06, MSL-R07, MSL-R08, MSL-R09, MSL-R36, MSL-R37, MSL-R39, MSL-E13, MSL-E17, MSL-E21, MSL-E25 |
| `SalesList(showBack = true)` — pushed from RecordSale "Check sales" | Owner, Manager | Same screen with a back arrow, no bottom bar, no FAB; Back returns to the kept form | MSL-R40, MSL-E02 |
| `RecordSale` | Owner, Manager | **Loading** (owner: branches; both: exchange rate) · **error** loading branches: M08/server text + Retry · **rate failed**: inline M08 + Retry, KHR option disabled, "≈" hidden until the rate loads (MSL-E26) · **form**: branch (manager: read-only own branch name; owner: picker of active branches, preselected when exactly one, "Add item" disabled until chosen, change clears lines and account) · currency toggle USD/KHR (USD at start, as web) · lines (item name, unit, stock at the branch, quantity, unit price, line total, remove; line errors M02 / M03 / M21 / M22 / M23; warning M18; a failed item read keeps the line out and shows M08 + Retry in the picker) · discount, tax (error M04) · input filters per MSL-R49 (no minus anywhere; KHR price/discount/tax whole riel, USD ≤ 2 decimals; quantity decimals are typed freely so M23 can show) · subtotal, discount, tax, total and "≈ {other}" · payment method chips (none preselected) · "Paid into" account picker (preselected per §3.9; M05 when none, saving blocked) · sale date-time (defaults to the moment the form opened; date picker allows today and earlier only; time picker's confirm is disabled while the chosen moment is after now; PERIOD_LOCKED message shows here) · save button "Record sale · {total}" enabled per MSL-R24, showing progress and ignoring taps while saving · **error banner** for §3.5 codes · **not-sent banner** M08 + Retry · **no-answer dialog** M19 ("Check sales", "Try again") · **branch-inactive (manager)** banner with only "Back" | MSL-R10, MSL-R11, MSL-R12, MSL-R14, MSL-R15, MSL-R16, MSL-R17, MSL-R18, MSL-R19, MSL-R20, MSL-R21, MSL-R22, MSL-R23, MSL-R24, MSL-R25, MSL-R26, MSL-R27, MSL-R36, MSL-R37, MSL-R40, MSL-R42, MSL-R43, MSL-R44, MSL-R45, MSL-R47, MSL-R48, MSL-R49, MSL-E01, MSL-E02, MSL-E04, MSL-E05, MSL-E07, MSL-E08, MSL-E09, MSL-E11, MSL-E12, MSL-E14, MSL-E18, MSL-E19, MSL-E20, MSL-E22, MSL-E23, MSL-E26, MSL-E28, MSL-E29 |
| Item picker (modal bottom sheet inside RecordSale) | Owner, Manager | Search field (name or SKU, debounce 400 ms; lists active items when empty) · loading · error M08 + Retry · empty "No items found" · results (name and SKU when there is one, max 2 lines; stock at the sale branch; "Out of stock" when `stock_status` is `OUT`) · next page on scroll. Picking an item reads `GET /items/{id}` (item + `stock_by_branch`), then adds the line (or +1 on an existing line) and closes; a failed read shows M08 + Retry in the sheet | MSL-R44, MSL-R14, MSL-E16, MSL-E20, MSL-E25, MSL-E31 |
| `SaleDetail(saleId: String, recorded: Boolean = false)` | Owner, Manager | **Loading** skeleton · **error** M08/server text + Retry · **not found** (404) M12 + "Back to sales" · **COMPLETED**: reference, local date-time, branch, source label, status, recorded by, lines (item, quantity, unit price, line total — full text, no truncation), subtotal, discount, tax, shipping (only when ≠ 0), total and "≈ {other}" (hidden with a Retry link when the rate failed, MSL-E27), payment method, paid-into account; "Reverse sale" button · **REVERSED**: same plus reversed at (local), reversed by, reason; no Reverse button · snackbar M06 once when opened with `recorded = true` · snackbar M15 after a reverse | MSL-R26, MSL-R28, MSL-R29, MSL-R30, MSL-R31, MSL-R34, MSL-R36, MSL-R37, MSL-R39, MSL-E03, MSL-E17, MSL-E18, MSL-E25, MSL-E27 |
| Reverse dialog (inside SaleDetail) | Owner, Manager | M13 title and body · required reason field · confirm with blank reason → M14 under the field, no call · confirming: progress, button ignores taps · success: closes, M15, detail shows REVERSED · 409 already reversed: M16 + reload · 409 period locked: M17 · 503 / 422 / 403: M09 / M11 with Retry, sale stays COMPLETED · connection failure: M08 + Retry, reason kept, sale as last loaded | MSL-R32, MSL-R33, MSL-R34, MSL-R35, MSL-R46, MSL-E06, MSL-E24, MSL-E30 |

Navigation: Record sale `201` → `SaleDetail(id, recorded = true)` with `popUpTo<RecordSale> { inclusive = true }`, so Back from the detail lands on the (refreshed) list (MSL-R26). "Check sales" → `SalesList(showBack = true)` pushed above `RecordSale` (MSL-R40). "Back to sales" → pop to the tab `SalesList`.

Labels with no text in spec §6 (tab names, titles, field labels, "Record sale", "Check sales", "Try again", "Back to sales", "Clear filters", "Out of stock", "No items found", source / method / status labels) take the web reference words where it has them (`web-ui src/features/item-management/content.ts`: "Walk-in", "POS", "Shopify", "WeBill365", "File import", "Cash", "KHQR", "ABA", "Wing", "Card", "Other", "Reversed"). Every string is in `features/src/main/res/values/strings.xml` (or `app/.../strings.xml` for the tab labels) and read with `stringResource()`; values with placeholders use `%1$s` formats, never concatenation (MSL-R38, MSL-D08).

## 5. Data

No table, column or migration changes in any repo. Names are the dictionary names proposed in spec §8 (from the migrations), JSON names from the code (§1).

| Table.field (dictionary name) | Read / write | Repo | IDs |
|---|---|---|---|
| `businesses.exchange_rate` | read (new endpoint, by `CurrentUser.businessId`) | identity-service | MSL-R15, MSL-R19, MSL-R43 |
| `businesses.exchange_rate` | read (§3.1) | android-ui | MSL-R15, MSL-R16, MSL-R19, MSL-R28, MSL-R43, MSL-E18, MSL-E23, MSL-E26, MSL-E27 |
| `users.branch_id` (via `GET /auth/me` `branch_id`) | read at sign-in, kept in the session | android-ui | MSL-R10, MSL-E04 |
| `branches.branch_id`, `branches.branch_name` (active only) | read | android-ui | MSL-R06, MSL-R11, MSL-R42 |
| `sales.branch_id`, `sales.sale_date`, `sales.source`, `sales.currency`, `sales.discount_amount`, `sales.tax_amount`, `sales.payment_method`, `sales.financial_account_id` | write through `POST /sales` (sales-service computes `subtotal`, `total_amount`, `sale_reference`, the USD value and stock) | android-ui | MSL-R10, MSL-R11, MSL-R12, MSL-R20, MSL-R21, MSL-R22, MSL-R23, MSL-R25 |
| `sale_items.item_id`, `sale_items.quantity_sold` (JSON `quantity` in the request), `sale_items.unit_price` | write through `POST /sales` | android-ui | MSL-R44, MSL-R15, MSL-R17, MSL-R47, MSL-R48 |
| `sales.reverse_reason` (JSON `reason`) | write through `POST /sales/{id}/reverse` | android-ui | MSL-R32, MSL-R33 |
| `sales.sale_reference`, `sale_date`, `status`, `source`, `currency`, `subtotal`, `discount_amount`, `tax_amount`, `shipping_amount`, `total_amount`, `payment_method`, `reversed_at`, `reverse_reason` (+ `branch_name`, `account_name`, `recorded_by_name`, `reversed_by_name` resolved by the server) | read | android-ui | MSL-R02, MSL-R28, MSL-R29, MSL-R31 |
| `sale_items.quantity_sold`, `unit_price`, `line_total` (+ `item_name`) | read | android-ui | MSL-R28 |
| `items.item_name`, `items.sku`, `items.selling_price`, `items.cost_price`, `items.allows_decimals`, `items.is_active` | read | android-ui | MSL-R44, MSL-R15, MSL-R17, MSL-R43, MSL-E31 |
| `item_stocks.quantity` (as `total_stock` per branch in §3.7 and `stock_by_branch[].quantity` in §3.8) | read | android-ui | MSL-R44, MSL-R18, MSL-E20 |
| finance `financial_accounts.financial_account_id`, `account_name`, `currency`, `is_business_wide`, `is_active`, `branch_id` (not in spec §8; finance-owned) | read | android-ui | MSL-R22 |
| KPI figures (computed by sales-service, no table) | read | android-ui | MSL-R01 |

## 6. Roles and branch scope

- **identity-service:** the new endpoint reads only `CurrentUser.businessId` (loaded from `users` on every request by `CurrentUserFilter`, never from token claims). No `requireOwner`, so a manager may call it; it exposes one integer of the caller's own business. The existing filter still blocks deactivated users (401), soft-deleted businesses (403 `BUSINESS_DELETED`) and un-onboarded users.
- **sales-service / finance-service (unchanged, relied on):** `BranchScope.readable` limits every read to the manager's branch (another branch's sale → 404, MSL-E03; `stock_by_branch` only lists readable branches); `BranchScope.requireWritable` refuses a manager writing another branch (403 `BRANCH_ACCESS_DENIED`, MSL-E04) and an inactive branch (409 `BRANCH_INACTIVE`, MSL-E07, MSL-E22); finance `AccountServiceImpl.list` forces the manager's own branch.
- **android-ui session:** `CurrentUser` (core) gains `branchId: String?` and `branchName: String?`, set by the auth feature at sign-in from `GET /auth/me` `branch_id` and the branch name it already loads, and persisted with "Keep me signed in". A remembered session saved before this change has no `branchId`; the app is unreleased, so such dev sessions just sign in again (no migration code).
- **android-ui nav:** both roles see the Sales tab and all three destinations (spec §2: both may record and reverse). No owner-only route exists in this feature.
- **android-ui UI:** the role comes from `SessionStore.currentUser.roleName` through a `GetSaleViewerUseCase` (`isOwner`, `branchId`, `branchName`). Manager: no branch filter chip, no branch column, branch on the form is read-only text of their own branch, `GET /branches` is never called. Owner: branch filter and picker from `GET /branches` (active).
- **android-ui data layer:** a manager's requests never carry a branch the user typed or picked: list and summary omit `branchId` (server scopes); `POST /sales` `branch_id`, items/accounts `branchId` and the `stock_by_branch` lookup use the session `branchId` (MSL-R10). An owner's `branchId` is only ever one from `GET /branches`.

## 7. Files

Android paths are under `features/src/main/java/com/kshrd/features/sales/` unless written in full; tests under `features/src/test/java/com/kshrd/features/sales/`. Identity paths are under `src/main/java/com/<root>/identity/` (§2 note).

| Repo | Create / change | Why (IDs) |
|---|---|---|
| identity-service | change `business/controller/BusinessController.java`: `@GetMapping("/me/exchange-rate")` returning `ApiResponse.ok(messages.get("business.exchange_rate.success"), …)` | MSL-R15, MSL-R19, MSL-R43 |
| identity-service | create `business/dto/ExchangeRateResponse.java` (record, `Integer exchangeRate` → `exchange_rate`) | MSL-R15 |
| identity-service | change `business/service/BusinessSettingsService.java` + `impl/BusinessSettingsServiceImpl.java`: `ExchangeRateResponse exchangeRate(CurrentUser)` — `businessRepository.findById(currentUser.businessId())`, else the existing `notFound()`; no owner check | MSL-R15, MSL-R19 |
| identity-service | change `src/main/resources/messages.properties`: `business.exchange_rate.success` | MSL-R15 |
| identity-service | create `src/test/java/.../business/controller/BusinessExchangeRateIT.java` (extends `IntegrationTest`): `@DisplayName("MSL-R15 MSL-R19 a manager gets their business exchange rate")`, `("MSL-R15 the owner gets the exchange rate")`, `("MSL-R15 without a token the exchange rate answers 401")`, `("MSL-R15 after the business is deleted the exchange rate answers 403 BUSINESS_DELETED")` | MSL-R15, MSL-R19, MSL-R43 |
| android-ui | change `core/src/main/java/com/kshrd/core/session/SessionStore.kt`: `CurrentUser.branchId`, `branchName` (defaults `null`), persisted keys | MSL-R10 |
| android-ui | change `core/src/main/java/com/kshrd/core/network/RemoteDataSource.kt`: pass the caught exception as `throwable` in both catch branches (today it is dropped), so the sales layer can tell a transport failure from other errors | MSL-R40, MSL-R45, MSL-R46 |
| android-ui | change `core/src/main/java/com/kshrd/core/navigation/AppDestinations.kt`: `SalesList(showBack: Boolean = false)`, `RecordSale`, `SaleDetail(saleId: String, recorded: Boolean = false)` | MSL-R09, MSL-R26, MSL-R40, MSL-R41 |
| android-ui | change `features/.../auth/domain/model/AuthModels.kt` (`AccountInfo.branchId`, `AuthSession.branchId`, defaults `null`), `auth/data/service/AuthServiceImpl.kt` (map `MeDto.branchId`), `auth/data/repository/AuthRepositoryImpl.kt` (`startSession` passes `branchId`, `branchName`) | MSL-R10 |
| android-ui | change `features/.../profile/presentation/screen/ProfileScreen.kt`: `onBack: (() -> Unit)? = null`, no back arrow when null (Profile as a tab) | MSL-R41 |
| android-ui | change `app/src/main/java/com/kshrd/starter/ui/navigation/MainGraph.kt`: home = `SalesList()` (replaces `NoteList` in `openHome`, business-setup `onDone`, and `startDestination`); `Scaffold` with the bottom bar on tab roots; `composable<SalesList>`, `composable<RecordSale>`, `composable<SaleDetail>` with the §4 navigation | MSL-R09, MSL-R26, MSL-R40, MSL-R41 |
| android-ui | create `app/src/main/java/com/kshrd/starter/ui/navigation/HomeBottomBar.kt` (Material 3 `NavigationBar`, icons `ShoppingCart` / `Person` from material-icons-core) + tab strings in `app/src/main/res/values/strings.xml` | MSL-R41, MSL-R38 |
| android-ui | create `data/api/SalesApi.kt` (Retrofit, all GETs of §3.1–§3.4, §3.7–§3.10) and `data/api/SaleWriteApi.kt` (`POST sales` with `@Tag attempt: SendAttempt`, `POST sales/{saleId}/reverse`), `data/api/SendAttempt.kt` (`connected` flag) and `data/api/ConnectedMarker.kt` (network interceptor setting the tag's flag) | MSL-R01 … MSL-R12, MSL-R14 … MSL-R35 (all §3 calls), MSL-R45 |
| android-ui | change `features/.../di/FeatureApiModule.kt`: `provideSalesApi(retrofit)`; `provideSaleWriteApi(retrofit, client)` = `retrofit.newBuilder().client(client.newBuilder().retryOnConnectionFailure(false).addNetworkInterceptor(ConnectedMarker()).build())` — OkHttp never silently re-sends a POST that may have reached the server, and the marker tells "not sent" from "no answer" | MSL-R25, MSL-R33, MSL-R45, MSL-E01, MSL-E02, MSL-E29 |
| android-ui | create `data/dto/SaleDtos.kt`, `data/dto/CatalogDtos.kt` (`ItemListRowDto`, `ItemDetailDto` with `item` and `stock_by_branch`), `data/dto/ReferenceDtos.kt` — `@SerializedName` exactly as §3, money as `BigDecimal` | MSL-R02, MSL-R44, MSL-R22, MSL-R28 |
| android-ui | create `data/mapper/SalesMappers.kt` (DTO → domain; UTC `LocalDateTime` → Asia/Phnom_Penh via `SalesTime`; branch stock = the sale branch's `stock_by_branch` entry, 0 when missing) | MSL-R39, MSL-R44, MSL-E13, MSL-E31 |
| android-ui | create `domain/service/SalesService.kt` + `data/service/SalesServiceImpl.kt` (each call in `RemoteDataSource`; `recordSale` turns a transport failure into `SubmitOutcome.NotSent` when `attempt.connected` is false and `SubmitOutcome.NoAnswer` when it is true, including a `2xx` whose body can't be read) | MSL-R27, MSL-R40, MSL-R45, MSL-E02, MSL-E29 |
| android-ui | create `domain/repository/SaleRepository.kt` + `data/repository/SaleRepositoryImpl.kt` (summary, page, detail, record, reverse; a `changes: StateFlow<Long>` bumped after a `201` record and a `200` reverse) and `domain/repository/SaleFormRepository.kt` + `data/repository/SaleFormRepositoryImpl.kt` (branches, rate, item search, item for sale, accounts, viewer from `SessionStore`); bind in `FeatureServiceModule`, `FeatureRepositoryModule` | MSL-R26, MSL-R34, MSL-R10 |
| android-ui | create `domain/model/SaleModels.kt` (enums `SaleSource`, `SaleStatus`, `PaymentMethod`, `Currency`; `SaleRow`, `SalePage`, `SaleSummary`, `SaleDetail`, `SaleLine`, `SaleFilters` with `isActive`), `domain/model/SaleFormModels.kt` (`SaleViewer`, `BranchOption`, `ItemOption`, `ItemForSale` with `branchStock`, `SaleAccount`, `SaleDraft`, `SubmitOutcome` = Recorded / Refused(code, message) / NotSent / NoAnswer) | MSL-R02, MSL-R05, MSL-R08 |
| android-ui | create `domain/model/SaleMath.kt` (pure, `BigDecimal`, HALF_UP like sales-service `Money`): KHR prefill = `selling_price × rate` to whole riel; line total = `quantity × unit_price` to 2 decimals; subtotal; total = subtotal − discount + tax; "≈" = USD × rate to whole riel / KHR ÷ rate to 2 decimals; below-cost check against `cost_price` (× rate, unrounded, for KHR) | MSL-R15, MSL-R19, MSL-R43, MSL-E12, MSL-E18 |
| android-ui | create `domain/model/SalesTime.kt`: zone `Asia/Phnom_Penh`, today's local date, UTC → local, picked local moment → ISO `+07:00`, "is in the future" | MSL-R01, MSL-R23, MSL-R39, MSL-E13, MSL-E14 |
| android-ui | create `domain/usecase/SaleFormRules.kt` (`ValidateSaleFormUseCase`: quantity empty/0 → M21, > 3 decimals → M23, fraction when `allows_decimals` is false → M03, above branch stock → M02; unit price empty → M22; discount/tax ≥ 0 and discount ≤ subtotal → M04; the MSL-R24 enable rule) and `domain/usecase/SaleAccounts.kt` (`ChooseSaleAccountsUseCase`, §3.9 rule) | MSL-R17, MSL-R18, MSL-R20, MSL-R22, MSL-R24, MSL-R47, MSL-R48, MSL-E09, MSL-E11, MSL-E20, MSL-E28 |
| android-ui | create `domain/usecase/SalesListUseCases.kt` (`GetSalesSummaryUseCase`, `GetSalesPageUseCase`, `ObserveSaleChangesUseCase`, `GetSaleViewerUseCase`, `GetSaleBranchesUseCase`), `domain/usecase/RecordSaleUseCases.kt` (`GetExchangeRateUseCase`, `SearchSaleItemsUseCase` with `branchId`, `GetItemForSaleUseCase` = `GET /items/{id}` + branch stock, `RecordSaleUseCase` building the §3.5 body with `source = WALK_IN`), `domain/usecase/SaleDetailUseCases.kt` (`GetSaleUseCase`, `ReverseSaleUseCase` with the blank-reason check) | MSL-R01, MSL-R03, MSL-R06, MSL-R44, MSL-R25, MSL-R32 |
| android-ui | create `presentation/model/SalesListUiContract.kt`, `RecordSaleUiContract.kt`, `SaleDetailUiContract.kt` (state / event / effect), `presentation/model/SaleFormReducer.kt` (pure: add item / +1 merge, branch change clears lines and account, currency change re-prefills prices, resets discount and tax to 0, re-suggests account), `presentation/model/MoneyText.kt` (`$1,234.56`, `៛12,300`), `presentation/model/SaleLabels.kt` (enum → string resource), `presentation/model/AmountInput.kt` (pure input filter: digits and one `.`, never `-`; price/discount/tax ≤ 2 decimals in USD, no decimals in KHR; quantity decimals not capped so M23 can show; 9 integer digits for quantity, 13 for money) | MSL-R11, MSL-R14, MSL-R16, MSL-R37, MSL-R38, MSL-R49 |
| android-ui | create `presentation/viewmodel/SalesListViewModel.kt` (card and list as separate `UiState`s; debounce; paging; pull-to-refresh; reload on `changes`) | MSL-R03, MSL-R04, MSL-R07, MSL-E21 |
| android-ui | create `presentation/viewmodel/RecordSaleViewModel.kt` (single in-flight guard on submit; maps §3.5 outcomes incl. NotSent → M08 banner and NoAnswer → M19 dialog; re-reads every line's item on `INSUFFICIENT_STOCK`; reloads branches on `BRANCH_INACTIVE`; saves the `SaleDraft` as JSON in `SavedStateHandle` on each change and restores it, re-reading items and accounts) | MSL-R25, MSL-R27, MSL-R40, MSL-R45, MSL-E01, MSL-E05, MSL-E07, MSL-E19, MSL-E22, MSL-E29 |
| android-ui | create `presentation/viewmodel/SaleDetailViewModel.kt` (`toRoute<SaleDetail>()`; rate loaded separately; reverse guard; connection failure keeps the dialog and reason; M06 once) | MSL-R30, MSL-R33, MSL-R35, MSL-R46, MSL-E27, MSL-E30 |
| android-ui | create `presentation/screen/SalesListScreen.kt`, `RecordSaleScreen.kt`, `SaleDetailScreen.kt` (stateful `*Screen` + stateless `*Content`, `DataStateScreenShell`, ≤ 250 lines each) | §4 rows |
| android-ui | create `presentation/components/`: `SalesKpiCard`, `SaleRow`, `SalesFilterChips` (+ date-range, method, status, branch sheets), `SalesListSkeleton`, `ItemPickerSheet`, `SaleLineRow`, `SaleTotals`, `SaleBranchField`, `CurrencyToggle`, `PaymentMethodChips`, `AccountPicker`, `SaleDateTimeField`, `SaveSaleButton`, `NoAnswerDialog`, `SaleDetailSections`, `SaleDetailSkeleton`, `ReverseSaleDialog`, `AmountField` (text field using `AmountInput`, decimal keyboard) — each with `@ThemePreviews` (+ `@FontScalePreviews` on rows, card, lines) | MSL-R01, MSL-R02, MSL-R44, MSL-R19, MSL-R21, MSL-R22, MSL-R23, MSL-R24, MSL-R32, MSL-R37, MSL-R49, MSL-E25 |
| android-ui | change `features/src/main/res/values/strings.xml`: all sales strings (M01–M23 client texts, labels) | MSL-R38 |
| android-ui | create tests: `domain/SaleMathTest.kt` (`MSL-R15 …`, `MSL-R19 …`, `MSL-R43 …`, `MSL-E12 …`, `MSL-E18 …`), `domain/SalesTimeTest.kt` (`MSL-R39 MSL-E13 a sale at 23:30 local …`, `MSL-R23 …`, `MSL-E14 …`, `MSL-R01 today is the Phnom Penh date`), `domain/SaleFormRulesTest.kt` (`MSL-R17 …`, `MSL-R47 MSL-E28 …`, `MSL-R48 …`, `MSL-R18 MSL-E20 …`, `MSL-R20 MSL-E11 …`, `MSL-R24 …`), `domain/SaleAccountsTest.kt` (`MSL-R22 MSL-E09 …`), `domain/ReverseSaleUseCaseTest.kt` (`MSL-R32 …`), `data/SalesMappersTest.kt` (`MSL-R44 MSL-E31 branch stock from stock_by_branch, missing entry is 0`), `data/SalesServiceOutcomeTest.kt` (`MSL-R45 MSL-E29 a failure before connecting is NotSent`, `MSL-R40 MSL-E02 a failure after connecting is NoAnswer` — incl. connect timeout vs read timeout via the `SendAttempt` flag), `presentation/AmountInputTest.kt` (`MSL-R49 …`), `presentation/SaleFormReducerTest.kt` (`MSL-R11 …`, `MSL-R14 …`, `MSL-R16 …`), `presentation/MoneyTextTest.kt` (`MSL-R37 …`), `presentation/RecordSaleViewModelTest.kt` with fakes and `Dispatchers.setMain` (`MSL-R10 …`, `MSL-R42 …`, `MSL-R25 MSL-E01 two taps send one POST`, `MSL-R26 …`, `MSL-R27 …` per code, `MSL-R45 …`, `MSL-E05 …`, `MSL-E07 …`, `MSL-E22 …`, `MSL-E26 …`, `MSL-E19 the draft is restored from SavedStateHandle`), `presentation/SalesListViewModelTest.kt` (`MSL-R03 …`, `MSL-R04 …`, `MSL-R06 …`, `MSL-R08 …`, `MSL-E21 …`, `MSL-R26 MSL-R34 a change reloads the card and page 0`). Names are backtick JUnit 4 names starting with the ID (the repo's style; JUnit 4 has no `@DisplayName`). `SaleDetailViewModel` is not unit-tested (it needs `toRoute`); its rules live in `ReverseSaleUseCase` and a pure `reverseOutcomeOf(error)` mapping tested as `MSL-R35 MSL-R46 MSL-E30 …` | all R/E with logic |

## 8. Edge cases

| ID | Where it's handled (repo, layer) |
|---|---|
| MSL-E01 | android: `RecordSaleViewModel` drops `Submit` while saving; button disabled with progress; `SaleWriteApi` client has `retryOnConnectionFailure(false)` |
| MSL-E02 | android: `SalesServiceImpl` → `SubmitOutcome.NoAnswer` (connected, then failed); VM shows `NoAnswerDialog` (M19); "Check sales" pushes `SalesList(showBack = true)`, "Try again" re-sends the same body; a duplicate is accepted (MSL-D16) |
| MSL-E03 | sales-service `requireVisible` → 404; android `SaleDetailViewModel` → M12 + "Back to sales" |
| MSL-E04 | sales-service `BranchScope.requireWritable` → 403; android never sends another branch (session `branchId`), maps 403 → M11 banner |
| MSL-E05 | android: VM on `INSUFFICIENT_STOCK` shows M02 (server text), keeps the form, re-reads every line with `GetItemForSaleUseCase` (`GET /items/{id}`); line errors then follow MSL-R18 |
| MSL-E06 | android: `SaleDetailViewModel` on `SALE_ALREADY_REVERSED` → M16, reloads `GET /sales/{id}` (shows REVERSED) |
| MSL-E07 | android: VM on `BRANCH_INACTIVE` (owner) → M07, reloads `GET /branches`, clears the branch (and so lines/account) if it's gone from the list |
| MSL-E08 | android: `ITEM_INACTIVE` → M10 (server text names the item) banner; lines stay so the user can remove one |
| MSL-E09 | android: `ChooseSaleAccountsUseCase` returns empty → M05, save disabled; switching currency re-runs it |
| MSL-E11 | android: `ValidateSaleFormUseCase` discount > subtotal → M04; save disabled |
| MSL-E12 | android: `SaleMath.isBelowCost` → M18 warning, not an error (save stays enabled) |
| MSL-E13 | android: `SalesTime` UTC → Asia/Phnom_Penh in mappers; list/summary `date`/`from`/`to` are local dates (server counts days in Asia/Phnom_Penh) |
| MSL-E14 | android: the draft holds the picked moment as an instant; save sends it unchanged; the "not in the future" check compares with now at save |
| MSL-E15 | android core (existing): `TokenRefreshAuthenticator` refreshes once; `SessionStore.expire()` → `MainGraph` pops to SignIn, the RecordSale entry (and its draft) is discarded |
| MSL-E16 | android: `ItemPickerSheet` empty state "No items found"; with no lines MSL-R24 keeps save disabled |
| MSL-E17 | android: `SaleRow` / `SaleDetailSections` label every source; Reverse offered on any COMPLETED sale; the KPI card is server-computed (WALK_IN only) |
| MSL-E18 | android: `MoneyText` KHR whole riel; "≈" from `SaleMath` with the §3.1 rate; stored totals always come from the server response |
| MSL-E19 | android: state in `ViewModel` (rotation); `SaleDraft` in `SavedStateHandle` (process death while backgrounded), restored with fresh item and account reads |
| MSL-E20 | android: search called with `branchId`, picker shows "Out of stock" (`stock_status` OUT); a picked line with branch stock 0 shows M02 via `ValidateSaleFormUseCase`; save disabled |
| MSL-E21 | android: `SalesListViewModel` keeps loaded rows, footer M08 + Retry for the failed page |
| MSL-E22 | android: VM on `BRANCH_INACTIVE` (manager) → M07 banner with only "Back"; save disabled |
| MSL-E23 | android: prices sent as shown; server recomputes totals and USD value at save; no client rate is sent |
| MSL-E24 | android: `SaleDetailViewModel` 503 → M09, 422/403 → M11, both with Retry; state stays COMPLETED (only a 200 changes it) |
| MSL-E25 | android: `SaleRow` and picker rows `maxLines = 2`, ellipsis; detail shows full text |
| MSL-E26 | android: `RecordSaleViewModel` rate state Error → inline M08 + Retry; `CurrencyToggle` KHR disabled; "≈" hidden until loaded |
| MSL-E27 | android: `SaleDetailViewModel` rate loaded independently; failure hides "≈" and shows a Retry link; the rest of the detail renders |
| MSL-E28 | android: `AmountInput` drops `-`; `ValidateSaleFormUseCase` → M21 (empty / 0) or M23 (> 3 decimals) on the line; save disabled |
| MSL-E29 | android: `ConnectedMarker` never ran → `SubmitOutcome.NotSent`; VM shows M08 banner with Retry, no M19 dialog, form kept |
| MSL-E30 | android: `SaleDetailViewModel` maps any transport failure on reverse to M08 + Retry, reason kept; a retry answering `409 SALE_ALREADY_REVERSED` → M16 and reload |
| MSL-E31 | android: search by name finds it (`sku` shown only when present); `GET /items/{id}` `stock_by_branch` gives its branch stock, no SKU needed |

## 9. Packages and verification

- **Packages:** none. Android uses what `libs.versions.toml` already has: Material 3 1.4.0 (`NavigationBar`, `PullToRefreshBox`, `DateRangePicker`, `DatePicker` with `selectableDates`, `TimePicker`, `ModalBottomSheet`), material-icons-core, navigation-compose 2.9.3, Retrofit 3 `@Tag`, OkHttp 4.12 network interceptors, Gson (via `converter-gson`), `java.time` (min SDK 26), JUnit 4 + kotlinx-coroutines-test with hand-written fakes (no mocking library). Identity uses its existing stack.
- **identity-service:** `./gradlew test --tests '*BusinessExchangeRateIT'`, then `./gradlew build`; Swagger at `:8081/swagger-ui.html` shows the endpoint. Reviewer checks: no owner check, business only from `CurrentUser`, envelope and snake_case `exchange_rate`, tests named with IDs, no production ID comments. `check-secrets` before any push.
- **android-ui:** `./gradlew assembleDebug test checkCode checkKtlint` (detekt must pass). `grep -rn 'Text("' features/src/main/java/com/kshrd/features/sales` returns nothing (MSL-R38). Device QA on an emulator against the local stack (gateway :8080, identity branch running) with an owner of two branches and a manager: every §4 state, KHR prefill and "≈", below-cost warning, an item without a SKU, airplane mode before tapping save (M08, MSL-E29), cutting the network after tapping save (M19, MSL-E02), cutting it during a reverse (M08, MSL-E30), process-death restore (`adb shell am kill` while backgrounded), owner vs manager differences. Reviewer checks: §6 scope rules, ViewModels call use cases only, all four `DataState` branches, previews on every component, no `Double` for money.

## 10. Questions and conflicts

None.

## Changes

| Date | Change | Why | Approved by |
|---|---|---|---|
| 2026-11-12 | §3.1: `ONBOARDING_INCOMPLETE` also covers a manager before the first-login password change | Found while building T01 | Vuththana |
