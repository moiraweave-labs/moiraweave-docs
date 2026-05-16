# Quickstart

Get your first MoiraWeave project running in under 10 minutes.

## 1. Prerequisites

- Python 3.13+
- `uv` package manager
- Docker with Docker Compose
- `curl` and `jq` (for health checks)

## 2. Install MoiraWeave CLI

```bash
uv tool install git+https://github.com/moiraweave-labs/moiraweave-cli
```

Verify installation:

```bash
moira --help
```

## 3. Create Your Project

```bash
moira project init
```

This scaffolds your workspace:

```
my-project-moira/
  moiraweave.yaml
  .env
  pipelines/
  steps/
  tasks/
  deploy/
```

Change to your project directory:

```bash
cd my-project-moira
```

## 4. Scaffold Your First Step

Create a simple text processing step:

```bash
moira step new text-process python
```

This generates:

```
steps/text-process-python/
  app/
    step.py        # Your step implementation
    config.py
    main.py
  tests/
  VERSION
  Dockerfile
  pyproject.toml
  step.yaml
```

## 5. Define Your First Pipeline

```bash
moira pipeline new hello-world
```

Edit `pipelines/hello-world/pipeline.yaml`:

```yaml
name: hello-world
version: 1.0
description: Hello World pipeline
trigger:
  type: redis-stream
  stream: pipelines:hello-world:jobs
steps:
  - id: step-1
    task: text-process
    url: http://text-process-python:8000
```

## 6. Validate Pipeline Contract

```bash
moira pipeline validate hello-world
```

Expected output:

```
✓ Pipeline 'hello-world' is valid
```

## 7. Run Locally

Start the local development stack:

```bash
moira pipeline dev hello-world
```

This starts all required services (API gateway, worker, Redis, Qdrant) via Docker Compose.

Validate health:

```bash
curl -s http://localhost:8000/health | jq
```

## 8. Run Your First Job

```bash
moira pipeline run hello-world --input '{"text": "Hello MoiraWeave!"}'
```

Check job status:

```bash
moira job status <job-id>
```

## Next Steps

### Add an Official Step from Catalog

```bash
moira step add --from-catalog text-embed-fastembed@1.0
```

Then reference it in your pipeline.

### Deploy to Kubernetes

Once your pipeline works locally, deploy to staging:

```bash
moira pipeline deploy hello-world --env staging
```

See [Deployment Guide](./deployment.md) for details.

### Write Custom Step Logic

Edit `steps/text-process-python/app/step.py` to implement your business logic.

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `moira` command not found | Verify installation: `uv tool list` should show moiraweave-cli |
| Docker Compose error | Ensure Docker is running: `docker ps` |
| Health check fails | Wait 10-15 seconds for services to start, then retry |
| Step not found | Verify step exists: `moira step list` |

## Where to Go From Here

- [Adding Custom Steps](./adding-a-step.md)
- [Pipeline Best Practices](./pipeline-best-practices.md)
- [Deployment to Kubernetes](./deployment.md)
- [Project Structure Reference](./repo-structure.md)
