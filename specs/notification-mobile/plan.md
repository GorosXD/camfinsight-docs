# Plan: Notifications on mobile (center and push)

**Spec:** `NTM` v4.0.0, commit `565f87e` · **Approved:** 2026-11-16 by Vuththana
**Repos:** ledgervision-identity-service (backend, api-dev; worktree `ledgervision-identity-service-ntm`, branch `feature/notification-mobile`, merged into `dev` by PR #42), ledgervision-android-ui (Android, android-dev; branch `feature/notification-mobile`, merged into `dev` by PR #4), ledgervision-web-ui (checked only, no change), camfinsight-docs (lead: `api/`, data dictionary, spec §8 / §9 states)

## 1. Scope (changed)

- **In:**
  - Spec §1 "In", and journeys §4a–§4e.
  - NTM-R01 … NTM-R55 and NTM-E01 … NTM-E49, except the withdrawn ones: NTM-R21 (with NTM-D05 and NTM-M06), NTM-R31, NTM-R43, NTM-E28, NTM-E38.
  - The §8 data changes: `users.notification_preferences.push`, the new `push_tokens`, and (v4.0.0) `push_tokens.token` at most 4096 characters.
  - The new and changed endpoints in §9, and (v4.0.0) the FCM data message keys.
  - The Firebase setup (NTM-D16, NTM-D29): a debug app id `com.kshrd.starter.dev`, and builds and tests that work without the Firebase files.
  - Device checks of push without real FCM, through a debug-only simulator (§9).
  - **v4.0.0 work:** everything parts A–E shipped stays; this change adds parts F, G and H in §2. The per-ID table in §2 says, for each amended ID, whether it is already built (test rename or test added) or needs new code.
- **Out:**
  - Everything in spec §1 "Out": NTF alert rules and texts, new alert types, opening the related record, Telegram linking changes, email settings, iOS, sounds, quiet hours.
  - Web-ui code. Its `signOut` already sends no `Authorization` header (NTM-D35; see §2). Its preferences calls stay unchanged (NTM-D28).
  - Importing the NTF docs.
  - Dictionary entries other than spec §8.
  - Removing NTM IDs from production comments outside the files this change touches.

## 2. Repos and parts (changed)

| Repo | Role (agent) | Part | Order |
|---|---|---|---|
| identity-service | api-dev | **A.** (shipped, PR #42) Migration `V3` (`push_tokens`), `push` in preferences, the two push-token endpoints, `push_token` on sign-out (deleted only after an accepted logout), and token deletion on deactivation, removal, ownership-transfer exit, anonymisation and business deletion | Done |
| identity-service | api-dev | **B.** (shipped, PR #42) Push delivery: `PushSender` (FCM via Admin SDK, off without a key), `PushDelivery` after commit next to `TelegramDelivery`, and `PushTokenCleanupJob` | Done |
| android-ui | android-dev | **C.** (shipped, PR #4) Notification center and the Dashboard bell | Done |
| android-ui | android-dev | **D.** (shipped, PR #4) The Notifications row in Settings and the notification settings screen | Done |
| android-ui | android-dev | **E.** (shipped, PR #4) Push client: debug app id `.dev`, `:core` push interfaces, Firebase only in `:app`, token sync, sign-out `push_token`, sign-out without a bearer (d2b8509), permission prompt, notification channel, tap → center, debug simulator | Done |
| identity-service | api-dev | **F. (v4.0.0, new)**<br>Code: the push data drops `title` and `body` (NTM-R51); `token` is capped at 4096 on register and unregister, with a `V4` CHECK constraint (NTM-R53).<br>Tests: renamed for NTM-R48, NTM-E42, NTM-E48, NTM-R51; added for NTM-R48 / NTM-E42 (replayed logout), NTM-E47, NTM-R52, NTM-R53 | First of the v4 work. Same worktree and branch, after merging `origin/dev` into it (`git fetch` first: the worktree's `origin/dev` is still 49cdf1e). New PR into `dev`. Android needs nothing from F: the app already ignores `title` / `body`, and real tokens are about 160 characters |
| android-ui | android-dev | **G. (v4.0.0, new)**<br>Code: a forced sign-out deletes the FCM token (NTM-R50, NTM-E43); a push is dropped while no one is signed in (NTM-R54); "Mark all read" while switching tab or branch (NTM-E44); the owner's branch list is fetched again after a failure (NTM-E46).<br>Tests: renamed for NTM-R06, NTM-R49, NTM-R51, NTM-E42, NTM-E47 | Parallel with F, on `feature/notification-mobile` (equal to the parent of `origin/dev`'s merge of PR #4; merge `origin/dev` first). New PR into `dev`. §10 Q1 answered (a): NTM-R54 as written |
| web-ui | — | None. NTM-R49 / NTM-D35 checked: `signOut` in `src/lib/api/auth.ts` calls `callGateway("/auth/sign-out", { method: "POST", body: { refresh_token } })` with no `token`, and `callGatewayWithMessage` adds `Authorization` only when a `token` is passed. Same on `origin/main`, `origin/dev-cleaned` and `feature/receipts`. The web repo has no `origin/dev` branch locally, which doesn't matter because there is no web work | — |
| camfinsight-docs | lead | **H. (v4.0.0)** `api/identity-service.md` edits for spec §9 (sign-out, push-tokens cap, `"push": null`, a new FCM data message section); `data-dictionary/notifications.md` `push_tokens.token` Length 4096; `api/` statuses 📝 → 🚧 → ✅ as F moves | With F: 📝 now, 🚧 when F is on its branch, ✅ when it merges into `dev` |
| both | api-qa, android-qa | Re-run of the QA scenarios named in §9 "v4.0.0 QA" | Last, after F and G merge into `dev` |

### v4.0.0, per amended ID

| ID | Already built? | Work |
|---|---|---|
| NTM-R48 (replaces R43) | **Built.** `AuthServiceImpl.signOut` deletes whenever `KeycloakTokenClient.logout` returns `true`, and real Keycloak answers 2xx to a replayed logout | Identity: rename the `NTM-R43` tests in `KeycloakTokenClientTest` (×4) and `AuthControllerIT` (×4) to NTM-R48. Add one `AuthControllerIT` test: NTM-R48 NTM-E42 NTM-D34, a replayed logout of an ended session (Keycloak 2xx) deletes the caller's `push_token`. api-qa re-runs scenarios R43-M2 and E38-M2 as NTM-R48-M2 and NTM-E42-M2 against real Keycloak |
| NTM-E42 (replaces E38) | **Built** | Rename `NTM-E38` → NTM-E42 in `AuthControllerIT` (×2), `KeycloakTokenClientTest` (×2) and android `SignOutPushTokenTest` (×1) |
| NTM-R49 | **Built** on Android (d2b8509: `/auth/sign-out` in `publicPaths`, so it gets no bearer and no refresh). Web-ui already sends none | Android: rename `AuthInterceptorsTest` "NTM-R28 NTM-D22 sign-out carries no bearer token…" to "NTM-R49 NTM-E47 NTM-R28 …". Rename `SignOutPushTokenTest` "NTM-E28 only the refresh token is needed…" to "NTM-R49 NTM-R28 …" |
| NTM-E47 | **Built.** The server's resource-server filter answers 401; the app never sends a bearer on sign-out | Identity: add an `AuthControllerIT` test. Android: covered by the rename above |
| NTM-E48 | **Built** (`pushToken == null \|\| pushToken.isBlank()`) | Rename `AuthControllerIT` "NTM-R43 no push_token, or a blank or null one…" to "NTM-E48 NTM-D45 NTM-R48 …" (it already sends `"  "`) |
| NTM-R50, NTM-E43 (replaces E28) | **Not built.** `TokenRefreshAuthenticator` calls `SessionStore.expire()`, and nothing deletes the FCM token | **New code**, android `PushTokenSync` (§7) |
| NTM-R51 (replaces R31) | **App built** (it reads only the three keys). **Server not built:** `FcmPushSender.toMessage` still puts `title` and `body` | **New code**, identity (§7). Android: rename `PushMessageHandlerTest` `NTM-R31` → NTM-R51 (×3) |
| NTM-R52 | **Built.** JSON `null` reaches `AlertChannelsRequest.push` as `null` (the `StrictBoolean` deserializer isn't called for null), and the service keeps the stored value | Identity: add a `NotificationPreferencesIT` test |
| NTM-R53, NTM-M31 | **Not built.** No size limit, and the column is `text` | **New code**, identity (§7) |
| NTM-R54 | **Not built.** `PushMessageHandler` posts whoever is signed in | **New code**, android (§7). §10 Q1 answered (a) |
| NTM-R55 | **Built.** `NotificationTabs` has had `horizontalScroll` since T15, and `@FontScalePreviews` exist. The QA "Stoc" was the edge of a row that scrolls | QA scenario NTM-R55-C1 only |
| NTM-R06, NTM-E31 (reworded) | **Built** (`NotificationTime.cardTime` in Asia/Phnom_Penh; `NotificationTimeTest` runs with the phone zone set to America/Los_Angeles) | Rename `NotificationTimeTest` "NTM-R06 a card shows HH-mm…" to "NTM-R06 NTM-E31 …", and add one assertion that starts from a UTC API time (`parseApi("2026-10-31T08:10:00Z")` → "15:10") |
| NTM-E18, NTM-E37 (reworded) | **Built** (T25) | None in code. QA re-runs NTM-E18-M1 together with NTM-R54 |
| NTM-R28, NTM-D07 (reworded) | **Built** | None |
| NTM-E44 | **Not built.** `finishMarkAll` marks the cards of whatever list is on screen when the answer arrives, and takes the Undo snapshot then. After a switch, it marks the wrong tab's cards, or offers an empty Undo | **New code**, android `NotificationsViewModel` (§7) |
| NTM-E45 | **Built.** `today` is set in `showFirstPage` on every load, and the groups and cards use `state.today` | QA scenario NTM-E45-C1 only |
| NTM-E46 | **Partly built.** A failed `GET /branches` gives an owner a workspace with no branches, so the chip shows only "All branches". But `WorkspaceRepositoryImpl` caches that workspace, so neither pull-to-refresh nor reopening fetches the list again | **New code**, android `NotificationsViewModel` (§7) |
| NTM-E49 | **Built** (accepted Android behaviour) | QA scenario NTM-E49-M1 only |

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
  - `401` → the existing refresh flow. If the refresh is rejected, the forced sign-out of NTM-R50 / NTM-E43 runs.
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

### `POST /api/v1/notifications/read-all`: exists, unchanged · NTM-R15, NTM-D12, NTM-E40, NTM-E44
- **Body (optional):** `{ feed?, alert_type?, branch_id? }`.
  - The app sends the `feed` or `alert_type` of the tab **at the moment of the tap**, plus the owner's `branch_id` at that moment (NTM-D38).
  - On All and Unread it sends no filter except `branch_id`.
- **200 OK** `data`: `{ updated_count }`, used as `{n}` in NTM-M04.
- **Errors:** `400 VALIDATION_FAILED` → NTM-M28. A connection failure → NTM-M03 (NTM-E04).

### `GET /api/v1/notifications/preferences`: changed (adds `push`) · NTM-R19, NTM-R20, NTM-R42, NTM-R44, NTM-R46, NTM-E32
- **200 OK** `data`: `{ alerts: { "<TYPE>": { in_app, telegram, push, locked } }, telegram_linked }`.
  - There is one entry per type the caller's role receives, in NTF §4a order.
  - `push` is the stored value; a type with nothing stored returns its Telegram default (NTM-D17).

### `PUT /api/v1/notifications/preferences`: changed (optional `push`) · NTM-R22, NTM-R42, NTM-R44, NTM-R45, NTM-R52, NTM-D20, NTM-D28, NTM-D30, NTM-D31, NTM-D43, NTM-D44
- **Body:** `{ alerts: { "<TYPE>": { in_app, telegram, push? } } }`, listing every type of the role.
  - `push` follows the same strict-boolean rule as `in_app` and `telegram`.
  - When `push` is absent **or `null`** (NTM-R52), the stored value is kept.
  - A type whose `push` was neither sent nor stored is written without a `push` key, so it keeps following its default.
  - There is no cross-field rule for `push`: any combination with `in_app` is accepted (NTM-D30).
  - The app always sends all three fields, with `telegram` unchanged. While a type's In-app is off, its `push` goes back as stored (NTM-R44).
- **200 OK** `data`: the same shape as GET.
- **Errors:** `400 VALIDATION_FAILED`. Its `data` is a flat map `{ "<field path>": "<message>" }`, with these keys:
  - `alerts.<TYPE>`: not for this role, or missing.
  - `alerts.<TYPE>.in_app`: missing, or locked.
  - `alerts.<TYPE>.telegram`: missing, or "Turn on in-app alerts to get them on Telegram." (`telegram_needs_in_app`).
  - Any toggle: "Enter a valid value." when it isn't a JSON boolean. A `null` `push` is not an error.
- **What the app shows on failure:** the changed switch reverts and one message shows.
  - When `data` has a key that starts with `alerts.<changed TYPE>`, its message shows (NTM-M30, NTM-R45, NTM-E41).
  - Otherwise, the envelope `message` shows (NTM-M28).
  - A connection failure shows NTM-M03.

### `POST /api/v1/notifications/push-tokens`: new · NTM-R25, NTM-R27, NTM-R29, NTM-R41, NTM-R53, NTM-D21, NTM-D23, NTM-D46
- **Who:** Owner, Manager (signed in).
- **Body:**
  - `token` (dictionary `push_tokens.token`): required, not blank, **at most 4096 characters** (NTM-R53).
  - `platform` (`push_tokens.platform`): required, `ANDROID`.
- **200 OK** `data`: `null`. The server upserts on `token`: a token held by another user moves to the caller, and `last_seen_at` is set to now on every call.
- **Errors:** `401`, and `400 VALIDATION_FAILED` for:
  - `token` missing or blank → `data.token` "This field is required.";
  - `token` over 4096 characters → `data.token` "Use at most 4096 characters." (NTM-M31). Nothing is stored;
  - `platform` missing or not `ANDROID` → `data.platform`.
- The app never shows these errors (NTM-R27).

### `POST /api/v1/notifications/push-tokens/unregister`: new · NTM-R25, NTM-R53
- **Body:** `token`: required, not blank, at most 4096 characters (NTM-R53).
- **200 OK** `data`: `null`. Only the caller's own row is deleted; a token that isn't the caller's, or doesn't exist, is ignored (still `200`).
- **Errors:** `401`, and `400 VALIDATION_FAILED` on `data.token` (blank, or over 4096 → NTM-M31). Nothing is deleted. Silent in the app.

### `POST /api/v1/auth/sign-out`: changed (optional `push_token`) (changed) · NTM-R28, NTM-R48, NTM-R49, NTM-D22, NTM-D33, NTM-D34, NTM-D35, NTM-D45, NTM-E20, NTM-E42, NTM-E47, NTM-E48
- **Who:** public. It needs only the refresh token, so it works with an expired access token.
- **Headers:** clients send **no `Authorization` header** (NTM-R49, NTM-D35).
  - The Android app leaves it out through `publicPaths` (shipped). The web BFF already passes no `token`.
  - A presented invalid or expired bearer is refused with `401` before the endpoint (NTM-E47). The server is not changed.
- **Body:**
  - `refresh_token`: required.
  - `push_token`: optional. Blank or whitespace-only counts as none (NTM-E48, NTM-D45).
  - No length check on `push_token`: the spec sets none for sign-out. A value over 4096 characters can't match a stored row, so nothing is deleted.
- **200 OK** `data`: `null`, always, as today.
- **When the push token is deleted:**
  - `KeycloakTokenClient.logout` returns `true` when Keycloak answers 2xx. This includes a replayed logout of an already ended session (NTM-R48, NTM-D34, NTM-E42).
  - Only then does the server delete `push_tokens` where `token = push_token` and `user_id` = the refresh token's `sub`.
  - A token that isn't the user's, or doesn't exist, is ignored.
  - A refresh token Keycloak rejects (garbage, tampered, expired) deletes nothing. That push token stays until a later registration moves it or the 60-day job removes it.
- **Errors:**
  - `400 VALIDATION_FAILED` (`refresh_token` blank), as today.
  - `401` for a presented bad bearer (NTM-E47).

### `GET /api/v1/branches`: exists, unchanged · NTM-R04, NTM-E07, NTM-E46
- The owner's branch chip uses the active branches of the existing `workspace` (`GetWorkspaceUseCase`, which already filters on `is_active`).
- After a failure, the list is fetched again with `forceRefresh = true` (§7, NTM-E46).

### FCM data message (identity-service → app): new (changed) · NTM-R30, NTM-R36, NTM-R51, NTM-R54, NTM-D07, NTM-D24, NTM-D42
- **Delivery:** a data-only message (no `notification` block), Android priority `HIGH`, no `collapse_key`. One message goes to each of the recipient's tokens, for each new notification.
- **`data` keys** (all strings), **exactly**:
  - `notification_id`.
  - `alert_type`.
  - `branch_name`, left out for a business-wide notification (`branch_id` null).
- **Never sent:** `title`, `body`, `message`, amounts or other names (NTM-R51). v4.0.0 removes `title` and `body` from the server.
- **What the app shows:**
  - **Title:** its own string for `alert_type` (NTM-M10 … NTM-M20, NTM-M22), or NTM-M21 for an unknown type (NTM-R39, NTM-E10).
  - **Text:** `branch_name`, or its own "All branches" string (NTM-M25).
- **When the app shows it:**
  - Only when someone is signed in on the phone (NTM-R54; see §10 Q1).
  - Only in the background (NTM-R36).
  - Only while the phone allows the app's notifications (NTM-E16).

## 4. Screens, routes and states (changed)

| Route | Who | States (loading / empty / error / each status) | IDs |
|---|---|---|---|
| `Dashboard` (changed: bell in `DashboardHeader`) | Owner, Manager | **Badge:** hidden when the count is unknown or 0; otherwise the number, capped at "99+".<br>**Count reload:** on `ScreenResumed` (covers opening and returning to the foreground), on a foreground push, and after every center action. A failed reload keeps the last count.<br>**Permission prompt:** once, on Android 13+, on the first Dashboard after the first sign-in on this phone.<br>**Bell tap:** opens `Notifications()` | NTM-R01, NTM-R02, NTM-R17, NTM-R26, NTM-E16, NTM-E17 |
| `Notifications(openedNotificationId: String? = null)` (changed) | Owner, Manager | **Top bar:** Back, title, and "Mark all read" (double-check icon). It is disabled when no loaded card of the current tab is unread, and while a mark-all is sending.<br>**Filters:**<br>- Tabs All / Unread {n} / Financial / Stock (NTM-M23), in a row that scrolls sideways when the labels don't fit; no label is cut (NTM-R55).<br>- The owner's branch chip: "All branches" plus active branches. When the branch list failed, it offers only "All branches", shows no error, and the list is fetched again on pull-to-refresh or reopening (NTM-E46).<br>**List:** day groups, with "Today" / "Yesterday" taken from the last load, not live at midnight (NTM-E45). The bottom bar shows with Home selected, as in Figma 18.21.<br>**Loading:** skeleton.<br>**Errors:** a first-page error shows NTM-M03 or NTM-M28 with Retry. A next-page error shows a footer with NTM-M03 and Retry.<br>**Empty:** NTM-M01, or NTM-M02 on the Unread tab.<br>**Refresh:** pull-to-refresh, plus a quiet reload on resume and on a foreground push.<br>**Cards:** unread cards have a dot and a tint; read cards have an overflow menu with "Mark as unread". Times are in Asia/Phnom_Penh (NTM-R06).<br>**Mark all while switching:** applies to the tab and branch of the tap. NTM-M04 with Undo shows; Undo restores the tapped tab's snapshot. The new tab's cards aren't marked on screen, and it loads as usual (NTM-E44).<br>**Snackbars:** NTM-M03, NTM-M04 with Undo, NTM-M05, NTM-M09.<br>**Opened from a push:** the read call first (failures silent apart from the `404` rule), then the All tab | NTM-R03 … NTM-R18, NTM-R35, NTM-R36, NTM-R38, NTM-R39, NTM-R55, NTM-E39, NTM-E40, NTM-E44, NTM-E45, NTM-E46 |
| `Profile` (Menu → Settings) | Owner, Manager | New "Notifications" row in the Preferences card, before "Telegram alerts"; it shows no value | NTM-R19 |
| `NotificationSettings` | Owner, Manager | **Layout:** no Figma (18.22 shows only the Settings list). Built from `SubpageScaffold`, `SettingsCard` and `SettingsSwitchRow`.<br>**Loading:** skeleton.<br>**Error:** NTM-M03 or NTM-M28 with Retry.<br>**Banner:** NTM-M07 with "Open settings" when the phone blocks the app's notifications; checked again on resume.<br>**Rows:** one per server type, titled NTM-M10 … NTM-M22 (NTM-M21 if unknown).<br>**Switches:** In-app is on and disabled when `locked`. Push is disabled while In-app is off, showing its stored value. Telegram shows "On" / "Off", read-only.<br>**Telegram note:** NTM-M29 when `telegram_linked` is false; tapping it opens `TelegramAlerts`.<br>**On resume:** the matrix reloads quietly, so a Telegram link made meanwhile shows.<br>**Saving:** a switch saves on change. On failure it reverts and a snackbar shows NTM-M30, NTM-M03 or NTM-M28 | NTM-R19, NTM-R20, NTM-R22, NTM-R23, NTM-R38, NTM-R39, NTM-R44, NTM-R45, NTM-R46, NTM-E41 |
| `TelegramAlerts` (exists, unchanged) | Owner, Manager | Opened from the NTM-M29 note; Back returns to `NotificationSettings` | NTM-R46 |
| `SignIn` after a rejected refresh (exists; changed behind it) | Owner, Manager | The existing `sessionExpired` navigation is unchanged. Behind it, with no UI and no sign-out call: the phone's FCM token is deleted (5 s timeout) and the kept `registered_token` is cleared (NTM-R50, NTM-E43) | NTM-R50, NTM-E43 |
| System notification (channel "Alerts", NTM-M08) (changed) | Owner, Manager | **Posted only when:** someone is signed in (NTM-R54), the app is in the background, and notifications are enabled. A push arriving a moment after Home may count as foreground and post nothing (NTM-E49).<br>**Content:** title per type, text per branch (NTM-R51); auto-cancel; one per notification.<br>**Tap:** opens the center through `MainActivity`. When signed out (a notification already in the tray), it goes `SignIn` → `Dashboard` → center (NTM-E18) | NTM-R35, NTM-R36, NTM-R37, NTM-R51, NTM-R54, NTM-E18, NTM-E49 |

## 5. Data (changed)

| Table.field (dictionary name) | Read / write | Repo | IDs |
|---|---|---|---|
| `notifications.*` (NTF, V1) | Read: list, count, read and unread through the existing service. Read by `PushDelivery` after commit | identity-service; android reads through §3 | NTM-R03 … NTM-R18, NTM-R30, NTM-R40 |
| `users.notification_preferences` (`alerts.<TYPE>.push`; `version` stays 1) | Read by `AlertPreferences` and the intake. Written by `PUT /preferences`; `null` or absent `push` keeps the stored value | identity-service | NTM-R20, NTM-R22, NTM-R24, NTM-R30, NTM-R42, NTM-R52 |
| `push_tokens` (`V3__create_push_tokens.sql`: `push_token_id` uuid PK; `user_id` FK `users`, not null; `token` text, not null, unique; `platform` enum `push_platform` (`ANDROID`); `created_at`; `last_seen_at`; indexes on `user_id` and `last_seen_at`). **v4.0.0:** `token` at most 4096 characters, enforced by the DTOs and by `ck_push_tokens_token_length CHECK (char_length(token) <= 4096)` in the next free migration (`V4__push_tokens_token_length.sql` if V4 is still free on `origin/dev`) | Upsert on register. Delete on unregister, sign-out (also on a replayed logout), invalid token, deactivation and the other paths in §6, and by the 60-day job | identity-service | NTM-R25, NTM-R28, NTM-R29, NTM-R32, NTM-R33, NTM-R41, NTM-R48, NTM-R53 |
| `users.is_active`, `users.deleted_at`, `users.business_id`, `businesses.deleted_at` | Read: the token-deletion hooks run inside the existing transactions | identity-service | NTM-R33 |
| `branches.branch_name`, `branches.is_active` | Read: card label, "(inactive)", push `branch_name`, owner chip | identity-service, android | NTM-R04, NTM-R07, NTM-R51, NTM-E07, NTM-E46 |
| Phone storage (not a dictionary table): SharedPreferences `push` with `registered_token` and `permission_asked` | Read / write. Both survive sign-out, except `registered_token`, which is cleared by sign-out and, from v4.0.0, by a forced sign-out | android-ui (`:core`) | NTM-R25, NTM-R26, NTM-R27, NTM-R28, NTM-R50 |

## 6. Roles and branch scope (changed)

- **identity-service:**
  - **Endpoints:** every endpoint takes `@AuthenticationPrincipal CurrentUser`. The push-token endpoints read and write only rows with `user_id = currentUser.userId()`, except the register upsert, which moves a token to the caller (NTM-D21).
  - **Sign-out:** deletes only for the `sub` of a refresh token Keycloak accepted the logout for. A replayed logout counts (NTM-D34), but only the holder of that user's own old refresh token can send one. No session check is added.
  - **Unchanged:** list, count, read and unread scope (NTF-R25 … NTF-R29, NTF-R73); preference types per role (`AlertType.forRole`).
  - **Push recipients:** exactly the users the intake stored a notification for (`AlertRecipients`), so push inherits NTF scope. The push carries no text of the notification (NTM-R51).
  - **Token deletion, in the same transaction as the change (NTM-R33):**
    - `ManagerServiceImpl.deactivate`, when the manager changed;
    - `ManagerRemoval.remove`, which covers single removal and `removeAll` during business deletion;
    - `UserServiceImpl.leaveAfterTransfer` (the old owner is deactivated);
    - `UserServiceImpl.anonymize`;
    - `BusinessDeletionServiceImpl.softDelete`, which deletes the tokens of every user of the business, owner included.
  - On `origin/dev`, deactivating a branch that has an active manager is refused (`BRANCH_HAS_MANAGERS`). So "managers deactivated with their branch" always goes through the manager paths above.
- **android-ui:**
  - **Navigation:** both roles reach the bell, the center and the settings. No route is role-gated.
  - **Branch chip and filter:** shown only when `Workspace.managerBranchId == null` (owner). The ViewModel sends `branch_id` only for an owner with a branch chosen (NTM-E09). The workspace is fetched again with `forceRefresh = true` only for an owner whose branch list is empty (NTM-E46).
  - **Settings:** list exactly the types the server returns.
  - **Push:**
    - The app shows what it receives, and only while a user is signed in (NTM-R54).
    - Opening a push for another user's notification gets `404`, which is ignored (NTM-E37).
    - Sign-out and forced sign-out both delete the phone's FCM token, so a phone stops getting the previous user's pushes once FCM reports the token invalid (NTM-R28, NTM-R50).

## 7. Files (changed)

Rows marked **v4** are the new work. The other rows are shipped and listed for the whole picture.

| Repo | Create / change | Why (IDs) |
|---|---|---|
| identity | create `src/main/resources/db/migration/V3__create_push_tokens.sql` (shipped) | `push_tokens` table and enum, unique `token`, indexes on `user_id` and `last_seen_at` (NTM-R25, NTM-R29, NTM-R41) |
| identity | **v4** create `src/main/resources/db/migration/V4__push_tokens_token_length.sql`, or the next free `V<n>` on `origin/dev` | `ALTER TABLE push_tokens ADD CONSTRAINT ck_push_tokens_token_length CHECK (char_length(token) <= 4096);` with a comment citing spec §8. Existing rows are about 160 characters, so nothing fails (NTM-R53) |
| identity | create `notification/entity/PushToken.java`, `PushPlatform.java` (shipped) | Mapped like `Notification.feed` (`PostgreSQLEnumJdbcType`) (NTM-R25) |
| identity | create `notification/repository/PushTokenRepository.java` (shipped) | **Upsert:** native `ON CONFLICT (token) DO UPDATE SET user_id, platform, last_seen_at`.<br>**Deletes:** by `(token, user_id)`, by user, by business (join `users`), and by `last_seen_at < cutoff`.<br>**Lookup:** a user's tokens (NTM-R25, NTM-R28, NTM-R29, NTM-R32, NTM-R33, NTM-R41, NTM-R48) |
| identity | create `notification/service/PushTokenService.java`, `service/impl/PushTokenServiceImpl.java` (shipped) | `register` and `unregister`; `deleteForUser(userId, token)`; `deleteAllOf(userId)` and `deleteAllInBusiness(businessId)` (`Propagation.MANDATORY`); `deleteNotSeenSince(cutoff)` (NTM-R25, NTM-R33, NTM-R41) |
| identity | create `notification/controller/PushTokenController.java` (shipped); **v4** change `notification/dto/RegisterPushTokenRequest.java`, `notification/dto/UnregisterPushTokenRequest.java` | `token` gains `@Size(max = 4096, message = "{validation.max_length}")` next to `@NotBlank`, and `@Schema(maxLength = 4096)`. The existing handler answers `400 VALIDATION_FAILED` with `data.token` = "Use at most 4096 characters." (NTM-R53, NTM-M31) |
| identity | change `notification/service/AlertPreferences.java`, `notification/dto/AlertChannelsRequest.java`, `AlertChannelsResponse.java`, `service/impl/NotificationPreferencesServiceImpl.java` (shipped) | Optional strict-boolean `push`, kept when absent or `null`, returned by GET and PUT. No cross-field rule. No code change in v4 (NTM-R20, NTM-R22, NTM-R42, NTM-R52, NTM-D30) |
| identity | change `notification/service/impl/AlertIntakeServiceImpl.java` (shipped) | Publish `PushDeliveryRequested` for newly inserted rows whose `channels.push()` is on (NTM-R24, NTM-R30, NTM-R40) |
| identity | create `notification/push/PushDeliveryRequested.java`, `PushDelivery.java` (shipped); **v4** change `PushDelivery.java` | **When:** `AFTER_COMMIT`, on a private two-thread daemon scheduler.<br>**Steps:** re-read the row, the user's tokens and the branch name; send; delete `INVALID_TOKEN` rows of that user.<br>**Errors and logs:** catch everything; log ids and counts only.<br>**v4:** `messageOf` builds `PushMessage(notificationId, alertType, branchName)`. The `Messages` dependency is removed from both constructors, and the Javadoc no longer mentions a title (NTM-R30, NTM-R32, NTM-R34, NTM-R40, NTM-R51, NTM-E25) |
| identity | **v4** change `notification/push/PushMessage.java`; delete `notification/push/PushText.java`; change `src/main/resources/messages.properties` | `PushMessage` becomes `(UUID notificationId, AlertType alertType, String branchName)`, with `branchName` null for business-wide. `PushText` has no other caller, so it goes. Remove the keys `notification.push.title.*` and `notification.push.all_branches`; keep `notification.push_token.*` (NTM-R51, NTM-D42) |
| identity | change `notification/push/FcmPushSender.java` (**v4**); `PushSender.java`, `PushSendResult.java`, `PushProperties.java`, `PushConfig.java` (shipped) | **v4:** `toMessage` puts only `notification_id`, `alert_type` and, when not null, `branch_name`. The `title` and `body` lines are removed; the Javadoc ID is updated.<br>**Unchanged:** `sendEach`, data-only, `HIGH`, no collapse key, 10 s timeouts, the error mapping, and the disabled sender without a key (NTM-R32, NTM-R34, NTM-R51) |
| identity | create `notification/job/PushTokenCleanupJob.java` (shipped) | Daily 02:45 Asia/Phnom_Penh, ShedLock `push-token-cleanup`, deletes rows not seen for 60 days (NTM-R41) |
| identity | change `auth/controller/AuthController.java`, `auth/dto/SignOutRequest.java`, `auth/service/AuthService.java`, `auth/service/impl/AuthServiceImpl.java`, `common/keycloak/KeycloakTokenClient.java` (shipped); **v4** comments only in `AuthService.java` | Sign-out deletes the push token after an accepted logout, a replayed one included; blank counts as none; always `200`. **v4:** no behaviour change; `AuthService` Javadoc cites NTM-R48 instead of NTM-R43 (NTM-R28, NTM-R48, NTM-E42, NTM-E48) |
| identity | change `manager/service/impl/ManagerServiceImpl.java`, `manager/service/ManagerRemoval.java`, `user/service/impl/UserServiceImpl.java`, `business/service/impl/BusinessDeletionServiceImpl.java` (shipped) | Token deletion in the same transaction as each change (NTM-R33) |
| identity | change `build.gradle`, `.env.example`, `consul/identity-service.yml`, `k8s/40-identity-service.yaml`, `k8s/DEPLOY.md`, `README.md`, `src/test/resources/application-it.yml` (shipped) | Admin SDK; `FCM_SERVICE_ACCOUNT_BASE64` (empty = push off); `ledgervision.push.fcm-service-account`; job cron; optional secret `camfinsight-fcm` (NTM-D16, NTM-R41). **v4:** the README's push section lists the three data keys if it lists keys today |
| identity | **v4** tests, each name starting with the ID | **`KeycloakTokenClientTest`:** rename NTM-R43 → NTM-R48 and NTM-E38 → NTM-E42 (4 tests).<br>**`AuthControllerIT`:**<br>- Rename the 4 NTM-R43 tests to NTM-R48, the 2 NTM-E38 tests to NTM-E42, and the "blank or null" one to "NTM-E48 NTM-D45 NTM-R48 …".<br>- Add "NTM-R48 NTM-E42 NTM-D34 a replayed logout of an ended session, which Keycloak accepts, deletes the caller's push token" (`logout` → `true` for a refresh token signed out before).<br>- Add "NTM-E47 NTM-D35 a sign-out with an invalid or expired bearer is 401 before the endpoint: no logout sent, no push token deleted".<br>**`NotificationPreferencesIT`:** add "NTM-R52 NTM-D44 `\"push\": null` keeps the stored value, and a type with none stored stays without a push key".<br>**`PushTokenControllerIT`:** add "NTM-R53 NTM-M31 a 4096-character token is saved; 4097 is 400 on `token` with \"Use at most 4096 characters.\" and nothing is stored", and the same for unregister (nothing deleted).<br>**`PushTokenRepositoryIT`:** add "NTM-R53 the database refuses a token over 4096 characters".<br>**`FcmPushSenderTest`:** rename NTM-R31 → NTM-R51; assert the `data` key set equals `{notification_id, alert_type, branch_name}`, or `{notification_id, alert_type}` for business-wide.<br>**`PushDeliveryTest` / `PushDeliveryIT`:** rename NTM-R31 → NTM-R51; assert no `title` / `body`.<br>**`PushConfigTest`:** drop NTM-R31 from the name (keep NTM-D16, add NTM-R34).<br>**`PushTextTest`:** delete, along with `PushText` |
| android | change `app/build.gradle.kts`, `e2e/android/*`, `gradle/libs.versions.toml`, root `build.gradle.kts`, `core/build.gradle.kts`, `.gitignore`, `local.properties.example`, `README.md` (shipped) | Debug app id `.dev`, Firebase wiring only when `google-services.json` exists, `PUSH_FAKE_TOKEN` (NTM-R47, NTM-D16, NTM-D29) |
| android | change `core/.../navigation/AppDestinations.kt`, `core/.../data/entity/DataState.kt`, `core/.../network/RemoteDataSource.kt` (shipped) | Destinations; field errors (NTM-R03, NTM-R19, NTM-R35, NTM-R45) |
| android | change `core/.../network/AuthInterceptors.kt` (shipped in d2b8509) | `/auth/sign-out` in `publicPaths`: no bearer, no refresh (NTM-R49, NTM-E47) |
| android | create `core/.../push/PushTokenSource.kt`, `PushTokenStore.kt`, `PushSignals.kt` (shipped); **v4** change `core/.../push/PushTokenSource.kt` | **v4:** gains the top-level `const val PUSH_TOKEN_DELETE_TIMEOUT_MILLIS = 5_000L`, moved from `features/.../profile/data/local/DeviceSignOut.kt` so sign-out and forced sign-out share it (NTM-R28, NTM-R50) |
| android | create `app/.../push/FirebasePushTokenSource.kt`, `AppMessagingService.kt`, `PushModule.kt`, `ic_stat_notification.xml`; change `AndroidManifest.xml`, `StarterApplication.kt`, `MainActivity.kt`, `ui/navigation/MainGraph.kt`; create `app/src/debug/.../DebugPushReceiver.kt` (shipped) | Firebase only in `:app`; channel; foreground tracking; tap → center; simulator (NTM-R25, NTM-R35, NTM-R36, NTM-R37, NTM-E18, NTM-E24). No v4 change: `PushTokenSync.start` (below) also starts the expiry collector |
| android | create `features/.../notification/data/{api,dto,mapper,service,repository}/*` (shipped) | Retrofit, DTOs, mappers, repositories (NTM-R02 … NTM-R26) |
| android | create `features/.../notification/data/push/PushTokenSync.kt` (shipped); **v4** change | **v4:** `start(scope)` also launches `sessionStore.expired.collect { forgetPhoneToken() }`. `forgetPhoneToken()`, under the same `lock`, runs `withTimeoutOrNull(PUSH_TOKEN_DELETE_TIMEOUT_MILLIS) { tokenSource.deleteToken() }` and then `repository.keepToken(null)`. It makes no network call (no `/auth/sign-out`, no unregister). Because it shares the Mutex, a sign-in right after waits for it and then registers the fresh token. An offline refresh (`RefreshResult.Failed`) doesn't call `expire()`, so it deletes nothing (NTM-R50, NTM-E43, NTM-D36) |
| android | create `features/.../notification/data/push/PushMessageHandler.kt`, `SystemNotifier.kt` (shipped); **v4** change `PushMessageHandler.kt` | **v4:** inject `SessionStore`. The first line of `handle` returns when `sessionStore.currentUser.value == null`: nothing posted, no signal, no count reload (NTM-R54, NTM-D41). Javadoc IDs move from NTM-R31 to NTM-R51; the same in `SystemNotifier.kt`. Unchanged: foreground → signal and count; background → post when enabled (NTM-R36, NTM-R37, NTM-R51, NTM-E10, NTM-E16) |
| android | create `features/.../notification/domain/**` (shipped) | Models, `NotificationTime` (Asia/Phnom_Penh), use cases (NTM-R05, NTM-R06, NTM-E31) |
| android | create `features/.../notification/presentation/model/*`, `components/*`, `screen/*` (shipped) | Center and settings UI. `NotificationTabs` scrolls sideways (NTM-R55) |
| android | create `features/.../notification/presentation/viewmodel/NotificationsViewModel.kt` (shipped); **v4** change | **NTM-E44, `markAll()`:**<br>- At the tap, capture `val tapped = query()` and `val snapshot = unread ids of the loaded cards` (`NotificationListReducer.markAllRead(items).wasUnread`), and send `markAllRead(tapped)`.<br>- `finishMarkAll(count, tapped, snapshot)` always sets `undoSnapshot = snapshot` and emits `MarkedAllRead(count)` (NTM-M04 with Undo).<br>- It marks the on-screen cards read only when `query()` still equals `tapped` and the list is a `Success`.<br>- Otherwise it leaves the list alone and starts `loadFirstPage(quiet = true)`, so the new tab shows what the server now holds.<br>**NTM-E46, `loadWorkspace(forceRefresh)`:**<br>- At init, call `getWorkspace()`. When the result is an owner workspace with no branches, call `getWorkspace(forceRefresh = true)` once.<br>- On `Refresh`, when `isOwner` and `branches` is empty, or when the workspace never loaded, call `loadWorkspace(forceRefresh = true)`.<br>- Failures stay silent; the chip keeps "All branches" |
| android | change `features/.../profile/data/local/DeviceSignOut.kt`, `features/.../profile/domain/service/ProfileService.kt` (shipped); **v4** | **v4:** use `PUSH_TOKEN_DELETE_TIMEOUT_MILLIS` from `:core`. Javadoc IDs move from NTM-E28 / NTM-E38 / NTM-R43 to NTM-R49 / NTM-E42 / NTM-R48. No behaviour change (NTM-R28, NTM-R49, NTM-E20, NTM-E42) |
| android | change `features/src/main/res/values/strings.xml` (shipped) | Every live message: NTM-M01 … NTM-M05, NTM-M07 … NTM-M29 (NTM-M28, NTM-M30 and NTM-M31 are server text) (NTM-R39) |
| android | change `features/.../di/*`, `dashboard/**`, `profile/**`, move `LoadMoreOnScroll` (shipped) | Bindings, bell, settings row, paging |
| android | **v4** tests, each name starting with the ID (fakes in `features/src/test/.../notification/fake/`) | **`AuthInterceptorsTest`:** rename to "NTM-R49 NTM-E47 NTM-R28 sign-out carries no bearer token and never triggers a refresh".<br>**`SignOutPushTokenTest`:** "NTM-E28 only the refresh token…" → "NTM-R49 NTM-R28 …"; "NTM-E38 NTM-R28 when the server rejects…" → "NTM-E42 NTM-R28 …".<br>**`PushMessageHandlerTest`:**<br>- Rename NTM-R31 → NTM-R51 (3 tests).<br>- Add "NTM-R54 NTM-D41 a push while no one is signed in posts nothing, sends no signal and reloads no count" (both foreground and background).<br>- Add "NTM-R51 extra `title` and `body` keys are ignored: the title is the app's own".<br>**`PushTokenSyncTest`:**<br>- Add "NTM-R50 NTM-E43 a rejected refresh deletes the phone's token and forgets the kept one, with no server call".<br>- Add "NTM-R50 a sign-in right after a forced sign-out registers the fresh token" (`FakeSessionStore.expire()` emits on `expired`).<br>**`NotificationsViewModelTest`:**<br>- Add "NTM-E44 NTM-D38 switching tab while Mark all read is sending: the request keeps the tapped tab and branch, NTM-M04 with Undo shows, the new tab's cards are not marked, and Undo restores the tapped tab's unread cards".<br>- Add "NTM-E46 NTM-D40 an owner whose branch list failed sees only All branches, with no error, and pull-to-refresh fetches it again with forceRefresh".<br>- Add "NTM-E46 reopening the center fetches an empty branch list again" (`FakeWorkspaceRepository` records `forceRefresh`).<br>**`NotificationTimeTest`:** "NTM-R06 a card shows…" → "NTM-R06 NTM-E31 …", plus the UTC-input assertion |
| camfinsight-docs | **v4** change `api/identity-service.md` (lead) | **Table:**<br>- `push-tokens` row: 📝 (v4.0.0: `token` at most 4096) and spec IDs NTM-R25, NTM-R29, NTM-R41, NTM-R53.<br>- `unregister` row: 📝, NTM-R25, NTM-R53.<br>- GET preferences: spec IDs NTM-R19, NTM-R20, NTM-R42, NTM-R46 (drop the withdrawn NTM-R21).<br>- PUT preferences: 📝 (v4.0.0: `"push": null` = absent), NTM-R22, NTM-R42, NTM-R52.<br>- Sign-out: 📝 (v4.0.0), NTM-R28, NTM-R48, NTM-R49.<br>**push-tokens section:** error row `400 VALIDATION_FAILED`, `token` over 4096 characters → `data.token` "Use at most 4096 characters.".<br>**unregister section:** the same row.<br>**PUT preferences section:** "When `push` is absent or `null` …" (NTM-D44).<br>**Sign-out section:** rewritten as §3:<br>- no `Authorization` header; an invalid or expired bearer → `401` before the endpoint;<br>- a blank or whitespace-only `push_token` counts as none;<br>- deleted when Keycloak accepts the logout, a replayed logout of an ended session included;<br>- a rejected refresh token deletes nothing;<br>- drop the old "Implementation note".<br>**New section "FCM data message (push) — 📝 planned (v4.0.0)":** data-only, `HIGH`, no collapse key, keys exactly `notification_id`, `alert_type`, `branch_name` (left out when business-wide), never `title` / `body` / `message`.<br>**Statuses:** 🚧 when part F is on its branch, ✅ when it merges |
| camfinsight-docs | **v4** change `data-dictionary/notifications.md` (lead) | `push_tokens.token`: Length `4096`; description adds "at most 4096 characters (`ck_push_tokens_token_length`)". Spec §8 state moves from proposed to done when F merges |

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
| NTM-E18 | android `MainGraph` keeps `pendingOpen` through sign-in (a notification already in the tray). Pushes arriving while signed out are never posted (`PushMessageHandler`, NTM-R54). The ViewModel ignores a `404` on the opening read |
| NTM-E19 | android ViewModel: a `404` on the opening read is ignored; the center loads All |
| NTM-E20 | android `DeviceSignOut` deletes the FCM token even when the sign-out call fails; identity drops the stale token later (NTM-R32 on the next send, or NTM-R41) |
| NTM-E21 | android `PushTokenSync.onNewToken` registers at once; identity deletes the old token when FCM reports it invalid |
| NTM-E22 | identity `PushDelivery` sends to every token of the user; there is no cross-device cancel |
| NTM-E23 | identity: push is decided at intake time from the stored preference; no client handling |
| NTM-E24 | android `MainActivity.onCreate` reads the extra on a cold start; the session is restored, then `pendingOpen` opens the center |
| NTM-E25 | identity `PushDelivery`: a sender failure is caught after commit; the notification and Telegram are unaffected (`PushDeliveryIT`) |
| NTM-E26 | android `NotificationSettingsViewModel`: the switch reverts; NTM-M03 |
| NTM-E27 | android `AlertPreferenceRow`: In-app is on and disabled when `locked` |
| ~~NTM-E28~~ | Withdrawn in v4.0.0 → NTM-E43 |
| NTM-E29 | identity `PushTokenRepository` upsert on `token` (`PushTokenControllerIT`) |
| NTM-E30 | identity (NTF scope); android opens the center on All with a fresh load |
| NTM-E31 | android `NotificationTime` ignores the phone's zone for day groups and card times; tested with the default zone set to America/Los_Angeles |
| NTM-E32 | identity `AlertPreferences` defaults, with push = the Telegram default; android shows them as returned |
| NTM-E33 | android settings ViewModel: saves are queued one after another (Mutex); each `PUT` carries the matrix at send time; a failure reverts only its own switch |
| NTM-E34 | identity read is idempotent; android repeats are harmless |
| NTM-E35 | android: tapping a push in the tray doesn't depend on the current Push setting |
| NTM-E36 | android `NotificationCard`: time and branch sit in a wrapping row and the branch shows in full; the phone shortens the push text |
| NTM-E37 | identity answers `404` for another user's notification (NTF-E20); android ignores it on the opening read |
| ~~NTM-E38~~ | Withdrawn in v4.0.0 → NTM-E42 |
| NTM-E39 | android `NotificationsViewModel`: an opening-read failure other than `404` is silent; the card stays unread and the list shows its own state |
| NTM-E40 | android `NotificationListReducer`: mark-all enabled = some loaded card of the current tab is unread; unloaded pages are not considered |
| NTM-E41 | android settings ViewModel and `PreferenceSaveError`: the `telegram_needs_in_app` field error reverts the In-app switch and shows that message (NTM-M30) |
| NTM-E42 | identity `AuthServiceImpl`: a rejected logout (`logout` → `false`) deletes nothing and answers `200`; an accepted one, a replayed logout of an ended session included, deletes the caller's `push_token` (`AuthControllerIT`, `KeycloakTokenClientTest`, api-qa NTM-E42-M2 on real Keycloak). android `DeviceSignOut` deletes the FCM token either way (`SignOutPushTokenTest`) |
| NTM-E43 | android `TokenRefreshAuthenticator` → `SessionStore.expire()` → sign-in (existing). `PushTokenSync` collects `expired` and deletes the FCM token and the kept token, with no sign-out call. identity drops the stale server row on its next send (NTM-R32) |
| NTM-E44 | android `NotificationsViewModel.markAll`: the query and the Undo snapshot are taken at the tap; on-screen marking happens only if the tab and branch are unchanged, otherwise the new tab reloads quietly; NTM-M04 with Undo shows either way |
| NTM-E45 | android: `today` is set on each first-page load (`showFirstPage`), and the groups and card times use `state.today`; nothing ticks at midnight |
| NTM-E46 | android `NotificationsViewModel.loadWorkspace`: a failed branch list leaves only "All branches", with no error; it is fetched again with `forceRefresh` on pull-to-refresh and on reopening |
| NTM-E47 | android `AuthInterceptors` never puts a bearer on `/auth/sign-out`; identity's resource-server filter answers `401` to a bad one (`AuthControllerIT`) |
| NTM-E48 | identity `AuthServiceImpl.signOut`: a `null`, blank or whitespace `push_token` deletes nothing; `200` |
| NTM-E49 | android `StarterApplication` foreground tracking (activity callbacks); accepted, no code. The bell and center show the notification on the next reload |

## 9. Packages and verification (changed)

**Packages:** none new in v4.0.0. Those added in parts A–E (Firebase BoM 34.x and `firebase-messaging` in android `:app`, the google-services plugin 4.4.x, and `firebase-admin` 9.x in identity) are unchanged.

**Firebase setup (NTM-D16, NTM-D29):** unchanged.
- **App ids:** debug `com.kshrd.starter.dev`, release `com.kshrd.starter`.
- **The handed-out file:** `google-services.json`, never committed.
- **Without the file:** push is off and no token is registered, unless `push.fakeToken=true` in debug.
- **identity:** `FCM_SERVICE_ACCOUNT_BASE64` in `.env`; empty = a disabled sender.

**Verification per repo (v4.0.0):**
- **identity-service** (worktree `ledgervision-identity-service-ntm`, after merging `origin/dev`):
  - `./gradlew build` (unit tests plus Testcontainers ITs, Docker running). The new migration applies on a fresh DB with `ddl-auto: validate`.
  - `./gradlew bootRun` with no key and the shared infra (real Keycloak):
    - **Sign-out with a garbage `refresh_token`:** `200`, no row deleted (NTM-E42).
    - **Replayed logout:** sign in, register a token, sign out, then sign out again with the same refresh token and `push_token`: `200`, and the row is deleted (NTM-R48, NTM-E42).
    - **Bad bearer:** sign-out with `Authorization: Bearer garbage` → `401`, no row deleted (NTM-E47). If it answers `200`, report it as a spec gap; don't change the server.
    - **Blank token:** sign-out with `"push_token": "   "` → `200`, nothing deleted (NTM-E48).
    - **Token cap:** register with a 4097-character token → `400`, `data.token` "Use at most 4096 characters."; 4096 → `200` (NTM-R53).
    - **Null push:** `PUT /notifications/preferences` with `"push": null` on one type → `200`, that type's stored `push` unchanged (NTM-R52).
  - **Push data:** with the fake sender in ITs (and with a key, on dev), the data keys are exactly `notification_id`, `alert_type` and `branch_name` (NTM-R51).
  - The `api/identity-service.md` entries match the code (dictionary-checker clean against the new migration).
- **android-ui:**
  - Without `google-services.json`: `./gradlew assembleDebug test checkCode checkKtlint`.
  - `e2e/android/run-all.sh` on the `.dev` build.
- **web-ui:** none (checked: no bearer on sign-out).

**v4.0.0 QA (after F and G are on `dev`):**
- **api-qa** (identity, real Keycloak, fake sender):
  - NTM-R48-M1, NTM-R48-M2 (replayed logout deletes).
  - NTM-E42-M1 (garbage), NTM-E42-M2 (replay).
  - NTM-E47-M1 (bad bearer → 401).
  - NTM-E48-M1 (blank and spaces).
  - NTM-R52-C1 (`null` keeps the value; nothing stored stays unwritten).
  - NTM-R53-V1 (4096 ok, 4097 refused, both endpoints, nothing stored or deleted).
  - NTM-R51-C1 (data keys).
  - Re-check NTM-R43-M2 and NTM-E38-M2 from the last report as NTM-R48-M2 and NTM-E42-M2.
- **android-qa** (`.dev` build, `push.fakeToken=true`, T27 simulator):
  - **NTM-R50-M1 / NTM-E43-M1:** force a rejected refresh by ending the user's sessions in Keycloak admin (Users → Sessions → Sign out) and waiting for the access token to expire (realm `accessTokenLifespan` 1800 s), or by editing the stored tokens of a kept session with `adb shell run-as com.kshrd.starter.dev`. Then:
    - the app opens sign-in;
    - the OkHttp log has no `/auth/sign-out`;
    - `registered_token` is gone from `shared_prefs/push.xml`;
    - the next sign-in registers again.
  - **NTM-R54-M1:** sign out, then simulate a push; no system notification.
  - **NTM-E18-M2:** a notification posted before sign-out and tapped after → sign-in, then the center.
  - **NTM-R55-C1:** at font scale 1.5 and 2.0, the tab row scrolls and "Stock" is readable in full after a swipe.
  - **NTM-E44-C1:** owner, Stock tab, tap "Mark all read" and switch to All at once. NTM-M04 with Undo shows; All's non-stock cards stay unread; Undo restores the stock ones.
  - **NTM-E45-C1:** turn off auto time, set the phone to 23:59 Asia/Phnom_Penh with the center open, wait past midnight. The headers don't change; pull-to-refresh moves them.
  - **NTM-E46-M1:** block `GET /branches` (gateway or proxy) or take the device offline after `/me`. The chip offers only "All branches" with no error; restore, pull to refresh, and the branches appear.
  - **NTM-E49-M1:** simulate a push 1–2 s after Home. It may post nothing; the bell and center show it on the next reload.
  - **Renamed IDs:** NTM-R06-C2 and NTM-E31-C1 (phone zone differs), NTM-R49-M1 (sign-out with an expired access token still deletes the server row).
- **Still blocked until both NTM-D16 files are handed out (real FCM):**
  - NTM-R30 and NTM-R51 end to end;
  - NTM-R36 in the background and in Doze;
  - NTM-R37 on a real push;
  - NTM-R32 by a real FCM answer;
  - NTM-R50 (real FCM token deleted, then the server drops it on the next send);
  - NTM-E20, NTM-E21, NTM-E22, NTM-E23.
- **Test names start with the ID:**
  - identity: `@DisplayName("NTM-R53 …")`.
  - android: `` fun `NTM-R54 …`() ``.
  - Every live R has a test or one of the QA scenarios above. NTM-R39 is checked by review and on the device.

## 10. Questions and conflicts (changed)

- **Q1 · NTM-R54 and a session not kept.** With "Keep me signed in" off, the session lives only in memory (`PreferencesSessionStore`: `remember = false`). After Android ends the app's process, which is common in the background, an FCM message starts the process with `currentUser == null`. Under NTM-R54 as written, that user then gets **no system notification at all** until they open the app and sign in again. Their token stays registered on the server, because they never signed out, so pushes keep arriving and being dropped.
  - *Options:*
    - (a) Accept: NTM-R54 as written. Push works for a session that isn't kept only while the app process is alive.
    - (b) For push, a phone counts as signed in while it holds a registered token for a user. `registered_token` is cleared only by sign-out and forced sign-out (NTM-R28, NTM-R50), not by the process ending. A user who didn't keep the session still gets pushes; tapping one opens sign-in, then the center (NTM-E18). Pushes after a real sign-out stay dropped. Needs `/sdd:amend` of NTM-R54 / NTM-D41.
  - *Suggested:* none. (a) is simplest. (b) keeps push useful for users who don't keep the session.
  - **Blocks:** only the NTM-R54 rows of part G (§7 `PushMessageHandler`, its test, QA NTM-R54-M1).
  - **Answered 2026-11-16 by Vuththana: (a).** NTM-R54 as written; part G's NTM-R54 rows are unblocked.

## Changes

| Date | Change | Why | Approved by |
|---|---|---|---|
| 2026-11-16 | §1, §2, §3, §4, §7, §8, §9, §10 changed for spec v3.0.0: NTM-D29 … NTM-D33, NTM-R43 (changed), NTM-R44 … NTM-R47, NTM-E38 … NTM-E41, NTM-M09 code, NTM-M29, NTM-M30; NTM-R21, NTM-D05, NTM-M06 withdrawn | Plan §10 Q1 … Q7 answered through /sdd:amend (v3.0.0, be5af46) | Vuththana |
| 2026-11-16 | §1–§10 changed for spec v4.0.0: NTM-R43 → R48, NTM-E38 → E42, NTM-E28 → E43, NTM-R31 → R51 withdrawn and replaced; NTM-R49, R50, R52, R53 (+ NTM-M31), R54, R55 and NTM-E44 … E49 added; NTM-R06, E31, E18, E37, R28, D07 reworded. Sign-out and FCM data contracts rewritten; new parts F (identity), G (android), H (docs); `push_tokens.token` ≤ 4096 with a CHECK migration; plan §10 Q1 on NTM-R54 | /sdd:amend v4.0.0 (565f87e), answering verify and QA gaps NTM-Q15 … NTM-Q30 | Vuththana |
