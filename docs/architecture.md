# Architecture

MoiraWeave is designed around a narrow runtime and explicit ownership
boundaries. It operates AI workloads without embedding customer business logic
or agent internals.

## Layers

| Layer | Responsibility | Owned by |
| --- | --- | --- |
| Workspace layer | Workload manifests, deployment values, artifacts, and secrets | The team using MoiraWeave |
| Runtime layer | API gateway, worker, control-plane storage, queueing, deployment templates, and observability | `moiraweave` |
| Experience layer | CLI and integrated Ops dashboard | `moiraweave-cli`, `moiraweave-ui` |

## Runtime Services

- API gateway: auth, workload templates, workload registration, preflight,
  deployment operations, run submission, sessions, messages, events, artifacts,
  audit events, secret inventory, and health.
- Worker: consumes Redis dispatch messages and calls model, pipeline, or agent executors.
- Postgres: source of truth for workloads, runs, sessions, messages, events,
  artifact metadata, and audit events.
- Redis Streams: queue and short-lived coordination layer.
- Qdrant: optional vector store for RAG/search workloads.
- UI: browser console for workloads, runs, agent sessions, artifact metadata
  inspection, local/PVC artifact preview and download, environment-scoped
  deployment health, deployment operation history, and audit trail inspection.

## End-to-End Run Flow

1. A user submits a workload run through CLI, UI, or API.
2. The API stores the run in Postgres and dispatches a message to Redis Streams.
3. The worker consumes the message and marks the run `starting` then `running`.
4. The executor calls the workload according to its type.
5. The worker stores events, assistant messages, artifacts, result, and final state.
6. UI and CLI read from the API only.

For long-running agents, Postgres is the durable source of truth. Workers update
`heartbeat_at`, stale active runs become `lost`, and Redis pending entries are
only reclaimed when the run state says it is safe. This prevents an idle Redis
pending message from duplicating an agent turn that is still heartbeating.
Transient executor failures are retried with bounded backoff inside the workload
timeout, and duplicate dispatch messages for already-active runs are acknowledged
without re-running the agent action. Retry classification is intentionally
conservative: transport failures, timeouts, retryable HTTP statuses, and
runtime-unavailable errors can retry; invalid payloads, bad manifests, missing
workloads, and deterministic pipeline/DAG errors fail without retry and emit a
`run.retry_skipped` event. Malformed or unrecoverable dispatch messages go to
the Redis dead-letter stream; operators can inspect and purge them through the
API or `moira run dead-letter list|replay|purge` without connecting to Redis
directly.

## Event Delivery For Long Runs

Run history is durable in Postgres, but live consumers do not repeatedly read
the whole timeline. `GET /v1/runs/{run_id}/events` accepts `after_id` and
`limit` for incremental reads, or `tail=true` to open the recent portion of a
long timeline. `GET /v1/runs/{run_id}/events/stream` honors `Last-Event-ID` and
uses the same cursor internally. The dashboard first loads the recent timeline
and then resumes from its latest event, so reconnects do not replay days of
agent activity.

Agent session lists use bounded `limit`/`offset` pages. Agent message history
opens the latest bounded page and uses `before_id` to load earlier messages in
chronological order. These controls keep a long-lived conversation usable
without asking the browser or API gateway to materialize its entire history.

Artifact Library uses the same bounded `limit`/`offset` contract while retaining
its workload, environment, session, run, date, and content-type filters. The
dashboard only fetches additional artifact pages when an operator asks for them.

Runs, deployment operations, and audit events also use bounded `limit`/`offset`
pages in the dashboard. Operators can load older pages explicitly, which keeps
high-volume Operations Center views responsive without hiding the API filters
for workload, target, environment, action, or resource.

## Agent Flow

Agent workloads use an adapter. The adapter sends a short-lived dispatch call to
the agent runtime, then MoiraWeave tracks the run through stored state and
events. Hermes, OpenClaw, LangGraph, or custom agents keep their own internal
reasoning loop.

Agent runtimes can be placed in two ways. Managed runtimes are deployed by
MoiraWeave as Docker Compose services or Kubernetes Deployments in the same
network/namespace as the worker. External runtimes are not deployed by
MoiraWeave; the manifest records `spec.endpoint`, and the adapter uses that
URL.

MoiraWeave supports multiple agents by treating each runtime/profile as its own
workload. A Hermes service, an OpenClaw gateway, and a custom HTTP agent can be
deployed together if their manifests declare distinct names, service names,
ports, and secrets. Sessions, messages, runs, events, artifacts, health, and
deployment records remain scoped to the selected workload. External agents are
registered as `target: external` deployment records so health and UI state still
show where the runtime lives even when MoiraWeave does not own the process.

## Environments

Deployment state is scoped by environment. A workload can have separate
deployment records for `local`, `dev`, `staging`, and `prod`, even when the
target is the same. This keeps a healthy local Compose service from masking a
missing Kubernetes deployment, and it lets Operations Center filter preflight,
health, deployment plans, deployment operations, and records to the environment
an operator is currently working on.

`moira up` and `moira deploy local` write `local` deployment records. Kubernetes
commands and deployment operations use the selected CLI/UI environment, commonly
`dev`, `staging`, or `prod`. Audit events include the environment as non-secret
metadata when a deployment record or operation is created.
`GET /v1/environments` summarizes the environments visible to the authenticated
subject from deployment records and deployment operations, so UI and CLI can
show environment counts without reaching into Postgres directly.

## Observability

The API gateway exposes Prometheus metrics at `/metrics` on its HTTP service.
The worker exposes a Prometheus metrics port named `metrics`. On Kubernetes,
`make helm-monitoring-install` installs the monitoring chart and applies the
MoiraWeave ServiceMonitor, PodMonitor, PrometheusRule, and Grafana dashboard
ConfigMaps from `infra/k8s/monitoring/`.

The monitoring chart installs Prometheus, Alertmanager, Grafana, Loki, Promtail,
and Jaeger. Prometheus is configured to discover ServiceMonitor, PodMonitor, and
PrometheusRule resources across namespaces. Grafana loads dashboards from
ConfigMaps labelled `grafana_dashboard=1`, and Loki is registered as a Grafana
datasource so control-plane logs and metrics can be inspected together.

When Kubernetes NetworkPolicy is enabled, the platform chart explicitly allows
the `monitoring` namespace to scrape the API gateway on port `8000` and the
worker on port `9090`. Without those rules, the monitoring resources can render
successfully while Prometheus cannot reach the targets.

The monitoring stack is intentionally separate from workload placement. Managed
agent/model workloads may expose their own metrics endpoints later, but the
core control-plane metrics are deployed with the platform monitoring install.
The API `/ready` response also reports `run_queue` state, including the Redis
stream, worker consumer group, attached consumers, pending count, and lag when
available.
Operations Center mirrors the same queue signal as an actionable alert when
Redis reports pending dispatch messages that may need worker recovery or reclaim.
It also surfaces duplicate dispatch acknowledgements as informational alerts:
the worker protected an active run from duplicate execution, but sustained
growth should trigger a producer/reclaim review.

The worker publishes operational counters for long-running workload recovery:

| Metric | Labels | Meaning |
| --- | --- | --- |
| `moiraweave_worker_dead_letter_total` | `reason` | Dispatch messages moved to the dead-letter stream. |
| `moiraweave_worker_pending_reclaim_messages_total` | `outcome` | Pending Redis Stream messages inspected by reclaim. |
| `moiraweave_worker_run_retry_total` | `workload` | Retry attempts scheduled after transient executor failures. |
| `moiraweave_worker_stale_run_lost_total` | none | Runs marked `lost` after stale worker heartbeat. |

Useful checks after a cluster install:

```bash
make helm-monitoring-install
kubectl -n monitoring get servicemonitor,podmonitor,prometheusrule
kubectl -n monitoring get configmap -l grafana_dashboard=1
kubectl -n monitoring port-forward svc/moiraweave-monitoring-grafana 3000:80
```

In Grafana, open the system overview dashboard and confirm that API request
rate, API error rate, worker jobs processed, and namespace pod health show data.
If those panels are empty, check Prometheus targets first, then the MoiraWeave
ServiceMonitor/PodMonitor selectors, then NetworkPolicy.

## UI And CLI Boundary

The Ops dashboard covers API-level operations: guided workload creation,
advanced manifest registration, run submission, run cancellation, live events,
artifact browsing, local/PVC artifact preview and download, metadata
inspection, cross-links from artifacts back to runs and agent sessions, agent
session health, agent messages, channel ownership, preflight, deployment
planning, secret inventory, deployment record sync, and health.
Operations Center keeps the selected environment explicit so an operator can
compare created, deployed, reachable, and healthy state without mixing local and
cluster records. Its deployment readiness guide uses the same product language
as `moira doctor`: missing secrets, deployment records, worker dispatch, runtime
reachability, Docker/Compose, and runtime boundary checks become concrete next
commands while secret values remain outside MoiraWeave. The Command Companion
also renders the exact local, Kubernetes, or external-runtime commands for the
selected workload, target, and environment. For Kubernetes operations, the
Controller Queue highlights queued/running controller work and gives operators
the matching `moira deploy controller run --target kubernetes --env <env>
--watch` command without exposing kubeconfig or deployment credentials to the
browser.

API access uses bearer credentials. Local development can issue demo JWTs when
`DEMO_AUTH_ENABLED=true` with `DEMO_USERNAME`, `DEMO_PASSWORD`, and
`DEMO_ROLE`; staging and production should disable demo auth. When demo auth is
disabled and no persistent admin exists, `POST /auth/bootstrap/admin` or
`moira security bootstrap-admin` creates the first admin and then closes that
bootstrap path. Admins can create persistent users through `/auth/users`, teams
through `/auth/teams`, and team memberships through
`/auth/teams/{team_id}/members`; persistent users authenticate through the same
`/auth/token` endpoint with PBKDF2-hashed passwords stored in Postgres.
Automation should use hashed API keys created by an admin through the Security
screen or `/auth/api-keys`; keys can optionally be scoped to a team.
The secret is shown once, then only metadata, team scope, hash, last-use
timestamp, and revocation state remain in Postgres. Existing keys can be rotated
with `POST /auth/api-keys/{key_id}/rotate`, which returns a new one-time secret,
keeps subject/role/team intent, and revokes the previous key. Static bootstrap
keys can still be defined as comma-separated `key:subject:role` entries in
`MOIRA_API_KEYS`. Clients resolve the current credential through `GET /auth/me`,
and the UI shows the subject, role, team scope, memberships, and API-key/JWT
credential type before enabling mutating actions. The initial role model is
intentionally small: `viewer` can inspect, `operator` can run, cancel, message
agents, preflight, and record deployment operations, and `admin` can create
workloads, inspect secret inventory, and manage users, teams, and API keys.
Admins see all control-plane resources. Non-admin users see resources owned by
their own subject plus resources owned by members of their teams; team-scoped
API keys inherit the same effective visibility through their subject and
`team_id`.

Secret inventory is deliberately metadata-only. The API returns required names,
presence, source, workload references, and remediation; it does not return
values. Local values stay in `.env` or the process environment, Kubernetes
values stay in Secrets or external secret managers, and the UI only displays
whether each required name is present from the API gateway point of view. When
operators need cluster-level validation, `moira secrets list --target
kubernetes` reads only Secret key names through `kubectl`; it does not decode or
print Secret values.

Audit events are stored in Postgres and scoped to the authenticated subject.
The current trail records login attempts, API key lifecycle changes, secret
inventory reads, deployment records, deployment operations, run cancellation,
agent messages, channel ingress messages, artifact previews, and artifact downloads. These records are
operational breadcrumbs for teams: who asked MoiraWeave to do something
sensitive, against which resource, and with which non-secret metadata. They are
available through `GET /v1/audit-events`; the UI can surface them without direct
database access.

The API can return a deployment plan for each workload and target, including
generated files, service endpoint, and the CLI/Helm commands needed to apply it.
It can also run preflight checks, surface recommended actions, and record
deployment operations. These APIs accept an environment filter so an operator can
plan, sync, diagnose, and inspect one environment at a time. Preflight now checks
manifest validity, target support, environment-scoped deployment records, secret
inventory, control-plane dependencies, and runtime reachability when a
registered endpoint exists. Deployment operations are stored as a navigable
history so operators can inspect plans, syncs, blocked applies/undeploys,
generated commands, next actions, events, timestamps, and outcomes after the
fact.

For Kubernetes, the API also exposes a controller contract for deployment
operations. A UI request can create `apply` or `undeploy` with
`executor: controller`, which stores the operation as `queued`. A deployment
controller or CI worker can list queued operations, claim one, append execution
events, and complete it as `succeeded`, `failed`, or `canceled`. Successful
`apply` and `undeploy` completions update the environment-scoped deployment
record for the original requesting user. The browser still never receives
cluster credentials.

The first executable implementation of that contract is the CLI controller:
`moira deploy controller run --env dev --watch`. It is intended for an operator
terminal, CI runner, secured automation worker, or the optional in-cluster
controller. The CLI image `ghcr.io/moiraweave-labs/moiraweave-cli:latest`
contains `moira`, Helm, and kubectl. The Helm chart can install it with
`deploymentController.enabled=true`, using a separate ServiceAccount and
namespace-scoped RBAC. The controller consumes an admin token from
`moiraweave-controller-token`, fetches workload manifests from the API when no
local workspace exists, applies the published OCI chart
`oci://ghcr.io/moiraweave-labs/charts/moiraweave`, fetches workload logs, and
deletes workload runtime resources by MoiraWeave labels while keeping kubeconfig
outside the UI. While a Helm or kubectl command is running, the CLI controller
continues refreshing the deployment-operation heartbeat so long operations do
not look abandoned merely because the command is still executing.

The CLI is still required for workspace-local actions that need filesystem,
Docker, Helm, or Kubernetes credentials: `moira init`, `moira up`, Compose/Helm
generation, `deploy local --up`, `deploy k8s --apply`, logs, and manual
undeploy-style operations. `moira doctor --json` exposes an `action_guide` so CI
or setup scripts can consume the same readiness guidance shown to operators. The
UI deliberately talks only to the API gateway and does not get direct access to
Redis, Docker, Kubernetes, or local files.

Artifact content is served only when the artifact URI resolves inside the API
gateway artifact storage root. Local Compose uses the configured
`ARTIFACTS_DIR`; Kubernetes uses the mounted PVC path. Remote runtime-owned
artifact URIs remain metadata-only unless a storage connector is added.

## Design Decisions

- Use one `workload.yaml` model for Compose, Kubernetes, API validation, and worker dispatch.
- Use stable workload service names so local and Kubernetes deployments resolve the same way.
- Keep Postgres as the durable control plane.
- Keep Redis out of durable state.
- Keep UI/API as the canonical MoiraWeave interaction surface.
- Treat Telegram, Slack, Discord, and webhooks as runtime-owned channels unless
  a project explicitly builds a MoiraWeave connector. MoiraWeave-owned
  authenticated connectors call `/v1/channels/{channel}/agents/{name}/messages`;
  external webhook connectors call
  `/v1/webhooks/{channel}/agents/{name}/messages` with
  `X-MoiraWeave-Signature: sha256=<hmac>`.

## Further Reading

- [Concepts](concepts.md)
- [Agent Operations Architecture](agent-ops-architecture.md)
- [Repository Structure](repo-structure.md)
