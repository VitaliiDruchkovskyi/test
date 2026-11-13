# Pipeline runbook

## Releasing

1. Merge a PR into `main`. The pipeline starts automatically. With `batch: true`, pushes that arrive while a run is in progress are combined into the next run.
2. Dev and Test deploy automatically after the build succeeds.
3. Prod waits for the approval configured on the `benefitnetflex-prod` environment. Approvers can see the build number (`yyyy.MM.dd.r`) and the commits it contains.
4. On approval, Prod deploys the **same artifact** that Dev and Test ran.

To release an older build, open that run and re-run or approve its Prod stage. Don't queue a new run from an old commit.

## Rollback

| What | How |
|---|---|
| **Web (Prod)** | Swap the slots back: in the Azure portal, go to App Service › Deployment slots › Swap, or run `az webapp deployment slot swap -g <rg> -n <prefix>-web --slot staging --target-slot production`. After a swap, the previous version is still in `staging`. |
| **Workers** | Re-run the Prod stage of the last good run. The workers have no slot. |
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
| Deploy stages skipped | Expected for PR builds and for any branch other than `main`. |
