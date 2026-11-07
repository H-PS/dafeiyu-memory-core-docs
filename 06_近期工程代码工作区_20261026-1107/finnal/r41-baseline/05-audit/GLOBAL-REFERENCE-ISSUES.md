# GLOBAL REFERENCE ISSUES — real scan results

**Agent H · `05-audit/scan-references.cjs` · Memory Core R4.1 baseline**

Everything below is output of an executed run, not an estimate. Raw artefacts:

| Artefact | Path |
|---|---|
| Scanner (plain CommonJS, `node` directly) | `05-audit/scan-references.cjs` |
| Machine-readable result (canonical) | `06-evidence/raw/audit/scan-references.json` (1 167 904 bytes) |
| Run log | `06-evidence/raw/audit/scan-references.log` |
| Second run for determinism | `06-evidence/raw/audit/scan-references.run2.json` |
| Corpus digest (pins the scanned tree) | `06-evidence/raw/audit/corpus-digest.json` — 212 files, 2 859 384 bytes, `sha256 56e64c38…88cd` |
| Parameter-identity probe + log | `05-audit/tools/param-id-count.cjs` → `06-evidence/raw/audit/param-id-count.log` |
| Contract-schema probe + log | `05-audit/tools/schema-probe.cjs` → `06-evidence/raw/audit/schema-probe.log` |

Run: `node .\r41-baseline\05-audit\scan-references.cjs` → **exit 0**, 7/7 canaries resolved, 127–134 ms.
Evidence records: `EV-AUDIT-H-001` (scan), `-002` (determinism), `-005` (corpus digest) in `06-evidence/audit.jsonl`.

---

## 1. Harness trustworthiness

Exit 0 is only meaningful if the detector can resolve references that *do* exist. Seven canaries are planted at fixed addresses; all must resolve or the run exits 1 and the counts must not be used.

| Canary target | Why this one | Result |
|---|---|---|
| `design-package/01-记忆本体与数据模型.md §7.6` | deep numeric heading in the largest current document | resolved |
| `design-package/02-核心算法规范.md §7.4` | heading containing a dot in its title (`7.4 ALG-03：RRF 融合`) | resolved |
| `design-package/15-参数注册表.md §7` | top-level numeric heading | resolved |
| `governance/03-m01.md §5.1` | legacy document, numeric heading | resolved |
| `governance/03-m05.md §10.4` | legacy document, three-level heading | resolved |
| `unified-set/05-STATE-MACHINES.md §3.1` | second set, numeric heading | resolved |
| `unified-set/06-ALGORITHM-DESIGN.md §1.3` | second set, three-level heading | resolved |

`canary_failures = 0`.

**Determinism:** the two runs produced byte-identical payloads except `duration_ms` (127 vs 134); verified by re-serialising with that field removed (`observed payloads identical ignoring duration: true`). File hashes therefore differ by design, not by content drift.

**Immutability:** I wrote nothing under `01-baseline-corpus/`; the scanner opens it read-only. One caveat that must travel with any immutability claim: `design-package/contracts/schemas/read-response.schema.json` carries mtime **2026-11-05 20:11:26**, i.e. it was written *during this audit window* (every other corpus file is dated 2026-11-04). A re-scan after the change produced an identical issue set (`unresolved_total 1782`, `severity1 295`, `issues_primary` byte-identical), so no finding here depends on it — but the corpus was demonstrably not frozen while the round was running. Recorded as `EV-AUDIT-H-006`, including the fact that mtime cannot attribute the write.

---

## 2. Headline counts

Scope: 212 files indexed, **129 markdown documents** = **89 primary** (three document sets) + **40 backup snapshots**. `governance/backup/` snapshots are historical copies of the same documents; they are scanned and reported **separately** and are **excluded from every headline number below**, because counting a stale copy's broken link twice inflates the defect count without adding information.

| Metric | Primary docs |
|---|---:|
| References seen | 5 701 |
| — by class: file token / self-reference / external source / docref / module label / path | 1 764 / 1 738 / 765 / 634 / 577 / 223 |
| **Unresolved total** | **1 782** |
| — severity 1: target does not exist | **295** |
| — severity 2: target named but undefined | **85** |
| — severity 3: informational (explained in §5, mostly *not* defects) | 1 402 |
| Backup snapshots (reported, excluded) | 40 docs, 317 unresolved |

Severity 1 by document set: `governance` 172 · `unified-set` 62 · `design-package` 61. 34 of the 89 primary documents contain at least one severity-1 reference; 191 distinct targets are involved.

---

## 3. Issue codes

| Code | Severity | Meaning |
|---|---|---|
| `E_FILE_NOT_FOUND` | 1 | cited file basename exists in none of the three sets |
| `E_PATH_NOT_FOUND` | 1 | cited relative/absolute path does not exist in the workspace |
| `E_SECTION_NOT_FOUND` | 1 | target document exists, cited `§n` is not a heading in it |
| `E_UNRESOLVED_SECTION` | 2 | bare `§n` that no document in the set defines |
| `E_DECISION_NOT_FOUND` | 2 | `M0x Dnn` where `Dnn` heads no section |
| `E_AMBIGUOUS_TARGET` | 2 | same basename exists in more than one set |
| `E_AMBIGUOUS_SECTION` | 3 | bare `§n` undefined here but defined by N other document(s) — resolvable in practice |
| `E_SOURCE_ABSENT` | 3 | citation attached to a source that is not shipped (`任务书`, `台账`, `sf`, `m.md`, `B-07`, `HANDOFF`) |
| `E_PLANNED_ARTIFACT` | 3 | runtime/config/data path (`config/*.json`, `manifest/`, `state/`) that cannot exist inside a design corpus |
| `E_LEGACY_BASENAME` | 3 | pre-import basename (`m01.md`, `adjudication-ledger.md`) that aliases to a shipped file |
| `E_ENUM_ITEM` | 3 | `§13.3` resolves only as *item 3 of the numbered list under §13*, not as a heading |
| `E_CJK_NUMBERING` | 3 | target numbers chapters `一、二、…` while the citation uses `§6` |

Full per-code counts: `references_seen_by_class` + `primary.by_code` in the JSON.

| Code | Refs | Distinct targets |
|---|---:|---:|
| `E_SOURCE_ABSENT` | 893 | — |
| `E_AMBIGUOUS_SECTION` | 312 | — |
| `E_FILE_NOT_FOUND` | 198 | 105 |
| `E_PLANNED_ARTIFACT` | 83 | — |
| `E_UNRESOLVED_SECTION` | 82 | 38 |
| `E_PATH_NOT_FOUND` | 74 | 69 |
| `E_ENUM_ITEM` | 56 | — |
| `E_LEGACY_BASENAME` | 34 | — |
| `E_CJK_NUMBERING` | 24 | — |
| `E_SECTION_NOT_FOUND` | 23 | 17 |
| `E_DECISION_NOT_FOUND` | 3 | 3 |

---

## 4. Where the real defects are

### 4.1 By document (severity 1 only)

| Refs | Document |
|---:|---|
| 103 | `governance/12-A-REVIEW-PROGRESS.md` |
| 29 | `governance/04-HANDOFF-20261103-R4.1-CR08A.md` |
| 28 | `unified-set/01-MATERIAL-INDEX.md` |
| 17 | `design-package/reviews/CROSS-DOCUMENT-CONSISTENCY-REPORT.md` |
| 14 | `unified-set/audit/SUBAGENT-1-AB-CROSSCHECK.md` |
| 13 | `governance/11-ALGORITHM-REVIEW-SCOPE.md` |
| 12 | `design-package/06-测试与评估规范.md` |
| 9 | `governance/07-记忆库框架-未落地方案-关键提示词.md` |
| 8 | `unified-set/audit/SUBAGENT-4-AGENT123-VERIFY.md` |
| 7 | `design-package/DD-AUTHORITY.md` |

Two structural observations, not two isolated files:

1. **The worst offenders are meta-documents** (progress logs, handoffs, material indexes, review reports), not specifications. They cite files from the *authoring workspace* (`A-REVIEW-PROGRESS.md`, `A-01-ISSUE-REGISTER.md`, `R3-Pluggable-Capability-Design.md`, `v4.6-latest-design-candidate.md`, `PROTOTYPE-PROGRESS.md`, `dream.md`, `PSYCH-PIPELINE.md`) that were never imported into the corpus. 103 of 295 severity-1 hits come from one progress file.
2. **The specification documents are comparatively clean**: within `design-package`, severity-1 hits are 1 in `00`, 2 in `01` and `14`, 3 in `12`, 5 in `07`/`08`/`13`, and **0** in `02`, `03`, `04`, `05`, `09`, `10`, `11`, `15` and every `contracts/` file. The defects concentrate where documents *describe* other documents — `06` (12, its fixture list), `DD-AUTHORITY` (7, its authoring paths) and the prior review report (17).

### 4.2 By missing target

| Refs | Target | Set | Nature |
|---:|---|---|---|
| 16 | `A-REVIEW-PROGRESS.md` | governance | reviewer artefact never shipped |
| 9 | `A-01-ISSUE-REGISTER.md` | governance | idem |
| 6 | `social-v2.json` | design-package | config artefact named as if present |
| 5 | `R3-Pluggable-Capability-Design.md` | governance | R3 input never shipped |
| 4 | `top-记忆库项目设计宗旨与统一设计原则.md.md` | unified-set | **the unified set's declared highest design basis** (`00-README.md:6`) does not exist in the corpus |
| 4 | `v4.6-latest-design-candidate.md`, `PSYCH-PIPELINE.md`, `dream.md`, `A-01-EVENT-L3-REVIEW.md` | mixed | upstream inputs never shipped |
| 3 | `queries.json`, `relevance.json` | design-package | test fixtures named in `06` §7 but absent |
| 3 | `embedding-migration-state.md` | design-package | runtime state described as a document |
| 2 | `contracts/api/permission-api.md` | design-package | cited as a contract file; only `contracts/api/api-contracts.md` exists |
| 2 | `memory-vectors.jsonl` | design-package | data artefact |

### 4.3 The one spec-level section defect that repeats

`02 §7.10.3` is cited 5× across `12`, `13`, `14` — `02-核心算法规范.md` has no §7.10.3 (its `§7.10` is `ALG-09：M05 权限判定` with no sub-headings). Every citation of the permission-decision detail therefore lands on nothing. `02 §7.9.2`, `§7.9.5`, `§7.14.3`, `§5.4`, `§5.4.1`, `14 §4.5`, `01 §1.3` are the same class of defect (1–2 refs each).

---

## 5. What is *not* a defect (honest calibration)

1 402 of 1 782 findings are severity 3, and most of them are citation *style*, not error:

- **`E_ENUM_ITEM` (56).** I originally scored `DD-AUTHORITY-V2 §13.1…§13.7` as broken (55 refs). Inspection showed `DD-AUTHORITY-V2.md:457` is `## 13. 写作硬规则` followed by seven numbered items and no sub-headings, so `§13.3` unambiguously means item 3. I therefore added enumerated-item resolution to the scanner and re-ran: severity 1 fell from 351 to 295. This is a corrected false positive, and it is the reason the numbers here differ from any earlier round.
- **`E_CJK_NUMBERING` (24).** Legacy documents number chapters `一、二、…`; callers write `§6`. The scanner converts (`六`→6) so the citation resolves, but reports the mismatch because an automated reader that does not know the convention would fail.
- **`E_PLANNED_ARTIFACT` (83).** `config/retrieval.json`, `manifest/schema-version.json`, `state/…` are *intended* to be produced at implementation time. Reporting them is correct, treating them as corpus defects is not.
- **`E_SOURCE_ABSENT` (893).** Every citation of `任务书`, `台账`, `sf`, `m.md`, `B-07`, `HANDOFF` points outside the corpus. This is not a broken link; it is the **provenance gap** described in §6.1 and it is the single largest information loss in the delivery.
- **`E_AMBIGUOUS_SECTION` (312).** A bare `§7.4` inside a document that lacks §7.4 but where exactly one sibling defines it. Usually the author meant the sibling. Recorded because the reader cannot tell.

---

## 6. Three systematic causes

### 6.1 The canonical sources are not in the delivery (893 refs)

The design package states its own basis as the task book, the adjudication ledger (`台账`), `m.md` and `sf` — and ships none of them. `DD-AUTHORITY.md:40–41` even cites absolute authoring paths (`C:\Users\[USER]\Desktop\work\03-m01.md`). The `unified-set` likewise declares `top-记忆库项目设计宗旨与统一设计原则.md.md` its highest basis; that file is absent (4 refs). Consequence: **no reader can verify any normative statement against its stated source**, which is exactly the condition under which prose drifts unnoticed (see the RRF and `content_kind` contradictions in `CROSS-DOCUMENT-CONSISTENCY-MATRIX.md`).

### 6.2 Pre-import basenames and set-external references (34 + 74 refs)

`m01.md`…`m05.md`, `adjudication-ledger.md`, `05-conflict-register.md`, `06-interface-map.md` are cited under their authoring names; the corpus ships them as `governance/03-m0x.md`, `08-…`, `09-…`, `10-…`. The scanner resolves these through an explicit alias map and reports them as informational, so the *content* is not lost — but a human following the citation will not find the file. Absolute paths into `C:\Users\[USER]\…` and `finnal/work-交付包/design/_sources/m.md` are the same problem, one level worse.

### 6.3 Test and data artefacts are contractual but absent (≈12 refs)

`06-测试与评估规范.md` §7 names `queries.json`, `memories.json`, `relevance.json`, `permission_cases.json`, `expected_results.json`, `conflict_cases.json`. Zero are present. The test contract is therefore unexecutable as written — this is the same gap recorded as inheritance row R-038.

---

## 7. Reproduction

```
cd E:\[PROJECT]
node .\r41-baseline\05-audit\scan-references.cjs                      # summary to stdout, exit 0
node .\r41-baseline\05-audit\scan-references.cjs --json out.json      # machine-readable
node .\r41-baseline\05-audit\scan-references.cjs --quiet; echo $LASTEXITCODE
```

Options: `--corpus <dir>` (default `E:\[PROJECT]\r41-baseline\01-baseline-corpus`), `--json <path>`, `--quiet`.
Exit codes: `0` sweep completed and all canaries resolved; `1` a canary failed (counts untrustworthy); `2` harness error. **Broken references never change the exit code** — they are the expected observable, not a harness failure.

## 8. Limits

- Line attribution is per-line; a reference spanning a line break is missed. Markdown tables and fenced blocks are handled (fences are skipped because ASCII directory trees are not citations; skipped-fence counts are in the JSON).
- The scanner does not judge whether a resolvable reference is *correct* — only whether its target exists. Semantic contradictions are in `CROSS-DOCUMENT-CONSISTENCY-MATRIX.md`.
- Cross-set resolution is intentionally not automatic: `01 §7.10` in `unified-set` and `01 §7.10` in `design-package` are different sections of different documents, and conflating them would hide the two-document-set problem rather than expose it.
