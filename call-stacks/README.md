# Call-stack walkthroughs

Step-by-step traces from a user-visible event down through the
openjd-rs code. Each walkthrough names the file path and line for
every frame so a reader can click through in their editor.

## [Template parsing](template-parsing/)

How a YAML/JSON template becomes a typed, validated, instantiated
`Job` ready to run.

- [01 — Parser → template data structure](template-parsing/01-parser-to-template-data-structure.md)
- [02 — Parser → environment template](template-parsing/02-parser-to-environment-template.md)
- [03 — Expression block end-to-end (string → AST → resolved value)](template-parsing/03-expression-block-end-to-end.md)
- [04 — Format string segment scanner (`{{ ... }}`)](template-parsing/04-format-string-segment-scanner.md)
- [05 — Python keyword-after-dot rewrite](template-parsing/05-python-keyword-after-dot-rewrite.md)
- [06 — Symbol-table seeding across three scopes](template-parsing/06-symbol-table-seeding-three-scopes.md)
- [07 — Parameter merge across env templates → symbol table](template-parsing/07-parameter-merge-and-symbol-table-seed.md)
- [08 — SimpleAction syntax sugar (`bash:` / `python:` / …)](template-parsing/08-simpleaction-syntax-sugar.md)
- [09 — Step dependency graph + topological sort](template-parsing/09-step-dependency-graph-and-topo-sort.md)
- [10 — Lazy task-parameter cartesian product](template-parsing/10-task-parameter-space-iteration.md)

## [Runtime](runtime/)

How a validated `Job` becomes a real subprocess on a worker, with
per-action symbol-table assembly, embedded files materialized,
environment variables propagated, and stdout directives folded back
into session state.

- [01 — Running a session, top to bottom](runtime/01-session-orchestration.md)
- [02 — Building a runtime symbol table from a `Job`](runtime/02-build-symbol-table.md)
- [03 — Resolving an action to argv](runtime/03-resolve-action-to-argv.md)
- [04 — Embedded files materialized to disk](runtime/04-embedded-files-materialization.md)
- [05 — Path-mapping rules at session start](runtime/05-path-mapping-materialization.md)
- [06 — `openjd_env` stdout directive flow](runtime/06-openjd-env-stdout-directives.md)
- [07 — Resolving wrap actions (RFC 0008)](runtime/07-wrap-action-resolution.md)
