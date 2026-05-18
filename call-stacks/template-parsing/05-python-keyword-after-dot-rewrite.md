# 05 — Python keyword-after-dot rewrite (`Param.if`, `Task.Param.return`, …)

**Use case:** A user writes `{{ Param.if }}` because their job parameter
is literally named `if`. Python's parser would reject `.if` as a syntax
error — but OpenJD lets users name parameters anything, including
Python reserved words. How does the expression parser cope?

This is one of the more clever pieces of code in the repo. It turns a
"Python parser said no" into "rename the keyword to something
not-a-keyword, retry, and unrename the AST nodes after the fact."

## The user's view

These all parse:

```yaml
{{ Param.if }}
{{ Task.Param.return + 1 }}
{{ len(Param.class) }}
{{ Param.True or Param.False }}
```

The Python parser would raise `SyntaxError: invalid syntax` on every
one. The OpenJD parser silently rewrites the source and retries.

## The call stack

This sits inside [koan 03](03-expression-block-end-to-end.md) stage 1 step 6.

1. **`ParsedExpression::with_profile(expr, profile)`** — [openjd-rs/crates/openjd-expr/src/eval/parse.rs:133](../../../openjd-rs/crates/openjd-expr/src/eval/parse.rs#L133)
   length-cap, fast-path vs worker-thread dispatch.

2. **`parse_inner(raw, expr_str, profile)`** — [eval/parse.rs:489](../../../openjd-rs/crates/openjd-expr/src/eval/parse.rs#L489)
   the retry loop.

3. **Body of the `loop { ... }`** at [eval/parse.rs:500](../../../openjd-rs/crates/openjd-expr/src/eval/parse.rs#L500):
   ```text
   loop:
       to_parse = source (paren-wrapped if multi-line)
       call ruff_python_parser::parse_expression(&to_parse)
       case Ok(parsed):
           run validate_structure, collect_symbols, check_comprehension_shadowing
           return ParsedExpression { ast, ..., keyword_renames }
       case Err(e):
           for kw in PYTHON_KEYWORDS:                       (~35 keywords)
               find ".{kw}" in source
               if followed by non-identifier char:
                   replacement = make_replacement(kw, source)  ─ same length
                   keyword_renames[replacement] = kw
                   source = source[..pos] + "." + replacement + source[after_pos..]
                   continue outer loop                        ─ retry parse
           if no keyword found:
               return Err with caret span
   ```

## `make_replacement(keyword, source)`

[eval/parse.rs:60](../../../openjd-rs/crates/openjd-expr/src/eval/parse.rs#L60).
Generates a same-length identifier that:

- Does not appear elsewhere in the source (so retries don't collide).
- Is not itself a Python keyword (avoids infinite-loop scenarios).
- Is deterministic for a given `(keyword, source)` pair (so error
  positions stay stable).

The scheme is "first letter of the alphabet that produces a unique
identifier; otherwise increment to the next letter." Examples from
the doc-comment:

```
"if"   → "xf"
"def"  → "xef"
"else" → "xlse"
```

If somehow every prefix collides, the fallback is `"x".repeat(len)` —
won't be unique but the parse will fail with a clean error rather
than loop.

**Same-length replacement is the key invariant.** Ruff's parser
reports error positions in source bytes; if rewrites changed length,
those positions would no longer line up with the user's original
string and caret diagnostics would point at the wrong column.

## Why the renames are stored

The returned `ParsedExpression` carries `keyword_renames:
HashMap<String, String>` ([eval/parse.rs:21](../../../openjd-rs/crates/openjd-expr/src/eval/parse.rs#L21)).
Two later stages need them:

1. **Symbol collection** ([eval/parse.rs:515](../../../openjd-rs/crates/openjd-expr/src/eval/parse.rs#L515) → `collect_symbols`)
   walks the AST and builds dotted names like `Param.xf`. Before
   inserting into `accessed_symbols`, it consults `keyword_renames`
   to undo: stores `Param.if`, not `Param.xf`.

2. **Evaluation** — [`ParsedExpression::evaluator`](../../../openjd-rs/crates/openjd-expr/src/eval/parse.rs#L255)
   passes `keyword_renames` to the `Evaluator` so attribute lookups
   like `Attribute(Name("Param"), "xf")` undo to a symbol table
   lookup of `"Param.if"`.

## The "after non-identifier char" check

```rust
let after_pos = pos + 1 + kw.len();
let after_char = source.chars().nth(after_pos);
if after_char.is_none_or(|c| !c.is_alphanumeric() && c != '_') {
```

This avoids false positives. Without it, `Param.iframe` would match
the keyword `if` followed by `rame`, and the rewriter would smash an
already-valid identifier. The check ensures we only rewrite when the
keyword stands alone — i.e., is followed by a space, paren, dot, or
end-of-string.

## Multi-keyword expressions

The loop is one rewrite per pass; if the expression contains both
`Param.if` and `Param.else`, the first `for` iteration rewrites `if`,
the parser fails at `else`, the loop body re-runs, finds `else`,
rewrites it, retries — and so on. Each rewrite keeps the same length,
so positions stay aligned. There is no upper bound on retries beyond
"all 35 keywords get rewritten" — bounded by the (constant) keyword
list.

## Why not preprocess once instead of looping?

Preprocessing every `.if` blindly would also rewrite `Param.iframe`
(see above) — the Python parser is the authoritative oracle for "is
this keyword acting as an identifier here?" Asking the parser first
and rewriting only what it complains about is more conservative than
trying to parse Python syntax in a pre-pass.

## What this lets you do

```yaml
parameterDefinitions:
  - name: if
    type: STRING
  - name: class
    type: STRING

steps:
  - name: example
    script:
      actions:
        onRun:
          command: echo
          args: ["{{Param.if}}", "{{Param.class}}"]
```

This is a real corner of the OpenJD spec — parameter names are
unconstrained user strings. Without the rewrite, users would be unable
to migrate jobs from systems where `class`, `id`, `for`, `from`, etc.
are common parameter names.

## Related

- [koan 03](03-expression-block-end-to-end.md) — where this fits in
  the larger pipeline.
- [koan 04](04-format-string-segment-scanner.md) — the segment scanner
  that hands strings to `ParsedExpression`.
