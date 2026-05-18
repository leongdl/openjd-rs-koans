# 03 — Expression block end-to-end (string → AST → resolved value)

**Use case:** A user writes `{{ Param.Frame * 2 + len(Task.Param.Names) }}`
inside a template field. Trace it from raw bytes on disk all the way to a
runtime-resolved value.

This is the deep dive that ties together [01](01-parser-to-template-data-structure.md)
(template decode) and the runtime side. Expression blocks have **three
distinct evaluation moments**, each on a different symbol table and a
different `HostContext`:

| Moment | Where | Symtab type | Library | What's being checked / produced |
| --- | --- | --- | --- | --- |
| 1. **Decode** (parse-only) | `serde::Deserialize` for `FormatString` | — | — | Lex into segments, parse each `{{…}}` to AST, collect symbols/functions |
| 2. **Validate** (type-check) | `validate_format_strings` pass | unresolved values | `template_lib` / `host_lib` | Every accessed symbol is defined; `apply_path_mapping` only used in host scope |
| 3. **Create-job** (resolve) | `create_job` → `instantiate_step` | concrete values | `template_lib` (no host) | Job-name, step-name, host requirements, param-space ranges, let bindings |
| 4. **Session runtime** (resolve) | `Session::run_action` etc. | concrete + Session.* + Task.* | `host_lib` (with rules) | Command, args, timeout, env-var values |

Moment 1 happens once when `serde_json::from_value` hits a
`FormatString` field. Moments 2–4 happen with separate symbol tables;
the same parsed AST is reused.

## Stage 1 — Lex into segments (decode time)

When `serde_json::from_value(template)` materializes a typed
`JobTemplate` ([parse.rs:221](../../../openjd-rs/crates/openjd-model/src/template/parse.rs#L221)),
every field whose Rust type is `FormatString` triggers
`FormatString`'s custom `Deserialize` impl:

1. **`<FormatString as Deserialize>::deserialize`** — [openjd-rs/crates/openjd-expr/src/format_string.rs:346](../../../openjd-rs/crates/openjd-expr/src/format_string.rs#L346)
   `FsVisitor` accepts strings, ints, floats, bools (all stringified)
   and routes to `FormatString::new`.

2. **`FormatString::new(input)`** — [format_string.rs:66](../../../openjd-rs/crates/openjd-expr/src/format_string.rs#L66)
   convenience constructor that calls `with_profile(input, &ExprProfile::latest())`.

3. **`FormatString::with_profile(input, profile)`** — [format_string.rs:76](../../../openjd-rs/crates/openjd-expr/src/format_string.rs#L76)
   enforces `MAX_FORMAT_STRING_LEN` (1 MB) and `MAX_FORMAT_STRING_SEGMENTS`
   (1,000), then calls `parse_segments`.

4. **`parse_segments(input, profile)`** — [format_string.rs:462](../../../openjd-rs/crates/openjd-expr/src/format_string.rs#L462)
   the hand-rolled scanner. Walks the string finding `{{` / `}}`
   pairs, errors on stray closing braces or unbalanced opens, and
   for each `{{...}}` calls `ParsedExpression::with_profile(et, profile)`
   on the trimmed inner expression.

5. **`ParsedExpression::with_profile(expr, profile)`** — [openjd-rs/crates/openjd-expr/src/eval/parse.rs:133](../../../openjd-rs/crates/openjd-expr/src/eval/parse.rs#L133)
   trims, length-caps (`MAX_PARSE_INPUT_LEN` = 64 KB), and either
   parses on the current thread (≤ 200 chars: `FAST_PATH_INPUT_LEN`)
   or spawns a stack-enlarged worker thread (32 MB stack) so the
   recursive-descent parser can't overflow.

6. **`parse_inner(...)`** — [eval/parse.rs:489](../../../openjd-rs/crates/openjd-expr/src/eval/parse.rs#L489)
   wraps multi-line input in parens, calls `ruff_python_parser::parse_expression`,
   then on syntax error tries the **keyword-after-dot rewrite** (`Param.if` →
   `Param.xf`) and retries. On success:
   - `validate_structure` — depth ≤ `MAX_EXPRESSION_DEPTH` (64), profile-gated syntax features
   - `collect_symbols` — `accessed_symbols`, `called_functions`, `local_bindings`
   - `check_comprehension_shadowing` — nested `for x in ... for x in ...` rejected
   - Returns a `ParsedExpression { ast, expr, keyword_renames, accessed_symbols, called_functions, local_bindings }`.

7. Back in `parse_segments`: each parsed expression becomes a
   `Segment::Expression { start, end, parsed }` in the `FormatString`'s
   `Vec<Segment>`. Literal text becomes `Segment::Literal(s)`.

The `JobTemplate` now holds parsed ASTs in every format-string field.
**No symbol resolution has happened yet** — the only thing checked is
syntactic well-formedness and structural depth.

## Stage 2 — Type-check against unresolved symtabs (validate time)

Driven by step 7 of koan [01](01-parser-to-template-data-structure.md),
specifically the format-strings pass:

1. **`validate_format_strings(jt, ctx, errors)`** — [validate_v2023_09/format_strings.rs:331](../../../openjd-rs/crates/openjd-model/src/template/validate_v2023_09/format_strings.rs#L331)
   builds two `ExprProfile`s: `template_profile` (`HostContext::None`)
   for template-scope strings and `host_profile` (`HostContext::Unresolved`)
   for session/task scopes. Two corresponding `FunctionLibrary`s are built
   via `FunctionLibrary::for_profile`.

2. Per scope, it builds a `SymbolTable` whose entries are
   `ExprValue::unresolved(T)` — names exist with the right type but
   no value:
   - **Template scope** ([format_strings.rs:60](../../../openjd-rs/crates/openjd-model/src/template/validate_v2023_09/format_strings.rs#L60)) — `Param.*` (non-PATH), `RawParam.*` (path → STRING)
   - **Session scope** ([format_strings.rs:95](../../../openjd-rs/crates/openjd-model/src/template/validate_v2023_09/format_strings.rs#L95)) — adds `Session.*`, `Env.File.*`, optionally `Step.Name`, `Job.Name`
   - **Task scope** ([format_strings.rs:152](../../../openjd-rs/crates/openjd-model/src/template/validate_v2023_09/format_strings.rs#L152)) — adds `Task.Param.*`, `Task.RawParam.*`, `Task.File.*`
   - **Wrap-hook scope** — clones session symtab and adds `WrappedAction.*`
     plus `Env.Wrapped.Name` (only for onWrapEnter/onWrapExit)
     ([format_strings.rs:1010, 1025](../../../openjd-rs/crates/openjd-model/src/template/validate_v2023_09/format_strings.rs#L1010))

3. **`validate_fs(fs, symtab, lib, path, errors)`** — [format_strings.rs:251](../../../openjd-rs/crates/openjd-model/src/template/validate_v2023_09/format_strings.rs#L251)
   skips literals, then calls `fs.validate_expressions(symtab, lib)`.

4. **`FormatString::validate_expressions`** — [format_string.rs:207](../../../openjd-rs/crates/openjd-model/src/template/validate_v2023_09/format_strings.rs)
   walks each `Segment::Expression { parsed, ... }`, builds
   `parsed.with_library(lib).evaluate(&[symtab])` — and **runs the
   evaluator**. Because every value is `Unresolved(T)`, the evaluator
   propagates types but never produces a concrete value; the result is
   `Ok(Unresolved(_))` if every name resolves and types align, or `Err`
   identifying the offending segment.

This is the spec's static-type-check trick: instead of writing a separate
type checker, run the evaluator with sentinels.

5. **Let bindings get a partial-resolution**: [format_strings.rs:1073](../../../openjd-rs/crates/openjd-model/src/template/validate_v2023_09/format_strings.rs#L1073)
   `validate_let_bindings` actually evaluates each binding (against a
   symtab still full of unresolveds) so subsequent bindings see the
   inferred type — this is why `let foo = Param.X + 1; let bar = foo * 2`
   works: at validate-time `foo` becomes `Unresolved(INT)` and `bar`
   type-checks.

## Stage 3 — Resolve template-scope strings (create_job)

Triggered by `openjd run`, REST-style job submission, or any caller that
turns a validated `JobTemplate` into a `Job`:

1. **`create_job(jt, params, ctx)`** — [openjd-rs/crates/openjd-model/src/job/create_job/mod.rs:49](../../../openjd-rs/crates/openjd-model/src/job/create_job/mod.rs#L49)
   builds a real symtab from `params` via `build_symbol_table` then
   resolves the job name with **`PathFormat::Posix`** (paths are
   normalized to forward slashes at template time; the runtime can
   re-convert on Windows).

2. **`jt.name.resolve_string_with(&symtab, &opts)`** — [format_string.rs:136](../../../openjd-rs/crates/openjd-expr/src/format_string.rs#L136)
   walks each segment, calling `eval_parsed` on expressions. Unlike
   stage 2 there's no library override defaulting through the host —
   `template_lib` is built without path-mapping rules (`HostContext::None`)
   so functions like `apply_path_mapping` are out of scope.

3. **`eval_parsed(parsed, symtab, library, path_format, target_type)`** — [format_string.rs:185](../../../openjd-rs/crates/openjd-expr/src/format_string.rs#L185)
   builds an `EvalBuilder`, applies overrides, then
   `builder.evaluate(&[symtab])` — same code path as stage 2, but
   symtab contains real values, so `Unresolved(_)` never appears.

4. **`Evaluator::evaluate(&parsed.ast)`** — [openjd-rs/crates/openjd-expr/src/eval/evaluator.rs](../../../openjd-rs/crates/openjd-expr/src/eval/evaluator.rs)
   walks the ruff AST recursively. Every node bumps an operation
   counter and tracks peak memory; both are bounded by
   `DEFAULT_OPERATION_LIMIT` and `DEFAULT_MEMORY_LIMIT`. Functions
   come from the `FunctionLibrary` (the `lib` overlay).

5. **`instantiate_step(st, &symtab, has_expr, &limits, ctx)`** — [openjd-rs/crates/openjd-model/src/job/create_job/instantiate.rs:20](../../../openjd-rs/crates/openjd-model/src/job/create_job/instantiate.rs#L20)
   clones the symtab, sets `Step.Name`, evaluates step-level let
   bindings, resolves task-parameter ranges, host requirements, and
   step environments. Each environment's symtab is **filtered** to
   only the names referenced by its format strings (so the
   serialized `Job` doesn't leak unrelated symbols downstream).

6. The result: a `job::Job` whose template-scope strings (job name,
   step name, parameter-space ranges, host requirements) are now
   **resolved Rust strings** — but session/task scope strings are
   still `FormatString` ASTs, deferred until session runtime.

## Stage 4 — Resolve session/task strings (session runtime)

On the worker, [openjd-sessions](../../../openjd-rs/crates/openjd-sessions/src/session.rs)
takes the `Job`, walks step env vars and action commands:

```rust
let value = fmt_str.resolve_string_with(
    &symtab,
    &openjd_expr::FormatStringOptions::new().with_library(self.lib()),
)?;
```

at [session.rs:870](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L870).
Here `self.lib()` is built from a profile carrying real path-mapping
rules (`HostContext::WithRules`) — `apply_path_mapping` is now a real
function. The same `parsed` AST that was lexed in stage 1, type-checked
in stage 2, and partially resolved in stage 3 is now finally evaluated
to a concrete string.

## Library-only minimal path (no template at all)

```rust
use openjd_expr::{ExprValue, FormatString, FormatStringOptions, SymbolTable};

let fs = FormatString::new("Hello {{ Param.Name + '!' }}").unwrap();
let mut st = SymbolTable::new();
st.set("Param.Name", "world").unwrap();

let s = fs.resolve_string_with(&st, &FormatStringOptions::default()).unwrap();
assert_eq!(s, "Hello world!");
```

`FormatString::new` runs stages 1; `resolve_string_with` runs the
equivalent of stages 2–4 collapsed into one (the symtab is fully
resolved, so it's just evaluation).

## Why this is the most-touched code in the repo

Every `command:`, every `args:`, every `timeout:`, every env-var
value, every host-requirement, every let-binding, every wrap-action
hook is a `FormatString` field. The lex-once / evaluate-many design
means: the ruff parser runs **at most once per literal `{{...}}`** in
the template, regardless of how many tasks the parameter space yields
or how many sessions reuse the job. Every per-task resolution is just
an AST walk against a fresh symtab.
