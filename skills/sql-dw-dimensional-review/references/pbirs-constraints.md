# PBIRS Constraints & Best Practices — Power BI Report Server with SSAS Tabular Live Connection

> **Target Stack:** Power BI Report Server (on-prem), SSAS Tabular (on-prem, CL 1200–1600),  
> SQL Server 2016–2022, Active Directory / Windows Authentication, Kerberos.  
> No Power BI Service. No Azure. No Microsoft Fabric.

---

## Table of Contents

1. [PBIRS vs. Power BI Service — Feature Comparison](#1-pbirs-vs-power-bi-service--feature-comparison)
2. [Version Compatibility Matrix](#2-version-compatibility-matrix)
3. [Live Connection Constraints](#3-live-connection-constraints)
4. [Report Design Best Practices for Live Connection](#4-report-design-best-practices-for-live-connection)
5. [PBIRS Authentication — Kerberos and Windows Auth](#5-pbirs-authentication--kerberos-and-windows-auth)
6. [Paginated Reports vs. Power BI Canvas Reports](#6-paginated-reports-vs-power-bi-canvas-reports)
7. [Deployment and Versioning](#7-deployment-and-versioning)
8. [Performance Tuning](#8-performance-tuning)

---

## 1. PBIRS vs. Power BI Service — Feature Comparison

Use this table to quickly determine what is and is not available when deploying on-prem with PBIRS.

| Feature | Power BI Service (Cloud) | PBIRS (On-Premises) | Notes |
|---|---|---|---|
| **Live connection to SSAS Tabular** | ✅ (via gateway) | ✅ (direct) | PBIRS preferred for on-prem — no gateway overhead |
| **Import mode (.pbix)** | ✅ | ✅ | Limited by PBIRS server RAM |
| **DirectQuery to SQL Server** | ✅ | ✅ | |
| **Composite models (Import + DQ)** | ✅ | ❌ | Not supported in PBIRS |
| **Report-level DAX measures** | ✅ | ❌ | Must exist in the Tabular model |
| **Calculated tables in report** | ✅ | ❌ | Not supported in live connection |
| **Power Query transforms in live connection** | ❌ | ❌ | Not supported in either |
| **Q&A (natural language queries)** | ✅ | ⚠️ Older releases only | Check PBIRS version |
| **AI Insights (AutoML, Key Influencers)** | ✅ | ❌ / Limited | Version-dependent; generally not supported |
| **Dataflows** | ✅ | ❌ | PBIRS has no dataflow support |
| **Row-level security (model-defined)** | ✅ | ✅ | Delegated via Kerberos |
| **Report RLS (defined in .pbix)** | ✅ | ❌ | Not supported in live connection mode |
| **Scheduled refresh** | ✅ | ✅ (import mode only) | Live connection — no scheduled refresh needed |
| **Subscriptions (email delivery)** | ✅ | ✅ (PBIRS + SSRS engine) | |
| **Paginated reports (SSRS)** | ✅ (Premium/PPU only) | ✅ (native) | Core PBIRS strength |
| **Mobile-optimised layouts** | ✅ | ✅ | |
| **Themes (.json)** | ✅ | ✅ (recent PBIRS) | |
| **Incremental refresh** | ✅ (Premium/PPU) | ❌ | Not available on-prem |
| **Deployment pipelines** | ✅ | ❌ | Manual deployment required on-prem |
| **Usage metrics** | ✅ | ❌ | No built-in usage analytics in PBIRS |
| **Power BI apps** | ✅ | ❌ | No app workspaces in PBIRS |
| **Sharing via web URL** | ✅ | ✅ (intranet only) | External sharing requires reverse proxy |
| **Embed in SharePoint** | ✅ | ✅ (PBIRS web part / iframe) | |
| **Calculation Groups authoring** | ✅ | ✅ (via Tabular Editor against SSAS) | Not in PBIRS report; in SSAS model |
| **Azure AD authentication** | ✅ | ❌ | PBIRS uses Windows / AD only |
| **Multi-factor authentication** | ✅ | ❌ | Windows Integrated Auth only |
| **Custom visuals (AppSource)** | ✅ | ⚠️ Certified only, older versions | Test each custom visual with PBIRS version |
| **Power Automate integration** | ✅ | ❌ | |
| **XMLA endpoint** | ✅ (Premium/PPU) | ❌ | SSAS itself exposes XMLA on-prem |
| **Large dataset storage** | ✅ | ❌ | PBIRS has no large dataset tier |
| **Aggregations (automatic)** | ✅ (Premium) | ❌ | Only user-defined aggs in SSAS on-prem |
| **Template apps** | ✅ | ❌ | |

---

## 2. Version Compatibility Matrix

### PBIRS Releases and .pbix Compatibility Levels

> **Critical rule:** A .pbix file saved with a higher compatibility level than the PBIRS server supports **will fail to open** with a cryptic error. Always test .pbix files against the exact PBIRS version before deploying.

| PBIRS Release Month | PBIRS Version | Max .pbix Compat Level | Power BI Desktop Version to Use |
|---|---|---|---|
| January 2020 | 15.0.1102.911 | 1460 | Power BI Desktop for RS — Jan 2020 |
| May 2020 | 15.0.1103.234 | 1460 | Power BI Desktop for RS — May 2020 |
| January 2021 | 15.0.1104.x | 1500 | Power BI Desktop for RS — Jan 2021 |
| May 2021 | 15.0.1105.x | 1500 | Power BI Desktop for RS — May 2021 |
| October 2021 | 15.0.1106.x | 1510 | Power BI Desktop for RS — Oct 2021 |
| May 2022 | 15.0.1107.x | 1520 | Power BI Desktop for RS — May 2022 |
| October 2022 | 15.0.1108.x | 1530 | Power BI Desktop for RS — Oct 2022 |
| January 2023 | 15.0.1109.x | 1540 | Power BI Desktop for RS — Jan 2023 |
| May 2023 | 15.0.1110.x | 1550 | Power BI Desktop for RS — May 2023 |
| January 2024 | 15.0.1111.x | 1560 | Power BI Desktop for RS — Jan 2024 |
| May 2024 | 15.0.1112.x | 1567 | Power BI Desktop for RS — May 2024 |

> **Always use "Power BI Desktop optimised for Power BI Report Server"** — this is a separate download from standard Power BI Desktop. Using standard Power BI Desktop creates .pbix files at the latest compatibility level which will fail on PBIRS.
>
> Download page: https://powerbi.microsoft.com/en-us/report-server/ → "Advanced download options"

### PBIRS and SSAS Tabular Compatibility

| SQL Server SSAS Version | Compatibility Level | Min PBIRS Version for Live Connection |
|---|---|---|
| SQL Server 2016 AS | 1200 | Any PBIRS version |
| SQL Server 2017 AS | 1400 | Any PBIRS version |
| SQL Server 2019 AS | 1500 | PBIRS January 2021+ |
| SQL Server 2022 AS | 1600 | PBIRS May 2022+ (recommended: Jan 2023+) |

### .pbix File Version — Checking Compatibility Level

```powershell
# Check the compatibility level of a .pbix file
# .pbix files are ZIP archives containing a DataModel component

Add-Type -AssemblyName System.IO.Compression.FileSystem

$pbixPath = "C:\Reports\MySalesReport.pbix"
$zipEntries = [System.IO.Compression.ZipFile]::OpenRead($pbixPath).Entries

# Look for DataModelSchema or DataModel
$entry = $zipEntries | Where-Object { $_.Name -eq "DataModelSchema" }
if ($entry) {
    $reader  = New-Object System.IO.StreamReader($entry.Open())
    $content = $reader.ReadToEnd()
    $reader.Close()
    # Parse compatibilityLevel from JSON schema
    if ($content -match '"compatibilityLevel"\s*:\s*(\d+)') {
        Write-Host "Compatibility Level: $($Matches[1])"
    }
}
```

---

## 3. Live Connection Constraints

### What You CANNOT Do in a PBIRS Live Connection Report

These are **hard limitations** — not workarounds. The feature literally does not exist in the report layer when connected live to SSAS.

| Constraint | Detail | Workaround |
|---|---|---|
| **No report-level DAX measures** | Cannot add new measures in Power BI Desktop that query the live model | Create the measure in the SSAS Tabular model; redeploy SSAS |
| **No calculated tables** | Cannot add a new table in the Data pane | Add to SSAS model |
| **No calculated columns (report level)** | Cannot add columns via DAX in the report | Add as computed column in SSAS model |
| **No Power Query transforms** | The Query Editor is disabled for live connection data sources | Transform data in the DW SQL view / partition query |
| **No composite model** | Cannot mix live connection with an additional import or DQ source | Create a separate import .pbix or use SSAS to host the additional data |
| **No report RLS** | Cannot define roles in the .pbix file | All RLS must be in SSAS model roles; Kerberos must delegate the user identity |
| **No field parameters (older PBIRS)** | Field parameters require compatibility level 1565+ | Check PBIRS version; upgrade if needed or use SSAS Calculation Groups |
| **No smart narrative visual** | Requires Power BI Service back-end AI | Remove from report before deploying to PBIRS |
| **No Anomaly Detection** | Cloud-only AI feature | Remove from report |
| **No Decomposition Tree drill-through to cloud data** | Local decomp tree visual works; cloud AI-powered version doesn't | Standard decomposition tree is available |
| **No export to .xlsx with formatted data from SSAS** | Live connection reports export raw data only | Use SSRS paginated report for formatted Excel export |
| **Cannot change the live connection target** | Once a .pbix is saved with live connection to SSAS, changing the target server requires re-authoring | Use a DNS alias (CNAME) for SSAS server name to enable seamless server migration |

### Composite Model Restriction — Details
In PBIRS, you cannot open a .pbix that contains a **composite model** (a model that combines a live connection source with an imported or DirectQuery source). This is a fundamental architecture restriction:
- PBIRS does not support the enhanced dataset metadata format required by composite models.
- If you attempt to open such a file on PBIRS, you will receive: *"This report requires a newer version of Power BI."*
- **Solution:** Either fully migrate to SSAS (move the import data into the Tabular model) or use import mode entirely (schedule refresh).

---

## 4. Report Design Best Practices for Live Connection

### Design Principle: The Tabular Model is the Semantic Layer

When authoring reports for live connection to SSAS on-prem, treat Power BI Desktop purely as a **visualisation tool**. All business logic belongs in the Tabular model.

| Design Decision | Recommendation |
|---|---|
| Calculated totals or sub-totals | Define as measures in SSAS; do not use visual-level aggregation overrides |
| Conditional formatting thresholds | Use a SSAS measure (e.g., `[Sales RAG Status]` returning 1/2/3); bind conditional formatting to that measure |
| Dynamic titles | Use a SSAS string measure (e.g., `[Report Title]`) in a card visual; PBIRS supports dynamic text binding |
| Parameter-driven measure selection | Use a Disconnected Table / Calculation Group in SSAS for measure switching |
| Date range filtering | Use slicers bound to DimDate hierarchy from SSAS; never create a local date table in the report |
| KPI targets | Store targets in a separate SSAS table; compute variance as a measure |
| URL drillthrough links | Report drillthrough (not URL actions) — URL actions pointing to another PBIRS report are supported |

### Managing the "No New Measures" Constraint

The most common frustration with live connection is needing a measure that doesn't exist in the model yet. Establish this workflow:

1. **Report author** documents the required measure in a specification (name, logic, expected format).
2. **SSAS developer** adds the measure to the model in a development SSAS instance.
3. **After testing**, the model is redeployed to production SSAS.
4. **Report author** refreshes the field list in Power BI Desktop (the measure now appears).
5. **Report is republished** to PBIRS.

> Maintain a **model change request log** to track measure additions across the release cycle.

### Hierarchies and Drill-Down

In live connection, the report consumes hierarchies defined in SSAS exactly as configured. Best practices:

```
SSAS Hierarchy Naming Convention:
  DimDate      → Calendar Hierarchy: Year > Quarter > Month > Date
  DimDate      → Fiscal Hierarchy:   Fiscal Year > Fiscal Quarter > Fiscal Month
  DimProduct   → Product Hierarchy:  Division > Category > Subcategory > Product
  DimEmployee  → Org Hierarchy:      Company > Division > Department > Employee

In PBIRS report:
  - Drag the SSAS hierarchy onto rows/columns — do not build your own with individual columns
  - This ensures consistent drill-path across all reports
```

### Slicer Sync in PBIRS

PBIRS supports slicer sync across pages. Configure in View → Sync Slicers panel:

- Sync the DimDate Year/Month slicer across all pages.
- Sync the DimRegion slicer across all pages that show regional data.
- Each sync group has a **Sync** toggle (updates when slicer changes) and a **Visible** toggle (shows/hides slicer on that page).

### Cross-Report Drillthrough

PBIRS supports cross-report drillthrough (navigate from one PBIRS report to a detail report):

```
Source report: Sales Summary
  Visual: Bar chart (Total Sales by Region)
  Right-click → Drillthrough → [Sales Detail by Region]
    → Target: /Reports/Sales/SalesDetailByRegion.pbix
    → Pass: DimRegion[RegionKey] as filter parameter
```

Configure in the target report: Pages → [Detail page] → Drillthrough → "Cross-report" toggle ON.

### Bookmarks and Report State

Bookmarks work in PBIRS live connection reports exactly as in Power BI Desktop:
- Save filter states, slicer states, and visual visibility.
- Use for "default view" vs. "YTD view" vs. "comparison view" state switching.
- Bookmarks are stored inside the .pbix file — they do not interact with SSAS.

### Performance Tip: Limit Default Load Volume

By default, a live connection report queries SSAS on every visual load. Design reports to require explicit filter selections before data loads:

```
Page-level filter: DimDate[CalendarYear] = [Current Year]    ← set as default
Slicer default:    DimRegion[Region] = "All"

-- SSAS measure to prevent blank page load:
Has Filter Applied :=
IF(
    ISFILTERED( DimDate[CalendarYear] ) || ISFILTERED( DimDate[MonthName] ),
    1, 0
)
-- Use in visual-level filter: [Has Filter Applied] = 1 on heavy visuals
```

---

## 5. PBIRS Authentication — Kerberos and Windows Auth

### Authentication Flow: PBIRS → SSAS

```
User Browser
     │  (1) User opens PBIRS URL — authenticates with Windows Integrated Auth (NTLM or Kerberos)
     ▼
PBIRS Web Portal (IIS)
     │  (2) PBIRS renders report, needs SSAS data
     │  (3) PBIRS connects to SSAS — must impersonate the user (double-hop)
     ▼
SSAS Tabular (on different server)
     │  (4) SSAS applies RLS based on delegated user identity (USERNAME())
     ▼
Data returned to PBIRS → rendered to user browser
```

**Step 3 requires Kerberos Constrained Delegation (KCD).** NTLM cannot be delegated (it is a challenge-response protocol that cannot be forwarded). If NTLM is used at step 1, step 3 fails.

### Kerberos Configuration Checklist

**Step 1: Register SPNs for PBIRS**
```cmd
:: Run as Domain Admin
setspn -S HTTP/pbirs-server.contoso.com CONTOSO\PBIRSServiceAccount
setspn -S HTTP/pbirs-server             CONTOSO\PBIRSServiceAccount

:: Verify
setspn -L CONTOSO\PBIRSServiceAccount
```

**Step 2: Register SPNs for SSAS**
```cmd
:: SSAS default instance on port 2383
setspn -S MSOLAPSvc.3/ssas-server.contoso.com:2383 CONTOSO\SSASServiceAccount
setspn -S MSOLAPSvc.3/ssas-server.contoso.com       CONTOSO\SSASServiceAccount
setspn -S MSOLAPSvc.3/ssas-server                   CONTOSO\SSASServiceAccount

:: Named SSAS instance
setspn -S MSOLAPSvc.3/ssas-server.contoso.com\INSTANCE CONTOSO\SSASServiceAccount

:: Verify
setspn -L CONTOSO\SSASServiceAccount
```

**Step 3: Configure Constrained Delegation for PBIRS service account**

In Active Directory Users and Computers:
1. Find the PBIRS service account (`PBIRSServiceAccount`).
2. Properties → Delegation tab.
3. Select **"Trust this user for delegation to specified services only"**.
4. Select **"Use Kerberos only"** (not "Use any authentication protocol" — that is Protocol Transition and is only needed for NTLM clients).
5. Click Add → Users or Computers → find the SSAS service account.
6. Select the SPN: `MSOLAPSvc.3/ssas-server.contoso.com`.
7. Apply and OK.

```powershell
# PowerShell alternative (requires Active Directory module)
Set-ADAccountControl -Identity "PBIRSServiceAccount" -TrustedForDelegation $false
Set-ADUser -Identity "PBIRSServiceAccount" `
    -Add @{ msDS-AllowedToDelegateTo = "MSOLAPSvc.3/ssas-server.contoso.com" }
```

**Step 4: Verify PBIRS IIS Application Pool uses the service account**
```powershell
# Check IIS App Pool identity
Import-Module WebAdministration
Get-Item "IIS:\AppPools\PowerBIReportServer" | Select-Object processModel
# processModel.userName should be CONTOSO\PBIRSServiceAccount
```

**Step 5: Set PBIRS to use Windows Authentication (Negotiate: Kerberos)**

In PBIRS Configuration Manager:
- Web Service URL → Advanced → Authentication: Negotiate (Kerberos preferred over NTLM)

Or edit `rsreportserver.config`:
```xml
<Authentication>
  <AuthenticationTypes>
    <RSWindowsNegotiate />
  </AuthenticationTypes>
</Authentication>
```

**Step 6: Test end-to-end**
```powershell
# On PBIRS server, test that the Kerberos ticket is being issued
klist purge
# Open PBIRS report that connects to SSAS
klist tickets
# Should see: MSOLAPSvc.3/ssas-server.contoso.com — confirms Kerberos issued
```

### Troubleshooting Kerberos Double-Hop

| Symptom | Likely Cause | Resolution |
|---|---|---|
| "Cannot connect to SSAS" for all users | SPN missing or wrong | Run `setspn -L` and compare |
| Works for some users, not others | Some users authenticate via NTLM (not Kerberos) | Check if users on same domain; check IE/Edge security zones |
| "Access denied" in SSAS log | User identity delegated but not in any SSAS role | Add user / AD group to SSAS model role |
| RLS not applied (user sees all data) | SSAS USERNAME() returns PBIRS service account, not end user | Kerberos delegation not configured; SSAS sees service account |
| "Login failed" in SSAS | Service account not in SSAS Administrators or Read role | Grant SSAS role membership to PBIRS service account for report rendering |

**SSAS Query Log — Verify delegated identity:**
```
C:\Program Files\Microsoft SQL Server\MSAS15.MSSQLSERVER\OLAP\Log\msmdsrv.log
```
Look for `EffectiveUserName=CONTOSO\enduser` — this confirms delegation is working.

### PBIRS Service Account Minimum Permissions

| Resource | Permission Required |
|---|---|
| SSAS Model | Member of an SSAS Role with Read permission |
| SSAS Server | Not an SSAS Administrator (unless needed for processing jobs) |
| DW SQL Server | Not required for live connection (SSAS handles data access) |
| PBIRS database (ReportServer DB) | db_owner on ReportServer and ReportServerTempDB |
| Network share (subscriptions) | Read/Write on delivery share |
| Windows local server | Logon as service right |

---

## 6. Paginated Reports vs. Power BI Canvas Reports

### Decision Framework

Use this matrix to determine which report type to use:

| Requirement | Power BI Canvas Report | SSRS/PBIRS Paginated Report |
|---|---|---|
| **Interactive exploration** (slicing, drill-down, cross-filtering) | ✅ | ❌ |
| **Fixed-format output** for print / PDF / Excel | ⚠️ Limited | ✅ |
| **Pixel-perfect layout** (invoices, certificates, compliance reports) | ❌ | ✅ |
| **Multi-page output** (1 page per region, data-driven subscriptions) | ❌ | ✅ |
| **Table/list with thousands of rows** | ❌ (performance degrades) | ✅ |
| **Parameterised drill-through** (pass filter from Power BI to paginated) | ✅ Source | ✅ Target |
| **Subreports** | ❌ | ✅ |
| **Matrix with subtotals and custom grouping** | ⚠️ Limited | ✅ |
| **Scheduled email delivery with formatted attachment** | ⚠️ PNG/PDF only | ✅ Excel, PDF, Word, CSV |
| **Group-by pagination** (new page per customer) | ❌ | ✅ |
| **Conditional show/hide of report sections** | ⚠️ Bookmarks workaround | ✅ Native visibility expressions |
| **Historical trend visualisation** | ✅ | ⚠️ Limited charts |
| **Dashboard summary (KPIs, gauge, sparklines)** | ✅ | ❌ |

### DAX Datasets for Paginated Reports

PBIRS paginated reports can use DAX queries against SSAS Tabular as their dataset:

```dax
-- DAX dataset for a Sales by Region paginated report
-- SSRS parameters: @StartYear (Int), @EndYear (Int), @RegionKey (Int)
EVALUATE
CALCULATETABLE(
    SUMMARIZECOLUMNS(
        DimRegion[RegionName],
        DimRegion[Territory],
        DimDate[CalendarYear],
        DimDate[MonthNameShort],
        DimDate[MonthNumber],
        "Total Sales",          [Total Sales],
        "Total Cost",           [Total Cost],
        "Gross Profit",         [Gross Profit],
        "Gross Margin %",       [Gross Margin %],
        "Units Sold",           [Units Sold]
    ),
    DimDate[CalendarYear] >= @StartYear,
    DimDate[CalendarYear] <= @EndYear,
    IF( @RegionKey = 0, TRUE(), DimRegion[RegionKey] = @RegionKey )
)
ORDER BY
    DimDate[CalendarYear],
    DimDate[MonthNumber],
    DimRegion[RegionName]
```

### MDX Datasets for Paginated Reports

MDX is often more natural for hierarchical / cross-tab paginated report layouts:

```mdx
-- MDX dataset for product category cross-tab with subtotals
WITH
  MEMBER [Measures].[Gross Margin %] AS
    IIF([Measures].[Total Sales] = 0, NULL,
        [Measures].[Gross Profit] / [Measures].[Total Sales])

SELECT
  NON EMPTY {
    [Measures].[Total Sales],
    [Measures].[Gross Profit],
    [Measures].[Gross Margin %]
  } ON COLUMNS,
  NON EMPTY {
    [DimProduct].[Product Hierarchy].[Division].MEMBERS *
    [DimProduct].[Product Hierarchy].[Category].MEMBERS
  } ON ROWS
FROM [SalesTabularModel]
WHERE (
  [DimDate].[Calendar Hierarchy].[Calendar Year].&[@StartYear]
  : [DimDate].[Calendar Hierarchy].[Calendar Year].&[@EndYear]
)
```

> **DAX vs. MDX for paginated reports:** Use DAX when:
> - You need flat tabular data (list reports, table reports).
> - You want to reuse DAX measures already defined in the model.
> Use MDX when:
> - You need hierarchical cross-tab output with automatic subtotals.
> - You need MEMBER expressions or calculated sets not easily expressed in DAX.
> - Your SSRS paginated reports already use MDX historically.

### Drillthrough from Power BI Canvas to Paginated Report

Configure in PBIRS to navigate from a canvas report to a paginated report with parameters:

```
In Power BI Canvas report (Sales Summary):
  Right-click on bar chart visual → Drillthrough

URL Action configuration:
  URL = CONCATENATE(
    "http://pbirs-server/ReportServer/Pages/ReportViewer.aspx?",
    "%2fReports%2fSales%2fSalesDetailPaginated",
    "&rs:Command=Render",
    "&StartYear=", SELECTEDVALUE(DimDate[CalendarYear]),
    "&RegionKey=", SELECTEDVALUE(DimRegion[RegionKey])
  )
```

---

## 7. Deployment and Versioning

### .pbix Deployment to PBIRS

**Method 1: PBIRS Web Portal (manual)**
- Navigate to the folder in the PBIRS web portal.
- Upload → Browse to .pbix file.
- Simple but not suitable for CI/CD.

**Method 2: PowerShell REST API (automated deployment)**

```powershell
# PBIRS REST API base URL
$PBIRSBase = "http://pbirs-server/ReportServer/api/v2.0"

# Function: Upload or overwrite a .pbix report to PBIRS
function Publish-PBIRSReport {
    param(
        [string] $PbixPath,
        [string] $TargetFolder,    # e.g. "/Reports/Sales"
        [string] $ReportName       # e.g. "SalesSummary"
    )

    $fileName   = Split-Path $PbixPath -Leaf
    $fileBytes  = [System.IO.File]::ReadAllBytes($PbixPath)
    $boundary   = [System.Guid]::NewGuid().ToString()

    $headers = @{
        "Content-Type"  = "multipart/form-data; boundary=$boundary"
        "Accept"        = "application/json"
    }

    # Build multipart body
    $body = "--$boundary`r`n"
    $body += "Content-Disposition: form-data; name=`"File`"; filename=`"$fileName`"`r`n"
    $body += "Content-Type: application/octet-stream`r`n`r`n"
    $bodyBytes = [System.Text.Encoding]::UTF8.GetBytes($body)
    $endBytes  = [System.Text.Encoding]::UTF8.GetBytes("`r`n--$boundary--`r`n")
    $payload   = $bodyBytes + $fileBytes + $endBytes

    # Upload to PBIRS (overwrite if exists)
    $url = "$PBIRSBase/PowerBIReports?path=$([Uri]::EscapeDataString("$TargetFolder/$ReportName"))"
    try {
        $resp = Invoke-WebRequest -Uri $url -Method POST -Body $payload `
                    -Headers $headers -UseDefaultCredentials -ContentType "multipart/form-data; boundary=$boundary"
        Write-Host "✅ Published: $ReportName → $TargetFolder (Status: $($resp.StatusCode))"
    }
    catch {
        Write-Host "❌ Failed to publish $ReportName : $_"
    }
}

# Example usage:
Publish-PBIRSReport `
    -PbixPath   "C:\Reports\SalesSummary.pbix" `
    -TargetFolder "/Reports/Sales" `
    -ReportName "SalesSummary"
```

**Method 3: PowerShell — List existing reports (inventory)**

```powershell
$PBIRSBase = "http://pbirs-server/ReportServer/api/v2.0"

# List all Power BI reports in a folder
$resp = Invoke-RestMethod "$PBIRSBase/PowerBIReports?`$expand=Properties" `
            -UseDefaultCredentials
$resp.value | Select-Object Name, Path, ModifiedDate | Format-Table -AutoSize
```

**Method 4: PBIRS PowerShell module (RS tools)**

```powershell
# Install community RS tools PowerShell module (open source, not official Microsoft)
Install-Module -Name ReportingServicesTools -Scope CurrentUser

# Upload .pbix using RS tools
Write-RsRestReport `
    -Path        "C:\Reports\SalesSummary.pbix" `
    -RsFolder    "/Reports/Sales" `
    -ReportServerUri "http://pbirs-server/ReportServer"
```

### .pbix File Version Control

PBIRS has **no built-in version history** for .pbix files. Implement version control externally:

```
Recommended approach:
1. Store .pbix files in a Git repository (or Azure DevOps / TFS)
2. Use a "Reports" folder in the repo with subdirectories matching PBIRS folder structure
3. Each deployment tags the git commit with the PBIRS server and folder
4. Deploy via PowerShell as part of a release pipeline (Azure DevOps / Jenkins / TeamCity)

Folder structure:
Reports/
├── Sales/
│   ├── SalesSummary.pbix
│   └── SalesDetailByRegion.pbix
├── Finance/
│   └── PLReport.pbix
└── HR/
    └── Headcount.pbix
```

### PBIRS Report Data Source Management

Centralise data source credentials using **Shared Data Sources** in PBIRS:

```powershell
# Create a shared SSAS data source via REST API
$dataSource = @{
    Name            = "SSAS_SalesModel"
    ConnectionString = "Data Source=ssas-server;Initial Catalog=SalesTabularModel"
    DataSourceType  = "OLEDB-MD"
    CredentialRetrieval = "Integrated"  # Windows/Kerberos
} | ConvertTo-Json

Invoke-RestMethod "$PBIRSBase/DataSources" `
    -Method POST `
    -Body $dataSource `
    -ContentType "application/json" `
    -UseDefaultCredentials
```

---

## 8. Performance Tuning

### Report Rendering Performance

**Issue:** Reports that query SSAS with many visuals render slowly.

**Root causes and fixes:**

| Root Cause | Diagnostic | Fix |
|---|---|---|
| Too many visuals on one page | Browser F12 Network tab — many parallel SSAS queries | Reduce visuals per page; use bookmarks to hide off-screen visuals |
| Heavy measures without aggregations | DAX Studio Server Timings — high FE time | Add user-defined aggregations to SSAS; optimise DAX |
| Default "Show All" on slicers forces full scan | SSAS query log — full dimension scans | Set slicer "Selection" to dropdown; reduce slicer member counts |
| No SSAS query result caching | Repeated identical queries to SSAS | Enable SSAS instance caching (on by default — check memory pressure) |
| Large model not fitting in SSAS memory | SSAS memory counters (Process Explorer or Perfmon) | Increase SSAS max memory; remove unused columns from model |
| PBIRS renders report for each subscriber | Subscription concurrency hitting SSAS simultaneously | Stagger subscription schedules; increase SSAS thread pool |

### SSAS Connection Pooling from PBIRS

PBIRS maintains a connection pool to SSAS. Each user session establishes its own connection (because Kerberos delegation means each connection carries the user's identity — no pooling across users).

Configuration in PBIRS:
- The PBIRS connection pool per user is generally small (1–3 connections).
- Do not tune SSAS `MaxConnectionsPerPool` below 10 — this is the SSAS-side per-process pool.
- Monitor SSAS thread usage: `$SYSTEM.DISCOVER_CONNECTIONS` DMV.

```dax
-- DAX Studio DMV: active connections to SSAS
SELECT
    CONNECTION_ID,
    CONNECTION_USER_NAME,
    CONNECTION_START_TIME,
    CONNECTION_SERVER_MODE,
    CONNECTION_CURRENT_DATABASE
FROM $SYSTEM.DISCOVER_CONNECTIONS
ORDER BY CONNECTION_START_TIME DESC
```

### SSAS Memory Configuration for PBIRS Workloads

```xml
<!-- msmdsrv.ini / SSAS Server Properties -->
<!-- Set via SSMS → SSAS Server Properties → Memory -->

<!-- Hard cap: SSAS will never use more than this -->
<HardMemoryLimit>80</HardMemoryLimit>    <!-- 80% of server RAM -->

<!-- Soft limit: SSAS starts releasing cache above this -->
<VertiPaqMemoryLimit>60</VertiPaqMemoryLimit>  <!-- 60% of server RAM -->

<!-- Low memory limit: aggressive cache eviction below this -->
<LowMemoryLimit>40</LowMemoryLimit>

<!-- Query memory limit: per-query memory cap (GB) — prevents runaway queries -->
<QueryMemoryLimit>10</QueryMemoryLimit>  <!-- 10GB per query -->
```

### PBIRS Capacity Planning

| Metric | Minimum | Recommended | Notes |
|---|---|---|---|
| **PBIRS Server RAM** | 8 GB | 16–32 GB | More RAM = more concurrent users and better import mode refresh |
| **PBIRS Server CPU** | 4 cores | 8–16 cores | Report rendering is CPU-bound |
| **SSAS Server RAM** | 16 GB | 32–128 GB | Must fit entire VertiPaq model + working set |
| **SSAS Server CPU** | 8 cores | 16+ cores | SE queries are parallelised |
| **Concurrent users (live connection)** | Up to 50 on modest hardware | Test with representative queries | Each live connection visual generates an SSAS query |
| **Disk I/O (SSAS)** | SSD for SSAS data directory | NVMe for large model cold start | Cold start reads entire model from disk |
| **Network (PBIRS ↔ SSAS)** | 1 Gbps | 10 Gbps for large result sets | Low-latency is more important than bandwidth |

### Perfmon Counters to Monitor

```powershell
# Key performance counters for SSAS + PBIRS monitoring
$Counters = @(
    # SSAS memory
    "\MSAS15:Memory\Memory Usage KB",
    "\MSAS15:Memory\Quota Blocked",
    # SSAS processing
    "\MSAS15:Processing\Rows read /sec",
    # SSAS queries
    "\MSAS15:MDX\MDX Flat cache: current entries",
    "\MSAS15:Locks\Current latch waits",
    # PBIRS
    "\ASP.NET Applications(__Total__)\Requests/Sec",
    "\ASP.NET Applications(__Total__)\Request Wait Time",
    # System
    "\Processor(_Total)\% Processor Time",
    "\Memory\Available MBytes"
)

Get-Counter -Counter $Counters -SampleInterval 5 -MaxSamples 12 |
    Export-Counter -Path "C:\Logs\SSAS_PBIRS_Perf.blg" -FileFormat BLG
```

### PBIRS Report Caching (Execution Snapshots)

For reports with heavy SSAS queries that don't need real-time data:

1. In PBIRS web portal: Report → Manage → Processing Options.
2. Set **Render this report from a report execution snapshot**.
3. Configure **Snapshot schedule** (e.g., refresh snapshot at 06:30 after SSAS processing completes).
4. All users viewing the report during business hours see the snapshot — zero SSAS query load.
5. **Trade-off:** Data is as fresh as the snapshot schedule, not live.

```powershell
# Set report execution snapshot via REST API
$settings = @{
    CacheOption = "Snapshot"
    ScheduleID  = "daily-0630"   # pre-existing shared schedule ID
} | ConvertTo-Json

Invoke-RestMethod "$PBIRSBase/PowerBIReports(Path='/Reports/Sales/SalesSummary')/CacheRefreshPlans" `
    -Method POST `
    -Body $settings `
    -ContentType "application/json" `
    -UseDefaultCredentials
```
