# Repository Structure and Ownership Model

MoiraWeave is split into a few small repositories with explicit ownership.

## Repository map

| Repository | Role | Owns |
| --- | --- | --- |
| `moiraweave-cli` | User entry point and scaffolding | Project initialization, step and pipeline commands, local deployment UX |
| Customer workspace | Product implementation | Custom pipelines, custom steps, task contracts, deployment overlays, secrets |
| `moiraweave-core` | Platform runtime | API gateway, worker, shared runtime code, infra templates, telemetry |
| `moiraweave-steps` | Official catalog | Reusable step implementations and official task schemas |
| `moiraweave-docs` | Public documentation | Tutorials, how-to guides, architecture, and reference material |
| `.github` | Org-level policy | Shared templates, policies, and reusable workflows |

## Ownership boundaries

### `moiraweave-cli`

Use this repository when you are changing the developer workflow itself: initialization, command behavior, or packaging of the CLI.

### Customer workspace

This is where business logic lives. A workspace owns pipelines, custom steps, task schemas, and environment-specific deployment settings.

### `moiraweave-core`

This repository owns the shared runtime and infrastructure surface. It should stay generic and should not carry customer pipelines or step-specific defaults.

### `moiraweave-steps`

This repository is the official step catalog. It contains reusable implementations that can be consumed by workspaces, but it is not required for a normal workspace-only deployment.

### `moiraweave-docs`

This repository explains how the system works. It should stay clear, task-oriented, and aligned with the public ownership model.

### `.github`

This repository stores organization-wide policy and workflow reuse.

## Decision guide

| Question | Target repository |
| --- | --- |
| Is this customer-specific workflow or configuration? | Customer workspace |
| Is this reusable platform behavior? | `moiraweave-core` |
| Is this an official reusable step? | `moiraweave-steps` |
| Is this CLI behavior? | `moiraweave-cli` |
| Is this user-facing documentation? | `moiraweave-docs` |
| Is this shared policy or template material? | `.github` |

## Typical flow

1. Install the CLI.
2. Create a workspace.
3. Define or reuse tasks and steps.
4. Compose a pipeline.
5. Validate locally.
6. Deploy against the shared runtime.

You do not normally need to clone `moiraweave-core` or `moiraweave-steps` to use the platform.
