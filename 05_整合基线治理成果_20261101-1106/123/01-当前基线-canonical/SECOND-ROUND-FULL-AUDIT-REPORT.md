# Memory Core R4.1 Second-Round Full Audit Report

Date: 2026-11-04 (Asia/Shanghai)  
Baseline: `Memory-Core-R4.1-Design`  
Status: **Second-round work is materially advanced but not complete; package-wide freeze has not been achieved.**  
Final implementation status: **`NOT_READY_FOR_IMPLEMENTATION`**.

## 1. Scope and evidence discipline

Reviewed the R4.1 package inventory (69 files after second-round evidence was added), core documents 00–15, decision/authority/status/issue/conflict files, all contract folders, diagrams, test specifications and prior review reports. The previous backup is `C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design.backup-20261104-192558`.

The parent `work` directory contains legacy M01–M05 documents (`03-m01.md` through `03-m05.md`, plus `03-m01-problem.md`). Their headers indicate mixed draft generations and scopes (including R3 / QQ Bridge). They are historical sources, not silently promoted authority. A full clause-by-clause inheritance matrix has **not** been completed; see MC-AUD-017.

Evidence grades: static reading is E1; JSON parse/meta-schema/ref checks are E2; isolated SQLite/FTS smoke runs are E3. No E4 LanceDB integration or E5 real-host verification was performed.

## 2. First-round issue disposition

| Issue | Disposition |
|---|---|
| MC-AUD-001 P0 authorization | Resolved in principle: this task authorizes document work and isolated verification only, not product implementation or real data. DESIGN-STATUS updated; remaining stale references still require sweep. |
| MC-AUD-002 contract baseline | Partially resolved: semantic authority and serialized-schema roles recorded; cross-document reconciliation remains open. |
| MC-AUD-003 SQLite driver/schema | Not resolved: `node:sqlite` is available and smoke-tested, but driver comparison and full file-backed matrix are incomplete. |
| MC-AUD-004 LanceDB FLAT | Not resolved; blocks M02 acceptance. SDK test not run. |
| MC-AUD-005 RRF channels | Decision resolved: only `fts` and `vector`; structured filters/routes only. Retrieval response schema updated; textual references still require complete regression sweep. |
| MC-AUD-006 parameter IDs | Decision resolved to registry canonical IDs, with Schema paths as locators. Full parameter/reference inventory not complete. |
| MC-AUD-007 permission identity | Fail-closed rule retained. Synthetic matrix and real host identity verification not run. |
| MC-AUD-008 Outbox retry | Open; operational retry/lease values and replay suite not fully validated. |
| MC-AUD-009 backup retention | Open; no production retention commitment. |
| MC-AUD-010 Evidence constraints | Open; schema relations not yet proven against actual SQLite DDL and negative tests. |
| MC-AUD-011 tier lifecycle | Open; one authoritative full lifecycle matrix not produced/verified. |
| MC-AUD-012 acceptance thresholds | Open/deferred to baseline evidence; no invented thresholds. |
| MC-AUD-013 status axes | Updated in DESIGN-STATUS to document/review/decision/freeze/test/acceptance axes. |
| MC-AUD-014 evidence bundle | Partial; SQLite and schema evidence exists, remaining harness evidence absent. |

## 3. Architecture, module boundaries and lifecycle

The design package establishes SQLite as the sole authority, FTS5/LanceDB as rebuildable indexes, a staged pipeline, M01–M05 boundaries, and fail-closed permission checks. However, API contracts and domain documents still contain historical names and potentially mismatched request/response representations. No complete dependency matrix, data ownership matrix, permitted/forbidden import matrix, or startup/shutdown order was validated in this pass.

Memory lifecycle and candidate status are represented in separate state-machine and schema material, but a single exhaustive tier matrix covering entry/retention/exit/duration/visibility/conversion/demotion/merge/split/delete/restore/permission/evidence was not independently verified. The final report therefore does not claim lifecycle freeze.

## 4. Data model and Evidence

The package has dedicated Memory, Evidence, Event, EmotionContext, Profile, relation and provenance concepts with machine schemas. The key invariant remains: formal memory writes must preserve source/provenance; LLM output is a proposal and cannot directly mutate authoritative data. Event facts and emotion context are separate by design.

Still open: full SQLite DDL/FK/uniqueness parity, evidence sharing/deletion semantics, retention, and transactionally valid references across all write APIs. JSON Schema presence does not establish relational integrity.

## 5. Retrieval, vector and text search

RRF has been adjudicated to two channels: `fts` and `vector`; structured filtering/router is excluded from RRF. The retrieval response Schema now enforces those channels. Embedding space Schema now separates `space_lifecycle` from `migration_state`.

SQLite/FTS5 local tests: Node v24.22.1, SQLite 3.53.4, Windows x64. File DB WAL, DDL, FK, CHECK, unique, rollback, reopen, migration smoke, VACUUM INTO backup/restore, lock contention and child-process crash recovery passed. FTS5 English query passed; `unicode61` Chinese test had no hit. This blocks any claim of adequate Chinese tokenization and leaves full driver comparison/long transaction stress open.

LanceDB 0.41.0 was installed in the isolated evidence folder using pnpm. With no ANN index created, a small fixed dataset matched brute-force cosine order/Top-K; max distance error was 5.7e-9 (tolerance 1e-5). Filter, empty result, invalid dimension rejection, update/delete and reopen persistence passed. This is a partial VF-001 result, not the complete required matrix or performance validation; M02 acceptance remains blocked pending expanded test coverage.

## 6. LLM, permissions and emotion

LLM architecture limits models to proposal/interpretation and keeps deterministic validation, permission, persistence and version transitions outside the model. Mock failure-mode tests were not executed.

Permission design is fail-closed. The synthetic permission matrix (owner/non-owner/missing identity/group/private/revocation/expiry/evidence access) was not run. Real QQ Bridge/DSH identity, group attribution, tenant and session mapping remains an external E5 gate; no real user data may be connected before it passes.

Event/EmotionContext separation and runtime-off defaults remain. No emotion runtime was enabled or validated.

## 7. First-stage implementation feasibility

P0 isolated verification work has begun. M01 cannot start product implementation until driver choice, DDL/schema, transaction, migration and file recovery evidence are complete and separately authorized. M02 cannot start acceptance until LanceDB exactness, FTS tokenizer behavior, RRF consistency, permission contract, API/schema and budget tests are complete. P3 acceptance and real-host/P9 work remain outside current authorization.

## 8. Newly recorded issues

The detailed, uniquely identified issue list is in `SECOND-ROUND-ISSUE-REGISTER.md` (MC-AUD-001–019). P0/P1 blockers include SQLite driver/DDL proof, LanceDB exactness, API/status/scope/schema mismatches, permission identity gates and legacy-source reconciliation. P2 items include operational values, evidence bundle and benchmark baselines.

## 9. Contract freeze statistics

A trustworthy numeric count of frozen contracts cannot be supplied: the package has many schemas/API/state/config contracts, but freeze requires global semantic consistency and validation evidence. Current state: **0 package-wide contract families accepted as fully verified/frozen by this review**; written contract families are enumerated in `CONTRACT-FREEZE-REPORT.md`; several decisions are fixed but verification and synchronization remain incomplete.

## 10. Tests and evidence

- JSON: 31 Draft 7 schemas passed meta-schema checks and local `$ref` resolution with jsonschema 4.26.0. Positive/negative fixtures also passed for updated retrieval-response and embedding-space schemas. Other schema business fixtures and version compatibility remain untested.
- SQLite/FTS5: partial file-backed isolated suite passed as detailed in `evidence/second-round/sqlite-file-result.json`; Chinese query was a no-hit.
- LanceDB: partial synthetic fixed-set verification; Outbox: isolated behavioral simulation passed normal/idempotent/partial-failure/replay cases but not product worker; full API contract, permission matrix, token budget, LLM Harness and end-to-end: NOT RUN.

## 11. Required next work

1. Finish legacy M01–M05 clause mapping and record accepted/deprecated decisions.
2. Resolve OI-016–021 in API/schema/algorithm/tests; align request, response envelope, status enums, N/T feature semantics, scope field and all permission test IDs.
3. Sweep every package file for stale RRF and parameter aliases; align JSON Schema, registry and tests.
4. Run file-backed SQLite driver comparison including concurrent writers, lock contention, migration, backup/restore and crash recovery; freeze one driver only after evidence.
5. Install/run the approved LanceDB Node SDK in an isolated harness and compare exact Top-K/rank/distance to brute-force cosine on a hashed dataset.
6. Run FTS tokenizer matrix and choose an explicit Chinese strategy based on results.
7. Run Outbox, permission, token-budget, API, LLM and segmented E2E harnesses with reproducible logs.
8. Update all status, issue, decision and changelog files; perform final global consistency sweep before declaring any package-wide freeze.

## 12. Conclusion

The architecture has substantial written detail and several sound boundaries, but this second-round execution has not met its stated completion criteria. A number of contract conflicts remain, core storage/vector verification is incomplete, and the legacy-source reconciliation is not finished. No third-round design discussion is needed for the already adjudicated decisions; however, required verification and synchronization work remains. **Do not hand this package to a code Agent as a frozen, independently implementable V1 baseline.**
