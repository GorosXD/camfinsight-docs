# Plan: Notifications on mobile (center and push)

**Spec:** `NTM` v6.0.0, commit `f4c8dc8` · **Approved:** 2026-11-17 by Vuththana
**Repos:** ledgervision-identity-service (backend, api-dev; worktree `ledgervision-identity-service-ntm`, branch `feature/notification-mobile`: parts A–B merged into `dev` by PR #42; part F v4.0.0 committed as T34–T37, not pushed), ledgervision-android-ui (Android, android-dev; branch `feature/notification-mobile`: parts C–E merged into `dev` by PR #4; part G committed as T38–T42, not pushed), ledgervision-web-ui (checked only, no change), camfinsight-docs (lead: `api/`, data dictionary, spec §8 / §9 states)

## 1. Scope (changed)

- **In:**
  - Spec §1 "In", and journeys §4a–§4e.
  - NTM-R01 … NTM-R57 and NTM-E01 … NTM-E56, except the withdrawn ones: NTM-R21 (with NTM-D05 and NTM-M06), NTM-R31, NTM-R43, NTM-R53 (with NTM-D46 and NTM-M31), NTM-E28, NTM-E38.
  - The §8 data changes:
    - `users.notification_preferences.push`;
    - the new `push_tokens`;
    - (v5.0.0) `push_tokens.token`: at most 2048 characters, and only `A–Z a–z 0–9 _ : . -`. A new migration after V4 sets this. V4 (4096) is already applied on the shared dev DB, so it stays as it is.
  - The new and changed endpoints in §9, and the FCM data message keys.
  - The Firebase setup (NTM-D16, NTM-D29): a debug app id `com.kshrd.starter.dev`, and builds and tests that work without the Firebase files.
  - Device checks of push without real FCM, through a debug-only simulator (§9).
  - **v5.0.0 work:** everything in parts A–E and in v4.0.0 parts F–G stays. This change:
    - adds the token rules (NTM-R56, NTM-R57, NTM-E53, NTM-E56) to part F;
    - adds tests only to part G: NTM-E50, NTM-E51, NTM-E52 and NTM-E54 describe Android behaviour that already exists;
    - adds the 2048 and character edits to part H.
  - The per-ID table in §2 says, for each v5 ID, whether it needs code or only a test or QA scenario.
- **Out:**
  - Everything in spec §1 "Out": NTF alert rules and texts, new alert types, opening the related record, Telegram linking changes, email settings, iOS, sounds, quiet hours.
  - Web-ui code:
    - Its `signOut` already sends no `Authorization` header (NTM-D35; see §2).
    - Its preferences calls stay unchanged (NTM-D28).
    - It never registers push tokens.
  - Android code for v5.0.0:
    - Real FCM tokens use only `A–Z a–z 0–9 _ : -` and are about 160 characters.
    - The debug fake token `debug-<uuid>` uses only letters, digits and `-`.
  - Editing `V4__push_tokens_token_length.sql`, including its comment.
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
| identity-service | api-dev | **F.** **v4.0.0 (committed, T34–T37):**<br>- the push data drops `title` and `body` (NTM-R51);<br>- sign-out tests (NTM-R48, NTM-E42, NTM-E47, NTM-E48);<br>- the `"push": null` test (NTM-R52);<br>- `token` ≤ 4096 with `V4`, now withdrawn (NTM-R53), but V4 stays applied.<br>**v5.0.0 (new):**<br>- a `V5` migration: delete bad rows, cap at 2048, add a character check (NTM-R56, NTM-R57, NTM-E53);<br>- `@Size(max = 2048)` and `@Pattern` on both push-token DTOs (NTM-R56, NTM-R57, NTM-E56);<br>- tests move from NTM-R53 to NTM-R56 / NTM-R57, plus a migration IT (NTM-E53) | Same worktree and branch, on top of T37. One PR into `dev` carries v4 and v5. **§10 Q2 blocks only the DTO validation order and the whitespace assertions of the NTM-R25 blank-token tests.** The V5 migration and every other v5 test can start now. Android needs nothing from F |
| android-ui | android-dev | **G.** **v4.0.0 (committed, T38–T42):** forced sign-out (NTM-R50), pushes dropped while signed out (NTM-R54), mark all while switching (NTM-E44), branch list re-fetch (NTM-E46), renames.<br>**v5.0.0 (new, tests only):** NTM-E50, NTM-E51, NTM-E52 and NTM-E54 in `PushTokenSyncTest` and `NotificationsViewModelTest`, and one option in `FakePushTokenSource` | Parallel with F, on the same branch, on top of T42. No production change. One PR into `dev` carries v4 and v5 |
| web-ui | — | None. NTM-R49 / NTM-D35 checked: `signOut` in `src/lib/api/auth.ts` calls `callGateway("/auth/sign-out", { method: "POST", body: { refresh_token } })` with no `token`, and `callGatewayWithMessage` adds `Authorization` only when a `token` is passed. Web-ui calls neither push-token endpoint, so NTM-R56 and NTM-R57 don't reach it | — |
| camfinsight-docs | lead | **H.** **v4.0.0:** already applied to `api/identity-service.md` (🚧 rows) and `data-dictionary/notifications.md` (4096).<br>**v5.0.0 (new):** the push-tokens and unregister rows and sections move to 2048 and the characters (NTM-R56, NTM-R57, NTM-M32, NTM-M33, NTM-E56). `push_tokens.token` becomes Length 2048 with the character rule and both checks (§7) | With F. The two push-token rows go back to 📝 now, 🚧 when V5 and the DTOs are on the branch, and ✅ when F merges into `dev`, together with the spec §8 state |
| both | api-qa, android-qa | The QA scenarios named in §9: "v4.0.0 QA" (with NTM-R53-V1 replaced) and "v5.0.0 QA" | Last, after F and G merge into `dev` and the identity build is deployed on dev |

### v5.0.0, per amended ID (new)

| ID | Already built? | Work |
|---|---|---|
| NTM-R56, NTM-M32, NTM-D49 (replace NTM-R53, NTM-M31, NTM-D46) | **Not built.** Both DTOs have `@Size(max = 4096)` (8fdd4b3), and V4 has `CHECK (char_length(token) <= 4096)` | **New code**, identity: `V5` (§7) and `@Size(max = 2048)`. The NTM-R53 tests in `PushTokenControllerIT` and `PushTokenRepositoryIT` are rewritten as NTM-R56 |
| NTM-R57, NTM-M33, NTM-D50 | **Not built.** There is no character rule. `validation.invalid_value` = "Enter a valid value." already exists in `messages.properties` | **New code**, identity: `@Pattern` on both DTOs and the `V5` character check. New NTM-R57 tests |
| NTM-E53, NTM-D54 | **Not built** | **New code**, identity: the `DELETE` at the top of `V5`. New `PushTokenCapMigrationIT`, modelled on `OneManagerMigrationIT` |
| NTM-E56 | **Not built** (it follows from NTM-R57). `GlobalExceptionHandler.handleValidation` keeps the first field error it reads (`putIfAbsent`), and Hibernate Validator's order isn't fixed, so the combined case gets either message, as the spec allows | Identity: no code beyond NTM-R57. Add one test that accepts either message |
| NTM-E50, NTM-D51 | **Built.**<br>- `PreferencesSessionStore` restores no user when `remember` is false.<br>- `PushMessageHandler` drops a push while `currentUser` is null (T39, test "NTM-R54 NTM-D41 …").<br>- `PushTokenSync` sends nothing and deletes nothing while no one is signed in, so `registered_token` and the server row stay | Android **test only**: `PushTokenSyncTest` (§7). QA NTM-E50-M1 |
| NTM-E51, NTM-D52 | **Built.** `isMarkAllEnabled` is `!isMarkingAll && …`, and `isMarkingAll` belongs to the screen, not the tab, so it stays true across a tab or branch switch until the answer | Android **test only**: `NotificationsViewModelTest` (§7). QA NTM-E51-C1 |
| NTM-E52, NTM-D53 | **Built** (T41). A failed `GET /auth/me` leaves `isOwner` false (no chip, no `branch_id`) and `workspaceLoaded` false, with no effect. `Refresh` calls `loadWorkspace(forceRefresh = true)` while `!workspaceLoaded`. The existing test "NTM-E46 pull-to-refresh fetches the workspace again when it never loaded …" already exercises it, but names NTM-E46 | Android **test only**: a new test naming NTM-E52 that also asserts no error and no `branch_id` (§7). QA NTM-E52-M1 |
| NTM-E54, NTM-D55 | **Built** (T38):<br>- `forgetPhoneToken` runs `withTimeoutOrNull(5 s) { deleteToken() }` and then always `keepToken(null)`.<br>- `PushTokenSource.deleteToken` never throws (`awaitOrNull`).<br>- Nothing retries.<br>- The next sign-in registers the phone's current token (the same one), which the server moves (NTM-D21).<br>- Pushes in between are dropped (NTM-R54) | Android **test only**: `PushTokenSyncTest` with a delete that never finishes (§7). QA NTM-E54-M1 (fake-token build, whose delete is a no-op). Real FCM: added to the blocked list in §9 |

### v4.0.0, per amended ID (built in T34–T42)

| ID | Done in | Notes |
|---|---|---|
| NTM-R48 (replaces R43), NTM-E42 (replaces E38), NTM-E47, NTM-E48 | T34 (292d83f), T38 (8e6d473) | Test renames and new `AuthControllerIT` cases. No sign-out behaviour change |
| NTM-R49 | T38 | Android test renames. `/auth/sign-out` was already in `publicPaths` (d2b8509) |
| NTM-R50, NTM-E43 (replaces E28) | T38 | `PushTokenSync` collects `sessionStore.expired` |
| NTM-R51 (replaces R31) | T37 (cd7eb64), T39 (547c40b) | The server sends exactly the three keys. Android test renames |
| NTM-R52 | T36 (393aa75) | Test only |
| ~~NTM-R53~~, ~~NTM-M31~~ | T35 (8fdd4b3) | **Withdrawn in v5.0.0** → NTM-R56 / NTM-R57 (table above). V4 stays applied |
| NTM-R54 | T39 | `PushMessageHandler` returns when no one is signed in |
| NTM-R55, NTM-E45, NTM-E49 | — | Built before v4. QA scenarios only |
| NTM-R06, NTM-E31 | T42 (e1b1441) | Test rename and a UTC-input assertion |
| NTM-E44 | T40 (f839444) | Query and Undo snapshot taken at the tap |
| NTM-E46 | T41 (dab61c1) | `loadWorkspace(forceRefresh)` |
| NTM-E18, NTM-E37, NTM-R28, NTM-D07 | — | Reworded only |

## 3. API contract (changed)

All endpoints go through the gateway: `/api/v1/notifications/**` and `/api/v1/auth/**` already route to identity-service. Every response uses the usual envelope `{ success, code, message, data, timestamp }`. Both roles may call every endpoint below, and each call only touches the caller's own data, so no endpoint has a `403` case.

### `GET /api/v1/notifications`: exists (NTF), unchanged · NTM-R03, NTM-R04, NTM-R05 … NTM-R12, NTM-E09, NTM-E52
- **Who:** Owner, Manager. A manager's `branch_id` is ignored by the server (NTF-R73), and the app never sends one. The app also sends none while the caller's workspace hasn't loaded (NTM-E52).
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

### `POST /api/v1/notifications/read-all`: exists, unchanged · NTM-R15, NTM-D12, NTM-E40, NTM-E44, NTM-E51
- **Body (optional):** `{ feed?, alert_type?, branch_id? }`.
  - The app sends the `feed` or `alert_type` of the tab **at the moment of the tap**, plus the owner's `branch_id` at that moment (NTM-D38).
  - On All and Unread it sends no filter except `branch_id`.
  - Only one is in flight at a time: the button is disabled on every tab until it answers (NTM-D52).
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

### `POST /api/v1/notifications/push-tokens`: new (changed) · NTM-R25, NTM-R27, NTM-R29, NTM-R41, NTM-R56, NTM-R57, NTM-D21, NTM-D23, NTM-D49, NTM-D50, NTM-E56
- **Who:** Owner, Manager (signed in).
- **Body:**
  - `token` (dictionary `push_tokens.token`):
    - required and not blank;
    - **at most 2048 characters** (NTM-R56);
    - **only `A–Z`, `a–z`, `0–9`, `_`, `:`, `.` and `-`** (NTM-R57).
  - `platform` (`push_tokens.platform`): required, `ANDROID`.
- **200 OK** `data`: `null`. The server upserts on `token`: a token held by another user moves to the caller, and `last_seen_at` is set to now on every call.
- **Errors:** `401`, and `400 VALIDATION_FAILED`. Nothing is stored in any of these cases:
  - `token` missing, `null` or empty → `data.token` "This field is required.". A whitespace-only `token` gets the answer chosen in §10 Q2.
  - `token` over 2048 characters → `data.token` "Use at most 2048 characters." (NTM-M32).
  - `token` with any other character (a space, a non-ASCII letter, `/`, `+`, `=` …) → `data.token` "Enter a valid value." (NTM-M33).
  - `token` both over 2048 and with another character → `data.token` is one of NTM-M32 or NTM-M33, whichever the validator reports first (NTM-E56).
  - `platform` missing or not `ANDROID` → `data.platform`.
- The database refuses both too (`ck_push_tokens_token_length`, `ck_push_tokens_token_chars`, §5). A token that passes the DTO always fits the unique index, so it never fails with `500`.
- The app never shows these errors (NTM-R27).

### `POST /api/v1/notifications/push-tokens/unregister`: new (changed) · NTM-R25, NTM-R56, NTM-R57, NTM-E56
- **Body:** `token`, with the same rules as for register: required, not blank, at most 2048 characters, the same characters only.
- **200 OK** `data`: `null`. Only the caller's own row is deleted; a token that isn't the caller's, or doesn't exist, is ignored (still `200`).
- **Errors:** `401`, and `400 VALIDATION_FAILED` on `data.token`, with the same messages and rules as register (NTM-M32, NTM-M33, NTM-E56, §10 Q2). Nothing is deleted. Silent in the app.

### `POST /api/v1/auth/sign-out`: changed (optional `push_token`) · NTM-R28, NTM-R48, NTM-R49, NTM-D22, NTM-D33, NTM-D34, NTM-D35, NTM-D45, NTM-E20, NTM-E42, NTM-E47, NTM-E48
- **Who:** public. It needs only the refresh token, so it works with an expired access token.
- **Headers:** clients send **no `Authorization` header** (NTM-R49, NTM-D35).
  - The Android app leaves it out through `publicPaths` (shipped). The web BFF already passes no `token`.
  - A presented invalid or expired bearer is refused with `401` before the endpoint (NTM-E47). The server is not changed.
- **Body:**
  - `refresh_token`: required.
  - `push_token`: optional. Blank or whitespace-only counts as none (NTM-E48, NTM-D45).
  - `push_token` has no length or character check, because the spec sets none for sign-out. A value over 2048 characters, or with other characters, can't match a stored row (the V5 checks), so nothing is deleted.
- **200 OK** `data`: `null`, always, as today.
- **When the push token is deleted:**
  - `KeycloakTokenClient.logout` returns `true` when Keycloak answers 2xx. This includes a replayed logout of an already ended session (NTM-R48, NTM-D34, NTM-E42).
  - Only then does the server delete `push_tokens` where `token = push_token` and `user_id` = the refresh token's `sub`.
  - A token that isn't the user's, or doesn't exist, is ignored.
  - A refresh token Keycloak rejects (garbage, tampered, expired) deletes nothing. That push token stays until a later registration moves it or the 60-day job removes it.
- **Errors:**
  - `400 VALIDATION_FAILED` (`refresh_token` blank), as today.
  - `401` for a presented bad bearer (NTM-E47).

### `GET /api/v1/auth/me` and `GET /api/v1/branches`: exist, unchanged · NTM-R04, NTM-E07, NTM-E46, NTM-E52
- The owner's branch chip uses the active branches of the existing `workspace` (`GetWorkspaceUseCase`, which reads `/auth/me` and `/branches` and already filters on `is_active`).
- **`/branches` fails:** the chip offers only "All branches". The list is fetched again with `forceRefresh = true` (§7, NTM-E46).
- **`/auth/me` fails:** there is no chip and no error, and no `branch_id` is sent. Pull-to-refresh fetches the workspace again with `forceRefresh = true` (NTM-E52).

### FCM data message (identity-service → app): new · NTM-R30, NTM-R36, NTM-R51, NTM-R54, NTM-D07, NTM-D24, NTM-D42, NTM-E50
- **Delivery:** a data-only message (no `notification` block), Android priority `HIGH`, no `collapse_key`. One message goes to each of the recipient's tokens, for each new notification.
- **`data` keys** (all strings), **exactly**:
  - `notification_id`.
  - `alert_type`.
  - `branch_name`, left out for a business-wide notification (`branch_id` null).
- **Never sent:** `title`, `body`, `message`, amounts or other names (NTM-R51). This was shipped in T37.
- **What the app shows:**
  - **Title:** its own string for `alert_type` (NTM-M10 … NTM-M20, NTM-M22), or NTM-M21 for an unknown type (NTM-R39, NTM-E10).
  - **Text:** `branch_name`, or its own "All branches" string (NTM-M25).
- **When the app shows it:**
  - Only when someone is signed in on the phone (NTM-R54). A session that wasn't kept is gone once Android ends the process (NTM-E50).
  - Only in the background (NTM-R36).
  - Only while the phone allows the app's notifications (NTM-E16).

## 4. Screens, routes and states (changed)

| Route | Who | States (loading / empty / error / each status) | IDs |
|---|---|---|---|
| `Dashboard` (bell in `DashboardHeader`) | Owner, Manager | **Badge:** hidden when the count is unknown or 0; otherwise the number, capped at "99+".<br>**Count reload:** on `ScreenResumed` (covers opening and returning to the foreground), on a foreground push, and after every center action. A failed reload keeps the last count.<br>**Permission prompt:** once, on Android 13+, on the first Dashboard after the first sign-in on this phone.<br>**Bell tap:** opens `Notifications()` | NTM-R01, NTM-R02, NTM-R17, NTM-R26, NTM-E16, NTM-E17 |
| `Notifications(openedNotificationId: String? = null)` (changed) | Owner, Manager | **Top bar:** Back, title, and "Mark all read" (double-check icon). It is disabled when no loaded card of the current tab is unread. While a mark-all is sending, it is disabled on **every** tab and branch until that one answers (NTM-E14, NTM-E51).<br>**Filters:**<br>- Tabs All / Unread {n} / Financial / Stock (NTM-M23), in a row that scrolls sideways when the labels don't fit; no label is cut (NTM-R55).<br>- The owner's branch chip: "All branches" plus active branches. When the branch list failed, it offers only "All branches", shows no error, and the list is fetched again on pull-to-refresh or reopening (NTM-E46).<br>- When the workspace (`/auth/me`) failed, there is no chip and no error, as for a manager; pull-to-refresh retries it (NTM-E52).<br>**List:** day groups, with "Today" / "Yesterday" taken from the last load, not live at midnight (NTM-E45). The bottom bar shows with Home selected, as in Figma 18.21.<br>**Loading:** skeleton.<br>**Errors:** a first-page error shows NTM-M03 or NTM-M28 with Retry. A next-page error shows a footer with NTM-M03 and Retry.<br>**Empty:** NTM-M01, or NTM-M02 on the Unread tab.<br>**Refresh:** pull-to-refresh, plus a quiet reload on resume and on a foreground push.<br>**Cards:** unread cards have a dot and a tint; read cards have an overflow menu with "Mark as unread". Times are in Asia/Phnom_Penh (NTM-R06).<br>**Mark all while switching:** applies to the tab and branch of the tap. NTM-M04 with Undo shows; Undo restores the tapped tab's snapshot. The new tab's cards aren't marked on screen, and it loads as usual (NTM-E44).<br>**Snackbars:** NTM-M03, NTM-M04 with Undo, NTM-M05, NTM-M09.<br>**Opened from a push:** the read call first (failures silent apart from the `404` rule), then the All tab | NTM-R03 … NTM-R18, NTM-R35, NTM-R36, NTM-R38, NTM-R39, NTM-R55, NTM-E39, NTM-E40, NTM-E44, NTM-E45, NTM-E46, NTM-E51, NTM-E52 |
| `Profile` (Menu → Settings) | Owner, Manager | New "Notifications" row in the Preferences card, before "Telegram alerts"; it shows no value | NTM-R19 |
| `NotificationSettings` | Owner, Manager | **Layout:** no Figma (18.22 shows only the Settings list). Built from `SubpageScaffold`, `SettingsCard` and `SettingsSwitchRow`.<br>**Loading:** skeleton.<br>**Error:** NTM-M03 or NTM-M28 with Retry.<br>**Banner:** NTM-M07 with "Open settings" when the phone blocks the app's notifications; checked again on resume.<br>**Rows:** one per server type, titled NTM-M10 … NTM-M22 (NTM-M21 if unknown).<br>**Switches:** In-app is on and disabled when `locked`. Push is disabled while In-app is off, showing its stored value. Telegram shows "On" / "Off", read-only.<br>**Telegram note:** NTM-M29 when `telegram_linked` is false; tapping it opens `TelegramAlerts`.<br>**On resume:** the matrix reloads quietly, so a Telegram link made meanwhile shows.<br>**Saving:** a switch saves on change. On failure it reverts and a snackbar shows NTM-M30, NTM-M03 or NTM-M28 | NTM-R19, NTM-R20, NTM-R22, NTM-R23, NTM-R38, NTM-R39, NTM-R44, NTM-R45, NTM-R46, NTM-E41 |
| `TelegramAlerts` (exists, unchanged) | Owner, Manager | Opened from the NTM-M29 note; Back returns to `NotificationSettings` | NTM-R46 |
| `SignIn` after a rejected refresh (exists; changed behind it) (changed) | Owner, Manager | The existing `sessionExpired` navigation is unchanged. Behind it, with no UI and no sign-out call, the phone's FCM token is deleted (5 s timeout) and the kept `registered_token` is cleared (NTM-R50, NTM-E43). If the deletion fails or times out, the kept token is cleared anyway and nothing retries. The token stays alive until the next sign-in moves it, and pushes meanwhile are dropped (NTM-E54) | NTM-R50, NTM-E43, NTM-E54 |
| `SignIn` at app start, session not kept (exists, unchanged) (changed) | Owner, Manager | With "Keep me signed in" off and the process ended by Android, the app starts signed out. Pushes received meanwhile post nothing, and the token stays registered (NTM-E50) | NTM-E50 |
| System notification (channel "Alerts", NTM-M08) (changed) | Owner, Manager | **Posted only when:** someone is signed in (NTM-R54, NTM-E50, NTM-E54), the app is in the background, and notifications are enabled. A push arriving a moment after Home may count as foreground and post nothing (NTM-E49).<br>**Content:** title per type, text per branch (NTM-R51); auto-cancel; one per notification.<br>**Tap:** opens the center through `MainActivity`. When signed out (a notification already in the tray), it goes `SignIn` → `Dashboard` → center (NTM-E18) | NTM-R35, NTM-R36, NTM-R37, NTM-R51, NTM-R54, NTM-E18, NTM-E49, NTM-E50, NTM-E54 |

## 5. Data (changed)

| Table.field (dictionary name) | Read / write | Repo | IDs |
|---|---|---|---|
| `notifications.*` (NTF, V1) | Read: list, count, read and unread through the existing service. Read by `PushDelivery` after commit | identity-service; android reads through §3 | NTM-R03 … NTM-R18, NTM-R30, NTM-R40 |
| `users.notification_preferences` (`alerts.<TYPE>.push`; `version` stays 1) | Read by `AlertPreferences` and the intake. Written by `PUT /preferences`; `null` or absent `push` keeps the stored value | identity-service | NTM-R20, NTM-R22, NTM-R24, NTM-R30, NTM-R42, NTM-R52 |
| `push_tokens` | **Schema:**<br>- `V3__create_push_tokens.sql`: `push_token_id` uuid PK; `user_id` FK `users`, not null; `token` text, not null, unique; `platform` enum `push_platform` (`ANDROID`); `created_at`; `last_seen_at`; indexes on `user_id` and `last_seen_at`.<br>- `V4__push_tokens_token_length.sql` (applied on the shared dev DB, never edited): `ck_push_tokens_token_length CHECK (char_length(token) <= 4096)`.<br>- **v5.0.0** `V5__push_tokens_token_format.sql` (or the next free `V<n>` after V4) deletes rows over 2048 or with other characters, replaces `ck_push_tokens_token_length` with `<= 2048`, and adds `ck_push_tokens_token_chars`. With ASCII-only tokens, characters equal bytes, so the longest token stays below the unique index's entry limit (NTM-D49, NTM-D50, NTM-D54).<br>**Writes:** upsert on register. Delete on unregister, sign-out (also on a replayed logout), invalid token, deactivation and the other paths in §6, by the 60-day job, and once by V5 | identity-service | NTM-R25, NTM-R28, NTM-R29, NTM-R32, NTM-R33, NTM-R41, NTM-R48, NTM-R56, NTM-R57, NTM-E53 |
| `users.is_active`, `users.deleted_at`, `users.business_id`, `businesses.deleted_at` | Read: the token-deletion hooks run inside the existing transactions | identity-service | NTM-R33 |
| `branches.branch_name`, `branches.is_active` | Read: card label, "(inactive)", push `branch_name`, owner chip | identity-service, android | NTM-R04, NTM-R07, NTM-R51, NTM-E07, NTM-E46 |
| Phone storage (not a dictionary table): SharedPreferences `push` with `registered_token` and `permission_asked` | Read / write. Both survive sign-out, except `registered_token`. Sign-out and a forced sign-out clear it, even when the FCM deletion failed (NTM-E54). It is not cleared when the process ends with a session that wasn't kept (NTM-E50) | android-ui (`:core`) | NTM-R25, NTM-R26, NTM-R27, NTM-R28, NTM-R50, NTM-E50, NTM-E54 |

## 6. Roles and branch scope (changed)

- **identity-service:**
  - **Endpoints:** every endpoint takes `@AuthenticationPrincipal CurrentUser`. The push-token endpoints read and write only rows with `user_id = currentUser.userId()`, except the register upsert, which moves a token to the caller (NTM-D21). The token rules (NTM-R56, NTM-R57) are the same for both roles.
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
  - The V5 deletion touches every user's rows, but only those no real FCM token can match (NTM-D54).
- **android-ui:**
  - **Navigation:** both roles reach the bell, the center and the settings. No route is role-gated.
  - **Branch chip and filter:**
    - Shown only once the workspace has loaded and `Workspace.managerBranchId == null` (owner).
    - While the workspace hasn't loaded, the app treats the caller as having no chip and sends no `branch_id` (NTM-E52).
    - The ViewModel sends `branch_id` only for an owner with a branch chosen (NTM-E09).
    - The workspace is fetched again with `forceRefresh = true` for an owner whose branch list is empty (NTM-E46), or when it never loaded (NTM-E52).
  - **Settings:** list exactly the types the server returns.
  - **Push:**
    - The app shows what it receives, and only while a user is signed in (NTM-R54, NTM-E50).
    - Opening a push for another user's notification gets `404`, which is ignored (NTM-E37).
    - Sign-out and forced sign-out both delete the phone's FCM token, so a phone stops getting the previous user's pushes once FCM reports the token invalid (NTM-R28, NTM-R50).
    - When that deletion fails, the next sign-in moves the token (NTM-E54).

## 7. Files (changed)

Rows marked **v5** are the new work. The other rows are shipped (parts A–E) or committed (v4, T34–T42), and are listed for the whole picture.

| Repo | Create / change | Why (IDs) |
|---|---|---|
| identity | create `src/main/resources/db/migration/V3__create_push_tokens.sql` (shipped) | `push_tokens` table and enum, unique `token`, indexes on `user_id` and `last_seen_at` (NTM-R25, NTM-R29, NTM-R41) |
| identity | `src/main/resources/db/migration/V4__push_tokens_token_length.sql` (committed in 8fdd4b3, **applied on the shared dev DB**) | **Never edited**, its comment included. V5 replaces its constraint |
| identity | **v5** create `src/main/resources/db/migration/V5__push_tokens_token_format.sql`. After `git fetch`, use the next free `V<n>` after V4 if `origin/dev` has taken V5 | A comment citing `specs/notification-mobile` §8 (no requirement IDs), then, in this order (Flyway runs it in one transaction):<br>`DELETE FROM push_tokens WHERE char_length(token) > 2048 OR NOT ((token COLLATE "C") ~ '^[A-Za-z0-9_:.-]*$');`<br>`ALTER TABLE push_tokens DROP CONSTRAINT ck_push_tokens_token_length;`<br>`ALTER TABLE push_tokens ADD CONSTRAINT ck_push_tokens_token_length CHECK (char_length(token) <= 2048);`<br>`ALTER TABLE push_tokens ADD CONSTRAINT ck_push_tokens_token_chars CHECK ((token COLLATE "C") ~ '^[A-Za-z0-9_:.-]*$');`<br>Notes:<br>- `COLLATE "C"` keeps the ranges to plain ASCII whatever the database collation.<br>- `*` rather than `+`, so the check adds no "not empty" rule: the DTOs own that, as in V3.<br>- No `identity.` prefix: the schema comes from Flyway, as in V3 and V4 (NTM-R56, NTM-R57, NTM-D49, NTM-D50, NTM-D54, NTM-E53) |
| identity | create `notification/entity/PushToken.java`, `PushPlatform.java` (shipped) | Mapped like `Notification.feed` (`PostgreSQLEnumJdbcType`) (NTM-R25) |
| identity | create `notification/repository/PushTokenRepository.java` (shipped) | **Upsert:** native `ON CONFLICT (token) DO UPDATE SET user_id, platform, last_seen_at`.<br>**Deletes:** by `(token, user_id)`, by user, by business (join `users`), and by `last_seen_at < cutoff`.<br>**Lookup:** a user's tokens (NTM-R25, NTM-R28, NTM-R29, NTM-R32, NTM-R33, NTM-R41, NTM-R48) |
| identity | create `notification/service/PushTokenService.java`, `service/impl/PushTokenServiceImpl.java` (shipped) | `register` and `unregister`; `deleteForUser(userId, token)`; `deleteAllOf(userId)` and `deleteAllInBusiness(businessId)` (`Propagation.MANDATORY`); `deleteNotSeenSince(cutoff)` (NTM-R25, NTM-R33, NTM-R41) |
| identity | create `notification/controller/PushTokenController.java` (shipped); **v5** change `notification/dto/RegisterPushTokenRequest.java`, `notification/dto/UnregisterPushTokenRequest.java` | **v5:** `token` keeps `@NotBlank(message = "{validation.required}")`, and gets:<br>- `@Size(max = 2048, message = "{validation.max_length}")`, replacing 4096;<br>- `@Pattern(regexp = "[A-Za-z0-9_:.-]*", message = "{validation.invalid_value}")`;<br>- `@Schema(maxLength = 2048, pattern = "[A-Za-z0-9_:.-]*")`.<br>Use the same literals in both DTOs. `*` lets an empty token get only "This field is required.".<br>The existing `GlobalExceptionHandler` answers `400 VALIDATION_FAILED` with `data.token` = "Use at most 2048 characters." (NTM-M32) or "Enter a valid value." (NTM-M33), and keeps the first error it reads for the field (NTM-E56).<br>**Whitespace-only `token`:** it fails both `@NotBlank` and `@Pattern`. What it answers depends on §10 Q2. Under (b), `@NotBlank` and `platform`'s `@NotNull` stay in the default group, and `@Size` / `@Pattern` move to a second group that runs only when the first passes (`@GroupSequence` on the record).<br>`messages.properties` needs no change (NTM-R56, NTM-R57, NTM-E56) |
| identity | change `notification/service/AlertPreferences.java`, `notification/dto/AlertChannelsRequest.java`, `AlertChannelsResponse.java`, `service/impl/NotificationPreferencesServiceImpl.java` (shipped) | Optional strict-boolean `push`, kept when absent or `null`, returned by GET and PUT. No cross-field rule (NTM-R20, NTM-R22, NTM-R42, NTM-R52, NTM-D30) |
| identity | change `notification/service/impl/AlertIntakeServiceImpl.java` (shipped) | Publish `PushDeliveryRequested` for newly inserted rows whose `channels.push()` is on (NTM-R24, NTM-R30, NTM-R40) |
| identity | `notification/push/PushDeliveryRequested.java`, `PushDelivery.java`, `PushMessage.java`, `FcmPushSender.java`, `PushSender.java`, `PushSendResult.java`, `PushProperties.java`, `PushConfig.java` (shipped; data keys in T37) | After commit on a private two-thread daemon scheduler. Re-reads the row, the user's tokens and the branch name; sends data-only with `notification_id`, `alert_type` and `branch_name` (when not null); `HIGH`; no collapse key; 10 s timeouts; deletes `INVALID_TOKEN` rows; disabled without a key; logs ids and counts only. No v5 change (NTM-R30, NTM-R32, NTM-R34, NTM-R40, NTM-R51, NTM-E25) |
| identity | create `notification/job/PushTokenCleanupJob.java` (shipped) | Daily 02:45 Asia/Phnom_Penh, ShedLock `push-token-cleanup`, deletes rows not seen for 60 days (NTM-R41) |
| identity | `auth/controller/AuthController.java`, `auth/dto/SignOutRequest.java`, `auth/service/AuthService.java`, `auth/service/impl/AuthServiceImpl.java`, `common/keycloak/KeycloakTokenClient.java` (shipped; Javadoc in T34) | Sign-out deletes the push token after an accepted logout, a replayed one included; blank counts as none; always `200`. No v5 change: `push_token` gets no length or character check (NTM-R28, NTM-R48, NTM-E42, NTM-E48) |
| identity | change `manager/service/impl/ManagerServiceImpl.java`, `manager/service/ManagerRemoval.java`, `user/service/impl/UserServiceImpl.java`, `business/service/impl/BusinessDeletionServiceImpl.java` (shipped) | Token deletion in the same transaction as each change (NTM-R33) |
| identity | change `build.gradle`, `.env.example`, `consul/identity-service.yml`, `k8s/40-identity-service.yaml`, `k8s/DEPLOY.md`, `README.md`, `src/test/resources/application-it.yml` (shipped) | Admin SDK; `FCM_SERVICE_ACCOUNT_BASE64` (empty = push off); `ledgervision.push.fcm-service-account`; job cron; optional secret `camfinsight-fcm` (NTM-D16, NTM-R41). No v5 change |
| identity | v4 tests (committed, T34–T37) | `KeycloakTokenClientTest`, `AuthControllerIT` (NTM-R48, NTM-E42, NTM-E47, NTM-E48), `NotificationPreferencesIT` (NTM-R52), `FcmPushSenderTest`, `PushDeliveryTest`, `PushDeliveryIT` (NTM-R51), `PushConfigTest` (NTM-D16, NTM-R34). `PushTextTest` was deleted |
| identity | **v5** tests, each name starting with the ID | **`PushTokenControllerIT`:**<br>- Replace the two NTM-R53 tests with:<br>  - "NTM-R56 NTM-M32 a 2048-character token is saved; 2049 is 400 on token with \"Use at most 2048 characters.\" and nothing is stored";<br>  - "NTM-R56 NTM-M32 unregister with a 2049-character token is 400 on token and deletes nothing; 2048 is 200".<br>- Add "NTM-R57 NTM-M33 a token of every allowed character (`A–Z a–z 0–9 _ : . -`) is saved; one with a space, `é`, `/`, `+`, `=` or `;` is 400 on token with \"Enter a valid value.\" and nothing is stored".<br>- Add the same for unregister, where nothing is deleted.<br>- Add "NTM-E56 a 2049-character token with a space is 400 on token with \"Use at most 2048 characters.\" or \"Enter a valid value.\", and nothing is stored".<br>- The two NTM-R25 blank-token tests keep `""`, `null` and missing as "This field is required.". Their whitespace-only bodies (`"   "`, `"  "`) assert what §10 Q2 decides.<br>**`PushTokenRepositoryIT`:**<br>- Replace "NTM-R53 the database refuses a token over 4096 characters" with "NTM-R56 the database stores a random 2048-character token of the allowed characters and refuses 2049 (`ck_push_tokens_token_length`)". Draw the characters at random from the 66 allowed, so the value doesn't compress and the unique index takes its real size.<br>- Add "NTM-R57 the database refuses a token with a space or a non-ASCII character (`ck_push_tokens_token_chars`)".<br>**New `support/PushTokenCapMigrationIT`** (like `OneManagerMigrationIT`: a throwaway schema `v5_check`, Flyway `target("4")`, then `target("5")`):<br>"NTM-E53 NTM-D54 V5 deletes stored tokens over 2048 characters or with other characters, keeps the rest, and adds both checks". V4 rows:<br>- a 160-character valid token;<br>- a 2048-character valid token;<br>- 2049 × `a`;<br>- `has space`;<br>- `café`.<br>After V5:<br>- only the first two remain;<br>- `pg_constraint` has `ck_push_tokens_token_length` (2048) and `ck_push_tokens_token_chars`;<br>- the V5 history row is successful.<br>**Check:** `grep -rn -e NTM-R53 -e NTM-M31 src/test src/main/java` prints nothing. Only the V4 SQL comment keeps NTM-R53 |
| android | change `app/build.gradle.kts`, `e2e/android/*`, `gradle/libs.versions.toml`, root `build.gradle.kts`, `core/build.gradle.kts`, `.gitignore`, `local.properties.example`, `README.md` (shipped) | Debug app id `.dev`, Firebase wiring only when `google-services.json` exists, `PUSH_FAKE_TOKEN` (NTM-R47, NTM-D16, NTM-D29) |
| android | change `core/.../navigation/AppDestinations.kt`, `core/.../data/entity/DataState.kt`, `core/.../network/RemoteDataSource.kt` (shipped) | Destinations; field errors (NTM-R03, NTM-R19, NTM-R35, NTM-R45) |
| android | change `core/.../network/AuthInterceptors.kt` (shipped in d2b8509) | `/auth/sign-out` in `publicPaths`: no bearer, no refresh (NTM-R49, NTM-E47) |
| android | `core/.../push/PushTokenSource.kt`, `PushTokenStore.kt`, `PushSignals.kt` (shipped; `PUSH_TOKEN_DELETE_TIMEOUT_MILLIS` moved here in T38) | Push interfaces. `deleteToken` never throws (NTM-R28, NTM-R50, NTM-E54) |
| android | `app/.../push/FirebasePushTokenSource.kt`, `AppMessagingService.kt`, `PushModule.kt`, `ic_stat_notification.xml`; `AndroidManifest.xml`, `StarterApplication.kt`, `MainActivity.kt`, `ui/navigation/MainGraph.kt`; `app/src/debug/.../DebugPushReceiver.kt` (shipped) | Firebase only in `:app`; channel; foreground tracking; tap → center; simulator. The fake token `debug-<uuid>` already fits NTM-R57. No v5 change (NTM-R25, NTM-R35, NTM-R36, NTM-R37, NTM-E18, NTM-E24) |
| android | `features/.../notification/data/{api,dto,mapper,service,repository}/*` (shipped) | Retrofit, DTOs, mappers, repositories (NTM-R02 … NTM-R26) |
| android | `features/.../notification/data/push/PushTokenSync.kt` (shipped; forced sign-out in T38) | Registration under one Mutex. `forgetPhoneToken()` on `sessionStore.expired`: delete with a 5 s timeout, then `keepToken(null)`, with no network call and no retry. No v5 change (NTM-R25, NTM-R27, NTM-R29, NTM-R50, NTM-E43, NTM-E50, NTM-E54) |
| android | `features/.../notification/data/push/PushMessageHandler.kt`, `SystemNotifier.kt` (shipped; signed-out drop in T39) | Returns at once while `currentUser` is null. Foreground → signal and count; background → post when enabled. No v5 change (NTM-R36, NTM-R37, NTM-R51, NTM-R54, NTM-E10, NTM-E16, NTM-E50) |
| android | `features/.../notification/domain/**` (shipped) | Models, `NotificationTime` (Asia/Phnom_Penh), use cases (NTM-R05, NTM-R06, NTM-E31) |
| android | `features/.../notification/presentation/model/*`, `components/*`, `screen/*` (shipped) | Center and settings UI. `NotificationTabs` scrolls sideways (NTM-R55). `isMarkAllEnabled` is false while `isMarkingAll` (NTM-E14, NTM-E51) |
| android | `features/.../notification/presentation/viewmodel/NotificationsViewModel.kt` (shipped; T40, T41) | `markAll` takes the query and snapshot at the tap (NTM-E44). `loadWorkspace(forceRefresh)` runs again for an empty owner branch list or a workspace that never loaded (NTM-E46, NTM-E52). No v5 change |
| android | `features/.../profile/data/local/DeviceSignOut.kt`, `features/.../profile/domain/service/ProfileService.kt` (shipped; Javadoc in T38) | Sign-out sends `push_token`, always deletes the FCM token, and clears the kept token (NTM-R28, NTM-R49, NTM-E20, NTM-E42) |
| android | `features/src/main/res/values/strings.xml` (shipped) | Every live message: NTM-M01 … NTM-M05, NTM-M07 … NTM-M29. NTM-M28, NTM-M30, NTM-M32 and NTM-M33 are server text and never shown by the app (NTM-R39) |
| android | `features/.../di/*`, `dashboard/**`, `profile/**`, `LoadMoreOnScroll` (shipped) | Bindings, bell, settings row, paging |
| android | **v5** change `features/src/test/.../notification/fake/FakePushTokenSource.kt` | Add `var deleteHangs = false`. When true, `deleteToken` counts the call, then calls `awaitCancellation()`, so the 5 s timeout runs out in virtual time (NTM-E54) |
| android | **v5** tests, each name starting with the ID (no production change) | **`PushTokenSyncTest`:**<br>- Add "NTM-E50 NTM-D51 an app start with no restored session sends nothing, deletes nothing and keeps the kept token, so the server row stays registered". Set `repository.kept = "t-1"`, with no sign-in. After `start`: no calls, `kept == "t-1"`, `deleteCalls == 0`.<br>- Add "NTM-E54 NTM-D55 a forced sign-out whose token deletion times out still forgets the kept token, makes no server call and doesn't retry; the next sign-in registers the same token again". Sign in A, `deleteHangs = true`, `expire()`, then advance past 5 s: `kept == null`, no calls. Sign in B: calls are `["register:t-1"]`, and `deleteCalls == 1`.<br>**`NotificationsViewModelTest`:**<br>- Add "NTM-E51 NTM-D52 while Mark all read is sending, the button is disabled on the tab switched to as well, and enabled there again once it answers". Gate `readAll`, tap on Stock, switch to All, which has an unread card: `isMarkAllEnabled` is false. Open the gate: it is true.<br>- Add "NTM-E52 NTM-D53 a failed workspace shows no chip and no error, sends no branch_id, and pull-to-refresh fetches it again". The first `getWorkspace` is offline: `isOwner` false, no effects, list `Success`, every page call has `branchId == null`. On `Refresh` (`forceRefresh = true`) it succeeds: `isOwner` true, and the branches show |
| camfinsight-docs | **v5** change `api/identity-service.md` (lead) | **Table:**<br>- `push-tokens` row: 📝, "(v5.0.0: `token` at most 2048, FCM characters only)", spec IDs NTM-R25, NTM-R29, NTM-R41, NTM-R56, NTM-R57.<br>- `unregister` row: 📝, the same note, NTM-R25, NTM-R56, NTM-R57.<br>- The other rows are unchanged (🚧, then ✅ when F merges).<br>**push-tokens section:**<br>- Body: `token` "(required; at most 2048 characters of `A–Z a–z 0–9 _ : . -`)".<br>- Replace the 4096 error row with: `token` longer than 2048 characters → `data.token` "Use at most 2048 characters." (NTM-R56); nothing stored.<br>- Add a row: `token` with any other character → `data.token` "Enter a valid value." (NTM-R57); nothing stored.<br>- Add one sentence: a token both too long and with other characters gets either message (NTM-E56).<br>- The blank row follows §10 Q2.<br>**unregister section:** the same rows, with "nothing deleted".<br>**Statuses:** 🚧 when V5 and the DTOs are on the branch, ✅ when F merges into `dev` (NTM-R56, NTM-R57, NTM-E56) |
| camfinsight-docs | **v5** change `data-dictionary/notifications.md` (lead) | `push_tokens.token`: Length `2048`. Description: "the FCM registration token; unique (one row per phone); at most 2048 characters, only `A–Z a–z 0–9 _ : . -` (`ck_push_tokens_token_length`, `ck_push_tokens_token_chars`, identity V5)". Spec §8 state moves from proposed to done when F merges (NTM-R56, NTM-R57) |

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
| NTM-E50 | android `PreferencesSessionStore` (no user restored when `remember` is false), `PushMessageHandler` (drops while `currentUser` is null) and `PushTokenSync` (sends and deletes nothing while signed out, so the server row stays). Existing code: `PushTokenSyncTest` NTM-E50, QA NTM-E50-M1 |
| NTM-E51 | android `NotificationsUiContract.isMarkAllEnabled` (`!isMarkingAll && …`, a screen-wide flag). Existing code: `NotificationsViewModelTest` NTM-E51, QA NTM-E51-C1 |
| NTM-E52 | android `NotificationsViewModel.loadWorkspace` / `refreshWorkspaceIfIncomplete`: a failed `/auth/me` leaves `isOwner` false and `workspaceLoaded` false with no effect; `Refresh` fetches it again with `forceRefresh`. Existing code: `NotificationsViewModelTest` NTM-E52, QA NTM-E52-M1 |
| NTM-E53 | identity `V5__push_tokens_token_format.sql`: the `DELETE` runs before both checks, in one transaction, so identity-service starts (`PushTokenCapMigrationIT`, QA NTM-E53-C1) |
| NTM-E54 | android `PushTokenSync.forgetPhoneToken`: a failed or timed-out deletion still clears `registered_token`, makes no call and doesn't retry. `PushMessageHandler` drops pushes while signed out. The next sign-in registers the same token, which identity moves (NTM-D21). Existing code: `PushTokenSyncTest` NTM-E54, QA NTM-E54-M1; real FCM in T33's list |
| NTM-E56 | identity DTO `@Size` + `@Pattern` and `GlobalExceptionHandler` (first error per field wins): either NTM-M32 or NTM-M33 on `token` (`PushTokenControllerIT` NTM-E56, QA NTM-E56-V1) |

## 9. Packages and verification (changed)

**Packages:** none new in v5.0.0 (nor in v4.0.0). Those added in parts A–E (Firebase BoM 34.x and `firebase-messaging` in android `:app`, the google-services plugin 4.4.x, and `firebase-admin` 9.x in identity) are unchanged.

**Firebase setup (NTM-D16, NTM-D29):** unchanged.
- **App ids:** debug `com.kshrd.starter.dev`, release `com.kshrd.starter`.
- **The handed-out file:** `google-services.json`, never committed.
- **Without the file:** push is off and no token is registered, unless `push.fakeToken=true` in debug.
- **identity:** `FCM_SERVICE_ACCOUNT_BASE64` in `.env`; empty = a disabled sender.

**Verification per repo (v5.0.0):**
- **identity-service** (worktree `ledgervision-identity-service-ntm`, on top of T37):
  - `git fetch` first, and check that no `V5__*` exists on `origin/dev`. If one does, use the next free number.
  - `./gradlew build` (unit tests plus Testcontainers ITs, Docker running). V5 applies on a fresh DB with `ddl-auto: validate`. `PushTokenCapMigrationIT` checks it on a V4 database holding bad rows.
  - `./gradlew bootRun` against a **local** Postgres (`infra/docker-compose.yml --profile db`), **not the shared dev DB**, while the V5 file may still change. Once applied there it can't be edited, as happened with V4. The shared dev DB gets V5 only from the deployed build, after approval and merge.
  - Then, with no key and the shared infra (real Keycloak):
    - **Token cap:** register with 2048 → `200`; 2049 → `400`, `data.token` "Use at most 2048 characters."; a random 4096-character token → `400`, never `500` (NTM-R56).
    - **Token characters:** `abc def`, `café` and `a/b` → `400`, `data.token` "Enter a valid value."; `aZ0_:.-` → `200` (NTM-R57).
    - **Unregister:** the same, and nothing is deleted.
    - **Both:** 2049 characters with a space → `400` with either message (NTM-E56).
    - **Migration:** `\d identity.push_tokens` lists `ck_push_tokens_token_length` (2048) and `ck_push_tokens_token_chars` (NTM-E53).
  - `grep -rn -e NTM-R53 -e NTM-M31 src/test src/main/java` prints nothing.
  - The `api/identity-service.md` entries match the code (dictionary-checker clean against V5).
- **android-ui:** `./gradlew test checkCode checkKtlint` without `google-services.json`. The v5 work is tests only, so no new e2e run is needed beyond T44's.
- **web-ui:** none (checked: no bearer on sign-out, no push-token calls).

**v4.0.0 QA (after F and G are on `dev`):**
- **api-qa** (identity, real Keycloak, fake sender):
  - NTM-R48-M1, NTM-R48-M2 (replayed logout deletes).
  - NTM-E42-M1 (garbage), NTM-E42-M2 (replay).
  - NTM-E47-M1 (bad bearer → 401).
  - NTM-E48-M1 (blank and spaces).
  - NTM-R52-C1 (`null` keeps the value; nothing stored stays unwritten).
  - ~~NTM-R53-V1~~: withdrawn, replaced by the v5.0.0 scenarios below.
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

**v5.0.0 QA (new; after F and G are on `dev` and identity is deployed with V5):**
- **api-qa** (identity, real Keycloak, fake sender):
  - **NTM-R56-V1:**
    - Register: 2048 → `200`; 2049 → `400` with NTM-M32 on `token`, nothing stored.
    - A random 4096-character token → `400`, not `500` (the Q31 finding).
    - Unregister: 2049 → `400` with NTM-M32, nothing deleted.
  - **NTM-R57-V1:**
    - Register and unregister with a space, `é`, an emoji, `/`, `+`, `=`, `;` → `400` with NTM-M33 on `token`, nothing stored or deleted.
    - A token using every allowed character → `200`.
  - **NTM-E56-V1:** 2049 characters with a space → `400` on `token`, with NTM-M32 or NTM-M33; nothing stored.
  - **NTM-E53-C1:**
    - On a scratch Postgres migrated to V4, insert a 2049-character token and `has space` for a test user, then start identity-service. It starts, both rows are gone, and the valid rows stay.
    - On the shared dev DB after deployment, `select count(*) from identity.push_tokens where char_length(token) > 2048 or token !~ '^[A-Za-z0-9_:.-]*$'` is `0`, and both checks exist.
  - **Whitespace-only token** on register and unregister → `400` on `token`, with the message §10 Q2 decides.
- **android-qa** (`.dev` build, `push.fakeToken=true`, T27 simulator):
  - **NTM-E50-M1:**
    - Sign in with "Keep me signed in" off, so the token is registered (`select user_id from identity.push_tokens`). Press Home, then `adb shell am kill com.kshrd.starter.dev`.
    - Simulate a push: no system notification. The server row is still there for that user.
    - Open the app: sign-in. After signing in, a simulated push in the background posts again.
  - **NTM-E51-C1:** owner, slow network (proxy delay on `read-all`). On Stock, tap "Mark all read" and switch to All at once: the button is disabled on All until the answer, then enabled if All still has an unread card.
  - **NTM-E52-M1:** owner, cold start with `GET /auth/me` blocked (proxy) after the session is restored. Open the center: there is no branch chip and no error, and the list loads. Unblock and pull to refresh: the chip appears.
  - **NTM-E54-M1:**
    - On the fake-token build, the FCM deletion does nothing, as when it fails. Force a rejected refresh as in NTM-R50-M1 for user A: sign-in opens, `registered_token` is gone, and the server row still belongs to A.
    - Simulate a push: no system notification.
    - Sign in as user B on the same phone: the same token's row now belongs to B (NTM-D21).
- **Still blocked until both NTM-D16 files are handed out (real FCM):**
  - NTM-R30 and NTM-R51 end to end;
  - NTM-R36 in the background and in Doze;
  - NTM-R37 on a real push;
  - NTM-R32 by a real FCM answer;
  - NTM-R50 (real FCM token deleted, then the server drops it on the next send);
  - NTM-E54 with a real deletion that times out (airplane mode right after the refresh is rejected);
  - NTM-E20, NTM-E21, NTM-E22, NTM-E23.
- **Test names start with the ID:**
  - identity: `@DisplayName("NTM-R56 …")`.
  - android: `` fun `NTM-E54 …`() ``.
  - Every live R has a test or one of the QA scenarios above. NTM-R39 is checked by review and on the device.

## 10. Questions and conflicts (changed)

- **Q1 · NTM-R54 and a session not kept.** Answered 2026-11-16 by Vuththana: (a), NTM-R54 as written. The spec has since stated it (NTM-D51, NTM-E50, v5.0.0).
- **Q2 · A whitespace-only push token (new, v5.0.0).**
  - **The problem:** `"   "` is blank, and it is also a token with spaces.
    - The shipped contract answers blank with `data.token` "This field is required." (plan §3, `api/` "missing or blank", and the NTM-R25 tests send `"   "` and `"  "`).
    - NTM-E56 says a token with a space gets NTM-M33, and its "either message" covers only the too-long case.
    - With `@NotBlank` and `@Pattern` both on `token`, such a value fails both. Today's handler then keeps whichever error the validator reports first, and that order isn't fixed, so the NTM-R25 tests would become flaky.
  - **Options:**
    - (a) Either message, as NTM-E56 does for the too-long case. No extra code. The NTM-R25 blank tests accept either message for whitespace-only bodies, and `api/` says so.
    - (b) Blank wins: "This field is required.". This matches the shipped contract, the existing tests and `api/`, and NTM-D45, which already treats a whitespace-only `push_token` as blank on sign-out. The DTOs get a `@GroupSequence`, so length and characters are checked only after `@NotBlank` and `platform` pass.
    - (c) NTM-M33 wins: "Enter a valid value.". This is the literal reading of NTM-E56. It needs a group sequence the other way round, and it changes the shipped blank contract for whitespace.
  - **Suggested:** (b), which keeps today's behaviour and agrees with NTM-D45. Any option needs one line in the spec through `/sdd:amend` (NTM-E56, or a new E).
  - **Blocks:** only the DTO validation order in part F and the whitespace assertions of the two NTM-R25 blank-token tests, plus that line in `api/`. V5, the NTM-R56 / NTM-R57 / NTM-E53 / NTM-E56 tests, all of part G and the rest of part H can go ahead.
  - **Answered 2026-11-17 by Vuththana: (b), blank wins.** Spec v6.0.0: NTM-E55 withdrawn → NTM-E56, new NTM-D56. The DTOs check not-blank before length and characters (`@GroupSequence`); the NTM-R25 blank tests keep "This field is required.".

## Changes

| Date | Change | Why | Approved by |
|---|---|---|---|
| 2026-11-16 | §1, §2, §3, §4, §7, §8, §9, §10 changed for spec v3.0.0: NTM-D29 … NTM-D33, NTM-R43 (changed), NTM-R44 … NTM-R47, NTM-E38 … NTM-E41, NTM-M09 code, NTM-M29, NTM-M30; NTM-R21, NTM-D05, NTM-M06 withdrawn | Plan §10 Q1 … Q7 answered through /sdd:amend (v3.0.0, be5af46) | Vuththana |
| 2026-11-16 | §1–§10 changed for spec v4.0.0: NTM-R43 → R48, NTM-E38 → E42, NTM-E28 → E43, NTM-R31 → R51 withdrawn and replaced; NTM-R49, R50, R52, R53 (+ NTM-M31), R54, R55 and NTM-E44 … E49 added; NTM-R06, E31, E18, E37, R28, D07 reworded. Sign-out and FCM data contracts rewritten; new parts F (identity), G (android), H (docs); `push_tokens.token` ≤ 4096 with a CHECK migration; plan §10 Q1 on NTM-R54 | /sdd:amend v4.0.0 (565f87e), answering verify and QA gaps NTM-Q15 … NTM-Q30 | Vuththana |
| 2026-11-17 | §1–§10 changed for spec v5.0.0: NTM-R53 → R56 / R57 (+ NTM-M32, M33; NTM-D46 → D49, D50). Part F gains a `V5` migration (delete bad rows, cap 2048, character check; V4 untouched), `@Size(2048)` + `@Pattern` on both push-token DTOs, tests moved to R56 / R57, and `PushTokenCapMigrationIT` (NTM-E53); NTM-E56 by test. NTM-E50, E51, E52, E54 are already built: part G adds tests only. Part H moves `api/` and the dictionary to 2048 and the characters. QA NTM-R53-V1 replaced by NTM-R56-V1, R57-V1, E55-V1, E53-C1, plus android NTM-E50-M1, E51-C1, E52-M1, E54-M1. Plan §10 Q2 (whitespace-only token) | /sdd:amend v5.0.0 (8dd6fad) answering verify gaps NTM-Q31 … NTM-Q37, and v6.0.0 (f4c8dc8) answering plan Q2 (NTM-E55 → E56, NTM-D56) | Vuththana |
