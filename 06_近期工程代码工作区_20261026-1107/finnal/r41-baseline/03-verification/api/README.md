# 03-verification/api — TECH-API-01 API contract harness

**This area executes the REST contract** in `01-baseline-corpus/design-package/contracts/api/api-contracts.md §1`
against a real HTTP server on a real loopback socket, validates the real responses with a real
draft-07 engine, and records one evidence row per executed check in `06-evidence/api.jsonl`.

> **SCOPE LABEL — read before quoting anything from here.**
> A `PASS` in this area is evidence about **the contract text and this harness**. It is **not** a
> product `PASS`. There is no M01–M05 implementation in this workspace, none was written, and
> `PRODUCT_IMPLEMENTATION_AUTHORIZATION` remains `NOT_AUTHORIZED` (`02-shared/00-PARENT-MANDATE.md §1`).
> Everything below runs against `mock-server.cjs`, which is a test double that this area owns.

## 1. Files

| File | Role |
|---|---|
| `mock-server.cjs` | the HTTP server: 8 documented endpoints, uniform envelope, context parser, fault injection for all 16 error codes, permission provider (ALLOW/DENY/INDETERMINATE/THROW/TIMEOUT), latency + server deadline. `node:http` only — there is no npm/pnpm on this machine, so no Express. |
| `api-contract-tests.cjs` | the suite. 63 executed checks; writes `06-evidence/api.jsonl` (rewritten per run) and `06-evidence/raw/api/summary.json`. |
| `run-all.cjs` | entry point: server selftest → suite → ledger audit. One exit code. |
| `lib/validator.cjs` + `lib/jsonschema_worker.py` | primary draft-07 engine: python `jsonschema` 4.26.0 driven over a JSON-line pipe. Real `allOf`/`if-then-else` and cross-file `$ref`, which the contract schemas actually use. |
| `lib/jsonschema-subset.cjs` | pure-JS fallback + independent second opinion. Used only if the python worker cannot start, and always cross-checked (`harness.validator_engine_agreement`). |
| `lib/evidence.cjs` | `06-evidence/api.jsonl` writer enforcing `02-shared/EVIDENCE-FORMAT.md`. |
| `schemas/envelope.schema.json` | **harness transcription** of the envelope from §0. No envelope schema is shipped in `contracts/schemas/`; that absence is itself a finding (`harness.envelope_schema_is_harness_local`). |
| `tools/audit-ledger.cjs` | self-audit of the evidence ledger (mandatory fields, unique ids, parse integrity). |
| `tools/provenance-probe.cjs` | read-only byte comparison of the whole 121-file corpus snapshot against the canonical package; exits 1 on any difference (see `TASK-RESULT.md` F7). |
| `tools/list-results.cjs` | prints the per-check table from `06-evidence/raw/api/summary.json`. |

## 2. Run

```powershell
cd E:\[PROJECT]\r41-baseline\03-verification\api
node run-all.cjs          # everything; exit 0 = all checks ran and passed, 1 = a check failed, 2 = harness error

node mock-server.cjs --selftest               # boot the server alone and hit two endpoints
node mock-server.cjs --port=0                 # leave it running for manual curl-style probing
node api-contract-tests.cjs                   # suite only
node tools/audit-ledger.cjs                   # ledger integrity only
```

The python engine needs `PYTHONPATH` pointing at the `pydeps` drop; `lib/validator.cjs` sets it from
`MC_PYDEPS` (default: the package's `evidence/second-round/pydeps`). If it cannot start, the run
proceeds with the JS subset and **every schema verdict is weaker** — the first evidence row says which
engine ran, and `engine` is repeated in `06-evidence/raw/api/summary.json`.

## 3. Wire format implemented

### 3.1 Endpoints (§1)

| §  | Endpoint | Mock behaviour |
|---|---|---|
| 1.1 | `POST /api/v1/memories/retain` | accepts items → `data.accepted[]` candidate ids + `data.rejected[{item_index,code,reason}]`; never writes `active` |
| 1.2 | `POST /api/v1/memories/recall` | request validated against `retrieval-request.schema.json`; response `data.retrieval` (retrieval-response) + `data.context` (context-response) |
| 1.3 | `POST /api/v1/memories/confirm` | `[FUTURE]` → always `403 PERMISSION_DENIED` (M04 review channel does not exist) |
| 1.4 | `POST /api/v1/memories/revise` | optimistic concurrency on `expected_version` → `VERSION_CONFLICT` on mismatch |
| 1.5 | `POST /api/v1/memories/revoke` | `mode` **required**; `availability` (reversible) vs `authorization` (destructive) |
| 1.6 | `GET /api/v1/memories/{id}` | returns a full `memory-object.schema.json` object; `?version=n` supported |
| 1.7 | `GET /api/v1/tasks/{id}` | SM-5 five-state task object (no shipped schema — only the state vocabulary is checked) |
| 1.8 | `GET /api/v1/health` | 8 health fields, no memory content ever; `degraded`/`unavailable` states |

### 3.2 Request context (§0)

All 8 fields are read from headers (the "trusted adapter" channel), with a `body.context` fallback:

```
x-request-id  x-tenant-id  x-actor-type  x-actor-id  x-subject-id
x-session-id  x-scope (comma list)  x-timestamp  x-idempotency-key
```

`x-scope` is a **comma list** because §0 declares `scope[]`. Accepting a bare string there would
paper over defect (a); the parser explicitly rejects a bare string supplied via `body.context.scope`
and reports it as `context.scope`.

`idempotency_key` is required on the four write endpoints (`retain/revise/revoke/confirm`) and
optional on reads (§0 says "all endpoints"; §1.2 says optional for `recall` — the mock follows the
stricter reading only where §1 says the key is mandatory).

### 3.3 Control headers (test harness only — NOT part of the contract)

| Header | Effect |
|---|---|
| `x-mock-fault: <CODE>` | return that canonical error code with its canonical seven attributes |
| `x-mock-permission` (constructor option `permission`) | `ALLOW` / `DENY` / `INDETERMINATE` / `THROW` / `TIMEOUT` |
| `x-mock-latency-ms: <n>` | delay the handler (slow endpoint) |
| `x-mock-deadline-ms: <n>` | override the server-side deadline (default 10 s, the contract's `[EXP-DEFAULT]`) |

### 3.4 HTTP status mapping — harness-defined, because the contract defines none

`contract.http_status_mapping_absent` establishes that no error code has a contract-defined HTTP
status (the only status in the documents is `200` for a degraded `/health`). The mock's mapping is a
proposal, not contract text:

| code | status | code | status |
|---|---|---|---|
| VALIDATION_ERROR, BUDGET_EXCEEDED | 400 | STORAGE_ERROR, INDEX_ERROR, EMBEDDING_ERROR, CONFIGURATION_ERROR, INTERNAL_ERROR, RETRIEVAL_FAILED | 500 |
| PERMISSION_DENIED | 403 | PERMISSION_ERROR, UNAVAILABLE | 503 |
| NOT_FOUND, TASK_NOT_FOUND | 404 | TIMEOUT | 504 |
| VERSION_CONFLICT, DUPLICATE_REQUEST | 409 | oversized body | 413 |

## 4. Interpretation decisions (contract silent → harness chooses, and says so)

1. **Duplicate idempotency key, same payload** → one effect, first result returned, `meta.replayed=true`,
   `errors: []`. Same key **with a different payload** → `409 DUPLICATE_REQUEST`, no second effect.
   The contract only says "return the original result"; refusing conflicting reuse is the fail-closed
   reading.
2. **Permission failure mapping.** `DENY` and `INDETERMINATE` → `PERMISSION_DENIED`; provider
   `THROW`/`TIMEOUT` → `PERMISSION_ERROR`. Both carry `meta.fail_closed=true`. The tests assert the
   *invariant* (closed door, no data, explicit failure), not this particular choice of code, because
   the contract does not fix it.
3. **Degradation.** With no approved degradation policy in the package, a failed channel yields
   `status=failed` + `RETRIEVAL_FAILED`; `status=partial` is only reachable by an explicitly
   unlicensed harness override (`allow_unlicensed_partial`), which stamps
   `meta.degradation_policy="UNLICENSED(forced by harness)"`.
4. **`revoke` without `mode`** → `400 VALIDATION_ERROR(field=mode)`. The mock refuses to invent the
   OI-03-01 default; defaulting to `authorization` would silently delete data.
5. **Version mismatch** → `404` + canonical code + `meta.supported_versions=["v1"]` and never data.
   §9 defines no mismatch behaviour at all, so the assertion is only "refuse cleanly".
6. **Oversized body** (1 MiB, harness config) → `413` + `VALIDATION_ERROR`. The contract states no
   size limit.
7. **`/health` requires the full context.** A literal §0 reading forces identity onto the health
   endpoint; the harness implements it and records the smell (`contract.health_requires_full_identity_context`).

## 5. What this harness does NOT do

- No product code, no real M05 engine, no real SQLite/LanceDB/embedding backends: the channel status
  is *configured*, not *measured*.
- No durability of the idempotency store, no restart/crash window, no concurrent-request race on one key.
- No TLS, auth, CORS, rate limiting, pagination — none are in the contract.
- No load or latency benchmarking; timing is measured only where a timeout threshold is the subject.
- No cancellation semantics (contract silent).
- No multi-tenant/identity isolation testing — that is `03-verification/identity/`'s scope.

Numbers, verdicts and the list of what did not run: see `TASK-RESULT.md`.
