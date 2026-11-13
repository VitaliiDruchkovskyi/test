# Solution: review, improvements and decisions

This document answers the assignment in [docs/task/](task/README.md) in one place:

1. **Review**: the main problems in the original pipeline ([`docs/task/azure.yml`](task/azure.yml)).
2. **Improve**: the improved Azure Pipelines YAML.
3. **Explain**: every change, with the reason for it.

Deeper detail is in [docs/pipeline/](pipeline/README.md). That folder covers how the pipeline works, branching, setup and the runbook.

---

## 1. Review: main problems in the original pipeline

| # | Problem | Why it matters |
|---|---|---|
| P1 | **CI only, no CD.** The pipeline stops at a `drop` artifact | Releases happen by hand or in classic Release pipelines, so they aren't versioned, reviewed or repeatable |
| P2 | **Environment config is baked in at build time.** `ninjectModules.Release.xml` is copied in before the compile | The build only fits one environment, so the same build can't be promoted from Dev to Prod |
| P3 | **No PR validation.** The pipeline triggers only on `main` | Broken code is found after it is merged |
| P4 | **Unpinned tooling.** `windows-latest` and an unpinned NuGet version | Builds break when the image changes, without any code change |
| P5 | **Packages before testing. Weak tests**: default assembly pattern, no coverage, passes when zero tests are found | Wasted build time. Tests can silently disappear |
| P6 | **Errors are hidden.** `SkipInvalidConfigurations=true`. `PublishSymbols: false` combined with `continueOnError` | Misconfigured projects build green, and there are no symbols for production debugging |
| P7 | **Bloated, mixed artifact.** The whole database source folder (`bin`, `obj`, sources) plus the web packages and blob files all go into one `drop`, using the legacy `PublishBuildArtifacts` | Slow uploads, and unclear what actually gets deployed |
| P8 | **Fragile, inconsistent paths and tasks.** `$(agent.builddirectory)\s`, wildcard solution paths, `VSBuild` and `MSBuild` mixed, variable casing differs | Hard to maintain. Changes to the agent layout break it |
| P9 | **No caching or parallelism** | Every run is slower than it needs to be |
| P10 | **No environments, approvals, versioning, rollback or credential model** | No audit trail, no control over who releases to Prod, no safe way back |

---

## 2. The improved pipeline

| File | Role |
|---|---|
| [`azure-pipelines.yml`](../azure-pipelines.yml) | Entry point: trigger, app list, Build stage, deploy stages |
| [`templates/deploy-environment.yml`](../templates/deploy-environment.yml) | One deploy job that every environment shares |

```
                       ┌─► development        (from main)
PR / push ─► Build ────┤
                       └─► staging ─► production ─► git tag prod/<build>   (from release/*)
                                     (approval)
```

- **Build stage**: it has two parallel jobs.
  - **App**: restore, compile, test, publish symbols, then publish each app.
  - **Database**: build the dacpac and collect the blob assets.
  - The outputs are three artifacts that fit any environment: `web`, `database` and `blob`.
- **Deploy stages**: each stage runs the same template, in this order: database schema, then blob assets, then the apps.
  - In production only, the Web app goes through a staging slot, a smoke test and a swap.

---

## 3. Changes and the decision behind each

Each entry shows: **Before**, the original pipeline. **After**, the new pipeline. **Why**, the decision. It also names the problem it fixes.

### Build

#### C1. One multi-stage pipeline: build *and* deploy in YAML (P1)
- **Before:** a single job that ends with a `drop` artifact.
- **After:** the stages `Build → development`, and `Build → staging → production`, all in `azure-pipelines.yml`.
- **Why:** releases become code. They are versioned, reviewed in PRs and repeatable, and every deployment can be traced to a commit and a run. YAML multi-stage pipelines are Microsoft's current recommended model. Classic Release pipelines are legacy.

#### C2. Environment-neutral artifacts; Ninject config chosen at deploy time (P2)
- **Before:** a PowerShell step overwrote `ninjectModules.xml` with the `Release` variant before the build.
- **After:** every `ninjectModules.*.xml` variant ships in the artifact. Each deploy stage copies the one it needs (`ninjectProfile` parameter), and fails loudly if the file is missing.
- **Why:** **build once, deploy many.** The bytes that reach production are the bytes that were tested in staging. Rebuilding per environment would mean shipping something that was never tested. *Next step:* choose the profile from an App Service app setting, so the file copy goes away completely.

#### C3. Three separate artifacts; dacpac only; Pipeline Artifacts (P7)
- **Before:** one `drop` artifact containing web packages, the whole database source tree and blob files, published with `PublishBuildArtifacts`.
- **After:** `web` (one folder per app), `database` (the compiled dacpac, referenced dacpacs and publish profiles) and `blob`, all published with `PublishPipelineArtifact`.
- **Why:** each deploy step downloads only what it needs. It's clear what is deployed. Pipeline Artifacts are faster (deduplicated) and are the current recommended task.

#### C4. Compile once, then publish each app to its own folder (P7, P9)
- **Before:** one solution-wide build with Web Deploy packaging (`WebPublishMethod=Package`).
- **After:** a plain compile of the solution, then a folder publish per app (`WebPublishMethod=FileSystem`, `BuildProjectReferences=false`). The existing precompile and merge settings are kept. Project-level builds use the platform `AnyCPU`, without a space.
- **Why:** nothing is compiled twice. Each app gets a clean, self-contained folder that `AzureWebApp` can zip-deploy. Keeping `PrecompileBeforePublish` and `WDPMergeOption` means runtime behaviour doesn't change. `Any CPU` with a space breaks `OutputPath` when you build a `.csproj` directly.

#### C5. Tests gate packaging; stricter test task (P5)
- **Before:** packaging ran first, then `VSTest` with defaults.
- **After:** tests run right after compiling and before any publishing. They use an explicit `*Tests.dll` pattern and collect code coverage. `failOnMinTestsNotRun` with `minimumExpectedTests: 1` fails the run if no tests are found.
- **Why:** fail fast, without wasting time packaging a broken build. The minimum-tests guard catches the case where a renamed test project quietly stops running.

#### C6. Errors are no longer hidden; build log on failure (P6)
- **Before:** `SkipInvalidConfigurations=true`.
- **After:** removed. An MSBuild binary log (`/bl`) is written and published as an artifact when the job fails.
- **Why:** a misconfigured project should fail the build, not produce a partial package. The binlog makes failures on the hosted agent debuggable, using MSBuild Structured Log Viewer.

#### C7. Symbols actually published (P6)
- **Before:** `PublishSymbols: false`, `continueOnError: true`, running after packaging, so the step did nothing.
- **After:** PDBs are source-indexed and published to the Azure Artifacts symbol server, only for `main` and `release/*` builds. The step runs *before* the publish steps, so the indexed PDBs are what ships.
- **Why:** you can debug production dumps against the exact source. PR builds skip it, to save time and keep the symbol server clean.

#### C8. Pinned agent image and NuGet version (P4)
- **Before:** `windows-latest`, and NuGet unpinned.
- **After:** `vmImage: windows-2025`, and NuGet `6.x`.
- **Why:** reproducible builds. Image upgrades happen deliberately in a PR, which the PR build validates, instead of surprising everyone on a Monday morning.

#### C9. NuGet cache and authenticated restore (P9)
- **Before:** a full restore on every run, without signing in to any feed.
- **After:**
  - `Cache@2`, keyed on the `packages.config` files. Restore is skipped on a cache hit.
  - `NuGetAuthenticate@1` before restore, with an optional `nuGetServiceConnections` parameter for external feeds that need credentials.
- **Why:**
  - The cache makes builds faster.
  - Signing in lets private and Azure Artifacts feeds work without storing passwords or tokens in the pipeline.
  - The recommended companion is an Azure Artifacts feed with nuget.org as upstream. It keeps a copy of every package, so a nuget.org outage doesn't break builds.
  - These steps stay inline, not in a separate stage. A separate stage runs on a fresh agent and would have to move the whole `packages` folder on every run.

#### C10. Parallel jobs (P9)
- **Before:** the database was built after the web build, in one job.
- **After:** the `App` and `Database` jobs run in parallel.
- **Why:** they don't depend on each other, so the build takes as long as the slower job instead of both added together.

#### C11. Explicit paths and consistent tasks (P8)
- **Before:** `$(agent.builddirectory)\s`, `**/` solution paths, a relative `blob` path, `VSBuild` and `MSBuild` mixed.
- **After:** explicit paths built from `$(Build.SourcesDirectory)` and `$(Build.ArtifactStagingDirectory)`, and `VSBuild` everywhere.
- **Why:** readable, predictable, and safe if the agent's folder layout changes.

#### C12. Build number, batched trigger and path filters (P10, P9)
- **After:** `name: $(Date:yyyy.MM.dd)$(Rev:.r)`, `batch: true`, and Markdown and `docs/**` changes excluded from the trigger.
- **Why:** a human-readable release identifier. No queue of runs during busy merges. Documentation edits don't start a build.

### Deploy

#### C13. One shared deploy template (P1, P10)
- **After:** development, staging and production all run `templates/deploy-environment.yml`. They differ only in parameters and variable groups.
- **Why:** environments can't drift apart. What is tested in staging is the same *process* that runs in production, not just the same build.

#### C14. Azure DevOps Environments `development`, `staging` and `production`, with checks (P10)
- **After:** each stage is a deployment job against its environment. Approvals, branch control, business hours and an exclusive lock are configured on the **environment** ([setup.md § 4](pipeline/setup.md#4-environments)).
- **Why:**
  - Checks on the environment can't be bypassed by editing YAML, and Azure DevOps keeps the approval history.
  - The names are short and match the language the team uses.
  - Production requires approval from release managers, not from the person who queued the run.

#### C15. Database first, backward-compatible, protected (P10)
- **After:** `SqlAzureDacpacDeployment` publishes `$(dacpacName).dacpac`, an explicit file name, with `BlockOnPossibleDataLoss=true`, before the apps are deployed. The build fails early if that dacpac is missing.
- **Why:**
  - The old app version keeps running until the swap, so every schema change must work with both versions (**expand/contract**).
  - Blocking possible data loss stops accidental column or table drops.
  - An explicit file name avoids a failure when referenced dacpacs such as `master.dacpac` sit next to the main one.

#### C16. Blob upload with Entra ID
- **After:** `az storage blob upload-batch --auth-mode login`, running through the service connection.
- **Why:** no storage account keys to store or rotate. Access is granted by an Azure role (`Storage Blob Data Contributor`).

#### C17. `zipDeploy` instead of run-from-package
- **Why:** run-from-package makes `wwwroot` read-only. Legacy ASP.NET apps often write to `App_Data`, and here they also get the Ninject file copied in at deploy time.

#### C18. Production Web: staging slot, then smoke test, then swap; workers deploy directly (P10)
- **After:**
  - In production, the Web app deploys to the `staging` slot, is smoke-tested, then swapped into production.
  - `BulkOperationWorker` and `MaintenanceWorker` (`slotSwap: false`) deploy directly.
- **Why:**
  - Zero-downtime releases for users, with instant rollback: swap back, and the previous version is still in the slot.
  - The workers skip the slot because a warm second copy would process the same background jobs twice.

#### C19. A smoke test that can't be fooled
- **After:**
  - The slot's real host name comes from `az webapp show --slot staging`.
  - Redirects are not followed, and only a direct HTTP 200 passes.
  - The test retries 10 times, 15 seconds apart.
- **Why:**
  - Newer App Service apps use unique default host names, so building `<app>-staging.azurewebsites.net` by hand can be wrong.
  - Following a redirect to a login page would report a broken deploy as healthy.

### Security

#### C20. Service connections without secrets, one per environment; variable groups (P10)
- **After:**
  - The ARM service connections `sc-benefitnetflex-<env>` use workload identity federation and are scoped to that environment's resource group.
  - Environment values live in the variable groups `benefitnetflex-<env>`, which can be linked to Key Vault.
- **Why:**
  - No client secrets to leak or rotate.
  - A development identity can't touch production.
  - Configuration values stay out of the repo.

### Process

#### C21. Release Flow branching, with deployment routed by branch (P1, P10)
- **After:**
  - `main` deploys to **development**. `release/<yyyy.MM>` deploys to **staging** and then **production**.
  - Hotfixes are made in `main` first, then cherry-picked into the release branch.
  - Branch control on the environments backs up the YAML conditions.
- **Why:**
  - Development keeps moving while a release is being validated.
  - Hotfixes have a clean path to production.
  - Production only ever runs a build that passed staging in the same run.
  - GitFlow, a single pipeline run going all the way to production, and one branch per environment were considered and rejected ([branching.md](pipeline/branching.md#why-release-flow-and-not-the-alternatives)).

#### C22. PR validation through branch policy (P3)
- **After:** "Build validation" branch policies on `main` and `release/*` run the Build stage on every PR. Deploy stages never run for PRs.
- **Why:** problems are found before merge. A branch policy is used because Azure Repos ignores the YAML `pr:` trigger.

#### C23. Git tag for every production release (P10)
- **After:** after a successful production deploy, the `TagRelease` job tags the commit `prod/<build number>` and adds the run tag `released-to-prod`.
- **Why:** the git history shows exactly what is in production. `git log prod/A..prod/B` lists the changes between two releases. It also makes it easy to find the last good run for a rollback.

#### C24. Documentation, and the original kept as reference
- **After:**
  - [docs/pipeline/](pipeline/README.md) covers the overview, branching, setup and runbook.
  - The original pipeline is kept unchanged in [docs/task/azure.yml](task/azure.yml).
- **Why:** the pipeline depends on Azure-side setup that YAML can't express, such as roles, environments and policies. Without documentation, the next person can't run it or fix it.

---

## 4. Trade-offs and known limits

- **Expand/contract is a team discipline.** The pipeline can block data loss, but it can't make a schema change backward compatible.
- **The Ninject file copy is a bridge.** Moving profile selection into an app setting is the cleaner end state.
- **The NuGet cache assumes `packages.config`.** After a move to `PackageReference`, remove the restore condition.
- **Assumptions to confirm before the first run:**
  - Hosting is Azure App Service, and the workers are web apps.
  - The database is Azure SQL with an SSDT project.
  - Project paths, the `*Tests.dll` naming, `dacpacName`, and that `healthPath` returns 200 anonymously.

  The full checklist is in [setup.md § 8](pipeline/setup.md#8-first-run-checklist).
- **Not yet run** against a real Azure DevOps project. Treat the first run as a dry run.

Further improvements are listed in [review.md § 4](pipeline/review.md#4-next-steps): health endpoint, DeployReport before production, security scanning, infrastructure as code.
