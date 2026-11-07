# SQLITE-DRIVER-VERIFICATION — TECH-SQLITE-01

Date: 2026-11-05 (Asia/Shanghai)
Area: `03-verification/sqlite/`
Evidence ledger: `06-evidence/sqlite.jsonl` (55 records)
Raw logs: `06-evidence/raw/sqlite/`
Task status: **PARTIALLY_VERIFIED(file-backed persistence, WAL, transactions, locking, migration, backup/restore, crash recovery)** — driver not frozen.

---

## 1. Verification target

Determine, from executed evidence on this machine, whether `node:sqlite` is a fit authoritative
store for Memory Core R4.1 (M01), and decide whether the driver may be frozen.

Secondary objective: adjudicate the contradiction between two reports inside the design package,
and determine which prior claims were backed by real artefacts.

## 2. Environment actually used

| Item | Value | Source |
|---|---|---|
| Node.js | **v24.21.0** | `process.version` in every evidence record |
| SQLite | **3.53.4** | `select sqlite_version()` |
| FTS5 | enabled (`ENABLE_FTS5`, `ENABLE_FTS3`, `ENABLE_FTS3_PARENTHESIS`) | `pragma compile_options` |
| OS | Windows NT 10.0.26200, x64 | `os.type()`/`os.release()` |
| Driver | `node:sqlite` (builtin) | `require('node:sqlite')` |
| Package manager | **none** — npm, pnpm, npx, corepack, yarn all absent | probed from inside `driver-comparison.cjs` |
| Working dir | `E:\[PROJECT]\r41-baseline` | — |

Every test used a throwaway file database under `03-verification/sqlite/_scratch/`, deleted after
each suite. No real memory database was touched.

## 3. The report contradiction — resolved

Two documents in the package disagree:

- `SECOND-ROUND-FULL-AUDIT-REPORT.md §5` claims WAL, migration, backup/restore, lock contention and
  crash recovery all "passed".
- `TECHNICAL-VERIFICATION-REPORT.md` attributes only an in-memory smoke test to
  `sqlite-smoke-result.json` (that file records `"journal_mode": "memory"`, which is consistent).

`audit-prior-evidence.cjs` resolved this mechanically by reading the artefacts and the harness
sources (`EV-SQLITE-*` records `sqlite.prior_artifacts_exist` and
`sqlite.prior_claims_backed_by_artifacts`):

| Finding | Result |
|---|---|
| Do all three prior result artefacts exist? | **Yes** — `sqlite-smoke-result.json`, `sqlite-file-result.json`, `sqlite-recovery-result.json` |
| Are the 13 claimed checks backed by both a recorded value and real harness code? | **13 / 13** |
| Runtime that produced them | node **v24.22.1**; this machine runs **v24.21.0** |

**Adjudication.** The Technical Verification Report is the inaccurate one: it described only the
in-memory smoke artefact and omitted the two file-backed artefacts that do exist. The Full Audit
Report's claim list is broadly supported by artefacts — **except** for the one claim that matters
most, "crash recovery":

> the prior harness does **not** simulate a crash. Its child process runs
> `BEGIN; INSERT ...; process.exit(0)` — a *clean* exit, not an abrupt termination. It therefore
> proves that SQLite discards an uncommitted transaction on a normal close. It does **not** exercise
> abrupt termination, torn WAL state, or fsync ordering.
>
> The `child_exit` check is additionally a tautology: the child is scripted to exit 0, so asserting
> `child.status === 0` tests the script, not the database.

So neither report is simply right. The honest summary is: **most prior SQLite claims were backed by
real artefacts; the "crash recovery" claim was overstated; and none of it is reproducible on this
machine because the producing runtime (v24.22.1) is not installed here.**

## 4. What was executed in this session

Ten suites, 55 checks, **0 failing checks**, 50 `PASS` and 5 `PARTIAL_PASS`.

| Suite | Checks | Result |
|---|---:|---|
| `file-backed-smoke.cjs` | 10 | all PASS |
| `wal-test.cjs` | 6 | all PASS |
| `transaction-test.cjs` | 6 | all PASS |
| `concurrency-test.cjs` | 5 | all PASS |
| `locking-test.cjs` | 5 | all PASS |
| `migration-test.cjs` | 5 | all PASS |
| `backup-restore-test.cjs` | 6 | 4 PASS, 2 PARTIAL_PASS |
| `crash-recovery-test.cjs` | 5 | 4 PASS, 1 PARTIAL_PASS |
| `driver-comparison.cjs` | 3 | 2 PASS, 1 PARTIAL_PASS |
| `audit-prior-evidence.cjs` | 4 | 3 PASS, 1 PARTIAL_PASS |

Run entry point: `node 03-verification/sqlite/run-all.cjs`.
Exit code semantics: `0` all green · `1` not fully green · `2` harness error. This run exits **1**
because 5 checks are honestly `PARTIAL_PASS`, not because anything failed.

### 4.1 Capabilities established

| Capability | Evidence | Measured result |
|---|---|---|
| File-backed database | `sqlite.file_database_creation` | real file on disk; not in-memory |
| Close/reopen persistence | `sqlite.close_and_reopen_persistence` | rows and content survived |
| WAL is real | `sqlite.wal_enabled`, `sqlite.wal_sidecars_during_write` | `journal_mode=wal`; `-wal`/`-shm` observed on disk |
| WAL persists across reopen | `sqlite.wal_persists_across_reopen` | stays `wal` without re-requesting |
| Checkpoint | `sqlite.wal_checkpoint_truncate` | TRUNCATE folds WAL into main file, no rows lost |
| Commit / rollback | `sqlite.transaction_atomic_transfer`, `sqlite.transaction_rollback_all_statements` | multi-statement atomicity; rollback reverts **all** statements |
| Constraint failure reverts txn | `sqlite.transaction_constraint_failure_reverts` | CHECK violation leaves no partial effect |
| FK / CHECK / UNIQUE | `sqlite.foreign_key_violation_rejected`, `sqlite.check_constraint_rejected`, `sqlite.unique_constraint_rejected` | all rejected; FK requires `pragma foreign_keys=on` per connection (verified) |
| Multi-connection | `sqlite.multi_connection_read` | two handles on one file see the same committed state |
| Write contention | `sqlite.lock_contention_writer_blocked` | errcode **5** (`SQLITE_BUSY`), never silent interleaving |
| Reader during writer | `sqlite.wal_reader_during_writer` | reader gets pre-commit snapshot without error |
| Cross-process lock + busy_timeout | `sqlite.cross_process_long_transaction_busy_timeout` | child handshake proved lock held; waiter waited **955 ms** then errcode 5 |
| Deferred-upgrade deadlock | `sqlite.locking_deadlock_resolved_by_failure` | second escalation fails instead of deadlocking |
| Migration | `sqlite.migration_forward` | versioned 0→3 via `user_version` |
| Migration failure recovery | `sqlite.migration_failure_recovery` | failed batch rolled back; `user_version` stayed 2; no partial table |
| Backup | `sqlite.backup_creates_artifact` | awaited `backup()` → 32 768-byte artefact, sha256 recorded |
| Restore | `sqlite.restore_content_equality` | content byte-identical, `integrity_check=ok` |
| FTS rebuild after restore | `sqlite.fts_rebuild_after_restore` | rebuild idempotent; English 1 hit, whole-CJK-run 1 hit |
| Crash: uncommitted | `sqlite.crash_recovery_uncommitted_rollback` | **SIGKILL** mid-transaction → 0 of 25 rows survived, integrity ok |
| Crash: committed | `sqlite.crash_immediately_after_commit` | kill racing the flush → **15 of 15** committed rows survived, integrity ok |
| Post-crash consistency | `sqlite.post_recovery_consistency` | integrity ok, 0 FK violations |

The crash-recovery checks here are materially stronger than the prior suite: a real `SIGKILL` was
issued after the child reported the write lock was held, and the kill-after-commit case races the
flush rather than assuming it.

## 5. Two driver behaviours found that the adapter contract must handle

1. **Unawaited `backup()` silently produces a 0-byte file.** `sqlite.backup(db, path)` returns a
   Promise. Called without `await`, it returns immediately and leaves an empty artefact. A backup
   routine that ignores the return value will "succeed" while producing nothing usable. Measured:
   unawaited → 0 bytes; awaited → 32 768 bytes.

2. **Backup on a connection holding its own open transaction fails with misleading diagnostics.**
   The call throws `ERR_SQLITE_ERROR` with `errcode 0` and message **"not an error"**. The failure is
   real but the error text is useless for diagnosis. Backing up from a *separate* connection works
   correctly and excludes uncommitted rows (`sqlite.backup_during_write_transaction`: 0 leaked rows).

Consequence for the `DatabaseAdapter` contract: it must expose an explicit
`backup(target)` that (a) awaits the driver promise, (b) refuses or routes around a connection with
an open transaction, and (c) validates the artefact (non-zero size + `integrity_check`) before
reporting success. This is now a written requirement, not a suggestion.

## 6. Driver comparison — bounded, and honestly so

A comparison was **not** possible. `driver-comparison.cjs` probes for package managers from inside
the process and found none (`npm`, `pnpm`, `npx`, `corepack`, `yarn` all absent), and a full-disk
search for `npm-cli.js` found nothing. No third-party driver can be installed.

`03-verification/sqlite/driver-matrix.json` therefore records:

| Candidate | Status | Basis |
|---|---|---|
| `node:sqlite` | INSTALLED_AND_TESTED | 13 capabilities measured this session |
| `better-sqlite3` | BLOCKED_NOT_INSTALLABLE | no package manager; needs native build toolchain |
| `node-sqlite3` | BLOCKED_NOT_INSTALLABLE | no package manager; needs native build toolchain |
| `sql.js`/WASM | NOT_SELECTED_BY_DESIGN | in-memory/WASM form does not satisfy file-backed authority |

**No ranking is produced.** Comparing one measured candidate against three unmeasured ones would be
a fabrication. The matrix records what lifts the constraint: install a package manager (or vendor the
driver package), provide a native build toolchain on Windows x64, then re-run *this same harness*
against each driver so the comparison is like-for-like.

## 7. Recommendation

**Recommended driver: `node:sqlite`.**

Selection basis — note carefully that this is *not* a comparative-performance argument:

1. It is the only driver installable in this environment, and it satisfies every functional
   requirement the design places on the authoritative store.
2. Zero third-party dependency: no native compilation, no lockfile churn, no Windows binary
   distribution risk. For a local single-instance deployment this is the dominant maintenance
   consideration.
3. Its failure modes are explicit and testable (`SQLITE_BUSY` errcode 5, `integrity_check`,
   `user_version`, `isTransaction`), which lets the adapter be defensive rather than hopeful.

**Abandoned / deferred options and why:** `better-sqlite3` and `node-sqlite3` are not rejected on
technical grounds — they were untestable here. Their comparison remains an open gate, not a
conclusion. `sql.js` is rejected on design grounds (not file-backed).

**Residual risks:**
- Driver availability is tied to the Node version; a downgrade below the `node:sqlite` support
  boundary removes the driver. Pin the Node version in the deployment contract.
- The two backup gotchas above must be encoded in the adapter or they will produce silent
  backup failures in production.
- Cross-process contention was measured on one machine under no other load; the retry/backoff policy
  is the adapter's responsibility and is not yet specified with evidence-backed numbers.
- Node v24.22.1 artefacts from the prior round are not reproducible here (v24.21.0), so those
  results should be treated as unverified provenance until re-run.

## 8. Freeze decision

**Driver status: `PARTIALLY_VERIFIED(file-backed persistence, WAL, transactions, locking, migration, backup/restore, crash recovery)` — NOT `FROZEN`.**

Freezing is blocked on exactly these gates:

| Gate | Why it blocks freeze | How to close |
|---|---|---|
| G-1 Comparative driver evidence | The comparison could not be run; selection would rest on adequacy alone | Install a package manager + native toolchain; run this harness against ≥1 alternative |
| G-2 Reproducible runtime | Prior artefacts came from v24.22.1, absent here | Pin Node version; re-run prior suites under the pinned version |
| G-3 Power-loss durability | SIGKILL is not power loss; fsync ordering untested | Fault-injection or `PRAGMA fullfsync` study on the target hardware |
| G-4 Long-run soak | No sustained multi-hour contention/checkpoint load | Soak test with concurrent writers over hours |

None of these invalidate the functional conclusion. They are the difference between
"verified within a stated scope" and "frozen".

## 9. Unresolved / not executed

- Network-filesystem behaviour (WAL is documented as unsafe there) — not tested.
- Savepoints and nested transactions — the driver exposes no transaction helper; not exercised.
- `enableLoadExtension` / extension loading path — not exercised; no extensions available.
- Encryption at rest (`BILI_ENCRYPTION_KEY` analogy) — out of scope for the driver.
- Performance benchmarking (throughput, p95 latency under sustained load) — not measured; no
  thresholds invented.

## 10. Files delivered

| Path | Purpose |
|---|---|
| `03-verification/sqlite/run-all.cjs` | entry point, 10 suites |
| `03-verification/sqlite/file-backed-smoke.cjs` | creation, reopen, WAL, FK/CHECK/UNIQUE, commit/rollback |
| `03-verification/sqlite/wal-test.cjs` | WAL observability, checkpoint, sidecars |
| `03-verification/sqlite/transaction-test.cjs` | atomicity, rollback scope, constraint failure |
| `03-verification/sqlite/concurrency-test.cjs` | multi-connection, contention, cross-process busy_timeout |
| `03-verification/sqlite/locking-test.cjs` | deferred upgrade, deadlock, busy_timeout timing |
| `03-verification/sqlite/migration-test.cjs` | versioned migration, failure recovery, rollback point |
| `03-verification/sqlite/backup-restore-test.cjs` | backup/restore, derived-index rebuild, two gotchas |
| `03-verification/sqlite/crash-recovery-test.cjs` | SIGKILL mid-transaction, kill racing commit |
| `03-verification/sqlite/driver-comparison.cjs` | bounded comparison + `driver-matrix.json` |
| `03-verification/sqlite/audit-prior-evidence.cjs` | contradiction adjudication |
| `03-verification/sqlite/_lib/` | evidence emitter, DB helpers, lock child process |
| `06-evidence/sqlite.jsonl` | 55 structured evidence records |
| `06-evidence/raw/sqlite/*.log` | full stdout/stderr per suite |

Reproduce with:

```
cd E:\[PROJECT]\r41-baseline
node 03-verification\sqlite\run-all.cjs
```
