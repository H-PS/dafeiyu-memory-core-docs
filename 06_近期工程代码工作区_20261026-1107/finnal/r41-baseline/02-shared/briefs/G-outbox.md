# BRIEF — Agent G · Outbox, backup/restore, startup/shutdown, recovery

Read first:
1. `02-shared\00-PARENT-MANDATE.md`
2. `02-shared\EVIDENCE-FORMAT.md`
3. `01-baseline-corpus\design-package\03-系统运行与协作协议.md` (transactional outbox, idempotency, 16 error codes)
4. `01-baseline-corpus\design-package\05-可靠性与运维规范.md` (failure matrix, backup six steps, recovery ten steps)
5. `contracts\schemas\outbox-event.schema.json`, `contracts\config\worker.schema.json`

Own: `03-verification\outbox\` and `04-contracts\` operations documents.
Evidence: `06-evidence\outbox.jsonl`; raw: `06-evidence\raw\outbox\`.

## Scope

Transactional outbox: enqueue in the same transaction as the authoritative write; lease; retry;
backoff; dead letter; idempotency; concurrent workers; crash recovery; manual recovery.
Backup/restore: what is authoritative, what is rebuildable, retention, verification of a restore.
Startup/shutdown: exact ordering, what must be ready before accepting reads, what must drain before exit.

## Code that must actually run (SQLite, `node:sqlite`, real file DB)

- `outbox-simulator.cjs` — outbox table with `state`, `attempts`, `lease_owner`, `lease_until`,
  `next_attempt_at`, `dead_lettered_at`, plus a claimed-worker loop.
- `outbox-tests.cjs` covering at minimum: normal processing; **duplicate delivery is idempotent**
  (same idempotency key twice produces one effect); partial failure does not mark success;
  retry with backoff actually backs off (measure the gap); **lease expiry** lets another worker
  reclaim the row; **two concurrent workers must not both process the same row** (this is the
  interesting race — use two real connections and assert exactly-once via a claim token);
  dead-letter after the configured attempt ceiling; replay from dead letter; crash mid-processing
  (kill a worker, verify the row is reclaimed, not lost and not double-applied).
- `crash-recovery-test.cjs` — kill the process between enqueue and effect, reopen, prove recovery
  is exactly-once.
- `backup-restore-test.cjs` — use `node:sqlite`'s `backup` API (or `VACUUM INTO`, say which),
  verify integrity and content equality, and verify what must be *rebuilt* rather than restored
  (FTS/vector indexes are derived, so a restore must be followed by a rebuild — test that).
- `run-all.cjs`, `README.md`, seeded fixtures with a recorded hash.

Concurrency assertions must use genuinely separate connections or child processes. A "concurrent"
test that is actually sequential is not acceptable and the parent will check.

## Deliverables (documents)

- `04-contracts\OUTBOX-CONTRACT.md` — states, transitions, lease semantics, backoff formula,
  dead-letter threshold, idempotency key definition, exactly-once claim mechanism, manual recovery
  procedure. Any numeric default that is not derived from evidence must be labelled
  `EXPERIMENTAL` / `VALIDATION_PENDING`, not presented as settled (MC-AUD-008).
- `04-contracts\BACKUP-RESTORE-CONTRACT.md` — with retention explicitly left as an **open decision**
  (MC-AUD-009): do not invent a retention period. State the options and what a decision must consider.
- `04-contracts\STARTUP-SHUTDOWN-MATRIX.md` — ordered steps, readiness/liveness distinction,
  failure behaviour at each step, and the shutdown drain order.

## Discipline

- Report measured backoff gaps and attempt counts, not assertions about them.
- If exactly-once cannot be proven with the mechanism you chose, say so plainly and describe the
  exact window where a duplicate could occur. That is a finding, not a failure of the task.
- `03-verification\outbox\TASK-RESULT.md` with all required sections.

Parent will re-run `run-all.cjs`.
