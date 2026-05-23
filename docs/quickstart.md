# Quickstart

This guide gets you from an empty workspace to a local agent workload.

## Prerequisites

- Python 3.13 or newer
- `uv`
- Docker and Docker Compose

## 1. Install the CLI

```bash
uv tool install moiraweave-cli
moira --help
```

## 2. Create a Workspace

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

The generated Compose stack includes the Ops dashboard. By default it is served
at `http://localhost:3000` and proxies API calls to the gateway service inside
the Compose network.

## 3. Create an Agent Workload

```bash
moira workload new hermes \
  --type agent-service \
  --image ghcr.io/nousresearch/hermes-agent:latest \
  --mode session \
  --timeout-seconds 172800 \
  --adapter hermes \
  --port 8642 \
  --env API_SERVER_ENABLED=true \
  --env API_SERVER_HOST=0.0.0.0 \
  --env API_SERVER_PORT=8642 \
  --env 'API_SERVER_KEY=${HERMES_API_SERVER_KEY}' \
  --secret OPENAI_API_KEY \
  --secret HERMES_API_SERVER_KEY \
  --auth-token-env HERMES_API_SERVER_KEY \
  --model hermes-agent \
  --persistence \
  --mount-path /data \
  --workspace-mount /workspace
```

This writes `.moiraweave/workloads/hermes/workload.yaml`.

## 4. Generate Local Deployment Assets

```bash
moira deploy local
```

Set required secrets in `.env`, then start the platform and workload services:

```bash
docker compose -f docker-compose.yml -f .moiraweave/deploy/docker-compose.workloads.yml up -d
```

## 5. Register And Run

```bash
export MOIRA_TOKEN=<token>
moira workload deploy hermes
moira workload status hermes
moira agent session create hermes
moira agent session message hermes <session-id> "hello" --watch
```

The API stores the session, user message, queued run, worker events, assistant
message, and any artifact metadata.

## 6. Use The Dashboard

Open `http://localhost:3000` and inspect:

- Workloads
- Runs and run timeline
- Agent sessions and chat
- Artifacts
- Deployment health

To test the channel gateway shape without a real Telegram or Slack bot:

```bash
moira agent channel-message hermes telegram user-123 "hello from telegram"
```

This creates or reuses a MoiraWeave agent session, records channel audit
metadata, queues a run, and keeps the interaction visible in the dashboard.

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `moira` command not found | CLI is not installed in the active shell | Re-run `uv tool install moiraweave-cli` |
| Workload is registered but not reachable | Compose or Kubernetes workload service is not deployed | Run `moira deploy local` or inspect workload logs |
| Agent message stays queued | Worker is not running or Redis is unavailable | Check `/ready`, worker logs, and Redis connectivity |
| Agent run fails after dispatch timeout | Runtime did not acknowledge quickly | Configure the adapter path or make the agent return an ack before long work |
