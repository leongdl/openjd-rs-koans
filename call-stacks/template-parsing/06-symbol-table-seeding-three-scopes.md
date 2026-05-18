# 06 — Symbol table seeding across three scopes

**Use case:** Trace why `{{ Param.X }}` is valid in a step `command`,
why `{{ Session.WorkingDirectory }}` is invalid in a job `name`, and
why `{{ Task.Param.Frame }}` is invalid inside a `jobEnvironments`
block. Who decides what's in scope where?

The OpenJD specification draws scope boundaries that the validator
enforces by **building a different `SymbolTable` for each scope**.
Format strings are blind to scope; their evaluator sees whatever is
in the symtab handed to it.

## The three scopes

| Scope | What's defined | Where it's used |
| --- | --- | --- |
| **Template** | `Param.*` (non-PATH), `RawParam.*`, `Job.Name`*, `Step.Name`* | Job name, host requirements, parameter-space ranges, step-level let bindings |
| **Session** | template + `Session.*`, `Env.File.*`, `Job.Name`*, `Step.Name`* (step envs only) | Environment scripts, environment variables |
| **Task** | session + `Task.Param.*`, `Task.RawParam.*`, `Task.File.*` | Step `script:` (command, args, timeout), embedded files |

\* gated on the EXPR extension.

The wrap-action hooks (RFC 0008) get a fourth, derived scope that
extends Session with `WrappedAction.*` and conditionally
`Env.Wrapped.Name`.

## The seeding functions

All three live in
[validate_v2023_09/format_strings.rs](../../../openjd-rs/crates/openjd-model/src/template/validate_v2023_09/format_strings.rs):

1. **`build_param_symtab(jt)`** — [format_strings.rs:29](../../../openjd-rs/crates/openjd-model/src/template/validate_v2023_09/format_strings.rs#L29)
   the inner shared helper. For each declared `Param.X`:
   - Sets `Param.X` to `Unresolved(<param_type>)`.
   - Sets `RawParam.X` to `Unresolved(STRING)` for PATH (or
     `list(STRING)` for LIST_PATH); the parameter's natural type
     otherwise. The `RawParam.*` overlay is the spec's "what the user
     typed before path mapping" view — always a string for path types.

2. **`build_template_scope_symtab(jt)`** — [format_strings.rs:60](../../../openjd-rs/crates/openjd-model/src/template/validate_v2023_09/format_strings.rs#L60)
   like `build_param_symtab` but **skips PATH-typed `Param.*`**. PATH
   values can only resolve at the host (path mapping happens at
   session start), so they're not in template scope at all. `RawParam.*`
   is still defined because it's a plain string.

3. **`build_session_scope_symtab(jt, env, is_step_env, expr_active)`** — [format_strings.rs:95](../../../openjd-rs/crates/openjd-model/src/template/validate_v2023_09/format_strings.rs#L95)
   starts from `build_param_symtab` (PATH `Param.*` *is* in scope here,
   since session-time evaluation happens after path mapping), then adds:
   - `Session.WorkingDirectory: PATH`
   - `Session.HasPathMappingRules: BOOL`
   - `Session.PathMappingRulesFile: PATH`
   - `Step.Name: STRING` if step env + EXPR
   - `Job.Name: STRING` if EXPR
   - `Env.File.<name>: PATH` for each embedded file in this env's script

4. **`build_task_scope_symtab(jt, step, expr_active)`** — [format_strings.rs:152](../../../openjd-rs/crates/openjd-model/src/template/validate_v2023_09/format_strings.rs#L152)
   starts from `build_param_symtab`, adds the same `Session.*`, then:
   - `Job.Name: STRING`, `Step.Name: STRING` (if EXPR)
   - For each task-parameter definition: `Task.Param.<name>` and
     `Task.RawParam.<name>` with the right types
   - `Task.File.<name>: PATH` for each step-script embedded file
   - **Crucially**: `Env.File.*` is NOT added — env files are scoped
     to the env script that defines them, not to step scripts that use
     that env.

5. **Wrap-hook overlay** — [format_strings.rs:1010](../../../openjd-rs/crates/openjd-model/src/template/validate_v2023_09/format_strings.rs#L1010) (`add_wrapped_action_scope`)
   and [format_strings.rs:1025](../../../openjd-rs/crates/openjd-model/src/template/validate_v2023_09/format_strings.rs#L1025) (`add_env_wrapped_name_scope`).
   The session symtab is **cloned**, then `WrappedAction.{Command, Args,
   Environment, Timeout}` is added; `Env.Wrapped.Name` is added too for
   `onWrapEnter` and `onWrapExit` (but not `onWrapTaskRun`). The clone
   keeps the per-hook narrowing local — the underlying session symtab
   doesn't acquire wrap symbols visible to non-wrap actions.

## How `validate_format_strings` uses them

[format_strings.rs:331](../../../openjd-rs/crates/openjd-model/src/template/validate_v2023_09/format_strings.rs#L331).
Pseudo-skeleton:

```text
template_symtab = build_template_scope_symtab(jt)
validate_fs(jt.name, &template_symtab, &template_lib, ...)

for each step:
    if step.host_requirements:
        symtab = build_template_scope_symtab(jt) + step.let_bindings
        validate every host_requirement format string

    for each job_environment env:
        symtab = build_session_scope_symtab(jt, env, false, expr_active) + env.let_bindings
        validate env.variables, on_enter, on_exit, on_wrap_*

    for each step.parameterSpace range:
        symtab = build_template_scope_symtab(jt) + step.let_bindings
        validate range expression

    task_symtab = build_task_scope_symtab(jt, step, expr_active) + step.let_bindings
    validate step.script.actions.on_run.{command, args, timeout}
    validate step.script.embedded_files

    for each step_environment env:
        symtab = build_session_scope_symtab(jt, env, true, expr_active) + step.let_bindings + env.let_bindings
        validate env.variables, on_enter, on_exit, on_wrap_*
```

The shape is: each kind of field gets a scope-appropriate symtab built
on demand. There is no global "everything visible everywhere" table —
that would let a job-name format string reference `Task.Param.X` and
fail late at session time instead of cleanly at validate time.

## The same pattern at runtime

Stages 3 and 4 of [koan 03](03-expression-block-end-to-end.md) build
*real-valued* symtabs of equivalent shape:

- [`create_job`](../../../openjd-rs/crates/openjd-model/src/job/create_job/mod.rs#L49)
  uses `build_symbol_table` (in `parameters.rs`) to seed `Param.*`/`RawParam.*`
  with concrete values, sets `Job.Name`, recurses into
  `instantiate_step` which sets `Step.Name`.
- [`Session::run_action`](../../../openjd-rs/crates/openjd-sessions/src/session.rs)
  builds a session-time symtab including resolved `Session.*` and the
  active `Task.Param.*` for the current task.

The validate-time shape is a *predictive subset*: every name that
exists at runtime exists at validate time as `Unresolved(T)`, so the
evaluator can type-check ahead of time without producing values.

## Why three (and a half) scopes

The spec's claim is that the boundaries are **portable**: they don't
assume a particular file system, queue, or worker — only "things
known when the template was authored", "things known when a session
starts on a worker", and "things known when a task starts within a
session." Each scope adds names that previously couldn't be known.
The wrap-hook scope is the half: it's session scope plus four names
that are only meaningful inside RFC 0008 wrap actions.

## Related

- [koan 03](03-expression-block-end-to-end.md) — the full evaluation
  pipeline these symtabs feed.
- [openjd-rs-koans/rust-internals/02-symbol-table.md](../../rust-internals/02-symbol-table.md) — internals
  of how `SymbolTable` itself works.
