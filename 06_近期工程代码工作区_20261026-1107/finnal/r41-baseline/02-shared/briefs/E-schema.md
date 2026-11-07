# BRIEF — Agent E · API / Schema / Parameter Registry conformance

Read first:
1. `02-shared\00-PARENT-MANDATE.md`
2. `02-shared\EVIDENCE-FORMAT.md`
3. `01-baseline-corpus\design-package\DD-AUTHORITY-V2.md` (§2 enumerations, §3 object field tables)
4. `01-baseline-corpus\design-package\15-参数注册表.md`
5. `01-baseline-corpus\design-package\contracts\` (schemas, config, api, state-machines, error-codes)

Own: `04-contracts\` and `03-verification\schema\`. Evidence: `06-evidence\schema.jsonl`; raw: `06-evidence\raw\schema\`.

## Load-bearing defect (B-1) — you own the fix

`contracts/schemas/evidence.schema.json` contradicts `DD-AUTHORITY-V2.md §3.2` on at least four axes:
`original_text` missing (authority requires it, and it MUST NOT be normalised away), `speaker_id`
missing (authority requires it), `source_type` is a **12-value v1 enum** where the authority fixes a
**5-value** enum (`chat/file/import/tool/system` for Evidence — note this differs from MemoryUnit's
5-value enum, they are different fields with different value sets), and `additionalProperties:false`
means a spec-compliant Evidence object is currently **invalid**.

Produce a corrected `evidence.schema.json` (v2) under `04-contracts\schemas\` and a written
reconciliation note stating, per field, either "aligned to authority" or "authority needs a change"
with reasoning. Do not silently pick a value: show the authority clause you are aligning to.
Where the authority itself is underspecified, record it as an open item rather than inventing a rule.

Also check every other schema for the same class of drift: v1 enum leakage, required-field mismatch,
`additionalProperties:false` blocking authority-required fields, and field-name divergence
(e.g. `observed_at` vs `timestamp`, `visibility_scope` vs `visibility`).

## Harness work (must actually run)

- `validate-schemas.cjs`: draft-07 meta-schema validation + local `$ref` resolution for every schema.
  Python 3.14.7 with `jsonschema` 4.26.0 is available (set
  `$env:PYTHONPATH='C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design\evidence\second-round\pydeps'`),
  or use a JS approach — but the run must actually happen and produce real output.
- Positive, negative and **boundary** fixtures for each schema. A negative fixture must fail
  **for the intended reason** — assert the failing keyword, not merely "it failed". A fixture that
  fails for an unrelated reason is a bad fixture, not a passing test.
- `validate-api-contracts.cjs`: check each documented method against the schema-backed request/response
  envelope, error codes present in `error-codes.md`, idempotency declaration, version declaration.
- `validate-parameter-registry.cjs`: each parameter must have a canonical ID; scan all 00–15 documents
  and contracts for parameter references and report every reference that does not resolve to a
  canonical registry ID, or that uses an alias/schema-path as if it were an ID (SRD-003 says the
  registry ID is canonical and the Schema path is a locator only).
- `check-cross-document-consistency.cjs`: detect RRF channel drift (only `fts`+`vector` may be ranked;
  `structured` is filter/router only, SRD-002), status-axis conflation, and v1/v2 enum mixing across
  documents.

## Deliverables

- `04-contracts\API-CONSISTENCY-REPORT.md`
- `04-contracts\SCHEMA-CONSISTENCY-REPORT.md`
- `04-contracts\PARAMETER-IDENTITY-REPORT.md`
- corrected schema(s) + fixtures, with a machine-readable result JSON
- `03-verification\schema\TASK-RESULT.md`

Report counts honestly: how many schemas validated, how many fixtures, how many passed/failed,
how many parameter references scanned, how many unresolved. Never round up.

Parent will re-run your validators. A validator that only prints "OK" without per-item detail is
not acceptable.
