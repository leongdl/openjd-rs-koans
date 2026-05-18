# 04 — Embedded files materialized to disk

**Use case:** A step has

```yaml
script:
  embeddedFiles:
    - name: prepare
      type: TEXT
      filename: "prepare.sh"
      runnable: true
      data: |
        #!/usr/bin/env bash
        echo "frame={{Task.Param.Frame}}" > out.txt
    - name: render
      type: TEXT
      filename: "render.py"
      data: |
        import subprocess
        subprocess.check_call(["bash", "{{Task.File.prepare}}"])
        # ... do the render
  actions:
    onRun:
      command: python
      args: ["{{Task.File.render}}"]
```

`render.py` references `Task.File.prepare` (the path of its sibling
embedded file). For that to work, **the path of `prepare` must exist
in the symtab before `render`'s `data:` is interpolated**. The
two-phase materialization makes this work.

## Two phases, two reasons

1. **`allocate_file_paths`** walks every embedded file and **picks a
   path** for each (without writing the body yet), updating the
   symtab so all `Task.File.*` / `Env.File.*` symbols are bound.

2. **`write_file_contents`** walks the same files, **resolves
   `data:`** against the now-complete symtab, and writes bytes.

Phase 1 must finish first because phase 2's `data:` interpolations
can reference any sibling's `Task.File.*`.

## Call stack

1. **`StepScriptRunner::run(...)`** —
   [runner/step_script.rs:114](../../../openjd-rs/crates/openjd-sessions/src/runner/step_script.rs#L114).
   After resolving step-level let bindings:

   ```rust
   if let Some(files) = &script.embedded_files {
       let mut ef = EmbeddedFiles::new(
           EmbeddedFilesScope::Step,        // → "Task.File" symtab key prefix
           self.base.files_directory.clone(),
           &self.base.session_id,
       ).with_user(self.base.user.clone());
       ef.allocate_file_paths(files, &mut final_symtab)?;
       ef.write_file_contents(&final_symtab, library)?;
   }
   ```

   `EnvScriptRunner` does the equivalent with `EmbeddedFilesScope::Env`
   → `Env.File` prefix.

2. **`EmbeddedFiles::allocate_file_paths(files, symtab)`** —
   [embedded_files.rs:253](../../../openjd-rs/crates/openjd-sessions/src/embedded_files.rs#L253).
   For each file:

   - **If `filename:` is set**, resolve it as a string (against the
     symtab as it currently stands). The filename can itself be an
     interpolation, but the spec restricts it to refer only to
     symbols knowable before this point — typically literal or
     pulling from `Task.Param.*`.
   - **Validate** the resolved filename: must be a plain basename, no
     path separators. The model layer rejects raw `/` or `\` in
     filename templates, but this is defense-in-depth in case a
     resolved value sneaks one in (e.g., `Task.Param.X` containing
     `..` after path mapping). On failure → `EmbeddedFilePath` error.
   - Joining `target_directory + filename` produces the path.
   - **If `filename:` is absent**, generate a random hex name
     (`random_hex_filename()`), pre-create an empty file with mode
     `0o600` (Unix), and use that path. This is the fallback when
     the user doesn't care what the file is called.
   - Insert `Task.File.<name>` (or `Env.File.<name>`) into the symtab
     as `ExprValue::Path` with host format.
   - Stash a `FileRecord { _symbol, filename, file: file.clone() }`
     for phase 2.

3. **`EmbeddedFiles::write_file_contents(symtab, library)`** —
   [embedded_files.rs:326](../../../openjd-rs/crates/openjd-sessions/src/embedded_files.rs#L326).
   For each `FileRecord`:

   - If `data:` is set, resolve it via `resolve_string_with(symtab,
     opts)` against the **fully populated** symtab — every sibling
     `Task.File.*` is now bound, so `data:` can reference any of
     them.
   - Write to disk via `write_embedded_file_with_options(filename,
     resolved, runnable, end_of_line)`:
     - Writes the bytes, applying `end_of_line` line-ending
       conversion if requested.
     - If `runnable: true`, sets executable bits (Unix) — without
       this, the script can't be `bash`-invoked directly.
     - Cross-user mode: actual writes happen via the helper, so the
       file ends up owned by the target user.

4. **At action time**, `resolve_action_args` ([koan 03](03-resolve-action-to-argv.md))
   runs against the same symtab; references like
   `{{Task.File.render}}` resolve to the host path that was bound in
   phase 1.

## What goes where on disk

`SessionConfig.files_directory` defaults to a session-scoped temp dir
([tempdir.rs](../../../openjd-rs/crates/openjd-sessions/src/tempdir.rs)).
Files materialized for a step go directly into that directory. There
is **no per-step subdirectory** — sibling step's embedded files share
the same directory. With user-supplied filenames this means a step
named `A` declaring `filename: "common.txt"` and a step named `B`
declaring `filename: "common.txt"` would collide *in the same session*.
This is intentional: per-task fresh sessions are the spec's recommended
isolation boundary; if the user wants step-scoped uniqueness they
should make filenames distinct or omit `filename:` to get a random
name.

Environment-scope embedded files live in the same directory; the
`Task.File` / `Env.File` symtab prefix is the only thing that
distinguishes them at lookup time.

## What `runnable: true` actually does

[`write_embedded_file_with_options` in embedded_files.rs](../../../openjd-rs/crates/openjd-sessions/src/embedded_files.rs)
sets mode `0o755` (Unix) when `runnable: true`. Without it, the file
is `0o644`. This is independent of `filename:` extension — a `.py`
file with `runnable: true` gets executable bits, a `.sh` file
without it doesn't.

`runnable` only matters when something else is going to invoke the
file directly (e.g., `bash some.sh` works on a non-runnable file
because bash reads it; `./some.sh` requires the executable bit). The
canonical SimpleAction-desugared step
([template parsing koan 08](../template-parsing/08-simpleaction-syntax-sugar.md))
sets `runnable: true` because the interpreter command takes the
filename as an arg (`bash <file>`), but a hand-rolled step might
launch the script via `./` and need the bit.

## Cleanup

`EmbeddedFiles` records are dropped when the runner finishes, but the
files themselves stay in `files_directory` until the session ends.
`Session::cleanup` ([session.rs:755](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L755))
removes the entire `files_directory` (and `working_directory`) when
called — this is what the worker invokes after the session is done.

## Related koans

- [01 — Session orchestration](01-session-orchestration.md) — where
  embedded-file materialization fits in the larger flow.
- [03 — Resolve action to argv](03-resolve-action-to-argv.md) — how
  `Task.File.*` symbols get used after phase 2.
- [Template parsing 06 — Symbol-table seeding](../template-parsing/06-symbol-table-seeding-three-scopes.md) —
  the validate-time treatment of `Task.File.*` / `Env.File.*` as
  unresolved PATH symbols.
- [Template parsing 08 — SimpleAction syntax sugar](../template-parsing/08-simpleaction-syntax-sugar.md) —
  the de-sugarer that creates embedded files for `bash:` / `python:` /
  etc.
