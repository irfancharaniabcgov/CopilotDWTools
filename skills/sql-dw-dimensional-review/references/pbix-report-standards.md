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
Two approaches exist; **Approach A (calculated column)** is used by the reference projects and is recommended.

---

### Approach A — Calculated Column `NOW()` (Recommended)

Each SSAS table carries a hidden calculated column `[Last Processed]` with expression `NOW()`.
Because DAX calculated columns are evaluated at **processing time**, `NOW()` captures the exact
moment that table was last processed. No external logging table or pipeline step is required.

#### Step 1: Add `[Last Processed]` to every SSAS data table

In Tabular Editor, add a calculated column to each data table:

```json
{
  "type": "calculated",
  "name": "Last Processed",
  "dataType": "dateTime",
  "description": "Shows when this SSAS table was last processed",
  "isHidden": true,
  "expression": "NOW()",
  "formatString": "yyyy-MMM-dd h:mm AM/PM",
  "displayFolder": "_Debug"
}
```

> **Why this works**: DAX calculated columns are computed during model processing.
> `NOW()` captures the processing timestamp and is stored in the column for all rows.
> When you query `MAX([Last Processed])`, you get when the table was last processed.

**TE3 C# script — add `[Last Processed]` to all non-hidden tables that don't already have it:**

```csharp
foreach (var table in Model.Tables.Where(t => !t.IsHidden))
{
    if (table.Columns.All(c => c.Name != "Last Processed"))
    {
        var col = table.AddCalculatedColumn("Last Processed");
        col.Expression    = "NOW()";
        col.DataType      = TabularEditor.TOMWrapper.DataType.DateTime;
        col.IsHidden      = true;
        col.FormatString  = "yyyy-MMM-dd h:mm AM/PM";
        col.Description   = "Shows when this SSAS table was last processed";
        col.DisplayFolder = "_Debug";
    }
}
```

#### Step 2: Add `Last Updated Tabular` calculated table

This table UNIONs `MAX([Last Processed])` from every data table into a single result set
for the Debug tab matrix visual.

```dax
Last Updated Tabular =
UNION(
    ROW(
        "Last Processed", FORMAT(MAX('Fact Table Name'[Last Processed]), "yyyy-MMM-dd h:mm AM/PM"),
        "Table Name",     "Fact Table Name"
    ),
    ROW(
        "Last Processed", FORMAT(MAX('Dimension Name'[Last Processed]), "yyyy-MMM-dd h:mm AM/PM"),
        "Table Name",     "Dimension Name"
    )
    -- ... add one ROW() per data table
)
```

**Table properties:**
- `IsHidden = false` — visible to the Debug tab visuals
- No relationships needed (used as standalone table in the Debug tab matrix)

#### Step 3: DW — Views for Source and DW freshness layers

```sql
-- DW database: SSAS schema
-- Layer 1: Source extract tracking (from Internal.LastUpdatedSource)
CREATE VIEW [SSAS].[Last Updated Source] AS
SELECT [TableName]  AS [Table Name]
      ,[UpdateDate] AS [Last Updated]
FROM [Internal].[LastUpdatedSource];

-- Layer 2: DW load tracking (from Internal.IncrementalLoads)
CREATE VIEW [SSAS].[Last Loaded DW] AS
SELECT [TableName] AS [Table Name]
      ,[LoadDate]  AS [Last Loaded]
FROM [Internal].[IncrementalLoads];
```

#### Step 4: Add `Last Updated Source` and `Last Loaded DW` as import tables in SSAS

These are regular import tables in the SSAS model, reading from the views above.
They appear on the Debug tab alongside `Last Updated Tabular`.

---

### Approach B — SSAS_ProcessLog Table (Alternative)

If you need pipeline-level process logging (e.g., to record who triggered processing, duration,
or failure reason), create a logging table in the DW and populate it from the ADO pipeline:

```sql
CREATE TABLE [Internal].[SSAS_ProcessLog] (
    LogID           INT IDENTITY(1,1) PRIMARY KEY,
    DatabaseName    NVARCHAR(255)   NOT NULL,
    ProcessType     NVARCHAR(50)    NOT NULL,   -- 'Full', 'Default', 'Calculate'
    ProcessedAt     DATETIME        NOT NULL DEFAULT GETDATE(),
    ProcessStatus   NVARCHAR(20)    NOT NULL,   -- 'Success', 'Failed'
    DurationSeconds INT             NULL,
    ErrorMessage    NVARCHAR(MAX)   NULL
);
GO
CREATE NONCLUSTERED INDEX IX_SSAS_ProcessLog_Status_Date
    ON [Internal].[SSAS_ProcessLog] (ProcessStatus, ProcessedAt DESC);
GO
```

ADO pipeline step after `Process-SsasDatabase.ps1` succeeds:

```powershell
$logSql = @"
INSERT INTO Internal.SSAS_ProcessLog
    (DatabaseName, ProcessType, ProcessedAt, ProcessStatus, DurationSeconds)
VALUES (N'$DatabaseName', N'$ProcessType', GETDATE(), 'Success', $([int]$elapsed.TotalSeconds));
"@
Invoke-Sqlcmd -ServerInstance $DwSqlServer -Database $DwDatabase -Query $logSql
```

> **When to use Approach B**: When Approach A's per-table granularity is insufficient and you
> need a full audit trail of all process runs, failures, and durations in the DW for ops reporting.
> Both approaches can coexist — Approach A gives per-table freshness; Approach B gives pipeline history.

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
