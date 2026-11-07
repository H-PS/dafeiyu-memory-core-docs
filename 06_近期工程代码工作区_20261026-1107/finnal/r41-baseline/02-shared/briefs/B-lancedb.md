# BRIEF — Agent B · TECH-LANCEDB-01 · LanceDB exactness vs an independent brute-force reference

Read first:
1. `E:\[PROJECT]\r41-baseline\02-shared\00-PARENT-MANDATE.md`
2. `E:\[PROJECT]\r41-baseline\02-shared\EVIDENCE-FORMAT.md`
3. `E:\[PROJECT]\r41-baseline\01-baseline-corpus\design-package\02-核心算法规范.md` (retrieval / vector chapter)
4. `C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design\evidence\second-round\lancedb-result.json`
   and `lancedb-harness.cjs` — the prior claim you must audit

Own this area only: `E:\[PROJECT]\r41-baseline\03-verification\lancedb\`
Evidence: `06-evidence\lancedb.jsonl`; raw logs: `06-evidence\raw\lancedb\`.

## Environment facts (parent-verified, rely on them)

- SDK: `@lancedb/lancedb` **0.41.0**, resolved via
  `C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design\evidence\second-round\node_modules`.
  There is **no npm/pnpm**, so you cannot install anything. Require it by absolute path:
  `require(path.join(NODE_MODULES, '@lancedb','lancedb'))`, with `NODE_MODULES` configurable by env var.
- **`:memory:` is NOT supported** by this SDK build: `connect(":memory:")` throws
  `Unable to created lance dataset at :memory: (os error 123)`. Use a real temp directory.
- Probe result the parent already confirmed (reproduce it, don't trust it): with vectors a=[1,0,0],
  b=[0.6,0.8,0], c=[0,1,0], query [1,0,0], SDK returns `_distance` = 0, 0.800000011920929, 2.0 —
  i.e. **`_distance` is squared L2 on raw vectors**. Confirm or refute this yourself and design your
  reference around what you actually observe.

## The audit you must perform

The prior report claims: 0.41.0, dataset hash `155b2634…`, max cosine-distance error 5.6926e-9,
tolerance 1e-5, and that topk/order/distance/filter/empty/bad-dimension/update/delete/reopen all
passed. Determine from the artefact itself:

- Was the dataset actually fixed and hashed? Can you regenerate the same hash?
- Was the brute-force reference genuinely independent of the SDK, or did it share code/semantics?
- Does the reported error even have the right shape? `_distance` = squared L2, while the reported
  comparison is a cosine *distance* of `1 - cos`. Check whether the harness compared quantities that
  are actually comparable, and report if it did not.
- Does the vector set contain ties (a and e both scoring 1) that the harness handled implicitly?

This is the highest-value part of your task. A prior claim that turns out to be arithmetically
hollow is a finding the parent must report.

## Reference implementation

Write `brute-force-reference.cjs`: pure JS, no SDK import, cosine similarity computed directly from
Float32Array inputs. Explicitly document and pin:

- dimension, normalisation rule, similarity definition, distance definition, sort direction,
- tie-breaking rule (define one, e.g. ascending id, and use it on both sides),
- top-K rule, float precision claim, and the origin of any tolerance you adopt.

Justify the tolerance from float32 representability for your magnitudes; do not copy 1e-5 blindly.

## Dataset

`dataset-generator.cjs` producing a **stored, hashable** dataset (save the JSON, record sha256 and
the seed). Cover: small, medium, random, near-duplicate/near-similar, exact duplicate vectors,
zero vector, invalid dimension, invalid values (NaN/Inf), boundary dimension, ties.
Save generated files under `03-verification\lancedb\_data\` and record each hash.

## Must test

top-K membership; full ordering; distance error distribution (report max and a histogram, not just
one number); tie-breaking; empty dataset; invalid dimension; invalid vector; filter; update; delete;
reopen; persistence across a fresh `connect()`; abnormal storage state; duplicate records;
retrieval consistency after mutation.

## Status discipline you must respect

Keep these four separate and report each on its own line:
- SDK availability
- basic retrieval works
- partial exactness passed
- complete exactness passed (and, separately) engineering reliability.

Do not let one imply another. If FLAT exactness holds only at small scale, say so and state the
scale bound.

## Deliverable

`03-verification\lancedb\TASK-RESULT.md`, plus `run-all.cjs` and a `README.md`.
Include: whether the prior claim is reproducible, the observed distance semantics, error
distribution, uncovered scenarios, recommended usage boundary, whether this contract may freeze,
and every limitation. Parent will re-run `run-all.cjs`.
