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
| Environment | A deployment scope such as `local`, `dev`, `staging`, or `prod` | You choose, MoiraWeave records |
| Audit event | Durable record of a sensitive platform action | MoiraWeave control plane |

## Design Principles

- Workload first: `workload.yaml` is the shared contract for Compose, Kubernetes, API, CLI, worker, and UI.
- Runtime boundary: MoiraWeave operates agents but does not own their reasoning loop, memory, tools, or provider calls.
- Durable control plane: Postgres stores workloads, runs, sessions, messages, events, artifact metadata, and audit events.
- Redis as queue only: Redis is used for dispatch and short-lived worker coordination.
- UI/API canonical channel: MoiraWeave owns dashboard/API sessions; Telegram, Slack, Discord, and webhooks can stay runtime-owned and supervised when duplicating them in the UI is not worth the product complexity.
- Environment-scoped operations: deployment records, deployment operations,
  workload health, and preflight can be filtered by environment so local tests do
  not hide production state.
- Audit actions, not secrets: deploy operations, run cancellation, agent/channel messages, and artifact access are traceable without storing secret values.
- Secret names only: manifests declare required secret names, while values stay in
  `.env`, Kubernetes Secrets, or an external secret manager. The API, CLI, and
  UI show presence and missing names, never secret values.
- Runtime-owned tools: `spec.agent.runtimeRequirements` describes filesystem,
  network, browser, terminal, MCP, messaging, and web-search boundaries that
  MoiraWeave should prepare or validate. It is not a tool registry and
  MoiraWeave does not route individual tool calls.

## Agent Boundary

MoiraWeave owns:

- deployment metadata
- sessions and messages
- run lifecycle and cancellation
- health and stale-run detection
- events, logs, and artifact metadata
- audit events for sensitive control-plane actions
- required secret names and preflight status
- runtime requirement validation for workspace, egress, channels, and secrets

Agent runtimes own:

- planning and reasoning
- tool execution implementation
- internal memory
- runtime-specific configuration
- model provider integrations
- browser, terminal, web-search, MCP, and messaging behavior when declared as
  runtime-owned

## Related Pages

- [Quickstart](quickstart.md)
- [Agent Operations Architecture](agent-ops-architecture.md)
- [Repository Structure](repo-structure.md)
