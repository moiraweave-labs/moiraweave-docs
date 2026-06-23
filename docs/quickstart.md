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
`moira doctor`, starts the platform and workloads, waits for API and UI
readiness, and registers workload/deployment records.

Local records are stored under the `local` environment. When you later use
Kubernetes or an external runtime, keep its records in `dev`, `staging`, `prod`,
or another explicit environment so health and preflight stay easy to reason
about.

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

Local semantic search is disabled by default so first-run startup does not
download embedding models from external registries. To enable it, set
`EMBEDDING_MODEL=BAAI/bge-small-en-v1.5` or another FastEmbed model in `.env`;
the generated Compose file points Hugging Face/FastEmbed caches at writable
temporary paths inside the API container.

Open `http://localhost:3000/agents`, then sign in with:

```text
admin / demo-password
```

The local demo user has the `admin` role by default. Admins can create
persistent users, teams, and team memberships from Security or from the CLI:

```bash
moira security user create alice --role operator
moira security team create agents "Agent Operators"
moira security team add-member agents alice --role operator
```

For staging or production, turn demo auth off and bootstrap the first admin
once:

```bash
export DEMO_AUTH_ENABLED=false
moira security bootstrap-admin owner --password '<strong-password>'
```

After a persistent admin exists, the bootstrap endpoint refuses new admins.
Use Security or `moira security user password-reset`, `moira security user
enable`, and `moira security user update` for normal user administration.

Persistent users sign in through the same `/auth/token` endpoint. For
automation, open Security in the dashboard or use
`moira security api-key create` to create a hashed API key, optionally scoped to
a team:

```bash
moira security api-key create "agent automation" alice --role operator --team-id agents
```

Copy the returned `mwk_...` secret
immediately; MoiraWeave stores only the hash, prefix, subject, role, team scope,
timestamps, and revocation state in Postgres.
Admins can rotate keys from Security or `moira security api-key rotate <key_id>`
when automation credentials need replacement; the old key is revoked and the new
secret is also shown once.
Static bootstrap keys are still supported through `MOIRA_API_KEYS` with
comma-separated `key:subject:role` entries. The dashboard resolves the active
credential with `GET /auth/me`, shows the current role in the header, and
disables actions that need a higher role before they fail server-side.

Check the active credential and environment summary from the terminal:

```bash
moira security me
moira env list
```

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

Both the CLI and dashboard expose a deployment readiness guide. In the CLI it is
printed after the doctor checks and returned as `action_guide` in JSON output.
In Operations Center it appears next to the operational snapshot. For real
agents, this guide turns missing secrets, Docker/Compose issues, worker
dispatch, runtime reachability, and deployment record gaps into concrete next
commands without exposing secret values.

If you run development builds or private registries, override platform images in
`.env` with `MOIRAWEAVE_API_GATEWAY_IMAGE`, `MOIRAWEAVE_WORKER_IMAGE`, and
`MOIRAWEAVE_UI_IMAGE`. `moira doctor` checks whether required images are locally
available or pullable before Docker starts.

Official images are built and pushed by GitHub Actions. GHCR package visibility
is a separate setting: the packages must be public for anonymous `docker pull`
and first-run `moira up` to work without a registry login.

## 3. Use The Product Flow

In the dashboard:

- Workloads: create workloads from templates, then jump directly to preflight
  or the agent console. Agent templates show runtime-owned capabilities such as
  web search, browser automation, terminal access, MCP, messaging, and external
  channels before the workload is created.
- Operations: run preflight, view recommended actions, inspect deployment
  records, view deployment plans, sync deployment records, and review
  deployment operation history. Select the environment first: `local` for
  `moira up`, or `dev`/`staging`/`prod` for cluster and external runtimes.
  Preflight separates Postgres, Redis, worker dispatch, environment-scoped
  deployment records, secrets, and runtime reachability so queued agent turns
  have an actionable cause. The readiness guide summarizes those checks as
  next commands such as setting missing secret names, syncing deployment records,
  checking worker logs, or inspecting runtime logs. The Command Companion next
  to deployment metadata shows the exact local, Kubernetes, or external runtime
  commands for the selected workload and environment.
- Agents: the first agent and existing session are selected automatically; start
  the first session from the empty-state CTA, send a message, cancel/retry a
  turn, watch session health, and follow the exact linked run status even when
  prompts repeat.
- Runs: inspect event timelines, payloads, results, cancellation, and artifacts.
- Artifacts: browse by workload, session, run, and content type; inspect
  metadata, preview/download local or PVC-backed files, follow run links, jump
  back to the exact agent session when available, and copy artifact references.
- Audit: query sensitive actions such as deployment syncs, run cancellation,
  agent/channel messages, and artifact access through `/v1/audit-events`.

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
  --auth-token-env HERMES_API_SERVER_KEY \
  --persistence \
  --mount-path /workspace \
  --workspace-mount /workspace
```

MoiraWeave deploys and supervises the runtime, but Hermes/OpenClaw keep their
own reasoning loop, tools, memory, and runtime-specific configuration. If those
runtimes use web search, browser automation, terminal backends, MCP servers, or
native channels such as Telegram, declare those needs in the workload manifest;
MoiraWeave prepares the deployment boundary and observes health, but it does not
proxy or reimplement the tools.

Before starting the stack, inspect required names without exposing values:

```bash
moira secrets list --workload hermes
```

For Kubernetes deployments, verify Secret keys from the operator machine or CI
runner that has `kubectl` access. Values remain in the cluster Secret or
external secret manager:

```bash
moira secrets list --workload hermes --target kubernetes --env dev --kubernetes-secret moiraweave-secrets
```

## CLI Boundaries

The UI never talks directly to Docker, Kubernetes, Redis, or the filesystem.
Local execution runs through `moira up`, `moira deploy local`, and Docker
Compose. Kubernetes execution should run through CLI, CI, or a deployment
controller/operator. Operations Center can request Apply, Logs, and Undeploy
operations. Local/external operations return commands and next actions for the
CLI or controller to execute. Kubernetes Apply/Undeploy requests are queued for
a deployment controller when one is installed; the controller claims the
operation, appends execution events, completes it, and syncs the deployment
record. The browser does not receive deployment credentials. The Controller
Queue in Operations Center shows queued/running controller work and the exact
`moira deploy controller run` command for the selected target and environment.

Run the CLI controller from an operator machine, CI runner, or secured
automation environment that already has `MOIRA_TOKEN`, `helm`, and `kubectl`
configured:

```bash
moira deploy controller run --env dev --watch
```

For long-lived Kubernetes environments, enable the in-cluster controller in the
MoiraWeave Helm chart. Create a token secret first; use an admin API key or
admin bearer token because the controller claims queued operations across users:

```bash
kubectl create secret generic moiraweave-controller-token \
  --from-literal=MOIRA_TOKEN=<admin-api-token> \
  --namespace moiraweave

helm upgrade --install moiraweave oci://ghcr.io/moiraweave-labs/charts/moiraweave \
  --namespace moiraweave --create-namespace \
  --set deploymentController.enabled=true
```

The controller image is `ghcr.io/moiraweave-labs/moiraweave-cli:latest`. It
includes `moira`, Helm, and kubectl. The chart creates a separate ServiceAccount,
Role, RoleBinding, Deployment, and NetworkPolicy for the controller, so the UI
still never receives kubeconfig or cluster credentials.

The first executable controller supports Kubernetes `apply` through Helm,
workload log collection through `kubectl logs`, and workload runtime deletion
through Kubernetes labels. It intentionally runs outside the browser so cluster
credentials stay with the operator process or in-cluster ServiceAccount.
While running, it claims operations, sends heartbeats, refreshes a lease, and
stores stdout/stderr summaries when completing the operation. Heartbeats keep
refreshing while Helm or kubectl commands are still running, so long operations
remain visibly owned by the controller. If the lease expires, Operations Center
shows a critical alert and another healthy controller can reclaim the operation.

### Optional kind Controller Smoke

The core repository includes an opt-in smoke test for the Kubernetes controller.
It is not part of normal CI because it needs a live API, a kube context, Helm,
kubectl, and permission to create/delete workload resources in the selected
namespace. The test creates a Demo Agent from the template API, queues
controller-backed Apply and Logs operations, claims and executes them through
the CLI controller, then simulates an abandoned Undeploy lease and verifies that
a healthy controller reclaims it.

Run it from the `moiraweave` repository after the platform API is reachable:

```bash
MOIRA_TOKEN=<admin-or-operator-token> \
E2E_BASE_URL=http://localhost:8000 \
make test-kind-controller
```

Useful overrides:

```bash
MOIRAWEAVE_CLI_BIN=/path/to/moira \
MOIRAWEAVE_KIND_NAMESPACE=moiraweave \
MOIRAWEAVE_KIND_RELEASE=moiraweave \
MOIRAWEAVE_KIND_CHART_REF=infra/helm/moiraweave \
MOIRAWEAVE_KIND_WORKLOAD=kind-demo-agent \
make test-kind-controller
```

The smoke intentionally validates the product boundary: the UI/API create and
observe operations, while the CLI/controller process owns Kubernetes credentials
and performs Helm/kubectl work.

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `moira` command not found | CLI is not installed in the active shell | Re-run `uv tool install moiraweave-cli` |
| `moira up` stops before Docker starts | `moira doctor` found a blocking local issue | Fix the ERROR rows from `moira doctor`, then rerun `moira up` |
| `moira up` cannot start containers | Docker is stopped or the port is busy | Run `moira doctor`, start Docker, and check ports 8000/3000/5432/6379 |
| `moira doctor` reports official images unavailable | GHCR images were pushed but package visibility is not public | In GitHub Packages, set `moiraweave/api-gateway`, `moiraweave/worker`, and `moiraweave-ui` to public, then rerun `moira doctor` |
| Helm cannot pull the MoiraWeave chart | GHCR chart package is private or not published yet | Set `charts/moiraweave` package visibility to public, or use `deploymentController.chartRef` with a private registry login |
| Deployment controller pod cannot claim operations | Missing or low-privilege `MOIRA_TOKEN` | Create `moiraweave-controller-token` with an admin token or API key |
| `moira doctor` warns about transient registry failures | Registry/network timeout, 429, or temporary GHCR issue | Retry `moira up`; Docker may still pull the images. If it persists, login to the registry or override `MOIRAWEAVE_*_IMAGE` |
| `moira doctor` reports custom images unavailable | The image is private, unpublished, or the registry login is missing | Publish/login to the registry or override `MOIRAWEAVE_*_IMAGE` in `.env` |
| `moira up` reports missing environment variables | Required workload secrets are not available locally | Run `moira doctor` and follow the readiness guide, or run `moira secrets list`, then add missing names to `.env` or export them |
| Login fails | Local demo auth is disabled or the password was overridden | Check `DEMO_AUTH_ENABLED`, `DEMO_USERNAME`, and `DEMO_PASSWORD` in `.env`; use a persistent user in staging/prod |
| API request returns `403` | The token role is too limited | Use an `operator` or `admin` token for mutating actions |
| Runs never start and dead-letter count grows | Worker cannot parse or resolve dispatch messages | Run `moira run dead-letter list`; inspect the reason, fix the root cause, then use `moira run dead-letter replay <message-id>` or purge with `moira run dead-letter purge <message-id>` |
| Workload is created but not healthy | Runtime service is missing or not reachable | Use Operations preflight and workload logs |
| Production looks healthy because local is running | Operations is filtered to the wrong environment | Switch the environment selector to `prod` and rerun preflight |
| A teammate changed/canceled/accessed something | Audit trail is needed | Query `/v1/audit-events` with `action`, `resource_type`, `resource_id`, or `env` filters |
| `/ready` shows `run_queue: degraded` | Worker consumer group is missing or no worker is attached | Start/restart the worker and check Redis connectivity |
| Agent message stays queued | Worker is stopped, Redis is unavailable, or no worker consumer is attached | Run Operations preflight and check `worker_dispatch`, `/ready`, worker logs, and Redis connectivity |
| Agent run fails after dispatch timeout | Runtime did not acknowledge quickly | Configure adapter paths or return an early ack before long work |
