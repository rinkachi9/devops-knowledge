# CI/CD

Continuous integration, delivery and deployment: pipelines, release strategies, GitOps and CI/CD servers.

## Map of the area

The diagram shows the notes in reading order; subareas are marked with (subarea).

```mermaid
flowchart LR
  n0["CI/CD overview"]
  n1["GitOps"]
  n0 --> n1
  n2["Jenkins"]
  n1 --> n2
```

## Notes in reading order

1. [CI/CD overview](ci-cd-overview.md): definitions, principles, workflow, release strategies and tools.
2. [GitOps](gitops.md): principles, reconciliation, Argo CD and Flux.
3. [Jenkins](jenkins.md): architecture, pipelines, plugins and security.

## Planned topics

- Deployment strategies in depth: blue-green, canary, feature flags
- GitHub Actions
- GitLab CI/CD
- Argo CD in depth
- Artifact management and registries
- Test automation in pipelines
- Pipeline security
