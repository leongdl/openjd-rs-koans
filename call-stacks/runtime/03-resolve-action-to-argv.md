# 03 — Resolving an action to argv

**Use case:** A step's `onRun` is:

```yaml
onRun:
  command: "{{ Task.Param.Interpreter }}"
  args:
    - "{{ Task.File.script }}"
    - "--frame={{ Task.Param.Frame }}"
    - "{{ if(Param.Verbose, '--verbose', None) }}"
    - "{{ Param.ExtraArgs }}"
  timeout: "{{ Param.Timeout * 60 }}"
```

`Param.Verbose` is `false`, `Param.ExtraArgs` is `["--quality", "high"]`,
`Task.Param.Frame` is `42`, `Task.Param.Interpreter` is `"python"`,
`Task.File.script` resolves to `/var/lib/openjd/sess123/files/script.py`,
`Param.Timeout` is `5`. What argv does `tokio::process::Command`
actually receive?

Answer: `["python", "/var/lib/openjd/sess123/files/script.py",
"--frame=42", "--quality", "high"]` with a 300-second timeout.

This koan walks the resolution helpers that produce that argv.

## Call stack

1. **`Runner::run_action(action, symtab, lib, env_vars, message_tx, default_timeout, default_cancel_period)`** —
   [runner/mod.rs:142](../../../openjd-rs/crates/openjd-sessions/src/runner/mod.rs#L142).
   Two resolution calls before subprocess launch:

   ```rust
   let args = resolve_action_args(action, symtab, library)?;
   let timeout = resolve_action_timeout(action, symtab, library, default_timeout)?;
   let cancel_method = cancel_method_for_action(&action.cancelation, default_cancel_period);
   ```

2. **`resolve_action_args(action, symtab, library)`** —
   [runner/mod.rs:268](../../../openjd-rs/crates/openjd-sessions/src/runner/mod.rs#L268).
   Produces `Vec<String>` with command at `[0]`. Sequence:

   - **Resolve `command`** as `String`:
     ```rust
     let command = action.command.resolve_string_with(
         symtab,
         &FormatStringOptions::new().with_library(library),
     )?;
     let mut args = vec![command];
     ```
     `resolve_string_with` always produces a `String` — single
     interpolation `{{Task.Param.Interpreter}}` returns the typed
     value coerced to display string.

   - **Walk each `args[]` entry** with `resolve_with` (typed) **first**,
     falling back to `resolve_string_with` (string) on error. Why two
     attempts? Because `resolve_with` returns a typed `ExprValue`,
     which lets us:

     - Skip `Null` values entirely:
       ```yaml
       "{{ if(Param.Verbose, '--verbose', None) }}"
       ```
       evaluates to `Null` when `Param.Verbose` is false. Argument
       collapses out of the argv. **Useful pattern for optional
       flags.**

     - Flatten **list values** in place:
       ```yaml
       "{{ Param.ExtraArgs }}"
       ```
       resolves to `ListString(["--quality", "high"], ...)`. Each
       element is pushed individually to `args`, so a single
       `args[]` entry can contribute multiple argv slots. **This
       is how variadic args work.**

     - Otherwise, push `val.to_display_string()` — same as
       `resolve_string_with` would produce.

   - The fallback to `resolve_string_with` matters when an arg has
     mixed literal+expression segments like `"--frame={{ Task.Param.Frame
     }}"`. `resolve_with` calls `resolve_inner`, which falls through
     to `resolve_string_with` for multi-segment cases anyway, so this
     case still works through the typed path.

3. **`resolve_action_timeout(action, symtab, library, default)`** —
   [runner/mod.rs:234](../../../openjd-rs/crates/openjd-sessions/src/runner/mod.rs#L234).
   Different shape because the result is a `Duration`, not a string:

   - If `action.timeout` is `None`, return the caller-supplied
     `default`.
   - Otherwise resolve as string, then `parse::<u64>()` it as
     seconds.
   - **Reject zero**: `"timeout must be a positive integer, got '0'"`.
     This catches `{{ Param.Timeout * 0 }}` and similar.
   - Return `Some(Duration::from_secs(secs))`.

4. **`cancel_method_for_action(cancelation, default_period)`** —
   [runner/mod.rs:212](../../../openjd-rs/crates/openjd-sessions/src/runner/mod.rs#L212).
   Maps the typed `CancelationMode`:
   - `None` or `Terminate` → `CancelMethod::Terminate`.
   - `NotifyThenTerminate { notify_period_in_seconds: Some(fs) }` →
     parse `fs.raw()` as `u64` (note: **doesn't go through expr
     evaluation** — this field is read raw as integer seconds). Falls
     back to `default_period` on parse failure.

5. **`SubprocessConfig` is built** from `args` + `timeout` +
   `cancel_method` + the working dir, env vars, and user. Then
   either `run_via_helper` (cross-user) or `run_subprocess`
   ([subprocess.rs:442](../../../openjd-rs/crates/openjd-sessions/src/subprocess.rs#L442))
   does the actual spawn.

## What `resolve_string_with` actually does

[`format_string.rs:136`](../../../openjd-rs/crates/openjd-expr/src/format_string.rs#L136).
For each `Segment`:

- `Literal(s)` → push to result.
- `Expression { parsed, ... }` → call `eval_parsed`, which builds a
  `EvalBuilder` with the host library, a `PathFormat`, and runs
  `parsed.with_*().evaluate(&[symtab])`. Result is converted via
  `to_display_string()`:
  - String → as-is.
  - Int → `"42"`.
  - Bool → `"true"` / `"false"`.
  - Path → just the path string (no scheme).
  - List → `[a, b, c]` style display string.
  - **Null → empty string** (skipped via the `!matches!(val, ExprValue::Null)`
    guard at [format_string.rs:153](../../../openjd-rs/crates/openjd-expr/src/format_string.rs#L153)).

## What `resolve_with` adds on top

Same code path, but for **single-expression** format strings (one
`{{...}}` and nothing else), it returns the **raw `ExprValue`**
without coercing to string. This is what enables the list-flatten and
null-skip behavior in `resolve_action_args`. A two-segment string
like `"--frame={{X}}"` still gets stringified.

## End-to-end for the example

```text
action.command = FormatString { raw: "{{ Task.Param.Interpreter }}", segments: [Expression] }
action.args = [
  FormatString("{{ Task.File.script }}"),                 ── 1 segment, Path value
  FormatString("--frame={{ Task.Param.Frame }}"),         ── 2 segments, multi → string
  FormatString("{{ if(Param.Verbose, '--verbose', None) }}"),  ── 1 segment, Null
  FormatString("{{ Param.ExtraArgs }}"),                  ── 1 segment, List value
]
action.timeout = FormatString("{{ Param.Timeout * 60 }}")

resolve_action_args:
  resolve command → "python"
  args = ["python"]
  args[0] resolve_with → Path value → push "/var/lib/openjd/.../script.py"
  args[1] resolve_with → falls through to string for multi-seg → push "--frame=42"
  args[2] resolve_with → Null → continue (skipped)
  args[3] resolve_with → List → flatten → push "--quality", push "high"
  → args = ["python", "/var/lib/openjd/.../script.py", "--frame=42", "--quality", "high"]

resolve_action_timeout:
  resolve "{{ Param.Timeout * 60 }}" → "300" → parse → Duration(300s)
```

## Why this design beats string concatenation

Before this layer, an OpenJD template author writing a "wrap or pass-
through" pattern would have to manage shell escaping themselves:

```yaml
# Hypothetical pre-typed-args design:
args: ["{{ if(Param.Verbose, '--verbose ' + Param.ExtraArgs.join(' '), Param.ExtraArgs.join(' ')) }}"]
```

With typed args, the same pattern is one expression that yields a
list, and the resolver flattens. Argument quoting is handled by
`tokio::process::Command::args` (which does proper exec-args
escaping); the user never sees a string-shell layer.

## Related koans

- [01 — Session orchestration](01-session-orchestration.md) — where
  this fits in the larger session lifecycle.
- [02 — Building a runtime symbol table](02-build-symbol-table.md) —
  the symtab passed in here.
- [Template parsing 04 — Format string segment scanner](../template-parsing/04-format-string-segment-scanner.md) —
  what the `Segment::Expression` parsing does at decode time.
