# Concepts

MoiraWeave is a self-hosted operations platform for AI workloads. It gives one
control plane for model services, pipelines, and agent runtimes.

## Core Terms

| Term | Meaning | Typical ownership |
| --- | --- | --- |
| Workspace | The project where workload manifests, deployment values, and secrets live | You |
| Workload | A deployable AI runtime described by `workload.yaml` | You |
| Model service | A workload that serves synchronous or asynchronous inference | You or a model team |
| Pipeline | A workload whose steps call other workloads | You |
| Agent service | A workload that exposes an agent runtime such as Hermes or OpenClaw | You or the agent runtime |
| Run | One execution request against a workload | MoiraWeave control plane |
| Session | A conversation or durable interaction context for an agent workload | MoiraWeave control plane |
| Event | Timeline entry emitted by API, worker, adapter, or runtime | MoiraWeave control plane |
| Artifact | Metadata for files or outputs produced by a run | Workload produces, MoiraWeave indexes |

## Design Principles

- Workload first: `workload.yaml` is the shared contract for Compose, Kubernetes, API, CLI, worker, and UI.
- Runtime boundary: MoiraWeave operates agents but does not own their reasoning loop, memory, tools, or provider calls.
- Durable control plane: Postgres stores workloads, runs, sessions, messages, events, and artifact metadata.
- Redis as queue only: Redis is used for dispatch and short-lived worker coordination.
- UI/API canonical channel: Telegram, Slack, Discord, and webhooks should enter through MoiraWeave connectors.

## Agent Boundary

MoiraWeave owns:

- deployment metadata
- sessions and messages
- run lifecycle and cancellation
- health and stale-run detection
- events, logs, and artifact metadata

Agent runtimes own:

- planning and reasoning
- tool execution implementation
- internal memory
- runtime-specific configuration
- model provider integrations

## Related Pages

- [Quickstart](quickstart.md)
- [Agent Operations Architecture](agent-ops-architecture.md)
- [Repository Structure](repo-structure.md)
