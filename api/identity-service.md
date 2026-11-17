# identity-service

| Status | Method | Path | Who | Spec |
|---|---|---|---|---|
| 🚧 | POST | /api/v1/notifications/push-tokens (v5.0.0: `token` at most 2048, FCM characters only) | Owner, Manager | [notification-mobile](../specs/notification-mobile/spec.md) NTM-R25, NTM-R29, NTM-R41, NTM-R56, NTM-R57 |
| 🚧 | POST | /api/v1/notifications/push-tokens/unregister (v5.0.0: `token` at most 2048, FCM characters only) | Owner, Manager | [notification-mobile](../specs/notification-mobile/spec.md) NTM-R25, NTM-R56, NTM-R57 |
| ✅ | GET | /api/v1/notifications/preferences (changed: `push`) | Owner, Manager | [notification-mobile](../specs/notification-mobile/spec.md) NTM-R19, NTM-R20, NTM-R42, NTM-R46 |
| 🚧 | PUT | /api/v1/notifications/preferences (changed: `push`; v4.0.0: `"push": null` = absent) | Owner, Manager | [notification-mobile](../specs/notification-mobile/spec.md) NTM-R22, NTM-R42, NTM-R52 |
| 🚧 | POST | /api/v1/auth/sign-out (changed: `push_token`; v4.0.0: no bearer, replayed logout) | Owner, Manager | [notification-mobile](../specs/notification-mobile/spec.md) NTM-R28, NTM-R48, NTM-R49 |
| 🚧 | FCM | push data message (identity-service → app; v4.0.0: three keys) | Owner, Manager | [notification-mobile](../specs/notification-mobile/spec.md) NTM-R51 |

The other notification endpoints (`GET /api/v1/notifications`, `GET …/unread-count`, `POST …/{notification_id}/read`,
`POST …/{notification_id}/unread`, `POST …/read-all`) exist in code and on the deployed gateway (web spec `notifications`,
NTF). They are used unchanged by notification-mobile and are not described here yet.

### POST /api/v1/notifications/push-tokens

Registers the phone's FCM token for the signed-in user, or refreshes it. Idempotent per token.

Body: `token` (required; the FCM registration token, at most 2048 characters of `A–Z a–z 0–9 _ : . -`), `platform` (required; `ANDROID`). `200 OK`, message "Push token saved.", `data`: `null`.
A token already registered to another user is moved to the caller (NTM-D21). Every call sets `last_seen_at` to now;
tokens not seen for 60 days are deleted by a daily job (NTM-D23).
A token both too long and with other characters gets either message (NTM-E56). Like every JSON string, `token` is
cleaned before these checks (NTM-D57): NBSP, zero-width characters, the word joiner and BOM are removed anywhere, and
whitespace is trimmed at both ends; a space, tab or newline left inside is refused (NTM-E57).

| Status | `code` | When |
|---|---|---|
| 400 | `VALIDATION_FAILED` | `token` missing, or empty after cleaning (only spaces, tabs, newlines or invisible characters): `data.token` = "This field is required.", checked first (NTM-D56); `platform` missing or not `ANDROID` (`data.platform`) |
| 400 | `VALIDATION_FAILED` | `token` longer than 2048 characters: `data.token` = "Use at most 2048 characters." (NTM-R56); nothing stored |
| 400 | `VALIDATION_FAILED` | `token` with any other character: `data.token` = "Enter a valid value." (NTM-R57); nothing stored |
| 401 | `UNAUTHENTICATED` | Not signed in, bad token |
| 401 | `ACCOUNT_DEACTIVATED` | The user is deactivated |

### POST /api/v1/notifications/push-tokens/unregister

Removes one of the signed-in user's tokens (used when FCM replaces the phone's token). Body: `token` (required; same rules as register).
The token is cleaned and checked exactly as on register (NTM-D57): `" abc "` unregisters `abc`, and a token both too long and with
other characters gets either message (NTM-E56).
`200 OK`, message "Push token removed.", `data`: `null`. A token that isn't the caller's, or doesn't exist, is ignored (still `200`).

| Status | `code` | When |
|---|---|---|
| 400 | `VALIDATION_FAILED` | `token` missing, or empty after cleaning (only spaces, tabs, newlines or invisible characters): "This field is required.", checked first (NTM-D56) |
| 400 | `VALIDATION_FAILED` | `token` longer than 2048 characters: `data.token` = "Use at most 2048 characters." (NTM-R56); nothing deleted |
| 400 | `VALIDATION_FAILED` | `token` with any other character: `data.token` = "Enter a valid value." (NTM-R57); nothing deleted |
| 401 | — | Not signed in, bad token |

### GET /api/v1/notifications/preferences — change

Each entry of `data.alerts` becomes `{ in_app, telegram, push, locked }` (`push` boolean, new). A type with no stored value
returns its Telegram default (NTM-D17).

### PUT /api/v1/notifications/preferences — change

Each entry of `alerts` accepts an optional `push` (boolean, same strict-boolean rule as `in_app` and `telegram`).
A non-boolean `push` gives `400 VALIDATION_FAILED` with `data["alerts.<TYPE>.push"]` = "Enter a valid value.".
When `push` is absent or `null`, the stored value (or the default) is kept, so clients that don't send it keep working (NTM-D28, NTM-D44).
The rest of the contract is unchanged: the whole matrix, last save wins.

### POST /api/v1/auth/sign-out — change

Body: `refresh_token` (required) and an optional `push_token` (NTM-D22). Send it **without** an `Authorization`
header: a presented invalid or expired bearer is refused with `401 UNAUTHENTICATED` before the endpoint, and no
logout or deletion happens (NTM-R49, NTM-D35). A blank or whitespace-only `push_token` counts as none (NTM-D45).

The push token is deleted for the signing-out user when Keycloak accepts the logout, including a replayed logout
of an already ended session, which Keycloak answers 2xx (NTM-R48, NTM-D34). A refresh token Keycloak rejects
(garbage, tampered, expired) deletes nothing (NTM-D33). A `push_token` that isn't the user's, or doesn't exist, is
ignored. Sign-out always answers `200`.

### FCM data message (push) — identity-service → app

Data-only FCM message, priority `HIGH`, no collapse key, one per notification and token (NTM-R51, NTM-D24, NTM-D42).
`data` holds exactly:

| Key | Value |
|---|---|
| `notification_id` | the notification's id (uuid) |
| `alert_type` | the `AlertType` value |
| `branch_name` | the branch name; left out for a business-wide notification |

No `title`, `body`, `notification` block, amounts, names or `message`. The app builds the title from `alert_type`
and the text from `branch_name` or "All branches".
