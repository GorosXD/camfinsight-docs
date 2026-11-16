# Plan: Notifications on mobile (center and push)

**Spec:** `NTM` v3.0.0, commit `be5af46` · **Approved:** 2026-11-16 by Vuththana
**Repos:** ledgervision-identity-service (backend, api-dev; `feature/notification-mobile` from `origin/dev` 49cdf1e), ledgervision-android-ui (Android, android-dev; `feature/notification-mobile` from `origin/dev` e0a790a), camfinsight-docs (lead: `api/` statuses, data dictionary, TRD §6)

## 1. Scope (changed)

- **In:**
  - Spec §1 "In", and journeys §4a–§4e.
  - NTM-R01 … NTM-R47 and NTM-E01 … NTM-E41, except the withdrawn ones: NTM-R21, and with it NTM-D05 and NTM-M06.
  - The §8 data changes: `users.notification_preferences.push` and the new `push_tokens`.
  - The new and changed endpoints in §9.
  - The Firebase setup (NTM-D16, NTM-D29): a debug app id `com.kshrd.starter.dev`, and builds and tests that work without the Firebase files.
  - Device checks of push without real FCM, through a debug-only simulator (§9).
- **Out:**
  - Everything in spec §1 "Out": NTF alert rules and texts, new alert types, opening the related record, Telegram linking changes, email settings, iOS, sounds, quiet hours.
  - Web-ui stays unchanged (NTM-D28).
  - Importing the NTF docs.
  - Dictionary entries other than spec §8.

## 2. Repos and parts (changed)

| Repo | Role (agent) | Part | Order |
|---|---|---|---|
| identity-service | api-dev | **A.** Migration `V3` (`push_tokens`), `push` in preferences, the two push-token endpoints, `push_token` on sign-out (deleted only after an accepted logout, NTM-D33), and token deletion on deactivation, removal, ownership-transfer exit, anonymisation and business deletion | First. Android needs only the §3 contract, not this build. Work in a separate worktree (§9) so the 32 uncommitted infra changes in the local checkout are never staged |
| identity-service | api-dev | **B.** Push delivery: `PushSender` (FCM via Admin SDK, off without a key), `PushDelivery` after commit next to `TelegramDelivery`, and `PushTokenCleanupJob` | After A, because it needs `push_tokens` and `Channels.push`. Parallel with all Android parts |
| android-ui | android-dev | **C.** Notification center and the Dashboard bell (these endpoints are already on dev) | Parallel from day one |
| android-ui | android-dev | **D.** The Notifications row in Settings and the notification settings screen: Push switch disabled while In-app is off, field-error messages, the Telegram note that opens Telegram alerts. Includes the shared `:core` change that keeps field errors | Parallel. Unit tests run on fakes; the device check needs part A on dev |
| android-ui | android-dev | **E.** Push client: debug app id `.dev` (with the e2e `PKG` fix), `:core` push interfaces, Firebase only in `:app`, token sync, sign-out `push_token`, permission prompt, notification channel, tap → center, and the debug simulator | Parallel. Nothing waits on the Firebase files |
| camfinsight-docs | lead | `api/identity-service.md` statuses (planned → on branch → merged); the dictionary per spec §8; a TRD §6 row for `push-token-cleanup` | With the merges of A and B |
| both | api-qa, android-qa | Server ITs; device QA with the simulator; the real-FCM scenarios once the NTM-D16 files are handed out (§9) | Last |

## 3. API contract (changed)

All endpoints go through the gateway: `/api/v1/notifications/**` and `/api/v1/auth/**` already route to identity-service. Every response uses the usual envelope `{ success, code, message, data, timestamp }`. Both roles may call every endpoint below, and each call only touches the caller's own data, so no endpoint has a `403` case.

### `GET /api/v1/notifications`: exists (NTF), unchanged · NTM-R03, NTM-R04, NTM-R05 … NTM-R12, NTM-E09
- **Who:** Owner, Manager. A manager's `branch_id` is ignored by the server (NTF-R73), and the app never sends one.
- **Query:**
  - `feed` (`FINANCIAL` | `SYSTEM`), `unread` (`true`), `alert_type` (an `AlertType` name), `branch_id` (uuid, owner only), `cursor` (the last `next_cursor`).
  - Tab mapping (NTM-D02): All → no filter; Unread → `unread=true`; Financial → `feed=FINANCIAL`; Stock → `alert_type=STOCK`.
  - When the owner has chosen a branch, every tab also sends its `branch_id`.
- **200 OK** `data`:
  - `{ items: [notification], next_cursor: string | null }`, 20 per page, newest first.
  - Each **notification** is `{ notification_id, feed, alert_type, message, params, related_record_type, related_record_id, is_read, read_at, created_at (UTC), branch: { branch_id, branch_name, is_active } | null }`.
  - The app reads `alert_type` as text, so unknown values are kept (NTM-R12).
  - The app ignores `params` and `related_record_*` (NTM-D04).
- **Errors:**
  - `400 VALIDATION_FAILED` (bad `cursor` or enum value) → NTM-M28.
  - `401` → the existing refresh flow (NTM-E28).
  - A connection failure → NTM-M03.

### `GET /api/v1/notifications/unread-count`: exists, unchanged · NTM-R01, NTM-R02, NTM-R03, NTM-R17
- **200 OK** `data`: `{ total, financial, system }`. The bell and the Unread tab use `total`.

### `POST /api/v1/notifications/{notification_id}/read` and `…/unread`: exist, unchanged · NTM-R13, NTM-R14, NTM-R16, NTM-R35
- **Body:** none.
- **200 OK** `data`: the **notification**. Repeating a call is harmless (NTM-E34).
- **Errors:** `404 NOTIFICATION_NOT_FOUND` (NTM-M09) when the notification was purged or belongs to another user (never `403`, NTF-E20).
  - From a card, the app removes the card and shows NTM-M09 (NTM-E06).
  - For the read sent when a push is opened, a `404` is ignored (NTM-E18, NTM-E19, NTM-E37).
  - Any other failure of that opening read is silent (NTM-E39).

### `POST /api/v1/notifications/read-all`: exists, unchanged · NTM-R15, NTM-D12, NTM-E40
- **Body (optional):** `{ feed?, alert_type?, branch_id? }`. The app sends the current tab's `feed` or `alert_type`, plus the owner's `branch_id` when one is chosen. On All and Unread it sends no filter except `branch_id`.
- **200 OK** `data`: `{ updated_count }`, used as `{n}` in NTM-M04.
- **Errors:** `400 VALIDATION_FAILED` → NTM-M28. A connection failure → NTM-M03 (NTM-E04).

### `GET /api/v1/notifications/preferences`: changed (adds `push`) · NTM-R19, NTM-R20, NTM-R42, NTM-R44, NTM-R46, NTM-E32
- **200 OK** `data`: `{ alerts: { "<TYPE>": { in_app, telegram, push, locked } }, telegram_linked }`.
  - There is one entry per type the caller's role receives, in NTF §4a order.
  - `push` is the stored value; a type with nothing stored returns its Telegram default (NTM-D17).

### `PUT /api/v1/notifications/preferences`: changed (optional `push`) · NTM-R22, NTM-R42, NTM-R44, NTM-R45, NTM-D20, NTM-D28, NTM-D30, NTM-D31
- **Body:** `{ alerts: { "<TYPE>": { in_app, telegram, push? } } }`, listing every type of the role.
  - `push` follows the same strict-boolean rule as `in_app` and `telegram`.
  - When `push` is absent, the stored value is kept.
  - A type whose `push` was neither sent nor stored is written without a `push` key, so it keeps following its default.
  - There is no cross-field rule for `push`: any combination with `in_app` is accepted (NTM-D30).
  - The app always sends all three fields, with `telegram` unchanged. While a type's In-app is off, its `push` goes back as stored (NTM-R44).
- **200 OK** `data`: the same shape as GET.
- **Errors:** `400 VALIDATION_FAILED`. Its `data` is a flat map `{ "<field path>": "<message>" }`, with these keys:
  - `alerts.<TYPE>`: not for this role, or missing.
  - `alerts.<TYPE>.in_app`: missing, or locked.
  - `alerts.<TYPE>.telegram`: missing, or "Turn on in-app alerts to get them on Telegram." (`telegram_needs_in_app`).
  - Any toggle: "Enter a valid value." when it isn't a JSON boolean.
- **What the app shows on failure:** the changed switch reverts and one message shows.
  - When `data` has a key that starts with `alerts.<changed TYPE>`, its message shows (NTM-M30, NTM-R45, NTM-E41).
  - Otherwise, the envelope `message` shows (NTM-M28).
  - A connection failure shows NTM-M03.

### `POST /api/v1/notifications/push-tokens`: **new** · NTM-R25, NTM-R27, NTM-R29, NTM-R41, NTM-D21, NTM-D23
- **Who:** Owner, Manager (signed in).
- **Body:** `token` (dictionary `push_tokens.token`; required, not blank), `platform` (`push_tokens.platform`; required, `ANDROID`).
- **200 OK** `data`: `null`. The server upserts on `token`: a token held by another user moves to the caller, and `last_seen_at` is set to now on every call.
- **Errors:** `400 VALIDATION_FAILED` (`token` missing or blank; `platform` missing or not `ANDROID`), `401`. The app never shows these errors (NTM-R27).

### `POST /api/v1/notifications/push-tokens/unregister`: **new** · NTM-R25
- **Body:** `token` (required, not blank).
- **200 OK** `data`: `null`. Only the caller's own row is deleted; a token that isn't the caller's, or doesn't exist, is ignored (still `200`).
- **Errors:** `400 VALIDATION_FAILED` (`token`), `401`. Silent in the app.

### `POST /api/v1/auth/sign-out`: changed (optional `push_token`) · NTM-R28, NTM-R43, NTM-D22, NTM-D33, NTM-E20, NTM-E28, NTM-E38
- **Who:** public. It needs only the refresh token, so it works with an expired access token.
- **Body:** `refresh_token` (required), `push_token` (optional).
- **200 OK** `data`: `null`, always, as today.
- **When the push token is deleted:**
  - `KeycloakTokenClient.logout` returns `true` only when Keycloak answered 2xx, which proves the refresh token is genuine.
  - Only then does the server delete `push_tokens` where `token = push_token` and `user_id` = the token's `sub`.
  - A token that isn't the user's, or doesn't exist, is ignored.
  - A refresh token Keycloak rejects, including an already ended session (`invalid_grant`), deletes nothing. That token stays until a later registration moves it or the 60-day job removes it.
- **Errors:** `400 VALIDATION_FAILED` (`refresh_token` blank), as today.

### `GET /api/v1/branches`: exists, unchanged · NTM-R04, NTM-E07
- The owner's branch chip uses the active branches of the existing `workspace` (`GetWorkspaceUseCase`, which already filters on `is_active`).

### FCM data message (identity-service → app): **new** · NTM-R30, NTM-R31, NTM-R36, NTM-D07, NTM-D24
- **Delivery:** a data-only message (no `notification` block), Android priority `HIGH`, no `collapse_key`. One message goes to each of the recipient's tokens, for each new notification.
- **`data` keys** (all strings):
  - `notification_id`, `alert_type`.
  - `branch_name`, left out for business-wide notifications.
  - `title`: the English title, NTM-M10 … NTM-M20 or NTM-M22.
  - `body`: the branch name, or "All branches".
  - Never `message`, amounts or other names.
- **What the app shows:**
  - The title comes from its own string for `alert_type`, or NTM-M21 for an unknown type (NTM-R39, NTM-E10).
  - The text is `branch_name`, or its own "All branches" string (NTM-M25).
  - `title` and `body` satisfy NTM-R31 on the server side; the app does not show them.

## 4. Screens, routes and states (changed)

| Route | Who | States (loading / empty / error / each status) | IDs |
|---|---|---|---|
| `Dashboard` (changed: bell in `DashboardHeader`) | Owner, Manager | **Badge:** hidden when the count is unknown or 0; otherwise the number, capped at "99+".<br>**Count reload:** on `ScreenResumed` (covers opening and returning to the foreground), on a foreground push, and after every center action. A failed reload keeps the last count.<br>**Permission prompt:** once, on Android 13+, on the first Dashboard after the first sign-in on this phone.<br>**Bell tap:** opens `Notifications()` | NTM-R01, NTM-R02, NTM-R17, NTM-R26, NTM-E16, NTM-E17 |
| `Notifications(openedNotificationId: String? = null)` (new) | Owner, Manager | **Top bar:** Back, title, and "Mark all read" (double-check icon). It is disabled when no loaded card of the current tab is unread.<br>**Filters:** tabs All / Unread {n} / Financial / Stock (NTM-M23); the owner's branch chip ("All branches" plus active branches).<br>**List:** day groups. The bottom bar shows with Home selected, as in Figma 18.21.<br>**Loading:** skeleton.<br>**Errors:** a first-page error shows NTM-M03 or NTM-M28 with Retry. A next-page error shows a footer with NTM-M03 and Retry.<br>**Empty:** NTM-M01, or NTM-M02 on the Unread tab.<br>**Refresh:** pull-to-refresh, plus a quiet reload on resume and on a foreground push.<br>**Cards:** unread cards have a dot and a tint; read cards have an overflow menu with "Mark as unread".<br>**Snackbars:** NTM-M03, NTM-M04 with Undo, NTM-M05, NTM-M09.<br>**Opened from a push:** the read call first (failures silent apart from the `404` rule), then the All tab | NTM-R03 … NTM-R18, NTM-R35, NTM-R36, NTM-R38, NTM-R39, NTM-E39, NTM-E40 |
| `Profile` (Menu → Settings; changed) | Owner, Manager | New "Notifications" row in the Preferences card, before "Telegram alerts"; it shows no value | NTM-R19 |
| `NotificationSettings` (new) | Owner, Manager | **Layout:** no Figma (18.22 shows only the Settings list). Built from `SubpageScaffold`, `SettingsCard` and `SettingsSwitchRow`.<br>**Loading:** skeleton.<br>**Error:** NTM-M03 or NTM-M28 with Retry.<br>**Banner:** NTM-M07 with "Open settings" when the phone blocks the app's notifications; checked again on resume.<br>**Rows:** one per server type, titled NTM-M10 … NTM-M22 (NTM-M21 if unknown).<br>**Switches:** In-app is on and disabled when `locked`. Push is disabled while In-app is off, showing its stored value. Telegram shows "On" / "Off", read-only.<br>**Telegram note:** NTM-M29 when `telegram_linked` is false; tapping it opens `TelegramAlerts`.<br>**On resume:** the matrix reloads quietly, so a Telegram link made meanwhile shows.<br>**Saving:** a switch saves on change. On failure it reverts and a snackbar shows NTM-M30, NTM-M03 or NTM-M28 | NTM-R19, NTM-R20, NTM-R22, NTM-R23, NTM-R38, NTM-R39, NTM-R44, NTM-R45, NTM-R46, NTM-E41 |
| `TelegramAlerts` (exists, unchanged) | Owner, Manager | Opened from the NTM-M29 note; Back returns to `NotificationSettings` | NTM-R46 |
| System notification (channel "Alerts", NTM-M08) | Owner, Manager | Posted only in the background, and only when notifications are enabled.<br>**Content:** title per type, text per branch; auto-cancel; one per notification.<br>**Tap:** opens the center through `MainActivity`. When signed out, it goes `SignIn` → `Dashboard` → center | NTM-R31, NTM-R35, NTM-R36, NTM-R37 |

## 5. Data

| Table.field (dictionary name) | Read / write | Repo | IDs |
|---|---|---|---|
| `notifications.*` (NTF, V1) | Read: list, count, read and unread through the existing service. Read by `PushDelivery` after commit | identity-service; android reads through §3 | NTM-R03 … NTM-R18, NTM-R30, NTM-R40 |
| `users.notification_preferences` (`alerts.<TYPE>.push`, new key; `version` stays 1) | Read by `AlertPreferences` and the intake. Written by `PUT /preferences` | identity-service | NTM-R20, NTM-R22, NTM-R24, NTM-R30, NTM-R42 |
| `push_tokens` (new, `V3__create_push_tokens.sql`: `push_token_id` uuid PK; `user_id` FK `users`, not null; `token` text, not null, unique; `platform` enum `push_platform` (`ANDROID`); `created_at`; `last_seen_at`; indexes on `user_id` and `last_seen_at`) | Upsert on register. Delete on unregister, sign-out, invalid token, deactivation and the other paths in §6, and by the 60-day job | identity-service | NTM-R25, NTM-R28, NTM-R29, NTM-R32, NTM-R33, NTM-R41, NTM-R43 |
| `users.is_active`, `users.deleted_at`, `users.business_id`, `businesses.deleted_at` | Read: the token-deletion hooks run inside the existing transactions | identity-service | NTM-R33 |
| `branches.branch_name`, `branches.is_active` | Read: card label, "(inactive)", push text, owner chip | identity-service, android | NTM-R04, NTM-R07, NTM-R31, NTM-E07 |
| Phone storage (not a dictionary table): SharedPreferences `push` with `registered_token` and `permission_asked`. It survives sign-out, except `registered_token`, which sign-out clears | Read / write | android-ui (`:core`) | NTM-R25, NTM-R26, NTM-R27, NTM-R28 |

## 6. Roles and branch scope

- **identity-service:**
  - **Endpoints:** every endpoint takes `@AuthenticationPrincipal CurrentUser`. The push-token endpoints read and write only rows with `user_id = currentUser.userId()`, except the register upsert, which moves a token to the caller (NTM-D21).
  - **Sign-out:** deletes only for the `sub` of a refresh token Keycloak accepted.
  - **Unchanged:** list, count, read and unread scope (NTF-R25 … NTF-R29, NTF-R73); preference types per role (`AlertType.forRole`).
  - **Push recipients:** exactly the users the intake stored a notification for (`AlertRecipients`), so push inherits NTF scope.
  - **Token deletion, in the same transaction as the change (NTM-R33):**
    - `ManagerServiceImpl.deactivate`, when the manager changed;
    - `ManagerRemoval.remove`, which covers single removal and `removeAll` during business deletion;
    - `UserServiceImpl.leaveAfterTransfer` (the old owner is deactivated);
    - `UserServiceImpl.anonymize`;
    - `BusinessDeletionServiceImpl.softDelete`, which deletes the tokens of every user of the business, owner included.
  - On `origin/dev`, deactivating a branch that has an active manager is refused (`BRANCH_HAS_MANAGERS`). So "managers deactivated with their branch" always goes through the manager paths above.
- **android-ui:**
  - **Navigation:** both roles reach the bell, the center and the settings. No route is role-gated.
  - **Branch chip and filter:** shown only when `Workspace.managerBranchId == null` (owner). The ViewModel sends `branch_id` only for an owner with a branch chosen (NTM-E09).
  - **Settings:** list exactly the types the server returns.
  - **Push:** the app shows what it receives. Opening a push for another user's notification gets `404`, which is ignored (NTM-E37).

## 7. Files (changed)

| Repo | Create / change | Why (IDs) |
|---|---|---|
| identity | create `src/main/resources/db/migration/V3__create_push_tokens.sql` | `push_tokens` table and enum, unique `token`, indexes on `user_id` and `last_seen_at` (NTM-R25, NTM-R29, NTM-R41) |
| identity | create `notification/entity/PushToken.java`, `PushPlatform.java` | Mapped like `Notification.feed` (`PostgreSQLEnumJdbcType`) (NTM-R25) |
| identity | create `notification/repository/PushTokenRepository.java` | **Upsert:** native `ON CONFLICT (token) DO UPDATE SET user_id, platform, last_seen_at`.<br>**Deletes:** by `(token, user_id)`, by user, by business (join `users`), and by `last_seen_at < cutoff`.<br>**Lookup:** a user's tokens (NTM-R25, NTM-R28, NTM-R29, NTM-R32, NTM-R33, NTM-R41, NTM-R43) |
| identity | create `notification/service/PushTokenService.java`, `service/impl/PushTokenServiceImpl.java` | `register` and `unregister`; `deleteForUser(userId, token)`; `deleteAllOf(userId)` and `deleteAllInBusiness(businessId)` (`Propagation.MANDATORY`); `deleteNotSeenSince(cutoff)` (NTM-R25, NTM-R33, NTM-R41) |
| identity | create `notification/controller/PushTokenController.java`, `dto/RegisterPushTokenRequest.java`, `dto/UnregisterPushTokenRequest.java` | The two §3 endpoints, `@Valid` (NTM-R25, NTM-R27) |
| identity | change `notification/service/AlertPreferences.java` | **Channels:** `Channels(inApp, telegram, push)`; `push` defaults to `defaultTelegram()`.<br>**Raw value:** the stored `push` is kept as nullable, so `toJson` writes `push` only when it was sent or stored (NTM-R42, NTM-D17, NTM-D28) |
| identity | change `notification/dto/AlertChannelsRequest.java`, `AlertChannelsResponse.java`, `service/impl/NotificationPreferencesServiceImpl.java` | Optional strict-boolean `push`, kept when absent, returned by GET and PUT. No cross-field rule (NTM-R20, NTM-R22, NTM-R42, NTM-D30) |
| identity | change `notification/service/impl/AlertIntakeServiceImpl.java` | Collect the ids of newly inserted rows whose `channels.push()` is on, and publish `PushDeliveryRequested` (NTM-R24, NTM-R30, NTM-R40) |
| identity | create `notification/push/PushDeliveryRequested.java`, `PushDelivery.java` | **When:** `AFTER_COMMIT`, on a private two-thread daemon scheduler.<br>**Steps:** re-read the row, the user's tokens and the branch name; send; delete `INVALID_TOKEN` rows of that user.<br>**Errors and logs:** catch everything; log ids and counts only (NTM-R30, NTM-R32, NTM-R34, NTM-R40, NTM-E25) |
| identity | create `notification/push/PushSender.java`, `PushMessage.java`, `PushSendResult.java` (`SENT`, `INVALID_TOKEN`, `FAILED`), `PushText.java` | Sender port, push texts and the data keys of §3 (NTM-R31, NTM-D07, NTM-D24) |
| identity | create `notification/push/FcmPushSender.java`, `PushProperties.java`, `PushConfig.java` | **Send:** Admin SDK `sendEach`, data-only, `HIGH` priority, no collapse key, 10 s timeouts.<br>**Error mapping:** `UNREGISTERED`, `INVALID_ARGUMENT` and `SENDER_ID_MISMATCH` → `INVALID_TOKEN`; anything else → `FAILED`.<br>**No key or a bad key:** a disabled sender, logged once (§9) (NTM-R31, NTM-R32, NTM-R34) |
| identity | create `notification/job/PushTokenCleanupJob.java` | **Schedule:** cron `${ledgervision.jobs.push-token-cleanup.cron:0 45 2 * * *}`, zone Asia/Phnom_Penh, `@SchedulerLock(name = "push-token-cleanup", lockAtMostFor = "PT10M")`.<br>**Work:** deletes rows with `last_seen_at` older than now − 60 days (NTM-R41) |
| identity | change `auth/controller/AuthController.java`; create `auth/dto/SignOutRequest.java` (`refresh_token`, `push_token`); change `auth/service/AuthService.java`, `auth/service/impl/AuthServiceImpl.java` | Sign-out deletes the push token only after an accepted logout. It always answers `200` (NTM-R28, NTM-R43, NTM-D33, NTM-E38) |
| identity | change `common/keycloak/KeycloakTokenClient.java` | **Return value:** `logout` returns `boolean`: `true` on a 2xx; `false` on a 4xx (`invalid_grant`, ended session) or any failure. It still never throws.<br>**Other callers:** they ignore the result (NTM-R43, NTM-D33) |
| identity | change `manager/service/impl/ManagerServiceImpl.java`, `manager/service/ManagerRemoval.java`, `user/service/impl/UserServiceImpl.java`, `business/service/impl/BusinessDeletionServiceImpl.java` | Token deletion in the same transaction as each change (NTM-R33) |
| identity | change `src/main/resources/messages.properties` | `notification.push.title.<type>` (NTM-M10 … NTM-M20, NTM-M22), `notification.push.all_branches`, `notification.push_token.registered` / `.unregistered` (NTM-R31) |
| identity | change `build.gradle`, `.env.example`, `consul/identity-service.yml`, `k8s/40-identity-service.yaml`, `k8s/DEPLOY.md`, `README.md`, `src/test/resources/application-it.yml` | Admin SDK; `FCM_SERVICE_ACCOUNT_BASE64` (empty = push off); `ledgervision.push.fcm-service-account`; job cron; optional secret `camfinsight-fcm` (NTM-D16, NTM-R41) |
| identity | create or change tests, each name starting with the ID: `AlertPreferencesTest`, `NotificationPreferencesIT`, `PushTokenControllerIT`, `AuthControllerIT`, `KeycloakTokenClientTest`, `PushDeliveryIT`, `PushDeliveryTest`, `FcmPushSenderTest`, `PushTokenCleanupJobIT`, `PushTokenLifecycleIT`, `AlertIntakeIT` | NTM-R24, NTM-R25, NTM-R29, NTM-R30 … NTM-R34, NTM-R40 … NTM-R43, NTM-E25, NTM-E29, NTM-E38 |
| android | change `app/build.gradle.kts` | **Debug app id:** `buildTypes.debug { applicationIdSuffix = ".dev" }`; `namespace` stays `com.kshrd.starter`.<br>**Firebase:** dependencies, and the google-services plugin applied only when `app/google-services.json` exists (NTM-R47, NTM-D29, NTM-D16) |
| android | change `e2e/android/helpers.js` (`PKG = 'com.kshrd.starter.dev'`), `e2e/android/10_ui_tour.js` (use `PKG`, not the literal), `e2e/android/README.md` | The e2e scripts drive the debug build (NTM-R47) |
| android | change `gradle/libs.versions.toml`, root `build.gradle.kts`, `core/build.gradle.kts`, `.gitignore`, `local.properties.example`, `README.md` | **Dependencies:** Firebase BoM and messaging, used in `:app` only. The google-services plugin is declared `apply false`.<br>**Debug token:** `BuildConfig.PUSH_FAKE_TOKEN` in `:core`, set from `push.fakeToken`.<br>**Ignore:** `google-services.json`.<br>**README:** the setup, the `.dev` id, and the simulator commands (NTM-D16, NTM-D29) |
| android | change `core/.../navigation/AppDestinations.kt` | `Notifications(openedNotificationId: String? = null)`, `NotificationSettings` (NTM-R03, NTM-R19, NTM-R35) |
| android | change `core/.../data/entity/DataState.kt`, `core/.../network/RemoteDataSource.kt`, `core/src/test/.../network/RemoteDataSourceTest.kt` | **New field:** `DataState.Error` gains `fieldErrors: Map<String, String> = emptyMap()`.<br>**Parsing:** `parseErrorBody` keeps `data` when it is an object of strings.<br>**Compatibility:** shared and backward compatible; existing callers are unaffected (NTM-R45) |
| android | create `core/.../push/PushTokenSource.kt`, `core/.../push/PushTokenStore.kt` (+ binding in `core/di/SessionModule.kt`), `core/.../push/PushSignals.kt` | `PushTokenSource`: `suspend currentToken(): String?` and `suspend deleteToken()`.<br>`PushSignals`: `foreground`, `received`, `pendingOpen` with `requestOpen` / `consumeOpen`, and `EXTRA_NOTIFICATION_ID` (NTM-R25, NTM-R26, NTM-R28, NTM-R35, NTM-R36) |
| android | create `app/.../push/FirebasePushTokenSource.kt`, `app/.../push/AppMessagingService.kt` (`@AndroidEntryPoint`), `app/.../push/PushModule.kt`, `app/src/main/res/drawable/ic_stat_notification.xml` | **Only Firebase code in the app.**<br>**Token source:** null without a Firebase app, or the fake token in debug with `push.fakeToken=true`.<br>**Messaging service:** passes tokens and messages to the feature classes (NTM-R25, NTM-R36) |
| android | change `app/src/main/AndroidManifest.xml` | `POST_NOTIFICATIONS`; the messaging service (`exported=false`, `com.google.firebase.MESSAGING_EVENT`) (NTM-R26, NTM-R36) |
| android | change `app/.../app/StarterApplication.kt` | Creates the "Alerts" channel, tracks the foreground through `registerActivityLifecycleCallbacks`, starts `PushTokenSync` (NTM-R25, NTM-R36, NTM-R37) |
| android | change `app/.../MainActivity.kt`, `app/.../ui/navigation/MainGraph.kt` | **Push extra:** `EXTRA_NOTIFICATION_ID` is read in `onCreate` and `onNewIntent`, then passed to `requestOpen`.<br>**Navigation:** once signed in, go to `Notifications(id)` (`popUpTo<Notifications>{inclusive}`).<br>**New routes:** the two destinations, `ProfileLink.Notifications`, the Dashboard bell callback, and `NotificationSettings` → `TelegramAlerts` (NTM-R01, NTM-R19, NTM-R35, NTM-R46, NTM-E18, NTM-E24) |
| android | create `app/src/debug/AndroidManifest.xml`, `app/src/debug/java/com/kshrd/starter/debug/DebugPushReceiver.kt` | **Access:** debug only, exported with `android:permission="android.permission.DUMP"`, so only adb shell can reach it.<br>**Actions:** `com.kshrd.starter.debug.PUSH` and `com.kshrd.starter.debug.NEW_TOKEN`, sent to the same handlers as real pushes (QA, §9) |
| android | create `features/.../notification/data/api/NotificationApi.kt`, `data/dto/NotificationDtos.kt`, `data/mapper/NotificationMappers.kt`, `data/service/NotificationServiceImpl.kt` | **API:** Retrofit for the nine §3 notification endpoints.<br>**Parsing:** `alert_type` kept as text; `push` nullable (NTM-R03 … NTM-R25) |
| android | create `features/.../notification/data/repository/NotificationRepositoryImpl.kt`, `UnreadCountRepositoryImpl.kt`, `NotificationPreferencesRepositoryImpl.kt`, `PushRepositoryImpl.kt` | **Count:** a singleton that keeps the last count.<br>**Push:** register and unregister; `permission_asked`; `areNotificationsEnabled()`; SDK check (NTM-R02, NTM-R17, NTM-R22, NTM-R23, NTM-R25, NTM-R26) |
| android | create `features/.../notification/data/push/PushTokenSync.kt` | **Locking:** one Mutex around every step.<br>**Trigger:** a user appears.<br>**Steps:** get the token; unregister a different kept token (clear it on success); register the new one (keep it on success).<br>**`onNewToken`:** the same steps when signed in.<br>**Failures:** silent (NTM-R25, NTM-R27, NTM-R41, NTM-E21) |
| android | create `features/.../notification/data/push/PushMessageHandler.kt`, `SystemNotifier.kt` | **Foreground:** emits a signal and refreshes the count; nothing is posted.<br>**Background:** posts only when notifications are enabled.<br>**Notification:** title by `alert_type` (NTM-M21 if unknown); id = `notification_id.hashCode()`; an immutable `PendingIntent` with `EXTRA_NOTIFICATION_ID` (NTM-R31, NTM-R35, NTM-R36, NTM-R37, NTM-E10, NTM-E16) |
| android | create `features/.../notification/domain/model/NotificationModels.kt`, `NotificationTime.kt` | Models; the Asia/Phnom_Penh zone and day keys (NTM-R05, NTM-R06) |
| android | create `features/.../notification/domain/{repository,service}/*.kt`, `domain/usecase/NotificationCenterUseCases.kt`, `UnreadCountUseCases.kt`, `NotificationSettingsUseCases.kt`, `PushUseCases.kt` | One use case per intention (NTM-R02 … NTM-R26, NTM-R44, NTM-R45) |
| android | create `features/.../notification/presentation/model/NotificationsUiContract.kt`, `NotificationSettingsUiContract.kt`, `AlertTypeViews.kt`, `NotificationListReducer.kt`, `PreferenceSaveError.kt` | **`NotificationListReducer`:** pure functions for optimistic read and unread, mark all, mark-all enabled = some loaded card of the tab is unread, remove on `404`, day groups, "99+", branch label with "(inactive)".<br>**`PreferenceSaveError`:** picks the message from field errors keyed `alerts.<TYPE>`, else the envelope message, else M03 (NTM-R01, NTM-R05 … NTM-R08, NTM-R12 … NTM-R16, NTM-R45, NTM-E40) |
| android | create `features/.../notification/presentation/components/NotificationTabs.kt`, `NotificationCard.kt`, `DayHeader.kt`, `NotificationBranchFilter.kt`, `NotificationsSkeleton.kt`, `NotificationListFooter.kt`, `AlertPreferenceRow.kt` (Push `enabled = inApp`), `NotificationsOffBanner.kt`, `TelegramNote.kt` (clickable, NTM-M29), each with `@ThemePreviews` (and `@FontScalePreviews` on cards and rows) | NTM-R03 … NTM-R14, NTM-R20, NTM-R23, NTM-R44, NTM-R46, NTM-E11, NTM-E36 |
| android | create `features/.../notification/presentation/screen/NotificationsScreen.kt`, `NotificationSettingsScreen.kt` (callbacks `onBack`, `onOpenTelegram`), `presentation/viewmodel/NotificationsViewModel.kt`, `NotificationSettingsViewModel.kt` | **Center:** Undo snackbar with `SnackbarDuration.Long`; the opening read is silent except for the `404` rule.<br>**Settings:** saves run one after another; the matrix reloads on resume.<br>**"Open settings":** launches `Settings.ACTION_APP_NOTIFICATION_SETTINGS` (NTM-R03 … NTM-R23, NTM-R35, NTM-R36, NTM-R38, NTM-R44 … NTM-R46, NTM-E39, NTM-E41) |
| android | create `core-ui/src/main/res/drawable/ic_bell.xml`, `ic_check_check.xml`, `ic_scale.xml` | Icons for the bell, mark all, and reconciliation types. Other types reuse existing drawables (NTM-R06) |
| android | change `features/src/main/res/values/strings.xml` | Every live message: NTM-M01 … NTM-M05, NTM-M07 … NTM-M29 (NTM-M28 and NTM-M30 are server text) (NTM-R39) |
| android | change `features/.../di/FeatureApiModule.kt`, `FeatureServiceModule.kt`, `FeatureRepositoryModule.kt` | Hilt bindings |
| android | change `features/.../dashboard/presentation/components/DashboardHeader.kt`, `model/DashboardUiContract.kt`, `viewmodel/DashboardViewModel.kt`, `screen/DashboardScreen.kt` | Bell and badge; count on resume and on push; permission prompt effect and its launcher (NTM-R01, NTM-R02, NTM-R26) |
| android | change `features/.../profile/presentation/model/ProfileUiContract.kt`, `components/ProfileSettingsSections.kt` | `ProfileLink.Notifications` and its row (NTM-R19) |
| android | change `features/.../profile/data/dto/ProfileDtos.kt`, `domain/service/ProfileService.kt`, `data/service/ProfileServiceImpl.kt`, `data/repository/ProfileRepositoryImpl.kt` | **Sign-out order:**<br>1. Send the kept token as `push_token`; the result is ignored.<br>2. Always delete the FCM token (5 s timeout).<br>3. Clear the kept token.<br>4. Call `sessionStore.signOut()` (NTM-R28, NTM-E20, NTM-E28, NTM-E38) |
| android | move `features/.../sales/presentation/components/LoadMoreOnScroll.kt` → `features/.../common/presentation/components/LoadMoreOnScroll.kt` | Shared paging trigger (NTM-R09) |
| android | create tests under `features/src/test/.../notification/` (fakes in `fake/`): `NotificationTimeTest`, `NotificationListReducerTest`, `AlertTypeViewsTest`, `PreferenceSaveErrorTest`, `NotificationsViewModelTest`, `NotificationSettingsViewModelTest`, `UnreadCountRepositoryTest`, `PushTokenSyncTest`, `PushMessageHandlerTest`, `PushUseCasesTest`; plus `profile/SignOutPushTokenTest` | Names start with the IDs. No Firebase on the test classpath |

## 8. Edge cases (changed)

| ID | Where it's handled (repo, layer) |
|---|---|
| NTM-E01 | android `NotificationsViewModel`: the first page goes Loading → Error NTM-M03 with Retry; nothing cached is shown |
| NTM-E02 | android ViewModel and `NotificationListFooter`: loaded cards stay; the footer shows NTM-M03 with Retry |
| NTM-E03 | android ViewModel and reducer: the optimistic read is reverted; NTM-M03 |
| NTM-E04 | android ViewModel: nothing changes on screen before the `200`; NTM-M03; no Undo |
| NTM-E05 | android: the center reloads on pull, tab switch, resume and push; the Dashboard count reloads on resume and push |
| NTM-E06 | android reducer: a `404` (`NOTIFICATION_NOT_FOUND`) on read or unread removes the card and shows NTM-M09 |
| NTM-E07 | android mapper adds "(inactive)"; the owner chip comes from `workspace`, which lists active branches only |
| NTM-E08 | identity (NTF recipients, unchanged); android shows what the server returns, with no client branch logic for managers |
| NTM-E09 | android ViewModel never sends `branch_id` for a manager; identity ignores it anyway (NTF-R73) |
| NTM-E10 | android `AlertTypeViews` (card) and `PushMessageHandler` (push title NTM-M21) |
| NTM-E11 | android `NotificationCard`: the message has no `maxLines` |
| NTM-E12 | android `NotificationTime`: day key in Asia/Phnom_Penh |
| NTM-E13 | android: an empty Unread tab shows NTM-M02, and "Mark all read" is disabled |
| NTM-E14 | android ViewModel: a second tap while a mark-all is in flight is ignored |
| NTM-E15 | android: the Undo snapshot is dropped when the snackbar is dismissed or times out, or the ViewModel is cleared |
| NTM-E16 | android `PushMessageHandler`: nothing is posted while notifications are disabled; the center and bell are unaffected |
| NTM-E17 | android `PushRepositoryImpl`: no prompt below API 33; `areNotificationsEnabled()` also covers the app-level toggle |
| NTM-E18 | android `MainGraph` keeps `pendingOpen` through sign-in; the ViewModel ignores a `404` on the opening read |
| NTM-E19 | android ViewModel: a `404` on the opening read is ignored; the center loads All |
| NTM-E20 | android `ProfileRepositoryImpl` deletes the FCM token even when the sign-out call fails; identity drops the stale token later (NTM-R32 on the next send, or NTM-R41) |
| NTM-E21 | android `PushTokenSync.onNewToken` registers at once; identity deletes the old token when FCM reports it invalid |
| NTM-E22 | identity `PushDelivery` sends to every token of the user; there is no cross-device cancel |
| NTM-E23 | identity: push is decided at intake time from the stored preference; no client handling |
| NTM-E24 | android `MainActivity.onCreate` reads the extra on a cold start; the session is restored, then `pendingOpen` opens the center |
| NTM-E25 | identity `PushDelivery`: a sender failure is caught after commit; the notification and Telegram are unaffected (`PushDeliveryIT`) |
| NTM-E26 | android `NotificationSettingsViewModel`: the switch reverts; NTM-M03 |
| NTM-E27 | android `AlertPreferenceRow`: In-app is on and disabled when `locked` |
| NTM-E28 | android: the existing `TokenRefreshAuthenticator` handles expiry; sign-out sends `push_token` with only the refresh token |
| NTM-E29 | identity `PushTokenRepository` upsert on `token` (`PushTokenControllerIT`) |
| NTM-E30 | identity (NTF scope); android opens the center on All with a fresh load |
| NTM-E31 | android `NotificationTime` ignores the phone's zone; tested with a different default zone and a fixed `Clock` |
| NTM-E32 | identity `AlertPreferences` defaults, with push = the Telegram default; android shows them as returned |
| NTM-E33 | android settings ViewModel: saves are queued one after another (Mutex); each `PUT` carries the matrix at send time; a failure reverts only its own switch |
| NTM-E34 | identity read is idempotent; android repeats are harmless |
| NTM-E35 | android: tapping a push in the tray doesn't depend on the current Push setting |
| NTM-E36 | android `NotificationCard`: time and branch sit in a wrapping row and the branch shows in full; the phone shortens the push text |
| NTM-E37 | identity answers `404` for another user's notification (NTF-E20); android ignores it on the opening read |
| NTM-E38 | identity `AuthServiceImpl`: `logout` returns `false`, so nothing is deleted and the answer is still `200` (`AuthControllerIT`, `KeycloakTokenClientTest`). android deletes the FCM token anyway (`SignOutPushTokenTest`) |
| NTM-E39 | android `NotificationsViewModel`: an opening-read failure other than `404` is silent; the card stays unread and the list shows its own state |
| NTM-E40 | android `NotificationListReducer`: mark-all enabled = some loaded card of the current tab is unread; unloaded pages are not considered |
| NTM-E41 | android settings ViewModel and `PreferenceSaveError`: the `telegram_needs_in_app` field error reverts the In-app switch and shows that message (NTM-M30) |

## 9. Packages and verification (changed)

**Packages (new, for approval; confirm the latest patch on Maven or Google Maven when adding):**
- **android `gradle/libs.versions.toml`:**
  - Firebase Android BoM `com.google.firebase:firebase-bom`: 34.x, at least 34.0.0.
  - `com.google.firebase:firebase-messaging`: version from the BoM, `:app` only.
  - Plugin `com.google.gms.google-services`: 4.4.x. It must work with AGP 9.2.1.
  - If the plugin doesn't work with AGP 9.2.1: initialise Firebase manually with `FirebaseApp.initializeApp(context, FirebaseOptions)`, from `local.properties` keys `firebase.appId`, `firebase.apiKey`, `firebase.projectId` and `firebase.senderId` (per build type), behind the same runtime guard. The PR says which one was used.
- **identity `build.gradle`:** `com.google.firebase:firebase-admin` 9.x (at least 9.4.3), excluding `google-cloud-firestore` and `google-cloud-storage`. `./gradlew build` must stay green.
- **No other new packages:** the `Task` await is hand-written, and foreground tracking uses activity callbacks.

**Firebase setup (NTM-D16, NTM-D29):**
- **App ids:** debug builds are `com.kshrd.starter.dev` (the dev Firebase app); release builds are `com.kshrd.starter` (the prod app).
  - `namespace` stays the same, so classes, `BuildConfig` packages and component names don't change.
  - The `.dev` build installs next to an old `com.kshrd.starter` debug install, which can be uninstalled. Its "Keep me signed in" session starts empty.
- **The handed-out file:** the project's `app/google-services.json`, listing both clients; the plugin picks the client by application id. It is never committed (`.gitignore`: `google-services.json`, `**/google-services.json`).
- **Building without the file:**
  - The plugin isn't applied and Gradle logs "push off in this build".
  - At runtime no `FirebaseApp` exists, so `FirebasePushTokenSource` returns null (logged once): no registration. The center, bell and settings work normally.
  - In debug with `push.fakeToken=true`, a stable `debug-<uuid>` token is registered instead.
  - Firebase classes live only in `:app`, so `:features` and `:core` unit tests never load them.
- **identity:**
  - `FCM_SERVICE_ACCOUNT_BASE64` (base64 of the service-account JSON) goes in `.env`. It is listed empty in `.env.example`, read through Consul `ledgervision.push.fcm-service-account`, and in k8s comes from the optional secret `camfinsight-fcm`.
  - Empty → a disabled sender and one INFO log: "Push notifications are off: FCM_SERVICE_ACCOUNT_BASE64 is empty".
  - Malformed → an ERROR log without the key content, push stays off, and the service starts.
  - ITs run with an empty key and an enabled fake `PushSender`.

**Verification per repo:**
- **identity-service:**
  - Work in a worktree: `git -C ledgervision-identity-service worktree add ../ledgervision-identity-service-ntm -b feature/notification-mobile origin/dev`.
  - Run `./gradlew build` (unit tests plus Testcontainers ITs, with Docker running).
  - Run `./gradlew bootRun` with no key and check:
    - the "push off" log line appears;
    - sign-in works;
    - sign-out with a garbage `refresh_token` answers `200` and deletes no row (NTM-E38).
  - **Owner:** preferences list the owner types, each with `push`.
  - **Manager:** preferences list only the manager types; a sent `branch_id` is ignored.
  - **Field errors:** a `PUT` with `in_app=false, telegram=true` returns `400` with an `alerts.<TYPE>.telegram` field message (NTM-R45).
- **android-ui:**
  - Without `google-services.json` (this is also CI): `./gradlew assembleDebug test checkCode` passes, plus `./gradlew checkKtlint` for the report.
  - **App ids (NTM-R47):** `apkanalyzer manifest application-id` on the debug APK prints `com.kshrd.starter.dev`, and on `assembleRelease` prints `com.kshrd.starter`. Run `e2e/android/run-all.sh` against the `.dev` build.
  - With the file: run `assembleDebug` again.
  - **Owner:** the branch chip and "(inactive)".
  - **Manager:** no chip and no `branch_id` in the requests (OkHttp log).
  - **Settings:**
    - Push is disabled while In-app is off and keeps its value after a save (NTM-R44).
    - Turning In-app off on a type with Telegram on shows the Telegram field message (NTM-E41).
    - The NTM-M29 note opens Telegram alerts; back on the screen, the matrix reloads (NTM-R46).
- **Device QA without real FCM (android-qa, dev backend with part A deployed):**
  - **Basics:** the center, bell, settings and permission prompt.
  - **Tokens (`push.fakeToken=true`):** registration, moving the token between two users on one phone, and deletion on sign-out. Check with `select user_id, last_seen_at from identity.push_tokens`.
  - **Simulated push:** `adb shell am broadcast -n com.kshrd.starter.dev/com.kshrd.starter.debug.DebugPushReceiver -a com.kshrd.starter.debug.PUSH --es notification_id <id> --es alert_type STOCK --es branch_name "Toul Kork"`, run for these cases:
    - app in the background, and app in the foreground;
    - tapping the notification;
    - signed out;
    - app killed;
    - an unknown type;
    - another user's id;
    - permission denied.
  - **Token rotation:** `…debug.NEW_TOKEN --es token debug-2`.
  - **Covered this way:** NTM-R25 … NTM-R29, NTM-R31 (app side), NTM-R35 … NTM-R37, NTM-R47, NTM-E10, NTM-E16 … NTM-E19, NTM-E24, NTM-E29, NTM-E34, NTM-E35, NTM-E37, NTM-E39.
- **Needs only the identity key:**
  - NTM-R32: dev sends to a `debug-` token, FCM rejects it, and the row is deleted.
  - The FCM-accepted path of NTM-R30.
- **Blocked until both NTM-D16 files are handed out:**
  - Real delivery: NTM-R30 and NTM-R31 end to end, NTM-R36 in the background and Doze, NTM-R37 on a real push.
  - Token cases: NTM-E20, NTM-E21.
  - Several phones and timing: NTM-E22, NTM-E23.
- **Test names start with the ID:**
  - identity: `@DisplayName("NTM-R30 …")`.
  - android: `` fun `NTM-R45 shows the field message of the refused type`() ``.
  - Every R has a test or one of the QA scenarios above. NTM-R39 is checked by review and on the device.

## 10. Questions and conflicts (changed)

None.

## Changes

| Date | Change | Why | Approved by |
|---|---|---|---|
| 2026-11-16 | §1, §2, §3, §4, §7, §8, §9, §10 changed for spec v3.0.0: NTM-D29 … NTM-D33, NTM-R43 (changed), NTM-R44 … NTM-R47, NTM-E38 … NTM-E41, NTM-M09 code, NTM-M29, NTM-M30; NTM-R21, NTM-D05, NTM-M06 withdrawn | Plan §10 Q1 … Q7 answered through /sdd:amend (v3.0.0, be5af46) | Vuththana |
