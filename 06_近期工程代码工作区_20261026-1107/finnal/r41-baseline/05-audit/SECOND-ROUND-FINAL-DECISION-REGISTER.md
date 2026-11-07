# SECOND-ROUND-FINAL-DECISION-REGISTER — round 4.1

Date: 2026-11-05 (Asia/Shanghai)

This register records decisions **taken in this round** and the decisions that **cannot** be taken
here. A decision is only listed as `DECIDED` if an authoritative artefact now reflects it.

Legend: `DECIDED` · `DECIDED_WITH_SCOPE` · `CONFIRMED_EXISTING` · `ESCALATED` · `DEFERRED`

---

## 1. Inherited decisions (SRD-001 … SRD-008) — status after this round

| ID | Decision | This round |
|---|---|---|
| SRD-001 | Authorization: documentation + isolated verification only; no product implementation, no real data, no host integration | **CONFIRMED_EXISTING** — restated in `00-PARENT-MANDATE.md` and honoured throughout. `PRODUCT_IMPLEMENTATION_AUTHORIZATION` = NOT_AUTHORIZED at the end as at the start. |
| SRD-002 | Only `fts` and `vector` participate in RRF; `structured` is filter/router only | **CONFIRMED_EXISTING** — and now *proved by execution*: `rrf.structured_channel_rejected` throws on any other channel; the structured-not-fused check constructs a case that would win if fused. |
| SRD-003 | Registry canonical ID is parameter identity; Schema path is a locator | **CONFIRMED_EXISTING** — but **not enforced**: 75 unresolved references across 55 names (MC-AUD-006 stays OPEN). |
| SRD-004 | Embedding `space_lifecycle` separate from `migration_state` | **CONFIRMED_EXISTING** — schema splits them; retrieval suite asserts a `draining` space is not treated as `active`. |
| SRD-005 | No SQLite driver frozen without file-backed evidence | **CONFIRMED_EXISTING** — and evidence is now much stronger (55 checks, 0 failing). Driver still not frozen: no comparative evidence is obtainable. |
| SRD-006 | Keep LanceDB; do not claim FLAT exactness without a fixed-dataset comparison | **CONFIRMED_EXISTING** — comparison executed; the prior claim did **not** survive audit. |
| SRD-007 | M02 final permission check is fail-closed; allow-all only in isolated synthetic tests | **CONFIRMED_EXISTING** — synthetic matrix executed; hardened profile blocks every attack. |
| SRD-008 | Machine schema owns serialized shape; decision register resolves semantics | **CONFIRMED_EXISTING** — with one addition: the shipped Evidence schema **did not** resolve semantics correctly (OI-R-04) and has been corrected in `04-contracts/`. |

## 2. Decisions taken this round

| ID | Decision | Basis | Where recorded |
|---|---|---|---|
| **R4.1-D-01** | The `evidence.schema.json` v1 contract is **defective and superseded**; v2 is defined in `04-contracts/schemas/` | Proven by execution: a §3.2-compliant instance is rejected with `additionalProperties`, `required`, `enum` failures. v2 accepts it and still rejects stale aliases | `04-contracts/schemas/evidence.schema.json`, `SCHEMA-CONSISTENCY-REPORT.md §3` |
| **R4.1-D-02** | The prior LanceDB exactness claim (`5.7e-9` error vs `1e-5` tolerance) is **not acceptable as evidence** | Three independent defects: non-comparable quantities, tolerance 1757× the error, no stored dataset | `LANCEDB-EXACTNESS-VERIFICATION.md §3` |
| **R4.1-D-03** | LanceDB exactness is a property of the **scan path only**; indexed retrieval is approximate and its recall must be measured before acceptance | `createIndex()` ignores `indexType`/`distanceType` (parent-confirmed on 3 option sets); measured ANN recall@10 = 0.30, `nprobes` inert | `EV-LANCEDB-P001/P002`, `EV-LANCEDB-SCALE-004` |
| **R4.1-D-04** | `unicode61` is **not adequate** as the Chinese retrieval channel, and no `unicode61` tuning fixes it | 15/19 Chinese word queries hit; 4 return nothing; model fit run-equality 0.975 vs substring 0.600; tuning knobs are CJK no-ops | `FTS-CHINESE-VERIFICATION` / `03-verification/fts/TASK-RESULT.md §9.4` |
| **R4.1-D-05** | The recommended Chinese FTS option is `unicode61` + **indexed unigram/bigram shadow**: no dependency, no architecture change, 0.956 hit@10, cost 3.0× index bytes | Measured against 12 alternatives | `TASK-RESULT.md §14.3` (recommendation, **not** a freeze) |
| **R4.1-D-06** | `node:sqlite` is the SQLite driver for this environment, **within the scope verified**; it is not frozen | Only installable candidate; 55 checks pass; comparative evidence unobtainable (no package manager) | `SQLITE-DRIVER-VERIFICATION.md §7–8` |
| **R4.1-D-07** | A spec-compliant Evidence object was impossible to serialise; `original_text` and `speaker_id` are mandatory and `additionalProperties:false` must not be used to exclude authority fields | Authority §3.2 + executed conformance | as R4.1-D-01 |
| **R4.1-D-08** | The six identity `LEAKED` records are **negative controls**, not live vulnerabilities; the hardened profile blocks all of them | Contrastive runs: `cache.revocation_not_served`, `cache.hardened.cross_group_blocked`, `cache.stores_ids_only`, `atk.F4.hardened_denies` all PASS | `PARENT-CORRECTIONS` companion, identity area |
| **R4.1-D-09** | `tier_review_approved` in the retrieval prototype is **not a design decision** and must not be read as one | The name appears **zero times** in the 24 design documents | `05-audit/PARENT-CORRECTIONS.md §2` |
| **R4.1-D-10** | The `01-baseline-corpus` tree is read-only; the single BOM removal is a recorded deviation, not precedent | Measured: exactly 1 file differs, only the BOM | `05-audit/PARENT-CORRECTIONS.md §1` |
| **R4.1-D-11** | Evidence records must state the **failing sub-condition** in `actual`, not a passing prefix | One agent FAIL was a false failure that took independent re-execution to detect | `EV-RETRIEVAL-P001` |

## 3. Escalated to the user — architecture decisions this round cannot take

| ID | Question | Options | Why it cannot be decided here |
|---|---|---|---|
| **E-1** (= OI-R-01) | Which R4.1 document set is authoritative: `Memory-Core-R4.1-Design` (with `contracts/`) or `R4.1-UNIFIED-MEMORY-CORE`? | P1 design-package wins + unified as annex · P2 unified wins · P3 merge · P4 defer | The two sets are **mutually unreferencing** (0 citations each way) and diverge in 7 architectural places. Choosing changes which documents are law. Agent H gave a reasoned recommendation (P1) but correctly refused to decide. |
| **E-2** (= OI-R-07) | What are the `memory_tier` transition rules, or is tier fixed at creation? | (a) define a transition table · (b) tier static, only `status` moves · (c) V1 treats tier as a label | **No tier transition rule exists in any of the 24 documents.** The design depends on tier for retention/expiry. This is a missing design section, not a bug. |
| **E-3** | Is `content_kind` abolished (V2 `00:76`) or mandatory with 8 enums (`01 §7.3`, `02:630`, `memory-object.schema.json`)? | (a) enforce V2 abolition and migrate the schema · (b) retain `content_kind` and withdraw the abolition | The corpus asserts both simultaneously. An implementer picking either is defensible, which means the contract is not. |
| **E-4** | Which object model is canonical: `memory-object.schema.json` (v1) or `memory-unit.schema.json` (V2)? | keep both with deprecation markers · delete v1 · regenerate | `contracts/` ships **two unmarked object models**. A schema-driven agent picks one at random. |
| **E-5** (= MC-AUD-009) | Backup retention period and policy | — | No number exists anywhere; inventing one would fabricate policy. |
| **E-6** | Is the vector channel allowed to run **indexed** (approx, recall 0.30 today) or **scan-only** (exact, slower)? | indexed with measured recall gate · scan-only · index only above a size threshold | A product trade-off with real cost consequences, and the index options are broken in this SDK. |

## 4. Deferred with explicit gates

| Item | Gate |
|---|---|
| Token-budget thresholds | a real tokenizer for the target model; today every number is `VALIDATION_PENDING` and none was invented |
| Driver freeze | a package manager + native toolchain, then this same harness against ≥1 alternative |
| Real host identity mapping | real DSH/QQ Bridge access, which is **barred this round** |
| Emotion runtime | separate authorization (`emotion.*` stays default-off) |
| LLM harness | a mock provider harness; no real model call |

## 5. What this register deliberately does not do

- It does not mark anything `FROZEN` on the strength of a green summary. Every freeze claim above
  carries its scope.
- It does not resolve E-1…E-6 by preference. Those are architecture-level and belong to the user.
- It does not convert agent recommendations into decisions. D-04 and D-05 record a *recommendation*
  and its measurements; adoption is a separate act.
