# Data Warehouse Review Checklist — On-Premises SQL Server / SSAS Tabular

> **Target Stack:** SQL Server 2016–2022 DW (on-prem), SSAS Tabular (on-prem, CL 1200–1600),  
> Power BI Report Server (PBIRS), SSIS ETL, SQL Server Agent, Active Directory / Windows Auth.  
> No Azure. No Power BI Service. No Microsoft Fabric.

---

## Severity Levels

| Level | Label | Meaning |
|---|---|---|
| 🔴 | **CRITICAL** | Data integrity risk, security vulnerability, or production outage risk. Block deployment. |
| 🟡 | **WARNING** | Performance risk, maintainability issue, or deviation from best practice. Address before next release. |
| 🟢 | **INFO** | Observation or recommendation for future improvement. |

---

## Section 1: SQL Server Data Warehouse — Physical Layer

### 1.1 Fact Table Audit

| Check | Severity | Notes |
|---|---|---|
| All fact tables have a surrogate integer primary key | 🟡 | Required for SSAS partition pruning efficiency |
| Foreign keys to all dimensions are defined (at minimum, trusted constraints for VertiPaq) | 🟡 | `WITH NOCHECK NOCHECK` constraints still inform query optimiser |
| Date dimension FK is NOT NULL on all fact tables | 🔴 | NULL date keys break time intelligence |
| No varchar/nvarchar measure columns (should be numeric) | 🔴 | Implicit conversion overhead in SSAS |
| Row count documented and tracked over time | 🟢 | Baseline for anomaly detection |
| Fact table grain documented | 🔴 | Undefined grain leads to incorrect aggregations |
| Degenerate dimensions stored as columns, not separate dim tables | 🟢 | e.g., InvoiceNumber, OrderNumber |
| Late-arriving facts handling documented | 🟡 | Policy for backdated records |

**Query: Fact table row counts and date range**
```sql
SELECT
    OBJECT_NAME(i.object_id)    AS TableName,
    SUM(p.rows)                 AS EstimatedRowCount,
    MIN(create_date)            AS TableCreatedDate
FROM sys.indexes i
JOIN sys.partitions p ON i.object_id = p.object_id AND i.index_id = p.index_id
JOIN sys.tables t ON i.object_id = t.object_id
WHERE t.is_ms_shipped = 0
  AND i.index_id IN (0, 1)  -- heap or clustered
  AND OBJECT_NAME(i.object_id) LIKE 'Fact%'
GROUP BY OBJECT_NAME(i.object_id)
ORDER BY EstimatedRowCount DESC;
```

---

### 1.2 Dimension Table Audit

| Check | Severity | Notes |
|---|---|---|
| All dimensions have surrogate integer primary key | 🔴 | Natural keys are not appropriate for SSAS relationships |
| Unknown/default member row exists (key = -1 or 0) | 🟡 | Handles orphan fact records gracefully |
| SCD Type 2 dimensions have IsCurrent, EffectiveFrom, EffectiveTo columns | 🟡 | Required for historical accuracy |
| Conformed dimensions shared across multiple fact tables (bus matrix) | 🔴 | Siloed dimensions prevent cross-subject-area analysis |
| Dimension attributes have human-readable labels (not just keys) | 🟡 | SSAS model usability |
| DimDate covers at least 5 years before earliest fact date and 3 years after today | 🔴 | Gaps break time intelligence |
| DimDate is marked as Date Table in SSAS | 🔴 | Required for DATESYTD, SAMEPERIODLASTYEAR, etc. |
| Large dimensions (>1M rows) partitioned or reviewed for memory impact | 🟡 | DimCustomer with millions of rows impacts SSAS memory |

**Query: Dimension tables without unknown member row**
```sql
SELECT
    t.name AS TableName,
    SUM(p.rows) AS RowCount
FROM sys.tables t
JOIN sys.partitions p ON t.object_id = p.object_id AND p.index_id IN (0,1)
WHERE t.name LIKE 'Dim%'
  AND t.is_ms_shipped = 0
  AND NOT EXISTS (
      SELECT 1 FROM sys.columns c
      WHERE c.object_id = t.object_id
        AND c.name IN ('IsCurrent','IsActive','IsUnknown')
  )
GROUP BY t.name
ORDER BY t.name;
```

---

### 1.3 Schema Health

| Check | Severity | Notes |
|---|---|---|
| All DW objects in a dedicated schema (`dw`, `stg`, `etl`) — not `dbo` | 🟡 | Separation of concerns |
| Staging tables in `stg` schema, not mixed with presentation layer | 🟡 | Prevents accidental SSAS model access to raw staging data |
| No orphaned tables (not referenced by SSAS model, not used in SSIS, not in any proc) | 🟢 | Technical debt |
| No SELECT * in views or stored procedures feeding SSAS partitions | 🟡 | Column additions silently break partition queries |
| Naming convention consistent (PascalCase, no spaces, no abbreviations) | 🟢 | Readability |
| Views used as SSAS partition sources (not direct table access) — allows schema evolution | 🟡 | Best practice for maintainability |

---

### 1.4 SQL Server Physical DW Performance

| Check | Severity | Notes |
|---|---|---|
| All large fact tables (>10M rows) have a **columnstore index** | 🔴 | Columnstore provides 10–100× compression and scan speed for analytical queries |
| Columnstore index type appropriate: clustered (CCI) for warehouse tables, non-clustered (NCCI) for OLTP-adjacent | 🟡 | CCI preferred for pure DW fact tables with no single-row lookups |
| Partition scheme aligns with ETL load pattern (monthly, daily) | 🟡 | Partition switching enables zero-downtime bulk loads |
| Statistics are up to date on all DW tables | 🔴 | Stale statistics cause query plan regressions |
| Auto-update statistics is ON for the DW database | 🟡 | `ALTER DATABASE [DWName] SET AUTO_UPDATE_STATISTICS ON` |
| Missing index DMV recommendations reviewed and actioned | 🟡 | See query below |
| No table scans on dimension tables >100K rows in SSAS partition queries | 🟡 | Check with Extended Events or Query Store |
| Query Store enabled on DW database (SQL Server 2016+) | 🟢 | Essential for regression detection |
| TempDB has one data file per logical processor core (up to 8) | 🟡 | Reduces allocation contention during large SSIS loads |
| Max degree of parallelism (MAXDOP) set appropriately for DW workload | 🟡 | MAXDOP 0 (all cores) acceptable for a dedicated DW server |

**Query: Missing index recommendations (top 20 by estimated impact)**
```sql
SELECT TOP 20
    ROUND(migs.avg_total_user_cost * migs.avg_user_impact * (migs.user_seeks + migs.user_scans), 0)
                                        AS EstimatedImprovement,
    mid.statement               AS TableName,
    mid.equality_columns,
    mid.inequality_columns,
    mid.included_columns,
    migs.user_seeks,
    migs.user_scans,
    migs.last_user_seek
FROM sys.dm_db_missing_index_groups   mig
JOIN sys.dm_db_missing_index_group_stats migs ON mig.index_group_handle = migs.group_handle
JOIN sys.dm_db_missing_index_details  mid  ON mig.index_handle = mid.index_handle
WHERE mid.database_id = DB_ID()
ORDER BY EstimatedImprovement DESC;
```

**Query: Check columnstore index presence on fact tables**
```sql
SELECT
    t.name          AS TableName,
    i.name          AS IndexName,
    i.type_desc     AS IndexType,
    SUM(p.rows)     AS Rows
FROM sys.tables t
JOIN sys.indexes i ON t.object_id = i.object_id
JOIN sys.partitions p ON i.object_id = p.object_id AND i.index_id = p.index_id
WHERE t.name LIKE 'Fact%'
  AND t.is_ms_shipped = 0
  AND i.type_desc IN ('CLUSTERED COLUMNSTORE', 'NONCLUSTERED COLUMNSTORE')
GROUP BY t.name, i.name, i.type_desc
ORDER BY Rows DESC;

-- Fact tables WITHOUT any columnstore index:
SELECT t.name AS FactTableWithoutColumnstore
FROM sys.tables t
WHERE t.name LIKE 'Fact%'
  AND t.is_ms_shipped = 0
  AND NOT EXISTS (
      SELECT 1 FROM sys.indexes i
      WHERE i.object_id = t.object_id
        AND i.type_desc IN ('CLUSTERED COLUMNSTORE','NONCLUSTERED COLUMNSTORE')
  );
```

**Query: Stale statistics**
```sql
SELECT
    OBJECT_NAME(s.object_id)    AS TableName,
    s.name                      AS StatName,
    sp.last_updated,
    sp.rows,
    sp.rows_sampled,
    sp.modification_counter
FROM sys.stats s
CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) sp
WHERE OBJECT_NAME(s.object_id) LIKE 'Fact%'
  AND sp.modification_counter > 0
ORDER BY sp.modification_counter DESC;
```

**Query: Table partition alignment**
```sql
SELECT
    OBJECT_NAME(p.object_id)    AS TableName,
    p.partition_number,
    r.value                     AS RangeValue,
    p.rows                      AS PartitionRows,
    p.data_compression_desc     AS Compression
FROM sys.partitions p
JOIN sys.indexes i ON p.object_id = i.object_id AND p.index_id = i.index_id
LEFT JOIN sys.partition_schemes ps ON i.data_space_id = ps.data_space_id
LEFT JOIN sys.partition_functions pf ON ps.function_id = pf.function_id
LEFT JOIN sys.partition_range_values r ON pf.function_id = r.function_id
    AND p.partition_number = r.boundary_id + 1
WHERE OBJECT_NAME(p.object_id) LIKE 'Fact%'
  AND i.index_id IN (0, 1, 5)  -- heap, clustered, CCI
ORDER BY OBJECT_NAME(p.object_id), p.partition_number;
```

---

### 1.5 ETL Process Audit

| Check | Severity | Notes |
|---|---|---|
| SSIS package inventory documented (package name, source, target, schedule, owner) | 🔴 | Undocumented packages are a maintenance and DR risk |
| Incremental load strategy defined per source table (CDC / high-watermark / full reload) | 🔴 | Full reloads of large tables are unacceptable in production |
| ETL control/audit table populated for every package run | 🔴 | Required for monitoring, SLA reporting, and troubleshooting |
| Error redirect paths configured on all Data Flow tasks (no silent discards) | 🔴 | Silent row drops lead to undetected data loss |
| Error rows written to a dedicated error table with source, error code, error description | 🟡 | Enables root cause analysis |
| SSIS packages use project-level connection managers (not package-level) | 🟡 | Avoids credential duplication; enables environment-specific overrides |
| SSIS packages deployed to SSIS Catalog (SSISDB) — not file system deployment | 🟡 | File system deployment lacks logging and environment management |
| SSIS Environment variables used for connection strings (not hard-coded) | 🔴 | Hard-coded connection strings are a security and maintainability risk |
| Checkpoints enabled on long-running packages | 🟡 | Enables restart from failure point, not from beginning |
| Load procedures are idempotent (safe to re-run without duplicate data) | 🔴 | Non-idempotent loads cause data corruption on re-run |
| Package execution logging level set to `Basic` or higher in SSISDB | 🟡 | `None` logging makes troubleshooting impossible |
| Row counts logged (rows inserted, updated, rejected) per package run | 🟡 | Required for data quality monitoring |

**Query: SSIS Catalog package inventory**
```sql
SELECT
    f.name          AS FolderName,
    prj.name        AS ProjectName,
    pkg.name        AS PackageName,
    pkg.description,
    pkg.package_format_version,
    prj.last_deployed_time
FROM SSISDB.catalog.packages pkg
JOIN SSISDB.catalog.projects prj ON pkg.project_id = prj.project_id
JOIN SSISDB.catalog.folders  f   ON prj.folder_id  = f.folder_id
ORDER BY f.name, prj.name, pkg.name;
```

**Query: Recent SSIS package execution status**
```sql
SELECT TOP 50
    e.execution_id,
    f.name      AS FolderName,
    prj.name    AS ProjectName,
    pkg.name    AS PackageName,
    e.status,   -- 1=Created 2=Running 3=Cancelled 4=Failed 5=Pending 6=Unexpected 7=Succeeded 8=Stopping 9=Completed
    e.start_time,
    e.end_time,
    DATEDIFF(SECOND, e.start_time, ISNULL(e.end_time, GETDATE())) AS DurationSeconds
FROM SSISDB.catalog.executions e
JOIN SSISDB.catalog.packages pkg ON e.package_name = pkg.name AND e.project_id = pkg.project_id
JOIN SSISDB.catalog.projects prj ON e.project_id = prj.project_id
JOIN SSISDB.catalog.folders  f   ON prj.folder_id = f.folder_id
ORDER BY e.start_time DESC;
```

**Query: SSIS execution error messages**
```sql
SELECT TOP 100
    m.execution_id,
    m.message_time,
    m.package_name,
    m.task_name,
    m.message
FROM SSISDB.catalog.event_messages m
WHERE m.event_name = 'OnError'
ORDER BY m.message_time DESC;
```

---

### 1.6 SQL Server Agent Job Audit

| Check | Severity | Notes |
|---|---|---|
| DW load jobs and SSAS processing jobs are inventoried and documented | 🔴 | Undocumented jobs are a DR risk |
| DW load completes before SSAS processing begins (enforced by job dependency or sequencing) | 🔴 | Processing before load completes causes stale or partial data in the model |
| SSAS processing job has failure alerting (operator email notification) | 🔴 | Silent SSAS processing failures mean reports show stale data without warning |
| DW load job has failure alerting | 🔴 | Silent ETL failures mean DW data goes stale |
| Job schedules documented with SLA (e.g., "DW load must complete by 06:00") | 🟡 | Enables proactive SLA monitoring |
| Jobs use Windows proxy accounts — not SA or sysadmin accounts | 🔴 | Least-privilege security requirement |
| Job failure history retained for at least 30 days | 🟡 | Default SQL Agent history may be too short |
| No jobs scheduled at identical times that could cause resource contention | 🟡 | Stagger SSIS loads vs. index rebuilds vs. SSAS processing |
| Job step failure action is defined (quit with failure, go to next step, or retry) | 🟡 | Default "quit with success" on failure is dangerous |
| Max concurrent SSAS processing jobs is 1 (sequential processing) | 🟡 | Parallel SSAS processing can exhaust memory on on-prem servers |

**Query: SQL Agent job schedule and last run status**
```sql
SELECT
    j.name                  AS JobName,
    j.enabled,
    CASE j.last_run_outcome
        WHEN 0 THEN 'Failed'
        WHEN 1 THEN 'Succeeded'
        WHEN 3 THEN 'Cancelled'
        ELSE 'Unknown'
    END                     AS LastRunOutcome,
    msdb.dbo.agent_datetime(j.last_run_date, j.last_run_time) AS LastRunDateTime,
    j.last_run_duration,    -- HHMMSS format
    s.name                  AS ScheduleName,
    s.freq_type,
    s.active_start_time
FROM msdb.dbo.sysjobs j
LEFT JOIN msdb.dbo.sysjobschedules js ON j.job_id = js.job_id
LEFT JOIN msdb.dbo.sysschedules s    ON js.schedule_id = s.schedule_id
WHERE j.enabled = 1
ORDER BY j.name;
```

**Query: Jobs that have failed in the last 7 days**
```sql
SELECT
    j.name          AS JobName,
    jh.run_date,
    jh.run_time,
    jh.run_duration,
    jh.message
FROM msdb.dbo.sysjobhistory jh
JOIN msdb.dbo.sysjobs j ON jh.job_id = j.job_id
WHERE jh.run_status = 0  -- 0 = Failed
  AND msdb.dbo.agent_datetime(jh.run_date, jh.run_time) >= DATEADD(DAY, -7, GETDATE())
ORDER BY msdb.dbo.agent_datetime(jh.run_date, jh.run_time) DESC;
```

**Query: Jobs with no failure notification operator**
```sql
SELECT
    j.name          AS JobName,
    j.notify_level_email,
    j.notify_email_operator_id,
    op.name         AS OperatorName
FROM msdb.dbo.sysjobs j
LEFT JOIN msdb.dbo.sysoperators op ON j.notify_email_operator_id = op.id
WHERE j.enabled = 1
  AND (j.notify_level_email = 0 OR j.notify_email_operator_id IS NULL)
ORDER BY j.name;
```

---

## Section 2: Dimension Table Audit (Extended)

### 2.1 DimDate Audit

| Check | Severity | Notes |
|---|---|---|
| DateKey is an INT in YYYYMMDD format (not a date/datetime) | 🟡 | Integer keys are more compact and faster in VertiPaq |
| Covers min(FactTable date) - 2 years to GETDATE() + 3 years | 🔴 | Missing dates break time intelligence functions |
| FiscalYear/FiscalQuarter/FiscalMonth populated if org uses non-calendar fiscal year | 🟡 | Required for fiscal time intelligence |
| IsWeekend, IsHoliday, IsBusinessDay flags populated | 🟢 | Enables business-day calculations |
| Marked as Date Table in SSAS model | 🔴 | Must be set for standard time intelligence to function |

---

## Section 3: SSAS Tabular Model Audit

### 3.1 Model Metadata

| Check | Severity | Notes |
|---|---|---|
| Compatibility level documented and appropriate for SQL Server version | 🔴 | CL 1200+ for SQL Server 2016, CL 1500+ for 2019, CL 1600 for 2022 |
| Model name, version, and owner documented | 🟡 | |
| Source partition queries do not use SELECT * | 🔴 | Column additions break model silently |
| All imported tables have meaningful names (not default query names) | 🟢 | |
| Model does not import unnecessary columns | 🟡 | Each imported column consumes VertiPaq memory |

**Query: SSAS model compatibility level (run against SSAS via DMV)**
```dax
-- In DAX Studio, connect to SSAS instance and run:
SELECT [SERVER_NAME], [PRODUCT_NAME], [PRODUCT_VERSION]
FROM $SYSTEM.DISCOVER_PROPERTIES
WHERE PROPERTYNAME = 'ProductVersion'
```

```sql
-- In SSMS, check SSAS model compatibility level via XML/A:
-- Or check deployment script header for CompatibilityLevel value
```

---

### 3.2 Relationship Audit

| Check | Severity | Notes |
|---|---|---|
| All active relationships use integer key columns | 🟡 | String key relationships have higher VertiPaq overhead |
| No relationships with cross-filter direction = Both unless explicitly required and documented | 🟡 | Bidirectional relationships can cause ambiguous filter paths |
| All inactive relationships documented with reason for inactivity | 🟢 | Role-playing dimensions should be explicitly noted |
| No circular relationship chains | 🔴 | SSAS will refuse to deploy the model |
| Fact-to-fact relationships avoided — use bridge tables instead | 🔴 | Direct fact-to-fact relationships cause double-counting |

---

### 3.3 Measure Audit

| Check | Severity | Notes |
|---|---|---|
| All measures use DIVIDE() not `/` operator | 🔴 | Division by zero will error in reports |
| All measures have Format String set (not "Auto") | 🟡 | Inconsistent display in reports |
| All measures have Description populated | 🟢 | Required for self-service BI adoption |
| All measures organised in Display Folders | 🟡 | |
| No measure uses implicit aggregation as its only definition | 🟡 | `SUM(Col)` measures should be explicit |
| Time intelligence measures tested at multiple date grains | 🔴 | |

---

### 3.4 RLS Audit

| Check | Severity | Notes |
|---|---|---|
| At least one SSAS role defined (not relying on SSAS admin access only) | 🔴 | Without roles, all connected users see all data |
| Dynamic RLS uses USERNAME() — not USERPRINCIPALNAME() | 🔴 | USERPRINCIPALNAME() is unreliable on on-prem SSAS |
| RLS tested with impersonation for representative users | 🔴 | |
| No "admin backdoor" role with no row filter that is visible to non-admins | 🔴 | |
| AD group membership used for role assignment (not individual user names) | 🟡 | Individual usernames require model redeployment to add/remove users |
| Column-level security (hiding sensitive columns per role) reviewed | 🟡 | |

**Query: SSAS role members (via SSMS XML/A or AMO)**
```powershell
# PowerShell — enumerate SSAS roles and members using AMO
[System.Reflection.Assembly]::LoadWithPartialName("Microsoft.AnalysisServices") | Out-Null
$server = New-Object Microsoft.AnalysisServices.Server
$server.Connect("Data Source=SSAS_SERVER_NAME")
$db = $server.Databases["ModelName"]

foreach ($role in $db.Roles) {
    Write-Host "Role: $($role.Name)  Permissions: $($role.ModelPermission)"
    foreach ($member in $role.Members) {
        Write-Host "  Member: $($member.Name)"
    }
}
$server.Disconnect()
```

---

### 3.5 Model Memory and Performance

| Check | Severity | Notes |
|---|---|---|
| Total model memory within 70% of SSAS max server memory setting | 🔴 | Exceeding memory limit causes processing failures or query errors |
| SSAS max server memory configured (not left at default unlimited) | 🔴 | Unlimited SSAS memory will starve the OS and SQL Server on a shared server |
| Largest table memory usage identified via DMV | 🟡 | See query below |
| Unused calculated columns removed | 🟡 | Calculated columns are materialised in memory at process time |
| Query cache hit rate acceptable (>80%) for known queries | 🟢 | Low cache hits indicate high query diversity or insufficient memory |

**Query: VertiPaq memory usage by table and column**
```dax
-- Run in DAX Studio against the SSAS instance (switch to DMV mode)
SELECT
    DIMENSION_NAME          AS TableName,
    ATTRIBUTE_NAME          AS ColumnName,
    DICTIONARY_SIZE         AS DictionaryBytes,
    USED_SIZE               AS UsedBytes,
    ROWS_COUNT
FROM $SYSTEM.DISCOVER_STORAGE_TABLE_COLUMN_SEGMENTS
ORDER BY USED_SIZE DESC
```

---

### 3.6 SSAS Processing Audit

| Check | Severity | Notes |
|---|---|---|
| Processing mode documented per object (Full / ProcessAdd / Defrag) | 🔴 | Wrong processing mode can trigger full reprocessing unexpectedly, blocking reports |
| Dimension processing always precedes fact/measure group processing | 🔴 | Processing facts before dimensions causes orphan key errors |
| Processing time SLA defined and measured | 🟡 | E.g., "Full model process must complete within 90 minutes" |
| Partition-level incremental processing used for large fact tables (>50M rows) | 🟡 | Full processing of huge fact tables is time-prohibitive; partition process only the new partition |
| Processing uses `ProcessAdd` or `ProcessUpdate` for fact partitions where possible | 🟡 | `ProcessFull` is required after schema changes; `ProcessAdd` for new data only |
| Processing job includes a `Recalc` step after partition loads | 🔴 | Without Recalc, relationship indexes and calc columns are stale |
| Processing job failure leaves model in last successful state (not partially processed) | 🔴 | Partial processing can corrupt the model metadata |
| SSAS processing does not run during business hours | 🟡 | Processing locks objects and can degrade query response time |
| Processing command logged (start time, end time, objects processed, success/failure) | 🟡 | Required for SLA monitoring |

**Query: SSAS partition processing history (via Windows Event Log or DMV)**
```dax
-- DAX Studio DMV: last processing date per partition
SELECT
    DATABASE_NAME,
    CUBE_NAME           AS ModelName,
    MEASURE_GROUP_NAME  AS TableName,
    PARTITION_NAME,
    LAST_DATA_UPDATE    AS LastProcessed,
    STATE               AS PartitionState  -- 0=Processed, 1=Unprocessed, etc.
FROM $SYSTEM.DISCOVER_PARTITION_STAT
ORDER BY LAST_DATA_UPDATE DESC
```

**PowerShell: Check SSAS processing state of all partitions**
```powershell
[System.Reflection.Assembly]::LoadWithPartialName("Microsoft.AnalysisServices") | Out-Null
$server = New-Object Microsoft.AnalysisServices.Server
$server.Connect("Data Source=SSAS_SERVER_NAME")
$db = $server.Databases["ModelName"]

foreach ($table in $db.Model.Tables) {
    foreach ($partition in $table.Partitions) {
        Write-Host "$($table.Name) | $($partition.Name) | State: $($partition.State) | Modified: $($partition.ModifiedTime)"
    }
}
$server.Disconnect()
```

---

### 3.7 PBIRS Live Connection Audit

| Check | Severity | Notes |
|---|---|---|
| PBIRS version documented and compatibility with SSAS version verified | 🔴 | Version mismatch can cause connection failures or feature gaps |
| .pbix files use a compatibility level supported by the PBIRS version | 🔴 | Newer .pbix compatibility levels will fail to open on older PBIRS |
| No report-level DAX measures in live connection .pbix files | 🔴 | Report-level measures are not supported in live connection mode |
| No calculated tables or calculated columns added in Power Query on live connection reports | 🔴 | Not supported in live connection mode; will cause open errors |
| No composite model features used in live connection reports | 🔴 | Composite models (mixing DirectQuery + import) not supported in PBIRS live connection |
| No AI Insights visuals or Q&A visuals used if PBIRS version predates their support | 🟡 | Check PBIRS release notes for supported visual versions |
| PBIRS service account has Read permission on SSAS model role | 🔴 | Service account must be a member of at least one SSAS model role |
| Kerberos delegation configured if PBIRS and SSAS are on different servers | 🔴 | Without Kerberos, double-hop fails and users see authentication errors |
| .pbix file version control in place (not just on the PBIRS server) | 🟡 | PBIRS has no built-in version history for .pbix files |
| Report RLS tested end-to-end: user in PBIRS → SSAS role resolved via Kerberos | 🔴 | RLS only works if user identity is successfully delegated to SSAS |

**Query: PBIRS version check via REST API**
```powershell
# Check PBIRS server version
$PBIRSBase = "http://PBIRS_SERVER/ReportServer"
$resp = Invoke-WebRequest "$PBIRSBase/api/v2.0/System" -UseDefaultCredentials
($resp.Content | ConvertFrom-Json).ProductVersion
```

**Compatibility matrix: PBIRS release → max .pbix compatibility level**
| PBIRS Release | Max .pbix Compat Level | Supported SSAS Version |
|---|---|---|
| Jan 2020 | 1460 | SSAS 2012–2019 |
| May 2021 | 1500 | SSAS 2012–2019 |
| May 2022 | 1520 | SSAS 2012–2022 |
| Jan 2023 | 1540 | SSAS 2012–2022 |
| May 2024 | 1567 | SSAS 2016–2022 |

> Always verify against the official PBIRS release notes. Compatibility levels increment with Power BI Desktop releases.

---

## Section 4: DAX Measure Pattern Review

| Check | Severity | Notes |
|---|---|---|
| No FILTER(AllTable, [Measure] > x) patterns | 🟡 | Causes FE overhead; replace with column filter |
| All time intelligence measures use the marked Date Table | 🔴 | |
| USERELATIONSHIP used correctly for role-playing dimensions | 🟡 | Inactive relationships exist for all date roles used |
| Calculation Groups (if present) have Precedence set and documented | 🟡 | |
| Dynamic RLS measures tested under impersonation | 🔴 | |
| No FORMAT() function inside measure body returning text | 🟡 | Use Format String property instead |
| DIVIDE() used exclusively — no `/` division operator | 🔴 | |

---

## Section 5: Security Review

### 5.1 Active Directory Group Assignments in SSAS Roles

| Check | Severity | Notes |
|---|---|---|
| SSAS roles are assigned AD groups — not individual user accounts | 🟡 | Individual accounts require model changes; AD groups are self-service managed |
| AD group names documented and linked to business function | 🟡 | e.g., `DW_SSAS_FinanceReadOnly` → Finance team read access |
| Each SSAS role has minimum necessary permissions (Read / ReadDefinition only for report users) | 🔴 | No report user should have Administrator or Process permissions |
| No default "Everyone" or "Domain Users" AD group in unrestricted SSAS roles | 🔴 | Grants all domain users full model access |
| SSAS administrators role restricted to named IT staff only | 🔴 | SSAS Administrator can see all data, bypass all RLS |

**Query: SSAS role membership (PowerShell)**
```powershell
[System.Reflection.Assembly]::LoadWithPartialName("Microsoft.AnalysisServices") | Out-Null
$svr = New-Object Microsoft.AnalysisServices.Server
$svr.Connect("Data Source=SSAS_SERVER_NAME")
$db = $svr.Databases["ModelName"]

$db.Roles | ForEach-Object {
    $roleName = $_.Name
    $_.Members | ForEach-Object {
        [PSCustomObject]@{
            Role   = $roleName
            Member = $_.Name
            Type   = $_.MemberType
        }
    }
} | Format-Table -AutoSize
$svr.Disconnect()
```

---

### 5.2 Kerberos Delegation (Double-Hop)

The "double-hop" problem arises when:
1. User authenticates to PBIRS (hop 1)
2. PBIRS attempts to connect to SSAS on behalf of the user (hop 2)

Without Kerberos Constrained Delegation (KCD), hop 2 fails silently or presents a generic authentication error.

| Check | Severity | Notes |
|---|---|---|
| Service Principal Name (SPN) registered for PBIRS service account | 🔴 | `HTTP/pbirs-server.domain.com` and `HTTP/pbirs-server` |
| SPN registered for SSAS service account | 🔴 | `MSOLAPSvc.3/ssas-server.domain.com` and short form |
| KCD configured: PBIRS service account allowed to delegate to SSAS SPN | 🔴 | Set in AD Users & Computers → PBIRS service account → Delegation tab |
| Delegation type is **Constrained Delegation** (not unconstrained) | 🔴 | Unconstrained delegation is a security risk |
| Both SPNs are for service accounts (not computer accounts) when services run under domain accounts | 🟡 | |
| Kerberos tested end-to-end: check SSAS connection log shows delegated user identity | 🟡 | SSAS logs `EffectiveUserName` on successful Kerberos auth |

**PowerShell: Check SPN registration**
```powershell
# Check SPNs for PBIRS service account
setspn -L DOMAIN\PBIRSServiceAccount

# Check SPNs for SSAS service account
setspn -L DOMAIN\SSASServiceAccount

# Expected SPNs for SSAS:
# MSOLAPSvc.3/ssas-server.domain.com:2383
# MSOLAPSvc.3/ssas-server
```

**CMD: Verify Kerberos ticket for SSAS**
```cmd
klist tickets
:: Look for ticket for MSOLAPSvc/ssas-server — confirms Kerberos is used
```

---

### 5.3 PBIRS Service Account Permissions

| Check | Severity | Notes |
|---|---|---|
| PBIRS service account is a dedicated service account (not a personal user account) | 🔴 | Personal accounts change password, leave the organisation |
| PBIRS service account has Read permission on SSAS model role | 🔴 | Required for service-level report rendering |
| PBIRS service account does NOT have SSAS Administrator or Process permissions | 🔴 | Over-privileged service accounts are a security risk |
| PBIRS service account has Read permission on shared network paths used for report subscriptions | 🟡 | Required for file delivery subscriptions |
| PBIRS service account is not a local administrator on the SSAS server | 🔴 | |
| Password expiry: service account uses "Password never expires" with a strong password (or gMSA) | 🟡 | Password expiry will break scheduled jobs |

---

### 5.4 Column-Level Security

| Check | Severity | Notes |
|---|---|---|
| Sensitive columns (salary, PII, financial detail) hidden per SSAS role | 🟡 | Column-level security in SSAS: set column Hidden = True per role via Tabular Editor |
| PII columns identified and documented | 🔴 | Data Protection compliance requirement |
| SSAS model does not import columns that are not required for any report | 🟡 | Remove unnecessary imports at partition query level |

---

### 5.5 Audit Logging

| Check | Severity | Notes |
|---|---|---|
| SQL Server Audit configured on DW database (DDL + DML access to sensitive tables) | 🟡 | SQL Server Audit writes to Windows Event Log or file |
| SSAS query log enabled (logs query text, user, duration) | 🟢 | `QueryLogConnectionString` and `QueryLogFrequency` in SSAS server properties |
| PBIRS access log reviewed periodically | 🟢 | PBIRS writes HTTP logs; review for unauthorised access patterns |
| ETL audit table retains load history for at least 90 days | 🟡 | Required for investigating data quality issues retrospectively |

**T-SQL: Enable SQL Server Audit on DW database**
```sql
-- Server audit definition (writes to Windows Application Log)
CREATE SERVER AUDIT DWDataAccessAudit
TO APPLICATION_LOG
WITH (QUEUE_DELAY = 1000, ON_FAILURE = CONTINUE);

-- Database audit specification
CREATE DATABASE AUDIT SPECIFICATION DWTableAccess
FOR SERVER AUDIT DWDataAccessAudit
ADD (SELECT, INSERT, UPDATE, DELETE ON SCHEMA::dw BY PUBLIC)
WITH (STATE = ON);

ALTER SERVER AUDIT DWDataAccessAudit WITH (STATE = ON);
```

---

## Section 6: Review Output Template

```markdown
## DW / SSAS Review — [Project Name] — [Date]

**Reviewer:** [Name]
**Review Scope:** [Full model / New feature / Post-incident]
**Environment:** [Server names, SSAS version, PBIRS version, SQL Server version]

---

### Critical Findings (🔴)

| # | Finding | Location | Recommendation | Owner | Due Date |
|---|---|---|---|---|---|
| C1 | | | | | |

### Warning Findings (🟡)

| # | Finding | Location | Recommendation | Owner | Due Date |
|---|---|---|---|---|---|
| W1 | | | | | |

### Info / Observations (🟢)

| # | Observation | Notes |
|---|---|---|
| I1 | | |

---

### Summary Scorecard

| Section | Critical | Warning | Info | Status |
|---|---|---|---|---|
| 1.1 Fact Table Audit | | | | ✅ / ⚠️ / ❌ |
| 1.2 Dimension Audit | | | | |
| 1.3 Schema Health | | | | |
| 1.4 Physical Performance | | | | |
| 1.5 ETL Process | | | | |
| 1.6 SQL Agent Jobs | | | | |
| 3.x SSAS Model | | | | |
| 3.6 SSAS Processing | | | | |
| 3.7 PBIRS Live Connection | | | | |
| 5.x Security | | | | |

**Overall Recommendation:** [ APPROVE | APPROVE WITH CONDITIONS | REJECT ]
```
