# 03 — Profile, revision, extension, host context

## What this is

The single concept that demystifies "what can this expression do?" An
`ExprProfile` is the tuple `(revision, extensions, host_context)` that
gates which functions, operators, and types are visible to a parser
and evaluator. Understanding it removes a category of confusion:
why some expressions parse fine in one context and fail in another,
why `apply_path_mapping()` only works at runtime, why the function
library has a global cache.

## Read these in order

- `crates/openjd-expr/src/profile.rs:1-80` — module docs, the three
  axes: `ExprRevision`, `ExprExtension`, `HostContext`.
- `crates/openjd-expr/src/profile.rs` (rest) — `ExprProfile`,
  `ProfileKey`, `with_extensions`, `with_host_context`.
- `crates/openjd-expr/src/default_library.rs:15-30` — the global
  `PROFILE_CACHE`.
- `crates/openjd-expr/src/default_library.rs:80-150` —
  `FunctionLibrary::for_profile`.
- `crates/openjd-model/src/template/job_template.rs:59-92` —
  `JobTemplate::profile()` derives an `ExprProfile` from the template
  itself.

## The three axes

```rust
// crates/openjd-expr/src/profile.rs:46
#[non_exhaustive]
pub enum ExprRevision {
    V2026_02,        // first revision to define the expression language (RFC 0005)
}

#[non_exhaustive]
pub enum ExprExtension {
    // empty today — RFC 0005 was the bootstrap; the "EXPR" gate in openjd-model
    // controls whether expressions exist at all, not which ones are registered.
}

pub enum HostContext {
    None,                                  // submission-time: no path mapping
    WithRules(Arc<Vec<PathMappingRule>>),  // runtime: path mapping available
}
```

| Axis | What it controls |
|---|---|
| **A — revision** | Which base functions and operators exist. New revisions can introduce or modify operators (e.g. add `+=` someday). |
| **B — extensions** | Which add-on functions exist, on top of the base revision. Today empty; reserved for the future. |
| **C — host state** | Whether host-context functions like `apply_path_mapping()` have *real* implementations or are stubs that error. |

A fourth axis (D — scope-specific symbol availability, e.g. `Task.*`
inside `onWrapTaskRun`) is handled by the *symbol table*, not the
profile. That's why the symbol table is the other input to evaluation
(doc 02): the two are orthogonal.

## ExprProfile

```rust
pub struct ExprProfile {
    revision: ExprRevision,
    extensions: HashSet<ExprExtension>,
    host_context: HostContext,
}

impl ExprProfile {
    pub fn new(revision: ExprRevision) -> Self;
    pub fn latest() -> Self;                          // unstable, every extension on
    pub fn with_extensions(self, ext: HashSet<ExprExtension>) -> Self;
    pub fn with_host_context(self, hc: HostContext) -> Self;
    pub fn key(&self) -> ProfileKey;                  // for caching
}
```

The two construction patterns:

| Pattern | When |
|---|---|
| `ExprProfile::latest()` | Tests, prototypes, ad-hoc parsing. **Deliberately unstable** — the meaning of "latest" changes when the crate adds revisions or extensions. |
| `ExprProfile::new(revision).with_extensions(set)` | Production. Pinning the revision means the same template parses identically across crate upgrades. |

The same dichotomy appears on `ParsedExpression::new` vs
`with_profile`, and on `FormatString::new` vs `with_profile`. The
`new` form is convenience; the `with_profile` form is what services
should use.

## The function library cache

```rust
// crates/openjd-expr/src/default_library.rs:27
static PROFILE_CACHE:
    LazyLock<Mutex<HashMap<ProfileKey, Arc<FunctionLibrary>>>> =
    LazyLock::new(|| Mutex::new(HashMap::new()));

// :104
impl FunctionLibrary {
    pub fn for_profile(profile: &ExprProfile) -> Arc<FunctionLibrary> {
        let key = profile.key();
        if let Some(cached) = PROFILE_CACHE.lock().unwrap().get(&key) {
            return Arc::clone(cached);
        }
        // Build the library, register every function for this profile,
        // then insert into the cache.
        ...
    }
}
```

`ProfileKey` is the rules-independent slice of an `ExprProfile`. Two
profiles that differ only in their `Arc<Vec<PathMappingRule>>` share
a cache entry. This matters because each task may have its own
mapping rules but the function table is identical.

The global mutex is documented in doc 13 as the *only* one in the
codebase outside test utilities. The hot path is one map lookup +
one `Arc::clone`.

## How profiles are derived from templates

```rust
// crates/openjd-model/src/template/job_template.rs:59
pub fn profile(&self) -> crate::ModelProfile {
    let revision = TemplateSpecificationVersion::from_str(&self.specification_version)
        .map(|v| v.revision())
        .unwrap_or(SpecificationRevision::V2023_09);
    let mut exts = Extensions::new();
    if let Some(list) = &self.extensions {
        for e in list {
            if let Ok(known) = ModelExtension::from_str(e.as_str()) {
                exts.insert(known);
            }
        }
    }
    ModelProfile::new(revision).with_extensions(exts)
}
```

Two layers of profile types live in the workspace:

- `ModelProfile` (in `openjd-model`) — covers spec-level extensions
  like `EXPR`, `FEATURE_BUNDLE_1`, `TASK_CHUNKING`, `WRAP_ACTIONS`,
  `REDACTED_ENV_VARS`.
- `ExprProfile` (in `openjd-expr`) — covers expression-language axes.

`ModelProfile::to_expr_profile()` translates between them. A template
that lists `EXPR` in `extensions:` produces a `ModelProfile` with
the EXPR extension on, which converts to an `ExprProfile` at the
revision the EXPR extension was introduced (`V2026_02`).

## Host context: why this matters for `apply_path_mapping`

```rust
pub enum HostContext {
    None,
    WithRules(Arc<Vec<PathMappingRule>>),
}
```

When the function library is built with `HostContext::None`, the
entry for `apply_path_mapping` is a stub that always raises an
error: "available only in host context." When the library is built
with `WithRules(rules)`, the entry calls into the actual mapping
logic with those rules.

This is the mechanism that makes RFC 0005 §Host-Context Function
Availability work. The validator (running at submission time, no
host context) sees the stub and rejects expressions that call
`apply_path_mapping`. The session runner (running on a worker, with
real rules) sees the real implementation.

## Exercise

```sh
# 1. Run the profile tests.
cargo test -p openjd-expr profile

# 2. Confirm the cache works. Run the same template through `check`
#    twice and verify (in profiling, or via tracing) that the second
#    invocation does not rebuild the function library.
./target/release/openjd-rs check sample.yaml
./target/release/openjd-rs check sample.yaml
# In a long-running service, the cache hit ratio should approach 100%.

# 3. Try to call apply_path_mapping at submission time:
./target/release/openjd-rs check <<'YAML'
specificationVersion: jobtemplate-2023-09
extensions: [EXPR]
parameterDefinitions:
  - { name: P, type: STRING, default: "/x" }
name: "{{ Param.P.apply_path_mapping() }}"
steps:
  - name: S
    script:
      actions:
        onRun: { command: "echo", args: [] }
YAML
# Expected: an error noting that apply_path_mapping is host-context only.
```

## What's next

[04 — Format string segments](./04-format-string-segments.md). Now you
know what input `ExprProfile` controls and what comes out of the
function library; doc 04 shows how the outermost layer (format strings)
splits literal text from `{{ ... }}` interpolations.
