## Purpose

This guide defines the organisation DAX coding standard for SSAS Tabular models. Copilot agents must apply this standard when generating DAX measures, reviewing DAX measures, or reviewing SSAS Tabular models. Every generated measure must conform to these rules.

## Guiding Design Philosophy — Upstream-First (Roche's Maxim)

> **"Data should be transformed as far upstream as possible, and as far downstream as necessary."**

This is the single most important principle governing DAX authorship in this organisation:

- **Simple DAX is good DAX.** The best measure is `SUM`, `COUNTROWS`, or `DIVIDE`. If you reach for `FILTER(ALL(...))` or nested `CALCULATE`, first ask whether the model shape is wrong.
- **Complex calculations belong in SQL first.** If a measure is computing something that could be a column in the DW load SP or an SSAS calculated column, push it upstream.
- **Upstream preference order:** Staging SP → Dimension/Fact load SP → DW computed column → SSAS calculated column → DAX measure.
- **Document exceptions.** When complex DAX is genuinely necessary (e.g., ad-hoc time window that cannot be pre-computed), document the reason in the measure `Description` field.

Apply this lens during every DAX review: if a measure could be simplified by a model design change, flag it as a design issue alongside the DAX finding.

### Organisational conventions

- SSAS Tabular only.
- Reports connect to SSAS Tabular using PBIRS live connection; report import mode is not used.
- Date dimension SSAS table: `'Calendar'`.
- Date dimension warehouse table: `Dimension.Calendar`.
- Surrogate keys follow `{EntityName}Key`, are hidden in the model, and are not exposed to report users.
- Visible attributes use Title Case with spaces.
- Roles follow `{Name} Consumers` for Read access and `{Name} Authors` for Read plus Process access.
- Every measure requires `Description`, `FormatString`, and `DisplayFolder`.

## Measure naming conventions

### General naming rules

- Use Title Case for visible measures.
- Use clear business terms, for example `Total Sales Amount`, `Average Order Value`, and `Customer Count`.
- Do not use abbreviations unless they are universally understood. `YTD` is acceptable; shortened names such as `Avg Rev Per Cust` are not.
- Base measures must be descriptive noun phrases, for example `Sales Amount` and `Order Count`.
- Time intelligence variants must start with the time period, for example `YTD Sales Amount`, `Prior Year Sales Amount`, and `YoY Sales Amount Var %`.

### Hidden helper and debug measures

- Hidden helper measures must start with `_`, for example `_Sales Amount Base`.
- Debug measures must start with `_Debug`, for example `_Debug Oldest Source` and `_Debug Model Processed`.
- Debug measures must be hidden unless there is an approved operational reporting use case.

## Mandatory measure properties

### Required properties

Every measure, with no exceptions, must include the following metadata:

```text
Description: [Business description].
Valid groupings: [List dimensions this measure can be meaningfully sliced by].
Notes: [Additive, semi-additive, or non-additive; include caveats].
FormatString: [For example #,##0.00 / #,##0 / 0.00% / "General Date"]
DisplayFolder: [Business area or sub-folder, for example Sales\Time Intelligence]
```

### Complete example

```text
[Total Sales Amount]
Description: Total net sales amount after discounts and returns.
Valid groupings: Calendar, Customer, Product, Region, Sales Channel.
Notes: Fully additive. Source: Fact.SalesTransaction.Amount.
FormatString: #,##0.00
DisplayFolder: Sales
```

## Formatting and indentation

### Formatting standard

- Use DAX Formatter conventions: https://www.daxformatter.com
- Indent 4 spaces per nesting level.
- Put each function argument on its own line when a function has two or more arguments.
- Put the opening parenthesis on the same line as the function name.
- Put the closing parenthesis on its own line, aligned with the function call.

### Bad formatting example

```dax
Total Sales Amount = CALCULATE(SUM(Fact_SalesTransaction[Amount]),FILTER(ALL(Dimension_Customer),Dimension_Customer[Region]="East"))
```

### Good formatting example

```dax
Total Sales Amount =
CALCULATE(
    SUM( 'Fact SalesTransaction'[Amount] ),
    FILTER(
        ALL( 'Customer' ),
        'Customer'[Region] = "East"
    )
)
```

## VAR and RETURN pattern

### Required usage

Any measure with more than one logical step must use `VAR` and `RETURN`. Use variables to capture reusable values, improve readability, and avoid repeated evaluation.

```dax
YoY Sales Amount Var % =
VAR CurrentPeriod = [Total Sales Amount]
VAR PriorPeriod   = [Prior Year Sales Amount]
RETURN
    DIVIDE(
        CurrentPeriod - PriorPeriod,
        PriorPeriod,
        BLANK()
    )
```

## Division standard

### Always use DIVIDE

Use `DIVIDE()` for all division. Do not use the division operator in measures.

```dax
Average Order Value =
DIVIDE(
    [Total Sales Amount],
    [Order Count],
    BLANK()
)
```

## BLANK and zero handling

### No-data scenarios

- Return `BLANK()` when no data is the correct answer.
- Return `0` only when zero is a meaningful business value.
- Derived measures should propagate blanks from base measures when a blank result avoids misleading visuals.

```dax
Sales Amount Per Customer =
IF(
    ISBLANK( [Total Sales Amount] ),
    BLANK(),
    DIVIDE(
        [Total Sales Amount],
        [Customer Count],
        BLANK()
    )
)
```

## Time intelligence

### Calendar table rules

- All time intelligence must reference the `'Calendar'` SSAS table.
- The `'Calendar'` table must be marked as a Date Table in the SSAS model.
- The date column used for time intelligence is `'Calendar'[Date]` and must use a Date data type.
- Confirm the organisational calendar grain and fiscal calendar requirements before creating fiscal calculations.

### Standard time intelligence patterns

```dax
YTD Sales Amount =
CALCULATE(
    [Total Sales Amount],
    DATESYTD( 'Calendar'[Date] )
)
```

```dax
Prior Year Sales Amount =
CALCULATE(
    [Total Sales Amount],
    SAMEPERIODLASTYEAR( 'Calendar'[Date] )
)
```

```dax
YoY Sales Amount Var % =
DIVIDE(
    [Total Sales Amount] - [Prior Year Sales Amount],
    [Prior Year Sales Amount],
    BLANK()
)
```

## Anti-patterns

### Forbidden patterns

| Anti-pattern | Why forbidden | Correct alternative |
|---|---|---|
| Division operator with a denominator measure | Divide-by-zero or misleading error behavior | Use `DIVIDE()` |
| `FILTER(ALL(Table), condition)` | Scans the entire table and can severely reduce performance | Use `KEEPFILTERS()` or `CALCULATETABLE()` |
| `EARLIER()` in measures | It is not appropriate for measure evaluation patterns | Use `VAR` to capture context |
| Implicit auto-measures | Required description, folder, and format metadata cannot be controlled | Always create explicit measures |
| `COUNTROWS(FILTER(...))` | Can force unnecessary table scans | Use `CALCULATE(COUNTROWS(...), condition)` |
| Hardcoded dates | Breaks across fiscal years and calendar changes | Use `'Calendar'` relative period columns |
| `TODAY()` or `NOW()` in cached measures | Cached results can appear stale after processing | Use a `'Calendar'[Is Today]` flag or a data freshness pattern |
| Referencing any date table other than `'Calendar'` | Date relationships and standard time intelligence may not work correctly | Always use `'Calendar'` |

## Display folder conventions

### Folder rules

- Top-level folders must be business area names matching the report domain, for example `Sales`, `Projects`, `Finance`, and `HR`.
- Use sub-folders for variants, for example `Sales\Time Intelligence` and `Sales\Ratios`.
- Debug measures must use the `_Debug` folder.
- Hidden helper measures may be placed in any relevant folder, but must still have a folder for maintainability.
- Never leave measures in the root. Always assign a `DisplayFolder`.

## PBIRS live-connection constraints

### Report authoring constraints

Because reports connect to SSAS Tabular using live connection, these constraints apply:

- Do not create report-level measures.
- Do not use composite model features.
- Do not use the Q&A visual.
- All calculated columns must be created in the SSAS model, not in the report.
- Field parameters are not supported for this reporting pattern.
- Users see exactly what the SSAS model exposes; hide columns and measures that should not be visible.

## Review checklist

### Mode D DAX measure review

When reviewing a DAX measure, check all of the following:

| Check | Severity if failed |
|---|---|
| Uses `DIVIDE()` for division | 🔴 CRITICAL |
| Has `Description` with valid groupings and notes | 🟠 HIGH |
| Has `FormatString` | 🟠 HIGH |
| Has `DisplayFolder` | 🟡 MEDIUM |
| Uses `VAR` and `RETURN` for multi-step logic | 🟡 MEDIUM |
| References `'Calendar'` for time intelligence | 🔴 CRITICAL |
| Avoids `FILTER(ALL())` | 🟠 HIGH |
| Avoids `EARLIER()` in measures | 🟠 HIGH |
| Avoids implicit auto-measures | 🟡 MEDIUM |

### Review outcome expectations

- Flag critical issues before style issues.
- Require remediation for missing metadata.
- Prefer explicit, readable measures over clever compact expressions.
- Confirm the measure can be meaningfully sliced by the stated valid groupings.
- Confirm hidden helper and debug measures are hidden and correctly named.
