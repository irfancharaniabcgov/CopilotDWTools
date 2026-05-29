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
| `references/devops-deployment-patterns.md` | ADO Server pipeline YAML, DACPAC/SSIS/SSAS/PBIRS deployment scripts, PowerShell standards |
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
4. Check staging tables: all nullable, no FKs, `staging.stg_<Prefix>_<TableName>` naming
5. Check transform SPs: upsert/MERGE pattern for dims, append+aggregate for facts
6. Check ELT control tables: `ELT_ControlTable` high-water mark, `ELT_BatchLog` row
7. Check package structure: all child packages run tasks in parallel; Master_Orchestrator runs children in sequence
8. Produce findings report with references to `elt-patterns.md` sections

### Mode G: DevOps Deployment Review
**Input**: Pipeline YAML, PowerShell deployment scripts, SSIS project structure, or SSAS model deployment approach
**Process**:
1. Run the deployment checklist from `devops-deployment-patterns.md` Section 9
2. Identify hardcoded values, missing exit codes, non-idempotent patterns
3. Flag any step that requires GUI/manual interaction
4. Check pipeline stage ordering: DB → SSIS → SSAS → PBIX
5. Check SSIS deployment uses project model + SSISDB environments (not package model)
6. Check SSAS deployment uses Tabular Editor CLI (not VS GUI)
7. Check PBIX upload includes data source update post-upload
8. Produce findings report with references to `devops-deployment-patterns.md` sections

## Output Standards

- Always produce a **severity-coded findings report** (🔴 Critical / 🟠 High / 🟡 Medium / 🔵 Low) using the template in `dw-review-checklist.md`
- Always cite the specific Kimball pattern, SQLBI pattern, or checklist item for each finding
- For extended properties output: produce complete, ready-to-run T-SQL using the upsert pattern
- For bus matrix output: produce a markdown table with ✓ marks for confirmed relationships
- For ELT output: generate parameterized T-SQL SPs and pipeline YAML steps, not GUI click instructions
- For deployment scripts: all PowerShell must follow the standards in `devops-deployment-patterns.md` Section 7 — `[CmdletBinding()]`, `$ErrorActionPreference = 'Stop'`, `exit 0/1`
- Ask clarifying questions before assuming the grain of a fact table — grain definition requires domain knowledge

## Interaction Style

- Be direct about design issues — use severity levels, not vague language
- When a schema shows SCD Type 2 candidates, always ask: "Are historical versions of this entity needed for reporting?"
- When reviewing DAX measures, always show the corrected version alongside the finding
- When generating extended properties, ask about business ownership, source system, and grain if not obvious from schema

## Dependencies

This skill works best when the `database-data-management:ms-sql-dba` agent or `ms-mssql.mssql` MCP tools are available for live database connectivity. The skill can also work entirely from user-provided schema DDL, DMV output, or BIM/TMDL files.
