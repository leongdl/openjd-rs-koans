# 15 — Expression pipeline

## What this is

How `"min(Task.Param.Frame + 1, Param.End)"` becomes the integer `5`.
This doc traces an expression string through every layer of
`openjd-expr`: keyword rewrite, ruff parser, structural validator,
symbol/function collection, AST transformation, bounded evaluation,
and final value.

By the end you'll know which file each operator dispatches through and
how memory + operation budgets are charged. This pairs with doc 14 —
that doc explains how the *outer* call (`FormatString::resolve`)
reaches the expression engine; this doc is what happens inside.

## Read these in order

- `crates/openjd-expr/src/lib.rs` — public API surface.
- `crates/openjd-expr/src/eval/parse.rs` — `ParsedExpression`,
  keyword rewrite, ruff integration, symbol collection.
- `crates/openjd-expr/src/eval/evaluator.rs` — `Evaluator`,
  AST walk, dispatch, budget tracking.
- `crates/openjd-expr/src/value.rs` — `ExprValue` runtime values.
- `crates/openjd-expr/src/types.rs` — `ExprType`, normalization.
- `crates/openjd-expr/src/symbol_table.rs` — hierarchical symbol lookup.
- `crates/openjd-expr/src/function_library.rs` — `FunctionImpl`, dispatch.
- `crates/openjd-expr/src/default_library.rs` — registers operators
  and built-ins.
- `crates/openjd-expr/src/format_string.rs` — segments and resolution.

## The pipeline at a glance

```
"min(Task.Param.Frame + 1, Param.End)"
  │
  ▼ keyword-rewrite preprocess              [parse.rs: PYTHON_KEYWORDS, make_replacement]
  │   if a Python keyword sits after `.`,
  │   substitute a same-length identifier
  │   and remember the rename
  │
  ▼ ruff_python_parser::parse_expression()  [parse.rs]
  ▼ ruff_python_ast::Expr                    ── Call / BinOp / Name / Constant ...
  │
  ▼ structural validator                    [parse.rs]
  │   • depth ≤ MAX_EXPRESSION_DEPTH (defaults: 100)
  │   • input length ≤ MAX_PARSE_INPUT_LEN (defaults: 64 KB)
  │   • reject unsupported syntax for the
  │     active ExprProfile
  │
  ▼ symbol / function / loop-var collection [parse.rs]
  ▼ ParsedExpression {
        ast,
        expr,                  ── trimmed source
        keyword_renames,
        accessed_symbols,      ── e.g. {"Task.Param.Frame", "Param.End"}
        called_functions,      ── e.g. {"min"}
        local_bindings,        ── loop-vars from list comprehensions
    }
  │
  │   (cached inside FormatString::Segment::Expression — parsed once,
  │    evaluated many times)
  │
  ▼ Evaluator::eval(&parsed, &symtabs, &lib, &mut ctx)  [evaluator.rs]
  │   AST walk, pre-order, with target TypeSet propagated downward
  │   At every node:
  │     • ctx.charge_op()
  │     • periodic ctx.check_memory()
  │     • operators rewritten to function calls (UFCS):
  │         BinOp(left, Add, right)  →  Call("__add__", [left, right])
  │         Call(Attribute(x, m), a) →  Call(m, [x] + a)
  │     • resolve_and_call():
  │         direct match  →  call signature
  │         coerce match  →  promote int↔float, path↔string, etc.
  │     • unresolved[T] propagates: any unresolved operand → unresolved result
  │
  ▼ EvalResult { value: ExprValue, ops_used, memory_high_water }
  │
  ▼ FormatString::resolve(symtabs)            [format_string.rs]
  │   For each Segment::Expression, coerce non-string ExprValue
  │   to string for emission; concatenate with Segment::Literal pieces.
  │
  ▼ String — the resolved value
```

## Stage 1 — Keyword rewrite

```rust
// crates/openjd-expr/src/eval/parse.rs:52
const PYTHON_KEYWORDS: &[&str] = &[
    "False", "None", "True", "and", "as", "assert", "async", "await",
    "break", "class", "continue", "def", "del", "elif", "else", "except",
    "finally", "for", "from", "global", "if", "import", "in", "is",
    "lambda", "nonlocal", "not", "or", "pass", "raise", "return",
    "try", "while", "with", "yield",
];
```

Why: OpenJD allows `Param.if` as a parameter name, but Python's parser
treats `if` as a keyword. The fix is a same-length rewrite:

```text
"min(Param.if, 1)"   →   "min(Param.xf, 1)"
                                     ^^ keyword_renames["xf"] = "if"
```

The replacement must be the *same length* so source positions in error
messages still line up with the original text. After parsing succeeds,
the evaluator (and any error formatter) translates `xf` back to `if`
via `keyword_renames`.

For our example expression, no keywords appear after a dot, so this
stage is a no-op.

## Stage 2 — ruff parsing

```rust
ruff_python_parser::parse_expression(rewritten_source)
```

Returns a `ruff_python_ast::Expr`. We use ruff because (a) it's already
a dependency of broader Rust tooling, (b) it produces source spans we
can attach to error messages, and (c) reusing a Python parser is the
core promise of RFC 0005 — same syntax, subset semantics.

Our example produces:

```text
Call(
    func: Name("min"),
    args: [
        BinOp(
            left: Attribute(value: Attribute(value: Name("Task"), attr: "Param"), attr: "Frame"),
            op: Add,
            right: Constant(Int(1)),
        ),
        Attribute(value: Name("Param"), attr: "End"),
    ],
)
```

## Stage 3 — Structural validator

A walk over the AST that:

- Counts depth, fails with `ExpressionTooDeep` if it exceeds
  `MAX_EXPRESSION_DEPTH`.
- Rejects syntax not in the OpenJD subset: `lambda`, f-strings,
  walrus, `dict`/`set` literals, `*args`, `**kwargs`, multiple `for`
  clauses in a comprehension. Each rejection raises
  `ExpressionErrorKind::UnsupportedSyntax` with the feature name.
- Honours the active `ExprProfile`. A future expr-level extension can
  enable a feature that is rejected under an older profile.

For our example, depth is 4, no banned syntax, accepted.

## Stage 4 — Symbol and function collection

The same walk also collects three sets:

- **`accessed_symbols`** — every dotted attribute chain rooted at a
  `Name`. For our example: `{"Task.Param.Frame", "Param.End"}`.
- **`called_functions`** — function names in `Call` nodes (after
  method-call rewriting): `{"min"}`. We don't include operators like
  `__add__` because they are universally available.
- **`local_bindings`** — comprehension loop variables. None here.

These are collected eagerly because static analysis (RFC 0005's
"validate references at submission time") needs them without
evaluating. The format-string validation pass in `openjd-model` reads
`accessed_symbols` to verify that every reference resolves to a known
symbol *before* runtime.

## Stage 5 — Parsed result is cached

```rust
// crates/openjd-expr/src/format_string.rs (paraphrased)
enum Segment {
    Literal(String),
    Expression { start: usize, end: usize, parsed: ParsedExpression },
}
```

Every `{{ ... }}` interpolation is parsed *once* when the
`FormatString` is constructed. `ParsedExpression` is `Clone`-cheap
(its AST is in an `Arc` internally) and re-evaluated for every task.
This is why a 10,000-task job doesn't pay parsing overhead 10,000
times.

## Stage 6 — Evaluation

```rust
// crates/openjd-expr/src/eval/evaluator.rs (paraphrased)
pub fn evaluate_expression(
    node: &Expr,
    target_types: TypeSet,         // None means unconstrained
    symtabs: &[&SymbolTable],      // local-to-global scope chain
    lib: &FunctionLibrary,
    ctx: &mut dyn EvalContext,     // charges ops + memory
) -> Result<ExprValue, ExpressionError>
```

The walk is pre-order with target-type propagation downward. Key
behaviours:

### Operator rewriting (UFCS)

Operators become function calls before dispatch. From RFC 0005 §AST
Transformation:

```text
BinOp(a, Add, b)        → Call("__add__", [a, b])
UnaryOp(Not, a)         → Call("__not__", [a])
Compare(a, [Lt], [b])   → Call("__lt__", [a, b])
Subscript(a, i)         → Call("__getitem__", [a, i])
Call(Attr(x, "f"), [a]) → Call("f", [x, a])    // method → free function
```

This means every node ultimately becomes a function-library lookup,
which simplifies dispatch.

### Function dispatch

```rust
// crates/openjd-expr/src/function_library.rs:42
pub type FunctionImpl = Arc<
    dyn Fn(&[ExprValue], &mut dyn EvalContext) -> Result<ExprValue, ExpressionError>
        + Send + Sync,
>;
```

The library is a `HashMap<String, Vec<Arc<FunctionImpl>>>`. Multiple
overloads share a name; `resolve_and_call` walks them looking for:

1. **Direct match** — argument types equal a signature's parameter
   types. Best case, ~`O(overloads)`.
2. **Coerced match** — at least one argument coerces non-destructively
   (`int → float`, `path → string`, `list[int] → list[float]`, etc).
   Method calls **skip coercion on the receiver** — see RFC 0005's
   "Method Call Coercion Restriction."

For `__add__(int, int) → int`, both arguments are `Int`, so the direct
match wins.

For `min(int, int) → int`, same.

### Budget charging

```rust
ctx.charge_op()         // every function call, every property access
ctx.charge_for_string(s) // strings cost ceil(len / 256) ops
ctx.charge_for_list(n)   // list-traversing builtins cost n ops
ctx.check_memory(estimated_size)
```

Defaults: 10 million ops, 100 MB memory. Both are configurable on
`EvalBuilder`. When either limit is reached, evaluation fails with
`OperationLimitExceeded` or `MemoryLimitExceeded`.

This is what makes the engine safe to run on untrusted templates.

### Worked example

For `min(Task.Param.Frame + 1, Param.End)` with `Task.Param.Frame = 4`,
`Param.End = 10`:

1. Walk into `Call("min", [...])`. Charge 1 op.
2. Walk into first arg `BinOp(Add)`. Charge 1 op for the `__add__` call.
   - Resolve `Task.Param.Frame` → lookup in symtab → `ExprValue::Int(4)`.
   - Resolve `Constant(1)` → `ExprValue::Int(1)`.
   - Dispatch `__add__(int, int) → int` → `ExprValue::Int(5)`.
3. Walk into second arg `Attribute(Param.End)`. Resolve to `ExprValue::Int(10)`.
4. Dispatch `min(int, int) → int` with `[Int(5), Int(10)]` → `ExprValue::Int(5)`.
5. Return `EvalResult { value: Int(5), ops_used: 4, memory: tiny }`.

## Stage 7 — Format-string emission

The outermost layer (called from `openjd-model` during job creation
and from `openjd-sessions` at task time) is `FormatString::resolve`:

```rust
// format_string.rs (paraphrased)
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

`to_string_for_format` is the implicit-coercion-to-string pass: an `int`
becomes `"5"`, a `path` becomes its OS-formatted string, a `list[int]`
becomes `"[1, 2, 3]"`. RFC 0005 §Format String Coercion specifies
exactly which conversions apply.

## Progressive evaluation across stages

The same evaluator runs at three different points with different symbol
tables:

| When | What's concrete | What's `unresolved[T]` |
|---|---|---|
| Template validation | (very little) | `Param.*`, `Task.Param.*`, `Session.*` |
| Job creation | `Param.*`, `let` bindings, `Job.Name` | `Task.Param.*`, `Session.*`, `Task.File.*`, `Env.File.*` |
| Worker host | everything | (nothing) |

`Unresolved(t)` is a real `ExprValue` variant
(`crates/openjd-expr/src/value.rs:155`) that carries its constraint
type. When any operand is unresolved, the result is unresolved with the
inferred type. This catches type errors at submission time even though
the value isn't yet known. RFC 0005 §Progressive Expression Evaluation
explains the model.

## Exercise

```sh
# 1. Run the parse tests in isolation.
cargo test -p openjd-expr eval::parse

# 2. Hit the depth limit.
./target/release/openjd-rs check <<'YAML'
specificationVersion: jobtemplate-2023-09
extensions: [EXPR]
name: "{{ ((((((((((1)))))))))) }}"
steps:
  - name: S
    script:
      actions:
        onRun: { command: "echo", args: [] }
YAML
# Add more parens until you see ExpressionTooDeep.

# 3. Hit the operation limit.
./target/release/openjd-rs check <<'YAML'
specificationVersion: jobtemplate-2023-09
extensions: [EXPR]
name: "{{ sum(range(20000000)) }}"
steps:
  - name: S
    script:
      actions:
        onRun: { command: "echo", args: [] }
YAML
# Should fail with OperationLimitExceeded — sum + range together cost
# 2 * N ops which exceeds the 10M default.

# 4. Force a type error caught at submission time.
./target/release/openjd-rs check <<'YAML'
specificationVersion: jobtemplate-2023-09
extensions: [EXPR]
parameterDefinitions:
  - { name: Count, type: INT, default: 1 }
name: "{{ Param.Count.upper() }}"
steps:
  - name: S
    script:
      actions:
        onRun: { command: "echo", args: [] }
YAML
# 'upper' is a string method; on int it's an error. Note that this
# fails at `check` time even though Param.Count is "concrete" — the
# evaluator still reports a TypeError because the dispatch fails.
```

For each error, find the matching `ExpressionErrorKind` variant in
`crates/openjd-expr/src/error.rs`.

## What's next

[16 — onWrap dispatch and parameter injection](./16-onwrap-and-parameter-injection.md).
The expression engine you just traced is what `Task.*` and
`Env.Wrapped.*` are seeded into; doc 16 shows how those overlays are
constructed at runtime.
