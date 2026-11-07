# OUTBOX CONTRACT (verification proposal)

| Field | Value |
|---|---|
| Contract id | `OUTBOX-CONTRACT` |
| Status | **PROPOSED** — verification output, not an approved design change |
| Basis | `01-baseline-corpus/design-package/03-系统运行与协作协议.md` §7.6/§7.7, `05-可靠性与运维规范.md` §7.5, `contracts/schemas/outbox-event.schema.json`, `contracts/config/worker.schema.json`, `contracts/state-machines/state-machines.md` SM-4 |
| Measured against | `03-verification/outbox/outbox-simulator.cjs`, `outbox-tests.cjs`, `crash-recovery-test.cjs` |
| Evidence | `06-evidence/outbox.jsonl` ids `EV-OUTBOX-001…021` (see §11) |
| Implementation authorization | none. `PRODUCT_IMPLEMENTATION_AUTHORIZATION` remains **NOT_AUTHORIZED**; this document describes an isolated prototype. |

Every numeric value in this document is either (a) quoted from the design package with its
source, or (b) labelled `EXPERIMENTAL / VALIDATION_PENDING`. Nothing here is a settled
operational value (MC-AUD-008).

---

## 1. What the outbox is for

The outbox carries "an authoritative memory change happened" from the committing transaction
to the derived indexes. It exists because the authority (SQLite) and the indexes (FTS5 in the
same file; LanceDB in another store) cannot be committed together. Its whole job is to make the
gap between them recoverable, not to make it atomic.

Two invariants bound everything below:

1. **No event without a write, no write without an event.** The authority row, its version row
   and the pending event are written in one transaction (`03 §7.4` five-step transaction).
2. **Index failure never rolls back authority.** A failed index write only changes the event's
   scheduling state (`03 §7.6.7` rule 9).

Both are measured: `EV-OUTBOX-001` (all three rows appear together), `EV-OUTBOX-002` (a failure
anywhere in the transaction leaves zero of the three), `EV-OUTBOX-016/017` (the same holds across
an OS-level kill).

---

## 2. States and transitions

Five states, exactly the `outbox-event.schema.json` `status` enum:
`pending` · `processing` · `succeeded` · `retry_wait` · `dead_letter`.

| # | From | To | Trigger | Actor | Terminal? | Measured by |
|---|---|---|---|---|---|---|
| 1 | *(none)* | `pending` | authority transaction commits | M01 write path | no | `EV-OUTBOX-001` |
| 2 | `pending` | `processing` | claim (atomic UPDATE, §3) | worker | no | `EV-OUTBOX-004`, `009` |
| 3 | `retry_wait` | `processing` | claim, only when `next_attempt_at <= now` | worker | no | `EV-OUTBOX-007` |
| 4 | `processing` | `succeeded` | every channel effect committed **and** the terminal CAS on the claim token wins | worker | yes | `EV-OUTBOX-004` |
| 5 | `processing` | `retry_wait` | channel failure, `attempts <= dead_letter_after` | worker | no | `EV-OUTBOX-006` |
| 6 | `processing` | `dead_letter` | channel failure, `attempts > dead_letter_after` | worker | yes (until replay) | `EV-OUTBOX-010` |
| 7 | `processing` | `succeeded` *(marked)* | `aggregate_version < current` ⇒ SkipOrSupersede; `last_error='superseded_by_version'` | worker | yes | `EV-OUTBOX-013` |
| 8 | `processing` | `succeeded` *(marked)* | aggregate row no longer exists; nothing to index; `last_error='aggregate_absent'` | worker | yes | `EV-OUTBOX-004` |
| 9 | `dead_letter` | `pending` | manual repair + replay, `attempts` reset to 0, audited | operator | no | `EV-OUTBOX-011` |
| 10 | `processing` | `processing` *(reclaim)* | lease expired; a different worker claims with a new token | worker | no | `EV-OUTBOX-008`, `018` |

Notes that the design does not settle but the prototype had to decide (all marked as decisions,
not as design facts):

- **D-1 (superseded / absent are not new states).** Both are recorded as `succeeded` with a
  `last_error` marker, following SM-4 row 6. No new enum value is introduced.
- **D-2 (`cancelled` is unusable as specified).** SM-4 row 8 has `pending/processing → cancelled`,
  but `outbox-event.schema.json` has `additionalProperties:false` and a five-value `status` enum
  that does not contain `cancelled`. An event carrying `cancelled` is invalid per the schema.
  This prototype therefore never emits it (see finding **O-F1** in §10).
- **D-3 (reclaim counts as an attempt).** A reclaim increments `attempts`, so a crash loop is
  bounded by the same ceiling as an error loop. Consequence: a worker that crashes on one event
  repeatedly will dead-letter it after `dead_letter_after + 1` claims — the ceiling arithmetic is
  measured for the error loop (`EV-OUTBOX-010`); that a reclaim consumes an attempt is measured for
  a single reclaim (`EV-OUTBOX-008`, `018`). A long crash loop was **not** driven to the ceiling.
- **D-4 (`retry_max` == `dead_letter_after` or refuse to start).** `storage.schema.json` defines
  both keys, both defaulting to 8, both meaning "the retry ceiling". The loader rejects a
  configuration where they disagree rather than letting one silently win (`EV-OUTBOX-014`).

---

## 3. Claim and lease semantics

**Claim** is a single statement, inside `BEGIN IMMEDIATE`, that both selects and mutates:

```sql
BEGIN IMMEDIATE;
UPDATE outbox
   SET status='processing', lease_owner=?, lease_until_ms=?, lease_until=?,
       claim_token=?, claimed_at_ms=?, attempts=attempts+1, retry_count=attempts+1,
       last_error=NULL
 WHERE event_id IN (
     SELECT event_id FROM outbox
      WHERE (status='pending'    AND (next_attempt_at_ms IS NULL OR next_attempt_at_ms <= ?))
         OR (status='retry_wait' AND (next_attempt_at_ms IS NULL OR next_attempt_at_ms <= ?))
         OR (status='processing' AND lease_until_ms IS NOT NULL AND lease_until_ms <= ?)
      ORDER BY CASE WHEN event_type='memory_delete' THEN 0 ELSE 1 END, created_at, event_id
      LIMIT ?)
RETURNING event_id, event_type, aggregate_id, aggregate_version, attempts, claim_token, …;
COMMIT;
```

Why this shape:

- `BEGIN IMMEDIATE` takes the write lock **before** the candidate rows are read, so two workers
  cannot both read the same `pending` row and only fail later on upgrade.
- The candidate set and the state change are one statement: there is no read-then-write gap for
  another worker to slip into.
- `ORDER BY` puts deletes first (`worker.delete_priority`, `03 §7.6.8`) and then oldest-first, so
  a backlog drains in age order.

**Lease.** `lease_until = claimed_at + worker.claim_timeout_ms`. While the lease is valid, no
other worker can claim the row; the only path back into `processing` for a live row is the
`processing AND lease_until_ms <= now` branch, i.e. expiry.

**Claim token = the fence.** Every claim mints `claim_token = <worker>.<epochms>.<random>`. The
terminal write is a compare-and-swap:

```sql
UPDATE outbox SET status=?, processed_at=?, …
 WHERE event_id=? AND claim_token=? AND status='processing';
```

If the row was reclaimed, the CAS affects 0 rows and the stale worker learns it lost
(`EV-OUTBOX-008`: `stale CAS changes=false`). **The fence protects the event's state; it cannot
undo an effect the stale worker already committed elsewhere** — see §6.

Measured lease behaviour (`EV-OUTBOX-008`, lease 250 ms):

| Observation | Value |
|---|---|
| lease granted to worker A | 249 ms (`lease_until - claimed_at`) |
| B could reclaim after | 414 ms after A's claim (lease + poll + scheduling) |
| A's late terminal CAS | 0 rows changed |
| final `attempts` | 2 |
| final owner | `w-B` |

Startup does not steal a live lease: with two `processing` rows (one live lease, one expired),
a freshly started instance reported `processing=2, expired_leases=1` and claimed exactly the
expired one (`EV-OUTBOX-032`).

**`claim_timeout_ms` is `EXPERIMENTAL / VALIDATION_PENDING`.** The contract default in
`worker.schema.json` is 60 000 ms; the harness used 150–5 000 ms to keep runs short. Neither number
is a recommendation. A lease must exceed the p99 of one event's processing time, including the
slowest channel; that measurement does not exist yet.

---

## 4. Retry and backoff

On a failed attempt, with `attempts` the value **after** the current attempt:

```
delay_ms = min(retry_backoff_base_ms * retry_backoff_factor^(attempts-1), retry_backoff_cap_ms)
next_attempt_at = now + delay_ms        -- and the row returns to retry_wait
```

The gate is the SQL predicate `next_attempt_at_ms <= now`; the polling loop does **not** implement
timing. Measured gaps (`EV-OUTBOX-007`, base 150 ms, factor 2, cap 1000 ms, poll 10 ms):

| Attempt | Expected delay | Measured gap | Delta |
|---|---|---|---|
| 2 | 150 ms | 163 ms | +13 |
| 3 | 300 ms | 325 ms | +25 |
| 4 | 600 ms | 608 ms | +8 |

Measured gaps are always ≥ the configured delay and exceed it by roughly the poll interval plus
scheduler latency, which is the expected direction. The harness asserts
`delay - 10 <= gap <= delay + 300`; it does not assert an exact value.

`retry_backoff_base_ms` / `retry_backoff_factor` / `retry_backoff_cap_ms` are
**EXPERIMENTAL / VALIDATION_PENDING**. The design's own values (1 000 / 2 / 60 000, `03 §7.7` item 7)
are stated as `EXP-DEFAULT` there and are not derived from any measurement of this system.

---

## 5. Dead-letter threshold

The design says `retry_count > retry_max ⇒ dead_letter` (`03 §7.6.6`, SM-4 row 5). The prototype
implements exactly that, with `attempts` as `retry_count`:

```
on failure: attempts += 1 happens at claim time;
            dead_letter if attempts > dead_letter_after, else retry_wait.
```

So with `dead_letter_after = 3`: attempts 1, 2, 3 → `retry_wait`; attempt 4 fails → `dead_letter`.
Measured (`EV-OUTBOX-010`): `status=dead_letter, attempts=4, retry_count=4, dead_lettered_at` set,
`next_attempt_at` cleared, and the attempt log stopped at 4 (a further 400 ms of polling produced
no new attempt).

`dead_letter_after` is **EXPERIMENTAL / VALIDATION_PENDING** (contract default 8, unexplored).

Two schema-level caveats are findings, not resolved here: the ceiling is configurable per the
design but the *reason* an event is dead-lettered is only in `last_error`, and `last_error` is a
1024-character sanitised string — it is not a machine-readable failure class. An operator tool
that groups dead letters by cause would need a code column the schema does not have.

---

## 6. Idempotency, and exactly-once — what holds and what does not

### 6.1 Three different keys, three different jobs

| Key | Scope | Enforces | Measured |
|---|---|---|---|
| `idempotency_key` | request → outbox row | submitting the same write twice creates one event | `EV-OUTBOX-003` (3 submissions → 1 row, 1 effect) |
| `event_id` | event identity | re-delivering the same event is safe | `EV-OUTBOX-005` |
| `(event_id, channel)` journal | one index channel | an already-applied channel is never applied again | `EV-OUTBOX-006`, `019` |

`event_id` is the outbox's own idempotency key (`outbox-event.schema.json`: "幂等键"), while
`idempotency_key` is the request-level key from `03 §7.3.2`. They are **not** the same field and
the schema only has the first — see finding **O-F1**.

### 6.2 The claim mechanism, stated precisely

Exactly-once for an effect holds **if and only if** the effect and its journal entry are committed
in the same SQLite transaction as the record that says "this channel is done". This prototype
implements that as `outbox_channel_state`, written with the index write:

```
BEGIN IMMEDIATE;
  <index write for channel c>
  INSERT INTO effects(...)                       -- audit
  INSERT/UPSERT outbox_channel_state(event,c,'applied')
COMMIT;
... later ...
UPDATE outbox SET status='succeeded' WHERE event_id=? AND claim_token=? AND status='processing';
```

If the process dies before that COMMIT, neither the effect nor the journal exists, and the
reclaiming worker applies the channel once. If it dies after, the channel is skipped on reclaim
and only the status flip remains. There is no state in which the effect exists and the journal
does not — **within one database file**.

Measured, kill-after-first-channel (`EV-OUTBOX-019`): 1 effect before the kill, journal says
`fts=applied`, recovery produced `effects fts=1 vec=1`, final status `succeeded`.

### 6.3 Where exactly-once does **not** hold — two windows

Both windows exist because LanceDB is a different store and cannot join the SQLite transaction.
The prototype reproduces the boundary with a second SQLite file (`index.sqlite`), which has the
same property that matters: no shared commit.

**Window W1 — crash between the external commit and the journal commit.**

```
  t0  external store: index row written, committed
  t1  <-- crash here
  t2  authority:      outbox_channel_state('applied') + status flip
```

Measured (`EV-OUTBOX-020`): after the kill, the external effect log held **1** application and the
authority held **0** journal rows; recovery re-applied the channel, so the external log held **2**
applications while the keyed index row stayed **1**. The duplicate is real; the business state
converges only because the write is an upsert keyed by `aggregate_id`.

**Window W2 — a live worker whose lease expires mid-effect.**

```
  t0  worker A: external commit
  t1  A stalls past claim_timeout_ms; its lease expires
  t2  worker B: claims, applies the same channel again, completes
  t3  A wakes, journals, tries the terminal CAS → fenced (0 rows)
```

Measured (`EV-OUTBOX-021`): marker written, claim token changed, B's attempt 2, A fenced (`true`),
external effects `fts=2, vec=1`, index rows `1`, final status `succeeded`. The fence stops A from
writing the *state*; it cannot stop the effect A already committed.

**Therefore:** the honest claim for this mechanism is

> **At-least-once per channel across a store that cannot join the authority transaction, and
> exactly-once per channel for effects that share the authority's transaction (FTS5).**

The duplicate is harmless for an idempotent upsert keyed by `(aggregate_id, aggregate_version)`:
the second application overwrites with the same content. It is **not** harmless for any effect
that appends, increments or emits an external side effect (notification, counter, audit row in
another system). For those, the two windows above produce a genuine duplicate, and no amount of
local journaling closes them without either (a) a transactional commit spanning both stores or
(b) an idempotency key inside the external store that makes the second write a no-op.

Consequences a decision must accept before this ships:

1. Every index write **must** be an upsert/delete keyed by `(aggregate_id, aggregate_version)`.
2. No non-idempotent side effect may be attached to the outbox path.
3. Duplicate applications are *detectable* (the `effects` log records each application with
   worker and attempt) and must be *counted* by monitoring, not treated as impossible.

---

## 7. Manual recovery procedure (dead letter)

Prerequisites: the operator can read the database, and the cause of the failure is fixed (an
embedding endpoint is back, a corrupt payload is corrected, a permission grant is restored).

1. **Stop the worker** for the affected event class, or set its dead-letter handling off, so the
   event is not claimed while it is being inspected.
2. **Read the row**: `status='dead_letter'`, `attempts`, `dead_lettered_at`, `last_error`, plus
   `outbox_channel_state` to see which channels already succeeded.
3. **Diagnose**. `last_error` is sanitised and must not contain memory text; if it is not enough,
   correlate with `correlation_id`/`request_id` in the runtime log.
4. **Fix the cause** outside the outbox. Do **not** edit the payload to make it pass; a changed
   payload with an unchanged `aggregate_version` breaks the version guard.
5. **Reset**: `status='pending'`, `attempts=0`, `retry_count=0`, `next_attempt_at=NULL`,
   `dead_lettered_at=NULL`, `last_error=NULL`, lease fields cleared. **Preserve
   `outbox_channel_state`** — a channel that already succeeded must not be re-applied.
6. **Audit** the reset (who, when, why) — the prototype inserts an `audit_log` row
   (`EV-OUTBOX-011`).
7. **Restart the worker** and confirm the event reaches `succeeded`.

Measured (`EV-OUTBOX-011`): after replay, the already-applied channel count did not change
(1 → 1) and only the failed channel was applied (0 → 1); the row went `dead_letter → pending
(attempts=0) → succeeded` with 1 audit row.

`resetChannels:true` (clearing the journal) exists as the destructive variant and is recorded in
the audit row when used. It re-applies every channel and is only correct when the index itself was
lost.

**Not covered:** replaying an event whose *aggregate* was deleted while it sat in `dead_letter`.
The version guard turns it into `superseded_by_version` only if the aggregate still exists to
compare against; if it does not, the prototype's `aggregate_absent` short-circuit applies. Neither
path was exercised with a dead-lettered event.

---

## 8. Configuration surface and its status

| Key | Source | Value used in the harness | Status |
|---|---|---|---|
| `worker.claim_timeout_ms` | `worker.schema.json` (default 60000) | 150–5000 ms per test | EXPERIMENTAL / VALIDATION_PENDING |
| `worker.max_events_per_tick` | `worker.schema.json` (default 500) | 1 (concurrency test), 50 (default) | EXPERIMENTAL / VALIDATION_PENDING |
| `worker.partial_failure_policy` | `worker.schema.json` (`retry_failed_only`) | `retry_failed_only` | contract value, tested (`EV-OUTBOX-006`) |
| `worker.delete_priority` | `worker.schema.json` (default true) | true | contract value, tested (`EV-OUTBOX-012`) |
| `storage.outbox.retry_backoff_base_ms` | default 1000 | 60–200 ms | EXPERIMENTAL / VALIDATION_PENDING |
| `storage.outbox.dead_letter_after` | default 8 | 2–6 | EXPERIMENTAL / VALIDATION_PENDING |
| `storage.outbox.retry_max` | default 8 | rejected if ≠ `dead_letter_after` | conflicts with `dead_letter_after` — finding O-F2 |
| backoff factor / cap | `03 §7.7` (2 / 60000) | 2 / 1000 ms | EXPERIMENTAL / VALIDATION_PENDING |

No key above is a settled operational value. A decision on any of them needs a measured p99
processing time per channel and a stated acceptable recovery latency, neither of which exists.

---

## 9. Known gaps this contract does **not** close

1. **Duplicate detection is passive.** The prototype detects duplicates by counting its own
   effect log. There is no production-side detector, alert threshold, or reconciliation job.
2. **No fencing of the external store.** The claim token fences the outbox row only. An
   external store that accepted a write from a fenced worker keeps it.
3. **Backlog precedence is age-based only.** A long backlog of deletes starves upserts of the same
   order of magnitude; no fairness rule exists in the design.
4. **`stale_event_age_ms` is unimplemented.** `worker.schema.json` defines a stale-event alarm;
   nothing here raises it. The startup report (`processing`, `expired_leases`) is the closest
   thing to the observable the alarm would need.
5. **No poison-payload protection.** An event whose payload can never succeed burns the full
   attempt budget before dead-lettering; the ceiling is the only backstop.

---

## 10. Findings for the parent (carried, not resolved)

| Id | Finding | Evidence |
|---|---|---|
| **O-F1** | `outbox-event.schema.json` has `additionalProperties:false` and no lease/claim fields. The mechanism the design mandates (`03 §7.7` item 2 "认领需可恢复/带租约或心跳", SM-4 row 1 "租约超时后回到 pending") cannot be expressed by the schema. Missing: `idempotency_key`, `lease_owner`, `lease_until`, `claim_token`, `dead_lettered_at` — five fields (`attempts` duplicates the existing `retry_count`; `next_attempt_at` duplicates `next_retry_at`). A conforming validator rejects a row produced by this prototype. | `EV-OUTBOX-015` (schema-gap audit) |
| **O-F2** | `storage.schema.json` defines two keys for one ceiling (`retry_max`, `dead_letter_after`), both defaulting to 8. Ambiguous which is authoritative. The loader refuses a disagreement. | `EV-OUTBOX-014` |
| **O-F3** | SM-4 adds a sixth state `cancelled` that the schema enum does not contain; an event in that state is schema-invalid. Either the enum or SM-4 must change. | `EV-OUTBOX-015` |
| **O-F4** | Exactly-once does not hold across the LanceDB boundary (windows W1/W2, §6.3). The design text ("事件处理 MUST 幂等", "相同事件重复执行 MUST NOT 产生重复业务结果") is achievable only as "idempotent effect", not as "effect executed once". This should be stated as such in `03`/`05`. | `EV-OUTBOX-020`, `021` |
| **O-F5** | The brief names the scheduling field `next_attempt_at`; the design and schema name it `next_retry_at`. The prototype uses the brief's name internally and treats `next_retry_at` as the contract-facing name. One name must win. | n/a (naming only) |

---

## 11. Evidence map

| Test id | Evidence | What it establishes |
|---|---|---|
| `outbox.enqueue_atomic_with_authority_write` | EV-OUTBOX-001 | five-step transaction co-commits memory/version/event |
| `outbox.enqueue_rollback_leaves_no_partial_state` | EV-OUTBOX-002 | no half-state on failure |
| `outbox.duplicate_enqueue_same_idempotency_key` | EV-OUTBOX-003 | enqueue-side idempotency |
| `outbox.normal_processing` | EV-OUTBOX-004 | backlog drains to index parity |
| `outbox.duplicate_delivery_is_idempotent` | EV-OUTBOX-005 | redelivery absorbed by the journal |
| `outbox.partial_failure_not_success` | EV-OUTBOX-006 | partial failure is not success; channel-level retry |
| `outbox.retry_backoff_measured` | EV-OUTBOX-007 | measured backoff gaps |
| `outbox.lease_expiry_reclaim` | EV-OUTBOX-008 | lease reclaim + fencing |
| `outbox.concurrent_workers_no_double_process` | EV-OUTBOX-009 | two OS processes, no double claim |
| `outbox.dead_letter_after_ceiling` | EV-OUTBOX-010 | ceiling semantics |
| `outbox.dead_letter_replay` | EV-OUTBOX-011 | manual recovery, journal preserved |
| `outbox.delete_priority` | EV-OUTBOX-012 | delete-first ordering |
| `outbox.version_guard_skip_or_supersede` | EV-OUTBOX-013 | stale event cannot overwrite a newer index |
| `outbox.config_guards` | EV-OUTBOX-014 | conflicting retry ceilings rejected |
| `outbox.schema_gap_audit` | EV-OUTBOX-015 | O-F1 / O-F3 measured |
| `recovery.*` | EV-OUTBOX-016…021 | kill/reclaim/exactly-once windows |

*(Evidence ids are assigned by append order; re-running `run-all.cjs` assigns a fresh contiguous
block. The test id is the stable key.)*
