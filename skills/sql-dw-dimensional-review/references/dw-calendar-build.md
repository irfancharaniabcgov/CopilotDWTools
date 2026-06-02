# DW Calendar Dimension Build Reference

Reference scripts for `Dimension.Calendar`, `Dimension.StatHolidays`, and the
`SSAS.v_Calendar` view. This is a **one-time build** (idempotent — safe to re-run).

**Design basis:** Adapted from the existing `dbo.Calendar` / `dbo.[LoadCalendar]` / `dbo.stat_holiday`
objects in the OLTP database. Key differences in the DW version:

| Aspect | OLTP (`dbo.Calendar`) | DW (`Dimension.Calendar`) |
|---|---|---|
| Primary key type | `DATE` (`DateKey DATE`) | `INT` YYYYMMDD (`[Date Key] INT`) — Kimball convention |
| Unknown/sentinel rows | `'1753-01-01'` and `'9999-12-31'` | `Date Key = -1` (aligns with `EndDateKey = -1` for open events) |
| Load approach | `WHILE` loop (row-by-row) | Recursive CTE (set-based — 10–100× faster) |
| Column naming | camelCase / no spaces | Title Case with spaces — SSAS/Kimball convention |
| StatHolidays | `dbo.stat_holiday` (BC + audit cols) | `Dimension.StatHolidays` (same structure, updated generator) |

**Fiscal year convention (org standard — confirmed from `dbo.LoadCalendar`):**
- Fiscal year = **starting** calendar year: **FY2024 = Apr 1, 2024 – Mar 31, 2025**
- `Fiscal Year = CASE WHEN MONTH >= 4 THEN YEAR ELSE YEAR - 1 END`
- Fiscal Period 1 = April, Fiscal Period 12 = March
- Week numbering: Sunday-start (`DATEFIRST 7`)

---

## 1. Dimension.StatHolidays Table

Stat holiday data is produced by the Nager.Date C# project (see Section 6 for
update notes). The project outputs a SQL `MERGE` script targeting this table.

```sql
IF NOT EXISTS (
    SELECT 1 FROM sys.tables t
    JOIN sys.schemas s ON t.schema_id = s.schema_id
    WHERE s.name = 'Dimension' AND t.name = 'StatHolidays'
)
BEGIN
    CREATE TABLE [Dimension].[StatHolidays] (
        [Stat Holiday Key]      INT           NOT NULL IDENTITY(1, 1),
        [Stat Holiday Date]     DATE          NOT NULL,
        [Stat Holiday Date Key] INT           NOT NULL,  -- YYYYMMDD FK to Dimension.Calendar
        [Stat Holiday Name]     NVARCHAR(200) NOT NULL,
        [Entry Date]            DATETIME2(7)  NOT NULL CONSTRAINT [DF_StatHolidays_EntryDate]  DEFAULT (GETDATE()),
        [Update Date]           DATETIME2(7)  NULL,
        [Is Deleted]            BIT           NOT NULL CONSTRAINT [DF_StatHolidays_IsDeleted] DEFAULT (0),
        CONSTRAINT [PK_Dimension_StatHolidays]
            PRIMARY KEY CLUSTERED ([Stat Holiday Key]),
        CONSTRAINT [UQ_Dimension_StatHolidays_Date]
            UNIQUE NONCLUSTERED ([Stat Holiday Date])
    );

    EXECUTE sp_addextendedproperty
        @name = N'MS_Description',
        @value = N'BC statutory and quasi-statutory holidays (Easter Monday, Boxing Day included per org policy). Populated by the Nager.Date C# project. Use SSAS.v_Calendar to join holiday data to Dimension.Calendar.',
        @level0type = N'SCHEMA', @level0name = N'Dimension',
        @level1type = N'TABLE',  @level1name = N'StatHolidays';
END
GO
```

**How the Nager.Date generator output integrates:**

The generator produces a MERGE statement like:
```sql
MERGE [Dimension].[StatHolidays] AS Target
USING @tbl AS Source
ON Source.holidayDate = Target.[Stat Holiday Date]
WHEN NOT MATCHED BY TARGET THEN
    INSERT ([Stat Holiday Date], [Stat Holiday Date Key], [Stat Holiday Name])
    VALUES (Source.holidayDate,
            YEAR(Source.holidayDate)*10000 + MONTH(Source.holidayDate)*100 + DAY(Source.holidayDate),
            Source.holidayName);
```

The C# project needs updating to target `Dimension.StatHolidays` using this schema
(see Section 6).

---

## 2. Dimension.Calendar Table

```sql
IF NOT EXISTS (
    SELECT 1 FROM sys.tables t
    JOIN sys.schemas s ON t.schema_id = s.schema_id
    WHERE s.name = 'Dimension' AND t.name = 'Calendar'
)
BEGIN
    CREATE TABLE [Dimension].[Calendar] (
        -- ── Key ─────────────────────────────────────────────────────────────────
        [Date Key]                  INT           NOT NULL,  -- YYYYMMDD; -1 = Unknown
        [Date]                      DATE          NULL,

        -- ── Calendar Day ─────────────────────────────────────────────────────
        [Day]                       TINYINT       NULL,      -- 1–31
        [Day Of Week]               TINYINT       NULL,      -- 1=Sun … 7=Sat (DATEFIRST 7)
        [Weekday Name]              VARCHAR(10)   NULL,      -- 'Sunday' … 'Saturday'
        [Weekday Name Short]        CHAR(3)       NULL,      -- 'Sun' … 'Sat'
        [Weekday Name First Letter] CHAR(1)       NULL,      -- 'S' … 'S'
        [Is Weekday]                BIT           NULL,      -- 1 = Mon–Fri
        [Is Weekend]                BIT           NULL,      -- 1 = Sat or Sun
        [Day Of Week In Month]      TINYINT       NULL,      -- Nth occurrence of weekday in month (e.g. 3 = 3rd Monday)
        [Day Of Week In Year]       TINYINT       NULL,      -- Nth occurrence of weekday in year
        [Day Of Year]               SMALLINT      NULL,      -- 1–366

        -- ── Calendar Week (Sunday-start) ─────────────────────────────────────
        [Week Of Month]             TINYINT       NULL,      -- 1–6
        [Week Of Year]              TINYINT       NULL,      -- 1–53, Sunday-start
        [First Date Of Week]        DATE          NULL,      -- Sunday of the week
        [Last Date Of Week]         DATE          NULL,      -- Saturday of the week

        -- ── Calendar Month ───────────────────────────────────────────────────
        [Month]                     TINYINT       NULL,      -- 1–12
        [Month Name]                VARCHAR(10)   NULL,      -- 'January' …
        [Month Name Short]          CHAR(3)       NULL,      -- 'Jan' …
        [Month Name First Letter]   CHAR(1)       NULL,      -- 'J' …
        [Month Year]                CHAR(8)       NULL,      -- 'Apr 2024'
        [YYYYMM]                    CHAR(6)       NULL,      -- '202404'
        [Days In Month]             TINYINT       NULL,      -- 28–31
        [First Date Of Month]       DATE          NULL,
        [Last Date Of Month]        DATE          NULL,

        -- ── Calendar Quarter ─────────────────────────────────────────────────
        [Quarter]                   TINYINT       NULL,      -- 1–4
        [Quarter Name]              CHAR(2)       NULL,      -- 'Q1' …
        [First Date Of Quarter]     DATE          NULL,
        [Last Date Of Quarter]      DATE          NULL,

        -- ── Calendar Year ────────────────────────────────────────────────────
        [Year]                      SMALLINT      NULL,      -- e.g. 2024
        [Is Leap Year]              BIT           NULL,
        [First Date Of Year]        DATE          NULL,
        [Last Date Of Year]         DATE          NULL,

        -- ── Pay Week (biweekly from 2000-01-01 root) ────────────────────────
        -- Used for payroll report filtering; 0 = even week, 1 = odd week
        [Is Pay Week]               BIT           NULL,

        -- ── Fiscal Year (Apr 1 – Mar 31; FY = starting calendar year) ───────
        -- FY2024 = Apr 1, 2024 – Mar 31, 2025
        -- FiscalYear = YEAR when Month >= 4; YEAR - 1 when Month < 4
        [Fiscal Year]               SMALLINT      NULL,      -- e.g. 2024 for Apr 2024 – Mar 2025
        [Fiscal Year Label]         CHAR(6)       NULL,      -- 'FY2024'
        [Fiscal Year Label Short]   CHAR(4)       NULL,      -- 'FY24'

        -- Fiscal Period: Apr=1, May=2, …, Dec=9, Jan=10, Feb=11, Mar=12
        [Fiscal Month]              TINYINT       NULL,      -- 1–12 (same as Fiscal Period)
        [Fiscal Period Label]       VARCHAR(10)   NULL,      -- 'FY2024 P01'
        [Fiscal Period Sort]        INT           NULL,      -- FiscalYear * 100 + FiscalMonth (for sorting)

        [Fiscal Quarter]            TINYINT       NULL,      -- 1–4 (Q1=Apr–Jun, Q4=Jan–Mar)
        [Fiscal Quarter Name]       CHAR(2)       NULL,      -- 'Q1' … 'Q4'
        [Fiscal Quarter Label]      VARCHAR(10)   NULL,      -- 'FY2024 Q1'
        [Fiscal Quarter Sort]       INT           NULL,      -- FiscalYear * 10 + FiscalQuarter

        CONSTRAINT [PK_Dimension_Calendar]
            PRIMARY KEY CLUSTERED ([Date Key])
    );

    EXECUTE sp_addextendedproperty
        @name = N'MS_Description',
        @value = N'Date dimension. Range 2000-01-01 to 2050-12-31. Date Key = -1 is unknown member (open/active records). Fiscal year uses starting year convention: FY2024 = Apr 1 2024 – Mar 31 2025. Week numbering is Sunday-start (DATEFIRST 7). Stat holidays and working-day flags are in SSAS.v_Calendar.',
        @level0type = N'SCHEMA', @level0name = N'Dimension',
        @level1type = N'TABLE',  @level1name = N'Calendar';
END
GO
```

---

## 3. Calendar Population Stored Procedure

Replaces the OLTP `dbo.LoadCalendar` WHILE loop with a set-based CTE approach.
Covers 2000-01-01 to 2050-12-31 (18,628 rows + 1 unknown member = 18,629 total).

```sql
CREATE OR ALTER PROCEDURE [Dimension].[PopulateCalendar]
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;
    SET DATEFIRST 7;   -- Sunday = 1 (US week convention, matches OLTP LoadCalendar)

    BEGIN TRY
        DECLARE @PayRootDate DATE = '2000-01-01';  -- biweekly pay cycle root

        -- ── Generate date spine ──────────────────────────────────────────────
        ;WITH DateSpine AS (
            SELECT CAST('2000-01-01' AS DATE) AS [Date]
            UNION ALL
            SELECT DATEADD(DAY, 1, [Date])
            FROM DateSpine
            WHERE [Date] < '2050-12-31'
        ),
        Base AS (
            SELECT
                d.[Date],
                YEAR(d.[Date]) * 10000
                    + MONTH(d.[Date]) * 100
                    + DAY(d.[Date])                     AS [Date Key],

                DAY(d.[Date])                           AS [Day],
                DATEPART(WEEKDAY, d.[Date])             AS [Day Of Week],  -- 1=Sun
                DATENAME(WEEKDAY,  d.[Date])            AS [Weekday Name],
                MONTH(d.[Date])                         AS [Month],
                DATENAME(MONTH,    d.[Date])            AS [Month Name],
                YEAR(d.[Date])                          AS [Year],

                -- Fiscal year: starting-year convention
                -- Apr–Dec: FY = current year  |  Jan–Mar: FY = prior year
                CASE WHEN MONTH(d.[Date]) >= 4
                     THEN YEAR(d.[Date])
                     ELSE YEAR(d.[Date]) - 1
                END                                     AS [Fiscal Year],

                -- Fiscal Period: Apr=1 … Mar=12
                CASE WHEN MONTH(d.[Date]) >= 4
                     THEN MONTH(d.[Date]) - 3
                     ELSE MONTH(d.[Date]) + 9
                END                                     AS [Fiscal Month],

                -- Fiscal Quarter: Q1=Apr–Jun, Q2=Jul–Sep, Q3=Oct–Dec, Q4=Jan–Mar
                CASE DATEPART(QUARTER, d.[Date])
                    WHEN 1 THEN 4
                    WHEN 2 THEN 1
                    WHEN 3 THEN 2
                    WHEN 4 THEN 3
                END                                     AS [Fiscal Quarter],

                -- IsLeapYear
                CAST(CASE
                    WHEN YEAR(d.[Date]) % 4   <> 0 THEN 0
                    WHEN YEAR(d.[Date]) % 100 <> 0 THEN 1
                    WHEN YEAR(d.[Date]) % 400 <> 0 THEN 0
                    ELSE 1
                END AS BIT)                             AS [Is Leap Year],

                -- Pay week: odd/even biweekly from root date
                CAST(ABS(DATEDIFF(WEEK, @PayRootDate, d.[Date])) & 1
                     AS BIT)                            AS [Is Pay Week],

                -- Period boundary dates
                DATEADD(DAY, -(DATEPART(WEEKDAY, d.[Date]) - 1), d.[Date])
                                                        AS [First Date Of Week],
                DATEADD(DAY, 7 - DATEPART(WEEKDAY, d.[Date]), d.[Date])
                                                        AS [Last Date Of Week],
                DATEFROMPARTS(YEAR(d.[Date]), MONTH(d.[Date]), 1)
                                                        AS [First Date Of Month],
                EOMONTH(d.[Date])                       AS [Last Date Of Month],
                DATEFROMPARTS(YEAR(d.[Date]), 1, 1)     AS [First Date Of Year],
                DATEFROMPARTS(YEAR(d.[Date]), 12, 31)   AS [Last Date Of Year],
                DATEADD(qq,  DATEDIFF(qq, 0, d.[Date]), 0)
                                                        AS [First Date Of Quarter],
                DATEADD(dd, -1, DATEADD(qq, DATEDIFF(qq, 0, d.[Date]) + 1, 0))
                                                        AS [Last Date Of Quarter]
            FROM DateSpine d
        )
        MERGE [Dimension].[Calendar] AS tgt
        USING (
            SELECT
                b.[Date Key],
                b.[Date],

                -- Day
                CAST(b.[Day]           AS TINYINT)  AS [Day],
                CAST(b.[Day Of Week]   AS TINYINT)  AS [Day Of Week],
                b.[Weekday Name],
                LEFT(b.[Weekday Name], 3)            AS [Weekday Name Short],
                LEFT(b.[Weekday Name], 1)            AS [Weekday Name First Letter],
                CAST(CASE WHEN b.[Day Of Week] BETWEEN 2 AND 6 THEN 1 ELSE 0 END AS BIT)
                                                    AS [Is Weekday],
                CAST(CASE WHEN b.[Day Of Week] IN (1, 7) THEN 1 ELSE 0 END AS BIT)
                                                    AS [Is Weekend],
                -- Day-of-weekday-in-month: e.g. 3 = 3rd Monday
                CAST((b.[Day] + 6) / 7              AS TINYINT)
                                                    AS [Day Of Week In Month],
                -- Day-of-weekday-in-year: nth occurrence in year
                CAST((DATEPART(DAYOFYEAR, b.[Date]) + 6) / 7  AS TINYINT)
                                                    AS [Day Of Week In Year],
                CAST(DATEPART(DAYOFYEAR, b.[Date])  AS SMALLINT)
                                                    AS [Day Of Year],

                -- Week
                CAST(
                    DATEPART(WEEK, b.[Date])
                    - DATEPART(WEEK, b.[First Date Of Month]) + 1
                    AS TINYINT)                      AS [Week Of Month],
                CAST(DATEPART(WEEK, b.[Date])        AS TINYINT)
                                                    AS [Week Of Year],
                b.[First Date Of Week],
                b.[Last Date Of Week],

                -- Month
                CAST(b.[Month]         AS TINYINT)  AS [Month],
                b.[Month Name],
                LEFT(b.[Month Name], 3)              AS [Month Name Short],
                LEFT(b.[Month Name], 1)              AS [Month Name First Letter],
                LEFT(b.[Month Name], 3) + ' ' + CAST(b.[Year] AS VARCHAR(4))
                                                    AS [Month Year],
                CAST(b.[Year] AS VARCHAR(4))
                    + RIGHT('0' + CAST(b.[Month] AS VARCHAR(2)), 2)
                                                    AS [YYYYMM],
                CAST(
                    CASE
                        WHEN b.[Month] IN (4,6,9,11) THEN 30
                        WHEN b.[Month] IN (1,3,5,7,8,10,12) THEN 31
                        WHEN b.[Month] = 2 AND b.[Is Leap Year] = 1 THEN 29
                        ELSE 28
                    END AS TINYINT)                 AS [Days In Month],
                b.[First Date Of Month],
                b.[Last Date Of Month],

                -- Quarter
                CAST(DATEPART(QUARTER, b.[Date])    AS TINYINT)
                                                    AS [Quarter],
                'Q' + CAST(DATEPART(QUARTER, b.[Date]) AS VARCHAR(1))
                                                    AS [Quarter Name],
                b.[First Date Of Quarter],
                b.[Last Date Of Quarter],

                -- Year
                CAST(b.[Year] AS SMALLINT)          AS [Year],
                b.[Is Leap Year],
                b.[First Date Of Year],
                b.[Last Date Of Year],

                -- Pay week
                b.[Is Pay Week],

                -- Fiscal
                CAST(b.[Fiscal Year]   AS SMALLINT) AS [Fiscal Year],
                'FY' + CAST(b.[Fiscal Year] AS VARCHAR(4))
                                                    AS [Fiscal Year Label],
                'FY' + RIGHT(CAST(b.[Fiscal Year] AS VARCHAR(4)), 2)
                                                    AS [Fiscal Year Label Short],
                CAST(b.[Fiscal Month]  AS TINYINT)  AS [Fiscal Month],
                'FY' + CAST(b.[Fiscal Year] AS VARCHAR(4))
                    + ' P' + RIGHT('0' + CAST(b.[Fiscal Month] AS VARCHAR(2)), 2)
                                                    AS [Fiscal Period Label],
                b.[Fiscal Year] * 100 + b.[Fiscal Month]
                                                    AS [Fiscal Period Sort],
                CAST(b.[Fiscal Quarter] AS TINYINT) AS [Fiscal Quarter],
                'Q' + CAST(b.[Fiscal Quarter] AS VARCHAR(1))
                                                    AS [Fiscal Quarter Name],
                'FY' + CAST(b.[Fiscal Year] AS VARCHAR(4))
                    + ' Q' + CAST(b.[Fiscal Quarter] AS VARCHAR(1))
                                                    AS [Fiscal Quarter Label],
                b.[Fiscal Year] * 10 + b.[Fiscal Quarter]
                                                    AS [Fiscal Quarter Sort]

            FROM Base b
        ) AS src ON tgt.[Date Key] = src.[Date Key]
        WHEN MATCHED THEN
            UPDATE SET
                tgt.[Date]                      = src.[Date],
                tgt.[Day]                       = src.[Day],
                tgt.[Day Of Week]               = src.[Day Of Week],
                tgt.[Weekday Name]              = src.[Weekday Name],
                tgt.[Weekday Name Short]        = src.[Weekday Name Short],
                tgt.[Weekday Name First Letter] = src.[Weekday Name First Letter],
                tgt.[Is Weekday]                = src.[Is Weekday],
                tgt.[Is Weekend]                = src.[Is Weekend],
                tgt.[Day Of Week In Month]      = src.[Day Of Week In Month],
                tgt.[Day Of Week In Year]       = src.[Day Of Week In Year],
                tgt.[Day Of Year]               = src.[Day Of Year],
                tgt.[Week Of Month]             = src.[Week Of Month],
                tgt.[Week Of Year]              = src.[Week Of Year],
                tgt.[First Date Of Week]        = src.[First Date Of Week],
                tgt.[Last Date Of Week]         = src.[Last Date Of Week],
                tgt.[Month]                     = src.[Month],
                tgt.[Month Name]                = src.[Month Name],
                tgt.[Month Name Short]          = src.[Month Name Short],
                tgt.[Month Name First Letter]   = src.[Month Name First Letter],
                tgt.[Month Year]                = src.[Month Year],
                tgt.[YYYYMM]                    = src.[YYYYMM],
                tgt.[Days In Month]             = src.[Days In Month],
                tgt.[First Date Of Month]       = src.[First Date Of Month],
                tgt.[Last Date Of Month]        = src.[Last Date Of Month],
                tgt.[Quarter]                   = src.[Quarter],
                tgt.[Quarter Name]              = src.[Quarter Name],
                tgt.[First Date Of Quarter]     = src.[First Date Of Quarter],
                tgt.[Last Date Of Quarter]      = src.[Last Date Of Quarter],
                tgt.[Year]                      = src.[Year],
                tgt.[Is Leap Year]              = src.[Is Leap Year],
                tgt.[First Date Of Year]        = src.[First Date Of Year],
                tgt.[Last Date Of Year]         = src.[Last Date Of Year],
                tgt.[Is Pay Week]               = src.[Is Pay Week],
                tgt.[Fiscal Year]               = src.[Fiscal Year],
                tgt.[Fiscal Year Label]         = src.[Fiscal Year Label],
                tgt.[Fiscal Year Label Short]   = src.[Fiscal Year Label Short],
                tgt.[Fiscal Month]              = src.[Fiscal Month],
                tgt.[Fiscal Period Label]       = src.[Fiscal Period Label],
                tgt.[Fiscal Period Sort]        = src.[Fiscal Period Sort],
                tgt.[Fiscal Quarter]            = src.[Fiscal Quarter],
                tgt.[Fiscal Quarter Name]       = src.[Fiscal Quarter Name],
                tgt.[Fiscal Quarter Label]      = src.[Fiscal Quarter Label],
                tgt.[Fiscal Quarter Sort]       = src.[Fiscal Quarter Sort]
        WHEN NOT MATCHED BY TARGET THEN
            INSERT (
                [Date Key], [Date],
                [Day], [Day Of Week], [Weekday Name], [Weekday Name Short],
                [Weekday Name First Letter], [Is Weekday], [Is Weekend],
                [Day Of Week In Month], [Day Of Week In Year], [Day Of Year],
                [Week Of Month], [Week Of Year], [First Date Of Week], [Last Date Of Week],
                [Month], [Month Name], [Month Name Short], [Month Name First Letter],
                [Month Year], [YYYYMM], [Days In Month],
                [First Date Of Month], [Last Date Of Month],
                [Quarter], [Quarter Name], [First Date Of Quarter], [Last Date Of Quarter],
                [Year], [Is Leap Year], [First Date Of Year], [Last Date Of Year],
                [Is Pay Week],
                [Fiscal Year], [Fiscal Year Label], [Fiscal Year Label Short],
                [Fiscal Month], [Fiscal Period Label], [Fiscal Period Sort],
                [Fiscal Quarter], [Fiscal Quarter Name], [Fiscal Quarter Label],
                [Fiscal Quarter Sort]
            )
            VALUES (
                src.[Date Key], src.[Date],
                src.[Day], src.[Day Of Week], src.[Weekday Name], src.[Weekday Name Short],
                src.[Weekday Name First Letter], src.[Is Weekday], src.[Is Weekend],
                src.[Day Of Week In Month], src.[Day Of Week In Year], src.[Day Of Year],
                src.[Week Of Month], src.[Week Of Year], src.[First Date Of Week], src.[Last Date Of Week],
                src.[Month], src.[Month Name], src.[Month Name Short], src.[Month Name First Letter],
                src.[Month Year], src.[YYYYMM], src.[Days In Month],
                src.[First Date Of Month], src.[Last Date Of Month],
                src.[Quarter], src.[Quarter Name], src.[First Date Of Quarter], src.[Last Date Of Quarter],
                src.[Year], src.[Is Leap Year], src.[First Date Of Year], src.[Last Date Of Year],
                src.[Is Pay Week],
                src.[Fiscal Year], src.[Fiscal Year Label], src.[Fiscal Year Label Short],
                src.[Fiscal Month], src.[Fiscal Period Label], src.[Fiscal Period Sort],
                src.[Fiscal Quarter], src.[Fiscal Quarter Name], src.[Fiscal Quarter Label],
                src.[Fiscal Quarter Sort]
            )
        OPTION (MAXRECURSION 0);  -- required: 18,628 rows exceeds default limit of 100

        -- ── Unknown member row (Date Key = -1) ──────────────────────────────
        IF NOT EXISTS (SELECT 1 FROM [Dimension].[Calendar] WHERE [Date Key] = -1)
        BEGIN
            INSERT INTO [Dimension].[Calendar] (
                [Date Key], [Date],
                [Day], [Day Of Week], [Weekday Name], [Weekday Name Short],
                [Weekday Name First Letter], [Is Weekday], [Is Weekend],
                [Day Of Week In Month], [Day Of Week In Year], [Day Of Year],
                [Week Of Month], [Week Of Year], [First Date Of Week], [Last Date Of Week],
                [Month], [Month Name], [Month Name Short], [Month Name First Letter],
                [Month Year], [YYYYMM], [Days In Month],
                [First Date Of Month], [Last Date Of Month],
                [Quarter], [Quarter Name], [First Date Of Quarter], [Last Date Of Quarter],
                [Year], [Is Leap Year], [First Date Of Year], [Last Date Of Year],
                [Is Pay Week],
                [Fiscal Year], [Fiscal Year Label], [Fiscal Year Label Short],
                [Fiscal Month], [Fiscal Period Label], [Fiscal Period Sort],
                [Fiscal Quarter], [Fiscal Quarter Name], [Fiscal Quarter Label],
                [Fiscal Quarter Sort]
            )
            VALUES (
                -1, NULL,
                NULL,NULL,'Unknown','Unk','U',NULL,NULL,NULL,NULL,NULL,
                NULL,NULL,NULL,NULL,
                NULL,'Unknown','Unk','U',
                'Unknown','N/A',NULL,NULL,NULL,
                NULL,'N/A',NULL,NULL,
                NULL,NULL,NULL,NULL,
                NULL,
                NULL,'Unknown','Unk',
                NULL,'Unknown',NULL,
                NULL,'N/A','Unknown',NULL
            );
        END

    END TRY
    BEGIN CATCH
        DECLARE @Msg NVARCHAR(2048) = ERROR_MESSAGE();
        RAISERROR(@Msg, 16, 1);
    END CATCH
END
GO

-- Run the population immediately on first deploy
EXEC [Dimension].[PopulateCalendar];
GO
```

---

## 4. SSAS View — Holiday Integration and Current-Date Columns

`SSAS.v_Calendar` is the **partition source** for the SSAS Tabular `'Calendar'` table.
It adds holiday flags and current-date columns computed at query time from `GETDATE()` —
no nightly refresh SP needed.

```sql
CREATE OR ALTER VIEW [SSAS].[v_Calendar]
AS
    SELECT
        c.[Date Key],
        c.[Date],

        -- ── Day ─────────────────────────────────────────────────────────────
        c.[Day],
        c.[Day Of Week],
        c.[Weekday Name],
        c.[Weekday Name Short],
        c.[Weekday Name First Letter],
        c.[Is Weekday],
        c.[Is Weekend],
        c.[Day Of Week In Month],
        c.[Day Of Week In Year],
        c.[Day Of Year],

        -- ── Holiday (joined from Dimension.StatHolidays) ─────────────────────
        CAST(CASE WHEN h.[Stat Holiday Date Key] IS NOT NULL THEN 1 ELSE 0 END
             AS BIT)                                    AS [Is Stat Holiday],
        h.[Stat Holiday Name],

        -- Is Working Day: weekday AND not a stat holiday
        CAST(
            CASE WHEN c.[Is Weekday] = 1
                  AND h.[Stat Holiday Date Key] IS NULL
                 THEN 1 ELSE 0 END
        AS BIT)                                         AS [Is Working Day],

        -- ── Relative / current-date columns (computed from GETDATE()) ────────
        CAST(CASE WHEN c.[Date] = CAST(GETDATE() AS DATE) THEN 1 ELSE 0 END
             AS BIT)                                    AS [Is Today],

        CAST(CASE WHEN c.[YYYYMM] =
                       CAST(YEAR(GETDATE()) AS VARCHAR(4))
                       + RIGHT('0' + CAST(MONTH(GETDATE()) AS VARCHAR(2)), 2)
                  THEN 1 ELSE 0 END
             AS BIT)                                    AS [Is Current Month],

        CAST(CASE WHEN c.[Year] = YEAR(GETDATE()) THEN 1 ELSE 0 END
             AS BIT)                                    AS [Is Current Calendar Year],

        CAST(CASE WHEN c.[Fiscal Year] =
                       CASE WHEN MONTH(GETDATE()) >= 4
                            THEN YEAR(GETDATE())
                            ELSE YEAR(GETDATE()) - 1
                       END
                  THEN 1 ELSE 0 END
             AS BIT)                                    AS [Is Current Fiscal Year],

        DATEDIFF(DAY, CAST(GETDATE() AS DATE), c.[Date])
                                                        AS [Relative Day],
        (YEAR(c.[Date]) - YEAR(GETDATE())) * 12
            + (MONTH(c.[Date]) - MONTH(GETDATE()))      AS [Relative Month],

        c.[Fiscal Year] - (
            CASE WHEN MONTH(GETDATE()) >= 4
                 THEN YEAR(GETDATE())
                 ELSE YEAR(GETDATE()) - 1
            END
        )                                               AS [Relative Fiscal Year],

        -- ── Week ─────────────────────────────────────────────────────────────
        c.[Week Of Month],
        c.[Week Of Year],
        c.[First Date Of Week],
        c.[Last Date Of Week],
        c.[Is Pay Week],

        -- ── Month ────────────────────────────────────────────────────────────
        c.[Month],
        c.[Month Name],
        c.[Month Name Short],
        c.[Month Name First Letter],
        c.[Month Year],
        c.[YYYYMM],
        c.[Days In Month],
        c.[First Date Of Month],
        c.[Last Date Of Month],

        -- ── Quarter ──────────────────────────────────────────────────────────
        c.[Quarter],
        c.[Quarter Name],
        c.[First Date Of Quarter],
        c.[Last Date Of Quarter],

        -- ── Calendar Year ────────────────────────────────────────────────────
        c.[Year],
        c.[Is Leap Year],
        c.[First Date Of Year],
        c.[Last Date Of Year],

        -- ── Fiscal Year ──────────────────────────────────────────────────────
        c.[Fiscal Year],
        c.[Fiscal Year Label],
        c.[Fiscal Year Label Short],
        c.[Fiscal Month],
        c.[Fiscal Period Label],
        c.[Fiscal Period Sort],
        c.[Fiscal Quarter],
        c.[Fiscal Quarter Name],
        c.[Fiscal Quarter Label],
        c.[Fiscal Quarter Sort]

    FROM [Dimension].[Calendar] c
    LEFT JOIN [Dimension].[StatHolidays] h
        ON c.[Date Key] = h.[Stat Holiday Date Key]
        AND h.[Is Deleted] = 0;
GO
```

---

## 5. SSAS Tabular Configuration Notes

### Date table marking
```
// Tabular Editor 2 C# script — mark Calendar as Date Table
Model.Tables["Calendar"].DataCategory = "Time";
Model.Tables["Calendar"].Columns["Date"].IsKey = true;
```

### Hidden columns (infrastructure — not shown in field list)
```
[Date Key], [Day Of Week], [Month], [Quarter],
[First Date Of Week], [Last Date Of Week],
[First Date Of Month], [Last Date Of Month],
[First Date Of Quarter], [Last Date Of Quarter],
[First Date Of Year], [Last Date Of Year],
[YYYYMM], [Fiscal Period Sort], [Fiscal Quarter Sort],
[Day Of Week In Month], [Day Of Week In Year],
[Weekday Name First Letter], [Month Name First Letter]
```

### Sort-By column assignments
| Column | Sort By |
|---|---|
| `[Weekday Name]` | `[Day Of Week]` |
| `[Weekday Name Short]` | `[Day Of Week]` |
| `[Month Name]` | `[Month]` |
| `[Month Name Short]` | `[Month]` |
| `[Quarter Name]` | `[Quarter]` |
| `[Month Year]` | `[YYYYMM]` |
| `[Fiscal Period Label]` | `[Fiscal Period Sort]` |
| `[Fiscal Quarter Label]` | `[Fiscal Quarter Sort]` |
| `[Fiscal Quarter Name]` | `[Fiscal Quarter]` |
| `[Fiscal Year Label]` | `[Fiscal Year]` |
| `[Fiscal Year Label Short]` | `[Fiscal Year]` |

### Display folders (recommended)
```
Calendar\Day          — Day, Day Of Week, Weekday Name, Weekday Name Short,
                        Is Weekday, Is Weekend, Day Of Year
Calendar\Week         — Week Of Month, Week Of Year, First/Last Date Of Week, Is Pay Week
Calendar\Month        — Month, Month Name, Month Name Short, Month Year, YYYYMM,
                        Days In Month, First/Last Date Of Month
Calendar\Quarter      — Quarter, Quarter Name, First/Last Date Of Quarter
Calendar\Year         — Year, Is Leap Year, First/Last Date Of Year
Fiscal\Fiscal Year    — Fiscal Year, Fiscal Year Label, Fiscal Year Label Short,
                        Is Current Fiscal Year, Relative Fiscal Year
Fiscal\Fiscal Period  — Fiscal Month, Fiscal Period Label
Fiscal\Fiscal Quarter — Fiscal Quarter, Fiscal Quarter Name, Fiscal Quarter Label
Flags                 — Is Stat Holiday, Stat Holiday Name, Is Working Day, Is Today,
                        Is Current Month, Is Current Calendar Year
Relative              — Relative Day, Relative Month, Relative Fiscal Year
```

---

## 6. StatHolidayGenerator — Update Notes

The existing project (`src/dev/Database/StatHolidayGenerator`) is outdated:

| Issue | Detail |
|---|---|
| Framework | .NET 6 → upgrade to .NET 8 |
| Nager.Date version | 1.46.0 → upgrade to 3.x (API changed; `HolidayClient` no longer used; use `DateSystem` static class) |
| Date range | Starts 2023, 20 years (→ 2042) → start 2000, cover to 2050 |
| Hardcoded output path | `C:\Temp\holidays.sql` → accept output path as command-line arg |
| Target table | `dbo.stat_holiday` → `Dimension.StatHolidays` with updated schema |
| BC conventions | Already correct: includes Easter Monday + Boxing Day; handles in-lieu |

**Updated Nager.Date 3.x usage pattern:**
```csharp
// Nager.Date 3.x uses DateSystem static class instead of HolidayClient
var holidays = DateSystem.GetPublicHolidays(year, CountryCode.CA);
// Filter for BC: holiday.Counties == null || holiday.Counties.Contains("CA-BC")
```

**Target MERGE statement for updated output:**
```sql
MERGE [Dimension].[StatHolidays] AS Target
USING @tbl AS Source
ON Source.holidayDate = Target.[Stat Holiday Date]
WHEN NOT MATCHED BY TARGET THEN
    INSERT ([Stat Holiday Date], [Stat Holiday Date Key], [Stat Holiday Name])
    VALUES (
        Source.holidayDate,
        YEAR(Source.holidayDate)*10000
            + MONTH(Source.holidayDate)*100
            + DAY(Source.holidayDate),
        Source.holidayName
    );
```

---

## 7. Fiscal Year Quick Reference

| Calendar Date | FY | Fiscal Period | Fiscal Quarter |
|---|---|---|---|
| Apr 1, 2024 | FY2024 | P01 | Q1 |
| Jun 30, 2024 | FY2024 | P03 | Q1 |
| Jul 1, 2024 | FY2024 | P04 | Q2 |
| Sep 30, 2024 | FY2024 | P06 | Q2 |
| Oct 1, 2024 | FY2024 | P07 | Q3 |
| Dec 31, 2024 | FY2024 | P09 | Q3 |
| Jan 1, 2025 | FY2024 | P10 | Q4 |
| Mar 31, 2025 | FY2024 | P12 | Q4 |
| Apr 1, 2025 | FY2025 | P01 | Q1 |

---

## 8. Deployment Checklist

- [ ] `Dimension.StatHolidays` table created (Section 1)
- [ ] `Dimension.Calendar` table created (Section 2)
- [ ] `Dimension.PopulateCalendar` SP created and executed — verify row count:
  ```sql
  SELECT COUNT(*) FROM [Dimension].[Calendar];  -- expected: 18629 (18628 dates + 1 unknown)
  ```
- [ ] Fiscal year boundary spot-check:
  ```sql
  SELECT [Date Key], [Date], [Fiscal Year], [Fiscal Year Label], [Fiscal Month], [Fiscal Quarter]
  FROM [Dimension].[Calendar]
  WHERE [Date] IN ('2024-03-31','2024-04-01','2024-12-31','2025-01-01','2025-03-31','2025-04-01')
  ORDER BY [Date Key];
  -- Expected FY: 2023, 2024, 2024, 2024, 2024, 2025
  ```
- [ ] Run updated StatHolidayGenerator (Section 6) and execute its output against `Dimension.StatHolidays`
- [ ] `SSAS.v_Calendar` view created (Section 4) — test:
  ```sql
  SELECT TOP 5 [Date], [Is Stat Holiday], [Stat Holiday Name], [Is Working Day]
  FROM [SSAS].[v_Calendar]
  WHERE [Date] BETWEEN '2025-04-01' AND '2025-04-25'
    AND [Is Stat Holiday] = 1;
  ```
- [ ] SSAS Tabular model: partition source → `SELECT * FROM [SSAS].[v_Calendar]`
- [ ] Date table marking applied in Tabular Editor (Section 5)
- [ ] Sort-by columns assigned (Section 5)
- [ ] Hidden columns set and display folders applied (Section 5)
- [ ] Extended properties generated via Mode C and deployed in DACPAC post-deploy script
