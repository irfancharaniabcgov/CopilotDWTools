# SQLBI DAX Patterns — On-Premises SSAS Tabular Reference

> **Target Stack:** SQL Server Analysis Services Tabular (on-prem, compatibility 1200–1600),  
> Power BI Report Server (PBIRS) live connection, SQL Server 2016–2022 DW.  
> No Azure. No Power BI Service. No Microsoft Fabric.

---

## Table of Contents

1. [Time Intelligence Patterns](#1-time-intelligence-patterns)
2. [Semi-Additive Measures](#2-semi-additive-measures)
3. [Many-to-Many Relationships](#3-many-to-many-relationships)
4. [Calculation Groups (Compatibility 1500+)](#4-calculation-groups-compatibility-1500)
5. [Disconnected Tables / Parameter Tables](#5-disconnected-tables--parameter-tables)
6. [Ranking Patterns](#6-ranking-patterns)
7. [Parent-Child Hierarchies](#7-parent-child-hierarchies)
8. [USERELATIONSHIP — Role-Playing Dimensions](#8-userelationship--role-playing-dimensions)
9. [ALLSELECTED Behaviour in Live Connection](#9-allselected-behaviour-in-live-connection)
10. [Variables (VAR) — Performance in On-Prem SSAS](#10-variables-var--performance-in-on-prem-ssas)
11. [Error Handling in DAX](#11-error-handling-in-dax)
12. [Dynamic Security with Active Directory Groups](#12-dynamic-security-with-active-directory-groups)
13. [Aggregation Tables (User-Defined, Compatibility 1500+)](#13-aggregation-tables-user-defined-compatibility-1500)
14. [DAX for Paginated Report Parameters (PBIRS/SSRS)](#14-dax-for-paginated-report-parameters-pbirsssrs)
15. [VertiPaq vs. Formula Engine Optimisation](#15-vertipaq-vs-formula-engine-optimisation)
16. [Measure Quality Checklist](#16-measure-quality-checklist)
17. [DAX Anti-Patterns](#17-dax-anti-patterns)
18. [Measure Organisation Conventions](#18-measure-organisation-conventions)
19. [Bus Matrix Validation in DAX](#19-bus-matrix-validation-in-dax)

---

## 1. Time Intelligence Patterns

### Prerequisites
- A dedicated `DimDate` table marked as a **Date Table** in SSAS (right-click → Mark as Date Table).
- A single active relationship between the fact table date key and `DimDate[DateKey]`.
- All time intelligence functions require a contiguous date table with no gaps.

```dax
-- Year-to-Date Sales
Sales YTD :=
CALCULATE(
    [Total Sales],
    DATESYTD( DimDate[Date] )
)

-- Rolling 12 Months (last 12 complete months, handles mid-month correctly)
Sales R12M :=
CALCULATE(
    [Total Sales],
    DATESINPERIOD(
        DimDate[Date],
        LASTDATE( DimDate[Date] ),
        -12,
        MONTH
    )
)

-- Same Period Last Year
Sales SPLY :=
CALCULATE(
    [Total Sales],
    SAMEPERIODLASTYEAR( DimDate[Date] )
)

-- Year-over-Year % Growth
Sales YoY % :=
VAR CurrentYear = [Total Sales]
VAR PriorYear   = [Sales SPLY]
RETURN
    DIVIDE( CurrentYear - PriorYear, PriorYear )

-- Month-to-Date (robust against partial months in current period)
Sales MTD :=
CALCULATE(
    [Total Sales],
    DATESMTD( DimDate[Date] )
)

-- Quarter-to-Date
Sales QTD :=
CALCULATE(
    [Total Sales],
    DATESQTD( DimDate[Date] )
)

-- Prior Month comparison
Sales Prior Month :=
CALCULATE(
    [Total Sales],
    DATEADD( DimDate[Date], -1, MONTH )
)

-- Fiscal Year YTD (fiscal year starts July 1)
Sales FYTD :=
CALCULATE(
    [Total Sales],
    DATESYTD( DimDate[Date], "6/30" )
)
```

### Handling Future Dates — Blank vs. Zero
```dax
-- Suppress blank future periods cleanly
Sales YTD (No Future) :=
IF(
    ISBLANK( [Total Sales] ),
    BLANK(),
    [Sales YTD]
)
```

---

## 2. Semi-Additive Measures

Use LASTNONBLANK / FIRSTNONBLANK for balances that should not be summed across time.

```dax
-- Closing Balance (e.g., inventory, account balance)
Closing Balance :=
CALCULATE(
    SUM( FactBalance[Balance] ),
    LASTNONBLANK(
        DimDate[Date],
        CALCULATE( SUM( FactBalance[Balance] ) )
    )
)

-- Opening Balance (first non-blank in period)
Opening Balance :=
CALCULATE(
    SUM( FactBalance[Balance] ),
    FIRSTNONBLANK(
        DimDate[Date],
        CALCULATE( SUM( FactBalance[Balance] ) )
    )
)

-- Average Daily Balance (true average, not sum)
Avg Daily Balance :=
AVERAGEX(
    VALUES( DimDate[Date] ),
    CALCULATE( SUM( FactBalance[Balance] ) )
)
```

---

## 3. Many-to-Many Relationships

### Bridge Table Pattern (Compatibility 1200+)
Used for customer-account, employee-project, or product-category many-to-many scenarios.

```dax
-- Many-to-many via bridge table
-- Relationships: FactSales → BridgeCustomerGroup (FilterDirection: Both)
--                BridgeCustomerGroup → DimCustomer (FilterDirection: Single)

Sales M2M :=
CALCULATE(
    [Total Sales],
    TREATAS(
        SUMMARIZE( DimCustomer, DimCustomer[CustomerKey] ),
        BridgeCustomerGroup[CustomerKey]
    )
)
```

### Native M2M (Compatibility 1500 — SSAS 2019/2022)
At compatibility 1500+, you can set a relationship's cross-filter direction to **Both** directly. Avoid this for large tables — it can cause explosive filter expansion. Prefer explicit TREATAS for control.

---

## 4. Calculation Groups (Compatibility 1500+)

> **Supported on:** SSAS Tabular 2019 (CL 1500) and SSAS Tabular 2022 (CL 1600).  
> **Tool required:** Tabular Editor 2.x or 3.x (cannot be authored in SSDT/VS directly in older tooling).  
> **Note:** Calculation Groups are NOT available at compatibility levels below 1500.

### Time Intelligence Calculation Group
```dax
-- Calculation Item: "Actual"
SELECTEDMEASURE()

-- Calculation Item: "YTD"
CALCULATE(
    SELECTEDMEASURE(),
    DATESYTD( DimDate[Date] )
)

-- Calculation Item: "SPLY"
CALCULATE(
    SELECTEDMEASURE(),
    SAMEPERIODLASTYEAR( DimDate[Date] )
)

-- Calculation Item: "YoY %"
VAR Current = CALCULATE( SELECTEDMEASURE() )
VAR Prior   = CALCULATE(
    SELECTEDMEASURE(),
    SAMEPERIODLASTYEAR( DimDate[Date] )
)
RETURN
DIVIDE( Current - Prior, Prior )

-- Calculation Item: "R12M"
CALCULATE(
    SELECTEDMEASURE(),
    DATESINPERIOD(
        DimDate[Date],
        LASTDATE( DimDate[Date] ),
        -12,
        MONTH
    )
)
```

### Format String Expressions (CL 1500+)
```dax
-- On YoY % calculation item, set Format String Expression:
"0.0%"

-- On currency items:
"£#,##0"
```

### Calculation Group Limitations on On-Prem SSAS
| Feature | SSAS 2019 (CL 1500) | SSAS 2022 (CL 1600) |
|---|---|---|
| Calculation Groups | ✅ | ✅ |
| Format String Expressions | ✅ | ✅ |
| Precedence (multiple groups) | ✅ | ✅ |
| Dynamic format strings on base measures | ❌ | ✅ |
| CALCULATION() function | Limited | ✅ |
| Authoring in SSDT | ❌ (need Tabular Editor) | ❌ (need Tabular Editor) |

### Precedence Rules
When using multiple calculation groups (e.g., Time Intelligence + Currency Conversion), set **Precedence** carefully. Lower number = higher precedence (evaluated last in filter context). Always test with `SELECTEDMEASURENAME()` for conditional logic:

```dax
-- Calculation Item: apply only to measures that are currency-based
IF(
    SELECTEDMEASURENAME() IN { "Total Sales", "Total Cost", "Gross Profit" },
    CALCULATE(
        SELECTEDMEASURE(),
        SAMEPERIODLASTYEAR( DimDate[Date] )
    ),
    SELECTEDMEASURE()
)
```

---

## 5. Disconnected Tables / Parameter Tables

Used for what-if analysis, slicer-driven parameters, and dynamic measure switching.

```dax
-- DimScenario table (not related to any fact — imported as a standalone table)
-- Columns: ScenarioKey (1,2,3), ScenarioName ("Budget","Forecast","Actual")

Selected Scenario Sales :=
VAR SelectedKey = SELECTEDVALUE( DimScenario[ScenarioKey], 3 ) -- default: Actual
RETURN
SWITCH(
    SelectedKey,
    1, [Budget Sales],
    2, [Forecast Sales],
    3, [Actual Sales],
    [Actual Sales]
)

-- Dynamic measure switching (no calculation groups available at CL 1200)
Dynamic KPI :=
SWITCH(
    SELECTEDVALUE( DimKPI[KPIKey] ),
    1, [Total Sales],
    2, [Total Cost],
    3, [Gross Profit],
    4, [Units Sold],
    BLANK()
)
```

---

## 6. Ranking Patterns

```dax
-- Rank products by sales within current filter context
Product Sales Rank :=
IF(
    ISBLANK( [Total Sales] ),
    BLANK(),
    RANKX(
        ALLSELECTED( DimProduct[ProductName] ),
        [Total Sales],
        ,
        DESC,
        DENSE
    )
)

-- Top N filter measure (use in visual-level filter: [Is Top N] = 1)
Is Top N :=
VAR N = SELECTEDVALUE( DimTopN[N], 10 )
RETURN
IF( [Product Sales Rank] <= N, 1, 0 )

-- Percentile rank
Product Sales Percentile :=
DIVIDE(
    RANKX( ALLSELECTED( DimProduct[ProductName] ), [Total Sales],, ASC, DENSE ) - 1,
    COUNTROWS( ALLSELECTED( DimProduct[ProductName] ) ) - 1
)
```

---

## 7. Parent-Child Hierarchies

SSAS Tabular does not natively support ragged/recursive hierarchies. Use PATH functions to flatten them at model refresh time (computed columns).

```dax
-- Computed columns on DimEmployee (add these as model computed columns)
EmployeePath    = PATH( DimEmployee[EmployeeKey], DimEmployee[ManagerKey] )
EmployeeDepth   = PATHLENGTH( DimEmployee[EmployeePath] )
Level1Key       = PATHITEM( DimEmployee[EmployeePath], 1, INTEGER )
Level2Key       = PATHITEM( DimEmployee[EmployeePath], 2, INTEGER )
Level3Key       = PATHITEM( DimEmployee[EmployeePath], 3, INTEGER )
Level4Key       = PATHITEM( DimEmployee[EmployeePath], 4, INTEGER )

Level1Name      = LOOKUPVALUE( DimEmployee[EmployeeName], DimEmployee[EmployeeKey], DimEmployee[Level1Key] )
Level2Name      = LOOKUPVALUE( DimEmployee[EmployeeName], DimEmployee[EmployeeKey], DimEmployee[Level2Key] )

-- Measure: rollup all subordinates of selected employee
Subordinate Sales :=
CALCULATE(
    [Total Sales],
    FILTER(
        DimEmployee,
        PATHCONTAINS( DimEmployee[EmployeePath], MAX( DimEmployee[EmployeeKey] ) )
    )
)
```

---

## 8. USERELATIONSHIP — Role-Playing Dimensions

### Pattern: Single Date Dimension with Multiple Fact Date Roles
SSAS Tabular allows only **one active relationship** between any two tables. For role-playing date dimensions, create **multiple inactive relationships** and activate them with `USERELATIONSHIP`.

**Model setup (in SSDT/Tabular Editor):**
- `FactSales[OrderDateKey]` → `DimDate[DateKey]` **(ACTIVE)**
- `FactSales[ShipDateKey]`  → `DimDate[DateKey]` **(INACTIVE)**
- `FactSales[DueDateKey]`   → `DimDate[DateKey]` **(INACTIVE)**

```dax
-- Sales by Ship Date
Sales by Ship Date :=
CALCULATE(
    [Total Sales],
    USERELATIONSHIP( FactSales[ShipDateKey], DimDate[DateKey] )
)

-- Sales by Due Date
Sales by Due Date :=
CALCULATE(
    [Total Sales],
    USERELATIONSHIP( FactSales[DueDateKey], DimDate[DateKey] )
)

-- YTD by Ship Date (combine USERELATIONSHIP with time intelligence)
Sales Ship YTD :=
CALCULATE(
    [Total Sales],
    USERELATIONSHIP( FactSales[ShipDateKey], DimDate[DateKey] ),
    DATESYTD( DimDate[Date] )
)
```

### Role-Playing with Multiple Date Dimensions (Alternative Pattern)
For complex scenarios, maintain separate **view-based** date dimension tables in the model:
`DimOrderDate`, `DimShipDate`, `DimDueDate` — all sourced from the same `DimDate` query but with aliases. This allows each to have its own active relationship and independent hierarchies, avoiding USERELATIONSHIP entirely. Preferred for models with heavy time intelligence usage on multiple date roles.

```dax
-- With separate dimension tables — no USERELATIONSHIP needed
Sales Ship YTD (Alt) :=
CALCULATE(
    [Total Sales],
    DATESYTD( DimShipDate[Date] )   -- uses active relationship
)
```

### Pitfalls with USERELATIONSHIP
- `USERELATIONSHIP` **deactivates** the currently active relationship for the scope of the CALCULATE block.
- Cannot use time intelligence functions that rely on the active relationship simultaneously without combining them in the same CALCULATE modifier list.
- In SSAS on-prem, `USERELATIONSHIP` with bidirectional cross-filter can produce unexpected results — always set cross-filter to **Single** on inactive relationships.

---

## 9. ALLSELECTED Behaviour in Live Connection

> **Critical for PBIRS live connection reports.** Behaviour differs subtly from Power BI Desktop (import mode).

### Context Propagation Rules
| Function | What it clears | Live Connection behaviour |
|---|---|---|
| `ALL( Table )` | All filters on table from any source | ✅ Consistent |
| `ALL( Column )` | All filters on that column | ✅ Consistent |
| `ALLSELECTED( Column )` | Clears inner filters, preserves slicer/page filters | ⚠️ See notes below |
| `ALLEXCEPT( Table, Col )` | All filters except named columns | ✅ Consistent |

### ALLSELECTED in Live Connection — Known Behaviours
1. **ALLSELECTED respects the outermost query filter context** — in PBIRS live connection, the outer context is the MDX/DAX query sent by the report. This is generally equivalent to "what is selected in slicers on the page."
2. **ALLSELECTED does NOT behave identically across visual types** — matrix visuals that generate sub-selects may produce a different ALLSELECTED scope than card visuals. Always test ranking measures in matrix visuals specifically.
3. **Report page filters count as outer context** — a page-level filter applied in PBIRS on a live connection report is included in the outer ALLSELECTED scope, which is the correct behaviour.
4. **No drillthrough context difference** — ALLSELECTED in a drillthrough target page uses the drillthrough filter as outer context, same as Power BI Service.

```dax
-- Safe percentage-of-total using ALLSELECTED (works correctly in live connection matrix)
Sales % of Total :=
DIVIDE(
    [Total Sales],
    CALCULATE( [Total Sales], ALLSELECTED( DimProduct[Category] ) )
)

-- Ranking within slicer selection (ALLSELECTED ensures rank resets to 1 within sliced set)
Product Rank (Sliced) :=
RANKX(
    ALLSELECTED( DimProduct[ProductName] ),
    [Total Sales],
    ,
    DESC,
    DENSE
)

-- AVOID: using ALLSELECTED on a table with many inactive relationships
-- It can evaluate slowly on SSAS on-prem due to FE overhead
-- PREFER: ALLSELECTED on specific columns
```

### ALLSELECTED vs. ALLEXCEPT — When to Use Each
- Use `ALLSELECTED( Column )` for **visual-relative percentage of total** — you want slicers to restrict the denominator.
- Use `ALL( Column )` for **absolute percentage of grand total** — slicers should NOT affect the denominator.
- Use `ALLEXCEPT` when you need to **preserve specific dimension filters** while removing all others in a complex filter context.

---

## 10. Variables (VAR) — Performance in On-Prem SSAS

### How Variables Work in the VertiPaq Engine
In on-premises SSAS Tabular, a `VAR` expression is evaluated **once** when first referenced and its result is **cached for the duration of the measure evaluation**. This is fundamentally different from subexpressions written inline, which may be re-evaluated multiple times.

```dax
-- ❌ AVOID: inline subexpressions evaluated multiple times
Gross Margin % (Slow) :=
DIVIDE(
    SUM( FactSales[Revenue] ) - SUM( FactSales[Cost] ),
    SUM( FactSales[Revenue] )
)
-- SUM(FactSales[Revenue]) is evaluated TWICE — two SE queries

-- ✅ PREFER: VAR caches the result — single SE query
Gross Margin % (Fast) :=
VAR Revenue = SUM( FactSales[Revenue] )
VAR Cost    = SUM( FactSales[Cost] )
RETURN
    DIVIDE( Revenue - Cost, Revenue )
```

### VAR and Filter Context
A critical rule: **VAR captures the filter context at the point it is declared**, not at the point it is used in RETURN. This enables clean "save and restore" filter context patterns:

```dax
-- Classic context save pattern
Prior Year Sales :=
VAR CurrentYearSales = [Total Sales]          -- evaluated in current context
VAR PYSales = CALCULATE(
    [Total Sales],
    SAMEPERIODLASTYEAR( DimDate[Date] )
)
RETURN
    DIVIDE( CurrentYearSales - PYSales, PYSales )
```

### Performance Guidelines for On-Prem SSAS VAR Usage
1. **Always use VAR for measures called more than once** in a RETURN expression.
2. **VAR across CALCULATE boundaries:** A VAR declared outside a CALCULATE cannot be re-evaluated inside it — this is intentional and correct. If you need a value computed inside a modified filter context, compute it inside the CALCULATE, assign to a VAR, then use it.
3. **Large SUMMARIZE results in VARs**: Storing large tables in VARs (e.g., `VAR T = SUMMARIZE(...)` over millions of rows) can consume significant VertiPaq memory. Keep table VARs as filtered as possible.
4. **VARs are not materialised globally** — they are scoped to the measure evaluation instance. There is no cross-query VAR caching.

```dax
-- Pattern: conditional calculation with VAR to avoid double evaluation
Conditional Growth :=
VAR Sales    = [Total Sales]
VAR PYSales  = CALCULATE( [Total Sales], SAMEPERIODLASTYEAR( DimDate[Date] ) )
VAR Growth   = DIVIDE( Sales - PYSales, PYSales )
RETURN
    IF( PYSales = 0 || ISBLANK( PYSales ), BLANK(), Growth )
```

---

## 11. Error Handling in DAX

### Division by Zero
```dax
-- ✅ Always use DIVIDE() — never use "/" operator for measures
Safe Ratio :=
DIVIDE( [Numerator Measure], [Denominator Measure] )
-- Returns BLANK() when denominator is 0 or BLANK

-- DIVIDE with explicit alternate result
Safe Ratio (Zero Default) :=
DIVIDE( [Numerator Measure], [Denominator Measure], 0 )

-- ❌ AVOID — will show error in report
Unsafe Ratio := [Numerator Measure] / [Denominator Measure]
```

### IFERROR and ISERROR
```dax
-- IFERROR wraps any expression and returns alternate on error
Safe Lookup :=
IFERROR(
    LOOKUPVALUE(
        DimProduct[StandardCost],
        DimProduct[ProductKey],
        MAX( FactSales[ProductKey] )
    ),
    0
)

-- ISERROR for conditional logic (slightly more expensive than IFERROR)
Has Valid Price :=
IF(
    ISERROR( LOOKUPVALUE( DimProduct[Price], DimProduct[ProductKey], MAX( FactSales[ProductKey] ) ) ),
    FALSE,
    TRUE
)

-- ISBLANK — preferred over = BLANK() for measures
Has Sales :=
NOT ISBLANK( [Total Sales] )
```

### Handling Missing Dimension Members
```dax
-- Return BLANK (not zero) when no matching rows — cleaner in visuals
Total Sales (Clean) :=
IF(
    COUNTROWS( FactSales ) = 0,
    BLANK(),
    SUM( FactSales[SalesAmount] )
)

-- Substitute unknown member label
Product Category Safe :=
COALESCE( SELECTEDVALUE( DimProduct[Category] ), "Unknown" )
```

### COALESCE (Compatibility 1550+ / SSAS 2022)
```dax
-- COALESCE returns first non-blank/non-null value (cleaner than nested IF ISBLANK)
Effective Price :=
COALESCE( [Override Price], [Standard Price], 0 )
```

> **Note:** `COALESCE` was introduced at model compatibility 1550 (SSAS 2022 or later cumulative updates). For SSAS 2019 (CL 1500), use nested `IF( ISBLANK(...), ..., ... )` instead.

---

## 12. Dynamic Security with Active Directory Groups

### On-Prem SSAS: USERNAME() vs. USERPRINCIPALNAME()
| Function | Returns | On-Prem SSAS Behaviour |
|---|---|---|
| `USERNAME()` | `DOMAIN\username` (NetBIOS format) | ✅ Correct for Windows Auth |
| `USERPRINCIPALNAME()` | `user@domain.com` (UPN format) | ⚠️ Returns same as USERNAME() on-prem in most cases; may return BLANK() |

> **On-prem SSAS always uses `USERNAME()`** — this returns the Windows identity in `DOMAIN\username` format. `USERPRINCIPALNAME()` was designed for Azure AS / Power BI Service and is unreliable on-prem. Always use `USERNAME()` for SSAS on-premises security.

### Row-Level Security with AD Group Membership

**Option A: User table with explicit assignments**
```dax
-- DimUserSecurity table: UserName (DOMAIN\user), RegionKey
[RLS Region Filter] :=
LOOKUPVALUE(
    DimUserSecurity[RegionKey],
    DimUserSecurity[UserName],
    USERNAME()
)

-- Role DAX filter on DimRegion table:
DimRegion[RegionKey] = LOOKUPVALUE(
    DimUserSecurity[RegionKey],
    DimUserSecurity[UserName],
    USERNAME()
)
```

**Option B: AD Group-based security (group → permission mapping)**
```dax
-- DimSecurityGroup table: GroupName (DOMAIN\groupname), RegionKey
-- DimUserGroup bridge table: UserName, GroupName (populated by ETL from AD)

-- Role DAX filter on DimRegion:
[RegionKey] IN
    CALCULATETABLE(
        VALUES( DimSecurityGroup[RegionKey] ),
        TREATAS(
            CALCULATETABLE(
                VALUES( DimUserGroup[GroupName] ),
                DimUserGroup[UserName] = USERNAME()
            ),
            DimSecurityGroup[GroupName]
        )
    )
```

### Dynamic RLS — Separate Security Table per Role
```dax
-- Universal RLS filter (handles both user-specific and group-based assignments)
-- SecurityPermission table: Principal (DOMAIN\user or DOMAIN\group), EntityKey, EntityType

-- Role filter on DimBranch:
DimBranch[BranchKey] IN
    CALCULATETABLE(
        VALUES( SecurityPermission[EntityKey] ),
        SecurityPermission[Principal] = USERNAME(),
        SecurityPermission[EntityType] = "Branch"
    )
```

### AD Group Refresh Strategy
The bridge table `DimUserGroup` must be refreshed by an SSIS/SQL Agent job that queries Active Directory. Use `System.DirectoryServices.DirectorySearcher` in a Script Task or `OPENQUERY` via a linked server to AD LDS. Refresh frequency should match AD group change frequency — typically nightly.

### Testing RLS in SSAS
In SSMS, connect to SSAS and use:
```
-- Effective permissions test
EXECUTE AS USER = 'DOMAIN\testuser'
```
Or in DAX Studio, use the **Roles** connection option to impersonate a specific Windows user.

---

## 13. Aggregation Tables (User-Defined, Compatibility 1500+)

> **Distinct from Power BI Premium automatic aggregations.** On-prem SSAS 1500+ supports **user-defined aggregations** configured manually via Tabular Editor or SSMS scripting. There are no automatic aggregations in on-prem SSAS.

### When to Use Aggregation Tables
- Fact tables exceeding **50–100 million rows** where common summary queries (by month, by region, by category) run slowly.
- Queries that consistently group by 2–3 dimensions at a high grain.
- When VertiPaq memory pressure is high and you want to reduce scan volume.

### Setting Up an Aggregation Table

**Step 1: Create the aggregation table in the DW**
```sql
-- Materialized aggregation in SQL Server DW
CREATE TABLE dbo.FactSales_Agg_MonthRegion (
    DateMonthKey    INT         NOT NULL,
    RegionKey       INT         NOT NULL,
    ProductCatKey   INT         NOT NULL,
    TotalSales      DECIMAL(18,2) NOT NULL,
    TotalCost       DECIMAL(18,2) NOT NULL,
    TotalUnits      INT         NOT NULL,
    CONSTRAINT PK_FactSales_Agg PRIMARY KEY (DateMonthKey, RegionKey, ProductCatKey)
);

-- Populate (refreshed by SSIS/SQL Agent nightly)
INSERT INTO dbo.FactSales_Agg_MonthRegion
SELECT
    d.MonthKey          AS DateMonthKey,
    f.RegionKey,
    p.CategoryKey       AS ProductCatKey,
    SUM(f.SalesAmount)  AS TotalSales,
    SUM(f.CostAmount)   AS TotalCost,
    SUM(f.Units)        AS TotalUnits
FROM dbo.FactSales f
JOIN dbo.DimDate    d ON f.DateKey    = d.DateKey
JOIN dbo.DimProduct p ON f.ProductKey = p.ProductKey
GROUP BY d.MonthKey, f.RegionKey, p.CategoryKey;
```

**Step 2: Import the aggregation table into the SSAS Tabular model**

**Step 3: Configure aggregation via Tabular Editor (BIM metadata)**

In Tabular Editor, on the `FactSales_Agg_MonthRegion` table:
- Set **Table Behaviour → Column mappings** to `Summarize` for each measure column.
- Set **Alternative Source** pointer from detail table columns to aggregation columns.
- The engine will automatically hit the aggregation when queries match the granularity.

```json
// Partial BIM annotation (Tabular Editor JSON)
{
  "name": "FactSales_Agg_MonthRegion",
  "isHidden": true,
  "partitions": [...],
  "columns": [
    {
      "name": "TotalSales",
      "dataType": "decimal",
      "summarizeBy": "sum",
      "alternateOf": {
        "tableName": "FactSales",
        "columnName": "SalesAmount",
        "summarization": "Sum"
      }
    }
  ]
}
```

### Aggregation Table Design Rules
1. The aggregation table must be **hidden** in the model — it should never appear in report field lists.
2. Relationships from aggregation table to dimensions must mirror the detail fact table relationships.
3. The aggregation grain must be **coarser than or equal to** the detail fact table grain.
4. Monitor whether queries actually use the aggregation via **DAX Studio Server Timings** — look for `VertiPaq Scan` hitting the aggregation table name.

---

## 14. DAX for Paginated Report Parameters (PBIRS/SSRS)

When PBIRS Paginated Reports (SSRS) connect to a Tabular model via MDX or DAX dataset, specific patterns apply for parameterised queries.

### DAX Dataset for SSRS Paginated Reports
SSRS paginated reports can use **DAX as dataset query language** when the data source is an SSAS Tabular model.

```dax
-- DAX query for an SSRS dataset (sales by product for a given date range)
-- @StartDate and @EndDate are SSRS report parameters passed as DAX variables
EVALUATE
CALCULATETABLE(
    SUMMARIZECOLUMNS(
        DimProduct[ProductName],
        DimProduct[Category],
        DimDate[CalendarYear],
        DimDate[MonthName],
        "Total Sales",   [Total Sales],
        "Total Cost",    [Total Cost],
        "Gross Profit",  [Gross Profit],
        "Units Sold",    [Units Sold]
    ),
    DimDate[Date] >= DATE(@StartYear, @StartMonth, 1),
    DimDate[Date] <= DATE(@EndYear, @EndMonth, 31)
)
ORDER BY DimDate[CalendarYear], DimDate[MonthName], DimProduct[ProductName]
```

> **SSRS parameter binding note:** SSRS report parameters map to DAX query parameters using `@ParamName` syntax. In the SSRS dataset query designer, switch to **text mode** and write the DAX query directly. Use integer parameters for year/month to avoid date parsing issues.

### MDX for Paginated Reports (Alternative)
SSRS has longer history with MDX against SSAS. For complex hierarchical reports (ragged hierarchies, subtotals), MDX is often more reliable than DAX for paginated output:

```mdx
-- MDX dataset for SSRS (product sales by category with subtotals)
SELECT
    NON EMPTY {
        [Measures].[Total Sales],
        [Measures].[Total Cost],
        [Measures].[Gross Profit]
    } ON COLUMNS,
    NON EMPTY {
        [DimProduct].[Category].[Category].MEMBERS *
        [DimProduct].[ProductName].[ProductName].MEMBERS
    } ON ROWS
FROM [SalesTabularModel]
WHERE (
    [DimDate].[CalendarYear].&[@StartYear]
)
```

### Parameterised Measure Selection
```dax
-- SSRS report with a @MeasureSelector parameter (values: 1=Sales, 2=Cost, 3=Profit)
-- DAX dataset query:
EVALUATE
CALCULATETABLE(
    ADDCOLUMNS(
        SUMMARIZE( DimProduct, DimProduct[ProductName], DimProduct[Category] ),
        "Selected Value",
        SWITCH(
            @MeasureSelector,
            1, [Total Sales],
            2, [Total Cost],
            3, [Gross Profit],
            [Total Sales]
        )
    )
)
```

### Cascading Parameters Pattern
```dax
-- Dataset 1: Available years (for @Year parameter dropdown)
EVALUATE
SUMMARIZECOLUMNS(
    DimDate[CalendarYear],
    "Has Data", CALCULATE( COUNTROWS( FactSales ) )
)
ORDER BY DimDate[CalendarYear] ASC

-- Dataset 2: Available regions for selected year (for @Region parameter dropdown)
EVALUATE
CALCULATETABLE(
    SUMMARIZECOLUMNS( DimRegion[RegionName], DimRegion[RegionKey] ),
    DimDate[CalendarYear] = @SelectedYear
)
ORDER BY DimRegion[RegionName]
```

---

## 15. VertiPaq vs. Formula Engine Optimisation

### Architecture Overview
SSAS Tabular's DAX engine has two components:

| Component | Abbreviation | Role | Parallelism |
|---|---|---|---|
| **Storage Engine** | SE | Scans compressed VertiPaq columnar data; executes simple aggregations | ✅ Multi-threaded |
| **Formula Engine** | FE | Evaluates DAX logic, iterators (X functions), context transitions | ❌ Single-threaded |

**Golden rule:** Push as much work as possible to the SE. If the FE is the bottleneck, the query will not benefit from additional CPU cores.

### Diagnosing SE vs. FE Bottleneck — DAX Studio Server Timings

1. Open **DAX Studio**, connect to SSAS on-prem instance.
2. Enable **Server Timings** (Query menu → Server Timings).
3. Run the measure/query.
4. Read the Server Timings pane:
   - **SE CPU**: time spent in Storage Engine scans.
   - **FE**: time spent in Formula Engine logic.
   - **Total**: wall-clock time.
   - **SE Queries**: number of storage engine cache hits vs. misses.
   - **Cache Hits**: high cache hits = good; measure has benefited from previous query caching.

```
Example Server Timings output (good — SE-dominant):
  Total:    450ms
  SE CPU:   420ms  (93% — SE doing the work, parallelised)
  FE:        30ms
  SE Queries: 4 (3 cache hits, 1 cache miss)

Example Server Timings output (bad — FE-dominant):
  Total:   3,200ms
  SE CPU:    80ms
  FE:      3,120ms  (98% — all work in single-threaded FE)
  SE Queries: 847   (excessive SE callouts from iterator)
```

### SE Query Types
DAX Studio shows individual SE queries in the **Query Plan** tab:
- **VertiPaq Scan**: direct columnar scan — efficient, parallelised.
- **Lookup**: key-value lookup — efficient.
- **Join**: cross-table join within SE — efficient up to moderate cardinality.
- **Callback DataID**: SE calls back to FE for a computed value — **expensive**, breaks parallelism.

> **CallbackDataID is the primary enemy of SE performance.** It occurs when the SE cannot evaluate an expression itself and must call the FE for each row. Common causes: nested iterators, complex IF logic inside SUMX/FILTER, measures referencing other measures inside row context.

### Patterns That Force FE (Avoid or Refactor)
```dax
-- ❌ FILTER with measure reference — forces FE
Bad Filter :=
CALCULATE(
    [Total Sales],
    FILTER( DimProduct, [Product Margin %] > 0.2 )  -- measure inside FILTER → FE per row
)

-- ✅ FILTER with column expression — stays in SE
Good Filter :=
CALCULATE(
    [Total Sales],
    FILTER( DimProduct, DimProduct[StandardMargin] > 0.2 )  -- column → SE scan
)

-- ❌ SUMX over large table with complex per-row logic
Slow SUMX :=
SUMX(
    FactSales,
    FactSales[Qty] * LOOKUPVALUE( DimProduct[Price], DimProduct[ProductKey], FactSales[ProductKey] )
)
-- Better: materialise [Price] as a column in FactSales via model relationship or DW ETL

-- ✅ Prefer SUM over pre-calculated column
Fast SUM := SUM( FactSales[PreCalcRevenue] )
```

### VertiPaq Compression Tips (Improve SE Speed)
1. **Sort tables by the column with highest cardinality** (in partition query ORDER BY) — VertiPaq run-length encodes sorted columns more efficiently.
2. **Reduce column cardinality** where possible — avoid storing full timestamps when date-only is sufficient; avoid free-text columns in the model.
3. **Avoid duplicating high-cardinality columns** across fact tables — use keys and join at query time via relationships.
4. **Use integer keys** rather than string keys — integer dictionary encoding is more compact.

### Key DMV Queries for VertiPaq Analysis
```dax
-- Memory usage by table/column (run in DAX Studio against SSAS instance)
SELECT
    CATALOG_NAME,
    CUBE_NAME,
    MEASURE_GROUP_NAME AS TableName,
    ATTRIBUTE_NAME     AS ColumnName,
    ROWS_COUNT,
    USED_SIZE          AS MemoryBytes,
    DICTIONARY_SIZE    AS DictionaryBytes
FROM $SYSTEM.DISCOVER_STORAGE_TABLE_COLUMN_SEGMENTS
ORDER BY USED_SIZE DESC
```

---

## 16. Measure Quality Checklist

Before deploying any measure to production SSAS:

- [ ] Uses `DIVIDE()` — not `/` operator — for all divisions
- [ ] Returns `BLANK()` (not 0) when there is no data — avoids misleading zero in visuals
- [ ] Tested in a matrix visual with row/column/page/slicer filters active
- [ ] Tested with no filters (grand total row) — grand total formula is correct
- [ ] VAR used for any subexpression referenced more than once
- [ ] No `FILTER( AllTable, [Measure] > x )` — replaced with column filter or KEEPFILTERS
- [ ] Format string set explicitly (not left as "Auto")
- [ ] Description populated in the measure properties
- [ ] Assigned to correct Display Folder
- [ ] Tested with DAX Studio Server Timings — FE time is not dominant
- [ ] RLS-sensitive measures tested with impersonation in DAX Studio
- [ ] Time intelligence measures tested at year/quarter/month/day grain

---

## 17. DAX Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| `SUM(Col) / SUM(Col2)` | Division by zero error | Use `DIVIDE()` |
| `FILTER( Table, [Measure] > x )` | Forces FE, one SE callout per row | Filter on a column, not a measure |
| `CALCULATE( X, ALL( Table ) )` inside an iterator | Removes all filters globally; unexpected for the user | Use `REMOVEFILTERS()` with scope, or `ALLEXCEPT` |
| `COUNTROWS( FILTER( Table, condition ) )` | Iterates entire table | Use `CALCULATE( COUNTROWS(Table), condition )` |
| Nested `CALCULATE` with no additive modifiers | Confusing; inner CALCULATE often redundant | Flatten into one CALCULATE |
| `IF( SUM() = 0, BLANK(), ... )` | `SUM()` returns `BLANK()` not 0 when no rows — condition may fail | Use `IF( ISBLANK( [Measure] ), BLANK(), ... )` or `IF( [Measure] = 0 || ISBLANK([Measure]), ...)` |
| `RELATED()` inside a measure | Only valid in row context — throws error in filter context | Use `LOOKUPVALUE()` or restructure with TREATAS |
| Calculated columns for values derivable by measures | Wastes VertiPaq memory; re-evaluated on every process | Use measures; only use calculated columns for grouping/slicing attributes |
| Unused columns imported into model | Bloats VertiPaq memory; slows SSAS processing | Remove from model or mark as Hidden and exclude from partitions |
| `FORMAT()` inside a measure body | Returns a text string; cannot be used in numeric aggregations | Apply format strings via Format property, not FORMAT() function in calculation |

---

## 18. Measure Organisation Conventions

### Display Folder Structure
```
📁 [Key Metrics]
    Total Sales
    Total Cost
    Gross Profit
    Gross Margin %
📁 [Time Intelligence]
    Sales YTD
    Sales MTD
    Sales SPLY
    Sales YoY %
    Sales R12M
📁 [Ratios & Rates]
    Avg Transaction Value
    Conversion Rate
    Return Rate %
📁 [Inventory]
    Closing Stock
    Opening Stock
    Stock Turns
📁 [_Debug]          ← hidden folder; visible to developers only
    _Debug Row Count
    _Debug Context Test
```

### Naming Conventions
- **Plain name** for base measures: `Total Sales`, `Units Sold`
- **Suffix** for time intelligence variants: `Sales YTD`, `Sales SPLY`, `Sales YoY %`
- **Prefix underscore** for helper/debug measures: `_Sales Base`, `_Has Filter`
- **No abbreviations** in public measure names — spell out in full
- **Consistent capitalisation**: Title Case for all measure names

### Documentation via Description Property
Every production measure must have a populated Description:
```
Total Sales
  Description: "Sum of SalesAmount from FactSales. Includes all channels.
                Excludes cancelled orders (OrderStatus = 'Cancelled' filtered in partition).
                Last reviewed: 2024-01-15 by [Author]."
```

---

## 19. Bus Matrix Validation in DAX

The bus matrix defines which dimensions are conformed across fact tables. Use these measures to verify conformance in the model:

```dax
-- Validate that DimProduct joins correctly to both FactSales and FactReturns
Product in Sales Count :=
CALCULATE(
    DISTINCTCOUNT( FactSales[ProductKey] ),
    ALLEXCEPT( DimProduct, DimProduct[ProductKey] )
)

Product in Returns Count :=
CALCULATE(
    DISTINCTCOUNT( FactReturns[ProductKey] ),
    ALLEXCEPT( DimProduct, DimProduct[ProductKey] )
)

-- Orphan check: products in fact with no dimension match
-- Run in DAX Studio to return a table
EVALUATE
FILTER(
    ADDCOLUMNS(
        VALUES( FactSales[ProductKey] ),
        "InDimProduct", CALCULATE( COUNTROWS( DimProduct ), DimProduct[ProductKey] = EARLIER( FactSales[ProductKey] ) )
    ),
    [InDimProduct] = 0
)
```
