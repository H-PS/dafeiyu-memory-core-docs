# V1 Implementation Baseline (review candidate)

Status: **NOT READY FOR PRODUCT IMPLEMENTATION**. This is a gated baseline, not authorization to build product M01/M02.

## Scope

V1 architecture covers P0–P6; current explicitly authorized verification scope is document review, contract edits, and isolated P0 harnesses only. No formal M01/M02 product code, real user data, real DSH/QQ Bridge integration, production allow-all, or emotion runtime is authorized by this review task.

## Authority

1. User task and accepted historical decisions.
2. `DD-AUTHORITY-V2.md` for semantic decisions, subject to explicit second-round decision register.
3. `contracts/` for serialized shapes and error/state interfaces.
4. `00`–`15` domain specs for algorithms and implementation behavior.
5. README is navigation only; stale summaries do not override authority.

## Fixed design decisions

- SQLite is the sole authoritative store; FTS5/LanceDB are derived and rebuildable.
- LanceDB remains the vector store. Exact FLAT behavior is unverified and blocks M02 acceptance.
- FTS5 handles text retrieval; tokenizer choice/Chinese behavior requires measurement.
- RRF channels: `fts` and `vector`; structured filters/router only.
- M02 must perform final permission check; fail closed. Test allow-all is isolated synthetic-only.
- LLM produces proposals; deterministic code owns validation, authorization, persistence, versioning and state transitions.
- Emotion runtime remains disabled; Event and EmotionContext remain separate.
- Outbox is transactional with authoritative writes; retries must be idempotent.

## Gate order

1. P0: resolve all schema/contract deltas; run file-backed SQLite matrix; run LanceDB exactness benchmark; validate JSON schemas and API contracts.
2. P1 M01: only after SQLite driver/schema/migration/transaction contract is evidence-backed and authorized separately.
3. P2 M02: only after LanceDB/FTS behavior, RRF, permission final-check and tokenizer/budget contracts are validated.
4. P3: integration/benchmark/failure recovery/E2E acceptance.
5. P4+: separate phase authorization.

## Required module contracts

- `DatabaseAdapter`: transactions, migrations, backup, integrity, parameterized SQL only; no business module depends on driver-specific API.
- `VectorStoreAdapter`: LanceDB-backed store, embedding-space isolation and exactness behavior; no SQLite vector substitution.
- `EmbeddingProvider`: explicit model/version/dimension/normalization/timeout/retry; model spaces never mix.
- `PermissionCheck`: ALLOW/DENY/INDETERMINATE; errors/timeouts/unknown => deny/exclude.
- API and state payloads must conform to versioned schemas and stable error envelope.

This candidate deliberately refuses to fill unspecified implementation details with guesses. It cannot be used as a stand-alone implementation handoff until the open gates in `SECOND-ROUND-ISSUE-REGISTER.md` close.
