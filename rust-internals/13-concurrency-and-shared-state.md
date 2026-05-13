# 13 — Concurrency and shared state

## What this is

The honest answer to "is this codebase DashMaps and async everywhere?"
**No.** There is no `dashmap` dependency. There is exactly one
`Mutex<HashMap>` in the whole workspace, and it's a pure cache. Almost
all "shared" state is `Arc<T>` over an immutable value. The async
surface is real but small, lives entirely inside `openjd-sessions`, and
is built on tokio channels rather than locks.

Reading this before doc 14 (parsing) and doc 15 (expression) saves time:
you'll already know that those crates are sync, single-threaded APIs and
stop looking for hidden synchronization.

## Read these in order

- `crates/openjd-expr/src/default_library.rs:15-30` — the only global mutex in the workspace.
- `crates/openjd-expr/src/function_library.rs:39-50` — `Arc<dyn Fn>` for function dispatch.
- `crates/openjd-expr/src/profile.rs:108-120` — `Arc<Vec<PathMappingRule>>` for shared rules.
- `crates/openjd-sessions/src/session.rs:1-200` — async API shape, tokio imports, `&mut self` methods.
- `crates/openjd-sessions/src/session.rs:1289-1469` — the cancellation + message-pump pattern (`run_task`).
- `crates/openjd-sessions/Cargo.toml:25-30` — tokio feature flags.

## The state of play

| Concern | What's used | Where |
|---|---|---|
| Global function-library cache | `LazyLock<Mutex<HashMap<...>>>` | `openjd-expr/src/default_library.rs:27` |
| Function dispatch | `Arc<dyn Fn>` (immutable) | `openjd-expr/src/function_library.rs:42` |
| Path-mapping rules | `Arc<Vec<PathMappingRule>>` (immutable) | `openjd-expr/src/profile.rs:111` |
| Cancel signal in async actions | `tokio::sync::watch` | `openjd-sessions/src/session.rs:854, 1111, 1312` |
| Action-message stream | `tokio::sync::mpsc::unbounded_channel` | `openjd-sessions/src/session.rs:977, 1225, 1469` |
| Cross-future cancellation | `tokio_util::sync::CancellationToken` | `openjd-sessions/src/session.rs:17` |
| Anything else | (nothing) | — |

`grep -rn dashmap crates/` returns nothing. There is no `RwLock` outside
test utilities. The `Arc<Mutex<...>>` count for non-test code is **one**.

## The one mental model

**Most of `openjd-rs` is single-threaded, sync, and immutable.
Concurrency is concentrated inside one `async fn` per action in
`openjd-sessions`, and even there the dominant pattern is "spawn three
tasks, glue them together with two channels."**

Specifically:

- `openjd-expr` is a synchronous CPU-bound library. `Evaluator`,
  `ParsedExpression`, `FormatString` — all blocking calls. The crate
  is `Send + Sync` where it can be, but contains no internal threads
  or futures.
- `openjd-model` is also synchronous. `decode_template`,
  `validate_v2023_09::*`, `create_job`, `StepDependencyGraph::build` —
  all blocking. No async, no spawning, no locks.
- `openjd-sessions` is async, on tokio. But `Session` itself is `&mut
  self` for every public lifecycle method (`enter_environment`,
  `run_task`, `exit_environment`, `cancel`, `run_subprocess`). One
  caller, one action at a time. Concurrency exists *between* sessions
  (callers spawn many) and *inside* one action (subprocess + stdout
  reader + stderr reader + cancel watcher all share state via
  channels).

## The one global mutex, demystified

```rust
// crates/openjd-expr/src/default_library.rs:27
static PROFILE_CACHE: LazyLock<Mutex<HashMap<ProfileKey, Arc<FunctionLibrary>>>> =
    LazyLock::new(|| Mutex::new(HashMap::new()));
```

This caches *constructed function libraries*, keyed on the rules-independent
profile key. Why it's safe to keep a single global `Mutex`:

- The hot path is `Arc::clone(&cached)` after a single map lookup. Lock
  hold time is microseconds.
- Building a library on cache miss takes ~1ms but happens once per
  unique profile per process.
- The values are `Arc<FunctionLibrary>` — readers don't hold the mutex
  across function calls. They clone the `Arc` and drop the lock.

If you ever see `Arc<Mutex<...>>` proposed anywhere else in this codebase,
push back: there is almost always a way to make the inner data immutable
and skip the mutex.

## The async surface, by the numbers

`openjd-sessions/Cargo.toml`:

```toml
tokio = { version = "1", features = [
    "rt-multi-thread", "process", "io-util", "time", "sync", "macros", "fs",
] }
tokio-util = "0.7"
```

The four primitives that account for almost all concurrent code:

### 1. `tokio::sync::watch::channel(initial)` — cancel signal

One per action invocation. The receiver is shared between the subprocess
spawner and the cancel poller. When `Session::cancel(time_limit)` is
called, it publishes `Some(grace)` into the watch channel; the spawned
task's `select!` arm wakes up.

```rust
// session.rs:854 (enter), 1111 (exit), 1312 (run_task)
let (cancel_tx, cancel_rx) = tokio::sync::watch::channel(None);
```

### 2. `tokio::sync::mpsc::unbounded_channel()` — action-message stream

One per action. The subprocess wrapper writes `ActionMessage::Stdout`,
`ActionMessage::Stderr`, `ActionMessage::Status`, `ActionMessage::Done`
events into the sender. The driving `async fn` (e.g. `run_task`) loops
on the receiver and emits them to the caller's callback.

```rust
// session.rs:977 (enter), 1225 (exit), 1469 (run_task)
let (tx, mut rx) = tokio::sync::mpsc::unbounded_channel();
```

### 3. `tokio_util::sync::CancellationToken` — composable cancel

Used at the `SessionConfig::cancel_token` boundary so a caller can cancel
the entire session externally. Internal action-level cancellation uses
the simpler `watch` channel.

### 4. `tokio::process::Command` — async subprocess

Spawning the subprocess yields a `tokio::process::Child` that integrates
with the runtime. Stdout/stderr piped through `tokio::io::AsyncReadExt`
and forwarded to the mpsc channel.

## Why no DashMap

DashMap is for many readers and many writers contending on the same map.
The closest thing in `openjd-rs` is the per-session `environments:
HashMap<EnvironmentIdentifier, Environment>` — but that map is owned by
one `Session`, mutated only via `&mut self`, and never crosses thread
boundaries without consuming the session. No contention, no need for a
concurrent map.

If a future feature lets multiple actions in a session run concurrently,
the right answer is *probably still not* DashMap — it's a redesign that
either splits the session into per-action substates or uses an actor
model with a single owner per piece of state. Watch out for proposals
that reach for DashMap as a shortcut.

## Sync vs async, summarized

| Crate | Public API | Internals |
|---|---|---|
| `openjd-expr` | Sync, blocking | No locks, no spawning |
| `openjd-model` | Sync, blocking | No locks, no spawning |
| `openjd-sessions` | `async fn` for lifecycle methods | tokio channels + child processes |
| `openjd-cli` | `tokio::main` (delegates to sessions) | — |
| `openjd-snapshots` | Sync (S3 client may be async; check at the boundary) | — |

If you're embedding the model + expr crates into another runtime
(JavaScript via `openjd-for-js`, a synchronous Python binding, a
non-tokio Rust app), you do not need an async runtime at all. This is
why `openjd-for-js` only exposes the model and expr crates: sessions'
tokio dependency would be a non-starter for those embedders.

## Exercise

Open `crates/openjd-sessions/src/session.rs` and find `Session::run_task`
(starts at `:1289`). Without reading every line, count:

1. How many `tokio::sync::*::channel` calls are made?
2. What does each channel carry?
3. Where is `cancel_tx` cloned, and where is the receiver dropped?
4. Does any of this code take a lock? (Search for `lock()` in the
   function body.)

Expected answers:

1. Two: one `watch` (cancel) and one `mpsc::unbounded` (`ActionMessage`).
2. `watch` carries `Option<Duration>` (`None` = no cancel requested,
   `Some(grace)` = cancel with this notify period). `mpsc` carries
   `ActionMessage` enum variants.
3. `cancel_tx` is held by the `Session` so external `cancel()` can
   publish into it. The receiver clone is moved into the spawned
   subprocess-driving task and dropped when that task ends.
4. No. The `&mut self` discipline removes the need.

If you can answer those four, you've internalised the concurrency model.

## What's next

[14 — Parsing call stack](./14-parsing-call-stack.md). Now that you know
the parsing crates are sync, you can read the full pipeline without
worrying about hidden async or shared mutation.
