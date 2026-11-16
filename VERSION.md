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
