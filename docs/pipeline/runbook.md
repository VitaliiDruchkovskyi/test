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
3. **Production** waits for the approval on the `production` environment. Approvers can see the build number (`yyyy.MM.dd.r`) and the commits it contains.
4. On approval, Production deploys the **same artifact** that Staging ran. The pipeline then tags the commit `prod/<build number>` and adds the run tag `released-to-prod`.

To find what is in Production: `git describe --tags --match "prod/*" --abbrev=0 origin/release/<current>`, or filter pipeline runs by the `released-to-prod` tag.

### Hotfix
1. Fix it in `main` via a `bugfix/*` PR. Development gets the fix first.
2. Cherry-pick the merged commit into the current `release/*` branch via a PR, using the **Cherry-pick** button on the completed PR in Azure Repos.
3. The merge builds the release branch. It deploys to Staging, then to Production after approval.

Never push directly to `release/*`, and never merge `release/*` back into `main`.

### What happens during a production canary
After the Web app is on the `pre` slot and its smoke test passes:

| Step | Users on the new version | Duration | Moves on when |
|---|---|---|---|
| `canary 10%` | about 10% | 10 min | The health probe hasn't failed 3 times in a row, **and** the Application Insights failed-request rate is ≤ 2% (if there were at least 50 canary requests) |
| `canary 50%` | about 50% | 10 min | Same checks |
| `reset traffic routing`, then `swap` | 100% | about 1–2 min | Done |

Each user is pinned to one version by the `x-ms-routing-name` cookie. If a check fails, routing is cleared automatically, so everyone is on the previous version and nothing is swapped. The run then fails with the reason. To abort by hand, **cancel the run**. The safety-net step clears the routing.

To watch a canary live, run this in Application Insights: `requests | where timestamp > ago(15m) | summarize count(), failed = countif(success == false) by cloud_RoleName, bin(timestamp, 1m)`.

To change the steps, duration or thresholds, edit the `canary` block on the Prod stage in `azure-pipelines.yml`, for example `steps: [ 5, 25, 50 ]`.

## Rollback

| What | How |
|---|---|
| **Web (Prod), during the canary** | Automatic. Routing is cleared and nobody stays on the new version. By hand: `az webapp traffic-routing clear -g <rg> -n <prefix>-web`, or cancel the run. |
| **Web (Prod), after the swap** | Swap the slots back: in the Azure portal, go to App Service › Deployment slots › Swap, or run `az webapp deployment slot swap -g <rg> -n <prefix>-web --slot pre --target-slot production`. After a swap, the previous version is still in `pre`. |
| **Workers** | Re-run the Prod stage of the last good release run: filter runs by the `released-to-prod` tag. The workers have no slot. |
| **Database** | A dacpac can't be rolled back automatically. This is why schema changes must be backward compatible (expand/contract): the previous app version keeps working against the new schema, so rolling back the app is enough. |

## Adding a new deployable app

1. Add an entry to the `webApps` parameter in `azure-pipelines.yml`, with `name`, `project` and `slotSwap`.
2. Create `<appServicePrefix>-<lower(name)>` in every environment. If `slotSwap: true`, also create a `pre` slot in Prod.
3. Make sure the project ships the `App_Data/ninjectModules.<profile>.xml` files as Content.

No change to the template is needed.

## Changing environment-specific config

- **Values** (connection strings, keys): put them in App Service settings or Key Vault, not in the repo.
- **Ninject module set**: add `App_Data/ninjectModules.<Profile>.xml` and set `ninjectProfile: <Profile>` on that stage in `azure-pipelines.yml`.

## Security scans

| Scan | When | Report | Gate (default: report-only) |
|---|---|---|---|
| OWASP Dependency-Check | Every run, Build stage (PRs too) | Artifact `dependency-check-<attempt>` (HTML/JSON). CVSS ≥ 7 findings also appear in the **Tests** tab | `dependencyCheckFailOnCvss`: `11` reports only. `7` fails the build on High/Critical |
| OWASP ZAP baseline | Every `release/*` run, after the Staging deploy | Artifact `zap-report-<attempt>` (HTML/JSON/XML). The job shows *partially succeeded* when ZAP warns | `zapFailOnFindings`: `false` reports only. `true` fails the Staging stage, so Production can't start |

**Rolling out the gates:**
1. Run in report-only mode for a few releases. Fix or triage the existing findings.
2. **Dependency-Check false positives**: add a `dependency-check-suppressions.xml` at the repo root. The pipeline picks it up automatically. Give each suppression a reason and an expiry date (`until=`).
3. **Noisy ZAP rules**: tune them with a ZAP rules file (`-c`), or fix the headers or cookies they flag.
4. Set `dependencyCheckFailOnCvss: 7` and `zapFailOnFindings: true` in `azure-pipelines.yml` through a PR.

**Approvers**: before you approve Production, open the Staging run's `zap-report` artifact and the Build stage's `dependency-check` report.

**Never** point a ZAP *full* (active) scan at Production. Active scans send attack traffic. Run them against Staging only, as a separate scheduled pipeline.

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
| Smoke test fails with a timeout | The app didn't start. In Prod, that means the `pre` slot, and production is unaffected because nothing has been swapped. Check the app's log stream and app settings. In Development or Staging, the new version is already live in that environment. Fix it, or re-run the last good run's stage. |
| `Run OWASP Dependency-Check` is very slow or times out | The `nvdApiKey` secret is missing or invalid, or the vulnerability-database cache was evicted. Check the `benefitnetflex-build` variable group. The first run each week refreshes the cache. |
| `Run OWASP Dependency-Check` fails with CVSS ≥ N | The gate is on (`dependencyCheckFailOnCvss` below 11) and a package has a known vulnerability. Upgrade the package, or suppress a confirmed false positive with a reason. |
| `Run ZAP baseline scan` exits with code 3 | ZAP couldn't run or couldn't reach the site. Check that Staging is reachable from hosted agents ([setup.md § 7a](setup.md#7a-security-scans-prerequisites)) and that the Docker image tag exists. |
| Staging stage shows *partially succeeded* | ZAP reported warnings in report-only mode. Read the `zap-report` artifact. Production can still be approved. |
| `canary N%` fails: "health probe failed 3 times" | The new version stopped answering 200 on `healthPath` under real traffic. Check the `pre` slot's logs. Users are already back on the previous version. |
| `canary N%` fails: "failed-request rate … above 2%" | The new version returns errors for real users. Check failures in Application Insights, filtered to the slot. Users are already back on the previous version. |
| `canary N%` warns: "Only 0 canary requests" | `canary.requestFilter` doesn't match the slot's `cloud_RoleName`, or there was very little traffic. Fix the filter ([setup.md § 7b](setup.md#7b-canary-releases-production-web-app)). Until then, the canary is judged by the health probe only. |
| `canary N%` fails: "could not query Application Insights" | The prod service connection is missing **Monitoring Reader** on Application Insights, or `appInsightsName` / `appInsightsResourceGroup` is wrong. |
| Users signed out or losing form data during a release | The `machineKey` differs between slots, or the app uses InProc session state ([setup.md § 7b](setup.md#7b-canary-releases-production-web-app)). |
| Deploy stages skipped | Expected: PR builds deploy nowhere, `main` deploys only to Development, and `release/*` deploys only to Staging and Production. Other branches deploy nowhere. |
| `Tag commit` fails with 403 / TF401027 | The build service identity is missing **Contribute** or **Create tag** on the repo ([setup.md § 5](setup.md#5-symbols-and-git-tags)). The deploy itself succeeded. Create the tag by hand or re-run the job. |
| `Tag commit` fails with "tag already exists" | The Prod stage was re-run for the same build. The tag already marks this commit, so this is safe to ignore. |
