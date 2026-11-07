# MEMORY-CORE-R4.1 — FINAL TECHNICAL AND IMPLEMENTATION REPORT

**Round 4.1 · 2026-11-05 · Asia/Shanghai**
Baseline: `E:\[PROJECT]\r41-baseline`
Canonical package (read-only): `C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design`

---

## 0. The eight questions the task book requires me to answer

**1. What has actually been solved?**
Seven things, each backed by executed evidence rather than prose: SQLite's file-backed behaviour
(55 checks, 0 failures); the LanceDB distance semantics and the true scope of "exactness"; the
Chinese FTS mechanism and its measured cost; fail-closed permission behaviour under injected
failure; the outbox's measured lease/backoff/crash behaviour; the Evidence schema defect and its
correction; and the reference/parameter-identity drift across the corpus. Total **549 evidence
records, 0 structural defects**.

**2. Which designs are now settled and need no further discussion?**
The enumerations, the five object field tables, RRF's two-channel rule (now proven by execution),
the `space_lifecycle`/`migration_state` split, the fail-closed permission principle, and the
sole-writer rule for `confirmed`. These are `CONFIRMED_EXISTING` — no further design debate needed.

**3. Which technologies now have real evidence?**
`node:sqlite` (file-backed WAL, transactions, locking, migration, backup/restore, crash recovery —
all measured on Windows x64). LanceDB 0.41.0 (scan-path exactness on fixed hashed datasets;
distance semantics; update/delete/reopen). FTS5 (12 tokenizer configurations measured on a
pre-registered labelled corpus). Synthetic identity/permission (222 assertions with contrastive
negative controls).

**4. Which technologies cannot be frozen, and why?**
All four. SQLite: no comparative driver evidence is obtainable (no package manager), no power-loss
test, no soak. LanceDB: indexed recall measured at **0.30** and `createIndex` ignores its own
options. FTS: Chinese fails its own product criterion (**4 of 19 queries return nothing**).
Identity: the real host was never touched, by design.

**5. What code now exists and could serve as a foundation?**
119+ CommonJS files across six verification areas, all runnable with bare `node` and no
dependencies: an RRF implementation, a lifecycle state validator with a 31-entry illegal-transition
matrix, an outbox simulator with real two-process concurrency, a mock permission engine with
contrastive mutants, dataset generators with recorded seeds and hashes, and evidence collectors.
**None of it is product code and none of it should be copied into a product module unexamined** —
each has a stated mock boundary in its own report.

**6. What genuinely remains to be solved?**
Eleven escalated items (E-1…E-6 in the Decision Register, plus OI-R-07/10/12/13). The five that
block most: the two-baseline precedence question; tier transition rules (none exist); the outbox
schema that cannot express its lease mechanism; FTS Chinese; and vector index policy.

**7. If product implementation started next, where should it start?**
**M01 storage**, and only after three cheap actions: merge the v2 Evidence schema, extend the outbox
schema (or move lease state out), and freeze `node:sqlite` within its stated scope. Those three are
proposals that already exist with evidence behind them — they cost review time, not engineering time.

**8. What still requires separate user authorization?**
Everything product-facing. `PRODUCT_IMPLEMENTATION_AUTHORIZATION = NOT_AUTHORIZED` at the end, as at
the start. Specifically barred: M01–M05 implementation, real user memory, real QQ Bridge/DSH identity,
real permission writes, Emotion Runtime, and the compression plane (FUT-09).

---

## 1. Scope and authorization ledger

| Axis | Value |
|---|---|
| `DESIGN_CLOSURE_STATUS` | **PARTIAL** |
| `CONTRACT_FREEZE_STATUS` | **NOT_FROZEN** (0 of 18 families frozen) |
| `TECHNICAL_VERIFICATION_STATUS` | **PARTIALLY_VERIFIED** |
| `CODE_IMPLEMENTATION_SCOPE` | ISOLATED_VERIFICATION_CODE (delivered) |
| `IMPLEMENTATION_READINESS_STATUS` | **PARTIALLY_READY** (M01 only, partially) |
| `PRODUCT_IMPLEMENTATION_AUTHORIZATION` | **NOT_AUTHORIZED** — unchanged |

These six are deliberately separate. Writing and running isolated verification code did not,
and must not be read as, a start of product implementation.

## 2. Asset inventory

| Item | Count | Location |
|---|---:|---|
| Verification source files (`.cjs`) | **119** | `03-verification/` |
| Evidence records | **549** | `06-evidence/*.jsonl` |
| Raw run logs | **114+** | `06-evidence/raw/` |
| Proposed contracts / reports | **15** | `04-contracts/` |
| Audit registers | **10** | `05-audit/` |
| TASK-RESULT reports | **6** | per area |
| Baseline snapshot (provenance) | 121 + 21 + 70 files | `01-baseline-corpus/` |

Evidence totals by area:

| Ledger | Records | PASS | PARTIAL | FAIL |
|---|---:|---:|---:|---:|
| sqlite | 55 | 50 | 5 | 0 |
| lancedb | 66 | 58 | 1 | 7 |
| fts | 71 | 56 | 13 | 2 |
| identity | 222 | 216 | 0 | 6 |
| retrieval | 77 | 75 | 1 | 1 |
| outbox | 34 | 34 | 0 | 0 |
| api | 63 | 62 | 0 | 1 |
| schema | 54 | 41 | 1 | 12 |
| audit | 6 | 6 | 0 | 0 |
| **Total** | **648** | **582** | **22** | **44** |

Structural integrity: 0 missing mandatory fields, 0 duplicate evidence ids, 0 unresolvable raw paths
(`05-audit/tools/audit-evidence-completeness.cjs`, 648 records, 0 defects).

Three areas closed this round that had **never been executed before**: the API contract harness
(63 checks against a real `node:http` mock server), business fixtures for **all 32 contract schemas**
(194 fixtures, **194/194 matched their expected verdict and keyword**), and six schema-conformance
proofs that turned previously-prose defects into executable ones. Both the API harness and the
fixtures were listed as the largest remaining gaps in MC-AUD-019 and MC-AUD-018.

The FAIL count rose from 21 to 44 not because things got worse, but because **six defects that were
previously only suspected are now proven**, and one of them is P0 (see §6).

## 3. Architecture review — findings

The module boundaries are coherent and acyclic (see `MODULE-BOUNDARY-MATRIX.md`). Four boundary
problems are recorded rather than smoothed over:

- **MB-1** M04 is the sole `confirmed` writer, yet no exception path is described for restore/migration.
- **MB-2** Write-side gate vs read-side final check are not explicitly declared non-substitutable.
- **MB-3** Two documents disagree on whether `structured` feeds RRF (`02:344/346`, `00:638` vs `11:617`, `06:558`).
- **MB-4** The module table has no row for a compression plane, yet a draft proposes one.

## 4. The four key technologies

### 4.1 SQLite driver — `PARTIALLY_VERIFIED(file-backed persistence, WAL, transactions, locking, migration, backup/restore, crash recovery)`

55 executed checks, **0 failing**. Real file databases, real child processes, real `SIGKILL`.

Established: file-backed creation and reopen; WAL mode and its `-wal`/`-shm` sidecars; commit and
transaction-wide rollback; FK/CHECK/UNIQUE enforcement; two connections on one file; write/write
contention surfacing as `SQLITE_BUSY` (errcode 5) and never as silent interleaving; cross-process
lock acquisition with `busy_timeout` measured at **929 ms** before errcode 5; deferred-upgrade deadlock
resolved by failure; versioned migration with failure rollback; backup and content-equal restore;
**SIGKILL mid-transaction rolled back 25 rows to zero**; 15 of 15 committed rows **survived** a kill
racing the flush — recorded `PARTIAL_PASS`, because survival there is timing-dependent rather than
guaranteed, and the record says so.

Two driver traps found that the adapter must handle: `backup()` on a connection holding its own
transaction throws **`errcode 0`, message "not an error"`**; and an unawaited `backup()` silently
produces a **0-byte file**.

The prior round's "crash recovery passed" was **overstated**: its child ran
`BEGIN; INSERT; process.exit(0)` — a clean exit, not a crash.

**Not frozen.** Gates: no comparative driver evidence (no package manager on this machine), no
power-loss test, no soak.

### 4.2 LanceDB — `PARTIALLY_VERIFIED(scanned-path exactness on fixed hashed datasets)`

The prior claim (`max cosine error 5.6926e-9`, tolerance `1e-5`) **did not survive audit**:

1. The comparison was between **non-comparable quantities** — `_distance` is squared L2, not `1−cos`.
2. The tolerance is **1757× the observed error**, 84× one binary32 ulp — it cannot distinguish correct
   from wrong at `1e-5`.
3. **No stored dataset existed**; the hash covered an in-memory literal array.

Three defects found in the SDK itself:

- `createIndex()` **ignores both `indexType` and `distanceType`** — every request yields `IvfPq`/`L2`,
  confirmed independently on three option sets including the SDK's own native `l2`.
- **ANN recall@10 = 0.30**, flat across `nprobes` 1→32 (the knob is inert).
- **`±Infinity` is accepted and persisted** while only `NaN` is rejected; an infinite row then
  disappears from cosine results and returns `_distance = Infinity`, which JSON-serialises to `null`.

Plus: cosine distance leaves `[0,2]` (self-match returns `−1.19e-7`), and tie order follows **insertion
order**, so ranking is not reproducible across a rebuild.

**Contract consequence:** exactness is a property of the **scan path only**; indexed retrieval is
approximate and must be gated behind a measured recall threshold. `VectorStoreAdapter` must not
expose `indexType`/`distanceType` as if they worked.

### 4.3 Chinese FTS — `PARTIALLY_VERIFIED`, product criterion **FAILS**

`unicode61` does not fail to tokenize Chinese — it makes an entire contiguous CJK run **one token**,
so retrieval is **run-equality**, not word retrieval. Model fit: 39/40 for run-equality vs 24/40 for
substring.

| Config | hit@10 | recall@10 | index B/doc @20k |
|---|---:|---:|---:|
| `unicode61` (current) | 0.800 | 0.668 | 449.5 |
| `trigram` | 0.444 | 0.444 | 946.2 |
| `unicode61` + bigram shadow | 0.911 | 0.856 | 797.1 |
| **`unicode61` + unigram+bigram shadow** | **0.956** | **0.942** | **1359.7 (3.0×)** |
| `LIKE` substring | 0.978 | 0.948 | 449.5 (index unused) |

**PC-1 FAILS**: 4 of 19 Chinese word/character queries return nothing (`爵士乐`, `咖啡`, `西`, `蛋糕`).
`trigram` is **worse** — it silently returns nothing for any query under 3 characters, which is the
normal case in Chinese.

**Recommended:** `unicode61` + indexed unigram/bigram shadow — pure JS, ~80 lines, no dependency, no
architecture change, column names and bm25 weights intact; cost is 3.0× index bytes.
`LIKE` is rejected because it produces **no bm25 score**, so `channel_score` feeding RRF would be
meaningless.

The agent's correction to my premise was partly right and I verified the rest: `loadExtension()` **is**
exposed, but `enableLoadExtension(true)` fails with *"disabled at database creation"* and no
constructor option enables it — a compile-time capability, not merely a missing DLL.

### 4.4 Host identity — `SYNTHETIC_IDENTITY_VERIFIED`, and explicitly **not** real-host

222 assertions, one run. **0 failures on the hardened profile.** 6 FAILs are negative controls.

| Attack | weak_v1 | hardened |
|---|---|---|
| revocation outrun by cache | LEAKED | BLOCKED |
| bodies cached at rest | LEAKED | BLOCKED |
| cross-group via cache key | LEAKED | BLOCKED |
| same bot, two groups | LEAKED | BLOCKED |
| evidence stricter than memory | LEAKED | BLOCKED |
| missing identity defaulted | LEAKED | BLOCKED |

This is the correct shape for a security suite: **break it deliberately and show the break matters.**
A green run would mean the mutant stopped mutating.

`INDETERMINATE` never becomes `ALLOW` — 7 injections, 0 allows. All 222 records carry
`synthetic_identity: true`, `real_host_verified: false`.

> `SYNTHETIC_IDENTITY_VERIFIED != REAL_HOST_MAPPING_VERIFIED`

## 5. Other verification

**Retrieval** (77 checks): RRF proven to exclude `structured` by constructing a case that would win
if fused; 31 illegal lifecycle transitions all rejected; space mismatch throws; token accounting
reported `UNAVAILABLE` rather than invented.

**Outbox** (34 checks): measured backoff 163/325/608 ms vs expected 150/300/600; lease 249 ms,
reclaim 414 ms later; two real worker processes split 6 events 3/3 with **0 duplicate effect groups**;
dead letter at attempts 4 > ceiling 3; backup under 32 concurrent commits with **0 torn pairs**.
**Exactly-once across the index boundary does not hold** — two duplicate windows were reproduced
rather than argued; safe only because writes are upserts keyed by `(aggregate_id, aggregate_version)`.

**Restore hazard (P0):** restoring an older backup **resurrects a deleted memory** and
**re-grants a revoked authorization**, until the newest tombstone/revocation sets are replayed.

**API contract** (63 checks) — the first API harness ever run on this project. A real `node:http`
mock server implements the documented endpoints; responses are validated against the shipped schemas;
all 16 error codes are producible; permission DENY/INDETERMINATE/THROW/TIMEOUT all fail closed.

**62 PASS, 1 FAIL.** The FAIL is a genuine contract defect, and it is the *opposite* of what my
document-level report predicted:

| My document claim | Executed reality |
|---|---|
| "`error.schema.json` cannot enforce canonical codes because it admits v1 aliases" | **Wrong.** The enum holds exactly the 16 canonical codes and rejects **0 of 7** tested aliases |
| — | The real defect: `error-codes.md §2` **claims** the enum is the union of both sides. It is not. |

So the schema is already correct and the **documentation is wrong** — which is the more dangerous
direction, because an implementer reading §2 would skip normalisation. I had inferred the defect from
the document's stated intent without reading the schema's enum; the harness caught it.

**Business fixtures** (194 fixtures across **all 32 contract schemas**) — the half of MC-AUD-018 that
had never been done. **194/194 matched their expected verdict and keyword**, exit 0.

A negative fixture must fail **for the intended reason**, and the runner asserts the exact keyword and
instance path (`assert [enum]@'/provider' -> KEYWORD_CONFIRMED`, not merely "it failed"). Fixtures are
organised per schema as `valid-*` / `invalid-*` / `boundary-*` / `finding-*`, and each carries its
authority citation.

Seven `finding-*` fixtures assert a specific authority drift. **The parent audited them and refuted
one:** `config/anchor` was flagged for omitting `ORIGINAL_QUOTE_QUERY` from `default_read_mode`, but
`anchor.schema.json` line 109 says in words that this is deliberate — *"仅登记非证据类意图；
ORIGINAL_QUOTE_QUERY 恒为证据模式，不在此配置"*. The omission is correct behaviour. Had that
false positive entered the register, someone would have "fixed" a schema that was already right.

One upheld finding is sharper than my own note: **v2 requires `tenant_id` but `DD-AUTHORITY-V2 §3.2`
does not list it**, so a strict transcription of the authority table fails against the corrected
schema. The requirement is deliberate (isolation is load-bearing) but the authority must be amended.

## 6. Defects found and fixed

| ID | Defect | Status |
|---|---|---|
| **OI-R-15** | **The authority's own default MemoryUnit state is invalid — a fresh memory has no legal state. Blocks every write path.** | **OPEN (P0) — escalated** |
| OI-R-10 | `outbox-event.schema.json` **cannot express the lease mechanism it mandates** — 7 required fields rejected by `additionalProperties:false` | **OPEN (P0)** |
| OI-R-12 | Restore resurrects deleted / re-grants revoked data | **OPEN (P0)** |
| OI-R-04 | `evidence.schema.json` **rejects any spec-compliant Evidence instance** (proven: fails on `additionalProperties`, `required`, `enum`) | **fix delivered** (`04-contracts/schemas/`) |
| OI-R-05 | UTF-8 BOM made `read-response.schema.json` unparseable by strict JSON | **fix delivered** |
| OI-R-06 | SM-1 mixes the lifecycle axis with the validation axis | **OPEN (P1)** |
| OI-R-07 | Tier rules **do** exist in `11 §7.12` (3×6 matrix + 5 rules) but **no state machine transcribes them**; 3 operational gaps remain (executor conditions, decidable L3 threshold, demotion paths) | **OPEN (P1) — escalated** |
| OI-R-01 | Two competing R4.1 baselines, mutually unreferencing, no precedence rule | **OPEN (P1) — escalated** |
| MC-AUD-006 | 355 parameter references, **75 unresolved across 55 names** | **OPEN** |
| OI-R-03 | The documented RRF worked example contains duplicate ranks and is invalid | **OPEN (P2)** |

### 6.1 The P0 — and why it matters more than the others

`DD-AUTHORITY-V2 §3.1` and `memory-unit.schema.json` both declare a fresh MemoryUnit to be
`status=ACTIVE` + `validation_state=unverified`. The schema's own `allOf[1]` requires
`ACTIVE ⇒ confirmed|disputed`. **The declared defaults are therefore invalid against the schema that
declares them.**

Proven, not inferred — `03-verification/schema/memory-unit-default-contradiction.cjs` builds the
minimal authority-conformant instance and hands it to the real Draft-07 engine:

```json
{ "valid": false,
  "errors": [ { "path": "validation_state", "keyword": "enum",
                "message": "'unverified' is not one of ['confirmed', 'disputed']" } ] }
```

**Why this is P0 and the others are not:** the evidence defect blocked *one* object type from being
serialised; the outbox defect blocked *one* table. This blocks **every memory write**, because every
write path starts from a default and there is no field combination that both satisfies `D-AP-05`
(confirmed only via M04) and produces a legal ACTIVE memory before M04 has reviewed it.

The design tension is legitimate — `D-AP-05` wants `confirmed` to be M04-only, and the schema tried to
enforce that *statically*. But a memory must exist before it can be reviewed, and the status enum has
no pre-confirmation value. **The fix is to move that rule from a static invariant to a transition
guard** (option A in the register): `ACTIVE+unverified` becomes legal, and the *transition* to
`confirmed` remains M04-only. That preserves both intents. It is a design decision, so it is escalated.

### 6.2 Three more proven defects from the fixture work

| ID | Claim | How proven |
|---|---|---|
| DRIFT-MEMORY-OBJECT-01 | `memory-object.schema.json` still accepts the retired v1 `type: "episodic"` while `memory-unit` rejects it | reproducing fixture |
| DRIFT-CANDIDATE-01 | §2.7's six-value `candidate.status` has no home; a competing 5-value `review_state` ships instead | reproducing fixture |
| DEFECT-EVENT-01 | `event.schema.json` does not enforce its own `$comment` MUST — `participants: []` validates | reproducing fixture |
| DRIFT-TEST-CONFIG-01 | `config/test` `reporting.status_values` contradicts `EVIDENCE-FORMAT.md` (misspells `NOT_RUN`, omits `PARTIAL_PASS`, invents `MOCKED`) | reproducing fixture |

## 7. What went wrong in this round, recorded honestly

Four process failures, all caught, none fatal:

1. **Custody breach (mine).** I stripped a BOM from the read-only provenance tree at 20:11:26.
   Measured blast radius: exactly 1 file, BOM only, canonical originals intact. Independently
   detected by another agent. Recorded in `PARENT-CORRECTIONS.md §1`.
2. **A fabricated precondition.** An agent gated a failing tier check on `tier_review_approved`,
   which appears **zero times** in the design corpus, turning a red finding green. Rejected;
   re-recorded as `PARTIAL_PASS` with the gap stated.
3. **Two self-corrections by me.** I first reported three lifecycle FAILs as "false failures" from a
   bad assertion; re-execution showed **one** was false and **two** were genuine (tier promotion
   refused). I then claimed the permission schema had "no requester field"; re-reading refuted it —
   `actor` **is** the requester and `subject_id` is the owner. The real defect is narrower:
   `actor.role` is optional.
4. **BOM contamination** from PowerShell `-Encoding utf8` — 33 files across two areas. All stripped;
   a `check-bom.cjs` gate now enforces it.

Each is recorded rather than edited away, because a baseline that hides its own defects is worth
nothing.

## 8. Contract freeze and readiness

**0 of 18 contract families are `FROZEN`.** 9 are `PARTIALLY_VERIFIED`, 7 `CONTRACT_DEFINED`,
1 `DESIGN_INCOMPLETE`, 1 `VALIDATION_PENDING`. Full detail in `CONTRACT-FREEZE-REGISTER.md`.

| Module | Impl. Ready | Authorized |
|---|---|---|
| M01 Storage | **PARTIAL** | **NO** |
| M02 Retrieval | **NOT_READY** | **NO** |
| M03 Extraction | NOT_READY | **NO** |
| M04 Integration | NOT_READY | **NO** |
| M05 Permission | **BLOCKED** | **NO** |

## 9. Next stage

**Start with M01**, after three cheap contract actions (merge the v2 Evidence schema, extend the
outbox schema, freeze the driver within scope). Then M02, gated on two decisions rather than more
verification: the Chinese tokenizer choice, and the vector index policy.

The two architecture questions that block everything downstream and **cannot be resolved by more
verification** are:

- **E-1** which R4.1 document set is authoritative;
- **E-2** what the tier transition rules are.

## 10. Stage G truth audit

`05-audit/tools/STAGE-G-TRUTH-AUDIT.cjs` — **8/8 checks pass**: required files exist (23), every area
has a TASK-RESULT (6/6), evidence complete (549 records, 0 defects), no BOM outside provenance,
provenance differs from canonical only by the recorded BOM deviation, **no product module directory
exists**, the SQLite suite reproduces end-to-end, and the five authorization axes are recorded
separately.

## 11. Deliverable map

| Report | Path |
|---|---|
| SQLite | `03-verification/sqlite/TASK-RESULT.md` + `README.md` |
| LanceDB | `03-verification/lancedb/TASK-RESULT.md` + `PARENT-TECHNICAL-NOTE.md` |
| FTS | `03-verification/fts/TASK-RESULT.md` + `PARENT-ADDENDUM-extension-loading.md` |
| Identity | `03-verification/identity/TASK-RESULT.md` + `HOST-IDENTITY-MAPPING-CONTRACT.md` |
| Retrieval | `03-verification/retrieval/TASK-RESULT.md` |
| Outbox | `03-verification/outbox/TASK-RESULT.md` |
| Schema/API | `03-verification/schema/TASK-RESULT.md` |
| Historical/consistency | `05-audit/TASK-RESULT.md` |
| Contracts | `04-contracts/` (15 files) |
| Registers | `05-audit/` (10 files) |

---

**Final statement.** Memory Core R4.1 has moved from a design described in documents to a baseline
with **measured contracts, independent verification tools, and reproducible evidence** — 549 records,
0 structural defects, 119 runnable files. Three claims from the previous round were found to be
unsupported and were corrected. Eighteen contract families remain unfrozen because they fail their
own criteria, and that is the accurate state, not a shortfall to be apologised for. Product
implementation remains **unauthorized**, and the next stage should not begin until the two
architecture questions are answered.
