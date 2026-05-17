# Architecture

MoiraWeave is designed around a narrow runtime and explicit ownership boundaries.

## The three layers

| Layer | Responsibility | Owned by |
| --- | --- | --- |
| Workspace layer | Pipelines, custom steps, task contracts, and environment overlays | The team using MoiraWeave |
| Catalog layer | Curated reusable steps and official task implementations | `moiraweave-steps` |
| Runtime layer | API gateway, worker, orchestration, persistence, observability, and deployment primitives | `moiraweave-core` |

## What each layer does

### Workspace layer

- Defines the business workflow.
- Stores pipeline YAML and custom step code.
- Owns deployment-specific values and secrets.
- Changes frequently and should remain isolated from the shared runtime.

### Catalog layer

- Publishes reusable step implementations.
- Keeps versioned contracts stable for consumers.
- Serves as an optional acceleration path, not a hard dependency.

### Runtime layer

- Exposes the API used to submit, track, and inspect jobs.
- Dispatches work to the configured step services.
- Persists state and emits telemetry.
- Stays generic so it can execute any compatible workspace pipeline.

## End-to-end flow

1. A user submits a pipeline job through the CLI or API.
2. The gateway validates the request and stores the job record.
3. The worker consumes the job from Redis Streams.
4. The runtime resolves the pipeline definition and step endpoints from the workspace.
5. Each step executes against its contract and returns data for the next stage.
6. The runtime records completion, failure, and trace information for inspection.

## Design decisions

- Keep pipelines declarative so orchestration is inspectable.
- Keep the runtime generic so no default domain pipelines leak into the platform.
- Keep task contracts explicit so compatibility failures appear early.
- Keep the workspace as the unit of customization so teams can own their delivery lifecycle.

## Further reading

- [Concepts](concepts.md)
- [Repository Structure](repo-structure.md)
- [Architecture Benchmark](architecture-benchmark.md)

See the [architecture diagram](assets/architecture.svg) for a compact visual summary.
