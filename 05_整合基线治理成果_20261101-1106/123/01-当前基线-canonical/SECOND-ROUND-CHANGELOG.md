# Second-Round Changelog

Date: 2026-11-04. All edits were made after a full package backup at `C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design.backup-20261104-192558`.

| File | Change | Reason | Contract impact | Compatibility | Decision |
|---|---|---|---|---|---|
| `contracts/schemas/retrieval-response.schema.json` | `source_channels` restricted to unique `fts`/`vector`; structured role clarified. | Apply explicit RRF decision. | Retrieval response | Breaking for any consumer emitting structured as a ranked channel; required correction. | SRD-002 |
| `contracts/schemas/embedding-space.schema.json` | Split status into required `space_lifecycle` and `migration_state`. | Separate space lifecycle from migration lifecycle. | Embedding space | Breaking schema shape; migration required. | SRD-004 |
| `OPEN-ISSUES.md` | OI-022/023/024 changed from unresolved proposals to adjudicated decisions with remaining sync/verification gates. | Avoid stale user-decision status. | Issue register | Administrative. | SRD-002/003/004 |
| `PENDING-USER-DECISIONS.md` | Mark first-round pending list historical and defer to second-round decisions/verification. | Task authorizes normal engineering decisions. | Authority/status | Administrative. | SRD-001–004 |
| `README.md` | Clarified historical status of pending decisions. | Avoid conflict with current authority. | Navigation | Administrative. | SRD-001 |
| `DESIGN-STATUS.md` | Split status axes and correct schema-validation/authorization claims. | Prevent document readiness from implying tested/frozen. | Status | Administrative. | SRD-001 |
| `SECOND-ROUND-*`, `CONTRACT-FREEZE-REPORT.md`, `TECHNICAL-VERIFICATION-REPORT.md`, `V1-IMPLEMENTATION-BASELINE.md` | Added second-round reports. | Required deliverables; explicitly retain blockers. | Process | New documents. | SRD series |
| `evidence/second-round/*` | Added isolated smoke and schema-validation harnesses/results. | Reproducible evidence. | Test evidence | No product code. | — |

Known limitation: global cross-document synchronization is incomplete; these modifications do not constitute a package-wide freeze.
