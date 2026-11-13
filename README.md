# test
teambase test

## BenefitNetFlex CI/CD

This repository holds the modernised Azure Pipelines definition for BenefitNetFlex, a .NET Framework ASP.NET MVC employee-benefits platform. The pipeline builds once and deploys the same artifacts. `main` deploys to Development, and `release/*` branches deploy to Staging and then Production (see [branching strategy](docs/pipeline/branching.md)).

| Path | Purpose |
|---|---|
| [`azure-pipelines.yml`](azure-pipelines.yml) | Pipeline entry point: the Build stage and the Development, Staging and Production deploy stages |
| [`templates/deploy-environment.yml`](templates/deploy-environment.yml) | The deploy job that every environment shares |
| [`docs/pipeline/`](docs/pipeline/README.md) | Documentation: how it works, the review of the old pipeline and the design decisions, setup, runbook |
| [`docs/task/`](docs/task/README.md) | The original assignment and the original build-only pipeline ([`azure.yml`](docs/task/azure.yml)), kept as reference |

Start with [docs/pipeline/README.md](docs/pipeline/README.md). Before the first run, work through [docs/pipeline/setup.md](docs/pipeline/setup.md).
