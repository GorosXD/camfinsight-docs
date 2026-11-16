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

## v3.1.0 — 2026-11-16
**Author:** Vuththana

- `specs/receipts/spec.md`: new spec, Receipts (📝 Draft, prefix RCT). It covers saving AI-read or typed-in receipts in finance-service, the possible-duplicate check, posting as an expense, discarding and reversing. It has 37 requirements, 18 edge cases, 16 decisions and 8 open questions.
- `specs/README.md`: added the Receipts row (stage spec).

**Impact:** None until approved.

## v0.1.0 — 2026-11-12
**Author:** Vuththana

- Docs created with `/sdd:init`: `README.md`, `TEAM-RULES.md`, `TRD.md`, `VERSION.md`, `data-dictionary/`, `api/`, `specs/` with its templates.

**Impact:** None.
