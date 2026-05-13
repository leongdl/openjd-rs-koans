# 04 — Format string segments

## What this is

`FormatString` is the type that holds every user-facing string field
in a template — names, commands, args, descriptions. It supports the
`{{ ... }}` interpolation syntax. This doc covers how the input is
split into segments, why each interpolation is parsed once and
evaluated many times, and the two defensive caps that protect against
pathological input.

## Read these in order

- `crates/openjd-expr/src/format_string.rs:1-100` — module docs,
  `Segment` enum, `FormatString::new`, `with_profile`.
- `crates/openjd-expr/src/format_string.rs:42-52` —
  `MAX_FORMAT_STRING_LEN`, `MAX_FORMAT_STRING_SEGMENTS`.
- `crates/openjd-expr/src/format_string.rs` (rest) — `parse_segments`,
  `resolve`, `escape_format_string`.

## The shape

```rust
// crates/openjd-expr/src/format_string.rs:19
enum Segment {
    Literal(String),
    Expression {
        start: usize,
        end: usize,
        parsed: ParsedExpression,
    },
}

pub struct FormatString {
    raw: String,
    segments: Vec<Segment>,
}
```

A format string is a list of alternating literal pieces and parsed
expressions. The `start`/`end` byte offsets mark where the
interpolation appeared in `raw`, used by error messages so a caret
in a downstream error points at the right place in the original
source.

## Why parse once

```rust
pub fn with_profile(input: &str, profile: &ExprProfile) -> Result<Self, ExpressionError> {
    if input.len() > MAX_FORMAT_STRING_LEN { /* reject */ }
    let segments = parse_segments(input, profile)?;
    if segments.len() > MAX_FORMAT_STRING_SEGMENTS { /* reject */ }
    Ok(Self { raw: input.to_string(), segments })
}
```

Construction calls `ruff_python_parser` for *every* `{{ ... }}`
interpolation, producing a `ParsedExpression` per segment. The
parsed forms are cached in the `Vec<Segment>` and reused for every
evaluation — so a 10,000-task job that uses
`{{Param.OutputDir}}/{{Task.Param.Frame}}.exr` pays parsing cost
twice (once for each interpolation), not 20,000 times.

The runtime path is just:

```rust
pub fn resolve(&self, symtabs: &[&SymbolTable]) -> Result<String, ExpressionError> {
    let mut out = String::new();
    for segment in &self.segments {
        match segment {
            Segment::Literal(s) => out.push_str(s),
            Segment::Expression { parsed, .. } => {
                let value = evaluator.eval(parsed, symtabs)?;
                out.push_str(&value.to_string_for_format()?);
            }
        }
    }
    Ok(out)
}
```

No parsing in the hot path. That's the design payoff of segment
caching.

## The two defensive caps

```rust
// :42
pub const MAX_FORMAT_STRING_LEN: usize = 1024 * 1024;     // 1 MB

// :51
pub const MAX_FORMAT_STRING_SEGMENTS: usize = 1_000;
```

`MAX_FORMAT_STRING_LEN` caps the raw source. 1 MB is several orders
of magnitude above any legitimate field — names are short, embedded
files have their own size constraints, command args are typically
under a kilobyte. An attempt to ship a megabyte template field
through this type is almost certainly an attack or a bug.

`MAX_FORMAT_STRING_SEGMENTS` caps the number of `{{ ... }}`
interpolations. Each segment costs a full ruff parse and a heap
allocation. A field with thousands of segments is suspicious.

The caps are enforced at construction time, *before* the segments
table is allocated, so a malicious input can't even allocate before
the rejection.

## Why `Vec<Segment>` and not a packed structure

Each `Segment::Expression` variant is large (it carries a
`ParsedExpression` with its own AST and HashSets). A more compact
representation would be possible, but:

- The `Vec` is allocated once at construction and never resized.
- Iteration is sequential — cache locality matters less than
  per-segment cost.
- The `#[allow(clippy::large_enum_variant)]` attribute on
  `Segment` is intentional: boxing the expression variant would add
  a pointer indirection per evaluation for negligible memory savings.

## `escape_format_string`

```rust
// re-exported from lib.rs
pub use format_string::escape_format_string;
```

A small helper that escapes `{` and `}` so untrusted text can be
embedded into a format string without becoming an unintended
interpolation. Used by the CLI when echoing user-supplied messages
and by tests that build templates from strings.

The pattern is `{` → `{{` (and the inverse rule when parsing). This
mirrors Python f-strings and Rust's `format!` semantics.

## Exercise

```sh
# 1. Run the format-string unit tests.
cargo test -p openjd-expr format_string

# 2. Construct a string with many segments and confirm the cap fires.
#    Concretely, the cap is 1,000; build something with ~1,001 and
#    expect a clean error.

# 3. Use a single 1 MB literal field and confirm it's accepted.
#    Use a single 2 MB literal field and confirm it's rejected.
#    (This is the MAX_FORMAT_STRING_LEN cap.)

# 4. Read parse_segments. Notice how it handles literal `{` (an
#    error or an escape, depending on context) and how it captures
#    the byte offsets of each `{{ ... }}` for error reporting.
```

## What's next

[05 — Model types and template shape](./05-model-types-and-template-shape.md).
You've now seen the foundational expr types and the format-string
container that uses them; doc 05 introduces the model layer's core
types — `SpecificationRevision`, `Extensions`, `ValidationContext` —
and the shape of a `JobTemplate`.
