# Extended Properties Documentation Templates

SQL Server extended properties are the standard mechanism for inline database documentation.
They are queryable, appear in SSMS object tooltips, and survive schema deployments when managed correctly.

---

## Core T-SQL Functions

```sql
-- Add a new extended property
EXEC sys.sp_addextendedproperty
    @name  = N'<PropertyName>',
    @value = N'<PropertyValue>',
    @level0type = N'SCHEMA', @level0name = N'<schema>',
    @level1type = N'<ObjectType>', @level1name = N'<ObjectName>',
    [@level2type = N'<SubObjectType>', @level2name = N'<SubObjectName>']

-- Update an existing extended property
EXEC sys.sp_updateextendedproperty
    @name  = N'<PropertyName>',
    @value = N'<PropertyValue>',
    @level0type = N'SCHEMA', @level0name = N'<schema>',
    @level1type = N'<ObjectType>', @level1name = N'<ObjectName>',
    [@level2type = N'<SubObjectType>', @level2name = N'<SubObjectName>']

-- Remove an extended property
EXEC sys.sp_dropextendedproperty
    @name  = N'<PropertyName>',
    @level0type = N'SCHEMA', @level0name = N'<schema>',
    @level1type = N'<ObjectType>', @level1name = N'<ObjectName>',
    [@level2type = N'<SubObjectType>', @level2name = N'<SubObjectName>']

-- Query all extended properties
SELECT * FROM sys.extended_properties

-- Query for a specific object
SELECT
    OBJECT_SCHEMA_NAME(ep.major_id) AS SchemaName,
    OBJECT_NAME(ep.major_id) AS ObjectName,
    COALESCE(c.[name], '') AS ColumnName,
    ep.[name] AS PropertyName,
    CAST(ep.[value] AS NVARCHAR(MAX)) AS PropertyValue
FROM sys.extended_properties ep
LEFT JOIN sys.columns c ON ep.major_id = c.object_id AND ep.minor_id = c.column_id
WHERE ep.class = 1  -- object or column level
ORDER BY SchemaName, ObjectName, ColumnName, PropertyName
```

---

## Standard Property Names

Use consistent property names across all objects for queryability and tooling support.

| Property Name | Level | Description |
|---|---|---|
| `MS_Description` | Table, Column, View, SP, Function | Primary description — shown in SSMS tooltips. **Always set this first.** |
| `BusinessDescription` | Table, Column | Business-friendly explanation for non-technical users |
| `SourceSystem` | Table, Column | Where this data originates (e.g., `ERP - SAP`, `CRM - Salesforce`, `Flat File`) |
| `SourceTable` | Table | Source table name in the originating system |
| `SourceColumn` | Column | Source column name in the originating system |
| `Grain` | Fact Table | One sentence describing the grain (e.g., `One row per daily product-customer transaction`) |
| `TableType` | Table | `Fact`, `Dimension`, `Bridge`, `Staging`, `Reference`, `Archive` |
| `SCDType` | Dimension Table | SCD handling: `1`, `2`, `3`, `6`, `None` |
| `ConformedDimension` | Dimension Table | `Yes` / `No` — whether this is a shared conformed dimension |
| `KeyColumn` | Column | `SurrogateKey`, `NaturalKey`, `ForeignKey`, `Measure`, `Attribute`, `Flag` |
| `DataClassification` | Column | Sensitivity: `Public`, `Internal`, `Confidential`, `Restricted` |
| `PIIFlag` | Column | `Yes` / `No` — contains personally identifiable information |
| `IsActive` | Table | `Yes` / `No` — is this object actively used in production |
| `BusinessOwner` | Table | Team or person accountable for the data |
| `DataSteward` | Table | Person responsible for data quality |
| `RefreshFrequency` | Table | `Daily`, `Hourly`, `Weekly`, `Monthly`, `On-demand` |
| `CreatedDate` | Table | Date the object was first created |
| `LastReviewedDate` | Table | Date the documentation was last reviewed/verified |
| `RelatedReport` | Table, View | Power BI report or SSRS report that consumes this object |

---

## Templates by Object Type

### Table Documentation

```sql
-- ============================================================
-- Table: dbo.Fact_SalesTransaction
-- ============================================================
EXEC sys.sp_addextendedproperty
    @name  = N'MS_Description',
    @value = N'Sales transactions at daily product-customer-store grain. One row per line item on a sales order. Source: ERP order management system.',
    @level0type = N'SCHEMA', @level0name = N'dbo',
    @level1type = N'TABLE',  @level1name = N'Fact_SalesTransaction';

EXEC sys.sp_addextendedproperty
    @name  = N'Grain',
    @value = N'One row per sales order line item per day per product per customer per store',
    @level0type = N'SCHEMA', @level0name = N'dbo',
    @level1type = N'TABLE',  @level1name = N'Fact_SalesTransaction';

EXEC sys.sp_addextendedproperty
    @name  = N'TableType',
    @value = N'Fact',
    @level0type = N'SCHEMA', @level0name = N'dbo',
    @level1type = N'TABLE',  @level1name = N'Fact_SalesTransaction';

EXEC sys.sp_addextendedproperty
    @name  = N'SourceSystem',
    @value = N'ERP - OrderManagement',
    @level0type = N'SCHEMA', @level0name = N'dbo',
    @level1type = N'TABLE',  @level1name = N'Fact_SalesTransaction';

EXEC sys.sp_addextendedproperty
    @name  = N'RefreshFrequency',
    @value = N'Daily',
    @level0type = N'SCHEMA', @level0name = N'dbo',
    @level1type = N'TABLE',  @level1name = N'Fact_SalesTransaction';

EXEC sys.sp_addextendedproperty
    @name  = N'BusinessOwner',
    @value = N'Finance - Revenue Reporting Team',
    @level0type = N'SCHEMA', @level0name = N'dbo',
    @level1type = N'TABLE',  @level1name = N'Fact_SalesTransaction';
```

### Dimension Table Documentation

```sql
-- ============================================================
-- Table: dbo.Dim_Customer
-- ============================================================
EXEC sys.sp_addextendedproperty
    @name  = N'MS_Description',
    @value = N'Customer dimension. One row per customer per SCD Type 2 version. Tracks changes to customer name, address, and segment over time.',
    @level0type = N'SCHEMA', @level0name = N'dbo',
    @level1type = N'TABLE',  @level1name = N'Dim_Customer';

EXEC sys.sp_addextendedproperty
    @name  = N'TableType',
    @value = N'Dimension',
    @level0type = N'SCHEMA', @level0name = N'dbo',
    @level1type = N'TABLE',  @level1name = N'Dim_Customer';

EXEC sys.sp_addextendedproperty
    @name  = N'SCDType',
    @value = N'2',
    @level0type = N'SCHEMA', @level0name = N'dbo',
    @level1type = N'TABLE',  @level1name = N'Dim_Customer';

EXEC sys.sp_addextendedproperty
    @name  = N'ConformedDimension',
    @value = N'Yes',
    @level0type = N'SCHEMA', @level0name = N'dbo',
    @level1type = N'TABLE',  @level1name = N'Dim_Customer';
```

### Column Documentation

```sql
-- ============================================================
-- Column: dbo.Dim_Customer.CustomerKey
-- ============================================================
EXEC sys.sp_addextendedproperty
    @name  = N'MS_Description',
    @value = N'Surrogate key. System-generated integer. Used as primary key and foreign key in all fact tables. Do not expose to end users.',
    @level0type = N'SCHEMA', @level0name = N'dbo',
    @level1type = N'TABLE',  @level1name = N'Dim_Customer',
    @level2type = N'COLUMN', @level2name = N'CustomerKey';

EXEC sys.sp_addextendedproperty
    @name  = N'KeyColumn',
    @value = N'SurrogateKey',
    @level0type = N'SCHEMA', @level0name = N'dbo',
    @level1type = N'TABLE',  @level1name = N'Dim_Customer',
    @level2type = N'COLUMN', @level2name = N'CustomerKey';

-- ============================================================
-- Column: dbo.Dim_Customer.CustomerID
-- ============================================================
EXEC sys.sp_addextendedproperty
    @name  = N'MS_Description',
    @value = N'Natural key from source CRM system. Retained for traceability. Do not use as foreign key in fact tables.',
    @level0type = N'SCHEMA', @level0name = N'dbo',
    @level1type = N'TABLE',  @level1name = N'Dim_Customer',
    @level2type = N'COLUMN', @level2name = N'CustomerID';

EXEC sys.sp_addextendedproperty
    @name  = N'KeyColumn',
    @value = N'NaturalKey',
    @level0type = N'SCHEMA', @level0name = N'dbo',
    @level1type = N'TABLE',  @level1name = N'Dim_Customer',
    @level2type = N'COLUMN', @level2name = N'CustomerID';

EXEC sys.sp_addextendedproperty
    @name  = N'SourceSystem',
    @value = N'CRM - Dynamics 365',
    @level0type = N'SCHEMA', @level0name = N'dbo',
    @level1type = N'TABLE',  @level1name = N'Dim_Customer',
    @level2type = N'COLUMN', @level2name = N'CustomerID';

-- ============================================================
-- Column: dbo.Dim_Customer.IsCurrent (SCD Type 2 indicator)
-- ============================================================
EXEC sys.sp_addextendedproperty
    @name  = N'MS_Description',
    @value = N'SCD Type 2 currency indicator. 1 = this is the currently active version of this customer record. 0 = historical version.',
    @level0type = N'SCHEMA', @level0name = N'dbo',
    @level1type = N'TABLE',  @level1name = N'Dim_Customer',
    @level2type = N'COLUMN', @level2name = N'IsCurrent';
```

### View Documentation

```sql
-- ============================================================
-- View: report.vw_SalesSummary
-- ============================================================
EXEC sys.sp_addextendedproperty
    @name  = N'MS_Description',
    @value = N'Aggregated daily sales summary by product and customer. Pre-joined view optimized for report layer queries. Do not use in ETL processes.',
    @level0type = N'SCHEMA', @level0name = N'report',
    @level1type = N'VIEW',   @level1name = N'vw_SalesSummary';

EXEC sys.sp_addextendedproperty
    @name  = N'RelatedReport',
    @value = N'Power BI: Sales Performance Dashboard',
    @level0type = N'SCHEMA', @level0name = N'report',
    @level1type = N'VIEW',   @level1name = N'vw_SalesSummary';
```

### Stored Procedure Documentation

```sql
-- ============================================================
-- Stored Procedure: dbo.usp_LoadFact_SalesTransaction
-- ============================================================
EXEC sys.sp_addextendedproperty
    @name  = N'MS_Description',
    @value = N'ETL load procedure for Fact_SalesTransaction. Performs incremental load using HighWaterMark from ETL_ControlTable. Idempotent — safe to re-run for the same load date.',
    @level0type = N'SCHEMA', @level0name = N'dbo',
    @level1type = N'PROCEDURE', @level1name = N'usp_LoadFact_SalesTransaction';

EXEC sys.sp_addextendedproperty
    @name  = N'ExecutionContext',
    @value = N'ETL - called by SSIS package DW_Load.dtsx, step LoadFacts',
    @level0type = N'SCHEMA', @level0name = N'dbo',
    @level1type = N'PROCEDURE', @level1name = N'usp_LoadFact_SalesTransaction';
```

---

## Upsert Pattern (Add or Update Safely)

When deploying to an environment where the property may already exist:

```sql
-- Safe upsert for MS_Description on a table
IF EXISTS (
    SELECT 1 FROM sys.extended_properties
    WHERE major_id = OBJECT_ID(N'dbo.Dim_Customer')
      AND minor_id = 0
      AND class = 1
      AND [name] = N'MS_Description'
)
    EXEC sys.sp_updateextendedproperty
        @name  = N'MS_Description',
        @value = N'<your description>',
        @level0type = N'SCHEMA', @level0name = N'dbo',
        @level1type = N'TABLE',  @level1name = N'Dim_Customer';
ELSE
    EXEC sys.sp_addextendedproperty
        @name  = N'MS_Description',
        @value = N'<your description>',
        @level0type = N'SCHEMA', @level0name = N'dbo',
        @level1type = N'TABLE',  @level1name = N'Dim_Customer';
```

---

## Bulk Audit — Find Objects Missing MS_Description

```sql
-- Tables missing MS_Description
SELECT
    s.[name] AS SchemaName,
    t.[name] AS TableName,
    'TABLE' AS ObjectType
FROM sys.tables t
JOIN sys.schemas s ON t.[schema_id] = s.[schema_id]
WHERE t.is_ms_shipped = 0
  AND NOT EXISTS (
    SELECT 1 FROM sys.extended_properties ep
    WHERE ep.major_id = t.[object_id]
      AND ep.minor_id = 0
      AND ep.class = 1
      AND ep.[name] = N'MS_Description'
  )

UNION ALL

-- Views missing MS_Description
SELECT s.[name], v.[name], 'VIEW'
FROM sys.views v
JOIN sys.schemas s ON v.[schema_id] = s.[schema_id]
WHERE v.is_ms_shipped = 0
  AND NOT EXISTS (
    SELECT 1 FROM sys.extended_properties ep
    WHERE ep.major_id = v.[object_id]
      AND ep.minor_id = 0
      AND ep.class = 1
      AND ep.[name] = N'MS_Description'
  )

UNION ALL

-- Stored procedures missing MS_Description
SELECT s.[name], p.[name], 'PROCEDURE'
FROM sys.procedures p
JOIN sys.schemas s ON p.[schema_id] = s.[schema_id]
WHERE p.is_ms_shipped = 0
  AND NOT EXISTS (
    SELECT 1 FROM sys.extended_properties ep
    WHERE ep.major_id = p.[object_id]
      AND ep.minor_id = 0
      AND ep.class = 1
      AND ep.[name] = N'MS_Description'
  )

ORDER BY ObjectType, SchemaName, TableName;
```

```sql
-- Columns missing MS_Description (for a specific table)
SELECT
    OBJECT_SCHEMA_NAME(c.object_id) AS SchemaName,
    OBJECT_NAME(c.object_id) AS TableName,
    c.[name] AS ColumnName,
    t.[name] AS DataType
FROM sys.columns c
JOIN sys.types t ON c.user_type_id = t.user_type_id
WHERE OBJECT_SCHEMA_NAME(c.object_id) NOT IN ('sys', 'INFORMATION_SCHEMA')
  AND OBJECTPROPERTY(c.object_id, 'IsMsShipped') = 0
  AND NOT EXISTS (
    SELECT 1 FROM sys.extended_properties ep
    WHERE ep.major_id = c.object_id
      AND ep.minor_id = c.column_id
      AND ep.class = 1
      AND ep.[name] = N'MS_Description'
  )
ORDER BY SchemaName, TableName, c.column_id;
```

---

## SSDT Integration Note

When managing a database project in SQL Server Data Tools (SSDT / `.sqlproj`):
- Extended property scripts belong in `Scripts\Post-Deployment` 
- Use `IF EXISTS / UPDATE ELSE ADD` pattern (upsert above) for idempotency
- Or use a dedicated `Scripts\Post-Deployment\ExtendedProperties\` subfolder with one file per table
- Extended properties defined in the DACPAC model will also round-trip if authored via table designer
