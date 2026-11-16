# identity-service

| Status | Method | Path | Who | Spec |
|---|---|---|---|---|
| ✅ | POST | /api/v1/notifications/push-tokens | Owner, Manager | [notification-mobile](../specs/notification-mobile/spec.md) NTM-R25, NTM-R29, NTM-R41 |
| ✅ | POST | /api/v1/notifications/push-tokens/unregister | Owner, Manager | [notification-mobile](../specs/notification-mobile/spec.md) NTM-R25 |
| ✅ | GET | /api/v1/notifications/preferences (changed: `push`) | Owner, Manager | [notification-mobile](../specs/notification-mobile/spec.md) NTM-R19 … NTM-R21, NTM-R42 |
| ✅ | PUT | /api/v1/notifications/preferences (changed: `push`) | Owner, Manager | [notification-mobile](../specs/notification-mobile/spec.md) NTM-R22, NTM-R42 |
| ✅ | POST | /api/v1/auth/sign-out (changed: `push_token`) | Owner, Manager | [notification-mobile](../specs/notification-mobile/spec.md) NTM-R28, NTM-R43 |

The other notification endpoints (`GET /api/v1/notifications`, `GET …/unread-count`, `POST …/{notification_id}/read`,
`POST …/{notification_id}/unread`, `POST …/read-all`) exist in code and on the deployed gateway (web spec `notifications`,
NTF). They are used unchanged by notification-mobile and are not described here yet.

### POST /api/v1/notifications/push-tokens

Registers the phone's FCM token for the signed-in user, or refreshes it. Idempotent per token.

Body: `token` (required; the FCM registration token), `platform` (required; `ANDROID`). `200 OK`, message "Push token saved.", `data`: `null`.
A token already registered to another user is moved to the caller (NTM-D21). Every call sets `last_seen_at` to now;
tokens not seen for 60 days are deleted by a daily job (NTM-D23).

| Status | `code` | When |
|---|---|---|
| 400 | `VALIDATION_FAILED` | `token` missing or blank (`data.token`), `platform` missing or not `ANDROID` (`data.platform`) |
| 401 | `UNAUTHENTICATED` | Not signed in, bad token |
| 401 | `ACCOUNT_DEACTIVATED` | The user is deactivated |

### POST /api/v1/notifications/push-tokens/unregister

Removes one of the signed-in user's tokens (used when FCM replaces the phone's token). Body: `token` (required).
`200 OK`, message "Push token removed.", `data`: `null`. A token that isn't the caller's, or doesn't exist, is ignored (still `200`).

| Status | `code` | When |
|---|---|---|
| 400 | `VALIDATION_FAILED` | `token` missing or blank |
| 401 | — | Not signed in, bad token |

### GET /api/v1/notifications/preferences — change

Each entry of `data.alerts` becomes `{ in_app, telegram, push, locked }` (`push` boolean, new). A type with no stored value
returns its Telegram default (NTM-D17).

### PUT /api/v1/notifications/preferences — change

Each entry of `alerts` accepts an optional `push` (boolean, same strict-boolean rule as `in_app` and `telegram`).
A non-boolean `push` gives `400 VALIDATION_FAILED` with `data["alerts.<TYPE>.push"]` = "Enter a valid value.".
When `push` is absent, the stored value (or the default) is kept, so clients that don't send it keep working (NTM-D28).
The rest of the contract is unchanged: the whole matrix, last save wins.

### POST /api/v1/auth/sign-out — change

Body gains an optional `push_token`. When given, that token is deleted for the signing-out user (NTM-D22). The call
still needs only the refresh token. A `push_token` that isn't the user's, or doesn't exist, is ignored. The token is
deleted only when Keycloak accepted the logout; sign-out still answers `200` as today, also for a refresh token
Keycloak rejects, which deletes no push token (NTM-D33). Implementation note: Keycloak logout must report whether it was accepted; today it returns nothing and swallows failures.
