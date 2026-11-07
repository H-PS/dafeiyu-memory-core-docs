# IMPLEMENTATION-READINESS-MATRIX — Memory Core R4.1, round 4.1

Date: 2026-11-05 (Asia/Shanghai)

**Headline: no module is `IMPLEMENTATION_AUTHORIZED`. M01 is the closest to readiness; M02 is blocked by its two retrieval channels; M03–M05 are contract-only.**

The matrix deliberately scores **eight independent axes per module**. A single "readiness %" would
hide exactly the distinction this project keeps getting wrong.

Legend: `READY` · `PARTIAL` · `NOT_READY` · `BLOCKED` · `N/A`

---

## 1. The matrix

| Module | Design | Contract | Schema | Dependency | Test | Evidence | Security | Impl. Ready | **Authorized** |
|---|---|---|---|---|---|---|---|---|---|
| **M01** Storage (SQLite authority) | READY | PARTIAL | PARTIAL | READY | READY | READY | PARTIAL | **PARTIAL** | **NO** |
| **M02** Retrieval (FTS + vector + RRF) | PARTIAL | PARTIAL | PARTIAL | PARTIAL | PARTIAL | PARTIAL | PARTIAL | **NOT_READY** | **NO** |
| **M03** Extraction / candidates | PARTIAL | PARTIAL | PARTIAL | NOT_READY | NOT_READY | NOT_READY | N/A | **NOT_READY** | **NO** |
| **M04** Integration / profile | PARTIAL | PARTIAL | PARTIAL | NOT_READY | NOT_READY | NOT_READY | NOT_READY | **NOT_READY** | **NO** |
| **M05** Permission | PARTIAL | PARTIAL | PARTIAL | BLOCKED | PARTIAL | PARTIAL | PARTIAL | **BLOCKED** | **NO** |

## 2. Per-module justification

### M01 — Storage

| Axis | State | Why |
|---|---|---|
| Design | READY | `01` + `DD-AUTHORITY-V2 §3` define the objects; SQLite is sole authority. |
| Contract | PARTIAL | `DatabaseAdapter` exists in `api-contracts.md §8`, but the two `backup()` traps require new clauses (OI-R-08). |
| Schema | PARTIAL | 31/31 meta-valid; **Evidence schema does not accept a compliant instance** (OI-R-04, fix delivered but unmerged). |
| Dependency | **READY** | `node:sqlite` is builtin — zero third-party dependency, no native build. Strongest axis in the whole project. |
| Test | READY | 55 executed checks, 0 failing. |
| Evidence | READY | Every check has a record + raw log; `integrity_check` used at restore and crash points. |
| Security | PARTIAL | Storage itself is not a permission boundary, but ciphertext/at-rest policy is undefined. |
| Impl. Ready | **PARTIAL** | Would be READY if the Evidence schema were merged and the backup contract amended. |
| Authorized | **NO** | SRD-001 stands: no product implementation this round. |

**Verdict:** M01 is the only module where the *technology* is demonstrated rather than described.

### M02 — Retrieval

| Axis | State | Why |
|---|---|---|
| Design | PARTIAL | `02` defines the five stages, but the tier/lifecycle material it depends on is incomplete (OI-R-07), and `02 §7.4` contradicts itself on `structured` (SC-02). |
| Contract | PARTIAL | `RETRIEVAL-CONTRACT.md` written this round; RRF documented example is invalid (OI-R-03). |
| Schema | PARTIAL | `retrieval-response.schema.json` restricts channels to `fts`/`vector`; `read-request` had a BOM. |
| Dependency | PARTIAL | LanceDB works on the scan path only; FTS5 Chinese fails its criterion. |
| Test | PARTIAL | RRF/dedup/lifecycle/pipeline executed (75 checks); **no end-to-end retrieval test against a real index**. |
| Evidence | PARTIAL | Strong per-component evidence; no integrated evidence. |
| Security | PARTIAL | Fail-closed proven in a synthetic harness; **the hardened cache/revalidation path lives in the prototype, not in a product module**. |
| Impl. Ready | **NOT_READY** | Both channels have unresolved acceptance failures. |
| Authorized | **NO** | — |

**Verdict:** M02 is where the two hardest blocking facts live: **Chinese FTS fails PC-1**, and **indexed vector recall is 0.30**.

### M03 — Extraction

| Axis | State | Why |
|---|---|---|
| Design | PARTIAL | `09` defines 16 stages well; candidate 6-state and AUTO_ACCEPT/CR-01 tension remains. |
| Contract | PARTIAL | `[FUTURE]`-tagged in `api-contracts.md §4` — a contract placeholder, not a contract. |
| Schema | PARTIAL | `candidate.schema.json` exists; business fixtures absent. |
| Dependency | **NOT_READY** | Requires an LLM Gateway that does not exist and must not be built this round. |
| Test | NOT_READY | No LLM harness executed (MC-AUD-019). |
| Evidence | NOT_READY | — |
| Security | N/A | M03 produces candidates only; it holds no authority. |
| Impl. Ready | **NOT_READY** | — |
| Authorized | **NO** | P3 requires separate phase authorization. |

### M04 — Integration

| Axis | State | Why |
|---|---|---|
| Design | PARTIAL | `12` defines 7-step profile integration; conflict semantics partly open. |
| Contract | PARTIAL | `[FUTURE]` placeholder. |
| Schema | PARTIAL | `person-profile.schema.json` present; no fixtures. |
| Dependency | NOT_READY | Depends on M03 output and the LLM Gateway. |
| Test | NOT_READY | — |
| Evidence | NOT_READY | — |
| Security | NOT_READY | **M04 is the sole writer of `validation_state=confirmed`.** No test proves that only M04 can write it, because M04 does not exist. |
| Impl. Ready | **NOT_READY** | — |
| Authorized | **NO** | P4 requires separate authorization. |

### M05 — Permission

| Axis | State | Why |
|---|---|---|
| Design | READY | `04` is unusually complete: ten principles, fail-closed, ten-link coverage. |
| Contract | PARTIAL | `permission-context.schema.json` + §6 API; `canRead` vs `read` naming mismatch. |
| Schema | PARTIAL | Enum is coherent; `visibility` vs `scope` duality persists across entities. |
| Dependency | **BLOCKED** | **Real host identity is unavailable and must not be connected.** Synthetic provider exists; real adapter does not. |
| Test | PARTIAL | 222 assertions executed synthetically. |
| Evidence | PARTIAL | All synthetic; labelled correctly. |
| Security | PARTIAL | Fail-closed holds under the hardened profile; six contrastive weak-profile leaks prove the hardening is load-bearing. **But a leak in a real cache implementation would be live.** |
| Impl. Ready | **BLOCKED** | Cannot be ready without a real identity source, which is barred by authorization. |
| Authorized | **NO** | — |

## 3. What would move each module forward

| Module | Single next action |
|---|---|
| M01 | merge the v2 Evidence schema; amend `DatabaseAdapter.backup()` contract; freeze driver within its stated scope |
| M02 | pick a Chinese tokenizer option (unigram+bigram shadow is measured best) and pass the FTS G1–G5 gate; resolve vector index policy (scan-path-only vs indexed with measured recall) |
| M03 | build the LLM harness against a mock provider — no product code, no real model |
| M04 | specify the conflict-resolution matrix precisely enough to test |
| M05 | define the host adapter interface and the real-host verification precondition list |

## 4. The authorization column, stated plainly

`IMPLEMENTATION_READY` and `IMPLEMENTATION_AUTHORIZED` are **different columns on purpose**, and this
round may not set the second. Even if M01's readiness became READY tomorrow, the authorization stays
**NO** until the user grants it. Isolated verification code — 119 source files, 598 evidence
records — does not constitute, and must not be read as, a start of product implementation.

## 5. Confidence

- **High** on M01's technology readiness: measured, reproducible, 0 failing checks.
- **High** on M02's blocking facts: both are measured (FTS hit rates, ANN recall 0.30).
- **Low** on M03/M04: almost nothing executed, so the NOT_READY is a statement about absence of
  evidence rather than about discovered problems.
- **Medium** on M05: the synthetic suite is thorough, but its results say nothing about a real host.
