# Analysis Services Tabular Best Practices Reference

Targeting: SQL Server Analysis Services (SSAS) Tabular mode, compatibility levels 1200–1600,
on-premises SQL Server 2016–2022, with Power BI Report Server (PBIRS) live connections.

---

## Table of Contents
1. [Model Formats: BIM vs TMDL](#model-formats-bim-vs-tmdl)
2. [Naming Conventions](#naming-conventions)
3. [Compatibility Levels](#compatibility-levels)
4. [Relationship Design](#relationship-design)
5. [Partition Strategy](#partition-strategy)
6. [Column Encoding](#column-encoding)
7. [Calculation Groups](#calculation-groups)
8. [Perspectives](#perspectives)
9. [Row-Level Security (RLS)](#row-level-security-rls)
10. [Kerberos & Windows Authentication](#kerberos--windows-authentication)
11. [SSAS Processing Strategies](#ssas-processing-strategies)
12. [XMLA / TMSL Scripting](#xmla--tmsl-scripting)
13. [VertiPaq Analyzer](#vertipaq-analyzer)
14. [DAX Studio](#dax-studio)
15. [Tabular Editor 2 / 3](#tabular-editor-23)
16. [ALM Toolkit](#alm-toolkit)
17. [Model Deployment](#model-deployment)
18. [Backup and Restore](#backup-and-restore)
19. [Power BI Report Server Live Connection Constraints](#power-bi-report-server-live-connection-constraints)
20. [SSAS Multidimensional Awareness](#ssas-multidimensional-awareness)
21. [DMV Reference Queries](#dmv-reference-queries)
22. [Common Review Findings](#common-review-findings)

---

## Model Formats: BIM vs TMDL

### BIM (Business Intelligence Model)
- Default format: a single `Model.bim` JSON file.
- Supported by all SSAS Tabular compatibility levels 1200+.
- Contains the entire model definition in one file — difficult to diff/merge in source control.
- Required for SSDT (Visual Studio) projects targeting on-premises SSAS.

### TMDL (Tabular Model Definition Language)
- Introduced in Tabular Editor 3 and Microsoft's open-source `microsoft/tml` library.
- Splits the model into folder + per-object files (one `.tmdl` per table, measure, etc.).
- **Vastly superior for source control** — granular diffs, merge-friendly, PR reviews.
- Readable, YAML-like syntax. Example table snippet:

```tmdl
table Sales
    lineageTag: a1b2c3d4-...

    partition Sales-Sales = m
        mode: import
        source =
            let
                Source = Sql.Database("DWSERVER", "DW_Production"),
                dbo_FactSalesOrder = Source{[Schema="dbo",Item="FactSalesOrder"]}[Data]
            in
                dbo_FactSalesOrder

    measure [Total Sales] = SUM(Sales[SalesAmount])
        formatString: "#,##0.00"
        displayFolder: Revenue

    column SalesOrderSK
        dataType: int64
        isHidden: true
        sourceColumn: SalesOrderSK
```

### BIM ↔ TMDL Conversion (Tabular Editor CLI)

```powershell
# Convert BIM to TMDL folder structure (Tabular Editor 3 CLI)
TabularEditor.exe "Model.bim" -S ".\TMDL" -F TMDL

# Convert TMDL back to BIM for deployment to SSAS
TabularEditor.exe ".\TMDL\database.tmdl" -B "Model.bim"
```

---

## Naming Conventions

| Object | Convention | Example |
|---|---|---|
| Tables | PascalCase, singular | `Sales`, `Customer`, `Date` |
| Columns | PascalCase | `SalesAmount`, `OrderDateKey` |
| Hidden columns (keys) | PascalCase + SK suffix | `CustomerSK` (hidden) |
| Measures | Title Case with spaces | `[Total Sales]`, `[YTD Revenue]` |
| Calculated columns | Avoid — use measures | (Only when aggregation is impossible) |
| Display folders | Category\SubCategory | `Revenue\YTD`, `Inventory\Counts` |
| Perspectives | User-facing names | `Sales Analysis`, `Finance View` |
| Roles | `Role_GroupName` or AD group name | `Role_SalesAnalysts` |
| Partitions | `TableName_YYYYMM` or `TableName_Current` | `Sales_202401`, `Sales_Current` |
| Calculation groups | Verb phrase | `Time Intelligence`, `Currency Conversion` |

**Measure naming rules:**
- Never prefix measures with `[M]` or `_` — use Display Folders instead.
- Use consistent verb tense: `[Total ...]`, `[Average ...]`, `[Count of ...]`, `[YTD ...]`.
- Hidden measures (used only in other measures) prefix with `_`: `[_Sales Base]`.

---

## Compatibility Levels

### Feature Matrix

| Feature | CL 1200 | CL 1400 | CL 1500 | CL 1600 |
|---|:---:|:---:|:---:|:---:|
| Tabular JSON metadata (replaces ASSL) | ✓ | ✓ | ✓ | ✓ |
| DAX query performance (batch mode) | ✓ | ✓ | ✓ | ✓ |
| Detail rows expressions | | ✓ | ✓ | ✓ |
| Composite models (limited in SSAS) | | ✓ | ✓ | ✓ |
| M (Power Query) partitions | | ✓ | ✓ | ✓ |
| Many-to-many relationships (native) | | | ✓ | ✓ |
| Calculation groups | | | ✓ | ✓ |
| Object-level security (OLS) | | ✓ | ✓ | ✓ |
| XMLA read/write endpoint | | | | ✓ |
| Query interleaving (priority queues) | | | | ✓ |

### SQL Server Version to Compatibility Level Mapping

| SQL Server Version | Default CL | Max CL Supported |
|---|---|---|
| SQL Server 2016 | 1200 | 1200 |
| SQL Server 2017 | 1400 | 1400 |
| SQL Server 2019 | 1500 | 1500 |
| SQL Server 2022 | 1600 | 1600 |

> **Important:** Upgrading compatibility level is **irreversible** without backup/restore. Test in a non-production instance. PBIRS version must support the CL of the connected SSAS model (see PBIRS constraints section).

```sql
-- Check current compatibility level via DMV
SELECT [compatibility_level] FROM $System.DBSCHEMA_CATALOGS;

-- Or via SSMS connect to SSAS → right-click Database → Properties → Compatibility Level
```

---

## Relationship Design

### Relationship Rules for SSAS Tabular
- Single-direction (one-to-many) is the default and safest.
- Bidirectional cross-filter: **use sparingly** — it causes ambiguous filter paths and can cause exponential query slowdown. Enable only when required for specific DAX measures.
- Many-to-many (CL 1500+): now supported natively without bridge table workarounds.
- All relationships must be active or explicitly activated in DAX with `USERELATIONSHIP()`.

### Role-Playing Dimensions Pattern

```dax
-- OrderDateKey is the active relationship; ShipDateKey is inactive
-- Measure using inactive relationship
Sales by Ship Date =
CALCULATE(
    [Total Sales],
    USERELATIONSHIP(Sales[ShipDateKey], 'Date'[DateKey])
)
```

### Inactive Relationship Setup (SSAS / TMDL)
```tmdl
relationship
    fromTable: Sales
    fromColumn: ShipDateKey
    toTable: Date
    toColumn: DateKey
    isActive: false
    crossFilteringBehavior: oneDirection
```

### Many-to-Many via Bridge Table (CL 1200/1400)

```dax
-- Account to Customer many-to-many via BridgeAccountCustomer
-- Filter propagation: Account → Bridge → Customer
-- Enable bidirectional on the Bridge relationship only
Sales by Account =
CALCULATE(
    [Total Sales],
    TREATAS(
        VALUES(BridgeAccountCustomer[AccountSK]),
        Sales[AccountSK]
    )
)
```

---

## Partition Strategy

### Why Partition?
- Enables incremental refresh: process only new/changed data, not the full table.
- Allows parallelism: multiple partitions can be processed simultaneously (up to available memory).
- Enables "hot/cold" separation: historical partitions stay compressed; only current partition is re-processed nightly.

### Recommended Partition Granularity

| Table Rows | Recommended Partition Grain |
|---|---|
| < 5M | Single partition (no partitioning needed) |
| 5M – 50M | Annual or quarterly |
| 50M – 500M | Monthly |
| > 500M | Monthly + consider aggregation tables |

### Partition Naming Convention
`TableName_YYYY` (annual), `TableName_YYYYQQ` (quarterly), `TableName_YYYYMM` (monthly), `TableName_Current` (rolling window).

### TMDL Partition Snippet (SQL Query–Based)

```tmdl
partition Sales_202401 = sql
    source =
        SELECT * FROM dbo.FactSalesOrder
        WHERE OrderDateKey >= 20240101
          AND OrderDateKey <  20240201
```

### Partition Processing Sequence
Always process **dimensions before facts**. Within facts, historical (read-only) partitions need `ProcessData` only if source changed; current partition needs `ProcessFull`.

---

## Column Encoding

### Encoding Types
| Type | When Applied | Notes |
|---|---|---|
| **Value encoding** | Numeric columns with high cardinality | Stores actual values; most efficient for measures |
| **Hash encoding** | String columns and low-cardinality numerics | Stores dictionary IDs; required for strings |

### Encoding Hints (Tabular Editor)
Force value encoding on frequently aggregated numeric columns to improve compression and query speed:

```tmdl
column SalesAmount
    dataType: decimal
    encodingHint: value    -- force value encoding; default for numerics but explicit is safer
```

### Column Cardinality Guidelines
- **Hide all key (SK) integer columns** — they bloat the dictionary without user value.
- Drop columns that are never used in reports or measures (reduce model size).
- For string columns with > 1M distinct values (e.g., transaction IDs): consider whether they belong in the model at all, or use as a drill-through detail column only.

```dax
-- Find high-cardinality columns via DAX Studio
EVALUATE
ADDCOLUMNS(
    INFO.COLUMNS(),
    "Cardinality", CALCULATE(COUNTROWS(VALUES(INFO.COLUMNS()[ExplicitName])))
)
ORDER BY [Cardinality] DESC
```

---

## Calculation Groups

Calculation groups (CL 1500+) eliminate duplicate time-intelligence measure proliferation.

### When to Use
- Time intelligence: YTD, QTD, MTD, PY, PY YTD, rolling 12-month.
- Currency conversion: multiple exchange rates applied to the same base measures.
- Scenario analysis: Actual vs. Budget vs. Forecast switching.

### TMDL Snippet — Time Intelligence Calculation Group

```tmdl
calculationGroup Time Intelligence
    precedence: 10

    calculationItem YTD =
        CALCULATE(SELECTEDMEASURE(), DATESYTD('Date'[FullDate]))

    calculationItem QTD =
        CALCULATE(SELECTEDMEASURE(), DATESQTD('Date'[FullDate]))

    calculationItem MTD =
        CALCULATE(SELECTEDMEASURE(), DATESMTD('Date'[FullDate]))

    calculationItem Prior Year =
        CALCULATE(SELECTEDMEASURE(), SAMEPERIODLASTYEAR('Date'[FullDate]))

    calculationItem YOY Change =
        VAR CurrentPeriod = SELECTEDMEASURE()
        VAR PriorYear =
            CALCULATE(SELECTEDMEASURE(), SAMEPERIODLASTYEAR('Date'[FullDate]))
        RETURN
            CurrentPeriod - PriorYear

    calculationItem YOY % Change =
        VAR CurrentPeriod = SELECTEDMEASURE()
        VAR PriorYear =
            CALCULATE(SELECTEDMEASURE(), SAMEPERIODLASTYEAR('Date'[FullDate]))
        RETURN
            DIVIDE(CurrentPeriod - PriorYear, ABS(PriorYear))
```

### Calculation Group Best Practices
- Set `Precedence` deliberately when multiple calculation groups interact — higher precedence evaluates first.
- Add a "No Calculation" item (returns `SELECTEDMEASURE()`) so users can clear the selection.
- Test with `SELECTEDMEASURENAME()` in `ISSELECTEDMEASURE()` guards if some measures should be excluded from a calculation item.

---

## Perspectives

Perspectives reduce model complexity for targeted user groups. They are **not a security feature** — all data remains accessible; perspectives only filter the visible object list.

```tmdl
perspective Sales View
    table Sales
        column CustomerSK: hidden
        column ProductSK:  hidden
        column OrderDateKey: hidden
    table Customer
    table Product
    table Date
    measure [Total Sales]
    measure [YTD Sales]
```

**Best practice:** Create one perspective per major subject area (Sales, Finance, HR). Hide all surrogate key columns globally (not just per perspective) unless needed for Power BI relationship mapping.

---

## Row-Level Security (RLS)

### RLS Role Design

```tmdl
role SalesRegionRole
    modelPermission: read

    tablePermission Sales =
        [SalesRegionCode] IN
            CALCULATETABLE(
                VALUES(SalesRegionMapping[RegionCode]),
                SalesRegionMapping[UserEmail] = USERPRINCIPALNAME()
            )
```

### Dynamic RLS Pattern (Active Directory)

```dax
-- Dynamic RLS using AD group membership mapping table
-- DimSecurityUserRegion: UserEmail (VARCHAR), RegionCode (VARCHAR)
[SalesRegionCode] IN
    CALCULATETABLE(
        VALUES(DimSecurityUserRegion[RegionCode]),
        DimSecurityUserRegion[UserEmail] = USERPRINCIPALNAME()
    )
```

### RLS Testing

```dax
-- Test RLS as a specific user in SSMS / DAX Studio (SSAS on-prem)
-- Connect to SSAS with admin rights, then:
EVALUATE
CALCULATETABLE(
    SUMMARIZE(Sales, Sales[SalesRegionCode], "Count", COUNTROWS(Sales)),
    CUSTOMDATA("domain\\testuser")
)
-- Or use SSMS: right-click role → Process Security
```

### Object-Level Security (OLS) — CL 1400+

```tmdl
role FinanceRole
    modelPermission: read

    tablePermission Payroll =
        none    -- completely hides the table from this role
```

- `none` = table or column is invisible and inaccessible.
- `read` = default; visible.
- Applying OLS to a column hides it from the model browser but still allows aggregation if used in a measure. Hide the measure too if the underlying column is sensitive.

---

## Kerberos & Windows Authentication

On-premises SSAS Tabular uses **Windows Authentication exclusively**. No service account passwords in connection strings.

### Authentication Flow: PBIRS → SSAS

```
Browser/Client
    │  (NTLM or Kerberos)
    ▼
Power BI Report Server (PBIRS)
    │  (Kerberos double-hop: PBIRS impersonates report viewer)
    ▼
SSAS Tabular (Windows Auth)
    │  (Connection under report viewer's identity → RLS evaluated)
    ▼
Row-Level Security enforced based on USERPRINCIPALNAME()
```

### Kerberos Double-Hop Requirements

1. **PBIRS Service Account SPN** — the PBIRS service account needs SPNs registered:
   ```
   setspn -S HTTP/pbirs.contoso.com CONTOSO\svc-pbirs
   setspn -S HTTP/pbirs         CONTOSO\svc-pbirs
   ```

2. **SSAS Service Account SPN:**
   ```
   setspn -S MSOLAPSvc.3/ssasserver.contoso.com CONTOSO\svc-ssas
   setspn -S MSOLAPSvc.3/ssasserver             CONTOSO\svc-ssas
   ```

3. **PBIRS Service Account must be trusted for delegation:**
   - Active Directory → PBIRS service account → Properties → Delegation tab.
   - Select: **"Trust this user for delegation to specified services only"** (Constrained Delegation).
   - Add the SSAS SPN (`MSOLAPSvc.3/ssasserver.contoso.com`).
   - Use **"Use any authentication protocol"** (Protocol Transition) if NTLM-to-Kerberos transition is needed.

4. **Verify Kerberos ticket** (run on PBIRS server under PBIRS service account):
   ```powershell
   klist tickets
   # Look for MSOLAPSvc.3/ssasserver.contoso.com in the ticket list
   ```

5. **Connection string on PBIRS data source:**
   ```
   Data Source=ssasserver.contoso.com;Initial Catalog=SalesModel;
   Integrated Security=SSPI;
   ```

### AD Group-Based SSAS Role Membership

```
SSAS Role: SalesAnalysts
  Members:
    CONTOSO\GRP-DW-SalesAnalysts   ← AD security group
    CONTOSO\GRP-DW-Managers        ← AD security group
  (Do NOT add individual user accounts — manage access via AD)
```

**Add AD group to SSAS role via SSMS:**
- SSAS → Databases → Model → Roles → [Role Name] → Membership → Add.
- Enter AD group name; SSAS verifies group existence against AD.

---

## SSAS Processing Strategies

### Processing Modes

| Mode | What It Does | When to Use |
|---|---|---|
| `ProcessFull` | Drop + reload all data + recalculate | Initial load; after schema change |
| `ProcessData` | Reload data without recalculating hierarchies | Partition data refresh (followed by ProcessRecalc) |
| `ProcessRecalc` | Recalculate hierarchies + relationships only | After ProcessData to complete refresh |
| `ProcessAdd` | Append new rows (no recalc) | Rarely used; very specific incremental scenarios |
| `ProcessDefrag` | Defragment VertiPaq storage | Monthly maintenance; reduces memory fragmentation |
| `ProcessClear` | Remove all data (keep structure) | Before full reload in scripted pipelines |

### Processing Sequence (Dimensions Before Facts)

Always process in this order to avoid "relationship integrity" errors during processing:
1. `ProcessFull` on all Dimension tables (or partitions).
2. `ProcessFull` / `ProcessData` on Fact table partitions.
3. `ProcessRecalc` on the database (or let ProcessFull handle it).

### XMLA-Based Incremental Processing Script (SQL Server Agent)

```xml
<!-- XMLA Refresh command — save as .xmla file, execute via SQL Agent SSAS step -->
<Batch xmlns="http://schemas.microsoft.com/analysisservices/2003/engine">
  <Parallel>
    <!-- Process all dimension tables first -->
    <Process>
      <Object>
        <DatabaseID>SalesModel</DatabaseID>
        <TableID>Customer</TableID>
      </Object>
      <Type>ProcessFull</Type>
    </Process>
    <Process>
      <Object>
        <DatabaseID>SalesModel</DatabaseID>
        <TableID>Product</TableID>
      </Object>
      <Type>ProcessFull</Type>
    </Process>
  </Parallel>
</Batch>
```

### TMSL Refresh Command (JSON — Preferred for CL 1200+)

```json
{
  "refresh": {
    "type": "full",
    "objects": [
      { "database": "SalesModel", "table": "Customer" },
      { "database": "SalesModel", "table": "Product" }
    ]
  }
}
```

```json
{
  "refresh": {
    "type": "dataOnly",
    "objects": [
      {
        "database": "SalesModel",
        "table": "Sales",
        "partition": "Sales_202401"
      }
    ]
  }
}
```

### AMO/TOM C# Processing Script

```csharp
// NuGet: Microsoft.AnalysisServices.retail.amd64 (AMO)
// or    Microsoft.AnalysisServices.Tabular.retail.amd64 (TOM — for Tabular)
using Microsoft.AnalysisServices.Tabular;

var connectionString = "Data Source=ssasserver.contoso.com;Integrated Security=SSPI;";
using var server = new Server();
server.Connect(connectionString);

var db = server.Databases["SalesModel"];

// Process a single partition
var table     = db.Model.Tables["Sales"];
var partition = table.Partitions["Sales_202401"];

partition.RequestRefresh(RefreshType.Full);
db.Model.SaveChanges();

Console.WriteLine("Processing complete.");
server.Disconnect();
```

### SQL Server Agent Job Pattern for Nightly Processing

```sql
-- SQL Server Agent Job Step: SSAS Command
-- Step Type: SQL Server Analysis Services Command
-- Server: ssasserver.contoso.com
-- Command (TMSL):
{
  "sequence": {
    "operations": [
      {
        "refresh": {
          "type": "full",
          "objects": [
            { "database": "SalesModel", "table": "Date" },
            { "database": "SalesModel", "table": "Customer" },
            { "database": "SalesModel", "table": "Product" }
          ]
        }
      },
      {
        "refresh": {
          "type": "full",
          "objects": [
            { "database": "SalesModel", "table": "Sales", "partition": "Sales_Current" }
          ]
        }
      }
    ]
  }
}
```

> **Tip:** Use `"sequence"` (not `"parallel"`) when dimensions must be processed before facts. Use `"parallel"` only within a group of tables that have no dependency on each other.

### Processing Error Handling

```powershell
# PowerShell: invoke SSAS processing and capture errors
$server = New-Object Microsoft.AnalysisServices.Server
$server.Connect("Data Source=ssasserver.contoso.com;Integrated Security=SSPI;")

$db = $server.Databases["SalesModel"]
$db.Process([Microsoft.AnalysisServices.ProcessType]::ProcessFull)

if ($server.Databases["SalesModel"].State -ne "Processed") {
    Write-Error "SSAS processing failed. Check SSAS logs at: C:\Program Files\Microsoft SQL Server\MSAS??...\OLAP\Log\msmdsrv.log"
    exit 1
}
$server.Disconnect()
```

---

## XMLA / TMSL Scripting

### XMLA Endpoint Connection (SSMS)
Connect SSMS to SSAS using the XMLA endpoint:
- **Server type:** Analysis Services
- **Server name:** `ssasserver.contoso.com` (or named instance: `ssasserver\TABULAR`)

### Common TMSL Commands

#### Get Database List
```xml
<Discover xmlns="urn:schemas-microsoft-com:xml-analysis">
  <RequestType>DBSCHEMA_CATALOGS</RequestType>
  <Restrictions />
  <Properties />
</Discover>
```

#### ProcessRecalc Only (After ProcessData)
```json
{
  "refresh": {
    "type": "calculate",
    "objects": [
      { "database": "SalesModel" }
    ]
  }
}
```

#### Synchronize Database (DR / Scale-Out)
```xml
<Synchronize xmlns="http://schemas.microsoft.com/analysisservices/2003/engine">
  <Object>
    <DatabaseID>SalesModel</DatabaseID>
  </Object>
  <SynchronizeSecurity>CopyAll</SynchronizeSecurity>
  <ApplyCompression>true</ApplyCompression>
  <Source>
    <ConnectionString>Data Source=ssasserver-primary.contoso.com;Integrated Security=SSPI;</ConnectionString>
    <Object>
      <DatabaseID>SalesModel</DatabaseID>
    </Object>
  </Source>
</Synchronize>
```

#### Detach and Attach (Migration)
```xml
<Detach xmlns="http://schemas.microsoft.com/analysisservices/2003/engine">
  <Object><DatabaseID>SalesModel</DatabaseID></Object>
  <Password>OptionalEncryptionPassword</Password>
</Detach>
```

---

## VertiPaq Analyzer

VertiPaq Analyzer (embedded in DAX Studio) inspects the in-memory VertiPaq storage engine to diagnose model size, cardinality, and compression issues.

### Launching VertiPaq Analyzer
1. Open DAX Studio → Connect to SSAS Tabular instance.
2. Click **VertiPaq Analyzer** in the ribbon (or View menu).
3. Click **Refresh** to load current model statistics.

### Key Metrics to Review

| Metric | What It Means | Action |
|---|---|---|
| **Table Size (MB)** | Memory consumed by the table | Large tables → review partitioning, drop unused columns |
| **Column Cardinality** | Distinct value count per column | High cardinality strings → consider hiding or removing |
| **Column Size (MB)** | Memory per column | Largest columns are optimization targets |
| **Dictionary Size** | Memory for hash-encoded value dictionary | High dictionary = high cardinality string columns |
| **Encoding** | Value or Hash | Numeric measures should be Value-encoded |
| **Segments** | Number of VertiPaq segments | Many segments with few rows = suboptimal partition sizes |

### VertiPaq DMV Queries (via DAX Studio or SSMS XMLA)

```dax
-- Table sizes
SELECT
    [DIMENSION_NAME]  AS TableName,
    [TABLE_ROWS_COUNT] AS Rows,
    [TABLE_STORAGE_SIZE] / 1048576.0 AS TableSizeMB
FROM $System.DISCOVER_STORAGE_TABLES
WHERE [TABLE_ID] NOT LIKE 'R$*'  -- exclude relationship tables
ORDER BY [TABLE_STORAGE_SIZE] DESC;
```

```dax
-- Column sizes and cardinality (identify top memory consumers)
SELECT
    [TABLE_NAME],
    [COLUMN_HIERARCHY_NAME] AS ColumnName,
    [COLUMN_ENCODING]       AS Encoding,
    [DICTIONARY_SIZE] / 1048576.0         AS DictionarySizeMB,
    [USED_SIZE] / 1048576.0               AS TotalColumnSizeMB,
    [TABLE_CARDINALITY]                   AS ColumnCardinality
FROM $System.DISCOVER_STORAGE_TABLE_COLUMNS
WHERE [COLUMN_TYPE] = 'BASIC_DATA'
ORDER BY [USED_SIZE] DESC;
```

### Model Size Reduction Checklist
- [ ] Remove columns not used in any measure, relationship, or visible field.
- [ ] Hide all surrogate key (SK) columns — they inflate dictionary size without benefit.
- [ ] Replace high-cardinality string columns with integer codes where possible (resolve in DW, keep label).
- [ ] Verify date columns are `INT` (DateKey) not `DATETIME` — DATETIME doubles storage.
- [ ] Check for accidental column duplicates (same data under two names).
- [ ] Use `DISCOVER_STORAGE_TABLES` and `DISCOVER_STORAGE_TABLE_COLUMNS` DMVs monthly.

---

## DAX Studio

DAX Studio is the primary query, profiling, and diagnostic tool for on-premises SSAS Tabular.

### Key Features for On-Premises SSAS

| Feature | How to Access | Use Case |
|---|---|---|
| **Server Timings** | View → Server Timings | Measure execution time breakdown (SE vs FE) |
| **Query Plan** | View → Query Plan | Logical and physical DAX query plan |
| **VertiPaq Analyzer** | Advanced → VertiPaq Analyzer | Model memory analysis |
| **Profiler Trace** | Advanced → Profiler | Capture SSAS trace events (like SQL Profiler) |
| **DMV Queries** | New Query → run `$System.*` queries | Metadata and statistics |
| **Format DAX** | Edit → Format DAX | Auto-format DAX code |
| **Export Data** | Output → File/Clipboard | Export DAX query results |

### Server Timings Interpretation

| Metric | What It Means |
|---|---|
| **SE CPU** | Storage Engine time — reading VertiPaq data; should be bulk of time |
| **FE CPU** | Formula Engine time — interpreting DAX; high FE = complex DAX to optimize |
| **SE Queries** | Number of VertiPaq queries generated; high count = complex filter context |
| **SE Cache** | Whether the SE result was served from cache |

**Rule of thumb:** FE > 20% of total query time = DAX optimization opportunity. Use `SUMMARIZE` → `ADDCOLUMNS` pattern instead of nested `CALCULATE` chains.

### Query Plan Analysis

```dax
-- Enable query plan in DAX Studio (checkbox), then run a measure:
DEFINE
    MEASURE Sales[Slow Measure] =
        SUMX(
            FILTER(ALL(Sales), Sales[CustomerSK] IN VALUES(Customer[CustomerSK])),
            Sales[SalesAmount]
        )
EVALUATE
{ [Slow Measure] }
```

Look for `CallbackDataID` lines in the query plan — these indicate the FE is calling the SE row-by-row (a "callback" pattern), which is a major performance anti-pattern.

### Profiler Trace for SSAS (On-Premises Only)

```
DAX Studio → Advanced → Profiler → Start Trace
Events to capture:
  - Query Begin / End
  - Progress Report Begin/End (for processing)
  - VertiPaq SE Query Begin/End
  - Direct Query Begin/End (if DirectQuery mode)
```

Profiler traces against on-prem SSAS are richer than Power BI Service traces — full SSAS trace event hierarchy is available.

---

## Tabular Editor 2/3

### Overview

| Version | License | Key Use Cases |
|---|---|---|
| **Tabular Editor 2** | Free / Open Source (MIT) | Model editing, BPA, basic scripting, CI/CD pipelines |
| **Tabular Editor 3** | Commercial (paid) | Full IDE, DAX editor with IntelliSense, TMDL, C# scripting, Diagram view |

Both versions connect to on-premises SSAS Tabular instances directly via AMO/TOM.

### Tabular Editor 2 — Connection to SSAS (On-Prem)
```
Server: ssasserver.contoso.com
        ssasserver.contoso.com\TABULAR   (named instance)
Database: SalesModel
Integrated Security: Windows (SSPI)
```

### Best Practice Analyzer (BPA)

Tabular Editor includes a rules engine (BPA) that runs automated model quality checks.

**Key built-in BPA rules for on-prem SSAS:**

| Rule | Issue Caught |
|---|---|
| `AVOID_CALCULATED_COLUMNS_IN_FACT_TABLES` | Calculated columns on large fact tables inflate VertiPaq size |
| `REMOVE_UNNECESSARY_COLUMNS` | Hidden columns never referenced in measures or relationships |
| `RELATIONSHIP_BOTH_DIRECTIONS` | Bidirectional relationships that may cause ambiguity |
| `NUMERIC_MEASURES_NO_FORMAT_STRING` | Measures missing format strings (appear as raw numbers in PBIRS) |
| `DISPLAY_FOLDER_EMPTY_STRING` | Measures with no display folder (clutter for end users) |
| `AVOID_FLOATING_POINT_DATATYPES` | `Double` data type columns — prefer `Decimal` for DW data |

```json
// Custom BPA rule file (rules.json) — add to TE2 settings
[
  {
    "ID":          "DW_SURROGATE_KEYS_HIDDEN",
    "Name":        "Surrogate key columns should be hidden",
    "Category":    "DW Standards",
    "Description": "All columns ending in 'SK' should have IsHidden = true",
    "Severity":    2,
    "Scope":       "Column",
    "Expression":  "Name.EndsWith(\"SK\") && !IsHidden"
  },
  {
    "ID":          "DW_MEASURES_HAVE_FORMAT",
    "Name":        "All measures must have a format string",
    "Category":    "DW Standards",
    "Description": "Measures without format strings display raw numbers",
    "Severity":    2,
    "Scope":       "Measure",
    "Expression":  "string.IsNullOrWhiteSpace(FormatString)"
  }
]
```

### C# Scripting in Tabular Editor 3

TE3 supports C# scripts that automate bulk model operations.

```csharp
// Script: Hide all surrogate key columns (columns ending in "SK")
foreach (var col in Model.AllColumns)
{
    if (col.Name.EndsWith("SK", StringComparison.OrdinalIgnoreCase))
    {
        col.IsHidden = true;
    }
}
Output("Done: hidden " + Model.AllColumns.Where(c => c.IsHidden).Count() + " SK columns.");
```

```csharp
// Script: Add standard audit measure to every table
foreach (var table in Model.Tables)
{
    if (table.Measures.Any(m => m.Name == "Row Count")) continue;

    var m = table.AddMeasure(
        "Row Count",
        $"COUNTROWS('{table.Name}')",
        "_Admin"     // display folder
    );
    m.FormatString = "#,##0";
    m.IsHidden = true;
}
```

### CI/CD Pipeline with Tabular Editor 2 CLI

```powershell
# Build step: validate model with BPA (fail if any high-severity issues)
TabularEditor.exe ".\TMDL\database.tmdl" -A ".\bpa-rules.json" -V

# Deploy step: deploy to SSAS (overwrite existing, keep partitions)
TabularEditor.exe ".\TMDL\database.tmdl" `
    -D "ssasserver.contoso.com" "SalesModel" `
    -O -R -P   # -O: overwrite, -R: retain partitions, -P: deploy roles
```

---

## ALM Toolkit

ALM Toolkit (Christian Wade, Microsoft) is the **schema comparison and deployment tool** for SSAS Tabular. It is the recommended production deployment tool over SSDT for change-controlled deployments.

### ALM Toolkit vs. SSDT Deployment

| Aspect | SSDT Deploy | ALM Toolkit |
|---|---|---|
| Deployment type | Full model overwrite | Delta/diff-based deployment |
| Partition preservation | ❌ Destroys all partitions | ✓ Preserves existing partitions |
| Data preservation | ❌ Requires full reprocess | ✓ Minimizes reprocessing |
| Schema diff visibility | ❌ None (blind deploy) | ✓ Visual diff before applying |
| Selective object deploy | ❌ All or nothing | ✓ Choose which objects to deploy |
| Recommended for production | ❌ High risk | ✓ Yes |

### ALM Toolkit Workflow

1. **Source:** Open `.bim` file or connect to DEV SSAS instance.
2. **Target:** Connect to PROD SSAS instance.
3. **Compare:** ALM Toolkit shows a diff of every object (tables, measures, relationships, roles).
4. **Review:** Uncheck objects you do NOT want to update (e.g., partition definitions managed by ETL).
5. **Update:** Apply changes. ALM Toolkit generates and executes the minimal XMLA/TMSL script.

### ALM Toolkit in CI/CD (Command Line)

```powershell
# ALM Toolkit CLI deployment
# alm-toolkit.exe is the CLI entry point (AlmToolkit.exe /?)
AlmToolkit.exe `
    -s ".\Model.bim" `                            # source: BIM file
    -t "ssasserver.contoso.com\SalesModel" `      # target: SSAS instance\database
    -o deploy `                                   # operation: deploy
    -skipPartitions `                             # preserve existing partitions
    -skipRoles:false                              # deploy role definitions
```

---

## Model Deployment

### Deployment Checklist

- [ ] Run BPA in Tabular Editor and resolve all high-severity issues.
- [ ] Verify compatibility level matches target SSAS instance (do not deploy CL 1500 model to SQL 2017 SSAS).
- [ ] Compare with ALM Toolkit — review diff before deploying.
- [ ] Deploy to DEV → UAT → PROD in sequence; never directly to PROD from local file.
- [ ] After deployment: run smoke tests (see below).
- [ ] After deployment: trigger processing via SQL Server Agent job or TMSL script.
- [ ] Verify PBIRS data sources point to correct SSAS server after any server migration.

### SSDT Deployment (Development Use Only)

```
SSDT (Visual Studio) → Deploy → Deployment Server: ssasserver-dev.contoso.com
Model.bim → Deployment Options:
  - Processing Option: Do Not Process (let ETL/Agent handle processing)
  - Transactional Deployment: Yes
  - Query Mode: In-Memory (not DirectQuery unless explicitly designed)
```

### Smoke Tests Post-Deployment

```dax
-- Run via DAX Studio immediately after deployment
-- Test 1: Row counts are non-zero
EVALUATE
{
    ( "Sales Rows",    COUNTROWS(Sales)    ),
    ( "Customer Rows", COUNTROWS(Customer) ),
    ( "Product Rows",  COUNTROWS(Product)  ),
    ( "Date Rows",     COUNTROWS('Date')   )
}

-- Test 2: Key measure returns a value
EVALUATE
{ [Total Sales] }

-- Test 3: RLS evaluation (as test user)
EVALUATE
CALCULATETABLE(
    { COUNTROWS(Sales) },
    CUSTOMDATA("CONTOSO\\testuser")
)
```

```powershell
# PowerShell smoke test: connect and verify model is queryable
$conn = New-Object Microsoft.AnalysisServices.AdomdClient.AdomdConnection
$conn.ConnectionString = "Data Source=ssasserver.contoso.com;Initial Catalog=SalesModel;Integrated Security=SSPI;"
$conn.Open()
$cmd = $conn.CreateCommand()
$cmd.CommandText = "EVALUATE { COUNTROWS(Sales) }"
$result = $cmd.ExecuteReader()
while ($result.Read()) {
    Write-Host "Sales rows: $($result[0])"
}
$conn.Close()
```

---

## Backup and Restore

### SSAS Database Backup (.abf)

SSAS Tabular databases can be backed up to `.abf` (Analysis Services Backup File) format.

```xml
<!-- XMLA Backup command -->
<Backup xmlns="http://schemas.microsoft.com/analysisservices/2003/engine">
  <Object>
    <DatabaseID>SalesModel</DatabaseID>
  </Object>
  <File>\\backupserver\SSASBackups\SalesModel_20240115.abf</File>
  <AllowOverwrite>true</AllowOverwrite>
  <ApplyCompression>true</ApplyCompression>
  <BackupRemotePartitions>true</BackupRemotePartitions>
  <Password>StrongEncryptionPassword123!</Password>
</Backup>
```

### SSAS Backup via SQL Server Agent

```sql
-- SQL Server Agent Job Step: Analysis Services Command
-- Server: ssasserver.contoso.com
-- Command:
{
  "backup": {
    "database": "SalesModel",
    "file": "\\\\backupserver\\SSASBackups\\SalesModel_{@date}.abf",
    "allowOverwrite": true,
    "applyCompression": true,
    "password": "$(BackupPassword)"
  }
}
```

> **Note:** TMSL `"backup"` command is supported from CL 1200+. Use XMLA for older models.

### Backup Schedule Recommendation

| Backup Type | Frequency | Retention |
|---|---|---|
| Post-processing backup | After nightly ETL completes | 7 days rolling |
| Pre-deployment backup | Before every model deployment | 30 days |
| Weekly full backup | Sunday | 90 days |

### Restore

```xml
<!-- XMLA Restore command -->
<Restore xmlns="http://schemas.microsoft.com/analysisservices/2003/engine">
  <File>\\backupserver\SSASBackups\SalesModel_20240115.abf</File>
  <DatabaseName>SalesModel_Restored</DatabaseName>
  <AllowOverwrite>true</AllowOverwrite>
  <Password>StrongEncryptionPassword123!</Password>
</Restore>
```

### Disaster Recovery Strategy

1. **Backup `.abf` files** to a separate storage target (UNC share, or copy to secondary server).
2. **Source-control the model** (BIM or TMDL) — the model definition can be re-deployed without a backup file if all data can be reloaded from the DW.
3. **Document processing scripts** — SQL Agent jobs that drive processing should themselves be scripted to source control.
4. **Test restore quarterly** — restore `.abf` to a test SSAS instance; verify queries execute correctly.

```powershell
# Automated backup script (run via SQL Server Agent CmdExec step)
$backupPath = "\\backupserver\SSASBackups"
$dateSuffix = Get-Date -Format "yyyyMMdd_HHmm"
$backupFile = "$backupPath\SalesModel_$dateSuffix.abf"

$tmslCommand = @"
{
  "backup": {
    "database": "SalesModel",
    "file": "$backupFile",
    "allowOverwrite": true,
    "applyCompression": true
  }
}
"@

Invoke-ASCmd -Server "ssasserver.contoso.com" -Query $tmslCommand

# Purge backups older than 7 days
Get-ChildItem $backupPath -Filter "SalesModel_*.abf" |
    Where-Object { $_.LastWriteTime -lt (Get-Date).AddDays(-7) } |
    Remove-Item -Force
```

---

## Power BI Report Server Live Connection Constraints

When connecting Power BI Desktop reports (deployed to PBIRS) to an on-premises SSAS Tabular model via **Live Connection**, the following capabilities are **not available**:

### What You CANNOT Do in a PBIRS Live Connection Report

| Capability | Available? | Notes |
|---|---|---|
| Create new measures in the report layer | ❌ No | All measures must exist in the SSAS model |
| Create calculated columns in the report layer | ❌ No | Must be in the SSAS model |
| Create calculated tables | ❌ No | Must be in the SSAS model |
| Composite models (mix live + imported data) | ❌ No | Not supported in PBIRS live connection |
| What-if parameters | ❌ No | Require import mode |
| Q&A visual | ❌ Limited | Only if model has Q&A synonyms defined |
| Incremental refresh (report-level) | ❌ N/A | Processing is SSAS-side only |
| Row-level security (report-level) | ❌ No | RLS must be defined in SSAS roles |
| Custom visuals requiring import data | ❌ Varies | Check visual compatibility with live connections |
| Multiple SSAS models in one report | ❌ No | One live connection source per report |
| DirectQuery to SQL + Live SSAS | ❌ No | Composite models not supported in PBIRS |

### What You CAN Do

| Capability | Notes |
|---|---|
| Use all measures defined in SSAS model | Full DAX measure library available |
| Apply report-level filters | Filter context works normally |
| Use perspectives | Show Perspective dropdown in field list |
| Drillthrough | Supported |
| Bookmarks | Supported |
| Conditional formatting | Supported (on model measures) |

### PBIRS ↔ SSAS Version Compatibility Matrix

| PBIRS Version | Max SSAS CL Supported | Notes |
|---|---|---|
| PBIRS 2017 (Jan 2017) | 1200 | Limited |
| PBIRS 2019 (Jan 2019) | 1400 | M partitions supported |
| PBIRS 2020 (May 2020) | 1500 | Calc groups, M2M |
| PBIRS 2022 (May 2022) | 1500 | Stable for production |
| PBIRS 2023 (Jan 2023) | 1600 | Full CL 1600 support |

> **Rule:** PBIRS version must be **equal to or newer than** the SQL Server version hosting SSAS. Always validate PBIRS release notes when upgrading SSAS compatibility level.

### PBIRS Data Source Configuration

```
Data Source Type: Analysis Services
Connection string: Data Source=ssasserver.contoso.com;Initial Catalog=SalesModel
Authentication: Windows integrated security (Kerberos for live connection with RLS)
Impersonation: Impersonate the Windows user viewing the report (required for RLS)
```

---

## SSAS Multidimensional Awareness

### Tabular vs. Multidimensional — Key Differences

| Dimension | SSAS Multidimensional (MD) | SSAS Tabular |
|---|---|---|
| Query language | MDX (primary), DAX (limited) | DAX (primary), MDX (via compat layer) |
| Storage engine | MOLAP (cubes) / ROLAP / HOLAP | VertiPaq (in-memory columnar) |
| Development tool | SSDT cube designer | SSDT model designer / Tabular Editor |
| Memory model | Disk-backed with cache | Fully in-memory |
| Hierarchies | First-class (attribute, user-defined) | Defined manually; parent-child via DAX |
| Many-to-many | Intermediate measure group pattern | Bridge table or CL 1500 native M2M |
| Actions | Supported | Not supported |
| KPIs | MDX-based KPIs | DAX-based KPIs |
| Perspectives | Supported | Supported |
| Writeback | Supported | Not supported |
| PBIRS compatibility | Limited (older) | Full support |
| Power BI live connection | ❌ Not supported | ✓ Supported |

### MDX vs. DAX

If your organization has existing MDX reports (SSRS, Excel pivot tables) pointing to SSAS MD, they will **not** run against a Tabular model without modification.

| MDX Concept | DAX Equivalent |
|---|---|
| `[Measures].[Sales Amount]` | `[Total Sales]` |
| `[Date].[Calendar].[Year]` | `'Date'[CalendarYear]` |
| `CROSSJOIN` | `CROSSJOIN()` function (limited use) |
| `NON EMPTY` | `FILTER(... , [Measure] <> BLANK())` |
| `WITH MEMBER` | `DEFINE MEASURE` (in DAX Studio) |
| Calculated members | Measures |
| Named sets | N/A — use calculation groups or disconnected tables |

### Migration Path: MD → Tabular

1. **Audit existing MD model:** Catalog all measure groups, dimensions, hierarchies, KPIs, and MDX expressions. Use SSAS MD DMVs.
2. **Map to Tabular equivalents:** Measure groups → tables; attributes → columns; hierarchies → DAX hierarchies or parent-child; MDX calculated members → DAX measures.
3. **Rebuild dimension tables** as Kimball-style wide flat dimensions in the DW (if snowflaked in MD).
4. **Migrate MDX reports to DAX:** SSRS reports using MDX → rewrite in DAX or rebuild in PBIRS Power BI reports.
5. **Run in parallel:** Keep MD model live for existing reports while Tabular model is validated.
6. **Decommission MD** after all consumers migrated.

---

## DMV Reference Queries

Connect via SSMS (Analysis Services connection) and run in an MDX or DAX query window. All `$System.*` DMVs work via DAX Studio as well.

```dax
-- List all tables in the model
SELECT [TableName], [TableCount] AS Rows
FROM $System.TMSCHEMA_TABLES;

-- List all measures with their DAX expressions
SELECT [TableName], [MeasureName], [Expression], [DisplayFolder]
FROM $System.TMSCHEMA_MEASURES
ORDER BY [TableName], [MeasureName];

-- List all columns with data type, encoding, cardinality
SELECT
    [TABLE_NAME],
    [COLUMN_HIERARCHY_NAME],
    [DATA_TYPE],
    [COLUMN_ENCODING],
    [TABLE_CARDINALITY]  AS Cardinality
FROM $System.DISCOVER_STORAGE_TABLE_COLUMNS
WHERE [COLUMN_TYPE] = 'BASIC_DATA'
ORDER BY [TABLE_CARDINALITY] DESC;

-- Active relationships
SELECT
    [FromTableName], [FromColumnName],
    [ToTableName],   [ToColumnName],
    [Active],        [CrossFilteringBehavior]
FROM $System.TMSCHEMA_RELATIONSHIPS;

-- Role members
SELECT [RoleName], [MemberName], [MemberType]
FROM $System.TMSCHEMA_ROLE_MEMBERSHIPS;

-- Current processing state of all partitions
SELECT
    [TableName],
    [PartitionName],
    [State]          AS ProcessingState,
    [ModifiedTime]
FROM $System.TMSCHEMA_PARTITIONS
ORDER BY [TableName], [PartitionName];

-- Memory usage by table (top 20)
SELECT TOP 20
    [DIMENSION_NAME]    AS TableName,
    [TABLE_ROWS_COUNT]  AS Rows,
    [TABLE_STORAGE_SIZE] / 1048576.0 AS TableSizeMB
FROM $System.DISCOVER_STORAGE_TABLES
WHERE [TABLE_ID] NOT LIKE 'R$*'
ORDER BY [TABLE_STORAGE_SIZE] DESC;

-- Server properties (version, CL, memory)
SELECT [PROPERTY_NAME], [PROPERTY_VALUE]
FROM $System.DISCOVER_PROPERTIES
WHERE [PROPERTY_NAME] IN (
    'ProductVersion', 'DefaultCompatibilityLevel',
    'MemoryLimit', 'LowMemoryLimit', 'TotalMemoryUsage'
);
```

---

## Common Review Findings

These are the most frequently encountered issues during SSAS Tabular model reviews in on-premises SQL Server environments.

| # | Finding | Severity | Correction |
|---|---|---|---|
| 1 | Surrogate key (SK) columns visible to users | Medium | Set `IsHidden = true` on all SK columns |
| 2 | Measures without format strings | High | Add `FormatString` to every measure; use `"#,##0.00"` for currency |
| 3 | Calculated columns on large fact tables | High | Replace with DAX measures; calculated columns bloat VertiPaq |
| 4 | Bidirectional relationships on all tables | High | Only enable bidirectional where explicitly required; document reason |
| 5 | No display folders on measures | Medium | Group measures into folders (e.g., `Revenue`, `Costs`, `KPIs`) |
| 6 | RLS not tested before production deploy | Critical | Always test with `CUSTOMDATA()` / `USERPRINCIPALNAME()` before go-live |
| 7 | SSAS not backed up before deployment | Critical | Always take `.abf` backup before deploying model changes |
| 8 | Processing sequence: facts before dimensions | High | Always process dimensions first; use `"sequence"` in TMSL |
| 9 | Duplicate time-intelligence measures per base measure | High | Replace with calculation groups (CL 1500+) |
| 10 | Date table not marked as Date Table in SSAS | High | Right-click table → Mark as Date Table → select DATE column |
| 11 | PBIRS report using new measures not in model | High | PBIRS live connection does not support report-layer measures |
| 12 | No partitioning on large (>10M row) fact tables | High | Add monthly partitions; enables incremental processing |
| 13 | Model deployed directly from SSDT to production | Medium | Use ALM Toolkit for production deployments |
| 14 | Kerberos double-hop not configured for PBIRS | Critical | Configure constrained delegation (see Kerberos section) |
| 15 | Compatibility level not matched to PBIRS version | High | Verify PBIRS supports target CL before upgrading |
| 16 | No BPA rules enforced in CI/CD | Medium | Add Tabular Editor BPA step to deployment pipeline |
| 17 | AMO/TOM version mismatch in processing scripts | Medium | Pin NuGet package to same major version as SSAS instance |
| 18 | Model has no perspectives | Low | Add at least one perspective per subject area |
| 19 | NULL FK values in source DW tables | High | Use `-1` Unknown surrogate key; NULL breaks SSAS relationship integrity |
| 20 | SSAS service account not in local Administrators | Low | SSAS service account needs `Log on as a service` and access to backup paths |
