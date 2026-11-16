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

## v5.0.0 — 2026-11-16
**Author:** Vuththana

- `specs/receipts/spec.md`: amended and re-approved (✅ v5.0.0) to settle the spec gaps from `/sdd:verify` on the finance build. The spec now has 52 requirements, 32 edge cases, 69 decisions and 30 messages. RCT-Q31..Q39 are answered as RCT-D61..D69, and RCT-D46..D60 were decided directly.
  - Withdrawn:
    - RCT-R15 → RCT-R56: a save that changes nothing writes no history row; a status or match change from the re-check counts as a change.
    - RCT-R34 → RCT-R55: a `REVERSED` receipt's detail also returns `transaction_id` and the transaction reference.
    - RCT-R39 → RCT-R54: reversing a receipt that isn't posted gets RCT-M30 "Only a posted receipt can be reversed." (new `409 RECEIPT_NOT_REVERSIBLE`).
    - RCT-E13 → RCT-E32: the same refusal for an already reversed receipt.
  - Added:
    - RCT-R57: a reading sent without a file is not stored.
    - RCT-R58: reversal transactions are never duplicate candidates.
    - RCT-E31, RCT-E33: the summary answers without the rate; USD totals that needed it are null.
    - RCT-E34: reversing still goes ahead with a turned-off account or category.
    - RCT-M29 "Marked as not a duplicate" and RCT-M30.
  - Reworded:
    - Messages: RCT-M04 (one message with a count), RCT-M09 (lowercase status), RCT-M13 and RCT-M21 (problem codes), RCT-M15 (also the edit), and the `{amount}` format ("1,250.50 USD", "41,000 KHR"). The RCT-M03 problem codes are listed in §6.
    - Requirements: R10, R25, R32, R33 and R38.
    - Edge cases: E02, E03 and E14.
  - §8: a `document_sequences` table row (proposed); `documents.extraction` is marked 📝 until the AI team confirms the confidence scale.
- `specs/receipts/tasks.md`: T07–T12, T14, T17, T20, T21 and T24–T26 are marked ⟳; finance T07–T12 and T14 are reopened.

**Impact:**
- finance-service:
  - Add `RECEIPT_NOT_REVERSIBLE` and the reverse refusal order (RCT-D61).
  - Skip the `UPDATED` row on a no-change save.
  - Summary: read the rate only for unposted non-USD receipts; null totals when it's missing.
  - Reversed detail: return the reference.
  - Messages: M04 singular, M09 lowercase, M18 amount format, M29.
  - Tests: rename those that name RCT-R15, R34, R39 and E13 (`ReceiptReviewIT`, `ReceiptDetailIT`, `ReceiptDiscardReverseIT`).
- web-ui:
  - Texts for M29, M30, the M04 count and the problem codes.
  - The reversed view's reference.
  - Null summary totals shown as "—".
  - `branch_name` from the API.
  - The reverse dialog for M30.
  - `e2e/contracts/api-responses.spec.ts` names withdrawn IDs.
- Plan:
  - §2 parts C and D;
  - §3 detail, PUT, summary, not-duplicate and reverse;
  - the §4 reverse dialog;
  - the §7 `ErrorCode` table;
  - the §8 edge-case table.

## v4.0.1 — 2026-11-16
**Author:** Vuththana

- `specs/receipts/plan.md`: Plan: receipts (approved by Vuththana). It covers finance-service parts A–E (the receipts backend and the receipt guard in the transactions endpoints) and web-ui parts F–H. Both repos branch from `dev-cleaned`, with PRs into `dev-cleaned`.
- `specs/README.md`: stage tasks.

**Impact:** None (plan only).

## v4.0.0 — 2026-11-16
**Author:** Vuththana

- `specs/receipts/spec.md`: amended and re-approved (✅ v4.0.0) to answer the planner's questions. RCT-Q22..Q30 are answered as RCT-D38..D45. The spec now has 50 requirements, 29 edge cases, 45 decisions and 27 messages.
  - Withdrawn: RCT-E20 → RCT-E29. An action that loses a race to a discard now gets 404 RCT-M08, not RCT-M09.
  - Added:
    - RCT-R47, R48, R52, R53: a receipt's transaction and its reversal row can't be reversed, PATCHed or bulk-categorized from the transactions endpoints (409 `TRANSACTION_NOT_REVERSIBLE`, RCT-M28).
    - RCT-R49: which duplicate match is kept (a manager's own branch first).
    - RCT-R50: a number search matches the total exactly.
    - RCT-R51: a reversal is dated today.
    - RCT-E29, RCT-E30, RCT-M28.
  - Reworded:
    - RCT-M06 is now "'{account}' is inactive".
    - RCT-M17 has a no-vendor text.
    - RCT-R38 says the reversal is dated today.
- `api/finance-service.md`: the receipt reversal is dated today. Three existing endpoints are marked as changed: `POST /bank-statements/transactions/{id}/reverse`, `PATCH /bank-statements/transactions/{id}` and `POST /bank-statements/transactions/categorize`.
- `specs/README.md`: Receipts ✅, stage plan.

**Impact:**
- **ledgervision-finance-service:** the receipts build follows v4.0.0. The transactions reverse, PATCH and categorize endpoints change behaviour for receipt transactions. No code or tests exist yet, so nothing is stale.
- **ledgervision-web-ui:**
  - show RCT-M28 on the transactions screens;
  - use the new M06 and M17 texts;
  - a lost discard race shows "not found".
- **Others:** none.

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
