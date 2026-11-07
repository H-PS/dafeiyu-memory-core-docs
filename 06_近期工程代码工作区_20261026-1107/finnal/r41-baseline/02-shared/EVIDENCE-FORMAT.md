# EVIDENCE FORMAT (binding)

Every executed check produces exactly one JSON object, appended as one line to
`E:\[PROJECT]\r41-baseline\06-evidence\<area>.jsonl`.

## Record schema

```json
{
  "evidence_id": "EV-SQLITE-001",
  "task_id": "TECH-SQLITE-01",
  "test_id": "sqlite.file_database_creation",
  "test_name": "Create a file-backed database and reopen it",
  "script_path": "03-verification/sqlite/file-backed-smoke.cjs",
  "environment": {
    "node": "v24.21.0",
    "sqlite": "3.53.4",
    "os": "Windows NT 10.0.26200.0",
    "arch": "x64",
    "cwd": "E:\\[PROJECT]\\r41-baseline"
  },
  "dependency_versions": { "node:sqlite": "builtin", "@lancedb/lancedb": null },
  "input_dataset": { "path": "03-verification/sqlite/_data/form-corpus.json", "sha256": "…" },
  "executed_at": "2026-11-05T20:10:00+08:00",
  "duration_ms": 42,
  "exit_code": 0,
  "expected": "row count after reopen == 3",
  "actual": "row count after reopen == 3",
  "status": "PASS",
  "conclusion": "File-backed database persists across close/reopen.",
  "limitations": "Single process, single connection; does not exercise concurrent writers.",
  "raw_output_path": "06-evidence/raw/sqlite/file-backed-smoke.run1.log"
}
```

## Status vocabulary (exactly these strings)

| Status | Meaning |
|---|---|
| `PASS` | The check executed on this machine in this session and met its stated expectation. |
| `PARTIAL_PASS` | Executed; some asserted sub-conditions met, others not, and the record says which. |
| `FAIL` | Executed; expectation not met. |
| `NOT_RUN` | Not executed in this session. No PASS record may exist for it. |
| `NOT_VERIFIED` | Executed indirectly or observed but not established to the required standard. |
| `BLOCKED` | Cannot execute because a precondition (dependency, host, authorization) is unmet. State the precondition. |
| `NOT_APPLICABLE` | The check does not apply in this environment; state why. |

## Hard rules

1. `executed_at`, `duration_ms`, `exit_code` must come from the actual run.
2. `expected` and `actual` are both mandatory on every executed record, in observable terms.
3. `limitations` is mandatory. An empty limitation field is rejected.
4. `VERIFIED` in a report must always carry its scope, e.g.
   `VERIFIED(file-backed WAL and transaction smoke tests)`, never a bare `VERIFIED(SQLite driver)`.
5. A `PASS` for a *harness* is not a `PASS` for the *technology*. Keep them as separate records.
6. If a dataset is generated rather than stored, store it **and** record the generator seed and
   the hash of the produced file.
