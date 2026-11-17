# Version History

Every change to these docs gets an entry here, newest first. Code repos read it to see what changed.

## Versioning

`MAJOR.MINOR.PATCH`

- **MAJOR:** breaking for code that follows the docs: a field or table renamed or removed, a type, length or enum changed, an approved spec's behaviour changed or withdrawn.
- **MINOR:** added: a table, field, endpoint or spec, or a spec approved (📝 → ✅).
- **PATCH:** wording, plans, tasks, fixes that change no behaviour or schema.

## Entry format

```markdown
## vX.Y.Z — YYYY-MM-DD
**Author:** <name>

- `<file>`: <what changed>

**Impact:** <which repos must update, or "None">
```

---

## v6.1.0 — 2026-11-17
**Author:** Vuththana

- `specs/notification-mobile/spec.md`: amended (✅ v6.1.0), answering NTM-Q38 → NTM-D57: a push `token` is cleaned like every JSON string (NBSP, zero-width, word joiner and BOM removed anywhere; whitespace trimmed at the ends) before the 2048 and character checks. Added NTM-E57; NTM-E56 and NTM-D56 reworded to "after cleaning" (no behaviour change).

**Impact:** identity-service: one test for NTM-E57 (no code change). android-ui: none.

## v6.0.1 — 2026-11-17
**Author:** Vuththana

- `specs/notification-mobile/plan.md`: Plan: notification-mobile changed for spec v6.0.0 (approved by Vuththana). Part F: a `V5` migration (delete stored tokens over 2048 or with other characters, replace the 4096 check with 2048, add a character check; V4 untouched), `@Size(2048)` and `@Pattern` on both push-token DTOs checked after not-blank, tests moved to NTM-R56 / R57, NTM-E56 and a migration test for NTM-E53, verified on a local Postgres. Part G: tests only for NTM-E50, E51, E52, E54. Part H: `api/` and the dictionary to 2048 and the characters. Plan Q2 answered (blank wins).
- `specs/README.md`: notification-mobile stage `tasks`.

**Impact:** None (plan only).

## v6.0.0 — 2026-11-17
**Author:** Vuththana

- `specs/notification-mobile/spec.md`: amended (✅ v6.0.0), answering plan Q2. Withdrawn NTM-E55 → E56: a blank or whitespace-only push `token` gets "This field is required." first; length and characters are checked after it (new NTM-D56).

**Impact:** identity-service: the push-token request DTOs check not-blank before length and characters (validation order); the NTM-R25 blank tests keep "This field is required.". android-ui: none.

## v5.0.0 — 2026-11-17
**Author:** Vuththana

- `specs/notification-mobile/spec.md`: amended and re-approved (✅ v5.0.0), answering the verify gaps NTM-Q31 … NTM-Q37 → NTM-D49 … NTM-D55.
  - Withdrawn: NTM-R53 → R56 (token at most 2048 characters, below the unique index's size limit) and R57 (only `A–Z a–z 0–9 _ : . -`); NTM-D46 → D49; NTM-M31 → M32 ("Use at most 2048 characters.").
  - Added: NTM-M33 ("Enter a valid value."), NTM-D50 … D55, NTM-E50 … E55 (un-kept session, one mark-all at a time, workspace failure, over-long stored tokens deleted by the migration, failed FCM deletion, invalid characters).
  - §8: `push_tokens.token` at most 2048 and the FCM characters, by a new migration after V4 (proposed). §9: push-tokens and unregister 📝 planned.
- `specs/notification-mobile/tasks.md`: T35 reopened; T30, T35, T43 marked ⟳; progress 40 / 45.

**Impact:**
- identity-service: new `V5` migration (delete tokens over 2048 or with other characters, replace `ck_push_tokens_token_length` with 2048, add a character check); `@Size(max = 2048)` and `@Pattern` on both push-token request DTOs; `PushTokenControllerIT` and `PushTokenRepositoryIT` move from NTM-R53 to R56 / R57. V4 stays (already applied on the shared dev DB).
- android-ui: none.
- plan.md: part F, §3 push-token contracts, QA scenario NTM-R53-V1 (`/sdd:plan`).

## v4.0.1 — 2026-11-16
**Author:** Vuththana

- `specs/notification-mobile/plan.md`: Plan: notification-mobile changed for spec v4.0.0 (approved by Vuththana). New parts F (identity: push data without `title` / `body`, `push_tokens.token` ≤ 4096 with a `V4` CHECK migration, tests), G (android: forced sign-out deletes the FCM token, pushes dropped while signed out, mark all read across a tab switch, branch list re-fetched after a failure, tests) and H (docs: `api/identity-service.md`, dictionary length). No web-ui work. Plan Q1 answered: NTM-R54 as written.
- `specs/README.md`: notification-mobile stage `tasks`.

**Impact:** None (plan only).

## v4.0.0 — 2026-11-16
**Author:** Vuththana

- `specs/notification-mobile/spec.md`: amended and re-approved (✅ v4.0.0), answering the verify and QA spec gaps (NTM-Q15 … NTM-Q30 → NTM-D34 … NTM-D48).
  - Withdrawn: NTM-R43 → R48 (a replayed logout of an ended session, which Keycloak accepts, deletes the caller's push token); NTM-E38 → E42; NTM-E28 → E43 (a forced sign-out deletes the phone's FCM token, no sign-out call); NTM-R31 → R51 (push data is exactly `notification_id`, `alert_type`, `branch_name`; no `title` / `body`).
  - Added: NTM-R49 (sign-out sent without a bearer), R50 (forced sign-out deletes the FCM token), R52 (`"push": null` = absent), R53 (token at most 4096 characters, NTM-M31), R54 (no system notification while signed out), R55 (tab row scrolls when labels don't fit); NTM-E44 … E49.
  - Reworded: NTM-R06 and E31 (card times in Asia/Phnom_Penh), E18, E37, R28, D07.
  - §8: `push_tokens.token` at most 4096 (proposed). §9: sign-out, preferences, push-tokens and the push data keys 📝 planned.
- `specs/notification-mobile/tasks.md`: T05, T07, T23, T24 reopened and, with T30 and T33, marked ⟳; progress 27 / 33.

**Impact:**
- identity-service: drop `title` / `body` from `FcmPushSender` data; 4096 cap on push-token register / unregister; tests naming NTM-R31, R43, E38 (`KeycloakTokenClientTest`, `AuthControllerIT`, `PushTextTest`, `PushDeliveryTest`, `PushDeliveryIT`, `PushConfigTest`, `FcmPushSenderTest`) renamed to the new IDs; QA scenarios NTM-R43-*, NTM-E38-* re-run as R48 / E42.
- android-ui: forced sign-out deletes the FCM token; drop pushes while signed out; scrollable tab row; tests `SignOutPushTokenTest`, `PushMessageHandlerTest` renamed; QA NTM-E28-M1 re-run as E43.
- web-ui: send `POST /auth/sign-out` without a bearer (NTM-R49).
- plan.md: the sign-out and FCM data message contract sections and the NTM-E28 / E38 rows need updating (`/sdd:plan`).

## v3.1.1 — 2026-11-16
**Author:** Vuththana

- `api/identity-service.md`: the five notification-mobile endpoints (push-tokens, push-tokens/unregister, preferences GET/PUT with `push`, sign-out with `push_token`) 📝 → ✅, merged into identity-service `dev` (PR #42).
- `TRD.md` §6: `push-token-cleanup` ✅ merged.
- `specs/README.md`: notification-mobile stage `shipped` (android-ui PR #4, identity-service PR #42, both merged into `dev`).
- `specs/notification-mobile/tasks.md`: T28 and T29 done; T30 (two API QA findings, to amend) and T33 (real FCM) open.

**Impact:** None. The endpoints are now on identity-service `dev`.

## v3.1.0 — 2026-11-16
**Author:** Vuththana

- `data-dictionary/notifications.md`: new; `notifications` (identity V1, owned by NTF) and `push_tokens` (identity V3, specs/notification-mobile §8).
- `data-dictionary/jsonb-shapes.md`: new; `users.notification_preferences` with the new per-type `push` toggle, and `notifications.params` per alert type.
- `data-dictionary/README.md`: file rows for both.
- `TRD.md` §6: the `push-token-cleanup` job (daily 02:45 Asia/Phnom_Penh, identity-service).

**Impact:** None. These document what identity-service already has on `feature/notification-mobile`.

## v3.0.1 — 2026-11-16
**Author:** Vuththana

- `specs/notification-mobile/plan.md`: Plan: notification-mobile (approved by Vuththana). It covers identity-service parts A and B and android-ui parts C, D and E.
  - **New dependencies approved:** Firebase BoM 34.x with firebase-messaging and the google-services plugin 4.4.x (android); firebase-admin 9.x (identity).
- `specs/README.md`: stage tasks.

**Impact:** None (plan only).

## v3.0.0 — 2026-11-16
**Author:** Vuththana

- `specs/notification-mobile/spec.md`: amended and re-approved (✅ v3.0.0), answering the planner's questions.
  - Withdrawn: NTM-D05 → D32, NTM-R21 → R46, NTM-M06 → M29. The Telegram note opens Settings → Telegram alerts.
  - Added:
    - D29 / R47: the debug app id is `com.kshrd.starter.dev`, the release one `com.kshrd.starter`.
    - D30 / R44: Push is disabled while In-app is off.
    - D31 / R45 / M30: the server's field message shows on a refused save.
    - D33: sign-out answers `200` as today; the push token is deleted only after an accepted logout. R43 is updated to match.
    - E38 … E41.
  - Wording: NTM-M09 code `NOTIFICATION_NOT_FOUND`; NTM-R15 "loaded cards"; references to withdrawn IDs updated.
- `api/identity-service.md`: the sign-out entry corrected to match NTM-D33.

**Impact:**
- `ledgervision-android-ui`: the debug `applicationIdSuffix = ".dev"`, the Push/In-app switch rules, field-error messages, and the Telegram note linking to Telegram alerts.
- `ledgervision-identity-service`: Keycloak logout must report acceptance; the sign-out push-token deletion depends on it.
- No plan or tasks were saved yet, so nothing goes stale.

## v2.5.0 — 2026-11-16
**Author:** Vuththana

- `specs/notification-mobile/spec.md`: clarified and approved (✅ v2.5.0).
  - NTM-Q01 … Q14 answered → NTM-D14, D16 … D28.
  - Proposals D11 … D16 and the push and card titles M10 … M22 are confirmed.
  - New: R40 … R43, M22 … M28 and E29 … E37.
- `api/identity-service.md` (new):
  - `POST /api/v1/notifications/push-tokens` and `POST /api/v1/notifications/push-tokens/unregister`, 📝 planned.
  - Changes to `GET`/`PUT /api/v1/notifications/preferences` (`push`) and `POST /api/v1/auth/sign-out` (`push_token`), 📝 planned.
- `api/README.md`: identity-service row.
- `specs/README.md`: notification-mobile ✅, stage plan.
- Data dictionary: not changed. §8 rows (`notifications`, `users.notification_preferences.push`, new `push_tokens`) stay proposed until the dictionary is brought into this repo.

**Impact:**
- `ledgervision-identity-service`:
  - new `push_tokens` table (migration) and the two token endpoints;
  - `push` in the notification preferences, with the default equal to the Telegram default;
  - optional `push_token` on sign-out;
  - an FCM sender after commit, next to Telegram delivery;
  - a 60-day stale-token job;
  - token cleanup on user deactivation, business deletion and anonymisation;
  - a Firebase service-account key in `.env`.
- `ledgervision-android-ui`:
  - notification center, settings screen and a bell in `DashboardHeader`;
  - FCM client (new dependency), with `google-services.json` git-ignored;
  - push registration, sign-out and tap handling.
- `ledgervision-web-ui`: none required. It keeps working without `push` (NTM-D28).

## v2.4.0 — 2026-11-16
**Author:** Vuththana

- `specs/notification-mobile/spec.md`: new spec "Notifications on mobile (center and push)", prefix NTM, 📝 Draft. It has 39 requirements, 28 edge cases, 16 decisions, 21 messages and 4 open questions.
- `specs/README.md`: row added.

**Impact:** None until approved.

## v0.1.0 — 2026-11-12
**Author:** Vuththana

- Docs created with `/sdd:init`: `README.md`, `TEAM-RULES.md`, `TRD.md`, `VERSION.md`, `data-dictionary/`, `api/`, `specs/` with its templates.

**Impact:** None.
