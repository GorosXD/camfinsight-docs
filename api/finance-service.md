# finance-service

Endpoints used by [mobile-sales-management](../specs/mobile-sales-management/spec.md). They exist in code on `dev`; shapes are taken from the code (finance-service `AccountController`, `AccountResponse`). JSON is snake_case; query parameters use their Java names.

| Status | Method | Path | Who | Spec |
|---|---|---|---|---|
| 📝 | GET | /api/v1/bank-statements/accounts | Owner, Manager | MSL-R22 |

📝 here means "exists in code, entry written from the code".

### GET /api/v1/bank-statements/accounts

Query: `branchId` (a manager is always forced to their own branch), `status` (`ACTIVE` …).

`200 OK`, `data`: a list (not paged), ordered by `account_name`, of the branch's accounts **and** the business-wide ones: `{ financial_account_id, account_name, account_type, currency, starting_balance, current_balance, usd_equivalent, is_business_wide, is_active, branch_id, branch_name, created_at, updated_at }`. A manager gets business-wide accounts without their balances (`null`).

Shared errors only (`401`, `403`).

The income transaction for a sale is written by sales-service calling finance-service internally (`FeignFinanceGateway.recordSaleIncome`); clients never write it.
