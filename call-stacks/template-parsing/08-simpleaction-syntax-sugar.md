# 08 — SimpleAction syntax sugar (`bash:` / `python:` / `cmd:` / `powershell:` / `node:`)

**Use case:** A user writes:

```yaml
steps:
  - name: render
    bash:
      script: |
        #!/usr/bin/env bash
        echo "Hello $1"
      args: ["{{Task.Param.Frame}}"]
      timeout: "{{Param.Timeout}}"
```

…instead of the verbose `script:` + `embeddedFiles:` + `actions.onRun`
block. How does this become a `StepScript` indistinguishable from one
the user typed by hand?

This is the FEATURE_BUNDLE_1 syntax-sugar feature. It happens during
job creation, not at parse time, so the `JobTemplate` still carries
the user-typed shape; only the materialized `Job` has the de-sugared
form.

## Call stack

1. **`instantiate_step(st, &symtab, has_expr, &limits, ctx)`** —
   [openjd-rs/crates/openjd-model/src/job/create_job/instantiate.rs:20](../../../openjd-rs/crates/openjd-model/src/job/create_job/instantiate.rs#L20).
   Triggered for every step during `create_job`. Pre-step bookkeeping
   (let bindings, `Step.Name`), then asks the step "what's your
   effective script?":

   ```rust
   let script_template = st.resolve_syntax_sugar()?.or_else(|| st.script.clone());
   ```

2. **`StepTemplate::resolve_syntax_sugar()`** —
   [openjd-rs/crates/openjd-model/src/template/step.rs:60](../../../openjd-rs/crates/openjd-model/src/template/step.rs#L60).
   The de-sugar itself. Logic:

   - **If `script:` is already set** → just `Ok(Some(script.clone()))`.
     User-authored explicit form takes precedence; the sugar fields
     are mutually exclusive with `script:` (enforced at validation
     time, not here).

   - **Otherwise**, walk this priority-ordered table:
     ```rust
     [("python", ".py", &[],          self.python),
      ("bash",   ".sh", &[],          self.bash),
      ("cmd",    ".bat", &["/C"],     self.cmd),
      ("powershell", ".ps1", &["-File"], self.powershell),
      ("node",   ".js", &[],          self.node)]
     ```
     For the first non-`None` entry, build:

     1. A safe filename slug from the step name (`alphanumeric → keep,
        else → '_'`, capped at 200 chars; if it starts with a digit,
        prepend `_`). Stops collisions with shell metacharacters and
        avoids `0_step` paths some tools mishandle.

     2. An `embedded_name` = `<safe>_script` and `filename` =
        `<embedded_name><ext>`.

     3. An `Action` whose:
        - `command:` is the literal interpreter name
          (`python` / `bash` / `cmd` / `powershell` / `node`) wrapped
          as a `FormatString` with no interpolation.
        - `args:` is `[interpreter_arg_prefix...] + ["{{Task.File.<embedded_name>}}"] + user_args`.
          For `cmd` this prepends `/C`; for `powershell`, `-File`; the
          others use no prefix.
        - `cancelation` and `timeout` come from the SimpleAction.

     4. An `EmbeddedFile`:
        - `name` = embedded_name
        - `filename` = the filename string (also a `FormatString`)
        - `data` = `FormatString::new(&sa.script)` — **this is the only
          place a parse error can surface**, mapped to a
          `DecodeValidation` error.
        - `runnable: Some(true)` — the runtime gives the file
          executable bits.
        - `file_type: Text`.

     5. The new `StepScript` carries the user's `let_bindings` from the
        SimpleAction, so `bash.let:` works identically to `script.let:`.

   - If none of the SimpleAction fields are set, return `Ok(None)` —
     the step has neither `script:` nor a sugar field. Validation
     would have already rejected this.

3. **`instantiate_step` continues** with the de-sugared `StepScript` —
   from this point downstream, the runtime sees a normal step with an
   `onRun` action that runs `bash <script-file> [user-args...]`.

## What this gives you

A user who writes:

```yaml
- name: render
  bash:
    script: |
      echo "Frame $1"
    args: ["{{Task.Param.Frame}}"]
```

ends up with the equivalent of:

```yaml
- name: render
  script:
    embeddedFiles:
      - name: render_script
        type: TEXT
        filename: "render_script.sh"
        runnable: true
        data: |
          echo "Frame $1"
    actions:
      onRun:
        command: bash
        args: ["{{Task.File.render_script}}", "{{Task.Param.Frame}}"]
```

…**at job-creation time**, not parse time. This means:

- The `JobTemplate` value the validator inspects still has
  `bash:` set, not `script:`. Validation rules that check whether
  a SimpleAction is permitted (FEATURE_BUNDLE_1 gating, `let:`
  attached to it) run against the original shape.

- Two callers can share a validated template and produce identical
  jobs — there's no "validate-then-rewrite-then-revalidate" cycle.

- A SimpleAction's `script` body must itself be parseable as a
  format string (since it can contain `{{...}}` interpolations like
  `{{Task.Param.Frame}}`). A malformed body fails at de-sugar time
  with `"SimpleAction script format string error: ..."`.

## Why is the prefix `["/C"]` for cmd and `["-File"]` for powershell?

These interpreters reject naked filenames. `cmd somefile.bat` opens
a new shell session. `cmd /C somefile.bat` runs the file and exits.
PowerShell needs `-File` to run a `.ps1` and not enter REPL mode.
Bash, Python, and Node all accept `interpreter <file> [args...]`
out of the box — no prefix needed.

## What stays user-facing

The step name in iteration error messages, embedded-file paths in
log output (`{{Task.File.<embedded>}}` resolves to a real path on
disk), and the `Task.File.<embedded_name>` symbol that's available
in the `args` interpolation. A user can, in principle, reference the
SimpleAction's embedded file by name from within their `args:` —
useful for, e.g., a Bash script wanting to know its own path:

```yaml
- name: emitpath
  bash:
    script: 'echo "$0"'
    args: []
```

The `$0` argv slot will contain the resolved
`{{Task.File.emitpath_script}}` path because the de-sugarer placed it
there.

## Related

- [koan 03](03-expression-block-end-to-end.md) — the
  `FormatString::new(...)` calls that wrap each piece feed into the
  larger pipeline.
- [koan 06](06-symbol-table-seeding-three-scopes.md) — `Task.File.*`
  is in task scope; this is what makes the `{{Task.File.<embedded>}}`
  reference resolve.
