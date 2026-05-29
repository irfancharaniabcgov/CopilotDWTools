# Power BI Report Standards for On-Premises SSAS Live Connection

**Scope**: Power BI Desktop report design standards for reports connected live to SSAS Tabular, published to Power BI Report Server (PBIRS).

**Deployed PBIRS version:** Power BI Report Server May 2026 (v1.26.9637.31070, build 15.0.1121.109).

> **Agent Rule**: When reviewing any Power BI report or discussing report design, always recommend the Debug tab and model hint patterns documented here. These are **mandatory standards**, not optional suggestions.

---

## 1. Debug Tab — Data Freshness (Required)

**Every report must include a "Debug" page as the last tab.**

The three-layer freshness model:
```
Source DB  →  Data Warehouse  →  SSAS Tabular Model
(extract       (T-SQL transform    (SSAS Process)
 last ran)      SP completed)
```

### Debug Tab Visuals

| Visual | Type | Fields |
|---|---|---|
| **Freshness table** | Matrix/Table | Layer, SystemName, TableName, LastRefreshed |
| **"Data as of" card** | Card | `_Debug Model Processed` — most prominent card |
| **"Oldest source" card** | Card | `_Debug Oldest Source` |
| **Staleness indicator** | Card | `_Debug Staleness` — conditional formatting |
| **Instructions text box** | Text | "If data appears stale, contact the data team with a screenshot of this page." |

Apply background color to `LastRefreshed`: 🟢 ≤4h · 🟡 4–12h · 🟠 12–24h · 🔴 >24h

### Debug Tab Page Properties

```
Page name:   "Debug"  (or "Data Freshness")
Visibility:  Visible to all users — not hidden
Page order:  Always the final tab
Background:  Light grey (#F5F5F5) — visually distinct from content pages
```

### Required DAX Measures

Add to a hidden `_Debug` or `_Admin` table in the SSAS model. They reference the `_DataFreshness` hidden table (see Section 5 for setup).

```dax
_Debug Oldest Source =
VAR _MinRefresh =
    CALCULATE( MIN( '_DataFreshness'[LastRefreshed] ), '_DataFreshness'[DataLayer] = "Source" )
RETURN IF( ISBLANK(_MinRefresh), "No data", FORMAT(_MinRefresh, "YYYY-MM-DD HH:MM") )
```

```dax
_Debug Model Processed =
VAR _ProcessTime =
    CALCULATE( MAX( '_DataFreshness'[LastRefreshed] ), '_DataFreshness'[DataLayer] = "TabularModel" )
RETURN IF( ISBLANK(_ProcessTime), "Unknown", FORMAT(_ProcessTime, "YYYY-MM-DD HH:MM") )
```

```dax
_Debug Data Age Hours =
VAR _OldestSource =
    CALCULATE( MIN( '_DataFreshness'[LastRefreshed] ), '_DataFreshness'[DataLayer] = "Source" )
RETURN IF( ISBLANK(_OldestSource), BLANK(), INT( ( NOW() - _OldestSource ) * 24 ) )
```

```dax
_Debug Staleness =
VAR _AgeHours = [_Debug Data Age Hours]
RETURN
    SWITCH( TRUE(),
        ISBLANK(_AgeHours),  "Unknown",
        _AgeHours <= 4,      "🟢 Fresh",
        _AgeHours <= 12,     "🟡 Aging",
        _AgeHours <= 24,     "🟠 Stale",
        "🔴 Very Stale (" & _AgeHours & " hrs)"
    )
```

---

## 2. Model Hints — Allowable Groupings

Model hints tell users which dimensions can slice each measure — critical for live connection where users may use Analyze in Excel or build ad-hoc views.

### Table Description Format

```
<Business description of what this table represents.>
Grain: one row per <entity/event>.
Can be grouped by: <comma-separated related dimension tables>.
Cannot be grouped by: <list + reason>.
SCD Type: <1 / 2 / N/A>  (dimension tables only)
Source: <source system> → DW <schema.TableName>.
```

**Example — SalesOrder (from `[Fact].[SalesOrder]` via `[SSAS].[SalesOrder]`):**
```
Sales order transactions at the order line grain.
Grain: one row per sales order line item.
Can be grouped by: Date (Order Date, Ship Date), Customer, Product, Region, Sales Rep, Order Status.
Cannot be grouped by: Supplier (no relationship — use Purchase facts for supplier analysis).
Source: ERP SalesOrder → DW [Fact].[SalesOrder].
```

### Measure Description Format

```
<What this measure calculates.>
Valid groupings: <dimensions that produce meaningful results>.
Notes: <aggregation restrictions, time intelligence dependencies, blank-handling behaviour>.
```

**Examples:**
```
[Total Sales Amount]
Sum of SalesAmount for all order lines in the current filter context.
Valid groupings: any combination of Date, Customer, Product, Region, Sales Rep.
Notes: Use Date[Year] or Date[Month] for time-series analysis.
```

```
[Inventory Quantity On Hand]
Semi-additive measure: last balance of inventory quantity at month-end.
Valid groupings: Product, Warehouse, Date (month-end only).
Notes: DO NOT sum across dates — snapshot balance, not a flow.
       Grouping by Customer is not valid (no customer relationship on inventory).
```

---

## 3. Power BI Visuals Policy

| Visual Type | Approved? | Notes |
|---|---|---|
| **Power BI out-of-the-box visuals** | ✅ Yes | Bar, column, line, pie, table, matrix, card, map, etc. |
| **Microsoft-certified custom visuals** | ✅ Yes | Must display the **blue Microsoft certification checkmark** in AppSource |
| **Uncertified custom visuals** | ❌ No | Not approved regardless of publisher or popularity |

**Finding a certified visual:** Power BI Desktop → Insert → More visuals → From AppSource. Look for the blue ✓ "Microsoft certified" badge on the tile. The badge means Microsoft reviewed the source code for security, privacy, and code quality.

**Agent behaviour for visual recommendations:**
- Suggest built-in visuals first.
- If recommending a custom visual, explicitly state: `"This visual must carry the blue Microsoft certification badge in AppSource before use."`
- If paid: flag `"⚠️ Licensing cost — confirm with team before adopting."`
- **Any uncertified custom visual in a report is a 🔴 Critical finding — must be replaced before publishing to PBIRS.**

---

## 4. Report Page Structure Standards

| Tab Order | Page Name | Purpose |
|---|---|---|
| 1 | Executive Summary | High-level KPI cards, top-level metrics |
| 2–N | [Subject pages] | Detailed analysis pages |
| Last | Debug | Data freshness — **always last, always present** |

- Page names: sentence case ("Sales by Region" not "SALES BY REGION").
- Debug tab name: exactly "Debug" or "Data Freshness" for consistency.
- Tooltip pages: use for dimension details; can be hidden from tab bar.

---

## 5. Data Freshness Infrastructure (DW + SSAS Side)

### Approach A — Calculated Column `NOW()` (Recommended)

Each SSAS table carries a hidden calculated column `[Last Processed]` with expression `NOW()`. Because calculated columns are evaluated at **processing time**, `NOW()` captures the exact moment the table was last processed.

**Step 1: Add `[Last Processed]` to every SSAS data table** (Tabular Editor):
```json
{
  "type": "calculated", "name": "Last Processed", "dataType": "dateTime",
  "isHidden": true, "expression": "NOW()",
  "formatString": "yyyy-MMM-dd h:mm AM/PM", "displayFolder": "_Debug"
}
```

**TE3 C# script — add to all non-hidden tables missing it:**
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

**Step 2: Add `Last Updated Tabular` calculated table** (UNIONs MAX([Last Processed]) from every data table):
```dax
Last Updated Tabular =
UNION(
    ROW("Last Processed", FORMAT(MAX('FactTableName'[Last Processed]), "yyyy-MMM-dd h:mm AM/PM"),
        "Table Name", "FactTableName"),
    ROW("Last Processed", FORMAT(MAX('DimName'[Last Processed]), "yyyy-MMM-dd h:mm AM/PM"),
        "Table Name", "DimName")
    -- add one ROW() per data table
)
```

**Step 3: DW views for Source and DW freshness layers:**
```sql
CREATE VIEW [SSAS].[Last Updated Source] AS
SELECT [TableName] AS [Table Name], [UpdateDate] AS [Last Updated]
FROM [Internal].[LastUpdatedSource];

CREATE VIEW [SSAS].[Last Loaded DW] AS
SELECT [TableName] AS [Table Name], [LoadDate] AS [Last Loaded]
FROM [Internal].[IncrementalLoads];
```

**Step 4:** Add `Last Updated Source` and `Last Loaded DW` as regular import tables in the SSAS model, reading from the views above.

### Approach B — SSAS_ProcessLog Table (Alternative)

Use when you need a full pipeline audit trail (who triggered, duration, failures):

```sql
CREATE TABLE [Internal].[SSAS_ProcessLog] (
    LogID           INT IDENTITY(1,1) PRIMARY KEY,
    DatabaseName    NVARCHAR(255) NOT NULL,
    ProcessType     NVARCHAR(50)  NOT NULL,   -- 'Full', 'Default', 'Calculate'
    ProcessedAt     DATETIME      NOT NULL DEFAULT GETDATE(),
    ProcessStatus   NVARCHAR(20)  NOT NULL,   -- 'Success', 'Failed'
    DurationSeconds INT           NULL,
    ErrorMessage    NVARCHAR(MAX) NULL
);
```

ADO pipeline step after processing:
```powershell
$logSql = "INSERT INTO Internal.SSAS_ProcessLog
    (DatabaseName, ProcessType, ProcessedAt, ProcessStatus, DurationSeconds)
    VALUES (N'$DatabaseName', N'$ProcessType', GETDATE(), 'Success', $([int]$elapsed.TotalSeconds));"
Invoke-Sqlcmd -ServerInstance $DwSqlServer -Database $DwDatabase -Query $logSql
```

Both approaches can coexist: Approach A gives per-table freshness; Approach B gives pipeline history.

---

## 6. SSAS Model Description Standards

### Templates

**Table description:**
```
<Business name and description>.
Grain: one row per <entity/event>.
Can be grouped by: <related dimension tables>.
Cannot be grouped by: <list + reason>.
SCD Type: <1 / 2 / N/A>  (dimensions only)
Source: <source system and table> → DW <schema.TableName>.
```

**Measure description:**
```
<Plain English description of what this measure calculates>.
Valid groupings: <dimensions that produce meaningful results>.
Notes: <time intelligence requirements, semi-additive warnings, blank-handling>.
```

**Column description:**
```
<What this column contains and how it is derived>.
Source: <source system column or derivation formula>.
Classification: <InformationType> / <SensitivityLabel>  (if applicable)
```

### Agent Behaviour Rules

1. **Reviewing SSAS model**: flag all visible tables and non-hidden measures missing descriptions as 🟡 Medium findings.
2. **Reviewing a Power BI report**: flag missing Debug tab as 🟠 High finding — "Report has no Debug/Data Freshness tab — users cannot self-diagnose stale data."
3. **Generating a new measure or table**: always include a description following the template above.