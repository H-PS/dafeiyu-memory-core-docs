# PARENT MANDATE — Memory Core R4.1 Round-4.1 Execution Baseline

Date: 2026-11-05 (Asia/Shanghai)
Parent workspace: `E:\[PROJECT]\r41-baseline`
Canonical package (READ-ONLY to this session): `C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design`
Writable snapshot of that package: `E:\[PROJECT]\r41-baseline\01-baseline-corpus\design-package`

## 0. Why a new workspace exists

The canonical design package sits under a path the current DSH file sandbox does not permit
writing (`Access to the path ... is denied`). The session workspace `E:\[PROJECT]` is writable.
Therefore this baseline is built as a **parallel, writable working tree** that:

1. contains a byte-identical snapshot of the canonical package (evidence provenance), and
2. holds all new verification code, contracts and reports.

No file under `C:\Users\[USER]\Desktop\work\` is modified by this session. Any diff between the
snapshot and the canonical package is therefore a deliberate, reviewable proposed revision.

## 1. Authorization axes — keep them separate (SRD-001)

Never collapse these into one status word.

| Axis | Value at start of this task |
|---|---|
| `CODE_IMPLEMENTATION_SCOPE` | Isolated verification code, harnesses, benchmarks, validators, minimal isolated prototypes under `E:\[PROJECT]\r41-baseline\03-verification\` |
| `PRODUCT_IMPLEMENTATION_AUTHORIZATION` | **NOT_AUTHORIZED** — no M01/M02/M03/M04/M05 product implementation, no real user memory, no real QQ Bridge / DSH identity data, no real permission writes, no Emotion Runtime |

Writing runnable isolated code is explicitly permitted. It does **not** upgrade the product
authorization. A subagent may never write to a product module path.

## 2. Standing prohibitions

- Do not connect real user memory, real QQ Bridge / DSH identity data, real permission state.
- Do not enable Emotion Runtime; `emotion.*` flags stay default-off.
- Do not mark an isolated prototype as a production implementation.
- Do not bypass a real permission check through test code.
- Do not describe synthetic-identity results as real-host verification.
- Do not write `FROZEN` for anything not verified within a stated scope.
- Do not invent thresholds, dataset results, hashes, timings or exit codes.

## 3. Environment facts established by the parent (do not re-derive, verify if you rely on them)

| Fact | Value | How established |
|---|---|---|
| Node.js | v24.21.0 | `node --version` |
| Built-in SQLite | `node:sqlite` available; `DatabaseSync`, `StatementSync`, `Session`, `constants`, `backup` | `require('node:sqlite')` |
| SQLite version | 3.53.4 | `select sqlite_version()` |
| Compile options | `ENABLE_FTS3`, `ENABLE_FTS3_PARENTHESIS`, `ENABLE_FTS5` | `pragma compile_options` |
| OS | Windows NT 10.0.26200.0, AMD64 | `.NET OSVersion` |
| **npm / pnpm / npx / corepack** | **NOT INSTALLED — none resolvable on PATH** | `Get-Command` sweep + full-disk `npm-cli.js` search |
| LanceDB | `@lancedb/lancedb` **0.41.0** present at `C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design\evidence\second-round\node_modules` | `require` of package.json |
| LanceDB `:memory:` | **NOT SUPPORTED** — throws `Unable to created lance dataset at :memory:` (os error 123) | runtime probe |
| Python | 3.14.7 at `C:\Users\[USER]\AppData\Local\Python\pythoncore-3.14-64\python.exe` | `python --version` |
| jsonschema | 4.26.0 importable via `PYTHONPATH` = the package's `evidence/second-round/pydeps` | `import jsonschema` |

**Consequence for Task A:** a third-party SQLite driver (better-sqlite3, etc.) **cannot be
installed**. The driver comparison is therefore bounded: `node:sqlite` is the only installable
candidate in this environment. Report that as an environment constraint with a concrete
"what would be needed to lift it" note. Do **not** simulate a comparison that did not happen.

**Consequence for Task B:** every LanceDB test must use a **file-backed temporary directory**
(`os.tmpdir()` or a scratch dir), never `:memory:`.

## 4. Baseline conflicts that already exist (carry them; do not silently resolve)

These were found by the parent before dispatch. Subagents must not contradict them silently.

| ID | Conflict | Owner |
|---|---|---|
| B-1 | `contracts/schemas/evidence.schema.json` is v1-stale: no `original_text`, no `speaker_id`, `source_type` is a 12-value v1 enum, `additionalProperties:false` — while `DD-AUTHORITY-V2.md §3.2` requires `original_text` and `speaker_id` and the 5-value enum. Per the schema, a spec-compliant Evidence object is **invalid**. | Agent E (load-bearing), Agent D (consumer) |
| B-2 | Every `evidence/second-round/*.json` result file records `"node": "v24.22.1"` but this machine runs **v24.21.0**. Provenance of those results cannot be reproduced here. | Agent A (report it), parent |
| B-3 | "Full Audit Report says file-backed WAL/migration/backup/lock/crash PASS; Technical Verification Report says only in-memory smoke was done." Both exist in the same package. | Agent A |
| B-4 | Two competing R4.1 document sets: `Memory-Core-R4.1-Design` (00–15 + contracts) vs `R4.1-UNIFIED-MEMORY-CORE` (00–16 + audit). No recorded precedence rule. | Agent H |
| B-5 | No `npm`/`pnpm` — dependency-level findings in prior reports are not re-derivable. | Agent A/B |
| B-6 | LanceDB `:memory:` unsupported — any prior LanceDB test claiming `:memory:` usage is suspect. | Agent B |

## 5. Isolation rules

- All new code lives under `E:\[PROJECT]\r41-baseline\03-verification\<area>\` or
  `04-contracts\`, never inside `01-baseline-corpus\` (that tree is evidence provenance only;
  it may be read, and copied into a working area, but not edited in place).
  Exception: Agent E may propose revised contract files under `04-contracts\` — see its brief.
- Every test DB is a throwaway file under the area's `_scratch\` directory and is deleted or
  clearly disposable. Never point at a real memory database.
- Every script has: explicit entry point, deterministic/reproducible input, explicit expected
  result, explicit exit code (0 = all checks ran and passed; 1 = at least one check failed;
  2 = harness error / precondition unmet), raw result written to `06-evidence\raw\`.

## 6. Evidence discipline

Every executed test emits one evidence record per the schema in
`E:\[PROJECT]\r41-baseline\02-shared\EVIDENCE-FORMAT.md`, appended to
`E:\[PROJECT]\r41-baseline\06-evidence\<area>.jsonl`, with raw stdout/stderr saved under
`E:\[PROJECT]\r41-baseline\06-evidence\raw\`.

A test that was not run gets **no** evidence record with `PASS`. Absence of an evidence record
means NOT_RUN. Never write a PASS record for a test that did not execute in this session on
this machine.

## 7. Reporting contract

Each subagent writes `TASK-RESULT.md` in its own area directory using the mandatory section
list in `02-shared\TASK-ASSIGNMENT.md`. Conclusions must be traceable to a real file path and a
real evidence id. The parent will re-run at least one script from each area and will reject
results that do not reproduce.
