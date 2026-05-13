# 07 — Expression parsing and keyword rewrite

## What this is

How `openjd-expr` turns an expression string into a parseable AST,
including the contextual-keyword trick that lets `Param.if`,
`Param.True`, and other Python-keyword attribute names work despite
ruff treating them as keywords. This is a focused look at one phase
of the pipeline; doc 15 shows it in the context of full evaluation.

## Read these in order

- `crates/openjd-expr/src/eval/parse.rs:1-80` — `ParsedExpression`
  struct, `PYTHON_KEYWORDS` list, `make_replacement`.
- `crates/openjd-expr/src/eval/parse.rs:90-200` — the parse loop:
  ruff parse → on error, locate keyword after `.`, substitute,
  retry.
- `crates/openjd-expr/src/eval/parse.rs` (rest) — symbol/function/
  loop-var collection walks.

## Why this is hard

OpenJD allows parameter names to be any identifier:

```yaml
parameterDefinitions:
  - name: if
    type: STRING
```

In the expression language, you'd then refer to it as `Param.if`. But
ruff (and any standard Python parser) treats `if` as a keyword and
will reject `Param.if` with a syntax error. RFC 0005's contextual-
keyword rule says: keywords are *only* keywords in operator position,
not after a `.`. The implementation has to bridge that gap.

## The strategy

1. **Try ruff parse on the original source.**
2. **If it fails with a syntax error, look at the error position.**
   If the failing token is preceded by a `.`, it's a contextual-
   keyword problem.
3. **Substitute the keyword with a same-length identifier** that
   doesn't collide with anything already in the source.
4. **Retry.** Repeat until ruff accepts.
5. **Record every rename** so the evaluator (and error formatter) can
   translate back.

The same-length constraint is critical: if we substitute `xf` for
`if`, every byte offset after that point still lines up with the
original source, so error spans from later phases continue to point
at the right text.

## The implementation

```rust
// crates/openjd-expr/src/eval/parse.rs:52
const PYTHON_KEYWORDS: &[&str] = &[
    "False", "None", "True", "and", "as", "assert", "async", "await",
    "break", "class", "continue", "def", "del", "elif", "else", "except",
    "finally", "for", "from", "global", "if", "import", "in", "is",
    "lambda", "nonlocal", "not", "or", "pass", "raise", "return",
    "try", "while", "with", "yield",
];

// :60
fn make_replacement(keyword: &str, source: &str) -> String {
    // Deterministic same-length scheme: prefix with a letter and
    // copy the rest of the keyword's structure.
    //   "if"   → "xf"
    //   "def"  → "xef"
    //   "else" → "xlse"
    // If the candidate collides, try a different prefix.
    ...
}
```

The candidate must (a) be the same byte length, (b) not appear
elsewhere in the source, (c) not itself be a Python keyword. Each
of those is cheap to check.

## ParsedExpression

After successful parse, the result carries everything downstream
needs:

```rust
pub struct ParsedExpression {
    ast: ruff_python_ast::Expr,
    expr: String,                      // trimmed source
    source: String,                    // original untrimmed
    keyword_renames: HashMap<String, String>,  // "xf" → "if"
    accessed_symbols: HashSet<String>,         // {"Param.if", "Task.Param.X"}
    called_functions: HashSet<String>,         // {"min", "len"}
    local_bindings: HashSet<String>,           // loop vars from list comps
}
```

Three accessors:

```rust
parsed.accessed_symbols()   // for static reference validation
parsed.called_functions()   // for "is apply_path_mapping called?" checks
parsed.local_bindings()     // for shadow detection
```

## Symbol/function collection

After parsing, a single AST walk populates the three sets above. The
walk is independent of evaluation — it answers "what does this
expression *reference*?" without ever looking up a value.

The model's format-string validator uses `accessed_symbols` to verify
that every reference resolves to a known symbol *before* runtime.
This is the static-validation half of RFC 0005's "fail-fast" goal.

`called_functions` is used to detect host-context-only functions
(like `apply_path_mapping`) appearing in submission-time scopes — the
validator can flag this without evaluating.

## Two parse entry points

```rust
impl ParsedExpression {
    // "Latest" profile — every revision + extension on. Unstable
    // across crate versions.
    pub fn new(expr: &str) -> Result<Self, ExpressionError>;

    // Stable: pin the profile and the same input parses identically
    // across crate upgrades.
    pub fn with_profile(expr: &str, profile: &ExprProfile)
        -> Result<Self, ExpressionError>;
}
```

The dichotomy from doc 03 again. Tests use `new`; production code
uses `with_profile` after deriving a profile from the template.

## Error mapping

| ruff error | Wrapped as |
|---|---|
| Syntax error not at a keyword-after-`.` position | `ExpressionErrorKind::ParseError(...)` |
| Unsupported feature (lambda, walrus, f-string, etc.) | `ExpressionErrorKind::UnsupportedSyntax { feature }` |
| Depth exceeded | `ExpressionErrorKind::ExpressionTooDeep { depth, limit }` |
| Input length exceeded | `ExpressionErrorKind::Other("input too long ...")` |

The wrapping happens during the parse-and-retry loop. Once
`ParsedExpression` succeeds, none of these errors can fire again.

## Exercise

```sh
# 1. Run the parse tests, especially the keyword-rewrite ones.
cargo test -p openjd-expr eval::parse::tests::keyword

# 2. Construct an expression that triggers a rewrite:
./target/release/openjd-rs check <<'YAML'
specificationVersion: jobtemplate-2023-09
extensions: [EXPR]
parameterDefinitions:
  - { name: if, type: STRING, default: "x" }   # parameter literally named "if"
name: "{{ Param.if + Param.if }}"
steps:
  - name: S
    script:
      actions:
        onRun: { command: "echo", args: [] }
YAML
# Should validate cleanly. The implementation rewrote `if` to `xf`,
# parsed, and then mapped back when constructing accessed_symbols.

# 3. Try an unsupported feature:
./target/release/openjd-rs check <<'YAML'
specificationVersion: jobtemplate-2023-09
extensions: [EXPR]
name: "{{ lambda x: x + 1 }}"
steps:
  - name: S
    script:
      actions:
        onRun: { command: "echo", args: [] }
YAML
# Should fail with UnsupportedSyntax (lambda).
```

## What's next

[08 — Evaluator walkthrough](./08-evaluator-walkthrough.md). Now that
you know how parsing produces a `ParsedExpression`, the next doc
follows that structure into the bounded AST evaluator.
