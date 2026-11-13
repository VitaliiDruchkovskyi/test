# One-time setup

Do these steps in Azure and Azure DevOps before the pipeline's first run. Repeat steps 1–4 for each environment (`development`, `staging`, `production`).

## 1. Azure resources (per environment)

| Resource | Name / requirement |
|---|---|
| App Service: Web | `<appServicePrefix>-web`. **Prod also needs a deployment slot named `staging`.** This is the slot that is swapped into production. It is not the `staging` environment, which has its own App Services |
| App Service: BulkOperationWorker | `<appServicePrefix>-bulkoperationworker` |
| App Service: MaintenanceWorker | `<appServicePrefix>-maintenanceworker` |
| Azure SQL Database | `<sqlServerName>` / `<sqlDatabaseName>`, with Entra ID authentication enabled on the server |
| Storage account | `<storageAccountName>`, with the container `<blobContainer>` |

Set **Always On** for the worker apps, otherwise App Service unloads them when idle. In Prod, mark slot-specific settings on the Web app (connection strings, keys) as **deployment slot settings**, so they stay on their slot during a swap.

## 2. Service connections

Create one Azure Resource Manager service connection per environment. Use the **Workload identity federation** auth type; it needs no secret.

- Names: `sc-benefitnetflex-dev`, `sc-benefitnetflex-staging`, `sc-benefitnetflex-prod`
- Scope: that environment's resource group only.
- Under **Security**, grant pipeline permission to this pipeline only. Don't select "Grant access permission to all pipelines".

Give each connection's identity these roles:

| Scope | Role | Why |
|---|---|---|
| Resource group | `Website Contributor` | Deploy apps, swap slots, read the slot host name |
| SQL server | `SQL Server Contributor` | The dacpac task adds a temporary firewall rule for the agent's IP |
| SQL database | Entra user with `db_owner`, created with `CREATE USER [<identity name>] FROM EXTERNAL PROVIDER; ALTER ROLE db_owner ADD MEMBER [<identity name>];` | Publish the schema |
| Storage account | `Storage Blob Data Contributor` | `--auth-mode login` blob upload |

## 3. Variable groups

Go to **Pipelines › Library**. Create `benefitnetflex-dev`, `benefitnetflex-staging` and `benefitnetflex-prod`. Link them to Key Vault if your policy requires it.

| Variable | Example (prod) |
|---|---|
| `appServicePrefix` | `bnf-prod` |
| `resourceGroupName` | `rg-benefitnetflex-prod` |
| `sqlServerName` | `sql-benefitnetflex-prod` (without `.database.windows.net`) |
| `sqlDatabaseName` | `BenefitNetFlex` |
| `storageAccountName` | `stbenefitnetflexprod` |
| `blobContainer` | `assets` |

Grant pipeline permission to this pipeline only.

Also create **`benefitnetflex-build`**, which the Build stage uses:

| Variable | Value |
|---|---|
| `nvdApiKey` (**secret**) | A free API key from [nvd.nist.gov/developers/request-an-api-key](https://nvd.nist.gov/developers/request-an-api-key). Without it, OWASP Dependency-Check's database update takes a very long time. |

## 4. Environments

Go to **Pipelines › Environments** and create `development`, `staging` and `production`. Recommended checks:

| Environment | Checks |
|---|---|
| development | **Branch control**: `refs/heads/main` |
| staging | **Branch control**: `refs/heads/release/*`. **Exclusive lock**, so a newer release run waits for the current one |
| production | **Branch control**: `refs/heads/release/*`. **Approvals** (release managers, not the person who queued the run). **Business hours**. **Exclusive lock** |

Branch control backs up the stage conditions in the YAML. Even if someone edits the conditions, Production only accepts runs from a release branch. See [branching.md](branching.md).

## 5. Symbols and git tags

- **Symbols**: the `PublishSymbols@2` task with `SymbolServerType: TeamServices` publishes to the Azure Artifacts symbol server. Make sure Azure Artifacts is enabled for the project and that the build service identity can publish symbols.
- **Git tags**: after a successful Production deploy, the `TagRelease` job pushes the tag `prod/<build number>`. Go to **Project settings › Repositories › BenefitNetFlex › Security**. For the `<Project> Build Service (<Org>)` identity, set **Contribute** = Allow and **Create tag** = Allow. Don't grant **Force push**.
- **Protect the tags**: restrict who can create or delete `prod/*` tags manually, through repository security or a tag policy, so only the pipeline writes them.

## 6. Branch policies (PR validation)

Azure Repos ignores the YAML `pr:` keyword. Set up the policies under **Repos › Branches**. For `release/*`, use **Branch policies** on the folder pattern: *Add branch protection* › `release/*`.

| Policy | `main` | `release/*` |
|---|---|---|
| Require a minimum number of reviewers | 2 (1 for small teams). Reset votes on new pushes | 1, from the release managers group |
| Check for linked work items | Required | Required |
| Check for comment resolution | Required | Required |
| Limit merge types | **Squash merge** only | Squash merge or rebase. These PRs carry cherry-picks |
| Build validation (this pipeline) | Required. Expires immediately when `main` is updated | Required |
| Who can push | Nobody directly. Everything goes through a PR | Nobody directly. Release managers create the branch |

Branch security: allow **Create branch** under `release/` only for the release managers group. Deny **Force push** on `main` and `release/*` for everyone.

On PR runs only the Build stage executes; the deploy stages are skipped.

## 7. NuGet authorization

The Build job runs `NuGetAuthenticate@1` ("Authenticate NuGet feeds") right before `NuGet restore`. It gives the restore credentials for authenticated feeds. No PATs or passwords are stored in the pipeline. The feeds themselves come from the solution's own `nuget.config`, if it has one, and otherwise from nuget.org.

- **Azure Artifacts feeds in this organization** are covered automatically, through the build service identity. On the feed, go to **Feed settings › Permissions** and add `<Project> Build Service (<Org>)` with the role **Feed and Upstream Reader (Collaborator)**. *Feed Reader* is not enough to pull a package from an upstream source for the first time.
  - If the feed is in a different project or is organization-scoped, either add `Project Collection Build Service (<Org>)`, or turn off **Limit job authorization scope to current project**.
- **External feeds that need credentials**, such as a vendor's private feed:
  1. Go to **Project settings › Service connections › New › NuGet**, using *ApiKey* or *Basic authentication*. Name it, for example, `sc-nuget-vendor`, and grant pipeline permission to this pipeline only.
  2. Set the pipeline parameter `nuGetServiceConnections` to that name, either as the default in `azure-pipelines.yml` or at queue time. Separate several connections with commas.

**Recommended, for the "package not available" problem:** create an Azure Artifacts feed with nuget.org as an **upstream source** (*Artifacts › Create feed › Include packages from common public sources*), and point the solution's `nuget.config` at it. The feed saves a copy of every package it serves, so a nuget.org outage or a deleted package no longer breaks the build. The authentication step above already covers that feed.

## 7a. Security scans: prerequisites

- **Dependency-Check**: needs the `benefitnetflex-build` variable group with `nvdApiKey` (§ 3). It downloads the Dependency-Check CLI from GitHub and the vulnerability data from NVD. If the organization restricts outbound traffic, allow `github.com`, `objects.githubusercontent.com` and `services.nvd.nist.gov`.
- **ZAP baseline scan**: runs on a Microsoft-hosted **Ubuntu** agent and scans the Staging Web app over HTTPS. Staging must be **reachable from Microsoft-hosted agents**. If it is behind a private endpoint, IP restrictions or a VPN, use a self-hosted agent inside the network, or allow the Azure DevOps IP ranges. It scans unauthenticated pages only. Scanning signed-in pages needs a ZAP authentication context, which is a later step.
- **Smoke tests**: for every app with `smokeTest: true`, `healthPath` must return 200 anonymously, without a redirect, in **every** environment, not only on the Prod slot.

## 8. First-run checklist

Check these against the real solution before you queue the first run:

- [ ] The `webApps` project paths in `azure-pipelines.yml` match the solution.
- [ ] The test assemblies match `*Tests.dll`. If they don't, update `testAssemblyVer2`.
- [ ] `dacpacName` matches the dacpac file the database project produces.
- [ ] `App_Data/ninjectModules.Release.xml` (and any other profiles) has **Build Action = Content** in each app project, so it is published.
- [ ] If restore uses Azure Artifacts or other authenticated feeds, the build service and service connections have access (§ 7).
- [ ] All projects use `packages.config`. If any use `PackageReference`, remove the restore `condition` (see the comment in the YAML).
- [ ] The projects' .NET Framework targeting pack is installed on the `windows-2025` image. Check the image's software list. If it isn't, pin `vmImage` to `windows-2022`.
- [ ] `healthPath` returns **200 without a redirect** for an anonymous request, in every environment, for apps with `smokeTest: true`.
- [ ] The `benefitnetflex-build` variable group exists with the `nvdApiKey` secret, and Staging is reachable from hosted agents for ZAP (§ 7a).
- [ ] If the old release process ran hand-written SQL scripts from the database folder, add them to the `database` artifact and a step for them. Only the dacpac is shipped now.
- [ ] Treat the first run as a dry run. Merge to `main` and verify Development. Then cut a first `release/*` branch and verify Staging before approving Production.
