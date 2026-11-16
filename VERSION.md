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

## v3.2.0 — 2026-11-16
**Author:** Vuththana

- `specs/receipts/spec.md`: clarified and approved (✅ v3.2.0). RCT-Q01..Q21 are answered as RCT-D17..D37, and proposals D14–D16 are confirmed. The spec now has 43 requirements, 28 edge cases, 37 decisions and 26 messages.
  - Decided:
    - the AI reading is stored as returned (≤256 KB)
    - the duplicate check is business-wide against receipts and posted expense transactions; a manager sees an other-branch match only as a warning
    - posting and reversal go through the finance ledger (`PERIOD_LOCKED` applies)
    - the receipt's currency must match the account's
    - USD values are null until posting, then copied from the transaction
    - lines are optional (≤100) and not cross-checked; tax is entered as printed
    - the list uses finance's existing paging
    - CSV export is left for later
  - Added: RCT-R41..R46, RCT-E19..E28, RCT-M19, RCT-M21..M27.
- `data-dictionary/finance.md`: new, with tables `receipts`, `receipt_lines` and `documents`.
- `data-dictionary/jsonb-shapes.md`: new, with `documents.extraction`. The confidence scale is to be confirmed by the AI team.
- `data-dictionary/README.md`, `api/README.md`: added rows for the new files.
- `api/finance-service.md`: new, with 10 receipt endpoints (📝 planned). Merge note: `feature/mobile-sales-management` also creates this file.
- `specs/README.md`: Receipts ✅, stage plan.

**Impact:**
- **ledgervision-finance-service:** new tables (migration V5), 10 endpoints, and new error codes `UNSUPPORTED_FILE_TYPE`, `FILE_TOO_LARGE`, `RECEIPT_INCOMPLETE`, `RECEIPT_DATE_IN_FUTURE`, `RECEIPT_NOT_EDITABLE`, `RECEIPT_POSSIBLE_DUPLICATE` and `RECEIPT_NOT_DUPLICATE`.
- **ledgervision-web-ui:** enable the hidden receipt screens against the API.
  - In the `Receipt` type, rename `category_id` → `business_category_id` and `created_by` → `created_by_user_id`.
  - Money fields are nullable before posting.
  - The upload copy adds WebP.
  - Hide Export.
- **ai-import-service:** confirm the extract response's confidence scale.
- **Android, identity-service and sales-service:** none.

## v3.1.0 — 2026-11-16
**Author:** Vuththana

- `specs/receipts/spec.md`: new spec, Receipts (📝 Draft, prefix RCT). It covers saving AI-read or typed-in receipts in finance-service, the possible-duplicate check, posting as an expense, discarding and reversing. It has 37 requirements, 18 edge cases, 16 decisions and 8 open questions.
- `specs/README.md`: added the Receipts row (stage spec).

**Impact:** None until approved.

## v0.1.0 — 2026-11-12
**Author:** Vuththana

- Docs created with `/sdd:init`: `README.md`, `TEAM-RULES.md`, `TRD.md`, `VERSION.md`, `data-dictionary/`, `api/`, `specs/` with its templates.

**Impact:** None.
