# Repository Structure and Ownership Model

MoiraWeave is centered on the `moiraweave` product repository and a small set of
supporting repositories. The long-term direction is fewer repositories: keep the
main runtime as the product home, and fold tightly-coupled experiences into it
when the CI/release migration is ready.

## Repository Map

| Repository | Role | Owns |
| --- | --- | --- |
| `moiraweave-cli` | User entry point | Workspace init, workload manifests, runs, agent sessions, and deploy commands |
| Customer workspace | Product implementation | Workload manifests, deployment values, secrets, artifacts, and application code |
| `moiraweave` | Product runtime | API gateway, worker, shared schemas, control-plane storage, Helm, Compose, and telemetry |
| `moiraweave-ui` | Ops dashboard | Workloads, runs, agent chat, artifacts, and health views |
| `moiraweave-docs` | Public documentation | Tutorials, concepts, architecture, and reference material |
| `.github` | Org-level policy | Shared templates, policies, and reusable workflows |

## Ownership Boundaries

### `moiraweave-cli`

Use this repository when changing the user workflow: initialization, workload
manifest generation, run commands, agent session commands, or deployment asset
generation.

### Customer Workspace

This is where product-specific behavior lives. A workspace owns its
`workload.yaml` files, secrets, deployment values, generated Compose or Helm
overlays, local artifacts, and application code.

### `moiraweave`

This repository owns the shared runtime and infrastructure surface. It should
stay generic and should not carry customer-specific agents, models, or secrets.

### `moiraweave-ui`

This repository owns the browser-based operations experience. It talks only to
the API gateway and does not access Redis, Kubernetes, or Postgres directly.

### `moiraweave-docs`

This repository explains how the system works. It should stay aligned with the
public product model: workloads, runs, sessions, events, artifacts, and agent
operations.

## Decision Guide

| Question | Target repository |
| --- | --- |
| Is this customer-specific runtime configuration? | Customer workspace |
| Is this shared platform behavior? | `moiraweave` |
| Is this UI behavior? | `moiraweave-ui` |
| Is this CLI behavior? | `moiraweave-cli` |
| Is this user-facing documentation? | `moiraweave-docs` |
| Is this shared policy or template material? | `.github` |

## Typical Flow

1. Install the CLI.
2. Create a workspace.
3. Run `moira up`.
4. Open the UI and chat with the demo agent.
5. Replace the demo workload with Hermes, OpenClaw, a generic HTTP agent, a model service, or a pipeline.
6. Operate runs, sessions, artifacts, and health from UI or CLI.

You do not normally need to clone `moiraweave` to use the platform.
