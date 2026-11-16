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

## v5.1.0 — 2026-11-16
**Author:** Vuththana

- `api/finance-service.md`: the receipts endpoints as built (🚧 on `feature/receipts`): the 10 receipt endpoints with the receipt and detail objects, the RCT-M03 problem codes, the error tables and their order, the `RECEIPT_NOT_REVERSIBLE` refusal, and the summary's null USD total with no 503. Also the receipt guard on the transactions reverse / PATCH / categorize endpoints, and the existing `GET /bank-statements/transactions/{id}` that the receipt duplicate panel uses.
- `specs/receipts/tasks.md`: T29 done.

**Impact:** web-ui and finance-service already follow these entries (receipts, `feature/receipts`); no other repo.

## v5.0.1 — 2026-11-16
**Author:** Vuththana

- `specs/receipts/plan.md`: Plan: receipts, updated for spec v5.0.0 (approved by Vuththana). The changes are the reverse refusal order and `RECEIPT_NOT_REVERSIBLE`, the PUT check order and the no-change rule, summary totals that are null without a rate, the reference on a reversed receipt, and the §7 file list brought in line with the working trees.
- `specs/README.md`: stage tasks.

**Impact:** None (plan only).

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

## v2.3.0 — 2026-11-16
**Author:** Vuththana

- `api/sales-service.md` (new): sales-service endpoints used by mobile-sales-management (plan §3.2–§3.8), 📝 "exists in code".
- `api/finance-service.md` (new): `GET /api/v1/bank-statements/accounts` (plan §3.9), 📝 "exists in code".
- `api/identity-service.md`: `GET /api/v1/businesses/me/exchange-rate` (plan §3.1, new). `GET /api/v1/branches` rows corrected (`403 ONBOARDING_INCOMPLETE`, `403 BUSINESS_DELETED`, `404` only for a missing row).
- `api/README.md`: sales-service and finance-service rows added.

**Impact:**
- `ledgervision-identity-service`: new endpoint `GET /api/v1/businesses/me/exchange-rate` (T01).
- `ledgervision-android-ui`: consumes these endpoints (mobile-sales-management).
- Other repos: none (the sales and finance entries describe existing code).

## v2.2.1 — 2026-11-15
**Author:** Vuththana

- `specs/mobile-sales-management/plan.md`: Plan: mobile-sales-management, updated for spec v2.2.0 (approved by Vuththana). It covers MSL-R53, MSL-R54 and MSL-E38 … E40 in §1, §3.5, §4, §7, §8 and §9. The changes are a three-state `sale_send_state`, `canRetry` on the not-sent banner, and Back blocked while saving.
- `specs/README.md`: stage tasks.

**Impact:** None (plan only).

## v2.2.0 — 2026-11-15
**Author:** Vuththana

- `specs/mobile-sales-management/spec.md`: amended and re-approved (✅ v2.2.0).
  - Added MSL-D33, MSL-R53, MSL-E38 and MSL-E39: after MSL-M19, the sale stays "may already be recorded" until a `201` or a confirmed Discard. A "Try again" that isn't sent or is refused shows its message, without its own Retry, and MSL-M19 again.
  - Added MSL-D34, MSL-R54 and MSL-E40: Back and the back arrow don't leave the form while `POST /sales` waits.
  - MSL-Q25 and MSL-Q26 answered.
  - Clarified, no change of meaning: MSL-R27, MSL-R45, MSL-D25 and MSL-E29 apply to the first send.

**Impact:**
- `ledgervision-android-ui`:
  - T13 and T14 are reopened (⟳). After MSL-M19, only a `201` or a Discard clears `sale_unconfirmed_send`. A refused or unsent "Try again" shows MSL-M19 again without the banner Retry. Back is blocked while saving.
  - The test `MSL-E36 a recorded, refused or not-sent answer clears the marker` must be rewritten.
  - Plan §3.5, §4, §7, §8 and §9 must be updated.
  - New tasks are needed for MSL-R53, MSL-R54 and MSL-E38 … E40.
- Other repos: none.

## v2.1.1 — 2026-11-15
**Author:** Vuththana

- `specs/mobile-sales-management/plan.md`: Plan: mobile-sales-management, updated for spec v2.1.0 (approved by Vuththana). §1, §3.5, §4, §7, §8, §9 and §10 cover MSL-R50 … R52 and MSL-E32 … E37, and the tab-bar leftovers are removed.
- `specs/README.md`: stage tasks.

**Impact:** None (plan only).

## v2.1.0 — 2026-11-15
**Author:** Vuththana

- `specs/mobile-sales-management/spec.md`: amended and re-approved (✅ v2.1.0).
  - Added MSL-D31 and MSL-E36: a `POST /sales` still waiting for its answer when the app is killed counts as a lost answer, so the form comes back with MSL-M19.
  - Added MSL-D32 and MSL-E37: Back or a tap outside MSL-M24 acts as "Cancel".
  - MSL-Q23 and MSL-Q24 answered.

**Impact:**
- `ledgervision-android-ui`: the plan in progress for v2.0.0 must include MSL-E36 (save an in-flight marker before sending) and MSL-E37, under T13 and T14 (⟳).
- Other repos: none.

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
