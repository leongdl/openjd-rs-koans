# 09 — Expression errors

## What this is

The thirteen structured `ExpressionErrorKind` variants and the
`ExpressionError` wrapper that attaches optional source-location
context. After this doc you'll know which variant fires for which
class of failure, and why caller code should match on the kind
instead of parsing the `Display` output.

## Read these in order

- `crates/openjd-expr/src/error.rs:1-80` — module docs,
  `ExpressionErrorKind`.
- `crates/openjd-expr/src/error.rs` (rest) — `ExpressionError`,
  `with_node`/`with_span`, `From` impls.
- `crates/openjd-expr/src/eval/parse.rs` — sites that raise
  `ParseError`, `UnsupportedSyntax`, `ExpressionTooDeep`.
- `crates/openjd-expr/src/eval/evaluator.rs` — sites that raise
  `TypeError`, `IntegerOverflow`, `DivisionByZero`,
  `IndexOutOfBounds`, `MemoryLimitExceeded`, `OperationLimitExceeded`,
  `ExplicitFail`, `UndefinedVariable`, `UnknownFunction`.

## The thirteen kinds

```rust
// crates/openjd-expr/src/error.rs:13
#[derive(Debug, Clone, thiserror::Error)]
#[non_exhaustive]
pub enum ExpressionErrorKind {
    UndefinedVariable { name: String, suggestion: String },
    UnknownFunction { name: String },
    TypeError { message: String },
    IntegerOverflow,
    DivisionByZero { op: &'static str },
    FloatError { message: String },
    IndexOutOfBounds { message: String },
    MemoryLimitExceeded { used: usize, limit: usize },
    OperationLimitExceeded { count: usize, limit: usize },
    ExpressionTooDeep { depth: usize, limit: usize },
    UnsupportedSyntax { feature: String },
    ExplicitFail(String),
    ParseError(String),
    Other(String),
}
```

| Kind | When | Caller intent on match |
|---|---|---|
| `UndefinedVariable` | Reference like `Param.NoSuch` | Suggest similar names from the symbol table (the `suggestion` field is precomputed via edit distance — see `edit_distance.rs`). |
| `UnknownFunction` | `frobnicate(1, 2)` — no such function | Same: suggest from the function library. |
| `TypeError` | Argument types don't fit any signature | Surface the message; it includes the offending types. |
| `IntegerOverflow` | i64 arithmetic overflowed | Don't retry; the values are too large. RFC 0005 §3 prohibits silent wrapping. |
| `DivisionByZero` | `/`, `//`, or `%` with zero divisor | Same. `op` discriminates the message. |
| `FloatError` | NaN or Inf produced (RFC 0005 §2) | Surface the offending operation. |
| `IndexOutOfBounds` | `list[5]` on a 3-element list | Includes the message describing index + length. |
| `MemoryLimitExceeded` | Heap usage crossed the limit | Caller may retry with a higher limit if they trust the source. |
| `OperationLimitExceeded` | Op count crossed the limit | Same. |
| `ExpressionTooDeep` | AST depth crossed `MAX_EXPRESSION_DEPTH` | Reject — the expression is pathological. |
| `UnsupportedSyntax` | Lambda, walrus, f-string, etc. | Surface the feature name. |
| `ExplicitFail` | The `fail("msg")` function called | The user message. Don't add framing. |
| `ParseError` | ruff syntax error not at a contextual-keyword position | Surface ruff's message. |
| `Other` | Catch-all for failures that don't fit a structured variant | Try not to add new producers; prefer adding a new variant. |

`#[non_exhaustive]` means callers must match a `_ =>` arm.

## ExpressionError — the wrapper

```rust
pub struct ExpressionError {
    kind: ExpressionErrorKind,
    node: Option<NodeContext>,           // AST node info for caret rendering
    span: Option<SourceSpan>,            // byte range in source
}
```

The wrapper attaches optional source-location context to a kind.
Two builder methods:

```rust
err.with_node(ast_node)   // Records node info for "expected: X, got: Y" carets
err.with_span(span)       // Records (start, end) byte offsets in source
```

The CLI's error formatter uses both to produce caret-annotated
messages like:

```
Method 'upper' not found
  Param.Count.upper()
  ~~~~~~~~~~~~^~~~~~~
```

A producer site that has the AST node available should always
attach it. A `?` propagation that doesn't have one is fine;
later layers will attach what they can.

## Edit-distance suggestions

```rust
// crates/openjd-expr/src/edit_distance.rs (paraphrased)
pub fn closest_match<'a>(target: &str, candidates: &'a [&str], threshold: usize)
    -> Option<&'a str>;
```

When `UndefinedVariable` or `UnknownFunction` fires, the producer
walks the available names through Damerau-Levenshtein and includes a
`Did you mean 'Param.Frame'?` suggestion in the error message. The
suggestion field is `String::new()` if no candidate is close enough;
the formatter just doesn't emit the hint then.

## Match-on-kind, not on Display

Two reasons callers should `match err.kind()` instead of substring-
testing the `Display` output:

1. **Display strings drift.** The crate freely changes wording,
   adds suggestions, tweaks formatting. Tests that grep for "Type
   error" become flaky.
2. **Kinds are SemVer-stable.** New variants land behind
   `#[non_exhaustive]`. Callers that match on the variants they care
   about and have a `_` arm continue to compile across upgrades.

Common pattern:

```rust
match err.kind() {
    ExpressionErrorKind::UndefinedVariable { name, .. } => {
        suggest_parameter(name)
    }
    ExpressionErrorKind::OperationLimitExceeded { .. } => {
        // Maybe the caller can offer to bump the limit.
    }
    _ => {
        return Err(err.into());
    }
}
```

## The bridge to ModelError

```rust
// crates/openjd-model/src/error.rs:62
impl From<openjd_expr::SymbolTableError> for ModelError {
    fn from(e: openjd_expr::SymbolTableError) -> Self {
        ModelError::Expression(openjd_expr::ExpressionError::from(e))
    }
}

// :39
#[error("Expression error: {0}")]
Expression(#[source] openjd_expr::ExpressionError),
```

The `#[source]` attribute means `std::error::Error::source()` walks
into the wrapped expression error, preserving the chain. A caller
that wants to reach `ExpressionErrorKind` from a `ModelError` does:

```rust
if let ModelError::Expression(err) = model_err {
    match err.kind() { ... }
}
```

## Exercise

```sh
# Force every variant in a small input. For each, find the matching
# kind in error.rs.

# UndefinedVariable
echo 'specificationVersion: jobtemplate-2023-09
name: "{{ Param.NoSuch }}"
steps:
  - { name: S, script: { actions: { onRun: { command: "echo", args: [] } } } }' \
| ./target/release/openjd-rs check /dev/stdin

# UnknownFunction
echo 'specificationVersion: jobtemplate-2023-09
extensions: [EXPR]
name: "{{ frobnicate(1) }}"
steps:
  - { name: S, script: { actions: { onRun: { command: "echo", args: [] } } } }' \
| ./target/release/openjd-rs check /dev/stdin

# TypeError
echo 'specificationVersion: jobtemplate-2023-09
extensions: [EXPR]
parameterDefinitions:
  - { name: C, type: INT, default: 1 }
name: "{{ Param.C.upper() }}"
steps:
  - { name: S, script: { actions: { onRun: { command: "echo", args: [] } } } }' \
| ./target/release/openjd-rs check /dev/stdin

# UnsupportedSyntax
echo 'specificationVersion: jobtemplate-2023-09
extensions: [EXPR]
name: "{{ lambda x: x }}"
steps:
  - { name: S, script: { actions: { onRun: { command: "echo", args: [] } } } }' \
| ./target/release/openjd-rs check /dev/stdin

# ExpressionTooDeep — see exercise in doc 15.

# OperationLimitExceeded
echo 'specificationVersion: jobtemplate-2023-09
extensions: [EXPR]
name: "{{ sum(range(20000000)) }}"
steps:
  - { name: S, script: { actions: { onRun: { command: "echo", args: [] } } } }' \
| ./target/release/openjd-rs check /dev/stdin
```

## What's next

[10 — Model errors and validation](./10-model-errors-and-validation.md).
The expression errors are *one* of the variants `ModelError` wraps;
doc 10 covers the outer enum and the structured `ValidationErrors`
list.
