# CROSS-DOCUMENT-CONSISTENCY-MATRIX — parent summary

Date: 2026-11-05 (Asia/Shanghai)

The full matrix (10 findings with anchors) is in
`05-audit/CROSS-DOCUMENT-CONSISTENCY-MATRIX.md` (28 KB, produced by the historical-consistency
workstream and reviewed by the parent). This file is the parent's distilled reading: what each
finding means for implementation, and whether it is now closed.

---

## 1. The ten findings and their disposition

| ID | Finding | Implementer impact | Status |
|---|---|---|---|
| **SC-01** | Parameter identity: registry declares `recall.*` the only legal ID, yet **53 `retrieval.*` occurrences (36 distinct, 20 unmapped)** remain, and `02` — the algorithm document — uses **zero** canonical IDs | A config reader cannot tell which name is authoritative | **OPEN** — matches the parent's independent count (355 refs / 75 unresolved / 55 names, different scope) |
| **SC-02** | RRF channel drift: `structured` excluded by the decided rule (`11:617`, `06:558`, OI-022) while `02:344/346` still defines `w_structured=1.0` and `00:638` still draws `STR → RRF`. `02:4` contradicts `02`'s own §7.4 | Two implementers would build different fusion pipelines from the same corpus | **OPEN** — but the *intent* is settled (SRD-002); the text has not caught up |
| **SC-03** | Object-model split: `memory-object.schema.json` (v1) and `memory-unit.schema.json` (V2) both ship, **unmarked** | A schema-driven agent picks one at random | **OPEN** — escalated as E-4 |
| **SC-04** | `content_kind` is **abolished** at `00:76` yet **mandatory with 8 enums** at `01 §7.3`/`§7.6.2`, required by `02:630`, and shipped in `memory-object.schema.json` | Implementers must choose between two contradictory contracts | **OPEN** — escalated as E-3 |
| **SC-05** | `evidence.schema.json` has 12 `source_type` values and no `original_text` | An Evidence object cannot be legally written | **CLOSED by proposal** — v2 in `04-contracts/schemas/`, proven by execution; needs merging |
| **SC-06** | Status-axis conflation: candidate status vs memory status vs validation_state vs tier | State machines become untranscribable | **PARTIAL** — schema now separates the axes; SM-1 still mixes lifecycle with validation (OI-R-06) |
| **SC-07** | v1/v2 enum mixing (seven `type` values, six `source_type` values vs the V2 six/five) | Old values validate where they should not | **OPEN** — partially addressed by the corrected Evidence schema |
| **SC-08** | Broken references: **295 severity-1 unresolved** across 5701 citations (893 to un-shipped sources, 198 missing files, 34 legacy basenames) | A reader cannot follow the chain to its source | **OPEN** — recorded in `GLOBAL-REFERENCE-ISSUES.md` |
| **SC-09** | Duplicate authoritative files claiming the same subject | Ambiguous authority | **OPEN** — part of the two-baseline problem (E-1) |
| **SC-10** | The prior `reviews/CROSS-DOCUMENT-CONSISTENCY-REPORT.md` claims 29 PASS/FIXED items yet cites `contracts/api/permission-api.md` twice — a file that **does not exist** — and scopes itself to one document set | A prior PASS verdict on consistency should **not be relied on** | **SUPERSEDED** — this round's matrix replaces it in scope |

## 2. The three that change what an implementer would build

### 2.1 `content_kind` (SC-04) — the sharpest contradiction

The corpus simultaneously says:

- `00-总体架构与设计原则.md:76` — `content_kind（八值） | 废弃 → semantic_type + semantic_tags + source_type`
- `00:7` — "`content_kind` 废弃，语义由 `semantic_type`(6 类) + `semantic_tags` 承载"
- `DD-AUTHORITY-V2.md:291` — "`content_kind(fact/opinion/…)` | **废弃** → source_type + semantic_tags"
- **but** `01-记忆本体与数据模型.md §7.3` defines it as an eight-value mandatory field,
  `01:161` says `type` and `content_kind` **MUST NOT** be merged, `01:206` builds normative rules on
  it, `02:630` requires it, and `memory-object.schema.json` **requires** it with the 8-value enum.

Verified directly by the parent: `required includes content_kind: True`.

This is not a wording difference. Either V2 abolished the field and the schema is stale, or the
schema is right and V2's abolition was never applied. **An implementer picking either is defensible,
which is precisely what makes the contract unusable.**

### 2.2 The object-model split (SC-03 / E-4)

Two entity schemas ship side by side with no deprecation marker. `memory-object` is the v1 composite
storage view; `memory-unit` is the V2 28-field object. `memory-unit.schema.json` even carries
`$comment` listing eight things that MUST be rejected — a level of rigour `memory-object` lacks.

Narrowing which one is canonical is a one-line decision with large downstream consequences, so it is
escalated rather than decided.

### 2.3 Reference rot (SC-08)

**295 severity-1 unresolved references out of 5701.** Causes: 893 citations to sources never shipped
(the task books, the ledger, `sf`, `m.md`); 198 references to files that do not exist (worst:
`A-REVIEW-PROGRESS.md` cited 16×); 34 legacy basenames.

This matters because the design's authority model is *citation-based*. A reference that cannot be
followed is an assertion without a source.

## 3. What the parent verified independently

| Claim | How verified | Result |
|---|---|---|
| `content_kind` mandatory in the schema | read `memory-object.schema.json` | **confirmed** (`required` contains it; enum is the 8 values) |
| Two object schemas ship unmarked | listed `contracts/schemas/` | **confirmed** |
| `read-response.schema.json` was unparseable | `JSON.parse` failed on the BOM | **confirmed**, then fixed |
| The prior consistency report cites a non-existent file | grep for `permission-api.md` | **confirmed** (2 occurrences) |
| Parameter identity drift | independent scan: 355 refs / 75 unresolved / 55 names | **confirmed**, consistent with SC-01 at a different scope |

## 4. Recommendation

Three of these are cheap and should be done before any implementation:

1. **Merge the corrected Evidence schema** (SC-05) — already proven; cost is review time.
2. **Decide `content_kind`** (E-3) — either enforcement direction is fine; keeping both is not.
3. **Mark or remove one object model** (E-4) — a deprecation marker would suffice initially.

The remaining findings (SC-01, SC-02 text, SC-06, SC-07, SC-08, SC-09) are text-synchronisation work:
real, tractable, and requiring no architectural decision.

---

## 5. STRUCTURAL DIVERGENCE — the authority documents a layout that was never built

Found by the parent while verifying SC-08. It is not a broken link; it is a **whole documented
substructure that does not exist in the shipped package.**

### 5.1 `contracts/api/` — documented 8 files, ships 1

`DD-AUTHORITY.md` (lines ~181-184) documents:

```
├── api/      m01-api.md m02-api.md m03-api.md m04-api.md m05-api.md
│             context-adapter-api.md permission-api.md database-adapter-api.md
```

Actual: `contracts/api/` contains **`api-contracts.md` only** (one file).

So `permission-api.md` — cited **5 times** across `DD-AUTHORITY.md:183`,
`DESIGN-DECISION-REGISTER.md:19` (decision AD-008 makes it a dependency),
`CROSS-DOCUMENT-CONSISTENCY-REPORT.md:141/193/200` — **does not exist anywhere in the package**
(verified: 0 files with that name on the whole machine).

### 5.2 `contracts/state-machines/` — documented 8 files, ships 1

Documented: `memory-lifecycle.md  validation-state.md  availability-state.md  outbox-state.md
worker-state.md  migration-state.md  deletion-state.md  embedding-migration-state.md`.

Actual: `state-machines.md` (one consolidated file containing SM-1…SM-8).

This consolidation is arguably **better** practice than eight fragments — the parent is not calling
it a defect. But `DD-AUTHORITY.md` still describes the eight-file layout, so the authority document
and the shipped tree disagree.

### 5.3 `contracts/error-codes/` is an EMPTY DIRECTORY

Actual state: `contracts/error-codes/` exists and contains **zero files**, while the real
`error-codes.md` lives at `contracts/error-codes.md` (a sibling, not inside the directory).

The directory has a trailing-slash entry in `DD-AUTHORITY.md` (`└── error-codes.md`) but an empty
`error-codes/` directory was nevertheless created — a scaffolding artefact that was never populated.
An implementer browsing the tree will find an empty directory and may conclude the error contract is
missing, when it is one level up.

### 5.4 `_sources/` is documented but absent from the snapshot

`DD-AUTHORITY.md` lists `_sources/  m.md、sf.md（备份副本，只读）`. In the canonical package these
are at `design/_sources/` (under a different parent), not under `contracts/`. Path divergence.

### 5.5 Assessment

**Severity: P2.** Nothing here blocks implementation — the consolidated files are usable. But it
breaks the authority model in two ways:

1. A decision (`AD-008`) declares a dependency on `04/permission-api`, a document that does not
   exist. So the decision cannot be traced to its stated basis.
2. Three prior-round reports cite `permission-api.md`. One of them
   (`CROSS-DOCUMENT-CONSISTENCY-REPORT.md`) also claimed **29 PASS/FIXED** consistency items while
   citing this missing file twice — which is why its verdict should not be relied on (SC-10).

**Recommended fix:** either (a) correct `DD-AUTHORITY.md` to describe the shipped layout, or
(b) create the missing per-module API files. (a) is cheaper and honest; (b) is what the document
intended. This is a documentation-consistency choice, not an architecture question, so it does not
need user escalation — but it does need an owner.

---

## 6. Fixture-level findings, and one FALSE POSITIVE caught

The schema workstream produced business fixtures for the contracts that had none, including eight
`finding-*` fixtures that assert a specific authority drift. The parent audited each against the
target contract's own text (`03-verification/schema/audit-fixture-findings.cjs`).

**Result: 8 findings inspected, 1 refuted, 7 upheld.**

### 6.1 REFUTED — `DRIFT-ANCHOR-CONFIG-01` (`config/anchor/finding-01-original-quote-query-read-mode`)

The fixture claimed drift because `anchor.schema.json`'s `intent_routing.default_read_mode` lists only
eight of the nine authority intents — `ORIGINAL_QUOTE_QUERY` is absent under `additionalProperties:false`.

**That is by design.** The schema itself says so, at line 109:

> `"description": "意图到读取模式的默认映射（§2.12 四值）。仅登记非证据类意图；ORIGINAL_QUOTE_QUERY 恒为证据模式，不在此配置。"`

Translation: *only non-evidence intents are registered here; `ORIGINAL_QUOTE_QUERY` is always evidence
mode and is deliberately not configured.*

So the omission is correct behaviour, not drift. The fixture is recorded as refuted and **must not**
be carried into the issue register as a defect.

This is worth stating plainly because it shows the value of the parent check: a fixture that a naive
reviewer would accept as a finding would have entered the register as a false defect, and then
someone would have "fixed" a schema that was already right.

### 6.2 UPHELD (pending manual review) — the strongest of the seven

`DRIFT-EVIDENCE-V2-01` (`schemas/evidence-v2/finding-01-strict-3.2-fields-rejected`):

> v2 requires `tenant_id`, but `DD-AUTHORITY-V2 §3.2`'s Evidence field table **does not list it**.
> An Evidence object built strictly from the §3.2 table — adding nothing the authority does not
> name — is rejected by the corrected schema.

This is a **new, sharper version** of a note the parent had already recorded. The parent wrote it up
as a "NOTE" (that `tenant_id` is required by v1 but absent from the authority table, and that keeping
it was defensible because isolation is load-bearing). The fixture proves the consequence: **v2 is not
a literal transcription of the authority**, and a strictly-conformant instance fails.

That is a real statement about the contract, and it is now recorded as such:

- **Deliberate**: yes — `tenant_id` is required because `04 §7.5` isolation depends on it.
- **Authority-transcription**: no — the authority must be amended to list `tenant_id` for Evidence,
  or the schema must document why it exceeds the authority.

The fix is a one-line amendment to `DD-AUTHORITY-V2 §3.2`, and it is now traceable to a reproducing
fixture instead of a parent's aside.

### 6.3 The other six upheld findings

`candidate/finding-01-authority-candidate-status-absent`; `event/finding-01-empty-participants-accepted`;
`memory-object/finding-01-accepts-retired-v1-type`; `memory-unit/finding-01-authority-default-state`;
`evidence-v1/finding-01-authority-3.2-instance-rejected`; plus the anchor one refuted above.
Each carries its authority citation and a reproducing instance. They await a human read before
entering the register as confirmed defects — the parent's heuristic cannot prove a drift, only
disprove an intentional exclusion.
