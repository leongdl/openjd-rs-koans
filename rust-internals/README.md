# openjd-rs internals — koan-style learning track

A reading curriculum for engineers ramping up on the Rust
implementation of Open Job Description (`openjd-rs`). Each doc is
short, self-contained, and points at concrete file:line locations in
the source tree so you can flip between this track and the code as
you read.

## Reading order

### Track 0 — Onboarding

| # | Doc | Time | Topic |
|---|---|---|---|
| 00 | [Getting started](./00-getting-started.md) | 15 min | Build, test, and the seven Rust idioms used pervasively in this codebase |

### Track 1 — Core data structures

| # | Doc | Time | Topic |
|---|---|---|---|
| 01 | [Expression types and values](./01-expr-types-and-values.md) | 25 min | `ExprType`, `TypeCode`, `ExprValue`, `Float64`, typed list variants |
| 02 | [Symbol table](./02-symbol-table.md) | 15 min | Hierarchical name → value lookup, `MAX_SYMBOL_TABLE_ENTRIES` |
| 03 | [Profile, revision, extension, host context](./03-profile-revision-extension-host.md) | 20 min | `ExprProfile`'s three axes, function-library cache |
| 04 | [Format string segments](./04-format-string-segments.md) | 15 min | Parse-once-evaluate-many, defensive caps |
| 05 | [Model types and template shape](./05-model-types-and-template-shape.md) | 20 min | `SpecificationRevision`, `Extensions`, `JobTemplate`, `ValidationContext` |

### Track 2 — Template parsing

| # | Doc | Time | Topic |
|---|---|---|---|
| 06 | [Parse pipeline overview](./06-parse-pipeline-overview.md) | 15 min | Four phases, what each produces |
| 07 | [Expression parsing and keyword rewrite](./07-expression-parsing-and-keyword-rewrite.md) | 20 min | The `Param.if` problem, ruff parser integration |
| 08 | [Evaluator walkthrough](./08-evaluator-walkthrough.md) | 30 min | AST walk, UFCS rewrite, dispatch, budgets, unresolved propagation |

### Track 3 — Errors

| # | Doc | Time | Topic |
|---|---|---|---|
| 09 | [Expression errors](./09-expression-errors.md) | 15 min | The thirteen `ExpressionErrorKind` variants |
| 10 | [Model errors and validation](./10-model-errors-and-validation.md) | 20 min | `ModelError`, `ValidationErrors`, accumulating-not-bailing |

### Track 4 — Operational envelope

| # | Doc | Time | Topic |
|---|---|---|---|
| 11 | [Limits and defensive caps](./11-limits-and-defensive-caps.md) | 15 min | Every `MAX_*` and `DEFAULT_*` constant in one place |
| 12 | [Public API surface](./12-public-api-surface.md) | 20 min | What each crate exports, what's hidden, what's stable |
| 13 | [Concurrency and shared state](./13-concurrency-and-shared-state.md) | 20 min | No DashMap; tokio + immutable `Arc` + one `LazyLock<Mutex>` cache |

### Track 5 — Deep dives

| # | Doc | Time | Topic |
|---|---|---|---|
| 14 | [Parsing call stack](./14-parsing-call-stack.md) | 30 min | String/file → `JobTemplate` → `Job` → execution tree, every function named |
| 15 | [Expression pipeline](./15-expr-pipeline.md) | 30 min | Expression string → ruff AST → bounded evaluator → `ExprValue`, worked example |
| 16 | [onWrap dispatch and parameter injection](./16-onwrap-and-parameter-injection.md) | 25 min | RFC 0008 wrap routing, where `Task.*` / `Env.Wrapped.*` get seeded |

### Track 6 — Python comparison

| # | Doc | Time | Topic |
|---|---|---|---|
| 17 | [Python ↔ Rust mapping](./17-python-vs-rust-mapping.md) | 20 min | File-by-file table for sessions, model, expr |
| 18 | [Python vs Rust feature parity](./18-python-vs-rust-feature-parity.md) | 15 min | Spec extension scorecard + architectural differences |
| 19 | [Cross-language embedding](./19-cross-language-embedding.md) | 15 min | `openjd-for-js` (WASM); why no Python binding |

## Conventions

Each doc has the same five-section shape so you can scan them
quickly:

```
# <Topic>
## What this is             — 1 paragraph: why this matters
## Read these in order      — file:line refs in the source
## The one mental model     — the single insight
## Exercise                 — runnable cargo command or tweak
## What's next              — pointer to the next doc
```

File references use the `file:line` format relative to the
`openjd-rs/` workspace root, e.g.
`crates/openjd-expr/src/types.rs:21`.

## How to run the exercises

All exercises are `cargo` commands runnable from the workspace root
(`openjd-rs/`). Most are single-crate test invocations:

```sh
cargo test -p openjd-expr <test_name>
cargo test -p openjd-model <test_name>
cargo test -p openjd-sessions <test_name>
```

A few use the CLI directly:

```sh
./target/release/openjd-rs check path/to/template.yaml
./target/release/openjd-rs run   path/to/template.yaml -p Key=Value
```

When an exercise expects a specific error, the doc shows the
substring you should look for in the output. Don't over-fit to the
exact wording — the codebase is under active development and error
strings drift faster than the structured `kind` enums.

## Reading orders for specific tasks

If you're not reading top-to-bottom, here are pre-built paths:

| Task | Read in this order |
|---|---|
| **Just orienting** | 00 → 13 → 17 (3 docs, ~55 min) |
| **Working in `openjd-expr`** | 00 → 01 → 02 → 03 → 04 → 07 → 08 → 09 → 11 → 15 |
| **Working in `openjd-model`** | 00 → 05 → 06 → 10 → 14 |
| **Working in `openjd-sessions`** | 00 → 13 → 16 → 17 |
| **Working on RFC 0008** | 00 → 13 → 16, then `../rfc-0008/` koans |
| **Migrating from Python** | 00 → 17 → 18 → 19, then pick a track |
