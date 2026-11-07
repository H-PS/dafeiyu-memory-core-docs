# PRE-REGISTRATION — TECH-FTS-01 (Agent C)

Written **before** any FTS5 index was created and before any `MATCH` query was executed.

## Frozen artefacts

| Artefact | sha256 | Frozen at |
|---|---|---|
| `_data/dataset.json` (46 documents + 55 queries with resolved `rel` sets) | `77489ddec2534cf98d0b2e5e4cd8239e7789e20ea6a559389dda280e12e0b969` | 2026-11-05T20:12Z (see `_data/dataset.meta.json`) |
| labels (`[[query_id, rel[]], …]`) | `10e92de851ec4c9443076f464900ffac066728015a3d20a39690e8bf24cb306e` | same |
| generator | `_data/generate-dataset.cjs`, seed `20261105`, version `generate-dataset.cjs@v1` | same |

`run-all.cjs` recomputes `sha256(dataset.json)` at start-up and refuses to run (exit 2) if it does
not match the value recorded in `_data/dataset.meta.json`.

## Rule that was applied

1. Documents were authored first (`_data/corpus.source.json`).
2. Each query's `rel` set was **materialised from the document text** by the generator, using a label
   rule declared in the source file (`substring` / `wordprefix` / `explicit` / `union`), and stored in
   `dataset.json` together with the rule that produced it (`label_rule`, `label_source`).
3. No `rel` set was produced, adjusted or re-checked by an FTS5 query. `run-all.cjs` only reads it.
4. An expectation that turns out to be wrong is recorded as a dataset defect in `TASK-RESULT.md`
   (see `DEF-01`), never edited away.

## What `rel` means here (scope of the claim)

`rel` is a **lexical** relevance label: a document is relevant to a query iff the query string occurs
in one of the four indexed fields (`title`, `content`, `tags`, `summary`), case-insensitively for
ASCII. There are **no human semantic-relevance judgements** for this corpus, so the numbers below are
retrieval-correctness numbers for lexical queries. They are not evidence about semantic recall, and
`TASK-RESULT.md` says so explicitly wherever a figure is quoted.

Queries with `rel = []` (q30, q35, q37, q38, q40–q42, q49–q51) contribute **no** recall/precision
value (spec `recall_at_k`: `|Rel| = 0 → 不适用（null）`, not counted in the mean); they contribute
false-positive counts only.
