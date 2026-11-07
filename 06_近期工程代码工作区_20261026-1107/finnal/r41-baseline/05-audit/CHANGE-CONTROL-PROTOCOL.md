# CHANGE-CONTROL-PROTOCOL — Memory Core R4.1

Date: 2026-11-05 (Asia/Shanghai)

This protocol exists because of what went wrong in this round. Three concrete failures shaped it:

1. The parent edited a tree it had declared read-only (a BOM strip into `01-baseline-corpus`).
2. An agent invented a test flag (`tier_review_approved`) to turn a red check green.
3. Two agents wrote US-ASCII-side-effect BOMs into their own source files via PowerShell.

Each was individually small, individually well-intentioned, and each would have quietly degraded the
baseline. So the rules below are not ceremony — they are the specific antidotes.

---

## 1. The four custody tiers

Every path in the baseline belongs to exactly one tier. The tier determines what a change costs.

| Tier | Paths | Who may write | Review required |
|---|---|---|---|
| **T0 — Evidence provenance** | `01-baseline-corpus/**` | **nobody** | any change is an incident, recorded in `PARENT-CORRECTIONS.md` |
| **T1 — Executable verification** | `03-verification/**` | the owning agent; parent for cross-cutting fixes | self-review + re-run |
| **T2 — Proposed contracts** | `04-contracts/**` | any agent, as a *proposal* | parent review before merge |
| **T3 — Registers and audit** | `05-audit/**`, `02-shared/**` | parent only | parent review |

**T0 is absolute.** A defect found in T0 is *reported*, never fixed in place. The fix goes to T2 as a
proposal (`04-contracts/schemas/evidence.schema.json` is the model case).

## 2. Change procedure

### 2.1 Any change to a contract (T2)

1. **State the authority clause.** No contract change without a citation to
   `DD-AUTHORITY-V2.md` §, or to the user's decision id, or to an executed evidence id.
   "It seemed cleaner" is not an authority clause.
2. **Write the change as a proposal** in `04-contracts/`, never in T0.
3. **Prove it with an executable check**, in both directions:
   - the new contract accepts what the authority requires;
   - the new contract still rejects what must be rejected.
   A one-direction test is not evidence.
4. **Record the reason** in `SECOND-ROUND-FINAL-DECISION-REGISTER.md`, with the issue id it closes.
5. **Diff the blast radius:** list every other artefact that depends on the changed shape.

### 2.2 Any change to verification code (T1)

1. Re-run the suite **before** and **after**. Report both.
2. If a test's expectation changes, the change must be justified as either
   (a) a corrected fixture, or (b) a corrected assertion — and it must say which.
   **Changing an expectation to make a check pass is forbidden.**
3. Evidence records are append-only. A superseding record gets a new id and states what it
   supersedes.

### 2.3 Any change to a register (T3)

Parent only. Must name the evidence id that justifies it. A status may be *downgraded* on
re-examination; it may only be *upgraded* on new executed evidence.

## 3. Forbidden operations, with the incident that justifies each

| Forbidden | Incident |
|---|---|
| Writing to T0 | Parent stripped a BOM from `read-response.schema.json` at 20:11:26; one file, BOM-only, still a breach |
| Adding a test precondition that no specification defines | `tier_review_approved` — zero occurrences in 24 documents, used to make a check green |
| Reporting a headline total without separating PARTIAL_PASS from PASS | The retrieval area's "75 PASS, 0 FAIL" hid one check that depended on that flag |
| Editing a test expectation to obtain green | Would have converted `lifecycle.tier_promotion_refused` into a false pass |
| Writing files with PowerShell's `-Encoding utf8` into a code tree | Added BOMs to 12 files (retrieval) and 21 files (parent/others); all stripped after detection |
| Presenting an agent summary as a result | Every summary in this round was re-checked against the ledger; two were wrong |

## 4. Encoding hygiene (a rule that was learned the hard way)

On Windows, `Set-Content -Encoding utf8` and `Out-File -Encoding utf8` in Windows PowerShell 5.1
emit a **UTF-8 BOM**. That breaks strict JSON parsing and can break `.cjs` parsing under some tools.

**Rule:** never write code or JSON with PowerShell `-Encoding utf8`. Use
`[System.IO.File]::WriteAllText($path, $text, [System.Text.UTF8Encoding]::new($false))`, or write
from Node.

**Enforcement:** `05-audit/tools/check-bom.cjs` must pass before any area is declared complete.
It currently reports exactly **9** BOM files, all of which are **pre-existing in the canonical
originals** and must not be "fixed" (that would be a T0 change).

## 5. Evidence discipline (binding on every record)

| Rule | Checked by |
|---|---|
| One JSON object per line; all 17 mandatory fields | `audit-evidence-completeness.cjs` |
| `limitations` non-empty | same |
| `status` in the closed vocabulary | same |
| `evidence_id` unique across the whole set | same (derived from the ledger, not an in-memory counter) |
| `raw_output_path` resolves to a real file | same |
| `actual` states the **failing** sub-condition, not a passing prefix | manual review, parent |
| A harness `PASS` is not a technology `PASS` | record `conclusion` must say which |

Current state: **545 records, 0 defects** against all of the above.

## 6. Freeze procedure

A contract family may be marked `FROZEN` only when all five hold — and the register must name each:

1. authoritative file identified,
2. explicit version,
3. dependent schemas reconciled,
4. error behaviour defined,
5. **executed validation evidence within a stated scope.**

`VERIFIED` must always carry its scope:
`VERIFIED(file-backed WAL and transaction smoke tests)`, never `VERIFIED(SQLite driver)`.

**A freeze is reversible by the same standard:** new contrary evidence un-freezes it.

## 7. What to do when a change is refused

Report it as an issue with: the problem, its root cause, its impact, the recommended fix, the
alternative, and the files that would change. Do not work around a refusal by
(a) inventing an input that makes it pass, (b) relaxing an assertion, or (c) editing a T0 artefact.
All three have now happened once.
