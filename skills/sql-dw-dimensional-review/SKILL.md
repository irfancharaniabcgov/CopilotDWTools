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
in `references/devops-deployment-patterns.md` Section 9.

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
| `references/devops-deployment-patterns.md` | ADO Server Classic pipeline structure, DACPAC/SSIS/SSAS/PBIRS deployment scripts, PowerShell standards |
| `references/pbirs-constraints.md` | PBIRS feature constraints vs cloud PBI, Kerberos KCD setup, live connection limits, REST API deployment, performance tuning |
| `references/data-classification.md` | SQL Server 2019+ native `ADD SENSITIVITY CLASSIFICATION`, org taxonomy (Protected A/B/C), audit queries, SSDT deployment pattern |
| `references/pbix-report-standards.md` | **Required** Debug/Data Freshness tab pattern, model hint descriptions, freshness infrastructure (DW view + SSAS hidden table), report page standards |

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
1. Run the deployment checklist from `devops-deployment-patterns.md` Section 9
2. Identify hardcoded values, missing exit codes, non-idempotent patterns
3. Flag any step that requires GUI/manual interaction
4. Check pipeline stage ordering: DB → SSIS → SSAS → PBIX
5. Check SSIS deployment uses project model + SSISDB environments (not package model)
6. Check SSAS deployment uses Tabular Editor CLI (not VS GUI)
7. Check PBIX upload includes data source update post-upload
8. Produce findings report with references to `devops-deployment-patterns.md` sections

### Mode H: DW Schema Scaffold
**Trigger**: Design spec confirmed (from dw-report-designer) OR user provides table requirements directly
**Input**: Confirmed grain, list of dimensions and facts, SCD types, sensitivity labels
**Process**:
1. Generate SSDT-compatible SQL files for each table:
   - `Dimension/Tables/{Name}.sql` — with `{EntityName}Key`, `_Source...` natural keys, and SCD columns where applicable
   - `Fact/Tables/{Name}.sql` — with `{Role}DateKey`, dimension `{EntityName}Key` FKs, measures, and schema-qualified references
   - `Staging/Tables/{Name}.sql` — with `{EntityName}Key IDENTITY`, business attributes, `_Source...` natural keys, and `LineageKey`
   - `Internal/Tables/` and `Internal/Stored Procedures/` — lineage/control objects when a new source is being added
2. Generate post-deploy script for `sp_addextendedproperty` (call Mode C for each object)
3. Generate `ADD SENSITIVITY CLASSIFICATION` statements for Protected columns (call Mode C from `data-classification.md`)
4. Output as ready-to-add SSDT SQL files using the org schemas (`Dimension`, `Fact`, `Staging`, `Internal`, `SSAS`)
**Conventions**: Follow naming from `elt-patterns.md` and `kimball-patterns.md`

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
**Reference**: `devops-deployment-patterns.md` for all pipeline configuration patterns and PowerShell standards

### Mode N: Full DW Scaffold (Orchestrated Build)
**Trigger**: Signed-off spec from dw-report-designer OR user requests end-to-end generation
**Input**: Complete design specification document
**Process**: Execute Modes H, I, J, K, L, M in dependency order:
1. **Mode H + Mode J in parallel** — DW schema SQL files and source SPs are independent of each other
2. **Mode I after Mode H** — SSAS TMDL scaffold needs DW tables defined first
3. **Mode L after Mode I** — DAX measures need the SSAS model structure
4. **Mode K after Mode H** — SSIS catalog configuration needs the DW server and database names
5. **Mode M last** — ADO pipeline configuration needs all artifact names from all previous modes

After all modes complete, produce a **build summary** listing:
- Every generated file with its target path
- Next manual steps required (for example: "Open the SSIS project in Visual Studio and add the generated packages to the project", "Review the generated TMDL in Tabular Editor 2 before deploying to UAT")

## Output Standards

- Always produce a **severity-coded findings report** (🔴 Critical / 🟠 High / 🟡 Medium / 🔵 Low) using the template in `dw-review-checklist.md`
- Always cite the specific Kimball pattern, SQLBI pattern, or checklist item for each finding
- For extended properties output: produce complete, ready-to-run T-SQL using the upsert pattern
- For bus matrix output: produce a markdown table with ✓ marks for confirmed relationships
- For ELT output: generate parameterized T-SQL SPs and Classic pipeline PowerShell task configurations, not GUI click instructions (not YAML unless user requests it)
- For deployment scripts: all PowerShell must follow the standards in `devops-deployment-patterns.md` Section 7 — `[CmdletBinding()]`, `$ErrorActionPreference = 'Stop'`, `exit 0/1`
- Ask clarifying questions before assuming the grain of a fact table — grain definition requires domain knowledge

## Interaction Style

- Be direct about design issues — use severity levels, not vague language
- When a schema shows SCD Type 2 candidates, always ask: "Are historical versions of this entity needed for reporting?"
- When reviewing DAX measures, always show the corrected version alongside the finding
- When generating extended properties, ask about business ownership, source system, and grain if not obvious from schema

## Dependencies

This skill works best when the `database-data-management:ms-sql-dba` agent or `ms-mssql.mssql` MCP tools are available for live database connectivity. The skill can also work entirely from user-provided schema DDL, DMV output, or BIM/TMDL files.
