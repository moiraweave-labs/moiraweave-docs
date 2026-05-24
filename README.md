# MoiraWeave Docs

[![Build & Deploy Docs](https://github.com/moiraweave-labs/moiraweave-docs/actions/workflows/deploy.yml/badge.svg?branch=main)](https://github.com/moiraweave-labs/moiraweave-docs/actions/workflows/deploy.yml)
[![GitHub Pages](https://img.shields.io/badge/GitHub%20Pages-live-blue)](https://moiraweave-labs.github.io/moiraweave-docs/)
[![License: MIT](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

Public documentation site for MoiraWeave, the self-hosted AI workload and agent
operations platform.

## What this repository contains

- quickstart and onboarding guides
- concepts and ownership model documentation
- architecture and benchmark notes
- contributor-facing docs content and styles

## Local preview

```bash
uv venv
uv pip install -r requirements.txt
uv run mkdocs serve
```

Then open `http://127.0.0.1:8000`.

## Build

```bash
uv run mkdocs build --strict
```

## Deployment

- Docs are built and deployed through `.github/workflows/deploy.yml`.
- GitHub Pages must be configured to deploy from **GitHub Actions**.

## Contributing

Please follow the organization contribution policy in:

- [CONTRIBUTING.md](https://github.com/moiraweave-labs/.github/blob/main/CONTRIBUTING.md)

## Related repositories

- [moiraweave-cli](https://github.com/moiraweave-labs/moiraweave-cli)
- [moiraweave](https://github.com/moiraweave-labs/moiraweave)
- [moiraweave-steps](https://github.com/moiraweave-labs/moiraweave-steps)
