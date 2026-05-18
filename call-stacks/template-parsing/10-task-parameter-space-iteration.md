# 10 — Lazy task-parameter cartesian product

**Use case:** A step has

```yaml
parameterSpace:
  taskParameterDefinitions:
    - name: Frame
      type: INT
      range: "1-100"
    - name: Layer
      type: STRING
      range: ["beauty", "diffuse", "spec"]
    - name: Camera
      type: STRING
      range: ["camA", "camB"]
  combination: "(Frame, Layer) * Camera"
```

That's `100 * 3` lockstep frames-and-layers paired up, then crossed
with 2 cameras = **600 tasks**. Or with chunk-int adaptive chunking,
maybe 60 tasks the first round and a different count later. How does
the runtime walk this without materializing 600 records up-front?

## Call stack

1. **`StepParameterSpaceIterator::new(space)`** —
   [openjd-rs/crates/openjd-model/src/job/step_param_space.rs:1225](../../../openjd-rs/crates/openjd-model/src/job/step_param_space.rs#L1225)
   builds a tree of `Node` trait objects. `new_with_chunk_override(space, Some(1))`
   ([step_param_space.rs:1231](../../../openjd-rs/crates/openjd-model/src/job/step_param_space.rs#L1231))
   forces single-task counting (used for total-task-count enforcement).

2. **`new_inner(space, chunk_override)`** —
   [step_param_space.rs:1238](../../../openjd-rs/crates/openjd-model/src/job/step_param_space.rs#L1238)
   the constructor logic. Three branches:

   - **No params** → `ZeroDimSpaceNode` (one task with no
     parameters; the step still runs).
   - **`combination: "*"` or absent** → cross product of all params
     in definition order. Builds one `make_leaf_node` per parameter
     (a `RangeExprNode` for INT / CHUNK_INT, a list-backed node for
     STRING / PATH).
   - **Explicit combination expression** → tokenize + recursive-descent
     parse.

3. **`tokenize(expr)`** —
   [step_param_space.rs:43](../../../openjd-rs/crates/openjd-model/src/job/step_param_space.rs#L43).
   Hand-rolled scanner: `*`, `(`, `)`, `,` are single-char tokens;
   identifiers absorb runs of non-special characters; whitespace is
   discarded.

4. **`parse_node_expr(tokens, space, adaptive_info, chunk_override)`** —
   [step_param_space.rs:1463](../../../openjd-rs/crates/openjd-model/src/job/step_param_space.rs#L1463)
   recursive descent, two productions:

   - `parse_node_product` ([step_param_space.rs:1480](../../../openjd-rs/crates/openjd-model/src/job/step_param_space.rs#L1480))
     — products separated by `*` → `ProductNode { children, length }`.
   - `parse_node_element` ([step_param_space.rs:1517](../../../openjd-rs/crates/openjd-model/src/job/step_param_space.rs#L1517))
     — `(a, b, c)` → `AssociationNode` (lockstep), or a bare name →
     leaf `make_leaf_node`. Associations require equal-length children
     (otherwise `Associative combination: all members must have the
     same number of values, got X and Y`).

5. **Length is computed eagerly with overflow checking** —
   `checked_product_len` ([step_param_space.rs:32](../../../openjd-rs/crates/openjd-model/src/job/step_param_space.rs#L32)).
   `usize::MAX` overflow → `Total parameter space size overflow`. A
   `Frame: "1-1000000000" * Layer: "1-1000000000"` would overflow
   `usize` on 64-bit; the iterator constructor catches this without
   trying to enumerate.

## The two index modes

### `ProductNode` — divmod indexing

Walking a product of three children with sizes `[100, 3, 2]` (total
600). Node `n`'s value at index `i ∈ [0, 599]` is computed by
**divmod from the right**:

```text
i = 423
camera_idx = 423 % 2     = 1   → camB
i //= 2 = 211
layer_idx  = 211 % 3     = 1   → diffuse
i //= 3 = 70
frame_idx  = 70 % 100    = 70  → 70
```

Result: `(Frame=70, Layer=diffuse, Camera=camB)`. The **rightmost
child moves fastest**, which is the convention from N-d array
linearization (and matches Python's spec implementation).

This is purely arithmetic on the index — no preallocated cross
product, no buffered intermediate. `O(depth)` per task lookup.

### `AssociationNode` — lockstep indexing

For `(Frame, Layer)` with both having length 3:

```text
i = 0 → (Frame=1, Layer=beauty)
i = 1 → (Frame=2, Layer=diffuse)
i = 2 → (Frame=3, Layer=spec)
```

All children share the same index `i`. `len()` is one child's length
(validated equal at parse time).

### Composition

`(Frame, Layer) * Camera` produces a tree:

```
ProductNode (length = 3 * 2 = 6)
├── AssociationNode (length = 3)
│   ├── Frame leaf  (3 values)
│   └── Layer leaf  (3 values)
└── Camera leaf     (2 values)
```

The same divmod indexing recurses into the association — the
left-hand of the product still moves slowest.

## Adaptive chunking

A `CHUNK_INT` parameter with `chunks.targetRuntimeSeconds` set
triggers adaptive chunking ([step_param_space.rs:1260](../../../openjd-rs/crates/openjd-model/src/job/step_param_space.rs#L1260)):

- An `Arc<AtomicUsize>` is initialized to `chunks.default_task_count.max(1)`.
- The chunk-int parameter is moved to be the **innermost / fastest-
  varying** child, so chunk-size changes only affect the rate at
  which the surrounding cross-product advances.
- The runtime calls `set_chunks_default_task_count(value)` after
  each chunk based on observed runtime — `Ordering::Relaxed` since
  chunks are processed sequentially.
- `len()` returns 0 for adaptive iterators (length isn't knowable
  in advance) and `is_empty()` returns `false` (there's always at
  least one task).

## Sequential vs random-access iteration

Most spaces support both modes:

- **Random-access**: `iter.get(i)` returns the i-th task in O(depth)
  time. Used by validators that need only the first N tasks, or by
  range queries.
- **Sequential**: `iter.next()` walks tasks in order, holding a
  `node_iter: Box<dyn NodeIterator>`. Used when the underlying
  computation is order-dependent.

`needs_sequential` is set when:
- Adaptive chunking is in use (chunk size changes each step).
- A "contiguous chunk" config is in play
  (`has_contiguous_chunks(space)` — chunks span gaps the iterator
  must walk in order).

When sequential, `len()` and `get(i)` lose meaning.

## Why no upfront materialization

A `Frame: "1-1000000"` step yields a million tasks. Each task slot
needs a `Frame` integer. Pre-computing 1,000,000 `TaskParameterValue`
records would burn ~32 MB on a single parameter dimension before any
task ran. The lazy iterator computes the value at index `i` on
demand from the integer; memory is bounded by O(parameter dimensions)
regardless of the space's logical size.

## Related

- [koan 01](01-parser-to-template-data-structure.md) step 7 — the
  `task_chunking` validation pass that enforces `CHUNK_INT`-specific
  rules at validate time.
- [koan 03](03-expression-block-end-to-end.md) — `Task.Param.*`
  values come from this iterator at session runtime.
- [koan 06](06-symbol-table-seeding-three-scopes.md) — task scope
  defines the symbols this iterator's values populate.
