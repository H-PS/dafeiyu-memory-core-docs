# STARTUP / SHUTDOWN MATRIX (verification proposal)

| Field | Value |
|---|---|
| Contract id | `STARTUP-SHUTDOWN-MATRIX` |
| Status | **PROPOSED** — verification output, not an approved design change |
| Basis | `03-m01.md` D32 (first init), D34 (process model, single instance), D28 (health), `03-系统运行与协作协议.md` §7.3.4 (health states) and §7.7 (worker), `05-可靠性与运维规范.md` §7.6 |
| Measured against | `03-verification/outbox/lib/lifecycle.cjs`, `startup-shutdown-test.cjs` |
| Evidence | `06-evidence/outbox.jsonl` ids `EV-OUTBOX-029…034` (see §8) |
| Implementation authorization | none. |

---

## 1. The distinction this matrix exists to protect

| Term | Meaning here | True when |
|---|---|---|
| **Liveness** | the process is up and answering | always, until the process exits — *including while not ready* |
| **Readiness** | every component a read depends on is usable | only after the whole start sequence succeeded |
| **Health** | the externally reported summary | `normal` (ready) / `degraded` (alive, not serving) / `error` (a component failed) |

Measured (`EV-OUTBOX-029`): before start, `liveness.alive=true` while `ready=false`, the read gate
returns `UNAVAILABLE`, and health reports `error`. After start, health is `normal` and the gate
passes. After shutdown, the gate refuses again.

**Rule: a read may only pass the gate.** Nothing may read authoritative data by opening the file
directly while the instance is starting or draining.

---

## 2. Start sequence (ordered, fail-fast)

Order enforced by construction (`STAGE_ORDER` in `lib/lifecycle.cjs`). A failing stage stops the
sequence; later stages never run.

| # | Stage | What it does | On failure | Verified |
|---|---|---|---|---|
| 1 | `lock` | take `<db>.lock`; **if the file exists, read its pid and probe liveness**; refuse if the holder is alive, take over if it is not | refuse to start; leave the live holder's file untouched | `EV-OUTBOX-031` |
| 2 | `open` | open SQLite with `busy_timeout`, `journal_mode=WAL`, `foreign_keys=ON` | refuse to start | `EV-OUTBOX-030` (stage ran) |
| 3 | `config` | validate the loaded configuration (including the `retry_max`/`dead_letter_after` conflict) | refuse to start (`CONFIGURATION_ERROR`) | `EV-OUTBOX-014`, `029` |
| 4 | `schema_version` | `PRAGMA user_version` must equal the supported version | refuse to start; **do not** attempt a migration here | `EV-OUTBOX-030` — a `user_version=999` file fails at this stage and never runs 5–8 |
| 5 | `integrity` | `PRAGMA quick_check` | refuse to start; a restore/repair is required | stage exercised on every start |
| 6 | `recover` | ensure the schema exists, drop stale worker heartbeats, **report** `processing` / `expired_leases` / `pending` | refuse to start only if the recovery reads fail | `EV-OUTBOX-032` |
| 7 | `worker` | build the worker's environment (connection + config + worker id) | refuse to start | `EV-OUTBOX-029` |
| 8 | `ready` | flip the readiness flag, open the read gate, begin accepting claims | — | `EV-OUTBOX-029` |

### 2.1 Why `recover` does not reclaim rows itself

Recovery **reports** stuck work; it does not steal it. A row in `processing` with a live lease may
belong to a worker that is still running (a second instance must not double-process it). The claim
predicate is what actually reclaims, and only after the lease expires.

Measured (`EV-OUTBOX-032`): with one live-lease and one expired-lease row, startup reports
`processing=2, expired_leases=1`, and the first claim takes exactly the expired one (attempt 2)
while the live-lease row stays untouched.

> A startup that "recovered" every `processing` row would be a duplicate generator. This is the
> single most important ordering constraint in this matrix.

### 2.2 Readiness gate

`assertReady()` throws `UNAVAILABLE` before stage 8 and after shutdown. The prototype makes this a
function call; the production form is an HTTP 503 on every read endpoint (the design's
`/api/v1/ health` surface is not implemented here).

---

## 3. Shutdown sequence (ordered, bounded)

Order: **`stop_accepting` → `drain_inflight` → `checkpoint` → `close` → `release_lock`**.

| # | Stage | What it does | Failure / timeout behaviour | Verified |
|---|---|---|---|---|
| 1 | `stop_accepting` | the claim path returns nothing from this instant; in-flight work continues | — | `EV-OUTBOX-033` |
| 2 | `drain_inflight` | wait until the in-flight count reaches 0, bounded by `graceful_shutdown_ms` | **on timeout report `drained=false`**; do not fake completion | `EV-OUTBOX-033` (drained), `EV-OUTBOX-034` (timeout) |
| 3 | `checkpoint` | `PRAGMA wal_checkpoint(TRUNCATE)` | a checkpoint failure is logged; the file is still closed | `EV-OUTBOX-033` (`wal_bytes=0` after close) |
| 4 | `close` | close the connection | — | `EV-OUTBOX-033` |
| 5 | `release_lock` | delete the lock file | a left-behind lock is taken over by the next start (stage 1) | `EV-OUTBOX-033` |

Measured drain (`EV-OUTBOX-033`): 1 in-flight event, `drained=true`, drain 60 ms, the event
completed exactly once (`attempts=1`, 2 effects), WAL truncated to 0 bytes, lock removed, and a
claim after shutdown returned 0 rows.

Measured grace expiry (`EV-OUTBOX-034`): grace 120 ms exceeded, `drained=false, remaining=1`, the
event left `processing` with `processed_at=NULL` and **0 effects**; the next instance saw
`expired_leases=1`, reclaimed it, applied both channels once, and reached `succeeded` on attempt 2.

**A drain timeout is not a shutdown failure — it is a reported incomplete drain.** The row must
remain claimable; marking it succeeded would be a fabricated recovery.

`graceful_shutdown_ms` default 30 000 (`storage.schema.json`) is **EXPERIMENTAL /
VALIDATION_PENDING**: no measurement of the worst-case in-flight processing time exists.

---

## 4. What must be true before the first read is served

Not negotiable, in this order:

1. The instance lock is held by this process (stage 1).
2. The schema version is the supported one (stage 4).
3. `quick_check` passed (stage 5).
4. If this start followed a restore: the newest delete and revocation state has been replayed, and
   the derived index has been rebuilt or is declared unavailable. **These are not automatic** — see
   `BACKUP-RESTORE-CONTRACT.md` §3/§4. A start that skips them serves resurrected or re-authorised
   content while reporting `normal`.
5. The worker is constructed (stage 7).

The prototype does not enforce item 4; it has no way to know whether a restore just happened. A
production implementation needs a "restore in progress" marker that blocks readiness, otherwise
the gate is a formality.

---

## 5. Failure behaviour per stage (what the operator sees)

| Stage | Symptom | Health | Safe action |
|---|---|---|---|
| `lock` (live holder) | refuses to start, exit non-zero, `UNAVAILABLE` | `error` | find the holder; do not delete the lock file — it is a real process |
| `lock` (stale file) | takes over silently, logs the takeover | `normal` | none |
| `open` | refuses to start | `error` | check path/permissions/disk |
| `config` | refuses to start with the failing key | `error` | fix config; the error names the key |
| `schema_version` | refuses to start | `error` | run the migration tool, then start |
| `integrity` | refuses to start | `error` | restore from backup (§4 of the backup contract) |
| `recover` | refuses to start | `error` | inspect the database manually |
| `ready` | refuses to serve | `error` | bug: the gate was reached without a usable component |
| drain timeout | starts anyway after the grace period, reports `drained=false` | `degraded` during the window | expect one reclaim per unfinished event |

`degraded` is not implemented as a distinct runtime state in the prototype (health returns
`normal`/`error` only). The design's three-state health is only partly realised — recorded as a gap
in §7.

---

## 6. Interaction with the outbox

- Startup must not reset `attempts`. A restart storm must not launder a poison event's attempt
  budget; only an explicit operator replay does that (`OUTBOX-CONTRACT.md` §7).
- Startup must not clear `claim_token` on `processing` rows. Doing so would let the reclaiming
  worker's CAS be confused with the original owner's.
- Shutdown must not touch `processing` rows it does not own. Only its own in-flight work is drained;
  everything else is left for lease expiry.

---

## 7. Gaps and unverified behaviour

1. **`degraded` health is not implemented.** Only `normal` / `error` exist in the prototype.
2. **No restore marker.** Item 4 of §4 is documentary only.
3. **No progress reporting.** A start that hangs in `integrity` on a large database reports nothing
   until it finishes; there is no watchdog.
4. **No OS file lock.** The lock is a pid-carrying JSON file plus a liveness probe. Two processes
   starting in the same millisecond could both pass the `exists` check before either writes. The
   window is not closed (no `O_EXCL` create was used in the prototype) — **NOT_VERIFIED**, and a
   real implementation should use an atomic exclusive create or an OS lock.
5. **`quick_check` only.** A full `integrity_check` is slower; the prototype uses `quick_check` at
   startup and `integrity_check` in tests. The design does not specify which.
6. **Grace period is wall-clock only.** A worker that ignores the drain and keeps writing after the
   grace period is not killed by the prototype.

---

## 8. Evidence map

| Test id | Evidence | What it establishes |
|---|---|---|
| `lifecycle.readiness_gate_blocks_reads_before_ready` | EV-OUTBOX-029 | readiness ≠ liveness; the gate refuses before ready and after shutdown |
| `lifecycle.startup_stage_order_and_short_circuit` | EV-OUTBOX-030 | stage order enforced; a bad schema version stops before integrity/recover/worker/ready |
| `lifecycle.instance_lock_live_vs_stale` | EV-OUTBOX-031 | live holder blocks start; dead holder is taken over; lock released on shutdown |
| `lifecycle.startup_recovery_reports_and_respects_leases` | EV-OUTBOX-032 | startup reports stuck work; only expired leases become claimable |
| `lifecycle.shutdown_drains_inflight_then_checkpoints` | EV-OUTBOX-033 | ordered shutdown, drain completes the in-flight event, WAL truncated, lock released |
| `lifecycle.shutdown_grace_expiry_leaves_event_reclaimable` | EV-OUTBOX-034 | drain timeout is reported, not faked; the event is reclaimed by the next start |

*(Evidence ids are assigned by append order; re-running `run-all.cjs` assigns a fresh contiguous
block. The test id is the stable key.)*
