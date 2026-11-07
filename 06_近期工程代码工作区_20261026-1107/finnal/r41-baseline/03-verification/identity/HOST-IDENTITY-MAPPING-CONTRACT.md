# HOST-IDENTITY-MAPPING-CONTRACT

**Task:** TECH-IDENTITY-01 (Agent D) — Memory Core R4.1 baseline
**Area:** `03-verification/identity/`
**Status:** DRAFT CONTRACT, proven only against synthetic identities.

> ### SYNTHETIC_IDENTITY_VERIFIED != REAL_HOST_MAPPING_VERIFIED
>
> Every identifier, mapping, session and revocation in this document was executed against
> `SyntheticIdentityProvider` — an in-repo generator of fake principals. **No QQ Bridge was
> contacted. No DSH identity data was read. No real host mapping was observed.**
> What is proven here is that the *rules* are internally consistent and fail closed when they are
> violated. Whether the real host supplies data that satisfies those rules is **NOT_VERIFIED** and
> cannot be verified in this session (see §12).

---

## 1. Scope and authority

This contract defines how a host-supplied identity becomes a decision-bearing principal, and what
must happen when that transformation cannot be completed. It exists because the design package
states the prohibitions but not the derivation:

| Source | What it fixes | What it leaves open |
|---|---|---|
| `04-权限与治理规范.md` §7.1–7.6 | five identifiers exist; default deny; group id ≠ user id; nickname ≠ identity; tenants never merge | no derivation function, no uniqueness domain, no conflict rule |
| `contracts/api/api-contracts.md` §6 | `permission-context.schema.json` shape | schema carries no field for the requester's own subject (see DG-2) |
| `08-DSH与QQBridge集成规范.md` §7.3–7.4 | host adapter duties; `private:<qq>` must not become a session id | no mapping table, no revocation feed format, no epoch semantics |
| `contracts/config/security.schema.json` | `test_allow_all.enabled` const false; `default_policy` const deny | — |

Everything below is *this baseline's* reading of those sources. Where a source is silent, the
choice is marked **[DERIVED]** and carries a rationale; those are the points a reviewer should
attack first.

**Normative language.** MUST / MUST NOT / SHOULD follow RFC 2119. A MUST that is violated by a
*returned value* rather than by an exception is still a violation: this harness records violations
as `FAIL` evidence, it does not throw.

---

## 2. The eleven objects at a glance

| # | Object | Source of truth | Shape (synthetic example) | Unique within | Identity-bearing? |
|---|---|---|---|---|---|
| O1 | **Host User ID** | QQ Bridge (`event.user_id`) | `qq:100002` | host platform | no — external handle |
| O2 | **Internal User ID** | derivation from O1 | `iu_621ac55a394e1d66` | tenant | yes |
| O3 | **Session ID** | host session builder | `s_priv_100002_01` | tenant | yes — isolation axis |
| O4 | **Group ID** | QQ Bridge (`event.group_id`) | `g_synth_01` | host platform | no — external handle |
| O5 | **Message ID** | QQ Bridge (`event.message_id`) | `msg_synth_0001` | (tenant, bot) | no — lineage only |
| O6 | **Conversation ID** | host addressing key | `private:100002` | host conversation space | **no — MUST NOT be promoted** |
| O7 | **Principal** | composed from O2+O3+O4 | `{subject_id, internal_user_id, …}` | per decision | yes — the decision subject |
| O8 | **Resource Owner** | memory record | `owner_subject_id = sub_330fc…` | per resource | yes |
| O9 | **Data Visibility** | memory record `scope` | `owner_private` | per resource | no — a policy input |
| O10 | **Permission Scope** | request `scopes_requested` + policy | `["owner_private"]` | per request | no — a policy input |
| O11 | **Group Subject** | derivation from O4 | `sub_3bc55bb0c1011afa938c` | tenant | yes — owner of group memories |

Derivation functions actually executed (module 01, `EV-IDENT-0001`…`EV-IDENT-0040`):

```
internal  = "iu_" | "ig_"  + first 16 hex of HMAC-SHA256(tenant_key, kind + 0x1f + host_id)
subject   = "sub_"         + first 20 hex of HMAC-SHA256(tenant_key, "subject" + 0x1f + internal)
```

* Internal user id length **19**, group internal id length **19**, subject id length **24**.
* `tenant_key` is derived from `(tenant_id, mapping_version)`; two tenants therefore cannot collide
  even on an identical host id — asserted by `contract.subject_unique_across_tenants`
  (`EV-IDENT-0221`).
* The derivation is deterministic and pure: same inputs ⇒ same output in every process, which is
  what makes the evidence file reproducible (`_scratch` determinism probe: 222 records, 0 diffs).

> These ids are **synthetic**. A different `tenant_key` in production yields different values; the
> statement being verified is "the function is total, injective on the tested domain, and
> independent per tenant", not "these are the production ids".

---

## 3. Per-object contract

Each object lists the ten facets the task requires. `n/a` entries state *why* the facet does not
apply — an empty facet is not permitted.

### O1 — Host User ID

| Facet | Rule |
|---|---|
| **Source** | QQ Bridge event payload, `user_id`, namespaced as `qq:<digits>`. |
| **Mapping rule** | Never used directly as a subject. Input to O2 only. A row in the mapping table maps `(tenant_id, host_user_id) → internal_user_id`. |
| **Uniqueness** | Unique per host platform. **Not** unique across tenants or platforms: `qq:100002` in tenant A and tenant B are different people until proven otherwise. |
| **Missing** | Empty/absent ⇒ `IDENTITY_MISSING` ⇒ DENY, no principal is constructed. Proven: `atk.F4.hardened_denies` (`EV-IDENT-0166`), `fc.identity.missing_*`. |
| **Invalid** | Present but malformed, unmapped, revoked, ambiguous or integrity-failed ⇒ specific reason code (§4), always DENY. |
| **Conflict** | Same host id resolving to two internal ids ⇒ `IDENTITY_CONFLICT`; the first row in registration order is primary, the alias needs an audited merge grant (`authorizeMerge` throws `MERGE_GRANT_UNAUDITED` without `granted_by` + `audit_id`). |
| **Session / group attribution** | Not attributable by itself — a host user id in a group message does **not** prove group membership. Membership comes from O4 + the membership table. |
| **Revocation** | Revoking the host id revokes every internal id that resolves to it, in the same epoch. |
| **Mapping version** | Every row carries the `mapping_version` that created it; a row from a superseded version is not usable. |
| **Audit trail** | The host id MUST NOT appear in the audit trail — only the internal id does (`audit.no_host_identifier`, 0 occurrences of `qq:` across all entries). |

### O2 — Internal User ID

| Facet | Rule |
|---|---|
| **Source** | `deriveInternalId(tenant, 'user', hostUserId)`. |
| **Mapping rule** | The *only* user identifier that may appear in a permission context, an audit record or a memory owner field. |
| **Uniqueness** | Unique within a tenant. Two distinct host ids in one tenant MUST NOT share one internal id (checked: 7 distinct users ⇒ 7 distinct ids in module 01). |
| **Missing** | The internal id is never optional once a principal exists; a principal without one is a construction error (`PROVIDER_NOT_PERMITTED` path in the gate). |
| **Invalid** | An internal id that does not appear in the mapping table is `IDENTITY_UNKNOWN` ⇒ DENY; it is not "trusted because it is well-formed" (`atk.F4.hardened_denies`). |
| **Conflict** | Same as O1 — one internal id may accumulate aliases only through the audited merge path. |
| **Session / group attribution** | Carries no attribution of its own; the same internal id can appear in many sessions, which is exactly why the session is a separate axis. |
| **Revocation** | `revoked` ⇒ `IDENTITY_REVOKED` ⇒ DENY, even for a resource the user owns. |
| **Mapping version** | Resolved id is stamped with `{mapping_version, mapping_epoch}` and that stamp is on the principal. |
| **Audit trail** | `actor` field of every decision. |

### O3 — Session ID

| Facet | Rule |
|---|---|
| **Source** | Host session builder. **[DERIVED]** format `s_<kind>_<host user digits>_<nn>` for private, `s_group_<nn>_<nn>` for group. |
| **Mapping rule** | MUST be minted by the host, MUST NOT be derived from `Conversation ID` (08 §7.3.2 forbids `private:<qq>` as a session id). A session belongs to exactly one principal and at most one group. |
| **Uniqueness** | Unique within a tenant, **globally for the lifetime of the mapping epoch** — session ids are never reused, not even after expiry. |
| **Missing** | Private request with no session ⇒ allowed to resolve but cannot read `session_only` resources. Group request with **no** session ⇒ `SESSION_MISSING` ⇒ DENY (proven `attrib.group_without_session_denied`, `EV-IDENT-0205`; reason `SESSION_OWNER_MISMATCH` for a foreign session, `idn.universe.I10_session_mismatch`, `EV-IDENT-0027`). |
| **Invalid** | Session that does not belong to the resolved principal (`sessionMismatch`, `qq:100010`) ⇒ `SESSION_OWNER_MISMATCH` ⇒ DENY. Session whose group differs from the request group ⇒ `SESSION_GROUP_MISMATCH` ⇒ DENY. |
| **Conflict** | A session presented by two principals ⇒ second use fails; no silent rebind. |
| **Session / group attribution** | This *is* the session axis. `session_only` memories are readable only when `request.session_id === resource.session_id` **and** the principal owns the resource. |
| **Revocation** | Ending a session invalidates `session_only` resources created in it (semantics still open, see DG-6/OI-04-03). |
| **Mapping version** | Session rows carry the epoch that minted them; a session from a previous epoch is not resolvable. |
| **Audit trail** | `request_id` is derived from the session + request sequence, so every decision is traceable to one session. |

### O4 — Group ID

| Facet | Rule |
|---|---|
| **Source** | QQ Bridge `group_id`, namespaced `g_synth_*` in this harness. |
| **Mapping rule** | Input to O11 (group subject) and to membership lookup. **A group id is never a user id** (04 §7.4.1 prohibition 1): it cannot be used as `actor.id`, and a group subject cannot own a private memory. |
| **Uniqueness** | Unique per host platform; two tenants may legitimately use the same group id for different groups, so the pair `(tenant_id, group_id)` is the key. |
| **Missing** | Absent group in a private request ⇒ `group_id = null`, which forbids `group_shared` and forbids any `owner_private` in a group context. |
| **Invalid** | A group the principal is not a member of ⇒ `GROUP_MEMBERSHIP_MISSING` ⇒ DENY (proven `atk.E2.non_member_in_group_context`, `EV-IDENT-0161`). |
| **Conflict** | Same `(tenant, group_id)` mapped to two group subjects ⇒ `GROUP_MAPPING_CONFLICT` ⇒ DENY. |
| **Session / group attribution** | Attribution requires **group match AND session match**; the session must be a session of that group (`SESSION_GROUP_MISMATCH` otherwise). |
| **Revocation** | Removing a member is a membership change, not an identity revocation; it MUST bump the epoch so caches cannot serve the old answer. |
| **Mapping version** | Membership rows carry the epoch. |
| **Audit trail** | Group reads record the target as `memory:<id>`; the group id appears only as part of the request context, never as an actor. |

### O5 — Message ID

| Facet | Rule |
|---|---|
| **Source** | QQ Bridge `message_id`. |
| **Mapping rule** | Lineage only. It identifies the message an evidence row came from (`evidence.source_message_id`). **It MUST NOT be used as a deduplication key for identity, nor as a principal.** |
| **Uniqueness** | Unique within `(tenant, bot)`; not stable across bridges or after a bridge migration. |
| **Missing** | Evidence that cannot name its source message MUST NOT be created — evidence without provenance is unauditable. |
| **Invalid** | A message id that resolves to a different author than the claimed principal ⇒ the evidence row is quarantined, not silently accepted. |
| **Conflict** | Duplicate message id across two authors ⇒ both rows quarantined; no last-writer-wins. |
| **Session / group attribution** | Carries the session/group of the message it came from; importing a message into another session is a provenance violation. |
| **Revocation** | Deleting a message MUST tombstone every evidence row that cites it (04 §7.9 delete propagation). |
| **Mapping version** | Message ids are not re-mapped; the epoch is recorded for provenance only. |
| **Audit trail** | Not in the decision audit (it is not a decision input). |

### O6 — Conversation ID

| Facet | Rule |
|---|---|
| **Source** | Host addressing key, e.g. `private:100002` / `group:g_synth_01`. |
| **Mapping rule** | Resolve to `(group_id \| host_user_id)` **and then** to a freshly minted session. **MUST NOT be promoted to a session id, a user id or a subject** — it is a routing label with a different lifecycle (08 §7.3.2). |
| **Uniqueness** | Unique in the host's addressing space only; reused after re-login, so it cannot carry identity. |
| **Missing** | A message with no conversation key cannot be routed ⇒ dropped before identity resolution, never defaulted. |
| **Invalid** | A conversation key whose embedded user id disagrees with `event.user_id` ⇒ `CONVERSATION_IDENTITY_MISMATCH` ⇒ DENY. |
| **Conflict** | Two users claiming one conversation key ⇒ DENY both; this is the impersonation shape. |
| **Session / group attribution** | Provides the *hint* for group/user; the actual attribution must come from O3/O4 minted by the host. |
| **Revocation** | n/a — not an identity, nothing to revoke; revoking the underlying user/session is the correct action. |
| **Mapping version** | Not versioned. |
| **Audit trail** | MUST NOT appear. `private:100002` contains the host user id, so recording it would defeat `audit.no_host_identifier`. |

### O7 — Principal

| Facet | Rule |
|---|---|
| **Source** | Composed by `getPrincipal({tenant_id, host_user_id, group_id, session_id})`. |
| **Mapping rule** | `{subject_id, internal_user_id, internal_group_id, session_id, tenant_id, role, derived_from, mapping_version, mapping_epoch}`. The `role` is one of the seven schema roles — no free text. |
| **Uniqueness** | One principal per (tenant, subject, session, group). Group principals are distinct objects derived from O11. |
| **Missing** | Any missing mandatory input ⇒ no principal ⇒ DENY. There is no default principal, no `anonymous`, no `system` fallback (`atk.F4`, `EV-IDENT-0165/0166`). |
| **Invalid** | Roles `memory_owner` / `system_service` MUST NOT be granted by a request; they are only reachable when the host supplies them out-of-band, and even then `elevated_role` is DENIED for user-facing reads. |
| **Conflict** | Two principals for one session ⇒ `SESSION_OWNER_MISMATCH`. |
| **Session / group attribution** | The principal is the carrier of both axes; `derived_from` records which input produced it (`user_id` vs `group_id`). |
| **Revocation** | A revoked identity produces no principal. |
| **Mapping version** | `mapping_version` + `mapping_epoch` are on the principal, so a decision can be replayed against the mapping it used. |
| **Audit trail** | `actor` + `request_id` link the decision to the principal and its session. |

### O8 — Resource Owner

| Facet | Rule |
|---|---|
| **Source** | Memory record `owner_kind` + `owner_ref`, resolved at write time to `owner_subject_id`. |
| **Mapping rule** | Always a **subject id**, never a host id and never a group id string. Group-owned memories carry the group's subject (O11). |
| **Uniqueness** | Per resource — exactly one owner. Co-ownership is not modelled and MUST NOT be faked by putting a group id in the owner field. |
| **Missing** | A resource with no owner is not readable by anyone (`OWNER_UNRESOLVED`); it is not "public". |
| **Invalid** | An owner that is a host id, or a group id used as a user, ⇒ the record is rejected at write time. |
| **Conflict** | Owner rewritten after creation ⇒ MUST bump the resource version so caches revalidate. |
| **Session / group attribution** | Ownership is orthogonal to session: `session_only` resources are owned *and* session-bound. |
| **Revocation** | Revoking an owner does not delete the memory; it removes all readers except `system_service`. |
| **Mapping version** | The owner subject is stable across epochs; if the mapping changes the subject, the memory MUST be re-owned in a migration, not silently re-interpreted. |
| **Audit trail** | Encoded in `target` as `memory:<id>`; the owner is derivable from the resource, not duplicated in the trail. |

### O9 — Data Visibility (scope)

| Facet | Rule |
|---|---|
| **Source** | Memory record `scope`, decided at write time. |
| **Mapping rule** | Closed set: `owner_private`, `group_shared`, `session_only`, `project_scoped`, `system_internal`, `export`. No other value is admissible; unknown scope ⇒ DENY (`SCOPE_UNKNOWN`, shadowed by schema validation — see DG-3). |
| **Uniqueness** | Exactly one scope per memory. |
| **Missing** | No scope ⇒ DENY. |
| **Invalid** | `owner_private` presented in a group context ⇒ `PRIVATE_SCOPE_IN_GROUP_CONTEXT` ⇒ DENY, regardless of who asks — including the owner (`atk.F1`, `EV-IDENT-0163`). |
| **Conflict** | Evidence inherited from a memory has its own scope; when they differ, the **stricter** wins and is re-checked (see O9-evidence below). |
| **Session / group attribution** | `group_shared` requires group match + membership; `session_only` requires session match; `project_scoped` requires project match + membership. |
| **Revocation** | Revoking a resource sets state `revoked`, which denies all scopes at once. |
| **Mapping version** | Scope is data, not mapping; not versioned. |
| **Audit trail** | Recorded in the decision reason code when it is the denying factor. |

**O9-evidence.** An evidence row inherits the *strictest* permission of the memories that cite it and MUST be re-checked on `related_evidence` (04 §7.11). Implemented as a second gate call with the evidence's own `resource_id`. The shortcut "if the memory is readable, its evidence is readable" is defect W3 and is **LEAKED** when implemented (`atk.C5`, `EV-IDENT-0154`).

### O10 — Permission Scope (requested)

| Facet | Rule |
|---|---|
| **Source** | Caller-supplied `scopes_requested` + the policy's allowed set. |
| **Mapping rule** | The request may only *narrow* what the policy allows; the engine is asked about one resource at a time. A request with no scopes ⇒ `SCOPE_MISSING` ⇒ DENY. |
| **Uniqueness** | n/a — a set. |
| **Missing** | DENY (`SCOPE_MISSING`). |
| **Invalid** | A scope not in the closed set ⇒ `SCOPE_UNKNOWN` ⇒ DENY. A scope not in `scopes_requested` ⇒ `SCOPE_NOT_REQUESTED` ⇒ DENY. |
| **Conflict** | Requested scope admits, resource scope differs ⇒ the intersection must be used; a resource scope outside the requested set is DENY, never "upgrade". |
| **Session / group attribution** | Each scope names its own attribution requirement (§3 O9). |
| **Revocation** | A revoked grant cannot be renewed by requesting a wider scope. |
| **Mapping version** | Not versioned; the policy version is (`m05-policy-hp1/1.0.0`). |
| **Audit trail** | `action` field. |

### O11 — Group Subject

| Facet | Rule |
|---|---|
| **Source** | `deriveSubject(tenant, 'group', groupInternalId)`. |
| **Mapping rule** | Owns every `group_shared` memory of that group. **Distinct from every member's subject** (`attrib.group_memory_owned_by_group`, `EV-IDENT-0206`): a group memory is not co-owned by its members. |
| **Uniqueness** | One per `(tenant, group)`. |
| **Missing** | Group memories with no group subject ⇒ unreadable. |
| **Invalid** | Using a member's subject as the owner of a group memory ⇒ the "members can read each other's writes" defect; forbidden. |
| **Conflict** | Two group ids deriving one subject ⇒ `GROUP_MAPPING_CONFLICT`. |
| **Session / group attribution** | The group subject is the *resource owner*; the *actor* is always a user principal. |
| **Revocation** | Disbanding a group revokes the subject; memories become unreadable rather than public. |
| **Mapping version** | Carries the epoch. |
| **Audit trail** | Appears as the target's owner, never as the actor. |

---

## 4. Failure rules — the closed set of reason codes

Every refusal names a reason. "Denied for security reasons" is not an acceptable outcome
(§7.10 requires a machine-readable reason on every decision).

| Reason code | Trigger | Fail-closed? |
|---|---|---|
| `IDENTITY_MISSING` | no host id / no principal | DENY |
| `IDENTITY_UNKNOWN` | host id not in the mapping table | DENY |
| `IDENTITY_REVOKED` | identity revoked in the current epoch | DENY |
| `IDENTITY_CONFLICT` | host id → two internal ids | DENY |
| `MAPPING_INTEGRITY_FAILURE` | malformed mapping row / failed integrity tag | DENY |
| `MAPPING_TYPE_MISMATCH` | a group id presented where a user id is required | DENY |
| `SESSION_MISSING` | group request without a session | DENY |
| `SESSION_OWNER_MISMATCH` | session not owned by the resolved principal | DENY |
| `SESSION_GROUP_MISMATCH` | session belongs to another group | DENY |
| `GROUP_MEMBERSHIP_MISSING` | actor is not a member of the resource's group | DENY |
| `PRIVATE_SCOPE_IN_GROUP_CONTEXT` | `owner_private` with a group id in context | DENY |
| `TENANT_MISMATCH` | resource tenant ≠ principal tenant | DENY |
| `RESOURCE_DELETED` / `RESOURCE_REVOKED` | tombstoned resource | DENY |
| `PERMISSION_SERVICE_ERROR` / `…_TIMEOUT` / `PERMISSION_INDETERMINATE` | engine failed | DENY |
| `M05_NOT_IMPLEMENTED` | `m05_mode=contract_deny` | DENY |
| `GRANT_MISSING` / `GRANT_EXPIRED` / `GRANT_RESOURCE_MISMATCH` | capability check failed | DENY |

`INDETERMINATE` is **never** converted to ALLOW: 7 injected indeterminate outcomes, 0 allows
(`fc.indeterminate.never_allow`, `EV-IDENT-0093`).

---

## 5. Mapping version and epoch

* `mapping_version` — a semver string (`1.0.0`). Changes when the *shape or semantics* of the
  mapping change. A principal carries the version it was resolved under.
* `mapping_epoch` — a monotonically increasing integer (`3`). Increments on any change that can
  invalidate a cached answer: revocation, membership change, alias merge, owner rewrite.
* **Cache rule:** the hardened recall cache key includes `tenant + subject + group + session +
  policy_version + epoch`, and each entry stores per-resource versions. On every hit it checks
  `isFresh(entry, {epoch, versions})`; a mismatch forces revalidation (proven:
  `cache.stale_on_epoch_change`, `cache.stale_on_resource_version_change`).
* A permission answer computed under epoch *N* MUST NOT be reused under epoch *N+1*.

---

## 6. Revocation

* Three independent revocation surfaces: **identity**, **membership**, **resource**. Each bumps the
  epoch (or the resource version) so that caches cannot outlive the change.
* The feed is `getRevocations(tenant)` → `{mapping_version, epoch, identities:[…], groups:[…],
  resources:[…]}`. It is a full snapshot per epoch, not a delta, so a consumer that misses a
  message still converges.
* **Semantics are partially open (OI-04-05).** The design package does not say whether a revoked
  identity loses (a) authorization only, or (b) availability of its own memories. This baseline
  implements (a) and never serves a revoked identity's `owner_private` to anyone. That choice is
  conservative and reversible; it is flagged as a design question, not a settled rule.
* Cached results MUST NOT survive revocation. The weak profile does survive it ⇒ **LEAKED**
  (`cache.weak.stale_after_revocation`, `EV-IDENT-0132`).

---

## 7. Audit trail

Eight mandatory fields (04 §7.10), all verified present on every entry
(`audit.eight_fields_complete` `EV-IDENT-0211`, `fc.audit.eight_fields_complete` `EV-IDENT-0114`):

`actor`, `action`, `target`, `result`, `reason_code`, `policy_version`, `request_id`, `timestamp`

Rules:

1. `actor` is an **internal** id. Host ids never appear (`audit.no_host_identifier`).
2. Memory body text never appears; the canary check asserts 0 occurrences (`audit.no_content`).
3. `target` is `<resource_type>:<resource_id>`.
4. One entry per decision — the count equals `gate.stats.total`.
5. Denials carry the specific reason code, not `UNKNOWN`.

---

## 8. Policy profile HP-1 (implemented by `mock-permission-provider.cjs`)

A single, explicit, tabulatable policy so that every matrix cell has a documented expectation.

| Scope | Required | Additional |
|---|---|---|
| `owner_private` | `subject_id == owner_subject_id` | and **no** `group_id` in context |
| `group_shared` | `group_id == resource.group_id` | and membership row exists |
| `session_only` | `session_id == resource.session_id` | and subject match |
| `project_scoped` | project match | and membership |
| `system_internal` | actor role `system_service` | never for user roles |
| `export` | `export_grant` in `authorized_by` | out of scope this round (OI-04-02) |
| any | elevated roles (`memory_owner`, `system_service`) | denied for user-facing reads |
| any | `resource.state != active` | `RESOURCE_DELETED` / `RESOURCE_REVOKED` |
| any | `resource.tenant_id != ctx.tenant_id` | `TENANT_MISMATCH` |

---

## 9. Isolation of the allow-all double

`AllowAllPermissionProvider` exists **only** so that a test can prove the rest of the stack refuses
to rely on the engine. Four independent controls, all executed (`EV-IDENT-0169`…`0189`):

1. **Call-site guard** — an isolation scope can only be opened from a file under `tests/`; a probe
   compiled with a `lib/` filename is refused with `ISOLATION_CALLER_NOT_TEST`.
2. **Construction guard** — `new AllowAllPermissionProvider()` without a live token throws
   `ISOLATION_REQUIRED`; `new PermissionGate({provider: allowAll})` without a token throws
   `PROVIDER_NOT_PERMITTED`.
3. **Static closure** — the transitive `require()` closure of the ten request-path modules does not
   contain `allow-all-provider.cjs`, nor `provider-registry.cjs`; the registry's single reference is
   a lazy require inside a function body.
4. **Config contract** — the real `security.schema.json` rejects `test_allow_all.enabled=true`
   (`const false`) and `default_policy=allow`.

And even a live allow-all engine cannot widen visibility under `m05_mode=contract_deny`: three
decisions, all `DENY / M05_NOT_IMPLEMENTED`, engine call count `0` (`EV-IDENT-0175/0176`).

**Limitation:** in-process JavaScript is not a security boundary. A determined caller can `require`
the module directly. Control #3 is a *reachability* proof for the normal code path, not a sandbox.
The real control must be deployment separation (the double must not ship). Stated again in §12.

---

## 10. Future host adapter interface

The contract the QQ Bridge adapter must satisfy before any of this can be validated for real.
All methods async, all failures **thrown or returned as a reason code — never as a default value**.

```ts
interface HostIdentityAdapter {
  // O1→O2→O7.  MUST return a principal or throw a coded refusal.  MUST NOT default.
  getPrincipal(input: {
    tenant_id: string; host_user_id: string;
    group_id?: string | null; session_id?: string | null;
  }): Promise<Principal | Refusal>;            // refuses: IDENTITY_MISSING | UNKNOWN | REVOKED | CONFLICT | MAPPING_INTEGRITY_FAILURE

  // O4→O11.  Membership is a fact about the host, not an inference from the request.
  getGroupMembership(input: {
    tenant_id: string; internal_group_id: string; internal_user_id: string;
  }): Promise<{ member: boolean; role: 'member'|'admin'|'owner'; since: string } | Refusal>;

  // O3.  MUST be minted by the host.  MUST NOT be derived from a conversation key.
  getSessionAttribution(input: {
    tenant_id: string; session_id: string;
  }): Promise<{ session_id: string; internal_user_id: string; group_id: string | null; started_at: string; active: boolean } | Refusal>;

  // §6.  Full snapshot per epoch; idempotent; monotonic epoch.
  getRevocations(tenant_id: string): Promise<{
    mapping_version: string; epoch: number;
    identities: string[]; groups: string[]; resources: string[];
  }>;

  // §5.  Must change whenever a cached permission answer could become wrong.
  getMappingVersion(tenant_id: string): Promise<{ mapping_version: string; mapping_epoch: number }>;
}
```

Adapter obligations, in priority order:

1. **Never default an identity.** No `unknown` user, no `guest`, no fall-through to a service
   principal. `atk.F4` shows what that costs.
2. **Never let a conversation key become a session id.** 08 §7.3.2, and O6 above.
3. **Epoch discipline.** Any change that could invalidate an answer bumps the epoch *before* the
   change is visible to readers.
4. **Deterministic derivation.** Same host id + same version ⇒ same internal id, in every process,
   forever. A derivation that depends on time or randomness makes replay impossible.
5. **No host ids in the audit trail.**

`SyntheticIdentityProvider` in `lib/synthetic-identity-provider.cjs` implements exactly this surface
against fake data, which is why the same tests can be re-pointed at a real adapter later without
rewriting the assertions.

---

## 11. Design ambiguities this contract had to resolve

Recorded because a future reader will hit them again; each is a place where the design package is
silent or self-contradictory.

| id | Ambiguity | Resolution taken | Risk if wrong |
|---|---|---|---|
| **DG-1** | 04 §7.4.2 and 08 §7.4.2 disagree on whether `session_only` for a Group Member requires a session match. | Require the match (stricter). | Under-served members; safe direction. |
| **DG-2** | `permission-context.schema.json` has no field for the *requester's* subject; `subject_id` is documented as "记忆所属主体". | The gate reads `ctx.subject_id` as the actor's subject and documents the schema gap. | An implementer following the schema literally would compare the actor to itself and allow everything. **Highest-risk ambiguity found.** |
| **DG-3** | `SCOPE_MISSING` / `SCOPE_UNKNOWN` / `UNTRUSTED_CONTEXT_SOURCE` guards are unreachable because schema validation rejects those inputs first. | Keep both; assert the guard exists, note that the schema shadows it. | Dead code mistaken for a control. |
| **DG-4** | Evidence "inherits the strictest permission" has no machine-readable way to compare scopes. | Compare by an explicit strictness order and require a second gate call. | Divergent implementations of "strictest". |
| **DG-5** | Nothing says whether the recall cache may be invalidated asynchronously. | Epoch+version check on every hit; no async invalidation. | Stale reads after revocation. |
| **DG-6** | Group-session attribution (OI-04-03) and revoke semantics (OI-04-05) are open in the design package. | Conservative reading, flagged as open. | Legitimate access denied until closed. |

---

## 12. Preconditions for real-host validation

**Real-host validation is NOT permitted in this session**, and none of the statements in this
document may be read as one. Before any real mapping is exercised, all of the following must hold:

1. **Authorization.** The host data owner authorizes the validation in writing, naming the tenant,
   the time window and the reviewer. (Not present in this session.)
2. **A real adapter.** An implementation of §10 against QQ Bridge, with the five methods and the
   five obligations, reviewed by someone other than its author. (Does not exist.)
3. **A redacted identity corpus.** A frozen, redacted extract of host ids/sessions/groups so the
   mapping can be replayed offline and compared, instead of trusting a live call.
4. **A live-tenant sandbox** for the write paths (revocation, membership change, message delete)
   that must not touch production.
5. **An independent reviewer** for the DSH identity side: confirm that DSH never receives host ids,
   and that the bridge's namespace boundary (`qq:`) is enforced on the receiving end, not only on
   the sending end.
6. **A migration/rollback story** for `mapping_version` and `mapping_epoch`, including what happens
   to memories owned by subjects that change.
7. **A decision on DG-2** — the schema gap where the requester's subject has no field — before any
   real engine is pointed at the schema.

Until 1–7 exist, the correct status of every real-host claim is `NOT_VERIFIED`, and the boundary
slogan stands:

**`SYNTHETIC_IDENTITY_VERIFIED != REAL_HOST_MAPPING_VERIFIED`**
