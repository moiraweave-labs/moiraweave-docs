# Agent Operations Architecture

MoiraWeave is the control plane around AI workloads. It deploys and operates
models, pipelines, and agents, but it does not replace the internals of Hermes,
OpenClaw, LangGraph, or a custom runtime.

## C4 Context

```mermaid
flowchart TD
  %% Custom styles
  classDef default fill:#0e1322,stroke:#1e293b,stroke-width:1.5px,color:#f8fafc,rx:8px;
  classDef primary fill:#10b981,stroke:#34d399,stroke-width:2px,color:#ffffff,rx:10px;
  classDef external fill:#0b1220,stroke:#334155,stroke-width:1px,color:#94a3b8,stroke-dasharray: 4 4,rx:8px;

  User["👤 Operator / Builder<br/><span style='font-size:11px;color:#94a3b8'>Deploys workloads & interacts with agents</span>"]:::primary
  
  Moira["⚙️ MoiraWeave (System)<br/><span style='font-size:11px;color:#94a3b8'>Self-hosted workload & agent operations platform</span>"]:::default

  subgraph External ["External Integration Ecosystem"]
    Agent["🤖 Agent Runtime<br/><span style='font-size:11px;color:#94a3b8'>Hermes, OpenClaw, LangGraph, or custom agent</span>"]:::external
    Model["🧠 Model Service<br/><span style='font-size:11px;color:#94a3b8'>KServe-compatible or custom inference endpoint</span>"]:::external
    Channel["💬 External Channels<br/><span style='font-size:11px;color:#94a3b8'>Telegram, Slack, Discord, and Webhooks</span>"]:::external
  end

  User -->|Uses UI, CLI, and API| Moira
  Moira -->|Deploys, messages, cancels, collects artifacts| Agent
  Moira -->|Submits sync or async runs| Model
  Channel -->|Runtime-owned channel| Agent
  Channel -. Optional connector .-> Moira

  style External fill:#0b1220,stroke:#334155,stroke-width:1.5px,color:#f8fafc;
```

## Containers

```mermaid
flowchart LR
  %% Custom class definitions
  classDef default fill:#0e1322,stroke:#1e293b,stroke-width:1.5px,color:#f8fafc,rx:8px;
  classDef primary fill:#10b981,stroke:#34d399,stroke-width:2px,color:#ffffff,rx:10px;
  classDef external fill:#0b1220,stroke:#334155,stroke-width:1px,color:#94a3b8,stroke-dasharray: 4 4,rx:8px;
  classDef database fill:#111827,stroke:#6366f1,stroke-width:1.5px,color:#f8fafc,rx:4px;
  classDef queue fill:#111827,stroke:#38bdf8,stroke-width:1.5px,color:#f8fafc,rx:4px;

  UI[Ops Dashboard]:::primary --> API[API Gateway]
  CLI[moira CLI]:::default --> API
  CH[Optional Channel Connectors]:::default --> API
  API --> PG[(Postgres)]:::database
  API --> REDIS[(Redis Streams)]:::queue
  Worker[Worker Fleet]:::default --> REDIS
  Worker --> PG
  Worker --> W1[Agent Workload]:::external
  Worker --> W2[Model Workload]:::external
  Worker --> W3[Pipeline Workload]:::external
  W1 --> FS[(Artifact / Workspace Volume)]:::database
  W2 --> FS
  W3 --> FS

  class API,Worker default;
```

## Agent Message Sequence

```mermaid
%%{init: { 'theme': 'dark', 'themeVariables': { 'actorBkg': '#0e1322', 'actorBorder': '#1e293b', 'actorTextColor': '#f8fafc', 'signalColor': '#38bdf8', 'signalTextColor': '#94a3b8', 'labelBoxBorderColor': '#1e293b', 'labelBoxBkgColor': '#0b0f19', 'labelTextColor': '#f8fafc' } } }%%
sequenceDiagram
  participant U as User
  participant UI as UI or Channel
  participant API as API Gateway
  participant DB as Postgres
  participant R as Redis Stream
  participant W as Worker
  participant A as Agent Runtime

  U->>UI: Send message
  UI->>API: POST /v1/agents/{name}/sessions/{id}/messages
  API->>DB: Store user message, queued run, and audit event
  API->>R: Dispatch run message
  W->>R: Consume run
  W->>DB: Mark starting/running, emit events
  W->>A: Adapter send_message
  A-->>W: Ack or response
  W->>DB: Store assistant message, artifacts, final state
  UI->>API: Stream events and refresh history
```

## Deployment And Health Sequence

```mermaid
sequenceDiagram
  participant O as Operator
  participant CLI as moira CLI
  participant API as API Gateway
  participant DB as Postgres
  participant UI as Ops Dashboard

  O->>CLI: moira deploy local or k8s --env prod
  CLI->>O: Generate/apply runtime manifests
  CLI->>API: POST /v1/workloads/{name}/deployments with env
  API->>DB: Upsert deployment record and audit event
  UI->>API: GET /v1/workloads/{name}/health?env=prod
  API->>DB: Read env-scoped deployment/run/session state
  API-->>UI: healthy, pending, degraded, or unknown
```

## Optional Channel Connector Sequence

Use this only when MoiraWeave owns the channel connector. If the runtime owns
Telegram, Slack, Discord, or a webhook directly, declare it in
`externalOwnedChannels` and let users interact through that runtime-owned
channel.

```mermaid
sequenceDiagram
  participant C as Telegram / Slack / Webhook
  participant GW as Channel Connector
  participant API as API Gateway
  participant DB as Postgres
  participant R as Redis Stream
  participant W as Worker
  participant A as Agent Runtime

  C->>GW: External message
  GW->>API: POST /v1/channels/{channel}/agents/{name}/messages
  API->>DB: Create/reuse session, store message, audit channel metadata
  API->>R: Queue run
  W->>A: Adapter send_message
  W->>DB: Store events, response, artifacts, final run state
```

## Operational Boundary

MoiraWeave owns:

- environment-scoped workload deployment metadata
- deployment health derived from control-plane state
- sessions, messages, runs, events, artifacts, and health
- audit records for deployment operations, cancellation, artifact access, UI/API
  agent messages, and MoiraWeave-owned connector ingress
- cancellation and stale-run detection
- UI/API/CLI surfaces
- environment-specific deployment assets and deployment operation history

Agent runtimes own:

- reasoning loop and planning
- tool execution implementation
- memory internals
- runtime-specific configuration
- provider-specific model calls

## Adapter Contract

Every agent adapter exposes the same operational shape:

- `send_message(payload)`: short-lived dispatch call
- `wait_for_completion(payload, accepted)`: follow runtime status/events until terminal state or timeout
- `get_status(payload)`: check runtime or session health
- `cancel(payload)`: cooperative cancellation hook
- `list_artifacts(payload)`: discover runtime-produced artifacts

Long-running agents should acknowledge a message quickly and continue work in
their own process. MoiraWeave follows progress through stored events, health
checks, and adapter status calls instead of holding a request open for hours.

Runtime-specific details live in
[Agent Runtime Integrations](agent-runtime-integrations.md).

## Current API Surface

- `POST /v1/workloads/{name}/deployments`: record local, Kubernetes, or external deployment state for an environment.
- `GET /v1/deployments`: list deployment records visible to the authenticated user, optionally filtered by workload and environment.
- `GET /v1/workloads/{name}/health`: summarize health from environment-scoped deployment state and probe deployment endpoints when present.
- `POST /v1/channels/{channel}/agents/{name}/messages`: authenticated inbound channel bridge.
- `GET /v1/agents/{name}/sessions/{session_id}/health`: summarize a session and its latest run.
- `GET /v1/audit-events`: list the authenticated subject's audit events, with filters for action and resource.
