# PERMISSION-SECURITY-VERIFICATION — TECH-IDENTITY-01 parent review

Date: 2026-11-05 (Asia/Shanghai)
Area: `03-verification/identity/`
Evidence: `06-evidence/identity.jsonl` — **222 records, one run**, ids `EV-IDENT-0001..0222`
Raw: `06-evidence/raw/identity/` (incl. `attacks.md`)
Status: **SYNTHETIC_IDENTITY_VERIFIED** — and that is explicitly **not** real-host verification.

---

## 1. The boundary this report must not cross

> `SYNTHETIC_IDENTITY_VERIFIED != REAL_HOST_MAPPING_VERIFIED`

| | Synthetic (this round) | Real host (not done) |
|---|---|---|
| Identities | constructed by a deterministic synthetic provider | must come from DSH / QQ Bridge |
| Group membership | supplied by the harness | must come from the host |
| Session attribution | supplied by the harness | must be derived from real session state |
| Revocation | simulated | must come from a real revocation feed |

All 222 records carry `synthetic_identity: true` and `real_host_verified: false`. **Zero** records
claim real-host verification. Real-host validation was **not permitted** and was not attempted.

## 2. Result

**222 executed assertions. 216 PASS, 6 FAIL. 0 failures on the hardened profile.**

The 6 FAILs are **negative controls**, not vulnerabilities: each is produced by a deliberately
broken `weak_v1` reference profile, and each has a hardened counterpart in the same run that BLOCKs
the identical attack.

| Attack | `weak_v1` (mutant) | Hardened (default) | What the contrast proves |
|---|---|---|---|
| Revocation outrun by cache | **LEAKED** `EV-IDENT-0132` | **BLOCKED** `EV-IDENT-0126` | per-hit revalidation is load-bearing |
| Memory bodies cached at rest | **LEAKED** `EV-IDENT-0133` | **BLOCKED** `EV-IDENT-0124` | "ids, not bodies" is a real requirement |
| Cross-group leak via cache key | **LEAKED** `EV-IDENT-0134` | **BLOCKED** `EV-IDENT-0135` | group/tenant/session/epoch in the key is load-bearing |
| Same bot, two groups | **LEAKED** `EV-IDENT-0162` | **BLOCKED** `EV-IDENT-0160` | the cache key, not only the engine, enforces isolation |
| Evidence stricter than its memory | **LEAKED** `EV-IDENT-0154` | **BLOCKED** `EV-IDENT-0150` | the second per-evidence check is load-bearing |
| Missing identity defaulted away | **LEAKED** `EV-IDENT-0165` | **BLOCKED** `EV-IDENT-0166` | "no default actor" is load-bearing |

This is the correct way to test a security property: **break it deliberately and show the break
matters**. A suite that only shows green tells you nothing about whether the rule earns its place.

**A green run here would be a harness regression**, not an improvement — it would mean the mutant
stopped mutating.

## 3. Fail-closed, proven by breaking the engine

| Failure injected | Result |
|---|---|
| missing identity | DENY (`0098/0080/0166`) |
| mapping integrity failure | DENY `MAPPING_INTEGRITY_FAILURE` (`0024/0081`) |
| permission engine throws | DENY (`0090`) |
| permission engine times out | DENY (`0091`) |
| unknown principal | DENY (`0022/0079/0039`) |
| revoked access | DENY (`0025/0082/0040`) |
| stale cache | revalidate (`0127/0129/0130/0126`) |
| **INDETERMINATE × 7 injections** | **0 allows** (`0093`) |
| total degradation | `failed`, 0 items, 0 ids (`0117`) |
| final M02 check | DENY on the final path (`0198/0200`) |

`INDETERMINATE` never becomes `ALLOW` — this is the single most important permission invariant and it
holds under injection.

## 4. Allow-all isolation

Four independent controls, 22 assertions, 0 failures:
1. call-site guard (`0169`)
2. construction guards (`0174`)
3. static `require`-closure excludes the double and the registry (`0179/0180`)
4. the **real** `security.schema.json` rejects `test_allow_all.enabled=true` and
   `default_policy=allow` (`0184/0185`)

Under `contract_deny`: 3 DENY (`M05_NOT_IMPLEMENTED`), **0 engine calls** (`0175/0176`).

The fourth control is the strongest: the allow-all double is rejected by the shipped configuration
schema, so it cannot be enabled by configuration in a non-test environment.

## 5. Open security questions (not closed by this round)

| ID | Question | Severity |
|---|---|---|
| SEC-1 | **`actor.role` is optional and nullable.** `actor` does carry the requester (`actor.type` + `actor.id`, with `role` including `requesting_user`), and `subject_id` is documented as the memory's owner — so the schema **does** separate requester from owner. But `actor.required` is only `["type","id"]`, and `role` permits `null`. A caller can therefore send a requester with no role, and any policy that keys off role must then fall back to a default. | P1 |
| SEC-2 | In-process isolation is not a boundary. The harness proves the double is unreachable *within this process*; packaging must guarantee it never ships. | P1 |
| SEC-3 | No asynchronous invalidation story exists. Revocation is only observed at the next per-hit revalidation. A cached result served before revalidation is stale by construction. | P1 |
| SEC-4 | Revoke semantics are undefined (authorization vs availability, OI-04-05). Two different operations share one verb. | P1 |
| SEC-5 | "Strictest evidence" has no formal order (DG-4). "Take the strictest of two visibilities" is described but not defined as an ordering relation. | P2 |
| SEC-6 | Group session attribution (OI-04-03) undefined: a session that spans group and private contexts has no specified attribution rule. | P1 |

**Correction to an earlier claim.** This report initially recorded SEC-1 as "the schema has no field
for the requester". That was **wrong**, and the parent's own re-read of
`permission-context.schema.json` refuted it:

- `actor{type,id}` **is** the requester (a principal distinct from the memory's owner);
- `actor.role` enumerates `requesting_user` alongside `memory_owner`, `group_member`,
  `group_administrator`, `bot_owner`, `system_service`, `maintenance_operator`;
- `subject_id` is explicitly documented as "记忆所属主体" — the memory's owner, not the caller.

So the requester/owner distinction **is** representable. The defect is the weaker, real one above:
`role` is optional and nullable, so the distinction is not *mandatory* at the contract level.

The agent's finding was directionally useful — it flagged that a literal reading of `subject_id`
as the caller would be catastrophic — but its stated form ("no field for the requester") was
incorrect, and the parent's ledger-level check should have caught it before it reached a report.
It did not, because the first re-read looked only at `required` and not at the `actor` sub-object.
That is recorded here rather than quietly edited away.

## 6. Verdict

| Question | Answer |
|---|---|
| Is the permission model fail-closed under injected failure? | **Yes**, proven by injection, not assertion |
| Is the hardening load-bearing? | **Yes**, proven by contrast against a mutant |
| Are the six FAILs vulnerabilities? | **No** — negative controls, each with a hardened counterpart |
| Is the *schema* adequate to express the model? | **No** — SEC-1, no requester field |
| Is real-host mapping verified? | **No, and not attempted** — barred |
| Can the permission contract freeze? | **No.** Gates: SEC-1 (schema), SEC-3 (invalidation), SEC-4 (revoke semantics), plus a real host. |

## 7. Cross-check performed by the parent

- Confirmed the 6 FAIL records are all from `playbook.profile = weak_v1` and that the 4 checked
  hardened counterparts PASS. (Reported by the agent; re-read from the ledger.)
- Confirmed **all 222** records have `synthetic_identity: true` / `real_host_verified: false`.
- Confirmed 222 unique evidence ids with no duplicates.
- Confirmed the ledger is truncated per run, so these 222 lines are **one** run, not an accumulation.
- Independently confirmed the requester-field gap is real by reading `permission-context.schema.json`:
  its `required` list is `actor, tenant_id, action, timestamp`, and `subject_id` is documented as
  "记忆所属主体". There is no separate requester field.
