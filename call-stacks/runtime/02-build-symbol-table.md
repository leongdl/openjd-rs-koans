# 02 — Building a runtime symbol table from a `Job`

**Use case:** A `Session` holds a `Job` produced by `create_job`. That
`Job` carries `resolved_symtab: SerializedSymbolTable` snapshots
attached to each step and environment, frozen with **template-side
scope semantics** (Posix paths, no `Session.*`, no `Task.Param.*`,
PATH params absent). The runtime needs a host-context symtab where
`Param.X` is real and path-mapped, `Task.Param.Frame` exists, and
`Session.WorkingDirectory` is the actual on-disk path. Trace the
transformation.

This is the bridge between [template parsing koan 03's stage 3](../template-parsing/03-expression-block-end-to-end.md)
(create-job, template scope) and stage 4 (session runtime, host scope).
Two views of the same parameters, with rules applied between them.

## What `resolved_symtab` carries

[`SerializedSymbolTable::from_symtab`](../../../openjd-rs/crates/openjd-expr/src/symbol_table.rs)
is built at create-job time
([instantiate.rs:249](../../../openjd-rs/crates/openjd-model/src/job/create_job/instantiate.rs#L249)
for steps,
[instantiate.rs:332](../../../openjd-rs/crates/openjd-model/src/job/create_job/instantiate.rs#L332)
for environments). It's deliberately **filtered**: only the symbols
referenced by the step/env's format strings get serialized. So if a
step doesn't reference `Param.OutputDir`, that param doesn't appear in
its `resolved_symtab`. This keeps unrelated job parameters from leaking
into a step's serialized payload (relevant for, e.g., a job whose
parameters include credentials but only one step needs them).

Crucially, **PATH-typed `Param.*` are excluded entirely**. They live
only in `RawParam.*` (as raw STRING) at create-job time, because the
session is the only place that knows the worker's path-mapping rules.

## Call stack

1. **`Session::run_task` / `Session::enter_environment`** —
   [session.rs:1341, 851](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L1341).
   Both call:

   ```rust
   let symtab = self.build_symbol_table(task_parameter_values, resolved_symtab)?;
   ```

2. **`Session::build_symbol_table(task_params, base)`** —
   [session.rs:1962](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L1962).
   Logic splits on whether `base` (the `resolved_symtab`) was provided:

   - **`Some(base)` — typical case after create_job**:
     - Deserialize via `base.to_symtab(host_path_format)` —
       Posix paths in the snapshot get **renormalized to host format**
       (Windows backslashes on Windows, etc.).
     - Walk `self.job_parameter_values` — these are the original
       (unmapped) values from `preprocess_job_parameters`. For each
       PATH or LIST_PATH:
       - Apply `apply_path_mapping_to_string` to the raw value.
       - Insert as `Param.<name>` with type `PATH` / `list(PATH)` and
         the host-format path.
     - Non-PATH params are already in the deserialized base — no
       overlay needed.

   - **`None` — caller didn't pre-resolve (rare)**:
     - Build a fresh `SymbolTable`, walk `job_parameter_values`
       directly, set both `RawParam.<name>` (always present) and
       `Param.<name>` (with path mapping for PATH types).

3. **`Session.*` overlay** at [session.rs:2076](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L2076):

   ```rust
   symtab.set("Session.WorkingDirectory",
              ExprValue::new_path(self.working_directory.to_string_lossy(), host_path_format))?;
   ```

   `Session.HasPathMappingRules` and `Session.PathMappingRulesFile`
   are added later by `materialize_path_mapping` ([koan 05](05-path-mapping-materialization.md))
   — `build_symbol_table` only sets `Session.WorkingDirectory`.

4. **`Task.Param.*` overlay** (only for `run_task`) at [session.rs:2087](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L2087):
   For each `(name, TaskParameterValue { param_type, value })`:
   - `Task.RawParam.<name>` — raw value as-is.
   - `Task.Param.<name>`:
     - PATH → `apply_path_mapping_to_string`, wrap as
       `ExprValue::new_path`, host format.
     - Otherwise → typed value as-is.

   Note that **task-parameter PATH values get path-mapped at the same
   layer** as job-parameter PATH values. Both cross the same boundary
   into host context.

5. **For wrap actions only** ([session.rs:1387–1409](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L1387)):
   the wrap env's own `resolved_symtab` is merged on top via
   `merge_from`. Wrap envs may carry `Param.<wrap_specific>` that
   only the wrap env uses; this layering makes them visible inside
   the wrap-action's symtab without polluting the regular task
   symtab.

## Path-mapping helpers

- **`apply_path_mapping_to_string(path)`** —
  [session.rs:2122](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L2122).
  Walks `self.path_mapping_rules` in order; the **first** matching
  rule's `apply()` wins; otherwise the input is returned unchanged.
  Order matters because rules are applied as longest-match-wins by
  the rule itself, but if two rules both match, the first declared
  one wins (simple iteration order, no priority field).

- **`apply_path_mapping_to_value(value)`** —
  [session.rs:2132](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L2132).
  Wrapper that handles `ExprValue::Path` and `ExprValue::ListPath`,
  delegating each path through `apply_path_mapping_to_string`.

## Why two builders (`build_symbol_table` vs the `parameters.rs` one)

There's also `build_symbol_table` in
[create_job/parameters.rs:1154](../../../openjd-rs/crates/openjd-model/src/job/create_job/parameters.rs#L1154).
That one runs **before** the host knows what its path-mapping rules
are — it produces the **template-scope** symtab used by `create_job`
to resolve job names, host requirements, and step let bindings. Path
params are intentionally absent there.

The session's `build_symbol_table` is the **host-scope** version: it
takes the template-scope snapshot and applies path mapping to produce
the final values an action will see.

## Visualizing the layering

```
JobParameterValues (raw user input, post-coercion)
        │
        ▼  build_symbol_table (template scope)   [parameters.rs]
template SymbolTable: Param.* (no PATH), RawParam.*, no Session.*
        │
        ▼  resolve job name + host requirements + step lets [create_job]
        │
        ▼  filter to symbols referenced by each step/env script [instantiate]
SerializedSymbolTable (frozen on Job::Step.resolved_symtab)
        │
        ▼  Session::build_symbol_table (host scope)   [session.rs]
host SymbolTable: Param.* (PATH-mapped), RawParam.*, Session.WorkingDirectory
        │
        ▼  + Task.Param.* (PATH-mapped) [run_task only]
        ▼  + Session.HasPathMappingRules + Session.PathMappingRulesFile [materialize_path_mapping]
        ▼  + WrappedAction.* + wrap env's resolved_symtab merged [if RFC 0008 wrap-routing]
        │
        ▼  evaluate_let_bindings (script-level let:)   [instantiate.rs:404]
        ▼  + Task.File.* / Env.File.* [embedded_files allocate_file_paths]
        │
        ▼  resolve_action_args, resolve_action_timeout, resolve env.variables
final argv + duration + env vars → run_subprocess
```

Every arrow is a transformation that adds names or applies a per-host
rewrite. Nothing is recomputed; the validate-time symtab in
[template parsing koan 06](../template-parsing/06-symbol-table-seeding-three-scopes.md)
is the *predicted shape* of this final host-scope symtab.

## Related koans

- [01 — Session orchestration](01-session-orchestration.md)
- [05 — Path-mapping materialization](05-path-mapping-materialization.md)
- [Template parsing 06 — Symbol table seeding](../template-parsing/06-symbol-table-seeding-three-scopes.md)
- [Template parsing 07 — Parameter merge](../template-parsing/07-parameter-merge-and-symbol-table-seed.md)
