# MODULE-BOUNDARY-MATRIX — Memory Core R4.1

Date: 2026-11-05 (Asia/Shanghai)
Basis: `00-总体架构与设计原则.md`, `DD-AUTHORITY-V2.md §5/§7`, `04-权限与治理规范.md`, `contracts/api/api-contracts.md`

Purpose: state each module's **responsibility, non-responsibility, data ownership, and permitted
dependencies**, so an implementer cannot drift across a boundary by accident. Where a boundary is
contradicted by the corpus, that is recorded rather than smoothed over.

---

## 1. The matrix

| | **M01 Storage** | **M02 Retrieval** | **M03 Extraction** | **M04 Integration** | **M05 Permission** |
|---|---|---|---|---|---|
| **Owns (authority)** | SQLite authority: MemoryUnit, Evidence, Event, EmotionContext, Profile, relations, versions, Outbox rows, tombstones | Nothing authoritative. Owns the *retrieval plan and ranked candidate set* | Nothing authoritative. Owns *candidates* in the queue | Profile derived views and `validation_state=confirmed` decisions | Nothing data-bearing. Owns *permission decisions* |
| **Does NOT own** | Retrieval ranking; extraction; conflict decisions; permission policy | Storage; writes; candidate commit; permission policy | Persistence; ordering vs profile; commit | Candidate extraction; storage; permission | Data; retrieval algorithm |
| **Inputs** | validated objects + transaction context | `RetrievalRequest` | source batches from Pipeline | candidates + evidence + profile state | `PermissionContext` + resource descriptor |
| **Outputs** | persisted objects, version history, Outbox events | `RecallResult` / `MemoryContext` | candidate set with quality flags | profile updates, conflict decisions | ALLOW / DENY / INDETERMINATE |
| **Persists?** | **YES — sole writer** | no | no | *through* M01 only | no |
| **May call** | nothing upward | M01 (read), M05 (final check), FTS5, LanceDB | M04 Gateway (LLM), M05 | M01, M05, L2 LLM via Gateway | nothing (leaf) |
| **Must not touch** | — | M01 internal tables directly; must go through the adapter | the DB at all | SQL directly | the DB at all |
| **Error responsibility** | STORAGE_ERROR, VERSION_CONFLICT, DUPLICATE_REQUEST | INDEX_ERROR, EMBEDDING_ERROR, RETRIEVAL_FAILED, BUDGET_EXCEEDED | VALIDATION_ERROR, EMBEDDING_ERROR | VALIDATION_ERROR, conflict errors | PERMISSION_DENIED, PERMISSION_ERROR |
| **Permission responsibility** | enforces nothing; *trusts* the caller's M05 decision but is the last place a wrong write could be refused | **final check** — fail-closed, per item | pre-check via M05 gate (stage 15) | via M05 | **sole policy authority** |
| **Lifecycle responsibility** | state transitions only as *executed*, never *decided* | none | candidate state | decides `confirmed`, profile version | none |

## 2. Dependency graph (must stay acyclic)

```text
Pipeline (coordinator, no data ownership)
   ├── M03 ──► LLM Gateway (L1/L2)
   │     └───► M05 (stage-15 pre-gate)
   ├── M04 ──► LLM Gateway (L2)
   │     ├───► M05
   │     └───► M01 (write, as executor)
   └── M01 (commit) ──► Outbox ──► FTS5 / LanceDB

Query path:
   Host ──► M02 ──► M01 (read authority)
             ├──► FTS5 (derived)
             ├──► LanceDB (derived)
             └──► M05 (FINAL check, fail-closed)
```

**Acyclic checks performed:**

| Potential cycle | Status |
|---|---|
| M02 → M01 → M02 | **none** — M01 never calls M02; index maintenance is via Outbox, asynchronous |
| M04 → M03 → M04 | **none** — M03 produces candidates; M04 consumes them; no back-call |
| M05 → any module | **none** — M05 is a leaf that answers questions and calls nothing |
| Pipeline → M01 → Pipeline | **none** — M01 commits and returns; it does not re-enter the Pipeline |

## 3. Data ownership conflicts found

### 3.1 `validation_state=confirmed` has a single declared writer (M04) — but `memory-unit.schema.json` permits it in ACTIVE

The schema's `allOf` requires `status=ACTIVE ⇒ validation_state ∈ {confirmed, disputed}`. So a
memory cannot be ACTIVE unless it is already confirmed or disputed. Combined with "only M04 writes
confirmed", this means **every ACTIVE memory must pass through M04**, including ones created by
revision. That is a real constraint, and it is consistent — but it makes M04 a bottleneck on a path
the design also describes as M01-authorised (`09 §7.24`). **Recorded as an open boundary question:**
can M01 create an ACTIVE memory in a restore/migration path where M04 is unavailable? The documents
say M04 must be the sole confirmed-writer, and do not describe that exception.

### 3.2 M02 performs the *final* permission check, but stage 15 of the Pipeline also gates writes

Two permission gates exist at different points for different directions (write vs read), which is
correct. The risk is **drift**: if the read-path final check were ever relaxed because "the Pipeline
already checked", the write-side rationale would be misapplied. The contract states they are separate;
nothing in the corpus forbids conflating them. **Recommendation:** one line in `04` saying the write
gate does not substitute for the read final check, and vice versa.

### 3.3 The Compression-adjacent concern (FUT-09) would add a plane 00 §0.4 does not list

`00 §0.4`'s module table has no entry for a "context compression layer". The FUT-09 draft in
`E:\[PROJECT]` correctly flags this (its OI-A). **Not adopted** — recorded so that if it is ever adopted,
the module table must gain a row and the ownership matrix above must be revised rather than quietly
extended.

## 4. Forbidden access, stated explicitly

| Forbidden | Why |
|---|---|
| Any module reading SQLite tables directly instead of via `DatabaseAdapter` | breaks the "禁止任意 SQL" rule and makes migration impossible |
| M02 writing to LanceDB or FTS5 synchronously during a query | derived indexes are Outbox-maintained; a synchronous write during read would put index maintenance on the query latency path |
| M03 or M04 writing to the authority store directly | M01 is the sole writer; confirmed-by-M04 must still be executed by M01 |
| M05 reading memory bodies to decide | permission decisions must not depend on content, or the decision leaks the content |
| Any module falling back to a SQLite vector store | explicitly forbidden (`fallback_to_sqlite_vector` is always false) |
| Any module silently degrading to another retrieval implementation | must raise `INDEX_ERROR`/`RETRIEVAL_FAILED` (INV-11) |

## 5. Boundary problems requiring a design decision

| ID | Problem | Impact |
|---|---|---|
| MB-1 | M04 is the sole `confirmed` writer, but no exception path is described for restore/migration | a restore may produce ACTIVE memories with no M04 involvement, or must defer activation |
| MB-2 | Write-side gate vs read-side final check are not explicitly declared non-substitutable | a future implementer could rationalise one away |
| MB-3 | Two documents disagree on whether `structured` feeds RRF (`02:344/346`, `00:638` vs `11:617`, `06:558`, OI-022) | implementers of M02 would build different pipelines from the same corpus |
| MB-4 | The module table has no row for a compression/context plane, yet a draft proposes one | adopting it silently would break the matrix above |

## 6. What this matrix does not claim

It does not claim the modules are implementable today — see `IMPLEMENTATION-READINESS-MATRIX.md`,
where M01 is `PARTIAL`, M02 `NOT_READY`, M03/M04 `NOT_READY`, M05 `BLOCKED`, and **none is
authorized**.
