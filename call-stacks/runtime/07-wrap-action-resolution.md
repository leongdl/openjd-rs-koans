# 07 — Resolving wrap actions (RFC 0008)

**Use case:** A session has entered a Docker-wrapping environment that
defines all three wrap hooks:

```yaml
environment:
  name: DockerWrap
  script:
    actions:
      onEnter:    { command: bash, args: ["-c", "docker pull alpine"] }
      onWrapEnter: { command: docker, args: ["start", "{{Env.Wrapped.Name}}"] }
      onWrapTaskRun: { command: docker, args: ["exec", "task", "{{WrappedAction.Command}}", "{{WrappedAction.Args}}"] }
      onWrapExit: { command: docker, args: ["stop", "{{Env.Wrapped.Name}}"] }
      onExit:    { command: bash, args: ["-c", "docker rmi alpine"] }
```

Then the session enters an inner env `BlenderEnv` and runs a step
whose `onRun: blender --frame=42`. What actually executes? Five
things, in this order:

1. `bash -c "docker pull alpine"` (DockerWrap's own `onEnter` — *not* wrapped)
2. `docker start BlenderEnv` (DockerWrap's `onWrapEnter`, wrapping BlenderEnv's `onEnter`)
3. `docker exec task blender --frame=42` (DockerWrap's `onWrapTaskRun`)
4. `docker stop BlenderEnv` (DockerWrap's `onWrapExit`, wrapping BlenderEnv's `onExit`)
5. `bash -c "docker rmi alpine"` (DockerWrap's own `onExit`)

How does the runtime arrive at that ordering, and how does
`WrappedAction.Command` know what the inner action's command was?

## Three dispatch paths, three call sites

Wrap routing happens in three places in
[session.rs](../../../openjd-rs/crates/openjd-sessions/src/session.rs),
once per wrap hook. All three follow the same shape but differ in
*which inner action they wrap* and *which symbols they seed*:

| Hook | Trigger | Wraps | Seeds |
| --- | --- | --- | --- |
| `onWrapEnter` | another env is being entered | inner env's `onEnter` | `WrappedAction.*` + `Env.Wrapped.Name` |
| `onWrapTaskRun` | step's `onRun` is about to fire | step's `onRun` | `WrappedAction.*` only |
| `onWrapExit` | another env is being exited | inner env's `onExit` | `WrappedAction.*` + `Env.Wrapped.Name` |

The wrap env's own `onEnter`/`onExit` are **never** wrapped (an env
can't wrap itself). And by spec validation
([template parsing koan 03](../template-parsing/03-expression-block-end-to-end.md)
stage 2), there's at most one wrap env active per session — but the
runtime defends against multiple anyway with an "innermost wins"
search.

## The three resolution helpers

### Stack queries

- **`active_wrap_env()`** — [session.rs:2177](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L2177)
  walks `environments_entered` **in reverse** and returns the first
  env with any wrap hook. Innermost wins.
- **`wrap_env_excluding(self_id)`** — [session.rs:2192](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L2192)
  same, but skips the named env. Used by `enter_environment` because
  the env being entered is already on the stack but shouldn't wrap
  its own `onEnter`.
- **`env_has_any_wrap_hook(env)`** — [session.rs:2210](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L2210)
  predicate the stack-walks use; checks for any of the three hooks
  being `Some`.

### Symbol overlay

- **`overlay_wrapped_action_symbols(symtab, wrapped_env_name, cmd, args, env, timeout_secs)`** —
  [session.rs:2236](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L2236)
  Sets `WrappedAction.Command` (string), `WrappedAction.Args`
  (`list[string]`), `WrappedAction.Environment` (`list[string]` of
  `KEY=value`), `WrappedAction.Timeout` (int seconds, `0` if unset).
  When `wrapped_env_name` is `Some`, also sets `Env.Wrapped.Name`.

The split is deliberate: `WrappedAction.*` is available in all three
hooks; `Env.Wrapped.Name` is only meaningful when wrapping an env
lifecycle hook (tasks have no equivalent symbol).

## Path 1 — `onWrapTaskRun` for tasks

This is the simplest path; covers the common case.

1. **`Session::run_task(...)`** — [session.rs:1327](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L1327).
   Standard task path through `build_symbol_table` →
   `materialize_path_mapping`. Then at [session.rs:1377](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L1377):

   ```rust
   let wrap_action: Option<openjd_model::job::Action> = self
       .active_wrap_env()
       .and_then(|wrap_env| {
           wrap_env.script.as_ref()
               .and_then(|s| s.actions.on_wrap_task_run.clone())
               .map(|action| (wrap_env, action))
       })
       .map(|(wrap_env, action)| (wrap_env.resolved_symtab.clone(), action))
       .map(|(wrap_symtab, action)| {
           if let Some(ser) = wrap_symtab.as_ref() {
               match ser.to_symtab(host_path_format) {
                   Ok(st) => action_symtab.merge_from(&st),
                   Err(e) => log::warn!(...),
               }
           }
           action
       });
   ```

   - `active_wrap_env` → innermost env with a wrap hook (no exclusion
     needed for tasks; the task isn't an env).
   - `s.actions.on_wrap_task_run.clone()` → if this hook is missing,
     no wrapping happens; `wrap_action` is `None` and the task runs
     as written.
   - **If wrapping**: `wrap_env.resolved_symtab` is the
     `SerializedSymbolTable` snapshot frozen at create-job time
     ([template parsing koan 06](../template-parsing/06-symbol-table-seeding-three-scopes.md)).
     It carries the wrap env's own `Param.*`/let bindings — symbols
     visible only to the wrap env. Merging it onto `action_symtab`
     lets the wrap action reference its own scope.
   - Failure to deserialize is logged but **doesn't fail the task**.
     The action might not actually need any of those symbols.

2. **Resolve the inner action's command/args/timeout** at [session.rs:1417](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L1417):

   ```rust
   if wrap_action.is_some() {
       let resolved_cmd = resolve_action_args(&script.actions.on_run, &action_symtab, Some(&lib))?;
       let (cmd, args) = match resolved_cmd.split_first() {
           Some((head, tail)) => (head.clone(), tail.to_vec()),
           None => (String::new(), Vec::new()),
       };
       let task_env: Vec<String> = self.env_vars.iter()
           .map(|(k, v)| format!("{k}={v}"))
           .collect();
       let wrapped_timeout = resolve_action_timeout(...)?;
       overlay_wrapped_action_symbols(
           &mut action_symtab,
           None,                              // tasks have no Env.Wrapped.Name
           &cmd, &args, &task_env,
           wrapped_timeout.map(|d| d.as_secs() as i64).unwrap_or(0),
       )?;
   }
   ```

   This is the **same** [`resolve_action_args`](03-resolve-action-to-argv.md)
   as a normal task — just used here to compute what the inner
   command *would have been*, so that string is available to the
   wrap hook as `WrappedAction.Command`.

   The seeded values are always computed when wrapping, even if the
   wrap hook doesn't reference all of them. They're cheap and
   `WrappedAction.*` references outside wrap hooks are a validation
   error, so seeding when the symbol won't be read is harmless.

3. **Substitute the script** at [session.rs:1509](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L1509):

   ```rust
   let effective_script: Cow<StepScript> = match wrap_action {
       Some(action) => Cow::Owned(StepScript {
           let_bindings: script.let_bindings.clone(),
           actions: StepActions { on_run: action },
           embedded_files: script.embedded_files.clone(),
       }),
       None => Cow::Borrowed(script),
   };
   ```

   The wrap action **replaces** the step's `onRun` while keeping the
   step's own `let_bindings` and `embedded_files`. The user's
   `Task.File.*` symbols remain bound — the wrap hook can pass the
   inner script's path through to a container, e.g.,
   `["docker", "exec", "task", "bash", "{{Task.File.script}}"]`.

4. **Run** through `StepScriptRunner::run` → `Runner::run_action` →
   `run_subprocess` exactly as a normal task ([koan 01](01-session-orchestration.md)).
   The runner has no idea this was a wrap action; it just sees a
   resolved command and arguments.

## Path 2 — `onWrapEnter` for env entry

In `Session::enter_environment_with_output` at [session.rs:921](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L921):

```rust
let wrap_action = self.wrap_env_excluding(&identifier).and_then(|outer| {
    outer.script.as_ref()
        .and_then(|s| s.actions.on_wrap_enter.as_ref())
        .cloned()
        .map(|action| (outer.resolved_symtab.clone(), action))
});
```

The key difference from the task path:

- **`wrap_env_excluding(&identifier)`** rather than `active_wrap_env()`.
  By the time this code runs, the env being entered is **already
  pushed** onto `environments_entered` (at [session.rs:863](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L863))
  — but it shouldn't wrap its own `onEnter`. Excluding it makes the
  search find the *outer* wrap env if any.

- The wrapped action is the inner env's `onEnter`, resolved with
  `resolve_action_args(inner_on_enter, ...)`.

- **`Env.Wrapped.Name` is set to the inner env's name** —
  `overlay_wrapped_action_symbols(&mut action_symtab, Some(&env.name), ...)`.
  This is what lets `docker start {{Env.Wrapped.Name}}` know which
  container to start.

- The runner is `EnvironmentScriptRunner::run_wrap_action` rather
  than `enter` ([env_script.rs:174](../../../openjd-rs/crates/openjd-sessions/src/runner/env_script.rs#L174)) —
  see "Why a separate `run_wrap_action`" below.

## Path 3 — `onWrapExit` for env exit

In `Session::exit_environment` at [session.rs:1169](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L1169):

```rust
let wrap_action = self.active_wrap_env().and_then(|outer| {
    outer.script.as_ref()
        .and_then(|s| s.actions.on_wrap_exit.as_ref())
        .cloned()
        .map(|action| (outer.resolved_symtab.clone(), action))
});
```

Symmetric to onEnter, with one exception: **`active_wrap_env`** is
used directly — *not* `wrap_env_excluding`. That's because the env
being exited has *already been popped* from the stack by the time
this code runs (per [session.rs:1162](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L1162)
comment). The remaining envs are exactly the ones that should be
considered for wrapping; no exclusion needed.

Same `Env.Wrapped.Name` seeding as onEnter.

## Why a separate `run_wrap_action`

[`EnvironmentScriptRunner::run_wrap_action`](../../../openjd-rs/crates/openjd-sessions/src/runner/env_script.rs#L174)
is a low-level pass-through: take an arbitrary `Action`, run it via
`base.run_action`. **It does not** materialize embedded files or
evaluate let bindings.

The reason: by the time the session calls `run_wrap_action`, the
caller has already:
- Built the symtab with `Env.Wrapped.Name` and `WrappedAction.*`
  overlaid.
- Resolved the inner action's command/args/timeout to seed those
  symbols.
- Decided the runner should execute *the wrap action*, not the
  inner env's lifecycle action.

The wrap env's own embedded files and let bindings were materialized
when the wrap env was originally entered (before any wrapping
happened). The inner env's are not relevant because the wrap action
is what runs.

By contrast, `EnvironmentScriptRunner::enter` and
`EnvironmentScriptRunner::exit` ([env_script.rs:118, 135](../../../openjd-rs/crates/openjd-sessions/src/runner/env_script.rs#L118))
go through `run_env_action` ([env_script.rs:197](../../../openjd-rs/crates/openjd-sessions/src/runner/env_script.rs#L197))
which **does** materialize embedded files and evaluate let bindings
for the env it's entering or exiting — that's the regular,
non-wrapped path.

## What the wrap script sees at runtime

Suppose `BlenderEnv.onEnter = { command: bash, args: ["-c", "blender --version"] }`
and DockerWrap's `onWrapEnter = { command: docker, args: ["run", "{{WrappedAction.Command}}", ...] }`.

Inside the wrap action's symtab at runtime:
- `WrappedAction.Command` = `"bash"`
- `WrappedAction.Args` = `["-c", "blender --version"]`
- `WrappedAction.Environment` = `["FOO=bar", "PATH=/usr/bin:..."]`
  (every entry currently in `self.env_vars` formatted as `KEY=value`)
- `WrappedAction.Timeout` = `0` (BlenderEnv.onEnter had no timeout)
- `Env.Wrapped.Name` = `"BlenderEnv"`
- Plus the wrap env's own `resolved_symtab` (Param.*/let-bindings),
  the regular session-scope symbols (`Session.WorkingDirectory`
  etc.), and any path mapping.

So the wrap script can construct the equivalent docker invocation
that runs the inner action inside the container.

## RFC 0008 invariants the runtime relies on

The session code assumes — and the model layer enforces:

- **At most one wrap env active per session.** The "innermost wins"
  search is defense in depth; in practice the validator
  ([template parsing koan 03](../template-parsing/03-expression-block-end-to-end.md)
  stage 2) rejects templates that put two wrap envs in the same
  session stack.
- **All-or-nothing wrap hooks.** An env that defines any of the
  three hooks must define all three. The runtime treats "no wrap
  hook" as "no wrapping" — but the spec guarantees that an env with
  `onWrapTaskRun` will also have `onWrapEnter` and `onWrapExit`,
  so the lifecycle is symmetric.
- **`WrappedAction.*` only referenced inside wrap hooks.** The
  validator rejects templates where a non-wrap hook references
  these symbols. The runtime seeds them only when wrapping is
  active, so a stray reference outside a wrap hook would resolve
  to "undefined symbol" at runtime — but it never reaches that
  point.

## Related koans

- [01 — Session orchestration](01-session-orchestration.md) — where
  these three dispatch points sit in the larger flow.
- [02 — Building a runtime symbol table](02-build-symbol-table.md) —
  base symtab the wrap overlays mutate.
- [03 — Resolving an action to argv](03-resolve-action-to-argv.md) —
  used to compute `WrappedAction.Command` / `Args` from the inner
  action.
- [Template parsing 03 — Expression block end-to-end](../template-parsing/03-expression-block-end-to-end.md) —
  validate-time wrap-hook scope (`add_wrapped_action_scope`).
