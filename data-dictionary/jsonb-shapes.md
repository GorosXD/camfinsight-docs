# JSONB shapes

The keys of every `jsonb` field. A field's row in its table file links here.

## users.notification_preferences

A user's delivery channels per alert type. Empty (null) = every type uses its defaults. Owner: identity-service.

```json
{
  "version": 1,
  "alerts": {
    "STOCK": { "in_app": true, "telegram": true, "push": true },
    "IMPORT": { "in_app": true, "telegram": false }
  }
}
```

| Key | Type | Required | Description |
|---|---|---|---|
| `version` | integer | Y | `1` |
| `alerts` | object | Y | one key per alert type the user's role receives (`notifications.alert_type` values); other types are ignored |
| `alerts.<TYPE>.in_app` | boolean |  | default `true`; always `true` for the locked types `INTEGRATION_FAILURE`, `REVIEW_SUBMITTED`, `REVIEW_RETURNED`, `CATEGORY_REQUEST` |
| `alerts.<TYPE>.telegram` | boolean |  | default per type: `false` for `IMPORT`, `UNMATCHED_LINES`, `REVENUE_DRIFT`, `CATEGORY_REQUEST`; `true` for the others |
| `alerts.<TYPE>.push` | boolean |  | mobile push (specs/notification-mobile). Missing = the type's Telegram default. A save without `push` keeps what is stored |

A missing type, or a missing or non-boolean toggle, reads as its default.

## notifications.params

The values `notifications.message` is built from. Keys per `alert_type`; all required unless marked optional. Dates are
`YYYY-MM-DD`, months `YYYY-MM`. Owner: identity-service (the alert event's `params`).

| `alert_type` | Keys |
|---|---|
| `DUE_DATE` | `record_kind` (`BILL`, `INVOICE`), `number` (text), `counterparty_name` (text), `due_date` (date), `amount` (number), `currency` (`USD`, `KHR`), `due_state` (`DUE_SOON`, `OVERDUE`) |
| `STOCK` | `item_name` (text), `quantity` (number), `minimum_stock` (number) |
| `IMPORT` | `import_status` (`COMPLETED`, `NEEDS_REVIEW`, `FAILED`), `source_name` (text) |
| `MONTHLY_REPORT` | `report_month` (month), `is_complete` (boolean) |
| `UNMATCHED_LINES` | `unmatched_count` (number), `source_name` (text) |
| `INTEGRATION_FAILURE` | `provider` (`SHOPIFY`, `WEBILL365`, `POS`), `reason` (text) |
| `REVENUE_DRIFT` | `drift_percent` (number), `period_start` (date), `period_end` (date) |
| `BUDGET_EXCEEDED` | `budget_name` (text), `category_name` (text) |
| `REVIEW_SUBMITTED` | `account_name` (text), `period_end` (date), `submitted_by_name` (text) |
| `REVIEW_RETURNED` | `account_name` (text), `period_end` (date), `review_note` (text) |
| `CATEGORY_REQUEST` | `category_name` (text), `category_type` (`INCOME`, `EXPENSE`), `reason` (text), `requested_by_name` (text) |
| `DAILY_SALES` | `date` (date), `sales_count` (number), `revenue_usd` (number), `top_item_name` (text, optional), `refunds_count` (number) |
