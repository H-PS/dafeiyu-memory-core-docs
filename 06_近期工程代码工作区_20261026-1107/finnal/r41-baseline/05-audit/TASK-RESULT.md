# TASK RESULT — Agent H · historical inheritance + global cross-document consistency

**Task:** Memory Core R4.1 baseline, round 4.1
**Agent:** H · working root `E:\[PROJECT]\r41-baseline`, write scope `05-audit/` + the two evidence targets
**Date:** 2026-11-05 (Asia/Shanghai)
**Brief:** `02-shared/briefs/H-historical.md`

## 1. Authorization status (unchanged by this work)

| Axis | Value | Basis |
|---|---|---|
| `CODE_IMPLEMENTATION_SCOPE` | ISOLATED_VERIFICATION (analysis tooling only) | `00-PARENT-MANDATE.md` |
| `PRODUCT_IMPLEMENTATION_AUTHORIZATION` | **NOT_AUTHORIZED** | idem; nothing here upgrades it |
| `DESIGN_CLOSURE_STATUS` | PARTIAL | parent register §3.3 |
| `CONTRACT_FREEZE_STATUS` | NOT_FROZEN | parent register §3.3 |

No product code was written. **I** created, modified or deleted no file under `01-baseline-corpus/` — the scanner opens the corpus read-only and only `05-audit/`, `06-evidence/audit.jsonl` and `06-evidence/raw/audit/` were written.

**Caveat found while verifying that claim:** `01-baseline-corpus/design-package/contracts/schemas/read-response.schema.json` has mtime **2026-11-05 20:11:26** — inside this session's audit window, while every other corpus file is dated 2026-11-04. The write was not mine. I re-ran the full scan afterwards: the issue set is byte-identical (`1782` unresolved, `295` severity-1), so no finding in this report depends on that file, but the corpus was not frozen during the round. Recorded as `EV-AUDIT-H-006`; mtime cannot attribute the write, and the register already logs a BOM fix to that very file under OI-R-05 (`SECOND-ROUND-FINAL-ISSUE-REGISTER.md:84`).

## 2. Deliverables

| # | File | What it is | Status |
|---|---|---|---|
| 1 | [`05-audit/SECOND-ROUND-HISTORICAL-INHERITANCE-MATRIX.md`](SECOND-ROUND-HISTORICAL-INHERITANCE-MATRIX.md) | **MC-AUD-017**, the clause-level inheritance matrix: 121 legacy clauses, 6-way classification, both anchors per row | COMPLETE |
| 2 | [`05-audit/CROSS-DOCUMENT-CONSISTENCY-MATRIX.md`](CROSS-DOCUMENT-CONSISTENCY-MATRIX.md) | 10 findings (SC-01…SC-10) across 6 axes, plus the B-4 two-document-set option analysis | COMPLETE |
| 3 | [`05-audit/GLOBAL-REFERENCE-ISSUES.md`](GLOBAL-REFERENCE-ISSUES.md) | Real scan counts, per-code definitions, top offenders, systematic causes, reproduction | COMPLETE |
| 4 | [`05-audit/scan-references.cjs`](scan-references.cjs) | The reference scanner: plain CommonJS, `node` directly, no dependencies, 7 canaries, exit 0/1/2 | RUNS, exit 0 |
| 5 | [`05-audit/tools/param-id-count.cjs`](tools/param-id-count.cjs) · [`tools/schema-probe.cjs`](tools/schema-probe.cjs) | The two bounded probes whose numbers appear in the reports | RUN, exit 0 |
| 6 | `06-evidence/audit.jsonl` | 6 evidence records, `EV-AUDIT-H-001…006` | APPENDED |
| 7 | `06-evidence/raw/audit/` | `scan-references.json` (1 167 904 B), `.log`, `.run2.json`, `.postchange.json`, `corpus-digest.json`, `param-id-count.log`, `schema-probe.log` | WRITTEN |

## 3. What was executed (not asserted)

```
node .\r41-baseline\05-audit\scan-references.cjs                 -> exit 0, canaries 7/7
node .\r41-baseline\05-audit\scan-references.cjs --json ...      -> 06-evidence/raw/audit/scan-references.json
node .\r41-baseline\05-audit\scan-references.cjs --json ...run2  -> payload identical ignoring duration_ms
node .\r41-baseline\05-audit\tools\param-id-count.cjs            -> 53 / 36 / 20
node .\r41-baseline\05-audit\tools\schema-probe.cjs              -> v1 vs V2 object split, 12 vs 5 source_type
```

Evidence ids: `EV-AUDIT-H-001` (reference scan, PASS), `-002` (determinism, PASS), `-003` (parameter identity, bounded, PASS), `-004` (contract divergence, PASS), `-005` (corpus digest, PASS), `-006` (corpus mutated during audit, PASS-with-caveat). Each carries `expected`, `actual`, `limitations`, `exit_code`, `duration_ms`.

Headline numbers: 5 701 references seen · **295** severity-1 unresolved · 85 severity-2 · 1 402 informational · 40 backup snapshots reported separately and excluded from the headline.

## 4. MC-AUD-017 — result

121 clauses classified, each with a legacy anchor (`03-m0x.md §/L`) and a current-destination anchor (`00`–`15` §/L):

| INHERITED | INHERITED_WITH_REVISION | SUPERSEDED | REJECTED | DEFERRED | UNRESOLVED |
|---:|---:|---:|---:|---:|---:|
| 45 | 52 | 9 | 4 | 7 | 4 |

**The four UNRESOLVED rows are the debt the register should carry** — read-consistency/snapshot (M01 D36), physical directory layout (D10), first-run initialisation (D32), permission-record persistence (M05 §13.1 / B-058). None was closed by assumption: each names the closing condition (which document, which section would have to say what).

No row was padded. Every `INHERITED*` row points at a destination section that exists; where the design package *declares* inheritance (`03:59`, `04:49`, `05:49`, `01:94`), the declaration is cited and then verified against the destination text — four declared items failed that check and are recorded as `INHERITED_WITH_REVISION`/`UNRESOLVED` with the reason.

## 5. Cross-document consistency — result

Ten findings, summarised in the matrix §1. The four that change what an implementer would do:

1. **SC-02 — RRF channel drift.** `structured` is excluded from RRF by the decided rule (`11:617`, `06:558`, V1 baseline, OI-022) while `02:344/346` still defines `w_structured=1.0` and `00:638` still draws `STR → RRF`. `02:4` contradicts its own §7.4 inside one file.
2. **SC-04 — v1/v2 enum mixing.** V2 abolishes `content_kind` and the seven-value `type` (`00:76–77`), while `01` §7.3/§7.6.2 makes `content_kind` **mandatory with eight enums**, `01` §7.4 builds a 7×8 matrix on it, `02:630` requires it, and `memory-object.schema.json` ships it.
3. **SC-05 — contract layer split.** `memory-object.schema.json` (v1) and `memory-unit.schema.json` (V2) both ship in `contracts/schemas/` with no deprecation marker; `evidence.schema.json` has 12 `source_type` values and no `original_text` (parent's OI-R-04 already proved it rejects a compliant instance; the fix is still unmerged).
4. **SC-01 — parameter identity.** Registry declares `recall.*` the only legal ID and orders full replacement (`15:138`); 53 `retrieval.*` occurrences remain, 36 distinct, 20 with no mapping at all, and `02` — the document that defines k and α…ζ — uses **zero** canonical IDs. Consistent with the parent's executed MC-AUD-006 (68/355/75/55), different scope, no contradiction.

## 6. B-4 — the two-document-set problem (needs an architecture decision)

Findings: the two sets are **mutually unreferencing** (`unified-set`/`R4.1-UNIFIED` appears 0× in the design package; `DD-AUTHORITY`/`contracts/` appears 0× in the unified set) and **architecturally incompatible in seven places** (object model, retrieval, ingestion, lifecycle axes, read/write surface, emotion scope, status of legacy M01–M05). `contracts/` exists only in `Memory-Core-R4.1-Design`, and the unified set never mentions a schema file.

I did not pick a winner. The matrix §8.5 gives four options with consequences. **Recommendation offered, flagged for the parent:** the design package as baseline including `contracts/`, unified set demoted to a review annex, **plus** a bounded import list from the unified set (B-058 permission tables, N-14 L3/contract mutual exclusion, the red-team items, the 47 A/B revisions) — because only the design package owns machine contracts and a ratified decision ledger, and the unified set's self-description ("candidate", "不构成实现授权") is the weaker authority claim.

**Three questions for the parent:**
1. P1 / P2 / P3 / P4 — and if P1, whether V2 objects or the V1 body wins *inside* the design package (SC-03/04/05 cannot stay open after a precedence choice).
2. Who owns the import list, and does importing B-058/N-14 block implementation before code starts?
3. Is `contracts/` frozen, revised or regenerated once the object model is settled — today two object schemas ship, so a schema-driven agent picks one at random.

## 7. Coordination gaps (reported, not worked around)

| Gap | Consequence |
|---|---|
| `02-shared/TASK-ASSIGNMENT.md` does not exist although `00-PARENT-MANDATE.md` §7 references it | the deliverable list had to be derived from the brief + mandate; if the file defines extra deliverables, this task may be short of them |
| Agent E's `PARAMETER-IDENTITY-REPORT` was never present in `05-audit/`, `06-evidence/`, `07-reports/` | SC-01 is a bounded cross-check against the parent's MC-AUD-006 numbers, not a diff against E's report. If E's report appears later, diff `tools/param-id-count.cjs` output against it |
| `design-package/reviews/CROSS-DOCUMENT-CONSISTENCY-REPORT.md` claims 29 PASS/FIXED items, yet still cites `contracts/api/permission-api.md` (2×) which does not exist, and scopes itself to one set | a prior PASS verdict on cross-document consistency should not be relied on; SC-01…SC-10 supersede it in scope |

## 8. What I did not do

- Did not re-execute SQLite/LanceDB/FTS/permission verification — the parent's register already carries those results and I cite instead of duplicating.
- Did not score semantic correctness of individual algorithms (that is the algorithm reviewers' scope); only inheritance and cross-document agreement.
- Did not verify the 40 backup snapshots in detail — they are scanned, counted (317 unresolved) and excluded, because a stale copy's broken link is not a new defect.
- Did not fix anything in the corpus. Every defect listed here is a read-only observation; fixes need the owner's authorization.

## 9. Reproduction

```
cd E:\[PROJECT]
node .\r41-baseline\05-audit\scan-references.cjs                 # summary, exit 0
node .\r41-baseline\05-audit\scan-references.cjs --json out.json # machine-readable
node .\r41-baseline\05-audit\tools\param-id-count.cjs
node .\r41-baseline\05-audit\tools\schema-probe.cjs
```
