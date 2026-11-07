# FTS addendum — parent verification of the `loadExtension` premise correction

Date: 2026-11-05
Related evidence: `EV-FTS-*` (`fts.extension_path`), `03-verification/fts/TASK-RESULT.md §2.1`

---

## 1. The correction the agent offered

Agent C's report says:

> "The brief states that no tokenizer extension can be installed. That is true **in effect**, but the
> reason is narrower than 'extension loading is impossible': `node:sqlite` **does** expose the
> extension-loading API. What is missing is a compiled win32-x64 tokenizer library and any means to
> obtain or build one here."

and concludes: *"unblocking it is a packaging task (ship a prebuilt `.dll`), not an architectural one."*

## 2. What the parent measured

```text
enableLoadExtension: function
loadExtension:        function
enableLoadExtension(true)      -> ERR: Cannot enable extension loading because it was disabled at database creation.
loadExtension("nonexistent")   -> ERR_INVALID_STATE | extension loading is not allowed
```

Attempting to create the database with each plausible option name:

| Constructor options tried | Result |
|---|---|
| `{enableExtensionLoading: true}` | still disabled at creation |
| `{allowExtensionLoading: true}` | still disabled at creation |
| `{loadExtension: true}` | still disabled at creation |
| `{}` | still disabled at creation |

`DatabaseSync` in this build takes **no** second-argument option that enables it. `node --help` shows
only `--no-experimental-sqlite`, i.e. a switch to *disable* the module, not to enable extension loading.

## 3. Verdict

The agent's correction is **directionally right and factually incomplete**.

| Statement | Verdict |
|---|---|
| "`node:sqlite` exposes the extension-loading API surface" | **Correct** — the methods exist. |
| "Only a prebuilt tokenizer DLL is missing" | **Incorrect** — the capability is disabled at database creation in this Node build, and the constructor offers no option to enable it. |
| "Unblocking it is a packaging task, not an architectural one" | **Not established.** Shipping a DLL would not be sufficient here. |

So the original brief's conclusion — *a third-party tokenizer is not reachable today* — **stands**,
but for a stronger reason than assumed: it is not merely an absent artefact, it is a **compile-time
capability that this runtime does not provide**.

## 4. Consequence for the design

Option F (third-party tokenizer) is not "a packaging spike away". It requires either:
- a Node build whose bundled SQLite has extension loading enabled, **and** a matching prebuilt
  tokenizer binary for that ABI, **or**
- a different SQLite binding altogether (which reintroduces the native-dependency problem the
  environment already cannot solve — no package manager, no verified C toolchain).

This makes the dependency-free recommendation (option D: `unicode61` + unigram/bigram shadow)
**stronger**, not weaker: it is now the only option that is reachable without changing the runtime.

## 5. What was not verified

- Whether a Node build compiled with `SQLITE_ENABLE_LOAD_EXTENSION` would accept the call — not
  tested, and not testable without such a build.
- Whether `node:sqlite`'s `enableLoadExtension` works in other Node versions — only v24.21.0 probed.
- Whether the DLL, if loadable, could register an FTS5 tokenizer (that is a documented SQLite
  capability but unverified on this toolchain).

These are recorded as open rather than answered by assumption.
