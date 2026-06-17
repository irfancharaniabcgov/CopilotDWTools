# Org Design Constraints — Always Pass to Sub-Agents

These are **binding organisational decisions**. Every sub-agent must apply them regardless of what a reference file or general best practice might suggest. Do not deviate without an explicit user override recorded in `design/decisions.md`.

---

## 1. SCD Default: Type 1

Default all dimensions to SCD Type 1 (overwrite, no history). Do **not** generate SCD Type 2 infrastructure (`Is Current Row`, `Valid From`, `Valid To`, history table) unless the user has explicitly confirmed that historical versions are required. When reviewing, do not flag missing SCD Type 2 columns as a defect — the absence is intentional.

## 2. SSAS Tabular Only

This organisation uses **SSAS Tabular exclusively**. Never generate, suggest, or review:
- Multidimensional models (cubes, MOLAP, ROLAP, HOLAP)
- MDX queries or MDX-based report datasets
- Calculated members or named sets (MDX constructs)

If asked, redirect: *"This toolkit only supports SSAS Tabular."*

## 3. ELT, Not ETL

All transformations happen **inside SQL Server in T-SQL stored procedures** — not inside SSIS data flows. SSIS is used for orchestration only (Execute SQL Task, calling SPs). Never generate or approve:
- SSIS Derived Column transforms
- SSIS Lookup transforms
- SSIS data flow expressions for business logic
- Any transformation that cannot be expressed in a T-SQL SP

## 4. Idempotency Required

All generated scripts must be safe to re-run without error or data corruption:
- Tables: `IF NOT EXISTS (SELECT 1 FROM sys.tables WHERE ...) CREATE TABLE ...`
- Views and SPs: `CREATE OR ALTER ...`
- Extended properties: `IF EXISTS ... sp_updateextendedproperty ELSE sp_addextendedproperty`
- Indexes: `IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE ...) CREATE INDEX ...`

Never generate non-idempotent scripts (bare `CREATE TABLE`, bare `CREATE PROCEDURE`).

## 5. No MAXDOP Hints

Do **not** add `OPTION (MAXDOP N)` to any generated T-SQL query or stored procedure. MAXDOP is controlled at the server/Resource Governor level by the DBA. If a review identifies a potential MAXDOP concern, raise it as an advisory note — never add the hint to generated code.

## 6. ADO Classic Pipelines Only

This organisation uses **Azure DevOps Server on-premises with Classic pipelines**. Never generate:
- YAML pipeline definitions
- GitHub Actions workflows
- Azure DevOps Service (cloud) pipeline configurations

All pipeline output must be in Classic pipeline task format (documented in `devops-deployment-patterns.md`).

## 7. Tabular Editor 2 Only

Only **Tabular Editor 2 (free version)** is available: `E:\Tools\TabularEditor\TabularEditor.exe`. Never reference or generate:
- Tabular Editor 3 APIs, features, or script syntax
- `.NET 5+` C# syntax in TE2 scripts (`async/await`, `record` types, nullable reference types, `Chunk`, `MaxBy`, `MinBy`)
- Any TE2 script that calls a TE3-only method

## 8. Upstream-First (Roche's Maxim)

> *"Data should be transformed as far upstream as possible, and as far downstream as necessary."*

Before recommending a DAX measure, SSAS calculated column, or calculated table, always ask: *can this live upstream?* Upstream preference order:

1. Staging SP (`Staging.Load*`)
2. Dimension/Fact load SP (`Dimension.Load*`, `Fact.Load*`)
3. DW computed column (simple, fixed derivations only)
4. SSAS calculated column (display-only)
5. DAX measure (last resort — only for aggregations that cannot exist at a fixed row level)

## 9. Classification Default: `Unreviewed`

For all new columns and tables, default `InformationType` and `SensitivityLabel` extended properties to `Unreviewed`. Never default to `Public`, `Protected A`, or any other label without explicit user confirmation. The `Unreviewed` label signals that human review is still required.

## 10. Self-Contained DW (No Cross-Database Queries)

Generated stored procedures must not use:
- 3-part or 4-part names referencing other databases (e.g., `OtherDB.dbo.Table`)
- Linked server references (`[LinkedServer].[DB].[schema].[object]`)
- `OPENROWSET`, `OPENQUERY`, or `BULK INSERT` from file paths
- CLR assemblies, `xp_cmdshell`, Database Mail, Service Broker

These are cloud-portability blockers. If found during a review, flag as 🟠 HIGH (portability risk) with a note that the pattern blocks a future Fabric/Azure SQL migration.

---

## How to Use This File

Include in every sub-agent prompt with: `**Always include**: decisions/org-design-constraints.md`

Sub-agents must apply these constraints **before** applying any pattern from a skill reference file. If a skill reference suggests a pattern that conflicts with a constraint above, the constraint wins. If genuinely uncertain, flag the conflict and let the orchestrator decide — do not silently apply the reference-file pattern.
