# 05 — Model types and template shape

## What this is

The vocabulary the model layer uses to talk about templates:
`SpecificationRevision`, `TemplateSpecificationVersion`, `Extensions`,
`ModelExtension`, `ValidationContext`, `CallerLimits`. Plus the actual
shape of a `JobTemplate` — the struct serde populates from YAML/JSON.
After this doc you'll know what every field on a parsed template
means and how to derive the profile that controls validation.

## Read these in order

- `crates/openjd-model/src/types.rs:1-200` — `FileType`, `EndOfLine`,
  `ObjectType`, `DataFlow`, `SpecificationRevision`,
  `TemplateSpecificationVersion`.
- `crates/openjd-model/src/types.rs` (rest) — `Extensions`,
  `ModelExtension`, `ValidationContext`, `CallerLimits`,
  `JobParameterType`, `TaskParameterType`.
- `crates/openjd-model/src/template/mod.rs` — module re-exports.
- `crates/openjd-model/src/template/job_template.rs` — the
  `JobTemplate` struct and its `profile()` method.
- `crates/openjd-model/src/template/environment_template.rs` — the
  parallel structure for environment templates.

## SpecificationRevision and TemplateSpecificationVersion

```rust
// crates/openjd-model/src/types.rs:99
#[non_exhaustive]
pub enum SpecificationRevision {
    V2023_09,
}

// :119
#[non_exhaustive]
pub enum TemplateSpecificationVersion {
    JobTemplate2023_09,
    EnvironmentTemplate2023_09,
}
```

`SpecificationRevision` is the year-month tag for the *spec* itself
(today only `2023-09`). `TemplateSpecificationVersion` is the
*template kind* tag, which includes both the document type
(`jobtemplate-2023-09`, `environment-2023-09`) and the revision.

The conversion from a string in YAML to one of these enums runs in
`FromStr`:

```rust
let v = TemplateSpecificationVersion::from_str("jobtemplate-2023-09")?;
let revision = v.revision();
```

When future revisions land (`2027-XX`, etc.), they appear here as new
variants. `#[non_exhaustive]` keeps adding them backward-compatible.

## Extensions and ModelExtension

```rust
// :270
#[non_exhaustive]
pub enum ModelExtension {
    Expr,
    FeatureBundle1,
    TaskChunking,
    WrapActions,           // RFC 0008
    RedactedEnvVars,
}

pub struct Extensions(HashSet<ModelExtension>);
```

`Extensions` is the set the template lists in its `extensions:`
field. Insertion is order-independent; lookup is `O(1)`.

The string forms expected in YAML are:

| Variant | YAML string |
|---|---|
| `Expr` | `EXPR` |
| `FeatureBundle1` | `FEATURE_BUNDLE_1` |
| `TaskChunking` | `TASK_CHUNKING` |
| `WrapActions` | `WRAP_ACTIONS` |
| `RedactedEnvVars` | `REDACTED_ENV_VARS` |

`ModelExtension::from_str` does the parsing; the validator uses
`Extensions::contains(ModelExtension::Expr)` to gate the EXPR-only
features.

## ValidationContext and CallerLimits

```rust
pub struct ValidationContext {
    profile: ModelProfile,
    caller_limits: CallerLimits,
    supported_extensions: Option<Vec<&'static str>>,
}

pub struct CallerLimits {
    pub max_template_size: Option<usize>,
    pub max_format_string_length: Option<usize>,
    pub max_format_string_segments: Option<usize>,
    // ... etc
}
```

`ValidationContext` is the "what does the caller permit?" knob
threaded through every validation function. Two parts:

- **`profile`**: revision + extension set. Determines which features
  the validator accepts.
- **`caller_limits`**: tighter caps a service can apply on top of the
  built-in defensive caps. A service that only ever sees small
  templates can lower `max_template_size` from 1 MB to 64 KB; the
  built-in caps still hold as a hard ceiling above whatever the
  caller chooses.

The convenience constructor `ValidationContext::from_profile(profile)`
uses default (i.e. unlimited beyond the built-in caps) caller limits.

## ModelProfile vs ExprProfile

Two profile types sit in the workspace:

| Type | Crate | Axes |
|---|---|---|
| `ModelProfile` | `openjd-model` | revision, model extensions (EXPR, FEATURE_BUNDLE_1, ...) |
| `ExprProfile` | `openjd-expr` | expression revision, expression extensions, host context |

`ModelProfile::to_expr_profile()` translates between them. Doc 03
covers the `ExprProfile` side.

## The JobTemplate struct

```rust
// crates/openjd-model/src/template/job_template.rs:13
#[derive(Debug, Clone, Deserialize)]
#[serde(rename_all = "camelCase", deny_unknown_fields)]
pub struct JobTemplate {
    pub specification_version: String,
    #[serde(rename = "$schema")]
    pub schema: Option<String>,
    pub extensions: Option<Vec<ExtensionName>>,
    pub name: FormatString,
    pub description: Option<Description>,
    pub parameter_definitions: Option<Vec<JobParameterDefinition>>,
    pub job_environments: Option<Vec<Environment>>,
    pub steps: Vec<StepTemplate>,
}
```

Every field maps directly to its YAML/JSON counterpart, with serde
handling the camelCase ↔ snake_case translation. `deny_unknown_fields`
turns typos like `parameterDefinitons` into hard errors.

The notable fields:

- `specification_version: String` — kept as a string here and parsed
  to `TemplateSpecificationVersion` later. This is so a template
  with a future spec version can still deserialize and produce a
  meaningful "unsupported version" error.
- `extensions: Option<Vec<ExtensionName>>` — `ExtensionName` is a
  thin newtype that enforces the `[A-Z_][A-Z0-9_]*` shape but
  doesn't try to validate against the known set; that happens in
  validation.
- `name: FormatString` — yes, the job *name* is itself a format
  string. So is every command, arg, and description.
- `steps: Vec<StepTemplate>` — required to be non-empty (validated
  in `structure.rs`).

## EnvironmentTemplate

```rust
// crates/openjd-model/src/template/environment_template.rs (paraphrased)
#[derive(Debug, Clone, Deserialize)]
#[serde(rename_all = "camelCase", deny_unknown_fields)]
pub struct EnvironmentTemplate {
    pub specification_version: String,
    pub schema: Option<String>,
    pub extensions: Option<Vec<ExtensionName>>,
    pub parameter_definitions: Option<Vec<JobParameterDefinition>>,
    pub environment: Environment,
}
```

A standalone environment template (the kind a queue manager loads
once and applies to many jobs). The structure mirrors `JobTemplate`
but contains a single `Environment` instead of a list of steps.

`Environment` itself contains the `<EnvironmentActions>` block where
RFC 0008's `onWrapEnter`, `onWrapTaskRun`, `onWrapExit` live.

## `profile()` and `default_validation_context()`

```rust
impl JobTemplate {
    pub fn profile(&self) -> ModelProfile;

    pub fn default_validation_context(&self) -> ValidationContext {
        ValidationContext::from_profile(self.profile())
    }
}
```

These are the standard "do what the template says" derivations. A
service that wants to *override* the template's claimed extensions
(strip EXPR regardless of what the template asked for, for example)
builds a `ValidationContext` explicitly and ignores these helpers.

## Exercise

```sh
# 1. Read the model types tests.
cargo test -p openjd-model types::tests

# 2. Find every place a `ModelExtension` variant is checked.
grep -rn "ModelExtension::Expr\b" crates/openjd-model/src/
grep -rn "ModelExtension::WrapActions\b" crates/openjd-model/src/
# Notice the gating pattern: every EXPR feature has exactly one
# `extensions.contains(ModelExtension::Expr)` check.

# 3. Construct a minimal template and confirm `profile()` returns
#    the expected revision + extensions:
cat > /tmp/m.yaml <<'YAML'
specificationVersion: jobtemplate-2023-09
extensions: [EXPR, WRAP_ACTIONS]
name: M
steps:
  - name: S
    script:
      actions:
        onRun: { command: "echo", args: [] }
YAML
./target/release/openjd-rs check /tmp/m.yaml --json | jq '.profile'
# (Or read the field through Rust API in a unit test.)
```

## What's next

[06 — Parse pipeline overview](./06-parse-pipeline-overview.md). You
now know the types the parser is targeting; doc 06 walks through the
four-phase pipeline that produces them.
