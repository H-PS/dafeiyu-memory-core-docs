# TASK-RESULT — retrieval area (Agent F)

| Field | Value |
|---|---|
| Area | `03-verification/retrieval` (M02 retrieval + the lifecycle/binding rules it depends on) |
| Run | `node run-all.cjs` — **75 checks: 74 PASS, 1 PARTIAL_PASS, 0 FAIL** (exit 0) |
| Environment | Node v24.21.0 · Windows x64 · no npm/pnpm · no dependencies |
| Evidence | `06-evidence/retrieval.jsonl` — 75 own records (74 PASS + 1 PARTIAL_PASS) + 2 retained audit records (`P001` PASS, `P002` FAIL, written by the parent, preserved verbatim). A raw tally of the file therefore reads 75 PASS / 1 PARTIAL_PASS / 1 FAIL; the FAIL is the parent's, not this area's |
| Contracts | `04-contracts/RETRIEVAL-CONTRACT.md`, `MEMORY-LIFECYCLE-MATRIX.md`, `EMBEDDING-MIGRATION-CONTRACT.md` |
| Product authorization | **NOT_AUTHORIZED** — nothing here is product code |

## 1. What ran

| Suite | Checks | Result | What it exercises |
|---|---:|---|---|
| `rrf-tests.cjs` | 15 | 15 PASS | RRF fusion as a pure function: formula, k, weights, tie-break, truncation, missing channel, duplicate handling, determinism, the `structured` exclusion proof |
| `dedup-tests.cjs` | 15 | 15 PASS | dedup/merge decisions over 11 literal fact pairs: exact, temporal restatement, scope variant, near-duplicate, conflicts, merge preconditions, "similarity never deletes" |
| `lifecycle-tests.cjs` | 24 | 23 PASS + **1 PARTIAL_PASS** | five orthogonal axes, 48 legal edges, 31 illegal edges, authority gates, coupling invariants, three document disagreements |
| `retrieval-pipeline-sim.cjs` | 20 | 20 PASS | the 12-stage chain end to end: filter, space isolation, permission, ranking, budget, assembly, degradation |
| `corpus.generator_deterministic` | 1 | PASS | the seeded corpus regenerates byte-identical (sha256 unchanged) |
| **Total** | **75** | **74 PASS, 1 PARTIAL_PASS, 0 FAIL** | |

Two consecutive full runs produced identical counts and identical aggregate results; the pipeline
suite additionally asserts run-to-run and input-order determinism internally.

**Why one check is not green.** `lifecycle.tier_promotion_gate_is_explicit` holds only because the
test hands the validator `ctx.tier_review_approved` — a flag that appears **zero times** in the 24
design documents. It therefore demonstrates the prototype's behaviour under an invented
precondition, not that the design permits a tier move. The parent's independent re-execution with
design-specified input only (`parent.reeval.tier_promotion_refused`) refused `L2 → L3` and still
fails; that is the finding, and it is recorded as `PARTIAL_PASS` with `OI-R-17` (no tier transition
table exists anywhere) as the open design gap. It must not be read as a green result.

## 2. Illegal lifecycle transitions: 31 tested, 31 rejected

Generated from `AXES × LEGAL_TABLE` (not hand-written), each row exercised individually, artifact
`ILLEGAL-TRANSITION-MATRIX.json`:

| Axis | Rows | Coverage |
|---|---:|---|
| `lifecycle_state` | 18 | skip-level (`candidate → active`), resurrection (`superseded → active`), terminality (`deleted → *`) |
| `status` | 5 | `DELETED → *`, `SUPERSEDED → ACTIVE`, `ARCHIVED → SUPERSEDED` |
| `validation_state` | 5 | `rejected → confirmed`, `confirmed → unverified`, `disputed → unverified` |
| `availability_state` | 1 | `suspended → available` (must pass `restricted`) |
| `memory_tier` | 2 | `L1 → L3`, `L3 → L1` (level skipping) |
| **Total** | **31** | **31/31 rejected**, 0 accepted, each with a specific code |

Legal side: **48** transitions accepted, 0 rejected. Under the `sm-2` profile the illegal set is
**30**, because SM-2 legalises `rejected → unverified` — that difference is a recorded test result,
not a tuning choice.

## 3. Retrieval invariants proved

1. **`structured` does not participate in fusion.** Constructed so it would win if wrongly fused:
   `d_struct` (fts 3, vector 3, **structured 1**) would score 0.048139 and rank first in a fused
   reading, versus `d_beat` (fts 1, vector 1) at 0.032787. The implementation ranks `d_beat` first;
   the same run computes the wrong order from a local reference implementation. Passing `structured`
   as a channel or weight **throws** `STRUCTURED_CHANNEL_NOT_FUSABLE`.
2. **Space isolation.** A query embedded in space A never compares against space B vectors — a
   document with *identical text* in the other space is never returned by the vector channel, and a
   cross-space comparison throws `SPACE_MISMATCH` instead of returning a number. Identity is the eight
   B-010 fields: flipping only `normalized` at identical dimensions changes the derived id.
3. **Lifecycle before ranking, permission before ranking.** Deleted / non-active / suspended memories
   are removed by the authoritative check even when permission would allow them; a non-restrictive
   `type_filter` leaves every fused score bit-identical, proving the filter scores nothing.
4. **Fail-closed permission.** Default provider denies everything; a provider that throws produces
   `partial` + exclusion, never inclusion. One channel down ⇒ `partial` naming the channel; both down
   ⇒ `failed` with an empty set and no filler.
5. **No invented token numbers.** The budget stage reports `injected_content_tokens = null`,
   `token_accounting = UNAVAILABLE`, `estimated = true`, `threshold_state = VALIDATION_PENDING`, and
   the 1200 figure is labelled a document default (EXP-DEFAULT), not a measurement.
6. **Degradation is explicit.** A degraded answer carries `degraded = true`, the reasons, and the
   `embedding_space_id` actually used — the B-011 ban on passing off old-model results as new.
7. **Determinism.** Both pure functions and the whole chain give identical output on repeat, and the
   pipeline result does not change when the corpus order is reversed.

## 4. Genuine document disagreements found

Recorded with **both readings**; none was silently resolved. Full text: `RETRIEVAL-CONTRACT.md` §8,
`MEMORY-LIFECYCLE-MATRIX.md` §7, `EMBEDDING-MIGRATION-CONTRACT.md` §9.

| ID | Disagreement |
|---|---|
| OI-R-01 | `pending_review → deleted` exists in SM-1 #3, has no row in 01 §7.7.2 |
| OI-R-02 | `archived → active` restore is called reversible by 01 but has no row there; only SM-1 #9 has the edge |
| OI-R-03 | `rejected → unverified`: 01 §7.7.1 says `rejected` is terminal; SM-2 calls the same edge "the only revival path" |
| OI-R-04 | cross-scope merge: 02 §5.3 ② "take the stricter scope" vs 02 §7.9 §10 "MUST reject" |
| OI-R-05 | `suspended → available`: 01 §7.7.3 forbids one step; SM-3 allows it |
| OI-R-06 | SM-1's lifecycle table carries `validation_state` edges (`active ↔ disputed`); 01 §7.7 forbids that |
| OI-R-07 | `draining` exists in the schema and 07 §8.2, is absent from 01 table 11, and its semantics are defined **nowhere** |
| OI-R-08 | does ALG-04's N feature (γ on `type_filter`) violate "structured only filters"? |
| OI-R-09 | budget truncation: skip-and-continue vs stop-at-first-oversize — unstated |
| OI-R-10 | 02 §7.4's own worked example gives two documents FTS rank 1, contradicting its precondition 4 |
| OI-R-11 | 02 §7.4 still lists `structured` as an RRF channel with `w=1.0` although OI-022/SRD-002 decided otherwise; the contract enum says `fts, vector` only |
| OI-R-12 | request schema field names (`current_message`, `retrieval_options.*`, no mandatory `subject_id`) vs 02 §7.4's input table (`query`, `budget_tokens`, mandatory `subject_id`) |
| OI-R-13 | the space lifecycle field is `space_lifecycle` in the schema but `status` in 07 §8.2 and 01 table 11 |
| OI-R-14 | "degradation must be explicit" is required by 07 §7.6.2 but no response field is defined for it |
| OI-R-15 | may an `active` space accept writes during `cutover`/`rollback`? |
| OI-R-16 | `migration_state` enum: schema's nine (`planned, cutover_ready, cutover, rollback, completed`) vs 07 §7.4.2/SM-8's nine (`switching, monitoring, retired, rolled_back, failed`) — overlap of only four |
| OI-R-17 | no document has a **tier transition table**; the prototype's `L2 → L3` gate on `tier_review_approved` is a prototype invention |

## 5. What must be decided

1. **OI-R-03** (`rejected` terminal or revivable) and **OI-R-01/02/05** — these four change the legal
   edge set, so the illegal matrix (31) and every "cannot happen" claim depend on them.
2. **OI-R-06** — is SM-1 or 01 §7.7 the lifecycle authority? One is a superset that mixes axes.
3. **OI-R-11** — remove the `structured` weight line from 02 §7.4 (already adjudicated, not applied).
   Until then the design package contradicts its own contract, and a reviewer reading only 02 will
   implement a three-channel fusion.
4. **OI-R-07 / OI-R-13 / OI-R-16** — `draining` semantics, the field name, and which nine-state enum
   governs migration. Migration code cannot be written coherently before these are settled.
5. **OI-R-04 / OI-R-08 / OI-R-09** — merge policy, whether a type filter may reorder results, and the
   budget truncation rule.
6. **Token budget thresholds** — every figure here is `VALIDATION_PENDING`; a real tokenizer and a
   measured context-window baseline are prerequisites, not something an agent may pick.

## 6. What was not run (and must not be read into these results)

- No FTS5, no LanceDB, no SQLite: FTS is a term-count stand-in, vectors are exact cosine over
  synthetic 8-dim vectors, and no latency/throughput figure exists anywhere in this area.
- No real tokenizer ⇒ no token count is reported.
- No real M05 ⇒ permission is an injectable mock; the deny-all default is a policy choice, not a
  measured service behaviour.
- **No embedding migration was executed**: no backfill, dual read, cutover, rollback or observation
  window. Every migration row is a contract, and `INV-E-08` (`retired` serves nothing) is recorded as
  `NOT_VERIFIED` because no retired space exists in the corpus.
- The `lifecycle-state-machine.cjs` is an **isolated prototype** with a stated mock boundary: no
  persistence, no transactions, no outbox, no audit ledger, no version increments, no tombstone
  replay.

## 7. Precondition gaps observed

1. `02-shared/TASK-ASSIGNMENT.md`, which `00-PARENT-MANDATE.md` names as the task inventory, **does
   not exist**. The task was therefore split from the brief's own deliverable list; the parent should
   confirm that mapping.
2. `design-package/02-核心算法规范.md` is reported as binary by the text-file reader; it had to be
   read through `Get-Content -Encoding UTF8`. Any agent following the mandate's "read the documents"
   step will hit this.
3. Piped stdio for a spawned program is blocked in this sandbox (`EPERM`), so `run-all.cjs` runs the
   suites in-process rather than as child processes.
4. Two audit records (`EV-RETRIEVAL-P001`, `P002`) written by another party into this area's ledger
   are **preserved verbatim** by `run-all.cjs`; its ledger rewrite keeps any record whose `task_id` is
   not `TECH-RETRIEVAL-01`. `P002` reported that the tier axis had no permitted transition. It is
   **not** reconciled away: the prototype move only succeeds behind `ctx.tier_review_approved`, which
   no design document defines, so the check is `PARTIAL_PASS` and the gap is open as **OI-R-17**.
5. The parent disclosed that the `read-response.schema.json` mtime I flagged was its own BOM strip,
   measured as exactly one file differing from canonical by the BOM only, and recorded in
   `05-audit/PARENT-CORRECTIONS.md`. My area wrote nothing under `01-baseline-corpus/`.
