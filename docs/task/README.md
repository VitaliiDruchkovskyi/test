# Task

This folder holds the original assignment and its input, kept as reference. The solution is in [`/azure-pipelines.yml`](../../azure-pipelines.yml) and [`/templates/deploy-environment.yml`](../../templates/deploy-environment.yml), and it is documented in [`docs/pipeline/`](../pipeline/README.md).

## Assignment

> **Context:** We maintain an employee benefits platform. It is a .NET Framework ASP.NET MVC app. We use Azure Repos and Azure Pipelines. The attached azure.yml is our build pipeline. We want to modernise the whole build-to-release flow.
>
> **What to do:**
>
> 1. Review it. List the main problems in the pipeline.
> 2. Improve it. Suggest an improved pipeline as Azure Pipelines YAML.
> 3. Explain your decisions.
>
> You're welcome to use documentation and AI tools.

## Input

[`azure.yml`](azure.yml) is the original build pipeline, exactly as provided. It is reference material only. Don't point a pipeline at it.

## Where each part is answered

All three parts are answered in one document, [docs/SOLUTION.md](../SOLUTION.md). The table below points to the detailed docs.

| Part | Answer |
|---|---|
| 1. Problems in the pipeline | [docs/pipeline/review.md § 1](../pipeline/review.md#1-problems-in-the-legacy-pipeline) |
| 2. Improved pipeline | [azure-pipelines.yml](../../azure-pipelines.yml), [templates/deploy-environment.yml](../../templates/deploy-environment.yml) |
| 3. Decisions | [docs/pipeline/review.md § 2–4](../pipeline/review.md#2-key-decisions), plus [setup](../pipeline/setup.md) and [runbook](../pipeline/runbook.md) |
| Git strategy (Development → Staging → Production) | [docs/pipeline/branching.md](../pipeline/branching.md) |
