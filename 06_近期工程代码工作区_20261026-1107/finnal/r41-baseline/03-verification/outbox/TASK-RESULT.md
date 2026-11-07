# TASK-RESULT — Agent G · outbox, backup/restore, startup/shutdown, recovery

> **Section note.** The mandate (`00-PARENT-MANDATE.md` §7) points at `02-shared\TASK-ASSIGNMENT.md`
> for the mandatory section list; **that file does not exist** in this workspace (verified:
> `02-shared\` contains only `00-PARENT-MANDATE.md`, `EVIDENCE-FORMAT.md` and `briefs\`). The
> sections below are therefore the brief's scope plus the mandate's reporting/evidence rules. If the
> parent has a different list, this file is one section list short and should be amended.

## 1. Status

**DELIVERED — 4 scripts, 34 checks, 34 PASS, 0 FAIL.** Every check really executed; there is no
placeholder result and no skipped assertion reported as a pass.

- Area code: `03-verification\outbox\`
- Contracts: `04-contracts\OUTBOX-CONTRACT.md`, `BACKUP-RESTORE-CONTRACT.md`, `STARTUP-SHUTDOWN-MATRIX.md`
- Evidence: `06-evidence\outbox.jsonl` (`EV-OUTBOX-001…034`, one coherent run)
- Raw logs: `06-evidence\raw\outbox\` (one `*.run1.log` per script plus `run-all.summary.json`)

Authorization: `CODE_IMPLEMENTATION_SCOPE` = isolated verification code only.
`PRODUCT_IMPLEMENTATION_AUTHORIZATION` = **NOT_AUTHORIZED**. No product file was modified;
`01-baseline-corpus\` was read only.

## 2. What actually ran

```
cd E:\[PROJECT]\r41-baseline\03-verification\outbox
node run-all.cjs
```

| Script | Checks | Result | Wall clock |
|---|---|---|---|
| `outbox-tests.cjs` | 15 | 15 PASS | 10 185 ms |
| `crash-recovery-test.cjs` | 6 | 6 PASS | 7 750 ms |
| `backup-restore-test.cjs` | 7 | 7 PASS | 1 516 ms |
| `startup-shutdown-test.cjs` | 6 | 6 PASS | 2 160 ms |
| **total** | **34** | **34 PASS** | ≈ 22 s |

Exit code `0`. `run-all.summary.json` records the same run's environment facts, per-script exit
codes and evidence deltas. The suite was executed twice back to back (34/34 both times, the second
block appended rather than overwriting the first), then once more from a deleted evidence file —
that third run is the single `EV-OUTBOX-001…034` block in `06-evidence\outbox.jsonl`. Each per-test
scratch directory is wiped before use, so the tests are re-runnable.

Environment: Node v24.21.0, SQLite 3.53.4, `node:sqlite` only, real file databases under
`_scratch\`, no npm/pnpm, no build step.

## 3. The headline finding: exactly-once holds, but not everywhere

**Exactly-once per channel holds** when the effect and its journal entry live in the same SQLite
transaction as the record that says "this channel is done". Measured: kill after the first channel
committed → journal `fts=applied`, recovery applied only `vec`, final `effects fts=1 vec=1`,
status `succeeded` (`EV-OUTBOX-019`). With two real OS processes racing the same backlog, 6 events
were split 3/3 with **0 duplicate effect groups** (`EV-OUTBOX-009`).

**Exactly-once does not hold across a store that cannot join the authority transaction** (LanceDB).
Two windows, both reproduced, not argued:

| Window | Sequence | Measured |
|---|---|---|
| **W1** crash between the external commit and the journal commit | external write committed → kill → recovery re-applies the channel | external effect log **1 → 2**, keyed index row stays **1** (`EV-OUTBOX-020`) |
| **W2** a live worker whose lease expires mid-effect | worker A commits externally, stalls past the lease, B reclaims and completes, A wakes and is fenced | A fenced (`true`), claim token changed, external effects `fts=2 vec=1`, index rows `1` (`EV-OUTBOX-021`) |

So the honest statement is: **at-least-once per channel across the index boundary, exactly-once for
effects inside the authority transaction.** The duplicate is harmless only because the index write
is an upsert keyed by `(aggregate_id, aggregate_version)`; it would be harmful for any append,
increment or externally visible side effect. This is finding **O-F4** in
`04-contracts\OUTBOX-CONTRACT.md` §6.3, and the design text ("事件处理 MUST 幂等") should say
"idempotent effect", not "executed once".

## 4. Measured behaviour (not asserted)

| Property | Measurement | Evidence |
|---|---|---|
| lease grant | `lease_until - claimed_at = 249 ms` for `claim_timeout_ms = 250` | EV-OUTBOX-008 |
| lease reclaim | another worker reclaimed **414 ms** after the claim; the stale worker's terminal CAS changed **0** rows | EV-OUTBOX-008 |
| backoff gaps | attempts 2/3/4 → **163 / 325 / 608 ms** against expected 150 / 300 / 600 | EV-OUTBOX-007 |
| concurrency | 2 separate `node` processes, barrier skew **0 ms**, claim windows overlapped, 3/3 split, **0** duplicates | EV-OUTBOX-009 |
| dead letter | `attempts=4 > dead_letter_after=3` → `dead_letter`, next attempt 400 ms later produced none | EV-OUTBOX-010 |
| replay | journal preserved: already-applied channel 1 → 1, failed channel 0 → 1, 1 audit row | EV-OUTBOX-011 |
| crash → reclaim | killed after claim, lease expired by 177 ms, reclaimed on attempt 2, `fts/vec = 1/1` | EV-OUTBOX-018 |
| backup copy | 29 pages / 5 ms, `integrity_check=ok`, 8/8 tables equal, source SHA-256 unchanged | EV-OUTBOX-022 |
| backup under load | 1 720 pages / 367 ms while **32 commits** landed, **0 torn memory/outbox pairs** | EV-OUTBOX-023 |
| `VACUUM INTO` cross-check | 8/8 tables equal to the API copy | EV-OUTBOX-024 |
| FTS rebuild | `fts_docs` rebuilt 11 rows = 11 active memories; tombstoned content absent before and after | EV-OUTBOX-025 |
| startup recovery | reports `processing=2, expired_leases=1`; claims exactly the expired one (attempt 2) | EV-OUTBOX-032 |
| shutdown drain | in-flight event completed (`attempts=1`, 2 effects), WAL truncated to **0 bytes**, lock released, 0 post-shutdown claims | EV-OUTBOX-033 |
| drain timeout | `drained=false, remaining=1`, `processed_at=NULL`, 0 effects → next start reclaims and completes on attempt 2 | EV-OUTBOX-034 |

## 5. Two restore hazards, reproduced and closed

1. **Deletion resurrection.** Restoring a backup taken before a delete brings the memory back as
   `active` with 0 tombstones in the copy. Replaying the newer tombstone set returns it to
   `deleted` and removes it from the rebuilt index (`EV-OUTBOX-026`).
2. **Re-authorization.** Restoring a backup taken while a memory was `granted` restores it as
   `granted` after it was revoked, with 0 revocation rows and a readable count one too high.
   Replaying the newer revocations fixes it (`EV-OUTBOX-027`).

Both mean a restore is not finished until the newest delete/revocation state is applied, and reads
must be refused in between. **The design does not say where that newest state lives if the whole
data directory is lost** — see decision D-3.

## 6. Contract gaps found (carried, not resolved)

| Id | Gap | Evidence |
|---|---|---|
| O-F1 | `outbox-event.schema.json` is `additionalProperties:false` and has **none** of `idempotency_key`, `lease_owner`, `lease_until`, `claim_token`, `dead_lettered_at`. The lease/claim mechanism the design mandates cannot be expressed by the published schema; a conforming validator rejects a row this prototype writes. | EV-OUTBOX-015 |
| O-F2 | `storage.schema.json` defines two keys for one ceiling (`retry_max`, `dead_letter_after`), both defaulting to 8. The loader refuses a disagreement rather than picking one. | EV-OUTBOX-014 |
| O-F3 | SM-4 has a `cancelled` transition; the schema `status` enum does not contain `cancelled`, so such an event is schema-invalid. | EV-OUTBOX-015 |
| O-F4 | Exactly-once does not hold across the index boundary (windows W1/W2). | EV-OUTBOX-020, 021 |
| O-F5 | Brief says `next_attempt_at`; design/schema say `next_retry_at`. The prototype uses the brief's name and treats `next_retry_at` as the contract-facing name. | n/a |

## 7. What is NOT proven (stated, not hidden)

- **LanceDB was never touched.** It is absent from this environment (parent mandate §3). The vector
  channel is a second SQLite file; the "external store" claims are claims about that stand-in. Real
  LanceDB backup/rebuild is **NOT_VERIFIED**.
- **No duplicate *detector*.** Duplicates are counted by the prototype's own effect log. There is no
  production-side reconciliation, alert or threshold.
- **Backup failure injection** (disk full, corrupt backup file, rebuild failing midway) was not
  exercised.
- **OS-level file lock not used.** The instance lock is a pid-carrying file plus a liveness probe;
  two processes starting in the same millisecond could both pass the "file exists" check. Window
  not closed — recorded as a gap in `STARTUP-SHUTDOWN-MATRIX.md` §7.4.
- **`degraded` health** is not implemented (only `normal` / `error`).
- **No restore marker**, so the startup gate cannot know a restore is pending; §4 of the matrix is
  documentary.
- **`stale_event_age_ms` alarm** is unimplemented; the startup report is only the observable it
  would need.
- Timings are from a synthetic database (29–1 720 pages). They are not performance figures.

## 8. Decisions needed from the parent

| Id | Decision | Why it blocks |
|---|---|---|
| **D-1** | Any numeric for `claim_timeout_ms`, backoff base/factor/cap, `dead_letter_after`, `max_events_per_tick` (**MC-AUD-008**) | every value used here is `EXPERIMENTAL / VALIDATION_PENDING`; a real one needs a measured p99 processing time per channel and a stated recovery-latency target, neither of which exists |
| **D-2** | Backup retention (**MC-AUD-009**) | **no value was invented.** `storage.schema.json` has no retention key (verified, `EV-OUTBOX-028`); `OI-05-05` and `03-m01` §9 item 11 both say TBD; the 365-day figure in `PENDING-USER-DECISIONS.md` OI-012 is **audit-log** retention (PD-013), not backups. `BACKUP-RESTORE-CONTRACT.md` §6 lists options R-1…R-4 and what a decision must consider |
| **D-3** | Where the newest delete/revocation state lives when the whole data directory is lost | without it, a restore provably resurrects deleted content and re-grants revoked access (§5) |
| **D-4** | Whether the outbox schema is extended with the five claim/lease fields or the mechanism moves to a side table (O-F1) | either choice changes a published contract |

## 9. Reproduction

```
cd E:\[PROJECT]\r41-baseline\03-verification\outbox
node run-all.cjs          :: 0 = all pass; appends 34 evidence records
node outbox-tests.cjs     :: 15 checks, ~10 s
node crash-recovery-test.cjs   :: 6 checks, ~8 s
node backup-restore-test.cjs   :: 7 checks, ~1 s
node startup-shutdown-test.cjs :: 6 checks, ~2 s
```

Fixtures are verified on load: `_data\outbox-fixtures.json`, sha256
`b8c143d56e5833cde2ccf77983280c09acd91c39d6fe6c62ba50af8c641b8db5`, seed `20261105`
(12 memories / 11 active / 1 deleted / 1 revoked / 12 events). A fixture edit without updating the
recorded hash aborts the run instead of quietly changing what the tests mean.

`_scratch\` is disposable and recreated per test.
