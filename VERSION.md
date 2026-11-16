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
