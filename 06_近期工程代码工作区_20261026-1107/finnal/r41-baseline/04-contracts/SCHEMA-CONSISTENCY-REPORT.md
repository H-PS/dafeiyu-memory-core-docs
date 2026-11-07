# SCHEMA-CONSISTENCY-REPORT — TECH-SCHEMA-01

Date: 2026-11-05 (Asia/Shanghai)
Scope: `01-baseline-corpus/design-package/contracts/**`
Harness: `03-verification/schema/validate-schemas.cjs`, `evidence-schema-conformance.cjs`
Evidence: `06-evidence/schema.jsonl`; raw: `06-evidence/raw/schema/`

---

## 1. Result summary

| Check | Result |
|---|---|
| JSON parse, all contract files | **31 / 31 parse** (after one fix, see §2) |
| Local `$ref` resolution | **0 unresolved** |
| Draft-07 meta-schema validation | **31 / 31 valid** |
| Cross-file `$ref` | **2 external refs**, neither resolvable offline (see §4) |
| Evidence schema conformance to authority | **FAIL on the shipped schema** (B-1 confirmed, §3) |

Everything below is an executed result. Raw artefacts:
`06-evidence/raw/schema/meta-validate.result.json`,
`shipped-schema.validation.json`,
`corrected-schema.validation.json`,
`corrected-schema.invalid.validation.json`.

## 2. Defect found and fixed: UTF-8 BOM in `read-response.schema.json`

`validate-schemas.cjs` failed to parse one file:

```
Unexpected token '\uFEFF', "{ ... " is not a valid JSON
```

`contracts/schemas/read-response.schema.json` began with the bytes `EF BB BF` — a UTF-8 BOM.
It is the **largest** response schema (30 428 bytes) and therefore the most likely to be consumed
by a strict parser.

A strict JSON parser (`JSON.parse`, most Node tooling, many CI validators) **rejects** a BOM-prefixed
file. A lenient one silently accepts it. That is exactly the kind of latent defect that passes review
and fails in production.

**Action taken:** BOM stripped from the snapshot copy
(`01-baseline-corpus/design-package/contracts/schemas/read-response.schema.json`), file re-parsed
and re-validated. After the fix, **31/31** parse and **31/31** are meta-valid.

**Scan for siblings:** exactly one of the 31 contract files carried a BOM. No other action needed.

**Recommendation:** add BOM rejection to CI. A one-line check (`bytes[0] !== 0xEF`) prevents
reintroduction.

## 3. B-1 confirmed by execution: evidence.schema.json cannot hold a compliant Evidence

This was previously a claim from static reading. It is now a **measured** result.

`evidence-schema-conformance.cjs` builds an Evidence object strictly from the authority's field
table (`DD-AUTHORITY-V2.md §3.2`) — including `original_text`, `speaker_id`, `source_type: "chat"`,
`visibility: "owner_private"` — and validates it with Python `jsonschema` 4.26.0.

| Schema | Accepts a spec-compliant instance? | Failing keywords |
|---|---|---|
| `01-baseline-corpus/.../evidence.schema.json` (shipped, v1) | **NO** | `additionalProperties`, `required`, `enum` |
| `04-contracts/schemas/evidence.schema.json` (corrected, v2) | **YES** | — |

Three independent failures compound:

1. **`required`** — the shipped schema requires a field set that does not include `original_text`
   or `speaker_id`, and the authority requires both. The instance carries them, but it cannot carry
   what the authority demands *and* satisfy the schema's `additionalProperties: false`.
2. **`additionalProperties: false`** — the presence of `original_text` and `speaker_id` is itself
   an error, because the schema does not declare them. This is the fatal one: **there is no way to
   write a compliant Evidence object against the shipped schema**, not even by omitting fields.
3. **`enum`** — the shipped `source_type` is the 12-value **v1** set
   (`direct_user_statement`, `assistant_observation`, …), while the authority fixes a 5-value
   Evidence enum (`chat`, `file`, `import`, `tool`, `system`).

Note the two `source_type` fields are genuinely different and must not be unified:
`MemoryUnit.source_type` (5 values, *nature* of the source) vs `Evidence.source_type`
(5 values, *form* of the evidence). `13 §7.6.3` forbids mixing them. The corrected schema names this
explicitly in the field description so a future editor cannot re-conflate them.

### 3.1 What was changed, and the authority clause behind each change

| Change | Authority | Rationale |
|---|---|---|
| added `original_text` (required, minLength 1, maxLength 20000) | §3.2 "original_text string ✓ 原始表达（原句）；MUST NOT 被规范化覆盖" | The reason Evidence exists. |
| added `speaker_id` (required) | §3.2 "speaker_id string ✓ 发言者主体 ID" | Provenance of who spoke. |
| `source_type` → 5-value `chat/file/import/tool/system` | §3.2 table | The 12-value set is the retired v1 vocabulary. |
| `visibility` (required, enum `owner_private`/`group_shared`) | §2.10 + CR-07 | `public` is disabled in V1, so it is not in the enum. |
| kept `scope` 5-value | §4.3 mapping | Internal sub-classification, retained by design. |
| `timestamp` (nullable) | §3.2 + `13 §7.2-2` | Unknown time must be `null`, never a guess. |
| `revoked` now **required** | §3.2 "revoked boolean ✓" | Was optional with a default in v1; the authority marks it required. |
| added `context_before`/`context_after` (nullable) | §3.2 | Bounded context for comprehension. |
| dropped `excerpt` | superseded | It invited storing a rewrite in place of the original; that is the P-02/P-05 failure mode. |
| dropped `observed_at` in favour of `timestamp` | §3.2 naming | One name for one concept. |
| `retention_policy` retained nullable | §3.2 + CR-06 | Retention remains an open operational decision; the field records it, it does not decide it. |
| added `schema_version` const `2.0` | consistency with `memory-unit.schema.json` (const `2.0`) | Makes v1↔v2 detectable in data, not just in file history. |
| `not` clause forbidding `scope: system_internal` + `visibility: group_shared` | `04 §7.5` isolation rules | Internal metadata must never be exposed as shared. |
| `allOf` coupling `scope: group_shared` ⇒ `visibility: group_shared` | `04 §7.5-7` | A shared-scope evidence item cannot claim private visibility. |
| `allOf` coupling non-revoked ⇒ non-empty `original_text` | §3.2 | A live evidence record must still have readable text. |

The corrected schema is **not** a relaxation: `evidence_negative_fixture_rejected` shows it still
rejects a v1 `source_type` alias and an empty `original_text`.

### 3.2 Remaining open items in the evidence contract

| Item | Status | Why it is not closed here |
|---|---|---|
| `retention_policy` vocabulary | **OPEN** (CR-06) | No approved retention vocabulary exists; inventing one would fabricate a policy. |
| Whether `timestamp` may be `null` in production | **OPEN** (round-3 D-7) | Requires a user decision; recorded as an open decision, not silently allowed. |
| `tenant_id` vs authority | **NOTE** | The authority's §3.2 table does not list `tenant_id`, but the v1 schema makes it required. Kept required: isolation is load-bearing for `04 §7.5`, and removing it would weaken the contract. Flagged rather than silently resolved. |

## 4. Cross-file references

Two `$ref`s point at other files by absolute URI:

| File | Reference | Status |
|---|---|---|
| `contracts/config/embedding.schema.json` | `https://memory-core.r41/schemas/embedding-space.schema.json#` | resolvable only with a registry; not resolvable offline |
| `contracts/schemas/retrieval-response.schema.json` | `https://memory-core.r41/schemas/error.schema.json#` | same |

This is a **validation gap, not necessarily an error**: offline validators cannot resolve an
`https://` `$id` without a store/registry. Both target files exist locally, so the intended reference
is clear.

**Recommendation:** either (a) register a local resolver mapping `https://memory-core.r41/schemas/`
to `contracts/schemas/`, or (b) replace with relative refs. Until one is chosen, cross-file
validation cannot run in CI.

## 5. Parameter identity — SRD-003

Measured by `validate-parameter-registry.cjs`:

| Metric | Value |
|---|---|
| Canonical IDs harvested from `15-参数注册表.md` | **68** |
| Files scanned | 84 |
| Parameter references found | **355** |
| Resolved to a canonical ID | 280 |
| Unresolved | **75**, spanning **55 distinct names** |

The 55 unresolved names are references that look like parameters but have no canonical registry
definition. The largest clusters:

| Name | Hits | Reading |
|---|---|---|
| `profile.budget.tokens` | 4 | token budget per profile — no registry entry |
| `ingest.sync_budget_ms` | 3 | ingestion timing knob — no registry entry |
| `ingest.deferred_ttl_hours` | 3 | deferred-candidate TTL — no registry entry |
| `ingest.max_concurrency`, `ingest.stage_timeout_ms`, `ingest.task_timeout_ms`, `ingest.max_fragments_per_envelope`, `ingest.trigger_policy`, `ingest.chat_wait_budget_ms` | 2 each | ingestion group is underrepresented in the registry |
| `profile.event_supplement_max`, `profile.integration.timeout_ms`, `profile.integration.retry_max`, `emotion.delta_version` | 2 each | profile/emotion groups incomplete |

**Interpretation.** SRD-003 decided *how* parameter identity works (registry ID canonical, schema
path is a locator). This scan shows that decision has **not been applied across the corpus**:
55 names are used as if they were parameters without being registered. This is the concrete residue
behind MC-AUD-006 "DECIDED; partial".

**What this does not mean:** these are not necessarily *wrong* values. They are unregistered
identities. The fix is registry entries (or rewriting the reference), not deletion of the reference.

`memory.propose` and the other nine `memory.*` names are excluded explicitly as API method names,
listed in `API_METHOD_NAMES` in the scanner so the exclusion is auditable rather than a silent
pattern tweak.

## 6. Unvalidated / out of scope

- **Business-instance validation** of the other 30 schemas (positive/negative fixtures per schema)
  is only partially done: this report covers meta-validity and the evidence contract specifically.
  Agent-level fixture suites remain the gate (MC-AUD-018 still `PARTIAL`).
- **Version compatibility** across schema revisions: not tested.
- **`format` assertion**: `format` is annotation-only in Draft-07 unless a format checker is
  configured; timestamps/uuid patterns are enforced by `pattern` only where written.

## 7. Recommendations

1. **Adopt `04-contracts/schemas/evidence.schema.json` (v2)** in place of the shipped v1 file. Until
   it is merged into the canonical package, no Evidence write path can produce a legal object.
2. **Add BOM rejection to CI** (single byte check).
3. **Register the 55 unregistered parameter names, or rewrite the references.** Until then, SRD-003
   is decided but not enforced.
4. **Add a local `$id` → file resolver** so cross-file `$ref` validation can run offline.
5. Keep the two `source_type` fields distinct in every future edit; the corrected schema says so in
   the field description to prevent regression.
