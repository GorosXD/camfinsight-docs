# identity-service

| Status | Method | Path | Who | Spec |
|---|---|---|---|---|
| 🚧 | GET | /api/v1/businesses/me/exchange-rate | Owner, Manager | [mobile-sales-management](../specs/mobile-sales-management/spec.md) MSL-R15, MSL-R19, MSL-R43 |

### GET /api/v1/businesses/me/exchange-rate

The caller's business exchange rate, read-only. Unlike `GET /api/v1/businesses/me` (owner only), a `BRANCH_MANAGER` may call it.

No parameters; the business is the caller's own. `200 OK`, message "The business exchange rate.", `data`: `{ "exchange_rate": <businesses.exchange_rate> }` — KHR per 1 USD (integer 1–100000).

| Status | `code` | When |
|---|---|---|
| 401 | — | Not signed in, bad token |
| 401 | `ACCOUNT_DEACTIVATED` | The user is deactivated |
| 401 | `SESSION_REVOKED` | The session was revoked |
| 403 | `BUSINESS_DELETED` | The caller's business is soft-deleted |
| 403 | `ONBOARDING_INCOMPLETE` | Onboarding is not finished: an owner with no business yet, or a manager who has not yet changed the temporary password |
| 404 | `NOT_FOUND` | The business row is missing (not expected) |
