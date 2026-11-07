# Memory Core R4.1 Second Round Decision Register

Date: 2026-11-04. Status: PARTIALLY DECIDED; not a global freeze declaration.

| ID | Issue | Final decision in this round | Rationale | Affected contracts | Verification/status |
|---|---|---|---|---|---|
| SRD-001 | P0 authority | User task authorizes document review, contract edits, P0 verification and isolated harnesses only; it does not authorize product M01/M02 implementation, real data, host integration, real permission writes or emotion enablement. | Explicit task §3.1. | DESIGN-STATUS, README, readiness report | Decision fixed; full document synchronization pending. |
| SRD-002 | RRF channels (OI-022) | Only `fts` and `vector` participate in RRF. `structured` is filter/router/constraint only. | Explicit task §3.5 and avoids duplicate weighting. | retrieval response schema, retrieval config, algorithm, tests | Schema changed; global textual consistency check pending. |
| SRD-003 | Parameter identity (OI-023) | Parameter identity is the canonical ID in the parameter registry; physical Schema path is a locator only. Do not treat Schema path and registry ID as co-equal IDs. | Existing registry already defines stable `recall.*` IDs; avoids unstable IDs coupled to document/schema layout. | Registry, config descriptions, tests/log fields | Decision resolved; full inventory/reconciliation pending. |
| SRD-004 | Embedding space status (OI-024) | Separate `space_lifecycle` (`active`,`draining`,`retired`) from `migration_state` (SM-8). | A storage-space lifecycle is not the migration workflow lifecycle. | embedding-space schema, 01/07, SM-8 | Schema changed; full contract validation pending. |
| SRD-005 | SQLite driver | No driver is selected from module availability alone. `node:sqlite` is available in Node v24.22.1, but required file-backed validation is incomplete. | A smoke test is not evidence for concurrency, backup, migration, or crash recovery. | DatabaseAdapter | BLOCKED pending complete file-backed harness. |
| SRD-006 | LanceDB exact scan | Keep LanceDB as the vector store; do not claim FLAT exact behavior without fixed-dataset comparison. | Explicit constraint in task §3.4. | VectorStoreAdapter | BLOCKED; SDK not installed / no benchmark run. |
| SRD-007 | Permission | M02 permission final check remains fail-closed; allow-all only in isolated synthetic tests. Real host identities/data remain barred pending host mapping tests. | Prevents private/group scope leakage. | M02/API/security | Contract principle retained; synthetic full suite not run. |
| SRD-008 | Schema source | Existing conflict baseline D-1 is superseded for this round by task-directed field decisions; each machine schema is authoritative for serialized shape, while decision register resolves semantics and conflicts. | Enables consistent contract output without erasing decision rationale. | all schemas/API | Requires global reconciliation; not yet complete. |

## Deferred with explicit gates

- OI-001 LanceDB FLAT: run fixed dataset + brute-force cosine comparison before M02 acceptance.
- OI-002 SQLite driver: complete file-backed Windows matrix before M01 acceptance.
- Host identity, session/group attribution, revoke propagation: validate before any real user data/host integration.
- Emotion runtime: disabled until separately authorized; schemas may be contract-validated.
- Benchmark quality thresholds: establish from P3 baseline; do not invent performance claims.
