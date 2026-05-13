# 01 — Expression types and values

## What this is

The two foundational data structures of `openjd-expr`: `ExprType` (the
type-level descriptor) and `ExprValue` (the runtime value). Almost
every other file in the workspace touches one of these, so reading
them first makes the rest of the codebase legible. By the end of this
doc you'll know why `ExprType` enforces normalization through
constructors, why `ExprValue` has *typed* list variants instead of a
single `Vec<ExprValue>`, and why both enums are `#[non_exhaustive]`.

## Read these in order

- `crates/openjd-expr/src/types.rs:14-160` — `TypeCode` enum,
  `ExprType` struct, constants, smart constructors (`list`, `union`,
  `unresolved`).
- `crates/openjd-expr/src/types.rs` (rest) — normalization rules
  (`normalize_union`, hoisting `unresolved` outward).
- `crates/openjd-expr/src/value.rs:14-100` — `Float64` invariants
  (no NaN, no Inf, normalized `-0.0`).
- `crates/openjd-expr/src/value.rs:118-260` — the `ExprValue` enum:
  `Null`, `Bool`, `Int`, `Float`, `String`, `Path`, the typed list
  variants, `RangeExpr`, `Unresolved`.
- `crates/openjd-expr/src/value.rs` — `Hash` impl: cross-type equality
  (`Int(1) == Float(1.0)`, `String("x") == Path { value: "x", .. }`)
  forces consistent hashing.

## The one mental model

**Types describe shape; values carry data; both enums are sealed for
forward compat. Construction is funnelled through smart constructors
so invariants (no NaN, normalized `-0.0`, `unresolved` always at the
outermost layer) hold by construction, not by audit.**

## TypeCode and ExprType

```rust
// crates/openjd-expr/src/types.rs:19
#[non_exhaustive]
pub enum TypeCode {
    NullType, Bool, Int, Float, String, Path, List, RangeExpr,
    Any, Union, NoReturn, Unresolved,
    TypeVarT, TypeVarT1, TypeVarT2, TypeVarT3,
    Signature,
}
```

`ExprType` itself wraps a `TypeCode` plus optional type parameters:

```rust
pub struct ExprType {
    code: TypeCode,
    params: Vec<ExprType>,        // private — go through constructors
}
```

The fields are `pub(crate)`. External callers use:

| Constructor | Purpose |
|---|---|
| Constants `ExprType::BOOL`, `INT`, `FLOAT`, `STRING`, `PATH`, … | Cheap, allocation-free for primitive types |
| `ExprType::list(elem)` | Wraps elem; **hoists `unresolved` outward** automatically |
| `ExprType::union(types)` | Sorts, dedupes, flattens nested unions, applies absorption rules (`Any` swallows everything; `NoReturn` collapses) |
| `ExprType::unresolved(constraint)` | Avoids `unresolved[unresolved[T]]` nesting |
| `ExprType::signature(param_types, return_type)` | Function signatures used by the dispatcher |

The hoisting rule is the trick that keeps the type system tractable.
`list[unresolved[T]]` would be evil — every list operation would have
to peer inside the element type. Instead, the constructor *normalizes*
to `unresolved[list[T]]`, so the unresolved wrapper is always
outermost and operations short-circuit on it cleanly.

## ExprValue and Float64

```rust
// crates/openjd-expr/src/value.rs:127
#[non_exhaustive]
pub enum ExprValue {
    Null,
    Bool(bool),
    Int(i64),
    Float(Float64),
    String(String),
    #[non_exhaustive]                       // see below
    Path { value: String, format: PathFormat },
    ListBool(Vec<bool>),
    ListInt(Vec<i64>),
    ListFloat(Vec<Float64>),
    ListString(Vec<String>, usize),         // (elements, cached heap size)
    ListPath(Vec<String>, PathFormat, usize),
    ListList(Vec<ExprValue>, ExprType, usize),
    RangeExpr(RangeExpr),
    Unresolved(ExprType),
}
```

Three details that surprise readers:

### 1. Two `#[non_exhaustive]` attributes

The outer one is for forward compat: future revisions may add a
`Duration`, `Url`, `Decimal` variant. External `match` statements
must include `_ => …`.

The inner one (on the `Path` variant) prevents external struct-literal
construction. Callers must use `ExprValue::new_path(…)`, which enforces
the separator-normalization invariant (`\` ↔ `/` per the path's
`PathFormat`, no normalization for URI paths).

### 2. Typed list variants instead of `Vec<ExprValue>`

`ListInt(Vec<i64>)` is one machine word per element. A hypothetical
`Vec<ExprValue>` of ints would be ~32 bytes per element (enum tag +
discriminant + payload). For a job iterating 100,000 frames as a
range, the difference is ~3 MB versus ~24 KB. The typed variants are
the cache-friendly representation.

When list elements are heterogeneous, `ListList` falls back to the
boxed form.

### 3. Cached heap size on string/path/list variants

`ListString(Vec<String>, usize)` carries a precomputed `heap_size()`
in the second field. The evaluator's memory budget (RFC 0005
§Memory-Bounded Evaluation) needs this constantly; recomputing it
would be O(n) per check. Construction goes through
`make_list_string`/`make_list_path`/`make_list_list` which compute
once.

### 4. `Float64` rejects NaN and Inf

```rust
// crates/openjd-expr/src/value.rs:39
impl Float64 {
    pub fn new(v: f64) -> Result<Self, ExpressionError> {
        let v = normalize_zero(v);            // -0.0 → 0.0
        if v.is_nan() { return Err(...); }
        if v.is_infinite() { return Err(...); }
        Ok(Self { value: v, original: None })
    }
}
```

This is RFC 0005 §3 ("No negative zero, infinity, or NaN"). The
invariants hold by construction, so the `Hash` and `PartialEq` impls
on `ExprValue` can rely on them.

`Float64` also carries an optional `original: Option<Box<str>>` —
the original string representation before float conversion. RFC
0005's "Float Value Pass-Through" rule preserves `"3.500"` as-is when
the value is only copied, switching to shortest-decimal only after
arithmetic.

## Cross-type equality drives the Hash impl

```rust
// crates/openjd-expr/src/value.rs:158 (paraphrased)
impl Hash for ExprValue {
    fn hash<H: Hasher>(&self, state: &mut H) {
        match self {
            Int(i) => { 2u8.hash(state); i.hash(state); }
            Float(f) => {
                let v = f.value();
                if v.fract() == 0.0 && v >= i64::MIN as f64 && v <= i64::MAX as f64 {
                    // Exact integer-valued float hashes as Int.
                    2u8.hash(state); (v as i64).hash(state);
                } else {
                    12u8.hash(state); v.to_bits().hash(state);
                }
            }
            String(s) | Path { value: s, .. } => { 3u8.hash(state); s.hash(state); }
            ...
        }
    }
}
```

Two observations:

- `Int(1)` and `Float(1.0)` must hash identically because RFC 0005
  says they're equal. The branch on `fract() == 0.0` is what ensures
  this.
- `String("x")` and `Path { value: "x", .. }` hash identically because
  RFC 0006 specifies `string == path` cross-type equality.

If you're ever tempted to add a new `ExprValue` variant, the `Hash`
impl is where you must update first.

## Exercise

```sh
# 1. Read the unit tests for type normalization.
cargo test -p openjd-expr types::tests

# 2. Try to construct a NaN float — confirm it errors.
cat > /tmp/test.rs <<'RS'
fn main() {
    use openjd_expr::value::Float64;
    let r = Float64::new(f64::NAN);
    assert!(r.is_err(), "NaN must be rejected");
    println!("NaN rejected: {:?}", r.err());
}
RS
# (Conceptual — the doc shows the API, you don't need to actually
# compile this snippet. Open value.rs and read Float64::new instead.)

# 3. Read the cross-type equality tests.
cargo test -p openjd-expr value::tests::eq_int_float
cargo test -p openjd-expr value::tests::eq_string_path
```

Confirm that `Int(1) == Float(1.0)` and that they hash to the same
value. This is the property that makes `[Int(1), Int(2)] ==
[Float(1.0), Float(2.0)]` work — without consistent hashing, the
typed list variants would silently misbehave.

## What's next

[02 — Symbol table](./02-symbol-table.md). Now that you know what an
`ExprValue` is, the next layer is the hierarchical lookup structure
that maps `Param.Frame`-style names to them.
