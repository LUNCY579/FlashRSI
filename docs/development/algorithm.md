# Add an algorithm

Implement your search policy in `Algorithm.evolve`. Use the shared resource and evaluation operations, while keeping proposal, feedback, selection, and memory decisions in your method.

## Declare its scope

Subclass [`Algorithm`](../../flashrsi/core/algorithm.py) and declare `evolve_scope` and `observation_scope` using the Target's component names. Use `{"*"}` only when the method is intended to operate on every declared component.

The constructor receives the method configuration and `target`, `worker`, and `benchmark`. It applies the scope and loads task splits. Check method-specific requirements in `_check`.

Give the class a stable `name` and a `description`. Define configuration defaults in `_defaults` and parameter constraints in `_ranges`, as [OPRO does](../../flashrsi/algorithms/opro/__init__.py). Reserve `_check` for requirements that depend on the actual collaborators, such as a nonempty train split or an instruction file in the Target's declared scope.

## Implement the search loop

```python
async def evolve(self):
    # Observe training evidence and the current resource version.
    # Ask the Worker for a proposal or a candidate file edit.
    # Commit permitted changes and evaluate the candidate.
    # Apply the method's own admission and selection policy.
    # Return the selected Snapshot.
    ...
```

Use the existing [algorithm implementations](../../flashrsi/algorithms) as executable examples of checkout, rollout, grading, and records. The runner assembles dependencies and evaluates the selected Snapshot on held-out test tasks; it should not acquire your method's search policy.

Keep test feedback out of proposal generation and selection. If a method uses dev feedback, state exactly how and when it influences candidate adoption.

### Worked example: OPRO

1. Declare `evolve_scope = {"prompt"}` and use `_instruction_path()` to resolve the Target's instruction file.
2. Score the starting Snapshot on train with `await self._evaluate(snapshot, self.benchmark.train, label="baseline")`. Use the returned `score` for selection and the recorded trajectories to form proposal evidence.
3. Call `await self.worker.ask(prompt)` and parse the reply in the algorithm. OPRO expects text in an `<INS>` tag. For a directory-editing algorithm, use `await self.worker.edit(directory, prompt)` with an Agent Worker that supports file edits.
4. Use `_candidate(parent, {path: text})` to create a new Snapshot and `_evaluate` to measure it on train. OPRO retains scored candidates in its history and keeps the earlier candidate on ties.
5. Return the selected Snapshot. OPRO uses `_promote` once at the end to record its selection. The runner then evaluates the selected version on test.

## Configure and register it

Add the class to [`flashrsi/algorithms/__init__.py`](../../flashrsi/algorithms/__init__.py) and the runner's [`_ALGORITHMS` mapping](../../flashrsi/runner.py). Add a YAML example that chooses an appropriate Worker and exposes the method's budget and search parameters.

Document the edited resources, feedback, adoption rule, termination condition, and differences from the upstream method. Validate key state transitions with deterministic offline cases, such as rejected edits or ties in candidate scores.

## Verify the integration

The example below uses the bundled [quickstart YAML](../../examples/quickstart.yaml) and local QA data. From the repository root, follow [Installation](../guide/installation.md), start Docker with a Linux daemon, and configure the Target and Worker model credentials independently. If you copy the YAML, keep it in `examples/` or adjust its relative paths.

```bash
flashrsi run --config examples/quickstart.yaml --algorithm opro --algorithm.max_steps 1 --algorithm.num_candidates 1 --check-only
flashrsi run --config examples/quickstart.yaml --algorithm opro --algorithm.max_steps 1 --algorithm.num_candidates 1
```

The first command checks initialization without model calls. The second uses the configured models. Inspect `trajectories/` for the baseline, the Worker proposal, and the final test outcomes. A duplicate or malformed proposal can leave the baseline selected. Check that `result.json` names the selected Snapshot and reports a nonempty test set. See [run outputs](../running-experiments.md) for the full layout.

For module-based edits, see [ModularRSI](../modular-rsi.md). To validate a new benchmark with the same workflow, see [Add a benchmark](./benchmark.md).
