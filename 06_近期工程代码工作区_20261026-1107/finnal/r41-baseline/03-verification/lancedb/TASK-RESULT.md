# TASK-RESULT — TECH-LANCEDB-01

Agent B · LanceDB 0.41.0 exactness against an independent brute-force reference,
plus an audit of the second-round claim in
`C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design\evidence\second-round\lancedb-result.json`.

Date: 2026-11-05 (Asia/Shanghai) · Node v24.21.0 · `@lancedb/lancedb` 0.41.0 ·
evidence `06-evidence/lancedb.jsonl` (64 records) · raw logs `06-evidence/raw/lancedb/`.

Everything below is traceable to a record id and a file path. Nothing was inferred from
documentation alone; where the SDK docs and the observed behaviour disagree, the
observation wins and is marked as such.

---

## 1. Status (five separate axes — do not collapse them)

```
SDK availability            : AVAILABLE — 0.41.0 loads by absolute path, exactly one copy, cannot be re-resolved (EV-LANCEDB-SEM-001)
Basic retrieval works       : YES — create/insert/query/update/delete/reopen/query all function (EV-LANCEDB-SEM-003…010, EV-LANCEDB-CONF-001, EV-LANCEDB-LIFE-001…005)
Partial exactness passed    : YES — top-K membership and complete ordering equal the independent reference at n=5000/d=64 and n=2000/d=384 (EV-LANCEDB-CONF-001/002, EV-LANCEDB-SCALE-001/002)
Complete exactness passed   : NO — 0 of 1000 distances are bit-identical; cosine distance leaves [0,2] by 1.19e-7; ±Infinity rows are accepted and then unrankable under cosine; ties have no defined order (EV-LANCEDB-CONF-003/005/013/014/022)
Engineering reliability     : PARTIAL — persistence across a real process restart, corruption detection, update/delete/optimize all pass; index type/metric options are ignored, nprobes is inert, non-finite input is only half-validated (EV-LANCEDB-LIFE-005/009, EV-LANCEDB-SCALE-003/004)
```

64 checks executed: **57 PASS, 6 FAIL, 1 PARTIAL_PASS**. Every FAIL is a named finding,
not a broken harness:

| evidence | check | one-line finding |
|---|---|---|
| EV-LANCEDB-CONF-013 | `conformance.infinity_rejected` | `+Infinity` / `-Infinity` components are **accepted and stored**; only NaN is refused |
| EV-LANCEDB-CONF-014 | `conformance.infinity_effect_on_retrieval` | such a row is invisible to cosine search and returns `_distance = Infinity` under l2, which JSON-serialises to `null` |
| EV-LANCEDB-SCALE-003 | `scale.index_type_and_metric_ignored` | `createIndex()` ignores both `indexType` and `distanceType`; every request produces an IVF_PQ index with metric **L2** |
| EV-LANCEDB-AUD-002 | `audit.prior_dataset_hash_identifies_no_artefact` | the prior `datasetHash` hashes an in-memory literal array; no stored dataset exists |
| EV-LANCEDB-AUD-007 | `audit.prior_tolerance_unjustified` | 1e-5 is 1757× the observed error, 84× one binary32 ulp at 1.0, 34× the analytic bound |
| EV-LANCEDB-AUD-012 | `audit.prior_node_version_not_reproducible` | the artefact records Node v24.22.1; this machine is v24.21.0 (conflict B-2) |

---

## 2. What actually ran

| suite | process exit | checks | evidence range |
|---|---|---|---|
| `run-all.cjs` preflight (environment, dataset determinism, reference self-test, scope scan) | 0 | 4 PASS | EV-LANCEDB-RUN-001…004 |
| `distance-semantics.cjs` | 0 | 10 PASS | EV-LANCEDB-SEM-001…010 |
| `conformance-suite.cjs` | 1 | 21 PASS, 2 FAIL | EV-LANCEDB-CONF-001…023 |
| `lifecycle-suite.cjs` | 0 | 9 PASS | EV-LANCEDB-LIFE-001…009 |
| `scale-suite.cjs` | 1 | 5 PASS, 1 FAIL | EV-LANCEDB-SCALE-001…006 |
| `audit-prior-claim.cjs` | 1 | 8 PASS, 3 FAIL, 1 PARTIAL_PASS | EV-LANCEDB-AUD-001…012 |

`node run-all.cjs` reproduces all of it in ~6 s and exits **1** because of the six findings
above. `_run/summary.json` holds the per-suite exit codes and the failure list.

The corpus is stored, not just generated: 12 datasets in
`03-verification/lancedb/_data/` with a sha256 manifest and a seed each
(`_data/manifest.json`). `EV-LANCEDB-RUN-002` regenerates every dataset in memory and
confirms all 12 hashes match both the manifest and the bytes on disk.

---

## 3. Observed distance semantics (the load-bearing measurement)

`_distance` is **not one quantity**. It is a function of `distanceType`, and the default
is not the metric any of the design documents discuss:

| `distanceType` | what `_distance` actually is | proof |
|---|---|---|
| omitted | **squared L2**, `Σ(a-b)²` | EV-LANCEDB-SEM-003, SEM-006 |
| `"l2"` | **squared L2**, not the Euclidean distance | SEM-003: q=`0`, v=`[3,4,0,0]` → 25, not 5; v=`[3,0,0,0]` → 4, not 2 |
| `"cosine"` | `1 - cos(a,q)` | SEM-004: p3 at 45° → 0.2928932309150696 = 1 − √½/1 |
| `"dot"` | `1 - dot(a,q)` (**not** `-dot`, **not** `dot`) | SEM-005: p1 (dot = 2) → −1 |

Three consequences that a contract must state explicitly:

1. **The parent's probe is right and so is the prior harness — for different calls.** Without
   `distanceType` you get squared L2 (confirmed). With `distanceType("cosine")` you get
   `1 - cos` (also confirmed). A report that says "`_distance` is squared L2" and a report
   that says "`_distance` is cosine distance" can both be true of the same build.
2. **`dot` is not an independent metric.** `1 - dot` coincides with `1 - cos` only for
   unit-norm vectors; for un-normalised vectors it silently changes the ranking. It is
   *not* the dot product and must not be documented as such (the SDK's own doc comment,
   `indices.d.ts:70`, says "Dot distance is the dot product", which is wrong for the value
   actually returned).
3. **Storage is binary32, and inputs are quantised on write** (SEM-007): `[0.9,0.1,0]`
   reads back as `[0.8999999761581421, 0.10000000149011612, 0]` — exactly `Math.fround`.
   Any reference that computes over the original doubles is measuring input quantisation
   as if it were engine error.

Other measured semantics:

- **Cosine is magnitude-invariant** (SEM-009): 1e-3·e1, e1, 10·e1 and 1000·e1 all return
  exactly 0 from the e1 query, while their squared-L2 distances differ by up to 1e6.
- **The zero vector is silently excluded from cosine search** (SEM-008): a table of 4 rows
  containing one all-zero row returns 3 rows under cosine and 4 under l2, with no error,
  no warning and no `null`. Docs (`indices.d.ts:66`) do state this; the silent part is
  that a zero or near-zero embedding reduces recall without any signal.
- **The zero vector is not rejected at write time** — only the search drops it.

---

## 4. Audit of the prior claim

**Verdict: the claim is reproducible and arithmetically real, but it is a smoke test, and
three of its words mean less than they appear to. It is not hollow; it is thin.**

### 4.1 Is it reproducible?

Yes, field for field. `EV-LANCEDB-AUD-011` re-executes the original
`lancedb-harness.cjs` with exactly **three** substitutions (SDK module path, SDK
`package.json` path, output path — the canonical tree is read-only and has no module
resolution from this workspace) and compares every field:

| field | result |
|---|---|
| `checks` (all nine booleans incl. `bad_dimension: "REJECTED"`) | identical |
| `datasetHash` | identical |
| `brute` / `lance` arrays | identical |
| `max_abs_cosine_distance_error` | identical, bit for bit (5.692600835160988e-9) |
| `tolerance` | identical (1e-5) |
| `sdk` | identical (0.41.0) |
| `node` | **differs** — recorded v24.22.1, replayed v24.21.0 |

The two path-substituted lines and the one output-path line are the only textual changes;
the audit records the diff. Nothing about the measurement itself was altered.

### 4.2 Was the dataset fixed and hashed?

The **hash reproduces exactly** (`EV-LANCEDB-AUD-001`: sha256 of
`JSON.stringify(rows)` = `155b2634…`), but it does **not** identify a stored dataset
(`EV-LANCEDB-AUD-002`). The six rows are an in-memory literal array in the harness; the
only file the harness writes is its own result JSON. The hash therefore certifies
"these six literals, in this order" and nothing else: it cannot be used to re-load the
corpus, and a reviewer cannot check what was inserted without re-reading and re-running
the source. There is no seed because there is no generator.

### 4.3 Was the reference independent?

In code, yes (`EV-LANCEDB-AUD-003`): the `cosine` arrow function contains no SDK call, no
`_distance` read, and is three lines of plain arithmetic. In method, it is weaker than it
looks:

- it evaluates in **binary64 over the binary64 literals**, while the engine stores and
  computes in binary32. Quantising the inputs before evaluating (`EV-LANCEDB-AUD-005`)
  moves the residual from 5.6926e-9 to 5.19e-9 — i.e. the residual is **kernel arithmetic,
  not input quantisation** — but the reference as written cannot separate the two, so it
  cannot say which part of its 5.7e-9 is the engine's error.
- it breaks ties with `String#localeCompare`, which is locale-dependent. The reference does
  not state a tie rule, and the rule it uses is machine-dependent.
- it has no closed-form self-test. Nothing in the prior harness would have failed if the
  reference itself were wrong.

### 4.4 Did it compare comparable quantities? (the parent's hypothesis)

**No — the hypothesis is refuted for this artefact. The comparison was dimensionally
correct** (`EV-LANCEDB-AUD-004`). The harness calls
`.distanceType('cosine')`, and that returns `1 - cos`; the reference computes `1 - cos`.
Same quantity, same units, same range.

The proof is not an assertion, it is a contrast: re-running the identical comparison with
the metric omitted (i.e. against the default) gives

| call | max \|_distance − (1−cos)\| |
|---|---|---|
| `distanceType("cosine")` | **5.693e-9** |
| default (squared L2) | **2.000** (3.5 × 10⁸ × larger) |

A harness that had compared squared L2 with `1 - cos` could not have produced 5.7e-9; it
would have produced O(1). The tiny residual is itself the evidence that the cosine metric
was active.

So: the parent's concern is valid **as a general hazard** — the default metric is squared
L2 and comparing it to `1 - cos` is exactly the mistake the parent probed for — but the
prior harness did not make that mistake. What it did instead is weaker in three other ways
(below). Any *new* comparison must set `distanceType` explicitly and say so; a contract
that leaves it implicit is one line away from the error.

### 4.5 Other defects, in order of how much they matter

1. **The reported maximum is effectively one sample** (`EV-LANCEDB-AUD-006`). Of the five
   comparisons, four (`a`, `c`, `d`, `e`) are exactly 0, 1, 2 and 0 because the vectors are
   axis-aligned unit vectors. One 3-dimensional dot product carries the entire error claim.
2. **The tolerance is not derived** (`EV-LANCEDB-AUD-007`). 1e-5 is 1757× the observed
   error, 84× one binary32 ulp at 1.0, and 34× the analytic `(d+2)·2⁻²⁴` bound for d=3
   (2.98e-7). A kernel wrong by 1e-5 would still pass. The check cannot discriminate.
3. **The passing `order` check depends on insertion order** (`EV-LANCEDB-AUD-008`). Rows
   `a` and `e` are byte-identical. Rebuilding the same table with `e` inserted before `a`
   returns **`e` first**, so the harness's `score desc, id.localeCompare` comparator would
   then fail. Ties break by insertion/rowid, not by id. `order: true` asserts nothing
   about equal distances.
4. **"restart_persistence" restarts a connection, not a process** (`EV-LANCEDB-AUD-009`).
   Both `connect()` calls run in the same Node process, and the check only asserts
   `length > 0`, never that the delete or the update survived. (The conclusion happens to
   be true: `EV-LANCEDB-LIFE-005` proves durability across a real `spawn`ed process.)
5. **Coverage** (`EV-LANCEDB-AUD-010`, PARTIAL_PASS): 9 scenario classes exercised, 13 not
   — distance-semantics pinning, error distribution, tie policy, duplicate assertion,
   zero vectors, NaN/±Inf, near-duplicate resolution, fresh-process restart, corruption,
   index/recall, version retention/erasure, mutation consistency at scale, tolerance
   justification.
6. **Environment provenance** (`EV-LANCEDB-AUD-012`): Node v24.22.1 vs v24.21.0 on this
   machine — baseline conflict B-2, applies to every file in `evidence/second-round`.

A subtle one worth recording because it nearly fooled this audit: the harness's own
`empty` check creates a table with a sentinel row and then deletes it. `createEmptyTable`
exists and works (`EV-LANCEDB-CONF-009`), so the workaround was unnecessary — but it is
harmless and the check does test what it claims.

---

## 5. Error distribution (not one number)

Reference: binary64 arithmetic over binary32-quantised inputs, `1 - cos` vs the engine's
`_distance` with `distanceType("cosine")`. Corpus `random-d384-n2000` (2000 × 384,
seed 20261105), 5 queries × K=200 → 1000 comparisons (`EV-LANCEDB-CONF-003`):

| statistic | value |
|---|---|
| n | 1000 |
| max \|error\| | **1.1920928932873665e-7** (= 2⁻²³, exactly one binary32 ulp at 1.0) |
| p50 | 1.545e-8 |
| p90 | 3.255e-8 |
| p99 | 4.366e-8 |
| mean | 1.695e-8 |
| bit-identical | **0 / 1000** |
| derived tolerance `4·γ_(d+2)` | 9.20e-5 |

Histogram of \|error\|:

| bin | count |
|---|---|
| 0 … 1e-9 | 31 |
| 1e-9 … 1e-8 | 299 |
| 1e-8 … 1e-7 | 667 |
| 1e-7 … 1e-6 | 3 |
| 1e-6 … 1e-5 | 0 |
| 1e-5 … 9.2e-5 | 0 |
| > 9.2e-5 | 0 |

Squared L2 (`EV-LANCEDB-CONF-004`, 300 comparisons): max relative error **1.96e-7**, max
absolute 3.58e-7, p50 relative 4.41e-8, 3/300 bit-identical — a purely relative quantity,
so an absolute tolerance for l2 is meaningless.

At d=3 the same comparison gives max 1.21e-8 (`EV-LANCEDB-SEM-004`), i.e. the error grows
with dimension but stays inside one ulp at unit magnitude.

**Reading of the distribution.** The engine's cosine distance is a binary32 quantity whose
error is bounded in practice by ~1 ulp at 1.0 (1.19e-7), and it is never bit-exact against
an f64 reference at d=384. The worst-case analytic bound (9.2e-5 at d=384) is 772× the
observed maximum, which means a worst-case-derived tolerance would be far too loose to be
useful as a contract. The honest contract statement is an **empirical envelope**: 
`|_err| ≤ 1.2e-7` on this corpus/dimension, with the analytic bound as the formal ceiling.

---

## 6. Does FLAT exactness hold, and at what scale bound?

**Set membership and total ordering: yes, at every FLAT configuration tested.**

| configuration | depth | result | evidence |
|---|---|---|---|
| n=5000, d=64, 5 queries | K=50 | 0 membership mismatches, max err 1.19e-7 | EV-LANCEDB-SCALE-001 |
| n=2000, d=384, 5 queries | K=100 | 0 order mismatches (sequence, not just set) | EV-LANCEDB-SCALE-002 |
| n=2000, d=384 | K=20 | 0 membership mismatches | EV-LANCEDB-CONF-001 |
| n=300, d=64 | K=300 (full) | sequence identical; smallest adjacent gap 8.32e-6, i.e. 70× the max error | EV-LANCEDB-CONF-002 |

The **bound is empirical, not structural**: the largest tested configurations are
n=5000/d=64 and n=2000/d=384 with K up to 100. Beyond that, "FLAT is exact" rests on FLAT
being an exhaustive scan — which is true by construction but was not exercised at larger
n here, and the scan cost grows linearly (~8 ms per query at n=2000, d=384, K=10;
`EV-LANCEDB-SCALE-006` measured p50 7.94 ms, p95 10.5 ms over 20 queries).

**Resolution floor.** `EV-LANCEDB-CONF-007` ranks a family of near-collinear vectors whose
cosine distances differ by 1e-7, 1e-6, 1e-5, 1e-4, 1e-3 and 5e-2 and gets the order right —
but the smallest gap (1.000e-7) is *equal* to the maximum observed error (1.000e-7). The
margin is 1×, not 10×. The defensible statement is: **gaps below ~2.3e-7 (two binary32 ulps
at 1.0) cannot be resolved reliably**, and the 300-row full ordering had a 70× margin only
because random data rarely produces 1e-7 gaps.

**Complete exactness does not hold** (`EV-LANCEDB-CONF-022`):
over 1000 distances, min = **−1.1920929e-7**, max = 9.358e-1, and **3 values are negative**.
An exact self-match at d=384 returns −1.19e-7, not 0, because the normalised dot product
can round above 1. So:

- `0 ≤ d ≤ 2` holds only to within one ulp — assert a band, never the closed interval;
- `d == 0` is **not** an identity test at high dimension;
- "distance 0" comparisons must be `|d| ≤ 1.2e-7`.

---

## 7. Index behaviour — why cosine exactness is currently "free" and fragile

This is the most operationally important finding after the distance semantics.

1. **`createIndex()` ignores `indexType` and `distanceType`** (`EV-LANCEDB-SCALE-003`).
   Requesting `IVF_FLAT` + `cosine` and requesting `IVF_PQ` + `dot` both produce
   `{"type":"IvfPq","metric":"L2"}` per `listIndices()`. A cosine index cannot be built
   through this API at all.
2. **Therefore every cosine query against any index falls back to a full scan**
   (`EV-LANCEDB-SCALE-005`): the plan contains no `ANNSubIndex`, results are bit-identical
   to the `bypassVectorIndex` ground truth, and the Rust layer prints, verbatim, in the
   suite log:
   `WARN lance::dataset::scanner] Requested metric Cosine is incompatible with index metric L2, falling back to brute-force search`.
   Cosine exactness is preserved **because the approximate path is never taken for this
   metric**, not because the engine is exact. The moment a cosine index becomes buildable,
   all cosine exactness claims must be re-verified.
3. **The l2 index is used, and it is approximate** (`EV-LANCEDB-SCALE-004`): the plan shows
   `ANNSubIndex: name=vector_idx, metric=L2` over `ANNIvfPartition`. Measured recall@10 on
   `random-d384-n2000` was **0.30** (0.30–0.36 across repeated runs; the k-means build is
   not deterministic), and the reported distance for an exact self-match was **0.4217**
   instead of 0 — the PQ approximation leaks into `_distance`.
4. **`nprobes` 1 → 32 changed nothing** in the returned set (recall 0.30 at every value)
   even though the plan shows `minimum_nprobes`/`maximum_nprobes` changing. `refineFactor(3)`
   raised recall to 0.60 and restored the exact distance for the head of the list (side
   probe, `_probe/index-probe6.cjs`).
5. Practical reading: on this build, use FLAT for cosine. A vector index buys nothing for
   cosine and costs ~70% recall for l2 on this corpus.

---

## 8. Other contract-relevant behaviours (measured)

| behaviour | observation | evidence |
|---|---|---|
| Filtering | default is **pre**-filter: `where` + `space='s2'` returns a full K of far-away rows even though the global top-10 are all `s1`; `.postfilter()` returns **0** rows for the same query | CONF-015, CONF-016 |
| Ties | 3 exact-tie groups come back at identical distances, ordered by **insertion/rowid**, not id; `id-ascending` is false | CONF-005, AUD-008 |
| Duplicate vectors | both copies stored and both returned at distance 0; no de-duplication | CONF-006 |
| Duplicate ids | accepted silently — `id` is not a key; the same id can appear twice in one top-K | CONF-018 |
| NaN | rejected at write time with `Vector column contains NaN or null values` | CONF-012 |
| ±Infinity | **accepted and persisted**; invisible to cosine; `_distance = Infinity` under l2 which serialises to `null` | CONF-013, CONF-014 |
| Wrong query dimension | rejected, message names both dimensions (`query dim(2) … vector dim(4)`) | CONF-010 |
| Wrong row dimension | rejected (row count unchanged) but the message is about **NaN/null**, not dimension — an operator cannot diagnose it from the text | CONF-011 |
| Empty table | `createEmptyTable` + query returns `[]` cleanly; `createTable(name, [])` is rejected (recorded) | CONF-009 |
| d=1 boundary | distances are exactly {0, 2}; ordering correct | CONF-008 |
| Large magnitudes | components spanning 1e-3…1e3 stay accurate (max err 1.2e-7) | CONF-019 |
| Update | immediate, new version, `rowsUpdated` reported | LIFE-001 |
| Delete | disappears from plain scan, filtered query and top-K | LIFE-002 |
| Fresh-process reopen | a `spawn`ed process reads the mutated state: count, `active=false`, `rank=999`, deleted row absent, identical top-5 | LIFE-005 |
| Optimize | compaction preserves the logical row set | LIFE-006 |
| Version retention | a deleted row is still readable via `checkout(previousVersion)` — **delete is not erasure** | LIFE-007 |
| Erasure | `optimize({cleanupOlderThan: now, deleteUnverified: true})` prunes the old version; `checkout(1)` then fails | LIFE-008 |
| Corrupted payload | detected at read (not silently returned): `Tail read position … is outside file of size 1603`; in this case `countRows` still reported 6 | LIFE-009 |

---

## 9. Scenarios NOT covered (so nobody infers a PASS from silence)

- Any n > 5000, or d > 384. No test of a multi-million-row corpus.
- Concurrent writers, or a crash **during** a write; no lock/lease behaviour.
- A single-bit flip inside a multi-megabyte payload (the corruption test destroyed a
  1.6 KB file almost entirely — the silent-corruption case is the *partial* one).
- Compaction behaviour after many delete/update cycles; fragment-level effects on ordering.
- Ties **at a K boundary** (does the engine return K rows or K+2 when the K-th distance is
  tied?). The tie tests use K = n.
- `out of range` / duplicate-`space` semantics for a partitioned design (space is just a
  string column here; no per-space isolation or index was tested).
- Denormal, overflow, or >1e3 magnitude components.
- ANN behaviour with a genuinely cosine-trained index — impossible on this build.
- Multi-connection write visibility while a second connection holds a stale snapshot.
- Filesystem-level erasure (backups, CoW snapshots) — only library-level pruning.
- Timing under load: the latency figures are single-run wall clock on a shared machine.

---

## 10. Recommended usage boundary

Within the verified envelope (FLAT, cosine, n ≤ 5000 & d ≤ 384, single writer):

1. **Always set `distanceType` explicitly.** Treat `_distance` as `1 - cos` and nothing
   else; the default is squared L2.
2. **Compare distances with a band**, e.g. `|Δ| ≤ 1e-6` (≈8 ulp at 1.0), never equality and
   never `d == 0`. For a formal worst case use `4·γ_(d+2)`; for a practical one use
   `1.2e-7` at d ≤ 384 (measured).
3. **Treat gaps below ~2.3e-7 as ties** and resolve them in application code; do not rely
   on engine order among equal or near-equal distances.
4. **De-duplicate by id yourself** — duplicate ids and duplicate vectors both survive.
5. **Validate before write**: reject non-finite components (the engine will accept ±Inf),
   reject zero vectors (they vanish from cosine results), and enforce the dimension
   yourself (the insert-time error message does not mention dimensions).
6. **Deletion is logical.** If erasure is a requirement, run
   `optimize({cleanupOlderThan: <now>, deleteUnverified: true})` and treat the row as
   recoverable until then.
7. **Do not create a vector index** if cosine exactness matters: the cosine index cannot be
   built (options ignored) and the l2 index costs ~70% recall at this scale. Budget
   ~8 ms/query at n=2000, d=384 and scale linearly with n.
8. **Treat storage corruption as a process-level failure**, not an exception: the corrupted
   probe surfaced as a read error, but `countRows` still reported the intact metadata.

What would lift the bound: a larger corpus (n ≥ 10⁵) and concurrent-writer tests; a cosine
index that actually honours `distanceType`; and a build that validates non-finite input.

---

## 11. May this contract freeze?

**No — not as a general vector-retrieval contract for this SDK, on this evidence.**

Reasons, each traceable:

- the default metric is not the one the design discusses, and one omitted call silently
  changes the measured quantity by 8 orders of magnitude (SEM-003/006, AUD-004);
- the engine is not bit-exact and cannot be made so through this API (0/1000 bit-identical,
  CONF-003);
- cosine distance leaves `[0,2]` (CONF-022);
- ties have no defined order, so any contract promising deterministic ordering is
  unimplementable here (CONF-005, AUD-008);
- non-finite input is only half-validated (CONF-013);
- the index API does not honour its own options (SCALE-003), so any performance or
  scale-out clause would freeze behaviour that does not exist;
- the bound is empirical (n ≤ 5000, d ≤ 384), not structural.

**What may freeze now**, as a narrow, explicitly bounded contract:

> "FLAT cosine retrieval on `@lancedb/lancedb` 0.41.0, single writer, n ≤ 5000, d ≤ 384,
> no vector index: distances are `1 - cos` and agree with an independent binary64 reference
> to within 1.2e-7 (measured max, 1000 samples); top-K membership and ordering are exact
> when adjacent gaps exceed 2.3e-7; no order is defined among equal or closer distances;
> the caller supplies unique ids, rejects non-finite and zero vectors, and performs its own
> de-duplication; deletion is logical until `optimize` prunes old versions."

Everything else — ANN behaviour, tie order, bit-exactness, erasure, any promise above the
tested scale — should stay **unfrozen** and be marked as open in the baseline.

---

## 12. Limitations of this work

1. One platform (win32-x64), one Node build (v24.21.0), one SDK copy (0.41.0, read-only,
   not re-installable). No version comparison was possible.
2. The reference quantises inputs to binary32 before computing. That is the right model for
   a Float32 column, but it means the reported error is *arithmetic* error, not storage
   error; gross storage bugs would show up as outliers rather than as a bias, and none were
   observed.
3. Tolerances are derived from Higham's `γ_n` with a factor-4 allowance. The factor is a
   judgement, stated in code; the observed errors are 3–4 orders below either bound.
4. The "exact" comparison uses one corpus family (Gaussian, unit scale) plus hand-built
   edge cases. Adversarial corpora (heavy clustering, one huge cluster, all-identical
   vectors) were only partly covered (ties, duplicates, near-duplicates).
5. The index experiments use one build configuration each; recall varies between runs
   (0.30–0.36) because k-means initialisation is not deterministic, and one run per
   configuration is not a benchmark.
6. Latency figures are single-run wall clock on a machine with other agents running.
7. The fresh-process test is a `spawn`ed child on the same machine and the same filesystem;
   it proves process-boundary durability, not power-loss durability.
8. The corruption probe overwrote a 1.6 KB payload almost entirely; partial bit rot in a
   large payload is untested and is the case most likely to be silent.
9. The prior-harness replay required three path substitutions (module path, SDK
   `package.json` path, output path). They are listed and diffed in the raw log; no logic
   was changed.
10. `_probe/` scripts are exploratory and emit no evidence records; no PASS in the evidence
    file depends on them.

---

## 13. Reproduce

```powershell
cd E:\[PROJECT]\r41-baseline\03-verification\lancedb
node dataset-generator.cjs      # optional: rewrites _data/ + manifest (hashes must match)
node run-all.cjs                # ~6 s, exit 1 (6 findings), writes _run/summary.json
```

Per-suite raw logs: `06-evidence/raw/lancedb/<suite>.log` (includes the Rust `WARN` lines),
per-check logs `06-evidence/raw/lancedb/<suite>/<test_id>.log`.
