# 18 — Python vs Rust feature parity

## What this is

A scorecard of which spec features each implementation supports
today, plus the architectural features (snapshots, JS bindings,
CLI) that are exclusive to one side. This complements doc 17 (which
is the *file*-level mapping); this doc is the *feature*-level view.

## Read these in order

- `openjd-rs/README.md` — conformance summary table.
- `openjd-rs/CHANGELOG.md` — recent feature additions.
- `openjd-sessions-for-python/CHANGELOG.md` — same.
- The conformance-tests directory in `openjd-specifications`.

## Spec extension parity

| Spec extension | Python | Rust | Notes |
|---|---|---|---|
| `EXPR` (RFC 0005, 0006) | ✓ | ✓ | 202 + 123 conformance tests pass on Rust. Same on Python. |
| `FEATURE_BUNDLE_1` | ✓ | ✓ | Sugar for `bash`/`python`/`cmd`/`powershell`/`node` script blocks. |
| `TASK_CHUNKING` (RFC 0001) | ✓ | ✓ | Frame-range chunking via `CHUNK[INT]` task params. |
| `REDACTED_ENV_VARS` | ✓ | ✓ | Pattern-driven env-var redaction in logs. |
| `WRAP_ACTIONS` (RFC 0008) | not yet | implemented (RFC pending) | Rust ships ahead of the RFC; Python will follow when accepted. |

The conformance scorecard from `openjd-rs/README.md`:

```
Template validation (base):              444 passed
Template validation (EXPR):              202 passed
Template validation (FEATURE_BUNDLE_1):   36 passed
Template validation (TASK_CHUNKING):      11 passed
Environment template validation:          31 passed
Job execution (base):                    163 passed
Job execution (EXPR):                    123 passed
Job execution (FEATURE_BUNDLE_1):         13 passed
Job execution (TASK_CHUNKING):             7 passed
Job execution (REDACTED_ENV_VARS):         8 passed
Total:                                  1,038
```

Rust passes 100% of the conformance suite on Linux; macOS and Windows
have small platform-conditional gaps (mostly around POSIX-only file
mode bits).

## Architectural features

| Feature | Python | Rust | Notes |
|---|---|---|---|
| Job-attachments snapshots | Lives in `deadline-cloud-for-python`, not in `openjd-sessions-for-python` itself | First-party `openjd-snapshots` crate | The Rust crate is content-addressed, uses xxHash3, supports S3 upload/download natively. |
| CLI binary | `openjd-cli` (Python entry point) | `openjd-rs` (single static binary) | Both expose `check` and `run`. The Rust binary is faster to start (~10ms vs ~200ms cold). |
| Cross-user subprocess | In-process `sudo` invocation (POSIX) and DACL helper (Windows) | Spawns a long-lived helper *binary* per session over a pipe | The Rust split isolates privileged operations into an auditable binary. |
| JS / web bindings | none | `openjd-for-js` (model + expr only; sessions deliberately omitted) | Python is itself an embedding target for many callers; Rust supports a non-Rust target as well. |
| Async runtime | `asyncio` + `threading` | `tokio` (multi-threaded scheduler) | See doc 13 for concurrency model. |
| Defensive caps | Mostly implicit in code | Explicit `MAX_*` / `DEFAULT_*` constants at every defense surface (see doc 11) | Rust's caps are testable and overridable via `CallerLimits`. |
| Static type-checking of expressions | Implemented but less complete | Implemented with `unresolved[T]` propagation (see doc 08) | Both are progressively-typed; Rust's `unresolved` machinery is more uniform. |

## Things that are subtly different

| Behavior | Python | Rust |
|---|---|---|
| YAML strict booleans | Lax (`yes`/`no` parsed as bool) | Strict (`yes`/`no` parsed as string) | Rust uses `serde-saphyr` with `strict_booleans: true` for spec correctness. |
| Document depth limit | None (Python's recursion fires unpredictably) | 128 (matches `serde_json` hardcoded limit) | Rust rejects pathological depth deterministically. |
| Float pass-through | Pydantic preserves through `Decimal` paths only | `Float64` carries the original `Box<str>` until arithmetic | Rust's pass-through is at the value level, not the field level. |
| Path mapping for URI paths | Python's `pathlib` doesn't handle URIs | First-class URI handling in `openjd_expr::uri_path` | Both behave correctly per RFC 0006; the Rust implementation is more testable. |
| Validation error layout | Pydantic's nested per-field messages | `ValidationErrors` with explicit `PathElement` + `DiagnosticSpan` | Rust messages include caret carets; Python messages don't. |

## Things that are the same enough that you don't need to think about it

- The set of OpenJD types: `bool`, `int`, `float`, `string`, `path`,
  `range_expr`, `list[T]`.
- The `{{...}}` interpolation syntax.
- The `let` binding scoping rules.
- The session lifecycle states (`READY`, `RUNNING`, etc.).
- The `ActionStatus` shape.
- The path-mapping rule structure.
- The `openjd_env: KEY=value` stdout protocol.
- The cross-user-helper protocol (between sessions and helper binary
  is similar; the helper binary itself is Rust-specific but the
  *contract* is shared).

## Where to follow updates

| Source | Update cadence |
|---|---|
| `openjd-rs/CHANGELOG.md` | Per release |
| `openjd-rs/COVERAGE_REPORT.md` | Per release |
| `openjd-sessions-for-python/CHANGELOG.md` | Per release |
| The OpenJD specifications repo `rfcs/` directory | When new RFCs land |

If you're maintaining a Python service that needs to lock-step with
Rust feature parity, watch the conformance-tests directory in the
specifications repo. New extensions add their own test directory;
both implementations land them in parallel.

## Exercise

```sh
# 1. Run the conformance tests against the Rust CLI.
./scripts/run_conformance.sh

# 2. Pick one extension you don't know well (e.g. TASK_CHUNKING).
#    Read the RFC, then read the Rust implementation:
$EDITOR openjd-rs/crates/openjd-model/src/template/task_parameters.rs
$EDITOR openjd-rs/crates/openjd-expr/src/range_expr.rs

# 3. Read the matching Python:
$EDITOR openjd-model-for-python/src/openjd/model/v2023_09/_task_parameters.py
$EDITOR openjd-model-for-python/src/openjd/model/_range_expr.py

# 4. List one thing the implementations agree on and one thing they
#    intentionally differ on. (Hint: parser choice; defensive caps.)
```

## What's next

[19 — Cross-language embedding](./19-cross-language-embedding.md). The
last doc in the curriculum: how Rust's model and expr crates show up
in non-Rust callers via `openjd-for-js`.
