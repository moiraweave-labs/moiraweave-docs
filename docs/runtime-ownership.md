# Runtime Ownership

MoiraWeave is the control and deployment plane for AI workloads. It should make
agents easy to deploy, observe, operate, and connect to channels, but it should
not replace the runtime-specific internals of Hermes, OpenClaw, LangGraph, or a
custom agent server.

## MoiraWeave Owns

- Workload manifests, deployment records, preflight checks, and health state.
- Runs, sessions, user messages, assistant responses, events, and artifacts.
- Local Compose and Kubernetes deployment assets.
- UI/API/CLI interaction paths.
- Cancellation requests, heartbeat, stale-run detection, Redis pending-message
  cleanup, and operational status.
- Secret references, channel routing metadata, and audit records.

## The Agent Runtime Owns

- Reasoning loop, planning strategy, tool execution, and memory internals.
- Runtime-specific configuration files and model/provider settings.
- Long-running task implementation details.
- Native channels that are not routed through MoiraWeave, such as a Telegram bot
  embedded directly in the runtime.
- Runtime-specific artifacts that are not exposed through the adapter contract.

## Integration Contract

MoiraWeave talks to agents through adapters. A good adapter exposes:

- `send_message`: accepts a short dispatch request and returns a run/turn handle.
- `get_status`: reports whether the runtime and active turn are healthy.
- `cancel`: requests cooperative cancellation.
- `list_artifacts`: returns runtime artifacts that MoiraWeave can index.

For agents that run for hours or days, the adapter should acknowledge quickly and
let MoiraWeave poll or stream events. Keeping a single HTTP request open for the
whole reasoning loop is not the intended operating model.

## Runtime Requirements And Boundaries

MoiraWeave does not register or orchestrate every runtime tool. It prepares the
environment the runtime needs and records the boundary in `workload.yaml`:

```yaml
spec:
  agent:
    adapter: hermes
    toolOwnership: runtime
    workspaceMount: /workspace
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
```

This means Hermes, OpenClaw, or a custom runtime decides when and how to call
web search, browser, terminal, MCP, messaging, memory, or provider tools.
MoiraWeave only checks that the declared environment is plausible: secrets are
present, network and workspace boundaries make sense, and deployment health is
visible. If a runtime needs direct access to a user's machine or a channel it
fully owns, run it as an external-owned runtime and let MoiraWeave supervise it
through the adapter.

Managed deployment isolation should not be interpreted as disabling the agent's
tools. A managed Hermes/OpenClaw container can still use web search, browser
automation, terminal backends, MCP servers, provider APIs, or runtime-native
channels when those capabilities are configured inside the runtime and the
workload declares the required egress, workspace, and secrets. MoiraWeave does
not proxy or approve each internal tool call; it gives operators a clear place
to see what boundary was requested and whether the runtime is healthy.

## Long-Running Recovery

Long-running runs are tracked in Postgres, not Redis. The worker writes
`heartbeat_at` while a run is active. If heartbeats stop, stale-run detection
marks active runs as `lost` and appends a `run.lost` event.

Redis Streams remain the dispatch queue. A healthy long-running run can keep its
stream message pending until the run finishes, so MoiraWeave does not reclaim
pending messages by idle time alone. Pending recovery first checks the durable
run state:

- `queued` and `cancel_requested` messages can be reclaimed by another worker.
- Terminal messages are acknowledged and removed from the pending list.
- `starting`, `running`, and `cancelling` messages are left alone while their
  run is active, so an agent turn is not duplicated.

The API `/ready` response includes `run_queue` metadata with the Redis stream,
consumer group, attached consumers, pending count, and lag when Redis exposes
it. Operators should treat `run_queue: degraded` as a worker/dispatch issue,
not as an agent-runtime failure.

## Channels

The UI and API are the canonical MoiraWeave-owned channels. Telegram, Slack,
Discord, and webhooks should normally enter through connector services that call
the MoiraWeave session API.

If an agent runtime already owns a channel, configure it as an external-owned
channel. MoiraWeave will supervise deployment, health, runs, events, and
artifacts where the runtime exposes them, but it will not try to mirror every
message unless the runtime provides that history.
