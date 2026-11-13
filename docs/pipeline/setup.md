# One-time setup

Do these steps in Azure and Azure DevOps before the pipeline's first run. Repeat steps 1–4 for each environment (`dev`, `test`, `prod`).

## 1. Azure resources (per environment)

| Resource | Name / requirement |
|---|---|
| App Service: Web | `<appServicePrefix>-web`. **Prod also needs a deployment slot named `staging`.** |
| App Service: BulkOperationWorker | `<appServicePrefix>-bulkoperationworker` |
| App Service: MaintenanceWorker | `<appServicePrefix>-maintenanceworker` |
| Azure SQL Database | `<sqlServerName>` / `<sqlDatabaseName>`, with Entra ID authentication enabled on the server |
| Storage account | `<storageAccountName>`, with the container `<blobContainer>` |

Set **Always On** for the worker apps, otherwise App Service unloads them when idle. In Prod, mark slot-specific settings on the Web app (connection strings, keys) as **deployment slot settings**, so they stay on their slot during a swap.

## 2. Service connections

Create one Azure Resource Manager service connection per environment. Use the **Workload identity federation** auth type; it needs no secret.

- Names: `sc-benefitnetflex-dev`, `sc-benefitnetflex-test`, `sc-benefitnetflex-prod`
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

Go to **Pipelines › Library**. Create `benefitnetflex-dev`, `benefitnetflex-test` and `benefitnetflex-prod`. Link them to Key Vault if your policy requires it.

| Variable | Example (prod) |
|---|---|
| `appServicePrefix` | `bnf-prod` |
| `resourceGroupName` | `rg-benefitnetflex-prod` |
| `sqlServerName` | `sql-benefitnetflex-prod` (without `.database.windows.net`) |
| `sqlDatabaseName` | `BenefitNetFlex` |
| `storageAccountName` | `stbenefitnetflexprod` |
| `blobContainer` | `assets` |

Grant pipeline permission to this pipeline only.

## 4. Environments

Go to **Pipelines › Environments** and create `benefitnetflex-dev`, `benefitnetflex-test` and `benefitnetflex-prod`. Recommended checks:

| Environment | Checks |
|---|---|
| dev | none |
| test | Exclusive lock (optional) |
| prod | **Approvals** (release managers, not the person who queued the run), **Business hours**, **Exclusive lock**, **Branch control** (`refs/heads/main` only) |

## 5. Symbols

The `PublishSymbols@2` task with `SymbolServerType: TeamServices` publishes to the Azure Artifacts symbol server. Make sure Azure Artifacts is enabled for the project and that the build service identity can publish symbols.

## 6. Branch policy (PR validation)

Azure Repos ignores the YAML `pr:` keyword. Go to **Repos › Branches › main › Branch policies › Build validation** and add this pipeline:

- Trigger: Automatic
- Policy requirement: Required
- Build expiration: Immediately when `main` is updated

On PR runs only the Build stage executes; the deploy stages are skipped.

## 7. First-run checklist

Check these against the real solution before you queue the first run:

- [ ] The `webApps` project paths in `azure-pipelines.yml` match the solution.
- [ ] The test assemblies match `*Tests.dll`. If they don't, update `testAssemblyVer2`.
- [ ] `dacpacName` matches the dacpac file the database project produces.
- [ ] `App_Data/ninjectModules.Release.xml` (and any other profiles) has **Build Action = Content** in each app project, so it is published.
- [ ] All projects use `packages.config`. If any use `PackageReference`, remove the restore `condition` (see the comment in the YAML).
- [ ] The projects' .NET Framework targeting pack is installed on the `windows-2025` image. Check the image's software list. If it isn't, pin `vmImage` to `windows-2022`.
- [ ] Prod `healthPath` returns **200 without a redirect** for an anonymous request on the staging slot.
- [ ] If the old release process ran hand-written SQL scripts from the database folder, add them to the `database` artifact and a step for them. Only the dacpac is shipped now.
- [ ] Run the pipeline once with only Dev wired up (or approve Test/Prod only after Dev succeeds), and treat that first run as a dry run.
