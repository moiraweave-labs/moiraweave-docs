# Adding a Pipeline

Use this guide when you want to compose existing steps into a new workflow.

## 1. Create the pipeline definition

Create `pipelines/<name>/pipeline.yaml` and describe the workflow in terms of step ids, tasks, and service URLs.

Example:

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

## 2. Validate the contract graph

```bash
moira pipeline validate <name>
```

Validation checks compatibility between adjacent steps before you run the workflow.

## 3. Keep deployment settings workspace-owned

Pipeline definitions belong in the workspace. Any environment-specific deployment settings should stay with the repository or overlay that owns the environment, not in the shared runtime.

## 4. Run and iterate

```bash
moira pipeline run <name> --input '{"text": "Hello MoiraWeave!"}'
moira job status <job-id>
```

If the pipeline evolves, update the YAML, validate again, and keep the contract surface explicit.
