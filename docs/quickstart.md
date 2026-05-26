# Quickstart

This guide gets you from an empty directory to a local MoiraWeave stack with a
working demo agent, dashboard, runs, events, and artifacts.

## Prerequisites

- Python 3.13 or newer
- `uv`
- Docker and Docker Compose

## 1. Install The CLI

```bash
uv tool install moiraweave-cli
moira --help
```

## 2. Create A Workspace

```bash
mkdir my-moiraweave-workspace
cd my-moiraweave-workspace
moira init --non-interactive
```

The workspace contains:

```text
.moiraweave/
  workloads/
  artifacts/
  deploy/
moiraweave.yaml
.env
docker-compose.yml
```

The generated Compose stack includes API Gateway, worker, Postgres, Redis,
Qdrant, and the Ops dashboard.

## 3. Add A Demo Agent

```bash
moira demo agent
```

This writes `.moiraweave/workloads/demo-agent/workload.yaml`. It uses a local
mock HTTP agent, so the first run needs no OpenAI key, Hermes runtime, or
OpenClaw runtime.

## 4. Start Everything

```bash
moira up
```

`moira up` initializes the workspace if needed, generates local workload Compose
services, starts the platform and workloads, waits for API readiness, and
registers workload/deployment records.

Open `http://localhost:3000/agents`, then sign in with:

```text
admin / demo-password
```

## 5. Use The Product Flow

In the dashboard:

- Workloads: create workloads from templates or inspect registered manifests.
- Operations: run preflight, view recommended actions, inspect deployment
  records, view deployment plans, sync deployment records, and review
  deployment operation history.
- Agents: the first agent and existing session are selected automatically; create
  a session, send a message, cancel/retry a turn, and follow the exact linked
  run status even when prompts repeat.
- Runs: inspect event timelines, payloads, results, cancellation, and artifacts.
- Artifacts: browse by workload, session, run, and content type.

The simple product loop is:

```text
create workload -> deploy/connect runtime -> interact -> observe -> correct
```

## Hermes And OpenClaw

For real agent runtimes, create them from the UI templates or CLI manifests:

```bash
moira workload new hermes \
  --type agent-service \
  --image ghcr.io/nousresearch/hermes-agent:latest \
  --service-name hermes \
  --mode session \
  --timeout-seconds 172800 \
  --adapter hermes \
  --port 8642 \
  --secret OPENAI_API_KEY \
  --secret HERMES_API_SERVER_KEY \
  --auth-token-env HERMES_API_SERVER_KEY \
  --persistence \
  --mount-path /workspace \
  --workspace-mount /workspace
```

MoiraWeave deploys and supervises the runtime, but Hermes/OpenClaw keep their
own reasoning loop, tools, memory, and runtime-specific configuration.

Before starting the stack, inspect required names without exposing values:

```bash
moira secrets list --workload hermes
```

## CLI Boundaries

The UI never talks directly to Docker, Kubernetes, Redis, or the filesystem.
Local execution runs through `moira up`, `moira deploy local`, and Docker
Compose. Kubernetes execution should run through CLI, CI, or a deployment
controller/operator.

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `moira` command not found | CLI is not installed in the active shell | Re-run `uv tool install moiraweave-cli` |
| `moira up` cannot start containers | Docker is stopped or the port is busy | Start Docker and check ports 8000/3000/5432/6379 |
| `moira up` reports missing environment variables | Required workload secrets are not available locally | Run `moira secrets list`, then add the missing names to `.env` or export them |
| Login fails | Local demo password was overridden | Check `DEMO_USERNAME` and `DEMO_PASSWORD` in `.env` |
| Workload is created but not healthy | Runtime service is missing or not reachable | Use Operations preflight and workload logs |
| Agent message stays queued | Worker or Redis is unavailable | Check `/ready`, worker logs, and Redis connectivity |
| Agent run fails after dispatch timeout | Runtime did not acknowledge quickly | Configure adapter paths or return an early ack before long work |
