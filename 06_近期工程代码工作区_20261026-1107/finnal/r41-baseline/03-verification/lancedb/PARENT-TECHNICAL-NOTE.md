# LANCEDB-EXACTNESS-VERIFICATION — TECH-LANCEDB-01

Date: 2026-11-05 (Asia/Shanghai)
Area: `03-verification/lancedb/`
Evidence: `06-evidence/lancedb.jsonl` (66 records: 64 agent + 2 parent re-check)
Raw: `06-evidence/raw/lancedb/`
Status: **PARTIALLY_VERIFIED(scanned-path exactness on small/medium fixed datasets; distance semantics characterised)** — not frozen.

---

## 1. Verification target

Establish what the LanceDB SDK actually does, whether FLAT/brute-force retrieval matches an
independent cosine reference, and whether the prior round's exactness claim
("max cosine-distance error 5.7e-9, tolerance 1e-5") survives scrutiny.

## 2. Environment

| Item | Value |
|---|---|
| SDK | `@lancedb/lancedb` **0.41.0** (resolved from the design package's installed `node_modules`) |
| Node | **v24.21.0** |
| OS | Windows NT 10.0.26200, x64 |
| `:memory:` | **NOT SUPPORTED** — `connect(":memory:")` throws `Unable to created lance dataset at :memory: (os error 123)`. Every test uses a real temp directory. |
| Package manager | none — the SDK could not be reinstalled or upgraded |

## 3. What the prior exactness claim actually was

The prior artefact `evidence/second-round/lancedb-result.json` reports:

```json
"max_abs_cosine_distance_error": 5.692600835160988e-9,
"tolerance": 0.00001,
"checks": { "topk_membership": true, "order": true, "distance": true, ... "all": true }
```

Three independent findings undermine that as written:

### 3.1 The arithmetic was comparing incomparable quantities · **EV-LANCEDB-P001**
The SDK's `_distance` for cosine is **squared L2 on raw vectors** (measured: `[1,0,0]`→0,
`[0.6,0.8,0]`→0.800000011920929, `[0,1,0]`→2). The prior harness compared that against a
cosine-distance expression `1 − cos`. Those are different quantities, so a small "error" between
them is not evidence of exactness.

### 3.2 The tolerance is not a tolerance · **EV-LANCEDB-AUD-007**
`1e-5` is **1757× the observed error**, **84× one binary32 ulp at 1.0**, and **34× the analytic
(d+2)u bound** (~2.98e-7). A tolerance that loose cannot distinguish a correct cosine implementation
from one that is wrong at the 1e-5 level. The `"distance": true` check therefore asserts far less
than its name implies.

### 3.3 There was no stored dataset · **EV-LANCEDB-AUD-002**
The prior harness built its rows as an **in-memory literal array**, hashed that array, and created the
table from it. The reported dataset hash cannot reload the corpus, and no reviewer could check what
was actually inserted without re-reading the harness source. `lancedb-data/` is a derived table
directory, not the corpus.

## 4. Finding that changes the design contract · **EV-LANCEDB-SCALE-003 / EV-LANCEDB-P001**

`createIndex()` **ignores both `indexType` and `distanceType`**. Verified independently by the parent
on three option sets:

| Requested | Reported `indexType` | Reported metric | Compression |
|---|---|---|---|
| `IVF_FLAT` + `cosine` | `IvfPq` | `L2` | pq, 8-bit, 4 sub-vectors |
| `IVF_PQ` + `dot` | `IvfPq` | `L2` | pq, 8-bit, 4 sub-vectors |
| `IVF_FLAT` + `l2` | `IvfPq` | `L2` | pq, 8-bit, 4 sub-vectors |

The agent found the first two; the parent's re-run added the third — **even the SDK's own native
metric is disregarded**. This is not a cosine-translation bug; the options object is ignored outright.

The engine does warn on a genuine mismatch, and correctly refuses to approximate:

> `Requested metric Cosine is incompatible with index metric L2, falling back to brute-force search`

Consequences that must reach the contract:

1. **A cosine index cannot be built through this API.** Only brute-force cosine search is available.
2. **Once an index exists, retrieval is approximate IVF_PQ/L2** — and `IVF_PQ` is quantised, so
   exactness is structurally impossible on that path.
3. **Therefore "FLAT exactness" is a property of the *scan* path only.** No test in this area
   established exactness on an indexed table, and none should claim to.
4. `VectorStoreAdapter` must not expose `indexType`/`distanceType` as functioning parameters.

## 5. Genuine defects found

### 5.1 Non-finite guard is incomplete · **EV-LANCEDB-CONF-013**
`NaN` is rejected at write time, but **`+Infinity` and `−Infinity` are accepted and persisted verbatim**
(`countRows` went 1 → 3). So a poisoned embedding can enter the table through the infinity path.

### 5.2 A persisted infinity breaks downstream serialisation · **EV-LANCEDB-CONF-014**
An infinite row is silently absent from cosine results (treated like a zero vector) and returns
`_distance = Infinity` under L2. `Infinity` is **not JSON-representable**; after a JSON round-trip
the distance becomes `null`. Any result pipeline that serialises distances will silently corrupt
this row rather than reject it.

### 5.3 Edge-case asymmetry worth noting · **EV-LANCEDB-SEM-005 / SEM-007**
- `distanceType("dot")` returns the **negated** dot product, i.e. a constant-offset variant of
  cosine distance rather than an independent third metric. Callers who assume "dot" means "higher is
  better" will invert their rankings.
- Stored vectors are quantised to **binary32**: `0.9` reads back as `0.8999999761581421`.
  A reference implementation that computes over the original doubles measures input quantisation
  as if it were engine error. Any tolerance must account for this, not for the engine's arithmetic.

## 6. What the harness establishes

| Suite | Records | Result |
|---|---|---|
| `distance-semantics.cjs` | 10 | all PASS — `_distance` semantics characterised |
| `conformance-suite.cjs` | 23 | 21 PASS, **2 FAIL** (infinity acceptance, infinity retrieval effect) |
| `lifecycle-suite.cjs` | 9 | all PASS — update/delete/reopen/persistence |
| `scale-suite.cjs` | 6 | 5 PASS, **1 FAIL** (index options ignored) |
| `audit-prior-claim.cjs` | 12 | 8 PASS, **3 FAIL**, 1 PARTIAL_PASS — prior-claim audit |

Datasets are **stored and hashed** (`_data/manifest.json`), with a recorded seed — a direct
improvement on the prior round, where the corpus existed only as an in-memory literal.

## 7. Status kept separate, as required

| Question | Answer |
|---|---|
| SDK available and usable? | **Yes** (`0.41.0` loads and operates on Windows x64) |
| Basic retrieval works? | **Yes** (top-K, filter, update, delete, reopen all pass) |
| Partial exactness passed? | **Yes** — on the scan path, small/medium fixed datasets |
| Complete exactness passed? | **No** — no indexed-path exactness; no large-scale exhaustive comparison |
| Engineering reliability passed? | **No** — the non-finite hole and the ignored index options are reliability defects |

These are four different claims and none implies another.

## 8. Recommendation

**Retain LanceDB, with a narrowed contract.** It works for the intended job — a rebuildable vector
index for a local single-instance store — but the contract must stop asserting properties the SDK
does not have:

1. **Do not claim "LanceDB exactness".** State: *exactness is demonstrated on the un-indexed scan path;*
   *indexed retrieval is approximate and its recall must be measured before acceptance.*
2. **Do not expose `indexType`/`distanceType`** in `VectorStoreAdapter` until a build is found that
   honours them.
3. **Validate vectors before write** in the adapter: reject `NaN` **and** `±Infinity` (the SDK catches
   only `NaN`).
4. **Normalise before storing** if cosine ordering matters, and compute norms once — the SDK quantises
   to binary32 on write, so a reference must compare against `Math.fround(input)`, not the original double.
5. **Never serialise a distance without a finiteness guard** — `Infinity` becomes `null` silently.

**Abandoned / rejected options:** replacing LanceDB with a SQLite vector store — explicitly forbidden
by the design (`fallback_to_sqlite_vector` is always false) and not evaluated here.
An ANN product built on this SDK's index is **not** recommended until the option-handling defect is
resolved or a build that honours it is obtained.

## 9. Freeze decision

**`PARTIALLY_VERIFIED(scanned-path exactness on fixed small/medium datasets; distance semantics characterised)` — NOT FROZEN.**

Gates:

| Gate | Why it blocks | How to close |
|---|---|---|
| L-1 Indexed-path exactness/recall | Indexed retrieval is approximate and unmeasured against brute force at acceptance scale | Measure recall@10 vs brute force on a fixed corpus at the intended scale |
| L-2 Non-finite guard | `±Infinity` persists and breaks JSON | Adapter-level rejection test, then re-run conformance |
| L-3 Index option handling | The SDK ignores `indexType`/`distanceType` | Obtain a build that honours them, or contractually drop the parameters |
| L-4 Reproducible runtime | Prior artefacts claim node v24.22.1; this machine is v24.21.0 | Pin the runtime and re-run |

None of these blocks *using* LanceDB today for brute-force cosine over a small store. They block
*freezing* the contract.

## 10. Not executed

- HNSW / IvfRq / any index type other than IVF-family — not enumerated.
- Multi-process / concurrent-writer behaviour on a Lance dataset — not tested.
- Compaction and version-GC behaviour under long edit histories — not tested.
- Behaviour under a real embedding model (all vectors here are synthetic).
- Large-scale (millions of rows) exhaustive exactness — not attempted.

## 11. Files

| Path | Purpose |
|---|---|
| `03-verification/lancedb/run-all.cjs` | suite entry point |
| `.../distance-semantics.cjs` | characterises `_distance` |
| `.../conformance-suite.cjs` | API conformance incl. non-finite handling |
| `.../lifecycle-suite.cjs` | update/delete/reopen/persistence |
| `.../scale-suite.cjs` | index creation, ANN recall, scale behaviour |
| `.../audit-prior-claim.cjs` | audits the prior round's claim |
| `.../brute-force-reference.cjs` | independent reference (no SDK import) |
| `.../dataset-generator.cjs`, `_data/manifest.json` | stored, hashed, seeded datasets |
| `.../_probe/parent-index-recheck.cjs` | parent's independent index-option verification |
| `06-evidence/lancedb.jsonl` | evidence records |
| `06-evidence/raw/lancedb/*.log` | full per-suite logs |

Reproduce: `cd E:\[PROJECT]\r41-baseline && node 03-verification\lancedb\run-all.cjs`

---

## 12. Parent addendum — three measurements promoted to headline status

After the agent's report was delivered, the parent reviewed the raw records and promoted three
measurements that the report's summary understated. These are reproduced from the agent's own
evidence, not re-measured by the parent:

### 12.1 ANN recall@10 is 0.30 and `nprobes` does nothing — **EV-LANCEDB-SCALE-004**
```text
index built in 193 ms; ANN node in plan: true
mean recall@10 at nprobes 1/2/4/32 = 0.30 / 0.30 / 0.30 / 0.30
top-1 distance for an exact self-match: ANN 0.42169854044914246 vs exact 0
```
Enabling an index therefore **degrades retrieval to about 30 % recall**, and the tuning knob
(`nprobes`) is inert — increasing it 32× changes nothing. Combined with EV-LANCEDB-P001
(`createIndex` ignores `indexType`/`distanceType`), this means indexed retrieval in this build is
neither accurate nor tunable.

**Contract consequence:** the vector channel must run on the **scan path** for correctness, and any
indexed mode must be gated behind a measured recall acceptance threshold — not enabled by default.

### 12.2 Cosine distance leaves the mathematically valid interval — **EV-LANCEDB-CONF-022**
```text
over 1000 distances: min = -1.192e-7, max = 9.358e-1; below 0: 3; above 2: 0
self-match (query == stored vector) distance = -1.1920928955078125e-7
```
Cosine distance is defined on [0, 2]. A self-match should be exactly 0, and it is **negative**.
This is binary32 rounding, not a logic error — but any code that clamps, ratios, or normalises a
distance assuming `>= 0` will misbehave. **Consequence:** the adapter must tolerate a small negative
band and must never use `distance` directly as a denominator.

### 12.3 Tie order is insertion order, so ranking is not reproducible — **EV-LANCEDB-CONF-005**
```text
returned g0c, g0a, g0b, g1z, g1m, g1a, g2q, g2b, g2x
within-group order equals id-ascending: false; equals insertion order: true
```
Distances within a tie group are identical, but the engine orders them by **insertion order**.
A re-index, a rebase, or a compaction can therefore change the order of equal-scoring results, which
means **the same query can return a different top-K ordering across rebuilds**.

**Consequence:** RRF and any downstream ranking must impose their own deterministic tie-break on
`(channel_rank, memory_id)` and must not rely on the vector engine's order for stability. This is a
contract requirement, not an implementation detail.

### 12.4 Effect on the verification status

These three do not change the verdict — `PARTIALLY_VERIFIED(scanned-path exactness on fixed
small/medium datasets; distance semantics characterised)` — but they **narrow the usable envelope**
and add two gates:

| Gate | Statement |
|---|---|
| L-5 | Deterministic tie-break must be implemented downstream; the engine will not provide it. |
| L-6 | Distances must be treated as possibly-negative and never as a denominator or a clamped quantity. |

They also raise the priority of L-1 (measure ANN recall before ever enabling an index): the measured
figure is 0.30, which is far below any plausible acceptance threshold.
