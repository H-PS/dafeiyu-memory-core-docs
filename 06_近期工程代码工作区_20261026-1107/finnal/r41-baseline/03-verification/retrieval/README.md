# 03-verification/retrieval — Agent F area README

Verification area for **M02 retrieval + the lifecycle/binding rules it depends on**, Memory Core R4.1.

- Contracts (design-facing): `04-contracts/RETRIEVAL-CONTRACT.md`,
  `MEMORY-LIFECYCLE-MATRIX.md`, `EMBEDDING-MIGRATION-CONTRACT.md`
- Result summary: `TASK-RESULT.md` (this directory)
- Evidence ledger: `06-evidence/retrieval.jsonl` (one JSON line per executed check)
- Raw logs / artifacts: `06-evidence/raw/retrieval/`

## Run it

```powershell
cd E:\[PROJECT]\r41-baseline\03-verification\retrieval
node run-all.cjs          # every suite: 74 PASS, 1 PARTIAL_PASS, 0 FAIL (exit 0)
```

Individual suites (each is self-contained and exits 0/1):

```powershell
node rrf-tests.cjs
node dedup-tests.cjs
node lifecycle-tests.cjs
node retrieval-pipeline-sim.cjs     # runs the chain AND its 20 checks
```

Requirements: `node` only (built and run on Node v24.21.0, Windows x64). **No npm, no pnpm, no
dependencies** — plain CommonJS `.cjs`, only `node:crypto`, `node:fs`, `node:path`.

## Layout

```
retrieval/
  _lib/
    ev.cjs                    evidence recorder: schema enforcement, hashing, ids EV-RETRIEVAL-NNN
    harness.cjs               suite/check harness; a thrown body is FAIL, never PASS
    embed.cjs                 deterministic SHA-256 "embedding" stand-in + cosine  (NOT semantic)
  _data/
    generate-corpus.cjs       seeded generator (mulberry32, seed 20261105) -> datasets
    candidate-corpus.json     24 synthetic memories across 2 embedding spaces
    dedup-corpus.json         11 literal fact pairs for the dedup classes
    corpus.meta.json          seed + sha256 + byte counts of both datasets
  rrf.cjs  rrf-tests.cjs                      RRF fusion (pure function) + 15 checks
  dedup.cjs  dedup-tests.cjs                  dedup/merge decision (pure function) + 15 checks
  lifecycle-state-machine.cjs  lifecycle-tests.cjs   isolated state-machine prototype + 24 checks
  retrieval-pipeline-sim.cjs                  12-stage chain simulation + 20 checks
  run-all.cjs                                 canonical entry point
```

## Non-negotiables this area was built under

| Rule | How it is honoured |
|---|---|
| `01-baseline-corpus/` is immutable | nothing under it was written; every design statement is a **quoted citation**, never a copy edited in place |
| Contracts in `04-contracts/`, code in `03-verification/retrieval/` | no document contains executable test code, no code file contains a contract |
| `node` directly, CommonJS | no package manager, no ESM, no build step |
| No invented numbers | every threshold that is not measured is labelled `[VALIDATION_PENDING]` or `[PROPOSED]`; the corpus seed and hashes are recorded, not estimated |
| Every test really runs | `run-all.cjs` exits 1 on any failure and 2 on a harness error; a check body that throws is recorded as `FAIL` |
| No product implementation | the state machine carries an explicit `ISOLATED PROTOTYPE` header with a stated mock boundary; nothing here is imported by product code |

## Evidence discipline

- **`PARTIAL_PASS` is used, not hidden.** A check whose assertion holds only because the test supplied an input that no design document defines is recorded as `PARTIAL_PASS` and listed separately in the run headline (`lifecycle.tier_promotion_gate_is_explicit`). It is never counted as a clean pass.
- One JSON line per **executed** check, mandatory `expected` / `actual` / `status` / `limitations` /
  measured `duration_ms` / `exit_code`. A check that did not run gets **no** line.
- Status vocabulary: `PASS`, `PARTIAL_PASS`, `FAIL`, `NOT_RUN`, `NOT_VERIFIED`, `BLOCKED`,
  `NOT_APPLICABLE`.
- `run-all.cjs` **archives** the previous ledger into `06-evidence/raw/retrieval/archive/` and then
  writes a fresh one, so the canonical file always describes exactly one complete run while older
  runs remain on disk. Raw per-suite logs are written next to it (`<suite>.run-all.log`).
- `corpus.generator_deterministic` re-runs the generator and asserts the two dataset hashes do not
  change.

## Datasets

| File | sha256 | Bytes | Contents |
|---|---|---|---|
| `candidate-corpus.json` | `79a125b3dab5054957fbc3fd2b18492d59fd5a71cda8ec185091ec7b60f14b39` | 26770 | 24 memories, 2 spaces, 8-dim vectors |
| `dedup-corpus.json` | `e81c38bca6734bcadcce574c71d6d0dd3eac528e1cd9a21c9a95f07091fa0ee6` | 9780 | 11 labelled fact pairs |

Seed `20261105`. The corpus is **synthetic and meaningless as content**: it exists to make every rule
falsifiable (an identical text in two spaces, an exact duplicate at two timestamps, a punctuation
variant that is a near-duplicate, a deleted row whose index would still match, etc.). It is not a
sample of real memory data and must never be used as one.

Both space ids are **derived**, not hand-written:
`"space-" + sha256(canonical_json(eight fields))[0:16]` → `space-1ba906b29154f6f2` (active) and
`space-3f57100b18debe1e` (draining). Regenerating with different fields yields different ids, which is
the whole point.

## Environment facts discovered while building this

- Piped stdio for a spawned program fails in this sandbox
  (`child_process.spawn` with the default `stdio:'pipe'` → `EPERM`). Consequence: `run-all.cjs`
  `require`s every suite **in-process** instead of spawning `node <suite>`; the only spawn uses
  `stdio:'ignore'`.
- `$LASTEXITCODE` is not reliable when a Node run is piped into `Select-Object`; exit codes were
  captured with `node x.cjs > out 2>&1; "EXIT=$LASTEXITCODE"`.
- `01-baseline-corpus/design-package/02-核心算法规范.md` is reported as binary by the file reader;
  it was read through `Get-Content -Encoding UTF8` instead. Anyone re-reading the design package
  should expect this.

## Open items

All are recorded with **both readings** in the contracts (§8 of `RETRIEVAL-CONTRACT.md`, §7 of
`MEMORY-LIFECYCLE-MATRIX.md`, §9 of `EMBEDDING-MIGRATION-CONTRACT.md`). None was resolved by
preference.

`OI-R-01` `pending_review → deleted` exists only in SM-1 · `OI-R-02` `archived → active` restore ·
`OI-R-03` `rejected → unverified` revival · `OI-R-04` cross-scope merge · `OI-R-05`
`suspended → available` · `OI-R-06` SM-1 mixes validation into lifecycle · `OI-R-07` meaning of
`draining` · `OI-R-08` does ALG-04's N feature count as "structured scoring" · `OI-R-09` budget
truncation skip-vs-stop · `OI-R-10` 02 §7.4's own example has duplicate channel ranks · `OI-R-11`
02 §7.4 still fuses `structured` · `OI-R-12` request field names · `OI-R-13` `space_lifecycle` vs
`status` · `OI-R-14` degradation marker field · `OI-R-15` writes during `cutover`/`rollback` ·
`OI-R-16` `migration_state` enum set.
