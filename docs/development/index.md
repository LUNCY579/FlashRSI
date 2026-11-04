# Add an integration

Start with the guide for the component you are adding. Use the closest existing implementation as a reference and keep your changes within that component's responsibilities.

| Component | What it owns | Start here |
| --- | --- | --- |
| Algorithm | Proposal, evaluation budget, selection, and the resources it may edit | [Add an algorithm](./algorithm.md), using OPRO |
| Benchmark | Tasks, splits, task environment, and grading | [Add a benchmark](./benchmark.md), using SearchQA and Terminal-Bench |
| Harness | Native agent execution and the resources that make up a Snapshot | [Add a harness](./harness.md), using MiniAgent |

The runner connects these components. The algorithm selects a Snapshot, and the runner evaluates that version on held-out test tasks. A Snapshot carries harness resources. The destination benchmark supplies its own task data, tools, and verifier when the Snapshot is evaluated elsewhere. See the [framework overview](./framework.md) and the [MVP contract](../../mvp_doc.md) for the interfaces.

For a contribution, include a configuration example, the supported capabilities and prerequisites, and the validation you performed. Distinguish an offline check, successful initialization, and a real run that reached the grader.
