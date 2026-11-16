# identity-service

| Status | Method | Path | Who | Spec |
|---|---|---|---|---|
| 📝 | POST | /api/v1/notifications/push-tokens | Owner, Manager | [notification-mobile](../specs/notification-mobile/spec.md) NTM-R25, NTM-R29, NTM-R41 |
| 📝 | POST | /api/v1/notifications/push-tokens/unregister | Owner, Manager | [notification-mobile](../specs/notification-mobile/spec.md) NTM-R25 |
| 📝 | GET | /api/v1/notifications/preferences (changed: `push`) | Owner, Manager | [notification-mobile](../specs/notification-mobile/spec.md) NTM-R19 … NTM-R21, NTM-R42 |
| 📝 | PUT | /api/v1/notifications/preferences (changed: `push`) | Owner, Manager | [notification-mobile](../specs/notification-mobile/spec.md) NTM-R22, NTM-R42 |
| 📝 | POST | /api/v1/auth/sign-out (changed: `push_token`) | Owner, Manager | [notification-mobile](../specs/notification-mobile/spec.md) NTM-R28, NTM-R43 |

The other notification endpoints (`GET /api/v1/notifications`, `GET …/unread-count`, `POST …/{notification_id}/read`,
`POST …/{notification_id}/unread`, `POST …/read-all`) exist in code and on the deployed gateway (web spec `notifications`,
NTF). They are used unchanged by notification-mobile and are not described here yet.

### POST /api/v1/notifications/push-tokens

Registers the phone's FCM token for the signed-in user, or refreshes it. Idempotent per token.

Body: `token` (required; the FCM registration token), `platform` (required; `ANDROID`). `200 OK`, `data`: `null`.
A token already registered to another user is moved to the caller (NTM-D21). Every call sets `last_seen_at` to now;
tokens not seen for 60 days are deleted by a daily job (NTM-D23).

| Status | `code` | When |
|---|---|---|
| 400 | `VALIDATION_FAILED` | `token` missing or blank, `platform` not `ANDROID` |
| 401 | — | Not signed in, bad token |

### POST /api/v1/notifications/push-tokens/unregister

Removes one of the signed-in user's tokens (used when FCM replaces the phone's token). Body: `token` (required).
`200 OK`, `data`: `null`. A token that isn't the caller's, or doesn't exist, is ignored (still `200`).

| Status | `code` | When |
|---|---|---|
| 400 | `VALIDATION_FAILED` | `token` missing or blank |
| 401 | — | Not signed in, bad token |

### GET /api/v1/notifications/preferences — change

Each entry of `data.alerts` (`{ in_app, telegram, locked }` today) gains `push` (boolean). A type with no stored value
returns its Telegram default (NTM-D17).

### PUT /api/v1/notifications/preferences — change

Each entry of `alerts` accepts an optional `push` (boolean, same strict-boolean rule as `in_app` and `telegram`).
When `push` is absent, the stored value (or the default) is kept, so clients that don't send it keep working (NTM-D28).
The rest of the contract is unchanged: the whole matrix, last save wins.

### POST /api/v1/auth/sign-out — change

Body gains an optional `push_token`. When given, that token is deleted for the signing-out user (NTM-D22). The call
still needs only the refresh token. A `push_token` that isn't the user's, or doesn't exist, is ignored (the sign-out
still answers as today). An invalid refresh token gives the existing sign-out error and deletes no push token.
