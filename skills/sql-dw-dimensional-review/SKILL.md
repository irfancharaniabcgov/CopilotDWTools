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
- Analyse a source database to identify fact/dimension candidates before DW design begins
- Design a new DW subject area where data exists in source systems but not yet in the DW
- Scaffold a new line-of-business DW from a confirmed design specification

## Reference Files

| File | Use When |
|---|---|
| `references/kimball-patterns.md` | Fact/dim design review, grain definition, SCD identification, bus matrix, bridge tables. **SQLBI authority applies to semantic layer — use this for physical DW design only.** |
| `references/kimball-advanced-patterns.md` | Data Vault bridging, late-arriving facts, snapshot DDL, advanced physical design. **SQLBI authority applies to semantic layer.** |
| `references/sqlbi-dax-patterns.md` | DAX measure review, time intelligence, semi-additive, many-to-many, calculation groups. **Primary authority for all DAX.** |
| `references/sqlbi-dax-patterns-advanced.md` | Situational DAX: paginated report parameters, M2M TREATAS, disconnected tables, aggregations |
| `references/sqlbi-dax-patterns-niche.md` | Rare patterns: currency conversion, survey/weighted average. Use sparingly. |
| `references/ssas-tabular-bp.md` | SSAS Tabular model review, naming conventions, relationships, DMV queries, partition strategy, BPA rules |
| `references/dax-style-guide.md` | DAX coding standard (naming, formatting, VAR/RETURN, filter functions, upstream-first principle) |
| `references/dax-studio-workflow.md` | DAX Studio: Server Timings, VertiPaq Analyzer, benchmarking, storage engine query analysis |
| `references/extended-properties-templates.md` | Generating sp_addextendedproperty scripts; InformationType + SensitivityLabel classification |
| `references/dw-review-checklist.md` | Structured end-to-end review producing a prioritized findings report |
| `references/dw-validation-patterns.md` | T-SQL validation queries: orphan facts, unknown members, calendar completeness, reconciliation |
| `references/dw-physical-design.md` | Index strategy (CIX/NCI/CCI), staging heap pattern, statistics guidance, partitioning (DATE type), physical design checklist |
| `references/dw-calendar-build.md` | Dimension.Calendar DDL + population SP (2000–2050, Apr–Mar fiscal), StatHolidays table, SSAS.v_Calendar view, sentinel design |
| `references/elt-patterns.md` | ELT pipeline review, SSIS 4-package structure, source SP patterns, staging/transform design |
| `references/ssdt-project-structure.md` | SSDT project layout, DACPAC publish profiles, pre/post-deploy scripts, database project conventions |
| `references/ssisdb-catalog-config.md` | SSISDB topology, environment variables, JSON config format, catalog configuration |
| `references/ssas-deployment-processing.md` | TE2 deploy commands, processing modes (ProcessFull/ProcessAdd/ProcessUpdate), SQL Agent job pattern |
| `references/tabular-editor-2-automation.md` | TE2 CLI flags, C# scripts library (HideKeyColumns, SetDisplayFolders, ApplyTitleCaseAliases), BPA rule JSON |
| `references/devops-deployment-patterns.md` | ADO Server Classic pipeline structure, DACPAC/SSIS/SSAS/PBIRS deployment scripts |
| `references/devops-operations-patterns.md` | ELT trigger, PowerShell standards, repo structure, shared PS library, ALM Toolkit, roll-forward incident response |
| `references/security-implementation.md` | PBIRS→SSAS→DW connection chain, SQL least-privilege grants, SSAS Tabular roles (fixed + TREATAS dynamic RLS), OLS, PBIRS folder permissions |
| `references/pbirs-constraints.md` | PBIRS feature constraints vs cloud PBI, Kerberos KCD setup, live connection limits, REST API deployment, performance tuning |
| `references/pbix-report-standards.md` | Debug/Data Freshness tab pattern, model hint descriptions, freshness infrastructure (DW view + SSAS hidden table), report page standards |
| `references/source-system-analysis.md` | Mode P source discovery: T-SQL query library (Q1–Q10: table inventory, date/status column detection, PK/FK map, NULL rate checks, duplicate PK check, date range profiling, CDC/CT detection, cardinality profiling), classification heuristics (fact/dim/bridge/ignore with zero-row fallback and Priority 4/5 clarification), Source Entity Map output format, grain proposal pattern |
| `references/data-classification.md` | SQL Server 2019+ native `ADD SENSITIVITY CLASSIFICATION`, org taxonomy (Protected A/B/C), audit queries, SSDT deployment |

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
4. Check role definitions against `security-implementation.md` Section 3 patterns (fixed vs dynamic, AD group membership, OLS)
5. Check measure quality against `sqlbi-dax-patterns.md` measure checklist
6. Check column encoding, hidden status, display folders
7. Produce findings report using Section 3 of `dw-review-checklist.md`

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
**Trigger**: User asks for a bus matrix or enterprise integration map. Also automatically invoked by `dw-report-designer.agent.md` after Phase 6 (Dimensions) sign-off — the agent synthesises the bus matrix from interview answers rather than querying a live schema; Mode E's SQL query is used when augmenting against an existing DW.

**Input (design-time)**: Confirmed fact tables + grains (Phase 3) and confirmed dimensions (Phase 6) from the `dw-report-designer` spec.  
**Input (existing DW)**: Live SQL Server connection.

**Process**:
1. Enumerate all fact tables and their FK columns (from Phase 6 or live schema query)
2. Map FK columns to their target dimension tables
3. Produce markdown bus matrix table — format from `references/kimball-patterns.md §Enterprise Bus Matrix`:
   - Rows = fact tables; Columns = Grain then dimensions (conformed dimensions first, **bold**; local dimensions last, labelled)
   - ✓ where FK exists; blank where it does not
4. Flag facts with no Calendar FK — 🔴 Critical
5. Flag potential non-conformed dimensions (dimension used by only one fact — confirm whether it should be conformed) — 🟠 High
6. For greenfield: present bus matrix to user for sign-off before any DDL is generated

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
1. Generate SSDT-compatible SQL files for each table — one `.sql` file per object using the flat repo layout defined in `devops-operations-patterns.md` Section 8:
   - `DW/Dimension/[TableName].sql` — with `[{EntityName} Key]`, `_Source...` natural keys, and SCD columns where applicable
   - `DW/Fact/[TableName].sql` — with `[{Role} Date Key]`, dimension `[{EntityName} Key]` FKs, measures, and schema-qualified references
   - `DW/Staging/[TableName].sql` — with `[{EntityName} Key] IDENTITY`, business attributes, `_Source...` natural keys, and `[Lineage Key]`
   - `DW/Internal/[TableName].sql` and `DW/Internal/[ProcedureName].sql` — lineage/control objects when a new source is being added
   - `DW/SSAS/[ViewName].sql` — one file per SSAS schema view
2. Apply index definitions from `dw-physical-design.md` for every generated table:
   - Fact tables: CIX on `[Date Key]` + NCI on each FK column (FILLFACTOR 80%)
   - Dimension tables: CIX on surrogate key + NCI on natural key; filtered NCI on `[Is Current Row] = 1` for SCD Type 2
   - Staging tables: heap (no CIX); comment that post-load NCI on natural key should be added by the load SP if MERGE performance requires it
3. Generate post-deploy script for `sp_addextendedproperty` (call Mode C for each object)
4. Generate `ADD SENSITIVITY CLASSIFICATION` statements for Protected columns (call Mode C from `data-classification.md`)
5. Output as ready-to-add SSDT SQL files using the org schemas (`Dimension`, `Fact`, `Staging`, `Internal`, `SSAS`)
**Conventions**: Follow naming from `elt-patterns.md` and `kimball-patterns.md`; index naming from `dw-physical-design.md` Section 1; file/folder layout from `devops-operations-patterns.md` Section 8

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
8. If RLS roles are required: generate role JSON stubs following `security-implementation.md` Section 3.2 pattern
**Reference**: `ssas-tabular-bp.md` for all naming and structure conventions; `security-implementation.md` for role patterns

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

**Prerequisites:** The `dw-report-designer.agent.md` interview protocol must be completed first. The agent will have produced a requirements artifact containing: confirmed grain, confirmed dimensions, confirmed measures, confirmed bus matrix (signed off), report layout, and user sign-off. **Mode N must refuse to proceed without a signed-off bus matrix.**

**Input:** Complete design specification document from the interview protocol.

#### Dependency DAG (fixed execution order)

Mode N executes build modes in this order. Each step must complete and be validated before the next starts. (Mode-letter mapping reflects this skill's actual modes: H=DW Schema, J=Source SPs, K=SSIS Catalog, I=SSAS Tabular, L=DAX, M=ADO Pipeline.)

```
0. Mode E  — Bus Matrix Validation (verify bus matrix from spec against live schema if DW exists;
              confirm conformed dimensions; flag any ✓ gaps before DDL is generated)
               ↓ (produces: validated/updated Bus Matrix markdown artifact)
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
| Mode E | `design/bus-matrix.md` (signed-off bus matrix — create or update in-place) | Mode H (table list + FK structure), Mode I (dimension relationships) |
| Mode H | `DW/Dimension/[TableName].sql` — one file per new Dimension table | Mode J (SPs reference these tables) |
| Mode H | `DW/Fact/[TableName].sql` — one file per new Fact table | Mode J |
| Mode H | `DW/Staging/[TableName].sql` — one file per new Staging table | Mode J |
| Mode H | `DW/Internal/[TableName].sql` — one file per new Internal table | Mode J |
| Mode H | `DW/SSAS/[ViewName].sql` — one file per new SSAS schema view | Mode I (partition source view names) |
| Mode J | `DW/Dimension/Load[EntityName].sql` — one SP file per dimension entity | Mode K (SSIS packages call these SPs) |
| Mode J | `DW/Fact/Load[EntityName].sql` — one SP file per fact entity | Mode K, Mode I |
| Mode J | `DW/Staging/Load[EntityName].sql` — one SP file per staging entity | Mode K |
| Mode K | `SSIS/{ProjectName}_SSIS/ssis_catalog_configuration.json` (create or update in-place) | Mode M (pipeline deploys SSIS project + configures catalog) |
| Mode I | `SSAS/{ModelName}/` (update existing TMDL — add tables/relationships, do not overwrite) | Mode L (measures added to this model) |
| Mode L | Updated `SSAS/{ModelName}/tables/[MeasureTable].tmdl` | Mode M (deployed via TE2 CLI) |
| Mode M | Build and deployment notes (pipeline definitions live in ADO Server UI, not in this repo) | User review |

#### Update-in-place rule for design artifacts

Design artifacts in `design/` are **living documents — updated in-place, never regenerated from scratch**:

- **`design/spec.md`**: Open the existing file; update only the section(s) affected by the current session. Do not touch sections that were not discussed.
- **`design/decisions.md`**: Update or add rows; never delete rows. If an answer changes, record the new answer in the Answer column and note the prior value in Notes.
- **`design/bus-matrix.md`**: Update the markdown table when fact tables or dimensions are added or changed. Add a change log entry with date and description.
- **`design/entity-map.md`**: Append new entities when Mode P is re-run; do not overwrite existing profiling data.
- **`design/glossary.md`**: Add new terms; update definitions only when a term is formally re-agreed with the user. Never remove terms — if a term is superseded, mark it as `[deprecated — see: NewTerm]`.

**Lazy creation**: Do not create any `design/` file until it has substantive content to write. The `design/` folder itself may be created empty.

If a file does not exist, create it from the relevant template in the `CopilotDWTools` toolkit.

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
| Bus Matrix | ✅ Updated | `design/bus-matrix.md` |
| Design Spec | ✅ Updated | `design/spec.md` |
| Decisions Register | ✅ Updated | `design/decisions.md` |
| Glossary | ✅ Updated | `design/glossary.md` (only if terms were agreed) |
| Source Entity Map | ✅ Updated | `design/entity-map.md` |
| DW Dimension tables | ✅ Created/Updated | `DW/Dimension/[Table].sql` (one file per object) |
| DW Fact tables | ✅ Created/Updated | `DW/Fact/[Table].sql` (one file per object) |
| DW Staging tables | ✅ Created/Updated | `DW/Staging/[Table].sql` (one file per object) |
| SSAS schema views | ✅ Created/Updated | `DW/SSAS/[View].sql` (one file per object) |
| Dimension load SPs | ✅ Created/Updated | `DW/Dimension/Load[Entity].sql` (one file per SP) |
| Fact load SPs | ✅ Created/Updated | `DW/Fact/Load[Entity].sql` (one file per SP) |
| Staging load SPs | ✅ Created/Updated | `DW/Staging/Load[Entity].sql` (one file per SP) |
| SSIS Catalog Config | ✅ Created/Updated | `SSIS/{ProjectName}_SSIS/ssis_catalog_configuration.json` |
| SSAS Model (TMDL) | ✅ Updated | `SSAS/{ModelName}/` (patched, not replaced) |
| ADO Pipeline | ✅ Notes generated | Pipeline definitions live in ADO Server UI — see next steps |

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

---

### Mode P: Source System Analysis

**Trigger:** User says "explore source", "analyse source database", "my data is in [SourceDB]", "I want to build a DW for [subject area]", or invokes Mode P explicitly. Also activated by `dw-report-designer` Phase 2 Step 3 for each source system named by the user.

**Input:** Source identifier + connection details. The shape of the input depends on the source type:
- **SQL Server**: server name + database name (+ optional filtered table list)
- **CSV**: one or more CSV files (the actual data, a header-only export, or a sample extract) + delimiter/quoting convention if non-standard
- **Other (manual)**: user-provided plain-language description of entities, keys, and relationships

#### Path A — SQL Server (automated, full profiling)

1. Connect to the source database using the `ms-mssql.mssql` MCP tool.
2. Run **Q1–Q5 once across the full source database** (or filtered table list if supplied):
   - **Q1** — Table Inventory with row counts and column counts
   - **Q2** — Date/status column detection *(triage output before feeding Q7 — name patterns produce false positives)*
   - **Q3** — Primary Key Map *(flag `uniqueidentifier` PKs as alternate-key candidates; generate INT surrogate in DW)*
   - **Q4** — FK Relationship Map *(if zero rows returned database-wide, activate No FK Constraints handling)*
   - **Q5** — Inbound/Outbound FK Count Summary
3. Apply the classification heuristics from `references/source-system-analysis.md` to classify each table.
4. **For each Fact candidate**, run:
   - **Q6** — NULL Rate Check on FK and key columns *(for tables > 10M rows, restrict to FK and date columns only)*
   - **Q8** — Duplicate PK Check on the candidate natural key *(duplicates = data quality issue; flag for ELT deduplication)*
5. **For each Status/Type column flagged in Q2** (after human triage), run **Q7** — Cardinality Profiling. Fewer than 20 distinct values → junk dimension candidate.
6. **For each date column on each Fact candidate**, run **Q9** — Date Range Profiling. Record `MIN`/`MAX` dates — these drive Calendar dimension start date for Mode H.
7. Run **Q10** — CDC/Change Tracking Detection once per source database. Record result; this drives the ELT incremental strategy for Mode K and Mode M.
8. Produce the **Source Entity Map** using the output format in `references/source-system-analysis.md`.
9. For each Fact candidate, propose a grain: *"Based on `[Table]`, a natural grain is: one row per [PK description]. Does this match your reporting requirement?"*

**No FK constraints:** When Q4 returns zero rows database-wide, apply implied FK detection — columns in high-row-count tables named `[EntityName]ID` / `[EntityName]Key` / `[EntityName]Code` whose names match PK column names in lower-row-count tables. Present these as *inferred* relationships in a separate section of the entity map, clearly labelled. Warn the user that these must be confirmed.

#### Path B — CSV (automated profiling)

Applies to direct flat-file feeds **and** CSV header/sample exports from any other source system (Salesforce, Oracle, PostgreSQL, MySQL, etc.). See `references/source-system-analysis.md` § "CSV Source Discovery" for the full procedure.

1. Profile each CSV using whichever tool is available: PowerShell `Import-Csv`, Python `pandas`, or bulk-load to SQL Server staging.
2. Derive the same outputs as Q1–Q9 (table inventory, date/status columns, PK candidates, inferred FKs, NULL rates, cardinality, duplicate PK check, date range profiling).
3. Q10 (CDC) is not applicable — ask the user whether the CSV represents a one-time load, an incremental drop, or a full snapshot per delivery; record the answer for Mode K.
4. All FK relationships derived from CSV are **inferred** and go into the "Inferred Relationships (low confidence)" section of the entity map.

#### Path C — Manual discovery (last resort)

Used only when the source is non-SQL **and** the user cannot provide a CSV export (e.g. legacy mainframe, proprietary API).

1. Ask the user to describe each entity in plain language (table name, key fields, relationships).
2. Build the entity map manually with confidence marked `low — no automated profiling`.
3. Record the connector requirement for the eventual SSIS data flow.

**Output:** Structured Source Entity Map document saved to `design/entity-map.md` (append new entities if file already exists; never overwrite). Contents:
- Source type marker (SQL Server / CSV / Manual) + discovery confidence rating
- Fact candidates (rows, date columns, FK columns, grain proposal)
- Dimension candidates (rows, natural key, SCD potential)
- Reference/lookup tables and junk dimension candidates
- Source Change Detection summary (CDC/CT status from Q10, or CSV delivery cadence)
- Inferred relationships (always for CSV; for SQL Server only when no FK constraints)
- Ignored tables (with reason)
- Data quality flags (NULL rates from Q6, duplicate PK issues from Q8)
- Date range summary per Fact candidate (from Q9)

**Consumer:** Phase 2 of `dw-report-designer.agent.md` calls this mode for each named source system. The entity map feeds Phase 3 (grain confirmation), Phase 4 (measure candidates from numeric columns), and Phase 5 (dimension design) — do not repeat discovery questions already answered in the entity map.

**Reference:** `references/source-system-analysis.md` for all query text, CSV profiling procedure, classification heuristics, and output format.



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
