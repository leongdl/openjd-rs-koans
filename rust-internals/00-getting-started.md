# 00 — Getting started

## What this is

The doc to read before you `cargo build`. It covers what's in the
workspace, how to build and test it, and the seven Rust idioms that show
up so often in this codebase that not knowing them will make every
subsequent doc harder. New contributors who skip this and jump straight
into `openjd-sessions/src/session.rs` (2,300 lines, async, cross-platform)
typically lose a day to "why is this `#[non_exhaustive]`?" and "where did
this `Arc` come from?" — both of which are answered here.

## Read these in order

- `openjd-rs/README.md` — top-level orientation, crate dependency graph, conformance scorecard.
- `openjd-rs/DEVELOPMENT.md` — official build/test/lint commands.
- `openjd-rs/Cargo.toml` — workspace members and shared dependency versions.
- Each crate's `Cargo.toml` and `src/lib.rs` — the public-API entry points.

## What's in the box

| Crate | Purpose | Depends on |
|---|---|---|
| `openjd-expr` | Expression language: types, ruff parser, bounded evaluator, range expressions, path mapping | (leaf) |
| `openjd-model` | Template parsing, validation, job creation, parameter spaces, dependency graphs | `openjd-expr` |
| `openjd-sessions` | Session runtime: async action execution, environment lifecycle, cross-user subprocess management | `openjd-expr`, `openjd-model` |
| `openjd-cli` | The `openjd-rs` binary: `check` and `run` | all three |
| `openjd-snapshots` | Job attachments: content-addressed file snapshots, xxHash3, S3 upload/download | (standalone) |
| `openjd-for-js` | NAPI/wasm bindings for the model + expr crates only | `openjd-expr`, `openjd-model` |

The dependency DAG is strictly layered. `openjd-expr` knows nothing about
templates. `openjd-model` knows nothing about subprocesses or async. This
matters for testing: you can exercise the entire expression engine without
touching tokio, and the entire template parser without touching a process
spawner.

## Build, test, lint

From the workspace root (`openjd-rs/`):

```sh
# Build everything
cargo build --release

# Run all tests (5,300+ unit + integration)
cargo test --workspace

# Run a single crate
cargo test -p openjd-expr
cargo test -p openjd-model
cargo test -p openjd-sessions

# Run a specific test by name
cargo test -p openjd-model decode_job_template

# Lints we gate on (run before pushing)
cargo clippy --all-features --all-targets --workspace -- -D warnings

# Formatting check (fix with `cargo fmt --all`)
cargo fmt --all -- --check

# Build docs
cargo doc --no-deps --workspace --open

# Code coverage (current ~87%)
scripts/coverage.sh
```

### Cross-user subprocess tests

Cross-user spawning (POSIX `sudo`, Win32 `CreateProcessAsUserW`) needs a
multi-user environment, so those tests live in Docker containers:

```sh
# Local users (primary)
scripts/run_cross_user_tests.sh

# LDAP-backed users (validates NSS/PAM)
scripts/run_cross_user_tests.sh --ldap
```

You don't need these on a normal inner loop. They're for changes touching
`crates/openjd-sessions/src/cross_user_helper.rs`,
`session_user.rs`, or the helper binary in `helper/`.

### Driving the CLI by hand

The fastest way to validate a change end-to-end:

```sh
./target/release/openjd-rs check path/to/template.yaml
./target/release/openjd-rs run   path/to/template.yaml -p Key=Value
./target/release/openjd-rs run   path/to/template.yaml -p file://params.yaml
```

`check` exercises only `openjd-model` + `openjd-expr`. `run` adds
`openjd-sessions`. Most contributor-facing bugs are reproducible with
`check` alone.

## The seven Rust idioms used pervasively in this codebase

These show up in every file. Internalise them now.

### 1. `#[non_exhaustive]` on every public enum

```rust
// crates/openjd-expr/src/types.rs:19-40
#[non_exhaustive]
pub enum TypeCode {
    NullType, Bool, Int, Float, String, Path, List, RangeExpr,
    Any, Union, NoReturn, Unresolved,
    TypeVarT, TypeVarT1, TypeVarT2, TypeVarT3, Signature,
}
```

The attribute means downstream crates *cannot* exhaustively match without
a wildcard arm — the compiler forces them to handle the case where a
future revision adds a new variant. This is how the crate stays
forward-compatible across spec revisions and new EXPR extensions without
SemVer breaks.

When you see `#[non_exhaustive]`, your match must include `_ => …`.

### 2. `thiserror` enums with structured `kind`s

There is no `anyhow` outside the CLI. Every error is a typed enum with
`#[derive(thiserror::Error)]`. The two main hierarchies:

- `openjd_expr::ExpressionError` wraps `ExpressionErrorKind` with optional source location.
- `openjd_model::ModelError` is the outer enum: `DecodeValidation`, `ModelValidation`, `FormatStringError`, `Expression`, `Compatibility`, `UnsupportedSchema`.

`From` impls bridge them — an `ExpressionError` becomes a
`ModelError::Expression` automatically. Callers should match on the
structured variant, not parse `Display` output, because messages drift
across revisions.

### 3. Builder + `#[must_use]` constructors with two profile constructors

A pattern you'll see on `ParsedExpression`, `EvalBuilder`, `FormatString`,
`SessionConfig`, `ValidationContext`:

```rust
// "latest profile" — convenient, deliberately unstable across crate versions
ParsedExpression::new(expr_str)?;

// "explicit profile" — stable across crate versions, recommended for production
ParsedExpression::with_profile(expr_str, &profile)?;
```

If you need stability across crate upgrades, always use `with_profile`.
The `new` form pulls in *every* known extension and revision; an
expression that parses today may fail tomorrow when the crate adds a new
keyword.

### 4. `pub use` cross-crate re-exports

`openjd-model/src/lib.rs` re-exports `FormatString` and `SymbolTable` from
`openjd-expr`. `openjd-sessions/src/lib.rs` re-exports `path_mapping`.
This is *deliberate*, mirroring the Python package layout where
`openjd.sessions.PathMappingRule` is the canonical name even though the
type lives in the expr layer. If you `cargo doc` and find yourself
hunting for a type, check the re-exports first.

### 5. `Arc<dyn Fn>` and `Arc<Vec<T>>` for shared immutable state

The expression function library is a `HashMap<String, Vec<Arc<dyn Fn(...)>>>`
(`crates/openjd-expr/src/function_library.rs:42`). The path mapping
rules are `Arc<Vec<PathMappingRule>>`. Both are immutable after
construction; `Arc::clone` is the only way they "move." This is why
there are very few `Mutex`es in the codebase — see doc 13.

### 6. `#[cfg(unix)]` / `#[cfg(windows)]` gates, not runtime dispatch

Win32-only types like `WindowsSessionUser` only *exist* on Windows
targets. POSIX-only types like `PosixSessionUser` only exist on Unix.
There is one shared `SessionUser` trait that both implement.

```rust
#[cfg(unix)]
use crate::cross_user_helper::CrossUserHelper;
#[cfg(windows)]
use crate::cross_user_helper::CrossUserHelperWin;
```

When you write platform-specific code, follow this pattern. Don't add
runtime `if cfg!(unix)` checks — they bloat the cross-platform binary.

### 7. `serde(deny_unknown_fields)` + post-deserialization validation

Serde derives catch *structural* errors only — unknown fields, wrong
types, missing required fields. Semantic rules (name uniqueness,
extension gating, single-wrap-layer-per-session, format-string
type-checking) live in a separate validation pass under
`crates/openjd-model/src/template/validate_v2023_09/`.

The rule of thumb: if it's expressible in the type system, let serde
catch it; otherwise, it's a `validate_*` function that walks the parsed
tree and accumulates `ValidationErrors`.

## Exercise

```sh
# 1. Build the workspace.
cd openjd-rs/
cargo build --release

# 2. Run a sanity-check template through the CLI.
./target/release/openjd-rs check ../openjd-rs-koans/rfc-0008/01-conditional-task-command.yaml

# 3. Make it fail. Edit the YAML to introduce a typo
#    (e.g. `Param.Quaity` instead of `Param.Quality`) and re-run.
#    Notice the error includes both the YAML field path and the
#    expression position with a caret.

# 4. Compile-check just the expr crate (fast inner loop).
cargo check -p openjd-expr
```

If step 3 produces a clear, position-annotated error and step 4 finishes
in under a second, your environment is ready.

## What's next

[13 — Concurrency and shared state](./13-concurrency-and-shared-state.md).
The seventh idiom hinted at the answer; doc 13 spells it out and tells
you where the few real synchronization primitives live.
