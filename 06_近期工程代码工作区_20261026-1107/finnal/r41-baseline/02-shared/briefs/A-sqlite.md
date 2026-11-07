# BRIEF — Agent A · TECH-SQLITE-01 · SQLite driver and persistence reliability

Read first, in this order:
1. `E:\[PROJECT]\r41-baseline\02-shared\00-PARENT-MANDATE.md`
2. `E:\[PROJECT]\r41-baseline\02-shared\EVIDENCE-FORMAT.md`
3. `E:\[PROJECT]\r41-baseline\01-baseline-corpus\design-package\01-记忆本体与数据模型.md` (SQLite DDL chapter)
4. `C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design\evidence\second-round\` (the prior results you must audit)
5. `E:\[PROJECT]\r41-baseline\01-baseline-corpus\design-package\SECOND-ROUND-FULL-AUDIT-REPORT.md` and `TECHNICAL-VERIFICATION-REPORT.md`

Own this area only: `E:\[PROJECT]\r41-baseline\03-verification\sqlite\`
Append evidence to `E:\[PROJECT]\r41-baseline\06-evidence\sqlite.jsonl`; raw logs to `06-evidence\raw\sqlite\`.
Do not edit any file outside your area except those two evidence targets.

## The contradiction you must resolve first

The package contains two reports that disagree about the same work:

- `SECOND-ROUND-FULL-AUDIT-REPORT.md §5` claims: "File DB WAL, DDL, FK, CHECK, unique, rollback, reopen,
  migration smoke, VACUUM INTO backup/restore, lock contention and child-process crash recovery passed."
- `TECHNICAL-VERIFICATION-REPORT.md` (row "SQLite runtime") attributes only in-memory DDL/FK/CHECK/
  rollback to `sqlite-smoke-result.json`, whose own content confirms `"journal_mode": "memory"`.

Read the actual result files and the actual scripts. Then answer, in writing, with file paths and
line references: which claims are backed by a real artefact, which are backed only by an
assertion in prose, and which are unsupported. Do not accept either report at face value.

Also check and report: every `evidence/second-round/*.json` claims `"node": "v24.22.1"`, but this
machine runs v24.21.0 (parent-verified). Decide and state what that means for reproducibility.

## Driver comparison — bounded by a hard environment fact

There is **no npm, pnpm, npx or corepack on this machine** (parent-verified, full-disk search).
You cannot install `better-sqlite3` or any other driver. Therefore:

- Do **not** simulate, mock, or table-fill a comparison against an uninstallable driver.
- The installable candidate set is exactly: `node:sqlite` (builtin).
- Produce a comparison table where every non-installable candidate carries
  `status: NOT_APPLICABLE` or `BLOCKED` with the concrete reason (`no package manager available`)
  and a one-line statement of what would be required to lift the constraint (a package manager,
  a toolchain for native builds, and a pinned driver version).
- Write this up as an environment constraint, not as a comparison result. The parent will report
  it as such.

## Must execute (file-backed, real files, isolated scratch DBs)

Required coverage, one and only one evidence record per check:

1. file database creation
2. close and reopen persistence
3. WAL enabled and observable (`pragma journal_mode`, `-wal`/`-shm` sidecar presence)
4. transaction commit
5. transaction rollback (and that rollback really reverts)
6. FK enforcement (needs `pragma foreign_keys=ON` — assert it, do not assume)
7. CHECK constraint
8. UNIQUE constraint
9. multi-connection access (two `DatabaseSync` handles on the same file)
10. lock contention (two writers, measure what actually happens: exception type, `SQLITE_BUSY`, retry)
11. long transaction (hold a write txn while a second connection reads/writes)
12. migration (apply a version step, verify `user_version`/schema change)
13. migration failure recovery (fail mid-migration, verify the DB is not left half-migrated)
14. backup (`node:sqlite` exposes `backup`; use it, or `VACUUM INTO` if `backup` misbehaves — say which)
15. restore (open the backup, verify content equality)
16. child-process abnormal exit recovery (spawn a child, kill it mid-write, reopen, run `pragma integrity_check`)
17. data consistency check after all of the above

`pragma integrity_check` must appear in at least the restore and crash-recovery records.

## Also required

- A `driver-comparison.cjs` that emits the (bounded) comparison table, honestly.
- `run-all.cjs` executing every script, with a machine-readable summary and a truthful exit code.
- A `README.md` with exact run commands.
- Explicit answer on: **recommend freezing the driver?** If the evidence is not sufficient,
  the status is `VALIDATION_PENDING` and you must say exactly which gate is missing.
- Report any **failure** you find plainly. A failed check is a valid and valuable deliverable.
  Never adjust an expectation to make a check pass.

## Deliverable

`03-verification\sqlite\TASK-RESULT.md` with: target, what you actually inspected (paths),
files created/modified, commands executed verbatim, per-test result table, evidence ids, failures
and their causes, what you did not run and why, recommended driver, confidence level,
freeze recommendation, impact on other modules, and anything the parent must decide.

Parent will re-run `run-all.cjs` and will reject non-reproducing claims.
