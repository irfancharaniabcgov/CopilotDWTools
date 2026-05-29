# Analysis Services Tabular Best Practices Reference

Based on Microsoft documentation, SQLBI guidance, and community best practices for SSAS Tabular (compatibility level 1200–1600+), Azure Analysis Services, and Power BI Premium Semantic Models.

---

## Model File Formats

### BIM (JSON) — Legacy Format
- Single `model.bim` file containing the entire model as JSON (Tabular Model Scripting Language — TMSL)
- Used by SSDT (SQL Server Data Tools) and older tooling
- Suitable for version control as a single file

### TMDL — Tabular Model Definition Language (Modern)
- Folder-based format introduced in compatibility level 1600+
- Each object (table, measure, column) is a separate file — enables git diff at the object level
- **Strongly preferred for version-controlled development**
- Structure:
  ```
  model/
    .tmdl                    ← model-level settings
    tables/
      Fact_Sales.tmdl
      Dim_Customer.tmdl
      Dim_Date.tmdl
    relationships.tmdl
    roles.tmdl
    cultures/
      en-US.tmdl
  ```

### Reading Model Content with Copilot
- For `.bim` files: read JSON directly; tables are under `model.tables[]`
- For TMDL: each `.tmdl` file is readable text; table definition contains columns, measures, hierarchies, partitions
- For live models: use SSAS DMV queries (see DMV Reference section below)

---

## Naming Conventions

### Tables
| Type | Convention | Example |
|---|---|---|
| Fact table | `Fact_<Subject>` | `Fact_SalesTransaction` |
| Dimension table | `Dim_<Entity>` | `Dim_Customer` |
| Bridge table | `Bridge_<M2M>` | `Bridge_ProductCategory` |
| Date table | `Dim_Date` | `Dim_Date` |
| Calculation group table | `CalcGroup_<Name>` | `CalcGroup_TimeIntelligence` |
| Parameter / disconnected | `Param_<Name>` | `Param_TopN` |
| Helper / staging (hidden) | Prefix with underscore | `_Staging_Sales` |

### Columns
- PascalCase: `CustomerName`, `OrderDateKey`, `IsCurrent`
- Avoid spaces in column names (complicates DAX syntax)
- FK columns: match the dimension key name exactly across fact and dimension: `DateKey`, `CustomerKey`
- Natural/business keys: `CustomerID`, `PolicyNumber` (retained as attribute, not used as FK)
- Boolean flags: `Is` prefix — `IsActive`, `IsOnline`, `IsHoliday`
- Date keys: `<Role>DateKey` — `OrderDateKey`, `ShipDateKey`, `EffectiveDateKey`

### Measures
- PascalCase with spaces: `Total Sales`, `Customer Count`, `Average Order Value`
- Hidden helpers: `_Total Sales (raw)` or prefix `_` 
- Format with `[` `]` in DAX expressions: `[Total Sales]`
- Group in display folders: `Sales\Volume`, `Sales\Value`, `Time Intelligence\YTD`

### Hierarchies
- Use business-friendly names: `Calendar`, `Fiscal Calendar`, `Geography`, `Product Category`
- Level names without the parent: `Year`, `Quarter`, `Month`, `Day` (not `CalendarYear`, `CalendarQuarter`)

---

## Relationship Design

### Cardinality and Direction
- **One-to-Many (1:*)** — Standard, always preferred. Dimension (1) → Fact (*)
- **Many-to-Many (*:*)** — Supported natively in 1500+; use only when required and document why
- **Single direction** — Default; use bidirectional only when explicitly required and aware of performance impact
- **Active vs. Inactive** — One relationship per column pair can be active; use `USERELATIONSHIP()` in DAX for inactive ones (role-playing dimensions)

### Referential Integrity
- Always set `Assume Referential Integrity = True` where the DW guarantees no orphan FKs
- This enables DirectQuery join optimization and reduces query plan complexity

### Cross-Source Relationships (Composite Models)
- Document all cross-source relationships explicitly in the model description
- Composite model relationships between Import and DirectQuery sources have significant performance implications

---

## Partition Strategy

### Why Partition
- Enables incremental refresh (process only new/changed data)
- Parallelizes processing across partitions
- Allows hot/cold data split for large fact tables

### Standard Pattern
```
Fact_SalesTransaction:
  Partition: Archive_2018    ← full year, static, processed once
  Partition: Archive_2019    ← full year, static, processed once
  Partition: Archive_2020    ← full year, static
  Partition: Rolling_Current ← last 2 years, refreshed daily
  Partition: Current_Month   ← current month, refreshed hourly
```

### Incremental Refresh Setup (Power BI / Azure AS)
- Define `RangeStart` and `RangeEnd` parameters in Power Query
- Filter source query using these parameters
- Set refresh policy: detect data changes column (optional), archive periods, incremental periods

---

## Column Storage & Encoding

SSAS Tabular uses VertiPaq in-memory columnar storage. Encoding type is auto-selected but can be hinted:

| Encoding | Best For | How to Force |
|---|---|---|
| **Hash** (default for text/low cardinality) | String columns, status codes, flags | Default for text |
| **Value** (numeric sequential) | Surrogate keys, sequential IDs | Set `EncodingHint = Value` on key columns |

**Memory reduction tips**:
- Remove unused columns — every column costs memory even if not in reports
- Split high-cardinality string columns (long descriptions) — they compress poorly
- Date/time: store as `Date` type, not `DateTime` if time-of-day not needed
- Integer surrogate keys compress far better than GUID or string keys

---

## Calculation Groups (SSAS 1500+ / Power BI Premium)

### Setup Requirements
- Model compatibility level ≥ 1500
- `DiscourageImplicitMeasures = True` must be set when using calculation groups (prevents ambiguity)

### Precedence Rules
- Lower precedence number = evaluated first (inner)
- Higher precedence number = evaluated last (outer / wraps inner)
- **Example**: Currency conversion (precedence 10) applied after time intelligence (precedence 5)

### Common Calculation Groups
1. **Time Intelligence** (precedence 5): Current, YTD, QTD, MTD, PY, YoY%, Rolling12M
2. **Currency** (precedence 10): Base currency, reporting currency
3. **Scenario** (precedence 15): Actual, Budget, Forecast, Variance

---

## Perspectives

- Use perspectives to simplify the user experience for role-specific audiences
- Perspectives are **views** — they do not provide security (use roles for that)
- Common perspectives: `Sales Reporting`, `Finance Reporting`, `Operations`
- Hide all technical/helper tables and columns from all perspectives

---

## Row-Level Security (RLS)

```dax
-- Dynamic RLS: filter data to the logged-in user's assigned region
[RegionKey] IN
    CALCULATETABLE(
        VALUES(Dim_UserRegionAccess[RegionKey]),
        Dim_UserRegionAccess[UserEmail] = USERPRINCIPALNAME()
    )
```

- Test with `CUSTOMDATA()` for application-level security token
- Document each role's filter expression and the dimension table it applies to
- Use `LOOKUPVALUE` for complex username-to-attribute mapping

---

## DMV Reference (Query Live SSAS Tabular Models)

Connect via DAX Studio, SSMS (Analysis Services endpoint), or mssql extension pointing at XMLA endpoint.

### List All Tables
```sql
SELECT [Name], [Description], [IsHidden], [StorageMode]
FROM $SYSTEM.TMSCHEMA_TABLES
WHERE [IsPrivate] = FALSE
ORDER BY [Name]
```

### List All Columns
```sql
SELECT t.[Name] AS TableName, c.[Name] AS ColumnName, c.[DataType], c.[Description], c.[IsHidden], c.[EncodingHint]
FROM $SYSTEM.TMSCHEMA_COLUMNS c
JOIN $SYSTEM.TMSCHEMA_TABLES t ON c.[TableID] = t.[ID]
WHERE t.[IsPrivate] = FALSE AND c.[Type] <> 3  -- exclude row number columns
ORDER BY t.[Name], c.[Name]
```

### List All Measures
```sql
SELECT t.[Name] AS TableName, m.[Name] AS MeasureName, m.[Expression], m.[Description], m.[IsHidden], m.[DisplayFolder], m.[FormatString]
FROM $SYSTEM.TMSCHEMA_MEASURES m
JOIN $SYSTEM.TMSCHEMA_TABLES t ON m.[TableID] = t.[ID]
ORDER BY t.[Name], m.[DisplayFolder], m.[Name]
```

### List All Relationships
```sql
SELECT
    ft.[Name] AS FromTable,
    fc.[Name] AS FromColumn,
    tt.[Name] AS ToTable,
    tc.[Name] AS ToColumn,
    r.[CrossFilteringBehavior],
    r.[IsActive],
    r.[RelyOnReferentialIntegrity]
FROM $SYSTEM.TMSCHEMA_RELATIONSHIPS r
JOIN $SYSTEM.TMSCHEMA_TABLES ft ON r.[FromTableID] = ft.[ID]
JOIN $SYSTEM.TMSCHEMA_COLUMNS fc ON r.[FromColumnID] = fc.[ID]
JOIN $SYSTEM.TMSCHEMA_TABLES tt ON r.[ToTableID] = tt.[ID]
JOIN $SYSTEM.TMSCHEMA_COLUMNS tc ON r.[ToColumnID] = tc.[ID]
ORDER BY ft.[Name]
```

### List Partitions
```sql
SELECT t.[Name] AS TableName, p.[Name] AS PartitionName, p.[Mode], p.[Source]
FROM $SYSTEM.TMSCHEMA_PARTITIONS p
JOIN $SYSTEM.TMSCHEMA_TABLES t ON p.[TableID] = t.[ID]
ORDER BY t.[Name], p.[Name]
```

### Table Size (Row Counts)
```sql
SELECT
    [DIMENSION_NAME] AS TableName,
    [DIMENSION_CARDINALITY] AS RowCount
FROM $SYSTEM.MDSCHEMA_DIMENSIONS
WHERE [CUBE_NAME] = '$SYSTEM' AND DIMENSION_IS_VISIBLE
ORDER BY [DIMENSION_CARDINALITY] DESC
```

### Column Cardinality and Encoding
```sql
SELECT
    [TABLE_ID] AS TableName,
    [COLUMN_ID] AS ColumnName,
    [DICTIONARY_SIZE],
    [COLUMN_ENCODING]
FROM $SYSTEM.DISCOVER_STORAGE_TABLE_COLUMNS
ORDER BY [DICTIONARY_SIZE] DESC
```

---

## TMDL Snippet Examples

### Table Definition (TMDL)
```tmdl
table Fact_Sales
    lineageTag: <guid>
    description: 'Sales transactions at daily product-customer-store grain'

    partition Fact_Sales
        mode: import
        source
            type: m
            expression:
                let
                    Source = Sql.Database("dwserver", "WAO_DW"),
                    Data = Source{[Schema="dbo",Item="Fact_Sales"]}[Data]
                in
                    Data

    column SalesKey
        dataType: int64
        isHidden: true
        description: 'Surrogate key'
        summarizeBy: none

    measure 'Total Sales' =
        SUMX(Fact_Sales, Fact_Sales[Quantity] * Fact_Sales[UnitPrice])
        formatString: "#,##0.00"
        description: 'Sum of all sales revenue at the row grain'
        displayFolder: Sales\Value
```

### Relationship (TMDL)
```tmdl
relationship
    fromTable: Fact_Sales
    fromColumn: DateKey
    toTable: Dim_Date
    toColumn: DateKey
    isActive: true
    relyOnReferentialIntegrity: true
```

---

## Common Review Findings in Tabular Models

| Finding | Severity | Description |
|---|---|---|
| Missing `description` on tables/columns/measures | Medium | Causes poor self-service experience; users cannot understand data |
| No date table marked | High | All time intelligence DAX will fail silently or return wrong results |
| Bidirectional relationships everywhere | High | Performance degradation, ambiguous filter paths, incorrect totals |
| Calculated columns for time intelligence | Medium | Should be DAX measures; calculated columns inflate model size |
| No partitioning on large fact tables | High | Full table refresh on every process; refresh time and memory issues |
| Measures without format strings | Low | Inconsistent display in reports |
| No RLS on sensitive dimensions | Critical | Data governance / compliance risk |
| GUID or string surrogate keys | Medium | Poor VertiPaq compression; prefer INT keys |
| Unused tables not hidden | Low | Confuses self-service users |
| No `DiscourageImplicitMeasures` with calc groups | High | Implicit measures interact unpredictably with calculation groups |
