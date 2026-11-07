# SECOND-ROUND HISTORICAL INHERITANCE MATRIX — MC-AUD-017

**Task:** Memory Core R4.1 baseline · Agent H · area `05-audit/`
**Register entry:** `MC-AUD-017` — open since round 2, never completed (`design-package/SECOND-ROUND-ISSUE-REGISTER.md`, `SECOND-ROUND-FULL-AUDIT-REPORT.md:12`). The parent's current disposition is **PARTIALLY_CLOSED**, "inheritance matrix being produced by the historical-consistency workstream; not yet verified by the parent" (`SECOND-ROUND-FINAL-ISSUE-REGISTER.md:36`). This file is that matrix, in a form the parent can verify row by row.
**Legacy source (immutable):** `01-baseline-corpus/governance/03-m01.md` … `03-m05.md` (R3 draft set).
**Current destination candidate:** `01-baseline-corpus/design-package/` 00–15 + `contracts/`.
**Scope note:** classification is against the **design-package** set only. The competing `unified-set` is handled separately in `CROSS-DOCUMENT-CONSISTENCY-MATRIX.md §B-4`; a clause classified `UNRESOLVED` here may be present in `unified-set`, which does **not** close the row — it re-opens the two-document-set question.

## 0. Method and counting rules

1. A **clause** is one numbered decision (`Dn`) or one numbered sub-section that carries normative content. Editorial prose, revision histories, freeze declarations and cross-module "consistency review" tables are not counted as clauses.
2. An **anchor** is `file §section (Lstart–Lend)`; every line range below was read in this session. No row is admitted without both a legacy anchor and a destination anchor — rows that would have had a destination only ("the R4.1 set probably covers this") were classified `UNRESOLVED` instead.
3. Classification vocabulary is exactly the six values in the brief.
4. The design package **declares** part of its inheritance, and where it does the declaration is cited rather than inferred:
   - `03-系统运行与协作协议.md:59` — `03-m01.md`(D1–D40) inherited for storage/transaction/backup/recovery detail.
   - `04-权限与治理规范.md:49` — `03-m01.md`(D3/D7/D13/D26) → write responsibility, deletion, data protection, import/export.
   - `05-可靠性与运维规范.md:49` — `03-m01.md`(D8/D9/D11/D12/D28/D29/D34/D37) → backup, integrity, concurrency, health, degradation, process model, directory migration.
   - `01-记忆本体与数据模型.md:94` — `03-m01.md`(D2/D4/D5/D19/D22), `03-m02.md`(D6/D12/D23).
   Declared inheritance is verified against the destination text, not taken at face value; four declared items failed that check and are recorded as `INHERITED_WITH_REVISION` or `UNRESOLVED` with the reason.
5. The legacy set is **not** authority in R4.1. `00-总体架构与设计原则.md:178` places `03-m01.md`/`03-m02.md` in "第二层" (frozen design), below the adjudication ledger, and `SECOND-ROUND-FULL-AUDIT-REPORT.md:12` states they are "historical sources, not silently promoted authority". A legacy clause therefore inherits only if a current document restates it.

## 1. Summary

| Classification | Count | Share |
|---|---:|---:|
| INHERITED | 45 | 37.2 % |
| INHERITED_WITH_REVISION | 52 | 43.0 % |
| SUPERSEDED | 9 | 7.4 % |
| REJECTED | 4 | 3.3 % |
| DEFERRED | 7 | 5.8 % |
| UNRESOLVED | 4 | 3.3 % |
| **Total classified clauses** | **121** | 100 % |

Per module: M01 40 (R-001…R-040) · M02 27 (R-041…R-067) · M03 13 (R-068…R-080) · M04 15 (R-081…R-095) · M05 13 (R-096…R-108) · cross-cutting 13 (R-109…R-121, introduced in §7 because they are the rows whose destination is *not* a single section). Row ids are contiguous, 001–121, no gaps; every row appears exactly once in a classification.

Distribution by finding: 80 % of legacy clauses survive in some form (INHERITED + REVISION), which is expected for a set the R4.1 package declares as its second-layer source. The material risk sits in the 9 SUPERSEDED + 4 REJECTED + 4 UNRESOLVED rows, listed in §7.

## 2. M01 — 记忆存储与数据管理 (40 clauses)

| # | Legacy anchor | Clause | Current destination | Class | Justification |
|---|---|---|---|---|---|
| R-001 | `03-m01.md` §2.1 D1 (L104–133) | SQLite = sole authority, LanceDB = derived vector store, no cross-DB transaction | `01` §7.1 (L157); `03` §7.1 (L108); `00` §7.2 (L275) | INHERITED | Same split, same "no cross-store atomicity" statement, Outbox-mediated. |
| R-002 | `03-m01.md` §2.2 D2 (L135–170) | Object inventory (Evidence / Fact-Event / Observation / Relation / Cue Anchor / Dream Report / Recall Cache / Task) | `01` §7.1 (L157), §7.11 (L517), §7.12 (L538); `01:94` declares D2 | INHERITED_WITH_REVISION | Inventory is re-cut around V2 objects; Observation and Candidate become FUTURE objects (`01` §7.11/§7.12), Dream Report moves to ALG-11 FUTURE. |
| R-003 | `03-m01.md` §2.3 D3 (L170–183) | Write responsibility: only M01 persists authority | `04` §7.8 (L198); `04:49` declares D3 | INHERITED | Write-validation seven items restate the same authority rule. |
| R-004 | `03-m01.md` §2.4 D4 (L183–210) | SQLite data organisation / table split | `01` §7.16 (L658) | INHERITED_WITH_REVISION | 13 PROPOSED tables replace the D4 split; table set is PROPOSED, not frozen. |
| R-005 | `03-m01.md` §2.5 D5 (L210–227) | ID and relation rules | `01` §7.6.1 (L247), §7.10 (L487) | INHERITED | Identifier/version fields and relation model carried with the same constraints. |
| R-006 | `03-m01.md` §3.1 D6 (L229–257) | Transaction and consistency model | `02` §7.2 (L140); `03` §7.8 (L512) | INHERITED | Five-step transaction + optimistic concurrency appear as ALG-01 and as the transaction-boundary table. |
| R-007 | `03-m01.md` §3.2 D7 (L257–300) | Delete / archive / lifecycle | `01` §7.7.2 (L351); `04` §7.9 (L224); `05` §7.11 (L243) | INHERITED_WITH_REVISION | Single `lifecycle_state` replaced by `memory_tier` + `status` + `candidate.status` (`00` §0.6, L75). |
| R-008 | `03-m01.md` §3.3 D24 (L300–324) | Version persistence | `01` §7.15 (L642) | INHERITED | Seven version requirements restated one-for-one. |
| R-009 | `03-m01.md` §3.4 D36 (L324–337) | Read consistency and snapshot | none | UNRESOLVED | No R4.1 clause defines read-consistency/snapshot isolation; grep for 读取一致性/快照隔离 over `design-package` returns only `05:106` (process crash) and `05:241` (instance lock). |
| R-010 | `03-m01.md` §4.1 D14 (L339–353) | Multi-channel retrieval foundation | `02` §7.3 (L198); `00` §0.1 (L16) | SUPERSEDED | Three co-equal channels replaced by five-stage retrieval + 8-anchor routing; `structured` demoted to filter/router. |
| R-011 | `03-m01.md` §4.2 D15 (L353–368) | Vector indexing unit, chunk_id discipline | `01` §7.13.3 (L610) | INHERITED_WITH_REVISION | Index unit preserved; chunking moves to Pipeline stage 5 (`09` §7.9, L375). |
| R-012 | `03-m01.md` §4.3 D16 (L368–380) | Embedding model management | `01` §7.13.2 (L593) | INHERITED_WITH_REVISION | Model version/dimension discipline becomes the eight-field Embedding Space; per-space isolation added. |
| R-013 | `03-m01.md` §4.4 D17 (L380–392) | Embedding generation/update, batch + incremental | `03` §7.7 (L497); `05` §7.4 (L145) | INHERITED | Batch-then-incremental pattern survives as Worker protocol + index rebuild. |
| R-014 | `03-m01.md` §4.5 D18 (L392–408) | FLAT exact search first, no ANN yet | `00` §7.8 (L~430 tech table); `02` §7.3 (L198) | INHERITED | LanceDB FLAT retained; ANN still excluded. |
| R-015 | `03-m01.md` §4.6 D19 (L408–445) | Vector record fields | `01` §7.13.3 (L610) | INHERITED_WITH_REVISION | Field list extended with `embedding_space_id`; "IDs only, fetch text from SQLite" preserved verbatim in intent. |
| R-016 | `03-m01.md` §4.7 D20 (L445–470) | Pre-filter / post-authority-check two-stage filtering | `04` §7.11 (L257); `06` §7.10 V-06 | INHERITED_WITH_REVISION | Two-stage rule kept; "index metadata is not authorisation" restated as a MUST in `04:267`. |
| R-017 | `03-m01.md` §4.8 D21 (L470–492) | FTS5 + replaceable Chinese tokenizer | `02` §7.3 (L198); `15` §7 (L103) | INHERITED_WITH_REVISION | FTS5 kept; tokenizer choice remains open and is registered as a parameter. |
| R-018 | `03-m01.md` §4.9 D22 (L492–513) | Unified Search Document projection | `01` §7.13.1 (L566) | INHERITED | Field-by-field table carried into the R4.1 Search Document definition. |
| R-019 | `03-m01.md` §4.10 D23 (L513–529) | Outbox-driven index sync, rebuild into new space | `03` §7.6 (L357), §7.7 (L497); `05` §7.4 (L145) | INHERITED | Idempotent, delete-first, rebuild-then-switch all restated. |
| R-020 | `03-m01.md` §5.1 D8 (L533–571) | Backup scope + 8-step restore order | `05` §7.3 (L114), §7.6 (L183) | INHERITED_WITH_REVISION | Restore order kept in substance but re-numbered to ten steps; anti-resurrection constraints retained. |
| R-021 | `03-m01.md` §5.2 D9 (L573–592) | Schema/migration version model | `00` §7.12 (six-version model); `05` §7.10 (L234) | INHERITED_WITH_REVISION | Three parallel version tracks become a six-version model with a "must register OI, do not self-reconcile" rule (`00:129`). |
| R-022 | `03-m01.md` §5.3 D10 (L594–624) | Physical directory layout | none | UNRESOLVED | No R4.1 document states the data/backup/runtime/log directory contract (grep 物理目录/目录布局 → no hit). Not covered by the `05:49` declaration, which lists D37 but not D10. |
| R-023 | `03-m01.md` §5.4 D26 (L624–643) | Import/export, no privilege escalation, second confirmation | `04` §7.9 (L224), §7.10 (L251); `04:49` declares D26 | INHERITED_WITH_REVISION | Import/export survives but the export boundary itself is **BLOCKED** by ledger K-2 (`04:247`, `13:346`); this is stricter, not looser. |
| R-024 | `03-m01.md` §5.5 D37 (L643–660) | Controlled directory migration, no hot migration | `05` §4 (L49) declares D37; `05` §7.10 (L234) | INHERITED_WITH_REVISION | Migration boundary restated only as a dependency declaration plus the maintenance window clause; no dedicated migration procedure exists in R4.1. |
| R-025 | `03-m01.md` §6.1 D11 (L664–696) | Integrity checks, P0–P3 severity, checker never guesses | `05` §7.7 (L205) | INHERITED | Four-way consistency check + reconciliation script is the same instrument. |
| R-026 | `03-m01.md` §6.2 D12 (L696–710) | Concurrency: multi-reader/single-writer, WAL, busy timeout | `05` §7.2 (L94); `03` §7.8 (L512) | INHERITED_WITH_REVISION | WAL and single-writer kept; the clause is now expressed inside the failure matrix rather than as a standalone concurrency contract. |
| R-027 | `03-m01.md` §6.3 D13 (L710–723) | v1 protection level, no key management, encryption ≠ authorisation | `04` §7.13 (L294); `04:49` declares D13 | INHERITED_WITH_REVISION | Same posture (no SQLCipher, no key infra) restated as a privacy boundary. |
| R-028 | `03-m01.md` §6.4 D27 (L723–742) | Automatic maintenance, cache expiry, no auto-delete of facts | `03` §7.1 (L108), §7.9 (L528) | INHERITED | Maintenance pipeline + task protocol carry the same rules. |
| R-029 | `03-m01.md` §6.5 D28 (L742–764) | Lightweight health check, three health states | `05` §7.8 (L220) | INHERITED | Monitoring metric list replaces the check list; same three-state vocabulary. |
| R-030 | `03-m01.md` §6.6 D29 (L764–783) | Degradation by failure domain, fail-closed on authority | `05` §7.2 (L94); `00` §10 (L705) | INHERITED | Ten-class / ≥12-row failure matrix is a strict superset. |
| R-031 | `03-m01.md` §6.7 D30 (L783–800) | No un-measured capacity ceiling; upgrade triggers | `07` §7 (roadmap, P0–P6) | DEFERRED | Capacity decisions are pushed behind the P0–P6 gate; no numeric target is stated anywhere, which is consistent with the legacy clause but leaves it without a home clause. |
| R-032 | `03-m01.md` §7.1 D25 (L805–819) | Storage API / Repository / Index Adapter split, no raw SQL | `03` §7.4 (L310) | INHERITED_WITH_REVISION | Split survives; the surface is fixed at seven M01 functions instead of an open "Storage API". |
| R-033 | `03-m01.md` §7.2 D33 (L819–838) | Centralised config, safety hard-constraints vs tunables | `03` §7.11 (L568); `15` §7 (L103) | INHERITED_WITH_REVISION | Eight config classes + a 15-attribute parameter registry; models/plugins still cannot relax safety constraints (`15:333`). |
| R-034 | `03-m01.md` §7.3 D34 (L838–851) | Local single instance, single writer, live-instance lock detection | `05` §7.10 (L241) — "实例锁 MUST 识别有效进程 …（03-m01 D34）"; `00` §7.2 (L275) | INHERITED | Only legacy decision cited by number in R4.1 operational text; note `07/2` of the same file already carries the identical lock rule. |
| R-035 | `03-m01.md` §7.4 D35 (L851–879) | Error taxonomy, retryable flag, "failure never disguised as success" | `03` §7.10 (L540) | INHERITED_WITH_REVISION | Taxonomy replaced by the REST error envelope; the "no fake success" rule survives as MUST. |
| R-036 | `03-m01.md` §7.5 D39 (L879–895) | Change log vs runtime log separation, logs are not a deleted-content bypass | `05` §7.9 (L226); `01` §7.9 (L457) | INHERITED | Provenance (nine operations) + log spec keep both rules. |
| R-037 | `03-m01.md` §8.1 D31 (L899–926) | T01–T10 acceptance list + life-like query set | `06` §7 (test spec), `02` §13 (L1500) | INHERITED_WITH_REVISION | T-items are absorbed into the test taxonomy; the life-like set survives as BM-01…BM-15. |
| R-038 | `03-m01.md` §8.2 D38 (L926–943) | Isolated test DB + fixed dataset | `06` §7 (L343–348) names `queries.json`, `memories.json`, `relevance.json`, `permission_cases.json` … | INHERITED_WITH_REVISION | Contract exists; **the fixtures themselves are absent from the corpus** — see `GLOBAL-REFERENCE-ISSUES.md` (E_FILE_NOT_FOUND on those five names). |
| R-039 | `03-m01.md` §8.3 D32 (L943–958) | First-run initialisation sequence, recoverable, no re-init of existing DB | none | UNRESOLVED | No bootstrap/initialisation clause in 00–15 (grep 首次启动/首次初始化/bootstrap → no hit). |
| R-040 | `03-m01.md` §8.5 D40 (L965–994) | v1 delivery scope vs deferred list | `07` §7 (P0–P6), `00` §7.11 (L~) | INHERITED_WITH_REVISION | Deferred list largely preserved (no distributed store, no ES, no ANN); the delivery gate is now P0–P6 with stage authorisation. |

## 3. M02 — 记忆检索与上下文构建 (27 clauses)

| # | Legacy anchor | Clause | Current destination | Class | Justification |
|---|---|---|---|---|---|
| R-041 | `03-m02.md` D1 (L44–64) | Two-stage hybrid retrieval (auto pre-retrieval + DSH active search) | `02` §7.3 (L198); `14` §7 (L189) | INHERITED_WITH_REVISION | Two stages kept but embedded in a five-stage pipeline; active search becomes nine formal read/write APIs. |
| R-042 | `03-m02.md` D2 (L66–79) | Pre-retrieval trigger scope, not every message | `02` §7.3 (L198) | INHERITED | Same trigger conditions, same "chat about nothing does not retrieve". |
| R-043 | `03-m02.md` D3 (L81–96) | Four channels + RRF fusion, keep per-channel scores | `02` §7.3 (L198), §7.4 (L307) | INHERITED_WITH_REVISION | RRF kept; `structured` removed from the fusion set — and **still violated** at `02:346` (`w_structured=1.0`), see consistency matrix SC-02. |
| R-044 | `03-m02.md` D4 (L98–108) | Five ranking factors, rule-first, no cross-encoder | `02` §7.5 (L369) | INHERITED_WITH_REVISION | Five factors become six weighted features (α…ζ); cross-encoder still excluded. |
| R-045 | `03-m02.md` D5 (L110–148) | L0/L1/L2 expansion, M02 owns traversal and total budget | `02` §7.3 (L198) | INHERITED | Budget ownership and "relation ≠ support" rule survive verbatim in intent. |
| R-046 | `03-m02.md` D6 (L387–401) | Retrieval result format | `02` §8.1 (L1118); `03` §7.3 (L135) | INHERITED_WITH_REVISION | `ScoredCandidate` + retrieval-response schema replace the informal field list. |
| R-047 | `03-m02.md` D7 (L402–420) | Dynamic context budget | `02` §7.6 (L483) | INHERITED_WITH_REVISION | Four budgets + hard caps replace the single dynamic budget. |
| R-048 | `03-m02.md` D8 (L421–442) | Cache policy: hit ≠ authorisation, same-request reuse only | `15` §7 (L403) `cache.*` forbidden; `14:707`; `09:690` | DEFERRED | Cache tables are DEFERRED by ledger U-1/U-2 across 01/06/08/10/11/12/13/15; the *rule* survives (`14:707`, `09:690`) but there is no cache design to attach it to. |
| R-049 | `03-m02.md` D9 (L443–463) | Retrieval failure and degradation | `05` §7.2 (L94) | INHERITED | Index/storage failure rows match. |
| R-050 | `03-m02.md` D10 (L464–486) | Retrieval quality metrics | `02` §7.13 (L958) | INHERITED_WITH_REVISION | Metric set generalised; five-stage counting added (`02:1045`). |
| R-051 | `03-m02.md` D11 (L489–501) | Retrieval input sources (trusted identity only) | `03` §7.3 (L135); `08` §7 (L143) | INHERITED | Trusted-context-only input preserved. |
| R-052 | `03-m02.md` D12 (L502–516) | Source and confidence propagation | `01` §7.9 (L457); `13` §7.6 (L370) | INHERITED_WITH_REVISION | `claim_type` three-value becomes `source_type` five-value; inference must be labelled. |
| R-053 | `03-m02.md` D13 (L519–526) | Deduplication | `02` §7.5 (L369) | INHERITED_WITH_REVISION | Dedup becomes the `P` penalty feature rather than a separate pass. |
| R-054 | `03-m02.md` D14 (L527–534) | Time-aware retrieval and validity | `01` §7.14 (L625) | INHERITED_WITH_REVISION | Three time fields become six; "fact time ≠ record time" made explicit. |
| R-055 | `03-m02.md` D15 (L535–546) | Person/entity retrieval | `11` §7 (8 anchors); `12` §7 (L179) | INHERITED_WITH_REVISION | Entity retrieval generalised into anchor routing + person-profile priority. |
| R-056 | `03-m02.md` D16 (L547–581) | DSH active retrieval rules | `14` §7 (L189); `08` §7 (L143) | INHERITED_WITH_REVISION | Becomes nine formal APIs with per-call permission checks. |
| R-057 | `03-m02.md` D17 (L582–591) | Retrieval results influence answers, marks retained | `02` §7.7 (L542) | INHERITED | Context Builder keeps fact/inference/uncertain marking. |
| R-058 | `03-m02.md` D18 (L592–603) | Degree of proactive recall | `07` §7 (P0–P6) | DEFERRED | No R4.1 clause fixes recall proactivity; it is behind stage authorisation. |
| R-059 | `03-m02.md` D19 (L606–621) | Memory Context presentation principles | `02` §7.7 (L542) | INHERITED | Same "no internal IDs, no scores" presentation rule. |
| R-060 | `03-m02.md` D20 (L622–638) | "Few but precise", inject on demand | `02` §7.6 (L483), §7.7 (L542) | INHERITED | Accuracy > relevance > completeness ordering preserved. |
| R-061 | `03-m02.md` D21 (L639–653) | Fuzzy recall and confidence handling | `01` §7.7.1 (L323) | INHERITED_WITH_REVISION | Confidence handling moves into `validation_state` + low-confidence-not-injected rule. |
| R-062 | `03-m02.md` D22 (L654–664) | Diversity control | `07` §7 `13-FUTURE-ROADMAP` (MMR listed) | DEFERRED | MMR explicitly parked in Future with a trigger condition. |
| R-063 | `03-m02.md` D23 (L665–701) | Forgetting curve / time decay, separate from long-term memory | `02` §7.5 (L369) `N` feature + λ table; `02` §7.11 (L849) | INHERITED_WITH_REVISION | Decay survives as a recency feature with half-life still TBD — parameter is EXP-DEFAULT, not frozen. |
| R-064 | `03-m02.md` D24 (L704–714) | Multi-turn reuse of already-injected context | `08` §7.14.3 (L489) | INHERITED_WITH_REVISION | Reuse rule kept but host-side cache implementation DEFERRED (U-1/U-2). |
| R-065 | `03-m02.md` D25 (L715–727) | Retrieval feedback loop | `02` §7.14 (L1080) | REJECTED | The feedback write path is **forbidden**: reinforce may only be triggered by a DSH-returned prompt memory ID, and is schema-only until E14/B04 land (`14:133`, `00:394`). |
| R-066 | `03-m02.md` D26 (L728–739) | Retrieval result lifecycle (staleness) | `03` §7.6 (L357) Outbox invalidation | INHERITED_WITH_REVISION | Staleness handled by version/`updated_at` invalidation rather than a result-lifetime object. |
| R-067 | `03-m02.md` D27 (L740–759) | Distinguish retrieval-purpose (answer vs background) | `02` §7.7 (L542); `14` §7.9 (L~) | INHERITED | Purpose separation survives as context-packaging rules. |

## 4. M03 — 记忆提取与候选生成 (12 clauses)

| # | Legacy anchor | Clause | Current destination | Class | Justification |
|---|---|---|---|---|---|
| R-068 | `03-m03.md` §1 (L15–29) | Candidate ≠ fact; M03 never writes authority | `09` §7.1 (L177), §7.2 (L201) | INHERITED | "Three-no" pipeline principles restate the boundary. |
| R-069 | `03-m03.md` §2.1–2.3 (L33–69) | Input contract, batch boundary, async batch processing | `09` §7.5 (L257), §7.3 (L214) | INHERITED | Input/batch rules become InputNormalizer + run model. |
| R-070 | `03-m03.md` §3 (L71–80) | Memory Share boundary is not M03's job | `04` §7.9 (L224) | INHERITED | Same delegation to M05. |
| R-071 | `03-m03.md` §4 (L82–103) | Seven candidate types | `01` §7.2 (L179) | INHERITED | Same seven values; they are now `semantic_type`-adjacent rather than a standalone `type`. |
| R-072 | `03-m03.md` §5 (L105–122) | `claim_type` three values, source_refs mandatory | `13` §7.6 (L370) | INHERITED_WITH_REVISION | Three-value `claim_type` replaced by five-value `source_type` with explicit/inferred separation. |
| R-073 | `03-m03.md` §6.1–6.2 (L124–162) | Five-dimension score R/P/E/T/U with fixed weights | `02` §7.8 (L609); `15` §7 (L103) | INHERITED | Formula carried; weights move into the parameter registry as EXP-DEFAULT. |
| R-074 | `03-m03.md` §6.3–6.5 (L164–243) | Score explanation, hard constraints, type differences | `09` §7.7 (L313), §7.18 (L652) | INHERITED | Hard-constraint-before-total-score rule preserved. |
| R-075 | `03-m03.md` §7.1–7.3 (L244–279) | Three-way routing and staging promotion | `09` §7.21 (L740) | INHERITED | review/observe/reject becomes the six-state candidate machine; "staged candidates never auto-promote" kept. |
| R-076 | `03-m03.md` §8 (L280–295) | Duplicate expression and evidence accumulation | `09` §7.16 (L599) | INHERITED | DuplicateDetector stage. |
| R-077 | `03-m03.md` §10 (L312–343) | Candidate data structure | `01` §7.12 (L538) | INHERITED_WITH_REVISION | Candidate is declared a **FUTURE object** in R4.1 while the Pipeline writes CandidateDraft (`09` §8.4) — two candidate shapes coexist. |
| R-078 | `03-m03.md` §11 (L344–371) | Adjustable interface surface | `09` §7.24 (L806) | INHERITED | Interface constraints against M01/M02/M04/M05 restated. |
| R-079 | `03-m03.md` §12 (L372–397) | Model interface and failure handling | `10` §7 (L155) | INHERITED_WITH_REVISION | Single model call becomes L1/L2/L3 tiering with degradation. |
| R-080 | `03-m03.md` §14 (L423–436) | Parameters not frozen (thresholds, TTL, batch size) | `15` §7 (L103) | INHERITED | All seven open items appear as registry entries with EXP-DEFAULT status. |

## 5. M04 — 记忆整合与视图维护 (15 clauses)

| # | Legacy anchor | Clause | Current destination | Class | Justification |
|---|---|---|---|---|---|
| R-081 | `03-m04.md` §2.1 (L66–136) | Three-layer model (Evidence → Fact/Event → Observation) | `01` §7.1 (L157), §7.8 (L402), §7.11 (L517) | SUPERSEDED | V2 replaces the three-layer model with MemoryUnit + Event + EmotionContext + PersonProfile; `00` §0.6 (L75–80) forbids keeping both, and Observation becomes a FUTURE derived object. |
| R-082 | `03-m04.md` §2.2 (L138–163) | Cue Anchors as derived retrieval entries | `11` §7 (8 anchors); `01` §7.16 (Cue table) | INHERITED_WITH_REVISION | Free-form cue anchors become an eight-anchor taxonomy with assignment rules (Pipeline stage 9). |
| R-083 | `03-m04.md` §3.1 (L166–178) | recorded_at vs valid_from/valid_to | `01` §7.14 (L625) | INHERITED_WITH_REVISION | Three time fields extended to six with explicit semantics. |
| R-084 | `03-m04.md` §3.2–3.4 (L179–229) | Current view, version publication, turn snapshot | `01` §7.15 (L642); `03` §7.8 (L512) | INHERITED_WITH_REVISION | View publication survives as version discipline; per-turn view pinning is no longer stated as its own clause. |
| R-085 | `03-m04.md` §4.1 (L232–256) | Pluggable Decision Provider | `02` §7.9 (L685) | INHERITED | Provider pluggability kept; C40 permission engine explicitly excluded from pluggability (`07:362`). |
| R-086 | `03-m04.md` §4.2–4.4 (L257–306) | Update decision, Observation formation, correction | `02` §7.9 (L685); `04` §7.9 (L224) | INHERITED_WITH_REVISION | Correction path preserved; Observation formation moves behind Pipeline stages 12–13. |
| R-087 | `03-m04.md` §5.1–5.3 (L307–358) | Relation types, dependency maintenance, Dream entry | `01` §7.10 (L487); `02` §7.12 (L908) | INHERITED_WITH_REVISION | Relation model carried; Dream entry becomes ALG-11 FUTURE. |
| R-088 | `03-m04.md` §6.1–6.5 (L359–438) | Rule-first / LLM-enhancement, Model Gateway, tiered routing | `10` §7 (L155); `00` §5 D-AP-02 | INHERITED | Gateway-as-sole-entry is a ratified decision (D-AP-02), i.e. inherited with added force. |
| R-089 | `03-m04.md` §7.1–7.6 (L441–541) | Unified task queue, priorities, DAG, checkpoints | `03` §7.9 (L528) | INHERITED | Task protocol carries queue, DAG and checkpoint semantics. |
| R-090 | `03-m04.md` §8.1–8.3 (L544–574) | View production: trigger, incremental, atomic publish | `01` §7.15 (L642); `03` §7.8 (L512) | INHERITED_WITH_REVISION | Atomic publish preserved as transaction boundary + version bump; thresholds moved to config. |
| R-091 | `03-m04.md` §9.1 (L579–593) | Four independent status dimensions | `01` §7.7 (L319 three state machines) | INHERITED_WITH_REVISION | Four dims → three axes (`validation_state`, lifecycle `status`, `availability_state`) **plus** `memory_tier`; the mapping is not one-to-one and the legacy dimension "审核状态" now lives on Candidate. The replacement is itself incomplete: no legal `memory_tier` transition table exists anywhere in the corpus (**OI-R-07**, `SECOND-ROUND-FINAL-ISSUE-REGISTER.md:98`), so what the legacy clause guaranteed operationally (four explicit dimensions) is weaker in R4.1 than the clause it replaced. |
| R-092 | `03-m04.md` §9.3 (L604–641) | Deletion responsibility split, no physical delete by M04 | `04` §7.9 (L224), §9.3 (L351) | INHERITED | Five-party split restated as a propagation diagram. |
| R-093 | `03-m04.md` §10.1 (L670–699) | Controlled autonomous expansion (DSH active recall) | `14` §7 (L189) | INHERITED_WITH_REVISION | Read-only expansion survives as formal API calls with call-count budgets. |
| R-094 | `03-m04.md` §16 (L947–962) | Three-layer quality assessment | `02` §7.13 (L958) | INHERITED | Data/semantic/usage layers map onto the evaluation algorithm. |
| R-095 | `03-m04.md` §19 (L1028–1093) | Phase 1–5 implementation plan | `07` §7 (P0–P6) | INHERITED_WITH_REVISION | Five phases re-cut into P0–P6 with stage authorisation; Phase 5 items move to Future. |

## 6. M05 — 权限与访问控制 (12 clauses)

| # | Legacy anchor | Clause | Current destination | Class | Justification |
|---|---|---|---|---|---|
| R-096 | `03-m05.md` §1 (L16–39) | M05 is the single permission authority | `04` §7.1 (L90); `00` §0.4 (L43) | INHERITED | Ratified as C40 not-pluggable. |
| R-097 | `03-m05.md` §3 (L88–131) | Three scopes private/group/shared, reference-not-copy for Shared | `04` §7.5 (L152) | INHERITED_WITH_REVISION | Five scopes + a three-value external `visibility`; the "reference authorisation, never copy" rule survives at `04` §7.11. |
| R-098 | `03-m05.md` §2.3 (L76–86) | Five identity fields are not interchangeable | `04` §7.6 (L167) | INHERITED | Same five identifiers, same non-substitution rule. |
| R-099 | `03-m05.md` §4 (L133–163) | Owner identity and owner powers, group admin ≠ data owner | `04` §7.2 (L105) | INHERITED | Role table carries owner/admin separation. |
| R-100 | `03-m05.md` §5 (L165–183) | Seven operation permissions | `04` §7.4 (L125) | INHERITED_WITH_REVISION | Seven operations → six actions × six resources × five scopes × seven roles. |
| R-101 | `03-m05.md` §6 (L185–238) | Memory Share reuses existing `qq_memory_share_request` approval flow | `04` §7.9 (L224) | INHERITED_WITH_REVISION | Share semantics kept; the **legacy approval-flow reuse is not restated** — R4.1 defines authorisation data, not the QQ-Bridge approval workflow, which leaves the migration of existing approval records unspecified. |
| R-102 | `03-m05.md` §7 (L239–262) | Immediate revocation, async cleanup, revoke-before-query ordering | `04` §9.2 (L337); `05` §7.11 (L243) | INHERITED | Propagation diagram + "authority state beats async deletion" rule. |
| R-103 | `03-m05.md` §9 (L280–314) | Derived-data permission inheritance, strictest-intersection | `04` §7.11 (L257) | INHERITED | Eleven derived object classes, same intersection rule. |
| R-104 | `03-m05.md` §10.1–10.2 (L315–366) | Permission request/response structure | `03` §7.5 (L342) | INHERITED_WITH_REVISION | Six-action Permission API contract replaces the ad-hoc request struct. |
| R-105 | `03-m05.md` §10.4 (L375–406) | Same-request decision reuse, never cross-request | `09` §7.19 (L682); `14:707` | INHERITED | Reuse window and the seven re-check triggers survive. |
| R-106 | `03-m05.md` §12 (L454–471) | Fail-closed, no existence leakage in denials | `04` §7.7 (L181) | INHERITED | Missing input ⇒ deny; denial must not reveal existence. |
| R-107 | `03-m05.md` §13.1 (L475–489) | M05 decides, M01 persists permission records | `04` §7.x; **B-058** open | UNRESOLVED | No authoritative permission/share table exists in M01's schema; recorded as P0 in `unified-set/02-ARCHITECTURE-OVERVIEW.md:91` and unresolved in `design-package` as well. Persistence duty is restated, the storage object is missing. |
| R-108 | `03-m05.md` §14–§17 (L491–603) | Audit fields, high-impact second confirmation, error matrix, test set | `04` §7.10 (L251), §12 (L395); `05` §7.2 (L94) | INHERITED_WITH_REVISION | Audit fields carried; eleven permission test scenarios replace the legacy list. |

## 7. Rows that need a decision (the 20 non-inherited outcomes)

### 7.1 SUPERSEDED (9)

| Row | Legacy anchor | Legacy clause | Superseded by | Consequence if the legacy intent is still required |
|---|---|---|---|---|
| R-010 | `03-m01.md` §4.1 (L339–353) | Three co-equal retrieval channels | Five-stage retrieval + 8-anchor routing (`00` §0.1, L16) | Any implementation written to D14 will emit `structured` as a ranked channel and violate the frozen `fts`+`vector` rule. |
| R-081 | `03-m04.md` §2.1 (L66–136) | Evidence → Fact/Event → Observation | MemoryUnit / Event / EmotionContext / PersonProfile (`00` §0.6, L75) | Legacy `Observation`/`Fact` names must not appear in new schemas; `01` §7.11 keeps Observation only as FUTURE. |
| R-109 | `03-m03.md` §4 (L82–103) | Seven-value `type` as the classification axis | `semantic_type`(6) + `semantic_tags` (`00` §0.6, L77; `DD-AUTHORITY-V2` §4.1) | Two classification schemes remain live in `02` §7.6 — see SC-04. |
| R-110 | `03-m01.md` §2.2 (L135–170); `02` §7.6 (L630) | `content_kind` eight-value axis | abolished by `00` §0.6 (L76) | `02:630` still requires "type 七类 + content_kind 八类" — a live contradiction inside the current set. |
| R-111 | `03-m01.md` §3.2 D7 (L257–300); `03-m04.md` §9.1 (L579–593) | Single `lifecycle_state` enum | `memory_tier` + `status` + `candidate.status` (`00` §0.6, L75) | Any legacy table carrying `lifecycle_state` is invalid in R4.1. |
| R-112 | `03-m02.md` D6 (L387–401) | Single `status` field on a retrieval result | `retrieval_status` / `permission_status` split (`00` §8 table, L542) | "No result" and "denied" stay conflated. |
| R-113 | `03-m05.md` §3 (L88–131) | Three scopes as the only scope axis | five scopes + external `visibility` (`04` §7.5, L152) | Code written to three scopes will silently under-restrict. |
| R-114 | `03-m03.md` §12 (L372–397) | One model call per batch | L1/L2/L3 tiered calling (`10` §7, L155) | Legacy cost/latency assumptions do not hold. |
| R-115 | `03-m04.md` §7.1 (L443–458) | Task identity `reflection` | `integrity_reflection` + Dream separation (same file, L451/L458) | Task-log joins across generations will mis-bucket. |

### 7.2 REJECTED (4)

| Row | Legacy anchor | Legacy clause | Why rejected | Rejecting anchor |
|---|---|---|---|---|
| R-065 | `03-m02.md` D25 (L715–727) | Retrieval feedback loop writing back to weights | Feedback writes forbidden; reinforce is schema-only and only DSH-returned prompt IDs may trigger it | `02` §7.14 (L1080); `14:133`; `00:394` |
| R-116 | `03-m02.md` D23 (L665–701) | `last_retrieved` / hit count as a positive ranking or decay weight | Explicitly forbidden as a positive weight | `01:315`; `11:633` |
| R-117 | `03-m01.md` §7.2 D33 (L819–838) | Models/plugins may adjust configured rules | Safety hard constraints cannot be model-modified | `03` §7.11 (L568); `15:333` |
| R-118 | `03-m05.md` §11.1 (L412–434) | LLM reads memory content after permission check | LLM must never reach SQLite/LanceDB; reads go through Memory API only | `00` §7.1 (L270); `09:690` |

### 7.3 DEFERRED (7)

| Row | Legacy anchor | Legacy clause | Deferred to | Blocking condition |
|---|---|---|---|---|
| R-031 | `03-m01.md` §6.7 D30 (L783–800) | Capacity/performance targets | P0–P6 gate (`07` §7) | No measured data; all thresholds EXP-DEFAULT |
| R-048 | `03-m02.md` D8 (L421–442) | Recall cache design | Implementation stage | Ledger U-1/U-2 freeze cache schema |
| R-058 | `03-m02.md` D18 (L592–603) | Proactive-recall degree | P3+ | No clause fixes the level |
| R-062 | `03-m02.md` D22 (L654–664) | Diversity / MMR | Future roadmap | Trigger-conditioned |
| R-119 | `03-m01.md` §4.5 D18 (L392–408) | Advanced ANN | Future roadmap | FLAT must be measured first |
| R-120 | `03-m02.md` D4 (L98–108) | Cross-encoder rerank | Future roadmap | Rule ranking must be measured first |
| R-121 | `03-m01.md` §6.3 D13 (L710–723) | Encryption / key management | Post-v1 | No key infrastructure in scope |

### 7.4 UNRESOLVED (4 rows — the real MC-AUD-017 debt)

| Row | Legacy anchor | Legacy clause | What is missing | What would close it |
|---|---|---|---|---|
| R-009 | `03-m01.md` §3.4 D36 (L324–337) | Read consistency and snapshot | No read-consistency or snapshot-isolation clause anywhere in 00–15; WAL appears only inside the failure matrix (`05` §7.2, L106). | One paragraph in `03` §7.8 or `01` §7.15: is the snapshot per request or per turn; which reads may observe uncommitted Outbox effects. |
| R-022 | `03-m01.md` §5.3 D10 (L594–624) | Physical directory layout | No directory contract (data/backup/runtime/logs/temp/manifest). `05:49` declares D37 but not D10. | An annex in `05` §7.3 or `07` naming the layout and the migration constraints. |
| R-039 | `03-m01.md` §8.3 D32 (L943–958) | First-run initialisation | No bootstrap sequence and no "existing DB must not be re-initialised" rule. | A clause in `05` §7.6 or `03` §7.1 covering cold start and idempotent init. |
| R-107 | `03-m05.md` §13.1 (L475–489) | M05 decides, M01 persists permission records | No authoritative permission/share table exists in the M01 schema (B-058 in the other set; unregistered here). | A `permissions` / `share_authorization` table in `01` §7.16 plus a `permission-context` schema revision. |

## 8. What this matrix does **not** claim

- It does not re-derive the legacy documents' own internal consistency; `03-m01.md` §9 and `03-m02.md` §12 revision logs were read but not scored.
- It does not treat `unified-set` as a destination. Three of the four UNRESOLVED rows are *also* unresolved there; R-107 is explicitly open there as B-058 (`unified-set/02-ARCHITECTURE-OVERVIEW.md:91`).
- It does not assert that INHERITED rows are *implemented*. Inheritance is a documentation relationship only; per the mandate's SRD-001 axes, `PRODUCT_IMPLEMENTATION_AUTHORIZATION` remains NOT_AUTHORIZED for this task.
