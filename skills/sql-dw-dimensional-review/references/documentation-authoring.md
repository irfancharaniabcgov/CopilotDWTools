# Documentation Authoring — Inference Heuristics & Coverage Audits

> **Companion to:** `extended-properties-templates.md` (templates + property taxonomy) and `ssas-tabular-bp.md` (SSAS BPA rules that flag missing descriptions).
>
> **Purpose:** Provide the heuristics, audit queries, and interview question library used by the `db-documenter` agent to backfill documentation on existing databases — source systems, the DW, and SSAS Tabular models.

This file is the source of truth for the **inference** and **gap-detection** layer. The actual T-SQL/TMDL output formats live in `extended-properties-templates.md`.

---

## Authority & Scope

This reference covers three documentation targets:

| Target | Mechanism | Audit query | Write target |
|---|---|---|---|
| SQL Server source database | Extended properties (`MS_Description` + classification) | Q-SRC-1 to Q-SRC-3 below | `sp_addextendedproperty` script (review with owner before run) |
| Data Warehouse (`Dimension.*`, `Fact.*`, `Staging.*`, `Internal.*`, `SSAS.*` schemas) | Extended properties (full property set per `extended-properties-templates.md`) | Q-DW-1 to Q-DW-3 below | Inline post-deploy script in SSDT project (`DW/Scripts/Post-Deployment/`) |
| SSAS Tabular model | TMDL `description` property on tables, columns, measures | Q-SSAS-1 to Q-SSAS-3 below | TMDL file edits in `SSAS/{ModelName}/tables/*.tmdl` |

---

## 1. Coverage Audit Queries

Run the appropriate audit first. Output drives the worklist: every undocumented object is a candidate for the inference + interview loop.

### Q-SRC-1 — Source DB: Tables and views missing MS_Description

```sql
-- Lists all user tables and views with NO MS_Description set at the object level.
-- Order by row count desc so high-volume tables are documented first.
SELECT
    s.[name] + '.' + o.[name]                AS [QualifiedName],
    o.[type_desc]                            AS [ObjectType],
    ISNULL(p.row_count, 0)                   AS [RowCount],
    o.[create_date],
    o.[modify_date]
FROM   sys.objects o
JOIN   sys.schemas s ON o.schema_id = s.schema_id
LEFT JOIN (
    SELECT object_id, SUM(rows) AS row_count
    FROM   sys.partitions
    WHERE  index_id IN (0, 1)
    GROUP BY object_id
) p ON p.object_id = o.object_id
LEFT JOIN sys.extended_properties ep
    ON ep.major_id = o.object_id
   AND ep.minor_id = 0
   AND ep.class    = 1
   AND ep.[name]   = N'MS_Description'
WHERE  o.[type] IN ('U', 'V')                -- U = table, V = view
   AND s.[name] NOT IN ('sys', 'INFORMATION_SCHEMA')
   AND ep.major_id IS NULL
ORDER BY [RowCount] DESC, [QualifiedName];
```

### Q-SRC-2 — Source DB: Columns missing MS_Description on documented tables

```sql
-- Only flag columns on tables that ARE documented at the object level.
-- This prevents noise: undocumented tables surface first via Q-SRC-1.
SELECT
    s.[name] + '.' + o.[name] + '.' + c.[name] AS [QualifiedColumn],
    TYPE_NAME(c.user_type_id)
        + CASE WHEN c.max_length > 0 AND TYPE_NAME(c.user_type_id) IN ('varchar','nvarchar','char','nchar','varbinary')
               THEN ' (' + CAST(CASE WHEN TYPE_NAME(c.user_type_id) LIKE 'n%' THEN c.max_length / 2 ELSE c.max_length END AS VARCHAR(10)) + ')'
               ELSE '' END                    AS [DataType],
    c.is_nullable,
    c.is_identity
FROM   sys.columns c
JOIN   sys.objects o ON c.object_id = o.object_id
JOIN   sys.schemas s ON o.schema_id = s.schema_id
JOIN   sys.extended_properties ep_table        -- table IS documented
    ON ep_table.major_id = o.object_id
   AND ep_table.minor_id = 0
   AND ep_table.class    = 1
   AND ep_table.[name]   = N'MS_Description'
LEFT JOIN sys.extended_properties ep_col        -- column is NOT documented
    ON ep_col.major_id = o.object_id
   AND ep_col.minor_id = c.column_id
   AND ep_col.class    = 1
   AND ep_col.[name]   = N'MS_Description'
WHERE  o.[type] IN ('U', 'V')
   AND s.[name] NOT IN ('sys', 'INFORMATION_SCHEMA')
   AND ep_col.major_id IS NULL
ORDER BY s.[name], o.[name], c.column_id;
```

### Q-SRC-3 — Source DB: Stored procedures and functions missing MS_Description

```sql
SELECT
    s.[name] + '.' + o.[name]                AS [QualifiedName],
    o.[type_desc]                            AS [ObjectType],
    o.[create_date],
    o.[modify_date]
FROM   sys.objects o
JOIN   sys.schemas s ON o.schema_id = s.schema_id
LEFT JOIN sys.extended_properties ep
    ON ep.major_id = o.object_id
   AND ep.minor_id = 0
   AND ep.class    = 1
   AND ep.[name]   = N'MS_Description'
WHERE  o.[type] IN ('P', 'FN', 'IF', 'TF')   -- P=SP, FN=scalar fn, IF=inline TVF, TF=multi-stmt TVF
   AND s.[name] NOT IN ('sys', 'INFORMATION_SCHEMA')
   AND ep.major_id IS NULL
ORDER BY s.[name], o.[name];
```

### Q-DW-1 / Q-DW-2 / Q-DW-3 — DW coverage

Same SQL as Q-SRC-1/2/3 but with the additional WHERE clause restricting schemas to DW conventions:

```sql
   AND s.[name] IN ('Dimension', 'Fact', 'Staging', 'Internal', 'SSAS')
```

For the DW, also report **completeness of the full property set** (not just `MS_Description`). Use this companion query for facts and dimensions:

```sql
-- DW completeness: every Fact/Dimension table should have ALL required properties set.
-- Required per extended-properties-templates.md: MS_Description, TableType, plus Grain (facts) or SCDType (dims).
WITH RequiredProps AS (
    SELECT 'MS_Description' AS PropertyName UNION ALL
    SELECT 'TableType'      UNION ALL
    SELECT 'Grain'          -- expected for Fact schema only
    -- Add 'SCDType', 'ConformedDimension' for Dimension schema via a CASE in production
)
SELECT
    s.[name] + '.' + o.[name]      AS [QualifiedName],
    rp.PropertyName                AS [MissingProperty]
FROM   sys.objects o
JOIN   sys.schemas s ON o.schema_id = s.schema_id
CROSS JOIN RequiredProps rp
LEFT JOIN sys.extended_properties ep
    ON ep.major_id = o.object_id
   AND ep.minor_id = 0
   AND ep.class    = 1
   AND ep.[name]   = rp.PropertyName
WHERE  o.[type] = 'U'
   AND s.[name] IN ('Fact', 'Dimension')
   AND ep.major_id IS NULL
ORDER BY s.[name], o.[name], rp.PropertyName;
```

### Q-SSAS-1 — SSAS Tabular: Visible tables missing description

Run via DMV against a live AS endpoint or by parsing TMDL files in `SSAS/{ModelName}/tables/`.

```sql
-- DMV path (live SSAS endpoint)
SELECT
    t.[Name]              AS [TableName],
    t.[IsHidden],
    t.[Description]
FROM   $SYSTEM.TMSCHEMA_TABLES t
WHERE  t.[IsPrivate]  = FALSE
  AND  t.[IsHidden]   = FALSE
  AND (t.[Description] IS NULL OR t.[Description] = '');
```

**TMDL file path (offline):** in each `tables/*.tmdl` file, look for a top-level `table '[Name]'` block and check whether a `description: "..."` line is present immediately after the table declaration. Absence = undocumented.

### Q-SSAS-2 — SSAS Tabular: Visible columns missing description

```sql
SELECT
    t.[Name]              AS [TableName],
    c.[Name]              AS [ColumnName],
    c.[IsHidden],
    c.[Description]
FROM   $SYSTEM.TMSCHEMA_COLUMNS c
JOIN   $SYSTEM.TMSCHEMA_TABLES  t ON c.[TableID] = t.[ID]
WHERE  t.[IsPrivate]  = FALSE
  AND  c.[IsHidden]   = FALSE
  AND (c.[Description] IS NULL OR c.[Description] = '');
```

### Q-SSAS-3 — SSAS Tabular: Visible measures missing description

```sql
SELECT
    t.[Name]              AS [TableName],
    m.[Name]              AS [MeasureName],
    m.[Expression],
    m.[Description]
FROM   $SYSTEM.TMSCHEMA_MEASURES m
JOIN   $SYSTEM.TMSCHEMA_TABLES   t ON m.[TableID] = t.[ID]
WHERE  m.[IsHidden]   = FALSE
  AND (m.[Description] IS NULL OR m.[Description] = '');
```

> The `DW_TABLES_HAVE_DESCRIPTION` and `DW_MEASURES_HAVE_DESCRIPTION` BPA rules in `ssas-tabular-bp.md` flag the same conditions. Use BPA in CI; use these DMV queries in interactive sessions.

---

## 2. Inference Heuristics — From Code to Draft Description

> **Documentation purpose** — every description is an investment that reduces the cost of every subsequent change. The goal is shared understanding for human + AI collaboration on the system. Document what is **unusual, ambiguous, or surprising**. Skip what is **conventional or self-evident** — that adds noise and obscures the signal.

For every object that survives the Convention vs. Surprise Test (§ 2.0.2), the agent should produce a **draft description** before asking the user. The user reviews concrete text, not blank fields. Heuristics by object type:

### 2.0 Convention Detection (run BEFORE per-table drafting)

The highest-leverage step in a documentation pass. Find recurring patterns that appear across many tables, confirm each once with the user, and apply a single boilerplate description to every occurrence.

#### 2.0.1 Detection queries

**Recurring column names across tables:**

```sql
-- Columns appearing in >= 30% of tables in the database.
-- Strong signal that the column is an org-wide convention (audit, lineage, soft-delete, etc.)
WITH TableCount AS (
    SELECT COUNT(*) AS TotalTables FROM sys.tables WHERE is_ms_shipped = 0
)
SELECT
    c.[name]                            AS [ColumnName],
    COUNT(DISTINCT c.object_id)         AS [TableOccurrences],
    (SELECT TotalTables FROM TableCount) AS [TotalTables],
    CAST(100.0 * COUNT(DISTINCT c.object_id) /
         NULLIF((SELECT TotalTables FROM TableCount), 0) AS DECIMAL(5,1)) AS [PercentOfTables],
    -- Sample a single canonical data type to show variation
    MAX(TYPE_NAME(c.user_type_id))      AS [SampleDataType],
    MAX(CAST(c.is_nullable AS INT))     AS [AnyNullable]
FROM   sys.columns c
JOIN   sys.tables t ON c.object_id = t.object_id
WHERE  t.is_ms_shipped = 0
GROUP BY c.[name]
HAVING COUNT(DISTINCT c.object_id) >= (SELECT TotalTables FROM TableCount) * 30 / 100
ORDER BY [TableOccurrences] DESC;
```

**Sample values for a candidate convention column (run per candidate):**

```sql
-- Use top 5 distinct values from the most-populated occurrence to confirm semantics.
-- Replace [ConventionColumn] with the column name surfaced above.
SELECT TOP 5
    [ConventionColumn], COUNT(*) AS [Occurrences]
FROM   [Schema].[Table]
GROUP BY [ConventionColumn]
ORDER BY COUNT(*) DESC;
```

#### 2.0.2 The Convention vs. Surprise Test

For each object (table, view, SP, column, relationship), apply this test before generating a description:

> **"If a competent developer looked at this object's name + data type + relationships, would they get the meaning right?"**
>
> - **Yes** → skip (self-evident; description adds noise)
> - **No** → document (this is what makes the system hard to reason about)

Decision matrix:

| Category | Document? |
|---|---|
| Standard PK named `[Entity]ID` / `[Entity]Key` | ❌ skip (universal convention) |
| Audit columns covered by Principle 5 blanket | ❌ skip (handled once via convention) |
| Self-evident columns: `FirstName`, `City`, `EmailAddress` | ❌ skip (name says it all) |
| Boolean flag where 1/0 meaning is non-obvious (`IsActive`, `IsLocked`) | ✅ document (domain meaning matters) |
| Numeric column where unit/currency/scale is ambiguous | ✅ document (e.g. "Amount in CAD; net of tax") |
| FK column whose target is non-obvious from name | ✅ document (relationship signal) |
| Column whose value is from a coded list maintained elsewhere | ✅ document (point to the code list) |
| Status / type columns with business rules | ✅ document (junk dimension candidates) |
| **Tables** | ✅ always (table is the unit of reasoning) |
| **Views** that hide non-trivial joins/filters | ✅ always (what subset of reality does this present?) |
| **Stored procedures** | ✅ always (body alone doesn't tell you when/why it's called) |
| **Triggers** | ✅ always (invisible side effects) |
| **Non-obvious / inferred relationships** | ✅ always (highest-value documentation for downstream reasoners) |
| **DAX measures** with non-trivial filter context | ✅ always (BPA-enforced) |

**Target outcome:** every table, view, SP, measure, and non-trivial relationship has a description. Columns are documented only when the test says "no" — typically 10–30% of columns in a well-named schema.

#### 2.0.3 Common conventions to detect and confirm

| Convention | Typical names | Typical blanket description (subject to user confirmation) |
|---|---|---|
| Audit — created | `CreatedDate`, `CreatedOn`, `CreatedBy`, `CreatedUserID`, `Entry_Date`, `Entry_User_ID`, `RowCreatedAt` | "Audit: when/by whom the row was first inserted in the source system. Populated by the application, not by ELT." |
| Audit — modified | `ModifiedDate`, `ModifiedOn`, `ModifiedBy`, `Update_Date`, `Update_User_ID`, `LastUpdated`, `RowModifiedAt` | "Audit: when/by whom the row was last modified in the source system. Used as the incremental watermark by ELT." |
| Soft delete | `IsDeleted`, `IsActive`, `DeletedDate`, `IsArchived` | "Soft-delete flag: 1 = logically deleted (excluded from default views); 0 = active. Rows are never physically removed." |
| Row version (concurrency) | `RowVersion`, `Timestamp` (binary) | "Concurrency token. Auto-maintained by SQL Server. Not for reporting." |
| Lineage (org pattern) | `LineageKey` | "ELT lineage key. Populated by the source SP from `Internal.Lineage`; links a row back to the batch that loaded it." |
| Sentinel dates | `'1753-01-01'` in DATE columns | "Sentinel for unknown/missing date. See `Internal.Sentinels` documentation." |
| Sentinel keys | `-1` in FK columns | "Sentinel for unknown member. Points to the row in the target dimension where `[Entity Key] = -1`." |
| Tenancy / org isolation | `TenantID`, `OrganizationID`, `CompanyCode` | "Multi-tenant isolation key. Filters all rows to the caller's tenant." |

After convention confirmation, apply the agreed description to every matching column in one batch. Record the confirmed list to `design/decisions.md` under `## Documentation Conventions` so future sessions can skip the detection step.

### 2.1 Table name patterns

| Name pattern | Likely meaning | Draft description starting point |
|---|---|---|
| `*_Header` / `*_Detail` | Order header + line items | "Header rows for [entity]; one row per [entity instance]. Detail rows linked via [FK]." |
| `tbl*`, `t_*` | Legacy Hungarian prefix | Strip prefix to derive entity name |
| `vw_*`, `v_*` | View | "View over [base tables]; serves [purpose]." Infer base tables from view definition. |
| `usp_*`, `sp_*`, `Load*`, `Extract*`, `Import*` | Stored procedure | "Procedure to [verb from name] [object]." See SP body inference below. |
| `Dim*`, `Fact_*`, `Bridge_*` | Pre-Kimball naming | Map to DW schema convention; flag for migration to schema-prefixed naming |
| `Audit*`, `Log*`, `History*`, `Archive*` | Operational | "[Audit / log / history / archive] of [entity]. Retention: [ask user]." |
| `*_Staging`, `Stg*`, `tmp*` | Staging | "Staging table — transient. Loaded by [SP from object dependencies]." |
| `*Reference`, `*Lookup`, `*Code` | Reference/lookup | "Reference list of [entity]." |

### 2.2 Column name patterns (only when not skipped by § 2.0.2)

Apply these only to columns that **survived** the Convention vs. Surprise Test — i.e. not handled by the convention blanket and not self-evident.

| Pattern | Likely meaning | Draft description |
|---|---|---|
| `*ID`, `*Key`, `*Code` *(when non-obvious target)* | Identifier or FK | "Identifier for [entity]." If suffixed `Key` and integer → "Surrogate key." If suffixed `ID` → "Natural identifier from source." |
| `*Date`, `*DateTime`, `*Timestamp`, `*_DT` *(non-audit)* | Business date column | "Date [business event] occurred." For DW: also flag for Calendar FK conversion. |
| `Is*`, `Has*`, `*Flag`, `*_YN` *(non-obvious)* | Boolean | "Indicator: [Yes value meaning] / [No value meaning]." Cardinality check (Q7) confirms domain. |
| `*Amount`, `*Amt`, `*Value`, `*Price`, `*Cost`, `*Total` | Numeric measure | "Amount in [currency — confirm]." Flag as additive measure candidate. |
| `*Qty`, `*Quantity`, `*Count`, `*Number` (numeric) | Numeric quantity | "Quantity of [thing]; additive." |
| `*Name`, `*Description`, `*Notes`, `*Comments` | Free text | "Descriptive text. Classification: Free-form Text." |
| `*Address*`, `*City*`, `*PostalCode*`, `*Phone*`, `*Email*` *(when scale/context is non-obvious)* | PII contact | "Contact information. Classification: Contact Info." |
| `*SIN*`, `*SocialInsurance*` | Sensitive PII | "**Sensitive.** Classification: SIN / Protected B." |
| `*Password*`, `*Token*`, `*Secret*` | Credential | "**Sensitive credential.** Classification: Credentials / Protected B." |
| Sentinel values `-1`, `'1753-01-01'`, `'9999-12-31'` *(only if not covered by § 2.0.3 convention pass)* | DW sentinel | "Sentinel: [unknown / open-ended]." |

> **Conventions already covered**: `Created*` / `Modified*` / audit columns and `RowVersion` are handled by the convention pass (§ 2.0); they are NOT re-documented per table.
>
> **Override rule:** if a column appears as a PK in `sys.indexes` and matches no naming pattern, draft: "Primary key. Natural identifier in source system." Then ask the user for the business meaning.

### 2.3 Stored procedure body inference

Read the SP body and extract signal from these patterns:

| SQL pattern | Signal | Description contribution |
|---|---|---|
| `MERGE` / `INSERT … FROM … LEFT JOIN target` | Upsert | "Upserts [target table] from [source]." |
| `TRUNCATE TABLE` followed by `INSERT` | Full reload | "Full-replace load of [target]." |
| `WHERE [date column] > @LastRun` or `@HighWaterMark` | Incremental | "Incremental load using [column] as watermark." |
| `OUTPUT INTO` or `OUTPUT DELETED.*` | Audit / SCD2 | "Captures changed rows for [audit / SCD2 history]." |
| Calls to `Internal.Lineage` / `Internal.RethrowError` | Org standard ELT pattern | "Follows org ELT pattern: lineage tracking, structured error handling." |
| Single `SELECT` returning rows | Read / report SP | "Returns [columns]. Used by [report / app — ask user]." |
| `RAISERROR` / `THROW` with specific codes | Validation | "Validates [condition]; raises error [code]." |

### 2.4 DAX measure expression inference

For SSAS Tabular measures, parse the expression to derive `Description`:

| DAX pattern | Inferred intent | Draft description |
|---|---|---|
| `SUM( Fact[...] )` | Simple additive | "Total [column name expanded]." |
| `DISTINCTCOUNT( Dim[Key] )` | Count of entities | "Count of distinct [entity name]." |
| `DIVIDE( [A], [B] )` | Ratio | "Ratio of [A] to [B]. Returns BLANK when [B] is zero." |
| `CALCULATE( [M], DATESYTD( ... ) )` | Year-to-date | "[M] from start of calendar year to selected date." |
| `CALCULATE( [M], DATESYTD( ..., "03-31" ) )` | Fiscal YTD | "[M] from start of fiscal year (Apr 1) to selected date." |
| `CALCULATE( [M], SAMEPERIODLASTYEAR(...) )` | Prior year | "[M] for the equivalent period in the prior calendar year." |
| `CALCULATE( [M], REMOVEFILTERS( Dim ) )` | Grand total | "[M] ignoring [dimension] filter." |
| `VAR x = ... RETURN ...` with `FILTER`/`ALL` | Complex pattern | Flag for user review — describe in plain language what filters are applied and removed. |

**Description format (org standard from `pbix-report-standards.md`):**
```
Valid groupings: [Dim 1], [Dim 2], …
Notes: [behavior at edges — blanks, ratios, time period assumptions]
```

### 2.5 Cross-object signal

When inference is uncertain, expand to neighbouring objects:

- Table description unclear? → Look at SPs that read from it (`sys.sql_dependencies`) and SPs that write to it (`Load*` naming) — the load SP usually states the table's purpose in its own header comment if any.
- Column description unclear? → Sample 10 distinct values and present them to the user with the draft.
- Measure description unclear? → Look at base measures it depends on (parse expression); the leaf measure usually has the meaning.

### 2.6 Relationship documentation

Relationships are often the highest-value documentation for human or AI reasoning — they reveal *how the system works internally* in a way that table descriptions alone cannot. Always generate relationship descriptions for:

- Every FK constraint whose meaning isn't trivially obvious from the column name
- Every **inferred** relationship (no FK constraint but a join is implied — e.g. `WorkOrders.CustomerID` matching `Customers.CustomerID` with no FK defined)
- Every many-to-many relationship and its bridge table
- Every self-referencing relationship (parent/child, hierarchy)
- Every relationship to a tenant/security boundary table

#### Where to attach the description

| Object | Mechanism |
|---|---|
| SQL Server FK constraint | `sp_addextendedproperty` with `@level0type='SCHEMA', @level1type='TABLE', @level2type='CONSTRAINT'` |
| SQL Server inferred relationship (no FK) | `MS_Description` on the FK column itself, prefixed with "Inferred FK: ..." |
| SSAS Tabular relationship | TMDL `description` property on the relationship block |

#### What to include in a relationship description

| Element | Example |
|---|---|
| Target entity (when name doesn't make it obvious) | "Points to `Identity.User` (not `dbo.Customer` — `UserID` here means the application user, not the customer)." |
| Cardinality | "Many-to-one. An Order belongs to exactly one Customer; a Customer can have many Orders." |
| Optionality | "Optional — guest checkouts have `CustomerID = 0` pointing to the 'Anonymous' sentinel row." |
| Business meaning | "An Order's Customer is the bill-to party at the time of order placement. If the customer changes their account after the order, the original Customer row is preserved via SCD2." |
| Special rules | "Soft-deleted Customers retain their Orders for historical reporting; the relationship traversal must not filter on `Customer.IsDeleted = 0`." |
| When inferred | "**Inferred** relationship — no FK constraint defined in source. Confirmed with data team [date]." |

Bad → Good examples:

| Bad | Good |
|---|---|
| "FK to Customer" | "Many-to-one to `Sales.Customer`. The Customer is the bill-to party at order time; preserved via SCD2 if the customer record changes." |
| "Links Order to LineItem" | "One Order has many `Order_Detail` rows (≥ 1). Cascade-delete is NOT configured; deleting an Order leaves orphan detail rows — handle in application." |

---

## 3. Interview Question Library

After producing a draft, the agent presents it to the user and asks **targeted confirmation questions** — not open-ended ones.

### Table-level questions (per undocumented table)

1. *"My draft for `[Schema].[Table]` is: '[draft text]'. Is this accurate, or should I revise?"*
2. *"What is the business name for this table? (How would a stakeholder refer to it in conversation?)"*
3. *"Who owns this table — which team or person is the data steward?"*
4. *"What's the source system? Is it the system of record, or a downstream copy?"*
5. *"Refresh frequency: hourly, daily, weekly, on-demand, or unknown?"*

Skip questions whose answers are obvious from the codebase (e.g. if the table is in the `Staging` schema and named `Load*` is its loader, skip ownership and refresh).

### Column-level questions (batch by table)

1. *"I drafted descriptions for [N] columns on `[Table]`. Here they are — flag any you want to revise: [list]"*
2. *"For columns I marked as PII (Contact Info / Personal / SIN / Health), confirm the sensitivity label: Public / Protected A / Protected B / Protected C?"*
3. *"For boolean-shaped columns (`Is*`, `*Flag`), confirm the domain: what does 1/0 (or Y/N) actually mean in business terms?"*
4. *"For status / code columns, confirm the value list — should this be a junk dimension?"*

### Measure-level questions (per undocumented SSAS measure)

1. *"My draft for measure `[Name]` is: '[draft]'. Confirm or revise?"*
2. *"What does a BLANK result mean for this measure — no data, zero activity, or 'not applicable'?"*
3. *"List the dimensions this measure makes sense to group by — and any it should NOT be grouped by (e.g. semi-additive measures across time)?"*

### Stored procedure questions

1. *"My draft for `[SP]` is: '[draft]'. The body suggests [pattern]. Confirm?"*
2. *"What calls this SP — SSIS package, SQL Agent job, application, or another SP?"*
3. *"Is this SP safe to re-run on the same input (idempotent)?"*

---

## 4. Description Style Guide

Apply these rules to every generated description, regardless of target:

| Rule | Why |
|---|---|
| First sentence ≤ 200 characters | Fits in SSMS / Tabular Editor tooltip |
| State **what it is** before **how it is loaded** | Business readers don't care about ELT mechanics |
| Use present tense, third person | "Stores customer master records" — not "Will store" or "I store" |
| Quantify when known | "One row per order line item" beats "Stores order detail" |
| Use canonical glossary terms (from `design/glossary.md` if present) | Consistency with the rest of the project |
| Do NOT include credentials, server names, or paths | Extended properties are dumped in many tools; treat as semi-public |
| Mark inferred descriptions with `[INFERRED]` prefix in the *draft* stage only | Removed before final write after user confirms |

### Bad → Good examples

| Bad | Good |
|---|---|
| "Customer table" | "Master list of customers. One row per customer per source system. SCD Type 2 — historical changes preserved via [Is Current Row]." |
| "Sales fact" | "Sales transactions at line-item grain. One row per sales order line per day. Loaded nightly from ERP." |
| "Calculates revenue" | "Total revenue from posted sales orders in the selected period. Excludes credits and refunds (see [Refunds] for those). Valid groupings: Calendar, Customer, Product, Region." |
| "Is the customer active" | "Indicator: 1 = customer has had activity in the past 12 months; 0 = inactive. Recomputed nightly." |

---

## 5. Batch Workflow

The `db-documenter` agent processes undocumented objects in this order — convention pass first, then per-table batches:

### Pass 0 — Convention Detection (run once per database)

1. Run the convention-detection query in § 2.0.1 to find recurring column names (≥ 30% of tables)
2. For each candidate, sample top 5 values from the most-populated occurrence (§ 2.0.1)
3. Present each detected convention to the user in **one message**:
   > *"I found these recurring patterns. For each, confirm the draft description, revise, or skip:*
   > *• `ModifiedDate` — appears in 47 of 52 tables. Sample values look like recent timestamps. Draft: '[from § 2.0.3]'. Confirm?*
   > *• `IsDeleted` — appears in 38 of 52 tables. Values: 0 (most rows), 1 (few rows). Draft: '[from § 2.0.3]'. Confirm?*
   > *• `Entry_User_ID` — appears in 41 of 52 tables. Sample values are short codes like 'JDOE'. Draft: 'Audit: user identifier of the person who created the row. Format: AD username, all caps.'. Confirm?"*
4. Apply confirmed conventions as a blanket — write descriptions to every occurrence
5. Record confirmed conventions to `design/decisions.md` under `## Documentation Conventions` so future passes skip detection

### Pass 1 — Per-table batches (run for each table in scope)

1. Run coverage audit (Q-SRC-1, Q-DW-1, or Q-SSAS-1 depending on target)
2. Prioritise: high-row-count tables first; tables with downstream dependencies next; archive/log tables last (often acceptable to leave undocumented)
3. **Filter undocumented columns through the Convention vs. Surprise Test (§ 2.0.2)** — discard any whose meaning is self-evident from name + data type + relationships. Typically 70–90% of columns are skipped at this stage.
4. For each table in the worklist:
   - Run the appropriate column audit (Q-SRC-2 / Q-DW-2 / Q-SSAS-2) restricted to that table
   - Generate drafts for: table description (always) + relationship descriptions (§ 2.6, for non-trivial FKs) + the column drafts that survived the test
   - Present the batch to the user in one message — table description first, then relationships, then columns in a markdown table
   - User reviews, confirms, or revises in one round
   - Write back per the target's write mechanism (§ Authority & Scope)
5. After all tables in scope: re-run the audit to confirm coverage; report any objects deliberately skipped (with reason)

**Expected coverage targets after a pass:**

| Object type | Target coverage |
|---|---|
| Tables | 100% (every table is the unit of reasoning) |
| Views | 100% |
| Stored procedures | 100% |
| Triggers | 100% |
| Non-trivial relationships | 100% |
| Columns | ~30% (convention columns covered by blanket + the unusual/ambiguous ones from per-table pass; the remaining 70% are self-evident and intentionally skipped) |
| SSAS measures | 100% (BPA-enforced) |

---

## 6. Skip Rules

Do not generate descriptions for these — they pollute the documentation and are conventionally understood:

- System tables and views (`sys.*`, `INFORMATION_SCHEMA.*`)
- Temp tables (`#*`)
- Diagram metadata (`dtproperties`, `sysdiagrams`)
- Replication objects (`MSrepl_*`, `sysmergesubscriptions`, etc.)
- SSAS internal tables marked `IsPrivate = TRUE`
- Hidden TMDL columns whose name starts with underscore (e.g. `_RowNumber`) unless the user explicitly requests

For columns: skip `RowVersion` / `timestamp` columns (auto-generated, no business meaning) unless asked.

---

## 7. Coverage Report Format

After a documentation pass, produce a brief coverage report saved to `design/documentation-coverage.md` (update in-place):

```markdown
# Documentation Coverage Report

**Target**: [Source DB: ServerName.DBName | DW: DBName | SSAS: ModelName]
**Pass date**: YYYY-MM-DD
**Agent**: db-documenter

## Summary

| Object type | Total | Documented before | Documented after | Coverage % | Target |
|---|---|---|---|---|---|
| Tables | N | N | N | XX% | 100% |
| Views | N | N | N | XX% | 100% |
| Stored procedures | N | N | N | XX% | 100% |
| Triggers | N | N | N | XX% | 100% |
| Non-trivial relationships | N | N | N | XX% | 100% |
| Columns — convention (blanket) | N | – | N | – | covered by Pass 0 |
| Columns — unusual / ambiguous | N | N | N | XX% | 100% of "surprising" set |
| Columns — self-evident (skipped intentionally) | N | – | – | – | not targeted |
| SSAS measures | N | N | N | XX% | 100% |

## Conventions Detected and Applied (Pass 0)

| Convention | Columns matched | Tables touched | Description applied (final, post-user-review) |
|---|---|---|---|
| Audit — created | 47 (CreatedDate, CreatedBy, Entry_Date, Entry_User_ID) | 47 of 52 | "Audit: when/by whom the row was first inserted..." |
| Soft delete | 38 (IsDeleted) | 38 of 52 | "Soft-delete flag: 1 = logically deleted..." |
| ... | ... | ... | ... |

Conventions also recorded in `design/decisions.md` § Documentation Conventions for future passes.

## Skipped (intentional — Convention vs. Surprise Test)

Number of columns skipped because they were self-evident (name + data type + relationships made the meaning obvious to a competent reader): N

A high skip count is healthy — it indicates good naming hygiene. List individual skips here only if the user explicitly requests; otherwise the count is sufficient.

## Skipped (skip rules)

| Object | Reason |
|---|---|
| ... | sys.* / temp / replication / IsPrivate |

## Deferred (user wants to revisit)

| Object | Note |
|---|---|
| ... | ... |
```

This file is the audit trail. It does not replace the actual extended properties / TMDL descriptions — those are written inline at the source.
