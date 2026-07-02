# Full Deployment Trial

This runbook is for proving the complete MoiraWeave product flow with a real
Hermes runtime:

```text
provision machine -> install platform -> bootstrap admin -> create Hermes workload
-> deploy via controller -> chat from UI -> inspect runs/events/artifacts
```

The browser is the operator cockpit. It creates workloads, runs preflight,
queues Kubernetes operations, shows health, and lets users chat with agents. It
does not store secret values, receive kubeconfig, or execute Docker/Helm/kubectl.

## What You Will Provision

For a realistic but still inexpensive trial, use two machines:

| Machine | Purpose | Recommended size |
| --- | --- | --- |
| Operator laptop/workstation | Browser, `kubectl`, `helm`, optional `moira` CLI | Any Linux/macOS workstation with outbound internet |
| Kubernetes VM | Single-node k3s cluster running API, worker, UI, Postgres, Redis, Qdrant, controller, and Hermes | 8 vCPU, 16 GiB RAM, 100 GiB SSD |

For a smaller smoke test, 4 vCPU and 8 GiB RAM can work, but real Hermes tool
use, browser backends, and longer sessions are more comfortable with 16 GiB.

Network rules for the VM:

- inbound SSH `22` from your IP;
- inbound Kubernetes API `6443` from your IP, or use an SSH tunnel instead;
- optional inbound `80/443` if you later add ingress;
- outbound internet access for image pulls, model provider calls, and Hermes
  runtime tools.

The guide below uses k3s because it gives you a complete Kubernetes cluster on
one Linux VM. Managed Kubernetes works too; skip the k3s section and point
`kubectl` at your managed cluster.

## 1. Prepare The Operator Machine

Install the client tools locally:

```bash
uv tool install moiraweave-cli
helm version
kubectl version --client
```

If PyPI is unavailable, install the CLI from a local `moiraweave-cli` checkout:

```bash
cd /path/to/moiraweave-cli
uv tool install .
```

Set placeholders you will reuse:

```bash
export VM_IP=<your-vm-public-ip>
export VM_USER=ubuntu
export MW_NS=moiraweave
export KUBECONFIG="$HOME/.kube/moiraweave-k3s.yaml"
```

## 2. Provision The Kubernetes VM

Create an Ubuntu 24.04 LTS VM with the size above. Then connect to it:

```bash
ssh "$VM_USER@$VM_IP"
```

Install k3s on the VM:

```bash
curl -sfL https://get.k3s.io | \
  sudo INSTALL_K3S_EXEC="--write-kubeconfig-mode 644" sh -

sudo kubectl get nodes
```

Leave the SSH session and copy kubeconfig to your operator machine:

```bash
mkdir -p "$HOME/.kube"
ssh "$VM_USER@$VM_IP" 'sudo cat /etc/rancher/k3s/k3s.yaml' > "$KUBECONFIG"
python -c 'import os, pathlib; p = pathlib.Path(os.environ["KUBECONFIG"]); p.write_text(p.read_text().replace("127.0.0.1", os.environ["VM_IP"]))'
chmod 600 "$KUBECONFIG"
kubectl get nodes
```

If you do not want to expose port `6443`, keep the kubeconfig server as
`https://127.0.0.1:6443` and run this tunnel before using `kubectl`:

```bash
ssh -L 6443:127.0.0.1:6443 "$VM_USER@$VM_IP"
```

## 3. Create Platform And Hermes Secrets

MoiraWeave stores secret names and status, not secret values. Create values from
the operator shell:

```bash
export JWT_SECRET_KEY="$(openssl rand -hex 32)"
export POSTGRES_PASSWORD="$(openssl rand -hex 24)"
export POSTGRES_ADMIN_PASSWORD="$(openssl rand -hex 24)"
export REDIS_PASSWORD="$(openssl rand -hex 24)"
export HERMES_API_SERVER_KEY="$(openssl rand -hex 24)"
export OPENAI_API_KEY=<your-provider-key>
```

Create the namespace and Secret:

```bash
kubectl create namespace "$MW_NS" --dry-run=client -o yaml | kubectl apply -f -

cat > /tmp/moiraweave-secrets.env <<EOF
JWT_SECRET_KEY=$JWT_SECRET_KEY
POSTGRES_DSN=postgresql://moiraweave:$POSTGRES_PASSWORD@moiraweave-postgresql:5432/moiraweave
POSTGRES_PASSWORD=$POSTGRES_PASSWORD
POSTGRES_POSTGRES_PASSWORD=$POSTGRES_ADMIN_PASSWORD
REDIS_PASSWORD=$REDIS_PASSWORD
OPENAI_API_KEY=$OPENAI_API_KEY
HERMES_API_SERVER_KEY=$HERMES_API_SERVER_KEY
EOF

kubectl create secret generic moiraweave-secrets \
  --from-env-file=/tmp/moiraweave-secrets.env \
  --namespace "$MW_NS" \
  --dry-run=client -o yaml | kubectl apply -f -

shred -u /tmp/moiraweave-secrets.env 2>/dev/null || rm -f /tmp/moiraweave-secrets.env
```

Verify only the key names:

```bash
kubectl get secret moiraweave-secrets \
  --namespace "$MW_NS" \
  -o json | python -c 'import json, sys; print("\n".join(sorted(json.load(sys.stdin)["data"].keys())))'
```

Do not paste secret values into the UI.

## 4. Install MoiraWeave Without The Controller

Install the platform first. The controller needs an admin token, and you create
that token after first-admin bootstrap.

```bash
helm upgrade --install moiraweave oci://ghcr.io/moiraweave-labs/charts/moiraweave \
  --namespace "$MW_NS" \
  --create-namespace \
  --set ui.enabled=true \
  --set apiGateway.replicaCount=1 \
  --set worker.replicaCount=1 \
  --set apiGateway.pdb.enabled=false \
  --set worker.pdb.enabled=false \
  --set-string apiGateway.extraEnv[0].name=DEMO_AUTH_ENABLED \
  --set-string apiGateway.extraEnv[0].value=false
```

Wait for the platform:

```bash
kubectl rollout status deploy/moiraweave-api-gateway --namespace "$MW_NS" --timeout=5m
kubectl rollout status deploy/moiraweave-worker --namespace "$MW_NS" --timeout=5m
kubectl rollout status deploy/moiraweave-ui --namespace "$MW_NS" --timeout=5m
kubectl get pods --namespace "$MW_NS"
```

Open the UI:

```bash
kubectl port-forward svc/moiraweave-ui 3000:80 --namespace "$MW_NS"
```

Visit:

```text
http://localhost:3000
```

## 5. Bootstrap Admin From The UI

In the UI:

1. Choose `First admin`.
2. Use a subject such as `owner`.
3. Set a strong password.
4. Submit. The UI stores the returned bearer token in local browser storage.

Then create an API key for the deployment controller:

1. Go to `Security`.
2. In API keys, create a key:

```text
name: k8s-controller
subject: owner
role: admin
team: empty
```

3. Copy the generated secret once.
4. In your operator shell:

```bash
export MOIRA_CONTROLLER_TOKEN=<copied-api-key-secret>
```

## 6. Enable The Deployment Controller

Create the controller token Secret:

```bash
kubectl create secret generic moiraweave-controller-token \
  --from-literal=MOIRA_TOKEN="$MOIRA_CONTROLLER_TOKEN" \
  --namespace "$MW_NS" \
  --dry-run=client -o yaml | kubectl apply -f -
```

Upgrade the chart with the controller enabled:

```bash
helm upgrade --install moiraweave oci://ghcr.io/moiraweave-labs/charts/moiraweave \
  --namespace "$MW_NS" \
  --reuse-values \
  --set deploymentController.enabled=true

kubectl rollout status deploy/moiraweave-deployment-controller \
  --namespace "$MW_NS" \
  --timeout=5m
```

In the UI, go to `Operations`. The Controller Queue should show no critical
token or lease alert.

## 7. Create Hermes From The UI

In the UI:

1. Go to `Workloads`.
2. Select `Hermes Agent`.
3. Keep the default values for the first trial:

```text
name: hermes
image: ghcr.io/nousresearch/hermes-agent:latest
port: 8642
adapter: hermes
persistence: enabled
workspace: /workspace
```

4. Click `Create`.
5. Click `Run preflight` or go to `Operations`.
6. Select:

```text
workload: hermes
target: kubernetes
environment: dev
```

7. Click `Preflight`.

The UI should show that required Secret keys must be verified by an operator.
Confirm from the shell without printing values:

```bash
for key in OPENAI_API_KEY HERMES_API_SERVER_KEY; do
  kubectl get secret moiraweave-secrets \
    --namespace "$MW_NS" \
    -o "jsonpath={.data.$key}" >/dev/null && echo "$key present"
done
```

## 8. Deploy Hermes From The UI

In `Operations`:

1. Click `Apply`.
2. Open the created deployment operation.
3. Watch operation events until it succeeds.

From the shell, you can also observe Kubernetes resources:

```bash
kubectl get deploy,svc,pvc \
  --namespace "$MW_NS" \
  -l moiraweave.io/workload=hermes

kubectl rollout status deploy/moiraweave-hermes \
  --namespace "$MW_NS" \
  --timeout=10m
```

Back in the UI:

1. Go to `Operations`.
2. Verify Hermes is `created`, `deployed`, `reachable`, and `healthy`.
3. If it is not reachable, open the operation logs and Kubernetes logs:

```bash
kubectl logs deploy/moiraweave-hermes --namespace "$MW_NS" --tail=200
```

## 9. Chat With Hermes From The UI

In the UI:

1. Go to `Agents`.
2. Select `hermes`.
3. Create a session.
4. Send:

```text
Hello. Confirm you are Hermes and list the runtime capabilities you can use.
```

5. Open the linked run for that message.
6. Verify:

- run status transitions from `queued` to `running` or `succeeded`;
- events stream in live;
- cancellation is available while the run is active;
- artifacts appear if Hermes returns or writes any.

Optional CLI check:

```bash
kubectl port-forward svc/moiraweave-api-gateway 8000:80 --namespace "$MW_NS"

export MOIRA_API_URL=http://localhost:8000
export MOIRA_TOKEN="$MOIRA_CONTROLLER_TOKEN"
moira agent chat hermes "hello from the CLI" --watch
```

## 10. Validate The Whole Product Flow

Use this checklist before calling the trial successful:

- `kubectl get pods -n moiraweave` shows API, worker, UI, Postgres, Redis,
  Qdrant, controller, and Hermes running.
- UI login works with the persistent admin.
- `Workloads` shows `hermes`.
- `Operations` shows no critical controller alerts.
- Hermes preflight has no blocker.
- `Apply` was launched from UI and executed by the controller.
- `Agents` can send a message to Hermes.
- The message links to a run.
- Run detail shows events and final status.
- Artifact Library can filter by `hermes`.
- Security console can create and revoke API keys.
- Secret values never appear in UI, API responses, or logs.

## 11. Common Failures

| Failure | Where to look | Fix |
| --- | --- | --- |
| UI cannot connect to API | `kubectl logs deploy/moiraweave-ui` and browser devtools | Keep `VITE_API_BASE_URL` empty in-cluster so the UI uses its API proxy |
| API pod crashloops | `kubectl logs deploy/moiraweave-api-gateway` | Check `JWT_SECRET_KEY` and `POSTGRES_DSN` in `moiraweave-secrets` |
| Worker not processing runs | Operations Center and `kubectl logs deploy/moiraweave-worker` | Check Redis, worker readiness, and run queue alerts |
| Controller queue never drains | Operations Center controller panel | Check `moiraweave-controller-token`, admin API key role, and controller pod logs |
| Hermes deployment fails | Deployment operation events and `kubectl describe pod -l moiraweave.io/workload=hermes` | Check image pull, CPU/memory, PVC, and required secrets |
| Hermes is running but chat fails | Run events and Hermes logs | Check `HERMES_API_SERVER_KEY`, Hermes API server settings, provider key, and egress |
| Hermes tools cannot access web/terminal/browser | Hermes logs and cluster network/storage policy | Fix Hermes runtime configuration; MoiraWeave does not own those tools |

## 12. Cleanup

Delete the trial:

```bash
helm uninstall moiraweave --namespace "$MW_NS"
kubectl delete namespace "$MW_NS"
```

If this was a disposable VM, destroy the VM from your cloud provider console.

## Notes For A Production-Like Follow-Up

For production-like validation, repeat the flow with:

- three Kubernetes nodes instead of one;
- ingress and TLS instead of port-forward;
- External Secrets or Vault instead of manual `kubectl create secret`;
- a real storage class with backups;
- NetworkPolicy enforcement through Cilium or Calico;
- monitoring enabled and dashboard access documented;
- separate `dev`, `staging`, and `prod` environments;
- non-admin controller token with the narrowest role MoiraWeave supports for
  deployment operations.
