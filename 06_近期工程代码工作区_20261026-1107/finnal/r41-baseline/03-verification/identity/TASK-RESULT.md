# TASK-RESULT — TECH-IDENTITY-01 (Agent D)

**Area:** `03-verification/identity/`
**Baseline:** Memory Core R4.1
**Harness:** Node v24.21.0, Windows NT 10.0.26200, plain CommonJS, no npm/pnpm
**Run:** single process, `node run-all.cjs`, **222 executed assertions**
**Verdict:** hardened profile **0 failures**; weak reference profile **6 leaks reproduced on purpose**

> ### SYNTHETIC_IDENTITY_VERIFIED != REAL_HOST_MAPPING_VERIFIED
> All identities are synthetic. No QQ Bridge was contacted, no DSH identity data was read, and no
> real host mapping was observed. **Real-host validation is NOT permitted in this session** (§10).

---

## 0. READ THIS FIRST — the six FAILs are expected negative controls, not defects

The run reports `216 PASS / 6 FAIL`. The six FAILs are **not** product vulnerabilities and **must
not** be counted as open security defects. Every one is produced by the deliberately broken
`weak_v1` reference profile, and every one has a hardened counterpart in the same run that BLOCKS
the identical attack.

| Attack | `weak_v1` result (mutant) | Hardened result (default) | What the contrast proves |
|---|---|---|---|
| Revocation outrun by the cache | **LEAKED** — revoked private body served from cache<br>`cache.weak.stale_after_revocation` · `EV-IDENT-0132` | **BLOCKED** — `status=empty`, 0 items<br>`cache.revocation_not_served` · `EV-IDENT-0126` | Per-hit revalidation against epoch + resource versions is load-bearing, not decorative |
| Memory bodies cached at rest | **LEAKED** — canary found inside cache entries<br>`cache.weak.content_at_rest` · `EV-IDENT-0133` | **BLOCKED** — entries hold `candidate_ids` only<br>`cache.stores_ids_only` · `EV-IDENT-0124` | "Ids, not bodies" is a real requirement |
| Cross-group leak through the cache key | **LEAKED** — G3 memory served in a G1 context<br>`cache.weak.cross_group_via_key` · `EV-IDENT-0134` | **BLOCKED** — same sequence yields no G3 memory<br>`cache.hardened.cross_group_blocked` · `EV-IDENT-0135` | Group/tenant/session/epoch in the key is load-bearing |
| Same bot, two groups | **LEAKED** — G2 memory served in a G1 context<br>`atk.E3.same_bot_two_groups_cache` · `EV-IDENT-0162` | **BLOCKED** — `atk.E1.cross_group_memory` · `EV-IDENT-0160` | 08 §7.4.1 prohibition 4 needs the cache key, not only the engine |
| Evidence stricter than its memory | **LEAKED** — private sentence released in a group context<br>`atk.C5.weak_evidence_leak` · `EV-IDENT-0154` | **BLOCKED** — `atk.C1.evidence_stricter_than_memory` · `EV-IDENT-0150` | The second, per-evidence permission check is load-bearing |
| Missing identity defaulted away | **LEAKED** — `system_internal` returned to an identity-less request<br>`atk.F4.missing_identity_defaults_to_system` · `EV-IDENT-0165` | **BLOCKED** — `denied / IDENTITY_MISSING`, 0 items<br>`atk.F4.hardened_denies` · `EV-IDENT-0166` | "No default actor" is load-bearing |

**Why the mutants exist.** `weak_v1` is a five-shortcut implementation of the same design, written
inside this harness (`lib/retrieval-pipeline.cjs`, `PROFILES.weak_v1`) purely as a negative control:

| Shortcut | Defect class | Design rule it violates |
|---|---|---|
| W1 identity defaulting | missing identity ⇒ synthetic `system` principal | 04 §7.4.1, 08 §7.3.1 |
| W2 cache as authorization | cache hit ⇒ skip the gate | 04 §7.11 item 5 |
| W3 evidence by memory | evidence inherits the memory's answer | 04 §7.11 (evidence is its own resource) |
| W4 degradation widening | engine failure ⇒ return candidate ids | 04 §7.8 (fail closed) |
| W5 no scope pre-filter | every candidate reaches the context builder | 08 §7.4.2 |

A run in which these six records are **PASS** would mean the attacks no longer reproduce — i.e. the
mutant changed. Treat that as a regression in the harness, not as an improvement in security.
This is mutation testing: the value of the suite is that it *detects* a known-bad implementation.

Consequence for automation: `node run-all.cjs` **exits 1 by design** on a healthy baseline, because
six security properties genuinely fail in the weak profile. `expected_exit_code: 1` and
`hardened_profile_failures: 0` are both machine-readable in
`06-evidence/raw/identity/summary.json`.

---

## 1. What was actually executed

```
cd 03-verification/identity
node run-all.cjs            # 222 assertions, exit code 1 (see §0)
```

| Module | Assertions | Failed | Scope |
|---|---|---|---|
| `01-identity-universe` | 40 | 0 | the eleven identity objects, derivation, uniqueness, the ten required identity shapes |
| `02-permission-matrix` | 49 | 0 | 44-row policy matrix (7 scopes × 6 actor classes) + 5 property checks |
| `03-fail-closed` | 32 | 0 | engine failure, timeout, indeterminate, malformed output, degradation |
| `04-cache-revocation` | 19 | 3 | cache freshness, revocation, grant lifetime — **3 leaks (mutant)** |
| `05-bypass-attacks` | 27 | 3 | 22 attacks across 6 families — **3 leaks (mutant)** |
| `06-allow-all-isolation` | 22 | 0 | four independent controls on the test double |
| `07-final-authorization` | 14 | 0 | M02 terminal check, context integrity |
| `08-attribution-audit` | 19 | 0 | session/group attribution, audit completeness, contract conformance |
| **total** | **222** | **6** | `216 PASS / 6 FAIL`, `0` hardened failures |

**Every assertion executed.** Nothing is reported as `NOT_RUN`; there are no skipped or commented-out
checks. `222` is the number of records written to the evidence file by this one run — not an
accumulation (§9).

Reproducibility was measured, not assumed: two consecutive full runs produced **222 records with 0
differences** in `(test_id, status, expected, actual)`.

---

## 2. Deliverables

| Path | What it is |
|---|---|
| `run-all.cjs` | entry point; single Node process, no child processes, deterministic |
| `HOST-IDENTITY-MAPPING-CONTRACT.md` | the eleven identity objects, failure rules, epoch/revocation, host adapter interface, real-host preconditions |
| `README.md` | how to run, what each file is, what the exit code means |
| `tests/01..08-*.cjs` | the eight modules above |
| `lib/*.cjs` | reference implementation of the design contract (gate, store, cache, pipeline, identity provider, audit, isolation) |
| `06-evidence/identity.jsonl` | 222 evidence records, one per executed assertion |
| `06-evidence/raw/identity/assertions.json` | same records, richer (full `detail`, `limitations`) |
| `06-evidence/raw/identity/attacks.md` | **attack / mechanism / result / evidence id** table, generated from the run |
| `06-evidence/raw/identity/permission-matrix.md` | the 44-row matrix as executed |
| `06-evidence/raw/identity/summary.json` | counts, verdict, expected exit code, module table |
| `06-evidence/raw/identity/schema-sha256.txt` | provenance of the two real contract files used |
| `06-evidence/raw/identity/run.log` | full console output |

---

## 3. Fail-closed evidence (the six required proofs)

Each required proof was executed by breaking the engine, not by asserting that hardening exists.

| Required proof | Evidence | Result |
|---|---|---|
| missing identity ⇒ DENY | `fc.identity.missing` `EV-IDENT-0098`; `pm.040` `EV-IDENT-0080`; `atk.F4.hardened_denies` `EV-IDENT-0166` | `IDENTITY_MISSING`, 0 items |
| mapping failure ⇒ DENY | `idn.universe.I07_invalid_mapping` `EV-IDENT-0024`; `pm.041` `EV-IDENT-0081` | `MAPPING_INTEGRITY_FAILURE` |
| permission-service failure ⇒ DENY | `fc.engine.throw` `EV-IDENT-0090`, `fc.engine.timeout` `EV-IDENT-0091` | `DENY / PERMISSION_SERVICE_ERROR` / `…_TIMEOUT` |
| unknown principal ⇒ DENY | `idn.universe.I05_unknown_user` `EV-IDENT-0022`; `pm.039` `EV-IDENT-0079`; `idn.e2e.unknown_never_reaches_gate` `EV-IDENT-0039` | `IDENTITY_UNKNOWN`, denied at the identity stage |
| revoked identity ⇒ DENY | `idn.universe.I08_revoked_identity` `EV-IDENT-0025`; `pm.042` `EV-IDENT-0082`; `idn.e2e.revoked_denied_at_identity` `EV-IDENT-0040` | `IDENTITY_REVOKED` |
| cache stale ⇒ revalidate | `cache.staleness_detected` `EV-IDENT-0127`, `cache.stale_on_epoch_change` `EV-IDENT-0129`, `cache.stale_on_resource_version_change` `EV-IDENT-0130`, `cache.revocation_not_served` `EV-IDENT-0126` | revalidation forced; 0 stale items served |
| final M02 check fail-closed | `fin.released_equals_assembled` `EV-IDENT-0198`, `fin.expired_grant_rejected_by_builder` `EV-IDENT-0200` | released set == assembled set; expired grant dropped with `GRANT_EXPIRED` |
| `INDETERMINATE` never becomes ALLOW | `fc.indeterminate.never_allow` `EV-IDENT-0093` | **7 injections, 0 allows** |

Degradation never widens visibility: on total engine failure the hardened path returns
`status=failed`, `0 items`, `0 degraded_ids` (`fc.degradation.status_failed` `EV-IDENT-0117`); a
single failed item produces `partial` with that item excluded and others present
(`EV-IDENT-0119`).

---

## 4. Permission matrix

44 rows = 7 data-visibility scopes (`owner_private`, `group_shared`, `session_only`,
`project_scoped`, `system_internal`, `export`, deleted) × the actor classes defined in
§Policy HP-1, plus 5 property checks (default deny, `contract_deny`, tenant isolation,
private-in-group, evidence strictness). Executed row by row against the real gate:
**15 ALLOW / 29 DENY across 10 identity shapes** (`pm.100.matrix_shape`, `EV-IDENT-0085`). The full
table is `06-evidence/raw/identity/permission-matrix.md`.

Highlights that are *design consequences*, not incidental:

* `owner_private` + group context ⇒ DENY **even for the owner** (`atk.F1`, `EV-IDENT-0163`,
  `pm.027` `EV-IDENT-0067`).
* `group_shared` needs **group match ∧ membership**; a non-member in a valid group context is denied
  (`EV-IDENT-0161`, `pm.025` `EV-IDENT-0065`).
* `session_only` for a group member requires the session match (**DG-1**: resolved stricter).
* `system_internal` is unreachable for every user role (`pm.016` `EV-IDENT-0056`); only a service
  principal.
* `m05_mode=contract_deny` turns all 15 designed ALLOW rows into `M05_NOT_IMPLEMENTED` denials
  (`pm.101` `EV-IDENT-0086`), and never calls the engine (`pm.102` `EV-IDENT-0087`).

---

## 5. Attack results — 22 attacks, 6 families

Full table with mechanism and evidence id: `06-evidence/raw/identity/attacks.md`.

| Family | Attempts | Blocked | Leaked |
|---|---|---|---|
| A. Retrieval bypass (direct index, no grant, forged decision, replayed grant, request flags, builder injection) | 6 | 6 | 0 |
| B. Cache bypass (poisoned candidate list, tenant key collision) | 2 | 2 | 0 |
| C. Evidence bypass (stricter evidence, raw join, deleted resource, memory grant on evidence, W3 mutant) | 5 | 4 | **1** |
| D. Degradation widening (engine throw, timeout) | 2 | 2 | 0 |
| E. Cross-group (direct, non-member, same-bot-two-groups cache) | 3 | 2 | **1** |
| F. Private → shared context (owner in group, pre-filter bypass, missing identity, sweep of 3 principals) | 4 | 3 | **1** |
| **total** | **22** | **19** | **3** |

Two structural results worth naming:

* **The cache can change *which* candidates are considered, never *whether* they are allowed.**
  A poisoned candidate list naming three private memories produced zero items
  (`atk.B1`, `EV-IDENT-0148`), because every replayed candidate is gated again.
* **The raw join helper is never used on the hardened path.** `store.evidenceInline()` performs no
  permission check by design (it is the hazard being tested); instrumented call count on the
  hardened path is `0` (`atk.C2`, `EV-IDENT-0152`) and `>0` on the mutant (`EV-IDENT-0155`).

---

## 6. Allow-all double isolation

Four independent controls, all executed:

| Control | Evidence | Result |
|---|---|---|
| Call-site guard: scope only from `tests/` | `iso.caller_outside_tests_refused` `EV-IDENT-0169` | `ISOLATION_CALLER_NOT_TEST` for a `lib/` caller |
| Construction guards | `iso.direct_construction_refused`, `iso.gate_refuses_provider` `EV-IDENT-0174` | `ISOLATION_REQUIRED` / `PROVIDER_NOT_PERMITTED` |
| Static closure over the request path | `iso.graph_closure_excludes_double` `EV-IDENT-0179`, `…_excludes_registry` `EV-IDENT-0180` | neither module reachable; registry reference is a lazy require |
| Real config contract | `iso.config.enabled_true_rejected` `EV-IDENT-0184`, `…default_policy_allow_rejected` `EV-IDENT-0185` | both rejected by `security.schema.json` |

And the double cannot widen anything even when it is live: under `contract_deny`, three decisions
all `DENY / M05_NOT_IMPLEMENTED` with **0 engine calls** (`EV-IDENT-0175/0176`). It returns ALLOW
only inside an explicit isolation scope with `mode=implemented` (`EV-IDENT-0177`) — which is the
statement "allow-all is permitted only in an isolated synthetic test".

---

## 7. Design conflicts and ambiguities found

| id | Issue | How it was resolved here |
|---|---|---|
| **DG-2** | `permission-context.schema.json` has **no field for the requester's subject**; `subject_id` is documented as "记忆所属主体". | Gate reads `ctx.subject_id` as the actor's subject and documents the gap. An implementer following the schema literally would compare the actor to itself and allow everything. **Highest-risk ambiguity found.** |
| **DG-1** | 04 §7.4.2 vs 08 §7.4.2 disagree on `session_only` for a Group Member. | Require the session match (stricter). |
| **DG-3** | `SCOPE_MISSING` / `SCOPE_UNKNOWN` / `UNTRUSTED_CONTEXT_SOURCE` guards are shadowed by schema validation, so they are unreachable in practice. | Kept both; the harness asserts the guard exists and notes it is dead code behind the schema. |
| **DG-4** | "Evidence inherits the strictest permission" has no machine-readable comparison. | Explicit strictness order + a second gate call per evidence row. |
| **DG-6** | Group-session attribution (OI-04-03) and revoke semantics (OI-04-05) remain open in the design package. | Conservative reading; flagged, not settled. |

Also missing: `02-shared/TASK-ASSIGNMENT.md`, referenced by the parent mandate, **does not exist**.
This report's section list was derived from the mandate and the brief instead.

---

## 8. Top unresolved security questions

1. **DG-2 is an authorization hole waiting to happen.** If the requester's subject has no schema
   slot, every implementation must invent one; two implementations will invent it differently. This
   needs a contract change before any real engine is pointed at the schema.
2. **In-process isolation is not a security boundary.** Controls #1–#4 stop accidental reachability
   of the allow-all double, but a caller can `require` it. The real control must be that the double
   does not ship, and that must be enforced at packaging time.
3. **No async invalidation story.** The hardened design revalidates on every hit, which is correct
   but expensive; the moment someone "optimises" it into async invalidation the LEAKED rows in §0
   come back. There is no test that would catch that regression except this one.
4. **Revocation semantics are undefined (OI-04-05).** Does a revoked identity lose authorization or
   availability of its own memories? This baseline chooses the conservative reading; the choice is
   reversible but must be made explicitly.
5. **Evidence strictness has no formal order (DG-4).** "Strictest" is currently a convention in one
   file. Two teams will disagree about `project_scoped` vs `group_shared`.
6. **Group session attribution (OI-04-03).** Whether a session belongs to a group or to a user
   determines whether `session_only` can coexist with group sharing. Unresolved in the design.

---

## 9. Evidence provenance

* `06-evidence/identity.jsonl` is **rewritten by every full run** (truncate + write), not appended.
  Its 222 lines are **exactly one run** — the run this document describes.
* A partial run (`node run-all.cjs --only=04`) writes to `partial-04-*.jsonl` /
  `partial-04-summary.json` and **cannot** touch the main file. This was added specifically so the
  evidence file can never be an accumulation of partial runs.
* Records are ordered by module then assertion; `evidence_id` is `EV-IDENT-0001`…`EV-IDENT-0222`,
  assigned in execution order, so the ids in this report are stable for this run.
* Determinism check: two consecutive runs, 222 records, **0 differences**.
* Provenance of the two real contract files (`06-evidence/raw/identity/schema-sha256.txt`):
  `permission-context.schema.json` = `0c5a62fabaf55c59aa1e8af9a012add6bb7d6cdd899ee25135860f37d58da382`,
  `security.schema.json` = recorded in the same file.

---

## 10. Preconditions for real-host validation — and why it is not permitted here

**Real-host validation is NOT permitted in this session.** No statement in any deliverable from
this area may be read as a real-host verification. Everything below is a prerequisite, not a
recommendation:

1. Written authorization from the host data owner, naming tenant, window and reviewer.
2. A reviewed implementation of the adapter interface in
   `HOST-IDENTITY-MAPPING-CONTRACT.md` §10 (`getPrincipal`, `getGroupMembership`,
   `getSessionAttribution`, revocation feed, mapping version).
3. A frozen, redacted identity corpus so the mapping can be replayed offline and diffed.
4. A sandbox tenant for the write paths (revocation, membership change, message delete).
5. An independent reviewer for the DSH side: confirm DSH never receives host ids, and that the
   `qq:` namespace boundary is enforced on receipt, not only on send.
6. A migration/rollback story for `mapping_version` / `mapping_epoch`.
7. A decision on DG-2.

Until then the correct status of every real-host claim is `NOT_VERIFIED`.

---

## 11. What is NOT verified

* Any real QQ Bridge behaviour, event shape, or field semantics.
* Any real DSH identity handling.
* Multi-process, multi-tenant, or persistent-storage behaviour — everything here is in one process
  with in-memory stores.
* Performance, timing side channels, or denial-of-service resistance.
* Encryption at rest, backup replay, and tombstone propagation (out of scope for this area).
* The `export` and `share` scopes (blocked by OI-04-02).
* SQLite / LanceDB interaction (not used by this task).

---

## 12. Reproducibility

```
cd 03-verification/identity
node run-all.cjs                 # full run  -> 222 assertions, exit 1 (6 expected negative controls)
node run-all.cjs --only=03       # one module -> partial artifacts only, main evidence untouched
```

Exit codes: `0` = every check passed (would mean the six mutants stopped leaking — investigate),
`1` = at least one check failed (the state of a healthy baseline, §0), `2` = harness error.
No child processes, no pipes, no network, no npm. The full run takes well under a second.
