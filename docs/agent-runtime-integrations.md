# Agent Runtime Integrations

MoiraWeave integrates agent runtimes as deployable workloads with an operations
contract around them: dispatch, session correlation, status, cancellation,
events, artifacts, and health. The runtime keeps ownership of planning, tools,
memory, provider configuration, and channel-specific behavior.

## Integration Rule

Use the runtime's public control surface.

- Hermes Agent: HTTP API server with OpenAI-compatible endpoints, `/v1/runs`,
  run status, run events, stop, and health.
- OpenClaw: Gateway WebSocket protocol v4 with JSON `req`, `res`, and `event`
  frames, operator scopes, sessions, chat, task, artifact, and health methods.
- Custom agents: `generic-http` with explicit message/status/cancel/artifact
  paths in `workload.yaml`.

MoiraWeave should not import runtime internals, patch agent memory, or call tool
implementations directly. That keeps upgrades and agent-specific configuration
inside the runtime.

## Versioned Examples

The `moiraweave` repository includes canonical workload examples under
`examples/workloads`:

- `hermes/workload.yaml`: managed Hermes Agent with HTTP `/health` probes,
  workspace persistence, OpenAI secret reference, and runtime-owned tools.
- `openclaw/workload.yaml`: managed OpenClaw Gateway with TCP probes,
  workspace persistence, gateway token secret reference, and runtime-owned
  browser/tool capabilities.
- `external-hermes/workload.yaml`: externally deployed Hermes runtime that
  MoiraWeave supervises through an endpoint without deploying the process.

Copy one into `.moiraweave/workloads/<name>/workload.yaml`, then run:

```bash
moira deploy local
moira up
moira workload deploy <name> --target local
moira agent session create <name>
```

Hermes and OpenClaw can be deployed together as separate workloads. Their
service names and ports must be unique, and each runtime should get its own
workspace and token secret.

## Compatibility Matrix

| Runtime | Adapter | Protocol | Default endpoint | Required secrets | Health check | Cancellation | Artifact discovery | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Hermes Agent | `hermes` | HTTP JSON API | `http://hermes:8642` | `OPENAI_API_KEY`, optional `HERMES_API_SERVER_KEY` | `/health` for probes, `/health/detailed` for adapter status | `POST /v1/runs/{run_id}/stop` | `artifacts` returned by `GET /v1/runs/{run_id}` | Best fit for MoiraWeave because it exposes run ids, status, cancellation, and health over HTTP. |
| OpenClaw Gateway | `openclaw` | WebSocket JSON-RPC v4 | `http://openclaw:18789` | optional `OPENCLAW_GATEWAY_TOKEN` | TCP probe plus Gateway `health` RPC | `sessions.abort` or `chat.abort` fallback | `artifacts.list` when supported by the gateway | Requires gateway/operator protocol support; MoiraWeave maps session ids to OpenClaw session keys. |
| Generic HTTP Agent | `generic-http` | HTTP JSON | workload `spec.endpoint` or first declared port | workload-defined | `statusPath` or `/health` | `cancelPath` or `/cancel` | `artifactsPath` or `/artifacts` | Use for custom agents that expose simple message/status/cancel/artifact endpoints. |
| External Hermes/OpenClaw | `hermes` or `openclaw` | Runtime-native remote endpoint | workload `spec.endpoint` | runtime-owned or referenced by `authTokenEnv` | adapter status call against remote endpoint | adapter-specific | adapter-specific | MoiraWeave records `target: external` and supervises the runtime, but does not deploy it. |

Compatibility means MoiraWeave can dispatch a session message, correlate a run,
poll or stream progress, request cancellation, discover artifacts, and report
health. It does not mean MoiraWeave owns the runtime's tool permissions,
provider keys, memory, browser process, terminal backend, MCP servers, or
native messaging bridges.

## Tool Ownership

Agent tools stay runtime-owned. MoiraWeave prepares the runtime boundary:
workspace, persistence, secrets, egress, resources, health, events, artifacts,
and cancellation. It does not decide when Hermes/OpenClaw uses web search,
browser automation, terminal backends, MCP servers, memory, or native channels.

Use `spec.agent.toolOwnership: runtime` and `spec.agent.runtimeRequirements` to
declare what the runtime needs. Preflight checks this declaration for obvious
environment conflicts, but it does not manage the tools themselves.

## Deployment Placement

Agent workloads have two placement modes:

- `deployment.mode: managed`: MoiraWeave deploys the runtime next to the
  control plane. In Docker Compose the service joins `moiraweave-net`. In
  Kubernetes the Helm chart creates an in-namespace Deployment, Service, and PVC
  when requested.
- `deployment.mode: external`: MoiraWeave does not deploy the runtime. The
  manifest must set `spec.endpoint`; the worker talks to that endpoint through
  the selected adapter.

Managed workloads use a stable service name. By default it is
`metadata.name`, so the same manifest resolves as `http://hermes:8642` in
Compose and Kubernetes. Set `deployment.serviceName` only when the runtime must
use a different DNS name.

Multiple agents are represented as multiple workloads. For example, `hermes`
and `openclaw` can be deployed together as separate services as long as each
workload has its own `metadata.name`, service name, ports, secrets, and adapter
configuration. MoiraWeave sessions and runs are scoped by workload name, so two
agents can run concurrently without sharing conversation ids or deployment
records.

## Channel Ownership

Every MoiraWeave-owned inbound channel must be declared in the workload
manifest before MoiraWeave accepts messages for it. The API gateway normalizes
channel names to lowercase and only accepts
`/v1/channels/{channel}/agents/{name}/messages` when `channel` is listed in
`spec.agent.exposedChannels`. Webhook connector services can use the alias
`/v1/webhooks/{channel}/agents/{name}/messages`; it follows the same auth,
session, run, audit, and channel-ownership rules.

Use `externalOwnedChannels` for integrations that the runtime owns itself. For
example, if a Hermes profile already runs its own Telegram bridge, declare
`externalOwnedChannels: [telegram]`; MoiraWeave will show that ownership in the
manifest/health context, but it will reject MoiraWeave-owned inbound messages
for that channel so traffic does not split across two controllers. In that
mode, users talk directly through Telegram and MoiraWeave remains the
deployment, health, runs, events, cancellation, and artifact plane. Telegram
messages are not mirrored into the MoiraWeave chat UI unless a dedicated
connector/audit bridge is added later.

## Hermes Agent

Hermes is the cleanest runtime to integrate because its API server exposes the
exact shape an operations platform needs:

- `POST /v1/runs` accepts a short request and returns a `run_id`.
- `GET /v1/runs/{run_id}` exposes terminal state, output, usage, and session
  correlation.
- `GET /v1/runs/{run_id}/events` exposes SSE progress for tool calls and
  lifecycle.
- `POST /v1/runs/{run_id}/stop` requests cooperative interruption.
- `/health` and `/health/detailed` expose process and active-agent health.

Recommended workload:

```yaml
apiVersion: moiraweave.io/v1alpha1
kind: Workload
metadata:
  name: hermes
spec:
  type: agent-service
  image: ghcr.io/nousresearch/hermes-agent:latest
  deployment:
    mode: managed
    targets: [local, kubernetes]
    serviceName: hermes
    replicas: 1
  execution:
    mode: session
    timeoutSeconds: 172800
  ports:
    - name: http
      port: 8642
  livenessProbe:
    httpGet:
      path: /health
      port: http
    initialDelaySeconds: 15
    periodSeconds: 30
    timeoutSeconds: 5
    failureThreshold: 3
  readinessProbe:
    httpGet:
      path: /health
      port: http
    initialDelaySeconds: 5
    periodSeconds: 10
    timeoutSeconds: 5
    failureThreshold: 6
  env:
    API_SERVER_ENABLED: "true"
    API_SERVER_HOST: "0.0.0.0"
    API_SERVER_PORT: "8642"
  secrets:
    - OPENAI_API_KEY
  persistence:
    enabled: true
    mountPath: /workspace
  agent:
    adapter: hermes
    toolOwnership: runtime
    authTokenEnv: HERMES_API_SERVER_KEY
    model: hermes-agent
    workspaceMount: /workspace
    exposedChannels: [ui, api]
    externalOwnedChannels: [telegram]
    runtimeRequirements:
      filesystem:
        persistentWorkspace: true
        workspaceMount: /workspace
      network:
        egress: enabled
      webSearch:
        enabled: true
      browser:
        mode: runtime-managed
      terminal:
        mode: runtime-managed
        approval: runtime
      mcp:
        enabled: true
      messaging:
        enabled: true
    dispatchTimeoutSeconds: 10
    pollIntervalSeconds: 2
```

`authTokenEnv` is treated as a secret environment variable by Docker Compose and
Helm generation. Use the env var name that the runtime reads for its API token.
For multiple Hermes profiles, prefer one token env var per workload so each
runtime can rotate independently.

Kubernetes probes use `/health` because it is safe for unauthenticated process
readiness. The Hermes adapter can still call `/health/detailed` with the
runtime token during preflight and runtime status checks when that endpoint is
available.

## OpenClaw

OpenClaw is not an HTTP agent endpoint. Its stable integration surface is the
Gateway WebSocket protocol. MoiraWeave therefore connects as an operator client,
sends a `connect` request with protocol v4, discovers supported methods from
the `hello-ok.features.methods` payload, and then uses the Gateway RPC surface.

The adapter prefers these methods when advertised:

- `sessions.describe` and `sessions.create` to resolve or create the target
  session.
- `sessions.send` to dispatch a message, with `chat.send` as compatibility
  fallback when the Gateway does not advertise `sessions.send`.
- `agent.wait`, then `tasks.get`, then `sessions.describe` for status
  reconciliation.
- `sessions.abort`, with `chat.abort` as compatibility fallback.
- `artifacts.list` for transcript-derived artifacts.
- `health` for gateway health checks.

Recommended workload:

```yaml
apiVersion: moiraweave.io/v1alpha1
kind: Workload
metadata:
  name: openclaw
spec:
  type: agent-service
  image: ghcr.io/openclaw/openclaw:latest
  deployment:
    mode: managed
    targets: [local, kubernetes]
    serviceName: openclaw
    replicas: 1
  execution:
    mode: session
    timeoutSeconds: 172800
  ports:
    - name: gateway
      port: 18789
  livenessProbe:
    tcpSocket:
      port: gateway
    initialDelaySeconds: 15
    periodSeconds: 30
    timeoutSeconds: 5
    failureThreshold: 3
  readinessProbe:
    tcpSocket:
      port: gateway
    initialDelaySeconds: 5
    periodSeconds: 10
    timeoutSeconds: 5
    failureThreshold: 6
  persistence:
    enabled: true
    mountPath: /workspace
  agent:
    adapter: openclaw
    toolOwnership: runtime
    authTokenEnv: OPENCLAW_GATEWAY_TOKEN
    agentId: main
    workspaceMount: /workspace
    exposedChannels: [ui, api]
    runtimeRequirements:
      filesystem:
        persistentWorkspace: true
        workspaceMount: /workspace
      network:
        egress: enabled
      webSearch:
        enabled: true
      browser:
        mode: runtime-managed
      terminal:
        mode: runtime-managed
        approval: runtime
      mcp:
        enabled: true
      messaging:
        enabled: true
    dispatchTimeoutSeconds: 10
    pollIntervalSeconds: 2
```

MoiraWeave session ids become OpenClaw session keys in the form
`agent:{agentId}:{session_id}` unless the payload already provides a
runtime-native `session_key`.

Kubernetes probes check that the Gateway socket is reachable. Protocol health is
still checked by the OpenClaw adapter through the Gateway `health` RPC, so
MoiraWeave can distinguish "process listening" from "agent protocol healthy".

Docker Compose does not inject image-dependent `curl` or shell healthchecks into
third-party agent images. Local reachability is checked by `moira doctor`,
preflight, and adapter health calls through the MoiraWeave API.

External OpenClaw or Hermes runtimes should use the same `agent` block but set:

```yaml
spec:
  type: agent-service
  endpoint: https://agents.example.com/hermes
  deployment:
    mode: external
  agent:
    adapter: hermes
    toolOwnership: runtime
    runtimeRequirements:
      network:
        egress: restricted
```

When the manifest is registered with `moira deploy local --register` or
`moira deploy k8s --register`, external agents are recorded as
`target: external` with their `spec.endpoint`. That lets the dashboard and
health API show the runtime location even though deployment is owned outside
MoiraWeave.

## Long-Running Behavior

Agent turns can run for hours or days. The production pattern is:

1. API creates a MoiraWeave run and stores the user message with that exact
   `run_id` so repeated prompts still map to distinct turns.
2. Worker dispatches a short request to the runtime and records the external
   run id or session key.
3. Worker emits MoiraWeave events while polling or streaming runtime progress.
4. Heartbeat keeps the MoiraWeave run alive while the runtime works.
5. Cancellation becomes a cooperative adapter call.
6. Artifacts are discovered through the runtime adapter and recorded in
   Postgres metadata.

For very high concurrency, the next evolution is to split dispatch and
reconciliation into separate queues so a worker process does not stay attached
to every long-running agent turn.

## Runtime Test Levels

MoiraWeave uses three test levels for agent integrations:

- Contract tests in CI mock the Hermes HTTP API and OpenClaw Gateway protocol.
  These prove request shapes, status polling, cancellation, artifact discovery,
  and fallback behavior without requiring paid providers or long-running
  runtime containers.
- Compose E2E tests use mock agents and models to validate the MoiraWeave
  control plane, Redis dispatch, event storage, cancellation, and artifacts.
- Optional live-runtime tests can be run against real Hermes/OpenClaw endpoints.
  They are skipped by default and enabled with environment variables. The basic
  health tests do not send an agent turn:

```bash
MOIRAWEAVE_REAL_AGENT_TESTS=1 \
MOIRAWEAVE_REAL_HERMES_URL=http://localhost:8642 \
MOIRAWEAVE_REAL_OPENCLAW_URL=http://localhost:18789 \
make test-real-agents
```

Use `MOIRAWEAVE_REAL_HERMES_AUTH_TOKEN_ENV` or
`MOIRAWEAVE_REAL_OPENCLAW_AUTH_TOKEN_ENV` when the runtime requires a bearer
token. The variable value should be the name of the env var containing the
token, for example `HERMES_API_SERVER_KEY`.

Turn tests are intentionally gated by separate flags because they create real
runtime work, may call external model providers, and also exercise adapter
artifact discovery after the turn completes:

```bash
MOIRAWEAVE_REAL_AGENT_TESTS=1 \
MOIRAWEAVE_REAL_HERMES_URL=http://localhost:8642 \
MOIRAWEAVE_REAL_HERMES_TURN_TEST=1 \
MOIRAWEAVE_REAL_HERMES_MESSAGE="Reply with the single word moiraweave-ok." \
make test-real-agents
```

For OpenClaw, use:

```bash
MOIRAWEAVE_REAL_AGENT_TESTS=1 \
MOIRAWEAVE_REAL_OPENCLAW_URL=http://localhost:18789 \
MOIRAWEAVE_REAL_OPENCLAW_AGENT_ID=main \
MOIRAWEAVE_REAL_OPENCLAW_TURN_TEST=1 \
make test-real-agents
```

`MOIRAWEAVE_REAL_AGENT_TURN_TIMEOUT_SECONDS` defaults to `120` and can be
raised for slower long-running agent profiles.

Cancellation tests are gated separately because they intentionally interrupt
live runtime work:

```bash
MOIRAWEAVE_REAL_AGENT_TESTS=1 \
MOIRAWEAVE_REAL_HERMES_URL=http://localhost:8642 \
MOIRAWEAVE_REAL_HERMES_CANCEL_TEST=1 \
make test-real-agents
```

For OpenClaw, use:

```bash
MOIRAWEAVE_REAL_AGENT_TESTS=1 \
MOIRAWEAVE_REAL_OPENCLAW_URL=http://localhost:18789 \
MOIRAWEAVE_REAL_OPENCLAW_AGENT_ID=main \
MOIRAWEAVE_REAL_OPENCLAW_CANCEL_TEST=1 \
make test-real-agents
```

The core repository also exposes a manual GitHub Actions workflow named
`Live Agent Integrations`. It runs the same `make test-real-agents` target.
Dispatch it with `hermes_url` and/or `openclaw_url`. If the runtimes require
tokens, configure repository secrets named `HERMES_API_SERVER_KEY` and
`OPENCLAW_GATEWAY_TOKEN`. The workflow runs health checks by default. Enable
`hermes_turn_test` or `openclaw_turn_test` only when you want it to create real
agent work. Enable `hermes_cancel_test` or `openclaw_cancel_test` only when you
want the workflow to certify cooperative cancellation against a live runtime.

## Sources

- Hermes Agent API Server: https://hermes-agent.nousresearch.com/docs/user-guide/features/api-server/
- Hermes Agent repository: https://github.com/NousResearch/hermes-agent
- OpenClaw Gateway protocol: https://docs.openclaw.ai/gateway/protocol
- OpenClaw Gateway repository docs: https://github.com/openclaw/openclaw/blob/main/docs/gateway/protocol.md
