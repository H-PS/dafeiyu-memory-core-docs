# EMBEDDING-MIGRATION-CONTRACT — Memory Core R4.1 (B-010 / B-011 / SM-8)

| Field | Value |
|---|---|
| Status | **PROPOSED** — verification contract, not an approved design decision |
| Author | Agent F (retrieval area) |
| Date | 2026-11-05 |
| Product authorization | **NOT_AUTHORIZED** — no migration runner, index writer or M01 change is authorised by this file |
| Backed by | `03-verification/retrieval/retrieval-pipeline-sim.cjs` (`deriveSpaceId`, `assertUsable`, `assertWritable`, `assertComparable`, `vectorSearch`) |
| Evidence | `06-evidence/retrieval.jsonl` — test ids `pipeline.space_*`, `pipeline.vector_*`, `pipeline.draining_*`, `pipeline.degradation_*` |
| Migration execution | **NOT RUN.** No migration has been performed, simulated end-to-end, or timed. Every §4 row is a contract, not a result |

---

## 1. Two axes that must never be merged

| Axis | Field in the machine contract | Values | Describes |
|---|---|---|---|
| Space lifecycle | `space_lifecycle` (`embedding-space.schema.json`) / `status` (07 §8.2, 01 table 11) | `active`, `draining`, `retired` (01: only `active`/`retired`) | whether this space may serve reads and accept writes |
| Migration progress | `migration_state` (`embedding-space.schema.json`) / `MigrationRecord.status` (07 §8.1) | see §7 — **three incompatible sets** | where the B-011 migration of a pair of spaces stands |

[DOC] `embedding-space.schema.json` states the separation explicitly: `space_lifecycle` is
"空间自身生命周期；迁移流程状态独立记录"; `migration_state` is "B-011 迁移状态；状态转换以 SM-8 为准".
[DOC] 07 §8.2 repeats it: "与迁移九态区分".

**Why it matters.** A space can be `active` while its migration is `dual_running` (the old space still
serves production), and `retired` only after `monitoring` passes. If one enum did both jobs, "the new
space is being backfilled" and "the old space is dead" would contend for the same field, and the
retrieval path could not tell whether it may still answer a query from the old space.

---

## 2. Identity: eight fields, not the dimension

**[DOC]** 07 §7.5 B-010, `embedding-space.schema.json` `required`.

| # | Field | Rule (verbatim constraints) | If missing / wrong |
|---|---|---|---|
| 1 | `model_id` | 1–128, `^[A-Za-z0-9._:-]+$`, case-sensitive, no whitespace | cannot determine the semantic space → **reject write and query** |
| 2 | `model_version` | 1–64, must be a concrete version, **not** `"latest"` | version drift untraceable → reject |
| 3 | `dimensions` | 1–65536, must equal the model's real output | mismatch → reject; **MUST NOT truncate or zero-pad** |
| 4 | `distance_metric` | `cosine` / `l2` / `dot_product` (`ip` = historical alias) | ordering meaningless → reject |
| 5 | `normalized` | boolean; **`normalized=true` + `cosine` and `normalized=false` + `cosine` are different spaces** | normalisation unknown → reject |
| 6 | `chunker_version` | 1–64, controlled identifier | chunk boundaries shifted → old vectors incomparable → reject |
| 7 | `text_projection_version` | 1–64, defines which fields and how they are concatenated | projection drift → vector meaning drifts → reject |
| 8 | `vector_schema_version` | `major.minor` | row structure unparseable → reject |

**Same-dimension ban.** [DOC] 07 §7.5: "`dimensions` 相同但任一其它字段不同，**MUST NOT** 视为同一空间";
SM-8 禁令 1; B-011 禁令 1 ("混合检索即 MUST 判 FAIL"). "1024" is a reference value of one deployment,
never a constant [DOC] 07 §7.5 / 裁决 R9.

**Derived identity (PROPOSED, from 07 §7.5).**

```
embedding_space_id = "space-" + sha256(canonical_json(eight fields))[0:16]
```

with field names sorted, booleans/integers in fixed form, and **no manual assignment**. This contract
implements that rule in both the corpus generator and the pipeline, so the datasets cannot contradict
it.

**[TEST]** `pipeline.space_identity_is_eight_fields`:
- flipping **only** `normalized` at identical `dimensions` produces a different id
  (`space-1d29e836dec64e95` vs `space-3f57100b18debe1e`);
- re-deriving with the same fields is stable;
- the corpus's own `embedding_space_id` values equal the derivation (no hand-written ids).
- `pipeline.space_identity_is_eight_fields` also asserts the pattern `^space-[0-9a-f]{16}$` from
  07 §8.2 implicitly by using that format.

**Three read points.** [DOC] 07 §7.5: the same `embedding_space_id` must be readable in index
metadata, the Memory record, and the retrieval response. **[TEST]**
`pipeline.no_item_has_foreign_space_vector_evidence` reads it from the response per item, and
`pipeline.draining_space_readable_and_reported` reads it from `embedding_space{}`.

---

## 3. Lifecycle rules for retrieval and writing

| `space_lifecycle` | Serve reads | Accept new vectors | Behaviour |
|---|---|---|---|
| `active` | yes | yes | normal |
| `draining` | **yes** (this contract) | **no** — `SPACE_NOT_WRITABLE` | reads continue so history stays answerable while the new space is backfilled; new writes go to the new space |
| `retired` | **no** — `SPACE_RETIRED` | no | only reachable after `monitoring → retired` with approval and a backup |

**[TEST]** `pipeline.draining_space_not_writable` (`draining` → `SPACE_NOT_WRITABLE`, `active` → `OK`)
and `pipeline.draining_space_readable_and_reported` (a query in a draining space answers, and the
response reports `space_lifecycle = "draining"` and the real `embedding_space_id`).
**[TEST]** `pipeline.incomplete_space_unusable`: deleting one of the eight fields yields
`SPACE_INCOMPLETE` naming the field — the space is never "repaired" by guessing a default.

**`draining` has no defined semantics anywhere** — it appears in `embedding-space.schema.json` and
07 §8.2, is absent from 01 table 11's `CHECK status IN (active, retired)`, and no document says what
it permits. The read-only reading above is **OI-R-07** and needs an owner, not a preference.

### 3.1 Additional refusal rules

- A space whose `migration_state ∈ {cutover, rollback}` refuses writes even if `active`: a cutover in
  flight means the routing decision is not final. [PROPOSED] — see OI-R-15.
- A query must be embedded **once**, in the space it will be compared against; the response reports
  that space. Embedding twice (once per candidate space) is how "silent mixing" starts.

---

## 4. Migration state machine (SM-8 + 07 §7.4.3 reconciled)

**This section is NOT RUN.** No migration was executed. The rows state obligations only.

| # | Transition | Trigger | Actor | Transaction requirement | Failure handling | Reversible |
|---|---|---|---|---|---|---|
| 1 | `not_started → backfilling` | authorised + eight fields complete + backup verified | migration runner | write the migration record **before** starting; no cross-DB transaction with business writes | stop, `→ failed`, delete nothing | yes |
| 2 | `backfilling → validating` | coverage = 100% (or approved partial) and the resume table is empty | migration runner | record state + coverage snapshot | — | yes |
| 3 | `validating → dual_running` | dimension/normalisation/metric verified, sampled self-retrieval consistent, **no cross-space contamination** | verifier | verification report stored on the record | — | yes |
| 4 | `dual_running → switching` | comparison within tolerance + dual-write consistency (if enabled) + explicit approval | approver + ops | routing change in **one** transaction, routing version + 1 | — | yes (config revert) |
| 5 | `switching → monitoring` | routing switched, health check green | ops | switch point recorded before switching; switch is atomic | — | yes |
| 6 | `monitoring → retired` | observation window passed + retirement approved + old space backed up | approver + ops | deletion batched and abortable; deletion counts recorded | abort → back to `monitoring` | **NO** (backup only) |
| 7 | any non-terminal `→ failed` | unrecoverable error (crash, disk, permission) | system | last known state recorded | recover per 05; authoritative data never lost | depends |
| 8 | `dual_running → rolled_back` / `monitoring → rolled_back` | comparison fails / dual-write inconsistent / quality regression / abandonment | approver | rollback record **with evidence** | stop dual write; **keep** the new space data | yes |
| 9 | `failed → rolled_back` | failure state cleaned up, old space confirmed usable | ops | rollback record | — | yes |
| 10 | `rolled_back → not_started` | re-initiation, **re-authorised** | ops | reset the record; reusing the same id **requires emptying the space first** | — | yes |

**Forbidden (MUST NOT) [DOC] 07 §7.4.3 / SM-8:**

| Forbidden | Reason |
|---|---|
| `not_started → switching`, `backfilling → switching` | switch without backfill (ban 2) |
| `validating → retired`, `dual_running → retired` | delete the old space without verification/observation (ban 3) |
| `retired → any` | terminal; only backup+rebuild |
| `any → not_started` except from `rolled_back` | loses audit continuity |
| two concurrent switches | routing version conflict, second MUST be refused |
| any cross-space vector comparison, ever | ban 1 |

These are **contract rows**, not executed checks. [`NOT_RUN`] No migration runner exists in this
area, so no illegal migration transition was exercised — the 31 illegal *memory lifecycle* rows in
`MEMORY-LIFECYCLE-MATRIX.md` are a different machine and must not be counted here.

---

## 5. Query behaviour during migration (the part retrieval must get right)

| `migration_state` | Old space | New space | What a query does | What the response must say |
|---|---|---|---|---|
| `not_started`, `planned` | serves | not queryable | normal, old space only | actual space id |
| `backfilling` | serves | **not queryable** (partial coverage) | old space only; a query routed to a backfilling space must fail rather than answer from a partial index | actual space id + `degraded` if anything is missing |
| `validating` | serves | not queryable for answers | verification queries are a separate, labelled path — never user traffic | — |
| `dual_running` | serves | queryable for comparison | dual read for the comparison harness only; user traffic stays on the routed space | both ids in the comparison record |
| `switching` | read-only | target | routing flips atomically; no query may see both | single id per response |
| `monitoring` | read-only / standby | serves | new space serves; old space kept for rollback | new id |
| `retired` | refuses (`SPACE_RETIRED`) | serves | new space only | new id |
| `rolled_back` | serves again | refuses | old space restored | old id + `degraded` while the rollback completes |
| `failed` | serves | refuses | old space continues, alarm raised | old id + explicit failure |

**The rule the whole table exists for** [DOC] 07 §7.6.2 ban 4: when the new model fails, the system
**MUST NOT** silently return old-model results dressed up as new-model results. Degradation must be
explicit: the response carries the `embedding_space_id` **actually used** plus a degradation marker
and an alarm.

**[TEST]** `pipeline.degradation_is_explicit`: a healthy run reports `degraded=false`; a run with the
vector channel down reports `degraded=true` **with a reason naming the vector channel**; a query in a
`draining` space carries `space_lifecycle=draining` as a degradation reason. The field name
`degraded` is this contract's construction — the documents require the behaviour but never name the
field (**OI-R-14**).

---

## 6. Rollback obligations (contract only — NOT RUN)

[DOC] 07 §7.9 "回滚六明确": object, condition, window, executor, verification, and post-rollback data
handling must all be named **before** a migration starts. A migration without a drill record must be
refused [DOC] 07 §7.6.1 #8.

| Obligation | Requirement | Status here |
|---|---|---|
| Rollback object | routing config + index version, named per migration | **NOT_RUN** |
| Trigger and window | explicit thresholds; window length is **TBD** | **TBD** |
| Old space retention | old data must not be deleted during any pre-`retired` state | contract only |
| Verification after rollback | query set replay with the pre-migration baseline | **NOT_RUN** |
| Data written during the failed migration | must have an explicit owner; dual-written rows are never silently discarded | **TBD** (OI-024 adjacent) |

Numbers that do **not** exist and must not be invented: backfill coverage threshold beyond "100% or
approved partial", comparison tolerance (07 §7.4.3 #6 "容差值 TBD；禁止以研究报告数值当容差"),
observation window length, error-rate/empty-result thresholds (#9 "阈值 TBD 至实测基线"). All are
**[VALIDATION_PENDING]** and must stay that way until measured on the real pipeline.

---

## 7. The enum conflict that blocks implementation

Three sources, three different sets for the same field:

| Source | Values | Count |
|---|---|---|
| `embedding-space.schema.json` → `migration_state` | `not_started, planned, backfilling, validating, dual_running, cutover_ready, cutover, rollback, completed` | 9 |
| 07 §7.4.2 / SM-8 → nine migration states | `not_started, backfilling, validating, dual_running, switching, monitoring, retired, rolled_back, failed` | 9 |
| 01 table 11 → `embedding_spaces` | only `status(active/retired)` — no migration column at all, no `draining` | 2 |

Overlap is only `not_started, backfilling, validating, dual_running`. The schema has
`planned / cutover_ready / cutover / rollback / completed`; the design has
`switching / monitoring / retired / rolled_back / failed`. **Neither is a superset.** A runner
implementing the schema cannot express `switching`, and a runner implementing 07 cannot emit
`cutover`.

OI-024 records the decision to **split the two fields** (space lifecycle vs migration progress), which
this contract follows, but the consumer-side enum sync is **still open**. Consequences that must be
decided before any migration code is written:

1. Which nine names are authoritative for `MigrationRecord.status`?
2. Does `retired` describe the **space** (lifecycle) or the **migration** (07 puts it in both)?
3. `failed` is a migration state in 07 but has no place in the schema enum — an unrecoverable
   migration is currently unrepresentable.

---

## 8. Invariants

| ID | Invariant | Established by |
|---|---|---|
| INV-E-01 | Identity covers all eight fields; same dimension ≠ same space | `pipeline.space_identity_is_eight_fields` |
| INV-E-02 | A query never compares vectors across spaces (it throws) | `pipeline.space_mismatch_throws` |
| INV-E-03 | No returned item carries foreign-space **vector** evidence | `pipeline.no_item_has_foreign_space_vector_evidence` |
| INV-E-04 | `draining` takes no new writes; `active` does | `pipeline.draining_space_not_writable` |
| INV-E-05 | A draining space still answers reads, and the response advertises it | `pipeline.draining_space_readable_and_reported` |
| INV-E-06 | An incomplete space is unusable, never defaulted | `pipeline.incomplete_space_unusable` |
| INV-E-07 | Degradation is explicit and names the real space used | `pipeline.degradation_is_explicit` |
| INV-E-08 | `retired` serves nothing | code path `SPACE_RETIRED`; **not exercised** (no retired space in the corpus) → [`NOT_VERIFIED`] |
| INV-E-09 | Migration transitions, coverage, tolerance, observation window | **NOT_RUN** — no runner exists |

INV-E-08 is listed as unverified deliberately: `assertUsable` throws `SPACE_RETIRED`, but no test
constructs a retired space, so the claim is a code-read, not a result. It should be added by whoever
owns the space registry.

---

## 9. Open items

| ID | Item | Reading A | Reading B | Needs |
|---|---|---|---|---|
| **OI-R-07** | Meaning of `space_lifecycle = "draining"` | read-only, still answers reads (implemented); required by `embedding-space.schema.json` + 07 §8.2 | 01 table 11 allows only `active/retired`, so `draining` should not exist at all and the state belongs to `migration_state` | design owner |
| **OI-R-13** | Field **name** for space lifecycle | `space_lifecycle` (schema) — implemented | `status` (07 §8.2, 01 table 11, and `MigrationRecord` uses `status` too) | design owner |
| **OI-R-14** | Degradation marker field | response carries `degraded: true` + actual space id (implemented, field name invented) | documents require only "显式降级" without naming a field, so the schema is incomplete | contract owner |
| **OI-R-15** | May a space accept writes while `migration_state ∈ {cutover, rollback}`? | no — routing is not final (implemented) | yes if `space_lifecycle = active`; migration state is informational | design owner |
| **OI-R-16** | `migration_state` enum set | schema's nine (`cutover`/`completed` …) | 07 §7.4.2's nine (`switching`/`monitoring`/`retired`/`failed`) | user (OI-024 follow-up) |

---

## 10. What this contract does not establish

- No migration was run: no backfill, no dual read, no cutover, no rollback, no timing, no coverage
  number. Every "MUST" above is an obligation on untested code.
- No LanceDB table-per-space behaviour was observed; isolation is proven at the function boundary
  over an in-memory corpus.
- No real embedding model was used: vectors come from a SHA-256 stand-in (`_lib/embed.cjs`) that is
  reproducible and space-labelled but carries **no semantics**. A cosine value in these tests says
  nothing about retrieval quality.
- The `dimensions` in the synthetic corpus is 8, chosen for test speed. It is not a recommendation,
  and 1024 must not be treated as a constant anywhere [DOC] 07 §7.5.
