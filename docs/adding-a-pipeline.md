# Adding a Pipeline

Use this guide when you want to compose existing steps into a new workflow.

## 1. Create the pipeline definition

Create `pipelines/<name>/pipeline.yaml` and describe the workflow in terms of step ids, tasks, and service URLs.

Single-step example:

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

Multi-step example — embed text then index into a vector store:

```yaml
name: text-search-rag
version: 1.0
description: Embed documents and index them for vector search
trigger:
  type: redis-stream
  stream: pipelines:text-search-rag:jobs
steps:
  - id: embed
    task: text-embed
    url: http://text-embed-fastembed:8000
  - id: index
    task: vector-index
    url: http://vector-index-qdrant:8000
    depends_on:
      - embed
```

Each step declares a `task` (the contract it implements) and a `url` (the running service). `depends_on` expresses the execution order.

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
