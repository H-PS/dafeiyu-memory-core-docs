# CROSS-DOCUMENT CONSISTENCY MATRIX — Memory Core R4.1 baseline

**Agent H · `05-audit/` · 2026-11-05**
Corpus scanned: `01-baseline-corpus/` — 212 files, 89 primary markdown documents (3 sets) + 40 backup snapshots.
Machine evidence: `05-audit/scan-references.cjs` → `06-evidence/raw/audit/scan-references.run1.json` / `.run2.json`; counts in `GLOBAL-REFERENCE-ISSUES.md`.

Each row states the two (or more) anchors that disagree, a verdict, and a severity. Severity: **S1** = a correct implementation cannot satisfy both; **S2** = one side is stale and would mislead an implementer; **S3** = editorial drift, no behavioural effect.

---

## 1. Findings summary

| ID | Axis | Finding (one line) | Severity | Status |
|---|---|---|---|---|
| SC-01 | Parameter identity | Registry declares `recall.*` the only ID and orders a full replacement; 53 ID-shaped `retrieval.*` occurrences remain, 20 of 36 distinct IDs have no canonical mapping at all | S2 | OPEN |
| SC-02 | RRF channel | `structured` is excluded from RRF by the decided rule, but `02 §7.4` still defines a `w_structured=1.0` weight and `00`'s diagram still routes `STR → RRF` | S1 | OPEN (partially fixed) |
| SC-03 | Status axes | Three different status decompositions coexist (3 state machines + tier, 3-part mapping, 4 legacy dimensions) and `status` is used for both memory status and candidate status | S1 | OPEN |
| SC-04 | v1/v2 enums | V2 "abolishes `content_kind`/`type`(7)" while `01 §7.3`/`§7.6.2`, `02:630` and `memory-object.schema.json` still **require** both | S1 | OPEN |
| SC-05 | Contract layer | `memory-object.schema.json` (v1) and `memory-unit.schema.json` (V2) both ship in `contracts/schemas/` with no deprecation marker; `evidence.schema.json` is v1-stale | S1 | OPEN (B-1) |
| SC-06 | Broken references | 295 unresolved S1 references (target absent / section absent) + 85 S2 (section named but undefined) | S2 | OPEN |
| SC-07 | Duplicate authority | `DD-AUTHORITY.md` and `DD-AUTHORITY-V2.md` both claim to be the highest behavioural basis | S2 | OPEN |
| SC-08 | Duplicate authority | Three document sets claim the same subjects (legacy M01–M05, design-package 00–15, unified-set 02–16) | S1 | OPEN (B-4) |
| SC-09 | Channel naming | `fts`/`fts5` and `vector`/`lance` appear as channel identifiers in different documents while the response schema enumerates a closed set | S3 | OPEN |
| SC-10 | Two-document-set precedence | No precedence rule exists; the two R4.1 sets are mutually unreferencing and architecturally divergent | S1 | NEEDS PARENT DECISION (B-4) |

---

## 2. Axis A — parameter identity (cross-check of Agent E, not a re-scan)

**Status of the cross-check.** Agent E's own `PARAMETER-IDENTITY-REPORT` was never present under `05-audit/`, `06-evidence/` or `07-reports/` (checked 3× during the session). However the parent's register records the executed result as **MC-AUD-006**: `validate-parameter-registry.cjs` — **68 canonical IDs vs 355 references, 75 unresolved across 55 distinct names** (`SECOND-ROUND-FINAL-ISSUE-REGISTER.md:25`). That is the authoritative number and I cross-check against it rather than duplicating the scan. My bounded count restricts to the one family where the registry declares an explicit migration table (`retrieval.*` → `recall.*`) and asks whether the *source documents* were edited.

| Measurement | Scope | Value | Anchor |
|---|---|---:|---|
| Canonical IDs / references / unresolved / distinct unresolved | whole registry (**MC-AUD-006**) | 68 / 355 / 75 / 55 | `SECOND-ROUND-FINAL-ISSUE-REGISTER.md:25` |
| Canonical `recall.*` IDs declared in the registry | 15 only | 19 | `15-参数注册表.md` §7.1.3 (L138–162), §7.17 |
| ID-shaped `retrieval.*` occurrences (non-review docs) | 15 (21), `02` (19), `11` (10), PENDING-USER-DECISIONS (2), `00` (1) | 53 | executed `tools/param-id-count.cjs` |
| Distinct `retrieval.*` IDs | — | 36 | idem |
| Distinct IDs covered by the §7.1.3 migration table | — | 16 | `15` §7.1.3 |
| Distinct IDs with **no** canonical mapping | — | 20 | e.g. `retrieval.fts.field_weights`, `retrieval.rrf.weights.structured`, `retrieval.rrf.version`, `retrieval.rank.min_importance`, `retrieval.tokenizer.id`, `retrieval.profile.event_supplement_max` |

**Agreement with MC-AUD-006:** no contradiction, different scope. MC-AUD-006 measures the registry's *aliases* and finds 75 unresolved references across 55 names; I measure the *documents* and find that the algorithm spec still speaks the superseded dialect entirely. Both point the same way: SRD-003 is decided but not enforced. My 20 unmapped IDs are a subset of the 55 unresolved names MC-AUD-006 counts, because `retrieval.*` is only one of the legacy naming families.

Verdict: the registry is internally consistent and even declares the remedy ("`MUST` 全表替换引用", `15:138`), but the corpus still carries both identities. The worst offender is the algorithm document itself: `02-核心算法规范.md` uses 19 `retrieval.*` IDs and **zero** `recall.*` IDs, so the document that defines k, α…ζ does not speak the registry's language. `11-MemoryAnchor与语义分类.md` adds 10 further IDs (`retrieval.anchor.*`, `retrieval.profile.*`) that post-date the migration table and have no mapping at all — a second-generation drift, not a leftover.


---

## 3. Axis B — RRF channel drift

Decided rule (three independent statements agree): RRF input is `fts` + `vector` only; `structured` is filter/router.
- `11-MemoryAnchor与语义分类.md:617` — structured **MUST NOT** be counted into RRF.
- `06-测试与评估规范.md:558` (V-06) — "RRF 输入仅含 fts/vector；structured 只过滤/路由".
- `V1-IMPLEMENTATION-BASELINE.md:22` — "RRF channels: `fts` and `vector`; structured filters/router only".
- `OPEN-ISSUES.md:64` (OI-022) and `SECOND-ROUND-ISSUE-REGISTER.md:11` (MC-AUD-005) record it as DECIDED; `SECOND-ROUND-CHANGELOG.md:7` records the schema edit (`retrieval-response.source_channels` restricted to unique `fts`/`vector`).

Contradicting text still present in the same corpus:

| Anchor | Text | Effect |
|---|---|---|
| `02-核心算法规范.md:344` | config list includes `retrieval.rrf.weights.structured` | defines a weight for a channel that must not be fused |
| `02-核心算法规范.md:346` | `w_structured=1.0`（EXP-DEFAULT，仅在有结构化通道时启用） | directly contradicts the MUST NOT above |
| `02-核心算法规范.md:4` (header) | "RRF 仍不计入 structured 通道（OI-022 结论）" | the *same file* contradicts its own §7.4 |
| `00-总体架构与设计原则.md:638` (mermaid) | `R3 --> R5[RRF 融合]` fed by `STR` | diagram still routes structured into RRF |
| `02-核心算法规范.md:113` | channel enum is `fts5 / vector / structured` | channel enum is wider than the fusion set — legitimate only if clearly separated, which it is not at L312/L344 |

Verdict: **S1 OPEN.** A implementer reading `02 §7.3–§7.4` (the section that owns the formula) will weight `structured`; a tester reading `06` V-06 will fail that implementation. The decision exists but the carrier document was never edited. Fix is 4 lines: `02:4` is already correct, `02:312/344/346` and `00:638` are not.

Two related defects already executed and adjudicated by the parent, not re-measured here: **OI-R-03** — the worked example in `02 §7.4` assigns `channel_rank 1` twice inside the `fts` channel, so the only hand-checkable RRF example in the corpus cannot be hand-checked (`SECOND-ROUND-FINAL-ISSUE-REGISTER.md:67`); and **MC-AUD-005** — the reference implementation rejects `structured` as a channel (`rrf.structured_channel_rejected`), which is the behaviour the text contradicts.


Note on the two-document problem: `unified-set` has the *other* drift — it keeps **four** channels in the fusion narrative (`02-ARCHITECTURE-OVERVIEW.md:64` "四渠道检索调度、RRF 融合") while its own chart says two-way (`:46`). See SC-10.

---

## 4. Axis C — status-axis conflation

Three decompositions are live simultaneously:

| Source | Decomposition | Anchor |
|---|---|---|
| Design package, declared mapping | `lifecycle_state`(6) → `memory_tier(L1/L2/L3)` + `status(ACTIVE/SUPERSEDED/ARCHIVED/DELETED)` + `candidate.status(6)` — **three** axes | `00` §0.6 (L75) |
| Design package, data model | `validation_state` + `lifecycle_state` + `availability_state` — **three different** axes, `lifecycle_state` retained | `01` §7.7 (L319–401), §7.6.3 (L271) |
| Legacy | 审核 / 有效 / 存储 / 权限 — **four** dimensions | `03-m04.md` §9.1 (L579–593) |

Consequences and specific conflicts:

1. **`lifecycle_state` is both abolished and mandatory.** `00:75` maps it away; `01` §7.6.3 requires it as a field; `memory-object.schema.json` lists it. Two of the three live schemas disagree with `00`.
2. **`status` is overloaded.** In `memory-unit.schema.json` `status` is the *memory* status; in `09` §7.21 (L740) the "候选状态 6 态" is the *candidate* status; `01` §7.12 (L538) declares Candidate a FUTURE object with its own state. A bare `status` token cannot be resolved without knowing which object it belongs to.
3. **`memory_tier` is not a status.** `00` §0.1 (L16) says tier is lifecycle level, not importance; `02` §7.11 (L849, lifecycle algorithm) still derives availability from `lifecycle_state`. Nothing in the corpus states how `memory_tier` interacts with `status=ARCHIVED`.
4. **`validation_state` vs `source_type`.** `13` §7.6 (L370) makes truthfulness a function of `source_type`; `01` §7.7.1 makes it a function of `validation_state`; both are mandatory fields, and no rule orders them.

Verdict: **S1 OPEN.** Minimal closing condition: one table in `01` §7.7 stating the authoritative axis list, marking `lifecycle_state` DEPRECATED or restoring it, and giving the tier×status legal combinations.

Two independent confirmations from the parent's executed verification, which I cite rather than repeat: **OI-R-06** — `contracts/state-machines/state-machines.md` SM-1 lists `active → disputed` as a *lifecycle* transition although `disputed` is a `validation_state` value, i.e. the axis conflation reached the state-machine contract and a literal transcription would implement a cross-axis jump (`SECOND-ROUND-FINAL-ISSUE-REGISTER.md:89`). **OI-R-07** — there is no table of legal `memory_tier` transitions anywhere in the corpus, so `L1→L2→L3` promotion, demotion and expiry are unimplementable without inventing policy (`:98`, `:114`). My items 1–4 above describe the textual cause; OI-R-06/07 are its contract- and behaviour-level effects.


---

## 5. Axis D — v1/v2 enum mixing

| Item | v1 form (still present) | V2 form (declared) | Conflict |
|---|---|---|---|
| Memory classification | `type` 7 values (`semantic/episodic/procedural/preference/intent/relationship/observation`) — `01` §7.2 (L179–191), `02:630` | `semantic_type` 6 values + `semantic_tags` — `00` §0.6 (L77) | Two classification axes; `01` §7.4 builds a 7×8 matrix on the v1 pair, which V2 has no slot for |
| Content nature | `content_kind` 8 values — `01` §7.3 (L193–206), `§7.6.2` L265 ("必填"), `02:630`, `memory-object.schema.json` | **abolished** — `00` §0.6 (L76) | A field declared abolished is mandatory in the data model and in the schema |
| Source type | 12 values — `evidence.schema.json` (`direct_user_statement`, `assistant_observation`, `imported_document`, `tool_result`, `system_event`, `approved_inference`, `explicit_user`, `conversation`, `repeated_statement`, `imported`, `system_generated`, `human_reviewed`) | 5 values — `13` §7.6 (L370), `memory-unit.schema.json` | 12 ⊃ 5 with three near-synonym pairs (`explicit_user`/`direct_user_statement`, `conversation`/`repeated_statement`, `imported`/`imported_document`); any Evidence object built to `13` will be schema-valid but semantically ambiguous |
| Scope | 3 (`private/group/shared`) — `03-m05.md` §3 | 5 (`owner_private/group_shared/session_only/project_scoped/system_internal`) — `01` §7.5 (L228) | Values do not map 1:1; `session_only` and `project_scoped` have no legacy counterpart |
| Visibility | absent | 3 values — `04` §7.5, `memory-unit.schema.json`, `evidence.schema.json` | `evidence` carries `visibility` while `memory-object` carries none |
| Lifecycle | `lifecycle_state` 6 | `memory_tier` + `status` | see SC-03 |

Verdict: **S1 OPEN.** The V2 upgrade was written as a header note (`00` §0 "本文件的新增首章") and a mapping table, but 01/02 — the two documents that define fields and enums — were not rewritten. `02:630` is the clearest single proof: the algorithm's classifier step still requires passing the 7×8 v1 matrix.

---

## 6. Axis E — broken references

Full detail and per-file tables: `GLOBAL-REFERENCE-ISSUES.md`. Headline from the executed scan (`node 05-audit/scan-references.cjs`, exit 0, canaries 7/7):

| Metric | Value |
|---|---:|
| References seen | 5 701 |
| Unresolvable, severity 1 (target does not exist) | 295 |
| Unresolvable, severity 2 (target named but undefined) | 85 |
| Informational (external source absent, alias-only, planned artefacts, enumerated-item citations) | 1 402 |
| **Unresolved total** (S1 + S2 + S3) | 1 782 |

Three systematic causes, not hundreds of independent mistakes:
1. **Citations into sources the corpus does not ship** — 893 references to `m.md`, `sf.md`, the task book (任务书), the adjudication ledger (台账) and reviewer artefacts. The design package's "highest behavioural basis" is not in the delivery.
2. **Legacy basenames** — `m01…m05.md`, `adjudication-ledger.md`, `05-conflict-register.md`, `06-interface-map.md` are cited but the corpus ships them as `03-m0x.md`, `08-adjudication-ledger.md`, `09-…`, `10-…` (34 refs, alias-resolvable).
3. **Fixture files named but not shipped** — `queries.json`, `memories.json`, `relevance.json`, `permission_cases.json`, `expected_results.json`, `conflict_cases.json` (`06` §7, L343–348): the test contract exists, the data does not.

---

## 7. Axis F — duplicate / near-duplicate authoritative files

| Duplicate subject | Claimants | Assessment |
|---|---|---|
| "highest behavioural basis" | `DD-AUTHORITY.md` (v1.0, APPROVED 2026-11-04) vs `DD-AUTHORITY-V2.md` ("效力高于本文任何 v1 表述", `00:6`) | Two authority documents, one declared superior but the v1 file is neither deleted nor marked superseded. **S2** |
| Memory object | `contracts/schemas/memory-object.schema.json` (v1: `content_kind`, `lifecycle_state`) vs `memory-unit.schema.json` (V2: `semantic_type`, `memory_tier`, `visibility`) | Both ship, neither is marked deprecated; both are reachable by schema path. **S1** |
| Evidence object | `contracts/schemas/evidence.schema.json` (12 `source_type` values, no `original_text`, `additionalProperties:false`) vs `13` §7.2/§8.1 (V2 field table) | Proven by execution in **OI-R-04**: a compliant instance is rejected with `additionalProperties`/`required`/`enum` failures; a fixed copy exists in `04-contracts/schemas/` but is **not merged** into the canonical package (`SECOND-ROUND-FINAL-ISSUE-REGISTER.md:74`). **S1** |
| Data model | legacy `03-m01.md` §2 | `01-记忆本体与数据模型.md` | `unified-set/03-DATA-FLOW-AND-LIFECYCLE.md` | Three data-model authorities. **S1** (see SC-10) |
| Retrieval algorithm | `02-核心算法规范.md` (five-stage) | `unified-set/06-ALGORITHM-DESIGN.md` (four-channel RRF) | Divergent, not complementary. **S1** |
| Permission model | `04-权限与治理规范.md` (5 scopes + visibility) | `unified-set/07-PERMISSION-AND-PRIVACY.md` (3 scopes) | Divergent. **S1** |
| Consistency audit | `design-package/reviews/CROSS-DOCUMENT-CONSISTENCY-REPORT.md` (prior report, marks 29 items PASS/FIXED) | this document | The prior report's own body still cites `contracts/api/permission-api.md`, a file that does not exist (`scan: E_PATH_NOT_FOUND` at `reviews/CROSS-DOCUMENT-CONSISTENCY-REPORT.md:141,193`). Its PASS verdicts should be re-read against this matrix. **S3/S2** |
| Test data | `06` §7 table vs shipped files | — | Names 6 fixture files; 0 present |

---

## 8. B-4 — the two-document-set problem

The parent register already records this as **OI-R-01, P1 · OPEN**, with the verdict "Architecture-level — reported to the user, not decided by me" (`SECOND-ROUND-FINAL-ISSUE-REGISTER.md:47`). What follows is the clause-level evidence behind that entry plus the option analysis it asks for; it does not re-open the finding, it supplies the material a decision needs.

### 8.1 What each set claims to be authoritative for

**`Memory-Core-R4.1-Design` (00–15 + `contracts/`)**
- Status: **PROPOSED** — "设计成稿，未获用户逐条批准" (`00` §1, L95); implementation authorisation: none (L96).
- Highest basis: `DD-AUTHORITY-V2.md`, itself "效力高于本文任何 v1 表述" (`00:6`); below it the adjudication ledger, then the legacy M01/M02 (second layer, `00:178`).
- Self-description: a **V2 architecture upgrade** (2026-11-04 task book): five core principles, 17 mandatory capabilities, 16 documents, nine Memory APIs, a 16-stage ingestion pipeline, MemoryUnit/Event/EmotionContext/PersonProfile, five-stage retrieval with 8-anchor routing (`00` §0.1–§0.5).
- Authority style: decision-id LEDGER entries (`D-AP-01…10`, `I-x`, `X-x`, `CR-xx`) with explicit binding force; the only set that owns machine contracts (`contracts/schemas/*.json`, 20 files).

**`R4.1-UNIFIED-MEMORY-CORE` (00–16 + `audit/`)**
- Status: **CANDIDATE_DELIVERY** — "候选版（Candidate），等待用户审阅。不构成用户批准、不构成实现授权" (`00-README.md:4`, `FINAL-REPORT.md:74`).
- Highest basis: `top-记忆库项目设计宗旨与统一设计原则.md.md` (`00-README.md:6`) — a file that is **not in the corpus**.
- Self-description: an **integration of the R3 five modules** ("整合骨架：原始 R3 五模块设计（m01–m05 + dream + PSYCH）＋ R4 六轮用户裁决条件", `00-README.md:5`), explicitly "A/B 是审查分工，不是架构分工" (`:14`), keeping the R3 skeleton "原样" (`FINAL-REPORT.md:11`).
- Authority style: citation of `m01…m05` decisions (`D-AP-01`, `m01 D1`, `§10.4`) plus its own issue ids (`B-058`, `N-14`, `RT-01`).

**The two sets do not reference each other.** `grep` for `unified-set`/`R4.1-UNIFIED` in the design package: 0 hits. `grep` for `DD-AUTHORITY` or `contracts/` in the unified set: 0 hits. They are disjoint baselines that happen to share a name and a version string.

### 8.2 Genuine contradictions (not wording differences)

| # | Subject | Design package | Unified set | Nature |
|---|---|---|---|---|
| C1 | Core object model | MemoryUnit (28 fields) + Event + EmotionContext + PersonProfile; MemoryObject kept only as a storage view (`01` §7.1 header note; `memory-unit.schema.json`) | Evidence → Fact/Event → Observation three-layer, "架构骨架维持 R3 五模块原样" (`FINAL-REPORT.md:11`; `02-ARCHITECTURE-OVERVIEW.md:132`) | Different data model. The words "MemoryUnit", "EmotionContext", "PersonProfile" appear **0 times** in the whole unified set. |
| C2 | Retrieval architecture | Five stages + 8-anchor routing (`00` §0.1, `02` §7.3) | Trigger → four-channel recall → RRF → rule ranking → dedup → budget (`06-ALGORITHM-DESIGN.md` §一) | No anchor router in the unified set; no four-channel RRF pipeline in the design package. |
| C3 | Ingestion | 16-stage Pipeline as a named coordinating component (`09`, 16 stages) | Asynchronous Retain through M03 → M04 → M01 (`02-ARCHITECTURE-OVERVIEW.md`, `03-DATA-FLOW`) | The unified set has no Pipeline component ("16 阶段"/"Ingestion" = 0 hits). |
| C4 | Lifecycle axes | `memory_tier` + `status` + `candidate.status` (3) | Four dimensions 审核/有效/存储/权限 (`05-STATE-MACHINES.md` §一, "四维分离，m04 §九，已冻结") | Directly different, and both call their version frozen/decided. |
| C5 | Read/write surface | Nine Memory Read/Write APIs (`14`, `00` §0.5) | `memory_search` + Retain; "第一版只实现聊天所需的 Recall 和 Retain" (`03-m05.md` §11.2, inherited) | The nine-API surface is absent from the unified set ("9 接口"/"memory.search" = 0 hits). |
| C6 | Emotion | EmotionContext object + ΔE = I×C×R×D with 8 gates (`13` §7.9–§7.10) | Psychology as an external extension without ranking power; no EmotionContext object | Different scope, different owner. |
| C7 | Status of legacy M01–M05 | second-layer, "historical sources, not silently promoted authority" (`SECOND-ROUND-FULL-AUDIT-REPORT.md:12`; `00:178`) | the skeleton itself, "原文不修改" (`00-README.md:75`) | Opposite treatment of the same five documents. |

### 8.3 Clauses present in one and absent in the other, that matter for implementation

| Present in | Absent in | Clause | Why it matters |
|---|---|---|---|
| design-package | unified-set | 8-anchor routing / `anchor-set.schema.json` | determines the retrieval contract; an implementation built on the unified set cannot satisfy it |
| design-package | unified-set | 16-stage Pipeline + `IngestEnvelope`/`CandidateDraft`/`ValidationReport` | determines the write path and idempotency keys |
| design-package | unified-set | 9 Memory Read/Write APIs + `read-request/read-response` schemas | determines the LLM-facing surface |
| design-package | unified-set | `memory_tier` (L1/L2/L3) | determines retention and injection rules |
| unified-set | design-package | Event-L3 evidence chain (12 design revisions, `08` doc) + explicit "L3 暂缓" red line | determines whether "memory actually reached the prompt" is observable; the design package has no L3 at all |
| unified-set | design-package | 47-item A/B revision register + red-team findings (`audit/INTEGRATION-MATRIX.md`, `RED-TEAM-REVIEW.md`) | these are the only recorded reviews of the R3 skeleton |
| unified-set | design-package | B-058 (permission tables missing) as an explicit P0 blocker | the design package has the same gap but no entry: my R-107 |
| unified-set | design-package | Explicit "14-OPEN-DECISIONS" dependency ordering | the design package scatters open items across `OPEN-ISSUES.md` + `PENDING-USER-DECISIONS.md` |

### 8.4 Which set does `contracts/` belong to?

**`contracts/` belongs to `Memory-Core-R4.1-Design` and to nothing else.** Evidence: the folder exists only at `01-baseline-corpus/design-package/contracts/` (20 schemas + `api/api-contracts.md` + `state-machines/state-machines.md`); the unified set contains zero references to `contracts/` or to any `*.schema.json` (grep: 0 hits), and zero mentions of `DD-AUTHORITY`. Any statement of the form "the unified set is the baseline" therefore either orphans the entire machine-contract layer or silently re-homes it to a set that has never seen it.

### 8.5 Precedence recommendation (evidence-based, not preference)

I do **not** declare a winner. The evidence supports a *conditional* precedence rule, and it must be made by the parent as an architecture decision.

**Facts the decision must respect**
1. Neither set has user approval: PROPOSED vs CANDIDATE. Mutual status does not establish precedence.
2. The design package is the only set with (a) machine contracts, (b) a decision ledger with ratified ids, (c) an explicit inheritance stance toward the legacy set.
3. The design package is **internally split** (SC-03/04/05): its own V1 body contradicts its own V2 header. Precedence between sets does not fix that.
4. The unified set is the only set that (a) explicitly integrates A/B review findings, (b) records B-058 as a blocker, (c) keeps the R3 skeleton the legacy documents were frozen against — i.e. it is the closer successor of `03-m01…m05`.
5. The two sets are architecturally incompatible in seven places (C1–C7); no mechanical merge exists.

**Options**

| Option | Rule | Consequences | Cost / risk |
|---|---|---|---|
| **P1 — Design package wins, unified set demoted to review annex** | `Memory-Core-R4.1-Design` is the baseline; `unified-set` is retained as an evidence/review artefact only; its findings (B-058, N-14, RT-01) are imported as issues | Preserves contracts, ledger discipline and V2 objects; keeps one implementation target. Loses the 47 A/B revisions unless imported. Requires fixing the V1/V2 split *inside* the winner before any code | Medium. Needs an import list (~47 items) and a `contracts/` re-freeze; the V2 objects must be propagated to `01`/`02` or dropped |
| **P2 — Unified set wins, design package demoted to contract annex** | `R4.1-UNIFIED-MEMORY-CORE` is the baseline; the design package survives only as `contracts/` supplier | Consistent with the R3 legacy skeleton and with my inheritance matrix (80 % of legacy clauses already land there); one coherent data model. But the contracts then describe objects the baseline does not have → the contract layer must be rewritten or dropped, and the V2 capabilities (17 items, 8 anchors, 9 APIs, Pipeline) are abandoned | High. Discards a whole design generation and the only machine-readable contracts |
| **P3 — Split by layer (recommended to consider)** | Behavioural/data-model baseline = design package **for V2-declared capabilities**; retrieval/ingestion/inheritance narrative = unified set where the design package is silent; contracts = design package, mandatory | Exploits each set's actual strength; matches the evidence that the sets are disjoint rather than competing on the same clauses | Highest complexity: requires the parent to publish a *clause-level* precedence table (this audit's matrix is the raw material) and to keep two documents in sync for as long as the split lasts. Risk of permanent dual maintenance |
| **P4 — Defer** | No rule; both remain PROPOSED/CANDIDATE and no implementation starts | Honest, cheap today; blocks every downstream agent that needs one baseline (D/E/F/G all consume schemas) | Immediate: at least 4 agents cannot conclude; the contradiction set grows |

**Recommendation (flagged for the parent as an architecture decision, not decided here):**
Adopt **P1 as the default with a bounded P3 concession** — design package as the baseline including `contracts/`, unified set demoted to a review annex, **and** an explicit import list from the unified set limited to findings the design package lacks (B-058 permission tables, N-14 L3/contract mutual exclusion, RT-01…RT-18 red-team items, the 47 A/B revisions). Rationale, in evidence order: (i) only the design package has machine contracts, so any other choice orphans or invalidates 20 schema files; (ii) only the design package has a ledger with ratified decision ids that other documents already cite by number; (iii) the unified set's own self-description ("candidate", "等待用户审阅", "不构成实现授权") is the weaker authority claim of the two; (iv) its unique value is review findings, which are importable as issues without importing its architecture.

**What the parent must decide (three questions):**
1. P1 / P2 / P3 / P4 — and if P1, whether the V2 objects (MemoryUnit/Event/EmotionContext/PersonProfile) or the V1 body wins *inside* the design package (SC-03/04/05 cannot be left open after a precedence choice).
2. Who owns the import list from the unified set, and whether importing B-058/N-14 creates implementation blockers before code starts.
3. Whether `contracts/` is frozen, revised, or re-generated once the object model is settled — currently `memory-object` and `memory-unit` both ship, so any schema-driven agent will pick one at random.

---

## 9. Limits of this matrix

- Counts come from the executed scanner and the bounded parameter count; both are reproducible (`GLOBAL-REFERENCE-ISSUES.md` §7).
- I did not evaluate whether either set is *correct* engineering; only what each claims and where they conflict.
- Agent E's `PARAMETER-IDENTITY-REPORT` was unavailable, so SC-01 is a bounded measurement cross-checked against the parent's executed **MC-AUD-006** (68/355/75/55) rather than a replacement for it. Likewise OI-R-03/04/06/07 are cited from `SECOND-ROUND-FINAL-ISSUE-REGISTER.md`, not re-executed.
- No file under `01-baseline-corpus/` was modified. All findings are read-only observations.
- Nothing in this document upgrades any authorization axis: `PRODUCT_IMPLEMENTATION_AUTHORIZATION` remains **NOT_AUTHORIZED**.
