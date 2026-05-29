# Kimball Dimensional Modeling Reference

Based on Ralph Kimball's *The Data Warehouse Toolkit* (3rd ed.) and the Kimball Group methodology.

---

## Core Vocabulary

| Term | Definition |
|---|---|
| **Grain** | The precise definition of what one row in a fact table represents. Must be declared before choosing dimensions or facts. |
| **Surrogate Key** | System-generated integer key (not business key) used as PK on all dimension tables. Enables SCD history and isolates DW from source system changes. |
| **Natural Key / Business Key** | The identifier from the source system (e.g., EmployeeID, PolicyNumber). Should be retained as an attribute but not used as FK. |
| **Conformed Dimension** | A dimension shared identically (or as a subset) across multiple fact tables, enabling drill-across queries. |
| **Conformed Fact** | A metric with a consistent definition across multiple fact tables. |
| **Bus Matrix** | An enterprise-level matrix showing which dimensions attach to which fact tables. |
| **Slowly Changing Dimension (SCD)** | A technique for tracking historical changes to dimension attributes. |

---

## Fact Table Types

### 1. Transaction Fact Table
- **Grain**: One row per transaction event (invoice line, claim submission, sensor reading)
- **Characteristics**: Sparse, additive measures, largest tables in DW
- **Example**: `Fact_ClaimTransaction`, `Fact_SalesOrder`
- **Key design rule**: All foreign keys must resolve to the declared grain

### 2. Periodic Snapshot Fact Table
- **Grain**: One row per period per entity (account balance on last day of month)
- **Characteristics**: Dense (all entities appear even with no activity), semi-additive measures
- **Example**: `Fact_AccountBalance`, `Fact_InventorySnapshot`
- **Key design rule**: Date dimension FK points to the snapshot date, not a transaction date

### 3. Accumulating Snapshot Fact Table
- **Grain**: One row per entity lifecycle (one row per insurance claim through all stages)
- **Characteristics**: Rows are updated; multiple date FKs for pipeline stages; lag measures
- **Example**: `Fact_ClaimLifecycle`, `Fact_OrderFulfillment`
- **Key design rule**: Include a `LastUpdatedDate` and date FKs for each pipeline milestone

### 4. Factless Fact Table
- **Grain**: An event or coverage that has no numeric measures
- **Two types**: Event coverage (what happened), Coverage (what could happen)
- **Example**: `Fact_StudentAttendance` (student was present — no measure), `Fact_PolicyCoverage`

---

## Dimension Table Types

### Standard Dimension
- Descriptive context for facts (Who, What, Where, When, Why, How)
- One row per entity, surrogate key PK, natural key retained as attribute
- Wide tables preferred (50–100+ columns is normal and correct in Kimball)
- **Denormalize**: Avoid snowflaking — flatten hierarchies into the dimension table

### Date Dimension (`Dim_Date`)
- **Never use a date/time column as a direct FK** — always join through a date dimension
- Populated for a range (e.g., 1990–2040), one row per calendar day
- Standard columns: `DateKey` (INT YYYYMMDD), `FullDate`, `CalendarYear`, `CalendarQuarter`, `CalendarMonth`, `CalendarMonthName`, `CalendarWeek`, `DayOfWeek`, `DayName`, `IsWeekend`, `IsHoliday`, `FiscalYear`, `FiscalQuarter`, `FiscalPeriod`
- **Fiscal calendar**: Must match the organization's fiscal year definition exactly

### Time-of-Day Dimension (`Dim_Time`)
- Only needed when grain includes sub-day intervals
- `TimeKey` (INT HHMMSS), `Hour`, `Minute`, `Second`, `AMPMIndicator`, `ShiftName`

### Role-Playing Dimension
- The same physical dimension referenced multiple times in a single fact table under different roles
- **SQL Server pattern**: Create views over `Dim_Date` — e.g., `vDim_OrderDate`, `vDim_ShipDate`, `vDim_DueDate`
- **SSAS Tabular**: Create multiple relationships and use `USERELATIONSHIP()` in DAX

### Junk Dimension (`Dim_TransactionType` or `Dim_Indicator`)
- Collects low-cardinality indicator flags and codes from the fact table
- Prevents fact table pollution with flag columns
- Cross-join all combinations at ETL time for small cardinality sets
- **Example**: Combine `IsOnline` + `IsPromotion` + `IsRefund` into one `Dim_TransactionFlag`

### Degenerate Dimension
- A dimensional attribute that lives in the fact table because it has no other dimension members
- **Example**: Invoice number, check number, transaction ID
- No separate dimension table; store directly in fact table with `_DD` suffix convention or just as a string column

### Outrigger Dimension
- A secondary dimension that attaches to a primary dimension (not directly to the fact table)
- **Use sparingly** — it is a controlled form of snowflaking
- **Example**: `Dim_Geography` → `Dim_Country` (acceptable if country has its own attributes)

### Bridge Table (for Many-to-Many)
- Required when a fact row legitimately relates to multiple dimension members
- Contains surrogate key pairs + optional `WeightingFactor`
- **Example**: `Bridge_ClaimDiagnosis` (one claim, many ICD-10 codes)
- Query pattern: Always aggregate through bridge with `DISTINCTCOUNT` guard

### Parent-Child / Ragged Hierarchy
- Variable-depth organizational hierarchies (org chart, account hierarchy)
- **Kimball approach**: Recursive bridge table with explicit depth levels
- **SSAS Tabular approach**: Use DAX `PATH()` / `PATHITEM()` / `PATHLENGTH()` functions
- Fixed-depth hierarchies are always preferred when depth is known

---

## Slowly Changing Dimensions (SCD)

### Type 0 — Retain Original
- Attribute never changes once set
- **Use when**: Birth date, original enrollment date, account open date

### Type 1 — Overwrite
- Replace old value with new value; no history retained
- **Use when**: Error corrections, non-analytical attributes
- **Implementation**: `UPDATE Dim_Customer SET City = @NewCity WHERE CustomerKey = @Key`

### Type 2 — Add New Row (Most Common)
- New row added with new `EffectiveDate` / `ExpirationDate`; old row expires
- **Required columns**: `RowEffectiveDate DATE NOT NULL`, `RowExpirationDate DATE NULL` (NULL = current), `IsCurrent BIT NOT NULL`
- **Surrogate key**: New row gets new surrogate key; natural key repeats
- **ETL pattern**: MERGE statement with `WHEN MATCHED AND attribute_changed THEN expire old + insert new`

```sql
-- SCD Type 2 standard columns
ALTER TABLE Dim_Customer ADD
    RowEffectiveDate DATE NOT NULL DEFAULT '1900-01-01',
    RowExpirationDate DATE NULL,        -- NULL = currently active
    IsCurrent BIT NOT NULL DEFAULT 1,
    RowCreatedDate DATETIME NOT NULL DEFAULT GETDATE(),
    RowUpdatedDate DATETIME NOT NULL DEFAULT GETDATE();
```

### Type 3 — Add New Attribute (Limited History)
- Add a `Current_X` and `Prior_X` column pair
- **Use when**: Only one prior value needed (e.g., `CurrentTerritory`, `PriorTerritory`)
- **Limitation**: Only captures one prior change; not scalable

### Type 4 — Mini-Dimension
- Rapidly changing attributes split into a separate smaller dimension
- **Use when**: Attributes change so frequently that SCD Type 2 creates row explosion
- **Example**: Customer demographics (age band, income band, credit score tier) split from `Dim_Customer` into `Dim_CustomerProfile`

### Type 6 — Combined (1 + 2 + 3)
- Row-versioning (Type 2) + current attribute copy on all rows (Type 1) + prior attribute (Type 3)
- Most common in real-world enterprise DWs
- Required columns: `RowEffectiveDate`, `RowExpirationDate`, `IsCurrent`, `CurrentAttributeValue`, `OriginalAttributeValue`

---

## Enterprise Bus Matrix

The bus matrix documents which dimensions attach to which fact tables. It is the master plan of the enterprise DW.

```
                        | Dim_Date | Dim_Customer | Dim_Product | Dim_Geography | Dim_Employee |
------------------------|----------|--------------|-------------|----------------|--------------|
Fact_SalesTransaction   |    ✓     |      ✓       |      ✓      |       ✓        |      ✓       |
Fact_InventorySnapshot  |    ✓     |              |      ✓      |       ✓        |              |
Fact_CustomerContact    |    ✓     |      ✓       |             |                |      ✓       |
```

**Rules**:
- Every fact table must have at least a Date dimension
- Conformed dimensions use the exact same grain, surrogate key, and attributes across all fact tables
- If `Dim_Customer` in Fact_A and Fact_B have different grains, they are NOT conformed — rename or reconcile

---

## Dimensional Design Anti-Patterns

| Anti-Pattern | Problem | Correct Approach |
|---|---|---|
| Smart keys in fact tables | Embed business logic in key values | Use dumb surrogate integers |
| Snowflaking dimensions | Normalized sub-tables off dimensions | Denormalize into flat dimension table |
| Null foreign keys | Unmapped facts | Create "Unknown" or "Not Applicable" dimension row (key = -1) |
| Measures in dimension tables | Aggregation anomalies | Move to fact table |
| Flags / codes in fact table | Fact table pollution | Junk dimension |
| Date stored as VARCHAR | Cannot use date dim efficiently | Store as INT DateKey (YYYYMMDD) or DATE FK |
| Using source system natural keys | Tight coupling to source, no SCD history | Surrogate keys only |
| Grain mixing in one fact table | Incorrect aggregations | Separate fact tables per grain |

---

## Conformed Dimension Checklist

When declaring a dimension as conformed:
- [ ] Same surrogate key sequence used across all fact tables
- [ ] Same natural key attribute retained (for debugging/tracing)
- [ ] Same attribute names and data types in all consuming fact tables
- [ ] Shared ETL process maintains the master dimension
- [ ] Changes to dimension grain communicated to all fact table ETL owners
- [ ] Date dimension shared: same `DateKey` format (INT YYYYMMDD recommended) and same fiscal calendar definition

---

## ETL / Staging Pattern

```
Source System → Landing/Raw Zone → Staging → Dimension Loads → Fact Loads
                                  (SCD logic here)
```

- **Landing/Raw**: Full copy of source, no transformation, timestamped
- **Staging**: Cleaned, typed, deduped — no surrogate keys yet
- **Dimension loads**: Apply SCD logic, assign surrogate keys
- **Fact loads**: Look up surrogate keys from dimensions, reject unmapped rows to error table
- **Error table**: Every unmapped FK should route to an error/audit fact table, not silently drop
