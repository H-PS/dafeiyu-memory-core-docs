# Agent B · Subagent Report

> Round: MC-CORE-CAPABILITY-REVIEW-01
> Deliverable: `E:\[PROJECT]\CORE-CAPABILITY-REVIEW-2026\02-LONG-TERM-MEMORY-STRUCTURE.md`
> Question: **is long-term memory internally organised, or one flat list?**
> Status: **COMPLETE**

## STATUS: COMPLETE

(Created per STEP 0 as IN PROGRESS **before any file was read**; flipped to COMPLETE after both deliverables were finalised.)

---

## Work Log

| Step | Action | Result |
|---|---|---|
| 0 | Created `02-LONG-TERM-MEMORY-STRUCTURE.md` and `SUBAGENT-REPORTS\B-REPORT.md` with heading + initial in-progress status **before reading anything** | Done — partial-progress survival requirement met |
| 1 | Read `_shared\STANDARDS.md` fully (149 lines) | Binding labels / P0–P2 / algorithm vocabulary / writing rules internalised |
| 2 | Enumerated baseline design files + `contracts\schemas\` tree; size/line census | Located 01, 02, 11, 12, 13, DD-AUTHORITY-V2 and the 5 required schemas |
| 3 | Read `DD-AUTHORITY-V2.md` (full) + `relation` / `memory-unit` / `anchor-set` / `person-profile` / `event` schemas (full) | Core object model, 8 anchors, 8 relation types, 11 profile sections, 6 semantic_type |
| 4 | Read baseline 01 (§7.1/§7.10/§7.16/§9.1), 02 (ALG-02/03/04/06/08), 11 (§7.1–§7.19), 12 (§7.1–§7.8), 13 (§7.1–§7.7), 14 (§7.2.6), 00 (§0.1/§7.5/§7.6), 15 (greps) | Full picture of classification / relation / tier / priority |
| 5 | Encoding scan found `02` has BOM + 1 NUL (byte 673) → read tool refuses as binary. Wrote a NUL-stripped/BOM-stripped working copy to `EVIDENCE\B\02-clean.md` (permitted by STANDARDS §7). **Baseline untouched.** | Readable source for 02 citations |
| 6 | Read forensic 03/04/05/09/00 selectively (§H/§I/§J, §5.3, §D-F003, §FC-040, §9.3, D-F015) — did **not** re-audit 49 conflicts | Extracted only capability-changing facts |
| 7 | Wrote §§0–3 of the deliverable (incl. mandatory matrix, Tree/Graph/Hybrid comparison, direct verdict, four-axis separation, 9 hard cases, anchor/cross-block/priority rules, 2 Mermaid diagrams) | STATUS kept IN PROGRESS |
| 8 | Wrote §§4–7 (concrete missing items, NEW PROPOSAL recommended design + 5-line rationale, handoffs, method appendix) and flipped STATUS → COMPLETE | Done |
| 9 | Wrote this B-REPORT | Done |

**Encoding / write discipline**: only `write` and `edit` tools for deliverables; one `[System.IO.File]::WriteAllBytes` for the EVIDENCE working copy. **No** `Set-Content`/`Out-File`/`Add-Content -Encoding utf8` anywhere (BOM hazard per STANDARDS §7). All writes confined to `E:\[PROJECT]\CORE-CAPABILITY-REVIEW-2026\`. No npm. No user questions. No `FC-`/`GAP-` ids allocated (parent-only).

---

## Headline answer (3 lines)

1. **Not a flat list, not a tree.** `00 §0.1` explicitly mandates "多维组织，不采用单一分类树" with nine independent dimensions; realised as orthogonal `semantic_type` (6) + `semantic_tags` (12) + `anchor` (8) + `subject_id` + `memory_tier` + `status` + `visibility/scope` + time + emotion.
2. **It is a faceted pool + a shallow typed memory-to-memory relation graph, not joined blocks.** `Relation` endpoints are only `memory_id` (`relation.schema.json`); cross-object links are untyped denormalised ID arrays; there is no `Project` entity and no 情境记忆 object → several brief-required relations cannot be expressed as relations (**P0, B-C001**).
3. **Retrieval priority is its own intent-routed axis** (`11 §7.17`, `anchor-set.schema.json`), explicitly NOT `tier`/`importance`/`last_retrieved`/global weights. But **nothing automatically co-retrieves related blocks** (`memory.search` returns a flat ranked set; `getRelated` is manual 1-hop, no depth/budget/visited spec; 图检索 REJECTED in `02 §3`) → a chat LLM can recall one block and miss related info (**P1, B-C002**).

**Verdict on Tree / Graph / Hybrid**: **HYBRID** — orthogonal faceted classification + separate tiers + separate intent-driven per-group priority + a bounded 1-hop typed relation graph. Grounded in: ranked Top-K under a 1200-token budget (`read-request.schema.json`), person-priority ordered list rather than a path (`11 §7.14`), `MUST NOT 以 tier 决定读取优先级` (`11 §7.12`), `MUST NOT 用单一固定权重全局排序` (`11 §7.17`), cross-block joins needing typed edges (`01 §7.10`), and `02 §3` REJECTED 图检索 / `01 §7.10` 无环约束 ruling out deep traversal. See deliverable §3.2–§3.3.

---

## Files Read

**Standards**: `E:\[PROJECT]\CORE-CAPABILITY-REVIEW-2026\_shared\STANDARDS.md` (full).

**Baseline** `C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design\`:
`00-总体架构与设计原则.md`, `01-记忆本体与数据模型.md`, `02-核心算法规范.md` (via `EVIDENCE\B\02-clean.md`), `11-MemoryAnchor与语义分类.md`, `12-人物画像整合.md`, `13-Evidence与EmotionContext.md`, `14-ChatLLM读写API.md`, `15-参数注册表.md`, `DD-AUTHORITY-V2.md`, `OPEN-ISSUES.md`, `contracts\schemas\{relation, memory-unit, anchor-set, person-profile, event, read-request}.schema.json`, `contracts\api\api-contracts.md`.

**Forensic** `E:\[PROJECT]\AUDIT-FORENSIC-2026\`:
`00-FINAL-FORENSIC-AUDIT.md`, `03-ARCHITECTURE-FACTS.md`, `04-CONTRACT-AND-SCHEMA-AUDIT.md`, `05-LIFECYCLE-AND-STATE-MACHINE-AUDIT.md`, `09-CONFLICT-REGISTER.md`, `14-SUBAGENT-REPORTS\D-REPORT.md`.

**Not examined**: forensic 06/07/08/10/11/12/13, `16-RAW\` (except one D-REPORT line), baseline `diagrams/`, `evidence/`, `reviews/`, `test-specs/`, and the full 49-conflict register — per STANDARDS §0.

---

## Findings Summary (B-Cnnn)

| ID | Priority | One-line | Key cite |
|---|---|---|---|
| **B-C001** | **P0** | Cross-block relation structure is not first-class — `Relation` endpoints are `memory_id` only; cross-object links are untyped ID arrays; no `Project` | `relation.schema.json` L11–12; `DD-AUTHORITY-V2 §2.15/§3.1/§3.3`; `01 §7.10/§9.1`; `event.schema.json` §$comment |
| B-C002 | P1 | No automatic cross-block joint retrieval; `getRelated` manual 1-hop with no depth/budget/visited/param; 图检索 REJECTED | `DD-AUTHORITY-V2 §8`; `11 §7.17`; `14 §7.2.6`; `15 §7.17`; `02 §3` L68; `FC-040` |
| B-C003 | P1 | 项目 is only `scope=project_scoped` (with `scope_id` missing, OI-020); 情境记忆 has no object (Session Context unmapped per D-F003) | `01 §7.5/§7.16`; `DD-AUTHORITY-V2 §2.10/§4.3`; `00 §7.5`; `05 §D-F003`; `OPEN-ISSUES OI-020` |
| B-C004 | P1 | `anchor-set.schema.json` global `weights` (PERSON_PROFILE 0.35 > SELF_EXPLICIT 0.2) contradicts ordered `anchor_sequence` and the per-group weighting rule | `anchor-set.schema.json` L99/128–141; `11 §7.14/§7.17` |
| B-C005 | P1 | `getProfile` defaults to all 11 sections, contradicting `14 §7.4` "MUST NOT 每轮读全 11 分区"; 11-section view mixes stable + volatile | `read-request.schema.json` L188; `12 §7.2/§7.7.1`; `14 §7.4` |
| B-C006 | P1 | `PERSON` overloaded — `DD-AUTHORITY-V2 §4.1` maps `procedural`/`semantic` into `PERSON`, while `11 §7.1` forbids dumping unclassifiable content into `PERSON`; 稳定事实 homeless | `DD-AUTHORITY-V2 §2.1/§4.1`; `11 §7.1/§7.2`; `01 §7.2` |
| B-C007 | P1 | Relation-change / deletion propagation covers `Observation`/`PersonProfile`/`Relation` but **not** the denormalised ID arrays (`evidence_refs` dangling proven by D-F015) | `02 §7.9 §5.5`; `13 §7.4.5/§7.4.6`; `12 §7.4`; `00-FINAL §9.3`, `FC-035`, `D-F015` |
| B-C008 | P1 | Cycle avoidance is a constraint ("递归检查") not an algorithm; `getRelated` has no recursion bound | `01 §7.10/§7.18 C-07`; `12 §7.6`; `14 §7.2.6` input table |
| B-C009 | P1 | v1 `memory-object.schema.json` and v2 `memory-unit.schema.json` coexist, both `required`-valid; `retrieval-request.schema.json` lacks `tier/status` filters | `DD-AUTHORITY-V2 §4`; `04 §C-F017`; `05 §X-D-03/§4.3` |

**P2 (archived, not led with)**: CR-02/03/04 emotion gates, 图检索 REJECTED, unvalidated EXP-DEFAULT weights. Not treated as "architecturally impossible" (STANDARDS §0).

**Return values**: recommended structure = **HYBRID**; 5-line rationale in deliverable §5.1; **block count = 9 semantic blocks (6 existing, 3 NEW PROPOSAL) + 2 supporting layers**; P0 = 1 (`B-C001`), P1 = 8 (`B-C002…B-C009`).

---

## Cross-scope handoffs

- **Agent C**: B-C009, OI-020 (`scope_id`), forensic `05 §4.3` (`retrieval-request.schema.json` no `tier/status` filter).
- **Agent A**: B-C002 (co-retrieval), B-C004 (global weights), `02 §3` + FC-040 (config for a rejected capability).
- **Agent D**: B-C007 (D-F015 dangling refs, `00-FINAL §9.3`, FC-035 restore), B-C008 (cycle-check algorithm).
- **Lead**: map B-Cnnn into `FC-`/`GAP-` registers (I allocated none).

---

## Limitations / Blockers

- **Read-only review**: no claim that any subsystem works. `02 §3` marks M03/M04/M05/Dream FUTURE; `DD-AUTHORITY-V2 §10` keeps P3–P6 "需阶段授权". All verdicts are design-level (STANDARDS §2: a schema is not a working subsystem; a module split is not module independence).
- **Baseline `02` is BOM + NUL corrupted** (pre-existing). The read tool refused it as binary. I did **not** modify the baseline; a cleaned working copy lives at `EVIDENCE\B\02-clean.md`. `01` also has a BOM but was readable. This did not block the review.
- **Evidence gap inherited from forensic**: `DD-AUTHORITY-V2` APPROVED basis has an evidence gap (STANDARDS §1); baseline status A/B/CMB/R3 was not re-litigated.
- **No approval requested**: this session has approval prompts disabled and a fixed delegated permission scope; no operation beyond it was attempted. No sandbox escalation requested. No user questions asked (per brief).
- **Not re-audited**: 49 conflicts, formatting/naming nits, low-priority ops — all out of scope per STANDARDS §0.
