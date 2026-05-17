# Architecture Benchmark

Date: 2026-05-15

This page compares MoiraWeave with established projects to make the design choices explicit and to identify the next documentation gaps.

## Decision matrix

| Area | MoiraWeave today | Reference pattern | Gap | Next improvement |
| --- | --- | --- | --- | --- |
| Step contract | KServe-style I/O and metadata endpoints in step services | KServe V2 Inference Protocol: https://kserve.github.io/website/modelserving/data_plane/v2_protocol/ | Endpoint shape is aligned, but the compliance story is not visible in docs | Add protocol-level contract tests and document expected request/response shapes |
| Pipeline composition | Declarative YAML plus runtime routing | Seldon Core 2 graphs: https://docs.seldon.ai/seldon-core-2 | The composition model is clear, but failure and retry semantics need more detail | Document backoff, retries, and step-level failure handling |
| Modular serving | Per-step deployability through a shared runtime | Ray Serve composition patterns: https://docs.ray.io/en/latest/serve/ | Modular by design, but the operational guidance is thinner than the platform story | Add deployment guidance for scaling and rollout scenarios |
| API runtime | FastAPI gateway with readiness and job lifecycle endpoints | FastAPI deployment docs: https://fastapi.tiangolo.com/deployment/ | Runtime foundations are solid; onboarding was previously fragmented | Keep the first-run path centralized in the quickstart |
| GitOps | ArgoCD app-of-apps and environment sync modes | ArgoCD docs: https://argo-cd.readthedocs.io/ | Pattern is in place; troubleshooting guidance should be easier to find | Link troubleshooting and runbooks from the main entry points |
| Progressive delivery | Argo Rollouts canary analysis | Argo Rollouts docs: https://argo-rollouts.readthedocs.io/ | Strategy exists, but success criteria are not yet documented as a decision table | Add canary thresholds and rollback criteria per environment |
| Packaging | Helm chart-driven deployment with overlays | Helm chart best practices: https://helm.sh/docs/chart_best_practices/ | Overall structure is aligned; the model is now generic | Keep the chart generic and avoid domain-specific defaults |

## Adoption verdict

- Adopted: KServe-style contracts, ArgoCD, Argo Rollouts, Helm packaging.
- Adapted: the pipeline router and job model to fit an async Redis-based runtime.
- Deferred: strict protocol conformance tests and more detailed rollout policy documentation.

## Priority follow-ups

1. Add conformance tests for the step I/O protocol.
2. Document retry and failure semantics in pipeline execution.
3. Add canary thresholds and rollback criteria in one operational table.
