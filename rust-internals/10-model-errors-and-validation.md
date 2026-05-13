# 10 — Model errors and validation

## What this is

The two-layer error model on the model crate: `ModelError` is the
outer enum returned from every public API; `ValidationErrors` is the
structured *list* of per-field problems carried inside one variant.
After this doc you'll know which variant fires for which class of
failure, what `PathElement` and `DiagnosticSpan` are for, and why
validation accumulates errors instead of bailing on the first.

## Read these in order

- `crates/openjd-model/src/error.rs:1-70` — `ModelError` enum and
  the format-string error helper.
- `crates/openjd-model/src/error.rs:80-160` — `ValidationError`,
  `ValidationErrors`, `PathElement`, `ErrorDetail`, `DiagnosticSpan`.
- `crates/openjd-model/src/template/validate_v2023_09/mod.rs` —
  validation orchestrator that accumulates problems.
- A few representative passes: `structure.rs`, `format_strings.rs`,
  `wrap_actions.rs`.

## ModelError — the outer enum

```rust
// crates/openjd-model/src/error.rs:10
#[derive(Debug, thiserror::Error)]
#[non_exhaustive]
pub enum ModelError {
    /// Phase-1 or phase-2 deserialization failure.
    DecodeValidation(String),

    /// Phase-3 semantic validation failure. Carries the structured list.
    ModelValidation(ValidationErrors),

    /// A specific format-string failed to parse with position info.
    FormatStringError {
        message: String,
        input: Option<String>,
        start: Option<usize>,
        end: Option<usize>,
    },

    /// An expression evaluation or symbol-table error. Preserves the
    /// full ExpressionError so callers can reach the kind.
    Expression(#[source] openjd_expr::ExpressionError),

    /// A compatibility issue (rare).
    Compatibility(String),

    /// The template's specificationVersion isn't recognized.
    UnsupportedSchema(String),
}
```

The variants serve different audiences:

| Variant | Carries | When |
|---|---|---|
| `DecodeValidation` | `String` | Phase 1 / 2: corrupt YAML, unknown field, type mismatch |
| `ModelValidation` | `ValidationErrors` (a list) | Phase 3: any number of semantic problems |
| `FormatStringError` | message + input + span | A specific format string failed to parse standalone |
| `Expression` | full `ExpressionError` | Job creation hit an evaluation failure |
| `Compatibility` | `String` | Two fields disagree in a way that's not a single-pass rule |
| `UnsupportedSchema` | `String` | `specificationVersion: jobtemplate-2099-XX` |

The `#[non_exhaustive]` on the enum means downstream `match`
statements need a `_` arm.

## ValidationErrors — the structured list

```rust
// :88
pub struct ValidationError {
    pub path: Vec<PathElement>,
    pub message: String,
    pub detail: Option<ErrorDetail>,
}

pub struct ValidationErrors(Vec<ValidationError>);
```

Every entry is one problem. The fields:

- `path`: a `Vec<PathElement>` describing where in the YAML/JSON
  tree the error lives. `Field("steps")`, `Index(0)`, `Field("script")`,
  ... lets a UI or editor navigate to the offending line.
- `message`: a fully-formatted human-readable string, ready to
  display.
- `detail`: optional structured data with `summary` and `spans` for
  callers that want to render carets without parsing `message`.

```rust
pub enum PathElement {
    Field(String),
    Index(usize),
}

pub struct ErrorDetail {
    pub summary: String,
    pub spans: Vec<DiagnosticSpan>,
}

pub struct DiagnosticSpan {
    pub summary: String,
    pub source: String,
    pub start: usize,
    pub end: usize,
}
```

`DiagnosticSpan` is the byte-range pointer that powers caret
annotations:

```
Method 'upper' not found
  Param.Count.upper()
  ~~~~~~~~~~~~^~~~~~~
```

The `~~~~^~~` is rendered from `(start, end)` against `source`.

## Why accumulate instead of bail

This is the key UX choice. RFC 0005 §Technical Requirements demands
"Fail-Fast Errors" — meaning at *submission time*, not at runtime.
But within submission validation, the user wants to see *every*
problem at once so they can fix them in a single edit cycle.

The validator therefore takes `&mut ValidationErrors` and *appends*
entries instead of returning early:

```rust
// Idiom in every validation pass
fn validate_some_thing(x: &Thing, errors: &mut ValidationErrors) {
    if x.field_a < 0 {
        errors.push(ValidationError { path: ..., message: ..., detail: ... });
    }
    if x.field_b.len() > MAX_LEN {
        errors.push(ValidationError { ... });
    }
    // ... continues even if earlier checks failed
}
```

After all passes complete:

```rust
if !errors.is_empty() {
    return Err(ModelError::ModelValidation(errors));
}
Ok(())
```

A user with five typos sees five errors; a user with three bad
expressions sees all three.

## Bridging from ExpressionError

```rust
// :62
impl From<openjd_expr::SymbolTableError> for ModelError {
    fn from(e: openjd_expr::SymbolTableError) -> Self {
        ModelError::Expression(openjd_expr::ExpressionError::from(e))
    }
}

// :68
impl From<openjd_expr::FormatStringValidationError> for ModelError {
    fn from(e: openjd_expr::FormatStringValidationError) -> Self {
        ModelError::FormatStringError {
            message: e.message, input: Some(e.input),
            start: Some(e.start), end: Some(e.end),
        }
    }
}
```

These let `?` propagate naturally from the expression layer into the
model layer. A caller that wants to reach the original
`ExpressionErrorKind` walks back through:

```rust
if let ModelError::Expression(expr_err) = model_err {
    match expr_err.kind() { /* the structured kind */ }
}
```

## Exercise

```sh
# 1. Read three different validation passes to see the accumulation
#    pattern.
$EDITOR crates/openjd-model/src/template/validate_v2023_09/structure.rs
$EDITOR crates/openjd-model/src/template/validate_v2023_09/format_strings.rs
$EDITOR crates/openjd-model/src/template/validate_v2023_09/wrap_actions.rs

# 2. Construct a template with multiple errors and confirm the
#    validator surfaces all of them in one pass:
cat > /tmp/many.yaml <<'YAML'
specificationVersion: jobtemplate-2023-09
extensions: [EXPR]
name: "{{ Param.NoSuch }}"        # error 1: undefined symbol
parameterDefinitions:
  - { name: x, type: INT, default: 1 }   # error 2: lowercase name (must start uppercase)
  - { name: X, type: INT, default: 1 }
  - { name: X, type: INT, default: 2 }   # error 3: duplicate name
steps: []                                 # error 4: empty steps
YAML
./target/release/openjd-rs check /tmp/many.yaml
# Notice all four errors appear together.

# 3. Match on ModelError variants in your own caller code (or a test).
```

## What's next

[11 — Limits and defensive caps](./11-limits-and-defensive-caps.md).
The validation passes you just read are bounded by a set of constants
sprinkled through the codebase; doc 11 collects them all in one place.
