# 16 — onWrap dispatch and parameter injection

## What this is

How RFC 0008's three wrap hooks (`onWrapEnter`, `onWrapTaskRun`,
`onWrapExit`) plug into the existing session lifecycle, and exactly
where the wrapped action's command, args, environment, and timeout get
injected as the `Task.*` / `Env.Wrapped.*` template variables that the
wrap script consumes.

This is the doc to read when you're touching wrap routing or
debugging "why did my wrap script see the wrong `Task.Command`?"

## Read these in order

- `crates/openjd-model/src/template/actions.rs:80-110` — model-level
  fields (`on_wrap_enter`, `on_wrap_task_run`, `on_wrap_exit`,
  `run_on_host`).
- `crates/openjd-model/src/template/validate_v2023_09/wrap_actions.rs` —
  extension gating, single-layer rule, structural checks.
- `crates/openjd-model/src/template/validate_v2023_09/format_strings.rs:946-1000`
  — scope-correct symbol table construction for hooks.
- `crates/openjd-sessions/src/session.rs:2120-2180` — `active_wrap_env`
  / `wrap_env_excluding` / `env_has_any_wrap_hook`.
- `crates/openjd-sessions/src/session.rs:2179-2286` — the two
  overlay helpers (`overlay_task_wrap_symbols`,
  `overlay_env_wrapped_symbols`).
- `crates/openjd-sessions/src/session.rs:868-905` — `enter_environment`
  wrap dispatch.
- `crates/openjd-sessions/src/session.rs:1120-1151` — `exit_environment`
  wrap dispatch.
- `crates/openjd-sessions/src/session.rs:1342-1369` — `run_task` wrap
  dispatch.

## The three layers a wrap traverses

| Layer | What it does | Where |
|---|---|---|
| Model | Stores hook fields on `Action` and gates with `WRAP_ACTIONS` | `openjd-model/src/template/actions.rs`, `validate_v2023_09/wrap_actions.rs` |
| Validation | Type-checks each hook under the *correct* symbol scope | `validate_v2023_09/format_strings.rs:946-1000` |
| Runtime | Substitutes the wrap action and overlays `Task.*` / `Env.Wrapped.*` | `openjd-sessions/src/session.rs` |

A wrap hook never *executes*. The wrap action runs in its place. The
wrap action's command/args are normal format strings that read from
the overlaid symbols.

## Layer 1 — Model

`actions.rs` (paraphrased):

```rust
#[derive(Debug, Clone, Deserialize)]
#[serde(rename_all = "camelCase", deny_unknown_fields)]
pub struct EnvironmentActions {
    pub on_enter: Option<Action>,
    pub on_wrap_enter: Option<Action>,      // RFC 0008
    pub on_wrap_task_run: Option<Action>,   // RFC 0008
    pub on_wrap_exit: Option<Action>,       // RFC 0008
    pub on_exit: Option<Action>,
}

#[derive(Debug, Clone, Deserialize)]
#[serde(rename_all = "camelCase", deny_unknown_fields)]
pub struct Action {
    pub command: FormatString,
    pub args: Option<Vec<FormatString>>,
    pub timeout: Option<u64>,
    pub cancelation: Option<Cancelation>,
    pub run_on_host: Option<bool>,           // RFC 0008
}
```

Serde hands you the typed shape. Nothing wrap-specific has happened yet.

## Layer 2 — Validation

`validate_v2023_09/wrap_actions.rs` enforces three things:

1. **Extension gating.** A template that uses any wrap hook *must*
   list `WRAP_ACTIONS` in `extensions:`. Templates that don't list
   it but define a hook fail validation with a clear message.
2. **Single-layer rule (RFC 0008 §Wrap ordering).** No more than one
   environment in the session stack may define any wrap hook. The
   validator detects this by walking the candidate environment chain
   for each step and counting hook-bearing environments.
3. **Structural placement.** `runOnHost: true` is meaningful only on
   `<Action>` instances inside an environment that *might* be wrapped
   (i.e. an inner environment or step). Setting it on an outer wrap
   hook is silently ignored, but the validator flags problematic
   placements.

`validate_v2023_09/format_strings.rs:946-1000` is the more subtle
piece. Each wrap hook's `<Action>` has its format strings type-checked
under a *different* symbol table than its host environment's normal
actions:

```rust
// crates/openjd-model/src/template/validate_v2023_09/format_strings.rs:951
if let Some(action) = &script.actions.on_wrap_enter {
    validate_action_format_strings_with_extra_symbols(
        action,
        &path_field(&actions_path, "onWrapEnter"),
        // include Env.Wrapped.* in scope, NOT Task.*
        ...
    )?;
}

// :966
if let Some(action) = &script.actions.on_wrap_task_run {
    validate_action_format_strings_with_extra_symbols(
        action,
        &path_field(&actions_path, "onWrapTaskRun"),
        // include Task.* in scope, NOT Env.Wrapped.*
        ...
    )?;
}
```

This is what makes `Task.Command` only valid inside `onWrapTaskRun`,
and `Env.Wrapped.Name` only valid inside `onWrapEnter` / `onWrapExit`.
A template that confuses the two fails at submission time, not at
runtime.

## Layer 3 — Runtime dispatch

The runtime decision tree is implemented in `session.rs` at three
sites (one per lifecycle phase). All three follow the same shape:

```rust
fn dispatch(inner_action: Action, self_id: &str) -> Action {
    if inner_action.run_on_host == Some(true) {
        // Spec opt-out: never intercept this action.
        return inner_action;
    }
    if let Some(outer) = self.wrap_env_excluding(self_id) {
        if let Some(wrap) = outer.script.as_ref()
              .and_then(|s| s.actions.<the matching hook>.as_ref()) {
            // Substitute the wrap action and overlay Task.* / Env.Wrapped.*
            // into the symbol table.
            return wrap.clone();
        }
    }
    // No active wrap, or this hook isn't defined → run on host.
    inner_action
}
```

The *exact* "matching hook" varies:

| Phase | Matching hook | Overlay applied |
|---|---|---|
| `enter_environment` | `on_wrap_enter` | `Env.Wrapped.*` |
| `run_task` | `on_wrap_task_run` | `Task.*` |
| `exit_environment` | `on_wrap_exit` | `Env.Wrapped.*` |

### `active_wrap_env` vs `wrap_env_excluding`

```rust
// crates/openjd-sessions/src/session.rs:2135
fn active_wrap_env(&self) -> Option<&Environment>;

// :2150
fn wrap_env_excluding(&self, self_id: &str) -> Option<&Environment>;
```

The first is "any active wrap environment, innermost wins." The second
excludes one environment by id — used during *enter/exit* dispatch
because **an environment's own lifecycle actions are never wrapped by
its own wrap hooks** (RFC 0008 §Wrap ordering).

The innermost-wins traversal is defensive. The validator already
rejects multi-layer wraps; this guarantees deterministic dispatch
even if a future extension allows nesting.

### `env_has_any_wrap_hook`

```rust
// :2168
fn env_has_any_wrap_hook(env: &Environment) -> bool {
    env.script.as_ref().map(|s|
        s.actions.on_wrap_enter.is_some()
            || s.actions.on_wrap_task_run.is_some()
            || s.actions.on_wrap_exit.is_some()
    ).unwrap_or(false)
}
```

Used by both env-finder helpers. An environment that defines *only*
`onWrapEnter` (not the other two) still counts as a wrap layer; inner
`onRun` and `onExit` actions in that case fall through to the host
because the matching hook is absent.

## How parameters from the wrapped action flow into the wrap script

This is the question the user asked most directly. The key insight:

> The wrapped action's format strings are resolved in the inner
> environment's symbol-table scope **first**, then the resolved
> values are seeded as concrete values into the wrap script's
> overlay.

In code, the order of operations on `run_task` is:

```text
1. The inner step's onRun is the candidate action.
2. Resolve its FormatString fields (command, args, env vars) using the
   inner-step symbol table. The runtime now has:
       wrapped_command:     String
       wrapped_args:        Vec<String>
       wrapped_environment: Vec<String>      // "K=v" lines
       wrapped_timeout:     Option<u64>
3. If active wrap exists and the inner action did not set runOnHost,
   call overlay_task_wrap_symbols(symtab, …) with those resolved
   strings to seed:
       Task.Command       = ExprValue::String(wrapped_command)
       Task.Args          = ExprValue::ListString(wrapped_args)
       Task.Environment   = ExprValue::ListString(wrapped_environment)
       Env.Action.Timeout = ExprValue::Int(wrapped_timeout_secs)
4. The wrap action's own FormatString fields (command/args) are then
   resolved against this overlaid symbol table. The wrap script sees
   Task.Command as a single concrete string — never the inner
   action's `{{Param.X}}` syntax.
5. Run the wrap action's command + args as the actual subprocess.
```

The two overlay helpers in `session.rs:2179-2286` are short, mechanical,
and worth reading top-to-bottom:

```rust
// :2193
fn overlay_task_wrap_symbols(
    symtab: &mut SymbolTable,
    wrapped_command: &str,
    wrapped_args: &[String],
    task_environment: &[String],
    env_action_timeout_secs: i64,
) -> Result<(), SessionError> {
    symtab.set("Task.Command",     ExprValue::String(wrapped_command.into()))?;
    symtab.set("Task.Args",        make_list_of_string(wrapped_args))?;
    symtab.set("Task.Environment", make_list_of_string(task_environment))?;
    symtab.set("Env.Action.Timeout", ExprValue::Int(env_action_timeout_secs))?;
    Ok(())
}

// :2238
fn overlay_env_wrapped_symbols(
    symtab: &mut SymbolTable,
    wrapped_name: &str,
    wrapped_command: &str,
    wrapped_args: &[String],
    wrapped_environment: &[String],
    wrapped_timeout_secs: i64,
) -> Result<(), SessionError> {
    symtab.set("Env.Wrapped.Name",        ExprValue::String(wrapped_name.into()))?;
    symtab.set("Env.Wrapped.Command",     ExprValue::String(wrapped_command.into()))?;
    symtab.set("Env.Wrapped.Args",        make_list_of_string(wrapped_args))?;
    symtab.set("Env.Wrapped.Environment", make_list_of_string(wrapped_environment))?;
    symtab.set("Env.Wrapped.Timeout",     ExprValue::Int(wrapped_timeout_secs))?;
    Ok(())
}
```

Two consequences worth knowing:

- **No double interpolation.** The wrap script can use
  `repr_sh(Task.Command)` safely because `Task.Command` is a
  literal string, not a format-string template. Whatever `{{...}}`
  the inner action used has already been resolved to a fixed value.
- **`Task.Environment` reflects what the runtime saw.** It includes
  every `openjd_env: KEY=value` line emitted by earlier actions in
  the session, exactly as RFC 0008 specifies. If the wrap script
  needs to filter or rewrite, it operates on already-collected
  values.

## Lifecycle walkthrough

Single-layer wrap (the only case this RFC permits): outer env A
declares all three hooks; inner step environment B does not.

```
A.onEnter                  → host
                             (A's own onEnter is never wrapped)

B.onEnter                  → resolve B.onEnter against host symtab
                             → seed Env.Wrapped.{Name=B, Command, Args,
                                                 Environment, Timeout}
                             → run A.onWrapEnter as the actual action

Task t1.onRun              → resolve t1.onRun against task symtab
                             → seed Task.{Command, Args, Environment, Timeout}
                             → run A.onWrapTaskRun

Task t2.onRun              → same as t1, fresh overlay each task

B.onExit                   → resolve B.onExit against host symtab
                             → seed Env.Wrapped.* (with B.onExit's command/args)
                             → run A.onWrapExit

A.onExit                   → host
                             (A's own onExit is never wrapped)
```

If at any point the inner action sets `runOnHost: true`, the dispatch
short-circuits — the inner action runs directly on the host with no
overlay, no wrap.

## Cancellation passes through the wrap hook

RFC 0008 §Cancelation behavior is explicit: the wrap hook's own
`<Cancelation>` governs cancellation of any wrapped action. The
session runtime honours this naturally because the *wrap* action is
what actually got spawned — its cancelation method is on the action
the runtime is tracking. The inner action's cancelation is not surfaced.

If you need to propagate the inner timeout into the wrapped execution
context, the wrap script reads `Env.Action.Timeout` (for task wraps)
or `Env.Wrapped.Timeout` (for enter/exit wraps) and passes it to
e.g. `docker container stop --timeout`. That's where the integer
travels through.

## Exercise

```sh
# 1. Run the wrap dispatch tests.
cargo test -p openjd-sessions on_wrap

# 2. Open one of the integration tests that toggles wrap on:
$EDITOR crates/openjd-sessions/tests/integration/test_session.rs
# Search for `on_wrap_task_run: Some(...)` and read the surrounding
# test. Notice how the test asserts on the resolved command line,
# not the format string.

# 3. Trace overlay_task_wrap_symbols. Set a breakpoint or add a dbg!()
#    on the line that sets Task.Command. Run the test. Confirm the
#    string you see is the *resolved* inner command, not the format
#    string source.

# 4. Confirm the single-layer rule.
./target/release/openjd-rs check <<'YAML'
specificationVersion: jobtemplate-2023-09
extensions: [WRAP_ACTIONS]
name: TwoWraps
jobEnvironments:
  - name: Outer
    script:
      actions:
        onEnter: { command: "true", args: [] }
        onWrapTaskRun: { command: "echo", args: ["{{Task.Command}}"] }
  - name: AlsoOuter
    script:
      actions:
        onEnter: { command: "true", args: [] }
        onWrapEnter: { command: "echo", args: ["{{Env.Wrapped.Name}}"] }
steps:
  - name: S
    script:
      actions:
        onRun: { command: "echo", args: ["hi"] }
YAML
# Should fail with "only one environment in the session stack may
# define any of onWrapEnter, onWrapTaskRun, onWrapExit (RFC 0008)."
```

## What's next

[17 — Python ↔ Rust mapping](./17-python-vs-rust-mapping.md). Now that
you understand wrap routing in Rust, the Python comparison sketches
where the equivalent (or absent) logic lives in
`openjd-sessions-for-python`.
