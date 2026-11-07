# SECOND-ROUND-FINAL-ISSUE-REGISTER — parent audit, R4.1 round 4.1

Date: 2026-11-05 (Asia/Shanghai)
Baseline: `E:\[PROJECT]\r41-baseline`
Supersedes in scope: `SECOND-ROUND-ISSUE-REGISTER.md` (MC-AUD-001…019) — **not** replaced; this file
records their current disposition plus the issues found in this round.

Status vocabulary: `OPEN` · `CLOSED` · `CLOSED_WITH_SCOPE` · `PARTIALLY_CLOSED` · `BLOCKED` ·
`DEFERRED` · `SUPERSEDED`

---

## 1. Disposition of MC-AUD-001 … 019

This table is the parent's own adjudication after inspecting the artefacts, not a copy of the
previous round's claims.

| ID | Previous status | **This round** | Basis (real evidence) | What still blocks closure |
|---|---|---|---|---|
| MC-AUD-001 | DECIDED; sync pending | **CLOSED_WITH_SCOPE** | `00-PARENT-MANDATE.md` separates `CODE_IMPLEMENTATION_SCOPE` (isolated verification) from `PRODUCT_IMPLEMENTATION_AUTHORIZATION` (**NOT_AUTHORIZED**). Every agent brief repeats it. | Nothing; but the separation must survive into the next round's instructions. |
| MC-AUD-002 | PARTIAL | **CLOSED_WITH_SCOPE** | 31/31 contract schemas meta-valid; `DD-AUTHORITY-V2` remains the semantic authority; `contracts/` is the serialized authority. | See MC-AUD-006 (identity) and the two-document-set conflict (§2.1). |
| MC-AUD-003 | PARTIAL; driver not frozen | **PARTIALLY_CLOSED** | 55 executed SQLite checks, 0 failing; `03-verification/sqlite/TASK-RESULT.md`. Prior "crash recovery" claim shown to be a clean `process.exit(0)`, not a crash. | Driver **not frozen**: no comparative evidence (no package manager), no power-loss test, no soak. |
| MC-AUD-004 | BLOCKED | **PARTIALLY_CLOSED** | LanceDB harness ran audited suites; see `03-verification/lancedb/`. | Exactness claim at scale still open; see §2.2. |
| MC-AUD-005 | DECIDED; sweep pending | **PARTIALLY_CLOSED** | RRF implementation rejects `structured` as a channel (`rrf.structured_channel_rejected`). | A hand-computed worked-example test fails because the *documented example itself* contains a duplicate `channel_rank` — the doc is wrong, not the code. See §2.3. |
| MC-AUD-006 | DECIDED; partial | **OPEN** | `validate-parameter-registry.cjs`: 68 canonical IDs vs **355 references, 75 unresolved across 55 distinct names**. | Register the 55 names or rewrite the references. SRD-003 is decided but not enforced. |
| MC-AUD-007 | OPEN / external gate | **PARTIALLY_CLOSED** | Synthetic identity suite executed; `SYNTHETIC_IDENTITY_VERIFIED != REAL_HOST_MAPPING_VERIFIED` honoured. | Real-host mapping remains untouched by design. |
| MC-AUD-008 | OPEN | **PARTIALLY_CLOSED** | Outbox simulator + suite executed; lease/retry/idempotency exercised. | Operational numeric defaults still `EXPERIMENTAL`. |
| MC-AUD-009 | OPEN | **OPEN** | No retention period decided anywhere in the corpus, correctly. | Needs a user decision, not an engineering guess. |
| MC-AUD-010 | OPEN | **PARTIALLY_CLOSED** | SQLite suite proves FK/CHECK/UNIQUE enforcement on a real file DB. | The actual product DDL still does not exist; only a representative schema was constrained. |
| MC-AUD-011 | OPEN | **PARTIALLY_CLOSED** | `MEMORY-LIFECYCLE-MATRIX` + illegal-transition matrix executed. | **Tier rules exist in `11 §7.12` (3×6 matrix + 5 rules) but no state machine transcribes them, and 3 operational gaps remain** — see corrected §3.1 (OI-R-07). |
| MC-AUD-012 | DEFERRED | **DEFERRED** | No thresholds invented anywhere. FTS report proposes an explicit gate rather than a fabricated pass mark. | Establish baseline at P3 as originally planned. |
| MC-AUD-013 | OPEN | **CLOSED** | Status axes are kept separate in this round's reporting (design / contract / verification / readiness / authorization). | — |
| MC-AUD-014 | PARTIAL | **PARTIALLY_CLOSED** | Every executed check emits a structured record with raw log path; 386+ records across 5 ledgers, all schema-valid. | Some agent records have under-specified `actual` strings (§3.2). |
| MC-AUD-015 | OPEN | **PARTIALLY_CLOSED** | `API-CONSISTENCY-REPORT.md` identifies 4 concrete contract defects (scope array/string collision, `revoke` TBD default, `partial` with no licensed trigger, error schema cannot enforce canonical codes). | API harness never executed. |
| MC-AUD-016 | DECIDED / schema edited | **CLOSED_WITH_SCOPE** | `embedding-space.schema.json` separates `space_lifecycle` from `migration_state`; retrieval suite tests that a `draining` space is not treated as `active`. | — |
| MC-AUD-017 | OPEN | **PARTIALLY_CLOSED** | Inheritance matrix being produced by the historical-consistency workstream. | Not yet verified by the parent. |
| MC-AUD-018 | PARTIAL PASS | **PARTIALLY_CLOSED** | 31/31 schemas meta-valid + a corrected Evidence schema with positive/negative fixtures. | Business fixtures for the other 30 schemas still missing. |
| MC-AUD-019 | OPEN | **OPEN** | API / permission-end-to-end / token-budget / LLM / E2E harnesses **not executed**. Permission was executed at the unit/synthetic level only. | Still the largest single gap. |

**Net movement:** 2 closed, 1 closed-with-scope ×2, 12 partially closed, 3 still open, 1 deferred.
No issue was marked closed on the strength of a claim alone.

---

## 2. New issues found this round

### 2.1 OI-R-01 — Two competing R4.1 baselines with no precedence rule  · **P1 · OPEN**

`Memory-Core-R4.1-Design` (00–15 + `contracts/`) and `R4.1-UNIFIED-MEMORY-CORE` (00–16 + `audit/`)
are both presented as the R4.1 design. Neither states which wins. The `contracts/` folder exists only
in the first set, so the machine schemas and the second set's prose are not reconciled.

**Impact:** any implementer can cite a document from set B that contradicts a schema in set A.
**Fix:** a recorded precedence decision. **Architecture-level — reported to the user, not decided by me.**

### 2.2 OI-R-02 — Prior LanceDB exactness claim compared non-comparable quantities · **P1 · PARTIALLY CLOSED**

The prior artefact claimed "max cosine-distance error 5.7e-9, tolerance 1e-5". The SDK's `_distance`
is **squared L2 on raw vectors**, verified independently in this session
(`[1,0,0]`→0, `[0.6,0.8,0]`→0.800000011920929 = 1−0.64²... i.e. squared L2, and `[0,1,0]`→2).
Comparing a squared-L2 number against a cosine-distance expression is dimensionally wrong; a tiny
number obtained that way is not evidence of exactness.
**Impact:** the "exactness verified" statement in the prior round does not stand as written.
**Fix:** recompute the reference in the metric the SDK actually uses (squared L2 on normalised
vectors ⇒ cosine-equivalent ordering), and state the tolerance's origin.

### 2.3 OI-R-03 — The documented RRF worked example is internally invalid · **P2 · OPEN**

`02 §7.4`'s example assigns `channel_rank 1` twice within the `fts` channel. A correct RRF
implementation must reject that input; `rrf.cjs` does (`RrfError: duplicate channel_rank 1 in channel 'fts'`).
**Impact:** the one hand-checkable example in the algorithm spec cannot be hand-checked.
**Fix:** correct the example's ranks (they should be 1,2,3,4 within a channel).

### 2.4 OI-R-04 — `evidence.schema.json` cannot hold a compliant Evidence object · **P0 · CLOSED (fix delivered)**

Proven by execution, not by reading: a §3.2-compliant instance is rejected by the shipped schema with
failing keywords `additionalProperties`, `required`, `enum`. The Technical Verification Report
described this correctly; the schema was never fixed.

**Fix delivered:** `04-contracts/schemas/evidence.schema.json` (v2), validated to accept the
compliant instance while still rejecting a v1 `source_type` alias and empty `original_text`.
**Remaining action:** merge into the canonical package (see §2.1 for why I did not overwrite it).

### 2.5 OI-R-05 — UTF-8 BOM in `read-response.schema.json` · **P2 · CLOSED (fix delivered)**

A strict JSON parser rejects the file; a lenient one hides it. One of 31 contract files affected.
**Fix:** BOM stripped in the snapshot. **Recommendation:** add a BOM check to CI.

### 2.6 OI-R-06 — `SM-1` mixes the lifecycle axis with the validation axis · **P1 · OPEN**

`contracts/state-machines/state-machines.md` SM-1 lists `active → disputed` as a **lifecycle**
transition, but `disputed` is a `validation_state` value. `01 §7.7` requires the two axes to be
separable. The prototype validator refuses the edge and names the axis confusion
(`lifecycle.disagreement_sm1_conflates_axes`).
**Impact:** a state machine transcribed literally from SM-1 would implement a cross-axis jump.
**Fix:** remove the validation-axis edges from SM-1 and point them at SM-2.

### 2.7 OI-R-07 — Tier rules exist in `11 §7.12` but are transcribed into no state machine · **P1 · OPEN** (see §3.1)

### 2.8 OI-R-08 — `node:sqlite` `backup()` has two dangerous behaviours · **P1 · OPEN (documented)**

1. Unawaited, it silently produces a **0-byte** file.
2. Called on a connection holding its own open transaction, it throws `errcode 0`, message
   **"not an error"**.
**Impact:** a naive `DatabaseAdapter.backup()` reports success while producing nothing, or fails with
an undiagnosable message.
**Fix:** the adapter must await, must use a dedicated connection, and must verify the artefact
(size > 0 + `integrity_check`). Recorded as a contract requirement in the SQLite report §5.

---

## 3. Findings escalated to the user

### 3.1 OI-R-07 — the tier axis is documented in prose but absent from every state machine

**CORRECTED.** This section originally said "there is no table of legal tier transitions … nothing says
what promotes a memory from L1 to L2 or L2 to L3". **That was too strong and is wrong.** Re-reading
`11-MemoryAnchor与语义分类.md §7.12` (L493–514) shows a substantial body of tier rules that already
exists:

| Location | Rule |
|---|---|
| `11` L493–506 | A **3×6 `semantic_type` × `tier` matrix** with `●`/`○`/`—` frequency markers, explicitly labelled "典型基准，非硬绑定" |
| `11` L510 | The final `tier` **MUST** be decided by M04 integration; M03 only supplies `lifecycle_candidate` |
| `11` L513 | **MUST NOT** set L3 merely because content is "important"; **MUST** have evidence of stable/definite/repeated/user-requested |
| `11` L514 | `L1` **MUST NOT** auto-promote to `L3` (X-2) |
| `11` L512 | **MUST NOT** treat `L1` as disposable |
| `11` L511 / L632 | **MUST NOT** use tier for read priority or as a sort weight |
| `11` L597 | tier **MUST NOT** act as a read gate |
| `11` L348 | `L1`/`L2` states **MUST** have an expiry/review mechanism |

What is genuinely missing is narrower and specific:

1. **An executor condition table** — "M04 decides" is stated, but there is no table of conditions under
   which M04 should promote or demote.
2. **A decidable L2→L3 threshold** — L513 names the *kinds* of justification (stable / definite /
   repeated / user-requested) but gives no testable criterion.
3. **Demotion paths** — L514 forbids L1→L3, but nothing defines L3→L2 or L2→L1.
4. **Transcription into the state machine** — `SM-1`…`SM-8` contain **zero** tier edges (this part of the
   original finding stands). So a state machine built from SM-1 as written has no tier axis at all.

Evidence for point 4: the executed validator, given only what `SM-1` transcribes, refuses an
`L2 → L3` move with `VALIDATION_ERROR` (`parent.reeval.tier_promotion_refused`).

**Why this still matters:** the rules exist but are *unreachable* from the state machine, and three
operational gaps remain. Expiry (required by L348) and promotion (assigned to M04 by L510) are
unimplementable from `SM-1`, and demotion is undefined everywhere.

**Options:**
| Option | Consequence |
|---|---|
| **A (recommended)** | Promote the existing `11 §7.12` matrix and its five rules into the authority (`01`/`02`), add the three missing pieces (executor conditions, decidable L3 threshold, demotion paths), and mirror them into `SM-1` | This is **relocation + 3 gap-fills**, not new design — most material already exists |
| B. Declare tier static at creation (only `status` moves) | Simplest, but directly contradicts `11` L510 (M04 decides) and L348 (L1/L2 need expiry/review) — it would require overturning two written rules |
| C. Defer tier semantics; V1 treats tier as a label only | Honest, but contradicts L513 (L3 needs justification) and L514 (no L1→L3), and must be written down |

**Recommendation: A.** It is the only option consistent with the existing corpus. B and C both require
overturning rules that are already written and approved.

This is a **design decision**, so I am not deciding it unilaterally.

### 3.2 OI-R-09 — Evidence records can under-specify their own failure (process defect)

Agent-generated records for the lifecycle orthogonality checks printed an `actual` string that
**omitted the decisive sub-condition**, making a FAIL unattributable from the ledger alone. On
independent re-execution one of them was a **false failure** (all sub-conditions held) and the other
was a genuine one (tier promotion refused).

**Impact:** a reader of the evidence ledger could neither trust nor refute the result.
**Fix:** `actual` must state the failing sub-condition, not a passing prefix. Adopted for the parent's
own records; the existing agent records are corrected by the `EV-RETRIEVAL-P*` parent records rather
than by rewriting history.

### 3.3 Scope of authority this round (authorization ledger)

| Axis | Value |
|---|---|
| `DESIGN_CLOSURE_STATUS` | **PARTIAL** |
| `CONTRACT_FREEZE_STATUS` | **NOT_FROZEN** (0 families frozen; see `CONTRACT-FREEZE-REGISTER.md`) |
| `TECHNICAL_VERIFICATION_STATUS` | **PARTIALLY_VERIFIED** |
| `IMPLEMENTATION_READINESS_STATUS` | **PARTIALLY_READY** |
| `PRODUCT_IMPLEMENTATION_AUTHORIZATION` | **NOT_AUTHORIZED** (unchanged) |

Writing and running isolated verification code did **not** upgrade the product authorization.

---

## 4. Additional issues found late in the round (from outbox and schema verification)

### 4.1 OI-R-10 — `outbox-event.schema.json` cannot express the lease mechanism it requires · **P0 · OPEN**

Proven by execution (`EV-SCHEMA-0xx`, `schema.outbox_cannot_express_lease_mechanism`):

The shipped schema declares `additionalProperties: false` and declares **none** of the seven fields
the mandatory claim/lease protocol needs: `idempotency_key`, `lease_owner`, `lease_until`,
`claim_token`, `attempts`, `dead_lettered_at`, `next_attempt_at`. Therefore **a worker row that is
fully compliant with the design's own lease semantics is invalid against the design's own schema.**

Compounding: the schema already carries `retry_count` / `next_retry_at`, while `03` and the outbox
contract use `attempts` / `next_attempt_at`. One concept, two spellings.

**Impact:** M01 cannot persist the worker protocol it is required to implement.
**Options:** (a) extend the schema with the lease fields; (b) move lease state to a separate table.
**Recommendation:** (a) — the schema is the serialized authority for the event row and the protocol is intrinsic to it. (b) is viable but means the schema stops documenting the worker protocol.

### 4.2 OI-R-11 — SM-4's `cancelled` state is absent from the outbox status enum · **P2 · OPEN**

`schema.outbox_status_enum_vs_sm4` confirms: SM-4 lists a `cancelled` state that the status enum does
not contain. A worker implementing SM-4 literally would emit a value the contract rejects.
**Note:** SM-4 rule 8 describes cancellation as an idempotent short-circuit, which may not require a
persisted state. So this may be an SM-4 wording defect rather than a schema defect — but as written,
the two disagree.

### 4.3 OI-R-12 — Restore can resurrect deleted and re-grant revoked data · **P0 · OPEN**

Reproduced by execution (agent G): restoring an **older** backup brings back a deleted memory
(as `active`, 0 tombstones) and re-grants a revoked authorization (granted, 0 revocations). Replaying
the newest tombstone and revocation sets fixes both.

**Impact:** a restore is not merely stale — it is a **data-resurrection and authorization-restoration
vector**. This is a security-relevant behaviour, not a convenience issue.
**Required contract clause:** a restore MUST NOT be considered complete until the newest delete and
revocation state has been applied, and reads MUST be refused in the intervening window.
**Open:** where the newest delete/revocation state lives if the whole data directory is lost.

### 4.4 OI-R-13 — `actor.role` is optional, so the requester/owner distinction is not mandatory · **P1 · OPEN**

**Corrected entry.** This was first recorded as "the schema has no field for the requester" — that
was wrong. Re-reading `permission-context.schema.json` shows `actor{type,id}` **is** the requester,
with `role` enumerating `requesting_user`; `subject_id` is documented as the memory's owner. The
distinction is representable.

The real, narrower defect: `actor.required` is only `["type","id"]` and `role` permits `null`. So a
caller may supply a requester with no role, and any policy keyed on role must then fall back to a
default — which is precisely how a permission model starts drifting.

**Fix:** make `role` required (non-null) for actions that depend on it, or define the explicit deny
behaviour when it is absent.
**Why it matters anyway:** the harness supplies the requester out-of-band, so a green suite cannot
prove the contract forces callers to declare who they are.

### 4.5 OI-R-14 — The outbox's "exactly-once" claim is only true inside one transaction · **P1 · OPEN (wording)**

Measured by execution: exactly-once per channel holds when the effect and its journal row share one
SQLite transaction. It does **not** hold across the index boundary. Two duplicate windows were
reproduced, not argued:
- crash between external commit and journal commit → external effects 1→2, keyed index row stays 1
- a worker stalled past its lease is fenced, yet its external write already landed → fts=2

Crossing the index boundary is therefore **at-least-once**, and safe only because index writes are
upserts keyed by `(aggregate_id, aggregate_version)`.

**Impact:** `03`/`05` say "exactly-once"; the measured behaviour is "idempotent effect". For an
upsert that is harmless; for any append, increment, or external side effect it is a duplicate.
**Fix:** reword the contracts to "idempotent effect (at-least-once delivery, upsert-keyed)".

---

## 7. P0 — the authority's own default MemoryUnit state is invalid (OI-R-15)

**This is the most consequential defect found in the entire round: it blocks every write path.**

### What the authority says

`DD-AUTHORITY-V2.md §3.1` and `memory-unit.schema.json` both declare two defaults:

| Field | Declared default |
|---|---|
| `status` | `ACTIVE` |
| `validation_state` | `unverified` |

### What the schema also requires

`memory-unit.schema.json` `allOf[1]`:

```json
"if":   { "properties": { "status": { "const": "ACTIVE" } }, "required": ["status"] },
"then": { "properties": { "validation_state": { "enum": ["confirmed","disputed"] } },
          "description": "ACTIVE 记忆的 validation_state MUST 为 confirmed（经 M04 通道）或 disputed；unverified/rejected 不得 ACTIVE（台账 D-AP-05）。" }
```

### The contradiction

`ACTIVE` **requires** `confirmed` or `disputed`. The declared default is `unverified`.
Therefore **the default pair is invalid against the schema that declares it.**

### Proven by execution, not by reading

`03-verification/schema/memory-unit-default-contradiction.cjs` builds the minimal
authority-conformant instance with `status=ACTIVE`, `validation_state=unverified`, and validates it
with the real Draft-07 engine (`jsonschema` 4.26.0):

```json
{ "valid": false, "count": 1,
  "errors": [ { "path": "validation_state", "keyword": "enum",
                "message": "'unverified' is not one of ['confirmed', 'disputed']" } ] }
```

One error, at the exact field the contradiction is about.

### Why it is P0

- **A fresh memory has no legal state.** Every ingest path that creates a memory with the documented
  defaults produces an object that cannot be serialised.
- **Every write path is blocked**, not just one: create, candidate-commit, revision, migration import —
  anything that starts from a default.
- **It cannot be worked around by filling in fields.** Setting `validation_state=confirmed` would
  violate `D-AP-05` (confirmed only via the M04 channel); setting `status` to anything other than
  ACTIVE means inventing a status value the enum does not contain.

The tension itself is real design intent: `D-AP-05` says only M04 may write `confirmed`, and the
schema tries to enforce that statically by forbidding `ACTIVE` before confirmation. But a memory has
to *exist* before M04 reviews it, and the design provides no state for that moment.

### Options

| # | Option | Consequence |
|---|---|---|
| **A** | Relax `allOf[1]`; move "confirmed only via M04" from a **static invariant** to a **transition guard** | Preserves both intents: a memory can exist as ACTIVE+unverified, and the *transition* to confirmed is still M04-only. Requires one schema edit plus a stated rule. |
| B | Change the default `status` to a pre-confirmation value | The status enum has no such value (`ACTIVE/SUPERSEDED/ARCHIVED/DELETED`), so this needs a new enum member — a bigger change than A |
| C | Change the default `validation_state` to `disputed` | Semantically wrong: a fresh memory is not disputed |
| D | Require every create to supply `confirmed` | Violates D-AP-05 |

**Recommendation: A.** It is the only option that keeps both the M04-only rule and the ability to
create a memory before review. It is a **design decision**, so it is escalated rather than applied.

### Related, but not the same

`evidence.schema.json` had the opposite shape (too strict on fields the authority requires); this one
is too strict on a *combination* of fields the authority itself defaults to. Both are the same class
of error: a static schema constraint written to enforce a *process* rule.
