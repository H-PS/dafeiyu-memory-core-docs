# STANDARDS — MC-CORE-CAPABILITY-REVIEW-01 (binding)

**Round**: core-capability review · 2026-11-06
**Output root**: `E:\[PROJECT]\CORE-CAPABILITY-REVIEW-2026\`
**Nature**: read-only. No design/schema/API/DDL/test edits. No V0.9 product code.

---

## 0. What this round is — and what it is NOT

This round answers **one question from the product side inward**:

> Does this architecture actually contain the complete foundation for a chat-AI memory system that can
> remember long-term, understand the user, organise experience, attach emotion, hold a stable persona,
> and keep interacting?

It is **not** another full forensic audit. The FORENSIC round already inventoried every file and opened
49 conflicts. **Do not re-audit all 49.** Do not re-count files. Do not argue about formatting, naming,
wording, or low-priority ops.

Extract only what **changes the answer to the seven capability questions**.

**Explicitly out of scope for this round**: large-scale security audit · encryption · real-host
penetration · parameter tuning · performance optimisation · naming/format cleanups ·
"not yet tested" treated as "architecturally impossible".

---

## 1. Navigation entry points (start here, go to originals only when needed)

`E:\[PROJECT]\AUDIT-FORENSIC-2026\`:
`00-FINAL-FORENSIC-AUDIT.md` · `02-DOCUMENT-AUTHORITY-MATRIX.md` · `03-ARCHITECTURE-FACTS.md` ·
`04-CONTRACT-AND-SCHEMA-AUDIT.md` · `05-LIFECYCLE-AND-STATE-MACHINE-AUDIT.md` ·
`08-HISTORICAL-DECISION-TRACE.md` · `09-CONFLICT-REGISTER.md` · `10-GAP-AND-RISK-REGISTER.md` ·
`11-DECISION-DEPENDENCY-GRAPH.md` · `12-IMPLEMENTATION-READINESS-ASSESSMENT.md`

Then the baseline itself:
`C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design\` (00–15 + contracts/ + DD-AUTHORITY-V2.md)

**Baseline status (already established — do not re-litigate)**: A = the only set with machine contracts
(31 files) but its APPROVED basis has an evidence gap; B (`E:\[PROJECT]\R4.1-UNIFIED-MEMORY-CORE`, 70 files,
0 schemas) is a competing design candidate and must not be discarded; CMB is an adjacent project, **not**
a third Memory Core baseline; R3 is historical source.

---

## 2. The mandatory three-way labelling

Every finding must be split into these three, and never blended:

| Label | Meaning | Requirement |
|---|---|---|
| **EXISTING DESIGN** | a formal design exists | cite file + section; do not paraphrase into something else |
| **DESIGN COMPLETENESS** | is that design sufficient to execute? | judge inputs, outputs, steps, conditions, module responsibility, boundaries |
| **RECOMMENDED ADDITION** | your proposal | label `NEW PROPOSAL`; **never** write it as if already approved |

A "we have L1/L2/L3" answer is **not** evidence of a memory lifecycle. A schema file is **not** evidence
of a complete emotion system. A module split **is not** evidence of module independence.

---

## 3. Every core problem must answer nine things

1. What exists today? 2. What is missing? 3. Why does it damage a **core capability**? 4. Is there a module
responsibility conflict? 5. Is an algorithm missing? 6. What do you recommend? 7. Which documents need
amendment? 8. Does it block V0.9? 9. Can it be settled inside V0.9 experiments?

A problem statement without a recommendation is rejected.

---

## 4. Priority (this round's definition — narrower than FORENSIC's)

| | Meaning |
|---|---|
| **P0** | a core capability **cannot exist** — memory cannot be created/read; no unified object contract; short-term cannot reach long-term; long-term structure cannot express relations; a key datum has no legal writer; two modules hold mutually exclusive write rules; Persona cannot obtain persona+memory context; the core retrieval algorithm cannot execute |
| **P1** | runs, but an important capability is missing or seriously distorted — no cross-block joint retrieval; event↔emotion disconnected; persona loses stable identity per turn; priority conflicts with query intent; a small module change requires touching many unrelated modules; updates easily duplicate/conflict/overwrite |
| **P2** | tunable later (parameters, efficiency, admin UI, monitoring) — **archive briefly, do not lead with it** |

Main report leads with P0/P1 only.

---

## 5. Algorithm completeness vocabulary (mandatory per algorithm)

`COMPLETE` — implementable without guessing ·
`MOSTLY_COMPLETE` — only local parameter/boundary gaps ·
`PARTIAL` — material gaps ·
`CONCEPT_ONLY` — only a concept/name ·
`CONFLICTED` — documents contradict ·
`MISSING` — no valid definition.

Per algorithm record: input · output · executing module · steps · formula · parameters · boundary
(empty/duplicate/conflict) · failure handling · determinism · LLM dependency · test criterion ·
dependency on other modules.

**Separate "the algorithm is defined" from "the parameters are experimentally validated."** An
un-validated default weight is not an optimum. Never let "not yet measured" become "not implementable".

---

## 6. Module change-impact vocabulary

`CONFIG_ONLY` · `MODULE_LOCAL_CHANGE` · `CONTRACT_CHANGE` · `SCHEMA_MIGRATION` ·
`MULTI_MODULE_CHANGE` · `ARCHITECTURE_CHANGE`

---

## 7. Writing rules

- Write ONLY into `E:\[PROJECT]\CORE-CAPABILITY-REVIEW-2026\`:
  - your deliverable `.md` at the root (the exact filename in your brief)
  - `SUBAGENT-REPORTS\<LETTER>-REPORT.md`
  - `EVIDENCE\<letter>\*` for raw outputs / scripts
- **Never** use PowerShell `Set-Content/Out-File/Add-Content -Encoding utf8` — it emits a UTF-8 BOM and
  has already corrupted files in this project. Use the `write` tool, or Node `fs.writeFileSync`.
- No npm/pnpm exists. Node v24.21.0 only. Python 3.14.7 + jsonschema 4.26.0 is available via
  `$env:PYTHONPATH='C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design\evidence\second-round\pydeps'`.
- Do not ask the user anything. Decide, record, move on. Escalate to the Lead only if blocked.

---

## 8. Report skeleton (every agent)

```
# <LETTER> — <title>
## 0. Scope / what I read / what I did NOT examine
## 1. Headline (capability verdict in 3 lines)
## 2. Findings (each with EXISTING DESIGN / COMPLETENESS / RECOMMENDED ADDITION + P0/P1/P2 + cite)
## 3. The required matrix/table for this brief
## 4. What is missing to make this implementable (concrete)
## 5. Recommended design (NEW PROPOSAL if it is not in the corpus)
## 6. Cross-scope handoffs to other agents
## 7. Method appendix: files read, commands run
```

Sections 0, 3, 7 are mandatory.

---

## 9. One shared vocabulary — use these terms exactly

`MemoryUnit` · `Evidence` · `Event` · `EmotionContext` · `PersonProfile` · `memory_tier` (L1/L2/L3) ·
`status` (ACTIVE/SUPERSEDED/ARCHIVED/DELETED) · `validation_state` · `candidate.status` ·
`semantic_type` (6) · `semantic_tags` · `anchor` (8) · `visibility` (3) · `scope` (5) ·
`space_lifecycle` vs `migration_state` · `RRF` (fts+vector only) · `M01…M05` ·
`Ingestion Pipeline` (16 stages) · `Context Builder` / `Context Adapter` · `Psychology` · `Persona` · `Dream`.

Do not invent synonyms. If the corpus uses two names for one thing, report the duplication; do not pick one.