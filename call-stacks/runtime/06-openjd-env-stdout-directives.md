# 06 — `openjd_env` stdout directive flow

**Use case:** A user's script writes:

```bash
echo "Computed dir: $TMPDIR/scene"
echo "openjd_env: SCENE_DIR=$TMPDIR/scene"
```

The next action in the same session — even one running a different
command in a different language — should see `SCENE_DIR` in its
environment. How does that happen, given that the writing process
exits before the next action starts?

This is the **out-of-band channel from a child process to its
session**: a small set of OpenJD-defined directives parsed out of
stdout and folded back into session state.

## The directives

```
openjd_progress: 50.0
openjd_status: rendering frame 42
openjd_fail: scene not found
openjd_env: KEY=value                 ← persists into next action's env
openjd_redacted_env: SECRET=swordfish ← like env, but value redacted in logs
openjd_unset_env: KEY                 ← removes from next action's env
openjd_session_runtime_loglevel: DEBUG
```

All seven are recognized by `parse_directive`
([action_filter.rs:45](../../../openjd-rs/crates/openjd-sessions/src/action_filter.rs#L45)).
A line starts with `openjd_<name>: ` (note: literal `<name>` followed
by colon-space) and the rest is the payload.

This koan focuses on `openjd_env` because it's the most consequential
— it actually mutates session state for **subsequent** actions.

## Call stack

The producer side is the user's script writing a stdout line. The
consumer side runs in the worker:

1. **The subprocess writes a line to stdout.**

2. **`run_subprocess`** ([subprocess.rs:442](../../../openjd-rs/crates/openjd-sessions/src/subprocess.rs#L442))
   reads stdout line-by-line via a tokio async reader. Each line is
   passed to `ActionFilter::filter_message`.

3. **`ActionFilter::filter_message(message, session_id)`** —
   [action_filter.rs:130](../../../openjd-rs/crates/openjd-sessions/src/action_filter.rs#L130).
   Three return values: `(callbacks, pass_through, modified_message)`.

   - **`parse_directive(message)`** is called first
     ([action_filter.rs:45](../../../openjd-rs/crates/openjd-sessions/src/action_filter.rs#L45)):
     - Strip `openjd_` prefix; require `: ` separator; pull the kind
       and payload.
     - Map kind string to `ActionMessageKind` (Progress, Status, Fail,
       Env, RedactedEnv, UnsetEnv, SessionRuntimeLoglevel). Anything
       else returns `None`.
   - For `Env` directives (`openjd_env: KEY=value`):
     - Parse `KEY=value` via `ENVVAR_SET_REGEX`
       ([action_filter.rs:82](../../../openjd-rs/crates/openjd-sessions/src/action_filter.rs#L82)).
       Quoted form `"KEY=value"` is allowed.
     - Build a `FilterCallback { kind: Env, value:
       SetEnv { name, value }, cancel: false }`.
   - **`pass_through = false`** for valid directive lines — the line
     is consumed and not echoed to the session log unless
     `echo_openjd_directives` is set.
   - **Malformed near-misses** (`openjd_env: foo` with no `=`,
     `openjd_envXfoo`, etc.) are detected by `is_malformed_env_command`
     and annotated in the log with `-- ERROR: ...` so the user sees
     why their directive was ignored.

4. **The runner pushes each callback into `message_tx`**:
   `mpsc::UnboundedSender<ActionMessage>`. This channel is the
   conduit from the per-process `ActionFilter` back up to the
   session.

5. **`Session::drive_action(...)`** —
   [session.rs:1722](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L1722).
   The `tokio::select!` loop:

   ```rust
   loop {
       tokio::select! {
           biased;
           msg = rx.recv(), if result.is_none() => {
               if let Some(msg) = msg { self.apply_message(msg, identifier) }
           }
           r = &mut action_fut, if result.is_none() => {
               result = Some(r);
           }
           else => break,
       }
       if result.is_some() {
           // Drain remaining messages
           while let Ok(msg) = rx.try_recv() {
               self.apply_message(msg, identifier);
           }
           break;
       }
   }
   ```

   `biased;` makes the message arm priority over the action future,
   so messages don't get dropped if the future completes in the same
   tick. After the future resolves, `try_recv` drains anything still
   in the channel before the loop exits — important when the
   subprocess wrote `openjd_env` lines just before `exit(0)`.

6. **`Session::apply_message(msg, identifier)`** —
   [session.rs:1813](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L1813).
   Per-message side effect:

   ```rust
   ActionMessage::SetEnv { name, value } => {
       let key = normalize_env_key(&name);
       self.env_vars.insert(key.clone(), value.clone());
       if let Some(changes) = self.created_env_vars.get_mut(identifier) {
           changes.insert(key, Some(value));
       }
   }
   ActionMessage::UnsetEnv { name } => {
       let key = normalize_env_key(&name);
       self.env_vars.remove(&key);
       if let Some(changes) = self.created_env_vars.get_mut(identifier) {
           changes.insert(key, None);
       }
   }
   ActionMessage::RedactedEnv { name, value } => {
       if self.redactions_enabled() {
           ... // same as SetEnv
       }
       self.redacted_values.insert(value);  // future log lines redact this string
   }
   ```

   Two state updates per env directive:
   - **`self.env_vars`** — the cumulative current env, used by the
     next action.
   - **`self.created_env_vars[identifier]`** — per-environment
     mutation log, so when an environment is later **exited** the
     mutations made *during* that environment can be reverted.

7. **The next action runs** via `Session::run_task` (or another
   `enter_environment`). When it reaches `evaluate_env_vars`
   ([session.rs:1920](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L1920)),
   it folds together: process env, optional `os_env_vars`, and
   `self.env_vars`. The new `SCENE_DIR=...` value is part of that
   merge and ends up in the `SubprocessConfig.env_vars` for
   `run_subprocess`.

## How environment exit reverts mutations

`Session::exit_environment(identifier)` finds the
`created_env_vars[identifier]` map. For each `(key, optional_value)`:

- `Some(value)` → the env had set this key. Restore the previous
  value (or remove it if the env was the one that originally set it).
- `None` → the env had unset this key. Restore the previous value if
  there was one in any prior environment's bookkeeping.

The bookkeeping is per-environment, not session-global, so the
**LIFO undo order** during exit gives you back exactly the env state
from before the environment was entered. This matches the spec's
"environments compose" rule — exiting an inner env shouldn't leak
its env-var mutations.

## Why pass via stdout instead of a separate channel

Because **the child process is fully sandboxed against the worker** —
it doesn't have a direct API into the worker's memory. Stdout is the
one channel guaranteed to reach the worker on every platform, and a
text directive with a stable prefix is parseable by any language.

A C program, a Bash script, a Python invocation, a Lua plugin — all
can write `openjd_env: KEY=value\n` and have it work. There's no SDK
required. This is the same pattern as Buck2's "worker protocol" or
GitHub Actions' workflow commands (`::set-output::`).

## Cancellation and failure directives

Two extra directives don't fit into the env model:

- **`openjd_fail: <message>`** — sets `self.action.fail_message`. The
  action's exit code is independent: a script can declare itself
  failed even if `exit(0)` would otherwise be reported as success.
  Used when a script wants to report a structured reason for failure.
- **`ActionMessage::CancelMarkFailed`** — internally generated when
  certain conditions are met. Sets `cancel.mark_failed = true`, which
  in `drive_action` causes a `Canceled` final state to be reported as
  `Failed`. This is how `cancelation: NotifyThenTerminate` interacts
  with a script that responds to the notify by failing cleanly.

## Related koans

- [01 — Session orchestration](01-session-orchestration.md) —
  `drive_action` and `apply_message` are at the heart of the
  orchestration loop.
- [03 — Resolve action to argv](03-resolve-action-to-argv.md) — the
  per-action `evaluate_env_vars` consumes the directives that earlier
  actions wrote.
- [Template parsing 03 — Expression block end-to-end](../template-parsing/03-expression-block-end-to-end.md) —
  this isn't an expression-evaluation channel, it's an out-of-band
  state mutation channel; useful contrast.
