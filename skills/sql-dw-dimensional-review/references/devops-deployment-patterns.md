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
    │       ├── Tabular Editor CLI (te.exe or te3.exe)
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

Azure DevOps Server uses **Classic Pipelines** — the visual task editor — as the primary
deployment mechanism. Classic pipelines are configured through the ADO Server UI, not stored
as YAML files in the repository.

> **YAML pipelines are optional.** All PowerShell scripts in Sections 2–9 work identically
> whether called from a Classic pipeline task or a YAML pipeline step. The scripts are what
> matter; the pipeline type is a configuration choice.

### Classic Release Pipeline Layout

Configure a **Release Pipeline** in ADO Server with one stage per environment:

```
Release Pipeline: DW_Deploy
│
├── Stage: Deploy to DEV     (auto-triggers on Build completion)
│   ├── Pre-deployment conditions: none (auto-deploy)
│   └── Tasks:
│       1. PowerShell: Deploy-DACPAC.ps1
│       2. PowerShell: Deploy-SsisProject.ps1
│       3. PowerShell: Deploy-SsasModel.ps1
│       4. PowerShell: Process-SsasDatabase.ps1
│       5. PowerShell: Deploy-PbixReports.ps1
│
├── Stage: Deploy to TEST    (manual trigger / approval gate)
│   ├── Pre-deployment conditions: 1 approver required
│   └── Tasks: (same 5 tasks, different variable group)
│
├── Stage: Deploy to UAT     (manual trigger / approval gate)
│   ├── Pre-deployment conditions: 2 approvers required
│   └── Tasks: (same 5 tasks, different variable group)
│
└── Stage: Deploy to PROD    (manual trigger / approval gate)
    ├── Pre-deployment conditions: 2 approvers required
    └── Tasks: (same 5 tasks, different variable group)
```

### Classic Build Pipeline Layout

Configure a **Build Pipeline** in ADO Server to compile artifacts:

```
Build Pipeline: DW_Build
│
└── Agent Job: Build Artifacts
    ├── Task: MSBuild — build SSDT DACPAC
    │     Solution: Database\DW.sqlproj
    │     MSBuild arguments: /p:Configuration=Release /p:OutputPath=$(Build.ArtifactStagingDirectory)\dacpac
    │
    ├── Task: MSBuild — build SSIS .ispac
    │     Solution: SSIS\DW_ELT.sln
    │     MSBuild arguments: /p:Configuration=Release /p:OutputPath=$(Build.ArtifactStagingDirectory)\ssis
    │
    └── Task: Publish Artifact — DWArtifacts
          Path: $(Build.ArtifactStagingDirectory)
```

### Adding a PowerShell Task in Classic Pipeline

Each deployment step is a **PowerShell** task configured as follows:

```
Task: PowerShell
  Type:          File Path
  Script Path:   $(System.DefaultWorkingDirectory)/DWArtifacts/scripts/Deploy-DACPAC.ps1
  Arguments:     -SqlPackagePath "$(SqlPackagePath)"
                 -DacpacPath "$(System.DefaultWorkingDirectory)/DWArtifacts/dacpac/DW.dacpac"
                 -TargetServer "$(DW_SQL_SERVER)"
                 -TargetDatabase "$(DW_DATABASE)"
  ErrorActionPreference: Stop     (set in Advanced tab, or the script sets it itself)
```

> **All variable references** use `$(VariableName)` syntax in Classic pipelines.
> Link a variable group to each stage via **Stage → Variables → Variable groups**.

### Variable Group Setup

Create one variable group per environment in **ADO Server → Library**:

| Variable | Example (Dev) | Example (Prod) |
|---|---|---|
| `DW_SQL_SERVER` | `DEVDB01\DW` | `PRODDB01\DW` |
| `DW_DATABASE` | `DW_Dev` | `DW_Prod` |
| `SSAS_SERVER` | `DEVDB01\SSAS` | `PRODDB01\SSAS` |
| `SSAS_DATABASE` | `DW_Model_Dev` | `DW_Model_Prod` |
| `PBIRS_URL` | `http://devpbirs/reports` | `http://prodpbirs/reports` |
| `PBIRS_FOLDER` | `/Reports/Dev` | `/Reports` |
| `SSISDB_ENVIRONMENT` | `Dev` | `Prod` |

Mark all credentials as **secret variables** (lock icon). Secret variables are never echoed in logs.

Link groups: **Release Pipeline → Stage → Variables → Variable groups → Link variable group**.
Each stage links its own group (DW-Dev, DW-Test, DW-UAT, DW-Prod).

### Self-Hosted Agent Requirements

The build/release agent must have these tools installed:

| Tool | Used by |
|---|---|
| `SqlPackage.exe` | Deploy-DACPAC.ps1 |
| `ISDeploymentWizard.exe` | Deploy-SsisProject.ps1 |
| Tabular Editor CLI (`te3.exe`) | Deploy-SsasModel.ps1 |
| PowerShell SqlServer module | Process-SsasDatabase.ps1 |
| PowerShell 5.1+ | All scripts |

---

### Optional: YAML Pipeline Equivalent

> Use YAML only if your team has moved to YAML-based pipelines or is using Azure DevOps Services.
> The Classic pipeline above is the primary standard for this on-premises ADO Server environment.

If YAML is preferred, the Classic pipeline above maps directly to:

```yaml
# azure-pipelines.yml — optional, not the primary standard
trigger:
  branches:
    include:
      - main
      - release/*

pool:
  name: 'OnPremAgents'        # Name of your self-hosted agent pool

variables:
  - group: DW-$(Environment)  # DW-Dev, DW-Test, DW-Prod (selected at queue time)
  - name: SqlPackagePath
    value: 'C:\Program Files\Microsoft SQL Server\160\DAC\bin\SqlPackage.exe'
  - name: TabularEditorPath
    value: 'C:\Tools\TabularEditor\te3.exe'

stages:
  - stage: Build
    jobs:
      - job: BuildArtifacts
        steps:
          - task: MSBuild@1
            displayName: 'Build SSDT DACPAC'
            inputs:
              solution: 'Database\DW.sqlproj'
              msbuildArguments: '/p:Configuration=Release /p:OutputPath=$(Build.ArtifactStagingDirectory)\dacpac'

          - task: MSBuild@1
            displayName: 'Build SSIS Project'
            inputs:
              solution: 'SSIS\DW_ELT.sln'
              msbuildArguments: '/p:Configuration=Release /p:OutputPath=$(Build.ArtifactStagingDirectory)\ssis'

          - task: PublishBuildArtifacts@1
            inputs:
              PathtoPublish: '$(Build.ArtifactStagingDirectory)'
              ArtifactName: 'DWArtifacts'

  - stage: DeployDB
    dependsOn: Build
    jobs:
      - deployment: DeployDACPAC
        environment: '$(Environment)'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: PowerShell@2
                  displayName: 'Deploy DACPAC'
                  inputs:
                    filePath: 'scripts/Deploy-DACPAC.ps1'
                    arguments: >
                      -SqlPackagePath "$(SqlPackagePath)"
                      -DacpacPath "$(Pipeline.Workspace)\DWArtifacts\dacpac\DW.dacpac"
                      -TargetServer "$(DW_SQL_SERVER)"
                      -TargetDatabase "$(DW_DATABASE)"

  - stage: DeploySsis
    dependsOn: DeployDB
    jobs:
      - deployment: DeploySsisPackages
        environment: '$(Environment)'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: PowerShell@2
                  displayName: 'Deploy SSIS .ispac'
                  inputs:
                    filePath: 'scripts/Deploy-SsisProject.ps1'
                    arguments: >
                      -IsServerName "$(DW_SQL_SERVER)"
                      -SsisDbFolder "DW"
                      -ProjectName "DW_ELT"
                      -IspacPath "$(Pipeline.Workspace)\DWArtifacts\ssis\DW_ELT.ispac"
                      -EnvironmentName "$(Environment)"

  - stage: DeploySsas
    dependsOn: DeploySsis
    jobs:
      - deployment: DeployTabularModel
        environment: '$(Environment)'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: PowerShell@2
                  displayName: 'Deploy SSAS Tabular Model'
                  inputs:
                    filePath: 'scripts/Deploy-SsasModel.ps1'
                    arguments: >
                      -TabularEditorPath "$(TabularEditorPath)"
                      -ModelPath "$(Pipeline.Workspace)\DWArtifacts\model"
                      -SsasServer "$(SSAS_SERVER)"
                      -DatabaseName "$(SSAS_DATABASE)"

  - stage: DeployPbix
    dependsOn: DeploySsas
    jobs:
      - deployment: DeployPbixReports
        environment: '$(Environment)'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: PowerShell@2
                  displayName: 'Deploy PBIX to PBIRS'
                  inputs:
                    filePath: 'scripts/Deploy-PbixReports.ps1'
                    arguments: >
                      -PbirsUrl "$(PBIRS_URL)"
                      -PbixFolder "$(Pipeline.Workspace)\DWArtifacts\reports"
                      -TargetFolder "$(PBIRS_FOLDER)"
                      -SsasServer "$(SSAS_SERVER)"
                      -SsasDatabase "$(SSAS_DATABASE)"
```

---

## 2. SSDT / DACPAC Deployment

### Script: `scripts/Deploy-DACPAC.ps1`

```powershell
<#
.SYNOPSIS
    Deploy a DACPAC to SQL Server using SqlPackage.exe.
    Idempotent — safe to run multiple times.
#>
[CmdletBinding()]
param(
    [Parameter(Mandatory)] [string] $SqlPackagePath,
    [Parameter(Mandatory)] [string] $DacpacPath,
    [Parameter(Mandatory)] [string] $TargetServer,
    [Parameter(Mandatory)] [string] $TargetDatabase,
    [Parameter(Mandatory)] [string] $SqlUser,
    [Parameter(Mandatory)] [string] $SqlPassword,
    [string] $PublishProfile = '',
    [switch] $WhatIf
)

$ErrorActionPreference = 'Stop'

if (-not (Test-Path $SqlPackagePath)) {
    Write-Error "SqlPackage.exe not found at: $SqlPackagePath"
    exit 1
}
if (-not (Test-Path $DacpacPath)) {
    Write-Error "DACPAC not found at: $DacpacPath"
    exit 1
}

$action = if ($WhatIf) { 'DeployReport' } else { 'Publish' }

$args = @(
    "/Action:$action"
    "/SourceFile:`"$DacpacPath`""
    "/TargetServerName:`"$TargetServer`""
    "/TargetDatabaseName:`"$TargetDatabase`""
    "/TargetUser:`"$SqlUser`""
    "/TargetPassword:`"$SqlPassword`""
    "/p:BlockOnPossibleDataLoss=true"
    "/p:DropObjectsNotInSource=false"     # Never drop objects not in model (safe default)
    "/p:ExcludeObjectTypes=Logins;Users;Permissions;RoleMembership"
)

if ($PublishProfile) {
    $args += "/Profile:`"$PublishProfile`""
}

Write-Host "Deploying DACPAC to [$TargetServer].[$TargetDatabase]..."
$result = & "$SqlPackagePath" @args 2>&1

if ($LASTEXITCODE -ne 0) {
    Write-Error "SqlPackage.exe failed with exit code $LASTEXITCODE`n$result"
    exit 1
}

Write-Host "DACPAC deployment successful."
exit 0
```

### SSDT Project Conventions for Automation

- **Publish profiles** (`*.publish.xml`) per environment in `Database\PublishProfiles\`:
  - `Dev.publish.xml`, `Test.publish.xml`, `Prod.publish.xml`
  - Each profile contains only server/db overrides — no secrets
- **Extended property scripts** in `Scripts\Post-Deployment\ExtendedProperties\` — use upsert pattern
- **Pre/post-deploy scripts** must be idempotent (`IF NOT EXISTS ... / IF EXISTS ...`)
- **SQLCMD variables** for environment-specific values referenced in scripts

---

## 3. SSIS Package Deployment

### Script: `scripts/Deploy-SsisProject.ps1`

```powershell
<#
.SYNOPSIS
    Deploy a compiled SSIS .ispac file to SSISDB on SQL Server.
    Creates the SSISDB catalog folder and project if they do not exist.
    Maps the project to the specified SSISDB environment.
#>
[CmdletBinding()]
param(
    [Parameter(Mandatory)] [string] $IsServerName,
    [Parameter(Mandatory)] [string] $SsisDbFolder,
    [Parameter(Mandatory)] [string] $ProjectName,
    [Parameter(Mandatory)] [string] $IspacPath,
    [Parameter(Mandatory)] [string] $EnvironmentName,
    [string] $SsisDbName = 'SSISDB'
)

$ErrorActionPreference = 'Stop'

if (-not (Test-Path $IspacPath)) {
    Write-Error ".ispac not found: $IspacPath"
    exit 1
}

# Load SSIS management assembly
Add-Type -Path "C:\Windows\Microsoft.NET\assembly\GAC_MSIL\Microsoft.SqlServer.Management.IntegrationServices\*\Microsoft.SqlServer.Management.IntegrationServices.dll" -ErrorAction Stop

$connectionString = "Data Source=$IsServerName;Initial Catalog=master;Integrated Security=SSPI;"
$sqlConnection    = New-Object System.Data.SqlClient.SqlConnection $connectionString
$integrationServices = New-Object Microsoft.SqlServer.Management.IntegrationServices.IntegrationServices $sqlConnection

$catalog = $integrationServices.Catalogs[$SsisDbName]
if (-not $catalog) {
    Write-Error "SSISDB catalog not found on $IsServerName. Create the SSIS catalog first."
    exit 1
}

# Ensure folder exists
$folder = $catalog.Folders[$SsisDbFolder]
if (-not $folder) {
    Write-Host "Creating SSISDB folder: $SsisDbFolder"
    $folder = New-Object Microsoft.SqlServer.Management.IntegrationServices.CatalogFolder(
        $catalog, $SsisDbFolder, "DW ELT packages")
    $folder.Create()
}

# Deploy project
Write-Host "Deploying $ProjectName from $IspacPath..."
$projectBytes = [System.IO.File]::ReadAllBytes($IspacPath)
$folder.DeployProject($ProjectName, $projectBytes) | Out-Null
Write-Host "Project deployed."

# Map to environment reference
$project = $folder.Projects[$ProjectName]
$envRef  = $project.References | Where-Object { $_.EnvironmentName -eq $EnvironmentName }
if (-not $envRef) {
    Write-Host "Adding environment reference: $EnvironmentName"
    $project.References.Add($EnvironmentName, $SsisDbFolder)
    $project.Alter()
}

Write-Host "SSIS project deployment complete."
exit 0
```

### SSISDB Environment Variables

Configure environment variables once per environment (Dev/Test/Prod). Map project parameters to them:

```sql
-- Create environment (run once per environment)
DECLARE @folder_id BIGINT;
SELECT @folder_id = folder_id FROM SSISDB.catalog.folders WHERE name = N'DW';

EXEC SSISDB.catalog.create_environment
    @environment_name = N'Prod',
    @folder_name      = N'DW';

-- Add variables
EXEC SSISDB.catalog.create_environment_variable
    @environment_name = N'Prod',
    @folder_name      = N'DW',
    @variable_name    = N'SourceConnectionString',
    @sensitive        = 1,
    @data_type        = N'String',
    @string_value     = N'Data Source=PRODDB01;Initial Catalog=SourceDB;Integrated Security=SSPI;';

-- Map project parameter to environment variable
EXEC SSISDB.catalog.set_object_parameter_value
    @object_type        = 20,   -- 20 = Project
    @folder_name        = N'DW',
    @project_name       = N'DW_ELT',
    @parameter_name     = N'SourceConnectionString',
    @parameter_value    = N'SourceConnectionString',
    @value_type         = R;    -- R = referenced from environment
```

### SSIS Project Conventions for Automation

- **Project deployment model only** — never package deployment (no `.dtsx` file-based deployment)
- **All connection strings via project parameters** mapped to SSISDB environment variables
- **No hardcoded paths or server names** inside any SSIS package
- **Master_Orchestrator.dtsx** is the only package executed from SQL Agent — one job, one step:

```sql
-- SQL Agent job step (T-SQL type)
DECLARE @execution_id BIGINT;
EXEC SSISDB.catalog.create_execution
    @package_name     = N'Master_Orchestrator.dtsx',
    @project_name     = N'DW_ELT',
    @folder_name      = N'DW',
    @use32bitruntime  = 0,
    @reference_id     = (
        SELECT reference_id FROM SSISDB.catalog.environment_references er
        JOIN SSISDB.catalog.projects p ON er.project_id = p.project_id
        JOIN SSISDB.catalog.folders f  ON p.folder_id   = f.folder_id
        WHERE f.name = N'DW' AND p.name = N'DW_ELT' AND er.environment_name = N'Prod'
    ),
    @execution_id     = @execution_id OUTPUT;

-- Pass load window (set by control table logic or pipeline variable)
EXEC SSISDB.catalog.set_execution_parameter_value
    @execution_id   = @execution_id,
    @object_type    = 50,       -- 50 = execution
    @parameter_name = N'LoadWindowStart',
    @parameter_value = @LoadWindowStart;

EXEC SSISDB.catalog.set_execution_parameter_value
    @execution_id   = @execution_id,
    @object_type    = 50,
    @parameter_name = N'LoadWindowEnd',
    @parameter_value = @LoadWindowEnd;

EXEC SSISDB.catalog.start_execution @execution_id;

-- Wait for completion
DECLARE @status INT = 1;
WHILE @status NOT IN (4, 7, 9)      -- 4=Succeeded, 7=Canceled, 9=Stopped, 5=Failed
BEGIN
    WAITFOR DELAY '00:00:10';
    SELECT @status = [status] FROM SSISDB.catalog.executions
    WHERE execution_id = @execution_id;
END

IF @status <> 4
BEGIN
    RAISERROR('SSIS execution failed. Check SSISDB.catalog.operation_messages.', 16, 1);
END
```

---

## 4. SSAS Tabular Model Deployment

### Environment Naming Convention

Each SSAS environment has a dedicated database with a `_<ENV>` suffix:

| Environment | Database Name | Deployed by |
|---|---|---|
| Developer local | `<ModelName>_DEV` | Developer (Tabular Editor, manual) |
| Shared test | `<ModelName>_TEST` | ADO pipeline |
| UAT | `<ModelName>_UAT` | ADO pipeline |
| Production | `<ModelName>_PROD` | ADO pipeline |

Example: `EAO_Tabular_DEV`, `EAO_Tabular_TEST`, `EAO_Tabular_UAT`, `EAO_Tabular_PROD`

Developers manually deploy only to `_DEV`. The ADO pipeline promotes the TMDL folder from git
through TEST → UAT → PROD. See `ssas-tabular-bp.md` for the full developer workflow.

### Source Control: Always Use TMDL Folder

- The **TMDL folder** is the source of truth in git (not `.bim`)
- Tabular Editor workflow: **Open from folder** → deploy to `_DEV` → edit → **Save as folder** → commit
- TMDL gives granular git diffs: one `.tmdl` file per table/role/relationship

### Option A: Tabular Editor CLI (Recommended)

```powershell
# scripts/Deploy-SsasModel.ps1
[CmdletBinding()]
param(
    [Parameter(Mandatory)] [string] $TabularEditorPath,   # path to te3.exe or te.exe
    [Parameter(Mandatory)] [string] $ModelPath,           # TMDL folder path (preferred) or .bim file
    [Parameter(Mandatory)] [string] $SsasServer,
    [Parameter(Mandatory)] [string] $DatabaseName,        # e.g. EAO_Tabular_TEST
    [switch] $WhatIf
)

$ErrorActionPreference = 'Stop'

if (-not (Test-Path $TabularEditorPath)) {
    Write-Error "Tabular Editor CLI not found: $TabularEditorPath"
    exit 1
}
if (-not (Test-Path $ModelPath)) {
    Write-Error "Model path not found: $ModelPath"
    exit 1
}

if ($WhatIf) {
    Write-Host "WhatIf: Would deploy [$ModelPath] to [$SsasServer/$DatabaseName]"
    exit 0
}

Write-Host "Deploying SSAS Tabular model to $SsasServer / $DatabaseName..."
Write-Host "  Source: $ModelPath"

# te3.exe / TabularEditor.exe syntax:
#   <model_path> -deploy <server> <database> [-O overwrite] [-R retain partitions] [-P deploy roles] -v
$result = & "$TabularEditorPath" "$ModelPath" `
    -deploy "$SsasServer" "$DatabaseName" `
    -O `    # Overwrite existing database
    -R `    # Retain existing partitions (preserves incremental partition data)
    -P `    # Deploy roles
    -v 2>&1

Write-Host $result

if ($LASTEXITCODE -ne 0) {
    Write-Error "Tabular Editor deploy failed with exit code $LASTEXITCODE"
    exit 1
}

Write-Host "SSAS model deployment successful: $DatabaseName"
exit 0
```

**In the ADO pipeline**, call this script per environment:

```yaml
- task: PowerShell@2
  displayName: 'Deploy SSAS Model to TEST'
  inputs:
    filePath: 'scripts/Deploy-SsasModel.ps1'
    arguments: >
      -TabularEditorPath "$(TabularEditorPath)"
      -ModelPath "$(Pipeline.Workspace)/DWArtifacts/ssas/EAO_Tabular"
      -SsasServer "$(SSAS_SERVER)"
      -DatabaseName "$(SSAS_DATABASE)"   # = EAO_Tabular_TEST in DW-Test variable group
```

### Option B: XMLA Script via PowerShell (AMO / SqlServer module)

```powershell
# scripts/Invoke-SsasXmla.ps1
[CmdletBinding()]
param(
    [Parameter(Mandatory)] [string] $SsasServer,
    [Parameter(Mandatory)] [string] $XmlaScriptPath   # path to .xmla file
)

$ErrorActionPreference = 'Stop'

Import-Module SqlServer -ErrorAction Stop

Write-Host "Executing XMLA script on $SsasServer..."
Invoke-ASCmd -Server $SsasServer -InputFile $XmlaScriptPath

if ($LASTEXITCODE -ne 0) {
    Write-Error "XMLA execution failed."
    exit 1
}

Write-Host "XMLA execution complete."
exit 0
```

### SSAS Processing Script

```powershell
# scripts/Process-SsasDatabase.ps1
[CmdletBinding()]
param(
    [Parameter(Mandatory)] [string] $SsasServer,
    [Parameter(Mandatory)] [string] $DatabaseName,
    [ValidateSet('Full','Default','Calculate','Automatic')]
    [string] $ProcessType = 'Default'
)

$ErrorActionPreference = 'Stop'
Import-Module SqlServer -ErrorAction Stop

Write-Host "Processing $DatabaseName on $SsasServer ($ProcessType)..."

$xmla = @"
{
  "refresh": {
    "type": "$($ProcessType.ToLower())",
    "objects": [
      {
        "database": "$DatabaseName"
      }
    ]
  }
}
"@

Invoke-ASCmd -Server $SsasServer -Query $xmla -QueryType Json

if ($LASTEXITCODE -ne 0) {
    Write-Error "SSAS processing failed."
    exit 1
}

Write-Host "SSAS processing complete."
exit 0
```

### SSAS Deployment Conventions

- **Source of truth**: TMDL folder in git — **not** `.bim`, not VS SSAS project
- **Developer workflow**: Open from folder → deploy to `_DEV` → edit → Save as folder → commit. See `ssas-tabular-bp.md` Developer Workflow section.
- **Environment databases**: `_DEV` (developer), `_TEST`, `_UAT`, `_PROD` (pipeline only)
- **Do not deploy from Visual Studio GUI or SSAS Deployment Wizard** — always use Tabular Editor CLI
- **Always pass `-R` (retain partitions)** in the deploy command to preserve incremental partition data from prior process runs
- **Data source connection string** in the model is parameterized — the pipeline variable group supplies the correct `_TEST`/`_UAT`/`_PROD` DW server and database per environment
- **Process after deploy**: separate pipeline step so a failed deploy doesn't leave a partially-processed model
- **Full process on schema change** (new tables/columns); `Default` for data refresh; `Calculate` after relationship-only changes

---

## 5. Power BI Report Server (PBIRS) Deployment

### Script: `scripts/Deploy-PbixReports.ps1`

```powershell
<#
.SYNOPSIS
    Upload .pbix files to Power BI Report Server using the PBIRS REST API.
    Creates the target folder if it does not exist.
    Idempotent — overwrites existing reports.
#>
[CmdletBinding()]
param(
    [Parameter(Mandatory)] [string] $PbirsUrl,         # http://server/reports
    [Parameter(Mandatory)] [string] $PbixFolder,       # local folder containing .pbix files
    [Parameter(Mandatory)] [string] $TargetFolder,     # PBIRS target folder path, e.g. /Reports/Finance
    [Parameter(Mandatory)] [string] $SsasServer,       # SSAS server for data source update
    [Parameter(Mandatory)] [string] $SsasDatabase,     # SSAS database for data source update
    [Parameter(Mandatory)] [string] $PbirsUser,
    [Parameter(Mandatory)] [string] $PbirsPassword
)

$ErrorActionPreference = 'Stop'

$credential = [Convert]::ToBase64String(
    [Text.Encoding]::ASCII.GetBytes("${PbirsUser}:${PbirsPassword}"))
$headers = @{
    Authorization  = "Basic $credential"
    'Content-Type' = 'application/json'
}
$apiBase = "$PbirsUrl/api/v2.0"

# Ensure target folder exists
function Ensure-PbirsFolder {
    param([string] $FolderPath)
    $encoded = [Uri]::EscapeDataString($FolderPath)
    $check = Invoke-RestMethod "$apiBase/Folders?`$filter=Path eq '$FolderPath'" `
        -Headers $headers -Method Get -ErrorAction SilentlyContinue
    if (-not $check.value) {
        $parentPath = ($FolderPath -split '/' | Select-Object -SkipLast 1) -join '/'
        if (-not $parentPath) { $parentPath = '/' }
        $folderName  = $FolderPath.Split('/')[-1]
        $body = @{ Name = $folderName; Path = $FolderPath } | ConvertTo-Json
        Invoke-RestMethod "$apiBase/Folders" -Headers $headers -Method Post -Body $body | Out-Null
        Write-Host "Created PBIRS folder: $FolderPath"
    }
}

Ensure-PbirsFolder -FolderPath $TargetFolder

# Upload each .pbix file
Get-ChildItem -Path $PbixFolder -Filter '*.pbix' | ForEach-Object {
    $pbixFile  = $_.FullName
    $reportName = $_.BaseName
    Write-Host "Uploading $reportName..."

    # Check if report already exists
    $existingUrl = "$apiBase/PowerBIReports?`$filter=Name eq '$reportName' and Path eq '$TargetFolder/$reportName'"
    $existing    = Invoke-RestMethod $existingUrl -Headers $headers -Method Get -ErrorAction SilentlyContinue

    $uploadHeaders = $headers.Clone()
    $uploadHeaders['Content-Type'] = 'application/octet-stream'

    if ($existing.value) {
        # Update existing
        $itemId = $existing.value[0].Id
        Invoke-RestMethod "$apiBase/PowerBIReports($itemId)/Content" `
            -Headers $uploadHeaders -Method Put -InFile $pbixFile | Out-Null
        Write-Host "  Updated: $reportName"
    } else {
        # Create new — POST as multipart/form-data with catalog item wrapper
        $nameHeader  = @{ headers = $headers.Clone() }
        $createBody  = @{
            '@odata.type' = '#Model.PowerBIReport'
            Name          = $reportName
            Path          = "$TargetFolder/$reportName"
        } | ConvertTo-Json

        # Step 1: Create catalog item
        $created = Invoke-RestMethod "$apiBase/PowerBIReports" `
            -Headers $headers -Method Post -Body $createBody
        $itemId  = $created.Id

        # Step 2: Upload content
        Invoke-RestMethod "$apiBase/PowerBIReports($itemId)/Content" `
            -Headers $uploadHeaders -Method Put -InFile $pbixFile | Out-Null
        Write-Host "  Created: $reportName"
    }

    # Update the SSAS data source connection to match environment
    Update-PbirsDataSource -ApiBase $apiBase -Headers $headers `
        -ReportId $itemId -SsasServer $SsasServer -SsasDatabase $SsasDatabase
}

function Update-PbirsDataSource {
    param($ApiBase, $Headers, $ReportId, $SsasServer, $SsasDatabase)
    $dataSources = Invoke-RestMethod "$ApiBase/PowerBIReports($ReportId)/DataSources" `
        -Headers $Headers -Method Get
    foreach ($ds in $dataSources.value) {
        if ($ds.DataSourceType -eq 'AnalysisServices') {
            $updateBody = @{
                DataSourceType      = 'AnalysisServices'
                ConnectionString    = "Data Source=$SsasServer;Initial Catalog=$SsasDatabase"
                CredentialRetrieval = 'Integrated'
            } | ConvertTo-Json
            Invoke-RestMethod "$ApiBase/DataSources($($ds.Id))" `
                -Headers $Headers -Method Patch -Body $updateBody | Out-Null
            Write-Host "  Updated data source to $SsasServer / $SsasDatabase"
        }
    }
}

Write-Host "PBIRS deployment complete."
exit 0
```

### PBIRS Deployment Conventions

- `.pbix` files for **live connection** reports contain no data — only model connection metadata
- Data source connection string is updated post-upload to match environment (see `Update-PbirsDataSource` above)
- **PBIRS folder structure** mirrors environment: `/Reports`, `/Reports/Finance`, `/Reports/Operations`
- **Naming convention**: PBIX file name becomes the report name in PBIRS — use consistent naming
- `ReportingServicesTools` PowerShell module (Microsoft open-source) is an alternative to direct REST API calls for older PBIRS versions

---

## 6. ELT Triggering from ADO Pipeline

For triggering a data load as part of the pipeline (e.g., after deploying SSAS model to refresh data):

### Option A: Trigger SQL Agent Job via T-SQL

```powershell
# scripts/Invoke-SqlAgentJob.ps1
[CmdletBinding()]
param(
    [Parameter(Mandatory)] [string] $SqlServer,
    [Parameter(Mandatory)] [string] $JobName,
    [int] $TimeoutSeconds = 3600
)

$ErrorActionPreference = 'Stop'
Import-Module SqlServer -ErrorAction Stop

# Start the job
Invoke-Sqlcmd -ServerInstance $SqlServer -Query "EXEC msdb.dbo.sp_start_job N'$JobName';"
Write-Host "SQL Agent job '$JobName' started."

# Poll until complete
$elapsed = 0
do {
    Start-Sleep -Seconds 15
    $elapsed += 15
    $status = Invoke-Sqlcmd -ServerInstance $SqlServer -Query @"
SELECT j.name, ja.run_requested_date,
       CASE ja.last_executed_step_id
           WHEN NULL THEN 'Running'
           ELSE 'Complete'
       END AS Status,
       jh.run_status  -- 1=Succeeded, 0=Failed, 3=Cancelled
FROM   msdb.dbo.sysjobs j
JOIN   msdb.dbo.sysjobactivity ja ON j.job_id = ja.job_id
LEFT   JOIN msdb.dbo.sysjobhistory jh
    ON j.job_id = jh.job_id AND jh.step_id = 0
WHERE  j.name = N'$JobName'
  AND  ja.session_id = (SELECT MAX(session_id) FROM msdb.dbo.sysjobactivity);
"@
    Write-Host "Job status: $($status.run_status) (elapsed: ${elapsed}s)"
} while ($status.run_status -notin @(1, 0, 3) -and $elapsed -lt $TimeoutSeconds)

if ($status.run_status -ne 1) {
    Write-Error "SQL Agent job '$JobName' failed or timed out (status: $($status.run_status))"
    exit 1
}

Write-Host "SQL Agent job '$JobName' completed successfully."
exit 0
```

### Option B: Execute SSIS Package Directly

For pipeline-triggered refreshes outside the SQL Agent schedule:

```powershell
# scripts/Invoke-SsisExecution.ps1
[CmdletBinding()]
param(
    [Parameter(Mandatory)] [string] $SqlServer,
    [Parameter(Mandatory)] [string] $FolderName,
    [Parameter(Mandatory)] [string] $ProjectName,
    [Parameter(Mandatory)] [string] $PackageName,
    [Parameter(Mandatory)] [string] $EnvironmentName,
    [datetime] $LoadWindowStart,
    [datetime] $LoadWindowEnd
)

$ErrorActionPreference = 'Stop'
Import-Module SqlServer -ErrorAction Stop

$sql = @"
DECLARE @exec_id BIGINT;
EXEC SSISDB.catalog.create_execution
    @package_name  = N'$PackageName',
    @project_name  = N'$ProjectName',
    @folder_name   = N'$FolderName',
    @reference_id  = (
        SELECT reference_id FROM SSISDB.catalog.environment_references er
        JOIN SSISDB.catalog.projects p ON er.project_id = p.project_id
        JOIN SSISDB.catalog.folders f  ON p.folder_id   = f.folder_id
        WHERE f.name = N'$FolderName' AND p.name = N'$ProjectName'
          AND er.environment_name = N'$EnvironmentName'),
    @execution_id  = @exec_id OUTPUT;
SELECT @exec_id AS ExecutionId;
"@

$result = Invoke-Sqlcmd -ServerInstance $SqlServer -Query $sql
$execId = $result.ExecutionId

Invoke-Sqlcmd -ServerInstance $SqlServer -Query "EXEC SSISDB.catalog.start_execution @execution_id = $execId;"

# Poll
$done = $false
do {
    Start-Sleep -Seconds 10
    $status = Invoke-Sqlcmd -ServerInstance $SqlServer -Query @"
SELECT [status] FROM SSISDB.catalog.executions WHERE execution_id = $execId;
"@
    $done = $status.status -in @(4, 7, 9)
} while (-not $done)

if ($status.status -ne 4) {
    Write-Error "SSIS execution $execId failed (status: $($status.status)). Check SSISDB."
    exit 1
}

Write-Host "SSIS execution completed successfully (id: $execId)."
exit 0
```

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
├── Database/                       # SSDT SQL Server project (.sqlproj)
│   ├── dbo/
│   │   ├── Tables/
│   │   ├── Views/
│   │   └── StoredProcedures/
│   ├── staging/
│   │   └── Tables/
│   └── Scripts/
│       └── Post-Deployment/
│           └── ExtendedProperties/  # sp_addextendedproperty upsert scripts
├── SSIS/                           # SSIS solution (.sln + .dtproj)
│   └── DW_ELT/
│       ├── Master_Orchestrator.dtsx
│       ├── Load_Staging.dtsx
│       ├── Load_Dimensions.dtsx
│       └── Load_Facts.dtsx
├── SSAS/                           # Analysis Services model
│   └── DW_Model/
│       └── Model.bim               # or TMDL folder
├── Reports/                        # Power BI reports
│   └── *.pbix
├── scripts/                        # ADO pipeline PowerShell scripts
│   ├── Deploy-DACPAC.ps1
│   ├── Deploy-SsisProject.ps1
│   ├── Deploy-SsasModel.ps1
│   ├── Process-SsasDatabase.ps1
│   ├── Deploy-PbixReports.ps1
│   ├── Invoke-SqlAgentJob.ps1
│   └── Invoke-SsisExecution.ps1
└── azure-pipelines.yml             # Optional — only if using YAML pipelines
```

---

## 9. Checklist — Is This Deployable via Pipeline?

Use this when generating or reviewing any solution component:

| Item | Required Behavior | ✓ |
|---|---|---|
| No hardcoded server names | All server references via parameters/variables | |
| No hardcoded credentials | Secrets come from ADO variable group (marked secret) | |
| Idempotent scripts | Re-running = update, not duplicate or error | |
| Exit codes set | `exit 0` / `exit 1` for ADO task success/failure detection | |
| SSIS project deployment model | `.ispac` deployment; connection strings in SSISDB env | |
| SSAS deployed via CLI | Tabular Editor CLI (`te.exe`), not VS GUI deploy | |
| SSAS connection parameterized | Data source connection uses environment-specific variables | |
| PBIX data source updated | Post-upload REST call updates SSAS connection per environment | |
| DACPAC safe defaults | `BlockOnPossibleDataLoss=true`, `DropObjectsNotInSource=false` | |
| Extended properties in SSDT | Post-deploy scripts, not ad-hoc SSMS execution | |
| SQL Agent job is the only SSIS trigger | Single step → `Master_Orchestrator.dtsx` only | |
| Pipeline stages ordered | DB → SSIS → SSAS → PBIX — later stages `dependsOn` earlier | |
| No manual approval gates (Dev/Test) | Automated, straight-through for lower environments | |
