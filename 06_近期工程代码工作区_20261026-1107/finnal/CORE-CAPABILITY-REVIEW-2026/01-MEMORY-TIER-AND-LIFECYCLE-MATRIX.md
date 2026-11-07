# Memory Tier and Lifecycle Matrix — Memory Core R4.1

Round: MC-CORE-CAPABILITY-REVIEW-01 · Agent A · Read-only · 2026-11-06

## STATUS: COMPLETE

> Verification: every EXISTING DESIGN claim is anchored to a baseline file + section (§14.1–14.4 lists
> what I opened and the two measurement caveats). Machine checks and their raw output live in
> `EVIDENCE\a\` (`a-machine-checks.json`, `a-checks2.json`). No corpus file was modified.

---

## 1. Tier verdict (headline)

**VERDICT: `memory_tier` is a label axis, not a memory tier system.** Short-term and long-term
memory are **not** real subsystems in this corpus: they are three enum values (`L1`/`L2`/`L3`) on
one field, with a definition (§2.5), a *non-binding* semantic-type prior (§7.12), and five
prohibitions — and **zero** transition edges, zero thresholds, zero writers, zero stores, zero
retention jobs, zero tests.

Three plain-language consequences:

1. **There is no short-term → long-term loop.** The only sanctioned route (L1→L2→L3 through
   Evidence → Validation → Review, executed by M04) has no algorithm, no threshold and no
   executor step; the only *explicit* tier rules in the corpus are five `MUST NOT`s that forbid
   the automatic version of that route.
2. **Long-term has no independent management mechanism.** L3 shares the same row, table, indexes,
   write path, permission path and retrieval path as L1 and L2. The only long-term-shaped
   structure that exists — `PersonProfile` — is explicitly forbidden to be an independent store.
3. **A long-term memory cannot be traced to the original input at the storage layer.** The
   Evidence *specification* carries `original_text`; neither the only SQLite DDL nor the only
   Evidence machine contract has a column/property for it.

The design is **not** guilty of equating tier with semantic category *in its normative text*
(three explicit orthogonality MUSTs), but the only *assignment guidance that exists anywhere* is a
`semantic_type → tier` matrix, and two subsystems hard-wire tier to non-lifecycle decisions
(`PersonProfile → (L3, ACTIVE)`; `AUTO_ACCEPT → L1/L2 only`). So the orthogonality is asserted and
then contradicted by every executable trace of it.

---

## 2. Scope, method, evidence levels

### 2.1 What I read (all pages, not excerpts, except where noted)

| # | File | Use |
|---|---|---|
| 1 | `DD-AUTHORITY-V2.md` (465 L, full) | tier/status/validation enumerations §2.5–§2.8; MemoryUnit field table §3.1; Evidence §3.2; Pipeline §5; 9 APIs §7; conflicts §12 |
| 2 | `00-总体架构与设计原则.md` §3.1, §5, §7.4, §7.5, §7.6, §7.7 (L120–413) | 时间尺度三层 + 晋升规则 §7.5; Session Memory boundary §7.7; X-2 ruling §5 |
| 3 | `01-记忆本体与数据模型.md` §7.6 (L276–320), §7.7 (L352–381), §7.9 (L462–485), §7.15–§7.18 (L642–755) | the only DDL (§7.16), lifecycle matrix T-L-01…09, operation 九种, migration maps §7.17 |
| 4 | `02-核心算法规范.md` ALG-04 六特征 (L396–443), ALG-08 (L685–759), ALG-10 (L849–906), ALG-11 (L920–950) — read from the NUL-sanitised copy `AUDIT-FORENSIC-2026\16-RAW\b\scratch\02-sanitized.md` (the original is rejected as binary; that copy is internally consistent but its line numbers diverge from some quoted in the forensic docs — see §14.4) | ranking features; consolidation; lifecycle algorithm; maintenance scheduler |
| 5 | `09-记忆摄取Pipeline.md` §6, §7.1, stage table (L241), stage 4 (L347), stage 6 fields (L422–443), stage 16 (L702–738), candidate 6-state (L740–764) | where `lifecycle_candidate` is produced; what M04's decision vocabulary is |
| 6 | `11-MemoryAnchor与语义分类.md` §7.1 (L172–179), §7.12 (L493–519), §7.14 (L543–564), §7.16 (L581–601), §7.17 (L603–632), §12 测试 (L883–930) | the 3×6 matrix, the 5 MUST rules, the 7 read capabilities, tier in sorting/filtering |
| 7 | `12-人物画像整合.md` §7.6 (L319–327), §7.7 (L385–390) | profile versioning; `lifecycle ← (L3, ACTIVE)` |
| 8 | `13-Evidence与EmotionContext.md` §7.3 (L240–270), §7.4 (L286–345) | Evidence field table incl. `original_text`; reverse-trace flow |
| 9 | `05-可靠性与运维规范.md` (full scan), `06-测试与评估规范.md` (full scan), `15-参数注册表.md` (full scan) | absence checks: retention jobs, tier tests, tier parameters |
| 10 | `contracts/state-machines/state-machines.md` (165 L, full) | SM-1…SM-8 |
| 11 | `contracts/schemas/*.json` (20) + `contracts/config/*.json` (11) = the 31 machine contracts | property-level checks |
| 12 | `contracts/error-codes.md`, `contracts/api/api-contracts.md` | error-code surface |
| 13 | `AUDIT-FORENSIC-2026\03-ARCHITECTURE-FACTS.md` §E (data ownership), §H (read path), §I (lifecycle), `09-CONFLICT-REGISTER.md` §C (FC-013…FC-017) | prior facts; not re-audited |

**Not examined**: `04`, `07`, `08`, `10`, `14` beyond targeted greps (they do not define tier
lifecycle; `14:601` and `08:3` only mention `memory_tier` as a pass-through field); the legacy
`03-m01…m05` frozen set; `E:\[PROJECT]\R4.1-UNIFIED-MEMORY-CORE` (has no L1/L2/L3 at all — confirmed by
`03-ARCHITECTURE-FACTS §B.3`, so it cannot answer this question).

### 2.2 Evidence levels

- **E1** = document text, cited by file + section (+ line).
- **E2** = machine-checked over the corpus: `EVIDENCE\a\a-machine-checks.cjs` → `a-machine-checks.json`,
  `a-checks2.cjs` → `a-checks2.json`. No corpus file was modified.

### 2.3 The one measurement I was asked to verify, and its result

See §9. Short answer: **the 0-edge measurement is confirmed exactly, and the gap is *not*
transcription** — §7.12 contains no edge to transcribe.

---

## 3. The four questions, answered explicitly

### Q1 — Can short-term information really be retained long-term?

**No.** Not "not yet implemented" — *no mechanism is specified end to end*, and five explicit
`MUST NOT`s remove the only cheap paths.

| Claim | Anchor | Status |
|---|---|---|
| Promotion requires `Evidence → Validation → Review` | `00 §7.5` L377 | **EXISTING DESIGN**, one sentence, no steps/threshold/executor |
| Final tier is decided by M04; M03 only proposes `lifecycle_candidate` | `11 §7.12` rule 1, L510; `09` stage 6 L422/443 | **EXISTING DESIGN** (responsibility only) |
| `L1` MUST NOT auto-promote to `L3` (X-2: Session end discards short-term state; L1 仅预留) | `11 §7.12` rule 5, L514; `09` L134; `00` §5 X-2, L214 | **EXISTING DESIGN** — a prohibition |
| `session_only` input MUST NOT auto-promote to long-term memory | `01` §7.6 L234; `09` L134 | **EXISTING DESIGN** — a prohibition |
| Session Context defaults to *no* promotion; discarded at session end | `00 §7.5` L370 | **EXISTING DESIGN** — a prohibition |
| Session Memory module: "不自动晋升长期记忆" | `00 §7.7` L407 | **EXISTING DESIGN**, module status **FUTURE** |

**DESIGN COMPLETENESS: `MISSING`.** There is no algorithm that takes an L1 record and produces an
L2 or L3 record: no triggering event, no sufficiency threshold, no executor step, no idempotency
key, no failure path, no test criterion, and no state-machine edge (§9). The nearest thing to a
condition — "需 Evidence + Validation + Review" — is a *gate ordering*, not a promotion rule; it
says what must have happened, not when or by whom or on what evidence count.
`ALG-10` (the lifecycle algorithm, `02 §7.11` L849–906) has an authorization matrix with four
groups (`T-V-*` validation, `T-L-*` lifecycle, `T-A-*` availability, deletion) and **no tier
group**; its config list (L898) has `lifecycle.expiry_policy.{type}` and
`lifecycle.archive_after_days.{type}` but not a single tier parameter.

**RECOMMENDED ADDITION (NEW PROPOSAL, not in corpus):** a `TierTransition` algorithm
`promote(memory_id, from_tier, to_tier, basis) → {applied, version, audit_id}` with (i) a decidable
promotion predicate per edge, (ii) M04 as the only writer, M01 as the only executor, (iii) a
`tier_change` provenance operation, (iv) idempotency on `(memory_id, from, to, version)`, and
(v) an acceptance test per edge. Details in §8 / §11 (A-C001).

### Q2 — Does long-term memory have an independent management mechanism, or just another enum value?

**Just another enum value.** Independent means: own store, own index, own writer, own retention
rule, own retrieval path. L3 has none of these:

| Independence test | L3 | Anchor |
|---|---|---|
| Own table or column | **No** — `memory_tier` occurs **0** times in the 13-table DDL; `memory_objects` indexes are `(tenant_id,scope,lifecycle_state)`, `(subject_id,type)`, `observed_at`, `updated_at` | E2; `01 §7.16` L666 |
| Own derived index entry | **No** — `search-document.schema.json` has `lifecycle_state`, no `memory_tier` | E2 |
| Own writer | **No** — no data-ownership row for `memory_tier` anywhere (compare the 14-row ownership matrix in `03-ARCHITECTURE-FACTS §E`) | E1/E2 |
| Own retention rule | **No** — the only expiry machinery is per-`type` decay (`02` λ table L404–414) and per-record `expires_at` (`01` L299, T-A-01 L395); no tier rule, and `05` has no retention job | E1 |
| Own retrieval path | **No** — tier is explicitly *not* a read gate (`11 §7.16` L597) and not a ranking feature (ALG-04's six features R/E/N/T/S/P key on `memory.type`, L396–402) | E1 |
| Own maintenance job | **No** — `Dream` (the only maintenance scheduler) is FUTURE, report-only, all tasks default off (`02 §7.12` L940, L950) | E1 |

The closest thing to an independent long-term structure is `PersonProfile`: separate object
(`DD-AUTHORITY-V2 §3.5` L257–269), own version (`profile_version`, `12 §7.6` L319–327), own read API
(`memory.getProfile`). But it is (a) keyed on `(subject_id, section)`, **not** on tier, and
(b) explicitly forbidden to be independent: "**MUST NOT** 拥有独立于 MemoryUnit 的权威写入路径"
(`12` L187). So the corpus has a consolidated *view*, not a long-term *store*.

**DESIGN COMPLETENESS: `CONCEPT_ONLY`.** `DD-AUTHORITY-V2 §2.5` (L89–94) gives three prose
definitions and one iron law. Nothing downstream distinguishes L3 from L1 except prose.

### Q3 — Is information lost during a tier transition?

There is no transition, so today the answer is "not measurable"; but three *provable* losses sit
exactly on the tier boundary, and a fourth is structural:

1. **Provable loss at the candidate → memory boundary (E2).** Stage 6 makes `lifecycle_candidate`
   a **required** candidate field (`09` L422, L885). The only candidate machine contract,
   `contracts/schemas/candidate.schema.json`, has `"additionalProperties": false` and **no**
   `lifecycle_candidate` property (E2: `has_lifecycle_candidate: false`). The classifier's tier
   rationale — the only tier evidence M03 ever produces — is therefore rejected at the contract
   boundary. It can never reach M04, which is supposed to make the final tier decision.
2. **Unbounded loss at L1 disposal.** "Session 结束即丢弃" (`00 §7.5` L370) names no operation:
   `status` has no `DISCARDED`, SM-1 has no session-end edge, `operation` 九种 has no `discard`
   (`01` L471–484), and the deletion protocol (`SM-7`) has no "expired" entry state. An
   implementation must either delete (irreversible, `SM-7` L137) or leave orphan rows forever.
3. **Provenance loss on promotion.** Promotion is supposed to carry evidence (`00` L377). Evidence
   rows can bind to memories (`memory_evidence_links`, `01` L669) but cannot hold the original
   text (see Q4/A-C004), so a promoted memory can only point at a host session locator — and the
   host history is declared *not* an authoritative layer (`00` L347).
4. **Structural loss if a naive implementation is written.** Because there is no demotion and no
   expiry, the natural implementation of "tier" is `UPDATE memory_objects SET memory_tier=?`,
   which silently rewrites the field with no version bump rule (`01 §7.15` defers version
   semantics to OI-01-05), no audit operation, and no M05 hook — i.e. a tier change would be the
   only state change in the system with no provenance. That is a *loss of change history*, not of
   content.

### Q4 — Are the tiers wrongly equated with semantic categories?

**Normatively no; operationally yes.**

*Normative text is clean (EXISTING DESIGN):*
- `DD-AUTHORITY-V2 §2.5` L89–95: "与 semantic_type 正交"; "铁律：tier ≠ 重要程度".
- `11 §7.12` L495: "`memory_tier` 与 `semantic_type` **正交**"; L511 forbids tier-driven read
  priority; L512 forbids treating L1 as discardable; L513 forbids promoting on "importance".
- Static check (E2): the memory-unit schema has **no** `allOf` clause coupling `memory_tier` to
  `semantic_type`/`status`; all 12 combos validate (`09-CONFLICT-REGISTER FC-014` reports the same).

*Operational reality contradicts it:*
1. **The only assignment guidance in the corpus is a category function.** `11 §7.12` L499–506 is a
   6×3 `semantic_type → tier` matrix (`PERSON → L3`, `EVENT → L2`, `EMOTION_CONTEXT → L1/L2`…).
   It is duplicated in prose twice more: `11 §7.1` L172–179 ("典型 tier" column) and `11 §7.18`
   intent table L668 ("tier 倾向"). `09` stage 3 L322 repeats the pattern ("稳定人物信息 … 倾向
   L3"). So the *only* thing a classifier can use to pick a tier is the semantic category — the
   exact conflation the iron law forbids.
2. **Tier is hard-wired to a derived view.** `12 §7.7` L390: profile rows **SHOULD** map
   `lifecycle ← (L3, ACTIVE)`. Every profile row is L3 by construction, so L3 stops meaning
   "long-term" and starts meaning "this is a consolidated row".
3. **Tier gates a trust decision.** `09 §7.21` L755: `AUTO_ACCEPT` "只允许用于 `L1`/`L2` 且
   `visibility=owner_private`" — i.e. tier is used as a *confidence/risk* proxy, which
   `validation_state` exists to express.
4. **Three-level vocabulary collision.** `L1`/`L2`/`L3` denote **three unrelated hierarchies**:

   | Token | Hierarchy A: memory tier | Hierarchy B: LLM call layer | Hierarchy C: time scale |
   |---|---|---|---|
   | `L1` | 临时记忆 (`V2 §2.5`) | 轻量理解层 (`V2 §6`; `10` 全书) | `Session Context` (`00 §7.5`) |
   | `L2` | 活跃记忆 | 强语义层 | `Working Memory` |
   | `L3` | 长期记忆 | 聊天模型层 (+ `L3 事件`/`Event-L3` injection reports, `08`) | `Long-term Memory` |

   Both hierarchy A and B live inside **one file**: `09` L241/L347/L924 use `L1/L2` for LLM layers
   while `09` L422/L443/L885 use `L1/L2/L3` for `lifecycle_candidate`. Corpus-wide counts (E2):
   `memory_tier` appears **11 times** across the 18 canonical documents (~1.06 MB), while the bare
   token `L1/L2/L3` appears 400+ times, of which the overwhelming majority mean the LLM layer
   (`10-LLM调用体系.md` alone: 161). Hierarchy C (`00 §7.5`) is never mapped to hierarchy A — no
   document states that `Session Context ≡ L1`, `Working Memory ≡ L2`, `Long-term ≡ L3`, even
   though the definitions are near-verbatim (`L1` = "当前对话/短期任务"; Session = "当前主题/当前任务").

**DESIGN COMPLETENESS for Q4: `CONFLICTED`** — the orthogonality is asserted in three places and
violated in four (assignment, profile mapping, AUTO_ACCEPT gate, vocabulary collision).

---

## 4. Mandatory matrix

Notation: `∅` = does not exist anywhere in the corpus; `—` = not applicable; **I** = intended by a
document but without an executable mechanism; **E2** = machine-verified.

| 层级 | 作用 | 主要数据 | 存储位置 | 写入者 | 读取者 | 进入条件 | 退出条件 | 晋升条件 | 降级条件 | 保留策略 | 检索策略 | 当前定义是否完整 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **L1 临时记忆** | "当前对话/短期任务/未确认临时状态/待处理候选；可快速过期；不自动视为长期事实" (`V2 §2.5` L91) | `CURRENT_STATE`(●), `EMOTION_CONTEXT`(●), `PERSON/PREFERENCE/RELATIONSHIP/EVENT`(○) — prior only (`11 §7.12` L501–506) | ∅ **no store**: `session_state` is a FUTURE table that "**MUST NOT** 在 P0–P3 提前创建" (`01 §7.16` L679); no `memory_tier` column in any DDL (E2) | **∅ none.** M03 may only emit `lifecycle_candidate` (`09` L422); M04 decides (`11` L510) but M04 = P4, unauthorised (`V2 §10`); M01 requires the field at insert (`V2 §3.1` L190) with no source | M02 (as a memory row, if it existed); no tier-specific reader; tier is explicitly **not** a read gate (`11 §7.16` L597) | I: created by ingest as a candidate with `lifecycle_candidate=L1` (`09 §7.6`); but no rule says *when* L1 vs L2 is chosen — "矩阵只是默认倾向" (`11` L510) | ∅ **undefined**: "Session 结束即丢弃" (`00 §7.5` L370) names no operation, no state, no job; `X-2` says "L1 仅预留" (`00 §5` L214) | I: L1→L2 then L2→L3, gated by Evidence+Validation+Review (`00` L377) — no condition, no threshold, no executor step | **∅** no demotion path exists anywhere in the corpus (E1: 0 hits for 降级/降层 as tier terms) | ∅ "可快速过期" (`V2` L91) with no TTL, no sweeper, no policy; `05` has no retention job; the only expiry machinery is per-`type` decay + per-record `expires_at` (`02` L404–414, `01` L299) | Inherits the one global retrieval path; tier MUST NOT gate reads (`11` L597) and MUST NOT be a sort weight (`11` L632); `read-request.schema.json` cannot express a tier filter (E2) | **NO** — name + one sentence + two prohibitions |
| **L2 活跃记忆** | "推进中计划/阶段性任务/近期重要事件/阶段性人物状态/关系变化；需定期维护与过期处理" (`V2 §2.5` L92) | `CURRENT_STATE`(●), `EVENT`(●), `PERSON/PREFERENCE/RELATIONSHIP`(○), `EMOTION_CONTEXT`(○) (`11 §7.12`) | ∅ same row/table as L1 and L3 (or rather: the row that cannot carry the field) | ∅ same as L1 | same as L1 | I: initial tier from the ingest prior, then M04 | ∅ undefined; "需定期维护与过期处理" (`V2` L92) has **no** maintenance job — the only maintenance scheduler (Dream) is FUTURE and report-only (`02 §7.12` L940/L950) | I: →L3 by M04 ("最终 tier MUST 由 M04 整合裁决") — no threshold; L513 forbids promoting on "importance" without saying what *is* sufficient except "稳定/明确/多次/用户明确要求" | **∅** none | ∅ "定期维护" undefined; no cadence, no owner, no parameter in `15` (E2: `lifecycle.*` absent from the registry) | same as L1 | **NO** |
| **L3 长期记忆** | "稳定人物描述/明确自我描述/长期偏好/长期关系/重要经历/长期目标/已验证长期特征；保留证据与版本；不因新信息无条件覆盖" (`V2 §2.5` L93) | `PERSON`/`PREFERENCE`/`RELATIONSHIP`(●), `EVENT`/`CURRENT_STATE`(○), `EMOTION_CONTEXT`(—) (`11 §7.12`) | ∅ no column, no index, no separate table (E2: 0/13 tables, 0/31 contracts besides the unit schema); the *derived* long-term view is `PersonProfile`, which MUST NOT be an independent store (`12` L187) | ∅ intended M04 + M01 transaction; in the authorised P0–P3 scope (M01+M02 only, `V2 §10`) **nobody** | M02 → Context Builder → Host; `memory.search`+`tier` filter is claimed as read capability #4 (`11 §7.16` L590) but the request contract has no tier field (E2) | I: promotion only (`11` L513 "MUST 有稳定/明确/多次/用户明确要求等依据") | ∅ undefined (no archive-by-tier, no decay-to-L2) | — terminal layer (`V2` L93) | **∅** no L3→L2 path; "不因新信息无条件覆盖" (L93) implies a *conditional* demotion that is never written | ∅ "保留证据与版本" — versions exist (`memory_versions`, `01` L667), evidence **links** exist (`memory_evidence_links`, L669), but the original text has no column (A-C004) | same as L1; used as a filter hint only in prose (`11` L590, L617, L723) | **NO** |

**Reading of the matrix.** Take the five lifecycle columns only (进入条件 / 退出条件 / 晋升条件 /
降级条件 / 保留策略) × 3 tiers = **15 cells**:

- **9 are `∅`** — no definition of any kind: all three 退出条件, all three 降级条件, L1/L2/L3 保留策略
  (the retention cells exist only as one-word intentions, "可快速过期" / "定期维护" / "保留证据与版本",
  with no rule, no owner, no parameter).
- **5 carry an intent with no executable mechanism** (`I`): the three 进入条件 and the two 晋升条件.
- **1 is not applicable** (L3 晋升 — terminal layer).

The other eight columns are populated only with *definitions*, *intended readers* and *prohibitions*;
column 13 (当前定义是否完整) is **NO for all three tiers**. In one sentence: the only cells in this
matrix that contain something a coder could execute are the prohibitions.

### 4.1 The alternative three-layer vocabulary that was never merged into the above

`00 §7.5` (L364–379) defines a **different** three-layer model with an explicit 晋升条件 column:

| 层 | 承载内容 | 生命周期特征 | 晋升条件 (`00` L368–372) |
|---|---|---|---|
| Session Context | 当前主题、当前任务、未完成问题、临时信息、当前互动状态 | 短、不保证跨会话 | "**默认不自动晋升**；Session 结束即丢弃" |
| Working Memory | 跨会话话题、未完成目标、近期事件、阶段计划、待确认事实、临时关系状态 | 跨会话、频繁更新、需过期检测、可由 Dream 维护 | "需 Evidence + Validation + Review 后才可晋升" |
| Long-term Memory | 稳定事实、长期偏好、重要经历、稳定关系、长期目标、程序性知识 | 长期保留、需来源与证据 | "已是最终层，受更严格权限与证据要求" |

This is **the only place in the corpus that contains the word 晋升 with a condition attached**
(E2: 8 occurrences in `00`, the rest are unrelated "不自动晋升" prohibitions). It is also the only
place where the middle layer gets a maintainer ("可由 Dream 维护") — and Dream is FUTURE.
`00 §7.5` rule 4 (L379) still speaks of `lifecycle_state`, i.e. the *abolished* v1 field, so this
section was never migrated to the V2 model (`DOCUMENT-CONFLICT-REGISTER-V2 SC-01` L13 records the
switch but no document reconciled §7.5).

**No document maps {Session, Working, Long-term} to {L1, L2, L3}.** An implementer must choose, and
whichever they choose, they silently discard one of the two authorities.

---

## 5. Short-term → long-term data flow (as designed, and where it stops)

```
  chat turn (host session history)
        │  C(DSH 历史) "不是第三个权威记忆层" ................ 00 L347
        ▼
  Host Adapter ──identity 5-tuple──► Memory API ──► Ingestion Pipeline stage 1..16
        │                                              (V2 §5; 09 §7.x)
        │  stage 3 MemoryValueEvaluator: SAVE_STABLE_PERSON "倾向 L3" ... 09 L322
        │  stage 6 MemoryClassifier  ──► lifecycle_candidate = L1|L2|L3  09 L422  ← only tier signal
        │                                            │
        │                     candidate.schema.json: additionalProperties=false,
        │                     no lifecycle_candidate  ──► ✖ FIELD DROPPED (E2, A-C008)
        ▼                                            ▼
   candidate.status 6 态  ──► stage 16.2 M04 裁决 ──► 7 decisions:
   (PENDING/AUTO_ACCEPT/          reject | hold | merge | supersede |
    NEEDS_REVIEW/REJECTED/        create_new | observe | dispute       09 L710
    DEFERRED/COMMITTED)                              │
   09 L744–751                                      │  no "set tier" decision exists
                                                     ▼
                              M01 事务五步 (Validate → Check version → Write memory
                              → Write memory version → Write outbox → Commit)  09 L711
                                                     │
                                                     ▼
                       memory_objects { …, status(4), validation_state(4) … }
                       ✖ no memory_tier column           01 §7.16 L666 (E2)
                                                     │
                                                     ├──► Outbox → Worker → FTS5 / LanceDB
                                                     │      search-document: no memory_tier (E2)
                                                     ▼
                       ┌─────────────── tier axis: ✖ no edge, ✖ no job, ✖ no demotion ───────────┐
                       │  L1 ──?──► L2 ──?──► L3          (11 §7.12 rule 1 says M04 decides)      │
                       │  intended by 00 §7.5 L377: Evidence → Validation → Review → 晋升          │
                       └──────────────────────────────────────────────────────────────────────────┘
```

**Where it stops, in one line:** between stage 6 (`lifecycle_candidate`) and `memory_objects`
(`memory_tier`) there is a contract that cannot carry the value, a decider (M04) that has no
decision for it, a table that has no column for it, and a state machine that has no edge for it.
The short-term signal dies four times before it could become long-term.

---

## 6. Tier vs status vs validation_state (and the two other axes that get confused with them)

| Axis | Field | Values | Who writes | What it answers | Where defined | Lifetime semantics | Coupled to tier? |
|---|---|---|---|---|---|---|---|
| **Tier** | `memory_tier` | `L1` `L2` `L3` | **nobody executable** (intended M04) | "how durable / how long is this meant to live" | `V2 §2.5` L89–94; `memory-unit.schema.json` | none defined | — |
| **Object status** | `status` | `ACTIVE` `SUPERSEDED` `ARCHIVED` `DELETED` | M04 decides, M01 persists (SM-1) | "is this the current version, or history, or gone" | `V2 §2.6` L97–103; SM-1 L8–33 | full state machine, 10 edges | statically orthogonal (E2: no `allOf`); operationally coupled via `AUTO_ACCEPT` gate (`09` L755) |
| **Validation** | `validation_state` | `unverified` `confirmed` `disputed` `rejected` | **only M04** for `confirmed` (D-AP-05) | "has this been verified, and by whom" | `V2 §2.8` L116–119; SM-2 L37–51 | 7 edges, incl. the only "resurrection" path `rejected→unverified` | not coupled |
| **Candidate status** | `candidate.status` | `PENDING` `AUTO_ACCEPT` `NEEDS_REVIEW` `REJECTED` `DEFERRED` `COMMITTED` | M03 produces, M04/Pipeline advances | "where is this candidate in the ingest funnel" | `V2 §2.7` L105–114; `09 §7.21` L744–764 | 6 states, `PENDING→COMMITTED` must pass through one of three | **coupled**: `AUTO_ACCEPT` only for `L1/L2` (`09` L755) |
| *(intruder 1)* Availability | `availability_state` | `available` `restricted` `suspended` | M05 decides, M01 persists | "may this be used right now" | v1 `01 §7.7.3` L385–395; SM-3 L55–67 | 5 edges | not coupled — **but it is the field that actually carries short-term behaviour** (`expires_at` expiry lands here, T-A-01 L395) |
| *(intruder 2)* Review flag | `needs_review` (legacy A-layer) / `lifecycle_state=pending_review` | boolean / state | legacy migration | "a human should look at this" | `01 §7.17.1` L708 | **TBD (OI-01-03)** | overlaps `candidate.status=NEEDS_REVIEW` |

**Three findings fall straight out of this table:**

1. **The four axes never collide in the schema but collide in practice.** Tier gates a candidate
   decision (`AUTO_ACCEPT`), fixes a profile row (`(L3, ACTIVE)`), and is claimed to drive expiry
   (`11` L598) while expiry is implemented on `expires_at` → `availability_state` (T-A-01).
2. **"Needs review" is not a tier property and is not a status property either** — it is three
   different things (`lifecycle_state=pending_review` in v1, `candidate.status=NEEDS_REVIEW` in V2,
   legacy `needs_review` boolean whose semantics are still **TBD**). Any implementation asking
   "which memories need review?" gets three contradictory answers and no index that serves the
   question (no index on `pending_review`; the only status index is
   `(tenant_id, scope, lifecycle_state)`).
3. **The short-term behaviour that actually exists in the design is on the *availability* axis, not
   the tier axis**: `T-A-01` (`01` L395) moves a record `available→restricted` when `expires_at`
   fires. That is the only "expiry" mechanism with an edge, an executor and a trigger — and it
   says nothing about tier.

---

## 7. Storage-structure map

| Structure | File / anchor | Carries tier? | Carries original text? | Carries evidence link? | E2 |
|---|---|---|---|---|---|
| `memory_objects` (authority table) | `01 §7.16` #2, L666 | **No** | n/a (`content` is normalised, `V2 §3.1` L184) | via link table | 0 hits for `memory_tier` in the DDL block; indexes `(tenant_id,scope,lifecycle_state)`, `(subject_id,type)`, `observed_at`, `updated_at` |
| `memory_versions` | `01 §7.16` #3, L667 | no (snapshot JSON only, `changed_fields`) | no | no | append-only, trigger-protected |
| `evidence` | `01 §7.16` #4, L668 | n/a | **No** — columns are `evidence_id, source_type, source_id, content_hash, observed_at, recorded_at, actor_id, tenant_id, scope, provenance_id, revoked` | — | **the decisive fact for A-C004** |
| `memory_evidence_links` | `01 §7.16` #5, L669 | n/a | n/a | **Yes**: `(memory_id, evidence_id, support_kind)` PK, `INDEX(evidence_id)` for reverse trace | supports `related_evidence` (`13 §7.4` L286–345) |
| `observations`, `memory_relations`, `subjects`, `outbox_events`, `event_ledger`, `index_registry`, `embedding_spaces`, `schema_migrations`, `audit_metadata` | `01 §7.16` #1, #6–#13 | no | no | no | 13 tables total, `memory_tier`: 0 |
| `event`, `emotion_context`, `person_profile`, `candidate`, `profile_version` | promised by `01` header L3 ⑥ ("SQLite **需新增** …表") | **no DDL exists anywhere** (E2: 0 `CREATE TABLE` in the whole corpus) | no | no | a to-do note, not a design |
| `session_state` (Session Memory) | `01 §7.16` L679 | the L1 store that would exist | — | — | **FUTURE**, "**MUST NOT** 在 P0–P3 提前创建" |
| `candidate_pool` | `01 §7.16` L679 | would carry the tier *proposal* | — | — | **FUTURE**; and no module is permitted to write it (`FC-019`; writer assignment `08`/`09` disputed per `03-ARCHITECTURE-FACTS §O` O5–O8) |
| `search-document` (FTS5 + LanceDB projection) | `search-document.schema.json` | **No** (`lifecycle_state` only) | no (`content` = projection) | `source_refs` | E2 |
| `read-response` | `read-response.schema.json` | **Yes** — `lifecycle` object `required:["tier","status"]`, 9 tier mentions | **Yes** — `evidence[].original_text` **required** | `evidence_refs` | E2 |
| `read-request` | `read-request.schema.json` | **No** — 17 properties, no tier, no `filters` object | — | — | **asymmetry: you can be given a tier but cannot ask for one** |
| `candidate` | `candidate.schema.json` | **No** — `additionalProperties:false`, no `lifecycle_candidate` | `proposed_content` | `evidence_refs` | E2 |
| `evidence` (machine contract) | `evidence.schema.json` | n/a | **No** — `original_text` absent, `excerpt` (≤2000, nullable) present, `additionalProperties:false` | — | E2 |
| proposed-write contract | *missing* | `V2 §11` L437 lists `propose-request` as a new contract; **no such file exists** in the 31 (E2) | — | — | `14` L601 makes `memory_tier` an optional propose field with no contract |

**Queryability verdict per tier:**

| Tier | JSON schema | DB column | DB index | Index document | Request filter | Write path | Verdict |
|---|---|---|---|---|---|---|---|
| L1 | ✔ (`memory-unit`) | ✖ | ✖ | ✖ | ✖ | ✖ | **label only** |
| L2 | ✔ | ✖ | ✖ | ✖ | ✖ | ✖ | **label only** |
| L3 | ✔ | ✖ | ✖ | ✖ | ✖ | ✖ | **label only** |

So: **not every tier has a queryable data structure — no tier has one.** The single place the value
can legally exist is a JSON Schema (`memory-unit.schema.json`), which is a *contract*, not storage.
`STANDARDS §2`: "A schema file is not proof of a working subsystem." Here it is not even proof of a
writable field.

---

## 8. ASCII lifecycle diagram

### 8.1 What the machine actually implements (E2-verified)

```
                    ┌──────────────── SM-1 status (10 edges) ────────────────┐
   (new) ──T-L-01──► candidate ──► pending_review ──► ACTIVE ──► SUPERSEDED │
                        │              │               │  │        │        │
                        └──T-L-08──────┴──T-L-03───────┘  │        │        │
                                        (M04 + canWrite)  │        │        │
                                     T-L-05 archive ◄─────┘        │        │
                                     T-L-07 delete  ◄──────────────┴────────┘
                    └────────────────────────────────────────────────────────┘
   SM-2 validation (7 edges)   SM-3 availability (5 edges)   SM-4 outbox (8)
   SM-5 worker (5)             SM-6 migration (7)           SM-7 deletion (6)
   SM-8 embedding migration (9)                            ── 57 transitions total
                                                            ── 0 on the tier axis   (E2)
```

### 8.2 What the prose requires (and nothing implements)

```
      L1  ─────────── ? ───────────►  L2  ─────────── ? ───────────►  L3
   (临时记忆)                      (活跃记忆)                       (长期记忆)
      │                               │                               │
      │  entry: ?                     │  maintenance: ?               │  retention: ?
      │  expiry: "可快速过期"          │  "需定期维护与过期处理"          │  "保留证据与版本"
      │  exit: "Session 结束即丢弃"     │  demotion: ?                  │  demotion: ?
      ▼                               ▼                               ▼
   ✖ no store                     ✖ no store                    ✖ no store
   ✖ no writer                    ✖ no writer                   ✖ no writer (M04 = P4, unauthorised)
   ✖ no edge                      ✖ no edge                     ✖ no edge

   Forbidden by explicit MUST NOT:
     L1 ──✖──► L3   (11 §7.12 r5: "L1 MUST NOT 自动晋升 L3")
     any ──✖──► tier-driven read gate / read priority / sort weight (11 L597, L632)
     any ──✖──► promote because "important" (11 L513)
```

### 8.3 The five intended tier outcomes (3 entry + 2 promotion), and who would own each

| # | Edge | Intended trigger (verbatim) | Intended decider | Executable? |
|---|---|---|---|---|
| 1 | (new) → L1 | none stated; `lifecycle_candidate` default never specified | M03 (proposal) / M04 (decision) | ✖ |
| 2 | (new) → L2 | "矩阵只是默认倾向" | M04 | ✖ |
| 3 | (new) → L3 | `SAVE_STABLE_PERSON … 倾向 L3` (`09` L322) | M04 | ✖ |
| 4 | L1 → L2 | "需 Evidence + Validation + Review" (`00` L377) | M04 | ✖ |
| 5 | L2 → L3 | "MUST 有稳定/明确/多次/用户明确要求等依据" (`11` L513) | M04 | ✖ |
| — | L3 → L2, L2 → L1 | **none — zero mentions in the corpus** | — | ✖ |

---

## 9. The 0-edge measurement: verified — and the missing part is *true absence*, not a transcription loss

### 9.1 Measurement (E2, reproduced independently)

| Measured | Value | Method |
|---|---|---|
| Machine state-machine contract | `contracts/state-machines/state-machines.md`, 9 945 B — the **only** state-machine contract (the directory contains no `.json`) | `a-machine-checks.cjs` |
| State machines | 8 (`SM-1`…`SM-8`) | headings L8, 37, 55, 71, 90, 106, 124, 141 |
| Transition rows | **57** (10+7+5+8+5+7+6+9) | row-by-row manual count from full text |
| Rows on the tier axis | **0** | — |
| Occurrences of `tier` / `memory_tier` / `L1` / `L2` / `L3` / `层级` in the whole file | **0** | regex over the full file (E2) |
| `11 §7.12` block | heading at **L493**, matrix 6 rows (L501–506), **5** MUST rules (L510–514), 0 rows of form `Lx → Ly` | E2 (`d11_7_12`) |
| 31 JSON contracts carrying `memory_tier` | **1** (`memory-unit.schema.json`) | E2 |
| 31 JSON contracts carrying `lifecycle_state` | **2** (`memory-object`, `search-document`) | E2 |
| Contracts carrying **both** | **0** | E2 |

The prior forensic figure ("57 transitions, 0 on the tier axis", `09-CONFLICT-REGISTER FC-013`) is
**confirmed exactly**. The brief's "0 tier edges" is correct.

### 9.2 Is the gap transcription or true absence? — **BOTH, and the transcription half is empty of executable content**

Forensic `FC-013` classifies this as type **B** ("rule exists, machine lacks it") and explicitly
corrects an earlier "total absence" claim: *"this is not 'no tier rules exist'. They exist in
`11 §7.12` and were never transcribed."* Having read §7.12 in full, I **partially disagree**, and
the distinction matters for what the Lead should ask the user to decide:

| §7.12 content | What it actually is | Can it be transcribed into an edge? |
|---|---|---|
| L497–506: 3×6 matrix with `●`/`○`/`—` | Explicitly "**典型基准，非硬绑定**" (typical baseline, **not** hard-bound) — a *prior over categories*, marked with frequency glyphs | **No.** It has no from-state, no to-state, no trigger, no condition. Transcribed verbatim it yields a `semantic_type → preferred_tier` lookup, i.e. exactly the conflation §7.12's own iron law forbids. |
| L510 rule 1 | Says *who decides* (M04) and *what M03 may emit* (`lifecycle_candidate`) — a responsibility split | Partially: it gives a decider, not a decision rule |
| L511 rule 2 | Prohibition: no tier-driven read priority | No (constraint, belongs in a MUST-NOT list) |
| L512 rule 3 | Prohibition: L1 ≠ discardable | No (constraint) |
| L513 rule 4 | Prohibition: no promotion on "importance"; "MUST 有稳定/明确/多次/用户明确要求等依据" | **Half of an edge**: it names four *kinds* of sufficiency with no counts, no thresholds, no combination rule, no verification procedure. "多次" without N is not decidable. |
| L514 rule 5 | Prohibition: L1 MUST NOT auto-promote to L3 | No (constraint) |

So the block decomposes into **1 non-binding prior (matrix) + 1 responsibility statement (r1) +
3 decidable prohibitions (r2, r3, r5) + 1 undecidable sufficiency sketch (r4)**. There is **no edge set
in §7.12 to lose in transcription**: a faithful transcription of the matrix into `SM-9` would produce
a classifier default; a faithful transcription of r1–r5 would produce one ownership row, three
`MUST NOT`s and one "MUST have basis" rule whose basis is unenumerated. None of that is an executable
transition, so an `SM-9` written *from §7.12 alone* would still contain **zero** edges.

**Verdict: TRUE ABSENCE of a tier transition system, plus a transcription gap of a *non-executable
prior*.** Concretely: what is missing was never written anywhere, not even in prose — executor
condition table, decidable L2→L3 threshold, L1 entry rule, all demotion paths, retention actions,
re-evaluation schedule, idempotency, audit operation, error code. Enumerated as 16 rules in §10.
The forensic framing is right that §7.12 exists and is not transcribed; it is too generous in
calling that "the rule exists", because a `●/○/—` prior cannot be executed as a lifecycle.

**Consequence for the Lead:** remedying this is *not* a documentation-transcription ticket
(effort: transcribe). It is a *design* ticket (effort: decide 16 rules, several of which are
product decisions: does L1 ever become L3? is there any demotion? does an expired L1 disappear?).

---

## 10. Explicit list of MISSING conversion rules (16)

Legend: **MISSING** = no valid definition anywhere · **PARTIAL** = a fragment exists, not executable.
Each row is a rule a coder would need before writing `TierTransition`.

| # | Rule needed | What exists (verbatim anchor) | Status |
|---|---|---|---|
| **MCR-01** | Entry rule: what tier a newly ingested MemoryUnit gets, and who sets it | "矩阵只是**默认倾向**；最终 `tier` **MUST** 由 M04 整合裁决" (`11 §7.12` L510); `memory_tier` is **required** at insert (`V2 §3.1` L190) | **MISSING** (a required field whose value has no rule) |
| **MCR-02** | `L1 → L2` promotion condition | "需 Evidence + Validation + Review 后才可晋升" (`00 §7.5` L371) | **MISSING** (gate ordering only) |
| **MCR-03** | `L2 → L3` promotion condition + decidable threshold | "**MUST** 有稳定/明确/多次/用户明确要求等依据" (`11` L513) | **MISSING** ("多次" undefined; no N, no window, no combination) |
| **MCR-04** | `L3 → L2` demotion condition | "不因新信息**无条件**覆盖" (`V2 §2.5` L93) implies conditional demotion; nothing defines the condition | **MISSING** |
| **MCR-05** | `L2 → L1` demotion condition | — (0 corpus mentions) | **MISSING** |
| **MCR-06** | Per-edge executor + authorization (who may write which edge) | "最终 tier **MUST** 由 M04 整合裁决" (`11` L510); `ALG-10 §4.1` authorization matrix has 4 groups, **no tier group** (`02` L861–866) | **PARTIAL** |
| **MCR-07** | Re-evaluation trigger/schedule (event-driven vs periodic; cadence; owner) | `L2` "**需定期维护**与过期处理" (`V2` L92); Dream tasks are 8 kinds, none is tier review (`02 §7.12` L932) | **MISSING** |
| **MCR-08** | Expiry action per tier (on end-of-life: demote to which tier? archive? delete?) | "`L1` 只是'可快速过期'" (`11` L512); `02` L880 forbids single-TTL deletion and lists 6 factors — none tier-based | **MISSING** |
| **MCR-09** | Session-end disposition of L1 (which operation, which status, tombstone or not) | "Session 结束即丢弃" (`00` L370); "`L1` 仅预留" (`00 §5` X-2, L214) | **MISSING** |
| **MCR-10** | Conflict rule when the assigned tier contradicts the 3×6 prior | prior is "非硬绑定" (`11` L497) — so contradiction is legal but unarbitrated | **MISSING** |
| **MCR-11** | Idempotency + version semantics of a tier write (version bump? `expected_version`? replay?) | `ALG-10 §7` bumps version for *state* changes (`02` L886); `01 §7.15` defers version policy to **OI-01-05** | **MISSING** |
| **MCR-12** | Permission hook: does a tier change require M05 `canUpdate`? | `ALG-10 §4.1` requires M05 only for availability/deletion (`02` L865–866) | **MISSING** |
| **MCR-13** | Audit/provenance: a `tier_change` operation + ledger row | `operation` is a closed set of **9** (`01` L471–484) with no tier value; `event_ledger.operation` = 九种 (`01` L673) | **MISSING** |
| **MCR-14** | Failure handling + error code for an illegal tier transition | generic: "非法转换 … MUST 返回 VALIDATION_ERROR" (`state-machines.md` L165); `error-codes.md` has no tier/transition code | **PARTIAL** |
| **MCR-15** | Tier of merged / superseded / derived records (merge result tier; inheritance on `supersede`; `SYSTEM_GENERATED` consolidation) | `merged_from` exists (`V2 §3.1` L202); M04 merge has six preconditions (`02 §7.9 ALG-08` §5.3, "① 主体一致；② Scope 兼容；③ 语义关系明确；④ 来源链可保留；⑤ 时间关系明确；⑥ 合并后不丢失重要信息") — **none of the six mentions tier** | **MISSING** |
| **MCR-16** | Back-fill of tier for existing rows (A/B migration + P0 seed) | `01 §7.17.1` field map has **16 rows and no `memory_tier` row**; several fields are forced to `null` "**MUST NOT** 编造" | **MISSING** (migrated rows cannot satisfy a required field without inventing a value — which the migration table forbids) |

**Count: 16 missing conversion rules — 14 `MISSING`, 2 `PARTIAL`.**

---

## 11. Findings register (A-Cnnn)

Every finding follows the nine required questions in compressed form:
*(1) what exists · (2) what is missing · (3) capability damage · (4) module-responsibility conflict ·
(5) missing algorithm · (6) recommendation · (7) documents to amend · (8) blocks V0.9? ·
(9) settleable inside V0.9 experiments?*

---

### A-C001 · The tier axis has no transition system at all — short-term cannot reach long-term · **P0**

- **Label**: EXISTING DESIGN (`V2 §2.5`; `11 §7.12`) / **DESIGN COMPLETENESS: `MISSING`** / RECOMMENDED ADDITION below.
- **(1)** Three tier values with prose definitions; a non-binding 3×6 prior; five MUST rules; 57 state-machine transitions, **0** on the tier axis (E2).
- **(2)** Entry rule, two promotion edges, two demotion edges, executor table, re-evaluation schedule, retention action, audit operation, error code — 16 rules (§10).
- **(3)** This *is* the "short-term → long-term" capability in `STANDARDS §4`'s P0 definition, verbatim. Without it a chat AI can never convert a conversation into a durable fact by any specified route.
- **(4)** **Yes, and it is the worst kind**: `11 §7.12` assigns the decision to M04, but `09 §7.20` gives M04 seven decisions and none is "set tier"; `02 §7.11 ALG-10` gives the lifecycle algorithm four authorization groups and none is tier. Three documents each believe another one owns it.
- **(5)** Yes — `TierTransition` is absent from the ALG-01…ALG-12 catalogue (`02` §7.1–§7.12).
- **(6)** **NEW PROPOSAL** — add `ALG-13 TierTransition` with the 16 rules of §10; add `SM-9` to the state-machine contract with **1 initial-state rule** (which tier is written at insert, per the bootstrap rule of A-C002) **+ 4 edges** (`L1→L2`, `L2→L3`, `L3→L2`, `L2→L1`), each with trigger/executor/audit/test; extend M04's decision enum with `set_tier`; add `tier_change` to `operation` 九种 (making it ten).
- **(7)** `DD-AUTHORITY-V2 §2.5` (add decision rule + executor), `11 §7.12` (convert prior into a decidable rule or demote it to a hint with no authority), `02` new ALG-13, `contracts/state-machines/state-machines.md` SM-9, `09 §7.20` decision enum, `01 §7.9` operation set, `06` new tests.
- **(8)** **Yes** — directly blocks the core "long-term memory" promise.
- **(9)** Partly. The *edge set* must be decided (product decision, e.g. "does L1 ever become L3 directly?"). But thresholds can be calibrated in V0.9 (e.g. N appearances in window W with evidence). Recommend: decide edges now (cheap, 5 yes/no questions), calibrate thresholds in V0.9.

---

### A-C002 · `memory_tier` has no storage footprint and no legal writer · **P0**

- **Label**: EXISTING DESIGN (`V2 §3.1` L190 required field; `memory-unit.schema.json` enum) / **DESIGN COMPLETENESS: `MISSING`**.
- **(1)** `memory_tier` is a **required** MemoryUnit field (E2: in `required[]` of the only contract that has it).
- **(2)** (a) **no column** in the 13-table DDL (E2: 0/13, `01 §7.16`); (b) **no index**; (c) **no** `memory_tier` in `search-document` so the derived indexes cannot filter on it; (d) **no** writer in the authorised P0–P3 scope — M03 may only emit `lifecycle_candidate` (`09` L422), the final decision belongs to M04, and M04 is P4 "需阶段授权" (`V2 §10` L411); (e) **no** migration source — `01 §7.17.1` has no tier row and forbids inventing values.
- **(3)** In the only scope anyone is authorised to build (P0–P3 = M01+M02), the system must insert a row containing a required field that no module may compute. Implementers will either hard-code a constant (destroying the axis: everything becomes one tier) or `null` (violating the contract). Either way, "long-term memory" is a constant, not a classification.
- **(4)** Yes: `V2 §3.1` (field owner = MemoryUnit/M01), `11 §7.12` (decider = M04), `09` (proposer = M03) — three owners, and the only currently authorised one (M01) is defined as a *persistence* module that "MUST NOT 自行决定业务状态" (`02` ALG-10 §17, L906).
- **(5)** Not an algorithm gap — a responsibility/phase-provenance gap: no document says which module may write the field in which implementation phase.
- **(6)** **NEW PROPOSAL** — (i) add `memory_tier` to the `memory_objects` DDL with `CHECK (memory_tier IN ('L1','L2','L3'))` and index `(tenant_id, memory_tier, status)`; (ii) add it to `search-document`; (iii) declare a **bootstrap rule** for the pre-M04 phases (e.g. "M01 writes `lifecycle_candidate` verbatim as `memory_tier` with `tier_source=PROVISIONAL` until M04 confirms"), so the field has a legal writer at every phase; (iv) add the migration row with an explicit, allowed default.
- **(7)** `01 §7.16` (+column, +index), `contracts/schemas/memory-object.schema.json`, `search-document.schema.json`, `V2 §3.1` (`tier_source`), `01 §7.17.1` (migration row).
- **(8)** **Yes** — a required field with no legal writer is in `STANDARDS §4`'s P0 list ("a key datum has no legal writer").
- **(9)** No — this is a schema/migration decision, not an experiment.

---

### A-C003 · L1 (short-term) has no store and no owner; short-term memory is architected *outside* Memory Core · **P0**

- **Label**: EXISTING DESIGN (`00 §7.5`, `00 §7.7` Session Memory, `01 §7.16` FUTURE tables) / **DESIGN COMPLETENESS: `CONCEPT_ONLY`**.
- **(1)** A `L1` enum value; a `session_state` table name; a `Session Memory` module row; a `scope=session_only` value that means "仅当前会话 … MUST NOT 跨会话召回 … MUST NOT 自动晋升长期记忆" (`01` L234).
- **(2)** Any store that exists in P0–P3. `session_state` is FUTURE and "**MUST NOT** 在 P0–P3 提前创建" (`01` L679); `Session Memory` is a FUTURE module (`00 §7.7` L407); `Context Compression` is declared "独立于 Memory Core" (`00` L409).
- **(3)** Memory Core cannot hold the conversation it is supposed to learn from. It can only receive whatever the host chooses to push. The "short-term" tier therefore has no data to be short-term *about* — the design has delegated the entire short-term layer to a host that has no contract for it (`08` defines Context Adapter/Session isolation but no session-state store).
- **(4)** Yes: `00 §7.7` puts session state in a FUTURE module, while `V2 §2.5` puts `L1` inside MemoryUnit. Both cannot be true.
- **(5)** Yes — no "session buffer → candidate" algorithm exists.
- **(6)** **NEW PROPOSAL** — choose explicitly (this is the one place I recommend a *product* decision): either **(A)** L1 is Memory Core's own table (activate `session_state` in P1, define TTL, define session-end disposition), or **(B)** delete `L1` from `memory_tier` and let short-term live entirely in the host/Session Memory module, with `memory_tier` becoming a 2-value axis (`L2`/`L3`). Option B is cheaper and matches `00 §7.5`'s severance; option A is what "chat AI that remembers" usually means.
- **(7)** `V2 §2.5`, `00 §7.5`/§7.7, `01 §7.16`, `09 §7.5`.
- **(8)** **Yes** — "short-term cannot reach long-term" is a P0 phrase in `STANDARDS §4`.
- **(9)** No — an architecture decision, not an experiment.

---

### A-C004 · A long-term memory cannot be traced to the original input at the storage layer · **P0**

- **Label**: EXISTING DESIGN (`V2 §3.2` L217 `original_text` 必填; `13 §7.3` L242–258 save-discipline; `13 §7.4` reverse-trace) / **DESIGN COMPLETENESS: `CONFLICTED`**.
- **(1)** A full Evidence design: `original_text` MUST be stored verbatim and MUST NOT be normalised (`13` L252), `content_hash = sha256(original_text)` (L255), `memory_evidence_links` M:N with `INDEX(evidence_id)` (`13` L225), a 6-step reverse query (`13 §7.4.1`), `revoked` propagation (`13 §7.4.5`).
- **(2)** A **place to put `original_text`**. E2: the only SQLite DDL for `evidence` (`01 §7.16` #4) has 11 columns, none of them `original_text`; the only machine contract `evidence.schema.json` has `additionalProperties:false` and **no** `original_text` — it has `excerpt` (≤2000 chars, nullable), and its `source_type` is a **12-value v1 enum** that contradicts the 5-value `V2 §3.2` enum. Meanwhile `read-response.schema.json` *requires* `evidence[].original_text`.
- **(3)** The chain "long-term memory → its evidence → the sentence the user actually said" terminates at `source_id`/`source_locator` — a pointer into host session history — and `00` L347 declares that history "**不是第三个权威记忆层**". So after any host-side history rotation/compaction (which `08` itself plans: "压缩协作 + Compaction 九字段", `08` 变更记录 L832), the original sentence is unrecoverable while the memory remains. The product consequence is severe: the system cannot show "you said this on 3 Nov" and cannot audit its own inferences.
- **(4)** Yes: `13` (Evidence owner) vs `01 §7.16` (storage owner) vs `contracts/schemas` (contract owner) — three artifacts, three incompatible shapes.
- **(5)** Not an algorithm gap; a contract/schema gap. (The reverse-query flow *is* an algorithm and is `MOSTLY_COMPLETE`, `13 §7.4.1`.)
- **(6)** **NEW PROPOSAL** — (i) add `original_text TEXT NOT NULL` (+ `context_before/after`, `speaker_id`, `timestamp`) to `01 §7.16` #4 with `INDEX(content_hash)` retained; (ii) replace `evidence.schema.json` with a V2-shaped contract (`original_text` required, 5-value `source_type`), keeping `excerpt` only as an optional projection; (iii) state the retention rule for `original_text` separately from export rights (K-2 / OI-030 remains BLOCKED — do not conflate); (iv) record in `read-response` that `original_text` comes from the authority, not from the host.
- **(7)** `01 §7.16`, `contracts/schemas/evidence.schema.json`, `V2 §3.2`, `13 §7.3`, `03-ARCHITECTURE-FACTS §E` row "Evidence content_hash".
- **(8)** **Yes** — "can a long-term memory still be traced to the original input" is the auditability half of the long-term capability; and `read-response` already *requires* a field storage cannot produce, so any conforming reader fails.
- **(9)** Partly — whether to store or reference the original text is a decision (storage cost / privacy); the *contract alignment* is not.

---

### A-C005 · Three unrelated three-level hierarchies share the tokens `L1/L2/L3`, with no mapping · **P1**

- **Label**: EXISTING DESIGN (`V2 §2.5`, `V2 §6`, `00 §7.5`) / **DESIGN COMPLETENESS: `CONFLICTED`**.
- **(1)** Hierarchy A = memory tier (`L1/L2/L3`, `V2 §2.5`); Hierarchy B = LLM call layers (`L1 轻量理解 / L2 强语义 / L3 聊天模型`, `V2 §6` L339–341, `10` throughout); Hierarchy C = time scales (`Session Context / Working Memory / Long-term Memory`, `00 §7.5`).
- **(2)** A mapping table A↔C; disambiguating prefixes; a single vocabulary.
- **(3)** Every reader and every coder must disambiguate by context. `09` uses `L1/L2` for LLM layers at L241/L347/L924 and `L1/L2/L3` for tier at L422/L443/L885 — **same file, same token, two meanings**. E2 counts: `memory_tier` 11 occurrences corpus-wide vs the bare token 400+. A code reviewer cannot tell whether `tier: "L1"` means "session-scoped memory" or "produced by the lightweight LLM".
- **(4)** No (a naming problem), but it produces module confusion: `10`'s `ingest.model_tier=L1` (`09` L924) sits one table away from `lifecycle_candidate=L1`.
- **(5)** Not an algorithm.
- **(6)** **NEW PROPOSAL** — rename the tier values to `T1/T2/T3` (or `SESSION/SHORT/LONG`) in `V2 §2.5`, the schema enum, and all prose; keep `L1/L2/L3` exclusively for LLM layers. Cheap, mechanical, and removes the single most likely source of implementation error in this axis.
- **(7)** `V2 §2.5`, `memory-unit.schema.json`, `read-response.schema.json`, `11`, `12`, `09`, `14`.
- **(8)** No (does not block), but it will *cause* defects if left.
- **(9)** Yes — mechanical rename, no experiment needed.

---

### A-C006 · The request contract cannot ask for a tier, so "read long-term memory directly" is not expressible · **P1**

- **Label**: EXISTING DESIGN (`11 §7.16` read capability #4, L590; `11 §7.17` L617 "结构化通道 MUST 只用于过滤（`tier`、`status`…）"; `11 §8.3` `ReadRequest` L723 `tier_filter` array\<enum\>) / **DESIGN COMPLETENESS: `MISSING`** at the contract.
- **(1)** Three prose statements that a tier filter exists and is used for filtering (not scoring), **and** a documented `ReadRequest` shape that contains the field (`11 §8.3` L713–725).
- **(2)** E2: `read-request.schema.json` has **17** top-level properties — `request_id, operation, query, subject_id, anchor, intent, read_mode, top_k, budget_tokens, budget_chars, include_evidence, include_emotion, requester, timestamp, sections, trace_id, idempotency_key` — **no** `tier_filter`, **no** `filters` object (identity is nested under `requester`, `required:["actor_type","actor_id","tenant_id"]`). `read-response` **requires** `lifecycle.tier`. Asymmetry: tier is mandatory in the answer and impossible in the question, and the corpus holds **two incompatible `ReadRequest` definitions** (`11 §8.3` L713–725 vs the contract). Note also that `read-request` covers only 3 of the 9 APIs (`memory.search`/`getByAnchor`/`getProfile`); the other six are delegated to "各自专用请求" which **do not exist** in the 31-contract set — so `getMemoryById` (the "直读" route that would need a tier argument) has no machine request contract at all.
- **(3)** Read capability #4 ("直读长期记忆 … 授权后可直接按 tier 取，`11 §7.16` L590") and its test `TS-11-17`/`TS-11-18` (`11` L904–905) cannot be executed. An agent asking "what do you know about me long-term" gets whatever ranking produces, not the L3 set.
- **(4)** Yes: `11` (anchor/read design) vs `contracts/schemas` (contract).
- **(5)** No — a contract omission.
- **(6)** **NEW PROPOSAL** — add `tier_filter?: array<enum L1,L2,L3>` (a *filter only*, explicitly non-authorising, per `11` L723) to `read-request`; add it to `search-document` so it can push down; forbid its use as a ranking weight (already `11` L632).
- **(7)** `contracts/schemas/read-request.schema.json`, `search-document.schema.json`, `11 §7.16`/§7.17/§8.3.
- **(8)** No — retrieval works without it; one of seven read capabilities is lost.
- **(9)** Yes — contract edit + one test.

---

### A-C007 · No retention, expiry, or review policy exists *per tier*; the only expiry machinery is per-type and per-record · **P1**

- **Label**: EXISTING DESIGN (`V2 §2.5` "可快速过期"/"需定期维护与过期处理"; `11 §7.16` rule 2 "tier 只影响 … 过期与复核策略") / **DESIGN COMPLETENESS: `MISSING`**.
- **(1)** Tier-level claims that expiry/review depend on tier (`11` L598). Per-`type` decay λ (7 types, `02` L404–414). Per-record `expires_at` with an availability edge `T-A-01` (`01` L299/L395). `retention_policy` on Evidence only (`V2 §3.2` L222; controlled vocabulary, still to be registered, `13` L270).
- **(2)** Any tier-scoped retention rule; any sweeper; any parameter. E2: `15-参数注册表` contains **no** `lifecycle.*` parameter at all — its only `*tier*` ID is `ingest.model_tier`, which is the **LLM layer** (`15` L204), not the memory tier — though `ALG-10 §13` (`02` L898) declares `lifecycle.expiry_policy.{type}` and `lifecycle.archive_after_days.{type}` as required parameters that "MUST 进 15 号参数注册表" (`V2 §13.3`).
- **(3)** "What is retained vs expires vs needs review" — the exact question in this brief — has no tier-level answer. L1 is described as "可快速过期" but nothing expires it (so L1 rows live forever, i.e. short-term becomes *permanent* by default — the opposite of the intent); L2 "需定期维护" with no maintainer; L3 "保留证据与版本" with evidence that cannot hold text (A-C004).
- **(4)** Yes: retention is claimed by `11`, implemented (per-type) by `02`, and provisioned nowhere (`15` missing, `05` no job, Dream FUTURE).
- **(5)** Yes — a retention algorithm is absent from ALG-01…12.
- **(6)** **NEW PROPOSAL** — a single `RetentionPolicy` table keyed `(tier, semantic_type)` with explicit columns `ttl_days`, `review_after_days`, `on_expiry ∈ {demote, archive, delete}`, `requires_evidence`; register it in `15`; schedule it via Dream **or** a documented maintenance job with an owner; make `on_expiry` auditable.
- **(7)** `V2 §2.5`, `11 §7.16` rule 2, `02` ALG-10 §13/§14, `15`, `05`, `01 §7.16`.
- **(8)** No — but without it L1 silently becomes permanent, which *is* a capability distortion.
- **(9)** Yes — TTLs are exactly what V0.9 experiments calibrate. The *existence* of the policy table is not an experiment.

---

### A-C008 · The candidate contract cannot carry `lifecycle_candidate`, so the tier proposal dies at the boundary · **P1** (upgrade path to P0 once A-C001 is fixed)

- **Label**: EXISTING DESIGN (`09` stage 6 L422, L443, L885 — `lifecycle_candidate` required) / **DESIGN COMPLETENESS: `CONFLICTED`**.
- **(1)** `lifecycle_candidate ∈ {L1,L2,L3}`, required, "不是最终 tier" (`09` L422).
- **(2)** E2: `candidate.schema.json` is `additionalProperties:false` with 21 properties and **no** `lifecycle_candidate`; it also has no `candidate.status` (it still carries v1 `quality_status: draft|checked|needs_review|rejected` + `review_state: queued|in_review|accepted|rejected|merged`), so the V2 6-state funnel (`V2 §2.7`) is also absent.
- **(3)** M03's only tier signal is rejected by the contract that is supposed to transmit it. Even if SM-9 existed, the pipeline could not feed it.
- **(4)** Yes: `09` (pipeline/candidate owner) vs `contracts/schemas` (contract owner) vs `V2 §2.7` (enumeration owner).
- **(5)** No.
- **(6)** **NEW PROPOSAL** — regenerate `candidate.schema.json` against `V2 §2.7`/`§3.1`: add `lifecycle_candidate`, `candidate_status` (6 values), `semantic_type`, `anchor`, `subject_type`, `source_type`(5), `tier_basis`; keep `additionalProperties:false` so the omission cannot recur silently.
- **(7)** `contracts/schemas/candidate.schema.json`, `09 §8.4`, `V2 §11` contract list.
- **(8)** No (P3 scope), but it is a hard blocker for A-C001's fix.
- **(9)** No — contract work.

---

### A-C009 · The orthogonality is asserted but the only assignment guidance is a category function; two subsystems hard-wire tier to other axes · **P1**

- **Label**: EXISTING DESIGN (`V2 §2.5` 铁律; `11 §7.12` 铁律; `11 §7.1`/§7.12/§7.18 tables; `12 §7.7` L390; `09 §7.21` L755) / **DESIGN COMPLETENESS: `CONFLICTED`**.
- **(1)** Three explicit orthogonality MUSTs; a static schema with no coupling (E2).
- **(2)** Any tier assignment rule that is *not* a function of `semantic_type`; and a decision on whether tier may gate trust.
- **(3)** Three separate couplings exist in executable text: (i) `semantic_type → tier` prior (three copies: `11` L172, L499, L668) is the *only* assignment guidance; (ii) `PersonProfile → (L3, ACTIVE)` (`12` L390) makes L3 mean "consolidated row"; (iii) `AUTO_ACCEPT → L1/L2 only` (`09` L755) makes tier a trust/risk proxy. Result: `tier` is a relabelled category plus a trust hint — precisely what the 铁律 forbids. A user asking "what do you remember long-term" gets, in effect, "things that were classified as PERSON/PREFERENCE/RELATIONSHIP", not "things that survived time".
- **(4)** Yes: `12` (profile) and `09` (candidate gating) both act on tier, while `11` says tier must not decide anything but ordering/expiry.
- **(5)** Yes — the missing piece is a *tier assignment predicate over evidence/stability*, not over category (see A-C001).
- **(6)** **NEW PROPOSAL** — restate the 3×6 matrix as "**classification hint for M03 only; may not be used to decide tier**"; define tier by evidence/stability predicates (`evidence_count ≥ n`, `distinct_sessions ≥ m`, `user_explicit`, `confirmed by M04`, `not contradicted for T`); remove `(L3, ACTIVE)` from the profile mapping (a profile row is a *view*, not a tier holder); move `AUTO_ACCEPT`'s precondition from tier to `validation_state`/`source_type`.
- **(7)** `11 §7.12`, `11 §7.1`, `11 §7.18`, `12 §7.7` L390, `09 §7.21` L755, `V2 §2.5`.
- **(8)** No — but it decides *what the product means* by "long-term", so it should be settled before any tier code exists.
- **(9)** Partly — the predicates are calibratable in V0.9; the removal of the two hard-wirings is a decision.

---

### A-C010 · "Needs review" is three different things on three axes, and nothing indexes the question · **P1**

- **Label**: EXISTING DESIGN (`V2 §2.7` `NEEDS_REVIEW`; v1 `lifecycle_state=pending_review`; A-layer `needs_review`) / **DESIGN COMPLETENESS: `CONFLICTED`**.
- **(1)** Three review signals: `candidate.status=NEEDS_REVIEW`; legacy `lifecycle_state=pending_review`; A-layer boolean `needs_review` whose mapping is **TBD** (`01 §7.17.1` L708: "`false` ⇒ 见 OI-01-03").
- **(2)** One definition of "this memory needs a human", and an index that serves "list what needs review".
- **(3)** The retention question in this brief ("what is retained vs expires vs **needs review**") has no single answer. Also unindexed: `memory_objects` indexes are `(tenant_id,scope,lifecycle_state)`, `(subject_id,type)`, `observed_at`, `updated_at` — a `pending_review` scan is a filter over the first index at best, and `candidate_pool` (where review work actually queues) is FUTURE with no legal writer (`FC-019`).
- **(4)** Yes: `01` (v1 lifecycle), `V2 §2.7` (candidate funnel), `09 §7.21` (state semantics) and OI-01-03 (unresolved) all touch it.
- **(5)** No.
- **(6)** **NEW PROPOSAL** — one review queue: `candidate.status=NEEDS_REVIEW` for pre-commit work (needs `candidate_pool` to be created and assigned a writer), and `validation_state=disputed` for post-commit doubt; retire `lifecycle_state=pending_review` explicitly with a mapping; resolve OI-01-03; add index `(tenant_id, status, validation_state)`.
- **(7)** `01 §7.7`, `01 §7.17.1` L708, `V2 §2.7`, `09 §7.21`, OI-01-03.
- **(8)** No.
- **(9)** Yes — a mapping decision plus an index.

---

### A-C011 · `ALG-10` declares lifecycle parameters that do not exist in the parameter registry · **P2**

- **Label**: EXISTING DESIGN (`02` L898, `L900`) / **DESIGN COMPLETENESS: `PARTIAL`**.
- **(1)** `lifecycle.expiry_policy.{type}`, `lifecycle.archive_after_days.{type}`, `lifecycle.decay_lambda.{type}` declared with EXP-DEFAULT/TBD status (`02` L898).
- **(2)** Their registration. E2: `15-参数注册表` contains **no** `lifecycle.*` entry (0 hits for `lifecycle`, `expiry`, `archive_after`, `tier` as parameter IDs).
- **(3)** `V2 §13.3` forbids hard-coded parameters ("业务代码禁止硬编码"), so a coder implementing ALG-10 must either hard-code or block. Low severity for the tier question (these are per-type, not per-tier), but it is the same class of defect as A-C007.
- **(4)** No.
- **(5)** No.
- **(6)** **NEW PROPOSAL** — register the three families in `15` with `TBD`/`EXP-DEFAULT` status and locators, together with the `RetentionPolicy` parameters proposed in A-C007.
- **(7)** `15`, `02` L898–900.
- **(8)** No.
- **(9)** Yes — parameter registration is mechanical; values come from V0.9.

---

### A-C012 · No test anywhere exercises a tier transition; the only tier tests assert *prohibitions* · **P2**

- **Label**: EXISTING DESIGN (`11 §12` L904–905: `TS-11-17` 直读能力, `TS-11-18` 生命周期不限制直读; `V-11-06` tier×type 正交) / **DESIGN COMPLETENESS: `MISSING`** for transitions.
- **(1)** Three tier-related tests, all negative/no-gate assertions.
- **(2)** Any acceptance criterion for promotion/demotion/expiry. E2: `06-测试与评估规范.md` contains **no** `tier` token at all (its `L1/L2/L3` hits are `TEL-1..TEL-10` test levels and one `L3 回传/reinforce` row).
- **(3)** The 16 rules of §10 cannot be verified; even after they are written, "not yet tested" would apply, and `STANDARDS §5` requires a test criterion per algorithm.
- **(4)** No.
- **(5)** No.
- **(6)** **NEW PROPOSAL** — per edge: `TC-TIER-01` entry defaults, `TC-TIER-02` L1→L2 positive, `TC-TIER-03` L1→L3 direct refused, `TC-TIER-04` L2→L3 without evidence refused, `TC-TIER-05` demotion audited, `TC-TIER-06` expiry action, `TC-TIER-07` idempotent replay, `TC-TIER-08` tier never gates a read, `TC-TIER-09` tier never appears in a ranking weight.
- **(7)** `06`, `11 §12`, `contracts/state-machines/state-machines.md` SM-9 test list.
- **(8)** No.
- **(9)** Yes.

---

### 11.1 Priority summary

| Priority | Findings | Count |
|---|---|---|
| **P0** | A-C001 (no transition system), A-C002 (no storage/writer), A-C003 (no L1 store), A-C004 (original text untraceable) | 4 |
| **P1** | A-C005 (vocabulary collision), A-C006 (no tier request filter), A-C007 (no retention per tier), A-C008 (candidate contract), A-C009 (de-facto category conflation), A-C010 ("needs review" ambiguity) | 6 |
| **P2** | A-C011 (unregistered lifecycle params), A-C012 (no tier tests) | 2 |

---

## 12. What is missing to make this implementable (concrete checklist)

1. **Decide the axis** (product): 3-value tier with an L1 store, or 2-value tier (`L2`/`L3`) with short-term living in the host. (A-C003)
2. **Write `SM-9 TierTransition`**: 1 initial-state rule + 4 edges (`L1→L2`, `L2→L3`, `L3→L2`, `L2→L1`; `L1→L3` stays forbidden), each with trigger, decider, executor, transaction requirement, audit, test. (A-C001)
3. **Write the 16 rules of §10** into a new `ALG-13`; convert `11 §7.12`'s prior into a **hint with no authority**. (A-C001, A-C009)
4. **Give the field a home**: DDL column + CHECK + index; `search-document` field; `tier_filter` in `read-request`; bootstrap rule for the pre-M04 phases. (A-C002, A-C006)
5. **Fix the Evidence chain**: `original_text` in the DDL and in the machine contract, aligned to `V2 §3.2`. (A-C004)
6. **Regenerate `candidate.schema.json`** from V2 so `lifecycle_candidate` and the 6-state funnel survive the boundary. (A-C008)
7. **Register the parameters**: `retention.*`, `tier.*`, `lifecycle.*` in `15`. (A-C007, A-C011)
8. **Rename the tokens** to stop `L1/L2/L3` meaning three things. (A-C005)
9. **Add the 9 tests** of A-C012, and one end-to-end: "a fact told once in a session becomes an L3 memory with traceable original text, and disappears correctly when it expires."

---

## 13. Cross-scope handoffs

| To | Item | Why them |
|---|---|---|
| **Agent B** (long-term structure) | `PersonProfile` is the only long-term-shaped structure and is `(L3, ACTIVE)`-hard-wired (`12` L390); tier is its only durability marker | profile/relation structure ownership |
| **Agent C** (Evidence/Emotion) | `evidence.schema.json` vs `V2 §3.2` vs `01 §7.16` #4 — three incompatible Evidence shapes; `source_type` 12 vs 5 values | evidence chain |
| **Agent D** (retrieval/context) | `read-request` cannot express `tier_filter` (`11` L590/L617/L723 vs 17-property contract); ALG-04's six features key on the abolished v1 `type` | retrieval contract |
| **Agent E** (lifecycle/state machines) | `SM-1` uses the abolished `lifecycle_state`; tier has no SM; `operation` 九种 has no tier operation | state machines |
| **Lead** | Two product decisions: (1) does an L1 exist inside Memory Core? (2) is there any demotion? Both are cheap to answer and expensive to defer. | adjudication |

---

## 14. Method appendix

### 14.1 Files read in full
`DD-AUTHORITY-V2.md` · `contracts/state-machines/state-machines.md` · `contracts/error-codes.md` (table) ·
`contracts/config/*.json` headers via E2 · all 31 `contracts/**/*.json` via E2 property checks.

### 14.2 Files read in the cited ranges
`00` L120–413 · `01` L228–239, L276–320, L352–395, L462–485, L640–755 · `02` (sanitised) L396–443,
L685–759, L845–906, L920–950 · `09` L126–205, L700–769 · `11` L166–195, L430–609, L700–725 ·
`12` L182–199, L319–327, L384–403 · `13` L167–181, L240–270, L286–345 · `08` L832 (changelog row only).

### 14.3 Commands / scripts (E2)
- `EVIDENCE\a\a-machine-checks.cjs` → `a-machine-checks.json` — state-machine row/tier census; §7.12 block extraction; 31-contract `memory_tier`/`lifecycle_state`/`original_text` census; 13-table DDL scan; corpus-wide token census.
- `EVIDENCE\a\a-checks2.cjs` → `a-checks2.json` — `read-request` property list; `candidate`/`evidence` contract property checks; `09` homonym line extraction.
- `grep`-equivalent scans over the corpus for `memory_tier`, `晋升/降级/升层/降层`, `CREATE TABLE`, `lifecycle.*` parameters, tier tokens in `05`/`06`/`15`.
- **No corpus file was written or modified.** Node v24.21.0; scripts wrote only under
  `E:\[PROJECT]\CORE-CAPABILITY-REVIEW-2026\EVIDENCE\a\` using `fs.writeFileSync(..., 'utf8')` (no BOM).

### 14.4 Known measurement caveats
- `02-核心算法规范.md` contains a NUL byte and is rejected as binary by the harness reader; all `02`
  line numbers here come from `AUDIT-FORENSIC-2026\16-RAW\b\scratch\02-sanitized.md`. **Caveat:** in
  that copy `ALG-08` starts at L685 and `ALG-10` at L849, so several line numbers quoted for `02`
  in `AUDIT-FORENSIC-2026\03-ARCHITECTURE-FACTS.md` (e.g. "ALG-08 `02:697-709`", "`02:743`") do not
  land on the same text. I therefore cite `02` by **section** (`§7.9 ALG-08`, `§7.11 ALG-10`,
  `§7.12 ALG-11`) wherever the exact line is not one I read myself, and keep line numbers only for
  the ranges I read in that copy (L396–443, L685–759, L845–906, L920–950).
- The transition-row count is 57 by manual row reading; an early regex pass returned 56 because one
  `SM-6` row's `From` cell is `not_started`. The tier count (0) is robust to this.
- `L1/L2/L3` token counts are raw regex counts and **include** the LLM-layer and `Event-L3`
  meanings; they are used only to show scale, never as a tier measure. The tier measure is
  `memory_tier` (11 occurrences corpus-wide).
- `03-ARCHITECTURE-FACTS.md` and `09-CONFLICT-REGISTER.md` are used only as corroboration and are
  cited as such; every EXISTING DESIGN claim in §1–§11 is anchored to a baseline file I opened.
