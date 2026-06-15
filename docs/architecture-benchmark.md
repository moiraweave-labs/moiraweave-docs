# Architecture Benchmark

Date: 2026-06-15

This page explains where MoiraWeave fits among adjacent systems. The goal is
not to claim MoiraWeave replaces agent frameworks, workflow engines, or model
serving platforms. The product boundary is narrower: MoiraWeave is the
self-hosted control plane that deploys, connects, observes, and operates AI
workloads.

## Positioning

| Area | MoiraWeave role | Reference pattern | Product stance |
| --- | --- | --- | --- |
| Agent runtime | Deploy and supervise Hermes, OpenClaw, LangGraph, or custom HTTP agents | LangGraph/LangSmith | MoiraWeave does not own the reasoning loop, memory, tools, or policy engine inside the agent. It owns deployment records, sessions, messages, runs, events, artifacts, health, cancellation, and audit. |
| Visual app building | Template-guided workload creation and Ops dashboard | Dify | MoiraWeave should stay ops-first. It can offer guided templates and YAML advanced mode, but it should not become a general visual agent builder. |
| Durable workflow execution | Run state, heartbeat, cancellation, stale recovery, and Redis pending reclaim | Temporal | MoiraWeave borrows durable-operation patterns, but it should not recreate a full workflow engine. Pipelines remain workload-level orchestration that calls other workloads. |
| Model serving | Generic `model-service` workloads and endpoints | Ray Serve, KServe | MoiraWeave deploys and calls model services through the same workload model, but specialized serving stacks keep ownership of scaling internals and protocol depth. |
| Local-first deployment | `moira up`, Compose generation, UI, demo agent, local artifacts | Docker Compose | First use should be almost instant and require no external model provider. |
| Kubernetes operations | Helm values, deployment records, preflight, monitoring, and optional controller boundary | Helm, ArgoCD | The browser never receives kubeconfig. Apply/log/undeploy should run through CLI, CI, GitOps, or a future controller. |
| Team operations | Users, teams, API keys, roles, audit, secret inventory | Internal platform consoles | MoiraWeave should expose enough governance for small teams without becoming a full IAM or secret manager. |

## Competitive Gaps To Respect

- LangGraph/LangSmith is stronger for building and debugging agent graphs.
  MoiraWeave should integrate those runtimes instead of competing with their
  graph semantics.
- Dify is stronger for no-code application creation. MoiraWeave should keep the
  UI focused on operations: create from template, deploy/connect, chat, observe,
  diagnose, and recover.
- Temporal is stronger for arbitrary long-running workflows. MoiraWeave should
  keep durable run semantics scoped to AI workloads and agent turns.
- Ray Serve and KServe are stronger for high-scale inference serving internals.
  MoiraWeave should provide a consistent deployment and operations surface for
  model-service workloads.

## Current Strengths

- One workload manifest drives local Compose, Kubernetes values, API
  registration, preflight, and worker execution.
- `moira up` gives an empty workspace a working platform, UI, and demo agent.
- The UI can create workloads from templates, chat with agents, inspect linked
  runs/events/artifacts, run preflight, and manage deployment records.
- Long-running agents are modeled with heartbeats, stale-run detection,
  cooperative cancellation, and safe Redis pending-message recovery.
- Hermes/OpenClaw-style tools stay inside the runtime boundary. MoiraWeave
  declares and displays the capability boundary without reimplementing web
  search, browser control, terminal access, MCP servers, or native messaging.
- GHCR image builds and public pull smoke tests are automated through GitHub
  Actions.

## Priority Follow-Ups

1. Certify real Hermes and OpenClaw integration with optional E2E runs gated by
   `MOIRAWEAVE_REAL_AGENT_TESTS=1`, including startup, session message, events,
   cancellation, artifacts, and failure diagnostics.
2. Add a Kubernetes deployment controller/operator contract for apply, logs,
   and undeploy so the UI can trigger operations without receiving cluster
   credentials.
3. Expand Operations Center into a release dashboard: environment comparison,
   last operation, preflight status, runtime health, and recommended next action
   per workload.
4. Add policy metadata for agent capabilities: network egress, browser,
   terminal, filesystem/workspace, MCP, native channels, and max runtime.
5. Add production-grade secret inventory integrations beyond local environment
   checks: Kubernetes Secrets, External Secrets Operator, and external-owned
   secret references.
6. Publish a real-agent compatibility matrix with runtime versions, required
   ports, secrets, health endpoints, adapter paths, supported channels, and known
   limitations.
