# 17 — Python ↔ Rust mapping

## What this is

A module-by-module table comparing `openjd-sessions-for-python` to the
Rust `openjd-sessions` crate, plus shorter tables for the model and
expression layers. If you came from the Python side you already know
the concepts; this doc is the index that points at the equivalent
file in the Rust tree.

The two implementations are *parallel*, not bridged. There is no FFI;
they share the same conformance test suite and the same RFCs but
otherwise evolve independently. Where they diverge, the divergence is
called out.

## Read these in order

- `openjd-sessions-for-python/src/openjd/sessions/__init__.py` — the
  Python public surface.
- `openjd-rs/crates/openjd-sessions/src/lib.rs` — the Rust public
  surface.
- `openjd-model-for-python/src/openjd/model/` — Python model layer.
- `openjd-rs/crates/openjd-model/src/lib.rs` — Rust model layer.

## Sessions: file-by-file map

| Python (`openjd.sessions`) | Rust (`openjd_sessions`) | Shape difference |
|---|---|---|
| `_session.py: Session` | `session.rs: Session` | Python uses callbacks + threading; Rust uses `&mut self` + `async fn` + tokio channels. |
| `_session.py: SessionState` | `session.rs: SessionState` | Same enum names (`READY`, `RUNNING`, `CANCELING`, `READY_ENDING`, `ENDED`). |
| `_session.py: ActionStatus` | `action_status.rs: ActionStatus` | Same fields. |
| `_session.py: ActionState` | `action.rs: ActionState` | Same enum. |
| `_session.py: SessionCallbackType` | `session.rs: SessionCallbackType = Box<dyn Fn(...) + Send + Sync>` | Python: any callable. Rust: trait-object closure. |
| `_session_user.py: SessionUser, PosixSessionUser, WindowsSessionUser` | `session_user.rs: SessionUser` (trait), `PosixSessionUser` (cfg unix), `WindowsSessionUser` (cfg windows) | Python uses runtime checks; Rust uses `cfg` attributes. |
| `_session_user.py: BadCredentialsException` | `session_user.rs: BadCredentialsError` (cfg windows) | Naming convention: Python `*Exception`, Rust `*Error`. |
| `_runner_base.py` | `runner/mod.rs` | Shared runner logic. |
| `_runner_step_script.py` | `runner/step_script.rs: StepScriptRunner` | Per-task runner. |
| `_runner_env_script.py` | `runner/env_script.rs: EnvironmentScriptRunner` | Per-env enter/exit runner. |
| `_subprocess.py` | `subprocess.rs` (+ `cross_user_helper.rs`, `helper_binary.rs`) | Rust splits the cross-user IPC into a separate helper binary. |
| `_embedded_files.py` | `embedded_files.rs` | Same algorithm: write embedded `<EmbeddedFile>` content to a tempdir, expose paths via `Task.File.*`/`Env.File.*`. |
| `_logging.py: LOG, LogContent` | `logging.rs: log_section_banner, LogContent` | Python uses stdlib `logging`; Rust uses `log` + `tracing`. |
| `_path_mapping.py: PathFormat, PathMappingRule` | re-exported from `openjd_expr::path_mapping` | Same shape. Both libraries re-export from the expression layer. |
| `_action_filter.py` | `action_filter.rs` | `REDACTED_ENV_VARS` extension support. |
| `_tempdir.py` | `tempdir.rs` (+ `StickyBitPolicy`) | Same per-session tempdir, same sticky-bit policy. |
| `_os_checker.py` | (inlined into `cfg` gates) | Rust does platform dispatch at compile time. |
| `_linux/`, `_win32/` | `cfg(unix)` / `cfg(windows)` modules | Python uses subpackages; Rust uses cfg attributes. |
| `_windows_permission_helper.py` | `win32_permissions.rs` (gated by `test-utils` feature for tests) | Rust exposes the helper for tests; production code paths don't depend on it. |
| `_windows_process_killer.py` | inlined into `subprocess.rs` (cfg windows) | Same logic, different shape. |

## Model: file-by-file map

| Python (`openjd.model`) | Rust (`openjd_model`) | Notes |
|---|---|---|
| `__init__.py` exports | `lib.rs` exports | Both flatten template + job + types into a single module surface. |
| `_parse.py` | `template/parse.rs` | Same dispatch on `specificationVersion`; both validate extensions list before deserialization. |
| `_format_string.py: FormatString` | re-exported from `openjd_expr::format_string` | Identical re-export structure on both sides. |
| `_symbol_table.py: SymbolTable` | re-exported from `openjd_expr::symbol_table` | Same. |
| `v2023_09/_template.py: JobTemplate, EnvironmentTemplate` | `template/job_template.rs`, `template/environment_template.rs` | serde derives instead of Pydantic models; same fields. |
| `v2023_09/_step.py` | `template/step.rs: StepTemplate, StepScript, SimpleAction` | Same. |
| `v2023_09/_environment.py` | `template/environment.rs: Environment, EmbeddedFile` | Same. |
| `v2023_09/_actions.py` | `template/actions.rs: Action, EnvironmentActions, CancelationMode` | RFC 0008 wrap fields appear here in both. |
| `v2023_09/_parameters.py` | `template/parameters.rs: JobParameterDefinition` | EXPR-extended types in `template/expr_parameters.rs` on the Rust side. |
| `v2023_09/_task_parameters.py` | `template/task_parameters.rs` | Includes `RangeConstraint`, `IntRange`, `FloatRange`, `StringRange`. |
| `v2023_09/_validation.py` | `template/validate_v2023_09/` (a *directory* of passes) | Rust splits validation into one file per concern; Python keeps it together. |
| `_create_job.py` | `job/create_job/` | Mostly equivalent. |
| `_step_dependency_graph.py` | `job/step_dependency_graph.rs` | Same DAG semantics. |
| `_step_param_space.py` | `job/step_param_space.rs: StepParameterSpaceIterator` | Same lazy cartesian product. |

## Expression: file-by-file map

| Python (`openjd.model._expr`, etc.) | Rust (`openjd_expr`) | Notes |
|---|---|---|
| `_expr/_types.py: ExprType, TypeCode` | `types.rs: ExprType, TypeCode` | Same shape; Rust uses `#[non_exhaustive]` on both. |
| `_expr/_value.py: ExprValue` | `value.rs: ExprValue` | Rust adds typed list variants (`ListInt`, `ListString`, `ListPath`, `ListList`) for memory locality; Python has a single `list` variant. |
| `_expr/_symbol_table.py: SymbolTable` | `symbol_table.rs: SymbolTable` | Same. |
| `_expr/_eval.py: Evaluator` | `eval/evaluator.rs: Evaluator` | Same algorithm; Rust enforces budgets through `EvalContext::charge_op`/`check_memory`. |
| `_expr/_parse.py` | `eval/parse.rs: ParsedExpression` | Python uses `ast.parse`; Rust uses `ruff_python_parser`. Both do the same-length keyword-rewrite trick. |
| `_expr/_function_library.py` | `function_library.rs` (+ `default_library.rs`) | Same multi-dispatch. |
| `_expr/_range_expr.py: RangeExpr` | `range_expr.rs: RangeExpr` | Same. |
| `_expr/_path_mapping.py: PathFormat, PathMappingRule` | `path_mapping.rs` | Same. |
| `_expr/_format_string.py: FormatString` | `format_string.rs: FormatString` | Same; Rust adds `MAX_FORMAT_STRING_LEN` and `MAX_FORMAT_STRING_SEGMENTS` defensive caps. |

## Architectural differences

### 1. Concurrency model

- Python: GIL serialises Python-level execution; subprocess and IO use
  `threading` + `asyncio` selectively. Public API is mostly sync.
- Rust: `Session` is `&mut self` (one caller at a time per session).
  Lifecycle methods are `async fn` on tokio. Internal concurrency is
  channel-based — no shared mutable state across tasks. See doc 13.

### 2. Errors

- Python: exceptions, `RuntimeError`, `ValueError`, custom
  `BadCredentialsException`. Stack-unwinding propagation.
- Rust: `Result<T, E>` everywhere. Two main `thiserror` enums:
  `ModelError` (with structured `ValidationErrors`) and
  `ExpressionError` (with structured `ExpressionErrorKind`). No
  panics in production paths.

### 3. Type system

- Python: Pydantic models (runtime validation), some `TypedDict`,
  many `Optional[X]` annotations.
- Rust: serde-derived structs with `#[serde(deny_unknown_fields)]`,
  `Option<T>` for optional fields, `#[non_exhaustive]` on every
  public enum for forward compat.

### 4. Cross-user subprocess

- Python: invokes `sudo` (POSIX) or uses `WindowsPermissionHelper` to
  set DACLs and call `CreateProcessAsUserW`. The helper logic is
  in-process.
- Rust: spawns a long-lived helper *binary* (`helper/`) per session
  for cross-user work, talks to it over a pipe. This isolates the
  privileged operations into a small, auditable binary.

### 5. Embedded files / tempdirs

Largely equivalent. Both write `<EmbeddedFile>` content to a
session-owned tempdir and expose `Task.File.<name>` /
`Env.File.<name>` as `path` values. The Rust side has an explicit
`StickyBitPolicy::Strict | Permissive` to tune POSIX behaviour when a
parent dir is world-writable; Python's behaviour is hard-coded to the
strict equivalent.

### 6. Logging

- Python: stdlib `logging`. `LOG` is a configured logger.
- Rust: `log` facade (so embedders pick a sink) + `tracing` for
  structured spans. Pretty banners via `log_section_banner`. Same
  `LogContent` enum shape on both sides.

### 7. JS/web bindings

- Python: none, but Python is itself an embedding target for many
  callers.
- Rust: `openjd-for-js` exposes the model + expr layers via NAPI/wasm.
  Sessions deliberately omitted (subprocess + cross-user logic does
  not translate).

## What is *not* the same

| Feature | Python | Rust |
|---|---|---|
| `WRAP_ACTIONS` (RFC 0008) | not yet | implemented in model + sessions; pending RFC acceptance |
| Job-attachments snapshots | lives in `deadline-cloud-for-python` | first-party `openjd-snapshots` crate |
| CLI binary | Python CLI (`openjd-cli` on the Python side) | Rust binary (`openjd-rs`) |
| Cross-user helper | in-process | separate binary spawned per session |
| Defensive caps on inputs | implicit | explicit constants (`MAX_*` everywhere) |

## Migration tips

If you're porting Python code to Rust:

- `dict[str, X]` → `HashMap<String, X>` for general maps,
  `IndexMap<String, X>` if order matters (the model uses this for
  parameter ordering).
- `Optional[X]` → `Option<X>`. Default values: `Option<X>::or` reads
  like Python's `value or default`, except `0`/`""`/`[]` are not
  falsy (matching EXPR semantics).
- `dataclass` / Pydantic model → `#[derive(serde::Deserialize)]`
  struct with `deny_unknown_fields`.
- Exception → `Result<T, E>` with a `thiserror` enum. The hardest
  habit to break is *not* using `?` to bubble errors out of a
  computation that should accumulate — for those (validation), use
  `&mut ValidationErrors`.
- Async callback → tokio `mpsc::unbounded_channel` if you need
  many-to-one streaming, or a `Box<dyn Fn>` callback if you need
  fire-and-forget.

## Exercise

```sh
# 1. Pick a Python file from openjd-sessions-for-python.
$EDITOR ~/path/to/openjd-sessions-for-python/src/openjd/sessions/_session.py

# 2. Find its counterpart in the Rust tree.
grep -rn "fn run_task" crates/openjd-sessions/src/

# 3. For one method, write a side-by-side diff in your head:
#    - What's the input shape?
#    - What's the return type / exception model?
#    - What concurrency primitives appear?
#    - What's the same? What's different?

# 4. Now go the other way: pick a Rust file with no obvious Python
#    counterpart (e.g. crates/openjd-sessions/src/cross_user_helper.rs).
#    What in the Python tree does the same job? Often it's spread
#    across _subprocess.py + _windows_permission_helper.py + _linux/.
```

## What's next

This is the last doc in the bootstrap track. From here, pick the area
you'll work in:

- **Working in `openjd-expr`?** Re-read doc 15 for the pipeline, then
  open `crates/openjd-expr/src/eval/evaluator.rs` and read top-to-bottom.
- **Working in `openjd-model`?** Re-read doc 14, then read each file
  in `crates/openjd-model/src/template/validate_v2023_09/` — they're
  short.
- **Working in `openjd-sessions`?** Re-read doc 13 and doc 16, then
  read `crates/openjd-sessions/src/session.rs` from top to bottom.
  It's the longest file in the workspace but every section is gated
  by a clear comment header.
- **Working on RFC 0008?** Re-read doc 16 and the koans in
  `../rfc-0008/`.
