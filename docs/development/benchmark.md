# Add a benchmark

Add tasks and grading behind the benchmark interface. Choose the environment path according to what the tasks need; keep agent resources separate from task data and verifiers.

## Choose an integration path

| Path | Use it when | What to provide |
| --- | --- | --- |
| Supplied context | The public question and context are sufficient to solve the task. | A loader and a grader. Existing adapters such as SearchQA use the shared `supplied_context` executor to isolate Target execution from host-side labels. |
| Harbor | The benchmark ships native Harbor task packages, such as Terminal-Bench. | Native task packages, environment and verifier binding, and reward mapping. |
| Dedicated integration | The benchmark needs a service, simulator, or execution and grading protocol of its own. | A benchmark adapter and the integration that supplies its task environment. |

SWE-bench Verified, AppWorld, and BFCL have native integrations of their own; Harbor is not required for every container task. Start from the closest existing adapter in [the benchmark tree](../../flashrsi/benchmarks).

## Implement tasks and grading

Subclass [`Benchmark`](../../flashrsi/core/benchmark.py). Its two required methods are:

```python
def load(self):
    # Return train/dev/test tuples of Task objects.
    ...

def grade(self, trajs):
    # Return trajectories with the primary metric attached.
    ...
```

Declare a stable `name`, a description, metadata, and `primary_metric`. `load()` must also retain its split mapping in `self._splits` so the `train`, `dev`, and `test` properties work. Each `Task` needs a unique `task_id`, a question, and any ground truth, environment reference, or source metadata needed by the adapter.

`Trajectory` is immutable. Grading returns updated records with finite numeric metrics, typically via `dataclasses.replace`. Define how errors and missing answers contribute to the primary score.

Keep task IDs stable and unique across the returned splits. Pin the upstream data revision where available and document any filtering and split policy. Use the native test split when the benchmark requires it. For custom partitions, document the seed and keep evaluation examples out of the evolution pool. Keep labels and verifier inputs on the grading side.

## Bind an environment when needed

Environment-backed tasks carry `Task.env`. The runner binds the environment executor to the Agent. The harness still uses its ordinary execution interface and returns a `Trajectory`; the benchmark owns task packages, state, and verification.

For another supplied-context suite, follow SearchQA's `Task.env = "supplied_context"` convention and register the new suite in the [startup classifier](../../flashrsi/_startup.py). The runner uses that classifier to bind the executor for both evolution and saved-Snapshot evaluation. A suite with its own executor needs a matching runner binding and startup checks for its dependencies. Binding should preserve the selected Snapshot and prepare fresh task state for each rollout.

Use the [Harbor notes](../terminal-bench.md) and [harness / benchmark boundary](../benchmark-harness-boundary.md) as references. Archive collected artifacts before removing temporary task environments.

## Register and document it

Export the adapter from [`flashrsi/benchmarks/__init__.py`](../../flashrsi/benchmarks/__init__.py) and add its name to the runner's [`_BENCHMARKS` mapping](../../flashrsi/runner.py). Add an example YAML and a configuration-index entry with the source revision, metric, split counts/policy, dependencies, services, and supported track.

Verify loading and grading offline with small fixtures. For environment integrations, also verify the native execution and verifier path; distinguish those runs from tests that only check interface compatibility.

## Verify the integration

[SearchQA](../../flashrsi/benchmarks/searchqa/__init__.py) is a supplied-context example. Its loader converts each source row into a `Task` with a public question and context, and retains the answer aliases for host-side grading. This offline example uses the [bundled JSONL](../../examples/data/quickstart.jsonl) from the repository root. It needs neither Docker nor model credentials.

```python
from flashrsi.benchmarks.searchqa import SearchQA
from flashrsi.core.trajectory import Trajectory

benchmark = SearchQA({
    "data_path": "examples/data/quickstart.jsonl",
    "split": {"train": 0.5, "dev": 0.25, "test": 0.25},
    "seed": 42,
})
benchmark.load()
assert benchmark.train and benchmark.dev and benchmark.test
task = benchmark.test[0]
correct, empty = benchmark.grade((
    Trajectory(task=task, output=task.ground_truth),
    Trajectory(task=task, output=""),
))
assert correct.metrics[benchmark.primary_metric] == 1.0
assert empty.metrics[benchmark.primary_metric] == 0.0
```

Then use the [small CLI experiment](./algorithm.md#verify-the-integration) to exercise startup, Target execution, and grading. For a new adapter, point the YAML at your fixture and choose an algorithm whose split requirements it satisfies. If the benchmark only exposes test tasks, use [saved-Snapshot evaluation](../guide/evaluation.md) to exercise the Target and grader without evolution. For environment tasks, inspect the native verifier output as well as the aggregate score.
