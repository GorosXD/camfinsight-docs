# Notifications

Owned by identity-service (schema `identity`). `notifications` comes from the web notifications spec (NTF, identity
V1); `push_tokens` from `specs/notification-mobile` (identity V3, token rules tightened in V4 and V5).

## notifications

One alert for one user. The consumer of the alert events writes one row per recipient.

| Field Name | Datatype | Length | Required | PK/FK | Default | Reference | Description |
|---|---|---|---|---|---|---|---|
| `notification_id` | uuid |  | Y | PK |  |  | auto generated |
| `feed` | enum |  | Y |  |  |  | `FINANCIAL`, `SYSTEM`; set from `alert_type` |
| `alert_type` | enum |  | Y |  |  |  | `DUE_DATE`, `STOCK`, `IMPORT`, `MONTHLY_REPORT`, `UNMATCHED_LINES`, `INTEGRATION_FAILURE`, `REVENUE_DRIFT`, `REVIEW_SUBMITTED`, `REVIEW_RETURNED`, `CATEGORY_REQUEST`, `DAILY_SALES`, `BUDGET_EXCEEDED` |
| `message` | text |  | Y |  |  |  | the text shown, built by identity-service from `params` |
| `params` | jsonb |  | Y |  | `'{}'` |  | always a JSON object; keys per `alert_type` in [jsonb-shapes.md](jsonb-shapes.md#notificationsparams) |
| `related_record_type` | varchar | 50 |  |  |  |  | one of `bills`, `invoices`, `items`, `import_jobs`, `monthly_reports`, `reconciliations`, `integrations`, `forecasts`, `target_budgets`, `business_categories`, `sales` |
| `related_record_id` | uuid |  |  |  |  |  | the record the alert is about, in the table `related_record_type` names (no FK: it may live in another service) |
| `is_read` | boolean |  | Y |  | `false` |  | true = read<br>false = unread |
| `read_at` | timestamp |  |  |  |  |  | set when the row is read, cleared when marked unread (set exactly when `is_read` is true); read rows are purged 90 days after `read_at` |
| `dedup_key` | varchar | 150 | Y |  |  |  | unique per user; a repeated event for the same user is ignored |
| `is_sent_to_telegram` | boolean |  | Y |  | `false` |  | true = sent to the user's Telegram |
| `created_at` | timestamp |  | Y |  | `current_timestamp` |  | when identity-service stored the row (not the event's `occurred_at`); shown in Asia/Phnom_Penh |
| `user_id` | uuid |  | Y | FK |  | `users` (identity) | the recipient |
| `branch_id` | uuid |  |  | FK |  | `branches` (identity) | the branch the alert is about; empty = business-wide ("All branches") |

## push_tokens

The Firebase Cloud Messaging token of one phone signed in to the app.

| Field Name | Datatype | Length | Required | PK/FK | Default | Reference | Description |
|---|---|---|---|---|---|---|---|
| `push_token_id` | uuid |  | Y | PK |  |  | auto generated |
| `user_id` | uuid |  | Y | FK |  | `users` (identity) | the user signed in on the phone; a token registered by another user moves to that user |
| `token` | text | 2048 | Y |  |  |  | the FCM registration token; unique (one row per phone); at most 2048 characters, only `A–Z a–z 0–9 _ : . -` (`ck_push_tokens_token_length`, `ck_push_tokens_token_chars`, identity V5) |
| `platform` | enum |  | Y |  |  |  | `ANDROID` |
| `created_at` | timestamp |  | Y |  | `current_timestamp` |  |  |
| `last_seen_at` | timestamp |  | Y |  | `current_timestamp` |  | updated on every registration; rows not seen for 60 days are deleted by the `push-token-cleanup` job |
