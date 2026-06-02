# On-Premises Azure DevOps Server — Deployment Automation Patterns

**Scope**: This reference covers all deployment and automation patterns for a traditional on-premises
Data Warehouse stack orchestrated from **Azure DevOps Server (on-premises)**. Every artifact —
database changes, SSIS packages, SSAS Tabular models, and PBIX reports — must be deployable via a
pipeline step with no manual GUI interaction.

---

## Core Principles

1. **Script-first**: Every deployment must be executable as a PowerShell or command-line step.
2. **Parameter-driven**: No hardcoded server names, database names, paths, or credentials anywhere. All values come from pipeline variables or SSISDB environments.
3. **Idempotent**: Scripts must be safe to re-run (deploy again = update, not duplicate).
4. **Exit-code aware**: All PowerShell scripts must return `exit 0` (success) or `exit 1` (failure) so ADO pipeline tasks correctly detect failures.
5. **Ordered deployment**: DB schema → ELT packages → SSAS model → PBIX reports. Later stages depend on earlier ones succeeding.
6. **Environment-segregated**: Dev / Test / Prod differences live in ADO variable groups and SSISDB environments, not in code.

---

## Stack Overview

```
On-Premises Azure DevOps Server
    │
    ├── Self-hosted Build/Release Agent (Windows Server)
    │       ├── Visual Studio Build Tools (MSBuild, SqlPackage.exe)
    │       ├── SQL Server Integration Services (ISDeploymentWizard.exe)
    │       ├── Tabular Editor 2 CLI (TabularEditor.exe — free, open-source)
    │       ├── Analysis Services PowerShell module (SqlServer)
    │       └── PowerShell 5.1+ / PowerShell Core
    │
    ├── Classic Pipeline Stages (run in sequence)
    │       1. Build          — compile DACPAC, .ispac, validate model files
    │       2. Deploy DB      — SqlPackage.exe publishes DACPAC to SQL Server
    │       3. Deploy SSIS    — .ispac deployed to SSISDB; environments configured
    │       4. Run ELT        — SQL Agent job executed (or direct SSIS invocation)
    │       5. Deploy SSAS    — Tabular model deployed via Tabular Editor CLI
    │       6. Process SSAS   — Full/incremental process via XMLA or PowerShell
    │       7. Deploy PBIX    — PBIX files uploaded via PBIRS REST API
    │
    └── ADO Variable Groups (Library)
            DW-Dev, DW-Test, DW-UAT, DW-Prod
            (SQL server, SSAS server, PBIRS URL, credentials)
            Linked to each Release pipeline stage
```

---

## 1. ADO Classic Pipeline Structure (Primary)

Azure DevOps Server uses **Classic Pipelines** - the visual task editor - as the primary
deployment mechanism. Classic pipelines are configured through the ADO Server UI, not stored
as YAML files in the repository.

> **YAML pipelines are optional.** The production source of truth for these DW deployments is the
> Classic build pipeline plus the Classic release pipeline stages described below.

### Classic Release Pipeline Layout

Configure a **Release Pipeline** in ADO Server with one stage per environment. Each stage uses the
same five phases; only the linked variable values change.

```
Release Pipeline: DW_Deploy
|
+-- Stage: Deploy to DEV / TEST / UAT / PROD / SUPPORT
|   (SUPPORT mirrors PROD configuration — used for production-support investigations
|    without touching PROD; shares PROD variable values with a separate agent pool slot)
|   +-- Phase 1: Deploy DW DB
|   |   1. PowerShell: $(tool_create_sql_login)
|   |   2. Command Line: $(sql_package_2019) publishes DW\Release\{ProjectName}.dacpac to $(dw_db_catalog)
|   |   3. PowerShell: $(tool_run_sql_file) runs loadStaging.sql against source DB $(db_catalog)
|   |
|   +-- Phase 2: Deploy SSIS
|   |   1. SSIS Deploy marketplace task (.ispac -> SSISDB)
|   |   2. Replace Tokens on ssis_catalog_configuration.json
|   |   3. Configure SSIS Catalog from the tokenized JSON file
|   |
|   +-- Phase 3: Deploy SSAS Tabular
|   |   1. Command Line: TabularEditor schema check on $(artifact_dir)\SSAS\Model.bim
|   |   2. Command Line: TabularEditor deploy to $(sass_server)\$(ssas_db) / $(ssas_catalog)
|   |
|   +-- Phase 4: Run ELT and Process SSAS
|   |   1. PowerShell: $(tool_runDbaAgentJob) runs the single SQL Agent job $(agent_job_name)
|   |
|   `-- Phase 5: Deploy Reports
|       1. PowerShell: $(tool_PBIRS-deployPbixReports)
|
`-- Each stage links:
    - shared variable group: Tools
    - environment-specific variables: db, db_server, db_catalog, dw_db_catalog, sass_server,
      ssas_db, ssas_catalog, agent_job_name, reportPortalUri, rsFolderPath, reportsToUpdate,
      and the required {DataSourceName}ConnectionString release variable(s)
```

### Classic Build Pipeline Layout

Configure a **Build Pipeline** in ADO Server to produce the `drop` artifact in this order:

```
Build Pipeline: DW_Build
|
`-- Agent Job: Build Artifacts
    1. NuGet restore
    2. DW Build (Visual Studio Build) - builds DW\{ProjectName}.sln
    3. Copy dacpac - **\**.dacpac -> $(Build.ArtifactStagingDirectory)\DW
    4. Copy sql files - **\**.sql -> $(Build.ArtifactStagingDirectory)\DW\SQL (Flatten = true)
    5. SSIS Build (Command Line / devenv.com) - {ProjectName}_SSIS.sln /Build Development
    6. SSIS Copy ISPAC - **\**.ispac -> $(Build.ArtifactStagingDirectory)\SSIS (Flatten = true)
    7. SSIS Copy catalog config - **\**.json -> $(Build.ArtifactStagingDirectory)\SSIS (Flatten = true)
    8. TabularEditor - Best Practices Analyzer
    9. TabularEditor - Validation Deployment
   10. TabularEditor - Build (TMDL folder -> Model.bim)
   11. Copy PBIX files - **\**.pbix -> $(Build.ArtifactStagingDirectory)\Reports (Flatten = true)
   12. Publish Artifact - drop
   13. Clean Agent Directories
```

### Adding a PowerShell Task in Classic Pipeline

Production pipelines use **two task types** in Classic release/build definitions:

**1. PowerShell task (`e213ff0f`)** - used for shared scripts in `E:\Tools\PowerShellScripts\`

```
Task Type:      PowerShell
Display Name:   Run loadStaging.sql on source DB
Type:           File Path
File Path:      $(tool_run_sql_file)
Arguments:      -sqlinstance "$(db_server)\$(db)"
                -dbname "$(db_catalog)"
                -file "$(artifact_dir)\DW\SQL\loadStaging.sql"
```

**2. Command Line task (`d9bafed4`)** - used for direct executables such as `sqlpackage.exe`
and `TabularEditor.exe`

```
Task Type:      Command Line
Display Name:   Deploy SSAS Tabular
Filename:       $(tool_tabular_editor)
Arguments:      "$(artifact_dir)\SSAS\Model.bim" -S "$(tool_tabular_editor_scripts_path)\SetConnectionStringFromEnv.cs" -D "Provider=MSOLAP;Data Source=$(sass_server)\$(ssas_db);" "$(ssas_catalog)" -O -C -V -E -R -M
Working Folder: $(System.DefaultWorkingDirectory)
```

> **All variable references** use `$(VariableName)` syntax in Classic pipelines.
> Link the shared **Tools** variable group plus the environment-specific variables to each stage.

### Variable Group Setup

Production pipelines use **two variable sources** per stage.

**Shared variable group: `Tools`**

| Variable | Value |
|---|---|
| `tool_tabular_editor` | `E:\Tools\TabularEditor\TabularEditor.exe` |
| `tool_tabular_editor_scripts_path` | `E:\Tools\TabularEditor\Scripts` |
| `tool_run_sql_file` | `E:\Tools\PowerShellScripts\runsqlfile.ps1` |
| `tool_runDbaAgentJob` | `E:\Tools\PowerShellScripts\runDbaAgentJob.ps1` |
| `tool_create_sql_login` | `E:\Tools\PowerShellScripts\createSqlLogin.ps1` |
| `tool_PBIRS-deployPbixReports` | `E:\Tools\PowerShellScripts\PBIRS-deployPbixReports.ps1` |
| `tool_archive_directory` | `E:\Tools\PowerShellScripts\archivedirectory.ps1` |
| `tool_db_backup` | `E:\Tools\PowerShellScripts\backupdatabase.ps1` |
| `tool_copyAndRestore` | `E:\Tools\PowerShellScripts\copyAndRefresh.ps1` |
| `artifact_dir` | `$(System.DefaultWorkingDirectory)\$(Build.DefinitionName)\drop` |
| `sql_package_2019` | `C:\Program Files (x86)\Microsoft Visual Studio\2019\Enterprise\Common7\IDE\Extensions\Microsoft\SQLDB\DAC\150\sqlpackage.exe` |

**Environment-specific variables / stage overrides**

| Variable | Purpose |
|---|---|
| `db` | SQL Server instance name |
| `db_server` | SQL Server host name |
| `db_catalog` | source OLTP/source-system database used by `loadStaging.sql` |
| `dw_db_catalog` | DW target database for the DACPAC publish |
| `sass_server` | SSAS host name used by the release pipeline (intentional variable name) |
| `ssas_db` | SQL instance name on the SSAS server |
| `ssas_catalog` | deployed SSAS catalog/database name |
| `agent_job_name` | single SQL Agent job that runs ELT and SSAS processing |
| `reportPortalUri` | PBIRS portal URL |
| `rsFolderPath` | PBIRS destination folder |
| `reportsToUpdate` | optional comma-separated PBIX names; blank = deploy all |
| `{DataSourceName}ConnectionString` | release variable consumed by TabularEditor C# scripts (for example `CTS_EAO_DWConnectionString`) |

Mark credentials as **secret variables**. Link both the shared `Tools` group and the environment-specific
variables under **Stage -> Variables**.

### Self-Hosted Agent Requirements

The build/release agent must have these tools and modules installed:

| Tool / Module | Install / Path | Used by |
|---|---|---|
| `sqlpackage.exe` | `$(sql_package_2019)` | DACPAC publish (Release Phase 1) |
| `TabularEditor.exe` | `$(tool_tabular_editor)` in `E:\Tools\TabularEditor\` | BPA, validation deploy, `.bim` build, SSAS deploy |
| Visual Studio `devenv.com` | `$(devenv_path)` | SSIS Build step |
| SSIS Deploy marketplace extension | Installed in ADO Server | SSIS `.ispac` deployment |
| SSIS Catalog Configuration marketplace extension | Installed in ADO Server | SSIS catalog environment configuration |
| Replace Tokens extension | Installed in ADO Server | token replacement in `ssis_catalog_configuration.json` |
| PowerShell module: **dbatools** | `Install-Module dbatools` | `runsqlfile.ps1`, `runDbaAgentJob.ps1`, `backupdatabase.ps1`, `createSqlLogin.ps1`, `copyAndRefresh.ps1` |
| PowerShell module: **ReportingServicesTools** | `Install-Module ReportingServicesTools` | `PBIRS-deployPbixReports.ps1`, `PBIRS-deployPaginatedReports.ps1` |
| PowerShell 5.1+ | Included with Windows Server | shared script execution |
| 7-Zip (`7z.exe`) | [7-zip.org](https://www.7-zip.org/) | `archivedirectory.ps1` |

> **Shared script library on the agent**: `E:\Tools\PowerShellScripts\`
> The shared **Tools** variable group exposes these paths as `$(tool_*)` variables. Do not hardcode
> agent paths in task configurations.

---

### Optional: YAML Pipeline Equivalent

> Use YAML only if your team has moved to YAML-based pipelines or is using Azure DevOps Services.
> For this environment, the Classic pipeline UI remains the source of truth. If a YAML version is
> created, it should mirror the same task order, task types, and `Tools` variable references shown above.

---
## 2. SSDT / DACPAC Deployment

### Production Phase 1 Pattern (replaces `scripts/Deploy-DACPAC.ps1`)

Production releases do **not** use a repo-local wrapper such as `Deploy-DACPAC.ps1`. The real
Classic release pattern is a three-step **Deploy DW DB** phase:

**Step 1 - PowerShell task (`e213ff0f`)**: `createSqlLogin.ps1`

```
Task Type:      PowerShell
Display Name:   Create DW SQL Login
Type:           File Path
File Path:      $(tool_create_sql_login)
Arguments:      -sqlInstance "$(db_server)\$(db)" -sqlLogin "dw" -sqlPass "dw"
```

> The `dw` SQL login and password are intentional for these environments.

**Step 2 - Command Line task (`d9bafed4`)**: publish the DACPAC with `sqlpackage.exe`

```
Task Type:      Command Line
Display Name:   Deploy DACPAC
Filename:       $(sql_package_2019)
Arguments:      /action:Publish /sourceFile:"$(artifact_dir)\DW\Release\{ProjectName}.dacpac" /targetconnectionstring:"Data Source=$(db_server)\$(db);Initial Catalog=$(dw_db_catalog);Integrated Security=True;Connection Timeout=150;Persist Security Info=False;"
```

> This uses **Windows Integrated Security** for the publish connection string, not the `dw` SQL login.

**Step 3 - PowerShell task (`e213ff0f`)**: run `loadStaging.sql` on the source database

```
Task Type:      PowerShell
Display Name:   Deploy loadStaging.sql
Type:           File Path
File Path:      $(tool_run_sql_file)
Arguments:      -sqlinstance "$(db_server)\$(db)" -dbname "$(db_catalog)" -file "$(artifact_dir)\DW\SQL\loadStaging.sql"
```

> `loadStaging.sql` is executed against the **source database** `$(db_catalog)`, not the DW database.
> This deploys the stored procedures that SSIS uses for incremental extraction.

### SSDT Project Conventions for Automation

- **Publish profiles** (`*.publish.xml`) per environment in `DW\PublishProfiles\`:
  - `Dev.publish.xml`, `Test.publish.xml`, `Prod.publish.xml`
  - Each profile contains only server/db overrides - no secrets
- **Extended property scripts** in `Scripts\Post-Deployment\ExtendedProperties\` - use upsert pattern
- **Pre/post-deploy scripts** must be idempotent (`IF NOT EXISTS ... / IF EXISTS ...`)
- **SQLCMD variables** for environment-specific values referenced in scripts

---
## 3. SSIS Package Deployment

### Production Phase 2 Pattern (replaces `scripts/Deploy-SsisProject.ps1`)

Production releases use three Classic tasks for **Deploy SSIS**:

**Task 1 - SSIS Deploy marketplace task (`06d2a35b-20bd-4f63-b331-b89eb168517b`)**

```
sourcePath:         $(System.DefaultWorkingDirectory)/{ArtifactAlias}/drop/SSIS/{ProjectName}_SSIS.ispac
destinationType:    ssisdb
destinationServer:  $(db_server)\$(db)
destinationPath:    /SSISDB/{ProjectName}-Dataload
authType:           win
whetherOverwrite:   yes
whetherContinue:    no
```

**Task 2 - Replace Tokens marketplace task (`a8515ec8`)**

```
rootDirectory:  $(artifact_dir)/SSIS
targetFiles:    ssis_catalog_configuration.json
tokenPattern:   doublebraces   (#{...}#)
```

**Task 3 - Configure SSIS Catalog marketplace task (`ff0067f5`)**

```
configSource:     filePath
configPath:       $(artifact_dir)/SSIS/ssis_catalog_configuration.json
targetServer:     $(db_server)\$(db)
authType:         win
rollBackOnError:  true
```

### `ssis_catalog_configuration.json`

- The file is stored with the SSIS solution in source control: `SSIS\{ProjectName}_SSIS\ssis_catalog_configuration.json`
- It contains token placeholders in the form `#{VariableName}#`
- The **Replace Tokens** task injects pipeline variable values into the JSON during release
- The **Configure SSIS Catalog** task then reads that JSON and creates/updates SSISDB environment values
- Standard variables configured this way are:
  - `ssis_param_LoadType` = `I` (incremental)
  - `ssis_param_SourceDB`
  - `ssis_param_SourceServer`
  - `ssis_param_TargetDB`
  - `ssis_param_TargetServer`

### SSISDB Environment Variables

In production, SSIS catalog variables are managed from `ssis_catalog_configuration.json` plus the
release variables above - not from an ad-hoc PowerShell deployment wrapper.

### SSIS Project Conventions for Automation

- **Project deployment model only** - never package deployment
- **All connection strings via project parameters** mapped to SSISDB environment variables
- **No hardcoded paths or server names** inside any SSIS package
- **The orchestrator package is triggered indirectly** by the Phase 4 SQL Agent job, not by a separate
  pipeline SQL snippet. The job runs the SSIS packages in sequence and then processes SSAS.

---
## 4. SSAS Tabular Model Deployment

### Environment Naming Convention

The source-controlled model folder name and the deployed SSAS catalog name are related but not
identical:

| Artifact | Example |
|---|---|
| TMDL folder / model source | `EAO_Tabular`, `ESB_Tabular` |
| Release target catalog (`$(ssas_catalog)`) | `EAO_UAT`, `ESB_UAT` |
| SSAS server / instance | `$(sass_server)\$(ssas_db)` |

Developers can still deploy locally for development, but release stages deploy the validated build
artifact to the environment-specific `$(ssas_catalog)`.

### Source Control: Always Use TMDL Folder

- The **TMDL folder** is the source of truth in git
- Tabular Editor workflow: **Open from folder** -> edit -> **Save as folder** -> commit
- TMDL gives granular git diffs: one `.tmdl` file per table/role/relationship

### Build vs. Release: TMDL -> `.bim` Workflow

- The **TMDL folder** is the source-control format used in `SSAS\{ModelName}\`
- **Build step 10** runs `TabularEditor.exe -B` to convert that TMDL folder into `$(Build.ArtifactStagingDirectory)\SSAS\Model.bim`
- The build also performs a **validation deployment** first, using `ClearConnectionString.cs`, to prove the model compiles and can deploy
- The **release pipeline deploys `Model.bim`**, not the TMDL folder
- This guarantees the exact artifact validated during build is the artifact promoted through release

### Option A: Tabular Editor CLI (Recommended)

Production releases use **two Command Line tasks** in Phase 3.

**Step 1 - TabularEditor Schema Check (`d9bafed4`)**

```
Task Type:      Command Line
Display Name:   TabularEditor Schema Check
Filename:       $(tool_tabular_editor)
Arguments:      "$(artifact_dir)\SSAS\Model.bim" -S "$(tool_tabular_editor_scripts_path)\SetConnectionStringFromEnv.cs" -SC -V
```

**Step 2 - TabularEditor Deploy (`d9bafed4`)**

```
Task Type:      Command Line
Display Name:   Deploy SSAS Tabular
Filename:       $(tool_tabular_editor)
Arguments:      "$(artifact_dir)\SSAS\Model.bim" -S "$(tool_tabular_editor_scripts_path)\SetConnectionStringFromEnv.cs" -D "Provider=MSOLAP;Data Source=$(sass_server)\$(ssas_db);" "$(ssas_catalog)" -O -C -V -E -R -M
```

Notes:
- Both steps deploy from the built `Model.bim` artifact
- `SetConnectionStringFromEnv.cs` reads release variables named `{DataSourceName}ConnectionString`
- Deploy flags used in production are `-O -C -V -E -R -M`

### Option B: XMLA Script via PowerShell (AMO / SqlServer module)

Production DW pipelines do **not** use a separate XMLA PowerShell deployment step. The real
pattern is the two TabularEditor Command Line tasks above.

### SSAS Processing Script

SSAS processing is **not** a separate pipeline script or task in the production release definition.
It is executed by the single Phase 4 SQL Agent job that runs ELT and then processes the model.

### TabularEditor Custom C# Scripts

These scripts are deployed with the TabularEditor tools package in `E:\Tools\TabularEditor\Scripts\`.
The development source is `C:\Projects\ISBDevOps\TabularEditor\Scripts\`.

| Script | Purpose | Inputs / Behavior | Used in |
|---|---|---|---|
| `SetConnectionStringFromEnv.cs` | Set SQL login-style provider connection strings before schema check or deploy | Reads environment variables named `{DataSourceName}ConnectionString` and applies them to matching `ProviderDataSource` objects | Release Phase 3 schema check + deploy |
| `SetADConnectionStringFromEnv.cs` | Alternative for Windows / AD impersonation deployments | Reads `service_account`, `service_account_pass`, and `{DataSourceName}ConnectionString`; sets connection string, account, password, and impersonation mode | Available for environments that need AD impersonation; not used in current pipelines |
| `ClearConnectionString.cs` | Remove real connection strings before build-time validation deploy | Replaces each provider data source connection string with the data source name placeholder | Build step 9 validation deployment |

### SSAS Deployment Conventions

- **Source of truth in git**: TMDL folder under `SSAS\{ModelName}\`
- **Release deployment artifact**: `Model.bim` built from TMDL in the build pipeline
- **Do not deploy from Visual Studio GUI or SSAS Deployment Wizard** - use TabularEditor Command Line tasks
- **Connection strings are injected at release time** by `SetConnectionStringFromEnv.cs` reading ADO release variables
- **Retain partitions on deploy** via `-R`
- **SSAS processing is not its own phase** - it is part of the same Phase 4 SQL Agent job that runs ELT

---
## 5. Power BI Report Server (PBIRS) Deployment

Three scripts cover all PBIRS deployment scenarios. All use the **`ReportingServicesTools`** PowerShell
module and Windows authentication (`-UseDefaultCredentials`).

> **Shared script path on the agent**: `E:\Tools\PowerShellScripts\`

---

### Script: `PBIRS-deployPbixReports.ps1` - Deploy Power BI (.pbix) Reports

Uploads `.pbix` files to PBIRS, then updates the SSAS connection string per environment.
Run this for **live-connection Power BI reports** (connected to SSAS Tabular).

```
Parameters:
  -reportPortalUri    URL of PBIRS (e.g. "https://test-reports.econ.gov.bc.ca/reports")
  -localFolderPath    Local folder containing .pbix files
  -rsFolderPath       PBIRS target folder path (e.g. "/EAO")
  -instance           SSAS server\instance (e.g. "$(sass_server)\$(ssas_db)")
  -database           SSAS database name (e.g. "$(ssas_catalog)")
  -reportsToUpdate    Optional - comma-separated list of report names to deploy
                      Omit to deploy all .pbix files in localFolderPath
  -accountName        Service account for datasource credential store (optional)
  -accountPass        Service account password (optional; mark as secret)
```

**ADO Classic Pipeline task - Deploy PBIX:**
```
Task Type:      PowerShell
Display Name:   Deploy PBIX Reports to PBIRS
Type:           File Path
File Path:      $(tool_PBIRS-deployPbixReports)
Arguments:      -reportPortalUri "$(reportPortalUri)"
                -localFolderPath "$(localFolderPath)"
                -rsFolderPath "$(rsFolderPath)"
                -instance "$(sass_server)\$(ssas_db)"
                -database "$(ssas_catalog)"
                -reportsToUpdate "$(reportsToUpdate)"
```

> **What this script does**: Uploads each `.pbix` (using `Write-RsRestCatalogItem`), then calls
> the PBIRS REST API to update the data source connection string to the environment-specific SSAS
> instance. Also tests the data source connection after update.

---

### Script: `PBIRS-deployPaginatedReports.ps1` - Deploy Paginated (.rdl) Reports

Deploys `.rdl`, `.rsd` (datasets), and `.rds` (data sources) to PBIRS. Configures the data
source with stored credentials. Verifies the connection after deployment.

```
Parameters:
  -ReportServiceHost      URI of PBIRS host (e.g. "https://prod-reports.example.gov.bc.ca")
  -ReportSourceFolder     Local folder containing .rdl/.rsd/.rds files
  -ReportDeployFolder     PBIRS target folder path
  -DatabaseConnectionString  Connection string for the data source
  -DatabaseUser           Data source username
  -DatabasePass           Data source password (mark as secret)
```

> **Important notes from script**:
> - Files are deployed in order: DataSources -> DataSets -> Reports (to preserve references)
> - Assumes a single data source per project - multi-datasource projects need manual configuration
> - Deploys with `-Overwrite` - safe to redeploy

**ADO Classic Pipeline task - Deploy Paginated Reports:**
```
Task Type:      PowerShell
Display Name:   Deploy Paginated Reports to PBIRS
Type:           File Path
Script Path:    E:\Tools\PowerShellScripts\PBIRS-deployPaginatedReports.ps1
Arguments:      -ReportServiceHost "$(reportPortalUri)"
                -ReportSourceFolder "$(System.DefaultWorkingDirectory)\$(Release.PrimaryArtifactSourceAlias)\PaginatedReports\"
                -ReportDeployFolder "$(rsFolderPath)"
                -DatabaseConnectionString "$(DW_CONNECTION_STRING)"
                -DatabaseUser "$(DB_USER)"
                -DatabasePass "$(DB_PASS)"
```

---

### Script: `PBIRS-updateConnection.ps1` - Update Connection String Only

Updates the SSAS connection string for reports already deployed to PBIRS. Use this when
the PBIRS report does not need to be re-uploaded (file unchanged) but the target SSAS
server/database has changed (e.g., post-restore or environment reconfiguration).

```
Parameters:
  -reportPortalUri    URL of PBIRS
  -localFolderPath    Local folder (used only for reference - not re-uploaded)
  -rsFolderPath       PBIRS folder path (include trailing /)
  -instance           SSAS server\instance
  -database           SSAS database
  -reportsToUpdate    Mandatory - comma-separated list of specific report names to update
  -accountName        Service account (optional)
  -accountPass        Service account password (optional)
```

---

### PBIRS Deployment Conventions

- **Live connection `.pbix` reports** contain no embedded data - only SSAS connection metadata
- **Ordered deployment**: SSAS model must be deployed and processed before PBIRS reports
- **Connection string format**: `Data Source=<server>;Initial Catalog=<database>;Cube=Model`
- **TLS 1.2**: The PBIRS scripts force `[Net.ServicePointManager]::SecurityProtocol = Tls12`
  - required because PBIRS servers may default to TLS 1.0
- **`ReportingServicesTools` module** must be installed on the build/release agent
- **Naming**: PBIX file name = report name in PBIRS - use consistent file naming across environments
- **PBIRS folder structure** mirrors environment, set via `$(rsFolderPath)` variable group variable:
  - Dev: `/Reports/Dev`, Test: `/Reports/Test`, Prod: `/Reports`

---
## 6. ELT Triggering from ADO Pipeline

SQL Agent is the production mechanism for triggering both SSIS ELT execution and SSAS processing
from the ADO pipeline. Use `runDbaAgentJob.ps1` from the shared library via `$(tool_runDbaAgentJob)`.

> **Shared script path on the agent**: `E:\Tools\PowerShellScripts\runDbaAgentJob.ps1`

### Script: `runDbaAgentJob.ps1` - Start SQL Agent Job and Wait

Starts a named SQL Agent job using dbatools and **waits for completion**, failing the pipeline
if the job fails.

```
Parameters:
  -sqlInstance    SQL Server instance hosting the Agent job (production pattern: "$(sass_server)\$(ssas_db)")
  -jobName        Name of the SQL Agent job to execute (production pattern: "$(agent_job_name)")
```

**What it does:**
1. Gets the job object from SQL Agent via `Get-DbaAgentJob`
2. Starts it with `Start-DbaAgentJob -Wait`
3. Checks `LastRunOutcome` and exits non-zero if the job did not succeed

**ADO Classic Pipeline task - Run ELT and Process SSAS:**
```
Task Type:      PowerShell
Display Name:   Run ELT and Process SSAS
Type:           File Path
File Path:      $(tool_runDbaAgentJob)
Arguments:      -sqlInstance "$(sass_server)\$(ssas_db)"
                -jobName "$(agent_job_name)"
```

> There is **one task** here, not separate ELT and SSAS-processing tasks.

### SQL Agent Job Conventions for SSIS / SSAS

- **One job per environment** named `{ENV}_{ProjectName}_Dataload` (for example `UAT_EAO_DW_Dataload`)
- The job runs the SSIS packages in sequence (staging -> dimensions -> facts) and then processes SSAS
- The SSIS orchestrator DTSX is triggered **inside the SQL Agent job**, not directly by the pipeline
- `runDbaAgentJob.ps1` is the pipeline wrapper that correctly fails the task if the SQL Agent job fails
- Job ownership should remain with the service account, not a personal login

---
## 7. PowerShell Script Standards for ADO Pipelines

All pipeline scripts must follow these conventions:

```powershell
# Template for any pipeline PowerShell script
[CmdletBinding()]
param(
    [Parameter(Mandatory)] [string] $RequiredParam,
    [string] $OptionalParam = 'DefaultValue'
)

# Always set this — pipeline needs non-zero exit on error
$ErrorActionPreference = 'Stop'

try {
    # ---- main logic here ----
    Write-Host "Step complete."
    exit 0
}
catch {
    Write-Error "Script failed: $_"
    exit 1   # ADO pipeline reads exit code; non-zero = fail
}
```

**Rules:**
- `[CmdletBinding()]` + `param()` block — all values come from parameters, never from the environment directly
- `$ErrorActionPreference = 'Stop'` — ensures any cmdlet error bubbles up
- `try/catch` wrapping main logic — always `exit 1` on failure
- `Write-Host` for progress (captured in ADO pipeline log)
- `Write-Error` before `exit 1` — shows red error in pipeline log
- **No `Write-Host` of secrets** — pass secrets as pipeline variables marked as `secret`

---

## 8. Repository Structure for Automated DW Projects

```
YourDWProject/
|-- DW/                                # SSDT solution and database project
|   |-- {ProjectName}.sln
|   |-- {ProjectName}.sqlproj
|   |-- Dimension/
|   |-- Fact/
|   |-- Staging/
|   |-- Internal/
|   |-- SSAS/
|   `-- Scripts/
|       `-- Post-Deployment/
|-- SSIS/
|   `-- {ProjectName}_SSIS/            # SSIS solution folder built by devenv.com
|       |-- {ProjectName}_SSIS.sln
|       |-- *.dtproj
|       |-- *.dtsx
|       `-- ssis_catalog_configuration.json
|-- SSAS/
|   `-- {ModelName}/                   # TMDL folder (Tabular Editor "Save as folder")
|       |-- database.tmdl
|       |-- model.tmdl
|       |-- tables/
|       |-- relationships/
|       `-- roles/
`-- Reports/
    `-- *.pbix
```

> **Classic pipeline definitions are stored in the ADO Server UI, not in this repo.**
> Shared deployment scripts run from the agent path `E:\Tools\PowerShellScripts\` and should be
> referenced through the shared `Tools` variable group wherever a `$(tool_*)` variable is available.

---
## 9. Checklist - Is This Deployable via Pipeline?

Use this when generating or reviewing any solution component:

| Item | Required Behavior | Y |
|---|---|---|
| No hardcoded server names | All server references via parameters/variables | |
| No hardcoded credentials | Secrets come from ADO variable group (marked secret) | |
| Idempotent scripts | Re-running = update, not duplicate or error | |
| Exit codes set | `exit 0` / `exit 1` for ADO task success/failure detection | |
| SSIS packages committed | `.dtsx` files committed to git; deployed from the built `.ispac` artifact | |
| SSIS deployed via marketplace task | Use the SSIS Deploy task, then Replace Tokens, then Configure SSIS Catalog | |
| SSAS deployed via TE CLI | Deploy `Model.bim` artifact built from TMDL in the build pipeline | |
| SSAS connection parameterized | `SetConnectionStringFromEnv.cs` reads environment-specific ADO variables | |
| PBIX connection updated | `PBIRS-deployPbixReports.ps1` updates SSAS connection per environment | |
| DACPAC safe defaults | Publish with the real `sqlpackage.exe` task and integrated security connection string | |
| Extended properties in SSDT | Post-deploy scripts, not ad-hoc SSMS execution | |
| Single SQL Agent job orchestrates runtime | `{ENV}_{Project}_Dataload` triggers both SSIS ELT **and** SSAS processing in sequence | |
| Pipeline stages ordered | DB schema -> SSIS deploy -> SSAS deploy -> ELT+SSAS process -> PBIX reports | |
| Shared scripts used | Reference agent scripts from `E:\Tools\PowerShellScripts\` via `$(tool_*)` variables where available | |
| dbatools installed on agent | Required by: `runsqlfile`, `runDbaAgentJob`, `backupdatabase`, `createSqlLogin` | |
| ReportingServicesTools installed | Required by: `PBIRS-deployPbixReports`, `PBIRS-deployPaginatedReports` | |

---
## 10. Shared PowerShell Script Library

Agent path:  `E:\Tools\PowerShellScripts\`
Dev source:  `C:\Projects\ISBDevOps\TFS\PowerShellScripts\`
ADO reference: via variable group **Tools** -> `$(tool_*)` variables

Classic pipelines should call the shared agent copies, not repo-local duplicates.

### DW Deployment Scripts

| Script | Purpose | Referenced as |
|---|---|---|
| `runsqlfile.ps1` | Run a `.sql` file against a target database | `$(tool_run_sql_file)` |
| `runDbaAgentJob.ps1` | Start a SQL Agent job and wait for completion | `$(tool_runDbaAgentJob)` |
| `createSqlLogin.ps1` | Create/repair the `dw` SQL login used by the environment | `$(tool_create_sql_login)` |
| `backupdatabase.ps1` | Backup a database before refresh/deployment workflows | `$(tool_db_backup)` |
| `copyAndRefresh.ps1` | Copy and restore a database backup into another environment | `$(tool_copyAndRestore)` |
| `archivedirectory.ps1` | Archive directories on the agent or shared storage | `$(tool_archive_directory)` |

### PBIRS Deployment Scripts

| Script | Purpose | Referenced as |
|---|---|---|
| `PBIRS-deployPbixReports.ps1` | Deploy `.pbix` files and update SSAS live-connection settings | `$(tool_PBIRS-deployPbixReports)` |
| `PBIRS-deployPaginatedReports.ps1` | Deploy `.rdl`/`.rsd`/`.rds` content to PBIRS | shared agent path / optional variable |
| `PBIRS-updateConnection.ps1` | Update report connection settings without re-uploading the file | shared agent path / optional variable |

### Module Installation (Run on Build/Release Agent)

```powershell
# Run once on the build/release agent as Administrator
Install-Module dbatools               -Scope AllUsers -Force
Install-Module ReportingServicesTools -Scope AllUsers -Force
```

### Common ADO Classic Pipeline Task Pattern

There are **two** common task patterns in the production pipelines:

**1. PowerShell task (`e213ff0f`)**

```
Task Type:      PowerShell
Type:           File Path
File Path:      $(tool_run_sql_file)
Arguments:      -sqlinstance "$(db_server)\$(db)"
                -dbname "$(db_catalog)"
                -file "$(artifact_dir)\DW\SQL\loadStaging.sql"
```

**2. Command Line task (`d9bafed4`)**

```
Task Type:      Command Line
Filename:       $(sql_package_2019)
Arguments:      /action:Publish /sourceFile:"$(artifact_dir)\DW\Release\{ProjectName}.dacpac" /targetconnectionstring:"Data Source=$(db_server)\$(db);Initial Catalog=$(dw_db_catalog);Integrated Security=True;Connection Timeout=150;Persist Security Info=False;"
```

> **Secret variables** use the same `$(VariableName)` syntax. ADO masks their values in logs.

---

## 11. TabularEditor Tools Package

The TabularEditor toolset is managed outside individual DW repos and deployed onto the agent.

- **Agent path**: `E:\Tools\TabularEditor\`
- **Dev source**: `C:\Projects\ISBDevOps\TabularEditor\`

**Package structure**

```
TabularEditor/
|-- Portable/
|   `-- TabularEditor.exe
|-- Rules/
|   `-- BPARules.json
`-- Scripts/
    |-- SetConnectionStringFromEnv.cs
    |-- SetADConnectionStringFromEnv.cs
    `-- ClearConnectionString.cs
```

**What each component is used for**

| Component | Purpose | Used in |
|---|---|---|
| `Portable\TabularEditor.exe` | Portable executable used by ADO Command Line tasks | Build steps 8-10 and Release Phase 3 |
| `Rules\BPARules.json` | Standard Best Practices Analyzer rules loaded by the `-A` flag | Build step 8 |
| `Scripts\ClearConnectionString.cs` | Clears connection strings before validation deployment | Build step 9 |
| `Scripts\SetConnectionStringFromEnv.cs` | Applies `{DataSourceName}ConnectionString` env vars before schema check/deploy | Release Phase 3 |
| `Scripts\SetADConnectionStringFromEnv.cs` | Alternative script for AD impersonation scenarios | Available but not used in current pipelines |

`BPARules.json` is managed as part of the shared TabularEditor package so every project uses the
same analyzer rules when the build pipeline runs the Best Practices Analyzer.

---

## ALM Toolkit Workflow

### Purpose
ALM Toolkit (free, https://alm-toolkit.com) is used for schema-only SSAS Tabular model comparisons and deployments. Unlike Tabular Editor deploy (which always does a full metadata replace), ALM Toolkit identifies delta changes and can deploy only what changed — reducing processing requirements and deployment risk.

### When to use ALM Toolkit vs Tabular Editor deploy
| Scenario | Tool |
|---|---|
| Initial model deployment (new model) | Tabular Editor 2 CLI (`-D` flag) |
| Full metadata replace (overwrite everything) | Tabular Editor 2 CLI (`-D` flag) |
| Schema-only delta deployment (add/modify measures, columns) | ALM Toolkit |
| Promoting a model from DEV → TEST with structural changes | ALM Toolkit |
| Checking what changed between two model versions | ALM Toolkit (compare only) |
| Partition management | SSMS or SQL Agent job (separate from model deploy) |

### ALM Toolkit comparison modes
1. **File vs Server**: compare a `.bim` file (artifact) to a live SSAS instance — used in release pipelines
2. **Server vs Server**: compare DEV SSAS to TEST SSAS directly — used for drift detection
3. **File vs File**: compare two `.bim` artifacts — used for code review / diff

### Release pipeline integration (ADO Classic)
ALM Toolkit does not have a native ADO task. The approved pattern uses PowerShell:

```powershell
# ALM Toolkit comparison and deployment via PowerShell
# Requires ALM Toolkit installed on the build agent at E:\Tools\ALMToolkit\

$almExe    = "E:\Tools\ALMToolkit\ALMToolkit.exe"
$sourceBim = "$(artifact_dir)\$(ssas_db).bim"
$targetAS  = "$(sass_server)"   # Note: sass_server is intentional historical variable name
$targetDB  = "$(ssas_db)"

# Generate comparison script (no deployment yet)
& $almExe -s $sourceBim -t "Provider=MSOLAP;Data Source=$targetAS;Initial Catalog=$targetDB" `
          -script "$(artifact_dir)\delta.xmla" -o

# Review $(artifact_dir)\delta.xmla in pipeline log (optional)

# Apply the delta
Invoke-ASCmd -Server $targetAS -Database $targetDB `
             -InputFile "$(artifact_dir)\delta.xmla"
```

Note: `$(sass_server)` is the intentional historical variable name — do not correct to `ssas_server`.

### Drift detection (Server vs Server)
Run this check as part of the DEV → TEST promotion gate to detect configuration drift:

```powershell
# Compare DEV to TEST and fail pipeline if structural differences exist
$almExe = "E:\Tools\ALMToolkit\ALMToolkit.exe"

& $almExe -s "Provider=MSOLAP;Data Source=$(sass_server_dev);Initial Catalog=$(ssas_db)" `
          -t "Provider=MSOLAP;Data Source=$(sass_server_test);Initial Catalog=$(ssas_db)" `
          -script "$(artifact_dir)\drift.xmla" -o

if ((Get-Content "$(artifact_dir)\drift.xmla") -match "alter|create|delete") {
    Write-Host "##vso[task.logissue type=warning]Structural drift detected between DEV and TEST"
}
```

### Options to preserve during deployment
When deploying with ALM Toolkit, always preserve:
- **Partitions** — never overwrite; partition management is handled separately
- **Role memberships** — AD group assignments are environment-specific; overwriting deletes them
- **Data sources** — connection strings are environment-specific

In ALM Toolkit GUI: Options → Deployment → check "Retain Partitions", "Retain Role Members", "Retain Data Sources".
In scripted mode: add `-RetainPartitions -RetainRoleMembers -RetainDataSources` flags (verify flag names match installed ALM Toolkit version).

### Review checklist (Mode G — Deployment Review)
| Check | Severity if failed |
|---|---|
| ALM Toolkit used for delta deployments (not always full replace) | 🟡 MEDIUM |
| Partitions preserved during ALM Toolkit deployment | 🔴 CRITICAL |
| Role memberships preserved during ALM Toolkit deployment | 🔴 CRITICAL |
| Data sources preserved (not overwritten with DEV connection strings) | 🔴 CRITICAL |
| Drift detection run before TEST/UAT promotion | 🟡 MEDIUM |

---

## 12. Incident Response � Roll-Forward Philosophy

### Philosophy

This organisation uses a **roll-forward** approach to production incidents. When a defect is
discovered in PROD, the response is to fix it, test it, and deploy the fix � not to revert to
a previous deployment. Reverting is a last resort reserved for catastrophic failures where a
forward fix cannot be produced quickly.

**Why roll-forward works here:**
- All code is in Git � every deployed state is reproducible from source
- The multi-environment promotion chain (DEV ? TEST ? UAT ? PROD) catches defects before PROD
- SUPPORT mirrors PROD � a safe diagnostic environment exists without touching live data
- All deployment steps are automated and idempotent � redeploying any commit is a pipeline run

### Branch Model

```
main        � always reflects what is currently deployed to PROD
dev         � long-running integration branch; all development and hotfixes land here first

feature/*   � short-lived branches off dev, merged to dev when ready
hotfix/*    � short-lived branches off dev for urgent fixes (optional; can also commit directly to dev)
```

**`main` is never directly committed to.** It is only updated by merging `dev` after a successful
PROD deployment, to keep it in sync with what is live.

### Normal Promotion Flow

```
dev  --? DEV pipeline  --? TEST pipeline  --? UAT pipeline  --? PROD pipeline
                                                                        �
                                                                        ?
                                                               merge dev ? main
                                                          (main now mirrors PROD)
```

Each environment runs the same pipeline definition with environment-specific variable groups.
Promotion to the next environment is a manual approval gate in ADO.

### Hotfix Flow

When a defect is found in PROD:

1. **Fix is committed to `dev`** � the same integration branch used for all development
2. **Pipeline is run from `dev`** � promotes through DEV ? TEST ? UAT ? PROD as normal
   - UAT gate can be fast-tracked if the fix is low-risk and urgency is high
   - SUPPORT environment can be used to verify the fix against a PROD-mirror before UAT if needed
3. **After successful PROD deployment** � merge `dev` ? `main` to keep `main` in sync

```
dev (with fix)  --? DEV  --? TEST  --? UAT (fast-track ok)  --? PROD
                                                                     �
                                                                     ?
                                                            merge dev ? main
```

### SUPPORT Environment

SUPPORT is a diagnostic mirror of PROD. It is refreshed from the same source as PROD on the
same schedule. Use it to:
- Reproduce a PROD incident without risk to live data or users
- Investigate data quality issues against PROD-equivalent data
- Optionally verify a hotfix before promoting to UAT (no restriction, but not the standard path)

SUPPORT is **never** the target of a feature deployment � only the nightly refresh pipeline runs
against it.

### Emergency Rollback (Last Resort)

Rollback is not the default response, but it is available. Because all artefacts are built from
Git-tracked source, "rollback" means re-running the pipeline against the last known-good commit.

**To redeploy a previous version:**
1. Identify the last good commit SHA or release tag in ADO
2. Trigger the pipeline manually, pointing it at that commit
3. The pipeline is idempotent � redeploying a previous state is safe

**Per-component notes:**

| Component | "Rollback" mechanism | Notes |
|---|---|---|
| **SQL DW (DACPAC)** | Re-deploy previous `.dacpac` built from last good commit | DACPAC is forward-only by default � no down scripts. SqlPackage will apply the diff between current schema and previous schema. Destructive changes (dropped columns) require explicit `--p:BlockOnPossibleDataLoss=false`. |
| **SSAS Tabular model** | Re-deploy previous TMDL JSON via TE2 CLI against PROD AS instance | Git-tracked TMDL files are the source of truth � no `.abf` backup required. TE2 deploy overwrites the model. |
| **SSIS packages** | Re-deploy previous `.ispac` via ISDeploymentWizard | SSIS catalog stores the active version � re-deployment replaces it. |
| **PBIRS reports** | Re-deploy previous `.pbix` via PBIRS REST API | The REST API upload overwrites the report. Previous versions are not retained by PBIRS natively � Git is the version store. |

> **Note on DACPAC data-loss risk:** If the bad PROD deployment included a destructive schema change
> (e.g., a column drop that removed data), a DACPAC rollback cannot recover that data. In this case,
> roll-forward with a schema fix is the only viable path. This scenario is mitigated by the
> UAT testing gate � destructive changes should never reach PROD untested.
