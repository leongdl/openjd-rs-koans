# 09 — Step dependency graph + topological sort

**Use case:** A job has steps `prepare → render → composite` with
`composite.dependencies: [{dependsOn: render}]` and
`render.dependencies: [{dependsOn: prepare}]`. How does the runtime
know "run `prepare` first, then `render`, then `composite`"? And what
happens if a user introduces a cycle?

## Call stack

Triggered by any host that needs an ordering — `openjd run` and any
worker that schedules tasks across steps.

1. **`StepDependencyGraph::new(job)`** —
   [openjd-rs/crates/openjd-model/src/job/step_dependency_graph.rs:46](../../../openjd-rs/crates/openjd-model/src/job/step_dependency_graph.rs#L46).
   Builds the graph from `Job::steps`:
   - `name_to_index`: a `HashMap<String, NodeIndex>` so dependency
     names ("`render`") can be resolved to `usize` array indices.
   - `nodes`: one per step, carrying `step_index`, `name`, and empty
     `in_edges` / `out_edges` lists.
   - `edges`: walks every step's `dependencies:`. For each
     `dependsOn`, look up the origin's `name_to_index`. If missing,
     return `ModelError::DecodeValidation("Step '<dep>' depends on
     unknown step '<origin>'")` — the dependency must reference a
     step in this job (cross-job deps are not modeled here).
   - Pushes the new edge index onto both `nodes[origin].out_edges`
     and `nodes[dep].in_edges`. Bidirectional indexing makes both
     "what depends on me?" and "what do I depend on?" cheap.

2. **`StepDependencyGraph::topo_sorted()`** —
   [step_dependency_graph.rs:139](../../../openjd-rs/crates/openjd-model/src/job/step_dependency_graph.rs#L139).
   The sort itself. **Iterative DFS** with a tri-state (`0` =
   unvisited, `1` = started, `2` = completed) per node, no recursion
   (deep dep chains can't blow the stack). Pseudo-skeleton:

   ```text
   for each i in 0..n:
       if state[i] != 0: continue
       stack = [i]
       while stack non-empty:
           top = stack.last()
           match state[top]:
               2 ─── pop, already done
               1 ─── post-order visit: state[top] = 2, push to result, pop
               0 ─── pre-order visit: state[top] = 1, push every unfinished dep
                     (sorted by reverse template index for determinism)
                     If any pushed dep is already in state 1 → CYCLE
   ```

   On cycle detection it walks the stack, filters to nodes still in
   state `1`, and produces an error message naming the cycle path:

   ```
   A circular dependency was found in the step dependency graph:
   composite -> render -> prepare -> composite
   ```

3. **`StepDependencyGraph::topo_sorted_names()`** —
   [step_dependency_graph.rs:199](../../../openjd-rs/crates/openjd-model/src/job/step_dependency_graph.rs#L199).
   Convenience that maps the indices back to names for display /
   logging.

## Two subtle design choices

### Deterministic tie-breaking

When a node has multiple unfinished dependencies, the order they're
pushed to the stack determines the order of the sort. The code
sorts dependency indices by **reverse template order**:

```rust
let mut dep_indices: Vec<NodeIndex> = ...;
dep_indices.sort_unstable_by(|a, b| b.cmp(a));
```

This is intentional: pushing in reverse means they pop in forward
order. The result is that the topological sort is stable across
runs and matches the user's mental model of "the step earlier in
the YAML appears earlier in the sort, all else being equal."
Without this, the iteration order of `Vec` indices is fine, but when
the dep set was sourced via cross-references through `name_to_index`,
ordering would not be preserved.

### Iterative, not recursive

A naive recursive topo sort would be:

```rust
fn visit(node) {
    for dep in node.deps { visit(dep); }
    result.push(node);
}
```

Idiomatic, but a job with a 10,000-deep dependency chain would blow
the default 2 MB Rust thread stack. The iterative version uses a
heap-allocated `Vec<NodeIndex>` for its working stack — bounded by
available memory, not stack frames. A worker validating a job that
came over the wire from an untrusted source should not panic on
adversarial dep chains.

The tri-state is what lets a stack-based DFS produce **post-order**
results (which is what topo sort needs):

- First visit (`0 → 1`): push deps onto stack, leave self.
- Second visit (`1 → 2`): all deps now in state 2, append self to
  result.

Without the post-order hook, you'd get a pre-order traversal which
isn't a valid topo sort.

## Cycle error format

The error message lists the actual cycle, not just "a cycle exists":

```
A circular dependency was found in the step dependency graph:
composite -> render -> prepare -> composite
```

Built by walking the working stack, filtering to nodes still in
state `1` (i.e. on the current DFS path), and joining with arrows.
This makes diagnosis a one-shot: the user sees exactly which steps
are involved.

## What lives on the graph beyond the sort

`StepDependencyGraph` exposes:
- `step_node(name)` — direct lookup by step name.
- `node_count()` / `node(i)` / `edge(i)` — array-style access.
- `max_indegree()` / `max_outdegree()` — used by validators to
  enforce caller-supplied caps on how complex a single step's
  dependency fan-in/-out can be.

The graph is built once per job and queried as the runtime makes
scheduling decisions; the runtime is free to walk it in any
direction.

## Related

- [koan 01](01-parser-to-template-data-structure.md) step 7 — where
  `dependsOn` strings are validated against the step set.
- [koan 03](03-expression-block-end-to-end.md) — the resolved `Job`
  this graph operates over.
