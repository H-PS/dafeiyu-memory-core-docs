# A — Memory Tier and Lifecycle (short-term / long-term)

Round: MC-CORE-CAPABILITY-REVIEW-01 · Read-only · 2026-11-06

## STATUS: COMPLETE

**Deliverable:** `E:\[PROJECT]\CORE-CAPABILITY-REVIEW-2026\01-MEMORY-TIER-AND-LIFECYCLE-MATRIX.md`
**Evidence:** `EVIDENCE\a\a-machine-checks.{cjs,json}`, `EVIDENCE\a\a-checks2.{cjs,json}`

---

## 0. Scope / what I read / what I did NOT examine

**Question:** are short-term and long-term memory REAL, or just three enum fields?

**Read in full:** `DD-AUTHORITY-V2.md`; `contracts/state-machines/state-machines.md` (165 L, the only
state-machine contract — no `.json` exists); `contracts/error-codes.md`; all 31 `contracts/**/*.json`
(property-level, by script).

**Read in cited ranges:** `00` L120–413 (§3.1, §5, §7.4, §7.5 时间尺度三层, §7.6, §7.7);
`01` L228–239 (Scope 五类), L276–320 (v1 字段表), L352–395 (lifecycle/availability 转换矩阵),
L462–485 (Provenance operation 九种), L640–755 (§7.15 版本, §7.16 13 张表 DDL, §7.17 A/B 迁移映射,
§7.18 完整性约束); `02` (via the NUL-sanitised, line-preserving copy) L396–443 (ALG-04 六特征),
L685–759 (ALG-08), L845–906 (ALG-10 生命周期算法), L920–950 (ALG-11 Dream);
`09` L126–205, L700–769; `11` L166–195, L430–609, L700–725; `12` L182–199, L319–327, L384–403;
`13` L167–181, L240–270, L286–345.

**Scanned whole-file for absence (E2):** `05`, `06`, `15`, `04`, `07`, `10`, `14`.

**Not examined:** the legacy frozen `03-m01…m05`; `E:\[PROJECT]\R4.1-UNIFIED-MEMORY-CORE` (contains no
L1/L2/L3 at all — `03-ARCHITECTURE-FACTS §B.3` — so it cannot answer this question); the CMB project;
`docs/diagrams` images.

**Prior round:** read only `03-ARCHITECTURE-FACTS` §B/§E/§H/§I/§M/§O and `09-CONFLICT-REGISTER` §C
(FC-013…FC-017). Did not re-audit the 49 conflicts; where I differ from FC-013 I say so explicitly (§9.2).

---

## 1. Headline (capability verdict in 3 lines)

1. **`memory_tier` is a label, not a tier system**: `L1/L2/L3` is one enum on one (non-existent) column,
   with a definition, a *non-binding* semantic-type prior, and five `MUST NOT`s — and **0** transitions,
   0 thresholds, 0 writers, 0 stores, 0 retention jobs, 0 tests (E2: 57 state-machine transitions,
   0 on the tier axis; `memory_tier` in 1 of 31 contracts; 0 of 13 SQLite tables).
2. **There is no short↔long loop and never was one written**: the only sanctioned route
   (L1→L2→L3 via Evidence→Validation→Review, decided by M04) has no algorithm, no decidable threshold
   and no executor step; M04 itself is P4-unauthorised, and four separate `MUST NOT`s forbid every
   automatic shortcut. 16 conversion rules are missing (14 `MISSING`, 2 `PARTIAL`).
3. **Long-term memory cannot be traced to the original input at the storage layer**: `V2 §3.2` and
   `13 §7.3` require `original_text`; the only SQLite DDL (`01 §7.16` #4) has no such column and the
   only Evidence contract has `additionalProperties:false` + `excerpt` instead — while
   `read-response` *requires* `original_text`.

**Explicit four answers:** (1) short-term → long-term: **no** executable route. (2) long-term
independent management: **no** (same row, table, index, writer, permission, retrieval path; the only
long-term-shaped structure, `PersonProfile`, is forbidden to be independent). (3) loss at transition:
one *provable* loss today (the `lifecycle_candidate` field is rejected by `candidate.schema.json`),
plus undefined disposal at session end and no demotion. (4) tiers vs semantic categories: **clean in
normative text, conflated in every executable trace** (the only assignment guidance is a
`semantic_type → tier` matrix; `PersonProfile → (L3, ACTIVE)`; `AUTO_ACCEPT → L1/L2` only) — plus
`L1/L2/L3` denote three unrelated hierarchies across the corpus.

---

## 2. Findings

Full nine-question treatment is in the deliverable §11; this is the register view.

| ID | Finding | Labels | Priority |
|---|---|---|---|
| **A-C001** | Tier axis has **no transition system** — no edge, threshold, executor, audit operation or error code. Short-term cannot reach long-term. | EXISTING DESIGN (`V2 §2.5`, `11 §7.12`) / COMPLETENESS `MISSING` / RECOMMENDED ADDITION: `ALG-13 TierTransition` + `SM-9` (5 edges) | **P0** |
| **A-C002** | `memory_tier` is a **required** field with **no column, no index, no writer** in the authorised P0–P3 scope and **no migration row** (`01 §7.17.1`). | EXISTING DESIGN (`V2 §3.1` L190; `memory-unit.schema.json`) / `MISSING` / ADDITION: DDL column + CHECK + index + bootstrap rule + migration row | **P0** |
| **A-C003** | **L1 has no store and no owner**: `session_state` is FUTURE and "MUST NOT 在 P0–P3 创建"; Session Memory is a FUTURE module; `X-2` says "L1 仅预留". Short-term is architected *outside* Memory Core. | EXISTING DESIGN (`00 §7.5`/§7.7, `01 §7.16` L679) / `CONCEPT_ONLY` / ADDITION: decide 3-value-with-store vs 2-value | **P0** |
| **A-C004** | **Long-term memory cannot be traced to the original input**: `original_text` required by `V2 §3.2`/`13 §7.3`, absent from the DDL and from `evidence.schema.json` (which has `excerpt`), yet required by `read-response`. | EXISTING DESIGN / `CONFLICTED` / ADDITION: align DDL + contract to V2 | **P0** |
| **A-C005** | **Three hierarchies share the tokens `L1/L2/L3`** (memory tier / LLM layer / time scale); `09` uses both meanings in one file; `memory_tier` occurs 11× corpus-wide vs 400+ bare `Lx` tokens. | EXISTING DESIGN / `CONFLICTED` / ADDITION: rename tier to `T1/T2/T3` | **P1** |
| **A-C006** | **No way to ask for a tier**: `read-request` (17 props) has no `tier_filter`, `11 §8.3` L723 defines one, `read-response` requires `tier`. Read capability #4 cannot be exercised; 6 of 9 APIs have no request contract. | EXISTING DESIGN / `MISSING` at contract / ADDITION: `tier_filter` (filter-only) | **P1** |
| **A-C007** | **No retention/expiry/review policy per tier**: only per-`type` decay + per-record `expires_at`; no sweeper; no `lifecycle.*` parameter in `15`; Dream is FUTURE/report-only. | EXISTING DESIGN / `MISSING` / ADDITION: `RetentionPolicy(tier × semantic_type)` | **P1** |
| **A-C008** | **`candidate.schema.json` cannot carry `lifecycle_candidate`** (`additionalProperties:false`, no field; also no 6-state `candidate.status`) though `09` makes it required → the tier proposal dies at the contract boundary. | EXISTING DESIGN (`09` L422/L885) / `CONFLICTED` / ADDITION: regenerate from `V2 §2.7`/§3.1 | **P1** (P0 once A-C001 is fixed) |
| **A-C009** | **Orthogonality asserted, conflation practised**: the only assignment guidance is a `semantic_type → tier` prior (3 copies: `11` L172, L499, L668; `09` L322); `PersonProfile → (L3, ACTIVE)` (`12` L390); `AUTO_ACCEPT` gated on L1/L2 (`09` L755). | EXISTING DESIGN / `CONFLICTED` / ADDITION: evidence-based tier predicate; drop the two hard-wirings | **P1** |
| **A-C010** | **"Needs review" is three different things** on three axes (`candidate.status=NEEDS_REVIEW`; `lifecycle_state=pending_review`; legacy boolean with semantics still TBD = OI-01-03), and nothing indexes the question. | EXISTING DESIGN / `CONFLICTED` / ADDITION: one queue + index | **P1** |
| **A-C011** | `ALG-10` declares `lifecycle.expiry_policy.{type}` / `archive_after_days.{type}` that do not exist in `15` (which has no `lifecycle.*`; its only `*tier*` ID is `ingest.model_tier`, an LLM layer). | EXISTING DESIGN / `PARTIAL` / ADDITION: register | **P2** |
| **A-C012** | **No test exercises a tier transition**; the only three tier tests assert prohibitions (`11` L904–905); `06` contains no `tier` token. | EXISTING DESIGN / `MISSING` / ADDITION: 9 test cases | **P2** |

Counts: **P0 = 4 · P1 = 6 · P2 = 2.**

---

## 3. The required matrix/table for this brief

The 13-column mandatory matrix (层级 · 作用 · 主要数据 · 存储位置 · 写入者 · 读取者 · 进入条件 ·
退出条件 · 晋升条件 · 降级条件 · 保留策略 · 检索策略 · 当前定义是否完整) is in the deliverable **§4**,
with the alternative `00 §7.5` vocabulary in **§4.1**. Accompanying required artefacts:

| Artefact | Deliverable section |
|---|---|
| short → long data flow | §5 |
| tier vs status vs validation_state (+candidate.status, +availability_state, +needs_review) | §6 |
| storage-structure map + per-tier queryability table | §7 |
| ASCII lifecycle diagram (implemented vs required) | §8 |
| explicit list of MISSING conversion rules (16) | §10 |
| the 0-edge vs `11 §7.12` measurement + transcription-or-absence verdict | §9 |

**One-line summary of the matrix:** across the five lifecycle columns × 3 tiers = 15 cells, **9 are
`∅`** (every 退出条件, every 降级条件, all three 保留策略), **5 are intent-only** (进入/晋升), 1 is n/a;
column 13 is **NO** for all three tiers.

---

## 4. What is missing to make this implementable (concrete)

1. **Decide the axis** (product): 3-value tier *with* an L1 store, or 2-value (`L2`/`L3`) with
   short-term living in the host/Session Memory. Nothing else can be settled first (A-C003).
2. **`SM-9 TierTransition`**: 1 initial-state rule + 4 edges (`L1→L2`, `L2→L3`, `L3→L2`, `L2→L1`),
   each with trigger, decider, executor, transaction, audit, test; `L1→L3` stays forbidden (A-C001).
3. **`ALG-13`**: the 16 rules of §10, including a decidable promotion predicate over
   evidence/stability instead of category (A-C001, A-C009).
4. **Give the field a home**: DDL column + `CHECK` + `INDEX(tenant_id, memory_tier, status)`;
   `search-document` field; `tier_filter` in `read-request`; a *bootstrap* rule so the field has a
   legal writer in P1/P2; a migration row for A/B data (A-C002, A-C006).
5. **Evidence chain**: `original_text` in the DDL **and** in the machine contract, aligned to `V2 §3.2`
   (5-value `source_type`, not the current 12-value v1 enum) (A-C004).
6. **Regenerate `candidate.schema.json`** from V2 so `lifecycle_candidate` and the 6-state funnel
   survive (A-C008).
7. **`RetentionPolicy` table** `(tier, semantic_type)` → `ttl_days`, `review_after_days`,
   `on_expiry ∈ {demote, archive, delete}`; register in `15` (A-C007, A-C011).
8. **Rename the tokens** so `L1/L2/L3` stops meaning three things (A-C005).
9. **One end-to-end acceptance test**: a fact stated once in a session becomes an L3 memory with a
   traceable original sentence, and disappears correctly when its retention rule fires (A-C012).

---

## 5. Recommended design (NEW PROPOSAL — not in the corpus)

Minimal coherent tier lifecycle that fits the corpus's existing owners (M03 proposes → M04 decides →
M01 writes), written here so it cannot be mistaken for an approved design:

```
TierTransition(memory_id, from, to, basis, expected_version)
  guard: basis ∈ {EVIDENCE_COUNT(n≥N), SESSION_COUNT(m≥M), USER_EXPLICIT, M04_CONFIRMED,
                  CONTRADICTION, RETENTION_EXPIRED, MERGE, MIGRATION}
  entry: one initial-state rule — the tier written at insert (bootstrap: M01 copies lifecycle_candidate,
         tier_source=PROVISIONAL) until M04 confirms it
  edges: L1→L2 | L2→L3 | L3→L2 | L2→L1
  forbidden: L1→L3 direct; any edge without basis; any tier-driven read/permission change
  effects: memory_tier := to; version+1; event_ledger.operation='tier_change';
           audit; outbox unchanged (tier is not indexed) — or index if tier_filter pushes down
  idempotent on (memory_id, from, to, version); illegal ⇒ VALIDATION_ERROR with reason_code
  failure: no partial application; tier and audit in one M01 transaction
```

Plus two boundary rules that the corpus currently violates:
`tier ≠ trust` (so `AUTO_ACCEPT` must key on `validation_state`/`source_type`, not on L1/L2) and
`profile rows are a view, not a tier holder` (so `(L3, ACTIVE)` must be removed from `12` L390).

---

## 6. Cross-scope handoffs

| To | Item | Anchor |
|---|---|---|
| **B** (long-term structure) | `PersonProfile` is the only long-term-shaped structure and is `(L3, ACTIVE)`-hard-wired; tier is its only durability marker | `12` L390; deliverable §3 Q2 |
| **C** (Evidence/Emotion) | Three incompatible Evidence shapes: `V2 §3.2` (13 fields incl. `original_text`, 5-value `source_type`) vs `01 §7.16` #4 (11 columns, no `original_text`) vs `evidence.schema.json` (12-value `source_type`, `excerpt`) | deliverable §7 |
| **D** (retrieval/context) | `read-request` cannot express `tier_filter`; `11 §8.3` and the contract are two different `ReadRequest`s; ALG-04's six features key on the abolished v1 `type` | deliverable A-C006 |
| **E** (state machines/lifecycle) | `SM-1` still encodes the abolished `lifecycle_state`; `ALG-08` §7 and `ALG-10` §4.1 also speak v1; `operation` 九种 has no tier operation | deliverable §6, MCR-13 |
| **Lead** | Two decisions worth asking the user: (1) does an `L1` exist *inside* Memory Core? (2) is there **any** demotion? Both are cheap now and expensive later; and note that fixing this is a *design* ticket, not a transcription ticket (§9.2). | deliverable §9.2, §12 |

---

## 7. Method appendix

**Scripts (E2, no corpus writes):**
- `EVIDENCE\a\a-machine-checks.cjs` → `a-machine-checks.json`: state-machine row/tier census
  (8 machines, 57 rows, 0 tier rows, 0 tier tokens in the file); `11 §7.12` block extraction
  (heading L493, 6 matrix rows, 5 MUST rules, 0 `Lx→Ly` edges); 31-contract census
  (`memory_tier` 1/31, `lifecycle_state` 2/31, both 0/31, `original_text` only in `read-response`);
  13-table DDL scan (0 `memory_tier`, evidence columns enumerated, FUTURE tables line);
  corpus-wide token census (18 docs, 1 064 353 bytes).
- `EVIDENCE\a\a-checks2.cjs` → `a-checks2.json`: `read-request` 17-property list; `candidate` /
  `evidence` contract checks; `09` homonym line extraction (LLM-layer `L1/L2` at L241/347/924 vs
  tier `L1/L2/L3` at L422/443/885); structural SM row tally (superseded by the manual count).

**Commands:** targeted `Select-String` scans over `05`, `06`, `15`, `error-codes.md`,
`read-request.schema.json`; `grep`-tool scans for `memory_tier`, `晋升|降级|升层|降层`,
`CREATE TABLE`, `lifecycle.*`.

**Caveats recorded in the deliverable §14.4:** (a) `02` is NUL-corrupted and read from the
line-preserving sanitised copy — and that copy's numbering does **not** match several `02` line
numbers quoted in `03-ARCHITECTURE-FACTS` (e.g. "ALG-08 `02:697-709`"), so I cite `02` by section
wherever I did not read the line myself; (b) the SM row count is 57 by manual reading (one regex pass
returned 56) — the tier count of 0 is unaffected; (c) `Lx` token counts include LLM-layer meanings and
are used only for scale.

**Encoding:** all files written with the `write` tool / `fs.writeFileSync(..., 'utf8')` — no BOM, no
PowerShell `-Encoding utf8`. Verified by reading the file back with the harness reader.
