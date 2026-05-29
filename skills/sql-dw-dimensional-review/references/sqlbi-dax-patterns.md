# SQLBI / DAX Patterns Reference

Based on SQLBI publications, daxpatterns.com (Marco Russo & Alberto Ferrari), and Microsoft DAX documentation.

---

## Core DAX Evaluation Rules (Always Apply)

1. **Row context vs. filter context** — never confuse the two; `CALCULATE` always transforms row context into filter context
2. **Context transition** — when a measure is called inside a row context (e.g., inside `ADDCOLUMNS`), `CALCULATE` implicitly applies context transition; use `EARLIER` sparingly
3. **`CALCULATE` always removes then re-adds filters** — `CALCULATE(expr, filter)` removes existing filters on the filtered column, then applies the new filter
4. **`ALL` vs `ALLSELECTED` vs `ALLEXCEPT`** — `ALL` ignores all filters including slicers; `ALLSELECTED` respects slicer context but removes other filters; `ALLEXCEPT` keeps named column filters only
5. **Avoid `FILTER(ALL(...))` on large tables** — always prefer direct column filters over table iterators when possible

---

## Time Intelligence Patterns

### Prerequisites
- A **marked date table** in the model (`Mark as Date Table` in SSAS Tabular / Power BI)
- Date table must be contiguous with no gaps, one row per day
- Date column must be `Date` data type

### Year-to-Date
```dax
Sales YTD =
CALCULATE(
    [Total Sales],
    DATESYTD('Date'[Date])
)

-- Fiscal YTD (fiscal year ends June 30)
Sales FYTD =
CALCULATE(
    [Total Sales],
    DATESYTD('Date'[Date], "06-30")
)
```

### Prior Year Comparison
```dax
Sales PY =
CALCULATE(
    [Total Sales],
    SAMEPERIODLASTYEAR('Date'[Date])
)

Sales YoY % =
VAR CurrentSales = [Total Sales]
VAR PriorSales = [Sales PY]
RETURN
    IF(
        NOT ISBLANK(PriorSales) && PriorSales <> 0,
        DIVIDE(CurrentSales - PriorSales, PriorSales),
        BLANK()
    )
```

### Rolling N Periods
```dax
Sales Rolling 12M =
CALCULATE(
    [Total Sales],
    DATESINPERIOD(
        'Date'[Date],
        LASTDATE('Date'[Date]),
        -12,
        MONTH
    )
)
```

### Month-to-Date / Quarter-to-Date
```dax
Sales MTD = CALCULATE([Total Sales], DATESMTD('Date'[Date]))
Sales QTD = CALCULATE([Total Sales], DATESQTD('Date'[Date]))
```

### Moving Annual Total (MAT) / Last 12 Months
```dax
Sales MAT =
CALCULATE(
    [Total Sales],
    DATESBETWEEN(
        'Date'[Date],
        NEXTDAY(SAMEPERIODLASTYEAR(LASTDATE('Date'[Date]))),
        LASTDATE('Date'[Date])
    )
)
```

---

## Semi-Additive Measures

Semi-additive measures aggregate normally over some dimensions (e.g., time) but use non-additive logic (last value, average, min/max) over others (typically account/entity).

### Last Non-Empty (Balance at End of Period)
```dax
Balance EOD =
CALCULATE(
    LASTNONBLANK(
        'Date'[Date],
        CALCULATE(SUM(Fact_Balance[Amount]))
    ),
    'Date'[Date] <= MAX('Date'[Date])
)

-- Preferred pattern for periodic snapshot facts:
Balance EOD =
CALCULATE(
    SUM(Fact_Balance[Amount]),
    LASTDATE('Date'[Date])
)
```

### Opening Balance (First Non-Empty)
```dax
Balance BOD =
CALCULATE(
    SUM(Fact_Balance[Amount]),
    FIRSTDATE('Date'[Date])
)
```

---

## Many-to-Many Patterns

### Bridge Table Pattern (Kimball-compatible)
```dax
-- Measure on fact table navigating through bridge to dimension
Sales by Tag =
CALCULATE(
    [Total Sales],
    TREATAS(
        VALUES(Bridge_ProductTag[ProductKey]),
        Fact_Sales[ProductKey]
    )
)
```

### Weak Relationship Many-to-Many (via shared dimension)
```dax
-- Classic M2M: Customer → Bridge_CustomerGroup → Group
Customers in Group =
CALCULATE(
    COUNTROWS(Dim_Customer),
    TREATAS(
        SUMMARIZE(
            Bridge_CustomerGroup,
            Bridge_CustomerGroup[CustomerKey]
        ),
        Dim_Customer[CustomerKey]
    )
)
```

---

## Calculation Groups

Calculation groups require SSAS Tabular 1500+ or Power BI Premium / Fabric. They replace many measures with a single template.

### Time Intelligence Calculation Group
```
Calculation Group: Time Intelligence
  Calculation Item: Current            → SELECTEDMEASURE()
  Calculation Item: YTD                → CALCULATE(SELECTEDMEASURE(), DATESYTD('Date'[Date]))
  Calculation Item: MTD                → CALCULATE(SELECTEDMEASURE(), DATESMTD('Date'[Date]))
  Calculation Item: PY                 → CALCULATE(SELECTEDMEASURE(), SAMEPERIODLASTYEAR('Date'[Date]))
  Calculation Item: YoY %              → DIVIDE(SELECTEDMEASURE() - CALCULATE(SELECTEDMEASURE(), SAMEPERIODLASTYEAR('Date'[Date])), CALCULATE(SELECTEDMEASURE(), SAMEPERIODLASTYEAR('Date'[Date])))
  Calculation Item: Rolling 12M        → CALCULATE(SELECTEDMEASURE(), DATESINPERIOD('Date'[Date], LASTDATE('Date'[Date]), -12, MONTH))
```

### Currency Conversion Calculation Group
```
Calculation Group: Currency
  Calculation Item: [Base Currency]    → SELECTEDMEASURE()
  Calculation Item: [Reporting Currency] → SUMX(VALUES('Date'[Date]), SELECTEDMEASURE() * RELATED(Dim_ExchangeRate[Rate]))
```

**Key rules for calculation groups**:
- Set `Precedence` to control evaluation order when multiple calculation groups apply
- Use `ISSELECTEDMEASURE()` to apply calculation items only to specific measures
- Avoid referencing specific measure names inside calculation items where possible

---

## Disconnected Tables / Parameter Tables

Used to create user-controlled slicer inputs (e.g., "Top N", budget scenario, what-if).

```dax
-- Parameter table: { 3, 5, 10, 20, 50 } for "Top N" selection
Top N Selected = SELECTEDVALUE('Parameter_TopN'[Value], 10)

Top N Customers =
CALCULATE(
    [Total Sales],
    TOPN(
        [Top N Selected],
        ALL(Dim_Customer),
        [Total Sales]
    )
)
```

---

## Ranking Patterns

### Static Rank (dense, across all)
```dax
Customer Rank =
RANKX(
    ALL(Dim_Customer[CustomerName]),
    [Total Sales],
    ,
    DESC,
    DENSE
)
```

### Rank Within Group (e.g., rank by region)
```dax
Customer Rank in Region =
RANKX(
    ALLEXCEPT(Dim_Customer, Dim_Geography[RegionName]),
    [Total Sales],
    ,
    DESC,
    DENSE
)
```

---

## Dynamic Segmentation (ABC Analysis / Banding)

```dax
Customer Segment =
VAR Sales = [Total Sales]
RETURN
    SWITCH(
        TRUE(),
        Sales >= 100000, "A - High",
        Sales >= 10000,  "B - Medium",
        Sales > 0,       "C - Low",
        "D - Inactive"
    )
```

---

## Parent-Child Hierarchies (using PATH functions)

For variable-depth hierarchies (org charts, account hierarchies) in SSAS Tabular:

```dax
-- Calculated column on Dim_Employee: build path
EmployeePath = PATH(Dim_Employee[EmployeeKey], Dim_Employee[ManagerKey])

-- Flatten to fixed depth (create calculated columns per level)
Level1 = LOOKUPVALUE(Dim_Employee[EmployeeName], Dim_Employee[EmployeeKey], PATHITEM([EmployeePath], 1, INTEGER))
Level2 = LOOKUPVALUE(Dim_Employee[EmployeeName], Dim_Employee[EmployeeKey], PATHITEM([EmployeePath], 2, INTEGER))
Level3 = LOOKUPVALUE(Dim_Employee[EmployeeName], Dim_Employee[EmployeeKey], PATHITEM([EmployeePath], 3, INTEGER))

-- Depth
HierarchyDepth = PATHLENGTH([EmployeePath])
```

---

## Measure Quality Checklist (SQLBI Standards)

- [ ] Every measure uses `DIVIDE()` instead of `/` to handle division by zero gracefully
- [ ] `BLANK()` is returned (not 0) when no data exists — avoids misleading aggregations
- [ ] `VAR ... RETURN` pattern used for readability in any measure longer than 2 lines
- [ ] No hardcoded date literals — use `TODAY()`, `MAX('Date'[Date])`, or parameters
- [ ] `CALCULATE` filters use column filters (`[Column] = value`) not full table filters where possible
- [ ] Measures that call other measures do so by reference — never duplicate logic
- [ ] All time intelligence measures have a prerequisite check that the date table is marked
- [ ] Semi-additive measures clearly documented with which dimensions are additive vs. not
- [ ] Calculation group precedence documented in model description
- [ ] No circular dependencies between measures or calculated columns

---

## DAX Anti-Patterns

| Anti-Pattern | Problem | Correct Approach |
|---|---|---|
| `FILTER(ALL(Table), condition)` on large tables | Full table scan on every eval | Use direct column filter: `CALCULATE(expr, Table[Col] = value)` |
| Division without `DIVIDE()` | Errors on zero denominator | `DIVIDE(numerator, denominator, 0)` |
| `COUNTROWS(FILTER(...))` | Inefficient iterator | `CALCULATE(COUNTROWS(Table), filter)` |
| Calculated columns for time intel | Computed at refresh, not at query time | Use measures with time intelligence functions |
| `RELATED()` in measures | Works but hides filter context intent | Use `CALCULATE` with explicit relationship traversal |
| Implicit `SUMX` over many rows | Performance issue | Use explicit aggregation + relationship |
| Overusing `ALLSELECTED` | Unpredictable with visual calculations | Only use when slicer preservation is explicitly needed |
| Nested `CALCULATE` with conflicting filters | Difficult to reason about precedence | Separate into VARs with clear intermediate steps |

---

## SSAS Tabular Measure Organization Conventions

- Group measures into **display folders** by type: `[Sales]`, `[Inventory]`, `[Time Intelligence]`, `[Ratios]`
- Hide all intermediate (helper) measures using `IsHidden = True`
- Prefix measure names consistently: `_helper` prefix for hidden measures
- All visible measures should have a `Description` set in the model
- Format strings must be explicit — avoid "Auto" format for production models
