# SECOND-ROUND-EVIDENCE-RECONCILIATION — round 4.1

Date: 2026-11-05 (Asia/Shanghai)

Reconciles **(a)** the evidence the prior rounds claimed, **(b)** the evidence this round produced,
and **(c)** where those disagree. Every line traces to a ledger record or a file.

---

## 1. Evidence volume, measured

| Ledger | Records | PASS | PARTIAL | FAIL | BLOCKED |
|---|---:|---:|---:|---:|---:|
| `06-evidence/sqlite.jsonl` | 55 | 50 | 5 | 0 | 0 |
| `06-evidence/lancedb.jsonl` | 66 | 58 | 1 | 7 | 0 |
| `06-evidence/fts.jsonl` | 71 | 56 | 13 | 2 | 0 |
| `06-evidence/identity.jsonl` | 222 | 216 | 0 | 6 | 0 |
| `06-evidence/retrieval.jsonl` | 77 | 75 | 1 | 1 | 0 |
| `06-evidence/outbox.jsonl` | 81 | 81 | 0 | 0 | 0 |
| `06-evidence/schema.jsonl` | 20 | 11 | 3 | 5 | 1 |
| `06-evidence/audit.jsonl` | 6 | 6 | 0 | 0 | 0 |
| **Total** | **598** | **553** | **23** | **21** | **1** |

Raw artefacts: `06-evidence/raw/` — 14 directories, real stdout/stderr per suite.

**All 598 records parse as JSON and carry `expected`, `actual`, `limitations`, `duration_ms`,
`exit_code` and a `raw_output_path`.** No record was found with an empty limitation field.

## 2. Prior-round claims vs this round's findings

| Prior claim | Source | This round | Verdict |
|---|---|---|---|
| "File DB WAL, migration, backup/restore, lock contention and crash recovery all passed" | `SECOND-ROUND-FULL-AUDIT-REPORT.md §5` | 13/13 claimed checks **are** backed by artefacts and harness code | **Upheld** — with one exception below |
| "only in-memory smoke was done" | `TECHNICAL-VERIFICATION-REPORT.md` | Two file-backed artefacts exist and were ignored by that report | **Report inaccurate** |
| "crash recovery passed" | Full Audit Report | The prior harness's child does `BEGIN; INSERT; process.exit(0)` — a **clean exit**, not a crash; and its `child_exit` check asserts `status===0` for a child scripted to exit 0 | **Overstated.** This round replaced it with real `SIGKILL` tests |
| LanceDB "max cosine-distance error 5.6926e-9, tolerance 1e-5" | `lancedb-result.json` | `_distance` is squared-L2; the comparison was between non-comparable quantities; tolerance is 1757× the error; no stored dataset exists | **Does not stand** |
| LanceDB "filter/update/delete/reopen passed" | same | Re-confirmed by this round | **Upheld** |
| "31 Draft-07 schemas meta-valid" | `schema-validation-result.json` | Reproduced: 31/31 meta-valid — **after** removing a BOM that made one file unparseable | **Upheld with correction** |
| "evidence.schema.json compliant with authority" | implied by contract layer | **Refuted**: rejects a §3.2-compliant instance | **Refuted** |
| FTS "English passed, Chinese no-hit" | `sqlite-file-result.json` | Refined: it is **run-equality retrieval**, not total failure — 15/19 Chinese queries do hit | **Upheld and refined** |
| Outbox "isolated simulation passed" | `outbox-simulation-result.json` | Extended to 81 checks including measured backoff, lease CAS, crash recovery | **Upheld and extended** |

## 3. Irreconcilable provenance facts

| Fact | Consequence |
|---|---|
| Every prior result file records **node v24.22.1**; this machine runs **v24.21.0** | Prior results cannot be reproduced byte-for-byte here. Their numbers may be right; they are not *this environment's* numbers. |
| No npm/pnpm/npx exists | Prior "installed via pnpm" evidence cannot be re-derived; no driver comparison is possible. |
| LanceDB `:memory:` is unsupported | Any prior LanceDB test implying `:memory:` is suspect. None was found, but the class exists. |

## 4. Defects found in this round's own evidence discipline

Recorded because an evidence set that hides its own weaknesses is worthless.

| Defect | Example | Handling |
|---|---|---|
| `actual` string omitted the failing sub-condition | `EV-RETRIEVAL-087/088` printed passing values while failing on an unprinted third condition | Parent re-executed; `EV-RETRIEVAL-P001` corrects a **false failure**; `P002` confirms a **genuine** one |
| Append-only ledgers without a `run_id` | `outbox.jsonl` has the same test 2–3× from separate runs with no run discriminator; `fts.jsonl` does carry `run_id` | Recorded; readers must not average across runs. Recommendation: mandatory `run_id` in the evidence schema |
| Duplicate-subject records inflating FAIL counts | `EV-LANCEDB-CONF-012` and `CONF-013` cover the same NaN-vs-Infinity subject | Flagged to the agent; not silently de-duplicated by the parent |
| A test flag invented to make a red check green | `tier_review_approved` — zero occurrences in the design | Rejected as evidence (`PARENT-CORRECTIONS §2`) |
| Custody breach | Parent stripped a BOM from the read-only corpus at 20:11:26 | Measured (1 file, BOM only), recorded (`PARENT-CORRECTIONS §1`), independently corroborated by `EV-AUDIT-H-006` |

## 5. What remains unverified

| Area | Status | Gate |
|---|---|---|
| API contract behaviour | **NOT_RUN** | needs a mock HTTP harness |
| LLM task behaviour | **NOT_RUN** | needs a mock provider |
| Token budget | **NOT_RUN** | needs a real tokenizer |
| End-to-end retrieval | **NOT_RUN** | needs the channels to pass their own gates first |
| Real host identity | **BLOCKED by authorization** | must not be attempted this round |
| ANN recall at acceptance scale | **NOT_RUN** | measured 0.30 at one scale; needs a fixed acceptance corpus |
| Power-loss durability | **NOT_RUN** | SIGKILL is not power loss |

## 6. Reconciliation verdict

The prior round was **partly right and partly unsupported**, and the distribution matters:

- Its **SQLite** claims were substantially backed — one claim ("crash recovery") was overstated,
  and one report understated the work that had actually been done.
- Its **LanceDB** claim did not survive audit: the number was real but measured the wrong quantity.
- Its **FTS** observation was correct and understated: "Chinese no hit" is really "Chinese is
  matched only by exact run equality", which is worse in consequence and better in precision.

This round produced **598 records** with raw logs and honest scoping. It also produced **21 FAIL
records**, and the majority are findings rather than harness breakage — but two were false failures
and one was a genuine one masked behind a fabricated precondition. All three are now separately
recorded so the count cannot be read naively.
