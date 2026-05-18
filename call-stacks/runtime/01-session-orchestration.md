# 01 — Running a session, top to bottom

**Use case:** A worker has a validated `Job` (the output of [template
parsing koan 03](../template-parsing/03-expression-block-end-to-end.md)
stage 3). It's holding `Vec<Environment>` for `jobEnvironments`, the
typed `Step` it needs to run, and a `TaskParameterSet` for the task it
chose. Walk how that becomes a real subprocess on the worker, with
the right env vars, the right argv, the right working dir.

This is the orchestration layer in
[openjd-sessions/src/session.rs](../../../openjd-rs/crates/openjd-sessions/src/session.rs).
It owns the session-state machine, the active environment stack,
mutating env vars across actions, and the connection to the runner
that actually launches the subprocess.

## Three big phases

```text
1. Session::with_config(SessionConfig)              — construct
   ├─ working dir, files dir, callbacks
   ├─ path-mapping rules
   ├─ profile (revision + extensions, used to pick the FunctionLibrary)
   └─ optional cross-user helper
                 │
2.  enter_environment(env, resolved_symtab, ...)    — repeat per env
   ├─ build session symtab (path mapping applied)
   ├─ resolve env.variables → env_vars
   ├─ run env.script.actions.on_enter via EnvScriptRunner
   └─ push identifier onto environments_entered
                 │
3.  run_task(script, task_params, resolved_symtab)  — repeat per task
   ├─ build task symtab (overlays Task.Param.* on session symtab)
   ├─ materialize_path_mapping (writes JSON, sets Session.* paths)
   ├─ RFC 0008 wrap-routing (substitute onWrapTaskRun if active)
   ├─ resolve script.actions.on_run.command/args/timeout
   ├─ evaluate let_bindings into the symtab
   ├─ allocate + write embedded files (Task.File.* paths)
   ├─ run via StepScriptRunner → Runner::run_action → run_subprocess
   └─ drive_action loop: stream stdout, parse openjd_* directives,
      fold env-var changes back into session.env_vars
                 │
4.  exit_environment(identifier)                   — once per entered env (LIFO)
   ├─ run env.script.actions.on_exit
   └─ revert env_vars based on created_env_vars[identifier]
```

The state machine ([session.rs:303-310](../../../openjd-rs/crates/openjd-sessions/src/session.rs))
tracks `Ready / Running / ReadyEnding / Ended` and gates which methods
can be called when (e.g., you can't `run_task` while another action
is `Running`).

## Top-level call stack for `run_task`

1. **`Session::run_task(script, task_params, resolved_symtab, os_env_vars)`** —
   [session.rs:1327](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L1327).
   The orchestrator. Sequence of steps:

   - State check: must be `Ready`.
   - **`build_symbol_table(task_params, resolved_symtab)`** — covered
     in [koan 02](02-build-symbol-table.md). Produces a host-context
     symtab carrying everything an action might reference.
   - Action state reset, transition to `Running`, fire callback.
   - Allocate a fresh `CancellationToken` and `tokio::sync::watch`
     channel for "user pressed Ctrl-C" signal delivery.
   - **`evaluate_env_vars(os_env_vars)`** ([session.rs:1920](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L1920))
     folds together: process env, optional `os_env_vars` overlay,
     and the per-environment overlays accumulated from prior
     `onEnter` directives.
   - Clone the symtab into `action_symtab` and call
     **`materialize_path_mapping(&mut action_symtab)`** —
     [koan 05](05-path-mapping-materialization.md). Writes the JSON
     rules file to disk and seeds `Session.HasPathMappingRules` /
     `Session.PathMappingRulesFile`.
   - **RFC 0008 wrap routing** ([session.rs:1377](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L1377)):
     if any active environment in `environments_entered` defines
     `onWrapTaskRun`, substitute that action and overlay
     `WrappedAction.{Command, Args, Environment, Timeout}` symbols.
   - Construct a `StepScriptRunner` and call
     **`runner.run(effective_script, &action_symtab, lib, &env_vars, tx)`** —
     [runner/step_script.rs:114](../../../openjd-rs/crates/openjd-sessions/src/runner/step_script.rs#L114).
   - **`drive_action(runner_fut, &mut rx, identifier)`** —
     [session.rs:1722](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L1722).
     The `tokio::select!` loop that races the action future
     against the message channel. Each `ActionMessage` (Progress,
     Status, Fail, SetEnv, UnsetEnv, RedactedEnv, …) is handled by
     `apply_message` ([session.rs:1813](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L1813)),
     which mutates `self.env_vars`, `self.action`, etc., **before**
     the action future is allowed to complete. After the future
     resolves, the loop drains any leftover messages.

2. **`StepScriptRunner::run(...)`** —
   [runner/step_script.rs:114](../../../openjd-rs/crates/openjd-sessions/src/runner/step_script.rs#L114).
   Two pre-action stages plus the action itself:

   - **Let bindings**: `evaluate_let_bindings(bindings, symtab, lib, host_path_format)` —
     [openjd-model/src/job/create_job/instantiate.rs:404](../../../openjd-rs/crates/openjd-model/src/job/create_job/instantiate.rs#L404).
     Each `name = expr` is parsed once, evaluated against the
     working symtab (so each binding can see prior bindings), and
     `result.set(name, value)` extends the table.
   - **Embedded files**: `EmbeddedFiles::allocate_file_paths` then
     `write_file_contents` — [koan 04](04-embedded-files-materialization.md).
     Two-phase so `Task.File.*` symbols are visible **before** the
     `data:` field's interpolations resolve (which can themselves
     reference other `Task.File.*` siblings).
   - **`Runner::run_action(action, symtab, lib, env_vars, tx, ...)`** —
     [runner/mod.rs:142](../../../openjd-rs/crates/openjd-sessions/src/runner/mod.rs#L142).

3. **`Runner::run_action`** — turns an `Action` into a subprocess:

   - **`resolve_action_args(action, symtab, lib)`** — [koan 03](03-resolve-action-to-argv.md).
     Walks `command` + `args[]` to produce `Vec<String>` argv.
   - **`resolve_action_timeout(action, symtab, lib, default)`** —
     parses `{{ Param.Timeout }}` or similar to a `Duration`.
   - **`cancel_method_for_action`** — picks `Terminate` or
     `NotifyThenTerminate { terminate_delay }` based on the action's
     `cancelation:` config.
   - Builds a `SubprocessConfig`, creates an `ActionFilter`
     (the stdout directive parser — [koan 06](06-openjd-env-stdout-directives.md)),
     and calls **either** `run_via_helper` (cross-user) or
     **`run_subprocess`** ([subprocess.rs:442](../../../openjd-rs/crates/openjd-sessions/src/subprocess.rs#L442)).

4. **`run_subprocess(config, filter, session_id, message_tx, cancel_token)`** —
   the bottom of the stack:

   - Merges process env + per-action overrides.
   - Spawns via `tokio::process::Command::new(&args[0]).args(&args[1..])`.
     Linux: wrapped in `setsid` so `kill -TERM -<pgid>` reaches the
     whole process tree. Windows: a Job Object is configured by
     `configure_command` for the same effect.
   - Streams stdout line-by-line through `ActionFilter::filter_message`,
     pushing parsed `ActionMessage`s through `message_tx`.
   - Returns `SubprocessResult { state, exit_code, stdout }` once the
     process exits or is terminated by timeout/cancel.

5. **Back in `drive_action`**: the loop terminates when both the
   future and the channel are done. Final state is updated, callback
   fires, session moves to `Ready` (or `ReadyEnding` if anything
   failed or `ending_only` was set).

## Why three runners (`StepScriptRunner`, `EnvScriptRunner`, base `Runner`)

`Runner` ([runner/mod.rs:116](../../../openjd-rs/crates/openjd-sessions/src/runner/mod.rs#L116))
is the action-launching primitive: take a resolved action plus a
symtab, run it as a subprocess. It doesn't know about let bindings,
embedded files, or where the script came from.

The two specialized runners wrap `Runner` with the per-context
pre-action work:

- **`StepScriptRunner`** ([runner/step_script.rs:31](../../../openjd-rs/crates/openjd-sessions/src/runner/step_script.rs#L31))
  knows that step scripts have `let_bindings` and `embedded_files` to
  resolve before `onRun` fires, and uses `Task` scope for embedded
  file naming.
- **`EnvScriptRunner`** ([runner/env_script.rs:35](../../../openjd-rs/crates/openjd-sessions/src/runner/env_script.rs#L35))
  knows that env scripts run `onEnter`/`onExit`/`onWrap*` (each gets
  its own symtab building) and uses `Env` scope for embedded files.

Both delegate the actual subprocess to `Runner::run_action`.

## Cross-user execution (briefly)

If `SessionConfig.user` is set to a different user from the worker
process, all subprocess launches go through a **helper binary**
([cross_user_helper.rs](../../../openjd-rs/crates/openjd-sessions/src/cross_user_helper.rs)).
The helper runs as the target user, accepts subprocess configs over
a pipe, executes them, and forwards stdout/exit codes back. This is
how a worker daemon running as one user can run jobs as a different
unprivileged user without dropping privileges in-process.

`Session::run_task` doesn't care which path is taken — `Runner::run_action`
checks `self.helper` and routes accordingly.

## State machine overview

```text
Ready  ──enter_environment──►  Running ──drive_action loop ends──►  Ready
                                  │
Ready  ────run_task────────►  Running ──drive_action loop ends──►  Ready (success)
                                  │                              ╲
                                  ╲                               ─►  ReadyEnding (failure / ending_only)
ReadyEnding ──exit_environment──► Running ──drive_action loop───► ReadyEnding
                                                                        │
                                                                        ╲
                                                          all envs exited ─► Ended
```

`Ready` accepts `enter_environment` and `run_task`. `ReadyEnding`
accepts only `exit_environment` (you finish unwinding the env stack
even after a failure). `Ended` accepts nothing — the session is
disposed.

## Related koans

- [02 — Building a runtime symbol table](02-build-symbol-table.md)
- [03 — Resolving an action to argv](03-resolve-action-to-argv.md)
- [04 — Embedded files materialization](04-embedded-files-materialization.md)
- [05 — Path-mapping materialization](05-path-mapping-materialization.md)
- [06 — `openjd_env` stdout directive flow](06-openjd-env-stdout-directives.md)
- [Template parsing 03 — Expression block end-to-end](../template-parsing/03-expression-block-end-to-end.md), stage 4
- [Template parsing 06 — Symbol table seeding](../template-parsing/06-symbol-table-seeding-three-scopes.md)
