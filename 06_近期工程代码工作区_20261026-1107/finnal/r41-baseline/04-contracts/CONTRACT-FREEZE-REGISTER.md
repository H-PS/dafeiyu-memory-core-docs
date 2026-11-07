# CONTRACT-FREEZE-REGISTER — Memory Core R4.1, round 4.1

Date: 2026-11-05 (Asia/Shanghai)
Baseline: `E:\[PROJECT]\r41-baseline`

**Headline: 0 contract families are `FROZEN`. 3 are `PARTIALLY_VERIFIED`. 13 remain `CONTRACT_DEFINED` or worse.**

A contract becomes `FROZEN` only when all five of these hold:
1. a named authoritative file,
2. an explicit version,
3. dependent schemas reconciled with it,
4. defined error behaviour,
5. executed validation evidence **within a stated scope**.

Most families in this package satisfy 1–2 and part of 3. Almost none satisfy 5.

---

## 1. The register

| # | Contract family | Authority file | States | Evidence | Blocking gate |
|---|---|---|---|---|---|
| C-01 | Core architecture & module boundaries | `00-总体架构与设计原则.md`, `DD-AUTHORITY-V2.md` | `CONTRACT_DEFINED` | static only | two competing baselines (OI-R-01); no executed boundary test |
| C-02 | MemoryUnit / Event / EmotionContext / Profile schemas | `contracts/schemas/*` | `PARTIALLY_VERIFIED` | 31/31 meta-valid, 0 unresolved `$ref` | business fixtures for 30 of 31 schemas absent (MC-AUD-018) |
| C-03 | **Evidence schema** | `04-contracts/schemas/evidence.schema.json` (v2, proposed) | `PARTIALLY_VERIFIED` | compliant instance accepted; negative fixtures rejected | **not merged into the canonical package** (OI-R-04) |
| C-04 | SQLite storage / DatabaseAdapter | `contracts/api/api-contracts.md §8` | `PARTIALLY_VERIFIED` | 55 executed checks, 0 failing | driver comparison impossible (no pkg manager); no power-loss test; no soak |
| C-05 | LanceDB / VectorStoreAdapter | `02`, `embedding-space.schema.json` | `PARTIALLY_VERIFIED` | 66 records; scan-path exactness on fixed datasets | indexed-path recall 0.30 and unmeasurable; `createIndex` ignores its options |
| C-06 | FTS5 / text channel | `02`, `15-参数注册表` | `PARTIALLY_VERIFIED` | 71 records; 12 tokenizer configs measured | **PC-1 FAIL**: 4 of 19 Chinese word queries return nothing; tokenizer undecided |
| C-07 | RRF / fusion | `02 §7.4`, `retrieval-response.schema.json` | `PARTIALLY_VERIFIED` | fusion determinism + `structured` exclusion tested | documented worked example is itself invalid (OI-R-03) |
| C-08 | API envelopes & request context | `contracts/api/api-contracts.md` | `CONTRACT_DEFINED` | document-level only | 4 defects (scope collision, `revoke` TBD, `partial` rule, error-code enforcement); **no API harness executed** |
| C-09 | Error codes | `contracts/error-codes.md` | `CONTRACT_DEFINED` | 16 codes with 7 attributes each | error schema cannot enforce canonical codes |
| C-10 | Permission / M05 | `04-权限与治理规范.md`, `permission-context.schema.json` | `PARTIALLY_VERIFIED` | synthetic matrix executed, fail-closed holds under hardened profile | real host identity never touched (by design); M05 itself not implemented |
| C-11 | Outbox / state machines | `03`, `contracts/state-machines/` | `CONTRACT_DEFINED` | simulator + suite running | **SM-1 mixes the lifecycle and validation axes** (OI-R-06); no tier transition table (OI-R-07) |
| C-12 | Memory lifecycle / tier | `01 §7.7`, `state-machines.md` | `DESIGN_INCOMPLETE` | illegal-transition matrix executed on a prototype | **no tier transition rules exist anywhere** (OI-R-07) |
| C-13 | Embedding space lifecycle | `embedding-space.schema.json` | `PARTIALLY_VERIFIED` | `space_lifecycle` vs `migration_state` separation tested | no executed migration cutover/rollback test |
| C-14 | Parameter registry | `15-参数注册表.md` | `CONTRACT_DEFINED` | 68 canonical IDs harvested | **355 references, 75 unresolved across 55 names** (MC-AUD-006) |
| C-15 | Token budget | `retrieval.schema.json`, `02/06/14` | `VALIDATION_PENDING` | none | no real tokenizer available; no thresholds may be invented |
| C-16 | LLM task contract | `09`, `10`, `14` | `CONTRACT_DEFINED` | none | no LLM harness executed |
| C-17 | Migration / backup / restore | `05`, SM-6/SM-8 | `PARTIALLY_VERIFIED` | SQLite migration + backup/restore executed | retention undecided (MC-AUD-009); no product-DDL migration |
| C-18 | Startup / shutdown | `05`, `08` | `CONTRACT_DEFINED` | none | ordered matrix not yet executed |

Count: **FROZEN 0 · PARTIALLY_VERIFIED 9 · CONTRACT_DEFINED 7 · DESIGN_INCOMPLETE 1 · VALIDATION_PENDING 1**

## 2. Why nothing is frozen

Freezing requires evidence *within an explicit scope*, and this round deliberately produced exactly
that — scoped verdicts, not global ones. The honest reading:

- The **storage technology** is in good shape: SQLite's file-backed behaviour, transactions,
  locking, migration, backup/restore and crash recovery are all demonstrated here, with 0 failing
  checks. That is much stronger than the prior round, whose "crash recovery" turned out to be a clean
  `process.exit(0)`.
- The **text channel** is not in good shape: Chinese retrieval fails its own product criterion, and
  the failure is structural rather than tunable.
- The **vector channel** is usable only on the scan path, and indexed retrieval measures 0.30 recall.
- The **contract layer** still has a P0 hole (Evidence) and a P1 hole (parameter identity).

Freezing a family whose core channel fails its own criterion would be exactly the failure mode the
task book forbids.

## 3. Conditions under which each family could freeze

| # | To freeze, first |
|---|---|
| C-02 | add business fixtures for the remaining 30 schemas |
| C-03 | merge the v2 Evidence schema into the canonical package and re-run conformance |
| C-04 | obtain a package manager, run this same harness against ≥1 alternative driver; add power-loss and soak tests |
| C-05 | measure ANN recall against brute force at acceptance scale, and resolve or contractually drop `indexType`/`distanceType` |
| C-06 | adopt one Chinese tokenizer option (the unigram+bigram shadow is measured best) and pass G1–G5 in the FTS report |
| C-07 | fix the documented RRF example so the spec and the code agree |
| C-08 | resolve the 4 API defects and execute a mock contract harness |
| C-09 | split canonical vs legacy error schemas so normalisation is enforceable |
| C-10 | — (can freeze only after a real host mapping is verified, which is out of scope) |
| C-11 | fix SM-1's axis mixing; add a tier transition table |
| C-12 | define tier transitions (design decision, escalated to the user) |
| C-13 | execute a real cutover/rollback/drain test |
| C-14 | register the 55 unresolved parameter names or rewrite the references |
| C-15 | obtain a real tokenizer and measure budgets against it |
| C-16 | execute an LLM harness with malformed/timeout/retry/failure cases |
| C-17 | decide retention (user decision) and migrate a real product DDL |
| C-18 | execute an ordered startup/shutdown matrix |

## 4. What would make this register wrong

This register is a snapshot. It would become misleading if:
- an agent's report were later amended after I wrote this (I would need to re-check);
- a contract were declared frozen on the strength of the *prior* round's claims, several of which
  this round showed to be unsupported (LanceDB tolerance, LanceDB dataset, SQLite crash recovery);
- the two competing baselines were reconciled by deletion rather than by an explicit precedence rule.
