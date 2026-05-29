# SQL Server Native Data Classification

**Scope**: SQL Server 2019+ native sensitivity classification using `ADD SENSITIVITY CLASSIFICATION`.
This reference covers the organization's Information Type and Sensitivity Label taxonomy, T-SQL
scripts for applying and auditing classifications, SSDT deployment patterns, and the relationship
to the extended properties approach.

---

## Overview

SQL Server 2019 introduced **native data classification** as a first-class database feature.
Unlike the extended properties approach (`sp_addextendedproperty`), native classification:

- Stores metadata in `sys.sensitivity_classifications` — a dedicated DMV, not generic key/value
- Is surfaced in SSMS Data Discovery & Classification (GUI tool for review and bulk apply)
- Is recognized by Microsoft Purview, Defender for SQL, and compliance scanning tools
- Supports `RANK` — a machine-readable severity integer (NONE/LOW/MEDIUM/HIGH/CRITICAL)
- Requires `ALTER ANY SENSITIVITY CLASSIFICATION` permission (not `db_ddladmin`)

**Supported objects**: `ADD SENSITIVITY CLASSIFICATION` applies to **table columns only**.
For views, stored procedures, and non-table objects, continue using `sp_addextendedproperty`
(see `extended-properties-templates.md`).

### Version Requirements

| Feature | Minimum Version |
|---|---|
| `ADD SENSITIVITY CLASSIFICATION` T-SQL | SQL Server 2019 (150) |
| `sys.sensitivity_classifications` DMV | SQL Server 2019 (150) |
| SSMS Data Discovery & Classification GUI | SSMS 18.0+ |
| DACPAC native serialization of classifications | SSDT 17.8+ (partial); use post-deploy scripts to be safe |
| `RANK` parameter | SQL Server 2019 (150) |

---

## Organization Classification Taxonomy

### Information Types

Used to describe *what kind of data* a column contains.

| Information Type | Description | Example Values |
|---|---|---|
| `Unreviewed` | Not yet assessed — default for new columns | *(any unreviewed column)* |
| `Banking` | Bank account data | Account numbers, routing numbers |
| `Contact Info` | Contact details of a person | Address, city, postal code, phone, URL |
| `Credentials` | Authentication credentials | Username, password, IDIR, BCeID |
| `Credit Card` | Payment card data | Card numbers, expiry dates, card types |
| `Date of Birth` | Birth-related data | Birthday, birth year, birth date |
| `Financial` | Financial transaction data | Amounts, invoices, payments, tax |
| `Free-form Text` | Unstructured text | Comments, descriptions, memo, notes |
| `Health` | Health identifier data | Health number, PHN, MSP number |
| `Identification` | Government-issued ID data | Passport number, driver's license |
| `Name` | Person's name | First name, middle name, last name, alias |
| `Networking` | Network infrastructure data | IP address, MAC address |
| `Other` | Catch-all for uncategorized data | Gender, race, indigeneity, ability, education |
| `Personal` | Demographic personal data | Gender, race, indigeneity, ability, education |
| `SIN` | Social Insurance Number | SIN |

> **Note on `Other` vs `Personal`**: Use `Personal` for demographic attributes (gender, race,
> indigeneity, ability, education). Use `Other` only for data that genuinely fits no other category.
> Confirm with your Privacy Officer for specific cases.

### Sensitivity Labels

Used to describe *how sensitive* the data is and what harm disclosure could cause.

| Label | RANK | Description | Examples |
|---|---|---|---|
| `Unreviewed` | `NONE` | Not yet assessed | *(any unreviewed column)* |
| `Public` | `LOW` | No harm from disclosure; no personal information | Reference codes, public names |
| `Protected A` | `MEDIUM` | Could cause harm or embarrassment to a person | Home address, date of birth |
| `Protected B` | `HIGH` | Could cause serious harm (reputation, financial loss) | Financial information, SIN |
| `Protected C` | `CRITICAL` | Could cause extremely grave harm, including loss of life; cabinet-level | Cabinet records, life-safety data |

**Rule**: A table's sensitivity label = the highest label of any column it contains.
Assign the table-level label via `sp_addextendedproperty` (tables are not supported by native classification).

---

## T-SQL Syntax

### ADD SENSITIVITY CLASSIFICATION

```sql
-- Basic syntax (SQL Server 2019+)
ADD SENSITIVITY CLASSIFICATION TO <schema>.<table>.<column>
WITH (
    LABEL           = '<sensitivity_label>',
    INFORMATION_TYPE = '<information_type>',
    RANK            = <NONE | LOW | MEDIUM | HIGH | CRITICAL>
);
```

### UPDATE (DROP + ADD)

There is no `ALTER SENSITIVITY CLASSIFICATION`. To change a classification, drop and re-add:

```sql
DROP SENSITIVITY CLASSIFICATION FROM dbo.Dim_Customer.PHN;

ADD SENSITIVITY CLASSIFICATION TO dbo.Dim_Customer.PHN
WITH (
    LABEL            = 'Protected B',
    INFORMATION_TYPE = 'Health',
    RANK             = HIGH
);
```

### Safe Upsert Pattern (Idempotent — for deployment scripts)

```sql
-- Drop if exists, then add — ensures script is safe to re-run
IF EXISTS (
    SELECT 1 FROM sys.sensitivity_classifications sc
    JOIN sys.columns c ON sc.object_id = c.object_id AND sc.column_id = c.column_id
    WHERE sc.object_id = OBJECT_ID(N'dbo.Dim_Customer')
      AND c.[name] = N'PHN'
)
    DROP SENSITIVITY CLASSIFICATION FROM dbo.Dim_Customer.PHN;

ADD SENSITIVITY CLASSIFICATION TO dbo.Dim_Customer.PHN
WITH (
    LABEL            = 'Protected B',
    INFORMATION_TYPE = 'Health',
    RANK             = HIGH
);
```

---

## Column Classification Templates by Information Type

### Health Data (Protected B)
```sql
-- PHN, Health Number, MSP
ADD SENSITIVITY CLASSIFICATION TO dbo.Dim_Patient.PHN
    WITH (LABEL = 'Protected B', INFORMATION_TYPE = 'Health', RANK = HIGH);
ADD SENSITIVITY CLASSIFICATION TO dbo.Dim_Patient.HealthNumber
    WITH (LABEL = 'Protected B', INFORMATION_TYPE = 'Health', RANK = HIGH);
```

### SIN (Protected B)
```sql
ADD SENSITIVITY CLASSIFICATION TO dbo.Dim_Employee.SIN
    WITH (LABEL = 'Protected B', INFORMATION_TYPE = 'SIN', RANK = HIGH);
ADD SENSITIVITY CLASSIFICATION TO dbo.Dim_Employee.SocialInsuranceNumber
    WITH (LABEL = 'Protected B', INFORMATION_TYPE = 'SIN', RANK = HIGH);
```

### Name — Personal Name (Protected A)
```sql
ADD SENSITIVITY CLASSIFICATION TO dbo.Dim_Customer.FirstName
    WITH (LABEL = 'Protected A', INFORMATION_TYPE = 'Name', RANK = MEDIUM);
ADD SENSITIVITY CLASSIFICATION TO dbo.Dim_Customer.LastName
    WITH (LABEL = 'Protected A', INFORMATION_TYPE = 'Name', RANK = MEDIUM);
ADD SENSITIVITY CLASSIFICATION TO dbo.Dim_Customer.FullName
    WITH (LABEL = 'Protected A', INFORMATION_TYPE = 'Name', RANK = MEDIUM);
```

### Contact Info (Protected A)
```sql
ADD SENSITIVITY CLASSIFICATION TO dbo.Dim_Customer.AddressLine1
    WITH (LABEL = 'Protected A', INFORMATION_TYPE = 'Contact Info', RANK = MEDIUM);
ADD SENSITIVITY CLASSIFICATION TO dbo.Dim_Customer.City
    WITH (LABEL = 'Protected A', INFORMATION_TYPE = 'Contact Info', RANK = MEDIUM);
ADD SENSITIVITY CLASSIFICATION TO dbo.Dim_Customer.PostalCode
    WITH (LABEL = 'Protected A', INFORMATION_TYPE = 'Contact Info', RANK = MEDIUM);
ADD SENSITIVITY CLASSIFICATION TO dbo.Dim_Customer.Phone
    WITH (LABEL = 'Protected A', INFORMATION_TYPE = 'Contact Info', RANK = MEDIUM);
ADD SENSITIVITY CLASSIFICATION TO dbo.Dim_Customer.Email
    WITH (LABEL = 'Protected A', INFORMATION_TYPE = 'Contact Info', RANK = MEDIUM);
```

### Date of Birth (Protected A)
```sql
ADD SENSITIVITY CLASSIFICATION TO dbo.Dim_Customer.DateOfBirth
    WITH (LABEL = 'Protected A', INFORMATION_TYPE = 'Date of Birth', RANK = MEDIUM);
ADD SENSITIVITY CLASSIFICATION TO dbo.Dim_Customer.BirthYear
    WITH (LABEL = 'Protected A', INFORMATION_TYPE = 'Date of Birth', RANK = MEDIUM);
```

### Credentials / Authentication (Protected B)
```sql
ADD SENSITIVITY CLASSIFICATION TO dbo.Dim_User.IDIRUsername
    WITH (LABEL = 'Protected B', INFORMATION_TYPE = 'Credentials', RANK = HIGH);
ADD SENSITIVITY CLASSIFICATION TO dbo.Dim_User.BCeIDUsername
    WITH (LABEL = 'Protected B', INFORMATION_TYPE = 'Credentials', RANK = HIGH);
```

### Financial (Protected B)
```sql
ADD SENSITIVITY CLASSIFICATION TO dbo.Fact_Payment.PaymentAmount
    WITH (LABEL = 'Protected B', INFORMATION_TYPE = 'Financial', RANK = HIGH);
ADD SENSITIVITY CLASSIFICATION TO dbo.Fact_Invoice.InvoiceAmount
    WITH (LABEL = 'Protected B', INFORMATION_TYPE = 'Financial', RANK = HIGH);
```

### Identification (Protected A or B depending on type)
```sql
-- Driver's license, passport → Protected A
ADD SENSITIVITY CLASSIFICATION TO dbo.Dim_Person.DriversLicenseNumber
    WITH (LABEL = 'Protected A', INFORMATION_TYPE = 'Identification', RANK = MEDIUM);

-- Combined with other identifiers → potentially Protected B — assess with Privacy Officer
```

### Personal / Demographic (Protected A)
```sql
ADD SENSITIVITY CLASSIFICATION TO dbo.Dim_Person.Gender
    WITH (LABEL = 'Protected A', INFORMATION_TYPE = 'Personal', RANK = MEDIUM);
ADD SENSITIVITY CLASSIFICATION TO dbo.Dim_Person.Indigeneity
    WITH (LABEL = 'Protected A', INFORMATION_TYPE = 'Personal', RANK = MEDIUM);
ADD SENSITIVITY CLASSIFICATION TO dbo.Dim_Person.Disability
    WITH (LABEL = 'Protected A', INFORMATION_TYPE = 'Personal', RANK = MEDIUM);
```

### Public / Reference Data (no personal data)
```sql
ADD SENSITIVITY CLASSIFICATION TO dbo.Dim_Region.RegionCode
    WITH (LABEL = 'Public', INFORMATION_TYPE = 'Other', RANK = LOW);
ADD SENSITIVITY CLASSIFICATION TO dbo.Dim_Product.ProductCode
    WITH (LABEL = 'Public', INFORMATION_TYPE = 'Other', RANK = LOW);
```

---

## Label-to-RANK Mapping Reference

When writing classification scripts, always match LABEL to RANK consistently:

```sql
-- Quick reference — always use this mapping
-- 'Unreviewed'  → RANK = NONE
-- 'Public'      → RANK = LOW
-- 'Protected A' → RANK = MEDIUM
-- 'Protected B' → RANK = HIGH
-- 'Protected C' → RANK = CRITICAL
```

---

## Audit Queries

### Full Classification Inventory

```sql
-- Complete classification report — use for privacy impact assessments
SELECT
    OBJECT_SCHEMA_NAME(sc.object_id)   AS SchemaName,
    OBJECT_NAME(sc.object_id)          AS TableName,
    c.[name]                            AS ColumnName,
    TYPE_NAME(c.user_type_id)           AS DataType,
    sc.information_type                 AS InformationType,
    sc.label                            AS SensitivityLabel,
    sc.rank_desc                        AS Rank,
    sc.rank                             AS RankInt
FROM sys.sensitivity_classifications sc
JOIN sys.columns c
    ON sc.object_id = c.object_id AND sc.column_id = c.column_id
JOIN sys.tables t
    ON c.object_id = t.object_id AND t.is_ms_shipped = 0
ORDER BY sc.rank DESC, SchemaName, TableName, c.column_id;
```

### Columns with No Classification (Unclassified)

```sql
-- Table columns with no entry in sys.sensitivity_classifications
SELECT
    OBJECT_SCHEMA_NAME(c.object_id)  AS SchemaName,
    OBJECT_NAME(c.object_id)         AS TableName,
    c.[name]                          AS ColumnName,
    TYPE_NAME(c.user_type_id)         AS DataType
FROM sys.columns c
JOIN sys.tables t ON c.object_id = t.object_id AND t.is_ms_shipped = 0
WHERE OBJECT_SCHEMA_NAME(c.object_id) NOT IN ('staging', 'sys', 'INFORMATION_SCHEMA')
  AND NOT EXISTS (
    SELECT 1 FROM sys.sensitivity_classifications sc
    WHERE sc.object_id = c.object_id AND sc.column_id = c.column_id
  )
ORDER BY SchemaName, TableName, c.column_id;
```

### Protected B and C Columns (Highest Risk)

```sql
-- Columns classified as Protected B or Protected C — highest priority for access control review
SELECT
    OBJECT_SCHEMA_NAME(sc.object_id)  AS SchemaName,
    OBJECT_NAME(sc.object_id)         AS TableName,
    c.[name]                           AS ColumnName,
    sc.information_type                AS InformationType,
    sc.label                           AS SensitivityLabel,
    sc.rank_desc                       AS Rank
FROM sys.sensitivity_classifications sc
JOIN sys.columns c ON sc.object_id = c.object_id AND sc.column_id = c.column_id
WHERE sc.rank >= 20   -- HIGH (Protected B) and CRITICAL (Protected C)
ORDER BY sc.rank DESC, SchemaName, TableName;
```

### Summary by Label

```sql
-- How many columns are at each sensitivity level?
SELECT
    sc.label                          AS SensitivityLabel,
    sc.rank_desc                      AS Rank,
    COUNT(*)                          AS ColumnCount,
    COUNT(DISTINCT sc.object_id)      AS TableCount
FROM sys.sensitivity_classifications sc
GROUP BY sc.label, sc.rank_desc, sc.rank
ORDER BY sc.rank DESC;
```

### Summary: Classified vs Unclassified (Coverage %)

```sql
WITH TotalColumns AS (
    SELECT COUNT(*) AS Total
    FROM sys.columns c
    JOIN sys.tables t ON c.object_id = t.object_id AND t.is_ms_shipped = 0
    WHERE OBJECT_SCHEMA_NAME(c.object_id) NOT IN ('staging')
),
ClassifiedColumns AS (
    SELECT COUNT(*) AS Classified
    FROM sys.sensitivity_classifications sc
    JOIN sys.tables t ON sc.object_id = t.object_id AND t.is_ms_shipped = 0
)
SELECT
    tc.Total                                               AS TotalColumns,
    cc.Classified                                          AS ClassifiedColumns,
    tc.Total - cc.Classified                               AS UnclassifiedColumns,
    CAST(cc.Classified * 100.0 / tc.Total AS DECIMAL(5,1)) AS CoveragePct
FROM TotalColumns tc, ClassifiedColumns cc;
```

### Tables with High-Sensitivity Columns (for table-level label assignment)

```sql
-- Shows the maximum column rank per table
-- Use this to assign the table-level SensitivityLabel extended property
SELECT
    OBJECT_SCHEMA_NAME(sc.object_id)   AS SchemaName,
    OBJECT_NAME(sc.object_id)          AS TableName,
    MAX(sc.rank)                        AS MaxRank,
    MAX(sc.rank_desc)                   AS MaxRankDesc,
    CASE MAX(sc.rank)
        WHEN 30 THEN 'Protected C'
        WHEN 25 THEN 'Protected C'
        WHEN 20 THEN 'Protected B'
        WHEN 10 THEN 'Protected A'
        WHEN 0  THEN 'Public'
        ELSE         'Unreviewed'
    END                                  AS RecommendedTableLabel,
    STRING_AGG(sc.label + ':' + c.[name], ', ')
        WITHIN GROUP (ORDER BY sc.rank DESC)
        AS HighestColumns
FROM sys.sensitivity_classifications sc
JOIN sys.columns c ON sc.object_id = c.object_id AND sc.column_id = c.column_id
JOIN sys.tables t  ON sc.object_id = t.object_id AND t.is_ms_shipped = 0
GROUP BY sc.object_id
ORDER BY MAX(sc.rank) DESC;
```

---

## Bulk Classification Generation Script

Use this to generate `ADD SENSITIVITY CLASSIFICATION` statements from the extended properties
already applied (migration from `InformationType`/`SensitivityLabel` extended properties to
native classification):

```sql
-- Generate ADD SENSITIVITY CLASSIFICATION statements from existing extended properties
-- Run in SSMS, copy output to a new script and review before running
SELECT
    'IF EXISTS (SELECT 1 FROM sys.sensitivity_classifications sc ' +
    'JOIN sys.columns c ON sc.object_id = c.object_id AND sc.column_id = c.column_id ' +
    'WHERE sc.object_id = OBJECT_ID(N''' + QUOTENAME(OBJECT_SCHEMA_NAME(c.object_id)) + '.' +
        QUOTENAME(OBJECT_NAME(c.object_id)) + ''') AND c.[name] = N''' + c.[name] + ''') ' + CHAR(13) +
    '    DROP SENSITIVITY CLASSIFICATION FROM ' + QUOTENAME(OBJECT_SCHEMA_NAME(c.object_id)) + '.' +
        QUOTENAME(OBJECT_NAME(c.object_id)) + '.' + QUOTENAME(c.[name]) + ';' + CHAR(13) +
    'ADD SENSITIVITY CLASSIFICATION TO ' + QUOTENAME(OBJECT_SCHEMA_NAME(c.object_id)) + '.' +
        QUOTENAME(OBJECT_NAME(c.object_id)) + '.' + QUOTENAME(c.[name]) + CHAR(13) +
    '    WITH (LABEL = ''' + CAST(sl.value AS NVARCHAR(100)) + ''', ' +
          'INFORMATION_TYPE = ''' + CAST(it.value AS NVARCHAR(100)) + ''', ' +
          'RANK = ' + CASE CAST(sl.value AS NVARCHAR(100))
              WHEN 'Protected C' THEN 'CRITICAL'
              WHEN 'Protected B' THEN 'HIGH'
              WHEN 'Protected A' THEN 'MEDIUM'
              WHEN 'Public'      THEN 'LOW'
              ELSE 'NONE'
          END + ');' + CHAR(13) AS GeneratedScript
FROM sys.columns c
JOIN sys.extended_properties it
    ON it.major_id = c.object_id AND it.minor_id = c.column_id
   AND it.class = 1 AND it.[name] = N'InformationType'
JOIN sys.extended_properties sl
    ON sl.major_id = c.object_id AND sl.minor_id = c.column_id
   AND sl.class = 1 AND sl.[name] = N'SensitivityLabel'
WHERE OBJECTPROPERTY(c.object_id, 'IsTable') = 1
  AND OBJECTPROPERTY(c.object_id, 'IsMsShipped') = 0
  AND CAST(sl.value AS NVARCHAR(100)) <> N'Unreviewed'
ORDER BY OBJECT_SCHEMA_NAME(c.object_id), OBJECT_NAME(c.object_id), c.column_id;
```

---

## SSMS Data Discovery & Classification (GUI)

SSMS 18+ includes a built-in GUI tool for reviewing and applying classifications:

1. In SSMS Object Explorer, right-click database → **Tasks** → **Data Discovery & Classification** → **Classify Data**
2. SSMS auto-scans column names and suggests Information Types based on name patterns
3. Review and accept/reject recommendations — suggested classifications appear in the review pane
4. Click **Save** to apply — this runs `ADD SENSITIVITY CLASSIFICATION` behind the scenes
5. Use **View Report** to generate a classification coverage summary

> **For pipeline deployments**: Do not rely on the GUI. All classifications must be scripted in
> `Scripts\Post-Deployment\DataClassification\` using the upsert pattern above.

---

## Relationship to Extended Properties (`extended-properties-templates.md`)

Both approaches are complementary and can coexist:

| Property | Native Classification | Extended Properties |
|---|---|---|
| **Scope** | Table columns only | Tables, columns, views, SPs, functions |
| **Storage** | `sys.sensitivity_classifications` | `sys.extended_properties` |
| **SSMS GUI** | ✅ Data Discovery & Classification | ❌ No GUI |
| **Machine-readable rank** | ✅ `rank` integer + `rank_desc` | ❌ Free text only |
| **Compliance tool integration** | ✅ Purview, Defender for SQL | ❌ Not integrated |
| **DACPAC support** | ⚠️ Use post-deploy scripts | ⚠️ Use post-deploy scripts |
| **Views, SPs** | ❌ Not supported | ✅ Supported |
| **SQL Server version** | SQL Server 2019+ only | All versions |

**Recommended dual approach:**
- **Columns in tables**: Use **native classification** (`ADD SENSITIVITY CLASSIFICATION`) — it's the modern standard
- **Table-level label**: Use **extended property** `SensitivityLabel` (set to the max column label) — tables don't support native classification
- **Views and stored procedures**: Use **extended properties** `InformationType` / `SensitivityLabel`

---

## SSDT Deployment Pattern

Classification scripts do **not** serialize in the DACPAC model. Place them in post-deploy scripts:

```
Database\
  Scripts\
    Post-Deployment\
      DataClassification\
        dbo.Dim_Customer.Classification.sql
        dbo.Dim_Employee.Classification.sql
        dbo.Fact_Payment.Classification.sql
```

Each file follows the idempotent pattern:

```sql
-- dbo.Dim_Customer.Classification.sql
-- Classification script — idempotent, safe to re-run
-- Generated: 2026-05-29 | Reviewed by: <Privacy Officer Name>

-- FirstName
IF EXISTS (SELECT 1 FROM sys.sensitivity_classifications sc
    JOIN sys.columns c ON sc.object_id = c.object_id AND sc.column_id = c.column_id
    WHERE sc.object_id = OBJECT_ID(N'dbo.Dim_Customer') AND c.[name] = N'FirstName')
    DROP SENSITIVITY CLASSIFICATION FROM dbo.Dim_Customer.FirstName;
ADD SENSITIVITY CLASSIFICATION TO dbo.Dim_Customer.FirstName
    WITH (LABEL = 'Protected A', INFORMATION_TYPE = 'Name', RANK = MEDIUM);

-- LastName
IF EXISTS (SELECT 1 FROM sys.sensitivity_classifications sc
    JOIN sys.columns c ON sc.object_id = c.object_id AND sc.column_id = c.column_id
    WHERE sc.object_id = OBJECT_ID(N'dbo.Dim_Customer') AND c.[name] = N'LastName')
    DROP SENSITIVITY CLASSIFICATION FROM dbo.Dim_Customer.LastName;
ADD SENSITIVITY CLASSIFICATION TO dbo.Dim_Customer.LastName
    WITH (LABEL = 'Protected A', INFORMATION_TYPE = 'Name', RANK = MEDIUM);

-- Email
IF EXISTS (SELECT 1 FROM sys.sensitivity_classifications sc
    JOIN sys.columns c ON sc.object_id = c.object_id AND sc.column_id = c.column_id
    WHERE sc.object_id = OBJECT_ID(N'dbo.Dim_Customer') AND c.[name] = N'Email')
    DROP SENSITIVITY CLASSIFICATION FROM dbo.Dim_Customer.Email;
ADD SENSITIVITY CLASSIFICATION TO dbo.Dim_Customer.Email
    WITH (LABEL = 'Protected A', INFORMATION_TYPE = 'Contact Info', RANK = MEDIUM);

-- Set table-level SensitivityLabel to highest column label (Protected A in this case)
IF EXISTS (SELECT 1 FROM sys.extended_properties
    WHERE major_id = OBJECT_ID(N'dbo.Dim_Customer') AND minor_id = 0
      AND class = 1 AND [name] = N'SensitivityLabel')
    EXEC sys.sp_updateextendedproperty
        @name = N'SensitivityLabel', @value = N'Protected A',
        @level0type = N'SCHEMA', @level0name = N'dbo',
        @level1type = N'TABLE', @level1name = N'Dim_Customer';
ELSE
    EXEC sys.sp_addextendedproperty
        @name = N'SensitivityLabel', @value = N'Protected A',
        @level0type = N'SCHEMA', @level0name = N'dbo',
        @level1type = N'TABLE', @level1name = N'Dim_Customer';
```

---

## Required Permission

```sql
-- Grant permission to apply/modify sensitivity classifications
-- Requires SQL Server 2019+
GRANT ALTER ANY SENSITIVITY CLASSIFICATION TO [DBA_Role];

-- Read permission (included in SELECT on the table by default via DMV)
-- To grant explicit read of sys.sensitivity_classifications:
GRANT VIEW ANY SENSITIVITY CLASSIFICATION TO [Reporting_Role];

-- Check who has the permission
SELECT dp.name, dp.type_desc, perm.permission_name, perm.state_desc
FROM sys.database_permissions perm
JOIN sys.database_principals dp ON perm.grantee_principal_id = dp.principal_id
WHERE perm.permission_name IN (
    'ALTER ANY SENSITIVITY CLASSIFICATION',
    'VIEW ANY SENSITIVITY CLASSIFICATION'
);
```

---

## Column Name Patterns → Suggested Classification

Use this as a starting point when reviewing a new schema. Always confirm with your Privacy Officer.

| Column name contains | Suggested Information Type | Suggested Label |
|---|---|---|
| `PHN`, `HealthNumber`, `MSP` | `Health` | Protected B |
| `SIN`, `SocialInsurance` | `SIN` | Protected B |
| `FirstName`, `LastName`, `FullName`, `Alias` | `Name` | Protected A |
| `DOB`, `DateOfBirth`, `BirthDate`, `BirthYear` | `Date of Birth` | Protected A |
| `Address`, `Street`, `City`, `PostalCode`, `ZipCode` | `Contact Info` | Protected A |
| `Phone`, `Cell`, `Mobile`, `Fax` | `Contact Info` | Protected A |
| `Email`, `EmailAddress` | `Contact Info` | Protected A |
| `IDIR`, `BCeID`, `Username`, `Password` | `Credentials` | Protected B |
| `CreditCard`, `CardNumber`, `CVV`, `ExpiryDate` | `Credit Card` | Protected B |
| `Amount`, `Payment`, `Invoice`, `Tax`, `Salary` | `Financial` | Protected B |
| `IPAddress`, `MACAddress` | `Networking` | Protected A |
| `Passport`, `DriversLicense`, `LicenseNumber` | `Identification` | Protected A |
| `Gender`, `Race`, `Indigeneity`, `Disability`, `Ability` | `Personal` | Protected A |
| `Comment`, `Note`, `Description`, `Memo` | `Free-form Text` | Protected A* |
| `RegionCode`, `ProductCode`, `StatusCode` | `Other` | Public |

> *Free-form text fields may contain any sensitivity level of data entered by users.
> Classify as Protected A minimum and flag for content-scanning review.

---

## References

- [ADD SENSITIVITY CLASSIFICATION (Transact-SQL) — Microsoft Learn](https://learn.microsoft.com/en-us/sql/t-sql/statements/add-sensitivity-classification-transact-sql)
- [sys.sensitivity_classifications (Transact-SQL) — Microsoft Learn](https://learn.microsoft.com/en-us/sql/relational-databases/system-catalog-views/sys-sensitivity-classifications-transact-sql)
- [SQL Data Discovery & Classification — Microsoft Learn](https://learn.microsoft.com/en-us/sql/relational-databases/security/sql-data-discovery-and-classification)
- `extended-properties-templates.md` — extended properties approach (for tables, views, SPs)
- BC Government information classification framework
