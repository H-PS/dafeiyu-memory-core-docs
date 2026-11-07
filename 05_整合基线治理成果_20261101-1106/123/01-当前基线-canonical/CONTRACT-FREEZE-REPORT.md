# Contract Freeze Report — Memory Core R4.1

Date: 2026-11-04. Overall state: **NOT FROZEN**. Individual written contracts are not equivalent to verified/frozen interfaces.

| Contract family | Authority | Current decision state | Verification state | Gate |
|---|---|---|---|---|
| Core architecture/modules | `DD-AUTHORITY-V2.md`, `00-总体架构与设计原则.md` | Written; module-stage boundary requires status synchronization | Static review in progress | Publish dependency/ownership matrix and initialize/shutdown order. |
| Memory/Evidence/Event/Profile schemas | `contracts/schemas/*.schema.json`, 01/12/13 | Written, not fully reconciled against SQL constraints | JSON syntax only | Validate schema refs/instances and actual DDL constraints. |
| SQLite schema/adapter | 01, `contracts/api/api-contracts.md` §8 | Driver not selected; adapter contract exists | Smoke only | Complete file-backed driver matrix. |
| LanceDB/vector adapter | 02/05, embedding/search schemas | LanceDB retained; FLAT behavior not verified | NOT RUN | SDK installation and exactness benchmark. |
| FTS5 | 01/02/05 | Design described; Chinese tokenizer behavior open | NOT RUN | Execute tokenizer matrix against selected SQLite runtime. |
| RRF | 02 + retrieval config/response schema | Two channels frozen semantically: fts/vector; structured excluded | Schema syntax parsed; global consistency pending | Contract harness and deterministic ranking tests. |
| API envelopes | `contracts/api/api-contracts.md`, request/response schemas | Multiple response layers described; not fully reconciled | NOT RUN | Schema-backed request/response contract tests. |
| Permission final check | 04, permission-context schema, API §6 | Fail-closed contract | NOT RUN (full suite) | Synthetic matrix; real host mapping remains separate gate. |
| Outbox/state machines | 03, SM-1…SM-8, outbox schema | Written; retry/lease operational values remain partly experimental | NOT RUN | Crash/replay/partial-index harness. |
| Parameters/config | 15, contracts/config | Registry ID is canonical; inventory reconciliation incomplete | Static partial scan | 100% reference map and schema/registry consistency check. |
| Token budget | retrieval config, 02/06/14 | Token and character budgets remain separate | NOT RUN | tokenizer-backed boundary and citation-integrity tests. |
| LLM tasks | 09/10/14 | Candidate-only, no direct DB write | NOT RUN | Mock harness for timeout/schema/retry/idempotency. |
| Migration/backup | 05/07, SM-6/SM-8 | Design exists; some operational policy remains deferred | NOT RUN | migration and restore evidence before production data. |

A contract becomes `CONTRACT_FROZEN` only after authoritative file, version, dependent schemas, error behavior and validation evidence are recorded. This report does not claim that condition is met package-wide.
