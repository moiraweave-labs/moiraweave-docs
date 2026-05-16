# Repository Structure and Ownership Model

MoiraWeave is organized as a multi-repository project. Each repository has a clear ownership boundary and role.

## Repository Map

### 1. moiraweave-cli (Your entry point)

**Role**: Developer UX and project scaffolding.

**You use it to**:
- Create your workspace: `moira project init`
- Define pipelines and steps in your workspace
- Deploy locally or to Kubernetes
- Manage your project configuration

**Structure**:
```
moiraweave-cli/
  moira_cli/          # CLI source code (don't touch this)
  tests/
  pyproject.toml
  README.md
```

### 2. Your Workspace (Customer-owned)

**Role**: Your project code and configuration.

**Structure**:
```
your-company-moira/  (your repository)
  moiraweave.yaml             # Project config (e.g., registry, environments)
  .env                        # Secrets (don't commit)
  pipelines/
    <pipeline-name>/
      pipeline.yaml           # Your pipeline definition
  steps/
    <task>-<impl>/            # Your custom steps
      app/
        step.py               # Implementation
        config.py
        main.py
      tests/
      Dockerfile
      VERSION
      pyproject.toml
      step.yaml
  tasks/
    <task>/
      schema.json             # Your task contract (if new)
  deploy/
    values-local.yaml         # Local deployment config
    values-staging.yaml       # Staging deployment config
    values-prod.yaml          # Production deployment config
```

**What you own**:
- Pipeline definitions
- Custom step implementations
- Task contracts you define
- Environment-specific deployment configuration
- Secrets and credentials

### 3. moiraweave-core (Platform runtime)

**Role**: Runtime services, infrastructure templates, and deployment orchestration.

**What it owns**:
- API gateway, worker services, monitoring
- Helm charts, Kubernetes manifests, Terraform modules
- Step SDK (the base class for steps)
- Reference pipeline definitions (for documentation)

**What it does NOT own**:
- Your project code
- Your pipelines
- Your custom steps
- Your environment configuration

**Structure**:
```
moiraweave-core/
  services/           # Platform services
    api-gateway/
    worker/
    shared/
    step-sdk/
  infra/
    helm/             # Helm charts for deployment
    k8s/              # Kubernetes manifests
    terraform/        # Terraform modules
  tests/              # Integration tests
```

### 4. moiraweave-steps (Official catalog)

**Role**: Reusable, tested step implementations published by the team.

**What it owns**:
- Official step implementations (e.g., text-embed-fastembed)
- Official task schemas
- Published container images and versions

**What users do**:
- Consume steps by reference and version via CLI
- Do NOT clone this repository normally
- Contribute new steps to the catalog (via PR)

**Structure**:
```
moiraweave-steps/
  steps/
    text-embed-fastembed/     # Official step
    vision-clip/              # Official step
  tasks/
    text-embed/schema.json    # Official task
```

### 5. moiraweave-docs (Documentation)

**Role**: User-facing documentation, guides, and API reference.

**What it owns**:
- Quickstart guides (using CLI-first approach)
- Architecture documentation
- API reference
- Deployment guides

### 6. .github (Org-wide policies)

**Role**: Shared community files and reusable workflows.

**What it owns**:
- Issue templates
- PR templates
- Security policy
- Code of conduct
- Org profile README

## Ownership Decision Tree

When deciding where code should live, ask:

1. **Is it customer business logic or configuration?** → Your workspace
2. **Is it a generic, reusable step?** → moiraweave-steps (if official) or your workspace (if custom)
3. **Is it runtime platform code?** → moiraweave-core
4. **Is it CLI and UX?** → moiraweave-cli
5. **Is it documentation or community policy?** → moiraweave-docs or .github

## Typical User Workflow

```
User installs CLI
    ↓
User creates workspace (moira project init)
    ↓
User defines pipelines and custom steps in workspace
    ↓
User optionally consumes official steps (moira step add --from-catalog)
    ↓
User validates and deploys from CLI
    ↓
Runtime (moiraweave-core) executes user's pipelines
```

**No cloning of moiraweave-core or moiraweave-steps required** for normal usage.

## When Do You Need to Clone Upstream Repos?

- **moiraweave-cli**: Only if contributing to the CLI itself
- **moiraweave-core**: Only if contributing to platform or understanding internals
- **moiraweave-steps**: Only if contributing official steps to the catalog
- **moiraweave-docs**: Only if improving documentation
