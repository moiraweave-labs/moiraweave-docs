# Adding a Custom Step

This guide shows how to create custom steps in your MoiraWeave workspace.

## Where Your Steps Live

Custom steps belong in your own project repository, not in moiraweave-steps:

```
your-workspace/
  steps/
    <task>-<impl>/        # Your custom step
  tasks/
    <task>/schema.json    # Your custom task contract
```

## Using the CLI (Recommended)

The easiest way to scaffold a step:

```bash
moira step new <task> <implementation>
```

Example:

```bash
moira step new fraud-detection ml-model
```

This generates:

```
steps/fraud-detection-ml-model/
  app/
    __init__.py
    config.py          # Settings via pydantic
    step.py            # Implement here
    main.py            # FastAPI entry
  tests/
    conftest.py
    test_step.py       # Add tests
  VERSION              # Semantic version
  step.yaml            # Step metadata
  Dockerfile           # Container build
  pyproject.toml       # Dependencies
```

## Manual Steps (For Advanced Use)

### 1. Define or Reuse a Task Schema

Task schemas define input/output contracts. Create `tasks/<task>/schema.json` if the contract is new:

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

### 2. Implement Step Package

Create `steps/<task>-<impl>/` with:
- `app/config.py` - Runtime settings
- `app/step.py` - Step logic (inherit from BaseStep)
- `app/main.py` - FastAPI server
- `step.yaml` - Step metadata
- `tests/` - Unit tests
- `VERSION` - Semantic version file
- `Dockerfile` - Container build
- `pyproject.toml` - Python dependencies

### 3. Validate

```bash
# Run unit tests
moira step test <task>-<impl>

# Validate task contracts
moira pipeline validate <pipeline-using-step>
```

## Building and Pushing Your Step

### Local Testing

```bash
moira step build fraud-detection-ml-model
```

### Push to Registry (for deployment)

```bash
moira step push fraud-detection-ml-model --bump patch
```

This:
1. Bumps the VERSION file.
2. Builds the Docker image.
3. Pushes to your registry (configured in moiraweave.yaml).

## Using Official Catalog Steps

To use official steps from moiraweave-steps without creating custom ones:

```bash
moira step add --from-catalog text-embed-fastembed@1.0
```

This downloads and pins the official step reference in your workspace.

## Step Lifecycle

1. **Create**: `moira step new` or scaffold manually.
2. **Develop**: Edit `app/step.py` and add tests.
3. **Test**: `moira step test <step-name>`.
4. **Build**: `moira step build <step-name>`.
5. **Validate**: Use in pipeline and validate contracts.
6. **Deploy**: `moira step push` if for production.

## Next: Use Your Step in a Pipeline

Once your step is ready, reference it in a pipeline definition:

```yaml
steps:
  - id: fraud-check
    task: fraud-detection
    url: http://fraud-detection-ml-model:8000
```

Then validate and run: `moira pipeline validate <name>` and `moira pipeline run <name>`.
