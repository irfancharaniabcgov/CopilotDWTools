# Kimball Dimensional Modeling Reference

Based on Ralph Kimball's *The Data Warehouse Toolkit* (3rd ed.) and the Kimball Group methodology.
Supplemented with SQL Server 2016–2022 on-premises physical design guidance.

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
| **ODS** | Operational Data Store — a subject-oriented, integrated, volatile, current-valued store used as a staging hub before the DW. |
| **Audit Column** | System metadata columns (RowCreatedDate, RowUpdatedDate, ETLBatchID, etc.) added to every DW table for lineage and troubleshooting. |
| **Late-Arriving Fact** | A fact row whose effective date falls in a past period; requires retroactive SCD Type 2 key resolution. |
| **Late-Arriving Dimension** | A dimension member that does not exist at fact load time; requires an "unknown" surrogate key placeholder. |

---

## Fact Table Types

### Transaction Fact Table
- One row per discrete business event (sale, claim, call).
- Lowest grain; most flexible; largest rowcount.
- Design: many FK columns to dimensions, few additive measures.

### Periodic Snapshot Fact Table
- One row per entity per standard time period (daily balance, monthly inventory).
- Grain: entity + calendar period.
- Includes semi-additive measures (balances) — do **not** SUM across time in SQL; handle in DAX with `LASTDATE` / `CALCULATE`.

### Accumulating Snapshot Fact Table
- One row per long-lived process instance (order lifecycle, claim lifecycle).
- Multiple date FKs (OrderDate, ShipDate, DeliveryDate, etc.) — each a FK to `DimDate`.
- Rows are **updated** as milestones are reached (unique among fact tables).
- Lag measures (days between milestones) are calculated columns or DAX measures.

### Factless Fact Table
- Records the occurrence of an event with no numeric measure (student attendance, product-promotion coverage).
- Used for coverage queries: "Which products were on promotion but had zero sales?"

---

## Dimension Table Types

### Standard (Type 2) Dimension
Full history tracking. New row per change. Surrogate key FK in fact table.

### Date Dimension
- Must be pre-populated for the full date range of the DW (e.g., 1990-01-01 → 2049-12-31).
- Never use a DATETIME or DATE column as a FK — always use an integer DateKey in `YYYYMMDD` format.
- Include fiscal calendar attributes aligned to your organization's fiscal year.
- Include `IsWeekend`, `IsHoliday`, `FiscalQuarter`, `FiscalYear`, `RelativeDayOffset` (days from today, useful for rolling windows).

```sql
-- Minimal Date Dimension scaffold
CREATE TABLE dbo.DimDate (
    DateKey        INT           NOT NULL PRIMARY KEY,  -- YYYYMMDD
    FullDate       DATE          NOT NULL,
    DayOfWeek      TINYINT       NOT NULL,
    DayName        VARCHAR(10)   NOT NULL,
    WeekOfYear     TINYINT       NOT NULL,
    MonthNumber    TINYINT       NOT NULL,
    MonthName      VARCHAR(10)   NOT NULL,
    CalendarQtr    TINYINT       NOT NULL,
    CalendarYear   SMALLINT      NOT NULL,
    FiscalMonth    TINYINT       NOT NULL,
    FiscalQtr      TINYINT       NOT NULL,
    FiscalYear     SMALLINT      NOT NULL,
    IsWeekend      BIT           NOT NULL DEFAULT 0,
    IsHoliday      BIT           NOT NULL DEFAULT 0,
    HolidayName    VARCHAR(50)   NULL,
    -- Audit
    RowCreatedDate DATETIME2(0)  NOT NULL DEFAULT SYSUTCDATETIME(),
    ETLBatchID     INT           NOT NULL DEFAULT -1
);
```

### Time-of-Day Dimension
Separate from Date. Integer TimeKey in `HHMM` or seconds-since-midnight.
Only include if grain requires intra-day analysis (call center, trading).

### Role-Playing Dimension
A single physical dimension table aliased multiple times in a fact table.
Example: `DimDate` plays `OrderDateKey`, `ShipDateKey`, `DueDateKey`.
In SSAS Tabular: create inactive relationships; activate with `USERELATIONSHIP()`.

```sql
-- Fact table with role-playing date keys
ALTER TABLE dbo.FactSalesOrder ADD
    OrderDateKey   INT NOT NULL REFERENCES dbo.DimDate(DateKey),
    ShipDateKey    INT     NULL REFERENCES dbo.DimDate(DateKey),
    DueDateKey     INT     NULL REFERENCES dbo.DimDate(DateKey);
```

### Junk Dimension
Combines low-cardinality flags and indicators (IsOnline, IsGift, IsRush) into a single dimension.
Pre-populate with the Cartesian product of all flag combinations.
Reduces fact table width and eliminates many tiny dimensions.

### Degenerate Dimension
A dimension attribute stored directly on the fact table (no separate dimension table).
Classic examples: Invoice Number, Order Number, Transaction ID.
Has no attributes beyond the key itself.

### Outrigger Dimension
A secondary dimension hanging off a primary dimension (avoid unless necessary).
Usually a sign of an over-normalized model. Prefer flattening into the main dimension.

### Bridge Table
Handles multi-valued dimension relationships (e.g., account–customer many-to-many).
Pattern: FactTable → BridgeTable → DimTable.
Each bridge row has a weighting factor if shares must sum to 1.0.

### Parent-Child (Ragged Hierarchy) Dimension
Recursive self-referencing structure (org chart, account hierarchy).
In Kimball: flatten to a fixed-depth dimension or use a bridge.
In SSAS Tabular: use PATH/PATHITEM DAX functions for dynamic depth.

---

## Slowly Changing Dimensions (SCD)

### Type 0 — Retain Original
Attribute never changes after first load. Source updates are ignored.
Use for: Birth Date, Original Contract Date.

### Type 1 — Overwrite
Overwrite the old value; no history retained.
Use for: Correcting typos, updating email addresses where history is irrelevant.

### Type 2 — Add New Row (Full History)
New row added for each change. Surrogate key increments.
Requires: `RowEffectiveDate`, `RowExpirationDate`, `IsCurrent` flag.

#### Standard SCD Type 2 Column Set
```sql
-- Required columns on every SCD Type 2 dimension
EmployeeSK        INT           NOT NULL IDENTITY(1,1) PRIMARY KEY,
EmployeeNK        INT           NOT NULL,   -- natural/business key from source
-- ... business attribute columns ...
RowEffectiveDate  DATE          NOT NULL,
RowExpirationDate DATE          NOT NULL DEFAULT '9999-12-31',
IsCurrent         BIT           NOT NULL DEFAULT 1,
-- Audit columns (see Audit Columns Standard section)
RowCreatedDate    DATETIME2(0)  NOT NULL DEFAULT SYSUTCDATETIME(),
RowUpdatedDate    DATETIME2(0)  NOT NULL DEFAULT SYSUTCDATETIME(),
ETLBatchID        INT           NOT NULL DEFAULT -1,
SourceSystemID    TINYINT       NOT NULL DEFAULT 1
```

#### SQL Server MERGE Implementation for SCD Type 2

```sql
-- ============================================================
-- SCD Type 2 MERGE pattern for DimEmployee
-- Assumes staging table: stg.Employee
-- ============================================================
BEGIN TRANSACTION;

-- Step 1: Expire rows where tracked attributes have changed
UPDATE dw.DimEmployee
SET    RowExpirationDate = CAST(GETDATE() AS DATE),
       IsCurrent         = 0,
       RowUpdatedDate    = SYSUTCDATETIME(),
       ETLBatchID        = @BatchID
WHERE  IsCurrent = 1
  AND  EmployeeNK IN (
        SELECT s.EmployeeID
        FROM   stg.Employee s
        JOIN   dw.DimEmployee d
               ON  d.EmployeeNK = s.EmployeeID
               AND d.IsCurrent  = 1
        WHERE  -- List every tracked (Type 2) attribute here
               d.LastName       <> s.LastName
            OR d.DepartmentName <> s.DepartmentName
            OR d.JobTitle       <> s.JobTitle
       );

-- Step 2: Insert new current rows for changed + brand-new members
INSERT INTO dw.DimEmployee (
    EmployeeNK, FirstName, LastName, DepartmentName, JobTitle,
    RowEffectiveDate, RowExpirationDate, IsCurrent,
    RowCreatedDate, RowUpdatedDate, ETLBatchID, SourceSystemID
)
SELECT
    s.EmployeeID,
    s.FirstName,
    s.LastName,
    s.DepartmentName,
    s.JobTitle,
    CAST(GETDATE() AS DATE)  AS RowEffectiveDate,
    '9999-12-31'             AS RowExpirationDate,
    1                        AS IsCurrent,
    SYSUTCDATETIME()         AS RowCreatedDate,
    SYSUTCDATETIME()         AS RowUpdatedDate,
    @BatchID                 AS ETLBatchID,
    1                        AS SourceSystemID
FROM stg.Employee s
WHERE NOT EXISTS (
    SELECT 1
    FROM   dw.DimEmployee d
    WHERE  d.EmployeeNK = s.EmployeeID
      AND  d.IsCurrent  = 1
);

-- Step 3: Type 1 overwrites (non-tracked attributes like email)
UPDATE d
SET    d.EmailAddress  = s.EmailAddress,
       d.RowUpdatedDate = SYSUTCDATETIME(),
       d.ETLBatchID    = @BatchID
FROM   dw.DimEmployee d
JOIN   stg.Employee   s ON d.EmployeeNK = s.EmployeeID
WHERE  d.IsCurrent = 1
  AND  d.EmailAddress <> s.EmailAddress;

COMMIT TRANSACTION;
```

> **Performance tip:** Add a non-clustered index on `(EmployeeNK, IsCurrent)` to accelerate the IsCurrent lookups. On large dimensions (>5M rows), consider a filtered index: `WHERE IsCurrent = 1`.

### Type 3 — Add New Column
Adds a "Previous Value" column alongside current value.
Supports only one prior change; limited history.
Use for: "Show current and prior year territory."

### Type 4 — Mini-Dimension / Rapid Change
Splits fast-changing attributes into a separate small "mini-dimension."
Fact table carries FK to both the main dimension (current SK) and the mini-dimension.
Example: Customer demographic band (age range, income tier) changes frequently.

### Type 6 — Hybrid (1+2+3)
Combines Type 1 overwrite + Type 2 new row + Type 3 prior-value column.
"Current" attribute columns are kept in sync across all rows for the natural key (Type 1 behavior).
Allows "as-was" and "as-is" reporting from the same row.

---

## Enterprise Bus Matrix

The Bus Matrix is the master blueprint of the DW. Rows = fact tables (business processes). Columns = dimensions.

| Business Process | Date | Customer | Product | Employee | Store | Order |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| Sales Orders | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Returns | ✓ | ✓ | ✓ | | ✓ | ✓ |
| Inventory Snapshot | ✓ | | ✓ | | ✓ | |
| HR Headcount Snapshot | ✓ | | | ✓ | ✓ | |

**Rules:**
- Dimensions must be conformed (same surrogate key, same attributes) to enable drill-across.
- A ✓ in the matrix = a FK in the fact table.
- Build the Bus Matrix before any physical design; validate with business stakeholders.

---

## Dimensional Design Anti-Patterns

| Anti-Pattern | Problem | Correct Approach |
|---|---|---|
| Snowflaking dimensions | Joins at query time; no performance benefit in columnar storage | Flatten into single wide dimension |
| Using natural keys as FKs | Breaks SCD; ties DW to source system changes | Always use surrogate keys |
| Storing measures in dimension | Mixes grain; causes query complexity | Move to appropriate fact table |
| Fact table without declared grain | Ambiguous rows; incorrect aggregations | Declare grain before design |
| Nulls in FK columns | Causes SSAS relationship errors; DAX blank propagation | Use -1 "Unknown" surrogate key |
| Date stored as VARCHAR | Prevents date math; prevents DimDate join | Use INT (YYYYMMDD) or DATE type |
| One big fact table (God Fact) | Violates single grain principle | Split by business process and grain |
| Over-using Type 2 SCD | Table bloat; slow lookups; SSAS processing cost | Only track attributes business needs historically |

---

## Conformed Dimension Checklist

Before declaring a dimension "conformed" across fact tables:

- [ ] Surrogate key is the same physical column (same data type, same domain).
- [ ] Natural/business key is stored as an attribute.
- [ ] Attribute names and data types are identical across all fact tables referencing this dimension.
- [ ] A single ETL process owns the dimension load (no duplicate SCD logic in multiple packages).
- [ ] The dimension is registered in the Enterprise Bus Matrix.
- [ ] A "subset" conformed dimension documents which columns are excluded for each subject area.
- [ ] The Unknown member (-1 SK) is defined with descriptive N/A values (not NULL or "Unknown").

---

## ODS (Operational Data Store) Layer

### Definition and Purpose
An **ODS** is a subject-oriented, integrated, volatile, current-valued data store that sits between source systems and the Data Warehouse. It is **not** a Data Warehouse — it holds current state only (no history by default) and is designed for operational queries and reporting with near-real-time latency.

### When to Use an ODS

| Scenario | Use ODS? |
|---|---|
| Operational staff need current-state reports (today's open orders) | ✓ Yes |
| Source system cannot be queried directly for reporting | ✓ Yes |
| Multiple source systems need to be integrated before DW load | ✓ Yes |
| Full historical analysis required | ✗ No — use DW |
| High-volume analytical queries with large date ranges | ✗ No — use DW |

### ODS vs. Staging vs. DW

| Layer | Volatility | Grain | History | Audience |
|---|---|---|---|---|
| **Staging** | Transient (truncate/reload) | Raw source rows | None — replaced each run | ETL only |
| **ODS** | Current-value, updatable | Subject-oriented, integrated | Minimal (current + 1 prior version) | Operational users, ETL feed |
| **DW (Kimball)** | Append-only (facts), SCD (dims) | Declared business grain | Full history | Analysts, BI tools |

### ODS Implementation Pattern (SQL Server)

```sql
-- ODS tables follow source structure but with integration keys
-- and audit columns. Use MERGE for upsert pattern.

CREATE TABLE ods.Customer (
    CustomerID      INT           NOT NULL PRIMARY KEY,  -- source NK
    CustomerName    VARCHAR(200)  NOT NULL,
    Email           VARCHAR(150)  NULL,
    Phone           VARCHAR(20)   NULL,
    AddressLine1    VARCHAR(200)  NULL,
    City            VARCHAR(100)  NULL,
    StateCode       CHAR(2)       NULL,
    PostalCode      VARCHAR(10)   NULL,
    -- Integration
    SourceSystemID  TINYINT       NOT NULL,
    -- Audit
    ODSInsertDate   DATETIME2(0)  NOT NULL DEFAULT SYSUTCDATETIME(),
    ODSUpdateDate   DATETIME2(0)  NOT NULL DEFAULT SYSUTCDATETIME(),
    ODSBatchID      INT           NOT NULL DEFAULT -1,
    IsActive        BIT           NOT NULL DEFAULT 1
);

-- MERGE-based ODS upsert from staging
MERGE ods.Customer AS tgt
USING stg.Customer  AS src
ON    tgt.CustomerID = src.CustomerID
  AND tgt.SourceSystemID = src.SourceSystemID
WHEN MATCHED AND (
         tgt.CustomerName <> src.CustomerName
      OR ISNULL(tgt.Email,'') <> ISNULL(src.Email,'')
      OR ISNULL(tgt.Phone,'') <> ISNULL(src.Phone,'')
    ) THEN
    UPDATE SET
        CustomerName   = src.CustomerName,
        Email          = src.Email,
        Phone          = src.Phone,
        ODSUpdateDate  = SYSUTCDATETIME(),
        ODSBatchID     = @BatchID
WHEN NOT MATCHED BY TARGET THEN
    INSERT (CustomerID, CustomerName, Email, Phone,
            SourceSystemID, ODSInsertDate, ODSUpdateDate, ODSBatchID)
    VALUES (src.CustomerID, src.CustomerName, src.Email, src.Phone,
            src.SourceSystemID, SYSUTCDATETIME(), SYSUTCDATETIME(), @BatchID)
WHEN NOT MATCHED BY SOURCE AND tgt.SourceSystemID = @SourceSystemID THEN
    UPDATE SET IsActive = 0, ODSUpdateDate = SYSUTCDATETIME();
```

### Layer Architecture (Three-Layer)
```
Source Systems
     │
     ▼
┌──────────────┐   Truncate/reload each run
│   Staging    │   Raw source format, no transformation
└──────────────┘
     │
     ▼
┌──────────────┐   MERGE upsert, current state only
│     ODS      │   Integrated, cleansed, conformed NKs
└──────────────┘
     │
     ▼
┌──────────────┐   SCD Type 2, surrogate keys, star schema
│  Data Warehouse│  Full history, Kimball dimensional model
└──────────────┘
     │
     ▼
┌──────────────┐
│  SSAS Tabular│   Semantic layer, DAX, PBIRS reports
└──────────────┘
```

---

## Data Vault 2.0 Awareness

### Overview
Data Vault 2.0 (Dan Linstedt) is an alternative modeling methodology to Kimball. Many organizations run **hybrid** architectures where Data Vault is used in the Raw/Integration Vault layer and Kimball Star Schema is used in the Information Mart (presentation layer fed to SSAS).

### Key Constructs

| Construct | Role | Kimball Equivalent |
|---|---|---|
| **Hub** | Stores business keys only (no attributes) | Natural key in a Dimension |
| **Link** | Stores many-to-many relationships between hubs | Bridge table or Fact table |
| **Satellite** | Stores descriptive attributes + full history with load date | SCD Type 2 Dimension |

### Kimball vs. Data Vault

| Dimension | Kimball Star Schema | Data Vault 2.0 |
|---|---|---|
| Primary use case | Presentation / BI / SSAS feed | Raw integration, auditability, agility |
| History model | SCD Types 0–6 (design choice) | All history, always (satellites append-only) |
| Query complexity | Low (star joins) | High (hub+satellite joins before mart) |
| Loading pattern | SCD MERGE | Insert-only (no updates) |
| SSAS compatibility | Direct — star schema maps naturally | Requires Information Mart (star) as intermediate |
| ETL change impact | Re-engineering of dimensions | Add new satellite (non-breaking) |

### Hybrid Pattern for On-Premises SQL Server
```
Raw Sources → [Staging] → [Raw Vault (Hubs + Links + Satellites)]
                                       │
                          [Business Vault (computed satellites)]
                                       │
                          [Information Mart (Star Schema / Kimball)]
                                       │
                              [SSAS Tabular Model]
                                       │
                              [PBIRS Reports]
```

> **Recommendation:** If your organization already has a Data Vault, **do not** try to point SSAS Tabular directly at vault tables. Always build a Kimball-style Information Mart (star schema) as the semantic layer between the vault and SSAS. DAX and SSAS relationships are optimized for star schema, not vault topology.

---

## SQL Server–Specific DW Physical Design

### Columnstore Indexes on Fact Tables

Columnstore indexes (introduced SQL Server 2012, mature from 2016+) are the **primary performance optimization** for DW fact tables. They compress data by column and enable batch-mode execution.

```sql
-- Clustered Columnstore Index on a fact table (recommended for all large fact tables)
-- Drop any existing clustered index first, then:
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactSalesOrder
ON dbo.FactSalesOrder
WITH (DATA_COMPRESSION = COLUMNSTORE_ARCHIVE,  -- for cold/historical partitions
      MAXDOP = 4);                             -- limit parallelism during rebuild

-- For the active/hot partition, use COLUMNSTORE (no ARCHIVE) for faster query:
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactSalesOrder
ON dbo.FactSalesOrder
WITH (DATA_COMPRESSION = COLUMNSTORE, MAXDOP = 4);
```

**Columnstore guidelines for DW fact tables:**
- Use **Clustered Columnstore Index (CCI)** on all fact tables > 1M rows.
- CCI eliminates the need for most traditional non-clustered indexes on fact tables.
- Row groups are 1,048,576 rows each. Bulk-load in batches ≥ 102,400 rows to avoid "delta store" accumulation.
- Run `ALTER INDEX ... REORGANIZE` (not REBUILD) on CCIs during maintenance windows — REORGANIZE is online.
- Check delta store health with `sys.column_store_row_groups`.

```sql
-- Monitor CCI row group health
SELECT  i.name,
        rg.state_desc,
        rg.total_rows,
        rg.deleted_rows,
        rg.size_in_bytes / 1024.0 / 1024.0 AS SizeMB
FROM    sys.column_store_row_groups rg
JOIN    sys.indexes i ON i.object_id = rg.object_id AND i.index_id = rg.index_id
WHERE   i.type IN (5, 6)   -- 5=CCI, 6=NCCI
ORDER   BY rg.object_id, rg.row_group_id;
```

### Table Partitioning Strategy

Partition large fact tables and periodic snapshot tables by the most common filter column (usually `DateKey` or a date range column).

```sql
-- Step 1: Create partition function (monthly for a sales fact table)
CREATE PARTITION FUNCTION PF_Monthly (INT)
AS RANGE RIGHT FOR VALUES (
    20230101, 20230201, 20230301, 20230401, 20230501, 20230601,
    20230701, 20230801, 20230901, 20231001, 20231101, 20231201,
    20240101  -- add each new month before the month begins
);

-- Step 2: Create partition scheme
CREATE PARTITION SCHEME PS_Monthly
AS PARTITION PF_Monthly
ALL TO ([PRIMARY]);          -- route all to PRIMARY; adjust for filegroup strategy

-- Step 3: Create fact table on partition scheme
CREATE TABLE dbo.FactSalesOrder (
    SalesOrderSK   BIGINT        NOT NULL,
    OrderDateKey   INT           NOT NULL,
    CustomerSK     INT           NOT NULL,
    ProductSK      INT           NOT NULL,
    SalesAmount    DECIMAL(18,4) NOT NULL,
    Quantity       INT           NOT NULL,
    -- Audit
    RowCreatedDate DATETIME2(0)  NOT NULL DEFAULT SYSUTCDATETIME(),
    ETLBatchID     INT           NOT NULL DEFAULT -1
) ON PS_Monthly (OrderDateKey);

-- Step 4: Create CCI aligned to partition scheme
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactSalesOrder
ON dbo.FactSalesOrder
ON PS_Monthly (OrderDateKey);
```

**Partition switching for ETL (zero-downtime load):**
```sql
-- Load new month data into a staging table, then switch in
CREATE TABLE stg.FactSalesOrder_202401 (... same schema as dbo.FactSalesOrder ...)
ON [PRIMARY];

-- Populate staging, then atomic switch
ALTER TABLE stg.FactSalesOrder_202401
    SWITCH TO dbo.FactSalesOrder PARTITION
    $PARTITION.PF_Monthly(20240101);
```

### Filegroup Strategy

Distribute I/O by placing different DW layers on separate filegroups backed by different storage tiers.

```sql
-- Recommended filegroup layout for a DW database
ALTER DATABASE [DW_Production] ADD FILEGROUP [FG_Dimensions];
ALTER DATABASE [DW_Production] ADD FILEGROUP [FG_Facts_Hot];    -- current 2 years
ALTER DATABASE [DW_Production] ADD FILEGROUP [FG_Facts_Cold];   -- historical, read-only

ALTER DATABASE [DW_Production]
ADD FILE (NAME = 'DW_Dims',    FILENAME = 'D:\SQLData\DW_Dims.ndf',    SIZE = 512MB, FILEGROWTH = 256MB)
TO FILEGROUP [FG_Dimensions];

ALTER DATABASE [DW_Production]
ADD FILE (NAME = 'DW_Hot',     FILENAME = 'D:\SQLData\DW_Hot.ndf',     SIZE = 4096MB, FILEGROWTH = 1024MB)
TO FILEGROUP [FG_Facts_Hot];

ALTER DATABASE [DW_Production]
ADD FILE (NAME = 'DW_Cold',    FILENAME = 'E:\SQLArchive\DW_Cold.ndf', SIZE = 8192MB, FILEGROWTH = 2048MB)
TO FILEGROUP [FG_Facts_Cold];

-- Mark cold filegroup read-only after archiving old partitions
ALTER DATABASE [DW_Production] MODIFY FILEGROUP [FG_Facts_Cold] READ_ONLY;
```

### Statistics Management

Statistics are critical for the Query Optimizer on DW tables — especially with skewed date distributions.

```sql
-- Update statistics on all DW tables with full scan (not sampled)
-- Run after each nightly ETL load
USE [DW_Production];
EXEC sp_updatestats;          -- quick sampled update of stale statistics

-- Full scan update for fact tables where skew is suspected
UPDATE STATISTICS dbo.FactSalesOrder WITH FULLSCAN;

-- Create additional statistics on frequently filtered non-key columns
CREATE STATISTICS STAT_FactSalesOrder_Channel
ON dbo.FactSalesOrder (SalesChannelCode);

-- Auto-update statistics async (set at database level)
ALTER DATABASE [DW_Production] SET AUTO_UPDATE_STATISTICS ON;
ALTER DATABASE [DW_Production] SET AUTO_UPDATE_STATISTICS_ASYNC ON;
```

---

## Surrogate Key Generation Strategies

### 1. IDENTITY (Simplest — Recommended for Most Cases)

```sql
-- Standard IDENTITY surrogate key
CREATE TABLE dbo.DimCustomer (
    CustomerSK  INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    CustomerNK  INT NOT NULL,
    ...
);
-- Note: Reserve negative values for special members
-- -1 = Unknown, -2 = Not Applicable, -3 = Error/Pending
-- Seed IDENTITY at 1; insert special members with SET IDENTITY_INSERT ON
SET IDENTITY_INSERT dbo.DimCustomer ON;
INSERT dbo.DimCustomer (CustomerSK, CustomerNK, CustomerName, ...)
VALUES (-1, -1, 'Unknown', ...),
       (-2, -2, 'Not Applicable', ...);
SET IDENTITY_INSERT dbo.DimCustomer OFF;
```

### 2. SEQUENCE Object (SQL Server 2012+)

Use SEQUENCE when surrogate keys must be coordinated across tables or generated outside INSERT statements (e.g., pre-assigned in ETL pipelines).

```sql
-- Create a shared sequence for a dimension family
CREATE SEQUENCE dbo.SEQ_DimCustomer
    AS INT
    START WITH 1
    INCREMENT BY 1
    MINVALUE 1
    NO MAXVALUE
    NO CYCLE
    CACHE 1000;          -- cache 1000 values for performance

-- Use in ETL
DECLARE @NewSK INT = NEXT VALUE FOR dbo.SEQ_DimCustomer;

-- Use DEFAULT in table definition
CREATE TABLE dbo.DimCustomer (
    CustomerSK INT NOT NULL DEFAULT (NEXT VALUE FOR dbo.SEQ_DimCustomer) PRIMARY KEY,
    ...
);
```

### 3. MERGE-Based Key Lookup (ETL Pattern)

When loading fact tables, resolve natural keys to surrogate keys via lookup. Always include the Unknown member fallback.

```sql
-- Surrogate key lookup with Unknown member fallback
-- Used in ETL staging → fact load
INSERT INTO dbo.FactSalesOrder (
    OrderDateKey, CustomerSK, ProductSK, SalesAmount, Quantity,
    RowCreatedDate, ETLBatchID
)
SELECT
    ISNULL(dd.DateKey,  -1)  AS OrderDateKey,
    ISNULL(dc.CustomerSK, -1) AS CustomerSK,
    ISNULL(dp.ProductSK,  -1) AS ProductSK,
    s.SalesAmount,
    s.Quantity,
    SYSUTCDATETIME(),
    @BatchID
FROM stg.SalesOrder s
LEFT JOIN dbo.DimDate d
    ON  d.FullDate = CAST(s.OrderDate AS DATE)
LEFT JOIN dbo.DimDate dd ON dd.DateKey = d.DateKey
LEFT JOIN dbo.DimCustomer dc
    ON  dc.CustomerNK = s.CustomerID
    AND dc.IsCurrent  = 1
LEFT JOIN dbo.DimProduct dp
    ON  dp.ProductNK  = s.ProductCode
    AND dp.IsCurrent  = 1;
```

---

## Late-Arriving Dimension Members

### Problem
A fact row arrives referencing a natural key (e.g., CustomerID = 9999) that does **not yet exist** in the dimension. A direct join returns NULL → if you use INNER JOIN, the fact is silently dropped.

### Solution: Unknown Member Pattern

1. **Pre-populate** the dimension with a surrogate key of `-1` representing "Unknown" or "Pending" for each dimension.
2. **ETL fact load**: Use `LEFT JOIN ... ISNULL(SK, -1)` (see surrogate key lookup above).
3. **Backfill job**: Run a periodic job to re-resolve fact rows with SK = -1 once the dimension member arrives.

```sql
-- Backfill previously unresolved fact rows
UPDATE f
SET    f.CustomerSK = dc.CustomerSK,
       f.ETLBatchID = @BatchID
FROM   dbo.FactSalesOrder f
JOIN   stg.SalesOrder_Unresolved u ON u.SalesOrderNK = f.SalesOrderNK
JOIN   dbo.DimCustomer dc
       ON  dc.CustomerNK = u.CustomerID
       AND dc.IsCurrent  = 1
WHERE  f.CustomerSK = -1;
```

### Impact on SSAS Tabular — Default/Unknown Member
In SSAS Tabular with live PBIRS connections:
- Rows with SK = -1 will appear under the "Unknown" member in slicers.
- Ensure the `-1` row exists in the dimension with user-friendly label (e.g., "Unknown Customer") — otherwise SSAS will show a blank label, which confuses users.
- The SSAS relationship integrity will not fail as long as the `-1` SK is present in the dimension.
- Do **not** use `NULL` as FK in fact tables for SSAS — SSAS treats NULL FK differently (excluded from aggregation, not grouped under Unknown).

---

## Late-Arriving Facts

### Problem
A fact row arrives with a transaction date that falls in a **past period** during which the related dimension members had different attribute values (SCD Type 2 history). Simply resolving to `IsCurrent = 1` assigns the wrong dimension SK.

### Solution: Point-in-Time SCD Type 2 Resolution

```sql
-- Resolve dimension SK as of the fact's transaction date
-- Not IsCurrent = 1, but the row that was current AT the transaction date
INSERT INTO dbo.FactSalesOrder (
    OrderDateKey, CustomerSK, ProductSK, SalesAmount, Quantity,
    RowCreatedDate, ETLBatchID
)
SELECT
    ISNULL(dd.DateKey, -1),
    ISNULL(dc.CustomerSK, -1),
    ISNULL(dp.ProductSK, -1),
    s.SalesAmount,
    s.Quantity,
    SYSUTCDATETIME(),
    @BatchID
FROM stg.LateArrivingSalesOrder s
-- Resolve CustomerSK as of OrderDate (point-in-time lookup)
LEFT JOIN dbo.DimCustomer dc
    ON  dc.CustomerNK       = s.CustomerID
    AND CAST(s.OrderDate AS DATE) >= dc.RowEffectiveDate
    AND CAST(s.OrderDate AS DATE) <  COALESCE(dc.RowExpirationDate, '9999-12-31')
LEFT JOIN dbo.DimProduct dp
    ON  dp.ProductNK        = s.ProductCode
    AND CAST(s.OrderDate AS DATE) >= dp.RowEffectiveDate
    AND CAST(s.OrderDate AS DATE) <  COALESCE(dp.RowExpirationDate, '9999-12-31')
LEFT JOIN dbo.DimDate dd
    ON  dd.FullDate = CAST(s.OrderDate AS DATE);
```

**Gotcha:** If no SCD Type 2 row covers the transaction date (data quality gap), fall back to the earliest available row for that NK:

```sql
-- Fallback: use earliest SCD row if point-in-time lookup returns NULL
LEFT JOIN dbo.DimCustomer dc_fallback
    ON  dc_fallback.CustomerNK = s.CustomerID
    AND dc_fallback.RowEffectiveDate = (
            SELECT MIN(RowEffectiveDate)
            FROM   dbo.DimCustomer
            WHERE  CustomerNK = s.CustomerID
        )
```

---

## Audit Columns Standard

**Every table in the Data Warehouse** (staging, ODS, dimension, fact, bridge) must carry the following audit columns. This is non-negotiable for lineage, troubleshooting, and reload scenarios.

| Column | Data Type | Default | Applied To | Purpose |
|---|---|---|---|---|
| `RowCreatedDate` | `DATETIME2(0)` | `SYSUTCDATETIME()` | All tables | When was this row first inserted |
| `RowUpdatedDate` | `DATETIME2(0)` | `SYSUTCDATETIME()` | Dim, ODS | When was this row last modified |
| `ETLBatchID` | `INT` | `-1` | All tables | Foreign key to ETL batch log table |
| `SourceSystemID` | `TINYINT` | `1` | All tables | Identifies the source system of the data |
| `RowHash` | `BINARY(32)` | Computed | Dim, ODS | SHA2_256 hash of tracked attributes for change detection |

### RowHash Change Detection Pattern

```sql
-- Compute hash of tracked attributes during staging load
-- Avoids column-by-column comparison in MERGE
ALTER TABLE stg.Employee ADD
    RowHash AS CAST(
        HASHBYTES('SHA2_256',
            CONCAT_WS('|',
                ISNULL(LastName,  ''),
                ISNULL(FirstName, ''),
                ISNULL(DepartmentName, ''),
                ISNULL(JobTitle, '')
            )
        ) AS BINARY(32)
    ) PERSISTED;

-- In the SCD MERGE, compare single hash column instead of N columns
WHEN MATCHED AND tgt.RowHash <> src.RowHash THEN
    UPDATE SET ...
```

### ETL Batch Log Table

```sql
CREATE TABLE dbo.ETLBatchLog (
    BatchID         INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    BatchName       VARCHAR(200)  NOT NULL,
    StartTime       DATETIME2(0)  NOT NULL DEFAULT SYSUTCDATETIME(),
    EndTime         DATETIME2(0)  NULL,
    Status          VARCHAR(20)   NOT NULL DEFAULT 'Running',  -- Running | Succeeded | Failed
    RowsInserted    BIGINT        NOT NULL DEFAULT 0,
    RowsUpdated     BIGINT        NOT NULL DEFAULT 0,
    RowsDeleted     BIGINT        NOT NULL DEFAULT 0,
    ErrorMessage    NVARCHAR(MAX) NULL,
    ServerName      NVARCHAR(128) NOT NULL DEFAULT @@SERVERNAME,
    ExecutedBy      NVARCHAR(128) NOT NULL DEFAULT SYSTEM_USER
);
```

---

## NULL Handling Strategy

### Core Rule
**No NULLs in surrogate key (FK) columns in fact tables.** NULLs in FK columns cause SSAS relationship violations and inconsistent DAX aggregation behavior.

### Standard NULL Substitution Values

| Scenario | Surrogate Key | Descriptive Attributes |
|---|---|---|
| FK lookup fails (member not found) | `-1` (Unknown) | `'Unknown'` |
| Attribute not applicable for this row type | `-2` (Not Applicable) | `'N/A'` |
| Attribute not yet known / pending | `-3` (Pending) | `'Pending'` |
| Truly optional relationship (nullable FK is by design) | `-4` (None) | `'None'` |

### Dimension Unknown Member Setup

```sql
-- Every dimension must have these reserved rows
-- Insert with IDENTITY_INSERT ON (see Surrogate Key section)
SET IDENTITY_INSERT dbo.DimCustomer ON;
INSERT INTO dbo.DimCustomer (
    CustomerSK, CustomerNK, CustomerName, City, StateCode,
    RowEffectiveDate, RowExpirationDate, IsCurrent,
    RowCreatedDate, RowUpdatedDate, ETLBatchID, SourceSystemID
) VALUES
    (-1, -1, 'Unknown',        'Unknown',  'XX', '1900-01-01', '9999-12-31', 1, SYSUTCDATETIME(), SYSUTCDATETIME(), -1, 0),
    (-2, -2, 'Not Applicable', 'N/A',      'XX', '1900-01-01', '9999-12-31', 1, SYSUTCDATETIME(), SYSUTCDATETIME(), -1, 0),
    (-3, -3, 'Pending',        'Pending',  'XX', '1900-01-01', '9999-12-31', 1, SYSUTCDATETIME(), SYSUTCDATETIME(), -1, 0);
SET IDENTITY_INSERT dbo.DimCustomer OFF;
```

### NULL Handling in Measures (Fact Columns)

- Additive measures (SalesAmount, Quantity): use `0` not NULL — NULL propagates through SUM differently from 0 in DAX.
- Semi-additive measures (Balance): `NULL` is valid and semantically correct — use `LASTNONBLANK` in DAX, not `LASTDATE`.
- Ratio denominators: keep `NULL` and handle division-by-zero in DAX with `DIVIDE()`.

```sql
-- Fact table measure column pattern
SalesAmount    DECIMAL(18,4) NOT NULL DEFAULT 0,    -- additive: no NULLs
CostAmount     DECIMAL(18,4) NOT NULL DEFAULT 0,    -- additive: no NULLs
EndingBalance  DECIMAL(18,4)     NULL,              -- semi-additive: NULL = no record for period
```

---

## ETL / Staging Pattern

### Layer Responsibilities

| Layer | Schema | Responsibility |
|---|---|---|
| Staging | `stg` | Raw extract from sources; truncate/reload each run; no transformations |
| ODS | `ods` | Integrated, current-state upsert; cleansed; surrogate-key–free |
| Dimensions | `dw` | SCD processing; surrogate key assignment |
| Facts | `dw` | Surrogate key lookup; audit column population |
| Archive | `arch` | Rejected/unresolved rows for investigation |

### Staging Pattern (SQL Server)

```sql
-- Staging tables: raw schema matching source, audit appended
CREATE TABLE stg.Customer (
    CustomerID      INT           NOT NULL,
    CustomerName    VARCHAR(200)  NOT NULL,
    Email           VARCHAR(150)  NULL,
    -- Staging audit
    StgLoadDate     DATETIME2(0)  NOT NULL DEFAULT SYSUTCDATETIME(),
    StgBatchID      INT           NOT NULL DEFAULT -1,
    StgSourceFile   VARCHAR(500)  NULL,
    StgRowNumber    BIGINT        NULL
);

-- Truncate before each load (staging is transient)
TRUNCATE TABLE stg.Customer;
```

### Reject / Error Row Handling

```sql
-- Archive rejected rows for investigation
CREATE TABLE arch.FactSalesOrder_Rejects (
    -- All columns of stg.SalesOrder
    RejectReason    VARCHAR(500)  NOT NULL,
    RejectDate      DATETIME2(0)  NOT NULL DEFAULT SYSUTCDATETIME(),
    BatchID         INT           NOT NULL
);

-- During fact load: capture rows that could not be resolved
INSERT INTO arch.FactSalesOrder_Rejects
SELECT s.*, 'CustomerSK not resolvable', SYSUTCDATETIME(), @BatchID
FROM stg.SalesOrder s
WHERE NOT EXISTS (
    SELECT 1 FROM dbo.DimCustomer dc
    WHERE dc.CustomerNK = s.CustomerID AND dc.IsCurrent = 1
);
```
