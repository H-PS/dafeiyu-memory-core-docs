# r41-baseline — index and reproduction guide

This directory is the **round 4.1 working baseline** for Memory Core R4.1: a writable tree containing a
byte-exact snapshot of the canonical design package, plus all the verification code, contracts and
evidence produced in this round.

Start here: **`07-reports/MEMORY-CORE-R4.1-FINAL-TECHNICAL-AND-IMPLEMENTATION-REPORT.md`**.

---

## Layout

| Path | What it is | Mutability |
|---|---|---|
| `01-baseline-corpus/` | snapshot of the canonical design package + legacy governance docs + the second R4.1 document set | **read-only** (see §Custody) |
| `02-shared/` | parent mandate, evidence format, subagent briefs | parent only |
| `03-verification/` | all executable verification code, six areas | owning area |
| `04-contracts/` | proposed contracts and the parent's consistency reports | proposals; parent reviews |
| `05-audit/` | registers, matrices, corrections, gates | parent only |
| `06-evidence/` | evidence ledgers (`.jsonl`) + raw run logs | append-only |
| `07-reports/` | the final report | parent only |

## Reproduce everything

```powershell
cd E:\[PROJECT]\r41-baseline

# each area runs standalone
node 03-verification\sqlite\run-all.cjs
node 03-verification\lancedb\run-all.cjs
node 03-verification\fts\run-all.cjs
node 03-verification\identity\run-all.cjs
node 03-verification\retrieval\run-all.cjs
node 03-verification\outbox\run-all.cjs
node 03-verification\schema\validate-schemas.cjs
node 03-verification\schema\outbox-schema-conformance.cjs

# integrity gates
node 05-audit\tools\audit-evidence-completeness.cjs
node 05-audit\tools\check-bom.cjs
node 05-audit\tools\STAGE-G-TRUTH-AUDIT.cjs
```

**Exit codes** are consistent across areas: `0` all green · `1` not fully green (includes
`PARTIAL_PASS`) · `2` harness error / precondition unmet. **A non-zero exit is usually the honest
verdict, not a crash** — several suites exit 1 because a product criterion genuinely fails.

## Requirements

- **Node.js with builtin `node:sqlite`** (verified on v24.21.0, SQLite 3.53.4). No npm packages, no build step.
- **No package manager is needed, and none exists on this machine** — every script is plain CommonJS.
- LanceDB tests additionally need `@lancedb/lancedb` 0.41.0; the harness resolves it by absolute path
  from the canonical package's installed `node_modules` (override with `$env:LANCE_NM`).
  `connect(":memory:")` does **not** work in this build — a real temp directory is required.
- Python 3.14.7 + jsonschema 4.26.0 is used for JSON-Schema validation, via
  `$env:PYTHONPATH='C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design\evidence\second-round\pydeps'`.

## Custody (§ important)

`01-baseline-corpus/` is **evidence provenance and must not be edited.** A defect found there is
*reported* and fixed as a proposal under `04-contracts/`.

One deviation exists and is recorded in `05-audit/PARENT-CORRECTIONS.md §1`: the parent stripped a
UTF-8 BOM from `contracts/schemas/read-response.schema.json` at 20:11:26 on 2026-11-05. Measured
blast radius: exactly that one file, BOM only; canonical originals remain intact.

Nine files in the corpus carry pre-existing UTF-8 BOMs **inherited from the canonical package**.
They are left alone deliberately; "fixing" them would be a further custody breach.

## Evidence discipline

Every executed check writes one JSON record to `06-evidence/<area>.jsonl` per
`02-shared/EVIDENCE-FORMAT.md`. Current state: **549 records, 0 missing fields, 0 duplicate ids,
0 unresolvable raw paths**, verified by `05-audit/tools/audit-evidence-completeness.cjs`.

A test that was not run has **no** PASS record. Absence of a record means `NOT_RUN`.

## Reading the results honestly

| Term | Meaning |
|---|---|
`PASS` | executed on this machine and met its stated expectation |
`PARTIAL_PASS` | executed; some sub-conditions met, stated explicitly — **not a clean pass** |
`FAIL` | executed; expectation not met. Over half of the 21 FAILs are *findings*, not harness breakage |
`BLOCKED` | a precondition is unmet; the reason is stated |

Headline totals in any area report count these separately on purpose. A "75 PASS, 0 FAIL" that hides
one PARTIAL is a misleading headline — that mistake happened once in this round and was corrected.

## What this is not

This is **not** product code. `PRODUCT_IMPLEMENTATION_AUTHORIZATION = NOT_AUTHORIZED`. No M01–M05
module exists. Each verification area states its mock boundary in its own `TASK-RESULT.md`, and none
of that code should be lifted into a product module without re-examination.
