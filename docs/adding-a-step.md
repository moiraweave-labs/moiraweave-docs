# Adding a Custom Step

Use this guide when you need a new step inside your own workspace.

> Custom steps live in the customer workspace. The official catalog is published separately in `moiraweave-steps`.

## 1. Scaffold the step

```bash
moira step new <task> <implementation>
```

Example:

```bash
moira step new fraud-detection ml-model
```

Generated structure:

```
steps/fraud-detection-ml-model/
  app/
    config.py
    main.py
    step.py
  tests/
  VERSION
  Dockerfile
  pyproject.toml
  step.yaml
```

## 2. Define the task contract

Task contracts describe the input and output shape of the step. Create `tasks/<task>/schema.json` if the task is new, or reuse an existing contract if one already fits.

Example:

```json
{
  "task": "fraud-detection",
  "version": "1.0",
  "description": "Detect fraudulent transactions",
  "inputs": [
    {
      "name": "transaction",
      "datatype": "BYTES",
      "shape": [1],
      "required": true
    }
  ],
  "outputs": [
    {
      "name": "fraud_score",
      "datatype": "BYTES",
      "shape": [1]
    }
  ]
}
```

## 3. Implement the service

Keep the step package focused:

- `app/config.py` for runtime settings
- `app/step.py` for the step implementation
- `app/main.py` for the service entry point
- `tests/` for unit coverage
- `step.yaml` for metadata
- `VERSION` for release management

## 4. Validate locally

```bash
moira step test <task>-<impl>
moira pipeline validate <pipeline-using-step>
```

If validation fails, fix the contract mismatch before building an image.

## 5. Build and publish

```bash
moira step build fraud-detection-ml-model
moira step push fraud-detection-ml-model --bump patch
```

`moira step push` bumps the version, builds the container image, and pushes it to the registry configured in the workspace.

## 6. Use the step in a pipeline

```yaml
steps:
  - id: fraud-check
    task: fraud-detection
    url: http://fraud-detection-ml-model:8000
```

Once the pipeline references the step, validate it again and then run it end to end.
