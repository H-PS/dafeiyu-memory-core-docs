# SQLite verification area — README

Isolated verification harness for `node:sqlite` as the Memory Core R4.1 authoritative store.
**This is verification code, not product code.** Nothing here is imported by M01 or any product module.

## Run

```
cd E:\[PROJECT]\r41-baseline
node 03-verification\sqlite\run-all.cjs
```

Exit codes: `0` all checks green · `1` not fully green (includes `PARTIAL_PASS`) · `2` harness error.

Each suite can also be run alone, e.g. `node 03-verification\sqlite\concurrency-test.cjs`.

## Outputs

- `06-evidence/sqlite.jsonl` — one JSON record per executed check (schema in
  `../../02-shared/EVIDENCE-FORMAT.md`).
- `06-evidence/raw/sqlite/<suite>.log` — full stdout/stderr per suite.
- `03-verification/sqlite/driver-matrix.json` — capability matrix with honest per-candidate status.

## Requirements

- Node.js with builtin `node:sqlite` (verified on v24.21.0). No npm packages, no build step.
- Windows or any OS that supports SQLite file locking. All results here are Windows x64.

## Scratch data

Every test creates its database under `03-verification/sqlite/_scratch/`, which is deleted per suite.
Nothing writes outside this directory. No real memory database is ever opened.

**Note:** scratch files live inside the area, not in the system temp directory — on this machine
`fs.rmSync` over `%TEMP%` paths hit `EPERM`, so the OS temp dir is deliberately avoided.

## Coverage map

| Suite | Establishes |
|---|---|
| `file-backed-smoke.cjs` | file-backed creation, reopen, WAL, FK/CHECK/UNIQUE, commit, rollback, integrity |
| `wal-test.cjs` | journal mode, WAL persistence, checkpoint, sidecars |
| `transaction-test.cjs` | multi-statement atomicity, rollback scope, constraint failure, autocommit |
| `concurrency-test.cjs` | multi-connection, write contention, reader-during-writer, cross-process lock + busy_timeout, killed-holder rollback |
| `locking-test.cjs` | deferred upgrade, deadlock resolution, busy_timeout timing |
| `migration-test.cjs` | versioned migration, failure recovery, pre-migration backup, restore point |
| `backup-restore-test.cjs` | backup artefact, restore equality, FTS rebuild, backup during write txn, same-connection gotcha |
| `crash-recovery-test.cjs` | SIGKILL mid-transaction, kill racing commit, post-recovery consistency |
| `driver-comparison.cjs` | what drivers are actually obtainable; capability matrix |
| `audit-prior-evidence.cjs` | adjudicates the two contradictory prior reports |

## Deliberate non-coverage

No network filesystems, no savepoints, no extension loading, no encryption, no performance
benchmarks. These are listed as unexecuted in `TASK-RESULT.md` §9 rather than silently omitted.
