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

## v2.0.0 — 2026-11-15
**Author:** Vuththana

- `specs/mobile-sales-management/spec.md`: amended and re-approved (✅ v2.0.0).
  - Withdrawn: MSL-D17 and MSL-R41 (Sales/Profile bottom bar) → MSL-D27 and MSL-R50. The feature follows the team's app shell: the Dashboard is the start screen, and Sales opens from Menu → Sales as a pushed screen with no bottom bar.
  - Added: MSL-D28 to MSL-D30, MSL-R51 and MSL-R52, MSL-M24, MSL-E32 to MSL-E35:
    - MSL-M19 gets "Discard", confirmed by MSL-M24, which returns to the refreshed list.
    - MSL-M19 is modal and survives rotation and process death.
    - Refreshing the Dashboard is out of scope.
  - Reworded: MSL-D16, MSL-R26, MSL-R40, MSL-E02 and MSL-E15, and the MSL-M19 text (adds "If you find it recorded, choose Discard." and names its buttons). MSL-Q14 to MSL-Q22 are answered.

**Impact:**
- `ledgervision-android-ui`:
  - T13, T14 and T16 are reopened (⟳): the M19 Discard flow, the M24 confirmation, M19 modal and restored, and navigation following the dev shell.
  - Plan §3.5, §4 (bottom bar, SalesList, RecordSale and Navigation rows) and the §5 change rows for `AppDestinations.kt`, `ProfileScreen.kt` and `MainGraph.kt` must be updated.
  - New tasks are needed for MSL-R50 to R52 and MSL-E32 to E35.
- Other repos: none.

## v1.0.1 — 2026-11-12
**Author:** Vuththana

- `specs/mobile-sales-management/plan.md`: Plan: mobile-sales-management (approved by Vuththana).
- `specs/README.md`: stage tasks.

**Impact:** None (plan only).

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
