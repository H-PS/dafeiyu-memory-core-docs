# SECOND-ROUND-REMAINING-TEST-GATES — round 4.1

Date: 2026-11-05 (Asia/Shanghai)

Everything here is **NOT_RUN**. It is listed so that absence of evidence is visible rather than
mistaken for success. Nothing in this file may be counted as a pass.

---

## 1. Unexecuted test families

| # | Family | Why it was not run | What would be needed | Bounded by authorization? |
|---|---|---|---|---|
| G-01 | **API contract behaviour** | no mock server existed | 已派新 Agent：`03-verification/api/` mock harness | no |
| G-02 | **Business fixtures for 30 schemas** | never written | 已派新 Agent：`03-verification/schema/fixtures/` | no |
| G-03 | **LLM task harness** | no mock provider; no LLM model may be called | a deterministic mock provider producing malformed / timeout / duplicate / low-confidence output | partly (no real model) |
| G-04 | **Token budget** | no real tokenizer for the target model | a pinned tokenizer, then measure; today every threshold is `VALIDATION_PENDING` | no |
| G-05 | **End-to-end retrieval** | the two channels have unresolved acceptance failures | fix FTS Chinese + decide vector index policy first | no |
| G-06 | **Real host identity mapping** | **barred by authorization** | a real DSH/QQ Bridge adapter **and** explicit user permission | **yes — forbidden this round** |
| G-07 | **ANN recall at acceptance scale** | measured only at one scale (0.30) | a fixed acceptance corpus and a threshold decision | no |
| G-08 | **Power-loss durability** | SIGKILL is not power loss | fault-injection hardware or `fullfsync` study | no |
| G-09 | **Driver comparison** | no package manager | npm/pnpm + native toolchain, then re-run the same harness | no |
| G-10 | **Backup failure injection** | not attempted | disk-full, corrupt-backup, mid-rebuild-failure scenarios | no |
| G-11 | **Concurrent multi-process FTS/LanceDB** | single-process only in these harnesses | multi-process soak after G-05 | no |
| G-12 | **Migration of a real product DDL** | the product DDL does not exist | M01 implementation (requires authorization) | **yes — forbidden** |
| G-13 | **Emotion runtime** | `emotion.*` default-off; enablement needs a decision | a user decision, then a separate suite | **yes — forbidden** |
| G-14 | **Context-compression plane (FUT-09)** | not adopted; would add a module the boundary matrix lacks | an architecture decision first | **yes — needs a decision** |

## 2. Per-area unexecuted items

### SQLite
- network-filesystem WAL behaviour (documented as unsafe there)
- savepoints / nested transactions (driver exposes no helper)
- extension loading (disabled at build time in this Node — verified)
- encryption at rest
- sustained-load soak (multi-hour)

### LanceDB
- any index type other than the IVF family
- multi-process concurrent writers on one dataset
- compaction / version-GC under long edit history
- behaviour with a real embedding model (all vectors here are synthetic)
- exhaustive exactness at millions of rows

### FTS
- tokenizers beyond the four FTS5 ships (`trigram case_sensitive`, `unicode61 categories`, richer `tokenchars`)
- ICU (unavailable)
- concurrent writers to an FTS table
- index rebuild after delete-propagation under load

### Identity
- anything involving a real host
- asynchronous revocation propagation (no such mechanism exists)
- cache invalidation across processes

### Retrieval
- embedding migration cutover/drain/rollback (every migration row is a contract, not an execution; `retired` is `NOT_VERIFIED`)
- real M05 integration
- real tokenizer participation in the budget path

### Outbox
- real LanceDB as the external store (a second SQLite file was used as the stand-in — this is stated in its report)
- production duplicate detector
- OS-level `flock` (a pid-file lock has a documented same-millisecond race)
- degraded health state
- a restore marker so readiness can know a restore is pending
- `stale_event_age_ms` alarm

## 3. The gates that block contract freeze

Reproduced from `CONTRACT-FREEZE-REGISTER.md §3`, ordered by how much they block:

| Rank | Gate | Blocks |
|---|---|---|
| 1 | Chinese FTS passes PC-1 (currently 4/19 queries return nothing) | C-06, and therefore M02 |
| 2 | Vector index policy decided (scan-only vs indexed with recall 0.30) | C-05, M02 |
| 3 | Evidence schema v2 merged into the canonical package | C-03, M01 |
| 4 | Outbox schema extended or lease state moved out (7 fields rejected today) | C-11, M01 |
| 5 | Tier transition table defined (design decision) | C-12, C-11, M02 |
| 6 | Two-baseline precedence decided (architecture decision) | every family's authority |
| 7 | 55 parameter names registered or rewritten | C-14 |
| 8 | API mock harness executed | C-08 |
| 9 | A real tokenizer available | C-15 |
| 10 | Retention decided | C-17 |

## 4. What would change the picture fastest

Three actions would move more gates than any others, and none requires product authorization:

1. **Adopt a Chinese tokenizer option.** The measurement is done; the choice is a decision, not
   research. It clears gate 1 and unblocks FTS.
2. **Merge the v2 Evidence schema and extend the outbox schema.** Both fixes already exist as
   concrete proposals with evidence behind them; adopting them clears gates 3 and 4.
3. **Decide the two architecture questions (E-1 precedence, E-2 tier rules).** These are user
   decisions. Until they are made, gates 5 and 6 block everything downstream regardless of how
   much more verification is done.

## 5. A caution about this list

Every row above is a **known absence**. There may be unknown ones. The verification in this round
found defects that the prior three rounds did not — a tier axis with no rules, an outbox schema
that cannot express its own lease mechanism, a LanceDB `createIndex` that ignores its arguments —
which is grounds for expecting more, not fewer, surprises. Any claim that "the remaining work is
just execution" should be treated with suspicion.
