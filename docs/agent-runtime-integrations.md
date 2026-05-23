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
  execution:
    mode: session
    timeoutSeconds: 172800
  ports:
    - name: http
      port: 8642
  env:
    API_SERVER_ENABLED: "true"
    API_SERVER_HOST: "0.0.0.0"
    API_SERVER_PORT: "8642"
    API_SERVER_KEY: "${HERMES_API_SERVER_KEY}"
  secrets:
    - OPENAI_API_KEY
    - HERMES_API_SERVER_KEY
  persistence:
    enabled: true
    mountPath: /data
  agent:
    adapter: hermes
    authTokenEnv: HERMES_API_SERVER_KEY
    model: hermes-agent
    workspaceMount: /workspace
    exposedChannels: [ui, api, telegram]
    dispatchTimeoutSeconds: 10
    pollIntervalSeconds: 2
```

For a single local Hermes runtime, `authTokenEnv: API_SERVER_KEY` also works if
the same `.env` value is shared with the worker. For multiple Hermes profiles,
prefer one token env var per workload and map it to `API_SERVER_KEY` inside the
runtime environment as shown above.

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
  execution:
    mode: session
    timeoutSeconds: 172800
  ports:
    - name: gateway
      port: 18789
  env:
    OPENCLAW_GATEWAY_TOKEN: "${OPENCLAW_GATEWAY_TOKEN}"
  secrets:
    - OPENCLAW_GATEWAY_TOKEN
  persistence:
    enabled: true
    mountPath: /data
  agent:
    adapter: openclaw
    authTokenEnv: OPENCLAW_GATEWAY_TOKEN
    agentId: main
    workspaceMount: /workspace
    exposedChannels: [ui, api]
    dispatchTimeoutSeconds: 10
    pollIntervalSeconds: 2
```

MoiraWeave session ids become OpenClaw session keys in the form
`agent:{agentId}:{session_id}` unless the payload already provides a
runtime-native `session_key`.

## Long-Running Behavior

Agent turns can run for hours or days. The production pattern is:

1. API stores the user message and creates a MoiraWeave run.
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

## Sources

- Hermes Agent API Server: https://hermes-agent.nousresearch.com/docs/user-guide/features/api-server/
- Hermes Agent repository: https://github.com/NousResearch/hermes-agent
- OpenClaw Gateway protocol: https://docs.openclaw.ai/gateway/protocol
- OpenClaw Gateway repository docs: https://github.com/openclaw/openclaw/blob/main/docs/gateway/protocol.md
