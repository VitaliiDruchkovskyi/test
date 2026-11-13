# Pipeline runbook

## Releasing

The branching model is described in [branching.md](branching.md).

### To Development
Merge a PR into `main`. The pipeline builds and deploys to **Development** automatically. With `batch: true`, pushes that arrive while a run is in progress are combined into the next run.

### To Staging and Production
1. A release manager cuts a release branch from `main`:
   ```bash
   git switch main && git pull
   git switch -c release/2026.11
   git push -u origin release/2026.11
   ```
2. The push starts a run that deploys to **Staging** automatically. QA and UAT sign off on Staging.
3. **Production** waits for the approval on the `benefitnetflex-prod` environment. Approvers can see the build number (`yyyy.MM.dd.r`) and the commits it contains.
4. On approval, Production deploys the **same artifact** that Staging ran. The pipeline then tags the commit `prod/<build number>` and adds the run tag `released-to-prod`.

To find what is in Production: `git describe --tags --match "prod/*" --abbrev=0 origin/release/<current>`, or filter pipeline runs by the `released-to-prod` tag.

### Hotfix
1. Fix it in `main` via a `bugfix/*` PR. Development gets the fix first.
2. Cherry-pick the merged commit into the current `release/*` branch via a PR, using the **Cherry-pick** button on the completed PR in Azure Repos.
3. The merge builds the release branch. It deploys to Staging, then to Production after approval.

Never push directly to `release/*`, and never merge `release/*` back into `main`.

## Rollback

| What | How |
|---|---|
| **Web (Prod)** | Swap the slots back: in the Azure portal, go to App Service › Deployment slots › Swap, or run `az webapp deployment slot swap -g <rg> -n <prefix>-web --slot staging --target-slot production`. After a swap, the previous version is still in `staging`. |
| **Workers** | Re-run the Prod stage of the last good release run: filter runs by the `released-to-prod` tag. The workers have no slot. |
| **Database** | A dacpac can't be rolled back automatically. This is why schema changes must be backward compatible (expand/contract): the previous app version keeps working against the new schema, so rolling back the app is enough. |

## Adding a new deployable app

1. Add an entry to the `webApps` parameter in `azure-pipelines.yml`, with `name`, `project` and `slotSwap`.
2. Create `<appServicePrefix>-<lower(name)>` in every environment. If `slotSwap: true`, also create a `staging` slot in Prod.
3. Make sure the project ships the `App_Data/ninjectModules.<profile>.xml` files as Content.

No change to the template is needed.

## Changing environment-specific config

- **Values** (connection strings, keys): put them in App Service settings or Key Vault, not in the repo.
- **Ninject module set**: add `App_Data/ninjectModules.<Profile>.xml` and set `ninjectProfile: <Profile>` on that stage in `azure-pipelines.yml`.

## Upgrading the build agent

`vmImage` is pinned to `windows-2025`. To move to a newer image, change the `vmImage` variable in a PR. The PR build validates the change before it reaches `main`.

## Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `NuGet restore` fails with 401 / 403 | The build identity is missing permission on the feed. It needs **Feed and Upstream Reader (Collaborator)**. For a feed in another project, also check *Limit job authorization scope* ([setup.md § 7](setup.md#7-nuget-authorization)). |
| `NuGet restore` fails with 404 / "unable to find package" | nuget.org or the feed is unavailable, or the build identity only has *Reader* on an Azure Artifacts feed and can't pull from upstream. To stop depending on nuget.org, restore through an Azure Artifacts feed with nuget.org as upstream ([setup.md § 7](setup.md#7-nuget-authorization)). |
| `NuGet restore` fails on an external feed | The NuGet service connection's key or password has expired, or the connection isn't listed in `nuGetServiceConnections`. |
| Restore uses stale packages after a `nuget.config` change | The cache key covers only the `packages.config` files. Change one of them, or bump the cache key, to force a fresh restore. |
| Build fails with no clear error | Download the `logs-<attempt>` artifact and open `build.binlog` in [MSBuild Structured Log Viewer](https://msbuildlog.com/). |
| `Unit tests` fails with "minimum expected tests not run" | No assemblies matched `*Tests.dll`. Check the pattern and make sure the test projects build in `Release`. |
| `Publish <App>` fails with an `OutputPath` error | The project doesn't define the `Release|AnyCPU` configuration. The project-level build uses `AnyCPU` without a space. |
| "Expected dacpac not found" | The `dacpacName` variable doesn't match the dacpac file the database project produces. |
| `Database schema` fails with "data loss" | The dacpac would drop or narrow a column that contains data. Split the change into expand/contract steps. Don't turn off `BlockOnPossibleDataLoss`. |
| `Database schema` fails with a login or firewall error | The service connection identity is missing the database user or the `SQL Server Contributor` role. See [setup.md](setup.md#2-service-connections). |
| `select ninject profile` fails | `ninjectModules.<profile>.xml` is not in the published output. Set its Build Action to Content. |
| `Upload blob assets` returns 403 | The service connection identity is missing `Storage Blob Data Contributor`. Role assignments can take a few minutes to apply. |
| Smoke test fails with HTTP 302 | `healthPath` redirects, for example to a login page. Point it at an anonymous endpoint that returns 200. |
| Smoke test fails with a timeout | The app didn't start on the staging slot. Check the slot's log stream and slot-specific app settings. Production is unaffected, because nothing has been swapped. |
| Deploy stages skipped | Expected: PR builds deploy nowhere, `main` deploys only to Development, and `release/*` deploys only to Staging and Production. Other branches deploy nowhere. |
| `Tag commit` fails with 403 / TF401027 | The build service identity is missing **Contribute** or **Create tag** on the repo ([setup.md § 5](setup.md#5-symbols-and-git-tags)). The deploy itself succeeded. Create the tag by hand or re-run the job. |
| `Tag commit` fails with "tag already exists" | The Prod stage was re-run for the same build. The tag already marks this commit, so this is safe to ignore. |
