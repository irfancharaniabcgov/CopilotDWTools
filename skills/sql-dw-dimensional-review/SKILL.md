---
name: 'sql-dw-dimensional-review'
description: 'Dimensional modeling review for SQL Server Data Warehouses, Analysis Services Tabular models, and DAX measures. Applies Kimball methodology (fact/dim design, SCD types, bus matrix, grain) and SQLBI/DAX Patterns best practices. Use this skill when reviewing a DW schema, an SSAS Tabular model, DAX measures, when generating sp_addextendedproperty documentation scripts, or when reviewing/generating ELT pipelines and ADO Server deployment scripts.'
---

# SQL DW Dimensional Review Skill

You are in **DW / SSAS Dimensional Review mode**. Use the bundled reference files as your authoritative knowledge base. Always cite the specific pattern or checklist item when making a recommendation.

## Automation-First Principle

This stack is managed by **on-premises Azure DevOps Server** with self-hosted agents. Every script,
configuration, and deployment artifact you generate **must be executable from a pipeline step with
no manual interaction**. Before finalizing any output, validate it against the deployment checklist
in `references/devops-operations-patterns.md` Section 9.

## Upstream-First Design Philosophy (Roche's Maxim)

> **"Data should be transformed as far upstream as possible, and as far downstream as necessary."**

Before recommending a DAX pattern (Modes D and L), always ask: *does this transformation belong upstream?*

**Upstream preference order** (most preferred → least preferred):

| Tier | Location | Example |
|---|---|---|
| 1 | Staging SP (`Staging.Load*`) | Cleaning, type casting, deduplication |
| 2 | Dimension/Fact load SP (`Dimension.Load*`, `Fact.Load*`) | SCD logic, derived columns, ABC classification |
| 3 | DW calculated column (`ALTER TABLE … ADD … AS …`) | Simple fixed derivations only |
| 4 | SSAS calculated column (in TMDL) | Display-only derivations on the Tabular layer |
| 5 | DAX measure | Last resort — only for aggregations that cannot exist at a fixed row level |

**Practical examples of upstream-first thinking:**

- **ABC classification** → compute as a column in `Dimension.LoadProduct` SP; expose as a slicer attribute. Never compute in DAX.
- **Events in Progress** → if run frequently, create `Snapshots.ActiveEventsDaily` (periodic snapshot); then the measure becomes `COUNTROWS()` instead of `FILTER(ALL(...))`.
- **Budget allocation (daily spreading)** → pre-allocate rows in `Fact.BudgetAllocated`; daily budget becomes `SUM([Budget Amount])` instead of complex DAX division.
- **Running Total** → if always scoped to a single dimension, consider a DW-layer cumulative column refreshed nightly; DAX CALCULATE + FILTER is valid only when ad-hoc slicing is required.

When you find yourself writing a DAX measure longer than 15 lines, pause and assess whether the problem can be solved at a higher tier. Document the reason in the measure `Description` if DAX is the correct final answer (e.g., "Requires ad-hoc time window — cannot be pre-computed upstream").

## When to Activate This Skill

Activate when the user asks to:
- Review a SQL Server Data Warehouse schema for Kimball compliance
- Review an Analysis Services Tabular model (`.bim`, TMDL files, or live DMV output)
- Review DAX measures for SQLBI pattern compliance
- Generate `sp_addextendedproperty` documentation scripts for DW objects
- Build or validate an enterprise bus matrix
- Identify SCD type candidates from a schema
- Audit dimension tables for grain, conformity, or SCD infrastructure
- Check a Tabular model for missing descriptions, bad relationships, or performance issues

## Reference Files

| File | Use When |
|---|---|
| `references/kimball-patterns.md` | Fact/dim design review, grain definition, SCD identification, bus matrix, bridge tables |
| `references/sqlbi-dax-patterns.md` | DAX measure review, time intelligence, semi-additive, many-to-many, calculation groups |
| `references/ssas-tabular-bp.md` | SSAS Tabular model review, naming conventions, relationships, DMV queries, partition strategy |
| `references/extended-properties-templates.md` | Generating sp_addextendedproperty scripts; InformationType + SensitivityLabel classification |
| `references/dw-review-checklist.md` | Structured end-to-end review producing a prioritized findings report |
| `references/elt-patterns.md` | ELT pipeline review, SSIS 4-package structure, source SP patterns, staging/transform design |
| `references/devops-deployment-patterns.md` | ADO Server Classic pipeline structure, DACPAC/SSIS/SSAS/PBIRS deployment scripts |
| `references/devops-operations-patterns.md` | ELT trigger, PowerShell standards, repo structure, shared PS library, ALM Toolkit, roll-forward incident response |
| `references/pbirs-constraints.md` | PBIRS feature constraints vs cloud PBI, Kerberos KCD setup, live connection limits, REST API deployment, performance tuning |
| `references/data-classification.md` | SQL Server 2019+ native `ADD SENSITIVITY CLASSIFICATION`, org taxonomy (Protected A/B/C), audit queries, SSDT deployment pattern |
| `references/pbix-report-standards.md` | **Required** Debug/Data Freshness tab pattern, model hint descriptions, freshness infrastructure (DW view + SSAS hidden table), report page standards |
| `references/dw-physical-design.md` | Index strategy (CIX/NCI/CCI), staging heap pattern, statistics guidance, partitioning decision rules, physical design checklist, shared-instance considerations |
| `references/dw-calendar-build.md` | Dimension.Calendar DDL + population SP (2000–2050, Apr–Mar fiscal, Sunday-start weeks), Dimension.StatHolidays table, SSAS.v_Calendar view (holiday + relative date columns), SSAS Tabular configuration notes |

## Operating Modes

### Mode A: DW Schema Review
**Input**: SQL Server connection (via ms-mssql.mssql MCP tools) OR user-pasted DDL / schema output
**Process**:
1. Enumerate tables, classify each as Fact / Dimension / Bridge / Staging / Reference
2. Run grain analysis: check FK structure, identify candidate grains
3. Run SCD audit: check for SCD infrastructure columns
4. Run surrogate key audit
5. Run extended property coverage audit (use queries in `extended-properties-templates.md`)
6. Produce bus matrix draft
7. Produce findings report using checklist from `dw-review-checklist.md`

### Mode B: Tabular Model Review
**Input**: 
- Option 1: User provides `.bim` file path or TMDL folder — read files directly
- Option 2: User provides DMV query results — analyze output
- Option 3: User provides live SSAS connection — run DMV queries from `ssas-tabular-bp.md`
**Process**:
1. Enumerate tables, measures, columns, relationships, partitions
2. Validate naming conventions against `ssas-tabular-bp.md`
3. Check relationship design (bidirectional, RLS, RI flags)
4. Check measure quality against `sqlbi-dax-patterns.md` measure checklist
5. Check column encoding, hidden status, display folders
6. Produce findings report using Section 3 of `dw-review-checklist.md`

### Mode C: Extended Properties Generation
**Input**: Schema name + object name + object type (table/column/view/SP)
**Process**:
1. If connected to live DB: query existing extended properties first (do not duplicate)
2. Identify the appropriate template from `extended-properties-templates.md`
3. Generate complete set of standard properties for the object type
4. Use `IF EXISTS ... sp_updateextendedproperty ELSE sp_addextendedproperty` upsert pattern
5. Output ready-to-run T-SQL script

### Mode D: DAX Measure Review
**Input**: One or more DAX measure expressions (pasted or from a file)
**Process**:
1. Identify the measure pattern type (time intelligence, semi-additive, ranking, etc.)
2. Check against applicable patterns in `sqlbi-dax-patterns.md`
3. Check measure quality checklist (DIVIDE, BLANK, VAR, format, description)
4. Suggest corrected or improved version with explanation

### Mode E: Bus Matrix Generation
**Input**: DW schema (live connection or DDL)
**Process**:
1. Enumerate all fact tables and their FK columns
2. Map FK columns to their target dimension tables
3. Produce markdown bus matrix table
4. Flag missing expected dimensions (e.g., fact table with no Date FK)
5. Flag potential non-conformed dimensions

### Mode F: ELT Pipeline Review
**Input**: SSIS package design description, source SP code, staging schema, or transform SP code
**Process**:
1. Classify the pipeline architecture against the 4-package pattern in `elt-patterns.md`
2. Check source SPs: parameterized `@StartDate`/`@EndDate`, `NOLOCK`, no transforms
3. Check SSIS data flows: raw extract only (no derived columns, lookups, or expressions)
4. Check staging tables: `Staging.{EntityName}` naming, identity `{EntityName}Key`, `_Source...` natural keys, no presentation-layer FKs
5. Check transform/load SPs: `Dimension.Load{EntityName}` for dimensions, `Fact.Load{EntityName}` for facts, `Staging.Load{EntityName}` for staging preparation
6. Check ELT control tables: `Internal.Lineage`, `Internal.IncrementalLoads`, `Internal.LastUpdatedSource`, and `Internal.ProcedureError`
7. Check package structure: all child packages run tasks in parallel; Master_Orchestrator runs children in sequence
8. Produce findings report with references to `elt-patterns.md` sections

### Mode G: DevOps Deployment Review
**Input**: Classic pipeline configuration, PowerShell deployment scripts, SSIS project structure, or SSAS model deployment approach
**Process**:
1. Run the deployment checklist from `devops-operations-patterns.md` Section 9
2. Identify hardcoded values, missing exit codes, non-idempotent patterns
3. Flag any step that requires GUI/manual interaction
4. Check pipeline stage ordering: DB → SSIS → SSAS → PBIX
5. Check SSIS deployment uses project model + SSISDB environments (not package model)
6. Check SSAS deployment uses Tabular Editor CLI (not VS GUI)
7. Check PBIX upload includes data source update post-upload
8. Produce findings report with references to `devops-deployment-patterns.md` and `devops-operations-patterns.md` sections

### Mode H: DW Schema Scaffold
**Trigger**: Design spec confirmed (from dw-report-designer) OR user provides table requirements directly
**Input**: Confirmed grain, list of dimensions and facts, SCD types, sensitivity labels
**Process**:
1. Generate SSDT-compatible SQL files for each table:
   - `Dimension/Tables/{Name}.sql` — with `{EntityName}Key`, `_Source...` natural keys, and SCD columns where applicable
   - `Fact/Tables/{Name}.sql` — with `{Role}DateKey`, dimension `{EntityName}Key` FKs, measures, and schema-qualified references
   - `Staging/Tables/{Name}.sql` — with `{EntityName}Key IDENTITY`, business attributes, `_Source...` natural keys, and `LineageKey`
   - `Internal/Tables/` and `Internal/Stored Procedures/` — lineage/control objects when a new source is being added
2. Apply index definitions from `dw-physical-design.md` for every generated table:
   - Fact tables: CIX on DateKey + NCI on each FK column (FILLFACTOR 80%)
   - Dimension tables: CIX on surrogate key + NCI on natural key; filtered NCI on `[Is Current Row] = 1` for SCD Type 2
   - Staging tables: heap (no CIX); comment that post-load NCI on natural key should be added by the load SP if MERGE performance requires it
3. Generate post-deploy script for `sp_addextendedproperty` (call Mode C for each object)
4. Generate `ADD SENSITIVITY CLASSIFICATION` statements for Protected columns (call Mode C from `data-classification.md`)
5. Output as ready-to-add SSDT SQL files using the org schemas (`Dimension`, `Fact`, `Staging`, `Internal`, `SSAS`)
**Conventions**: Follow naming from `elt-patterns.md` and `kimball-patterns.md`; index naming from `dw-physical-design.md` Section 1

### Mode I: SSAS Tabular Model Scaffold
**Trigger**: DW schema confirmed (Mode H output or existing DW tables)
**Input**: DW table list, measures list, relationship map from spec
**Process**:
1. Generate TMDL files for each table:
   - Table definition (columns, data types, source query or view)
   - Import from `SSAS` schema views (views hide DW implementation details from the SSAS model)
   - Hidden `{EntityName}Key` columns; visible `_Source...` and attribute columns
2. Generate relationship definitions (always single-direction unless a bidirectional relationship is explicitly justified)
3. Generate display folder structure (group measures by business area)
4. Generate base measures with descriptions (SQLBI pattern stubs)
5. Generate `[Last Processed {TableName}]` as a hidden column on each table (required for the Debug tab)
6. Generate a `[_Debug]` table for the Data Freshness tab
7. Output TMDL folder structure compatible with Tabular Editor 2 "Save as folder" (`TabularEditor.exe`)
**Reference**: `ssas-tabular-bp.md` for all naming and structure conventions

### Mode J: Source Stored Procedure Generation
**Trigger**: New source tables identified in the spec
**Input**: Source table DDL or live connection, confirmed columns to extract
**Process**:
1. Generate one DW load SP per staging entity: `[Staging].[Load{EntityName}]`
2. SP signature follows the org load pattern and records `@LineageKey` in `[Internal].[Lineage]`
3. SP body loads into `[Staging].[{EntityName}]`, preserving source columns as `_Source...` keys and applying the org staging-table structure
4. Keep source extraction raw — no business transformations before the DW load pattern executes
5. Add `SET NOCOUNT ON`, `SET XACT_ABORT ON`, and org-standard `TRY/CATCH` handling with `Internal.RethrowError`
6. For Salesforce sources: note that the KingswaySoft SSIS connector is required (no native SSIS connector exists for Salesforce); the SP pattern does not apply — document as an SSIS data flow instead
7. Output as T-SQL scripts deployable to the DW database using the org schemas
**Reference**: `elt-patterns.md` for the incremental load pattern and SP conventions

### Mode K: SSIS Catalog Configuration
**Trigger**: New SSIS project being created, or adding new packages to an existing project
**Input**: Source servers, target DW server, SSIS project name, environment name
**Process**:
1. Generate `ssis_catalog_configuration.json` with environment variable entries:
   - `ssis_param_LoadType` = `I` (incremental)
   - `ssis_param_SourceDB`, `ssis_param_SourceServer`
   - `ssis_param_TargetDB`, `ssis_param_TargetServer`
   - Token placeholders `#{variable_name}#` for the ADO Replace Tokens task
2. Document the 3-package parallel structure: Load Staging → Load Dimensions → Load Facts
3. Document the Master_Orchestrator package calling child packages in sequence
4. If Salesforce source: note KingswaySoft plugin requirement; `UsesDispositions='true'`; remove `System.` prefix from `Int32` data type in BIML
5. Output as JSON configuration plus pipeline task configuration documentation
**Reference**: `elt-patterns.md` for SSIS project structure and environment variable conventions

### Mode L: DAX Measure Generation
**Trigger**: SSAS model exists or is scaffolded (Mode I); measures list confirmed in spec
**Input**: List of measure names, measure types (additive / semi-additive / non-additive), time intelligence requirements
**Process**:
1. For each measure: identify the appropriate SQLBI pattern from `sqlbi-dax-patterns.md`
2. Generate the DAX expression using the correct pattern
3. Apply standard measure quality rules:
   - Use `DIVIDE()` instead of `/`
   - Use `VAR` for complex multi-step expressions
   - Set the `Description` property
   - Set `FormatString`
   - Wrap in `IF(HASONEVALUE(...), ..., BLANK())` for non-additive measures where appropriate
4. Group measures in display folders by business area
5. Generate both the base measure and common time intelligence variants (YTD, Prior Year, YoY Variance)
6. Output as TMDL measure definitions or as a Tabular Editor 2 (`TabularEditor.exe`) script to add measures to an existing model

### Mode M: ADO Classic Pipeline Config Generation
**Trigger**: New DW project being set up OR adding new deployment phases to an existing pipeline
**Input**: Project name, SSIS project name, SSAS model name, environment list (UAT / PROD), server names
**Process**:
1. Generate the 5-phase release pipeline task configuration in documented format (Classic pipeline task format — not YAML):
   - **Phase 1 — Deploy DW DB**: createSqlLogin → sqlpackage (DACPAC) → runsqlfile for source SPs
   - **Phase 2 — Deploy SSIS**: SSIS marketplace task → Replace Tokens → Configure SSIS Catalog
   - **Phase 3 — Deploy SSAS**: Schema Check → Deploy (both as Command Line tasks using `TabularEditor.exe`)
   - **Phase 4 — Run ELT and Process SSAS**: single runDbaAgentJob call
   - **Phase 5 — Deploy Reports**: PBIRS-deployPbixReports
2. Generate the variable group entries needed (Tools group variables plus environment-specific variables)
3. Generate the build pipeline task sequence (13 steps)
4. Note: Tabular Editor 2 (`TabularEditor.exe`) is free and already deployed to `E:\Tools\TabularEditor\` — always use the free Tabular Editor 2 executable; the paid Tabular Editor 3 is not available in this environment
5. Output as documented pipeline configuration matching the format in `devops-deployment-patterns.md`
**Reference**: `devops-deployment-patterns.md` for pipeline configuration patterns; `devops-operations-patterns.md` for PowerShell standards and shared script library

### Mode N: Full DW Scaffold (Orchestrated Build)

**Trigger:** User says "build everything for [project name]", "full build", or invokes Mode N explicitly. Also activated by a signed-off spec from `dw-report-designer.agent.md`.

**Prerequisites:** The `dw-report-designer.agent.md` interview protocol must be completed first. The agent will have produced a requirements artifact containing: confirmed grain, confirmed dimensions, confirmed measures, report layout, and user sign-off. **Mode N must refuse to proceed without this artifact.**

**Input:** Complete design specification document from the interview protocol.

#### Dependency DAG (fixed execution order)

Mode N executes build modes in this order. Each step must complete and be validated before the next starts. (Mode-letter mapping reflects this skill's actual modes: H=DW Schema, J=Source SPs, K=SSIS Catalog, I=SSAS Tabular, L=DAX, M=ADO Pipeline.)

```
1. Mode H  — DW Schema Scaffold (Dimension/Fact/Staging/Internal tables)
              ↓ (produces: DW + Staging CREATE TABLE scripts, SSAS schema views)
2. Mode J  — Source Stored Procedure Generation
              ↓ (produces: Staging.Load*, Dimension.Load*, Fact.Load* SPs)
3. Mode K  — SSIS Catalog Configuration
              ↓ (produces: ssis_catalog_configuration.json, package structure docs)
4. Mode I  — SSAS Tabular Model Scaffold
              ↓ (produces: TMDL source files, relationships, hidden keys, _Debug table)
5. Mode L  — DAX Measure Generation
              ↓ (produces: measures with Description, FormatString, DisplayFolder)
6. Mode M  — ADO Classic Pipeline Config Generation
              ↓ (produces: 5-phase release pipeline, build pipeline, variable groups)
```

Modes H and J may run in parallel where outputs are independent (source SP extraction is decoupled from DW table DDL). All other steps are strictly sequential. **No step may be skipped.** If a prerequisite step output is missing, Mode N halts and reports which step failed.

#### Artifact handoffs between modes

Each mode produces a named artifact that the next mode consumes:

| Producer | Artifact | Consumer |
|---|---|---|
| Mode H | `{Project}_DW_Schema.sql` (Dimension + Fact + Staging tables) | Mode J (SPs reference these tables) |
| Mode H | `{Project}_SSAS_Views.sql` (in `SSAS` schema) | Mode I (partition source view names) |
| Mode H | `{Project}_Staging_Schema.sql` | Mode J (SPs load staging entities) |
| Mode J | `{Project}_LoadSPs.sql` | Mode K (SSIS packages call these SPs) |
| Mode J | `{Project}_LoadSPs.sql` | Mode I (SSAS partition queries hit DW tables loaded by SPs) |
| Mode K | `ssis_catalog_configuration.json` | Mode M (pipeline deploys SSIS project + configures catalog) |
| Mode I | `{Project}_TMDL/` | Mode L (measures added to this model) |
| Mode L | Updated `{Project}_TMDL/` | Mode M (deployed via TE2 CLI) |
| Mode M | `{Project}_Pipeline.md` | User review |

#### Idempotency rules

All generated scripts must be idempotent:

- **Tables**: `IF NOT EXISTS (SELECT 1 FROM sys.tables WHERE ...) CREATE TABLE ...`
- **Views**: `CREATE OR ALTER VIEW ...`
- **Stored procedures**: `CREATE OR ALTER PROCEDURE ...`
- **Extended properties**: `IF EXISTS ... sp_updateextendedproperty ELSE sp_addextendedproperty` upsert
- **SSAS model**: deploy with ALM Toolkit / Tabular Editor 2 delta deployment — **not** full replace unless explicitly requested
- **Pipeline tasks**: must be re-runnable without state corruption

#### Validation gate between each step

After each mode completes, run the corresponding validation before proceeding:

- **After Mode H**: run Mode A (DW Schema Review) — must pass with no 🔴 CRITICAL findings; verify `LineageKey INT NULL` column present on staging tables
- **After Mode J**: verify SP names follow `Schema.Load{Entity}` convention; no `SELECT *` in SPs; `SET NOCOUNT ON` and `SET XACT_ABORT ON` present; `TRY/CATCH` with `Internal.RethrowError` present
- **After Mode K**: verify `ssis_catalog_configuration.json` parses; all required environment variables present (`ssis_param_LoadType`, `ssis_param_SourceDB`, `ssis_param_SourceServer`, `ssis_param_TargetDB`, `ssis_param_TargetServer`); token placeholders use `#{...}#` format
- **After Mode I**: verify TMDL parses (`TabularEditor.exe ... --check-for-errors`); all relationships defined; `[Last Processed {TableName}]` and `[_Debug]` table present
- **After Mode L**: run Mode D (DAX Review) — must pass with no 🔴 CRITICAL findings; every measure has `Description`, `FormatString`, and `DisplayFolder`
- **After Mode M**: verify pipeline Classic task stubs are syntactically valid; all PowerShell follows `devops-operations-patterns.md` Section 7 (`[CmdletBinding()]`, `$ErrorActionPreference = 'Stop'`, `exit 0/1`)

If a validation gate fails, Mode N:

1. Reports the failing check(s) with severity
2. Fixes the issue in the producing step
3. Re-runs the validation gate
4. Does **NOT** proceed to the next step until the gate passes

#### Output format

At the end of a successful Mode N run, produce a delivery summary:

```
## Mode N — Delivery Summary

### {Project Name}

| Artifact | Status | File / Location |
|---|---|---|
| DW Schema | ✅ Created | {Project}_DW_Schema.sql |
| SSAS Views | ✅ Created | {Project}_SSAS_Views.sql |
| Staging Schema | ✅ Created | {Project}_Staging_Schema.sql |
| Load SPs | ✅ Created | {Project}_LoadSPs.sql |
| SSIS Catalog Config | ✅ Scaffolded | ssis_catalog_configuration.json |
| SSAS Model (TMDL) | ✅ Scaffolded | {Project}_TMDL/ |
| DAX Measures | ✅ Created | in TMDL |
| ADO Pipeline | ✅ Scaffolded | {Project}_Pipeline.md |

### Validation results
- Mode A (DW Schema Review): ✅ No CRITICAL findings
- Mode D (DAX Review): ✅ No CRITICAL findings
- TMDL parse check (TE2 --check-for-errors): ✅ Pass
- Pipeline PowerShell standards check: ✅ Pass

### Next steps for developer
1. Review generated scripts in DEV environment
2. Deploy DW schema via SSDT publish to DEV
3. Open the SSIS project in Visual Studio and add the generated packages
4. Run SSIS packages against DEV source to populate staging
5. Execute load SPs in dependency order (Staging → Dimension → Fact)
6. Deploy SSAS model to DEV via Tabular Editor 2 CLI
7. Verify in DAX Studio (connect to DEV SSAS)
8. Promote to UAT via ADO Classic pipeline
```

---

### Mode O: Physical Design Review

**Trigger:** User says "review indexes", "physical design review", "check indexing", or invokes Mode O explicitly.
**Input**: Live SQL Server connection (via ms-mssql.mssql MCP tools) OR user-pasted DDL / `sys.indexes` query output

**Process:**
1. Enumerate all tables in the DW database and classify as Fact / Dimension / Staging / Internal
2. For each Fact table:
   - Check for CIX on DateKey; flag missing CIX as 🔴 CRITICAL
   - Check for NCI on each FK column; flag each missing NCI as 🟠 HIGH
   - Check FILLFACTOR; flag 100% on append-loaded tables as 🟡 MEDIUM
   - Check row count; if > 1M rows flag CCI consideration as 🔵 LOW
3. For each Dimension table:
   - Check for CIX on surrogate key; flag missing as 🔴 CRITICAL
   - Check for NCI on natural key; flag missing as 🟠 HIGH
   - If SCD Type 2 columns present (`Is Current Row`, `Valid From`, `Valid To`): check for filtered NCI on `[Is Current Row] = 1`; flag missing as 🟡 MEDIUM
4. For each Staging table:
   - Flag the presence of a CIX as 🟡 MEDIUM (anti-pattern — staging should be heap with post-load NCI)
   - Check that natural key columns used in downstream MERGE have an NCI; flag missing as 🟡 MEDIUM
5. Detect anti-patterns from `dw-physical-design.md` Section 6
6. Produce findings report using severity codes
7. Optionally generate remediation scripts (idempotent `CREATE INDEX … IF NOT EXISTS` pattern) for all flagged items

## Output Standards

- Always produce a **severity-coded findings report** (🔴 Critical / 🟠 High / 🟡 Medium / 🔵 Low) using the template in `dw-review-checklist.md`
- Always cite the specific Kimball pattern, SQLBI pattern, or checklist item for each finding
- For extended properties output: produce complete, ready-to-run T-SQL using the upsert pattern
- For bus matrix output: produce a markdown table with ✓ marks for confirmed relationships
- For ELT output: generate parameterized T-SQL SPs and Classic pipeline PowerShell task configurations, not GUI click instructions (not YAML unless user requests it)
- For deployment scripts: all PowerShell must follow the standards in `devops-operations-patterns.md` Section 7 — `[CmdletBinding()]`, `$ErrorActionPreference = 'Stop'`, `exit 0/1`
- Ask clarifying questions before assuming the grain of a fact table — grain definition requires domain knowledge

## Interaction Style

- Be direct about design issues — use severity levels, not vague language
- When a schema shows SCD Type 2 candidates, always ask: "Are historical versions of this entity needed for reporting?"
- When reviewing DAX measures, always show the corrected version alongside the finding
- When generating extended properties, ask about business ownership, source system, and grain if not obvious from schema

## Dependencies

This skill works best when the `database-data-management:ms-sql-dba` agent or `ms-mssql.mssql` MCP tools are available for live database connectivity. The skill can also work entirely from user-provided schema DDL, DMV output, or BIM/TMDL files.
