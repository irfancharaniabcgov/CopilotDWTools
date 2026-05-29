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
| `InformationType` | Column | The category of information stored in this column. See **Information Type Reference** below. Default new columns to `Unreviewed`. |
| `SensitivityLabel` | Column, Table | The sensitivity classification of this object. See **Sensitivity Label Reference** below. Default new columns to `Unreviewed`. |
| `IsActive` | Table | `Yes` / `No` — is this object actively used in production |
| `BusinessOwner` | Table | Team or person accountable for the data |
| `DataSteward` | Table | Person responsible for data quality |
| `RefreshFrequency` | Table | `Daily`, `Hourly`, `Weekly`, `Monthly`, `On-demand` |
| `CreatedDate` | Table | Date the object was first created |
| `LastReviewedDate` | Table | Date the documentation was last reviewed/verified |
| `RelatedReport` | Table, View | Power BI report or SSRS report that consumes this object |

---

## Data Classification Reference

These are the **official classification values** for this organization. Always use exact casing as shown.

### Information Types

Apply `InformationType` to individual **columns** to describe what category of data is stored.
A column may contain multiple information types — use comma-separated values or set multiple properties.

| InformationType Value | Description | Example Columns |
|---|---|---|
| `Unreviewed` | This column has not been reviewed — **default for all new columns** | Any new column not yet assessed |
| `Banking` | Bank-related data | `BankAccountNumber`, `RoutingNumber`, `TransitNumber` |
| `Contact Info` | Contact data of a person | `AddressLine1`, `City`, `PostalCode`, `PhoneNumber`, `CellNumber`, `URL`, `EmailAddress` |
| `Credentials` | Credential-related data | `Username`, `PasswordHash`, `IDIRUsername`, `BCeIDUsername` |
| `Credit Card` | Credit card data | `CreditCardNumber`, `CardExpiry`, `CardType`, `CVV` |
| `Date of Birth` | Person's date of birth | `DateOfBirth`, `BirthYear`, `BirthDate` |
| `Financial` | Finance-related data | `InvoiceAmount`, `PaymentAmount`, `TaxAmount`, `WageAmount` |
| `Free-form Text` | Free-form/unstructured text | `Comments`, `Description`, `MemoField`, `Notes`, `Remarks` |
| `Health` | Health-related data | `HealthNumber`, `PHN`, `MSPNumber` |
| `Identification` | Person identification data | `PassportNumber`, `DriversLicenseNumber`, `IdentificationNumber` |
| `Name` | Person's name data | `FirstName`, `LastName`, `MiddleName`, `Alias`, `LegalName` |
| `Networking` | Network/infrastructure data | `IPAddress`, `MACAddress`, `HostName` |
| `Personal` | Personal characteristics | `Gender`, `Race`, `Indigeneity`, `Ability`, `Education` |
| `Other` | Data not covered by above types | `Gender`, `Race`, `Indigeneity`, `Ability` *(also listed under Personal)* |
| `SIN` | Social Insurance Number | `SIN`, `SocialInsuranceNumber` |

> **Note**: `Personal` and `Other` have overlapping examples in the source taxonomy (gender, race, indigeneity, ability, education). Apply `Personal` when the column specifically stores demographic/personal characteristics about an individual. Use `Other` when no other type fits.

---

### Sensitivity Labels

Apply `SensitivityLabel` to **both tables and columns**. The table-level label should reflect the highest sensitivity of any column it contains.

| SensitivityLabel Value | Description | Disclosure Risk | Examples |
|---|---|---|---|
| `Unreviewed` | Not yet reviewed — **default for all new objects** | Unknown | Any new table/column not yet assessed |
| `Public` | No harm to an individual, organization or government | None — widest permissible distribution. **Does not include personal information.** | Published statistics, lookup codes, reference data |
| `Protected A` | Harm to an individual, organization or government | Unauthorized disclosure could cause **harm or embarrassment** — positive identification risk | Home address, date of birth |
| `Protected B` | Serious harm to an individual, organization or government | Unauthorized disclosure likely causes **serious harm** such as loss of reputation | Financial information, health records |
| `Protected C` | Extremely grave harm to an individual, organization or government | Unauthorized disclosure could cause **loss of life or extremely grave harm**; intended for named individuals only | Cabinet-related information, witness protection data |

#### Sensitivity Label Decision Guide

```
Is the data personal information?
  └─ No  → Public (if cleared for distribution) or Unreviewed
  └─ Yes → Does it identify a person?
            └─ Could cause harm/embarrassment if disclosed?  → Protected A
               (e.g., home address, date of birth, contact info)
            └─ Could cause SERIOUS harm if disclosed?       → Protected B
               (e.g., financial info, health data, SIN, credentials)
            └─ Could cause LOSS OF LIFE or EXTREMELY GRAVE harm? → Protected C
               (extremely rare — escalate to Privacy Officer)
```

---

## Column Classification Templates

### Setting InformationType and SensitivityLabel on a Column

```sql
-- ============================================================
-- Example: Person name column
-- ============================================================
EXEC sys.sp_addextendedproperty
    @name  = N'InformationType',
    @value = N'Name',
    @level0type = N'SCHEMA', @level0name = N'dbo',
    @level1type = N'TABLE',  @level1name = N'Dim_Person',
    @level2type = N'COLUMN', @level2name = N'FirstName';

EXEC sys.sp_addextendedproperty
    @name  = N'SensitivityLabel',
    @value = N'Protected A',
    @level0type = N'SCHEMA', @level0name = N'dbo',
    @level1type = N'TABLE',  @level1name = N'Dim_Person',
    @level2type = N'COLUMN', @level2name = N'FirstName';

-- ============================================================
-- Example: Financial amount column
-- ============================================================
EXEC sys.sp_addextendedproperty
    @name  = N'InformationType',
    @value = N'Financial',
    @level0type = N'SCHEMA', @level0name = N'dbo',
    @level1type = N'TABLE',  @level1name = N'Fact_Payment',
    @level2type = N'COLUMN', @level2name = N'PaymentAmount';

EXEC sys.sp_addextendedproperty
    @name  = N'SensitivityLabel',
    @value = N'Protected B',
    @level0type = N'SCHEMA', @level0name = N'dbo',
    @level1type = N'TABLE',  @level1name = N'Fact_Payment',
    @level2type = N'COLUMN', @level2name = N'PaymentAmount';

-- ============================================================
-- Example: SIN column
-- ============================================================
EXEC sys.sp_addextendedproperty
    @name  = N'InformationType',
    @value = N'SIN',
    @level0type = N'SCHEMA', @level0name = N'dbo',
    @level1type = N'TABLE',  @level1name = N'Dim_Person',
    @level2type = N'COLUMN', @level2name = N'SocialInsuranceNumber';

EXEC sys.sp_addextendedproperty
    @name  = N'SensitivityLabel',
    @value = N'Protected B',
    @level0type = N'SCHEMA', @level0name = N'dbo',
    @level1type = N'TABLE',  @level1name = N'Dim_Person',
    @level2type = N'COLUMN', @level2name = N'SocialInsuranceNumber';

-- ============================================================
-- Example: Setting table-level SensitivityLabel
-- (Set to the highest label of any column in the table)
-- ============================================================
EXEC sys.sp_addextendedproperty
    @name  = N'SensitivityLabel',
    @value = N'Protected B',
    @level0type = N'SCHEMA', @level0name = N'dbo',
    @level1type = N'TABLE',  @level1name = N'Dim_Person';

-- ============================================================
-- Example: Non-personal reference data
-- ============================================================
EXEC sys.sp_addextendedproperty
    @name  = N'InformationType',
    @value = N'Financial',
    @level0type = N'SCHEMA', @level0name = N'dbo',
    @level1type = N'TABLE',  @level1name = N'Fact_Invoice',
    @level2type = N'COLUMN', @level2name = N'InvoiceTotal';

EXEC sys.sp_addextendedproperty
    @name  = N'SensitivityLabel',
    @value = N'Protected B',
    @level0type = N'SCHEMA', @level0name = N'dbo',
    @level1type = N'TABLE',  @level1name = N'Fact_Invoice',
    @level2type = N'COLUMN', @level2name = N'InvoiceTotal';
```

### Common Column → Classification Mapping

| Column Name Pattern | Likely InformationType | Minimum SensitivityLabel |
|---|---|---|
| `*FirstName*`, `*LastName*`, `*FullName*`, `*Alias*` | `Name` | `Protected A` |
| `*DateOfBirth*`, `*BirthDate*`, `*BirthYear*` | `Date of Birth` | `Protected A` |
| `*Address*`, `*Street*`, `*City*`, `*PostalCode*`, `*ZipCode*` | `Contact Info` | `Protected A` |
| `*Phone*`, `*Cell*`, `*Mobile*`, `*Fax*` | `Contact Info` | `Protected A` |
| `*Email*` | `Contact Info` | `Protected A` |
| `*SIN*`, `*SocialInsurance*` | `SIN` | `Protected B` |
| `*PHN*`, `*HealthNumber*`, `*MSP*` | `Health` | `Protected B` |
| `*Password*`, `*PasswordHash*`, `*IDIR*`, `*BCeID*`, `*Username*` | `Credentials` | `Protected B` |
| `*BankAccount*`, `*RoutingNumber*`, `*Transit*` | `Banking` | `Protected B` |
| `*CreditCard*`, `*CardNumber*`, `*CVV*`, `*CardExpiry*` | `Credit Card` | `Protected B` |
| `*Amount*`, `*Payment*`, `*Invoice*`, `*Wage*`, `*Salary*`, `*Tax*` | `Financial` | `Protected B` |
| `*Passport*`, `*DriversLicense*`, `*DriverLicense*` | `Identification` | `Protected B` |
| `*Gender*`, `*Race*`, `*Indigeneity*`, `*Ethnicity*`, `*Ability*` | `Personal` | `Protected B` |
| `*IPAddress*`, `*MACAddress*`, `*HostName*` | `Networking` | `Protected A` |
| `*Comment*`, `*Notes*`, `*Memo*`, `*Description*`, `*Remarks*` | `Free-form Text` | `Protected A` (may contain personal info) |
| `*Code*`, `*Type*`, `*Status*`, `*Flag*`, `*Indicator*` | *(review individually)* | `Public` or `Protected A` |

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

## Bulk Audit — Data Classification Coverage

Use these queries to find columns missing `InformationType` or `SensitivityLabel`, and to
produce a full classification inventory for privacy/security reviews.

```sql
-- ----------------------------------------------------------------
-- Columns missing InformationType (not yet classified)
-- ----------------------------------------------------------------
SELECT
    OBJECT_SCHEMA_NAME(c.object_id)  AS SchemaName,
    OBJECT_NAME(c.object_id)         AS TableName,
    c.[name]                          AS ColumnName,
    TYPE_NAME(c.user_type_id)         AS DataType,
    c.max_length,
    c.is_nullable
FROM sys.columns c
WHERE OBJECTPROPERTY(c.object_id, 'IsMsShipped') = 0
  AND OBJECTPROPERTY(c.object_id, 'IsTable') = 1
  AND OBJECT_SCHEMA_NAME(c.object_id) NOT IN ('sys', 'INFORMATION_SCHEMA', 'staging')
  AND NOT EXISTS (
    SELECT 1 FROM sys.extended_properties ep
    WHERE ep.major_id = c.object_id
      AND ep.minor_id = c.column_id
      AND ep.class = 1
      AND ep.[name] = N'InformationType'
  )
ORDER BY SchemaName, TableName, c.column_id;
```

```sql
-- ----------------------------------------------------------------
-- Columns with InformationType = 'Unreviewed' (pending assessment)
-- ----------------------------------------------------------------
SELECT
    OBJECT_SCHEMA_NAME(c.object_id)            AS SchemaName,
    OBJECT_NAME(c.object_id)                   AS TableName,
    c.[name]                                    AS ColumnName,
    CAST(ep.[value] AS NVARCHAR(100))           AS InformationType
FROM sys.columns c
JOIN sys.extended_properties ep
    ON ep.major_id  = c.object_id
   AND ep.minor_id  = c.column_id
   AND ep.class     = 1
   AND ep.[name]    = N'InformationType'
WHERE CAST(ep.[value] AS NVARCHAR(100)) = N'Unreviewed'
  AND OBJECTPROPERTY(c.object_id, 'IsMsShipped') = 0
ORDER BY SchemaName, TableName, c.column_id;
```

```sql
-- ----------------------------------------------------------------
-- Full classification inventory — all classified columns
-- Use for privacy impact assessments and data governance reporting
-- ----------------------------------------------------------------
SELECT
    OBJECT_SCHEMA_NAME(c.object_id)    AS SchemaName,
    OBJECT_NAME(c.object_id)           AS TableName,
    c.[name]                            AS ColumnName,
    TYPE_NAME(c.user_type_id)           AS DataType,
    CAST(it.value  AS NVARCHAR(100))    AS InformationType,
    CAST(sl.value  AS NVARCHAR(100))    AS SensitivityLabel
FROM sys.columns c
LEFT JOIN sys.extended_properties it
    ON it.major_id = c.object_id AND it.minor_id = c.column_id
   AND it.class = 1 AND it.[name] = N'InformationType'
LEFT JOIN sys.extended_properties sl
    ON sl.major_id = c.object_id AND sl.minor_id = c.column_id
   AND sl.class = 1 AND sl.[name] = N'SensitivityLabel'
WHERE OBJECTPROPERTY(c.object_id, 'IsMsShipped') = 0
  AND OBJECTPROPERTY(c.object_id, 'IsTable') = 1
  AND OBJECT_SCHEMA_NAME(c.object_id) NOT IN ('sys', 'INFORMATION_SCHEMA', 'staging')
  AND (it.value IS NOT NULL OR sl.value IS NOT NULL)
ORDER BY
    CAST(sl.value AS NVARCHAR(100)) DESC,   -- Protected C first
    SchemaName,
    TableName,
    c.column_id;
```

```sql
-- ----------------------------------------------------------------
-- Tables whose SensitivityLabel is Unreviewed or missing
-- (table-level label should reflect highest column label)
-- ----------------------------------------------------------------
SELECT
    s.[name] AS SchemaName,
    t.[name] AS TableName,
    ISNULL(CAST(ep.[value] AS NVARCHAR(100)), 'MISSING') AS TableSensitivityLabel
FROM sys.tables t
JOIN sys.schemas s ON t.[schema_id] = s.[schema_id]
LEFT JOIN sys.extended_properties ep
    ON ep.major_id = t.[object_id]
   AND ep.minor_id = 0
   AND ep.class    = 1
   AND ep.[name]   = N'SensitivityLabel'
WHERE t.is_ms_shipped = 0
  AND s.[name] NOT IN ('staging')
  AND (ep.value IS NULL OR CAST(ep.value AS NVARCHAR(100)) = N'Unreviewed')
ORDER BY SchemaName, TableName;
```

```sql
-- ----------------------------------------------------------------
-- Summary: Column counts by SensitivityLabel
-- Use to gauge overall classification completeness
-- ----------------------------------------------------------------
SELECT
    ISNULL(CAST(sl.value AS NVARCHAR(100)), 'Not classified') AS SensitivityLabel,
    COUNT(*) AS ColumnCount
FROM sys.columns c
JOIN sys.tables t  ON c.object_id = t.object_id AND t.is_ms_shipped = 0
JOIN sys.schemas s ON t.schema_id = s.schema_id
LEFT JOIN sys.extended_properties sl
    ON sl.major_id = c.object_id AND sl.minor_id = c.column_id
   AND sl.class = 1 AND sl.[name] = N'SensitivityLabel'
WHERE s.[name] NOT IN ('staging', 'sys', 'INFORMATION_SCHEMA')
GROUP BY CAST(sl.value AS NVARCHAR(100))
ORDER BY
    CASE CAST(sl.value AS NVARCHAR(100))
        WHEN 'Protected C' THEN 1
        WHEN 'Protected B' THEN 2
        WHEN 'Protected A' THEN 3
        WHEN 'Public'      THEN 4
        WHEN 'Unreviewed'  THEN 5
        ELSE 6
    END;
```

---

## SSDT Integration Note

When managing a database project in SQL Server Data Tools (SSDT / `.sqlproj`):
- Extended property scripts belong in `Scripts\Post-Deployment`
- Use `IF EXISTS / UPDATE ELSE ADD` pattern (upsert above) for idempotency
- Recommended subfolder structure:
  ```
  Scripts\Post-Deployment\
    ExtendedProperties\
      Tables\
        dbo.Dim_Customer.sql        ← MS_Description, TableType, SensitivityLabel
        dbo.Fact_SalesOrder.sql
      Columns\
        dbo.Dim_Customer.Columns.sql ← InformationType, SensitivityLabel per column
      Views\
      Procedures\
  ```
- Extended properties defined in the DACPAC model will also round-trip if authored via table designer
- All new tables/columns should default `InformationType` to `Unreviewed` and `SensitivityLabel` to `Unreviewed` in the post-deploy script, pending formal privacy review
