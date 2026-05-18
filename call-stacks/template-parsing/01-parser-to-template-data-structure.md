# 01 — Parser → template data structure

**Use case:** Given a YAML/JSON template file (or string), return the typed
`JobTemplate` / `EnvironmentTemplate` data structure with all validation
passes applied.

## Call stack

Entry path through `openjd-cli` for `openjd check <path>`:

1. **`main()`** — [openjd-rs/crates/openjd-cli/src/main.rs:101](../../../openjd-rs/crates/openjd-cli/src/main.rs#L101)
   clap dispatches `Commands::Check` to `check::execute`.

2. **`check::execute(args)`** — [openjd-rs/crates/openjd-cli/src/check.rs:21](../../../openjd-rs/crates/openjd-cli/src/check.rs#L21)
   reads the file, picks `DocumentType::Yaml`/`Json` from the extension,
   then calls `document_string_to_object` → `decode_job_template` (or
   `decode_environment_template`).

3. **`read_input_file(path)`** — [openjd-rs/crates/openjd-cli/src/common.rs:104](../../../openjd-rs/crates/openjd-cli/src/common.rs#L104)
   slurps the file to a `String`.

4. **`parse::document_string_to_object(...)`** — [openjd-rs/crates/openjd-model/src/template/parse.rs:41](../../../openjd-rs/crates/openjd-model/src/template/parse.rs#L41)
   runs `serde_saphyr` (YAML) or `serde_json` (JSON) into a generic
   `serde_json::Value`. Enforces `caller_limits.max_template_size`. This
   is the *parse* step.

5. **`parse::decode_job_template(value, ext, limits)`** — [openjd-rs/crates/openjd-model/src/template/parse.rs:185](../../../openjd-rs/crates/openjd-model/src/template/parse.rs#L185)
   extracts `specificationVersion`, dispatches by [`SpecificationRevision`](../../../openjd-rs/crates/openjd-model/src/template/parse.rs#L218),
   and `serde_json::from_value` materializes the typed [`JobTemplate`](../../../openjd-rs/crates/openjd-model/src/template/job_template.rs)
   struct. Then [`validate_extensions_list`](../../../openjd-rs/crates/openjd-model/src/template/parse.rs#L110)
   checks the `extensions:` list.

   *(Environment templates take the parallel branch [`decode_environment_template`](../../../openjd-rs/crates/openjd-model/src/template/parse.rs#L245),
   and `decode_template` at [parse.rs:307](../../../openjd-rs/crates/openjd-model/src/template/parse.rs#L307)
   auto-detects.)*

6. **`validate::validate_job_template(&jt, &ctx)`** — [openjd-rs/crates/openjd-model/src/template/validation/mod.rs:41](../../../openjd-rs/crates/openjd-model/src/template/validation/mod.rs#L41)
   revision-neutral dispatch.

7. **`validate_v2023_09::validate_job_template(&jt, ctx)`** — [openjd-rs/crates/openjd-model/src/template/validate_v2023_09/mod.rs:161](../../../openjd-rs/crates/openjd-model/src/template/validate_v2023_09/mod.rs#L161)
   runs all passes (limits, structure, FEATURE_BUNDLE_1, format-strings,
   TASK_CHUNKING, [`wrap_actions::validate_wrap_actions_job_template`](../../../openjd-rs/crates/openjd-model/src/template/validate_v2023_09/wrap_actions.rs#L110)).

8. Returns the populated `JobTemplate` back up to step 5's caller. The
   CLI throws it away after printing "passes validation"; library
   callers keep it.

## Library-only variant (no CLI)

Two function calls from a `&str` to a validated `JobTemplate`:

```rust
use openjd_model::parse::{self, DocumentType};
use openjd_model::CallerLimits;

let value = parse::document_string_to_object(
    yaml_str,
    DocumentType::Yaml,
    &CallerLimits::default(),
)?;
let jt = parse::decode_job_template(value, Some(&supported_exts), &CallerLimits::default())?;
```

## Why two stages

`document_string_to_object` is format-aware (YAML vs JSON) but
schema-blind — it produces a `serde_json::Value`. `decode_*_template` is
format-agnostic but schema-aware — it dispatches on
`specificationVersion`, calls `serde_json::from_value` to materialize
the typed struct, and runs the revision-specific validation pipeline.
This split lets a caller hold the untyped `Value` (e.g. to peek at
`specificationVersion` first) before committing to a typed decode.
