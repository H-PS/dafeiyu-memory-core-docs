# BRIEF — Agent F · Memory lifecycle, embedding space, retrieval chain

Read first:
1. `02-shared\00-PARENT-MANDATE.md`
2. `02-shared\EVIDENCE-FORMAT.md`
3. `01-baseline-corpus\design-package\01-记忆本体与数据模型.md` (MemoryUnit, Evidence, Event, tier)
4. `01-baseline-corpus\design-package\02-核心算法规范.md` (retrieval, RRF, ranking, dedup)
5. `01-baseline-corpus\design-package\contracts\state-machines\state-machines.md` (SM-1…SM-8)
6. `contracts\schemas\embedding-space.schema.json`, `retrieval-request.schema.json`, `retrieval-response.schema.json`

Own: `03-verification\retrieval\` and `04-contracts\` lifecycle documents listed below.
Evidence: `06-evidence\retrieval.jsonl`; raw: `06-evidence\raw\retrieval\`.

## What you must nail down (and what you must not

invent)

- `fts` and `vector` participate in RRF. `structured` participates only as Filter/Router/Constraint.
  Prove this with a deterministic test, not a quote: build candidates where a structured match would
  rank first if wrongly fused, and show it does not.
- RRF must be deterministic and stable: define k, weight normalisation, tie-breaking and top-K
  truncation explicitly, then test them. Implement `rrf.cjs` (pure function) + `rrf-tests.cjs`.
- Deduplication and merge rules: identical content, near-duplicate, same fact restated at different
  times (must NOT merge across time), conflicting facts (must not silently overwrite).
- `space_lifecycle` (`active`/`draining`/`retired`) is separate from `migration_state` (SM-8).
  Test that an embedding space in `draining` is not silently treated as `active` for new writes,
  and that retrieval never mixes vectors from two different embedding spaces.
- Embedding space isolation: model/version/dimension/normalisation must be part of the space
  identity; a query embedded with space A must never be compared against vectors of space B.

## Deliverables (documents)

- `04-contracts\RETRIEVAL-CONTRACT.md` — the full chain
  Query → Router → Filter → FTS → Vector → RRF → Merge → Dedup → Permission → Ranking → Token Budget → Context Assembly,
  with the invariant stated at each stage and what each stage must NOT do.
- `04-contracts\MEMORY-LIFECYCLE-MATRIX.md` — Create → Validate → Persist → Index → Retrieve →
  Update → Supersede → Expire → Archive → Delete → Rebuild → Restore. For **each** transition:
  precondition, authority (which module may perform it), state before/after, failure handling,
  consistency requirement, and which derived indexes must be updated in the same logical step.
  Also cover tier L1/L2/L3 × status ACTIVE/SUPERSEDED/ARCHIVED/DELETED explicitly, since these axes
  are orthogonal and were previously conflated.
- `04-contracts\EMBEDDING-MIGRATION-CONTRACT.md` — space lifecycle vs migration state, cutover,
  rollback, what happens to in-flight queries during a drain.

## Deliverables (code that runs)

- `03-verification\retrieval\rrf.cjs`, `rrf-tests.cjs`, `dedup-tests.cjs`,
  `lifecycle-state-machine.cjs`, `lifecycle-tests.cjs`, `retrieval-pipeline-sim.cjs`,
  `run-all.cjs`, `README.md`, plus a synthetic candidate corpus with a recorded seed and hash.

`lifecycle-state-machine.cjs` is an **isolated prototype** of the state validator, not product code:
label it as such, list what it implements and what it does not, and state its mock boundary.
Also emit an **illegal-transition matrix**: every transition the machine must reject, and a test
proving each rejection.

## Discipline

- Do not invent token-budget numbers. There is no real tokenizer for the target model here, so any
  budget figure must be labelled as a placeholder and the contract must say the threshold is
  `VALIDATION_PENDING`.
- Where the design documents genuinely disagree, record the disagreement in
  `04-contracts\RETRIEVAL-CONTRACT.md` as an open item with both readings; do not pick one silently.
- `03-verification\retrieval\TASK-RESULT.md` with all required sections.

Parent will re-run `run-all.cjs`.
