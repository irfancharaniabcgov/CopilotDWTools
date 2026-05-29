# ELT Patterns Reference — SQL Server Data Warehouse

Based on James Serra's ELT guidance (jamesserra.com), the user's preferred architecture,
and SQL Server best practices for traditional on-premises DW pipelines.

> **This toolkit uses ELT (Extract, Load, Transform), not ETL.**
>
> In ETL, transformations happen inside the integration tool (e.g., SSIS data flows) before reaching the database.
> In ELT, raw data is loaded first into the database (staging), and all transformations happen in T-SQL
> inside the data warehouse — leveraging the SQL Server engine's full power.

---

## Architecture Overview

```
┌─────────────────────┐     SSIS (extract only)      ┌─────────────────────────────────────────┐
│   SOURCE DATABASE   │ ──────────────────────────►  │         DATA WAREHOUSE DATABASE         │
│                     │   raw copy, no transforms    │                                         │
│  usp_Extract_*      │                              │  ┌─────────┐    ┌──────┐   ┌────────┐  │
│  (date-parameterized│                              │  │ Staging │ ►  │ dbo  │ ► │  SSAS  │  │
│   stored procedures)│                              │  │  layer  │    │ DW   │   │Tabular │  │
└─────────────────────┘                              │  └─────────┘    └──────┘   └────────┘  │
                                                     │  (T-SQL transforms everything here)     │
                                                     └─────────────────────────────────────────┘
```

**Key principle** (James Serra): *"The process of moving data from the source to the staging tables is done
in SSIS using data flows, but the process of moving the data from the staging tables to the data warehouse
can be done with T-SQL instead for a performance boost along with the fact that it is usually easier to code
than using SSIS data transformations."*

### Why ELT over ETL for this stack

| Factor | ETL | ELT (preferred) |
|---|---|---|
| Where transforms run | SSIS data flow engine | SQL Server T-SQL engine |
| Performance on large datasets | SSIS memory-bound | SQL Server columnstore + parallel query |
| Maintainability | SSIS XML packages, hard to diff | Plain T-SQL stored procedures, easy to version |
| Debugging | SSIS debugger | SQL Server Profiler, SSMS, extended events |
| Staging visibility | Data never persists mid-flight | Full audit trail — staging tables queryable at any time |
| Source system load | Transform CPU on source side | Minimal source impact — raw extract only |
| Re-runnability | Re-run whole SSIS package | Re-run individual T-SQL SPs independently |

---

## DW Schema Convention

Both reference projects (`EAO_DW`, `SNOW_DW`) use **schema-based separation** instead of naming prefixes.
This is the primary convention for this toolkit.

| Schema | Purpose | Example objects |
|---|---|---|
| `Staging` | Raw data landed by SSIS — truncated and reloaded each run | `Staging.AdviceAndAssistance`, `Staging.project_tasks` |
| `Dimension` | Conformed dimension tables + load SPs | `Dimension.Calendar`, `Dimension.LoadCalendar` |
| `Fact` | Fact tables + load SPs | `Fact.AdviceAndAssistance`, `Fact.LoadAdviceAndAssistance` |
| `Internal` | Control/metadata tables and utility SPs | `Internal.Lineage`, `Internal.IncrementalLoads`, `Internal.GetLastLoadedDate` |
| `SSAS` | Views exposed to the SSAS data source — SSAS account gets `SELECT` here only | `SSAS.[Advice And Assistance]`, `SSAS.[Last Loaded DW]` |
| `Security` | Schema-level permission grants (in SSDT post-deploy) | `GRANT SELECT ON SCHEMA::SSAS TO [dw]` |

> **Why schemas instead of prefixes?**
> Schema-based separation means you never need to write `dim_`, `fact_`, or `stg_` prefixes.
> The schema IS the classification. `SELECT * FROM Dimension.Calendar` is unambiguous.
> Security is also cleaner: `GRANT SELECT ON SCHEMA::SSAS TO [ssas_service_account]` in one line.

### SP Naming Convention

SPs live **in the schema they operate on**, with no `usp_`/`sp_` prefix:

| Pattern | Example | Purpose |
|---|---|---|
| `[Schema].[Load<TableName>]` | `Fact.LoadAdviceAndAssistance` | Load a DW table from staging |
| `[Schema].[Get<Resource>]` | `Internal.GetLastLoadedDate` | Lookup/helper SP |
| `[Schema].[Reset<Scope>]` | `Internal.ResetForFullLoad` | Maintenance SP |
| `Internal.RethrowError` | — | Error re-throw (called from all CATCH blocks) |
| `Internal.ProcedureErrorInsert` | — | Detailed error logging |

Source-side extract SPs (in the source DB, not the DW) use: `dbo.usp_Extract_<TableName>`.

### Internal Control Tables (project pattern)

These four tables live in the `Internal` schema and track all load activity:

```sql
-- Tracks in-progress and completed loads with row count and status
-- Status: 'P' = in progress, 'S' = success, 'F' = failed
-- Type: 'F' = full, 'I' = incremental
CREATE TABLE [Internal].[Lineage] (
    [LineageKey]     INT            IDENTITY (1, 1) NOT NULL PRIMARY KEY,
    [TableName]      NVARCHAR (200) NOT NULL,
    [StartLoad]      DATETIME2 (7)  NOT NULL,
    [FinishLoad]     DATETIME2 (7)  NULL,
    [LastLoadedDate] DATETIME2 (7)  NOT NULL,
    [Status]         NVARCHAR (1)   DEFAULT (N'P') NOT NULL,  -- P=in-progress, S=success, F=failed
    [Type]           NVARCHAR (1)   DEFAULT (N'F') NOT NULL,  -- F=full, I=incremental
    [RowCount]       BIGINT         DEFAULT ((0)) NOT NULL
);

-- High-water mark: last successful load date per table (used for incremental extraction)
CREATE TABLE [Internal].[IncrementalLoads] (
    [LoadDateKey]  INT            IDENTITY (1, 1) NOT NULL PRIMARY KEY,
    [TableName]    NVARCHAR (100) NOT NULL,
    [LoadDate]     DATETIME2 (7)  NOT NULL
);

-- Source system "last updated" tracking (for the Debug tab Source layer)
CREATE TABLE [Internal].[LastUpdatedSource] (
    [UpdateDateKey] INT            IDENTITY (1, 1) NOT NULL PRIMARY KEY,
    [TableName]     NVARCHAR (100) NOT NULL,
    [UpdateDate]    DATETIME2 (7)  NOT NULL
);

-- Error log: captures CATCH block details from any SP
CREATE TABLE [Internal].[ProcedureError] (
    [ErrorID]        INT             IDENTITY (1, 1) NOT NULL PRIMARY KEY,
    [ErrorLine]      INT             NULL,
    [ErrorMessage]   NVARCHAR (4000) NULL,
    [ErrorNumber]    INT             NULL,
    [ErrorProcedure] NVARCHAR (126)  NULL,
    [ErrorSeverity]  INT             NULL,
    [ErrorState]     INT             NULL,
    [DBName]         NVARCHAR (128)  NULL,
    [UserStamp]      NVARCHAR (50)   DEFAULT (suser_sname()) NOT NULL,
    [TimeStamp]      DATETIME        DEFAULT (getutcdate()) NOT NULL
);

-- Schema version tracking: drives versioned post-deployment scripts
CREATE TABLE [Internal].[DbVersion] (
    [VersionNumber] INT      NOT NULL,
    [Updated]       DATETIME NOT NULL DEFAULT GETDATE()
);
```

> **Note:** The generic patterns in Layer 4 below use `dbo.ELT_ControlTable` / `dbo.ELT_BatchLog`
> naming. These are equivalent in purpose to `Internal.Lineage` / `Internal.IncrementalLoads`.
> Either naming approach is valid; the `Internal` schema approach is used by the reference projects.

### Standard SP Error Handling Template

Every load SP uses this pattern (from `Internal.RethrowError`):

```sql
CREATE PROCEDURE [Schema].[LoadTableName]
AS
SET NOCOUNT ON;
SET XACT_ABORT ON;

DECLARE @LastDateLoaded DATETIME2(7);
DECLARE @TableName NVARCHAR(200) = N'Schema.TableName';

BEGIN TRY
    IF @@NESTLEVEL = 1 AND @@TRANCOUNT = 0 BEGIN TRANSACTION

    DECLARE @LineageKey INT = (
        SELECT TOP(1) LineageKey FROM Internal.Lineage
        WHERE TableName = @TableName AND FinishLoad IS NULL
        ORDER BY LineageKey DESC
    );

    -- ... load logic here ...

    UPDATE Internal.Lineage
    SET FinishLoad = SYSDATETIME(), Status = 'S',
        @LastDateLoaded = LastLoadedDate, [RowCount] = @@ROWCOUNT
    WHERE LineageKey = @LineageKey;

    UPDATE Internal.IncrementalLoads
    SET LoadDate = @LastDateLoaded
    WHERE TableName = @TableName;

    IF @@NESTLEVEL = 1 AND @@TRANCOUNT > 0 COMMIT TRANSACTION
    RETURN 0

END TRY
BEGIN CATCH
    IF @@NESTLEVEL = 1 AND @@TRANCOUNT > 0 ROLLBACK TRANSACTION
    EXECUTE Internal.RethrowError
    RETURN ERROR_NUMBER()
END CATCH
```

### Fact Load Pattern — DELETE + INSERT (preferred over MERGE)

```sql
-- Delete existing records that overlap with the current staging window
-- (prevents duplicates on re-run)
DELETE f
FROM Fact.AdviceAndAssistance f
    INNER JOIN Staging.AdviceAndAssistance s
    ON f._SourceAdviceAndAssistanceID = s._SourceAdviceAndAssistanceID;

-- Insert from staging, resolving dimension keys with LEFT JOIN + ISNULL(key, 0)
-- Unknown member (key = 0) must exist in every dimension for unresolvable rows
INSERT INTO Fact.AdviceAndAssistance (...)
SELECT d.DimKey, ISNULL(r.RegionKey, 0), ...
FROM Staging.AdviceAndAssistance s
    INNER JOIN Dimension.Calendar c ON s.ServiceStartDate = c.DateKey
    LEFT  JOIN Dimension.Region   r ON s.RegionCode = r._SourceRegionCode;
```

> **Why DELETE + INSERT over MERGE?**
> Simpler code, easier to read and debug, no MERGE concurrency edge cases.
> Comment from reference project: *"Code here simpler/easier to read than MERGE statement"*.
> Use MERGE only when you need to update existing fact rows in place (rare).

---

## Layer 1: Source Database — Extract Stored Procedures

### Design Principles
- Source SPs are **read-only** — never modify source data
- Accept `@StartDate` and `@EndDate` parameters for **incremental extraction**
- Return a raw, unmodified result set — same column names as source tables
- Filter only to reduce volume (unnecessary rows/columns) — no business transformations
- Use `NOLOCK` or read-committed snapshot isolation to avoid blocking production queries
- Document with extended properties: what it extracts, what date column drives the window

### Standard Source SP Pattern

```sql
-- Source database: CRM, ERP, OLTP system, etc.
CREATE PROCEDURE dbo.usp_Extract_Customer
    @StartDate  DATETIME,
    @EndDate    DATETIME
AS
BEGIN
    SET NOCOUNT ON;

    SELECT
        CustomerID,
        FirstName,
        LastName,
        Email,
        Phone,
        AddressLine1,
        AddressLine2,
        City,
        StateProvince,
        PostalCode,
        CountryCode,
        CustomerTypeCode,
        IsActive,
        CreatedDate,
        ModifiedDate
    FROM dbo.Customer WITH (NOLOCK)
    WHERE ModifiedDate >= @StartDate
      AND ModifiedDate <  @EndDate;
END;
GO

-- Always document the SP
EXEC sys.sp_addextendedproperty
    @name  = N'MS_Description',
    @value = N'Incremental extract for DW load. Returns all Customer rows modified within the @StartDate (inclusive) to @EndDate (exclusive) window. Uses ModifiedDate as the change detection column. Read-only, NOLOCK.',
    @level0type = N'SCHEMA', @level0name = N'dbo',
    @level1type = N'PROCEDURE', @level1name = N'usp_Extract_Customer';
```

### Date Window Strategy

```sql
-- The DW controls the date window via an ETL control table (see Section 4)
-- Source SPs should NEVER hardcode dates or determine their own window

-- Pattern: Closed-open interval [StartDate, EndDate)
-- StartDate = LastSuccessfulLoadEnd from control table
-- EndDate   = GETDATE() at job start (captured once per job run, not per SP call)

-- For append-only sources (no updates), use CreatedDate
-- For mutable sources (rows can be updated), use ModifiedDate or RowVersion/timestamp
-- For sources with neither, use full reload with TRUNCATE + reload pattern
```

### Handling Sources Without a Change Date Column

```sql
-- Option A: Full reload pattern (for small, slowly changing reference tables)
-- Just SELECT * with no date filter — staging TRUNCATE handles idempotency

-- Option B: Row hash comparison
-- In the staging transform SP, compare HASHBYTES('SHA2_256', CONCAT(col1, '|', col2, ...))
-- against stored hash to detect changes without a ModifiedDate

-- Option C: SQL Server Change Tracking (lightweight, no history)
DECLARE @last_sync_version BIGINT = (SELECT LastSyncVersion FROM ELT_ControlTable WHERE SourceTable = 'Customer');

SELECT c.*, ct.SYS_CHANGE_OPERATION
FROM CHANGETABLE(CHANGES dbo.Customer, @last_sync_version) AS ct
JOIN dbo.Customer c ON ct.CustomerID = c.CustomerID;

-- Option D: CDC (Change Data Capture) — captures all changes including deletes
-- See Layer 1c below for full CDC patterns
```

---

## Layer 1c: CDC (Change Data Capture) Source Pattern

Use CDC when a source table has **no reliable ModifiedDate**, or when you need to capture **deletes**
(which the date-window SP pattern cannot detect). CDC captures every INSERT/UPDATE/DELETE from the
SQL Server transaction log without touching the source application.

### Enabling CDC

```sql
-- Step 1: Enable CDC on the source database (requires sysadmin)
USE SourceDB;
EXEC sys.sp_cdc_enable_db;

-- Step 2: Enable CDC on each table you want to track
EXEC sys.sp_cdc_enable_table
    @source_schema       = 'dbo',
    @source_name         = 'SalesOrder',
    @role_name           = NULL,        -- NULL = sysadmin/db_owner access only
    @supports_net_changes = 1;          -- enables fn_cdc_get_net_changes_* function

-- Verify
SELECT name, is_cdc_enabled FROM sys.databases WHERE name = 'SourceDB';
SELECT source_schema, source_name, capture_instance FROM cdc.change_tables;
```

**Prerequisites:**
- Source database recovery model must be **FULL** or **BULK_LOGGED**
- SQL Agent must be running (CDC uses capture and cleanup Agent jobs: `cdc.<db>_capture` / `cdc.<db>_cleanup`)
- Default cleanup retention is **3 days** — increase if ELT can fail for several days:
  `EXEC sys.sp_cdc_change_job @job_type='cleanup', @retention=7200;` (minutes; 7200 = 5 days)
- Do not use CDC on tables with frequent DDL changes (column adds/renames require re-enabling)

### CDC Change Table Structure

CDC creates a shadow table `cdc.dbo_<TableName>_CT` with these metadata columns:

| Column | Meaning |
|---|---|
| `__$start_lsn` | Log Sequence Number at change time |
| `__$seqval` | Row sequence within transaction |
| `__$operation` | 1=Delete, 2=Insert, 3=Before-update, 4=After-update |
| `__$update_mask` | Bitmask of changed columns |

### LSN-Based Load Window (replaces date-based window for CDC sources)

```sql
-- Map last-loaded timestamp → starting LSN
DECLARE @FromLSN BINARY(10) = sys.fn_cdc_map_time_to_lsn(
    'smallest greater than or equal',
    @LastLoadTime   -- DATETIME from ELT_ControlTable.LastSuccessfulLoadEnd
);
DECLARE @ToLSN BINARY(10) = sys.fn_cdc_get_max_lsn();

-- Get all net changes in window (final state of each row)
SELECT *
FROM cdc.fn_cdc_get_net_changes_dbo_SalesOrder(@FromLSN, @ToLSN, 'all');
-- __$operation values in net changes: 1=Delete, 2=Insert, 5=Update (merged)

-- Get all individual changes in window (audit trail)
SELECT *
FROM cdc.fn_cdc_get_all_changes_dbo_SalesOrder(@FromLSN, @ToLSN, 'all')
ORDER BY __$start_lsn, __$seqval;
```

### Transform SP Pattern for CDC Sources

For CDC sources the transform SP handles deletes explicitly — the standard date-window SP pattern cannot:

```sql
CREATE PROCEDURE dbo.usp_Transform_Fact_SalesOrder_CDC
    @FromLSN    BINARY(10),
    @ToLSN      BINARY(10),
    @BatchID    INT
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;

    BEGIN TRANSACTION;
    BEGIN TRY
        -- 1. Process DELETEs (soft-delete or hard-delete depending on DW policy)
        UPDATE f
        SET f.IsDeleted = 1, f.ELT_BatchID = @BatchID
        FROM dbo.Fact_SalesOrder f
        WHERE f.SalesOrderID IN (
            SELECT SalesOrderID
            FROM SourceDB.cdc.fn_cdc_get_net_changes_dbo_SalesOrder(@FromLSN, @ToLSN, 'all')
            WHERE __$operation = 1  -- delete
        );

        -- 2. MERGE INSERTs and UPDATEs (net change = final state, __$operation 2 or 5)
        MERGE dbo.Fact_SalesOrder AS tgt
        USING (
            SELECT
                c.SalesOrderID,
                d.DateKey    AS OrderDateKey,
                cust.CustomerKey,
                c.Quantity,
                c.UnitPrice,
                c.Quantity * c.UnitPrice AS SalesAmount
            FROM SourceDB.cdc.fn_cdc_get_net_changes_dbo_SalesOrder(@FromLSN, @ToLSN, 'all') c
            JOIN dbo.Dim_Date     d    ON CAST(c.OrderDate AS DATE) = d.FullDate
            JOIN dbo.Dim_Customer cust ON c.CustomerID = cust.SourceCustomerID
                                      AND cust.IsCurrent = 1
            WHERE c.__$operation IN (2, 5)  -- Insert or Update
        ) AS src ON tgt.SalesOrderID = src.SalesOrderID

        WHEN MATCHED THEN UPDATE SET
            tgt.OrderDateKey   = src.OrderDateKey,
            tgt.Quantity       = src.Quantity,
            tgt.UnitPrice      = src.UnitPrice,
            tgt.SalesAmount    = src.SalesAmount,
            tgt.ELT_BatchID    = @BatchID

        WHEN NOT MATCHED BY TARGET THEN INSERT
            (SalesOrderID, OrderDateKey, CustomerKey, Quantity, UnitPrice, SalesAmount, ELT_BatchID)
        VALUES
            (src.SalesOrderID, src.OrderDateKey, src.CustomerKey,
             src.Quantity, src.UnitPrice, src.SalesAmount, @BatchID);

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
        THROW;
    END CATCH;
END;
GO
```

### ELT_ControlTable for CDC Sources

CDC sources store the last processed LSN (as a converted datetime) rather than a plain datetime:

```sql
-- For CDC sources, LastSuccessfulLoadEnd stores the high-water LSN as a datetime
-- Use sys.fn_cdc_map_time_to_lsn() to convert back to LSN for the next run
INSERT INTO dbo.ELT_ControlTable (SourceSystem, SourceTable, ChangeDetectionColumn, LastSuccessfulLoadEnd)
VALUES ('ERP_CDC', 'SalesOrder', 'CDC_LSN', '2000-01-01');

-- After successful load, advance watermark to time corresponding to @ToLSN
UPDATE dbo.ELT_ControlTable
SET LastSuccessfulLoadEnd = sys.fn_cdc_map_lsn_to_time(@ToLSN)
WHERE SourceSystem = 'ERP_CDC' AND SourceTable = 'SalesOrder';
```

### CDC vs Date-Window: When to Use Each

| Factor | Date-Window SP | CDC |
|---|---|---|
| Source has ModifiedDate | ✅ Use this | Overkill |
| Need to capture deletes | ❌ Cannot detect | ✅ Required |
| Source has no change date | ❌ Must full-reload | ✅ Preferred |
| Source is not SQL Server | ✅ Works cross-platform | ❌ SQL Server only |
| Source recovery model is SIMPLE | ✅ Works | ❌ Requires FULL/BULK_LOGGED |
| Very high-volume tables | ✅ Low overhead | Monitor log growth |

SQL Server Integration Services has **no native Salesforce connector**. The recommended solution
for Salesforce extraction in an on-premises SSIS + SQL Server DW environment is the
**KingswaySoft SSIS Integration Toolkit** (formerly *SSIS Integration Toolkit for Salesforce*,
now bundled in the unified product).

> Product page: https://www.kingswaysoft.com/products/ssis-integration-toolkit-for-salesforce
> BIML guide: https://www.kingswaysoft.com/blog/2018/12/21/Using-BIML-with-KingswaySoft-SSIS-Components

### KingswaySoft Components

| Component | Role |
|---|---|
| **Salesforce Connection Manager** | Stores Salesforce credentials/OAuth config; supports REST and SOAP APIs; TLS 1.2 / OAuth2 |
| **Salesforce Source** | SSIS data flow source; reads via SOQL query or object browser; supports Bulk API v1/v2 |
| **Salesforce Destination** | SSIS data flow destination; supports Create, Update, Upsert, Delete, HardDelete |

For DW ELT purposes only the **Source** component is used. The Destination component is
only needed for write-back scenarios (e.g., updating Salesforce with DW-generated keys).

### Incremental Refresh via SOQL Date Filter

Salesforce objects expose `LastModifiedDate` and `CreatedDate` system fields on every standard
and custom object. Use these in a parameterized SOQL query to replicate the `@StartDate/@EndDate`
pattern used for SQL source SPs:

```
-- SOQL query in the KingswaySoft Source component (parameterized via SSIS expression)
SELECT Id, Name, AccountNumber, BillingCity, BillingState, BillingPostalCode,
       Industry, AnnualRevenue, NumberOfEmployees, OwnerId, IsDeleted,
       CreatedDate, LastModifiedDate
FROM Account
WHERE LastModifiedDate >= @[User::LoadWindowStart]
  AND LastModifiedDate <  @[User::LoadWindowEnd]
```

In the SSIS package, set the SOQL query as an expression on the component using the same
`LoadWindowStart` / `LoadWindowEnd` package variables passed from `Master_Orchestrator.dtsx`.

### SSIS Data Flow: Salesforce → Staging (ELT Pattern)

Following the ELT principle — **no transforms in the data flow**:

```
[Salesforce Source]     SOQL incremental query, Bulk API v2 for large objects
       │
       │  (raw columns, Salesforce data types as-is)
       ▼
[OLE DB Destination]    staging.stg_SF_<ObjectName>, fast load, no constraint checks
```

- Do **not** add Derived Column, Lookup, Data Conversion, or Conditional Split transforms
- All type coercion and business logic happen in `usp_Transform_Dim_*` / `usp_Transform_Fact_*`
- Enable **Bulk API v2** in the Source component for objects with > 50,000 rows

### Staging Table Naming for Salesforce

```sql
-- Salesforce prefix: SF
staging.stg_SF_Account
staging.stg_SF_Contact
staging.stg_SF_Opportunity
staging.stg_SF_OpportunityLineItem
staging.stg_SF_Case
staging.stg_SF_User
```

All staging tables follow the same conventions as SQL-sourced staging (all nullable, heap,
no FKs) — see Layer 2. Salesforce field names map directly to column names without renaming.

### BIML Code Generation for Salesforce Packages

When loading multiple Salesforce objects with the same pattern (incremental source → staging),
use **BIML + BIMLScript** to generate packages instead of building each one manually.

**Tools:**
- **BIMLExpress** — free Visual Studio add-in; sufficient for most generation tasks
- **BIMLStudio** — commercial IDE with full BIML features including import/export
- **BIMLOnline** — web-based IDE; useful for reverse-engineering existing packages to BIML

**Recommended workflow:**
1. Build **one** Salesforce → Staging package manually in Visual Studio using the KingswaySoft designer
2. In BIMLExpress, right-click the SSIS project → **Convert SSIS Packages to BIML** to reverse-engineer it
3. Use the reverse-engineered BIML as your template — the exact KingswaySoft XML attribute syntax is captured
4. Wrap the template with BIMLScript (C# loops) to generate all remaining object packages from a metadata table
5. Right-click the `.biml` file → **Generate SSIS Packages** to produce the `.dtsx` files

> **Key tip**: The reverse-engineer step (step 2) is the most reliable way to get the exact XML
> attribute spelling and casing for KingswaySoft components. Do not hand-write these from scratch.

### Critical BIML Gotchas for KingswaySoft Components

These issues will cause package generation to fail silently or at runtime:

#### 1. Remove `System.` prefix from autogenerated data types

BIMLExpress / BIMLScript may autogenerate column data type attributes with a `System.` prefix:

```xml
<!-- ❌ FAILS — "System." prefix is not valid in BIML data type attribute -->
<Column Name="NumberOfEmployees" DataType="System.Int32" />

<!-- ✅ CORRECT — remove "System." -->
<Column Name="NumberOfEmployees" DataType="Int32" />
```

This applies to all .NET primitive type names: `System.String` → `String`,
`System.Double` → `Double`, `System.Decimal` → `Decimal`, `System.Boolean` → `Boolean`,
`System.DateTime` → `DateTime`, etc.

#### 2. Include `UsesDispositions='true'` on KingswaySoft source components

KingswaySoft source components use **error output rows** (dispositions) to handle partial failures,
API errors, and Salesforce governor limit responses. Without this attribute, BIML will generate a
package that cannot properly handle component error outputs:

```xml
<!-- ❌ MISSING — package will fail at runtime on any non-success row -->
<CustomComponent Name="SalesforceSource_Account"
                 ComponentClassID="{...KingswaySoft CLSID...}">

<!-- ✅ CORRECT — always include UsesDispositions='true' -->
<CustomComponent Name="SalesforceSource_Account"
                 ComponentClassID="{...KingswaySoft CLSID...}"
                 UsesDispositions="true">
```

> The exact `ComponentClassID` GUIDs for each KingswaySoft component are captured when
> you reverse-engineer an existing package. Always copy from the reverse-engineered output
> rather than looking them up manually.

### BIML Template Structure for Salesforce → Staging Packages

```xml
<Biml xmlns="http://schemas.varigence.com/biml.xsd">

  <!-- Connection Managers -->
  <Connections>
    <CustomConnection Name="SF_Conn"
                      ConnectionString="..."
                      CreateInProject="true" />
    <OleDbConnection Name="DW_Conn"
                     ConnectionString="..."
                     CreateInProject="true" />
  </Connections>

  <!-- Generate one package per Salesforce object from metadata table -->
  <#
    // BIMLScript: pull object list from metadata table
    var conn = new OleDbConnection("Data Source=...;...");
    conn.Open();
    var cmd = new OleDbCommand("SELECT ObjectName, StagingTable, SoqlQuery FROM dbo.BIML_SalesforceObjects", conn);
    var reader = cmd.ExecuteReader();
    var objects = new List<dynamic>();
    while (reader.Read()) {
        objects.Add(new { ObjectName = reader[0], StagingTable = reader[1], SoqlQuery = reader[2] });
    }
    conn.Close();
  #>

  <Packages>
    <# foreach (var obj in objects) { #>
    <Package Name="Load_SF_<#= obj.ObjectName #>"
             ConstraintMode="Linear"
             ProtectionLevel="DontSaveSensitive">

      <Variables>
        <Variable Name="LoadWindowStart" DataType="DateTime">1900-01-01</Variable>
        <Variable Name="LoadWindowEnd"   DataType="DateTime">9999-12-31</Variable>
      </Variables>

      <Tasks>
        <Dataflow Name="DFT_SF_<#= obj.ObjectName #>_To_Staging">
          <Transformations>

            <!-- KingswaySoft Salesforce Source -->
            <CustomComponent Name="SRC_SF_<#= obj.ObjectName #>"
                             ComponentClassID="{KINGSWAYSOFT-SOURCE-CLSID}"
                             UsesDispositions="true"
                             ContactInfo="Salesforce Source;KingswaySoft Inc.">
              <CustomProperties>
                <CustomProperty Name="AccessMode" Value="1" />  <!-- 1 = SOQL query -->
                <CustomProperty Name="SqlCommand"
                                Value="<#= obj.SoqlQuery #>" />
                <CustomProperty Name="UseBulkApi" Value="true" />
              </CustomProperties>
              <Connections>
                <Connection Name="SalesforceConnection"
                            ConnectionName="SF_Conn" />
              </Connections>
              <!-- Output columns: DataType must NOT have "System." prefix -->
              <OutputBuffer Name="Output">
                <!-- Columns defined per object — generated from Salesforce metadata -->
              </OutputBuffer>
            </CustomComponent>

            <!-- OLE DB Destination to staging -->
            <OleDbDestination Name="DST_Staging_<#= obj.ObjectName #>"
                              ConnectionName="DW_Conn"
                              KeepIdentity="false"
                              TableLock="true"
                              CheckConstraints="false"
                              RowsPerBatch="10000">
              <ExternalTableOutput Table="<#= obj.StagingTable #>" />
            </OleDbDestination>

          </Transformations>
        </Dataflow>
      </Tasks>
    </Package>
    <# } #>
  </Packages>

</Biml>
```

### Salesforce-Specific Staging Considerations

```sql
-- Salesforce IDs are 18-character case-sensitive strings
-- Always store as NCHAR(18) or NVARCHAR(18), never INT or BIGINT
CREATE TABLE staging.stg_SF_Account (
    Id                  NCHAR(18)       NULL,   -- Salesforce 18-char ID
    Name                NVARCHAR(255)   NULL,
    AccountNumber       NVARCHAR(40)    NULL,
    BillingCity         NVARCHAR(40)    NULL,
    BillingState        NVARCHAR(80)    NULL,
    BillingPostalCode   NVARCHAR(20)    NULL,
    Industry            NVARCHAR(255)   NULL,   -- Picklist → store as text
    AnnualRevenue       DECIMAL(18,2)   NULL,
    NumberOfEmployees   INT             NULL,
    OwnerId             NCHAR(18)       NULL,   -- FK to User object
    IsDeleted           BIT             NULL,   -- Soft delete flag
    CreatedDate         DATETIME        NULL,
    LastModifiedDate    DATETIME        NULL,

    -- ELT metadata
    stg_LoadedDateTime  DATETIME        NOT NULL DEFAULT GETDATE(),
    stg_SourceSystem    NVARCHAR(50)    NOT NULL DEFAULT 'Salesforce',
    stg_ETLBatchID      INT             NULL
);
```

**Salesforce data type mapping for staging:**

| Salesforce Type | SQL Server Staging Type |
|---|---|
| `ID` (18-char) | `NCHAR(18)` |
| `String` / `TextArea` | `NVARCHAR(n)` — use Salesforce field length |
| `Picklist` / `MultiPicklist` | `NVARCHAR(255)` — store label text, not ordinal |
| `Date` | `DATE` |
| `DateTime` | `DATETIME` |
| `Boolean` | `BIT` |
| `Integer` | `INT` |
| `Double` / `Currency` / `Percent` | `DECIMAL(18,6)` |
| `Reference` (lookup FK) | `NCHAR(18)` — resolves to related object ID |
| `Phone` / `Email` / `URL` | `NVARCHAR(255)` |

### ELT Control Table Entry for Salesforce Objects

```sql
-- Add Salesforce objects to ELT_ControlTable with their change detection column
INSERT INTO dbo.ELT_ControlTable (SourceSystem, SourceTable, ChangeDetectionColumn, LastSuccessfulLoadEnd)
VALUES
    ('Salesforce', 'Account',             'LastModifiedDate', '2000-01-01'),
    ('Salesforce', 'Contact',             'LastModifiedDate', '2000-01-01'),
    ('Salesforce', 'Opportunity',         'LastModifiedDate', '2000-01-01'),
    ('Salesforce', 'OpportunityLineItem', 'LastModifiedDate', '2000-01-01'),
    ('Salesforce', 'Case',                'LastModifiedDate', '2000-01-01'),
    ('Salesforce', 'User',                'LastModifiedDate', '2000-01-01');
```

The same `usp_ELT_GetLoadWindow` / `usp_ELT_CompleteLoadWindow` pattern used for SQL sources
applies — the Salesforce SSIS packages receive `LoadWindowStart` and `LoadWindowEnd` as package
variables from `Master_Orchestrator.dtsx` and the high-water mark advances only on full success.

---

## Layer 2: Staging Tables — Raw Copy Zone

### Design Principles
- **Column names match source exactly** — no renaming in staging
- **All columns are nullable** — staging is a raw landing zone, not enforced
- **No foreign keys** — staging tables have no FK constraints
- **No identity columns** — use source PKs as-is
- **No triggers** — staging must be fast
- **Truncate before each load** — staging is not historical; it holds the current window only
- **Schema**: use a dedicated `staging` schema (not `dbo`)

### Naming Convention

```
staging.stg_<SourceSystemPrefix>_<SourceTableName>

Examples:
  staging.stg_CRM_Customer
  staging.stg_ERP_SalesOrder
  staging.stg_ERP_SalesOrderLine
  staging.stg_HR_Employee
```

### Staging Table DDL Pattern

```sql
CREATE TABLE staging.stg_CRM_Customer (
    -- Source columns (exact names, all nullable)
    CustomerID      INT             NULL,
    FirstName       NVARCHAR(100)   NULL,
    LastName        NVARCHAR(100)   NULL,
    Email           NVARCHAR(255)   NULL,
    Phone           NVARCHAR(50)    NULL,
    AddressLine1    NVARCHAR(200)   NULL,
    AddressLine2    NVARCHAR(200)   NULL,
    City            NVARCHAR(100)   NULL,
    StateProvince   NVARCHAR(100)   NULL,
    PostalCode      NVARCHAR(20)    NULL,
    CountryCode     NCHAR(2)        NULL,
    CustomerTypeCode NVARCHAR(20)   NULL,
    IsActive        BIT             NULL,
    CreatedDate     DATETIME        NULL,
    ModifiedDate    DATETIME        NULL,

    -- ELT metadata columns (added by SSIS/load process)
    stg_LoadedDateTime  DATETIME    NOT NULL DEFAULT GETDATE(),
    stg_SourceSystem    NVARCHAR(50) NOT NULL DEFAULT 'CRM',
    stg_ETLBatchID      INT         NULL    -- FK to ELT_BatchLog
);
GO

-- Heap table (no clustered index) — staging is write-once, read-once
-- If staging table is queried for transform joins, add a nonclustered index on the source PK
CREATE NONCLUSTERED INDEX IX_stg_CRM_Customer_CustomerID
    ON staging.stg_CRM_Customer (CustomerID);
```

---

## Layer 3: SSIS Package Architecture

### Four-Package Structure

The ELT pipeline is implemented as **four SSIS packages** run in sequence by a single orchestrator.
Each child package runs its tasks **in parallel** within its stage.

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Master_Orchestrator.dtsx   (runs child packages in SEQUENCE)            │
│                                                                          │
│   Step 1 ──► Load_Staging.dtsx      (all extracts run in PARALLEL)       │
│                  │                                                        │
│   Step 2 ──► Load_Dimensions.dtsx   (all dim transforms in PARALLEL)     │
│  (only after         │                                                    │
│   Step 1 succeeds)   │                                                    │
│   Step 3 ──► Load_Facts.dtsx        (all fact transforms in PARALLEL)    │
│  (only after                                                              │
│   Step 2 succeeds)                                                        │
└──────────────────────────────────────────────────────────────────────────┘
```

**Why this structure:**
- **Sequential stages** enforce correctness: facts can't load before dimensions are current
- **Parallel tasks within each stage** maximise throughput — all staging extracts run simultaneously, all dimension SPs run simultaneously
- **Single orchestrator** is the only entry point for the SQL Agent job — one job step, one failure point to monitor
- **Child packages are independently runnable** for debugging or re-running a single stage

---

### Package 1: `Master_Orchestrator.dtsx` (also called `Package.dtsx`)

> **Naming**: The reference projects use `Package.dtsx` as the orchestrator name (the SSIS default).
> Both `Package.dtsx` and `Master_Orchestrator.dtsx` are acceptable names — the key is that
> the SQL Agent job calls **only this one package**.

Runs the three child packages in strict sequence. Uses **Execute Package Task** for each child.
Captures overall job status and advances the high-water mark only on full success.

```
Control Flow — Master_Orchestrator.dtsx
────────────────────────────────────────────────────────────────
[Execute SQL]  Log_JobStart
    → INSERT INTO dbo.ELT_BatchLog (PackageName='Master', Status='Running', ...)
    → Sets User::MasterBatchID

[Execute SQL]  Get_LoadWindow
    → EXEC dbo.usp_ELT_GetLoadWindow
    → Sets User::LoadWindowStart, User::LoadWindowEnd

[Execute Package]  Run_Load_Staging          ← Package: Load_Staging.dtsx
    → Pass variables: LoadWindowStart, LoadWindowEnd, MasterBatchID
    → On failure: jump to OnError handler

[Execute Package]  Run_Load_Dimensions       ← Package: Load_Dimensions.dtsx
    → Runs only if Run_Load_Staging succeeded (precedence constraint: Success)
    → Pass variables: MasterBatchID

[Execute Package]  Run_Load_Facts            ← Package: Load_Facts.dtsx
    → Runs only if Run_Load_Dimensions succeeded (precedence constraint: Success)
    → Pass variables: MasterBatchID

[Execute SQL]  Advance_HighWaterMark
    → EXEC dbo.usp_ELT_CompleteLoadWindow
    → Runs only if ALL three child packages succeeded

[Execute SQL]  Log_JobSuccess
    → UPDATE dbo.ELT_BatchLog SET Status='Success', JobEndTime=GETDATE()

── OnError event handler ──────────────────────────────────────
[Execute SQL]  Log_JobFailure
    → UPDATE dbo.ELT_BatchLog SET Status='Failed', ErrorMessage=@[System::ErrorDescription]
[Send Mail]    Alert_On_Failure
    → To: dw-alerts@yourorg.com
    → Subject: DW Load FAILED — @[System::PackageName] — @[System::StartTime]
```

**Key configuration:**
- `DelayValidation = True` on all child Execute Package Tasks (child packages may not be deployed at design time)
- Parent package variables (`LoadWindowStart`, `LoadWindowEnd`, `MasterBatchID`) passed as package configurations or parent variable mappings
- Set `MaxConcurrentExecutables = 1` at the master level — children run sequentially at this level

---

### Package 2: `Load_Staging.dtsx`

Extracts all source tables to staging in **parallel**. Each extract is a self-contained sequence container.
Tasks run simultaneously — use `MaxConcurrentExecutables = -1` (unlimited) or set a limit based on source DB capacity.

```
Control Flow — Load_Staging.dtsx
────────────────────────────────────────────────────────────────
Variables: LoadWindowStart (DateTime), LoadWindowEnd (DateTime), MasterBatchID (Int32)

All containers run IN PARALLEL (no precedence constraints between them):

┌─────────────────────────────┐  ┌─────────────────────────────┐  ┌─────────────────────────────┐
│ SEQ: Extract_Customer        │  │ SEQ: Extract_SalesOrder      │  │ SEQ: Extract_Product         │
│  1. Truncate stg_CRM_Customer│  │  1. Truncate stg_ERP_Sales.. │  │  1. Truncate stg_ERP_Product  │
│  2. DFT: Extract → Staging   │  │  2. DFT: Extract → Staging   │  │  2. DFT: Extract → Staging   │
│  3. Log rows extracted       │  │  3. Log rows extracted       │  │  3. Log rows extracted       │
└─────────────────────────────┘  └─────────────────────────────┘  └─────────────────────────────┘
         (add one SEQ per source table — all run simultaneously)
```

**Each Sequence Container contains:**
```
[Execute SQL]  Truncate_StagingTable
    SQL: TRUNCATE TABLE staging.stg_CRM_Customer
    (runs before extract — ensures staging is clean for this window)

[Data Flow]    Extract_To_Staging
    Source:      OLE DB Source (Source DB connection)
                 Access mode: SQL Command
                 SQL: EXEC dbo.usp_Extract_Customer ?, ?
                 Parameter mapping: 0=LoadWindowStart, 1=LoadWindowEnd
    Destination: OLE DB Destination (DW connection)
                 Table: staging.stg_CRM_Customer
                 Fast Load: ✓  Keep NULLs: ✓
                 Rows per batch: 10000  Max insert commit: 100000
    (NO transforms in the data flow — source to destination only)

[Execute SQL]  Log_Rows_Extracted
    SQL: UPDATE dbo.ELT_BatchLog
         SET RowsExtracted = ROWCOUNT_BIG()
         WHERE BatchID = ? AND SourceTable = 'stg_CRM_Customer'
```

**Parallelism control:**
- Set `MaxConcurrentExecutables = -1` in package properties to let SSIS decide
- Or cap at the number of source DB connections your source system can sustain without performance impact
- Each sequence container has its own connection managers — they do not share connections

---

### Package 3: `Load_Dimensions.dtsx`

Executes all dimension transform stored procedures **in parallel**.
Each T-SQL SP is a single Execute SQL Task — no data flows in this package.

```
Control Flow — Load_Dimensions.dtsx
────────────────────────────────────────────────────────────────
Variables: MasterBatchID (Int32)

All tasks run IN PARALLEL (no precedence constraints between them):

[Execute SQL]  Transform_Dim_Date          → EXEC dbo.usp_Transform_Dim_Date
[Execute SQL]  Transform_Dim_Customer      → EXEC dbo.usp_Transform_Dim_Customer
[Execute SQL]  Transform_Dim_Product       → EXEC dbo.usp_Transform_Dim_Product
[Execute SQL]  Transform_Dim_Employee      → EXEC dbo.usp_Transform_Dim_Employee
[Execute SQL]  Transform_Dim_Geography     → EXEC dbo.usp_Transform_Dim_Geography
         (add one Execute SQL Task per dimension — all run simultaneously)
```

**Notes:**
- If Dimension B depends on Dimension A (e.g., `Dim_SubCategory` needs `Dim_Category` loaded first), add a **precedence constraint** between those two specific tasks only — do not serialize the entire package
- Each Execute SQL Task connects to the DW database
- Connection timeout should be generous (e.g., 300s) — SCD Type 2 MERGEs can be long-running
- `CommandTimeout = 0` (no timeout) for large dimension loads

---

### Package 4: `Load_Facts.dtsx`

Executes all fact table transform stored procedures **in parallel**.
Runs only after `Load_Dimensions.dtsx` has completed successfully.

```
Control Flow — Load_Facts.dtsx
────────────────────────────────────────────────────────────────
Variables: MasterBatchID (Int32)

All tasks run IN PARALLEL (no precedence constraints between them):

[Execute SQL]  Transform_Fact_SalesOrder       → EXEC dbo.usp_Transform_Fact_SalesOrder
[Execute SQL]  Transform_Fact_Inventory        → EXEC dbo.usp_Transform_Fact_Inventory
[Execute SQL]  Transform_Fact_CustomerContact  → EXEC dbo.usp_Transform_Fact_CustomerContact
         (add one Execute SQL Task per fact table — all run simultaneously)
```

**Notes:**
- Fact transforms can run in parallel because they write to different fact tables
- Fact transforms read from staging (not from other fact tables) — no cross-fact dependencies
- If a fact requires an aggregate from another fact first, that dependency is exceptional and must be documented + sequenced explicitly

---

### SSIS Data Flow: Extract Only (within Load_Staging)

```
OLE DB Source (Source DB):
  Connection:   CRM_SourceDB connection manager
  Access mode:  SQL Command
  SQL Command:  EXEC dbo.usp_Extract_Customer ?, ?
  Parameters:   Parameter 0 → User::LoadWindowStart (DateTime)
                Parameter 1 → User::LoadWindowEnd   (DateTime)

  ↓  (no transforms, no lookups, no derived columns)

OLE DB Destination (DW):
  Connection:   DW_Database connection manager
  Table:        staging.stg_CRM_Customer
  Access mode:  Table or view — fast load
  Keep nulls:   ✓
  Rows per batch:         10,000
  Max insert commit size: 100,000
```

---

### Package Deployment & Folder Structure

```
SSISDB Catalog (or file system):
  /ELT/
    Master_Orchestrator.dtsx    ← SQL Agent job calls THIS package only
    Load_Staging.dtsx
    Load_Dimensions.dtsx
    Load_Facts.dtsx

  /ELT/Connections/             ← Shared connection managers (if using project deployment model)
```

**Project deployment model** (recommended over package deployment):
- All four packages share connection managers defined at the project level
- Environment variables in SSISDB store connection strings (no hardcoded server names)
- Deploy all four packages together as a single SSIS project (`.ispac` file)

---

## Layer 4: ELT Control Table — Orchestration

### ETL Control Table Design

```sql
-- Main control table: tracks high-water mark per source table
CREATE TABLE dbo.ELT_ControlTable (
    ControlID           INT IDENTITY(1,1) PRIMARY KEY,
    SourceSystem        NVARCHAR(100)   NOT NULL,
    SourceTable         NVARCHAR(200)   NOT NULL,
    ChangeDetectionCol  NVARCHAR(100)   NOT NULL,   -- e.g., 'ModifiedDate'
    LastSuccessfulLoad  DATETIME        NOT NULL DEFAULT '1900-01-01',
    WindowSizeHours     INT             NOT NULL DEFAULT 24,
    IsActive            BIT             NOT NULL DEFAULT 1,
    Notes               NVARCHAR(500)   NULL,

    CONSTRAINT UQ_ELT_Control UNIQUE (SourceSystem, SourceTable)
);
GO

-- Batch log: one row per load attempt
CREATE TABLE dbo.ELT_BatchLog (
    BatchID         INT IDENTITY(1,1) PRIMARY KEY,
    SourceSystem    NVARCHAR(100)   NOT NULL,
    SourceTable     NVARCHAR(200)   NOT NULL,
    LoadWindowStart DATETIME        NOT NULL,
    LoadWindowEnd   DATETIME        NOT NULL,
    JobStartTime    DATETIME        NOT NULL DEFAULT GETDATE(),
    JobEndTime      DATETIME        NULL,
    Status          NVARCHAR(20)    NOT NULL DEFAULT 'Running',   -- Running, Success, Failed
    RowsExtracted   INT             NULL,
    RowsInserted    INT             NULL,
    RowsUpdated     INT             NULL,
    RowsRejected    INT             NULL,
    ErrorMessage    NVARCHAR(MAX)   NULL,
    LoadedBy        NVARCHAR(200)   NOT NULL DEFAULT SYSTEM_USER
);
```

### Getting and Advancing the Date Window

```sql
-- SP: Get the current load window for a source table
CREATE PROCEDURE dbo.usp_ELT_GetLoadWindow
    @SourceSystem   NVARCHAR(100),
    @SourceTable    NVARCHAR(200),
    @StartDate      DATETIME OUTPUT,
    @EndDate        DATETIME OUTPUT,
    @BatchID        INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    SELECT
        @StartDate = LastSuccessfulLoad,
        @EndDate   = DATEADD(HOUR, WindowSizeHours, LastSuccessfulLoad)
    FROM dbo.ELT_ControlTable
    WHERE SourceSystem = @SourceSystem AND SourceTable = @SourceTable;

    -- Cap EndDate at current time (don't look into the future)
    IF @EndDate > GETDATE()
        SET @EndDate = GETDATE();

    -- Log the batch start
    INSERT INTO dbo.ELT_BatchLog (SourceSystem, SourceTable, LoadWindowStart, LoadWindowEnd)
    VALUES (@SourceSystem, @SourceTable, @StartDate, @EndDate);

    SET @BatchID = SCOPE_IDENTITY();
END;
GO

-- SP: Mark the load window as successfully completed
CREATE PROCEDURE dbo.usp_ELT_CompleteLoadWindow
    @BatchID        INT,
    @SourceSystem   NVARCHAR(100),
    @SourceTable    NVARCHAR(200),
    @RowsExtracted  INT,
    @RowsInserted   INT,
    @RowsUpdated    INT
AS
BEGIN
    SET NOCOUNT ON;

    -- Advance the high-water mark
    UPDATE dbo.ELT_ControlTable
    SET LastSuccessfulLoad = (SELECT LoadWindowEnd FROM dbo.ELT_BatchLog WHERE BatchID = @BatchID)
    WHERE SourceSystem = @SourceSystem AND SourceTable = @SourceTable;

    -- Update batch log
    UPDATE dbo.ELT_BatchLog
    SET Status          = 'Success',
        JobEndTime      = GETDATE(),
        RowsExtracted   = @RowsExtracted,
        RowsInserted    = @RowsInserted,
        RowsUpdated     = @RowsUpdated
    WHERE BatchID = @BatchID;
END;
```

---

## Layer 5: DW Transformation — T-SQL Stored Procedures

All business logic, data type conversions, SCD handling, and DW table population is done here.

### Naming Convention for Transform SPs

```
dbo.usp_Transform_<ObjectType>_<ObjectName>

Examples:
  dbo.usp_Transform_Dim_Customer        -- loads/updates Dim_Customer from staging
  dbo.usp_Transform_Dim_Date            -- populates Dim_Date (one-time + extensions)
  dbo.usp_Transform_Fact_SalesOrder     -- loads Fact_SalesOrder from staging
  dbo.usp_Transform_Bridge_ClaimDx      -- loads Bridge_ClaimDiagnosis
```

### Transform SP — Dimension Load (SCD Type 2 MERGE)

```sql
CREATE PROCEDURE dbo.usp_Transform_Dim_Customer
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        BEGIN TRANSACTION;

        -- Step 1: Ensure Unknown member exists
        IF NOT EXISTS (SELECT 1 FROM dbo.Dim_Customer WHERE CustomerKey = -1)
            INSERT INTO dbo.Dim_Customer (CustomerKey, CustomerID, FirstName, LastName,
                CustomerTypeCode, IsCurrent, RowEffectiveDate, RowExpirationDate,
                RowCreatedDate, RowUpdatedDate)
            VALUES (-1, -1, 'Unknown', 'Unknown', 'Unknown', 1, '1900-01-01', NULL, GETDATE(), GETDATE());

        -- Step 2: Expire changed rows (SCD Type 2)
        UPDATE d
        SET d.IsCurrent         = 0,
            d.RowExpirationDate = CAST(GETDATE() AS DATE),
            d.RowUpdatedDate    = GETDATE()
        FROM dbo.Dim_Customer d
        JOIN staging.stg_CRM_Customer s ON d.CustomerID = s.CustomerID
        WHERE d.IsCurrent = 1
          AND (   d.FirstName          <> s.FirstName
               OR d.LastName           <> s.LastName
               OR d.Email              <> ISNULL(s.Email, '')
               OR d.CustomerTypeCode   <> ISNULL(s.CustomerTypeCode, 'Unknown')
               -- Add more Type 2 tracked columns here
              );

        -- Step 3: Insert new rows (new customers + new versions of changed customers)
        INSERT INTO dbo.Dim_Customer (
            CustomerID, FirstName, LastName, Email, Phone,
            AddressLine1, AddressLine2, City, StateProvince, PostalCode, CountryCode,
            CustomerTypeCode, IsActive,
            IsCurrent, RowEffectiveDate, RowExpirationDate, RowCreatedDate, RowUpdatedDate
        )
        SELECT
            s.CustomerID,
            ISNULL(s.FirstName, 'Unknown'),
            ISNULL(s.LastName, 'Unknown'),
            s.Email,
            s.Phone,
            s.AddressLine1, s.AddressLine2, s.City, s.StateProvince, s.PostalCode, s.CountryCode,
            ISNULL(s.CustomerTypeCode, 'Unknown'),
            ISNULL(s.IsActive, 0),
            1,                          -- IsCurrent
            CAST(GETDATE() AS DATE),    -- RowEffectiveDate
            NULL,                       -- RowExpirationDate (active)
            GETDATE(),                  -- RowCreatedDate
            GETDATE()                   -- RowUpdatedDate
        FROM staging.stg_CRM_Customer s
        WHERE NOT EXISTS (
            SELECT 1 FROM dbo.Dim_Customer d
            WHERE d.CustomerID = s.CustomerID AND d.IsCurrent = 1
        );

        -- Step 4: Type 1 overwrites (non-historical attributes — e.g., phone, address)
        UPDATE d
        SET d.Phone         = s.Phone,
            d.AddressLine1  = s.AddressLine1,
            d.AddressLine2  = s.AddressLine2,
            d.City          = s.City,
            d.StateProvince = s.StateProvince,
            d.PostalCode    = s.PostalCode,
            d.RowUpdatedDate = GETDATE()
        FROM dbo.Dim_Customer d
        JOIN staging.stg_CRM_Customer s ON d.CustomerID = s.CustomerID
        WHERE d.IsCurrent = 1;

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
        THROW;
    END CATCH;
END;
GO
```

### Transform SP — Fact Table Load

```sql
CREATE PROCEDURE dbo.usp_Transform_Fact_SalesOrder
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        BEGIN TRANSACTION;

        -- Idempotency: delete rows for this load window before reinserting
        -- This makes the SP safe to re-run
        DELETE f
        FROM dbo.Fact_SalesOrder f
        WHERE EXISTS (
            SELECT 1 FROM staging.stg_ERP_SalesOrder s
            WHERE s.OrderID = f.OrderID  -- degenerate dimension match
        );

        -- Insert with surrogate key lookups
        INSERT INTO dbo.Fact_SalesOrder (
            OrderDateKey,
            CustomerKey,
            ProductKey,
            OrderID,            -- degenerate dimension
            Quantity,
            UnitPrice,
            DiscountAmount,
            ExtendedAmount,
            RowCreatedDate,
            RowUpdatedDate
        )
        SELECT
            -- Date key lookup
            ISNULL(d.DateKey, -1),

            -- Customer surrogate key lookup (current version)
            ISNULL(c.CustomerKey, -1),

            -- Product surrogate key lookup
            ISNULL(p.ProductKey, -1),

            s.OrderID,
            s.Quantity,
            s.UnitPrice,
            ISNULL(s.DiscountAmount, 0),
            s.Quantity * s.UnitPrice - ISNULL(s.DiscountAmount, 0),
            GETDATE(),
            GETDATE()
        FROM staging.stg_ERP_SalesOrder s
        -- Date dimension lookup
        LEFT JOIN dbo.Dim_Date d
            ON d.DateKey = CONVERT(INT, CONVERT(NVARCHAR(8), s.OrderDate, 112))
        -- Customer dimension: match to current version
        LEFT JOIN dbo.Dim_Customer c
            ON c.CustomerID = s.CustomerID AND c.IsCurrent = 1
        -- Product dimension
        LEFT JOIN dbo.Dim_Product p
            ON p.ProductID = s.ProductID AND p.IsCurrent = 1;

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
        THROW;
    END CATCH;
END;
GO
```

---

## Layer 6: SQL Server Agent Job Orchestration

### Single Job Step — One Entry Point

The SQL Agent job has **a single step** that calls `Master_Orchestrator.dtsx`.
The orchestrator handles all sequencing, error handling, logging, and high-water mark advancement internally.

```
SQL Server Agent Job: DW_Daily_Load
─────────────────────────────────────────────────────────────────
Schedule: Daily at 02:00 (or as required)

Step 1 (only step): Run_ELT_Orchestrator
  Type:          SQL Server Integration Services Package
  Package:       /SSISDB/ELT/Master_Orchestrator.dtsx
                 (or file path if using package deployment model)
  On Success:    Quit the job reporting success
  On Failure:    Quit the job reporting failure

  (All staging, dimension, fact loading, and SSAS processing
   happens INSIDE the orchestrator and child packages)
─────────────────────────────────────────────────────────────────
```

**Why a single job step:**
- Failure of any child package propagates up to the orchestrator, which fails the job
- The SQL Agent email alert fires once on the overall job failure — not per-package
- Re-running the job after a fix re-runs the full sequence from the beginning (idempotent)
- No need to manage SQL Agent step dependencies — SSIS precedence constraints handle all logic

### What Happens Inside the Orchestrator (Sequence)

```
Master_Orchestrator.dtsx
  │
  ├─ Log job start + get load window
  │
  ├─► Load_Staging.dtsx          ← all source extracts run in PARALLEL
  │         Customer extract  ──┐
  │         SalesOrder extract ─┤  (simultaneous)
  │         Product extract   ──┘
  │                              ↓ (only if ALL parallel tasks succeeded)
  ├─► Load_Dimensions.dtsx       ← all dimension SPs run in PARALLEL
  │         Dim_Date          ──┐
  │         Dim_Customer      ─┤  (simultaneous)
  │         Dim_Product       ──┘
  │                              ↓ (only if ALL parallel tasks succeeded)
  ├─► Load_Facts.dtsx            ← all fact SPs run in PARALLEL
  │         Fact_SalesOrder   ──┐
  │         Fact_Inventory    ─┤  (simultaneous)
  │         Fact_Contact      ──┘
  │                              ↓ (only if ALL parallel tasks succeeded)
  ├─ Advance high-water mark
  └─ Log job success / send failure alert on any error
```

### SSAS Processing — After Facts Load

SSAS processing is triggered from within `Master_Orchestrator.dtsx` after `Load_Facts.dtsx` succeeds,
using an Execute SQL Task with an XMLA command via a linked server or Analysis Services connection:

```xml
<!-- XMLA command executed from SSIS Execute SQL Task or SSAS Execute DDL Task -->
<Batch xmlns="http://schemas.microsoft.com/analysisservices/2003/engine">
  <Parallel>
    <!-- Process all dimensions first -->
    <Process xmlns:xsd="http://www.w3.org/2001/XMLSchema">
      <Object><DatabaseID>DW_Tabular</DatabaseID><DimensionID>Dim_Customer</DimensionID></Object>
      <Type>ProcessFull</Type>
    </Process>
    <Process xmlns:xsd="http://www.w3.org/2001/XMLSchema">
      <Object><DatabaseID>DW_Tabular</DatabaseID><DimensionID>Dim_Product</DimensionID></Object>
      <Type>ProcessFull</Type>
    </Process>
  </Parallel>
  <!-- Then process facts (separate step — after dimensions complete) -->
  <Process xmlns:xsd="http://www.w3.org/2001/XMLSchema">
    <Object><DatabaseID>DW_Tabular</DatabaseID><MeasureGroupID>Fact_SalesOrder</MeasureGroupID></Object>
    <Type>ProcessData</Type>
  </Process>
</Batch>
```

### Critical Rule: Dimensions Before Facts in SSAS Processing

```
SSAS Tabular processing MUST follow this order:
  1. Process all Dimension tables (full or incremental) — can run in parallel
  2. Process all Fact tables (partition-based incremental) — can run in parallel
  3. ProcessRecalc (if deferred recalc is used)

Never process a fact table before its dimension tables —
SSAS will report referential integrity violations and the fact table will fail to process.
```

### Failure Recovery Pattern

```sql
-- When a job fails midway, the ELT_ControlTable high-water mark is NOT advanced
-- Re-running the job will re-extract the same window and re-transform
-- This is why transform SPs must be idempotent (delete + re-insert or MERGE)

-- Check for failed/stuck batches
SELECT *
FROM dbo.ELT_BatchLog
WHERE Status = 'Running'
  AND JobStartTime < DATEADD(HOUR, -2, GETDATE());  -- running for more than 2 hours = stuck
```

---

## ELT Design Checklist

### Source Layer
- [ ] Source SPs accept `@StartDate` / `@EndDate` parameters
- [ ] Source SPs use `NOLOCK` or RCSI to avoid blocking production
- [ ] Source SPs are read-only — no DML
- [ ] Source SPs documented with `MS_Description` and `ExecutionContext` extended properties

### Staging Layer
- [ ] Staging tables match source column names exactly
- [ ] Staging tables are truncated before each load (inside `Load_Staging.dtsx`)
- [ ] Staging schema is separate from DW schema (`staging.*`)
- [ ] Staging tables have no FK constraints and no triggers

### SSIS Package Structure
- [ ] Four packages exist: `Master_Orchestrator`, `Load_Staging`, `Load_Dimensions`, `Load_Facts`
- [ ] `Master_Orchestrator` calls child packages in strict sequence (Staging → Dimensions → Facts)
- [ ] `Load_Staging` runs all extract tasks in parallel (no precedence constraints between extracts)
- [ ] `Load_Dimensions` runs all dimension transform SPs in parallel
- [ ] `Load_Facts` runs all fact transform SPs in parallel
- [ ] SSIS data flows contain NO transforms — raw extract to staging only
- [ ] SQL Agent job has a **single step** calling `Master_Orchestrator.dtsx` only
- [ ] Parent variables (`LoadWindowStart`, `LoadWindowEnd`, `MasterBatchID`) passed to child packages
- [ ] Project deployment model used (`.ispac`) with SSISDB environment variables for connection strings

### ELT Control & Audit
- [ ] ELT control table tracks high-water mark per source table
- [ ] ELT batch log records every load attempt with row counts and status
- [ ] High-water mark advances ONLY after full orchestrator success
- [ ] Job failure sends email alert via SSIS `OnError` event handler in orchestrator

### DW Transform Layer
- [ ] Transform SPs are idempotent (safe to re-run for same window)
- [ ] Transform SPs use explicit transactions with `TRY/CATCH`
- [ ] Unknown/default dimension members exist (key = -1) for all dimensions
- [ ] Fact load SPs use surrogate key lookups with `ISNULL(..., -1)` fallback
- [ ] All transform SPs documented with `MS_Description` extended property

### SSAS Processing
- [ ] SSAS processing triggered from orchestrator after `Load_Facts` succeeds
- [ ] Dimensions processed before facts (enforced by XMLA batch structure)
- [ ] SSAS processing failure fails the orchestrator job and sends alert

---

## References

- James Serra: [Difference between ETL and ELT](https://www.jamesserra.com/archive/2012/01/difference-between-etl-and-elt/)
- James Serra: [Starting Your First Data Warehouse — A Practical Learning Guide](https://www.jamesserra.com/archive/2025/11/starting-your-first-data-warehouse-a-practical-learning-guide/)
- Kimball Group: *The Data Warehouse Toolkit, 3rd Edition* (Chapter 22: ETL Subsystems)
- Microsoft: [SQL Server Change Data Capture](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/about-change-data-capture-sql-server)
