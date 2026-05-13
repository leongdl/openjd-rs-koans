# 19 — Cross-language embedding

## What this is

How the Rust crates show up in non-Rust callers. The only first-party
binding today is `openjd-for-js`, a WebAssembly target that exposes
the model and expression layers to ECMAScript (browser, VS Code
extensions, Node.js). Sessions deliberately omitted. There is no
Python binding — the Python implementation is a separate parallel
codebase, not an FFI shim.

## Read these in order

- `crates/openjd-for-js/Cargo.toml` — `crate-type = ["cdylib",
  "rlib"]`, dependency choices.
- `crates/openjd-for-js/src/lib.rs` — module structure.
- `crates/openjd-for-js/src/model.rs` — template parse / validate /
  job-create wrappers.
- `crates/openjd-for-js/src/expr.rs` — expression evaluation
  wrappers.
- `crates/openjd-for-js/src/errors.rs` — JS-shaped error types.

## What's exposed

```rust
// crates/openjd-for-js/src/lib.rs
pub mod errors;
pub mod expr;
pub mod model;
```

Three groups of `wasm_bindgen` exports, roughly:

| Group | What it does |
|---|---|
| `model::*` | Parse a template string, validate, derive a profile, instantiate a job, walk parameter space. |
| `expr::*` | Parse an expression, evaluate against a JS-supplied symbol table, return the result as a JS value. |
| `errors::*` | Structured error types that round-trip through `serde-wasm-bindgen`. |

Sessions does *not* appear. Reasons:

- Subprocess + IPC + signal handling don't translate to WASM.
- Cross-user is meaningless in a browser sandbox.
- The async runtime (tokio) doesn't compile to WASM without
  `wasm-unknown-unknown`-specific feature flags that complicate the
  rest of the workspace.

## Why WASM, not NAPI

The Rust target is WebAssembly via `wasm-bindgen`. There is no
NAPI/Node.js native add-on. The reasoning:

- The primary consumers are browsers and VS Code (which loads the
  WASM via its built-in support).
- WASM avoids per-platform native binaries — one `.wasm` ships
  everywhere.
- Performance is sufficient: validation runs in milliseconds even
  for large templates.

A NAPI binding could be added later for Node.js servers that want
sub-millisecond startup, but isn't currently on the roadmap.

## Error shape across the boundary

Rust's `ModelError` and `ExpressionError` are unsuitable for direct
JS consumption — they carry trait objects (`#[source]`),
non-serializable spans, and non-exhaustive enums.

`crates/openjd-for-js/src/errors.rs` mirrors them as plain
serializable structs:

```rust
#[derive(serde::Serialize)]
pub struct JsExpressionError {
    pub kind: String,           // "UndefinedVariable", "TypeError", ...
    pub message: String,
    pub span: Option<JsSpan>,
}
```

Conversion happens at the boundary. The `kind` field maps to
`ExpressionErrorKind` variant names so JS callers can branch on
strings.

## Why no Python binding

A reasonable expectation, given how widely Python is used in this
ecosystem. The Python *implementation* is a parallel codebase
(`openjd-model-for-python`, `openjd-sessions-for-python`,
`openjd-cli` Python). They are not shims — they are independent
implementations of the same spec, sharing the conformance test
suite.

This decision was deliberate:

- Python users expect Pythonic types (`dict`, `dataclass`,
  exceptions, `pathlib.Path`). A Rust-shim would either re-implement
  those at the boundary (expensive) or expose Rust idioms through
  Python (unfamiliar).
- The Python library predates the Rust one. Replacing it with an
  FFI shim would risk breakage for every existing caller.
- Conformance testing keeps the two implementations aligned without
  forcing a shared codebase.

If a Python embedder genuinely needs Rust performance, the path is
to call the Rust CLI as a subprocess (e.g. `openjd-rs check`) or
to write a small NAPI binding in their own crate. The workspace
doesn't preclude either.

## Trade-offs you'll feel as a contributor

| If you're working on... | Implication |
|---|---|
| `openjd-expr`, `openjd-model` | Your changes flow into JS via WASM automatically. Run `cargo check -p openjd-for-js` before pushing. |
| `openjd-sessions` | No JS impact. Don't worry about WASM compatibility. |
| Anything that adds a new `pub` type | Decide if it should also be exposed through `openjd-for-js`. If not, document why in the PR. |
| Performance-sensitive WASM-side code | `wasm-bindgen`'s data marshaling is the dominant cost; minimise round-trips. |

## Exercise

```sh
# 1. Build the WASM crate.
cd openjd-rs/crates/openjd-for-js
cargo build --target wasm32-unknown-unknown

# 2. Browse what's exported. Look for `#[wasm_bindgen]` attributes.
grep -rn 'wasm_bindgen' src/

# 3. Compare what's exposed vs what's available in openjd-model:
diff <(grep '^pub fn' ../openjd-model/src/lib.rs) \
     <(grep '#\[wasm_bindgen\]' src/model.rs | head -20)

# 4. Read the errors module. Notice how `ExpressionErrorKind` —
#    a #[non_exhaustive] enum that doesn't serialize cleanly through
#    wasm-bindgen — is flattened into a `kind: String` discriminator.
$EDITOR src/errors.rs
```

## What's next

This is the last doc in the curriculum. If you've read everything
from 00 through 19, you've covered:

- 00: Build, test, idioms.
- 01-05: Core data structures (`ExprValue`, `SymbolTable`,
  `ExprProfile`, `FormatString`, `JobTemplate`).
- 06-08: Parse pipeline (deserialization, expression parser,
  evaluator).
- 09-10: Errors (expression and model layers).
- 11-12: Limits and public API.
- 13: Concurrency.
- 14-16: Deep dives (parsing call stack, expression pipeline,
  RFC 0008 wrap dispatch).
- 17-19: Python comparison and JS embedding.

Pick a real task next:

- **Improve a koan.** If you found something confusing, fix the doc
  for the next reader.
- **Pick up an issue.** The `good-first-issue` label in
  `openjd-rs/issues` is a calibrated entry point.
- **Land an RFC.** The wrap-actions implementation in this codebase
  predates the RFC's acceptance — there's plenty of room to drive
  the next one.
