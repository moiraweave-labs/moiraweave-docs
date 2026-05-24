# Architecture

MoiraWeave is designed around a narrow runtime and explicit ownership
boundaries. It operates AI workloads without embedding customer business logic
or agent internals.

## Layers

| Layer | Responsibility | Owned by |
| --- | --- | --- |
| Workspace layer | Workload manifests, deployment values, artifacts, and secrets | The team using MoiraWeave |
| Runtime layer | API gateway, worker, control-plane storage, queueing, deployment templates, and observability | `moiraweave-core` |
| Experience layer | CLI and integrated Ops dashboard | `moiraweave-cli`, `moiraweave-ui` |

## Runtime Services

- API gateway: auth, workload registration, run submission, sessions, messages, events, artifacts, and health.
- Worker: consumes Redis dispatch messages and calls model, pipeline, or agent executors.
- Postgres: source of truth for workloads, runs, sessions, messages, events, and artifact metadata.
- Redis Streams: queue and short-lived coordination layer.
- Qdrant: optional vector store for RAG/search workloads.
- UI: browser console for workloads, runs, agent sessions, artifacts, and deployment health.

## End-to-End Run Flow

1. A user submits a workload run through CLI, UI, or API.
2. The API stores the run in Postgres and dispatches a message to Redis Streams.
3. The worker consumes the message and marks the run `starting` then `running`.
4. The executor calls the workload according to its type.
5. The worker stores events, assistant messages, artifacts, result, and final state.
6. UI and CLI read from the API only.

## Agent Flow

Agent workloads use an adapter. The adapter sends a short-lived dispatch call to
the agent runtime, then MoiraWeave tracks the run through stored state and
events. Hermes, OpenClaw, LangGraph, or custom agents keep their own internal
reasoning loop.

Agent runtimes can be placed in two ways. Managed runtimes are deployed by
MoiraWeave as Docker Compose services or Kubernetes Deployments in the same
network/namespace as the worker. External runtimes are not deployed by
MoiraWeave; the manifest records `spec.endpoint`, and the adapter uses that
URL.

## Design Decisions

- Use one `workload.yaml` model for Compose, Kubernetes, API validation, and worker dispatch.
- Use stable workload service names so local and Kubernetes deployments resolve the same way.
- Keep Postgres as the durable control plane.
- Keep Redis out of durable state.
- Keep UI/API as the canonical interaction surface.
- Model Telegram, Slack, Discord, and webhooks as connectors into MoiraWeave, not direct agent access.

## Further Reading

- [Concepts](concepts.md)
- [Agent Operations Architecture](agent-ops-architecture.md)
- [Repository Structure](repo-structure.md)
