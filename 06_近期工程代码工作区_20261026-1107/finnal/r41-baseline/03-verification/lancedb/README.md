# TECH-LANCEDB-01 — LanceDB 0.41.0 exactness vs an independent brute-force reference

Agent B area. Everything here is isolated verification code: it writes only inside
`03-verification/lancedb/`, `06-evidence/lancedb.jsonl` and `06-evidence/raw/lancedb/`.
No product module, no real memory data, no permission state.

**Conclusion first:** read `TASK-RESULT.md`. This file explains how to run the code
and what each pinned definition means.

## Run it

```powershell
cd E:\[PROJECT]\r41-baseline\03-verification\lancedb
node run-all.cjs          # full run, ~5 s, exit 1 because 6 checks legitimately FAIL
```

`run-all.cjs` truncates `06-evidence/lancedb.jsonl`, deletes `_scratch/`, rebuilds and
verifies the datasets, runs the five suites in separate processes, and writes
`_run/summary.json`.

Exit codes follow `00-PARENT-MANDATE.md §5`:

| code | meaning |
|---|---|
| 0 | every check ran and passed |
| 1 | at least one check FAILED (**the expected outcome on this build**) |
| 2 | harness error / precondition unmet |

Do not read exit 1 as "the harness is broken": each FAIL in `_run/summary.json` is a
named finding, and `TASK-RESULT.md §4` says which ones matter.

## Files

| path | role |
|---|---|
| `run-all.cjs` | entry point; preflight + all suites + aggregation |
| `dataset-generator.cjs` | deterministic generator; writes `_data/*.json` + `_data/manifest.json` with sha256 per dataset. Also importable: `buildDatasets()` returns the bytes for a determinism check |
| `brute-force-reference.cjs` | **independent ground truth** — no `require` at all, no SDK, no `_distance`. All distance and ordering definitions are pinned in its header |
| `distance-semantics.cjs` | what `_distance` means for `l2` (default), `cosine` and `dot`; storage precision; zero-vector handling |
| `conformance-suite.cjs` | top-K, full ordering, error distribution, ties, duplicates, near-duplicates, boundaries, filters, invalid input, mutation consistency |
| `lifecycle-suite.cjs` | update/delete/optimize, same-process and fresh-process reopen, version retention, corrupted payload |
| `_children/reopen-child.cjs` | fresh OS process that re-reads the database |
| `_children/corrupt-child.cjs` | corrupts a `.lance` payload and reports what the engine does (may die) |
| `scale-suite.cjs` | FLAT exactness at n=5000/d=64 and n=2000/d=384, index type/metric behaviour, ANN recall, latency |
| `audit-prior-claim.cjs` | audits `evidence/second-round/lancedb-{result.json,harness.cjs}` and replays the original harness |
| `lib/` | evidence recorder, dataset loader, SDK loader (the only file that imports the SDK) |
| `_data/` | stored datasets + manifest (12 files, ~6 MB, all hashed) |
| `_probe/` | throwaway probes used while establishing the semantics. Not part of `run-all`; kept for process transparency |
| `_scratch/` | disposable per-run LanceDB directories, deleted by each suite |

## Environment constraints (verified, not assumed)

- `@lancedb/lancedb` **0.41.0** is loaded by absolute path from
  `C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design\evidence\second-round\node_modules`.
  Override with `LANCEDB_NODE_MODULES`. There is no npm/pnpm, so nothing can be installed.
- `connect(":memory:")` throws (`Unable to created lance dataset at :memory:`), so every
  test uses a disposable directory under `_scratch/`.
- Node on this machine is **v24.21.0**; the prior artefacts record v24.22.1 (finding AUD-012).

## Pinned definitions

The reference pins, in its header and in code:

- inputs quantised to binary32 (`Math.fround`) before arithmetic, because the stored
  column is Float32; arithmetic then in binary64;
- cosine distance = `1 - cos`, squared L2 = `Σ(a-b)²` — two functions, two names, never
  interchanged;
- ascending distance, ties by **UTF-16 code unit** id order (never `localeCompare`);
- top-K = first K after that sort; zero vectors excluded from cosine ranking and reported;
- tolerance derived from Higham's `γ_n`, not copied from any earlier report:
  `tol_cos(d) = 4·γ_(d+2)`, `tol_sqL2(d, ref) = 4·γ_(2d)·|ref|`.

`distance-semantics.cjs` establishes empirically that these names match the engine:
default/`l2` → **squared** L2, `cosine` → `1 - cos`, `dot` → `1 - dot`.

## Evidence

One JSON line per executed check in `06-evidence/lancedb.jsonl` (64 records per full run),
per-check raw log in `06-evidence/raw/lancedb/<suite>/<test_id>.log`, per-suite console log
(including the Rust `WARN` lines) in `06-evidence/raw/lancedb/<suite>.log`.
The record schema follows `02-shared/EVIDENCE-FORMAT.md`; `metrics` is an extra key, no
required key is replaced. `exit_code` on a record is the exit code of the check as an
execution unit (0 pass, 1 expectation not met, 2 the check threw) — the suite/`run-all`
process exit codes are separate and are recorded in `_run/summary.json`.

Every `PASS` in that file was produced by a check that actually ran on this machine in
this session. `_probe/` scripts emit no evidence records on purpose: they were exploratory.
