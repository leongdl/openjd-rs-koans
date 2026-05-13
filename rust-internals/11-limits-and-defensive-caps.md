# 11 — Limits and defensive caps

## What this is

Every defensive constant in `openjd-rs`, in one place. New
contributors are otherwise blindsided by these in tests ("why does
my 1.5 MB template field error?"). Grouped by what they protect
against, with the actual numeric value, the rationale, and where
they're enforced.

## Read these in order

- `crates/openjd-model/src/template/parse.rs:30-40` —
  `MAX_DOCUMENT_DEPTH`.
- `crates/openjd-model/src/types.rs` — `CallerLimits` knobs.
- `crates/openjd-expr/src/eval/parse.rs` — `MAX_EXPRESSION_DEPTH`,
  `MAX_PARSE_INPUT_LEN`.
- `crates/openjd-expr/src/format_string.rs:42-52` —
  `MAX_FORMAT_STRING_LEN`, `MAX_FORMAT_STRING_SEGMENTS`.
- `crates/openjd-expr/src/symbol_table.rs:38-56` —
  `MAX_SYMBOL_TABLE_ENTRIES`.
- `crates/openjd-expr/src/range_expr.rs:13-32` —
  `MAX_RANGE_EXPR_CHUNKS`.
- `crates/openjd-expr/src/eval/evaluator.rs` —
  `DEFAULT_OPERATION_LIMIT`, `DEFAULT_MEMORY_LIMIT`.

## Three categories of limit

| Category | What it protects | Examples |
|---|---|---|
| **Transport** | Untrusted JSON/YAML — applied at deserialization | `MAX_DOCUMENT_DEPTH`, `MAX_SYMBOL_TABLE_ENTRIES`, `CallerLimits::max_template_size` |
| **Parser / structural** | Pathological expression source — applied at parse time | `MAX_EXPRESSION_DEPTH`, `MAX_PARSE_INPUT_LEN`, `MAX_FORMAT_STRING_LEN`, `MAX_FORMAT_STRING_SEGMENTS`, `MAX_RANGE_EXPR_CHUNKS` |
| **Evaluation** | Resource consumption during evaluation — applied per `EvalContext` | `DEFAULT_OPERATION_LIMIT`, `DEFAULT_MEMORY_LIMIT` |

The categories are different because they fire at different times and
have different override stories. Transport caps are hard ceilings.
Parser caps are also hard ceilings. Evaluation budgets are *defaults*
that callers can raise or lower per evaluation.

## The full table

| Constant | Value | Where enforced | Why |
|---|---:|---|---|
| `MAX_DOCUMENT_DEPTH` | 128 | `serde-saphyr` budget on YAML; `serde_json`'s built-in limit on JSON | Stack-exhaustion guard on deeply nested input |
| `MAX_FORMAT_STRING_LEN` | 1 048 576 (1 MB) | `FormatString::with_profile` | A 1 MB template field is almost certainly an attack |
| `MAX_FORMAT_STRING_SEGMENTS` | 1 000 | `FormatString::with_profile` | Each segment is a full ruff parse + heap allocation |
| `MAX_SYMBOL_TABLE_ENTRIES` | 100 000 | `SymbolTable::Deserialize` | Untrusted JSON could otherwise inflate the table |
| `MAX_PARSE_INPUT_LEN` | 65 536 (64 KB) | `ParsedExpression::with_profile` | An expression source longer than 64 KB is suspicious |
| `MAX_EXPRESSION_DEPTH` | 100 | parser structural validator + evaluator | Stack-exhaustion guard on `((((1))))`-style nesting |
| `MAX_RANGE_EXPR_CHUNKS` | 10 000 | `RangeExpr::parse` | Bounds the heap-side of `Vec<IntRange>` for crafted inputs |
| `DEFAULT_OPERATION_LIMIT` | 10 000 000 | `EvalContext::charge_op` | Stops combinatorially explosive expressions |
| `DEFAULT_MEMORY_LIMIT` | 100 000 000 (100 MB) | `EvalContext::check_memory` | Bounds string-multiply / list-build memory |

`CallerLimits` (in `openjd-model/src/types.rs`) lets a service set
*tighter* caps than the built-in defaults:

```rust
pub struct CallerLimits {
    pub max_template_size: Option<usize>,
    pub max_format_string_length: Option<usize>,
    pub max_format_string_segments: Option<usize>,
    // ...
}
```

The built-in caps are hard ceilings; `CallerLimits` is how a service
tightens them further (e.g. set `max_template_size: 64 * 1024` to
reject anything larger than 64 KB).

## What each cap is *not*

| Cap | Doesn't protect against |
|---|---|
| `MAX_RANGE_EXPR_CHUNKS` | The *logical* size of a single chunk. `"1-100000000000"` is one chunk; the cap doesn't fire. (Materialization to a list is governed by the evaluator's memory limit.) |
| `MAX_SYMBOL_TABLE_ENTRIES` | In-process `SymbolTable::set` calls — only the JSON deserializer counts. |
| `DEFAULT_OPERATION_LIMIT` | Wall-clock time. A single dispatch call counts as one op regardless of how long it takes. The op limit is correlated with time but not equal to it. |
| `DEFAULT_MEMORY_LIMIT` | RSS. Tracks the high-water *live value* size in the evaluator, not the process's actual heap. |

## How to override

For tests:

```rust
let result = EvalBuilder::new()
    .with_operation_limit(1_000_000_000)
    .with_memory_limit(1024 * 1024 * 1024)   // 1 GB
    .eval(&parsed, &symtabs)?;
```

For services that want tighter caps:

```rust
let ctx = ValidationContext::from_profile(profile)
    .with_caller_limits(CallerLimits {
        max_template_size: Some(64 * 1024),
        max_format_string_length: Some(8 * 1024),
        ..Default::default()
    });
```

The general rule: `CallerLimits` only goes *down* from the built-in
defaults. Setting `max_template_size: Some(usize::MAX)` is allowed
but the built-in caps still hold above it.

## Errors produced

| Cap exceeded | Error variant |
|---|---|
| `MAX_DOCUMENT_DEPTH` | `ModelError::DecodeValidation(...)` |
| `CallerLimits::max_template_size` | `ModelError::ModelValidation(...)` |
| `MAX_FORMAT_STRING_LEN` / `MAX_FORMAT_STRING_SEGMENTS` | `ExpressionError::Other(...)` |
| `MAX_PARSE_INPUT_LEN` | `ExpressionError::Other(...)` |
| `MAX_EXPRESSION_DEPTH` | `ExpressionErrorKind::ExpressionTooDeep { depth, limit }` |
| `MAX_RANGE_EXPR_CHUNKS` | `RangeExprError` (wrapped to `ExpressionError`) |
| `MAX_SYMBOL_TABLE_ENTRIES` | `serde::de::Error` (during deserialization) |
| `DEFAULT_OPERATION_LIMIT` | `ExpressionErrorKind::OperationLimitExceeded { count, limit }` |
| `DEFAULT_MEMORY_LIMIT` | `ExpressionErrorKind::MemoryLimitExceeded { used, limit }` |

## Exercise

```sh
# 1. Run the tests that exercise each cap.
cargo test -p openjd-expr -- limit
cargo test -p openjd-expr -- depth
cargo test -p openjd-expr -- max_

# 2. Find each constant's definition site:
grep -rn "MAX_DOCUMENT_DEPTH\b" crates/
grep -rn "MAX_FORMAT_STRING_LEN\b" crates/
grep -rn "MAX_EXPRESSION_DEPTH\b" crates/
grep -rn "MAX_RANGE_EXPR_CHUNKS\b" crates/
grep -rn "MAX_SYMBOL_TABLE_ENTRIES\b" crates/
grep -rn "DEFAULT_OPERATION_LIMIT\b" crates/
grep -rn "DEFAULT_MEMORY_LIMIT\b" crates/

# 3. Add a CallerLimits override and check it in a unit test.
```

## What's next

[12 — Public API surface](./12-public-api-surface.md). You now know
what's enforced; doc 12 shows what's exported as the stable public
contract.
