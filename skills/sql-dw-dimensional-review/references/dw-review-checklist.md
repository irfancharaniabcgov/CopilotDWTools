# DW & Tabular Model Review Checklist

A structured review checklist for SQL Server Data Warehouses and Analysis Services Tabular models,
based on Kimball methodology and SQLBI/SSAS best practices. Use this checklist to systematically
assess an existing DW or model and produce a prioritized finding report.

---

## Section 1: SQL Server Data Warehouse Schema Review

### 1.1 Fact Table Audit

For each fact table, verify:

- [ ] **Grain documented**: Is there a `Grain` extended property or equivalent documentation?
- [ ] **Grain consistency**: Every column in the fact table is either a FK, measure, or degenerate dimension — no non-grain attributes
- [ ] **Date dimension FK**: Every date/time column in a fact table references `Dim_Date` via a DateKey (INT YYYYMMDD) — no raw `DATETIME` FK columns
- [ ] **Surrogate keys used as FKs**: All dimension foreign keys are surrogate integers, not natural/business keys
- [ ] **No NULL FKs without "unknown" row**: If NULLs are allowed on a FK, there must be a corresponding "Unknown" / "Not Applicable" row in the dimension (key = -1 or 0)
- [ ] **Measures are additive or documented as semi-additive**: Columns holding non-additive values (balances, counts at point-in-time) are flagged
- [ ] **No measures stored as VARCHAR**: All measure columns must be numeric types
- [ ] **Degenerate dimensions are strings, not FKs**: Invoice numbers, order numbers etc. stored as NVARCHAR, not FK to a dimension

**Queries to run**:
```sql
-- Find fact table FK columns that reference datetime directly (grain smell)
SELECT OBJECT_SCHEMA_NAME(fk.parent_object_id) + '.' + OBJECT_NAME(fk.parent_object_id) AS FactTable,
       COL_NAME(fkc.parent_object_id, fkc.parent_column_id) AS Column,
       TYPE_NAME(c.user_type_id) AS DataType
FROM sys.foreign_keys fk
JOIN sys.foreign_key_columns fkc ON fk.[object_id] = fkc.constraint_object_id
JOIN sys.columns c ON fkc.parent_object_id = c.object_id AND fkc.parent_column_id = c.column_id
WHERE TYPE_NAME(c.user_type_id) IN ('datetime', 'datetime2', 'date', 'smalldatetime');
```

### 1.2 Dimension Table Audit

- [ ] **Surrogate key present**: Every dimension has an INT surrogate PK
- [ ] **Natural/business key retained**: Source system key is stored as an attribute column
- [ ] **SCD Type documented**: `SCDType` extended property set on table
- [ ] **SCD Type 2 infrastructure**: If SCD Type 2, `RowEffectiveDate`, `RowExpirationDate`, and `IsCurrent` columns present
- [ ] **No snowflaking**: Dimension attributes are flat — no separate normalized sub-tables that should be denormalized
- [ ] **"Unknown" member row**: Row with surrogate key = -1 (or 0) exists to capture unmatched fact FK rows
- [ ] **Date dimension populated**: `Dim_Date` covers the full range of dates present in fact tables + future periods
- [ ] **Fiscal calendar correct**: Fiscal year/period columns match organization's fiscal calendar definition
- [ ] **Conformed dimensions**: Tables shared across fact tables are identified and their grain confirmed to be identical

**Queries to run**:
```sql
-- Dimensions potentially missing surrogate key (no INT identity PK)
SELECT OBJECT_SCHEMA_NAME(t.object_id) + '.' + t.[name] AS TableName
FROM sys.tables t
WHERE t.is_ms_shipped = 0
  AND NOT EXISTS (
    SELECT 1 FROM sys.columns c
    JOIN sys.index_columns ic ON c.object_id = ic.object_id AND c.column_id = ic.column_id
    JOIN sys.indexes i ON ic.object_id = i.object_id AND ic.index_id = i.index_id
    WHERE c.object_id = t.object_id
      AND i.is_primary_key = 1
      AND c.is_identity = 1
      AND TYPE_NAME(c.user_type_id) = 'int'
  );
```

### 1.3 General Schema Health

- [ ] **Naming conventions consistent**: Fact/Dim/Bridge/Staging prefix conventions followed
- [ ] **No orphan tables**: Every table has at least one relationship to either a fact or dimension
- [ ] **Extended properties coverage**: `MS_Description` set on all tables, views, key columns
- [ ] **No deprecated objects**: Archive/retired objects in `Archive` schema, not `dbo`
- [ ] **No tables with no PK**: Every table has a primary key defined
- [ ] **FK constraints present**: Fact-to-dimension FK constraints defined (even if not enforced)

```sql
-- Tables without a primary key
SELECT OBJECT_SCHEMA_NAME(t.object_id) + '.' + t.[name] AS TableName
FROM sys.tables t
WHERE t.is_ms_shipped = 0
  AND NOT EXISTS (
    SELECT 1 FROM sys.indexes i
    WHERE i.object_id = t.object_id AND i.is_primary_key = 1
  );
```

---

## Section 2: Stored Procedure / View Audit

- [ ] **Naming conventions**: SPs prefixed with `usp_` (or similar convention), views with `vw_`
- [ ] **Documentation present**: `MS_Description` + `ExecutionContext` extended properties set
- [ ] **No `SELECT *`**: Views and SPs explicitly name all columns
- [ ] **No implicit schema**: All object references are schema-qualified (`dbo.TableName`, not `TableName`)
- [ ] **Error handling present**: ETL SPs have `TRY/CATCH` with logging to an audit/error table
- [ ] **Idempotent ETL SPs**: Load procedures are safe to re-run (no duplicate row creation)
- [ ] **High-water mark or incremental logic documented**: How does each load SP determine what's new?
- [ ] **Report views isolated**: Reporting views are in a separate schema (`report`, `rpt`, or similar) and do not contain ETL logic

---

## Section 3: Analysis Services Tabular Model Review

### 3.1 Model Structure

- [ ] **Date table marked**: A table is designated as the date table (`Mark as Date Table`)
- [ ] **Date table is contiguous**: No gaps in dates; covers the full fact table date range
- [ ] **`DiscourageImplicitMeasures`**: Set to `True` if calculation groups are used
- [ ] **No unused tables/columns**: All imported tables and columns are actually used in relationships, measures, or reports
- [ ] **Hidden appropriately**: Surrogate key columns, technical helper columns, and intermediate calculation tables are hidden
- [ ] **Perspectives defined**: At least one perspective per audience/subject area

### 3.2 Relationship Audit

- [ ] **All fact-to-dim relationships active**: No required relationships left inactive without corresponding DAX `USERELATIONSHIP`
- [ ] **Minimal bidirectional**: No bidirectional cross-filters on fact tables; explain every bidirectional relationship in documentation
- [ ] **Referential integrity flag set**: `RelyOnReferentialIntegrity = True` where DW guarantees no orphan FKs
- [ ] **Role-playing dimensions**: Multiple relationships to the same date dimension implemented as inactive with `USERELATIONSHIP()` in measures

### 3.3 Measure Quality

- [ ] **All measures have descriptions**: Check `$SYSTEM.TMSCHEMA_MEASURES` for blank `Description`
- [ ] **All measures have format strings**: No `Auto` format in production measures
- [ ] **All measures in display folders**: No measures at the root level
- [ ] **`DIVIDE()` used for all division**: No `/` operator in measures
- [ ] **`BLANK()` returned when no data**: No artificial `0` returns masking absence of data
- [ ] **VAR/RETURN pattern**: Multi-step measures use `VAR` not nested `CALCULATE` chains
- [ ] **No hardcoded dates or IDs**: Measures use `TODAY()`, `SELECTEDVALUE`, or parameters
- [ ] **Time intelligence measured correctly**: Time intelligence measures validated against known values for a test period

```sql
-- DMV: Find measures without descriptions
SELECT t.[Name] AS TableName, m.[Name] AS MeasureName, m.[Description]
FROM $SYSTEM.TMSCHEMA_MEASURES m
JOIN $SYSTEM.TMSCHEMA_TABLES t ON m.[TableID] = t.[ID]
WHERE ISNULL(m.[Description], '') = ''
ORDER BY t.[Name], m.[Name];
```

### 3.4 Column Encoding & Performance

- [ ] **Integer keys have `EncodingHint = Value`**: Surrogate key columns force value encoding for better compression
- [ ] **High-cardinality string columns reviewed**: Columns with >1M unique values flagged for possible removal or encoding change
- [ ] **No GUID columns**: GUIDs in Tabular compress poorly; convert to INT surrogate keys at DW layer
- [ ] **Partitioning on large fact tables**: Tables with >10M rows should have at least year-based partitions

```sql
-- DMV: Columns with high dictionary size (cardinality pressure)
SELECT TOP 20
    [TABLE_ID] AS TableName,
    [COLUMN_ID] AS ColumnName,
    [DICTIONARY_SIZE],
    [COLUMN_ENCODING]
FROM $SYSTEM.DISCOVER_STORAGE_TABLE_COLUMNS
ORDER BY [DICTIONARY_SIZE] DESC;
```

### 3.5 Security

- [ ] **RLS roles defined**: At least one role with appropriate RLS DAX filter
- [ ] **No static username in RLS**: Use `USERPRINCIPALNAME()` not hardcoded email addresses
- [ ] **Sensitive columns restricted**: PII/confidential columns hidden or restricted via column security
- [ ] **Roles tested**: Each role tested with test users or `CUSTOMDATA()`

---

## Section 4: DAX Measure Pattern Review

For each measure, evaluate against `sqlbi-dax-patterns.md`:

- [ ] Time intelligence measures use a marked date table, not a regular date column
- [ ] Semi-additive measures (e.g., balance totals) use `LASTNONBLANK` or `LASTDATE` pattern, not `SUM`
- [ ] Many-to-many measures navigate through bridge tables correctly
- [ ] Parent-child hierarchies use `PATH()` functions, not recursive calculated columns
- [ ] Calculation group precedence is documented and intentional

---

## Section 5: Bus Matrix Validation

Create or verify the enterprise bus matrix:

1. List all fact tables
2. List all dimension tables
3. For each fact table, verify:
   - Which dimensions attach (FK relationships present)
   - Grain is declared and documented
   - All expected dimensions are present (e.g., every fact should have at least a Date dimension)
4. For each dimension, verify:
   - Which fact tables reference it
   - Grain is the same across all fact tables (conformed)
   - If NOT conformed — rename to indicate the scope (e.g., `Dim_Customer_Sales` vs `Dim_Customer_HR`)

---

## Severity Levels

| Severity | Description |
|---|---|
| 🔴 Critical | Data correctness or security at risk; must fix before production |
| 🟠 High | Significant performance, maintainability, or analytical accuracy risk |
| 🟡 Medium | Design anti-pattern that will cause future problems or confusion |
| 🔵 Low | Documentation gap, naming inconsistency, or improvement opportunity |

---

## Review Output Template

```markdown
## DW / Tabular Model Review: <Database/Model Name>
**Date**: <date>
**Reviewer**: GitHub Copilot (ssas-tabular-dw-architect agent)

### Summary
- Total findings: X (Y Critical, Z High, W Medium, V Low)
- Documentation coverage: X% tables with MS_Description, X% columns documented
- SCD coverage: X tables with SCD Type defined

### Critical Findings
| # | Object | Finding | Recommendation |
|---|---|---|---|

### High Findings
| # | Object | Finding | Recommendation |
|---|---|---|---|

### Bus Matrix
| Fact Table | Grain | Dim_Date | Dim_Customer | ... |
|---|---|---|---|---|

### Extended Properties Coverage
(Output from bulk audit queries)

### Next Steps
1. ...
```
