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

## API Keys And Teams

Use persistent API keys for automation instead of sharing user passwords:

```bash
moira security team create agents "Agent Operators"
moira security team add-member agents deploy-bot --role operator
moira security api-key create "agent deploy" deploy-bot --role operator --team-id agents
```

The `mwk_...` secret is shown once. MoiraWeave stores only the hash, prefix,
subject, role, optional team scope, timestamps, and revocation state. Rotate and
revoke keys explicitly:

```bash
moira security api-key rotate <key_id>
moira security api-key revoke <key_id>
```

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

## Operations

Use Operations Center or CLI alerts before touching Redis/Postgres directly:

```bash
moira ops alerts --env prod --scope all
moira run dead-letter list
moira run dead-letter replay <message_id>
```

Dead-letter replay validates the run and workload, records an audit event,
re-enqueues the worker message, and removes the original dead-letter entry.

## Kubernetes

Keep the UI away from kubeconfig and Docker credentials. For Kubernetes apply,
logs, and undeploy operations, use one of these executor paths:

- CLI from an operator workstation
- optional deployment controller with a scoped token

Production Helm values should set:

- `demoAuth.enabled=false`
- controller token Secret when the deployment controller is enabled
- NetworkPolicy for API, worker, Redis, Postgres, Qdrant, UI, and workloads
- resource requests and limits
- probes for API, worker, UI, and workloads
- persistent artifact storage with backups

## Backups

Back up:

- Postgres control-plane database
- artifact backend or PVC
- Kubernetes Secrets or external secret-manager state
- workload manifests and generated deployment values

Redis is queue/coordination state only and should not be treated as durable
control-plane storage.
