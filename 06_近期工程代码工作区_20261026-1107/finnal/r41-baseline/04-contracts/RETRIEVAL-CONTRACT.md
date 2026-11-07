# RETRIEVAL-CONTRACT — Memory Core R4.1 (retrieval chain)

| Field | Value |
|---|---|
| Status | **PROPOSED** — verification contract, not an approved design decision |
| Author | Agent F (retrieval) |
| Date | 2026-11-05 |
| Product authorization | **NOT_AUTHORIZED** — this file authorises no M01/M02/M03/M04/M05 implementation |
| Code that backs it | `03-verification/retrieval/rrf.cjs`, `dedup.cjs`, `retrieval-pipeline-sim.cjs` |
| Tests that back it | `rrf-tests.cjs`, `dedup-tests.cjs`, `retrieval-pipeline-sim.cjs`; evidence `06-evidence/retrieval.jsonl` |
| Re-run | `node 03-verification/retrieval/run-all.cjs` from the area directory |

Every claim below is marked with how it is established:

- **[DOC]** quoted from a design document (file + section given);
- **[TEST]** backed by an executed check (test id given, reproducible);
- **[PROPOSED]** my contract decision where the documents are silent or disagree — needs adjudication;
- **[VALIDATION_PENDING]** a number or threshold that no experiment on this machine has established.

This contract describes an **in-memory simulation**. FTS5, LanceDB, SQLite, the real tokenizer and
the M05 permission engine were **not** exercised here; the simulation substitutes documented
stand-ins (see §10). Nothing in this file may be cited as evidence that the real components behave
this way.

---

## 1. Stage list and ordering

The chain is fixed and ordered. Order is an invariant, not a preference: ranking before permission
leaks the existence of forbidden data through ordering; permission before filtering wastes work but
is not unsafe.

| # | Stage | Test |
|---|---|---|
| 1 | Query normalization | `pipeline.stage_order_and_trace`, `pipeline.query_preserves_raw_and_negation` |
| 2 | Router (intent / anchor) | `pipeline.stage_order_and_trace` |
| 3 | Structured Filter (hard constraints) | `pipeline.structured_filter_adds_no_score` |
| 4 | FTS channel | `pipeline.fts_down_is_partial_not_silent` |
| 5 | Vector channel | `pipeline.vector_channel_never_crosses_spaces` |
| 6 | RRF fusion | §4 below, `rrf.*` |
| 7 | Merge (authoritative回源) | `pipeline.stage_order_and_trace` |
| 8 | Dedup within results | `pipeline.dedup_exact_drops_near_keeps` |
| 9 | Permission (M05 final check) | `pipeline.permission_default_deny`, `pipeline.permission_error_fail_closed` |
| 10 | Ranking (six features) | `pipeline.stage_order_and_trace` |
| 11 | Token budget | `pipeline.budget_is_labelled_placeholder` |
| 12 | Context assembly | `pipeline.assembly_traceable` |

**[TEST]** The simulation records exactly these twelve stage names in this order
(`pipeline.stage_order_and_trace`).

**Ordering invariant INV-R-01.** Permission (stage 9) MUST precede ranking (stage 10) and assembly
(12). [DOC] 02 §7.5 precondition: "候选均已通过 M05 终检（MUST 先判权再排序，避免对无权数据排序泄露）".

---

## 2. Per-stage contract

Each stage lists its invariant, what it must not do, and its failure mode. "MUST NOT" items are the
ones a reviewer should check first, because each of them is a plausible implementation shortcut.

### 2.1 Query normalization

- **Input** `current_message` (raw). **Output** `{raw_query, normalized_query, terms, structured
  filters}`.
- **Invariants.**
  - `raw_query` is preserved byte-for-byte for audit and replay [DOC] `retrieval-response.schema.json`
    `query_features.raw_query`: "原始查询，审计与复现 MUST 保留". [TEST]
    `pipeline.query_preserves_raw_and_negation`.
  - Negation words (不/没/未/无/非) survive normalization. [TEST] same check.
  - Zero-width and control characters are stripped before tokenisation. [TEST] same check.
- **MUST NOT** rewrite the query with an LLM, translate it, or drop negation to "improve recall".
- **MUST NOT** infer structured filters that the request did not carry; in particular it may not
  widen `scope`. [DOC] `retrieval-request.schema.json`: "结构化过滤不能替代 M05 权限判断".

### 2.2 Router

- **Output** intent (one of the nine V2 intents) plus the set of channels to run.
- **Invariants.** The router selects *channels*, never *scopes*: it may not grant access to a scope
  the request did not ask for. [TEST] the trace records `scope_granted: false`.
- **MUST NOT** become a second permission system. [DOC] 04 §7.11: search/vector index metadata
  "MUST NOT 作为授权依据…只作检索前过滤优化".
- **Failure mode** unknown intent → fall back to `GENERAL_MEMORY_QUERY` with both channels; never to
  "no channels" (an empty recall is indistinguishable from "nothing is remembered").

### 2.3 Structured filter

- **Input** candidates with authoritative lifecycle/availability/scope/type/time. **Output** a kept
  set plus a per-item exclusion reason.
- **Invariants.**
  - Hard exclusions are authoritative, not index-derived: `lifecycle_state=deleted`,
    any non-`active` lifecycle, and `availability_state=suspended` are removed **before** any channel
    runs. [DOC] 01 §7.18 C-13 / SM-7 "立即生效：检索侧权威排除". [TEST]
    `pipeline.deleted_never_returned`.
  - The filter contributes **no score**. [TEST] `pipeline.structured_filter_adds_no_score`: a
    non-restrictive `type_filter` leaves every `rrf_score` and `source_channels` entry bit-identical
    to a run with no filter at all.
- **Nuance that must not be glossed over.** ALG-04's feature **N** (类型匹配, γ=0.10) *does* read the
  same `type_filter` and therefore *does* change `final_score` when a filter is present. That is a
  ranking feature, not a fusion channel; the test reports both facts
  (`finalScoreUsesN=true`) rather than pretending the filtered and unfiltered runs are identical.
  See **OI-R-08** for the reading question this raises.
- **MUST NOT** treat a scope filter as a permission decision, and MUST NOT return a "redacted" item
  for a filtered-out memory (no placeholders — an attacker learns existence from a placeholder).

### 2.4 FTS channel

- **Output** `[{memory_id, rank, score, index_version}]`, `rank` starting at 1, unique per channel.
- **Invariants.** `bm25` scores are comparable only *inside* this channel. [DOC]
  `retrieval-response.schema.json` `source_scores.bm25`: "仅 FTS 通道内部使用，禁止与向量相似度直接
  比较". [TEST] structurally enforced: `rrf.cjs` never reads `channel_score` when scoring.
- **Failure mode** channel error → `channel_status.fts = "failed"`, `retrieval_status = "partial"`,
  and the failure is named in `errors`. [TEST] `pipeline.fts_down_is_partial_not_silent`.
- **MUST NOT** silently return an empty FTS result set as if the index had no matches.

### 2.5 Vector channel

- **Output** `[{memory_id, rank, score, index_version}]` from **one** embedding space.
- **Invariants.**
  - Only vectors whose `embedding_space_id` equals the query space are compared. [DOC]
    `embedding-space.schema.json`: "同维度不代表同语义空间；不同空间禁止混合检索"; SM-8 禁令 1.
    [TEST] `pipeline.vector_channel_never_crosses_spaces` — a document with *identical text* in
    another space is never returned by the vector channel.
  - A query embedded in space A compared against a space-B vector **throws**
    `SPACE_MISMATCH`; it does not return a number. [TEST] `pipeline.space_mismatch_throws`.
- **Failure mode** space incomplete → `SPACE_INCOMPLETE`; space retired → `SPACE_RETIRED`;
  channel down → `partial`. [TEST] `pipeline.incomplete_space_unusable`.
- **MUST NOT** pad, truncate or re-normalise a vector to make it fit another space. [DOC] 07 §7.5
  "维度不匹配 → 拒绝，不得截断/补零".

### 2.6 RRF — see §4.

### 2.7 Merge

- Content returned to the caller comes from the **authoritative** record, never from an index
  payload. [DOC] TC-M02-009 "回源覆盖". [TEST] the simulation resolves every item from the corpus
  record, not from the channel row.
- Channel evidence is preserved through the merge: each surviving candidate keeps
  `channel_evidence[]` with `channel`, `channel_rank`, `channel_score`, `index_version`. [DOC]
  `retrieval-response.schema.json`: "禁止在去重时丢失通道排名".
- **MUST NOT** drop a candidate merely because one channel missed it (§4.4).

### 2.8 Dedup within results

- **Policy.** Only **exact duplicates** by normalized content are collapsed, keeping the
  higher-ranked one; near duplicates are kept and penalised later by ranking feature P. [TEST]
  `pipeline.dedup_exact_drops_near_keeps` (m03/m05 collapse, m03/m04 both survive).
- **Invariants.** Collapsing must not lose channel evidence of the survivor (the survivor keeps its
  own evidence) and must be reported (`dropped_duplicates[]`).
- **MUST NOT** merge or delete anything in the store here: retrieval dedup is not M04 consolidation.
  [DOC] 02 §7.9 is FUTURE and runs through M01.

### 2.9 Permission (M05 final check)

- **Invariants.**
  - Default **deny**. [DOC] 04 §7.11 "无法计算安全权限交集 → 默认拒绝"; 02 §7.10 `allow_all=false`.
    [TEST] `pipeline.permission_default_deny`.
  - Error, timeout, or an unknown/absent decision is **excluded**, never allowed. [DOC] 04 §10 table
    rows 7–9. [TEST] `pipeline.permission_error_fail_closed`.
  - Every surviving item carries a `decision_id`. [DOC] 02 §7.7 §10. [TEST]
    `pipeline.assembly_traceable`.
  - A denied item is absent, not redacted, and its content never appears in the response or in an
    error message.
- **MUST NOT** reuse index metadata as authorisation, and MUST NOT let `retrieval_status=partial`
  turn into "allow what we could check".

### 2.10 Ranking

- Six features `R,E,N,T,S,P` with α=0.55, β=0.15, γ=0.10, δ=0.10, ε=0.10, ζ=0.10 — **[DOC]** 02
  §7.5, all EXP-DEFAULT, none approved.
- **Invariants.**
  - Positive features and the penalty are normalised **separately**; the penalty is subtracted after
    the positive normalisation. [DOC] 02 §7.5 归一化总则.
  - Missing feature → `available=false` and the weight is redistributed
    (`Score⁺ = Σ_{i∈A} w_i f_i / Σ_{i∈A} w_i`), never imputed as 0. [DOC] same.
  - `importance` is a filter and a tie-break only; it never enters the score. [DOC] 02 §7.5 §3.5.
    [TEST] `rankCandidates` implements the tie-break order `final_score → importance → observed_at →
    memory_id` and no `+ω·importance` term exists (code inspection of
    `retrieval-pipeline-sim.cjs::rankCandidates`).
  - `final_score` is clamped to [0,1] and never NaN. [DOC] same.
- **MUST NOT** use `last_retrieved` or retrieval frequency as a positive weight. [DOC] 02 §7.14 红线.
- **MUST NOT** apply any emotion/psychology bonus to ranking. [DOC] 02 §7.14.3 (6).

### 2.11 Token budget

**This is the stage where numbers are most likely to be invented, so the rule is absolute.**

- No tokenizer for the target model exists on this machine. Therefore:
  - `token_usage.injected_content_tokens`, `memory_retrieval_tokens`, `module_calls_tokens`,
    `system_prompt_tokens`, `chat_history_tokens` are all **`null`** — not estimated, not zero-filled.
  - `token_accounting = "UNAVAILABLE"`, `estimated = true`, `proxy = "character_count"`.
  - `budget_tokens = 1200` is a **placeholder copied from the documents** (02 §7.6 EXP-DEFAULT,
    `retrieval-request.schema.json` default), not a measurement.
  - `threshold_state = "VALIDATION_PENDING"`: the truncation threshold used here (characters) has no
    empirical basis. [TEST] `pipeline.budget_is_labelled_placeholder`.
- Truncation policy implemented: greedy in rank order; an item that would exceed the budget is
  skipped, and later smaller items may still fit; `budget_status ∈ {ok, truncated, oversize}`.
  [PROPOSED] — the documents specify a budget but not the skip-vs-stop rule (**OI-R-09**).
- **MUST NOT** report a character count as a token count, and MUST NOT claim the budget was met in
  tokens.

### 2.12 Context assembly

- Every item carries `memory_id`, `version`, `zone`, `source_marker`, `time_marker`,
  `uncertainty_marker`, `decision_id`. [DOC] 02 §8.2. [TEST] `pipeline.assembly_traceable`.
- `uncertainty_marker` is non-null for anything not `validation_state=confirmed`; an inference is
  never presented as a statement. [DOC] 02 §7.7 §17.
- Zones partition the item set with no duplication, no gaps.
- **MUST NOT** invent filler when the result set is empty; an empty result is a legal answer.
  [DOC] TC-M02-012 "空结果合法 … 不注入凑数内容".

---

## 3. Invariant register

| ID | Invariant | Established by |
|---|---|---|
| INV-R-01 | Permission precedes ranking and assembly | `pipeline.stage_order_and_trace` |
| INV-R-02 | `raw_query` preserved; negation preserved | `pipeline.query_preserves_raw_and_negation` |
| INV-R-03 | Deleted / non-active / suspended are excluded before any channel runs | `pipeline.deleted_never_returned` |
| INV-R-04 | `structured` contributes no fused score | `rrf.structured_not_fused_proof`, `pipeline.structured_filter_adds_no_score` |
| INV-R-05 | Vector comparisons never cross spaces | `pipeline.vector_channel_never_crosses_spaces`, `rrf.*` (n/a), `pipeline.space_mismatch_throws` |
| INV-R-06 | A draining space takes no new writes | `pipeline.draining_space_not_writable` |
| INV-R-07 | Same input ⇒ identical output, including order | `rrf.deterministic_under_permutation`, `rrf.pure_no_input_mutation`, `pipeline.determinism_including_input_order` |
| INV-R-08 | Missing channel contributes exactly 0 | `rrf.single_channel_missing_is_zero` |
| INV-R-09 | No token count is ever claimed without a tokenizer | `pipeline.budget_is_labelled_placeholder` |
| INV-R-10 | Every injected item is traceable to `memory_id`+`version`+`decision_id` | `pipeline.assembly_traceable` |
| INV-R-11 | Permission defaults to deny; error/timeout ⇒ exclude | `pipeline.permission_default_deny`, `pipeline.permission_error_fail_closed` |
| INV-R-12 | One channel failing yields `partial`, never a silent empty result | `pipeline.fts_down_is_partial_not_silent` |
| INV-R-13 | Both channels failing yields `failed` and an empty set, not filler | `pipeline.both_channels_down_failed` |

---

## 4. RRF specification (normative for `rrf.cjs`)

`RRF(d) = Σ_{r ∈ R(d)} w_r / (k + rank_r(d))`

| Element | Decision | Basis |
|---|---|---|
| `k` | integer ≥ 1, default **60**, echoed in every result | [DOC] 02 §7.4, `retrieval.schema.json#/fusion/k` default 60, `recall.rrf_k` (EXP-DEFAULT) |
| `rank_start` | 1 | [DOC] `retrieval.schema.json`: "rank_start=1" |
| `w_r` | finite, `0 < w_r ≤ 2`; default 1.0 per channel | [DOC] 02 §7.4 "w_r ∈ (0, 2]" |
| weight normalisation | **explicit mode**, never inferred: `"raw"` (default) uses `w` as given; `"sum_to_one"` divides by the sum over the **enabled channels** | [PROPOSED] the documents never say whether weights are normalised; the published upper bound `Σw/(k+1)` implies raw. Mode is echoed as `weight_mode`, `weight_sum` |
| ties | (1) `rrf_score` DESC, exact float compare, no epsilon; (2) `observed_at` DESC, nulls last; (3) `memory_id` ASC by code unit | [DOC] 02 §7.4 step 6 (score→observed_at→memory_id); null handling is [PROPOSED] |
| truncation | `topK = null` ⇒ **no truncation** (full set to ALG-04); `topK ≥ 1` ⇒ keep first N, report `truncated` + `dropped[]` | [DOC] 02 §7.4 step 7 "输出全量候选集交 ALG-04". The *option* is [PROPOSED] |
| missing channel | contributes exactly **0** | [DOC] 02 §7.4 step 4; explicitly not "worst rank" |
| duplicate in one channel | merge, keep **min** rank, count `merged_duplicates` | [DOC] 02 §7.4 step 2 |
| duplicate **rank value** across two documents in one channel | **reject** with `VALIDATION_ERROR` | [DOC] 02 §7.4 precondition 4 ("channel_rank 从 1 起且无重复"). Note the document's own worked example violates this — see OI-R-10 |
| sum order | fixed channel order `fts, vector` regardless of input key order | [PROPOSED] determinism requirement; float addition is order-sensitive once a third term appears |
| channels | `fts`, `vector` only. `structured` **throws** `STRUCTURED_CHANNEL_NOT_FUSABLE` | §5 |

Hand-checked values (reproduced by `rrf.formula_matches_documented_scores`):

| candidate | ranks (fts, vector) | RRF, k=60, w=1 |
|---|---|---|
| d4 | 1, 1 | 1/61 + 1/61 = 0.0327868852 |
| d1 | 1, 3 | 1/61 + 1/63 = 0.0322664585 |
| d3 | —, 1 | 1/61 = 0.0163934426 |
| d2 | 2, — | 1/62 = 0.0161290323 |

`score_upper_bound = Σw/(k+1)` is computed and returned so a caller can assert the documented range
without re-deriving it.

---

## 5. `structured` does not participate in fusion — proof, not quotation

This is the one claim in this contract that had to be *constructed* rather than cited.

**Construction** (`rrf.structured_not_fused_proof`). Three candidates, k=60, w=1:

| candidate | fts rank | vector rank | structured rank | implemented RRF | hypothetical RRF *if* structured were fused |
|---|---|---:|---:|---|---|
| `d_struct` | 3 | 3 | **1** | 1/63 + 1/63 = 0.031746 | 1/63+1/63+1/61 = **0.048139** |
| `d_beat` | 1 | 1 | (not matched) | 1/61 + 1/61 = **0.032787** | 1/61+1/61 = 0.032787 |
| `d_other` | 2 | 2 | — | 0.031746 | 0.031746 |

- If `structured` were fused, `d_struct` would rank **first** (0.048139 > 0.032787).
- Under the implementation, `d_beat` ranks **first**
  (0.032787 > 0.031746), because `structured` is not a channel at all.
- [TEST] `rrf.structured_not_fused_proof` asserts **both** orders in one run: `implemented_order[0] =
  d_beat` **and** `hypothetical_fused_order[0] = d_struct`, using a local reference implementation of
  the wrong behaviour, so the difference is observed rather than argued.
- [TEST] `rrf.structured_channel_rejected`: passing `structured` as a channel, or supplying a weight
  for it, throws `STRUCTURED_CHANNEL_NOT_FUSABLE` instead of being silently ignored — a wrong wiring
  fails loudly.
- [TEST] every returned candidate's `channel_evidence` contains only `fts`/`vector`.

**Why it matters.** `structured` is a *constraint*: it removes candidates and can be a routing hint.
Fusing it would double-weight "the document matched the filter I asked for", which is a tautology —
every returned document already satisfies the filter.

**The documents disagree here.** Both readings, recorded rather than silently resolved:

| Reading | Text | Status |
|---|---|---|
| **A — not fused** (implemented) | `retrieval-response.schema.json` `source_channels` enum is `["fts","vector"]` with the note "structured 只过滤/路由，不作为候选来源通道"; `retrieval.schema.json#/fusion` description: "structured 只过滤不参与 RRF 计分（OI-022 结论）"; `15-参数注册表.md` `recall.rrf_k`; SRD-002; DC-08; `PENDING-USER-DECISIONS.md` OI-022 answer **A** | machine contracts + recorded adjudication |
| **B — fused** | 02 §7.4 ALG-03 input: `channel ∈ {fts5, vector, structured}`; §14 default `retrieval.rrf.weights.structured = 1.0` ("仅在有结构化通道时启用", EXP-DEFAULT); the §7.4 upper bound is written `Σ w_r/(k+1)` over all listed channels | design document, **not yet amended** |

OI-022 is recorded as DECIDED with the follow-up "02 删 structured w=1.0 行" still **open**. So this
is not an undecided disagreement; it is a **document that still contradicts its own adjudication**.
Nobody should treat 02 §7.4 as current until that line is removed — that is **OI-R-11**, and it needs
an owner, not a decision from me.

---

## 6. Deduplication, merge and conflict rules

Implemented in `dedup.cjs`, tested in `dedup-tests.cjs` (15 checks). Two normalisations are used and
deliberately kept distinct:

| Function | Used for | Rule |
|---|---|---|
| `exactKey()` | exact-duplicate test | whitespace collapsed only — matches 01 §7.17.1's conservative `normalized_content` ("仅空白与全半角") |
| `grams3()` / `jaccard()` | near-duplicate test | punctuation and whitespace removed, latin case-folded — matches 02 §3.4 ("3-gram 集合（去空白与标点）") |

Consequence, asserted as `dedup.near_duplicate_vs_exact`: `用户的血型是 O 型` vs `用户的血型是O型。`
are **not** exact duplicates (different `exactKey`) but **are** a Jaccard-1.0 near duplicate.

### 6.1 Classification and disposition

| Class | Condition | Disposition | Test |
|---|---|---|---|
| `exact_duplicate` | same `exactKey` + same type + same scope/scope_id + same subject + **same observed_at** | refuse the new row, merge evidence links into the survivor | `dedup.exact_duplicate_refused`, `dedup.exact_duplicate_merges_evidence_links` |
| `temporal_restatement` | same `exactKey` + same scope/subject, **different observed_at** | keep **both**; never merge across time; M04 decides version evolution vs coexistence | `dedup.same_fact_restated_different_time_not_merged` |
| `scope_variant` | same `exactKey`, different scope | not a duplicate (C-12 uniqueness is scoped) → new row | `dedup.different_scope_not_duplicate` |
| `semantic_duplicate` | Jaccard ≥ 0.9 and same type | merge **only if** all six preconditions hold | `dedup.semantic_merge_all_preconditions` |
| `partial_overlap` | 0.5 ≤ Jaccard < 0.9 | new row (+ relation) | — |
| `related` | Jaccard < 0.5 | new row | — |
| `numeric_conflict` | same attribute, different numbers | both rows kept, both `disputed`, **no silent overwrite** | `dedup.numeric_conflict_kept_both` |
| `state_conflict` | same attribute, mutually exclusive values | both rows kept, both `disputed` | `dedup.state_conflict_kept_both` |
| `subject_conflict` | different `subject_id` | merge forbidden unconditionally | `dedup.different_subject_refused` |

### 6.2 Merge: six preconditions, all required

[DOC] 02 §5.3: ① same subject ② scope compatible (stricter wins) ③ semantic relation explicit
④ source chain preserved ⑤ time relation explicit ⑥ no information loss. Each is returned
individually with pass/fail and a detail string so a refusal is explainable —
[TEST] `dedup.six_preconditions_enumerated`.

**Document tension (OI-R-04).** §5.3 ② says "Scope 兼容（取更严格）", which implies a *compatible*
wider/narrower pair may merge; §7.9 §10 says "跨 Scope 合并 MUST 拒绝；MUST NOT 扩大 Scope".
The implementation enforces the **stricter** reading (any scope difference blocks the merge) and the
test (`dedup.cross_scope_merge_refused`) says so explicitly. Neither reading is picked silently:
which one is intended changes whether a group-shared preference may absorb a private duplicate.

### 6.3 Things this module refuses to do

- It never deletes or downgrades an existing record. A "merge" updates the target and marks the
  absorbed record as merged-from; nothing disappears.
- It never merges on vector similarity alone. [DOC] 02 §5.1 "MUST NOT：仅凭向量相似度删除记忆".
  [TEST] `dedup.vector_similarity_never_deletes` forces cosine to **exactly 1.0** between two records
  with conflicting facts and shows both survive.
- It never merges two records that differ in `subject_id`, even with identical text. [DOC] 02 §5.2
  `subject_conflict`.

---

## 7. Embedding space isolation (summary; full contract in EMBEDDING-MIGRATION-CONTRACT.md)

- Space identity is the **eight** B-010 fields, not the dimension. [DOC] 07 §7.5; SM-8 禁令 1.
  [TEST] `pipeline.space_identity_is_eight_fields`: flipping only `normalized` changes the derived id
  at identical dimensions.
- `embedding_space_id` is derived, not hand-assigned:
  `"space-" + sha256(canonical_json(8 fields))[0:16]`. [PROPOSED, from 07 §7.5].
- A query is embedded **in one space** and may only be compared with vectors of that space.
- `space_lifecycle ∈ {active, draining, retired}` is **separate** from `migration_state`.
  `draining` accepts no new writes (**OI-R-07** — no document defines its semantics; this contract
  reads it as read-only) and `retired` serves nothing.
- The lexical channel is space-agnostic by construction (it indexes text, not vectors), so an
  FTS-only candidate may belong to another space. That is **not** a mixing violation — no vectors are
  compared — but the response must report each candidate's own `embedding_space_id`, and the test
  `pipeline.no_item_has_foreign_space_vector_evidence` pins the distinction.

---

## 8. Open items

Each item states **both** readings. None of them is resolved in this contract.

| ID | Item | Reading A | Reading B | Consequence if unresolved | Needs |
|---|---|---|---|---|---|
| **OI-R-01** | `pending_review → deleted` exists only in SM-1 (#3); 01 §7.7.2 has no such row | SM-1 is authoritative → transition legal (implemented) | 01 is the field authority → transition must be rejected | a rejected review could strand a candidate | design owner |
| **OI-R-02** | `archived → active` (restore) | 01 declares T-L-05 reversible "需审核恢复" and SM-1 #9 has the edge → legal (implemented) | 01's matrix has no row, so "not enumerated" ⇒ default deny | a user cannot un-archive, or can without review | design owner |
| **OI-R-03** | `rejected → unverified` ("revival") | 01 §7.7.1: `rejected` is terminal, revival on the same object is forbidden | SM-2: `rejected → unverified` is "唯一'复活'路径" | rejected memories either are permanently dead or become revivable | design owner |
| **OI-R-04** | Cross-scope merge | 02 §5.3 ② "scope compatible, take stricter" ⇒ allowed | 02 §7.9 §10 "跨 Scope 合并 MUST 拒绝" ⇒ never | scope creep vs lost dedup | design owner |
| **OI-R-05** | `suspended → available` | 01 §7.7.3 requires going through `restricted` (implemented) | SM-3 allows `suspended → available/restricted` directly | a suspension can be lifted without re-checking | design owner |
| **OI-R-06** | SM-1's table mixes validation states into the lifecycle machine (`active → disputed`, `disputed → active`) | one 6-state machine (SM-1) | two orthogonal axes (01 §7.7 / DD-AUTHORITY-V2 §2.6) — implemented | "was it rejected or was it archived?" becomes unanswerable | design owner |
| **OI-R-07** | Meaning of `space_lifecycle = "draining"` | read-only during drain (implemented) | follows from migration_state alone; `draining` is redundant | silent acceptance of writes into a dying space | design owner + user |
| **OI-R-08** | Does "structured only filters" forbid ALG-04's N feature from scoring on `type_filter`? | filtering and the N feature are separate, so scoring is fine (implemented) | "no structured contribution to ranking" should also exclude N | a type filter silently reorders results | design owner |
| **OI-R-09** | Budget truncation rule | greedy skip-and-continue (implemented) | stop at the first item that does not fit | tail items of a high rank can be dropped while low-rank short items survive | design owner |
| **OI-R-10** | 02 §7.4's worked example gives two documents FTS rank 1 | example is illustrative; reject duplicate ranks (implemented) | allow duplicate ranks in a channel | the document's own numbers cannot be produced by a valid channel output | doc owner |
| **OI-R-11** | 02 §7.4 still lists `structured` as an RRF channel and defaults `w_structured=1.0`, while OI-022/SRD-002 decided otherwise | remove the line (decided, not yet done) | keep fusing structured | two documents, two systems | doc owner |
| **OI-R-12** | `retrieval-request.schema.json` field names vs 02 §7.4's input table | schema is the machine contract (`current_message`, `requester_id`, `retrieval_options.token_budget`, `retrieval_options.fts_top_k/vector_top_k`) | 02 uses `query`, `actor`, `budget_tokens`, `top_k_per_channel`, and makes `subject_id` required | an implementation cannot satisfy both | design owner (Agent E's area) |

The remaining open items live in the other two contracts of this area:
**OI-R-13 … OI-R-17** (space-lifecycle naming, degradation field, writes during cutover, `migration_state` enum, missing tier transition table) are in
`EMBEDDING-MIGRATION-CONTRACT.md` §9 and `MEMORY-LIFECYCLE-MATRIX.md` §7.

**One check is a `PARTIAL_PASS`, not a pass.** `lifecycle.tier_promotion_gate_is_explicit` holds only because the test supplies `ctx.tier_review_approved`, a flag that appears **zero times** in the 24 design documents. The area headline is therefore **74 PASS, 1 PARTIAL_PASS, 0 FAIL** — the missing tier transition table (OI-R-17) is a design gap that no test can close, and it must not be read as a green result.

---

## 9. Failure and degradation matrix

| Condition | `channel_status` | `retrieval_status` | Items | Extra |
|---|---|---|---|---|
| all healthy | ok/ok | `complete` | as ranked | — |
| FTS fails, vector ok | `fts:failed` | `partial` | vector-only | failure named in `errors` |
| vector fails, FTS ok | `vector:failed` | `partial` | FTS-only | failure named |
| both fail | failed/failed | `failed` | **empty** | no filler, no fallback index |
| space incomplete/retired | `vector:failed` | `partial` (or `failed` if FTS also down) | FTS-only | `SPACE_INCOMPLETE` / `SPACE_RETIRED` |
| permission error/timeout | unchanged | `partial` | denied items excluded | fail-closed |
| budget exceeded | unchanged | unchanged | truncated | `budget_status=truncated`, character proxy |

[DOC] 02 §7.1 (6): "MUST NOT 静默降级：任何通道失败必须显式标记 `retrieval_status=partial|failed`".
[TEST] all six rows exercised in `retrieval-pipeline-sim.cjs`.

---

## 10. What was NOT exercised (limits of this contract's evidence)

| Claimed area | What was actually run | What was not |
|---|---|---|
| RRF | pure function, 15 checks, hand-computed values | no real FTS5 or LanceDB rank production |
| Dedup/merge | pure function, 15 checks | no FreeText NLP, no vector-dup threshold calibration (`vector_dup_threshold` is TBD in 02 §5.1) |
| Lifecycle | isolated in-memory validator, 23 checks | no SQLite, no transactions, no Outbox, no real M05, no audit ledger |
| Pipeline | in-memory simulation, 20 checks | no FTS5, no LanceDB, no real tokenizer, no real permission service, no latency claim |
| Space isolation | derived ids + channel filtering | no LanceDB table-per-space behaviour |

No latency, throughput or token figures are reported anywhere in this contract, because none was
measured.
