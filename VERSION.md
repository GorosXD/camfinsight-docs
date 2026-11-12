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

## v1.0.0 — 2026-11-12
**Author:** Vuththana

- `specs/mobile-sales-management/spec.md`: amended and re-approved (✅).
  - Withdrawn: MSL-R13 → MSL-R44, MSL-D20 → MSL-D24 (branch stock from `GET /items/{id}` `stock_by_branch`; `GET /items/stocks` no longer used), MSL-E10 → MSL-E28.
  - Added: MSL-D25 (connection failures before sending, and on reverse → MSL-M08 + Retry), MSL-D26 (no minus sign; KHR whole riel), MSL-R45–R49, MSL-M21–M23, MSL-E29–E31.
- `specs/README.md`: mobile-sales-management ✅ again.

**Impact:**
- `ledgervision-android-ui`: not built yet; the plan (not yet saved) must use the amended IDs.
- `ledgervision-identity-service`: none beyond v0.3.0 (exchange-rate endpoint).
- Other repos: none.

## v0.3.0 — 2026-11-12
**Author:** Vuththana

- `specs/mobile-sales-management/spec.md`: clarified (MSL-Q01–Q13 → MSL-D12–D23, new MSL-R39–R43, MSL-M18–M20, MSL-E20–E27) and approved (✅).
- `api/identity-service.md`: new file; `GET /api/v1/businesses/me/exchange-rate` 📝 planned (owner and manager, MSL-D15).
- `api/README.md`: identity-service added to the services table.
- `specs/README.md`: mobile-sales-management ✅, stage plan.

**Impact:**
- `ledgervision-identity-service`: add `GET /api/v1/businesses/me/exchange-rate` (owner and manager).
- `ledgervision-android-ui`: build the feature (bottom navigation, sales list, record, detail, reverse).
- `ledgervision-sales-service`, `ledgervision-finance-service`: none (existing endpoints are used as they are).
- `ledgervision-web-ui`: none.

## v0.2.0 — 2026-11-12
**Author:** Vuththana

- `specs/mobile-sales-management/spec.md`: new spec (📝 Draft) — walk-in sales on Android: day KPIs, list, record, view, reverse.
- `specs/README.md`: added the feature row.

**Impact:** None until approved.

## v0.1.0 — 2026-11-12
**Author:** Vuththana

- Docs created with `/sdd:init`: `README.md`, `TEAM-RULES.md`, `TRD.md`, `VERSION.md`, `data-dictionary/`, `api/`, `specs/` with its templates.

**Impact:** None.
