# TASK-RESULT — TECH-API-01: API contract harness (executed)

**Area:** `03-verification/api/`
**Baseline:** Memory Core R4.1, workspace `E:\[PROJECT]\r41-baseline`
**Target:** `01-baseline-corpus/design-package/contracts/api/api-contracts.md §1`, `contracts/error-codes.md`,
`contracts/schemas/*.json`
**Runtime:** Node v24.21.0, Windows NT 10.0.26200, plain CommonJS, `node:http` only (no npm/pnpm)
**Validation engine:** python `jsonschema` **4.26.0** (real draft-07), driven from Node over a JSON-line pipe
**Run:** `node run-all.cjs` — one run, 2026-11-05T12:45:12Z, suite wall time 1162 ms

> ## SCOPE LABEL
> **A PASS here is evidence about the CONTRACT TEXT and about this harness. It is not a product PASS.**
> Every request went to `03-verification/api/mock-server.cjs`, a test double owned by this area. No
> M01–M05 code exists in this workspace and none was written; `PRODUCT_IMPLEMENTATION_AUTHORIZATION`
> stays `NOT_AUTHORIZED`. Nothing under `01-baseline-corpus/` was modified.

---

## 1. Real counts

| Measure | Value |
|---|---|
| Executed checks (one evidence row each) | **63** |
| `PASS` | **62** |
| `FAIL` | **1** — `EV-API-058 defect.d_schema_admits_v1_aliases`, and it is a **refutation of a parent finding**, not a harness defect (§5d) |
| `PARTIAL_PASS` / `NOT_RUN` / `BLOCKED` inside the suite | 0 / 0 / 0 |
| Mock-server boot selftest | PASS (exit 0) |
| Evidence-ledger self-audit | PASS — 63 unique ids, no missing mandatory fields, no unparseable lines |
| Suite exit code | **1** (because of the one refutation row) |
| Harness exit code (`run-all.cjs`) | **1** |
| Evidence | `06-evidence/api.jsonl` |
| Raw logs | `06-evidence/raw/api/` (`run-all.*.log`, `api-contract-tests.*.log`, `mock-server-selftest.*.log`, `ledger-audit.*.log`, `summary.json`, `run-all.summary.json`) |

Coverage by area (from `06-evidence/raw/api/summary.json`):

| Area | Checks | Result |
|---|---|---|
| harness self-tests (is the instrument real?) | 4 | 4 PASS |
| endpoint envelope + payload conformance (§1.1–§1.8) | 8 | 8 PASS |
| the 16 canonical error codes (injected) | 16 | 16 PASS |
| error codes reached through real handler paths | 6 | 6 PASS |
| idempotency | 2 | 2 PASS |
| permission tri-state + fail-closed + mutation control | 7 | 7 PASS |
| timeout (server deadline, client abort, retryability) | 3 | 3 PASS |
| version negotiation | 3 | 3 PASS |
| boundary inputs | 5 | 5 PASS |
| the four parent defects | 5 | 4 PASS, **1 FAIL (refutation)** |
| cross-check of the parent report's own string | 1 | 1 PASS |
| contract coverage gaps | 3 | 3 PASS |

## 2. What actually ran

1. `mock-server.cjs` was started on an ephemeral loopback port and exercised over real HTTP
   (`node:http` client, no mocks of the transport).
2. Every response was validated against a real draft-07 schema file with `jsonschema` 4.26.0 —
   including `allOf`/`if-then-else` (`memory-object.schema.json`) and cross-file `$ref`
   (`retrieval-response.schema.json → error.schema.json`), which a subset validator would have
   silently skipped.
3. The suite validated its own instrument first (`harness.validator_engine_primary`,
   `harness.validator_engine_agreement`): a second, from-scratch JS reader agreed with the python
   engine on 8/8 probe instances, so a verdict is not an artefact of one implementation.
4. A deliberate mutant (`permission_bug:true`, drops a DENY on the floor) was driven through the same
   assertion that the DENY tests use, and it **leaked the canary** (`EV-API-043`). That is mutation
   testing: the DENY checks would fail if enforcement were removed, so they are not vacuous.
5. The suite caught a real defect **in this harness** on its first run: `error.natural.UNAVAILABLE`
   returned `200` because `health_status` was never wired into the mock config. It was a genuine
   finding (recorded in `06-evidence/raw/api/api-contract-tests.2026-11-05T12-39-48-504Z.log`, run 1:
   60/63 PASS) and was fixed before the final run. It is reported here rather than quietly dropped.

### Endpoint coverage (§1.1–§1.8) — all 8 return a schema-valid envelope

| Endpoint | Asserted | Evidence |
|---|---|---|
| `POST /memories/retain` | envelope valid, `request_id` echoed, one uuid accepted, the **stored candidate** validates against `candidate.schema.json` | EV-API-004 |
| `POST /memories/recall` | request validates against `retrieval-request.schema.json`; `data.retrieval` against `retrieval-response.schema.json`; `data.context` against `context-response.schema.json` | EV-API-005 |
| `POST /memories/confirm` | `[FUTURE]` → 403 `PERMISSION_DENIED` as §1.3 documents | EV-API-006 |
| `POST /memories/revise` | optimistic concurrency, v1→v2 | EV-API-007 |
| `POST /memories/revoke` | `mode=availability` → suspended, reversible | EV-API-008 |
| `GET /memories/{id}` | payload validates against `memory-object.schema.json` (28-field object) | EV-API-009 |
| `GET /tasks/{id}` | SM-5 five-state vocabulary; **no shipped schema exists** | EV-API-010 |
| `GET /health` | 8 documented health fields, no memory content in the body | EV-API-011 |

### Error codes

All **16** canonical codes were produced (`EV-API-012` … `EV-API-027`) and each error object validated
against `contracts/schemas/error.schema.json` with its canonical `category`/`retryable`/`severity`.
Injection alone would be a sham, so **six** codes were additionally reached through real handler
paths: `VALIDATION_ERROR` (empty items), `NOT_FOUND` (unknown id), `TASK_NOT_FOUND`,
`RETRIEVAL_FAILED` (channel failure with no approved degradation), `BUDGET_EXCEEDED` (token budget),
`UNAVAILABLE` (health down). `PERMISSION_DENIED` / `PERMISSION_ERROR` / `TIMEOUT` /
`DUPLICATE_REQUEST` / `VERSION_CONFLICT` also occur naturally in their own sections.
`EMBEDDING_ERROR`, `STORAGE_ERROR`, `CONFIGURATION_ERROR`, `INTERNAL_ERROR`, `INDEX_ERROR` are
**injection-only** in this harness (no embedding backend, no SQLite authority, no startup path).

### Idempotency

- Same key twice → `state.effects.retain == 1`, identical `data.accepted`, `meta.replayed=true` (EV-API-034).
- Same key, different payload → `409 DUPLICATE_REQUEST`, still one effect (EV-API-035). The contract
  does not cover this case; refusing is the harness's fail-closed reading and is labelled as such.

### Permission fail-closed

`ALLOW` is the control (data served). `DENY`, `INDETERMINATE`, `THROW`, `TIMEOUT` **all** returned a
closed door: non-2xx, `status=failed`, empty `data`, no canary in the response, `meta.fail_closed=true`
(EV-API-036 … EV-API-040). `INDETERMINATE` never appeared as `ALLOW` anywhere in the body
(EV-API-041). `m05_mode=contract_deny` denied all three sampled endpoints with
`M05_NOT_IMPLEMENTED` (EV-API-042).

### Timeout

Server deadline 200 ms vs 1200 ms handler → `504 TIMEOUT` in **205 ms** (EV-API-044). Client abort at
300 ms against a 1500 ms endpoint → observed as `client_timeout=true` at **315 ms**, no body
(EV-API-045). `TIMEOUT` carries `retryable=true`, `severity=warning` (EV-API-046).

### Version negotiation

`/api/v2/...` → `404`, `status=failed`, `data: {}`, canonical code, `meta.supported_versions=["v1"]`
(EV-API-047). Unsupported `x-api-version` on a v1 path → same refusal (EV-API-048). §9 of the contract
names **no** error code and **no** HTTP status for a mismatch (EV-API-049), so those two tests can only
assert "refuses cleanly", not "matches the contract".

### Boundaries

8/8 mandatory context fields independently enforced (EV-API-050); empty strings
(`content:"   "`, `x-tenant-id:""`) rejected (EV-API-051); 1.5 MB body → `413 VALIDATION_ERROR`, no
socket reset (EV-API-052); 4/4 type confusions rejected with the field named (EV-API-053); malformed
JSON and `{}` body → `400 VALIDATION_ERROR` (EV-API-054).

## 3. The four parent defects — verdicts

| # | Parent finding | Verdict | Evidence |
|---|---|---|---|
| a | `scope` is an array in the request context but a singular string on entities | **CONFIRMED** | EV-API-055 |
| b | `POST /memories/revoke` requires `mode` whose default is TBD (OI-03-01) | **CONFIRMED** | EV-API-056 |
| c | `status:"partial"` is declared but no approved degradation policy licenses it | **CONFIRMED** | EV-API-057 |
| d | `error.schema.json` cannot enforce canonical-code normalisation because it also admits v1 aliases | **REFUTED as stated** — the shipped enum is canonical-only and rejects every alias. The real defect is a **false statement in `error-codes.md §2`** | EV-API-058 (FAIL = claim did not reproduce), EV-API-059 (relocated defect CONFIRMED) |

**(a) CONFIRMED.** `api-contracts.md §0` writes `scope[]`; `permission-context.schema.json` uses
`scopes_requested` (array); `memory-object`, `memory-unit`, `evidence` and `candidate` all use a
**singular string** `scope`/`proposed_scope`. Operationally: a context with
`scope=["owner_private","group_shared"]` and no per-item `scope` has **no contract rule** for which
single scope the new entity gets — the mock refuses (`400`, reason quotes the collision). With an
explicit `item.scope`, two requested scopes silently collapse to one stored scope. One name, two
shapes, no mapping rule.

**(b) CONFIRMED.** §1.5 states `mode` is required and that the default is TBD (OI-03-01). Revoke
without `mode` cannot be implemented from the text; the mock refuses rather than guessing. The two
modes are materially different: `availability` → `reversible=true`, `lifecycle_state` unchanged;
`authorization` → `lifecycle_state=deleted`, `reversible=false`, SM-7 deletion propagation. Guessing
this default is a data-loss decision, not an implementation detail.

**(c) CONFIRMED.** The envelope admits `partial` (and a forced `partial` response is schema-valid),
`error-codes.md §3.5` requires an **approved** degradation policy for partial results, and no file in
the package matches `/degrad/i`. With one channel failing and no policy, the honest answer is
`status=failed` + `RETRIEVAL_FAILED`, which is what happens by default. `partial` is reachable only
through a harness override that stamps `meta.degradation_policy="UNLICENSED(forced by harness)"`.

**(d) REFUTED — and this matters more than the original finding.** The claim was that
`error.schema.json` admits both canonical codes and v1 aliases, so it cannot enforce normalisation.
The shipped file's `code` enum holds **exactly the 16 canonical values** and none of the 7 aliases;
every alias probe (`INVALID_ARGUMENT`, `UNAUTHORIZED`, `FORBIDDEN`, `VALIDATION_FAILED`,
`STORAGE_UNAVAILABLE`, `INDEX_UNAVAILABLE`, `EMBEDDING_FAILED`) is **rejected** at `$.code`
(EV-API-058). So the schema *does* enforce the normalisation rule — it is stricter than the report
says. The defect relocates to `error-codes.md §2`, which asserts 「`error.schema.json` 的 enum 已包含
两侧名称全集」 ("the enum already contains the full name set of both sides"). That sentence is
**false** (EV-API-059). Consequence: a consumer who trusts §2 builds a permissive validator that the
real schema contradicts, and the alias table's practical purpose (tolerating legacy producers)
disappears without anyone noticing. Recommendation: fix the sentence in `error-codes.md §2`, and if
legacy tolerance is genuinely needed, ship a separate permissive inbound schema rather than describing
one that does not exist.

## 4. Other real findings (not in the parent's list)

| # | Finding | Evidence |
|---|---|---|
| F1 | **No response-envelope schema is shipped.** The envelope exists only as prose + a JSON snippet in §0, so envelope conformance is measured against a harness transcription (`03-verification/api/schemas/envelope.schema.json`). That is a weaker claim than "validates against the contract". | EV-API-003 |
| F2 | **No HTTP status mapping exists.** No 4xx/5xx status appears in either document; the only status mentioned is `200` for a degraded `/health`. Every status asserted by this harness is a harness decision. | EV-API-062 |
| F3 | **Schema coverage is 1 of 8 endpoints.** Only `recall` has shipped input *and* output schemas. `getMemory` reuses an entity schema as a payload schema. The other 6 endpoints' payload assertions are hand-written against prose. | EV-API-061 |
| F4 | **§9 defines no version-mismatch behaviour** — no code, no status, no negotiation rule. | EV-API-049 |
| F5 | **A literal §0 reading forces a full identity context onto `/health`** (tenant, actor, subject, session, scope, idempotency key). The harness implements it; the design smell needs an explicit decision, not an implicit one. | EV-API-063 |
| F6 | **The parent report misquotes a `reason_code`**: `API-CONSISTENCY-REPORT.md §2` says `M05_NOT_INITIALIZED`; the contract and the mock say `M05_NOT_IMPLEMENTED`. Cosmetic, but it is a misquote inside an evidence document. | EV-API-060 |
| F7 | **Provenance deviation in the immutable corpus** (found while checking that I had not written to it): `01-baseline-corpus/design-package/contracts/schemas/read-response.schema.json` is **not byte-identical** to the canonical package — the snapshot has lost the UTF-8 BOM (30425 vs 30428 bytes; `canonical[3..] === snapshot` byte-for-byte, all 1847 CJK characters intact, valid JSON). File mtime `2026-11-05T12:11:26Z`, i.e. **before this session started**; a prior process rewrote it after reading it as UTF-8. Not caused by this area, does not affect this harness (it reads `retrieval-response`/`context-response`, not `read-response`), but it means the snapshot is no longer byte-identical to the canonical package, contrary to `00-PARENT-MANDATE.md §0`. | `tools/provenance-probe.cjs` output (`sha256 snap=36e602c4…` vs `canon=3b41ee80…`) |

## 5. Everything that failed, plainly

**One** check failed: `EV-API-058 defect.d_schema_admits_v1_aliases`.

- **Expected** (the parent's claim): every v1 alias validates against `error.schema.json`.
- **Actual**: `0/7` aliases validate; `"INVALID_ARGUMENT"` is rejected at `$.code` — `'INVALID_ARGUMENT'
  is not one of ['VALIDATION_ERROR', 'NOT_FOUND', …]`. The enum has 16 canonical values, parsed
  independently from `error-codes.md §1` (16) and compared programmatically.
- **Why it stays a `FAIL`**: the record's `expected` is the claim under test. Rewriting it to match
  what was observed would be exactly the "weaken the expectation until it passes" move this task
  forbids. The refutation is the deliverable, and `metrics.claim_verdict="REFUTED"` plus the
  `conclusion` field state it unambiguously.

No other check was weakened, skipped, or retried until green. The one earlier failure
(`error.natural.UNAVAILABLE`, run 1) was a real harness bug and is disclosed in §2.5.

## 6. What did NOT run, and why

| Not run | Reason | What it would need |
|---|---|---|
| Any test against a real Memory Core server | No M01–M05 implementation exists; product implementation is `NOT_AUTHORIZED` | an authorized implementation |
| Real M05 permission engine decisions | C40: the engine is not pluggable; only the mock provider and `m05_mode=contract_deny` were exercised | the real engine |
| Real SQLite authority / LanceDB / embedding behaviour behind the endpoints | `STORAGE_ERROR`, `INDEX_ERROR`, `EMBEDDING_ERROR`, `CONFIGURATION_ERROR`, `INTERNAL_ERROR` were **injection-only** | the real subsystems (covered separately in `03-verification/{sqlite,outbox,lancedb}`) |
| Idempotency durability across restart/crash; concurrent same-key race | The store is in-memory and the suite is sequential | a restart harness + concurrent clients |
| Cancellation / abort semantics server-side | The contract is silent; only client-side abort was observed | a contract rule first |
| Real `M04` confirm path | M04 is `[FUTURE]`; the endpoint is documented to fail | M04 |
| Load, latency, throughput measurement | Out of scope: timing was measured only where a timeout threshold was the subject | a benchmark harness |
| TLS, authn/authz transport, CORS, rate limiting, pagination | Not in the contract | contract additions |
| Multi-tenant / identity isolation through the API | Owned by `03-verification/identity/` | (already covered there) |

## 7. What the parent must decide

1. **Accept the (d) refutation** and correct `04-contracts/API-CONSISTENCY-REPORT.md §4` and §7
   ("four defects" becomes "three defects + one false statement in `error-codes.md §2`"). This changes
   a document the parent owns, so it is the parent's call, not mine.
2. **`error-codes.md §2` wording** — delete or correct the "enum contains both sides" sentence, or ship
   the permissive inbound schema the sentence implies. Contract edits belong to Agent E / the parent.
3. **Add `contracts/schemas/response-envelope.schema.json`?** Without it, envelope conformance can
   never be more than harness-transcribed (F1).
4. **Adopt or replace the harness HTTP status mapping** (F2). Right now the contract has none and any
   client implementer will invent their own.
5. **Schema coverage for the 7 uncovered endpoints** (F3) — at minimum `retain`, `revise`, `revoke`,
   `getTask`, `health`.
6. **`scope` vs `scopes_requested`** — the parent already recommends the rename; it is blocked on a
   contract edit, and this harness will fail loudly if the entity side ever starts accepting arrays.
7. **OI-03-01 (`revoke` default)** — needs a user ruling. The harness deliberately refuses to guess.
8. **`partial`** — either approve a degradation policy or mark `partial` `[CONTRACT]`-only.
9. **`/health` identity context** (F5) — exempt health from §0, or state why it needs identity.
10. **Fix the `M05_NOT_INITIALIZED` misquote** in the parent report (F6).
11. **Restore `read-response.schema.json` in the corpus** (F7). It is content-identical to the canonical
    file minus the UTF-8 BOM. I did not touch it — `01-baseline-corpus` is immutable to this area — so
    the restore is the parent's call. `node 03-verification/api/tools/provenance-probe.cjs` re-checks
    the whole 121-file snapshot against `C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design` and exits
    1 while any byte differs.

## 8. Reproduce

```powershell
cd E:\[PROJECT]\r41-baseline\03-verification\api
node run-all.cjs
# expect: mock-server selftest PASS, ledger audit PASS, suite 62/63 PASS with the single
#         documented refutation at EV-API-058, run-all exit 1
```

Individual steps: `node mock-server.cjs --selftest`, `node api-contract-tests.cjs`,
`node tools/audit-ledger.cjs`, `node tools/list-results.cjs`.

The python engine needs `PYTHONPATH` = the package's `evidence/second-round/pydeps`
(`lib/validator.cjs` sets it; override with `MC_PYDEPS`). If it is missing, the run still completes on
the JS subset but `harness.validator_engine_primary` fails and every schema verdict is weaker — that is
by design, so a degraded run cannot masquerade as a full one.
