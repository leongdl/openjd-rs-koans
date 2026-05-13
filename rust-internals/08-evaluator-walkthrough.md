# 08 — Evaluator walkthrough

## What this is

The AST walker that produces an `ExprValue` from a `ParsedExpression`.
Doc 15 traces the full pipeline end-to-end with a worked example;
this doc focuses on the evaluator's internal mechanics: target-type
propagation, UFCS rewriting, dispatch, budget tracking, unresolved
propagation.

## Read these in order

- `crates/openjd-expr/src/eval/evaluator.rs` (top) — `Evaluator`,
  `EvalResult`, `DEFAULT_MEMORY_LIMIT`, `DEFAULT_OPERATION_LIMIT`.
- `crates/openjd-expr/src/function_library.rs:39-100` —
  `FunctionImpl`, `EvalContext` trait, dispatch entry points.
- `crates/openjd-expr/src/eval/evaluator.rs` (rest) — the recursive
  walk, target-type propagation, comprehension scope.

## The traversal

The evaluator is a pre-order recursive walk over `ruff_python_ast::Expr`
nodes. Each node has a *target type set* propagated from its parent
(or `None` for "unconstrained") and produces an `ExprValue` plus an
updated budget.

```
evaluate_expression(node, target_types, symtabs, lib, ctx) -> ExprValue
```

The walk dispatches on AST node kind:

| Node | What happens |
|---|---|
| `IfExp` | Evaluate `test` with `{BOOL}`. Branch on result; recurse into `body` or `orelse` with parent's target types. |
| `BoolOp(And/Or)` | Short-circuit value-returning evaluation. Operands evaluated unconstrained. |
| `UnaryOp` / `BinOp` / `Compare` | Rewrite to `Call("__op__", [...])` (UFCS), evaluate operands unconstrained, dispatch via function library. |
| `Subscript` | Rewrite to `Call("__getitem__", [...])`. |
| `Call(Name(f), args)` | Function call: find candidate signatures, propagate per-arg type sets, evaluate args, dispatch. |
| `Call(Attribute(x, m), args)` | Method call: evaluate receiver unconstrained, then dispatch as `f(receiver, ...args)` with **no coercion on receiver**. |
| `Attribute` / `Name` | Look up in `symtabs`; check type compatibility against target. |
| `Constant` | Wrap in `ExprValue`; check type compatibility. |
| `List` | Recurse into elements with extracted element type; assemble typed list variant. |
| `ListComp` | Push child symtab with loop variable; iterate; collect results into typed list. |

## Target type propagation

The evaluator threads a `TypeSet` (an `Option<HashSet<ExprType>>`)
downward. `None` means "unconstrained — return whatever the
expression naturally produces."

The propagation rules (RFC 0005 §Target Type Propagation Rules):

| Node | What's propagated to children |
|---|---|
| `IfExp` | `test`: `{BOOL}`. `body`/`orelse`: inherit. |
| `BoolOp` | Both: `None`. |
| `BinOp` / `UnaryOp` / `Compare` | All operands: `None`. |
| `Call` (function) | Per-argument type sets computed from candidate signatures. |
| `Call` (method) | Receiver: `None`. Other args: from signatures. |
| `Subscript` | Value: `None`. Index/slice: `{INT}` or `{INT?}`. |
| `List` | Element type extracted from parent target types. |
| `ListComp` | Element expr: parent's element type. Iter: `None`. Conditions: `{BOOL}`. |

The key rationale: arithmetic operators have *fixed* signatures, so
constraining their operands by an outer `{STRING}` target would be
wrong. Instead, evaluate the operands unconstrained, perform the
operation, and coerce *the result* to the target context.

## UFCS rewriting

```text
BinOp(left, Add, right)         →  Call("__add__", [left, right])
UnaryOp(Not, operand)           →  Call("__not__", [operand])
Compare(left, [Lt], [right])    →  Call("__lt__", [left, right])
Subscript(value, index)         →  Call("__getitem__", [value, index])
Call(Attribute(x, "f"), [a])    →  Call("f", [x, a])  // method → free function
```

After this transform, every operation is a function-library lookup.
The dispatcher only needs one code path. The naming convention
(`__add__`, `__not__`, etc.) matches Python's dunder methods so the
function library can be documented and tested uniformly.

## Function dispatch

```rust
// crates/openjd-expr/src/function_library.rs:42
pub type FunctionImpl = Arc<
    dyn Fn(&[ExprValue], &mut dyn EvalContext) -> Result<ExprValue, ExpressionError>
        + Send + Sync,
>;
```

The library is a `HashMap<String, Vec<Arc<FunctionImpl>>>` (with
typed signature metadata in a parallel structure). Multiple overloads
of `min`, `__add__`, `len`, etc. share a name; dispatch picks the
first signature that matches.

The dispatch order:

1. **Direct match** — argument types equal a signature's parameter
   types. Cheapest, used most.
2. **Coerced match** — at least one argument coerces non-destructively
   (`int → float`, `path → string`, etc.). For method calls, the
   receiver does *not* participate in coercion (see RFC 0005 §Method
   Call Coercion Restriction).

Dispatch raises `TypeError` if no signature matches.

## Budget tracking

The evaluator threads a `&mut dyn EvalContext` everywhere. The
context exposes:

```rust
trait EvalContext {
    fn charge_op(&mut self) -> Result<(), ExpressionError>;       // 1 op
    fn charge_for_string(&mut self, s: &str) -> Result<(), ExpressionError>;
    fn charge_for_list(&mut self, n: usize) -> Result<(), ExpressionError>;
    fn check_memory(&mut self, estimated_size: usize) -> Result<(), ExpressionError>;
    // … (path mapping rules accessor when host context is present)
}
```

The default implementation tracks an op count + a high-water memory
mark and fails with `OperationLimitExceeded` /
`MemoryLimitExceeded` when either threshold is crossed.

Defaults:

```rust
pub const DEFAULT_OPERATION_LIMIT: usize = 10_000_000;
pub const DEFAULT_MEMORY_LIMIT: usize    = 100_000_000;   // 100 MB
```

Both can be overridden via `EvalBuilder`. Validators in long-running
services typically tighten them by an order of magnitude.

## Unresolved propagation

`Unresolved(t)` is a real `ExprValue` variant. When any operand is
`Unresolved`, the result is too:

```text
Int(1) + Unresolved(int)                  →  Unresolved(int)
Unresolved(int) + Unresolved(float)       →  Unresolved(float)
[Int(1), Unresolved(int)]                 →  Unresolved(list[int])  // hoisted
```

The hoisting rule (`unresolved` always outermost) from doc 01 is
what keeps this clean. Operations test the type code first; if it's
`Unresolved`, they synthesise an `Unresolved(result_type)` without
trying to compute on placeholder data.

This is what makes static type-checking work: validate-time
evaluation runs the *same code path* as runtime evaluation, with
some symbols replaced by `Unresolved(t)` placeholders. Type errors
caught at validate time are real — they would fire at runtime with
real data too.

## List comprehensions

```rust
// ListComp(elt, generator)
let iter_val = evaluate(generator.iter, None, symtabs);
for item in iterable_items(iter_val) {
    let local_symtab: SymbolTable = [(generator.target, item)].into();
    let local_symtabs = [&local_symtab].chain(symtabs);
    if all generator.ifs evaluate to true under local_symtabs {
        results.push(evaluate(elt, elem_target, local_symtabs));
    }
}
```

The loop variable is added to a *child* symbol table, layered on top
of the existing scope chain. This is why doc 02 emphasised the list-
of-tables shape — comprehensions need scope nesting.

## EvalResult

```rust
pub struct EvalResult {
    pub value: ExprValue,
    pub ops_used: usize,
    pub memory_high_water: usize,
}
```

Returned from `Evaluator::eval`. Useful for benchmarking and for
asserting in tests that an expression stayed under a specific budget.

## Exercise

```sh
# 1. Run the evaluator tests.
cargo test -p openjd-expr eval::evaluator

# 2. Construct an expression that exercises each major node type:
#    - IfExp:    {{ 1 if True else 2 }}
#    - BoolOp:   {{ True and False }}
#    - BinOp:    {{ 1 + 2 }}
#    - Compare:  {{ 1 < 2 }}
#    - Call:     {{ min(1, 2) }}
#    - Method:   {{ "x".upper() }}
#    - Subscript:{{ [1, 2, 3][1] }}
#    - ListComp: {{ [x * 2 for x in [1, 2, 3]] }}

# 3. Force budget exhaustion.
#    Op limit:    {{ sum(range(20000000)) }}
#    Memory limit:{{ "a" * 200000000 }}

# 4. Force an unresolved propagation. In a `check` context, where
#    Param.* is unresolved at validate time, write:
#      name: "{{ Param.X.upper() }}"
#    with Param.X declared as INT.
#    The evaluator dispatches `upper(int)`, fails to find a matching
#    signature, and reports a TypeError at submit time — even though
#    Param.X has a real value. The unresolved walk still catches
#    the type mismatch.
```

## What's next

[09 — Expression errors](./09-expression-errors.md). The evaluator
produces typed errors throughout; doc 09 enumerates the variants and
shows how to match on them in caller code.
