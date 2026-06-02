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
