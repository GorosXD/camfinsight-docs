# Spec: Notifications on mobile (center and push)

**Prefix:** `NTM` · **Status:** ✅ Approved · **Version:** v2.5.0 · **Author:** Vuththana
**Tables:** `notifications`, `users.notification_preferences` (identity-service; not yet in the data dictionary — see §8), new `push_tokens` (§8)
**Endpoints:** identity-service `/api/v1/notifications/**` (exist in code, not yet in `api/`) · new ones listed in §9
**Routes (Android destinations):** `Notifications` (owner, manager) · `NotificationSettings` (owner, manager)

<!--
HOW TO WRITE THIS FILE (delete nothing below; replace the <placeholders>)
- Status is one of: 📝 Draft · ✅ Approved · ⟳ Amending. Only /sdd:clarify sets ✅; only /sdd:amend sets ⟳.
- An ID is DEFINED where it starts a list item or table row in bold: `- **XXX-R01** …` or `| **XXX-D01** | …`.
  Anywhere else, write it plain (XXX-R01) — that's a reference. sddtool.py relies on this.
- IDs are never renumbered or reused. Withdraw by striking through: `- ~~**XXX-R07**~~ withdrawn in v5.3.0 — <why>`.
- One requirement = one testable sentence: who, when, what happens, which message. Long rules get several IDs.
- Values (limits, lengths, enums) come from the data dictionary. Quote the field, don't restate its type.
- Section numbers (§1 … §10, and §4a, §4b …) are stable once approved: other docs link to them.
-->

## 1. Summary and scope

Owners and branch managers already get in-app, email and Telegram alerts (overdue invoices, low stock, unmatched bank
lines, reviews to sign off …) from identity-service, but only the web app shows them. This feature brings them to the
Android app: a notification center reached from a bell added to the Dashboard header (`DashboardHeader`, NTM-D26), a
per-type notification settings screen reached from the existing Settings screen, and push
notifications on the phone. The center and settings use the existing identity-service notification API; push is new
and adds device-token storage and an FCM sender in identity-service.

- **In:** a Dashboard bell with unread count (new in `DashboardHeader`); notification center (tabs All / Unread / Financial / Stock, owner branch
  filter, day groups, read / unread, mark all read with Undo); notification settings (per alert type: In-app, Push,
  Telegram status); push registration and sign-out unregistration; receiving and opening a push; identity-service
  sending push for the alerts a user has push turned on for.
- **Out:**
  - Which alerts exist, who receives them, their texts, de-duplication and retention: the web spec `notifications`
    (prefix `NTF`), not in this docs repo; this spec cites its rules and the identity-service code.
  - New alert types (e.g. "invoice paid", "review signed off" seen in designs) — a later NTF change (NTM-D06).
  - Opening the related record (bill, item, statement) from a notification or push (NTM-D04).
  - Linking or unlinking Telegram on mobile (shown read-only, NTM-D05); email delivery settings.
  - iOS, the web app's push, notification sounds per type, quiet hours.
- **Depends on:** Android sign-in and session (`auth`), the team's app shell (Dashboard, Menu → Settings), the
  `NTF` rules for recipients and branch scope (NTF-R25 … NTF-R29), in-app lock (NTF-D09) and owner branch filter
  (NTF-R73) as implemented in identity-service.

## 2. Roles and branch scope

| Role | Can | Scope |
|---|---|---|
| `BUSINESS_OWNER` | See and read notifications, mark all read, change notification settings, receive push | Everything the server sends them; can filter the center by one branch or "All branches" |
| `BRANCH_MANAGER` | Same | Only what the server sends them (their own branch and business-wide alerts, NTF-R25 … NTF-R29); no branch filter |

- Scope is enforced by identity-service: the list, counts and read actions only ever touch the caller's own
  notifications; a manager's `branch_id` filter is ignored by the server. The app never sends another user's id.
- Which alert types a role receives comes from the server (`AlertType`: owner / manager flags); the settings screen
  lists exactly the types the server returns for the caller.
- Business-wide notifications have no branch (`branch` null) and show "All branches" (NTM-R07).
- A deactivated user or a deleted business loses their push tokens and gets no push (NTM-R33).

## 3. Decisions

| ID | Topic | Decision |
|---|---|---|
| **NTM-D01** | Scope | One mobile spec for the notification center, notification settings and push. Server-side alert rules stay in NTF and are cited, not copied. (User, 2026-11-16) |
| **NTM-D02** | Tabs | All / Unread / Financial / Stock as in Figma 18.21: Unread = `unread=true`, Financial = `feed=FINANCIAL`, Stock = `alert_type=STOCK` (a stock alert also appears under Financial). No API change. (User, 2026-11-16) |
| **NTM-D03** | Push per type | Push is a third channel per alert type next to In-app and Telegram, chosen by the user in the settings screen and stored with the existing preferences. (User, 2026-11-16) |
| **NTM-D04** | Tapping | Tapping a notification card marks it read; tapping a push opens the notification center and marks that notification read. Neither opens the related record in this version. (User, 2026-11-16) |
| **NTM-D05** | Settings on mobile | A "Notifications" row in the Preferences section of the existing Settings screen (Menu → Settings opens `Profile`) lists the caller's alert types with In-app (shown locked on where the server locks it, NTF-D09), Push, and Telegram as a read-only status ("Set up on the web" when not linked). (User, 2026-11-16) |
| **NTM-D06** | Alert types | The app shows the alert types the server sends; an unknown future type is shown with its `message` and a generic icon and title. Missing types from the design are a later NTF change. (User, 2026-11-16) |
| **NTM-D07** | Push text | A push shows only the alert type's title and the branch name ("All branches" for business-wide), never amounts, names or the `message`, because it can appear on a locked screen. (User, 2026-11-16) |
| **NTM-D08** | Unread count | The Dashboard bell shows the unread total from `GET /notifications/unread-count`, refreshed when the Dashboard opens, when the app returns to the foreground and when a push arrives while the app is open. (User, 2026-11-16) |
| **NTM-D09** | Devices | A user may receive push on several phones. Signing in registers the phone's token for that user; signing out unregisters it; the server drops tokens FCM reports as invalid and all tokens of a deactivated user. (User, 2026-11-16) |
| **NTM-D10** | Mark all read | "Mark all read" acts at once, without confirmation, and offers Undo in a snackbar (NTM-D18). (User, 2026-11-16) |
| **NTM-D11** | Branch filter | the owner's center has a branch chip ("All branches" + active branches) sent as `branch_id`; every card shows its branch. A manager has no chip. (User, 2026-11-16) |
| **NTM-D12** | Mark all and tabs | "Mark all read" applies to the current tab and the owner's branch filter (`feed`, `alert_type`, `branch_id` of `POST /read-all`); on All and Unread it marks everything in the branch filter. (User, 2026-11-16) |
| **NTM-D13** | Push while open | a push that arrives while the app is in the foreground shows no system banner; it refreshes the bell count and, if the center is open, its first page. (User, 2026-11-16) |
| **NTM-D14** | Endpoints for tokens | identity-service adds `POST /api/v1/notifications/push-tokens` (register or refresh, idempotent per token) and `POST /api/v1/notifications/push-tokens/unregister` (remove one token), both for the signed-in user only, in the team's action style (NTM-Q06). (User, 2026-11-16) |
| **NTM-D15** | Sign-out | On sign-out the app sends its push token with `POST /auth/sign-out` (NTM-D22), then always deletes the phone's FCM token locally, so a token the server still holds becomes invalid and is dropped on the next send (NTM-R32). (User, 2026-11-16) |
| **NTM-D16** | Firebase config | One team Firebase project with a dev and a prod Android app. The app's `google-services.json` and identity-service's service-account key are handed out per environment like `.env` and never committed: `google-services.json` is added to the android-ui `.gitignore`, identity-service reads the key from `.env` (TEAM-RULES secrets rule; NTM-Q01). (User, 2026-11-16) |
| **NTM-D17** | Push default | A type's Push switch defaults to that type's Telegram default (NTF `defaultTelegram`): on for due date, stock, monthly report, integration failure, budget exceeded, review submitted / returned and daily sales; off for the others. A stored value always wins. (User, 2026-11-16, NTM-Q02) |
| **NTM-D18** | Undo | Undo after "Mark all read" re-marks unread, one call each, the notifications that were unread on the pages the app had loaded; unloaded ones stay read. No server change. (User, 2026-11-16, NTM-Q03) |
| **NTM-D19** | Permission prompt | On Android 13+ the app asks for the notification permission once, on the Dashboard right after the first sign-in on that phone. (User, 2026-11-16, NTM-Q04) |
| **NTM-D20** | Preferences race | Saving preferences replaces the whole matrix; the last save wins (NTF-E07). After every save the app reloads the matrix. (User, 2026-11-16, NTM-Q05) |
| **NTM-D21** | Shared phone | Registering a token already held by another user moves it to the caller (upsert on `token`); the previous user gets no more pushes on that phone. (User, 2026-11-16, NTM-Q07) |
| **NTM-D22** | Sign-out token | `POST /auth/sign-out` (needs only the refresh token) accepts an optional `push_token` and deletes it, so sign-out unregisters even when the access token has expired. (User, 2026-11-16, NTM-Q08) |
| **NTM-D23** | Stale tokens | Every registration refreshes `last_seen_at`; a daily server job deletes tokens not seen for 60 days. (User, 2026-11-16, NTM-Q09) |
| **NTM-D24** | Push grouping | No collapsing or grouping in this version: one push per notification. (User, 2026-11-16, NTM-Q10) |
| **NTM-D25** | Unread tab | A card read on the Unread tab stays, shown as read, until the next reload (refresh, tab switch, reopening). (User, 2026-11-16, NTM-Q11) |
| **NTM-D26** | Bell | This feature adds the bell with its badge to the team's `DashboardHeader` (the Dashboard owner is told). (User, 2026-11-16, NTM-Q12) |
| **NTM-D27** | Permission denied | With the phone's notification permission denied, Push switches stay editable (the choice is stored) and NTM-M07 shows above them. (User, 2026-11-16, NTM-Q13) |
| **NTM-D28** | Older clients | A `PUT /preferences` without `push` keeps each type's stored `push` (or its default), so the web app works unchanged. (User, 2026-11-16, NTM-Q14) |

## 4. Journeys

> Diagrams are Mermaid. All server steps go through the API Gateway (`/api/v1/...`) with the user's token.

### 4a. See notifications

```mermaid
sequenceDiagram
    actor U as Owner / Manager
    participant D as Dashboard
    participant C as Notifications
    participant I as identity-service
    D->>I: GET /notifications/unread-count
    I-->>D: 200 total, financial, system
    U->>D: tap the bell
    D->>C: open
    C->>I: GET /notifications?[feed|unread|alert_type][&branch_id]
    I-->>C: 200 items + next_cursor
    U->>C: switch tab / branch, scroll, pull to refresh
    C->>I: GET /notifications?…&cursor=…
```

Realises NTM-R01 … NTM-R12.

### 4b. Read, unread and mark all

```mermaid
sequenceDiagram
    actor U as Owner / Manager
    participant C as Notifications
    participant I as identity-service
    U->>C: tap a card
    C->>I: POST /notifications/{id}/read
    U->>C: "Mark as unread" on a card
    C->>I: POST /notifications/{id}/unread
    U->>C: tap "Mark all read"
    C->>I: POST /notifications/read-all {feed, alert_type, branch_id}
    I-->>C: 200 updated_count
    U->>C: Undo (snackbar)
```

Realises NTM-R13 … NTM-R18.

### 4c. Notification settings

```mermaid
sequenceDiagram
    actor U as Owner / Manager
    participant S as NotificationSettings
    participant I as identity-service
    U->>S: Menu → Settings → Notifications row
    S->>I: GET /notifications/preferences
    I-->>S: 200 alerts {type: in_app, telegram, push, locked}, telegram_linked
    U->>S: toggle Push or In-app for a type
    S->>I: PUT /notifications/preferences {alerts}
    I-->>S: 200 / 400
```

Realises NTM-R19 … NTM-R24, NTM-R42.

### 4d. Register the phone for push

```mermaid
sequenceDiagram
    actor U as Owner / Manager
    participant A as App
    participant F as FCM
    participant I as identity-service
    U->>A: sign in (or open the app signed in)
    A->>U: ask notification permission (Android 13+, first sign-in, NTM-D19)
    A->>F: get token
    A->>I: POST /notifications/push-tokens {token}
    F-->>A: new token (refresh)
    A->>I: POST /notifications/push-tokens/unregister {old token}
    A->>I: POST /notifications/push-tokens {new token}
    U->>A: sign out
    A->>I: POST /auth/sign-out {refresh_token, push_token}
    A->>F: delete token
```

Realises NTM-R25 … NTM-R29, NTM-R41, NTM-R43.

### 4e. Receive and open a push

```mermaid
sequenceDiagram
    participant Q as RabbitMQ alert event
    participant I as identity-service
    participant F as FCM
    participant A as App
    actor U as Owner / Manager
    Q->>I: alert event (NTF)
    I->>I: create in-app notification per recipient (NTF)
    I->>F: send to each token of recipients with push on
    F-->>A: push {title, branch, notification_id}
    U->>A: tap the push
    A->>I: POST /notifications/{id}/read
    A->>U: Notifications (All)
```

Realises NTM-R30 … NTM-R37, NTM-R40.

## 5. Requirements

### Notification center (§4a)

- **NTM-R01** · Owner, Manager · §4a — The Dashboard header shows a bell button with the unread total from `GET /notifications/unread-count` as a badge (hidden at 0, "99+" above 99); tapping it opens the notification center (NTM-D08).
- **NTM-R02** · Owner, Manager · §4a — The unread count is reloaded when the Dashboard opens, when the app returns to the foreground and when a push arrives while the app is open; a failed reload keeps the last count.
- **NTM-R03** · Owner, Manager · §4a — The notification center shows the tabs All, Unread, Financial and Stock; Unread shows the unread total next to its label; each tab loads `GET /notifications` with its filter (NTM-D02).
- **NTM-R04** · Owner · §4a — An owner can filter the center by one active branch or "All branches" (the start value), sent as `branch_id`; a manager sees no branch filter (NTM-D11).
- **NTM-R05** · Owner, Manager · §4a — Notifications are listed newest first in groups by the Asia/Phnom_Penh day of `created_at` ("Today" and "Yesterday" also in Asia/Phnom_Penh, whatever the phone's zone), headed "Today · {d MMM}", "Yesterday · {d MMM}" or "{d MMM yyyy}".
- **NTM-R06** · Owner, Manager · §4a — Each card shows the alert type's icon and title (NTM-M10 … NTM-M22), the `message` in full, the local time ("HH:mm" today, else "d MMM, HH:mm") and the branch.
- **NTM-R07** · Owner, Manager · §4a — The branch on a card is `branch.branch_name`, "All branches" when `branch` is null, and is marked "(inactive)" when `branch.is_active` is false.
- **NTM-R08** · Owner, Manager · §4a — An unread card is visibly marked (dot and tinted background); a read card is not.
- **NTM-R09** · Owner, Manager · §4a — The list loads the next page with `cursor` = the last `next_cursor` when the user scrolls to the end, until `next_cursor` is null.
- **NTM-R10** · Owner, Manager · §4a — Pull-to-refresh reloads the first page of the current tab and the unread count.
- **NTM-R11** · Owner, Manager · §4a — A tab with no notifications shows NTM-M01 (All, Financial, Stock) or NTM-M02 (Unread).
- **NTM-R12** · Owner, Manager · §4a — An alert type the app doesn't know is shown with a generic icon, the title NTM-M21 and its `message` (NTM-D06).

### Read, unread and mark all (§4b)

- **NTM-R13** · Owner, Manager · §4b — Tapping an unread card marks it read at once on screen and sends `POST /notifications/{id}/read`; on failure the card returns to unread and NTM-M03 shows. On the Unread tab the card stays, shown as read, until the next reload (NTM-D25).
- **NTM-R14** · Owner, Manager · §4b — Each read card offers "Mark as unread", which sends `POST /notifications/{id}/unread` and marks the card unread; on failure it returns to read and NTM-M03 shows.
- **NTM-R15** · Owner, Manager · §4b — The top bar's "Mark all read" sends `POST /notifications/read-all` with the current tab's `feed` or `alert_type` and the owner's `branch_id`, marks the affected cards read and shows NTM-M04 with Undo; it is disabled when the current view has no unread notification (NTM-D10, NTM-D12).
- **NTM-R16** · Owner, Manager · §4b — Undo sends `POST /notifications/{id}/unread` for each notification that was unread on the loaded pages before "Mark all read"; if any call fails, NTM-M05 shows (NTM-D18).
- **NTM-R17** · Owner, Manager · §4b — Every read, unread or mark-all action reloads the unread count, so the Unread tab label and the Dashboard badge stay in step.
- **NTM-R18** · Owner, Manager · §4b — Tapping a card does nothing besides marking it read (NTM-D04).

### Notification settings (§4c)

- **NTM-R19** · Owner, Manager · §4c — The Settings screen (Menu → Settings) shows a "Notifications" row in its Preferences section, which opens the notification settings screen with the alert types returned by `GET /notifications/preferences`, each with its title (NTM-M10 … NTM-M20, NTM-M22).
- **NTM-R20** · Owner, Manager · §4c — Each type has an In-app switch, shown on and disabled when the server marks it `locked` (NTF-D09), and a Push switch.
- **NTM-R21** · Owner, Manager · §4c — The screen shows Telegram per type as "On" / "Off" and, when `telegram_linked` is false, the note NTM-M06; Telegram can't be changed on mobile (NTM-D05).
- **NTM-R22** · Owner, Manager · §4c — Changing a switch saves at once with `PUT /notifications/preferences` carrying every type's `in_app`, `telegram` and `push`; after a save the app reloads the matrix (NTM-D20); on failure the switch returns to its previous value and NTM-M03 or the server message shows.
- **NTM-R23** · Owner, Manager · §4c — When the phone blocks notifications for the app, the screen shows NTM-M07 with "Open settings" (the phone's app notification settings) above the list; Push switches stay editable and their choice is saved (NTM-D27).
- **NTM-R24** · Owner, Manager · §4c — No notification and no push exist for a type with In-app off: the server creates the notification only when In-app is on, and push follows it.

### Push registration (§4d)

- **NTM-R25** · Owner, Manager · §4d — After sign-in, and on every app start while signed in, the app sends the phone's FCM token with `POST /notifications/push-tokens`; the app keeps the last token it registered, and when FCM issues a new token it sends that kept one to `POST /notifications/push-tokens/unregister` and registers the new one (NTM-D09, NTM-D14, NTM-D23).
- **NTM-R26** · Owner, Manager · §4d — On Android 13 and later the app asks for the notification permission once, on the Dashboard right after the first sign-in on the phone (NTM-D19); later the user can only turn it on from the phone's settings (NTM-M07); denying it keeps in-app notifications working.
- **NTM-R27** · Owner, Manager · §4d — A failed token registration is retried on the next app start or token refresh and never shows an error to the user.
- **NTM-R28** · Owner, Manager · §4d — Signing out sends the phone's token as `push_token` with `POST /auth/sign-out`, which deletes it on the server, and then deletes the FCM token on the phone whether or not the call succeeded (NTM-D15, NTM-D22).
- **NTM-R29** · Owner, Manager · §4d — The next user who signs in on the same phone registers the phone's token for themselves; a token still held by another user is moved to the caller, so the previous user gets no more pushes on that phone (NTM-D21).

### Sending and opening push (§4e)

- **NTM-R30** · Server · §4e — When identity-service creates an in-app notification for a recipient (NTF), it sends one push to each of that recipient's tokens if push is on for that alert type (NTM-D03).
- **NTM-R31** · Server · §4e — The push carries the alert type's title (NTM-M10 … NTM-M20, NTM-M22) as its title, the branch name or "All branches" as its text, and `notification_id` and `alert_type` as data; no amounts, names or `message` (NTM-D07); each notification is its own push, never collapsed (NTM-D24).
- **NTM-R32** · Server · §4e — A token FCM reports as unregistered or invalid is deleted.
- **NTM-R33** · Server · §4e — Deactivating a user (including managers deactivated with their branch), deleting a business, and the anonymisation purge of a user delete all their tokens, in the same transaction as that change; no push is sent to them afterwards.
- **NTM-R34** · Server · §4e — A failed or slow push never delays or blocks creating the in-app notification or the other channels.
- **NTM-R35** · Owner, Manager · §4e — Tapping a push opens the notification center on All and marks that notification read (`POST /notifications/{id}/read`); if the app was signed out, it opens sign-in first and then the center.
- **NTM-R36** · Owner, Manager · §4e — Pushes are FCM data messages that the app turns into a system notification itself; one that arrives while the app is in the foreground shows no system notification, reloads the unread count and, when the center is open, its first page (NTM-D13).
- **NTM-R37** · Owner, Manager · §4e — Pushes appear in one Android notification channel named NTM-M08.

### Common

- **NTM-R38** · Owner, Manager · all — Every screen shows a loading state; a failed load shows NTM-M03 (connection) or the server message with Retry.
- **NTM-R39** · Owner, Manager · all — Every user-visible string is an Android string resource (English only for now).
- **NTM-R40** · Server · §4e — The push is sent after the notification's transaction commits (as Telegram delivery is), and only for a notification that was newly created, so a redelivered alert event sends no second push.
- **NTM-R41** · Server · §4d — Every registration refreshes the token's `last_seen_at`; a daily background job (identity-service `job/`, with ShedLock like the notification purge job, TRD §6) deletes tokens not seen for 60 days (NTM-D23).
- **NTM-R42** · Server · §4c — The preferences store a `push` value per alert type; a type with no stored `push` reads as its Telegram default (NTM-D17), and a `PUT` without `push` keeps the stored values (NTM-D28).
- **NTM-R43** · Server · §4d — `POST /auth/sign-out` accepts an optional `push_token`; when given, that token is deleted for the signing-out user (NTM-D22).

### Fields

| Field | Rule | Requirement |
|---|---|---|
| `notifications.notification_id` | read only; id for read / unread and the push data | NTM-R13, NTM-R31 |
| `notifications.feed` | `FINANCIAL` / `SYSTEM` (server) | NTM-R03 |
| `notifications.alert_type` | the server's `AlertType` values | NTM-R06, NTM-R12 |
| `notifications.message` | server text, shown in full | NTM-R06 |
| `notifications.is_read`, `read_at` | server state | NTM-R08, NTM-R13 |
| `notifications.created_at` | UTC; shown in Asia/Phnom_Penh | NTM-R05 |
| `notifications.branch_id` | null = business-wide | NTM-R07 |
| `users.notification_preferences` | per type `in_app`, `telegram`, new `push` (§8) | NTM-R20, NTM-R22 |
| `push_tokens.token` | new (§8) | NTM-R25 |

## 6. Messages

| ID | HTTP · code | Text (English; Khmer later) | Used by |
|---|---|---|---|
| **NTM-M01** | — | "No notifications yet." | NTM-R11 |
| **NTM-M02** | — | "You're all caught up." | NTM-R11 |
| **NTM-M03** | connection error | "Can't reach the server. Check your connection and try again." | NTM-R13, NTM-R14, NTM-R22, NTM-R38 |
| **NTM-M04** | `200` read-all | "{n} marked as read" · action "Undo" | NTM-R15 |
| **NTM-M05** | — | "Couldn't undo. Mark them unread one by one." | NTM-R16 |
| **NTM-M06** | — | "Telegram isn't linked. Set it up on the web." | NTM-R21 |
| **NTM-M07** | — | "Notifications are off for CamFinSight on this phone." · action "Open settings" | NTM-R23 |
| **NTM-M08** | — | Channel name "Alerts" | NTM-R37 |
| **NTM-M09** | `404 · NOT_FOUND` | "This notification is no longer available." | NTM-E06 |
| **NTM-M10** | — | `DUE_DATE` title — "Payment due" | NTM-R06, NTM-R31 |
| **NTM-M11** | — | `STOCK` — "Stock running low" | NTM-R06, NTM-R31 |
| **NTM-M12** | — | `IMPORT` — "Import finished" | NTM-R06, NTM-R31 |
| **NTM-M13** | — | `MONTHLY_REPORT` — "Monthly report ready" | NTM-R06, NTM-R31 |
| **NTM-M14** | — | `UNMATCHED_LINES` — "Bank lines unmatched" | NTM-R06, NTM-R31 |
| **NTM-M15** | — | `INTEGRATION_FAILURE` — "Integration stopped syncing" | NTM-R06, NTM-R31 |
| **NTM-M16** | — | `REVENUE_DRIFT` — "Revenue is off its usual pace" | NTM-R06, NTM-R31 |
| **NTM-M17** | — | `BUDGET_EXCEEDED` — "Budget exceeded" | NTM-R06, NTM-R31 |
| **NTM-M18** | — | `REVIEW_SUBMITTED` — "Reconciliation to sign off" | NTM-R06, NTM-R31 |
| **NTM-M19** | — | `CATEGORY_REQUEST` — "Category requested" | NTM-R06, NTM-R31 |
| **NTM-M20** | — | `DAILY_SALES` — "Daily sales summary" | NTM-R06, NTM-R31 |
| **NTM-M21** | — | Unknown type — "Notification" | NTM-R12 |
| **NTM-M22** | — | `REVIEW_RETURNED` — "Reconciliation returned" | NTM-R06, NTM-R31 |
| **NTM-M23** | — | Tab labels "All", "Unread", "Financial", "Stock"; Unread with its count "Unread {n}" | NTM-R03 |
| **NTM-M24** | — | Actions "Mark all read", "Mark as unread" | NTM-R14, NTM-R15 |
| **NTM-M25** | — | Branch labels "All branches" and the suffix " (inactive)" | NTM-R04, NTM-R07, NTM-R31 |
| **NTM-M26** | — | Day headers "Today · {d MMM}", "Yesterday · {d MMM}", "{d MMM yyyy}"; badge cap "99+" | NTM-R01, NTM-R05 |
| **NTM-M27** | — | Settings labels "Notifications", "In-app", "Push", "Telegram", "On", "Off" | NTM-R19 … NTM-R21 |
| **NTM-M28** | server error | The server's `message` for any other error, shown as is with Retry | NTM-R38 |

## 7. Edge cases

- **NTM-E01** · Connectivity — The center opens with no connection → loading, then NTM-M03 with Retry; nothing cached is shown.
- **NTM-E02** · Connectivity — Loading the next page fails while earlier cards are shown → the cards stay and the list end shows NTM-M03 with Retry.
- **NTM-E03** · Connectivity — Marking a card read fails → the card returns to unread and NTM-M03 shows (NTM-R13).
- **NTM-E04** · Connectivity — "Mark all read" fails → nothing changes on screen and NTM-M03 shows; no Undo is offered.
- **NTM-E05** · Concurrency — The user reads notifications on the web or another phone → the center and bell show it after the next reload (refresh, tab switch, resume, push).
- **NTM-E06** · State — A notification on screen was purged on the server (read more than 90 days ago, NTF) and the user acts on it → `404`; the card is removed and NTM-M09 shows.
- **NTM-E07** · State — A notification belongs to a branch that is now inactive → it stays listed with "(inactive)" after the branch name (NTM-R07); an inactive branch is not offered in the owner's filter.
- **NTM-E08** · Permissions — A manager is moved to another branch → the center shows what the server now returns; old branch alerts the server still sends remain (NTF rules).
- **NTM-E09** · Permissions — A manager's app sends a `branch_id` → the server ignores it (NTF-R73); the app never sends one for a manager (NTM-R04).
- **NTM-E10** · Data — An unknown `alert_type` arrives → generic icon, title NTM-M21 and its `message` (NTM-R12); a push for it uses the title NTM-M21.
- **NTM-E11** · Data — A very long `message` → shown in full on the card, wrapping; nothing is cut.
- **NTM-E12** · Time — A notification created at 23:30 local time appears under that local day, not the UTC day (NTM-R05).
- **NTM-E13** · Empty — The Unread tab after everything is read → NTM-M02; "Mark all read" is disabled.
- **NTM-E14** · Actions — The user taps "Mark all read" twice quickly → one request; the second tap is ignored while the first is sending.
- **NTM-E15** · Actions — The user taps Undo after the snackbar is gone, or leaves the screen → no Undo; the notifications stay read.
- **NTM-E16** · Push — The phone has the notification permission denied → no system notification shows; the bell count and center still work (NTM-R26).
- **NTM-E17** · Push — Android 12 or earlier → no permission prompt; pushes show unless the user turned the app's notifications off in the phone's settings.
- **NTM-E18** · Push — The user taps a push while signed out → sign-in, then the center; the notification is marked read only if it belongs to the user who signed in; a `404` on the read call is ignored and the center opens normally (as NTM-E19).
- **NTM-E19** · Push — The user taps a push for a notification that was already read or purged → the center opens; a `404` on the read call is ignored.
- **NTM-E20** · Push — Signing out with no connection → the sign-out call fails, the phone's FCM token is deleted anyway, and the server drops the stale token on its next send or after 60 days (NTM-R28, NTM-R32, NTM-R41).
- **NTM-E21** · Push — FCM issues a new token while the app runs → it is registered at once (NTM-R25); the old one is dropped by the server when FCM reports it invalid.
- **NTM-E22** · Push — The same user is signed in on two phones → both receive the push; reading it on one phone doesn't remove the system notification on the other.
- **NTM-E23** · Push — A user turns Push off for a type while a push for it is already on the way → that push may still arrive; later ones don't.
- **NTM-E24** · Push — The app is killed when a push is tapped → the app starts, restores the session and opens the center (NTM-R35).
- **NTM-E25** · Push — FCM is unreachable from identity-service → the in-app notification and Telegram are delivered as usual; that push is lost (NTM-R34).
- **NTM-E26** · Settings — Saving a switch fails → the switch returns to its old value and NTM-M03 shows (NTM-R22).
- **NTM-E27** · Settings — The server returns a type with `locked` true → its In-app switch shows on and can't be changed (NTM-R20).
- **NTM-E28** · Session — The token expires while the center is open → the existing refresh runs; if it fails, sign-in; the token is removed by the sign-out call, which needs no access token (NTM-R28).
- **NTM-E29** · Permissions — A different user signs in on the same phone while the old token is still registered to the previous user → the token moves to the new user (NTM-D21); the previous user gets no more pushes on that phone.
- **NTM-E30** · Permissions — A manager's branch is deactivated while a push is on its way → the push may still arrive; tapping it opens the center on All, which lists only what the server now returns.
- **NTM-E31** · Time — The phone's clock or zone differs from Asia/Phnom_Penh → day groups and "Today" follow Asia/Phnom_Penh (NTM-R05).
- **NTM-E32** · Data — A first-time user with no stored preferences → the screen shows the server's defaults for the types returned, Push per NTM-D17.
- **NTM-E33** · Actions — The user toggles several switches quickly → saves are sent one after another; the last one wins; a failed save reverts only its own switch.
- **NTM-E34** · Actions — The same notification is opened from a card and from a push, or a push is tapped twice → repeated read calls are harmless.
- **NTM-E35** · Push — The user turns Push off for a type while an earlier push for it is still in the phone's tray → it stays; tapping it works (NTM-R35).
- **NTM-E36** · Text — A long branch name → the card shows it in full on its own line; a push shows it shortened by the phone.
- **NTM-E37** · Push — A push for a notification of another user (stale token on a shared phone) is tapped → the read call answers `404` (never `403`, NTF-E20) and is ignored; the center opens.

## 8. Data dictionary changes

| Table.field | Change | Why | State |
|---|---|---|---|
| `notifications` (all columns) | Add the table to `data-dictionary/` as it is in identity-service V1 (owned by NTF) | Fields cited in §5 | proposed |
| `users.notification_preferences` | jsonb shape (`jsonb-shapes.md#usersnotification_preferences`, `{"version":1,"alerts":{TYPE:{in_app,telegram}}}`) gains `push` (boolean) per alert type; `version` stays 1 and a missing `push` reads as the type's Telegram default (NTM-D17); a PUT without `push` keeps the stored values (NTM-D28) | NTM-R20, NTM-R22, NTM-R30 | proposed |
| `push_tokens` | New identity-service table: `push_token_id` (uuid, PK), `user_id` (FK `users`, not null), `token` (text, not null, unique — a re-registration moves it, NTM-D21), `platform` (`ANDROID`), `created_at`, `last_seen_at` (deleted after 60 days, NTM-D23) | NTM-R25, NTM-R28, NTM-R29, NTM-R32, NTM-R33, NTM-R41 | proposed |

## 9. Endpoints

| Method · path | Service | Who | Requirements | In api/? |
|---|---|---|---|---|
| `GET /api/v1/notifications` | identity-service | Owner, Manager | NTM-R03 … NTM-R12 | 📝 exists in code, not yet listed |
| `GET /api/v1/notifications/unread-count` | identity-service | Owner, Manager | NTM-R01, NTM-R02, NTM-R17 | 📝 exists in code, not yet listed |
| `POST /api/v1/notifications/{notification_id}/read` | identity-service | Owner, Manager | NTM-R13, NTM-R35 | 📝 exists in code, not yet listed |
| `POST /api/v1/notifications/{notification_id}/unread` | identity-service | Owner, Manager | NTM-R14, NTM-R16 | 📝 exists in code, not yet listed |
| `POST /api/v1/notifications/read-all` | identity-service | Owner, Manager | NTM-R15 | 📝 exists in code, not yet listed |
| `GET /api/v1/notifications/preferences` | identity-service — **changed** (adds `push`) | Owner, Manager | NTM-R19 … NTM-R21 | 📝 planned |
| `PUT /api/v1/notifications/preferences` | identity-service — **changed** (`AlertChannelsRequest` gains optional `push`, same strict-boolean rule, kept when absent; `AlertChannelsResponse` gains `push`) | Owner, Manager | NTM-R22, NTM-R42 | 📝 planned |
| `POST /api/v1/notifications/push-tokens` | identity-service — **new** | Owner, Manager | NTM-R25, NTM-R27, NTM-R29, NTM-R41 | 📝 planned |
| `POST /api/v1/notifications/push-tokens/unregister` | identity-service — **new** | Owner, Manager | NTM-R25 | 📝 planned |
| `POST /api/v1/auth/sign-out` | identity-service — **changed** (optional `push_token`) | Owner, Manager | NTM-R28, NTM-R43 | 📝 planned |
| `GET /api/v1/branches` | identity-service (`status` defaults to active) | Owner | NTM-R04 | 📝 exists in code (as in `mobile-sales-management` §9, MSL-R06) |

The unchanged notification endpoints marked "exists in code, not yet listed" stay out of `api/` until the NTF docs are brought into this repo; only the new and changed ones are listed in `api/identity-service.md`.

Push sending runs inside identity-service where alerts are fanned out (`AlertEventListener`, next to Telegram delivery); FCM is called only by identity-service.

## 10. Questions

- **NTM-Q01** · §1 — Who owns the Firebase project and where do the app's client file and the server key live for each environment (local, dev, prod)? *Options:* (a) one team Firebase project with dev and prod apps, files handed out like `.env` — simple; (b) a project per environment — cleaner separation, more setup. *Suggested:* (a), per NTM-D16. → NTM-D16 (answered 2026-11-16 by Vuththana)
- **NTM-Q02** · §8 — Default of the new Push switch for existing and new users? *Options:* (a) on for every type the user receives in-app — they get push right away; (b) the same as each type's Telegram default (NTF) — urgent types only; (c) off for all — the user opts in. *Suggested:* (b). → NTM-D17 (answered 2026-11-16 by Vuththana)
- **NTM-Q03** · §4b — What does Undo after "Mark all read" restore? `read-all` returns only a count. *Options:* (a) the unread notifications the app had loaded, marked unread one by one — no server change, older ones stay read; (b) a server change: `read-all` returns the changed ids (or accepts them back) so Undo restores exactly those — accurate, needs identity work. *Suggested:* none; (a) is cheaper, (b) is exact. → NTM-D18 (answered 2026-11-16 by Vuththana)
- **NTM-Q04** · §4d — When does the app ask for the notification permission (Android 13+)? *Options:* (a) on the Dashboard right after the first sign-in — most users allow it; (b) only when the user turns a Push switch on — fewer prompts, fewer pushes; (c) on first sign-in with a short explanation screen first. *Suggested:* (a). → NTM-D19 (answered 2026-11-16 by Vuththana)
- **NTM-Q05** · §4c — `PUT /preferences` replaces the whole matrix (last save wins, NTF-E07), and the app sends Telegram unchanged; a phone save could undo a Telegram change made on the web a moment before. *Options:* (a) accept it, the app reloads the matrix after each save — no server change; (b) the server keeps the stored `telegram` when a PUT omits it — safe, needs identity change; (c) the app re-reads before every PUT. *Suggested:* (a). → NTM-D20 (answered 2026-11-16 by Vuththana)
- **NTM-Q06** · §9 — Shape of unregister? *Options:* (a) `POST /notifications/push-tokens/unregister {token}` — works everywhere; (b) `DELETE /push-tokens/{token}` — token in URL and access logs; (c) `DELETE` with a body — some clients and gateways drop it. *Suggested:* (a). → NTM-D14 (answered 2026-11-16 by Vuththana)
- **NTM-Q07** · §8 — A token already registered to another user is registered again (shared phone, failed offline sign-out)? *Options:* (a) move it to the caller (upsert on token) — the previous user stops getting pushes at once; (b) reject with `409`; (c) keep both rows. *Suggested:* (a). → NTM-D21 (answered 2026-11-16 by Vuththana)
- **NTM-Q08** · §4d — Unregister needs a valid access token, which an expired session or an offline sign-out doesn't have. *Options:* (a) accept it: the token stays until FCM reports it invalid or it is moved (NTM-Q07); (b) `POST /auth/sign-out` also accepts an optional `push_token` and removes it (it needs only the refresh token). *Suggested:* (b) for a clean sign-out. → NTM-D22 (answered 2026-11-16 by Vuththana)
- **NTM-Q09** · §8 — When does the server drop a token FCM never reports invalid (an abandoned phone)? *Options:* (a) after N days without a refresh (refreshed on every app start, NTM-R25), N = 60; (b) only on FCM errors. *Suggested:* (a), 60 days. → NTM-D23 (answered 2026-11-16 by Vuththana)
- **NTM-Q10** · §4e — An owner of several branches can get many pushes (stock, daily sales). Collapse them? *Options:* (a) no collapsing in this version; (b) a collapse key per alert type, a newer push replaces the older one; (c) one Android group per alert type. *Suggested:* (a). → NTM-D24 (answered 2026-11-16 by Vuththana)
- **NTM-Q11** · §4b — On the Unread tab, a card the user taps becomes read: does it leave the list at once? *Options:* (a) it stays until the next reload — nothing jumps; (b) it disappears at once. *Suggested:* (a). → NTM-D25 (answered 2026-11-16 by Vuththana)
- **NTM-Q12** · §1 — `origin/dev`'s `DashboardHeader` has no bell. *Options:* (a) this feature adds it to the team's shell (with the team's OK); (b) wait for the team to add it and depend on that. *Suggested:* (a). → NTM-D26 (answered 2026-11-16 by Vuththana)
- **NTM-Q13** · §4c — With the phone's notification permission denied, Push switches do nothing. *Options:* (a) editable, with the NTM-M07 banner (as written); (b) disabled until the permission is granted. *Suggested:* (a). → NTM-D27 (answered 2026-11-16 by Vuththana)
- **NTM-Q14** · §9 — A `PUT /preferences` without `push` (today's web app) — what happens? *Options:* (a) the stored `push` values are kept — the web keeps working unchanged; (b) `400` — the matrix must be whole, the web must send `push` too. *Suggested:* (a). → NTM-D28 (answered 2026-11-16 by Vuththana)
