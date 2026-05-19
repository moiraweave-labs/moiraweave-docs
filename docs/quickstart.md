# Quickstart

This guide gets you from a blank workspace to a validated local run.

## Prerequisites

- Python 3.13 or newer
- `uv`
- Docker and Docker Compose
- `curl` and `jq` for health checks

## 1. Install the CLI

```bash
uv tool install moiraweave-cli
moira --help
```

If the command is available, the CLI is ready.

## 2. Create a workspace

```bash
moira init
cd my-project-moira
```

The workspace should now contain the project configuration, deployment overlays, pipelines, steps, and task contracts.

## 3. Scaffold a first step

First create the task contract if it doesn't already exist:

```bash
moira task new text-process
```

Then scaffold the step:

```bash
moira step new text-process python
```

Typical output:

```
.moiraweave/steps/text-process-python/
  app/
    __init__.py
    config.py
    main.py
    step.py
  tests/
    __init__.py
    conftest.py
    test_step.py
  VERSION
  Dockerfile
  pyproject.toml
  step.yaml
```

Steps are the deployable units that implement a task contract.

## 4. Define a pipeline

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
  - id: text-process
    task: text-process
    url: http://text-process-python:8000
```

## 5. Validate the contract

```bash
moira pipeline validate hello-world
```

Expected result:

```bash
✓ Pipeline 'hello-world' is valid
```

Validation catches contract mismatches before you run the stack.

## 6. Run the local stack

```bash
moira pipeline dev hello-world
```

The local profile starts the API gateway, worker, Redis, and Qdrant through Docker Compose.

Wait a few seconds for all services to be ready, then check the health endpoint:

```bash
curl -s http://localhost:8000/health | jq
```

The response should show `"status": "ok"`. If it does not, wait a few more seconds and retry — the worker and vector store may still be initialising.

## 7. Run a job

```bash
moira pipeline run hello-world --input '{"text": "Hello MoiraWeave!"}'
```

Then inspect the job:

```bash
moira job status <job-id>
```

## Next steps

- Add an official step from the catalog with `moira step add --from-catalog text-embed-fastembed@1.0`.
- Read [Adding a Step](adding-a-step.md) to implement custom logic.
- Read [Adding a Pipeline](adding-a-pipeline.md) to compose multiple steps.
- Read [Architecture](architecture.md) to understand the runtime model.

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `moira` command not found | CLI not installed in the active environment | Re-run `uv tool install moiraweave-cli` |
| Docker Compose fails to start | Docker daemon is not running | Confirm `docker ps` works locally |
| Health check still fails | Services are still booting | Wait a few seconds and retry |
| Pipeline validation fails | Step or task contract mismatch | Re-run `moira pipeline validate <name>` and inspect the step contract |
