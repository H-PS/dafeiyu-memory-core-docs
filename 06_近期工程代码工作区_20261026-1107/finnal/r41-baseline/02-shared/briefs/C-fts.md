# BRIEF — Agent C · TECH-FTS-01 · Chinese / mixed-language FTS5

Read first:
1. `E:\[PROJECT]\r41-baseline\02-shared\00-PARENT-MANDATE.md`
2. `E:\[PROJECT]\r41-baseline\02-shared\EVIDENCE-FORMAT.md`
3. `E:\[PROJECT]\r41-baseline\01-baseline-corpus\design-package\02-核心算法规范.md` (text retrieval chapter)
4. `E:\[PROJECT]\r41-baseline\01-baseline-corpus\design-package\15-参数注册表.md` (any FTS/tokenizer params)
5. Prior artefact: `...\evidence\second-round\sqlite-file-result.json` — note `fts_chinese_unicode61: "NO_HIT"`

Own this area only: `03-verification\fts\`. Evidence: `06-evidence\fts.jsonl`; raw: `06-evidence\raw\fts\`.

## Environment facts (parent-verified — reproduce, do not assume further)

Node v24.21.0, SQLite **3.53.4**, compile options include `ENABLE_FTS3`, `ENABLE_FTS3_PARENTHESIS`,
`ENABLE_FTS5`. **No third-party tokenizer extension can be installed** (no package manager, no
extension loading of unbuilt binaries). So the realistic candidate set is what FTS5 ships natively.

Parent's own quick probe (reproduce these before building on them):

| tokenizer | query `中文` | query `中文测试内容` | query `记忆库` | query `mixed` in `混合mixed文本` |
|---|---|---|---|---|
| `unicode61` (default) | no hit | hits | no hit | no hit |
| `trigram` | no hit | hits | hits | hits |

Note carefully: `unicode61` **does** match the full contiguous run because the whole CJK run becomes
one token — so "no hit" is not a total failure, it is a *segmentation* failure. `trigram` matches
substrings but still missed the 2-character query `中文`. Your job is to characterise this precisely
rather than to declare pass/fail.

## What you must separate

- English FTS working (already claimed) is **not** evidence for Chinese FTS.
- A tokenizer matching a query you designed around it is not recall. You need labelled expectations
  written **before** running queries.

## Dataset

Build a fixed, labelled corpus (`dataset.json` + generator with a recorded seed + recorded sha256).
Cover: continuous Chinese text; Chinese phrases; Chinese single characters; Chinese keywords;
Chinese punctuation; mixed Chinese/English; English; digits; abbreviations; Unicode edge cases;
whitespace; special symbols; the same word in different contexts; and at least one negative control
(document that must NOT match).

**Every query carries a pre-declared expected hit set** (document ids). Write the expectation
before you run the query and do not edit an expectation after seeing a result. If an expectation
turns out to be wrong, record it as a dataset defect in the TASK-RESULT — do not silently fix it.

## Metrics you can honestly produce

Hit rate; Recall@K; Precision@K where labels genuinely support it; ranking output; empty-query
behaviour; query latency; index size on disk; false positives; false negatives; tokenization
consistency (same string → same tokens across open/close). **Do not invent accuracy figures where
labels are insufficient** — say "not computable from this labelling" instead.

## Compare, then recommend

Compare at minimum: `unicode61` (default), `unicode61` tuned options (`remove_diacritics`,
`tokenchars`, `separators`), `trigram`, and prefix/phrase query forms. Then state plainly:

- Is current `unicode61` adequate for Chinese retrieval in this product? Support it with the hit-rate
  numbers you produced.
- If not, what is the alternative and **what does it cost** (dependency, Windows compatibility,
  install complexity, index size, migration, maintenance)?
- Does the retrieval architecture need to change, or can the fix stay inside the tokenizer choice?
  Prefer a solution that does not change the overall architecture.
- Never lower the test standard to declare success.

`trigram` has a documented constraint (it requires ≥3-character queries); if your data hits that,
record it as a real limitation with the exact failing queries, not as a harness bug.

## Deliverable

`03-verification\fts\TASK-RESULT.md` plus `run-all.cjs` and `README.md`.
Include the tokenizer comparison table, the per-query expected-vs-actual table, latency and index
size, dependency risk, recommendation, and whether FTS may freeze — and if not, the exact gate.
Parent will re-run `run-all.cjs`.
