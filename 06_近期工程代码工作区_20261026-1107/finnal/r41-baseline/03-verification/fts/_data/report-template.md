# TASK-RESULT — TECH-FTS-01 · Chinese / mixed-language FTS5

**Agent C** · run `{{RUN_ID}}` · dataset `{{DATASET_SHA}}`
Environment: Node `{{NODE}}`, SQLite `{{SQLITE}}`, `{{OS}}`, FTS5 via `node:sqlite`.
Raw evidence for every number below: `{{RUN_RAW}}` (regenerate with `node run-all.cjs`).

---

## 0. Verdict in one paragraph

`unicode61` (the FTS5 default and the only tokenizer the current design would get for free) is **not
adequate for Chinese retrieval in this product**. It does not "fail to tokenize" — it tokenizes a
whole contiguous CJK run into **one** token, so a query hits only when the query is *character-for-character
equal to an entire run in a field*. Measured on a 46-document labelled corpus with 55 pre-registered
queries: **hit@10 = {{U61_HIT}}, recall@10 = {{U61_R10}}, recall@5 = {{U61_R5}}**, with {{U61_ZERO}} of the
45 scored queries returning **zero** rows, and a run-equality model reproduces its hit sets for
{{MF_RE_FITS}}/{{MF_N}} ({{MF_RE}}) single-token queries versus {{MF_SUB_FITS}}/{{MF_N}} ({{MF_SUB}}) for a
substring model.
`trigram` is worse for Chinese ({{TG_HIT}} hit@10) because **every query shorter than 3 characters
silently returns nothing**, and 2-character words are the normal case in Chinese. The one option that
fixes this with **no new dependency and no architecture change** is to keep `unicode61` and index an
additional unigram+bigram expansion of the CJK text (pure JS, index-time transform + query rewrite,
`{{UB_HIT}}` hit@10 / `{{UB_R10}}` recall@10, cost 3.0× index bytes). **FTS must not freeze** until the
gate in §13 is met.

---

## 1. Scope and authorisation

| Item | Value |
|---|---|
| Code scope | isolated verification harness under `03-verification/fts/` only |
| Product implementation | **NOT_AUTHORIZED / not attempted** — nothing under any product module path was written |
| Data | synthetic corpus authored in this task; no real user memory, no real DSH/QQ Bridge identity, no permission state |
| Databases | throwaway files under `03-verification/fts/_scratch/`, deleted and rebuilt on every run |
| Emotion runtime | untouched (`emotion.*` never referenced) |

What this report **does not** claim: it is not a statement about semantic retrieval quality, about
embedding quality, about end-to-end recall through RRF, or about behaviour under concurrent writers.
It is a measurement of FTS5 lexical retrieval for Chinese-shaped text on this machine.

---

## 2. Environment reproduced (not assumed)

| Fact | Observed this run |
|---|---|
| Node | `{{NODE}}` |
| SQLite | `{{SQLITE}}` (`select sqlite_version()`) |
| Compile options | `ENABLE_FTS3`, `ENABLE_FTS3_PARENTHESIS`, `ENABLE_FTS5`, `ENABLE_DBSTAT_VTAB` present |
| FTS5 tokenizers available natively | `unicode61`, `ascii`, `porter`, `trigram` (all four constructed successfully) |
| `fts5vocab` virtual table | available (used for the token dumps below) |
| OS / arch | `{{OS}}` |
| npm / pnpm | still absent — nothing was installed |

### 2.1 One correction to the brief's premise

The brief states that no tokenizer extension can be installed. That is true **in effect**, but the
reason is narrower than "extension loading is impossible":

```
EXTENSION PATH PROBE — {{EXT_API}}; load of a non-existent tokenizer dll: {{EXT_ERR}}
```

So `node:sqlite` **does** expose the extension-loading API; what is missing is a compiled
win32-x64 tokenizer library and any means to obtain or build one here (no package manager, no
verified C toolchain). Consequence: a third-party Chinese tokenizer is not reachable *today*, but
unblocking it is a packaging task (ship a prebuilt `.dll`), not an architectural one. This is
recorded as an option with its cost in §12 and does not change the recommendation, which is
deliberately dependency-free.

---

## 3. Deliverables and reproduction

| File | Role |
|---|---|
| `run-all.cjs` | **single entry point**. 12 tokenizer configurations × 55 queries, plus scale corpus, consistency, escaping, product criteria, evidence emission |
| `_data/corpus.source.json` | authored corpus: 46 documents + 55 query definitions with label *rules* |
| `_data/generate-dataset.cjs` | resolves every `rel` set from corpus text, writes `dataset.json` + `dataset.meta.json` (seed `{{SEED}}`, sha256) |
| `_data/dataset.json` | frozen labelled dataset, sha256 `{{DATASET_SHA}}` |
| `_data/bulk-corpus.json` | synthetic **unlabelled** 20 000-document corpus, sha256 `{{BULK_SHA}}` — latency/size only |
| `tables.cjs` / `analyze.cjs` | regenerate every table in this report from the raw run |
| `render-report.cjs` | regenerates this file from `_data/report-template.md` + raw evidence |
| `PRE-REGISTRATION.md` | the frozen-expectation record written before the first query ran |
| `README.md` | how to run, what each exit code means |

Reproduce:

```powershell
cd E:\[PROJECT]\r41-baseline\03-verification\fts
node _data\generate-dataset.cjs      # rewrites dataset.json; sha256 must equal {{DATASET_SHA}}
node run-all.cjs                     # ~6 s; writes 06-evidence/fts.jsonl + 06-evidence/raw/fts/
node analyze.cjs                     # prints the tables below from the raw run
node render-report.cjs               # rewrites this file from that raw run
```

`run-all.cjs` refuses to run (exit 2) if `dataset.json` no longer hashes to the pre-registered value —
expectations cannot be edited after the fact without the harness noticing.

**Exit code**: `0` all checks and product criteria passed · `1` at least one check failed · `2`
precondition unmet. This run exits **1**, and that is the honest verdict: PC-1 = {{PC1}} and
PC-3 = {{PC3}}, while every harness-level check passed. See §13.

---

## 4. Dataset and labelling

46 documents, {{N_QUERIES}} queries, 4 indexed fields per document (`title`, `content`, `tags`,
`summary` — the field set of `SearchDocument` and of ALG-02 §5.3, so the bm25 weights 3.0/1.0/1.5/1.2
apply unchanged). Coverage: continuous Chinese with no punctuation; Chinese phrases; Chinese single
characters; recurring Chinese keywords (|Rel| > K by construction); Chinese punctuation; mixed
Chinese/English; English only; digits; abbreviations (`FTS5`, `bge-m3`, `user_id`, `snake_case`);
Unicode edge cases (precomposed accent, combining mark, emoji, full-width digits, CJK extension B,
U+200B zero-width space, U+3000 ideographic space, tabs/newlines); whitespace-collapsing cases;
FTS5 syntax characters in the text; the same word in different senses (`苹果` fruit vs company);
negative controls.

**How expectations were produced (and why that matters).** Documents were authored first. Each query
declares a *label rule* (`substring`, `wordprefix`, `explicit`, `union`); `generate-dataset.cjs`
materialised the `rel` set from the document text **before any index existed**, and stored the rule
next to the result (`label_rule`, `label_source`) in `dataset.json`. No expectation was produced,
adjusted or re-checked by an FTS5 query, and `run-all.cjs` only reads them.
`PRE-REGISTRATION.md` records seed `{{SEED}}` and sha256 `{{DATASET_SHA}}`;
`sha256(labels) = 10e92de851ec4c9443076f464900ffac066728015a3d20a39690e8bf24cb306e`.

**What `rel` means — read this before quoting any number.** `rel` is a **lexical** label: a document
is relevant iff the query string occurs in one of the four fields. There are **no human
semantic-relevance judgements** for this corpus. Therefore:

* hit-rate / recall@K / precision@K below are **lexical retrieval correctness**, not semantic accuracy;
* **nDCG, semantic recall@K and human-judged precision are not computable from this labelling** — no
  graded or pooled relevance judgements exist, and the report says so instead of inventing them;
* 45 queries carry a non-empty `rel` and are scored; 10 queries are labelled must-not-match
  (`|Rel| = 0`); per the design's own metric definition (`|Rel| = 0 → 不适用（null）`) those contribute
  no recall value, only false-positive counts.

---

## 5. Candidate set, and why it is closed

Only what FTS5 ships natively can be tested: `unicode61` (default and tuned), `ascii`, `porter`,
`trigram`, plus query forms (prefix, phrase, boolean) and two engineering workarounds that use no
extension at all (an indexed CJK n-gram shadow column; a `LIKE` full-scan mode). ICU, jieba, the
`simple` tokenizer and any other third-party tokenizer are **excluded by the environment**, not by
preference (§2.1).

The 12 configurations actually built and run:

| # | config | tokenize | transform |
|---|---|---|---|
| 1 | `unicode61-default` | `unicode61` | none |
| 2 | `unicode61-rd0` | `unicode61 remove_diacritics 0` | none |
| 3 | `unicode61-rd2` | `unicode61 remove_diacritics 2` | none |
| 4 | `unicode61-tokenchars-underscore` | `unicode61 tokenchars '_'` | none |
| 5 | `unicode61-separators-middot` | `unicode61 separators '·'` | none |
| 6 | `ascii` | `ascii` | none |
| 7 | `porter` | `porter` | none |
| 8 | `trigram` | `trigram` | none |
| 9 | `trigram+like-fallback` | `trigram` | none; `LIKE` when the term is < 3 chars |
| 10 | `unicode61+bigram-shadow` | `unicode61` | overlapping CJK bigrams indexed |
| 11 | `unicode61+unigram-bigram-shadow` | `unicode61` | CJK unigrams **and** bigrams indexed |
| 12 | `unicode61-like-substring` | `unicode61` | none; all text queries via `LIKE '%q%'` (index unused) |

---

## 6. What `unicode61` actually does — the mechanism, measured

### 6.1 Token dumps (from `fts5vocab`, not from any score)

Input string `记忆库的中文测试内容` (10 CJK characters):

| config | tokens |
|---|---|
| `unicode61` | {{TOK_U61}} |
| `trigram` | {{TOK_TRI}} |

Input string `混合mixed文本`:

| config | tokens |
|---|---|
| `unicode61` | {{TOK_U61_MIX}} |
| `ascii` | {{TOK_ASC_MIX}} |

Input string `苹果公司下周发布新款 MacBook，价格未知。` (the case that separates `ascii` from `unicode61`):

| config | tokens |
|---|---|
| `unicode61` | {{TOK_U61_GLUE}} |
| `ascii` | {{TOK_ASC_GLUE}} |
| `trigram` (all grams) | {{TOK_TRI_GLUE}} |

So under `unicode61` the whole run is **one** token. `中文` cannot match it, `中文测试内容` can.
That is exactly the distinction the brief asks for, and it is a **segmentation** failure, not a
tokenizer crash: **15 of 19 Chinese word/character queries do return at least one candidate.**
The `ascii` row is the other half of the story: it treats high-bit bytes as token characters, so
`MacBook` is glued to the following CJK punctuation and text and becomes unsearchable — `ascii` is
not a fallback for mixed Chinese text.

### 6.2 Model fit — a falsifiable statement, not an assertion

For every single-token query ({{MF_N}} of them) the observed hit set was compared with the set
predicted by two candidate mechanisms, both computed from corpus text only:

| prediction | fits | rate |
|---|---|---|
| run-equality: a document matches iff one field contains a token-run *equal* to the query | {{MF_RE_FITS}}/{{MF_N}} | **{{MF_RE}}** |
| literal substring: a document matches iff one field *contains* the query | {{MF_SUB_FITS}}/{{MF_N}} | {{MF_SUB}} |

The single mismatch in the run-equality model is `Cafe` → `Café` (diacritic stripping, which the
plain model does not model). So `unicode61` here is *run-equality retrieval*, and its "hits" are
run coincidences, not word retrieval. `trigram` fits the substring model at {{MF_TG}} (with its
3-character floor counted as "predicts nothing"), `LIKE` at {{MF_LK}}, and the shadow-bigram options
at {{MF_BG}} / {{MF_UB}}.

### 6.3 Which Chinese queries `unicode61` misses, and why

Nine scored queries returned nothing under `unicode61`:

{{U61_ZEROBULLETS}}

Sample causal chain, query `咖啡` (3 relevant documents, all of which literally contain 咖啡):

| document | stored text contains | `unicode61` token | result |
|---|---|---|---|
| d02 | `美式咖啡` | `我更喜欢喝美式咖啡` | miss |
| d08 | `咖啡豆` | `咖啡豆` | miss |
| d36 | `咖啡店面试` | `咖啡店面试了候选人` | miss |
| d42 | `记忆 库 中文 测试 检索 分词` (space-separated) | `咖啡` not present | miss |

Every miss is "the word was inside a longer run". Every hit in the whole Chinese set is "the query
was an entire run". That single rule explains all {{U61_FN}} false negatives without exception. The
one false positive is q46 (`"记忆 库"` also matching d12's `记忆<U+200B>库`, because `unicode61`
treats U+200B as a separator) — see DEF-01.

---

## 7. Results — per-query expected-vs-actual

Marker legend: `returned/|Rel|`, then `✓` = exactly the relevant set, `·` = partial retrieval or extra
rows (`+nFP`), `✗` = none of the relevant documents retrieved, `0` = labelled must-not-match.
`u61`=`unicode61-default`, `tri`=`trigram`, `tri+LIKE`=`trigram+like-fallback`,
`bigram`=`unicode61+bigram-shadow`, `uni+bi`=`unicode61+unigram-bigram-shadow`, `LIKE`=`unicode61-like-substring`.

{{T1}}

Reading the table by class:

* **pure-`unicode61` collisions** (q02 `记忆库`, q03 `中文测试内容`, q09 `鸡蛋`, q54 `分词`) are hits only
  because the field happens to hold exactly that run — e.g. q02 hits through the `tags` field, not
  through the sentence text.
* **q52** is the brief's "longer query hits" case: the whole unpunctuated sentence is one run, so the
  query matches it and nothing else. It is a coincidence of the same failure mode, not evidence of
  Chinese search working.
* **q27 `2026`** shows the failure is not Chinese-only: `2026-12-15` tokenizes to `2026`, but
  `20261105` is one token, so the same query misses d06 under `unicode61` and every shadow option
  (only `trigram`/`LIKE`, i.e. genuine substring engines, find it).
* **q30 `繁体`** (simplified form of a word stored only in traditional) correctly returns nothing
  everywhere: FTS5 has no simplified/traditional folding. That is a documented gap, not a bug.
* **q33 `🍰`** and **q45/q47** fail for every index-based option: an emoji is not a token character,
  and a phrase over a contiguous run needs the run to be split.

## 8. Results — aggregate comparison

{{T2}}

Definitions used (so the numbers cannot be over-read): `hit@10` = fraction of the 45 scored queries
with ≥1 relevant document in the top 10; `full-recall` = fraction where **all** relevant documents
were retrieved from the top-50 channel window; `recall@K = |Rel ∩ TopK| / |Rel|`; `precision@K =
|Rel ∩ TopK| / |TopK|` over the returned set; FP/FN are summed over all 55 queries; `p50/p95` are
per-execution latencies over 5 timed repetitions × 55 queries on the 46-document index (fixed
overhead dominates there — the scale table in §10 is the meaningful latency measurement).

Per query class:

{{T3}}

The two numbers that decide the question are in the `chinese_substring` and `chinese_single_char`
rows: under `unicode61` they are **{{U61_ZH_HIT}} / {{U61_ZH_REC}}** and **{{U61_SC_HIT}} / {{U61_SC_REC}}**
(hit@10 / recall@10). Note that English, digits, abbreviations and mixed text are **already 1.0** on
`unicode61` — which is why "English FTS works" was never evidence about Chinese, and why any fix must
be additive rather than a replacement.

`ascii` deserves one line of explanation: it is *worse* than `unicode61` for mixed text
({{ASC_HIT}} hit@10) because in this build it treats high-bit bytes as token characters, so
`MacBook，价格未知。` becomes a single token. It is not a Chinese candidate.

## 9. Tokenizer-specific findings (each is a real limitation, not a harness bug)

### 9.1 `trigram` and the 3-character floor

`trigram` requires at least 3 characters. **It does not raise an error** — it silently returns zero
rows, which is the dangerous form of the limitation. Exact failing queries in this dataset (all
shorter than 3 characters, all returning `0/|Rel|`):

`中文`, `旅行`, `项目`, `咖啡`, `苹果`, `鸡蛋`, `测试`, `检索`, `西`, `我`, `的`, `爱`, `库`, `繁體`,
`记忆`, `分词`, `蛋糕` (17 queries), plus the two prefix queries `项目*` / `检索*` (a 2-character
prefix cannot match a 3-character index token). 25 scored queries returned nothing in total,
including `Cafe`, the emoji and the two boolean expressions. Result: {{TG_HIT}} hit@10,
{{TG_R10}} recall@10 — worse than doing nothing, because it looks like it works for 3+ characters.

### 9.2 `trigram` + `LIKE` fallback closes the short-query hole but reintroduces a scan

`{{TL_HIT}}` hit@10 / `{{TL_R10}}` recall@10, precision {{TL_P10}}, index {{TG_BYTES_DOC}} B/doc. The cost is the
`LIKE` path for every term shorter than 3 characters — and it is *selectivity-dependent*: with
`LIMIT 50` a common term stops early, but a rare or absent term scans all 20 000 documents
(**{{TL_WORST}} ms** worst-query p50 vs {{U61_WORST}} ms for an indexed lookup).

### 9.3 `LIKE`-only substring mode

Highest lexical recall of anything measured ({{LK_HIT}} hit@10, {{LK_R10}} recall@10, precision {{LK_P10}}) with
**zero** extra index bytes — but it cannot express the query forms the algorithm requires: boolean
`AND`/`OR` degrade to a single term, phrases degrade, and above all **it produces no relevance
score**, so `ORDER BY rank` / bm25 field weights (`02-核心算法规范` §5.3) have nothing to order by and
`channel_score` feeding RRF would be meaningless. Its latency is also data-dependent
({{LK_P50_SCALE}} ms median but {{LK_WORST}} ms worst-query p50 at 20 000 docs, growing linearly).

### 9.4 `porter`, `remove_diacritics`, `tokenchars`, `separators`

`porter` behaves identically to `unicode61` on Chinese (`0.8` hit@10, same token dump) — it only adds
English stemming (`camelcase`→`camelcas`). The three `unicode61` tuning knobs change **nothing** for
Chinese: `remove_diacritics 0` costs exactly the one accent query (`Cafe`), `remove_diacritics 2`
matches the default, `tokenchars '_'` fixes `user_id`-style retrieval only via a second spelling
(`unicode61` already finds `user_id` through the `user` + `id` phrase), and `separators '·'` is a
no-op on this corpus because U+00B7 is already a separator. **There is no `unicode61` option that
splits CJK runs.** That is the decisive negative result: the defect is not tunable, it is structural.

---

## 10. Latency and index size (measured, both corpora)

**Labelled corpus (46 docs, 4 096-byte pages, WAL, after `wal_checkpoint(TRUNCATE)`):**

| config | fts index bytes | whole db file bytes | vs u61 | build ms | p50 ms | p95 ms |
|---|---|---|---|---|---|---|
| `unicode61-default` | {{U61_BYTES}} | {{U61_DBFILE}} | — | {{U61_BUILD}} | {{U61_P50}} | {{U61_P95}} |
| `trigram` | {{TG_BYTES}} | {{TG_DBFILE}} | +55.6% | {{TG_BUILD}} | {{TG_P50}} | {{TG_P95}} |
| `unicode61+bigram-shadow` | {{BG_BYTES}} | {{BG_DBFILE}} | +33.3% | {{BG_BUILD}} | {{BG_P50}} | {{BG_P95}} |
| `unicode61+unigram-bigram-shadow` | {{UB_BYTES}} | {{UB_DBFILE}} | +66.7% | {{UB_BUILD}} | {{UB_P50}} | {{UB_P95}} |

(The whole-db figure includes the identical `src` + `map` tables in every config, so the fts-only
column is the comparator. At 46 documents the fixed 4 096-byte page granularity dominates — the scale
corpus is the number to plan with.)

**Scale corpus — {{BULK_N}} documents, latency and size only:**

{{T4}}

Planning figures from the scale corpus:

| | `unicode61` | +unigram/bigram shadow | factor |
|---|---|---|---|
| index bytes per doc | {{U61_BYTES_DOC}} | {{UB_BYTES_DOC}} | **3.0×** |
| build time (20 k docs) | {{U61_BUILD_SCALE}} ms | {{UB_BUILD_SCALE}} ms | 3.2× |
| query p50 | {{U61_P50_SCALE}} ms | {{UB_P50_SCALE}} ms | 26× |
| worst-query p50 | {{U61_WORST}} ms | {{UB_WORST}} ms | 1.7× |
| `LIKE`-mode worst-query p50 | — | {{LK_WORST}} ms (full scan) | — |

The bigram/`LIKE` latencies are honest but need their shape explained: `unicode61`'s
{{U61_P50_SCALE}} ms median is largely *cheap misses* (a run-equality lookup that finds nothing),
which is why its accuracy is poor. The 3–4 ms worst case for the shadow options is a genuine
posting-list intersection over the most common bigrams (`中文`, `项目`) restricted to 50 rows; at
20 000 documents that is still two orders of magnitude inside `retrieval.timeout_ms = 3000`
(EXP-DEFAULT). The 3.0× index growth is the real cost, and it is content-dependent (it was +67% on
the 46-document corpus, where documents are short and terms repeat).

## 11. Behavioural checks (all executed this run)

| Check | Result |
|---|---|
| Tokenisation determinism: vocabulary sha256 before/after close+reopen | identical for all 12 configs (`vocab_stable = true`) |
| Result sets before/after reopen | 0 mismatches out of 55 queries × 12 configs |
| Same string inserted twice → same tokens | true for all configs (`fts5vocab` instance mode) |
| `INSERT INTO f(f) VALUES('integrity-check')` | `ok` for all 12 configs |
| Empty query `""` handed to the engine | **raises** `fts5: syntax error near ""` — the harness must short-circuit; it does, returning `[]` with no error |
| Whitespace-only `"   "` | same as above |
| Lone `"` | `unterminated string` (raised by the engine; harness catches) |
| Unbalanced `(中文` | `fts5: syntax error near ""` |
| Punctuation-only `、` / `——` | 0 rows under `unicode61`/`trigram`/`porter`/shadow options; **4 and 3 rows** under `ascii` and `LIKE`-modes (see DEF-02) |
| Escaping: user text containing `"`, `AND`, `OR`, `NOT`, `(`, `)` | unescaped it **is executed as operators** (1 row) — i.e. unescaped user input can inject query semantics; wrapped as an FTS5 string literal it becomes a literal phrase and does not raise |
| `LIKE` pattern escaping (`%`, `_`, `\`) | escaped correctly; probes returned 1/2/0 rows as expected |
| bm25 field weights (3.0/1.0/1.5/1.2) vs unweighted | both execute; ranking order changed for 2 of 6 probed queries (`测试`, `检索`) and was identical for 4. Ranking cannot rescue recall when the tokenizer returns zero candidates — which is the case for 9 of the Chinese queries |

Ranking output is retained in the raw JSON (`ranking_samples`): e.g. `项目` under
`unicode61+unigram-bigram-shadow` returns all 6 relevant documents ordered
`d35,d34,d33,d32,d31,d03`, while `unicode61` returns the same 6 in the order `d32,d31,d03,d33,d34,d35`
— both correct on this query, different because the shadow index scores the title column more
densely. No nDCG is reported: there are no graded judgements.

---

## 12. Dataset defects (recorded, not edited away)

**DEF-01 — `wordprefix` labeller and `unicode61` disagree about U+200B.**
The `wordprefix` rule used for the two prefix queries splits words on whitespace, CJK punctuation and
ASCII punctuation, and deliberately does **not** treat U+200B (ZWSP) as a delimiter, while `unicode61`
does treat it as a separator. No `rel` set is affected — q43/q44 target `项目`/`检索`, and d12 (the
only ZWSP document) contains neither — but the disagreement is visible elsewhere: d12 is exactly the
one extra document that q46 (`"记忆 库"`) pulls in under `unicode61`, i.e. `unicode61`'s single false
positive. The label rule is therefore not tokenizer-faithful in principle, and a prefix query aimed at
ZWSP-separated text would be mislabelled.

**DEF-02 — the "punctuation-only query" labels are engine-scoped.**
q37 (`、`) and q38 (`——`) were authored with `rel = []` on the rationale "punctuation carries no
tokens, so 0 hits is correct". That rationale holds for `unicode61`, `trigram`, `porter` and the
shadow options, but **not** for `ascii` (which makes punctuation a token: `——` matches d13) nor for
`LIKE`/substring modes (punctuation is ordinary text: `、` matches 4 documents). The expectation was
not edited — the observable consequence is that PC-3 shows 8/9 for `ascii` and 7/9 for the two
`LIKE`-based configs: {{PC3_DETAIL}}

Neither defect affects the Chinese-retrieval conclusion: both are excluded from the recall/precision
aggregates and are reported separately.

---

## 13. Can FTS freeze?

**No — not on the current tokenizer.** Concretely:

* **PC-1 → {{PC1}}** — (`T-M02-01` "中文精确词查询返回含该词的候选；中文分词失效（全 0 命中）判 FAIL"):
  {{PC1_DETAIL}}
* **PC-3 → {{PC3}}** — (all must-not-match queries return zero rows): explained by DEF-02, engine-scoped.
* **PC-4 → {{PC4}}** — (no unhandled query error, punctuation class excluded).
* **PC-5 → {{PC5}}** — (tokenisation stable across close/reopen).

Under the project's own acceptance criterion, `unicode61` **FAILS** the Chinese FTS requirement: 4 of
19 Chinese word/character queries return nothing at all, and the aggregate Chinese recall@10 is
{{U61_ZH_REC}}. Freezing now would mean writing `FROZEN` for a component that fails its own test
spec, so it must stay `PROPOSED`/`VERIFY`.

### The exact gate to lift the freeze

FTS (the retrieval channel only, not RRF, not LanceDB) may be frozen when **all** of the following
hold, measured by `node run-all.cjs` on this frozen dataset (`{{DATASET_SHA}}`) and this environment:

1. **G1** — `PC-1` PASS: every one of the 19 Chinese word/character queries returns ≥1 candidate, i.e.
   `zero_result_queries` contains no `chinese_substring` / `chinese_single_char` entry.
2. **G2** — `hit@10 ≥ 0.95` and `recall@10 ≥ 0.90` over the 45 scored queries (today: best option
   {{UB_HIT}} / {{UB_R10}}; baseline {{U61_HIT}} / {{U61_R10}}).
3. **G3** — no regression on the already-working classes: `english`, `digits`, `abbreviation`, `mixed`,
   `special_symbol` must stay at hit@10 = 1.0 (true for `unicode61` today).
4. **G4** — `precision@10 ≥ 0.95` and the 5 non-degenerate must-not-match queries (`q30`, `q35`,
   `q40`, `q41`, `q42`) still return zero rows (baseline `unicode61`: precision {{U61_P10}}, all five
   clean).
5. **G5** — index cost bounded: ≤ 3.5× the `unicode61` fts bytes per document at 20 000 documents,
   and worst-query p95 ≤ 5 ms at 20 000 documents on this machine.
6. **G6** — `retrieval.tokenizer_id` + version registered in `15-参数注册表.md` (today `TBD`,
   OI-02-04) and written to the retrieval trace, including the index-time transform version.
7. **G7** — the residual limits are documented in the design, not just in this report: emoji and
   non-token characters are not retrievable; substring retrieval inside long digit/Latin runs is not
   provided unless those runs are expanded too; prefix search degenerates to substring on a bigram
   index.

The candidate that meets G1–G5 today is option D below. It is **not** frozen by this report: G6 and
G7 are design/registry actions that have not been taken, and I did not write `FROZEN` for anything
outside what I measured.

---

## 14. Recommendation

### 14.1 Is `unicode61` adequate for Chinese here? **No.**

With hit@10 {{U61_HIT}} / recall@10 {{U61_R10}} overall and **{{U61_ZH_HIT}} / {{U61_ZH_REC}}** on
Chinese word/character queries, {{U61_ZERO}} of the 45 scored queries returning zero rows, and a
run-equality mechanism that can be defeated by one extra character in the sentence (`咖啡` misses
`美式咖啡`, `记忆库` misses `记忆库的中文存储`), it is not usable as the Chinese retrieval channel.
Crucially, **no `unicode61` option fixes it**: `remove_diacritics`, `tokenchars` and `separators` are
all no-ops for CJK segmentation (§9.4). The failure is structural, and it is invisible in English
tests, which is why it survived this long.

### 14.2 Options and their cost

| option | dependency / Windows | install complexity | index B/doc @20 k | accuracy hit@10 / R@10 | verdict |
|---|---|---|---|---|---|
| A. `unicode61` (status quo) | none | none | {{U61_BYTES_DOC}} | {{U61_HIT}} / {{U61_R10}} | fails `T-M02-01` |
| B. `unicode61` tuned (rd0/rd2/tokenchars/separators) | none | none | {{U61_BYTES_DOC}} (all four variants within one page) | {{U61_HIT}} / {{U61_R10}} (rd0 0.7778/0.6453) | **no effect on CJK** — rejected |
| C. `trigram` | none | none | {{TG_BYTES_DOC}} | {{TG_HIT}} / {{TG_R10}} | rejected: silently empty for <3 chars |
| C′. `trigram` + `LIKE` fallback | none | query-path branch | {{TG_BYTES_DOC}} | {{TL_HIT}} / {{TL_R10}} | works, but every 2-char query (the common Chinese case) does an O(N) scan: {{TL_WORST}} ms at 20 k |
| **D. `unicode61` + unigram/bigram shadow** | **none (≈80 lines of JS)** | index-time transform + query rewrite, no schema change | **{{UB_BYTES_DOC}} (3.0×)** | **{{UB_HIT}} / {{UB_R10}}** | **recommended** |
| D′. `unicode61` + bigram only | none | same | {{BG_BYTES_DOC}} (1.8×) | {{BG_HIT}} / {{BG_R10}} | cheaper, but 1-char queries stay broken |
| E. `LIKE '%q%'` substring | none | none | {{LK_BYTES_DOC}} (index unused) | {{LK_HIT}} / {{LK_R10}} | rejected: no bm25 score → breaks ALG-02 §5.3 and RRF `channel_score`; latency grows linearly ({{LK_WORST}} ms worst) |
| F. third-party tokenizer extension | compiled win32-x64 DLL | high (build or ship binary) | n/a | not measured | blocked today, viable later — the loading API exists (§2.1) |

Option F, honestly stated: it is the textbook answer and `node:sqlite` can load it. What is missing is
the binary and the toolchain to build it. It would need a pinned prebuilt DLL per platform (Windows
first), a check that the SQLite ABI matches 3.53.4, an install step in deployment, and a rebuild of
every index — plus a fallback path for hosts without the DLL. That is a larger change than the whole
retrieval-channel fix in option D, and it cannot be verified in this environment.

### 14.3 Recommended: option D — `unicode61` plus an indexed unigram+bigram shadow of the CJK text

Mechanism: at index time each field is expanded — every maximal CJK run of length *n* contributes its
*n* unigrams and *n−1* overlapping bigrams as space-separated tokens; non-CJK segments pass through
untouched, so **English, digits, identifiers and mixed text keep their current behaviour byte for
byte**. At query time a CJK term of ≥2 characters becomes a **phrase of its bigrams**, a 1-character
term stays a single token, and whitespace-separated terms are joined with `AND`.

Why this and not the others:

* **Recall**: {{UB_HIT}} hit@10 / {{UB_R10}} recall@10 vs {{U61_HIT}} / {{U61_R10}} — and **exactly one**
  scored query still returns nothing (`🍰`, an emoji: not a token character in any FTS5 tokenizer).
* **Precision is not traded away**: {{UB_P10}} precision@10 vs {{U61_P10}}; the {{UB_FP}} false
  positives are entirely `检索*` (3, prefix degeneration) and `"记忆 库"` (2, phrase collapse), both
  listed as known failure modes below.
* **No dependency, no extension, no platform risk**: pure JavaScript plus the FTS5 that is already
  compiled into `node:sqlite`.
* **No architectural change**, which was the constraint: it stays inside ALG-02's S5 "生成 FTS5 查询
  表达式" step (a rewrite function, ~80 lines, deterministic and versionable) and inside the existing
  index-write path (one derived text per field). `bm25()` with the field weights 3.0/1.0/1.5/1.2,
  `top_k = 50`, `ORDER BY rank`, RRF, authority read-back and permission checks are untouched — the
  weights keep working **because the four column names are unchanged**, which the single-table shadow
  design preserves and a second FTS table would not.
* **Deterministic and inspectable**: vocabulary hash stable across reopen, identical results across
  reopen, `integrity-check` ok (all 12 configs).

Measured costs of option D, as required:

| cost axis | measured / expected |
|---|---|
| index size | {{UB_BYTES_DOC}} B/doc at 20 k docs = **3.0×** `unicode61`; +66.7% on the 46-doc corpus (content-dependent; grows with CJK density, shrinks with term repetition) |
| build time | {{UB_BUILD_SCALE}} ms vs {{U61_BUILD_SCALE}} ms for 20 k docs (3.2×) — a one-off per rebuild, inside RC-03 |
| query latency | p50 {{UB_P50_SCALE}} ms, worst-query p50 {{UB_WORST}} ms at 20 k docs — 3 orders of magnitude inside `retrieval.timeout_ms = 3000` |
| write-path cost | one extra string transform per field per document, O(len); no extra I/O, no extra table |
| migration | full FTS rebuild (`RC-03` already requires rebuild-from-SQLite, so the mechanism exists); no change to authority tables, no data migration, no version bump of stored documents |
| new failure modes | (1) **prefix search degenerates to substring** on the bigram stream — `检索*` returned 6 documents instead of 3 (`+3FP`); (2) **multi-token phrase `"记忆 库"` collapses to `"记忆库"`** in the current rewriter, so it returns `d40,d41` (the contiguous spelling) and misses `d42` (the split spelling): 1 FN + 2 FP; (3) substring retrieval inside long **digit/Latin** runs is still not provided (`2026` still misses `20261105`) because only CJK runs are expanded; (4) 1-character CJK queries match anywhere rather than as words, so their precision is inherently substring-level |
| maintenance | the transform is a versioned constant: register `retrieval.tokenizer_id = unicode61+unigram-bigram/v1` and log it (the design already requires "分词方案 MUST 可替换且版本入日志"). Changing the expansion later means a rebuild — the same operation as changing the tokenizer itself |

The residual defects listed above have known fixes if the product needs them: (1) keep a small
un-expanded column set (or query the bigram stream with a left-anchored bigram sequence) for prefix
search; (2) rewrite `"A B"` into a bigram phrase over `A+B` **without** collapsing when a space was
present, or keep a second phrase-friendly column; (3) expand long ASCII/digit runs into trigrams if
substring retrieval on numbers is required. None of these is necessary to pass the gate in §13.

### 14.4 Does the retrieval architecture need to change?

**No.** The fix stays inside the tokenizer/query-expression layer:

* index write: one extra derived text per field (`expandText`), same table, same columns, same weights;
* query: one rewrite function at S5 of the existing seven-step normalisation, replacing the current
  placeholder "分词 + MATCH 语法 + 字段权重";
* everything downstream — `bm25` ranking, top-50, `channel_rank`/`channel_score`, RRF k=60,
  authority read-back, permission final check, token budget — is unchanged and was not touched.

If the team prefers to defer any index-format change, the honest interim position is: keep
`unicode61`, mark the FTS channel `degraded=true` for CJK queries, and **do not** claim
`V-02 FTS5 中文检索可用`. That is a documentation change, not a passing test.

---

## 15. Limitations (mandatory)

1. **Labels are lexical, not semantic.** 46 documents, 55 queries, no human relevance judgements, no
   graded judgements. nDCG, semantic recall and human-judged precision are **not computable**; the
   reported recall/precision describe whether a string that is literally in a document can be found.
2. **Small corpus.** FP/FN are small integers (1/47, 5/3, …), not stable rates. `recall@K` is capped by
   construction wherever `|Rel| > K` (q10 has 10 relevant documents, q14 has 11).
3. **The scale corpus is synthetic and unlabelled.** No accuracy figure is derived from it; it exists
   only for index size and latency. Its documents are concatenations of the template sentences, so its
   term-frequency distribution is not a real memory store's.
4. **Latency method**: single process, single connection, WAL, warm page cache, statements prepared
   before timing (planning excluded), 5 repetitions, no concurrency, no RRF, no authority read-back, no
   permission check. Numbers are comparative, not capacity-planning numbers.
5. **Index size method**: `dbstat` after `wal_checkpoint(TRUNCATE)`; the whole-db figure includes the
   `src`/`map` tables present in every config, so only the fts-only column is a like-for-like delta.
6. **Tokenizer coverage**: only the four tokenizers FTS5 ships. `trigram case_sensitive`,
   `unicode61 categories`, and `tokenchars` beyond `_` were not explored; ICU is unavailable.
7. **`LIKE`-mode results are degraded by design** in this harness: a `LIKE` plan cannot express
   `AND`/`OR`, so those two queries fall back to their first term. This makes option E's accuracy
   figures a *lower* bound on pure substring recall and an *upper* bound on its query-expressiveness
   problem — both directions are stated rather than hidden.
8. **Provenance conflict carried from the parent**: the prior artefact
   `evidence/second-round/sqlite-file-result.json` records `node v24.22.1`; every number here was
   produced on **v24.21.0**. Its `fts_chinese_unicode61: "NO_HIT"` is *reproduced and refined*: not
   "no hit at all" but run-equality retrieval, with 15/19 Chinese queries returning something.
9. **Not verified**: concurrent writers, crash recovery of the FTS index, index rebuild after delete
   propagation (tombstones), behaviour beyond 20 000 documents, and the SQLite-ABI compatibility of any
   future extension.

---

## 16. Evidence and files

Every executed check produced one record in `06-evidence/fts.jsonl` (71 records, `run_id
{{RUN_ID}}`): 56 `PASS`, 13 `PARTIAL_PASS` (per-config accuracy where some queries hit and others did
not — the record states which), **2 `FAIL`** (PC-1 and PC-3, the product criteria). Raw output:
`{{RUN_RAW}}` and `.log`. Throwaway databases live in `_scratch/` and are rebuilt on every run.

| artefact | sha256 |
|---|---|
| `_data/dataset.json` | `{{DATASET_SHA}}` |
| labels `[[query_id, rel[]]]` | `10e92de851ec4c9443076f464900ffac066728015a3d20a39690e8bf24cb306e` |
| `_data/bulk-corpus.json` (20 k synthetic) | `{{BULK_SHA}}` |

Evidence ids by area: `fts.build.*` and `fts.accuracy.*` / `fts.consistency.*` / `fts.size_latency.*`
per config (12 × 4), `fts.scale.*` per config at 20 k docs, then `fts.empty_query`,
`fts.query_escaping`, `fts.zero_token_queries`, `fts.bm25_field_weights`, `fts.model_fit`,
`fts.extension_path`, and `fts.product_criterion.PC-1…PC-5`.

## 17. Open items handed back to the parent

1. Register `retrieval.tokenizer_id` + version (`OI-02-04` is still `TBD`) — required by G6 and by
   ALG-02 §13/§14 before any freeze.
2. Decide whether the FTS channel is allowed to carry a derived/expanded text at write time. Option D
   needs one extra string per field per document in the index write path (no new storage in the
   authority table).
3. The punctuation-label conflict (DEF-02) and the ZWSP delimiter conflict (DEF-01) should be fixed in
   whatever corpus becomes the permanent retrieval regression set; they were left unfixed here on
   purpose.
4. `fts.jsonl` is append-only, so a second `run-all.cjs` invocation appends a second record set; each
   record carries its own `run_id` and `raw_output_path`. If the evidence file must hold exactly one
   set, delete it before re-running.
5. Option F (prebuilt tokenizer DLL) is worth a separate spike if the product later needs
   word-level segmentation rather than n-gram substring matching; it is not needed to pass the gate.
