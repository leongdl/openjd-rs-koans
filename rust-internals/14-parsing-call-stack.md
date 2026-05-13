# 14 — Parsing call stack

## What this is

Step-by-step from a string or file on disk down to a typed `JobTemplate`,
then to an instantiated `Job`, then to the lazy iterator over its task
parameter space — the "execution tree" the runtime consumes. Every row
of the call stack names a function, the file it's in, and the type
shape it produces.

If doc 06 is the *what*, this doc is the *how* — concrete enough that
you could re-implement a stage by reading only this and the linked
source.

## Read these in order

- `crates/openjd-cli/src/check.rs` — the CLI entry point, the smallest
  caller that drives the full pipeline.
- `crates/openjd-model/src/template/parse.rs` — `document_string_to_object`,
  `decode_template`, extension allowlist negotiation.
- `crates/openjd-model/src/template/job_template.rs` — the typed
  `JobTemplate` struct, `profile()` derivation.
- `crates/openjd-model/src/template/validate_v2023_09/mod.rs` — the
  semantic validation entry point.
- `crates/openjd-model/src/job/create_job/mod.rs` — instantiation of a
  template into a runnable `Job`.
- `crates/openjd-model/src/job/step_dependency_graph.rs` — topo-sort over
  step `dependencies:`.
- `crates/openjd-model/src/job/step_param_space.rs` — lazy cartesian
  product over `taskParameterDefinitions`.

## The pipeline at a glance

```
String (YAML/JSON bytes)        ── input
     │
     ▼ document_string_to_object()      [parse.rs]
serde_json::Value                ── untyped tree
     │
     ▼ decode_template()                [parse.rs]
DecodedTemplate { JobTemplate, … } ── typed, structurally valid
     │
     ▼ validate_v2023_09::validate()    [validate_v2023_09/mod.rs]
JobTemplate (semantically valid) ── ready for create_job
     │
     ▼ create_job()                     [job/create_job/mod.rs]
Job (instantiated)               ── parameters bound, paths mapped
     │
     ├─▶ StepDependencyGraph::build()   [step_dependency_graph.rs]
     │    StepDependencyGraph (DAG)    ── topo-sortable execution order
     │
     └─▶ StepParameterSpaceIterator::new(step)  [step_param_space.rs]
          Iterator<Item = TaskParameterSet>  ── the per-task stream
```

## Stage 1 — Bytes to `serde_json::Value`

```rust
// crates/openjd-model/src/template/parse.rs:41
pub fn document_string_to_object(
    document: &str,
    doc_type: DocumentType,
    caller_limits: &CallerLimits,
) -> Result<serde_json::Value, ModelError>
```

Three things happen:

1. **Size cap.** If `caller_limits.max_template_size` is set and exceeded,
   reject before parsing. The cap is enforced *before* allocating
   intermediate state.
2. **Format dispatch.** `DocumentType::Json` calls `serde_json::from_str`.
   `DocumentType::Yaml` calls `serde_saphyr::from_str_with_options`
   with `strict_booleans: true` and `max_depth: MAX_DOCUMENT_DEPTH`
   (= 128, matching `serde_json`'s hardcoded recursion limit).
3. **Shape check.** The result must be an object (key-value at the
   top). A bare scalar or list is a `DecodeValidation` error.

**Output:** `serde_json::Value::Object(...)` — an untyped tree.

Errors at this stage become `ModelError::DecodeValidation(String)`.

## Stage 2 — Typed deserialization

```rust
// crates/openjd-model/src/template/parse.rs (within decode_template)
let template: JobTemplate = serde_json::from_value(value)?;
```

Driven by `#[derive(serde::Deserialize)]` on `JobTemplate` and every
nested type. The crucial annotations:

- `#[serde(rename_all = "camelCase")]` — YAML/JSON uses camelCase,
  Rust uses snake_case.
- `#[serde(deny_unknown_fields)]` — typos like `parameterDefinitons`
  (missing `i`) become hard errors.
- Field-level `#[serde(rename = "$schema")]` for symbols Rust can't
  spell.

The `extensions` allowlist negotiation runs *before* `serde_json::from_value`:

```rust
// parse.rs:110
fn validate_extensions_list(
    template_exts: Option<&[ExtensionName]>,
    supported_extensions: Option<&[&str]>,
    errors: &mut ValidationErrors,
) -> Extensions
```

If the template lists `WRAP_ACTIONS` but the caller has not opted in,
that's an error here (and validation continues so the user sees every
problem in one pass).

**Output:** `JobTemplate { specification_version, extensions, name,
parameter_definitions, job_environments, steps }` — the
revision-independent shape from
`crates/openjd-model/src/template/job_template.rs`.

Errors at this stage become `ModelError::DecodeValidation(String)`
(serde errors) or accumulate into `ValidationErrors` (extension list
problems).

## Stage 3 — Semantic validation

`crates/openjd-model/src/template/validate_v2023_09/mod.rs` orchestrates
about a dozen passes, each in its own submodule:

| Pass | File | What it checks |
|---|---|---|
| Structure | `structure.rs` | Cardinality (`steps` non-empty), name uniqueness, "must define ≥1 lifecycle action," wrap-hook structural placement |
| Format strings | `format_strings.rs` | Every `{{...}}` parses, references existing symbols, type-checks under the right scope (Param / Task / Session / Env) |
| Wrap actions | `wrap_actions.rs` | RFC 0008: `WRAP_ACTIONS` extension gating, single-wrap-layer rule, `runOnHost` placement |
| Parameter constraints | `expr_parameters.rs` (and others) | `allowedValues` / `minLength` / `range` consistency, default-value validity |
| Step dependencies | dedicated pass | No cycles, references to existing steps |

Each pass takes the parsed template and a mutable
`ValidationErrors`, and *accumulates* problems instead of bailing on the
first. After all passes complete, `ValidationErrors::into_result()`
either returns `Ok(())` or `Err(ModelError::ModelValidation(errors))`.

This is the "Fail-Fast Errors" requirement from RFC 0005 §Technical
Requirements: a submitter sees every problem in one round-trip.

**Output:** the same `JobTemplate`, now confirmed semantically valid.

## Stage 4 — Job instantiation

```rust
// crates/openjd-model/src/lib.rs:33
pub use job::create_job::{
    build_symbol_table, convert_environment, create_job, evaluate_let_bindings,
    merge_job_parameter_definitions, preprocess_job_parameters,
    MergedParameterDefinition, PathParameterOptions,
};
```

`create_job` runs five sub-stages:

```rust
// crates/openjd-model/src/job/create_job/mod.rs (paraphrased)
pub fn create_job(
    template: &JobTemplate,
    parameter_values: JobParameterInputValues,
    ctx: &ValidationContext,
) -> Result<Job, ModelError> {
    // 1. Merge parameter defaults with caller-supplied values.
    let merged = merge_job_parameter_definitions(template, parameter_values)?;
    // 2. Apply path mapping, type coercion, allowedValues check.
    let preprocessed = preprocess_job_parameters(&merged, ctx)?;
    // 3. Build the symbol table seeded with Param.* and Job.Name.
    let mut symtab = build_symbol_table(template, &preprocessed)?;
    // 4. Evaluate StepTemplate `let` bindings, layered into the symtab.
    let with_let = evaluate_let_bindings(template, symtab)?;
    // 5. Convert environments + steps into instantiated forms.
    let environments = template.job_environments
        .as_deref().unwrap_or(&[])
        .iter().map(|e| convert_environment(e, &with_let)).collect()?;
    // ...
}
```

Two things to notice:

- **Format strings are *not* fully resolved here.** The job-creation
  pass evaluates `let` bindings and `Param.*`-only expressions, but
  any expression referencing `Task.Param.*`, `Session.*`, or
  `Env.File.*` stays as a `FormatString` and is resolved later on the
  worker host. This is the progressive evaluation model from RFC 0005.
- **Path mapping is applied here, not at runtime.** PATH parameter
  values are run through the session's path-mapping rules during
  `preprocess_job_parameters` so that downstream code sees concrete,
  worker-local paths. The unmapped originals stay accessible through
  `RawParam.<name>`.

**Output:** `Job` — an instantiated structure with parameter values
bound, environments materialised, and steps ready to iterate.

## Stage 5 — Step dependency graph

```rust
// crates/openjd-model/src/job/step_dependency_graph.rs
pub struct StepDependencyGraph { ... }

impl StepDependencyGraph {
    pub fn build(job: &Job) -> Result<Self, ModelError> { ... }
    pub fn topological_order(&self) -> Vec<&str> { ... }
    pub fn dependents_of(&self, step: &str) -> &[&str] { ... }
}
```

Linear scan of `job.steps`, building an adjacency list keyed by step
name. Cycle detection happens during the topo sort. The graph is
read-only after construction.

**Output:** `StepDependencyGraph` — execution order for the job's
steps.

## Stage 6 — Lazy task iteration

```rust
// crates/openjd-model/src/job/step_param_space.rs
pub struct StepParameterSpaceIterator { ... }

impl StepParameterSpaceIterator {
    pub fn new(step: &Step) -> Self { ... }
}

impl Iterator for StepParameterSpaceIterator {
    type Item = TaskParameterSet;
    // Cartesian product across taskParameterDefinitions, yielded one task at a time.
}
```

This is the "execution tree" — a stream of `TaskParameterSet`s, one
per task. It's an iterator, not a vec, because the parameter space
can be enormous (chunks of millions of frames) and the consumer only
wants one task at a time anyway.

The iterator is **lazy** — it doesn't allocate the full product. Each
`next()` advances a small internal odometer and produces the
next combination. Memory stays O(number of dimensions), not O(product).

## Cross-stage error model

| Stage | Error variant |
|---|---|
| Bytes to value | `ModelError::DecodeValidation(String)` |
| Typed deserialization | `ModelError::DecodeValidation(String)` |
| Extension negotiation | accumulates into `ValidationErrors` |
| Semantic validation | `ModelError::ModelValidation(ValidationErrors)` |
| Format string parse failures inside validation | `ValidationErrors` entries with `ErrorDetail` + `DiagnosticSpan` |
| Job creation expression failures | `ModelError::Expression(ExpressionError)` |
| Path mapping failures | `ModelError::Expression(...)` (wrapped) |

`ModelError::ModelValidation` is the only variant that carries a
*list* of problems. Everything else is a single error. When you write
caller code, prefer matching on the variant over parsing the `Display`
output.

## Exercise

```sh
# 1. Walk the pipeline yourself with a small template.
cat > /tmp/tiny.yaml <<'YAML'
specificationVersion: jobtemplate-2023-09
name: Tiny
parameterDefinitions:
  - name: Frame
    type: INT
    default: 1
steps:
  - name: Render
    parameterSpace:
      taskParameterDefinitions:
        - name: F
          type: INT
          range: "1-3"
    script:
      actions:
        onRun:
          command: echo
          args: ["{{Task.Param.F}}"]
YAML

# 2. Check it (stages 1-3).
./target/release/openjd-rs check /tmp/tiny.yaml

# 3. Run it (stages 1-6 + sessions runtime).
./target/release/openjd-rs run /tmp/tiny.yaml

# 4. Force each stage to fail individually:
#
#    Stage 1: corrupt YAML
sed -i.bak '1s/^/!!!/' /tmp/tiny.yaml
./target/release/openjd-rs check /tmp/tiny.yaml   # → DecodeValidation
mv /tmp/tiny.yaml.bak /tmp/tiny.yaml

#    Stage 2: unknown field
sed -i.bak 's/^name:/nayme:/' /tmp/tiny.yaml
./target/release/openjd-rs check /tmp/tiny.yaml   # → DecodeValidation (unknown field)
mv /tmp/tiny.yaml.bak /tmp/tiny.yaml

#    Stage 3: undefined symbol
sed -i.bak 's/Task.Param.F/Task.Param.Z/' /tmp/tiny.yaml
./target/release/openjd-rs check /tmp/tiny.yaml   # → ModelValidation with caret on the {{...}}
mv /tmp/tiny.yaml.bak /tmp/tiny.yaml
```

Each error class should look distinctly different — that's the payoff
for keeping the stages separate.

## What's next

[15 — Expression pipeline](./15-expr-pipeline.md). Stage 3's format-string
validation hands off to the expression layer; doc 15 traces what
happens inside that handoff.
