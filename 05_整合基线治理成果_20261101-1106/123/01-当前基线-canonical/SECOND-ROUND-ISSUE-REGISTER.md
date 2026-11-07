# Second Round Issue Register

Date: 2026-11-04. This is the live issue ledger for the second-round pass; issues remain open until evidence and synchronization are complete.

| ID | Severity | Issue | Decision/action | Status | Gate |
|---|---|---|---|---|---|
| MC-AUD-001 | P0 | P0 authorization ambiguity | Restricted to document work and isolated verification; no product implementation or real data. | DECIDED; sync pending | Readiness/status docs updated consistently. |
| MC-AUD-002 | P0 | Contract source and unresolved decision set | Schema owns serialized shape; task-directed semantic decisions recorded in second-round register. | PARTIAL | Reconcile all API/schema/text references. |
| MC-AUD-003 | P0 | SQLite driver/schema not validated | node:sqlite file-backed WAL/constraints/rollback/migration smoke/backup-restore/lock/crash tests pass; no driver comparison or stress matrix. | PARTIAL; driver selection not frozen | Complete comparison and concurrent/long transaction matrix. |
| MC-AUD-004 | P1 | LanceDB FLAT exactness unverified | Preserve LanceDB; no substitute or exactness claim. | BLOCKED | SDK benchmark against brute-force cosine. |
| MC-AUD-005 | P1 | RRF channel inconsistency | Freeze fts/vector only; structured is filter/router. | DECIDED; schema changed | Sweep all text, tests, response schema and verify. |
| MC-AUD-006 | P1 | Parameter ID inconsistencies | Registry canonical IDs; schema paths are locators. | DECIDED; partial | Full reference scan and correction. |
| MC-AUD-007 | P1 | Real host identity/session attribution unverified | Default deny; no real data until host mapping tests. | OPEN / external gate | Synthetic matrix then host E5 validation. |
| MC-AUD-008 | P2 | Outbox retry/lease thresholds | Keep explicit config defaults experimental; no silent hardcoding. | OPEN | Run replay/crash/partial failure harness. |
| MC-AUD-009 | P2 | Backup retention and restore policy | No production retention claim; design describes recovery. | OPEN | Decide operational retention before real data. |
| MC-AUD-010 | P1 | Evidence FK/relationship constraints not proven in DDL | Require FK/unique/CHECK constraints in actual schema. | OPEN | DDL and negative tests. |
| MC-AUD-011 | P1 | Tier lifecycle semantics dispersed | Existing state and tier definitions need one validated lifecycle matrix. | OPEN | Cross-doc lifecycle consistency review. |
| MC-AUD-012 | P2 | Acceptance thresholds / tokenizer metrics incomplete | Do not invent benchmark pass thresholds. | DEFERRED with gate | Establish baseline during P3 before acceptance. |
| MC-AUD-013 | P2 | DESIGN-STATUS mixes document readiness with runtime readiness | Split six axes per task. | OPEN | Rewrite module status matrix. |
| MC-AUD-014 | P2 | Evidence bundle incomplete | Capture reproducible logs, config, versions, data hashes. | PARTIAL | Add remaining validation artifacts. |
| MC-AUD-015 | P1 | OI-016–021 cross-document API/status/feature/scope/test mismatches | Must align request/response layering, enums, features, schema, permission tests. | OPEN | Contract tests and full schema diff. |
| MC-AUD-016 | P1 | OI-024 embedding space lifecycle conflated with migration state | Split in schema into `space_lifecycle` and `migration_state`. | DECIDED / schema edited | Verify all consumers and JSON Schema semantics. |
| MC-AUD-017 | P1 | Old M01–M05 historical source reconciliation incomplete | Old docs are not authority but must be compared for unported valid constraints. | OPEN | Full-text mapping table. |
| MC-AUD-018 | P2 | JSON schema fixtures incomplete | 31 Draft-07 schemas meta-validated and refs resolved; updated retrieval/embedding schemas valid/invalid cases pass. | PARTIAL PASS | Add business fixtures and version compatibility tests. |
| MC-AUD-019 | P2 | API, permission, token budget, LLM and E2E harnesses not run | Design tests exist, execution evidence absent. | OPEN | Isolated harness suite and explicit MOCKED labels. |
