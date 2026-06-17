# Nano Task Identification (GPT-5 nano — $0.20/1M tokens)

**Purpose**: Identify tasks in this toolkit where GPT-5 nano ($0.20/1M input) is a safe, cost-effective replacement for Haiku ($1.00/1M) or Sonnet ($3.00/1M).

**Nano eligibility criteria (must meet ALL)**:
1. **Template fill-in** — structured input, fixed output shape (no invented structure)
2. **No reasoning/judgment** — pure substitution or rule lookup (not "what type is this?")
3. **Deterministic** — same input always produces identical output
4. **Safe failure** — errors are immediately obvious (syntax error, missing field) — not subtly wrong
5. **Short output** — under ~500 tokens (nano degrades on long generation tasks)

---

## ✅ Confirmed Nano Tasks (3)

### Nano Task 1: sp_addextendedproperty Upsert Boilerplate (Mode C)

**Trigger**: Agent has determined property values (from inference or user confirmation) and needs to emit the T-SQL script for a single object.

**Input** (structured, complete, no inference):
```
schema: Fact
object: SalesTransaction
object_type: TABLE
property_name: MS_Description
property_value: "Sales transactions at daily product-customer-store grain."
```

**Output** (deterministic):
```sql
IF EXISTS (
    SELECT 1 FROM sys.extended_properties
    WHERE major_id = OBJECT_ID(N'Fact.SalesTransaction')
      AND minor_id = 0
      AND name = N'MS_Description'
)
    EXEC sys.sp_updateextendedproperty
        @name = N'MS_Description',
        @value = N'Sales transactions at daily product-customer-store grain.',
        @level0type = N'SCHEMA', @level0name = N'Fact',
        @level1type = N'TABLE',  @level1name = N'SalesTransaction';
ELSE
    EXEC sys.sp_addextendedproperty
        @name = N'MS_Description',
        @value = N'Sales transactions at daily product-customer-store grain.',
        @level0type = N'SCHEMA', @level0name = N'Fact',
        @level1type = N'TABLE',  @level1name = N'SalesTransaction';
GO
```

**Why nano is safe**: No inference about what the description should say — the orchestrator (Sonnet/Haiku) already determined values. Nano only renders the boilerplate. SQL syntax error is immediately obvious.

**Frequency**: Very high — one call per property × per column × per table. A 200-column DW with 5 properties per column = 1,000 calls.

**Cost savings**:
- 1,000 calls × 500 tokens avg = 500K tokens per project
- Nano: $0.20/1M × 500K = **$0.10**
- Haiku: $1.00/1M × 500K = **$0.50**
- Annual (100 projects): **$40/year nano** vs **$200/year Haiku** → $160 saved

**Constraint**: Nano receives **fully resolved values** from orchestrator — it never infers descriptions, classifications, or labels. Orchestrator stays on Haiku/Sonnet.

---

### Nano Task 2: SSIS Catalog Config JSON (Mode K)

**Trigger**: After Phase 2 confirms source/target connection details.

**Input** (structured, complete):
```
project_name: EAO_DW
environments: [DEV, TEST, UAT, PROD, SUPPORT]
source_server: SQL01\MSSQL2022
source_db: EAO_OLTP
target_server: DWSQL01\MSSQL2022
target_db: EAO_DW
```

**Output**: Complete `ssis_catalog_configuration.json` with `#{variable_name}#` token placeholders per environment.

**Why nano is safe**: Literal template substitution. No inference, no classification, no judgment. The JSON structure never changes — only values differ between projects. Token placeholder format `#{...}#` is fixed. JSON parse errors catch bad output.

**Frequency**: Once per project (low), but excellent as a proof-of-concept for nano reliability.

**Cost savings**:
- Low absolute savings (single call, ~2K tokens)
- **Primary value**: Proof-of-concept for nano reliability in this toolkit before enabling for high-frequency tasks

---

### Nano Task 3: Measure FormatString + DisplayFolder + Description Stub (Mode L sub-task)

**Trigger**: After measure list is confirmed; orchestrator has determined measure names and types.

**Input** (structured, complete):
```
measure_name: Total Revenue
measure_type: additive_currency
business_area: Sales
description_body: "Sum of revenue amount. Valid groupings: All dimensions except [Budget]. Notes: Additive."
```

**Output** (deterministic lookup):
```
FormatString: "$#,##0.00"
DisplayFolder: "Sales"
Description: "Sum of revenue amount. Valid groupings: All dimensions except [Budget]. Notes: Additive."
```

**FormatString lookup table** (pass to nano as reference):
| Measure Type | FormatString |
|---|---|
| additive_integer | `"#,##0"` |
| additive_currency | `"$#,##0.00"` |
| additive_decimal | `"#,##0.00"` |
| percentage | `"0.00%"` |
| count | `"#,##0"` |
| days_duration | `"#,##0.0"` |
| ratio_non_additive | `"0.00"` |

**Why nano is safe**: Pure lookup + pass-through. FormatString is a deterministic table lookup. DisplayFolder = business_area (already provided). Description = pass-through (orchestrator already composed it).

**Frequency**: Medium — one per measure. Typical DW: 30–100 measures.

**Cost savings**:
- 100 measures × 300 tokens = 30K tokens per project
- Nano: $0.006 vs Haiku: $0.03 → $0.024 saved per project
- Annual: ~$2.40/year (small absolute savings, but low-risk proof)

---

## ❌ NOT Nano — Requires At Least Haiku

| Task | Mode | Reason |
|---|---|---|
| Table classification (Fact/Dim/Bridge) | A, P | Inference from row count, FK structure, naming — not deterministic |
| Severity classification (🔴/🟠/🟡) | A, B, D, O | Requires judgment about impact; subtle cases exist |
| SCD type determination | A, H | Requires business context (is history needed?) |
| Grain confirmation | 3 | Domain reasoning — edge cases require judgment |
| Inferred FK detection (no constraints) | P | Requires reasoning about column name similarities |
| BPA findings assembly | B | Findings may be obvious per rule, but the report requires grouping + prioritization |
| Measure description inference | C, L | Orchestrator infers from column name / business context → Haiku/Sonnet |
| Data classification (Sensitivity labels) | C | Privacy officer escalation logic; `Unreviewed` default but routing requires judgment |
| Conflict resolution in Phase 3b | 3b | User requirement vs discovered entity mapping — reasoning required |
| DAX expression generation | L | Pattern selection requires semantic understanding |
| SP body (INSERT/MERGE logic) | J | Requires understanding column mappings and source structure |

---

## ⚠️ Already Handled by Tools (Not AI)

These tasks were initially considered for nano, but TE2 BPA rules or T-SQL already handle them automatically:

| Task | Already Handled By |
|---|---|
| Naming convention check (Title Case) | `ORG_COLUMN_TITLECASE` BPA rule in `BPARules.json` |
| Missing Description on measures | `ORG_MEASURE_HAS_DESCRIPTION` BPA rule |
| Missing FormatString | `ORG_MEASURE_HAS_FORMATSTRING` BPA rule |
| Missing DisplayFolder | `ORG_MEASURE_HAS_DISPLAYFOLDER` BPA rule |
| Raw division check (uses `/`) | `ORG_MEASURE_NO_RAW_DIVISION` BPA rule (regex) |
| Bidirectional relationship flag | `ORG_NO_BIDIRECTIONAL_RELATIONSHIPS` BPA rule |
| HideKeyColumns.cs (hide `*Key` cols) | Fixed TE2 script — no generation needed |
| Extended property query (audit) | Fixed T-SQL in `extended-properties-templates.md` — no generation |

**Key insight**: Much of what seemed "nano-eligible" is already automated by BPA rules or fixed scripts. The remaining nano candidates are **output rendering tasks** (emitting boilerplate from already-resolved values), not analysis.

---

## Implementation Plan

### Phase 1: Proof of Concept (Mode K — SSIS Catalog JSON)

**Why start here**: Lowest risk (single call, non-critical artifact, JSON parse catches errors), and fastest to validate.

**Steps**:
1. Update Mode K in `ssas-tabular-dw-architect.agent.md` to route SSIS catalog JSON generation to nano sub-task
2. Prompt: pass input variables + fixed JSON template → nano fills in values
3. Validate JSON parses correctly + all required variables present
4. If pass → proceed to Phase 2

**Effort**: 30 min

---

### Phase 2: High-Frequency Task (Mode C — sp_addextendedproperty Boilerplate)

**Why do this second**: Highest volume (1,000+ calls per project), largest absolute savings.

**Pre-condition**: Phase 1 nano test passes (confirms nano can follow template instructions)

**Steps**:
1. Update Mode C in `ssas-tabular-dw-architect.agent.md`:
   - Orchestrator (Haiku) infers/confirms property values
   - Routes each emit call to nano sub-task
   - Nano emits upsert T-SQL
2. Batch calls when possible (10–20 properties per nano call) to reduce overhead
3. Validate: T-SQL parses, `IF EXISTS` pattern correct, all properties present

**Effort**: 1 hour

---

### Phase 3: Low-Risk High-Confidence (Mode L — FormatString/DisplayFolder/Description)

**Steps**:
1. Embed FormatString lookup table in nano prompt (as reference)
2. Route per-measure formatting to nano
3. Validate: format strings match expected values per type

**Effort**: 30 min

---

## Cost Impact Summary

| Task | Freq/Project | Tokens/Call | Nano Cost | Haiku Cost | Saving/Project | Saving/Year (100) |
|---|---|---|---|---|---|---|
| sp_addextendedproperty | 1,000 | 500 | $0.10 | $0.50 | $0.40 | **$40** |
| SSIS catalog JSON | 1 | 2,000 | $0.0004 | $0.002 | $0.0016 | $0.16 |
| Measure format/display/desc | 50 | 300 | $0.003 | $0.015 | $0.012 | **$1.20** |
| **Total** | | | **$0.103** | **$0.517** | **$0.41** | **$41/year** |

**Note**: $41/year is modest in absolute terms. The real value is:
1. **Responsiveness** — nano is faster than Haiku; high-frequency Mode C calls complete sooner
2. **Proof of concept** — establishes nano reliability for future opportunities
3. **Validation of sub-agent pattern** — confirms the routing architecture works end-to-end

---

## Risk Assessment

| Risk | Likelihood | Mitigation |
|---|---|---|
| Nano generates syntactically invalid T-SQL | Low | SQL parser catches immediately; orchestrator retries |
| Nano generates wrong property name | Medium | Pass property_name explicitly in input (not inferred by nano) |
| Nano misapplies FormatString type | Low | Embed lookup table as reference in nano prompt |
| Nano fails on multi-line output | Low | Batch ≤10 properties per call; test first |
| Nano not available in Copilot CLI session | Unknown | Fallback: Haiku handles same task; zero impact on output quality |

---

## Decision: What to Implement Now

Given the modest cost savings, the sub-agent routing architecture (already committed in b946d9e) delivers more value. However, implementing Mode K (SSIS JSON) nano task is a **low-effort proof of concept** that validates the routing pattern without any quality risk.

**Recommendation**:
1. ✅ **Implement Mode K nano** (30 min) — proof of concept, validate routing
2. ⏳ **Defer Mode C nano** — implement after Phase 1 Sonnet validation passes (quality gate first)
3. ⏳ **Defer Mode L nano** — low savings, implement as final optimization

