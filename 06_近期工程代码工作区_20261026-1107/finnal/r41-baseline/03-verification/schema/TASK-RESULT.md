# TASK-RESULT — TECH-SCHEMA-02 · business fixtures for the contract schemas

Task: `TECH-SCHEMA-02` (MC-AUD-018, remaining half) · Date: 2026-11-05 (Asia/Shanghai)
Workspace: `E:\[PROJECT]\r41-baseline` · Authorisation: `CODE_IMPLEMENTATION_SCOPE` only, no product code.
Evidence: `06-evidence/schema.jsonl` **EV-SCHEMA-919 … EV-SCHEMA-960** (42 records, all `TECH-SCHEMA-02`).
Raw: `06-evidence/raw/schema/fixtures.run1.log` (executed transcript), `fixtures.run1.json` (per-fixture detail).
Harness: `03-verification/schema/fixtures/run-fixtures.cjs` + `_tools/`.
Dataset fingerprint: `sha256 0ab46d95850d1523cf37769fdd8e5f6aa171c547a63b9d09ed3dba1c8025e02b` over 194 fixture files.
Run: `2026-11-05T20:44:03+08:00`, 228 ms, exit code 0.

---

## 1. Result summary

| Metric | Value |
|---|---|
| Contract schemas in scope | 31 (`schemas/` 20 + `config/` 11) |
| Extra schema covered | 1 — `04-contracts/schemas/evidence.schema.json` (v2) |
| Schemas that now have fixtures | **32 / 32** |
| Contract schemas with **no** fixtures | **0** |
| Fixtures executed | **194** |
| Fixtures producing the expected verdict | **194** |
| Fixtures failing | **0** |
| Expected-valid fixtures | 87 |
| Expected-invalid fixtures | 107 |
| Negatives that failed at the asserted keyword **and** instance path | 107 / 107 |
| Boundary fixtures | 51 |
| Fixture files carrying a `finding_id` | 8 |
| Harness self-test | 3 / 3 deliberately broken fixtures flagged (exit 0) |

The 194 figures are one run of `run-fixtures.cjs` against the run described above. A PASS at fixture
level means the schema produced the asserted verdict by the asserted keyword; it does **not** by itself
mean the schema matches the authority — see §5.

**Authority-drift findings proved: 7 defects + 1 known deviation, 1 candidate finding withdrawn.** (§5)

## 2. Per-schema coverage

`valid` / `invalid` are the expected verdicts; `boundary` and `finding` are separate kinds that overlap
them. No schema has fewer than 3 fixtures; the seven schemas that already shipped an `examples` entry
were additionally checked against their own example (`_tools/probe-examples.cjs`: 8/8 examples are
valid — the package is at least self-consistent on its own samples).

| schema | fixtures | valid | invalid | boundary | finding |
|---|---|---|---|---|---|
| `config/anchor.schema.json` | 7 | 3 | 4 | 2 | |
| `config/core.schema.json` | 6 | 3 | 3 | 2 | |
| `config/embedding.schema.json` | 6 | 3 | 3 | 2 | |
| `config/emotion.schema.json` | 6 | 3 | 3 | 1 | |
| `config/memory.schema.json` | 5 | 3 | 2 | 1 | |
| `config/profile.schema.json` | 4 | 2 | 2 | 1 | |
| `config/retrieval.schema.json` | 5 | 3 | 2 | 1 | |
| `config/security.schema.json` | 4 | 2 | 2 | 1 | |
| `config/storage.schema.json` | 4 | 2 | 2 | 0 | |
| `config/test.schema.json` | 3 | 1 | 2 | 0 | 1 |
| `config/worker.schema.json` | 4 | 2 | 2 | 1 | |
| `schemas/anchor-set.schema.json` | 5 | 2 | 3 | 1 | |
| `schemas/candidate.schema.json` | 7 | 3 | 4 | 2 | 1 |
| `schemas/context-response.schema.json` | 5 | 2 | 3 | 2 | |
| `schemas/embedding-space.schema.json` | 6 | 2 | 4 | 3 | |
| `schemas/emotion-context.schema.json` | 9 | 3 | 6 | 2 | |
| `schemas/error.schema.json` | 6 | 3 | 3 | 2 | |
| `schemas/event.schema.json` | 8 | 4 | 4 | 2 | 1 |
| `schemas/evidence.schema.json` (shipped v1) | 6 | 3 | 3 | 1 | 1 |
| `schemas/memory-object.schema.json` | 5 | 3 | 2 | 1 | 1 |
| `schemas/memory-unit.schema.json` | 12 | 4 | 8 | 3 | 1 |
| `schemas/observation.schema.json` | 4 | 2 | 2 | 1 | |
| `schemas/outbox-event.schema.json` | 6 | 3 | 3 | 2 | |
| `schemas/permission-context.schema.json` | 6 | 3 | 3 | 2 | |
| `schemas/person-profile.schema.json` | 7 | 3 | 4 | 2 | |
| `schemas/read-request.schema.json` | 8 | 3 | 5 | 3 | |
| `schemas/read-response.schema.json` | 9 | 4 | 5 | 2 | 1 |
| `schemas/relation.schema.json` | 4 | 2 | 2 | 1 | |
| `schemas/retrieval-request.schema.json` | 8 | 3 | 5 | 3 | |
| `schemas/retrieval-response.schema.json` | 6 | 3 | 3 | 1 | |
| `schemas/search-document.schema.json` | 5 | 2 | 3 | 2 | |
| `04-contracts/schemas/evidence.schema.json` (v2) | 8 | 3 | 5 | 1 | 1 |

The 13 schemas named in the brief as "important" all exceed the minimum and carry boundaries:
memory-unit (12), evidence v1 (6) + v2 (8), event (8), emotion-context (9), person-profile (7),
anchor-set (5), candidate (7), retrieval-request (8), retrieval-response (6), context-response (5),
permission-context (6), embedding-space (6), outbox-event (6).

## 3. Validator, and what a PASS actually rests on

`_tools/validate_instance.py` — **Python 3.14.7 + jsonschema 4.26.0, `Draft7Validator`**, the same
stack the parent used. All 194 instances are batched into one invocation; `run-fixtures.cjs` only
compares verdicts, it never re-implements JSON Schema. Error objects are compared on
`(validator, absolute_path, message)`, so a negative fixture passes only if the named keyword fired at
the named instance pointer.

Two gaps are stated rather than papered over:

- **`format` is not asserted.** Draft-07 treats `format` as an annotation unless a `FormatChecker` is
  configured; the parent recorded this in `SCHEMA-CONSISTENCY-REPORT.md` §6. Enabling one here would
  make negatives fail for a format reason instead of the asserted keyword. `uuid` / `date-time`
  therefore remain **NOT VERIFIED**; where a schema also ships a `pattern`, the pattern *is* asserted.
- **Cross-file `$ref`**: the two `https://memory-core.r41/...` references are resolved through a local
  `$id`→file store (the parent's recommendation #4, implemented rather than left open). It is *proved*
  to be applied, not skipped: `retrieval-response/invalid-03-error-code-inside-ref` is rejected on
  `code` inside the referenced `error.schema.json`.

## 4. Cross-check (brief item 6): does the evidence defect still hold?

Re-established with **independently authored** fixtures, not the parent's:

| Check | Schema | Instance | Verdict | Failing keywords |
|---|---|---|---|---|
| v1 rejects a §3.2 Evidence object | `contracts/schemas/evidence.schema.json` | `schemas/evidence-v1/finding-01-authority-3.2-instance-rejected.json` | **invalid** (asserted) | `additionalProperties` (*original_text*), plus `enum` on `source_type` and `required` on `tenant_id`/`recorded_at` |
| v2 accepts the same object | `04-contracts/schemas/evidence.schema.json` | `schemas/evidence-v2/valid-01-authority-3.2.json` | **valid** | — |

The defect is confirmed and unchanged: `additionalProperties:false` is fatal, so **no** §3.2 Evidence
object can be written against the shipped v1 schema by omitting anything. The corrected v2 is not a
loosening — it still rejects `original_text: ""` (`minLength`), a v1 `source_type` alias (`enum`),
a missing `speaker_id` (`required`) and `scope=group_shared` + `visibility=owner_private` (`const`).

One new *quantified* consequence: v1 and `read-response.schema.json` disagree about the **same field
name**. `read-response`'s `evidence[].source_type` already uses the authority's five values and accepts
`"chat"`; the shipped `evidence.schema.json` rejects it. A response produced by the read path cannot be
persisted as Evidence. That is `DRIFT-READ-RESPONSE-01` / `DRIFT-EVIDENCE-V1-01`, two halves of one defect.

## 5. Authority-drift findings

Each row is a real executing fixture; the "verdict" column is what the schema actually did. **A FAIL is
a finding** — none of these was softened to make a suite green.

### 5.1 Proved defects

**(1) `DRIFT-EVIDENCE-V1-01` — shipped `evidence.schema.json` cannot hold a §3.2 Evidence object**
Field: `original_text`, `speaker_id`, `visibility`, `source_type`. Clause: `DD-AUTHORITY-V2 §3.2`.
Fixture: `schemas/evidence-v1/finding-01-authority-3.2-instance-rejected.json` → rejected at
`additionalProperties`. Parent's B-1; re-verified independently. *(EV-SCHEMA-953)*

**(2) `DRIFT-MEMORY-UNIT-01` — MemoryUnit rejects the default state §3.1 describes**
Field: `validation_state` × `status`. Clause: `DD-AUTHORITY-V2 §3.1` ("validation_state … 默认
unverified", "status … 默认 ACTIVE"). Fixture: `memory-unit/finding-01-authority-default-state.json`.
`allOf[1]` requires `status=ACTIVE ⇒ validation_state ∈ {confirmed,disputed}`, so the pair the authority
calls the default is invalid, and a freshly created MemoryUnit has **no legal state**. Either §3.1 or
the D-AP-05 interlock must move; the schema has silently chosen. *(EV-SCHEMA-955)*

**(3) `DRIFT-MEMORY-OBJECT-01` — retired v1 `type` value still accepted by a second "memory" schema**
Field: `type`. Clause: `DD-AUTHORITY-V2 §2.1` + `§4.1` (`episodic → semantic_type EVENT`).
Fixture: `memory-object/finding-01-accepts-retired-v1-type.json` → `type: "episodic"` **validates**,
while `memory-unit` rejects the same semantic. Two schemas in one package both claim to describe a
memory and disagree on the vocabulary. *(EV-SCHEMA-956)*

**(4) `DRIFT-CANDIDATE-01` — §2.7 `candidate.status` has no home in the Candidate object**
Field: missing `status` / competing `review_state`. Clause: `DD-AUTHORITY-V2 §2.7`.
Fixture: `candidate/finding-01-authority-candidate-status-absent.json` → `status: "PENDING"` rejected at
`additionalProperties`. The schema ships a five-value `review_state`
(`queued/in_review/accepted/rejected/merged`) sharing no value with §2.7, while
`config/memory.schema.json` pins the correct six values as `candidate.status_values`. Config and object
model disagree about the same concept. *(EV-SCHEMA-957)*

**(5) `DEFECT-EVENT-01` — a documented MUST is not implemented**
Field: `participants`. Clause: `event.schema.json` own `$comment` ("participants 为空 … MUST 被拒")
+ `§2.4`. Fixture: `event/finding-01-empty-participants-accepted.json` → `participants: []`
**validates** (`uniqueItems` present, `minItems` absent). This is the class of gap a comment-only rule
always leaves. *(EV-SCHEMA-959)*

**(6) `DRIFT-READ-RESPONSE-01` — one field name, two vocabularies** (see §4)
Fixture: `read-response/finding-01-evidence-source-type-v2-accepted.json`. Recorded as PASS because
`read-response` is the *correct* side; the defect is charged to `evidence.schema.json`. *(EV-SCHEMA-958)*

**(7) `DRIFT-TEST-CONFIG-01` — `config/test.schema.json` contradicts the shared evidence vocabulary**
Field: `reporting.status_values`. Clause: `02-shared/EVIDENCE-FORMAT.md` "Status vocabulary" (not
DD-AUTHORITY-V2 — stated explicitly so it is not mistaken for an authority clause).
Fixture: `config/test/finding-01-status-values-drift.json` → the EVIDENCE-FORMAT set is rejected at
`const`. The pinned list `["PASS","FAIL","BLOCKED","NOT RUN","MOCKED"]` omits `PARTIAL_PASS`,
`NOT_VERIFIED`, `NOT_APPLICABLE`, spells `NOT_RUN` as two words, and invents `MOCKED`. *(EV-SCHEMA-960)*

### 5.2 Known deviation, not a defect

**(8) `NOTE-EVIDENCE-V2-01` — v2 requires `tenant_id`; the §3.2 table does not list it**
Field: `tenant_id`. Fixture: `schemas/evidence-v2/finding-01-strict-3.2-fields-rejected.json` → an
object built *only* from the §3.2 table is rejected at `required`. This is the parent's §3.2 NOTE, now
pinned by a test. Keeping the field is defensible (isolation is load-bearing for `04 §7.5`) and v2
documents its purpose. **Recommendation: keep, and record the divergence in the reconciliation note**
rather than as a schema bug. *(EV-SCHEMA-954)*

### 5.3 A finding I withdrew

`DRIFT-ANCHOR-CONFIG-01` was drafted and is **withdrawn as a false positive**. I had read
`config/anchor.schema.json`'s `intent_routing.default_read_mode` as "8 of the 9 §2.11 intents,
`ORIGINAL_QUOTE_QUERY` missing under `additionalProperties:false`". The contract says otherwise in
words (line 109): *"仅登记非证据类意图；ORIGINAL_QUOTE_QUERY 恒为证据模式，不在此配置"* — only
non-evidence intents are configured here, because `ORIGINAL_QUOTE_QUERY` is always evidence mode; the
full nine-intent routing lives in `anchor-set.schema.json`. I re-read the file and confirmed the text
myself; a concurrent audit (`TECH-SCHEMA-04`, EV-SCHEMA-916…918) independently reached the same
conclusion. The fixture was **kept as a negative test** of the enforced exclusion (renamed
`config/anchor/invalid-03-original-quote-query-read-mode.json`) and does not appear in the issue list.
Consequence: the intent-set coverage argument must rest on `anchor-set`, where I do test all nine
routes and the three interlock clauses.

## 6. Fixtures that were broken during development (and fixed)

Recorded because "a fixture that fails for an unrelated reason is a broken fixture": three were caught
before any evidence was written.

1. Two config fixtures asserted the wrong thing on first build — an `additionalProperties` error whose
   `absolute_path` is the *object holding* the extra key, not `""`; and an "empty-string enum"
   boundary built on a misread of the pretty-printed enum (`null` is an enum member, the empty string
   is not). Both were wrong about the schema, not findings about it. Fixed; the false "empty string"
   claim was removed rather than retained as a defect.
2. A hash helper produced 192-character "sha256" strings; 7 fixtures failed `pattern`. Fixed.
3. One finding fixture was written as `kind: invalid` and therefore landed under an `invalid-` filename
   while the evidence emitter looked for `finding-`; the record came out `BLOCKED` and was corrected
   before the real append.

## 7. What is NOT covered

- **`format`** (`uuid`, `date-time`) — annotation-only in Draft-07, deliberately not asserted (§3).
- **Referential integrity across objects** — no fixture checks that `MemoryUnit.evidence_refs` points at
  an extant Evidence, or that `event_refs` points at an Event. Fixtures are single-schema instances.
- **v1 → v2 migration** — the seven mapping rows of §4.1 and the field mapping of §4.2 are not tested by
  any instance. `memory-object` and `search-document` are tested for *shape*, not for whether they
  should still exist.
- **`§4.3 visibility ↔ scope` matrix** — only `visibility=public` rejection and one
  `scope=group_shared ⇒ visibility=group_shared` coupling are tested, not the full 3×5 mapping.
- **Prose contracts** — `contracts/error-codes.md` (the 16 codes are mirrored in `error.schema.json`
  and *that* enum is tested; the `.md` is not compared), `contracts/api/api-contracts.md`,
  `contracts/state-machines/state-machines.md`. Not schemas, so out of scope here.
- **Live write path** — every instance is hand-built. Nothing shows that a producer emits these shapes.
- **`config/retrieval.schema.json` `reranking.weights`** — five unbounded numbers; a weight of 10⁹ is
  legal and untested. Noted, not asserted.

## 8. Decisions needed from the parent

1. **`DRIFT-MEMORY-UNIT-01`** — which side moves: the §3.1 default pair (`unverified` + `ACTIVE`) or the
   D-AP-05 interlock? Until one moves, a freshly created MemoryUnit cannot validate. This is the highest
   severity item in this report because it blocks every write path, not just a projection.
2. **`DRIFT-CANDIDATE-01`** — add `status` (§2.7, six values) to `candidate.schema.json` and retire
   `review_state`, or record §2.7 as unused and align `config/memory.schema.json` to `review_state`?
3. **`DEFECT-EVENT-01`** — add `minItems: 1` to `participants`? It is a one-line change to a schema the
   package itself says must reject the empty case.
4. **`DRIFT-EVIDENCE-V1-01` / `DRIFT-READ-RESPONSE-01`** — confirm the merge of the v2 evidence contract
   into the canonical package, and confirm no consumer persists `read-response.evidence[]` directly as
   Evidence while the two vocabularies coexist.
5. **`memory-object` / `search-document`** — are the v1 models authoritative for any module? If not, mark
   them deprecated in-file; otherwise a consumer that validates a MemoryUnit against `memory-object`
   will accept `type: "episodic"`, which §4.1 retires.
6. **`NOTE-EVIDENCE-V2-01`** — accept the `tenant_id` deviation as a recorded divergence (my
   recommendation), or amend the authority table.
7. **`format` in CI** — do you want a `FormatChecker` in the validator? It would enforce `uuid`/
   `date-time`, but some currently-accepted hand-written documents may then fail; that is a policy
   decision, not a fixture decision.
8. **Ledger concurrency** — `06-evidence/schema.jsonl` is being written by `TECH-SCHEMA-03` and
   `TECH-SCHEMA-04` while this task ran. Ids were allocated from the live maximum (911 → next) and the
   file verified after the append: 53 lines, 53 distinct `evidence_id`, 0 duplicates, 0 unparseable.
   If the ledger is later compacted or renumbered, `EV-SCHEMA-919…960` is the range this report cites.
