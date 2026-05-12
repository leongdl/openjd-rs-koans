# RFC 0008 — Environment Wrap Actions: Worked Examples

Hands-on examples exploring [RFC 0008 — Environment Wrap Actions](../../openjd-specifications/rfcs/0008-environment-wrap-task-run.md)
and how it composes with the `EXPR` extension
([RFC 0005](../../openjd-specifications/rfcs/0005-expression-language.md) +
[RFC 0006](../../openjd-specifications/rfcs/0006-expression-function-library.md)).

## Contents

### Core patterns

| # | File | What it shows |
|---|------|---------------|
| 01 | [`01-conditional-task-command.yaml`](./01-conditional-task-command.yaml) | Task command using EXPR ternaries: scalar swap, optional flag, grouped args, path conditional |
| 02 | [`02-docker-wrap-conditional.yaml`](./02-docker-wrap-conditional.yaml) | Docker wrap environment with conditional `--user root` / `--gpus all` and safe `repr_sh` quoting |
| 03 | [`03-echo-wrap.yaml`](./03-echo-wrap.yaml) | Dry-run wrap environment that echoes every wrapped action without executing it |

### Deeper wrap-hook applications (container-focused)

| # | File | What it shows |
|---|------|---------------|
| 04 | [`04-env-var-filtering.yaml`](./04-env-var-filtering.yaml) | Secrets firewall: blacklist, whitelist, key rewrite, and wrap-owned env injection |
| 05 | [`05-command-classification.yaml`](./05-command-classification.yaml) | Route wrapped commands to GPU flags, root user, or per-command container images via regex + `in` |
| 06 | [`06-timeout-math.yaml`](./06-timeout-math.yaml) | Arithmetic on `Task.Timeout`; `min()`-capped graceful-shutdown propagation |
| 07 | [`07-let-bindings-factoring.yaml`](./07-let-bindings-factoring.yaml) | Environment-scoped `let` bindings shared across `onEnter`/`onWrap*`/`onExit` |
| 08 | [`08-cross-language-wrappers.yaml`](./08-cross-language-wrappers.yaml) | Python / JSON / PowerShell / cmd wrap scripts via `repr_py`, `repr_json`, `repr_pwsh`, `repr_cmd` |
| 09 | [`09-preflight-fail.yaml`](./09-preflight-fail.yaml) | Preflight assertions with `fail()`: empty-command guard, banned flags, path allow-list, name policy |
| 10 | [`10-path-manipulation.yaml`](./10-path-manipulation.yaml) | Synthesize bind-mounts from touched paths; rewrite `.log` args to container-local scratch |
| 11 | [`11-args-slicing-reordering.yaml`](./11-args-slicing-reordering.yaml) | Head/tail split, flag-before-positional reordering, bounded arg truncation |
| 12 | [`12-regex-name-policy.yaml`](./12-regex-name-policy.yaml) | Regex-driven per-inner-env policy on `Env.Wrapped.Name` (Setup→root, Profile*→perf stat) |
| 13 | [`13-null-coalescing-defaults.yaml`](./13-null-coalescing-defaults.yaml) | `or`-based defaults for nullable fields, and why `""` needs an explicit `== ''` check |

### Beyond containers: other wrap-hook applications

| # | File | What it shows |
|---|------|---------------|
| 14 | [`14-ssh-remote-execution.yaml`](./14-ssh-remote-execution.yaml) | Forward every wrapped action to a remote host via SSH; demonstrates double-level shell quoting |
| 15 | [`15-profiling-instrumentation.yaml`](./15-profiling-instrumentation.yaml) | Profile/trace every action with `perf stat`, `strace`, or `/usr/bin/time` — no containers |
| 16 | [`16-privilege-isolation-sudo.yaml`](./16-privilege-isolation-sudo.yaml) | Drop privileges via `sudo -u` and/or isolate with `unshare` — container-free sandboxing |
| 17 | [`17-audit-logging.yaml`](./17-audit-logging.yaml) | Append a JSONL audit record for every wrapped action using `repr_json()` |
| 18 | [`18-retry-with-backoff.yaml`](./18-retry-with-backoff.yaml) | Bounded retry with exponential backoff; `range()`, `min()`, precomputed sleep schedule |
| 19 | [`19-conda-activation.yaml`](./19-conda-activation.yaml) | Centralize conda env activation in the wrap layer so inner envs don't know about conda |
| 20 | [`20-resource-limits-prlimit.yaml`](./20-resource-limits-prlimit.yaml) | Enforce memory / CPU / fd / process limits via `prlimit` — different profile per action type |
| 21 | [`21-wine-cross-platform.yaml`](./21-wine-cross-platform.yaml) | Transparently run Windows `.exe` commands through Wine on Linux workers |

## Running these

These are demonstrative templates; they require a scheduler that implements
the `WRAP_ACTIONS` and `EXPR` extensions. The examples focus on showing the
shape of the EXPR expressions and how `repr_sh()` keeps shell composition safe
when reconstructing wrapped commands.

## Key patterns to notice

### The safety pattern (every wrap hook)

- **`repr_sh(Task.Command) repr_sh(Task.Args)`** — the canonical safe-quoting
  idiom for reconstructing a wrapped command inside a shell script.
- **`flatten([['-e', e] for e in Task.Environment])`** — list comprehension
  plus `flatten()` to turn `["K=v", ...]` into `-e K=v -e ...` flag pairs.

### Conditional emission

- **`A if cond else null`** in an `args` list — emits or drops the item
  because `null` list items are skipped.
- **`[a, b] if cond else null`** — emits multiple args or nothing because
  `list[T]` results are flattened inline.
- **`[...] if cond else []`** inside `repr_sh()` — emits extra tokens or
  nothing, with quoting applied uniformly at the list level.

### Policy and validation

- **`fail(msg)`** composed with `or` — assertion idiom that surfaces bad
  inputs at template eval time, not mid-job.
- **`re_match(Env.Wrapped.Name or '', r'...')`** — regex-driven policy on
  inner env names without hard-coding lists.
- **`X or DEFAULT`** — null-coalescing default, valid for genuinely nullable
  types like `Env.Wrapped.Name`. For STRING params with an empty default,
  use `X == '' ? fallback : X` instead (EXPR does not treat `""` as falsy).

### Factoring

- **`let:` at the environment `script` level** — shared computation visible
  in every action and embedded file in the environment.
- **`repr_py` / `repr_json` / `repr_pwsh` / `repr_cmd`** — swap `repr_sh` for
  a different target language when the wrap script isn't bash.

## Why wrap hooks are not just for containers

The 14-21 koans exist because "run a command in a different context" covers
far more than containers. Every application below reuses the same mechanic
— intercept an action, reconstruct it safely via `repr_sh`, execute it
somewhere/somehow/someway different — but the "somewhere else" varies:

| Application | The "somewhere else" |
|-------------|----------------------|
| SSH forwarding (14) | a remote host |
| Profiling (15) | under a measurement tool |
| Privilege isolation (16) | as a different user, in a new namespace |
| Audit logging (17) | recorded to a log before and after |
| Retry (18) | attempted multiple times with backoff |
| Conda activation (19) | with a conda env's PATH/PYTHONPATH active |
| Resource limits (20) | under kernel-enforced rlimits |
| Wine (21) | translated through a syscall emulator |

Any transformation of "what command runs, where, and how" is a candidate
for a wrap hook, and they compose with `runOnHost: true` on inner actions
for the bits that must escape the wrapper.

## Reading order suggestion

1. `01`–`03` — understand the mechanic (ternaries, safe quoting, dry-run)
2. `07` — learn `let` bindings because the later examples lean on them
3. `04`, `05`, `09` — the three highest-value production patterns
   (secrets filter, runtime routing, preflight)
4. `06`, `10`–`13` — specialized techniques you'll reach for when a
   specific problem demands them
5. `08` — for when bash isn't your wrap language
6. `14`–`21` — each is a standalone demonstration of a non-container
   application; read any in isolation once you understand the mechanic
