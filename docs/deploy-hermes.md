# Deploy A Real Hermes Agent

This guide is the practical path for running a real Hermes Agent through
MoiraWeave. It assumes MoiraWeave is the control plane: it creates the workload,
deploys or connects the runtime, stores sessions/runs/events/artifacts, and
shows health. Hermes keeps ownership of its tools, memory, model provider,
browser or terminal backends, MCP servers, and native channels.

Use the UI for workload creation, preflight, deployment operations, chat,
events, logs, artifacts, and health. Use CLI or a Kubernetes controller only for
actions that need host Docker access, kubeconfig, or secret values.

## Prerequisites

- Docker and Docker Compose for local deployments, or a Kubernetes cluster with
  Helm and kubectl for cluster deployments.
- A running MoiraWeave workspace. For local onboarding:

```bash
mkdir hermes-moiraweave
cd hermes-moiraweave
moira up
```

- A model/provider key for Hermes. The default MoiraWeave Hermes template uses
  `OPENAI_API_KEY`.
- A random `HERMES_API_SERVER_KEY` for the Hermes API server. Treat it as a
  runtime token. Do not paste it into the UI.
- Enough network egress for the Hermes runtime to reach its model provider and
  any runtime-owned tools it is configured to use.

Upstream Hermes supports CLI, gateway, messaging channels, model/provider
configuration, tools, memory, and scheduled automations. Configure those inside
Hermes or its runtime environment; MoiraWeave should not reimplement them.

## Local Path

1. Open the UI printed by `moira up`.

2. Sign in with the dev credentials printed by `moira up`. If you disabled demo
   auth and no admin exists yet, choose `First admin` on the login screen.

3. Go to `Workloads`.

4. Select the `Hermes Agent` template.

5. Keep the defaults unless you need a custom image:

```text
name: hermes
image: ghcr.io/nousresearch/hermes-agent:latest
port: 8642
adapter: hermes
workspace: /workspace
persistence: enabled
```

6. Click `Create`.

7. Go to `Operations` and run `Preflight` for `hermes`.

8. Add missing secret names locally. The UI shows names only; values stay in
   your shell or `.env`.

```bash
printf 'OPENAI_API_KEY=...\nHERMES_API_SERVER_KEY=...\n' >> .env
```

9. Apply the local runtime from the operator shell. The browser cannot and
   should not access the host Docker socket.

```bash
moira up
```

10. Return to `Operations` and check that `hermes` is `deployed`, `reachable`,
    and `healthy`.

11. Go to `Agents`, select `hermes`, create a session, and send a message.

12. Open the linked run from the message to inspect live events, logs, result,
    and artifacts.

CLI equivalent for the first message:

```bash
moira agent chat hermes "hello, confirm your runtime tools and workspace" --watch
```

## Kubernetes Path

In Kubernetes, the UI still creates workloads and deployment operations, but an
operator shell, CI job, or in-cluster MoiraWeave deployment controller executes
Helm and kubectl.

1. Create the platform and Hermes runtime Secret in the target namespace.

```bash
kubectl create namespace moiraweave

kubectl create secret generic moiraweave-secrets \
  --from-literal=JWT_SECRET_KEY=<32-char-secret> \
  --from-literal=POSTGRES_DSN=postgresql://moiraweave:<postgres-password>@moiraweave-postgresql:5432/moiraweave \
  --from-literal=POSTGRES_PASSWORD=<postgres-password> \
  --from-literal=POSTGRES_POSTGRES_PASSWORD=<postgres-admin-password> \
  --from-literal=REDIS_PASSWORD=<redis-password> \
  --from-literal=OPENAI_API_KEY=<provider-key> \
  --from-literal=HERMES_API_SERVER_KEY=<random-runtime-token> \
  --namespace moiraweave
```

2. Install or upgrade MoiraWeave with UI and the optional deployment controller.

```bash
kubectl create secret generic moiraweave-controller-token \
  --from-literal=MOIRA_TOKEN=<admin-api-token> \
  --namespace moiraweave

helm upgrade --install moiraweave oci://ghcr.io/moiraweave-labs/charts/moiraweave \
  --namespace moiraweave --create-namespace \
  --set ui.enabled=true \
  --set deploymentController.enabled=true
```

3. Open the UI through your ingress or port-forward.

```bash
kubectl port-forward svc/moiraweave-api-gateway 8000:80 --namespace moiraweave
```

4. Sign in. If production auth is enabled and no admin exists yet, use `First
   admin` on the login screen.

5. Go to `Workloads`, select `Hermes Agent`, and click `Create`.

6. Go to `Operations`, select:

```text
workload: hermes
target: kubernetes
environment: dev
```

7. Run `Preflight`. Fix any missing Secret keys, port conflicts, or controller
   alerts shown by Operations Center.

8. Click `Apply`. The operation is queued in MoiraWeave and claimed by the
   deployment controller.

9. Watch the operation events in the UI until the operation succeeds.

10. Verify `Workloads -> hermes` reports:

```text
created: yes
deployed: yes
reachable: yes
healthy: yes
```

11. Use `Agents` to create a Hermes session and send a message. The message row
    links to its run, events, cancellation control, and artifacts.

## External Hermes Runtime

Use this when Hermes already runs somewhere else, for example on a VM that owns
its own Telegram or Discord gateway.

1. In `Workloads`, choose `External Agent` or create a manifest based on
   `examples/workloads/external-hermes/workload.yaml`.

2. Set:

```yaml
spec:
  type: agent-service
  endpoint: https://agents.example.com/hermes
  deployment:
    mode: external
    targets:
      - external
  agent:
    adapter: hermes
    authTokenEnv: HERMES_API_SERVER_KEY
```

3. Store `HERMES_API_SERVER_KEY` in the runtime environment, local `.env`, or
   Kubernetes Secret as appropriate.

4. Register the external deployment from the UI or CLI, run preflight, then use
   `Agents` to chat through the MoiraWeave session API.

## Tool And Channel Ownership

Hermes can use its own web search, browser, terminal, MCP, scheduled jobs, and
messaging gateways. MoiraWeave only needs to know what boundary it is deploying
and observing.

- If Hermes owns Telegram/Discord/Slack, keep those channels runtime-owned.
  MoiraWeave supervises health and runs when the runtime exposes them, but it
  does not mirror every external conversation into the UI.
- If you want MoiraWeave to be the canonical channel, use UI/API sessions first.
  Signed webhooks are the v1 inbound connector path.
- If a Hermes tool fails, fix the Hermes configuration, provider key, network
  policy, workspace permissions, or runtime image. Do not add the tool to
  MoiraWeave unless the product explicitly needs a platform-level connector.

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Preflight says `OPENAI_API_KEY` is missing | Secret key is not present in `.env` or Kubernetes Secret | Add the key outside the UI and rerun preflight |
| Preflight says runtime is unreachable | Container is not running, wrong port, bad service name, or health endpoint not ready | Check Operations events, `docker compose logs hermes`, or `kubectl logs deploy/hermes` |
| Chat creates a run but no answer arrives | Hermes API server is not ready or auth token does not match | Check `HERMES_API_SERVER_KEY`, `/health`, adapter events, and runtime logs |
| Hermes tools cannot browse or execute commands | Runtime sandbox, network policy, or Hermes tool config blocks them | Fix Hermes runtime configuration or Kubernetes network/filesystem policy |
| Telegram messages do not appear in MoiraWeave UI | Telegram is runtime-owned by Hermes | Use Hermes channel logs, or route inbound messages through MoiraWeave signed webhooks if UI/API must be canonical |
| Kubernetes Apply stays queued | No controller is running or controller token is invalid | Start `moira deploy controller run --env dev --watch` or fix `moiraweave-controller-token` |

## Success Checklist

- `hermes` exists in `Workloads`.
- Preflight has no blockers.
- Deployment record exists for the selected target and environment.
- Runtime is reachable on port `8642`.
- Agent session can send a message from the UI.
- The message links to a run with events.
- Artifacts appear when Hermes returns or writes them.
- Secret values never appear in MoiraWeave UI, API responses, logs, or docs.

## References

- Hermes Agent repository: <https://github.com/NousResearch/hermes-agent>
- Hermes Agent documentation: <https://hermes-agent.nousresearch.com>
- MoiraWeave runtime ownership: `runtime-ownership.md`
- MoiraWeave agent integrations: `agent-runtime-integrations.md`
