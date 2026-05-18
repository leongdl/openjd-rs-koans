# 02 — Parser → environment template data structure

**Use case:** Given a YAML/JSON environment template file (or string),
return the typed `EnvironmentTemplate` data structure with all
validation passes applied.

Companion to [01 — Parser → template data structure](01-parser-to-template-data-structure.md),
which covers job templates. The two paths share the front half (file
read + YAML/JSON parse) and diverge at the decode and validation stages.

## Where the paths diverge

| Step | Job template | Environment template |
| --- | --- | --- |
| 1. CLI main | same | same |
| 2. CLI dispatch | `decode_job_template` branch | `decode_environment_template` branch |
| 3. file read | same | same |
| 4. YAML/JSON parse | same | same |
| 5. typed decode | `decode_job_template` | `decode_environment_template` (**no `caller_limits` param**) |
| 6. validation dispatch | `validate_job_template` (`pub(crate)`) | `validate_environment_template` (`pub`) |
| 7. v2023_09 passes | limits, structure, FEATURE_BUNDLE_1, format strings, TASK_CHUNKING, WRAP_ACTIONS | env-param cap, script-or-variables presence, `validate_single_environment`, WRAP_ACTIONS |

The job-template pass set in step 7 is a strict superset of the env
template's: env templates skip `limits::enforce_limits`,
`feature_bundle_1`, `format_strings`, and `task_chunking` entirely.

## Call stack

1. **`main()`** — [openjd-rs/crates/openjd-cli/src/main.rs:101](../../../openjd-rs/crates/openjd-cli/src/main.rs#L101)
   clap dispatches `Commands::Check`.

2. **`check::execute(args)`** — [openjd-rs/crates/openjd-cli/src/check.rs:53](../../../openjd-rs/crates/openjd-cli/src/check.rs#L53)
   the env-template branch (`v.is_environment_template()`) calls
   `decode_environment_template` instead of `decode_job_template`.

3. **`read_input_file(path)`** — [openjd-rs/crates/openjd-cli/src/common.rs:104](../../../openjd-rs/crates/openjd-cli/src/common.rs#L104)
   slurps the file to a `String`.

4. **`parse::document_string_to_object(...)`** — [openjd-rs/crates/openjd-model/src/template/parse.rs:41](../../../openjd-rs/crates/openjd-model/src/template/parse.rs#L41)
   `serde_saphyr` (YAML) / `serde_json` (JSON) → `serde_json::Value`.
   `caller_limits.max_template_size` is enforced here.

5. **`parse::decode_environment_template(value, ext)`** — [openjd-rs/crates/openjd-model/src/template/parse.rs:245](../../../openjd-rs/crates/openjd-model/src/template/parse.rs#L245)
   extracts `specificationVersion`, rejects job-template versions,
   dispatches on [`SpecificationRevision`](../../../openjd-rs/crates/openjd-model/src/template/parse.rs#L275),
   and `serde_json::from_value` materializes the typed [`EnvironmentTemplate`](../../../openjd-rs/crates/openjd-model/src/template/environment_template.rs)
   struct. [`validate_extensions_list`](../../../openjd-rs/crates/openjd-model/src/template/parse.rs#L110)
   checks the `extensions:` list (shared with the job-template path).

6. **`validate::validate_environment_template(&et, &ctx)`** — [openjd-rs/crates/openjd-model/src/template/validation/mod.rs:54](../../../openjd-rs/crates/openjd-model/src/template/validation/mod.rs#L54)
   revision-neutral dispatch (note: `pub`, unlike the `pub(crate)`
   job-template sibling — env-template validation is part of the
   library's public surface).

7. **`validate_v2023_09::validate_environment_template(&et, ctx)`** — [openjd-rs/crates/openjd-model/src/template/validate_v2023_09/mod.rs:191](../../../openjd-rs/crates/openjd-model/src/template/validate_v2023_09/mod.rs#L191)
   runs the env-template-specific passes:
   - `parameterDefinitions` empty/cap/duplicate checks (`limits.max_env_template_param_count`,
     intentionally not scaled by FEATURE_BUNDLE_1 in 2023-09)
   - `environment` must define at least one of `script` or `variables`
   - [`structure::validate_single_environment`](../../../openjd-rs/crates/openjd-model/src/template/validate_v2023_09/structure.rs)
   - [`wrap_actions::validate_wrap_actions_environment_template`](../../../openjd-rs/crates/openjd-model/src/template/validate_v2023_09/wrap_actions.rs#L167)

8. Returns the populated `EnvironmentTemplate`.

## Library-only variant

```rust
use openjd_model::parse::{self, DocumentType};
use openjd_model::CallerLimits;

let value = parse::document_string_to_object(
    yaml_str,
    DocumentType::Yaml,
    &CallerLimits::default(),
)?;
let et = parse::decode_environment_template(value, Some(&supported_exts))?;
```

Note `decode_environment_template` does *not* take `&CallerLimits` —
the only limit applied to env templates is the document-size cap
already enforced inside `document_string_to_object`.

## Auto-detect

If you don't know whether a document is a job or env template,
[`decode_template`](../../../openjd-rs/crates/openjd-model/src/template/parse.rs#L307)
peeks at `specificationVersion` and routes to the matching decoder,
returning a `DecodedTemplate::Job(_)` or `DecodedTemplate::Environment(_)`.
