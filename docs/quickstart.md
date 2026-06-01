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

## 2. Start The Local Product

```bash
mkdir my-moiraweave-workspace
cd my-moiraweave-workspace
moira up
```

`moira up` initializes the workspace if needed, creates a no-secret demo agent
when there are no workloads, generates local workload Compose services, runs
`moira doctor`, starts the platform and workloads, waits for API readiness, and
registers workload/deployment records.

The generated workspace contains:

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

Open `http://localhost:3000/agents`, then sign in with:

```text
admin / demo-password
```

The local demo user has the `admin` role by default. For automation, set
`MOIRA_API_KEYS` with comma-separated `key:subject:role` entries and use the key
as a bearer token. The dashboard resolves the active credential with
`GET /auth/me`, shows the current role in the header, and disables actions that
need a higher role before they fail server-side.

For a terminal-only smoke test, use one command that creates a session when
needed, sends the message, and watches the associated run:

```bash
moira agent chat demo-agent "hello from the CLI" --watch
```

If the local stack does not start cleanly, run:

```bash
moira doctor
```

For automation, use `moira doctor --json`.

If you run development builds or private registries, override platform images in
`.env` with `MOIRAWEAVE_API_GATEWAY_IMAGE`, `MOIRAWEAVE_WORKER_IMAGE`, and
`MOIRAWEAVE_UI_IMAGE`. `moira doctor` checks whether required images are locally
available or pullable before Docker starts.

## 3. Use The Product Flow

In the dashboard:

- Workloads: create workloads from templates, then jump directly to preflight
  or the agent console.
- Operations: run preflight, view recommended actions, inspect deployment
  records, view deployment plans, sync deployment records, and review
  deployment operation history.
- Agents: the first agent and existing session are selected automatically; start
  the first session from the empty-state CTA, send a message, cancel/retry a
  turn, and follow the exact linked run status even when prompts repeat.
- Runs: inspect event timelines, payloads, results, cancellation, and artifacts.
- Artifacts: browse by workload, session, run, and content type; inspect
  metadata, preview/download local or PVC-backed files, follow run links, and
  copy artifact references.

The simple product loop is:

```text
create workload -> deploy/connect runtime -> interact -> observe -> correct
```

## Hermes And OpenClaw

For real agent runtimes, create them from the UI templates or CLI manifests.
`moira init` and `moira demo agent` remain available when you want explicit
setup steps, but they are not required for the first local demo.

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
| `moira up` stops before Docker starts | `moira doctor` found a blocking local issue | Fix the ERROR rows from `moira doctor`, then rerun `moira up` |
| `moira up` cannot start containers | Docker is stopped or the port is busy | Run `moira doctor`, start Docker, and check ports 8000/3000/5432/6379 |
| `moira doctor` reports container images unavailable | The image is private, unpublished, or the registry login is missing | Publish/login to the registry or override `MOIRAWEAVE_*_IMAGE` in `.env` |
| `moira up` reports missing environment variables | Required workload secrets are not available locally | Run `moira doctor` or `moira secrets list`, then add missing names to `.env` or export them |
| Login fails | Local demo password was overridden | Check `DEMO_USERNAME` and `DEMO_PASSWORD` in `.env` |
| API request returns `403` | The token role is too limited | Use an `operator` or `admin` token for mutating actions |
| Workload is created but not healthy | Runtime service is missing or not reachable | Use Operations preflight and workload logs |
| Agent message stays queued | Worker or Redis is unavailable | Check `/ready`, worker logs, and Redis connectivity |
| Agent run fails after dispatch timeout | Runtime did not acknowledge quickly | Configure adapter paths or return an early ack before long work |
