# 07 — Parameter merge across env templates → symbol table

**Use case:** A job is submitted with a job template plus two environment
templates. Each template declares a `Frame` parameter with overlapping
constraints (`minValue: 1` in one, `maxValue: 100` in another, an
`allowedValues` list in a third). Trace how those collapse to one
typed entry per parameter, get coerced from CLI strings, and become
the seed of every symbol table from this job onward.

The §1.2.1 spec says: env templates first (in order), job template
last; types must match across all sources; constraints **tighten**
across the merge (mins go up, maxes go down, allowedValues
intersect).

## Call stack

Triggered from `openjd run` ([cli/src/run/mod.rs:103](../../../openjd-rs/crates/openjd-cli/src/run/mod.rs#L103))
or any host that calls `preprocess_job_parameters`:

1. **`preprocess_job_parameters(job_template, input_values, env_templates, path_options)`** —
   [openjd-rs/crates/openjd-model/src/job/create_job/parameters.rs:833](../../../openjd-rs/crates/openjd-model/src/job/create_job/parameters.rs#L833).
   Top of the pipeline. Validates that `job_template_dir` is absolute
   (unless walk-up is allowed), then orchestrates merge → satisfiability
   → coercion → constraint check → defaults → "extras"/"missing"
   reporting. **Collects errors instead of bailing on first failure**
   so users see every problem in one round trip.

2. **`merge_job_parameter_definitions(jt, env_templates)`** —
   [parameters.rs:24](../../../openjd-rs/crates/openjd-model/src/job/create_job/parameters.rs#L24).
   The merge itself. Walks env templates first (in order), then the
   job template. For each parameter `p`:
   - **First sighting** → fresh `MergedParameterDefinition` with this
     parameter's type, default, `objectType`, `dataFlow`, and
     constraints. `merge_constraints` records the initial bounds.
   - **Subsequent sightings**:
     - Type mismatch → `ModelError::Compatibility` with both source
       names. (E.g. "`Frame` is INT in `EnvironmentTemplate 'A'` and
       STRING in `JobTemplate`.")
     - For PATH params: `objectType` and `dataFlow` mismatches → same
       error class.
     - Default → **last writer wins** (§1.2.1).
     - `merge_constraints(p)` → tightens.

3. **`MergedParameterDefinition::merge_constraints(def)`** —
   [parameters.rs:177](../../../openjd-rs/crates/openjd-model/src/job/create_job/parameters.rs#L177).
   The "tighten" rule:
   - `minValue` → `max(existing, new)` (more restrictive).
   - `maxValue` → `min(existing, new)`.
   - `minLength` → `max`, `maxLength` → `min`.
   - `allowedValues` → set intersection. (Single-template entry →
     no-op; multi-template → intersect; if the intersection is empty,
     `validate_satisfiable` will catch it later.)
   - For `LIST_*` types, `item_*` constraints follow the same logic.

4. **`MergedParameterDefinition::validate_satisfiable()`** —
   [parameters.rs:287](../../../openjd-rs/crates/openjd-model/src/job/create_job/parameters.rs#L287).
   After merging, this catches infeasible constraints: empty
   `allowedValues` intersections, `minValue > maxValue`,
   `minLength > maxLength`, etc. Run **before** trying to coerce any
   user input so a user sees "constraint conflict between templates"
   rather than a confusing "value 5 is below minValue 10" when the
   real problem is that template A's `maxValue: 4` and template B's
   `minValue: 10` are mutually exclusive.

5. **Per-parameter input handling** at [parameters.rs:856–](../../../openjd-rs/crates/openjd-model/src/job/create_job/parameters.rs#L856):
   For each merged parameter:
   - If the input was provided:
     - PATH-typed inputs go through URI handling (EXPR-only) or
       relative-path joining against `current_working_dir`.
     - **`coerce_to_type(&value, param_type)`** — [parameters.rs:662](../../../openjd-rs/crates/openjd-model/src/job/create_job/parameters.rs#L662)
       — converts CLI strings ("42", "3.14", "true") to the
       parameter's typed `ExprValue`. INT → FLOAT widening is
       allowed, FLOAT → INT is not.
     - `param.check_constraints(&expr_value)` runs the merged
       constraints against the typed value.
     - On success: insert into the result map.
   - If no input: try the default (per-parameter coercion + constraint
     check applied to the default), else add to the `missing` list.
   - Inputs with no matching parameter accumulate into the `extras`
     list.

6. **`build_symbol_table(params)`** —
   [parameters.rs:1154](../../../openjd-rs/crates/openjd-model/src/job/create_job/parameters.rs#L1154).
   For each typed parameter value:
   - Sets `Param.<name>` to the typed value — **except** PATH /
     LIST_PATH, which are excluded (their resolved paths depend on
     session-time path mapping).
   - Sets `RawParam.<name>`:
     - For PATH → STRING of the user's raw input (no path mapping).
     - For LIST_PATH → `ListString`.
     - Otherwise → same as `Param.*`.

   This is the symtab that seeds [`create_job`](../../../openjd-rs/crates/openjd-model/src/job/create_job/mod.rs#L62)
   and threads downstream into every step's resolution.

## Error reporting strategy

`preprocess_job_parameters` is unusual in that it **doesn't fail
fast**. Errors accumulate into a `Vec<String>` and are returned all
at once. Order matches the user's mental model:

1. Per-parameter satisfiability (constraint conflicts after merging).
2. Per-parameter coercion / constraint failures on the user's input.
3. Per-parameter default-handling failures.
4. One line listing **all** unmatched extras.
5. One line listing **all** missing required values.

This means a user who submits a job with three problems gets all
three in one error, not "fix one, retry, find the next" loops.

## Why constraint *tightening* (not loosening)

If env template A says "Frame must be 1–100" and B says "Frame must
be 50–200", the only safe interpretation is "1–100 AND 50–200" =
"50–100" — loosening to "1–200" would let a `Frame=150` value sneak
past A's check at runtime. The tightening rule preserves every
template's invariants.

## Related

- [koan 06](06-symbol-table-seeding-three-scopes.md) — what
  `build_symbol_table` produces is the seed for every other scope.
- [koan 03](03-expression-block-end-to-end.md) — stages 3–4 consume
  this symtab.
