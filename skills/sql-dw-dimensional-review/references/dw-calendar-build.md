# DW Calendar Dimension Build Reference

Reference scripts for building and maintaining the `Dimension.Calendar` table and its
companion objects. This is a **one-time build** (idempotent — safe to re-run).

**Design decisions:**
- Date range: 2000-01-01 → 2050-12-31
- Fiscal year: Apr 1 – Mar 31 (FY2025 = Apr 2024 – Mar 2025)
- Week numbering: Sunday-start (US-style, `DATEFIRST 7`)
- Stat holidays: stored in a separate `Dimension.StatHolidays` table populated by the
  Nager.Date C# project; integrated into Calendar via `SSAS.v_Calendar` view
- Relative/current-date columns (`Is Current Day`, `Is Working Day`, etc.) are computed
  in the view using `GETDATE()` — **never stored in the table** (avoids nightly update SP)
- Unknown member row: `Date Key = -1` (aligns with `EndDateKey = -1` convention for open events)

---

## 1. Dimension.StatHolidays Table

The stat holiday data is produced by a separate C# project using Nager.Date plus custom
logic. That project outputs a SQL `INSERT` script targeting this table.

```sql
-- Dimension.StatHolidays
IF NOT EXISTS (
    SELECT 1 FROM sys.tables t
    JOIN sys.schemas s ON t.schema_id = s.schema_id
    WHERE s.name = 'Dimension' AND t.name = 'StatHolidays'
)
BEGIN
    CREATE TABLE [Dimension].[StatHolidays] (
        [Stat Holiday Key]  INT           NOT NULL IDENTITY(1, 1),
        [Date Key]          INT           NOT NULL,   -- FK to Dimension.Calendar.[Date Key]
        [Date]              DATE          NOT NULL,
        [Holiday Name]      NVARCHAR(100) NOT NULL,
        [Province Code]     CHAR(2)       NULL,       -- NULL = national; 'ON', 'BC', etc.
        [Country Code]      CHAR(2)       NOT NULL CONSTRAINT [DF_StatHolidays_Country] DEFAULT ('CA'),
        CONSTRAINT [PK_Dimension_StatHolidays] PRIMARY KEY CLUSTERED ([Stat Holiday Key]),
        CONSTRAINT [UQ_StatHolidays_DateKey_Province]
            UNIQUE NONCLUSTERED ([Date Key], [Province Code])
    );

    CREATE NONCLUSTERED INDEX [NIX_Dimension_StatHolidays_DateKey]
        ON [Dimension].[StatHolidays] ([Date Key])
        INCLUDE ([Province Code]);
END
GO
```

**Notes:**
- Run the Nager.Date output script against `Dimension.StatHolidays` **after** this table
  is created and **after** `Dimension.Calendar` is populated (the NCI join relies on
  `Date Key` matching calendar rows).
- `Province Code = NULL` means the holiday applies nationally.
- To update holidays for a new year, run the Nager.Date script with `IF NOT EXISTS`
  guards or use a `MERGE` on `([Date Key], [Province Code])`.

---

## 2. Dimension.Calendar Table

```sql
-- Drop and recreate extended properties will be handled by post-deploy scripts.
IF NOT EXISTS (
    SELECT 1 FROM sys.tables t
    JOIN sys.schemas s ON t.schema_id = s.schema_id
    WHERE s.name = 'Dimension' AND t.name = 'Calendar'
)
BEGIN
    CREATE TABLE [Dimension].[Calendar] (
        -- ── Key ─────────────────────────────────────────────────────────────────
        [Date Key]                  INT           NOT NULL,   -- YYYYMMDD; -1 = Unknown
        [Date]                      DATE          NULL,

        -- ── Calendar Day ─────────────────────────────────────────────────────
        [Day Of Month]              TINYINT       NULL,       -- 1–31
        [Day Of Week]               TINYINT       NULL,       -- 1=Sunday … 7=Saturday
        [Day Name]                  VARCHAR(10)   NULL,       -- 'Sunday' … 'Saturday'
        [Day Name Short]            CHAR(3)       NULL,       -- 'Sun' … 'Sat'
        [Is Weekday]                BIT           NULL,       -- 1=Mon–Fri
        [Day Of Year]               SMALLINT      NULL,       -- 1–366

        -- ── Calendar Week ────────────────────────────────────────────────────
        [Week Of Year]              TINYINT       NULL,       -- 1–53, Sunday-start
        [Week Start Date Key]       INT           NULL,       -- YYYYMMDD of Sunday
        [Week End Date Key]         INT           NULL,       -- YYYYMMDD of Saturday

        -- ── Calendar Month ───────────────────────────────────────────────────
        [Month]                     TINYINT       NULL,       -- 1–12
        [Month Name]                VARCHAR(10)   NULL,       -- 'January' …
        [Month Name Short]          CHAR(3)       NULL,       -- 'Jan' …
        [Month Start Date Key]      INT           NULL,       -- YYYYMMDD of 1st of month
        [Month End Date Key]        INT           NULL,       -- YYYYMMDD of last of month
        [Year Month]                INT           NULL,       -- YYYYMM

        -- ── Calendar Quarter ─────────────────────────────────────────────────
        [Quarter]                   TINYINT       NULL,       -- 1–4
        [Quarter Label]             CHAR(2)       NULL,       -- 'Q1' …
        [Quarter Start Date Key]    INT           NULL,
        [Quarter End Date Key]      INT           NULL,

        -- ── Calendar Year ────────────────────────────────────────────────────
        [Year]                      SMALLINT      NULL,       -- e.g. 2025
        [Year Label]                CHAR(4)       NULL,       -- '2025'
        [Year Start Date Key]       INT           NULL,       -- YYYYMMDD of Jan 1
        [Year End Date Key]         INT           NULL,       -- YYYYMMDD of Dec 31

        -- ── Fiscal (Apr 1 – Mar 31) ──────────────────────────────────────────
        -- FY2025 = Apr 2024 – Mar 2025
        -- Fiscal Period 1 = April, Period 12 = March
        [Fiscal Year]               SMALLINT      NULL,       -- e.g. 2025
        [Fiscal Year Label]         CHAR(6)       NULL,       -- 'FY2025'
        [Fiscal Year Label Short]   CHAR(4)       NULL,       -- 'FY25'
        [Fiscal Year Start Date Key] INT          NULL,       -- YYYYMMDD of Apr 1
        [Fiscal Year End Date Key]   INT          NULL,       -- YYYYMMDD of Mar 31

        [Fiscal Period]             TINYINT       NULL,       -- 1–12 (Apr=1 … Mar=12)
        [Fiscal Period Label]       VARCHAR(10)   NULL,       -- 'FY2025 P1'
        [Fiscal Period Sort]        SMALLINT      NULL,       -- YYYY*100 + FP (for sorting)

        [Fiscal Quarter]            TINYINT       NULL,       -- 1–4
        [Fiscal Quarter Label]      VARCHAR(10)   NULL,       -- 'FY2025 Q1'
        [Fiscal Quarter Sort]       SMALLINT      NULL,       -- YYYY*10 + FQ (for sorting)

        CONSTRAINT [PK_Dimension_Calendar] PRIMARY KEY CLUSTERED ([Date Key])
    );
END
GO
```

---

## 3. Calendar Population Stored Procedure

```sql
CREATE OR ALTER PROCEDURE [Dimension].[PopulateCalendar]
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;
    SET DATEFIRST 7;  -- Sunday = 1 (US week convention)

    BEGIN TRY
        -- ── Generate date spine ──────────────────────────────────────────────
        ;WITH DateSpine AS (
            SELECT CAST('2000-01-01' AS DATE) AS [Date]
            UNION ALL
            SELECT DATEADD(DAY, 1, [Date])
            FROM DateSpine
            WHERE [Date] < '2050-12-31'
        ),
        Calculated AS (
            SELECT
                d.[Date],
                YEAR(d.[Date]) * 10000 + MONTH(d.[Date]) * 100 + DAY(d.[Date])
                                                            AS [Date Key],

                -- ── Day ─────────────────────────────────────────────────────
                DAY(d.[Date])                               AS [Day Of Month],
                DATEPART(WEEKDAY, d.[Date])                 AS [Day Of Week],  -- 1=Sun, DATEFIRST 7
                DATENAME(WEEKDAY, d.[Date])                 AS [Day Name],
                LEFT(DATENAME(WEEKDAY, d.[Date]), 3)        AS [Day Name Short],
                CASE WHEN DATEPART(WEEKDAY, d.[Date]) BETWEEN 2 AND 6
                     THEN CAST(1 AS BIT) ELSE CAST(0 AS BIT) END
                                                            AS [Is Weekday],
                DATEPART(DAYOFYEAR, d.[Date])               AS [Day Of Year],

                -- ── Week (Sunday-start) ─────────────────────────────────────
                DATEPART(WEEK, d.[Date])                    AS [Week Of Year],
                DATEADD(DAY, -(DATEPART(WEEKDAY, d.[Date]) - 1), d.[Date])
                                                            AS [Week Start Date],
                DATEADD(DAY,  (7 - DATEPART(WEEKDAY, d.[Date])), d.[Date])
                                                            AS [Week End Date],

                -- ── Month ───────────────────────────────────────────────────
                MONTH(d.[Date])                             AS [Month],
                DATENAME(MONTH, d.[Date])                   AS [Month Name],
                LEFT(DATENAME(MONTH, d.[Date]), 3)          AS [Month Name Short],
                DATEFROMPARTS(YEAR(d.[Date]), MONTH(d.[Date]), 1)
                                                            AS [Month Start Date],
                EOMONTH(d.[Date])                           AS [Month End Date],
                YEAR(d.[Date]) * 100 + MONTH(d.[Date])      AS [Year Month],

                -- ── Quarter ─────────────────────────────────────────────────
                DATEPART(QUARTER, d.[Date])                 AS [Quarter],

                -- ── Year ────────────────────────────────────────────────────
                YEAR(d.[Date])                              AS [Year],

                -- ── Fiscal Year (Apr 1 – Mar 31) ────────────────────────────
                -- FY = calendar year + 1 when month >= April; else calendar year
                CASE WHEN MONTH(d.[Date]) >= 4
                     THEN YEAR(d.[Date]) + 1
                     ELSE YEAR(d.[Date])
                END                                         AS [Fiscal Year],

                -- Fiscal Period: Apr=1, May=2, ..., Dec=9, Jan=10, Feb=11, Mar=12
                CASE WHEN MONTH(d.[Date]) >= 4
                     THEN MONTH(d.[Date]) - 3
                     ELSE MONTH(d.[Date]) + 9
                END                                         AS [Fiscal Period]

            FROM DateSpine d
        ),
        WithDerived AS (
            SELECT
                c.*,
                -- Quarter label
                'Q' + CAST(c.[Quarter] AS VARCHAR(1))       AS [Quarter Label],
                -- Quarter start / end date keys
                DATEFROMPARTS(c.[Year], (c.[Quarter] - 1) * 3 + 1, 1)
                                                            AS [Quarter Start Date],
                EOMONTH(DATEFROMPARTS(c.[Year], c.[Quarter] * 3, 1))
                                                            AS [Quarter End Date],

                -- Fiscal quarter from fiscal period
                CEILING(CAST(c.[Fiscal Period] AS FLOAT) / 3.0)
                                                            AS [Fiscal Quarter],

                -- Fiscal year start / end dates
                DATEFROMPARTS(c.[Fiscal Year] - 1, 4, 1)   AS [Fiscal Year Start Date],
                DATEFROMPARTS(c.[Fiscal Year],     3, 31)   AS [Fiscal Year End Date]

            FROM Calculated c
        )

        -- ── Upsert into Dimension.Calendar ──────────────────────────────────
        MERGE [Dimension].[Calendar] AS tgt
        USING (
            SELECT
                w.[Date Key],
                w.[Date],

                -- Day
                CAST(w.[Day Of Month]   AS TINYINT)     AS [Day Of Month],
                CAST(w.[Day Of Week]    AS TINYINT)     AS [Day Of Week],
                w.[Day Name],
                w.[Day Name Short],
                w.[Is Weekday],
                CAST(w.[Day Of Year]    AS SMALLINT)    AS [Day Of Year],

                -- Week
                CAST(w.[Week Of Year]   AS TINYINT)     AS [Week Of Year],
                YEAR(w.[Week Start Date]) * 10000 + MONTH(w.[Week Start Date]) * 100 + DAY(w.[Week Start Date])
                                                        AS [Week Start Date Key],
                YEAR(w.[Week End Date])   * 10000 + MONTH(w.[Week End Date])   * 100 + DAY(w.[Week End Date])
                                                        AS [Week End Date Key],

                -- Month
                CAST(w.[Month]          AS TINYINT)     AS [Month],
                w.[Month Name],
                w.[Month Name Short],
                YEAR(w.[Month Start Date]) * 10000 + MONTH(w.[Month Start Date]) * 100 + DAY(w.[Month Start Date])
                                                        AS [Month Start Date Key],
                YEAR(w.[Month End Date])   * 10000 + MONTH(w.[Month End Date])   * 100 + DAY(w.[Month End Date])
                                                        AS [Month End Date Key],
                w.[Year Month],

                -- Quarter
                CAST(w.[Quarter]        AS TINYINT)     AS [Quarter],
                w.[Quarter Label],
                YEAR(w.[Quarter Start Date]) * 10000 + MONTH(w.[Quarter Start Date]) * 100 + DAY(w.[Quarter Start Date])
                                                        AS [Quarter Start Date Key],
                YEAR(w.[Quarter End Date])   * 10000 + MONTH(w.[Quarter End Date])   * 100 + DAY(w.[Quarter End Date])
                                                        AS [Quarter End Date Key],

                -- Year
                CAST(w.[Year]           AS SMALLINT)    AS [Year],
                CAST(w.[Year]           AS CHAR(4))     AS [Year Label],
                w.[Year] * 10000 + 101                  AS [Year Start Date Key],
                w.[Year] * 10000 + 1231                 AS [Year End Date Key],

                -- Fiscal Year
                CAST(w.[Fiscal Year]    AS SMALLINT)    AS [Fiscal Year],
                'FY' + CAST(w.[Fiscal Year] AS VARCHAR(4))
                                                        AS [Fiscal Year Label],
                'FY' + RIGHT(CAST(w.[Fiscal Year] AS VARCHAR(4)), 2)
                                                        AS [Fiscal Year Label Short],
                (w.[Fiscal Year] - 1) * 10000 + 401    AS [Fiscal Year Start Date Key],
                w.[Fiscal Year] * 10000 + 331           AS [Fiscal Year End Date Key],

                -- Fiscal Period
                CAST(w.[Fiscal Period]  AS TINYINT)     AS [Fiscal Period],
                'FY' + CAST(w.[Fiscal Year] AS VARCHAR(4))
                    + ' P' + RIGHT('0' + CAST(w.[Fiscal Period] AS VARCHAR(2)), 2)
                                                        AS [Fiscal Period Label],
                CAST(w.[Fiscal Year] * 100 + w.[Fiscal Period] AS SMALLINT)
                                                        AS [Fiscal Period Sort],

                -- Fiscal Quarter
                CAST(w.[Fiscal Quarter] AS TINYINT)     AS [Fiscal Quarter],
                'FY' + CAST(w.[Fiscal Year] AS VARCHAR(4))
                    + ' Q' + CAST(w.[Fiscal Quarter] AS VARCHAR(1))
                                                        AS [Fiscal Quarter Label],
                CAST(w.[Fiscal Year] * 10 + w.[Fiscal Quarter] AS SMALLINT)
                                                        AS [Fiscal Quarter Sort]

            FROM WithDerived w
        ) AS src ON tgt.[Date Key] = src.[Date Key]
        WHEN MATCHED THEN
            UPDATE SET
                tgt.[Date]                      = src.[Date],
                tgt.[Day Of Month]              = src.[Day Of Month],
                tgt.[Day Of Week]               = src.[Day Of Week],
                tgt.[Day Name]                  = src.[Day Name],
                tgt.[Day Name Short]            = src.[Day Name Short],
                tgt.[Is Weekday]                = src.[Is Weekday],
                tgt.[Day Of Year]               = src.[Day Of Year],
                tgt.[Week Of Year]              = src.[Week Of Year],
                tgt.[Week Start Date Key]       = src.[Week Start Date Key],
                tgt.[Week End Date Key]         = src.[Week End Date Key],
                tgt.[Month]                     = src.[Month],
                tgt.[Month Name]                = src.[Month Name],
                tgt.[Month Name Short]          = src.[Month Name Short],
                tgt.[Month Start Date Key]      = src.[Month Start Date Key],
                tgt.[Month End Date Key]        = src.[Month End Date Key],
                tgt.[Year Month]                = src.[Year Month],
                tgt.[Quarter]                   = src.[Quarter],
                tgt.[Quarter Label]             = src.[Quarter Label],
                tgt.[Quarter Start Date Key]    = src.[Quarter Start Date Key],
                tgt.[Quarter End Date Key]      = src.[Quarter End Date Key],
                tgt.[Year]                      = src.[Year],
                tgt.[Year Label]                = src.[Year Label],
                tgt.[Year Start Date Key]       = src.[Year Start Date Key],
                tgt.[Year End Date Key]         = src.[Year End Date Key],
                tgt.[Fiscal Year]               = src.[Fiscal Year],
                tgt.[Fiscal Year Label]         = src.[Fiscal Year Label],
                tgt.[Fiscal Year Label Short]   = src.[Fiscal Year Label Short],
                tgt.[Fiscal Year Start Date Key]= src.[Fiscal Year Start Date Key],
                tgt.[Fiscal Year End Date Key]  = src.[Fiscal Year End Date Key],
                tgt.[Fiscal Period]             = src.[Fiscal Period],
                tgt.[Fiscal Period Label]       = src.[Fiscal Period Label],
                tgt.[Fiscal Period Sort]        = src.[Fiscal Period Sort],
                tgt.[Fiscal Quarter]            = src.[Fiscal Quarter],
                tgt.[Fiscal Quarter Label]      = src.[Fiscal Quarter Label],
                tgt.[Fiscal Quarter Sort]       = src.[Fiscal Quarter Sort]
        WHEN NOT MATCHED BY TARGET THEN
            INSERT (
                [Date Key], [Date],
                [Day Of Month], [Day Of Week], [Day Name], [Day Name Short],
                [Is Weekday], [Day Of Year],
                [Week Of Year], [Week Start Date Key], [Week End Date Key],
                [Month], [Month Name], [Month Name Short],
                [Month Start Date Key], [Month End Date Key], [Year Month],
                [Quarter], [Quarter Label],
                [Quarter Start Date Key], [Quarter End Date Key],
                [Year], [Year Label], [Year Start Date Key], [Year End Date Key],
                [Fiscal Year], [Fiscal Year Label], [Fiscal Year Label Short],
                [Fiscal Year Start Date Key], [Fiscal Year End Date Key],
                [Fiscal Period], [Fiscal Period Label], [Fiscal Period Sort],
                [Fiscal Quarter], [Fiscal Quarter Label], [Fiscal Quarter Sort]
            )
            VALUES (
                src.[Date Key], src.[Date],
                src.[Day Of Month], src.[Day Of Week], src.[Day Name], src.[Day Name Short],
                src.[Is Weekday], src.[Day Of Year],
                src.[Week Of Year], src.[Week Start Date Key], src.[Week End Date Key],
                src.[Month], src.[Month Name], src.[Month Name Short],
                src.[Month Start Date Key], src.[Month End Date Key], src.[Year Month],
                src.[Quarter], src.[Quarter Label],
                src.[Quarter Start Date Key], src.[Quarter End Date Key],
                src.[Year], src.[Year Label], src.[Year Start Date Key], src.[Year End Date Key],
                src.[Fiscal Year], src.[Fiscal Year Label], src.[Fiscal Year Label Short],
                src.[Fiscal Year Start Date Key], src.[Fiscal Year End Date Key],
                src.[Fiscal Period], src.[Fiscal Period Label], src.[Fiscal Period Sort],
                src.[Fiscal Quarter], src.[Fiscal Quarter Label], src.[Fiscal Quarter Sort]
            );
        -- Note: no WHEN NOT MATCHED BY SOURCE — never delete calendar rows while fact FK constraints exist

        -- ── Insert unknown member row ────────────────────────────────────────
        IF NOT EXISTS (SELECT 1 FROM [Dimension].[Calendar] WHERE [Date Key] = -1)
        BEGIN
            INSERT INTO [Dimension].[Calendar] (
                [Date Key], [Date],
                [Day Of Month], [Day Of Week], [Day Name], [Day Name Short],
                [Is Weekday], [Day Of Year],
                [Week Of Year], [Week Start Date Key], [Week End Date Key],
                [Month], [Month Name], [Month Name Short],
                [Month Start Date Key], [Month End Date Key], [Year Month],
                [Quarter], [Quarter Label],
                [Quarter Start Date Key], [Quarter End Date Key],
                [Year], [Year Label], [Year Start Date Key], [Year End Date Key],
                [Fiscal Year], [Fiscal Year Label], [Fiscal Year Label Short],
                [Fiscal Year Start Date Key], [Fiscal Year End Date Key],
                [Fiscal Period], [Fiscal Period Label], [Fiscal Period Sort],
                [Fiscal Quarter], [Fiscal Quarter Label], [Fiscal Quarter Sort]
            )
            VALUES (
                -1, NULL,
                NULL, NULL, 'Unknown', 'Unk',
                NULL, NULL,
                NULL, NULL, NULL,
                NULL, 'Unknown', 'Unk',
                NULL, NULL, NULL,
                NULL, 'Unknown',
                NULL, NULL,
                NULL, 'Unknown', NULL, NULL,
                NULL, 'Unknown', 'Unk',
                NULL, NULL,
                NULL, 'Unknown', NULL,
                NULL, 'Unknown', NULL
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

**Recursion depth note:** The CTE with `OPTION (MAXRECURSION 0)` may be needed for the
date spine. Add it to the final SELECT if SQL Server raises "maximum recursion 100 exceeded":

```sql
-- Append to the final MERGE ... statement:
OPTION (MAXRECURSION 0);
```

---

## 4. SSAS View — Including Holiday and Relative Date Columns

The `SSAS.v_Calendar` view is the **partition source** for the SSAS Tabular 'Calendar'
table. It adds stat holiday integration and current-date columns computed at query time.

```sql
CREATE OR ALTER VIEW [SSAS].[v_Calendar]
AS
    SELECT
        c.[Date Key],
        c.[Date],

        -- ── Day ─────────────────────────────────────────────────────────────
        c.[Day Of Month],
        c.[Day Of Week],
        c.[Day Name],
        c.[Day Name Short],
        c.[Is Weekday],
        c.[Day Of Year],

        -- ── Holiday integration ──────────────────────────────────────────────
        -- Is Stat Holiday: national or any-province match
        CAST(
            CASE WHEN h.[Date Key] IS NOT NULL THEN 1 ELSE 0 END
        AS BIT)                                             AS [Is Stat Holiday],

        -- Is Working Day: weekday AND not a stat holiday
        CAST(
            CASE WHEN c.[Is Weekday] = 1
                  AND h.[Date Key] IS NULL
                 THEN 1 ELSE 0 END
        AS BIT)                                             AS [Is Working Day],

        -- ── Relative / Current (computed from GETDATE()) ─────────────────────
        CAST(
            CASE WHEN c.[Date] = CAST(GETDATE() AS DATE) THEN 1 ELSE 0 END
        AS BIT)                                             AS [Is Current Day],

        CAST(
            CASE WHEN c.[Year Month] = YEAR(GETDATE()) * 100 + MONTH(GETDATE())
                 THEN 1 ELSE 0 END
        AS BIT)                                             AS [Is Current Month],

        CAST(
            CASE WHEN c.[Year] = YEAR(GETDATE())
                 THEN 1 ELSE 0 END
        AS BIT)                                             AS [Is Current Calendar Year],

        CAST(
            CASE WHEN c.[Fiscal Year] =
                     CASE WHEN MONTH(GETDATE()) >= 4
                          THEN YEAR(GETDATE()) + 1
                          ELSE YEAR(GETDATE())
                     END
                 THEN 1 ELSE 0 END
        AS BIT)                                             AS [Is Current Fiscal Year],

        DATEDIFF(DAY, CAST(GETDATE() AS DATE), c.[Date])   AS [Relative Day],

        (YEAR(c.[Date]) - YEAR(GETDATE())) * 12
            + (MONTH(c.[Date]) - MONTH(GETDATE()))          AS [Relative Month],

        c.[Fiscal Year] - (
            CASE WHEN MONTH(GETDATE()) >= 4
                 THEN YEAR(GETDATE()) + 1
                 ELSE YEAR(GETDATE())
            END
        )                                                   AS [Relative Fiscal Year],

        -- ── Week ─────────────────────────────────────────────────────────────
        c.[Week Of Year],
        c.[Week Start Date Key],
        c.[Week End Date Key],

        -- ── Month ────────────────────────────────────────────────────────────
        c.[Month],
        c.[Month Name],
        c.[Month Name Short],
        c.[Month Start Date Key],
        c.[Month End Date Key],
        c.[Year Month],

        -- ── Quarter ──────────────────────────────────────────────────────────
        c.[Quarter],
        c.[Quarter Label],
        c.[Quarter Start Date Key],
        c.[Quarter End Date Key],

        -- ── Calendar Year ────────────────────────────────────────────────────
        c.[Year],
        c.[Year Label],
        c.[Year Start Date Key],
        c.[Year End Date Key],

        -- ── Fiscal Year ──────────────────────────────────────────────────────
        c.[Fiscal Year],
        c.[Fiscal Year Label],
        c.[Fiscal Year Label Short],
        c.[Fiscal Year Start Date Key],
        c.[Fiscal Year End Date Key],

        -- ── Fiscal Period ────────────────────────────────────────────────────
        c.[Fiscal Period],
        c.[Fiscal Period Label],
        c.[Fiscal Period Sort],

        -- ── Fiscal Quarter ───────────────────────────────────────────────────
        c.[Fiscal Quarter],
        c.[Fiscal Quarter Label],
        c.[Fiscal Quarter Sort]

    FROM [Dimension].[Calendar] c
    LEFT JOIN (
        -- Collapse multi-province holidays: a date is a stat holiday if it has
        -- any national (Province Code IS NULL) OR any matching-province row.
        -- This view uses a national OR any-province union — extend the WHERE
        -- clause if province-specific filtering is needed at the report level.
        SELECT DISTINCT [Date Key]
        FROM [Dimension].[StatHolidays]
        -- Remove the WHERE to flag a day as holiday if any province observes it,
        -- or add: WHERE [Province Code] IS NULL  (national only)
        -- or add: WHERE [Province Code] IN ('ON', 'BC') (subset)
    ) h ON c.[Date Key] = h.[Date Key];
GO
```

**Province filtering note:** The default view above marks a date as `[Is Stat Holiday] = 1`
if *any* holiday row exists for that date (national or provincial). If reports need
province-specific working-day counts, add a `[Province Code]` parameter or create
separate filtered views (e.g., `SSAS.v_Calendar_ON`).

---

## 5. SSAS Tabular Configuration Notes

When the SSAS Tabular model imports `'Calendar'` from `[SSAS].[v_Calendar]`, configure
the following in Tabular Editor / TMDL:

### Hidden columns (used for sort-by, not shown in reports)
```
[Date Key], [Day Of Week], [Month], [Quarter],
[Week Start Date Key], [Week End Date Key],
[Month Start Date Key], [Month End Date Key],
[Quarter Start Date Key], [Quarter End Date Key],
[Year Start Date Key], [Year End Date Key],
[Fiscal Year Start Date Key], [Fiscal Year End Date Key],
[Fiscal Period Sort], [Fiscal Quarter Sort]
```

### Sort-By column assignments
| Column | Sort By |
|---|---|
| `[Day Name]` | `[Day Of Week]` |
| `[Day Name Short]` | `[Day Of Week]` |
| `[Month Name]` | `[Month]` |
| `[Month Name Short]` | `[Month]` |
| `[Quarter Label]` | `[Quarter]` |
| `[Fiscal Period Label]` | `[Fiscal Period Sort]` |
| `[Fiscal Quarter Label]` | `[Fiscal Quarter Sort]` |
| `[Fiscal Year Label]` | `[Fiscal Year]` |
| `[Fiscal Year Label Short]` | `[Fiscal Year]` |

### Date table marking
Mark `'Calendar'` as a Date Table in SSAS using the `[Date]` column. This enables
SSAS built-in time intelligence functions.

```
// Tabular Editor 2 C# script to mark as Date Table:
Model.Tables["Calendar"].DataCategory = "Time";
Model.Tables["Calendar"].Columns["Date"].IsKey = true;
```

### Display folders (recommended)
```
Calendar\Day          — Day Of Month, Day Of Week, Day Name, Day Name Short,
                        Is Weekday, Day Of Year
Calendar\Week         — Week Of Year, Week Start Date Key, Week End Date Key
Calendar\Month        — Month, Month Name, Month Name Short, Year Month,
                        Month Start Date Key, Month End Date Key
Calendar\Quarter      — Quarter, Quarter Label, Quarter Start/End Date Key
Calendar\Year         — Year, Year Label, Year Start/End Date Key
Fiscal\Fiscal Year    — Fiscal Year, Fiscal Year Label, Fiscal Year Label Short,
                        Fiscal Year Start/End Date Key, Is Current Fiscal Year,
                        Relative Fiscal Year
Fiscal\Fiscal Period  — Fiscal Period, Fiscal Period Label, Fiscal Period Sort
Fiscal\Fiscal Quarter — Fiscal Quarter, Fiscal Quarter Label, Fiscal Quarter Sort
Flags                 — Is Stat Holiday, Is Working Day, Is Current Day,
                        Is Current Month, Is Current Calendar Year
Relative              — Relative Day, Relative Month, Relative Fiscal Year
```

---

## 6. Extended Properties

Run `Dimension.PopulateCalendar` first, then execute the extended properties script.
Key properties to document (generate full script via Mode C):

| Object | Property | Value |
|---|---|---|
| `Dimension.Calendar` | `MS_Description` | `Date dimension. Range 2000-01-01 to 2050-12-31. Date Key = -1 is unknown member. Fiscal year is Apr 1 – Mar 31 (FY2025 = Apr 2024 – Mar 2025). Week numbering is Sunday-start (US convention).` |
| `Dimension.Calendar.[Date Key]` | `MS_Description` | `Integer surrogate key in YYYYMMDD format. -1 = Unknown member (for open/active records without a known date).` |
| `Dimension.Calendar.[Fiscal Year]` | `MS_Description` | `Fiscal year integer. FY2025 covers Apr 1 2024 – Mar 31 2025.` |
| `Dimension.Calendar.[Is Working Day]` | `MS_Description` | `In SSAS.v_Calendar view only. 1 = weekday and not a stat holiday per Dimension.StatHolidays.` |
| `Dimension.StatHolidays` | `MS_Description` | `Statutory and public holidays. Populated by Nager.Date C# project output. Province Code = NULL means national holiday.` |

---

## 7. Deployment Checklist

- [ ] `Dimension.StatHolidays` table created (Section 1)
- [ ] `Dimension.Calendar` table created (Section 2)
- [ ] `Dimension.PopulateCalendar` SP created and executed — verify 18,628 rows + 1 unknown member row
- [ ] Nager.Date holiday script executed against `Dimension.StatHolidays`
- [ ] `SSAS.v_Calendar` view created (Section 4) — test with `SELECT TOP 10 * FROM SSAS.v_Calendar WHERE [Is Stat Holiday] = 1`
- [ ] SSAS Tabular model updated: 'Calendar' partition source → `SELECT * FROM [SSAS].[v_Calendar]`
- [ ] Date table marking applied in Tabular Editor (Section 5)
- [ ] Sort-by columns assigned (Section 5)
- [ ] Hidden columns set (Section 5)
- [ ] Display folders assigned (Section 5)
- [ ] Extended properties generated and deployed via DACPAC post-deploy script
- [ ] SSAS model processed and validated in DAX Studio

**Row count validation:**
```sql
-- Expect 18,628 rows (2000-01-01 to 2050-12-31) + 1 unknown member (-1) = 18,629
SELECT COUNT(*) FROM [Dimension].[Calendar];  -- expected: 18629

-- Verify fiscal year boundaries
SELECT [Date Key], [Date], [Fiscal Year], [Fiscal Year Label], [Fiscal Period]
FROM [Dimension].[Calendar]
WHERE [Date] IN ('2024-03-31', '2024-04-01', '2024-12-31', '2025-01-01', '2025-03-31', '2025-04-01')
ORDER BY [Date Key];

-- Expected:
-- 2024-03-31  FY2024  FY2024  12 (last day of FY2024)
-- 2024-04-01  FY2025  FY2025   1 (first day of FY2025)
-- 2025-03-31  FY2025  FY2025  12 (last day of FY2025)
-- 2025-04-01  FY2026  FY2026   1 (first day of FY2026)

-- Verify working days
SELECT TOP 10 [Date], [Day Name], [Is Stat Holiday], [Is Working Day]
FROM [SSAS].[v_Calendar]
WHERE [Date] BETWEEN '2025-01-01' AND '2025-01-15'
ORDER BY [Date Key];
```
