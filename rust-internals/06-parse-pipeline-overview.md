# 06 — Parse pipeline overview

## What this is

A short orientation to the four-phase pipeline that turns a string
into a typed, validated `JobTemplate`. Doc 14 traces the same path
in deep detail with line numbers and per-stage error mapping; this
doc is the executive summary, focused on the *why* of each phase.
Read this before doc 14 if you want the shape first.

## Read these in order

- `crates/openjd-model/src/template/parse.rs:1-100` — module docs,
  `DocumentType`, `MAX_DOCUMENT_DEPTH`, `document_string_to_object`.
- `crates/openjd-model/src/template/parse.rs:108-160` —
  `validate_extensions_list`.
- `crates/openjd-model/src/template/parse.rs` (rest) —
  `decode_template`, `decode_job_template`, `decode_environment_template`.
- `crates/openjd-model/src/template/validate_v2023_09/mod.rs` —
  the validation orchestrator.

## The four phases

```
                     ┌──────────────────────────────────────┐
   bytes (str)  ────►│ 1. structural parse                  │
                     │    serde_json::from_str | serde-saphyr│
                     └──────────────────────────────────────┘
                                       │
                                       ▼
                       serde_json::Value
                                       │
                     ┌──────────────────────────────────────┐
                     │ 2. typed deserialization             │
                     │    serde_json::from_value::<JT>()    │
                     │    deny_unknown_fields               │
                     └──────────────────────────────────────┘
                                       │
                                       ▼
                       JobTemplate (typed, structurally valid)
                                       │
                     ┌──────────────────────────────────────┐
                     │ 3. semantic validation               │
                     │    validate_v2023_09::validate(...)  │
                     │    accumulates ValidationErrors      │
                     └──────────────────────────────────────┘
                                       │
                                       ▼
                       JobTemplate (semantically valid)
                                       │
                     ┌──────────────────────────────────────┐
                     │ 4. job creation (later, on submit)   │
                     │    create_job(template, params, ctx) │
                     └──────────────────────────────────────┘
                                       │
                                       ▼
                       Job (instantiated)
```

## Why split this way

**Each phase produces a strictly more-meaningful artifact than the
phase before, and each phase has its own error class.** That
separation is the reason the validator can show *every* problem in
one round-trip instead of bailing on the first one.

| Phase | Input | Output | Errors look like |
|---|---|---|---|
| 1. Structural | bytes | untyped JSON tree | `DecodeValidation("not a valid YAML document...")` |
| 2. Typed | JSON tree | typed struct | `DecodeValidation("unknown field `parameterDefinitons`...")` |
| 3. Semantic | typed struct | validated struct | `ModelValidation` with caret-annotated `ValidationErrors` |
| 4. Job creation | validated struct + params | runnable `Job` | `ModelError::Expression(...)`, `ModelError::Compatibility(...)` |

If the structural phase fails, you don't even reach the typed phase.
If typed fails, you don't reach semantic. But within phase 3, all
passes run and accumulate errors — one bad expression doesn't hide
five other bad fields.

## Phase 1 — structural parse

```rust
pub fn document_string_to_object(
    document: &str,
    doc_type: DocumentType,        // Json or Yaml
    caller_limits: &CallerLimits,
) -> Result<serde_json::Value, ModelError>
```

- Enforces `caller_limits.max_template_size` *before* parsing.
- For YAML, uses `serde-saphyr` with `strict_booleans` (so `yes`/`no`
  are strings, not bools) and `max_depth: 128`.
- For JSON, uses `serde_json` (which has its own hardcoded depth
  limit at 128, matching).
- Rejects scalar or list documents — must be a top-level object.

## Phase 2 — typed deserialization

`serde_json::from_value::<JobTemplate>(value)` runs the derived
`Deserialize` impl. Three things this catches:

- Missing required fields (`steps` not present).
- Unknown fields (`#[serde(deny_unknown_fields)]`).
- Type mismatches (`extensions: 5` instead of a list).

After this phase, every field you read from the struct is
type-correct. You can write `template.steps[0].name` without
worrying about whether `name` is missing.

## Phase 3 — semantic validation

Roughly a dozen passes under
`crates/openjd-model/src/template/validate_v2023_09/`, each in its
own file and each focused on one concern:

| File | What it checks |
|---|---|
| `structure.rs` | `steps` non-empty, name uniqueness, "must define ≥1 lifecycle action," wrap-hook structural placement |
| `format_strings.rs` | every `{{...}}` parses, references resolve, types check (per scope) |
| `wrap_actions.rs` | RFC 0008 single-layer rule, `WRAP_ACTIONS` extension gating |
| `expr_parameters.rs` | EXPR-extended parameter type constraints |
| `parameters.rs` | base parameter constraints (`allowedValues`, `minLength`, default validity) |
| (more) | step-dependency cycle detection, host requirements, etc. |

Each pass takes `&mut ValidationErrors` and *appends* problems.
After all passes finish, the orchestrator returns
`ValidationErrors::into_result()` — `Ok(())` if empty,
`Err(ModelError::ModelValidation(errors))` otherwise.

## Phase 4 — job creation

This is when caller-supplied parameter values arrive and the template
is *instantiated* into a runnable `Job`. Path mapping is applied,
`let` bindings are evaluated, environments are materialised, and the
`Job` is ready for the dependency graph + parameter-space iteration.

Doc 14 has the full sub-stage breakdown.

## Where each phase produces its error

| Phase | Module |
|---|---|
| 1 | `parse::document_string_to_object` |
| 2 | `parse::decode_template` (calls `serde_json::from_value`) |
| 3 | `validate_v2023_09::validate` |
| 4 | `job::create_job::*` |

The four corresponding `ModelError` variants are listed in doc 10.

## Exercise

```sh
# 1. Force each phase to fail in turn:
#
# Phase 1: corrupt YAML
echo '!!!corrupt!!!' | ./target/release/openjd-rs check /dev/stdin
# → DecodeValidation
#
# Phase 2: unknown field
cat > /tmp/p.yaml <<'YAML'
specificationVersion: jobtemplate-2023-09
nayme: Wrong
steps: []
YAML
./target/release/openjd-rs check /tmp/p.yaml
# → DecodeValidation (unknown field `nayme`)
#
# Phase 3: undefined symbol
cat > /tmp/p.yaml <<'YAML'
specificationVersion: jobtemplate-2023-09
name: P
steps:
  - name: S
    script:
      actions:
        onRun:
          command: "echo"
          args: ["{{Param.NoSuch}}"]
YAML
./target/release/openjd-rs check /tmp/p.yaml
# → ModelValidation, with caret on the {{...}}.

# 2. Compare the formatted error text from each phase. Notice that
#    phases 1 and 2 are single-line; phase 3 has the structured
#    caret + path layout.
```

## What's next

[07 — Expression parsing and keyword rewrite](./07-expression-parsing-and-keyword-rewrite.md).
You've seen the pipeline shape; doc 07 zooms into the trick that
makes `Param.if` (a Python keyword!) parse through ruff.
