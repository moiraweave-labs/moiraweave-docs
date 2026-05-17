# Concepts

MoiraWeave separates reusable contracts from workspace-owned implementation.

## Core terms

| Term | Meaning | Typical ownership |
| --- | --- | --- |
| Workspace | The customer project where pipelines, steps, and deployment overlays live | You |
| Task | The contract that defines input and output semantics for a unit of work | You or the official catalog |
| Step | A deployable service that implements a task contract | You or `moiraweave-steps` |
| Pipeline | A declarative workflow that composes steps | You |
| Runtime | The shared services that execute jobs, coordinate work, and expose APIs | `moiraweave-core` |
| Catalog | The curated collection of reusable steps maintained by the team | `moiraweave-steps` |

## Design principles

- Contract first: tasks define the shape of data before implementation details.
- Workspace owned: pipelines and custom steps belong to the project that uses them.
- Generic runtime: the platform executes workloads without embedding domain-specific defaults.
- Explicit ownership: each repository should have a single, obvious responsibility.

## How the pieces fit together

1. You define or reuse a task contract.
2. You implement or reference a step.
3. You compose steps into a pipeline.
4. The runtime executes the pipeline and records job state.
5. You observe and iterate from your workspace.

## Related pages

- [Quickstart](quickstart.md)
- [Architecture Overview](architecture.md)
- [Repository Structure](repo-structure.md)