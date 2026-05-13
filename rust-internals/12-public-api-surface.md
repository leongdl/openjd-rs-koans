# 12 — Public API surface

## What this is

What each crate's `lib.rs` actually exposes to consumers, what's
deliberately `pub(crate)`, what's gated behind `#[cfg(test)]` or
feature flags, and which types are forward-compatible vs. stable.
After this doc you'll know which symbols you can rely on, which
might disappear, and where to add a new public type when the time
comes.

## Read these in order

- `crates/openjd-expr/src/lib.rs` — every `pub` and `pub use`.
- `crates/openjd-model/src/lib.rs` — same.
- `crates/openjd-sessions/src/lib.rs` — same.
- `crates/openjd-cli/src/main.rs` — what the binary actually calls.

## openjd-expr — exports

```rust
// crates/openjd-expr/src/lib.rs
pub mod default_library;
pub(crate) mod edit_distance;       // internal Damerau-Levenshtein
pub mod error;
pub mod eval;
pub mod format_string;
pub mod function_library;
pub mod functions;
pub mod path_mapping;
pub mod profile;
pub mod range_expr;
pub mod symbol_table;
pub mod types;
pub mod uri_path;
pub mod value;

pub use error::{ExpressionError, ExpressionErrorKind};
pub use eval::{
    EvalBuilder, EvalResult, ParsedExpression,
    DEFAULT_MEMORY_LIMIT, DEFAULT_OPERATION_LIMIT,
    MAX_EXPRESSION_DEPTH, MAX_PARSE_INPUT_LEN,
};
pub use format_string::escape_format_string;
pub use format_string::FormatString;
pub use format_string::FormatStringOptions;
pub use format_string::FormatStringValidationError;
pub use function_library::{EvalContext, FunctionLibrary};
pub use path_mapping::{PathFormat, PathMappingRule};
pub use profile::{ExprExtension, ExprProfile, ExprRevision, HostContext};
pub use range_expr::{RangeExpr, RangeExprError, MAX_RANGE_EXPR_CHUNKS};
pub use symbol_table::{
    SerializedSymbolTable, SymbolTable, SymbolTableError, MAX_SYMBOL_TABLE_ENTRIES,
};
pub use types::{ExprType, TypeCode};
pub use value::ExprValue;
```

The flat re-exports at the bottom mean callers can write
`openjd_expr::ExprValue` instead of `openjd_expr::value::ExprValue`.
Both work; the flat form is the canonical one.

`edit_distance` is `pub(crate)` because it's an implementation
detail of error suggestions. If you find yourself wanting to use it
externally, it's a sign the suggestion logic itself should move into
this crate.

## openjd-model — exports

```rust
// crates/openjd-model/src/lib.rs
pub mod error;
pub mod job;
pub(crate) mod template;             // submodules expose what's needed
pub mod capabilities;
pub mod types;

pub use job::create_job;
pub use job::step_dependency_graph;
pub use job::step_param_space;
pub use template::parse;
pub use template::{EnvironmentTemplate, JobParameterDefinition, JobTemplate};

// Re-exports from openjd-expr (mirrors Python package shape)
pub use openjd_expr::format_string;
pub use openjd_expr::format_string::FormatString;
pub use openjd_expr::symbol_table;
pub use openjd_expr::symbol_table::SymbolTable;

pub use error::ModelError;
pub use error::{DiagnosticSpan, ErrorDetail};
pub use error::{PathElement, ValidationError, ValidationErrors};
pub use job::create_job::{
    build_symbol_table, convert_environment, create_job, evaluate_let_bindings,
    merge_job_parameter_definitions, preprocess_job_parameters,
    MergedParameterDefinition, PathParameterOptions,
};
pub use parse::{
    decode_environment_template, decode_job_template, decode_template,
    DecodedTemplate, DocumentType,
};
pub use step_dependency_graph::StepDependencyGraph;
pub use step_param_space::StepParameterSpaceIterator;
pub use template::TaskParameterDefinition;
pub use types::{
    CallerLimits, DataFlow, EndOfLine, Extensions, FileType,
    JobParameterInputValues, JobParameterType, JobParameterValue, JobParameterValues,
    ModelExtension, ModelProfile, ObjectType, SpecificationRevision, TaskParameterSet,
    TaskParameterType, TaskParameterValue, TemplateSpecificationVersion,
    ValidationContext,
};
```

A few notable details:

- `template` is `pub(crate)`, but specific submodules are exposed
  via re-exports at the top level. This means the *layout* of the
  template module can change without breaking callers.
- The `pub use openjd_expr::format_string` and
  `pub use openjd_expr::symbol_table` are deliberate — Python's
  `openjd.model.FormatString` is the canonical name for that type
  across both implementations, and Rust mirrors the convention.
- `convert_environment`, `build_symbol_table`,
  `evaluate_let_bindings`, etc. are exposed because the CLI and
  external embedders need to drive job creation step-by-step (e.g.
  to inspect the symbol table at each layer). They're stable but
  noisy; most callers want the higher-level `create_job`.

## openjd-sessions — exports

```rust
// crates/openjd-sessions/src/lib.rs
pub mod action;
pub(crate) mod action_filter;
pub mod action_status;
pub(crate) mod cross_user_helper;
pub mod embedded_files;
pub mod error;
pub(crate) mod helper_binary;
pub mod let_bindings;
pub mod logging;
pub mod runner;
pub mod session;
pub mod session_user;
pub(crate) mod subprocess;
pub mod tempdir;
#[cfg(windows)]
pub mod win32;
#[cfg(windows)]
pub(crate) mod win32_locate;
#[cfg(all(windows, not(feature = "test-utils")))]
pub(crate) mod win32_permissions;
#[cfg(all(windows, feature = "test-utils"))]
pub mod win32_permissions;             // exposed only under test-utils

pub use openjd_expr::path_mapping;     // mirrors Python

pub use action::{ActionMessage, ActionResult, ActionState};
pub use action_status::ActionStatus;
pub use error::SessionError;
pub use logging::LogContent;
pub use openjd_expr::path_mapping::{PathFormat, PathMappingRule};
pub use runner::{CancelMethod, ScriptRunnerState};
pub use session::{EnvironmentIdentifier, Session, SessionConfig, SessionState};
#[cfg(windows)]
pub use session_user::BadCredentialsError;
#[cfg(unix)]
pub use session_user::PosixSessionUser;
pub use session_user::SessionUser;
#[cfg(windows)]
pub use session_user::WindowsSessionUser;
pub use subprocess::SubprocessResult;
pub use tempdir::StickyBitPolicy;
pub use tempdir::TempDir;
```

Three subtleties:

- **Cross-user helper modules are `pub(crate)`.** Callers go
  through `Session`'s public API, never directly into the helper
  binary IPC. Exposing those would couple downstream code to an
  internal protocol.
- **`win32_permissions` is exposed *only* under the `test-utils`
  feature.** Production builds keep it `pub(crate)`. This lets
  integration tests drive DACL setup without making the helper
  callable in production.
- **Platform-specific types appear behind `cfg`.** `PosixSessionUser`
  doesn't exist on Windows. `WindowsSessionUser` and
  `BadCredentialsError` don't exist on POSIX. Code that uses both
  needs cfg gates of its own.

## openjd-cli — what the binary calls

```rust
// crates/openjd-cli/src/main.rs (paraphrased)
fn main() {
    use openjd_model::{decode_job_template, JobTemplate};
    use openjd_model::create_job;
    use openjd_sessions::{Session, SessionConfig};
    // ...
}
```

The CLI uses only the flat re-exports. If you can write a CLI
command without reaching into a submodule, the public API is
correctly factored.

## Stable vs. unstable

| Pattern | Stability |
|---|---|
| `Type::with_profile(...)` | Stable — pinning the profile means consistent behaviour across upgrades |
| `Type::new(...)` | Deliberately unstable — uses "latest" profile, which changes |
| Bare struct construction with public fields | Stable as long as the struct doesn't grow `#[non_exhaustive]` |
| `#[non_exhaustive]` enum match | Need a `_` arm; new variants don't break SemVer |
| Crate-private modules and items | Free to reshape; no guarantees |
| `#[cfg(feature = "test-utils")]` items | Available only when feature is on |
| `#[cfg(unix)]` / `#[cfg(windows)]` items | Available only on the target platform |

## Mirror points with Python

The flat-re-export shape is intentional. `openjd-sessions-for-python`
re-exports `PathMappingRule` from `openjd.model._path_mapping`;
`openjd-rs` does the same with `pub use openjd_expr::path_mapping::PathMappingRule`.
This keeps doc 17's mapping table simple — for any Python name like
`openjd.sessions.PathMappingRule`, the Rust equivalent
`openjd_sessions::PathMappingRule` exists.

## Exercise

```sh
# 1. Build the docs and browse the public API.
cargo doc --no-deps --workspace --open

# 2. Find an item that's pub(crate) and ask: should this be public?
#    (Most should not. The pub(crate) gates exist for a reason.)

# 3. Walk the import list of one CLI command (e.g. check.rs) and
#    confirm every `use` resolves through a top-level re-export, not
#    a deep submodule path.

# 4. Run cargo doc with all features:
cargo doc --no-deps --workspace --features test-utils
# Compare against the default. The win32_permissions module appears
# under test-utils; everything else is identical.
```

## What's next

[13 — Concurrency and shared state](./13-concurrency-and-shared-state.md)
(if you haven't read it yet) — the next topic in the operational
track. Or jump to [14 — Parsing call stack](./14-parsing-call-stack.md)
to start the deep dives.
