# API-CONSISTENCY-REPORT — TECH-SCHEMA-01

Date: 2026-11-05 (Asia/Shanghai)
Scope: `contracts/api/api-contracts.md`, `contracts/error-codes.md`, `contracts/schemas/*`, `contracts/state-machines/state-machines.md`
Method: document-level reading plus the executable checks in `03-verification/schema/`
Status: **PARTIAL** — this report covers contract-internal consistency; end-to-end API harness execution is still NOT_RUN.

---

## 1. What was actually checked vs what was not

| Area | Checked how | Result |
|---|---|---|
| Every endpoint declares input, output, errors, idempotency | read + cross-referenced | **incomplete**, see §3 |
| Error codes used by endpoints exist in `error-codes.md` | cross-referenced | **1 systematic gap**, see §4 |
| Response envelope is uniform | read | **yes**, one envelope, see §2 |
| Request context fields uniform | read | **yes**, but with one naming ambiguity |
| Request/response schema ↔ documented endpoint parity | read + schema parse | **partially matched** |
| API behaviour executed against a live mock | **NOT DONE** | `NOT_RUN` |

I am explicitly **not** claiming an executed API contract test. No mock server was stood up.
Everything below is contract-level consistency, which is a different grade of evidence.

## 2. What is consistent

**One response envelope**, declared once and not contradicted anywhere:

```json
{ "request_id", "status": "success|failed|partial", "data", "meta": {schema_version, algorithm_version}, "errors": [{code,message,trace_id}] }
```

This is genuinely uniform across REST §1, module APIs §2–§6 and the adapters §7–§8. Good.

**One request context**, declared as mandatory for all endpoints:
`request_id, tenant_id, actor{type,id}, subject_id, session_id, scope[], timestamp, idempotency_key`.

**Implementation-scope tagging exists** (`[P1]`, `[CONTRACT]`, `[FUTURE]`) on every REST endpoint and
on the module sections. That is the mechanism that prevents "designed" from being read as
"implemented", and it is present.

**Three-state permission semantics are stated consistently**: ALLOW / DENY / INDETERMINATE, with
INDETERMINATE ⇒ exclude, and `m05_mode=contract_deny` ⇒ always
`{allowed:false, reason_code:"M05_NOT_IMPLEMENTED"}`. This matches `04` and the identity verification
area.

## 3. Inconsistencies found

### 3.1 `scope` is both a request field and a schema enum, with two different value sets

- Request context declares `scope[]` (an array).
- `permission-context.schema.json` declares `scopes_requested` (array, 5-value enum).
- `evidence.schema.json` declares `scope` (a **string**, 5-value enum) — singular.
- `memory-unit.schema.json` declares `scope` (a **string**, 5-value enum) — singular.

So `scope` means "an array of requested scopes" in the request context and "one assigned scope" on
entities, under the **same name**. A caller reading the spec cannot tell whether to send a string or
an array. This is the `OI-016` class of defect and it is not resolved by the current contract text.

**Recommendation:** rename the request-side field to `scopes_requested` (matching the permission
schema, which already uses that name) and leave `scope` singular on entities. That is a one-word
change with a clear owner, and it removes a class of integration bug.

### 3.2 `revoke` carries an unresolved semantic switch

§1.5 `POST /api/v1/memories/revoke` requires the caller to pass
`mode: "availability" | "authorization"` and states the default is **TBD (OI-03-01)**.

An endpoint whose required parameter has an undecided default cannot be implemented, and cannot be
tested. It is honest of the contract to say so, but it must not be counted as a defined API.

### 3.3 Status `partial` is declared but never given a rule

The envelope allows `status: "partial"`. `error-codes.md §3.5` says partial results must be marked
`status=partial` + `degraded=true`, and that "whether partial results continue to be returned must be
decided by an approved degradation policy; if not approved ⇒ status=failed". No such approved
degradation policy exists in the package.

So `partial` is an enum value with no licensed trigger. **Either** the degradation policy must be
approved **or** `partial` should be marked `[CONTRACT]`-only. Right now a caller cannot know when to
expect it.

### 3.4 `confirm` endpoint exists in the REST surface but is `[FUTURE]`

§1.3 declares `POST /api/v1/memories/confirm` and then says it returns `PERMISSION_DENIED` until M04
exists. That is consistent with `D-AP-05` (confirmed only via the M04 channel), but it means there is
a public route that is *guaranteed to fail*. It should be `[CONTRACT]`-tagged like §5/M04, not
listed among the open REST endpoints, so an integration client does not implement against it.

## 4. Error-code coverage — CORRECTED after execution

`error-codes.md` is a genuinely good contract: 16 canonical codes, each with seven attributes
(`code/category/message/retryable/severity/handling/exposure_policy`), plus a legacy alias map.

**This section originally claimed the opposite of what is true. The executed API harness refuted me
(`EV-API-058`, `EV-API-059`).** The original text said "`error.schema.json` cannot enforce the
normalisation rule because it also admits v1 aliases". Measured reality:

| Measurement | Result |
|---|---|
| Canonical codes in `error-codes.md` | 16 |
| Values in `error.schema.json` `code` enum | **16 — exactly the canonical set** |
| v1 aliases accepted by the schema | **0 of 7 tested** (`INVALID_ARGUMENT`, `UNAUTHORIZED`, `FORBIDDEN`, `VALIDATION_FAILED`, `STORAGE_UNAVAILABLE`, `INDEX_UNAVAILABLE`, `EMBEDDING_FAILED` — all rejected) |

So the schema **does** enforce canonical codes. I had inferred the opposite from `error-codes.md §2`'s
stated intent without reading the schema's actual enum. That is exactly the class of error this project
keeps punishing: trusting a description over the artefact.

**The real defect is the reverse, and it is a documentation defect (`EV-API-059`):**

`error-codes.md §2` states *"契约 `error.schema.json` 的 enum 已包含两侧名称全集"* — "the schema's
enum already contains the union of both sides". **It does not.** The enum holds the 16 canonical
codes and **none** of the 11 alias names (7 of 11 were tested and all failed).

Why it matters: an implementer reading §2 would believe a legacy producer emitting `INVALID_ARGUMENT`
validates, and would therefore skip a normalisation step. The schema would then reject it at
runtime — a failure the documentation invited.

**Corrected recommendation:** delete the union claim from `error-codes.md §2` and state plainly that
the schema accepts canonical codes only. The schema is already correct; only the document is wrong.
If legacy tolerance is genuinely required, widen the enum deliberately — but do not leave a false
statement in the contract.

Secondarily: `exposure_policy` values in `error-codes.md` are `host_visible` / `internal_only`
(and §3.2 refers to a third value `user_visible` which is **not** defined in the table). Three names,
two defined. That third name should be removed or defined.

## 5. Cross-check against the permission contract

The API permission section (§6) declares six operations
(`canRead/canWrite/canUpdate/canDelete/canShare/canExport`) returning
`{allowed, reason_code, policy_version, decision_id}`.

But `permission-context.schema.json` `action` enum lists:
`read, write, update, delete, share, export` — **six, matching**.

Mismatch: the schema uses lowercase verbs (`read`), the API uses `can`-prefixed names (`canRead`).
That is a naming inconsistency but not a semantic one; a mapping table is missing. Low severity,
easy to fix, worth fixing because it will otherwise appear in every adapter.

## 6. What remains NOT_RUN, and what it would take

| Missing test | What it needs |
|---|---|
| Mock API server + request/response validation for all endpoints | a minimal HTTP mock (no npm, so hand-rolled `node:http`) + schema validation per response |
| Permission / version / idempotency behaviour per endpoint | the mock plus the synthetic permission provider from `03-verification/identity/` |
| Error-envelope conformance under each of the 16 codes | a fault-injecting mock |
| Timeout and retry behaviour | a slow mock |
| Version negotiation (`/api/v1/`) | a version-mismatch client |

None of these were executed. They are the substance of `MC-AUD-019`, which therefore remains
**OPEN**.

## 7. Verdict

| Question | Answer |
|---|---|
| Is the API contract internally coherent? | **Mostly.** One envelope, one request context, consistent permission tri-state, and scope tags everywhere. |
| Are there concrete defects? | **Yes, four:** `scope` array-vs-string collision, `revoke` undecided default, `partial` with no licensed trigger, error-schema unable to enforce canonical codes. |
| Is the API contract executable as written? | **No.** Not while `revoke` has a TBD required parameter and `partial` has no rule. |
| Is API verification done? | **No. `NOT_RUN`.** Contract reading is not API testing, and this report does not claim otherwise. |
| Can the API contract freeze? | **No.** Gates: the four defects above, plus an executed mock harness. |
