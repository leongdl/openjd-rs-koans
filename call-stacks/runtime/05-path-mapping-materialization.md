# 05 — Path-mapping rules at session start

**Use case:** A worker holds path-mapping rules like:

```
source_path_format: POSIX
source_path: "/mnt/share/projects"
destination_path: "Z:\\projects"
```

A job declares `Param.OutputDir: PATH = "/mnt/share/projects/render-out"`.
On a Windows worker, the user's command at runtime should see
`Z:\projects\render-out`, and a script that wants to load **the rules
themselves** should be able to find them. Both happen via path
mapping: the *PATH parameter rewrite* and the *rules-file
materialization*.

This koan covers the second one: how rules end up as a JSON file on
disk so that `{{Session.PathMappingRulesFile}}` in a script resolves
to a real path the script can `cat` / parse.

## Two operations, one set of rules

`Session::with_config` accepts `Vec<PathMappingRule>` via
[`with_path_mapping`](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L552).
Those rules drive two distinct operations:

1. **`apply_path_mapping_to_string` / `_to_value`** —
   [session.rs:2122–2162](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L2122).
   Used during `build_symbol_table` ([koan 02](02-build-symbol-table.md))
   to rewrite PATH `Param.*` and `Task.Param.*` values from
   submission paths to host paths.

2. **`materialize_path_mapping`** ([session.rs:1861](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L1861))
   — writes a JSON file containing the full set of rules to disk and
   sets two symtab entries pointing at it. Used by scripts that need
   to do path-mapping themselves (e.g., a Python script that opens a
   file containing more paths and rewrites them in turn).

## Call stack

`materialize_path_mapping` is called from the action-resolution path,
not session construction:

1. **`Session::run_task`** —
   [session.rs:1356](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L1356)
   and **`Session::enter_environment_with_output`** at the equivalent
   point. After cloning `symtab` into `action_symtab`, both call:
   ```rust
   self.materialize_path_mapping(&mut action_symtab)?;
   ```

2. **`Session::materialize_path_mapping(symtab)`** —
   [session.rs:1861](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L1861).
   Logic:

   - **Build JSON payload**:
     - If rules exist:
       ```json
       {
         "version": "pathmapping-1.0",
         "path_mapping_rules": [
           {
             "source_path_format": "POSIX",
             "source_path": "/mnt/share/projects",
             "destination_path": "Z:\\projects"
           }
         ]
       }
       ```
       `source_path_format` is one of `"POSIX"`, `"WINDOWS"`, `"URI"`.
     - If rules are empty: `"{}"` — empty object, not the wrapped
       envelope. This is the spec's wire format for "no rules
       active." A script reading the file can detect emptiness by
       absence of `path_mapping_rules`.

   - **Set `Session.HasPathMappingRules`** as a `Bool` symbol —
     scripts can check this before bothering to read the file:
     ```python
     if {{Session.HasPathMappingRules}}: ...
     ```

   - **Write the JSON** to a fresh path:
     ```rust
     let filename = self.working_directory.join(format!(
         "pathmapping_{}.json",
         uuid::Uuid::new_v4().simple()
     ));
     std::fs::write(&filename, &rules_json)?;
     ```
     Fresh UUID per call → safe across concurrent calls; no cleanup
     needed (the working dir is wiped at session end).

   - **Set `Session.PathMappingRulesFile`** as a `Path` symbol with
     host format. The action's `args:` can reference it:
     ```yaml
     command: my-tool
     args: ["--path-mapping={{Session.PathMappingRulesFile}}"]
     ```

3. **The action runs** — `resolve_action_args` ([koan 03](03-resolve-action-to-argv.md))
   sees the now-populated `Session.*` symbols and resolves the
   `{{Session.PathMappingRulesFile}}` interpolation to the path
   `materialize_path_mapping` just wrote.

## Why per-action instead of once-per-session

Two reasons:

- **Adding rules mid-session is allowed.**
  [`extend_path_mapping_rules`](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L561)
  lets a caller append rules after construction. The next action will
  see the appended rules in its JSON file. If the file were materialized
  once at session start, late additions would be invisible.

- **Each action gets its own UUID-named file.** Concurrent reads from
  multiple actions in different sessions on the same worker can't
  trip over each other.

## Why `Posix` is the canonical format internally

PATH `Param.*` values are stored in template scope using
`PathFormat::Posix` ([instantiate.rs](../../../openjd-rs/crates/openjd-model/src/job/create_job/instantiate.rs)).
This is the spec's neutral form: a Posix-style path that's normalized
to the host's native format only at `build_symbol_table` time, when
the host actually knows what it is.

Path-mapping rules carry their **source** format explicitly because
the path *being rewritten* may not be in the host's native format —
it may be a path from a Linux submitter being processed on a Windows
worker. The rule says "if you see paths shaped like POSIX with this
prefix, rewrite to this destination."

## What `apply_path_mapping_to_string` does

[session.rs:2122](../../../openjd-rs/crates/openjd-sessions/src/session.rs#L2122).
Iterates the rules in declared order; the **first match wins** via
each rule's `apply()` method ([openjd-expr/src/path_mapping.rs](../../../openjd-rs/crates/openjd-expr/src/path_mapping.rs)).
Order matters: if two rules both match a given path (e.g., a more
specific rule and a more general one), the one declared first
wins.

If no rule matches, the input is returned unchanged. This is why a
worker without rules works identically to one with rules that don't
match any of the job's paths.

## What scripts do with the file

A typical use is a Python script that opens the rules file, parses
it, and applies the same rewrite logic to additional paths it
discovers at runtime (e.g., paths referenced inside a scene file
that the OpenJD spec couldn't have seen). The template ships:

```yaml
script:
  embeddedFiles:
    - name: rewrite
      type: TEXT
      filename: rewrite.py
      data: |
        import json, sys, pathlib
        with open(sys.argv[1]) as f: m = json.load(f)
        rules = m.get("path_mapping_rules", [])
        # ... apply to discovered paths in scene file ...
  actions:
    onRun:
      command: python
      args:
        - "{{Task.File.rewrite}}"
        - "{{Session.PathMappingRulesFile}}"
        - "{{Param.SceneFile}}"
```

Because the script gets the rules in a stable JSON format with a
`version:` field, it can be written once and ship across spec
revisions; new rule fields would bump the version.

## Related koans

- [01 — Session orchestration](01-session-orchestration.md)
- [02 — Building a runtime symbol table](02-build-symbol-table.md) —
  where `apply_path_mapping_to_string` is called in the parameter
  rewrite path.
- [Template parsing 06 — Symbol-table seeding](../template-parsing/06-symbol-table-seeding-three-scopes.md) —
  validate-time treatment of `Session.HasPathMappingRules` /
  `Session.PathMappingRulesFile` as unresolved BOOL/PATH symbols.
