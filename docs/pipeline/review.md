# Pipeline review and design decisions

This review covers the legacy pipeline ([docs/task/azure.yml](../task/azure.yml)): what was wrong with it, how the new pipeline addresses each problem, and why.

## 1. Problems in the legacy pipeline

| # | Problem | Impact |
|---|---|---|
| 1 | **CI only, no CD.** The pipeline stops after producing a `drop` artifact. | Releases happen through classic Release pipelines or by hand, so they are not versioned or reviewed in the repo. |
| 2 | **Environment config is baked in at build time.** A PowerShell step overwrites `ninjectModules.xml` with the `Release` variant before the compile. | The artifact is tied to one environment, so the same build can't be promoted from Dev to Prod. The step also uses hard-coded paths and has no error handling. |
| 3 | **No PR validation.** The pipeline triggers only on `main`. | Broken code is only found after it is merged. |
| 4 | **The agent changes without notice.** `windows-latest` moves to a new image, with a new VS and MSBuild, without warning. The NuGet version is not pinned either. | Builds break without any code change, and results can't be reproduced. |
| 5 | **Packaging happens before testing.** Everything is packaged first, then tested. `VSTest` uses the default assembly pattern and no coverage, and it passes when it finds zero tests. | Build time is wasted on packages that fail tests. Tests can disappear without anyone noticing. |
| 6 | **Errors are hidden.** `SkipInvalidConfigurations=true` masks bad project configurations. `PublishSymbols` publishes nothing (`PublishSymbols: false`), has `continueOnError: true`, and runs after packaging. | Misconfigured projects build "green". There are no symbols for debugging production. |
| 7 | **The artifact is bloated and mixed.** The step copies the entire database source folder (`**`, including `bin`, `obj` and sources) instead of the dacpac. Web packages, database files and blob files all go into one `drop`. It uses the legacy `PublishBuildArtifacts`. | Artifacts are slow and large, and it is unclear what is deployed. |
| 8 | **Paths are fragile and inconsistent.** It uses `$(agent.builddirectory)\s` instead of `$(Build.SourcesDirectory)`, wildcard `**/` solution paths, and a relative `blob` folder. The web build uses `VSBuild` while the database build uses `MSBuild`, and the variable casing differs between them. | Hard to read, and changes to the agent layout break it. |
| 9 | **No caching or parallelism.** Every run does a full NuGet restore, and the database build runs after the web build. | Builds take longer than they need to. |
| 10 | **No versioning, gates or security model.** There is no build-number scheme, no approvals, no environments, and no defined way for credentials to reach a release. | There is no audit trail, and no control over who can release to Prod. |

## 2. Key decisions

### Build once, deploy many
The Build stage produces three environment-neutral artifacts: `web`, `database` and `blob`. Every `ninjectModules.*.xml` variant ships in `web`, and the deploy stage picks one per environment (`ninjectProfile`). Development, Staging and Production all use one template, `templates/deploy-environment.yml`, so they can't drift apart. *(Fixes 1 and 2.)*

### One folder publish per app instead of one Web Deploy package for the whole solution
The solution is compiled once. Each app is then published to its own folder with `BuildProjectReferences=false`, so nothing is rebuilt. The existing precompile and merge settings (`PrecompileBeforePublish`, `WDPMergeOption`, `ContentAssemblyName=PagesAndControls`) are kept, so runtime behaviour doesn't change. The project-level builds use the platform `AnyCPU`, without a space, because `Any CPU` breaks `OutputPath` when you build a `.csproj` directly. *(Fixes 7.)*

### Tests gate packaging
Tests run straight after the compile. They use an explicit assembly pattern, collect coverage, and fail the run if no tests are found (`failOnMinTestsNotRun`). The app job and the database job run in parallel. *(Fixes 5 and 9.)*

### The database deploys first and must stay backward compatible
The dacpac is published with `BlockOnPossibleDataLoss=true` before the apps are deployed. In Prod the old app version keeps serving traffic until the slot swap, so every schema change must work with both the old and the new code. In practice this means **expand/contract migrations**: add the new column, ship code that uses it, and drop the old column in a later release.

### In Prod, only Web uses a staging slot, smoke test and swap
The Web app deploys to the `staging` slot, is smoke-tested, then swapped in. This gives zero-downtime releases and a one-click rollback (swap back). The workers deploy directly (`slotSwap: false`), so two copies never process the same background jobs at once. Deployment uses `zipDeploy` rather than run-from-package, because run-from-package makes `wwwroot` read-only and legacy apps often write to `App_Data`.

### Release Flow branching
`main` deploys to Development. Release branches (`release/*`) deploy to Staging and then Production. Hotfixes go into `main` first and are cherry-picked into the release branch. Development keeps moving while a release is validated, and there is a clean hotfix path. Every environment still receives an artifact built once from one commit. Each Production release is tagged `prod/<build number>` in git. The reasoning and the alternatives considered are in [branching.md](branching.md). *(Fixes 1 and 10.)*

### Security is handled outside the YAML
- Service connections use **workload identity federation**, so no client secrets are stored or rotated.
- Each environment has its own **variable group**, linked to Key Vault.
- Blob upload authenticates with **Entra ID** (`--auth-mode login`) instead of storage account keys.
- **Approvals and checks** are set on the Environments, so changing the YAML can't bypass them.
- Each environment has its own service connection. A Dev identity can't touch Prod.

*(Fixes 10.)*

### Hygiene
- The agent image (`windows-2025`) and the NuGet major version (`6.x`) are pinned. *(Fixes 4.)*
- NuGet packages are cached and restore is skipped on a cache hit. *(Fixes 9.)*
- `NuGetAuthenticate@1` runs before restore, so authenticated feeds work without stored secrets: Azure Artifacts through the build identity, external feeds through NuGet service connections (`nuGetServiceConnections` parameter). Together with an Azure Artifacts feed that has nuget.org as upstream, packages stay available during public outages ([setup.md § 7](setup.md#7-nuget-authorization)).
- The build number is date-based (`yyyy.MM.dd.r`). *(Fixes 10.)*
- Pipeline Artifacts replace Build Artifacts. *(Fixes 7.)*
- An MSBuild binlog is published when the build fails. Symbols go to the Azure Artifacts symbol server, for `main` and `release/*` builds only. *(Fixes 6.)*
- All paths are explicit and use predefined variables. *(Fixes 8.)*
- PR validation runs through a branch policy. *(Fixes 3. See [setup.md](setup.md#6-branch-policies-pr-validation).)*

## 3. Corrections made while adding the proposal to the repo

The first proposal had a few defects. They were fixed in the committed files:

| Issue in the proposal | Fix |
|---|---|
| The dacpac was deployed with the pattern `database/*.dacpac`. If the project references another dacpac, such as `master.dacpac`, the pattern matches more than one file and `SqlAzureDacpacDeployment` fails. | A new variable, `dacpacName`, sets the file. The build checks that the file exists, and the deploy targets it by name. Referenced dacpacs still ship next to it. |
| The smoke test built the host name as `<app>-staging.azurewebsites.net`. That is wrong for apps created with a *unique default hostname* (`<app>-<hash>.<region>.azurewebsites.net`). | The smoke test now asks Azure for the slot's real host name (`az webapp show --slot staging`). |
| The smoke test followed redirects. On an app behind sign-in, the login page's `200` made a broken deploy look healthy. | Redirects are no longer followed. Only a direct `200` passes. |
| The `ninjectProfile` parameter existed but no stage set it, so nobody could see where to change it. | Every stage now sets it explicitly. |
| Display names contained an en-dash (`–`). | Replaced with a plain hyphen. |

## 4. Next steps

1. **Remove the config-file swap entirely.** Choose the Ninject module set from an App Service app setting at startup, for example `NinjectProfile=Prod`. The pipeline step then goes away, and config lives only in App Service settings and Key Vault.
2. **Move from `packages.config` to `PackageReference`.** This enables `packages.lock.json`, deterministic restores, `dotnet list package --vulnerable`, and a simpler cache key. Then drop the restore condition, because the cache no longer covers `obj/project.assets.json`.
3. **Add a dacpac DeployReport before Prod.** Run `SqlPackage /Action:DeployReport` against Prod in the Staging stage and attach the report to the Prod approval.
4. **Add a dedicated `/health` endpoint** that checks the database and dependencies, is anonymous, and doesn't redirect. Point `healthPath` at it, and use it for App Service Health check too.
5. **Run security scanning in the Build stage.** Options are Microsoft Security DevOps or GitHub Advanced Security for Azure DevOps (dependency, secret and code scanning).
6. **Define the infrastructure as code** with Bicep: App Services, slots, SQL, storage and role assignments. This keeps the assumptions in [setup.md](setup.md) reproducible.
