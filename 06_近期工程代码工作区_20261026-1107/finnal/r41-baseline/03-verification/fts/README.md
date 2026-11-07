# TECH-FTS-01 — Chinese / mixed-language FTS5 · verification harness

Agent C area for the Memory Core R4.1 baseline. Everything here is **isolated verification code**:
no product module, no real memory, no real identity, no permission state.

The finding: FTS5's default `unicode61` tokenizer indexes a whole contiguous CJK run as **one**
token, so Chinese keyword search only works when the query equals an entire run. Measured hit@10
0.80 / recall@10 0.6675 on a labelled corpus, and no `unicode61` option changes it. The recommended
fix needs no dependency and no architecture change. Read `TASK-RESULT.md`.

## Run it

```powershell
cd E:\[PROJECT]\r41-baseline\03-verification\fts

node _data\generate-dataset.cjs     # deterministic dataset: 46 docs, 55 queries, seed 20261105
node run-all.cjs                    # the harness (~6 s): 12 configs, labelled + 20k scale corpus
node analyze.cjs                    # print the tables from the raw run (no FTS is executed)
node render-report.cjs              # regenerate TASK-RESULT.md from that raw run
```

Optional: `node run-all.cjs --quick` skips the synthetic scale corpus (accuracy only, ~1 s).

### Exit codes

| code | meaning |
|---|---|
| `0` | every check ran and passed, **including** the product criteria |
| `1` | at least one check failed. **This is the expected code today**: product criteria PC-1 and PC-3 fail on `unicode61`, while all harness-level checks pass. The console prints `PRODUCT CRITERIA` and `EXIT 1 — …` explicitly so this cannot be mistaken for a crash. |
| `2` | harness error / precondition unmet (dataset missing, `dataset.json` hash mismatch, FTS5 absent) |

A failing product criterion is a real FAIL. Nothing in this harness converts it to a pass.

## Files

| path | role |
|---|---|
| `run-all.cjs` | single entry point: builds 12 indexes, runs 55 labelled queries, measures latency/size/consistency, checks escaping and empty queries, evaluates product criteria, appends evidence |
| `_data/corpus.source.json` | authored documents + query definitions carrying **label rules** |
| `_data/generate-dataset.cjs` | resolves every query's `rel` set from corpus text; writes `dataset.json` (seed + sha256 recorded) |
| `_data/dataset.json` | the frozen, labelled dataset |
| `_data/dataset.meta.json` | seed, dataset sha256, labels sha256, counts |
| `_data/bulk-corpus.json` | 20 000 synthetic, **unlabelled** documents — latency and index size only |
| `_data/report-template.md` | prose of `TASK-RESULT.md` with table placeholders |
| `tables.cjs` | table builders shared by `analyze.cjs` and `render-report.cjs` |
| `analyze.cjs` | reprints the report tables from the raw run |
| `render-report.cjs` | rewrites `TASK-RESULT.md` from template + raw run |
| `PRE-REGISTRATION.md` | proof that expectations were frozen before the first query |
| `_scratch/` | throwaway databases and logs; safe to delete at any time |

## Evidence

* `..\..\06-evidence\fts.jsonl` — one record per executed check (append-only; every record carries
  `run_id` and `raw_output_path`). 71 records per run: 56 `PASS`, 13 `PARTIAL_PASS`, 2 `FAIL`.
* `..\..\06-evidence\raw\fts\run-<run_id>.json` — full raw result (per query, per config: expected
  `rel`, actual ids, latency samples, index sizes, token dumps, model-fit mismatches).
* `..\..\06-evidence\raw\fts\run-<run_id>.log` — console log of that run.

Re-running appends another record set; delete `06-evidence/fts.jsonl` first if exactly one set is
required.

## Invariants this harness enforces

1. **Expectations cannot move.** `run-all.cjs` recomputes `sha256(dataset.json)` and exits 2 if it
   differs from `dataset.meta.json`. Labels are derived from corpus text and authored constants only —
   never from an FTS5 result.
2. **The engine, not the test, is measured.** Query expressions actually executed are recorded per
   query per config (`executed_expr`), so a "hit" produced by a query designed around the tokenizer is
   visible as such in the raw JSON.
3. **Accuracy and latency are separated.** Accuracy comes only from the labelled 46-document corpus;
   the 20 000-document corpus is synthetic and unlabelled and yields **no** accuracy figure.
4. **Determinism is checked, not assumed.** Vocabulary hash and result sets are compared across
   close/reopen for every config, and the same string is inserted twice to compare token streams.

## Environment

Node `v24.21.0`, SQLite `3.53.4` (`node:sqlite`), Windows 10.0.26200 x64. No npm/pnpm, no
third-party tokenizer, nothing installed. `node:sqlite` does expose `loadExtension()`, but no compiled
tokenizer library exists on this machine and none can be built or fetched here — recorded as
`fts.extension_path` evidence.
