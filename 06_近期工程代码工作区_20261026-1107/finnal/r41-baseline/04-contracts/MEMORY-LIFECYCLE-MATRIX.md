# MEMORY-LIFECYCLE-MATRIX — Memory Core R4.1

| Field | Value |
|---|---|
| Status | **PROPOSED** — verification contract, not an approved design decision |
| Author | Agent F (retrieval area) |
| Date | 2026-11-05 |
| Product authorization | **NOT_AUTHORIZED** — no M01/M03/M04 implementation is authorised by this file |
| Backed by | `03-verification/retrieval/lifecycle-state-machine.cjs` (isolated prototype) + `lifecycle-tests.cjs` (24 checks) + `ILLEGAL-TRANSITION-MATRIX.json` |
| Evidence | `06-evidence/retrieval.jsonl`, ids `EV-RETRIEVAL-*` with `test_id` prefix `lifecycle.` |

Markers used below: **[DOC]** quoted from a design document · **[TEST]** executed check, id given ·
**[PROPOSED]** my construction where documents are silent or disagree · **[PROTOTYPE_POLICY]** a rule
that exists **only** in the prototype and has no design source at all — it must not be cited as design.

---

## 1. Why the axes must be listed separately

Older drafts used one `status` word for things that change for different reasons and by different
authority. The consequence is not cosmetic: "was this memory rejected, or archived, or superseded?"
must have three different answers because the three have different recovery paths and different
retrieval consequences.

Five axes are modelled, and **a change on one axis never implies a change on another** — except
where §5 states an explicit coupling.

| Axis | Values | Initial | Changed by | Source |
|---|---|---|---|---|
| `memory_tier` | `L1`, `L2`, `L3` | `L1` | M04 (review) | [DOC] DD-AUTHORITY-V2 §2.5 |
| `status` | `ACTIVE`, `SUPERSEDED`, `ARCHIVED`, `DELETED` | `ACTIVE` | M04 via M01 | [DOC] DD-AUTHORITY-V2 §2.6 |
| `lifecycle_state` | `candidate`, `pending_review`, `active`, `superseded`, `archived`, `deleted` | `candidate` | M03/M04 | [DOC] 01 §7.7.2, SM-1 |
| `validation_state` | `unverified`, `confirmed`, `disputed`, `rejected` | `unverified` | **M04 only** for `confirmed` | [DOC] 01 §7.7.1, SM-2, D-AP-05 |
| `availability_state` | `available`, `restricted`, `suspended` | `available` | **M05** decides, M01 persists | [DOC] 01 §7.7.3, SM-3 |

`candidate.status` (`PENDING`, `AUTO_ACCEPT`, `NEEDS_REVIEW`, `REJECTED`, `DEFERRED`, `COMMITTED`) is
a **sixth** axis belonging to the candidate-pool object, not to a stored memory. [DOC]
DD-AUTHORITY-V2 §2.7. It is not modelled here.

**Mapping between the V2 and v1 vocabularies** [PROPOSED — no document gives it as a table]:

| `lifecycle_state` | derived `status` | Note |
|---|---|---|
| `candidate`, `pending_review` | *(none)* | not yet a MemoryUnit; only exists in the candidate pool |
| `active` | `ACTIVE` | |
| `superseded` | `SUPERSEDED` | |
| `archived` | `ARCHIVED` | |
| `deleted` | `DELETED` | |

**[TEST]** `lifecycle.orthogonal_tier_and_status` proves all **12** `tier × status` combinations are
representable and invariant-clean, i.e. no tier forces a status. **This is the matrix the brief
asked for:**

|  | ACTIVE | SUPERSEDED | ARCHIVED | DELETED |
|---|---|---|---|---|
| **L1** (temporary) | normal transient fact | an L1 fact replaced within the session | a session-only note archived on expiry | a dropped candidate row |
| **L2** (active) | a plan or phase in progress | an earlier version of that plan | a finished plan kept for reference | a plan the user cancelled |
| **L3** (long-term) | a stable self-description | an outdated long-term statement (history preserved) | a long-term fact retired without contradiction | **requires explicit deletion authority** |

Semantics per cell (what the pair permits):

- **tier never changes what `status` is allowed to be** — every cell above is a legal object.
- **status never changes the tier** — `SUPERSEDED` does not demote an L3 to L2; the tier is a
  retention promise, the status is a current-validity flag. [DOC] DD-AUTHORITY-V2 §2.5 铁律:
  "tier ≠ 重要程度".
- A `DELETED` object keeps its last tier only as forensic metadata; it is not retrievable at any tier.
- **`L1 + DELETED`** and **`L3 + DELETED`** differ in what the deletion must propagate (an L3 may have
  derived Observations and Relations; an L1 usually has none) — that is a propagation cost, not a
  status difference.

**[TEST]** `lifecycle.orthogonal_lifecycle_and_validation`: `active → superseded` leaves
`validation_state` untouched, and `confirmed → disputed` leaves `lifecycle_state` untouched.

---

## 2. The twelve operations

For every operation: precondition, authority (which module may perform it), state before/after,
failure handling, consistency requirement, and which derived indexes must be updated **in the same
logical step**.

"Same logical step" is used precisely:
- **authoritative step** = one SQLite transaction (authority row + `memory_versions` + `outbox_events`
  + `event_ledger`). [DOC] 02 §7.2 ALG-01 seven steps.
- derived indexes (FTS5, LanceDB) are **never** in that transaction; they are driven by the Outbox and
  are *eventually* consistent. [DOC] 01 §7.16 rule; SM-4.

### 2.1 Create
| | |
|---|---|
| Precondition | Schema valid; `subject_id` exists; M05 `canWrite` allows; `evidence_refs` non-empty for a formal memory; idempotency key unused |
| Authority | **M01 only** (single authority path) |
| Before → after | *(absent)* → `lifecycle_state = candidate`, `status = ACTIVE`, `validation_state = unverified`, `version = 1` |
| Authoritative step | INSERT memory + version snapshot + outbox `memory_upsert` + ledger |
| Derived, same logical step | **none.** `index_registry` rows are created by the Worker when it processes the event, not by M01 |
| Failure handling | whole transaction rolls back; no "memory written but event missing"; replay with the same `idempotency_key` returns the first result |
| Consistency | `memory_versions` count == 1; `outbox_events.aggregate_version == 1`; no orphan event |
| Test | `lifecycle.legal_transitions_accepted`; coupling checks |

### 2.2 Validate
| | |
|---|---|
| Precondition | object exists; evidence sufficient; requester is M04 |
| Authority | **M04 only.** M01/M02/M03 writing `confirmed` is an authorization violation [DOC] D-AP-05 |
| Before → after | `unverified → confirmed` (also `→ disputed`, `→ rejected`) |
| Authoritative step | UPDATE validation_state + version + ledger (`operation = confirm`) |
| Derived, same logical step | none transactional; ranking feature S is NOT a validation signal |
| Failure handling | reject, state unchanged; **no partial**: validation may not change while lifecycle stays inconsistent (see §5) |
| Consistency | `rejected` is terminal under the 01 reading (see OI-R-03) |
| Test | `lifecycle.authority_only_m04_confirms`, `lifecycle.illegal_rejected_validation` |

### 2.3 Persist
| | |
|---|---|
| Precondition | as Create/Update |
| Authority | M01 |
| Before → after | durable row; `recorded_at` fixed; `observed_at` unchanged |
| Authoritative step | same transaction as the write that produced it |
| Derived | none |
| Failure handling | `STORAGE_ERROR`; retry with same idempotency key |
| Consistency | `PRAGMA journal_mode=WAL`, `foreign_keys=ON`, `busy_timeout` set [DOC] 01 §7.16 |
| Test | not exercised here (no SQLite in this area) — Agent A owns it |

### 2.4 Index
| | |
|---|---|
| Precondition | an outbox event with `status=pending` and `aggregate_version = current` |
| Authority | **Outbox Worker only**; M01 may not write FTS5/LanceDB directly |
| Before → after | `outbox: pending → processing → succeeded`; `index_registry.status: lagging → in_sync` |
| Authoritative step | none (this *is* the derived step); the event row changes status |
| Derived, same logical step | FTS5 document **and** vector row **and** `index_registry` — all three, or the event is not `succeeded` |
| Failure handling | partial failure → `retry_wait` (never a partial success); `retry_count > retry_max` → `dead_letter` + alarm; **the authoritative row is never rolled back** [DOC] SM-4 |
| Consistency | `index_registry.last_synced_at` monotone per `(index_name, memory_id)`; re-processing an old version must not overwrite a newer one (`SkipOrSupersede`) |
| Test | not exercised here — Agent G owns the outbox |

### 2.5 Retrieve
| | |
|---|---|
| Precondition | `lifecycle_state = active` **and** `availability_state ≠ suspended`; M05 allows each item |
| Authority | M02 (read only; may write `last_retrieved_at` and nothing else) |
| Before → after | no state change; `last_retrieved_at` may advance |
| Authoritative step | none |
| Derived | none |
| Failure handling | channel failure ⇒ `partial`; permission error ⇒ exclude (fail-closed) |
| Consistency | a `deleted` memory is invisible even if its index row survives [DOC] 01 C-13 / SM-7 |
| Test | `lifecycle.retrievability_gate_matches_document`, `pipeline.deleted_never_returned` |

### 2.6 Update
| | |
|---|---|
| Precondition | object exists; `expected_version` matches; M05 `canUpdate`; merge/consolidation passed if it is a merge |
| Authority | M01 (content) with M04 driving consolidation; M02 may never trigger a write |
| Before → after | `active → active`, `version + 1`; old version row retained |
| Authoritative step | update memory + insert version snapshot + outbox `memory_upsert` (`aggregate_version = new`) + ledger |
| Derived, same logical step | none transactional; the Worker must reindex both FTS5 and the vector for the new text/projection |
| Failure handling | `VERSION_CONFLICT` on mismatch (no last-write-wins); rollback |
| Consistency | `memory_objects.version` == `max(memory_versions.version)`; outbox `aggregate_version` monotone |
| Test | `lifecycle.legal_transitions_accepted` (the `active → active` row is SM-1 #4) |

### 2.7 Supersede
| | |
|---|---|
| Precondition | a newer version exists and is at least as validated; M04 decision recorded |
| Authority | M04 decides, M01 persists |
| Before → after | old `active → superseded`; new object `active`; `SUPERSEDES` relation written |
| Authoritative step | both objects' status + relation + outbox events + ledger (`operation = supersede`) |
| Derived | old object must leave the default recall set on the next query; the *indexes* keep the row, the retrieval gate excludes it |
| Failure handling | if the relation write fails, the whole transaction rolls back (no "superseded with no successor") |
| Consistency | exactly one `active` member per supersession chain; `superseded → active` is forbidden (**[TEST]** `lifecycle.illegal_rejected_lifecycle`) |
| Test | `lifecycle.legal_transitions_accepted`, `lifecycle.retrievability_gate_matches_document` |

### 2.8 Expire
| | |
|---|---|
| Precondition | per-type policy: `valid_until`/`expires_at` reached, source no longer valid, or superseded; **not** a single global TTL |
| Authority | M04 (policy) + M01 (persist) |
| Before → after | `active → archived` (default) or `→ deleted` only with explicit authority |
| Authoritative step | update + outbox `memory_reindex`/`memory_delete` + ledger |
| Derived | depends on the outcome: archive keeps the index row (retrieval excludes it); delete needs the full SM-7 propagation |
| Failure handling | auto-archive is **not authorised** [DOC] 02 §7.11 §13/§14 "自动归档/自动删除 MUST 分别授权" — an implementation must not enable it by default |
| Consistency | decay affects **ranking only** and never deletes [DOC] 02 §7.11 §5.2 |
| Test | `lifecycle.legal_transitions_accepted` (`active → archived`); auto-expiry **not** implemented here |

### 2.9 Archive
| | |
|---|---|
| Precondition | human-controlled policy or explicit user action; M05 read/write allow |
| Authority | user / authorised maintenance via M04, persisted by M01 |
| Before → after | `active → archived`; `status = ARCHIVED`; still searchable only through an explicit archive scope |
| Authoritative step | update + ledger |
| Derived | index rows retained; retrieval gate excludes archived from the default set |
| Failure handling | archive failure ⇒ state unchanged |
| Consistency | archived ≠ deleted (a deletion must never be implemented as "archive + filter") [DOC] 02 §7.11 §17 |
| Test | `lifecycle.retrievability_gate_matches_document` |

### 2.10 Delete
| | |
|---|---|
| Precondition | M04 judgement **and** M05 `canDelete`; tombstone written |
| Authority | M04 + M05; M01 persists and performs physical cleanup in the Worker |
| Before → after | `active|superseded|archived → deleted`; `status = DELETED`; `deletion` machine `requested → tombstoned` |
| Authoritative step | tombstone + outbox `memory_delete` + ledger — **the authoritative deletion takes effect immediately** |
| Derived, same logical step | **none can be transactional.** Deletion propagation is asynchronous (SM-7 six steps: FTS5 → LanceDB → cache → derived Observation/Relation → audit). The authority is removed *now*; index rows are removed *later* |
| Failure handling | a failed propagation returns to `retry_wait`; **the authoritative deletion never rolls back** [DOC] SM-7 rules |
| Consistency | `deleted → any` is forbidden (**[TEST]** `lifecycle.deleted_blocks_every_transition`); a repeated delete is idempotent and must not resurrect (**[TEST]** `lifecycle.legal_transitions_accepted` includes `deleted → deleted`) |
| Test | above + `lifecycle.authority_delete_needs_candelete` |

### 2.11 Rebuild
| | |
|---|---|
| Precondition | index version mismatch, projection/chunker change, or corruption detected; approval per 07 §12 |
| Authority | operational tooling, not M01/M02; the authoritative rows are only **read** |
| Before → after | `index_registry.status: in_sync → lagging → in_sync`; no memory state change |
| Authoritative step | none |
| Derived, same logical step | the rebuilt index and its `index_version` must switch together; the old index is kept for one observation window [DOC] 07 §7.7 |
| Failure handling | abort leaves the old index serving; never "half rebuilt and switched" |
| Consistency | rebuild counts must reconcile with `memory_objects` (no memory lost, none invented) |
| Test | **not exercised** — this area has no index implementation |

### 2.12 Restore
| | |
|---|---|
| Precondition | backup verified; for a delete, the tombstone must be replayed after restore [DOC] SM-7 |
| Authority | operational recovery + user authority; **not** an ordinary M01 write |
| Before → after | `(absent)` or `deleted → active` **only via the restore procedure**, never via an update retry |
| Authoritative step | restore transaction + ledger |
| Derived, same logical step | indexes are rebuilt from the authority, not restored from the backup as truth |
| Failure handling | partial restore must be detectable: `verifyIntegrity` before serving [DOC] TC-M01-010 |
| Consistency | restoring a backup must **not** resurrect a memory that was deleted after that backup (tombstone replay) |
| Test | **not exercised** — the prototype refuses `deleted → active` entirely, which is the conservative side of this rule |

---

## 3. Authority matrix

[DOC] 02 §7.11 §4.1, 01 §7.7, 04 §7.11.

| Transition group | Authorised module | M01's role | Everyone else |
|---|---|---|---|
| validation (`unverified/confirmed/disputed/rejected`) | **M04** (M03 may only set `rejected` from `unverified`) | persists | **forbidden** — `M01/M02/M03` writing `confirmed` is an authorization violation (D-AP-05) |
| lifecycle (`candidate…deleted`) | M03 (candidate edges) / M04 (review and evolution) | persists | **M02 forbidden** |
| availability (`available/restricted/suspended`) | **M05** decides | persists | others may only *request* |
| delete | M04 judgement + M05 `canDelete` | persists + physical cleanup | — |
| tier (`L1/L2/L3`) | M04 | persists | Dream may only *suggest* |

**[TEST]** `lifecycle.authority_m02_cannot_change_lifecycle` (3 legal edges attempted by M02, all
`PERMISSION_DENIED`), `lifecycle.authority_only_m04_confirms`, `lifecycle.authority_availability_only_m05`,
`lifecycle.authority_delete_needs_candelete`.

---

## 4. Illegal transitions

The machine rejects **31** transitions under the strict (`v1-01`) profile. Each row is generated from
`AXES × LEGAL_TABLE` — not hand-written — and each row was individually exercised: the artifact
`ILLEGAL-TRANSITION-MATRIX.json` records, per row, the axis, from, to, reason, and the code the
machine actually returned in this run.

| Axis | Illegal rows | Examples | Test |
|---|---:|---|---|
| `lifecycle` | 18 | `candidate → active` (skips M04), `superseded → active` (no in-place rollback), `deleted → *` (terminal) | `lifecycle.illegal_rejected_lifecycle` |
| `status` | 5 | `DELETED → *`, `SUPERSEDED → ACTIVE`, `ARCHIVED → SUPERSEDED` | `lifecycle.illegal_rejected_status` |
| `validation` | 5 | `rejected → confirmed`, `confirmed → unverified`, `disputed → unverified` | `lifecycle.illegal_rejected_validation` |
| `availability` | 1 | `suspended → available` (must pass through `restricted` under the 01 reading) | `lifecycle.illegal_rejected_availability` |
| `tier` | 2 | `L1 → L3`, `L3 → L1` (level skipping) | `lifecycle.illegal_rejected_tier` |

Self-transitions (`active → active`, `deleted → deleted`, …) are treated as no-ops: the first is a
content update (SM-1 #4), the second is idempotent deletion (SM-1 TC-SM1-03).

**The `sm-2` profile has 30 illegal rows, not 31**, because SM-2 legalises `rejected → unverified`.
The difference is itself a test result (`lifecycle.disagreement_rejected_to_unverified`), not a
tuning choice.

---

## 5. Coupling constraints (where axes are *not* independent)

| ID | Constraint | Enforcement |
|---|---|---|
| C-02 | `active` requires `validation_state ∈ {confirmed, disputed}` | **[TEST]** `lifecycle.coupling_activation_requires_validation` — `pending_review → active` alone is refused and the object is unchanged |
| C-02b | `candidate`/`pending_review` requires `validation_state = unverified` | same test family; `pending_review + disputed` is refused |
| C-03 | `pending_review` requires non-empty `evidence_refs` | **[TEST]** `lifecycle.coupling_pending_review_needs_evidence` |
| C-status | `status` must agree with `lifecycle_state` (derived, not independent) | **[TEST]** `lifecycle.orthogonal_tier_and_status` builds all 12 legal pairs |
| C-delete | `to = deleted` requires `ctx.canDelete === true` | **[TEST]** `lifecycle.authority_delete_needs_candelete` |

**Atomicity rule.** Because of C-02/C-02b, several legal moves are only legal *together*: activation
and confirmation must happen in one logical step. The prototype therefore validates invariants on the
**result** of a batch, not on intermediate states, and rolls the whole batch back on violation.

**[TEST]** `lifecycle.coupling_batch_atomic_success` (confirmation + activation accepted together)
and `lifecycle.coupling_batch_atomic_rollback` (a batch whose result would violate C-02 leaves the
object byte-identical). This is the "MUST NOT 部分应用" rule of [DOC] 02 §7.11 §9.

---

## 6. Prototype boundary — what this does **not** implement

`lifecycle-state-machine.cjs` is an **isolated prototype**, explicitly not product code:

| Not modelled | Consequence |
|---|---|
| SQLite persistence, transactions, WAL | atomicity is proven in memory only; no crash behaviour |
| Outbox / Worker / SM-4 | index propagation is described in §2 but not executed |
| Audit ledger / provenance records | "MUST 记录 actor + reason + version" is stated, not enforced |
| Real M05 | `actor` is a caller-supplied string; `canDelete` is a caller-supplied boolean |
| Version increments | no `version + 1` bookkeeping (OI-01-05 leaves this open) |
| Timestamps, ids | not generated |
| Tombstone replay / restore | restore is refused outright (`deleted` is terminal in the prototype) |

**[PROTOTYPE_POLICY]** Tier transitions are **not defined by any design document**. The rules used
here (no level skipping; L3 promotion requires `confirmed`; any move requires explicit review) exist
only to exercise the axis, are labelled `VALIDATION_PENDING`, and must not be cited as design.

---

## 7. Open items carried by this matrix

Detail and both readings for each: `RETRIEVAL-CONTRACT.md` §8. Summary of the ones that change this
matrix:

| ID | Question | Effect on this matrix |
|---|---|---|
| OI-R-01 | is `pending_review → deleted` legal? | one extra legal edge, or one more illegal row |
| OI-R-02 | is `archived → active` legal? | restore path exists or only via backup |
| OI-R-03 | is `rejected → unverified` legal? | 31 vs 30 illegal rows; terminal vs revivable |
| OI-R-05 | may `suspended` resume directly? | one illegal row disappears under the SM-3 reading |
| OI-R-06 | is `disputed` a lifecycle state or a validation state? | whether SM-1 or 01 §7.7 is the machine |
| OI-R-17 | **there is no tier transition table anywhere in the 24 design documents.** The prototype gates `L2 → L3` on `ctx.tier_review_approved`, a flag that appears **zero times** in the corpus. Recorded as `PARTIAL_PASS` (`lifecycle.tier_promotion_gate_is_explicit`): the check shows the prototype's behaviour under an invented precondition, **not** that the design permits a tier move. An independent re-execution with design-specified input only refused the move and recorded "the tier axis has no permitted transition" (`EV-RETRIEVAL-P002`, which labelled it OI-R-07 — a numbering collision with the `draining` item; both were reported to the parent, who keeps OI-R-07 for the gap) | whether tier may move at all, and on what authority | the tier axis' legal rows are **PROTOTYPE_POLICY**; §2's `memory_tier` column and the C-* invariants rest on it. **The green on this axis is not evidence of design support** |

Two further conflicts this matrix inherits from the parent's register (not mine to resolve):
**OI-01-05** (does a status change increment `version`?) and **OI-01-03** (may a migrated object be
retrievable before confirmation?). Both change the "before → after" column of §2.2/§2.6.
