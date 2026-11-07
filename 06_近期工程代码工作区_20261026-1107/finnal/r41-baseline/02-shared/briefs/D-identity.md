# BRIEF — Agent D · TECH-IDENTITY-01 · Host identity mapping and permission safety

Read first:
1. `E:\[PROJECT]\r41-baseline\02-shared\00-PARENT-MANDATE.md`
2. `E:\[PROJECT]\r41-baseline\02-shared\EVIDENCE-FORMAT.md`
3. `01-baseline-corpus\design-package\04-权限与治理规范.md` (the whole document — it is the authority here)
4. `01-baseline-corpus\design-package\contracts\schemas\permission-context.schema.json`
5. `01-baseline-corpus\design-package\contracts\api\api-contracts.md` §6 (permission section)
6. `01-baseline-corpus\design-package\08-DSH与QQBridge集成规范.md` (identity mapping section)

Own this area only: `03-verification\identity\`. Evidence: `06-evidence\identity.jsonl`; raw: `06-evidence\raw\identity\`.

## Absolute boundary — read twice

`SYNTHETIC_IDENTITY_VERIFIED != REAL_HOST_MAPPING_VERIFIED`.

You are forbidden from connecting real QQ Bridge or DSH identity data. Every identity in your
harness is synthetic and must be labelled synthetic in both the record and the report. You must
never phrase a synthetic result in a way that reads as a real-host verification. The parent will
check your wording specifically.

## Contract work (part of your deliverable)

Produce `03-verification\identity\HOST-IDENTITY-MAPPING-CONTRACT.md` defining, with explicit
uniqueness and failure rules for each:

Host User ID · Internal User ID · Session ID · Group ID · Message ID · Conversation ID ·
Principal · Resource Owner · Data Visibility · Permission Scope.

It must state: identity source; mapping rule; uniqueness constraint; behaviour on missing identity;
behaviour on invalid mapping; identity-conflict resolution; session attribution; group attribution;
identity revocation; mapping version; audit trail.

**Do not assume Host ID equals Internal User ID.** The whole point is that it does not.

## Synthetic identity provider

`SyntheticIdentityProvider` covering at minimum: owner; ordinary user; group member; non-member;
unknown user; missing identity; invalid mapping; revoked identity; conflicting identity;
session mismatch. Deterministic and seeded, so a rerun produces the same universe.

## Permission matrix you must actually execute

Private read; shared read; owner access; non-owner access; cross-group access; non-member access;
session mismatch; missing identity; mapping failure; permission-service failure; revocation;
cache staleness; evidence-reference access; final retrieval authorization.

`mock-permission-provider.cjs` must be able to return ALLOW / DENY / INDETERMINATE / THROW / TIMEOUT,
so that every failure path is exercised, not just the happy path.

## Invariants to prove by test, not by assertion

- Identity missing → DENY
- Mapping failure → DENY
- Permission service failure → DENY
- Unknown principal → DENY
- Revoked access → DENY
- Cache stale → revalidation (not a stale ALLOW)
- Final M02 check fail-closed; `INDETERMINATE` never becomes ALLOW
- Allow-all is permitted **only** inside an explicitly isolated synthetic test, and the test must
  prove that the allow-all provider cannot be reached from a non-test code path.

Actively attempt to break these and record the attempts:

- retrieval bypassing permission
- cache bypassing permission
- evidence access bypassing permission
- error degradation widening visibility
- cross-group data leakage
- private memory entering a shared context

For each attempted bypass, record: attack, mechanism, result (`BLOCKED` / `LEAKED`), evidence id.
**A `LEAKED` finding is a high-value deliverable — report it plainly and loudly.**

## Scale

Aim for a real matrix: on the order of 100+ executed assertions across the ten identities × the
permission scenarios × the failure modes, not a handful of spot checks. Record the actual count.

## Deliverable

`03-verification\identity\TASK-RESULT.md` plus `run-all.cjs` and `README.md`.
Include: the mapping contract, the executed permission matrix, fail-closed evidence, every
unresolved security question, the future host adapter interface (what a real adapter must supply:
`getPrincipal`, `getGroupMembership`, `getSessionAttribution`, revocation feed, mapping version),
and the precise preconditions required before any real-host verification may begin.
State explicitly whether real-host validation is permitted — it is not, in this session.
Parent will re-run `run-all.cjs`.
