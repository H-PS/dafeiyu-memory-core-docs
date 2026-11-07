# PARAMETER-IDENTITY-REPORT — TECH-SCHEMA-01

Date: 2026-11-05 (Asia/Shanghai)
Harness: `03-verification/schema/validate-parameter-registry.cjs`
Raw: `06-evidence/raw/schema/parameter-registry.result.json`
Evidence: `06-evidence/schema.jsonl` (`schema.parameter_registry_canonical_ids`, `schema.parameter_reference_resolution`)

---

## 1. The rule being tested (SRD-003)

> Parameter identity is the canonical ID in the parameter registry; the physical Schema path is
> a locator only. Schema path and registry ID must not be treated as co-equal identifiers.

This report tests whether the corpus *obeys* that rule, not whether the rule is correct.

## 2. Measured result

| Metric | Value |
|---|---|
| Canonical registry IDs extracted from `15-参数注册表.md` | **68** |
| Files scanned (all `.md` + `.json` in the design package, excluding the registry itself) | 84 |
| Parameter-style references found | **355** |
| Resolved to a canonical registry ID | 280 (78.9 %) |
| **Unresolved** | **75 (21.1 %)**, across **55 distinct names** |

So roughly **one in five parameter references in the corpus does not correspond to a registered
parameter**. SRD-003 is decided; it is not enforced.

## 3. The 55 unregistered names

Full list with hit counts and file locations is in
`06-evidence/raw/schema/parameter-registry.result.json` → `unresolved_by_ref` / `unresolved_detail`.
The clustered picture:

### 3.1 `ingest.*` group — most under-registered (about 18 names)

`sync_budget_ms`, `deferred_ttl_hours`, `max_concurrency`, `stage_timeout_ms`, `task_timeout_ms`,
`max_fragments_per_envelope`, `extract`, `trigger_policy`, `chat_wait_budget_ms`, …

The registry's §7.4 declares 6 `ingest.*` parameters. The documents reference far more than 6.
Either the registry is incomplete or the documents invented knobs. This is the clearest single
inconsistency in the corpus.

### 3.2 `profile.*` group

`budget.tokens` (4 hits — the most-referenced unregistered name), `event_supplement_max`,
`integration.timeout_ms`, `integration.retry_max`.

The registry's §7.12 declares 3 `profile.*` parameters, none of which are these.

### 3.3 `emotion.*`, `vector.*`, `indexing.*` — scattered singles

e.g. `emotion.delta_version`, which appears to be a schema/contract version marker rather than a
parameter — a case of the SRD-003 confusion running in the other direction (a non-parameter written
to look like one).

## 4. Deliberately excluded, and why

`memory.search`, `memory.propose`, `memory.getProfile`, `memory.getByAnchor`, `memory.getEvent`,
`memory.getEvidence`, `memory.getRelated`, `memory.getEmotionContext`, `memory.getMemoryById`
are **API method names** from `DD-AUTHORITY-V2 §7`, not parameters. They share the `memory.` prefix
with the registry's `memory.*` parameter group, so a naive scan reports them as unregistered
parameters.

These are excluded via an explicit `API_METHOD_NAMES` set in the scanner — listed in source so the
exclusion is reviewable — and call-syntax occurrences (`name(`) are excluded structurally as well.
Excluded count: 60 occurrences.

If this exclusion were removed the unresolved figure would be ~130 rather than 75, almost entirely
noise. Flagging it here because an inflated defect count is its own form of dishonesty.

## 5. What this means

1. **The registry is not the actual parameter universe.** Points in code or documents that read
  `ingest.max_concurrency` will find no canonical definition. Either it gets registered or the
  reference must be rewritten to an existing ID.
2. **It is not a correctness bug** — these names may carry perfectly sensible defaults described
   in prose. The defect is *identity*, which is what makes configuration auditable at all.
3. **MC-AUD-006 status must stay `DECIDED; NOT ENFORCED`**, not "resolved". The decision exists;
   the corpus does not implement it.

## 6. Recommended remediation

Ordered by cost, cheapest first:

| # | Action | Scope | Effect |
|---|---|---|---|
| 1 | Add the ~18 `ingest.*` names to registry §7.4 with the 15 required attributes | one doc + contract | removes the largest cluster |
| 2 | Add `profile.budget.tokens` and the three `profile.integration.*` | one doc | removes the second cluster |
| 3 | For names that are really schema locators (`emotion.delta_version`), rewrite the reference as a path, not an ID | several docs | fixes the reverse SRD-003 confusion |
| 4 | Make `validate-parameter-registry.cjs` a CI gate | tooling | prevents drift from recurring |

Recommended CI threshold: **0 unresolved non-API references**. Until #1–#3 are done that threshold
cannot pass, which is the point of setting it.

## 7. Honest limits of this scan

- Pattern-based extraction. An ID written without a backtick or without a group prefix is missed,
  so the **68** canonical count is a floor, not a ceiling.
- A name that merely looks like `group.member` (a JSON path inside a schema, a dotted property)
  is counted as a parameter reference. Some of the 55 may therefore be locators rather than
  parameters — which is itself the SRD-003 confusion being measured.
- Only the design package was scanned, not the legacy M01–M05 corpus. Cross-checking that legacy set
  is Agent H's inheritance matrix, not duplicated here.
