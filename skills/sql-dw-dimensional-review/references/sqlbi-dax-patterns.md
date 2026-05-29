 SQLBI DAX Patterns — On-Premises SSAS Tabular Reference

> **Stack:** SSAS Tabular on-prem (CL 1200–1600), PBIRS live connection, SQL Server 2016–2022.
> No Azure / Power BI Service / Fabric.

---

## 1. Time Intelligence Patterns

**Prerequisites:** `Calendar` marked as Date Table; single active relationship to fact date key; contiguous date range (no gaps).

```dax
Sales YTD        := CALCULATE( [Total Sales], DATESYTD( Calendar[Date] ) )
Sales MTD        := CALCULATE( [Total Sales], DATESMTD( Calendar[Date] ) )
Sales QTD        := CALCULATE( [Total Sales], DATESQTD( Calendar[Date] ) )
Sales SPLY       := CALCULATE( [Total Sales], SAMEPERIODLASTYEAR( Calendar[Date] ) )
Sales Prior Month := CALCULATE( [Total Sales], DATEADD( Calendar[Date], -1, MONTH ) )
Sales FYTD       := CALCULATE( [Total Sales], DATESYTD( Calendar[Date], "6/30" ) ) -- fiscal end June

-- Rolling 12 months
Sales R12M :=
CALCULATE( [Total Sales], DATESINPERIOD( Calendar[Date], LASTDATE( Calendar[Date] ), -12, MONTH ) )

-- YoY % Growth
Sales YoY % :=
VAR Curr = [Total Sales]
VAR Prev = [Sales SPLY]
RETURN DIVIDE( Curr - Prev, Prev )

-- Suppress blank future periods
Sales YTD (No Future) := IF( ISBLANK( [Total Sales] ), BLANK(), [Sales YTD] )
```

---

## 2. Semi-Additive Measures

Use `LASTNONBLANK`/`FIRSTNONBLANK` for balances that must not be summed across time.

```dax
Closing Balance :=
CALCULATE( SUM( Balance[Balance] ),
    LASTNONBLANK( Calendar[Date], CALCULATE( SUM( Balance[Balance] ) ) ) )

Opening Balance :=
CALCULATE( SUM( Balance[Balance] ),
    FIRSTNONBLANK( Calendar[Date], CALCULATE( SUM( Balance[Balance] ) ) ) )

Avg Daily Balance :=
AVERAGEX( VALUES( Calendar[Date] ), CALCULATE( SUM( Balance[Balance] ) ) )
```

---

## 3. Many-to-Many Relationships

### Bridge Table (CL 1200+)
Relationships: `Sales → BridgeCustomerGroup` (Both), `BridgeCustomerGroup → DimCustomer` (Single).

```dax
Sales M2M :=
CALCULATE(
    [Total Sales],
    TREATAS(
        SUMMARIZE( DimCustomer, DimCustomer[CustomerKey] ),
        BridgeCustomerGroup[CustomerKey]
    )
)
```

### Native M2M (CL 1500+)
Set relationship cross-filter to **Both** directly. Avoid on large tables (explosive filter expansion). Prefer `TREATAS` for control.

---

## 4. Calculation Groups (CL 1500+)

> Requires `TabularEditor.exe` (Tabular Editor 2.x). Not available in SSDT. Not available below CL 1500.

```dax
-- Calculation Item: "Actual"
SELECTEDMEASURE()

-- Calculation Item: "YTD"
CALCULATE( SELECTEDMEASURE(), DATESYTD( Calendar[Date] ) )

-- Calculation Item: "SPLY"
CALCULATE( SELECTEDMEASURE(), SAMEPERIODLASTYEAR( Calendar[Date] ) )

-- Calculation Item: "YoY %"
VAR Curr = CALCULATE( SELECTEDMEASURE() )
VAR Prev = CALCULATE( SELECTEDMEASURE(), SAMEPERIODLASTYEAR( Calendar[Date] ) )
RETURN DIVIDE( Curr - Prev, Prev )

-- Calculation Item: "R12M"
CALCULATE( SELECTEDMEASURE(), DATESINPERIOD( Calendar[Date], LASTDATE( Calendar[Date] ), -12, MONTH ) )

-- Format String Expression on YoY % item:
"0.0%"

-- Conditional: apply only to named measures (use SELECTEDMEASURENAME())
IF(
    SELECTEDMEASURENAME() IN { "Total Sales", "Total Cost", "Gross Profit" },
    CALCULATE( SELECTEDMEASURE(), SAMEPERIODLASTYEAR( Calendar[Date] ) ),
    SELECTEDMEASURE()
)
```

| Feature | CL 1500 (2019) | CL 1600 (2022) |
|---|---|---|
| Calculation Groups | ✅ | ✅ |
| Format String Expressions | ✅ | ✅ |
| Dynamic format on base measures | ❌ | ✅ |
| CALCULATION() function | Limited | ✅ |
| Authoring in SSDT | ❌ | ❌ |

**Precedence:** Lower number = higher priority (evaluated last). When using multiple groups (e.g. Time Intelligence + Currency), set Precedence carefully and test with `SELECTEDMEASURENAME()`.

---

## 5. Disconnected Tables / Parameter Tables

```dax
-- DimScenario: ScenarioKey (1/2/3), ScenarioName. Not related to any fact table.
Selected Scenario Sales :=
VAR Sel = SELECTEDVALUE( DimScenario[ScenarioKey], 3 ) -- default: Actual
RETURN SWITCH( Sel,
    1, [Budget Sales], 2, [Forecast Sales], 3, [Actual Sales], [Actual Sales] )

-- Dynamic measure switch (use when calc groups unavailable at CL 1200)
Dynamic KPI :=
SWITCH( SELECTEDVALUE( DimKPI[KPIKey] ),
    1, [Total Sales], 2, [Total Cost], 3, [Gross Profit], 4, [Units Sold], BLANK() )
```

---

## 6. Ranking Patterns

```dax
Product Sales Rank :=
IF( ISBLANK( [Total Sales] ), BLANK(),
    RANKX( ALLSELECTED( DimProduct[ProductName] ), [Total Sales],, DESC, DENSE ) )

-- Top N filter (use in visual-level filter: [Is Top N] = 1)
Is Top N :=
VAR N = SELECTEDVALUE( DimTopN[N], 10 )
RETURN IF( [Product Sales Rank] <= N, 1, 0 )
```

---

## 7. Parent-Child Hierarchies

SSAS Tabular has no native ragged/recursive hierarchy support. Flatten using PATH computed columns at model refresh.

```dax
-- Computed columns on DimEmployee:
EmployeePath  = PATH( DimEmployee[EmployeeKey], DimEmployee[ManagerKey] )
EmployeeDepth = PATHLENGTH( DimEmployee[EmployeePath] )
Level1Key     = PATHITEM( DimEmployee[EmployeePath], 1, INTEGER )
Level2Key     = PATHITEM( DimEmployee[EmployeePath], 2, INTEGER )
Level3Key     = PATHITEM( DimEmployee[EmployeePath], 3, INTEGER )
Level1Name    = LOOKUPVALUE( DimEmployee[EmployeeName], DimEmployee[EmployeeKey], DimEmployee[Level1Key] )
Level2Name    = LOOKUPVALUE( DimEmployee[EmployeeName], DimEmployee[EmployeeKey], DimEmployee[Level2Key] )

-- Rollup all subordinates of selected employee
Subordinate Sales :=
CALCULATE( [Total Sales],
    FILTER( DimEmployee,
        PATHCONTAINS( DimEmployee[EmployeePath], MAX( DimEmployee[EmployeeKey] ) ) ) )
```

---

## 8. USERELATIONSHIP — Role-Playing Dimensions

Only one active relationship per table pair. Activate inactive relationships inside CALCULATE.

**Model:** `Sales[OrderDateKey] → Calendar[DateKey]` **(ACTIVE)**; ShipDateKey, DueDateKey **(INACTIVE)**.

```dax
Sales by Ship Date :=
CALCULATE( [Total Sales], USERELATIONSHIP( Sales[ShipDateKey], Calendar[DateKey] ) )

-- Combine USERELATIONSHIP with time intelligence in same CALCULATE
Sales Ship YTD :=
CALCULATE( [Total Sales],
    USERELATIONSHIP( Sales[ShipDateKey], Calendar[DateKey] ),
    DATESYTD( Calendar[Date] ) )
```

**Alternative:** Use separate view-sourced tables (`DimOrderDate`, `DimShipDate`, `DimDueDate`) each with an active relationship. Preferred for heavy time intelligence on multiple date roles — no `USERELATIONSHIP` needed.

**Pitfalls:** `USERELATIONSHIP` deactivates the active relationship for the CALCULATE scope. Set cross-filter to **Single** on inactive relationships — bidirectional + USERELATIONSHIP causes unexpected results on-prem.

---

## 9. ALLSELECTED Behaviour in Live Connection

| Function | Clears | Live Connection |
|---|---|---|
| `ALL( Table/Column )` | All filters | ✅ Consistent |
| `ALLSELECTED( Column )` | Inner; preserves slicer/page | ⚠️ Varies by visual type |
| `ALLEXCEPT( Table, Col )` | All except named | ✅ Consistent |

**Key behaviours (PBIRS live connection):**
- Outer context = slicer + page filters — ALLSELECTED respects these correctly.
- Matrix sub-selects can produce a different ALLSELECTED scope than card visuals — always test ranking measures in matrix.
- Avoid `ALLSELECTED( Table )` on tables with many inactive relationships — FE overhead on-prem.

```dax
-- % of total within slicer selection
Sales % of Total :=
DIVIDE( [Total Sales], CALCULATE( [Total Sales], ALLSELECTED( DimProduct[Category] ) ) )

-- Ranking within sliced set
Product Rank (Sliced) :=
RANKX( ALLSELECTED( DimProduct[ProductName] ), [Total Sales],, DESC, DENSE )
```

**When to use:** `ALLSELECTED( Column )` — visual-relative % (slicers restrict denominator). `ALL( Column )` — absolute % of grand total. `ALLEXCEPT` — preserve specific filters, remove all others.

---

## 10. Variables (VAR) — Performance in On-Prem SSAS

`VAR` evaluates once and caches the result for the measure instance. Inline subexpressions may be evaluated multiple times (multiple SE queries).

```dax
-- ❌ SUM(Revenue) evaluated twice — two SE queries
Gross Margin % (Slow) := DIVIDE( SUM( Sales[Revenue] ) - SUM( Sales[Cost] ), SUM( Sales[Revenue] ) )

-- ✅ VAR — single SE query each
Gross Margin % :=
VAR Revenue = SUM( Sales[Revenue] )
VAR Cost    = SUM( Sales[Cost] )
RETURN DIVIDE( Revenue - Cost, Revenue )

-- Context save pattern: VAR captures filter context at declaration point
Sales YoY% :=
VAR Curr = [Total Sales]
VAR Prev = CALCULATE( [Total Sales], SAMEPERIODLASTYEAR( Calendar[Date] ) )
RETURN DIVIDE( Curr - Prev, Prev )

-- Conditional with VAR (avoids double evaluation)
Conditional Growth :=
VAR Sales   = [Total Sales]
VAR PYSales = CALCULATE( [Total Sales], SAMEPERIODLASTYEAR( Calendar[Date] ) )
VAR Growth  = DIVIDE( Sales - PYSales, PYSales )
RETURN IF( ISBLANK( PYSales ), BLANK(), IF( PYSales = 0, BLANK(), Growth ) )
```

**Rules:**
1. Use VAR for any subexpression referenced more than once.
2. VAR captures context at declaration — compute inside CALCULATE if you need a modified context value in the VAR.
3. Keep table VARs (`VAR T = SUMMARIZE(...)`) as filtered as possible — large table VARs consume VertiPaq memory.
4. VARs are scoped to the measure evaluation instance — no cross-query caching.

---

## 11. Error Handling in DAX

```dax
-- ✅ Always DIVIDE() — never / operator
Safe Ratio            := DIVIDE( [Numerator], [Denominator] )       -- returns BLANK on zero
Safe Ratio (0 default) := DIVIDE( [Numerator], [Denominator], 0 )

-- ❌ Avoid — errors in reports
Unsafe Ratio := [Numerator] / [Denominator]

-- IFERROR for lookup fallbacks
Safe Lookup :=
IFERROR( LOOKUPVALUE( DimProduct[StandardCost], DimProduct[ProductKey], MAX( Sales[ProductKey] ) ), 0 )

-- ISBLANK (preferred over = BLANK() for measures)
Has Sales := NOT ISBLANK( [Total Sales] )

-- COALESCE: first non-blank value (CL 1550+ / SSAS 2022 only)
-- For CL 1500: use nested IF( ISBLANK(...), ..., ... ) instead
Effective Price        := COALESCE( [Override Price], [Standard Price], 0 )
Product Category Safe  := COALESCE( SELECTEDVALUE( DimProduct[Category] ), "Unknown" )

-- Return BLANK (not 0) for no-match rows — cleaner in visuals
Total Sales (Clean) := IF( COUNTROWS( Sales ) = 0, BLANK(), SUM( Sales[SalesAmount] ) )
```

---

## 12. Dynamic Security with Active Directory Groups

**Always use `USERNAME()`** (returns `DOMAIN\username`). `USERPRINCIPALNAME()` is unreliable on-prem SSAS — designed for Azure AS.

```dax
-- Option A: User table (DimUserSecurity: UserName, RegionKey)
-- Role filter on DimRegion:
DimRegion[RegionKey] = LOOKUPVALUE(
    DimUserSecurity[RegionKey], DimUserSecurity[UserName], USERNAME() )

-- Option B: AD Group bridge (DimUserGroup: UserName, GroupName; refreshed from AD nightly)
[RegionKey] IN
    CALCULATETABLE(
        VALUES( DimSecurityGroup[RegionKey] ),
        TREATAS(
            CALCULATETABLE( VALUES( DimUserGroup[GroupName] ), DimUserGroup[UserName] = USERNAME() ),
            DimSecurityGroup[GroupName]
        )
    )

-- Universal filter (SecurityPermission: Principal, EntityKey, EntityType)
DimBranch[BranchKey] IN
    CALCULATETABLE( VALUES( SecurityPermission[EntityKey] ),
        SecurityPermission[Principal] = USERNAME(),
        SecurityPermission[EntityType] = "Branch" )
```

**AD group bridge refresh:** SSIS/SQL Agent job querying AD via `System.DirectoryServices.DirectorySearcher` or linked server. Refresh nightly.

**Test RLS:** In DAX Studio, use Roles connection option. In SSMS: `EXECUTE AS USER = 'DOMAIN\testuser'`.

---

## 13. Aggregation Tables (User-Defined, CL 1500+)

> On-prem SSAS has **no automatic aggregations** — configure manually via `TabularEditor.exe`.

**Use when:** fact table >50–100M rows with repeating summary queries at coarser grain (month/region/category).

```sql
-- Step 1: Create agg table in DW (refreshed nightly)
CREATE TABLE [Fact].[Sales_Agg_MonthRegion] (
    DateMonthKey  INT          NOT NULL,
    RegionKey     INT          NOT NULL,
    ProductCatKey INT          NOT NULL,
    TotalSales    DECIMAL(18,2) NOT NULL,
    TotalCost     DECIMAL(18,2) NOT NULL,
    TotalUnits    INT          NOT NULL,
    CONSTRAINT PK_FactSales_Agg PRIMARY KEY (DateMonthKey, RegionKey, ProductCatKey)
);

INSERT INTO [Fact].[Sales_Agg_MonthRegion]
SELECT d.MonthKey, f.RegionKey, p.CategoryKey,
    SUM(f.SalesAmount), SUM(f.CostAmount), SUM(f.Units)
FROM dbo.Sales f
JOIN dbo.Calendar    d ON f.DateKey    = d.DateKey
JOIN [Dimension].[Product] p ON f.ProductKey = p.ProductKey
GROUP BY d.MonthKey, f.RegionKey, p.CategoryKey;
```

```json
// Step 2: In Tabular Editor — set alternateOf on agg table columns (partial BIM)
{
  "name": "TotalSales", "summarizeBy": "sum",
  "alternateOf": { "tableName": "Sales", "columnName": "SalesAmount", "summarization": "Sum" }
}
```

**Rules:** Agg table must be **hidden**. Grain ≥ detail fact grain. Relationships mirror detail fact. Verify via DAX Studio Server Timings → `VertiPaq Scan` hitting agg table name.

---

## 14. DAX for Paginated Report Parameters (PBIRS/SSRS)

Use `@ParamName` syntax in SSRS dataset query text mode. Use integer year/month params to avoid date parsing issues.

```dax
-- Parameterised dataset (params: @StartYear, @StartMonth, @EndYear, @EndMonth)
EVALUATE
CALCULATETABLE(
    SUMMARIZECOLUMNS(
        DimProduct[ProductName], DimProduct[Category],
        Calendar[CalendarYear], Calendar[MonthName],
        "Total Sales", [Total Sales], "Total Cost", [Total Cost],
        "Gross Profit", [Gross Profit], "Units Sold", [Units Sold]
    ),
    Calendar[Date] >= DATE(@StartYear, @StartMonth, 1),
    Calendar[Date] <= DATE(@EndYear, @EndMonth, 31)
)
ORDER BY Calendar[CalendarYear], Calendar[MonthName], DimProduct[ProductName]

-- Measure selector (@MeasureSelector: 1=Sales, 2=Cost, 3=Profit)
EVALUATE CALCULATETABLE(
    ADDCOLUMNS( SUMMARIZE( DimProduct, DimProduct[ProductName], DimProduct[Category] ),
        "Value", SWITCH( @MeasureSelector, 1, [Total Sales], 2, [Total Cost], 3, [Gross Profit], [Total Sales] ) ) )

-- Cascading param: years with data
EVALUATE SUMMARIZECOLUMNS( Calendar[CalendarYear], "HasData", CALCULATE( COUNTROWS( Sales ) ) )
ORDER BY Calendar[CalendarYear] ASC

-- Cascading: regions for selected year
EVALUATE CALCULATETABLE(
    SUMMARIZECOLUMNS( DimRegion[RegionName], DimRegion[RegionKey] ),
    Calendar[CalendarYear] = @SelectedYear )
ORDER BY DimRegion[RegionName]
```

---

## 15. VertiPaq vs. Formula Engine Optimisation

| Component | Role | Parallelism |
|---|---|---|
| **Storage Engine (SE)** | Scans VertiPaq columnar data; simple aggregations | ✅ Multi-threaded |
| **Formula Engine (FE)** | DAX logic, iterators, context transitions | ❌ Single-threaded |

**Golden rule:** Push work to SE. FE bottleneck = no benefit from extra CPU.

**Diagnose in DAX Studio:** Server Timings → SE CPU vs. FE time. SE-dominant (≥80% SE) = good.
High FE + many SE Queries = `Callback DataID` (SE calling FE per row).
Caused by: measures inside FILTER/SUMX, nested iterators, measures referencing measures in row context.

```
Good (SE-dominant): Total 450ms | SE 420ms (93%) | FE 30ms | SE Queries: 4
Bad  (FE-dominant): Total 3200ms | SE 80ms | FE 3120ms (98%) | SE Queries: 847
```

```dax
-- ❌ FILTER with measure — FE per row (CallbackDataID)
CALCULATE( [Total Sales], FILTER( DimProduct, [Product Margin %] > 0.2 ) )

-- ✅ FILTER with column — stays in SE
CALCULATE( [Total Sales], FILTER( DimProduct, DimProduct[StandardMargin] > 0.2 ) )

-- ❌ SUMX + LOOKUPVALUE per row — FE
SUMX( Sales, Sales[Qty] * LOOKUPVALUE( DimProduct[Price], DimProduct[ProductKey], Sales[ProductKey] ) )

-- ✅ Pre-calculated column or relationship
Fast SUM := SUM( Sales[PreCalcRevenue] )
```

**VertiPaq compression tips:**
1. Sort partition query `ORDER BY` highest-cardinality column — better run-length encoding.
2. Avoid full timestamps when date-only sufficient; avoid free-text model columns.
3. Use integer keys, not string keys.

```dax
-- DMV: memory by column (DAX Studio → DMV mode against SSAS)
SELECT MEASURE_GROUP_NAME AS TableName, ATTRIBUTE_NAME AS ColumnName,
    ROWS_COUNT, USED_SIZE AS MemoryBytes, DICTIONARY_SIZE
FROM $SYSTEM.DISCOVER_STORAGE_TABLE_COLUMN_SEGMENTS
ORDER BY USED_SIZE DESC
```

---

## 16. Measure Quality Checklist

- [ ] `DIVIDE()` — not `/` — for all divisions
- [ ] Returns `BLANK()` (not 0) when no data
- [ ] Tested in matrix visual with row/column/page/slicer filters active
- [ ] Tested at grand total (no filters)
- [ ] `VAR` for any subexpression used more than once
- [ ] No `FILTER( AllTable, [Measure] > x )` — column filter or KEEPFILTERS instead
- [ ] Format string set explicitly (not "Auto")
- [ ] Description populated in measure properties
- [ ] Assigned to Display Folder
- [ ] Server Timings checked in DAX Studio — FE time not dominant
- [ ] RLS-sensitive measures tested with impersonation in DAX Studio
- [ ] Time intelligence tested at year / quarter / month / day grain

---

## 17. DAX Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| `SUM(Col) / SUM(Col2)` | Division by zero error | `DIVIDE()` |
| `FILTER( Table, [Measure] > x )` | FE per row (CallbackDataID) | Filter on column, not measure |
| `CALCULATE( X, ALL( Table ) )` in iterator | Removes all filters globally | `REMOVEFILTERS()` with scope or `ALLEXCEPT` |
| `COUNTROWS( FILTER( Table, cond ) )` | Iterates full table in FE | `CALCULATE( COUNTROWS(Table), cond )` |
| `IF( SUM() = 0, BLANK(), ... )` | `SUM()` returns BLANK not 0 when no rows | `IF( ISBLANK( [Measure] ), BLANK(), ... )` |
| `RELATED()` inside measure | Only valid in row context | `LOOKUPVALUE()` or TREATAS |
| Calculated columns for derivable values | Materialised in VertiPaq; wastes memory | Use measures; calc cols only for grouping/slicing |
| Unused columns imported | Bloats memory; slows processing | Remove or exclude from partition query |
| `FORMAT()` inside measure body | Returns text; breaks numeric aggregations | Use Format String property instead |

---

## 18. Measure Organisation Conventions

### Display Folder Structure
```
📁 [Key Metrics]       Total Sales, Total Cost, Gross Profit, Gross Margin %
📁 [Time Intelligence] Sales YTD, MTD, SPLY, YoY %, R12M
📁 [Ratios & Rates]    Avg Transaction Value, Conversion Rate, Return Rate %
📁 [Inventory]         Closing Stock, Opening Stock, Stock Turns
📁 [_Debug]            _Debug Row Count, _Debug Context Test  ← hidden; devs only
```

### Naming Conventions
- Base measures: `Total Sales`, `Units Sold` — plain Title Case
- Time intelligence: `Sales YTD`, `Sales SPLY`, `Sales YoY %` — suffix variant
- Helper/debug: `_Sales Base`, `_Has Filter` — underscore prefix
- No abbreviations in public names; no `FORMAT()` in bodies

### Description (every production measure)
`"Sum of Sales[SalesAmount]. Includes all channels. Excludes cancelled orders (OrderStatus='Cancelled' filtered at partition). Last reviewed: YYYY-MM-DD by [Author]."`

---

## 19. Bus Matrix Validation in DAX

```dax
-- Verify DimProduct conforms across both fact tables
Product in Sales Count :=
CALCULATE( DISTINCTCOUNT( Sales[ProductKey] ), ALLEXCEPT( DimProduct, DimProduct[ProductKey] ) )

Product in Returns Count :=
CALCULATE( DISTINCTCOUNT( FactReturns[ProductKey] ), ALLEXCEPT( DimProduct, DimProduct[ProductKey] ) )

-- Orphan check: fact keys with no dimension match (run in DAX Studio)
EVALUATE
FILTER(
    ADDCOLUMNS( VALUES( Sales[ProductKey] ),
        "InDimProduct",
        CALCULATE( COUNTROWS( DimProduct ), DimProduct[ProductKey] = EARLIER( Sales[ProductKey] ) ) ),
    [InDimProduct] = 0
)
```
