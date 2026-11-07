# BACKUP / RESTORE CONTRACT (verification proposal)

| Field | Value |
|---|---|
| Contract id | `BACKUP-RESTORE-CONTRACT` |
| Status | **PROPOSED** — verification output, not an approved design change |
| Basis | `01-baseline-corpus/design-package/05-可靠性与运维规范.md` §7.3, `03-m01.md` D8 (§5.1) / D9 / D10, `contracts/config/storage.schema.json` |
| Measured against | `03-verification/outbox/backup-restore-test.cjs` |
| Evidence | `06-evidence/outbox.jsonl` ids `EV-OUTBOX-022…028` (see §9) |
| Retention | **OPEN DECISION — MC-AUD-009.** No period, no count, no schedule is stated in this document. See §6. |
| Implementation authorization | none. |

---

## 1. What is authoritative and what is rebuildable

| Artefact | Classification | Must be in the backup? | Can it be rebuilt? | Measured |
|---|---|---|---|---|
| `memory.sqlite` — memory rows, versions, permissions, delete state, outbox, tasks/checkpoints | **Authority** | yes | no | `EV-OUTBOX-022` |
| FTS5 full-text index (same file) | derived | may be included, but must not be *trusted* after a restore | yes, from the authority | `EV-OUTBOX-025` |
| LanceDB vector index (separate store) | derived | optional ("可以备份，但必须可重建", D8) | yes, from the authority + embeddings | not exercised (no LanceDB authorised here; see §8) |
| Recall cache / `cache.sqlite` | **must not be backed up** ("Recall Cache 不备份", D8) | no | n/a | not exercised |
| Runtime logs | operational | only as far as the retention decision says | no | n/a |

The classification is the load-bearing part. A backup that contains the index but is restored
without verification lets a corrupt or stale index masquerade as authoritative.

---

## 2. How a backup is taken

**Method: `node:sqlite`'s `backup()` API** — the design's "SQLite Backup API 或一致性快照，不直接
复制正在运行的数据库文件" (D8). Direct file copy of a live database is forbidden and was not used.

`backup()` returns a Promise, so in this harness it runs in its own process
(`lib/backup-child.cjs`): a real second connection, which is also how an operator's backup
process would run. `VACUUM INTO` was used as an independent cross-check of content equality, not
as the primary method.

Verified properties:

| Property | Measured | Evidence |
|---|---|---|
| copy is readable and integrity-clean | `integrity_check=ok` | EV-OUTBOX-022 |
| content equality with the source | 8/8 compared tables identical by content hash (memories, memory_versions, outbox, outbox_channel_state, tombstones, permission_revocations, audit_log, effects) | EV-OUTBOX-022 |
| source not mutated by taking the backup | source file SHA-256 unchanged | EV-OUTBOX-022 |
| backup under a concurrent writer | snapshot at one point in time; memory count within `[start,end]`; **0 torn memory/outbox pairs** | EV-OUTBOX-023 |
| `VACUUM INTO` agrees with the API copy | 8/8 tables equal | EV-OUTBOX-024 |

Copy sizes/durations observed (synthetic, not a performance claim): 29 pages / 5 ms for the fixture
database; 1 720 pages / 367 ms for a database padded with 1 500 × 4 KiB blobs while 32 commits
landed during the copy.

**Torn-pair check** is the important assertion: the outbox's whole premise is that a memory row and
its event are committed together. A backup taken mid-transaction must never contain one without
the other. It did not (`torn_pairs=0`, EV-OUTBOX-023).

---

## 3. What a restore must verify, in order

Restoring is not "copy the file back". The design's order (`03-m01` D8, "恢复顺序") is preserved
here, with the checks this harness actually ran:

| # | Step | Failure behaviour | Measured |
|---|---|---|---|
| 1 | Restore the SQLite file(s) | stop; nothing is served | EV-OUTBOX-022…027 |
| 2 | Open read-only first; `PRAGMA integrity_check` | abort the restore; keep the previous data directory | EV-OUTBOX-022 |
| 3 | Schema version check (`PRAGMA user_version`) before serving | abort; an unknown schema must not serve | EV-OUTBOX-030 (startup stage) |
| 4 | Replay the **latest** delete state (tombstones) | refuse to open reads until replayed | EV-OUTBOX-026 |
| 5 | Replay the **latest** permission revocations | fail-closed (deny) until replayed | EV-OUTBOX-027 |
| 6 | Rebuild the derived indexes from the authority | index unavailable; reads continue without it | EV-OUTBOX-025 |
| 7 | Restore/repair outbox and task state; let the worker drain | events resume; index catches up | EV-OUTBOX-016…021 |
| 8 | Consistency check (authority ↔ index row/version parity) | do not declare "fully recovered" | EV-OUTBOX-004, 024 |
| 9 | Open reads | — | EV-OUTBOX-029 |

Steps 4 and 5 are the ones that cannot be inferred from the backup itself; see §4.

---

## 4. The two ways a restore silently corrupts state

Both were reproduced, not hypothesised.

### 4.1 An old backup resurrects a deleted memory

Measured (`EV-OUTBOX-026`): a backup taken at T0 → the memory is deleted at T1 → restoring the T0
copy brings the memory back as `lifecycle='active'`, and the restored copy contains **0 tombstone
rows** for it. The backup alone cannot know it was deleted.

After replaying the newer tombstone set, the memory is `deleted` again and the rebuilt FTS index
does not contain it; an unrelated memory survives untouched.

> **Contract requirement:** a restore is not complete until the *newest* delete state has been
> applied. Reads must be refused in between (default deny).

### 4.2 An old backup re-authorizes a revoked memory

Measured (`EV-OUTBOX-027`): a backup taken while a memory was `granted`; the memory is revoked
afterwards; restoring the T0 copy shows it `granted` with **0 revocation rows**, and the readable
count is one higher than the true state. After replaying the newer revocation set the memory is
`revoked` and the readable count drops by exactly one.

> **Contract requirement:** permission/revocation state is replayed from the newest source, never
> from the backup, and "permission state unclear ⇒ deny" is the only safe intermediate state.

**Open question this raises and cannot settle:** where does the newest delete/revocation state live
when the entire data directory is lost? The design says the *newer* state wins, which presupposes a
surviving newer copy. Options are (a) a separate revocation/tombstone journal outside the data
directory, (b) an append-only audit log that is itself backed up more often, (c) accepting that a
full-directory loss restores to the backup's state. **This needs a decision; it is not a
retention question and it is not covered by MC-AUD-009.**

---

## 5. Derived-index rebuild

Measured (`EV-OUTBOX-025`), FTS5 (`unicode61`):

| Step | Observation |
|---|---|
| healthy state | `fts_docs` 11 rows = 11 active memories = outbox-maintained `idx_fts` 11 rows |
| index destroyed (`DROP TABLE`) | search returns `no such table: fts_docs`; the database is otherwise intact |
| rebuild from the authority | 11 rows again, equal to both counts |
| keyword `k05` before / after | hit `mem_fixture_05` / hit `mem_fixture_05` |
| tombstoned memory's keyword `k08` | no hit before, no hit after |

> **Contract requirement:** a restore is followed by a rebuild; the rebuild reads the authority
> only and never the restored index. A restore that ships a stale index without rebuilding can
> answer a search with content that no longer exists.

The vector index was not rebuilt here: LanceDB is not installed/authorised in this area, and
`:memory:` is unsupported on this host (parent mandate §3). **The rebuild procedure for LanceDB is
therefore NOT_VERIFIED.** The authority-side precondition it needs (a complete, versioned set of
active memory rows with a document hash) is present and was exercised.

---

## 6. Retention — OPEN DECISION (MC-AUD-009)

**No retention period, count, or schedule is stated in this contract.** A number here would be
invented; the design package explicitly leaves it open:

- `05-可靠性与运维规范.md` §14, `OI-05-05`: "备份周期与保留数量 | TBD（03-m01 待确认清单 11）".
- `03-m01.md` §9 pending list item 11: "备份周期和保留数量".
- `PENDING-USER-DECISIONS.md`, `OI-012`: "备份周期/保留数量（PD-013 默认 365 天）| 随 D-2" — the
  365-day figure belongs to **audit-log retention** (PD-013), not to database backups, and the row
  itself says the decision travels with D-2.
- `contracts/config/storage.schema.json`: contains `backup_dir` and `backup_method` only. It has
  **no** retention key — verified, `EV-OUTBOX-028` (`retention_key_in_schema=none`).

### Options a decision must choose between

| Option | Shape | Cost | What it cannot survive |
|---|---|---|---|
| **R-1 Fixed window** | keep N daily copies (e.g. 7) plus the last known-good | simple, predictable disk | a corruption discovered later than N days |
| **R-2 Generational** | daily/weekly/monthly tiers with decreasing density | bounded growth, longer horizon | still a fixed horizon; needs a pruning job |
| **R-3 Never delete** | all backups retained | maximal recoverability | unbounded disk; conflicts with any future data-deletion obligation |
| **R-4 Deletion-driven** | keep until the content is deleted, then purge backups containing it | aligns with delete/revocation promises | expensive: requires per-backup content mapping, and purging an old backup destroys the ability to restore that point in time |

### What a decision must consider (and what is still missing)

1. **The legal/product deletion promise.** If a user deletes a memory, how long may a backup still
   contain it? `state-machines.md` already warns that "备份保留策略不得违反已确认的数据清除要求".
   R-1/R-2 must state whether a backup is exempt until it ages out.
2. **The measured restore-drive time.** Retention only matters if a restore can be performed.
   No restore-time measurement exists; the harness restores 29–1 720 pages in milliseconds, which
   says nothing about a production-sized database.
3. **Disk budget and its failure mode.** D8/§7.2 treat "磁盘空间不足" as a failure class; a retention
   policy is the only thing that bounds backup growth. No disk budget has been stated.
4. **Who prunes, and how pruning is audited.** No pruning mechanism exists in this harness.
5. **Interaction with the refresh/restore question in §4** — if the newest delete state lives
   outside the data directory, that store needs its own retention, which may need to be *longer*
   than the backup's.

**Recommendation: do not encode any retention value until (1) and (3) are decided.** The harness
therefore has no retention code, and `EV-OUTBOX-028` will fail the moment someone adds an
unratified numeric key to `storage.schema.json` — that is deliberate.

---

## 7. Failure behaviour

| Failure | Behaviour | Verified? |
|---|---|---|
| backup destination unwritable / disk full | `backup()` rejects; source untouched (the API never writes to the source) | NOT_VERIFIED — no disk-full injection |
| backup file corrupt | `integrity_check` fails on restore → abort | NOT_VERIFIED — a deliberately corrupted backup was not tested |
| restore of a newer schema into an older binary | refused at the schema-version stage | PARTIAL — stage exists and short-circuits (`EV-OUTBOX-030`), but a *newer* file was simulated with `user_version=999` only |
| restore while the old instance still runs | instance lock refuses the second writer | EV-OUTBOX-031 (lock stage) |
| index rebuild fails midway | index unavailable; authority unaffected | NOT_VERIFIED — rebuild is a single synchronous transaction here |
| backup while the database is being written | consistent snapshot, no torn pairs | EV-OUTBOX-023 |

Unverified rows are listed as unverified rather than described as working.

---

## 8. Environment limits on this contract

- **LanceDB is absent from the verification environment.** The parent mandate records that
  `@lancedb/lancedb` 0.41.0 exists only inside the canonical package tree and that `:memory:` is
  unsupported. No LanceDB backup/rebuild was performed here; the vector-index rows above are
  NOT_VERIFIED by construction.
- `node:sqlite`'s `backup()` is the only installable backup mechanism (no npm/pnpm). The `sqlite3`
  CLI is not present. `VACUUM INTO` works and was used as a cross-check.
- Backup sizes and durations here are synthetic and must not be quoted as performance figures.

---

## 9. Evidence map

| Test id | Evidence | What it establishes |
|---|---|---|
| `backup.backup_api_content_and_integrity` | EV-OUTBOX-022 | API copy is integrity-clean and content-equal; source unchanged |
| `backup.under_concurrent_writer_is_consistent` | EV-OUTBOX-023 | consistent snapshot under concurrent commits; no torn memory/outbox pairs |
| `backup.vacuum_into_cross_check` | EV-OUTBOX-024 | `VACUUM INTO` agrees with the API copy |
| `restore.derived_fts_index_requires_rebuild` | EV-OUTBOX-025 | FTS5 is derived; rebuild from the authority restores parity, excludes tombstones |
| `restore.tombstone_replay_prevents_resurrection` | EV-OUTBOX-026 | old backup resurrects a deletion; replay fixes it |
| `restore.revocation_replay_prevents_reauthorization` | EV-OUTBOX-027 | old backup re-grants access; replay fixes it |
| `backup.retention_policy_is_unspecified` | EV-OUTBOX-028 | no retention value exists in the contracts (MC-AUD-009 open) |

*(Evidence ids are assigned by append order; re-running `run-all.cjs` assigns a fresh contiguous
block. The test id is the stable key.)*
