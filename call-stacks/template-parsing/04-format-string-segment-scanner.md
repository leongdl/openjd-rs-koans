# 04 — Format string segment scanner (`{{ ... }}`)

**Use case:** How does `"a{{X}}b{{Y}}c"` become a `Vec<Segment>` of
`[Literal("a"), Expression(X), Literal("b"), Expression(Y), Literal("c")]`?

This is a small, hand-rolled lexer — not a generated parser. Worth
understanding because it's the choke point through which **every**
template field flows.

## The whole call site

Reached from [koan 03](03-expression-block-end-to-end.md) stage 1, but
also directly callable as `FormatString::new(input)` for ad-hoc
parsing.

1. **`FormatString::new(input)`** — [openjd-rs/crates/openjd-expr/src/format_string.rs:66](../../../openjd-rs/crates/openjd-expr/src/format_string.rs#L66)
   thin wrapper that locks in `ExprProfile::latest()`. Use
   `with_profile` for stability across crate versions.

2. **`FormatString::with_profile(input, profile)`** — [format_string.rs:76](../../../openjd-rs/crates/openjd-expr/src/format_string.rs#L76)
   - rejects inputs longer than `MAX_FORMAT_STRING_LEN` (1 MB)
   - calls `parse_segments`
   - rejects results with more than `MAX_FORMAT_STRING_SEGMENTS`
     (1,000) segments
   - stores `raw + segments` so `raw()` is always recoverable

3. **`parse_segments(input, profile)`** — [format_string.rs:462](../../../openjd-rs/crates/openjd-expr/src/format_string.rs#L462)
   the scanner proper.

## What `parse_segments` does

A single while loop walking byte offsets:

```text
pos = 0
loop:
    find next "{{" from pos
    case None:                     ─ no more interpolations
        if a "}}" still appears   → "Missing opening braces" error
        else                      → push remaining literal, return
    case Some(op):                 ─ found opening braces at op
        if "}}" appears earlier   → "Braces mismatch" error
        push Literal(input[pos..op])
        find next "}}" from op+2
        case None                 → "Braces mismatch" error
        case Some(ee):
            et = input[op+2..ee].trim()
            if et.is_empty()      → "Empty expression" error
            parsed = ParsedExpression::with_profile(et, profile)?
            push Expression { start: op, end: ee+2, parsed }
            pos = ee+2
```

A few details that matter:

- **Trim happens before parse.** `{{   Param.X   }}` parses identically
  to `{{Param.X}}`. Whitespace inside the braces is harmless;
  whitespace outside (between segments) becomes part of the surrounding
  literal segment.

- **Spans are recorded in source-byte offsets** (`start` and `end` on
  `Segment::Expression`). When evaluation later fails, those offsets
  are what produces caret-annotated diagnostics pointing at the exact
  `{{...}}` in the original string.

- **The expression is parsed eagerly.** Every `{{...}}` triggers a full
  ruff parse, structural-depth check, and symbol collection at decode
  time. No deferred parsing. This is why the segment cap exists: a
  format string with thousands of interpolations would balloon decode
  time.

- **Imbalance detection is deliberately asymmetric.** `}}` before
  `{{` is *always* an error. `{{` without `}}` after it is *also* an
  error. But `}}` *after* a closed segment is just literal text — the
  scanner only looks for `}}` after seeing `{{`.

- **No nested `{{...}}`.** The first `}}` after `{{` ends the segment.
  An expression that genuinely contains `}}` (rare — Python
  expressions can't naturally produce them) is unrepresentable by
  design. RFC-level decision; not enforced separately.

## What gets stored

```rust
enum Segment {
    Literal(String),
    Expression {
        start: usize,           // byte offset of "{{"
        end: usize,             // byte offset *past* "}}"
        parsed: ParsedExpression,
    },
}
```

The `[#allow(clippy::large_enum_variant)]` on `Segment` is a deliberate
choice ([format_string.rs:14](../../../openjd-rs/crates/openjd-expr/src/format_string.rs#L14)):
boxing `ParsedExpression` would add a pointer indirection on every
evaluation just to shrink the always-heap-allocated `Vec<Segment>`'s
per-element size. Wrong tradeoff.

## How error messages reach the user

Errors carry a `with_span(input, start, end)` so the rendered diagnostic
shows the offending `{{...}}` underlined in the original string —
*not* relative to the inner expression text. This is why decode errors
look like:

```
Failed to parse interpolation expression at [4, 18]. Reason: Empty expression.
   1 | echo {{   }} world
       |      ^^^^^^^
```

and not just "Empty expression at column 8 of `   `."

## Edge case: `serde::Deserialize`

`FormatString` implements its own `Visitor` ([format_string.rs:346](../../../openjd-rs/crates/openjd-expr/src/format_string.rs#L346))
that accepts not just strings but also `i64`, `u64`, `f64`, and `bool`
(stringified). This is so YAML scalars like `timeout: 300` (an int) or
`runnable: true` (a bool) can deserialize directly into a
`FormatString` field without forcing every template author to quote
plain numbers.

## Related

- [koan 03](03-expression-block-end-to-end.md) — the full pipeline this
  scanner feeds into.
- [koan 05](05-python-keyword-after-dot-rewrite.md) — what happens
  after `parse_segments` hands the inner string to `ParsedExpression`.
