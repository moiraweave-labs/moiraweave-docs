# Security Hardening

This guide turns a local MoiraWeave setup into a safer self-hosted deployment.
MoiraWeave remains a control plane: it stores metadata, state, and audit events,
but it does not store secret values or manage internal agent tools.

## Authentication

Disable demo auth outside local development:

```bash
DEMO_AUTH_ENABLED=false
```

When demo auth is disabled and no persistent admin exists, bootstrap exactly one
admin:

```bash
moira security bootstrap-admin owner --password '<strong-password>'
```

The API endpoint is `POST /auth/bootstrap/admin`. It is only available while no
active persistent admin exists. After that, administer users through Security or
the CLI:

```bash
moira security user create alice --role operator
moira security user update alice --role viewer
moira security user password-reset alice
moira security user disable alice
moira security user enable alice
```

Successful and failed password logins are audited as `auth.login.succeeded` and
`auth.login.failed`. Audit metadata includes credential type and client host when
available, but never passwords or tokens.

## API Keys And Teams

Use persistent API keys for automation instead of sharing user passwords:

```bash
moira security team create agents "Agent Operators"
moira security team add-member agents deploy-bot --role operator
moira security api-key create "agent deploy" deploy-bot --role operator --team-id agents
```

The `mwk_...` secret is shown once. MoiraWeave stores only the hash, prefix,
subject, role, optional team scope, timestamps, and revocation state. Admins can
see all resources; non-admin users and team-scoped keys see resources owned by
their own subject plus members of their teams. Rotate and revoke keys
explicitly:

```bash
moira security api-key rotate <key_id>
moira security api-key revoke <key_id>
```

## Workload Ownership

Workloads are shared platform definitions by default, so operators can use a
centrally registered runtime without duplicating its manifest. To make a
workload private to a team, select the team in the dashboard's **Create
Workload** form or add this annotation in the advanced manifest:

```yaml
metadata:
  name: research-agent
  annotations:
    moiraweave.io/team-id: agents
```

The named team must already exist. Only its members and admins can list, inspect,
preflight, deploy, create sessions for, or run that workload. The team ownership
is stored with the manifest and is preserved when an admin updates the manifest
without specifying a new team. The registered creator remains audit metadata;
it does not make an unscoped workload private.

## Secrets

Workload manifests should declare required secret names, not values. Keep values
in one of these places:

- local `.env` for development
- Kubernetes Secrets for clusters
- an external secret manager that injects environment variables

Use the inventory to verify presence without revealing values:

```bash
moira secrets list
```

Secret inventory reads are audited as `secret_inventory.read`. The audit event
stores required secret names, total count, and missing count only.

## Signed Webhooks

MoiraWeave-owned API/UI channels use bearer auth. External webhook channels use
HMAC signatures instead:

```text
POST /v1/webhooks/{channel}/agents/{name}/messages
X-MoiraWeave-Signature: sha256=<hmac>
```

The HMAC is SHA-256 over the raw request body using `WEBHOOK_SIGNING_SECRET`.
Only expose channels listed in `spec.agent.exposedChannels`. If the runtime owns
Telegram, Slack, Discord, or another channel directly, declare it under
`externalOwnedChannels` and supervise it from MoiraWeave instead of duplicating
traffic.

For team-scoped workloads, put `team_id` in the signed JSON body. MoiraWeave
uses that team as the webhook subject scope and stores it in channel/session
metadata and audit context. Do not send team scope in an unsigned header or URL
parameter. Bearer-authenticated channel requests may also include `team_id`,
but the requested team must be visible to the user or API key.

## Operations

Use Operations Center or CLI alerts before touching Redis/Postgres directly:

```bash
moira ops alerts --env prod --scope all
moira run dead-letter list
moira run dead-letter replay <message_id>
```

Dead-letter replay validates the run and workload, records an audit event,
re-enqueues the worker message, and removes the original dead-letter entry.
Operations alerts also surface pending Redis Stream dispatch messages so
operators can distinguish a worker/reclaim issue from an agent-runtime issue.

## Schema Migrations

Alembic owns the Postgres control-plane schema. The runtime verifies the expected
Alembic revision during startup and should not run inline DDL in production.

Release validation can run destructive migration checks against a disposable
Postgres database:

```bash
MOIRAWEAVE_POSTGRES_MIGRATION_DSN=postgresql://user:pass@host/db \
MOIRAWEAVE_POSTGRES_MIGRATION_DSN_IS_DISPOSABLE=1 \
uv run pytest tests/integration/test_control_plane_postgres_migrations.py --no-cov
```

The test drops and recreates the `public` schema. Never point it at shared,
staging, or production databases.

## Kubernetes

Keep the UI away from kubeconfig and Docker credentials. For Kubernetes apply,
logs, and undeploy operations, use one of these executor paths:

- CLI from an operator workstation
- optional deployment controller with a scoped token

Production Helm values should set:

- `demoAuth.enabled=false`
- controller token Secret when the deployment controller is enabled; Helm fails
  template rendering if `deploymentController.auth.existingSecret` or
  `deploymentController.auth.tokenKey` is empty
- NetworkPolicy for API, worker, Redis, Postgres, Qdrant, UI, and workloads
- resource requests and limits
- probes for API, worker, UI, and workloads
- persistent artifact storage with backups

Deployment controller operations store `controller_id`, `heartbeat_at`, and
`lease_expires_at` in Postgres. The CLI controller renews heartbeats while Helm
or kubectl commands are running. Treat expired leases as operational incidents:
restart the controller, check its logs, then let a healthy controller reclaim the
operation rather than editing Redis/Postgres manually.

## Backups

Back up:

- Postgres control-plane database
- artifact backend or PVC
- Kubernetes Secrets or external secret-manager state
- workload manifests and generated deployment values

Redis is queue/coordination state only and should not be treated as durable
control-plane storage.
