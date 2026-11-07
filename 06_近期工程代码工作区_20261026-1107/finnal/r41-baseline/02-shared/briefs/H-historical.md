# BRIEF — Agent H · Historical inheritance and global cross-document consistency

Read first:
1. `02-shared\00-PARENT-MANDATE.md`
2. `02-shared\EVIDENCE-FORMAT.md`
3. `01-baseline-corpus\governance\` (the legacy M01–M05 set: `03-m01.md` … `03-m05.md`, `08-adjudication-ledger.md`,
   `09-05-conflict-register.md`, `10-06-interface-map.md`, `13-M02检索算法实现规约.md`,
   `14-MemoryCore-R4.1算法实施规格.md`, `15-MemoryCore-R4.1目标架构与实施基线.md`)
4. `01-baseline-corpus\design-package\` (the current 00–15 + contracts + governance files)
5. `01-baseline-corpus\unified-set\` (the second R4.1 document set — 00–16 + audit)

Own: `05-audit\`. Evidence: `06-evidence\audit.jsonl`; raw: `06-evidence\raw\audit\`.

## The inheritance matrix

Build a clause-level inheritance matrix for the legacy M01–M05 material against the current design
package, classifying every substantive clause as one of:
`INHERITED` · `INHERITED_WITH_REVISION` · `SUPERSEDED` · `REJECTED` · `DEFERRED` · `UNRESOLVED`.

Every row needs: legacy source file + anchor, current destination (file + section) or "none",
classification, one-line justification, and what would close an `UNRESOLVED` row. Cite real anchors.
This is MC-AUD-017, open since round 2 and never completed.

## The two-document-set problem (B-4)

`Memory-Core-R4.1-Design` (00–15 + `contracts/`) and `R4.1-UNIFIED-MEMORY-CORE` (00–16 + `audit/`)
are two competing R4.1 baselines. There is no recorded precedence rule. Determine, from the documents
themselves, what each set claims to be authoritative for, and identify:

- clauses where they genuinely contradict (not just differ in wording),
- clauses present in one and absent in the other that matter for implementation,
- which set the `contracts/` folder belongs to.

Do **not** declare a winner by preference. Produce an evidence-backed precedence recommendation and
flag it for the parent as an architecture-level decision, with options and consequences.

## Consistency sweep

`CROSS-DOCUMENT-CONSISTENCY-MATRIX.md` covering at least:

- parameter identity (registry canonical ID vs schema path used as ID) — coordinate with Agent E's
  PARAMETER-IDENTITY-REPORT; do not duplicate their scan, cross-check their result instead
- RRF channel drift (`fts`+`vector` only; `structured` = filter/router) anywhere in text or tests
- status-axis conflation (candidate status vs memory status vs validation_state vs lifecycle tier)
- v1/v2 enum mixing (the v1 seven `type` values and six `source_type` values vs the V2 six/five)
- broken references: every `file §section` citation that does not resolve; every referenced path that
  does not exist
- duplicate/near-duplicate authoritative files claiming the same subject

Write a machine-checkable scan (`05-audit\scan-references.cjs`) that walks the corpus and reports
unresolvable references with file:line. It must actually run; report real counts.

## Deliverables

- `05-audit\SECOND-ROUND-HISTORICAL-INHERITANCE-MATRIX.md`
- `05-audit\CROSS-DOCUMENT-CONSISTENCY-MATRIX.md`
- `05-audit\GLOBAL-REFERENCE-ISSUES.md`
- `05-audit\scan-references.cjs` + its real output
- `05-audit\TASK-RESULT.md` with all required sections

Report honest counts: clauses classified, unresolved rows, unresolvable references found. Do not
inflate coverage with rows that carry no real anchor.
