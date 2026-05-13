# 02 — Symbol table

## What this is

The hierarchical name → value structure that backs every expression
lookup. `Param.Frame`, `Task.Param.F`, `Session.WorkingDirectory`,
`Env.File.Run` — every dotted reference resolves through a
`SymbolTable`. The structure is small, recursive, and worth reading
once carefully so the dispatch logic in doc 08 looks obvious.

## Read these in order

- `crates/openjd-expr/src/symbol_table.rs:1-100` — module docs,
  `MAX_SYMBOL_TABLE_ENTRIES`, `SymbolTableEntry`.
- `crates/openjd-expr/src/symbol_table.rs` (rest) — `SymbolTable::set`,
  `set_table`, `lookup`, the `symtab!` macro.
- `crates/openjd-expr/src/symbol_table.rs` (`SerializedSymbolTable`)
  — the JSON transport format used between sessions and clients.

## The shape

```rust
// crates/openjd-expr/src/symbol_table.rs:93
pub enum SymbolTableEntry {
    Table(SymbolTable),
    Value(ExprValue),
}

pub struct SymbolTable {
    entries: HashMap<String, SymbolTableEntry>,
}
```

A table is a map. Each entry is either *another table* (for nested
namespaces like `Param`, `Task.Param`) or *a value* (the leaf data the
evaluator reads). That's the entire data structure.

## Dotted-path API

```rust
// Construction (verbose form)
let mut st = SymbolTable::new();
st.set("Param.Frame", 42)?;             // creates Param table, then Frame value
st.set("Param.Name", "test")?;
st.set("Session.WorkingDirectory",
       ExprValue::new_path("/tmp", PathFormat::Posix))?;
```

`set("A.B.C", value)` walks the dotted path, auto-creating
intermediate `Table` entries, and inserts at the leaf. Conflicts (for
example, calling `set("Param.Frame.Foo", ...)` after `set("Param.Frame",
42)` was already a value) raise `SymbolTableError`.

Lookups follow the inverse:

```rust
let val: Option<&ExprValue> = st.lookup("Param.Frame");
```

The evaluator (doc 08) actually uses *a list* of symbol tables —
local-to-global — to support list-comprehension scope:

```rust
let symtabs: Vec<&SymbolTable> = vec![&loop_var_scope, &task_scope, &job_scope];
```

The first table that has the key wins. This is how `[x for x in
Param.Items]` shadows `x` only inside the comprehension.

## The `symtab!` macro

For tests and ad-hoc code, the macro is the concise form:

```rust
let st = symtab! {
    "Param.Frame" => 42,
    "Param.Name"  => "test",
    "Session.Dir" => ExprType::PATH,    // auto-wraps as Unresolved(PATH)
};
```

Three nice tricks:

- Integer / string / bool literals coerce into `ExprValue` via the
  inherent `From` impls on `ExprValue`.
- An `ExprType` value is interpreted as `Unresolved(type)`. This is
  the common shape during static type-checking, where you know the
  declared type of a parameter but not its value yet.
- An explicit `ExprValue` always works for cases the inference can't
  handle.

## `MAX_SYMBOL_TABLE_ENTRIES`

```rust
// :56
pub const MAX_SYMBOL_TABLE_ENTRIES: usize = 100_000;
```

This cap applies **only** to the `serde::Deserialize` impl on
`SymbolTable` (the JSON transport format used to ship symbols
between processes). The cap exists because untrusted callers could
otherwise produce a multi-million-entry table to inflate the worker's
memory before evaluation begins.

The cap does *not* apply to direct in-process API calls. If you have
a legitimate test that needs a million entries, build it with
`SymbolTable::set` calls — those are trusted host code.

## `SerializedSymbolTable` — the wire format

```rust
pub struct SerializedSymbolTable { ... }
```

A flat array of `(dotted_key, ExprValue)` pairs that round-trips a
`SymbolTable` through JSON. Used by:

- The session/worker protocol to ship resolved parameters.
- The expression CLI for testing.
- The JS bindings (`openjd-for-js`) to receive a populated table from
  JavaScript.

The flat form is deliberate: nested JSON would be more compact for
deeply-nested namespaces, but the flat form is trivial to validate
and bound, and `MAX_SYMBOL_TABLE_ENTRIES` applies to the array
length directly.

## Why hierarchical instead of just `HashMap<String, ExprValue>`?

The flat form would work for lookup, but breaks two important uses:

1. **Static introspection.** The validator wants to ask "does the
   `Param.*` namespace exist?" without knowing every parameter name.
   With nested tables, that's a single lookup; with flat keys, it's
   a prefix scan.
2. **Set-as-table for scoped overlays.** During expression evaluation
   for a step's `let` bindings, the runtime layers a fresh nested
   table on top. That's a single `set_table` call; with flat keys
   it would be a key-by-key copy.

The hierarchical form also matches the Python implementation, which
keeps the on-the-wire format consistent across runtimes.

## Exercise

```sh
# 1. Read the unit tests for symbol table dotted paths.
cargo test -p openjd-expr symbol_table

# 2. Open the macro definition. (Search for `macro_rules! symtab`.)
$EDITOR crates/openjd-expr/src/symbol_table.rs

# 3. Construct a deliberately conflicting setup and confirm it errors:
#
#    let mut st = SymbolTable::new();
#    st.set("Param.Frame", 42)?;             // ok
#    st.set("Param.Frame.Foo", 1)?;          // err: Param.Frame is value
```

The third call should produce `SymbolTableError { key: "Param.Frame.Foo",
conflict: "Param.Frame" }`.

## What's next

[03 — Profile, revision, extension, host context](./03-profile-revision-extension-host.md).
The symbol table is one of two inputs to evaluation; the other is the
profile that determines which functions and types are available.
