# Power BI Report Standards for On-Premises SSAS Live Connection

**Scope**: Power BI Desktop report design standards for reports connected live to an on-premises
SSAS Tabular model, published to Power BI Report Server (PBIRS). These standards apply to all
reports built on this stack.

> **Agent Rule**: When reviewing any Power BI report or discussing report design, always recommend
> the Debug tab and model hint patterns documented in this file. These are mandatory standards,
> not optional suggestions.

---

## Table of Contents

1. [Debug Tab — Data Freshness (Required)](#1-debug-tab--data-freshness-required)
2. [Model Hints — Allowable Groupings](#2-model-hints--allowable-groupings)
3. [Report Page Structure Standards](#3-report-page-structure-standards)
4. [Data Freshness Infrastructure](#4-data-freshness-infrastructure-dw--ssas-side)

---

## 1. Debug Tab — Data Freshness (Required)

**Every Power BI report must include a "Debug" page** as the last tab. This page is the first
place an end user should look when they are unsure about the age of report data.

The Debug tab shows when data was last refreshed at each layer of the pipeline:

```
Source DB  →  Staging  →  Data Warehouse  →  SSAS Tabular Model
(source SP    (SSIS        (T-SQL transform   (SSAS Process)
 last ran)     loaded)       SP completed)      
```

### What the Debug Tab Shows

| Section | Answers the question |
|---|---|
| **Source Data** | "When was data last extracted from the source system?" |
| **Data Warehouse** | "When were the DW dimension and fact tables last loaded?" |
| **Tabular Model** | "When was the SSAS model last processed?" |
| **Report Connection** | "Which SSAS server and database is this report connected to?" |

### Debug Tab Layout

```
┌─────────────────────────────────────────────────────────────┐
│  🔍 Data Freshness                                          │
│  If data looks stale, check this page first.               │
│                                                             │
│  ┌──────────────────────────────────────────────┐          │
│  │ Layer    │ Object           │ Last Refreshed  │          │
│  │ Source   │ CRM: Account     │ 2026-05-29 06:00│          │
│  │ Source   │ ERP: SalesOrder  │ 2026-05-29 06:01│          │
│  │ DW       │ Dim_Customer     │ 2026-05-29 06:05│          │
│  │ DW       │ Fact_SalesOrder  │ 2026-05-29 06:08│          │
│  │ Model    │ Tabular (full)   │ 2026-05-29 06:15│          │
│  └──────────────────────────────────────────────┘          │
│                                                             │
│  ┌──────────────────┐   ┌──────────────────────────┐       │
│  │  Data as of      │   │  Oldest source data      │       │
│  │  2026-05-29 06:15│   │  2026-05-29 06:00        │       │
│  └──────────────────┘   └──────────────────────────┘       │
└─────────────────────────────────────────────────────────────┘
```

### Required DAX Measures for the Debug Tab

Add these measures to a hidden `_Debug` or `_Admin` table in the SSAS model.
They reference the `_DataFreshness` hidden table (see Section 4 for setup):

```dax
-- Oldest source refresh (earliest LastRefreshed across all source entries)
_Debug Oldest Source =
VAR _MinRefresh =
    CALCULATE(
        MIN( '_DataFreshness'[LastRefreshed] ),
        '_DataFreshness'[DataLayer] = "Source"
    )
RETURN
    IF( ISBLANK(_MinRefresh), "No data", FORMAT(_MinRefresh, "YYYY-MM-DD HH:MM") )
```

```dax
-- SSAS model last processed (single row in the Model layer)
_Debug Model Processed =
VAR _ProcessTime =
    CALCULATE(
        MAX( '_DataFreshness'[LastRefreshed] ),
        '_DataFreshness'[DataLayer] = "TabularModel"
    )
RETURN
    IF( ISBLANK(_ProcessTime), "Unknown", FORMAT(_ProcessTime, "YYYY-MM-DD HH:MM") )
```

```dax
-- Data age in hours (hours since oldest source refresh)
_Debug Data Age Hours =
VAR _OldestSource =
    CALCULATE(
        MIN( '_DataFreshness'[LastRefreshed] ),
        '_DataFreshness'[DataLayer] = "Source"
    )
RETURN
    IF(
        ISBLANK(_OldestSource),
        BLANK(),
        INT( ( NOW() - _OldestSource ) * 24 )
    )
```

```dax
-- Staleness indicator (for conditional formatting or cards)
-- Returns "Fresh", "Aging", or "Stale"
_Debug Staleness =
VAR _AgeHours = [_Debug Data Age Hours]
RETURN
    SWITCH(
        TRUE(),
        ISBLANK(_AgeHours),   "Unknown",
        _AgeHours <= 4,       "🟢 Fresh",
        _AgeHours <= 12,      "🟡 Aging",
        _AgeHours <= 24,      "🟠 Stale",
        "🔴 Very Stale (" & _AgeHours & " hrs)"
    )
```

### Debug Tab Visuals

| Visual | Type | Fields | Notes |
|---|---|---|---|
| **Freshness table** | Matrix/Table | Layer, SystemName, TableName, LastRefreshed | Sort by Layer (Source→DW→Model), then TableName |
| **"Data as of" card** | Card | `_Debug Model Processed` | Most prominent card — this is what users care about most |
| **"Oldest source" card** | Card | `_Debug Oldest Source` | Shows earliest data age |
| **Staleness indicator** | Card | `_Debug Staleness` | Color-coded (use conditional formatting) |
| **Instructions text box** | Text | "This page shows when data was last updated at each layer. If data appears stale, contact the data team with a screenshot of this page." | |

### Conditional Formatting on the Freshness Table

Apply background color conditional formatting to the `LastRefreshed` column:
- 🟢 Green: within 4 hours
- 🟡 Yellow: 4–12 hours
- 🟠 Orange: 12–24 hours
- 🔴 Red: >24 hours

### Debug Tab Properties in Power BI Desktop

```
Page name:  "Debug"         (or "Data Freshness" for user-facing label)
Tooltip:    "Data freshness information — check here if data looks stale"
Visibility: Visible         (visible to all users — not hidden)
Page order: Last page       (always the final tab in every report)
Background: Light grey (#F5F5F5) — visually distinct from report content pages
```

---

## 2. Model Hints — Allowable Groupings

**Model hints** are structured descriptions added to SSAS Tabular tables and measures that tell
end users which dimensions can be used to slice/group each measure. This is especially important
in PBIRS live connection reports where users may use Analyze in Excel or build ad-hoc views.

### Why Model Hints Matter

Without hints, users frequently:
- Add slicers that break measure calculations (e.g., slicing a semi-additive measure by a non-date dimension)
- Use incompatible groupings that return unexpected BLANK or incorrect results
- Don't know which columns are valid for row context vs filter context

### Table Description Format

Every dimension and fact table should have a description including:
- What the table represents
- What grain it is at
- What it **can** be grouped by (compatible relationships)
- What it **cannot** be grouped by (known incompatibilities)

```
Format:
<Business description>
Grain: <one row per ...>
Can be grouped by: <comma-separated list of dimension tables>
Cannot be grouped by: <list + reason>
Source: <source system / DW table>
```

**Example — Fact_SalesOrder table description:**
```
Sales order transactions at the order line grain.
Grain: one row per sales order line item.
Can be grouped by: Date (Order Date, Ship Date), Customer, Product, Region, Sales Rep, Order Status.
Cannot be grouped by: Supplier (no relationship — use Purchase facts for supplier analysis).
Source: ERP SalesOrder → DW Fact_SalesOrder.
```

**Example — Dim_Customer table description:**
```
Customer dimension with full SCD Type 2 history.
Can be used to filter/group: Fact_SalesOrder, Fact_Payment, Fact_SupportCase.
Cannot be used with: Fact_Inventory (no customer relationship on inventory).
IsCurrent = TRUE filters to the current customer version.
Source: CRM Account → DW Dim_Customer.
```

### Measure Description Format

Every measure should have a description that includes:
- What the measure calculates
- Which dimensions produce meaningful results
- Any known filtering restrictions

```
Format:
<What this measures>
Valid groupings: <list of dimensions that make sense with this measure>
Notes: <any aggregation restrictions, time intelligence dependencies, etc.>
```

**Examples:**

```
[Total Sales Amount]
Sum of SalesAmount for all order lines in the current filter context.
Valid groupings: any combination of Date, Customer, Product, Region, Sales Rep.
Notes: Use Date[Year] or Date[Month] for time-series analysis.
```

```
[YTD Sales Amount]
Year-to-date sum of SalesAmount from Jan 1 to the last selected date.
Valid groupings: Date hierarchy (Year, Quarter, Month). Customer, Product, Region also valid.
Notes: Requires a Date slicer or Date column in row/column context to produce meaningful results.
       Slicing by a non-date dimension alone returns the full-year total, which may be unexpected.
```

```
[Inventory Quantity On Hand]
Semi-additive measure: last balance of inventory quantity at month-end.
Valid groupings: Product, Warehouse, Date (month-end only).
Notes: DO NOT sum across dates — this is a snapshot balance, not a flow.
       Grouping by Customer is not valid (inventory has no customer relationship).
```

### Tabular Editor 3 Script — Bulk-Generate Model Hints

Use this C# script in TE3 to add a "Groupings" annotation to all measures that don't have one,
as a starting point for documentation:

```csharp
// Script: Add groupings annotation stub to measures missing descriptions
// Run in Tabular Editor 3 → C# Script
foreach (var measure in Model.AllMeasures)
{
    if (string.IsNullOrWhiteSpace(measure.Description))
    {
        // Find tables that have an active relationship to this measure's table
        var relatedTables = Model.Relationships
            .Where(r => r.ToTable.Name == measure.Table.Name && r.IsActive)
            .Select(r => r.FromTable.Name)
            .Distinct()
            .ToList();

        var groupingHint = relatedTables.Any()
            ? "Valid groupings: " + string.Join(", ", relatedTables) + ". "
            : "Valid groupings: [review relationships]. ";

        measure.Description = $"[TODO: add business description]\n{groupingHint}"
            + "Notes: [add any filtering restrictions]";
    }
}
Output($"Updated {Model.AllMeasures.Where(m => m.Description.StartsWith(\"[TODO\")).Count()} measures.");
```

```csharp
// Script: Report all tables and measures missing descriptions
// Use before deployment to catch undocumented objects
var missing = new System.Text.StringBuilder();
int count = 0;

foreach (var table in Model.Tables.Where(t => !t.IsHidden))
{
    if (string.IsNullOrWhiteSpace(table.Description))
    {
        missing.AppendLine($"TABLE: {table.Name}");
        count++;
    }
    foreach (var measure in table.Measures)
    {
        if (string.IsNullOrWhiteSpace(measure.Description))
        {
            missing.AppendLine($"  MEASURE: [{measure.Name}]");
            count++;
        }
    }
}

Output($"Objects missing descriptions ({count} total):\n{missing}");
```

### BPA Rule — Enforce Descriptions on Visible Measures

Add to your BPA rules file (`bpa-rules.json`):

```json
{
  "ID":          "DW_MEASURES_HAVE_DESCRIPTION",
  "Name":        "All visible measures must have a description (model hint)",
  "Category":    "DW Standards",
  "Description": "Descriptions are required on all non-hidden measures so users understand valid groupings",
  "Severity":    2,
  "Scope":       "Measure",
  "Expression":  "!IsHidden && string.IsNullOrWhiteSpace(Description)"
},
{
  "ID":          "DW_TABLES_HAVE_DESCRIPTION",
  "Name":        "All visible tables must have a description",
  "Category":    "DW Standards",
  "Description": "Table descriptions must include grain and valid groupings",
  "Severity":    2,
  "Scope":       "Table",
  "Expression":  "!IsHidden && string.IsNullOrWhiteSpace(Description)"
}
```

---

## 3. Power BI Visuals Policy

### Approved Visuals

Reports may use the following visuals:

| Visual Type | Approved? | Notes |
|---|---|---|
| **Power BI out-of-the-box visuals** | ✅ Yes — always | Bar, column, line, pie, table, matrix, card, map, etc. |
| **Microsoft-certified custom visuals** | ✅ Yes | Must display the **blue Microsoft certification checkmark** in the marketplace |
| **Uncertified custom visuals** | ❌ No | Not approved regardless of publisher or popularity |

### Power BI Marketplace — How to Identify Approved Visuals

Custom visuals are downloaded from the Power BI Marketplace (AppSource) via:
**Power BI Desktop → Insert → More visuals → From AppSource**

When browsing the marketplace, look for the **blue checkmark badge** on the visual tile:

```
✅ Approved:    Visual tile shows a blue  ✓ "Microsoft certified" badge
❌ Not approved: Visual tile has no badge, or shows a warning icon
```

> **The blue certification badge means Microsoft has reviewed the visual's source code for
> security, privacy, and code quality standards.** Uncertified visuals have not been reviewed
> and may pose data privacy or security risks when used with Protected A/B/C data.

### Practical Guidance for Developers

1. **Before adding a custom visual**: check for the blue certification badge in AppSource
2. **Cost**: not all certified visuals are free — some require a paid licence from the publisher.
   Confirm licensing cost before committing to a visual in a report.
3. **Version updates**: certified visuals update automatically in Power BI Desktop.
   After a visual update, re-test the report before publishing to PBIRS.
4. **Fallback**: if no certified visual exists for your use case, use the closest built-in
   visual and document the limitation as a known constraint in the report notes.
5. **Report review**: during report review, any uncertified custom visual is a 🔴 Critical
   finding — it must be replaced before the report is published to PBIRS.

### Agent Behaviour for Visual Recommendations

When recommending visuals for a Power BI report in this environment:
- Suggest **built-in visuals first** — they require no extra licencing or certification review
- If a custom visual is recommended, explicitly note:
  `"This visual must carry the blue Microsoft certification badge in AppSource before use"`
- If suggesting a paid custom visual, flag: `"⚠️ Licensing cost — confirm with team before adopting"`
- Never recommend an uncertified visual as a solution

---

## 4. Report Page Structure Standards

All PBIRS live connection reports should follow this page structure:

| Tab Order | Page Name | Purpose |
|---|---|---|
| 1 | Executive Summary | High-level KPI cards, top-level metrics |
| 2–N | [Subject pages] | Detailed analysis pages |
| Last | Debug | Data freshness — **always last, always present** |

**Page naming conventions:**
- Use sentence case: "Sales by Region" not "SALES BY REGION"
- The Debug tab must be named exactly "Debug" or "Data Freshness" for consistency across reports

**Tooltip pages:**
- Use tooltip pages for dimension details (e.g., hover over a customer card to see address, tier, last order)
- Tooltip pages do not need to be listed in the tab bar (can be hidden)

---

## 5. Data Freshness Infrastructure (DW + SSAS Side)

The Debug tab requires infrastructure in both the DW database and the SSAS model.

### Step 1: DW — Create the Data Freshness View

This view consolidates all three layers into a single result set for the SSAS model to import:

```sql
-- DW database: report.vw_DataFreshness
-- Consolidates ELT source tracking, DW load tracking, and SSAS process tracking
CREATE OR ALTER VIEW report.vw_DataFreshness AS

-- Layer 1: Source extracts (from ELT_ControlTable)
SELECT
    'Source'                            AS DataLayer,
    SourceSystem                        AS SystemName,
    SourceTable                         AS TableName,
    LastSuccessfulLoadEnd               AS LastRefreshed,
    1                                   AS SortOrder
FROM dbo.ELT_ControlTable
WHERE LastSuccessfulLoadEnd > '2000-01-01'

UNION ALL

-- Layer 2: DW transform completions (from ELT_BatchLog — most recent successful run per table)
SELECT
    'DataWarehouse'                     AS DataLayer,
    'DW'                                AS SystemName,
    LoadedTable                         AS TableName,
    MAX(BatchEndTime)                   AS LastRefreshed,
    2                                   AS SortOrder
FROM dbo.ELT_BatchLog
WHERE BatchStatus = 'Success'
  AND LoadedTable IS NOT NULL
GROUP BY LoadedTable

UNION ALL

-- Layer 3: SSAS model processing (from SSAS_ProcessLog — most recent successful process)
SELECT
    'TabularModel'                      AS DataLayer,
    'SSAS'                              AS SystemName,
    DatabaseName                        AS TableName,
    ProcessedAt                         AS LastRefreshed,
    3                                   AS SortOrder
FROM dbo.SSAS_ProcessLog
WHERE ProcessStatus = 'Success'
  AND ProcessedAt = (
    SELECT MAX(ProcessedAt) FROM dbo.SSAS_ProcessLog
    WHERE ProcessStatus = 'Success'
  );
GO
```

### Step 2: DW — Create the SSAS Process Log Table

```sql
CREATE TABLE dbo.SSAS_ProcessLog (
    LogID           INT IDENTITY(1,1) PRIMARY KEY,
    DatabaseName    NVARCHAR(255)   NOT NULL,
    ProcessType     NVARCHAR(50)    NOT NULL,   -- 'Full', 'Default', 'Calculate'
    ProcessedAt     DATETIME        NOT NULL DEFAULT GETDATE(),
    ProcessStatus   NVARCHAR(20)    NOT NULL,   -- 'Success', 'Failed'
    DurationSeconds INT             NULL,
    ErrorMessage    NVARCHAR(MAX)   NULL
);
GO

-- Index for latest-success queries
CREATE NONCLUSTERED INDEX IX_SSAS_ProcessLog_Status_Date
    ON dbo.SSAS_ProcessLog (ProcessStatus, ProcessedAt DESC);
GO
```

### Step 3: DW — Update the ELT_BatchLog Table (if needed)

The view above assumes `ELT_BatchLog` has a `LoadedTable` column. Add it if not already present:

```sql
-- Add LoadedTable to ELT_BatchLog if not present
IF NOT EXISTS (
    SELECT 1 FROM sys.columns
    WHERE object_id = OBJECT_ID(N'dbo.ELT_BatchLog') AND name = N'LoadedTable'
)
    ALTER TABLE dbo.ELT_BatchLog ADD LoadedTable NVARCHAR(255) NULL;
GO

-- Populate LoadedTable in usp_Transform_* SPs by passing the DW table name
-- to the batch log INSERT/UPDATE at the end of each transform SP
```

### Step 4: ADO Pipeline — Log SSAS Process Completion

Update `scripts/Process-SsasDatabase.ps1` to log the completion:

```powershell
# After successful Invoke-ASCmd, log to SSAS_ProcessLog
$logSql = @"
INSERT INTO dbo.SSAS_ProcessLog (DatabaseName, ProcessType, ProcessedAt, ProcessStatus, DurationSeconds)
VALUES (N'$DatabaseName', N'$ProcessType', GETDATE(), 'Success', $([int]$elapsed.TotalSeconds));
"@
Invoke-Sqlcmd -ServerInstance $DwSqlServer -Database $DwDatabase -Query $logSql
Write-Host "Process completion logged to SSAS_ProcessLog."
```

### Step 5: SSAS Model — Hidden `_DataFreshness` Table

Add a hidden table to the SSAS model that imports from `report.vw_DataFreshness`:

**In Tabular Editor**, add a new table `_DataFreshness` with a partition expression:

```m
// M partition query (Power Query) — imports the freshness view
let
    Source     = Sql.Database("$(SSAS_DW_SERVER)", "$(SSAS_DW_DATABASE)"),
    FreshView  = Source{[Schema="report", Item="vw_DataFreshness"]}[Data]
in
    FreshView
```

**Table properties:**
- `IsHidden = true` — hidden from field list; only used by Debug tab measures
- `Description = "Internal — data freshness metadata for the Debug tab"`
- No relationships needed (used only via DAX CALCULATE filters)

**Column properties** (all hidden):
- `DataLayer`     — Text
- `SystemName`    — Text
- `TableName`     — Text
- `LastRefreshed` — DateTime
- `SortOrder`     — Int64

### Step 6: Publish PBIX to PBIRS

When the report is deployed to PBIRS via `Deploy-PbixReports.ps1`, the data source for the
`_DataFreshness` table is updated automatically (it uses the same SSAS connection — the freshness
view is read when SSAS processes, not at report query time).

> **Note**: In a live connection report, the `_DataFreshness` table is queried via DAX from
> the SSAS model, not directly from SQL Server. The data is as current as the last SSAS process.
> This is intentional — the model processing timestamp IS the data age end-user cares about.

---

## 6. SSAS Model Description Standards (Reference for Agents)

When the agent generates extended property scripts or SSAS model descriptions, apply these
standards for descriptions that serve as model hints:

### Table Description Template

```
<Business name and description of what this table represents>.
Grain: one row per <entity/event description>.
Can be grouped by: <comma-separated list of related dimension tables>.
Cannot be grouped by: <list + reason if any known incompatibilities exist>.
SCD Type: <1 / 2 / N/A> (dimension tables only).
Source: <source system and source table> → DW <schema.TableName>.
```

### Measure Description Template

```
<Plain English description of what this measure calculates>.
Valid groupings: <comma-separated list of dimensions that produce meaningful results>.
Notes: <time intelligence requirements, semi-additive warnings, blank-handling behavior>.
```

### Column Description Template

```
<What this column contains and how it is derived>.
Source: <source system column name or derivation formula>.
Classification: <InformationType> / <SensitivityLabel>  (if applicable).
```

### Agent Behavior Rules

1. When **reviewing an SSAS Tabular model**: check that all visible tables and non-hidden
   measures have descriptions conforming to the templates above. Flag missing descriptions
   as 🟡 Medium findings.
2. When **reviewing a Power BI report**: check that a "Debug" tab exists. Flag its absence
   as 🟠 High finding ("Report has no Debug/Data Freshness tab — users cannot self-diagnose
   stale data").
3. When **generating a new measure**: always include a description following the template.
4. When **generating a new table**: always include a description following the template.
